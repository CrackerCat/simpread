> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278647.htm)

> [原创] 将 rwProcMem33 编译进安卓内核

[原创] 将 rwProcMem33 编译进安卓内核

17 小时前 352

### [原创] 将 rwProcMem33 编译进安卓内核

 [![](http://passport.kanxue.com/upload/avatar/320/963320.png?1667284712)](user-home-963320.htm) [oacia](user-home-963320.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 17 小时前  352

> 为了做出腾讯游戏安全竞赛初赛的这道安卓题, 开始学习`rwProcMem33` 的使用来打硬件断点了
> 
> 在 [juice4fun](https://bbs.kanxue.com/user-home-831526.htm) 师傅做腾讯游戏安全竞赛初赛的安卓题的 [writeup](https://bbs.kanxue.com/thread-276896.htm#msg_header_h2_2) 时, 使用了 [rwProcMem33](https://github.com/abcz316/rwProcMem33) 来对安卓手机打下硬件断点分析反调试, 我也对在安卓手机中打硬件断点的工具很感兴趣, 所以就学习一下编译和使用的方法啦
> 
> 要想使用 rwProcMem33, 编译环境 (即 AOSP 安卓内核源码环境) 的搭建过程是必不可少的, 因为最终内核模块是运行在安卓手机的 linux 内核中, 而非虚拟机的 linux 内核中, 所以内核源码是有必要下载的
> 
> 所以这篇文章不仅是硬件断点工具的编译和使用笔记, 也是安卓内核的编译笔记, 用来记录我在编译内核的过程中遇到的困难, 以及如何克服的
> 
> 为了更快的下载速度, 可以选择配置代理, 也可以手动切换下载源, 只要不出现网络问题导致下载失败就行
> 
> docker 的使用完全是因为我的虚拟机 shell 环境崩溃, 从而导致无法编译, 如果对自己虚拟机的 shell 环境足够自信, 不使用 docker 也是可以的

编译环境搭建
======

虚拟机配置
-----

我使用的虚拟机为`Ubuntu22.04`, 虚拟机的参考配置如下

![](https://bbs.kanxue.com/upload/attach/202308/963320_PETCHPHSF5TSJTH.png)

为虚拟机配置代理
--------

### 查看虚拟机代理地址

打开`clash for windows`, 并打开`Allow LAN`的开关, 随后点击`network interfaces`

![](https://bbs.kanxue.com/upload/attach/202308/963320_USNYY8N3ZYGGU87.png)

请注意我的虚拟机使用的网络连接方式为 NAT 模式, 所以需要关注`VMnet8`的地址, 所以对于该虚拟机, 代理地址为`192.168.27.1`, 端口就是`clash`中`Port`选项所显示的端口

![](https://bbs.kanxue.com/upload/attach/202308/963320_F5Y9EE4YGGJ2PP7.png)

### 启用虚拟机代理

依次点击如下选项进入代理配置

![](https://bbs.kanxue.com/upload/attach/202308/963320_MSKAYGFMEHH78HB.png)

输入代理地址保存即可

![](https://bbs.kanxue.com/upload/attach/202308/963320_2UY6VUJFA74ZMGP.png)

配置 docker ubuntu 镜像
-------------------

> 本人因为不小心运行了一个命令`source _setup_env.sh`, 导致虚拟机的 shell 环境整个崩掉了,`build.sh`也屡屡运行失败, 看了眼`_setup_env.sh`我真是只能苦涩的笑...
> 
> ![](https://bbs.kanxue.com/upload/attach/202308/963320_UNSWBP4C8JHX4QW.png)
> 
> 所以不得不用 docker 了, 不过用下来发现竟然意外的好用

### 安装 docker

```
sudo apt install docker.io

```

### docker pull 代理配置

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo gedit /etc/systemd/system/docker.service.d/proxy.conf

```

输入以下代理服务器内容

```
[Service]
Environment="HTTP_PROXY=http://192.168.27.1:7890/"
Environment="HTTPS_PROXY=http://192.168.27.1:7890/"
Environment="NO_PROXY=localhost,127.0.0.1"

```

### 刷新配置并重启 docker 服务

```
sudo systemctl daemon-reload
sudo systemctl restart docker

```

### docker 镜像代理配置

```
sudo mkdir -p ~/.docker/
sudo gedit ~/.docker/config.json

```

输入以下内容

```
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://192.168.27.1:7890/",
     "httpsProxy": "http://192.168.27.1:7890/",
     "noProxy": "localhost,127.0.0.1"
   }
 }
}

```

### 下载 Ubuntu 镜像

```
docker pull ubuntu

```

### 运行 Ubuntu 镜像

```
docker run -it --net host --name Akernel ubuntu /bin/bash

```

安装 sudo,vim
-----------

```
apt-get update
apt-get install vim
apt-get install sudo

```

修改 apt-get 的软件源为阿里源
-------------------

```
sudo cp /etc/apt/sources.list /etc/apt/sources.list_backup
sudo vim /etc/apt/sources.list

```

替换为如下内容

```
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
 
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
 
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
 
# deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
 
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse

```

随后将`apt-get`更新至最新版本

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install build-essential

```

安装必要的库
------

```
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig libssl-dev bc kmod cpio git curl

```

为 git 配置基本信息
------------

```
git config --global user.email "xxx@gmail.com"
git config --global user.name "xxx"
git config --global http.proxy 192.168.27.1:7890

```

安装 repo
-------

```
mkdir ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo

```

修改 repo 的下载源为清华源, 并添加 repo 至全局变量
--------------------------------

### 打开全局变量配置文件

```
sudo vim ~/.bashrc

```

### 添加全局变量

在末尾添加这三行并保存

```
# repo
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'
export PATH="~/bin:$PATH"

```

### 使配置文件生效

```
source ~/.bashrc

```

安装 python
---------

如果使用`python --version`有打印 python 版本的话, 那么这一步就不需要了, 如果 docker 中没有安装`python`, 在 docker 内使用如下命令安装

```
sudo apt-get install software-properties-common
add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.9
sudo ln -s /usr/bin/python3 /usr/bin/python

```

修改交换区大小
-------

为了防止编译源码的过程中由于交换区不足而失败, 所以我们需要去修改**虚拟机**的交换区的大小

```
sudo swapoff /swapfile
sudo rm /swapfile
 
# 设置了32g交换区, 防止编译失败，执行下列命令需要花费一段时间，如果执行命令后没有输出，请耐心等待命令执行完毕
sudo dd if=/dev/zero of=/swapfile bs=1GB count=32
sudo chmod 600 /swapfile
sudo mkswap -f /swapfile
sudo swapon /swapfile

```

下载内核源码
======

查看内核版本号
-------

这里我的`pixel3`的内核版本为`Linux version 4.9.270-g862f51bac900-ab7613625`

```
adb shell cat /proc/version
Linux version 4.9.270-g862f51bac900-ab7613625 (android-build@abfarm-east4-101) (Android (7284624, based on r416183b) clang version 12.0.5 (https://android.googlesource.com/toolchain/llvm-project c935d99d7cf2016289302412d708641d52d2f7ee)) #0 SMP PREEMPT Thu Aug 5 07:04:42 UTC 2021

```

查看内核源码分支
--------

进入 [building-kernels](https://source.android.com/setup/build/building-kernels) 中查看自己的源码分支, 如图我的手机型号为`pixel3`, 并且内核版本为 4.9, 于是就知道内核源码的代号是`android-msm-crosshatch-4.9`

![](https://bbs.kanxue.com/upload/attach/202308/963320_8HQ82VK39A9MFGD.png)

之后进入[安卓内核源码列表](https://android.googlesource.com/kernel/manifest/+refs)中, 搜索内核源码代号`android-msm-crosshatch-4.9`

![](https://bbs.kanxue.com/upload/attach/202308/963320_A5EU9GBYQE628Y3.png)

我的手机是安卓 12, 所以我下载的内核版本为`android-msm-crosshatch-4.9-android12`

这里的`qpr1`和`qpr2`, 我们先看一下`QPR`的定义, 简单来说就是`QPR`后面跟的数字越高, 内核版本就越新

> QPR, of course, is short for the **Quarterly Platform Release**, which Google first introduced with Android 12. These are not full system updates, but they bring a few select changes to the Pixels and other great high-end phones that opt to receive them.

注意一点就是例如你想要下载 [android-msm-crosshatch-4.9-android10](https://android.googlesource.com/kernel/manifest/+/refs/heads/android-msm-crosshatch-4.9-android10), 请先进入你想要下载的 AOSP 的地址, 看看仓库中的`default.xml`文件, 重点关注`<project path="build"` , 如果`revision`的值为`main`, 请千万不要下载, 否则你就会发现下载下来之后根本无法 build!!, **这是由于 build 仓库和内核源码仓库不同步导致的**

![](https://bbs.kanxue.com/upload/attach/202308/963320_KN6A4YGXMDX846X.png)

当然你也可以选择进入 [kernel/build](https://android.googlesource.com/kernel/build/+refs) 中找到适合你的`build file`, 不过我还是建议能一键编译就一键编译, 比如我下载的`android-msm-crosshatch-4.9-android12`,`build`和`kernel source code`就是同步的 (惨痛的教训, 头铁想要用安卓 9 的内核源码编译, 结果根本无法 build... 最后还是妥协把手机刷成安卓 12 了)

下载安装内核源码
--------

```
mkdir android-kernel && cd android-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b android-msm-crosshatch-4.9-android12
repo sync -j4

```

切换 git 分支
---------

我的手机的 Linux 内核版本为`Linux version 4.9.270-g862f51bac900-ab7613625`,`g`后面跟的是 git 分支, 所以切换的分支为`862f51bac900`

```
cd private/msm-google
git checkout 862f51bac900

```

编译内核源码
======

解包 boot.img
-----------

首先下载 [android-image-kitchen](https://forum.xda-developers.com/attachments/android-image-kitchen-v3-8-win32-zip.5300919/)

然后将`boot.img`放在工具的根目录下, 这里的`boot.img`就是网上下载的刷机包解压之后其中的`boot.img`

![](https://bbs.kanxue.com/upload/attach/202308/963320_B6JZYZ8DF4YC7MU.png)

然后运行`unpackimg.bat`, 运行之后的窗口请不要关闭, 因为输出中有需要后续使用到的参数, 当然也可以将输出的内容复制下来到 txt 中

```
Android Image Kitchen - UnpackImg Script
by osm0sis @ xda-developers
 
Supplied image: boot.img
 
Removing old work folders and files . . .
 
Setting up work folders . . .
 
Image type: AOSP
 
Signature with "AVBv2" type detected.
 
Splitting image to "split_img/" . . .
 
ANDROID! magic found at: 0
BOARD_KERNEL_CMDLINE console=ttyMSM0,115200n8 androidboot.console=ttyMSM0 printk.devkmsg=on msm_rtb.filter=0x237 ehci-hcd.park=3 service_locator.enable=1 cgroup.memory=nokmem lpm_levels.sleep_disabled=1 usbcore.autosuspend=7 loop.max_part=7 androidboot.boot_devices=soc/1d84000.ufshc androidboot.super_partition=system buildvariant=user
BOARD_KERNEL_BASE 0x00000000
BOARD_NAME
BOARD_PAGE_SIZE 4096
BOARD_HASH_TYPE sha1
BOARD_KERNEL_OFFSET 0x00008000
BOARD_RAMDISK_OFFSET 0x01000000
BOARD_SECOND_OFFSET 0x00000000
BOARD_TAGS_OFFSET 0x00000100
BOARD_OS_VERSION 12.0.0
BOARD_OS_PATCH_LEVEL 2021-10
BOARD_HEADER_VERSION 2
BOARD_HEADER_SIZE 1660
BOARD_DTB_SIZE 863100
BOARD_DTB_OFFSET 0x01f00000
 
Unpacking ramdisk to "ramdisk/" . . .
 
Compression used: gzip
56773 blocks
 
Done!
 
请按任意键继续. . .

```

当`unpackimg.bat`运行完毕后, 我们进入`split_img/`, 然后解压其中的`boot.img-ramdisk.cpio.gz`, 并将解压后的`boot.img-ramdisk.cpio`文件复制到内核源码根目录中

```
root@oacia-virtual-machine:/home/oacia/Desktop/# docker cp boot.img-ramdisk.cpio Akernel:/android-kernel/

```

下载 mkbootimg.py
---------------

我们还需要下载 [mkbootimg.py](http://aospxref.com/android-11.0.0_r21/xref/system/tools/mkbootimg/mkbootimg.py), 并将其复制到放到内核源码根目录

```
docker cp mkbootimg.py Akernel:/android-kernel/

```

修改 build.sh
-----------

在内核源码根目录, 进入`build/build.sh`, 找到下方代码的位置

```
echo "========================================================"
echo " Files copied to ${DIST_DIR}"

```

并在这两行代码之前加上下列命令

```
if [ -f "${VENDOR_RAMDISK_BINARY}" ]; then
cp ${VENDOR_RAMDISK_BINARY} ${DIST_DIR}
fi

```

![](https://bbs.kanxue.com/upload/attach/202308/963320_H5J3UM8FPRGRKUB.png)

下载 rwProcMem33
--------------

下载地址

```
https://github.com/abcz316/rwProcMem33

```

然后将解压后的文件夹复制到 docker 中内核源码目录下的`private/msm-google/drivers/`中

```
docker cp rwProcMem33 Akernel:/android-kernel/private/msm-google/drivers/

```

修改 rwProcMem33
--------------

### ver_control.h

将`MY_LINUX_VERSION_CODE`切换到对应的安卓内核版本, 我们在下载内核源码阶段已经通过`cat /proc/version`知道了内核的版本号为`Linux version 4.9.270-g862f51bac900-ab7613625`, 所以在`private/msm-google/drivers/rwProcMem33/ver_control.h`和`private/msm-google/drivers/rwProcMem33/hwBreakpointProcModule/hwBreakpointProc/ver_control.h`中我们也将内核切换到对应的`4.9`版本, 选择`MY_LINUX_VERSION_CODE`的原则选这里出现的版本号中越接近自己手机内核版本的版本号

![](https://bbs.kanxue.com/upload/attach/202308/963320_F4BYD36TCJJJ275.png)

选择`启用页表计算物理内存的地址`, 同时注释掉`启用读取pagemap文件来计算物理内存的地址`如图所示

![](https://bbs.kanxue.com/upload/attach/202308/963320_667XQAH44UAH233.png)

### phy_mem.h

打开安卓内核源码目录下的`private/msm-google/mm/maccess.c`文件, 来到这个位置查看内核内存拷贝函数的函数名, 需要知道的是`Long __weak`后面修饰的函数名, 我内核版本`4.9.270`对应内核内存拷贝函数名为`probe_kernel_read`

![](https://bbs.kanxue.com/upload/attach/202308/963320_YWRUU4ACZCEKC4F.png)

在`private/msm-google/drivers/rwProcMem33/phy_mem.h`中全局搜索`x_probe_kernel_read`函数的位置, 将这个函数名修改为刚刚从`private/msm-google/mm/maccess.c`获取的函数名`probe_kernel_read`

![](https://bbs.kanxue.com/upload/attach/202308/963320_QYJRC5X4BRG9YSA.png)

修改 drivers 的 Makefile
---------------------

在`private/msm-google/drivers/Makefile`的开头加入下列命令

```
obj-y += rwProcMem33/hwBreakpointProcModule/hwBreakpointProc/
obj-y += rwProcMem33/


```

修改 msm-google 的 Makefile
------------------------

在编译`rwProcMem33`内核模块时, 由于内核编译时会将警告视为错误导致编译内核停止, 所以我们要修改 Makefile 来忽视 warning

在`private/msm-google/Makefile`找到如下位置, 在`-Wno-format-security`后加上一个`-w`参数

![](https://bbs.kanxue.com/upload/attach/202308/963320_MFFQ6TSK7H673BG.png)

开始编译
----

在安卓内核源码的根目录打开终端使用如下命令开始编译

> 命令的参数为使用`android-image-kitchen`解包`boot.img`之后, 控制台所打印的参数请务必替换为相对应的参数!!!

参数对应关系为

<table><thead><tr><th><code>android-image-kitchen</code>解包参数名称</th><th>值</th><th>编译命令参数名称</th></tr></thead><tbody><tr><td>BOARD_KERNEL_CMDLINE</td><td>console=ttyMSM0,115200n8 androidboot.console=ttyMSM0 printk.devkmsg=on msm_rtb.filter=0x237 ehci-hcd.park=3 service_locator.enable=1 cgroup.memory=nokmem lpm_levels.sleep_disabled=1 usbcore.autosuspend=7 loop.max_part=7 androidboot.boot_devices=soc/1d84000.ufshc androidboot.super_partition=system buildvariant=user</td><td>KERNEL_CMDLINE</td></tr><tr><td>BOARD_KERNEL_BASE</td><td>0x00000000</td><td>BASE_ADDRESS</td></tr><tr><td>BOARD_PAGE_SIZE</td><td>4096</td><td>PAGE_SIZE</td></tr><tr><td>BOARD_HEADER_VERSION</td><td>2</td><td>BOOT_IMAGE_HEADER_VERSION</td></tr></tbody></table>

编译命令中的`BUILD_CONFIG`为 AOSP 源码根目录的`build.config`的软连接所指向的配置文件

![](https://bbs.kanxue.com/upload/attach/202308/963320_SCTQ3S5JTATEDRE.png)

所以最终的编译命令为

```
BUILD_CONFIG=private/msm-google/build.config.bluecross BUILD_BOOT_IMG=1 MKBOOTIMG_PATH=mkbootimg.py VENDOR_RAMDISK_BINARY=boot.img-ramdisk.cpio KERNEL_BINARY=Image.lz4 BOOT_IMAGE_HEADER_VERSION=2 KERNEL_CMDLINE="console=ttyMSM0,115200n8 androidboot.console=ttyMSM0 printk.devkmsg=on msm_rtb.filter=0x237 ehci-hcd.park=3 service_locator.enable=1 cgroup.memory=nokmem lpm_levels.sleep_disabled=1 usbcore.autosuspend=7 loop.max_part=7 androidboot.boot_devices=soc/1d84000.ufshc androidboot.super_partition=system buildvariant=user" BASE_ADDRESS=0x00000000 PAGE_SIZE=4096 build/build.sh

```

经过一段时间的等待, 编译成功!

![](https://bbs.kanxue.com/upload/attach/202308/963320_SC42B8MGHAF7JG5.png)

生成的`boot.img`的路径为`/android-kernel/out/android-msm-pixel-4.9/dist/boot.img`

使用 Magisk 修补 boot.img 实现 root
=============================

安装 Magisk [下载地址](https://github.com/topjohnwu/Magisk/releases)

```
adb install "D:\TOOLS\pixel3\Magisk-v26.1.apk"

```

将镜像解压后, 把`boot.img`传到手机上

```
adb push boot.img /sdcard/

```

然后再手机上打开`Magisk`, 依次点击`安装->选择并修补一个文件->/sdcard/boot.img->开始`

待修补完成后, 将修补后的`boot.img`传到电脑上

```
adb pull /storage/emulated/0/Download/magisk_patched-xxxxx_xxxxx.img

```

刷入内核
====

```
adb reboot bootloader
fastboot flash boot boot-android12-rwProcMem33-root.img
fastboot reboot

```

编译 HwBpClient 客户端
=================

进入`rwProcMem33\hwBreakpointProcModule\testHwBpClient`文件夹, 双击`testHwBpClient.vcxproj`在`visual studio`中打开

编译的程序位数应为 **64 位**

对于 **Windows 平台**编译的`HwBpClient`, 并且需要在`testHwBpClientDlg.cpp`的这个位置进行修改, 将这个位置的`llX`改为`I64X`,`%zu`改成`I64d`, 否则将无法正确输入内容

![](https://bbs.kanxue.com/upload/attach/202308/963320_X5V86DP37H26XMY.png)

原因在于 C/C++ 输出 64 位数在 window 下和 linux 下是不一样的

linux

```
printf("%lld/n",a);
printf("%llu/n",a);
printf("%llx/n",a);

```

windows

```
printf("%I64d/n",a);
printf("%I64u/n",a);
printf("%I64x/n",a);

```

修改完成后如图所示

![](https://bbs.kanxue.com/upload/attach/202308/963320_GW9BWUGN75GWMVH.png)

接下来按下 ctrl+B 生成即可

![](https://bbs.kanxue.com/upload/attach/202308/963320_KHJ9X4BKEV3M6Z9.png)

编译 HwBpServer 服务端
=================

编译 HwBpServer 服务端需要 NDK, 并将 NDK 引入环境变量中

NDK 可以在`android studio`中进行安装, 依次点击`File->Project Structure->SDK Location`, 找到`Android NDK location`, 点击`Download`即可开始安装, 我这里使用的 ndk 的版本为`ndk25.2.9519653`

如果没有安装`Android studio`,NDK 的安装方法可以在谷歌上找到

NDK 安装完成后, 进入到`rwProcMem33\hwBreakpointProcModule\testHwBpServer\jni`, 打开 cmd 运行命令

```
ndk-build

```

即可完成编译, 编译后的文件在`rwProcMem33\hwBreakpointProcModule\testHwBpServer\libs`中, 选择对应手机架构的 ELF,`push`到手机中运行即可

获取手机的 IPv4 地址
=============

将电脑和我们的测试手机连接**同一个手机热点**, 然后在测试手机中打开`设置->WLAN`点击我们连接的热点的高级选项, 来查看手机的 IP 地址

运行 HwBpServer 服务端
=================

将编译出来的程序复制到手机中并运行

```
adb push rwProcMem33\hwBreakpointProcModule\testHwBpServer\libs\arm64-v8a\testHwBpServer.out /data/local/tmp
adb shell
su
cd /data/local/tmp
chmod 777 testHwBpServer.out
./testHwBpServer.out

```

查看打印出来的端口号`3170`

![](https://bbs.kanxue.com/upload/attach/202308/963320_ASHSYVFXXSPRC3P.png)

运行 HwBpClient 客户端
=================

在电脑中运行编译完成的 HwBpClient 客户端, 填入手机的 IP 地址以及由`testHwBpServer.out`打印出来的端口号

![](https://bbs.kanxue.com/upload/attach/202308/963320_CUZGWHFHDT9Q8Y7.png)

点击连接即可开始打硬件断点

![](https://bbs.kanxue.com/upload/attach/202308/963320_62SPJPAK3XF96Q6.png)

查询进程 PID
========

```
ps -A

```

查询目标 so 的基址
===========

可以使用命令来查询

```
cat /proc/[app的PID]/maps

```

也可以使用下面的 frida 脚本查询 so 的基址

```
function dump_so(so_name) {
    Java.perform(function () {
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = currentApplication.getApplicationContext().getFilesDir().getPath();
        var libso = Process.getModuleByName(so_name);
        console.log("[name]:", libso.name);
        console.log("[base]:", libso.base);
        console.log("[size]:", ptr(libso.size));
        console.log("[path]:", libso.path);
        }
    });
}
 
rpc.exports = {
    dump_so: dump_so
};

```

接下来就可以愉快的打硬件断点啦~

参考资料
====

*   [[原创] 开源一个 Linux 内核里进程内存管理模块源码](https://bbs.kanxue.com/thread-259317.htm)
    
*   [万字长文教你使用安卓内核驱动进行内存读写](https://blog.csdn.net/qq_46832407/article/details/129971512)
    
*   [安卓内核驱动编译](https://www.bilibili.com/video/BV1hR4y1k795)
    
*   [AOSP Android 10 内核编译刷入 Pixel3](https://www.sunofbeach.net/a/1624287020915548162)
    
*   [kernel 编译的哪些坑](https://zhuanlan.zhihu.com/p/582258078)
    
*   [eBPF on Android 之打补丁和编译内核修正版](https://blog.seeflower.dev/archives/174/)
    
*   [Android10 内核编译笔记](https://bbs.kanxue.com/thread-277224.htm#msg_header_h2_4)
    

  

[顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》！](https://www.kanxue.com/book-section_list-170.htm)

[#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)