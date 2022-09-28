> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274572.htm#msg_header_h3_1)

> [原创]Lsposed 技术原理探讨 && 基本安装使用

一 Lsposed 技术原理探讨 && 基本安装使用
--------------------------

目前市场上主流的 Hook 框架有两款，一个是 Frida，另一个是 Xposed。他们之间各有优缺点，简单总结来说：Frida 快，但是不稳定；Xposed 稳定，但是操作繁琐，减缓了分析的操作的速度。

### 1.1 Xposed && Lsposed

#### 1.1.1 Xposed 系列工具发展历程

本章，先对 Xposed 展开讲解。那 Lsposed 是什么呢？和 xposed 有什么联系呢？既然大家能读到这本书，相信大家的技术水平已经不是停留在小白的水平了。这里只讲述核心的知识点。

 

对于 Xposed、Edposed、Lsposed，非常有必要说明下它们的发展历程，方便于读者的理解。Xposed 是早期的 Hook 框架，并且有成熟的社区以及 API 来支撑，但是它的作者在 2017 年就停止了项目的维护，如图 1-1 所示，Github 的上 Xposed 的最新版本是 v89，这个是后来增加的，最稳定的新版是 v82 版本，我们现在的框架依赖依然使用 v82 的版本。

 

![](https://bbs.pediy.com/upload/attach/202209/799845_P78UEH3VPDSNGR5.png)  
图 1-1 Xposed API

 

现对于 2017 年，已经过去 5 年的时间。技术是在不停的迭代升级的，虽然 Xposed 还能是使用，但是它本身繁琐的操作，每次编写完 Hook 代码后，需要重启手机，这样大大的减缓了分析的过程，并且浪费我们的生命。由于技术的升级，Xposed 的特征也越来越多，被反调成了常有的事情。Xposed 的作者对 Xposed 停止维护后，此框架依然起着很大的作用，后来出现了 Edposed，并接管了 Xposed 的位置。但是 Edposed 的存在期间很短，框架本身也有很多弊病。于是，对于 Edposed 的改良框架 Lspoded 脱颖而出。Edposed 我们在后面也不会提及，因为它只是一个过渡版本。

 

Lsposed 是在 Edposed 的基础上进行改良的新框架。并且接管了 Xposed 的 API，可以很好的兼容 Xposed 的 API。所以我们后面的开发工作都是基于 Xposed 的 API 进行开发，再配合上 Lsposed 的优秀特性，体验感十分良好。

 

对于 Xposed 的弊端，这里有必要说明一下：Xposed 会对所有的应用都进行注入，也就是全局模式，导致应用启动变得非常的慢，这个在 Lsposed 上有了很大的改良。在 Lsposed 上，我们可以对目标 app 选择注入，并且支持多选。这项改进也不算是重大的技术升级，说到底就是引导用户正确的使用 Xposed，确保 Xposed 框架和模块不会做额外的事情。

 

![](https://bbs.pediy.com/upload/attach/202209/799845_PGXMDUDXPBRPMHC.png)  
图 1-2 Lsposed github official declare

 

图 1-2 是 Lsposed 的官方声明，下面是对介绍的翻译：

 

Riru / Zygisk 模块试图提供一个 ART Hook 框架，该框架利用 LSPlant 挂钩框架提供与 OG Xposed 一致的 API。

 

Xposed 是一个模块框架，可以在不触及任何 APK 的情况下改变系统和应用程序的行为。这很棒，因为这意味着模块可以在不同版本甚至 ROM 上工作而无需任何更改（只要原始代码没有太大更改）。它也很容易撤消。由于所有更改都在内存中完成，您只需停用模块并重新启动即可恢复原始系统。还有许多其他优点，但这里只是一个优点：多个模块可以对系统或应用程序的同一部分进行更改。对于修改后的 APK，您必须选择一个。没有办法组合它们，除非作者用不同的组合构建了多个 APK。

 

从图 1-2 中还能看到它支持 Android 8.1-13 的系统版本。补充说明下，Xposed 旧版的 API 只支持的 Android7，后来更新的出来的一些版本，如 v89，是支持 Android 8 的版本的。

#### 1.1.2 Xposed && Lsposed 框架原理

Xposed 的 Hook 原理是从整个 Android 的启动流程入手而设计出来的框架，看懂 Android 的启动流程，也是从从按了开机键后，从硬件到软件，到底做了什么事情，我们才能更好的理解 Xposed 框架。

##### 1.1.2.1 Android 启动流程

如图 1-3 所示：是 Android 启动的整个流程图，下面我们一一对每个节点进行介绍。

 

![](https://bbs.pediy.com/upload/attach/202209/799845_RU4GJA9FCUAKVKE.png)  
图 1-3 Android 启动流程图

 

Android 整系统分为四层，分别为 kernel、Native、FrameWork、应用层（APP），loader 也可以单独算一层，是硬件的启动加载预置项。

1.  首先当我们长按开机键（电源按钮）开机，此时会引导芯片开始从固化到 ROM 中的预设代码处执行，然后加载引导程序到 RAM。然后启动加载的引导程序，引导程序主要做一些基本的检查，包括 RAM 的检查，初始化硬件的参数。
    
2.  到达内核层的流程后，这里初始化一些进程管理、内存管理、加载各种 Driver 等相关操作，如 Camera Driver、Binder Driver 等。下一步就是内核线程，如软中断线程、内核守护线程。下面一层就是 Native 层，这里额外提一点知识，层于层之间是不可以直接通信的，所以需要一种中间状态来通信。Native 层和 Kernel 层之间通信用的是 syscall，Native 层和 Java 层之间的通信是 JNI。
    
3.  在 Native 层会初始化 init 进程，也就是用户组进程的祖先进程。init 中加载配置文件 init.rc，init.rc 中孵化出 ueventd、logd、healthd、installd、lmkd 等用户守护进程。开机动画启动等操作。核心的一步是孵化出 Zygote 进程，此进程是所有 APP 的父进程，这也是 Xposed 注入的核心，同时也是 Android 的第一个 Java 进程（虚拟机进程）。
    
4.  进入框架层后，加载 zygote init 类，注册 zygote socket 套接字，通过此套接字来做进程通信，并加载虚拟机、类、系统资源等。zygote 第一个孵化的进程是 system_server 进程，负责启动和管理整个 Java Framework，包含 ActivityManager、PowerManager 等服务。
    
5.  应用层的所有 APP 都是从 zygote 孵化而来
    

##### 1.1.2.2 Xposed 注入源码剖析

上述是 Android 的大致流程，接下来我们从代码执行的角度来看执行链。

 

从图 1-3 中，我们可以总结出一条启动链：

```
init => init.rc => app_process => zygote => ...

```

一个应用的启动，核心的步骤是 Framework 层的 zygote 启动，zygote 是所有进程的父进程，也就是说，所有 APP 应用进程都是由 zygote 孵化而来。为什么要从 Native 层开始说明呢？这是因为 Native 层开始就是 Android 源码的运行过程，Xposed 的注入也就是从 Native 层开始的。

 

根据 Android 启动流程图可知，zygote 是由 app_process 初始化产生的。app_process 是一个二进制可执行文件，它的表现形式是一个 bin 文件，它位于`/system/bin/app_process`，如图 1-4 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_YUGY8VKZ8ZXKNMH.png)  
图 1-4 Android app_process

 

app_process 是由 app_main.cpp 编译而来，它的源码路径位于`/frameworks/base/cmds/app_process/app_main.cpp`，Xposed 就是将 XposedInstaller.jar 包替换原始的 app_process 来实现全局注入，所以我们每次编写完 Hook 代码后，需要重启手机才能生效。

*   init

从 Android 启动的流程图可知，首先进行的是 init 进程的启动，它在源码中对应着`/system/core/init/init.cpp`文件，如图 1-5 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_T32T6BCVUNSRA8Y.png)  
图 1-5 /system/core/init/init.cpp

 

C++ 文件的启动从 main 开始，所以我们从 main 开始追，核心代码如下所示：

```
int main(int argc, char** argv) {
    ...
 
    if (bootscript.empty()) {            // 第一次开机
        parser.ParseConfig("/init.rc");  // 解析配置文件 init.rc
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }
 
    ...
 
    while (true) { ... }
 
    return 0;
}

```

可以看到，第一次开机后，它会解析 init.rc 配置文件，此配置文件中会创建文件，并做一些用户组分配、权限赋予的操作，它位于`/system/core/rootdir/init.rc`，如图 1-6 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_2YJGSTWFJ4EMBFR.png)  
图 1-6 /system/core/rootdir/init.rc

 

