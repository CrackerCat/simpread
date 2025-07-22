> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287494.htm)

> [原创] 基础 so 注入的实现

android 平台的 so 注入
=================

前言：由于安卓的进程隔离机制，我们在 hook 或操作其他进程时，往往需要先把 so 注入到目标进程

0x01 什么是 so 注入
--------------

一句话概括下来就是 `把自己的so加载到目标进程的地址空间`

0x02 主流的注入方式
------------

据我所知，目前有两种：

1.  注入 zygote 进程
2.  通过 ptrace 直接注入目标进程

先来聊聊第一点吧，我们知道安卓中所有应用进程都`fork`自`zygote`进程，所以直接把 so 注入 zygote 进程，app 在启动时会 fork zygote，从而达到注入的目。那 zygisk 是如何注入进 zygote 进程的呢，其实也是通过`ptrace`哈哈哈。所以写 zygisk 模块可以轻松帮我们注入自己的 so。同理，用 xposed 插件也可以，而且更简单。这里不细说，我们主要聊一聊第二种方式

0x03 如何使用 ptrace 注入 so
----------------------

既然要使用 ptrace，那么我们就得先知道 ptrace 是什么东西

### 0x03.1 什么是 ptrace

Ptrace 是 Linux 提供的进程调试接口，可以让一个进程去控制另一个进程。通过 ptrace，我们能附加到目标进程上，读写它的内存和寄存器。有了这些能力，远程调用目标进程的函数就成为可能了。  
主要用到的 ptrace 操作包括：

PTRACE_ATTACH：附加目标进程  
PTRACE_GETREGSET/SETREGSET：读写寄存器  
PTRACE_PEEKDATA/POKEDATA：读写内存

### 0x03.2 使用 ptrace 注入 so

知道了 ptrace 是干什么的，现在就能来注入 so 了。既然要把自己的 so 塞到目标进程里，那么直接远程调用对应进程的 dlopen 加载我们自己的 so 不就好了吗~。这就是我们的终极目标了

#### 那我们就先来实现一下远程调用：

```
bool ProcessUtils::SetupRemoteCall(RemoteCallContext* ctx, uint64_t func_addr,
                                   const std::vector& args) {
    // 复制原始寄存器
    ctx->regs = ctx->orig_regs;
 
    // 设置栈指针 - 确保16字节对齐
    ctx->regs.sp = (ctx->orig_regs.sp - 0x100) & ~0xF;
 
    // 设置参数（ARM64前8个参数通过x0-x7传递）
    for (size_t i = 0; i < args.size() && i < 8; i++) {
        ctx->regs.regs[i] = args[i];
    }
 
    // 如果参数超过8个，需要压栈
    if (args.size() > 8) {
        MemoryUtils memory_utils;
        uint64_t stack_addr = ctx->regs.sp;
        for (size_t i = 8; i < args.size(); i++) {
            if (!memory_utils.WriteProcessMemory(ctx->pid, stack_addr, &args[i], sizeof(uint64_t))) {
                LOGE("Failed to write stack argument %zu", i);
                return false;
            }
            stack_addr += sizeof(uint64_t);
        }
    }
 
    // 设置PC指向目标函数
    ctx->regs.pc = func_addr;
 
    // 设置返回地址为0，这样函数返回时会触发SIGSEGV
    ctx->regs.regs[30] = 0;  // x30是链接寄存器(LR)
 
    LOGD("Setting up remote call: PC=0x%lx, SP=0x%lx", ctx->regs.pc, ctx->regs.sp);
 
    return SetRegisters(ctx->pid, &ctx->regs);
} 
```

原理就是通过设置 pc(程序计数器) 指向目标函数的地址。比如我们要远程调用 dlopen，就得先拿到 dlopen 的地址，然后将参数写入对应寄存器，再修改 pc 指针指向 dlopen 的地址即可

#### 现在远程调用是解决了，该解决 dlopen 的地址问题了：

我们知道，内存中的函数地址是由基地址 (函数所在的 so 的起始地址)+ 偏移地址(函数相对于 so 的偏移) 确定的，而同一个 so 的函数在不同进程里和在磁盘中的偏移是一样的。所以我们有两种办法拿到函数的真实地址(准确来说是相对于 so 的偏移地址)

