> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265924.htm)

承接上文 [Linux ptrace 详细分析系列（一）](https://bbs.pediy.com/thread-265812.htm)，本次主要介绍 2 个不同版本下的源码分析和 2 个关键参数解析。

 

目录

*            [三、sys_ptrace 函数源码分析](#三、sys_ptrace函数源码分析)
*                    1. Linux-2.6 版本的源码分析
*                            1. 源码分析
*                            2. 流程梳理
*                            3. 其他
*                    2. Linux-5.9 版本的源码分析
*                            1. 源码分析
*                            2. 流程梳理
*                            3. 其他
*            [四、Request 参数详解](#四、request参数详解)
*                    1. 参数简述
*                    2. 重要参数详解
*                            1. PTRACE_TRACEME
*                            2. PTRA CE_ATTACH

[](#三、sys_ptrace函数源码分析)三、sys_ptrace 函数源码分析
------------------------------------------

### 1. Linux-2.6 版本的源码分析

#### 1. 源码分析

首先看一下 linux-2.6.0 的`sys_ptrace`的处理流程（以`/arch/i386/kernel/ptrace.c`为例）：

```
/*
 * Note that this implementation of ptrace behaves differently from vanilla
 * ptrace.  Contrary to what the man page says, in the PTRACE_PEEKTEXT,
 * PTRACE_PEEKDATA, and PTRACE_PEEKUSER requests the data variable is not
 * ignored.  Instead, the data variable is expected to point at a location
 * (in user space) where the result of the ptrace call is written (instead of
 * being returned).
 */
asmlinkage int sys_ptrace(long request, long pid, long addr, long data)
{
    struct task_struct *child;
    struct user * dummy = NULL;
    int i, ret;
 
    lock_kernel();
    ret = -EPERM;
    if (request == PTRACE_TRACEME) {  // 请求为PTRACE_TRACEME
        /* 检查是否做好被跟踪的准备 */
        if (current->ptrace & PT_PTRACED)
            goto out;
        ret = security_ptrace(current->parent, current);
        if (ret)
            goto out;
    /* 检查通过，在process flags中设置ptrace位*/
        current->ptrace |= PT_PTRACED;
        ret = 0;
        goto out;
    }
 
  /* 非PTRACE_TRACEME的请求*/
    ret = -ESRCH;   // 首先设置返回值为ESRCH，表明没有该进程，宏定义在errno-base.h头文件中
    read_lock(&tasklist_lock);
    child = find_task_by_pid(pid);        // 查找task结构
    if (child)
        get_task_struct(child);
    read_unlock(&tasklist_lock);
    if (!child)     // 没有找到task结构，指明所给pid错误
        goto out;
 
    ret = -EPERM;   // 返回操作未授权
    if (pid == 1)        // init进程不允许被调试
        goto out_tsk;
 
  /* 请求为 PTRACE_ATTACH 时*/
    if (request == PTRACE_ATTACH) {
        ret = ptrace_attach(child); // 进行attach
        goto out_tsk;
    }
 
  /* 检查进程是否被跟踪，没有的话不能执行其他功能；
   * 当不是PTRACE_KILL时，要求进程状态为TASK_STOPPED；
   * 被跟踪进程必须为当前进程的子进程
   * 在之前是直接在该代码处实现以上逻辑，现在重新将以上功能封装成了ptrace_check_attach函数
   */
    ret = ptrace_check_attach(child, request == PTRACE_KILL); 
    if (ret < 0)
        goto out_tsk;
 
  /* 以下就为根据不同的request参数进行对应的处理了，用一个switch来总括，流程比较简单。*/
    switch (request) {
    /* when I and D space are separate, these will need to be fixed. 这算预告吗？23333*/
    case PTRACE_PEEKTEXT: /* read word at location addr. */
    case PTRACE_PEEKDATA: {
      unsigned long tmp;
      int copied;
 
      copied = access_process_vm(child, addr, &tmp, sizeof(tmp), 0);
      ret = -EIO;  // 返回I/O错误
      if (copied != sizeof(tmp))
        break;
      ret = put_user(tmp,(unsigned long *) data);
      break;
    }
 
    /* read the word at location addr in the USER area. */
    case PTRACE_PEEKUSR: {
      unsigned long tmp;
 
      ret = -EIO;
      if ((addr & 3) || addr < 0 ||
          addr > sizeof(struct user) - 3)
        break;
 
      tmp = 0;  /* Default return condition */
      if(addr < FRAME_SIZE*sizeof(long))
        tmp = getreg(child, addr);
      if(addr >= (long) &dummy->u_debugreg[0] &&
         addr <= (long) &dummy->u_debugreg[7]){
        addr -= (long) &dummy->u_debugreg[0];
        addr = addr >> 2;
        tmp = child->thread.debugreg[addr];
      }
      ret = put_user(tmp,(unsigned long *) data);
      break;
    }
 
    /* when I and D space are separate, this will have to be fixed. */
    case PTRACE_POKETEXT: /* write the word at location addr. */
    case PTRACE_POKEDATA:
      ret = 0;
      if (access_process_vm(child, addr, &data, sizeof(data), 1) == sizeof(data))
        break;
      ret = -EIO;
      break;
 
    case PTRACE_POKEUSR: /* write the word at location addr in the USER area */
      ret = -EIO;
      if ((addr & 3) || addr < 0 ||
          addr > sizeof(struct user) - 3)
        break;
 
      if (addr < FRAME_SIZE*sizeof(long)) {
        ret = putreg(child, addr, data);
        break;
      }
      /* We need to be very careful here.  We implicitly
         want to modify a portion of the task_struct, and we
         have to be selective about what portions we allow someone
         to modify. */
 
        ret = -EIO;
        if(addr >= (long) &dummy->u_debugreg[0] &&
           addr <= (long) &dummy->u_debugreg[7]){
 
          if(addr == (long) &dummy->u_debugreg[4]) break;
          if(addr == (long) &dummy->u_debugreg[5]) break;
          if(addr < (long) &dummy->u_debugreg[4] &&
             ((unsigned long) data) >= TASK_SIZE-3) break;
 
          if(addr == (long) &dummy->u_debugreg[7]) {
            data &= ~DR_CONTROL_RESERVED;
            for(i=0; i<4; i++)
              if ((0x5f54 >> ((data >> (16 + 4*i)) & 0xf)) & 1)
                goto out_tsk;
          }
 
          addr -= (long) &dummy->u_debugreg;
          addr = addr >> 2;
          child->thread.debugreg[addr] = data;
          ret = 0;
        }
        break;
 
    case PTRACE_SYSCALL: /* continue and stop at next (return from) syscall */
    case PTRACE_CONT: { /* restart after signal. */
      long tmp;
 
      ret = -EIO;
      if ((unsigned long) data > _NSIG)
        break;
      if (request == PTRACE_SYSCALL) {
        set_tsk_thread_flag(child, TIF_SYSCALL_TRACE);
      }
      else {
        clear_tsk_thread_flag(child, TIF_SYSCALL_TRACE);
      }
      child->exit_code = data;
    /* make sure the single step bit is not set. */
      tmp = get_stack_long(child, EFL_OFFSET) & ~TRAP_FLAG;
      put_stack_long(child, EFL_OFFSET,tmp);
      wake_up_process(child);
      ret = 0;
      break;
    }
 
  /*
   * make the child exit.  Best I can do is send it a sigkill.
   * perhaps it should be put in the status that it wants to
   * exit.
   */
    case PTRACE_KILL: {
      long tmp;
 
      ret = 0;
      if (child->state == TASK_ZOMBIE)    /* already dead */
        break;
      child->exit_code = SIGKILL;
      /* make sure the single step bit is not set. */
      tmp = get_stack_long(child, EFL_OFFSET) & ~TRAP_FLAG;
      put_stack_long(child, EFL_OFFSET, tmp);
      wake_up_process(child);
      break;
    }
 
    case PTRACE_SINGLESTEP: {  /* set the trap flag. */
      long tmp;
 
      ret = -EIO;
      if ((unsigned long) data > _NSIG)
        break;
      clear_tsk_thread_flag(child, TIF_SYSCALL_TRACE);
      if ((child->ptrace & PT_DTRACE) == 0) {
        /* Spurious delayed TF traps may occur */
        child->ptrace |= PT_DTRACE;
      }
      tmp = get_stack_long(child, EFL_OFFSET) | TRAP_FLAG;
      put_stack_long(child, EFL_OFFSET, tmp);
      child->exit_code = data;
      /* give it a chance to run. */
      wake_up_process(child);
      ret = 0;
      break;
    }
 
    case PTRACE_DETACH:
      /* detach a process that was attached. */
      ret = ptrace_detach(child, data);
      break;
 
    case PTRACE_GETREGS: { /* Get all gp regs from the child. */
        if (!access_ok(VERIFY_WRITE, (unsigned *)data, FRAME_SIZE*sizeof(long))) {
        ret = -EIO;
        break;
      }
      for ( i = 0; i < FRAME_SIZE*sizeof(long); i += sizeof(long) ) {
        __put_user(getreg(child, i),(unsigned long *) data);
        data += sizeof(long);
      }
      ret = 0;
      break;
    }
 
    case PTRACE_SETREGS: { /* Set all gp regs in the child. */
      unsigned long tmp;
        if (!access_ok(VERIFY_READ, (unsigned *)data, FRAME_SIZE*sizeof(long))) {
        ret = -EIO;
        break;
      }
      for ( i = 0; i < FRAME_SIZE*sizeof(long); i += sizeof(long) ) {
        __get_user(tmp, (unsigned long *) data);
        putreg(child, i, tmp);
        data += sizeof(long);
      }
      ret = 0;
      break;
    }
 
    case PTRACE_GETFPREGS: { /* Get the child FPU state. */
      if (!access_ok(VERIFY_WRITE, (unsigned *)data,
               sizeof(struct user_i387_struct))) {
        ret = -EIO;
        break;
      }
      ret = 0;
      if (!child->used_math)
        init_fpu(child);
      get_fpregs((struct user_i387_struct __user *)data, child);
      break;
    }
 
    case PTRACE_SETFPREGS: { /* Set the child FPU state. */
      if (!access_ok(VERIFY_READ, (unsigned *)data,
               sizeof(struct user_i387_struct))) {
        ret = -EIO;
        break;
      }
      child->used_math = 1;
      set_fpregs(child, (struct user_i387_struct __user *)data);
      ret = 0;
      break;
    }
 
    case PTRACE_GETFPXREGS: { /* Get the child extended FPU state. */
      if (!access_ok(VERIFY_WRITE, (unsigned *)data,
               sizeof(struct user_fxsr_struct))) {
        ret = -EIO;
        break;
      }
      if (!child->used_math)
        init_fpu(child);
      ret = get_fpxregs((struct user_fxsr_struct __user *)data, child);
      break;
    }
 
    case PTRACE_SETFPXREGS: { /* Set the child extended FPU state. */
      if (!access_ok(VERIFY_READ, (unsigned *)data,
               sizeof(struct user_fxsr_struct))) {
        ret = -EIO;
        break;
      }
      child->used_math = 1;
      ret = set_fpxregs(child, (struct user_fxsr_struct __user *)data);
      break;
    }
 
    case PTRACE_GET_THREAD_AREA:
      ret = ptrace_get_thread_area(child,
                 addr, (struct user_desc __user *) data);
      break;
 
    case PTRACE_SET_THREAD_AREA:
      ret = ptrace_set_thread_area(child,
                 addr, (struct user_desc __user *) data);
      break;
 
    default:
      ret = ptrace_request(child, request, addr, data);
      break;
    }
out_tsk:
    put_task_struct(child);
out:
    unlock_kernel();
    return ret;
}

```

整体来看较为简单，经过简单的验证后根据不同的`request`参数进入不同的处理流程。

#### 2. 流程梳理

根据源码分析结果，梳理函数的整体处理流程如下（为保证图片清晰，将图片进行了切割）：

 

![](https://bbs.pediy.com/upload/attach/202102/779730_V3JFBX68T9Y96CB.png)

 

![](https://bbs.pediy.com/upload/attach/202102/779730_VKC7JJ5SG8PSSQF.png)

 

上述流程图基本描述清晰了 Linux-2.6 版本下的`sys_ptrace`函数的执行流程。其中可以看参数为到`PTRACE_TRACEME, PTRACE_ATTACH`时进行了特殊处理，其他情况下，流程基本相同，根据不同的`request`的值调用对应的 handler 函数即可。

#### 3. 其他

在 Linux-2.6 版本中，针对不同 platform 设计了不同的函数实现，总体流程上没有改变，只是根据不同的 platform 特点在某些位置坐了不同的处理方式。各 platform 对应的函数实现文件如下：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeiimage-20210205101058491.png)

 

在`kernel/ptrace.c`中实现对公共函数的实现，此处不做过多介绍，感兴趣的师傅可自行研究：

```
/*
 * linux/kernel/ptrace.c
 *
 * (C) Copyright 1999 Linus Torvalds
 *
 * Common interfaces for "ptrace()" which we do not want
 * to continually duplicate across every architecture.
 */
 
... ...
 
/*
 * ptrace a task: make the debugger its new parent and
 * move it to the ptrace list.
 *
 * Must be called with the tasklist lock write-held.
 */
void __ptrace_link(task_t *child, task_t *new_parent)
{
    ... ...
}
 
/*
 * unptrace a task: move it back to its original parent and
 * remove it from the ptrace list.
 *
 * Must be called with the tasklist lock write-held.
 */
void __ptrace_unlink(task_t *child)
{
    ... ...
}
 
/*
 * Check that we have indeed attached to the thing..
 */
int ptrace_check_attach(struct task_struct *child, int kill)
{
    if (!(child->ptrace & PT_PTRACED))
        return -ESRCH;
 
    if (child->parent != current)
        return -ESRCH;
 
    if (!kill) {
        if (child->state != TASK_STOPPED)
            return -ESRCH;
        wait_task_inactive(child);
    }
 
    /* All systems go.. */
    return 0;
}
 
int ptrace_attach(struct task_struct *task)
{
    ... ...
}
 
int ptrace_detach(struct task_struct *child, unsigned int data)
{
    ... ...
}
 
/*
 * Access another process' address space.
 * Source/target buffer must be kernel space,
 * Do not walk the page table directly, use get_user_pages
 */
 
int access_process_vm(struct task_struct *tsk, unsigned long addr, void *buf, int len, int write)
{
    ... ...
}
 
int ptrace_readdata(struct task_struct *tsk, unsigned long src, char __user *dst, int len)
{
    ... ...
}
 
int ptrace_writedata(struct task_struct *tsk, char __user *src, unsigned long dst, int len)
{
    ... ...
}
 
static int ptrace_setoptions(struct task_struct *child, long data)
{
    ... ...
}
 
static int ptrace_getsiginfo(struct task_struct *child, siginfo_t __user * data)
{
    ... ...
}
 
static int ptrace_setsiginfo(struct task_struct *child, siginfo_t __user * data)
{
    ... ...
}
 
int ptrace_request(struct task_struct *child, long request,
           long addr, long data)
{
    int ret = -EIO;
 
    switch (request) {
#ifdef PTRACE_OLDSETOPTIONS
    case PTRACE_OLDSETOPTIONS:
#endif
    case PTRACE_SETOPTIONS:
        ret = ptrace_setoptions(child, data);
        break;
    case PTRACE_GETEVENTMSG:
        ret = put_user(child->ptrace_message, (unsigned long __user *) data);
        break;
    case PTRACE_GETSIGINFO:
        ret = ptrace_getsiginfo(child, (siginfo_t __user *) data);
        break;
    case PTRACE_SETSIGINFO:
        ret = ptrace_setsiginfo(child, (siginfo_t __user *) data);
        break;
    default:
        break;
    }
 
    return ret;
}
 
void ptrace_notify(int exit_code)
{
    BUG_ON (!(current->ptrace & PT_PTRACED));
 
    /* Let the debugger run.  */
    current->exit_code = exit_code;
    set_current_state(TASK_STOPPED);
    notify_parent(current, SIGCHLD);
    schedule();
 
    /*
     * Signals sent while we were stopped might set TIF_SIGPENDING.
     */
 
    spin_lock_irq(¤t->sighand->siglock);
    recalc_sigpending();
    spin_unlock_irq(¤t->sighand->siglock);
}
 
EXPORT_SYMBOL(ptrace_notify);

```

### 2. Linux-5.9 版本的源码分析

#### 1. 源码分析

Linux-5.9 版本的源码分析：

```
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
        unsigned long, data)
{
    struct task_struct *child;
    long ret;
 
    if (request == PTRACE_TRACEME) { // 请求是否为PTRACE_TRACEME
        ret = ptrace_traceme();
        if (!ret)                       
            arch_ptrace_attach(current);
        goto out;
    }
 
    child = find_get_task_by_vpid(pid); // 通过pid请求task结构
    if (!child) {  // 请求失败，返回ESRCH
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
 
  /* 根据不同的架构进行不同的处理 */
    ret = arch_ptrace(child, request, addr, data);
    if (ret || request != PTRACE_DETACH)
        ptrace_unfreeze_traced(child);
 
 out_put_task_struct:
    put_task_struct(child);
 out:
    return ret;
}

```

#### 2. 流程梳理

梳理上述源码，可以得到函数流程图如下：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeiimage-20210205114716666.png)

#### 3. 其他

Linux-5.9 中使用了宏的方式，在进行函数调用时先进行函数替换解析出完整的函数体再进行具体执行（详细替换可参考系列（一）中的函数定义部分内容）。而且与 Linux-2.6 不同的是，`kernel/ptrace.c`负责总体调度，使用`arch_ptrace`进行不同架构的处理的选择：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeiimage-20210205110727411.png)

 

Linux-5.9 版本的这种改动相比 Linux-2.6 的设计，更为清晰也更为安全（个人十分喜欢这种设计，由衷佩服这些优秀的开发者）。

[](#四、request参数详解)四、Request 参数详解
--------------------------------

### 1. 参数简述

`ptrace`总计有 4 个参数，其中比较重要的是第一个参数 --`request`，该参数决定了具体执行的系统调用功能。可取值如下（部分）：

<table><thead><tr><th>Request</th><th>Description</th></tr></thead><tbody><tr><td>PTRACE_TRACEME</td><td>进程被其父进程跟踪，其父进程应该希望跟踪子进程。该值仅被 tracee 使用，其余的 request 值仅被 tracer 使用</td></tr><tr><td>PTRACE_PEEKTEXT, PTRACE_PEEKDATA</td><td>从 tracee 的 addr 指定的内存地址中读取一个字节作为 ptrace() 调用的结果</td></tr><tr><td>PTRACE_PEEKUSER</td><td>从 tracee 的 USER 区域中便宜为 addr 处读取一个字节，该值保存了进程的寄存器和其他信息</td></tr><tr><td>PTRACE_POKETEXT, PTRACE_POKEDATA</td><td>向 tracee 的 addr 内存地址处复制一个字节数据</td></tr><tr><td>PTRACE_POKEUSER</td><td>向 tracee 的 USER 区域中偏移为 addr 地址处复制一个字节数据</td></tr><tr><td>PTRACE_GETREGS</td><td>复制 tracee 的通用寄存器到 tracer 的 data 处</td></tr><tr><td>PTRACE_GETFPREGS</td><td>复制 tracee 的浮点寄存器到 tracer 的 data 处</td></tr><tr><td>PTRACE_GETREGSET</td><td>读取 tracee 的寄存器</td></tr><tr><td>PTRACE_SETREGS</td><td>设置 tracee 的通用寄存器</td></tr><tr><td>PTRACE_SETFPREGS</td><td>设置 tracee 的浮点寄存器</td></tr><tr><td>PTRACE_CONT</td><td>重新运行 stopped 状态的 tracee 进程</td></tr><tr><td>PTRACE_SYSCALL</td><td>重新运行 stopped 状态的 tracee 进程，但是使 tracee 在系统调用的下一个 entry 或从系统调用退出或在执行一条指令后 stop</td></tr><tr><td>PTRACE_SINGLESTEP</td><td>设置单步执行标志</td></tr><tr><td>PTRACE_ATTACH</td><td>跟踪指定 pid 的进程</td></tr><tr><td>PTRACE_DETACH</td><td>结束跟踪</td></tr></tbody></table>

 

备注：上述参数中，`PTRACE_GETREGS, PTRACE_SETREGS, PTRACE_GETFPREGS, PTRACE_SETFPREGS`参数为 Interl386 特有。

 

各参数所代表的值由`/usr/include/sys/ptrace.h`文件指定：

 

![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebei20210201202646.png)

### 2. 重要参数详解

下面将对`request`中几个常见、重要的参数进行详细解析：

#### 1. PTRACE_TRACEME

1.  描述  
    本进程被其父进程跟踪，如果子进程没有被其父进程跟踪，不能使用该选项。`PTRACE_TRACEME` 只被`tracee`使用。
    
2.  定义
    
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
     
        write_lock_irq(&tasklist_lock);   // 首先让writer拿到读写lock，并且会disable local irp
     
        /* Are we already being traced? */
     
        // 是否已经处于ptrace中
        if (!current->ptrace) {
            ret = security_ptrace_traceme(current->parent);
            /*
            * Check PF_EXITING to ensure ->real_parent has not passed
            * exit_ptrace(). Otherwise we don't report the error but
            * pretend ->real_parent untraces us right after return.
            */
            if (!ret && !(current->real_parent->flags & PF_EXITING)) {
                // 检查通过，将子进程链接到父进程的ptrace链表中
                current->ptrace = PT_PTRACED;
                ptrace_link(current, current->real_parent);
            }
        }
     
        write_unlock_irq(&tasklist_lock);
     
        return ret;
    }
    
    ```
    
3.  分析
    
    通过分析源码我们可以明确看到，`PTRACE_TRACEME`并没有真正使子进程停止。它内部完成的操作只有对父进程是否能对子进程进行 trace 的合法性检查，然后将子进程链接到父进程的饿 ptrace 链表中。真正导致子进程停止的是`exec`系统调用。
    
    在系统调用成功后，kernel 会判断该进程是否被 ptrace 跟踪。如果处于跟踪状态，kernel 将会向该进程发送`SIGTRAP`信号，正是该信号导致了当前进程的停止。
    
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
    
    在`exec.c`中对该函数的调用如下：
    
    ```
    static int exec_binprm(struct linux_binprm *bprm)
    {
        pid_t old_pid, old_vpid;
        int ret, depth;
     
        /* Need to fetch pid before load_binary changes it */
        old_pid = current->pid;
        rcu_read_lock();
        old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
        rcu_read_unlock();
        .......
        audit_bprm(bprm);
        trace_sched_process_exec(current, old_pid, bprm);
     
      // 调用ptrace_event,传入的event为PTRACE_EVENT_EXEC
      // 直接走发送SIGTRAP的逻辑
        ptrace_event(PTRACE_EVENT_EXEC, old_vpid); 
     
        proc_exec_connector(current);
        return 0;
    }
    
    ```
    
    `SIGTRAP`信号的值为 5，专门为调试设计。当 kernel 发生`int 3`时，触发回掉函数`do_trap()`，其代码如下：
    
    ```
    asmlinkage void do_trap(struct pt_regs *regs, unsigned long address)
    {
        force_sig_fault(SIGTRAP, TRAP_TRACE, (void __user *)address);
     
        regs->pc += 4;
    }
     
    int force_sig_fault(int sig, int code, void __user *addr
        ___ARCH_SI_TRAPNO(int trapno)
        ___ARCH_SI_IA64(int imm, unsigned int flags, unsigned long isr))
    {
        return force_sig_fault_to_task(sig, code, addr
                           ___ARCH_SI_TRAPNO(trapno)
                           ___ARCH_SI_IA64(imm, flags, isr), current);
    }
    
    ```
    
    父进程唤醒`wait`对子进程进行监控，`wait`有 3 种退出情况（子进程正常退出、收到信号退出、收到信号暂停），对于`PTRACE_TRACEME`来说，对应的是第三种情况 -- 收到信号后暂停。
    
    `PTRACE_TRACEME`只是表明了子进程可以被 trace，如果进程调用了`PTRACE_TRACEME`，那么该进程处理信号的方式会发生改变。例如一个进程正在运行，此时输入`ctrl+c(SIGINT)`, 则进程会直接退出；如果进程中有`ptrace (PTRACE_TRACEME,0，NULL,NULL)`，当输入`CTRL+C`时，该进程将会处于 stopped 的状态。
    
    在`sys_ptrace`函数中，该部分的处理流程如下：
    
    ![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeiimage-20210205142516160.png)
    
    在 5.9 版中，单独写成了`ptrace_traceme()`函数，而在 2.6 版本中，直接在`sys_ptrace`的逻辑中进行实现：
    
    ![](https://cdn.jsdelivr.net/gh/AlexsanderShaw/BlogImages@main/img/vuln/shebeiimage-20210205142732193.png)
    
    虽然 2 个版本的核心功能相同，但是 5.9 版本的处理逻辑和情况考量相比 2.6 版本上升了很大高度。
    

#### 2. PTRA CE_ATTACH

1.  描述
    
    attach 到 pid 指定的进程，使其成为调用进程的`tracee`。`tracer`会向`tracee`发送一个`SIGSTOP`信号，但不一定已通过此调用完成而停止；`tracer`使用`waitpid()`等待`tracee`停止。
    
2.  定义
    
    ```
    static int ptrace_attach(struct task_struct *task, long request,
                 unsigned long addr,
                 unsigned long flags)
    {
        bool seize = (request == PTRACE_SEIZE);
        int retval;
     
        retval = -EIO;  /* I/O error*/
     
        /*
        * 判断request是PTRACE_SEIZE还是PTRACE_ATTACH。
        * 如果request为PTRACE_SEIZE，则进行必要的参数检查，错误时退出。
        */
        if (seize) {
            if (addr != 0)
                goto out;
            if (flags & ~(unsigned long)PTRACE_O_MASK)
                goto out;
            flags = PT_PTRACED | PT_SEIZED | (flags << PT_OPT_FLAG_SHIFT);
        } else {
            flags = PT_PTRACED;
        }
     
        audit_ptrace(task);
     
        /*
        * 判断task进程是否为kernel thread（PF_KTHREAD），
        * 调用same_thread_group(task, current)，判断task是否和current进程在同一个线程组，查看current进程是否有权限trace task进程。
        * 如果不符合要求，则直接退出。
        */
        retval = -EPERM;  /* Operation not permitted, retval = -1 */
        if (unlikely(task->flags & PF_KTHREAD))
            goto out;
        if (same_thread_group(task, current))
            goto out;
     
        /*
         * Protect exec's credential calculations against our interference;
         * SUID, SGID and LSM creds get determined differently
         * under ptrace.
         */
        retval = -ERESTARTNOINTR;
        if (mutex_lock_interruptible(&task->signal->cred_guard_mutex))
            goto out;
     
        task_lock(task);
        retval = __ptrace_may_access(task, PTRACE_MODE_ATTACH_REALCREDS);
        task_unlock(task);
        if (retval)
            goto unlock_creds;
     
        write_lock_irq(&tasklist_lock);
        retval = -EPERM;
        if (unlikely(task->exit_state))
            goto unlock_tasklist;
        if (task->ptrace)
            goto unlock_tasklist;
     
        /*
         * 设置子进程task->ptrace = PT_TRACED，被跟踪状态
         */
        if (seize)
            flags |= PT_SEIZED;
        task->ptrace = flags;
     
        /*
         * 调用__ptrace_link(task, current)，将task->ptrace_entry链接到current->ptraced链表中。
         */
        ptrace_link(task, current);
     
        /* SEIZE doesn't trap tracee on attach */
        /*
         * 如果是PTRACE_ATTACH请求（PTRACE_SEIZE请求不会停止被跟踪进程），
         * 则调用send_sig_info(SIGSTOP,SEND_SIG_PRIV, task);
         * 发送SIGSTOP信号，中止task运行，设置task->state为TASK_STOPPED
         */
        if (!seize)
            send_sig_info(SIGSTOP, SEND_SIG_PRIV, task);
     
        spin_lock(&task->sighand->siglock);
     
        /*
         * If the task is already STOPPED, set JOBCTL_TRAP_STOP and
         * TRAPPING, and kick it so that it transits to TRACED.  TRAPPING
         * will be cleared if the child completes the transition or any
         * event which clears the group stop states happens.  We'll wait
         * for the transition to complete before returning from this
         * function.
         *
         * This hides STOPPED -> RUNNING -> TRACED transition from the
         * attaching thread but a different thread in the same group can
         * still observe the transient RUNNING state.  IOW, if another
         * thread's WNOHANG wait(2) on the stopped tracee races against
         * ATTACH, the wait(2) may fail due to the transient RUNNING.
         *
         * The following task_is_stopped() test is safe as both transitions
         * in and out of STOPPED are protected by siglock.
         *
         *
         *
         * 等待task->jobctl的JOBCTL_TRAPPING_BIT位被清零，
         * 阻塞时进程状态被设置为TASK_UNINTERRUPTIBLE，引发进程调度
         */
     
        if (task_is_stopped(task) &&
            task_set_jobctl_pending(task, JOBCTL_TRAP_STOP | JOBCTL_TRAPPING))
            signal_wake_up_state(task, __TASK_STOPPED);
     
        spin_unlock(&task->sighand->siglock);
     
        retval = 0;
    unlock_tasklist:
        write_unlock_irq(&tasklist_lock);
    unlock_creds:
        mutex_unlock(&task->signal->cred_guard_mutex);
    out:
        if (!retval) {
            /*
             * We do not bother to change retval or clear JOBCTL_TRAPPING
             * if wait_on_bit() was interrupted by SIGKILL. The tracer will
             * not return to user-mode, it will exit and clear this bit in
             * __ptrace_unlink() if it wasn't already cleared by the tracee;
             * and until then nobody can ptrace this task.
             */
            wait_on_bit(&task->jobctl, JOBCTL_TRAPPING_BIT, TASK_KILLABLE);
            proc_ptrace_connector(task, PTRACE_ATTACH);
        }
     
        return retval;
    }
    
    ```
    
3.  分析
    
    代码上可以看出，`PTRACE_ATTACH`处理的方式与`PTRACE_TRACEME`处理的方式不同。`PTRACE_ATTACH`会使父进程直接向子进程发送`SIGSTOP`信号，如果子进程停止，那么父进程的`wait`操作被唤醒，从而成功 attach。一个进程不能 attach 多次。
    
    在 2.6 版本中的实现如下 (`kernel/ptrace.c`)：
    
    ```
    int ptrace_attach(struct task_struct *task)
    {
        int retval;
        task_lock(task);
        retval = -EPERM;
        if (task->pid <= 1)        // 不能调试init
            goto bad;
        if (task == current)  // 不能调试自身
            goto bad;
        if (!task->mm)
            goto bad;
      /* 鉴权 */
        if(((current->uid != task->euid) ||
            (current->uid != task->suid) ||
            (current->uid != task->uid) ||
             (current->gid != task->egid) ||
             (current->gid != task->sgid) ||
             (current->gid != task->gid)) && !capable(CAP_SYS_PTRACE))
            goto bad;
        rmb();
        if (!task->mm->dumpable && !capable(CAP_SYS_PTRACE))
            goto bad;
     
        if (task->ptrace & PT_PTRACED)   // 一个进程不能被attach多次
            goto bad;
        retval = security_ptrace(current, task);
        if (retval)
            goto bad;
     
        /* Go */
        task->ptrace |= PT_PTRACED;
        if (capable(CAP_SYS_PTRACE))
            task->ptrace |= PT_PTRACE_CAP;
        task_unlock(task);
     
        write_lock_irq(&tasklist_lock);
        __ptrace_link(task, current);      // 调用__ptrace_link(task, current)，将task->ptrace_entry链接到current->ptraced链表中
        write_unlock_irq(&tasklist_lock);
     
        force_sig_specific(SIGSTOP, task);  // 发送SIGSTOP，终止运行
        return 0;
     
    bad:
        task_unlock(task);
        return retval;
    }
    
    ```
    

（未完待续）

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

最后于 3 小时前 被有毒编辑 ，原因：