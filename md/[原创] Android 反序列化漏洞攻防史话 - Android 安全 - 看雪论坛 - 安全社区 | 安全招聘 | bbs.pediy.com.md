> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-276428.htm)

> [原创] Android 反序列化漏洞攻防史话

[原创] Android 反序列化漏洞攻防史话

43 分钟前 78

### [原创] Android 反序列化漏洞攻防史话

 [![](http://passport.kanxue.com/upload/avatar/554/844554.png?1584801497)](user-home-844554.htm) [evilpan](user-home-844554.htm) ![](https://bbs.kanxue.com/view/img/rank/11.png) 12  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 43 分钟前  78

> 本文初发表至笔者[博客](https://evilpan.com/)，为防止不可抗力，故也在看雪上发布一下备份。

 

Java 在历史上出现过许多反序列化的漏洞，但大部分出自 J2EE 的组件。即便是 FastJSON 这种漏洞，似乎也很少看到在 Android 中被实际的触发和利用。本文即为对历史上曾出现过的 Android Java 反序列化漏洞的分析和研究记录。

 

目录

*   [前言](#前言)
*   Parcel 101
*            [AIDL](#aidl)
*            [Parcelable](#parcelable)
*            [内存布局](#内存布局)
*                    [String](#string)
*                    [Array](#array)
*                    [Parcelable](#parcelable)
*                    [FileDescriptor](#filedescriptor)
*                    [Binder](#binder)
*                    [其他](#其他)
*   [Bundle](#bundle)
*            [序列化](#序列化)
*            [存储](#存储)
*   反序列化与 Bundle 风水
*            自修改 Bundle
*            [漏洞利用](#漏洞利用)
*            [漏洞修复](#漏洞修复)
*   [CVE-2021-0928](#cve-2021-0928)
*            [漏洞介绍](#漏洞介绍)
*            Android 广播
*            Java 类型擦除
*            [其他](#其他)
*            [漏洞修复](#漏洞修复)
*   [LeakValue](#leakvalue)
*            深入 LazyValue
*            Parcel 内存管理
*            Parcel UAF
*            [漏洞利用](#漏洞利用)
*   [总结](#总结)
*   [参考资料](#参考资料)

前言
==

序列化和反序列化是指将内存数据结构转换为字节流，通过网络传输或者保存到磁盘，然后再将字节流恢复为内存对象的过程。在 Web 安全领域，出现过很多反序列化漏洞，比如 PHP 反序列化、Java 反序列化等。由于在反序列化的过程中触发了非预期的程序逻辑，从而被攻击者用精心构造的字节流触发并利用漏洞从而最终实现任意代码执行等目的。

 

Android 中除了传统的 Java 序列化机制，还有一个特殊的序列化方法，即 [Parcel](https://developer.android.com/guide/components/activities/parcelables-and-bundles)。根据官方文档的介绍，Parcelable 和 Bundle 对象主要的作用是用于跨进程边界的数据传输 (IPC/Binder)，但 Parcel 并不是一个通用的序列化方法，因此不建议开发者将 Parcel 数据保存到磁盘或者通过网络传输。

 

作为 IPC 传输的数据结构，Parcel 的设计初衷是轻量和高效，因此缺乏完善的安全校验。这就引发了历史上出现过多次的 Android 反序列化漏洞，本文就按照时间线对其进行简单的分析和梳理。

> 注: 本文中所展现的 AOSP 示例代码，如无特殊说明则都来自文章发表时的 master 分支。

Parcel 101
==========

在介绍漏洞之前，我们还是按照惯例先来了解下基础知识。对于有过 Android 开发或者逆向分析经验的同学应该对 Parcel 都不陌生，但通常也很少直接使用该类去序列化 / 反序列化数据然后进行 IPC 通信，而是通过 AIDL 等方法去自动生成模版，然后集成实现对应接口。

AIDL
----

关于 AIDL 开发的示例可以参考 [Android 进程间通信与逆向分析](https://evilpan.com/2020/07/11/android-ipc-tips/) 一文，简单来说，假设有以下 AIDL 文件:

```
package com.evilpan;
 
interface IFooService {
    parcelable Person {
        String name;
        int age;
        boolean gender;
    }
    String foo(int a, String b, in Person c);
}

```

那么生成的 (Java) 模版大致结构如下:

```
public interface IFooService extends android.os.IInterface {
 
    public java.lang.String foo(int a, java.lang.String b, com.evilpan.IFooService.Person c) throws android.os.RemoteException;
    public static class Person implements android.os.Parcelable { /* ... */ }
 
    public static abstract class Stub extends android.os.Binder implements com.evilpan.IFooService {
        @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            data.enforceInterface(descriptor);
            // ...
            switch(code) {
                case TRANSACTION_foo {
                    int _arg0;
                    _arg0 = data.readInt();
                    java.lang.String _arg1;
                    _arg1 = data.readString();
                    com.evilpan.IFooService.Person _arg2;
                    _arg2 = _Parcel.readTypedObject(data, com.evilpan.IFooService.Person.CREATOR);
                    java.lang.String _result = this.foo(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeString(_result);
                    break;
                }
            }
        }
    }
 
    private static class Proxy implements com.evilpan.IFooService {
        @Override public java.lang.String foo(int a, java.lang.String b, com.evilpan.IFooService.Person c) throws android.os.RemoteException
        {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            java.lang.String _result;
            try {
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeInt(a);
            _data.writeString(b);
            _Parcel.writeTypedObject(_data, c, 0);
            boolean _status = mRemote.transact(Stub.TRANSACTION_foo, _data, _reply, 0);
            _reply.readException();
            _result = _reply.readString();
            }
            finally {
            _reply.recycle();
            _data.recycle();
            }
            return _result;
        }
    }
    // ...
}

```

其中 `IFooService.Stub` 类是本地的 IPC 实现，即服务端代码通过继承至该类并实现其 `foo` 方法；而 `Proxy` 则是客户端的的辅助类，客户端可以通过调用 `Proxy.foo` 方法间接地调用服务端的对应代码。数据传输的过程通过 `transact` 方法实现，其底层是 Android 的 Binder IPC；而数据的封装过程则通过 Parcel 实现。

 

可以看到上面模版代码中客户端分别调用了 `writeInterfaceToken`、`writeInt`、`writeString` 和 `writeTypedObject` 来填充传输的 `_data`，而 Stub 类的 onTransact 中以同样的顺序分别调用了 `enforceInterface`、`readInt`、`readString`、`readTypedObject` 来获取 `_data` 中的数据。

Parcelable
----------

在上面的 AIDL 中，我们还定义了一个数据结构 Person，该结构同样会由 AIDL 生成对应的模版类:

```
public static class Person implements android.os.Parcelable
{
  public java.lang.String name;
  public int age = 0;
  public boolean gender = false;
  public static final android.os.Parcelable.Creator CREATOR = new android.os.Parcelable.Creator() {
    @Override
    public Person createFromParcel(android.os.Parcel _aidl_source) {
      Person _aidl_out = new Person();
      _aidl_out.readFromParcel(_aidl_source);
      return _aidl_out;
    }
    @Override
    public Person[] newArray(int _aidl_size) {
      return new Person[_aidl_size];
    }
  };
  @Override public final void writeToParcel(android.os.Parcel _aidl_parcel, int _aidl_flag)
  {
    int _aidl_start_pos = _aidl_parcel.dataPosition();
    _aidl_parcel.writeInt(0);
    _aidl_parcel.writeString(name);
    _aidl_parcel.writeInt(age);
    _aidl_parcel.writeInt(((gender)?(1):(0)));
    int _aidl_end_pos = _aidl_parcel.dataPosition();
    _aidl_parcel.setDataPosition(_aidl_start_pos);
    _aidl_parcel.writeInt(_aidl_end_pos - _aidl_start_pos);
    _aidl_parcel.setDataPosition(_aidl_end_pos);
  }
  public final void readFromParcel(android.os.Parcel _aidl_parcel)
  {
    int _aidl_start_pos = _aidl_parcel.dataPosition();
    int _aidl_parcelable_size = _aidl_parcel.readInt();
    try {
      if (_aidl_parcelable_size < 4) throw new android.os.BadParcelableException("Parcelable too small");;
      if (_aidl_parcel.dataPosition() - _aidl_start_pos >= _aidl_parcelable_size) return;
      name = _aidl_parcel.readString();
      if (_aidl_parcel.dataPosition() - _aidl_start_pos >= _aidl_parcelable_size) return;
      age = _aidl_parcel.readInt();
      if (_aidl_parcel.dataPosition() - _aidl_start_pos >= _aidl_parcelable_size) return;
      gender = (0!=_aidl_parcel.readInt());
    } finally {
      if (_aidl_start_pos > (Integer.MAX_VALUE - _aidl_parcelable_size)) {
        throw new android.os.BadParcelableException("Overflow in the size of parcelable");
      }
      _aidl_parcel.setDataPosition(_aidl_start_pos + _aidl_parcelable_size);
    }
  }
  @Override
  public int describeContents() {
    int _mask = 0;
    return _mask;
  }
} 
```

其中关键的是 writeToParcel 和 `CREATOR.createFromParcel` 方法，分别填充了该自定义结构序列化和反序列化的实现，当然我们也可以自己继承 `Parcelable` 去实现自己的可序列化数据结构。

内存布局
----

从接口上看，Parcel 可以支持按照一定顺序写入和读取 int、long 等原子数据，也支持 String、IBinder、和 FileDescriptor 这些复杂的数据结构。为了理解后文介绍的漏洞，还需要了解在二进制层面这些数据的存储方式。

 

Parcel 的代码接口实现在 _android/os/Parcel.java_ 中，但大部分方法最终都会调用到其中的 native 方法，其 JNI 定义在 _frameworks/base/core/jni/android_os_Parcel.cpp_ 文件里，最终的实现则是在 _frameworks/native/libs/binder/Parcel.cpp_ 中。

 

以 `Parcel.writeInt` 为例，其 Java 实现很简单，直接转到 native 方法:

```
private static native int nativeWriteInt(long nativePtr, int val);
public final void writeInt(int val) {
    int err = nativeWriteInt(mNativePtr, val);
    if (err != OK) {
        nativeSignalExceptionForError(err);
    }
}

```

C++ 中的 JNI 实现则是先将 nativePtr 转换为 Parcel 指针，而后直接调用 `writeInt32` 方法:

```
static int android_os_Parcel_writeInt(jlong nativePtr, jint val) {
    Parcel* parcel = reinterpret_cast(nativePtr);
    return (parcel != NULL) ? parcel->writeInt32(val) : OK;
} 
```

接下来就是最终实际的实现了:

```
status_t Parcel::writeInt32(int32_t val)
{
    return writeAligned(val);
}
 
template status_t Parcel::writeAligned(T val) {
    static_assert(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));
    static_assert(std::is_trivially_copyable_v);
 
    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        memcpy(mData + mDataPos, &val, sizeof(val));
        return finishWrite(sizeof(val));
    }
 
    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
} 
```

`writeAligned` 是个模版函数，用于写入基础的 C++ 数据类型，即 int、float、double 等，也可以写入指针数据。实现也相对简单，这里面就涉及到了 Parcel 内部的几个重要数据结构:

*   mData: 序列化数据内存缓冲区的内存起始地址 (指针)；
*   mDataPos: 序列化数据当前解析到的相对位置；
*   mDataCapacity: 缓冲区的总大小；

还有一个字段 mDataSize 表示当前序列化数据的大小，其实这个字段基本上和 mDataPos 的值是一致的，二者都在 finishWrite 函数中进行更新:

```
status_t Parcel::finishWrite(size_t len)
{
    if (len > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }
 
    //printf("Finish write of %d\n", len);
    mDataPos += len;
    ALOGV("finishWrite Setting data pos of %p to %zu", this, mDataPos);
    if (mDataPos > mDataSize) {
        mDataSize = mDataPos;
        ALOGV("finishWrite Setting data size of %p to %zu", this, mDataSize);
    }
    //printf("New pos=%d, size=%d\n", mDataPos, mDataSize);
    return NO_ERROR;
}

```

而如果当前缓冲区的内存不足，则会使用 growData 方法进行更新:

```
status_t Parcel::growData(size_t len)
{
    if (len > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }
 
    if (len > SIZE_MAX - mDataSize) return NO_MEMORY; // overflow
    if (mDataSize + len > SIZE_MAX / 3) return NO_MEMORY; // overflow
    size_t newSize = ((mDataSize+len)*3)/2;
    return (newSize <= mDataSize)
            ? (status_t) NO_MEMORY
            : continueWrite(std::max(newSize, (size_t) 128));
}

```

`continueWrite` 方法的实现比较复杂，因为其中还支持传入小于 mDataSize 的参数去缩小 Parcel 内存，不过在我们这里的上下文中仅用于增长内存，因此实际上最终只是通过 `realloc` 或者 `malloc` 去分配更多的内存:

```
if (desired > mDataCapacity) {
    uint8_t* data = reallocZeroFree(mData, mDataCapacity, desired, mDeallocZero);
    if (data) {
        LOG_ALLOC("Parcel %p: continue from %zu to %zu capacity", this, mDataCapacity,
                desired);
        gParcelGlobalAllocSize += desired;
        gParcelGlobalAllocSize -= mDataCapacity;
        mData = data;
        mDataCapacity = desired;
    } else {
        mError = NO_MEMORY;
        return NO_MEMORY;
    }
}

```

这部分目前只需要了解即可，我们关注的还是前面写入数据的逻辑，回顾一下是在 writeAligned 方法中，直接通过 `memcpy` 去写入的数据，因此对于基础数字类型是没有额外开销的，且序列化的字节序就是当前机器的字节序。这也是为什么 Parcel 只适合在同一设备中实现 IPC，如果对于不同设备中可能会出现字节序的问题。

### String

说完了 Int 我们接着看常用的 String 类型，JNI 的定义和实现如下:

```
static void android_os_Parcel_writeString16(JNIEnv *env, jclass clazz, jlong nativePtr,
        jstring val) {
    Parcel* parcel = reinterpret_cast(nativePtr);
    if (parcel != nullptr) {
        status_t err = NO_ERROR;
        if (val) {
            // NOTE: Keep this logic in sync with Parcel.cpp
            const size_t len = env->GetStringLength(val);
            const size_t allocLen = len * sizeof(char16_t);
            err = parcel->writeInt32(len);
            char *data = reinterpret_cast(parcel->writeInplace(allocLen + sizeof(char16_t)));
            if (data != nullptr) {
                env->GetStringRegion(val, 0, len, reinterpret_cast(data));
                *reinterpret_cast(data + allocLen) = 0;
            } else {
                err = NO_MEMORY;
            }
        } else {
            err = parcel->writeString16(nullptr, 0);
        }
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
} 
```

和之前差别不大，值得注意的是 `Parcel::writeInplace` 返回的是待写入的内存地址，直接用了 `JNIEnv::GetStringRegion` 去进行写入。如果 Java 传入的字符串是 `null`，则使用 `writeString16(nullptr)` 去写入，其内部实现是写入特殊的整数 `-1`:

> [GetStringRegion](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html) 返回的是 Unicode 字符，因此每个字符占 2 字节；

```
status_t Parcel::writeString16(const char16_t* str, size_t len)
{
    if (str == nullptr) return writeInt32(-1);
    // ...
}

```

因此对于字符串结构，Parcel 的序列化也是无开销顺序写入的。

### Array

数组也是个常用的数据类型，但不同的数组传输格式有所不同。对于 char/int/long 等原始类型而言，传输数组实际上就是逐个写入每个元素，并且在前面写入数组的大小:

```
public final void writeIntArray(@Nullable int[] val) {
    if (val != null) {
        int N = val.length;
        writeInt(N);
        for (int i=0; i
```

但是 wirteByteArray 有所优化:

```
public final void writeByteArray(@Nullable byte[] b, int offset, int len) {
    if (b == null) {
        writeInt(-1);
        return;
    }
    ArrayUtils.throwsIfOutOfBounds(b.length, offset, len);
    nativeWriteByteArray(mNativePtr, b, offset, len);
}

```

JNI 中直接使用 memcpy 去写入:

```
static void android_os_Parcel_writeByteArray(JNIEnv* env, jclass clazz, jlong nativePtr,
                                             jobject data, jint offset, jint length)
{
    Parcel* parcel = reinterpret_cast(nativePtr);
    parcel->writeInt32(length);
    void* dest = parcel->writeInplace(length);
    jbyte* ar = (jbyte*)env->GetPrimitiveArrayCritical((jarray)data, 0);
    if (ar) {
        memcpy(dest, ar + offset, length);
        env->ReleasePrimitiveArrayCritical((jarray)data, ar, 0);
    }
} 
```

### Parcelable

对于 `Parcelable` 类型的数据，使用 writeParcelable 方法进行写入:

```
public final void writeParcelable(@Nullable Parcelable p, int parcelableFlags) {
    if (p == null) {
        writeString(null);
        return;
    }
    writeParcelableCreator(p);
    p.writeToParcel(this, parcelableFlags);
}
 
public final void writeParcelableCreator(@NonNull Parcelable p) {
    String name = p.getClass().getName();
    writeString(name);
}

```

其中需要注意的是在调用 `Parcelable.writeToParcel` 之前，会先获取 Parcelable 实际的类名，并以字符串的方式写入。

### FileDescriptor

Andorid IPC 的一个特点是可以支持传输文件句柄，其 JNI 实现如下:

```
static void android_os_Parcel_writeFileDescriptor(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    Parcel* parcel = reinterpret_cast(nativePtr);
    if (parcel != NULL) {
        const status_t err =
                parcel->writeDupFileDescriptor(jniGetFDFromFileDescriptor(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
} 
```

object 为 `FileDescriptor` 对象，这里的实现就是获取 `FileDescriptor.fd`，即 `open(2)` 返回的，int 格式的文件描述符，进行 `fcntl(oldFd, F_DUPFD_CLOEXEC, 0)` 复制后进行写入。

 

Binder 的系统调用本身就支持传输文件类型的数据，因此这里 Parcel 的实现只需要做好上层的封装:

```
status_t Parcel::writeFileDescriptor(int fd, bool takeOwnership) {
    // ...
    switch (rpcFields->mSession->getFileDescriptorTransportMode()) {
        case RpcSession::FileDescriptorTransportMode::UNIX:
        case RpcSession::FileDescriptorTransportMode::TRUSTY: {
            if (status_t err = writeInt32(RpcFields::TYPE_NATIVE_FILE_DESCRIPTOR); err != OK) {
                return err;
            }
            if (status_t err = writeInt32(rpcFields->mFds->size()); err != OK) {
                return err;
            }
        }
    }
    flat_binder_object obj;
    obj.hdr.type = BINDER_TYPE_FD;
    obj.flags = 0;
    obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
    obj.handle = fd;
    obj.cookie = takeOwnership ? 1 : 0;
    return writeObject(obj, true);
}

```

值得一提的是，writeObject 中写入数据除了更新上面提到的 mData 等字段，还需要更新 `kernelFields`，如 `mObjects` 字段，作为内核参数的记录。

### Binder

在 Android IPC 中一个重要的参数类型就是回调，即客户端发送一个 IBinder 类型的对象给服务端，然后服务端可以调用其 onTransact 方法实现反向异步的数据传输。对于这种类型的数据，主要通过 writeStrongBinder 方法:

```
public final void writeStrongBinder(IBinder val) {
    nativeWriteStrongBinder(mNativePtr, val);
}

```

JNI 实现和传输文件类似:

```
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    Parcel* parcel = reinterpret_cast(nativePtr);
    if (parcel != NULL) {
        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
} 
```

主要分为两步，第一步取出 IBinder Java 对象的 Native 指针，即 `IBinder.mObject`，然后将其转为 C++ 的 IBinder 指针传入:

```
status_t Parcel::writeStrongBinder(const sp& val)
{
    return flattenBinder(val);
} 
```

传输 IBinder 类型的数据同样需要更新 mObjects 缓存。

### 其他

除了上面介绍的这些，Parcel 实现中还有许多值得关注的细节，比如 writeBlob 同样也是写入 `byte[]`，但对于过大的数据会选择用共享内存的方式去进行传输。但根据上面的简单分析，我们也能大致看出 Parcel 的一些问题，即序列化和反序列化纯属手工操作，并且在某些操作中没有严格的类型检查等，下面我们就来逐一探讨。

Bundle
======

在 Andorid 中，Bundle 类是一个类似 HashMap 的数据结构，但其是为了在 Parcel 序列化 / 反序列化的使用中而高度优化的。因此理解 Bundle 本身的序列化过程对我们理解后面的内容也至关重要。

序列化
---

Bundle 的序列化过程调用链路如下:

*   Bundle.writeToParcel
*   BaseBundle.writeToParcelInner
*   parcel.writeArrayMapInternal

writeToParcelInner 中主要负责写入 Bundle 相关的头部字段:

```
void writeToParcelInner(Parcel parcel, int flags) {
    // Keep implementation in sync with writeToParcel() in
    // frameworks/native/libs/binder/PersistableBundle.cpp.
    final ArrayMap map;
    synchronized (this) {
        // ...
    }
 
    // Special case for empty bundles.
    if (map == null || map.size() <= 0) {
        parcel.writeInt(0);
        return;
    }
    int lengthPos = parcel.dataPosition();
    parcel.writeInt(-1); // dummy, will hold length
    parcel.writeInt(BUNDLE_MAGIC);
 
    int startPos = parcel.dataPosition();
    parcel.writeArrayMapInternal(map);
    int endPos = parcel.dataPosition();
 
    // Backpatch length
    parcel.setDataPosition(lengthPos);
    int length = endPos - startPos;
    parcel.writeInt(length);
    parcel.setDataPosition(endPos);
} 
```

writeArrayMapInternal 则主要实现 Bundle 内部字典数据的写入:

```
void writeArrayMapInternal(@Nullable ArrayMap val) {
    if (val == null) {
        writeInt(-1);
        return;
    }
    // Keep the format of this Parcel in sync with writeToParcelInner() in
    // frameworks/native/libs/binder/PersistableBundle.cpp.
    final int N = val.size();
    writeInt(N);
    int startPos;
    for (int i=0; i
```

可以看到序列化 Bundle 的过程是和 序列化 ArrayMap 一致的，即先写入整型的 size，然后依次写入每个 key 和 value。这里值得注意的是 wirteValue 的实现:

```
public final void writeValue(@Nullable Object v) {
    if (v == null) {
        writeInt(VAL_NULL);
    } else if (v instanceof String) {
        writeInt(VAL_STRING);
        writeString((String) v);
    } else if (v instanceof Integer) {
        writeInt(VAL_INTEGER);
        writeInt((Integer) v);
    } else if (v instanceof Map) {
        writeInt(VAL_MAP);
        writeMap((Map) v);
    } // ....
    else {
        Class clazz = v.getClass();
        if (clazz.isArray() && clazz.getComponentType() == Object.class) {
            // Only pure Object[] are written here, Other arrays of non-primitive types are
            // handled by serialization as this does not record the component type.
            writeInt(VAL_OBJECTARRAY);
            writeArray((Object[]) v);
        } else if (v instanceof Serializable) {
            // Must be last
            writeInt(VAL_SERIALIZABLE);
            writeSerializable((Serializable) v);
        } else {
            throw new RuntimeException("Parcel: unable to marshal value " + v);
        }
    }
}

```

> 注意，为了便于按照时间线理解历史漏洞，这里代码使用的是 [Android 8.0 中的版本](http://androidxref.com/8.0.0_r4/xref/frameworks/base/core/java/android/os/Parcel.java#writeValue)。

 

`writeValue` 会根据对象类型分别写入一个代表类型的整数以及具体的数据。所支持的类型如下所示:

```
// Keep in sync with frameworks/native/include/private/binder/ParcelValTypes.h.
private static final int VAL_NULL = -1;
private static final int VAL_STRING = 0;
private static final int VAL_INTEGER = 1;
private static final int VAL_MAP = 2;
private static final int VAL_BUNDLE = 3;
private static final int VAL_PARCELABLE = 4;
private static final int VAL_SHORT = 5;
private static final int VAL_LONG = 6;
private static final int VAL_FLOAT = 7;
private static final int VAL_DOUBLE = 8;
private static final int VAL_BOOLEAN = 9;
private static final int VAL_CHARSEQUENCE = 10;
private static final int VAL_LIST  = 11;
private static final int VAL_SPARSEARRAY = 12;
private static final int VAL_BYTEARRAY = 13;
private static final int VAL_STRINGARRAY = 14;
private static final int VAL_IBINDER = 15;
private static final int VAL_PARCELABLEARRAY = 16;
private static final int VAL_OBJECTARRAY = 17;
private static final int VAL_INTARRAY = 18;
private static final int VAL_LONGARRAY = 19;
private static final int VAL_BYTE = 20;
private static final int VAL_SERIALIZABLE = 21;
private static final int VAL_SPARSEBOOLEANARRAY = 22;
private static final int VAL_BOOLEANARRAY = 23;
private static final int VAL_CHARSEQUENCEARRAY = 24;
private static final int VAL_PERSISTABLEBUNDLE = 25;
private static final int VAL_SIZE = 26;
private static final int VAL_SIZEF = 27;
private static final int VAL_DOUBLEARRAY = 28;

```

可以看到其中支持大部分基础类型以及 Parcelable、IBinder 等 Parcel 本身所支持的类型。

 

综上所述，我们可以大致得出 Bundle 的内存布局:

<table><thead><tr><th>size</th><th>type</th><th>value</th></tr></thead><tbody><tr><td>4</td><td>Int</td><td>length</td></tr><tr><td>4</td><td>Int</td><td>BUNDLE_MAGIC (0x4C444E42)</td></tr><tr><td>4</td><td>Int</td><td>map.size()</td></tr><tr><td>xxx</td><td>String</td><td>key[0]</td></tr><tr><td>xxx</td><td>Dynamic</td><td>val[0]</td></tr><tr><td>...</td><td>...</td><td>...</td></tr><tr><td>xxx</td><td>String</td><td>key[map.size() - 1]</td></tr><tr><td>xxx</td><td>Dynamic</td><td>val[map.size() - 1]</td></tr></tbody></table> 

其中 key 因为是 String 类型，因此内部是长度 + String16 的布局，而 value 则根据类型不同使用不同的结构。

 

key:

*   length Int32
*   data String16

value:

*   VAL_XXX Int32
*   data writeInt/writeString/...

下面是在一个 Bundle 序列化的示例，原始内容为:

```
Bundle A = Bundle()
A.putInt("key1", 0x1337)
A.putString("key2", "hello")
A.putLong("key33", 0x0123456789)

```

使用 Parcel 序列化后的二进制数据如下:

 

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

 

其中有几个需要注意的细节:

*   length 字段是不包括 length+magic 部分的，因此实际上序列化数据的总大小会比头部的结果大 8 字节即 100；
*   String 数据是 4 字节对齐的 (如 key33)；如果本身已经对齐，也会在后面进行特殊拓展，如 (key1/key2)；特殊拓展的值为 `Parcel::writeInplace` 中的 padMask，对于已经 4 字节对齐的 mask 正好是 0；

存储
--

Bundle 虽然内部是 ArrayMap 结构，但实际存储的时候还有一点优化。在平时开发中细心的朋友可能会发现，有时候在获取远程的 Bundle 比如 CotentResolver.call 的结果时候，直接打印输出会是 `Bundle[mParcelledData.dataSize=xxx]` 的结果。但是对于自己构建的 Bundle，却可以打印出完整的 Map 元素。

 

这其实可以从 Bundle 的源码中看出来:

```
@Override
public synchronized String toString() {
    if (mParcelledData != null) {
        if (isEmptyParcel()) {
            return "Bundle[EMPTY_PARCEL]";
        } else {
            return "Bundle[mParcelledData.dataSize=" +
                    mParcelledData.dataSize() + "]";
        }
    }
    return "Bundle[" + mMap.toString() + "]";
}

```

关键就是这个 `mParcelledData` 字段，该字段定义在父类即 BaseBundle 中，为 Parcel 类型。由于 Bundle 经常在进程间进行传输，因此设计上认为可以不要到对端的时候马上就反序列化出所有的 ArrayMap，而是等需要用到时再进行解析，这也可以认为是一种懒加载的技术。对于一些收到 Bundle 后马上又传输给其他服务的场景，这种设计可以直接传输未解析的 Parcel 数据而不需要来回的序列化。

 

因此对于 Bundle 而言，对于一些需要获取内部元素的调用时才会进行反序列，比如 size 和 isEmpty 的实现:

```
public int size() {
    unparcel();
    return mMap.size();
}
 
public boolean isEmpty() {
    unparcel();
    return mMap.isEmpty();
}

```

值得注意的是，在 Bundle 内部，ArrayMap(mMap) 和 Parcel(mParcelledData) 同一时间只能存在一个，反序列化后会填充 mMap 把 mParcelledData 回收并置为 null。

反序列化与 Bundle 风水
===============

最早提交 Android 反序列化漏洞的是 @BednarTildeOne 在 2014 年提交的 ParceledListSlice 漏洞，但第一次引起公众关注的应该是 CVE-2017-0806。也是同样的作者，不过给出了详细的分析以及漏洞 POC 代码，这才得以引起关注。

 

该漏洞的原理已经有很多师傅分析过了，就不再赘述，这里直接给出一个简化的版本。假设有这么一个 Parcelable 数据结构，请问是否存在漏洞？漏洞如何利用？

```
public class Vulnerable implements Parcelable {
    private long mData;
 
    protected Vulnerable(Parcel in) {
        mData = m.readInt();
    }
 
    @Override
    public void writeToParcel(Parcel parcel, int flags) {
        parcel.writeLong(mData)
    }
    // ...
}

```

问题很明显，可以看到其中序列化使用了 writeLong，但是反序列化过程却用了 readInt，二者不匹配，这可能会导致一些数据错误，但这能算得上一个漏洞吗？

 

回想 LunchAnywhere 漏洞以及对应的 [patch](https://android.googlesource.com/platform/frameworks/base/+/5bab9da%5E%21/)，实际上是用户可控的 Intent 数据中带有 `KEY_INTENT` extra 导致的启动任意 Activity 的问题。修复过程是判断用户提供的 Intent 是否带有该 extra，如果有则校验调用者的签名。当时的修复可以说没什么问题，但如果配合上述的漏洞，就有可能出现绕过。

 

这里还是以测试代码举例，假设我们的校验函数如下:

```
@Override
public void onResult(Bundle result) {
    Intent intent = result.getParcelable(AccountManager.KEY_INTENT)
    if (intent != null) {
        checkCallingSignature()
    }
    // ...
    send(result)
}
 
public void send(Bundle result); // AIDL

```

校验时如果 intent 不为空则进行签名校验，随后会调用 AIDL 的客户端方法 `send`，该方法将 bundle 再次序列化然后发送给服务端，而服务端的实现如下:

```
class Server extends IServer.Stub {
    @Override
    public void send(Bundle result) {
        Intent intent = bundle.getParcelable(KEY_INTENT);
        if (intent != null) {
            mContext.startActivity(intent)
        }
    }
}

```

因为客户端进行了校验，所以服务就不需要再次校验而是直接信任该 Bundle 并使用了。这里其实有个经典的漏洞模式即 TOCTOU 问题，一般情况下这个逻辑是没问题的，即 bundle 检查完后不会被再次修改。但在我们的漏洞场景中，就有可能会出现不同的 bundle，即 `Self-Changing Bundle`。

自修改 Bundle
----------

Bundle 自修改，主要还是针对上面的这种 IPC 场景，一端对 Bundle 进行了校验，发现没问题后使用 IPC 发送给另一端，且对方不加校验就直接使用。即 Bundle 在序列化 + 反序列化的过程中进行了改变。

 

假设服务端 B 接收我们的 Bundle 数据进行反序列化后再次序列化发送给服务端 C，而我们的 bundle 中带有一个类型为 Vunerable 的 key，那在 B 接收到数据后再发出去的数据会如下图右边所示，其中本该是 Int 的字段被写成了 Long，导致 C 再次反序列化时候对后续数据的解析出现异常:

 

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

 

通过精心构造发送给 B 的数据，我们可以令 B 和 C 都能正常序列化出 Bundle 对象，甚至让这两个 Bundle 含有不同的 key！

漏洞利用
----

该漏洞如何利用呢？前文中我们已知 Bundle 序列化数据的头部后面每个元素都是 key+value 的组合，那利用思路应该就是将多出来的 4 字节进行类型混淆。这种场景类似于二进制的缓冲区溢出，我们需要做的就是布局好数据使得同时满足下述条件:

1.  类型混淆前后，Bundle 的 length 和 items 字段保持一致；
2.  两次反序列化得到的对象都需要是合法对象；
3.  第二次反序列化要能够出现一个前面没有的 Bundle key；

溢出的 4 个字节会作为后面一个 key 的一部分，而 key 的类型是 String，根据前面的介绍，其内存是长度 (Int32) + String16。其中 String16 是 unicode，使用 2 字节保存一个字符。因此，如果 map 元素足够的话，这 4 个字节会形成下一个 key 的长度。

 

在大部分情况下，溢出的部分是 Long 数据的高有效位，因此会是 0，如果此时第二次反序列化处理到这里，会认为这有个 key 长度为 0 的元素，查看读取字符串相关的代码，如下所示:

```
const char16_t* Parcel::readString16Inplace(size_t* outLen) const
{
    int32_t size = readInt32();
    // watch for potential int overflow from size+1
    if (size >= 0 && size < INT32_MAX) {
        *outLen = size;
        const char16_t* str = (const char16_t*)readInplace((size+1)*sizeof(char16_t));
        if (str != nullptr) {
            if (str[size] == u'\0') {
                return str;
            }
            android_errorWriteLog(0x534e4554, "172655291");
        }
    }
    *outLen = 0;
    return nullptr;
}

```

考虑到 padding 的存在，即便 size 为 0，还是会读出 `(0+1)x2` 即 2 字节的 `\x00`。`readInplace` 中还会对齐，因此最终读出的是 4 字节数据。因此，后续的反序列化都会在原来的基础上后移 4 字节。

 

这样一来利用思路就比较清晰了:

1.  构造一个恶意 Bundle，其中包含两个元素 A0 和 A1，其中 A0 为 Vunlerable，在 A1 中布置 payload；
2.  反序列化 + 序列化后，数据多出 4 字节，且下个元素的解析从 A1 的 key 内容开始；
3.  构造 A1 的 key 和 value，使得第 2 步中解析出新的 key；

还是以前面的 Vulnerable 类为例，POC 代码如下:

```
val p = Parcel.obtain()
 
val lengthPos = p.dataPosition()
// header
p.writeInt(-1) // length, back-patch later
p.writeInt(0x4C444E42) // magic
val startPos = p.dataPosition();
p.writeInt(3) // numItem
 
// A0
p.writeString("A")
p.writeInt(Type.VAL_PARCELABLE.value)
p.writeString("com.evilpan.poc.Vulnerable")
p.writeInt(666) // mData
 
// A1
p.writeString("\u000d\u0000\u0008") // 0d00 0000 0800
p.writeInt(Type.VAL_BYTEARRAY.value)
p.writeInt(28)
p.writeString("intent")              // 4 + padSize((6 + 1) * 2) = 4+16 = 20
p.writeInt(Type.VAL_INTEGER.value)   // = 4
p.writeInt(0x1337)                   // = 4
 
// A2
p.writeString("BBB")
p.writeInt(Type.VAL_NULL.value)
 
// Back-patch length && reset position
 
val A = unparcel(p)
Log.i("A = " + inspect(A))
Log.i("A.containsKey: " + A.containsKey("intent"))
 
val B = unparcel(parcel(A))
Log.i("B = " + inspect(B))
Log.i("B.containsKey: " + B.containsKey("intent"))

```

在上述 POC 中，我们明面上构造了一个含有 3 个元素的 Bundle，分别是:

*   A0: Parcelable 类型，元素为我们带有漏洞的 Parcelable；
*   A1: ByteArray 类型，长度为 28 字节，ByteArray 的内容为隐藏的 Intent 元素，即 20+4+4；
*   A2: NULL 类型；

其中关键的是 A1 的 key，在触发漏洞后，会被解析为第二个元素的元素类型和长度，即解析出来的内容为:

*   0d000000: 解析为元素类型，等于 VAL_BYTEARRAY(13)；
*   08000000: 解析为 ByteArray 长度，即 8 字节；注意这里额外的 0 是字符串写入时候 pad 出来的。

之所以是 8 字节，是为了把后面的长度字段吞掉，使得解析下一个元素可以直接到我们隐藏的 intent 中。

 

第三个元素 A2 只是用于占位，在第一次序列化时候有用，第二次时直接被我们隐藏的 intent 替代了。上述代码的运行结果如下:

```
[INFO]: A = Bundle(item = 3) length: 144
[INFO]:   => 41000000 = VAL_PARCELABLE:com.evilpan.poc.Vulnerable{mData=666}
[INFO]:   => 0d00000008000000 = VAL_BYTEARRAY:0600000069006e00740065006e007400000000000100000037130000
[INFO]:   => 4200420042000000 = VAL_NULL:null
[INFO]: A.containsKey: false
[INFO]: === writeToParcel ===
[INFO]: B = Bundle(item = 3) length: 148
[INFO]:   => 41000000 = VAL_PARCELABLE:com.evilpan.poc.Vulnerable{mData=666}
[INFO]:   => 03000000 = VAL_BYTEARRAY:0d0000001c000000
[INFO]:   => 69006e00740065006e00740000000000 = VAL_INTEGER:1337
[INFO]: B.containsKey: true

```

可以看到第一次反序列化出来的 Bundle A 包含三个 key，其中并没有 intent 字段，但经过序列化再反序列化后的 Bundle B 已经修改了，第二个元素的变成了长度为 8 字节的 ByteArray，实际上其 key 长度为；第三个元素则是我们伪造的 intent 元素，值为 0x1337，因此我们成功地绕过了第一次 containsKey 的校验，而在第二次中进行触发！

> 上述例子在 Android 12 中进行测试，理论上之前的版本也类似。

 

实际上的漏洞与这个例子大同小异，由于 Android 系统对 IPC 的大量使用，许多操作都会在 `system_server`、Settings 应用、用户应用中来回穿梭，也就造成了许多上述 TOCTOU 的问题。对其中细节感兴趣的可以参考下面的分析文章:

*   [ReparcelBug - PoC for CVE-2017-0806](https://github.com/michalbednarski/ReparcelBug)
*   [CVE-2017-0806 原理分析](https://github.com/michalbednarski/IntentsLab/issues/2#issuecomment-344365482)
*   [Bundle 风水——Android 序列化与反序列化不匹配漏洞详解](https://xz.aliyun.com/t/2364)

漏洞修复
----

上述漏洞的修复似乎很直观，只需要把 Vulnerable 类中不匹配的读写修复就行了。但实际上这类漏洞并不是个例，历史上由于代码编写人员的粗心大意，曾经出现过许多因为读写不匹配导致的提权漏洞，包括但不限于:

*   CVE-2017-0806 GateKeeperResponse
*   CVE-2017-0664 AccessibilityNodelnfo
*   CVE-2017-13288 PeriodicAdvertisingReport
*   CVE-2017-13289 ParcelableRttResults
*   CVE-2017-13286 OutputConfiguration
*   CVE-2017-13287 VerifyCredentialResponse
*   CVE-2017-13310 ViewPager's SavedState
*   CVE-2017-13315 DcParamObject
*   CVE-2017-13312 ParcelableCasData
*   CVE-2017-13311 ProcessStats
*   CVE-2018-9431 OSUInfo
*   CVE-2018-9471 NanoAppFilter
*   CVE-2018-9474 MediaPlayerTrackInfo
*   CVE-2018-9522 StatsLogEventWrapper
*   CVE-2018-9523 Parcel.wnteMapInternal0
*   CVE-2021-0748 ParsingPackagelmpl
*   CVE-2021-0928 OutputConfiguration
*   CVE-2021-0685 ParsedIntentInfol
*   CVE-2021-0921 ParsingPackagelmpl
*   CVE-2021-0970 GpsNavigationMessage
*   CVE-2021-39676 AndroidFuture
*   CVE-2022-20135 GateKeeperResponse
*   ...

另一个修复思路是修复 TOCTOU 漏洞本身，即确保检查和使用的反序列化对象是相同的，但这种修复方案也是治标不治本，同样可能会被攻击者找到其他的攻击路径并绕过。

 

因此，为了彻底解决这类层出不穷的问题，Google 提出了一种简单粗暴的缓释方案，即直接从 Bundle 类中下手。虽然 Bundle 本身是 ArrayMap 结构，但在反序列化时候即便只需要获取其中一个 key，也需要把整个 Bundle 反序列化一遍。这其中的主要原因在于序列化数据中每个元素的大小是不固定的，且由元素的类型决定，如果不解析完前面的所有数据，就不知道目标元素在什么地方。

 

为此在 21 年左右，AOSP 中针对 Bundle 提交了一个称为 [LazyBundle(9ca6a5)](https://cs.android.com/android/_/android/platform/frameworks/base/+/9ca6a5e21a1987fd3800a899c1384b22d23b6dee) 的 patch。其主要思想为针对一些长度不固定的自定义类型，比如 Parcelable、Serializable、List 等结构或容器，会在序列化时将对应数据的大小添加到头部。这样在反序列化时遇到这些类型的数据，可以仅通过检查头部去选择性跳过这些元素的解析，而此时 sMap 中对应元素的值会设置为 LazyValue，在实际用到这些值的时候再去对特定数据进行反序列化。

 

这个 patch 可以在一定程度上缓释针对 Bundle 风水的攻击，而且在提升系统健壮性也有所助益，因为即便对于损坏的 Parcel 数据，如果接收方没有使用到对应的字段，就可以避免异常的发生。对于之前的 Bundle 解析策略，哪怕只调用了 `size` 方法，也会触发所有元素的解析从而导致异常。 在这个 patch 中 `unparcel` 还增加了一个 boolean 参数 `itemwise`，如果为 true 则按照传统方式解析每个元素，否则就会跳过 LazyValue 的解析。

CVE-2021-0928
=============

在前面反序列化的示例中，漏洞主要出在一个自定义的 Vulnerable 类中，即手工编写的 readFromParcel/writeToParcel 不匹配问题。在现实中，这种出现问题的类通常只在进程间使用而几乎不用于跨进程，否则在正常 IPC 调用时候就会出现明显的数据错误。但也还有一种情况，比如在 readFromParcel 方法调用的过程中出现异常，但是上级进行了 “优雅” 的 `try-catch`，从而程序并不会报错，但异常发生后 Parcel 中还有剩余数据并未消费，因此会被后续的解析错误使用，比如在 AIDL 调用时候就会被后续的参数用到。

 

`CVE-2021-0928` 就是这么一个经典的漏洞。漏洞本身只存在于 Andorid 12 beta3 的短暂时间窗口中，因此我们并不需要太过于关注细节，此外漏洞的作者也已经在 Github 上公开了该漏洞的详细介绍以及 POC 代码。作为历史漏洞研究，笔者这里只做简单的介绍，并重点关注一些值得学习的漏洞利用思路和漏洞模式。完整的漏洞细节分析和 POC 可以参考原作者的分享:

*   [CVE-2021-0928, writeToParcel/createFromParcel serialization mismatch in android.hardware.camera2.params.OutputConfiguration](https://github.com/michalbednarski/ReparcelBug2)

漏洞介绍
----

主要出现漏洞的类为 `android.hardware.camera2.params.OutputConfiguration`，其代码片段如下所示:

```
package android.hardware.camera2.params;
 
public final class OutputConfiguration implements Parcelable {
    private OutputConfiguration(@NonNull Parcel source) {
        int rotation = source.readInt();
        int surfaceSetId = source.readInt();
        // ...
        boolean isMultiResolution = source.readInt() == 1; // New in Android 12
        ArrayList sensorPixelModesUsed = new ArrayList(); // New in Android 12
        source.readList(sensorPixelModesUsed, Integer.class.getClassLoader()); // New in Android 12
        // ...
    }
 
    public static final @android.annotation.NonNull Parcelable.Creator CREATOR =
            new Parcelable.Creator() {
        @Override
        public OutputConfiguration createFromParcel(Parcel source) {
            try {
                OutputConfiguration outputConfiguration = new OutputConfiguration(source);
                return outputConfiguration;
            } catch (Exception e) {
                Log.e(TAG, "Exception creating OutputConfiguration from parcel", e);
                return null;
            }
        }
    };
} 
```

其中在 `createFromParcel` 中对构造函数进行了异常捕捉，但是对于异常仅仅是打印并返回，这导致 Parcel 数据残留从而影响后续的数据解析。

 

该漏洞本身并不复杂，是个很常见的 Parcel 序列化 / 反序列化不匹配问题，关键是这个漏洞如何利用？由于漏洞的本质是序列化数据残留，因此我们可以主要关注一些 AIDL 调用场景的参数，这样解析下一个参数时其内容也就会被我们所污染的数据控制。其中一个漏洞触发链路就是 sendBroadcast。

Android 广播
----------

关于 Android 广播的发送和接收流程应该已经有很多其他博客介绍过了，这里将其简化为下面的阶段，假设广播由应用 A 发送给应用 B；

1.  A 调用 sendBroadcast，这实际上是个 AIDL 接口，接收方即实际实现为 system_server；
2.  system_server 接收到 Intent 后进行一系列处理，判断合法的接收方，然后调用 `IApplicationThread.scheduleReceiver` 发送广播数据；这同样是一个 AIDL 接口，由各个应用在启动之初通过 `IActivityManager.attachApplication()` 传递给 system_server；因此这个调用实际上会进入到应用 B 中；
3.  B 触发 `ActivityThread.handleMessage`，进而调用 `handleReceiver`；
4.  handleReceiver 中通过传来的数据去实例化应用上下文，获取对应的 receiver，最终调用开发者注册的 onReceive 回调；

第 2 步中 scheduleReceiver 的定义如下:

```
public final void scheduleReceiver(Intent intent, ActivityInfo info,
    CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
    boolean sync, int sendingUser, int processState);

```

其中第一个 intent 是 A 应用指定的 sendBroadcast 的参数，第二个参数则是 system_server 传递给应用 B 的。漏洞利用思路就是通过 intent 参数的反序列化数据残留，间接地修改 info 参数，因为应用 B 会使用 ActivityInfo 中的数据去实例化代码，具体来说就是:

```
private void handleReceiver(ReceiverData data) {
    String component = data.intent.getComponent().getClassName();
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    app = packageInfo.makeApplicationInner(false, mInstrumentation);
    // ...
}

```

应用会通过传入的 ActivityInfo 构造 LoadedApk，其中会使用内部的 `zipPaths` 创建 ClassLoader，攻击者如果可以控制这个字段，就能实现针对应用 B 的任意代码执行。

 

要实现这个目的，还有亿点细节需要解决，首先我们先来看一个 Java 语言的特性。

Java 类型擦除
---------

Java 类型擦除也称为 [Type Erasure](https://www.baeldung.com/java-type-erasure)，主要产生在 Java 的泛型代码中。Java 虚拟机中并没有泛型类型的概念，因此在编译时实际类型会被替换为其限制对象或者原始对象 `Object` 类型，这个过程就叫做类型擦除。

 

例如，下述代码:

```
public static void printArray(E[] array) {
    for (E element : array) {
        System.out.printf("%s ", element);
    }
} 
```

会在编译时变成:

```
public static void printArray(Object[] array) {
    for (Object element : array) {
        System.out.printf("%s ", element);
    }
}

```

即模版类型 E 变成了原始类型 Object。对于有限制类型的模版，比如:

```
public static > void printArray(E[] array) {
    for (E element : array) {
        System.out.printf("%s ", element);
    }
} 
```

会在编译时替换成其限制类型:

```
public static void printArray(Comparable[] array) {
    for (Comparable element : array) {
        System.out.printf("%s ", element);
    }
}

```

那么，这个特性对于漏洞的利用有什么帮助呢？回顾 `OutputConfiguration` 类，其中有个序列化的字段 `sensorPixelModesUsed` 是 `ArrayList<Integer>` 类型，在反序列化该字段时，使用的是 `Parcel.readList` 方法，其调用链路是:

*   Parcel.readList
*   Parcel.readListInternal
*   Parcel.readValue

最终是使用 readValue 读取 List 中每个元素的值。因此实际上可以读出任何 readValue 支持的类型，比如 Parcelable、IBinder 等，并不局限于 Integer。

 

仅仅进行反序列化并不会出现任何问题，只不过在**使用**具体的元素时，如果我们实际读取的类型无法转换为整数，就会出现 `ClassCastException` 异常。在这个漏洞场景中，我们并不会使用这个数组的元素，因此我们可以指定任意的序列化类。又由于我们需要使目标类在序列化 / 反序列化过程产生不匹配，那么就需要找到一个类，使得该类可以在 system_server 中成功反序列化，但是在应用 B 中出现异常，比如 `ClassNotFoundException`。

 

原作者的利用是使用了 `PackageManagerException`，这是一个 `Serializable` 类而不是 `Parcelable`，因为 [readSerializable/ObjectInputStream 不需要指定 ClassLoader](https://cs.android.com/android/platform/superproject/+/master:libcore/ojluni/src/main/java/java/io/ObjectInputStream.java;drc=9b25969d7bf3e31c5b7fec5b34c37a304b6a7fa7;l=678?q=libcore%2Fojluni%2Fsrc%2Fmain%2Fjava%2Fjava%2Fio%2FObjectInputStream.java&ss=android%2Fplatform%2Fsuperproject)。当然肯定还存在其他可用的类，感兴趣的可以自行发掘。

> 实际利用中由于读取使用了带 classLoader 的 readList，因此到 ObjectInputStream 时 `latestUserDefinedLoader` 会被影响，解决方案是将 Serializable 再套一层 Parcelable，使用不带 ClassLoader 的 readList 去进行恢复。这里选用的是 `WindowContainerTransaction` 类。

其他
--

上面只介绍了漏洞利用的大致流程，完整的利用还有一些细节需要注意，比如:

1.  如何将任意 Parcelable 放到 Intent 中；
2.  精细的内存布局；

对于问题 1，使用 putExtras(Bundle) 并不可行，因为在 Intent 中 Bundle 会作为一个整体进行拷贝，因此 Bundle 中的反序列化错误并不会影响 Intent 本身。作者是用了一个在 Android 12 中加入的 Intent.ClipData 字段去实现的，不过目前已经修复了就不再展开了。

 

对于问题 2，有几个关键点需要注意。一是在触发异常后，并没有直接跳到第二个参数，而是会继续上级 (解析了一半的) 容器处理，因此后续 payload 需要维持这部分数据的栈平衡；另外一个问题是在 Andorid 中新增了许多隐藏 API 的限制，使得我们无法通过调用这些方法去构造 ActivityInfo 等数据，这个解决方法是参考源码直接通过 Parcel 去写入数据，只是过程繁琐了一点，当然也可以用其他方式去[绕过 Hidden API 的限制](https://www.xda-developers.com/bypass-hidden-apis/)。

漏洞修复
----

这个漏洞本身对终端用户的影响不是很大，毕竟只在 Android 12 Preview 版本中就修复了。但通过这个漏洞，Google 引入了许多修复和缓释方案，直接影响了后续的漏洞挖掘和利用思路。

 

首先针对漏洞本身，修复方案为:

1.  对上述类去除隐式的异常处理，修复读写不一致的问题；
2.  使用 readIntArray 而不是 readList/readValue 去读取数据，消灭类型擦除的副作用；
3.  防止 ClipData.mActivityInfo 写入 Parcel，除非显式指定。这消除了向 Intent 写入任意 Parcelable 的一个攻击链路；

另外，在 Andorid 13 中，引入了更强的反序列化缓释方案:

1.  新增了一个 readListInternal 方法的重载，增加额外的 `Class` 参数，显式指定读取列表的元素的类型，并且将原来的方法标记为 `@Deprecated`；
2.  新增了 [Parcel.enforceNoDataAvail](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=961;drc=9b25969d7bf3e31c5b7fec5b34c37a304b6a7fa7?q=enforceNoDataAvail&sq=&ss=android%2Fplatform%2Fsuperproject) 方法，用于确保反序列化结束后，Parcel 中不再存在多余的数据；回想上一节中 Bundle 风水的利用，实际上第三个元素在第二次反序列化中是多出来的，因此这个修改会导致上述 Bundle 风水的失败；当然也有一些绕过的手法，比如通过更复杂的风水使得第二次反序列化能够到 Parcel 的末尾即可；
3.  即上文说过的 LazyBundle patch，在 LazyBundle 实现中，Parcelable、List 等类型会在序列化数据的元素开头单独存储长度信息。不过，这个 patch 并不会影响非 Bundle 造成的反序列化漏洞，比如这个漏洞。

LeakValue
=========

时间来到 2022 年，Google 推出了 Android 13，在其中正式启用了 LazyBundle 的 patch。我们在前面 Bundle 风水中已经简要介绍过大致原理，这里再深入去分析其实现。

深入 LazyValue
------------

由于 LazyValue 是在使用时才进行反序列化的，因此在读取值时，需要预先知道它在 Parcel 中所占的数据区间，读取后还需要修改 Parcel 结构中对应的偏移。这也是为什么 LazyValue 需要在序列化数据中写入其数据长度的原因，因为对于这类数据 (如 Parcelable)，无法仅通过类型得知其数据长度。

 

读取 LazyValue 的代码实现如下所示:

```
public Object readLazyValue(@Nullable ClassLoader loader) {
    int start = dataPosition();
    int type = readInt();
    if (isLengthPrefixed(type)) {
        int objectLength = readInt();
        if (objectLength < 0) {
            return null;
        }
        int end = MathUtils.addOrThrow(dataPosition(), objectLength);
        int valueLength = end - start;
        setDataPosition(end);
        return new LazyValue(this, start, valueLength, type, loader);
    } else {
        return readValue(type, loader, /* clazz */ null);
    }
}

```

这里面有几个关键点。首先，LazyValue 中存储的是 Parcel 的引用，以及其在 Parcel 中所占的数据区间，使用其内部属性 mPosition、mLength 表示，mPostion 对应上述代码中的 `start`，mLength 对应 `valueLength`。其中 valueLength 是添加到序列化数据头部的长度字段，包括长度和类型所占的空间。

 

其次，在 Parcel 中构造 LazyValue 之后，会将 dataPostion 设置到对象对应序列化数据的尾部。在需要实际数据时，会调用 `LazyValue.apply` 方法进行真正的反序列化，如下所示:

```
@Override
public Object apply(@Nullable Class[] itemTypes) {
    Parcel source = mSource;
    if (source != null) {
        synchronized (source) {
            // Check mSource != null guarantees callers won't ever see different objects.
            if (mSource != null) {
                int restore = source.dataPosition();
                try {
                    source.setDataPosition(mPosition);
                    mObject = source.readValue(mLoader, clazz, itemTypes);
                } finally {
                    source.setDataPosition(restore);
                }
                mSource = null;
            }
        }
    }
    return mObject;
}

```

可以看到在反序列化时候会将原始 Parcel 的 dataPostion 保存，并设置指向到 LazyValue 中的位置，然后调用 readValue 去进行反序列化，完成后再次恢复 Parcel 调用前的 dataPostion。

 

这可能会涉及到一些安全问题，比如多线程之间的条件竞争，在修改 dataPostion 之后被其他线程使用，当然这通过 `synchronized` 同步块防御住了。另一个问题是，在 LazyValue 未使用之前，其所对应的 Parcel 是不能被释放的，否则就会出现类似 UAF 的内存问题。考虑到对于 Parcel 的内存分配策略而言，大多数是使用手工管理的 obtain/recycle 方式，这个问题是有可能存在的。

Parcel 内存管理
-----------

由于 Parcel 本身是为了频繁的 IPC 传输而设计的，因此对于其分配和释放通常使用手工管理的方式，以避免 Java 堆分配或者 GC 带来的性能损耗。在阅读系统源码或者 AIDL 生成的模版代码时都能发现，Parcel 使用 obtain 进行分配，使用 recycle 进行释放。

 

分配的代码实现如下:

```
public static Parcel obtain() {
    Parcel res = null;
    synchronized (sPoolSync) {
        if (sOwnedPool != null) {
            res = sOwnedPool;
            sOwnedPool = res.mPoolNext;
            res.mPoolNext = null;
            sOwnedPoolSize--;
        }
    }
    if (res == null) {
        res = new Parcel(0);
    } else {
        res.mRecycled = false;
        res.mReadWriteHelper = ReadWriteHelper.DEFAULT;
    }
    return res;
}

```

其中 `sOwnedPool` 是静态属性，`mPoolNext` 是成员属性，二者都是 Parcel 类型。因此内存池中的 Parcel 可以看做是一个表头为 sOwnedPool 的单链表结构。obtain 本质上是从链表中取出表头的数据。

 

对于 IPC 接收到的 Parcel 数据分配方式略有不同，因为这些 Parcel 在 C++ 层由系统创建，因此使用不同的链表，表头为 `sHolderPool` 静态属性。这些 Parcel 通过构造函数 `Parcel(long nativePtr)` 去构建，生命周期由系统管理 (`mOwnsNativeParcelObject` 为 `false`)，因此需要区分开来。这类 Parcel 的分配通过重载的 `obtain(long)` 方法去创建，与上述实现大同小异。

 

在释放过程会将这两种情况区分开来:

```
public final void recycle() {
    if (mRecycled) {
        Log.wtf(TAG, "Recycle called on unowned Parcel. (recycle twice?) Here: "
                + Log.getStackTraceString(new Throwable())
                + " Original recycle call (if DEBUG_RECYCLE): ", mStack);
 
        return;
    }
    mRecycled = true;
 
    // We try to reset the entire object here, but in order to be
    // able to print a stack when a Parcel is recycled twice, that
    // is cleared in obtain instead.
 
    mClassCookies = null;
    freeBuffer();
 
    if (mOwnsNativeParcelObject) {
        synchronized (sPoolSync) {
            if (sOwnedPoolSize < POOL_SIZE) {
                mPoolNext = sOwnedPool;
                sOwnedPool = this;
                sOwnedPoolSize++;
            }
        }
    } else {
        mNativePtr = 0;
        synchronized (sPoolSync) {
            if (sHolderPoolSize < POOL_SIZE) {
                mPoolNext = sHolderPool;
                sHolderPool = this;
                sHolderPoolSize++;
            }
        }
    }
}

```

释放操作相当于单链表的插入，即将释放的 Parcel 放入 sOwnPool/sHolderPool 链表的头部。从上述代码可以看出，Parcel 的分配和释放过程是后进先出 (LIFO) 的，即 Parcel obtain 会分配出最近一次释放的对象。

Parcel UAF
----------

通过 LazyValue 的实现以及 Parcel 内存管理的策略，我们似乎可以找到一个攻击场景: 在 Parcel 读取 LazyValue 之后，将 Parcel 进行释放，而后再读取对应的 LazyValue，此时如果 Parcel 被分配并填充了敏感数据，那么我们的 LazyValue 就可以读取出这些敏感内容造成数据泄露。如果泄露的数据来自其他进程，且数据中包含特权的 IBinder 等结构，那么还可能造成提权或者 RCE 的危害！

 

为此，我们首先需要找到一个 recycle 后再次使用 LazyValue 的场景。接下来，就是漏洞上场的时间了，出现上述问题的漏洞编号是 [CVE-2022-20452](https://android.googlesource.com/platform/frameworks/base/+/1aae720772a86e2db682d2e9ed77937334e475f3%5E%21/)，其 patch 代码为:

```
diff --git a/core/java/android/os/BaseBundle.java b/core/java/android/os/BaseBundle.java
index 0418a4b..b599028 100644
--- a/core/java/android/os/BaseBundle.java
+++ b/core/java/android/os/BaseBundle.java
@@ -438,8 +438,11 @@
             map.ensureCapacity(count);
         }
         try {
+            // recycleParcel being false implies that we do not own the parcel. In this case, do
+            // not use lazy values to be safe, as the parcel could be recycled outside of our
+            // control.
             recycleParcel &= parcelledData.readArrayMap(map, count, !parcelledByNative,
-                    /* lazy */ true, mClassLoader);
+                    /* lazy */ recycleParcel, mClassLoader);
         } catch (BadParcelableException e) {
             if (sShouldDefuse) {
                 Log.w(TAG, "Failed to parse Bundle, but defusing quietly", e);
@@ -1845,7 +1848,6 @@
             // bundle immediately; neither of which is obvious.
             synchronized (this) {
                 initializeFromParcelLocked(parcel, /*recycleParcel=*/ false, isNativeBundle);
-                unparcel(/* itemwise */ true);
             }
             return;
         }

```

问题还是出现在 Bundle 反序列化的过程中，之前说过 Bundle 通过 readFromParcelInner 去进行反序列化，其实现如下:

```
private void readFromParcelInner(Parcel parcel, int length) {
    final int magic = parcel.readInt();
    final boolean isJavaBundle = magic == BUNDLE_MAGIC;
    final boolean isNativeBundle = magic == BUNDLE_MAGIC_NATIVE;
    if (!isJavaBundle && !isNativeBundle) {
        throw new IllegalStateException("Bad magic number for Bundle: 0x"
                + Integer.toHexString(magic));
    }
 
    if (parcel.hasReadWriteHelper()) {
        // If the parcel has a read-write helper, it's better to deserialize immediately
        // otherwise the helper would have to either maintain valid state long after the bundle
        // had been constructed with parcel or to make sure they trigger deserialization of the
        // bundle immediately; neither of which is obvious.
        synchronized (this) {
            initializeFromParcelLocked(parcel, /*recycleParcel=*/ false, isNativeBundle);
            unparcel(/* itemwise */ true);
        }
        return;
    }
    // 直接使用 Parcel.appendFrom 拷贝原 Parcel 的数据
    Parcel p = Parcel.obtain();
    p.appendFrom(parcel, offset, length);
    mParcelledData = p;

```

注意 if 分支外部，这是大部分 Bundle 反序列化的流程，即通过新分配一个 parcel 去拷贝原始数据并保存到 mParcelledData 中。因为开发者知道 parcel 参数的生命周期不由自身控制。

 

而在 if 内部，调用了 initializeFromParcelLocked，其中会使用 readArrayMap 对 parcel 数据进行反序列化，这其中是带有 LazyValue 的。由于开发者知道 parcel 生命周期不可控，因此在后续调用了 `unparcel(true)` 去强制对每个 LazyValue 进行反序列化以去除对 parcel 数据的依赖。

 

如果 unparcel 内部某些 LazyValue 能够被攻击者绕过解析，那么这里就存在一个 UAF 漏洞，后续读取 LazyValue 时就能泄露出复用的 Parcel 中的数据。当然，要触发这个漏洞需要有几个前提:

1.  hasReadWriteHelper 返回 true；
2.  unparcel 能被绕过；

对于问题 1，我们可以在 AOSP 代码中搜索符合条件的 Parcel 类，比如 `RemoteViews`；

 

对于问题 2，我们可以尝试使其中某个 LazyValue 解析失败，比如 Parcelable 的类不存在时，代码会抛出 ClassNotFoundException，被捕捉后再次抛出 BadParcelableException。有趣的是这个异常会在 getValueAt 中被捕捉，如下所示:

```
@Nullable
final T getValueAt(int i, @Nullable Class clazz, @Nullable Class... itemTypes) {
    Object object = mMap.valueAt(i);
    if (object instanceof BiFunction) {
        try {
            object = ((BiFunction, Class[], ?>) object).apply(clazz, itemTypes);
        } catch (BadParcelableException e) {
            if (sShouldDefuse) {
                Log.w(TAG, "Failed to parse item " + mMap.keyAt(i) + ", returning null.", e);
                return null;
            } else {
                throw e;
            }
        }
        mMap.setValueAt(i, object);
    }
    return (clazz != null) ? clazz.cast(object) : (T) object;
} 
```

如果 `sShouldDefuse` 为 true，那么异常不会被再次抛出，而只是打印异常并返回 null。对于 `system_server` 而言该条件是满足的，因为其目的就是为了防止系统重要服务频繁崩溃。因此，我们可以将 Bundle 的某个值设置为另外一个容器，比如 List，然后在容器中存储一个不存在的 Parcelable，那么 List 中后续的 LazyValue 将不会被递归解析到，我们也就获得了一个 UAF 对象。

漏洞利用
----

该漏洞的利用过程比较曲折，由于漏洞已经修复，因此笔者对于复现的兴趣缺缺。此外原作者也公开了详细利用思路和漏洞利用代码，感兴趣的可以自行参考:

*   [Exploit for CVE-2022-20452, privilege escalation on Android from installed app to system app (or another app) via LazyValue using Parcel after recycle()](https://github.com/michalbednarski/LeakValue)

这里提炼一下其中值得学习的点:

1.  UAF 漏洞的核心利用目标是获取目标应用的 `IApplicationThread` 句柄，这通常是应用启动初期使用 `attachApplication()` 传递给 `system_server` 的，主要用于让后者给应用发送四大组件的生命周期回调。通过滥用这个 Binder 句柄可以实现 RCE，参考前面的 CVE-2021-0928 (scheduleReceiver)；
2.  为了能从 `system_server` 中泄露任意应用的 IApplicationThread 句柄，需要两个条件。首先是有一个接口可以发送 Parcelable 数据并将其取回，AppWidgetHost、Notification.contentView、Mediasession 都可以满足，作者选用的是 MediaSession.get/setQueue 接口；
3.  其次，要使得 UAF 对象能被包含 IApplicationThread 的 Parcel 复用，需要准确的时机。但是 attachApplication 只在应用启动时一个很小的时间窗口中被调用，因此作者寻找其中可能存在的锁去拓展这个窗口；
4.  ParceledListSlice 是一个在 IPC 间传输的特殊数据结构，在其数据量小时可以同步传输，而对于大量的数据，会将其转换为 binder 发送给对方然后进行异步传输；
5.  `ActivityManager.moveTaskToFront()` 调用时可以提供 ActivityOptions 参数，以 Bundle 形式进行封装，并在 system_server 端进行反序列化。因此攻击者可以在 Bundle 中加入 ParceledListSlice 类型的数据，从而在反序列化时回调到自身进行阻塞。该函数调用时候持有 `mGlobalLock` 锁，因此可以阻塞 attachApplication 的执行，从而更好地构造 UAF 风水；
6.  由于调用了 moveTaskToFront，种种原因导致无法使用 startActivity 来启动目标应用，因此作者使用 ContentProvider 来启动目标应用。
7.  在我们的 UAF LazyValue Parcel 被成功占位后，对应的 IApplicationThread 实际上在比较靠前的位置，但我们的 LazyValue 所在位置比较靠后因此无法读取到对应 binder。解决方案是可以找其他占位对象，不过作者这里利用了一个新的漏洞 [CVE-2022-20474](https://android.googlesource.com/platform/frameworks/base/+/569c3023f839bca077cd3cccef0a3bef9c31af63%5E%21/)，即 readLazyValue 方法中 objectLenth 只检查了证书溢出，却没检查负数的情况，因此设置负数的 prefixLength 实际上可以读取 Parcel 前方的数据，从而绕过该限制。

其他还有亿点细节，就需要动手复现才能真正理解了。理论上该漏洞可以影响安全补丁在 2022-11 之前的 Android 13 设备。 攻击的目标应用可以是任意应用，因此可以选择 `Settings.apk`，其 UID=system，攻击成功后就相当于获取到了 system 权限。这在 Goole VRP 中应该是 10 万刀乐的标准。

总结
==

本文算是笔者学习 Android 反序列化漏洞的一个笔记，按照时间线记录了几个公开的经典反序列化漏洞，并且介绍相关的修复策略以及绕过方法。从中我们可以看到，Parcel 作为轻量级的序列化方案，许多操作都需要手动管理，这导致了许多读写不匹配的问题，虽然后续引进了 LazyBundle 优化，但又引发了新的内存管理问题，使得传统二进制的 UAF 甚至 Double Free 类漏洞可以在 Java 世界重现。

 

漏洞本身只是一个引子，指导我们未来的研究方向。从这些漏洞中，我们也可以发现 Security By Default 的重要性，比如序列化 Java 容器导致的类型擦除问题和 Bundle 的序列化问题。在后期虽然尝试进行了缓释，但由于历史负担，很多不安全的设计只能缝缝补补延续下去，这也是后续挖掘和审计其他产品漏洞的一个重要着手点。

参考资料
====

*   [ReparcelBug](https://github.com/michalbednarski/ReparcelBug)
*   [ReparcelBug2](https://github.com/michalbednarski/ReparcelBug2)
*   [LeakValue](https://github.com/michalbednarski/LeakValue)
*   [Blackhat EU22 Android parcels: the bad, the good and the better - Introducing Android’s Safer Parcel](https://www.blackhat.com/eu-22/briefings/schedule/index.html#android-parcels-the-bad-the-good-and-the-better---introducing-androids-safer-parcel-28404)

  

[[2023 春季班]《安卓高级研修班 (网课)》月薪两万班招生中～](https://www.kanxue.com/book-section_list-83.htm)

[#漏洞相关](forum-161-1-123.htm)