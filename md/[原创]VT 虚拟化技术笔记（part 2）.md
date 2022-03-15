> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271869.htm)

> [原创]VT 虚拟化技术笔记（part 2）

目录

*   VT 虚拟化技术笔记（part 2）
*   [设置 vmcs 控制区的 guest 和 host 区域](#设置vmcs控制区的guest和host区域)
*            vmxinit 函数框架的搭建以及 guest rip,rsp 的获取方法
*            [设置 vmcs 字段的 vmxinitvmcs 函数解析](#设置vmcs字段的vmxinitvmcs函数解析)
*            GDT 表项属性的分离与填充（es cs ss ds fs gs ldtr）
*            [tr 寄存器 gdt 表项的填充](#tr寄存器gdt表项的填充)
*            [guest 区域中其他寄存器的填充](#guest区域中其他寄存器的填充)
*            [guest 区域填充完整代码](#guest区域填充完整代码)
*            [host 区域填充](#host区域填充)
*            [参考文献](#参考文献)

VT 虚拟化技术笔记（part 2）
==================

*   最近在学习 VT 技术，想把学习过程中记录的笔记分享出来。技术不精，有不对的地方还望指正。代码会发到 https://github.com/smallzhong/myvt 这个仓库中，目前还在施工中，还没写完。欢迎 star，预计这个月内完工。

设置 vmcs 控制区的 guest 和 host 区域
============================

*   本文讲解 vmcs 控制区中 Guest state fields 和 Host state fields 字段的填充方法，vm-control fields 的相关填充方法将会在下一篇文章中讲解。
    
*   ![](https://bbs.pediy.com/upload/attach/202203/899076_6YFUHF7XBNZM8EG.jpg)
    
    在 intel 白皮书的 24.3 章中，详细描述了 VMCS 控制区的字段。如上一篇文章所说，需要设置的 vmcs 字段为
    
    1.  Guest state fields，在 VT 从虚拟机中退出时，处理器的状态（寄存器等）会被存储在该区域中。而进入虚拟机（开启 VT）时，虚拟机中的各种处理器的状态都由进入虚拟机时该区域的对应字段的值来决定。
    2.  Host state fields，在从虚拟机中退出时，host 对 CPU 进行接管。host 接管后各种寄存器的状态存储在这个区域。也就是说当虚拟机中发生了 vm-exit 事件，CPU 会从 guest 返回到 host，这个区域中的值会被设置到对应的寄存器中，然后按照设置完之后的 eip 继续执行。
    3.  vm-control fields
        
        1.  vm execution control
        2.  vm exit control
        3.  vm entry control
        
        除这 5 个区域之外，vmcs 还有一个区域是 vm exit 信息区域。这个区域是只读的，存储的是 vmx 指令失败后失败代码的编号。
        
*   在《处理器虚拟化技术》3.4 章节中，展示了需要填充的字段以及其对应的 ID。对于 guest 区域和 host 区域，需要填充的字段如下：
    
    长度为 16 位的字段:
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_E68WY9C3N6SSQ7T.jpg)
    
    长度为 64 位的字段
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_23Y8PGAVABRS7T6.jpg)
    
    长度为 32 位的字段
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_ZDRNBTNW6P2R8XX.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_QT3KHE59D2KPMK3.jpg)
    
    可以看到基本上都是进入 guest 区域之后的寄存器。其中对段寄存器的填充尤其麻烦，需要分别填充 base、attribute、limit、selector。因此需要手动获取段寄存器并将其进行拆分然后进行相应的填充。根据保护模式的知识，段寄存器的结构大致如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_5TZGMCZ3KJJJCN5.jpg)
    
    在填充时需要将其拆分并填充，此步骤较为繁琐。
    
*   填充 guest 和 host 区域时还有四个重要的字段需要获取。分别是进入 guest 区域之后的 rip 和 rsp 以及从 guest 返回到 host 区域之后的 rip rsp。这里我们要使得进入 guest 区域之后系统仍然正常进行，还是从原来的地方往下跑。因此要通过函数获取需要返回的上一层函数的返回地址以及 rsp。而对于返回 host 区域之后的 host eip，由于从虚拟机中返回一定是发生了 vmexit 事件，需要对该事件进行处理，因此从虚拟机中返回之后的 rip 一定要设置为 vmexit 事件的处理函数。而 rsp 则需要重新开辟一块内存区域供 vmexit 事件处理函数使用。如果还是使用 guest 返回之前的堆栈，则会破坏堆栈中的内容，导致无法预知的结果。
    
*   接下来开始具体字段的填充讲解。
    

vmxinit 函数框架的搭建以及 guest rip,rsp 的获取方法
-------------------------------------

*   这里封装一个 vmxinit 函数用于进行 vmon 和 vmcs 区域的初始化。其中 vmon 区域的初始化较为简单，申请内存后将开头 4 个字节填充为 IA32_VMX_BASE 的后四字节，然后根据前一篇文章讲到的规则进行 cr0，cr4 的设置，最后通过 vmxon 指令进行开柜门即可。详情看代码的实现，如果对其中操作有不清楚的地方可以参考上一篇文章。

```
int VmxInitVmOn()
{
    PVMXCPUPCB pVcpu = VmxGetCurrentCPUPCB();
 
    PHYSICAL_ADDRESS lowphys,heiPhy;
 
    lowphys.QuadPart = 0;
    heiPhy.QuadPart = -1;
 
    pVcpu->VmxOnAddr = MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, lowphys, heiPhy, lowphys, MmCached);
 
    if (!pVcpu->VmxOnAddr)
    {
        //申请内存失败
        return -1;
    }
 
    memset(pVcpu->VmxOnAddr, 0, PAGE_SIZE);
 
    pVcpu->VmxOnAddrPhys = MmGetPhysicalAddress(pVcpu->VmxOnAddr);
 
    //填充ID
    ULONG64 vmxBasic = __readmsr(IA32_VMX_BASIC);
 
    *(PULONG)pVcpu->VmxOnAddr = (ULONG)vmxBasic;
 
    //CR0,CR4
 
    ULONG64 vcr00 = __readmsr(IA32_VMX_CR0_FIXED0);
    ULONG64 vcr01 = __readmsr(IA32_VMX_CR0_FIXED1);
 
    ULONG64 vcr04 = __readmsr(IA32_VMX_CR4_FIXED0);
    ULONG64 vcr14 = __readmsr(IA32_VMX_CR4_FIXED1);
 
    ULONG64 mcr4 = __readcr4();
    ULONG64 mcr0 = __readcr0();
 
    mcr4 |= vcr04;
    mcr4 &= vcr14;
 
    mcr0 |= vcr00;
    mcr0 &= vcr01;
 
    //
    __writecr0(mcr0);
 
    __writecr4(mcr4);
 
    int error = __vmx_on(&pVcpu->VmxOnAddrPhys.QuadPart);
 
    if (error)
    {
        //释放内存，重置CR4
        mcr4 &= ~vcr04;
        __writecr4(mcr4);
        MmFreeContiguousMemorySpecifyCache(pVcpu->VmxOnAddr, PAGE_SIZE, MmCached);
        pVcpu->VmxOnAddr = NULL;
        pVcpu->VmxOnAddrPhys.QuadPart = 0;
    }
 
    return error;
}

```

*   在初始化 vmcs 区域之前，首先要获取进入 guest 之后 guest 的 rip 以及 rsp。由于我们希望在进入 guest 之后可以让虚拟机继续在我们进入 guest 之前的代码上执行，因此我们要获取 vmxinit 函数的返回地址以及堆栈中保存的上一层函数的 rsp。在进入到 guest 之后直接从 vmxinit 函数的返回地址开始跑，并将堆栈设置为上一层函数的堆栈。这里要使用到一个 vs 的内嵌函数 `_AddressOfReturnAddress` 。这个函数在编译时会返回指向堆栈中上一层函数的返回地址的指针。因此通过这个函数可以得到指向需要使用的 rip 的指针。而 rip 存储的位置往下 8 个字节便是上一层函数的 rsp。因此获取 guest 函数的 rip 和 rsp 的代码如下

```
PULONG64 retAddr = (PULONG64)_AddressOfReturnAddress();
ULONG64 guestEsp = retAddr + 1;
ULONG64 guestEip = *retAddr;

```

*   因此，vmxinit 函数的大体框架如下。其中 hosteip 传入的是 vmexit 的处理函数的地址。在发生 vmexit 事件之后跳到 vmexit 处理函数中进行相应处理。

```
int VmxInit(ULONG64 hostEip)
{
 
    PVMXCPUPCB pVcpu = VmxGetCurrentCPUPCB();
 
    pVcpu->cpuNumber = KeGetCurrentProcessorNumberEx(NULL);
 
    PULONG64 retAddr = (PULONG64)_AddressOfReturnAddress();
    ULONG64 guestEsp = retAddr + 1;
    ULONG64 guestEip = *retAddr;
 
    int error = VmxInitVmOn();
 
    if (error)
    {
        DbgPrintEx(77, 0, "[db]:vmon 初始化失败 error = %d,cpunumber %d\r\n", error, pVcpu->cpuNumber);
 
        return error;
    }
 
    error = VmxInitVmcs(guestEip, guestEsp, hostEip);
 
    if (error)
    {
        DbgPrintEx(77, 0, "[db]:vmcs 初始化失败 error = %d,cpunumber %d\r\n", error, pVcpu->cpuNumber);
 
 
        VmxDestory();
        return error;
    }
 
    return 0;
}

```

设置 vmcs 字段的 vmxinitvmcs 函数解析
----------------------------

*   `VmxInitVmcs` 函数中进行对 vmcs 的设置。

```
int VmxInitVmcs(ULONG64 GuestEip,ULONG64 GuestEsp, ULONG64 hostEip)
{
    PVMXCPUPCB pVcpu = VmxGetCurrentCPUPCB();
 
    PHYSICAL_ADDRESS lowphys, heiPhy;
 
    lowphys.QuadPart = 0;
    heiPhy.QuadPart = -1;
 
    pVcpu->VmxcsAddr = MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, lowphys, heiPhy, lowphys, MmCached);
 
    if (!pVcpu->VmxcsAddr)
    {
        //申请内存失败
        return -1;
    }
 
    memset(pVcpu->VmxcsAddr, 0, PAGE_SIZE);
 
    pVcpu->VmxcsAddrPhys = MmGetPhysicalAddress(pVcpu->VmxcsAddr);
 
 
    pVcpu->VmxHostStackTop = MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE * 36, lowphys, heiPhy, lowphys, MmCached);
 
    if (!pVcpu->VmxHostStackTop)
    {
        //申请内存失败
 
        return -1;
    }
 
    memset(pVcpu->VmxHostStackTop, 0, PAGE_SIZE * 36);
 
    pVcpu->VmxHostStackBase = (ULONG64)pVcpu->VmxHostStackTop + PAGE_SIZE * 36 - 0x200;
 
    //填充ID
    ULONG64 vmxBasic = __readmsr(IA32_VMX_BASIC);
 
    *(PULONG)pVcpu->VmxcsAddr = (ULONG)vmxBasic;
 
    //加载VMCS
    __vmx_vmclear(&pVcpu->VmxcsAddrPhys.QuadPart);
 
    __vmx_vmptrld(&pVcpu->VmxcsAddrPhys.QuadPart);
 
    VmxInitGuest(GuestEip, GuestEsp);
 
    VmxInitHost(hostEip);
}

```

和设置 vmon 区域类似，首先也是申请内存区域然后填充 IA32_VMX_BASIC。而在进行基本 ID 的填充之后，要通过 vmclear 初始化内存并通过 vmptrld 选择 vmcs 区域。这两步操作对应的是前一篇文章说的拔电源以及选中机器。在完成之后，便是最复杂的 vmcs 字段的填充。这里对于每一块 vmcs 字段，分别封装一个函数进行初始化。本篇文章讨论的是 guest 区域和 host 区域的初始化，分别对应的是 `VmxInitGuest` 和 `VmxInitHost` 函数。

GDT 表项属性的分离与填充（es cs ss ds fs gs ldtr）
--------------------------------------

*   对于 guest 相关的字段，需要传入的是 guesteip 和 guestesp，以用来确定在进入 guest 虚拟机之后 guest 从哪里开始跑。其他的字段全部根据当前状态填入。首先需要填入的便是 gdt 表中各个段寄存器的 base、limit、attribute、selector。在对其 ID 进行观察后可以发现，这些字段的 id 都是连在一起的，id 的值相差 2。而对于这些段寄存器，分离 base、limit、attribute、selector 的方法也非常类似。因此可以考虑将填写段寄存器的属性的方法封装成一个函数。这里将其封装成 `fillGdtDataItem` 函数。对于各个属性的分离，依照下图来进行。具体分离的细节不赘述，建议仔细读懂代码中切割 bit 的方法。
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_5TZGMCZ3KJJJCN5.jpg)
    

```
void fillGdtDataItem(int index,short selector)
{
    GdtTable gdtTable = {0};
    AsmGetGdtTable(&gdtTable);
 
    selector &= 0xFFF8;
 
    ULONG limit = __segmentlimit(selector);
    PULONG item = (PULONG)(gdtTable.Base + selector);
 
    LARGE_INTEGER itemBase = {0};
    itemBase.LowPart = (*item & 0xFFFF0000) >> 16;
    item += 1;
    itemBase.LowPart |= (*item & 0xFF000000) | ((*item & 0xFF) << 16);
 
    //属性
    ULONG attr = (*item & 0x00F0FF00) >> 8;
 
    if (selector == 0)
    {
        attr |= 1 << 16;
    }
 
    __vmx_vmwrite(GUEST_ES_BASE + index * 2, itemBase.QuadPart);
    __vmx_vmwrite(GUEST_ES_LIMIT + index * 2, limit);
    __vmx_vmwrite(GUEST_ES_AR_BYTES + index * 2, attr);
    __vmx_vmwrite(GUEST_ES_SELECTOR + index * 2, selector);
}

```

tr 寄存器 gdt 表项的填充
----------------

*   对于 tr 寄存器的 gdt 表项，并不能像其他寄存器一样进行填充。因为在 64 位下，tr 寄存器的 gdt 表项是 128 位的。因此需要单独设置。64 位下 tr 寄存器的 gdt 表项格式在 intel 白皮书的 7.2.3 章节中有解析。其结构如下
    
    ![](https://bbs.pediy.com/upload/attach/202203/899076_CZCJKEW278F44DZ.jpg)
    
    因此其需要用另外一套代码进行设置，具体代码如下。思路与其他 gdt 表项的设置思路相同，都是取出对应的位填入 vmcs 区域中。
    

```
GdtTable gdtTable = { 0 };
AsmGetGdtTable(&gdtTable);
 
ULONG trSelector = AsmReadTR();
 
trSelector &= 0xFFF8;
ULONG trlimit = __segmentlimit(trSelector);
 
LARGE_INTEGER trBase = {0};
 
PULONG trItem = (PULONG)(gdtTable.Base + trSelector);
 
//读TR
trBase.LowPart = ((trItem[0] >> 16) & 0xFFFF) | ((trItem[1] & 0xFF) << 16) | ((trItem[1] & 0xFF000000));
trBase.HighPart = trItem[2];
 
//属性
ULONG attr = (trItem[1] & 0x00F0FF00) >> 8;
__vmx_vmwrite(GUEST_TR_BASE, trBase.QuadPart);
__vmx_vmwrite(GUEST_TR_LIMIT, trlimit);
__vmx_vmwrite(GUEST_TR_AR_BYTES, attr);
__vmx_vmwrite(GUEST_TR_SELECTOR, trSelector);

```

guest 区域中其他寄存器的填充
-----------------

*   接下来是其他一些特殊寄存器的填充。这里不具体深入说。但是其实这里一些特殊寄存器的特殊属性可以用来做虚拟机检测。在进行一些特殊的操作之后可以使得在 host 状态下和 guest 状态下的结果不一样，从而检测到 VT 的存在。例如在之后的 msr 设置中，如果在 guest 中尝试读取一个超出 msr 范围的寄存器，在真机中会抛错，但是在虚拟机中不具体处理的话会产生不可预料的结果。虽然 intel 保证无法在 guest 中检测到自身是否为 guest，但是还是可以通过很多方法进行相应的检测。反虚拟机和反反虚拟机都要求开发者对 CPU 的各种行为有很深入的理解。

```
__vmx_vmwrite(GUEST_CR0, __readcr0());
__vmx_vmwrite(GUEST_CR4, __readcr4());
__vmx_vmwrite(GUEST_CR3, __readcr3());
__vmx_vmwrite(GUEST_DR7, __readdr(7));
__vmx_vmwrite(GUEST_RFLAGS, __readeflags());
__vmx_vmwrite(GUEST_RSP, GuestEsp);
__vmx_vmwrite(GUEST_RIP, GuestEip);
 
__vmx_vmwrite(VMCS_LINK_POINTER, -1);
 
__vmx_vmwrite(GUEST_IA32_DEBUGCTL, __readmsr(IA32_MSR_DEBUGCTL));
__vmx_vmwrite(GUEST_IA32_PAT, __readmsr(IA32_MSR_PAT));
__vmx_vmwrite(GUEST_IA32_EFER, __readmsr(IA32_MSR_EFER));
 
__vmx_vmwrite(GUEST_FS_BASE, __readmsr(IA32_FS_BASE));
__vmx_vmwrite(GUEST_GS_BASE, __readmsr(IA32_GS_BASE));
 
__vmx_vmwrite(GUEST_SYSENTER_CS, __readmsr(0x174));
__vmx_vmwrite(GUEST_SYSENTER_ESP, __readmsr(0x175));
__vmx_vmwrite(GUEST_SYSENTER_EIP, __readmsr(0x176));
 
 
//IDT GDT
 
GdtTable idtTable;
__sidt(&idtTable);
 
__vmx_vmwrite(GUEST_GDTR_BASE, gdtTable.Base);
__vmx_vmwrite(GUEST_GDTR_LIMIT, gdtTable.limit);
__vmx_vmwrite(GUEST_IDTR_LIMIT, idtTable.limit);
__vmx_vmwrite(GUEST_IDTR_BASE, idtTable.Base);

```

guest 区域填充完整代码
--------------

```
void fillGdtDataItem(int index,short selector)
{
    GdtTable gdtTable = {0};
    AsmGetGdtTable(&gdtTable);
 
    selector &= 0xFFF8;
 
    ULONG limit = __segmentlimit(selector);
    PULONG item = (PULONG)(gdtTable.Base + selector);
 
    LARGE_INTEGER itemBase = {0};
    itemBase.LowPart = (*item & 0xFFFF0000) >> 16;
    item += 1;
    itemBase.LowPart |= (*item & 0xFF000000) | ((*item & 0xFF) << 16);
 
    //属性
    ULONG attr = (*item & 0x00F0FF00) >> 8;
 
    if (selector == 0)
    {
        attr |= 1 << 16;
    }
 
    __vmx_vmwrite(GUEST_ES_BASE + index * 2, itemBase.QuadPart);
    __vmx_vmwrite(GUEST_ES_LIMIT + index * 2, limit);
    __vmx_vmwrite(GUEST_ES_AR_BYTES + index * 2, attr);
    __vmx_vmwrite(GUEST_ES_SELECTOR + index * 2, selector);
}
 
void VmxInitGuest(ULONG64 GuestEip, ULONG64 GuestEsp)
{
    fillGdtDataItem(0, AsmReadES());
    fillGdtDataItem(1, AsmReadCS());
    fillGdtDataItem(2, AsmReadSS());
    fillGdtDataItem(3, AsmReadDS());
    fillGdtDataItem(4, AsmReadFS());
    fillGdtDataItem(5, AsmReadGS());
    fillGdtDataItem(6, AsmReadLDTR());
 
    GdtTable gdtTable = { 0 };
    AsmGetGdtTable(&gdtTable);
 
    ULONG trSelector = AsmReadTR();
 
    trSelector &= 0xFFF8;
    ULONG trlimit = __segmentlimit(trSelector);
 
    LARGE_INTEGER trBase = {0};
 
    PULONG trItem = (PULONG)(gdtTable.Base + trSelector);
 
 
    //读TR
    trBase.LowPart = ((trItem[0] >> 16) & 0xFFFF) | ((trItem[1] & 0xFF) << 16) | ((trItem[1] & 0xFF000000));
    trBase.HighPart = trItem[2];
 
    //属性
    ULONG attr = (trItem[1] & 0x00F0FF00) >> 8;
    __vmx_vmwrite(GUEST_TR_BASE, trBase.QuadPart);
    __vmx_vmwrite(GUEST_TR_LIMIT, trlimit);
    __vmx_vmwrite(GUEST_TR_AR_BYTES, attr);
    __vmx_vmwrite(GUEST_TR_SELECTOR, trSelector);
 
    __vmx_vmwrite(GUEST_CR0, __readcr0());
    __vmx_vmwrite(GUEST_CR4, __readcr4());
    __vmx_vmwrite(GUEST_CR3, __readcr3());
    __vmx_vmwrite(GUEST_DR7, __readdr(7));
    __vmx_vmwrite(GUEST_RFLAGS, __readeflags());
    __vmx_vmwrite(GUEST_RSP, GuestEsp);
    __vmx_vmwrite(GUEST_RIP, GuestEip);
 
    __vmx_vmwrite(VMCS_LINK_POINTER, -1);
 
    __vmx_vmwrite(GUEST_IA32_DEBUGCTL, __readmsr(IA32_MSR_DEBUGCTL));
    __vmx_vmwrite(GUEST_IA32_PAT, __readmsr(IA32_MSR_PAT));
    __vmx_vmwrite(GUEST_IA32_EFER, __readmsr(IA32_MSR_EFER));
 
    __vmx_vmwrite(GUEST_FS_BASE, __readmsr(IA32_FS_BASE));
    __vmx_vmwrite(GUEST_GS_BASE, __readmsr(IA32_GS_BASE));
 
    __vmx_vmwrite(GUEST_SYSENTER_CS, __readmsr(0x174));
    __vmx_vmwrite(GUEST_SYSENTER_ESP, __readmsr(0x175));
    __vmx_vmwrite(GUEST_SYSENTER_EIP, __readmsr(0x176));
 
 
    //IDT GDT
 
    GdtTable idtTable;
    __sidt(&idtTable);
 
    __vmx_vmwrite(GUEST_GDTR_BASE, gdtTable.Base);
    __vmx_vmwrite(GUEST_GDTR_LIMIT, gdtTable.limit);
    __vmx_vmwrite(GUEST_IDTR_LIMIT, idtTable.limit);
    __vmx_vmwrite(GUEST_IDTR_BASE, idtTable.Base);
 
}

```

host 区域填充
---------

*   host 区域填充的内容和 guest 填充的内容类似。注意 host 中的 gdt 表项并不需要填充各项属性，只需要填充 selector。另一个需要注意的点是 host 的 rsp 必须使用自己申请的一块内存。如果还是使用 guest 退出时的 rsp，一定会导致 guest 中堆栈被破坏从而导致不可预知的结果。host 区域填充的代码如下

```
void VmxInitHost(ULONG64 HostEip)
{
    GdtTable gdtTable = { 0 };
    AsmGetGdtTable(&gdtTable);
 
    PVMXCPUPCB pVcpu = VmxGetCurrentCPUPCB();
 
    ULONG trSelector = AsmReadTR();
 
    trSelector &= 0xFFF8;
 
    LARGE_INTEGER trBase = { 0 };
 
    PULONG trItem = (PULONG)(gdtTable.Base + trSelector);
 
 
    //读TR
    trBase.LowPart = ((trItem[0] >> 16) & 0xFFFF) | ((trItem[1] & 0xFF) << 16) | ((trItem[1] & 0xFF000000));
    trBase.HighPart = trItem[2];
 
    //属性
    __vmx_vmwrite(HOST_TR_BASE, trBase.QuadPart);
    __vmx_vmwrite(HOST_TR_SELECTOR, trSelector);
 
    __vmx_vmwrite(HOST_ES_SELECTOR, AsmReadES() & 0xfff8);
    __vmx_vmwrite(HOST_CS_SELECTOR, AsmReadCS() & 0xfff8);
    __vmx_vmwrite(HOST_SS_SELECTOR, AsmReadSS() & 0xfff8);
    __vmx_vmwrite(HOST_DS_SELECTOR, AsmReadDS() & 0xfff8);
    __vmx_vmwrite(HOST_FS_SELECTOR, AsmReadFS() & 0xfff8);
    __vmx_vmwrite(HOST_GS_SELECTOR, AsmReadGS() & 0xfff8);
 
 
 
    __vmx_vmwrite(HOST_CR0, __readcr0());
    __vmx_vmwrite(HOST_CR4, __readcr4());
    __vmx_vmwrite(HOST_CR3, __readcr3());
    __vmx_vmwrite(HOST_RSP, (ULONG64)pVcpu->VmxHostStackBase);
    __vmx_vmwrite(HOST_RIP, HostEip);
 
 
    __vmx_vmwrite(HOST_IA32_PAT, __readmsr(IA32_MSR_PAT));
    __vmx_vmwrite(HOST_IA32_EFER, __readmsr(IA32_MSR_EFER));
 
    __vmx_vmwrite(HOST_FS_BASE, __readmsr(IA32_FS_BASE));
    __vmx_vmwrite(HOST_GS_BASE, __readmsr(IA32_GS_KERNEL_BASE));
 
    __vmx_vmwrite(HOST_IA32_SYSENTER_CS, __readmsr(0x174));
    __vmx_vmwrite(HOST_IA32_SYSENTER_ESP, __readmsr(0x175));
    __vmx_vmwrite(HOST_IA32_SYSENTER_EIP, __readmsr(0x176));
 
 
    //IDT GDT
 
    GdtTable idtTable;
    __sidt(&idtTable);
 
    __vmx_vmwrite(HOST_GDTR_BASE, gdtTable.Base);
    __vmx_vmwrite(HOST_IDTR_BASE, idtTable.Base);
}

```

*   **下一篇文章将会讲解 vm-control fields 的填充。本篇文章的代码稍晚会传到 github 仓库中。**

参考文献
----

1.  intel 白皮书
2.  邓志《处理器虚拟化技术》
3.  B 站周壑 VT 教学视频
4.  https://github.com/qq1045551070/VtToMe
5.  火哥上课讲的内容

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 20 小时前 被 smallzhong_编辑 ，原因：

[#HOOK / 注入](forum-41-1-133.htm) [#驱动开发](forum-41-1-132.htm) [#系统内核](forum-41-1-131.htm) [#虚拟化](forum-41-1-136.htm) [#开源分享](forum-41-1-134.htm)