

# 利用 SysRq 键排除和诊断系统故障

罗 布 和 张 冬
2009 年 8 月 06 日发布



原文地址：

https://www.ibm.com/developerworks/cn/linux/l-cn-sysrq/



## SysRq 是什么

你是否遇到服务器不能通过 SSH 登录，也不能通过本地终端登录，但是却能 ping 通，数字键盘锁还可以响应击键操作的情况？在这种情况下，你除了按下电源或复位键之外，还做过什么吗？你是否想过这种情况是可能恢复的呢？你是否想过收集更多的信息来定位这次系统挂起的原因呢？上述情况，可称之为“可中断的系统挂起”。换句话来讲，系统因为某种原因已经停止对大部分正常服务的响应，但是系统仍然可以响应键盘的按键中断请求。在这种情况下，一组称为 SysRq 的按键组合将发挥它的神奇作用。

SysRq 经常被称为 Magic System Request，它被定义为一系列按键组合。之所以说它神奇，是因为它在系统挂起，大多数服务已无法响应的情况下，还能通过按键组合来完成一系列预先定义的系统操作。通过它，不但可以在保证磁盘数据安全的情况下重启一台挂起的服务器，避免数据丢失和重启后长时间的文件系统检查，还可以收集包括系统内存使用，CPU 任务处理，进程运行状态等系统运行信息，甚至还可能在无需重启的情况下挽回一台已经停止响应的服务器。

## 启用 SysRq

### 内核的支持

要启用 SysRq 功能，首先必须确保内核已经加入 CONFIG_MAGIC_SYSRQ 支持。在现今 Linux 发行版中，无一例外的均已加入该功能的支持，验证如下：

```
# grep “ CONFIG_MAGIC_SYSRQ ” /boot/config-`uname – r` `` ``CONFIG_MAGIC_SYSRQ=y
```

### 通过 sysctl 启用

SysRq 功能默认在 RHEL5u2 上是禁用的。可以通过 proc 文件系统来启用它。使用 sysctl 命令启用它，并通过 /proc 来检查其可用性。

```
# sysctl -w kernel.sysrq=1 `` ``kernel.sysrq = 1 `` ``# cat /proc/sys/kernel/sysrq `` ``1
```

kernel.sysrq 还可接受除 0 和 1 以外的启用参数，详情请参考 sysrq 内核文档。

### 保持重启后生效

通过把” kernel.sysrq = 1 ”设置到 /etc/sysctl.conf 中，可以使 SysRq 在下次系统重启后仍然生效。请注意，在非 RHEL 系统中，也许需要通过其它的配置文件来使之重启后生效。

## 使用 SysRq

网上有道题，问在只有 shell，init、halt、shutdown 等命令都不工作的情况下如何重启系统。答案就是 SysRq，通过 SysRq – B 来完成系统的重启。

早期的 SysRq 只支持键盘操作。要使用 SysRq，必须直接对主机进行键盘操作。要想执行 SysRq-B 来重启系统，只能通过直接键盘操作 Alt – SysRq – B 来完成（这里的 B 仅指 B 按键，不区分大小写）。 kernel 2.5.64 上的一个 patch 增加了 /proc/sysrq-trigger 接口，使得用户可能通过 /proc 接口来进行 SysRq 操作，换而言之，在现今大部分构建在 2.6 内核上的发行版，对主机键盘的物理接触已经不再是 SysRq 的必要条件。用户只需要登录到系统上，就可以直接使用 echo “ b ” > /proc/sysrq-trigger 来重启系统。在下文中，为描述的简洁，SysRq-<?> 均代表 Alt-SysRq-<?> 或者 echo “ ? ” > /proc/sysrq-trigger 。

众所周知，系统挂起的很多时候 ssh 登录也未必响应，在缺乏对主机物理操作条件下，/proc/sysrq-trigger 也因为无法获取登录 shell 而无法操作。于是出现了一个名为 sysrqd 的开源项目，它允许通过网络来直接来触发 SysRq 。该程序只有 300 行左右代码，监听 TCP 端口 4094，通过自定义密码验证过后，即可对 /proc/sysrq-trigger 进行操作。但是由于此程序在用户空间实现，在系统挂起时该程序的可用性，以及其安全性均受到广泛质疑。其实如果这个服务做到内核空间，以类似响应 ARP 形式进行处理，再加上合理的认证方式，或许在大多数系统挂起的时候可以起到更加实际的作用。当然，在现代服务器的远程管理模块日趋先进的前提下，是否能通过网络来触发 SysRq 好像并不是那么重要。

