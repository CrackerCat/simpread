> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277163.htm)

> QEMU tcg 源码分析与 unicorn 原理

一. 前言
-----

最近对于虚拟化技术在操作系统研究以及在二进制逆向 / 漏洞分析上的能力很感兴趣。看雪最近两年的 sdc 也都有议题和虚拟化相关 (只不过出于性能的考虑用的是 kvm 而不是 tcg):  
2021 年的是 "基于 Qemu/kvm 硬件加速下一代安全对抗平台"  
2022 年的是 "基于硬件虚拟化技术的新一代二进制分析利器"

 

跨平台模拟执行 unicorn 框架和 qiling 框架都是基于 qemu 的 tcg, 本文的内容就是描述一下 qemu tcg 与 unicorn 的原理。

 

TCG 的英文含义是 Tiny Code Generator, Qemu 可以在没开启硬件虚拟化支持的时候实现全系统的虚拟化，Qemu 结合下面几种技术共同实现虚拟化:

1.  soft tlb / Softmmu / 内存模拟
2.  虚拟中断控制器 / 中断模拟
3.  总线 / 设备模拟
4.  TCG 的 CPU 模拟

qemu 进程代表着一个完整的虚拟机在运行，它没有特殊的权限却能正常的运行各种操作系统如 windows/linux 等，在没有硬件虚拟化支持的时候靠的最主要的角色就是 TCG, 它满足了 Popek/Goldberg 对于虚拟化的三大要求:

1.  等价性
2.  安全性
3.  性能

[https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements](https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements)

 

后来出现了硬件支持的虚拟化, kvm 因此成为主流, 在云平台上运行的虚拟化都是有硬件支持的, 但是 TCG 却仍然是不可替代的, 因为硬件虚拟化只能在源 ISA 和目标 ISA 都相同的情况下才能工作 (比如在 x86 平台虚拟化 x86 操作系统, 或者在 arm 平台下虚拟化 arm 操作系统), 而如果源 ISA 和目标 ISA 不同的情况下如在 x86 平台运行 arm 操作系统, 只能靠 TCG 实现。

 

学习 TCG 的好处:

1.  可以理解像 libhoudini.so 这样的转码技术是如何实现的
2.  对理解应用层的虚拟机如 java 虚拟机中的 jit 技术很有帮助
3.  可以帮助理解 cpu 包括整个计算机体系结构是如何工作的
4.  可以帮助理解和定制二进制分析框架如 unicorn/qiling，因为它们都是基于 TCG
5.  某些 vmp 是基于 unicorn 来实现的, 理解 TCG 可以基于此实现自己的 vmp / 加深对 vmp 的理解

二. QEMU TCG
-----------

### 1. DBT

TCG 本质上属于 DBT, 即 dynamic binary translation 动态二进制转换，相应的还有 SBT, 即 static binary translation 静态二进制转换。拿 Android 平台举例, SBT 就相当于 ART 虚拟机中的 AOT(ahead-of-time compilation), 而 DBT 就相当于 ART 虚拟机中的 JIT(Just-In-Time compilation)。

 

假如想在 x86 平台运行 arm 程序，称 arm 为 source ISA, 而 x86 为 target ISA, 在虚拟化的角度来说 arm 就是 Guest, x86 为 Host。

 

最简单的解决方案是实现一个解释器，在一个循环中不断的读入 arm 程序指令，反汇编并用代码去模拟指令的执行。但是解释器的问题在于性能太低，后来就出来了 DBT 技术 (QEMU 也有解释器模块，具体搜索 CONFIG_TCG_INTERPRETER)，它也需要读入 arm 程序指令并进行反汇编，不过接下来流程会进入即时编译环节，将 arm 指令转换成 x86 指令，最终执行的时候会直接跳转到转换过的 x86 指令执行，得到媲美于本地执行的性能。

 

DBT 和 JIT 这两个名词经常可以互换使用，不过我的理解是 JIT 环境中的输入是特意被设计过的可被模拟的指令格式 (更多的是高层虚拟机如 java 虚拟机中的字节码)，而 DBT 的输入则是不同平台的 ISA 指令。

 

对于虚拟化来说, 可以采用 SBT 将 Guest 代码事先编译好然后直接运行吗? 对于模拟某些 ISA 如 x86 来说会遇到问题，因为 x86 的指令是不定长的，和反汇编器会遇到的问题一样，有时候是无法准确区分出哪些是数据哪些是指令，当遇到一些运行时才知道目标的跳转指令，SBT 技术会遇到问题。这种问题被称为 Code-Discovery Problem。

 

而 DBT 则不会有此问题，以下称 source ISA 中的 pc 指针为 SPC, target ISA 中的 pc 指针为 TPC, 对于模拟一个 arm 系统来说，arm 系统刚上电 cpu 会从物理地址 0 从开始执行，此时 SPC=0, 假设此处的指令为 "mov r0, #0", 而经过 DBT 转换以后，转换的代码位于 Qemu 进程的虚拟地址 0x7fbdd0000100 处，此时的 TPC=0x7fbdd0000100, 转换后的指令为 x86 指令  
"movl $0, (%rbp)"，DBT 技术中会实时记录 SPC 与 TPC 的关系，遇到跳转指令的时候可以得到跳转指令的目标地址，因此不会有 SBT 中的问题。

### 2.QEMU IR

