> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [paul.pub](https://paul.pub/android-binder-java/)

本文是 Android Binder 机制解析的第三篇，也是最后一篇文章。本文会讲解 Binder Framework Java 部分的逻辑。

*   [主要结构](#id-主要结构)
*   [JNI 的衔接](#id-jni的衔接)
*   [Java Binder 服务举例](#id-java-binder服务举例)
*   [Java 层的 ServiceManager](#id-java层的servicemanager)
*   [关于 AIDL](#id-关于aidl)
*   [参考资料和推荐读物](#id-参考资料和推荐读物)

本系列的文章列表如下：

*   [理解 Android Binder 机制 (1/3)：驱动篇](https://paul.pub/android-binder-driver/)
*   [理解 Android Binder 机制 (2/3)：C++ 层](https://paul.pub/android-binder-cpp/)
*   [理解 Android Binder 机制 (3/3)：Java 层](https://paul.pub/android-binder-java/)

下文所讲内容的相关源码，在 AOSP 源码树中的路径如下：

```
// Binder Framework JNI
/frameworks/base/core/jni/android_util_Binder.h
/frameworks/base/core/jni/android_util_Binder.cpp
/frameworks/base/core/jni/android_os_Parcel.h
/frameworks/base/core/jni/android_os_Parcel.cpp

// Binder Framework Java接口
/frameworks/base/core/java/android/os/Binder.java
/frameworks/base/core/java/android/os/IBinder.java
/frameworks/base/core/java/android/os/IInterface.java
/frameworks/base/core/java/android/os/Parcel.java


```

Android 应用程序使用 Java 语言开发，Binder 框架自然也少不了在 Java 层提供接口。

前文中我们看到，Binder 机制在 C++ 层已经有了完整的实现。因此 Java 层完全不用重复实现，而是通过 JNI 衔接了 C++ 层以复用其实现。

下图描述了 Binder Framework Java 层到 C++ 层的衔接关系。

![](https://qiangbo-workspace.oss-cn-shanghai.aliyuncs.com/AndroidNewFeatureBook/Chapter1/Binder_JNI.png)

这里对图中 Java 层和 JNI 层的几个类做一下说明 ( 关于 C++ 层的讲解请看[这里](https://paul.pub/AndroidAnatomy_Binder_CPP/) )：

<table><thead><tr><th>名称</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>IInterface</td><td>interface</td><td>供 Java 层 Binder 服务接口继承的接口</td></tr><tr><td>IBinder</td><td>interface</td><td>Java 层的 IBinder 类，提供了 transact 方法来调用远程服务</td></tr><tr><td>Binder</td><td>class</td><td>实现了 IBinder 接口，封装了 JNI 的实现。Java 层 Binder 服务的基类</td></tr><tr><td>BinderProxy</td><td>class</td><td>实现了 IBinder 接口，封装了 JNI 的实现。提供 transact 方法调用远程服务</td></tr><tr><td>JavaBBinderHolder</td><td>class</td><td>内部存储了 JavaBBinder</td></tr><tr><td>JavaBBinder</td><td>class</td><td>将 C++ 端的 onTransact 调用传递到 Java 端</td></tr><tr><td>Parcel</td><td>class</td><td>Java 层的数据包装器，见 C++ 层的 Parcel 类分析</td></tr></tbody></table>

这里的 IInterface，IBinder 和 C++ 层的两个类是同名的。这个同名并不是巧合：它们不仅仅同名，它们所起的作用，以及其中包含的接口都是几乎一样的，区别仅仅在于一个是 C++ 层，一个是 Java 层而已。

除了 IInterface，IBinder 之外，这里 Binder 与 BinderProxy 类也是与 C++ 的类对应的，下面列出了 Java 层和 C++ 层类的对应关系：

<table><thead><tr><th>C++</th><th>Java 层</th></tr></thead><tbody><tr><td>IInterface</td><td>IInterface</td></tr><tr><td>IBinder</td><td>IBinder</td></tr><tr><td>BBinder</td><td>Binder</td></tr><tr><td>BpProxy</td><td>BinderProxy</td></tr><tr><td>Parcel</td><td>Parcel</td></tr></tbody></table>

JNI 全称是 Java Native Interface，这个是由 Java 虚拟机提供的机制。这个机制使得 native 代码可以和 Java 代码互相通讯。简单来说就是：我们可以在 C/C++ 端调用 Java 代码，也可以在 Java 端调用 C/C++ 代码。

关于 JNI 的详细说明，可以参见 Oracle 的官方文档：[Java Native Interface](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/) ，这里不多说明。

实际上，在 Android 中很多的服务或者机制都是在 C/C++ 层实现的，想要将这些实现复用到 Java 层，就必须通过 JNI 进行衔接。AOSP 源码中，/frameworks/base/core/jni/ 目录下的源码就是专门用来对接 Framework 层的 JNI 实现的。

看一下 Binder.java 的实现就会发现，这里面有不少的方法都是用`native`关键字修饰的，并且没有方法实现体，这些方法其实都是在 C++ 中实现的：

```
public static final native int getCallingPid();

public static final native int getCallingUid();

public static final native long clearCallingIdentity();

public static final native void restoreCallingIdentity(long token);

public static final native void setThreadStrictModePolicy(int policyMask);

public static final native int getThreadStrictModePolicy();

public static final native void flushPendingCommands();

public static final native void joinThreadPool();


```

在 android_util_Binder.cpp 文件中的下面这段代码，设定了 Java 方法与 C++ 方法的对应关系：

```
static const JNINativeMethod gBinderMethods[] = {
    { "getCallingPid", "()I", (void*)android_os_Binder_getCallingPid },
    { "getCallingUid", "()I", (void*)android_os_Binder_getCallingUid },
    { "clearCallingIdentity", "()J", (void*)android_os_Binder_clearCallingIdentity },
    { "restoreCallingIdentity", "(J)V", (void*)android_os_Binder_restoreCallingIdentity },
    { "setThreadStrictModePolicy", "(I)V", (void*)android_os_Binder_setThreadStrictModePolicy },
    { "getThreadStrictModePolicy", "()I", (void*)android_os_Binder_getThreadStrictModePolicy },
    { "flushPendingCommands", "()V", (void*)android_os_Binder_flushPendingCommands },
    { "init", "()V", (void*)android_os_Binder_init },
    { "destroy", "()V", (void*)android_os_Binder_destroy },
    { "blockUntilThreadAvailable", "()V", (void*)android_os_Binder_blockUntilThreadAvailable }
};


```

这种对应关系意味着：当 Binder.java 中的`getCallingPid`方法被调用的时候，真正的实现其实是`android_os_Binder_getCallingPid`，当`getCallingUid`方法被调用的时候，真正的实现其实是`android_os_Binder_getCallingUid`，其他类同。

然后我们再看一下`android_os_Binder_getCallingPid`方法的实现就会发现，这里其实就是对接到了 libbinder 中了：

```
static jint android_os_Binder_getCallingPid(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->getCallingPid();
}


```

这里看到了 Java 端的代码是如何调用的 libbinder 中的 C++ 方法的。那么，相反的方向是如何调用的呢？最关键的，libbinder 中的`BBinder::onTransact`是如何能够调用到 Java 中的`Binder::onTransact`的呢？

这段逻辑就是 android_util_Binder.cpp 中`JavaBBinder::onTransact`中处理的了。JavaBBinder 是 BBinder 子类，其类结构如下：

![](https://qiangbo-workspace.oss-cn-shanghai.aliyuncs.com/AndroidNewFeatureBook/Chapter1/JavaBBinder.png)

`JavaBBinder::onTransact`关键代码如下：

```
virtual status_t onTransact(
   uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
{
   JNIEnv* env = javavm_to_jnienv(mVM);

   IPCThreadState* thread_state = IPCThreadState::self();
   const int32_t strict_policy_before = thread_state->getStrictModePolicy();

   jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
       code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
   ...
}


```

请注意这段代码中的这一行：

```
jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
  code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);


```

这一行代码其实是在调用 mObject 上 offset 为 mExecTransact 的方法。这里的几个参数说明如下：

*   mObject 指向了 Java 端的 Binder 对象
*   gBinderOffsets.mExecTransact 指向了 Binder 类的 execTransact 方法
*   data 调用 execTransact 方法的参数
*   code, data, reply, flags 都是传递给调用方法 execTransact 的参数

而`JNIEnv.CallBooleanMethod`这个方法是由虚拟机实现的。即：虚拟机会提供 native 方法来调用一个 Java Object 上的方法（关于 Android 上的 Java 虚拟机，今后我们会专门讲解）。

这样，就在 C++ 层的`JavaBBinder::onTransact`中调用了 Java 层`Binder::execTransact`方法。而在`Binder::execTransact`方法中，又调用了自身的 onTransact 方法，由此保证整个过程串联了起来：

```
private boolean execTransact(int code, long dataObj, long replyObj,
       int flags) {
   Parcel data = Parcel.obtain(dataObj);
   Parcel reply = Parcel.obtain(replyObj);
   boolean res;
   try {
       res = onTransact(code, data, reply, flags);
   } catch (RemoteException|RuntimeException e) {
       if (LOG_RUNTIME_EXCEPTION) {
           Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
       }
       if ((flags & FLAG_ONEWAY) != 0) {
           if (e instanceof RemoteException) {
               Log.w(TAG, "Binder call failed.", e);
           } else {
               Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
           }
       } else {
           reply.setDataPosition(0);
           reply.writeException(e);
       }
       res = true;
   } catch (OutOfMemoryError e) {
       RuntimeException re = new RuntimeException("Out of memory", e);
       reply.setDataPosition(0);
       reply.writeException(re);
       res = true;
   }
   checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
   reply.recycle();
   data.recycle();

   StrictMode.clearGatheredViolations();

   return res;
}


```

和 C++ 层一样，这里我们还是通过一个具体的实例来看一下 Java 层的 Binder 服务是如何实现的。

下图是 ActivityManager 实现的类图:

![](https://qiangbo-workspace.oss-cn-shanghai.aliyuncs.com/AndroidNewFeatureBook/Chapter1/Binder_ActivityManager.png)

下面是上图中几个类的说明：

<table><thead><tr><th>类名</th><th>说明</th></tr></thead><tbody><tr><td>IActivityManager</td><td>Binder 服务的公共接口</td></tr><tr><td>ActivityManagerProxy</td><td>供客户端调用的远程接口</td></tr><tr><td>ActivityManagerNative</td><td>Binder 服务实现的基类</td></tr><tr><td>ActivityManagerService</td><td>Binder 服务的真正实现</td></tr></tbody></table>

看过 Binder C++ 层实现之后，对于这个结构应该也是很容易理解的，组织结构和 C++ 层服务的实现是一模一样的。

对于 Android 应用程序的开发者来说，我们不会直接接触到上图中的几个类，而是使用`android.app.ActivityManager`中的接口。

这里我们就来看一下，`android.app.ActivityManager`中的接口与上图的实现是什么关系。我们选取其中的一个方法来看一下：

```
public void getMemoryInfo(MemoryInfo outInfo) {
   try {
       ActivityManagerNative.getDefault().getMemoryInfo(outInfo);
   } catch (RemoteException e) {
       throw e.rethrowFromSystemServer();
   }
}


```

这个方法的实现调用了`ActivityManagerNative.getDefault()`中的方法，因此我们在来看一下`ActivityManagerNative.getDefault()`返回到到底是什么。

```
static public IActivityManager getDefault() {
   return gDefault.get();
}

private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
   protected IActivityManager create() {
       IBinder b = ServiceManager.getService("activity");
       if (false) {
           Log.v("ActivityManager", "default service binder = " + b);
       }
       IActivityManager am = asInterface(b);
       if (false) {
           Log.v("ActivityManager", "default service = " + am);
       }
       return am;
   }
};


```

这段代码中我们看到，这里其实是先通过`IBinder b = ServiceManager.getService("activity");` 获取 ActivityManager 的 Binder 对象（“activity” 是 ActivityManagerService 的 Binder 服务标识），接着我们再来看一下`asInterface(b)`的实现：

```
static public IActivityManager asInterface(IBinder obj) {
   if (obj == null) {
       return null;
   }
   IActivityManager in =
       (IActivityManager)obj.queryLocalInterface(descriptor);
   if (in != null) {
       return in;
   }

   return new ActivityManagerProxy(obj);
}


```

这里应该是比较明白了：首先通过`queryLocalInterface`确定有没有本地 Binder，如果有的话直接返回，否则创建一个`ActivityManagerProxy`对象。很显然，假设在 ActivityManagerService 所在的进程调用这个方法，那么`queryLocalInterface`将直接返回本地 Binder，而假设在其他进程中调用，这个方法将返回空，由此导致其他调用获取到的对象其实就是`ActivityManagerProxy`。而在拿到`ActivityManagerProxy`对象之后在调用其方法所走的路线我想读者应该也能明白了：那就是通过 Binder 驱动跨进程调用 ActivityManagerService 中的方法。

这里的`asInterface`方法的实现会让我们觉得似曾相识。是的，因为这里的实现方式和 C++ 层的实现是一样的模式。

源码路径：

```
frameworks/base/core/java/android/os/IServiceManager.java
frameworks/base/core/java/android/os/ServiceManager.java
frameworks/base/core/java/android/os/ServiceManagerNative.java
frameworks/base/core/java/com/android/internal/os/BinderInternal.java
frameworks/base/core/jni/android_util_Binder.cpp


```

有 Java 端的 Binder 服务，自然也少不了 Java 端的 ServiceManager。我们先看一下 Java 端的 ServiceManager 的结构：

![](https://qiangbo-workspace.oss-cn-shanghai.aliyuncs.com/AndroidNewFeatureBook/Chapter1/ServiceManager_Java.png)

通过这个类图我们看到，Java 层的 ServiceManager 和 C++ 层的接口是一样的。

然后我们再选取`addService`方法看一下实现：

```
public static void addService(String name, IBinder service, boolean allowIsolated) {
   try {
       getIServiceManager().addService(name, service, allowIsolated);
   } catch (RemoteException e) {
       Log.e(TAG, "error in addService", e);
   }
}

   private static IServiceManager getIServiceManager() {
   if (sServiceManager != null) {
       return sServiceManager;
   }

   // Find the service manager
   sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
   return sServiceManager;
}


```

很显然，这段代码中，最关键就是下面这个调用：

```
ServiceManagerNative.asInterface(BinderInternal.getContextObject());


```

然后我们需要再看一下 BinderInternal.getContextObject() 和 ServiceManagerNative.asInterface 两个方法。

BinderInternal.getContextObject() 是一个 JNI 方法，其实现代码在 android_util_Binder.cpp 中：

```
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}


```

而 ServiceManagerNative.asInterface 的实现和其他的 Binder 服务是一样的套路：

```
static public IServiceManager asInterface(IBinder obj)
{
   if (obj == null) {
       return null;
   }
   IServiceManager in =
       (IServiceManager)obj.queryLocalInterface(descriptor);
   if (in != null) {
       return in;
   }

   return new ServiceManagerProxy(obj);
}


```

先通过`queryLocalInterface`查看能不能获得本地 Binder，如果无法获取，则创建并返回 ServiceManagerProxy 对象。

而 ServiceManagerProxy 自然也是和其他 Binder Proxy 一样的实现套路：

```
public void addService(String name, IBinder service, boolean allowIsolated)
       throws RemoteException {
   Parcel data = Parcel.obtain();
   Parcel reply = Parcel.obtain();
   data.writeInterfaceToken(IServiceManager.descriptor);
   data.writeString(name);
   data.writeStrongBinder(service);
   data.writeInt(allowIsolated ? 1 : 0);
   mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
   reply.recycle();
   data.recycle();
}


```

有了上文的讲解，这段代码应该都是比较容易理解的了。

作为 Binder 机制的最后一个部分内容，我们来讲解一下开发者经常使用的 AIDL 机制是怎么回事。

AIDL 全称是 Android Interface Definition Language，它是 Android SDK 提供的一种机制。借助这个机制，应用可以提供跨进程的服务供其他应用使用。AIDL 的详细说明可以参见官方开发文档：https://developer.android.com/guide/components/aidl.html 。

这里，我们就以官方文档上的例子看来一下 AIDL 与 Binder 框架的关系。

开发一个基于 AIDL 的 Service 需要三个步骤：

1.  定义一个. aidl 文件
2.  实现接口
3.  暴露接口给客户端使用

aidl 文件使用 Java 语言的语法来定义，每个. aidl 文件只能包含一个 interface，并且要包含 interface 的所有方法声明。

默认情况下，AIDL 支持的数据类型包括：

*   基本数据类型（即 int，long，char，boolean 等）
*   String
*   CharSequence
*   List（List 的元素类型必须是 AIDL 支持的）
*   Map（Map 中的元素必须是 AIDL 支持的）

对于 AIDL 中的接口，可以包含 0 个或多个参数，可以返回 void 或一个值。所有非基本类型的参数必须包含一个描述是数据流向的标签，可能的取值是：`in`，`out`或者`inout`。

下面是一个 aidl 文件的示例：

```
// IRemoteService.aidl
package com.example.android;

// Declare any non-default types here with import statements

/** Example service interface */
interface IRemoteService {
    /** Request the process ID of this service, to do evil things with it. */
    int getPid();

    /** Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}



```

这个文件中包含了两个接口 ：

*   getPid 一个无参的接口，返回值类型为 int
*   basicTypes，包含了几个基本类型作为参数的接口，无返回值

对于包含. aidl 文件的工程，Android IDE（以前是 Eclipse，现在是 Android Studio）在编译项目的时候，会为 aidl 文件生成对应的 Java 文件。

针对上面这个 aidl 文件生成的 java 文件中包含的结构如下图所示：

![](https://qiangbo-workspace.oss-cn-shanghai.aliyuncs.com/AndroidNewFeatureBook/Chapter1/aidl_java.png)

在这个生成的 Java 文件中，包括了：

*   一个名称为 IRemoteService 的 interface，该 interface 继承自 android.os.IInterface 并且包含了我们在 aidl 文件中声明的接口方法
*   IRemoteService 中包含了一个名称为 Stub 的静态内部类，这个类是一个抽象类，它继承自 android.os.Binder 并且实现了 IRemoteService 接口。这个类中包含了一个`onTransact`方法
*   Stub 内部又包含了一个名称为 Proxy 的静态内部类，Proxy 类同样实现了 IRemoteService 接口

仔细看一下 Stub 类和 Proxy 两个中包含的方法，是不是觉得很熟悉？是的，这里和前面介绍的服务实现是一样的模式。这里我们列一下各层类的对应关系：

<table><thead><tr><th>C++</th><th>Java 层</th><th>AIDL</th></tr></thead><tbody><tr><td>BpXXX</td><td>XXXProxy</td><td>IXXX.Stub.Proxy</td></tr><tr><td>BnXXX</td><td>XXXNative</td><td>IXXX.Stub</td></tr></tbody></table>

为了整个结构的完整性，最后我们还是来看一下生成的 Stub 和 Proxy 类中的实现逻辑。

Stub 是提供给开发者实现业务的父类，而 Proxy 的实现了对外提供的接口。Stub 和 Proxy 两个类都有一个`asBinder`的方法。

Stub 类中的 asBinder 实现就是返回自身对象：

```
@Override
public android.os.IBinder asBinder() {
	return this;
}


```

而 Proxy 中 asBinder 的实现是返回构造函数中获取的 mRemote 对象，相关代码如下：

```
private android.os.IBinder mRemote;

Proxy(android.os.IBinder remote) {
	mRemote = remote;
}

@Override
public android.os.IBinder asBinder() {
	return mRemote;
}


```

而这里的 mRemote 对象其实就是远程服务在当前进程的标识。

上文我们说了，Stub 类是用来提供给开发者实现业务逻辑的父类，开发者者继承自 Stub 然后完成自己的业务逻辑实现，例如这样：

```
private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {
   public int getPid(){
       return Process.myPid();
   }
   public void basicTypes(int anInt, long aLong, boolean aBoolean,
       float aFloat, double aDouble, String aString) {
       // Does something
   }
};


```

而这个 Proxy 类，就是用来给调用者使用的对外接口。我们可以看一下 Proxy 中的接口到底是如何实现的：

Proxy 中`getPid`方法实现如下所示：

```
@Override
public int getPid() throws android.os.RemoteException {
	android.os.Parcel _data = android.os.Parcel.obtain();
	android.os.Parcel _reply = android.os.Parcel.obtain();
	int _result;
	try {
		_data.writeInterfaceToken(DESCRIPTOR);
		mRemote.transact(Stub.TRANSACTION_getPid, _data, _reply, 0);
		_reply.readException();
		_result = _reply.readInt();
	} finally {
		_reply.recycle();
		_data.recycle();
	}
	return _result;
}


```

这里就是通过 Parcel 对象以及 transact 调用对应远程服务的接口。而在 Stub 类中，生成的 onTransact 方法对应的处理了这里的请求：

```
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
		throws android.os.RemoteException {
	switch (code) {
	case INTERFACE_TRANSACTION: {
		reply.writeString(DESCRIPTOR);
		return true;
	}
	case TRANSACTION_getPid: {
		data.enforceInterface(DESCRIPTOR);
		int _result = this.getPid();
		reply.writeNoException();
		reply.writeInt(_result);
		return true;
	}
	case TRANSACTION_basicTypes: {
		data.enforceInterface(DESCRIPTOR);
		int _arg0;
		_arg0 = data.readInt();
		long _arg1;
		_arg1 = data.readLong();
		boolean _arg2;
		_arg2 = (0 != data.readInt());
		float _arg3;
		_arg3 = data.readFloat();
		double _arg4;
		_arg4 = data.readDouble();
		java.lang.String _arg5;
		_arg5 = data.readString();
		this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
		reply.writeNoException();
		return true;
	}
	}
	return super.onTransact(code, data, reply, flags);
}


```

`onTransact`所要做的就是：

1.  根据 code 区分请求的是哪个接口
2.  通过 data 来获取请求的参数
3.  调用由子类实现的抽象方法

有了前文的讲解，对于这部分内容应当不难理解了。

到这里，我们终于讲解完 Binder 了。

恭喜你，已经掌握了 Android 系统最复杂的模块，的其中之一了 ：）

– 以上 –

*   [Android Binder](https://www.nds.rub.de/media/attachments/files/2012/03/binder.pdf)
*   [Android Interface Definition Language](https://developer.android.com/guide/components/aidl.html)
*   [Android Bander 设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)
*   [Binder 系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)
*   [彻底理解 Android Binder 通信架构](http://gityuan.com/2016/09/04/binder-start-service/)
*   [binder 驱动——- 之内存映射篇](http://blog.csdn.net/xiaojsj111/article/details/31422175)
*   [Android Binder 机制 (一) Binder 的设计和框架](http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/)
*   [Android Binder 分析——内存管理](http://light3moon.com/2015/01/28/Android%20Binder%20%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)