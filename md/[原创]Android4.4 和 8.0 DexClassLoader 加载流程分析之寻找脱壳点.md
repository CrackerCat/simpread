> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274110.htm)

> [原创]Android4.4 和 8.0 DexClassLoader 加载流程分析之寻找脱壳点

[](#一、启动app的启动流程分析)一、启动 app 的启动流程分析
===================================

【前言】  
**Android 手机开机后 --> 启动 Zygoye 进程 --> 创建 SystemServer 进程**  
1.Zygoye 进程孵化器：Android 中的所有进程，如 系统进程、应用进程、SystemServer 进程，都是由 Zygoye 调用 fork 方法创建的  
2.SysremServer 进程：就是核心服务所在进程，核心服务如 WindowsManagerServer、PowerManagerService、ActivityManagerService 等系统服务  
3.ActivityManagerService 服务：简称 AMS，该服务由 SystemServer 启动，主要功能是控制四大组件启动和调度工作，控制应用程序的管理和调度工作  
【应用程序启动】  
**Launcher 应用 (系统主界面)--> 最终获得 ActivityManagerService-->ActivityManagerService.start()方法 -->判断要启动的应用是否存在 -->存在则直接切换到前台 -->不存在则调用 Process 类，通过 Process 类调用 Zygoye 的 fork 方法创建进程 -->调用 ActivityThread 的 main 函数**

【ActivityThread.main 函数引发的调用流程源码分析】
-----------------------------------

### 【1】创建 ActivityThread 对象、调用 Looper.loop 无限循环处理消息

```
ActivityThread类                           
 
 public static void main(String[] args) {                           
        SamplingProfilerIntegration.start();                           
 
        CloseGuard.setEnabled(false);                           
 
        Environment.initForCurrentUser();                           
 
        EventLogger.setReporter(new EventLoggingReporter());                           
 
       Security.addProvider(new AndroidKeyStoreProvider());                           
 
        Process.setArgV0("");                           
 
        Looper.prepareMainLooper();                           
 
        ActivityThread thread = new ActivityThread(); //创建ActivityThread对象                           
       thread.attach(false);                           
 
        if (sMainThreadHandler == null) {                           
            sMainThreadHandler = thread.getHandler();                           
        }                           
 
        AsyncTask.init();                           
 
       if (false) {                           
            Looper.myLooper().setMessageLogging(new                           
                    LogPrinter(Log.DEBUG, "ActivityThread"));                           
        }                           
 
        Looper.loop();     //启动Looper.loop无限循环处理消息                       
 
        throw new RuntimeException("Main thread loop unexpectedly exited");                           
    }                           
} 
```

### 【2】msg.target.dispatchMessage(msg) 进行分发消息

```
Looper类                   
 
 public static void loop() {                   
        final Looper me = myLooper();                   
        if (me == null) {                   
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");                   
        }                   
        final MessageQueue queue = me.mQueue;                   
        Binder.clearCallingIdentity();                   
        final long ident = Binder.clearCallingIdentity();                   
 
       for (;;) {                   
            Message msg = queue.next(); // might block                   
            if (msg == null) {                   
                return;                   
            }                   
 
           Printer logging = me.mLogging;                   
           if (logging != null) {                   
               logging.println(">>>>> Dispatching to " + msg.target + " " +                   
                        msg.callback + ": " + msg.what);                   
           }                   
 
            msg.target.dispatchMessage(msg);                   
 
           if (logging != null) {                   
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);                   
           }                   
 
          // Make sure that during the course of dispatching the                   
           // identity of the thread wasn't corrupted.                   
           final long newIdent = Binder.clearCallingIdentity();                   
           if (ident != newIdent) {                   
                Log.wtf(TAG, "Thread identity changed from 0x"                   
                        + Long.toHexString(ident) + " to 0x"                   
                       + Long.toHexString(newIdent) + " while dispatching to "                   
                        + msg.target.getClass().getName() + " "                   
                        + msg.callback + " what=" + msg.what);                   
            }                   
 
            msg.recycle();                   
        }                   
    }
```

### 【3】handleMessage(msg) 进行消息处理

```
Handler类               
 
    public void dispatchMessage(Message msg) {               
        if (msg.callback != null) {               
            handleCallback(msg);               
        } else {               
            if (mCallback != null) {               
                if (mCallback.handleMessage(msg)) {               
                    return;               
                }               
            }               
           handleMessage(msg);               
       }               
    }
```

### 【4】通过消息类型分别调用 handleLaunchActivity(r, null)[Activity 流程代码] 和 handleBindApplication(data)[Application 流程代码]

```
ActivityThread$H类                               
 
 public void handleMessage(Message msg) {                               
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));                               
            switch (msg.what) {                               
                case LAUNCH_ACTIVITY: {                               
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");                               
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;                               
 
                    r.packageInfo = getPackageInfoNoCheck(                               
                            r.activityInfo.applicationInfo, r.compatInfo);                               
                    handleLaunchActivity(r, null);                               
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);                               
                } break;                               
                case RELAUNCH_ACTIVITY: {                               
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");                               
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;                               
                    handleRelaunchActivity(r);                               
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);                               
                } break;                               
    ...                           
                case BIND_APPLICATION:                               
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");                               
                    AppBindData data = (AppBindData)msg.obj;                               
                    handleBindApplication(data);                               
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);                               
                    break;
```

### [](#【5】activity调用流程)【5】Activity 调用流程

handleLaunchActivity(r, null)-->performLaunchActivity(r, customIntent)--> mInstrumentation.callActivityOnCreate(activity, r.state)-->activity.performCreate(icicle)-->onCreate(icicle)  
注意，此时调用到的 onCreate 方法就是我们平常写 Android 代码时候的 Activity 的 onCreate 方法  
![](https://bbs.pediy.com/upload/attach/202208/941979_B7978ZVTKC78R3A.png)

### [](#【6】application的调用流程)【6】Application 的调用流程

handleBindApplication(data)-->data.info.makeApplication(data.restrictedBackupMode, null)-->mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext)--> app.attach(context)-->attachBaseContext(context)  
handleBindApplication(data)-->mInstrumentation.callApplicationOnCreate(app)--> app.onCreate()  
可以分析到 Application 最终是分别调用了两个方法 attachBaseContext 和 onCreate，此处的 onCreate 和 Activity 的 onCreate 是两码事

### [](#【7】小结)【7】小结

综上分析可以得出，在真正调用到 Activity 的 onCreate 方法之前，还会经过两个关键的函数，就是 Application 的 attachBaseContext 和 onCreate 方法  
【可以做什么？】  
可以在 Application 调用的函数中进行真正的 dex 代码的加载替换。达到加壳的目的，因为在调用 Activity 之前回调用 Application 的两个函数，所以在此期间将真正程序的代码给替换进来即可。

[](#二、classloader的加载流程-找脱壳点)二、ClassLoader 的加载流程 - 找脱壳点
======================================================

【前言】  
了解 Android 的 ClassLoader 的继承关系  
![](https://bbs.pediy.com/upload/attach/202208/941979_FG8UQJFGAKXB4GD.png)

1.Android4.4 Dalvik 模式下加载 Dex 流程
--------------------------------

### [](#【1】从dexclassloader开始入手分析，因为加载dex文件一般调用dexclassloader创建对象，发现该类的构造方法调用了父类类构造)【1】从 DexClassLoader 开始入手分析，因为加载 Dex 文件一般调用 DexClassLoader 创建对象，发现该类的构造方法调用了父类类构造

【文件目录】libcore  
【类名】DexClassLoader  
【构造方法】public DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent)  
【父类构造】super(dexPath, new File(optimizedDirectory), libraryPath, parent);  
继承自父类 BaseDexClassLoader(pathClassLoader 也是继承自此父类)  
父类构造方法  
![](https://bbs.pediy.com/upload/attach/202208/941979_PHXTNFDY4CRCGU5.png)

### [](#【2】basedexclassloader是一个关键的类，很多的实现方法都封装于此类，调用构造向父类传递了类加载器的父类)【2】BaseDexClassLoader 是一个关键的类，很多的实现方法都封装于此类，调用构造向父类传递了类加载器的父类

【文件目录】libcore  
【类名】BaseDexClassLoader  
【构造方法】public BaseDexClassLoader(String dexPath, File optimizedDirectory, String libraryPath, ClassLoader parent)  
【父类构造】super(parent);  
![](https://bbs.pediy.com/upload/attach/202208/941979_9GCEBUYRPCA22TX.png)

### [](#【3】classloader唯一的作用就是指定了父类)【3】ClassLoader 唯一的作用就是指定了父类

【文件目录】libcore  
【类名】ClassLoader  
【构造方法】protected ClassLoader(ClassLoader parentLoader)  
【初始化】ClassLoader(ClassLoader parentLoader, boolean nullAllowed){ parent = parentLoader;}  
![](https://bbs.pediy.com/upload/attach/202208/941979_46W73MYZ643RMD5.png)

### [](#【4】basedexclassloader创建dexpathlist对象)【4】BaseDexClassLoader 创建 DexPathList 对象

【构造方法初始化】this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);  
【类成员】private final DexPathList pathList;  
![](https://bbs.pediy.com/upload/attach/202208/941979_6HBDCY4CQ64F2UF.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_9DAN25Z4E97GF7K.png)

### [](#【5】dexpathlist构造函数)【5】DexPathList 构造函数

【文件目录】libcore  
【类名】DexPathList  
【构造方法】public DexPathList(ClassLoader definingContext, String dexPath, String libraryPath, File optimizedDirectory)  
【成员变量】private final Element[] dexElements;  
5.1 对类加载器、dex 文件路径、文件优化路径等参数基本判断  
5.2 调用方法 makeDexElements 对成员变量赋值 dexElements  
![](https://bbs.pediy.com/upload/attach/202208/941979_TH5NKRBR8ZBJ86B.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_TB3QWWXVUVGVY4H.png)

### [](#【6】分析makedexelements方法)【6】分析 makeDexElements 方法

【方法原型】private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory  
6.1 判断文件名后缀  
6.2 loadDexFile 加载 dex 后缀的文件  
![](https://bbs.pediy.com/upload/attach/202208/941979_64J6H55NEAZVZMA.png)  
6.3 根据加载进来的 dex 文件 new Element 对象，并且赋值到成员变量中  
6.4 返回局部变量 Element 对象数组  
![](https://bbs.pediy.com/upload/attach/202208/941979_7GN2TBVNVTKXBX6.png)

### [](#【7】分析loaddexfile方法)【7】分析 loadDexFile 方法

【方法原型】private static DexFile loadDexFile(File file, File optimizedDirectory)  
7.1 如果不存在优化后的 dex 文件，则直接创建 DexFile 类对象  
7.2 如果存在优化后的 dex 文件，则调用 DexFile 类的 loadDex 方法加载 dex 文件  
![](https://bbs.pediy.com/upload/attach/202208/941979_CX5PG2G2ZR5NX35.png)  
7.3 这里对于优化后的文件，以及 dex 如何被优化的，在稍后有专门描述该内容

### [](#【8】分析dexfile类构造方法)【8】分析 DexFile 类构造方法

【文件目录】libcore  
【类名】DexFile  
【构造方法】public DexFile(String fileName) throws IOException  
【成员变量】private int mCookie;  
8.1 调用方法 openDexFile 获取 Cookie 值  
![](https://bbs.pediy.com/upload/attach/202208/941979_9D5KV48RZ2MQDY9.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_QUW6S7FKWGTEU4Y.png)

### [](#【9】分析opendexfile方法)【9】分析 openDexFile 方法

【参数说明】：参数 1：dex 路径名；参数 2：优化文件路径名  
【函数说明】：调用 openDexFileNative 方法进入 c++ 代码，并且返回 cookie 值  
![](https://bbs.pediy.com/upload/attach/202208/941979_3F3QT2H837ZA463.png)

### 【10】进入 native 层的 Dalvik_system_DexFile.cpp 文件分析

【文件目录】dalvik  
【文件名】dalvik_system_DexFile  
【方法】static void Dalvik_dalvik_system_DexFile_openDexFileNative(const u4 _args, JValue_ pResult)  
10.1 java 层进入 native 层，获得参数，并且转换为 c 的类型格式  
![](https://bbs.pediy.com/upload/attach/202208/941979_Y9PNMUZB95RGHPK.png)  
10.2 dvmClassPathContains 判断 dex 文件路径是否当前类加载器加载或者存在的  
![](https://bbs.pediy.com/upload/attach/202208/941979_5KBX6MMPFV8YNEA.png)  
10.3 hasDexExtension 如果以 dex 结尾返回 true  
10.4 dvmRawDexFileOpen 打开没有被优化的 dex 文件  
10.5 pDexorJar 管理 dex 文件内部结构  
![](https://bbs.pediy.com/upload/attach/202208/941979_4PTRUUCHJKTKDEM.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_7P47TT9P5K36TBV.png)  
【如果类路径包含特定路径返回 true】  
![](https://bbs.pediy.com/upload/attach/202208/941979_G6WN52KTPQV7VMR.png)  
【如果 name 以 dex 结尾，返回 true】  
![](https://bbs.pediy.com/upload/attach/202208/941979_5S8U3ZCCN2X5SQ6.png)

### 【11】分析 RawDexFile.cpp 文件

【文件目录】vm/dalvik  
【文件名】RawDexFile.cpp  
【方法】int dvmRawDexFileOpen(const char _fileName, const char_ odexOutputName, RawDexFile** ppRawDexFile, bool isBootstrap)  
11.1 dvmOptimizeDexFile 生成优化版的 dex 文件  
![](https://bbs.pediy.com/upload/attach/202208/941979_PHWFAS37PMHUJMQ.png)

### 【12】分析 DexPrepare.cpp 文件

【文件目录】vm/dalvik  
【文件名】DexPrepare.cpp  
【方法】bool dvmOptimizeDexFile(int fd, off_t dexOffset, long dexLength, const char* fileName, u4 modWhen, u4 crc, bool isBootstrap)  
![](https://bbs.pediy.com/upload/attach/202208/941979_4E8NV8BVA9WZP5U.png)

### [](#【13】通过创建的新线程实现的优化代码中，可以找到脱壳点)【13】通过创建的新线程实现的优化代码中，可以找到脱壳点

13.1 包括 Dex 文件缓冲区和长度  
![](https://bbs.pediy.com/upload/attach/202208/941979_JDZKSA32Z59GM9V.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_7X7V8A3CVFCZ8ZC.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_TJC4RWXYNSQ9FQ2.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_HZ5RNMTZJWGC9A9.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_54D8QYY5MUJ63Y4.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_7BCSYTGXM2CJ23J.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_4ZJUP4E7RXBKZDQ.png)

### [](#【14】小结)【14】小结

14.1 加载 Dex 的过程中，加载一个 dex 文件并赋值 DexPathList 的字段 Element[] dexElements  
14.2 创建一次 Dex 对象，打开 dex 文件时回赋值 mCookie 值  
14.3 进入 native 层会对 dex 文件进行优化

2.Android8.0 Art 模式下加载 Dex 流程 - Native 层开始分析
--------------------------------------------

### 【1】/art/runtime/native/dalvik_system_DexFile.cc/ DexFile_openDexFileNative

![](https://bbs.pediy.com/upload/attach/202208/941979_WEN455SGUZKTSGS.png)

### [](#【2】opendexfilesfromoat)【2】OpenDexFilesFromOat

![](https://bbs.pediy.com/upload/attach/202208/941979_PZY4D5JHTU7GGDJ.png)

### [](#【3】makeuptodate)【3】MakeUpToDate

![](https://bbs.pediy.com/upload/attach/202208/941979_DSF5U4BBWEPK8TR.png)  
3.1 GenerateOatFileNoChecks  
![](https://bbs.pediy.com/upload/attach/202208/941979_GFUAC75JF3CYRYC.png)  
3.2 Dex2Oat  
![](https://bbs.pediy.com/upload/attach/202208/941979_EUR8ZVUCZF38S4B.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_FA73AS565QP96UT.png)  
3.3 Exec  
![](https://bbs.pediy.com/upload/attach/202208/941979_E6TAX6WVGEBX469.png)  
3.4 ExecAndReturnCode  
![](https://bbs.pediy.com/upload/attach/202208/941979_TK43XMYKKYRZEKZ.png)

### 【4】/art/dex2oat/dex2oat.cc

![](https://bbs.pediy.com/upload/attach/202208/941979_EH7QZ8R8H4JG527.png)  
4.1 Dex2oat()  
![](https://bbs.pediy.com/upload/attach/202208/941979_5PTCTJVVSDA4FME.png)  
4.2 Setup()  
![](https://bbs.pediy.com/upload/attach/202208/941979_AMDKNMDHS94TWEP.png)

### [](#【5】如果阻断了oat的生成)【5】如果阻断了 Oat 的生成

![](https://bbs.pediy.com/upload/attach/202208/941979_9EXTVS55X5M7NTP.png)  
【OpenDexFilesFromOat 函数中】  
![](https://bbs.pediy.com/upload/attach/202208/941979_Q2VTVGVMFMTC44D.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_STY7QQB7E8482VS.png)

### 【6】 DexFile::Open

【脱壳点】OpenAndReadMagic、DexFile::OpenFile  
![](https://bbs.pediy.com/upload/attach/202208/941979_SX8JY8FAGQMUYYP.png)  
6.1 DexFile::OpenFile  
![](https://bbs.pediy.com/upload/attach/202208/941979_6X6YNA4KKDMAKPA.png)  
6.2 OpenCommon  
![](https://bbs.pediy.com/upload/attach/202208/941979_627JWAK5CMMF86C.png)

3.Android8.0 Art 模式下 ImMemoryDexClassLoader 加载 Dex 流程 - Native 层开始分析
--------------------------------------------------------------------

【特点】：仅加载 DEX 文件，并不会对文件进行优化生成 oat 文件

### [](#【1】inmemorydexclassloader)【1】InMemoryDexClassLoader

【文件目录】libore  
【父类】BaseDexClassLoader  
![](https://bbs.pediy.com/upload/attach/202208/941979_5WPUUWAQW2R9C8F.png)

### [](#【2】basedexclassloader)【2】BaseDexClassLoader

【父类】ClassLoader  
2.1 父类构造函数，设置父类变量 parent  
2.2 创建 DexPathList 对象  
![](https://bbs.pediy.com/upload/attach/202208/941979_B8DZPN4T4PT3DCV.png)

### [](#【3】dexpathlist)【3】DexPathList

【构造函数】public DexPathList(ClassLoader definingContext, ByteBuffer[] dexFiles)  
![](https://bbs.pediy.com/upload/attach/202208/941979_WBUW2398TVC633R.png)  
【3.1】makInMemoryDexElements  
返回值：Element 数组（dex 文件信息数组）  
dexFiles：dex 缓冲区数组  
创建 DexFile 文件对象并根据 DexFile 对象创建 Element 对象  
![](https://bbs.pediy.com/upload/attach/202208/941979_QK5R8ZYV8AXFWNV.png)

### [](#【4】dexfile)【4】DexFile

【构造方法】：DexFile(ByteBuffer buf) throws IOException  
![](https://bbs.pediy.com/upload/attach/202208/941979_RFQRA2U2K7Q8T27.png)  
【4.1】openInMemoryDexFile  
调用 native 层方法进入 c++  
![](https://bbs.pediy.com/upload/attach/202208/941979_6K94WV32C7MZ8S2.png)

### 【5】dalvik_system_DexFile.cc 文件

【文件目录】/art/runtime/native/dalvik_system_DexFile.cc  
【5.1】DexFile_createCookieWithDirectBuffer  
![](https://bbs.pediy.com/upload/attach/202208/941979_T2SUPKM2BQUZRR9.png)  
【5.2】DexFile_createCookieWithArray  
![](https://bbs.pediy.com/upload/attach/202208/941979_RGRHFU8AFTS9G5Y.png)  
【5.3】 CreateSingleDexFileCookie  
创建 DexFile 实例对象和 ConvertDexFilesToJavaArray  
![](https://bbs.pediy.com/upload/attach/202208/941979_JWFNXZCQMHRDGNS.png)  
【5.4】CreateDexFile  
DexFile::Open 和 dex_file.release()  
![](https://bbs.pediy.com/upload/attach/202208/941979_NJ77R93GHQ879HM.png)  
【5.5】/art/runtime/dex_file.cc/DexFile::Open  
![](https://bbs.pediy.com/upload/attach/202208/941979_S936Z5T5V5B6BA6.png)  
【5.6】DexFile::OpenCommon  
![](https://bbs.pediy.com/upload/attach/202208/941979_BQX2E2JYGBWPSYG.png)  
【5.7】DexFile 构造函数  
![](https://bbs.pediy.com/upload/attach/202208/941979_KFC8A8NBPPHFMVE.png)  
【5.8】InitializeSectionsFromMapList  
判断比较 Dex 文件格式是否有误  
![](https://bbs.pediy.com/upload/attach/202208/941979_XFDVDHF2D6YU937.png)

### 【6】/art/runtime/dex_file_verifier.cc/DexFileVerifier::Verify

![](https://bbs.pediy.com/upload/attach/202208/941979_WAKN4TZSVVTBV2B.png)  
![](https://bbs.pediy.com/upload/attach/202208/941979_DBBSGJFQKEGCCU2.png)

### [](#【7】convertdexfilestojavaarray)【7】ConvertDexFilesToJavaArray

![](https://bbs.pediy.com/upload/attach/202208/941979_R7335QW2FPVF6QE.png)

[[2022 夏季班]《安卓高级研修班 (网课)》月薪两万班招生中～](https://www.kanxue.com/book-section_list-83.htm)

[#基础理论](forum-161-1-117.htm)