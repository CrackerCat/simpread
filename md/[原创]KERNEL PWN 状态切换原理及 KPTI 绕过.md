> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-258975.htm)

> [原创]KERNEL PWN 状态切换原理及 KPTI 绕过

> 本文整理了内核 pwn 中提权返回到用户态时的相关知识点，求大佬轻喷

0x0 System call and return method
=================================

> 本章不关注系统调用函数的参数，以及返回值，只关注系统调用指令本身。
> 
> 这里就拿经典的 int 0x80 与 syscall 来说

int 0x80
--------

int 0x80 是传统的系统调用，它过中断 / 异常实现，在执行 int 指令时，发生 trap。硬件根据向量号 0x80 找到在中断描述符表中的表项，在自动切换到内核栈 (tss.ss0 : tss.esp0) 后根据中断描述符的 segment selector 在 GDT / LDT 中找到对应的段描述符，从段描述符拿到段的基址，加载到 cs ，将 offset 加载到 eip。最后硬件将用户态 ss / sp / eflags / cs / ip / error code 依次压到内核栈。然后会执行 eip 的 entry 函数，通常在保存一系列寄存器后会 SET_KERNEL_GS 设置内核 GS。

 

返回时，最后会执行 SWAPGS 交换内核和用户 GS 寄存器，然后执行 iret 指令将先前压栈的 ss / sp / eflags / cs / ip 弹出，恢复用户态调用时的寄存器上下文。

 

**总结一下**：在提权时，如要使用 64 位的 iretq 指令 从内核态返回到用户态，我们首先要执行 SWAPGS 切换 GS，然后执行 iretq 指令时的栈布局应该如下：

```
rsp ---> rip
         cs
         rflags
         rsp
         ss

```

syscall
-------

