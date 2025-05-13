> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286801.htm)

> [原创]zygisk 注入对抗之 dlclose

前言
--

用了很久的 zygiskNext 不开源了，于是自己从零写了一个项目类似的项目 https://github.com/Thehepta/Android-Debug-Inject，  
其实本来是在 zygiskNext 的基础上进行魔改的，但是后来发现有个对抗问题确实绕不过，于是只能全部重构了。

注入检测问题
------

老版本的 zygisNext 本身自己有很多特征，这些检测我就不写了，我只写一个就是老版本 zygiskNext 和面具的 zygisk 这种注入方式可以比较稳定检测的技术（新版本的 zygiskNext 已经修复了这个问题了，另外，这个技术不敢保证 100% 哦）。

### 检测原理概述

遍历 soinfo 列表，如果发生空隙那说明在 zygote 进程中有过早加载的 so 调用了 dlclose，这种情况基本都可以说明 zygote 被入侵了。

#### zygisk so 自卸载

想要知道通杀检测技术，就要知道 zygisk 具体做了。  
zygisk 有一个思路很巧妙的 so 自卸载技术，我当时在学习的时候也是惊为天人。  
我们都知道 zygisk 有一个 so 注入到了 zygote 进程了，但是在 app 进程启动以后，我们会发现这个 so 直接从 app 进程里消失了，开始我以为是开了一个监控进程或者是 ptrace 监控什么的，后来发现都不是，然后我去研究代码，发现了下面这个 hook 的代码

```
// We cannot directly call `dlclose` to unload ourselves, otherwise when `dlclose` returns,
// it will return to our code which has been unmapped, causing segmentation fault.
// Instead, we hook `pthread_attr_destroy` which will be called when VM daemon threads start.
DCL_HOOK_FUNC(int, pthread_attr_destroy, void *target) {
    int res = old_pthread_attr_destroy((pthread_attr_t *)target);
 
    // Only perform unloading on the main thread
    if (gettid() != getpid())
        return res;
 
    LOGV("pthread_attr_destroy\n");
    if (should_unmap_zygisk) {
        unhook_functions();
        if (should_unmap_zygisk) {
            // Because both `pthread_attr_destroy` and `dlclose` have the same function signature,
            // we can use `musttail` to let the compiler reuse our stack frame and thus
            // `dlclose` will directly return to the caller of `pthread_attr_destroy`.
            [[clang::musttail]] return dlclose(self_handle);
        }
    }
 
    return res;
}
```

这是一个 plt hook 的代码看起来没什么神奇的，但是正是在这里，so 对自身完成了卸载。  
他的原理是 hook 了一个和 dlclose 签名一样，函数返回值，包括返回内容可能也差不多的函数，我们看下这个函数编译出来的样子

