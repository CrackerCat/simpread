> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289706.htm)

> [原创] 基于 Apatch 模块的安卓 Linux 内核硬件断点

1. 前沿
=====

自己在处理某大厂的软件的时候处理检测点太恶心了，由于已经知道点位了，所以需要一个无痕断点能断下。在此背景下，开发一款基于内核的硬件调试，来快速通杀这个问题。（ps：后面用这个定位 crc 也好使，下次在单开一个帖子吧）

2. 前置准备
=======

设备：一加 ace 5 pro  
电脑：Windows 10  
内核版本：Linux6.16  
Root 组件：Apatch

3. 基础思路
=======

3.1 遇到的对抗点
----------

Linux 内核提供了 register_user_hw_breakpoint() 等函数来设置硬件断点，但这些函数内部走的还是 perf_event 框架，最终会在进程的 thread_struct 里留下调试状态。目标进程通过检查自身线程状态或者 /proc 下的调试信息，依然能发现被调试的痕迹，所以还要 HOOK 等各种处理，太麻烦了。（ps：就是想要通杀）

3.2 解决方案
--------

由于处理各种检测太麻烦了，Apatch 可以直接对内核打补丁，操作起来比魔改 aosp 方便多了。（ps：我的 px1 太老了懒得折腾了）  
处理思路如图:  
![](https://bbs.kanxue.com/upload/attach/202601/992284_9FT4KF6G6SPNSTC.webp)  
本质上实际上和 Windows 一样，设置 CPU 异常 -> 接管异常——> 处理异常（筛选过滤做我们的处理逻辑即可）。

4. 实现细节
-------

### 4.1 前置条件实现

虽然说起来很简单，但是实际上操作的时候存在大量的问题，最核心的问题就是每个版本的内核是未知，并且编译选项不确定导致内核结构体不确定，ARM64 调试异常触发时，内核里进程信息存在 task_struct 结构体中，结构体的字段偏移在不同内核版本是不一样的，除非自己逆向这个工作量太大了（ps: 源码是很难稳定的推导出结构的，因为很多都是编译选项）  
这个时候我想到 Windows 中的 pdb，经过我的了解 linux 也有内核版本的 btf（pdb）。

### 4.2 BTF 解析

这里给不了解 btf 的介绍一下：  
BTF 是 Linux 内核的一种类型信息格式，内核编译时会把结构体定义、字段偏移这些信息打包进去，也就是我们说的 Windows 的 pdb，并且比 Windows 更爽，因为是在本地的，不用担心服务器下载的问题。需要注意的是必须要版本大于 5.2。这里简单展示一下怎么解析内核中的 BTF，也比较简单，就是一个解析没啥含金量（实在不会就 ai 吧）。  
如图：  
![](https://bbs.kanxue.com/upload/attach/202601/992284_JDTEA7AEXB4VCP8.webp)  
来点伪代码：

```
// 初始化：获取 btf_vmlinux 指针
g_btf_vmlinux_sym = kallsyms_lookup_name("btf_vmlinux");
g_btf_vmlinux = *(struct btf **)g_btf_vmlinux_sym;
 
// 使用：获取 task_struct->pid 的偏移
int offset = kr_btf_offsetof("task_struct", "pid");
 
// 读取当前进程 PID
void *task = get_current_task();
int pid = *(int *)((char *)task + offset);

```

### 4.3 设置调试寄存器

ARM64 提供了专门的调试寄存器，监视点用 DBGWVR（地址）+ DBGWCR（控制），断点用 DBGBVR + DBGBCR。直接内联汇编读写就行。  
写完寄存器后要执行 isb（Instruction Synchronization Barrier），这是 ARM 的指令同步屏障，确保之前的系统寄存器写入在后续指令执行前生效。不加这个的话，CPU 可能还在用旧的寄存器值，断点就不会立即生效。  
WCR 控制寄存器的几个关键位：  
E (bit 0)：启用位，1 = 启用该监视点  
PAC (bit 1-2)：权限控制，3 = EL0 + EL1 都监控  
LSC (bit 3-4)：读写控制，1 = 读，2 = 写，3 = 读写  
BAS (bit 5-12)：字节选择掩码，0xFF = 监控 8 字节  
还有一个关键点：必须启用 MDSCR_EL1 的 MDE 位（bit 15），否则断点 / 监视点不会触发异常。  
核心代码：

```
/ 写入监视点值寄存器 DBGWVR_EL1
static inline void write_dbgwvr(int n, unsigned long val)
{
    switch (n) {
        case 0: asm volatile("msr dbgwvr0_el1, %0" : : "r" (val)); break;
        case 1: asm volatile("msr dbgwvr1_el1, %0" : : "r" (val)); break;
        case 2: asm volatile("msr dbgwvr2_el1, %0" : : "r" (val)); break;
        case 3: asm volatile("msr dbgwvr3_el1, %0" : : "r" (val)); break;
        // ...
    }
    asm volatile("isb");
}
 
// 写入监视点控制寄存器 DBGWCR_EL1
static inline void write_dbgwcr(int n, unsigned long val)
{
    switch (n) {
        case 0: asm volatile("msr dbgwcr0_el1, %0" : : "r" (val)); break;
        case 1: asm volatile("msr dbgwcr1_el1, %0" : : "r" (val)); break;
        case 2: asm volatile("msr dbgwcr2_el1, %0" : : "r" (val)); break;
        case 3: asm volatile("msr dbgwcr3_el1, %0" : : "r" (val)); break;
        // ...
    }
    asm volatile("isb");
}
 
// 在单个 CPU 上设置监视点
static void set_watchpoint_on_cpu(void *info)
{
    int slot = ((int *)info)[0];
    unsigned long addr = g_watchpoints[slot].address;
    int type = g_watchpoints[slot].type;
    int len = g_watchpoints[slot].len;
    unsigned long wcr;
    unsigned long mdscr;
    unsigned long aligned_addr;
 
    // 8 字节对齐
    aligned_addr = (addr & 0x00FFFFFFFFFFFFFFULL) & ~0x7UL;
 
    // 启用 MDSCR_EL1 的 MDE 位
    mdscr = read_mdscr_el1();
    if (!(mdscr & DBG_MDSCR_MDE)) {
        mdscr |= DBG_MDSCR_MDE;
        write_mdscr_el1(mdscr);
    }
 
    write_dbgwvr(slot, aligned_addr);
    wcr = build_wcr(type, len, addr);
    write_dbgwcr(slot, wcr);
} 
```

### 4.4 多核同步

ARM64 的调试寄存器是 per-CPU 的，也就是每个 CPU 核心都有自己独立的一套 DBGWVR/DBGWCR。你在 CPU0 上设置了监视点，CPU1、CPU2 上是没有的。  
问题来了：进程会被调度到不同 CPU 上运行。如果目标进程跑到没设置监视点的 CPU 上，断点就不会触发。  
解决方法是用 smp_call_function，这个内核函数可以让指定的回调在所有其他 CPU 上执行一遍。加上当前 CPU 自己执行一次，就能保证所有核心都设置上监视点。（ps: 这里是笔者遇到的第一个坑）

```
// 先在当前 CPU 执行
slot_arg = slot;
set_watchpoint_on_cpu(&slot_arg);
// 再在其他 CPU 上执行
if (fn_smp_call_function)
    fn_smp_call_function(set_watchpoint_on_cpu, &slot_arg, 1);

```

### 4.5 异常接管

再讲异常接管之前我们需要了解一下 linux 异常整个处理链是什么样子：  
![](https://bbs.kanxue.com/upload/attach/202601/992284_ZTA8WNRRUDXZ69E.webp)

#### 4.5.1 Hook do_debug_exception

监视点设置好之后，CPU 访问目标地址会触发调试异常，内核统一走 do_debug_exception 处理。我们 Hook 这个函数，就能拦截所有调试异常。  
Hook 之后要做几件事：  
解析 ESR（Exception Syndrome Register）判断异常类型  
判断触发地址是不是我们设置的监视点  
是我们的就记录信息，不是就交给原函数处理  
ESR 的 bit [29:27] 表示调试事件类型：  
0x0：硬件断点 (Breakpoint)  
0x1：硬件单步 (Single Step)  
0x2：硬件监视点 (Watchpoint)

```
// Hook do_debug_exception
// 原型: void do_debug_exception(unsigned long addr, unsigned long esr, struct pt_regs *regs)
static void do_debug_exception_before(hook_fargs3_t *args, void *udata)
{
    unsigned long addr = args->arg0;
    unsigned long esr = args->arg1;
    void *regs = (void *)args->arg2;
     
    // 从 ESR 提取事件类型
    unsigned long evt = (esr >> 27) & 0x7;
     
    // 监视点异常
    if (evt == 0x2) {
        int slot = find_our_watchpoint(addr);
        if (slot >= 0) {
            // 是我们的监视点，记录命中信息
            record_watchpoint_hit(slot, addr, esr, regs);
             
            // 设置单步恢复
             
            args->skip_origin = 1;  // 不调用原函数
            return;
        }
    }
     
    // 不是我们的，交给原函数处理
}
 
// 安装 Hook
g_do_debug_exception_addr = kallsyms_lookup_name("do_debug_exception");
hook_wrap3(g_do_debug_exception_addr, do_debug_exception_before, NULL, NULL);

```

### 4.6 异常处理和单步恢复细节问题

在讲异常处理之前我们需要了解一下 Linux 异常整个处理链是什么样子：  
![](https://bbs.kanxue.com/upload/attach/202601/992284_35PNJHBW6PKB4ZD.webp)  
监视点触发后如果不做单步恢复，或者漏掉异常，会有两种情况：  
死循环卡死：返回用户态后 CPU 再次执行那条指令，又触发监视点异常，无限循环。  
进程被杀：如果我们 Hook 了但没正确处理（比如 skip_origin=1 但没做单步恢复），异常没人接管，最终会走到 arm64_notify_die。用户态进程收到 SIGTRAP 信号被杀掉，内核态直接 die() panic。（ps: 之前漏掉了好多异常没有处理导致我一直看黑屏，卡了好久）

### 单步处理的小细节：

单步恢复过程中有一个关键问题：调试寄存器是 per-CPU 的。  
假设进程在 CPU0 上触发了监视点，我们在 CPU0 上临时禁用了监视点并设置了单步。但如果这时候进程被调度器迁移到 CPU1 上执行，问题就来了：  
CPU1 上的监视点还是启用的（我们只禁用了 CPU0 的）  
进程在 CPU1 上执行那条指令，又触发监视点异常，但 CPU1 上没有我们的单步状态记录，逻辑混乱，可能死循环或者崩溃。（ps：这里又卡了我好一上午）  
来点核心代码：

```
static void do_debug_exception_before(hook_fargs3_t *args, void *udata)
{
    unsigned long addr = args->arg0;
    unsigned long esr = args->arg1;
    void *regs = (void *)args->arg2;
    unsigned long evt = (esr >> 27) & 0x7;
    per_cpu_step_state_t *cpu_state = get_cpu_step_state();
 
    // 单步异常 - 恢复阶段
    if (evt == ESR_EVT_HWSS) {
        if (cpu_state->stepping_pid != 0) {
            unsigned long mdscr = read_mdscr_el1();
            mdscr &= ~DBG_MDSCR_SS;
            write_mdscr_el1(mdscr);
 
            enable_our_watchpoints_on_cpu();
 
            if (cpu_state->migration_disabled && fn_migrate_enable) {
                fn_migrate_enable();
                cpu_state->migration_disabled = 0;
            }
 
            cpu_state->stepping_pid = 0;
            args->skip_origin = 1;
            return;
        }
    }
 
    // 监视点异常 - 触发阶段
    if (evt == ESR_EVT_HWWP) {
        int slot = find_our_watchpoint(addr);
        if (slot >= 0) {
            record_watchpoint_hit(slot, addr, esr, regs);
 
            if (fn_migrate_disable && !cpu_state->migration_disabled) {
                fn_migrate_disable();
                cpu_state->migration_disabled = 1;
            }
 
            disable_our_watchpoints_on_cpu();
 
            unsigned long *pstate_ptr = (unsigned long *)((char *)regs + 0x108);
            *pstate_ptr |= DBG_SPSR_SS;
 
            unsigned long mdscr = read_mdscr_el1();
            mdscr |= DBG_MDSCR_SS;
            write_mdscr_el1(mdscr);
 
            cpu_state->stepping_pid = get_current_pid();
            cpu_state->stepping_slot = slot;
 
            args->skip_origin = 1;
            return;
        }
    }
}

```

### 4.7 断下后的操作

断点触发后，我们在 do_debug_exception_before 这个 Hook 函数里处理。调用 record_watchpoint_hit 记录命中信息时，可以拿到完整的上下文：触发地址、ESR、pt_regs（包含所有寄存器）。  
在这里你可以加入自己的逻辑，想干什么都行。  
调用栈回溯：  
通过 FP (Frame Pointer) 链可以遍历整个用户态调用栈。ARM64 的栈帧结构是 [prev_fp, return_addr]，沿着 FP 一直往上走就能拿到完整调用链。

```
// 在 do_debug_exception_before 里，regs 就是 pt_regs
void *regs = (void *)args->arg2;
// 从 pt_regs 获取寄存器
unsigned long pc = *(unsigned long *)((char *)regs + 0x100);
unsigned long lr = *(unsigned long *)((char *)regs + 0xF0);  // X30
unsigned long fp = *(unsigned long *)((char *)regs + 0xE8);  // X29
// 遍历 FP 链
while (fp != 0 && depth < MAX_DEPTH) {
    unsigned long frame[2];
    // frame[0] = prev_fp, frame[1] = return_addr
    if (kr_copy_from_user(frame, (void __user *)fp, sizeof(frame)) != 0)
        break;
     
    pr_info("[%d] LR = 0x%lx\n", depth, frame[1]);
     
    fp = frame[0];
    depth++;
}

```

寄存器 / 参数读取  
pt_regs 里存着触发时刻所有寄存器的值，ARM64 函数调用约定前 8 个参数走 X0-X7：

```
// pt_regs 布局: X0-X30 依次排列，每个 8 字节
unsigned long x0 = *(unsigned long *)((char *)regs + 0x00);  // 第1个参数
unsigned long x1 = *(unsigned long *)((char *)regs + 0x08);  // 第2个参数
unsigned long x2 = *(unsigned long *)((char *)regs + 0x10);  // 第3个参数
// ...

```

这里不做介绍了，有时候遇到一些难搞的点，但是活得又快又好，可以用这个操作代替一些 frida 的分析过程，暴力快速解决，早点定位到位置，早点下班！！！

总结
--

整个实现流程不难，主要还是对 linux 源码熟悉，这里我建议用 ai 阅读源码，然后自己编写，这样快速熟悉 linux 内核。

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#源码框架](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)