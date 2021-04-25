https://zhuanlan.zhihu.com/p/89794261





## Wireshark大法-WiFi6无线抓包

Wireshark是世界上最重要和最广泛使用的网络协议分析器。它让你在微观层面上看到你的网络上发生了什么。

Wi-Fi 6，也被称为802.11ax，是不间断创新之旅的最新一步。该标准建立在802.11ac的基础上，同时增加了效率、灵活性和可伸缩性，允许新网络和现有网络通过下一代应用程序提高速度和容量。

Intel®Wi-Fi 6 AX200适配器是为支持即将推出的IEEE 802.11ax标准- Wi-Fi 6技术和Wi-Fi联盟Wi-Fi 61认证而设计的。该产品支持2×2的Wi-Fi - 6技术，包括UL和DL OFDMA和1024QAM等新功能，提供高达2.4Gbps2的数据速率和增加的网络容量。

目前很少有免费的协议嗅探分析软件可以支持Wi-Fi 6标准，特别是支持无线帧捕获。

在本教程中，我将向您展示在ubuntu18.04上安装wireshark的步骤

## 硬件需求

-一台运行Ubuntu 18.04的笔记本电脑。

-一台安装了英特尔AX200 802.11ax无线网卡的笔记本电脑。

## 开始安装固件的

```text
sudo apt-get update -y
sudo apt-get upgrade -y
```

系统更新后，重新启动系统以应用更改。

由于系统当前Kernel并不包含最新的驱动程序，你可以选择升级Linux内核5.1或者安装新的驱动程序，二选一，并使用最新的无线适配器固件。

## 升级Linux内核

需要升级到最新的Linux内核>5.1，并使用最新的无线适配器固件。我已经将Linux内核升级到5.1版本

下载和安装内核的官方网站:

```text
cd /tmp/
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v5.1/linux-headers-5.1.0-050100_5.1.0-050100.201905052130_all.deb
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v5.1/linux-headers-5.1.0-050100-generic_5.1.0-050100.201905052130_amd64.deb
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v5.1/linux-image-unsigned-5.1.0-050100-generic_5.1.0-050100.201905052130_amd64.deb
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v5.1/linux-modules-5.1.0-050100-generic_5.1.0-050100.201905052130_amd64.deb
sudo dpkg -i *.deb
```

安装完成后，重新启动ubuntu系统。

```text
sudo reboot
```

然后检查linux内核版本:

```text
uname -a
```

## 升级Wi-Fi驱动程序

升级iwlwifi驱动程序，如下命令

```text
git clone --single-branch --branch release/core45 https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/backport-iwlwifi.git
cd backport-iwlwifi/
make defconfig-iwlwifi-public
sed -i 's/CPTCFG_IWLMVM_VENDOR_CMDS=y/# CPTCFG_IWLMVM_VENDOR_CMDS is not set/' .config
make -j4
sudo make install
```

## 安装AX 200最新固件

因为驱动程序本身还没有进入Ubuntu18.04和Ubuntu19.04Linux内核的内核。所以我们需要将AX 200固件安装到Linux上。

从下面的链接下载最新的固件。

[https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-i-o/wireless-networking.html](https://link.zhihu.com/?target=https%3A//www.intel.com/content/www/us/en/support/articles/000005511/network-and-i-o/wireless-networking.html)

安装固件的： 1.将文件复制到特定于发行版的固件目录/lib/固件中 2.如果目录不工作，请参考您的分发文档。 3.如果您自己配置内核，请确保启用了固件加载。

## 将无线适配器配置为监视模式

为了查看802.11头文件，您必须在监视器模式下捕获。为接口手动打开或关闭监视模式的最简单方法是使用aircrack-ng中的airmon-ng脚本；您的发行版可能已经有了一个用于aircrack-ng的包。所以我们需要先安装气裂包。

```text
sudo apt-get install aircrack-ng
```

Note that the behavior of airmon-ng will differ between drivers that support the new mac80211 framework and drivers that don’t. For drivers that support it, a command such as

请注意，在支持新mac80211框架的驱动程序和不支持新框架的驱动程序之间，airmon-ng的行为将有所不同。

```text
sudo airmon-ng start wlan0
```

然后Linux终端将产生输出：（例子，仅作参考）

```text
Interface Chipset Driver
wlan0 Intel 4965 a/b/g/n iwl4965 – [phy0]
(monitor mode enabled on mon0)
```

“在mon0上启用的监视模式”意味着必须在“mon0”接口上捕获，而不是在“wlan0”接口上捕获。若要关闭监视器模式，请使用如下命令

```text
sudo airmon-ng stop mon0
```

不能用如下命令

```text
sudo airmon-ng stop wlan0
```

## 安装Wireshark开发版本以支持最新的Wi-Fi 6规范

添加PPA存储库并安装Wireshark。

```text
sudo add-apt-repository ppa:wireshark-dev/stable
sudo apt update
sudo apt -y install wireshark
sudo apt -y install wireshark-qt
```

安装Wireshark开发版本以获得开发版本，添加

```text
sudo add-apt-repository ppa:dreibh/ppa
```

从存储库安装Wireshark

```text
sudo apt update
sudo apt -y install wireshark
```

## 验证Wireshark能捕获11 AX帧

```text
sudo wireshark
```