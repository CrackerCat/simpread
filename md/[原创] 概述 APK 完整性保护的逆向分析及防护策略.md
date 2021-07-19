> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268480.htm)

> [原创] 概述 APK 完整性保护的逆向分析及防护策略

目录

*   [概述 APK 完整性保护的逆向分析及防护策略](#概述apk完整性保护的逆向分析及防护策略)
*            一. 概要
*            二. 准备
*            三. 案例
*                    3.1 集成项目
*                    3.2 分析问题
*                    3.3 getPackageCodePath
*            四. 源码分析
*                    4.1 Application 的创建过程
*                    4.2 hook application
*                    4.3 成功调用
*            五. 防护策略
*            六. 总结

概述 APK 完整性保护的逆向分析及防护策略
======================

一. 概要
-----

APK 的完整性保护在 Android 逆向安全中是一个常见的话题，通常指为了防二次打包，防篡改，防独立调用 so 等，通过验证 apk 包名、签名，及 apk 包本身（如 META-INF 下签名文件，MD5 等）来达到防范的目的。笔者曾分析过一些大型的 APK，发现即使使用了包名、签名、apk 完整性验证，仍然可以通过一定手段来绕过验证。即便多年前尼古拉斯四哥的 [apk 签名爆破](https://github.com/fourbrother/HookPmsSignature)，到现在也非常好用，因此本文的案例中也集成了四哥的签名爆破。  
本文主要对 apk 完整性进行逆向及分析相应的安全策略。

二. 准备
-----

本文以 Android8.0 源码为例，涉及代理 application 的机制，会分析到 Android 中一个 application 的创建，以及如何通过代理 Application hook 掉目标 application 中有关 apk 包的属性（LoadedApk）

```
相关源码路径
frameworks\base\core\java\android\app\ActivityThread.java
frameworks\base\core\java\android\app\LoadedApk.java
frameworks\base\core\java\android\app\ContextImpl.java
frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java
frameworks\base\cmds\pm\src\com\android\commands\pm\Pm.java
frameworks\base\services\core\java\com\android\server\pm\PackageManagerService.java

```

三. 案例
-----

本着以实用主义的原则，本文决定不以分析源码为开篇，而是以一个实际的案例（如 tb 无线保镖，wb 等具有完整性保护的绕过原理相同），拆分其 so 库到自己的 Android 项目，绕过其包名签名检测，APK 完整性校验以成功调用 so。

### 3.1 集成项目

![](https://bbs.pediy.com/upload/attach/202107/804803_ZTKA2ZT5VFJYU33.png)  
首先将目标 so 拷贝出来，新应用的包名需与原来的一致，并配置好 so 所需环境，在 application 中调用四哥的 PMS 签名爆破。对于一般的项目来说，绝大多数的 so 是能调用成功的。  
这里简单提供一下目标签名的获取：

```
frida：
function call_signature(){
    Java.perform(function(){
        var context = Java.use("android.app.ActivityThread").currentApplication().getApplicationContext();
        var packageInfo = context.getPackageManager().getPackageInfo(context.getPackageName(), 64);
        //.signatures[0]
        var Signatures =packageInfo.signatures.value;
        console.log("Signatures: ",Signatures.length,Signatures[0].toCharsString())
    })
}
 
Xposed：
大多数情况下xposed hook提示class not find的原因是classloader不对
XposedHelpers.findAndHookMethod("com.kwai.hotfix.loader.app.TinkerApplication",loadPackageParam.classLoader , "attachBaseContext", Context.class, new XC_MethodHook() {
    @Override
    protected void beforeHookedMethod(MethodHookParam param) throws Throwable {    
            Context context = (Context) param.args[0];   
            String signature = context.getPackageManager().getPackageInfo(context.getPackageName(),PackageManager.GET_SIGNATURES).signatures[0].toCharsString();   
            LogUtil.d(TAG, "PackageName"+context.getPackageName()+"   signature: "+signature);   
            super.beforeHookedMethod(param);
    }
}

```

但是这里我们调用返回的却是 null。这说明了一个问题，我们的环境相对于 so 原本的环境，必然有某些地方不一致，so 仍能校验出来。且不同于某些 0335 之类的图片签名，也不同于某些在 assets 目录中读取文件的 so，我们的 demo 并未发生崩溃，也没有其他异常信息。我们已经加好了签名爆破，那么就有可能是 apk 的完整性问题了。

### 3.2 分析问题

上文猜测有可能是 apk 的完整性保护问题。那么我们应该如何绕过呢？

 

我们知道一个 release 版本的 apk 是需要签名才能打包的，并且会在 META-INF 文件下生成三个签名文件，  
MANIFEST.MF、CERT.SF、CERT.RSA。其中 CERT.RSA 文件以签名的方式 “包含” 了前两个文件，并且也会将公钥证书一同写进去。（名称解释：RSA 算法为非对称加密，通常来说，以公钥加密、私钥解密称为加解密过程；以私钥加密、公钥解密称为签名、验证的过程）

 

而里面包含的公钥证书，我们可以通过 openssl 命令提取

```
openssl pkcs7 -inform DER -in CERT.RSA -print_certs | openssl x509 -outform DER -out CERT.cer

```

提取出来的 CERT.cer 证书文件，在 010 中打开，可以发现其 hex 值正是在 Android 中获取的签名值（toCharsString()）。

 

当然，我们并不确定 so 中是否真的去校验这个签名值，更简单的方式可能直接校验 CERT.RSA  
的 MD5，而不用去提取证书。但是有一点我们可以确定，so 中一定会通过某种方式获取到原始 apk 文件。

### 3.3 getPackageCodePath

一个 apk 在安装成功后，会在 / data/app / 包名 下存有原始 apk 的信息，且这个路径是动态变化的。一般获取 apk 的完整路径，都会使用 context.getPackageCodePath() 这个方法来获取。因此，我们的目标就变成了如何 hook 掉 getPackageCodePath，并转移其指向到原本的 apk。  
当然，比较直接的方式就是通过 frida 或者 xposed 进行 hook，但是使用这两者都需要额外的成本，尤其是如果搭建模拟器服务时，会更加复杂。不过 getPackageCodePath 方法既然是在 Java 层，我们便可以利用 Java 厉害的 “武器” 反射，来手动 hook 掉他。

四. 源码分析
-------

Android 中 context 的实现类是 ContextImpl，因此 context 通过 ContextWrapper 调用到 ContextImpl 的 getPackageCodePath() 方法

```
//ContextImpl.java
@Override
    public String getPackageCodePath() {
        if (mPackageInfo != null) {
            return mPackageInfo.getAppDir();
        }
        throw new RuntimeException("Not supported in system context");
    }

```

这里的 mPackageInfo 的类型是 LoadedApk，getAppDir() 返回的是其字段 mAppDir。也就是说，如果我们能在运行时，动态替换掉 LoadedApk，并把 mAppDir 字段设置为虚假的 apk 路径，那么是否就能够绕过 apk 的完整性验证？

 

这里我们可以参考插件化的思想，应用启动时先加载代理 application，在创建代理 application 的过程中，替换为目标 application，并修改其值。因此我们需要了解一个 application 是如何创建的。

### 4.1 Application 的创建过程

首先，这张图很好的概括了，当一个应用启动后，Launcher 进程首先会通过 binder IPC 的方式将请求转发到远端服务 AMS 中（system_server 进程），接着又会通过 socket，在 Zygote 进程中 fork 出我们的应用进程，也就是所谓所有的应用程序都是由 zygote 孵化而来，最终会通过反射调用到 ActivityThread.main() 方法。  
![](https://bbs.pediy.com/upload/attach/202107/804803_EB93SMJD83JQ7M9.png)（[图来源](http://gityuan.com/2016/03/26/app-process-create/)）

 

因此，我们从 ActivityThread.main() 方法开始，作为分析 application 创建的入口点

```
//android.app.ActivityThread
public static void main(String[] args) {
    ......
    thread.attach(false);
}
 
private void attach(boolean system) {
    ......
    if (!system) {
        final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
    }
}

```

在 attach 方法中，getService() 最终会通过 binder 驱动在 server_manager 中获取已注册好的远端服务 AMS 的对象，并调用其（AMS）attachApplication 方法。

```
//ActivityManagerService.java
@Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        ......
        if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial);
            }
 }

```

调用到 bindApplication 方法中  
![](https://bbs.pediy.com/upload/attach/202107/804803_EH62HYB7BE97K92.png)  
这个方法会创建一个 AppBindData 对象，并通过 handler 的方式发送出去，这个消息将会在**主线程**的 handleMessage 里面接收，此时 swich 的分支是 BIND_APPLICATION，接着就走到 handleBindApplication 并把新创建的 AppBindData 对象传进去  
![](https://bbs.pediy.com/upload/attach/202107/804803_S9SE7URCJDUHZWE.png)  
在 handleBindApplication 中做了最关键的事情就是 makeApplication 方法，这个方法是在 data.info 也就是 LoadedApk 中

```
private void handleBindApplication(AppBindData data) {
    ......
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    ......
    Application app;
    try {
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
    }
    ......
}

```

![](https://bbs.pediy.com/upload/attach/202107/804803_GETKV72VUPGXZFR.png)

 

makeApplication 这个方法非常关键，他代表了一个 application 是如何创建的。  
1 部分首先判断当前进程的 application 是否为空，也就是判断当前进程的 application 是否已经被加载过，如果加载过，则直接返回。然后会获取此 application 的全限定类名，获取 classloader，通过 2 处，创建出 application，最后把创建出来的 application 添加进集合，并返回创建出的 application，也就是 3 处。

 

来看一下 2 处的 newApplication 中做了什么？  
![](https://bbs.pediy.com/upload/attach/202107/804803_NKV5HA3QYX9QQPK.png)

 

![](https://bbs.pediy.com/upload/attach/202107/804803_CSSDFBWVFC8VFFY.png)  
发现最终会调用到 attachBaseContext 方法，也就是通常在应用层面，继承 Application 复写的那个方法，所以 attachBaseContext 被调用的时机，其实还是很早的，此时 2、3 处的代码还没有执行到。**因此我们可以在代理 application 中复写 attachBaseContext，在此方法内通过反射，重新调用一次 makeApplication，并把新的 application 替换掉原有的 application，并完成相应属性的替换工作。**

### 4.2 hook application

这里我们会通过大量的反射获取成员对象、调用方法，因此下边的表格表示了我们需要反射获取的全部属性 \ 方法，如有忘记可以上下查看。

<table><thead><tr><th>重要字段 / 方法</th><th>所属类</th><th>描述</th></tr></thead><tbody><tr><td>currentActivityThread()</td><td>android.app.ActivityThread</td><td>该方法返回一个全局的 ActivityThread 对象</td></tr><tr><td>mBoundApplication</td><td>android.app.ActivityThread</td><td>类型为 ActivityThread 的静态内部类 AppBindData，包含着存有 apk 类的信息，LoadedApk，ApplicationInfo 等</td></tr><tr><td>info</td><td>ActivityThread$AppBindData</td><td>类型为 LoadedApk，表示一个 apk 在当前内存的信息</td></tr><tr><td>mApplication</td><td>android.app.LoadedApk</td><td>类型为 Application，表示当前进程的 Application，在 makeApplication() 过程中，最先判断此值是否为空</td></tr><tr><td>mInitialApplication</td><td>android.app.ActivityThread</td><td>Application，makeApplication() 的返回结果会赋给此值</td></tr><tr><td>mAllApplications</td><td>android.app.ActivityThread</td><td>List&lt;Application&gt;,makeApplication() 过程中，将新创建的 application 保存到集合中</td></tr><tr><td>mApplicationInfo</td><td>android.app.LoadedApk</td><td>ApplicationInfo, 封装了一个应用的基本信息</td></tr><tr><td>appInfo</td><td>ActivityThread$AppBindData</td><td>ApplicationInfo, 同上</td></tr></tbody></table>

 

上节提到，我们需要做的就是重新调用 makeApplication 创建出一个新的 application，来看核心代码

```
public class ProxyApplication extends Application {
    //新的application
    private App app;
    //当前application的attachBaseContext被调用时机很早
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        //创建虚假apk路径，将真实的apk复制进去
        boolean b = initFakeFiles(base);
        if (b){   
            //传入类的全限定名
            app = (App) makeApplication(App.class.getName());}
    }
    public Application makeApplication(String appClassName){
        try {
            //反射获取ActivityThread全局对象
            Object currentActivityThread = getCurrentActivityThread();
            // hook LoadedApk
            Object loadedApkInfo = getLoadedApk(appClassName, currentActivityThread);
            //通过反射调用makeApplication，重新加载 application
            return makeApplication(loadedApkInfo);
        }   ......
    }
    public Object getLoadedApk(String appClassName,Object currentActivityThread){
        //AppBindData
        Object mBoundApplication = getFieldValue(currentActivityThread,"mBoundApplication");
        //LoadedApk
        Object loadedApkInfo = getFieldValue(mBoundApplication, "info");
        //把当前进程的mApplication设置成null，否则调用makeApplication会直接返回mApplication
        setFieldValue(loadedApkInfo,"mApplication",null);
        //hook mAppDir 创建虚假路径
        String fakePath = new File(getFilesDir(), Utils.fakeAPK).getAbsolutePath();
        setFieldValue(loadedApkInfo,"mAppDir",fakePath);
        //handleBindApplication的时候，调用LoadedApk中的makeApplication返回赋值给mInitialApplication
        Object oldApplicaiton = getFieldValue(currentActivityThread,"mInitialApplication");
        ArrayList mAllApplications = getFieldValue(currentActivityThread,"mAllApplications");
        //删除oldApplicaiton
        mAllApplications.remove(oldApplicaiton);
        ApplicationInfo lodaedApk = getFieldValue(loadedApkInfo,"mApplicationInfo");
        ApplicationInfo appBindData = getFieldValue(mBoundApplication,"appInfo");
        //用于makeApplication中读取到的appClass名字
        lodaedApk.className = appClassName;
        appBindData.className = appClassName;
        lodaedApk.sourceDir = fakePath;
        appBindData.sourceDir = fakePath;
        return loadedApkInfo;
    }
} 
```

### 4.3 成功调用

完成 hook 工作后，我们可以验证一下，发现已经能调用成功，apk 的完整性校验已经绕过！  
![](https://bbs.pediy.com/upload/attach/202107/804803_K3MA78X56BRZMGY.png)  
虽然绕过检测的方式不是很复杂，简而言之其实就一句话，hook 掉 so 中获取的 apk 路径，但本案例仍然是相对安全的典范，这种方式大约是笔者在 2 年前发现的，不过当时笔者在其他完整性保护的项目中，是 hook libc 中 fopen 或 open 来转移指向，通过 inline hook 的方式集成在单独应用中，但在此案例中并未 hook 成功，如果排除笔者的操作失误，那么大概率此案例对 libc 中的系统库有保护操作，或者以系统调用来完成读取文件路径。  
其次，本案例的完整性检查是针对 apk 全文件的，不同于其他案例只保留 apk 中 meta-inf 文件夹下的签名文件就可以调用成功，因此需要将大概 70M 的 apk 拷贝至指定的位置，才能调用成功。

五. 防护策略
-------

通过上文分析，无论是通过 open/fopen，还是 getPackageCodePath，都是需要获得 apk 的路径，而我们绕过检测的方式也正是基于此。那么有没有其他的方式可以动态获取 apk 的路径，并且不会被上述的 hook 绕过呢？答案是有的，就是使用 pm 命令。

```
adb shell pm path <包名>

```

adb shell 的命令基本上都会通过 binder 调用到远端服务，如 pm 相关的命令最终会调用到 PMS 系统服务。我们简单看一下使用 pm path 之后做了哪些操作

```
//Pm.java
public final class Pm {
    public static void main(String[] args) {
         ......
        exitCode = new Pm().run(args);
    }
    public int run(String[] args){
        if ("path".equals(op)) {
            return runPath();
        }
    }
}

```

![](https://bbs.pediy.com/upload/attach/202107/804803_ZSEQJNK8D55VTAD.png)  
首先获取 userId，可以看到通过 pm 命令获取到的 userId 是系统用户的 Id，0。

```
private int displayPackageFilePath(String pckg, int userId) {
    try {
            PackageInfo info = mPm.getPackageInfo(pckg, 0, userId);
            if (info != null && info.applicationInfo != null) {
                System.out.print("package:");
                System.out.println(info.applicationInfo.sourceDir);
                ......
            }
        } ......
}

```

其次会调用到 pms 的 getPackageInfo 方法，在这个方法里面，最终会通过 generatePackageInfo 方法重新生成 apk 相关信息，因此我们在 application 中替换的值也就没有用了。  
这里以 Java 为例，执行 pm 命令，可以看到 pm 打印的 apk 原始路径是没有被 hook 掉的。  
![](https://bbs.pediy.com/upload/attach/202107/804803_GMZJUK9NCNB4DW2.png)  
因此，针对 apk 的完整性保护，可以采取双重验证，一方面通过 context 读取路径，另一方面通过 pms 服务获取路径。

六. 总结
-----

本文概述了 apk 完整性检测的绕过方式，并增加了一种新的检测方式，不过攻防总是相对的，即便是本文提出的保护方案，可能也会有其他的绕过方式，有兴趣的读者可以继续研究。

 

最后总结一下独立调用 so 的好处，最简单的就是搭建 Android 服务，且不依赖于第三方 hook 工具，效率很高。其次调用成功可以完全了解 so 所处环境，作为 unicorn 模拟调用的环境补全。最后，独立调用可以很好的控制 so 加载的时机，避免了在原始环境下分析 so 的困境。

 

本文案例已上传 [https://github.com/SliverBullet5563/ApkIntegrityCheck](https://github.com/SliverBullet5563/ApkIntegrityCheck)（如侵删！），谢谢！

 

参考：

 

[https://github.com/fourbrother/HookPmsSignature](https://github.com/fourbrother/HookPmsSignature)  
[http://gityuan.com/2017/04/02/android-application/](http://gityuan.com/2017/04/02/android-application/)  
[https://blog.csdn.net/AndrExpert/article/details/103535201](https://blog.csdn.net/AndrExpert/article/details/103535201)  
[http://gityuan.com/2016/03/26/app-process-create/](http://gityuan.com/2016/03/26/app-process-create/)  
[https://www.jianshu.com/p/3ef82ac4bf4f](https://www.jianshu.com/p/3ef82ac4bf4f)  
[https://bbs.pediy.com/thread-250990.htm](https://bbs.pediy.com/thread-250990.htm)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#逆向分析](forum-161-1-118.htm) [#源码分析](forum-161-1-127.htm)