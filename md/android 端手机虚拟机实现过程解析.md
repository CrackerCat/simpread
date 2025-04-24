> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2025427-1-1.html)

> [md]## android 端手机虚拟机实现过程 以下是 **VirtualAPP 的执行流程图 **：我们通过这样的一个开源的 **virtualApp......

![](https://avatar.52pojie.cn/data/avatar/001/97/85/65_avatar_middle.jpg)chenchenchen777 _ 本帖最后由 chenchenchen777 于 2025-4-21 18:02 编辑_  

### android 端手机虚拟机实现过程

以下是 **VirtualAPP 的执行流程图**：

![](https://attach.52pojie.cn/forum/202504/18/190850f4g4jguglu7gwp6f.png)

**image-20250418190817332.png** _(137.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3MjI5NXxiZGQzNmQ0Y3wxNzQ1NDYyNTUyfDIxMzQzMXwyMDI1NDI3&nothumb=yes)

2025-4-18 19:08 上传

我们通过这样的一个开源的 **virtualApp** 来了解一下，一个应用程序是怎么去实现在虚拟机里面运行的。

#### Virtual space

首先是应用程序的下载安装。实际上，在虚拟机中运行的应用程序是下载到的虚拟机空间（virtual space）中的。

#### Virtual Framework

虚拟机中的 App（比如 App1、App2）原本会直接调用系统 Framework，但是在虚拟机中的 APP 需要的就不同

*   “以为” 自己访问的是系统服务，
*   实际上却访问的是 VA 模拟的服务（Hook 之后的）。

##### **Android Framework**

**Android Framework** 是 Android 系统提供的一整套 API，它是 **系统级服务（如 AMS、PMS、LocationService）** 的接口，比如：

*   `ActivityManagerService (AMS)`：管理 Activity 的启动与调度。
*   `PackageManagerService (PMS)`：管理应用包的安装与信息。
*   `LocationManager`：提供位置信息。
*   `ClipboardManager`：剪贴板服务。

这些都属于 **系统原生的 Android Framework**，运行在系统进程（如 `system_server`）中，不是用户 App 能随意改动的。而在虚拟机中的 Framework，则是自开发的可改动的 Framework。

而在 Virtual Framework 中，需要做的事情就是让 **APP 觉得自己是在原生系统下执行的**，根据 Android Frameword 中的应用程序的执行流程，我们可以知道的是

##### 一个**应用程序的执行流程**大体是这样的：

*   点击桌面程序
*   将 Activity 顶上第一个设置为**暂停状态**，**等待启动的应用**
*   判断进程，先进程再程序
*   通过 **Zygote 开始 fork()** 应用程序进程了
*   执行 **ActivityThread.main()**
*   ActivityThread 通过 Binder 将自身的 ApplicationThread 传给 AMS，方便 **AMS** 通过 **ActivityThread 启动应用**
*   **AMS** 通过 ApplicationThread.scheduleLaunchActivity() 通知**启动 Activity**
*   **ActivityThread** 通过反射加载目标 **Activity 类**
*   调用 Instrumentation.callActivityOnCreate()，执行程序的 OnCreate() 函数

应用的真正启动是 `ActivityThread` 执行的，但整个过程是由 `AMS` 控制调度的，双方通过 Binder 建立通信桥梁。

#### 虚拟 App vs 原生安装 App

<table><thead><tr><th>方面</th><th>虚拟机中的 App（VA App）</th><th>系统安装的 App（原生 App）</th></tr></thead><tbody><tr><td><strong>安装方式</strong></td><td>并未真正通过系统 PMS 安装，仅在 VA 虚拟空间中注册</td><td>通过系统 PMS 安装，注册了 Activity、Service 等</td></tr><tr><td><strong>运行路径</strong></td><td>安装路径、数据路径等都是被 <strong>重定向 / 虚拟的</strong>（如 <code>/data/data/com.xxx -&gt; /data/data/io.virtual.app/space/0/...</code>）</td><td>使用 Android 系统真实的 <code>/data/data/com.xxx</code> 路径</td></tr><tr><td><strong>访问系统服务</strong></td><td>所有访问系统服务的调用（如 Location、Clipboard）都被 VA Hook 拦截和 “伪造”</td><td>调用系统原生 <code>ServiceManager</code> 提供的服务，直接连接 <code>system_server</code></td></tr><tr><td><strong>AMS / PMS 管理</strong></td><td>由 VA 模拟的 AMS/PMS 进行调度（Activity 启动、Service 调用等）</td><td>由系统的 <code>ActivityManagerService</code> 和 <code>PackageManagerService</code> 控制</td></tr><tr><td><strong>权限模型</strong></td><td>权限请求和判断被 VA 接管（可以伪造授予，也可以强制拦截）</td><td>权限由 Android 系统控制，用户手动授权</td></tr><tr><td><strong>多开与隔离能力</strong></td><td>可以任意多开，同一 App 多个实例之间相互隔离</td><td>系统默认一个包名只允许一个实例，数据也无法隔离</td></tr><tr><td><strong>文件 / IO 控制</strong></td><td>VA Hook 底层文件系统访问，可实现只读 / 禁止 / 伪造 / 隔离等策略</td><td>系统文件访问直接作用于真实文件系统</td></tr><tr><td><strong>运行时感知</strong></td><td>可以 “欺骗” App，使其以为自己运行在正常系统中（比如感知不到虚拟环境）</td><td>真实环境，App 可以访问所有支持的设备资源</td></tr><tr><td><strong>安全性控制</strong></td><td>更灵活，可以模拟环境、劫持函数、限制行为</td><td>安全性高但开放度小，无法灵活控制运行环境</td></tr></tbody></table>

那么实际上在虚拟机下应用程序的执行流程也应该是这样的，但是不同的是，这里的 **Server 服务不是 Android 原生下的 framework，而是虚拟机的 framework**。

而这里就应用到的是 APP HOOK 的主要实现了

#### APP HOOK

##### Hook 的目标是 **系统服务类**，主要分两种：

###### 1. Java 层服务（运行在 system_server）

VA 会通过 **反射、代理（比如 Binder 代理）或动态注入**，替换系统服务对象。

举个例子：

*   App 调用 `Context.getSystemService("location")` 拿到 `LocationManager`。
*   正常系统下，它返回的是 Binder 连接到 system_server 的服务。
*   VA Hook 后返回的是它自己实现的 “假 LocationManager”，比如总是返回一个虚假的地理位置。

###### 2. Binder 接口 Hook

*   VA 使用类似 **“Binder IPC 劫持”** 的方式，在 Java 层模拟 AMS、PMS 等 Binder 服务。
*   这就像劫持了一条 “电话线”，App 以为在和系统通信，其实是和 VA 自己通信。

细节的说一下可能出现的具体实现：

*   **App 调用 `ActivityManager.startActivity()`**
*   **APP Hook 中的 AMS Hook** 通过代理劫持系统服务调用（比如修改 `IActivityManager` 的 Binder）
*   它并**不真的去调用系统 AMS**，而是把调用重定向到了 VirtualApp 自己实现的 **VA Server → AMS**
*   VA Server 的 AMS 执行启动流程（比如在 VirtualActivityStack 中注册一个 Activity 记录）
*   最后，伪装成系统返回正常的 `ActivityResult`，App 觉得自己 “真的启动了 Activity”

而在 APP HOOK 拦截完成之后，VA Server 就需要开始去提供对应的服务来实现拦截之后的服务请求。

所以：APP Hook 是用来 “**拦截”App 调用系统服务行为**的；而 VA Server 是用来 **“虚拟提供” 这些服务的实现端。**

#### Virtual Native

在这一层主要为了完成 2 个工作，**IO 重定向**和 **VA APP 与 Android 系统交互**的请求修改。

##### IO 重定向

比如在一些 APP 中，会把一些配置、缓存、数据库等文件的路径**写死**在代码里

```
new File("/data/data/com.example.app/shared_prefs/config.xml")
fopen("/data/data/com.example.app/files/config.dat", "r");
```

这种情况下的，用虚拟机打开的程序就**无法获取适配对应路径的文件和配置信息**。

所以说出现了 **VA Native** 的存在的意义：

*   拦截所有 `open`, `fopen`, `access`, `stat` 等系统调用
*   判断路径是否是虚拟 App 的目标路径
*   替换为虚拟路径，到真实的程序的虚拟路径的位置

不过这里应该算得上是 Native 层的 HOOK 了。

##### JNI Native 函数

以上的 APP HOOK 都是基于 Java 层进行的，但是相应的 Android 有大量的系统行为是通过 Native 实现的。这时候对应的应用程序适配就需要 Native HOOK 来实现

<table><thead><tr><th>情况</th><th>举例</th><th>解决方法</th></tr></thead><tbody><tr><td>直接访问文件</td><td><code>open</code>, <code>fopen</code>, <code>access</code></td><td>hook libc</td></tr><tr><td>直接调用系统服务</td><td><code>getuid()</code>, <code>getpid()</code></td><td>hook syscall</td></tr><tr><td>访问特殊设备</td><td><code>/dev/*</code>, <code>/proc/*</code></td><td>重定向路径</td></tr><tr><td>ART Runtime 优化</td><td><code>libart.so</code> 调用</td><td>hook libart</td></tr><tr><td>动态链接时 resolve</td><td><code>dlsym</code> 取函数地址</td><td>hook linker</td></tr></tbody></table>

这里举例的程序应用就是需要通过 HOOK so 层来实现分析和获取适配对应路径的文件和配置信息。

### 源码分析部分

#### mirror

在虚拟机中很多的细节其实是通过 HOOK，或者通过反射的操作调用实现的。比如拦截真实的 AMS,PMS 等服务，返回自写的各种服务。而在虚拟机框架中为了使得调用拦截等更加方便则出来对于反射等的封装。而在这里的封装就叫 **mirror**

正常执行下去拦截改定位:

```
private void hookInstrumentation() {
    try {
        // 加载 ActivityThread 类
        Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");

        // 获取 'currentActivityThread' 方法
        Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
        currentActivityThreadMethod.setAccessible(true);

        // 调用该方法获取当前的 ActivityThread 实例
        Object currentThread = currentActivityThreadMethod.invoke(null);

        // 获取 'mInstrumentation' 字段
        Field instrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
        instrumentationField.setAccessible(true);

        // 获取原始的 Instrumentation 实例
        Instrumentation originalInstrumentation = (Instrumentation) instrumentationField.get(currentThread);

        // 创建 InstrumentationDelegate 并替换原始的 Instrumentation
        InstrumentationDelegate instrumentationDelegate = new InstrumentationDelegate(originalInstrumentation);
        instrumentationField.set(currentThread, instrumentationDelegate);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

mirror **拦截改定位**

```
private void hookInstrumentation(){
        Object currentThread ActivityThread.currentActivityThread.call();
        Instrumentation originInstrumentation         ActivityThread.mInstrumentation.get(currentThread);
        ActivityThread.mInstrumentation.set(currentThread,new InstrumentationDelegate(originInstrumentation));
```

这里实际上就去做了对应的函数封装，从而更加快速实现的代码逻辑

#### Java 层 HOOK

举例**应用调用 AMS 执行流程**的过程来看看 Java 层 HOOK

Android 系统中，所有四大组件的管理都要通过系统的 **ActivityManagerService（AMS）** 完成，客户端通过 **`IActivityManager` 接口** 调用 AMS。

所以 **Hook AMS = 控制所有组件的启动流程**！得到`IActivityManager` 接口就可以**拦截执行流程**

```
Activity.startActivity(Intent)
    ↓
Instrumentation.execStartActivity(...)
    ↓
ActivityManager.getService().startActivity(...)
```

这里会走 **ActivityManager.getService()** ，而在 android8.0 的位置

```
// frameworks/base/core/java/android/app/ActivityManager.java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

// frameworks/base/core/java/android/util/Singleton.java
private static final Singleton<IActivityManager> IActivityManagerSingleton =
    new Singleton<IActivityManager>() {
        @Override
        protected IActivityManager create() {
            // 注意这行
            return ActivityManagerNative.getDefault();
        }
    };
```

```
getService() 会返回 IActivityManagerSingleton.get();
```

所以在

```
public class ActivityManagerStub extends MethodInvocationProxy<MethodInvocationStub<IInterface>> {
    public ActivityManagerStub() {
        // 基于 mirror 获取 AMS 的 IActivityManager 接口对象，并创建其动态代理对象
        super(new MethodInvocationStub<>(ActivityManagerNative.getDefault.call()));//这里通过反射去获取系统 IActivityManager 实例。
    }  

public void inject() throws Throwable {
        if (BuildCompat.isOreo()) {
            // Android 8.x 及以上版本
            Object singleton = ActivityManager.IActivityManagerSingleton.get();
            // 替换原 AMS 对象为我们的动态代理对象
            Singleton.mInstance.set(singleton, getInvocationStub().getProxyInterface());
        } else {
            // Android 8.x 以下的实现（略）
        }
    }
```

用反射将其 `mInstance` 替换成我们刚才构造的 **代理对象**，应用对 AMS 的所有调用（比如 `startActivity()`），都会经过我们这个 **Hook 代理对象**。

```
// 方法 hook 实现
    static class StartActivity extends MethodProxy {
        @Override
        public String getMethodName() {
            return "startActivity";
        }
        @Override
        public Object call(Object who, Method method, Object... args) throws Throwable {
            // 可以对参数进行处理
            int res = VActivityManager.get().startActivity(args);
            return res;
        }
    }
}
```

在这里去调用 int res = VActivityManager.get().**startActivity(args)**;  此处直接将调用转发到 `VActivityManager` 中（**VirtualApp 的虚拟 AMS**）。

**“拦截 AMS 服务”** 的核心目的，实际上就是为了拿到 `IActivityManager` 的实例，并用我们构造的 **代理对象**去**替换掉它**。因为 Android 应用中的 **组件调度**（包括启动 Activity、Service、广播等）全都需要通过这个接口来向系统发送请求。

```
ActivityManager.getService().startActivity(...)
ActivityManager.getService().broadcastIntent(...)
ActivityManager.getService().startService(...)
```

这些都是 `IActivityManager` 的方法。

#### native 层 hook

上面也已经说过了 Native 层 HOOK 的原理，这里就执行来看看 io 重定向，实现修改的文件读写的路径的操作

```
@SuppressLint("SdCardPath")
private void startIOUniformer() {
    ApplicationInfo info = mBoundApplication.appInfo;
    int userId = VUserHandle.myUserId();
    String wifiMacAddressFile = deviceInfo.getwifiFile(userId).getPath();

    // 重定向 WiFi 相关地址
    NativeEngine.redirectDirectory("/sys/class/net/wlan0/address", wifiMacAddressFile);
    NativeEngine.redirectDirectory("/sys/class/net/eth0/address", wifiMacAddressFile);
    NativeEngine.redirectDirectory("/sys/class/net/wifi/address", wifiMacAddressFile);

    // 重定向 data/data 路径
    NativeEngine.redirectDirectory("/data/data/" + info.packageName, info.dataDir);
    NativeEngine.redirectDirectory("/data/user/0/" + info.packageName, info.dataDir);

    if (Build.VERSION.SDK_INT > Build.VERSION_CODES.N) {
        // Android 7.0 以上
        NativeEngine.redirectDirectory("/data/user_de/0/" + info.packageName, info.dataDir);

        String libPath = VEnvironment.getAppLibDirectory(info.packageName).getAbsolutePath();
        String userLibPath = new File(
            VEnvironment.getUserSystemDirectory(userId),
            info.packageName + "/lib"
        ).getAbsolutePath();

        // 重定向 native lib 路径
        NativeEngine.redirectDirectory(userLibPath, libPath);
        NativeEngine.redirectDirectory("/data/data/" + info.packageName + "/lib/", libPath);
        NativeEngine.redirectDirectory("/data/user/0/" + info.packageName + "/lib/", libPath);
    }

    // 虚拟存储处理
    VirtualStorageManager vsManager = VirtualStorageManager.get();
    String vsPath = vsManager.getVirtualStorage(info.packageName, userId);
    boolean enable = vsManager.isVirtualStorageEnable(info.packageName, userId);

    if (enable && vsPath != null) {
        File vsDirectory = new File(vsPath);
        if (vsDirectory.exists() || vsDirectory.mkdirs()) {
            HashSet<String> mountPoints = getMountPoints();
            for (String mountPoint : mountPoints) {
                NativeEngine.redirectDirectory(mountPoint, vsPath);
            }
        }
    }

    NativeEngine.enableIORedirect();
}
```

可以看到这里利用 redirectDirectory 函数，去重定向了对应可能读取的位置，替换成了虚拟机对应位置的内容。

```
//int faccessat(int dirfd,const char *pathname,int mode,int flags);
HOOK DEF(int,faccessat,int dirfd,const char *pathname,int mode,int flags){
    int res;
    const char *redirect_path relocate_path(pathname,&res);
    int ret syscall(_NR_faccessat,dirfd,redirect_path,mode,flags);
    FREE(redirect_path,pathname);
    return ret;
}
```

该宏会生成一个以 `new_` 为前缀的函数，例如：

```
int new_faccessat(...) { ... }
```

*   判断是否需要重定向（如 `/data/data/com.xxx` → `/virtual/data/com.xxx`）
*   如果是，则返回新的路径；否则返回原路径`res` 用于记录是否发生了实际替换

举个例子：  
`pathname = "/data/data/com.test/files/a.txt"`  
如果这个 app 被 VA 管控了，那路径会被转向  
`redirect_path = "/data/user/0/com.va.wrapper/files/a.txt"`

从这里就对于开头的那个图片进行了总体的大致讲解，但是其中虚拟机下运行应用的很多问题以及配置环境更多的是需要在实践运用上去体现。

**参考资料**：  
https://github.com/asLody/VirtualApp  
https://juejin.cn/post/7028124957141893150#heading-11  
https://blog.csdn.net/weixin_33816300/article/details/88772462  
https://blog.csdn.net/ganyao939543405/article/details/76146760  
**文章 PDF**：  
通过网盘分享的文件：Android 虚拟机原理总. pdf  
链接: https://pan.baidu.com/s/1lbpLG_SbsF21yzmJSJ6HCg 提取码: chen  
-- 来自百度网盘超级会员 v4 的分享![](https://avatar.52pojie.cn/data/avatar/001/97/85/65_avatar_middle.jpg)

> [q265354 发表于 2025-4-21 11:11](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52903148&ptid=2025427)  
> PDF 好像不是全的，到了 native hook 那里就没了？

对的，我重新上一个 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) chenchenchen777 结论  
本文通过分析 VirtualApp 的开源实现，揭示了 Android 虚拟机如何通过 Java/Native 双层级 Hook 和 路径重定向 实现应用沙箱化。其核心价值在于：    
多开与隔离：支持同一应用多实例并行运行。    
隐私保护：可伪造敏感数据（如设备信息、权限状态）。    
开发灵活性：为安全研究、测试提供可控的虚拟环境。  
参考资料：    
VirtualApp GitHub    
详细原理 PDF（提取码：chen）  
适用场景：移动安全研究、应用多开、隐私测试工具开发。 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 虽然看不懂，但是还是觉得楼主很厉害啊 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xintian 如果 hook 了这些参数，应该就能实现一些原本不支持模拟器的 app 在模拟器上运行了对吗 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) radar12345 底层知识结构，学习学习，感谢分享！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)oddant 收藏下 说不定会用到 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) ethanlaux2000 nice(&#65377;&#8226;&#768;&#7447;-)，感觉不错，有时间再来看看，先水一下，以防万一号没了。能学到知识的帖子。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Tomlls 感谢分享，一起学习一起进步 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fdjdf PDF 好像不是全的，到了 native hook 那里就没了？