![](https://bbs.kanxue.com/upload/attach/202505/767217_C8EB97TTC6Q7M94.webp)

看下调用 dlclose 函数，一般来说调用系统函数都需要使用 bl 指令，但是在这里使用的是 b 指令，我们执行完我们的代码，带着上一个被 hook 函数的返回地址，或者说堆栈去 b dlclose 函数，dlclose 函数会直接像直接调用 bl 的那样返回，同时通过上一个堆栈的地址进行函数返回。

clang::musttail 这个 可能就是将 bl 变成 b 指令的关键。

#### 基于 dlclose 的 so 加载检测

#### soinfo 申请

可以看到上面的函数调用了 dlclose，我们正常的思维里，调用了 dlopen 以后，函数完成关闭，就没有然后了，但是其实不然。  
我们看一下 soinfo 的生成逻辑

```
soinfo* soinfo_alloc(android_namespace_t* ns, const char* name,const struct stat* file_stat, off64_t file_offset,uint32_t rtld_flags) {
 
    if (strlen(name) >= PATH_MAX) {
        async_safe_fatal("library name \"%s\" too long", name);
    }
    TRACE("name %s: allocating soinfo for ns=%p", name, ns);
    soinfo* si = new (g_soinfo_allocator.alloc()) soinfo(ns, name, file_stat,
    file_offset, rtld_flags);
    solist_add_soinfo(si);
    si->generate_handle();
    ns->add_soinfo(si);
    TRACE("name %s: allocated soinfo @ %p", name, si);
    return si;
}
```

这个 new 生成 soinfo ，会在 g_soinfo_allocator.alloc() 返回的地址上分配内存，这也就导致了所有的 soinfo 的地址是紧紧挨在一起的。如果我们关闭中间的 soinfo，这个中间位置就会空出来。而我们的检测就是检测中间是否有空出来的 soinfo。

#### 查看 soinfo 间距的代码

```
#include "jni.h"
#include "vector"
#include "elf_symbol_resolver.h"
 
#include "android/log.h"
 
#define LOGV(...)  ((void)__android_log_print(ANDROID_LOG_INFO, "soinfo", __VA_ARGS__))
 
 
#define soinfo uintptr_t
 
 
soinfo* (*solist_get_head)() = (soinfo* (*)()) linkerResolveElfInternalSymbol(
        get_android_linker_path(), "__dl__Z15solist_get_headv");
 
soinfo* (*solist_get_somain)() = (soinfo* (*)()) linkerResolveElfInternalSymbol(
        get_android_linker_path(), "__dl__Z17solist_get_somainv");
 
char* (*soinfo_get_soname)(soinfo*) = (char* (*)(soinfo*)) linkerResolveElfInternalSymbol(
        get_android_linker_path(), "__dl__ZNK6soinfo10get_sonameEv");
 
 
 
 
 
//通过动态计算结构内部成员的位置，循环找到soinfo 内部的next的位置，然后通过next进行遍历
void find_all_library_name(){
    std::vector linker_solist;
 
    static uintptr_t *solist_head = NULL;
    if (!solist_head)
        solist_head = (uintptr_t *)solist_get_head();
 
 
    static uintptr_t somain = 0;
 
    if (!somain)
        somain = (uintptr_t)solist_get_somain();
 
    // Generate the name for an offset.
#define PARAM_OFFSET(type_, member_) __##type_##__##member_##__offset_
#define STRUCT_OFFSET PARAM_OFFSET
    int STRUCT_OFFSET(solist, next) = 0;
    for (size_t i = 0; i < 1024 / sizeof(void *); i++) {
        if (*(uintptr_t *)((uintptr_t)solist_head + i * sizeof(void *)) == somain) {
            STRUCT_OFFSET(solist, next) = i * sizeof(void *);
            break;
        }
    }
 
    linker_solist.push_back(solist_head);
 
    uintptr_t sonext = 0;
    sonext = *(uintptr_t *)((uintptr_t)solist_head + STRUCT_OFFSET(solist, next));
    while (sonext) {
        linker_solist.push_back((void *)sonext);
        sonext = *(uintptr_t *)((uintptr_t)sonext + STRUCT_OFFSET(solist, next));
        if(sonext == 0){
            continue;
        }
        char* ret_name = soinfo_get_soname(reinterpret_cast(sonext));
        LOGV("soinfo addr:%p %s",sonext,ret_name);
    }
 
    return ;
} 
```

#### 不同 soinfo 对比

正常的 soinfo

```
soinfo_addr:0x72a9159268 size: 0x0,    linux-vdso.so.1
soinfo_addr:0x72a91594c0 size: 0x258,  libandroid_runtime.so
soinfo_addr:0x72a9159718 size: 0x258,  libbinder.so
soinfo_addr:0x72a9159970 size: 0x258,  libcutils.so
soinfo_addr:0x72a9159bc8 size: 0x258,  libhidlbase.so
soinfo_addr:0x72a9159e20 size: 0x258,  liblog.so
soinfo_addr:0x72a915a078 size: 0x258,  libnativeloader.so
soinfo_addr:0x72a915a2d0 size: 0x258,  libsigchain.so
soinfo_addr:0x72a915a528 size: 0x258,  libutils.so
soinfo_addr:0x72a915a780 size: 0x258,  libwilhelm.so
soinfo_addr:0x72a915a9d8 size: 0x258,  libc++.so
soinfo_addr:0x72a915ac30 size: 0x258,  libc.so
soinfo_addr:0x72a915ae88 size: 0x258,  libm.so
soinfo_addr:0x72a915b0e0 size: 0x258,  libdl.so
soinfo_addr:0x72a915b338 size: 0x258,  libbase.so
soinfo_addr:0x72a915b590 size: 0x258,  libharfbuzz_ng.so
soinfo_addr:0x72a915b7e8 size: 0x258,  libminikin.so
soinfo_addr:0x72a915ba40 size: 0x258,  libz.so
soinfo_addr:0x72a915bc98 size: 0x258,  android.media.audio.common.types-V1-cpp.so
soinfo_addr:0x72a915bef0 size: 0x258,  audioclient-types-aidl-cpp.so
soinfo_addr:0x72a915c148 size: 0x258,  audioflinger-aidl-cpp.so
soinfo_addr:0x72a915c3a0 size: 0x258,  audiopolicy-types-aidl-cpp.so
soinfo_addr:0x72a915c5f8 size: 0x258,  spatializer-aidl-cpp.so
soinfo_addr:0x72a915c850 size: 0x258,  av-types-aidl-cpp.so
soinfo_addr:0x72a915caa8 size: 0x258,  android.hardware.camera.device@3.2.so
soinfo_addr:0x72a915cd00 size: 0x258,  libandroid_net.so
soinfo_addr:0x72a915cf58 size: 0x258,  libandroidicu.so
soinfo_addr:0x72a915d1b0 size: 0x258,  libbattery.so
soinfo_addr:0x72a915d408 size: 0x258,  libnetdutils.so
soinfo_addr:0x72a915d660 size: 0x258,  libmemtrack.so
soinfo_addr:0x72a915d8b8 size: 0x258,  libandroidfw.so
soinfo_addr:0x72a915db10 size: 0x258,  libappfuse.so
soinfo_addr:0x72a915dd68 size: 0x258,  libcrypto.so
soinfo_addr:0x72a915dfc0 size: 0x258,  libdebuggerd_client.so
soinfo_addr:0x72a915e218 size: 0x258,  libbinder_ndk.so
soinfo_addr:0x72a915e470 size: 0x258,  libui.so
soinfo_addr:0x72a915e6c8 size: 0x258,  libgraphicsenv.so
soinfo_addr:0x72a915e920 size: 0x258,  libgui.so
soinfo_addr:0x72a915eb78 size: 0x258,  libhwui.so
soinfo_addr:0x72a915edd0 size: 0x258,  libmediandk.so
soinfo_addr:0x72a915f028 size: 0x258,  libpermission.so
soinfo_addr:0x72a915f280 size: 0x258,  libsensor.so
soinfo_addr:0x72a915f4d8 size: 0x258,  libinput.so
soinfo_addr:0x72a915f730 size: 0x258,  libcamera_client.so
soinfo_addr:0x72a915f988 size: 0x258,  libcamera_metadata.so
soinfo_addr:0x72a915fbe0 size: 0x258,  libprocinfo.so
soinfo_addr:0x72a915fe38 size: 0x258,  libsqlite.so
soinfo_addr:0x72a9160090 size: 0x258,  libEGL.so
soinfo_addr:0x72a91602e8 size: 0x258,  libGLESv1_CM.so
soinfo_addr:0x72a9160540 size: 0x258,  libGLESv2.so
soinfo_addr:0x72a9160798 size: 0x258,  libGLESv3.so
soinfo_addr:0x72a91609f0 size: 0x258,  libincfs.so
soinfo_addr:0x72a9160c48 size: 0x258,  libdataloader.so
soinfo_addr:0x72a9160ea0 size: 0x258,  libvulkan.so
soinfo_addr:0x72a91610f8 size: 0x258,  libETC1.so
soinfo_addr:0x72a9161350 size: 0x258,  libjpeg.so
soinfo_addr:0x72a91615a8 size: 0x258,  libhardware.so
soinfo_addr:0x72a9161800 size: 0x258,  libhardware_legacy.so
soinfo_addr:0x72a9161a58 size: 0x258,  libselinux.so
soinfo_addr:0x72a9161cb0 size: 0x258,  libmedia.so
soinfo_addr:0x72a9161f08 size: 0x258,  libmedia_helper.so
soinfo_addr:0x72a9162160 size: 0x258,  libmediametrics.so
soinfo_addr:0x72a91623b8 size: 0x258,  libmeminfo.so
soinfo_addr:0x72a9162610 size: 0x258,  libaudioclient.so
soinfo_addr:0x72a9162868 size: 0x258,  libaudioclient_aidl_conversion.so
soinfo_addr:0x72a9162ac0 size: 0x258,  libaudiofoundation.so
soinfo_addr:0x72a9162d18 size: 0x258,  libaudiopolicy.so
soinfo_addr:0x72a9162f70 size: 0x258,  libusbhost.so
soinfo_addr:0x72a91631c8 size: 0x258,  libpdfium.so
soinfo_addr:0x72a9163420 size: 0x258,  libimg_utils.so
soinfo_addr:0x72a9163678 size: 0x258,  libnetd_client.so
soinfo_addr:0x72a91638d0 size: 0x258,  libprocessgroup.so
soinfo_addr:0x72a9163b28 size: 0x258,  libnativebridge_lazy.so
soinfo_addr:0x72a9163d80 size: 0x258,  libnativeloader_lazy.so
soinfo_addr:0x72a9163fd8 size: 0x258,  libmemunreachable.so
soinfo_addr:0x72a9164230 size: 0x258,  libvintf.so
soinfo_addr:0x72a9164488 size: 0x258,  libnativedisplay.so
soinfo_addr:0x72a91646e0 size: 0x258,  libnativewindow.so
soinfo_addr:0x72a9164938 size: 0x258,  libdl_android.so
soinfo_addr:0x72a9164b90 size: 0x258,  libtimeinstate.so
soinfo_addr:0x72a9164de8 size: 0x258,  server_configurable_flags.so
soinfo_addr:0x72a9165040 size: 0x258,  libvndksupport.so
soinfo_addr:0x72a9165298 size: 0x258,  libnativebridge.so
soinfo_addr:0x72a91654f0 size: 0x258,  libbase.so
soinfo_addr:0x72a9165748 size: 0x258,  libc++.so
soinfo_addr:0x72a91659a0 size: 0x258,  framework-permission-aidl-cpp.so
soinfo_addr:0x72a9165bf8 size: 0x258,  libmedia_codeclist.so
soinfo_addr:0x72a9165e50 size: 0x258,  libaudiomanager.so
soinfo_addr:0x72a91660a8 size: 0x258,  libdatasource.so
soinfo_addr:0x72a9166300 size: 0x258,  libstagefright.so
soinfo_addr:0x72a9166558 size: 0x258,  libstagefright_foundation.so
soinfo_addr:0x72a91667b0 size: 0x258,  libstagefright_http_support.so
soinfo_addr:0x72a9166a08 size: 0x258,  libicu.so
soinfo_addr:0x72a9166c60 size: 0x258,  effect-aidl-cpp.so
soinfo_addr:0x72a9166eb8 size: 0x258,  shared-file-region-aidl-cpp.so
soinfo_addr:0x72a9167110 size: 0x258,  android.hardware.camera.common@1.0.so
soinfo_addr:0x72a9167368 size: 0x258,  android.hardware.graphics.common@1.0.so
soinfo_addr:0x72a91675c0 size: 0x258,  libbase.so
soinfo_addr:0x72a9167818 size: 0x258,  libicuuc.so
soinfo_addr:0x72a9167a70 size: 0x258,  libicui18n.so
soinfo_addr:0x72a9167cc8 size: 0x258,  libc++.so
soinfo_addr:0x72a9167f20 size: 0x258,  android.hardware.memtrack@1.0.so
soinfo_addr:0x72a9168178 size: 0x258,  android.hardware.memtrack-V1-ndk.so
soinfo_addr:0x72a91683d0 size: 0x258,  libandroid_runtime_lazy.so
soinfo_addr:0x72a9168628 size: 0x258,  android.hardware.graphics.allocator@2.0.so
soinfo_addr:0x72a9168880 size: 0x258,  android.hardware.graphics.allocator@3.0.so
soinfo_addr:0x72a9168ad8 size: 0x258,  android.hardware.graphics.allocator@4.0.so
soinfo_addr:0x72a9168d30 size: 0x258,  android.hardware.graphics.allocator-V1-ndk.so
soinfo_addr:0x72a9168f88 size: 0x258,  android.hardware.graphics.common-V3-ndk.so
soinfo_addr:0x72a91691e0 size: 0x258,  android.hardware.graphics.common@1.2.so
soinfo_addr:0x72a9169438 size: 0x258,  android.hardware.graphics.mapper@2.0.so
soinfo_addr:0x72a9169690 size: 0x258,  android.hardware.graphics.mapper@2.1.so
soinfo_addr:0x72a91698e8 size: 0x258,  android.hardware.graphics.mapper@3.0.so
soinfo_addr:0x72a9169b40 size: 0x258,  android.hardware.graphics.mapper@4.0.so
soinfo_addr:0x72a9169d98 size: 0x258,  libgralloctypes.so
soinfo_addr:0x72a9169ff0 size: 0x258,  libsync.so
soinfo_addr:0x72a916a248 size: 0x258,  android.hardware.graphics.bufferqueue@1.0.so
soinfo_addr:0x72a916a4a0 size: 0x258,  android.hardware.graphics.bufferqueue@2.0.so
soinfo_addr:0x72a916a6f8 size: 0x258,  android.hardware.graphics.common@1.1.so
soinfo_addr:0x72a916a950 size: 0x258,  android.hidl.token@1.0-utils.so
soinfo_addr:0x72a916aba8 size: 0x258,  libbufferhub.so
soinfo_addr:0x72a916ae00 size: 0x258,  libbufferhubqueue.so
soinfo_addr:0x72a916b058 size: 0x258,  libpdx_default_transport.so
soinfo_addr:0x72a916b2b0 size: 0x258,  libpng.so
soinfo_addr:0x72a916b508 size: 0x258,  libdng_sdk.so
soinfo_addr:0x72a916b760 size: 0x258,  libpiex.so
soinfo_addr:0x72a916b9b8 size: 0x258,  libexpat.so
soinfo_addr:0x72a916bc10 size: 0x258,  libheif.so
soinfo_addr:0x72a916be68 size: 0x258,  android.hardware.graphics.composer3-V1-ndk.so
soinfo_addr:0x72a916c0c0 size: 0x258,  libprotobuf-cpp-lite.so
soinfo_addr:0x72a916c318 size: 0x258,  libft2.so
soinfo_addr:0x72a916c570 size: 0x258,  libjpeg-alpha.so
soinfo_addr:0x72a916c7c8 size: 0x258,  vendor.mediatek.hardware.mms@1.0.so
soinfo_addr:0x72a916ca20 size: 0x258,  vendor.mediatek.hardware.mms@1.1.so
soinfo_addr:0x72a916cc78 size: 0x258,  vendor.mediatek.hardware.mms@1.2.so
soinfo_addr:0x72a916ced0 size: 0x258,  vendor.mediatek.hardware.mms@1.3.so
soinfo_addr:0x72a916d128 size: 0x258,  libion.so
soinfo_addr:0x72a916d380 size: 0x258,  libdmabufheap.so
soinfo_addr:0x72a916d5d8 size: 0x258,  libJpegOal.so
soinfo_addr:0x72a916d830 size: 0x258,  libmediadrm.so
soinfo_addr:0x72a916da88 size: 0x258,  libmedia_omx.so
soinfo_addr:0x72a916dce0 size: 0x258,  libmedia_jni_utils.so
soinfo_addr:0x72a916df38 size: 0x258,  libmediandk_utils.so
soinfo_addr:0x72a916e190 size: 0x258,  android.hardware.drm-V1-ndk.so
soinfo_addr:0x72a916e3e8 size: 0x258,  libbacktrace.so
soinfo_addr:0x72a916e640 size: 0x258,  android.hardware.configstore@1.0.so
soinfo_addr:0x72a916e898 size: 0x258,  android.hardware.configstore-utils.so
soinfo_addr:0x72a916eaf0 size: 0x258,  libSurfaceFlingerProp.so
soinfo_addr:0x72a916ed48 size: 0x258,  libgralloc_extra_sys.so
soinfo_addr:0x72a916efa0 size: 0x258,  libgpud_sys.so
soinfo_addr:0x72a916f1f8 size: 0x258,  libziparchive.so
soinfo_addr:0x72a916f450 size: 0x258,  android.system.suspend-V1-ndk.so
soinfo_addr:0x72a916f6a8 size: 0x258,  libpcre2.so
soinfo_addr:0x72a916f900 size: 0x258,  libpackagelistparser.so
soinfo_addr:0x72a916fb58 size: 0x258,  mediametricsservice-aidl-cpp.so
soinfo_addr:0x72a916fdb0 size: 0x258,  libaudioutilmtk.so
soinfo_addr:0x72a9170008 size: 0x258,  audiopolicy-aidl-cpp.so
soinfo_addr:0x72a9170260 size: 0x258,  capture_state_listener-aidl-cpp.so
soinfo_addr:0x72a91704b8 size: 0x258,  libaudioutils.so
soinfo_addr:0x72a9170710 size: 0x258,  libmediautils.so
soinfo_addr:0x72a9170968 size: 0x258,  libnblog.so
soinfo_addr:0x72a9170bc0 size: 0x258,  libshmemcompat.so
soinfo_addr:0x72a9170e18 size: 0x258,  packagemanager_aidl-cpp.so
soinfo_addr:0x72a9171070 size: 0x258,  libcgrouprc.so
soinfo_addr:0x72a91712c8 size: 0x258,  libhidl-gen-utils.so
soinfo_addr:0x72a9171520 size: 0x258,  libtinyxml2.so
soinfo_addr:0x72a9171778 size: 0x258,  libbpf_bcc.so
soinfo_addr:0x72a91719d0 size: 0x258,  libbpf_minimal.so
soinfo_addr:0x72a9171c28 size: 0x258,  android.hardware.media.omx@1.0.so
soinfo_addr:0x72a9171e80 size: 0x258,  libstagefright_framecapture_utils.so
soinfo_addr:0x72a91720d8 size: 0x258,  libcodec2.so
soinfo_addr:0x72a9172330 size: 0x258,  libcodec2_vndk.so
soinfo_addr:0x72a9172588 size: 0x258,  libmedia_omx_client.so
soinfo_addr:0x72a91727e0 size: 0x258,  libsfplugin_ccodec.so
soinfo_addr:0x72a9172a38 size: 0x258,  libsfplugin_ccodec_utils.so
soinfo_addr:0x72a9172c90 size: 0x258,  libstagefright_codecbase.so
soinfo_addr:0x72a9172ee8 size: 0x258,  libstagefright_omx_utils.so
soinfo_addr:0x72a9173140 size: 0x258,  libRScpp.so
soinfo_addr:0x72a9173398 size: 0x258,  libhidlallocatorutils.so
soinfo_addr:0x72a91735f0 size: 0x258,  libhidlmemory.so
soinfo_addr:0x72a9173848 size: 0x258,  android.hidl.allocator@1.0.so
soinfo_addr:0x72a9173aa0 size: 0x258,  android.hardware.cas.native@1.0.so
soinfo_addr:0x72a9173cf8 size: 0x258,  android.hardware.drm@1.0.so
soinfo_addr:0x72a9173f50 size: 0x258,  android.hardware.common-V2-ndk.so
soinfo_addr:0x72a91741a8 size: 0x258,  android.hardware.media@1.0.so
soinfo_addr:0x72a9174400 size: 0x258,  android.hidl.token@1.0.so
soinfo_addr:0x72a9174658 size: 0x258,  libjpeg-alpha-oal.so
soinfo_addr:0x72a91748b0 size: 0x258,  libmediadrmmetrics_lite.so
soinfo_addr:0x72a9174b08 size: 0x258,  android.hardware.drm@1.1.so
soinfo_addr:0x72a9174d60 size: 0x258,  android.hardware.drm@1.2.so
soinfo_addr:0x72a9174fb8 size: 0x258,  android.hardware.drm@1.3.so
soinfo_addr:0x72a9175210 size: 0x258,  android.hardware.drm@1.4.so
soinfo_addr:0x72a9175468 size: 0x258,  libunwindstack.so
soinfo_addr:0x72a91756c0 size: 0x258,  android.hardware.configstore@1.1.so
soinfo_addr:0x72a9175918 size: 0x258,  libsf_cpupolicy.so
soinfo_addr:0x72a9175b70 size: 0x258,  libged_sys.so
soinfo_addr:0x72a9175dc8 size: 0x258,  libutilscallstack.so
soinfo_addr:0x72a9176020 size: 0x258,  libspeexresampler.so
soinfo_addr:0x72a9176278 size: 0x258,  libshmemutil.so
soinfo_addr:0x72a91764d0 size: 0x258,  android.hardware.media.bufferpool@2.0.so
soinfo_addr:0x72a9176728 size: 0x258,  libfmq.so
soinfo_addr:0x72a9176980 size: 0x258,  libstagefright_bufferpool@2.0.1.so
soinfo_addr:0x72a9176bd8 size: 0x258,  android.hardware.media.c2@1.0.so
soinfo_addr:0x72a9176e30 size: 0x258,  libcodec2_client.so
soinfo_addr:0x72a9177088 size: 0x258,  libstagefright_bufferqueue_helper.so
soinfo_addr:0x72a91772e0 size: 0x258,  libstagefright_omx.so
soinfo_addr:0x72a9177538 size: 0x258,  libstagefright_surface_utils.so
soinfo_addr:0x72a9177790 size: 0x258,  libstagefright_xmlparser.so
soinfo_addr:0x72a91779e8 size: 0x258,  android.hidl.memory@1.0.so
soinfo_addr:0x72a9177c40 size: 0x258,  android.hidl.memory.token@1.0.so
soinfo_addr:0x72a9177e98 size: 0x258,  android.hardware.cas@1.0.so
soinfo_addr:0x72a91780f0 size: 0x258,  liblzma.so
soinfo_addr:0x72a9178348 size: 0x258,  android.hidl.safe_union@1.0.so
soinfo_addr:0x72a91785a0 size: 0x258,  android.hardware.media.c2@1.1.so
soinfo_addr:0x72a91787f8 size: 0x258,  android.hardware.media.c2@1.2.so
soinfo_addr:0x72a9178a50 size: 0x258,  libcodec2_hidl_client@1.0.so
soinfo_addr:0x72a9178ca8 size: 0x258,  libcodec2_hidl_client@1.1.so
soinfo_addr:0x72a9178f00 size: 0x258,  libcodec2_hidl_client@1.2.so
soinfo_addr:0x72a9179158 size: 0x258,  libged_kpi.so
soinfo_addr:0x72a91793b0 size: 0x258,  libnativehelper.so
soinfo_addr:0x72a9179608 size: 0x258,  libart.so
```

有问题的 soinfo

```
soinfo_addr:0x72b1530a50 size: 0x258,  libcodec2_hidl_client@1.0.so
soinfo_addr:0x72b1530ca8 size: 0x258,  libcodec2_hidl_client@1.1.so
soinfo_addr:0x72b1530f00 size: 0x258,  libcodec2_hidl_client@1.2.so
soinfo_addr:0x72b1531158 size: 0x258,  libged_kpi.so
soinfo_addr:0x72b1531608 size: 0x4b0,  libnativehelper.so
soinfo_addr:0x72b1531860 size: 0x258,  libart.so
```

我们可以看到有问题的这个 soinfo, 其中有一个间距为 0x4b0，这说明中间刚好关闭了一个 soinfo

### 如何发现和分析这个问题的

是通过一款检测软件 homles 检测了这个问题，我当时魔改 zygiskNext 基本已经完成，所有的特征全部修改，但是还是显示注入，这引起了我对于 zygiskNext 本身这种注入方式的怀疑，于是我从来开始一步一步严重，最终发现在使用 so 自卸载技术的时候会发生被检测到底情况，如果不是用这种技术，就不会被检测。然后我立马就锁定了 dlclose 所带来的 soInfo 间距检测问题。

### 为什么 zygisk 会有这个问题

（在重复一次，新版本的 zygiskNext 已经修复了这个问题了，这个技术不敢保证 100% 哦）  
老版本的 zygiskNext 和 目前的 magisk zygisk （v28.1）加载 libzgysik.so 的时机太早了，导致 libzgysik.so 的 soinfo 太过于靠前，尤其是 libart.so 这些都没有加载，时机太早了，当用户进程启动以后，会关闭一些 soinfo, 然后在打开一些 soinfo, 如果关闭 soinfo 无法回退到到这个位置，导致这个位置发生了空隙，尤其明显的是，会发生一个 soinfo 的空隙，而且会在某些 soinfo 前面，比如 libart.so。  
但是这目前是我的猜测。因为我并没有去验证 soinfo 关闭以后再次申请 soinfo 的 机制，同时我也测试过一些手机，发现实际只要在 libart.so 的后面好像就没有问题，难道刚好就关闭到 libart.so 的 soinfo 后面吗

### 解决方案

只要将 libzygisk.so 的加载时机向后延，我开始是在 zygote 启动 systemservice 进程之前，后来我在测试工具的时候，发现放到 libart.so 加载时机之后也没问题。

### 最后

这个技术目前来说，已经有了可以绕过的方案，并且我自己也完成了，有点价值，但是不大，不过确实是有值得学习，所以写个博客蹭点流量。  
如果你对我写的感兴趣就关注一下我的公众号吧。再次推销下我的项目  
https://github.com/Thehepta/Android-Debug-Inject

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 1 天前 被 Thehepta 编辑 ，原因：

[#HOOK 注入](forum-161-1-125.htm)