> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288389.htm)

> [原创]qemu v0.4.4 代码情景分析

为什么使用这个版本
=========

qemu 现在已经很庞大了，直接分析最新代码难度太大，所以选择从低版本开始分析。

qemu 关键特性
=========

分析的 qemu 是指在 Linux 和 x86 上运行的 qemu，qemu 载入 Linux 下的 ELF 可执行文件，然后仿真运行这个文件。  
本文关注的问题是：

*   ELF 文件中的代码是如何运行的？
*   中断是如何模拟的？

qemu 流程
=======

*   载入 ELF 文件到内存中，此时可以从内存中获得可执行原始代码
*   编译原始代码到中间代码，需要注意的是一条原始代码可能会被翻译为多条中间代码
*   根据中间代码生成可在 x86 的 Linux 上运行的二进制代码

原始代码到中间代码
=========

关键函数是

```
disas_insn

```

这个函数每次将一条 ELF 文件中的指令转换为中间代码，需要注意的是在转换时不会一次性将所有 ELF 文件中的指令转换，当遇到 JMP 指令时就停止转换，当转换的指令太多超过大小限制时也会停止转换，转换后的代码块包含多条原始代码的对应代码。

中间代码到可在 x86 的 Linux 上运行的二进制代码
=============================

关键函数是

```
dyngen_code

```

这个函数的做法是取编译器编译的函数的一部分代码到指令缓存中，比如对于中间代码

```
INDEX_op_movl_A0_EAX

```

会将对应的函数

```
op_movl_A0_EAX

```

中的前 3 个字节复制到指令缓存中，这个函数如下

```
void OPPROTO glue(op_movl_A0,REGNAME)(void)
{
    A0 = REG;
}

```

这是一个宏，当 REGNAME 和 REG 为 EAX 时，该函数变为

```
void OPPROTO op_movl_A0_EAX(void)
{
    A0 = EAX;
}

```

该函数完成的功能是将寄存器 EAX 中的数据拷贝到中间代码寄存器 A0 中。这个函数会被编译器编译为二进制代码，因此取该函数的前 3 个字节到指令缓存中，然后执行指令缓存中的代码便完成了对原有 ELF 代码的执行。

注意
--

qemu 第一次执行一条原始代码翻译后的二进制代码时会翻译该条原始代码及其之后的若干条代码（翻译指原始代码到中间代码和中间代码到可在 x86 的 Linux 上运行的二进制代码两个步骤），当下次运行到以该原始代码为第一条代码时会直接执行翻译后的对应代码块而不会去再次翻译，这样提高了性能。

解释完了指令的执行过程，再来介绍中断的模拟。

时钟中断的模拟：
========

对于模拟操作系统这样的软件而言，时钟中断是必不可少的，操作系统借助时钟中断切换任务。  
对于 qemu，模拟时钟中断的关键是利用宿主 Linux 中的时钟事件，当 Linux 中发出时钟事件时会调用 qemu 中的对应函数

```
host_alarm_handler

```

在该函数中会设置

```
timer_irq_pending = 1;
cpu_x86_interrupt(global_env, CPU_INTERRUPT_EXIT);

```

这样系统就知道发生了时钟中断，由于设置了 CPU_INTERRUPT_EXIT，所以在 cpu_exec 函数中

```
if (interrupt_request & CPU_INTERRUPT_EXIT) {
    env->interrupt_request &= ~CPU_INTERRUPT_EXIT;
    env->exception_index = EXCP_INTERRUPT;
    cpu_loop_exit();
}

```

这段代码中需要关注的是 EXCP_INTERRUPT，由于设置了这里，在随后的

```
if (env->exception_index >= EXCP_INTERRUPT) {
    /* exit request from the cpu execution loop */
    ret = env->exception_index;
    break;
}

```

这里的 break 语句会导致退出 cpu_exec 函数，然后会运行的代码是

```
if (timer_irq_pending) {
    pic_set_irq(0, 1);
    pic_set_irq(0, 0);
    timer_irq_pending = 0;
}

```

由于在 host_alarm_handler 函数中设置了 timer_irq_pending，所以这里代码会执行，pic_set_irq 中产生了定时器中断，该函数中会调用

```
cpu_x86_interrupt(global_env, CPU_INTERRUPT_HARD);

```

该函数设置了中断标志变量

```
interrupt_request

```

然后程序会继续调用 cpu_x86_exec 函数去翻译和执行，但和平时不同，由于设置了 interrupt_request，说明此时有中断产生了，所以会调用

```
do_interrupt

```

去执行中断处理函数，中断就这样被模拟出来了。

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)