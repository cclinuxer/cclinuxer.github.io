---
layout:     post
title:      一次硬件故障导致的kernelpanic排查
subtitle:   一次硬件故障导致的kernelpanic排查
date:       2021-05-11
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - linux
   - tools 
   - debug
---

# 一次硬件故障导致的kernel-panic排查

## 前言

​	最近遇到一个由于硬件故障导致的`kernel panic`, 把过程记录下来，方便后续整理思路，逐渐完成自己的知识库。之前记录这些太少了，导致很多之前的经验后面又需要重头分析一遍，很花费时间。

## 分析过程

​	第一步收集内核的`coredump`以及带符号表的`vmlinux`二进制文件，关于如何打开内核`coredump`以及获取对应的`vmlinux`文件，这里不再多说，请自行`Google`。

​	先用看下内核挂掉的时候`调用栈`在哪个地方，看下能否发现一些蛛丝蚂迹：

![image-20210511101311031](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511101311031.png)

  结合使用使用`dmesg`看下内核挂在什么地方，看下能不能看到些什么异常，看是否能够匹配上。

![image-20210511102153284](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511102153284.png)

​		从`dmesg`的信息基本可以确认，这个`kernel panic`是由于在 `__rmqueue+0x35a/0x4d0`这条指令的地方访问了一个非法地址导致的`kernel panic`  ,  我们想要知道这条指令是什么，于是用`crash`工具里面的`dis`命令看下这条指令是什么。

​	   `dis -l __rmqueue -x`，输出如下：我这里仅仅是截图 `__rmqueue+0x35a`附近的代码出来

![image-20210511103108800](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511103108800.png)

打开对应的源代码，看下是在干什么事情：

![image-20210511103546625](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511103546625.png)

![image-20210511103738502](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511103738502.png)

这里很奇怪，不是说是在`__rmqueue` 这个函数中吗？为什么跑到`__rmqueue_smallest`里面来了，请仔细看：`__rmqueue` 中调用了

`__rmqueue_smallest`，而`__rmqueue_smallest`是一个`inline`的内联函数，所以本质上也是属于`__rmqueue` 的。这里就不再多说了。

```c
static struct page *__rmqueue(struct zone *zone, unsigned int order,
						int migratetype)
{
	struct page *page;

retry_reserve:
	page = __rmqueue_smallest(zone, order, migratetype);

	if (unlikely(!page) && migratetype != MIGRATE_RESERVE) {
		page = __rmqueue_fallback(zone, order, migratetype);

		/*
		 * Use MIGRATE_RESERVE rather than fail an allocation. goto
		 * is used because __rmqueue_smallest is an inline function
		 * and we want just one call site
		 */
		if (!page) {
			migratetype = MIGRATE_RESERVE;
			goto retry_reserve;
		}
	}

	trace_mm_page_alloc_zone_locked(page, order, migratetype);
	return page;
}

```

从源代码可以看出，看起来是从伙伴系统上面取页描述符下来的时候出了问题，第一反应是，这个地址怎么会出现错误呢，这可是一个基础函数啊，这怎么可能出错。我们继续看下这个出错的内存地址（`ffffe200d1eef028`），看下有内容没有，是否可以通过这些内存的内容排查下：

![image-20210511111632011](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511111632011.png)

用`rd`命令看下这段内存，发现这段内存是一个非法的内存地址，于是我们只能在看代码了。

```c
static inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
						int migratetype)
{
	unsigned int current_order;
	struct free_area * area;
	struct page *page;

	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = &(zone->free_area[current_order]);
		if (list_empty(&area->free_list[migratetype]))
			continue;

		page = list_entry(area->free_list[migratetype].next,
							struct page, lru);
		list_del(&page->lru);
		rmv_page_order(page);
		area->nr_free--;
		expand(zone, page, order, current_order, area, migratetype);
		return page;
	}

	return NULL;
}
```

从代码可以看到，是在操作`&page->lru`这个字段出现的问题，我们首先排查下这个`struct page`, 而这个`struct page`是从伙伴系统拿下来的，我们来看看`struct free_area`里面的内容是什么。看下能不能找到什么异常，我们可以通过汇编代码看下，怎么去取得`struct free_area`的内容，首先看下`__rmqueue`函数的参数传递：

![image-20210511142225390](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511142225390.png)

在之前的文章中我们讲到，一般来说`x64`进行函数参数传递的时候，一般会把第一个参数传递给`rdi`这个寄存器，函数一进来又把`rdi`的值传递给`r13`. 也就是说后续我们可以在`r13`里面取`struct zone`的地址。接下来看下出问题附近的代码，看下是不是这样，因为那附近也有用`zone`值的代码：

