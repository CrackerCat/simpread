> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xweiwei.github.io](https://xweiwei.github.io/post/kernel_debug/)

Published on: 4 May 2017

Tags: [linux](https://xweiwei.github.io/tags/linux) [debug](https://xweiwei.github.io/tags/debug)

在分析 Linux 的内核 CVE，或者是编写相关的 POC，有时候可能结果与预期不太符合，这个时候就需要查看内核的运行过程。

kgdb 提供了一种使用 gdb 调试 Linux 内核的机制。使用 KGDB 可以象调试普通的应用程序那样，在内核中进行设置断点、检查变量值、单步跟踪程序运行等操作。

首先是去`https://source.android.com/source/building-kernels`下载对应的内核源代码。

msm 对应大部分的 nexus 以及 pixel 手机 `git clone https://android.googlesource.com/kernel/msm.git`

下面以 nexus5x(bullhead) 为例子, 生成一个. config

```
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-android-
$ cd msm
$ git checkout android-msm-bullhead-3.10-nougat
$ make bullhead_defconfig



```

然后配置. config

```
$ make menuconfig 


```

主要开启

```
[*] Kernel hacking 
     [*] Compile the kernel with debug info 
     [*] KGDB: kernel debugging with remote gdb —>
[*] Enable dynamic printk() call support 



```

编译得到的 arch/arm64/boot/Image

模拟器启动 `emulator -verbose -show-kernel -netfast -avd -avd Pixel_API_25 -kernel arch/arm64/boot/Image -qemu -gdb tcp::1234,ipv4 -S`

-verbose -show-kernel 选项可以看到内核的详细输出，-no-window -no-audio 选项可以不启动界面，-qumu -s -S 选项可以启动调试监听让内核启动时在端口 1234 等待。

利用目标平台的 gdb 去加载 vmlinux

```
aarch64-linux-android-gdb ./vmlinux
target remote localhost:1234


```

内核已经跑起来了，这个时候直接 c 就可以继续执行了。

但是在调试的时候想查看具体的某个变量的时候，会发现：

```
(gdb) p copied
$1 = <optimized out>



```

但是在编译的时候默认是 - O2 优化，所以查看变量时会出现`optimized out`

> gdb 调试程序的时候打印变量值会出现 情况, 可以在 gcc 编译的时候加上 -O0 参数项, 意思是不进行编译优化, 调试的时候就会顺畅了, 运行流程不会跳来跳去的, 发布项目的时候记得不要在使用 -O0 参数项, gcc 默认编译或加上 - O2 优化编译会提高程序运行速度.

[gdb 关于 value optimized out](http://dsl000522.blog.sohu.com/180439264.html)

并且很多时候程序跑的时候，代码也是对应不上的。

尝试用 - O0 的选项重新编译内核。

这个时候，编译内核代码会出现问题

```
In file included from ./arch/arm64/include/asm/bitops.h:19:0,
                 from include/linux/bitops.h:36,
                 from arch/arm64/mm/context.c:20:
In function '__xchg',
    inlined from 'flush_context' at arch/arm64/mm/context.c:58:10:
include/linux/compiler.h:437:38: error: call to '__compiletime_assert_67' declared with attribute error: BUILD_BUG failed
  _compiletime_assert(condition, msg, __compiletime_assert_, __LINE__)
                                      ^
include/linux/compiler.h:420:4: note: in definition of macro '__compiletime_assert'
    prefix 
    ^
include/linux/compiler.h:437:2: note: in expansion of macro '_compiletime_assert'
  _compiletime_assert(condition, msg, __compiletime_assert_, __LINE__)
  ^
include/linux/bug.h:50:37: note: in expansion of macro 'compiletime_assert'
 
                                     ^
include/linux/bug.h:84:21: note: in expansion of macro 'BUILD_BUG_ON_MSG'
 
                     ^
./arch/arm64/include/asm/cmpxchg.h:67:3: note: in expansion of macro 'BUILD_BUG'
   BUILD_BUG();
   ^
In function '__xchg',
    inlined from 'check_and_switch_context' at arch/arm64/mm/context.c:153:9:
include/linux/compiler.h:437:38: error: call to '__compiletime_assert_67' declared with attribute error: BUILD_BUG failed
  _compiletime_assert(condition, msg, __compiletime_assert_, __LINE__)
                                      ^
include/linux/compiler.h:420:4: note: in definition of macro '__compiletime_assert'
    prefix 
    ^
include/linux/compiler.h:437:2: note: in expansion of macro '_compiletime_assert'
  _compiletime_assert(condition, msg, __compiletime_assert_, __LINE__)
  ^
include/linux/bug.h:50:37: note: in expansion of macro 'compiletime_assert'
 
                                     ^
include/linux/bug.h:84:21: note: in expansion of macro 'BUILD_BUG_ON_MSG'
 
                     ^
./arch/arm64/include/asm/cmpxchg.h:67:3: note: in expansion of macro 'BUILD_BUG'
   BUILD_BUG();



```

内核代码很多地方就是依赖优化的，如果关闭的话，编译不过。

编译 kernel 不使用 - O2 的补丁，不过，没有尝试这个方法。 [build kernel without -O2 option](https://sourceware.org/ml/gdb/2010-12/msg00009.html)

最后一个可以靠谱的方法就是在部分代码中关闭优化。

[Android Linux 内核编译调试](http://www.joenchen.com/archives/1093)

[内核调试](http://freemandealer.github.io/2014/11/29/debug-android-kernel/)

[KGDB + Eclipse debug Kernel on i.mx6 Android](http://fatalfeel.blogspot.com/2013/09/kgdb-eclipse-debug-kernel-on-imx6.html)

[Practical Android Debugging Via KGDB](http://blog.trendmicro.com/trendlabs-security-intelligence/practical-android-debugging-via-kgdb/)

[用 kGDB 调试 Linux 内核](http://tinylab.org/kgdb-debugging-kernel/)

[Enabling KGDB for Android](http://bootloader.wikidot.com/android:kgdb)

[Android 内核编译调试](https://geneblue.github.io/2016/01/27/Android%E5%86%85%E6%A0%B8%E7%BC%96%E8%AF%91%E8%B0%83%E8%AF%95/)

[KGDB 在 ARM 平台上的交叉编译和使用](http://blog.hibeiyu.com/archives/432)