---
layout:     post
title:      ARM 32位栈帧浅析
subtitle:   ARM栈帧浅析
date:       2020-12-24
author:     Albert
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
   - Blog
   - arm
   - debug
---



# ARM 32位栈帧浅析

## 前言

​		为了在实际的开发项目过程中，在应用层发生段错误的时候，能够像内核kernel panic一样打印出错时候的栈回溯信息，我们首先应该对各类体系架构的栈帧相关知识进行研究。这样为后续实现在内核中打印应用层栈回溯功能打下一个基础。关于什么是堆栈，请参考[堆栈原理揭秘](https://zhuanlan.zhihu.com/p/142964520)。关于ARM 过程调用标准，请参考[Procedure Call Standard for the ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf)， 不习惯看英文的可以看下[中文翻译](https://blog.csdn.net/weixin_34277853/article/details/93185653?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control)，中文只能说可以快速理解一下，毕竟是母语。

​		理论上来说，ARM的15个通用寄存器是通用的，但实际上并非如此，特别是在过程调用的过程中。`PCS(Procedure Call Standard for Arm architecture)`就定义了过程调用中，寄存器的特殊用途。关于arm的知识请参考如下几篇文档以及文档中的扩展阅读：

   [ARM汇编语言](https://zhuanlan.zhihu.com/p/82490125)、[ARM汇编](https://azeria-labs.com/writing-arm-assembly-part-1/)。

​	   下面这张图可以作为一个参考。

![img](https://gitee.com/cclinuxer/blog_image/raw/master/image/cheatsheetv1.1-1920x1080.png)

## 一、栈帧概述

​		`stack`我们都知道，每一个进程都有自己的栈。考虑进程执行时发生函数调用的场景，母函数和子函数使用的是同一个栈，在通常的情况下，我们并不需要区分母函数和子函数分别使用了栈的哪个部分。但是，当我们需要在执行过程中对函数调用进行`backtrace`的时候，这一信息就很重要了。

​		简单的说，`stack frame`就是一个函数所使用的`stack`的一部分，所有函数的`stack frame`串起来就组成了一个完整的栈。`stack frame`的两个边界分别由`FP`（栈基址寄存器：`r11`）和`SP`(栈顶寄存器)来限定。**FP和SP之间所包含的区域就是函数的栈帧。**

​		我在网上找了一幅图，这幅图也是目前网络上出现最多的，但是下面这幅图是描述`x86`的，在研究`arm`的汇编代码的时候，很多压栈操作和下面这幅图是**对应不上的**，所以在学习`arm`汇编或者是栈帧的时候，不要在细节上对这幅图陷入过多，因为arm体系架构的压栈、出栈操作可能和下面的图存在较大差距。但是思想都是一样的，栈存在的意义就是CPU的寄存器的数量是有限的，但是我们在函数调用的过程中，每一级函数调用都存在很多临时变量，需要保存在栈区。而每一个函数的栈区域由`FP`和`SP`之间的区域限定。

​       下图和arm体系架构**是对应不上的**，我之所以把图放在这里是为了避免大家或者我自己去查找资料的时候陷入下面这幅图。我在第二章会通过汇编代码一步一步讲述ARM的压栈和出栈操作。

[![img](http://blog.chinaunix.net/attachment/201109/30/25871104_1317397448ARNz.png)](http://blog.chinaunix.net/attachment/201109/30/25871104_1317397448ARNz.png)



## 二、一个简单的例子

​		下面我们以一个简单的C程序代码来跟踪arm的函数调用过程中的压栈以及出栈过程，C代码如下：

```c
/*************************************************************************
	> File Name: arm_call_test.c
	> Author: Albert Jie
	> Mail: huangjieajy@163.com 
	> Created Time: 2020年12月24日 星期四 17时48分09秒
 ************************************************************************/

int add(int a, int b)
{
	int c=a+b;
	return c;
}

int main()
{
	int a = 3;
	int b = 2;
	int c = 0;
	c = add(a, b);
	return c;
}
```

​		用如下指令进行编译，其中`gcc`用于需要用你想要观察栈回溯的体系架构，路径可能和下面的不一样：

> `/opt/toolchain-arm_cortex-a7_gcc-5.2.0_musl-1.1.16_eabi/bin/arm-openwrt-linux-gcc  -o arm_call  ./arm_call_test.c`

​		由于我们需要分析arm体系架构的栈回溯流程，那么我们需要进一步进行 获取程序的汇编代码:

> `/opt/toolchain-arm_cortex-a7_gcc-5.2.0_musl-1.1.16_eabi/bin/arm-openwrt-linux-objdump  -D arm_call > arm_call.s`

 `arm_call.s`中就是该程序的汇编代码了，我们对此进行详细的分析，并以图形的方式来阐述这个过程

```assembly
0001042c <add>:
   1042c:	e52db004 	push	{fp}		; (str fp, [sp, #-4]!)  将fp的内容存放到sp-4的位置，并将sp = sp - 4(栈向下增长)
   10430:	e28db000 	add	fp, sp, #0      ; 将fp=sp. 这个时候fp = sp. 也就是开始下一个栈帧了（fp和sp之间的区域是栈帧）
   10434:	e24dd014 	sub	sp, sp, #20     ；将sp = sp - 20
   10438:	e50b0010 	str	r0, [fp, #-16]  ; ro放到fp - 16 位置的地方
   1043c:	e50b1014 	str	r1, [fp, #-20]  ； r1的值放到fp - 20的地方
   10440:	e51b2010 	ldr	r2, [fp, #-16]  ； 将fp-16内存存放的参数r2寄存器
   10444:	e51b3014 	ldr	r3, [fp, #-20]  ； 将fp-20内存存放的参数r3寄存器
   10448:	e0823003 	add	r3, r2, r3      ; 将参数相加
   1044c:	e50b3008 	str	r3, [fp, #-8]
   10450:	e51b3008 	ldr	r3, [fp, #-8]
   10454:	e1a00003 	mov	r0, r3          ;将计算出来的C值存放到r0寄存器
   10458:	e24bd000 	sub	sp, fp, #0      ；sp = fp - 0
   1045c:	e49db004 	pop	{fp}		; (ldr fp, [sp], #4)，将fp = *sp, sp = sp + 4
   10460:	e12fff1e 	bx	lr          ; 跳转回lr寄存器的值

00010464 <main>:
   10464:	e92d4800 	push	{fp, lr} ;/* 序幕开始：保存帧指针和返回地址到堆栈, push是压栈操作*/
   10468:	e28db004 	add	fp, sp, #4   ；/* fp指针的值等于fp = sp + 4 */
   1046c:	e24dd010 	sub	sp, sp, #16  ；/* 将sp的内容指向向下移动16个字节 */
   10470:	e3a03003 	mov	r3, #3       ；/* 将数值3移动到r3寄存器 */
   10474:	e50b3008 	str	r3, [fp, #-8]；/* 将r3的值放到fp指向的地址，向下偏移8个字节处 */
   10478:	e3a03002 	mov	r3, #2         ；/* 将2的值放到r3寄存器 */
   1047c:	e50b300c 	str	r3, [fp, #-12] ;/* 将r3的寄存器值，放到fp向下偏移12个字节处 */
   10480:	e3a03000 	mov	r3, #0         ;/* 将r3寄存器设置为0 */
   10484:	e50b3010 	str	r3, [fp, #-16] ；/* 将c的值放到fp向下偏移16个字节处 */
   10488:	e51b100c 	ldr	r1, [fp, #-12] ；/* 将b的值放到r1 */
   1048c:	e51b0008 	ldr	r0, [fp, #-8]  ；/* 将a的值放到r0 */
   10490:	ebffffe5 	bl	1042c <add>     ;/* 函数跳转到 add函数，这里做了几件事：BL操作将PC压栈、将下一条指令放到LR寄存器，然后跳转到函数处（将PC设置为1042c sub*/
   10494:	e50b0010 	str	r0, [fp, #-16]
   10498:	e51b3010 	ldr	r3, [fp, #-16]
   1049c:	e1a00003 	mov	r0, r3
   104a0:	e24bd004 	sub	sp, fp, #4
   104a4:	e8bd8800 	pop	{fp, pc}

```

接下来我们根据汇编代码一步一步来解析`ARM 32`位系统的压栈以及出栈过程。我们逐步分析每一行汇编代码后面的含义，并且**写出临时寄存器的值的变化以及栈帧的变化**

#### 2.1  第1步

​	假设我们在执行`main`函数第一条指令之前，**最开始**的`FP`和`SP`如下图所示：

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210104150010603.png" alt="image-20210104150010603" style="zoom:33%;" />

#### 2.2 第2步  

​	 执行如下指令后，栈的变化如图二所示：

```assembly
 10464:	e92d4800 	push	{fp, lr} ;/* 序幕开始：保存帧指针和返回地址到堆栈, push是压栈操作*/
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105100023211.png" alt="image-20210105100023211" style="zoom:33%;" />

#### 2.3 第3步

 

```assembly
  10468:	e28db004 	add	fp, sp, #4   ；/* fp指针的值等于fp = sp + 4 */
```

以上指令执行后，程序栈变化如下：

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105150859729.png" alt="image-20210105150859729" style="zoom:33%;" />

#### 2.4  第4步

```assembly
  1046c:	e24dd010 	sub	sp, sp, #16  ；/* 将sp的内容指向向下移动16个字节 */
```

​     以上指令执行后，程序栈变化如下：

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105151105119.png" alt="image-20210105151105119" style="zoom:33%;" />

#### 2.5  第5步

```assembly
   10470:	e3a03003 	mov	r3, #3       ；/* 将数值3移动到r3寄存器 */
   10474:	e50b3008 	str	r3, [fp, #-8]；/* 将r3的值放到fp指向的地址，向下偏移8个字节处 */
   10478:	e3a03002 	mov	r3, #2         ；/* 将2的值放到r3寄存器 */
   1047c:	e50b300c 	str	r3, [fp, #-12] ;/* 将r3的寄存器值，放到fp向下偏移12个字节处 */
   10480:	e3a03000 	mov	r3, #0         ;/* 将r3寄存器设置为0 */
   10484:	e50b3010 	str	r3, [fp, #-16] ；/* 将c的值放到fp向下偏移16个字节处 */
   10488:	e51b100c 	ldr	r1, [fp, #-12] ；/* 将b的值放到r1 */
   1048c:	e51b0008 	ldr	r0, [fp, #-8]  ；/* 将a的值放到r0 */
```

 上面几行代码执行完毕后：

`r3`寄存器的值为`0`， `r1`的寄存器值为`2`，`r0`寄存器的值为`3`. 程序栈如下图所示：

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105151445162.png" alt="image-20210105151445162" style="zoom:33%;" />

#### 2.6 第6步

```assembly
  10490:	ebffffe5 	bl	1042c <add>     ;/* 函数跳转到 add函数，这里做了几件事：BL将下一条指令放到LR寄存器，然后跳转到函数处（将PC设置为1042c sub*/
```

​		以上指令会设置`PC`寄存器的值为`1042c`，也就是跳转到add函数处，并且保存了`10494`这个指令地址到`LR`寄存器，后续**返回函数**就直接将`PC`设置为`LR`寄存器的值就完成了函数**回跳**。

​		`pc`寄存器的值设置为了`1042c`，这个时候就**跳转**到了`add`函数执行：`add`代码的汇编如下:

```assembly
0001042c <add>:
   1042c:	e52db004 	push	{fp}		; (str fp, [sp, #-4]!)  将fp的内容存放到sp-4的位置，并将sp = sp - 4(栈向下增长)
   10430:	e28db000 	add	fp, sp, #0      ; 将fp=sp. 这个时候fp = sp. 也就是开始下一个栈帧了（fp和sp之间的区域是栈帧）
   10434:	e24dd014 	sub	sp, sp, #20     ；将sp = sp - 20
   10438:	e50b0010 	str	r0, [fp, #-16]  ; ro放到fp - 16 位置的地方
   1043c:	e50b1014 	str	r1, [fp, #-20]  ； r1的值放到fp - 20的地方
   10440:	e51b2010 	ldr	r2, [fp, #-16]  ； 将fp-16内存存放的参数r2寄存器
   10444:	e51b3014 	ldr	r3, [fp, #-20]  ； 将fp-20内存存放的参数r3寄存器
   10448:	e0823003 	add	r3, r2, r3      ; 将参数相加
   1044c:	e50b3008 	str	r3, [fp, #-8]
   10450:	e51b3008 	ldr	r3, [fp, #-8]
   10454:	e1a00003 	mov	r0, r3          ;将计算出来的C值存放到r0寄存器
   10458:	e24bd000 	sub	sp, fp, #0      ；sp = fp - 0
   1045c:	e49db004 	pop	{fp}		; (ldr fp, [sp], #4)，将fp = *sp, sp = sp + 4
   10460:	e12fff1e 	bx	lr          ; 跳转回lr寄存器的值
```

#### 2.7  第7步

`add`函数执行第一行代码后程序的栈变化如下图：

```assembly
1042c:	e52db004 	push	{fp}		; (str fp, [sp, #-4]!)  将fp的内容存放到sp-4的位置，并将sp = sp - 4(栈向下增长)
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105164155287.png" alt="image-20210105164155287" style="zoom:33%;" />

#### 2.8 第8步

 执行如下代码后，函数的栈变化如下：

```assembly
  10430:	e28db000 	add	fp, sp, #0      ; 将fp=sp. 这个时候fp = sp. 也就是开始下一个栈帧了（fp和sp之间的区域是栈帧）
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105164452697.png" alt="image-20210105164452697" style="zoom:33%;" />



#### 2.9 第9步

  执行如下代码后：栈又发生了变化：

```assembly
 10434:	e24dd014 	sub	sp, sp, #20     ；将sp = sp - 20
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105164749194.png" alt="image-20210105164749194" style="zoom:33%;" />

#### 2.10 第10步

​	记住前面在`main`函数中，已经将参数传到`arm`临时寄存器里面了。`r3`寄存器的值为`0`， `r1`的寄存器值为`2`，`r0`寄存器的值为`3`. 执行如下指令后程序栈如下图所示：执行完毕后，返回值`5`存放在`r0`寄存器中，作为`add`函数返回值。后续在`main`函数中获取返回值就直接去`r0`寄存器中取值。

```assembly
   10438:	e50b0010 	str	r0, [fp, #-16]  ; ro放到fp - 16 位置的地方
   1043c:	e50b1014 	str	r1, [fp, #-20]  ； r1的值放到fp - 20的地方
   10440:	e51b2010 	ldr	r2, [fp, #-16]  ； 将fp-16内存存放的参数r2寄存器
   10444:	e51b3014 	ldr	r3, [fp, #-20]  ； 将fp-20内存存放的参数r3寄存器
   10448:	e0823003 	add	r3, r2, r3      ; 将参数相加
   1044c:	e50b3008 	str	r3, [fp, #-8]
   10450:	e51b3008 	ldr	r3, [fp, #-8]
   10454:	e1a00003 	mov	r0, r3          ;将计算出来的C值存放到r0寄存器，也就是r0的值是5
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105165411923.png" alt="image-20210105165411923" style="zoom: 33%;" />

#### 2.11 第11步

执行如下指令后，函数栈帧变化如下图所示：

```assembly
   10458:	e24bd000 	sub	sp, fp, #0      ；sp = fp - 0
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105165639136.png" alt="image-20210105165639136" style="zoom:33%;" />



#### 2.12  第12步

执行如下指令后：栈帧变化：然后跳回`lr`寄存器里面存放的地址，也就是去执行`10494:	e50b0010 	str	r0, [fp, #-16]`这条代码。

```assembly
   1045c:	e49db004 	pop	{fp}		; (ldr fp, [sp], #4)，将fp = *sp, sp = sp + 4
   10460:	e12fff1e 	bx	lr          ; 跳转回lr寄存器的值
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105165829429.png" alt="image-20210105165829429" style="zoom:33%;" />



#### 2.13  第13步

为了方便查看汇编代码，我们再次贴一下`main`函数的汇编指令：

```c
00010464 <main>:
   10464:	e92d4800 	push	{fp, lr} ;/* 序幕开始：保存帧指针和返回地址到堆栈, push是压栈操作*/
   10468:	e28db004 	add	fp, sp, #4   ；/* fp指针的值等于fp = sp + 4 */
   1046c:	e24dd010 	sub	sp, sp, #16  ；/* 将sp的内容指向向下移动16个字节 */
   10470:	e3a03003 	mov	r3, #3       ；/* 将数值3移动到r3寄存器 */
   10474:	e50b3008 	str	r3, [fp, #-8]；/* 将r3的值放到fp指向的地址，向下偏移8个字节处 */
   10478:	e3a03002 	mov	r3, #2         ；/* 将2的值放到r3寄存器 */
   1047c:	e50b300c 	str	r3, [fp, #-12] ;/* 将r3的寄存器值，放到fp向下偏移12个字节处 */
   10480:	e3a03000 	mov	r3, #0         ;/* 将r3寄存器设置为0 */
   10484:	e50b3010 	str	r3, [fp, #-16] ；/* 将c的值放到fp向下偏移16个字节处 */
   10488:	e51b100c 	ldr	r1, [fp, #-12] ；/* 将b的值放到r1 */
   1048c:	e51b0008 	ldr	r0, [fp, #-8]  ；/* 将a的值放到r0 */
   10490:	ebffffe5 	bl	1042c <add>     ;/* 函数跳转到 add函数，这里做了几件事：BL操作将PC压栈、将下一条指令放到LR寄存器，然后跳转到函数处（将PC设置为1042c sub*/
   10494:	e50b0010 	str	r0, [fp, #-16]
   10498:	e51b3010 	ldr	r3, [fp, #-16]
   1049c:	e1a00003 	mov	r0, r3
   104a0:	e24bd004 	sub	sp, fp, #4
   104a4:	e8bd8800 	pop	{fp, pc}
```

由前面提到的内容可以知道，从`add`函数返回后，应该执行如下指令，由**第10步**执行结果可以知道，`r0`存放的是`add`的返回值`5`：

```assembly
10494:	e50b0010 	str	r0, [fp, #-16] ；将r0寄存器的值放到fp-16处
10498:	e51b3010 	ldr	r3, [fp, #-16] ；将fp-16处的值放回到r3
1049c:	e1a00003 	mov	r0, r3;将r3的值放到r3. 感觉这几条汇编有点冗余。
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105170352147.png" alt="image-20210105170352147" style="zoom:33%;" />

#### 2.14 第14步

```assembly
   104a0:	e24bd004 	sub	sp, fp, #4
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105170517688.png" alt="image-20210105170517688" style="zoom:33%;" />

#### 2.15  第15步

PC寄存器在执行下面的汇编指令后，他的值又变为调用main函数的函数指令的下一条指令了。r0存放了函数的返回值。

```assembly
   104a4:	e8bd8800 	pop	{fp, pc} ;现在栈布局又回到调用main函数的状态了。
```

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105170718668.png" alt="image-20210105170718668" style="zoom:33%;" />

## 三、栈回溯`backtrace`

​		在程序执行过程中（通常是发生了某种意外情况而需要进行调试），通过`SP`和`FP`所限定的`stack frame`，就可以得到母函数的`SP`和`FP`，从而得到母函数的`stack frame`（`PC，LR，SP，FP`会在函数调用的第一时间压栈），以此追溯，即可得到所有函数的调用栈，我们将这些栈帧的关键信息打印出来就形成了栈回溯信息。 

<img src="https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210105165411923.png" alt="image-20210105165411923" style="zoom: 33%;" />

​		对于`arm`来说，以上图为例，一般来说最后一级的函数调用的**返回地址**是存在**LR寄存器**里面的，`SP和FP`也是寄存器，如果我想要到上级函数的栈帧，只需要让`SP=FP`.  `arm`系统架构`F`P指针指向的内存存放的值是上一个函数`FP`所在的内存地址。而返回地址如果不是最后一级函数，那么返回地址是放在`FP-4`这个位置的**（也就是一个栈帧总是以上一个函数的返回地址开始的）**，相关知识请参考：[functions-and-the-stack](https://azeria-labs.com/functions-and-the-stack-part-7/)。所以我们可以通过`FP、SP、`以及`LR`层层递归，从而找到函数的异常调用栈。



## 参考文档

[《C语言在ARM中函数调用时，栈是如何变化的？》](https://cloud.tencent.com/developer/article/1593645)

[《函数调用过程中栈到底是怎么压入和弹出的？》](https://www.zhihu.com/question/22444939)

https://manybutfinite.com/post/journey-to-the-stack/

https://azeria-labs.com/functions-and-the-stack-part-7/

