> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286233.htm)

> [原创]Frida 检测思路

1、frida-core
============

Frida 注入目标进程后会使用 Interceptor.attach 对退出进程的方法进行 inline hook。此处可以定位三个关键函数：exit、_exit、abort  
GitHub 源码链接：  
https://github.com/frida/frida-core/blob/1018aca28d49ace21cbb5d54d2328c0283d327fe/lib/payload/exit-monitor.vala#L34

```
#if WINDOWS
            interceptor.attach ((void *) Gum.Process.find_module_by_name ("kernel32.dll").find_export_by_name ("ExitProcess"),
                listener);
#else
            var libc = Gum.Process.get_libc_module ();
            const string[] apis = {
                "exit",
                "_exit",
                "abort",
            };
            foreach (var symbol in apis) {
                interceptor.attach ((void *) libc.find_export_by_name (symbol), listener);
            }
#endif
```

2、frida-gum
===========

frida 调用了 gum_interceptor_replace 函数对 libc 库的 signal 和 sigaction 函数进行了 inline hook。  
GitHub 链接：https://github.com/frida/frida-gum/blob/0671c27d941d77490fff4ca6dbb9ca10a0f872bb/gum/backend-posix/gumexceptor-posix.c#L228

```
static void
gum_exceptor_backend_attach (GumExceptorBackend * self)
{
  GumInterceptor * interceptor = self->interceptor;
  const gint handled_signals[] = {
    SIGABRT,
    SIGSEGV,
    SIGBUS,
    SIGILL,
    SIGFPE,
    SIGTRAP,
    SIGSYS,
  };
  gint highest, i;
  struct sigaction action;
 
  highest = 0;
  for (i = 0; i != G_N_ELEMENTS (handled_signals); i++)
    highest = MAX (handled_signals[i], highest);
  g_assert (highest > 0);
  self->num_old_handlers = highest + 1;
  self->old_handlers = g_new0 (struct sigaction *, self->num_old_handlers);
 
  action.sa_sigaction = gum_exceptor_backend_on_signal;
  sigemptyset (&action.sa_mask);
  action.sa_flags = SA_SIGINFO | SA_NODEFER;
#ifdef SA_ONSTACK
  action.sa_flags |= SA_ONSTACK;
#endif
  for (i = 0; i != G_N_ELEMENTS (handled_signals); i++)
  {
    gint sig = handled_signals[i];
    struct sigaction * old_handler;
 
    old_handler = g_slice_new0 (struct sigaction);
    self->old_handlers[sig] = old_handler;
    gum_original_sigaction (sig, &action, old_handler);
  }
 
  gum_interceptor_begin_transaction (interceptor);
 
  gum_interceptor_replace (interceptor, gum_original_signal,
      gum_exceptor_backend_replacement_signal, self, NULL);
  gum_interceptor_replace (interceptor, gum_original_sigaction,
      gum_exceptor_backend_replacement_sigaction, self, NULL);
 
  gum_interceptor_end_transaction (interceptor);
}
```

3、检测思路
======

前提：已知 frida 注入目标进程会 hook libc 库以下目标函数：  
{"sigaction", "signal", "exit", "abort", "_exit"}

a. 获取目标函数在 libc 库的地址，并 copy 这些函数的入口机器码。
---------------------------------------

b. 获取目标函数的偏移。
-------------

c. 获取这些函数入口的原始机器码，读取磁盘文件 / system/lib64/libc.so（此处为 arm64-v8 架构），通过获取的偏移地址定位目标函数在 libc.so 文件的位置；copy 这些函数的入口机器码。
----------------------------------------------------------------------------------------------------------------

d. 比较从磁盘文件读取的机器码和从内存中读取的机器码，不同则可以确定 frida 已经注入。
-----------------------------------------------

实现代码如下：

