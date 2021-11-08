> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270168.htm)

> [原创] 小菜花的 Riru 原理浅析和源码阅读

一，引言
----

因为最近在研究在 svc bypass，frida 脚本就随便写了  
落地到生产环境，首先用的是 xposed + inlinehook，就是用 xposed hook runtime 的 loadLibrary，当发现目标 so load 的时候，就开始主动 load 我们的 inlinehook so，inlinehook so 中的逻辑就是，对目标 so 进行 svc 指令内存扫描，然后对扫描到的 svc 地址进行 hook 操作

那么问题来了，这个 inlinehook so 的 load 时机如何选：  
1，after loadLibrary(目标 so)，这个时机的话，目标 so 在 dlopen 中都执行完 init 和 init_arrary 了，如果检测放在 init 中刚好人家检测完，你才注入 inlinehook so，那时机就太晚了。

2，before loadLibrary(目标 so)，这个时机的话，你 inlinehook so 工作原理就是需要在内存中扫描目标 so 然后再 hook，此时目标 so 还没加载，根本拿不到地址没发扫描啊，所以这个时机也不行。（ps：或许主动去 getSymFromLinker(find_libraries)，然后主动调用 find_libraries 去加载链接目标 so？这个思路是写文章的时候想到的，未进行尝试，因为就想玩 riru，手动狗头（ps：find_libraries 是笔者阅读源码 android8.1 选取的装载了 so 却没执行 init 函数的一个时机））

综上所述，两个时机都不太行

