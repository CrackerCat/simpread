> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271556.htm)

> [原创] 一种基于 JDWP 动态注入代码的方案

前言
==

在逆向开发中，一般都需要对目标 App 进行代码注入。主流的代码注入工具是 [Frida](https://github.com/frida/frida)，这个工具能稳定高效实现 java 代码 hook 和 native 代码 hook，不过缺点是需要使用 Root 设备，而且用 js 开发，入门门槛较高。最近发现一种非 Root 环境下对 Debug App 进行代码注入的方案，原理是利用 Java 调试框架，通过调试器与目标虚拟机之间通讯，实现对虚拟机进程的修改。

JPDA 框架和 JDWP 协议
================

Java SE 从 1.2.2 版本以后推出了 [JPDA 框架](http://download.oracle.com/otn_hosted_doc/jdeveloper/904preview/jdk14doc/docs/guide/jpda/)（Java Platform Debugger Architecture，Java 平台调试体系结构）。JPDA 定义了一套独立且完整的调试体系，它由三个相对独立的模块组成，分别为：

*   [JVM TI](https://docs.oracle.com/javase/7/docs/platform/jvmti/jvmti.html#whatIs)：Java 虚拟机工具接口（被调试者）。被调试者运行在我们想要调试的虚拟机上，它可以通过 JVM TI 这个标准接口监控当前虚拟机的信息。
*   [JDWP](http://download.oracle.com/otn_hosted_doc/jdeveloper/904preview/jdk14doc/docs/guide/jpda/jdwp-spec.html)：Java Debug Wire Protocol，Java 调试协议（通道）。在调试者和被调试者之间，通过 JDWP 传输层传输消息。
*   [JDI](http://download.oracle.com/otn_hosted_doc/jdeveloper/904preview/jdk14doc/docs/guide/jpda/jdi/index.html)：Java Debug Interface，Java 调试接口（调试者）。调试者定义了用户可以使用的调试接口，用户可以通过这些接口对被调试虚拟机发送调试命令，同时显示调试结果。
    
    ![](https://bbs.pediy.com/upload/attach/202202/849838_DABHGR9EV6PDUTG.png)  
    Reference: [https://docs.oracle.com/javase/7/docs/technotes/guides/jpda/architecture.html](https://docs.oracle.com/javase/7/docs/technotes/guides/jpda/architecture.html)
    

其中，JDWP 协议是用于调试器与目标虚拟机之间进行调试交互的通信协议。

 

JDWP 大致分为两个阶段：握手和应答。握手是在传输层连接建立完成后，做的第一件事：  
调试器发送 14 bytes 的字符串 “JDWP-Handshake” 到目标虚拟机，虚拟机回复“JDWP-Handshake”，从而完成握手。

 

握手完成后，调试器就可以向虚拟机发送命令了。JDWP 是通过命令（command）和回复（reply）进行通信，这与 HTTP 有些相似。JDWP 本身是无状态的，因此对 command 出现的顺序并不受限制。

 

JDWP 有两种基本的包（packet）类型：命令包（command packet）和回复包（reply packet）。

 

调试器和目标虚拟机都有可能发送 command packet。调试器通过发送 command packet 获取虚拟机的信息以及控制程序的执行。虚拟机通过发送 command packet 通知调试器某些事件的发生，如到达断点或是产生异常。

 

Reply packet 是用来回复 command packet 该命令是否执行成功，如果成功 reply packet 还有可能包含 command packet 请求的数据，比如当前的线程信息或者变量的值。从虚拟机发送的事件消息是不需要回复的。

 

数据包部分 JDWP 协议按照功能大致分为 [18 组命令](http://download.oracle.com/otn_hosted_doc/jdeveloper/904preview/jdk14doc/docs/guide/jpda/jdwp-protocol.html)，包含了虚拟机、引用类型、对象、线程、方法、堆栈、事件等不同类型的操作命令。  
ART 虚拟机对 JDWP 协议的支持基本是完整的，具体信息可以参考 [ART-JDWP](https://android.googlesource.com/platform/art/+/android-cts-7.0_r9/runtime/jdwp/jdwp_handler.cc#1443) 中所支持的消息。

JDWP 协议的实现
==========

JDWP 协议内容比较多，要自行实现协议内容工作量还是比较大。庆幸的是，国外已有大神将 JDWP 协议大部分用 python 实现好了，我们只需要直接使用即可，非常方便。

 

python 实现的 jdwp 协议源码地址：[jdwp-shellifier](https://github.com/IOActive/jdwp-shellifier)

基于 JDWP 的代码注入方案
===============

Native 代码注入
-----------

既然利用 JDWP 可以让调试器跟虚拟机进行交互，我们可以通过调用基于 JDWP 协议的相关接口，向虚拟机进程中注入代码。假如只是注入 c/c++ 代码的话，实现起来很轻松，我们在 App 进程启动时加上一个断点，在断点处执行加载 so 的代码即可， 流程如下：

1.  利用 JDWP 的命令，在 App 进程启流程的某个方法中添加断点，使得 App 启动前能执行到我们注入的指令。为了使注入的代码尽早执行，这里选择`android.app.LoadedApk.makeApplication`方法处加断点；
2.  在断点处通过 jdwp 协议执行下面的 Java 代码：
    
    ```
    Runtime.getRuntime().load("data/data/package_name/libnative_injecter.so")
    
    ```
    
3.  注入的 so 加载完成后，继续 App 的启动流程;

这样 native 代码就注入到了目标 App 中。

Java 代码注入
---------

通过 jdwp 协议封装的接口可以实现 java 代码的注入，通过这种方式注入少量 Java 代码还比较轻松，大量 java 代码都用 jdwp 来实现，难度将会非常大。

 

我们可以将 java 代码编译成的是 dex 文件，然后用 c/c++ 实现 dex 文件的加载以及 dex 方法的执行，便可实现 java 代码的注入。

 

在插件化开发中，加载 dex 文件大致有两种方案，一种是多 ClassLoader 方案，一种是单 ClassLoader 方案。多 ClassLoader 方案就是根据插件 dex 路径，每个插件构造自己的`DexClassLoader`，然后用这个 classLoader 加载插件中的类。单 ClassLoader 就是将插件的 ClassLoader 里的 Element 合并到 App 的 ClassLoader 中，然后使用 App 的 ClassLoader 来加载插件里的类。

 

这里我们选择单 ClassLoader 的方案，具体步骤如下:

1.  根据插件 dex 或者 Apk 路径，反射调用`DexPathList.java`的`makePathElements`静态方法，构造出来一个用于类加载的`Element`数组；
2.  获取 App 的 Classloader，这是一个`BaseDexClassLoader`对象，反射获取成员变量`pathList`，其类型时`DexPathList`，再反射获取成员变量`dexElements`，得到一个`Element`数组；
3.  将第一步获取到的`Element`数组合并到第二步获取到的`dexElements`对象对应的`Element`数组中;
4.  最后用 App 的`classLoader`加载插件 Apk 中的类，并执行插件入口方法；

以上流程需要用 c/c++ 来实现。

Xposed 模块加载器的注入
---------------

为了在注入的代码中更方便地修改被注入 App Java 代码，我们希望注入的代码能够给 App 代码加钩子。因此，在注入代码中接入了稳定性较好的一个 Android Art Hook 库: [SandHook](https://github.com/asLody/SandHook)。

 

接入这个 Hook 库的方法有两种：

1.  方法一：在每个需要动态注入的插件工程中接入 SandHook，然后在插件工程中使用 Xposed Api 来 Hook Java 代码；
2.  方法二：将 Xposed Api 的 SandHook 所有 Java 代码 (dex 文件) 和 Native 代码 (so 文件) 注入到目标 App 中，并增加加载 Xposed 插件的相关逻辑。注入的插件工程就可以按照一个 Xposed Module 工程模式来开发，这样能显著降低插件工程的接入成本。

方案二优势更明显，因此这里采用了方案二来实现。

 

最终，在利用 JDWP 协议注入的 so 文件中，需要实现以下功能：

1.  将加载 Xposed 插件的功能编译出 dex 文件，用 dex 构造出来新的 Element 合并到 App 的 ClassLoader 中，同时 so 路径也要合并到 App 的 ClassLoader nativeBianryPath 中；
2.  调用 dex 文件中的加载指定 Xposed 模块的方法，完成外置插件的动态注入；

整体流程大致如此，不过其中还有不少细节需要处理，比如，SandHook 初始化时需要传 App Context，但是我们这个注入流程是在 LoadedApk.makeApplication 之前，此时 App 的 Context 并没有创建出来，因此，需要通过反射主动构造出一个 Context 对象：

```
LoadedApk loadedApk = ActivityThread.currentActivityThread().mBoundApplication.info;
ContextImpl appContext = ContextImpl.createAppContext(activityThread, loadedApk);

```

还有，为了能够加载 Xposed 插件中的 so 库，在加载插件 Apk 之前，需要将 Apk 中的 so 文件拷贝到 data/data 目录下，并将 so 路径传给`DexClassLoader`构造方法的最后一个参数。为了更高效地拷贝 so，这里反射调用了 Framework 里的内部类`NativeLibraryHelper`。App 安装时的 so 拷贝就是用`NativeLibraryHelper`实现的，具体拷贝操作在 native 层完成，效率更高。

 

另外，Android9 及以上的系统限制了 App 对隐藏 Api 的调用。我们可以在注入 so 的 JNI_Onload 函数中加入以下代码，便可简单绕过这种限制：

```
static void BypassHiddenApi(JNIEnv *env) {
    jclass vmRumtime_class = env->FindClass("dalvik/system/VMRuntime");
    void *getRuntime_art_method = env->GetStaticMethodID(vmRumtime_class,
                                              "getRuntime",
                                              "()Ldalvik/system/VMRuntime;");
    jobject vmRuntime_instance = env->CallStaticObjectMethod(vmRumtime_class, (jmethodID)getRuntime_art_method);
 
    jstring mystring = env->NewStringUTF("L");
    jclass cls = env->FindClass("java/lang/String");
    jobjectArray jarray = env->NewObjectArray(1, cls, nullptr);
    env->SetObjectArrayElement(jarray, 0, mystring);
 
    void *setHiddenApiExemptions_art_method = env->GetMethodID(vmRumtime_class,
                                                          "setHiddenApiExemptions",
                                                          "([Ljava/lang/String;)V");
    env->CallVoidMethod(vmRuntime_instance, (jmethodID)setHiddenApiExemptions_art_method, jarray);
}

```

使用方法
====

以上流程的完整实现已经上传到 github 上：[jdwp-xposed-injector](https://github.com/WindySha/jdwp-xposed-injector)

 

使用方法：

1.  git clone https://github.com/WindySha/jdwp-xposed-injector.git，下载工具文件；
2.  下载插件模板仓库：[XposedModuleSample](https://github.com/WindySha/XposedModuleSample), 将工程导入到 Android Studio 中，打开插件工程；
3.  在插件 AS 工程中加入自己需要注入的业务代码并编译出插件 Apk。注入的代码可以是：Hook 某些 java 方法，Hook 某些 c/c++ 函数，添加魔改 ART 虚拟机的逻辑等。
4.  连接 android 设备，在命令行中执行以下命令：

```
// 第一个参数是需要注入的Debug App的包名，第二个参数是需要注入的插件Apk路径
$ injector.sh  com.pkg.test  ../XposedModuleSample.apk
 
// 对于同一个App，第二次执行命令时，可以加上fast_mode参数，避免了重复复制dex和so文件到data/data目录下，提升启动速度
$ injector.sh  com.pkg.test  ../XposedModuleSample.apk  fast_mode

```

最终，我们在 Android 设备上启动了目标 App，并且 Xposed 插件 Apk 中的代码被注入到目标 App 中。

Debuggable App
==============

本工具唯一的要求是 App 必须是 Debuggable 的，那么如何让一个 App 变成 Debuggable 的？大致总结了以下几种方式：

1.  对于项目开发中的 App，打出 Debug 包即可；
2.  利用 [ApkTool](https://github.com/iBotPeaches/Apktool) 对 Release 包进行反编译，得到 AndroidManifest.xml 文本文件，直接修改这个文本文件，在 application 标签下添加`android:debuggable="true"`，然后将再用 Apktool 将修改后的包打包成 Apk 并签名即可；
3.  直接解压 Apk 文件，利用 [ManifestEditor](https://github.com/WindySha/ManifestEditor) 或者其他 Axml 二进制修改器直接修改解压出来的 AndroidManifest.xml 二进制文件，然后再压缩成 Apk 文件并重新签名。
4.  对于已 Root 的设备，可以通过设置系统属性 ro.debuggable 的值为 1，将设备中所有 App 都设置为 Debuggable 的，具体方法可以参考这个文章: [Android 修改 ro.debuggable 的四种方法](https://www.cxyzjd.com/article/jinmie0193/111355867)。

Some Issues
===========

1.  在已启动 Android studio 的情况下，执行注入命令会偶现 jdwp 传输数据解析失败的问题，一般重新注入一次都可以恢复正常；
2.  目前仅支持 arm,arm64 处理器，仅测试了 android 5 及以上的部分机器，未对多种机型做测试，可能存在部分机型兼容性问题。

Reference
=========

1.  [jdwp-shellifier](https://github.com/IOActive/jdwp-shellifier)
2.  [SandHook](https://github.com/asLody/SandHook)
3.  [xposed_module_loader](https://github.com/WindySha/xposed_module_loader)

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)