此外，还会有一些触发器的操作，源码如下所示：

```
# Now we can start zygote for devices with file based encryption
trigger zygote-start

```

zygote 的生成就是由触发器生成的，在稍高一点的 Android 版本中，会分为 32 位和 64 位的 zygote，如图 1-7 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_HBF3SWCVPBCTWHB.png)  
图 1-7 不同位数的 zygote

 

这里以`init.zygote32.rc`为例做说明，其它东西都是一样的，就是架构有区别。在`init.zygote32.rc`配置文件中，有如下代码：

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks

```

我们只看第一句命令，这个是启动 zygote 进程的核心。这里加载了 app_process 文件，那么这个文件做了什么呢？app_process 对应着 Android 源码中的 app_main.cpp, 他的位置如图 1-8 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_NR5DBQJTEAF4TUV.png)  
图 1-8 app_main.cpp 位置

 

同样，我们还是看它的 main 函数，只不过这里要看他后面跟的参数是什么含义，参数匹配如图 1-9 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_3FEXPV9GMGX7F2T.png)  
图 1-9 app_main.cpp main 参数匹配

 

代码中，可以看到对 zygote 做了标记，那我们继续追踪，看看哪里引用了这个 bool 变量，全局搜索后，最后匹配到了文件的末尾，代码如图 1-10 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_UR28U6PAVZ6R8TN.png)  
图 1-10 zygote 引用

 

runtime.start 加载了`com.android.internal.os.ZygoteInit`类，继续追踪 runtime.start，最后定位到`/frameworks/base/core/jni/AndroidRuntime.cpp`，核心代码如下所示：

```
void AndroidRuntime::start(const char* className, const Vector& options, bool zygote)
{
    ...
 
    // 初始化 JNI 接口
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
 
    // 创建 env 指针
    JNIEnv* env;
    //
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
 
    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
 
    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
 
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
 
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
 
    /*
      * Start VM.  This thread becomes the main thread of the VM, and will
      * not return until the VM exits.
      */
    // 开启 VM 虚拟机，这个线程变成 VM 主线程，直到 VM 退出才结束
    // className => com.android.internal.os.ZygoteInit
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {  // 把 main 函数调起来
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
 
#if 0   // 条件编译
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);
    ...
} 
```

在后半段代码可以看出，会把`com.android.internal.os.ZygoteInit`类，通过 Native 层反射调用起来。根据类名，我们找到此文件的位置，

```
public static void main(String argv[]) {
    try {
 
        ...
 
        // socket 通信注册
        registerZygoteSocket(socketName);
 
        // 预加载所需资源到VM中，如class、resource、OpenGL、公用Library等；
        // 所有fork的子进程共享这份空间而无需重新加载，减少了应用程序的启动时间，
        // 但也增加了系统的启动时间，Android启动最耗时的部分之一。
        preload();
 
        // 初始化gc，只是通知VM进行垃圾回收，具体回收时间、怎么回收，由VM内部算法决定。
        // gc()需在fork前完成，这样将来复制的子进程才能有尽可能少的垃圾内存没释放；
        gcAndFinalize();
 
        // 启动system_server，即fork一个Zygote子进程
        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
 
            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }
 
        // 进入循环模式，获取客户端连接并处理
        runSelectLoop(abiList);
 
        // 关闭和清理zygote socket
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}

