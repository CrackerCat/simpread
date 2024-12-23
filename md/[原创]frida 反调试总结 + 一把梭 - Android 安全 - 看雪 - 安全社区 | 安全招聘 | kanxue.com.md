> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284941.htm)

> [原创]frida 反调试总结 + 一把梭

Frida 反调试
=========

目录
--

*   [检测](#%E6%A3%80%E6%B5%8B)
    *   [普通 Frida 检测](#%E6%99%AE%E9%80%9AFrida%E6%A3%80%E6%B5%8B)
    *   [自实现 Frida 检测](#%E8%87%AA%E5%AE%9E%E7%8E%B0Frida%E6%A3%80%E6%B5%8B)
*   [绕过](#%E7%BB%95%E8%BF%87)
    *   [普通检测绕过](#%E6%99%AE%E9%80%9A%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87)
    *   [自实现检测绕过](#%E8%87%AA%E5%AE%9E%E7%8E%B0%E6%A3%80%E6%B5%8B%E7%BB%95%E8%BF%87)

检测
==

普通 Frida 检测
-----------

普通的检测总结以下几种手段：

1.  检测 / data/local/tmp 下的 frida 特征文件，frida 默认端口 27042
    
2.  双进程检测
    
3.  检测 / proc/pid/maps、/proc/pid/task/tid/stat、/proc/pid/fd 中的 frida 特征
    
4.  检测 D-BUS
    
    安卓系统使用 Binder 机制来实现进程间通信（IPC），而不是使用 D-Bus。检测函数向所有端口发送 d-bus 消息，如果返回 reject 就说明 fridaserver 开启。
    

自实现 Frida 检测
------------

我们可以看到，上面使用的绕过技巧，大多都和系统函数有关系，那么如果 app 中不调用这些系统函数，而是用自实现的函数来进行操作，不就很难 hook 了吗？

这里就有一种自实现函数的技术：`svc`。

通过安卓架构的学习，我们知道了安卓从上到下也是由层来分隔的，而层与层之间不能直接交互，而是需要一个中间层来进行操作。我们常见的`jni`就是`Java`层和`native`层的交互。而`syscall`就是`kernel`和`native`之间的中间层。

`svc`是 x86 架构中的一个指令，用于在用户模式下发起系统调用。当执行`svc`指令时，处理器会从用户态转换为内核态，执行内核级别的命令。  
![](https://secure2.wostatic.cn/static/tsxmMhApN6wr111VwTuGcB/image.png?auth_key=1734919891-w2WdhUjXtqdHFiv9RJwnZw-0-d37abeb015264c6442124dbac0e68d45)

开发者可以通过`syscall`来执行内核函数，而不是直接使用系统函数，下面介绍几种防护手段：

1.  直接使用 syscall 替代 libc 函数：  
    不使用标准 C 库函数，而是直接调用系统调用。这样可以绕过常见的 hook 点，因为大多数 hook 工具主要针对 libc 函数。
    
    ```
    #include //SYS_open SYS_read SYS_close都是syscall.h中的常量
                              //代表系统调用的编号。在Linux系统中，每个系统调用都有一个唯一的编号
    #include #include int my_open(const char *pathname, int flags) {
        return syscall(SYS_open, pathname, flags);
    }
     
    ssize_t my_read(int fd, void *buf, size_t count) {
        return syscall(SYS_read, fd, buf, count);
    }
     
    int my_close(int fd) {
        return syscall(SYS_close, fd);
    }
     
    // 使用示例
    int main() {
        int fd = my_open("/path/to/file", O_RDONLY);  //fd代表文件标识符，代表打开的是哪个文件
        if (fd != -1) {
            char buffer[100];
            ssize_t bytes_read = my_read(fd, buffer, sizeof(buffer));
            my_close(fd);
        }
        return 0;
    } 
    ```
    
2.  实现关键功能的自定义 syscall wrapper：
    
    为关键的系统调用创建自己的包装函数
    
    ```
    #include #include #include #include ssize_t secure_read(int fd, void *buf, size_t count) {
        // 完整性检查
        if (syscall(SYS_gettid) != syscall(SYS_getpid)) {
            // 可能正在被调试,终止操作
            return -1;
        }
         
        // 执行实际的读取操作
        ssize_t bytes_read = syscall(SYS_read, fd, buf, count);
         
        // 数据校验 (简单示例,实际应用中可能需要更复杂的校验)
        if (bytes_read > 0) {
            for (ssize_t i = 0; i < bytes_read; i++) {
                ((char*)buf)[i] ^= 0x55;  // 简单的XOR操作
            }
        }
         
        return bytes_read;
    } 
    ```
    
3.  动态生成 syscall：  
    在运行时动态生成 syscall 指令（直接使用机器码，和下面的汇编差不多）
    
    ```
    #include #include typedef long (*syscall_fn)(long, ...);
     
    syscall_fn generate_write_syscall() {
        // 分配可执行内存
        void* mem = mmap(NULL, 4096, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
         
        // x86-64 架构的 write syscall 机器码
        unsigned char code[] = {
            0x48, 0xc7, 0xc0, 0x01, 0x00, 0x00, 0x00,  // mov rax, 1 (write syscall number)
            0x0f, 0x05,                                // syscall
            0xc3                                       // ret
        };
         
        // 复制代码到可执行内存
        memcpy(mem, code, sizeof(code));
         
        return (syscall_fn)mem;
    }
     
    // 使用示例
    int main() {
        syscall_fn my_write = generate_write_syscall();
        const char *msg = "Hello, World!\n";
        my_write(1, msg, strlen(msg));
        return 0;
    } 
    ```
    
4.  使用汇编实现`syscall`：  
    直接使用汇编语言实现系统调用
    
    ```
    .global my_write
    my_write:
        mov x8, #64         // write syscall number for ARM64
        svc #0              // trigger syscall
        ret                 // return to caller
    
    // C代码调用示例
    // extern ssize_t my_write(int fd, const void *buf, size_t count);
    //
    // int main() {
    //     const char *msg = "Hello, World!\n";
    //     my_write(1, msg, strlen(msg));
    //     return 0;
    // }
    ```
    

最后再举一个实际检测 frida 的`syscall`例子：

```
#include #include #include #include JNIEXPORT jboolean JNICALL
Java_com_example_SecurityCheck_detectFrida(JNIEnv *env, jobject thiz) {
    char line[256];
    int fd = syscall(SYS_open, "/proc/self/maps", O_RDONLY);
    if (fd != -1) {
        while (syscall(SYS_read, fd, line, sizeof(line)) > 0) {
            if (strstr(line, "frida") || strstr(line, "gum-js-loop")) {
                syscall(SYS_close, fd);
                return JNI_TRUE;
            }
        }
        syscall(SYS_close, fd);
    }
    return JNI_FALSE;
} 
```

绕过
==

普通检测绕过
------

网上对 frida 的检测通常会使用 openat、open、strstr、pthread_create、snprintf、sprintf、readlinkat 等一系列函数，这里 hook 了 strstr 和 strcmp 函数。

```
function replace_str() {
    var pt_strstr = Module.findExportByName("libc.so", 'strstr');
    var pt_strcmp = Module.findExportByName("libc.so", 'strcmp');
  
    Interceptor.attach(pt_strstr, {
        onEnter: function (args) {
            var str1 = args[0].readCString();
  
            var str2 = args[1].readCString();
            if (str2.indexOf("tmp") !== -1 ||
                str2.indexOf("frida") !== -1 ||
                str2.indexOf("gum-js-loop") !== -1 ||
                str2.indexOf("gmain") !== -1 ||
                str2.indexOf("gdbus") !== -1 ||
                str2.indexOf("pool-frida") !== -1||
                str2.indexOf("linjector") !== -1) {
                //console.log("strcmp-->", str1, str2);
                this.hook = true;
            }
        }, onLeave: function (retval) {
            if (this.hook) {
                retval.replace(0);
            }
        }
    });
  
    Interceptor.attach(pt_strcmp, {
        onEnter: function (args) {
            var str1 = args[0].readCString();
            var str2 = args[1].readCString();
            if (str2.indexOf("tmp") !== -1 ||
                str2.indexOf("frida") !== -1 ||
                str2.indexOf("gum-js-loop") !== -1 ||
                str2.indexOf("gmain") !== -1 ||
                str2.indexOf("gdbus") !== -1 ||
                str2.indexOf("pool-frida") !== -1||
                str2.indexOf("linjector") !== -1) {
                //console.log("strcmp-->", str1, str2);
                this.hook = true;
            }
        }, onLeave: function (retval) {
            if (this.hook) {
                retval.replace(0);
            }
        }
    })
  
}
  
replace_str(); 一把梭方案
```

检测点说明：

1.  gmain：Frida 使用 Glib 库，其中的主事件循环被称为 GMainLoop。在 Frida 中，gmain 表示 GMainLoop 的线程。
2.  gdbus：GDBus 是 Glib 提供的一个用于 D-Bus 通信的库。在 Frida 中，gdbus 表示 GDBus 相关的线程。
3.  gum-js-loop：Gum 是 Frida 的运行时引擎，用于执行注入的 JavaScript 代码。gum-js-loop 表示 Gum 引擎执行 JavaScript 代码的线程。
4.  pool-frida：Frida 中的某些功能可能会使用线程池来处理任务，pool-frida 表示 Frida 中的线程池。
5.  linjector 是一种用于 Android 设备的开源工具，它允许用户在运行时向 Android 应用程序注入动态链接库（DLL）文件。通过注入 DLL 文件，用户可以修改应用程序的行为、调试应用程序、监视函数调用等，这在逆向工程、安全研究和动态分析中是非常有用的。

自实现检测绕过
-------

我们通过上面的学习看到，自实现系统函数一个重要的前提就是它们都有标准的系统调用号，标准的机器码。所以我们绕过的时候也可以用同样的思路。

Frida 的 Memory API 可以直接查找整个系统的内存内容，我们直接搜索对应函数的特征码，定位到之后再使用 Interceptor 进行 Hook。（要注意每个架构对应的特征可能不一样）

<table border="0" cellpadding="0" cellspacing="0"><tbody><tr><td>123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263</td><td><code>function hookSysOpen() {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let</code> <code>SYS_OPEN;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let</code> <code>SVC_INSTRUCTION_HEX;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>const</code> <code>arch = Process.arch;</code>&nbsp;<code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arch === </code><code>"arm64"</code><code>) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>SYS_OPEN = 56;&nbsp; </code><code>// ARM64架构下open系统调用的编号</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>SVC_INSTRUCTION_HEX = </code><code>"01 00 00 D4"</code><code>;&nbsp; </code><code>// ARM64架构下svc指令的十六进制表示</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>} </code><code>else</code> <code>if</code> <code>(arch === </code><code>"arm"</code><code>) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>SYS_OPEN = 5;&nbsp; </code><code>// ARM架构下open系统调用的编号</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>SVC_INSTRUCTION_HEX = </code><code>"00 00 00 EF"</code><code>;&nbsp; </code><code>// ARM架构下svc指令的十六进制表示</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>} </code><code>else</code> <code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"不支持的架构: "</code> <code>+ arch);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code><code>;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code>&nbsp;<code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"当前架构: "</code> <code>+ arch);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"开始搜索SYS_OPEN系统调用..."</code><code>);</code>&nbsp;<code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>//系统调用指令（如svc）通常位于可执行代码段中（r-x）</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Process.enumerateRanges(</code><code>'r-x'</code><code>).forEach(function(range) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(range.file &amp;&amp; range.file.path &amp;&amp; range.file.path.endsWith(</code><code>".so"</code><code>)) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"搜索模块: "</code> <code>+ range.file.path);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code>&nbsp;<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Memory.scan(range.</code><code>base</code><code>, range.size, SVC_INSTRUCTION_HEX, {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>onMatch: function(address) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let</code> <code>sysCallNumber;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arch === </code><code>"arm64"</code><code>) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>// 在ARM64中，系统调用号在svc指令之前的指令中</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>sysCallNumber = address.sub(4).readU32() &amp; 0xFFFF;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>} </code><code>else</code> <code>if</code> <code>(arch === </code><code>"arm"</code><code>) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>// 在ARM中，系统调用号通常在r7寄存器中，这里我们只能近似处理</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>sysCallNumber = address.sub(4).readU16() &amp; 0xFF;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code>&nbsp;<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(sysCallNumber === SYS_OPEN) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"找到SYS_OPEN调用，地址: "</code> <code>+ address);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code>&nbsp;<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Interceptor.attach(address, {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>onEnter: function(args) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let</code> <code>fileName;</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(arch === </code><code>"arm64"</code><code>) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>fileName = args[1].readUtf8String();</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>} </code><code>else</code> <code>if</code> <code>(arch === </code><code>"arm"</code><code>) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>fileName = args[0].readUtf8String();</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"SYS_OPEN被调用，文件名: "</code> <code>+ fileName);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>},</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>onLeave: function(retval) {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"SYS_OPEN返回值: "</code> <code>+ retval);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>});</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>},</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>onComplete: function() {</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"搜索完成"</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>});</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>});</code><code>}</code>&nbsp;<code>hookSysOpen();</code></td></tr></tbody></table>

[[招生] 科锐逆向工程师培训 (2024 年 11 月 15 日实地，远程教学同时开班, 第 51 期)](https://bbs.kanxue.com/thread-51839.htm)

[#HOOK 注入](forum-161-1-125.htm) [#基础理论](forum-161-1-117.htm)