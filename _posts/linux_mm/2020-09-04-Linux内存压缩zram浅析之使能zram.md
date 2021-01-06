---
layout:     post
title:      linux内存压缩浅析-使能zram
subtitle:   linux内存压缩浅析-使能zram
date:       2020-09-04
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - linux
   - memory management  
---

## linux内存压缩浅析-使能zram

​         为了验证zram,  我是在openwrt项目上面进行zram验证，打开zram需要打开内核的一些选项

make kernel_menuconfig后打开如下选项：

```
CONFIG_SWAP=y
CONFIG_MEMCG_SWAP=y
CONFIG_MEMCG_SWAP_ENABLED=y
```

make menuconfig后打开如下选项：

```
#打开内核支持swap分区
CONFIG_KERNEL_SWAP=y
#如果需要cgroup支持的组支持swap, 那么需要打开
CONFIG_KERNEL_MEMCG_SWAP=y
CONFIG_KERNEL_MEMCG_SWAP_ENABLED=y

#打开openwrt zram swap的支持, 这个主要是zram的初始化脚本，用于创建交换分区、设置zram分区大小等
CONFIG_PACKAGE_zram-swap=y

#zram分区加密算法
CONFIG_PACKAGE_kmod-lib-lz4=y
CONFIG_PACKAGE_kmod-lib-lzo=y

#打开内核zram模块
CONFIG_PACKAGE_kmod-zram=y
```

​       

打开上述选项后，重新编译固件，开机即可创建zram交换分区成功：

![image-20200904144959188](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20200904144959188.png)

如果想要更改swap分区的大小，那么可以更改/etc/init.d/zram脚本里面的参数设置。

