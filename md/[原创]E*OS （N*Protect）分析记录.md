> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267587.htm)

E*OS N*Protect 分析记录

前段时间想玩下这个游戏，模拟器运行时候发现有 root 检测，想看下它的实现，准备调试过掉，但是没有 X86 库，又发现之前的调试手机版本太低了，手上有个小米 8，然后经历重新下载源码，编译 LineageOs17.1，刷机，调试，根据调试情况修改系统源码，中间碰到盲区，又要去翻对应知识点，中间还被其它事打断过，断断续续持续了很长时间，写个文档做下记录。

**1.** **软硬件环境** (游戏有点大, 就不挂附件了)

EOS 2.2.87 版本：[https://apkpure.com/kr/search?q=EOS&t=app](https://apkpure.com/kr/search?q=EOS&t=app)

IDA 7.5

Frida 14.2.2

Gda3.86

JEB

LineageOs 17.1 (android 10)

小米 8

**2.** **流水账 (****直接按实际经历的时间顺序来了)**

最开始拿到 APK 后，用 GDA 打开

![](https://bbs.pediy.com/upload/attach/202105/44250_J54GENGA8MAEN8P.jpg)

![](https://bbs.pediy.com/upload/attach/202105/44250_GBYFW99NZ8GDPK6.jpg)

直接翻了下各个信息，有函数名称混淆，字符串加密等，看起来是有壳的，直接搜相关信息，搜到这篇：[https://blog.csdn.net/weixin_30512785/article/details/99559394](https://blog.csdn.net/weixin_30512785/article/details/99559394)。

这个是旧版本的 Nprotect，新版本跟这个比起来，dex 和 so 的保护强度都加强了，包括 so 文件（抹去了文件头等信息，干扰静态分析），so 的加载（libengine.so 没有使用系统 API 装载），符号表的加密，信息流也不再直来直去，还有使用了 Ollvm(Obfuscator-LLVM clang version 4.0.1) 等。

新旧版本虽然有很多不同了，并且跟我看的游戏也不同，但是仍然有很多相似的地方，这篇文章还是有所帮助的，省了不少时间，非常感谢这位作者。

我是想看下这个 root 检测，那篇没提到，可能是旧版本没有，就准备自己看了，这中间穿插了编译系统刷机，也是一番波折，还好系统都刷好了。

模拟器中开这个游戏的提示：

![](https://bbs.pediy.com/upload/attach/202105/44250_R3J4R22UWVTQJBC.jpg)

用 uiautomatorviewerStrong 看了下，没找到切入点，想通过提示字符串查找关键点，搜不到，字符串都加密了，并且直接打开的 APK JAVA 代码也是不全的，接着就准备 dump class 文件了，这个用 github 上的 frida_dump 脚本就行，会得到 8 个 dex 文件:

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202105/44250_ENFG32VYPBN974F.jpg)

然后就是反编译，翻代码，对我这边相关代码是在 class2.dex 中，发现字符串都是加密的，

![](https://bbs.pediy.com/upload/attach/202105/44250_UQAYMUMM49FSXR2.jpg)

加密代码都是比较简单的（so 里面的字符串也是加密的，不过是 AES 的）, 不过要注意的是，这个加密函数不止一个，不同的字符串可能用的函数不同的，算法类似，只是里面的 2 个 XOR 常量不同：

    **static** **public** String **IiIIiiIiii**(String p0){ 

       int vi;

       int ilength = p0.length();

       char[] ocharArray = **new** char[ilength];

       ilength = ilength-**1**;

       **while** (ilength>= **0**) {   

          vi = ilength-**1**;

          ocharArray[ilength]=(char)(p0.charAt(ilength)^**0x3c**);

          **if** (vi>= **0**) {

             ilength = vi-**1**;

             ocharArray[vi]=(char)(p0.charAt(vi)^**0x60**);

          }**else** {

             **break** ; 

          } 

       }

       **return** **new** String(ocharArray);

}

后面就是找到感兴趣的类，写程序解密字符串了，最后找到这个提示点：

package com.inca.security.Core;

public class AppGuardEngine implements WeakRefHandler$IOnHandleMessage, BaseEventInvoker

这个类中的代码：

![](https://bbs.pediy.com/upload/attach/202105/44250_H876DRM2DJSJXAN.jpg)

This app will be terminated because a security policy violation has been detected!

后面 code 显示是 10 进制。

这里要说下 GDA 确实强，下面这个 JEB 反编译不出来的：

![](https://bbs.pediy.com/upload/attach/202105/44250_W3A4276YEZZGRNZ.jpg)

这里面包括各个检测码的定义，还原字符串后是这样的：

![](https://bbs.pediy.com/upload/attach/202105/44250_56YWQEY2R5G8YSP.jpg)

根据这个看，34 正好就是 DETECT_ROOTING_ENVIRONMENT，表示检测到 root 环境了，顺便提下，翻代码过程中，发现几个反调试的地方：

类名： com.inca.security.IiIIiiiiIi

Debug.isDebuggerConnected()：

    public boolean iiIIIiiiIi() {

        return Debug.isDebuggerConnected();

    }

通过执行时间差 Debug.threadCpuTimeNanos()，判断是否被调试

做 100W 次加法计算，看时间是否超过 100ms (100000000 纳秒)

    public boolean iIIIiiiIII() {

        boolean v0 = false;

        long v4 = Debug.threadCpuTimeNanos();

        int v1 = 0;

        int v2;

        for(v2 = 0; v1 < 1000000; v2 = v1) {

            v1 = v2 + 1;

        }

        if(Debug.threadCpuTimeNanos() - v4 >= 100000000) {

            v0 = true;

        }

        return v0;

}

通过引用分析，最后确定出检测框的入口是这个函数：

public void conditionCallback(int arg23, int arg24, byte[] arg25)

然后上 frida，hook 这个调用点

MainActivity = Java.use('com.inca.security.Core.AppGuardEngine');

if (MainActivity != null) {       

     MainActivity.conditionCallback.implementation = function (arg0, arg1, arg2) {

                    //send('Statr! Hook!'); //python call back

                    console.log("call conditionCallback");

                    console.log(arg0);

                    console.log(arg1);

                    console.log(arg2);

                    showStacks();

                    return this.conditionCallback(arg0, arg1, arg2);               

}

}

frida -U -l E:\node_proj\TcpsocketTest\fridaHook2.js -f com.bluepotiongames.eosm

Spawned `com.bluepotiongames.eosm`. Use %resume to let the main thread start executing!

[MI 8::com.bluepotiongames.eosm]-> hook_eos();

[MI 8::com.bluepotiongames.eosm]-> %resume

发现很快就退出了，到不了提示窗口那，那就是被检测到了，要上动态调试了。

这个启动后是 3 个进程的，互相 ptrace，还是用 frida 启动进程方式，然后通过 IDA 附加游戏进程调试。

保护相关的 SO 是下面几个：

libcompatible.so

libstub.so

libengine-hlp.so

libengine.so

首先上来肯定就是找 JNI_OnLoad 了，直接跑 IDC 脚本：

    //android 10(lineage 17.1)

    //LoadNativeLibrary 偏移: 0000007BAE70AC70 - 0000007BAE395000 = 375C70

    auto soBase=0;

    soBase=getModuleBase("libart.so");

    auto addrArtBp=soBase + 0x375C70;

    MakeComm(addrArtBp,"LoadNativeLibrary");   

    auto addrArtCallOnload=soBase + 0x376910;

    AddBpt(addrArtCallOnload); 

    MakeComm(addrArtCallOnload,"call JNI_ONLOAD");

进到 libcompatible.so 的 JNI_OnLoad：

![](https://bbs.pediy.com/upload/attach/202105/44250_8K93ZH6MWY69AGE.jpg)

有些代码是运行中解压的，这种可以调试时候 dump 对应的数据下来，然后合并到原 so 文件中，可以方便 IDA 分析。

JNI_OnLoad 里面包含反调试的相关处理，检查 status 信息，fork 子进程互相 ptrace，注册 inotify_add_watch，包含下面几个检测：

/proc/xxx/mem

/proc/xxx/maps

/dev/input      这里下面 3 个是调试 libengine.so 发现的

/system/bin/input

/system/bin/monkey

![](https://bbs.pediy.com/upload/attach/202105/44250_59UQGM3FEUSAAUJ.jpg)

最后走的都是 svc 调用.

这些检测，直接改内核过滤掉了，status 的相关修改网上都很多，关于这个 inotify_add_watch 的修改，在 fs/notify/ 下:

/fs/notify/inotify/inotify_user.c 中的 inotify_add_watch

（这里修改给自己挖了个坑，只过滤了了主进程的，导致线程触发的还是被检测了，下面会提到，根本原因是子线程的 current->parent task_struct 结构直接是主进程的 parent 的了，导致得到的进程名称不是预想的本进程名称，而是父进程的父进程的名称）

处理 ptrace:

bionic/libc/bionic/ptrace.cpp

对 svc 调用的要改内核部分 ptrace.c 中的 ptrace_attach，对这个进程都直接返回就行。

这里实际调试花了点时间，中间也输出过 so 里面各个 jni 接口函数（包括后续 libstub.so）,

这里提下，libstub.so 中的.init_proc 会调用 libcompatible.so 中的导出函数 SoLibraryStart 来解密代码，

后来发现这里不是主要关注点，就换了个思路。

处理完上面检测后，就可以继续跑 hook conditionCallback 脚本了，这个时候就可以显示堆栈了：

java.lang.Exception  
        at com.inca.security.Core.AppGuardEngine.conditionCallback(Native Method)

GetMethodID   Pid: 11681Path: conditionCallback

Backtrace:

0x76046fe75c

0x7635df2808

0x7635df2808

然后 IDA 附加上去，去到 0x76046fe75c，通过对比分析，确定了这就是 libengine.so 的代码空间，可以 dump 出来用 IDA 打开，然后动态静态结合看，内存中文件头都没了，IDA 里面定位有点麻烦，这里可以把调试得到的相关偏移写到 idc 脚本中，每次新的调试，跑下脚本，就可以识别出之前已经分析的点。

![](https://bbs.pediy.com/upload/attach/202105/44250_S33FEEDNCZTKSEX.jpg)

双击输出的地址，就可以到达对应代码点，还是很方便的

通过 0x76046fe75c 这个点只是搞清楚了检测码的读取，写入是另外的线程，这里读取到后，就会调用 java 显示提示窗口了。

因为前面修改 inotify_add_watch 的坑，准备这里入手找切入点的时候，发现会跑飞。

后来尝试了几种方法，没找到切入点，还是回到系统修改上，直接在 pthread_create 那输出了线程入口地址，然后结合上面堆栈输出的地址，确定了几个相关的线程地址，然后修改修改 libengine.so 文件的入口地址为 00 00 00 14，直接入口循环，然后附加，还原入口代码并断点，确定相关的入口点：

![](https://bbs.pediy.com/upload/attach/202105/44250_BVQ9PBN4ZPC8PYY.jpg)

Root 相关检测就是这个线程了，跑起来后遇到新的问题，就是不是提示检测码 34，而是 9，查找之前的码表：

       stringArray[7]=("DETECT_INVALID_LIBDVM_SO");

       stringArray[8]=("detectinvVALID_LIBRUNTIME_SO");

       stringArray[9]=("DETECT_INVALID_APPLIB_SO");

       stringArray[10]=("DETECT_INVALID_LIBENGINE_SO");

9 就是 DETECT_INVALID_APPLIB_SO，看起来是修改了 libengine.so 导致被检测到了，不过现在已经确定了线程入口点了，后面直接顺着调试了。

![](https://bbs.pediy.com/upload/attach/202105/44250_279VRG32KPS9B5G.jpg)

主要就是 hash 比较，这个文件校验会涉及到 assets\appguard 目录下的 3 个文件：

81,936 sign.axml

256 sign.crt

510,368 sign.mf

涉及到的算法有 sha256 ,RSA.

检测到修改后，会设置检测码 9，如下：

![](https://bbs.pediy.com/upload/attach/202105/44250_FBQ6CPWA3JAD3VC.jpg)

对于这个检测，通过修改内核的目录列表返回，过滤掉了 libengine.so。

对应修改点 fs/readdir.c 中的 filldir64，相关的修改都是一些字符串比较，就是过滤，并且不同系统内核也不同，就不复制代码占篇幅了。

现在就可以继续调试了，找到 root 检测点，包括 su 文件检测和 apk 包检测：

![](https://bbs.pediy.com/upload/attach/202105/44250_GZZVKU5HZWK2JCD.jpg)

![](https://bbs.pediy.com/upload/attach/202105/44250_8QMDYUWG9VJ87KU.jpg)

![](https://bbs.pediy.com/upload/attach/202105/44250_FJYR6YSAQ2VHV8A.jpg)

相关的路径和包名：

/system/bin/su

/system/xbin/su

/sbin/su

/system/su

/system/xbin/ku.sud

/system/xbin/sutemp

/su/bin/su

/root/magisk

/sbin/adbd

daemonsu

eu.chainfire.supersu

com.topjohn

com.topjohnwu.magisk

1、  查找上面几个 su 文件路径。

2、  调用命令 which su

3、  调用 pm list packages，查找是否有

com.topjohn

com.topjohnwu.magisk

回过头来看，其实也可以用 frida hook java.lang.Runtime.exec 找到 app 调用的命令:

hookAllOverloads: exec

arguments: pm,path,com.bluepotiongames.eosm

arguments: which,su

arguments: pm,list,packages

处理方法，也是直接在内核中过滤 su 相关路径，pm list 中过滤包返回。

顺便提下，还有个读取__system_property 相关，这个有 JAVA 层，也有 native 层的，可以根据情况过滤。

对于通过 SystemProperties.get 读取 build.prop 文件信息的，这个我是直接在内核 open 函数那，重定向到 /data/local/tmp/build2.prop 了，后面直接改这个文件就好了.

在处理完文件校验，找到线程切入点后，基本后面各需求都可以顺着调试了，提下模拟器相关检测：

/system/bin/nox   

/system/bin/ttVM-prop  

/system/app/MOMOStore/MOMOStore.apk  

/system/lib/vboxsf.ko

/system/lib/vboxguest.ko  

/system/lib/vboxvideo.ko  

/system/bin/nemuVM-nemu-service

boolean isEmulator = SystemProperties.get("ro.kernel.qemu").equals("1");

附：

处理完这个 root 检测后，安装建 行的 app（ver 5.0.2）试了下，发现还是被检测（这个看网上很多也是说新版本用面具隐藏模块也不行，梆梆保护），看了下，是多了下面路径访问检测的，比如 / data 目录非 root 是访问不了的：

/system/bin

/system/xbin

/system/product/bin

/odm/bin

/vendor/bin

/data/local/tmp

/postinstall

/data

这些路径对线程名称 getprop 过滤掉就可以了。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)