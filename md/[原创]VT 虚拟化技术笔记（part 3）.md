> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271928.htm)

> [原创]VT 虚拟化技术笔记（part 3）

目录

*   VT 技术笔记（part 3）
*   [vm-control 相应字段的设置](#vm-control相应字段的设置)
*            vm-entry control 字段
*                    [VM_ENTRY_CONTROLS 字段](#vm_entry_controls字段)
*                    [MSR-load 字段](#msr-load字段)
*                    [VM_ENTRY_INTR_INFO_FIELD 字段](#vm_entry_intr_info_field字段)
*                    vm-entry control 字段填写总结
*            vm-exit control 字段
*            vm-execution control 字段
*                    pin-based vm-execution control 字段
*                    processor-based vm-execution control 字段
*   [对 vm-exit 处理函数进行框架搭建](#对vm-exit处理函数进行框架搭建)
*   [参考文献](#参考文献)

VT 技术笔记（part 3）
===============

*   最近在学习 VT 技术，想把学习过程中记录的笔记分享出来。技术不精，有不对的地方还望指正。代码会发到 https://github.com/smallzhong/myvt 这个仓库中，目前还在施工中，还没写完。欢迎 star，预计 4 月份完工（最近有点事，有个小测验要准备，可能得拖更一段时间）。
*   本篇文章讲解 vm-control 相应字段的设置与填写以及 vm-exit 事件处理函数的框架搭建。

vm-control 相应字段的设置
==================

vm-entry control 字段
-------------------

*   在《处理器虚拟化技术》3.6 章节中详细讲解了 vm-entry 控制类字段的填写与对应的属性。需要填写的字段如下
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220316101031615.png)
    
    在 vm-entry 时，如果 CPU 检查到这些字段没有被正确填写，将会抛错并退出。
    

### VM_ENTRY_CONTROLS 字段

*   这个字段的长度为 32 位，每个位对应一个控制功能。其控制的是进入虚拟机时处理器所进行的一些操作。如是否在进入虚拟机时加载 dr0~dr7 寄存器、是否加载时进入 IA-32e 模式、是否加载 IA32_PERF_GLOBAL_CTRL、IA32_PAT、IA32_EFER 寄存器等。具体位的作用如书中表 3-9 所示。这本书成书时间比较早，可能 CPU 已经添加了一些别的字段，具体可以参考 intel 白皮书中相关章节。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220316101508220.png)
    
    在查看这个表后可以注意到，有一些位置被固定为了 1，有一些位置被固定为了 0。这些位置有些是还未被使用的，用于将来拓展其他功能的时候添加的。这些位可能在将来将不再固定为 1 或 0，而是用于控制某个新推出的功能。因此不能把固定为 0 或者固定为 1 的位写死填进去。这里需要根据一个算法来算出固定为 0 和固定为 1 的位，并填入 `VM_ENTRY_CONTROLS` 寄存器中。算法具体内容如下
    
    *   首先读取 `IA32_VMX_BASIC` 寄存器。判断其第 55 位。如果为 1，则之后的操作都使用下表右侧带 “TRUE” 的寄存器，如果为 0，则之后的操作都使用左边不带“TRUE“的寄存器。实际使用中发现现在的电脑很多都是使用了右边带”TRUE“的寄存器。因此之后描述中均使用带 TRUE 的寄存器进行。但为了兼容性还是需要每次进行判断应使用哪一组寄存器。
        
        ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220316104226856.png)
        
    *   之后对固定位进行设置的方法在书中 2.5.5 章节有详细描述。以下进行简略介绍。 `IA32_MSR_VMX_TRUE_ENTRY_CTLS` 这个寄存器是一个 64 位的寄存器，需要被设置的 `VM_ENTRY_CONTROLS` 是一个 32 位的寄存器。 `IA32_MSR_VMX_TRUE_ENTRY_CTLS` 对 `VM_ENTRY_CONTROLS` 的控制大体如下图所示。
        
        ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220316104819075.png)
        
        可见，当 `IA32_MSR_VMX_TRUE_ENTRY_CTLS` 低 32 位某一位为 1 时， `VM_ENTRY_CONTROLS` 寄存器中对应位必须为 1。 `IA32_MSR_VMX_TRUE_ENTRY_CTLS` 高 32 位中某一位为 0 时， `VM_ENTRY_CONTROLS` 寄存器中对应位必须为 0。可以大致总结出如下的伪代码
        

```
VM_ENTRY_CONTROLS = 想要设置的位 | 低32位 & 高32位

```

*   在刚开始进行框架的搭建的时候，并不需要处理其他的字段。自定义的字段中只需要填入第 9 位进入 IA-32e 模式即可。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220316110701936.png)
    
    其他的位在刚开始的时候可以不进行设置。但并不代表这些位不重要。如第 2 位规定是否在进入虚拟机时加载当前 dr 寄存器。对这个功能的合理利用可能可以实现一些特殊的调试功能。这里不展开说。那么我们首先只填入第 9 位以及其他保留位。设置保留位的代码如下
    
    ```
    ULONG64 VmxAdjustContorls(ULONG64 value, ULONG64 msr)
    {
        LARGE_INTEGER msrValue;
        msrValue.QuadPart = __readmsr(msr);
        value = (msrValue.LowPart | value ) & msrValue.HighPart;
     
        return value;
    }
    
    ```
    
    填写 `VM_ENTRY_CONTROLS` 的代码如下
    
    ```
    ULONG64 vmxBasic = __readmsr(IA32_VMX_BASIC);
    ULONG64 mseregister = ( (vmxBasic >> 55) & 1 ) ? IA32_MSR_VMX_TRUE_ENTRY_CTLS  :IA32_VMX_ENTRY_CTLS;
     
    ULONG64 value = VmxAdjustContorls(0x200,mseregister);
    __vmx_vmwrite(VM_ENTRY_CONTROLS, value);
    
    ```
    
    可以看到这里自定义值只填入了 0x200，只设置了第 9 位 IA-32e mode。
    

### MSR-load 字段

*   ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220317142040884.png)
    
    在《处理器虚拟化技术》3.6.2 章节中描述了 MSR-load 字段。这两个字段用来控制进入虚拟机的时候是否需要加载 msr 寄存器。这里我们并不需要其在进入虚拟机的时候加载 msr 寄存器。因为 vm 的退出和加载是很频繁的。如果每次都加载一遍 msr 寄存器会降低性能。而且如果我们想要对 msr 寄存器进行拦截或者 hook，还有另外的方法可以进行。因此这两个字段均填为 0 即可。
    