1.  直接解析磁盘文件，比如 dlopen 函数，我们可以解析磁盘里的 libc.so，找到 dlopen 的偏移，然后加上基地址，就是真实地址了，即 target_real_addr = target_base + offset
2.  在自己的进程中 dlopen 目标函数所在的 so，然后使用 dlsym 查找，但是这里 dlsym 查找到的是对应函数在自己进程的绝对地址，所以需要额外的计算，即 target_real_addr = dlsym_res - my_base + target_base，这种方式就无需解析 elf 了，更简单实用

获取基地址就没啥好说的了，用户层直接读 / proc/{pid}/maps 就行了

```
uint64_t ProcessUtils::GetModuleBase(pid_t pid, const std::string& module_name) {
        char maps_path[256];
        snprintf(maps_path, sizeof(maps_path), "/proc/%d/maps", pid);
 
        LOGD("GetModuleBase: pid=%d, module=%s", pid, module_name.c_str());
 
        std::ifstream maps(maps_path);
        if (!maps.is_open()) {
            LOGE("Failed to open %s: %s", maps_path, strerror(errno));
            return 0;
        }
 
        std::string line;
        bool found = false;
        while (std::getline(maps, line)) {
            if (line.find(module_name) != std::string::npos &&
                line.find(" r-xp ") != std::string::npos) {  // 只查找可执行段
                // 解析基址
                uint64_t base;
                if (sscanf(line.c_str(), "%lx", &base) == 1) {
                    maps.close();
                    LOGD("  Found module %s at base 0x%lx", module_name.c_str(), base);
                    LOGD("  Map line: %s", line.c_str());
                    return base;
                }
            }
        }
 
        maps.close();
        LOGD("  Module %s not found in process %d", module_name.c_str(), pid);
        return 0;
    }

```

```
uint64_t Injector::GetRemoteAddress(pid_t pid, const std::string& module_name,
                                        const std::string& func_name) {
        LOGD("GetRemoteAddress: module=%s, function=%s", module_name.c_str(), func_name.c_str());
 
        // 获取目标进程中的模块基址
        uint64_t remote_base = process_utils_.GetModuleBase(pid, module_name);
        if (remote_base == 0) {
            LOGD("  Module %s not found in process %d", module_name.c_str(), pid);
            return 0;
        }
 
        // 获取本地进程中的模块基址
        uint64_t local_base = process_utils_.GetModuleBase(getpid(), module_name);
        if (local_base == 0) {
            LOGD("  Module %s not found in local process", module_name.c_str());
 
            // 对于loader模块，尝试动态加载
            if (module_name.find("yuuki_transit") != std::string::npos ||
                module_name == LOADER_PATH) {
                LOGD("  Trying to load loader module locally to get function offset");
 
                void* handle = dlopen(LOADER_PATH, RTLD_NOW | RTLD_LOCAL);
                if (handle) {
                    void* func = dlsym(handle, func_name.c_str());
                    if (func) {
                        // 再次获取本地基址
                        local_base = process_utils_.GetModuleBase(getpid(), "yuuki_transit.so");
                        if (local_base == 0) {
                            local_base = process_utils_.GetModuleBase(getpid(), LOADER_PATH);
                        }
 
                        if (local_base != 0) {
                            uint64_t offset = (uint64_t)func - local_base;
                            uint64_t remote_addr = remote_base + offset;
                            dlclose(handle);
                            LOGD("  Found %s at offset 0x%lx, remote addr: 0x%lx",
                                 func_name.c_str(), offset, remote_addr);
                            return remote_addr;
                        }
                    }
                    dlclose(handle);
                }
            }
 
            return 0;
        }

```

#### 写入 so 路径：

值得注意的是，我们在使用 dlopen 的时候需要传入 so 的路径，这个值是一个字符串，更准确的来讲，dlopen 接收到的字符串的首地址。所以我们需要远程把 so 的路径写入到目标进程中。我们可以借助 ptrace 的 PTRACE_POKEDATA 实现写入，在此之前，我们得获取一块稳定已知的可写内存，所以还需要远程调用一次 mmap，远程调用函数的逻辑和上面一样，直接用就行

