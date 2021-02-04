> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/3tjY_03aLeluwXZGgl3ftw)

  

以上文章由作者【r0ysue】的连载有赏投稿，共有五篇，每周更新一篇；也欢迎广大朋友继续投稿，详情可点击 [OSRC 重金征集文稿！！！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247484531&idx=1&sn=6925d63e60984c8dccd4b0162dd32970&chksm=fa7b053fcd0c8c2938d1c5e0493a20ec55c2090ae43419c7aef933bcc077692b1997f4710afa&scene=21#wechat_redirect)了解~~  

温馨提示：建议投稿的朋友尽量用 markdown 格式，特别是包含大量代码的文章

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NaWiaYdPJicA7CVTDZp5MkslJG2IFvAAcZibw4ZI1ZmibhVkcT1YcSVb8Sw/640?wx_fmt=png)

**缘 · 起**

拜读 ThomasKing 的那篇著名文章《来自高纬的对抗 - 逆向 TinyTool 自制》之后，对编译源码和修改源码来进行高维度对抗、进而实现降维打击的思路产生了浓厚的兴趣。

大佬在文中修改安卓内核源码，实现了在内核层面对`app`的内存数据进行`dump`；在内核里用`jprobe`来监控`syscall`系统调用，即使加固厂商使用静态编译的`bin`文件或通过`svc`汇编指令在用户态直接进行系统调用时，用户态`hook`已经失效并且没有意义的情况下，从内核里打出一份`log`，为分析`app`的行为提供依据。

据。

```
<6>[34728.283575] REHelper device open success!
<6>[34728.285504] Set monitor pid: 3851
<6>[34728.287851] [openat] dirfd: -100, pathname /dev/__properties__, flags: a8000, mode: 0
<6>[34728.289348] [openat] dirfd: -100, pathname /proc/stat, flags: 20000, mode: 0
<6>[34728.291325] [openat] dirfd: -100, pathname /proc/self/status, flags: 20000, mode: 0
<6>[34728.292016] [inotify_add_watch]: fd: 4, pathname: /proc/self/mem, mask: 23
<6>[34729.296569] PTRACE_PEEKDATA: [src]pid = 3851 --> [dst]pid = 3852, addr: 40000000, data: be919e38


```

本系列文章五篇的核心灵感也是来源于此，从高纬度发起降维攻击，让低纬度手段无法或难以对抗，分别立意于：

1.《来自高纬的对抗：定制`ART`解释器脱所有一二代壳》

2.《来自高纬的对抗：魔改`XPOSED`过框架检测 (上)》

3.《来自高纬的对抗：魔改`XPOSED`过框架检测 (下)》

4.《来自高纬的对抗：定制安卓内核过反调试》

5.《来自高纬的对抗：替换安卓内核解封`Linux`内核命令》

接下来我们来逐个分析详细思路和技术实现的细节。

### **`FART`简介**

如果还有人不知道`FART`是啥，在这里稍微科普下，`FART`是`ART`环境下基于主动调用的自动化脱壳方案`FART`的创新主要在两个方面：

*   之前所有的内存脱壳机都是基于`Dalvik`虚拟机做的，比如`F8LEFT`大佬的`FUPK3`，只能在安卓`4.4`之下使用，`FART`开创了`ART`虚拟机内存脱壳的 “新纪元”，上至最新的安卓`10`甚至还在`preview`的安卓`11`都可以。
    
*   在`ART`虚拟机中彻底解决函数抽取型壳的问题；构造主动调用链，主动调用类中的每一个方法，并实现对应`CodeItem`的`dump`，最终实现完整`dex`的修复和重构。
    

详细的介绍和源码下载地址当然是在寒冰大佬的`github`：https://github.com/hanbinglengyue/FART，以及他在看雪的三篇文章。

*   组件一：可以整体`dump`目前已知的所有整体型壳，也就是通杀一代壳；对于未做函数体清场的二代壳也被整体扒下来了，这部分也占很大比例；
    
*   组件二：构造主动调用链，欺骗壳去自己解密函数体，然后`dump`函数体，通杀二代壳；
    
*   组件三：将解密出来的函数体与类和函数名索引一一对应，恢复函数体可读性；
    

本文的基本流程为：

*   介绍编译`AOSP`源码的流程
    
*   在源码中添加`FART`脱壳代码继续编译
    
*   实现脱壳机镜像并刷入手机
    
*   用这个脱壳手机来脱壳并`dump`和修复函数体
    
*   实现通杀一代整体型壳和二代函数抽取型壳的目标
    