那么来看 [https://bbs.pediy.com/thread-268256.htm](https://bbs.pediy.com/thread-268256.htm "小菜花的frida-gadget持久化方案汇总") ，该文章选取的注入 frida-gadget 的时机点是 com_android_internal_os_Zygote_nativeForkAndSpecialize，该函数是 fork 应用程序进程的，这个时机就比较好，fork 好应用程序进程，咱们就开始 hook linker 的 find_libraries，等目标 so 加载链接好之后，就开始扫描内存，对 svc address 进行 hook 操作

一切就是这么的顺其自然，嗯，大致就这个思路吧。

要对 nativeForkAndSpecialize 进行动手，首先应该想到的是 magisk 的 riru 模块了，riru 模块提供了下面三个函数的注入时机

*   nativeForkAndSpecialize
*   nativeSpecializeAppProcess
*   nativeForkSystemServer

提供了 pre 和 post 函数，就可以随便玩了。那么开始动手搞 riru 模块开发，本着多多学习的原则，来看下 riru 源码和原理，

看源码是为了更好的 cv，看原理是为了知其所以然，然后达到一个理所当然的状态。

二，Magisk
--------

看 riru 之前，肯定要看下 Magisk 了，因为 riru 属于 magisk 模块，需要大致知道 magisk 是咋回事。

先来看下 magisk 官方文档：[https://topjohnwu.github.io/Magisk/guides.html![](https://bbs.pediy.com/upload/attach/202111/844301_D2A23NNQEJH6C6T.jpg)](https://topjohnwu.github.io/Magisk/guides.html)

![](https://bbs.pediy.com/upload/attach/202111/844301_4P3462UDKQEPUTH.jpg)

关键点看下来就是：  

1，magisk 会在两个时机执行 post-fs-data.sh 和 service.sh

2,  system.prop 会被 magisk 加载作为系统属性

3，没特殊设置的话，magisk 会挂载模块内的 system 文件夹，这里是存放 replace/inject files 的

三，Riru
------

### Riru 原理  

老规矩，先来 git 看下 readme: [https://github.com/RikkaApps/Riru](https://github.com/RikkaApps/Riru)

![](https://bbs.pediy.com/upload/attach/202111/844301_V685RMX3KKMVJ6Y.jpg)

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)其实这里就已经把 riru 原理介绍完了，大致总结下就是：

1，注入原理：动了 ro.dalvik.vm.native.bridge，可以被系统自动 dlopen 特定 so 文件

2，hook 原理：hook 了 android_runtime 的 jniRegisterNativeMethods 对关键 jni 函数做了替换（ps：系统启动的时候先启动 init 进程，init 进程再启动 zygote 进程，而 zygote 进程会创建 java 虚拟机，并为其注册 jni 方法）

### 注入原理

那么来详细看下注入原理的由来：[https://blog.canyie.top/2020/08/18/nbinjection/](https://blog.canyie.top/2020/08/18/nbinjection/ "通过系统的native bridge实现注入zygote")，大致总结下就是：

1，作者分析源码的时候发现，在 startVM 函数流程中 Runtime::Init 会加载 native bridge

2，LoadNativeBridge 会 dlopen 了一个特定的 so

3，这个特定的 so 的由来是系统属性 ro.dalvik.vm.native.bridge 的值

4，有特殊情况 native bridge 的 so 会被系统卸载掉，所以直接当个 loader 去 load 其他的 so 比较好

### riru zip 包分析

从 git 上下载 riru release 包解压下来看看

![](https://bbs.pediy.com/upload/attach/202111/844301_3KG5UKUU9PZGBTE.jpg)  

这几个脚本文件相对生疏一点，回过头来看下 magisk 文档

![](https://bbs.pediy.com/upload/attach/202111/844301_N2M97SPRRFMF6EQ.jpg)  

再打开这些脚本文件看下大致内容：

util_functions.sh 是搞了一些 funtion ui_print

verify.sh 是校验文件完整性的

update-binary 和 customize.sh 都是做一些安装过程准备的，涉及一些检查环境，解压，删除文件啥的

![](https://bbs.pediy.com/upload/attach/202111/844301_A5A7UYMCZASPW5X.jpg)  

稍微关键点的是，搞了个文件夹，移动了一下文件，记住这个 $(magisk --path)/.magisk/modules/riru-core 目录，来设备中看下大致长啥样

![](https://bbs.pediy.com/upload/attach/202111/844301_WE6TTTJMKVARWQT.jpg)

这下清晰多了，文件结构就和上文 Magisk Modules 介绍的一摸一样，那么根据上文介绍的 magisk 情况，可以明确的是：  

1，system.prop 会被 magisk 加载作为系统属性，当前 system.prop 的内容为

ro.dalvik.vm.native.bridge=libriruloader.so，嘿，这不注入原理和上面说的对应上了。

2，magisk 会在两个时机执行 post-fs-data.sh 和 service.sh，打开看下这两个文件有没有啥关键信息

![](https://bbs.pediy.com/upload/attach/202111/844301_6DJ64NSVRZ742AZ.jpg)  

![](https://bbs.pediy.com/upload/attach/202111/844301_584C45W9BCMMCAX.jpg)  

稍微关键点的好像就这句了：

```
flock "module.prop"
unshare -m sh -c "/system/bin/app_process -Djava.class.path=rirud.apk /system/bin --nice-

```

那就来源码 rirud riru.Daemon 看下干了嘛

![](https://bbs.pediy.com/upload/attach/202111/844301_SQ23R5BZEE6NHVN.jpg)  

```
private void onRiruLoad() {
    allowRestart = true;
    Log.i(TAG, "Riru loaded, reset native bridge to " + DaemonUtils.getOriginalNativeBridge() + "...");
    DaemonUtils.resetNativeBridgeProp(DaemonUtils.getOriginalNativeBridge());
 
    Log.i(TAG, "Riru loaded, stop rirud socket...");
    serverThread.stopServer();
 
    var loadedModules = DaemonUtils.getLoadedModules().toArray();
 
    StringBuilder sb = new StringBuilder();
    if (loadedModules.length == 0) {
        sb.append(DaemonUtils.res.getString(R.string.empty));
    } else {
        sb.append(loadedModules[0]);
        for (int i = 1; i < loadedModules.length; ++i) {
            sb.append(", ");
            sb.append(loadedModules[i]);
        }
    }
 
    if (DaemonUtils.hasIncorrectFileContext()) {
        DaemonUtils.writeStatus(R.string.bad_file_context_loaded, loadedModules.length, sb);
    } else {
        DaemonUtils.writeStatus(R.string.loaded, loadedModules.length, sb);
    }
}

```

大致看了下，好像就是搞了个 socket 暴露了一些 api 用来进程通信了然后还有一些环境操作，比如 riru load 之后把 native bridge 设置会原来的，那就大致先这样吧。

### riru loader 源码分析

回过头来看 system.prop 的 native bridge 的 libriruloader.so，根据 riru/src/main/cpp/CMakeLists.txt 可知，其入口为 riru/src/main/cpp/loader/loader.cpp，关键代码如下

```
__used __attribute__((constructor)) void Constructor() {
    if (getuid() != 0) {
        return;
    }
 
    std::string_view cmdline = getprogname();
 
    if (cmdline != "zygote" &&
        cmdline != "zygote32" &&
        cmdline != "zygote64" &&
        cmdline != "usap32" &&
        cmdline != "usap64") {
        LOGW("not zygote (cmdline=%s)", cmdline.data());
        return;
    }
 
    LOGI("Riru %s (%d) in %s", riru::versionName, riru::versionCode, cmdline.data());
    LOGI("Android %s (api %d, preview_api %d)", android_prop::GetRelease(),
         android_prop::GetApiLevel(),
         android_prop::GetPreviewApiLevel());
 
    constexpr auto retries = 5U;
    RirudSocket rirud{retries};
 
    if (!rirud.valid()) {
        LOGE("rirud connect fails");
        return;
    }
 
    std::string magisk_path = rirud.ReadMagiskTmpfsPath();
    if (magisk_path.empty()) {
        LOGE("failed to obtain magisk path");
        return;
    }
 
    BuffString riru_path;
    riru_path += magisk_path;
    riru_path += "/.magisk/modules/riru-core/lib";
#ifdef __LP64__    riru_path += "64";#endif    riru_path += "/libriru.so";
 
    auto *handle = DlopenExt(riru_path, 0);
    if (handle) {
        auto init = reinterpret_cast(dlsym(
                handle, "init"));
        if (init) {
            init(handle, magisk_path.data(), rirud);
        } else {
            LOGE("dlsym init %s", dlerror());
        }
    } else {
        LOGE("dlopen riru.so %s", dlerror());
    }
 
#ifdef HAS_NATIVE_BRIDGE    auto native_bridge = rirud.ReadNativeBridge();
    if (native_bridge.empty()) {
        LOGW("Failed to read original native bridge from socket");
        return;
    }
 
    LOGI("original native bridge: %s", native_bridge.data());
 
    if (native_bridge == "0") {
        return;
    }
 
    original_bridge = dlopen(native_bridge.data(), RTLD_NOW);
    if (original_bridge == nullptr) {
        LOGE("dlopen failed: %s", dlerror());
        return;
    }
 
    auto *original_native_bridge_itf = dlsym(original_bridge, "NativeBridgeItf");
    if (original_native_bridge_itf == nullptr) {
        LOGE("dlsym failed: %s", dlerror());
        return;
    }
 
    int sdk = 0;
    std::array value;
    if (__system_property_get("ro.build.version.sdk", value.data()) > 0) {
        sdk = atoi(value.data());
    }
 
    auto callbacks_size = 0;
    if (sdk >= __ANDROID_API_R__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_R__>);
    } else if (sdk == __ANDROID_API_Q__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_Q__>);
    } else if (sdk == __ANDROID_API_P__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_P__>);
    } else if (sdk == __ANDROID_API_O_MR1__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_O_MR1__>);
    } else if (sdk == __ANDROID_API_O__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_O__>);
    } else if (sdk == __ANDROID_API_N_MR1__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_N_MR1__>);
    } else if (sdk == __ANDROID_API_N__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_N__>);
    } else if (sdk == __ANDROID_API_M__) {
        callbacks_size = sizeof(NativeBridgeCallbacks<__ANDROID_API_M__>);
    }
 
    memcpy(NativeBridgeItf, original_native_bridge_itf, callbacks_size);
#endif} 
```

大致总结下就是：

1，从 magisk_path/.magisk/modules/riru-core/lib 下 dlopen libriru.so，如果有 init sym 的话执行之

2，如果原来有 native bridge 的话就和 rirud 进程通信获取一下，然后仿照源码逻辑操作一波

### riru.so 源码分析

接下来看 libriru.so, 根据 riru/src/main/cpp/CMakeLists.txt 可知，其入口为 riru/src/main/cpp/entry.cpp，关键代码如下

```
extern "C" [[gnu::visibility("default")]] [[maybe_unused]] void// NOLINTNEXTLINEinit(void *handle, const char* magisk_path, const RirudSocket& rirud) {
    self_handle = handle;
 
    magisk::SetPath(magisk_path);
    hide::PrepareMapsHideLibrary();
    jni::InstallHooks();
    modules::Load(rirud);
}

```

正好和上面 libriruloader.so 中的操作对应，执行 libriru.so 的 init 函数

#### hide::PrepareMapsHideLibrary

先来看 hide::PrepareMapsHideLibrary(), 就是从 libriruhide.so 获取了下 riru_hide_func，先记住这个 riru_hide_func，下面会用到

```
void PrepareMapsHideLibrary() {
    auto hide_lib_path = magisk::GetPathForSelfLib("libriruhide.so");
 
    // load riruhide.so and run the hide    LOGD("dlopen libriruhide");
    riru_hide_handle = DlopenExt(hide_lib_path.c_str(), 0);
    if (!riru_hide_handle) {
        LOGE("dlopen %s failed: %s", hide_lib_path.c_str(), dlerror());
        return;
    }
    riru_hide_func = reinterpret_cast(dlsym(riru_hide_handle, "riru_hide"));
    if (!riru_hide_func) {
        LOGE("dlsym failed: %s", dlerror());
        dlclose(riru_hide_handle);
        return;
    }
} 
```

#### jni::InstallHooks()

接着看 jni::InstallHooks()

```
void jni::InstallHooks() {
    XHOOK_REGISTER(".*\\libandroid_runtime.so$", jniRegisterNativeMethods)
 
    if (xhook_refresh(0) == 0) {
        xhook_clear();
        LOGI("hook installed");
    } else {
        LOGE("failed to refresh hook");
    }
    //  省略多行代码
}

```

来看 XHOOK_REGISTER 这个宏定义

```
#define XHOOK_REGISTER(PATH_REGEX, NAME) \
    if (xhook_register(PATH_REGEX, #NAME, (void*) new_##NAME, (void **) &old_##NAME) != 0) \
        LOGE("failed to register hook " #NAME "."); \

```

再来看个宏和函数

```
#define NEW_FUNC_DEF(ret, func, ...) \
    using func##_t = ret(__VA_ARGS__); \
    static func##_t *old_##func; \
    static ret new_##func(__VA_ARGS__)
 
NEW_FUNC_DEF(int, jniRegisterNativeMethods, JNIEnv *env, const char *className,
             const JNINativeMethod *methods, int numMethods) {
    LOGD("jniRegisterNativeMethods %s", className);
 
    auto newMethods = handleRegisterNative(className, methods, numMethods);
    int res = old_jniRegisterNativeMethods(env, className, newMethods ? newMethods.get() : methods,
                                           numMethods);
    /*if (!newMethods) {        NativeMethod::jniRegisterNativeMethodsPost(env, className, methods, numMethods);    }*/    return res;
}

```

这样一来，new_jniRegisterNativeMethods 和 old_jniRegisterNativeMethods 就有了（ps：和 va 核心 io 那里的写法很像吧，大佬们优雅的写法如出一辙）

该来看关键点 handleRegisterNative 了

![](https://bbs.pediy.com/upload/attach/202111/844301_BN7C4Q49Z4XWMC4.jpg)  

适配下版本，比较下 method 的 name 和 signature，替换下函数指针，hook 原理就来了，点开看下 nativeForkAndSpecialize_r 代码如下：

```
jint nativeForkAndSpecialize_r(
        JNIEnv *env, jclass clazz, jint uid, jint gid, jintArray gids, jint runtime_flags,
        jobjectArray rlimits, jint mount_external, jstring se_info, jstring se_name,
        jintArray fdsToClose, jintArray fdsToIgnore, jboolean is_child_zygote,
        jstring instructionSet, jstring appDataDir, jboolean isTopApp, jobjectArray pkgDataInfoList,
        jobjectArray whitelistedDataInfoList, jboolean bindMountAppDataDirs,
        jboolean bindMountAppStorageDirs) {
 
    nativeForkAndSpecialize_pre(env, clazz, uid, gid, gids, runtime_flags, rlimits, mount_external,
                                se_info, se_name, fdsToClose, fdsToIgnore, is_child_zygote,
                                instructionSet, appDataDir, isTopApp, pkgDataInfoList,
                                whitelistedDataInfoList,
                                bindMountAppDataDirs, bindMountAppStorageDirs);
 
    jint res = ((nativeForkAndSpecialize_r_t *) jni::zygote::nativeForkAndSpecialize->fnPtr)(
            env, clazz, uid, gid, gids, runtime_flags, rlimits, mount_external, se_info, se_name,
            fdsToClose, fdsToIgnore, is_child_zygote, instructionSet, appDataDir, isTopApp,
            pkgDataInfoList,
            whitelistedDataInfoList, bindMountAppDataDirs, bindMountAppStorageDirs);
 
    nativeForkAndSpecialize_post(env, clazz, uid, is_child_zygote, res);
    return res;
}

```

这就是 pre 和 post 函数的由来了。

替换了目标函数的函数指针后，返回新的 JNINativeMethod[]，再由真正的 old_jniRegisterNativeMethods 去注册 jni 函数，这样就完成了 hook 注入，riru 完成暴露了自己的 api

#### modules::Load(rirud)

接下来就开始加载基于 riru 开发的模块（modules::Load(rirud)）了

![](https://bbs.pediy.com/upload/attach/202111/844301_5N5QCZJ8DW84SYP.jpg)  

#### LoadModule(id, path, magisk_module_path)

先看 LoadModule(id, path, magisk_module_path); 如下所示 dlopen 和 init 来了

![](https://bbs.pediy.com/upload/attach/202111/844301_8MPTCRHKS7Z9U6V.jpg)  

#### hide::HideFromMaps()

再看 hide::HideFromMaps(); 往下跟一下关键点就到了 riruhide.so 的 riru_hide 和 do_hide

```
void HidePathsFromMaps(const std::set &names) {
    if (!riru_hide_func) return;
 
    LOGD("do hide");
    riru_hide_func(names);
 
    // cleanup riruhide.so    LOGD("dlclose");
    if (dlclose(riru_hide_handle) != 0) {
        LOGE("dlclose failed: %s", dlerror());
        return;
    }
} 
```

#### module.onModuleLoaded()

再看 module.onModuleLoaded() 就是 riru 模块提供的另一个 api 了，在 riru 提供的开发模块中可以看到注释介绍

```
static void onModuleLoaded() {
    // Called when this library is loaded and "hidden" by Riru (see Riru's hide.cpp)
        // If you want to use threads, start them here rather than the constructors
            // __attribute__((constructor)) or constructors of static variables,
                // or the "hide" will cause SIGSEGV
                }

```

至此，riru 模块加载原理的来龙去脉就根据源码看完了。。。

四，参考文献
------

Riru 原理浅析和 EdXposed 入口分析：[https://bbs.pediy.com/thread-263018.htm](https://bbs.pediy.com/thread-263018.htm)

and  

上面提到的所有链接

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

最后于 7 小时前 被 huaerxiela 编辑 ，原因：

[#源码分析](forum-161-1-127.htm) [#HOOK 注入](forum-161-1-125.htm) [#基础理论](forum-161-1-117.htm)