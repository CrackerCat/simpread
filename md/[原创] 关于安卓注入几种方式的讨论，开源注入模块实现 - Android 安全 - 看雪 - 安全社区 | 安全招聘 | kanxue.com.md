> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283790.htm)

> [原创] 关于安卓注入几种方式的讨论，开源注入模块实现

概念
==

在 android 系统中，进程之间是**相互隔离**的，两个进程之间是没办法直接跨进程访问其他进程的空间信息的。那么在 android 平台中要对某个 app 进程进行内存操作，并获取目标进程的地址空间内信息或者修改目标进程的地址空间内的私有信息，就需要涉及到注入技术。

通过**注入技术可以将指定 so 模块或代码注入到目标进程**中，只要注入成功后，就可以进行访问和篡改目标进程空间内的信息，包括数据和代码。

写这篇文章的初衷：
=========

在我们进行算法还原再或者进行 APP 的 RPC 算法调用的时候，**都有对 APP 注入的需求**。

只不过目前的工具比较成熟，大家都**忽略了**注入的这个过程。

随着厂商对常见注入工具 Frida、Xposed 的**特征检测**，注入后 APP 会发生崩溃不可运行的问题。

众所周知：游戏安全对抗领域往往要比常见的应用安全领先多个领域，当然很**多大厂也开始上了策略检测注入**，但更多的只是风控策略，不会发生闪退（因为不同的厂商对于系统多少有些修改，万一有些系统会注入辅助 so，那么会造成很大的误伤）

应用场景：
=====

例如 BB 企业版、爱加密企业版、360 企业版都对 frida、xposed 等工具进行检测，那么我们就可以手动注入 dobby hook 以及支持 Java 的一些 sandhook 来辅助分析，当然分析效率没有 frida 高，但是不会触发闪退检测策略。（当然本工具后期有打算进一步开发隐藏注入，这对游戏安全是小儿科，但是应用安全隐藏的话效果还是很可观的）

所以本文章首先讨论多种注入方式，并给出开源的面具模块供大家编译使用，注入自己开发的 so，或者是调用成品库，进行 hook 以及高性能的 RPC。

本文会罗列出几个常见的注入技术，以及列出使用该原理的工具，并重点讲一下 zygote 注入的模块开发。

我会详细讲解我比较熟悉的两种注入方式（修改 aosp、zygisk），以及简单带过一些可能的注入方式，并后续补充注入材料。

常见的注入方式：
========

### **静态注入（重打包，需要过签名检测）**

静态注入，静态解析 ELF 文件，增加一个依赖 SO，或新增一个 section 节（注入代码在 section 字段），代码节是自己的注入代码，然后修复 ELF 文件结构。

修改 dex，增加静态 dex 段，system.load 加载自己的 so

实现案例：平头哥，一些虚拟 xposed 框架

```
static {
        try {
            String soName;
            if (Process.is64Bit()) {
                soName = path/to/lib64;
            } else {
                soName = path/to/lib32;
            }
            System.loadLibrary(soName);
        } catch (Throwable e) {
            CLog.e("static loadLibrary error", e);
        }
}
```

这种方式的优点：

免 root、便于分发、打包速度一般

缺点：

对于签名检测的 pass 难度比较高

**动态注入（基于系统提供的调试 API 注入）**
--------------------------

### ptrace 注入

** 由于 Android 是基于 linux 内核的操作系统，所以 Android 下的注入也是基于 Linux 下的系统调用函数`ptrace()`实现的。** 即在获得 root 权限后，通过 ptrace() 系统调用将 stub（桩代码）注入到指定 pid 的进程中。

常见使用工具：IDA、GDB、LLDB、Frida 等常见工具

我们也可以自己写一个 ptrace 简单的注入 so，下面我给出一个项目，感兴趣的大佬可以自己编译进行尝试。

这里进行**预告**：后面我会自己写一个调试器（基于 ptrace），会写出文章进行分享，目前已经在做了。

