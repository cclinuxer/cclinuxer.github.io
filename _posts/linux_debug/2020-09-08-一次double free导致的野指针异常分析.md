







# 一次内存double free导致的野指针异常分析



## 1、问题背景

测试环境：设备配置路由模式（pppoe）连接外网；连接无线终端25个进行业务测试，挂机5天；

测试结果：设备存在重启2次以及无线驱动异常导致所有无线终端掉线3次的情况，恢复时间无法确定；

## 2、原理分析

### 2.1 802.11协议中BA机制

详情参考[Block ACK机制原理](http://wiki.360iot.qihoo.net/pages/viewpage.action?pageId=20454197)中的介绍，BA流程如下：

![ADDBA](https://gitee.com/cclinuxer/blog_image/raw/master/image/ADDBA.png)

### **2.2 无线驱动代码流程**

**MT7915方案BA会话代码流程图：**

![image2020-9-11_16-23-18](https://gitee.com/cclinuxer/blog_image/raw/master/image/image2020-9-11_16-23-18.png)

通过分析BA代码流程，我们了解了MT7915方案中ADDBA、DELBA的实际处理过程。

## 3、问题复现

该问题复现概率较小，经过多次挂机才能复现，所以添加打印信息等待问题复现，并搜集尽可能多的日志。

1、开启SLUB_DEBUG：

1）make kernel_menuconfig

2) 进入General setup，选中Enable SLUB debugging support，如下图所示。

![image2020-9-24_11-26-5](https://gitee.com/cclinuxer/blog_image/raw/master/image/image2020-9-24_11-26-5.png)

3）进入kernel_hacking — memory Debugging，选中SLUB debugging on by default。

![image2020-9-24_11-29-1](https://gitee.com/cclinuxer/blog_image/raw/master/image/image2020-9-24_11-29-1.png)

2、开启SLUB_DEBUG之后，复现问题即可搜集内存相关信息，如下图。

![image2020-9-23_11-13-26](https://gitee.com/cclinuxer/blog_image/raw/master/image/image2020-9-23_11-13-26.png)

从上述log可以看到和ba_resrc_rec_del有关，通过gdb可以找到该函数中double free的地方如下图。

![image2020-9-23_11-16-52](https://gitee.com/cclinuxer/blog_image/raw/master/image/image2020-9-23_11-16-52.png)



分析结果：Wcid 1 先free了 8d94c280，此地址又被wcid 4使用，然后ba_resrc_rec_del-> os_free_mem，再free的时候提示Object already free

日志及分析结果：

1）DELBA打印：Line 12199: [13-09-58-40]:[ 955.408720] ba_free_rec_entry pBAEntry->ba_rec_dbg =8d94c280 **Idx = 2,** REC_BA_Status = 0, Wcid(pBAEntry) = 1,      Wcid(pEntry) = 1, Tid = 7 *//* 此处wcid1 free了ba_rec_dbg

2）ADDBA打印：Line 13263: [13-09-58-58]:[ 973.483019] ba_resrc_rec_add(**5**): **Idx = 2**, BAWinSize = 64 //因为此时numAsRecipient=5，所以并没有分配内存空间给这个pBAEntry，而是直接使用这个pBAEntry原来的值，当Idx为2时，刚好和上一个free过的wcid使用同一个pBAEntry。

3）Line 14305: [13-09-59-00]:[ 975.500739] ba_free_rec_entry pBAEntry->ba_rec_dbg =8d94c280 **Idx = 2**, REC_BA_Status = 0, Wcid(pBAEntry) = 4,      Wcid(pEntry) = 4, Tid = 6 //再free 8d94c280

4）Line 14313: [13-09-59-00]:[ 975.500846] INFO: Slab 0x81f68900 objects=3 used=2 fp=0x8d94c280 flags=0x40004081 //系统提示Object already free

因此可以确定memory double free的原因：

- 客户端数量达到5个以上，且都支持BA会话。
- 同时有5个以上BA会话。
- 触发流程：前5个BA会话建立后，某一个BA会话退出，此时新的BA会话建立连接就会出问题。

## 4、解决方案

方案：在每一次释放pBAEntry→ba_rec_dbg指向的内存时，同时将pBAEntry→ba_rec_dbg指向NULL，从而避免了**野指针**导致的double free。