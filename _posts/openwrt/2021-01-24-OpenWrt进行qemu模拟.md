---
layout:     post
title:      使用QEMU搭建OpenWrt开发环境
subtitle:   使用QEMU搭建OpenWrt开发环境
date:       2021-01-24
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - linux
   - OpenWrt
   - debug  
---

# 使用QEMU搭建OpenWrt开发环境

## 前言

​			作为一名玩OpenWrt系统多年的老菜鸟，如果想要在OpenWrt上实现某些功能，或者验证一些想法，以及学习OpenWrt一些原理，可以在网上去购买二手硬件，拆机、焊串口以及刷机来达到这一目的。但是这种方式比较繁琐，我一直在想有没有一种方法，可以快速的达到这一目的呢。近年来虚拟技术发展很快，是否可以通过虚拟化技术来简化这一过程呢，答案是肯定的，接下来我们一步一步来搭建这个过程。

​			关于OpenWrt的编译框架、以及如何刷机、如何编译、配置内核、以及包管理，请参考[官方文档](https://openwrt.org/docs/guide-developer/start)，这里面详细介绍了OpenWrt的方方面面。对OpenWrt感兴趣的同学一定要仔细阅读。

## 一、下载编译源码

### 1.1 安装虚拟机

​	如何安装linux系统的虚拟机，请直接Google.会有很多安装linux虚拟机的方法。这里不再详细介绍。

### 1.2 搭建虚拟机环境

​	 主要是在虚拟机上面安装一些工具，从而满足OpenWrt的编译，因为OpenWrt在编译过程中会使用到这些工具或者库，如果没有安装完全，这个OpenWrt编译就不会通过。

​	  我用的是`18.04.1-Ubuntu` 执行如下指令进行搭建虚拟开发环境。如果在安装过程中遇到问题，可以自行Google解决。

```shell
sudo apt-get update

sudo apt-get install

sudo apt-get install git gcc g++ binutils patch bzip2 flex bison make autoconf gettext texinfo unzip sharutils libncurses5-dev ncurses-term zlib1g-dev gawk asciidoc libz-dev git-core uuid-dev libacl1-dev liblzo2-dev pkg-config libc6-dev curl libxml-parser-perl ocaml-nox
```

​      也可以分开安装：

```shell
sudo apt-get install git

sudo apt-get install vim-gtk

sudo apt-get install python

sudo apt-get install gcc

sudo apt-get install g++

sudo apt-get install binutils

sudo apt-get install patch

sudo apt-get install bzip2

sudo apt-get install flex

sudo apt-get install bison

sudo apt-get install make

sudo apt-get install autoconf

sudo apt-get install gettext

sudo apt-get install texinfo

sudo apt-get install unzip

sudo apt-get install sharutils

sudo apt-get install subversion

sudo apt-get install libncurses5-dev

sudo apt-get install ncurses-term

sudo apt-get install zlib1g-dev

sudo apt-get install gawk

sudo apt-get install asciidoc

sudo apt-get install libz-dev

sudo apt-get install git-core

sudo apt-get install uuid-dev

sudo apt-get install livacl1-dev

sudo apt-get install liblzo2-dev

sudo apt-get install pkg-config

sudo apt-get install libc6-dev

sudo apt-get install cuel

sudo apt-get install libtool

sudo apt-get install libssl-dev

sudo apt-get install libssl0.9.8

sudo apt-get install xz-utils
```

​		由于每个虚拟机的环境不同，在后续编译过程中如果出现错误，一般来说把提示错误的那个包或者工具安装一下就可以了。

### 1.3 下载代码

​	可以从我的仓库中下载OpenWrt源码， 这个是我从官方fork过来的，而且定期会把社区的一些提交拉过来

```shell
mkdir openwrt
sudo chmod 777 openwrt
cd openwrt
git clone  https://github.com/cclinuxer/openwrt.git source
```

### 1.4  更新源

​	更新软件包，安装最新包

```shell
cd source
./scripts/feeds update -a
./scripts/feeds install -a
```

### 1.5 配置固件

  先执行如下两条命令， 运行下面的命令让OpenWrt编译系统检查你的编译环境中缺失的软件包，如果有缺失的话请先安装上。

```shell
make defconfig
make prereq
```

​	执行`make menuconfig`命令配置固件

####    1.5.1 **选择平台**

   **进入 Target System**：

![image-20210124152353337](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124152353337.png)

**选择ARM QEMU虚拟机**：

![image-20210124152524757](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124152524757.png)

**选择ARM64位**：

![image-20210124152740876](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124152740876.png)

![image-20210124153008043](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124153008043.png)

​		`cortex-A53`是`ARMv8`也就是64位的arm系统，而cortex-a15是arm 32位。 这里我选arm64位来做固件。其他诸如`luci`、`uhttpd`相关的包请自行添加。然后开始编译固件

### 1.6  编译固件

​		 配置好软件包以后，就可以开始编译了。使用如下命令进行编译固件：请参考：[OpenWrt官方文档](https://openwrt.org/docs/guide-developer/quickstart-build-images).

```shell
	make V=s -j4
```

编译好的固件在编译目录下的：`source/bin/targets/armvirt/64`下，我们看下都有哪些文件

![image-20210124180530110](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124180530110.png)

​		其中`openwrt-armvirt-64-Image-initramfs`就是我们需要用到的文件

​	    **如果编译不过，很有可能是你需要一个`梯子翻墙`**

## 2  运行虚拟机

#### 安装arm的qemu

```shell
sudo apt-get install qemu-system-arm
sudo apt-get install qemu-system-aarch64
```

#### 启动qemu

我们将启动QEMU虚拟机写成脚本`qemu_start_openwrt.sh`, 内容如下：

```shell
#!/bin/bash
IMAGE=/home/jie/share/src/study/qemu_openwrt/source/bin/targets/armvirt/64/openwrt-armvirt-64-Image-initramfs
LAN=openwrt_tap0
# create tap interface which will be connected to OpenWrt LAN NIC
ip tuntap add mode tap $LAN
ip link set dev $LAN up
# configure interface with static ip to avoid overlapping routes
ip addr add 192.168.1.241/24 dev $LAN
qemu-system-aarch64 \
    -device virtio-net-pci,netdev=lan \
    -netdev tap,id=lan,ifname=$LAN,script=no,downscript=no \
    -device virtio-net-pci,netdev=wan \
    -netdev user,id=wan \
    -M virt -nographic -m 512 -cpu cortex-a53 -smp 4  -kernel $IMAGE
# cleanup. delete tap interface created earlier
ip addr flush dev $LAN
ip link set dev $LAN down
ip tuntap del mode tap dev $LAN

```

其中需要将`IMAGE`变量替换为你自己的Ubuntu里面编译出来的产物，这个就是给虚拟机运行的固件，在我的环境中是：

```
IMAGE=/home/jie/share/src/study/qemu_openwrt/source/bin/targets/armvirt/64/openwrt-armvirt-64-Image-initramfs
```

另外-m 512表示给QEMU使用的内存。 -netdev和网络相关详细信息请参考[这篇文章](https://www.jianshu.com/p/110b60c14a8b)，[QEMU和外部网络通信](https://blog.csdn.net/zhaihaifei/article/details/58624063)。

具体参数的含义请找Linux里面的男人：

```shell
man qemu-system-aarch64
```

然后用如下命令运行脚本 ， 注意需要sudo执行：（放置脚本的目录下执行）

```shell
 sudo ./openwrt_qemu_start.sh
```

OpenWrt启动了：

![image-20210124184441063](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124184441063.png)

`br-lan`的IP地址为：`192.168.1.1`

![image-20210124184612528](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124184612528.png)

打开一个新的终端，执行 `ifconfig` 可以看到多了一个 `openwrt_tap0` ，`Ubuntu`里面新生成了一个接口`openwrt_tap0`和`OpenWrt`通信， `IP`地址为`192.168.1.124`

![image-20210124184842675](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124184842675.png)

在新的终端中`PING`测试一下网络是否通：

![image-20210124185102639](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124185102639.png)

访问OpenWrt网页：

打开浏览器，地址栏输入`192.168.1.1`即可访问`LuCI`界面：

![image-20210124185243087](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210124185243087.png)

## 3、退出QEMU

`qemu` 退出方法 为`ctrl + a`  按 `x`



## 参考文档

[ARM平台使用qemu运行OpenWrt虚拟机](https://www.sdnlab.com/20532.html)

[Running OpenWrt ARM under QEMU](https://gist.github.com/extremecoders-re/f2c4433d66c1d0864a157242b6d83f67)

[OpenWrt QEMU仿真](http://arkinux.com/posts/detail/mTv042wZR)

[Gdb调QEMU](https://blog.csdn.net/zjhsucceed_329/article/details/79827960)

[QEMU进行内核源码级调试](https://blog.csdn.net/xj178926426/article/details/53323615)