## 查看 SysRq 输出

SysRq 并不只能重启服务器，如果只是这样，那我们也不需要查看它的输出了。很多时候，我们使用 SysRq 是为了查看服务器的 CPU，内存或进程信息。 SysRq 默认会输出到本地控制台终端，并写入 syslog，但这并不是个好主意。因为对于大多数系统挂起的情况，我们已经无法再访问这两个信息源，在无法判定系统故障的情况下，只好无奈的使用后面即将介绍的 R-E-I-S-U-B 序列来重启服务器。接下来将介绍 SysRq 输出的几种方法。只有最大程度的获取到 SysRq 输出，才能更好的利用 SysRq 提供的其它功能。

### 输出到本地终端

SysRq 默认会根据 console_loglevel 输出到本地终端。只要 console_loglevel 大于 default_message_loglevel，SysRq 信息就会输出到本地控制台终端。它在测试的时候都好用，但一到关键时刻，系统挂起无法切换，大量输出造成滚屏，以及信息无法记录等问题随之而来。

### 输出到 syslog

根据 syslog 的默认配置，SysRq 默认会记录到 /var/log/messages，并且这里记录的信息与 console_loglevel 无关，基本是完整的。但是由于负责记录日志的 syslogd 本身也是一个用户进程，在执行后面即将介绍的 SysRq-E, SysRq-I 时也会被终结，这就意味着 syslog 记录的信息在一定情况下将不再完整。同时由于系统挂起时查看 syslog 日志本身就是一件难上加难的事，这里记录的信息往往只能用在系统恢复过后的故障分析，对故障发生时的实时诊断并没有太大的帮助。

### 通过 netconsole 输出

利用 netconsole 功能，可以获得一个通过远程 syslog 服务器输出的显示终端，服务器的 SysRq 输出将被远程的 syslog 服务器捕获。在服务器挂起，无法查看 syslog 日志，同时也无法切换到本地控制台终端的时候，这种形式的输出就显得举足轻重。与本地终端类似，只要 console_loglevel 大于 default_message_loglevel，SysRq 信息就会通过 netconsole 输出到远程。它在大多数情况都生效，哪怕是内核网络部分出现问题。因为 netconsole 的网络发送部分是独立存在的，并不依赖于网卡驱动程序。

### 输出到串口终端

要想通过串口获取 SysRq 输出，首先需要在 grub 的 kernel 行添加类似 ” console=ttyS0,115200 ” 的串口输出配置，重启服务器以启用内核串口输出。然后从另一台主机上用串口线连接服务器，并用 minicom 等程序捕获其输出。这是一种通常的使用方式。然而利用 Serial over IP 产品，管理员无需现身嘈杂的机房就能通过网络获得服务器的串口输出，查看并截取字符形式的输出。这是相对现代的使用方式。通过这两种方式，我们都可以方便的获取到 SysRq 在串口上输出。

## SysRq 功能键组合

### 安全重启系统

到目前为止，我们可见到的大多数 SysRq 推荐用法都是系统挂起后的安全重启，用此方法来避免数据丢失。这个 SysRq 序列是 R-E-I-S-U-B 。要知道，该序列早在 SysRq 首次于 Linux 实现的 2.1.43 内核中就存在了。它基本等价于 reboot 命令，会依次停止系统上运行的进程，回写磁盘缓冲区，再安全的重启系统。需要注意的是，E 会向除 init 以外所有进程发送可捕获的 SIGTERM 信号，这就意味着程序可能需要一定时间来进行结束进程前的善后处理，视系统负载和任务数量，这个时间可能会达到几十秒。 I 发送的是不可捕获的 SIGKILL 信号，相对而言没有更多的延迟。同时，S 和 U 这两个动作均与磁盘相关。当系统具有一定负载时，这两个动作均不会立即完成，而是需要一定的时间，通常为几秒钟。所以，R-E-I-S-U-B 这个序列的推荐使用方式是：R – 1 秒 – E – 30 秒 – I – 10 秒 – S – 5 秒 – U – 5 秒 – B，而不是一气呵成地按下这六个键，试想一次正常的 reboot 命令也不是在一瞬间完成的吧。

下面列出各个序列的示例输出及简单说明：

**R - 把键盘设置为 ASCII 模式**

