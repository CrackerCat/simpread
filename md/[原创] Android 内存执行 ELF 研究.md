> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-275261.htm)

> [原创] Android 内存执行 ELF 研究

Android 内存执行 ELF 研究
===================

初遇
==

在 Linux 系统中，我们可以通过 memfd_create 和 execve 实现内存运行 ELF，但是在 Android 中，却遇到了这样的问题

```
CANNOT LINK EXECUTABLE "/data/local/tmp/paylaod": library "libicu.so" not found: needed by /system/lib64/libharfbuzz_ng.so in namespace (default)

```

通过 [这篇文章](https://github.com/5ec1cff/my-notes/blob/master/linker-log.md) 了解到可以通过 LD_DEBUG 环境变量开启 linker 日志，执行

 

`LD_DEBUG=1 /data/local/tmp/memexec`

 

并通过 logcat 抓到了如下日志

```
W linker  : [ Android dynamic linker (64-bit) ]
W linker  : [ Linking executable "/data/local/tmp/memexec" ]
W linker  : [ Linking "[vdso]" ]
W linker  : [ Reading linker config "/linkerconfig/ld.config.txt" ]
W linker  : [ Using config section "**unrestricted**" ]
W linker  : [ Linking "/data/local/tmp/memexec" ]
W linker  : [ Linking "/system/lib64/liblog.so" ]
W linker  : [ Linking "/apex/com.android.runtime/lib64/bionic/libm.so" ]
W linker  : [ Linking "/apex/com.android.runtime/lib64/bionic/libdl.so" ]
W linker  : [ Linking "/apex/com.android.runtime/lib64/bionic/libc.so" ]
W linker  : [ Linking "/system/lib64/libc++.so" ]
W linker  : [ Linking "/system/lib64/libnetd_client.so" ]
W linker  : [ Linking "/system/lib64/libcutils.so" ]
W linker  : [ Linking "/system/lib64/libbase.so" ]
W linker  : [ CFI add 0x61abb23000 + 0x67000  ]
W linker  : [ CFI add 0x77780d3000 + 0x1000 linux-vdso.so.1 ]
W linker  : [ CFI add 0x7776c48000 + 0x10000 liblog.so ]
W linker  : [ CFI add 0x7773d87000 + 0x37000 libm.so ]
W linker  : [ CFI add 0x7773dc0000 + 0x5000 libdl.so ]
W linker  : [ CFI add 0x7773a1e000 + 0x318000 libc.so ]
W linker  : [ CFI add 0x7776cd2000 + 0xae000 libc++.so ]
W linker  : [ CFI add 0x7773945000 + 0x2b000 libnetd_client.so: 0x777394a000 ]
W linker  : [ CFI add 0x777391b000 + 0x12000 libcutils.so ]
W linker  : [ CFI add 0x77738c4000 + 0x3b000 libbase.so ]
W linker  : [ CFI add 0x777391b000 + 0x12000 libcutils.so ]
W linker  : [ CFI add 0x77738c4000 + 0x3b000 libbase.so ]
W linker  : [ Jumping to _start (0x61abb418e0)... ]
W linker  : [ Android dynamic linker (64-bit) ]
W linker  : [ Linking executable "/memfd:jit-cache (deleted)" ]
W linker  : [ Linking "[vdso]" ]
W linker  : [ Reading linker config "/linkerconfig/ld.config.txt" ]
W linker  : [ Linking "/memfd:/jit-cache (deleted)" ]
F linker  : CANNOT LINK EXECUTABLE "/data/local/tmp/payload": library "libicu.so" not found: needed by /system/lib64/libharfbuzz_ng.so in namespace (**default**)

```

看起来是命名空间的问题，我们的 payload 的命名空间为 default，但是为什么会这样呢，明明 memexec 是 unrestricted。

相知
==

通过 Google 搜索 **android linker namespace 看到官方文档** **[链接器命名空间](https://source.android.com/docs/core/architecture/vndk/linker-namespace?hl=zh-cn#how-does-it-work)** ，在 **/linkerconfig/ld.config.txt** 文件中配置了相关信息

```
dir.system = /system/bin/
......
dir.unrestricted = /data/nativetest64/unrestricted
dir.isolated = /data/local/tmp/isolated
dir.product = /data/local/tests/product
dir.system = /data/local/tests/system
dir.unrestricted = /data/local/tests/unrestricted
dir.vendor = /data/local/tests/vendor
dir.unrestricted = /data/local/tmp
......
[system]
additional.namespaces = com_android_adbd,com_android_appsearch,com_android_art,com_android_conscrypt,com_android_i18n,com_android_media,com_android_neuralnetworks,com_android_os_statsd,com_android_resolv,com_android_runtime,com_android_tethering,rs,sphal,vndk,vndk_product
namespace.default.isolated = true
namespace.default.visible = true
namespace.default.search.paths = /system/${LIB}
......

```

首先我们要知道这个文件可不可以写，查看 mount 信息发现该文件所在的不是只读文件系统，所以可以直接修改

```
# cat /proc/mounts | grep linkerconfig
tmpfs /linkerconfig tmpfs rw,seclabel,nosuid,nodev,noexec,relatime,size=5846680k,nr_inodes=1461670,mode=755 0 0
tmpfs /linkerconfig tmpfs rw,seclabel,nosuid,nodev,noexec,relatime,size=5846680k,nr_inodes=1461670,mode=755 0 0

```

> 以下使用 `/data/adb/magisk/busybox vi /linkerconfig/ld.config.txt` 编辑文件

接
-

原来命名空间是根据路径来的，于是编辑该文件，在最前面手动加上一行

```
dir.unrestricted = /memfd:

```

然后重新执行，问题依旧。

化
-

难道是没有路径分割符的问题，于是在代码中给文件名加上一个 **`/`**

```
memfd_create(“/jit-cache”, MFD_CLOEXEC);

```

编译并重新执行，还是一样的报错。

发
-

尝试重新编辑，把它设置成根目录

```
dir.unrestricted = /

```

这次报错多了一行

```
WARNING: linker: /linkerconfig/ld.config.txt:1: warning: property value is empty (ignoring this line)

```

看样子是没办法通过单纯的改配置文件完成了，于是前往阅读 linker 的源码

相识
==

> 由于 aospxref 服务最近不太稳定，我在看的时候刚好打不开，使用了 android 官方的[源码在线浏览工具](https://cs.android.com/)，然而这个玩意并不好用

 

众所周知，linker 的源码在 bionic/linker 目录下。

 

通过字符串 “Using config section” 定位到 linker_config.cpp 的 parse_config_file 函数，在阅读代码后，我们上面的尝试为什么会失败，就已经有了答案。

为什么设置 `/memfd:` 目录不行
--------------------

line242

```
if (access(value.c_str(), R_OK) != 0) {
    if (errno == ENOENT) {
        // no need to test for non-existing path. skip.
        continue;
    }
    // If not accessible, don't call realpath as it will just cause
    // SELinux denial spam. Use the path unresolved.
    resolved_path = value;
}

```

因为它不是一个真实存在的目录，在代码中通过 access 函数判断改路径不存在，所以不行

为什么设置根目录不行
----------

line227

```
// remove trailing '/'
while (!value.empty() && value.back() == '/') {
    value.pop_back();
}
 
if (value.empty()) {
    DL_WARN("%s:%zd: warning: property value is empty (ignoring this line)",
            ld_config_file_path,
            cp.lineno());
    continue;
}

```

因为代码中删除了末尾所有的`/` 符号，传入根目录之后字符串变成了空字符串，于是抛出了那个 Warning

相杀
==

修改无非两个思路：

1.  静态替换 linker
2.  动态 hook

虽然有 magisk，替换 system 分区下文件变得方便了起来，但是替换 linker 依然是一件麻烦事，动态 Hook 对于解决这个问题来说，我觉得更加方便。由于目标代码位置的特殊性，现有 inline hook, plt hook 轮子都不能取得比 linker 更早的执行时机。

> 也许是我孤陋寡闻，如果有请务必告诉我

 

最终我选择的方案是先 fork 生成一个子进程，对子进程进行 ptrace，修改逻辑后再 detach 掉。

 

对于逻辑的修改，主要有两步：

1.  给 **/linkerconfig/ld.config.txt** 添加 `"dir.unrestricted = /memfd:/\n”`
2.  让 access 这个目录返回 0

第一步
---

直接编辑该文件添加上述行即可

第二步
---

```
bool mem_load(const std::string& image, char** argv, char** envp){
    // 创建内存文件，设置这个参数会在exec后自动关闭这个文件
    int fd = memfd_create("/jit-cache", MFD_CLOEXEC);
    ftruncate(fd, (long)image.size()); // 设置文件长度
    void *mem = mmap(nullptr, image.size(), PROT_WRITE, MAP_SHARED, fd, 0);
    memcpy(mem, image.data(), image.size());
    munmap(mem, image.size());
    // 此时已经将ELF内容写入到内存文件里面
    int pid = fork();
    if (pid < 0) {
        printf("mem_load failed\n"); // fork 失败
        return false;
    } else if (pid == 0) {
        // 这里是子进程，使用 PTRACE_TRACEME 主动建立连接
        ptrace(PTRACE_TRACEME);
        fexecve(fd, argv, envp);
    }
    // 这里是父进程
    int status;
    struct user_regs_struct regs{};
    struct iovec iov{};
    iov.iov_base = ®s;
    iov.iov_len = sizeof(user_regs_struct);
    while(true){
        wait(&status); // 等待子进程暂停
        if(WIFEXITED(status)){
            break; // 子进程退出
        }
        // 读取通用寄存器，系统调用号
        ptrace(PTRACE_GETREGSET, pid, NT_PRSTATUS, &iov);
        if (regs.regs[8] == __NR_faccessat){ // access函数使用的系统调用号
            char path[] = "/memfd:";
            long word;
            // 注意，PTRACE_PEEKDATA 固定一次读取 sizeof(long) 字节长度的内容
            word = ptrace(PTRACE_PEEKDATA, pid, regs.regs[1], NULL);
            if (strcmp(path, (char*)&word) == 0){ // 判断是否是我们添加的目录
                ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL); // 单步执行，让系统调用执行完
                wait(nullptr); // 等待系统调用执行完
                ptrace(PTRACE_GETREGSET, pid, NT_PRSTATUS, &iov); // 读取寄存器
                regs.regs[0] = 0; // 修改返回值为寄存器
                ptrace(PTRACE_SETREGSET, pid, NT_PRSTATUS, &iov); // 修改寄存器
                ptrace(PTRACE_DETACH, pid, NULL, NULL); // detach 进程
                return true;
            }
        }
        ptrace(PTRACE_SYSCALL, pid, NULL, NULL); // 执行到下一个系统调用时暂停
    }
    return false;
}

```

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

最后于 8 小时前 被 Ylarod 编辑 ，原因： 代码缩进调整

[#系统相关](forum-161-1-126.htm) [#工具脚本](forum-161-1-128.htm)