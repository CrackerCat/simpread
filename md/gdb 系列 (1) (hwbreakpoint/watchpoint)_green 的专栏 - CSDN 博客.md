> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/green369258/article/details/55510650)

1. 软硬件环境
========

android 7.0(n)  
QCOM 骁龙 820

2. 背景
=====

我最初是因为要做一件类似这样的事情的时候才研究这个的  
art debug 过程中我们发现 经常有 SIGSEGV 的问题, 而且是死在 java 代码里 (art 已经使用 dexoat 把 java code compile 成了机器码) 几经分析发现是在对象的 method 里执行的时候 this 指针被改了, 导致了取对象的一些成员的时候出现了非法地址, 为了抓到野指针的写 this 指针的第一现场, 我们选择了 hwbreakpoint\watchpoint  
希望在进入某个 method 的时候启动这个 watchpoint, 退出这个 method 的时候关闭这个 watchpoint  
类似这个哥们做的事情

> [http://blog.csdn.net/_xiao/article/details/40619797](http://blog.csdn.net/_xiao/article/details/40619797)

因为看到了这个

```
1. 在ARM的架构文档中（ARM官网可下载），Cortex A8, A9, A15才支持硬件断点（通过协处理器CP14操作调试寄存器DBGWCR和DBGWVR来下数据断点(watchpoint)，Processor和JTAG Debuger均可以操作它们）（现在跑安卓的机器一般都是A9以上的架构，所以基本都支持硬件断点）。
2. Linux内核要在2.6.37以后的版本才支持对ARM添加硬件断点。
3. GDB要在7.3以后的版本才支持对ARM添加硬件断点（最新版本是7.8.1，2014年10月）。
```

经过确认 820 是肯定支持硬件断点的, 加大了我们做这个事情的信心  
CPU 为 Qualcomm Technologies, Inc MSM8996  
kernel 3.18.31  
GNU gdbserver (GDB) 7.11

3.hwbreakpoint\watchpoint 简介
============================

gdb 介绍

> [https://sourceware.org/gdb/onlinedocs/gdb/Set-Watchpoints.html](https://sourceware.org/gdb/onlinedocs/gdb/Set-Watchpoints.html)  
> You can use a watchpoint to stop execution whenever the value of an expression changes, without having to predict a particular place where this may happen. (This is sometimes called a data breakpoint.) The expression may be as simple as the value of a single variable, or as complex as many variables combined by operators.  
> 你可以使用一个监测点停止执行时，表达式的值更改，而不必预测一个特定的地方，这可能会发生。（有时也称为数据断点）表达式可能与单个变量的值一样简单，或与运算符组合的多个变量一样复杂。
> 
> On some systems, such as most PowerPC or x86-based targets, gdb includes support for hardware watchpoints, which do not slow down the running of your program.  
> 在某些系统中，如大多数的基于 PowerPC 或 x86 的目标机，GDB 包含硬件观察点的支持，这并不会减慢你的程序的运行。

4. 源码级确认
========

OK 强迫症患者, 为了确定我们真的支持 Read the fucking source code  
kernel

```
这里可以确定如何读取有几个hw breakpoint 有几个hw watchpoint
 在我们平台上这两个值分别是
 get_num_wrps() = 4
 get_num_brps() = 8 
arch/arm64/kernel/hw_breakpoint.c 
/* Determine number of BRP registers available. */
static int get_num_brps(void)
{
    return ((read_cpuid(ID_AA64DFR0_EL1) >> 12) & 0xf) + 1;
}

/* Determine number of WRP registers available. */
static int get_num_wrps(void)
{
    return ((read_cpuid(ID_AA64DFR0_EL1) >> 20) & 0xf) + 1;
}

这里可以确定有debug arch version
在我们平台上这个值是 6(AARCH64_DEBUG_ARCH_V8)
arch/arm64/kernel/debug-monitors.c
/* Determine debug architecture. */
u8 debug_monitors_arch(void)
{
    return read_cpuid(ID_AA64DFR0_EL1) & 0xf;
}
arm 官方描述
> http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0488d/CIHEIGGB.html
Bits    Name    Function
[63:32] -   Reserved, res0
[31:28] CTX_CMPs    Returns 0x1 to indicate support for two context-aware breakpoints
[27:24] -   Reserved, res0
[23:20] WRPs    Returns 0x3 to indicate support for four watchpoints
[19:16] -   Reserved, res0
[15:12] BRPs    Returns 0x5 to indicate support for six breakpoints
[11:8]  PMUVer  
Returns 0x1 to indicate that the Performance Monitors (PMUv3) System registers are implemented
[7:4]   TraceVer    Returns 0x0 to indicate that the Trace System registers are not implemented
[3:0]   DebugVer    Returns 0x6 to indicate that the v8-A Debug architecture is implemented
```

gdb

```
 通过 aarch64_linux_get_debug_reg_capacity来获取
 hw watchpoint(aarch64_num_wp_regs) 
 hw breakpoint(aarch64_num_bp_regs)
 在我们平台上这两个值分别是
 aarch64_num_wp_regs = 4
 aarch64_num_bp_regs = 8 
 最新的gdb 7.12中支持的debug arch更全面，我们目前这个值是0x6
/* Macro for the expected version of the ARMv8-A debug architecture.  */
#define AARCH64_DEBUG_ARCH_V8 0x6
#define AARCH64_DEBUG_ARCH_V8_1 0x7
#define AARCH64_DEBUG_ARCH_V8_2 0x8

/* Get the hardware debug register capacity information from the
   process represented by TID.  */

void
aarch64_linux_get_debug_reg_capacity (int tid)
{
  struct iovec iov;
  struct user_hwdebug_state dreg_state;

  iov.iov_base = &dreg_state;
  iov.iov_len = sizeof (dreg_state);

  /* Get hardware watchpoint register info.  */
  if (ptrace (PTRACE_GETREGSET, tid, NT_ARM_HW_WATCH, &iov) == 0
      && AARCH64_DEBUG_ARCH (dreg_state.dbg_info) == AARCH64_DEBUG_ARCH_V8)
    {
      aarch64_num_wp_regs = AARCH64_DEBUG_NUM_SLOTS (dreg_state.dbg_info);
      if (aarch64_num_wp_regs > AARCH64_HWP_MAX_NUM)
    {
      warning (_("Unexpected number of hardware watchpoint registers"
             " reported by ptrace, got %d, expected %d."),
           aarch64_num_wp_regs, AARCH64_HWP_MAX_NUM);
      aarch64_num_wp_regs = AARCH64_HWP_MAX_NUM;
    }
    }
  else
    {
      warning (_("Unable to determine the number of hardware watchpoints"
         " available."));
      aarch64_num_wp_regs = 0;
    }

  /* Get hardware breakpoint register info.  */
  if (ptrace (PTRACE_GETREGSET, tid, NT_ARM_HW_BREAK, &iov) == 0
      && AARCH64_DEBUG_ARCH (dreg_state.dbg_info) == AARCH64_DEBUG_ARCH_V8)
    {
      aarch64_num_bp_regs = AARCH64_DEBUG_NUM_SLOTS (dreg_state.dbg_info);
      if (aarch64_num_bp_regs > AARCH64_HBP_MAX_NUM)
    {
      warning (_("Unexpected number of hardware breakpoint registers"
             " reported by ptrace, got %d, expected %d."),
           aarch64_num_bp_regs, AARCH64_HBP_MAX_NUM);
      aarch64_num_bp_regs = AARCH64_HBP_MAX_NUM;
    }
    }
  else
    {
      warning (_("Unable to determine the number of hardware breakpoints"
         " available."));
      aarch64_num_bp_regs = 0;
    }
}
```

kernel

```
 要想使用 ptrace (PTRACE_GETREGSET, tid, NT_ARM_HW_WATCH, &iov)
 request 是PTRACE_GETREGSET 需要开启如下选项CONFIG_HAVE_ARCH_TRACEHOOK 是因为
 kernel/ptrace.c的function 
 int ptrace_request(struct task_struct *child, long request,
           unsigned long addr, unsigned long data)
 中有如下代码片段
 #ifdef CONFIG_HAVE_ARCH_TRACEHOOK
    case PTRACE_GETREGSET:
    case PTRACE_SETREGSET: {
        struct iovec kiov;
        struct iovec __user *uiov = datavp;

        if (!access_ok(VERIFY_WRITE, uiov, sizeof(*uiov)))
            return -EFAULT;

        if (__get_user(kiov.iov_base, &uiov->iov_base) ||
            __get_user(kiov.iov_len, &uiov->iov_len))
            return -EFAULT;

        ret = ptrace_regset(child, request, addr, &kiov);
        if (!ret)
            ret = __put_user(kiov.iov_len, &uiov->iov_len);
        break;
    }
#endif
并且addr 是NT_ARM_HW_WATCH或者NT_ARM_HW_BREAK的时候
 需要开启如下选项CONFIG_HAVE_HW_BREAKPOINT是因为
 enum aarch64_regset {
    REGSET_GPR,
    REGSET_FPR,
    REGSET_TLS,
#ifdef CONFIG_HAVE_HW_BREAKPOINT
    REGSET_HW_BREAK,
    REGSET_HW_WATCH,
#endif
    REGSET_SYSTEM_CALL,
};
 如果没有CONFIG_HAVE_HW_BREAKPOINT会导致
 PTRACE_GETREGSET 失败 
 kernel/ptrace.c的function 
 static int ptrace_regset(struct task_struct *task, int req, unsigned int type,
             struct iovec *kiov)
 中返回-EINVAL的错误,因为没有 REGSET_HW_BREAK REGSET_HW_WATCH 这两个的支持
 find_regset 找不到
{
    const struct user_regset_view *view = task_user_regset_view(task);
    const struct user_regset *regset = find_regset(view, type);
    int regset_no;

    if (!regset || (kiov->iov_len % regset->size) != 0)
        return -EINVAL;

    regset_no = regset - view->regsets;
    kiov->iov_len = min(kiov->iov_len,
                (__kernel_size_t) (regset->n * regset->size));

    if (req == PTRACE_GETREGSET)
        return copy_regset_to_user(task, view, regset_no, 0,
                       kiov->iov_len, kiov->iov_base);
    else
        return copy_regset_from_user(task, view, regset_no, 0,
                         kiov->iov_len, kiov->iov_base);
}
 这里 说明一下
 static const struct user_regset aarch64_regsets[] = {
    #ifdef CONFIG_HAVE_HW_BREAKPOINT
        [REGSET_HW_BREAK] = {
            .core_note_type = NT_ARM_HW_BREAK,
            .n = sizeof(struct user_hwdebug_state) / sizeof(u32),
            .size = sizeof(u32),
            .align = sizeof(u32),
            .get = hw_break_get,
            .set = hw_break_set,
        },
        [REGSET_HW_WATCH] = {
            .core_note_type = NT_ARM_HW_WATCH,
            .n = sizeof(struct user_hwdebug_state) / sizeof(u32),
            .size = sizeof(u32),
            .align = sizeof(u32),
            .get = hw_break_get,
            .set = hw_break_set,
        },
    #endif
};
 里面的.core_note_type就是find_regset中寻找的时候匹配的type id 
 下面的两个值是定义在 ptrace.h中的标准值
 #define PTRACE_GETREGSET   0x4204
 #define PTRACE_SETREGSET   0x4205
 下面的两个值是定义在 /usr/include/elf.h中的标准值
 #define NT_ARM_HW_BREAK    0x402       /* ARM hardware breakpoint registers */
 #define NT_ARM_HW_WATCH    0x403       /* ARM hardware watchpoint registers */
gdb 和kernel是统一的
```

虽然我们的平台是 arm64 But 上面分析结果我们确定 QCOM 骁龙 820+Android 7.0 一定是支持的, 并且我们通过分析知道 kernel, 需要开启那些选项, 而且出错后我们可以 debug 了  
好, 打消只有 x86 支持硬件断点的疑虑, arm 一样可以，我们继续

5. enable it
============

kernel config
-------------

config 文件加入如下配置

```
CONFIG_HAVE_HW_BREAKPOINT=y
CONFIG_HAVE_ARCH_TRACEHOOK=y
```

好, 去哪里找配置文件当然是在这里

```
 out/target/product/$(product)/obj/KERNEL_OBJ/.config
```

那么去哪里改呢?

```
cd device
cd $(product)
ack-grep KERNEL_DEFCONFIG
```

然后你就可以看到这个的赋值文件了, 直接改它之后

```
 make bootimage
 adb reboot bootloader
 fastboot flash boot boot.img
 fastboot reboot
```

好熟练的操作了一遍, 之后信心满满的开启了 gdbserver

```
gdbserver tcp:1234 /system/xbin/gdb-sample
```

terminal 很痛快的做了下面的输出

```
Process /system/xbin/gdb-sample created; pid = 5167
Unable to determine the number of hardware watchpoints available.
Unable to determine the number of hardware breakpoints available.
Listening on port 1234
```

很明显没有检测到 hardware watchpoints && hardware breakpoints  
继续讲分析过程  
一开始以为编译选项开启后, 肯定没问题

```
CONFIG_HAVE_HW_BREAKPOINT=y
CONFIG_HAVE_ARCH_TRACEHOOK=y
```

不怀疑, 不猜测, 傻呵呵的的就进入了动态分析过程, kernel 里面开始加 printk, 感觉好流弊

```
 分析结果为调用
 ptrace (PTRACE_GETREGSET, tid, NT_ARM_HW_WATCH\NT_ARM_HW_BREAK, &iov)
 返回-1 errno错误码为-EINVAL
 跟踪代码发现
 const struct user_regset_view *view = task_user_regset_view(task);
 const struct user_regset *regset = find_regset(view, type);
 找到regset 从而导致返回了-EINVAL;
 根据上面的源码分析按肯定是
 CONFIG_HAVE_HW_BREAKPOINT
 开启没有成功
 cat out/target/product/$(product)/obj/KERNEL_OBJ/.config | grep CONFIG_HAVE_HW_BREAKPOINT
 cat out/target/product/$(product)/obj/KERNEL_OBJ/.config | grep CONFIG_HAVE_ARCH_TRACEHOOK
 惊喜的发现,一个也没有,可是我config文件里加了呀
 find out/target/product/$(product)/obj/KERNEL_OBJ/ -name hw_breakpoint.o
 我的make kernelconfig一直起不来,于是用了下面的招数
 cd kernel
 cp ../out/target/product/gemini/obj/KERNEL_OBJ/.config .
 make menuconfig
 搜索CONFIG_HAVE_HW_BREAKPOINT
 发现
 Symbol: HAVE_HW_BREAKPOINT [=y]
 Type  : boolean 
      Defined at arch/Kconfig:241 
      Depends on: PERF_EVENTS [=y]
      Selected by: X86 [=y]
 理解的大致的意思是只有X86平台才会被选中,好吧google了一通,换了一招
 将
 config HAVE_HW_BREAKPOINT
    bool
    depends on PERF_EVENTS
 加了一句
 config HAVE_HW_BREAKPOINT
    bool
    depends on PERF_EVENTS
    default y
 make bootimage 
 cat out/target/product/$(product)/obj/KERNEL_OBJ/.config | grep CONFIG_HAVE_HW_BREAKPOINT
 cat out/target/product/$(product)/obj/KERNEL_OBJ/.config | grep CONFIG_HAVE_ARCH_TRACEHOOK
 find out/target/product/$(product)/obj/KERNEL_OBJ/ -name hw_breakpoint.o
 都能找到了
 上面的解决方法比较暴力,欢迎各位大神提出新思路
 重新编译后刷机 重启 重新启动gdbserver
 Process /system/xbin/gdb-sample created; pid = 5116
 Listening on port 1234
 输出了我们期望的信息,找到hardware watchpoints && hardware breakpoints了
```

6 为什么用 gdb
==========

其实硬件断点本身就是一个 debug register  
我参考了 kernel 操作 arm64 的时候如何设置的 debug register  
其实它真正操作寄存器在如下两个函数里

```
arch/arm64/kernel/hw_breakpoint.c
static u64 read_wb_reg(int reg, int n)
{
    u64 val = 0;
    //add by us
    pr_warning("read_wb_reg reg:%d, n:%d", reg, n);

    switch (reg + n) {
    GEN_READ_WB_REG_CASES(AARCH64_DBG_REG_BVR, AARCH64_DBG_REG_NAME_BVR, val);
    GEN_READ_WB_REG_CASES(AARCH64_DBG_REG_BCR, AARCH64_DBG_REG_NAME_BCR, val);
    GEN_READ_WB_REG_CASES(AARCH64_DBG_REG_WVR, AARCH64_DBG_REG_NAME_WVR, val);
    GEN_READ_WB_REG_CASES(AARCH64_DBG_REG_WCR, AARCH64_DBG_REG_NAME_WCR, val);
    default:
        pr_warning("attempt to read from unknown breakpoint register %d\n", n);
    }

    return val;
}

static void write_wb_reg(int reg, int n, u64 val)
{
    //add by us
    pr_warning("write_wb_reg reg:%d, n:%d, val:%llu", reg, n, val);
    switch (reg + n) {
    GEN_WRITE_WB_REG_CASES(AARCH64_DBG_REG_BVR, AARCH64_DBG_REG_NAME_BVR, val);
    GEN_WRITE_WB_REG_CASES(AARCH64_DBG_REG_BCR, AARCH64_DBG_REG_NAME_BCR, val);
    GEN_WRITE_WB_REG_CASES(AARCH64_DBG_REG_WVR, AARCH64_DBG_REG_NAME_WVR, val);
    GEN_WRITE_WB_REG_CASES(AARCH64_DBG_REG_WCR, AARCH64_DBG_REG_NAME_WCR, val);
    default:
        pr_warning("attempt to write to unknown breakpoint register %d\n", n);
    }
    isb();
}
```

于是我在想我们可以直接使用这段代码在用户态完成 debug register 设置，然后就不用麻烦 kernel 来操作了  
可实际上生成的这种指令  
`msr dbgbvr0_el1, x8`  
在用户模式 (Usr) 的时候会被识别为非法指令，无法执行  
还有一种解决方案  
ptrace 自己，结果是返回 No such process  
跟踪 kernel 代码发现在系统调用 ptrace 里

```
 kernel/ptrace.c
 SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
        unsigned long, data)
 调用
　ptrace_check_attach　返回了一个-ESRCH
　#define    ESRCH        3  /* No such process */
```

然后只能按照标准方式来

```
x86 example
 #include <sys/ptrace.h>  
 #include <sys/types.h>  
 #include <sys/wait.h>  
 #include <unistd.h>  
 #include <linux/user.h>   /* For constants ORIG_EAX etc */  
int main()  
{  
   pid_t child;  
    long orig_eax;  
    child = fork();  
    if(child == 0) {  
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);  
        execl("/bin/ls", "ls", NULL);  
    }
    else {  
        wait(NULL);  
        orig_eax = ptrace(PTRACE_PEEKUSER,   
                          child, 4 * ORIG_EAX,   
                          NULL);  
        printf("The child made a "  
               "system call %ld ", orig_eax);  
        ptrace(PTRACE_CONT, child, NULL, NULL);  
    }  
    return 0;  
}
```

　这样的话，gdb 就最合适了，想要的 ptrace 功能全部都有

7. usage
========

example code

```
#include <stdio.h>
#include <sys/uio.h>
#include <asm/ptrace.h>
#include <elf.h>
#include <sys/types.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>

int nGlobalVar = 0;

int tempFunction(int a, int b)
{
    printf("tempFunction is called, a = %d, b = %d \n", a, b);
    return (a + b);
}

int main()
{
    int n;
    n = 1;
    n++;
    n--;

    nGlobalVar += 100;
    nGlobalVar -= 12;

    printf("n = %d, nGlobalVar = %d \n", n, nGlobalVar);

    n = tempFunction(1, 2);
    printf("n = %d\n", n);

    return 0;
}
 可以在aosp下编译的代码可以通过如下命令下载
 git clone git@github.com:green130181/kernel-study.git
 然后　
 mmm kernel-study/gdb-sample
 adb push out/target/product/$(product)/system/xbin/gdb-sample /system/xbin/gdb-sample
 即可
```

　运行

```
 phone:gdbserver tcp:1234 /system/xbin/gdb-sample
    Process /system/xbin/gdb-sample created; pid = 5230
    Listening on port 1234
 pc: gdbclient 5230 1234 #gdbclient 是AOSP提供的一个脚本可以完成启动gdbserver\符号加载\端口转发启动\连接gdbserver等一系列动作
    或者使用
    pc:adb forward tcp:1234 tcp:1234
    pc:gdb
    pc(gdb):target remote :1234
    pc(gdb):hbreak *main
    pc(gdb):hbreak *(address)
    pc(gdb):watch nGlobalVar　//针对上面的例子
    pc(gdb):continue
    之后在走到你设置的地址的时候,程序会停下来然后打印如下内容
    Old value = 0
    New value = 88
    main () at kernel-study/gdb-sample/gdb-sample.c:28
    28      printf("n = %d, nGlobalVar = %d \n", n, nGlobalVar);
    说明在变量更换的时候就会程序停止，hw watchpoint hw breakpoint都成功了
    实现我们上面的进入某个function的时候加入watch point,退出function的时候取消watchpoint 就不难了
    hbreak *yourfunction
    commands
    > watch variable
    > continue 
    > end
    hbreak *(yourfunction+code_size)
    commands
    > info watchpoints 
            Num     Type           Disp Enb Address            What
            2       hw watchpoint  keep y                      nGlobalVar
            breakpoint already hit 1 time
    > delete 2
    > continue 
    > end
```

参考  
1. 讲述了一些 gdb 基本用法  
[http://blog.csdn.net/xinfuqizao/article/details/7955346](http://blog.csdn.net/xinfuqizao/article/details/7955346)  
2. 介绍了一些新的工具  
[http://www.voidcn.com/blog/kernel_learner/article/p-3555727.html](http://www.voidcn.com/blog/kernel_learner/article/p-3555727.html)

```
一些辅助的诊断及调试工具：
 1）strace：跟踪系统调用情况
 2）ltrace：跟踪动态库的调用情况
 3）mtrace，pmalloc：跟踪内存使用情况，需要嵌入代码，打印内存使用记录。
 4）Binuitls：Toolchain的工具，参考我的上一篇总结。
 5）Valgrind：非常好的内存泄露检测工具，限于i386
 6）oprofile， NPTL Trace Tool等
 7）ald：汇编语言调试器
 8）Dude：另一个运行linux上的调试器，未使用ptrace实现
 9）Linice（http://www.linice.com/）是SoftIce在Linux中的模拟软件，用于调试没有源代码的二进制文件的内核级调试器。
```