根据 Intel SDM，[syscall](https://www.felixcloutier.com/x86/syscall.html) 指令执行时会将当前 rip（syscall 的下一条指令地址） 存到 rcx ，将 rflags 保存到 r11 中。 然后使用 MSR 寄存器中的 IA32_FMASK 屏蔽 rflags，将 IA32_LSTAR 加载到 rip (entry_SYSCALL_64)，同时将 IA32_STAR[47:32] 加载到 cs，IA32_STAR[47:32] + 8 加载到 ss (在 GDT 中，ss 就跟在 cs 后面)。

 

其中的 MSR IA32_LSTAR (MSR_LSTAR) 和 IA32_STAR (MSR_STAR) 在 arch/x86/kernel/cpu/common.c 的 syscall_init 中初始化：

```
void syscall_init(void)
{
    wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
    wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
 
#ifdef CONFIG_IA32_EMULATION
    wrmsrl(MSR_CSTAR, (unsigned long)entry_SYSCALL_compat);
    /*
     * This only works on Intel CPUs.
     * On AMD CPUs these MSRs are 32-bit, CPU truncates MSR_IA32_SYSENTER_EIP.
     * This does not cause SYSENTER to jump to the wrong location, because
     * AMD doesn't allow SYSENTER in long mode (either 32- or 64-bit).
     */
    wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)__KERNEL_CS);
    wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
    wrmsrl_safe(MSR_IA32_SYSENTER_EIP, (u64)entry_SYSENTER_compat);
#else
    wrmsrl(MSR_CSTAR, (unsigned long)ignore_sysret);
    wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
    wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
    wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);
#endif
 
    /* Flags to clear on syscall */
    wrmsrl(MSR_SYSCALL_MASK,
           X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
           X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
}

```

可以看到 MSR_STAR 的第 32-47 位设置为 kernel mode 的 cs，48-63 位设置为 user mode 的 cs。而 IA32_LSTAR 被设置为函数 entry_SYSCALL_64 的起始地址。

 

于是 syscall 时，跳转到 entry_SYSCALL_64 开始执行。

```
ENTRY(entry_SYSCALL_64)
 /* SWAPGS_UNSAFE_STACK是一个宏，x86直接定义为swapgs指令 */
 SWAPGS_UNSAFE_STACK
 
 /* 保存栈值，并设置内核栈 */
 movq %rsp, PER_CPU_VAR(rsp_scratch)
 movq PER_CPU_VAR(cpu_current_top_of_stack), %rsp
 
 
/* 通过push保存寄存器值，形成一个pt_regs结构 */
/* Construct struct pt_regs on stack */
pushq  $__USER_DS      /* pt_regs->ss */
pushq  PER_CPU_VAR(rsp_scratch)  /* pt_regs->sp */
pushq  %r11             /* pt_regs->flags */
pushq  $__USER_CS      /* pt_regs->cs */
pushq  %rcx             /* pt_regs->ip */
pushq  %rax             /* pt_regs->orig_ax */
pushq  %rdi             /* pt_regs->di */
pushq  %rsi             /* pt_regs->si */
pushq  %rdx             /* pt_regs->dx */
pushq  %rcx tuichu    /* pt_regs->cx */
pushq  $-ENOSYS        /* pt_regs->ax */
pushq  %r8              /* pt_regs->r8 */
pushq  %r9              /* pt_regs->r9 */
pushq  %r10             /* pt_regs->r10 */
pushq  %r11             /* pt_regs->r11 */
sub $(6*8), %rsp      /* pt_regs->bp, bx, r12-15 not saved */
 
.......
.......

```

首先通过 swapgs 切换 GS 段寄存器，将 GS 寄存器值和一个特定位置的值进行交换，目的是保存 GS 值，同时将该位置的值作为内核执行时的 GS 值使用。

 

然后将当前栈顶（用户空间栈顶）记录在 CPU 独占变量区域里，将 CPU 独占区域里记录的内核栈顶放入 rsp/esp。

 

最后通过 push 保存各寄存器值......

 

syscall 里面的细节我们不探究，直接看从 syscall 返回那部分

```
LOCKDEP_SYS_EXIT   // 宏的实现与 CONFIG_DEBUG_LOCK_ALLOC 内核配置选项相关，该配置允许在退出系统调用时调试锁。
 TRACE_IRQS_ON       /* user mode is traced as IRQs on */
 movq    RIP(%rsp), %rcx
 movq    EFLAGS(%rsp), %r11
 
 RESTORE_C_REGS_EXCEPT_RCX_R11
 // 恢复除 rxc 和 r11 外所有通用寄存器, 因为 rcx 寄存器为调用系统调用的应用程序的返回地址， r11 寄存器为老的 flags register
 
 /* 根据压栈的内容，恢复 rsp 为用户态的栈顶 */
 movq    RSP(%rsp), %rsp
 
 USERGS_SYSRET64
/* 调用宏 USERGS_SYSRET64 ，其扩展调用 swapgs 指令交换用户 GS 和内核GS， sysret 指令执行从系统调用处理退出 */

```

关注一下 [sysret](https://www.felixcloutier.com/x86/sysret) 指令，它是 syscall 从内核态返回用户态的伴随指令。执行 sysret 时，它从 rcx 加载 rip，并从 r11 加载 rflags，从 MSR 的 IA32_STAR[63:48] 加载 CS ，从 IA32_STAR[63:48] + 8 加载 SS。SYSRET 指令不会修改堆栈指针（ESP 或 RSP），因此在执行 SYSRET 之前 rsp 必须切换到用户堆栈，当然还要切换 GS 寄存器。

 

**总结一下:** 在提权时，当我们使用 sysret 指令从内核态中返回前，我们需要先设置 rcx 为用户态 rip，设置 r11 为用户态 rflags，设置 rsp 为一个用户态堆栈，并执行 swapgs 交换 GS 寄存器。

0x1 About KPTI
==============

在这之前你需要了解内存分页机制。

before
------

每个进程都有一套指向进程自身的页表，由 CR3 寄存器指向。

 

早期的 Linux 内核，每当执行用户空间代码（应用程序）时，Linux 会在其分页表中保留整个内核内存的映射 (内核地址空间和用户地址空间共用一个页全局目录表 PGD)，并保护其访问。这样做的优点是当应用程序向内核发送系统调用或收到中断时，内核页表始终存在，可以避免绝大多数上下文交换相关的开销（TLB 刷新、页表交换等）。

 

尽管阻止了对这些内核映射的访问，但在之后的一段时间，英特尔 x86 处理器还是被爆出了可用于页表泄露的旁路攻击，可能绕过 KASLR.

kpti
----

KPTI(Kernel PageTable Isolation) 全称内核页表隔离, 它通过完全分离用户空间与内核空间页表来解决页表泄露。

 

KPTI 中每个进程有两套页表——内核态页表与用户态页表 (两个地址空间)。内核态页表只能在内核态下访问，可以创建到内核和用户的映射（不过用户空间受 SMAP 和 SMEP 保护）。用户态页表只包含用户空间。不过由于涉及到上下文切换，所以在用户态页表中必须包含部分内核地址，用来建立到中断入口和出口的映射。

 

当中断在用户态发生时，就涉及到切换 CR3 寄存器，从用户态地址空间切换到内核态的地址空间。中断上半部的要求是尽可能的快，从而切换 CR3 这个操作也要求尽可能的快。为了达到这个目的，KPTI 中将内核空间的 PGD 和用户空间的 PGD 连续的放置在一个 8KB 的内存空间中（内核态在低位，用户态在高位）。这段空间必须是 8K 对齐的，这样将 CR3 的切换操作转换为将 CR3 值的第 13 位 (由低到高) 的置位或清零操作，提高了 CR3 切换的速度。

 

![](https://bbs.pediy.com/upload/attach/202004/793907_CSYS7KVBCGWNUB4.jpg)  
开启 KPTI 后，再想提权就比较有局限性，比如我们常用的 ret2usr 方式在 KPTI 下将成为过去时。

swap CR3
--------

下面我们来看一个开启 KPTI 内核的 entry_SYSCALL_64 函数

```
ENTRY(entry_SYSCALL_64)
    /*
     * Interrupts are off on entry.
     * We do not frame this tiny irq-off block with TRACE_IRQS_OFF/ON,
     * it is too small to ever cause noticeable irq latency.
     */
    SWAPGS_UNSAFE_STACK
    // KPTI 进内核态需要切到内核页表
    SWITCH_KERNEL_CR3_NO_STACK
    /*
     * A hypervisor implementation might want to use a label
     * after the swapgs, so that it can do the swapgs
     * for the guest and jump here on syscall.
     */
GLOBAL(entry_SYSCALL_64_after_swapgs)
    // 将用户栈偏移保存到 per-cpu 变量 rsp_scratch 中
    movq    %rsp, PER_CPU_VAR(rsp_scratch)
    // 加载内核栈偏移
    movq    PER_CPU_VAR(cpu_current_top_of_stack), %rsp
 
    TRACE_IRQS_OFF
 
    /* Construct struct pt_regs on stack */
    pushq   $__USER_DS          /* pt_regs->ss */
    pushq   PER_CPU_VAR(rsp_scratch)    /* pt_regs->sp */
    pushq   %r11                /* pt_regs->flags */
    pushq   $__USER_CS          /* pt_regs->cs */
    pushq   %rcx                /* pt_regs->ip */
    pushq   %rax                /* pt_regs->orig_ax */
    pushq   %rdi                /* pt_regs->di */
    pushq   %rsi                /* pt_regs->si */
    pushq   %rdx                /* pt_regs->dx */
    pushq   %rcx                /* pt_regs->cx */
    pushq   $-ENOSYS            /* pt_regs->ax */
    pushq   %r8             /* pt_regs->r8 */
    pushq   %r9             /* pt_regs->r9 */
    pushq   %r10                /* pt_regs->r10 */
    pushq   %r11                /* pt_regs->r11 */
    // 为r12-r15, rbp, rbx保留位置
    sub $(6*8), %rsp            /* pt_regs->bp, bx, r12-15 not saved */
 
    /*
     * If we need to do entry work or if we guess we'll need to do
     * exit work, go straight to the slow path.
     */
    movq    PER_CPU_VAR(current_task), %r11
    testl   $_TIF_WORK_SYSCALL_ENTRY|_TIF_ALLWORK_MASK, TASK_TI_flags(%r11)
    jnz entry_SYSCALL64_slow_path
 
entry_SYSCALL_64_fastpath:
    /*
     * Easy case: enable interrupts and issue the syscall.  If the syscall
     * needs pt_regs, we'll call a stub that disables interrupts again
     * and jumps to the slow path.
     */
    TRACE_IRQS_ON
    ENABLE_INTERRUPTS(CLBR_NONE)
#if __SYSCALL_MASK == ~0
    // 确保系统调用号没超过最大值，超过了则跳转到后面的符号 1 处进行返回
    cmpq    $__NR_syscall_max, %rax
#else
    andl    $__SYSCALL_MASK, %eax
    cmpl    $__NR_syscall_max, %eax
#endif
    ja  1f              /* return -ENOSYS (already in pt_regs->ax) */
    // 除系统调用外的其他调用都通过 rcx 来传第四个参数，因此将 r10 的内容设置到 rcx
    movq    %r10, %rcx
 
    /*
     * This call instruction is handled specially in stub_ptregs_64.
     * It might end up jumping to the slow path.  If it jumps, RAX
     * and all argument registers are clobbered.
     */
    // 调用系统调用表中对应的函数
    call    *sys_call_table(, %rax, 8)
.Lentry_SYSCALL_64_after_fastpath_call:
    // 将函数返回值压到栈中，返回时弹出
    movq    %rax, RAX(%rsp)
1:
 
    /*
     * If we get here, then we know that pt_regs is clean for SYSRET64.
     * If we see that no exit work is required (which we are required
     * to check with IRQs off), then we can go straight to SYSRET64.
     */
    DISABLE_INTERRUPTS(CLBR_NONE)
    TRACE_IRQS_OFF
    movq    PER_CPU_VAR(current_task), %r11
    testl   $_TIF_ALLWORK_MASK, TASK_TI_flags(%r11)
    jnz 1f
 
    LOCKDEP_SYS_EXIT   // 宏的实现与 CONFIG_DEBUG_LOCK_ALLOC 内核配置选项相关，该配置允许在退出系统调用时调试锁。
    TRACE_IRQS_ON       /* user mode is traced as IRQs on */
    movq    RIP(%rsp), %rcx
    movq    EFLAGS(%rsp), %r11
    RESTORE_C_REGS_EXCEPT_RCX_R11
    // 恢复除 rxc 和 r11 外所有通用寄存器, 因为 rcx 寄存器为调用系统调用的应用程序的返回地址， r11 寄存器为老的 flags register
    /*
     * This opens a window where we have a user CR3, but are
     * running in the kernel.  This makes using the CS
     * register useless for telling whether or not we need to
     * switch CR3 in NMIs.  Normal interrupts are OK because
     * they are off here.
     */
    SWITCH_USER_CR3    // KPTI 返回用户态需要切回用户页表
    /* 根据压栈的内容，恢复 rsp 为用户态的栈顶 */
    movq    RSP(%rsp), %rsp
    USERGS_SYSRET64
   /* 调用宏 USERGS_SYSRET64 ，其扩展调用 swapgs 指令交换用户 GS 和内核GS， sysret 指令执行从系统调用处理退出 */
 
........
........

```

可以看出，在入口和结束的地方都加了 SWITCH_CR3 相关的宏定义，尝试着分析 SWITCH_KERNEL_CR3_NO_STACK，里面汇编实现如下：

```
mov     rdi, cr3
nop
nop
nop
nop
nop
and     rdi, 0xFFFFFFFFFFFFE7FF
mov     cr3, rdi

```

拆分 FFFFFFFFFFFFE7FF，它的第 12 和 13 位是零，这段代码目的就是将 CR3 的第 12 位与第 13 位置零（页表的第 12 位在 CR4 寄存器的 PCIDE 位未开启的情况下，都是保留给 OS 留做他用），我们只关心 13 位置零，就相当于 CR3-0x1000，从用户态 PGD 转换成内核态 PGD。

 

再看 SWITCH_USER_CR3 宏定义的汇编：

```
mov     rdi, cr3
or      rdi, 1000h
mov     cr3, rdi

```

同理，将 CR3 第 13 位置 1，相当于 CR3+0x1000，从内核态 PGD 切换成用户态 PGD。

0x2 Bypass KPTI
===============

在开启 KPTI 内核，提权返回到用户态（iretq/sysret）之前如果不设置 CR3 寄存器的值，就会导致进程找不到当前程序的正确页表，引发段错误，程序退出。

 

知道 KPTI 原理，在 kernel 提权返回用户态的时候绕过 kpti 的话就很简单了，利用内核映像中现有的 gadget

```
mov     rdi, cr3
or      rdi, 1000h
mov     cr3, rdi

```

来设置 CR3 寄存器，并按照 iretq/sysret 的需求构造内容，再返回就 OK 了。

 

有一种比较懒惰的方法就是利用 swapgs_restore_regs_and_return_to_usermode 这个函数返回：

 

`cat /proc/kallsyms| grep swapgs_restore_regs_and_return_to_usermode`

```
arch/x86/entry/entry_64.S
 
SYM_INNER_LABEL(swapgs_restore_regs_and_return_to_usermode, SYM_L_GLOBAL)
 
    POP_REGS pop_rdi=0
 
    /*
     * The stack is now user RDI, orig_ax, RIP, CS, EFLAGS, RSP, SS.
     * Save old stack pointer and switch to trampoline stack.
     */
    movq    %rsp, %rdi
    movq    PER_CPU_VAR(cpu_tss_rw + TSS_sp0), %rsp
 
    /* Copy the IRET frame to the trampoline stack. */
    pushq    6*8(%rdi)    /* SS */
    pushq    5*8(%rdi)    /* RSP */
    pushq    4*8(%rdi)    /* EFLAGS */
    pushq    3*8(%rdi)    /* CS */
    pushq    2*8(%rdi)    /* RIP */
 
    /* Push user RDI on the trampoline stack. */
    pushq    (%rdi)
 
    /*
     * We are on the trampoline stack.  All regs except RDI are live.
     * We can do future final exit work right here.
     */
    STACKLEAK_ERASE_NOCLOBBER
 
    SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi
 
    /* Restore RDI. */
    popq    %rdi
    SWAPGS
    INTERRUPT_RETURN

```

纯汇编代码如下：

```
swapgs_restore_regs_and_return_to_usermode
 
.text:FFFFFFFF81600A34 41 5F                          pop     r15
.text:FFFFFFFF81600A36 41 5E                          pop     r14
.text:FFFFFFFF81600A38 41 5D                          pop     r13
.text:FFFFFFFF81600A3A 41 5C                          pop     r12
.text:FFFFFFFF81600A3C 5D                             pop     rbp
.text:FFFFFFFF81600A3D 5B                             pop     rbx
.text:FFFFFFFF81600A3E 41 5B                          pop     r11
.text:FFFFFFFF81600A40 41 5A                          pop     r10
.text:FFFFFFFF81600A42 41 59                          pop     r9
.text:FFFFFFFF81600A44 41 58                          pop     r8
.text:FFFFFFFF81600A46 58                             pop     rax
.text:FFFFFFFF81600A47 59                             pop     rcx
.text:FFFFFFFF81600A48 5A                             pop     rdx
.text:FFFFFFFF81600A49 5E                             pop     rsi
.text:FFFFFFFF81600A4A 48 89 E7                       mov     rdi, rsp    <<<<<<<<<<<<<<<<<<
.text:FFFFFFFF81600A4D 65 48 8B 24 25+                mov     rsp, gs: 0x5004
.text:FFFFFFFF81600A56 FF 77 30                       push    qword ptr [rdi+30h]
.text:FFFFFFFF81600A59 FF 77 28                       push    qword ptr [rdi+28h]
.text:FFFFFFFF81600A5C FF 77 20                       push    qword ptr [rdi+20h]
.text:FFFFFFFF81600A5F FF 77 18                       push    qword ptr [rdi+18h]
.text:FFFFFFFF81600A62 FF 77 10                       push    qword ptr [rdi+10h]
.text:FFFFFFFF81600A65 FF 37                          push    qword ptr [rdi]
.text:FFFFFFFF81600A67 50                             push    rax
.text:FFFFFFFF81600A68 EB 43                          nop
.text:FFFFFFFF81600A6A 0F 20 DF                       mov     rdi, cr3
.text:FFFFFFFF81600A6D EB 34                          jmp     0xFFFFFFFF81600AA3
 
.text:FFFFFFFF81600AA3 48 81 CF 00 10+                or      rdi, 1000h
.text:FFFFFFFF81600AAA 0F 22 DF                       mov     cr3, rdi
.text:FFFFFFFF81600AAD 58                             pop     rax
.text:FFFFFFFF81600AAE 5F                             pop     rdi
.text:FFFFFFFF81600AAF FF 15 23 65 62+                call    cs: SWAPGS
.text:FFFFFFFF81600AB5 FF 25 15 65 62+                jmp     cs: INTERRUPT_RETURN
 
_SWAPGS
.text:FFFFFFFF8103EFC0 55                             push    rbp
.text:FFFFFFFF8103EFC1 48 89 E5                       mov     rbp, rsp
.text:FFFFFFFF8103EFC4 0F 01 F8                       swapgs
.text:FFFFFFFF8103EFC7 5D                             pop     rbp
.text:FFFFFFFF8103EFC8 C3                             retn
 
 
_INTERRUPT_RETURN
.text:FFFFFFFF81600AE0 F6 44 24 20 04                 test    byte ptr [rsp+0x20], 4
.text:FFFFFFFF81600AE5 75 02                          jnz     native_irq_return_ldt
.text:FFFFFFFF81600AE7 48 CF                          iretq

```

在 ROP 时，将程序流程控制到 mov rdi, rsp 指令，栈布局如下就行：

```
rsp  ---->  mov_rdi_rsp
            0
            0
            rip
            cs
            rflags
            rsp
            ss

```

当然改 modprobe_path 也是一个不错的方法，返回后当前进程 Segmentation fault 也不影响提权。

0x3 Reference
=============

[如何检查我的 Ubuntu 上是否启用了 KPTI？](https://ubuntuqa.com/article/1242.html)  
[【内核漏洞利用】TokyoWesternsCTF-2019-gnote Double-Fetch](https://mp.weixin.qq.com/s?__biz=MzI2ODM4NzUyNQ==&mid=2247484029&idx=1&sn=f6bfec08b43c75ca7af6bd12f4999113&chksm=eaf117d7dd869ec1eee66d75abac5d38112b7a63c3bf28edc32960c60d74541a75bb2e4dfe9d&mpshare=1&scene=23&srcid=&sharer_sharetime=1585655592580&sharer_shareid=85875ea7087d5e89ec671378652f4aac#rd)  
[Linux 系统调用过程分析 - 知乎](https://zhuanlan.zhihu.com/p/79236207?from_voters_page=true)  
[KPTI 补丁分析_运维_Linux 阅码场 - CSDN 博客](https://blog.csdn.net/juS3Ve/article/details/79544927)  
[内核页表隔离_百度百科](https://baike.baidu.com/item/%E5%86%85%E6%A0%B8%E9%A1%B5%E8%A1%A8%E9%9A%94%E7%A6%BB/22785908?fr=aladdin)  
[https://www.felixcloutier.com/x86](https://www.felixcloutier.com/x86)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 2020-4-21 22:30 被 b0ldfrev 编辑 ，原因：