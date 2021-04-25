



[这篇文章](https://zhuanlan.zhihu.com/p/67936075)在介绍VMA的时候曾讲到：为了提高查找速度，对VMA的管理在传统的双向链表的基础上，加入了rbtree（红黑树），使得查找VMA的时间复杂度由线性的O( ![[公式]](https://www.zhihu.com/equation?tex=n) )降为了对数的O( ![[公式]](https://www.zhihu.com/equation?tex=log_%7B2%7Dn) )（考虑到需要平衡的时间，实际会略慢一些）。

然而，现在某些进程的地址空间中，VMA可能多达上百个，而查找VMA又是内核中非常频繁的操作（比如发生page fault的时候）。为了进一步加快查找速度，内核采用了把最近访问的一个VMA保存起来，作为下次查找的cache的[mmap_cache](https://zhuanlan.zhihu.com/p/68398179)的方式。

**【新的方案】**

这种简易的cache的命中率可以达到35%-50%，但对效率无止境的追求，让内核的开发者没有就此满足，他们设计了一个[新的VMA cache方案](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/588807/)，将原来一个进程（或者说一个地址空间）拥有一个VMA cache，变成了该进程中的每个线程（也就是共享这个地址空间的所有线程）各有一个VMA cache，而且，每个VMA cache的item数量还不是1个，而是4个。

```c
struct task_struct {
    struct vmacache vmacache;
    ...
}

struct vmacache {
    struct vm_area_struct *vmas[4];
    ...
};
```

经过在不同场景下的测试，采用这种**per-thread VMA cache**，可以将命中率提高到75%以上。不过，cache的设计不能光看命中率，还要看延迟，cache容量越大，命中率自然越高，但cache查找的耗时也会越多。最极端的情况下，cache的容量和被它缓存的内容所占的空间一样，那么命中率就是100%，但此时的cache已经没有了任何意义。

**【具体实现】**

来看下这套per-thread VMA cache的算法到底是如何运作的吧。熟悉[硬件cache](https://zhuanlan.zhihu.com/p/65245043)原理的应该知道，内存地址的一部分bits被选作cache的**index**，以定位该地址属于哪个cache line，这是一种hash的思想。同样的，VMA cache也是使用地址中的2个bits作为"vmas[4]"数组的index，至于选其中的哪2个bits则是有讲究的。

对于使能了MMU的系统，CPU使用的是[虚拟地址](https://zhuanlan.zhihu.com/p/66794639)，地址范围较大，因而把虚拟地址的[PMD](https://zhuanlan.zhihu.com/p/67813716)中的最低2个bits作为index，效果较好。否则使用的就是物理地址，选择PTE中的最低2个bits作为index更为合适（参考[这个patch](https://link.zhihu.com/?target=https%3A//github.com/torvalds/linux/commit/ddbf369c0a33924f76d092985bd20d9310f43d7f%23diff-615c4cb1cfe26584ba140a2df47362f2)）。

```go
#ifdef CONFIG_MMU
#define VMACACHE_SHIFT	PMD_SHIFT
#else
#define VMACACHE_SHIFT	PAGE_SHIFT
#endif
#define VMACACHE_HASH(addr) ((addr >> VMACACHE_SHIFT) & 0x3)
```

> **VMA cache的替换**

当一个地址最近被访问到，就会从该地址中抽取对应的2个bits作为"vmas[4]"数组的下标，将该地址所在的VMA的"vm_area_struct"的指针填入这个小标对应的数组item，替换的原则同硬件cache类似。

```go
void vmacache_update(unsigned long addr, struct vm_area_struct *newvma)
{
    if (vmacache_valid_mm(newvma->vm_mm))
        current->vmacache.vmas[VMACACHE_HASH(addr)] = newvma;
}
```

> **VMA cache的查找**

对于index相同的内存地址，硬件cache的**tag比对**使用的是地址中除了index以外的另一部分bits，而VMA cache的tag比对就没那么简单了，因为"vmas[]"数组存储的并不是程序使用的地址，而是"vm_area_struct"的指针的值。

要判断一个地址所在的VMA是否已经包含在了VMA cache中，需要判断该地址是否“落”在其中一个VMA cache的"vm_start"和"vm_end"之间（代码位于"/mm/vmacache.c"）。

```go
struct vm_area_struct *vmacache_find(struct mm_struct *mm, unsigned long addr)
{
    int idx = VMACACHE_HASH(addr);

    // 遍历vma cache数组
    for (int i = 0; i < 4; i++) {
        struct vm_area_struct *vma = current->vmacache.vmas[idx];

        if (vma) {
            // 比较地址范围
            if (vma->vm_start <= addr && vma->vm_end > addr) {
                count_vm_vmacache_event(VMACACHE_FIND_HITS);
                return vma;
            }
        }
        if (++idx == 4)
            idx = 0;
    }

    //cache没有命中，只能去rbtree找
    return NULL;
}
```

这个顺次查找"vmas[]"数组的“for循环”看似平淡无奇，但细细研究，其实很有说道。选择"PMD_SHIFT"或者"PAGE_SHIFT"作为划分，依据的是VMA的常规大小，目的就是为了让一个VMA的地址区域尽可能落在同一个"index"所属的范围内。

所以，一个地址通过自己的"index"找到的VMA cache，正好就是自己所属的VMA的**概率**是最大的。但是，一个VMA的地址范围也完全有可能跨越多个"index"对应的区域。如下图所示，当访问了"addr 1"，那么"addr 1"所在的VMA就会被添加到"vmas[2]"中，而这个VMA的范围横跨了两个"index"区域。

![img](https://pic1.zhimg.com/80/v2-a0eb0289a6225b35499d023b3b7a5284_720w.jpg)

接下来，当访问"addr 2"时，由于其包含的"index"为"0 1"，因而最先找到的VMA cache就是"vmas[1]"，但这并不意味着"addr 2"所属的VMA（即"vmas[2]"）不在既有的VMA cache中，所以还需要继续查找。直到遍历完"vmas[]"数组都没找到，才能说明这个地址所属的VMA确实不在VMA cache中，才会去VMA的rbtree中查找。

其他的几种场景，笔者就没有一一构造了，但都是有可能发生的，从当前index开始，逐次加1，直到为4时归0，概率大体上应该是依次降低的。

查看"per-thread VMA cache"的patch提交的历史记录可以发现，其[最初的设计](https://link.zhihu.com/?target=https%3A//github.com/torvalds/linux/commit/615d6e8756c87149f2d4c1b93d471bca002bd849)就是只查询一个地址的"index"对应的VMA cache是不是自己所属的VMA，此时设计4个items的目的也仅仅是**减少hash collision**。而后经过优化，修改为目前这种遍历整个数组所有items的方案，4个items又增加了一个进一步**提高命中率**的作用。

同CPU提供的用于缓存内存的硬件cache，以及内存中用于缓存磁盘文件的[page cache](https://zhuanlan.zhihu.com/p/68071761)不同，VMA cache和被它缓存的VMA都存在于内存中，处于同种存储介质上，cache的作用只是缩小了查找的范围。而硬件cache和page cache都和被它们缓存的内容处在速度差别很大的存储介质上，藉此获得访问速度上的提升。

> **VMA cache的失效**

当一个VMA被移除/合并时，其对应的"vm_area_struct"就不复存在了，如果它存在于VMA cache中，那么这个VMA cache就应该被标记为**invalidate**状态，避免接下来继续访问已经不存在的VMA。

因为很多线程的VMA cache中可能都含有这个"vm_area_struct"，一个个去设置为NULL来标记比较麻烦，所以同硬件cache使用"modified"和"invalidate"标志位不同，"per-thread VMA cache"是用数字"**vmacache_seqnum**"的相对大小来表达这种关系。

```c
struct mm_struct {
    u64 vmacache_seqnum;
    ...
}

struct vmacache {
    u64 seqnum;
    ..
};
```

当一个VMA cache失效后，其所属进程（地址空间）的"mm_struct"中的"vmacache_seqnum"的值就会加1，而VMA cache中的"seqnum"的值保持不变。人家变你不变，就是“过时”，挺巧妙的。

```go
static inline void vmacache_invalidate(struct mm_struct *mm)
{
    mm->vmacache_seqnum++;
}
```

接下来线程每次查找VMA cache的时候，都会比较自己手头的VMA cache中的"seqnum"的值，和线程所属进程地址空间的"vmacache_seqnum"的值，如果不等，就说明这个VMA cache是失效的。这时再做cache的invalidate/flush操作也不迟，又一个"lazy"延迟操作思想的体现。

```go
static bool vmacache_valid(struct mm_struct *mm)
{
    struct task_struct *curr = current;

    if (mm->vmacache_seqnum != curr->vmacache.seqnum) {
        curr->vmacache.seqnum = mm->vmacache_seqnum;
        vmacache_flush(curr);
        return false;
    }
    return true;
}
```

这个小模块的代码整个加起来也不到200行，但蕴含了很多设计思想，非常适合逐行做透彻的研究。



**参考：**

[Optimizing VMA caching](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/589475/)

[mm: per-thread vma caching](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/588807/)