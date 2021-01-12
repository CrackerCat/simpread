> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/zhangjg_blog/article/details/84291663)

### 使用 Android 模拟器调试 linux 内核

*   *   [为什么需要调试 linux 内核](#linux_3)
    *   [如何在 Android 上调试内核](#Android_9)
    *   [开发环境](#_39)
    *   [创建模拟器](#_74)
    *   [下载 goldfish 内核源码](#goldfish_137)
    *   [编译 goldfish 内核](#goldfish_162)
    *   [编译内核遇到的问题](#_321)
    *   [使用自己编译的 linux 内核启动模拟器](#linux_439)
    *   [使用 gdb 调试内核](#gdb_483)
    *   [参考](#_659)

为什么需要调试 linux 内核
----------------

最近几年一直在学习 linux 内核，源码也看过一部分，但是没有系统的分析。正好最近想研究 Android 上的 sdcardfs 源码，想趁着这个机会把文件系统这部分好好梳理一下。 sdcardfs 虽然代码量不是很大，但是对于我目前对 linux 源码的熟悉程度，还是有一定的难度。所以需要能够断点调试，来跟踪内核的执行流程。通过断点调试，能够查看变量的值，能够查看调用栈。磨刀不误砍柴工，断点调试对于分析内核源码，能起到事半功倍的作用。

如何在 Android 上调试内核
-----------------

在 Android 上调试内核，一般都要借助于内核中的 kgdb。kgdb 是内核对 gdb 的支持，通过编译配置 kgdb 和其他相关配置，可以通过 gdb 远程连接到内核中的 kgdb。kgdb 只能远程调试，也就是说，要有一台被调试的机器 (target machine) 和一台开发机器(develop machine)。如果真机调试，target 就是手机，develop 就是 ubuntu 主机。如果是模拟器调试，target 就是模拟器，develop 是 ubuntu 主机。可以参考以下链接：

[Using kgdb, kdb and the kernel debugger internals](https://www.kernel.org/doc/html/v4.18/dev-tools/kgdb.html)

有两种调试环境可供选择。

一种方式是使用真机调试，另一种是使用模拟器。

真机调试也是可行的，网上比较靠谱的教程是这个:

[Practical Android Debugging Via KGDB](https://blog.trendmicro.com/trendlabs-security-intelligence/practical-android-debugging-via-kgdb/)

这种方式比较折腾。因为 ubuntu 主机要通过串口和手机相连，但是 Android 手机没有串口，所以这篇文章中介绍如何通过配置 usb，让 usb 当做串口来使用。并且需要修改 usb 的驱动，让 kgdb I/O 可以通过 usb 读写数据。并且作者也给出了 patch:

[KernelDebugOnNexus6P](https://github.com/jacktang310/KernelDebugOnNexus6P)

作者提供的是 kernel-3.10 的 patch，我手里有一台 pixel，运行 Android P，内核版本是 3.18。移植作者提供的 usb 驱动 patch 时，发现驱动代码有改动，连蒙带猜移植完之后，编译出的内核刷到手机上频繁重启。又加上自己根本不懂 usb 驱动，也不知道问题出在哪里。所以就放弃了。

网上还有一中方法，好像是通过耳机插孔模拟串口通信，对这种方式我也是一脸懵逼。需要懂一些硬件知识才搞的定:

[Building a Nexus 4 UART Debug Cable](https://www.optiv.com/blog/building-a-nexus-4-uart-debug-cable)

如果你想用真机调试，并且通过 usb 来模拟串口。需要你能移植 usb driver 的 patch，如果你有嵌入式开发的经验，应该不难搞定。或者编译和文中作者使用的同一版本的内核，这样移植起来应该不会出问题。

我是通过模拟器调试内核。虽然网上参考资料不少，但是自己在尝试的过程中还是遇到不少问题。下面我会详细的写出我搭建环境的流程，会列出遇到的问题，并且给出解决方案。如果你的环境跟我相同，按照同样的步骤应该也能成功。

开发环境
----

我的环境如下:

[Ubuntu 18.04 LTS (作为开发主机)](https://www.ubuntu.com/download/desktop)  
[AndroidStudio(用于下载和管理 SDK)](https://developer.android.com/studio/releases/)  
Android sdk (提供模拟器)  
[Android ndk (提供交叉编译环境)](https://developer.android.com/ndk/downloads/?hl=zh-cn)  
AOSP 中的 prebuilts/gdb/linux-x86 (提供 gdb 工具)

注：prebuilts/gdb/linux-x86 是 AOSP 中的一个 git 仓库，可以通过以下命令 clone 下来:  
`git clone https://android.googlesource.com/platform/prebuilts/gdb/linux-x86`

如果下载速度太慢，可以去 tuna 下载:

`git clone https://aosp.tuna.tsinghua.edu.cn/platform/prebuilts/gdb/linux-x86`

下载下来后，可以 check 到最新的 android-9.0.0_r18:

```
zhangjg@zjg:~/deve/aosp/prebuilts/gdb/linux-x86$ git checkout android-9.0.0_r18
注意：正在检出 'android-9.0.0_r18'。

您正处于分离头指针状态。您可以查看、做试验性的修改及提交，并且您可以通过另外
的检出分支操作丢弃在这个状态下所做的任何提交。

如果您想要通过创建分支来保留在此状态下所做的提交，您可以通过在检出命令添加
参数 -b 来实现（现在或稍后）。例如：

  git checkout -b <新分支名>

HEAD 目前位于 4adfde8 Update gdb prebuilt. am: 01e05a5d04 am: bc2150e0bf am: 8862ccbbd4 am: 172e21a7c8


```

创建模拟器
-----

首先打开 Android Studio，点击菜单中的这个按钮启动 sdk:  
![](https://img-blog.csdnimg.cn/20181120153442561.png)  
在 sdk 中，确保模拟器工具已经安装：  
![](https://img-blog.csdnimg.cn/20181120153710186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JyYXZlMjIxMQ==,size_16,color_FFFFFF,t_70)

然后点击 Android Studio 界面上的这个按钮启动 avd manager:  
![](https://img-blog.csdnimg.cn/2018112015405675.png)

在 avd manager 中点击界面上的`create virtual device`来创建模拟器:  
![](https://img-blog.csdnimg.cn/20181120154302382.png)

这里我创建的是一个 Pixel 2 XL:

![](https://img-blog.csdnimg.cn/20181120154422841.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JyYXZlMjIxMQ==,size_16,color_FFFFFF,t_70)

点击`next`，选择 API Level 28，ABI 为 x86_64(x86_64 的模拟器速度快很多，官方说比 arm 快 10 倍)：

![](https://img-blog.csdnimg.cn/2018112015474082.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JyYXZlMjIxMQ==,size_16,color_FFFFFF,t_70)

如果 Image 还没下载，需要先点击`Download`下载。选好之后，点击`next`到下一步:  
![](https://img-blog.csdnimg.cn/20181120154948494.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JyYXZlMjIxMQ==,size_16,color_FFFFFF,t_70)

我们为模拟器起名为`Pixel2XL-x86_64`，点击 finish 完成创建。

然后到你的 Sdk 目录中找 emulator 工具，并且将 emulator 工具放到 PATH 环境变量中：

```
zhangjg@zjg:/deve/tools/Sdk$ find . -name emulator -type f
./tools/emulator
./emulator/emulator

```

这里有两个 emulator，如果你之前下载过 emulator 工具，老版本的 emulator 放在 tools 目录中，新版本的放在 emulator 目录下，这里我们要用 emulator 目录下的那个。将 emulator 目录放到 PATH 环境变量中。打开~/.bashrc，向里面加入以下两句：

```
export ANDROID_SDK=/home/zhangjg/deve/tools/Sdk                                                                                                                                    
export PATH=$PATH:$ANDROID_SDK/emulator 

```

然后`source ~/.bashrc`，通过`emulator -list-avds`命令查看模拟器：

```
zhangjg@zjg:/deve/tools/Sdk$ source ~/.bashrc
zhangjg@zjg:/deve/tools/Sdk$ emulator -list-avds
Pixel2XL-x86_64

```

可以看到已经列出了我们之前创建的`Pixel2XL-x86_64`这个模拟器。然后验证能够启动:

```
zhangjg@zjg:/deve/tools/Sdk$ emulator -avd Pixel2XL-x86_64
Your emulator is out of date, please update by launching Android Studio:
 - Start Android Studio
 - Select menu "Tools > Android > SDK Manager"
 - Click "SDK Tools" tab
 - Check "Android Emulator" checkbox
 - Click "OK"

```

模拟器能够成功启动:

![](https://img-blog.csdnimg.cn/20181120160408927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JyYXZlMjIxMQ==,size_16,color_FFFFFF,t_70)

注意，这里启动模拟器，最好用`Sdk/emulator/emulator`，而不是`Sdk/tools/emulator`，如果用`Sdk/tools/emulator`，可能会报错。

下载 goldfish 内核源码
----------------

既然要调试 linux 内核，那么我们要下载和编译自己的内核，在内核中加入调试信息，并且启用 kgdb。

这里[编译内核](https://source.android.com/setup/build/building-kernels)列出了 google 针对不同硬件平台所支持的内核。模拟器使用的内核名为`goldfish`：

![](https://img-blog.csdnimg.cn/20181120161508836.png)  
所以我们通过以下命令下载`goldfish`内核:

`git clone https://android.googlesource.com/kernel/goldfish`

下载完成后，我们要切到对应的分支。首先查看我们创建的模拟器所使用的内核，我们要编译的内核最好和它原生使用的内核版本一致。通过`adb shell umane -a`可以查看模拟器的内核:

```
zhangjg@zjg:~$ adb shell uname -a
Linux localhost 4.4.124+ #1 SMP PREEMPT Wed Sep 26 01:14:51 UTC 2018 x86_64

```

可以看到，内核版本为 4.4。所以我们的 goldfish 源码要切到 4.4 分支`android-goldfish-4.4-dev`:

```
git checkout -b android-goldfish-4.4-dev remotes/origin/android-goldfish-4.4-dev

```

编译 goldfish 内核
--------------

下面我们编译 goldfish 内核，首先配置环境变量，主要是配置交叉编译工具。

[这里](https://developer.android.com/ndk/guides/standalone_toolchain?hl=zh-cn)列出了编不同平台的内核所对应的工具:  
![](https://img-blog.csdnimg.cn/20181120163413355.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JyYXZlMjIxMQ==,size_16,color_FFFFFF,t_70)

可以看到 x86_64 平台对应的是 x86_64-<gcc-version>，在上面我们已经下载了 android ndk，我们可以在 ndk 中找到对应的 gcc 工具：

```
zhangjg@zjg:/deve/tools/android-ndk-r16b/toolchains/x86_64-4.9/prebuilt/linux-x86_64/bin$ ll | grep gcc
-rwxr-xr-x 1 zhangjg zhangjg  809480 12月  1  2017 x86_64-linux-android-gcc*
-rwxr-xr-x 1 zhangjg zhangjg    1733 12月  1  2017 x86_64-linux-android-gcc-4.9*
-rwxr-xr-x 1 zhangjg zhangjg  809480 12月  1  2017 x86_64-linux-android-gcc-4.9.x*
-rwxr-xr-x 1 zhangjg zhangjg   25440 12月  1  2017 x86_64-linux-android-gcc-ar*
-rwxr-xr-x 1 zhangjg zhangjg   25408 12月  1  2017 x86_64-linux-android-gcc-nm*
-rwxr-xr-x 1 zhangjg zhangjg   25408 12月  1  2017 x86_64-linux-android-gcc-ranlib*

```

`x86_64-linux-android-gcc`就是我们编译内核所使用的 gcc，所以我们把`/deve/tools/android-ndk-r16b/toolchains/x86_64-4.9/prebuilt/linux-x86_64/bin`加到 PATH 环境变量中:

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ export PATH=/home/zhangjg/deve/tools/android-ndk-r16b/toolchains/x86_64-4.9/prebuilt/linux-x86_64/bin/:$PATH

```

然后指定要编译的内核的架构为`x86_64`:

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ export ARCH=x86_64
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ export CROSS_COMPILE=x86_64-linux-android-

```

下面配置内核。要能编出能够正常调试的内核，需要正确进行配置。首先看一下 google 给我们提供了什么默认配置:

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ find ./arch/x86/ -name *defconfig
./arch/x86/configs/x86_64_ranchu_defconfig
./arch/x86/configs/x86_64_defconfig
./arch/x86/configs/i386_ranchu_defconfig
./arch/x86/configs/i386_defconfig

```

这里我们选用`x86_64_ranchu_defconfig`。

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ make x86_64_ranchu_defconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  SHIPPED scripts/kconfig/zconf.tab.c
  SHIPPED scripts/kconfig/zconf.lex.c
  SHIPPED scripts/kconfig/zconf.hash.c
  HOSTCC  scripts/kconfig/zconf.tab.o
  HOSTLD  scripts/kconfig/conf
arch/x86/configs/x86_64_ranchu_defconfig:211:warning: override: reassigning to symbol IP_NF_TARGET_MASQUERADE
arch/x86/configs/x86_64_ranchu_defconfig:457:warning: override: reassigning to symbol DEVPORT
arch/x86/configs/x86_64_ranchu_defconfig:469:warning: override: USB_CONFIGFS changes choice state
arch/x86/configs/x86_64_ranchu_defconfig:476:warning: override: reassigning to symbol USELIB
arch/x86/configs/x86_64_ranchu_defconfig:477:warning: override: reassigning to symbol SYSVIPC
arch/x86/configs/x86_64_ranchu_defconfig:481:warning: override: reassigning to symbol DEVKMEM
arch/x86/configs/x86_64_ranchu_defconfig:482:warning: override: reassigning to symbol DEVMEM
arch/x86/configs/x86_64_ranchu_defconfig:483:warning: override: reassigning to symbol INET_LRO
#
# configuration written to .config
#

```

在源码根目录下生成了. config 文件。但是默认配置生成的. config 文件并不能满足我们的需求。我们要改一下这个. config 文件，确保以下配置:

```
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_FRAME_POINTER=y
CONFIG_KGDB=y 
CONFIG_DEBUG_RODATA=n
CONFIG_RANDOMIZE_BASE=n

```

`CONFIG_DEBUG_RODATA`这个选项我们虽然手动设置为 n，但是执行 make 后会被覆盖，所以我们要改以下两个文件，确保`CONFIG_DEBUG_RODATA`不开启:

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ git diff
diff --git a/arch/arm/mm/Kconfig b/arch/arm/mm/Kconfig
index 41218867a9a6..e67810313d97 100644
--- a/arch/arm/mm/Kconfig
+++ b/arch/arm/mm/Kconfig
@@ -1052,7 +1052,7 @@ config ARM_KERNMEM_PERMS
 config DEBUG_RODATA
        bool "Make kernel text and rodata read-only"
        depends on ARM_KERNMEM_PERMS
-       default y
+       default n
        help
          If this is set, kernel text and rodata will be made read-only. This
          is to help catch accidental or malicious attempts to change the
diff --git a/arch/x86/Kconfig b/arch/x86/Kconfig
index ad1f3bfafe75..50fa4dc68eff 100644
--- a/arch/x86/Kconfig
+++ b/arch/x86/Kconfig
@@ -307,7 +307,7 @@ config FIX_EARLYCON_MEM
        def_bool y
 
 config DEBUG_RODATA
-       def_bool y
+       def_bool n
 
 config PGTABLE_LEVELS
        int

```

注意，一定确保`CONFIG_DEBUG_RODATA`和`CONFIG_RANDOMIZE_BASE`不开启，如果开启这两个选项，通过 gdb 不能设置断点，报如下错误:

```
(gdb) b vfs_write
Breakpoint 1 at 0xffffffff803474d8: file fs/read_write.c, line 524.
(gdb) c
Continuing.
Warning:
Cannot insert breakpoint 1.
Cannot access memory at address 0xffffffff803474d8

```

下面通过`make -j8`（开启 8 线程编译）命令编译内核，执行`make -j8`后，会有一些命令行的选项让你手动选择，原则上是，能不选的就不选，选的多了也会出问题。下面是我的选择，按这个选择是没问题的。

```
Serial console over KGDB NMI debugger port (SERIAL_KGDB_NMI) [N/y/?] (NEW) N

```

```
KGDB: kernel debugger (KGDB) [Y/n/?] y
  KGDB: use kgdb over the serial console (KGDB_SERIAL_CONSOLE) [Y/n/m/?] (NEW) n
  KGDB: internal test suite (KGDB_TESTS) [N/y/?] (NEW) N
  KGDB: Allow debugging with traps in notifiers (KGDB_LOW_LEVEL_TRAP) [N/y/?] (NEW) N
  KGDB_KDB: include kdb frontend for kgdb (KGDB_KDB) [N/y/?] (NEW) N

```

之前我有些选项选错了，出现了很多问题。

并且执行`make -j8`后，.config 文件的内容还是会有所改变，`CONFIG_DEBUG_RODATA`和`CONFIG_RANDOMIZE_BASE`变成如下的配置:

```
# CONFIG_RANDOMIZE_BASE is not set
# CONFIG_DEBUG_RODATA is not set

```

这样是没问题的，编出的内核是可以使用的。

命令行输出如下信息，说明编译成功:

```
Kernel: arch/x86/boot/bzImage is ready  (#1)

```

在`arch/x86/boot/`中生成 bzImage 内核镜像:

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ ls arch/x86/boot/bzImage 
arch/x86/boot/bzImage

```

在内核源码根目录生成`vmlinux`文件:

```
zhangjg@zjg:~/deve/open_source/android-kernel/goldfish$ ls -lh vmlinux
-rwxr-xr-x 1 zhangjg zhangjg 173M 11月 20 17:19 vmlinux

```

编译内核遇到的问题
---------

下面是编译内核时遇到的**问题**:

**1 KGDB: waiting… or $3#33 for KDB**

如果`KGDB_KDB: include kdb frontend for kgdb (KGDB_KDB) [N/y/?]`选择了 y，或者在. config 文件中配置了`CONFIG_KGDB_KDB=y`，相当于开启了 KDB，内核启动时会有如下报错：

```
[  431.346565] BUG: unable to handle kernel NULL pointer dereference at           (null)
[  431.347386] IP: [<          (null)>]           (null)
[  431.347556] PGD 0 
[  431.347556] Oops: 0010 [#98098] SMP 
[  431.347556] KGDB: waiting... or $3#33 for KDB
[  431.347556] CR2: 0000000000000000
[  431.347556] ---[ end trace c7b14e8179f4cf53 ]---

```

应该是需要需要处理 KGDB 和 KDB 的切换，不知道怎么处理。关了 KDB 就可以正常启动了。

**2 KGDB: Waiting for remote debugger**

如果`KGDB: internal test suite (KGDB_TESTS) [N/y/?]`选择了 y，或者配置了`CONFIG_KGDB_TEST=y`，内核启动时如下报错，并且是循环报错:

```
[   86.189089] BUG: unable to handle kernel NULL pointer dereference at           (null)
[   86.189886] IP: [<          (null)>]           (null)
[   86.190096] PGD 0 
[   86.190096] Oops: 0010 [#18499] SMP 
[   86.190096] KGDB: Waiting for remote debugger
[   86.190096] CR2: 0000000000000000
[   86.190096] ---[ end trace 1b2e3d6555164c5f ]---

```

因为对 kgdb 原理还不了解，不知到问题原因，但是关闭`KGDB_TEST`就可以了

**3 无法设置断点**  
这个问题上面说过了，如果`CONFIG_DEBUG_RODATA`或`CONFIG_RANDOMIZE_BASE`没有关闭，断点是不能设置成功的：

```
(gdb) b vfs_write
Breakpoint 1 at 0xffffffff803474d8: file fs/read_write.c, line 524.
(gdb) c
Continuing.
Warning:
Cannot insert breakpoint 1.
Cannot access memory at address 0xffffffff803474d8

Command aborted.

```

解决的办法是确保`CONFIG_DEBUG_RODATA`或`CONFIG_RANDOMIZE_BASE`都关闭。方案上面也说过了

**4 bt 命令无法打印调用栈，next 单步执行跳转错误**

使用 bt 无法打印调用栈

```
#0  vfs_write (file=0xffff880038281200, buf=0x7ffd92804d90 "\001", count=8, pos=0xffff880047493f20) at fs/read_write.c:524
#1  0xffffffff80347ff3 in SYSC_write (count=<optimized out>, buf=<optimized out>, fd=<optimized out>) at fs/read_write.c:585
#2  SyS_write (fd=<optimized out>, buf=<optimized out>, count=<optimized out>) at fs/read_write.c:577
#3  0xffffffff809414a1 in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:192
#4  0x0000000013d93ae0 in ?? ()
#5  0x00007ffd92804f00 in ?? ()
#6  0x0000000000000007 in irq_stack_union ()
#7  0x000076a0a1ae03a0 in ?? ()
#8  0x0000000000000007 in irq_stack_union ()
#9  0x0000000013cc5778 in ?? ()

```

`next`命令跳转错误

```
Thread 4 hit Breakpoint 2, vfs_open (path=0xffff88003d073dc0, file=0xffff88000bf6a400, cred=0xffff88004b602000) at fs/open.c:855
855	{
(gdb) list
850	 * @file: newly allocated file with f_flag initialized
851	 * @cred: credentials to use
852	 */
853	int vfs_open(const struct path *path, struct file *file,
854		     const struct cred *cred)
855	{
856		struct inode *inode = vfs_select_inode(path->dentry, file->f_flags);
857	
858		if (IS_ERR(inode))
859			return PTR_ERR(inode);
(gdb) 
860	
861		file->f_path = *path;
862		return do_dentry_open(file, inode, NULL, cred);
863	}
864	
865	struct file *dentry_open(const struct path *path, int flags,
866				 const struct cred *cred)
867	{
868		int error;
869		struct file *f;
(gdb) n
read_hpet (cs=0xffffffff80e24a80 <clocksource_hpet>) at arch/x86/kernel/hpet.c:766
766		return (cycle_t)hpet_readl(HPET_COUNTER);
#10 0x0000000000000246 in irq_stack_union ()
#11 0x000000000000000b in irq_stack_union ()

```

在`vfs_open`起始处打断点，通过 n(next) 命令单步跟踪，应该调到`vfs_select_inode`，但是确调到了`arch/x86/kernel/hpet.c`中。

以上两个问题应该是 gcc 编译优化导致的 (-O2)，我尝试去掉优化，然而没有成功。

使用 - O0 编译内核是编不过的  
-O1, -O2 都是可以编过，但是还是会存在优化的问题  
这里我使用了 - Og 来编译，参考了 [kernel hacking: GCC optimization for better debug experience (-Og)](https://lwn.net/Articles/754219/)，移植了一些 patch，并且加入两个 Config 选项，编过之后能看到大多数的方法名和变量的值，但是单步跟踪还是会跳乱。

关于这个问题，我在 stackoverflow 上有提问，但是目前还没有解决:

[Next step error when debugging Android kernel](https://stackoverflow.com/questions/53478689/next-step-error-when-debugging-android-kernel)

stackoverflow 上还有一些相似的问题:  
[How to de-optimize the Linux kernel to and compile it with -O0?](https://stackoverflow.com/questions/29151235/how-to-de-optimize-the-linux-kernel-to-and-compile-it-with-o0?rq=1)  
[How to de-optimize the Linux kernel to avoid value optimized out](https://stackoverflow.com/questions/43336100/how-to-de-optimize-the-linux-kernel-to-avoid-value-optimized-out)  
[Linux cannot compile without GCC optimizations; implications?](https://unix.stackexchange.com/questions/153788/linux-cannot-compile-without-gcc-optimizations-implications)

使用自己编译的 linux 内核启动模拟器
---------------------

使用我们自己编译出的`bzImage`来启动模拟器

```
zhangjg@zjg:/deve/open_source/android-kernel/goldfish$emulator -avd Pixel2XL-x86_64 -show-kernel -verbose -wipe-data -netfast -kernel arch/x86/boot/bzImage  -qemu -s

```

模拟器能够启动成功。通过`adb shell uname -a`查看一下，运行的内核是否为我们刚编出的内核:

```
zhangjg@zjg:~$ adb shell uname -a
Linux localhost 4.4.124+ #1 SMP PREEMPT Tue Nov 20 17:19:44 CST 2018 x86_64

```

可以看到，时间为 2018-11-20 17:19:44，确实是刚编译的内核。

下面说一下 emulator 的各个参数的:

*   `-avd <name> use a specific android virtual device`  
    指定模拟器
    
*   `-show-kernel display kernel messages`  
    会打印内核信息
    
*   `-verbose same as '-debug-init`  
    会打印一些启动信息
    
*   `-wipe-data reset the user data image (copy it from initdata)`  
    清除数据。这里需要注意，如果不加这个选项，可能不能加载我们指定的内核。我就是因为没有运行我指定的内核，才加上这个选项。加上后就可以了。
    
*   `-netfast disable network shaping` 应该是关闭一些网络优化，不要篡改网络包，因为我们需要通过 tcp 连接 kgdb，如果不加这个选项可能会影响调试。从别人博客看的，需要加上这个选项。
    
    关于`network shaping`的介绍:[WHAT IS TRAFFIC SHAPING?](https://www.a10networks.com/resources/articles/traffic-shaping)
    
*   `-kernel use specific emulated kernel`  
    指定模拟器的内核，这里指定我们自己编译的内核`arch/x86/boot/bzImage`
    
*   `-qemu args... pass arguments to qemu`  
    传递 qemu 参数，emulator 就是基于`qemu`开发的
    
*   `-s`  
    是 qemu 参数，等同于`-gdb tcp::1234`，意思就是通过 tcp 的 1234 端口，gdb 可以连接到内核中的 kgdb。一般连接 kgdb 都要通过串口来连接，但是 qemu 通过指定 - gdb tcp::1234 就可以了，不知到原理是什么。
    

使用 gdb 调试内核
-----------

使用我们自己编译的内核启动模拟器后，现在我们使用`gdb`连接上内核进行调试。

这里要注意，我们使用的 gdb 命令必须是和体系结构对应的，比如，如果你编译的是 arm/arm64 的内核，使用 ubuntu 上的 x86_64 的 gdb 是不行的，因为我们编译的就是 x86_64 内核，所以可以直接使用本机上的 gdb 命令。但是为了安全起见，我们最好使用`aosp/prebuilts/gdb/linux-x86`里的 gdb，这个版本的 gdb 是兼容所有体系结构的。  
gdb 命令在`aosp/prebuilts/gdb/linux-x86/bin`目录中：

```
zhangjg@zjg:~/deve/aosp/prebuilts/gdb/linux-x86/bin$ ll gdb
-rwxr-xr-x 1 zhangjg zhangjg 94 11月 20 15:24 gdb*

```

将这个目录加到 PATH 中：

```
zhangjg@zjg:/deve/open_source/android-kernel/goldfish$ export PATH=/home/zhangjg/deve/aosp/prebuilts/gdb/linux-x86/bin:$PATH
zhangjg@zjg:/deve/open_source/android-kernel/goldfish$ which gdb
/home/zhangjg/deve/aosp/prebuilts/gdb/linux-x86/bin/gdb

```

首先运行`gdb /path/to/vmlinux`命令，如下:

```
zhangjg@zjg:/deve/open_source/android-kernel/goldfish$ gdb ./vmlinux 
GNU gdb (GDB) 7.11
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./vmlinux...done.
(gdb) 


```

这里的`./vmlinux`在 linux 内核源码的根目录中，内核编过之后，就会默认生成在内核源码的根目录中，有将近 200M，里面包含调试所用的符号信息。

接下来在`gdb`中使用`target remote :1234`来连接内核进行调试:

```
(gdb) target remote :1234
Remote debugging using :1234
warning: while parsing target description (at line 1): Could not load XML document "i386-64bit.xml"
warning: Could not load XML target description; ignoring
0xffffffff8023f26e in native_safe_halt () at ./arch/x86/include/asm/irqflags.h:49
49		asm volatile("sti; hlt": : :"memory");
(gdb) 


```

输出`Remote debugging using :1234`说明连接成功。

使用`b`(break) 命令在`sdcardfs_open`函数开始处打一个断点:

```
(gdb) b sdcardfs_open
Breakpoint 1 at 0xffffffff80478843: file fs/sdcardfs/file.c, line 223.

```

执行`c`(continue) 命令，让内核执行到断点处，这里的断点是在 sdcardfs 中 (如果没有执行到这个断点，可以在模拟器中打开相机应用拍张照片，拍完照肯定要存在 sdcard 中，所以会触发这里的断点)：

```
(gdb) c
Continuing.
[Switching to Thread 1]

Thread 1 hit Breakpoint 1, sdcardfs_open (inode=0xffff88000988c940, file=0xffff880039849100) at fs/sdcardfs/file.c:223
223	{


```

通过`list`命令查看当前的源码:

```
(gdb) list
218	out:
219		return err;
220	}
221	
222	static int sdcardfs_open(struct inode *inode, struct file *file)
223	{
224		int err = 0;
225		struct file *lower_file = NULL;
226		struct path lower_path;
227		struct dentry *dentry = file->f_path.dentry;


```

使用`bt`命令，查看当前的调用栈：

```
(gdb) bt
#0  sdcardfs_open (inode=0xffff88000988c940, file=0xffff880039849100) at fs/sdcardfs/file.c:223
#1  0xffffffff80381c42 in do_dentry_open (f=0xffff880039849100, inode=0xffff88000988c940, open=<optimized out>, cred=0xffff8800348130c0) at fs/open.c:749
#2  0xffffffff80382da8 in vfs_open (path=0xffff8800069ebdd0, file=0xffff880039849100, cred=0xffff8800348130c0) at fs/open.c:862
#3  0xffffffff80390e91 in do_last (nd=0xffff8800069ebdd0, file=<optimized out>, op=0xffff8800069ebef4, opened=<optimized out>) at fs/namei.c:3221
#4  0xffffffff8039109c in path_openat (nd=0xffff8800069ebdd0, op=0xffff8800069ebef4, flags=65) at fs/namei.c:3357
#5  0xffffffff803937b7 in do_filp_open (dfd=<optimized out>, pathname=<optimized out>, op=0xffff8800069ebef4) at fs/namei.c:3392
#6  0xffffffff80383149 in do_sys_open (dfd=159959360, filename=<optimized out>, flags=<optimized out>, mode=<optimized out>) at fs/open.c:1038
#7  0xffffffff8038328e in SYSC_openat (mode=<optimized out>, flags=<optimized out>, filename=<optimized out>, dfd=<optimized out>) at fs/open.c:1065
#8  SyS_openat (dfd=<optimized out>, filename=<optimized out>, flags=<optimized out>, mode=<optimized out>) at fs/open.c:1059
#9  0xffffffff80a8b7a1 in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:192


```

栈信息还是看的比较清晰的，对我们跟踪源码会有很大帮助。

使用`p`(print) 命令打印变量的信息：

```
(gdb) p *file
$1 = {f_u = {fu_llist = {next = 0x0 <irq_stack_union>}, fu_rcuhead = {next = 0x0 <irq_stack_union>, func = 0x0 <irq_stack_union>}}, f_path = {
    mnt = 0xffff880038d0f360, dentry = 0xffff88003602c780}, f_inode = 0xffff88000988c940, f_op = 0xffffffff80c45600 <sdcardfs_main_fops>, f_lock = {{rlock = {
        raw_lock = {val = {counter = 0}}}}}, f_count = {counter = 1}, f_flags = 33345, f_mode = 98334, f_pos_lock = {count = {counter = 1}, wait_lock = {{rlock = {
          raw_lock = {val = {counter = 0}}}}}, wait_list = {next = 0xffff880039849150, prev = 0xffff880039849150}, owner = 0x0 <irq_stack_union>, osq = {tail = {
        counter = 0}}}, f_pos = 0, f_owner = {lock = {raw_lock = {cnts = {counter = 0}, wait_lock = {val = {counter = 0}}}}, pid = 0x0 <irq_stack_union>, 
    pid_type = PIDTYPE_PID, uid = {val = 0}, euid = {val = 0}, signum = 0}, f_cred = 0xffff8800348130c0, f_ra = {start = 0, size = 0, async_size = 0, ra_pages = 0, 
    mmap_miss = 0, prev_pos = 0}, f_version = 0, f_security = 0xffff880037f02080, private_data = 0x0 <irq_stack_union>, f_ep_links = {next = 0xffff8800398491d8, 
    prev = 0xffff8800398491d8}, f_tfile_llink = {next = 0xffff8800398491e8, prev = 0xffff8800398491e8}, f_mapping = 0xffff88000988ca98}
(gdb) p *inode
$2 = {i_mode = 33277, i_opflags = 4, i_uid = {val = 0}, i_gid = {val = 1015}, i_flags = 2, i_acl = 0xffffffffffffffff, i_default_acl = 0xffffffffffffffff, 
  i_op = 0xffffffff80c45780 <sdcardfs_main_iops>, i_sb = 0xffff8800483c4800, i_mapping = 0xffff88000988ca98, i_security = 0xffff88003a3e1f00, i_ino = 36698, {
    i_nlink = 1, __i_nlink = 1}, i_rdev = 0, i_size = 0, i_atime = {tv_sec = 1543306447, tv_nsec = 236000000}, i_mtime = {tv_sec = 1543306447, tv_nsec = 236000000}, 
  i_ctime = {tv_sec = 1543306447, tv_nsec = 236000000}, i_lock = {{rlock = {raw_lock = {val = {counter = 0}}}}}, i_bytes = 0, i_blkbits = 12, i_blocks = 0, 
  i_state = 0, i_mutex = {count = {counter = 1}, wait_lock = {{rlock = {raw_lock = {val = {counter = 0}}}}}, wait_list = {next = 0xffff88000988c9f0, 
      prev = 0xffff88000988c9f0}, owner = 0x0 <irq_stack_union>, osq = {tail = {counter = 0}}}, dirtied_when = 0, dirtied_time_when = 0, i_hash = {
    next = 0x0 <irq_stack_union>, pprev = 0xffff88004b995440}, i_io_list = {next = 0xffff88000988ca30, prev = 0xffff88000988ca30}, i_lru = {
    next = 0xffff88000988ca40, prev = 0xffff88000988ca40}, i_sb_list = {next = 0xffff88004565a368, prev = 0xffff8800483c4e08}, {i_dentry = {
      first = 0xffff88003602c830}, i_rcu = {next = 0xffff88003602c830, func = 0xffffffff8047b59b <i_callback>}}, i_version = 2, i_count = {counter = 1}, 
  i_dio_count = {counter = 0}, i_writecount = {counter = 1}, i_fop = 0xffffffff80c45600 <sdcardfs_main_fops>, i_flctx = 0x0 <irq_stack_union>, i_data = {
    host = 0xffff88000988c940, page_tree = {height = 0, gfp_mask = 34078752, rnode = 0x0 <irq_stack_union>}, tree_lock = {{rlock = {raw_lock = {val = {
              counter = 0}}}}}, i_mmap_writable = {counter = 0}, i_mmap = {rb_node = 0x0 <irq_stack_union>}, i_mmap_rwsem = {count = 0, wait_list = {
        next = 0xffff88000988cac8, prev = 0xffff88000988cac8}, wait_lock = {raw_lock = {val = {counter = 0}}}, osq = {tail = {counter = 0}}, 
      owner = 0x0 <irq_stack_union>}, nrpages = 0, nrshadows = 0, writeback_index = 0, a_ops = 0xffffffff80c45d80 <sdcardfs_aops>, flags = 37880010, private_lock = {{
        rlock = {raw_lock = {val = {counter = 0}}}}}, private_list = {next = 0xffff88000988cb18, prev = 0xffff88000988cb18}, private_data = 0x0 <irq_stack_union>}, 
  i_devices = {next = 0xffff88000988cb30, prev = 0xffff88000988cb30}, {i_pipe = 0x0 <irq_stack_union>, i_bdev = 0x0 <irq_stack_union>, 
    i_cdev = 0x0 <irq_stack_union>, i_link = 0x0 <irq_stack_union>}, i_generation = 0, i_fsnotify_mask = 0, i_fsnotify_marks = {first = 0x0 <irq_stack_union>}, 
  i_private = 0x0 <irq_stack_union>}

```

但是当使用`n`(next) 命令单步执行的时候，会出现跳转错误，总是调到错误的位置：

```
(gdb) n
read_hpet (cs=0xffffffff81025440 <clocksource_hpet>) at arch/x86/kernel/hpet.c:766
766		return (cycle_t)hpet_readl(HPET_COUNTER);

```

这个问题目前没有解决，**如果谁有解决方案，请一定要告诉我**

如果单步调试正常，这个方案就完美了。

最后使用`q`(quit) 命令退出 gdb：

```
(gdb) q
A debugging session is active.

	Inferior 1 [Remote target] will be detached.

Quit anyway? (y or n) y
Detaching from program: /deve/open_source/android-kernel/goldfish/vmlinux, Remote target
Ending remote debugging.


```

其他 gdb 命令请参考 gdb 教程。

参考
--

[Android Linux 内核编译调试](http://www.joenchen.com/archives/1093)  
[用 GDB 远程调试 Linux 内核](https://www.jianshu.com/p/02557f0d29dc)  
[gdb 调试利器](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/gdb.html)