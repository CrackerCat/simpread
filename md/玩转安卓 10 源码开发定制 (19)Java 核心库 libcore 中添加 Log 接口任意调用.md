> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Kch0K0bFCTgenIY_Rt3ZyQ)

  

一、前言
====

最近在弄系统定制功能的时候 (比如打印 File 构造函数的参数), 需要修改 libcore 中的核心代码并打印日志输出。虽然 Android 提供了 android.utils.Log 日志工具类，但是不能在 android java 层的核心库 libcore 中调用。虽然可以使用 System.out 输出，但是不能满足需求。所以参考 android.utils.Log 的实现方式移植到 libcore 核心库中供整个安卓系统 java 层调用。

二、移植 Log 的方式讨论
==============

将 android.utils.Log 移植到 libcore 中可以两种方式:

**(1)、在现有的类中添加方法，并在 native 追加对应的方法，这种方式相对简单**

比如 java.lang.System 中添加相应的方法，在对应 System.c 中添加对应的 jni 方法。

**(2)、在 libcore 中添加新的接口类，并实现 native 的方法，这种方式相对复杂一点**

本着学习研究的心态，以下选方案 2 来操作。

三、移植过程
======

1. 添加 java.lang.XLog 日志类
------------------------

### （1）. 创建 java.lang.XLog 类

在 libcore 路径位置:

```
/home/qiang/lineageOs/libcore/ojluni/src/main/java/java/lang


```

目录下新建 XLog 文件，如下所示:

```
qiang@ubuntu:~/lineageOs/libcore/ojluni/src/main/java/java/lang$ ls -la XLog.java 
-rwxrwxrwx 1 qiang qiang 2124 1月  19 20:50 XLog.java


```

### （2）. 参考 android.utils.Log 的实现添加 XLog 日志方法

此处只添加几个常用的方法，添加之后核心代码如下所示:

```
 ...省略
public static native int println_native(int bufID, int priority, String tag, String msg);
  
public static int d(@Nullable String tag, @NonNull String msg) {
     return println_native(LOG_ID_MAIN, DEBUG, tag, msg);
}
 ...省略


```

### （3）、将 XLog.java 文件添加到编译文件链

在 / home/qiang/lineageOs/libcore/penjdk_java_files.bp 文件中加入 XLog.java 到编译文件链。如下所示:

```
// Classes which are part of the public API, except where classes and
// members are hidden using @hide javadoc tags.
filegroup {
    name: "openjdk_javadoc_files",
    srcs: [
        ...省略
        "ojluni/src/main/java/java/lang/System.java",
        ///ADD START
        "ojluni/src/main/java/java/lang/XLog.java",
        ///ADD END
        "ojluni/src/main/java/java/lang/ThreadDeath.java",
        ...省略
        
    ],
}
...省略


```

2.XLog.java jni 实现
------------------

### （1）. 在 libcore native 层添加 XLog.c

XLog.c 主要是实现 XLog 的 native 方法。XLog.c 创建之后如下:

```
qiang@ubuntu:~/lineageOs/libcore/ojluni/src/main/native$ pwd
/home/qiang/lineageOs/libcore/ojluni/src/main/native
qiang@ubuntu:~/lineageOs/libcore/ojluni/src/main/native$ ls -la XLog.c 
-rwxrwxr-x 1 qiang qiang 9678 1月  20 00:13 XLog.c



```

参考 android_util_Log.cpp 中 println_native 实现方式，在 XLog.c 中添加如下实现:

```
static jint java_lang_XLog_println_native(JNIEnv* env, jobject clazz,
        jint bufID, jint priority, jstring tagObj, jstring msgObj)
{
    const char* tag = NULL;
    const char* msg = NULL;

    if (msgObj == NULL) {
        JNU_ThrowNullPointerException(env, "println needs a message");
        return -1;
    }

    if (bufID < 0 || bufID >= LOG_ID_MAX) {
        JNU_ThrowNullPointerException(env, "bad bufID");
        return -1;
    }

    if (tagObj != NULL)
        tag = (*env)->GetStringUTFChars(env,evn,tagObj, NULL);
    msg = (*env)->GetStringUTFChars(env,msgObj, NULL);

    int res = __android_log_buf_write(bufID, (android_LogPriority)priority, tag, msg);

    if (tag != NULL)
        (*env)->ReleaseStringUTFChars(env,tagObj, tag);
    (*env)->ReleaseStringUTFChars(env,msgObj, msg);

    return res;
}


```