### VM_ENTRY_INTR_INFO_FIELD 字段

*   在《处理器虚拟化技术》3.6.3.1 章节中有对该字段的解析。大致作用是根据一定的规则进行这个字段的填写之后，在进入虚拟机后相应的中断或者异常会被触发。这里我们暂时不需要用到这个功能。可以看到如果将最高位设置为 0，这个字段就被看作 invalid。因此直接将这个字段填充为 0 即可。以后需要用到这个功能的时候再对齐进行相应的处理。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220317174606240.png)
    

### vm-entry control 字段填写总结

*   vm-entry control 用来控制进入虚拟机时候的操作。并不算复杂。具体填充代码如下
    
    ```
    ULONG64 VmxAdjustContorls(ULONG64 value, ULONG64 msr)
    {
        LARGE_INTEGER msrValue;
        msrValue.QuadPart = __readmsr(msr);
        value = (msrValue.LowPart | value ) & msrValue.HighPart;
     
        return value;
    }
     
    void VmxInitEntry()
    {
        ULONG64 vmxBasic = __readmsr(IA32_VMX_BASIC);
        ULONG64 mseregister = ( (vmxBasic >> 55) & 1 ) ? IA32_MSR_VMX_TRUE_ENTRY_CTLS  :IA32_VMX_ENTRY_CTLS;
     
        ULONG64 value = VmxAdjustContorls(0x200,mseregister);
        __vmx_vmwrite(VM_ENTRY_CONTROLS, value);
        __vmx_vmwrite(VM_ENTRY_MSR_LOAD_COUNT, 0);
        __vmx_vmwrite(VM_ENTRY_INTR_INFO_FIELD, 0);
    }
    
    ```
    

vm-exit control 字段
------------------

*   vm-exit control 字段和 vm-entry 字段很类似。用来规定退出虚拟机时需要进行的操作。vm-entry 时候进行的操作和 vm-exit 时候进行的操作可以对应起来。vm-exit 的时候对 msr 寄存器进行保存，那么 vm-entry 的时候就可以加载 msr 寄存器。在填写字段以及进行功能的设置的时候应注意要将进入虚拟机时的操作和退出虚拟机时的操作对应起来。
    
*   在《处理器虚拟化技术》3.7 章节中描述了 vm-exit 字段的填写规则。大体和 vm-entry 填写规则相对应，这里不再赘述。注意两个点，第 15 位 acknowledge interrupt on exit 规定了是否在由于外部中断导致退出的时候读取并保存中断向量号。这里可以填 0 或 1 都不影响使用，但是为了能够在以后的时候用到这个保存的信息，可以将其填为 1，并不会影响性能。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220317182427238.png)
    
    第二点是第 22 位，有一个类似定时器的装置。但是很多 CPU 并不支持这个功能。如果为了兼容性建议不要使用这个功能。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220317182534128.png)
    

vm-execution control 字段
-----------------------

*   这是处理操作中最重要的字段，用来设置拦截哪些事件不拦截哪些事件。我们想要对虚拟机中的一些事件进行监控就要对这个字段进行相应的设置。对于 vm-execution 控制类字段，在《处理器虚拟化技术》3.5 章节中有详细的解析。

### pin-based vm-execution control 字段

*   这个字段用于对外部中断和 NMI 的处理进行相关的配置。在有需要对外部中断进行监控的时候可以使用。但是我们的目的是对一些软件的行为进行监控，并不需要对外部中断进行监控，因此这里把这个字段的可选位均填充为 0，都不进行拦截即可。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220317200829803.png)
    

