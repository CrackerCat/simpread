> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266767.htm)

![](https://bbs.pediy.com/upload/attach/202103/632473_KYVKHZXRMY6ZFAK.png)

FridaManager:Frida 脚本持久化解决方案
============================

FridaManager 持久化原理简介
--------------------

Frida 强大的功能让无数安全研究人员沉溺其中。我们往往使用 pc 编写好 frida 的 js 脚本，然后使用 spawn 或者 attach 的方式使得我们的 hook 脚本生效。那么能否做到像 Xposed 一样，让我们编写的 hook 脚本不依赖于 pc，从而达到 hook 脚本的持久化呢？同时，是否能够在手机上单独设置哪一个 js 脚本只对哪一个 app 生效呢？答案是可以，得益于 fridagadget 共享库的强大功能，可以让我们达成 frida 脚本的持久化！众所周知，要想使用 fridagadget，就需要对 app 进行重打包，注入该 so 库，首先，鉴于众多 app 开发人员安全意识的增强，app 大多都会对签名进行校验，使得重打包后的 app 并不会正常的运行，往往会伴随大量的 crash。同时，众多加固厂商的技术也不容小觑。不管是从重打包繁琐的流程还是从 app 保护的角度出发，对 app 进行重打包来使用 fridagadget 都不是一个良好的解决方案。那么有没有一个一劳永逸同时足够简单的方案呢？自然是有，我们依然从 app 的生命周期出发，在我在 FART 系列脱壳文章中针对加壳 app 的运行流程做了简单的介绍，当我们点击一个 app 的图标后，最先进入的便是 ActivityThread 这个类，同时，不同于对加固 app 的脱壳，需要选择尽可能晚的时机来完成对已经解密 dex 的 dump 和修复，要想对任意一款 app 实现 Frida hook 脚本的持久化，我们的 hook 时机应当是尽可能的早，如果时机晚了，就可能会错过对 app 中自身类函数的 hook 时机，比如壳自身的类函数。因此，我们对于 fridagadget 注入的时机至少要早于 app 自身声明的 APPlication 子类的 attachBaseContext 函数的调用。因此，我们依然可以从 ActivityThread 中选择足够多的时机（只要还没有完成对 APPlication 子类的 attachBaseContext 函数的调用的时机都可以）。同时，我们又可以根据不同的 app 的包名来对当前运行的 app 进程注入不同的 js 脚本，这样，我们就不仅能够实现对 Frida 脚本的持久化，同时，还能够根据配置，针对不同的 app 进程，选择不同的 js 脚本。FridaManager 就是基于此原理实现。

FridaManager 使用流程
-----------------

### [](#1、下载fridamanager定制rom，刷入手机)1、下载 FridaManager 定制 rom，刷入手机

这里暂时只提供了 nexus 5x 的 7.1.2 , 之后会放出 Android 10 的 FridaManager 版本 rom，后期根据需求会逐步增加更多版本 rom。

### 2、在手机中安装 FridaManager app

安装后进入设置授予该 app 读写 sd 卡权限，FridaManager app 主要用于方便对不同的 app 选择需要持久化的 js 脚本，界面非常简单（请忽略 FridaManager 丑陋的界面......）  
![](https://bbs.pediy.com/upload/attach/202103/632473_9334ZCC2K99J9M7.png)

### [](#3、在手机中安装要进行持久化hook的目标app)3、在手机中安装要进行持久化 hook 的目标 app

特别注意：安装完成后不要立即打开该 app，同时注意到设置中授予该 app 读写 sd 卡权限（因为接下来要读取位于 sd 卡中的配置文件以及要持久化的 js 文件）

### [](#4、点击进入fridamanager主界面，完成对要持久化hook的app的设置)4、点击进入 FridaManager 主界面，完成对要持久化 hook 的 app 的设置

FridaManager 的主界面主要分为两块，左边是用于选择要持久化的 app；在选择好要进行注入的 app 后，接下来可以通过右边选择对其持久化的 js 脚本文件，此时可通过进入 js 脚本所在目录，点击选中。  
当选中要持久化的 app 以及其对应的 js 脚本后，会弹出对话框，这时，便配置成功 (如下图)。  
![](https://bbs.pediy.com/upload/attach/202103/632473_JXQ5QQXCJGZNY2U.png)  
接下来只需要退出 FridaManager，打开要持久化 hook 的 app 即可，这便是 FridaManager 的简单的使用流程。（同时，需要注意，当修改了需要持久化的 js 脚本文件后，需要 kill 掉 app 进程，重新打开才能生效）

体验第一个 FridaManager 持久化脚本：FridaManager 版本的 helloworld, 将下面代码保存命名为 helloworld.js，拷贝到手机 sd 卡当中。
--------------------------------------------------------------------------------------------

```
function Log(info) {
    Java.perform(function () {
        var LogClass = Java.use("android.util.Log");
        LogClass.e("FridaManager", info);
    })
}
function main() {
    Log("hello fridamanager!");
    Log("goodbye fridamanager!");
}
 
setImmediate(main);

```

代码逻辑非常清晰，只是使用 Frida 对 Log 类函数进行了主动调用，打印出 hello fridamanager! 以及 goodbye fridamanager! 这两个字符串.  
接下来，我们只需要设置该 frida 脚本生效的 app（由于需要读取 sd 卡，自然 app 需要有 sd 卡读写权限）。此时，需要使用 FridaManager 进行对特定 app 的配置，配置该 app 需要持久化的该 js 脚本，然后打开 app，便可看到 logcat 中的信息：  
![](https://bbs.pediy.com/upload/attach/202103/632473_NVWJNZECG3WVE6R.png)

编写 FridaManager 持久化脚本跟踪 jni 函数的静态绑定以及动态绑定流程
-------------------------------------------

将下面代码保存命名为 traceJNIRegisterNative.js，拷贝到手机 sd 卡当中

```
function Log(info) {
    Java.perform(function () {
        var LogClass = Java.use("android.util.Log");
        LogClass.e("FridaManager", info);
    })
}
 
function readStdString(str) {
    const isTiny = (str.readU8() & 1) === 0;
    if (isTiny)
        return str.add(1).readUtf8String();
    return str.add(2 * Process.pointerSize).readPointer().readUtf8String();
}
function hook_register_native() {
    var libartmodule = Process.getModuleByName("libart.so");
    var PrettyMethodaddr = null;
    var RegisterNativeaddr = null;
    libartmodule.enumerateExports().forEach(function (symbol) {
        //android7.1.2
        if (symbol.name == "_ZN3art12PrettyMethodEPNS_9ArtMethodEb") {
            PrettyMethodaddr = symbol.address;
        }
    });
 
    var PrettyMethodfunc = new NativeFunction(PrettyMethodaddr, ["pointer", "pointer", "pointer"], ["pointer", "int"]);
    Interceptor.attach(RegisterNativeaddr, {
        onEnter: function (args) {
            var ArtMethodptr = args[0];
            this.JniFuncaddr = args[1];
            var result = PrettyMethodfunc(ArtMethodptr, 1);
            var stdstring = Memory.alloc(3 * Process.pointerSize);
            ptr(stdstring).writePointer(result[0]);
            ptr(stdstring).add(1 * Process.pointerSize).writePointer(result[1]);
            ptr(stdstring).add(2 * Process.pointerSize).writePointer(result[2]);
            this.funcnamestring = readStdString(stdstring);
            Log("[RegisterJni begin]" + this.funcnamestring + "--addr:" + this.JniFuncaddr);
        }, onLeave: function (retval) {
            Log("[RegisterJni over]" + this.funcnamestring + "--addr:" + this.JniFuncaddr);
        }
    })
}
function main() {
    Log("hello fridamanager!");
    hook_register_native();
    Log("goodbye fridamanager!");
}
setImmediate(main);

```

接下来安装一个加壳了的 app，并到设置中授予 sd 卡读写权限，之后在 FridaManager 中完成对该 app 要持久化的 traceJNIRegisterNative.js 的配置，  
接下来只需要打开该 app，便可以观察到该 app 中每一个 jni 函数的绑定流程（包括静态注册和动态注册的 jni 函数）。  
![](https://bbs.pediy.com/upload/attach/202103/632473_TJNEKABRYT7TPC5.png)

 

好了，就到这里吧，想要体验的可以 gitub 或者到微信群中去下载，enjoy the frida world!  
后续更新会放在 github:  
[FridaManager](https://github.com/hanbinglengyue/FridaManager "FridaManager")

 

交流体验、获取更多机型支持以及反馈问题可加微信 hanbing1e, 进群深入讨论

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)

最后于 4 天前 被 hanbingle 编辑 ，原因： 更新 github