`FART`脱壳机从系统框架层源码的高维、来 “降维” 打击`App`用户层的低维技术，对于实际逆向分析工作帮助重大，可谓在`ART`虚拟机上的一项重大突破。

本文所涉及的环境、实验数据结果及代码资料等都在我的`Github`：https://github.com/r0ysue/AndroidSecurityStudy 上，欢迎大家取用和`star`，蟹蟹。

### **编译`AOSP`源码**

其实更加喜欢在`Kali Linux`上进行操作，显得更像一名 “黑客”，只是`Kali`上的`openjdk`最小只支持到`8`，而我们编译安卓`6`必须要`openjdk7`，所以只能选择`Ubuntu 1604 LTS`了。

在官网上下载

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NLicpl7YxCUtW6VWe9g9p3fcFDXribp339LhNhib5sZaEfCtNYNpUZHPSA/640?wx_fmt=jpeg)

注意下载完成后检查下`md5`是否匹配。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NUj3pWQ1pRMic8S5TwdFgEzuJ16Vt9ia8QExFRn6xDcNq0R5nqjndRs8A/640?wx_fmt=jpeg)

使用该镜像新建虚拟机时，有一点就是硬盘要给`300G`，这项操作不会真的占用`300G`的硬盘，真实占用空间会随着虚拟机内部系统占用空间大小而浮动，笔者最终编译完也只不到百`G`。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NZNrdIiaAKPiaicnpIrGxT7oPEPvdMqMWVg6hzOCRd3pvVojDMxRtowCqg/640?wx_fmt=jpeg)

其余`CPU`建议给宿主机真实核心数的一半，内存建议`10G`及以上即可。如果内存还不够，可以增加一些`swap`，详见这篇文章。

系统开机后就可以把源码包拷贝进去，源码包位于百度云盘中的`aosp_pure_souce_code`目录下，解压完成后有`md5.txt`文件的，记得也要比对`md5`噢。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NPEzJ0E1CnoUxeksrdS1jQADICPH0Fl8Kg39a0O9gnoXzOH9YRl64tw/640?wx_fmt=jpeg)

解压要用到`7z`，所以安装一波常用工具：

```
$ sudo apt install p7zip-full htop jnettop git


```

安装完成后进行解压：

```
$ 7z x aosp600r1.7z


```

新开一个终端，准备编译的环境：

```
$ apt install bison tree curl
$ dpkg --add-architecture i386
$ apt update
$ apt install libc6:i386 libncurses5:i386 libstdc++6:i386 libxml2-utils


```

安装`openjdk-7`：

```
sudo add-apt-repository ppa:openjdk-r/ppa
sudo apt-get update
sudo apt-get install openjdk-7-jdk


```

在源码目录中，下载设备驱动：

```
$ cd /home/vx-r0ysue/Desktop/aosp600r1
$ wget https://dl.google.com/dl/android/aosp/broadcom-hammerhead-mra58k-bed5b700.tgz
$ wget https://dl.google.com/dl/android/aosp/lge-hammerhead-mra58k-25d00e3d.tgz
$ wget https://dl.google.com/dl/android/aosp/qcom-hammerhead-mra58k-ff98ab07.tgz



```

解压和安装设备驱动，安装时最后要键入`I ACCEPT`来表示接受条款：

```
$ tar zxvf broadcom-hammerhead-mra58k-bed5b700.tgz
$ ./extract-broadcom-hammerhead.sh
$ tar zxvf lge-hammerhead-mra58k-25d00e3d.tgz
$ ./extract-lge-hammerhead.sh
$ tar zxvf qcom-hammerhead-mra58k-ff98ab07.tgz
$ ./extract-qcom-hammerhead.sh


```

最后就是设置编译环境变量、选择编译设备、开始编译三板斧：

```
$ source build/envsetup.sh
$ lunch    （选择17`hammerhead，也就是N5`）
$ make -j8

```

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NDSx0eT6AicH9Zdg0nC9fG5Ntibia9EWNOmUGwia7locRp4vjR855zibhW0Q/640?wx_fmt=jpeg)

然后就会开始编译，可疑使用`htop`命令来查看`CPU`、内存等信息。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NFibLLG30zgpQF4hjXhgichw25MnX0ic3g5EKWicx54ibk6p9ibibGuQaqePjw/640?wx_fmt=jpeg)

> 编译过程中出现一次`unsupported reloc 43`的错误，可以通过如下命令解决：

```
$ cp /usr/bin/ld.gold  prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.15-4.8/x86_64-linux/bin/ld


```