```

这里是 zygote 的初始化，首先开启了 socket 的通信。在 proload 中预加载一些资源到 VM 中，所有 fork 后的子进程都可以共享这份资源，而无须重新启动。与此同时，增加了系统启动时间，这个环节是整个 Android 启动链条上耗时最长的部分。

 

核心在于 forkSystemServer，通过此函数 fork 系统服务进程，代码如下所示：

```
private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
 
    ...
 
    int pid;
 
    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
 
        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }
 
    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
 
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }
}

```

以上就是 zygote 的启动流程追踪，相信大家到这里都明白了。那么 xposed 如何 Hook zygote，进而实现应用程序的 Hook 呢？这也是根据上述流程来的，上面说了，核心是替换 app_process,app_process.cpp 对应的文件是 app_main.cpp，我们翻看 Github 上的 Xposed 源码，如图 1-11 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_2NZF7E8HQJR5D66.png)  
图 1-11 xposed app_main.cpp

 

在图片中，我们看到有两个 app_main.cpp 出现，那它们之间有什么区别的，这就需要看编译文件了，编译配置文件就是 Android.mk，进去之后，最开始就看到它们是如何编译的，如图 1-12 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_242ZWXEZRQB7BD3.png)  
图 1-12 Android.mk

 

PLATFORM_SDK_VERSION 是 SDK 的版本，大于等于 21 以上使用 app_main2.cpp，而 SDK21 对应着 Android5，如图 1-13 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_KSJT9465EK8NZDJ.png)  
图 1-13 Android 版本

 

通过 Xposed 的编译配置文件，可以得出 Xposed 定制了 app_process 文件，我们直接看 app_main2.cpp，目前大家使用的手机型号都来到了 Android 8 及以上。在 app_main2.cpp 文件中，我们同样查看被标记的 zygote 代码，如图 1-14 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_4H2PRXHKGT2XCU9.png)  
图 1-14 xposed app_main2.cpp 核心

 

它使用 Xposed.initialize 进行初始化，我们追进去看它的实现，源码位于 xposed.cpp 中，源码如下所示：

```
/** Initialize Xposed (unless it is disabled). */
bool initialize(bool zygote, bool startSystemServer, const char* className, int argc, char* const argv[]) {
 
    // 参数接管
    xposed->zygote = zygote;
    xposed->startSystemServer = startSystemServer;
    xposed->startClassName = className;
    xposed->xposedVersionInt = xposedVersionInt;
 
    // XposedBridge.jar 加载到 ClassPath 中
    return addJarToClasspath();
}