在 XLog.c 中添加 register_java_lang_XLog 方法来注册 XLog 的 jni 的方法。如下所示:

```
void register_java_lang_XLog(JNIEnv* env) {
  jniRegisterNativeMethods(env, "java/lang/XLog", gMethods, NELEM(gMethods));
}



```

### (2).OnLoad.cpp 中声明和调用 register_java_lang_XLog

OnLoad.cpp 文件路径如下:

```
libcore/ojluni/src/main/native/OnLoad.cpp


```

在 OnLoad.cpp 中添加 register_java_lang_XLog 方法的声明和调用。如下所示:

```
...省略
///ADD START
extern "C" void register_java_lang_XLog(JNIEnv* env);
///ADD END
...省略
extern "C" JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void*) {
  ...省略
  ///ADD START
  register_java_lang_XLog(env);
  ///ADD END
  ...省略
}
...省略


```

### （3）.Android.bp 中添加 XLog.c 文件到编译文件中

在文件 "/home/qiang/Desktop/test/sourcecode/libcore/ojluni/src/main/native/Android.bp" 中添加 XLog.c 到编译文件中。添加之后如下:

```
filegroup {
    name: "libopenjdk_native_srcs",
    srcs: [
         ...省略
        ///ADD START
        "XLog.c",
        ///ADD END
        ...省略
    ],
}



```

四、编译更新系统 api
============

由于向系统加了新的接口，需要执行如下命令更新系统 api:

```
source build/envsetup.sh
make update-api


```

五、测试验证
======

我在 java.io.File 构造函数中调用 XLog 打印文件。如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433xPAnEKibVHW6w6CcwReaicf2GZQWjExKGPkGdA8MsW6LyDZkKXjpSeQ6tw14ibY5xibFD82DKPUlzmA/640?wx_fmt=png)

  

编译系统刷机，终端查看日志:

```
C:\Users\Qiang>adb logcat  FILE:D *:s
--------- beginning of system
--------- beginning of main
01-20 19:28:24.077  1373  1518 D FILE    : File==>/system/app/SecurityManager/lib/arm64/wrap.sh
01-20 19:28:24.087   921   921 D FILE    : File==>/proc/self/task
01-20 19:28:24.173  4704  4704 D FILE    : File==>/system/etc/security/cacerts
01-20 19:28:24.173  4704  4704 D FILE    : File==>/data/misc/keychain
01-20 19:28:24.214  4704  4704 D FILE    : File==>/data/user/0/com.android.securitymanager
01-20 19:28:24.214  4704  4704 D FILE    : File==>/data/user_de/0/com.android.securitymanager
01-20 19:28:24.214  4704  4704 D FILE    : File==>/data/user/0/com.android.securitymanager
01-20 19:28:24.223  4704  4704 D FILE    : File==>/system/app/SecurityManager/SecurityManager.apk
01-20 19:28:24.223  4704  4704 D FILE    : File==>/system/app/SecurityManager/SecurityManager.apk
01-20 19:28:24.229  4704  4704 D FILE    : File==>/system/app/SecurityManager/lib/arm64
01-20 19:28:24.229  4704  4704 D FILE    : File==>/system/lib64
01-20 19:28:24.229  4704  4704 D FILE    : File==>/system/product/lib64
01-20 19:28:24.229  4704  4704 D FILE    : File==>/system/vendor/lib64
01-20 19:28:24.229  4704  4704 D FILE    : File==>/system/lib64
01-20 19:28:24.230  4704  4704 D FILE    : File==>/system/product/lib64


```

**完整修改的代码关注公众号之后发送消息 "****002****" 下载。**

上一篇[玩转安卓 10 源码开发定制 (18) 编译 Windows 平台 adb 和 fastboot 工具](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484205&idx=1&sn=3ab53e36478bdcd5ad7ed2b2bd88106c&scene=21#wechat_redirect)

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)

扫一扫关注公众号