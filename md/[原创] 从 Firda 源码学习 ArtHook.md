> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269014.htm)

> [原创] 从 Firda 源码学习 ArtHook

为了填 [AppInspect](https://bbs.pediy.com/thread-269004-1.htm) 的坑，开始研究 ArtHook。

 

从 Frida 入手是因为的资料多，更新频繁。

 

坏处是 Frida [体系庞大](https://frida.re/docs/hacking/)，比其他单纯的 Hook 框架复杂。

 

Frida 的 native hook 代码在 [frida-gum](https://github.com/frida/frida-gum)，虽然是 C 语言，但用了 gobject 框架，对没接触过的人需要一定时间习惯。

 

Frida 的 art hook 代码在 [frida-java-bridge](https://github.com/frida/frida-java-bridge/) ，纯 js 代码实现。

 

通过 [enumerateLoadedClasses](https://github.com/frida/frida-java-bridge/blob/master/index.js#L114) 的执行流程，作为入口开始代码走读。

 

第一步是初始化 Runtime，这个 Runtime 是 Art Runtime 在 js 侧的一个代理，是 Frida ArtHook 的核心。

 

通过 [getApi()](https://github.com/frida/frida-java-bridge/blob/master/index.js#L54) 获取 Art Runtime 的 Api 接口。

 

怎么理解获取 Api 呢，

 

Android 程序运行时，art 运行时会以 libart.so 等动态库的形式加载在当前进程的地址空间中。那我们能不能像访问 libc.so 里的函数一样访问里面的函数呢？

 

答案是肯定的，

 

从 CPU 运行流程看，调用一个函数就是，准备好函数的参数，跳到函数地址执行。

 

这里只有两个问题要解决：

1.  如何知道函数的参数。
2.  如何知道函数的地址。

只是如果你在 ndk 的代码里调用 c 库的函数，头文件解决了参数问题，而函数地址，在编译时和运行时由链接器帮你填好。

 

但对于 libart.so 我们只能自己动手了，好在 Linux 有一套运行时加载 so 的机制。

 

https://tldp.org/HOWTO/html_single/C++-dlopen/

```
void* handle = dlopen("./hello.so", RTLD_LAZY);
typedef void (*hello_t)();
hello_t hello = (hello_t) dlsym(handle, "hello");
hello();

```

但在 Android 7 以后，限制了 App 直接调用 dlopen 打开 libart.so 等系统库，

```
void* handle = dlopen("libart.so", RTLD_LAZY);
LOGD("dlopen %p",handle); //返回0

```

作为例外 libc .so, libandroid.so 等在白名单里的不受影响。白名单在 / system/etc/public.libraries.txt

 

除了白名单，还有一个在代码里写死的灰名单，需在编译 App 时把 targetSdk 设定为 23 之前才能有效。

 

那么怎么绕过这个限制等呢：

 

有两种方法：

1.  我们要调用 dlopen/dlsym 的目的，并不是真的要加载 so 到内存，而是想获取函数在内存中的地址。Linux 系统（包括 Android）会把加载动态库的地址范围信息，通过 / proc/$pid/maps 文件暴露出来，只要读取这个文件就可以获取到 so 的基址，然后可通过解析 elf 头，获取到各个函数的地址信息。
2.  不直接调用 libc 的 dlopen，而是使用 android 系统库 libdl.so 的__loader_dlopen，

```
void* __loader_dlopen(const char* filename, int flags, const void* caller_addr) {
  return dlopen_ext(filename, flags, nullptr, caller_addr);
}

```

跟踪调用，真正干活的是 linker 中的 do_dlopen 函数，它会根据__loader_dlopen 传进来的 caller_addr，检查调用者所属的模块，然后根据模块查找其对应命名空间，判断是否允许加载。

```
soinfo* const caller = find_containing_library(caller_addr);
android_namespace_t* ns = get_caller_namespace(caller);

```

所以只要传一个系统库中的函数，就能加载其他在同一个命名空间里的系统模块。

 

如何找一个跟目标模块命名空间相同的地址呢,

 

方法 1，调用 [dl_iterate_data](https://linux.die.net/man/3/dl_iterate_phdr)

> The **dl_iterate_phdr**() function walks through the list of an application's shared objects and calls the function _callback_ once for each object,

 

参考[快手的 KOOM](https://github.com/KwaiAppTeam/KOOM)

 

原理就是调用 dl_iterate_phdr 后，会在 callback 里返回 App 加载的所有模块信息，当找到目标模块时，把它基址保存下来，作为__loader_dlopen 的 caller_addr 参数。

 

方法 2，通过 / proc/self/maps 文件获取到目标模块的基址作为 caller_addr。

 

另外，还有一个更简单的方法是传一个 libc 里的函数地址，但 Android Q 以后增加了新的命名空间，libc 和 libart 并不在同一个命名空间里。

 

Rust 版代码验证

```
type __loader_dlopen_t = unsafe extern "C" fn(*const c_char, i32, *const c_void) -> *const c_void;
 
unsafe fn test_dlopen() -> anyhow::Result<()> {
    //获取libdl.so中的__loader_dlopen地址（libdl.so在白名单中不受限）
    let libdl = libc::dlopen("libdl.so".to_c_string().as_ptr(), libc::RTLD_LAZY);
    log::debug!("libdl: {:?}", libdl);
    let __loader_dlopen_ptr = libc::dlsym(libdl, "__loader_dlopen".to_c_string().as_ptr());
    let __loader_dlopen: __loader_dlopen_t = std::mem::transmute(__loader_dlopen_ptr);
 
      //通过proc maps获取libart.so的基址
    let module_info = util_rs::proc_map::get_module_info("libart.so", "/proc/self/maps")
        .context("fail to parse map proc!")?;
    let ptr = module_info.range_start as *const c_void;
    log::debug!("find libart.so start ptr: {:p}", ptr);
 
    //libart.so的基址作为caller_addr调用__loader_dlopen
    let handle = __loader_dlopen("libart.so".to_c_string().as_ptr(), libc::RTLD_LAZY, ptr);
    log::debug!("__loader_dlopen : {:?}", handle);
 
    //直接使用dlsym查找符号表
      let sym = libc::dlsym(
            handle as *mut c_void,
        "JNI_GetCreatedJavaVMs".to_c_string().as_ptr(),
    );
    log::debug!("sym: {:?}", sym);
 
    Ok(())
}

```

看到这里，可能会有个疑问，为什么谷歌这么弱，弄个安全机制这么简单就被破解了，

 

看看官方对链接器命名空间的介绍,

 

https://source.android.com/devices/architecture/vndk/linker-namespace?hl=zh-cn

 

链接器命名空间解决的问题是：可以隔离不同链接器命名空间中的共享库，以确保具有相同库名称和不同符号的库不会发生冲突。

 

所以它不是一种安全机制，只是为了安全的解析符号，前面的操作只是对这种限制的规避，不能算漏洞，谷歌也不会去修复，所以预估能够长期使用。

 

回到 Frida，再看看 Frida 是怎么解决这个 dlopen 限制的

 

因为手头只有 Android 10 的设备，这里只关注 Android29 的版本

 

frida-gum 在里查找`__dl___loader_dlopen`（地址就是 libdl.so 的 **loader_dlopen），如果没有找到会通过搜索内存对比指令集特征的方式继续查找。但在 Android 10 上，这一步就找到了，之后的操作和前述 `**loader_dlopen` 方式一致。

 

[gum_linker_api_try_init](https://github.com/frida/frida-gum/blob/master/gum/backend-linux/gumandroid.c#L1116)

 

理解了这些基本原理后，后面再回到 frida-java-bridge，看看 js 端如何访问 Android 运行时。

 

参考：

 

https://bbs.pediy.com/thread-257022.htm

 

http://weishu.me/2018/06/07/free-reflection-above-android-p/

 

https://fadeevab.com/android-linker-namespace-security-flaws/

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码分析](forum-161-1-127.htm)