```

初始化完成后进入魔改的 runtimeStart：

```
runtimeStart(runtime, isXposedLoaded ? XPOSED_CLASS_DOTS_ZYGOTE : "com.android.internal.os.ZygoteInit", args, zygote);

```

调用 XPOSED_CLASS_DOTS_ZYGOTE，即 xposedBridge 类的 main 方法，如图 1-15 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_YKDETK5RT6FC5Y7.png)  
图 1-15 XposedBridge

 

查看 xposedBridge 类中的 main 方法，源码如下所示：

```
protected static void main(String[] args) {
    // Initialize the Xposed framework and modules
    try {
        if (!hadInitErrors()) {
            initXResources();
 
            SELinuxHelper.initOnce();
            SELinuxHelper.initForProcess(null);
 
            runtime = getRuntime();
            XPOSED_BRIDGE_VERSION = getXposedVersion();
 
            // 初始化
            if (isZygote) {
                XposedInit.hookResources();
                XposedInit.initForZygote();
            }
 
            XposedInit.loadModules();  // 加载 Xposed 模块
        } else {
            Log.e(TAG, "Not initializing Xposed because of previous errors");
        }
    } catch (Throwable t) {
        Log.e(TAG, "Errors during Xposed initialization", t);
        disableHooks = true;
    }
 
    // Call the original startup code  => 原始执行链
    if (isZygote) {
        ZygoteInit.main(args);
    } else {
        RuntimeInit.main(args);
    }
}

```

从源码得知，它会在此加载 Xposed 的资源文件，以此完成后续的 Hook 操作。

##### 1.1.2.3 zygisk 技术原理

而新的 Lsposed 是基于 Magisk 的插件 zigisk 完成的，它也是在 Android 启动链上入手，实现 Hook。

 

zygisk 相当于 zygote 的 magisk，它会在 zygote 进程中运行 magisk 的一部分，这也使得 Magisk 模块更加强大。我们对于 Magisk 的使用需求一般是获取手机的 root 权限。使用 zygisk 可以实现动态替换 app_process。

### 1.2 Lsposed 环境搭建

测试机硬件条件

*   机器型号：Nexux 5X
*   系统：Android 8.1.0
*   镜像下载：Android 8.1.0  
    可以到`https://developers.google.com/android/images#bullhead`下载