```
bool MemoryUtils::WriteWord(pid_t pid, uint64_t addr, long value) {
    if (ptrace(PTRACE_POKEDATA, pid, addr, value) == -1) {
        LOGE("PTRACE_POKEDATA failed at 0x%lx: %s", addr, strerror(errno));
        return false;
    }
    return true;
}

```

```
bool MemoryUtils::WriteProcessMemory(pid_t pid, uint64_t addr, const void* buf, size_t size) {
    LOGD("WriteProcessMemory: pid=%d, addr=0x%lx, size=%zu", pid, addr, size);
 
    const uint8_t* src = (const uint8_t*)buf;
    size_t remaining = size;
 
    while (remaining > 0) {
        size_t to_write = (remaining > sizeof(long)) ? sizeof(long) : remaining;
 
        long data = 0;
        if (to_write < sizeof(long)) {
            // 需要先读取原始数据，保持未修改的字节
            if (!ReadWord(pid, addr, &data)) {
                LOGE("  Failed to read original data at 0x%lx", addr);
                return false;
            }
        }
 
        memcpy(&data, src, to_write);
 
        if (!WriteWord(pid, addr, data)) {
            LOGE("  Failed to write at 0x%lx", addr);
            return false;
        }
 
        src += to_write;
        addr += to_write;
        remaining -= to_write;
    }
 
    LOGD("  Successfully wrote %zu bytes", size);
    return true;
}

```

#### selinux 模式切换：

这样主要的逻辑就全都实现了，剩下的就是处理 selinux 相关代码，通过修改`/sys/fs/selinux/enforce`文件的值实现 enforce 和 permissive 模式的切换

```
bool SELinuxUtils::SetEnforcing() {
    LOGI("Setting SELinux to enforcing mode");
    return SetEnforceStatus(1);
}
 
bool SELinuxUtils::SetPermissive() {
    LOGI("Setting SELinux to permissive mode");
    return SetEnforceStatus(0);
}
 
int SELinuxUtils::GetEnforceStatus() {
    LOGD("Checking SELinux enforce status...");
 
    std::ifstream enforce_file("/sys/fs/selinux/enforce");
    if (!enforce_file.is_open()) {
        LOGD("  /sys/fs/selinux/enforce not found, trying old path...");
        // 尝试旧路径
        enforce_file.open("/selinux/enforce");
        if (!enforce_file.is_open()) {
            LOGD("  SELinux appears to be disabled");
            return -1;
        }
    }
 
    int status;
    enforce_file >> status;
    enforce_file.close();
 
    LOGD("  SELinux enforce status: %d (%s)", status,
         status == 1 ? "enforcing" : status == 0 ? "permissive" : "unknown");
 
    return status;
}
 
bool SELinuxUtils::SetEnforceStatus(int status) {
    // 需要root权限
    int fd = open("/sys/fs/selinux/enforce", O_WRONLY);
    if (fd < 0) {
        // 尝试旧路径
        fd = open("/selinux/enforce", O_WRONLY);
        if (fd < 0) {
            LOGE("Failed to open SELinux enforce file: %s", strerror(errno));
            return false;
        }
    }
 
    char status_str[2];
    snprintf(status_str, sizeof(status_str), "%d", status);
 
    ssize_t written = write(fd, status_str, 1);
    close(fd);
 
    if (written != 1) {
        LOGE("Failed to write SELinux enforce status: %s", strerror(errno));
        return false;
    }
 
    LOGI("SELinux enforce status set to %d", status);
    return true;
}

```

0x04 小结
-------

这只是实现了最简单最基础的 attach 模式 so 注入，会留下痕迹，虽然痕迹不多，但是都比较致命，基本上有检测的 app 注入进去就会挂掉。spawn 模式下一次再介绍吧，至于隐藏痕迹，我感觉用户层也没啥好隐藏的，Memory Remapping 依然会带来新的检测点，处理 solist 和 maps 里注入 so 的路径完全可以通过自定义 linker 来加载规避掉，但这会带来较差的兼容性和比较大的工程量。最简单的就是进到内核层操作 seq_file 来处理 maps 里的信息，但是我感觉只要路径正常，so 名称随机，maps 和 solist 不处理也没啥问题，但是 dlopen 只能加载那几个路径下的 so 哈哈哈哈

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#HOOK 注入](forum-161-1-125.htm) [#源码框架](forum-161-1-127.htm)