```
SysRq: Keyboard mode set to XLATE
```

有关键盘工作模式，请参考资料中的 [kbd_mode](http://linux.die.net/man/1/kbd_mode) 手册。

**E - 向除 init 以外所有进程发送 SIGTERM 信号**

```
SysRq: Terminate All Tasks
```

因为 syslogd 本身也被结束，所以 SysRq 也许不会被记录下来。但是查看 /var/log/messages 会看到类似下面的消息：

```
exiting on signal 15(SIGTERM)
```

**I - 向除 init 以外所有进程发送 SIGKILL 信号**

```
SysRq: Kill All Tasks
```

与 E 类似，因为 syslogd 本身也被结束，除非 netconsole 或串口记录已打开，否则连上面的信息都无法捕捉。同时，因为 SIGKILL 是不可捕获的信号，/var/log/messages 里面也不会留下任何线索。

**S - 磁盘缓冲区同步**

```
SysRq : Emergency Sync `` ``Emergency Sync complete
```

该操作会把磁盘缓冲区的数据回写，以防止数据丢失，通常会有一定延时。在能看到输出的情况下，请等到 ” Emergency Sync complete ” 过后再继续后续操作。否则，等十秒钟左右，再进行后续 SysRq 操作。

**U - 重新挂载为只读模式**

```
SysRq : Emergency Remount R/O `` ``Emergency Remount complete
```

该操作会把磁盘重挂载为只读模式，以防止数据的损坏。与 S 类似，该操作通常也有一定延时。请等到 ” Emergency Remount complete ” 出现过后再进行后续操作，或者等候十秒钟再进行后续 SysRq 操作。

**B - 立即重启系统**

```
SysRq: Resetting
```

该操作会立即重启系统，比想象中要快。

### 恢复系统挂起

仅从系统挂起后安全重启来看，R-E-I-S-U-B 序列似乎已经令人满意了。但对于一些挂起，只是因为一部分进程消耗过多 CPU，内存等系统资源引起的。对于这样的情形，可以通过结束“凶手”进程来恢复已经挂起的系统。

笔者曾亲历 Acrobat Reader 在 Firefox 中内存泄露引起的系统挂起。每在 Firefox 中浏览完成一个 PDF 后，acroread 进程不会退出，相反其内存使用量逐步攀升，在一段时间内消耗完系统的内存和 swap，系统持续 swapping，使系统进入挂起状态，不响应桌面，鼠标，以及所有的应用程序。

笔者还经历一例屏保程序引起的系统挂起。锁定一段时间后，键盘鼠标停止响应，无法切换到本地控制台，远程登录正常，查看内存使用正常，CPU 被屏保程序吃尽。

SysRq 定义了一组与结束进程相关的序列：E-I-K-F，我们可以用它们来恢复系统挂起。

E 和 I 已经介绍过，他们都会结束除 init 以外的所有进程，理所当然都可以恢复类似的系统挂起。笔者在早期也是这样做的。但是这种方法似乎过于暴力，恢复过后的系统基本上除了 uptime 是连续的，数据未损坏以外，余下的状态并不比重启一次系统好。因为所有的服务都已中止，还需手工干预才能恢复正常。笔者的经验，经过 E-I 恢复的系统，如不是时间紧迫，即使系统已经恢复响应，最好继续完成 S-U-B 操作。因为对于一些复杂业务系统，难免因为人为因素造成某些服务忘记启动而埋下日后的定时炸弹。

K 和 F 正是它们的补充。它们仅结束符合特定条件的进程。 K 只结束与当前控制台相关的进程组。 K 代表 saK，saK 的全称为 Secure Access Key 。从字面意思看似乎有些深奥，它的本意是出于安全考虑，为了杀掉类似木马一类套取密码的伪登录程序，让 init 重新启动正在的 getty 登录界面。但在实际应用过程中，特别在 X 窗口下某些程序内存泄露或其它异常行为引起系统挂起时，就像上面两个例子，可以相当准确而快捷的使系统恢复正常。在理解 SysRq-K 之前，笔者曾一度使用 SysRq-E 来解决问题，但随之而来的系统服务恢复则成了一大难题。 F 则利用 OOM-Killer 选取一个进程然后结束之。这对于内存问题引起的挂起可以起到比 SysRq-K 更加准确的作用。但是有些时候 OOMKiller 也会误判而杀掉一些长时间运行的后台服务，引起一些不必要的麻烦。

下面列出各个序列的示例输出及简单说明：

**E - 向所有进程发送 SIGTERM 信号**

```
SysRq: Terminate All Tasks
```

**I - 向所有进程发送 SIGKILL 信号**

```
SysRq: Kill All Tasks
```

**K - 结束与当前控制台相关的全部进程**

```
SysRq : SAK `` ``SAK: killed process 7254 (top): p->signal->session==tty->session `` ``SAK: killed process 7223 (bash): p->signal->session==tty->session
```

该操作结束了文本控制台下正在运行的 top 程序，以及登录的 shell 。

**F - 人为触发 OOM Killer**

```
SysRq : Manual OOM execution `` ``events/0 invoked oom-killer: gfp_mask=0xd0, order=0, oomkilladj=0 `` ``[<``c04584e9``>] out_of_memory+0x72/0x1a5 `` ``[<``c053a2eb``>] moom_callback+0x13/0x15 `` ``[<``c04331de``>] run_workqueue+0x78/0xb5 `` ``[<``c053a2d8``>] moom_callback+0x0/0x15 `` ``[<``c0433a92``>] worker_thread+0xd9/0x10b `` ``[<``c042027b``>] default_wake_function+0x0/0xc `` ``[<``c04339b9``>] worker_thread+0x0/0x10b `` ``[<``c0435ea1``>] kthread+0xc0/0xeb `` ``[<``c0435de1``>] kthread+0x0/0xeb `` ``[<``c0405c3b``>] kernel_thread_helper+0x7/0x10 `` ``======================= `` ``Mem-info: ``…… (snip) ……`` ``Out of memory: Killed process 4860 (xfs).
```

该操作人为触发 OOM Killer，OOM Killer 将根据各进程的内存处理情况选取最合适的“凶手”进程，并向其发送 SIGKILL 信号，中止其运行。 SysRq 输出包括运行栈，内存使用信息，和“凶手”进程的标识信息。在此例中 PID 为 4860 的 xfs 进程被中止。在实际情况中，除非可以确认是内存使用问题，尽量避免使用这个组合键。因为 OOM-Killer 自动挑选的进程不一定是真正的“凶手”。相比之下，SysRq-K 结束的进程更有针对性，特别是对 X 桌面下程序引起的系统挂起。由于桌面下启动的程序多数为非关键应用，结束并重启它们在大多数情况下并不会对系统造成太多影响。

考虑篇幅关系，省略了内存情况的输出，因为这部分与 SysRq-M 输出一致。

### 获取系统信息

SysRq 提供了 M-P-T-W 序列，在恢复系统挂起之前，这是一个推荐执行的序列。它会记录下当前系统的内存使用情况，当前 CPU 寄存器的状态，进程运行状态，以及所有 CPU 及寄存器的状态。通过这些信息，可以对挂起的原因做粗略的分析，然后结合之前介绍的 E-I-K-F 序列对症下药进行恢复性操作，或许还可以即时恢复一部分已经挂起的系统，而不是每遇到系统挂起就盲目的按电源，或机械地操作 R-E-I-S-U-B 序列。就算不能找到原因并成功恢复，也将会为以后的故障分析留下宝贵的证据。要知道，能通过 syslog 找出原因的系统挂起少之又少。

下面列出各个序列的示例输出及简单说明：

**M - 打印内存使用信息**

```
SysRq : Show Memory `` ``Mem-info: `` ``DMA per-cpu: `` ``cpu 0 hot: high 0, batch 1 used:0 `` ``cpu 0 cold: high 0, batch 1 used:0 `` ``DMA32 per-cpu: empty `` ``Normal per-cpu: `` ``cpu 0 hot: high 186, batch 31 used:12 `` ``cpu 0 cold: high 62, batch 15 used:13 `` ``HighMem per-cpu: empty `` ``Free pages:    94460kB (0kB HighMem) `` ``Active:34941 inactive:64962 dirty:1 writeback: 0 unstable:0 free:23615 slab:3755 ``       ``mapped-file:2075 mapped-anon:7412 pagetables:3 26 `` ``DMA free:12016kB min:88kB low:108kB high:132kB active:16kB inactive:0kB ``        ``present:16384kB pages_scanned:0 all_unreclaimable? no ``        ``lowmem_reserve[]: 0 0 496 496 `` ``DMA32 free:0kB min:0kB low:0kB high:0kB active :0kB inactive:0kB ``      ``present:0kB pages_scanned:0 all_unreclaimable? no ``      ``lowmem_reserve[]: 0 0 496 496 `` ``Normal free:82444kB min:2804kB low:3504kB high :4204kB active:139748kB ``      ``inactive:259848kB present:507904kB pages_scanned:0 all_u nreclaimable? no ``      ``lowmem_reserve[]: 0 0 0 0 `` ``HighMem free:0kB min:128kB low:128kB high:128k B active:0kB inactive:0kB ``            ``present:0kB pages_scanned:0 all_unreclaimable? no ``            ``lowmem_reserve[]: 0 0 0 0 `` ``DMA: 38*4kB 37*8kB 33*16kB 29*32kB 24*64kB 17* 128kB 9*256kB ``      ``4*512kB 0*1024kB 1*2048kB 0*4096kB = 12016kB `` ``DMA32: empty `` ``Normal: 1*4kB 1*8kB 124*16kB 110*32kB 54*64kB 52*128kB 103*256kB ``       ``31*512kB 20*1024kB 2*2048kB 0*4096kB = 82444kB `` ``HighMem: empty `` ``92491 pagecache pages `` ``Swap cache: add 0, delete 0, find 0/0, race 0+ 0 `` ``Free swap = 1048568kB `` ``Total swap = 1048568kB `` ``Free swap:    1048568kB `` ``131072 pages of RAM `` ``0 pages of HIGHMEM `` ``2263 reserved pages `` ``26019 pages shared `` ``0 pages swap cached `` ``1 pages dirty `` ``0 pages writeback `` ``2075 pages mapped `` ``3755 pages slab `` ``326 pages pagetables
```

该操作显示了 cpu 相关分区信息，全局页使用情况，分区页使用情况，分区 slab 使用情况，页缓存使用情况，swap 使用情况等等。

**P - 打印当前 CPU 寄存器信息**

```
SysRq : Show Regs ` ` ``Pid: 0, comm:       swapper `` ``EIP: 0060:[<``c042a771``>] CPU: 0 `` ``EIP is at __do_softirq+0x51/0xbb `` ``EFLAGS: 00000286  Not tainted (2.6.18-92.el5 #1) `` ``EAX: c0722380 EBX: 00000002 ECX: c072b000 EDX: 00ce1e00 `` ``ESI: c06ddb00 EDI: 0000000a EBP: 00000000 DS: 007b ES: 007b `` ``CR0: 8005003b CR2: b7f85000 CR3: 14d6f000 CR4: 000006d0 `` ``[<``c0407461``>] do_softirq+0x52/0x9d `` ``[<``c04059bf``>] apic_timer_interrupt+0x1f/0x24 `` ``[<``c0403b98``>] default_idle+0x0/0x59 `` ``[<``c0403bc9``>] default_idle+0x31/0x59 `` ``[<``c0403c90``>] cpu_idle+0x9f/0xb9 `` ``[<``c06ec9ee``>] start_kernel+0x379/0x380 `` ``=======================
```

该操作显示了正在执行的进程名，运行函数，寄存器上下文，以及程序的调用栈回溯等信息。这对于分析死锁引起的系统挂起有着非常重要的作用。一般来说我们会多采几次重复样本，以便更加准确的做出系统运行状态的判断。

**T - 打印进程列表**

```
SysRq : Show State ` `                        ``sibling `` ``task       PC   pid father child younger older ``…… (snip) ……`` ``syslogd    S 00003280 2104 4313   1     4316 4279 (NOTLB) ``    ``dfc1cb68 00000082 b9d5334a 00003280 dd931a70 dd931a70 e08e9376 00000007 ``    ``dfc1eaa0 c06713c0 b9d85c35 00003280 000328eb 00000000 dfc1ebac c1404fe0 ``    ``00000010 df4f79c0 df4bd53c c0457dc0 df6438f4 ffffffff 00000000 00000000 `` ``Call Trace: `` ``[<``e08e9376``>] ext3_mark_iloc_dirty+0x2d8/0x333 [ext3] `` ``[<``c0457dc0``>] mempool_alloc+0x28/0xc9 `` ``[<``c06084b1``>] schedule_timeout+0x13/0x8c `` ``[<``c05ac8f2``>] datagram_poll+0x15/0xb8 `` ``[<``c0480ed8``>] do_select+0x371/0x3cb `` ``[<``c048145b``>] __pollwait+0x0/0xb2 `` ``…… (snip) ……`` ``[<``c0429566``>] do_setitimer+0x4a6/0x4b0 `` ``[<``c04817d7``>] sys_select+0xcf/0x180 `` ``[<``c0404eff``>] syscall_call+0x7/0xb `` ``======================= `` ``klogd     R running 2664 4316   1     4369 4313 (NOTLB)
```

该操作显示了进程列表，包含各进程的名称，进程 PID，父 PID 及兄弟 PID 等相关信息，以及进程的运行状态。对于正在运行中的进程（R），没有太多的信息。对于处于睡眠状态的进程，列出其调用栈回溯信息，以便进行调试跟踪。由于篇幅关系，去掉了大部分冗余的信息。

**W - 打印 CPU 信息**

```
SysRq : Show CPUs `` ``CPU0: `` ``c074be84 00000000 00000000 c053a1fa d23c0000 c0406531 c062ccf1 c0650e94 `` ``00000000 00000000 c053a221 c0641b83 00000000 c042a111 00000000 c068c630 `` ``00000001 00000077 c053a0cd 00000000 c053a14d c0640322 c0641d0f c06e7fa4 `` ``Call Trace: `` ``[<``c053a1fa``>] showacpu+0x0/0x32 `` ``[<``c0406531``>] show_stack+0x20/0x25 `` ``[<``c053a221``>] showacpu+0x27/0x32 `` ``[<``c042a111``>] on_each_cpu+0x17/0x1f `` ``[<``c053a0cd``>] sysrq_handle_showcpus+0x10/0x12 `` ``[<``c053a14d``>] __handle_sysrq+0x7e/0xf6 `` ``[<``c053a1d5``>] handle_sysrq+0x10/0x12 `` ``[<``c053567d``>] kbd_event+0x2e0/0x507 `` ``[<``c058e1c1``>] input_event+0x3e6/0x407 `` ``[<``c0591a2b``>] atkbd_interrupt+0x3f5/0x4da `` ``[<``c058b058``>] serio_interrupt+0x33/0x6a `` ``[<``c058bcb6``>] i8042_interrupt+0x1dd/0x1ef `` ``[<``c044dd9b``>] handle_IRQ_event+0x23/0x49 `` ``[<``c044de45``>] __do_IRQ+0x84/0xd6 `` ``[<``c04073f4``>] do_IRQ+0x93/0xae `` ``[<``c040592e``>] common_interrupt+0x1a/0x20 `` ``[<``c0403b98``>] default_idle+0x0/0x59 `` ``[<``c0403bc9``>] default_idle+0x31/0x59 `` ``[<``c0403c90``>] cpu_idle+0x9f/0xb9 `` ``[<``c06ec9ee``>] start_kernel+0x379/0x380 `` ``======================= `` ``CPU1: `` ``c14a1f40 00000000 00000000 00000000 00000000 c0406531 c062ccf1 c0650e94 `` ``00000000 00000000 c053a221 c0641b83 00000001 c0417b4c 00000001 c040599b `` ``00000001 c0403b98 c14a1000 00000000 00000000 00000000 00000000 0000007b `` ``Call Trace: `` ``[<``c0406531``>] show_stack+0x20/0x25 `` ``[<``c053a221``>] showacpu+0x27/0x32 `` ``[<``c0417b4c``>] smp_call_function_interrupt+0x39/0x52 `` ``[<``c040599b``>] call_function_interrupt+0x1f/0x24 `` ``[<``c0403b98``>] default_idle+0x0/0x59 `` ``[<``c0403bc9``>] default_idle+0x31/0x59 `` ``[<``c0403c90``>] cpu_idle+0x9f/0xb9 `` ``=======================
```

该操作显示了每 CPU 的寄存器上下文和程序调用栈回溯信息。

### 其它功能键组合

**H - 帮助**

```
SysRq : HELP : loglevel0-8 reBoot Crashdump tErm Full kIll saK showMem ``Nice powerOff showPc unRaw Sync showTasks Unmount shoWcpus
```

它显示了当前系统支持的所有 SysRq 组合，所有的按键均用大写字母表示。

0-8 - 更改 console_loglevel

```
SysRq : Changing Loglevel `` ``Loglevel set to 6
```

上面是 SysRq-6 的输出，它等同于 echo "6" > /proc/sys/kernel/printk 操作，将 console_loglevel 设置为 6 。

**C - 触发 Crashdump**

```
SysRq : Trigger a crashdump `` ``Kexec: Warning: crash image not loaded `` ``Kernel panic - not syncing: SysRq-triggered panic!
```

在大多数情况下，我们通过前面的方法即可完成对系统挂起的基本诊断和数据收集。但是在一些特殊情况下，我们仍然需要一份完整的 crashdump 。毕竟 crashdump 包含比 SysRq – MPT 更多的信息，以利用后期做故障分析。

**N - 降低实时任务运行优化级**

```
SysRq : Nice All RT Tasks
```

这对于由实时任务消耗 CPU 引起的系统挂起会起到立竿见影的作用。

**O - 关机**

```
SysRq : Power Off
```

该操作会立即关机，一般很少使用。在必要的情况下，也推荐跟随 S – U 一起使用。

## 结束语

SysRq 的处理机制可圈可点，它能够在系统处于极端环境时响应按键并完成相应的处理。这在大多数时候有用，但也不是绝对的。一种情况是由于磁盘故障 syslogd 可能不再会往磁盘回写 log，因此 SysRq 输出将得不到记录。即使配置了 netconsole，如果恰好是网络相关功能出现错误时，也可能导致 SysRq 输出不可记录。相比之下，将服务器串口终端打开并用软件记录输出的方式虽然原始，但也相对保险。无论如何，SysRq 已经在大多数情况下帮到大忙了。

笔者认为，SysRq 在处理系统挂起时安全重启方面已经比较完善了。但是在提供故障诊断方面还有可待改进的地方。通过现有的 M-P-T 等信息，很难判断每一次系统挂起的真正原因。现实世界的信息总是综合多变的，如果能够提供包括网络 I/O，磁盘 I/O，进程的 CPU 及内存使用情况等等与性能方面的数据，虽然会产生一定的开销，但对于系统挂起时的故障分析来讲，是利大于弊的。



本文的下篇将讲述如何扩展自定义的 SysRq 请求，以便在系统挂起时获取更为丰富的诊断信息，通过这些信息来判断系统挂起的原因，并快速准确的化解它们。

#### 相关主题

- 查看 [SysRq 内核文档](http://lxr.linux.no/linux/Documentation/sysrq.txt)，该文档描述了有关 SysRq 的所有信息。
- 查看 [SysRq 维基百科](http://en.wikipedia.org/wiki/Magic_SysRq_key)，也可以获取丰富的信息。
- 参考 [sysrqd](http://freshmeat.net/projects/sysrqd/) 项目，它是一个提供通过网络进行 SysRq 操作的服务程序。
- 查看 [kbd_mode](http://linux.die.net/man/1/kbd_mode) 手册，了解键盘的几种工作模式。
- 查看 [patch-2.1.43.bz2](http://www.kernel.org/pub/linux/kernel/v2.1/patch-2.1.43.bz2)，了解 SysRq 在 kernel 2.1.43 中的第一版实现。
- 查看 [proc-sysrq-trigger.patch](http://kernel.org/pub/linux/kernel/people/akpm/patches/2.5/2.5.64/2.5.64-mm7/broken-out/proc-sysrq-trigger.patch)，了解 /proc/sysrq-trigger 接口的实现。
- 参考 [Serial over IP](http://www.connectivity.avocent.com/products/network-based/)，了解通过 IP 网络访问串口终端的信息。
- 查看文章“[奇妙的 sys 请求](http://www.ibm.com/developerworks/cn/linux/l-tip-prompt/tip12/)”，可以了解到 SysRq 早在 9 年前就得以应用。
- 在 [developerWorks Linux 专区](http://www.ibm.com/developerworks/cn/linux/) 寻找为 Linux 开发人员（包括 [Linux 新手入门](http://www.ibm.com/developerworks/cn/linux/newto/)）准备的更多参考资料，查阅我们 [最受欢迎的文章和教程](http://www.ibm.com/developerworks/cn/linux/top10/index.html)。
- 在 developerWorks 上查阅所有 [Linux 技巧](http://www.ibm.com/developerworks/cn/views/linux/libraryview.jsp?search_by=Linux+技巧) 和 [Linux 教程](http://www.ibm.com/developerworks/cn/views/linux/libraryview.jsp?type_by=教程)。



![image-20210203140802306](https://gitee.com/cclinuxer/blog_image/raw/master/image/image-20210203140802306.png)