```assembly
/usr/src/debug/kernel-default-3.0.101/linux-3.0/mm/page_alloc.c: 835
0xffffffff81104542 <__rmqueue+0x342>:   lea    (%rcx,%rdx,2),%rdx
0xffffffff81104546 <__rmqueue+0x346>:   shl    $0x3,%rdx
0xffffffff8110454a <__rmqueue+0x34a>:   lea    (%r11,%rdx,1),%rax
0xffffffff8110454e <__rmqueue+0x34e>:   mov    0x88(%r13,%rax,1),%rsi ;这种move指令一般是访问数组元素
0xffffffff81104556 <__rmqueue+0x356>:   lea    -0x28(%rsi),%r14
/usr/src/debug/kernel-default-3.0.101/linux-3.0/include/linux/list.h: 106
0xffffffff8110455a <__rmqueue+0x35a>:   mov    0x28(%r14),%rcx
0xffffffff8110455e <__rmqueue+0x35e>:   mov    0x30(%r14),%rax
```

重点关注汇编代码：`0x88(%r13,%rax,1),%rsi`，其对应的c代码如下：

![image-20210511142906011](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511142906011.png)

通过这里可以确认，函数代码执行到出问题的地方，`r13`寄存器里面存放的还是`struct zone`,  `0x88`是`free_area`在`struct zone`中的偏移，我们可以通过`struct`命令来进一步确认一下：

![image-20210511143316995](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511143316995.png)

没错吧，`r13`现在存放的是`struct zone`的地址。我们查看下`zone->free_area`的内容：先看下`r13`的内容

![image-20210511143549749](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511143549749.png)

`ffff88407ff98e80`就是`struct zone`的内容。我们使用如下命令`zone.free_area`  `ffff88407ff98e80`查看下`free_area`的内容：

```
  free_area = {{
      free_list = {{
          next = 0xffff88407ff98f08,
          prev = 0xffff88407ff98f08
        }, {
          next = 0xffffea00d182e3e0,
          prev = 0xffffea00d182e3e0
        }, {
          next = 0xffff88407ff98f28,
          prev = 0xffff88407ff98f28
        }, {
          next = 0xffff88407ff98f38,
          prev = 0xffff88407ff98f38
        }, {
          next = 0xffff88407ff98f48,
          prev = 0xffff88407ff98f48
        }},
      nr_free = 1
    }, {
      free_list = {{
          next = 0xffff88407ff98f60,
          prev = 0xffff88407ff98f60
        }, {
          next = 0xffffea00d182e418,
          prev = 0xffffea00d182e418
        }, {
          next = 0xffff88407ff98f80,
          prev = 0xffff88407ff98f80
        }, {
          next = 0xffff88407ff98f90,
          prev = 0xffff88407ff98f90
        }, {
          next = 0xffff88407ff98fa0,
          prev = 0xffff88407ff98fa0
        }},
      nr_free = 1
    }, {
      free_list = {{
          next = 0xffff88407ff98fb8,
          prev = 0xffff88407ff98fb8
        }, {
          next = 0xffffea00d182e488,
          prev = 0xffffea00d182e488
        }, {
          next = 0xffff88407ff98fd8,
          prev = 0xffff88407ff98fd8
        }, {
          next = 0xffff88407ff98fe8,
          prev = 0xffff88407ff98fe8
        }, {
          next = 0xffff88407ff98ff8,
          prev = 0xffff88407ff98ff8
        }},
      nr_free = 1
    }, {
      free_list = {{
          next = 0xffff88407ff99010,
          prev = 0xffff88407ff99010
        }, {
          next = 0xffffea00d182e568,
          prev = 0xffffea00d182e568
        }, {
          next = 0xffffea00d174f368,
          prev = 0xffffea00d174f368
        }, {
          next = 0xffff88407ff99040,
          prev = 0xffff88407ff99040
        }, {
          next = 0xffff88407ff99050,
          prev = 0xffff88407ff99050
        }},
      nr_free = 2
    }, {
      free_list = {{
          next = 0xffff88407ff99068,
          prev = 0xffff88407ff99068
        }, {
          next = 0xffff88407ff99078,
          prev = 0xffff88407ff99078
        }, {
          next = 0xffff88407ff99088,
          prev = 0xffff88407ff99088
        }, {
          next = 0xffff88407ff99098,
          prev = 0xffff88407ff99098
        }, {
          next = 0xffff88407ff990a8,
          prev = 0xffff88407ff990a8
        }},
      nr_free = 0
    }, {
      free_list = {{
          next = 0xffff88407ff990c0,
          prev = 0xffff88407ff990c0
        }, {
          next = 0xffffea00d182e728,
          prev = 0xffffea00d182e728
        }, {
          next = 0xffffea00d174f528,
          prev = 0xffffea00d174f528
        }, {
          next = 0xffff88407ff990f0,
          prev = 0xffff88407ff990f0
        }, {
          next = 0xffff88407ff99100,
          prev = 0xffff88407ff99100
        }},
      nr_free = 2
    }, {
      free_list = {{
          next = 0xffff88407ff99118,
          prev = 0xffff88407ff99118
        }, {
          next = 0xffffea00d182ee28,
          prev = 0xffffea00d182ee28
        }, {
          next = 0xffff88407ff99138,
          prev = 0xffff88407ff99138
        }, {
          next = 0xffff88407ff99148,
          prev = 0xffff88407ff99148
        }, {
          next = 0xffff88407ff99158,
          prev = 0xffff88407ff99158
        }},
      nr_free = 1
    }, {
      free_list = {{
          next = 0xffff88407ff99170,
          prev = 0xffff88407ff99170
        }, {
          next = 0xffffea00d182fc28,
          prev = 0xffffea00d182fc28
        }, {
          next = 0xffffea00d174fc28,
          prev = 0xffffea00d174fc28
        }, {
          next = 0xffff88407ff991a0,
          prev = 0xffff88407ff991a0
        }, {
          next = 0xffff88407ff991b0,
          prev = 0xffff88407ff991b0
        }},
      nr_free = 2
    }, {
      free_list = {{
          next = 0xffff88407ff991c8,
          prev = 0xffff88407ff991c8
        }, {
          next = 0xffffea00d1831828,
          prev = 0xffffea00d1831828
        }, {
          next = 0xffffea00d1751828,
          prev = 0xffffea00d1751828
        }, {
          next = 0xffff88407ff991f8,
          prev = 0xffff80407ff991f8
        }, {
          next = 0xffff88407ff99208,
          prev = 0xffff88407ff99208
        }},
      nr_free = 2
    }, {
      free_list = {{
          next = 0xffffe200d1eef028,
          prev = 0xffffea00d1eef028
        }, {
          next = 0xffffea00d1835028,
          prev = 0xffffea00d1835028
        }, {
          next = 0xffffea00d1755028,
          prev = 0xffffea00d1755028
        }, {
          next = 0xffff88407ff99250,
          prev = 0xffff88407ff99250
        }, {
          next = 0xffff88407ff99260,
          prev = 0xffff88407ff99260
        }},
      nr_free = 3
    }, {
      free_list = {{
          next = 0xffff88407ff99278,
          prev = 0xffff88407ff99278
        }, {
          next = 0xffff88407ff99288,
          prev = 0xffff88407ff99288
        }, {
          next = 0xffffea00d1740028,
          prev = 0xffffea0071c0e028
        }, {
          next = 0xffffea0071c00028,
          prev = 0xffffea0071c00028
        }, {
          next = 0xffff88407ff992b8,
          prev = 0xffff88407ff992b8
        }},
      nr_free = 28001
    }},
```

