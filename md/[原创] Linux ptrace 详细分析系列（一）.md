> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265812.htm)

Linux ptrace 详解
===============

（该系列将深入分析 Linux ptrace 的方方面面，争取一次把它搞定，对后续调试或开发都有益处。不定期更新。）

 

目录

*   Linux ptrace 详解
*            [一、简述](#一、简述)
*            [二、函数原型及初步使用](#二、函数原型及初步使用)
*                    1. 函数原型
*                    2. 函数定义
*                    2. 初步使用
*                            1. 最简单的 ls 跟踪
*                            2. 系统调用查看参数
*                            3. 系统调用参数 - 改进版
*            [参考文献](#参考文献)

 

**备注：文章中使用的 Linux 内核源码版本为 Linux 5.9，使用的 Linux 版本为 Linux ubuntu 5.4.0-65-generic**

[](#一、简述)一、简述
-------------

ptrace 系统调用提供了一个进程 (`tracer`) 可以控制另一个进程 (`tracee`) 运行的方法，并且`tracer`可以监控和修改`tracee`的内存和寄存器，主要用作实现断点调试和系统调用追踪。

 

`tracee`首先要被 attach 到`tracer`上，这里的 attach 以线程为对象，在多线程场景（这里的多线程场景指的使用`clone CLONE_THREAD` flag 创建的线程组）下，每个线程可以分别被 attach 到`tracer`上。ptrace 的命令总是以下面的调用格式发送到指定的`tracee`上：

```
ptrace(PTRACE_foom, pid, ...)   // pid为linux中对应的线程ID

```

一个进程可以通过调用`fork()`函数来初始化一个跟踪，并让生成的子进程执行`PTRACE_TRACEME`，然后执行`execve`(一般情况下) 来启动跟踪。进程也可以使用`PTRACE_ATTACH`或`PTRACE_SEIZE`进行跟踪。

 

当处于被跟踪状态时，`tracee`每收到一个信号就会 stop，即使是某些时候信号是被忽略的。`tracer`将在下一次调用`waitpid`或与`wait`相关的系统调用之一）时收到通知。该调用会返回一个状态值，包含`tracee`停止的原因。`tracee`发生 stop 时，`tracer`可以使用各种 ptrace 的`request`来检查和修改`tracee`。然后，`tracer`使`tracee`继续运行，选择性地忽略所传递的信号（甚至传递一个与原来不同的信号）。

 

当`tracer`结束跟踪后，发送`PTRACE_DETACH`信号释放`tracee`，`tracee`可以在常规状态下继续运行。

[](#二、函数原型及初步使用)二、函数原型及初步使用
---------------------------

### 1. 函数原型

ptrace 的原型如下：

```
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);

```

其中`request`参数表明执行的行为（后续将重点介绍）， `pid`参数标识目标进程，`addr`参数表明执行`peek`和`poke`操作的地址，`data`参数则对于`poke`操作，指明存放数据的地址，对于`peek`操作，指明获取数据的地址。

 

返回值，成功执行时，`PTRACE_PEEK`请求返回所请求的数据，其他情况时返回 0，失败则返回 - 1。`errno`被设置为

### 2. 函数定义

ptrace 的内核实现在`kernel/ptrace.c`文件中，内核接口是`SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr, unsigned long, data)`。其代码如下，整体逻辑简单，需要注意的是对`PTRACE_TRACEME`和`PTRACE_ATTACH`进行了特殊处理（对于该函数的参数后续将进行深入解析）。

```
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,unsigned long, data)
{
    struct task_struct *child;
    long ret;
 
    if (request == PTRACE_TRACEME) {
        ret = ptrace_traceme();
        if (!ret)
            arch_ptrace_attach(current);
        goto out;
    }
 
    child = find_get_task_by_vpid(pid);
    if (!child) {
        ret = -ESRCH;
        goto out;
    }
 
    if (request == PTRACE_ATTACH || request == PTRACE_SEIZE) {
        ret = ptrace_attach(child, request, addr, data);
        /*
        * Some architectures need to do book-keeping after
        * a ptrace attach.
        */
        if (!ret)
            arch_ptrace_attach(child);
        goto out_put_task_struct;
    }
 
    ret = ptrace_check_attach(child, request == PTRACE_KILL ||
                request == PTRACE_INTERRUPT);
    if (ret < 0)
        goto out_put_task_struct;
 
    ret = arch_ptrace(child, request, addr, data);
    if (ret || request != PTRACE_DETACH)
        ptrace_unfreeze_traced(child);
 
out_put_task_struct:
    put_task_struct(child);
out:
    return ret;
}

```

系统调用都改为了`SYSCALL_DEFINE`的方式。如何获得上面的定义的呢？这里需要穿插一下`SYSCALL_DEFINE`的定义 (syscall.h):

```
#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

```

宏定义进行展开：

```
#define SYSCALL_DEFINEx(x, sname, ...)                \
    SYSCALL_METADATA(sname, x, __VA_ARGS__)            \
    __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
 
/*
 * The asmlinkage stub is aliased to a function named __se_sys_*() which
 * sign-extends 32-bit ints to longs whenever needed. The actual work is
 * done within __do_sys_*().
 */
#ifndef __SYSCALL_DEFINEx
#define __SYSCALL_DEFINEx(x, name, ...)                    \
    __diag_push();                            \
    __diag_ignore(GCC, 8, "-Wattribute-alias",            \
              "Type aliasing is used to sanitize syscall arguments");\
    asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))    \
        __attribute__((alias(__stringify(__se_sys##name))));    \
    ALLOW_ERROR_INJECTION(sys##name, ERRNO);            \
    static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
    asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));    \
    asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))    \
    {                                \
        long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
        __MAP(x,__SC_TEST,__VA_ARGS__);                \
        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));    \
        return ret;                        \
    }                                \
    __diag_pop();                            \
    static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
#endif /* __SYSCALL_DEFINEx */

```

`__SYSCALL_DEFINEx`中的`x`表示系统调用的参数个数，且`sys_ptrace`的宏定义如下：

```
/* kernel/ptrace.c */
asmlinkage long sys_ptrace(long request, long pid, unsigned long addr,
               unsigned long data);

```

所以对应的`__SYSCALL_DEFINEx`应该是`SYSCALL_DEFINE4`，这与上面的定义`SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr, unsigned long, data)`一致。

 

仔细观察上面的代码可以发现，函数定义其实在最后一行，结尾没有分号，然后再加上花括号即形成完整的函数定义。前面的几句代码并不是函数的实现（详细的分析可以跟踪源码，出于篇幅原因此处不放出每个宏定义的跟踪）。

 

定义的转换过程：

```
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr, unsigned long, data)
--> SYSCALL_DEFINEx(4, _ptrace, __VA_ARGS__) 
    -->  __SYSCALL_DEFINEx(4, __ptrace, __VA_ARGS__)
      #define __SYSCALL_DEFINEx(x, name, ...) \
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__)) \
      --> asmlinkage long sys_ptrace(__MAP(4,__SC_DECL,__VA_ARGS__))

```

而对`__MAP`宏和`__SC_DECL`宏的定义如下：

```
/*
 * __MAP - apply a macro to syscall arguments
 * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
 *    m(t1, a1), m(t2, a2), ..., m(tn, an)
 * The first argument must be equal to the amount of type/name
 * pairs given.  Note that this list of pairs (i.e. the arguments
 * of __MAP starting at the third one) is in the same format as
 * for SYSCALL_DEFINE/COMPAT_SYSCALL_DEFINE */
#define __MAP0(m,...)
#define __MAP1(m,t,a,...) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)
 
#define __SC_DECL(t, a)    t a 
```

按照如上定义继续进行展开

```
__MAP(4,__SC_DECL, long request, long pid, unsigned long addr,
               unsigned long data)
-->  __MAP4(__SC_DECL, long, request, long, pid, unsigned long, addr,
               unsigned long, data)
-->  __SC_DECL(long, request), __MAP3(__SC_DECL, __VA_ARGS__)
  __MAP3(__SC_DECL, long, pid, unsigned long, addr, unsigned long, data)
  --> __SC_DECL(long, pid), __MAP2(__SC_DECL, unsigned long, addr, unsigned long, data)       
          -->__SC_DECL(unsigned long, addr), __MAP1(__SC_DECL, __VA_ARGS__)
              unsigned long addr, __SC_DECL(unsigned long, data)
              --> unsigned long data
  long pid, __SC_DECL(unsigned long, addr), __MAP1(__SC_DECL, __VA_ARGS__)
  --> long pid, unsigned long addr, unsigned long data
-->  long request, __SC_DECL(long, pid), __MAP2(__SC_DECL, __VA_ARGS__)
-->  long request, long pid, unsigned long addr, unsigned long data

```

最后调用`asmlinkage long sys_ptrace(long request, long pid, unsigned long addr, unsigned long data);`。

 

为什么要将系统调用定义成宏？主要是因为 2 个内核漏洞 CVE-2009-0029，CVE-2010-3301，Linux 2.6.28 及以前版本的内核中，将系统调用中 32 位参数传入 64 位的寄存器时无法作符号扩展，可能导致系统崩溃或提权漏洞。

 

内核开发者通过将系统调用的所有输入参数都先转化成 long 类型（64 位），再强制转化到相应的类型来规避这个漏洞。

```
asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__)) \
{ \
        long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
        __MAP(x,__SC_TEST,__VA_ARGS__); \
        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__)); \
        return ret; \
} \
 
define __TYPE_AS(t, v) __same_type((__force t)0, v) /*判断t和v是否是同一个类型*/
 
define __TYPE_IS_L(t) (__TYPE_AS(t, 0L)) /*判断t是否是long 类型,是返回1*/
 
define __TYPE_IS_UL(t) (__TYPE_AS(t, 0UL)) /*判断t是否是unsigned long 类型,是返回1*/
 
define __TYPE_IS_LL(t) (__TYPE_AS(t, 0LL) || __TYPE_AS(t, 0ULL)) /*是long类型就返回1*/
 
define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a /*将参数转换成long类型*/
 
define __SC_CAST(t, a) (__force t) a /*转成成原来的类型*/
 
define __force __attribute__((force)) /*表示所定义的变量类型可以做强制类型转换*/

```

### 2. 初步使用

#### 1. 最简单的 ls 跟踪

首先通过一个简单的例子来熟悉一下`ptrace`的使用：

```
#include #include #include #include #include #include int main(int argc, char *argv[]){
 
    pid_t child;
    long orig_rax;
 
    child = fork();
 
    if(child == 0){
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);  // Tell kernel, trace me
            execl("/bin/ls", "ls", NULL);
    }else{  
        /*Receive certification after child process stopped*/
        wait(NULL);
 
        /*Read child process's rax*/
        orig_rax = ptrace(PTRACE_PEEKUSER, child, 8*ORIG_RAX, NULL);
        printf("[+] The child made a system call %ld.\n", orig_rax);
 
        /*Continue*/
        ptrace(PTRACE_CONT, child, NULL, NULL);
        }
 
    return 0;
} 
```

运行结果如下：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeiptrace_demo.png)

 

打印出系统调用号，并等待用户输入。查看`/usr/include/x86_64-linux-gnu/asm/unistd_64.h`文件（64 位系统）查看 59 对应的系统调用：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeisystem_call_execve.png)

 

59 号恰好为`execve`函数调用。对上面的过程进行简单总结：

1.  父进程通过调用`fork()`来创建子进程，在子进程中，执行`execl()`之前，先运行`ptrace()`，`request`参数设置为`PTRACE_TRACEME`来告诉 kernel 当前进程正在被 trace。当有信号量传递到该进程，进程会 stop，提醒父进程在`wait()`调用处继续执行。然后调用`execl()`，执行成功后，新程序运行前，`SIGTRAP`信号量被发送到该进程，子进程停止，父进程在`wait()`调用处收到通知，获取子进程的控制权，查看子进程内存和寄存器相关信息。
    
2.  当发生系统调用时，kernel 保存了`rax`寄存器的原始内容，其中存放的是系统调用号，我们可以使用`request`参数为`PTRACE_PEEKUSER`的`ptrace`来从子进程的`USER`段读取出该值。
    
3.  系统调用检查结束后，子进程通过调用`request`参数为`PTRACE_CONT`的`ptrace`函数继续执行。
    

#### 2. 系统调用查看参数

```
#include #include #include #include #include #include #include #include int main(int argc, char *argv[]){
    pid_t child;
    long orig_rax, rax;
    long params[3];
    int status;
    int insyscall = 0;
 
    child = fork();
    if(child == 0){
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }else{
        while(1){
            wait(&status);
            if(WIFEXITED(status))
                break;
            orig_rax = ptrace(PTRACE_PEEKUSER, child, 8 * ORIG_RAX, NULL);
            if(orig_rax == SYS_write){
                if(insyscall == 0){
                    insyscall = 1;
                    params[0] = ptrace(PTRACE_PEEKUSER, child, 8 * RBX, NULL);
                    params[1] = ptrace(PTRACE_PEEKUSER, child, 8 * RCX, NULL);
                    params[2] = ptrace(PTRACE_PEEKUSER, child, 8 * RDX, NULL);
                    printf("Write called with %ld, %ld, %ld\n", params[0], params[1], params[2]);
                }else{
                    rax = ptrace(PTRACE_PEEKUSER, child, 8 * RAX, NULL);
                    printf("Write returned with %ld\n", rax);
                    insyscall = 0;
                }
            }
            ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        }
    }
    return 0;
} 
```

执行结果：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeisystem_call_ls.png)

 

在上面的程序中，跟踪的是`wirte`的系统调用，`ls`命令总计进行了三次`write`的调用。`request`参数为`PTEACE_SYSCALL`时的`ptrace`使 kernel 在进行系统调用进入或退出时 stop 子进程，这等价于执行`PTRACE_CONT`并在下一次系统调用进入或退出时 stop。

 

`wait`系统调用中的`status`变量用于检查子进程是否已退出，这是用来检查子进程是否被 ptrace 停掉或是否退出的典型方法。而宏`WIFEXITED`则表示了子进程是否正常结束（例如通过调用`exit`或者从`main`返回等），正常结束时返回`true`。

#### 3. 系统调用参数 - 改进版

前面有介绍`PTRACE_GETREGS`参数，使用它来获取寄存器的值相比前面一种方法要简单很多：

```
#include #include #include #include #include #include #include #include int main(int argc, char *argv[]){
 
    pid_t child;
    long orig_rax, rax;
    long params[3];
    int status;
    int insyscall = 0;
    struct user_regs_struct regs;
 
    child = fork();
    if(child == 0){
        ptrace(PTRACE_TRACEME, child, 8 * ORIG_RAX, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else{
        while(1){
            wait(&status);
            if(WIFEXITED(status))
                break;
            orig_rax = ptrace(PTRACE_PEEKUSER, child, 8*ORIG_RAX, NULL);
            if(orig_rax == SYS_write){
                if(insyscall == 0){
                    insyscall == 1;
                    ptrace(PTRACE_GETREGS, child, NULL, ®s);
                    printf("Write called with %lld,  %lld,  %lld\n", regs.rbx, regs.rcx, regs.rdx);
                }else{
                    rax = ptrace(PTRACE_PEEKUSER, child, 8*rax, NULL);
                    printf("Write returned with %ld\n", rax);
                    insyscall = 0;
                }
            }
            ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        }
    }
    return 0;
} 
```

执行结果：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeisystem_call_getregs.png)

 

整体输出与前面的代码无所差别，但在代码开发上使用了`PTRACE_GETREGS`来获取子进程的寄存器的值，简洁了很多。

参考文献
----

[1]. https://www.linuxjournal.com/article/6100  
[2]. https://blog.csdn.net/u012417380/article/details/60468697  
[3]. Linux ptrace man page

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)

最后于 1 小时前 被有毒编辑 ，原因：