### processor-based vm-execution control 字段

*   这个字段用来对一些软件的行为进行拦截。对一些特定的指令、一些特定寄存器的读写进行拦截。因为我们的目的是通过 VT 技术对一些行为进行监控，因此这个字段是我们需要重点关注的部分。详细解析可以查看《处理器虚拟化技术》的 3.5.1 章节。
    
*   这个字段第 31 位如果置为 1，可以激活另一个 `SECONDARY_VM_EXEC_CONTROL` 寄存器。这个寄存器可以设置更多拦截的操作。
    
    ![](https://cdn.jsdelivr.net/gh/smallzhong/new_new_picgo_picbed@main/image-20220317203750490.png)
    
*   本篇文章中暂时不对这些操作进行任何拦截，在之后的文章中有需要时再进行相应的拦截。
    
    ```
    void VmxInitControls()
    {
        ULONG64 vmxBasic = __readmsr(IA32_VMX_BASIC);
     
        ULONG64 mseregister = ((vmxBasic >> 55) & 1) ? IA32_MSR_VMX_TRUE_PINBASED_CTLS : IA32_MSR_VMX_PINBASED_CTLS;
     
        ULONG64 value = VmxAdjustContorls(0, mseregister);
     
        __vmx_vmwrite(PIN_BASED_VM_EXEC_CONTROL, value);
     
        mseregister = ((vmxBasic >> 55) & 1) ? IA32_MSR_VMX_TRUE_PROCBASED_CTLS : IA32_MSR_VMX_PROCBASED_CTLS;
     
        value = VmxAdjustContorls(0, mseregister);
     
        __vmx_vmwrite(CPU_BASED_VM_EXEC_CONTROL, value);
     
        /*
        //扩展部分
        mseregister = IA32_MSR_VMX_PROCBASED_CTLS2;
     
        value = VmxAdjustContorls(0, mseregister);
     
        __vmx_vmwrite(SECONDARY_VM_EXEC_CONTROL, value);
        */
    }
    
    ```
    

对 vm-exit 处理函数进行框架搭建
====================

*   在之前 vmcs 的填写过程中，vmexit 事件发生之后回到 host 中之后的 rip 被设置成了 vm-exit 处理函数的地址。在 vmexit 事件发生回到 host 之后自动从这个处理函数开始跑。这个函数还未被实现。这里将其简略实现一下，为下一篇文章的 vmexit 处理打好基础。
    
*   首先这个函数开始一定要保存所有的寄存器，并在返回虚拟机之前恢复所有的寄存器。否则退出虚拟机之前寄存器中的内容和返回虚拟机之后寄存器中的内容不一样的话一定会导致不可预知的结果。因此这个函数一定得是汇编写的裸函数。在将寄存器进行相应的把保存之后跳到 C 语言写的函数中继续执行。在 C 语言写的函数中完成对 vmexit 消息的处理并返回之后，再恢复之前保存的所有寄存器，然后通过 `vmresume` 指令恢复虚拟机的执行，重新跳回到虚拟机之中。具体汇编实现如下
    
    ```
    AsmVmxExitHandler proc
        push r15;
        push r14;
        push r13;
        push r12;
        push r11;
        push r10;
        push r9;
        push r8;
        push rdi;
        push rsi;
        push rbp;
        push rsp;
        push rbx;
        push rdx;
        push rcx;
        push rax;
     
        mov rcx,rsp;
        sub rsp,0100h
        call VmxExitHandler
        add rsp,0100h;
     
        pop rax;
        pop rcx;
        pop rdx;
        pop rbx;
        pop rsp;
        pop rbp;
        pop rsi;
        pop rdi;
        pop r8;
        pop r9;
        pop r10;
        pop r11;
        pop r12;
        pop r13;
        pop r14;
        pop r15;
        vmresume
        ret
    AsmVmxExitHandler endp
    
    ```
    
    至于 vmxExitHandler 这个 C 语言的函数中应该如何进行 vmexit 事件的处理以及应该如何通过设置 processor-based vm-execution control 字段对特定事件进行拦截，将会在下一篇文章中会进行详细的讲解。
    
*   **本篇文章对应的代码晚些时候会传到 github 仓库中。**

参考文献
====

1.  intel 白皮书
2.  邓志《处理器虚拟化技术》
3.  B 站周壑 VT 教学视频
4.  https://github.com/qq1045551070/VtToMe
5.  火哥上课讲的内容（这里帮火哥打个广告，火哥 qq471194425，上课会讲很多干货，也会在讲解技术的时候同时讲解一些运用的方法，报火哥的班绝对是物超所值）

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 54 分钟前 被 smallzhong_编辑 ，原因：

[#HOOK / 注入](forum-41-1-133.htm) [#驱动开发](forum-41-1-132.htm) [#系统内核](forum-41-1-131.htm) [#开源分享](forum-41-1-134.htm) [#虚拟化](forum-41-1-136.htm)