类似于 LLVM,QEMU 也定义有自己的 IR:  
[https://www.qemu.org/docs/master/devel/tcg-ops.html](https://www.qemu.org/docs/master/devel/tcg-ops.html)

 

转换过程如下:  
![](https://bbs.kanxue.com/upload/attach/202305/607812_MA2SPK7AVBM8QTR.png)

 

引入 IR 的好处自然是当引入一种新的 source ISA 的时候，只需要完成 source binary code 到 IR 的转换，IR 到 target binary code 直接用现成的即可。

 

从上面 QEMU IR 的链接中可知, QEMU IR 的指令主要分为函数调用指令、跳转指令、算术指令、逻辑指令、条件移动指令、类型转换指令、加载 / 存储指令等构成，那么问题来了，仅靠固定的 IR 是无法模拟所有的 ISA 指令的，比如 x86 架构的 cpuid 指令并没有与之对应的 IR, 遇到这种指令如何生成对应的 IR?

 

这就涉及到执行上下文的概念，QEMU 本身是 Host 上一个普通的进程，运行在 QEMU 上下文，而执行转换后的目标代码则运行在虚拟机上下文，当运行在虚拟机上下文的程序遇到一些条件时会退出至 QEMU 上下文处理，像在 arm 平台执行 cpuid 指令就是这种情况，需要生成 IR 调用 QEMU 中的 helper 函数来模拟 cpuid 指令，模拟完了再回退到虚拟机上下文去执行。每个体系结构对应的 helper 函数在 target/xxx/helper.h 头文件中定义。

 

include/tcg/tcg-op.h 文件声明了在实现一个生成 IR 的前端时可以调用的一些函数, 这些函数以 tcg_gen_ 开头。

### 3.Basic Block/Translation Block

TCG 的二进制转换是以块为基本单元，即 Basic Block，当 Guest 指令遇到下面几种情况时会被分割成一个 Basic Block:

1.  遇到分支指令
2.  遇到系统调用
3.  达到页边界 / 最大长度限制

而 TranslationBlock 是 QEMU 中用来表示转换过的 Host 指令的数据结构 (以下简称 TB)，执行时的基本控制流程如下:  
![](https://bbs.kanxue.com/upload/attach/202305/607812_T6XTB8SWCNNDMFS.png)

 

QEMU TCG Engine 运行在 QEMU 上下文，当一个 Basic Block 被转换成 Tranlated Block 以后, QEMU 可以直接跳转过去以虚拟化上下文去执行，这种跳转是以函数调用的形式来实现的，因此还需要执行一些 prologue"前言" 代码来保存函数调用时的信息，需要切换回 TCG 上下文时需要执行一些 epilogue"序言" 代码来恢复函数调用前的信息。

 

拿 x86_64 平台举例，每次执行上下文切换需要执行大约 20 条指令 (指令还会进行内存的读写)，因此 DBT 的优化措施之一就是减少上下文切换，实现 TB 之间的直接链接，这种优化措施称为 Direct block chaining:  
![](https://bbs.kanxue.com/upload/attach/202305/607812_Q7P37XFW4F9XNPD.png)

 

这种优化措施可以显著的增加性能，但是这种优化方式还需要解决自修改代码引发的问题，在收到硬件中断时还需要快速的返回至 QEMU 上下文处理，等后面具体分析代码的时候会描述。

 

不过 chained tb 有一个限制: 两个 chained tb 对应的 guest 指令需要在同一个 guest 的 page 里。

 

将指令分割为 Basic Block 的一个主要原因是 TB 的缓存机制，当一个 Basic Block 被 DBT 转换为 TB 以后，下次再执行到相同的 Basic Block 直接从缓存中获取 TB 执行即可，无需再经过转换:  
![](https://bbs.kanxue.com/upload/attach/202305/607812_TQCUGEVX8ZR9C26.png)

### 4. 代码环境

以 x86_64 平台上运行一段 arm 程序做为研究对象，使用的 QEMU 源码分支为 stable-8.0。  
需要准备三个文件 startup.s, test.c,test.ld:

 

startup.s 文件内容:

```
.global _Reset
_Reset:
 LDR sp, =stack_top
 BL c_entry
 B .

```

test.c 文件内容:

```
volatile unsigned int * const UART0DR = (unsigned int *)0x101f1000;
void print_uart0(const char *s) {
    while(*s != '\0') { /* Loop until end of string */
        *UART0DR = (unsigned int)(*s); /* Transmit char */
        s++; /* Next char */
    }
}
void c_entry() {
    print_uart0("Hello world!\n");
}

```

test.ld 文件内容:

```
ENTRY(_Reset)
SECTIONS
{
 . = 0x10000;
 .startup . : { startup.o(.text) }
 .text : { *(.text) }
 .data : { *(.data) }
 .bss : { *(.bss COMMON) }
 . = ALIGN(8);
 . = . + 0x1000; /* 4kB of stack memory */
 stack_top = .;
}

```

**  
编译:**

```
arm-none-eabi-gcc -c -mcpu=arm926ej-s -g test.c -o test.o
arm-none-eabi-as -mcpu=arm926ej-s -g startup.s -o startup.o
arm-none-eabi-ld -T test.ld test.o startup.o -o test.elf
arm-none-eabi-objcopy -O binary test.elf test.bin

```

以上代码的用途是往串口 0x101f1000 处写入 Hello World，代码的链接地址为 0x10000, 它期望被加载到物理内存的地址也是 0x10000，很多 arm 机器将内核加载至此。

 

**启动:**

```
qemu-system-arm -M versatilepb -m 128 -kernel test.bin -nographic

```

会在屏幕上打印出 Hello World!，此时退出 QEMU 的快捷键为 Ctrl+A X

 

qemu-system-arm 程序是由 QEMU 源码编译出来的，-M versatilepb 表示模拟的 arm 硬件为 versatilepb(Arm Versatile boards),-m 128 参数表示指定的机器内存为 128M, -kernel 参数为 QEMU 的 Direct Linux Boot 机制，由 QEMU 而不是磁盘上的 Bootloade 来将内核加载至内存。这种情况下启动，arm 会从物理地址 0 开始执行，事实上 0 地址处是 qemu 实现的一小段 bootloader，只是用来将控制跳转到 0x10000 内核处执行 (test.bin)，代码在 hw/arm/boot.c 文件中:

```
/* A very small bootloader: call the board-setup code (if needed),
 * set r0-r2, then jump to the kernel.
 * If we're not calling boot setup code then we don't copy across
 * the first BOOTLOADER_NO_BOARD_SETUP_OFFSET insns in this array.
 */
static const ARMInsnFixup bootloader[] = {
    { 0xe28fe004 }, /* add     lr, pc, #4 */
    { 0xe51ff004 }, /* ldr     pc, [pc, #-4] */
    { 0, FIXUP_BOARD_SETUP },
#define BOOTLOADER_NO_BOARD_SETUP_OFFSET 3
    { 0xe3a00000 }, /* mov     r0, #0 */
    { 0xe59f1004 }, /* ldr     r1, [pc, #4] */
    { 0xe59f2004 }, /* ldr     r2, [pc, #4] */
    { 0xe59ff004 }, /* ldr     pc, [pc, #4] */
    { 0, FIXUP_BOARDID },
    { 0, FIXUP_ARGPTR_LO },
    { 0, FIXUP_ENTRYPOINT_LO },
    { 0, FIXUP_TERMINATOR }
};

```

### 5. 打印出 TCG 转换的文件

从 qemu 7.1 开始反汇编引擎已经替换为 Capstone, 因此需要安装 capstone:

```
sudo apt install libcapstone-dev

```

qemu 提供了一些调试手段可以显示出 TCG 转换过程的内容:

```
qemu-system-arm -M versatilepb -m 128 -kernel test.bin -nographic -d in_asm -D in_asm.txt
 
qemu-system-arm -M versatilepb -m 128 -kernel test.bin -nographic -d op -D op.txt
 
qemu-system-arm -M versatilepb -m 128 -kernel test.bin -nographic -d out_asm -D out_asm.txt

```

in_asm.txt 为 arm 反汇编程序的结果  
op.txt 为生成的 IR 指令的内容  
out_asm 为转换后的 Host 指令的内容

 

分析 TCG 的时候，由于它拥有全系统虚拟化的能力，因此需要思考如下几种情况是如何实现的:

1.  普通算术逻辑运算指令如何更新 Host 体系结构相关寄存器
2.  内存读写如何处理
3.  分支指令 (条件跳转、非条件跳转、返回指令）
4.  目标机器没有的指令、特权指令、敏感指令
5.  非普通内存读写如设备寄存器访问 MMIO
6.  指令执行出现了同步异常如何处理 (如系统调用)
7.  硬件中断如何处理

### 6.TCG 相关数据结构

qemu 中一个 tcg 线程可以模拟多个 vcpu, 也可以多个 tcg 线程每个对应模拟一个 vcpu, 后者称为 Multi-Threaded TCG (MTTCG), 是否为 MTTCG 由全局变量 bool mttcg_enabled 决定。对于此处的示例 MTTCG 是开启状态，不过简单起见这里假设机器只有一个 vcpu。

 

先来看一下 TCG 的一些重要数据结构:

1.  TranslationBlock:  
    顾名思义，存放编译后的 TB 相关信息，包括指向目标机器执行码的指针
    
2.  CPUArchState:  
    由于是模拟的 cpu, 因此存放着体系结构的 cpu 信息，比如对于 arm 平台它定义在 target/arm/cpu.h 文件中，成员包括所有通用寄存器以及状态码等模拟 cpu 硬件所必需的信息。
    
3.  TCGContext:  
    存放 tcg 中间存储数据的结构体，包括转换后的 IR, tcg 的核心就是围绕此结构展开, 前端 IR 以 TCGOp 列表的形式存放在 TCGContext 的 ops 对象中
4.  TCGTemp:  
    对应于 tcg IR 中的变量，存放在 TCGContext 的 temps 数组中, 变量有几种不同的作用域类型  
    `tcg_temp_new_internal`分配`TEMP_EBB, TEMP_TB`类型的 TCGTemp 变量  
    `tcg_global_alloc`分配`TEMP_GLOBAL`类型的 TCGTemp 变量  
    `tcg_global_reg_new_internal`分配`TEMP_FIXED`类型的 TCGTemp 变量  
    `tcg_constant_internal`分配`TEMP_CONST`类型的 TCGTemp 变量

TCPTemp 每个类型的含义如下:

```
typedef enum TCGTempKind {
    /*
     * Temp is dead at the end of the extended basic block (EBB),
     * the single-entry multiple-exit region that falls through
     * conditional branches.
     */
    TEMP_EBB,
    /* Temp is live across the entire translation block, but dead at end. */
    TEMP_TB,
    /* Temp is live across the entire translation block, and between them. */
    TEMP_GLOBAL,
    /* Temp is in a fixed register. */
    TEMP_FIXED,
    /* Temp is a fixed constant. */
    TEMP_CONST,
} TCGTempKind;

```

那么编译后的 Host 代码存放在哪里? 先看一下这幅图:  
![](https://bbs.kanxue.com/upload/attach/202305/607812_S84X6S37KEF9NM9.png)

 

在 qemu 启动的早期会执行一个函数叫`tcg_init_machine`  
在这个函数中会调用`qemu_memfd_create()`函数创建出一个匿名文件, 该匿名文件的大小是根据当前 Host 机器的物理内存计算出来的，比如我的电脑是 64G, 最终计算出来的匿名文件大小为 1G。

 

然后对匿名文件做两次映射，一次映射为读写:`(PROT_READ | PROT_WRITE)`，称之为 **buf_rw**。

 

一次映射为写执行`(PROT_READ | PROT_EXEC)`，称之为 **buf_rx**, **buf_rw 和 buf_rx** 之间的差值由全局变量`tcg_splitwx_diff`表示。

 

tcg 在翻译代码的过程中会利用 buf_rw 写这 1G 的空间，而执行的过程中则依赖于 buf_rx 在这 1G 的空间中执行代码。由于 buf_rw 和 buf_rx 映射的是同一个文件且指定了 MAP_SHARED 参数，因此对 buf_rw 做出的修改会在 buf_rx 的空间可见。

 

`tcg_init_machine`函数还会调用`tcg_target_qemu_prologue`函数创建出对应于 Host 的 prologue 和 epilogue, 并且分别由全局变量`tcg_qemu_tb_exec`和`tcg_code_gen_epilogue`指向 (如上图)。

 

对于 Host 为 x86_64 来说，它的 prologue 如下:

```
//保存callee需要保存的寄存器
0x7fffac000000:  55                       pushq    %rbp
0x7fffac000001:  53                       pushq    %rbx
0x7fffac000002:  41 54                    pushq    %r12
0x7fffac000004:  41 55                    pushq    %r13
0x7fffac000006:  41 56                    pushq    %r14
0x7fffac000008:  41 57                    pushq    %r15
 
//第一个参数赋值给%rbp
0x7fffac00000a:  48 8b ef                 movq     %rdi, %rbp
 
//预留栈空间
0x7fffac00000d:  48 81 c4 78 fb ff ff     addq     $-0x488, %rsp
 
//跳转到第二个参数地址处执行,第二个参数即为TranslationBlock.tc.ptr
0x7fffac000014:  ff e6                    jmpq     *%rsi

```

它的 epilogue 如下:

```
//恢复栈空间及callee需要保存的寄存器
0x7fffac000016:  33 c0                    xorl     %eax, %eax
0x7fffac000018:  48 81 c4 88 04 00 00     addq     $0x488, %rsp
0x7fffac00001f:  c5 f8 77                 vzeroupper
0x7fffac000022:  41 5f                    popq     %r15
0x7fffac000024:  41 5e                    popq     %r14
0x7fffac000026:  41 5d                    popq     %r13
0x7fffac000028:  41 5c                    popq     %r12
0x7fffac00002a:  5b                       popq     %rbx
0x7fffac00002b:  5d                       popq     %rbp
0x7fffac00002c:  c3                       retq

```

假设现在正在翻译第一个 TB,`TranslationBlock`结构也是在 1G 的空间内分配，第一个 TB 紧接着 epilogue, 并且分配了 TB 以后 TCGContext 的 code_gen_ptr 将会指向 TB 的末端，该 TB 对应的 Host 机器码地址存放在`TranslationBlock.tc.ptr`中，属于 buf_rx 空间。

 

而 buf_rw 空间中 TB 对应的 Host 机器码的开头由 TCGContext 的 code_buf 指向，末端由 TCGContext 的 code_ptr 指向，两者之差则为机器码的长度。需要翻译第二个 TB 时, 第二个 TranslationBlock 结构则会在`TCGContext.code_ptr`的后面再分配，TCGContext 的 code_buf 和 code_ptr 则再指向第二个 TB 对应的 Host 机器码的开头和末端，此时 TCGContext 的 code_gen_ptr 则再更新为第二个 TB 末端的位置。

 

如何执行编译后的代码? 直接执行`tcg_qemu_tb_exec()`函数即可, 该函数接受两个参数，第一个参数为`CPUArchState`，第二个参数为`TranslationBlock.tc.ptr`, 因此 TB 执行逻辑为:

1.  prologue
2.  TranslationBlock.tc.ptr
3.  epilogue

如果做了 Direct block chaining 优化则不会再有 epilogue，会跳转到下一个 TB 执行。

### 7.tcg 执行流程

tcg 线程始于`mttcg_cpu_thread_fn`, 执行流程为:

```
mttcg_cpu_thread_fn:
    do{
        if (cpu_can_run(cpu)) {
            ...
            tcg_cpus_exec(cpu)
                cpu_exec_start(cpu)
                cpu_exec(cpu)
                    cpu_exec_enter(cpu)
                    cpu_exec_setjmp(cpu, &sc)
                        sigsetjmp(cpu->jmp_env, 0) //设置同步异常退出点
                        cpu_exec_loop(cpu, sc)
                    cpu_exec_exit(cpu)
                cpu_exec_end(cpu)
            ...
        }
    } while (!cpu->unplug || cpu_can_run(cpu));

```

主要执行函数在`cpu_exec_loop`中, 它的执行过程为:

```
cpu_exec_loop:
    while (!cpu_handle_exception(cpu, &ret)) { //处理同步异常
        while (!cpu_handle_interrupt(cpu, &last_tb)) { //处理异步中断
            cpu_get_tb_cpu_state()
            tb = tb_lookup() //查找tb缓存
            if (tb == NULL) {
                tb = tb_gen_code() //进行dbt转换
                    setjmp_gen_code()
                        gen_intermediate_code() //将Guest代码转换为IR
                        tcg_gen_code() //根据IR生成Host代码
            }
            tb_add_jump() //Direct block chaining优化             
            cpu_loop_exec_tb() //执行Host目标代码
        }
    }

```

tcg_gen_code 在生成 Host 代码之前还会基于当前的 IR 做一些优化。优化函数有`tcg_optimize, reachable_code_pass,liveness_pass_0,liveness_pass_1,liveness_pass_2`等。

### 8. 普通算术逻辑运算指令如何更新 Host 体系结构相关寄存器

对于一条 Guest 指令来说, tcg 的处理是将它翻译为语义等价的多条 IR(称为微码), 比如  
`in_asm.txt`文件中显示出 0 地址处的 arm 指令为:

```
0x00000000:  e3a00000  mov      r0, #0

```

它编译为微码 IR 的结果为:

```
---- 00000000 00000000 00000000
 mov_i32 loc5,$0x0　//0x0赋值给loc5变量
 mov_i32 r0,loc5 //loc5再赋值给r0

```

loc5 这种变量为 tcg 的 TCGTemp, 而 r0 则对应着 arm 的 r0 寄存器，因此 tcg 的 IR 其实并非和 llvm 中的 IR 那样和平台完全无关，它是和平台相关的。

 

这种翻译方式的优点是可以避免处理不同指令集的复杂性，但是缺点是以性能为代价（通常减慢 5-10 倍）。

 

再来看一下这条指令:

```
0x00000004:  e59f1004  ldr      r1, [pc, #4]

```

它对应的 IR:

```
---- 00000004 00000000 00000e04
 add_i32 loc6,pc,$0x10   //loc6 = pc + 0x10
 mov_i32 loc9,loc6      //loc9 = loc6
 qemu_ld_i32 loc8,loc9,leul,2 //loc9处的内存加载至loc8变量,leul的含义为Little Endian unsigned long
 mov_i32 r1,loc8 //loc8赋值给r1寄存器

```

因此通过组合微码以及结合 qemu 的 helper 函数，可以将 Guest 的所有指令都编译为语义等价的 IR，在微码的基础上进行一些优化以后再根据微码一条一条的翻译成 Host 指令。

 

DBT 需要解决的一个问题是如何进行 state mapping 状态绑定，拿 0x00000000 处的指令举例，这条指令将 r0 寄存器的值赋给 0, 当执行完编译过的 Host 指令以后，需要相应的在某个状态中记录下 r0 寄存器值为 0, 如果 Host 的寄存器数量很多，完全可以选一个 x86_64 寄存器作为 arm 中 r0 寄存器的对应物 (寄存器绑定)，否则就需要保存在内存中了。

 

对于 tcg 来说有个特殊的寄存器叫`TCG_AREG0`, 它表示用哪个 Host 寄存器来指向 Guest 体系结构的`CPUArchState`, 对于 x86_64 来说 TCG_AREG0 为 %rbp(对于 arm 来说 TCG_AREG0 为 r6 寄存器)，也就是说通过 rbp 寄存器可以找到 arm 的`CPUArchState`。qemu 中有专门的 TEMP_FIXED 类型的 TCGTemp 用于表示`TCG_AREG0`:

```
ts = tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env");
cpu_env = temp_tcgv_ptr(ts);

```

cpu_env 在 IR 中被使用的话，在生成 Host 代码阶段对于 x86_64 来说将会绑定到 rbp 寄存器。

 

事实上 qemu 中一共只有两个 TEMP_FIXED 类型的 TCGTemp, 一个叫 env, 一个叫_frame。

 

来看一下 CPUArchState 的通用寄存器成员为

```
uint32_t regs[16];

```

因此 arm 指令

```
0x00000000:  e3a00000  mov      r0, #0

```

对应编译过的 x86_64 代码如下:

```
movl     $0, 0(%rbp)  //rbp指向CPUArchState,更新arm CPUArchState的regs[0]即r0寄存器

```

同样的对于这条 arm 指令它最终改变了 r2:

```
ldr      r2, [pc, #4]

```

对应编译过的 x86_64 代码如下:

```
movl     %r12d, 8(%rbp) //rbp指向CPUArchState,更新arm CPUArchState的regs[2]即r2寄存器

```

当然如果在一个 TB 内如果每遇到一个指令改变了寄存器都要写入 x86_64 的 rbp 对应的内存地址是非常慢的，tcg 有个优化措施是可以在 TB 结束之前只执行一次更新操作从而减少写内存的操作。

 

因此通过`TCG_AREG0`寄存器，x86_64 指令在执行的时候可以找到 CPUArchState 结构从而更新所有 Guest 体系结构的 CPU 状态。

### 9. 内存读写如何处理

对于 qemu 来说读写内存涉及到内存模拟模块，qemu 还模拟了 tlb，因此读写一块 arm 的虚拟内存地址 (Guest Virtual Address -> GVA) 首先会查询 tlb, 如果 tlb 不命中的话会走 tlb 慢路径。tlb 慢路径要经由 guest 的 mmu 经页表转换为物理内存地址(Guest Physics Address -> GPA), 再经过 qemu 内存管理模块转换为 qemu 进程的虚拟地址(Host Virtual Address -> HVA)。  
那么读写 GVA 的 arm 指令编译成 X86_64 指令就是读写对应的 HVA 即可。

 

tlb 相应的数据结构在`include/exec/cpu-defs.h`文件中定义，其中结构体 CPUTLB 由 ArchCPU 中的`CPUNegativeOffsetState neg`所引用。

 

TLB 命中时对应 CPUTLBEntry 对象的 addend + GVA = HVA。

 

CPUTLBEntry 对象的`addr_read, addr_write, addr_code`分别对应着读写执行指令的地址，地址的构成部分注释中有描述:

```
/* bit TARGET_LONG_BITS to TARGET_PAGE_BITS : virtual address
       bit TARGET_PAGE_BITS-1..4  : Nonzero for accesses that should not
                                    go directly to ram.
       bit 3                      : indicates that the entry is invalid
       bit 2..0                   : zero
    */

```

以如下指令举例:

```
0x00000004:  e59f1004  ldr      r1, [pc, #4]

```

它对应的 IR 为:

```
---- 00000004 00000000 00000e04
 add_i32 loc6,pc,$0x10    
 mov_i32 loc9,loc6
 qemu_ld_i32 loc8,loc9,leul,2
 mov_i32 r1,loc8

```

上面最主要的是 qemu_ld_i32 这条 IR,loc9 的值为 GVA,qemu_ld_i32 则将 loc9 地址处的内存加载至 loc8 变量中并最终赋值给 r1 寄存器。

 

qemu_ld_i32 这条 IR 它对应的 x86_64 代码如下:

```
-- guest addr 0x00000004
0x7ff9c0000119:  41 8b fc                 movl     %r12d, %edi
0x7ff9c000011c:  c1 ef 05                 shrl     $5, %edi
0x7ff9c000011f:  23 bd 10 ff ff ff        andl     -0xf0(%rbp), %edi
0x7ff9c0000125:  48 03 bd 18 ff ff ff     addq     -0xe8(%rbp), %rdi
0x7ff9c000012c:  41 8d 74 24 03           leal     3(%r12), %esi
0x7ff9c0000131:  81 e6 00 fc ff ff        andl     $0xfffffc00, %esi
0x7ff9c0000137:  3b 37                    cmpl     0(%rdi), %esi
0x7ff9c0000139:  41 8b f4                 movl     %r12d, %esi
0x7ff9c000013c:  0f 85 9c 00 00 00        jne      0x7ff9c00001de
0x7ff9c0000142:  48 03 77 10              addq     0x10(%rdi), %rsi
0x7ff9c0000146:  44 8b 26                 movl     0(%rsi), %r12d

```

乍一看相当复杂的不知道在做什么，其实上面执行的逻辑是创建出调用 qemu tlb 的环境，先去 tlb 查询是否有对应的 HVA, 如果没有的话会生成一段 tlb slow path 的代码并跳转到 tlb slow path 去执行。生成这段 x86_64 的代码位于`tcg/i386/tcg-target.c.inc`文件的`tcg_out_qemu_ld`函数。

 

一条条来解释:

```
-- guest addr 0x00000004
//r12寄存器包含着要读取的地址的低位部分addrlo(这里要读取的地址为0x10)，赋值给edi,edi为x86平台函数调用的第一个参数寄存器
0x7ff9c0000119:  41 8b fc                 movl     %r12d, %edi
 
//地址 >> (TARGET_PAGE_BITS - CPU_TLB_ENTRY_BITS) = 5
0x7ff9c000011c:  c1 ef 05                 shrl     $5, %edi
 
//-0xf0为偏移量,rbp为CPUArchState,-0xf0分为两部计算，首先获取neg.tlb.f[IDX]在CPUArchState中的偏移，再获取CPUTLBDescFast结构中mask成员的偏移, 因此-0xf0就为CPUTLBDescFast结构中mask成员的偏移,因此这条指令等于是执行了一个函数叫tlb_index(CPUArchState *env, uintptr_t mmu_idx,target_ulong addr)
0x7ff9c000011f:  23 bd 10 ff ff ff        andl     -0xf0(%rbp), %edi
 
//-0xe8为CPUTLBDescFast结构中的table成员的偏移，因此这条指令等于是执行了一个函数叫tlb_entry(CPUArchState *env, uintptr_t mmu_idx,target_ulong addr)
0x7ff9c0000125:  48 03 bd 18 ff ff ff     addq     -0xe8(%rbp), %rdi
 
//addrlo + (s_mask - a_mask)赋值给%esi, esi为x86平台函数调用的第二个参数寄存器
0x7ff9c000012c:  41 8d 74 24 03           leal     3(%r12), %esi
 
//地址 & (TARGET_PAGE_MASK | a_mask)这样提取出地址的除了页偏移的其他部分
0x7ff9c0000131:  81 e6 00 fc ff ff        andl     $0xfffffc00, %esi
 
//0(%rdi)的值为对应CPUTLBEntry的addr_read成员变量的值，和要取的地址进行比较
0x7ff9c0000137:  3b 37                    cmpl     0(%rdi), %esi
 
//原始地址赋值给%esi
0x7ff9c0000139:  41 8b f4                 movl     %r12d, %esi
 
//如果CPUTLBEntry的addr_read成员变量的值和要取的地址不相等则表示tlb不命中，跳转至tlb慢路径地址0x7ff9c00001de处执行
0x7ff9c000013c:  0f 85 9c 00 00 00        jne      0x7ff9c00001de
 
//如果没有进入tlb慢路径表示tlb命中，0x10(%rdi)的值为CPUTLBEntry的addend成员变量的值，加上原始地址即为HVA
0x7ff9c0000142:  48 03 77 10              addq     0x10(%rdi), %rsi
 
//读取HVA地址处的值并赋值给%r12d
0x7ff9c0000146:  44 8b 26                 movl     0(%rsi), %r12d

```

生成 tlb slow path 的代码在 tcg/tcg.c 文件的 tcg_gen_code 函数中的:

```
/* Generate TB finalization at the end of block */
#ifdef TCG_TARGET_NEED_LDST_LABELS
    i = tcg_out_ldst_finalize(s);
    if (i < 0) {
        return i;
 
    }

```

tlb slow path 的代码位于每个 TB 的尾端:

```
-- tb slow paths + alignment
//准备好第一个参数tcg_target_call_iarg_regs[0]，它的值为CPUArchState env
0x7ff9c00001de:  48 8b fd                 movq     %rbp, %rdi
 
//准备好第三个参数tcg_target_call_iarg_regs[2]，它的值为TCGMemOpIdx oi = 0x22
0x7ff9c00001e1:  ba 22 00 00 00           movl     $0x22, %edx
 
//准备好第四个参数tcg_target_call_iarg_regs[3]，它的值为retaddr
0x7ff9c00001e6:  48 8d 0d 5c ff ff ff     leaq     -0xa4(%rip), %rcx
 
//调用函数helper_le_ldul_mmu
//helper_le_ldul_mmu(CPUArchState *env, target_ulong addr,TCGMemOpIdx oi, uintptr_t retaddr)
0x7ff9c00001ed:  ff 15 4d 00 00 00        callq    *0x4d(%rip)
 
//获取返回值
0x7ff9c00001f3:  44 8b e0                 movl     %eax, %r12d
 
//跳转回之前不命中的地方继续执行
0x7ff9c00001f6:  e9 4e ff ff ff           jmp      0x7ff9c0000149

```

`helper_le_ldul_mmu`还会再检测一次 tlb 是否命中，如果不命中将会调用体系结构相关函数做下一步的处理。

 

事实上会先访问快速路径，再访问 victim tlb, victim tlb 机制见:  
[https://patchwork.ozlabs.org/project/qemu-devel/patch/1390930309-21210-1-git-send-email-trent.tong@gmail.com/](https://patchwork.ozlabs.org/project/qemu-devel/patch/1390930309-21210-1-git-send-email-trent.tong@gmail.com/)

```
/* If the TLB entry is for a different page, reload and try again.  */
if (!tlb_hit(tlb_addr, addr)) {
    if (!victim_tlb_hit(env, mmu_idx, index, tlb_off,
                        addr & TARGET_PAGE_MASK)) {
        tlb_fill(env_cpu(env), addr, size,
                 access_type, mmu_idx, retaddr);
        index = tlb_index(env, mmu_idx, addr);
        entry = tlb_entry(env, mmu_idx, addr);
    }
    tlb_addr = code_read ? entry->addr_code : entry->addr_read;
    tlb_addr &= ~TLB_INVALID_MASK;
}

```

### 10. 分支指令 (条件跳转、非条件跳转、返回指令）

这条指令会间接的改变 pc 的值从而产生跳转 (从而终结当前 TB)

```
0x0000000c:  e59ff004  ldr      pc, [pc, #4]

```

一个 TB 终结以后怎么执行有两种可能:

1.  直接执行下一个 TB
2.  回到 qemu 上下文继续编译执行

它的 IR 如下:

```
---- 0000000c 00000000 00000000
 mov_i32 tmp3,$0x18  //0x18处为pc应该更新到的值即pc + 4
 mov_i32 tmp7,tmp3   
 qemu_ld_i32 tmp6,tmp7,leul,10  //将(pc + 4)内存地址处的值取出存放于tmp6
 and_i32 pc,tmp6,$0xfffffffe　　//这里的逻辑对应于target/arm/tcg/translate.c文件的gen_bx函数,注意SPC值发生了改变
 and_i32 tmp6,tmp6,$0x1        //同样位于gen_bx函数
 st_i32 tmp6,env,$0x220        //赋值给env中的thumb成员
 
 //这条IR产生的原因是上面的gen_bx函数中的语句: s->base.is_jmp = DISAS_JUMP,从而退出translator_loop中的while循环，调用ops->tb_stop(db, cpu)从而调用gen_goto_ptr()产生此条IR
 call lookup_tb_ptr,$0x6,$1,tmp12,env　
 goto_ptr tmp12

```

再来具体看一下如下两条 IR:

```
call lookup_tb_ptr,$0x6,$1,tmp12,env
goto_ptr tmp12

```

那么`call lookup_tb_ptr`后面的参数是什么含义? 具体可以参考`tcg/tcg.c`文件的`tcg_dump_ops`函数:

1.  lookup_tb_ptr 为 TCGOp 对象所对应的 TCGHelperInfo 对象的 name 字段
2.  $0x6 为 TCGOp 对象所对应的 TCGHelperInfo 对象的 flags 字段
3.  $1 为 TCGOp 对象的 param2 成员, 即 nb_oargs, 表示输出参数的个数为 1
4.  tmp12 为 op->args[] 中输出参数的字符串表示
5.  env 为 op->args[] 中输入参数的字符串表示

因此 lookup_tb_ptr 这个 helper 函数输入参数个数就是 1, 即为 CPUArchState env, 输出参数为 tmp12, 然后 goto_ptr tmp12 就跳转至此地址处从而终结当前 TB。

 

这段 IR 对应的 x86_64 target 代码为:

```
0x7f2d53e7e1aa:  48 8b fd                 movq     %rbp, %rdi  //rbp为CPUArchState env赋值给第一个参数寄存器%rdi
//调用%eip + 0x65处的函数,即(helper_lookup_tb_ptr函数)
0x7f2d53e7e1ad:  ff 15 65 00 00 00        callq    *0x65(%rip)
0x7f2d53e7e1b3:  ff e0                    jmpq     *%rax　//跳转至函数返回值处执行

```

lookup_tb_ptr 函数的功能属于 tcg 相关，因此它的实现位于`accel/tcg/cpu-exec.c`:

```
const void *HELPER(lookup_tb_ptr)(CPUArchState *env) //它的名字经扩展后为helper_lookup_tb_ptr

```

需要跳转的目标地址在 env 结构的 pc 成员中，这个函数中会通过 hash 表查询是否有目标 TranslationBlock 的缓存。如果有则跳转至 TranslationBlock.tc.ptr 执行即可 (即下一个)，如果没有则跳转至 tcg_code_gen_epilogue 执行。

 

`tcg_code_gen_epilogue`指向了 tcg 的 epilogue 处，因此如果跳转至`tcg_code_gen_epilogue`执行最终结果是`tcg_qemu_tb_exec(env, tb_ptr)`函数返回，从而回到了 qemu tcg 上下文处进行下一个 TB 的转换执行。

 

综上所述这种跳转指令会生成 helper 函数`lookup_tb_ptr`，它要么成功找到下一个 TB 的地址并跳转过去执行要么返回 qemu tcg 上下文执行。

 

再看一下另外一种跳转指令:

```
0x00010004:  eb000017  bl       #0x10068

```

bl 这条指令首先需要反汇编, 会进入到`libqemu-arm-softmmu.fa.p/decode-a32.c.inc`文件的`disas_a32_extract_branch`函数，a->imm 为 pc 相对跳转的偏移值。

 

然后需要转换成 IR, 对应的函数为`target/arm/tcg/translate.c`文件的 trans_BL 函数。  
转换以后对应的 IR 为:

```
add_i32 r14,pc,$0x8
add_i32 pc,pc,$0x68
goto_tb $0x0
 
 
//0x7f666c000280即val的值为当前的TranslationBlock在buf_rx处的指针:
//uintptr_t val = (uintptr_t)tcg_splitwx_to_rx((void *)tb) + idx;
exit_tb $0x7f666c000280

```

这种跳转和上面的跳转不同的是 goto_tb $0x0 以及 exit_tb

 

在这个 jmp_diff 函数中还可以看到对 arm 来说 pc 真正的值为当前执行指令在 arm 模式下 + 8, 在 thumb 模式下是 + 4:

```
static target_long jmp_diff(DisasContext *s, target_long diff)
{
    return diff + (s->thumb ? 4 : 8);
}

```

goto_tb $0x0 对应的目标代码如下:

```
//生成的代码只是用于跳转到下一条指令
0x7fff70000397:  e9 00 00 00 00           jmp      0x7fff7000039c

```

它由`tcg/i386/tcg-target.c.inc`文件中的`tcg_out_goto_tb`函数生成:

```
static void tcg_out_goto_tb(TCGContext *s, int which)
{
    /*
     * Jump displacement must be aligned for atomic patching;
     * see if we need to add extra nops before jump
     */
    int gap = QEMU_ALIGN_PTR_UP(s->code_ptr + 1, 4) - s->code_ptr;
    if (gap != 1) {
        tcg_out_nopn(s, gap - 1);
    }
    tcg_out8(s, OPC_JMP_long); /* jmp im */
    set_jmp_insn_offset(s, which);
    tcg_out32(s, 0);
    set_jmp_reset_offset(s, which);
}

```

这个函数除了生成 jmp 指令跳转到下一条指令外还执行了如下两个函数, 作用如下:

```
set_jmp_insn_offset(s, which); //设置当前TB的jmp_insn_offset[0]为tcg_current_code_size(s)
set_jmp_reset_offset(s, which);//设置当前TB的jmp_reset_offset[0]为tcg_current_code_size(s)

```

exit_tb 0x7f666c000280 对应的目标代码为:

```
//-0x123(%rip)的值就是0x7f666c000280，赋值给%rax
0x7fff7000039c:  48 8d 05 dd fe ff ff     leaq     -0x123(%rip), %rax
 
//0x7fff70000018的值就是tb_ret_addr，即TB epilogue
0x7fff700003a3:  e9 70 fc ff ff           jmp      0x7fff70000018

```

![](https://bbs.kanxue.com/upload/attach/202305/607812_AMCRWTK3NXDR7GK.png)

 

因此总结起来 goto_tb $0x0 和 exit_tb 0x7f666c000280 的作用是:

1.  设置当前 TB 的 jmp_insn_offset[0] 和 jmp_reset_offset[0]
2.  将当前 TB 在 buf_rx 处的指针 (0x7f666c000280) 赋值给 %rax
3.  跳转至 TB epilogue 处即从 tcg_qemu_tb_exec(env, tb_ptr) 函数处返回

从 tcg_qemu_tb_exec(env, tb_ptr) 函数处返回以后接着处理下一个 TB，因此当前 TB 就变成了 last_tb，返回到 cpu_exec_loop 函数中执行如下逻辑:

```
if (last_tb) {
   tb_add_jump(last_tb, tb_exit, tb);
}

```

最终 tb_add_jump 的逻辑为:

```
void tb_set_jmp_target(TranslationBlock *tb, int n, uintptr_t addr)
{
    /*
     * Get the rx view of the structure, from which we find the
     * executable code address, and tb_target_set_jmp_target can
     * produce a pc-relative displacement to jmp_target_addr[n].
     */
    const TranslationBlock *c_tb = tcg_splitwx_to_rx(tb);
    uintptr_t offset = tb->jmp_insn_offset[n];
    uintptr_t jmp_rx = (uintptr_t)tb->tc.ptr + offset;
    uintptr_t jmp_rw = jmp_rx - tcg_splitwx_diff;
    tb->jmp_target_addr[n] = addr;
    tb_target_set_jmp_target(c_tb, n, jmp_rx, jmp_rw);
}

```

用图来表示是这样的效果:  
![](https://bbs.kanxue.com/upload/attach/202305/607812_3XQ6QDQT9GTP4UV.png)

 

对内存的修改发生在 buf_rw 区域，而执行代码则位于 buf_rx 区域，两个区域是镜像关系。

 

先看 buf_rw 区域，需要对`Last TB CODE`中的`"e9 00 00 00 00"`进行 patch 让它指向`Next TB CODE`, 这样下次再执行`Last TB CODE`时`"e9 xx xx xx xx"`处将会直接跳转到`Next TB CODE`处执行，无需再退出至 qemu 上下文。这个就叫`Direct block chaining优化`。

 

上面看到`jmp_insn_offset[0]`指向的是需要 patch 的指令在 code 区域的偏移，而`jmp_reset_offset[0]`则指向了需要 patch 指令的下一条指令，当需要断开当前 TB 与下一条 TB 的`Direct block chaining`链接时，再执行 patch，目标是`jmp_reset_offset[0]`即可恢复当前 TB 的跳转。

 

`Direct block chaining`还需要解决的一个问题是自修改代码, 即当代码会对代码区域作修改时，这个代码区域之前旧的翻译指令不再有效，它和其他 TB 之间的链接也可能不再有效。

 

tcg 针对自修改代码也做了处理，结合着 soft mmu 的机制，当产生自修改代码时会调用`do_tb_phys_invalidate`函数从而重置 TB 的一些状态，其中就包括它所链接到其他 TB 的状态。

### 11. 目标机器没有的指令、特权指令、敏感指令

前面提到过，tcg 需要依赖于 Guest 的 helper 函数来模拟各种 Guest 的特殊指令, 每个体系结构对应的 helper 函数在`target/xxx/helper.h`头文件中声明。  
所有 helpers 的数组结构如下:

```
//所有helpers的数组
static const TCGHelperInfo all_helpers[] = {
#include "exec/helper-tcg.h"   //包含#include "helper.h"
};

```

比如对于 arm 的除法指令 udiv，在 x86_64 平台是没有的，最终调用 target/arm/helper.c 文件 udiv 函数:

```
uint32_t HELPER(udiv)(CPUARMState *env, uint32_t num, uint32_t den)
{
    if (den == 0) {
        handle_possible_div0_trap(env, GETPC()); //引发除0异常
        return 0;
    }
    return num / den;
}

```

比如 x86 中的 cpuid 指令, arm 平台是没有的，最终调用的函数为:

```
void helper_cpuid(CPUX86State *env)
{
    uint32_t eax, ebx, ecx, edx;
    cpu_svm_check_intercept_param(env, SVM_EXIT_CPUID, 0, GETPC());
    cpu_x86_cpuid(env, (uint32_t)env->regs[R_EAX], (uint32_t)env->regs[R_ECX],
                  &eax, &ebx, &ecx, &edx);
    env->regs[R_EAX] = eax;
    env->regs[R_EBX] = ebx;
    env->regs[R_ECX] = ecx;
    env->regs[R_EDX] = edx;
}

```

其他的特权指令敏感指令同理。

### 12. 非普通内存读写如设备寄存器访问 MMIO

某些内存地址指向的并不是 ram, 而是设备的寄存器，这种内存地址叫 MMIO，tcg 必须正确处理这种内存地址访问，当访问 MMIO 时将与模拟设备进行通信。

 

这一块涉及到 qemu 的内存模块 MemoryRegion 和 AddressSpace，是相当复杂的概念，限于篇幅不再描述, 只需要知道 tcg 会生成访问内存的 helper 函数如`helper_le_stl_mmu`, 然后进入到 qemu 的内存模块做下一步的处理。

### 13. 指令执行出现了同步异常如何处理 (如系统调用)

真实的 cpu 在执行的过程中会遇到异常和中断，既然是模拟 cpu,tcg 也需要处理好异常和中断，先以同步异常系统调用举例, 其实它是通过长跳转来直接跳出 tcg 执行循环的，如下面的指令

```
0x00010004:  ef000000  svc      #0

```

tcg 解析到这条指令的时候会进入到 trans_SVC 函数. 将 DisasContextBase 的 is_jmp 设置为 DISAS_SWI 表示当前 tb 的终结。

 

随后退出到 tb 循环中执行 arm_tr_tb_stop 函数进入 DISAS_SWI 分支:

```
case DISAS_SWI:
    gen_exception(EXCP_SWI, syn_aa32_svc(dc->svc_imm, dc->thumb));

```

生成对应的 IR 为:

```
add_i32 pc,pc,$0x8
call exception_with_syndrome,$0x8,$0,env,$0x2,$0x46000000

```

最终在执行目标代码时会调用的函数为:

```
//函数名称为helper_exception_with_syndrome
//excp的值为: #define EXCP_SWI  2
//syndrome的值为0x46000000,由syn_aa32_svc()函数计算得出，可以认为是常量
//target_el值为1表示执行系统调用会切换exception level至1
void HELPER(exception_with_syndrome)(CPUARMState *env, uint32_t excp,
                                     uint32_t syndrome, uint32_t target_el)
{
    raise_exception(env, excp, syndrome, target_el);
}
 
void raise_exception(CPUARMState *env, uint32_t excp,
                     uint32_t syndrome, uint32_t target_el)
{
    CPUState *cs = env_cpu(env);
    if (target_el == 1 && (arm_hcr_el2_eff(env) & HCR_TGE)) {
        /*
         * Redirect NS EL1 exceptions to NS EL2. These are reported with
         * their original syndrome register value, with the exception of
         * SIMD/FP access traps, which are reported as uncategorized
         * (see DDI0478C.a D1.10.4)
         */
        target_el = 2;
        if (syn_get_ec(syndrome) == EC_ADVSIMDFPACCESSTRAP) {
            syndrome = syn_uncategorized();
        }
    }
    assert(!excp_is_internal(excp));
    cs->exception_index = excp;          //更新CPU状态的异常下标
    env->exception.syndrome = syndrome;　
    env->exception.target_el = target_el;　
    cpu_loop_exit(cs);　　//请求退出执行循环
}

```

cpu_loop_exit 代码如下:

```
void cpu_loop_exit(CPUState *cpu)
{
    /* Undo the setting in cpu_tb_exec.  */
    cpu->can_do_io = 1;
    /* Undo any setting in generated code.  */
    qemu_plugin_disable_mem_helpers(cpu);
    siglongjmp(cpu->jmp_env, 1);  //执行长跳转退出至执行循环
}

```

最终会退出当前 cpu 的执行循环进入异常的处理过程:

```
cpu_exec_loop(CPUState *cpu, SyncClocks *sc)
{
    int ret;
    /* if an exception is pending, we execute it here */
    while (!cpu_handle_exception(cpu, &ret)) { //执行异常处理
        TranslationBlock *last_tb = NULL;
        int tb_exit = 0;
        while (!cpu_handle_interrupt(cpu, &last_tb)) {

```

cpu_handle_exception 函数通过调用`cc->tcg_ops->do_interrupt(cpu)`进入最终的异常处理, 从而调用到`target/arm/helper.c`文件中的函数:

```
void arm_cpu_do_interrupt(CPUState *cs) //逻辑是addr=8; addr += A32_BANKED_CURRENT_REG_GET(env, vbar);

```

也就是通过 vbar 寄存器`(Vector Base Address Register)`计算出异常处理函数的地址 newpc, 并且通过`take_aarch32_exception`函数将 pc 置为异常处理函数地址并跳转过去执行:

```
env->regs[15] = newpc;  //r15就是pc寄存器

```

因此可以看到对于系统调用来说始终没有跳出 tcg 线程，因为系统调用为同步异常。

### 14. 硬件中断如何处理

硬件中断属于异步事件，对于真实的 cpu 来说，它会在执行每条指令以后检查中断引脚的信号判断是否有外部中断产生。对于 tcg 来说显然粒度做不到这么细，因为这么做性能太低了，但又必须能够及时响应外部中断。

 

外部硬件中断发生在 IO 线程，发生硬件中断时会经由平台的模拟中断控制器一堆复杂的逻辑以后最终调用如下函数通知 vcpu:

```
void mttcg_kick_vcpu_thread(CPUState *cpu)
{
    cpu_exit(cpu);
}
 
void cpu_exit(CPUState *cpu)
{
    qatomic_set(&cpu->exit_request, 1);
    /* Ensure cpu_exec will see the exit request after TCG has exited.  */
    smp_wmb();
    //Set to -1 to force TCG to stop executing linked TBs for this CPU and return to its top level loop (even in non-icount mode).
    qatomic_set(&cpu->icount_decr_ptr->u16.high, -1);
}

```

当把`cpu->icount_decr_ptr->u16.high`置为 - 1 时就是告诉 tcg 线程中正在执行的 tb 尽快退出，回到 qemu 上下文进行外部中断的处理。icount_decr_ptr 还涉及到 qemu 的一个特性叫`TCG Instruction Counting`:  
[https://qemu.readthedocs.io/en/latest/devel/tcg-icount.html](https://qemu.readthedocs.io/en/latest/devel/tcg-icount.html)

 

为了让 TB 执行的时候可以快速响应退出的指令, tcg 在每个 TB 的开头和结尾生成了如下代码:

```
OP:
//开头
 ld_i32 loc3,env,$0xfffffffffffffff0  //对应于cpu->icount_decr_ptr->u16.high
 brcond_i32 loc3,$0x0,lt,$L0　//如果cpu->icount_decr_ptr->u16.high < 0则跳转至结尾处的$L0
...
//结尾
set_label $L0   
exit_tb $0x7f884c000043  //收到中断通知，退出执行循环

```

因此 tcg 对硬件中断的响应是以 TB 为粒度。  
退出执行循环以后会进入

```
while (!cpu_handle_interrupt(cpu, &last_tb)) {

```

处理中断，下面的代码就是和 qemu 硬件模拟相关的代码，不再细述。

三. unicorn 原理分析
---------------

unicorn 是基于 qemu tcg 的，但 qemu tcg 还是太复杂了，它模拟的是一个完整的系统，unicorn 只需要模拟执行一个可执行文件甚至一段代码片段，因此 unicorn 中的 tcg 可以说是轻量级的 tcg, 这也是 unicorn 被称为 cpu 模拟器的原因。

 

unicorn 的特点:

1.  只保留 qemu tcg cpu 模拟器的部分, 移除掉其他如 device,rom/bios 等和系统模拟相关的代码
2.  尽量维持 qemu cpu 模拟器部分不变，这样才容易和上游的 qemu tcg 代码同步
3.  重构 tcg 的代码从而可以更好的实现线程安全性及同时运行多个 unicorn 实例
4.  qemu tcg 并非一个 Instrumentation 框架，而 unicorn 的目标是实现一个有多种语言绑定的 Instrumentation 框架，可以在多个级别跟踪代码的运行并执行设置好的回调函数。

因此 unicorn 的研究重点就在于研究它如何提供在指令级别、内存访问级别的 dynamic Instrumentation。

 

unicorn 不仅实现了 cpu 模拟, 还需要实现内存模拟, 因此让 unicorn 能运行起来还需要设置内存映射。

 

unicorn 使用 qemu 的 tcg 做为 cpu 模拟的实现，使用`tlb/softmmu/MemoryRegion`做为内存模拟的实现。

 

设置内存映射以 c 语言的设置为例:

```
uc_mem_map(*uc, code_start, code_len, UC_PROT_ALL)

```

这段代码设置了 code_start 到 code_len 之间的区域为虚拟 cpu 所使用的虚拟地址空间，由于 unicorn 中并没有相应的操作系统代码来设置页表开启 mmu, 因此 unicorn 中的 mmu 并没有开启:

```
//返回true表示没有开启mmu
if (regime_translation_disabled(env, mmu_idx)) {

```

在 unicorn 中虚拟地址 (GVA) 就等于物理地址(GPA)。

 

调用 uc_mem_map 函数设置内存映射，本质是创建出一个 MemoryRegion 对象，初始化这个对象并添加到 system_memory 这个全局的 MemoryRegion 树层次结构中，这里涉及到 qemu 中的内存对象`AddressSpace,MemoryRegion,FlatView和RAMBlock`。这块的机制相当复杂，描述它就得需要一两篇博客的篇幅，这里只是简单介绍一下概念:

1.  AddressSpace 是全局的内存视图，它被每个体系结构的 cpu 结构引用, 它的成员 MemoryRegion *root 引用根 MemoryRegion 所表示的 MemoryRegion 树
2.  多个 MemoryRegion 组成了一个树结构，MemoryRegion 它可以表示 ram,rom,MMIO 等多种类型的内存设备，可以认为 MemoryRegion 是 Guest 物理内存设备的表示
3.  MemoryRegion 如果对应着 ram 内存设备，它的 RAMBlock ram_block 成员就表示 Host 一侧分配出来的虚拟地址空间, 所有的 RAMBlock 被存放在 RAMList 表示的列表中
4.  MemoryRegion 是树结构，它的平坦化线性模型由 FlatView 对象表示

设置好内存映射，初始化相应的数据结构以后，unicorn 就可以设置寄存器，设置好 hook 回调并且启动 unicorn 引擎:

```
uc_hook_add(uc, &hook, UC_HOOK_CODE, my_callback,&count, 1, 0)
uc_reg_write(uc, UC_ARM_REG_R0, &r_r0)
uc_reg_write(uc, UC_ARM_REG_R2, &r_r2)
uc_emu_start(uc, code_start, code_start + sizeof(code) - 1, 0, 0)

```

### 1.UC_HOOK_CODE:

unicorn 的 Hook 类型有很多种，一个一个来看, 首先是 UC_HOOK_CODE 类型, 它可以设置回调在每条指令执行前调用。

 

当调用了`uc_hook_add`函数以后，其实是创建出了`struct hook`对象并添加到了 uc_struct 这个全局对象的`struct list hook[UC_HOOK_MAX]`链表中去，unicorn 其实是在 IR 层添加了相应的代码来设置回调, 比如对于

```
mov r0, #1

```

这么一条简单的指令，如果设置了`uc_hook_add(uc, &hook, UC_HOOK_CODE, my_callback,&count, 1, 0)`, 调试打印 OPCode:

```
UNICORN_DEBUG=1 ./test_arm my_hook_test

```

打印出的 IR 是这样的:

```
insn_idx=0 ---- 00001000 00000000 00000000
 1:  movi_i32 pc,$0x1000
 2:  movi_i32 tmp3,$0x4
 3:  movi_i64 tmp5,$0x55e38ad72840
 4:  movi_i64 tmp6,$0x1000
 5:  movi_i64 tmp7,$0x7fff1dd92190
 6:  call hookcode_4_55e38928e9a9,$0x0,$0,tmp5,tmp6,tmp3,tmp7
 7:  ld_i32 tmp3,env,$0xfffffffffffffff0
 8:  movi_i32 tmp4,$0x0
 9:  brcond_i32 tmp3,tmp4,lt,$L0
 10:  movi_i32 tmp3,$0x1
 11:  mov_i32 r0,tmp3

```

第 1 条到第 6 条是 unicorn 添加的用于 hook 的 IR, 产生这些 IR 的代码位于反编译 Guest 指令之前，对于 arm 来说就是 disas_arm_insn 函数中:

```
// Unicorn: trace this instruction on request
  if (HOOK_EXISTS_BOUNDED(s->uc, UC_HOOK_CODE, s->pc_curr)) {
      // Sync PC in advance
      gen_set_pc_im(s, s->pc_curr);
 
      gen_uc_tracecode(tcg_ctx, 4, UC_HOOK_CODE_IDX, s->uc, s->pc_curr);
      // the callback might want to stop emulation immediately
      check_exit_request(tcg_ctx);
  }

```

`gen_uc_tracecode`的逻辑就是创建出调用`hookcode_4_55e38928e9a9`这个 helper 函数的 IR, 这个 helper 函数的实现就是设置进来的回调函数，它是动态被创建出来并且添加到 helper 函数的 hashtable 中的。

 

代码解读如下:

```
//将当前正在执行指令的地址放置于pc寄存器
 1:  movi_i32 pc,$0x1000
 
//4的值为跟踪的指令字节个数
 2:  movi_i32 tmp3,$0x4
 
//$0x55e38ad72840为uc_struct uc指令的值
 3:  movi_i64 tmp5,$0x55e38ad72840
 
//0x1000为当前pc执行的地址
 4:  movi_i64 tmp6,$0x1000
 
//$0x7fff1dd92190的值为设置回调时传递的user_data指针值
 5:  movi_i64 tmp7,$0x7fff1dd92190
 
//调用到回调函数 : void my_callback(uc_engine *uc, uint64_t address,　uint32_t size, void *user_data)
 6:  call hookcode_4_55e38928e9a9,$0x0,$0,tmp5,tmp6,tmp3,tmp7

```

### 2.UC_HOOK_INSN:

UC_HOOK_INSN 回调和 UC_HOOK_CODE 的回调原理差不多，只不过它可以用于跟踪某些特定指定的执行。比如当追踪 x86 的 inb 指令时，当执行到 cpu_inb 这个 helper 函数时就会调用回调:

```
uint8_t cpu_inb(struct uc_struct *uc, uint32_t addr)
{
    // uint8_t val;
    // address_space_read(&uc->address_space_io, addr, MEMTXATTRS_UNSPECIFIED,
    //                    &val, 1);
    //LOG_IOPORT("inb : %04"FMT_pioaddr" %02"PRIx8"\n", addr, val);
    // Unicorn: call registered IN callbacks
    struct hook *hook;
    HOOK_FOREACH_VAR_DECLARE;
    HOOK_FOREACH(uc, hook, UC_HOOK_INSN) {
        if (hook->to_delete)
            continue;
        if (hook->insn == UC_X86_INS_IN)
            return ((uc_cb_insn_in_t)hook->callback)(uc, addr, 1, hook->user_data);
    }
    return 0;
}

```

### 3.UC_HOOK_BLOCK

`UC_HOOK_BLOCK`用于跟踪 basic block 的执行，因此最好的跟踪点是一个 basic block 在处理之前:`qemu/accel/tcg/translator.c`的 translator_loop 函数 for 循环开始处:

```
if (HOOK_EXISTS_BOUNDED(uc, UC_HOOK_BLOCK, tb->pc)) {
    prev_op = tcg_last_op(tcg_ctx);
    block_hook = true;
    gen_uc_tracecode(tcg_ctx, 0xf8f8f8f8, UC_HOOK_BLOCK_IDX, uc, db->pc_first);
}

```

用于生成 IR 的函数仍然是 gen_uc_tracecode 函数

### 4.UC_HOOK_MEM_ 开头的

以 UC_HOOK_MEM_ 开头跟踪的是没有映射的内存读写执行、正常的内存读写执行以及和权限相关的内存读写执行。

 

前面在分析 qemu tcg 的时候我们可以看到内存读写会先检查 tlb 是否命中，如果不命中则会调用 tlb helper 函数走 mmu 并查找相应的 HVA 这条路，和内存加载相关的 helper 函数是 load_helper(), 和内存存储相关的 helper 函数是 store_helper()。

 

那么在这两个函数中添加代码去调用 unicorn 回调函数不就可以达到跟踪内存访问的目的了吗?

 

unicorn 也正是这么做的，不过如果 tlb 命中了就不会调用 tlb helper 函数怎么办？unicorn 的解决方案是判断如果有 hook mem 的回调函数，强制让流程执行 tlb slow path:

```
// Unicorn: fast path if hookmem is not enable
  if (!HOOK_EXISTS(s->uc, UC_HOOK_MEM_READ) && !HOOK_EXISTS(s->uc, UC_HOOK_MEM_WRITE))
      //没有回调时走之前的逻辑
      tcg_out_opc(s, OPC_JCC_long + JCC_JNE, 0, 0, 0);
  else
      /* slow_path, so data access will go via load_helper() */
      tcg_out_opc(s, OPC_JMP_long, 0, 0, 0);

```

这样一改所有内存读写都会走`load_helper()`和`store_helper()`。

 

unicorn 为了快速判断哪些内存访问属于`"UNMAPPED"`访问，在 uc_struct 结构中的`MemoryRegion **mapped_blocks`成员中存放当前所有设置过内存映射的`MemoryRegion`区域，给定一个地址，调用如下函数可以快速得到对应的 MemoryRegion:

```
MemoryRegion *memory_mapping(struct uc_struct *uc, uint64_t address)

```

如果获取到的对象为 null 则表示该内存访问属于 "UNMAPPED" 访问，从而调用相应的`UC_HOOK_MEM_WRITE_UNMAPPED`等回调。

### 5.UC_HOOK_INTR

由于 unicorn 没有设备模拟的功能，因此 UC_HOOK_INTR 无法监听硬件中断，只能监听同步异常如系统调用，前面在分析 tcg 功能的时候提到 cpu_handle_exception 函数是 tcg 处理同步异常的地方，unicorn 也是在这里处理 UC_HOOK_INTR 回调的:

```
HOOK_FOREACH(uc, hook, UC_HOOK_INTR) {
           if (hook->to_delete) {
               continue;
           }
       　　//cpu->exception_index即是中断号
           ((uc_cb_hookintr_t)hook->callback)(uc, cpu->exception_index, hook->user_data);
           catched = true;
       }

```

四. 总结:
------

理解了 qemu tcg 的机制以后再去理解 unicorn 的功能就比较容易了，限于篇幅还有一些内容没有描述清楚，欢迎大家一起交流、讨论。

[IDA 插件开发入门](https://www.kanxue.com/book-section_list-108.htm)

[#漏洞分析](forum-150-1-153.htm) [#Andorid](forum-150-1-162.htm)