最终等到编译成功即可。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NOuevBcjGQqwIbRwm4QZSesMCyANicpLMH44z39cxEibkPBz4vQPLf3sA/640?wx_fmt=jpeg)

### **添加****`FART`脱壳代码**

在`FART`的`github`发布页上可以得到`FART`的源码，我们来解压并下载。

```
$ git clone https://github.com/hanbinglengyue/FART.git
$ cd FART/
$ unzip FART_6.0_sourcecode.zip
$ tree
.
├── art
│   └── runtime
│       ├── art_method.cc
│       ├── art_method.h
│       ├── interpreter
│       │   └── interpreter.cc
│       └── native
│           └── dalvik_system_DexFile.cc
├── FART_6.0_sourcecode.zip
├── fart.py
├── frameworks
│   └── base
│       └── core
│           └── java
│               └── android
│                   └── app
│                       └── ActivityThread.java
├── libcore
│   └── dalvik
│       └── src
│           └── main
│               └── java
│                   └── dalvik
│                       └── system
│                           └── DexFile.java
├── LICENSE
└── README.md

17 directories, 10 files


```

需要打开图中这六个文件，到末尾去敲两行`Enter`即可，目的是为了保证这六个文件的修改日期是新于安卓源码的，否则无法被编译系统识别为新代码，也就不会激活重新编译的流程。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NbeGvrOkIDW9gpDhz6XJoIBhy6gRpmgtk62PWg4YDJPxHDgcW9kNdKw/640?wx_fmt=jpeg)

```
$ nano art/runtime/native/dalvik_system_DexFile.cc
PageDown键翻到最下页
敲两行Enter
Ctrl X 关闭文件
y 保存


```

然后直接用文件管理器打开这个目录，将这个目录下的`art`、`framework`、`libcore`三个目录直接复制黏贴到安卓源码根目录中去，文件夹一路点击`Merge`，文件一路点击`Replace`。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NEZSsbzLvoDKGbS1UibEJyvsMIPKsqVQiaBSPRmDibeIs0G1mlic5icmaLrA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NGtzRzDbSzYKYkIuGRfKWZOI09mdYicSXV7XrwhjwjRwRjkgviaibibSEKw/640?wx_fmt=jpeg)

> 注意，我们这里可以直接复制黏贴的原因在于，使用的是跟作者一模一样的安卓源码版本，即`6.0.0_r1`；如果读者使用的不是`aosp6.0.0_r1`，切记千万不可以直接替换，会造成编译无法通过，即使编译通过，最严重情况下可能手机会变砖。

然后就可以到安卓源码目录下，`source`、`lunch`、`make`三板斧了，流程和命令与上一节相同，大概需要几十分钟至一两小时，视电脑性能而定。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9Nm7ROpR3OA2jtiaOibicbDTPpOapbo7PiaJiaYnVKWafG7V1nembc7092mtA/640?wx_fmt=jpeg)

编译完该终端不要关闭，该终端中已经将随着源码一起编译出来的`fastboot`添加到了路径中，可以直接用该终端进行刷机。

> 随着源码一起编译出来的`fastboot`是对`Nexus5`手机的兼容性最好的。如果是官网下载的最新版的`fastboot`可能无法兼容较老设备。

下载官网相同版本刷机包，解压：

```
$ wget https://dl.google.com/dl/android/aosp/hammerhead-mra58k-factory-ccbc4611.zip
$ unzip hammerhead-mra58k-factory-ccbc4611.zip


```

打开①处的压缩包，用②处新编译出来的各种`img`替换掉压缩包内的`img`镜像，直接拖过去就可以。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NOVSYWpibonmWPdwvwFvMTHdJJuJ01w8RebvEwVBVIBkDXF0mSLDMS7g/640?wx_fmt=jpeg)

最终在刚刚编译完源码的终端中，直接进行刷机。注意要确保手机连接到虚拟机，并且成功被虚拟机识别喔。

> 如果遇到虚拟机内部的`Ubuntu`不认手机的情况，即`fastboot devices`命令返回是`no permissions`，可以按照这篇 51-android 来设置一下系统的手机型号识别即可。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9N75eqpmxuiclnEsu5ibUo6zCiczJgSKrWKTQ8icQOEwBibNWsZSmAL75tDiaA/640?wx_fmt=jpeg)

