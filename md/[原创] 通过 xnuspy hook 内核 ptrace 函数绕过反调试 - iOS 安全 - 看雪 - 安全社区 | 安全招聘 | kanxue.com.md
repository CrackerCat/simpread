> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282045.htm)

> [原创] 通过 xnuspy hook 内核 ptrace 函数绕过反调试

> 昨天实践了[修改 kernproc 结构体绕过反调试](https://iosre.com/t/%E5%9F%BA%E4%BA%8E%E4%BF%AE%E6%94%B9ios%E5%86%85%E6%A0%B8%E7%BB%95%E8%BF%87ios-%E5%9F%BA%E4%BA%8Esvc-0x80%E7%9A%84ptrace%E5%8F%8D%E8%B0%83%E8%AF%95/24496). 今天突然想到是不是直接内核 hook 更方便点儿

### 0x01 先阅读自己手机系统 _iPhone 7 的 14.1_ 对应的 xnu 开源代码里面的 ptrace 代码

*   xnu-xnu-7195.50.7.100.1/bsd/kern/mach_process.c

```
int
ptrace(struct proc *p, struct ptrace_args *uap, int32_t *retval)
{
    struct proc     *t; /* target process */
    task_t          task;
    thread_t        th_act;
    struct uthread  *ut;
    int tr_sigexc = 0;
    int error = 0;
    int stopped = 0;
 
    AUDIT_ARG(cmd, uap->req);
    AUDIT_ARG(pid, uap->pid);
    AUDIT_ARG(addr, uap->addr);
    AUDIT_ARG(value32, uap->data);
 
    if (uap->req == PT_DENY_ATTACH) {
#if (DEVELOPMENT || DEBUG) && !defined(XNU_TARGET_OS_OSX)
        if (PE_i_can_has_debugger(NULL)) {
            return 0;
        }
#endif
        proc_lock(p);
        if (ISSET(p->p_lflag, P_LTRACED)) {
            proc_unlock(p);
            KERNEL_DEBUG_CONSTANT(BSDDBG_CODE(DBG_BSD_PROC, BSD_PROC_FRCEXIT) | DBG_FUNC_NONE,
                p->p_pid, W_EXITCODE(ENOTSUP, 0), 4, 0, 0);
            exit1(p, W_EXITCODE(ENOTSUP, 0), retval);
 
            thread_exception_return();
            /* NOTREACHED */
        }
        SET(p->p_lflag, P_LNOATTACH);
        proc_unlock(p);
 
        return 0;
    }
 
    if (uap->req == PT_FORCEQUOTA) {
        if (kauth_cred_issuser(kauth_cred_get())) {
            t = current_proc();
            OSBitOrAtomic(P_FORCEQUOTA, &t->p_flag);
            return 0;
        } else {
            return EPERM;
        }
    }
 
    /*
     *  Intercept and deal with "please trace me" request.
     */
    if (uap->req == PT_TRACE_ME) {
retry_trace_me: ;
        proc_t pproc = proc_parent(p);
        if (pproc == NULL) {
            return EINVAL;
        }
#if CONFIG_MACF
        /*
         * NB: Cannot call kauth_authorize_process(..., KAUTH_PROCESS_CANTRACE, ...)
         *     since that assumes the process being checked is the current process
         *     when, in this case, it is the current process's parent.
         *     Most of the other checks in cantrace() don't apply either.
         */
        struct proc_ident p_ident = proc_ident(p);
        struct proc_ident pproc_ident = proc_ident(pproc);
        kauth_cred_t pproc_cred = kauth_cred_proc_ref(pproc);
 
        proc_rele(pproc);
        error = mac_proc_check_debug(&pproc_ident, pproc_cred, &p_ident);
        kauth_cred_unref(&pproc_cred);
 
        if (error != 0) {
            return error;
        }
        if (proc_find_ident(&pproc_ident) == PROC_NULL) {
            return ESRCH;
        }
#endif
        proc_lock(p);
        /* Make sure the process wasn't re-parented. */
        if (p->p_ppid != pproc->p_pid) {
            proc_unlock(p);
            proc_rele(pproc);
            goto retry_trace_me;
        }
        SET(p->p_lflag, P_LTRACED);
        /* Non-attached case, our tracer is our parent. */
        p->p_oppid = p->p_ppid;
        proc_unlock(p);
        /* Child and parent will have to be able to run modified code. */
        cs_allow_invalid(p);
        cs_allow_invalid(pproc);
        proc_rele(pproc);
 
        return error;
    }
    if (uap->req == PT_SIGEXC) {
        proc_lock(p);
        if (ISSET(p->p_lflag, P_LTRACED)) {
            SET(p->p_lflag, P_LSIGEXC);
            proc_unlock(p);
            return 0;
        } else {
            proc_unlock(p);
            return EINVAL;
        }
    }
 
    /*
     * We do not want ptrace to do anything with kernel or launchd
     */
    if (uap->pid < 2) {
        return EPERM;
    }
 
    /*
     *  Locate victim, and make sure it is traceable.
     */
    if ((t = proc_find(uap->pid)) == NULL) {
        return ESRCH;
    }
 
    AUDIT_ARG(process, t);
 
    task = t->task;
    if (uap->req == PT_ATTACHEXC) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
        uap->req = PT_ATTACH;
        tr_sigexc = 1;
    }
    if (uap->req == PT_ATTACH) {
#pragma clang diagnostic pop
        int             err;
 
#if !defined(XNU_TARGET_OS_OSX)
        if (tr_sigexc == 0) {
            error = ENOTSUP;
            goto out;
        }
#endif
 
        if (kauth_authorize_process(proc_ucred(p), KAUTH_PROCESS_CANTRACE,
            t, (uintptr_t)&err, 0, 0) == 0) {
            /* it's OK to attach */
            proc_lock(t);
            SET(t->p_lflag, P_LTRACED);
            if (tr_sigexc) {
                SET(t->p_lflag, P_LSIGEXC);
            }
 
            t->p_oppid = t->p_ppid;
            /* Check whether child and parent are allowed to run modified
             * code (they'll have to) */
            proc_unlock(t);
            cs_allow_invalid(t);
            cs_allow_invalid(p);
            if (t->p_pptr != p) {
                proc_reparentlocked(t, p, 1, 0);
            }
 
            proc_lock(t);
            if (get_task_userstop(task) > 0) {
                stopped = 1;
            }
            t->p_xstat = 0;
            proc_unlock(t);
            psignal(t, SIGSTOP);
            /*
             * If the process was stopped, wake up and run through
             * issignal() again to properly connect to the tracing
             * process.
             */
            if (stopped) {
                task_resume(task);
            }
            error = 0;
            goto out;
        } else {
            error = err;
            if (error == ESRCH) {
                /*
                 * The target 't' is not valid anymore as it
                 * could not be found after the MAC check.
                 */
                return error;
            }
            /* not allowed to attach, proper error code returned by kauth_authorize_process */
            if (ISSET(t->p_lflag, P_LNOATTACH)) {
                psignal(p, SIGSEGV);
            }
            goto out;
        }
    }
 
    /*
     * You can't do what you want to the process if:
     *  (1) It's not being traced at all,
     */
    proc_lock(t);
    if (!ISSET(t->p_lflag, P_LTRACED)) {
        proc_unlock(t);
        error = EPERM;
        goto out;
    }
 
    /*
     *  (2) it's not being traced by _you_, or
     */
    if (t->p_pptr != p) {
        proc_unlock(t);
        error = EBUSY;
        goto out;
    }
 
    /*
     *  (3) it's not currently stopped.
     */
    if (t->p_stat != SSTOP) {
        proc_unlock(t);
        error = EBUSY;
        goto out;
    }
 
    /*
     *  Mach version of ptrace executes request directly here,
     *  thus simplifying the interaction of ptrace and signals.
     */
    /* proc lock is held here */
    switch (uap->req) {
    case PT_DETACH:
        if (t->p_oppid != t->p_ppid) {
            struct proc *pp;
 
            proc_unlock(t);
            pp = proc_find(t->p_oppid);
            if (pp != PROC_NULL) {
                proc_reparentlocked(t, pp, 1, 0);
                proc_rele(pp);
            } else {
                /* original parent exited while traced */
                proc_list_lock();
                t->p_listflag |= P_LIST_DEADPARENT;
                proc_list_unlock();
                proc_reparentlocked(t, initproc, 1, 0);
            }
            proc_lock(t);
        }
 
        t->p_oppid = 0;
        CLR(t->p_lflag, P_LTRACED);
        CLR(t->p_lflag, P_LSIGEXC);
        proc_unlock(t);
        goto resume;
 
    case PT_KILL:
        /*
         *  Tell child process to kill itself after it
         *  is resumed by adding NSIG to p_cursig. [see issig]
         */
        proc_unlock(t);
#if CONFIG_MACF
        error = mac_proc_check_signal(p, t, SIGKILL);
        if (0 != error) {
            goto resume;
        }
#endif
        psignal(t, SIGKILL);
        goto resume;
 
    case PT_STEP:                   /* single step the child */
    case PT_CONTINUE:               /* continue the child */
        proc_unlock(t);
        th_act = (thread_t)get_firstthread(task);
        if (th_act == THREAD_NULL) {
            error = EINVAL;
            goto out;
        }
 
        /* force use of Mach SPIs (and task_for_pid security checks) to adjust PC */
        if (uap->addr != (user_addr_t)1) {
            error = ENOTSUP;
            goto out;
        }
 
        if ((unsigned)uap->data >= NSIG) {
            error = EINVAL;
            goto out;
        }
 
        if (uap->data != 0) {
#if CONFIG_MACF
            error = mac_proc_check_signal(p, t, uap->data);
            if (0 != error) {
                goto out;
            }
#endif
            psignal(t, uap->data);
        }
 
        if (uap->req == PT_STEP) {
            /*
             * set trace bit
             * we use sending SIGSTOP as a comparable security check.
             */
#if CONFIG_MACF
            error = mac_proc_check_signal(p, t, SIGSTOP);
            if (0 != error) {
                goto out;
            }
#endif
            if (thread_setsinglestep(th_act, 1) != KERN_SUCCESS) {
                error = ENOTSUP;
                goto out;
            }
        } else {
            /*
             * clear trace bit if on
             * we use sending SIGCONT as a comparable security check.
             */
#if CONFIG_MACF
            error = mac_proc_check_signal(p, t, SIGCONT);
            if (0 != error) {
                goto out;
            }
#endif
            if (thread_setsinglestep(th_act, 0) != KERN_SUCCESS) {
                error = ENOTSUP;
                goto out;
            }
        }
resume:
        proc_lock(t);
        t->p_xstat = uap->data;
        t->p_stat = SRUN;
        if (t->sigwait) {
            wakeup((caddr_t)&(t->sigwait));
            proc_unlock(t);
            if ((t->p_lflag & P_LSIGEXC) == 0) {
                task_resume(task);
            }
        } else {
            proc_unlock(t);
        }
 
        break;
 
    case PT_THUPDATE:  {
        proc_unlock(t);
        if ((unsigned)uap->data >= NSIG) {
            error = EINVAL;
            goto out;
        }
        th_act = port_name_to_thread(CAST_MACH_PORT_TO_NAME(uap->addr),
            PORT_TO_THREAD_NONE);
        if (th_act == THREAD_NULL) {
            error = ESRCH;
            goto out;
        }
        ut = (uthread_t)get_bsdthread_info(th_act);
        if (uap->data) {
            ut->uu_siglist |= sigmask(uap->data);
        }
        proc_lock(t);
        t->p_xstat = uap->data;
        t->p_stat = SRUN;
        proc_unlock(t);
        thread_deallocate(th_act);
        error = 0;
    }
    break;
    default:
        proc_unlock(t);
        error = EINVAL;
        goto out;
    }
 
    error = 0;
out:
    proc_rele(t);
    return error;
}

```

*   通过阅读源码发现只需要修改第二个参数的结构体属性 uap->req 就可以达到绕过目的 (uap->req == PT_DENY_ATTACH)

### 定位内核 ptrace 函数偏移地址

![](https://bbs.kanxue.com/upload/attach/202406/757881_YT6TMVTEZPC9WHX.jpg)

### 分析第二个参数 ptrace_args 的内部结构

让我们详细分析 `struct ptrace_args` 在内存中的布局。在 64 位系统上，考虑到内存对齐和每个成员的大小，下面是详细的内存布局。

假设 `pid_t` 是 4 字节，`user_addr_t` 是 8 字节。在 64 位系统上，指针类型通常是 8 字节并且需要对齐到 8 字节边界。

首先，结构体定义如下：

```
struct ptrace_args {
    int req;         // 4 bytes
    void * unknow[4];        //和实际内存的中偏移不符.实际pid 是+8，所以加了unknow填充
    pid_t pid;       // 4 bytes
    user_addr_t addr;// 8 bytes (on a 64-bit system)
    int data;        // 4 bytes
};

```

### 内存布局和对齐分析：

1.  `int req`:
    
    *   大小: 4 bytes
    *   对齐: 4 bytes
    *   偏移: 0
    *   结束偏移: 4
2.  `pid_t pid`:
    
    *   大小: 4 bytes
    *   对齐: 4 bytes
    *   偏移: 4（紧接在 `req` 之后）
    *   结束偏移: 8
3.  `user_addr_t addr`:
    
    *   大小: 8 bytes
    *   对齐: 8 bytes
    *   由于 `addr` 需要 8 字节对齐，而当前偏移 8 已经对齐到 8 字节边界，因此无需填充
    *   偏移: 8
    *   结束偏移: 16
4.  `int data`:
    
    *   大小: 4 bytes
    *   对齐: 4 bytes
    *   偏移: 16（紧接在 `addr` 之后）
    *   结束偏移: 20

由于数据对齐要求，编译器可能会在结构体的末尾添加填充以确保结构体大小是对齐的。因此，整个结构体的大小是 24 字节（4 + 4 + 8 + 4 + 4（填充））。

### 详细内存布局：

```
+-------------+--------------+--------------+--------------+
| req (4 bytes)               | unknow (4 bytes)           |
+-------------+--------------+--------------+--------------+
| pid (4 bytes)               |     padding (4 bytes)      |
+-------------+--------------+--------------+--------------+
| addr (8 bytes)                                                  |
+-------------+--------------+--------------+--------------+
| data (4 bytes)                | padding (4 bytes)        |
+-------------+--------------+--------------+--------------+

```

### 解释：

*   `req` 占用偏移 0 到 3。
*   `unknow` 紧接其后，占用偏移 4 到 7。
*   `pid` 占用偏移 8 到 15。

这就是 `struct ptrace_args` 在内存中的布局。

#### 编写代码

```
static int (*ptrace_orig)(struct proc *p, struct ptrace_args *uap, int32_t *retval);
 
static int my_ptrace(struct proc *p, struct ptrace_args *uap, int32_t *retval){
 
 
 
    kprintf("\n ptrace -- start \n");
 
    uint8_t cpu = curcpu();
    pid_t caller = caller_pid();
 
    char *caller_name = unified_kalloc(MAXCOMLEN + 1);
 
    if(!caller_name){
    kprintf("\n ptrace -- caller_name \n");
        return ptrace_orig(p,uap,retval);
    }
     
 
    /* proc_name doesn't bzero for some version of iOS 13 */
    _bzero(caller_name, MAXCOMLEN + 1);
    proc_name(caller, caller_name, MAXCOMLEN + 1);
 
    kprintf("$$$$$$$$$$$$ %s: (CPU %d): '%s' (%d) ptrace_hook  to req : %d pid : %d  \n", __func__, cpu,
            caller_name, caller,uap->req,uap->pid);
 
 
 
    unified_kfree(caller_name);
 
 
    if (uap->req == 31 || uap->req == 14)
    {
        /* code */
 
 
   
        kprintf("uap->req == 31 uap->req %d \n",uap->req);
         
        kprintf("uap->req == 31 uap->pid %d \n",uap->pid);
         
        kprintf("uap->req == 31 uap->addr %d \n",uap->addr);
        kprintf("uap->req == 31 uap->data %d \n",uap->data);
 
         
        return 0;
   
 
    }
 
        return ptrace_orig(p,uap,retval);
}
 
    /* iPhone 7 14.1 */                                //FFFFFFF007584D9C                         
    ret = syscall(SYS_xnuspy_ctl, XNUSPY_INSTALL_HOOK, 0xFFFFFFF007584D9C,
            my_ptrace, &ptrace_orig);

```

[完整代码在 ptrace_hook](https://github.com/yuzhouheike/xnuspy/blob/master/example/ptrace_hook.c)

#### 还是拿宇宙最强 App 抖音测试下 . ****成功!****

![](https://bbs.kanxue.com/upload/attach/202406/757881_ZJWKTRMDAT59BC4.jpg)

#### 测试下自己的 app

```
void antiDebug() {
    // ARM syscall to invoke ptrace with PT_DENY_ATTACH
     
     
 
     
#ifdef __arm64__
        asm volatile(
            "mov x0,#26\n"
            "mov x1,#31\n"
            "mov x2,#0\n"
            "mov x3,#0\n"
            "mov x16,#0\n"
            "svc #128\n"
        );
#endif
    __asm__ volatile(
        "mov x0, #31\n"        // PT_DENY_ATTACH
        "mov x1, #0\n"
        "mov x2, #0\n"
        "mov x3, #0\n"
        "mov x8, #26\n"        // syscall number for ptrace
        "svc #0x80\n"
    );
}
 
void antiDebug1() {
     
     
   
    typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
 
//    ptrace(PT_DENY_ATTACH, 0, 0, 0);
 
    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = (ptrace_ptr_t)dlsym(handle, "ptrace");
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
}

```

### 运行 app 后也成功 hook

![](https://bbs.kanxue.com/upload/attach/202406/757881_85BYTKCWYKNE3U9.jpg)

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌 握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 12 小时前 被 yuzhouheike 编辑 ，原因：

[#基础理论](forum-166-1-188.htm) [#HOOK 注入](forum-166-1-192.htm) [#逆向分析](forum-166-1-189.htm)