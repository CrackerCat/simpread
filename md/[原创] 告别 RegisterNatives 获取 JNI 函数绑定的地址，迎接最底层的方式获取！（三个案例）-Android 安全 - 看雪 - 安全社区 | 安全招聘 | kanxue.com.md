> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282707.htm)

> [原创] 告别 RegisterNatives 获取 JNI 函数绑定的地址，迎接最底层的方式获取！（三个案例）

[原创] 告别 RegisterNatives 获取 JNI 函数绑定的地址，迎接最底层的方式获取！（三个案例）

发表于: 12 小时前 980

### [原创] 告别 RegisterNatives 获取 JNI 函数绑定的地址，迎接最底层的方式获取！（三个案例）

 [![](http://passport.kanxue.com/upload/avatar/562/967562.png?1669963296)](user-home-967562.htm) [mb_qzwrkwda](user-home-967562.htm) ![](https://bbs.kanxue.com/view/img/rank/4.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 12 小时前  980

告别 RegisterNatives 获取 JNI 函数绑定的地址，迎接最底层的方式获取！
=============================================

前言：
===

很多小伙伴在逆向的时候定位到了 Java 层的 Native 函数，如果要进一步进行分析，就需要找到 so 中注册的 Native 函数。

第一种情况，函数静态注册，可以直接在 so 的导出符号表中找到静态注册的函数地址（这里使用的方法是 **dlsym**）。

第二种情况，函数动态注册，在 JNI_ONLOAD 中使用 **RegisterNatives** 这个函数进行注册。

但是出现了一些特殊的情况，hook 了这两个函数，却没有找到目标函数的注册方法。

文章结构：
=====

本文章将分多个部分讲解：

本帖子是转储 notion 的，下面如果格式跟不上请查看：

[https://fortunate-decimal-730.notion.site/RegisterNatives-JNI-b83f71b4a9dc4b30a00177b71cee242c?pvs=4](https://fortunate-decimal-730.notion.site/RegisterNatives-JNI-b83f71b4a9dc4b30a00177b71cee242c?pvs=4)

获取更好的阅读体验，谢谢大家～

1、从 AOSP 源码的角度讲解 RegisterNatives 函数具体的流程

2、从 AOSP 源码出发，探究 Java 的类加载时，如何注册自己的函数地址

3、讲解函数绑定的地址究竟在哪里，如何从根本上拿到绑定函数的地址

4、如何使用工具拿到属于自己唯一的偏移地址

5、小试牛刀，用学到的知识初步测试

6、利用两个群友遇到问题的例子，一个简单的，一个复杂的，来实战应用技术

*   群友提问
    
    1 . 首先用 yang 的那个 dump so 脚本 hook 不到，然后用他那个 hook regestive 的脚本也 hook 不到注册函数
    
    2. 为什么我 hook 了 dlsym、jni 的 RegisterNative、枚举所有模块的所有导出函数都没有找到我要的函数
    

参考资料
====

*   Fart 脱壳王课程、看雪 3w 课程

脚本部分来源：Fart 脱壳王课件

寒冰老师提出的这个方法，我并不是原创，我只是实现了一个小工具以及提供了两个具体案例来实现。

欢迎大家购买看雪 2W、3W 班，以及 FART 脱壳王课程来支持寒冰老师，并获得更加充分的售后指导。

RegisterNatives 函数具体的流程
=======================

```
static jint RegisterNatives(JNIEnv* env,
2460                                jclass java_class,
2461                                const JNINativeMethod* methods,
2462                                jint method_count) {
2463      if (UNLIKELY(method_count < 0)) {
2464        JavaVmExtFromEnv(env)->JniAbortF("RegisterNatives", "negative method count: %d",
2465                                         method_count);
2466        return JNI_ERR;  // Not reached except in unit tests.
2467      }
2468      CHECK_NON_NULL_ARGUMENT_FN_NAME("RegisterNatives", java_class, JNI_ERR);
2469      ScopedObjectAccess soa(env);
2470      StackHandleScope<1> hs(soa.Self());
2471      Handle c = hs.NewHandle(soa.Decode(java_class));
2472      if (UNLIKELY(method_count == 0)) {
2473        LOG(WARNING) << "JNI RegisterNativeMethods: attempt to register 0 native methods for "
2474            << c->PrettyDescriptor();
2475        return JNI_OK;
2476      }
2477      CHECK_NON_NULL_ARGUMENT_FN_NAME("RegisterNatives", methods, JNI_ERR);
2478      for (jint i = 0; i < method_count; ++i) {
2479        const char* name = methods[i].name;
2480        const char* sig = methods[i].signature;
2481        const void* fnPtr = methods[i].fnPtr;
2482        if (UNLIKELY(name == nullptr)) {
2483          ReportInvalidJNINativeMethod(soa, c.Get(), "method name", i);
2484          return JNI_ERR;
2485        } else if (UNLIKELY(sig == nullptr)) {
2486          ReportInvalidJNINativeMethod(soa, c.Get(), "method signature", i);
2487          return JNI_ERR;
2488        } else if (UNLIKELY(fnPtr == nullptr)) {
2489          ReportInvalidJNINativeMethod(soa, c.Get(), "native function", i);
2490          return JNI_ERR;
2491        }
2492        bool is_fast = false;
2493        // Notes about fast JNI calls:
2494        //
2495        // On a normal JNI call, the calling thread usually transitions
2496        // from the kRunnable state to the kNative state. But if the
2497        // called native function needs to access any Java object, it
2498        // will have to transition back to the kRunnable state.
2499        //
2500        // There is a cost to this double transition. For a JNI call
2501        // that should be quick, this cost may dominate the call cost.
2502        //
2503        // On a fast JNI call, the calling thread avoids this double
2504        // transition by not transitioning from kRunnable to kNative and
2505        // stays in the kRunnable state.
2506        //
2507        // There are risks to using a fast JNI call because it can delay
2508        // a response to a thread suspension request which is typically
2509        // used for a GC root scanning, etc. If a fast JNI call takes a
2510        // long time, it could cause longer thread suspension latency
2511        // and GC pauses.
2512        //
2513        // Thus, fast JNI should be used with care. It should be used
2514        // for a JNI call that takes a short amount of time (eg. no
2515        // long-running loop) and does not block (eg. no locks, I/O,
2516        // etc.)
2517        //
2518        // A '!' prefix in the signature in the JNINativeMethod
2519        // indicates that it's a fast JNI call and the runtime omits the
2520        // thread state transition from kRunnable to kNative at the
2521        // entry.
2522        if (*sig == '!') {
2523          is_fast = true;
2524          ++sig;
2525        }
2526 
2527        // Note: the right order is to try to find the method locally
2528        // first, either as a direct or a virtual method. Then move to
2529        // the parent.
2530        ArtMethod* m = nullptr;
2531        bool warn_on_going_to_parent = down_cast(env)->GetVm()->IsCheckJniEnabled();
2532        for (ObjPtr current_class = c.Get();
2533             current_class != nullptr;
2534             current_class = current_class->GetSuperClass()) {
2535          // Search first only comparing methods which are native.
2536          m = FindMethod(current_class, name, sig);
2537          if (m != nullptr) {
2538            break;
2539          }
2540 
2541          // Search again comparing to all methods, to find non-native methods that match.
2542          m = FindMethod(current_class, name, sig);
2543          if (m != nullptr) {
2544            break;
2545          }
2546 
2547          if (warn_on_going_to_parent) {
2548            LOG(WARNING) << "CheckJNI: method to register \"" << name << "\" not in the given class. "
2549                         << "This is slow, consider changing your RegisterNatives calls.";
2550            warn_on_going_to_parent = false;
2551          }
2552        }
2553 
2554        if (m == nullptr) {
2555          c->DumpClass(LOG_STREAM(ERROR), mirror::Class::kDumpClassFullDetail);
2556          LOG(ERROR)
2557              << "Failed to register native method "
2558              << c->PrettyDescriptor() << "." << name << sig << " in "
2559              << c->GetDexCache()->GetLocation()->ToModifiedUtf8();
2560          ThrowNoSuchMethodError(soa, c.Get(), name, sig, "static or non-static");
2561          return JNI_ERR;
2562        } else if (!m->IsNative()) {
2563          LOG(ERROR)
2564              << "Failed to register non-native method "
2565              << c->PrettyDescriptor() << "." << name << sig
2566              << " as native";
2567          ThrowNoSuchMethodError(soa, c.Get(), name, sig, "native");
2568          return JNI_ERR;
2569        }
2570 
2571        VLOG(jni) << "[Registering JNI native method " << m->PrettyMethod() << "]";
2572 
2573        if (UNLIKELY(is_fast)) {
2574          // There are a few reasons to switch:
2575          // 1) We don't support !bang JNI anymore, it will turn to a hard error later.
2576          // 2) @FastNative is actually faster. At least 1.5x faster than !bang JNI.
2577          //    and switching is super easy, remove ! in C code, add annotation in .java code.
2578          // 3) Good chance of hitting DCHECK failures in ScopedFastNativeObjectAccess
2579          //    since that checks for presence of @FastNative and not for ! in the descriptor.
2580          LOG(WARNING) << "!bang JNI is deprecated. Switch to @FastNative for " << m->PrettyMethod();
2581          is_fast = false;
2582          // TODO: make this a hard register error in the future.
2583        }
2584 
2585        const void* final_function_ptr = m->RegisterNative(fnPtr);
2586        UNUSED(final_function_ptr);
2587      }
2588      return JNI_OK;
2589    } 
```

首先我们拿到 RegisterNative 的函数实现部分

有两个重点关注的地方：

[http://aospxref.com/android-10.0.0_r47/xref/art/runtime/jni/jni_internal.cc#2459](http://aospxref.com/android-10.0.0_r47/xref/art/runtime/jni/jni_internal.cc#2459)

java 对象转 artmethod 对象的过程

![](https://bbs.kanxue.com/upload/attach/202407/967562_U9NE3AHQ7GVXVKP.png)

在这里将 java 的 class 和签名都传入

![](https://bbs.kanxue.com/upload/attach/202407/967562_D56MRBCPKMRHFQ3.png)

从内存中遍历 artmethod，匹配出符合条件的 artmethod

第二个重要的地方

![](https://bbs.kanxue.com/upload/attach/202407/967562_J4K7TYJ6NZSDSZN.png)

artmethod 调用自己的 RegisterNative 方法

这里就**有些厂商**下沉到 artmethod 的注册方法，导致脚本 hook 不到。

![](https://bbs.kanxue.com/upload/attach/202407/967562_EWX272FUAR3XDEC.png)

```
  ALWAYS_INLINE void SetNativePointer(MemberOffset offset, T new_value, PointerSize pointer_size) {
822      static_assert(std::is_pointer::value, "T must be a pointer type");
823      const auto addr = reinterpret_cast(this) + offset.Uint32Value();
824      if (pointer_size == PointerSize::k32) {
825        uintptr_t ptr = reinterpret_cast(new_value);
826        *reinterpret_cast(addr) = dchecked_integral_cast(ptr);
827      } else {
828        *reinterpret_cast(addr) = reinterpret_cast(new_value);
829      }
830    } 
```

在这里 对 artmethod 的指针进行设置，完成对 jni 函数的绑定

总结一下：RegisterNative 的核心就是调用 SetNativePointer 这个函数，将函数的地址保存到 artmethod 中

reinterpret_cast<uintptr_t>(this) + offset.Uint32Value();

这一行正是他保存的偏移地址，artmethod 指针的偏移 32 位在源码里体现出来了，当然我们可以通过计算的方式拿到偏移地址。

![](https://bbs.kanxue.com/upload/attach/202407/967562_N9SEMEP4BXXWJ9A.png)

这个参数就是 artmethod 存储地址的地方

可以根据结构体计算出 data_的偏移

看到这里，可以揭露下本文章的核心了，就是通过 frida 拿到 artmethod 结构体，在计算出当前机器的偏移数量，查看 data_数据的内容，那么就是该 jni 地址绑定的 artmehod 的地址了

Java 的类加载时，如何注册自己的函数地址
======================

在这个板块，我们将从 LoadClass 这个函数作为切入点

![](https://bbs.kanxue.com/upload/attach/202407/967562_S9225C4JN7WQ7UJ.png)

在这个函数里有 LoadMethod 和 Linkcode 这两个核心函数

每个函数第一次都要进行一次链接绑定

![](https://bbs.kanxue.com/upload/attach/202407/967562_333J9A9G4A6BJ49.png)

在这里判断函数是否要在本地实现

重点：根据函数类型走不同的分支，我们查看 method->IsNative()

这个分支

![](https://bbs.kanxue.com/upload/attach/202407/967562_3FCDWQFKWSXCUYM.png)

发现函数调用了  
[**`UnregisterNative`](http://aospxref.com/android-10.0.0_r47/s?refs=UnregisterNative&project=art)这个方法 **

![](https://bbs.kanxue.com/upload/attach/202407/967562_TX6M2PCR8RQ2C2H.png)

在函数链接的时候，所有的 native 函数都会调用一遍 unregisternative

`SetEntryPointFromJni`

[http://aospxref.com/android-10.0.0_r47/s?defs=SetEntryPointFromJni&project=art](http://aospxref.com/android-10.0.0_r47/s?defs=SetEntryPointFromJni&project=art)

和 registernative 一样 调用了设置入口函数 而入口函数来源于 [`GetJniDlsymLookupStub](http://aospxref.com/android-10.0.0_r47/s?defs=GetJniDlsymLookupStub&project=art)()`

![](https://bbs.kanxue.com/upload/attach/202407/967562_9BNC4WZSKYWGTS4.png)

这个函数是一段内联汇编

其中内部调用了 [artFindNativeMethod](http://aospxref.com/android-10.0.0_r47/xref/art/runtime/entrypoints/jni/jni_entrypoints.cc?fi=artFindNativeMethod#artFindNativeMethod) 这个方法

[http://aospxref.com/android-10.0.0_r47/xref/art/runtime/entrypoints/jni/jni_entrypoints.cc?fi=artFindNativeMethod#artFindNativeMethod](http://aospxref.com/android-10.0.0_r47/xref/art/runtime/entrypoints/jni/jni_entrypoints.cc?fi=artFindNativeMethod#artFindNativeMethod)

最终这个函数调用了 真正的 RegisterNative 函数

![](https://bbs.kanxue.com/upload/attach/202407/967562_ACFG5FWBXXE7Z2U.png)

在这个函数里

![](https://bbs.kanxue.com/upload/attach/202407/967562_GHSUBVVSNQM72FV.png)

有着寻找函数符号的过程，可以看到静态注册的规则

![](https://bbs.kanxue.com/upload/attach/202407/967562_AQ6ZMAE3C48XFRP.png)

将 long_name 和 short_name 做拼接去寻找符号，如果没找到则保留 null，等待开发人员进行绑定

我们可以理解为，jni 函数一开始都绑定在一个地址上，程序员需要在 jni_onload 再去二次绑定上自己的真实的地址（这里在后面有一个坑）

函数绑定的地址究竟在哪里，如何拿到对应的偏移?
=======================

认真阅读的读者心中已经有了答案，就在 Artmethod 的 data_这个属性里，我们只需要拿到函数的 **artmethod 指针**以及知道**自己系统 artmethod 的储存绑定地址的偏移**即可。

偏移地址如何优雅的获取？
------------

我们可以自己写一个小 demo，手动调用 registernative，绑定我们自己的地址到函数上，然后拿到对应的 artmethod，对内存进行搜索，取出符合条件的 index

demo 开发原理
---------

在 aosp8.0-aosp10 的系统上，artmethod 的指针就是 jmethodid 的数值，这里我们可以通过源码来查看 在 aosp11 的时候这一特性发生了变化，aosp 为了安全，将 artmetod 指针建立了一个数组，并返回了一个 id 作为 index

![](https://bbs.kanxue.com/upload/attach/202407/967562_3PNYB495FGZFACC.png)

![](https://bbs.kanxue.com/upload/attach/202407/967562_W97BSS4NB8Q3P97.png)

从这里看到，jmethoidid 只是将 artmethod 强转了

所以在 aosp10 以下，可以直接通过

![](https://bbs.kanxue.com/upload/attach/202407/967562_MEYFPZNXEFF5DMU.png)

来直接获取到手机的偏移地址

在 aosp10 以上怎么办？ 非常好办，frida 就可以帮你做内存检索，虽然比 app 一键获取要来的慢

下面我们进入下一个篇章，如何用开发的 demo 获取到你手机目前的偏移地址

利用自写的工具，拿到你当前手机的 ArtMethod 偏移
=============================

aosp10.0 以下：
------------

打开我们自己实现的 app

![](https://bbs.kanxue.com/upload/attach/202407/967562_JE9SUF6BFUEMX87.png)

我们可以看到是 4 个指针大小（并不是字节，上面打错了）

如果你的 app 运行在 32 位模式下，那么就是 4x4（32 位指针大小 4 字节）=16 字节

```
adb install --abi armeabi-v7a xxx.apk

```

这样安装会让你的 apk 强制运行在 32 位模式下，其余手机基本默认都运行在 64 位下

不确定的可以调用 frida 的 api Proces.pointersize

我的 app 是运行在 64 位模式下，那么就是 4X8（64 位指针大小 8 字节）=32 字节

如果你的手机系统在安卓 10 以上
-----------------

打开 app 是另外一个界面

![](https://bbs.kanxue.com/upload/attach/202407/967562_8TCMTH78AQYXJY7.png)

我们首先获取目标类的 artmethod 地址

将 frida 挂载到 demo app 上面

```
function getHandle(object) {
    var handle = null;
    try {
        handle = object.$handle;
    } catch (e) {
    }
    if (handle == null) {
        try {
            handle = object.$h;
        } catch (e) {
        }
 
    }
    if (handle == null) {
        try {
            handle = object.handle;
        } catch (e) {
        }
 
    }
    return handle;
}
 
Java.perform(function () {
    let ReadableNativeMap = Java.use("com.example.getoffsite.MainActivity");
    console.log(getHandle(ReadableNativeMap["stringFromJNI"]))
 
});

```

不做任何修改的运行

![](https://bbs.kanxue.com/upload/attach/202407/967562_DYTZR9T7PF9AHFQ.png)

拿到第一个值 也就是 artmethod 的地址 0x754d267ed8

接下来从界面上抄来第二个值，填入下面的脚本

```
var startAddress = ptr('0x754d267ed8');  // artmethod地址
var targetValue = ptr('0x74d82ebbb0');  // app界面上的值
var scanLength = 1024;  // 扫描长度（字节数）
 
function scanMemory(address, target, length) {
    for (var i = 0; i < length; i++) {
        var currentAddress = address.add(i);
        var currentValue = Memory.readPointer(currentAddress);
 
        if (currentValue.equals(target)) {
            console.log('Found match at address: ' + currentAddress);
            console.log("offsite",currentAddress.sub(startAddress));
            return;
        }
    }
    console.log('No match found within the specified range.');
}
 
scanMemory(startAddress, targetValue, scanLength);

```

注入脚本

![](https://bbs.kanxue.com/upload/attach/202407/967562_HSQSVMP2JUEN4B7.png)

就可以获取到你偏移的字节了这里是 0x10 也就是 16(64 位下）

如果目标 app 比较老 运行在 32 位模式下

```
adb install --abi armeabi-v7a demo.apk

```

强制 demo app 强制运行在 32 位模式下，即可拿到 32 位的偏移

小试牛刀，获取一个 demoapp 的 jni 绑定地址
============================

我们目标要获取的类名是

com.example.test_1.MainActivity

方法名是

public native String stringFromJNI();

首先启动好 app，frida 进行附加

运行脚本，获取到目标类的 artmethod

```
function getHandle(object) {
    var handle = null;
    try {
        handle = object.$handle;
    } catch (e) {
    }
    if (handle == null) {
        try {
            handle = object.$h;
        } catch (e) {
        }
 
    }
    if (handle == null) {
        try {
            handle = object.handle;
        } catch (e) {
        }
 
    }
    return handle;
}
 
Java.perform(function () {
    let ReadableNativeMap = Java.use("com.example.test_1.MainActivity");
    console.log(getHandle(ReadableNativeMap["stringFromJNI"]))
 
});

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_EFCGGTZR5M3AKTH.png)

之后阅读偏移的 16 个字节（上一个板块的获取到的）的信息

```
ptr(0x75480b3ed8).add(16).readPointer();

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_67H78B9KV4KDUUD.png)

这就是这个 art 方法绑定的方法了 我们使用 DebugSymbol.fromAddress 查看具体符号信息

![](https://bbs.kanxue.com/upload/attach/202407/967562_YXTVAUJAHDYQ4X7.png)

简单计算一下偏移

```
Process.getModuleByName("libtest_1.so")

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_ZFFXJG465QMJNQJ.png)

使用获取到的地址减去模块的 base，得到偏移

![](https://bbs.kanxue.com/upload/attach/202407/967562_H2DHF62KP8FKCBP.png)

0x1dd80

至此，我们的小试牛刀结束了，下面循序渐进的解决两位群友问题

样本 1:
=====

问题：为什么我 hook 了 dlsym、jni 的 RegisterNative、枚举所有模块的所有导出函数都没有找到我要的函数

app 名称：人保 e 通

目标类型和函数

com.facebook.react.bridge.ReadableNativeMap

![](https://bbs.kanxue.com/upload/attach/202407/967562_6JDUHCCJQBZUMSU.png)

第一步，使用脚本拿到 artmethod 地址：

```
function getHandle(object) {
    var handle = null;
    try {
        handle = object.$handle;
    } catch (e) {
    }
    if (handle == null) {
        try {
            handle = object.$h;
        } catch (e) {
        }
 
    }
    if (handle == null) {
        try {
            handle = object.handle;
        } catch (e) {
        }
 
    }
    return handle;
}
Java.perform(function () {
    let ReadableNativeMap = Java.use("com.facebook.react.bridge.ReadableNativeMap");
    console.log(getHandle(ReadableNativeMap["importValues"]))
 
});

```

拿到了目标地址

![](https://bbs.kanxue.com/upload/attach/202407/967562_P5KBNTFF4TV2YD4.png)

第二步，阅读指针内容

```
ptr(0x79b4c96368).add(32).readPointer();   //这里我使用的安卓8.0系统 32是4个指针乘8字节

```

成功拿到地址：

![](https://bbs.kanxue.com/upload/attach/202407/967562_B4WESU4WJ6QPUSG.png)

解析下符号：

![](https://bbs.kanxue.com/upload/attach/202407/967562_AHVR5HKCQWFXXHC.png)

样本 2: 压轴戏（推荐观看）
---------------

问题：. 首先用 yang 的那个 dump so 脚本 hook 不到，然后用他那个 hook regestive 的脚本也 hook 不到注册函数

目标样本 app: 正保会计网校

老套路，获取到目标类型的 artmethod：

```
function getHandle(object) {
    var handle = null;
    try {
        handle = object.$handle;
    } catch (e) {
    }
    if (handle == null) {
        try {
            handle = object.$h;
        } catch (e) {
        }
 
    }
    if (handle == null) {
        try {
            handle = object.handle;
        } catch (e) {
        }
 
    }
    return handle;
}
 
Java.perform(function() {
    // 定位类
    var targetClass = Java.use('com.cdel.encode.TSEncode');
    console.log(getHandle(targetClass.de1))
});

```

0x7b3ff992c8

![](https://bbs.kanxue.com/upload/attach/202407/967562_5FY6VSUJ5EF8BZE.png)

拿到目标函数地址：

```
ptr(0x7b3ff992c8).add(32).readPointer();

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_HT5TCQ7RPVDHFA5.png)

奇怪？ 为什么他绑定在了 art 里面呢，仔细一看

art_jni_dlsym_lookup_stub

这不就是第一次统一 unregisternative 的地址吗

具体原理请看上面的第三部分

我们该怎么办？

非常简单，主动调用一次即可！

```
Java.perform(function() {
    // 定位类
    var targetClass = Java.use('com.cdel.encode.TSEncode');
 
    // 定义要传递的参数
    var param = "7ZvLaMCWJPFQmQX87ZvLaMCWJPEFUzIwJGPZwXlCunyRfQ8xqyCsSt1ADfx3xI3LZkeb.w__X8bMvisv";
 
    // 调用目标方法并获取返回值
    var result = targetClass.de1(param);
 
    // 输出结果
    console.log("Result: " + result);
});

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_K4ZMSPV7P776E59.png)

调用成功后我们再次查看地址

![](https://bbs.kanxue.com/upload/attach/202407/967562_S8DNMCN2QXSNBF9.png)

果然 地址发生了变化

奇怪的事情来了，他并没有任何符号，仅仅是一个地址

![](https://bbs.kanxue.com/upload/attach/202407/967562_K42HXZXT5RPK5MU.png)

难道我们的字节读取错误了吗

使用 hexdump 查看一下 artmethod 在内存中的值

```
[Pixel::com.cdel.accmobile ]-> console.log(hexdump(ptr(0x7b3ff992c8)))
             0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
7b3ff992c8  80 fd e4 14 09 01 00 00 00 00 00 00 2f 1e 00 00  ............/...
7b3ff992d8  03 00 00 00 00 00 00 00 00 b0 10 53 7b 00 00 00  ...........S{...
7b3ff992e8  **80 20 2c 3d 7b** 00 00 00 60 0b b1 68 7b 00 00 00  . ,={...`..h{...
7b3ff992f8  80 fd e4 14 09 01 00 00 00 00 00 00 30 1e 00 00  ............0...
7b3ff99308  04 00 00 00 00 00 00 00 00 b0 10 53 7b 00 00 00  ...........S{...
7b3ff99318  a0 6d b0 68 7b 00 00 00 60 0b b1 68 7b 00 00 00  .m.h{...`..h{...
7b3ff99328  80 fd e4 14 09 01 00 00 00 00 00 00 31 1e 00 00  ............1...
7b3ff99338  05 00 00 00 00 00 00 00 00 b0 10 53 7b 00 00 00  ...........S{...
7b3ff99348  a0 6d b0 68 7b 00 00 00 60 0b b1 68 7b 00 00 00  .m.h{...`..h{...
7b3ff99358  01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7b3ff99368  00 00 00 00 ff ff ff ff 00 00 00 00 00 00 00 00  ................
7b3ff99378  00 00 00 00 00 00 00 00 a0 93 f9 3f 7b 00 00 00  ...........?{...
7b3ff99388  70 08 b1 68 7b 00 00 00 00 00 00 00 00 00 00 00  p..h{...........
7b3ff99398  00 00 00 00 00 00 00 00 30 a3 68 70 00 00 00 00  ........0.hp....
7b3ff993a8  d8 2e 70 70 00 00 00 00 00 00 00 00 00 00 00 00  ..pp............
7b3ff993b8  00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00  ................

```

对比了下标横线的地址，我们获取的并没有错误，我们该怎么办？

当然是去 map 查找他所在的段，查看是不是可执行的，如果是，那么目标 so 就使用了动态释放内存的操作，将可执行代码用 mmap 释放到内存中并执行

![](https://bbs.kanxue.com/upload/attach/202407/967562_PDXFNCVTZR7PZRD.png)

获取到了目标进程的 pid，我们在开启一个 shel

```
cat /proc/7129/maps >/data/local/tmp/map.txt

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_D8VFW4NYU2T358Q.png)

找到了三个可以的段 连名字都没有

而且发现 目标地址正是在

7b3d228000-7b3d453000 rwxp 00000000 00:00 0

这个段中

并且这个段还有执行权限，非常可疑，我们来进行内存 dump

有三种方式可以 dump

第一种 使用 dd 命令 dd if = 具体可以问 gpt 如何操作

第二种 使用 frida 脚本 dump 下 memory 使用 file 写入文件

第三种 使用开源项目

[https://github.com/kp7742/MemDumper](https://github.com/kp7742/MemDumper)

[https://github.com/maiyao1988/elf-dump-fix](https://github.com/maiyao1988/elf-dump-fix)

文章结尾会打包好 所有需要的文件 下面我们开始 dump

```
255|sailfish:/data/local/tmp # ./memdumper64 -m -s 7b3d228000 -e 7b3d453000 -n 123.bin -i 7129 -o /sdcard

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_M5GGTTMXP42FFSX.png)

进行 dump 后 我们拿到目标文件查看

![](https://bbs.kanxue.com/upload/attach/202407/967562_UGCWMUZKVCF2NP7.png)

是一个 elf 文件

进行修复后我们导入 ida

并计算偏移地址

base:7b3d228000

func ptr :0x7b3d2c2080

![](https://bbs.kanxue.com/upload/attach/202407/967562_DR5KND2HZDYWMK2.png)

计算出偏移地址：

0x9a080

发现就是我们想要的函数

![](https://bbs.kanxue.com/upload/attach/202407/967562_ZTKYTVCS822TZA7.png)

小彩蛋：

[libproxy.so](http://libproxy.so) 在 init_proc 中 很奔放的写出了释放过程，大家可以去 debug 学习下

![](https://bbs.kanxue.com/upload/attach/202407/967562_NZEE5XFWYSFJV4K.png)

尾言：
===

所有用到的文件打包地址：

链接: [https://pan.baidu.com/s/1d3Ym-piDQe49A9-XcJVrhA?pwd=euwa](https://pan.baidu.com/s/1d3Ym-piDQe49A9-XcJVrhA?pwd=euwa) 提取码: euwa

第二第三部分写的非常有瑕疵，欢迎大佬来指正，我会及时修改帖子内容！

希望大家能从我的帖子学到一些东西，现在的东西深度不是很够，我会努力学习给大家带来高质量的帖子～

大家有问题可以给我留言，我会每天看 3-5 次来解决大家的问题！

  

[[竞赛]2024 KCTF 大赛征题截止日期 08 月 10 日！](https://bbs.kanxue.com/thread-281194.htm)

最后于 37 分钟前 被 mb_qzwrkwda 编辑 ，原因： [#逆向分析](forum-161-1-118.htm) [#系统相关](forum-161-1-126.htm) [#工具脚本](forum-161-1-128.htm)