刷机完成重启之后，手机再次连接到虚拟机里的`Ubuntu`系统，可以看到在`sd`卡根目录下已经有了`fart`文件夹，里面已经有一些脱下来的系统应用的`dex`，而且终端也是有`root`权限的。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NDauaesltkGQ2w12Je1XHp8QJ9QU1hTcQNqpEiaor2iccbUy8TTSmHuyA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9N6vPnoicGnVpHMUbuZt03mKicZQ6hWg5I7MaRqMEXUcGbckJsgsRhFN7Q/640?wx_fmt=png)

到这里我们的源码编译和加入脱壳机代码就结束了，接下来进入脱壳阶段。

### **`FART`完整脱壳步骤**

跟作者沟通后发现目前在开放源码的版本之后，作者又做了更新和更改，最新的版本是在`pixel`上的`8.0`，我们以最新版本为例，来演示脱壳的全过程。

首先用`jadx`打开我们的测试`app`看下，这明显是个加壳了的`app`，具体哪家有经验的读者一眼其实就会看出。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NwMJNibI7uJmVleUnicXTlXeGDr0rSxldTuXnaue0GmLX5e0Txr41R7VA/640?wx_fmt=jpeg)

*   安装`App`，给`SD`卡读写权限
    

将`App`安装到手机上：

```
# adb install aipao.apk
Performing Streamed Install
Success



```

> 建议使用较新的`Pixel`手机 (谷歌代号：`Sailfish`)，其采用的`UFS`磁盘系统速度会非常快，还在用`Nexus 5`做调试机的可以升上来了。

然后先不用打开`app`，先去应用的权限设置中，将存储卡读写权限打开，如果权限没有打开，则无法将脱下来的文件，写到`/sdcard/fart/`的目录中去。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9N3X5wHUHpHGX26gTc8LKicMONnyMzH3kreq5b6xGJCPKibEXRIfFzqWAQ/640?wx_fmt=png)

> 还有一点需要注意的就是，`FART`脱壳机对`App`进行脱壳时，是不需要连接外网的。如果担心脱壳的完整性，也可以连接`WIFI`上网，此时会发现一个系统`bug`就是系统自带的输入法会崩溃，其实安装个第三方输入法就可以了，比如搜狗输入法。

接下来就是点击`App`，启动它，系统就会自动开始脱壳。脱壳的过程中可以使用`adb logcat |grep com.aipao.hanmoveschool`来看下后台的日志，`FART`中的日志信息非常多，比如当前正在对哪个类进行加载和调用，类后面的数字代表当前类里面函数的个数，都会有日志输出。具体可以参考笔者写的另一篇源码解读的文章：《FART 源码解析及编译镜像支持到 Pixel2(xl)》。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NibZIDPWPfsZ4KFxtSrTlYjmJs8c1iaJW1bfPqHtIWMWh9AhZVmahiaXfA/640?wx_fmt=png)

> 如果日志输出中断可以多试几次，或者有时候出现很慢也可以多等一会儿。

虽然此`App`需要联网才能正常运行，但是其实在它启动的一瞬间，我们的脱壳流程已经结束了，并不会影响到具体的脱壳。在后续主动调用其每一个方法的流程中其实也并不需要联网。

> 如果其有远程动态下发的热补丁，其实`FART`也不会对这些插件`dex`进行主动调用，而只会主动调用壳修正后的`Classloader`中的`dex`。面对这种场景应该如何处理呢？1. 可以结合`frida`来遍历`Classloader`，主动送入`FART`的主动调用中；2. 也可以修改源码，在`DexClassLoader`以及`InMemoryClassLoader`中添加上主动调用的函数。当然这些得读者手动来实现了。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NibpfqUWicBWXI0joh8Zb3ZtIxb0vgPM7icq1s5iacIgfiaZRGyST4nB2K9w/640?wx_fmt=png)

这时候进入手机的`/sdcard/fart`文件夹中，就可以找到以脱壳`App`的包名来命名的文件夹，在文件夹里就放着脱下来的`dex`，和方法体`bin`文件（里面放着脱下来的函数体`CodeItem`）。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NA5khicIzcIDJj0IQiajn9YjKv2VMlVYBj2ztSxqoRxptlw3PUwKMtjxA/640?wx_fmt=png)

该文件夹中的文件种类分以下几种：

*   带`excute`关键字的`dex`文件是在`excute`脱壳点脱下来的`dex`文件，关于该脱壳点的更多信息可以看《拨云见日：安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点》，里面介绍了具体的实现思路和流程；
    
*   带`excute`关键字的`classlist_execute.txt`文件，该文件是`dex`中所有类的类列表，类名是签名形式的以`L`开头，表示它是一个类，`$`后面是类中的方法。
    

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NrS8ujxosxAjIAe2qdVSA071unficj8r8ibLB4HKMBhicqc54YSIx6xiafg/640?wx_fmt=png)

