---
layout:     post
title:      使用linux page owner看黑洞内存
subtitle:   使用linux page owner看黑洞内存
date:       2021-05-10
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - linux
   - memory management
   - debug
---

# 使用linux page owner看黑洞内存

## 前言

​	在前面的文章中《linux内存管理浅析》中，我们提到了“黑洞内存”，也就是这些内存在内核中是以页为单位被申请走的，并且在内核中无法通过/proc或者/sys文件系统查看这部分内存被谁申请走了。通常情况下是设备驱动或者内核模块调用kmalloc申请了大于8K（slab是4M,这里是以slub为例子）的连续物理内存，或者调用`alloc_page`直接向伙伴系统申请物理内存，并且没有将这部分内存统计在内核的相关字段中，所以这些内存往往无法通过常规的工具进行排查。

​	之前在项目中经常遇到`free`命令看到总的物理内存总是和实际的内存条的内存对应不上，因为free中看到的总的物理内存是统计的伙伴系统中的内存，有些内存可能被预留给一些硬件使用了，这些内存可以在DTS（ARM）文件中看到，这部分内存甚至不是通过伙伴系统分配的，在系统启动的时候就直接划给相应的模块使用了。如下图所示：

![image-20210510095211036](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210510095211036.png)

​	另外用于描述伙伴系统中的内存的`struct page`这个页描述符也可能会占用一些内存，目前内核中的一些优化`patch`也是在想办法减少这块内存的使用。例如如下的一些`patch`:

​    https://lore.kernel.org/linux-mm/CAMZfGtWaSGCUaubv6kwc1hzRoc9=O2eXJBcU9t8bX3XeQtP9Yw@mail.gmail.com/#r

​    https://mp.weixin.qq.com/s/DmoUynuyI8Wdwa_6kixgjw

​	本文的重点不是系统预留以及`page struct`预留的内存，而是系统起来后驱动或者是内核模块申请内存最终通过alloc_pages从伙伴系统申请走了的内存。

## 工具

​     主要是打开内核page owner，hook内存使用，然后通过脚本工具来筛选出感兴趣的内存，实现进一步优化。工具使用请参考如下内核文档

​      https://www.kernel.org/doc/html/v4.18/vm/page_owner.html

​      工具的输出数据如下:

```
Page allocated via order 9, mask 0x4742ca(GFP_TRANSHUGE|__GFP_THISNODE)
PFN 23552 type Movable Block 46 type Movable Flags 0xfffffc0048068(uptodate|lru|active|head|swapbacked)
 __alloc_pages_nodemask+0xf9/0x270
 khugepaged_alloc_page+0x39/0x70
 khugepaged_scan_mm_slot+0x746/0x1cb0
 khugepaged+0x134/0x4b0
 kthread+0xf8/0x130
 ret_from_fork+0x35/0x40

Page allocated via order 9, mask 0x4742ca(GFP_TRANSHUGE|__GFP_THISNODE)
PFN 24064 type Movable Block 47 type Movable Flags 0xfffffc0048068(uptodate|lru|active|head|swapbacked)
 __alloc_pages_nodemask+0xf9/0x270
 khugepaged_alloc_page+0x39/0x70
 khugepaged_scan_mm_slot+0x746/0x1cb0
 khugepaged+0x134/0x4b0
 kthread+0xf8/0x130
 ret_from_fork+0x35/0x40

Page allocated via order 9, mask 0x4742ca(GFP_TRANSHUGE|__GFP_THISNODE)
PFN 24576 type Movable Block 48 type Movable Flags 0xfffffc0048068(uptodate|lru|active|head|swapbacked)
 __alloc_pages_nodemask+0xf9/0x270
 khugepaged_alloc_page+0x39/0x70
 khugepaged_scan_mm_slot+0x746/0x1cb0
 khugepaged+0x134/0x4b0
 kthread+0xf8/0x130
 ret_from_fork+0x35/0x40

Page allocated via order 9, mask 0x4742ca(GFP_TRANSHUGE|__GFP_THISNODE)
PFN 25088 type Movable Block 49 type Movable Flags 0xfffffc0048068(uptodate|lru|active|head|swapbacked)
 __alloc_pages_nodemask+0xf9/0x270
 khugepaged_alloc_page+0x39/0x70
 khugepaged_scan_mm_slot+0x746/0x1cb0
 khugepaged+0x134/0x4b0
 kthread+0xf8/0x130
 ret_from_fork+0x35/0x40
 ........................
```

通过上面的输出，我们可以去查看系统中哪些模块申请了大量的黑洞内存，从而进行内存进一步优化。