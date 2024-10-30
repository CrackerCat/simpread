> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277637.htm)

> [原创]聊聊大厂设备指纹其三 & 如何在风控对抗这场 “猫鼠游戏” 中转换角色

. 聊聊大厂设备指纹其三 & 如何在风控对抗这场 “猫鼠游戏” 中转换角色！
======================================

目录

*   [. 聊聊大厂设备指纹其三 & 如何在风控对抗这场 “猫鼠游戏” 中转换角色！](#.聊聊大厂设备指纹其三&如何在风控对抗这场“猫鼠游戏”中转换角色！)
*            [前奏知识：](#前奏知识：)
*                    [IPC 代理 & IPC 协议是什么？](#ipc代理&ipc协议是什么？)
*                            [题外话：](#题外话：)
*                    [动态代理 IPC：](#动态代理ipc：)
*            [设备指纹：](#设备指纹：)
*                    [IPCAndroid_Id](#ipcandroid_id)
*                    [IPCAppSign](#ipcappsign)
*                    [Maps 解析 Apk 签名：](#maps解析apk签名：)
*                    [其他字段 IPC](#其他字段ipc)
*                    [IPC 总结 & 反思：](#ipc总结&反思：)
*                    [服务端级别设备指纹：](#服务端级别设备指纹：)
*            [Hunter 检测 & 反制：](#hunter检测&反制：)
*                    MapIo 重定向 Anti:
*                    [“反调试” 进程检测实现细节：](#“反调试”进程检测实现细节：)
*                    自实现 RegisterNativeMethod:
*            [风控 “猫鼠游戏” 规则解密：](#风控“猫鼠游戏”规则解密：)
*            [无 Root 情况客户端对抗的边界值在哪里？](#无root情况客户端对抗的边界值在哪里？)
*            [“边界值” 拦截技术实现：](#“边界值”拦截技术实现：)
*                    [IPC 协议拦截：](#ipc协议拦截：)
*                    [SVC 拦截：](#svc拦截：)
*                    [文件读取：](#文件读取：)
*            [总结：](#总结：)

 

这篇文章是大厂设备指纹其三，可能是最终一篇，也可能不是（主要看后续有没有更好的代替）

 

看这篇文章之前需要准备一下前奏知识，之前文章叙述过的，这里面不就一一叙述了。

 

[风控对抗基础随笔](https://bbs.kanxue.com/thread-273838.htm)

 

设备指纹第一篇：

 

[聊聊大厂设备指纹获取和对抗 & 设备指纹看着一篇就够了!](https://bbs.kanxue.com/thread-273759.htm)

 

设备指纹第二篇：

 

[聊聊大厂设备指纹其二 & Hunter 环境检测思路详解!](https://bbs.kanxue.com/thread-277402.htm)

 

这篇文章会详细叙述客户端风控对抗的 “边界值” 在哪里，如果你是在做风控对抗 ，不管你是这场游戏中在演 “猫” 的角色还是 “老鼠” 的角色 。

 

这篇文章将站在上帝视角去讲解对应的 “规则” 和 “玩法”，以及如何实现角色转换。

 

通过之前的文章，配合这篇文章希望每个小白玩家都能知道大厂是怎么玩的，如何设置游戏规则，我们应该如何进行解谜。

[](#前奏知识：)前奏知识：
---------------

### IPC 代理 & IPC 协议是什么？

在第一篇文章中介绍了一个细节点，就是 IPC 代理 ，但是现在在讲这篇文章的时候，需要更详细的叙述一下 。

 

第一篇文章里面介绍了 Android 是基础的 CS 架构，客户端和服务端架构 。安卓为什么要这么设计呢？当时问了 GTP，他给出的回答是稳定性。如果服务端和客户端在一个进程内，客户端崩溃了，服务端也会一起崩溃，导致整个系统不稳定 。

 

安卓和 Java 相比多个一个 Context，这个 Context 是调用安卓本身提供 api 的桥梁 ，里面有各种安卓系统提供的各种基础 API ，

 

这些 API 可以直接操作 Android 系统 ，安卓本身通过各种各样的 Manager 去提供对应的 Api 去获取和修改 。比如 PackageManager，ActivityManager 等，这些 Manager 里面都会持有一个代理人 。当我们去调用这个 Manager 里面的一些 Api 的时候，一些简单的 Api 他会尝试去自己在本进程 Native 或者 Java 去实现，如果一些复杂的字段，比如查询系统的一些信息，或者调用一些系统关键函数，这种时候他会去调用 “IPC 代理人”，这个 IPC 代理人就是像服务端通讯的关键 。他相当于是向服务端的传话得人 ，代理设计模式 。对不同的 Manager 提供不一样的功能 ，而他传的话就是对应的 IPC 协议 。这个协议如何传递的，就是通过底层的共享内存 Binder 去实现的 。

 

而这个协议里面具体发送的内容，就是 IPC 协议装的 “包裹” 就是用的 Parcel 。

 

这块举个栗子，当用户调用一个未初始化的 API 时候，需要跨进程通讯，到底发生了哪些动作。

 

用户调用系统 API->Manager 收到调用消息 -> 判断是否需要调用服务端 -> 调用 IPC 代理里面的方法 ->

 

IPC 代理构建发送的数据包调用 Binder 进行通讯数据写入以后返回。

 

这个 IPC 代理实现了 Binder 的接口，当前进程调用的最后一个 API 就是

```
"android.os.BinderProxy"->transact
```

也就是说这个方法底层调用的是 Binder 的驱动，最终会去 native 层写入，剩下的就是开始运行服务端的逻辑了。把数据写入到 transact 方法的参数 3 里面。然后程序返回，下面是这个方法的原型 。

```
/**
     * Perform a binder transaction on a proxy.
     *
     * @param code The action to perform.  This should
     * be a number between {@link #FIRST_CALL_TRANSACTION} and
     * {@link #LAST_CALL_TRANSACTION}.
     * @param data Marshalled data to send to the target.  Must not be null.
     * If you are not sending any data, you must create an empty Parcel
     * that is given here.
     * @param reply Marshalled data to be received from the target.  May be
     * null if you are not interested in the return value.
     * @param flags Additional operation flags.  Either 0 for a normal
     * RPC, or {@link #FLAG_ONEWAY} for a one-way RPC.
     *
     * @return
     * @throws RemoteException
     */
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            ...
    }
```

第一个参数就是 code，这个 code 指的是具体的事件类型，不同的事件传入的数字也不一样 。

 

第二参数是发送的数据包，这时候 IPC 已经往里面进行了写入对应的数据包。

 

第三个参数是 reply，也就是服务端返回的保存内容，

 

注意：

> <u> **这块可能存在一个问题，有的数据，这个时候你在 after 去复写这个参数 3 已结晚了，因为数据在别的进程已结写入了 。**</u>  
> <u> **如果你想对这个参数进行修改，最好的办法是直接模拟服务端手动往 reply 进行写入** </u>

 

第四个参数是 flags，举个例子，比如你想获取正常的 PackageInfo，不需要获取签名之类的信息，你传入 0 即可 。

 

如果想获取签名就需要传入 PackageManager.GET_SIGNATURES，这个 flag 相当于告诉服务端，都需要哪些功能 。

```
getPackageManager().getPackageInfo("aaa",0);
getPackageManager().getPackageInfo("aaa",PackageManager.GET_SIGNATURES);
```

比如第一个 Api 获取到的 PackageInfo 里面是不包含签名信息的，第二个则包含 。当你需要什么功能的时候使用 “或” 连接即可。

 

基础知识介绍完毕：

> 这块我们得到一个结论，这个方法是 Java 层通讯最后一个方法，也就是当前进程能操作的最后一个方法，剩下的就是服务端进程的事情了。这个方法是 Java 层的 “边界值”，这个边界值记住后面在总结里面会介绍到不同的边界值和风控的关系。

#### [](#题外话：)题外话：

这块还有的大厂更恶心，他不走 transact 方法，因为 transact 方法底层走的就是 Binder，可以直接在 Native 层调用的 Binder 驱动，实现了 transact 这个方法 。然后进行 IPC 通讯，直接不走 Java 层 。

 

当然他这种方法也是很不稳定，需要对每个 android 版本都进行兼容，属于伤敌 1000 自损 800 类型 ，适配难度也很大。

 

随着安卓不断增强安全性，后面这种方式肯定会慢慢被 PASS 掉 ，现在利用跨进程在低版本越权 App 的太多了 。

### [](#动态代理ipc：)动态代理 IPC：

这块还有一个知识点：

 

就是我不用 hook 可以实现 IPC 的代理人替换么？

> 这块有一个动态代理的知识点，就是他代理人本身是实现了一个接口，我们可以直接反射把他这个代理人给替换成我们的，然后我们使用 Proxy.newProxyInstance 动态代理这个接口类，也可以实现不需要 Hook 框架的情况下实现动态代理 。比如一些 VA 之类的用的就是这种，因为 Hook 其实稳定性啥的没有动态代理的稳定性好，Hook 的话需要对不同版本兼容，一旦版本发生变化需要适配很多东西，而动态代理则不需要。
> 
> Hook 的话痕迹可能更少一点，动态代理检测的话只需要反射这个 IPC 代理人，然后 getClass().getName() 里面直接就有 proxy 之类的关键字 ，各有各的好处 。
> 
> 自己决定使用那种方式 。

[](#设备指纹：)设备指纹：
---------------

### IPCAndroid_Id

在之前第二篇设备指纹里面介绍了获取 Android Id 的五种方式，第五种方式因为当时没时间也没对高版本兼容，所以一直没发 ，这块抽空对照不同 Android 完善一下 。

 

直接构建 IPC 协议和服务端进行通讯 ，这块 targetSdkVersion 必须升级到 32 以上，因为 getAttributionSource 这个玩意 32 版本以上好像才有。

 

为了兼容 Android 高版本需要升级 。不过还好是 Java 方法，就算出异常也可以 try 住 。

 

不过这种方式在高版本不一定能用了，低版本可以，因为我看 API 提示,

 

Reflective access to CALL_TRANSACTION will throw an exception when targeting API 33 and above

> 当针对 API 33 及以上目标时，对 CALL_TRANSACTION 的反射性访问将抛出异常

```
public String getAndroidId5(Context context) {
        try {
            // Acquire the ContentProvider
            Class activityThreadClass = Class.forName("android.app.ActivityThread");
            Method currentActivityThreadMethod = activityThreadClass.getMethod("currentActivityThread");
            Object currentActivityThread = currentActivityThreadMethod.invoke(null);
            Method acquireProviderMethod = activityThreadClass.getMethod("acquireProvider", Context.class, String.class, int.class, boolean.class);
            Object provider = acquireProviderMethod.invoke(currentActivityThread, context, "settings", 0, true);
 
            // Get the Binder
            Class iContentProviderClass = Class.forName("android.content.IContentProvider");
            Field mRemoteField = provider.getClass().getDeclaredField("mRemote");
            mRemoteField.setAccessible(true);
            IBinder binder = (IBinder) mRemoteField.get(provider);
 
            // Create the Parcel for the arguments
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken("android.content.IContentProvider");
            if (android.os.Build.VERSION.SDK_INT
                    >= android.os.Build.VERSION_CODES.S) {
                context.getAttributionSource().writeToParcel(data, 0);
                data.writeString("settings"); //authority
                data.writeString("GET_secure"); //method
                data.writeString("android_id"); //stringArg
                data.writeBundle(Bundle.EMPTY);
            } else if (android.os.Build.VERSION.SDK_INT
                    == android.os.Build.VERSION_CODES.R) {
                //android 11
                data.writeString(context.getPackageName());
                data.writeString(null); //featureId
 
                data.writeString("settings"); //authority
                data.writeString("GET_secure"); //method
                data.writeString("android_id"); //stringArg
                data.writeBundle(Bundle.EMPTY);
            } else if (android.os.Build.VERSION.SDK_INT
                    == android.os.Build.VERSION_CODES.Q) {
                //android 10
                data.writeString(context.getPackageName());
 
                data.writeString("settings"); //authority
                data.writeString("GET_secure"); //method
                data.writeString("android_id"); //stringArg
                data.writeBundle(Bundle.EMPTY);
            } else {
                data.writeString(context.getPackageName());
 
                data.writeString("GET_secure"); //method
                data.writeString("android_id"); //stringArg
                data.writeBundle(Bundle.EMPTY);
            }
 
            Parcel reply = Parcel.obtain();
            binder.transact((int) iContentProviderClass.getDeclaredField("CALL_TRANSACTION").get(null), data, reply, 0);
            reply.readException();
            Bundle bundle = reply.readBundle();
            reply.recycle();
            data.recycle();
 
            return bundle.getString("value");
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
```

### IPCAppSign

在第二篇帖子里面，还有一种检测签名方法遗漏，IPC 获取签名也是一些大厂经常用检测签名的办法，修改的话也很简单，直接替换掉数据包即可。

 

具体获取方法如下 。

```
public static int TRANSACTION_getPackageInfo() {
    if(TRANSACTION_getPackageInfo == -1) {
         try {
                Field field = null;
                try {
                    Class pkmIPCClazz = Class.forName("android.content.pm.IPackageManager$Stub");
                    field = pkmIPCClazz.getDeclaredField("TRANSACTION_getPackageInfo");
                } catch (Throwable e) {
                    CLog.e(">>>>>>>>>> getTranscationId forName error " + e.getMessage());
                }
                assert field != null;
                field.setAccessible(true);
                TRANSACTION_getPackageInfo = field.getInt(null);
            } catch (Throwable e) {
                e.printStackTrace();
                CLog.e(">>>>>>>>>> getTranscationId error " + e.getMessage());
            }
        }
        return TRANSACTION_getPackageInfo;
}
 
try {
    PackageManager packageManager = getBaseContext().getPackageManager();
 
    Object IPC_PM_Obj = RposedHelpers.getObjectField(packageManager, "mPM");
    //取binder
    IBinder mRemote = (IBinder) RposedHelpers.getObjectField(IPC_PM_Obj, "mRemote");
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    _data.writeInterfaceToken("android.content.pm.IPackageManager");
    _data.writeString(getPackageName());
    _data.writeLong(PackageManager.GET_SIGNATURES);
    _data.writeInt(android.os.Process.myUid());
 
    boolean _status = mRemote.transact(TransactCase.TRANSACTION_getPackageInfo(), _data, _reply, 0);
    _reply.readException();
    PackageInfo packageInfo = _reply.readTypedObject(PackageInfo.CREATOR);
    _data.recycle();
    _reply.recycle();
    CLog.e("签名信息: "+packageInfo.signatures[0].toCharsString());
} catch (Throwable e) {
    CLog.i("IPC_TEST_getPackageInfo error "+e);
}
```

### [](#maps解析apk签名：)Maps 解析 Apk 签名：

这块还有一个方案，主要实现思路就是因为我们当前进程去打开 apk 是存在风险的 。

 

很有可能被 IO 重定向，导致得到的签名是错误的，所以我们可以让三方进程去加载当前 apk 文件，通过共享内存的方式，然后当前进程对 apk 文件 maps 里面的内存签名进行解析即可 。这块需要双进程通讯 。

### 其他字段 IPC

根据上面的两个经典 IPC 例子，可以发现只要是服务端获取的都可以使用 IPC 协议的方式去获取 。

 

其他字段其实一样也可以这么玩 ，如果需要什么字段，就对照安卓源码客户端往里面写入对应的数据，直接 IPC 即可 。

 

我一般分析的 SO 文件的时候直接对 jni 交互进行监听，配合以前自己写的一套 [jnitrace](https://github.com/w296488320/JnitraceForCpp)，在保存的调用栈里面，看他如果调用了 Parcel.obtain() 初始化或者 这种 writeLong () 写入数据的方法，基本就可以确认他是 IPC 获取的一些字段，具体看他写入的内容是什么，或者看他写入的 token 是什么，比如上面的获取签名的 token 就是 "android.content.pm.IPackageManager" ，即可知道他想做什么字段的获取。

### IPC 总结 & 反思：

后来我又思考了一下，IPC 服务端这些设备指纹，或者说这些配置到底哪里来的，一直在源码里面跟。

 

发现就拿 android id 来说，他最终读取的文件路径是 / data/system/users/0/settings_ssaid.xml ，这个目录下，/data/system/users/0 / 我发现这里面全是各种注册表和各种配置信息 。，我这边尝试改了一下里面的 android id 。然后直接手机重启 ，我发现我之前自己写的 Hunter 获取的设备指纹 android id 竟然变了 。

 

后来我把这些文件都拷贝出来，把里面熟悉的值都随机了一份，通过 magisk 插件系统文件替换的方式，对文件 / data/system/users/0 / 进行替换 ，真没想到以前被封的设备解封了。而且不需要回复出厂设置，只需要软重启一下就行 。

 

**而且基本可以做到无痕 ，因为没有对 apk 任何修改，改的全是系统级别的变量，而我只需要 Root ，替换系统文件，相当于每一次都是恢复出厂设置 。**

 

现在基本大厂想要在回复出厂设置保持设备指纹不变基本不可能 。这套方案我测试过一段时间，现阶段基本大厂从客户端角度基本没办法对抗 。只能靠一些服务端指纹去做检测 。（我课程里面会更详细的去讲这套方案的落地实现 ，包括途中踩得一些坑。）

 

这套方案也是我 2 年前对抗风控用的主要方案 ，现在分享一下 。我给攻击方添把柴 。加速行业内卷我辈刻不容缓！

 

我猜测这种方案会成为未来的主流风控对抗方案 。

> 这块还有个细节，如果我实现了系统文件替换，应该如何多开呢？ 其实非常简单，我直接利用系统级别的多开 ，这时候肯定会有人过来问 。
> 
> 有的 Apk 系统不让多开怎么办呀 ？
> 
> 你忘记你会 Hook 了么，他判断条件本身就是在系统桌面和 setting 里面 ，直接 Hook，让每个 Apk 都可以实现多开不就完事了 。
> 
> 就比如核心破解啥的，可以 Hook 系统的签名解析进行 anti，实现不签名直接安装，他 Apk 层拿什么去检测 签名被修改了？这种方案 anti 掉各种企业壳，亲测 。

 

当然文章继续往下看 ，会给在没有 Root 的情况下如何进行对抗呢？

### [](#服务端级别设备指纹：)服务端级别设备指纹：

还是针对上面的场景，如果我是防御方应该怎么对抗呢？我之前一直思考这个问题，如果客户端拿到的数据都是不安全的，那么有没有一种方案可以服务端获取设备指纹呢？

 

其实也是有的，其实我之前的文章讲过一点 。但是没细化，这块我会详细介绍一下架构和设计模式 。

 

就是服务端去获取客户端的 IPV6 信息，配合客户端上报 。IPV6 号称能给世界上每粒沙子分配一个 ip , 2 的 128 次方 。

 

可以在设备指纹初始化的时候调用一下接口，服务端网关层去获取 IpV6 信息 。将信息保存作为客户端设备指纹。

 

tls 最新版 + socket 进行通讯 ，用这个 ipv6 作为设备指纹 。当然这块也需要防代理，具体的方案也很多，比如一些大厂会去购买一些

 

代理 IP，试用的时候去请求自己的网关，然后把这些 IP 都拉黑 。当然 ，对抗方式也有很多 ，这里不一一叙述了。 下面我会更详细的叙述一下对抗场景 。

Hunter 检测 & 反制：
---------------

这块主要是介绍一些比较新奇的对抗和检测 ，也是我之前在做黑产对抗的时候发现的一些办法。

### MapIo 重定向 Anti:

一般在实现一机多号的时候，因为需要对不同的账号进行 IO 重定向，把不同的账号，保存到自己的虚拟分身里面 。

 

这时候如果你要读取 Maps 去遍历 Item 的时候就会发现这个 Item 异常，一般沙箱开发者会将 MapsIo 重定向 。当发现读取 Maps 的时候把

 

指向自己的文件，因为这个 Maps 是不断变化的，所以需要在 svc openat 这块进行拦截生成 一份新的。然后指向到这份新的文件，在新的 maps 里面他会对里面的 item 路径进行反转，转换成正常的目录，而不是包含沙箱的目录 。导致获取的数据被欺骗 。

 

这块读文件偏移完全可以不读取 Maps ，而是读取 proc/self/maps_files 对这个文件进行 opendir ，对每个文件进行遍历，然后再路径拼接，通过 readlinkat 去反查路径 即可 。

### [](#“反调试”进程检测实现细节：)“反调试” 进程检测实现细节：

一般 Apk 都会开启一条线程作为检测反调试线程，这条线每隔几秒对线程进行一些特征进行检查当前进程是否被调试 。

 

有很多攻击者会 Hook 线程创建的办法，然后在线程启动的时候进行 pass，不让其启动 ，以实现逃过检测的办法 。

 

这种情况其实对抗也很简单，可以在主线程搞个 flag，只有在调试线程开启的时候，并且检测执行成功的时候使用 process_vm_writev 对 flag 进行写入 。

 

因为是异步，所以主线程可以延迟 2 秒钟对这个 flag 进行检测，判断调试线程是否开启，如果没开启上报埋点即可 。

### 自实现 RegisterNativeMethod:

我们正常注册一个 native 方法是调用的 env->RegisterNatives ，但是这种直接 api 调用很有可能被 Hook 。

 

所以我们可以自己实现一份 ，因为 native 注册底层本质上是给 artmethod 里面的 fnptr 进行赋值，最终调用 artmethod 里面的 RegisterNative 方法，所以我们可以不直接调用 Jni 直接走 artmethod 里面的注册方法。具体实现如下，因为 artmethod 里面的注册方法每个版本的实现都不一样 ，所以这块需要根据不同版本进行 case 分发 。

```
//call art method register
if (!RegisterNativeMethod(env, NativeEngine,
                          SignatureFixMethods,
                          sizeof(SignatureFixMethods) / sizeof(SignatureFixMethods[0]))) {
    LOG(ERROR) << "JNI_OnLoad call art method register fail ,start env register natives! ";
 
    env->RegisterNatives(NativeEngine,
                         SignatureFixMethods,
                         sizeof(SignatureFixMethods) / sizeof(SignatureFixMethods[0]));
}
```

调用的话很简单直接尝试调用我们自己实现的方法，如果失败了则调用系统的 api ，这样可以有效防止 jni 被 hook 实现，jni RegisterNative 函数被监听 。

```
//
// Created by Zhenxi on 2022/8/22.
//
 
#include 
 
#include "../include/logging.h"
#include "../include/libpath.h"
#include "../include/dlfcn_compat.h"
#include "../include/version.h"
#include "../include/main.h"
 
static void *art_method_register = nullptr;
 
static void *class_linker_ = nullptr;
 
size_t OffsetOfJavaVm(bool has_small_irt, int SDK_INT) {
 
    if (has_small_irt) {
        switch (SDK_INT) {
            case ANDROID_T:
            case ANDROID_SL:
            case ANDROID_S:
                return sizeof(void *) == 8 ? 624 : 300;
            case ANDROID_R:
            case ANDROID_Q:
                return sizeof(void *) == 8 ? 528 : 304;
            default:
                LOGE("OffsetOfJavaVM Unexpected android version %d", SDK_INT);
                abort();
        }
    } else {
        switch (SDK_INT) {
            case ANDROID_T:
            case ANDROID_SL:
            case ANDROID_S:
                return sizeof(void *) == 8 ? 520 : 300;
            case ANDROID_R:
            case ANDROID_Q:
                return sizeof(void *) == 8 ? 496 : 288;
            default:
                LOGE("OffsetOfJavaVM Unexpected android version %d", SDK_INT);
                abort();
        }
    }
}
 
template int findOffset(void *start, size_t len, size_t step, T value) {
 
    if (nullptr == start) {
        return -1;
    }
 
    for (int i = 0; i <= len; i += step) {
        T current_value = *reinterpret_cast((size_t) start + i);
        if (value == current_value) {
            return i;
        }
    }
    return -1;
}
 
/**
* 根据runtime获取class_linker
* https://github.com/magician8520/BlackBox/blob/99f26925aa303fd0a71543e3713ef3fc57a08e81/Bcore/pine-core/src/main/cpp/android.h#L36
*/
void *getClassLinker() {
    if (class_linker_ != nullptr) {
        return class_linker_;
    }
    int SDK_INT = get_sdk_level();
    // If SmallIrtAllocator symbols can be found, then the ROM has merged commit "Initially allocate smaller local IRT"
    // This commit added a pointer member between `class_linker_` and `java_vm_`. Need to calibrate offset here.
    // https://android.googlesource.com/platform/art/+/4dcac3629ea5925e47b522073f3c49420e998911
    // https://github.com/crdroidandroid/android_art/commit/aa7999027fa830d0419c9518ab56ceb7fcf6f7f1
    bool has_smaller_irt = getSymCompat(getlibArtPath(),
                                        "_ZN3art17SmallIrtAllocator10DeallocateEPNS_8IrtEntryE") !=
                           nullptr;
    size_t jvm_offset = OffsetOfJavaVm(has_smaller_irt, SDK_INT);
    auto runtime_instance_ = *reinterpret_cast (getSymCompat(getlibArtPath(), "_ZN3art7Runtime9instance_E"));
 
    auto val = jvm_offset
               ? reinterpret_cast *>(
                       reinterpret_cast(runtime_instance_) + jvm_offset)->get()
               : nullptr;
    if (val == getVm()) {
        LOGD("JavaVM offset matches the default offset");
    } else {
        LOGW("JavaVM offset mismatches the default offset, try search the memory of Runtime");
        int offset = findOffset(runtime_instance_, 1024, 4, getVm());
        if (offset == -1) {
            LOGE("Failed to find java vm from Runtime");
            return nullptr;
        }
        jvm_offset = offset;
        LOGW("Found JavaVM in Runtime at %zu", jvm_offset);
    }
    const size_t kDifference = has_smaller_irt
                               ? sizeof(std::unique_ptr) + sizeof(void *) * 3
                               : SDK_INT == ANDROID_Q
                                 ? sizeof(void *) * 2
                                 : sizeof(std::unique_ptr) + sizeof(void *) * 2;
 
    class_linker_ = *reinterpret_cast(reinterpret_cast(runtime_instance_) +
                                               jvm_offset - kDifference);
    return class_linker_;
}
 
 
bool call_MethodRegister(JNIEnv *env, void *art_method, void *native_method) {
    if (art_method_register == nullptr) {
        if (get_sdk_level() < ANDROID_S) {
            //android 11
            art_method_register = getSymCompat(getlibArtPath(),
                                               "_ZN3art9ArtMethod14RegisterNativeEPKv");
            if (art_method_register == nullptr) {
                art_method_register = getSymCompat(getlibArtPath(),
                                                   "_ZN3art9ArtMethod14RegisterNativeEPKvb");
            }
        } else {
            //12以上还是在libart里面,但是在linker里面实现,符号名称存在变化
            art_method_register = getSymCompat(getlibArtPath(),
                                               "_ZN3art11ClassLinker14RegisterNativeEPNS_6ThreadEPNS_9ArtMethodEPKv");
        }
        if (art_method_register == nullptr) {
            LOG(ERROR) << "register native method  get art_method_register = null  ";
            return false;
        }
    }
    if (get_sdk_level() >= ANDROID_S) {
        //12以上
        //const void* RegisterNative(Thread* self, ArtMethod* method, const void* native_method)
        auto call = reinterpret_cast(art_method_register);
        //get self thread
        void *self = getSymCompat(getlibArtPath(), "_ZN3art6Thread14CurrentFromGdbEv");
        if (self == nullptr) {
            LOG(ERROR) << "register native method  get CurrentFromGdb = null  ";
            return false;
        }
        //手动计算一下linker实例地址
        void *classLinker = getClassLinker();
        if (classLinker == nullptr) {
            LOG(ERROR) << "register native method  get getClassLinker = null  ";
            return false;
        }
        call(classLinker, self, art_method, native_method);
        //LOG(ERROR) << "register native method  get getClassLinker success!  ";
    } else if (get_sdk_level() >= ANDROID_R) {
        auto call = reinterpret_cast(art_method_register);
        call(art_method, native_method);
    } else {
        auto call = reinterpret_cast(art_method_register);
        call(art_method, native_method, true);
    }
    return true;
}
 
 
inline static bool IsIndexId(jmethodID mid) {
    return ((reinterpret_cast(mid) % 2) != 0);
}
 
static jfieldID field_art_method = nullptr;
 
bool RegisterNativeMethod(JNIEnv *env,
                          jclass clazz,
                          const JNINativeMethod *methods,
                          size_t nMethods) {
    if (env == nullptr) {
        LOG(ERROR) << "register native method  JNIEnv = null  ";
        return false;
    }
    void *arm_method = nullptr;
 
    for (int i = 0; i < nMethods; i++) {
        jmethodID methodId = env->GetMethodID(clazz, methods[i].name, methods[i].signature);
        if (methodId == nullptr) {
            //maybe static
            env->ExceptionClear();
            methodId = env->GetStaticMethodID(clazz, methods[i].name, methods[i].signature);
            if (methodId == nullptr) {
                LOG(ERROR) << "register native method  get orig method  == null  "
                           << methods[i].signature;
                env->ExceptionClear();
                return false;
            }
        }
        if (get_sdk_level() >= ANDROID_R) {
            if (field_art_method == nullptr) {
                jclass pClazz = env->FindClass("java/lang/reflect/Executable");
                field_art_method = env->GetFieldID(pClazz, "artMethod", "J");
            }
            if (field_art_method == nullptr) {
                LOG(ERROR) << "register native method  get artMethod  == null  ";
                return false;
            }
            if (IsIndexId(methodId)) {
                jobject method = env->ToReflectedMethod(clazz, methodId, true);
                arm_method = reinterpret_cast(env->GetLongField(method, field_art_method));
                //LOG(ERROR) << "arm_method   "<
```

[](#风控“猫鼠游戏”规则解密：)风控 “猫鼠游戏” 规则解密：
---------------------------------

我们假设一个场景，在一个农场里面有一些粮仓。

 

这些粮仓有一些猫在守护，猫的角色任务是守护好粮仓里面的粮食，而老鼠的任务是为了填饱肚子偷粮（“抓数据”）。

 

当然，农场很大，有很多粮仓，很多只猫，有的保护的粮仓有大有小，保存的粮食也不一样（“不同数据”） 。

 

当然也会有很多很多的老鼠，老鼠的数量肯定是比猫多的。猫通过眼睛和耳朵（“情报”）去探查消息 ，不同老鼠的气味（“设备风险标签”）的方式去抓不同的老鼠 。

 

防止老鼠去通过一些特殊的手段去偷粮 ，规则是每个老鼠每天只能拿一粒米 。

 

当然有的老鼠很笨，他每一次只拿 1 粒米，猫也不会去管 。但是有的老鼠很聪明，可以通过逃逸 “气味” 的方式，

 

每次用不同身份去偷米，偷得次数多了，猫发现米变少了，猫就需要去找到这个老鼠的行动方式，或者他身上的 “气味”，去查看他是否存在作弊的情况 。

 

可以给老鼠一些假大米（“脏数据”）去定位，有很多老鼠不知道自己的大米有问题，正在吃的时候就被猫抓到了，或者对老鼠的搬运速度进行限制（“请求速度”），或者当发现某个老鼠带着包裹进来粮仓的时候都进行限制（“策略”）

 

当然老鼠也会有更多办法，每次可以不带包裹进入，换一些别的可以携带的大量米粒的方式进入。

 

这时候猫就需要去检查都有哪些可以装数据的办法（” 定制策略 “）去分析老鼠的行为，看看不同的老鼠都在都在做什么。当然每次定制的策略都不一样，老鼠也不知道，只有猫知道 ，所以老鼠一直处在明，而猫在暗 。

 

时间久了 ，猫很聪明，需要根据不同 “气味” 去定位不同的老鼠，如果发现同一个气味的老鼠进入多次，拿到的粮食过多，的时候，就下令将这只老鼠永久不得入内 。

 

那么你现在演的是老鼠？你应该如何偷到更多的大米呢？

 

你如果演的是猫，你应该如何保护自己的粮食不被偷呢？

 

**<u> 首先我们先说老鼠角色如何胜利：**</u>

> 这场战斗中其实有一点很明显，就是猫是通过不同的 “气味” 去定位这个老鼠是否存在风险 ，如果可以去掉本身的气味，让 “设备风险标签” 失效老鼠就赢了 ，这场游戏局势便从老鼠在明，猫在暗 。
> 
> 转换立场，变成猫在明，老鼠在暗。在配合多只老鼠即可实现大量粮食的盗取 。

 

**<u><u > 猫的胜利规则：**</u></u>

> 不断找出变化 “气味” 的老鼠，通过不同老鼠的气味，实现最准确的判断，影响老鼠的数据搬运，分子和分母越大，找出作弊老输越准确，覆盖率越高，猫得到的奖励越多 。

[](#无root情况客户端对抗的边界值在哪里？)无 Root 情况客户端对抗的边界值在哪里？
-----------------------------------------------

ok 经过上面的例子总结和反思，我们发现一个问题，如果在不 Root 的情况下，注入方法主要两种，重打包或者把 Apk 放到沙箱里面 。并且在不修改系统文件，那么我应该如何修改 “气味” 呢？

 

决定猫和老鼠明暗关系位置的关系本质上是 “气味”主导因素 。这个 “设备风险标签” 是这场游戏中决定胜败的主要因素 ，如果“设备风险标签” 是没问题，也配合一些多开软件，云手机等控制多只老鼠即可 。

 

这里面的标签分为很多种 。每个子项又分为很多小项，不同的标签颜色不同或者说不同价值的标签对不同猫咪的反应程度也不一样 。比如重打包这种标签，在一些高度敏感的场景，会直接被猫进行封号。

 

主要的三个核心功能组成水桶木板分别如下 。

*   设备指纹
*   环境 & 风险检测能力
*   代码防护

第一项每个大厂都不一样，根据不同的策略每个字段的比重占比也都不一样 。

 

把一些常见的或者第一篇和第二篇提到的对着改一下即可 。

 

第三项现在 So 层基本大厂都差不多，都是各种混淆配合控制流 ，但是 Java 层防护做的不够 ，java 层其实可以参考我之前 19 年搞的 Java 控制流混淆 。https://bbs.kanxue.com/thread-255514.htm ，可以直接废掉 Jadx 反编译软件 。

 

这块重点介绍一下第二项，包括一些常见的子项 ，每个子项还可以继续划分各种检测方式 。

*   环境 & 风险检测能力
    
    *   重打包检测能力
        
    *   Hook 检测能力 。
        
    *   模拟器 & 云手机 & 自定义 ROM 检测能力
    *   多开 & 沙箱检测能力
    *   风险 Apk 检测能力

上面说的这几项便是不同”气味 “的组成部分，而这个” 气味” **采集方式的边界值**又分为三部分 。

*   Java 层就是 IPC 协议 ，因为 IPC 协议是当前进程可以操作的最后一个方法 。剩下的就是服务端给喂数据了。
*   Native 层就是 SVC 拦截，因为 SVC 是 Linux 进入内核的最后一条指令。
*   还有一种是读文件 ，这块区分成两部分
    *   进程文件，也就是 / proc / 下面的
    *   系统文件，系统提供的一些文件可供读取。

好 ，根据上面的总结，**只要我们在上面的三个边界值进行拦截理论上就是最完美的方案 。**

 

很多小白基本都是遇到一个指纹，咦，发现自己没有修改 。赶紧去 Hook 修改一下。去打个补丁 ，在我看来这是一种很 Low 的办法 。绕来绕去人家采集一个字段有 N 种手段，很容易导致遗漏，特别是一些大厂基本采集一个字段都是 N 种获取方式，就比如第二篇文章里面的磁盘大小，或者 Android id 的获取五种方式 。

 

你打了一个补丁补上去，在其他地方设备指纹或者环境风险又泄漏了，最后代码写的破破烂烂 。所以在边界值修改对应的值是最完美的方案 。先把架子搭好了，后面发现什么直接在边界值处的 callback 进行修改即可 。

[](#“边界值”拦截技术实现：)“边界值” 拦截技术实现：
------------------------------

下面的架子是我自己的沙箱的设计模式 ，这块也是分享一下对应的 “架构” 。

 

如果说你想实现上面的功能按照我的架子去搭建应该是比较完美的方案 ，毕竟我已经把坑踩得差不多了。

### [](#ipc协议拦截：)IPC 协议拦截：

先说 IPC，IPC 的话很简单，我在上面也说了可以动态代理，也可以直接去用 Hook 框架 Hook binder 里面的交互方法 。当发现触发指定的 IPC 协议的时候，直接模拟服务端往里面写入即可。

 

这块还有个细节点，为了防止程序直接通过 cache 获取，因为有的字段初始化以后可能被保存到 cache 里面 ，如果不存在的话再通过 ipc 去获取。Apk 在启动一瞬间就进行了初始化，cache 会被保存 。很多 IPC 代理人会这么设计 ，所以需要清理掉 cache ，这个 cache 可以是 Parcel 的 cache 也可以是 IPC 代理人里面的 cache 。比如 Parcel 里面的 mCreators 或者 sPairedCreators 都需要清空 。如果是 IPC 代理人的话也可以看代码看具体实现，看看是否包含 cache，有的话清掉即可 。

### [](#svc拦截：)SVC 拦截：

这块不多说了，我之前的帖子里面介绍过，主要用的是 ptrace+seccomp 做的架子 。

 

https://bbs.kanxue.com/thread-273160.htm 详细文章可以看这个 。

### [](#文件读取：)文件读取：

这块分为两部分，proc 下的文件我会使用 fuse 对整个 proc 进行模拟 ，这是完美方案 。proot 代码写好现成的，迁移到 android 上直接用就好了。

 

如果是读取系统文件的话，可以直接使用 IO 重定向配合 SVC 拦截即可，SVC 都可以拦截了，任何文件读取你都可以随便修改 。

 

因为不管如何，最终都会调用到系统内核去读取文件，都会被转换成 SVC 指令 。

[](#总结：)总结：
-----------

这三篇文章也是我自己对风控对抗的一点点见解 。如果你三篇文章都读完，应该是有那么亿点收获的。

 

直接通过协议的话肯定是白费，很多心跳和埋点没办法完全模拟 。

 

如果想做自动化的方式，怎么把自己模拟的更像一个真实的 “老鼠” ，比如可以在自动化点击的记录一些人手操作的路径 ，而不是单纯地去点击 。在点击过程中添加一些随机路径 ，这些都是很不错的对抗手段 。

 

IP 啥的能用真实的电话卡肯定是最好的，还有账号权重问题 。这些也都需要去解决 。

 

不过还好，只要风险标签可以做到逃逸，其他的都是小问题 ，只要花点时间都可以解决 。

 

因为在你做到标签逃逸以后，头疼的就不是老鼠，而是猫了 。

 

这时候有人可能会问，我应该如何知道我自己的设备是否存在问题呢？

 

你可以用我的 Hunter 去检测自己的设备是否存在问题 。Hunter 会把每一个可能存在的风险项都展示出来 。对着改就可以了 。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm)