> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-257766.htm)

**android 安全技术点小结**
===================

移动端安全的东西很多，花样也很多，不论是从硬件架构，操作系统，还是其他安全角度来讲，每接触一项新事物，都有可能需要学习一整个生态内的安全知识。  
最近开学杂事逐渐增多。。。就把我自己关于 android 的一些学习笔记和内容整理了下发出来，以铺大饼的形式尽量囊括在入门 Android 安全所学习过的一些知识点，希望对大家有所帮助。

 

ps：以下内容没有具体例子的分析，原因是其中的大部分知识点，我都是从其他师傅们的文章和代码一路学习过来的。文章也整理了以下放有我自己学习下来感觉收获较大的资源链接。

 

总结下来，对于一个平台的逆向工程技术需要掌握的技能大致如下：

*   操作系统的安全架构
*   操作系统中可执行文件格式
*   各类工具的使用
*   反汇编代码的阅读理解
*   调试器的使用（单独从工具中列出来以示重要性）
*   网络抓包工具的使用

本文大致分为以下几个模块：

*   android 端的基本安全知识
*   android 端的调试逆向
*   android 端的漏洞挖掘

**android 端的基本安全知识**
--------------------

1.  编译与反编译
2.  虚拟机，汇编，文件结构
3.  Android 系统和程序的启动过程

其实总的来说就是 ---> 逆向工程 orz

### **编译与反编译**

apktool：可以将 apk 文件反编译生成 smali 格式的反汇编代码，也可将 apktool 重新编译生成 apk 文件。这里提醒一点就是具体厂商 apktool 使用会涉及到具体的资源包。

 

