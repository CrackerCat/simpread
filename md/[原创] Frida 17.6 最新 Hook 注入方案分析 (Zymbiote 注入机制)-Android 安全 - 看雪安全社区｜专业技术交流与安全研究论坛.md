> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289980.htm)

> [原创] Frida 17.6 最新 Hook 注入方案分析 (Zymbiote 注入机制)

[原创] Frida 17.6 最新 Hook 注入方案分析 (Zymbiote 注入机制)

发表于: 13 小时前 229

[举报](javascript:void(0);)

### [原创] Frida 17.6 最新 Hook 注入方案分析 (Zymbiote 注入机制)

 [![](http://passport.kanxue.com/upload/avatar/584/994584.png?1762403320)](user-home-994584.htm) [Elenia](user-home-994584.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png) 7  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 13 小时前  229

> 本文主要是想要了解清楚 Frida 在新版本有什么地方不一样了。本文主要还是着重在 Zymbiote 的注入机制上，其他相关板块后续再出文章分析。
> 
> 然后在阅读 Frida 源码的过程也发现了很多有意思的小 trick，鉴于本人水平有限如果有问题请各位大佬指出。
> 
> 版本说明:
> 
> Frida 17.6
> 
> 参考文章:
> 
> *   https://mp.weixin.qq.com/s?__biz=MzU3MTY5MzQxMA==&mid=2247485002&idx=1&sn=1f0e7077f192703dbc07935a63113980&chksm=fd3f4571a6363e6f9368aa180876c6027ca25f049b52030539adae7d4924b9874905c78164d1&mpshare=1&scene=23&srcid=0121HWdNu3KCeBsfOR7dswf6&sharer_shareinfo=09579c27053b7a0aa2edf48906540280&sharer_shareinfo_first=ca990fdd18b19aad1ba9c8738149bfdb#rd

目录
--

```
1. 前言
   1.1 版本说明
   1.2 参考文章
 
2. 历史Hook方案
   2.1 旧方案问题
   2.2 新方案
 
3. 流程图
 
4. Zymbiote注入阶段
   4.1 函数调用链
   4.2 补丁
       4.2.1 Payload补丁
       4.2.2 ArtMethod补丁
   4.3 Boot Heap
       4.3.1 Boot Image文件
   4.4 ensure_loaded (初始化)
   4.5 inject_zymbiote (核心注入逻辑)
   4.6 do_prepare_zymbiote_injection(核心准备逻辑)
       4.6.1 调用链
       4.6.2 初始化变量
       4.6.3 解析进程内存映射
       4.6.4 获取libc与runtime信息
       4.6.5 定位ArtMethod信息
       4.6.6 填充payload

```

历史 Hook 方案
----------

### 旧方案问题

*   需要 Zygote 内注入 agent
    *   需要 ptrace 暂停 / 恢复线程
    *   在所有子进程中留下痕迹
    *   需要隐藏 FD，否则 Zygote 会 abort
*   依赖于 syscall tracing
    *   Ptrace 的 syscall tracing 逻辑复杂且脆弱
*   Teardown 路径不安全
    *   难以保证在各种时机正确回收
    *   容易在 Zygote 和 system_server 中造成不稳定

### 新方案

*   完全外部化
    *   不注入 Zygote agent
    *   通过 /proc/$pid/mem 写入小型 payload
    *   不使用 ptrace
*   短生命周期设计
    *   Payload 仅负责握手和暂停
    *   执行后立即回滚，不留常驻代码
*   利用稳定入口点
    *   使用 android.os.Process.setArgV0Native() 作为触发点
    *   该函数在子进程启动时必然被调用

流程图
---

![](https://bbs.kanxue.com/upload/attach/202602/994584_T2XQP4FDPFBD2CW.webp)

Zymbiote 注入阶段
-------------

> Zymbiote 是 Frida 在 Android 上实现轻量级 Zygote Hook 的机制，用于在应用进程启动时注入 Frida
> 
> 命名来源：Zymbiote = Zygote + Symbiote（共生体），表示与 Zygote 进程共生。

### 函数调用链

```
enable_spawn_gating() (linux-host-session.vala:261)
  └─ robo_launcher.enable_spawn_gating() (linux-host-session.vala:263)
      └─ ensure_loaded() (linux-host-session.vala:661)
          ├─ 创建 Unix Socket 服务器
          ├─ handle_zymbiote_connections.begin() (linux-host-session.vala:785)
          └─ 枚举 Zygote 进程
              └─ do_inject_zygote_agent.begin() (linux-host-session.vala:816)
                  └─ inject_zymbiote() (linux-host-session.vala:838)
                      ├─ prepare_zymbiote_injection() (linux-host-session.vala:847)
                      │   └─ do_prepare_zymbiote_injection() (linux-host-session.vala:905)
                      │       ├─ 解析 /proc/<pid>/maps
                      │       ├─ 定位 payload 基址 (libstagefright.so)
                      │       ├─ 解析 libc.so 和 libandroid_runtime.so
                      │       ├─ 定位 ArtMethod 槽位
                      │       └─ 准备 payload 数据
                      ├─ 暂停 Zygote 进程 (SIGSTOP)
                      ├─ 应用补丁 (ZymbiotePatches.apply)
                      │   ├─ 写入 payload 到可执行映射
                      │   └─ 替换 ArtMethod 槽位指针
                      └─ 恢复 Zygote 进程 (SIGCONT)

[子进程 fork 后]
  └─ frida_zymbiote_replacement_set_argv0() (zymbiote.c:44)
      ├─ 回滚 ArtMethod 槽位
      ├─ 调用原始 setArgV0Native
      ├─ 连接 Unix Socket
      ├─ 发送 Hello 消息
      ├─ 等待 ACK
      └─ 触发 SIGSTOP

[Frida Core 处理]
  └─ handle_zymbiote_connection() (linux-host-session.vala:1128)
      ├─ read_hello() (linux-host-session.vala:1130)
      ├─ 判断是否需要 spawn gating
      └─ connection.resume() (linux-host-session.vala:1150)
          ├─ 发送 ACK
          ├─ 等待进程停止
          ├─ 回滚子进程补丁
          └─ 发送 SIGCONT


```

### 补丁

> 后文会出现补丁相关的代码, 所以这里解释一下。其实很好理解, 这里补丁就是我们 patch 上去的代码
> 
> 补丁目前分为两种类型

#### Payload 补丁

> 这里很好理解，payload 相当于我们传入的二进制代码，然后我们在 ArtMethod 改对应属性指向我们的 payload，这样他们执行函数的时候就会先执行我们的 payload
> 
> *   内容：编译后的 zymbiote 二进制代码（约 800-900 字节）
> *   位置：libstagefright.so 的最后一页（可执行映射）（这里为什么选择这个 so 捏，因为这是系统库，通常已加载）
> *   作用：在目标进程中放置可执行代码

```
// 将 payload 代码写入到可执行内存
patches.apply(payload, process_memory, payload_base);


```

#### ArtMethod 补丁

> *   内容：指针值（4 或 8 字节）
> *   位置：boot heap 中的 ArtMethod 槽位
> *   作用：将函数指针从原始函数改为指向 payload

```
// 替换 ArtMethod 槽位指针
patches.apply(replaced_ptr, process_memory, art_method_slot);


```

### Boot Heap

> 这个概念在后面也会出现，我们这里可以解释一下。
> 
> Boot Heap = Android ART 运行时在启动时预加载的内存区域。

```
Boot Heap 存储:
├─ 预编译的 Android 框架类
├─ ArtMethod 对象（包括 setArgV0Native 的 ArtMethod）
├─ 预加载的类元数据
└─ 系统库的方法信息
private static bool is_boot_heap (string path) {
    return
        "boot.art" in path ||
        "boot-framework.art" in path ||
        "dalvik-LinearAlloc" in path;
}


```

我们从代码可以知道，系统判定为 boot_heap 就是判断路径是否包含 boot.art，boot-framework.art，dalvik-LinearAlloc 这些 Boot Image

#### Boot Image 文件

> 关于 Boot Image 我们可以后续再出一个文章再详细介绍这个地方。
> 
> Android 启动时:
> 
> 1.  Zygote 进程启动
> 2.  加载 boot.art → 映射到内存 (boot heap)
> 3.  加载 boot-framework.art → 映射到内存 (boot heap)
> 4.  预编译的类和方法信息都在这里
> 5.  ArtMethod 对象也存储在这里 ✅

```
boot.art:
├─ 预编译的核心 Android 类
├─ 系统库的预编译代码
└─ 启动时加载到内存

boot-framework.art:
├─ Android 框架的预编译类
├─ 系统服务的预编译代码
└─ 启动时加载到内存

/proc/<pid>/maps 输出:

7f1234000000-7f1235000000 rw-p 00000000 /dev/ashmem/dalvik-LinearAlloc
7f1235000000-7f1236000000 rw-p 00000000 /system/framework/boot.art
7f1236000000-7f1237000000 rw-p 00000000 /system/framework/boot-framework.art


```

### ensure_loaded (初始化)

> 统一初始化入口，确保 Socket 服务器就绪，并发注入多个 Zygote 进程

*   这里 wait_async 再返回就是避免让程序以为是返回成功了的，我们等待其他实列完成。
    
*   创建 Unix Socket 服务器
    
    *   生成随机名称：/frida-zymbiote-{uuid}
    *   使用抽象命名空间（UnixSocketAddressType.ABSTRACT）
    *   启动监听，等待 Payload 连接
*   枚举并注入所有 Zygote 进程
    
    *   查找 zygote、zygote64、usap32、usap64 (Unspecialized APP Process 也就是 Android 的预 fork 进程池机制，用于加速应用启动) 这样尽可能遍历到了所有进程类型，只要有一个新的进程产生就一定会被 hook，因为父进程中相关的函数已经被我们修改了。这里的实现非常优雅
        
        *   ```
            传统方式（无USAP）:
            应用启动请求 → Zygote fork → 初始化 → 应用运行
                          ↑ 耗时较长
            
            USAP方式:
            Zygote 预fork → USAP池 (空闲进程)
            应用启动请求 → 从USAP池取进程 → 快速配置 → 应用运行
                          ↑ 更快
            
            
            ```
            
    *   对每个进程调用 do_inject_zygote_agent()
        
    *   使用并发注入，等待所有完成
        

```
private async void ensure_loaded (Cancellable? cancellable) throws Error, IOError {
    while (ensure_request != null) {
        try {
            
            yield ensure_request.future.wait_async (cancellable);
            return;
        } catch (Error e) {
            throw e;
        } catch (IOError e) {
            cancellable.set_error_if_cancelled ();
        }
    }
    ensure_request = new Promise<bool> ();

    if (server_name == null) {
        string name = "/frida-zymbiote-" + Uuid.string_random ().replace ("-", "");
        var address = new UnixSocketAddress.with_type (name, -1, UnixSocketAddressType.ABSTRACT);

        try {
            var socket = new Socket (SocketFamily.UNIX, SocketType.STREAM, SocketProtocol.DEFAULT);
            socket.bind (address, true);
            socket.listen ();

            server_name = name;
            server_address = address;

            handle_zymbiote_connections.begin (socket);
        } catch (GLib.Error raw_err) {
            var err = new Error.TRANSPORT ("%s", raw_err.message);
            ensure_request.reject (err);
            throw err;
        }
    }

    uint pending = 1;
    GLib.Error? first_error = null;

    CompletionNotify on_complete = error => {
        pending--;
        if (error != null && first_error == null)
            first_error = error;

        if (pending == 0) {
            var source = new IdleSource ();
            source.set_callback (ensure_loaded.callback);
            source.attach (MainContext.get_thread_default ());
        }
    };

    foreach (HostProcessInfo info in System.enumerate_processes (new ProcessQueryOptions ())) {
        var name = info.name;
        if (name == "zygote" || name == "zygote64" || name == "usap32" || name == "usap64") {
            uint pid = info.pid;
            if (zymbiote_patches.has_key (pid))
                continue;

            pending++;
            do_inject_zygote_agent.begin (pid, name, cancellable, on_complete);
        }
    }

    on_complete (null);

    yield;

    on_complete = null;

    if (first_error == null) {
        ensure_request.resolve (true);
    } else {
        ensure_request.reject (first_error);
        ensure_request = null;

        throw_api_error (first_error);
    }
}


```

### inject_zymbiote (核心注入逻辑)

> 我们可以看见 do_inject_zygote_agent 包装了我们的 inject_zymbiote

*   准备注入数据
    *   调用 prepare_zymbiote_injection() 获取 ZymbiotePrepResult
    *   包含 payload、内存地址、ArtMethod 槽位等
*   暂停目标进程
    *   发送 SIGSTOP 暂停 Zygote
    *   等待进程停止，确保内存写入安全
*   应用补丁
    *   创建 ZymbiotePatches 管理补丁
    *   处理已补丁情况：
    *   若 already_patched，从文件读取原始内容
    *   写入 payload 到可执行映射（payload_base）
    *   替换 ArtMethod 槽位指针为 payload 地址
    *   保存补丁记录到 zymbiote_patches[pid]
*   恢复进程
    *   finally 中发送 SIGCONT 恢复执行

```
private async void do_inject_zygote_agent (uint pid, string name, Cancellable? cancellable, CompletionNotify on_complete) {
    try {
        yield inject_zymbiote (pid, cancellable);

        on_complete (null);
    } catch (GLib.Error e) {
        on_complete (e);
    }
}
private async void inject_zymbiote (uint pid, Cancellable? cancellable) throws Error, IOError {
    // 调用 prepare_zymbiote_injection() 获取 ZymbiotePrepResult
    var prep = yield prepare_zymbiote_injection (pid, cancellable);
    // 发送 SIGSTOP 信号，暂停进程
    Posix.kill ((Posix.pid_t) pid, Posix.Signal.STOP);
    yield wait_until_stopped (pid, cancellable);
    try {
        // 创建 ZymbiotePatches 管理补丁
        var patches = new ZymbiotePatches ();
        // 获取 payload 数据
        unowned uint8[] payload = prep.payload.get_data ();
        // 如果已经补丁，则打开 payload 文件，读取原始数据
        if (prep.already_patched) {
            var handle = Posix.open (prep.payload_path, Posix.O_RDONLY);
            if (handle == -1) {
                throw new Error.PERMISSION_DENIED ("Unable to open payload backing file: %s",
                    strerror (errno));
            }
            var backing_file = new FileDescriptor (handle);
            var original = new uint8[payload.length];
            backing_file.pread_all (original, prep.payload_file_offset);
            //写入 payload 到可执行映射（payload_base）
            patches.apply (payload, prep.process_memory, prep.payload_base, new Bytes.take ((owned) original));
        } else {
            patches.apply (payload, prep.process_memory, prep.payload_base);
        }

        if (prep.already_patched) {
            // 替换 ArtMethod 槽位指针为 payload 地址
            patches.apply (prep.replaced_ptr, prep.process_memory, prep.art_method_slot,
                new Bytes (prep.original_ptr));
        } else {
            patches.apply (prep.replaced_ptr, prep.process_memory, prep.art_method_slot);
        }
        // 保存补丁记录到 zymbiote_patches[pid]
        zymbiote_patches[pid] = patches;
    } finally {
        Posix.kill ((Posix.pid_t) pid, Posix.Signal.CONT);
    }
}


```

### do_prepare_zymbiote_injection(核心准备逻辑)

> 这里算得上核心注入最核心的部分，也就是解析进程内存映射，解析符号地址，定位 ArtMethod 槽位，准备 Payload 数据。

```
private async ZymbiotePrepResult prepare_zymbiote_injection (uint pid, Cancellable? cancellable) throws Error, IOError {
    var task = new Task (this, cancellable, (obj, res) => {
        prepare_zymbiote_injection.callback ();
    });
    task.set_task_data ((void *) pid, null);
    task.run_in_thread ((t, source_object, task_data, c) => {
        unowned RoboLauncher launcher = (RoboLauncher) t.get_unowned_source_object ();
        uint pid_to_prep = (uint) task_data;
        try {
            var r = do_prepare_zymbiote_injection (pid_to_prep, launcher.server_name);
            t.return_pointer ((owned) r, Object.unref);
        } catch (GLib.Error e) {
            t.return_error ((owned) e);
        }
    });
    yield;

    try {
        return (ZymbiotePrepResult) (owned) task.propagate_pointer ();
    } catch (GLib.Error e) {
        throw_api_error (e);
    }
}
private static ZymbiotePrepResult do_prepare_zymbiote_injection (uint pid, string server_name) throws Error, IOError {
    // 进行一些变量的初始化
    uint64 payload_base = 0;
    string? payload_path = null;
    uint64 payload_file_offset = 0;
    string? libc_path = null;
    string? runtime_path = null;
    Gee.List<Gum.MemoryRange?> heap_candidates = new Gee.ArrayList<Gum.MemoryRange?> ();

    var iter = ProcMapsIter.for_pid (pid);
    while (iter.next ()) {
        string path = iter.path;
        string flags = iter.flags;
        if (path.has_suffix ("/libstagefright.so") && "x" in flags) {
            if (payload_base == 0) {
                payload_base = iter.end_address - Gum.query_page_size ();
                payload_path = path;
                payload_file_offset = iter.file_offset;
            }
        } else if (path.has_suffix ("/libc.so")) {
            if (libc_path == null)
                libc_path = path;
        } else if (path.has_suffix ("/libandroid_runtime.so")) {
            if (runtime_path == null)
                runtime_path = path;
        } else if (flags == "rw-p" && is_boot_heap (path)) {
            uint64 start = iter.start_address;
            uint64 end = iter.end_address;
            heap_candidates.add (Gum.MemoryRange () {
                base_address = start,
                size = (size_t) (end - start),
            });
        }
    }

    if (payload_base == 0)
        throw new Error.NOT_SUPPORTED ("Unable to pick a payload base");
    if (libc_path == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libc.so path");
    if (runtime_path == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libandroid_runtime.so path");
    if (heap_candidates.is_empty)
        throw new Error.NOT_SUPPORTED ("Unable to detect any VM heap candidates");

    var libc_entry = ProcMapsSoEntry.find_by_path (pid, libc_path);
    if (libc_entry == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libc.so entry");

    var runtime_entry = ProcMapsSoEntry.find_by_path (pid, runtime_path);
    if (runtime_entry == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libandroid_runtime.so entry");

    Gum.ElfModule libc;
    try {
        libc = new Gum.ElfModule.from_file (libc_path);
    } catch (Gum.Error e) {
        throw new Error.NOT_SUPPORTED ("Unable to parse libc.so: %s", e.message);
    }

    Gum.ElfModule runtime;
    try {
        runtime = new Gum.ElfModule.from_file (runtime_path);
    } catch (Gum.Error e) {
        throw new Error.NOT_SUPPORTED ("Unable to parse libandroid_runtime.so: %s", e.message);
    }

    uint64 set_argv0_address = 0;
    runtime.enumerate_exports (e => {
        if (e.name == "_Z27android_os_Process_setArgV0P7_JNIEnvP8_jobjectP8_jstring") {
            set_argv0_address = runtime_entry.base_address + e.address;
            return false;
        }
        return true;
    });
    if (set_argv0_address == 0)
        throw new Error.NOT_SUPPORTED ("Unable to locate android.os.Process.setArgV0(); please file a bug");

    uint pointer_size = ("/lib64/" in libc_path) ? 8 : 4;

    var original_ptr = new uint8[pointer_size];
    var replaced_ptr = new uint8[pointer_size];

    (new Buffer (new Bytes.static (original_ptr), ByteOrder.HOST, pointer_size)).write_pointer (0, set_argv0_address);
    (new Buffer (new Bytes.static (replaced_ptr), ByteOrder.HOST, pointer_size)).write_pointer (0, payload_base);

    var fd = open_process_memory (pid);

    uint64 art_method_slot = 0;
    bool already_patched = false;
    foreach (var candidate in heap_candidates) {
        var heap = new uint8[candidate.size];
        var n = fd.pread (heap, candidate.base_address);
        if (n != heap.length)
            throw new Error.NOT_SUPPORTED ("Short read");

        void * p = memmem (heap, original_ptr);
        if (p == null) {
            p = memmem (heap, replaced_ptr);
            already_patched = p != null;
        }

        if (p != null) {
            art_method_slot = candidate.base_address + ((uint8 *) p - (uint8 *) heap);
            break;
        }
    }
    if (art_method_slot == 0)
        throw new Error.NOT_SUPPORTED ("Unable to locate method slot; please file a bug");

    var blob = (pointer_size == 8)
#if ARM || ARM64
        ? Frida.Data.Android.get_zymbiote_arm64_bin_blob ()
        : Frida.Data.Android.get_zymbiote_arm_bin_blob ();
#else
        ? Frida.Data.Android.get_zymbiote_x86_64_bin_blob ()
        : Frida.Data.Android.get_zymbiote_x86_bin_blob ();
#endif

    unowned uint8[] payload_template = blob.data;
    void * p = memmem (payload_template, "/frida-zymbiote-00000000000000000000000000000000".data);
    assert (p != null);
    size_t data_offset = (uint8 *) p - (uint8 *) payload_template;

    var payload = new Buffer (new Bytes (payload_template), ByteOrder.HOST, pointer_size);

    size_t cursor = data_offset;
    payload.write_string (cursor, server_name);
    cursor += 64;

    payload.write_pointer (cursor, art_method_slot);
    cursor += pointer_size;

    payload.write_pointer (cursor, set_argv0_address);
    cursor += pointer_size;

    string[] wanted = {
        "socket",
        "connect",
        "__errno",
        "getpid",
        "getppid",
        "sendmsg",
        "recv",
        "close",
        "raise",
    };

    var index_of = new Gee.HashMap<string, int> ();
    for (int i = 0; i != wanted.length; i++)
        index_of[wanted[i]] = i;

    var addrs = new uint64[wanted.length];
    uint pending = wanted.length;
    libc.enumerate_exports (e => {
        if (index_of.has_key (e.name)) {
            int idx = index_of[e.name];
            addrs[idx] = libc_entry.base_address + e.address;
            pending--;
        }
        return pending != 0;
    });

    for (int i = 0; i != addrs.length; i++) {
        assert (addrs[i] != 0);
        payload.write_pointer (cursor, addrs[i]);
        cursor += pointer_size;
    }

    return new ZymbiotePrepResult () {
        process_memory = fd,
        already_patched = already_patched,

        art_method_slot = art_method_slot,
        original_ptr = original_ptr,
        replaced_ptr = replaced_ptr,

        payload = payload.bytes,
        payload_base = payload_base,
        payload_path = payload_path,
        payload_file_offset = payload_file_offset,
    };
}


```

#### 调用链

```
do_prepare_zymbiote_injection() (linux-host-session.vala:925)
          ├─ ProcMapsIter.for_pid() - 创建内存映射迭代器
          ├─ ProcMapsIter.next() - 遍历内存映射
          ├─ ProcMapsSoEntry.find_by_path() - 查找 SO 映射条目
          ├─ Gum.ElfModule.from_file() - 解析 ELF 文件
          ├─ Gum.ElfModule.enumerate_exports() - 枚举导出符号
          ├─ open_process_memory() - 打开进程内存 (linux-host-session.vala:1692)
          │   └─ Posix.open("/proc/<pid>/mem") - 打开 /proc/<pid>/mem
          ├─ FileDescriptor.pread() - 读取进程内存
          ├─ memmem() - 内存搜索 (linux-host-session.vala:1110)
          ├─ is_boot_heap() - 判断是否为 boot heap (linux-host-session.vala:1107)
          └─ Buffer.write_pointer() / write_string() - 填充 payload


```

#### 初始化变量

```
            uint64 payload_base = 0;
            string? payload_path = null;
            uint64 payload_file_offset = 0;
            string? libc_path = null;
            string? runtime_path = null;
            Gee.List<Gum.MemoryRange?> heap_candidates = new Gee.ArrayList<Gum.MemoryRange?> ();


```

*   payload_base: payload 注入地址
*   payload_path: payload 所在文件路径
*   payload_file_offset: 文件偏移（用于已补丁情况）
*   libc_path: libc.so 路径
*   runtime_path: libandroid_runtime.so 路径
*   heap_candidates: boot heap 候选区域列表

#### 解析进程内存映射

*   创建内存映射迭代器
    *   ProcMapsIter.for_pid(pid) 读取 /proc/<pid>/maps
    *   逐行解析内存映射
*   定位 payload 基址（17.6.1 改进）
    *   查找 libstagefright.so 且可执行（"x" in flags）
    *   选择映射末页：end_address - page_size
    *   记录文件路径和偏移，用于已补丁情况
*   定位 libc.so 与 libandroid_runtime.so
    *   记录第一个匹配的 libc.so 路径
    *   记录第一个匹配的 libandroid_runtime.so 路径
*   收集 boot heap 候选
    *   检查 flags == "rw-p" 且通过 is_boot_heap() 判断
    *   is_boot_heap() 检查路径是否包含：
    *   "boot.art"
    *   "boot-framework.art"
    *   "dalvik-LinearAlloc"
    *   添加到 heap_candidates

```
var iter = ProcMapsIter.for_pid (pid);
while (iter.next ()) {
    string path = iter.path;
    string flags = iter.flags;
    if (path.has_suffix ("/libstagefright.so") && "x" in flags) {
            if (payload_base == 0) {
                    payload_base = iter.end_address - Gum.query_page_size ();
                    payload_path = path;
                    payload_file_offset = iter.file_offset;
            }
    } else if (path.has_suffix ("/libc.so")) {
            if (libc_path == null)
                    libc_path = path;
    } else if (path.has_suffix ("/libandroid_runtime.so")) {
            if (runtime_path == null)
                    runtime_path = path;
    } else if (flags == "rw-p" && is_boot_heap (path)) {
            uint64 start = iter.start_address;
            uint64 end = iter.end_address;
            heap_candidates.add (Gum.MemoryRange () {
                    base_address = start,
                    size = (size_t) (end - start),
            });
    }
}


```

#### 获取 libc 与 runtime 信息

> 其实就是校验我们必须需要的信息是否存在，然后再根据我们的 libc_path 与 runtime_path 找到 libc 与 runtime 的基地址。然后对应进行符号解析得到 ELF 对象, 然后查找我们的 setArgV0Native 函数 (_Z27android_os_Process_setArgV0P7_JNIEnvP8_jobjectP8_jstring)。
> 
> 为什么需要查找我们的 setArgV0Native 函数捏? 因为这是后续查找 ArtMethod 的关键，artMethod 的函数指针就是存储的 setArgV0Native 函数地址，所以我们可以通过解析符号表定位到 setArgV0Native 函数就可以间接定位到 ArtMethod

```
if (payload_base == 0)
        throw new Error.NOT_SUPPORTED ("Unable to pick a payload base");
if (libc_path == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libc.so path");
if (runtime_path == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libandroid_runtime.so path");
if (heap_candidates.is_empty)
        throw new Error.NOT_SUPPORTED ("Unable to detect any VM heap candidates");

var libc_entry = ProcMapsSoEntry.find_by_path (pid, libc_path);
if (libc_entry == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libc.so entry");

var runtime_entry = ProcMapsSoEntry.find_by_path (pid, runtime_path);
if (runtime_entry == null)
        throw new Error.NOT_SUPPORTED ("Unable to detect libandroid_runtime.so entry");
        
Gum.ElfModule libc;
try {
        libc = new Gum.ElfModule.from_file (libc_path);
} catch (Gum.Error e) {
        throw new Error.NOT_SUPPORTED ("Unable to parse libc.so: %s", e.message);
}

Gum.ElfModule runtime;
try {
        runtime = new Gum.ElfModule.from_file (runtime_path);
} catch (Gum.Error e) {
        throw new Error.NOT_SUPPORTED ("Unable to parse libandroid_runtime.so: %s", e.message);
}

uint64 set_argv0_address = 0;
runtime.enumerate_exports (e => {
        if (e.name == "_Z27android_os_Process_setArgV0P7_JNIEnvP8_jobjectP8_jstring") {
                set_argv0_address = runtime_entry.base_address + e.address;
                return false;
        }
        return true;
});
if (set_argv0_address == 0)
        throw new Error.NOT_SUPPORTED ("Unable to locate android.os.Process.setArgV0(); please file a bug");


```

#### 定位 ArtMethod 信息

> 根据前面获取的 setArgV0Native 函数指针在 boot_heap 区域搜索就可以找到我们的 ArtMethod

```
var fd = open_process_memory (pid);

uint64 art_method_slot = 0;
bool already_patched = false;
foreach (var candidate in heap_candidates) {
        var heap = new uint8[candidate.size];
        var n = fd.pread (heap, candidate.base_address);
        if (n != heap.length)
                throw new Error.NOT_SUPPORTED ("Short read");

        void * p = memmem (heap, original_ptr);
        if (p == null) {
                p = memmem (heap, replaced_ptr);
                already_patched = p != null;
        }

        if (p != null) {
                art_method_slot = candidate.base_address + ((uint8 *) p - (uint8 *) heap);
                break;
        }
}
if (art_method_slot == 0)
        throw new Error.NOT_SUPPORTED ("Unable to locate method slot; please file a bug");


```

#### 填充 payload

> 这里搜索 frida-zymbiote-00000000000000000000000000000000，是因为在 zymbiote 中初始化 name 就是这个名字
> 
> static volatile const FridaApi frida =
> 
> {
> 
> .name = "/frida-zymbiote-00000000000000000000000000000000",
> 
> };
> 
> 只要找到这个位置，那么就根据这个位置后面填写我们对应的数据即可，完善这个 FridaApi 结构体

```
var blob = (pointer_size == 8)
#if ARM || ARM64
        ? Frida.Data.Android.get_zymbiote_arm64_bin_blob ()
        : Frida.Data.Android.get_zymbiote_arm_bin_blob ();
#else
        ? Frida.Data.Android.get_zymbiote_x86_64_bin_blob ()
        : Frida.Data.Android.get_zymbiote_x86_bin_blob ();
#endif

unowned uint8[] payload_template = blob.data;
void * p = memmem (payload_template, "/frida-zymbiote-00000000000000000000000000000000".data);
assert (p != null);
size_t data_offset = (uint8 *) p - (uint8 *) payload_template;

var payload = new Buffer (new Bytes (payload_template), ByteOrder.HOST, pointer_size);

size_t cursor = data_offset;
payload.write_string (cursor, server_name);
cursor += 64;

payload.write_pointer (cursor, art_method_slot);
cursor += pointer_size;

payload.write_pointer (cursor, set_argv0_address);
cursor += pointer_size;

string[] wanted = {
        "socket",
        "connect",
        "__errno",
        "getpid",
        "getppid",
        "sendmsg",
        "recv",
        "close",
        "raise",
};

var index_of = new Gee.HashMap<string, int> ();
for (int i = 0; i != wanted.length; i++)
        index_of[wanted[i]] = i;

var addrs = new uint64[wanted.length];
uint pending = wanted.length;
libc.enumerate_exports (e => {
        if (index_of.has_key (e.name)) {
                int idx = index_of[e.name];
                addrs[idx] = libc_entry.base_address + e.address;
                pending--;
        }
        return pending != 0;
});

for (int i = 0; i != addrs.length; i++) {
        assert (addrs[i] != 0);
        payload.write_pointer (cursor, addrs[i]);
        cursor += pointer_size;
}


```

*   我们可以知道 payload 的结构体

> 其实就是将 Payload 需要的 libc 函数地址, ArtMethod 函数地址填充进去，方便后续运作
> 
> 1.  name[64] - Unix socket 名称
> 2.  art_method_slot - ArtMethod 槽位指针地址
> 3.  original_set_argv0 - 原始函数指针
> 4.  socket - libc socket 函数
> 5.  connect - libc connect 函数
> 6.  __errno - libc errno 函数
> 7.  getpid - libc getpid 函数
> 8.  getppid - libc getppid 函数
> 9.  sendmsg - libc sendmsg 函数
> 10.  recv - libc recv 函数
> 11.  close - libc close 函数
> 12.  raise - libc raise 函数

```
struct _FridaApi
{
  char name[64];
 
  void    ** art_method_slot;
  void    (* original_set_argv0) (JNIEnv * env, jobject clazz, jstring name);
 
  int     (* socket) (int domain, int type, int protocol);
  int     (* connect) (int sockfd, const struct sockaddr * addr, socklen_t addrlen);
  int *   (* __errno) (void);
  pid_t   (* getpid) (void);
  pid_t   (* getppid) (void);
  ssize_t (* sendmsg) (int sockfd, const struct msghdr * msg, int flags);
  ssize_t (* recv) (int sockfd, void * buf, size_t len, int flags);
  int     (* close) (int fd);
  int     (* raise) (int sig);
};

```

  

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)