> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273160.htm)

> [原创]SVC 的 TraceHook 沙箱的实现 & 无痕 Hook 实现思路

[](#前言：)前言：
-----------

二年前因为工作需要，自己尝试开发过一套沙盒，普通的 Linux IO 函数可以通过 Hook Libc 去实现，比如 Hook openat 等。大部分文件都可以进行处理（比如 VA 的 IO 处理）。但是一直有个心病困扰我半年之久，就是如何处理**系统调用 / 内联 SVC 指令**的拦截和处理。

 

大厂基本为了程序的安全，会使用大量内联 SVC 去调用系统函数，以此来保护程序的安全。以防被 Hook。

 

如何实现 SVC 指令的 IO 重定向，成为最大的问题。尝试在国内查找这块资料，发现基本很少，大部分都是介绍基础而不是去讲如何进行 Hook 和修改，还有的就是通过刷机改源码的方式，但是大部分大厂都是有自己的真机库，基本谷歌系列，很容易就被认定为危险设备。如果通过修改型号等方式去 mock 成国内的型号也可以，但是这种方式有弊端，如果有的 App 调用三方服务，比如小米会自带一些系统服务，这些是没办法进行 mock 的。很容导致 app 崩溃 。通过不断去 Google 去查阅大量文章，问了很多老外，看代码后来发现一套比较成熟的方案就是 **ptrace+seccomp**，两者缺一不可。

[](#前奏知识：)前奏知识：
---------------

### [](#什么是svc指令？什么是syscall？)什么是 SVC 指令？什么是 Syscall？

根据我个人的理解，在 Linux 里面内存主要分为 Linux 用户态，内核态。

 

当用户自定义运行的函数在用户态。内核态是当 Linux 需要处理文件，或者进行中断 IO 等操作的时候就会进入内核态。

 

syscall 就是这个内核态的入口。而 syscall 函数里面的实现就是一段汇编（具体实现参考如下），汇编里面便是调用了 svc 的这条指令。

 

当 arm 系列 cpu 发现 svc 指令的时候，就会陷入中断，简称 0x80 中断。开始执行内核态逻辑，这个时候程序会进入暂停状态。

 

优先去执行内核的逻辑。以此保证程序的安全性。（当我们自己去设计系统的时候，肯定也不希望在系统执行的时候被程序去干扰，导致系统崩溃）

 

Linux 内核本身提供很多函数，比如常见的文件函数, openat，execve 都是 Linux 内核提供的。这些函数都可以通过 svc 指令的方式去调用，只是实现的 sysnum 不一样。传入的参数不一样而已。

 

通过 svc 执行的函数无法进行 inlinehook Hook ，所以会提升程序的安全度。

 

总结：

 

**svc 是一条 arm 指令，Syscall 函数是 libc 函数，实现底层使用了 svc 指令。**

 

syscall 32&64 位具体实现如下。

#### [](#32位：)32 位：

```
raw_syscall:
        MOV             R12, SP
        STMFD           SP!, {R4-R7}
        MOV             R7, R0
        MOV             R0, R1
        MOV             R1, R2
        MOV             R2, R3
        LDMIA           R12, {R3-R6}
        SVC             0
        LDMFD           SP!, {R4-R7}
        mov             pc, lr
```

#### [](#64位：)64 位：

```
raw_syscall:
        MOV             X8, X0
        MOV             X0, X1
        MOV             X1, X2
        MOV             X2, X3
        MOV             X3, X4
        MOV             X4, X5
        MOV             X5, X6
        SVC             0
        RET
```

### 什么是 Ptrace&Seccomp ？

#### Ptrace:

ptrace 是 linux 提供的调试函数，很多好用的工具，IDA ,LLDB 等调试器都是通过 ptrace 去实现的。

 

这个函数里面有很多 action 每个 action 都包含一个功能，比如注入进程，暂停，修改寄存器等常用功能。

 

ptrace 当注入当前进程的时候是不需要 root。如果注入非自己的进程是需要 root 才可以。调用注入的时候选择一个 pid 即可。

 

ptrace 可以在任何内存地方下断点，修改对应位置的数据。

 

**ptrace 的权限非常高。ptrace 还可以调试内核态。**

 

所以也可以用来处理 svc 的参数和返回值。

 

ptrace 具体 API 说明官方文档如下：

 

https://man7.org/linux/man-pages/man2/ptrace.2.html

#### Seccomp:

Seccomp 是 Linux 的一种安全机制，android 8.1 以上使用了 Seccomp

 

**主要功能是限制直接通过 syscall 去调用某些系统函数**，当开启了 Seccomp 的进程在此调用的时候会变走异常的回调。

 

之前 B 佬的文章里面便采用了 frida+seccomp 的方式去做的 svc 拦截。也是很好的思路，帖子地址如下。

 

https://bbs.pediy.com/thread-271815.htm

 

Seccomp 的过滤模式有两种 (strict&filter)，

##### strict

strict 模式如果开启以后，只支持四个函数的系统调用 (read,write,exit,rt_sigreturn)

 

如果一旦使用了其他的 syscall 则会收到 SIGKILL 信号

```
#include #include #include #include #include #include int main(int argc, char **argv)
{
int output = open(“output.txt”, O_WRONLY);
const char *val = “test”;
//通过prctl函数设置seccomp的模式为strict
printf(“Calling prctl() to set seccomp strict mode…\n”);
 
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
 
printf(“Writing to an already open file…\n”);
//尝试写入
write(output, val, strlen(val)+1);
printf(“Trying to open file for reading…\n”);
 
//设置完毕seccomp以后再次尝试open （因为设置了secomp的模式是strict，所以这行代码直接sign -9 信号）
int input = open(“output.txt”, O_RDONLY);
printf(“You will not see this message — the process will be killed first\n”);
} 
```

##### [](#filter（bpf）)filter（BPF）

全程 Seccomp-bpf，BPF 是一种过滤模式，只有在 Linux 高版本会存在该功能，当某进程调用了 svc 以后，

 

如果发现当前 sysnum 是我们进行过滤的 sysnum，首先会进入我们自己写的 BPF 规则

 

通过我们自己的写的规则，进行判断该系统调用是否被运行调用，应该怎么进行处理，常用的指令如下

```
BPF_LD, BPF_LDX加载指令
BPF_ST, BPF_STX存储指令
BPF_ALU, 计算指令
BPF_JMP, 跳转指令
BPF_RET, 返回指令 （结束指令）
BPF_MISC 其他指令
```

指令之间可以相加或者相减，来完成一条 JUMP 操作，这块挺复杂的。具体就不详细去说了。

 

如果对这块规则感兴趣可以看一本书《Linux 内核观测技术 BPF》，老外写的，里面很详细的介绍了 BPF 得使用规则。

 

包括如何配合 Seccomp 去做系统调用的拦截和 trace。微信读书上面就有不需要去购买纸质版本。

[](#开发过程：)开发过程：
---------------

### ptrace:

根据之前介绍的思路，第一版本主要通过 ptrace 去实现 svc 的参数 / 返回值的修改。首先先 fork 出来一条线程。用于跟踪 main 进程。

 

开启死循环，当使用 ptrace 的时候需要区分，**调试线程（tracer）和被调试线程 (tracee)**，他们是两条线程。

```
/*
 * 用fork出来的进程去attch主进程
 */
int trace_current_process(int sdkVersion) {
    ALOGE("start trace_current_process ");
    prctl(PR_SET_DUMPABLE, 1, 0, 0, 0);
    mainProcessPid = getpid();
    pid_t child = fork();
    if (child < 0) {
        ALOGE("ptrace svc  fork() error ");
        return -errno;
    }
    //init first tracer
    Tracer *first = get_tracer(nullptr, mainProcessPid, true);
    if (child == 0) {
        // attch main pid
        int status = ptrace(PTRACE_ATTACH, mainProcessPid, NULL, NULL);
        if (status != 0) {
            //attch失败
            ALOGE(">>>>>>>>> error: attach target process %d ", status);
            return -errno;
        }
        first->wait_sigcont = true;
        //开始执行死循环代码,因为处于一直监听状态,理论上该进程不会退出
        exit(event_loop());
    } else {
        //init seccomp by main process
        //the seccomp filtering rule is intended only for the current process
        enable_syscall_filtering(first);
    }
    return 0;
}
```

也就是当发现 svc 指令的一个回调。也就是（SIGTRAP | 0x80），这个时候开始执行**调试线程**逻辑。**被调试线程**进入等待状态。

 

通过调用 ptrace 提供的 api 进行 attch，**调试进程**是一个 while true 死循环。这样就可以一直监听**被调试线程**的状态，**调试线程**通过 linux waitpid 函数进行处理和回调，等待**被调试线程**进入指定回调。

```
while (true) {
        int tracee_status;
        Tracer *tracee;
        int signal;
        pid_t pid;
        free_terminated_tracees();
        //-1 all thread
        pid = waitpid(-1, &tracee_status, 0);
        if (pid < 0) {
            ALOGE(">>>>>>>>>> !!!!! error: waitpid() %d  %s", pid, strerror(errno))
            if (errno != ECHILD) {
                return EXIT_FAILURE;
            }
            break;
        }
        tracee = get_tracer(nullptr, pid, true);
        assert(tracee != nullptr);
 
        //handle action
        signal = handle_tracee_event(tracee, tracee_status);
        //restart
        (void) restart_tracee(tracee, signal);
    }
    ALOGE("<<<<<<<<<<<< listening was error ,main listener stop !!")
    return last_exit_status;
}
```

主要包含如下几种状态。包括正常退出，异常退出，结束，或者进入系统调用等。代码如下。

```
if (WIFEXITED(tracee_status)) {
        //WEXITSTATUS取得子进程exit（）返回的结束代码
        last_exit_status = WEXITSTATUS(tracee_status);
        //ALOGI("normal exit -> [%d] exit with status: %d  ", tracee->pid, tracee_status);
        //被跟踪者进程 正常执行结束,释放当前（被跟踪者）
        terminate_tracee(tracee);
  } else if (WIFSIGNALED(tracee_status)) {
        int signNum = WTERMSIG(tracee_status);
        //被跟踪进程因为信号退出
        ALOGE("[%d]  process exit with signal: 终止信号 = %d  异常原因 = %s ",
              tracee->pid,
              signNum,
              strsignal(signNum)
        )
        terminate_tracee(tracee);
  } else if (WIFSTOPPED(tracee_status)) {
        signal = (tracee_status & 0xfff00) >> 8;
        switch (signal) {
            //svc
            case SIGTRAP | 0x80:
            //被调试线程调用svc，开始处理参数和返回值
            ...
```

当参数或者返回值处理完毕以后，通过给**调试线程**，调用 ptrace 设置**被调试线程**的启动 PTRACE_SYSCALL 事件

 

（当被调试进程执行了某些 SIGTRAP 事件，程序就会进入暂停，这个时候**调试线程**开始处理对应的逻辑

 

常用的恢复暂停事件有两个，PTRACE_SYSCALL 和 PTRACE_CONT ，可以理解成一个是调试的单步执行，一个是继续执行

 

PTRACE_SYSCALL 方式重新启动**被调试线程**以后，下次遇到 SVC 的 before 和 after 还会继续暂停。

 

）

 

也就是当 svc **执行之前（before）**和**执行之后 (after)** 被调试线程都会暂停。

 

先把每个版本不同的寄存器进行匹配。用来区分 LR,SP,PC 等常用寄存器。

```
#elif defined(ARCH_ARM_EABI)
static off_t reg_offset[] = {
        [SYSARG_NUM]    = USER_REGS_OFFSET(uregs[7]),
        [SYSARG_1]      = USER_REGS_OFFSET(uregs[0]),
        [SYSARG_2]      = USER_REGS_OFFSET(uregs[1]),
        [SYSARG_3]      = USER_REGS_OFFSET(uregs[2]),
        [SYSARG_4]      = USER_REGS_OFFSET(uregs[3]),
        [SYSARG_5]      = USER_REGS_OFFSET(uregs[4]),
        [SYSARG_6]      = USER_REGS_OFFSET(uregs[5]),
        [SYSARG_RESULT] = USER_REGS_OFFSET(uregs[0]),
        [FRAME_POINTER] = USER_REGS_OFFSET(uregs[12]),
        [STACK_POINTER] = USER_REGS_OFFSET(uregs[13]),
        [LINK_REGISTER] = USER_REGS_OFFSET(uregs[14]),
        [INSTR_POINTER] = USER_REGS_OFFSET(uregs[15]),
        [USERARG_1]     = USER_REGS_OFFSET(uregs[0]),
 
};
#elif defined(ARCH_ARM64)
#undef  USER_REGS_OFFSET
#define USER_REGS_OFFSET(reg_name) offsetof(struct user_regs_struct, reg_name)
 
static off_t reg_offset[] = {
[SYSARG_NUM]    = USER_REGS_OFFSET(regs[8]),
[SYSARG_1]      = USER_REGS_OFFSET(regs[0]),
[SYSARG_2]      = USER_REGS_OFFSET(regs[1]),
[SYSARG_3]      = USER_REGS_OFFSET(regs[2]),
[SYSARG_4]      = USER_REGS_OFFSET(regs[3]),
[SYSARG_5]      = USER_REGS_OFFSET(regs[4]),
[SYSARG_6]      = USER_REGS_OFFSET(regs[5]),
[SYSARG_RESULT] = USER_REGS_OFFSET(regs[0]),
//https://zhuanlan.zhihu.com/p/42486116
//http://blog.chinaunix.net/uid-25564582-id-5852920.html
[FRAME_POINTER]     = USER_REGS_OFFSET(regs[29]),
//64位30是LR寄存器
[LINK_REGISTER]     = USER_REGS_OFFSET(regs[30]),
[STACK_POINTER] = USER_REGS_OFFSET(sp),
[INSTR_POINTER] = USER_REGS_OFFSET(pc),
[USERARG_1]     = USER_REGS_OFFSET(regs[0]),
};
```

这个时候我们可以直接去获取寄存器内容，判断路径，是否是我们需要修改的文件路径

```
/**
 * Return the *cached* value of the given @Tracers' @reg.
 * 返回给定@Tracers@reg的缓存值
 */
word_t peek_reg(const Tracer *Tracer, RegVersion version, Reg reg) {
    word_t result;
 
    assert(version < NB_REG_VERSION);
 
    result = REG(Tracer, version, reg);
 
    /* Use only the 32 least significant bits (LSB) when running
     * 32-bit processes on a 64-bit kernel.  */
    if (is_32on64_mode(Tracer))
        result &= 0xFFFFFFFF;
 
    return result;
}
 
/**
 * Set the *cached* value of the given @Tracers' @reg.
 *
 * 修改寄存器的内容方法,value标识的是指针
 */
void poke_reg(Tracer *Tracer, Reg reg, word_t value) {
    //设置之前先判断是否相等
    if (peek_reg(Tracer, CURRENT, reg) == value)
        //相等直接返回
        return;
 
    REG(Tracer, CURRENT, reg) = value;
    //标识他已经被修改
    Tracer->_regs_were_changed = true;
}
```

将修改完毕的寄存器内容保存到数组里面，最后通过 ptrace PTRACE_SETREGSET action 进行寄存器的 set

```
regs.iov_base = ¤t_sysnum;
regs.iov_len = sizeof(current_sysnum);
 
status = ptrace(PTRACE_SETREGSET, Tracer->pid, NT_ARM_SYSTEM_CALL, ®s);
if (status < 0) {
    //note(Tracer, WARNING, SYSTEM, "can't set the syscall number");
    return status;
}
```

修改被调试进程的寄存器内容，已达到修改 svc 参数和返回结果的目的。

 

这个思路确实是可以实现 svc 的参数和返回值的修改。但是**存在问题**。

 

**效率太低，调试线程和被调试线程本身是两条线程，主要通过线程间交互进行传递消息，而且被 attch 的进行会进行**

 

**大量的暂停，甚至本身的 libc 去调用 svc 的时候也会进行暂停。导致程序卡顿。**

 

当时为了解决这个问题也花了很久查了很多资料。

### Seccomp+ptrace:

为了解决这个效率低的问题，看了很多开源框架，比如 Strace 也在使用 ptrace 进行 svc 的跟踪。

> 一个打印 Syscall 调用方法的插件，可以很清楚的打印全部的系统调用，比如文件相关类型函数
> 
> 网络相关类型的函数，等... 他同样也可以用在安卓上面， 具体使用方式，国内资料比较多，可以去看一下。

 

Strace 是怎么解决的？用的就是 Seccomp+ptrace 去做拦截，我们只需要关注我们需要进行拦截的函数即可

 

比如常见的 IO 函数，access,openat,open,fstart 等即可。而不需要关注别的系统调用。

 

seccomp 初始化完毕以后，我们只需要在 ptrace PTRACE_SETOPTIONS 的时候加上 PTRACE_O_TRACESECCOMP 参数即可。

```
const unsigned long default_ptrace_options = (
        PTRACE_O_TRACESYSGOOD|
        PTRACE_O_TRACEFORK |
        PTRACE_O_TRACEVFORK |
        PTRACE_O_TRACEVFORKDONE |
        PTRACE_O_TRACEEXEC |
        PTRACE_O_TRACECLONE |
        PTRACE_O_TRACEEXIT);
 
//尝试开启ptrace+seccomp
status = ptrace(PTRACE_SETOPTIONS, tracee->pid, NULL,default_ptrace_options | PTRACE_O_TRACESECCOMP);
```

这样一来当目标 App 调用了被我们拦截的系统调用的时候就会走如下 case。

```
case SIGTRAP | PTRACE_EVENT_SECCOMP << 8:
```

我们直接在这个执行上面的流程设置参数，也是没问题的。但是这个时候又来一个问题

 

**Seccomp 只能处理 svc 的 before ，也就是当 svc 执行之前进入到这个 case，不能处理 after。**

 

因为修改 svc 的返回结果必须在 after 里面处理，有人可能会问了为什么要处理返回结果呢? 文件重定向只需要处理参数就行了

#### [](#参数完全可以在before里面进行处理。为啥还要处理after呢？)参数完全可以在 before 里面进行处理。为啥还要处理 after 呢？

答：做指纹 mock 时候需要在 after 里面处理，比如 socket 常见得通讯函数，recv,recvfrom,recvmsg

 

他们都是在原始函数调用完毕以后把数据参数放到一个数组里面，如果在 before 处理这个数组肯定是 NULL

 

只有在函数执行完毕以后才会将数组的内容进行赋值。比如我之前讲的通过 netlinker 去获取设备指纹。

 

https://bbs.pediy.com/thread-271698.htm

 

这也是大厂的贯通套路，这种指纹想要去 mock 很难，特别是用内联 svc 的方式去获取。但是用了 ptrace 想去修改的话就很简单了。代码如下，先通过 peek_reg 寄存器把数据读到手，然后在把数据处理完毕以后在 poke_reg 回去。

```
void NetlinkMacHandler::netlinkHandler_recv(Tracer *tracee) {
    ssize_t bytes_read = TEMP_FAILURE_RETRY(peek_reg(tracee, CURRENT, SYSARG_RESULT));
    if (bytes_read > 0) {
        word_t buff = peek_reg(tracee, CURRENT, SYSARG_2);
        //buff长度
        auto size = (size_t) peek_reg(tracee, CURRENT, SYSARG_3);
        char tempBuff[size];
        int readStr_ret = read_data(tracee, tempBuff, buff, size);
        if (readStr_ret != 0) {
            LOGE("svc netlink handler read_string error  %s", strerror(errno))
            return;
        }
        auto *hdr = reinterpret_cast(tempBuff);
        //netlink数据包结构体
        NetlinkMacHandler::handler_mac_callback_svc(tracee,hdr, bytes_read);
        //将数据写入覆盖掉原来的数据
        write_data(tracee, buff, tempBuff, size);
    }
} 
```

为了解决 ptrace+seccomp 不能处理 after 的问题想了好久 ，卡了我几个月之久。后来通过问 VA 作者发现一个很不错开源的项目就是 proot

 

项目地址 ->https://github.com/proot-me/proot

 

proot 的解决方案也很简单，只需要一行即可。果然天才需要的是灵感。

```
poke_reg(tracee, STACK_POINTER, peek_reg(tracee, ORIGINAL, STACK_POINTER));
```

**修改 SP 寄存器，让这个方法二次进入，一次修改参数，一次修改返回结果即可。**

 

简单介绍一下 proot 这个项目。他就完全符合我们的需求，处理逻辑也类似。

 

在 Linux 里面有一个 chroot 函数，这个函数可以修改 root 用户根目录的位置，但是这个函数需要 root 权限才可以用，

 

比如我想把 / data/zhenxi / 路径变成 Linux 的根目录。而不是最原始的 /，

 

就可以使用这个 proot 编译好以后，直接启动就可以在免 root 的环境下进行限制和使用。他的原理也是使用 Ptace+seccomp 进行限制。

 

对执行的文件路径进行替换和处理。**我上面发的代码也都是来自 proot。**

 

当然我说的这些也是 proot 的一小部分功能，更重要的功能是利用 ptrace+seccomp 实现沙盒文件限制的逻辑。

 

搞定以后就需要进行注入了，如何把拦截和修改的功能注入到目标 App 里面，因为 Linux 特性 ptrace 只针对当前进程才可以免 root 进行 attch。

[](#注入方式：)注入方式：
---------------

### Xposed 注入 So:

优点非入侵式，不需要修改 apk 签名，只在 onload 里面进行 attch 当前线程，可实现全部进程的 svc 修改和 mock

 

包括文件监听，过检测等。

### [](#二次打包注入：)二次打包注入：

入侵式，优点就是可以在免 root 环境下进行 attch，直接把 So 打进去，然后加载即可，缺点就是修改签名需要绕过，不过绕过更简单了。

 

直接通过 SVC 的 IO 重定向把 原始的 apk 放到任意私有路径，当对方读取 / data/app / 包名 / base.apk 的时候直接把参数替换成原始的 apk 路径即可。**这种方式对抗企业壳的重打包检测依然有效。**

[](#使用场景：)使用场景：
---------------

### 文件读取监听 & 合规检测：

很多 app 会去读取大量别的 app 私有目录，比如去遍历 / data/data/xxx / 下的文件路径，获取读取 SD 卡下的其他文件。这些都是不合规或者

 

存在安全隐患问题，用 SVC 文件文件监听的方式把对方读取的路径打印出来，可以快速的去分析对方 app 是否合规，是否存在安全隐患。

 

方便进行快速分析和定位。

 

打印效果如下, 读取哪些文件也是一清二楚：

```
2022-06-04 15:31:06.910 13927-13960/  I/Zhenxi: io sandbox  /vendor/lib64/hw/ -> /vendor/lib64/hw/
2022-06-04 15:31:06.910 13951-13951/? I/Zhenxi: io sandbox  /vendor/lib64/hw/ -> /vendor/lib64/hw/
2022-06-04 15:31:06.910 13927-13960/  I/Zhenxi: io sandbox  /vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl-qti-display.so -> /vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl-qti-display.so
2022-06-04 15:31:06.910 13927-13960/  I/Zhenxi: io sandbox  /vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl-qti-display.so -> /vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl-qti-display.so
2022-06-04 15:31:06.911 13951-13951/? I/Zhenxi: io sandbox  /vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl-qti-display.so -> /vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl-qti-display.so
2022-06-04 15:31:06.911 13951-13951/? I/Zhenxi: io sandbox  /proc/self/fd/109 -> /proc/self/fd/109
2022-06-04 15:31:06.911 13927-13960/  I/Zhenxi: io sandbox  /proc/self/maps -> /proc/self/maps
2022-06-04 15:31:06.911 13951-13951/? I/Zhenxi: io sandbox  /data/user/0/ /app_virtual_devices/START_UP_0/data/nativeCache/dev_maps_13951_13951 -> /data/user/0/ /app_virtual_devices/START_UP_0/data/nativeCache/dev_maps_13951_13951
2022-06-04 15:31:06.916 13927-13960/  I/Zhenxi: io sandbox  libadreno_utils.so -> libadreno_utils.so
2022-06-04 15:31:06.916 13927-13960/  I/Zhenxi: io sandbox  /proc/self/maps -> /proc/self/maps
2022-06-04 15:31:06.916 13951-13951/? I/Zhenxi: io sandbox  /data/user/0/ /app_virtual_devices/START_UP_0/data/nativeCache/dev_maps_13951_13951 -> /data/user/0/ /app_virtual_devices/START_UP_0/data/nativeCache/dev_maps_13951_13951
2022-06-04 15:31:06.930 13927-13960/  I/Zhenxi: io sandbox  /data/user_de/0/ /code_cache/com.android.opengl.shaders_cache -> /data/user/0/ /app_virtual_devices/START_UP_0/user_de/code_cache/com.android.opengl.shaders_cache
2022-06-04 15:31:06.930 13951-13951/? I/Zhenxi: io sandbox  /data/user/0/ /app_virtual_devices/START_UP_0/user_de/code_cache/com.android.opengl.shaders_cache -> /data/user/0/ /app_virtual_devices/START_UP_0/user_de/code_cache/com.android.opengl.shaders_cache
2022-06-04 15:31:06.961 13927-13960/  I/Zhenxi: io sandbox  libboost.so -> libboost.so
2022-06-04 15:31:06.962 13927-13960/  I/Zhenxi: io sandbox  /system/lib64/libboost.so -> /system/lib64/libboost.so
2022-06-04 15:31:06.962 13951-13951/? I/Zhenxi: io sandbox  /system/lib64/libboost.so -> /system/lib64/libboost.so
2022-06-04 15:31:06.962 13951-13951/? I/Zhenxi: io sandbox  /proc/self/fd/110 -> /proc/self/fd/110
2022-06-04 15:31:06.962 13927-13960/  I/Zhenxi: io sandbox  /proc/self/maps -> /proc/self/maps
2022-06-04 15:31:06.962 13951-13951/? I/Zhenxi: io sandbox  /data/user/0/ /app_virtual_devices/START_UP_0/data/nativeCache/dev_maps_13951_13951 -> /data/user/0/ /app_virtual_devices/START_UP_0/data/nativeCache/dev_maps_13951_13951
2022-06-04 15:31:06.967 13927-13960/  I/Zhenxi: io sandbox  /proc/13927/cmdline -> /proc/13927/cmdline
2022-06-04 15:31:06.967 13951-13951/? I/Zhenxi: io sandbox  /proc/13927/cmdline -> /proc/13927/cmdline
2022-06-04 15:31:06.967 13927-13960/  I/Zhenxi: io sandbox  /data/system/migt/migt -> /data/system/migt/migt
2022-06-04 15:31:06.967 13951-13951/? I/Zhenxi: io sandbox  /data/system/migt/migt -> /data/system/migt/migt
 
... ...
```

### [](#pass环境检测：)PASS 环境检测：

#### Root&magisk

因为一切文件读取最终底层读取都是 SVC 函数去读取，我们只需要写个 sandbox

 

把我们认为的问题关键目录直接全部都进行 PASS 即可，当目标 App 去读到这个路径以后我们将方法的路径设置成一个不存在的路径即可。

 

代码如下

```
else if (strstr(result, "magisk")) {
    //直接包含magisk的都给干掉
    result = NULL;
} else if (strstr(result, "edxposed")) {
    result = NULL;
} else if (strstr(result, "edxp")) {
    result = NULL;
}else if (strstr(result, "lsposed")) {
    result = NULL;
} else if (strstr(result, "libriru") || strstr(result, "/riru")) {
    result = NULL;
} else if (strstr(result, "sandhook")) {
    result = NULL;
} else if (endsWith(result, "/su")) {
    //su结尾的root文件都直接干掉
    result = NULL;
}else if (strstr(result, "zygisk")) {
    result = NULL;
}else if (strstr(result, "/data/adb/")) {
    //这个文件里面包含很多magisk相关的,比如模块的list /data/adb/modules/
    //https://github.com/LSPosed/NativeDetector/blob/master/app/src/main/jni/activity.cpp
    result = NULL;
}
```

当然这些还是不够，还有 / proc/mounts 里面也有一堆 magisk 文件特征。这个文件可以里面也有一堆特征。

 

可以在执行之前先生成一份，然后当发现读取到我们需要 IO 重定向的路径直接修改成我们生成的文件即可。

#### [](#debugcheck：)DebugCheck：

还有一些反调试文件，stat status wchan 这些也都是在 ptrace attch 之前进行 mock 一份，然后 IO 重定向到新生成的文件路径即可。

#### [](#mapscheck：)MapsCheck：

maps io 重定向的话，不能提前生成，必须在目标 app 读取之前进行创建，因为 maps 是不断变化的。以防万一有人扫描 maps 去检测。

 

在 svc 的 before 里面**调试线程**去创建，然后修**改被调试线程**的读取路径。

#### AppSign:

很多大厂或者壳子检测签名无非几种，java 层检测和 native 检测，这些完全可以 Hook 在注入的时候顺便把 sandhook 等框架一起注入。

 

在配合 ptrace+seccomp，Java 层的话就是 Hook 获取签名的那几个方法，然后记得把内存的变量也需要通过反射的方式去 set 上去。

 

不能只 Hook 方法。这块需要过掉 9.0 反射限制，可以参考 LSP 的绕过反射限制代码项目。

 

native 检测大部分都是 svc openat 去读取文件，然后把 apk 当成 zip 进行解压缩，解析，去计算 apk 的签名文件。判断是否正确

 

这种方式很简单绕过，只需要去在他读取 / data/app/xxx/base.apk 的时候，我们把他指向原始包即可绕过。

### 沙箱的打磨 & 实践：

有时候我们经常需要分析一个 So, 看看这个 So 里面读取了哪些文件，做了哪些事情，调用了哪些 Java 方法

 

我以前挺喜欢用 unidbg 的，但是发现痛点太多，就是需要补环境，有很多 So 会调用高版本 api，我记得我以前用的时候

 

只支持 23 和 26 版本的 SDK，这个时候如果去补环境，真的很累，特别是很多 So 会去扫描大量的系统文件。这些系统文件都是 unidbg 里面没有的，不如我们直接在安卓系统上直接运行这个 So。

 

我们可以先搞个 helloword 然后先启动我们 ptrace 进行 attch。

 

在配合 sandhook 和 jnitraceforcpp(https://github.com/w296488320/JnitraceForCpp)

> 这个 jnitraceforcpp 不是 frida 的 jnitrace，是我以前空闲的时候写的，代码没多少行，但是 Hook 了全部的 jniEnv 里面的方法，对方不管调用了什么我们这边都可以进行打印。hook 方法如下
> 
> ```
> HOOK_JNI(env, CallObjectMethodV)
> HOOK_JNI(env, CallBooleanMethodV)
> HOOK_JNI(env, CallByteMethodV)
> HOOK_JNI(env, CallCharMethodV)
> HOOK_JNI(env, CallShortMethodV)
> HOOK_JNI(env, CallIntMethodV)
> HOOK_JNI(env, CallLongMethodV)
> HOOK_JNI(env, CallFloatMethodV)
> HOOK_JNI(env, CallDoubleMethodV)
> HOOK_JNI(env, CallVoidMethodV)
>  
> HOOK_JNI(env, CallStaticObjectMethodV)
> HOOK_JNI(env, CallStaticBooleanMethodV)
> HOOK_JNI(env, CallStaticByteMethodV)
> HOOK_JNI(env, CallStaticCharMethodV)
> HOOK_JNI(env, CallStaticShortMethodV)
> HOOK_JNI(env, CallStaticIntMethodV)
> HOOK_JNI(env, CallStaticLongMethodV)
> HOOK_JNI(env, CallStaticFloatMethodV)
> HOOK_JNI(env, CallStaticDoubleMethodV)
> HOOK_JNI(env, CallStaticVoidMethodV)
>  
> HOOK_JNI(env, GetObjectField)
> HOOK_JNI(env, GetBooleanField)
> HOOK_JNI(env, GetByteField)
> HOOK_JNI(env, GetCharField)
> HOOK_JNI(env, GetShortField)
> HOOK_JNI(env, GetIntField)
> HOOK_JNI(env, GetLongField)
> HOOK_JNI(env, GetFloatField)
> HOOK_JNI(env, GetDoubleField)
>  
> HOOK_JNI(env, GetStaticObjectField)
> HOOK_JNI(env, GetStaticBooleanField)
> HOOK_JNI(env, GetStaticByteField)
> HOOK_JNI(env, GetStaticCharField)
> HOOK_JNI(env, GetStaticShortField)
> HOOK_JNI(env, GetStaticIntField)
> HOOK_JNI(env, GetStaticLongField)
> HOOK_JNI(env, GetStaticFloatField)
> HOOK_JNI(env, GetStaticDoubleField)
> //常用的字符串操作函数
> HOOK_JNI(env, NewStringUTF)
> HOOK_JNI(env, GetStringUTFChars)
> //HOOK_JNI(env, FindClass)
> HOOK_JNI(env, ToReflectedMethod)
> HOOK_JNI(env, FromReflectedMethod)
> HOOK_JNI(env, GetFieldID)
> HOOK_JNI(env, GetStaticFieldID)
> HOOK_JNI(env, NewObjectV)
> ```

 

还有一些常用的字符串操作的函数

```
    HOOK_SYMBOL_DOBBY(handle, strstr);
    HOOK_SYMBOL_DOBBY(handle, strcmp);
    HOOK_SYMBOL_DOBBY(handle, strcpy);
    HOOK_SYMBOL_DOBBY(handle, strdup);
    HOOK_SYMBOL_DOBBY(handle, strxfrm);
//    HOOK_SYMBOL_DOBBY(handle, memcpy);
 
//    HOOK_SYMBOL_DOBBY(handle, sprintf);
//    HOOK_SYMBOL_DOBBY(handle, printf);
//    HOOK_SYMBOL_DOBBY(handle, snprintf);
//    HOOK_SYMBOL_DOBBY(handle, vsnprintf);
```

把这方法全部进行 hook 以后，再把对方的 So 加载进来，对方调用了什么方法，做了哪些事情一目了然。

 

而且最重要的是不需要去补环境。分析效率很高。需要处理什么直接 Hook 即可。

### [](#指纹的对抗：)指纹的对抗：

很多大厂会去获取设备指纹，Java 层那些方法不多说，直接 hook 就行。

 

核心的都是在 native 层去处理，system_property_get&read，bootid，UUID，反射内存 android id，netlinker 获取网卡

 

还有一些就是内联 svc 调用读文件函数也是可以获取到网卡信息，比如

 

/sys/class/net/wlan0/address & /sys/devices/virtual/net/wlan0/address

 

build.prop popen 读取一些设备 如 /sys/devices/soc0/serial_number 类似这种

 

还有 execve 去获取一些设备信息，这些通过 svc 的 IO 重定向很容易就可以实现 mock 和 pass。

 

当读取的时候我们生产一份新得，指向到新生成的文件即可。

### [](#无痕hook的实现思路：)无痕 Hook 的实现思路：

现在很多 native hook 思路都是 inlinehook ，Got 表 , 异常 hook（异常信号劫持 hook）。

 

这些思路都是很好的思路，各有各的好处，但是都是有特征，比如 inlinehook crc 检测很容易就检测出来，并且

 

有很多大厂会用 shellcode 进行绕过，我们能去修改这段指令跳转到这个某个函数，他当然也可以修改回来。

 

他只需要把某个方法的指令换成原始的指令，这样就可以防止被 inlinehook Hook（他需要获取原始指令，可以解析本地的 So 文件，解析 text 段，得到最真实的指令信息，保存，然后在函数执行之前在 set 回内存, 都是很好的办法，还有的干脆直接服务端配置一个服务，直接服务端拉取某个函数的正确指令，都是很好的思路） 当然对抗这也不是没办法，我只需要在 hook 完毕以后再把内存设置成可读，不可写

 

然后 Hook mprotect ，不让他调用 mprotect 这样就可以被 shellcode 绕过。

 

说的有点多，重点说一下无痕 Hook 的实现思路，ptrace 有一个很重要的功能就是下断点，我们只需要在指定内存下断点，

 

当方法执行到这里以后，直接通过 ptrace 修改 PC 寄存器。跳转到到指定函数即可，把参数也带过去，

 

因为是**执行阶段才会修改**，而且是直接修改的寄存器，不存在修改指令。所以无需担心指令的 crc 检测不过问题。

 

测试在 64 位还是很稳定的，32 位的话总有问题，很多 32 位程序走的是 64 位的 sysnum 。一直报 bad syscall num 。原因未知。一直在采坑，还没填上去。不过还好大部分 app 都是 64 位的。

[](#结束语：)结束语：
-------------

文章还是挺长的，很感谢能看到最后，上述的 svc 拦截核心代码大部分都能在

 

proot 项目里面找到，需要自己移植到 android 上面 。另外如果对指纹或者 App 检测比较感兴趣的技术，也可以私信我加个好友。

 

算做个技术交流吧 。

[【看雪培训】《Adroid 高级研修班》2022 年春季班招生中！](https://bbs.pediy.com/thread-271992.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#漏洞相关](forum-161-1-123.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm) [#其他](forum-161-1-129.htm)