> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287331.htm)

> [原创] 利用 ArtMethod，Frida Attach 获取已注册的 JNI Native 方法地址

[原创] 利用 ArtMethod，Frida Attach 获取已注册的 JNI Native 方法地址

发表于: 2 小时前 53

[举报](javascript:void(0);)

### [原创] 利用 ArtMethod，Frida Attach 获取已注册的 JNI Native 方法地址

 [![](http://passport.kanxue.com/upload/avatar/098/956098.png?1654817794)](user-home-956098.htm) [lihe07](user-home-956098.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star_0.gif) [ 举报](javascript:void(0);) 2 小时前  53

**问题背景**  

-----------

在实际脱壳中，经常遇到应用程序无法 Frida Spawn，只能 Frida Attach 进行分析的情况。大部分的安卓壳都会在 App 启动阶段加入 Hook 检测和反调试，而在 App 启动后 Attach 可以很好地避免被检测到。

但跳过 App 启动阶段，导致我们无法采用常见的 Hook ArtMethod::RegisterNative 方法来获得动态绑定的 Native 方法地址。本文以 360 加固为例，讨论一种获取正在运行中的 App 的 Native 方法地址的方法。

环境
--

1.  360 加固的 APP（包含 Native 方法 + exports 隐藏 + JNI_OnLoad 动态绑定）
    
2.  Android 9 Pie
    
3.  Frida v16
    

动态绑定流程
------

关于 Native 方法的动态绑定，论坛上已经有很多文章了，不过为了方便参考，此处便再次进行赘述。

当 App 调用 System.loadLibrary 加载 包含 JNI 的 .so 时，首先会像加载一个常规 .so 一样，调用 dlopen。这之后，如果发现该 .so 包含 JNI_OnLoad 方法，则会立即调用。大部分的动态绑定就是在这一阶段进行的（也有可能在之后进行，JNI 允许更换绑定）。

动态绑定需要调用这样的方法：

```
static JNINativeMethod gMethods[] = {
    {"my_method", "()Ljava/lang/String;", (void*)my_method }};
 
(*env)->RegisterNatives(env, clazz, gMethods, sizeof(gMethods) / sizeof(gMethods[0]));
```

在 Android 平台上，JNI 的 RegisterNatives 方法由 Dalvik (Android 5 之前) 或者 ART (Android 5 或者更新) 实现。此处我们以 Android 9 为例，找到 ART 这部分的源代码

runtime/jni_internal.cc:

```
static jint RegisterNatives(JNIEnv* env,
                              jclass java_class,
                              const JNINativeMethod* methods,
                              jint method_count) {
    // 省略各种检查...
    for (jint i = 0; i < method_count; ++i) {
        // ...
        ArtMethod* m = FindMethod(current_class.Ptr(), name, sig); // 获取到对应的ArtMethod
        // ...
        const void* final_function_ptr = m->RegisterNative(fnPtr); // ArtMethod上的注册方法
        // ...
    }
} 
```

继续查看 ArtMethod::RegisterNative 的实现

runtime/art_method.cc:

```
const void* ArtMethod::RegisterNative(const void* native_method) {
    CHECK(IsNative()) << PrettyMethod();
    CHECK(native_method != nullptr) << PrettyMethod();
    void* new_native_method = nullptr;
    Runtime::Current()->GetRuntimeCallbacks()->RegisterNativeMethod(this,
                                                                  native_method,
                                                                  /*out*/&new_native_method);
    SetEntryPointFromJni(new_native_method);
    return new_native_method;
}
```

不难发现，如果我们能获得一个 Native 方法对应的 ArtMethod 对象，并读取它的 EntryPoint field，便可获得内存中已经解密后的 Native 方法位置，辅助我们进行 dump 分析。

Frida 实现
--------

借助 Frida-Java-Bridge，可以很轻松地实现这个操作。参考 [Frida-Java-Bridge 源代码](elink@248K9s2c8@1M7q4)9K6b7g2)9J5c8W2)9J5c8X3N6A6N6r3S2#2j5W2)9J5k6h3y4G2L8g2)9J5c8X3k6J5K9h3c8S2i4K6u0r3k6Y4u0A6k6r3q4Q4x3X3c8B7j5i4k6S2i4K6u0V1j5Y4u0A6k6r3N6W2i4K6u0r3j5X3I4G2j5W2)9J5c8X3#2S2K9h3&6Q4x3V1k6D9K9h3u0Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0W2K9Y4x3`.) ， android.js 中已经提供了一些 ArtMethod 的封装方法（但没有导出）。

将其复制出，并根据 libart.so 中获取的 ArtMethod 字段的偏移量，我们可以写出如下封装：

```
function getApi(): any {
  return (Java as any).api
}
 
class StdString {
  handle: NativePointer;
 
  constructor() {
    this.handle = Memory.alloc(3 * Process.pointerSize);
  }
 
  dispose() {
    const [data, isTiny] = this._getData();
    if (!isTiny) {
      getApi().$delete(data);
    }
  }
 
  disposeToString() {
    const result = this.toString();
    this.dispose();
    return result;
  }
 
  toString() {
    const str = this.handle;
    const isTiny = (str.readU8() & 1) === 0;
    const data = isTiny ? str.add(1) : str.add(2 * Process.pointerSize).readPointer();
    return data.readUtf8String();
  }
 
  _getData() {
    const str = this.handle;
    const isTiny = (str.readU8() & 1) === 0;
    const data = isTiny ? str.add(1) : str.add(2 * Process.pointerSize).readPointer();
    return [data, isTiny];
  }
}
 
class ArtMethod {
  handle: NativePointer;
 
  constructor(handle: NativePointer) {
    this.handle = handle;
  }
 
  prettyMethod(withSignature = true) {
    const result = new StdString();
    getApi()['art::ArtMethod::PrettyMethod'](result, this.handle, withSignature ? 1 : 0);
    return result.disposeToString();
  }
 
  toString() {
    return `ArtMethod(handle=${this.handle})`;
  }
 
  methodIdx() {
    return this.handle.add(12).readU32();
  }
 
  isNative() {
    return (this.handle.add(4).readU32() & 0x100) != 0;
  }
 
  getJniEntry() {
    return this.handle.add(0x18).readPointer()
  }
}
```

封装的使用方法也十分简单：  

```
const MyJni = Java.use("com.xxxx.app.MyJni");
 
const MyArtMethod = new ArtMethod(MyJni.myMethod.handle);
 
console.log(MyArtMethod.prettyMethod(true))
console.log("isNative:", MyArtMethod.isNative())
console.log("jniEntry:", MyArtMethod.getJniEntry())
```

  

[[培训] 科锐逆向工程师培训第 53 期 2025 年 7 月 8 日开班！](https://bbs.kanxue.com/thread-51839.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)