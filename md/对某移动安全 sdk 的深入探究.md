> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2076506-1-1.html)

> [md]# 某 SDK Frida 检测绕过深度分析对一些 app 进行初步的分析之后发现，很多 app 都集成了该 sdk，绕过方法网上记载的也是很多，这篇文章深入挖掘一下如何绕过检测的始末。 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)bananashipsBBQ _ 本帖最后由 bananashipsBBQ 于 2025-11-29 20:12 编辑_  

某 SDK Frida 检测绕过深度分析
--------------------

对一些 app 进行初步的分析之后发现，很多 app 都集成了该 sdk，绕过方法网上记载的也是很多，这篇文章深入挖掘一下如何绕过检测的始末。

### 0x01 动态定位分析

这里直接选用一款使用该 sdk 的 app 作为 demo 展开分析，直接挂载 frida 之后会直接死掉，所以先检查一下是在挂载了哪个具体的 so 文件之后挂掉的，定位到该 so 文件之后挂掉了。  
![](https://attach.52pojie.cn/forum/202511/29/163027jvxl7pvpi7x8pudi.jpg)

**1.jpg** _(668.14 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyMXwzOWZmZmIzZXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:30 上传

然后需要具体定位一下是在该 so 文件的具体哪个地方导致的 frida 死掉，使用下面这段脚本，过滤到目标 so 之后，看是否能正常打印 Jni_Onload 的函数：

```
function hookDlopen() {
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){//主要逻辑是hook linker64中的do_dlopen函数，然后在该函数返回的时候检查目标检测库是否加载
        this.name = args[0].readCString()
        console.log(`dlopen onEnter ${this.name}`)
      }, onLeave: function(retval){
        console.log(`dlopen onLeave name: ${this.name}`)
                if (this.name != null && this.name.indexOf('目标so文件') >= 0) {
          let JNI_OnLoad = Module.getExportByName(this.name, 'JNI_OnLoad')
          console.log(`dlopen onLeave JNI_OnLoad: ${JNI_OnLoad}`)
        }//如果加载了目标库的话则获取该库的 JNI_OnLoad 函数地址并打印
      }
    })
  }

```

结果如下，很明显是没有走到 Jni_OnLoad 的，根据 so 文件的加载流程可以初步判断该检测是在 so 的 **init_proc** 中或者 **init_array 段**做的。  
![](https://attach.52pojie.cn/forum/202511/29/163916n0bnlcaanleqq02a.jpg)

**2.jpg** _(355 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyMnxmY2Y0Yjc2N3wxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:39 上传

为了进一步去判断该程序的检测退出的触发，既然挂上 Frida 后进程就退出，那么我们就来分析是调用哪个系统调用退出的，可以通过 **strace** 查看系统调用，但在执行 strace 时需要在 dlopen 加载目标 so 文件之前让线程 sleep 10 秒，以便留出 strace 执行时机。使用脚本如下：

```
function hookDlopen() {
    // 获取libc中的sleep函数
    let sleep = new NativeFunction(Module.getExportByName('libc.so', 'sleep'), 'int', ['int'])
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        console.log(`dlopen onEnter ${this.name}`)
        if (this.name != null && this.name.indexOf('目标so文件') >= 0) {
          sleep(10) // sleep10秒留出strace执行时机
        }
      }, onLeave: function(retval){
        console.log(`dlopen onLeave name: ${this.name}`)
      }
    })
  }

setImmediate(hookDlopen)

```

上述脚本执行之后，找到目标进程的 pid 之后执行命令`strace -e trace=process -i -f -p "目标pid"`，挂载系统函数的调用监视使用 strace 在进入目标检测 so 之前挂载上之后截取了目标进程的一系列系统调用，主要关注导致进程退出的系统调用，由下图可见是在进程下面的一个线程实现的闪退效果，执行了系统调用`exit_group(0 <unfinished ...>`。

同时查看了目标进程中的 libc 的地址也与导致闪退的地址不符，这就说明这个内容是动态加载的，故而会有一个 mmap 创建内存的操作。  
![](https://attach.52pojie.cn/forum/202511/29/164252vh4gvh39129u1ojj.jpg)

**3.jpg** _(615.75 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyNHwyODMxNjVjMnwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:42 上传

动态释放代码一定是要操作内存的，接下来我们用前面相同的逻辑，用 strace 查看调用了哪些和内存相关的系统调用，`strace -e trace=process,memory -i -f -p 目标pid`，如下图，mmap 一块儿内存之后执行了造成闪退的系统调用（注意：下图是再一次的启动进程打印出来的内容，故而地址与上图可能有点出入）。  
![](https://attach.52pojie.cn/forum/202511/29/164348isihcuytt5yxitkv.jpg)

**4.jpg** _(536.84 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyNXw5YTQ4MjUwYXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:43 上传

于是我们可以 hook mmap 打印一下调用栈，从而定位到在目标 so 文件中创建这块内存造成闪退的偏移地址：

```
function hook_mmap() {//打印一下mmap系统调用的调用栈，从而追踪调用源
  const mmap = Module.getExportByName("libc.so", "mmap");
  console.log("mmap address: " + mmap);
  Interceptor.attach(mmap, {
    onEnter: function (args) {
      let length = args[1].toString(16)
      if (parseInt(length, 16) == 28) {
        console.log('backtrace:\n' + Thread.backtrace(this.context, Backtracer.ACCURATE)
                                                  .map(DebugSymbol.fromAddress).join('\n') + '\n');
      }
    }
  })
}
function hookDlopen() {
    // 获取libc中的sleep函数
    let sleep = new NativeFunction(Module.getExportByName('libc.so', 'sleep'), 'int', ['int'])
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        console.log(`dlopen onEnter ${this.name}`)
        if (this.name != null && this.name.indexOf('目标so文件') >= 0) {
          //sleep(15) // sleep10秒留出strace执行时机
          hook_mmap()
        }
      }, onLeave: function(retval){
        console.log(`dlopen onLeave name: ${this.name}`)
      }
    })
  }

hookDlopen()

```

会发现能够打印 mmap 的地址，但是还没到打印使用该函数的调用栈的时候就挂掉了，此处我理解的是出现这种问题的原因是做 hook 的时机还是不对，我们的 hook 还是被检测到了。  
![](https://attach.52pojie.cn/forum/202511/29/164432jb2wx0boosircjx2.jpg)

**5.jpg** _(737.75 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyNnxiNGEwMzQzZXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:44 上传

因此对于 hook 的时机问题再探究一下，这里借用一张安卓中 so 文件的加载图来分析一下何时 hook 才能实现我们的目的：  
![](https://attach.52pojie.cn/forum/202511/29/164517n5oubooomup5lhbp.jpg)

**6.jpg** _(194.45 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyN3w4YzE4MTA4OXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

上图描述的更多的是一个流程，结合如下的安卓源码，我理解的是 do_dlopen 和 call_constructors 应该是包含关系，call_constructors 于 do_dlopen 的流程内执行。

```
void* do_dlopen(const char* name, int flags,
const android_dlextinfo* extinfo,
const void* caller_addr) {
    std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
    ScopedTrace trace(trace_prefix.c_str());
    ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
    soinfo* const caller = find_containing_library(caller_addr);
    android_namespace_t* ns = get_caller_namespace(caller);

    LD_LOG(kLogDlopen,
        "dlopen(name=\"%s\", flags=0x%x, extinfo=%s, caller=\"%s\", caller_ns=%s@%p, targetSdkVersion=%i) ...",
        name,
        flags,
        android_dlextinfo_to_string(extinfo).c_str(),
        caller == nullptr ? "(null)" : caller->get_realpath(),
        ns == nullptr ? "(null)" : ns->get_name(),
        ns,
        get_application_target_sdk_version());

    auto purge_guard = android::base::make_scope_guard([&]() { purge_unused_memory(); });

    auto failure_guard = android::base::make_scope_guard(
    [&]() { LD_LOG(kLogDlopen, "... dlopen failed: %s", linker_get_error_buffer()); });

    if ((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL|RTLD_NODELETE|RTLD_NOLOAD)) != 0) {
        DL_OPEN_ERR("invalid flags to dlopen: %x", flags);
        return nullptr;
    }

    if (extinfo != nullptr) {
        if ((extinfo->flags & ~(ANDROID_DLEXT_VALID_FLAG_BITS)) != 0) {
            DL_OPEN_ERR("invalid extended flags to android_dlopen_ext: 0x%" PRIx64, extinfo->flags);
            return nullptr;
        }

        if ((extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD) == 0 &&
            (extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET) != 0) {
            DL_OPEN_ERR("invalid extended flag combination (ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET without "
                "ANDROID_DLEXT_USE_LIBRARY_FD): 0x%" PRIx64, extinfo->flags);
            return nullptr;
        }

        if ((extinfo->flags & ANDROID_DLEXT_USE_NAMESPACE) != 0) {
            if (extinfo->library_namespace == nullptr) {
                DL_OPEN_ERR("ANDROID_DLEXT_USE_NAMESPACE is set but extinfo->library_namespace is null");
                return nullptr;
            }
            ns = extinfo->library_namespace;
        }
    }

    // Workaround for dlopen(/system/lib/<soname>) when .so is in /apex. http://b/121248172
    // The workaround works only when targetSdkVersion < Q.
    std::string name_to_apex;
    if (translateSystemPathToApexPath(name, &name_to_apex)) {
        const char* new_name = name_to_apex.c_str();
        LD_LOG(kLogDlopen, "dlopen considering translation from %s to APEX path %s",
            name,
            new_name);
        // Some APEXs could be optionally disabled. Only translate the path
        // when the old file is absent and the new file exists.
        // TODO(b/124218500): Re-enable it once app compat issue is resolved
        /*
      if (file_exists(name)) {
        LD_LOG(kLogDlopen, "dlopen %s exists, not translating", name);
      } else
      */
        if (!file_exists(new_name)) {
        LD_LOG(kLogDlopen, "dlopen %s does not exist, not translating",
               new_name);
      } else {
        LD_LOG(kLogDlopen, "dlopen translation accepted: using %s", new_name);
        name = new_name;
      }
    }
    // End Workaround for dlopen(/system/lib/<soname>) when .so is in /apex.

    std::string translated_name_holder;

    assert(!g_is_hwasan || !g_is_asan);
    const char* translated_name = name;
    if (g_is_asan && translated_name != nullptr && translated_name[0] == '/') {
      char original_path[PATH_MAX];
      if (realpath(name, original_path) != nullptr) {
        translated_name_holder = std::string(kAsanLibDirPrefix) + original_path;
        if (file_exists(translated_name_holder.c_str())) {
          soinfo* si = nullptr;
          if (find_loaded_library_by_realpath(ns, original_path, true, &si)) {
            PRINT("linker_asan dlopen NOT translating \"%s\" -> \"%s\": library already loaded", name,
                  translated_name_holder.c_str());
          } else {
            PRINT("linker_asan dlopen translating \"%s\" -> \"%s\"", name, translated_name);
            translated_name = translated_name_holder.c_str();
          }
        }
      }
    } else if (g_is_hwasan && translated_name != nullptr && translated_name[0] == '/') {
      char original_path[PATH_MAX];
      if (realpath(name, original_path) != nullptr) {
        // Keep this the same as CreateHwasanPath in system/linkerconfig/modules/namespace.cc.
        std::string path(original_path);
        auto slash = path.rfind('/');
        if (slash != std::string::npos || slash != path.size() - 1) {
          translated_name_holder = path.substr(0, slash) + "/hwasan" + path.substr(slash);
        }
        if (!translated_name_holder.empty() && file_exists(translated_name_holder.c_str())) {
          soinfo* si = nullptr;
          if (find_loaded_library_by_realpath(ns, original_path, true, &si)) {
            PRINT("linker_hwasan dlopen NOT translating \"%s\" -> \"%s\": library already loaded", name,
                  translated_name_holder.c_str());
          } else {
            PRINT("linker_hwasan dlopen translating \"%s\" -> \"%s\"", name, translated_name);
            translated_name = translated_name_holder.c_str();
          }
        }
      }
    }
    ProtectedDataGuard guard;
    soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);
    loading_trace.End();

    if (si != nullptr) {
      void* handle = si->to_handle();
      LD_LOG(kLogDlopen,
             "... dlopen calling constructors: realpath=\"%s\", soname=\"%s\", handle=%p",
             si->get_realpath(), si->get_soname(), handle);
      si->call_constructors();//！！！重点看这里！！！
      failure_guard.Disable();
      LD_LOG(kLogDlopen,
             "... dlopen successful: realpath=\"%s\", soname=\"%s\", handle=%p",
             si->get_realpath(), si->get_soname(), handle);
      return handle;
    }

    return nullptr;
}```

跟进到call_constructors函数之后会发现会先调用`.init_xxxx`函数，接着调用`.init_array`中的函数。







7.jpg (198.17 KB, 下载次数: 0)

下载附件



2025-11-29 16:45 上传








故而我们的hook应该更精确到call_constructors，所以可以把钩子挂载在call_constructors函数处，来执行我们对目标so文件中mmap系统调用的拦截：

```javascript
function hook_mmap() {//打印一下mmap系统调用的调用栈，从而追踪调用源
  const mmap = Module.getExportByName("libc.so", "mmap");
  console.log("mmap address: " + mmap);
  Interceptor.attach(mmap, {
    onEnter: function (args) {
      let length = args[1].toString(16)
      if (parseInt(length, 16) == 28) {//过滤到28字节的创建
        console.log('backtrace:\n' + Thread.backtrace(this.context, Backtracer.ACCURATE)
                                                  .map(DebugSymbol.fromAddress).join('\n') + '\n');
      }
    }
  })
}
function hook_linker_call_constructors() {
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x52110//此处偏移的获取使用命令: readelf -sW /apex/com.android.runtime/bin/linker64 | grep call_constructors
    let call_constructors = linker64_base_addr.add(offset)
    let listener = Interceptor.attach(call_constructors,{
        onEnter:function(args){
            console.log('hook_linker_call_constructors onEnter')
            let secmodule = Process.findModuleByName("目标so文件")
            if (secmodule != null){
                // do something
                hook_mmap()
            }
        }
    })
}
hook_linker_call_constructors();

```

结果如下，成功得到了崩溃前对应的 mmap 函数的调用栈。  
![](https://attach.52pojie.cn/forum/202511/29/164523rdtt4rrgdsztg064.jpg)

**8.jpg** _(476.33 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzcyOXw1OGRiMDY1OHwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

### 0x02 静态定位分析

那么现在我们能够通过上述最后 hook 到的 mmap 调用的偏移定位到目标 so 中可能存在检测的位置，从而一步步往上交叉引用来进行分析，由调用栈的最顶层 0x2358C 可以跟到下图这个位置，可以大概看出似乎是在解密一段 shellcode 然后开辟内存来执行。  
![](https://attach.52pojie.cn/forum/202511/29/164525lwkq2ggbwrbfww4o.jpg)

**9.jpg** _(140.65 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczMHwzYzhiMDA4MXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

使用脚本将此执行的 shellcode dump 下来之后选择 arm64 架构进行反编译之后发现还是一段跳板，有跳转到指定地址执行的逻辑，看来是为了防止直接调用系统函数被轻易追踪到而设计。  
由函数 sub_234E0 一路交叉引用上去，其上一层有一段检查 dex 文件完整性的逻辑。  
![](https://attach.52pojie.cn/forum/202511/29/164527kwieh0bbmkqkllg3.jpg)

**10.jpg** _(119.08 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczMXxkMWZmZDM1ZHwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

![](https://attach.52pojie.cn/forum/202511/29/164530hu2e6duyf7mug2d2.jpg)

**11.jpg** _(244.66 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczMnxhYTEwZGE5ZHwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

继续由函数 sub_11FA4 往上回溯，发现于函数 sub_19E0C 中出现一段无限循环持续检测设备是否处于 ADB 调试模式的逻辑，当在 sub_19E0C 函数中检测到 ADB 调试模式时，会调用 sub_17C8C 和 sub_19A58 这两个函数进行进一步的环境验证，如若被检测出来了的话直接走我们刚刚交叉引用上来的闪退逻辑。  
![](https://attach.52pojie.cn/forum/202511/29/164533rarwya1l05w5hsas.jpg)

**12.jpg** _(186.61 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczM3xhOGVlMDA5MHwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

继续往上回溯，发现下面第二张图中调用了 sub_19E0C，但是调用的方法有点奇怪，是作为 v24 函数的参数进行使用的，同时该段通过动态解密的操作，使用解密后的第一个字符串作为库名，调用 dlopen 动态加载该库，使用解密后的第二个字符串作为函数名，调用 dlsym 获取该函数地址，同时最后获取到的函数使用了 sub_1B8D4 和 sub_19E0C 作为参数，sub_19E0C 是我们回溯上来的引线。  
![](https://attach.52pojie.cn/forum/202511/29/164535e5xooh38e8h854yo.jpg)

**13.jpg** _(126.02 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczNHxjNTExODUwMnwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

![](https://attach.52pojie.cn/forum/202511/29/164538h4o9cl4um6e66t4r.jpg)

**14.jpg** _(111.98 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczNXw2MjBiNjNlYnwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

所以这里要搞懂这个函数中做了一件什么事，就得知道它调用 dlsym 获取了哪个函数的地址，从而进行使用，使用下列 frida 的脚本进行 hook，同样是在 call_constructors 处做挂载：

```
function hook_do_dlsym() {
    // 1. 获取 linker64 基址
    const linker = Process.arch === "arm64" ? "linker64" : "linker";
    const base = Module.getBaseAddress(linker);
    if (!base) {
        console.log("[!] Cannot find linker base.");
        return;
    }
    // 2. do_dlsym 偏移
    const offset = 0x3c5b0;//使用命令readelf -sW /apex/com.android.runtime/bin/linker64 | grep do_dlsym
    const do_dlsym_addr = base.add(offset);
    // 3. Hook 函数
    Interceptor.attach(do_dlsym_addr, {
        onEnter(args) {
            try {
                // args[1] 是符号名 char*
                let symPtr = args[1];
                let symname = symPtr.readCString();

                console.log(`[do_dlsym] symbol = ${symname}`);
            } catch (e) {
                console.log("[error] readCString failed:", e);
            }
        }
    });
}
function hookdoDlopen() {
    // 获取libc中的sleep函数
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x3ba00 // __dl__Z9do_dlopenPKciPK17android_dlextinfoPKv
    let android_dlopen_ext = linker64_base_addr.add(offset)
    Interceptor.attach(android_dlopen_ext, {
      onEnter: function(args){
        this.name = args[0].readCString()
        console.log(`dlopen onEnter ${this.name}`)
      }, onLeave: function(retval){
        console.log(`dlopen onLeave name: ${this.name}`)
      }
    })
  }
function hook_linker_call_constructors() {
    let linker64_base_addr = Module.getBaseAddress('linker64')
    let offset = 0x52110//此处偏移的获取使用命令: readelf -sW /apex/com.android.runtime/bin/linker64 | grep call_constructors
    let call_constructors = linker64_base_addr.add(offset)
    let listener = Interceptor.attach(call_constructors,{
        onEnter:function(args){
            let secmodule = Process.findModuleByName("目标so文件")
            if (secmodule != null){
                // do something
                hookdoDlopen()
                hook_do_dlsym()
            }
        }
    })
}

hook_linker_call_constructors();

```

得到结果如下，很明显那么上面那段函数就是去动态获取了 libc.so 库中的地址，读取了函数 pthread_create 的地址，然后创建了两个线程，最后都有执行到闪退函数 sub_234E0 的逻辑。  
![](https://attach.52pojie.cn/forum/202511/29/164540iaudus0msrmamgsw.jpg)

**15.jpg** _(300.65 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczNnxhZWQxZDE0OHwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

那么这段逻辑就稍微明了了，sub_1B924 中动态解密了库名 libc.so 和函数名 pthread_create，开创了两个检测的线程。  
![](https://attach.52pojie.cn/forum/202511/29/164543x7i0fflfgr9gtxiz.jpg)

**16.jpg** _(179.23 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczN3wxOGUzNTNiYXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

对此处几个关键函数进行分析之后可知如下图，其中标红的三个函数中都有开创新线程执行持续扫描一些特征的操作：  
![](https://attach.52pojie.cn/forum/202511/29/164545urjjwqmrrwcnw4v1.jpg)

**17.jpg** _(219.41 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczOHw5N2JmMGU2M3wxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

那么此时我们继续沿着上图有三处创建新线程的框架函数继续往上回溯，发现到达了 **init_proc**，也就是 so 加载之后走的第一个流程块，其中框出来的即是我们上图中的内容，因此我们可以根据造成程序异常退出的这三处创建处以及根据创建线程必走的一些系统调用来 bypass 掉这个检测。  
![](https://attach.52pojie.cn/forum/202511/29/164548n3gb34gfeffb3m3z.jpg)

**18.jpg** _(195.85 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzczOXwzM2RhZmZiOXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

### 0x03 bypass 手段

由上述分析我们可以得知，三处创建线程逻辑的地址，分别于 sub_1BEC4 函数中建立，根据上文中的详细描述可以锁定到这几处：  
![](https://attach.52pojie.cn/forum/202511/29/164551s6k7ghno6kvghhqo.jpg)

**19.jpg** _(288.46 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0MHxlNDhhNzZmYnwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

![](https://attach.52pojie.cn/forum/202511/29/164553jgxz6ubwxgp1bg91.jpg)

**20.jpg** _(120.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0MXwzOTU5OGE2NnwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

可以知道创建线程之后执行的函数分别是 sub_1B8D4、sub_19E0C、sub_1C544，故而可以直接将几个线程执行的函数直接 patch 掉，实现绕过，但是是有下列脚本之后还是被杀掉了：

```
const patchedOffsets = new Set();
function nopFunctionEntry(module, offset) {
    const addr = module.base.add(offset);
    if (patchedOffsets.has(offset)) {
        console.log("[nopFunctionEntry] already patched 0x" + offset.toString(16));
        return;
    }
    try {
        Memory.patchCode(addr, 4, code => {
            const writer = new Arm64Writer(code, { pc: addr });
            writer.putRet();
            writer.flush();
        });
        patchedOffsets.add(offset);
        console.log("[nopFunctionEntry] patched 0x" + offset.toString(16));
    } catch (e) {
        console.log("[nopFunctionEntry] patch failed 0x" + offset.toString(16) + " -> " + e);
    }
}
function patch_target_addr() {
    const module = Process.findModuleByName("目标so文件.so");
    if (!module) return;
    nopFunctionEntry(module, 0x1B8D4);
    nopFunctionEntry(module, 0x19E0C);        //对应的三个线程执行的函数
    nopFunctionEntry(module, 0x1C544);
}

function hook_linker_call_constructors() {
    const linker64_base_addr = Module.getBaseAddress('linker64');
    if (!linker64_base_addr) {
        console.log("linker64 not found");
        return;
    }
    const offset = 0x52110;
    const call_constructors = linker64_base_addr.add(offset);
    Interceptor.attach(call_constructors, {
        onEnter: function (args) {
            const secmodule = Process.findModuleByName("目标so文件.so");
            if (secmodule) {
                console.log('hook_linker_call_constructors onEnter 目标so文件');
                patch_target_addr();
            }
        }
    });
}

hook_linker_call_constructors();

```

![](https://attach.52pojie.cn/forum/202511/29/164555ffje6cjt769c99t0.jpg)

**21.jpg** _(96.9 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0MnxjMWVlZDVmMnwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

所以还是有点问题，我们继续分析，发现 sub_1B8D4 和 sub_19E0C 两个函数最终都会调用 **sub_234E0** 这个函数来实现闪退，那么我们直接 patch 掉这个函数不就等同于对这两个线程函数检测的 bypass 了嘛。  
![](https://attach.52pojie.cn/forum/202511/29/164558zz4q612x22ztoox4.jpg)

**22.jpg** _(144.76 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0M3xlMzJmNDVhZXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:45 上传

然后发现在线程函数 sub_1C544 中对应执行类似的持续扫描闪退的操作的函数应该是 sub_26334 函数。  
![](https://attach.52pojie.cn/forum/202511/29/164600xp8ud0ksuuixgs9g.jpg)

**23.jpg** _(119.7 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0NHw2YTQzM2M5MXwxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:46 上传

![](https://attach.52pojie.cn/forum/202511/29/164602uv3ia3pfk8vdkyy0.jpg)

**24.jpg** _(135.86 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0NXxjMDM2NjU0N3wxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:46 上传

有此重大发现之后可以将 bypass 的脚本更新为如下，如此可以最小程度上的对程序进行修改的同时实现 bypass：

```
const patchedOffsets = new Set();
function nopFunctionEntry(module, offset) {
    const addr = module.base.add(offset);
    if (patchedOffsets.has(offset)) {
        console.log("[nopFunctionEntry] already patched 0x" + offset.toString(16));
        return;
    }
    try {
        Memory.patchCode(addr, 4, code => {
            const writer = new Arm64Writer(code, { pc: addr });
            writer.putRet();
            writer.flush();
        });
        patchedOffsets.add(offset);
        console.log("[nopFunctionEntry] patched 0x" + offset.toString(16));
    } catch (e) {
        console.log("[nopFunctionEntry] patch failed 0x" + offset.toString(16) + " -> " + e);
    }
}

function patch_target_addr() {
    const module = Process.findModuleByName("目标so文件.so");
    if (!module) return;
    nopFunctionEntry(module, 0x234E0);//直接造成检测闪退的地址
    nopFunctionEntry(module, 0x26334);//直接造成检测闪退的地址
}

function hook_linker_call_constructors() {
    const linker64_base_addr = Module.getBaseAddress('linker64');
    if (!linker64_base_addr) {
        console.log("linker64 not found");
        return;
    }
    const offset = 0x52110;
    const call_constructors = linker64_base_addr.add(offset);
    Interceptor.attach(call_constructors, {
        onEnter: function (args) {
            const secmodule = Process.findModuleByName("目标so文件.so");
            if (secmodule) {
                console.log('hook_linker_call_constructors onEnter 目标so文件');
                patch_target_addr();
            }
        }
    });
}

hook_linker_call_constructors();

```

可以成功绕过检测：

  
![](https://attach.52pojie.cn/forum/202511/29/164605ocy0u0cugcgt1r1g.jpg)

**25.jpg** _(184.95 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNzc0NnxjNWY3MDAzM3wxNzY0ODk2MzE2fDIxMzQzMXwyMDc2NTA2&nothumb=yes)

2025-11-29 16:46 上传

### 0x04 总结

此文分析仅代表个人见解，希望师傅们多多指正捏~

参考文章：[参考文章 1](https://xiaoeeyu.github.io/2024/08/09/%E7%BB%95%E8%BF%87%E7%88%B1%E5%A5%87%E8%89%BAlibmsaoaidsec-so%E7%9A%84Frida%E6%A3%80%E6%B5%8B/index.html)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)heigui520 希望佬分析分析广告 sdk![](https://avatar.52pojie.cn/images/noavatar_middle.gif)tcdgy1001 希望佬分析分析广告 sdk ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 慵懒丶 L 先森 感谢分享，老实说这篇文章涵盖的知识点挺多的，看懂理解实操学会了基本上已经能干掉市面上一半以上的 SO 验证对抗了![](https://static.52pojie.cn/static/image/smiley/laohu/laohu3.gif)学无止境 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) rainforestyu 感谢大佬, 虽然感觉过于高深, 但是思路还是学习到了..![](https://avatar.52pojie.cn/images/noavatar_middle.gif)pplus 感觉很高深，学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zorua 学到真东西了，感谢大佬 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) saiksy 打开了眼界，难怪老是闪退 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) oneday11111 学习了，感谢大佬