> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.freebuf.com](https://www.freebuf.com/sectool/257766.html)

项目地址： https://github.com/chroblert/JC-AntiPtrace

环境：kali2020，ndkr17c,arm64,pixel2,android8.1

![](https://image.3001.net/images/20201214/1607928713_5fd70b899022e5c217081.png!small?1607928713467)

描述：

1.  zygote 通过 fork() 系统调用，fork 出一个 app
2.  app 内通过 ptrace(PTRACE_TRACEME,0,0,0); 将父进程 zygote 做为自己的 tracer
3.  这样其他进程就无法 ptrace() 到 app 进程了

1.  demo 版 app 放在 GitHub 中，app-debug.apk
2.  app 运行后查看 / proc/<pid>/status 中的 TracerPid
    *   ![](https://image.3001.net/images/20201214/1607928723_5fd70b93b3c004edc8575.png!small?1607928724194)
3.  使用 frida 进行调试，可以看到显示报错如下：
    *   ![](https://image.3001.net/images/20201214/1607928730_5fd70b9a4b23cf712ab1a.png!small?1607928730087)

**注：对于这种很简单的反调试，frida 可以用 - f 进行绕过**。

![](https://image.3001.net/images/20201214/1607928745_5fd70ba927570b7772082.png!small?1607928745266)

(一)、思路
------

1.  (这里主要来自看雪论坛 https://bbs.pediy.com/thread-260731.htm)
    *   使用 ptrace 附加 zygote 进程
    *   拦截 zygote 的 fork 调用，在 fork 子进程时候获取当前进程名称，判断是不是我们想要的那个应用，若是则保存子进程 pid
    *   获取到子进程 pid 后，再拦截子进程的系统调用，判断此系统调用是不是 ptrace，并且参数是 **PTRACE****_TRACEME**
    *   拦截到指定系统调用后，修改调用参数，让 **ptrace(PTRACE_TRACEME);** 执行失败

(二)、差异：
-------

由于 ptrace 对底层架构的依赖很高，同一份代码不适用于所有架构的手机，因而原作者给出的代码不适用于我的需求，我这里的是 android 8.1，pixel2，**arm64**。

1.  有些代码需要做一些变换，如下：

| 

**ARM32 (AARCH32)**

 | 

**ARM64 (AARCH64)**

 |
| 

PTRACE_GETREGS

 | 

PTRACE_GETREGSET

 |
| 

pid

 | 

pid

 |
| 

NULL

 | 

NT_PRSTATUS

 |
| 

struct pt_regs *

 | 

{struct user_pt_regs *ubf, size_t len}

 |
| 

GETREGS

 | 

NT_PRSTATUS

 |
| 

PTRACE_SETREGS

 | 

PTRACE_SETREGSET

 |

2.  有些结构体在 arm64 中的头文件中的声明不一样，也需要重新 define, 如下：   

```
#define pt_regs user_pt_regs
#define uregs regs
#define ARM_r0 regs[0]
#define ARM_r7 regs[7]
#define ARM_1r regs[30]
#define ARM_sp sp
#define ARM_pc pc
#define ARM_cpsr pstate
#define NT_PRSTATUS 1
#define NT_foo 1

```

3.  关于传参和返回值用的寄存器也有些差异，如下：

| 

**arch**

 | 

**syscall NR**

 | 

**return**

 | 

**arg0**

 | 

**arg1**

 | 

**arg2**

 | 

**arg3**

 | 

**arg4**

 | 

**arg5**

 |
| 

arm

 | 

r7

 | 

r0

 | 

r0

 | 

r1

 | 

r2

 | 

r3

 | 

r4

 | 

r5

 |
| 

arm64

 | 

x8

 | 

x0

 | 

x0

 | 

x1

 | 

x2

 | 

x3

 | 

x4

 | 

x5

 |
| 

x86

 | 

eax

 | 

eax

 | 

ebx

 | 

ecx

 | 

edx

 | 

esi

 | 

edi

 | 

ebp

 |
| 

x86_64

 | 

rax

 | 

rax

 | 

rdi

 | 

rsi

 | 

rdx

 | 

r10

 | 

r8

 | 

r9

 |

(三)、实现
------

代码在 GitHub 仓库中：[https://github.com/chroblert/JC-AntiPtrace](https://github.com/chroblert/JC-AntiPtrace)

(四)、使用
------

1.  查看 zygote64 的 pid
    *   ![](https://image.3001.net/images/20201214/1607928782_5fd70bceae7b0b1376903.png!small?1607928782465)
    *   我这里的手机是 arm64 的，所有用 zygote64 的 pid4979
2.  执行编译后的 JC-AntiPtrace-v1-arm64-1214.o 程序
    *   ./JC-AntiPtrace-v1-arm64-1214.o -p 4979 -t com.zer0ne_sec.ptrace.jnitest -n 117 -r1 -e
3.  执行 frida 附加
4.  效果如下：
    *   ![](https://image.3001.net/images/20201214/1607928803_5fd70be3c0e12a503d273.png!small?1607928803834)
    *   成功拦截并修改 ptrace() 系统调用的结果，frida 成功 hook

(五)、使用说明
--------

```
JC-AntiPtrace-v1-arm64.o [-v] -p <zygote_pid> -t <appname> [-n <syscallNo>] [-r<returnValue> [-e]]
options:
-v : verbose
-p <zygote_pid> : pid of zygote or zygote64
-t <appname> : application name of to hook
-n <syscallno> : syscalll number to hook(十进制)
117:ptrace
220:clone
260:wait
-r<returnValue> : update return value of the syscallno
-h : show helper
-e : detach when updated return value

```

### (1)、用于监控系统调用

```
JC-AntiPtrace-v1-arm64.o [-v] -p <zygote_pid> -t <appname> [-n <syscallNo>]
-v: 表示显示详细输出，包含有每个系统调用的参数值与返回值
-p: 表示zygote进程的pid
-t: 要监控的app的名称
-n：表示监控某一个特定的系统调用

```

### (2)、用于绕过 ptrace 反调试

```
JC-AntiPtrace-v1-arm64.o [-v] -p <zygote_pid> -t <appname> [-n 117] [-r0 [-e]]
-v: 表示显示详细输出，包含有每个系统调用的参数值与返回值
-p: 表示zygote进程的pid
-t: 要监控的app的名称
-n：表示监控某一个特定的系统调用
-r: 表示要修改返回值为某个值。-r后面紧跟数值
-e: 表示修改返回值后就detach

```

本篇文章及代码只是对简单 ptrace 反调试的一种绕过，仅供研究。之后会继续研究其他 ptrace 反调试的场景，欢迎关注 https://github.com/chroblert/JC-AntiPtrace 该项目。

参考资料：[使用 ptrace 过 ptrace 反调试](https://bbs.pediy.com/thread-260731.htm)

参考资料：[一种绕过 ptrace 反调试的方法](https://www.cnblogs.com/gm-201705/p/9864051.html)

参考资料：[ptrace.h](https://sites.uclouvain.be/SystInfo/usr/include/sys/ptrace.h.html)

参考资料：[在 Android 操作系统中设置永久环境变量](https://blog.csdn.net/Ls4034/article/details/77774330)

参考资料：[Shared Library Injection on Android 8.0](https://fadeevab.com/shared-library-injection-on-android-8/)

参考资料：[NT_PRSTATUS](https://www.man7.org/linux/man-pages/man2/ptrace.2.html)

参考资料：[arm64 vs arm32](https://undo.io/resources/arm64-vs-arm32-whats-different-linux-programmers/)

参考资料：[linux 沙箱之 ptrace](https://blog.betamao.me/2019/02/02/Linux%E6%B2%99%E7%AE%B1%E4%B9%8Bptrace/)

参考资料：[arm64: ptrace: reload a syscall number after ptrace operations](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1406020499-5537-2-git-send-email-takahiro.akashi@linaro.org/)

参考资料：[ptrace change syscall number arm64](https://stackoverflow.com/questions/63620203/ptrace-change-syscall-number-arm64)

参考资料：[linux system call table](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md)