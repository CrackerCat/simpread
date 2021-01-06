> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483936&idx=1&sn=89cdee39c0bda4cca7ea4d289a03c62f&chksm=ce075365f970da732f7325b1f58a7d825a8e29ce4c0d1e58c054db5769a92b2123d785babd63&scene=126&sessionid=1609918324&key=e74904ee7729bbf0d44d08ba986caf8781d274405ae9b679ef8267d4ec6d85540b9d80dd52a0c5d61e337545155c0f074d180dfafe9dca9752517a8c014b51181270802a77b2dc8360474a9070cdfdfe5b5dd6f101956e5cd4a3532716b31e954490ba655ec9e4da2f2b7310150196e56a55db817d6b2e40ae178591ea0c8b9b&ascene=1&uin=MTEzMjI1NDQxNg%3D%3D&devicetype=Windows+10+x64&version=62090538&lang=zh_CN&exportkey=AzMa4L3Bbzbjf%2BOFC%2F51lMY%3D&pass_ticket=KeN0lFiEaCpzdNEokHqqE8pzcMGIUgpc01WlY5hdkgj0ioW%2BCI05n5rTwCiI7uLM&wx_header=0)

一、 开发前期准备
---------

本文中使用的是 linageOs 源码中下载的 oneplus3 安卓 10 内核源码进行研究测试。交叉编译链使用的是 linageOs 源码中的交叉编译链。

lineageOs 源码中 oneplus3 内核源码位置路径:

```
/home/qiang/lineageOs/kernel/oneplus/msm8996


```

lineageOs 源码中交叉编译目录位置路径:

```
/home/qiang/lineageOs/prebuilts/gcc/linux-x86


```

为了方便研究测试，不破坏 lineageOs 中的内核源码结构。我新建一个目录专门存放内核源码、内核模块源码。并将内核源码拷贝到该目录。

本文后续测试的内核源码目录路径:

```
 home/qiang/myproject/kernel/oneplus3/msm8996


```

本文后续内核模块编写存放目录路径:

```
/home/qiang/myproject/kernel/oneplus3/modules


```

二、编译内核源码
--------

1.  找到 oneplus3 设备的内核源码配置
    

安卓源码中 **device / 厂商 / 手机型号 / BoardConfig.mk** 文件中配置了内核源码路径和编译配置文件。因此在 **device/oneplus/oneplus3/BoardConfig.mk** 中存放了相关的内核配置信息，如下所示:

```
BOARD_KERNEL_BASE := 0x80000000
BOARD_KERNEL_PAGESIZE := 4096
BOARD_KERNEL_TAGS_OFFSET := 0x02000000
BOARD_RAMDISK_OFFSET     := 0x02200000
BOARD_KERNEL_IMAGE_NAME := Image.gz-dtb
TARGET_KERNEL_SOURCE := kernel/oneplus/msm8996
TARGET_KERNEL_CONFIG := lineageos_oneplus3_defconfig


```

以上 TARGET_KERNEL_CONFIG 变量指定了 oneplus3 内核的编译配置文件名为:**lineageos_oneplus3_defconfig**。

在内核源码中编译配置文件一般存放在路径 **arch / 处理器平台 / configs** 下面。由于一加 3 手机为 arm64，所以在路径 **arch/arm64/configs** 下找到配置文件 **lineageos_oneplus3_defconfig**。如下所示:

```
qiang@ubuntu:~/myproject/kernel/oneplus3/msm8996/arch/arm64/configs$ pwd
/home/qiang/myproject/kernel/oneplus3/msm8996/arch/arm64/configs

qiang@ubuntu:~/myproject/kernel/oneplus3/msm8996/arch/arm64/configs$ ls -la lineageos_oneplus3_defconfig
-rw-rw-r-- 1 qiang qiang 15001 1月   3 23:00 lineageos_oneplus3_defconfig


```

2.  为编译配置添加内核可加载、卸载选项
    

由于编译内核模块的时候需要依赖于已经编译过的内核输出，并且内核需要配置为可加载才能正常编译内核模块。所以需要修改一下 **arch/arm64/configs/lineageos_oneplus3_defconfig**，添加如下配置项。

```
# 开启内核模块可加载
CONFIG_MODULES=y
# 开启内核模块可卸载
CONFIG_MODULE_UNLOAD=y


```

3.  配置编译内核源码脚本
    

脚本如下

```
# 设置编译平台为64位arm
export ARCH=arm64
export SUBARCH=arm64
# 配置arm64的交叉编译路径
export PATH=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin:$PATH
# 配置32位arm的交叉编译路径
# 测试过程中32位的不设置居然编译不过
export PATH=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin:$PATH

# 设置64位交叉编译工具前缀
# 这个前缀其实就是交叉编译链路径/home/qiang/lineageOs/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin 下面的工具的公共前缀
export CROSS_COMPILE=aarch64-linux-android-

# 设置32位交叉编译工具前缀
# 这个前缀其实就是交叉编译链路径/home/qiang/lineageOs/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin 下面的工具的公共前缀
export CROSS_COMPILE_ARM32=arm-linux-androideabi-

# 通过make命令生成编译配置文件.config
# O=out指定输出目录  lineageos_oneplus3_defconfig:这个是oneplus3的内核编译配置
make  O=out lineageos_oneplus3_defconfig

#  执行内核编译
make -j2 O=out ARCH=arm64


```

