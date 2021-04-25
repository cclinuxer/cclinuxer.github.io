---
layout:     post
title:      Systemtap移植到Openwrt
subtitle:   Systemtap移植到Openwrt
date:       2020-09-22
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - linux
   - tools 
---

#  Systemtap移植到Openwrt





https://zhuanlan.zhihu.com/p/28680568



比较好的安装教程

https://www.cnblogs.com/danxi/p/6417821.html



使用技巧：

https://segmentfault.com/a/1190000010774974



staprun安装流程：

https://segmentfault.com/a/1190000020600682?utm_source=sf-related



内核需要支持的编译选项：

https://sourceware.org/systemtap/wiki/SystemtapOnFedoraArm



解决版本不一致的问题：

https://www.bbsmax.com/A/A7zgNoykd4/



systemtap使用笔记

[https://nanxiao.me/category/%E6%8A%80%E6%9C%AF/systemtap-%E7%AC%94%E8%AE%B0/](https://nanxiao.me/category/技术/systemtap-笔记/)

 sudo stap -r /data/jingdong_v6/mesh/kernel/linux-4.4 -a arm -B CROSS_COMPILE=/opt/toolchain-arm_cortex-a7_gcc-5.2.0_musl-1.1.16_eabi/bin/arm-openwrt-linux-muslgnueabi- -m hello.ko  -e 'probe begin {printf("read performed\n"); exit()}'



解决掉coredump问题;

原因是reader_thread中read的buf申请过大，导致core dump. 这点需要注意下。会导致访问溢出， 还有就是buffer生成有问题



解决kernel panic问题：

root@OpenWrt:/tmp# insmod sys_read.ko 
      [  136.520834] vmap allocation for size 3854336 failed: use vmalloc=<size> to increase size.

加载内核模块的时候显示错误看起来是虚拟内存空间不够导致

