> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [pwn4.fun](http://pwn4.fun/2019/10/15/Android10%E5%86%85%E6%A0%B8%E8%B0%83%E8%AF%95/)

> 编译 Android10.0.0_r21. 选择 aosp_x86_64-eng 版本，因为编译成 aosp_arm64-eng 的启动不了。

[](#编译Android10-0-0-r2 "编译Android10.0.0_r2")编译 Android10.0.0_r2
---------------------------------------------------------------

1. 选择 aosp_x86_64-eng 版本，因为编译成 aosp_arm64-eng 的启动不了。  
2.emulator 启动

```
emulator: ERROR: x86_64 emulation currently requires hardware acceleration!
Please ensure KVM is properly installed and usable.
CPU acceleration status: This user doesn't have permissions to use KVM (/dev/kvm)
```

解决方法：  
（1）安装 kvm

```
$ sudo apt-get install qemu-kvm cpu-checker
$ kvm-ok
  INFO: /dev/kvm exists
  KVM acceleration can be used
```

（2）创建 kvm 用户组并把当前用户加入

```
$ sudo addgroup kvm
$ sudo usermod -a -G kvm fanrong
$ sudo chgrp kvm /dev/kvm
$ sudo vim /etc/udev/rules.d/60-qemu-kvm.rules
KERNEL=="kvm", GROUP="kvm", MODE="0660"
```

（3）重启运行 emulator  
可以成功启动模拟器，但此时的内核使用的是默认内核:

```
$ adb shell uname -r
4.14.112+
```

[](#编译goldfish内核 "编译goldfish内核")编译 goldfish 内核
----------------------------------------------

接下来下载 goldfish 内核编译，goldfish 是专门供模拟器使用的内核。

```
$ git clone https://android.googlesource.com/kernel/goldfish
$ cd goldfish
$ git branch -a # 查看所有版本
remotes/origin/HEAD -> origin/master
remotes/origin/android-3.18
...
$ git checkout -t origin/android-goldfish-4.14-dev # 切换到所需分支版本
```

AOSP 里有编译工具链，下面开始编译过程：

```
$ export CROSS_COMPILE=x86_64-linux-android-
$ export ARCH=x86_64
$ export PATH=$PATH:/aosp/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9/bin
$ make x86_64_ranchu_defconfig # 网上说模拟器使用的是ranchu这个默认配置，不知道代表的啥意思
```

这样就能在根目录生成. config 文件，但是这个默认的. config 文件编译后调试会出问题，需要如下修改：

```
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_FRAME_POINTER=y
CONFIG_KGDB=y 
CONFIG_DEBUG_RODATA=n
CONFIG_RANDOMIZE_BASE=n
```

一定确保 CONFIG_DEBUG_RODATA 和 CONFIG_RANDOMIZE_BASE 不开启，如果开启这两个选项，通过 gdb 不能设置断点，报如下错误:

```
(gdb) b vfs_write
Breakpoint 1 at 0xffffffff803474d8: file fs/read_write.c, line 524.
(gdb) c
Continuing.
Warning:
Cannot insert breakpoint 1.
Cannot access memory at address 0xffffffff803474d8
```

这样配置编译后可以调试，但是调试时没有符号信息，原因是`Makefile`里的编译选项为`-O2`，但是修改为`-O0`会编译不过，[解决方法](https://lwn.net/Articles/754219/)是改成`-Og`。

```
$ make -j8
$ cd aosp
$ emulator -show-kernel -kernel ../goldfish/arch/x86/boot/bzImage -qemu -s
```

emulator 是基于 qemu 开发的，`-s`是 qemu 参数，等同于`-gdb tcp::1234`，意思就是通过 tcp 的 1234 端口，gdb 可以连接到内核中的 kgdb。一般连接 kgdb 都要通过串口来连接，但是 qemu 通过指定`-gdb tcp::1234`就可以了。

[](#使用gdb调试内核 "使用gdb调试内核")使用 gdb 调试内核
-------------------------------------

gdb 使用的是 aosp/prebuilts/gdb/linux-x86/bin 里的 gdb，它是兼容所有体系结构的：

```
$ aosp/prebuilts/gdb/linux-x86/bin/gdb vmlinux
gdb-peda$ target remote :1234
...
native_safe_halt() at ./arch/x86/include/asm/irqflags.h:58
58     }
gdb-peda$ break sdcardfs_open
Breakpoint 1 at 0xffffffff8046ad25: file fs/sdcardfs/file.c, line 231.
gdb-peda$ c
Continuing.
```

这里我安装了 peda，在`sdcardfs_open`函数下断点，继续运行，点击相机的照相功能，拍照后会保存到 sdcard，会调用 sdcardfs_open 函数，触发断点，然后就可以单步调试啦。

**reference**  
[https://blog.csdn.net/zhangjg_blog/article/details/84291663](https://blog.csdn.net/zhangjg_blog/article/details/84291663)  
[https://gist.github.com/yan12125/78a9004acb1bed5faf2ffd442163e2ef](https://gist.github.com/yan12125/78a9004acb1bed5faf2ffd442163e2ef)  
[http://pwn4.fun/2016/08/19/Android%E5%86%85%E6%A0%B8%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91%E8%B0%83%E8%AF%95/](http://pwn4.fun/2016/08/19/Android%E5%86%85%E6%A0%B8%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91%E8%B0%83%E8%AF%95/)  
[https://lwn.net/Articles/754219/](https://lwn.net/Articles/754219/)