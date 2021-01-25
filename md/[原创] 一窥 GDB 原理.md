> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265599.htm)

一窥 GDB 原理
=========

目录

*   [一窥 GDB 原理](#一窥gdb原理)
*   GDB 调试原理简述
*                    gdb ./a.out
*                    attach pid
*                    gdb server 的 target remote
*            [ptrace](#ptrace)
*                    [（1）PTRACE_TRACEME](#（1）ptrace_traceme)
*                    [（2）PTRACE_ATTACH](#（2）ptrace_attach)
*                    [（3）PTRACE_CONT](#（3）ptrace_cont)
*                    [（4）PTRACE_PEEKUSER](#（4）ptrace_peekuser)
*                    [（5）PTRACE_SINGLESTEP](#（5）ptrace_singlestep)
*                    [demo](#demo)
*            [breakpoints（断点）](#breakpoints（断点）)
*            [watchpoint（硬件断点）](#watchpoint（硬件断点）)
*   [用 ptrace 实现一个 tracer](#用ptrace实现一个tracer)
*                    [ELF 文件解析](#elf文件解析)
*                    [断点注入](#断点注入)
*                    [断点追踪](#断点追踪)
*   [参考](#参考)

本文想要达到的目的：

1.  **核心目标**：了解 GDB 的调试原理，学习 ptrace 的使用。
2.  **支线目标**：实现一个可以用来追踪的 tiny debugger（迷你调试器）。

名词：

*   tracer：追踪者
*   tracee：被追踪者

**作为一个 PWN 手，平常总是跟 GDB 打交道，这篇文章简单来介绍一下 GDB 相关的基本原理~（在会用工具的同时也要了解工具的基本原理）**

GDB 调试原理简述
==========

GDB 整体简要的结构体如下：

 

![](https://s3.ax1x.com/2021/01/22/solChd.png)

 

当我们用 gdb 调试一个可执行文件时都发生了什么？

### gdb ./a.out

以这种方式直接运行时，首先，gdb 解析 a.out 文件的符号。接下来我们输入 `run` 命令，gdb 通过 `fork()` 一个新进程，然后通过 `ptrace(PTRACE_TRACEME, 0, NULL, NULL);` 设置 traceme 模式。最后执行 `exec` 启动加载要调试的文件。

### attach pid

在调试 PWN 题时，通过 attach pid 来追踪要调试的进程。gdb 通过执行 `ptrace(PTRACE_ATTACH，pid, 0, 0)` 来对目标进程进行追踪。

### gdb server 的 target remote

在 gdb+qemu 调试内核时，经常用到 target remote 来 attach 到 qemu 上对 vmlinux 进行调试。二者之间有特殊的定义好的数据信息通信的格式，进行通信。

ptrace
------

ptrace 可以说是 gdb 的灵魂了。

 

https://man7.org/linux/man-pages/man2/ptrace.2.html

 

ptrace 原型如下：

```
long ptrace(enum __ptrace_request request, pid_t pid,void *addr, void *data);

```

官方对其进行了 DESCRIPTION 如下：

```
The ptrace() system call provides a means by which one process
      (the "tracer") may observe and control the execution of another
      process (the "tracee"), and examine and change the tracee's
      memory and registers.  It is primarily used to implement
      breakpoint debugging and system call tracing.

```

翻译一下：ptrace() 系统调用提供了一种方法可以使得追踪者（tracer）来对被追踪者（tracee）进行观察与控制。具体表现为可以检查 tracee 中内存以及寄存器的值。**ptrace 首要地被用于实现断点 debug 与系统调用追踪**。

```
A tracee first needs to be attached to the tracer.  Attachment
       and subsequent commands are per thread: in a multithreaded
       process, every thread can be individually attached to a
       (potentially different) tracer, or left not attached and thus not
       debugged.  Therefore, "tracee" always means "(one) thread", never
       "a (possibly multithreaded) process".  Ptrace commands are always
       sent to a specific tracee using a call of the form

```

首先，tracee process 必须要被 tracer attach 上（也就是我们启动 gdb 后的 attach pid），需要注意的是，attach 和后续的命令是针对每个线程来说的。如果是一个多线程的程序，每个线程都要被单独的 attach 才行。这里主要强调了，tracee（被追踪者）是一个单独的 **thread**，而非一个整个的多线程程序。

```
While being traced, the tracee will stop each time a signal is
      delivered, even if the signal is being ignored.  (An exception is
      SIGKILL, which has its usual effect.)  The tracer will be
      notified at its next call to waitpid(2) (or one of the related
      "wait" system calls); that call will return a status value
      containing information that indicates the cause of the stop in
      the tracee.  While the tracee is stopped, the tracer can use
      various ptrace requests to inspect and modify the tracee.  The
      tracer then causes the tracee to continue, optionally ignoring
      the delivered signal (or even delivering a different signal
      instead).

```

当追踪时，tracee 每次发送一个信号就会停一次，即使这个 signal 被会忽略掉。而 tracer 将会捕捉到 tracee 的下一个调用（通过 waitpid 或 wait 类似系统调用）。而这个调用将会告诉 tracer，tracee 停止的原因以及相关信息。所以当 tracee 停下来，tracer 可以通过 ptrace 的多种模式来进行监控甚至修改 tracee，然后 tracer 会告诉 tracee 继续运行。

 

ptrace 四个参数的含义解释如下：

*   第一参数 request ：request 的值确定要执行的操作

ptrace 的第一个参数可以是如下的值：

```
PTRACE_TRACEME,   本进程被其父进程所跟踪。其父进程应该希望跟踪子进程
PTRACE_PEEKTEXT,  从内存地址中读取一个字节，内存地址由addr给出
PTRACE_PEEKDATA,  同上
PTRACE_PEEKUSER,  可以检查用户态内存区域(USER area),从USER区域中读取一个字节，偏移量为addr
PTRACE_POKETEXT,  往内存地址中写入一个字节。内存地址由addr给出
PTRACE_POKEDATA,  往内存地址中写入一个字节。内存地址由addr给出
PTRACE_POKEUSER,  往USER区域中写入一个字节，偏移量为addr
PTRACE_GETREGS,    读取寄存器
PTRACE_GETFPREGS,  读取浮点寄存器
PTRACE_SETREGS,  设置寄存器
PTRACE_SETFPREGS,  设置浮点寄存器
PTRACE_CONT,    重新运行
PTRACE_SYSCALL,  重新运行
PTRACE_SINGLESTEP,  设置单步执行标志
PTRACE_ATTACH，追踪指定pid的进程
PTRACE_DETACH，  结束追踪

```

*   第二参数 pid ：指示 ptrace 要跟踪的进程。
*   第三参数 addr ：指定 ptrace 要读取 or 监控的内存地址。
*   第四参数 data ：如果我们要向目标进程写入数据，那么 data 是我们要写入的数据；如果我们从目标进程中读出数据，那么读出的数据放在 data。

接下来我们来看 request 中富有代表性的几个模式。

### [](#（1）ptrace_traceme)（1）PTRACE_TRACEME

这个模式 **只被 tracee 使用**，使用它的进程将会被其父进程追踪。父进程通过 wait() 获知子进程的信号。

```
/**
 * ptrace_traceme  --  helper for PTRACE_TRACEME
 *
 * Performs checks and sets PT_PTRACED.
 * Should be used by all ptrace implementations for PTRACE_TRACEME.
 */
static int ptrace_traceme(void)
{
    int ret = -EPERM;
 
    write_lock_irq(&tasklist_lock);
    /* Are we already being traced? */
    if (!current->ptrace) {
        ret = security_ptrace_traceme(current->parent);
        /*
         * Check PF_EXITING to ensure ->real_parent has not passed
         * exit_ptrace(). Otherwise we don't report the error but
         * pretend ->real_parent untraces us right after return.
         */
        if (!ret && !(current->real_parent->flags & PF_EXITING)) {
            current->ptrace = PT_PTRACED;
            __ptrace_link(current, current->real_parent);
        }
    }
    write_unlock_irq(&tasklist_lock);
 
    return ret;
}

```

当我们只用 traceme 模式时，内核首先会让写者拿到读写锁，并禁止本地中断。

 

接下来判断是否我们当前进程已经被追踪，接着将子进程链接到父进程的 ptrace 链表中。

 

最后放掉锁。

### [](#（2）ptrace_attach)（2）PTRACE_ATTACH

在 attach 模式下，通过指定一个 tracee 的 pid，tracee 向 tracer 发送 **SIGSTOP** 信号。而 tracer 使用 waitpid() 等待 tracee 停止。

```
if (request == PTRACE_ATTACH) {
       if (child == current)              
           goto out;
       if ((!child->dumpable ||                //这里检查了进程权限
           (current->uid != child->euid) ||
           (current->uid != child->suid) ||
           (current->uid != child->uid) ||
           (current->gid != child->egid) ||
           (current->gid != child->sgid) ||
           (!cap_issubset(child->cap_permitted, current->cap_permitted)) ||
           (current->gid != child->gid)) && !capable(CAP_SYS_PTRACE))
           goto out;                  
       if (child->flags & PF_PTRACED)
           goto out;
       child->flags |= PF_PTRACED;           //设置进程标志位PF_PTRACED
 
       write_lock_irqsave(&tasklist_lock, flags);
       if (child->p_pptr != current) {     //设置进程为当前进程的子进程。
           REMOVE_LINKS(child);
           child->p_pptr = current;
           SET_LINKS(child);
       }
       write_unlock_irqrestore(&tasklist_lock, flags);
       send_sig(SIGSTOP, child, 1);      //向子进程发送一个SIGSTOP，使其停止
       ret = 0;
       goto out;
   }

```

### [](#（3）ptrace_cont)（3）PTRACE_CONT

tracer 通过这个模式，向 tracee 发信号，让停止的 tracee 继续运行。

```
case PTRACE_CONT:
            long tmp;
            ret = -EIO;
            if ((unsigned long) data > _NSIG)       //信号是否超过范围？
                goto out;
            if (request == PTRACE_SYSCALL)
                child->flags |= PF_TRACESYS;        //如果是PTRACE_SYSCALL就设置PF_TRACESYS标志
            else
                child->flags &= ~PF_TRACESYS;         //如果是PF_CONT，去除PF_TRACESYS标志
            child->exit_code = data;                //设置继续处理的信号 
            tmp = get_stack_long(child, EFL_OFFSET) & ~TRAP_FLAG; //清除TRAP_FLAG
            put_stack_long(child, EFL_OFFSET,tmp); 
            wake_up_process(child);                 //唤醒停止的子进程
            ret = 0;
            goto out;

```

### [](#（4）ptrace_peekuser)（4）PTRACE_PEEKUSER

在 tracee 的 USER 区域的 addr 处读取一个 word。读取的这个字为返回值。

### [](#（5）ptrace_singlestep)（5）PTRACE_SINGLESTEP

而具体到 ptrace 中的 PTRACE_SINGLESTEP 来说，这是基于 eflags 寄存器的 TF 位（陷阱标志）实现的。他强迫子进程，执行下一条汇编指令，然后又停止他，此时子进程产生一个 debug exception 而对应的异常处理程序负责清掉这个标志位，并强迫当前进程停止。然后发送 SIGCHLD 信号给父进程。

### demo

有了以上基础后我们来看如下 demo

```
#include #include #include #include #include /* For constants  ORIG_RAX etc */
#include int main()
{  
 
    char * argv[ ]={"ls","-al","/etc/passwd",(char *)0};
    char * envp[ ]={"PATH=/bin",0};
    pid_t child;
    long orig_rax;
    child = fork();
    if(child == 0)
    {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        printf("Try to call: execl\n");
        execve("/bin/ls",argv,envp);
        printf("child exit\n");
    }
    else
    {
        wait(NULL);             //等待子进程
        orig_rax = ptrace(PTRACE_PEEKUSER,
                          child, 8 * ORIG_RAX,
                          NULL);
        printf("The child made a "
               "system call %ld\n", orig_rax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
        printf("Try to call:ptrace\n");
    }
    return 0;
} 
```

这段程序输出如下：

```
root@ubuntu:~/tiny_debugger# ./demo1
Try to call: execl
The child made a system call 59
Try to call:ptrace
root@ubuntu:~/tiny_debugger# -rw-r--r-- 1 root root 2446 Dec 13 19:53 /etc/passwd

```

我们来梳理一下他的过程。

1.  fork 一个子进程。子进程标记为 tracee。（PTRACE_TRACEME）
2.  子进程调用 execve，向父进程发送一个 SIGCHLD。同时，子进程由于 SIGTRAP 停止。
3.  父进程捕捉到 SIGCHLD，同时使用 ptrace 获取子进程的系统调用号（59）
4.  父进程告诉子进程继续执行（PTRACE_CONT），子进程输出 ls 的内容。
5.  子进程的 `printf("child exit\n");` **不会被执行**，因为 execve 丢弃原来的子进程`execve()`之后的部分，而子进程的栈、数据会被新进程的相应部分所替换。永不返回。

需要注意的是，让子进程停住的是子进程中的 **exec 族** 函数。

```
/**
 * ptrace_event - possibly stop for a ptrace event notification
 * @event:    %PTRACE_EVENT_* value to report
 * @message:    value for %PTRACE_GETEVENTMSG to return
 *
 * Check whether @event is enabled and, if so, report @event and @message
 * to the ptrace parent.
 *
 * Called without locks.
 */
static inline void ptrace_event(int event, unsigned long message)
{
    if (unlikely(ptrace_event_enabled(current, event))) {
        current->ptrace_message = message;
        ptrace_notify((event << 8) | SIGTRAP);
    } else if (event == PTRACE_EVENT_EXEC) {
        /* legacy EXEC report via SIGTRAP */
        if ((current->ptrace & (PT_PTRACED|PT_SEIZED)) == PT_PTRACED)
            send_sig(SIGTRAP, current, 0);
    }
}

```

当 tracee 触发一个 exec 的时候，会通过 `send_sig(SIGTRAP, current, 0)` 产生一个 **SIGTRAP** ，**导致停止**。

 

![](https://images2015.cnblogs.com/blog/676200/201602/676200-20160220161141175-432928373.png)

 

![](https://s3.ax1x.com/2021/01/23/sohyE6.png)

 

接下来总结一下 ptrace 是如何起作用的：

*   通过 `copy_from_user` `copy_to_user` 读取与修改数据。
*   通过 `copy_regset_from/to_user` 访问寄存器。而寄存器数据保存在 task struct 中。
*   单步（Single Stepping）：每步进 (step) 一次，CPU 会一直执行到有分支、中断或异常。而 ptrace 通过设置对应的标志位在进程的 thread_info.flags 和 MSR 中打标启用单步调试。

```
void user_enable_single_step(struct task_struct *child)
{
    enable_step(child, 0);
}

```

```
/*
 * Enable single or block step.
 */
static void enable_step(struct task_struct *child, bool block)
{
    /*
     * Make sure block stepping (BTF) is not enabled unless it should be.
     * Note that we don't try to worry about any is_setting_trap_flag()
     * instructions after the first when using block stepping.
     * So no one should try to use debugger block stepping in a program
     * that uses user-mode single stepping itself.
     */
 
    if (enable_single_step(child) && block)
        set_task_blockstep(child, true);
    else if (test_tsk_thread_flag(child, TIF_BLOCKSTEP))
        set_task_blockstep(child, false);
}

```

在 `enable_single_step` 中设置了 `X86_EFLAGS_TF` 以及 `TIF_SINGLESTEP` 标志位。

 

在 `test_tsk_thread_flag` 中检查了对应进程 thread_info 中的 `TIF_BLOCKSTEP` 标志位。

```
void set_task_blockstep(struct task_struct *task, bool on)
{
    unsigned long debugctl;
    local_irq_disable();
    debugctl = get_debugctlmsr();
    if (on) {
        debugctl |= DEBUGCTLMSR_BTF;
        set_tsk_thread_flag(task, TIF_BLOCKSTEP);
    } else {
        debugctl &= ~DEBUGCTLMSR_BTF;
        clear_tsk_thread_flag(task, TIF_BLOCKSTEP);
    }
    if (task == current)
        update_debugctlmsr(debugctl);
    local_irq_enable();
}

```

接下来在 set_task_blockstep 中设置或清除了 `DEBUGCTLMSR_BTF` 以及 对应 thread info 的 `TIF_BLOCKSTEP` 。

 

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zdGF0aWMwMDEuZ2Vla2Jhbmcub3JnL3Jlc291cmNlL2ltYWdlLzMxLzJkLzMxZDE1YmNkMmEwNTMyMzViNTU5MDk3N2QxMmZmYTJkLmpwZWc?x-oss-process=image/format,png)

[](#breakpoints（断点）)breakpoints（断点）
-----------------------------------

首先明确一点，**breakpoints 并不是 ptrace 的实现的一部分**。并且，当处于 attach 和 traceme 状态下，**交付给 tracee 的任何信号首先都会被 GDB 截获**。

 

breakpoints 大体的实现如下：

 

假设我们想在 addr 处停下来。

 

那么 GDB 会做如下事情。

 

1. 读取 addr 处的指令的位置，存入 GDB 维护的断点链表中。

 

2. **将中断指令 INT 3 （0xCC）打入原本的 addr 处。也就是将 addr 处的指令掉换成 INT 3**

 

3. 当执行到 addr 处（INT 3）时，CPU 执行这条指令的过程也就是发生断点异常（breakpoint exception），tracee 产生一个 SIGTRAP，此时我们处于 attach 模式下，tracee 的 SIGTRAP 会被 tracer（GDB）捕捉。

 

然后 GDB 去他维护的断点链表中查找对应的位置，如果找到了，说明 hit 到了 breakpoint。

 

4. 接下来，如果我们想要 tracee 继续正常运行，GDB 将 INT 3 指令换回原来正常的指令，回退重新运行正常指令，然后接着运行。

 

我们可以看如下 demo：

```
#include #include #include int main(){
    printf("test INT 3\n");
    __asm__("int $0x3");
    printf("breakpoint");
    return 0;
} 
```

当我们独立运行时：

```
root@ubuntu:~/tiny_debugger# ./bp
test INT 3
Trace/breakpoint trap (core dumped)

```

当放到 GDB 中时：

 

![](https://s3.ax1x.com/2021/01/23/sTdAVs.png)

 

可以看到已经由于 INT 3 的存在停下来了。如果我们 c 下去，就可以正常运行结束。

[](#watchpoint（硬件断点）)watchpoint（硬件断点）
-------------------------------------

在 GDB 中另一个非常有用的是 watch 命令。用于监控某一内存位置或者寄存器的变化。

 

watch 的实现与 CPU 的相关寄存器有关。我们以 80386 为例。

 

存在 DR0 到 DR7 这八个特殊的寄存器来实现**硬件断点**。

 

![](https://s3.ax1x.com/2021/01/23/s7vktg.png)

 

**（1）DR0-DR3**：每个寄存器都保存着对应条件断点的线性地址。而每个断点更进一步的信息储存在 DR7 中。需要注意的是，由于储存的是线性地址，所以是否开启分页是不影响的。如果开启 paging，那么线性地址由 mmu 转换到物理地址；否则线性地址与物理地址等效。

 

**（2）DR7 调试控制寄存器 debug control**：DR7 的低八位（0、2、4、6 和 1、3、5、7）有选择地启用四个条件断点。启用级别有两个：本地（0,2,4,6）和全局（1,3,5,7）。处理器会在每个任务切换时自动重置本地启用位，以避免在新任务中出现不必要的断点情况。全局启用位不会由任务开关重置；因此，它们可以用于所有任务的全局条件。

 

16-17 位（对应于 DR0），20-21（DR1），24-25（DR2），28-29（DR3）定义了断点触发的时间。每个断点都有一个两位对应，用于指定它们是在执行（00b），数据写入（01b），数据读取还是写入（11b）时中断。10b 被定义为表示 IO 读取或写入中断，但没有硬件支持它。位 18-19（DR0），22-23（DR1），26-27（DR2），30-31（DR3）定义了断点监视多大的内存区域。同样，每个断点都有一个两位对应，指定他们 watch 一个（00b），两个（01b），八（10b）还是四（11b）个字节。

 

（3）**DR6 调试状态寄存器**：告诉调试器哪些断点已经发生了。

用 ptrace 实现一个 tracer
====================

这部分主要是参考：  
[On the subject of debuggers and tracers](http://researchcomplete.blogspot.com/2016/08/on-subject-of-debuggers-and-tracers_5.html?m=1)

 

完整代码加完整注释已放到 Github：  
[OrangeGzY/tiny_debugger](https://github.com/OrangeGzY/tiny_debugger)

 

我们希望能写一个迷你的 **tracer** 来实现打断点和追踪。

 

最简单的，我们需要两个进程，子进程负责启动 **tracee** 程序，主进程负责追踪 tracee。

 

**tracee** 程序如下：

```
#include int func1()
{
    printf("function1");
}
 
void func2()
{
    printf("function2");
}
 
void func3()
{
    printf("fucntion3");
}
 
int main()
{
    //printf("===========\n");
    func1();
    func3();
    func2();
    func2();
    func3();
    //printf("===========\n");
    return 0;
} 
```

tracee 程序由 fork 出的子进程 execl 启动。

```
if(child == 0){
    ptrace(PTRACE_TRACEME, 0, NULL, NULL);
    execl(argv[1],argv[1],NULL);
    perror("fail exec");
    exit(-1);
}

```

接下来我们来看 **tracer** 的实现。

### ELF 文件解析

首先，我们需要解析 tracee 对应的一些 ELF 信息。以获取对应的 **函数符号、函数名、函数地址** 的信息。

```
fread(&header, sizeof(Elf64_Ehdr), 1, fp);    //read elf header of target file
fseek(fp, header.e_shoff, SEEK_SET);        //move the pointer to Section Header Table

```

我们首先读取 ELF header，然后将文件内部指针 **移动到段表的位置**。

 

接下来我们扫描每一个 **section header** ，直到找到我们 **符号表** 的信息。

```
for(int i=0;i < header.e_shnum;i++)
{
    fread(§ion_header, sizeof(Elf64_Shdr), 1, fp);
    if(section_header.sh_type == SHT_SYMTAB)
    {
        ......
    }
    ......
}

```

找到符号表后，我们定位到 **字符串表 header(strtab header)** 在段表中的位置，并读取相关信息。

```
fseek(fp,str_table_offset,SEEK_SET);                    //定位到字符串表header
fread(&str_section_header, sizeof(Elf64_Shdr), 1, fp);    //读取字符串表表头

```

现在符号表的字符串表的基本信息我们都有了。可以接着进行下一步。

 

我们扫描符号表中的每一项 `Elf64_Sym`

 

如果这个符号是一个函数并且地址存在且合法，那么这就是我们要追踪的函数之一。

 

**我们通过在每个函数开始的位置打断点实现追踪。**

 

扫描到了需要追踪的函数，我们将其对应的信息（**函数地址、函数名、地址处对应的指令**）放入定义好的 Breakpoint 结构体中。

```
int bp = 0;        //global
typedef struct breakpoint{
    size_t addr;
    char name[25];
    size_t orig_code;
}Breakpoint;
Breakpoint bp_list[N];

```

```
for(int i=0;i
```

当扫描结束时，所有的需要追踪（打断点）的函数都被加入我们的 **bp_list** 中。

### 断点注入

我们通过遍历 `Breakpoint bp_list[N]` 来对每一个需要追踪的函数实现断点注入。

```
int breakpoint_injection(pid_t child){
    /* 我们向每个函数的第一条指令的位置注入INT 3 即 0xCC  */
    for(int i=0 ; i
```

步骤如下：

 

（1）读出要打断点处的原本的指令并保存在 `bp_list[i].orig_code` 中。

 

（2）**通过 ptrace 向 addr 处注入 0xCC （INT 3）**。相当于修改了此处的指令，动态的进行的 patch。

 

（3）检查是否注入成功。

 

至此，我们的断点就打完了，INT 3 已经注入到每一个函数起始位置。

### 断点追踪

在 tracer 中我们做完了断点注入之后。

 

首先等待子进程 execl 执行。

 

接下来判断 wait 等到的信号的类型。如果是一个 SIGSTOP 的话进一步判断是否是 SIGTRAP。

 

如果是 SIGTRAP 再去判断是否踩到了断点。此时我们做如下步骤。

 

（1）保存用户态寄存器，为回退做准备。

 

（2）扫描断点列表，看当前的 rip 寄存器中的值减 1 是否与断点列表中某一项的地址相同，如果是，说明断点命中（hit），若命中，输出函数名称。

 

（3）接下来，为了程序的正常执行，我们用 ptrace 将之前注入 INT 3 的地址处的指令恢复成 `orgi_code` 。

 

（4）之后，通过 ptrace 设置寄存器，让执行流回退一步，执行应该执行的正常指令（int 3 此时被改回来了）。

 

（5）单步步进一下，越过这条正常指令，再次重新对这个地址注入断点。

 

至此，实现了断点的追踪与维持，并维护了程序的正常执行流程和断点信息。

```
if(WIFSTOPPED(status))
            {
                /* 如果是STOP信号 */
                if(WSTOPSIG(status)==SIGTRAP)
                {                //如果是触发了SIGTRAP,说明碰到了断点
                    ptrace(PTRACE_GETREGS,child,0,®s);    //读取此时用户态寄存器的值，准备为回退做准备
                    //printf("[+] SIGTRAP rip:0x%llx\n",regs.rip);
                    /* 将此时的addr与我们bp_list中维护的addr做对比，如果查找到，说明断点命中 */
                    if((hit_index=if_bp_hit(regs))==-1)
                    {
                        /*未命中*/
                        printf("MISS, fail to hit:0x%llx\n",regs.rip);
                        exit(-1);
                    }
                    else
                    {
                        /*如果命中*/
                        /*输出命中信息*/
                        printf("%s()\n",bp_list[hit_index].name);
                        /*把INT 3 patch 回本来正常的指令*/
                        ptrace(PTRACE_POKETEXT,child,bp_list[hit_index].addr,bp_list[hit_index].orig_code);
                        /*执行流回退，重新执行正确的指令*/
                        regs.rip = bp_list[hit_index].addr;
                        ptrace(PTRACE_SETREGS,child,0,®s);
 
                        /*单步执行一次，然后恢复断点*/
                        ptrace(PTRACE_SINGLESTEP,child,0,0);
                        wait(NULL);
                        /*恢复断点*/
                        ptrace(PTRACE_POKETEXT, child, bp_list[hit_index].addr, (bp_list[hit_index].orig_code & 0xFFFFFFFFFFFFFF00) | INT_3);
                    }
                }   
            }
            ptrace(PTRACE_CONT,child,0,0);

```

最终输出：

 

![](https://s3.ax1x.com/2021/01/24/sbq6Gd.png)

 

如果我们开启 debug 模式，输出如下：

 

![](https://s3.ax1x.com/2021/01/24/sbqWsP.png)

 

可以看到，整个流程非常清晰。

参考
==

https://man7.org/linux/man-pages/man2/ptrace.2.html

 

https://sourceware.org/gdb/wiki/Internals

 

https://www.jianshu.com/p/b1f9d6911c90

 

https://blog.csdn.net/u012417380/article/details/60468697

 

https://blog.csdn.net/reliveIT/article/details/108269437

 

[Ptrace--Linux 中一种代码注入技术的应用](https://blog.csdn.net/litost000/article/details/82813641?utm_medium=distribute.pc_relevant_download.none-task-blog-BlogCommendFromBaidu-1.nonecase&depth_1-utm_source=distribute.pc_relevant_download.none-task-blog-BlogCommendFromBaidu-1.nonecas)

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

最后于 12 小时前 被 ScUpax0s 编辑 ，原因：