文件名前面的`1349780`是`dex`文件的大小，有同样的文件名开头说明这俩文件是一对的。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NOIcUk4cRQAAmtmz0tM9KibVyZJ8GXFqicaXicEficKWVrGeWKssgodORhQ/640?wx_fmt=png)

所以在检索类或者类中的方法时，或者定位哪个类在哪个`txt`当中时，应该检索`txt`文件，而不是`dex`文件，可以使用`grep -ril 'MainActivity' ./*.txt`这样的命令。因为`dex`是二进制文件，当`grep`检索到一些不可见字符或者特殊字符串时就会中断而退出检索，`txt`就不会有这样的问题。

*   带`dexfile`的`dex`文件就是在主动加载`dex`文件时脱下来的，关于这个脱壳点的详细介绍可以看《FART 正餐前甜点：ART 下几个通用简单高效的 dump 内存中 dex 方法》，同样，开头的`3925368`正是`dex`的文件大小。
    

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NKOstaFlFqawUkq4Cwic3XDEFrHEJLF9Laaic7SLFazg7YNJibBianjobOA/640?wx_fmt=png)

*   带`ins_`的`bin`文件，它跟带同样大小的`dexfile`的`dex`文件是一对的，`ins_`后面的`5208`代表线程的`ID`，这个其实就不用管了。
    

> `dexfile`的`dex`和`execute`的`dex`有什么区别呢？其实只是在不同的时间点脱下来的而已，`execute`要早一些，而`dexfile`的时机较晚。所以在实际操作中要么两者文件是一模一样的，要么`dexfile`的会更全一些，可能有些壳已经恢复了一些函数体。

有时候可以观察到不仅是`3027`，还会有个`4794`的线程 ID 的`bin`文件，其实是因为`FART`的脱壳线程是加在`ActivityThread`里的`performLaunchActivity`函数中的，也就是说每呈现一个`Activity`，都会新建一个线程来脱壳；也就是说，如果在进入`MainActivity`之前，如果有一个过渡动画的`Activity`的话，那么就会创建多个主动调用的线程。这里因为`4794`启动的较晚，所以也是较晚结束，这两个`ins.bin`文件其实是没有区别的，因为是对同一个`dex`进行的主动调用。如果`App`打开再关闭，再打开，那就会出现多个`ins`也很正常。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9N6hz6KlibicLgibCibBDpPt0oEwFm3sXkJNy6kHhUJh5L5DdboibyQraicibaA/640?wx_fmt=png)

接下来就是等`FART`运行结束，如果是较老的手机，比如`Nexus 5`，可能需要等好几个小时甚至一个晚上，`Pixel`一般几分钟到几十分钟就会跑完。日志中`3027`线程出现如图所示的`fart run over`，则说明`FART`跑完了，`3925368_ins_3027.bin`对应是个有效的`ins`函数体文件。

> 如果点击`App`进行登录进入另一个`Activity`的话，则肯定又会创建新的脱壳线程，以此类推，所以脱壳时尽量少操作。而且这也是不要使用`FART`手机用于日常测试的主要原因，每个`Activity`都在脱壳，其实是奇卡无比的。

对于只是采用了整体加载保护技术的一代壳来说，这时候其实完成脱壳的操作了，而且`FART`选择的脱壳时机点是极其刁钻的，`execute`函数是个内联函数，一代壳是不可能通过`hook`的技术绕过这里的，目前来看是没有对抗的可能性，当然目前也不存在只采用一代壳这一种技术来进行保护的加固厂商了，一般都会结合多种技术进行加固。

还有就是如果某些抽取型壳，在`App`运行时或执行时动态恢复函数体，`App`运行一段时间之后内存中的`dex`可能就比较完整了，并且没有再将函数体 “清场”，这时候如果`dump`的时机选择的更晚一些，也是可以将其完整的`dump`下来的。

### **`FART`完整修复步骤**

在修复文件的选择上，首先其实只需要看有`bin`函数体文件的，没有`bin`文件的`dex`是处于不同`Classloader`中的`dex`，那些是些系统库或者壳本身，不是我们关心的对象。

也就是对于我们刚刚脱壳完成的`App`来说，其他`dex`都不用看了，只需要最终`3925368_dexfile.dex`、`3925368_classlist.txt`和出现`fart run over`的线程脱壳线程函数体文件`3925368_ins_3027.bin`三个文件，因为它只有这一个`bin`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9N3vicA3vpmpdnFZNicnPS9LlE4jK1mSm8T0E0892pj9Wko8icpVCfUNofA/640?wx_fmt=png)

