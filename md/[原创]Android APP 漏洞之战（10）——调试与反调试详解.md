> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272452.htm)

> [原创]Android APP 漏洞之战（10）——调试与反调试详解

Android APP 漏洞之战（10）——调试与反调试技巧详解
================================

目录

*   Android APP 漏洞之战（10）——调试与反调试技巧详解
*            [一、前言](#一、前言)
*            [二、相关介绍](#二、相关介绍)
*                    [1. 模拟器检测](#1.模拟器检测)
*                    [2.Android 动静态调试方法](#2.android动静态调试方法)
*                            [（1）静态分析](#（1）静态分析)
*                            [（2）静态分析的防护策略](#（2）静态分析的防护策略)
*                            [（3）动态分析](#（3）动态分析)
*                                    <1>AndroidStudio 动态调试
*                                    <2>IDA 动态调试步骤
*                                    <3>GDB 动态调试
*                            [（4）动态分析的防护策略](#（4）动态分析的防护策略)
*                    [3.Root 检测](#3.root检测)
*                            （1) 检测目录中是否含 su
*                            [（2）检测系统是否为测试版](#（2）检测系统是否为测试版)
*                            [（3）使用 which 命令查看是否有 su](#（3）使用which命令查看是否有su)
*                            [（4）检测 Magisk 或 Superuser.apk](#（4）检测magisk或superuser.apk)
*                            [（5）执行 Busybox](#（5）执行busybox)
*                            [（6）访问私有目录](#（6）访问私有目录)
*                            [（7）读取 build.prop 中的关键属性](#（7）读取build.prop中的关键属性)
*                            [（8）检测市面上主流的模拟器](#（8）检测市面上主流的模拟器)
*                            [（9）检测 hook 框架特征](#（9）检测hook框架特征)
*                    [4.hook 检测](#4.hook检测)
*                            [（1）Xposed 检测](#（1）xposed检测)
*                            [（2）frida 检测](#（2）frida检测)
*            [三、动静态分析实例分析](#三、动静态分析实例分析)
*                    [1. 静态分析与脱壳](#1.静态分析与脱壳)
*                            [（1）脱壳](#（1）脱壳)
*                            [（2）静态分析](#（2）静态分析)
*                    [2. 动态调试过反调试](#2.动态调试过反调试)
*                            [（1）java 层](#（1）java层)
*                            [（2）java 层过反调](#（2）java层过反调)
*                            [（3）so 层](#（3）so层)
*                                    <1>IDA 普通调试
*                            [（4）so 层过反调](#（4）so层过反调)
*                                    <1>IDA 挂起动态调试
*            [四、其他防护过反调试](#四、其他防护过反调试)
*                    [1. 过模拟器检测](#1.过模拟器检测)
*                    [2 过 root 检测](#2过root检测)
*                    [3. 过 frida 检测](#3.过frida检测)
*                    [4. 过 Xposed 检测](#4.过xposed检测)
*            [五、过反调试的 APP 漏洞挖掘](#五、过反调试的app漏洞挖掘)
*                    [1. 加密传输漏洞](#1.加密传输漏洞)
*                            [（1）hook 方法](#（1）hook方法)
*                            [（2）算法还原方法](#（2）算法还原方法)
*                    [2. 敏感信息披露漏洞](#2.敏感信息披露漏洞)
*            [六、实验总结](#六、实验总结)
*            [七、参考文献](#七、参考文献)

[](#一、前言)一、前言
-------------

撰写了好长时间，终于写完这篇帖子了，这主要是为了解决当下 APP 中存在的一些常见调试检测策略，前面系列的文章介绍了很多 Android APP 中漏洞挖掘的手段和原理，但是面对当下防护手段如此复杂的 APP，不掌握一些基本的逆向技巧，我们就更别谈进行 APP 漏洞挖掘了，本文将开始总结了当下 APP 的一些安全防护手段和技巧，帮助大家更加高效的进行漏洞挖掘。

 

本文通过收集了大量的资料，参考了看雪上众多大佬的帖子，肉丝大佬的知识星球等，本文的知识结构为：

 

本文第二节主要将检测防护手段的原理

 

本文第三节主要介绍当下 APP 中的动静态防护策略

 

本文第四节主要讲当下其他常见的反调试策略绕过方式

 

本文第五节将反调试技巧与案例结合，并列举了 APP 漏洞挖掘实例

[](#二、相关介绍)二、相关介绍
-----------------

### 1. 模拟器检测

模拟器是当时比较流行的工具，可以帮助工作人员更加便捷的进行调试工作。而随着 APP 安全防护技术的进一步发展，模拟器检测技术不断进行完善，使得很多 APP 不能在模拟器上运行，下面本文收集了当下模拟器检测技术的情况，并在后面拿案例进行讲解

 

模拟器检测可以参考:

 

看雪 sossai 大佬的文章：[Android 模拟器检测体系梳理](https://bbs.pediy.com/thread-255672.htm)

 

看雪大佬 Vancir 大佬的文章：[检测 Android 虚拟机的方法和代码实现](https://bbs.pediy.com/thread-225717.htm)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_5K9FKEHQZ3MA7Q9.png)

 

后面我们将拿一个具体案例来看看 android 模拟器检测如何具体实现

### 2.Android 动静态调试方法

#### [](#（1）静态分析)（1）静态分析

Android 上一般使用`GDA+jadx-gui+AndroidKiller`对 APP java 层进行静态分析，一般使用`ida`对 so 层进行静态分析

 

**java 层静态分析：**

 

我们拿到一个 APP，一般先使用 GDA 查看是够有加壳，如果有加壳，我们则需要对其进行脱壳

 

![](https://bbs.pediy.com/upload/attach/202204/905443_JHHGEZ5MNW4V9NZ.png)

 

针对于加壳的类型一般分为 dex 加固和 so 层加固，我们大多时候只需要解决 dex 加壳问题，就可以满足我们的一般需求了

 

dex 加壳一般分为三类：`dex整体加壳、函数抽取、dex2c/VMP`

 

dex 脱壳解决方案：

```
dex整体加固：这种方法往往通过动态加载的形式，交换Application的执行，一般我们可以通过hook方法，找到dex_file的起始地址或大小，进行脱取，也可以通过定制Room方法对关键的函数进行插桩，代表有fdex2、Frida_Dump
函数抽取：这种方法往往通过将函数代码抽取放入so文件中，执行时再从so文件读取还原，我们一般可以通过被动调用延时Dump的方法，或主动调用ArtMethod中invoke函数，触发每一个函数，然后进行回填，代表有youpk和fart
VMP：通过定制的指令集进行解释，这时往往需要手工分析，找到指令的映射表，然后进行一步步解释

```

我们脱壳后，就进入静态分析的流程

 

首先我们需要找到函数的入口点函数：![](https://bbs.pediy.com/upload/attach/202204/905443_MZZ8Z3WWUVKVYVK.png)

 

我们在 AndroidManifest 里面找打 Main 的 Activity，或通过 AndroidKiller 直接找到

 

![](https://bbs.pediy.com/upload/attach/202204/905443_ATUJZGQ7BZ2PQCT.png)

 

然后我们进入对应的入口函数，结合 Activity 的生命周期，执行流程一步步的分析源码

 

我们进行静态分析时，可以进行字符串定位来快速定位到我们需要定位的代码段

 

使用`GDA+jadx-gui`可以很好的查看静态代码段，使用`AndroidKiller`可以对代码进行修改，然后重新打包签名，当然后面需要面临的就是进一步的绕过签名机制

 

**so 层静态分析：**

 

分析 java 层代码时，会可能碰到 native 函数，这样我们的分析就自然从 java 层过渡到了 so 层

 

首先我们确定 jni 函数对应的 so 文件

 

静态注册和动态注册的区别：

```
Java_完整包名_类名_方法名：静态注册
JNI_Onload:             动态注册

```

动态注册加载 so 文件的两种形式：

```
动态注册加载so文件两种方式：
    (1)System.loadLibrary("native-lib");
    (2)System.load(so文件绝对路径)

```

所以我们可以直接搜`JNI_Onload`来判断是否为动态注册

 

动态注册：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_GVNMHWEDGDD9ZQV.png)

 

静态注册：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_BJK9Y2V394VKPQJ.png)

 

然后我们可以进一步进行分析 so 层中的代码段，下面是一些 ida 的快捷指令：

```
(1)  空格键：切换文本视图与图表视图
(2)  ESC：返回上一个操作地址
(3)  G：搜索地址和符号
(4)  N：对符号进行重命名
(5)  冒号键：常规注释
(6)  分号键：可重复注释
(7)  Alt+M：添加标签
(8)  Ctrl+M:查看标签
(9)  Ctrl+S:查看节的信息
(10)  X：查看交叉应用
(11)  F5:查看伪代码
(12)  Alt+T:搜索文本
(13)  Alt+B:搜索十六进制
(14)  代码数据切换
    C-->代码/D-->数据/A-->ascii字符串/U-->解析成未定义的内容
(15)  拷贝伪C代码到反汇编窗口:右键>copy to -assembly   
(16) IDA可以修改so的hex操作数来修改so文件，右键点击“edit”进行修改，
       然后右键点击“edit-patchrogram”提交

```

#### [](#（2）静态分析的防护策略)（2）静态分析的防护策略

```
java层：dex加壳技术、混淆技术
so层：so加壳技术、ollvm高级混淆技术

```

#### [](#（3）动态分析)（3）动态分析

Android 中动态分析，java 层我们一般使用`Android Studio`和`Jeb`两类工具进行动态调试，so 层我们会使用`IDA`和`GDB`来完成动态调试，后面我们会拿一些实质的案例来进行一步步的操作

##### <1>AndroidStudio 动态调试

首先使用 AndroidKiller 对 apk 进行反编译，查看其反编译后的工程

 

![](https://bbs.pediy.com/upload/attach/202204/905443_33KD52EFD7UKV8E.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_XK47C84RUDVQFV6.png)

 

把 project 文件导入 Android stdio

 

![](https://bbs.pediy.com/upload/attach/202204/905443_Y9WFBZQT9QPKSQU.png)

 

问题：导入报错，显示没有 setting.zip，可能是 Android stdio 版本的问题，这里使用的是 4.0 版本

 

安装插件 smalidea：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_ERRKZR7ZSGT4AR8.png)

 

配置 Android stdio

 

1）给 smail 代码一个 root 权限

 

![](https://bbs.pediy.com/upload/attach/202204/905443_QMV8ANFCFEHUTJQ.png)

 

2）配置项目的 jdk, 已经配置后可以不用

 

3）配置远程调试

 

![](https://bbs.pediy.com/upload/attach/202204/905443_W7WFH9ESUGJESRY.png)

 

添加远程调试 remote

 

![](https://bbs.pediy.com/upload/attach/202204/905443_M8UPKMU73QZ235C.png)

 

4）建立连接

```
adb shell ps  显示当前的注册信息（adb shell ps | find）| grep

```

![](https://bbs.pediy.com/upload/attach/202204/905443_98DMNWT4SDXBDUZ.png)

 

查找到当前的进程 PID

 

![](https://bbs.pediy.com/upload/attach/202204/905443_NTXP3M7QNBP9AV9.png)

 

开始转发端口

```
adb forward tcp:8700 jdwp:3924

```

或者将程序挂起：

```
adb shell am start -D -n My.XuanAo.LiuYao/.main（包名加进程）

```

点击调试:

 

![](https://bbs.pediy.com/upload/attach/202204/905443_RAUFKYEH8433AU4.png)

 

5）开始调试

 

下断点

 

![](https://bbs.pediy.com/upload/attach/202204/905443_ZVQMF4XHZ3DUBWV.png)

```
注意：Android stdio不能下断点，这是由于Android stdio 版本过高引起的

```

在虚拟机中运行 APP, 触发断点，既可以进行调试

 

![](https://bbs.pediy.com/upload/attach/202204/905443_NQ6QRWR7WPRRW6X.png)

##### <2>IDA 动态调试步骤

```
1.创建模拟器（最好使用真机）
2.在IDA里面找到android_server(dbgsrv目录)
3.把android_server文件放到手机/data/local/tmp
    adb push 文件名 /data/local/tmp
4.打开一个cmd窗口：运行android_server
    1)adb shell 连接手机
    2）给一个最高权限：su
    3）来到/data/local/tmp:cd /data/local/tmp
    4)给androi_server一个最高的权限：chmod 777 android_server
    5)查看android_server是否拥有权限：ls -l
    6)运行andorid_server: ./android_server（端口号默认是：23946）
    补充：运行andorid_server并且修改端口号：./android_server -p端口号
5.端口转发：
    adb forward tcp:端口号 tcp:端口号（之前转发的端口号是什么，这里就是什么）
6.打开DDMS：观察程序的端口号
7.挂起程序：
    adb shell am start -D -n 包名/类名
    例子：adb shell am start -D -n com.example.javandk1/.MainActivity
    补充：此时观察DDMS，被调试的程序前面有一个红色的虫子；
8.IDA里面勾选三项
    1）打开ida,选择debugger -第二项-Remote ARMlinux（第四项）
    2）添加hostname和portt：
        hostname：主机号（默认127.0.0.1）
        port：端口号（之前android_server运行时的端口号或者端口转发的端口号）
    3）出来进程列表：选择要调试的程序（可以ctrl+f，搜索包名）
    4）进来后，勾选三项：
    Suspend on process entry point程序入口点 断下
    Suspend on thread start/exit线程的退出或启动 断下
    Suspend on library load/unload库的加载和卸载 断下
补充：可以直接在这里F9（左上角有一个三角形）运行程序，然后放手；
     也可以，直接执行第九步，然后IDA运行程序
9.挂载、释放（放手）
    jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=端口号(是ddms里显示的端口8600)
    补充：此时观察DDMS，被调试的程序前面有一个绿色的虫子；

```

##### <3>GDB 动态调试

GDB 动态调试推荐使用可视化工具`HyperPwn`,GDB 的命令操作手册如下：

 

[GDB 命令操作手册](https://lldb.llvm.org/use/map.html)

 

首先，将 gdbserver 进行启动

 

![](https://bbs.pediy.com/upload/attach/202204/905443_JZWWCBYBUJABNTR.png)

 

先使用 objection 来观察要调试的 so 文件

 

![](https://bbs.pediy.com/upload/attach/202204/905443_RASRD95V3BYT544.png)

 

例如，我们要调试 libcamera_client.so

 

![](https://bbs.pediy.com/upload/attach/202204/905443_MAK62J5M5TSFPR3.png)

 

我们此时查看目标进程的状态：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_SJ2S5X74PM5YZRV.png)

 

此时我们就可以使用 gdb 去调试端口

```
./gdbserver 0.0.0.0:23946 --attach 11021

```

![](https://bbs.pediy.com/upload/attach/202204/905443_Q37U87KXKWSGQUE.png)

 

我们再打开 hyper

 

![](https://bbs.pediy.com/upload/attach/202204/905443_VNWT5TRFUUZRVJF.png)

```
gdb-multiach

```

![](https://bbs.pediy.com/upload/attach/202204/905443_YGSTKK6GQNZ5FYG.png)

 

然后我们设置

```
set arch arm
set arm fallback-mode thumb

```

![](https://bbs.pediy.com/upload/attach/202204/905443_ENJCNYTCGVUHPEA.png)

 

然后我们远程连接

```
target remote 172.31.99.61:23946

```

ip 是我们手机的 ip 地址，端口号是我们 gdbserver 转发的端口号，这里我们要注意我们的 gdbserver 不能关闭

 

![](https://bbs.pediy.com/upload/attach/202204/905443_6HZV33CGSMXZMYH.png)

 

这样就进入了我们的调试界面

```
F8下一跳
F7步入
C 进入下一个断点
ctrl+shift+pageup 显示上一行状态
ctrl+shift+pagedown 显示下一行状态
如果没有调用，我们可以使用frida主动调用

```

#### [](#（4）动态分析的防护策略)（4）动态分析的防护策略

```
java层：
    android.debuggable=false
so层：
    通过检测ptrace值的变化

```

### 3.Root 检测

root 检测逐渐成为现在的 APP 防护的一种方式，而我们要进行 hook 等更多操作，必须要获得 root 权限

 

root 检测目前一般分为下面的一些方法：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_238EAYXYKY4BBFH.png)

#### （1) 检测目录中是否含 su

```
          String[] stringArray = new String[]{"/system/app/Superuser.apk","/sbin/su","/system/bin/su","/system/xbin/su","/data/local/xbin/su","/data/local/bin/su","/system/sd/xbin/su","/system/bin/failsafe/su","/data/local/su","/su/bin/su"};
          //遍历数组路径
          //执行su
           String[] stringArray1 = new String[]{"/system/xbin/which","su"};
           process = Runtime.getRuntime().exec(stringArray1);
//遍历到说明含root，遍历不到说明为含root

```

#### [](#（2）检测系统是否为测试版)（2）检测系统是否为测试版

编译 Android 源码系统时，如果我们编译生成的是测试版，是自动拥有 root 权限的，这时我们将 su 修改名字就可以绕过 root 检测了

```
cat /system/build.prop | grep ro.build.tags

```

![](https://bbs.pediy.com/upload/attach/202204/905443_626NQGVUSDJR4FY.png)

 

这返回结果 “test-keys”，代表此系统是测试版

 

返回结果 “release-keys”，代表此系统是正式版

#### [](#（3）使用which命令查看是否有su)（3）使用 which 命令查看是否有 su

```
which su

```

![](https://bbs.pediy.com/upload/attach/202204/905443_WF7MXNWB7UFM4XC.png)

#### （4）检测 Magisk 或 Superuser.apk

```
File file = new File("/system/app/Superuser.apk");
    if (file.exists())
    {
        Log.i(LOG_TAG, "/system/app/Superuser.apk exist");
        return true;
    }
//检测Magisk 是否安装包名为 com.topjohnwu.magisk

```

#### [](#（5）执行busybox)（5）执行 Busybox

设备被 root，很有可能被安装 busybox

```
which busybox

```

![](https://bbs.pediy.com/upload/attach/202204/905443_QMVCPN3MC33JBNS.png)

 

执行 busybox:

```
String[] strCmd = new String[] {"busybox","df"};
    ArrayList execResult = executeCommand(strCmd);
    if (execResult != null){
        Log.i(LOG_TAG,"成功");
        return true;
    }
    else{
        Log.i(LOG_TAG,"失败");
        return false;
    } 
```

#### [](#（6）访问私有目录)（6）访问私有目录

Android 系统中私有目录必须要获取 root 权限才能进行访问，例如 /data、/system、/etc 等，可以通过读写测试来判断：

```
Boolean writeFlag = writeFile("/data/su_test", fileContent);
            if (writeFlag) {
                Log.i(LOG_TAG, "write ok");
            } else {
                Log.i(LOG_TAG, "write failed");
            }
String strRead = readFile("/data/su_test");
            Log.i(LOG_TAG, "strRead=" + strRead);
            if (fileContent.equals(strRead)) {
                return true;
            } else {
                return false;
            }

```

#### （7）读取 build.prop 中的关键属性

```
getprop ro.build.type
getprop ro.build.tags

```

![](https://bbs.pediy.com/upload/attach/202204/905443_TASJE26KB8QFK26.png)

 

我们可以看出这是测试版，就具有 root 权限

#### [](#（8）检测市面上主流的模拟器)（8）检测市面上主流的模拟器

一般市面上的模拟器都带有 root 权限，比如我们可以使用上文的模拟器检测来进一步检测 root，比如夜神模拟器`nox`等

#### [](#（9）检测hook框架特征)（9）检测 hook 框架特征

无论 Xposed 还是 frida 都需要 root，我们也可以通过检测 hook 框架来判断是否进行 root

### 4.hook 检测

#### [](#（1）xposed检测)（1）Xposed 检测

Xposed 是一个动态插桩的 hook 框架，通过替换 app_process 原始进程，将 java 函数注册为 native 函数，从而获得更早的运行时机，Xposed 检测详细可以参考我之前的文章 [Xposed 定制](https://bbs.pediy.com/thread-269627.htm)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_VA6EMV8AEZ2X64K.png)

 

图中的特征修改点就是我们可以进行检测的地方

#### [](#（2）frida检测)（2）frida 检测

frida 和 Xposed 原理差不多，同样是一个动态插桩工具，详细的源码分析可以参考文章：[frida 源码分析](https://mabin004.github.io/2018/07/31/Mac%E4%B8%8A%E7%BC%96%E8%AF%91Frida/)

 

frida 检测的途径很多，这里只简单介绍一些

 

**端口和 frida_server 检测：**

 

最简单的一种检测方式，厂商通过检测端口是否为固定的 27047，还可以检测运行的 frida_server 名称

 

**so 层系统 API 检测：**

```
遍历连接手机所有端口发送D-bus消息，如果返回"REJECT"这个特征则认为存在frida-server
直接调用openat的syscall的检测在text节表中搜索frida-gadget*.so / frida-agent*.so字符串，避免了hook libc来anti-anti的方法
内存中存在frida rpc字符串，认为有frida-server

```

**so 层非系统 API 检测：**

 

遍历连接手机所有端口发送 D-bus 消息，如果返回 "REJECT" 这个特征则认为存在 frida-server

```
/*
 * Mini-portscan to detect frida-server on any local port.
 */
for(i = 0 ; i <= 65535 ; i++) {
    sock = socket(AF_INET , SOCK_STREAM , 0);
    sa.sin_port = htons(i);
    if (connect(sock , (struct sockaddr*)&sa , sizeof sa) != -1) {
        __android_log_print(ANDROID_LOG_VERBOSE, APPNAME,  "FRIDA DETECTION [1]: Open Port: %d", i);
        memset(res, 0 , 7);
        // send a D-Bus AUTH message. Expected answer is “REJECT"
        send(sock, "\x00", 1, NULL);
        send(sock, "AUTH\r\n", 6, NULL);
        usleep(100);
        if (ret = recv(sock, res, 6, MSG_DONTWAIT) != -1) {
            if (strcmp(res, "REJECT") == 0) {
               /* Frida server detected. Do something… */
            }
        }
    }
    close(sock);
}

```

**检测内存库来检测：**

 

Frida 的各个模式都是用来注入的，我们可以利用的点就是 frida 运行时映射到内存的库。最直接的是挨个检查加载的库

```
char line[512];
FILE* fp;
fp = fopen("/proc/self/maps", "r");
if (fp) {
    while (fgets(line, 512, fp)) {
        if (strstr(line, "frida")) {
            /* Evil library is loaded. Do something… */
        }
    }
    fclose(fp);
    } else {
       /* Error opening /proc/self/maps. If this happens, something is off. */
    }
}

```

**inlinehook 检测 frida：**

 

frida 实现 hook 一定实现了 inlinehook 技术，所以我们还可以通过 inlinehook 库来检测

 

参考文章: [从 inlinehook 角度检测 frida](https://bbs.pediy.com/thread-269862.htm)

[](#三、动静态分析实例分析)三、动静态分析实例分析
---------------------------

### 1. 静态分析与脱壳

实验案例：client.apk，漏洞银行. apk

#### [](#（1）脱壳)（1）脱壳

我们打开 client.apk，发现使用 360 加壳

 

![](https://bbs.pediy.com/upload/attach/202204/905443_6VR9KXHKPGTJJ4X.png)

 

然后我们使用葫芦娃大佬 Frida_Dexdump 脱壳工具进行脱壳，项目代码：https://github.com/hluwa/frida-dexdump

```
(1)启动frida_server
(2)直接frida-dexdump -U -d -f 包名 -o 路径名

```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_KK7W4FAYM5TACEX.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_8YB2YSB5HHGZCPP.png)

#### [](#（2）静态分析)（2）静态分析

我们脱壳完成后就进入正常的静态分析流程

 

首先，从入口点出发：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_CFHURAT2U9EEGJX.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_WYP4U9EXPABH5BF.png)

 

我们可以看见入口类最后一个函数是启动一个延时器，3 秒后延时实例化`SplachScreen$a()`类，然后我们进入此类

 

![](https://bbs.pediy.com/upload/attach/202204/905443_UJEM4Z3X7SWGWXH.png)

 

我们可以发现该类实现 Runnable 接口，启动一个线程，然后里面完成从当前的 Activity 跳转到`MainActivity`类

 

进入`MainAcitivity`类：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_EJUZVP9TRTBB6D4.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_UT97ECDSFQZXQAH.png)

 

我们简单分析一下代码逻辑，可以发现该 APP 实现了一些常见的反调试手段：

```
（1）模拟器检测
（2）动态调试检测
（3）Root检测
（4）Frida检测

```

### 2. 动态调试过反调试

#### [](#（1）java层)（1）java 层

案例：wifiKiller.apk

 

这里我们使用 AndroidStudio 进行动态调试，发现没有 debuggable 的值，就默认 ro.debuggable = false 防止反调：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_FVATGTW3CT9QKXU.png)

 

Android studio 在动态调试的时候出现错误:

 

![](https://bbs.pediy.com/upload/attach/202204/905443_54BMC5CZGC2QAX4.png)

#### [](#（2）java层过反调)（2）java 层过反调

java 层过反调方法：

```
mprop模块
xposed模块
定制Room

```

这里我们使用第一种方法

```
./mprop ro.debuggable 1
getprop ro.debuggable
start;stop

```

![](https://bbs.pediy.com/upload/attach/202204/905443_EZBAE5847DQAYCW.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_XQ5YVSB47JVAYDW.png)

 

我们发现 ddms 可以查看进程说明已经过 java 层反调试了

#### [](#（3）so层)（3）so 层

##### <1>IDA 普通调试

样例：AliCrakme.apk

 

我们进行静态分析，分析到了 native 函数，就需要对 so 层进行分析了

 

![](https://bbs.pediy.com/upload/attach/202204/905443_CE7C7BCXVN58Z2C.png)

 

我们继续上面的案例，首先运行 android_server (这里这样是防止 android_server 反调试)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_WM3RFBBKY4AK97U.png)

 

然后进行端口转发

```
adb forward tcp:39026 tcp:39026

```

然后附加程序

 

![](https://bbs.pediy.com/upload/attach/202204/905443_APKUTSZ3RHMCNUQ.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_W8SRTQGDTJRD6T2.png)

 

在 module 中查找对应的 so 文件

 

![](https://bbs.pediy.com/upload/attach/202204/905443_MPC9FJPGGN42Y8P.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_CNZHFTPKKTRPVNU.png)

 

直接进入 securityCheck 函数

 

我们在函数头 f2 下一断点，开始动态调试

 

![](https://bbs.pediy.com/upload/attach/202204/905443_UMGVS3FVWCGDTUN.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_WJ2EDJAW8ZZVTCG.png)

 

以上是可以直接将 so 文件，附加进来，但是有时 IDA 版本或者手机系统版本原因，我们不能在 moudle 模块中找到

 

解决办法：

```
绝对地址 = 相对地址（native函数）+ 基地址（so文件）

```

首先，我们 Ctrl+s 找到 so 文件对应的基地址

 

![](https://bbs.pediy.com/upload/attach/202204/905443_SXY5XB4MGD9TFS5.png)

 

其次我们相对地址就是我们静态调试时的函数地址

 

![](https://bbs.pediy.com/upload/attach/202204/905443_UZRPSYVBSYXVBG6.png)

 

相对地址 = 11AB+B3AFA000=B3AF A1AB 我们可以发现这和我们附加进来的地址一致

 

![](https://bbs.pediy.com/upload/attach/202204/905443_5GSGKDMHHBC68UG.png)

 

这说明程序一定含有反调试的策略，我们接下来应该解决程序的反调试策略

#### [](#（4）so层过反调)（4）so 层过反调

我们可以查看 TrancePId 值来验证 (我们需要重新执行以上步骤动态调试，不过不运行)

 

首先，我们查看程序进程状态

```
ps |grep tong

```

![](https://bbs.pediy.com/upload/attach/202204/905443_TFGX4U2JXDS23RP.png)

 

其次我们，根据找到的 PID 号，查看对应状态：

```
cat /proc/25196/status

```

![](https://bbs.pediy.com/upload/attach/202204/905443_82VK7CZWNH22PRR.png)

 

你发现这里的值不为 0，你可以验证没有进行 IDA 调试时候的值，经过以上分析我们可以确定该 APK 采用了反调试策略

##### <1>IDA 挂起动态调试

首先完成基础配置：

```
a.运行android_server
b.转发端口
c.打开ddms,查看进程状态
d.挂起程序 adb shell am start -D -n 包名/.MainActivity

```

![](https://bbs.pediy.com/upload/attach/202204/905443_5QY3YYSDK33FRNJ.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_RFGYWES8C3ADKJA.png)

 

你要获得后面的程序，可以打开程序后，输入 adb shell dumpsys activity top 则可以查看

 

我们发现 ddms 上的进程前面出现红色小虫子，说明程序被挂起，而手机端也会显示挂起的界面

 

然后进行附加进程，挂载三项

 

![](https://bbs.pediy.com/upload/attach/202204/905443_VCMSNHTTCVCDP9J.png)

 

然后按 F9 运行，我们发现此时程序退出，这是因为我们此时是挂起调试，程序还没有初始化

 

我们运行:

```
jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8609

```

![](https://bbs.pediy.com/upload/attach/202204/905443_DENDJTXBZ2QJUVG.png)

 

IDA 附加 so 文件，我们点击取消就可以加载进来（要是没显示就按照上文地址的计算方法）

 

我们此时进入 libCramke.so 文件，并进入 JNI_OnLoad 方法（我们进入这里主要是为了解决反调试）

 

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

 

我们按 F9 运行，并按 F8 进行单步步入

 

下面使用 F8 开始单步调试了，发现每次到达 BLX R7 这条指令执行完之后，JNI_OnLoad 函数就退出了，这个地方存在问题，可能就是反调试的地方了。我们再次进入调试，看见 BLX 跳转的地方 R7 寄存器中是 pthread_create 函数

 

![](https://bbs.pediy.com/upload/attach/202204/905443_7SU4NWWH3CEQTFM.png)

 

到了这里我们就找到了出问题的地方，接下来我们只需要修改对应的位置就可以了。

 

可以把 BLX R7 这条指令给 nop 掉，也就是把这条指令变成空指令（相当于删除这条指令）这样 apk 就不会新建线程去执行检测代码了。

 

我们在直接静态分析 so 的 IDA 中找到 BLX R7 的位置

 

![](https://bbs.pediy.com/upload/attach/202204/905443_E56PWCJ23Q7T2QA.png)

 

我们把这条指令给 nop 掉，我们打开其对应 16 进制的窗口，并把值改为 0

 

![](https://bbs.pediy.com/upload/attach/202204/905443_UQ86EBWA9JXTBB6.png)

 

我们保存修改后的 so 文件

 

![](https://bbs.pediy.com/upload/attach/202204/905443_VX9XKV3UKHUKTR9.png)

 

我们保存，再 AndroidKiller 里面进行回编译，则得到绕过反调试的 apk

[](#四、其他防护过反调试)四、其他防护过反调试
-------------------------

我们上面静态分析了漏洞银行的案例，我们发现其采用了模拟器检测、root 检测、frida 检测，下面我们依次进行反调试绕过

### 1. 过模拟器检测

首先我们定位到该处的代码段

 

![](https://bbs.pediy.com/upload/attach/202204/905443_2CU8X82CU8FJHPX.png)

 

经过分析我们可以分析关键变量：

```
str2----->d.a----->i3

```

即决定模拟器检测的变量是`i3`

 

![](https://bbs.pediy.com/upload/attach/202204/905443_BE52XF7W7DESZU5.png)

 

我们从下至上逐一分析代码逻辑：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_KUF2WDP25NAZ962.png)

```
这里检测windows下bluestacks模拟器的文件夹是否存在，存在i3值即增加

```

![](https://bbs.pediy.com/upload/attach/202204/905443_2XC4TBV48SZD5FV.png)

```
这里使用GLES20着色器来检测Bluestacks模拟器和Translator模拟器

```

![](https://bbs.pediy.com/upload/attach/202204/905443_G8BFDZ9B63K6TCH.png)

```
上面部分代码就是从sdk等方面检测各个模拟器，我们可以看见我们这里的夜神模拟器nox

```

我们分析了模拟器检测的代码，但是这里我们发现这部分代码并没有直接导致程序崩溃的代码段

 

我们继续分析，发现

 

![](https://bbs.pediy.com/upload/attach/202204/905443_PKNAKV27WWDRWJY.png)

 

只有当我们进行 root、frida 等情况，才会将 APP 关闭，而此时我们还没有使用 frida，这就是模拟器 root 的原因导致的

 

这里首先我们关闭模拟器的 root 设置，进行重启，按照判断，应该是可以正常安装

 

![](https://bbs.pediy.com/upload/attach/202204/905443_7FAF8M4U63BPNAW.png)

 

再次安装，发现奇怪还是一样安装失败

 

![](https://bbs.pediy.com/upload/attach/202204/905443_ATNTHBJFK9UT3PG.png)

 

此时我们进一步分析，按理说就算检测到模拟器，检测 root 也应该是可以安装成功，只是打开时候结束，这里说明还有其他地方有防护

 

经过分析，我们在`AndroidManifest.xml`中找到一个关键属性`hardwareAccelerated`

 

![](https://bbs.pediy.com/upload/attach/202204/905443_6P7DGG98S74J28E.png)

 

`hardwareAccelerated=ture`意味着 APP 计划利用移动设备中 GPU 资源来使其运行，但是很显然我们的模拟器是没有 GPU 的，所以就会导致直接崩溃，安装不上去

 

这里我们简单将`hardwareAccelerated`值改为 false，然后重编译

 

![](https://bbs.pediy.com/upload/attach/202204/905443_U3YJNN8UAD3G5UF.png)

 

然后我们再次安装

 

![](https://bbs.pediy.com/upload/attach/202204/905443_XKSRYPEK7MVHBKZ.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_RRYN3U3NZ4AZ39K.png)

 

程序成功安装，并可以打开，这里我们启动 root 可以进一步验证一下刚才的分析：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_DAWX4EFUT6TQFJK.png)

 

果然程序就打开后，直接闪退，并报错检测到 root，这和我们上面的分析一致，也证明这里我们解决了模拟器检测绕过

### 2 过 root 检测

上面虽然我们关闭 root 可以解决模拟器检测绕过问题，但是很多时候没有 root，我们就很受限了，这里我们尝试进一步绕过 root 检测

 

![](https://bbs.pediy.com/upload/attach/202204/905443_6GQG22SB5CWB5Y9.png)

 

这里我们可以发现只需要将`a.R()`的返回值修改就可以了，我们可以采用 hook 手段，如 Xposed 或 frida

 

我们进一步分析，root 检测实现：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_YTWBWJ9T9F9MVQS.png)

```
我们可以发现检测root的原理很简单：
（1）列举所有su存在的集合
（2）遍历集合，执行su，看是否成功
（3）成功返回ture

```

绕过此处 root 方式很多，比如该程序没有签名校验机制，最简单直接修改代码，反编译，这里我们为了学习，通过 hook 技术，比如 Xposed 可以绕过，但是经过前面分析，该 APP 有 frida 检测，那我们就使用 frida 来进行测试

### 3. 过 frida 检测

我们随便创建一个 js 脚本，然后启动 frida

 

![](https://bbs.pediy.com/upload/attach/202204/905443_H2WUWRYH3AS4NJ6.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_M4ENSX7ZGE7QZTW.png)

 

发现程序检测到 Frida 在运行，然后直接崩溃，我们进一步分析 Frida 检测的代码

 

![](https://bbs.pediy.com/upload/attach/202204/905443_TE2QX9567YH3Z7R.png)

 

可以发现主要是`fridaCheck（）这个函数`

 

![](https://bbs.pediy.com/upload/attach/202204/905443_WN555WRP3TGGJEV.png)

 

进一步分析，可以发现该方法是 native 方法，因此我们进一步进入 so 层分析，我们打开`frida-check`so 文件

 

![](https://bbs.pediy.com/upload/attach/202204/905443_7SBMY3TM3A9EVJ4.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_DA4VSTTB9PSSYMV.png)

 

没有`JNI_Onload`程序是静态注册

 

![](https://bbs.pediy.com/upload/attach/202204/905443_44TXTRTFJFP4EFA.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_8JQMHBUSJ26V4U2.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_A4DJVGG55A6JRWA.png)

 

代码逻辑很简单：

```
（1）函数的返回值为result
（2）v2值给result
（3）connect函数是否>=0决定最后程序的返回值为true还是false
（4）addr值可以发现就是结构体的首地址，即为0xA2690002
（5）我们需要考虑二进制文件中的大小端序的问题，需要将值进一步的转变，将8位分为两半0xA269和0x0002,然后变为十进制，前面变为0x69A2，即27042
（6）所以我们可以发现connect使用该值就是其端口号27042
总结，APP检测Frida是否运行在27042的套接字上，如果允许在上面就返回true，从而可以判断程序在使用frida,然后关闭程序

```

这里我们分析结束，很简单只需要不将 frida_server 运行在 27042 端口上即可

 

![](https://bbs.pediy.com/upload/attach/202204/905443_GBJ7T2XEGHRX937.png)

 

再次启动 frida 看是否生效

 

![](https://bbs.pediy.com/upload/attach/202204/905443_EVV3BQYYXB4ESA9.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_7USNZDGQGS8EV5F.png)

 

此时我们发现程序 frida 并未检测到，这说明我们上面的分析是正确的

 

接下来只需要编写 hook 代码，将 root 检测绕过即可

```
function test() {
    Java.perform(function () {
        var class_obj = Java.use("a.a.a.a.a");
        class_obj.R.implementation = function () {
            var result = this.R();
            console.log("result:"+result);
            return false;
        }
    })
}
 
function main(params) {
    test()
}
 
setImmediate(main)

```

![](https://bbs.pediy.com/upload/attach/202204/905443_MBSD6DTW92KQZSB.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_MYGG9KKUDBQE6BR.png)

 

这里我们就发现成功的 hook 程序，进入了正常的界面

### 4. 过 Xposed 检测

过 Xposed 检测详细可以参考文章：[Xposed 定制](https://bbs.pediy.com/thread-269627.htm)

[](#五、过反调试的app漏洞挖掘)五、过反调试的 APP 漏洞挖掘
-----------------------------------

案例：漏洞银行. apk

 

前面我们采用了一些方法进行了反调试绕过，这里我们拿漏洞银行的例子，去联合我们前面的漏洞挖掘技巧和反调试技巧，展现如何过反调试后挖掘 Android 里面的一些漏洞

### 1. 加密传输漏洞

经过我们上篇帖子，讲述了如何解决抓包中的安全防护绕过问题，参考 [Android APP 漏洞之战（9）——验证码漏洞挖掘详解](https://bbs.pediy.com/thread-272270.htm), 当我们绕过这些常见的抓包防护后，最后就是 hook 和算法还原，我们前面也绕过了 hook 检测

 

我们配置好抓包环境：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_SCDTC4AGEGJYMXR.png)

 

我们使用 burpsuit 进行抓包：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_XFPHNWRU5SMQ7QX.png)

 

我们发现这里是加密的字段，我们可以全文搜索`enc_data`

 

![](https://bbs.pediy.com/upload/attach/202204/905443_TR9TXFP8ESG3B25.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_6DF8KFXQV64VK8Z.png)

 

![](https://bbs.pediy.com/upload/attach/202204/905443_2PSSMQ9DAGSM9AZ.png)

#### [](#（1）hook方法)（1）hook 方法

我们就可以定位到`e`类中的两个函数段, 这样我们只需要对这两处函数进行 hook，将参数打印出来就是我们传输的字符串了

```
function hookentry() {
    Java.perform(function () {
        var class_obj = Java.use("c.b.a.e");
        class_obj.a.implementation = function (arg0) {
            var result = this.a(arg0);
            console.log("arg0_a:"+arg0);
            console.log("result_a:"+result)
            return result;
        }
        class_obj.b.implementation = function (arg0) {
            var result = this.b(arg0);
            console.log("arg0_b:"+arg0);
            console.log("result_b:"+result)
            return result;
        }
    })
}

```

![](https://bbs.pediy.com/upload/attach/202204/905443_MWDY4CU9ZJ3BAF5.png)

 

我们进行注册：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_UHAV2DHGF5GN6KA.png)

 

先看抓包情况：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_XN9UCKSQC293FTU.png)

 

再看脚本 hook 值：

 

![](https://bbs.pediy.com/upload/attach/202204/905443_S6W97MN6FCH6EGN.png)

 

这样我们就成功的将加密前的数据和加密后的数据打印出来了

 

我们通过动态调试验证一样

 

![](https://bbs.pediy.com/upload/attach/202204/905443_889GU4K3MUGB763.png)

#### [](#（2）算法还原方法)（2）算法还原方法

上面我们利用 hook 的方法成功的获得解密后的数据，一般在做逆向开发过程时，还可以通过编写解密函数来实现解密

 

首先我们要分析加密逻辑

 

![](https://bbs.pediy.com/upload/attach/202204/905443_U7EV89RQGX49HKJ.png)

 

我们可以发现这里就是先通过 c 函数进行加密，然后使用 base64 进行加密

 

编写解密函数：

```
const SECRET = 'amazing';
const SECRET_LENGTH = SECRET.length;
 
const operate = (input) => {
  let result = "";
  for (let i in input) {
    result += String.fromCharCode(input.charCodeAt(i)^SECRET.charCodeAt(i%SECRET_LENGTH));
  }
  return result;
}
 
const decrypt = (encodedInput) => { 
  let input = Buffer.from(encodedInput, 'base64').toString();
  let dec = operate(input);
  console.log(dec);
  return dec;
}
 
decrypt("Gk8UCQwcCQAABFhTTBQHHhIJS0JFEQwSCR4BFQVPW1gaCAFDEA==")

```

![](https://bbs.pediy.com/upload/attach/202204/905443_EHMEJG8MMPT5NHD.png)

 

这样也可以成功的解密

### 2. 敏感信息披露漏洞

我们在前面的文章中也讲述了此类漏洞问题，主要是因为应用中采用了硬编码导致的

```
adb shell ps -ef | grep damn

```

![](https://bbs.pediy.com/upload/attach/202204/905443_FEA6CXJ6KTFJA2E.png)

```
adb logcat | grep 8992

```

![](https://bbs.pediy.com/upload/attach/202204/905443_Z4URQ9BHXQ2W55B.png)

 

这里我们可以发现，可以获取部分的敏感信息

[](#六、实验总结)六、实验总结
-----------------

本文总结并一一分析了当下 Android APP 中常见的防护策略，并通过案例实操实现了反调试策略的过反调，学习并掌握本节可以更加进一步助力 Android APP 漏洞挖掘中的漏洞学习，最后我们将反调试技巧和漏洞挖掘实例进行结合，并实操了案例，本文后续会将实验案例逐一上传到知识星球中。

 

github 首页：[github](https://github.com/WindXaa/Android-Vulnerability-Mining)

[](#七、参考文献)七、参考文献
-----------------

```
https://bbs.pediy.com/thread-225717.htm
https://nszdhd1.github.io/2020/03/09/Magisk%E6%A3%80%E6%B5%8B/
https://mabin004.github.io/2018/07/31/Mac%E4%B8%8A%E7%BC%96%E8%AF%91Frida/
https://www.jianshu.com/p/f679cb404524
https://bbs.pediy.com/thread-269862.htm#msg_header_h3_4
https://bbs.pediy.com/thread-270269.htm
https://juejin.cn/post/6844903733248131079#heading-5
https://bbs.pediy.com/thread-268586.htm
https://blog.51cto.com/u_15308480/3140020

```

[【公告】 [2022 大礼包]《看雪论坛精华 22 期》发布！收录近 1000 余篇精华优秀文章!](https://bbs.pediy.com/thread-271749.htm)

最后于 1 分钟前 被随风而行 aa 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm) [#漏洞相关](forum-161-1-123.htm) [#HOOK 注入](forum-161-1-125.htm) [#其他](forum-161-1-129.htm)