```
#include #include #include #include #include #include #include #include #include #include #include #define LOG_TAG "HookDetection"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
#define BYTE_BUFFER_SIZE 16
 
// 获取函数地址
void* get_function_address(void* handle, const char* func_name) {
    void* func_addr = dlsym(handle, func_name);
    if (!func_addr) {
        LOGD("[-] Function %s not found in global symbol table", func_name);
    } else {
        LOGD("[+] Function: %s, addr: 0x%lx", func_name, (uintptr_t)func_addr);
    }
    return func_addr;
}
 
// 获取函数偏移
uintptr_t get_function_offset(void* func_addr) {
    Dl_info info;
    if (dladdr(func_addr, &info) == 0) {
        LOGD("[-] Unable to get function info");
        return 0;
    }
    return (uintptr_t)func_addr - (uintptr_t)info.dli_fbase;
}
 
// 读取库文件中的字节
bool read_bytes_from_libso(const char* libpath, uintptr_t offset, uint8_t* buffer, size_t size) {
    int fd = open(libpath, O_RDONLY);
    if (fd == -1) {
        LOGD("[-] Failed to open %s: %s", libpath, strerror(errno));
        return false;
    }
 
    if (lseek(fd, offset, SEEK_SET) == -1) {
        LOGD("[-] Seek failed at %lx in %s: %s", (unsigned long)offset, libpath, strerror(errno));
        close(fd);
        return false;
    }
 
    ssize_t bytes_read = read(fd, buffer, size);
    close(fd);
 
    if (bytes_read != (ssize_t)size) {
        LOGD("[-] Read %zd bytes, expected %zu from %s at offset %lx", bytes_read, size, libpath, (unsigned long)offset);
        return false;
    }
 
    return true;
}
 
// 字节数组转十六进制字符串
void bytes_to_hex_string(const uint8_t* bytes, size_t size, char* hex_string) {
    for (size_t i = 0; i < size; i++) {
        sprintf(hex_string + i * 2, "%02x", bytes[i]);
    }
    hex_string[size * 2] = '\0';
}
 
// 比较内存中的字节和库文件中的字节
bool compare_function_bytes(void* func_addr, uint8_t* file_bytes, size_t size) {
    uint8_t mem_bytes[BYTE_BUFFER_SIZE];
    memcpy(mem_bytes, func_addr, size);
 
    char mem_hex_string[BYTE_BUFFER_SIZE * 2 + 1];
    char file_hex_string[BYTE_BUFFER_SIZE * 2 + 1];
 
    bytes_to_hex_string(mem_bytes, size, mem_hex_string);
    bytes_to_hex_string(file_bytes, size, file_hex_string);
 
    LOGD("[*] Memory: %s | File: %s", mem_hex_string, file_hex_string);
 
    return memcmp(mem_bytes, file_bytes, size) == 0;
}
 
// Hook 检测函数
bool detect_hook(void* handle, const char* lib_path, const char* func_name) {
    void* func_addr = get_function_address(handle, func_name);
    if (!func_addr) return false;
 
    uintptr_t offset = get_function_offset(func_addr);
    if (offset == 0) {
        LOGD("[-] Failed to get offset for %s", func_name);
        return false;
    }
 
    uint8_t file_bytes[BYTE_BUFFER_SIZE];
    if (!read_bytes_from_libso(lib_path, offset, file_bytes, sizeof(file_bytes))) {
        LOGD("[-] Failed to read bytes for %s", func_name);
        return false;
    }
 
    bool is_hooked = !compare_function_bytes(func_addr, file_bytes, sizeof(file_bytes));
    LOGD("[+] %s in %s is %s", func_name, lib_path, is_hooked ? "HOOKED" : "NOT HOOKED");
    return is_hooked;
}
 
// 执行 Hook 检测
int do_hook_check() {
    const char* libpath = "/system/lib64/libc.so";
    void* handle = dlopen(libpath, RTLD_LAZY);
    if (!handle) {
        LOGD("[-] Failed to load %s", libpath);
        return -1;
    }
 
    const char* funcs[] = {"sigaction", "signal", "exit", "abort", "_exit"};
    bool hooked = false;
 
    for (size_t i = 0; i < sizeof(funcs) / sizeof(funcs[0]); i++) {
        if (detect_hook(handle, libpath, funcs[i])) {
            hooked = true;
        }
    }
 
    dlclose(handle);
    return hooked;
} 
```

4、总结
====

分析源码中 frida 注入逻辑进行检测，这种方法类似于开卷考试，不过优点明显：针对性很强，检测逻辑复杂度很低，性能开销小。  
不知道后续 frida 会不会修改相关逻辑。

[[招生] 科锐逆向工程师培训 (2025 年 3 月 11 日实地，远程教学同时开班, 第 52 期)！](https://bbs.kanxue.com/thread-51839.htm)

[#HOOK 注入](forum-161-1-125.htm)