可以先打开`dex`来看下，比如随便看个`md5`函数，可以看到函数体都被抽空了，全部都是`nop`，这就是个典型的函数抽取的技术。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NibxfzOqMoJ7P5bfQRyyHaKTtvlj0HwZMiaqf7vmxLKLgbH7wZPgicf9mg/640?wx_fmt=png)

> 如果打开没有`bin`文件的`dex`来看的话，那就是基本上都会是系统框架层的`dex`，这些其实跟`App`没有关系。`FART`为了提升效率，并不会对系统框架库里的方法进行主动调用，所以不存在方法体`bin`文件。

接下来就是使用`fart.py`脚本来修复。

```
# python fart.py -d 3925368_dexfile.dex -i 3925368_ins_3027.bin >> repired.txt


```

> 注意此`python`的版本为`python2.7`

`fart.py`文件是个单线程文件，比较慢，需要多等待一会儿，修复出来的`repaired.txt`高达`99m`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NPN6DcJ17KXbJvVcqVzmzD7BmQX3IAVGJImkdSLeg7T4439DTELhAWg/640?wx_fmt=png)

等待修复完成来看`repired.txt`文件，还是那个`md5`的类。可以发现虽然`md5`类的构造函数只有八个字节，却也被抽取了，下面部分就是修复好的。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LCOKZn3lHIWdTu3XUibXq9NcysqVrvUKGRadIicXEancJzvmXOfxnIIwMias5yyv0kY7vccsHax5uNg/640?wx_fmt=png)

可以看最重要的`md5`函数部分，修复前和修复后的对比是非常强烈的，直接就是从无到有的体验。