以上脚本在终端难的一个一个的输入，我将上面的弄成一个 **make.sh** 文件，到时候直接执行。make.sh 内容如下:

```
#!/bin/bash

# 切换到内核源码根目录去 
cd ./msm8996
make mrproper
make clean
export ARCH=arm64
export SUBARCH=arm64
export PATH=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin:$PATH
export PATH=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin:$PATH
export CROSS_COMPILE=aarch64-linux-android-
export CROSS_COMPILE_ARM32=arm-linux-androideabi-
make -j2 O=out lineageos_oneplus3_defconfig
make -j2  O=out ARCH=arm64


```

在终端执行 make.sh 之后就可以看到编译内核了，如下所示:

```
qiang@ubuntu:~/myproject/kernel/oneplus3$ ./make.sh 
make[1]: Entering directory '/home/qiang/myproject/kernel/oneplus3/msm8996/out'
  GEN     ./Makefile
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  HOSTCC  scripts/kconfig/zconf.tab.o
  HOSTLD  scripts/kconfig/conf



```

编译完成之后，可以在目录 / home/qiang/myproject/kernel/oneplus3/msm8996/out 下面找到编译产生的文件和内核镜像。

三、编写 helloworld 模块
------------------

此处编写一个简单的 HelloWorld 模块进行研究测试。

1.  创建 helloworld.c 模块源文件
    

文件代码如下:

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/sched.h>
static int __init hello_init(void){
   printk(KERN_ALERT "Hello World!\n");
   return 0;
}
static void __exit hello_exit(void){
   printk(KERN_ALERT "See You,Hello World!\n");
}
module_init(hello_init);
module_exit(hello_exit);


```

2.  创建模块编译配置文件 Makefile
    

Makefile 如下:

```
# 设置内核源码编译的输出目录
KERNEL_OUT=/home/qiang/myproject/kernel/oneplus3/msm8996/out
# 设置arm64交叉编译链工具路径
TOOLCHAIN=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android-
# 设置arm32交叉编译链工具路径
TOOLCHAIN32=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin/arm-linux-androideabi-
# 设置模块
obj-m := helloworld.o
# 编译命令配置 
all:
 make ARCH=arm64 CROSS_COMPILE_ARM32=$(TOOLCHAIN32)  CROSS_COMPILE=$(TOOLCHAIN) -C $(KERNEL_OUT) M=$(shell pwd)  modules
# 清理编译命令
clean:
 make -C $(KERNEL_OUT) M=$(shell pwd) clean


```

3.  在 helloworld 模块目录执行 **make** 命令进行编译
    

编译如下:

```
qiang@ubuntu:~/myproject/kernel/oneplus3/modules/helloworldmodule$ pwd
/home/qiang/myproject/kernel/oneplus3/modules/helloworldmodule

qiang@ubuntu:~/myproject/kernel/oneplus3/modules/helloworldmodule$ ls -la
total 16
drwxrwxr-x 2 qiang qiang 4096 1月   5 21:18 .
drwxrwxr-x 6 qiang qiang 4096 1月   5 21:17 ..
-rw-rw-r-- 1 qiang qiang  310 1月   5 21:06 helloworld.c
-rw-rw-r-- 1 qiang qiang  498 1月   5 21:08 Makefile

qiang@ubuntu:~/myproject/kernel/oneplus3/modules/helloworldmodule$ make
make ARCH=arm64 CROSS_COMPILE_ARM32=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin/arm-linux-androideabi-  CROSS_COMPILE=/home/qiang/lineageOs/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android- -C /home/qiang/myproject/kernel/oneplus3/msm8996/out M=/home/qiang/myproject/kernel/oneplus3/modules/helloworldmodule  modules
make[1]: Entering directory '/home/qiang/myproject/kernel/oneplus3/msm8996/out'
  CC [M]  /home/qiang/myproject/kernel/oneplus3/modules/helloworldmodule/helloworld.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/qiang/myproject/kernel/oneplus3/modules/helloworldmodule/helloworld.mod.o
  LD [M]  /home/qiang/myproject/kernel/oneplus3/modules/helloworldmodule/helloworld.ko
make[1]: Leaving directory '/home/qiang/myproject/kernel/oneplus3/msm8996/out'


```

模块编译好之后就可以 adb push 到手机，使用 insmod 加载模块进行测试验证了。以后就可以通过写内核系统 hook 模块进行系统调用内核层拦截、写 netfileter hook 模块进行网络管控等等操作。

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。关注公众号第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)