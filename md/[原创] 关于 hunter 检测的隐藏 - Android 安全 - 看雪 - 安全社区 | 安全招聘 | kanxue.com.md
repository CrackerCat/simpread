> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286674.htm)

> [原创] 关于 hunter 检测的隐藏

检测点
---

![](https://bbs.kanxue.com/upload/attach/202505/98159_ZWAFFA3WFCGRPKR.webp)  
![](https://bbs.kanxue.com/upload/attach/202504/98159_8JQ8YMR23UCF7KJ.webp)

### Find Xposed Api

这种最好过的了, 直接自编译一下`lsposed`, 改改包名类名就行了.

### Hunter 可能已经被注入

这种就是有可执行的匿名内存, 参考: https://github.com/reveny/Android-Library-Remap-Hide  
我的方法是`open`一个`/system/lib`下的正常 so 文件 (要求文件大小比内存块大), `mmap`后, `move`内存后`mremap`回去, 但有些应用会崩溃, 原因没查出来, 所以**看看**就好, 生产环境不能用, 过`hunter` `momo` `nativeTest`是没有问题的

#### 底下那一堆 Find JVM Method Is Hooked 也是在匿名内存搞定后就没了

### inMemoryDexClassLoader

这个就更好过了

```
jmethodID initMid;
if (sdk_int < __ANDROID_API_Q__) {
    // https://cs.android.com/android/platform/superproject/+/android-9.0.0_r1:libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java;l=123
    initMid = JNI_GetMethodID(env, in_memory_classloader, "",
                              "([Ljava/nio/ByteBuffer;Ljava/lang/ClassLoader;)V");
} else {
    // https://cs.android.com/android/platform/superproject/+/android-10.0.0_r1:libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java;l=173
    initMid = JNI_GetMethodID(env, in_memory_classloader, "",
                              "([Ljava/nio/ByteBuffer;Ljava/lang/String;Ljava/lang/ClassLoader;)V");
}
auto byte_buffer_class = JNI_FindClass(env, "java/nio/ByteBuffer");
auto dex_buffer = env->NewDirectByteBuffer(dex_data, dex_size);
auto dex_buffer_array = JNI_NewObjectArray(env, 1, byte_buffer_class, dex_buffer);
 
if (sdk_int < __ANDROID_API_Q__) {
    if (auto my_cl = JNI_NewObject(env, in_memory_classloader, initMid,
                                   dex_buffer_array, sys_classloader)) {
        inject_class_loader_ = JNI_NewGlobalRef(env, my_cl);
    }
} else {
    jstring librarySearchPath = nullptr;
    if (auto my_cl = JNI_NewObject(env, in_memory_classloader, initMid,
                                   dex_buffer_array, librarySearchPath, sys_classloader)) {
        inject_class_loader_ = JNI_NewGlobalRef(env, my_cl);
    }
} 
```

直接调它的父类

### CRC 校验

这个也简单, 可以参考: https://bbs.kanxue.com/thread-285790.htm

hunter 的检测相对 NativeTest 简单点
---------------------------

![](https://bbs.kanxue.com/upload/attach/202504/98159_F8AA5WZN4H4PZB6.webp)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 2025-4-30 14:11 被无味编辑 ，原因： 格式

[#逆向分析](forum-161-1-118.htm) [#其他](forum-161-1-129.htm) [#漏洞相关](forum-161-1-123.htm)