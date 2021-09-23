> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269456.htm)

> [原创] 一加 7t 自定义内核编译, 以及 android10, syscalltable hook 实现

op7t
====

oneplus 7t 自定义内核

使用 github ci
------------

[https://github.com/yhnu/op7t](https://github.com/yhnu/op7t)

 

目前已经使用 github ci 进行自动化构建, 欢迎 fork and star

手机环境
----

a. oneplus 7t

 

b. 对应 Android 版本为 Hydrogen OS 10.0.7.HD65  
![](https://bbs.pediy.com/upload/attach/202109/933020_4UYKAQEKR4C9MDN.png)

 

c. 对应版本全量包

```
OnePlus7THydrogen_14.H.09_OTA_009_all_2001030048_d935aae55ac_1007.zip //一加手机论坛下载

```

Linux 编译环境
----------

1.  操作系统 ubuntu20

```
➜  /share cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04 LTS"

```

1.  依赖安装

```
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5-dev lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
 
sudo apt-get install libssl-dev
sudo apt-get install dos2unix
sudo apt-get install libncurses5

```

1.  编译依赖

百度网盘整个工程包括编译链

 

链接: https://pan.baidu.com/s/1RQaPSjLPNX99XBVCCAcoZA 提取码: bbki 复制这段内容后打开百度网盘手机 App，操作更方便哦

```
cd /share
git clone https://gitee.com/yhnu/op7t.git
cd /share/op7t/buildtool
just c

```

1.  编译源码

为了简单方便及稳定, 我们基于第三方内核进行修改, [https://github.com/engstk/op7/tree/r70](https://github.com/engstk/op7/tree/r70)

```
cd /share/op7t/blu7t
just c
source env.config
source proxy.config
just make
just j16

```

1.  打包 image

```
just image

```

boot.img 提取
-----------

```
curl -L -O https://github.com/yhnu/op7t/releases/download/v1.0/payload_dumper-win64.zip
curl -L -O https://otafsc.h2os.com/patch/CHN/OnePlus7THydrogen/OnePlus7THydrogen_14.H.09_009_2001030048/OnePlus7THydrogen_14.H.09_OTA_009_all_2001030048_d935aae55ac.zip
unzip -q OnePlus7THydrogen_14.H.09_009_2001030048/OnePlus7THydrogen_14.H.09_OTA_009_all_2001030048_d935aae55ac.zip
# 然后参考payload_dumper的使用说明即可

```

拯救 OnePlus7t
------------

开发的路上难免磕磕碰碰, 手机就启动不了, 做好防身技能

```
curl -L -O https://otafsc.h2os.com/patch/CHN/OnePlus7THydrogen/OnePlus7THydrogen_14.H.09_009_2001030048/OnePlus7THydrogen_14.H.09_OTA_009_all_2001030048_d935aae55ac.zip
curl -L -O https://github.com/yhnu/op7t/releases/download/v1.0/recovery-oneplus7t-3.4.2-10.0-b26.img
fastboot set_active a #is a or b
fastboot erase recovery
fastboot.exe flash recovery recovery-oneplus7t-3.4.2-10.0-b26.img
fastboot.exe reboot recovery
adb sideload F:\F2021-07\one7t_kernel\OnePlus7THydrogen_14.H.09_OTA_009_all_2001030048_d935aae55ac.zip

```

开发记录
----

2021 年 9 月 7 日 14:33:03

 

a. 修改内核源码绕过反调试检测

 

参考文章:

 

[安卓 10 源码学习开发定制 (6) 修改内核源码绕过反调试检测](https://mp.weixin.qq.com/s?__biz=Mzg5MzU3NzkxOQ==&mid=2247483992&idx=4&sn=1137e15288c238668fb39462e295d82c&chksm=c02dfd88f75a749e38c772d45d620a615d77d21b2f3c729ecd767a13a4c674204fe6120beea2&scene=178&cur_album_id=1799542483832324102#rd)

 

2021 年 9 月 10 日 08:56:45

 

a. 编写 hellomod 模块

```
# Code wrire     : yhnu
# code date     : 2021年9月10日 09:46:05
# e-mail    : buutuud@gmail.com
#
# THis Makefile is a demo only for ARM-architecture
#
 
MODULE_NAME := hello
 
ifneq ($(KERNELRELEASE),)
 
    obj-m := hello.o
 
else
    CROSS_COMPILE := /share/op7t/buildtool/aarch64-linux-android-4.9-uber-master/bin/aarch64-linux-android-
    #CC = CROSS_COMPILE   
    #KERNELDIR ?= /lib/modules/$(shell uname -r)/build
    PWD    :=$(shell pwd)
    KDIR := /share/op7t/blu7t/op7-r70/out
 
modules:
    make -C $(KDIR) REAL_CC=$(GITHUB_WORKSPACE)/buildtool/toolchains/llvm-Snapdragon_LLVM_for_Android_8.0/prebuilt/linux-x86_64/bin/clang CROSS_COMPILE=/share/op7t/buildtool/aarch64-linux-android-4.9-uber-master/bin/aarch64-linux-android- CLANG_TRIPLE=aarch64-linux-gnu- ARCH=arm64 M=$(PWD) modules CONFIG_MODULE_UNLOAD=y CONFIG_RETPOLINE=y
 
clean:
    rm -rf *.o *.order *.symvers *.ko *.mod* .*.cmd .*.*.cmd .*.*.*.cmd
    @rm -fr .tmp_versions Module.symvers modules.order
endif

```

b. 注意:

 

因为内核编译使用的是 Clang 编译, 因此对应 module 的编译也需要使用 Clang 编译

 

[module 的加载过程](https://www.cnblogs.com/sky-heaven/p/5569240.html)

 

[clang 交叉编译](https://blog.csdn.net/qq_23599965/article/details/90901235)

```
triple 的一般格式为---，其中：
 
arch = x86_64、i386、arm、thumb、mips等。
sub = v5, v6m, v7a, v7m等。
vendor = pc, apple, nvidia, ibm,等。
sys = none, linux, win32, darwin, cuda等。
abi = eabi, gnu, android, macho, elf等。 
```

2021 年 9 月 10 日 16:33:14

 

a. linux 设备驱动开发学习经典教程

 

[https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

 

2021 年 9 月 11 日 21:14:44

查看 KernelLog
------------

```
adb logcat -b kernel,default

```

2021 年 9 月 17 日 09:01:09

krhook 模块开发
-----------

https://github.com/yhnu/op7t/tree/dev/krhook

 

参考链接:

 

https://bbs.pediy.com/thread-267004.htm

 

现在 Android 手机大都使用了 MSM 平台 和 kernel， 高通下面的一个 patch 引入了 kernel 代码段内存 RO 属性. 因此需要做一些修改

 

2021 年 9 月 17 日 14:28:50

 

友人提醒可以直接通过下面的编译器进行编译, 不用改代码 Makefile

 

https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads

 

![](https://cdn.jsdelivr.net/gh/yhnu/PicBed/images3cb1209765988e17c4de9b078a07eea.png)

 

2021 年 9 月 17 日 16:28:58

1.  /sys/fs/pstore 内核崩溃日志信息存放的位置
    
2.  linux 使用 lwp 机制, 导致 task_struct 的 pid 在线程中是线程 id, 需要使用 tgid, 或者使用 uid 进行识别
    

http://www.opensourceforu.com/2011/08/light-weight-processes-dissecting-linux-threads/

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 6 分钟前 被 wx_易罗阳编辑 ，原因： fix img

[#调试逆向](forum-4-1-1.htm) [#系统底层](forum-4-1-2.htm) [#软件保护](forum-4-1-3.htm) [#VM 保护](forum-4-1-4.htm)