1.  root 手机，使用 Magisk root 即可。在官网下载 Magisk-v25.1.apk，安装到手机上。

1.  对下载好的镜像进行 boot.img 提取，如图 1-16 所示：

![](https://bbs.pediy.com/upload/attach/202209/799845_4U9UG76FPENE4SA.png)  
图 1-16 boot.img 提取

1.  按照官网的方式进行 boot.img patch，如图 1-17 所示：

![](https://bbs.pediy.com/upload/attach/202209/799845_562SH93X549CJYV.png)  
图 1-17 boot.img patch

 

patch 完成后，会生成一个 patch 后的 boot.img 文件，带有 magisk 的标识，如图 1-18 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_T4HP2DVZYKRC2YW.png)  
图 1-18 patch 后的 boot.img

1.  将 patch 后的 boot.img 在 bootloader 模式使用 fastboot flash boot boot.img 命令刷入手机，至此 Magisk 刷入完成。
    
2.  在 Magisk app 设置中打开 zigisk
    
3.  新版的 Magisk 仓库没有插件，需要手动进行下载，我们到 Github 上去下载，如图 1-19 所示：
    

![](https://bbs.pediy.com/upload/attach/202209/799845_AP4KQH4T78TTWRQ.png)  
图 1-19 Lsposed 安装包

1.  本地安装，点击模块后，有一个本地安装的按钮，如图 1-20 所示：

![](https://bbs.pediy.com/upload/attach/202209/799845_EHHY5YUJ5HSENAG.png)  
图 1-20 Lsposed 安装

 

本地安装完成后就可以使用 Lsposed 了

#### 1.2.1 Lsposed Hook 环境搭建

Lsposed 的开发环境同 Xposed 的一致。

1.  app 级 build.gradle 加入图 1-21 所示的配置

![](https://bbs.pediy.com/upload/attach/202209/799845_6P44WX9VU6WNE7Z.png)  
图 1-21 build.gradle

1.  AndroidManifest.xml 配置，如图 1-22 所示

![](https://bbs.pediy.com/upload/attach/202209/799845_UFKM5QBRZAN2M69.png)  
图 1-22 AndroidManifest.xml

1.  入口文件建立

在 main 下新建 asserts 资源目录，如图 1-23 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_7V3HKS9UEH38RDQ.png)  
图 1-23 asserts

 

并在下一级建立 xposed_init 文件，如图 1-24 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_GCG5YDF52GMXK6C.png)  
图 1-24 xposed_init 文件

1.  添加入口类，如图 1-25 所示

![](https://bbs.pediy.com/upload/attach/202209/799845_XMGW5S7Q7RKT4FM.png)  
图 1-25 入口类

 

HookTest 是新建的一个类，用来测试我们的 demo

1.  编写测试代码

![](https://bbs.pediy.com/upload/attach/202209/799845_RWS59SEKUPBTUCV.png)  
图 1-26 测试代码

 

测试代码的功能是打印目标 app 的包名

#### 1.2.2 基本使用

1.  激活模块

运行 Hook 代码后，打开 Lsposed，模块第一次启动还未激活，如图 1-27 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_R78SDBGK8D7XZU9.png)  
图 1-27 未激活的模块

1.  选择应用

激活模块后，我们可以选择待 Hook 的应用，应用是可以多选的，如图 1-28 所示：

 

![](https://bbs.pediy.com/upload/attach/202209/799845_2PYU8HU5CEVFXXX.png)  
图 1-28 应用选择

1.  重启应用，Hook 代码即可生效，如图 1-29 所示：

![](https://bbs.pediy.com/upload/attach/202209/799845_WDBD6FVJDCZN875.png)  
图 1-29 Hook 生效

[[2022 冬季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

最后于 16 分钟前 被 r0ysue 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#源码框架](forum-161-1-127.htm)