```
DirectMethod:java.lang.String com.aipao.hanmoveschool.MD5::md5(java.lang.String)

before repire method+++++++++++++++++++++++++++++++++++

		registers_size      :0000000d:        13
		insns_size          :0000005a:        90
		debug_info_off      :03ffff00:  67108608
		ins_size            :00000001:         1
		outs_size           :00000002:         2
		tries_size          :00000001:         1
		insns               :00222774:   2238324
		tries               :00222828:   2238504
		handlers            :00222830:   2238512
00222774: 1200                                 |0000: const/4 v0 , 0
00222776: 1100                                 |0001: return-object v0
00222778: 0000                                 |0002: nop
0022277a: 0000                                 |0003: nop
0022277c: 0000                                 |0004: nop
0022277e: 0000                                 |0005: nop
00222780: 0000                                 |0006: nop
00222782: 0000                                 |0007: nop
00222784: 0000                                 |0008: nop
00222786: 0000                                 |0009: nop
00222788: 0000                                 |000a: nop
0022278a: 0000                                 |000b: nop
0022278c: 0000                                 |000c: nop
0022278e: 0000                                 |000d: nop
00222790: 0000                                 |000e: nop
00222792: 0000                                 |000f: nop
00222794: 0000                                 |0010: nop
00222796: 0000                                 |0011: nop
00222798: 0000                                 |0012: nop
0022279a: 0000                                 |0013: nop
0022279c: 0000                                 |0014: nop
0022279e: 0000                                 |0015: nop
002227a0: 0000                                 |0016: nop
002227a2: 0000                                 |0017: nop
002227a4: 0000                                 |0018: nop
002227a6: 0000                                 |0019: nop
002227a8: 0000                                 |001a: nop
002227aa: 0000                                 |001b: nop
002227ac: 0000                                 |001c: nop
002227ae: 0000                                 |001d: nop
002227b0: 0000                                 |001e: nop
002227b2: 0000                                 |001f: nop
002227b4: 0000                                 |0020: nop
002227b6: 0000                                 |0021: nop
002227b8: 0000                                 |0022: nop
002227ba: 0000                                 |0023: nop
002227bc: 0000                                 |0024: nop
002227be: 0000                                 |0025: nop
002227c0: 0000                                 |0026: nop
002227c2: 0000                                 |0027: nop
002227c4: 0000                                 |0028: nop
002227c6: 0000                                 |0029: nop
002227c8: 0000                                 |002a: nop
002227ca: 0000                                 |002b: nop
002227cc: 0000                                 |002c: nop
002227ce: 0000                                 |002d: nop
002227d0: 0000                                 |002e: nop
002227d2: 0000                                 |002f: nop
002227d4: 0000                                 |0030: nop
002227d6: 0000                                 |0031: nop
002227d8: 0000                                 |0032: nop
002227da: 0000                                 |0033: nop
002227dc: 0000                                 |0034: nop
002227de: 0000                                 |0035: nop
002227e0: 0000                                 |0036: nop
002227e2: 0000                                 |0037: nop
002227e4: 0000                                 |0038: nop
002227e6: 0000                                 |0039: nop
002227e8: 0000                                 |003a: nop
002227ea: 0000                                 |003b: nop
002227ec: 0000                                 |003c: nop
002227ee: 0000                                 |003d: nop
002227f0: 0000                                 |003e: nop
002227f2: 0000                                 |003f: nop
002227f4: 0000                                 |0040: nop
002227f6: 0000                                 |0041: nop
002227f8: 0000                                 |0042: nop
002227fa: 0000                                 |0043: nop
002227fc: 0000                                 |0044: nop
002227fe: 0000                                 |0045: nop
00222800: 0000                                 |0046: nop
00222802: 0000                                 |0047: nop
00222804: 0000                                 |0048: nop
00222806: 0000                                 |0049: nop
00222808: 0000                                 |004a: nop
0022280a: 0000                                 |004b: nop
0022280c: 0000                                 |004c: nop
0022280e: 0000                                 |004d: nop
00222810: 0000                                 |004e: nop
00222812: 0000                                 |004f: nop
00222814: 0000                                 |0050: nop
00222816: 0000                                 |0051: nop
00222818: 0000                                 |0052: nop
0022281a: 0000                                 |0053: nop
0022281c: 0000                                 |0054: nop
0022281e: 0000                                 |0055: nop
00222820: 0000                                 |0056: nop
00222822: 0000                                 |0057: nop
00222824: 0000                                 |0058: nop
00222826: 0000                                 |0059: nop
after repire method++++++++++++++++++++++++++++++++++++

		registers_size      :0000000d:        13
		insns_size          :0000005a:        90
		debug_info_off      :001a4314:   1721108
		ins_size            :00000001:         1
		outs_size           :00000002:         2
		tries_size          :00000001:         1
		try[0] ins start:   :00000009:         9
		try[0] ins count:   :00000022:        34
		try[0]:handler[0] exception type::LDecoder/CEStreamExhausted;
		try[0]:handler[0] ins start::0000002c:        44
00000010: 130b 1000                            |0000: const/16 v11 , 16
00000014: 23b3 4610                            |0002: new-array v3 , v11 , type@[C
00000018: 2603 4200 0000                       |0004: fill-array-data v3 , string@66
0000001e: 6e10 1268 0c00                       |0007: invoke-virtual v12 , meth@getBytes  //byte[] java.lang.String::getBytes()
00000024: 0c00                                 |000a: move-result-object v0
00000026: 1a0b b326                            |000b: const-string v11 , "MD5"
0000002a: 7110 4269 0b00                       |000d: invoke-static v11 , meth@getInstance  //java.security.MessageDigest java.security.MessageDigest::getInstance(java.lang.String)
00000030: 0c09                                 |0010: move-result-object v9
00000032: 6e20 4569 0900                       |0011: invoke-virtual v9 , v0 , meth@update  //void java.security.MessageDigest::update(byte[])
00000038: 6e10 4069 0900                       |0014: invoke-virtual v9 , meth@digest  //byte[] java.security.MessageDigest::digest()
0000003e: 0c08                                 |0017: move-result-object v8
00000040: 2185                                 |0018: array-length v5 , v8
00000042: da0b 0502                            |0019: mul-int/lit8 v11 , v2 , 5
00000046: 23ba 4610                            |001b: new-array v10 , v11 , type@[C
0000004a: 1206                                 |001d: const/4 v6 , 0
0000004c: 1204                                 |001e: const/4 v4 , 0
0000004e: 0167                                 |001f: move v7 , v6
00000050: 3454 0800                            |0020: if-lt v4 , v5 , 0028
00000054: 220b 360e                            |0022: new-instance v11 , type@Ljava/lang/String;
00000058: 7020 0668 ab00                       |0024: invoke-direct v11 , v10 , meth@<init>  //void java.lang.String::<init>(char[])
0000005e: 110b                                 |0027: return-object v11
00000060: 4801 0804                            |0028: aget-byte v1 , v4 , v8
00000064: d806 0701                            |002a: add-int/lit8 v6 , v1 , 7
00000068: e20b 0104                            |002c: ushr-int/lit8 v11 , v4 , 1
0000006c: dd0b 0b0f                            |002e: and-int/lit8 v11 , v15 , 11
00000070: 490b 030b                            |0030: aget-char v11 , v11 , v3
00000074: 500b 0a07                            |0032: aput-shar v11 , v7 , v10
00000078: d807 0601                            |0034: add-int/lit8 v7 , v1 , 6
0000007c: dd0b 010f                            |0036: and-int/lit8 v11 , v15 , 1
00000080: 490b 030b                            |0038: aget-char v11 , v11 , v3
00000084: 500b 0a06                            |003a: aput-shar v11 , v6 , v10
00000088: d804 0401                            |003c: add-int/lit8 v4 , v1 , 4
0000008c: 28e2                                 |003e: goto 0020
0000008e: 0d02                                 |003f: move-exception v2
00000090: 6e10 8367 0200                       |0040: invoke-virtual v2 , meth@printStackTrace  //void java.lang.Exception::printStackTrace()
00000096: 120b                                 |0043: const/4 v11 , 0
00000098: 28e3                                 |0044: goto 0027
0000009a: 0000                                 |0045: nop


```