如下的内容比较奇怪：

![image-20210511144248853](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511144248853.png)

  我解释一下下面的内容，由于`nr_free`等于3， 代表这个`free_list`中有三个块， 当`free_list`链表为空的时候：

   ![image-20210511145134111](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511145134111.png)

也就是`next`和`prev`都是指向的`free_list`, 我们可以反推如果为空，那么`next`和`prev`的地址都在`struct zone`中, 如果`prev`和`next`指针相等，那么还有一种情况，就是链表只有一个元素，但是`next`和`prev`地址都不在`struct zone`中

![image-20210511145955432](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210511145955432.png)

  那么可以推测出：

```
      free_list = {{
          next = 0xffffe200d1eef028,
          prev = 0xffffea00d1eef028
        }, {
          next = 0xffffea00d1835028,
          prev = 0xffffea00d1835028
        }, {
          next = 0xffffea00d1755028,
          prev = 0xffffea00d1755028
        }, {
          next = 0xffff88407ff99250,
          prev = 0xffff88407ff99250
        }, {
          next = 0xffff88407ff99260,
          prev = 0xffff88407ff99260
        }},
      nr_free = 3
```

前三项都只有一个元素，后两项为空。于是我们看下第一项的内容

```
      free_list = {{
          next = 0xffffe200d1eef028
          prev = 0xffffea00d1eef028
```

`0xffffe200d1eef028`这个地址理论上应该是`0xffffea00d1eef028`， 而两个地址只有"`ea`"和“`e2`”这里不同， `a=0x1010 , 2=0x0010`, 一般来说踩内存问题不会只更改一个`bit`位，而且这个位在中间，很大可能是内存硬件`bit`位翻转导致。于是讲问题交给硬件去排查，果然发现了一些异常信息。这个锅甩给硬件了。

值得注意的是，这种问题也不一定百分百是硬件问题，万一有个执行路径，就是随机踩掉这一个bit也是有可能的，但是这种bit位变化，可以先让硬件排查，如果是其他问题，那么就得继续分析了。踩内存一向比较棘手，后面遇到了再具体分析。