学习途径：[官方文档](https://ibotpeaches.github.io/Apktool/documentation/)

 

adb：android sdk 自带的命令行工具，用于与设备通信，便于执行各种设备操作

 

学习途径：官方文档配上 awesome 系列基本上就够用了

1.  [官方文档](https://developer.android.google.cn/studio/command-line/adb?hl=zh_cn2)
    
2.  [awesome-adb](https://github.com/mzlogin/awesome-adb)
    

放几条常用指令：

 

通过此指令可以获取所有包名

*   adb shell pm list packages -f

启动 app

*   adb shell am start -n [PACKAGE-NAME]/[ACTIVITY-NAME]

列出所有正在执行 app 的 activity

*   adb shell dumpsys activity

停止应用程序

*   adb shell am force-stop [PACKAGE-NAME] 或者 adb shell am kill-all [PACKAGE-NAME]

signapk，keytool 和 jarsigner： 用于给 apk 签名的工具（在这踩过坑，建议搞清楚），这里放一条我常用的解决办法，两条命令，前者是生成自己的密钥，后者是签名，具体选项代表自查

```
- keytool -genkey -keystore test.keystore  -alias test -keyalg RSA -validity 10000
- jarsigner -verbose -keystore test.keystore -signedjar signed.apk unsigned.apk test

```

jd-gui 和 dex2jar: 反编译工具，通常用于阅读反编译生成的 java 代码

 

当然这些都是些传统的工具，推荐在熟悉这些工具的具体运作方式之后再选择高效的集成工具，比如说 jadx，jeb 等。

 

apk 的打包流程：

*   用 aapt 打包资源文件生成 R.java 文件
*   处理 aidl 文件，生成相应 java 文件
*   **编译**工程源代码，生成相应 class 文件（重要的一步）
*   转换所有 class 文件，并生成 classes.dex 文件（主要是将 java 字节码转换为 Dalvik 字节码）
*   打包生成 APK 文件
*   对 APK 文件**签名**
*   对签名后的 APK 文件进行对齐处理

APK 的文件结构：  
![](https://bbs.pediy.com/upload/attach/202002/837839_KCNJSNVN7H7PD9G.png)

 

整体流程如下：  
![](https://bbs.pediy.com/upload/attach/202002/837839_V2FVPV53FDKYPX9.png)

 

app 的安装途径：系统程序安装，android 市场安装，adb 工具安装，sd 卡安装  
app 安装过程可以追踪分析 android 系统程序 PackageInstaller 中 PackageInstallerActivity 来去理解，具体内容这里不再展开。

### **虚拟机**，汇编，文件结构

**Dalvik 虚拟机**：其设计的初衷也许是是提高运行效率并规避与 oracle 的版权纠纷  
其特点有：

1.  体积小，占用内存空间小
2.  dex 可执行文件格式
3.  常量池采用 32 为索引值
4.  基于寄存器架构，有一套完整的指令集（同时有些寄存器没有用到）
5.  提供了对象生命周期管理，堆栈管理，线程管理，安全和异常管理，垃圾回收等功能
6.  android 程序都运行在 android 系统进程里，每个进程对应一个 Dalvik 虚拟机实例

Dalivik 文件结构与 java 文件结构不同，Dalvik 虚拟机通过 dx 工具对 java 类文件中常量池进行分解，消除冗余信息后在组合成新常量池

 

![](https://bbs.pediy.com/upload/attach/202002/837839_YVZKWCF7TGJABXR.png)

 

又由于 Dalvik 虚拟机是基于寄存器架构的，相比于基于栈架构的 java 虚拟机，数据访问会快很多，同时 Dalvik 的指令即更加精简，程序的执行速度会快些。  
android 系统架构图：

 

![](https://bbs.pediy.com/upload/attach/202002/837839_PNK86A8KKXXBTFX.png)

 

可见 Dalvik 属于 Android 运行时环境，和核心库共同承担 Android 应用程序的运行工作  
这里有一点需要注意就是以消息通信的角度来看又可以分成另一个架构图，其中我们常关注的是 native 层和 java 层

 

![](https://bbs.pediy.com/upload/attach/202002/837839_PWZKVQXVE3TA392.png)

 

初次之外还有 **art**，**jvm**，可以看这篇博客

*   [art,jvm,Dalvik](https://blog.csdn.net/evan_man/article/details/52414390)

**Android 系统和程序的启动过程：**

 

Android 系统启动加载内核后 ----> 执行 init 进程 ----> 启动 Zygote 进程 ----> 初始化 Dalvik 虚拟机 ---> 启动 system_server 进入 Zygote 模式，用 socket 等待命令 ---> Zygote 收到命令后 fork 一个 Dalvik 虚拟机实例来执行程序入口函数  
流程大致如下图：

 

![](https://bbs.pediy.com/upload/attach/202002/837839_KWRJ9W553KGBBQV.png)

 

其中 Zygote 有三种创建进程的方法:

*   fork() 创建 Zygote 进程
*   forkAndSpecialize() 创建非 Zygote 进程
*   forkSystemServer() 创建系统服务进程

fork 后 ---> 虚拟机通过 loadClassFromDex 完成装载 (用 gDvm.loadedClass 全局哈希表存储查询类) ---> dvmVerifyCodeFlow 对代码检验 ---> FindClass 查找装载 main 方法类 ---> dvmInterpret 初始化解释器并执行字节码流

 

以上就是 Dalvik 在程序执行时的要点，还有其涉及到的 JIT 技术和 Dalvik 的汇编代码内容繁杂，建议直接阅读官方文档或者相应书籍  
其中 dex 文件主流反汇编工具有 BakSmali 与 Dedexer，详细的 dex 文件格式内容也建议直接阅读相关资料或源码，这里就出 DexFile 的数据结构。

```
Struct DexFile{
    DexHeader Header;
    DexStringId StringIds[stringIdsSize];
    DexTypeId TypeIds[typeIdsSize];
    DexProtoId ProtoIds[protoIdsSize];
    DexFieldID FieldIds[fileIdsSize];
    DexMethodId ClassDefs[classDefsSize];
    DexData Data[];
    DexLink LinkData;
}

```

结构图  
![](https://bbs.pediy.com/upload/attach/202002/837839_FHY43AX9VEE94CY.png)

### **模拟器体系结构**

[关于模拟器体系结构的梳理](https://bbs.pediy.com/thread-255672.htm)

 

这个链接放在这里的原因是作者自己曾经根深蒂固地把 arm 和 android 两个概念紧紧的结合在一起了，忽略了 android studio 创建的模拟器是 Intel x86 架构的，导致踩过坑。

**android 端的调试**
----------------

这里说是调试而不说逆向的原因是因为逆向的内容实在是太多了，入门可以参考《android 软件安全与逆向分析》，《android 攻防权威指南》以及《漏洞战争》中的部分内容来学习，这里选择调试中重要的内容来介绍。姑且就分为以下两者吧

*   手动调试
*   半自动化工具调试

### **smali 的调试**

在 smali 语法中，使用的都是寄存器，但是其在解释执行的时候，很多都会映射到栈中。通常每个 smali 会对应一个类。

 

**编译 - smali2dex**

 

给定一个 smali 文件，我们可以使用如下方式将 smali 文件编译为 dex 文件。

*   java -jar smali.jar assemble src.smali -o src.dex

**运行 smali**

 

在将 smali 文件编译成 dex 文件后，我们可以进一步执行  
首先，使用 adb 将 dex 文件 push 到手机上

*   adb push main.dex /sdcard/

其次使用如下命令执行

*   adb shell dalvikvm -cp /sdcard/main.dex main

**AS + smalidea**

1.  打开设备中需要调试的 apk，在 cmd 中运行命令 ：adb shell "dumpsys activity top | grep --color=always ACTIVITY"，就可以看到需要调试 apk 的包名、主 acitivity 以及进程号（或者也可以使用 adb shell "dumpsys activity activities | grep xxxActivity"）
2.  使用命令以 debug 模式启动 apk:adb shell am start -D -n 包名 / 主 activity 名，然后你的设备可以看到 wait for debugger
3.  再一次运行第一步的命令，获取新的进程号：adb shell "dumpsys activity top | grep --color=always ACTIVITY" 或者运行命令： adb shell "ps | grep 包名" 亦可
4.  adb forward tcp:debug 端口 jdwp:apk 进程号
5.  以上步骤可以直接在 as 中 logcat 里选定调试进程，然后选择 Attach Debugger To Android Process
6.  查看与包名相关 JDWP 命令：adb shell "ps -t | grep -A 8 包名"
7.  查看所有 JDWP 进程命令：adb shell "ps -t | grep -B 6 JDWP"

**smali 的修改**

 

之前尝试用 apk 改之理，但是由于版本太老了有较多不方便，所以弃坑，现在我通常是 apktool 反编译后修改，然后再编译并使用自签名，这样也可以达到修改安装使用的效果

 

**基本原生程序**

 

如 elf 文件就可以

*   使用 android_server 的 PIE 版本
*   利用 010Editor 将可执行 ELF 文件的 header 中的 elf header 字段中的 e_type 改为 ET_DYN(3)。

**so 原生程序的调试**

 

其加载方式有 system.load 和 system.loadlibrary 两种，前者加载绝对路径，后者加载 libs 下的 so 文件，两者都会在内部调用 doload 函数，其流程大致如下

 

doload -> nativeload -> 对应到 Dalvik_java_lang_Runtime_nativeLoad-> dvmLoadNativeCode 加载相应的 native code -> findSharedLibEntry(判断是否已经加载了这个库以及是否是对应的 class loader) -- 如没有加载 --> dlopen 打开 --> si->CallConstructors() 初始化 --> 创建表且用 dlsym 获取对应 so 文件中 JNI_OnLoad 函数

 

静态分析 java 层： 没什么好说的，理解程序逻辑去做就好了

 

静态分析原生层程序基本的过程如下

*   提取 so 文件
*   ida 反编译 so 文件阅读 so 代码
*   根据 java 层的代码来分析 so 代码。
*   根据 so 代码的逻辑辅助整个程序的分析。

在 android studio 3.12 后已经将 ddms 移除了，所以官方的建议是

 

![](https://bbs.pediy.com/upload/attach/202002/837839_W7QAEZJQM7EJ3DP.png)

 

当我们不能使用 ddms 时意味着我们使用 jdb 进行转发时无法确定具体的 port，有一种方法是加载程序运行时需要的 so 文件，然后在一些关键函数比如 jniString() 函数下断，运行 apk 后，然后 attach 其进程即可。

 

当然有时候会需要在 jni_load 下断，而 so 文件又被处理或者干脆 jni_load 被加密了，这时需要 pull 出来 libdvm so 文件, 参考我们加载方式的流程，直接查找 dvmLoadNativeCode，函数中调用 dlopen 加载 so，返回时 so 已经加载且已经初始化完成，调试下就能找到。

 

**hook**

 

hook 的方法很多，但原理就是这么几种，同时又有对 Dalvik,ART,inline,GOT 等对象 hook，这个有太多大佬写过各种类型的 hook 了，hook 的框架也有许多，展开分析内容太多了，就不再累述了，这里就推荐下以 frida 入门。

 

Frida hook 分为注入进程，直接在源码中修改，以及动态链接三种方式

 

frida 的交互实现大致可以这么理解：默认监听 27042 端口，对目标进程 gadget 在启动时进行阻塞，直到实现 attach 进程或者在使用 spawn() 后再 resume 恢复进程，当然这些状态都可以通过配置修改，也可以提前设置好过滤器将特定脚本加载到特定应用中。

 

由于 frida 是个轻量级的 hook 框架，所以还是比较容易添加自己想要的功能，具体请看官网的 frida 架构图。

 

frida 有一个功能可以为我们生成一个进程而不是将它注入到运行中的进程中，它注入到 Zygote 中，生成我们的进程并且等待输入。即 spawn 是注入 zygote 而 attach 是注入当前进程。

 

这里有遇到过的一些小坑，分别是设备检测问题和 windows 端的编码问题：

 

https://github.com/frida/frida/issues/1111

 

https://github.com/rkern/line_profiler/issues/37

**android 漏洞挖掘**
----------------

一直想挖 android 上的漏洞，但真正上手的时候发现自己缺乏很多真实场景的经验和思路，毕竟漏洞挖掘和 ctf 题差距还是不一般的大（汗），于是就整理了下 android 上常见的问题，尽可能的去整理些思路出来。

 

ps：这里有一个点没有涉及，就是关于 android 端会涉及到较多的抓包分析工作，但无奈我是个二进制菜鸡，就不班门弄斧的推荐了。

 

我们都知道 android 四大组件，所以这里的大部分内容来自对瘦蛟舞大佬多年前文章的阅读笔记（汗，果然小白只能玩大佬玩剩下的）

### **Activity**

**常见的关注点**

*   activity 的生命周期
*   launch mode
*   taskAffinity
*   task and activity
*   android:exported
*   android:permission

**关键方法**

```
onCreate(Bundle savedInstanceState)
setResult(int resultCode, Intent data)
startActivity(Intent intent)
startActivityForResult(Intent intent, int requestCode)
onActivityResult(int requestCode, int resultCode, Intent data)
setResult (int resultCode, Intent data)
getStringExtra (String name)
addFlags(int flags)
setFlags(int flags)
setPackage(String packageName)
getAction()
setAction(String action)
getData()
setData(Uri data)
getExtras()
putExtra(String name, String value)

```

activity 分为四种，在考虑安全时分别从创建 activity 和使用 activity 时考虑

 

广播接收器既可以在 manifest 文件中声明，也可以在代码中进行动态的创建，并以调用 Context.registerReceiver() 的方式注册至系统。

 

注意关注类型 protectionlevel 权限

### **Broadcast**

广播需要注意广播的对象范围以及持久性带来的影响

 

**关键方法**

```
sendBroadcast(intent)
sendOrderedBroadcast(intent, null, mResultReceiver, null, 0, null, null)
onReceive(Context context, Intent intent)
getResultData()
abortBroadcast()
registerReceiver()
unregisterReceiver()
LocalBroadcastManager.getInstance(this).sendBroadcast(intent)
sendStickyBroadcast(intent)

```

ContentProvider 用来存放和获取数据并使这些数据可以被所有的应用程序访问。它们是应用程序之间共享数据的唯一方法；不包括所有 Android 软件包都能访问的公共储存区域。

### **Content Provider**

**关键方法**

```
public void addURI (String authority, String path, int code)
public static String decode (String s)
public ContentResolver getContentResolver()
public static Uri parse(String uriString)
public ParcelFileDescriptor openFile (Uri uri, String mode)
public final Cursor query(Uri uri, String[] projection,String selection, String[] selectionArgs, String sortOrder)
public final int update(Uri uri, ContentValues values, String where,String[] selectionArgs)
public final int delete(Uri url, String where, String[] selectionArgs)
public final Uri insert(Uri url, ContentValues values)
public abstract void grantUriPermission (String toPackage, Uri uri, int modeFlags) //Grant permission to access a specific Uri to another package, regardless of whether that package has general permission to access the Uri's content provider. 临时授权
public abstract void revokeUriPermission (Uri uri, int modeFlags) //Remove all permissions to access a particular content provider Uri that were previously added with grantUriPermission(String, Uri, int). 移动授权

```

### **Service**

startService 与 bindService 两者的区别就是使 Service 的周期改变. 由 startService 启动的 Service 必须要有 stopService 来结束 Service, 不调用 stopService 则会造成 Activity 结束了而 Service 还运行着. bindService 启动的 Service 可以由 unbindService 来结束, 也可以在 Activity 结束之后 (onDestroy) 自动结束.

 

**关键方法**

```
onStartCommand()
OnBind()
OnCreate()
OnDestroy()
public abstract boolean bindService (Intent service, ServiceConnection conn, int flags)
startService()
protected abstract void onHandleIntent (Intent intent)
public boolean onUnbind (Intent intent)

```

**android 端的一些常见漏洞**

 

1、组件公开安全漏洞

 

2、Content Provider 文件目录遍历漏洞

 

3、AndroidManifest.xml 中 AllowBackup 安全检测

 

4、Intent 劫持风险安全检测

 

5、数据存储安全检测

 

6、拒绝服务攻击安全检测

 

7、随机数生成函数使用错误

 

8、中间人攻击漏洞

 

9、从 sdcard 加载 dex 漏洞

 

10、Activity 被劫持风险

 

11、WebView 高危接口安全检测

 

12、WebView 明文存储密码漏洞

 

13、Android HTTPS 中间人劫持漏洞

 

14、Webview file 跨域访问

### **热修复以及脱壳和混淆**

有 bug 就要有 patch，所以不可避免的会涉及到热修复。  
同时，加壳和混淆是为了保护产权。

 

因为暂时没接触到太多样例，这里也就不班门弄斧了。

 

热修复可以理解为紧急补丁但无需重新发布版本的意思，主要分为代码修复，资源修复，so 库修复，框架有

*   native hook：dexposed，andfix， hotfix
*   java multidex：qzone， qfix， nuwa， rocoofix
*   java hook：aceso，robust
*   dex 替换：amigo，tinker
*   混合优化：sophix

[热修复具体内容可以从这看](https://www.cnblogs.com/popfisher/p/8543973.html)

 

其余的一些好资源推荐：

 

**hook 以及 frida**：

*   看雪学院有 js api 文档，当然看官方文档也很舒服
*   [roysue 大大的系列文章](https://github.com/r0ysue/AndroidSecurityStudy)
*   [泉哥博客关于 frida fuzz 的一些内容](http://riusksk.me/2019/11/30/Frida%E6%A1%86%E6%9E%B6%E5%9C%A8Fuzzing%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8/)
*   project zero：Adventures in Video Conferencing 系列
*   蒸米师傅的安卓七种武器 hook 上下篇，清晰的介绍了一个轻便的 hook 框架

**漏洞挖掘**

*   [瘦蛟舞大大的文章](https://github.com/WooyunDota/DroidDrops)
*   乌云历史镜像 ：）

**练习题**

*   只做过攻防世界的，感觉一般，不太推荐。。。
*   GitHub 上有一个 awesome mobile ctf，推荐下：[awesome-mobile-CTF](https://github.com/xtiankisutsa/awesome-mobile-CTF)

**综合**

*   看雪啊。。。还用想吗
*   非虫大佬在 isc 2016 会议上的一份 ppt，仍有很好的学习意义，github 上有分享

这篇纯粹是水文，断断续续地记录下来的，本来是打算放自己博客的，但后来一想放下自己的学习笔记或许对和我一样的小白们在道路上会有所帮助，就放个帖子。下次尽可能放些漏洞挖掘，fuzz 以及有意思的 mobile 题：）

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 2020-2-23 23:49 被 Dawuge 编辑 ，原因：