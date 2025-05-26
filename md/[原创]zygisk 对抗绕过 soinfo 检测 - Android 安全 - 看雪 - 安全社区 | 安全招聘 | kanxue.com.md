> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286980.htm)

> [原创]zygisk 对抗绕过 soinfo 检测

上一篇博客写了老版本 zygisk 使用 dlclose 关闭 so，所导致的问题，但是起始有些问题没有写清楚，所以有了第二篇

上一篇博客中的错误和解决
------------

### libart 加载之后注入

libart.so 加载之后注入 so，时机太早，是无法规避直接调用 dlclose 带来的 soinfo 空隙问题，上一篇博客写的确实有问题，当时我也有些疑问的，但是没有深入测试，而且在测试前期有些代码由 bug 导致从开始就造成了认知错误。

### 正确的时机和做法

我在脚本中有写，_ZN3artL25ZygoteHooks_nativePreForkEP7_JNIEnvP7_jclass 这个时机是我开始测试的时机，是没有问题的，他会在 nativeForkSystemServer 这个函数的前面

所以正确的时机是在 _ZN3artL25ZygoteHooks_nativePreForkEP7_JNIEnvP7_jclass 这个函数执行的时候，注入 zygisk.so 文件，这样只需要稍微修改一下 zygisk 的代码便可直接使用

第二种绕过方式
-------

在我发博客以后，有兄弟直接进行了测试，后来还直接发给我解决这个问题的项目，这里很感谢。  
参考了这个项目的代码 (https://github.com/JingMatrix/NeoZygisk)。

在上一篇中，因为 zygisk 使用了 dlopen 加载 so，导致了 zygisk 的 soinfo 加入到 linker 的 soinfo 列表里，在应用进程启动的时候，调用 dlclose 从 soinfo 的列表中删除 zygisk 的 soinfo，因为 so 加载时机太早，zygisk 的 soinfo 位置太靠前，所以删除 soinfo 的时候导致出现了空隙。我采用了将 libzygisk 加载时机后延的方法。

其实防止检测的思想就是防止 linker soinfo 的列表出现空隙的。也可以说这个空隙是专门针对 zygisk 通过 dlclose 自卸载的检测。

所以我们在完成对抗的同时需要达到以下条件

*   zygisk 的 soinfo 必须从 linker 的 soinfo 列表中删除
*   不能在 soinfo 中留下空隙
*   libzygisk.so 需要进行在用户进程启动以后完成自卸载

### 对抗思路和原理

在 libzygisk.so 加载以后，直接删除 linker 中 zygisk.so 的 soinfo，并且不关闭 libzygisk.so，同时更换自卸载函数，使用 munmap 完成自卸载。  
这个思路从自卸载的角度来看，和 dlclose 大同小异，但是在操作上可控性更高。调用 dlclose 完成卸载，相当于同时删除 soinfo，然后 munmap 删除已经加载到内存中 libzygisk.so。但是由于 so 加载时机很早，卸载时机又很晚，导致 linker 的 soinfo 出现了空隙。现在将这个两个操作分开来，加载以后，直接删除 soinfo，但是不释放已经加载到内存中的 so，这样后续的 so 加载就会覆盖当前的位置，就不会出现空隙了，然后在应用进程启动以后在使用 munmap 关闭 soinfo。

### 代码分析

替换自卸载函数

```
DCL_HOOK_FUNC(static int, pthread_attr_setstacksize, void *target, size_t size) {
    int res = old_pthread_attr_setstacksize((pthread_attr_t *) target, size);
 
    LOGV("pthread_attr_setstacksize called in [tid, pid]: %d, %d", gettid(), getpid());
 
    // Only perform unloading on the main thread
    if (gettid() != getpid()) return res;
 
    if (g_hook->should_unmap) {
        g_hook->restore_plt_hook();
        if (g_hook->should_unmap) {
            void *start_addr = g_hook->start_addr;
            size_t block_size = g_hook->block_size;
            delete g_hook;
 
            // Because both `pthread_attr_setstacksize` and `munmap` have the same function
            // signature, we can use `musttail` to let the compiler reuse our stack frame and thus
            // `munmap` will directly return to the caller of `pthread_attr_setstacksize`.
            LOGD("unmap libzygisk.so loaded at %p with size %zu", start_addr, block_size);
            [[clang::musttail]] return munmap(start_addr, block_size);
        }
    }
 
    delete g_hook;
    return res;
}
```

删除 soinfo

```
bool dropSoPath(const char *target_path) {
    bool path_found = false;
    if (solist == nullptr && !initialize()) {
        LOGE("failed to initialize solist");
        return path_found;
    }
    for (auto *iter = solist; iter; iter = iter->getNext()) {
        if (iter->getPath() && strstr(iter->getPath(), target_path)) {
            SoList::ProtectedDataGuard guard;
            LOGD("dropping solist record for %s with size %zu", iter->getPath(), iter->getSize());
            if (iter->getSize() > 0) {
                iter->setSize(0);
                SoInfo::soinfo_free(iter);
                path_found = true;
            }
        }
    }
    return path_found;
}
```

删除计数

```
void resetCounters(size_t load, size_t unload) {
    if (solist == nullptr && !initialize()) {
        LOGE("failed to initialize solist");
        return;
    }
    if (g_module_load_counter == nullptr || g_module_unload_counter == nullptr) {
        LOGD("g_module counters not defined, skip reseting them");
        return;
    }
    auto loaded_modules = *g_module_load_counter;
    auto unloaded_modules = *g_module_unload_counter;
    if (loaded_modules >= load) {
        *g_module_load_counter = loaded_modules - load;
        LOGD("reset g_module_load_counter to %zu", (size_t) *g_module_load_counter);
    }
    if (unloaded_modules >= unload) {
        *g_module_unload_counter = unloaded_modules - unload;
        LOGD("reset g_module_unload_counter to %zu", (size_t) *g_module_unload_counter);
    }
}
```

最后
--

就写到这吧，感谢开源项目提供的解决方案，我也不知都那位大佬搞得这个方案，只能说思路很新颖，写的很好。  
这是大佬项目地址 https://github.com/JingMatrix/NeoZygisk，我已经抄袭到我的项目上了  
我的项目上一次已经放过地址了，这次就不放了

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#HOOK 注入](forum-161-1-125.htm)