这里简单附上几篇 ptrace 的文章，感兴趣的大佬可以尝试。

因为我研究的实在是不多。

[https://blog.csdn.net/hp910315/article/details/77335058](https://blog.csdn.net/hp910315/article/details/77335058)

[https://blog.csdn.net/jinzhuojun/article/details/9900105](https://blog.csdn.net/jinzhuojun/article/details/9900105)

这种方式的优点：

注入速度快，注入不容易检测到（ptrace 注入完成以后直接取消 ptrace，在后面检测不到）

缺点：

需要 root、有一定的 ptrace 检测（像 ida 这样的注入，会在 maps 扫描到当前正在被调试）

attach 方式被 ptrace 占坑方式搞得不好绕过（ida 表示非常难受）。

### zygote 注入

常见使用工具：xposed 实现工具：Riru(早期)、Zygisk（常用）

zygote 注入是属于全局注入的方式，它主要是依赖于 fork() 子进程方式进行注入的。  
目前市面上比较成熟的注入工具 xposed 就是基于 zygote 的全局注入。

它有两大优点: 主要在于 zygote 是系统进程，通过系统进程 fork 出来后它就具备隐蔽性，强大性。

常见的一些工具都是使用 Zygisk 注入，比如知名的开源项目 [**Zygisk-Il2CppDumper**](https://github.com/Perfare/Zygisk-Il2CppDumper)

以及寒冰大佬开发的 FrdiaManager 还有 Xposed 框架都支持 Zygsik 注入

下面我来讲一下我开发的模块是如何注入自己的 so 的（本模块是基于 [**Zygisk-Il2CppDumper](https://github.com/Perfare/Zygisk-Il2CppDumper) 项目进行修改，因为作者写的 Gradle 实在是太好用啦）**

系统注入（修改 AOSP 源码，进行插桩）
---------------------

通过修改 aosp 系统的源码，在 app 加载之前插桩语句，加载自定义库。

后面会有一个小模块进行讨论。

Zygisk 自定义注入 so(dex) 插件的实现
==========================

模块开发前置知识
--------

```
class ModuleBase {
public:
 
    // 这个方法在模块被加载到目标进程时立即被调用。
    // 会传递一个 Zygisk API 句柄作为参数。
    virtual void onLoad([[maybe_unused]] Api *api, [[maybe_unused]] JNIEnv *env) {}
 
    // 这个方法在应用进程被专门化之前被调用。
    // 在这个时候，进程刚刚从 zygote 进程中分叉出来，但尚未应用任何特定于应用的专门化。
    // 这意味着进程没有任何沙箱限制，并且仍然以 zygote 的相同权限运行。
    //
    // 所有将要传递并用于应用程序专门化的参数都被封装在一个 AppSpecializeArgs 对象中。
    // 您可以读取和覆盖这些参数，以改变应用程序进程的专门化方式。
    //
    // 如果您需要以超级用户权限运行一些操作，可以调用 Api::connectCompanion() 来
    // 获取一个套接字，用于与根陪伴进程进行 IPC 调用。
    // 请参阅 Api::connectCompanion() 以获取更多信息。
    virtual void preAppSpecialize([[maybe_unused]] AppSpecializeArgs *args) {}
 
    // 这个方法在应用进程专门化之后被调用。
    // 在这个时候，进程已经应用了所有沙箱限制，并以应用自身代码的权限运行。
    virtual void postAppSpecialize([[maybe_unused]] const AppSpecializeArgs *args) {}
 
    // 这个方法在系统服务器进程被专门化之前被调用。
    // 请参阅 preAppSpecialize(args) 以获取更多信息。
    virtual void preServerSpecialize([[maybe_unused]] ServerSpecializeArgs *args) {}
 
    // 这个方法在系统服务器进程专门化之后被调用。
    // 在这个时候，进程以 system_server 的权限运行。
    virtual void postServerSpecialize([[maybe_unused]] const ServerSpecializeArgs *args) {}
};
```

重点就是这几个 api, 看注释理解  
用最通俗粗略的理解来表示的话:  
pre 是刚从 zygote fork 出来没有沙箱限制的时候  
postAppSpecialize 相当于 app 进程启动, 这里可以做自定义 dex 加载的一些动作  
postServerSpecialize 相当于系统服务也就是 system server 运行

官方提供了一个 [https://github.com/topjohnwu/zygisk-module-sample 案例，也可以读一读](https://github.com/topjohnwu/zygisk-module-sample%E6%A1%88%E4%BE%8B%EF%BC%8C%E4%B9%9F%E5%8F%AF%E4%BB%A5%E8%AF%BB%E4%B8%80%E8%AF%BB)

模块实现细节：
-------

实现原理非常简单：

从 app 可以访问的路径 copy 要注入的 so 到自己的私有目录 (因为有 selinux 的限制）

之后使用 dl_open 加载目标 so

```
#include #include #include #include #include #include #include #include #include "hack.h"
#include "zygisk.hpp"
#include "game.h"
#include "log.h"
#include "dlfcn.h"
using zygisk::Api;
using zygisk::AppSpecializeArgs;
using zygisk::ServerSpecializeArgs;
 
class MyModule : public zygisk::ModuleBase {
public:
    void onLoad(Api *api, JNIEnv *env) override {
        this->api = api;
        this->env = env;
    }
 
    void preAppSpecialize(AppSpecializeArgs *args) override {
        auto package_name = env->GetStringUTFChars(args->nice_name, nullptr);
        auto app_data_dir = env->GetStringUTFChars(args->app_data_dir, nullptr);
        LOGI("preAppSpecialize %s %s", package_name, app_data_dir);
        preSpecialize(package_name, app_data_dir);
        env->ReleaseStringUTFChars(args->nice_name, package_name);
        env->ReleaseStringUTFChars(args->app_data_dir, app_data_dir);
    }
 
    void postAppSpecialize(const AppSpecializeArgs *) override {
        if (enable_hack) {
            std::thread hack_thread(hack_prepare, _data_dir, data, length);
            hack_thread.detach();
        }
    }
 
private:
    Api *api;
    JNIEnv *env;
    bool enable_hack;
    char *_data_dir;
    void *data;
    size_t length;
 
    void preSpecialize(const char *package_name, const char *app_data_dir) {
        if (strcmp(package_name, AimPackageName) == 0) {
            LOGI("成功注入目标进程: %s", package_name);
            enable_hack = true;
            _data_dir = new char[strlen(app_data_dir) + 1];
            strcpy(_data_dir, app_data_dir);
 
#if defined(__i386__)
            auto path = "zygisk/armeabi-v7a.so";
#endif
#if defined(__x86_64__)
            auto path = "zygisk/arm64-v8a.so";
#endif
#if defined(__i386__) || defined(__x86_64__)
            int dirfd = api->getModuleDir();
            int fd = openat(dirfd, path, O_RDONLY);
            if (fd != -1) {
                struct stat sb{};
                fstat(fd, &sb);
                length = sb.st_size;
                data = mmap(nullptr, length, PROT_READ, MAP_PRIVATE, fd, 0);
                close(fd);
            } else {
                LOGW("Unable to open arm file");
            }
#endif
        } else {
            api->setOption(zygisk::Option::DLCLOSE_MODULE_LIBRARY);
        }
    }
};
 
REGISTER_ZYGISK_MODULE(MyModule) 
```

这里主要实现了面具模块主要提供的 api

```
void preAppSpecialize(AppSpecializeArgs *args) override {
       auto package_name = env->GetStringUTFChars(args->nice_name, nullptr);
       auto app_data_dir = env->GetStringUTFChars(args->app_data_dir, nullptr);
       LOGI("preAppSpecialize %s %s", package_name, app_data_dir);
       preSpecialize(package_name, app_data_dir);
       env->ReleaseStringUTFChars(args->nice_name, package_name);
       env->ReleaseStringUTFChars(args->app_data_dir, app_data_dir);
   }
```

我们的实现主要是这个实现的函数，此时 app 已经处于沙盒中了，只有 app 自身的权限

```
if (strcmp(package_name, AimPackageName) == 0) {
           LOGI("成功注入目标进程: %s", package_name);
           enable_hack = true;
           _data_dir = new char[strlen(app_data_dir) + 1];
           strcpy(_data_dir, app_data_dir);
```

在这里我们需要修改要注入的包名，不然模块不会进一步注入

![](https://bbs.kanxue.com/upload/attach/202410/967562_KSASC9H7HSXW2JW.png)

主要功能实现

```
void hack_start(const char *game_data_dir,JavaVM *vm) {
    bool load = false;
    LOGI("hack_start %s", game_data_dir);
    // 构建新文件路径
    char new_so_path[256];
    snprintf(new_so_path, sizeof(new_so_path), "%s/files/%s.so", game_data_dir, "test");
 
    // 复制 /sdcard/test.so 到 game_data_dir 并重命名
    const char *src_path = "/data/local/tmp/test.so";
    int src_fd = open(src_path, O_RDONLY);
    if (src_fd < 0) {
        LOGE("Failed to open %s: %s (errno: %d)", src_path, strerror(errno), errno);
        return;
    }
 
    int dest_fd = open(new_so_path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (dest_fd < 0) {
        LOGE("Failed to open %s", new_so_path);
        close(src_fd);
        return;
    }
    // 复制文件内容
    char buffer[4096];
    ssize_t bytes;
    while ((bytes = read(src_fd, buffer, sizeof(buffer))) > 0) {
        if (write(dest_fd, buffer, bytes) != bytes) {
            LOGE("Failed to write to %s", new_so_path);
            close(src_fd);
            close(dest_fd);
            return;
        }
    }
 
    close(src_fd);
    close(dest_fd);
    if (chmod(new_so_path, 0755) != 0) {
        LOGE("Failed to change permissions on %s: %s (errno: %d)", new_so_path, strerror(errno), errno);
        return;
    } else {
        LOGI("Successfully changed permissions to 755 on %s", new_so_path);
    }
    void * handle;
    // 使用 xdl_open 打开新复制的 so 文件
    for (int i = 0; i < 10; i++) {
//        void *handle = xdl_open(new_so_path, 0);
        handle = dlopen(new_so_path, RTLD_NOW | RTLD_LOCAL);
        if (handle) {
            LOGI("Successfully loaded %s", new_so_path);
            load = true;
            break;
        } else {
            LOGE("Failed to load %s: %s", new_so_path, dlerror());
            sleep(1);
        }
    }
    if (!load) {
        LOGI("test.so not found in thread %d", gettid());
    }
    void (*JNI_OnLoad)(JavaVM *, void *);
    *(void **) (&JNI_OnLoad) = dlsym(handle, "JNI_OnLoad");
    if (JNI_OnLoad) {
        LOGI("JNI_OnLoad symbol found, calling JNI_OnLoad.");
        JNI_OnLoad(vm, NULL);
    } else {
        LOGE("JNI_OnLoad symbol not found in %s", new_so_path);
    }
 
}
```

复制过程，主要就是把 / data/local/tmp/test.so 复制到私有目录，然后修改权限为 0755 不然 dlopen 没法加载

```
char new_so_path[256];
    snprintf(new_so_path, sizeof(new_so_path), "%s/files/%s.so", game_data_dir, "test");
 
    // 复制 /sdcard/test.so 到 game_data_dir 并重命名
    const char *src_path = "/data/local/tmp/test.so";
    int src_fd = open(src_path, O_RDONLY);
    if (src_fd < 0) {
        LOGE("Failed to open %s: %s (errno: %d)", src_path, strerror(errno), errno);
        return;
    }
 
    int dest_fd = open(new_so_path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (dest_fd < 0) {
        LOGE("Failed to open %s", new_so_path);
        close(src_fd);
        return;
    }
    // 复制文件内容
    char buffer[4096];
    ssize_t bytes;
    while ((bytes = read(src_fd, buffer, sizeof(buffer))) > 0) {
        if (write(dest_fd, buffer, bytes) != bytes) {
            LOGE("Failed to write to %s", new_so_path);
            close(src_fd);
            close(dest_fd);
            return;
        }
    }
 
    close(src_fd);
    close(dest_fd);
    if (chmod(new_so_path, 0755) != 0) {
        LOGE("Failed to change permissions on %s: %s (errno: %d)", new_so_path, strerror(errno), errno);
        return;
    } else {
        LOGI("Successfully changed permissions to 755 on %s", new_so_path);
    }
```

尝试打开十次，获取到 so 的 handle

```
for (int i = 0; i < 10; i++) {
//        void *handle = xdl_open(new_so_path, 0);
        handle = dlopen(new_so_path, RTLD_NOW | RTLD_LOCAL);
        if (handle) {
            LOGI("Successfully loaded %s", new_so_path);
            load = true;
            break;
        } else {
            LOGE("Failed to load %s: %s", new_so_path, dlerror());
            sleep(1);
        }
    }
```

寻找符号并执行

```
void (*JNI_OnLoad)(JavaVM *, void *);
   *(void **) (&JNI_OnLoad) = dlsym(handle, "JNI_OnLoad");
   if (JNI_OnLoad) {
       LOGI("JNI_OnLoad symbol found, calling JNI_OnLoad.");
       JNI_OnLoad(vm, NULL);
   } else {
       LOGE("JNI_OnLoad symbol not found in %s", new_so_path);
   }
```

可以自己写一个自己的函数在用 dlsym 调用，这里就不多说了

源码导入 android studio 就可以构建出面具模块了，再次感谢原作者的项目

![](https://bbs.kanxue.com/upload/attach/202410/967562_7QVDDB9BJXMFUBD.png)

使用方法：
=====

在编译之前应该修改目标 app 的包名，如果不修改不会注入（后面会考虑做一个和 shamiko 一样的黑白名单）初代版本大家先手动修改

编译自己的插件 so 实现自己的功能

这里需要了解的是 dlopen 的加载流程。见番外篇。

当然可以修改插件，使用 dlsym 找到自己函数的符号，手动加载。

我已经实现了 JNI_ONLOAD 的加载。

移动 so 到 / data/local/tmp 目录下 命名为 test.so

享受注入！

插件 so 源码：

![](https://bbs.kanxue.com/upload/attach/202410/967562_NC3KMKMQUQ6U4TV.png)

```
__attribute__((constructor))
void my_init_function() {
    std::string hello = "我来自其他模块";
    __android_log_print(6, "jiqiu2021", "%s", hello.c_str());
}
```

注入效果：

![](https://bbs.kanxue.com/upload/attach/202410/967562_SCJE7MBX42BD2XW.png)

番外一: dlopen 的简单解析
=================

[http://aospxref.com/android-12.0.0_r3/xref/bionic/libdl/libdl.cpp](http://aospxref.com/android-12.0.0_r3/xref/bionic/libdl/libdl.cpp)

![](https://bbs.kanxue.com/upload/attach/202410/967562_7U7V28UD539J4SQ.png)

之后调用

![](https://bbs.kanxue.com/upload/attach/202410/967562_VUQFSRY3AGE8KTM.png)

之后调用

![](https://bbs.kanxue.com/upload/attach/202410/967562_SG35QTSTWUQ2DUM.png)

来到真正的 dlopen 加载的地方

```
void* do_dlopen(const char* name, int flags,
2064                  const android_dlextinfo* extinfo,
2065                  const void* caller_addr) {
2066    std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
2067    ScopedTrace trace(trace_prefix.c_str());
2068    ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
2069    soinfo* const caller = find_containing_library(caller_addr);
2070    android_namespace_t* ns = get_caller_namespace(caller);
2071 
2072    LD_LOG(kLogDlopen,
2073           "dlopen(name=\"%s\", flags=0x%x, extinfo=%s, caller=\"%s\", caller_ns=%s@%p, targetSdkVersion=%i) ...",
2074           name,
2075           flags,
2076           android_dlextinfo_to_string(extinfo).c_str(),
2077           caller == nullptr ? "(null)" : caller->get_realpath(),
2078           ns == nullptr ? "(null)" : ns->get_name(),
2079           ns,
2080           get_application_target_sdk_version());
2081 
2082    auto purge_guard = android::base::make_scope_guard([&]() { purge_unused_memory(); });
2083 
2084    auto failure_guard = android::base::make_scope_guard(
2085        [&]() { LD_LOG(kLogDlopen, "... dlopen failed: %s", linker_get_error_buffer()); });
2086 
2087    if ((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL|RTLD_NODELETE|RTLD_NOLOAD)) != 0) {
2088      DL_OPEN_ERR("invalid flags to dlopen: %x", flags);
2089      return nullptr;
2090    }
2091 
2092    if (extinfo != nullptr) {
2093      if ((extinfo->flags & ~(ANDROID_DLEXT_VALID_FLAG_BITS)) != 0) {
2094        DL_OPEN_ERR("invalid extended flags to android_dlopen_ext: 0x%" PRIx64, extinfo->flags);
2095        return nullptr;
2096      }
2097 
2098      if ((extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD) == 0 &&
2099          (extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET) != 0) {
2100        DL_OPEN_ERR("invalid extended flag combination (ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET without "
2101            "ANDROID_DLEXT_USE_LIBRARY_FD): 0x%" PRIx64, extinfo->flags);
2102        return nullptr;
2103      }
2104 
2105      if ((extinfo->flags & ANDROID_DLEXT_USE_NAMESPACE) != 0) {
2106        if (extinfo->library_namespace == nullptr) {
2107          DL_OPEN_ERR("ANDROID_DLEXT_USE_NAMESPACE is set but extinfo->library_namespace is null");
2108          return nullptr;
2109        }
2110        ns = extinfo->library_namespace;
2111      }
2112    }
2113 
2114    // Workaround for dlopen(/system/lib/) when .so is in /apex. http://b/121248172
2115    // The workaround works only when targetSdkVersion < Q.
2116    std::string name_to_apex;
2117    if (translateSystemPathToApexPath(name, &name_to_apex)) {
2118      const char* new_name = name_to_apex.c_str();
2119      LD_LOG(kLogDlopen, "dlopen considering translation from %s to APEX path %s",
2120             name,
2121             new_name);
2122      // Some APEXs could be optionally disabled. Only translate the path
2123      // when the old file is absent and the new file exists.
2124      // TODO(b/124218500): Re-enable it once app compat issue is resolved
2125      /*
2126      if (file_exists(name)) {
2127        LD_LOG(kLogDlopen, "dlopen %s exists, not translating", name);
2128      } else
2129      */
2130      if (!file_exists(new_name)) {
2131        LD_LOG(kLogDlopen, "dlopen %s does not exist, not translating",
2132               new_name);
2133      } else {
2134        LD_LOG(kLogDlopen, "dlopen translation accepted: using %s", new_name);
2135        name = new_name;
2136      }
2137    }
2138    // End Workaround for dlopen(/system/lib/) when .so is in /apex.
2139 
2140    std::string asan_name_holder;
2141 
2142    const char* translated_name = name;
2143    if (g_is_asan && translated_name != nullptr && translated_name[0] == '/') {
2144      char original_path[PATH_MAX];
2145      if (realpath(name, original_path) != nullptr) {
2146        asan_name_holder = std::string(kAsanLibDirPrefix) + original_path;
2147        if (file_exists(asan_name_holder.c_str())) {
2148          soinfo* si = nullptr;
2149          if (find_loaded_library_by_realpath(ns, original_path, true, &si)) {
2150            PRINT("linker_asan dlopen NOT translating \"%s\" -> \"%s\": library already loaded", name,
2151                  asan_name_holder.c_str());
2152          } else {
2153            PRINT("linker_asan dlopen translating \"%s\" -> \"%s\"", name, translated_name);
2154            translated_name = asan_name_holder.c_str();
2155          }
2156        }
2157      }
2158    }
2159 
2160    ProtectedDataGuard guard;
2161    soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);
2162    loading_trace.End();
2163 
2164    if (si != nullptr) {
2165      void* handle = si->to_handle();
2166      LD_LOG(kLogDlopen,
2167             "... dlopen calling constructors: realpath=\"%s\", soname=\"%s\", handle=%p",
2168             si->get_realpath(), si->get_soname(), handle);
2169      si->call_constructors();
2170      failure_guard.Disable();
2171      LD_LOG(kLogDlopen,
2172             "... dlopen successful: realpath=\"%s\", soname=\"%s\", handle=%p",
2173             si->get_realpath(), si->get_soname(), handle);
2174      return handle;
2175    }
2176 
2177    return nullptr;
2178  } 
```

这里就是对 so 的各个段的装载，我们目光聚焦于结尾的部分

![](https://bbs.kanxue.com/upload/attach/202410/967562_5CKG7XNN5PA7DYK.png)

在这个函数里有:

```
// DT_INIT should be called before DT_INIT_ARRAY if both are present.
547    call_function("DT_INIT", init_func_, get_realpath());
548    call_array("DT_INIT_ARRAY", init_array_, init_array_count_, false, get_realpath());
549 
```

对 DT_INIT 和 DT_INIT_ARRAY 的调用

所以我们 dlopen 如果成功打开了 so，就会对这两个地方调用

所以说插件的入口可以选择在这两个段里 **attribute**((constructor))

```
**__attribute__((constructor))**
void my_init_function() {
    std::string hello = "我来自其他模块";
    __android_log_print(6, "jiqiu2021", "%s", hello.c_str());
}
```

番外二：定植 AOSP 进行 so 的注入
=====================

这里我通过修改源码去注入 so，so 注入的时机我开始的选择是越早越好。

这里选在在 handleBindApplication 处，创建 ContextImpl 对象时进行一系列的复制注入操作。

我们流程选择先将需要注入的 so 放到 sd 卡目录下，然后判断 app 为非系统 app 时进行复制到 app 目录，注入 app 等一系列操作。 我们找到源码，目录 AOSP/frameworks/base/core/java/android/app/ActivityThread.java，

找到 handleBindApplication，定位到”final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);” 这一行。

![](https://bbs.kanxue.com/upload/attach/202410/967562_8M3USU5KAH3522C.png)

开始加入我们自己的代码：

和上面的实现一样，就是 copyso 然后使用 system.load 即可加载

网上有很多实现，还可以自定义 selinux 标签，配合系统服务和配套 app 达到自定义注入

TODO：实战：使用自己写的 hook 工具分析强检测 frida 的 APP
=======================================

如果文章反响还不错，我会继续更新一些 frida、xposed 分析不了的 app（被反调试 block 掉的）

来进一步加深大家对这个框架的使用。

未来的框架的更新方向：
===========

增加第二种注入方式：将插件 so 打包到框架里，隐藏落地文件的特征。

通过借鉴 Riru 的注入方式，隐藏注入（对一些厂商管用）

进一步研究完美隐藏方式

项目地址：  
[https://github.com/jiqiu2022/Zygisk-MyInjector](https://github.com/jiqiu2022/Zygisk-MyInjector)  
附件注入的包名已经固定，需要自己编译

[[课程]Linux pwn 探索篇！](https://www.kanxue.com/book-section_list-172.htm)

最后于 4 天前 被 mb_qzwrkwda 编辑 ，原因： 整理排版

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm)

上传的附件：

*   [zygisk-myinjector-v0.01-debug.zip](javascript:void(0);) （493.43kb，17 次下载）