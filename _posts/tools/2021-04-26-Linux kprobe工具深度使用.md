---
layout:     post
title:      Linux kprobe工具深度使用
subtitle:   Linux kprobe工具深度使用
date:       2021-04-26
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - linux
   - tools 
   - debug
---

# Linux kprobe工具深度使用

## 一、前言

​	在日常的工作中，无论是在Linux上调试进程还是IO问题、网络问题，很多时候我们想要知道内核里面到底在做什么事情，如何将问题现象和内核行为匹配上来，从而分析问题。 最近分析问题，很多时候一些老的linux系统上面并没有systemtap、更不用说是ebpf这类工具了。之前也时常用kprobe做一些简单工作，由于最近遇到没有前面讲的那些高级工具比较多，所以说我准备系统的把kprobe的用法讲解一下，以便于后续查阅。这个[文档](https://github.com/torvalds/linux/blob/master/Documentation/features/debug/kprobes/arch-support.txt)讲解了哪些架构的内核包括了kprobe. 你在特定体系架构上面使用的时候，请查阅下该文档。

​	kprobe传参需要特定体系架构的函数调用传参ABI相关知识，我在文章后面会贴出用得比较多的几种体系架构相关的文档，其他体系架构相关的知识类似，请自行查找。

​	 由于我自己的实验环境是X86-64的，所以本文讲解的一切除非是特殊说明，否则都是在X86-64适用，其他体系架构可能有些许差别，请理解后自行适配。

## 二、二进制ABI以及函数调用规则

​	在讲解具体的kprobe操作之前，有必要讲解下ABI. 这样我们才能够在使用kprobe的时候知道如何hook函数的参数，局部变量等。关于ABI的概念，网络上讲解的很多，总的来说：它定义了函数被调用的规则：参数在调用者和被调用者之间如何传递，返回值怎么提供给调用者，库函数怎么被应用，以及程序怎么被加载到内存。维基百科上面也有很多讲解，请用搜索引擎搜索相关知识。我这里有篇文章介绍：[What Is the ABI](http://laoar.github.io/blogs/316/). 

​	对于x86_64位， 请参考英特尔的[官方文档](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)。我简单提一下一些调用规则。X64机器有16个通用的寄存器操作数据，它们分别是 *RAX*, *RBX*, *RCX*, *RDX*, *RDI*, *RSI*, *RSP*, *RBP* 以及*R8* 到*R15*，总共16个寄存器。光从名字上面来看，并不能知道每个寄存器的作用，但是接下来我会简单介绍一下这些寄存器的作用。当你在X64上调用一个函数，调用方式以及寄存器的使用都遵循一定的规则，参数在调用者和被调用者之间如何传递，返回值怎么提供给调用者，库函数怎么被应用，以及程序怎么被加载到内存。这也是为什么不同编译器编译出来的二进制可以协同使用的原因。

​	对于kprobe， 使用得最多的就是参数如何传递的。我先给出x64参数传递规则(大于6个参数后，一般使用栈来传递参数，一般情况下，多余6个参数的函数比较少见，如果遇到了，请查看该体系架构的ABI. )然后写个demo程序来证明这个过程。

​	![image-20210426205943951](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426205943951.png)

​	下面，我写一段代码来描述下这些寄存器的作用，同样，我也会用到gdb来协助理解这个过程。

​	  

```c
/*************************************************************************
	> File Name: main.c
 ************************************************************************/

#include<stdio.h>
int test_abi(int a, int b, int c, int d, int e, int f)
{
	int k = a+b+c+d+e+f;
	return c;
}

int main(int argc, int *argv[])
{
	int m = test_abi(1, 2, 3, 4, 5, 6);
	return m;
}

```

使用`gcc`编译

`gcc main.c`得到`a.out`程序

使用`gdb`进行调试`gdb a.out`, 在`test_abi`函数这里打一个断点，然后执行`run`

![image-20210426211009312](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426211009312.png)

执行`run`

![image-20210426211208070](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426211208070.png)

查看这个时候的寄存器值：`info registers`

![image-20210426211349353](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426211349353.png)

看到了把，传递给`test_abi`的参数被依次存放在了寄存器`rdi、rsi、rdx、rcx、r8、r9`.   由于这个例子并没有覆盖所有函数调用的场景，如果遇到输出和预期不符合，请仔细参考相应体系架构的ABI文档。

## 三、详细展示如何hook内核函数

​	接下来我结合`crash`工具来详细示范几个例子展示如何使用kprobe hook内核函数，使用脚本就可以完成这项工作，而不用编写C代码内核官方文档值得参考：内核kprobe[文档](https://www.kernel.org/doc/Documentation/trace/kprobetrace.txt)， 其实这个文档已经将用法讲得比较详细了，唯一的缺点就是缺少例子，我这里补充几个例子，这样更加方便使用。

​	由于我最近看的是网络问题，那么我的例子就讲解如何解析`hook ping`包。希望大家能见微知著，发挥kprobe更多的用处。

​	我首先贴出PING包的收包函数的C代码。

```c
/*
 *	Deal with incoming ICMP packets.
 */
int icmp_rcv(struct sk_buff *skb)
{
	struct icmphdr *icmph;
	struct rtable *rt = skb_rtable(skb);
	struct net *net = dev_net(rt->dst.dev);
	bool success;

	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
		struct sec_path *sp = skb_sec_path(skb);
		int nh;

		if (!(sp && sp->xvec[sp->len - 1]->props.flags &
				 XFRM_STATE_ICMP))
			goto drop;

		if (!pskb_may_pull(skb, sizeof(*icmph) + sizeof(struct iphdr)))
			goto drop;

		nh = skb_network_offset(skb);
		skb_set_network_header(skb, sizeof(*icmph));

		if (!xfrm4_policy_check_reverse(NULL, XFRM_POLICY_IN, skb))
			goto drop;

		skb_set_network_header(skb, nh);
	}

	__ICMP_INC_STATS(net, ICMP_MIB_INMSGS);

	if (skb_checksum_simple_validate(skb))
		goto csum_error;

	if (!pskb_pull(skb, sizeof(*icmph)))
		goto error;

	icmph = icmp_hdr(skb);

	ICMPMSGIN_INC_STATS(net, icmph->type);
	/*
	 *	18 is the highest 'known' ICMP type. Anything else is a mystery
	 *
	 *	RFC 1122: 3.2.2  Unknown ICMP messages types MUST be silently
	 *		  discarded.
	 */
	if (icmph->type > NR_ICMP_TYPES)
		goto error;


	/*
	 *	Parse the ICMP message
	 */

	if (rt->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST)) {
		/*
		 *	RFC 1122: 3.2.2.6 An ICMP_ECHO to broadcast MAY be
		 *	  silently ignored (we let user decide with a sysctl).
		 *	RFC 1122: 3.2.2.8 An ICMP_TIMESTAMP MAY be silently
		 *	  discarded if to broadcast/multicast.
		 */
		if ((icmph->type == ICMP_ECHO ||
		     icmph->type == ICMP_TIMESTAMP) &&
		    net->ipv4.sysctl_icmp_echo_ignore_broadcasts) {
			goto error;
		}
		if (icmph->type != ICMP_ECHO &&
		    icmph->type != ICMP_TIMESTAMP &&
		    icmph->type != ICMP_ADDRESS &&
		    icmph->type != ICMP_ADDRESSREPLY) {
			goto error;
		}
	}

	success = icmp_pointers[icmph->type].handler(skb);

	if (success)  {
		consume_skb(skb);
		return NET_RX_SUCCESS;
	}

drop:
	kfree_skb(skb);
	return NET_RX_DROP;
csum_error:
	__ICMP_INC_STATS(net, ICMP_MIB_CSUMERRORS);
error:
	__ICMP_INC_STATS(net, ICMP_MIB_INERRORS);
	goto drop;
}
```

​	用`crash`来查看函数偏移是一个十分好的方法，这样我们在用kprobe的时候就可以随心所欲的获取参数的值，包括结构体的成员变量（当然你也可以用其他方式来获取偏移值）。

​	首先进入`crash`工具： `sudo crash /usr/lib/debug/boot/vmlinux-5.4.0-48-generic`（我是在ubuntu18.04上面调试的，在使用crash之前，需要安装`kernel debuginfo`.请google搜索安装，这里不再多说）

​	![image-20210426231943183](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426231943183.png)

​	我这里是想hook `icmp_rcv` 函数，主要是读取入参`sk_buff`以及他的成员变量。首先我会读取`sk_buff`的值，以及ping包的数据长度。用`crash`的`struct`命令读取一下`sk_buff`的成员以及偏移：

![image-20210426232653618](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426232653618.png)

​	熟悉内核网络的同学都知道，`sk_buff`的`len`字段代表了数据包的长度，并且是包含`icmp`头部的8个字节的。首先在另外一台电脑上面`ping`当前电脑，指定`ping`包大小为200个字节。

​	![image-20210426234113753](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426234113753.png)

​	所以我在`192.168.68.106`这台机子上面用如下的kprobe进行hook `icmp_rcv`收到的数据包的长度：（前面讲过x64的第一个入参是存放在`rdi`寄存器里面的，使用kprobe的时候，一般`rdi`的r是省略的，就变成了`di`, 以此内推，第二个参数就是`si`）， （我这里用的kprobe命令是用的性能优化大师[brendangregg](https://github.com/brendangregg)的工具：[kprobe工具链接](https://github.com/cclinuxer/perf-tools/blob/master/kernel/kprobe)，大师在`ftrace`的kprobe基础上封装了一下，使kprobe更好使用）

​	`sudo ./kprobe   'p:icmp_rcv skb_addr=%di  len=+112(%di):u32'` 得到如下输出：

​	![image-20210426234329862](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210426234329862.png)

看到了吧，我这样就知道我收到的这个数据包的长度是多少，（这里是208, 也就是8个字节的`icmp`头部加上200个字节数据）以及`sk_buff`的地址。你以为这就完了吗？假设`sk_buff`的一个成员也是结构体怎么办呢？ 没关系的，多层嵌套就可以了。例如我想获取发送过来的`icmp`的`type`, 那我们首先要获取到`struct icmphdr`.  分析代码我们知道`struct icmphdr`在这个`icmp_rcv`函数中，就是`skb->data`, 我们来看下`skb->data`的偏移。同样通过crash工具来看：

![image-20210427000254030](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427000254030.png)

​	从上图可以看到， `skb->data`在`sk_buff`中的偏移是`200`.  这里的首地址正好是存放`struct icmphdr`， 我们再来看下

`struct icmphdr`的布局， 在crash中执行命令：`struct icmphdr -o`,  type偏移为0

​	![image-20210427000722516](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427000722516.png)

​     于是我可以使用如下的`kprobe`命令进行`hook`:

​	`sudo ./kprobe   'p:icmp_rcv skb_addr=%di len=+112(%di):u32 icmp_type=+0(+200(%di)):u8 icmp_code=+1(+200(%di)):u8'`, 输出如下：

![image-20210427001257064](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427001257064.png)

​	看到了吧，连结构体中的成员的成员也可以打印出来。关于如何使用内存偏移请参考[kprobe官方文档](https://github.com/torvalds/linux/blob/master/Documentation/trace/kprobetrace.rst)。

​	 它甚至可以访问`bit`位：例如，我想访问`sk_buff`的`pkt_type`, 它在`sk_buff`的`128`个字节偏移处，占用了三个`bit`

​		![image-20210427002325120](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427002325120.png)

​		官方文档对于bit位的说明如下：

![image-20210427002644342](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427002644342.png)

​	`b<bit-width>@<bit-offset>/<container-size>`

​	于是我们使用如下的kprobe来看下`pkt_type`这个三个bit位的数值：

​	`sudo ./kprobe 'p:icmp_rcv skb_addr=%di len=+112(%di):u32 icmp_type=+0(+200(%di)):u8 icmp_code=+1(+200(%di)):u8 pkg_type=+128(%di):b3@0/8'`

![image-20210427003512670](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427003512670.png)

​	由于是发送给本机的数据包，所以`pkg_type`=`PACKET_HOST`，也就是0. 于是我们就成功的hook了其中的一些bit位

```
/**
 * 包的目的地址与收到它的网络设备的L2地址相等。换句话说，这个包是发给本机的。
 */
#define PACKET_HOST		0		/* To us		*/
```

​	关于更多的使用方法，请参考[kprobe官方文档](https://github.com/torvalds/linux/blob/master/Documentation/trace/kprobetrace.rst)。包括字符串、内存偏移、栈帧、地址、符号等。	

## 四、 设置filter

​	请参考这个[官方文档](https://github.com/torvalds/linux/blob/master/Documentation/trace/events.rst)的第五章，里面讲解了这些事件设置filter的方法。我简单举个例子，假设我只打印`ping`数据包大于`300`的数据包，于是我可以这样写：

​	`sudo ./kprobe  'p:icmp_rcv skb_addr=%di len=+112(%di):u32' 'len > 300'`

​	我先测试让`ping`数据包等于200 看下有输出没有：

​	![image-20210427113405066](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427113405066.png)

​	可以看到，当`ping`数据包小于`300`的时候就被`filter`掉了。是不是很神奇。![image-20210427113541660](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427113541660.png)

​    现在，我们来看下当`ping`数据包大于`300`的时候，是否有输出：

​	![image-20210427113818819](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427113818819.png)

​	看吧，数据包大于`300`的时候，成功输出了信息。![image-20210427113845671](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427113845671.png)

​	有了这个功能，排查系统问题就方便多了，比如看下`kfree`丢包，`kmalloc`申请的内存，这样排查很多问题就清晰多了。

## 五、打印调用栈

​	最后介绍一下调用栈，kprobe同样可以打印调用栈，以及指定`pid`等。`./kprobe --help`可以知道，加上-s就是打印出调用栈。

​	`sudo ./kprobe  -s 'p:icmp_rcv skb_addr=%di len=+112(%di):u32' 'len > 300'`， 输出如下：

​	![image-20210427114508273](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210427114508273.png)

​	是不是很方便，很强大。好好使用这个工具吧，据我目前工作来看，绝大部分linux设备都支持`kprobe`, 甚至是嵌入式设备也可以很方便的加上，而且在不使用这个工具下，几乎没有性能损耗。赶快用上吧。

## 六、参考文档

https://www.kernel.org/doc/Documentation/trace/kprobetrace.txt

https://events.static.linuxfound.org/slides/lfcs2010_hiramatsu.pdf

https://github.com/torvalds/linux/blob/master/Documentation/trace/events.rst

x64函数调用规则：

https://www.raywenderlich.com/615-assembly-register-calling-convention-tutorial

arm64 kprobe:

https://linux.cn/article-9098-1.html

https://www.linaro.org/blog/kprobes-event-tracing-armv8/#

arm kprobe例子以及传参：

https://blog.csdn.net/luckyapple1028/article/details/52972315

http://laoar.github.io/blogs/316/

x86:

https://en.wikipedia.org/wiki/X86_calling_conventions

https://docs.windriver.com/bundle/Wind_River_Linux_Tutorial_Dynamic_Kernel_Debugging_with_ftrace_and_kprobes_LTS_1/page/twb1552585245824.html

uprobe:

https://www.kernel.org/doc/Documentation/trace/uprobetracer.txt

移植BCC到ARM:

http://events17.linuxfoundation.org/sites/events/files/slides/ELC_2017_NA_dynamic_tracing_tools_on_arm_aarch64_platform.pdf