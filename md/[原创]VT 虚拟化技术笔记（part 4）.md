> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271967.htm)

> [原创]VT 虚拟化技术笔记（part 4）

目录

*   VT 虚拟化技术笔记（part 4）
*   [无条件退出的虚拟机事件](#无条件退出的虚拟机事件)
*            [大体框架](#大体框架)
*            [处理 vmx 指令导致的 vm-exit 事件](#处理vmx指令导致的vm-exit事件)
*            [处理 cpuid 导致的 vm-exit 事件](#处理cpuid导致的vm-exit事件)
*            处理 getsec invd xsetbv 导致的 vm-exit 事件
*            [guest 与 host 的通信 && 关闭 VT](#guest与host的通信&&关闭vt)
*   [有条件退出的虚拟机事件](#有条件退出的虚拟机事件)
*            [对 msr 寄存器的有条件拦截](#对msr寄存器的有条件拦截)
*            为了支持 win10 拦截 rdtscp invpcid xsaves/xsrstors
*                    [处理 rdtscp 指令导致的有条件退出](#处理rdtscp指令导致的有条件退出)
*                    [处理 invpcid 指令导致的有条件退出](#处理invpcid指令导致的有条件退出)
*                    [处理 xsaves 指令导致的有条件退出](#处理xsaves指令导致的有条件退出)
*   [参考文献](#参考文献)

VT 虚拟化技术笔记（part 4）
==================

*   最近在学习 VT 技术，想把学习过程中记录的笔记分享出来。技术不精，有不对的地方还望指正。代码会发到 https://github.com/smallzhong/myvt 这个仓库中，目前还在施工中，还没写完。欢迎 star，预计 4 月份完工（最近有点事，有个小测验要准备，可能得拖更一段时间）。
*   本篇文章讲解在 vm-exit 处理函数中对无条件退出的虚拟机事件的处理、对 MSR 读写事件的拦截以及对在 WIN10 下必须设置的有条件退出事件的处理。

无条件退出的虚拟机事件
===========

大体框架
----

*   在《处理器虚拟化技术》3.10.1.2 中，描述了 vm-exit 产生的原因。其中指出了会无条件导致 vmexit 事件产生的指令，如下图所示
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_AYAVJZ4VC5JXQJU.jpg)
    
    可见，在虚拟机中，执行所有除 vmfunc 之外的指令时，都会无条件导致 vmexit 事件的发生。除此之外， `cpuid` 、 `getsec` 、 `invd` 、 `xsetbv` 指令也会无条件导致 vmexit 事件的发生。那么我们需要进行处理的无条件退出事件如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_892CZTTAVGKHXSN.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_G3T2EWZJF79TXUJ.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_K3PPQFQN82H75MR.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_2AYCXTEG8KYR6PB.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_5FGDC5859R5YU76.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_7FHQ9QE253RDXKT.jpg)
    
*   在 3.4.3 32 位字段 ID 中，可以找到控制区中对应的 vmexit 信息字段。
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_UP5QYVXB6KM8SHX.jpg)
    
    可以找到退出原因、导致退出的指令的长度、信息。
    
    *   只读字段里面的 vm-instuction error 字段在 vm 指令失败的时候会被设置，可以从中读取出 fail 事件发生的原因。在《处理器虚拟化技术》2.6.3 章节中有 vmfailvalid 事件发生的原因的编号表。
    *   `exit-reason` 表示导致 vm-exit 事件发生的原因
    *   `vm-exit instruction length` 表示导致 vm-exit 事件发生的指令的长度。通过这个字段可以在进行进一步处理的时候更加方便。在我们对产生 vmexit 的事件进行处理之后，我们不可能重新跳回发生 vmexit 事件的地方继续执行。比如 vmexit 事件是由于 cpuid 指令导致的，那么在对这个事件进行处理之后，就要跳过原来的 cpuid，继续执行下一条指令，而不是回去之后仍然执行 cpuid 对应的那条指令。当然，也不是所有产生 vmexit 事件的时候都需要跳过原来执行的指令。比如缺页之类的原因造成的 vmexit 事件，就不能跳过产生 vmexit 事件的指令。不过大多事件都需要跳过执行的指令。
    *   `vm-exit instruction information` 这个字段在以后会用到，其存储的是指令详细信息，以后用到的时候再进行说明。
*   在《处理器虚拟化技术》3.10.1.1 中，列出了 exit reason 字段的组成。如下图
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_837HNZ46JWKXJFV.jpg)
    
    可见，0~15 位表明了 vm 退出的原因，其他位还有其他的指示作用。因此这里应该将其他位取出，只通过 0~15 位进行 vm 退出原因的判断。
    
*   综上所述，可以搭建出一个 vmexit 事件处理函数的框架如下
    

```
EXTERN_C VOID VmxExitHandler(PGuestContext context)
{
    ULONG64 reason = 0;
    ULONG64 instLen = 0;
    ULONG64 instinfo = 0;
    ULONG64 mrip = 0;
    ULONG64 mrsp = 0;
 
    __vmx_vmread(VM_EXIT_REASON, &reason);
    __vmx_vmread(VM_EXIT_INSTRUCTION_LEN, &instLen); // 获取指令长度
    __vmx_vmread(VMX_INSTRUCTION_INFO, &instinfo); //指令详细信息
    __vmx_vmread(GUEST_RIP, &mrip); //获取客户机触发VT事件的地址
    __vmx_vmread(GUEST_RSP, &mrsp);
 
    //获取事件码
    reason = reason & 0xFFFF;
 
    switch (reason)
    {
        case EXIT_REASON_CPUID:
        case EXIT_REASON_GETSEC:
        case EXIT_REASON_TRIPLE_FAULT:
        case EXIT_REASON_INVD:
 
        case EXIT_REASON_VMCALL            :
        case EXIT_REASON_VMCLEAR        :
        case EXIT_REASON_VMLAUNCH        :
        case EXIT_REASON_VMPTRLD        :
        case EXIT_REASON_VMPTRST        :
        case EXIT_REASON_VMREAD            :
        case EXIT_REASON_VMRESUME        :
        case EXIT_REASON_VMWRITE        :
        case EXIT_REASON_VMXOFF            :
        case EXIT_REASON_VMXON            :
 
        case EXIT_REASON_MSR_READ:
        case EXIT_REASON_MSR_WRITE:
 
        case EXIT_REASON_XSETBV:
    }
 
    __vmx_vmwrite(GUEST_RIP, mrip + instLen);
    __vmx_vmwrite(GUEST_RSP, mrsp);
}

```

1.  获取指令长度、指令信息、EIP ESP
2.  获取事件码
3.  对事件进行相应的处理
4.  rip+= 指令长度
5.  将 rip 和 rsp 写回并返回，回到虚拟机中产生 vm-exit 事件的指令的下一条指令处继续执行。

处理 vmx 指令导致的 vm-exit 事件
-----------------------

*   由于我们并不准备对 VT 嵌套进行实现，因此这里我们需要对 guest 中进行的 vmx 事件返回错误。在我们之前的文章中提到过在开启 VT 的时候，如果出现错误，则 CF 和 ZF 不全为 0。只有 vmx 指令被成功执行的时候 CF 和 ZF 才会全部被置为 0。因此这里为了让虚拟机其意识到无法继续进入 VT 环境，需要把 CF 和 ZF 置为 1。对于执行错误时对 rflags 寄存器的影响在《处理器虚拟化技术》2.6.2 中有说明，如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_MEAFBRMUKCFEYWS.jpg)
    
*   rflag 寄存器中位置如图（rflag 寄存器是 eflag 寄存器的简单扩充，高 32 位没有使用，全为 0）
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_GANCUATHB9KH25D.jpg)
    
    可见 CF 和 ZF 分别为第 0 位和第 6 位。只要将这两个位置为 1 并返回即可。代码如下
    

```
case EXIT_REASON_VMCALL            :
case EXIT_REASON_VMCLEAR        :
case EXIT_REASON_VMLAUNCH        :
case EXIT_REASON_VMPTRLD        :
case EXIT_REASON_VMPTRST        :
case EXIT_REASON_VMREAD            :
case EXIT_REASON_VMRESUME        :
case EXIT_REASON_VMWRITE        :
case EXIT_REASON_VMXOFF            :
case EXIT_REASON_VMXON            :
{
    ULONG64 rfl = 0;
    __vmx_vmread(GUEST_RFLAGS, &rfl);
    rfl |= 0x41;
    __vmx_vmwrite(GUEST_RFLAGS, &rfl);
}
    break;

```

处理 cpuid 导致的 vm-exit 事件
-----------------------

*   cpuid 也是一定会导致 vm-exit 事件发生的指令。如果不需要对 cpuid 的一些特定行为进行特定的处理，直接在处理函数中进行一次 cpuid 然后将得到的值返回给 guest 即可。这里注意处理函数相当于是在 host 环境下，在这里进行 cpuid 并不会重复导致 vm-exit 事件的发生。
    
*   为了检验我们是否真的拦截到了 cpuid 指令，可以通过对特殊值的判断来进行，如下
    

```
VOID VmxHandlerCpuid(PGuestContext context)
{
    if (context->mRax == 0x8888)
    {
        context->mRax = 0x11111111;
        context->mRbx = 0x22222222;
        context->mRcx = 0x33333333;
        context->mRdx = 0x44444444;
    }
    else
    {
        int cpuids[4] = {0};
        __cpuidex(cpuids,context->mRax, context->mRcx);
        context->mRax = cpuids[0];
        context->mRbx = cpuids[1];
        context->mRcx = cpuids[2];
        context->mRdx = cpuids[3];
    }
}

```

如果 rax 的值为 0x8888 的话，就将 rax rbx rcx rdx 分别置为特殊值。结果如下

 

![](https://bbs.pediy.com/upload/attach/202203/899076_EVESYNFVGK8C4M7.jpg)

 

可见我们已经成功 hook 了 cpuid 指令。

处理 getsec invd xsetbv 导致的 vm-exit 事件
------------------------------------

*   `getsec` 这个指令一般不会被调用，在开启 `SGX` 的时候可能会调用。不过我们用不到，因此这里可以暂时不进行处理。
    
*   `invd` 直接简单在 host 环境下进行一次 `invd` 指令并返回即可。
    

```
AsmInvd proc
    invd;
    ret
AsmInvd endp;

```

```
case EXIT_REASON_INVD:
{
    AsmInvd();
}
break;

```

*   `xsetbv` 同理，按照相应的规则调用 `xsetbv` 并返回即可。这里要注意这个指令和 rdmsr 指令类似，也是做了 32 位兼容的，要把 eax 和 edx 组合起来作为第二个参数，如下

```
#define MAKE_REG(A,B) ((A & 0xFFFFFFFF) | (B<<32))
case EXIT_REASON_XSETBV:
{
    ULONG64 value = MAKE_REG(context->mRax, context->mRdx);
    _xsetbv(context->mRcx, value);
}

```

guest 与 host 的通信 && 关闭 VT
-------------------------

*   当需要在虚拟机内部和虚拟机外部进行通信的时候，我们可以用任意一个会产生 vmexit 的事件来进行。只要在寄存器中存放约定好的参数即可。这里我们利用这个特性来实现关闭 VT 的功能。
    
*   我们规定这样一个规则，使用 vmcall 指令导致退出时，如果发现当前 rax 为'abcd'，那么就退出 VT 环境。这里要注意 vmcall 这个函数 vs 并没有提供接口，因此需要自己进行汇编代码的编写。如下
    

```
AsmVmCall proc
    mov rax,rcx
    vmcall
    ret;
AsmVmCall endp;

```

则需要退出虚拟机时进行如下调用即可

```
AsmVmCall('abcd');

```

*   那么我们需要在 vmexit 事件处理函数中对其进行判断

```
case EXIT_REASON_VMCALL:
{
    if (context->mRax == 'abcd')
    {
        __vmx_off();
        AsmJmpRet(mrip + instLen, mrsp);
        return;
    }
    else
    {
        ULONG64 rfl = 0;
        __vmx_vmread(GUEST_RFLAGS, &rfl);
        rfl |= 0x41;
        __vmx_vmwrite(GUEST_RFLAGS, &rfl);
    }
}
    break;

```

当发现退出事件是 `vmcall` ，且 rax='abcd'时，直接调用 `__vmx_off()` 指令关闭 VT。要注意的是，在关闭了 VT 之后，还是要跳回到 vmcall 指令的下一条指令继续执行。因此这里我们需要通过汇编直接修改 rsp 和 rip 跳回去。我们封装一个 `AsmJmpRet` 函数，传入需要返回的 rip 和返回后的 rsp，进行 rsp 的设置和 rip 的跳转。

```
AsmJmpRet proc
    mov rsp,rdx;
    jmp rcx;
    ret;
AsmJmpRet endp;

```

有条件退出的虚拟机事件
===========

对 msr 寄存器的有条件拦截
---------------

*   在上一篇文章中，我们提到了 processor-based vm-execution control 字段。对这个字段的设置可以让执行一些特定指令或者读写某些特定寄存器的时候产生 vm-exit 事件。上一篇文章中我们给这个字段填为全 0，并未对其进行相应设置。现在我们对其进行一定的设置。如图，可以看到该寄存器的第 28 位为 1 的话，则表示将会启动 MSR bitmap。
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_48AFCJ3QE8XEWJF.jpg)
    
    MSR bitmap 的具体含义如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_XA5BASYNBRSV9HD.jpg)
    
    即当 Use MSR bitmap 位为 1 时，可以为 `MSR_BITMAP` 字段提供一个 MSR bitmap 区域的物理地址。根据一定规则对其进行填充后，读写相应的寄存器时即会产生有条件的 vm-exit 事件。至于这块内存区域应该如何填充才能对特定 MSR 进行拦截，其规则在《处理器虚拟化技术》3.5.15 小节中有具体描述，规则如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_T7K4Z8HCGCP5UZK.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_6A95SZT47S3MZYF.jpg)
    
    *   MSR bitmap 区域为 4KB 大小（一个页大小）
    *   0~1KB：控制编号范围为 `00000000~00001fff` 的 MSR 寄存器的读访问是否会产生 vm-exit 事件。如果对应位为 1，则读取该 MSR 寄存器会产生 vm-exit 事件
    *   1~2KB：控制编号范围为 `C0000000~C0001FFF` 的 MSR 寄存器的读访问是否会产生 vm-exit 事件。如果对应位为 1，则读取该 MSR 寄存器会产生 vm-exit 事件
    *   2~3KB：控制编号范围为 `00000000~00001fff` 的 MSR 寄存器的写访问是否会产生 vm-exit 事件。如果对应位为 1，则写入该 MSR 寄存器会产生 vm-exit 事件
    *   3~4KB：控制编号范围为 `C0000000~C0001FFF` 的 MSR 寄存器的写访问是否会产生 vm-exit 事件。如果对应位为 1，则写入该 MSR 寄存器会产生 vm-exit 事件
*   对 MSR bitmap 的设置较为简单，对想要拦截的位置 1 即可，代码如下，不做过多阐述。
    

```
BOOLEAN VmxSetReadMsrBitMap(PUCHAR msrBitMap, ULONG64 msrAddrIndex, BOOLEAN isEnable)
{
    if (msrAddrIndex >= 0xC0000000)
    {
        msrBitMap += 1024;
        msrAddrIndex -= 0xC0000000;
    }
 
    ULONG64 moveByte = 0;
    ULONG64 setBit = 0;
 
    if (msrAddrIndex != 0)
    {
        moveByte = msrAddrIndex / 8;
 
        setBit = msrAddrIndex % 8;
 
        msrBitMap += moveByte;
    }
 
    if (isEnable)
    {
        *msrBitMap |= 1 << setBit;
    }
    else
    {
        *msrBitMap &= ~(1 << setBit);
    }
 
    return TRUE;
 
}
 
BOOLEAN VmxSetWriteMsrBitMap(PUCHAR msrBitMap, ULONG64 msrAddrIndex, BOOLEAN isEnable)
{
    msrBitMap += 0x800;
 
    return VmxSetReadMsrBitMap(msrBitMap, msrAddrIndex, isEnable);
 
}

```

*   可以通过对 c0000082 的拦截来进行 SSDThook。不过这个方法兼容性很差。在这里不深入展开。https://github.com/qq1045551070/VtToMe 仓库中有 SSDThook 相关的代码，有兴趣可以进行研究。

为了支持 win10 拦截 rdtscp invpcid xsaves/xsrstors
--------------------------------------------

*   论坛中小宝来了前辈发过一篇分析如何让 VT 支持 win10 的精华帖 https://bbs.pediy.com/thread-212786.htm，其中提到了为了让 VT 支持 win10，需要对 rdtscp 指令进行处理。如果不处理，则会因为产生 #UD 异常而导致系统崩溃。
    
*   为了防止因为指令不存在而出现 #UD 异常，我们需要将所有如果不处理的话可能导致 #UD 异常的指令进行处理。由于《处理器虚拟化技术》这本书成书时间比较早，书上对 Secondary Processor-Based VM-Execution Controls 字段的描述并不是很全，因此这里我们看 intel 白皮书中的相关内容。在 intel 白皮书 24.6 中，有 Secondary Processor-Based VM-Execution Controls 字段的描述表格。可以看到里面有如下几个不处理会导致 #UD 异常的指令
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_9XUX7SANKH5KFSW.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_KVXFCRBCZTTZA2U.jpg)
    
    这里首先要对 Secondary Processor-Based VM-Execution Controls 字段进行相应设置，使其能够拦截以上的指令
    

```
mseregister = IA32_MSR_VMX_PROCBASED_CTLS2;
ULONG64 secValue = SECONDARY_EXEC_ENABLE_RDTSCP | SECONDARY_EXEC_ENABLE_INVPCID | SECONDARY_EXEC_XSAVES;
value = VmxAdjustContorls(secValue, mseregister);
__vmx_vmwrite(SECONDARY_VM_EXEC_CONTROL, value);

```

*   接下来便是对这些指令的具体处理。

### 处理 rdtscp 指令导致的有条件退出

*   对于 rdtscp 指令，其作用如下
    
    > 于是我搜索了下 RDTSCP 这个指令的用途，它是 RDTSC 的升级版，在一些比较新的处理器中用于获得 CPU 时间计数器。
    
*   按照 intel 白皮书上的调用方法写出对应处理流程即可，其具体做了什么并不用太关心。
    

```
case EXIT_REASON_RDTSCP:
{
    int aunx = 0;
    LARGE_INTEGER in = {0};
    in.QuadPart =  __rdtscp(&aunx);
    context->mRax = in.LowPart;
    context->mRdx = in.HighPart;
    context->mRcx = aunx;
}

```

小宝在进行处理的时候将 `rdtsc` 指令也进行了处理。在 intel 白皮书 25.1.3 中写到，如果处理 rdtsc 和 rdtscp 的位同时置为 1 才会产生 vm-exit 事件

 

![](https://bbs.pediy.com/upload/attach/202203/899076_593XPTE52AJS9WW.jpg)

 

25.3 中说明了

 

![](https://bbs.pediy.com/upload/attach/202203/899076_E48EBRMQXZJ7MAS.jpg)

 

如果不设置 `rdtsc exiting` 和 `use tsc offsetting` 的话，那么 `rdtscp` 会正常执行，因此这里其实可以只设置 `SECONDARY_EXEC_ENABLE_RDTSCP` ，其他不设置。这样的话 `rdtscp` 指令的执行并不会导致 vm-exit 事件的发生。这里我尝试了只将 `SECONDARY_EXEC_ENABLE_RDTSCP` 控制位置位，其他位不动，发现完全不会进入到 vm-exit 事件中。因此如果图省事的话其实可以只将 `SECONDARY_EXEC_ENABLE_RDTSCP` 置位，其他位不进行操作即可。

### 处理 invpcid 指令导致的有条件退出

*   在《处理器虚拟化技术》3.10.4.2 中描述了 invept、invpcid、invvpid 指令 vm-exit 事件时寄存器中保存的信息以及应该如何对其进行处理。
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_RH5Z9BUF3WPP7FF.jpg)
    
    在由这三条指令产生 vm-exit 事件时，vm-exit qualification 字段中会记录指令操作数中的偏移量，也就是图中的 disp 值。而 vm-exit instruction information 字段中会记录其他的信息。具体保存的信息内容如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_PKAZR3E2W3H5V5P.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_2YTPNV7WFK7U74J.jpg)
    
    可见其中存储了 `register operand` `segment` `base` `index` `scale` 信息。只需要将这些寄存器填入获取地址，然后调用 `_invpcid` 指令即可。注意字段中还有 `index invalid` `base invalid` 字段用来指示对应的字段是否有效以及 `address size` 用来指示地址的大小。对这些标志也要进行相应的判断。代码如下
    

```
typedef struct _INVPCID
{
    ULONG64 scale : 2;
    ULONG64 und : 5;
    ULONG64 addrssSize : 3;
    ULONG64 rev1 : 1;
    ULONG64 und2 : 4;
    ULONG64 segement : 3;
    ULONG64 index : 4;
    ULONG64 indexInvaild : 1;
    ULONG64 base : 4;
    ULONG64 baseInvaild : 1;
    ULONG64 regOpt : 4;
    ULONG64 un3 : 32;
}INVPCID,*PINVPCID;
 
VOID VmxExitInvpcidHandler(PGuestContext context)
{
    ULONG64 mrsp = 0;
    ULONG64 instinfo = 0;
    ULONG64 qualification = 0;
    __vmx_vmread(VMX_INSTRUCTION_INFO, &instinfo); //指令详细信息
    __vmx_vmread(EXIT_QUALIFICATION, &qualification); //偏移量
    __vmx_vmread(GUEST_RSP, &mrsp);
 
    PINVPCID pinfo = (PINVPCID)&instinfo;
 
    ULONG64 base = 0;
    ULONG64 index = 0;
    ULONG64 scale = pinfo->scale ? (1 << pinfo->scale) : 0;
    ULONG64 addr = 0;
    ULONG64 regopt = ((PULONG64)context)[pinfo->regOpt];;
 
    if (!pinfo->baseInvaild)
    {
        if (pinfo->base == 4)
        {
            base = mrsp;
        }
        else
        {
            base = ((PULONG64)context)[pinfo->base];
        }
 
    }
 
    if (!pinfo->indexInvaild)
    {
        if (pinfo->index == 4)
        {
            index = mrsp;
        }
        else
        {
            index = ((PULONG64)context)[pinfo->index];
        }
 
    }
 
    if (pinfo->addrssSize == 0)
    {
        addr = *(PSHORT)(base + index * scale + qualification);
    }
    else if (pinfo->addrssSize == 1)
    {
        addr = *(PULONG)(base + index * scale + qualification);
    }
    else
    {
        addr = *(PULONG64)(base + index * scale + qualification);
    }
 
    _invpcid(regopt, &addr);
}

```

*   跟上面的类似，如果 `invlpg exiting` 没有被设置，那么也不会导致 vm-exit 事件的发生。因此这里其实也可以只设置 `SECONDARY_EXEC_ENABLE_INVPCID` ，不设置 `invlpg exiting` 。这样可以让其正常执行，不用在 vm-exit 处理函数中对其进行复杂的处理。
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_3KZ7QY996P5YEXD.jpg)
    

### 处理 xsaves 指令导致的有条件退出

*   这个指令好像没看到有人处理，https://github.com/qq1045551070/VtToMe 这个仓库中也同样没有处理这个事件。不过 windows 内核中确实有这个指令的调用。这个指令产生 vm-exit 的逻辑也与前面两个指令类似
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_PK2Y5N2DP8VN5TD.jpg)
    
    因此这里不进行处理，只设置 `SECONDARY_EXEC_XSAVES` 位，不设置 `xss-exiting bitmap` ，让其不产生 vm-exit 事件，进行正常的处理。
    
*   **本篇文章对应的代码晚些会传到 github 仓库 https://github.com/smallzhong/myvt 中**
    

参考文献
====

1.  intel 白皮书
2.  邓志《处理器虚拟化技术》
3.  B 站周壑 VT 教学视频
4.  https://github.com/qq1045551070/VtToMe
5.  https://bbs.pediy.com/thread-212786.htm
6.  火哥上课讲的内容（这里帮火哥打个广告，火哥 qq471194425，上课会讲很多干货，也会在讲解技术的时候同时讲解一些运用的方法，报火哥的班绝对是物超所值）

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 18 小时前 被 smallzhong_编辑 ，原因：

[#系统内核](forum-41-1-131.htm) [#驱动开发](forum-41-1-132.htm) [#HOOK / 注入](forum-41-1-133.htm) [#开源分享](forum-41-1-134.htm) [#虚拟化](forum-41-1-136.htm)