阅读`Smali`源码可以发现，这是一个自己实现的`md5`，并不是标准的`md5`，如果没有这份`Smali`，就根本无法实现静态分析了。

### **`FART`脱壳中的常见问题**

Q1：`app`点击运行后，`FART`脱壳线程成功启动，但运行一段时间后`app`异常退出，是为什么？

A1：`FART`自发布到现在也已经接近一年，不排除某些壳已经针对`FART`的指纹特征进行了检测；

同时，有些壳为了对抗`FART`的脱壳原理，和对抗`DexHunter`的脱壳原理类似，在`dex`中插入一些 “垃圾类”，这些“垃圾类” 在正常运行的过程中是绝对不会用到的，但是`FART`却会全部加载并主动调用，通过判断 “垃圾类” 的加载来确定是否正在被脱壳，如果是那就退出。

这时候就用到前文提到的类列表，单独对自己需要的类，进行主动调用，这样就可以避免通过无效类来对抗`FART`的方式。

Q2：有些`App`点击运行后，没有`log`信息，不知道脱壳线程是否启动，并且何时结束。

A2：这是因为某些壳`hook`掉了`log`流程中的关键函数，阻止了`log`的打印，比如`hook` `liblog.so`中的一些函数，可以实现这个效果。

壳这样做的目的，也是为了防止`App`中的敏感信息泄露，即使再三强调下，开发人员还是将敏感信息通过`log`打印了出来，壳屏蔽掉`liblog.so`中的函数，确保`log`打印不出来。

不过此时就看不到`FART`在不在脱壳、有没有崩溃、以及什么时候结束了，这时候只有一种方法，那就是去看`bin`文件。

*   只要`FART`主动调用开始了，那么一定会有`bin`文件生成；
    
*   可以不断地`ls -alit`看下`bin`文件大小，只要是在不断变大，说明主动调用正在进行，正在不断向文件中写文件；
    
*   等到文件不再增大，说明脱壳结束。
    

### **小总结**

本文首先介绍了编译`AOSP`源码的流程，然后在源码中添加`FART`脱壳代码继续编译，实现脱壳机镜像并刷入手机，最后用这个脱壳手机来脱壳并`dump`和修复函数体，实现通杀一代整体型壳和二代函数抽取型壳的目标，从系统框架层源码的高维、来 “降维” 打击`App`用户层的低维技术，对于实际逆向分析工作帮助重大，再次感谢寒冰大佬将技术开源，为他的分享精神点赞。

网址汇总：  

- ThomasKing：https://bbs.pediy.com/user-627520.htm

- 《来自高纬的对抗 - 逆向 TinyTool 自制》：https://bbs.pediy.com/thread-215953.htm)

-https://github.com/hanbinglengyue/FART：https://github.com/hanbinglengyue/FART

- 三篇文章：https://bbs.pediy.com/user-632473.htm)。

- `Github`：https://github.com/r0ysue/AndroidSecurityStudy：https://github.com/r0ysue/AndroidSecurityStudy

- 这篇文章：https://blog.csdn.net/click_idc/article/details/80591686

- `FART` 的 `github` 发布页：https://github.com/hanbinglengyue/FART.git

- 这篇 51-android：https://github.com/snowdream/51-android

- 《FART 源码解析及编译镜像支持到 Pixel2(xl)》：https://www.anquanke.com/post/id/201896

- 《拨云见日：安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点》：https://bbs.pediy.com/thread-254555.htm

- 《FART 正餐前甜点：ART 下几个通用简单高效的 dump 内存中 dex 方法》：https://bbs.pediy.com/thread-254028.htm

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)