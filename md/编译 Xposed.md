> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/9524651.html)

Xposed 是 Android 平台上的有名的 Hook 工具，用它可以修改函数参数，函数返回值和类字段值等等，也可以用它来进行调试。Xposed 有几个部分组成：

*   修改过的 android_art, 这个项目修改部分 art 代码，使 Hook 成为可能
*   Xposed native 部分，该部分主要提供给 XposedBridge 可调用 api 和调用修改过的 android_art 的 api, 还有生成可替换的 app_process 程序
*   XposedBridge, 该项目是主要功能是提供给 Xposed 的模块开发者 api，它将编译成 XposedBridge.jar
*   XposedInstaller, 该项目是 Xposed 安装器，使得普通用户在使用 Xposed 更方便，同时，它还可以管理手机上已经安装的 Xposed 模块，它编译完成后将生成 apk 文件，本文不讨论如何编译它。  
    了解了这些，下面我们来试试如何编译它们

准备[#](#准备)
----------

*   Ubuntu 系统，推荐 16.04 及以上，本文用的 18.04
*   Android Studio
*   Android 源码 (下载链接，请百度)
*   修改过的 android_art:[https://github.com/rovo89/android_art](https://github.com/rovo89/android_art)
*   Xposed native 部分：[https://github.com/rovo89/Xposed](https://github.com/rovo89/Xposed)
*   XposedBridge:[https://github.com/rovo89/XposedBridge](https://github.com/rovo89/XposedBridge)
*   Xposed 构建工具，XposedTools:[https://github.com/rovo89/XposedTools](https://github.com/rovo89/XposedTools)

配置[#](#配置)
----------

### Android ART[#](#android-art)

将 Android 源码下的 art 目录移动到其他路径备份，比如 Android 源码的上层路径。在 Android 源码目录执行`git clone https://github.com/rovo89/android_art -b xposed-nougat-mr2 art`, 将修改过的 android art 下载到 Android 源码根目录。

> 注：请注意上面选择的分支是`xposed-nougat-mr2`, 我使用的是 Android7.1.2 的源码，所以选择该分支。请根据 Android 源码版本选择分支。

### Xposed Native[#](#xposed-native)

转到`frameworks/base/cmds`目录, 执行`git clone https://github.com/rovo89/Xposed xposed`，将 Xposed Native 部分的源码下载。

### XposedBridge[#](#xposedbridge)

在任意目录执行`git clone https://github.com/rovo89/XposedBridge -b art`，然后导入 Android Studio 中，点 Build->Rebuild Project，会在`app/build/intermediates/transform/preDex/release`目录下生成. jar 文件，将生成的 jar 文件重命名为`XposedBridge.jar`，放入 Android 源码目录下的`out/java/`下。也可以直接生成 apk, 然后将生成的 apk 后缀改为 jar

> 注：如果想生成供 Xposed 模块调用的 XposedBridge.jar，则在 Android Studio 的右侧打开 Gradle Project，双击`jarStubs`就会在`app/build/api`生成 api.jar

### XposedTools[#](#xposedtools)

在任意目录执行`git clone https://github.com/rovo89/XposedTools`, 将 XposedTools 目录下的`build.conf.sample`复制一份，并将它重命名为`build.conf`，build.conf 文件用于配置构建环境，我们来看他的内容：

```
[General]
outdir = /android/out
javadir = /android/XposedBridge

[Build]
# Please keep the base version number and add your custom suffix
version = 65 (custom build by xyz / %s)
# makeflags = -j4

[GPG]
sign = release
user = 852109AA!

# Root directories of the AOSP source tree per SDK version
[AospDir]
19 = /android/aosp/440
21 = /android/aosp/500

# SDKs to be used for compiling BusyBox
# Needs https://github.com/rovo89/android_external_busybox
[BusyBox]
arm = 21
x86 = 21
armv5 = 17


```

*   outdir: 指定 Android 源码中的 out 目录
*   javadir: 指定 XposedBridge 目录，如果你不需要编译 XposedBridge.jar 可以不指定
*   version:Xposed 版本，这个版本号将显示在 XposedInstaller 上
*   ApospDir 下的数字: 设置 sdk 版本对应的 Android 源码
*   [BusyBox] 标签: busybox，可以不指定

配置完成后，就可以执行 build.pl 编译了, 以下有几个例子：

`./build.pl -a java`  
编译 XposedBridge.jar，需要在`build.conf`里指定`javadir`

`./build.pl -t arm:25`  
编译生成供 cpu 架构为 arm，sdk 为 25 平台使用的 Xposed

编译完成后，将在`Android源码目录/out/sdk25/arm`生成可刷入手机的 zip 文件

常见问题[#](#常见问题)
--------------

1. 执行 build.pl 的时候提示找不到函数，比如提示找不到`Config::IniFiles`.

可以通过下面的方式来寻找并安装依赖：  
（1）执行`apt-cache search Config::IniFiles`寻找 Config::IniFiles 所依赖的库文件：

> libconfig-inifiles-perl - Read .ini-style configuration files

（2）执行`sudo apt install libconfig-inifiles-perl`安装所依赖的库

[![](http://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**