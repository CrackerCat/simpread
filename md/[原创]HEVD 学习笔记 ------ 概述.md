> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270159.htm)

> [原创]HEVD 学习笔记 ------ 概述

一. 前言  

========

HEVD 作为一个优秀的内核漏洞靶场受到大家的喜欢，靶场地址 [HackSysExtremeVulnerableDriver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver)。这里选择 x86 的驱动来进行黑盒测试学习内核漏洞，作为学习笔记记录下来。  

实验环境:

<table border="1"><tbody><tr><td valign="top"><strong>操作系统</strong></td><td valign="top">Win 7 X86</td></tr><tr><td valign="top"><strong>调试器</strong></td><td valign="top">WinDbg，IDA Pro</td></tr><tr><td valign="top"><strong>编译器</strong></td><td valign="top">Visual Studio 2017</td></tr></tbody></table>

二. 驱动信息  

==========

1.WinDbg    
------------

装载驱动以后首先使用 WinDbg 查看驱动的内容

```
SXS.DLL: Read 0 bytes from XML stream; HRESULT returned = 0x00000000
SXS.DLL: Creating 756 byte file mapping
                                         
 ##     ## ######## ##     ## ########  
 ##     ## ##       ##     ## ##     ## 
 ##     ## ##       ##     ## ##     ## 
 ######### ######   ##     ## ##     ## 
 ##     ## ##        ##   ##  ##     ## 
 ##     ## ##         ## ##   ##     ## 
 ##     ## ########    ###    ########  
   HackSys Extreme Vulnerable Driver    
             Version: 3.00              
[+] HackSys Extreme Vulnerable Driver Loaded
 
2: kd> lm m HEVD
start    end        module name
98c78000 98cc2000   HEVD       (deferred)             
2: kd> !drvobj HEVD 2
Driver object (87d68b90) is for:
*** ERROR: Module load completed but symbols could not be loaded for HEVD.sys
 \Driver\HEVD
DriverEntry:   98cc00ea  HEVD
DriverStartIo: 00000000   
DriverUnload:  98cbc000   HEVD
AddDevice:     00000000   
 
Dispatch routines:
[00] IRP_MJ_CREATE                      98cbc048 HEVD+0x44048
[01] IRP_MJ_CREATE_NAMED_PIPE           98cbc5c2    HEVD+0x445c2
[02] IRP_MJ_CLOSE                       98cbc048    HEVD+0x44048
[03] IRP_MJ_READ                        98cbc5c2   HEVD+0x445c2
[04] IRP_MJ_WRITE                       98cbc5c2    HEVD+0x445c2
[05] IRP_MJ_QUERY_INFORMATION           98cbc5c2    HEVD+0x445c2
[06] IRP_MJ_SET_INFORMATION             98cbc5c2  HEVD+0x445c2
[07] IRP_MJ_QUERY_EA                    98cbc5c2   HEVD+0x445c2
[08] IRP_MJ_SET_EA                      98cbc5c2 HEVD+0x445c2
[09] IRP_MJ_FLUSH_BUFFERS               98cbc5c2    HEVD+0x445c2
[0a] IRP_MJ_QUERY_VOLUME_INFORMATION    98cbc5c2   HEVD+0x445c2
[0b] IRP_MJ_SET_VOLUME_INFORMATION      98cbc5c2 HEVD+0x445c2
[0c] IRP_MJ_DIRECTORY_CONTROL           98cbc5c2    HEVD+0x445c2
[0d] IRP_MJ_FILE_SYSTEM_CONTROL         98cbc5c2  HEVD+0x445c2
[0e] IRP_MJ_DEVICE_CONTROL              98cbc064 HEVD+0x44064
[0f] IRP_MJ_INTERNAL_DEVICE_CONTROL     98cbc5c2  HEVD+0x445c2
[10] IRP_MJ_SHUTDOWN                    98cbc5c2   HEVD+0x445c2
[11] IRP_MJ_LOCK_CONTROL                98cbc5c2   HEVD+0x445c2
[12] IRP_MJ_CLEANUP                     98cbc5c2  HEVD+0x445c2
[13] IRP_MJ_CREATE_MAILSLOT             98cbc5c2  HEVD+0x445c2
[14] IRP_MJ_QUERY_SECURITY              98cbc5c2 HEVD+0x445c2
[15] IRP_MJ_SET_SECURITY                98cbc5c2   HEVD+0x445c2
[16] IRP_MJ_POWER                       98cbc5c2    HEVD+0x445c2
[17] IRP_MJ_SYSTEM_CONTROL              98cbc5c2 HEVD+0x445c2
[18] IRP_MJ_DEVICE_CHANGE               98cbc5c2    HEVD+0x445c2
[19] IRP_MJ_QUERY_QUOTA                 98cbc5c2  HEVD+0x445c2
[1a] IRP_MJ_SET_QUOTA                   98cbc5c2    HEVD+0x445c2
[1b] IRP_MJ_PNP                         98cbc5c2  HEVD+0x445c2

```

驱动装载的地址是 0x98C78000，DriverEntry 的地址是 0x98CC00EA，所以 DriverEntry 的偏移地址是 0x480EA。IRP_MJ_DEVICE_CONTROL 的分发函数偏移地址 0x445C2.  

2.IDA Pro  

------------

使用 IDA 对驱动进行分析，可以看到在 DriverEntry 首先是创建了设备对象  

```
INIT:00448036                 push    eax             ; DeviceObject
INIT:00448037                 push    edi             ; Exclusive
INIT:00448038                 push    FILE_DEVICE_SECURE_OPEN ; DeviceCharacteristics
INIT:0044803D                 push    FILE_DEVICE_UNKNOWN ; DeviceType
INIT:0044803F                 lea     eax, [ebp+DestinationString]
INIT:00448042                 push    eax             ; DeviceName
INIT:00448043                 push    edi             ; DeviceExtensionSize
INIT:00448044                 push    ebx             ; DriverObject
INIT:00448045                 call    ds:IoCreateDevice

```

随后就是对分发函数的赋值以及符号链接的创建

```
INIT:00448075                 push    1Ch
INIT:00448077                 pop     ecx
INIT:00448078                 mov     eax, offset DispatchCommon
INIT:0044807D                 lea     edi, [ebx+DRIVER_OBJECT.MajorFunction]
INIT:00448080                 rep stosd
INIT:00448082                 mov     eax, offset DispatchCreateAndClose
INIT:00448087                 mov     dword ptr [ebx+70h], offset DispatchIoCtrl
INIT:0044808E                 mov     [ebx+38h], eax
INIT:00448091                 mov     [ebx+40h], eax
INIT:00448094                 mov     eax, [ebp+DeviceObject]
INIT:00448097                 mov     [ebx+_DRIVER_OBJECT.DriverUnload], offset DriverUnload
INIT:0044809E                 or      [eax+DEVICE_OBJECT.Flags], DO_DIRECT_IO
INIT:004480A2                 mov     eax, [ebp+DeviceObject]
INIT:004480A5                 and     [eax+DEVICE_OBJECT.Flags], 0FFFFFF7Fh
INIT:004480AC                 lea     eax, [ebp+DestinationString]
INIT:004480AF                 push    eax             ; DeviceName
INIT:004480B0                 lea     eax, [ebp+SymbolicLinkName]
INIT:004480B3                 push    eax             ; SymbolicLinkName
INIT:004480B4                 call    ds:IoCreateSymbolicLink

```

而根据 IDA 识别的结果就可以得知符号名，根据符号名就可以完成和驱动的连接与通信  

```
INIT:00448134 aDeviceHacksyse:                        ; DATA XREF: DriverEntry(x,x)+14↑o
INIT:00448134                 text "UTF-16LE", '\Device\HackSysExtremeVulnerableDriver',0
INIT:00448182 ; const WCHAR aDosdevicesHack_0
INIT:00448182 aDosdevicesHack_0:                      ; DATA XREF: DriverEntry(x,x)+25↑o
INIT:00448182                 text "UTF-16LE", '\DosDevices\HackSysExtremeVulnerableDriver',0

```

而在 DispatchIoCtrl 中，程序将 IoControlCode 取出减去 0x222003 以后得到下标，在用这个下标从 Index_Table 中取出函数地址表的下标。在根据这个地址表的下标从 Func_Table 中获得函数地址以后跳转到该函数执行  

```
PAGE:00444064 ; int __stdcall DispatchIoCtrl(int, PIRP Irp)
PAGE:00444064 DispatchIoCtrl  proc near               ; DATA XREF: DriverEntry(x,x)+87↓o
PAGE:00444064
PAGE:00444064 Irp             = dword ptr  0Ch
PAGE:00444064
PAGE:00444064                 push    ebp
PAGE:00444065                 mov     ebp, esp
PAGE:00444067                 push    ebx
PAGE:00444068                 push    esi
PAGE:00444069                 push    edi
PAGE:0044406A                 mov     edi, [ebp+Irp]
PAGE:0044406D                 mov     ebx, STATUS_NOT_SUPPORTED
PAGE:00444072                 mov     eax, [edi+60h]  ; 取出CurrentStackLocation指针赋给eax
PAGE:00444075                 test    eax, eax
PAGE:00444077                 jz      loc_4444C5
PAGE:0044407D                 mov     ebx, eax
PAGE:0044407F                 mov     ecx, [ebx+IO_STACK_LOCATION.Parameters.DeviceIoControl.IoControlCode]
PAGE:00444082                 lea     eax, [ecx-222003h] ; switch 109 cases
PAGE:00444088                 cmp     eax, 6Ch
PAGE:0044408B                 ja      loc_4444AD      ; jumptable 00444098 default case
PAGE:00444091                 movzx   eax, ds:Index_Table[eax]
PAGE:00444098                 jmp     ds:Func_Table[eax*4] ; switch jump

```

而这两张表的内容如下，其中的 FuncTable 中的每一个地址都代表了不同的漏洞  

```
PAGE:004444E0 Func_Table      dd offset loc_44409F, offset loc_4440CF, offset loc_4440F1
PAGE:004444E0                                         ; DATA XREF: DispatchIoCtrl+34↑r
PAGE:004444E0                 dd offset loc_444113, offset loc_444135, offset loc_44415A ; jump table for switch statement
PAGE:004444E0                 dd offset loc_44417F, offset loc_4441A4, offset loc_4441C9
PAGE:004444E0                 dd offset loc_4441EE, offset loc_444213, offset loc_444238
PAGE:004444E0                 dd offset loc_44425D, offset loc_444282, offset loc_4442A7
PAGE:004444E0                 dd offset loc_4442CC, offset loc_4442F1, offset loc_444316
PAGE:004444E0                 dd offset loc_44433B, offset loc_444360, offset loc_444385
PAGE:004444E0                 dd offset loc_4443AA, offset loc_4443CF, offset loc_4443F4
PAGE:004444E0                 dd offset loc_444419, offset loc_44443E, offset loc_444463
PAGE:004444E0                 dd offset loc_444488, offset loc_4444AD
PAGE:00444554 Index_Table     db      0,   1Ch,   1Ch,   1Ch
PAGE:00444554                                         ; DATA XREF: DispatchIoCtrl+2D↑r
PAGE:00444554                 db      1,   1Ch,   1Ch,   1Ch ; indirect table for switch statement
PAGE:00444554                 db      2,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      3,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      4,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      5,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      6,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      7,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      8,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db      9,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    0Ah,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    0Bh,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    0Ch,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    0Dh,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    0Eh,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    0Fh,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    10h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    11h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    12h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    13h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    14h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    15h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    16h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    17h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    18h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    19h,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    1Ah,   1Ch,   1Ch,   1Ch
PAGE:00444554                 db    1Bh
PAGE:004445C1                 align 2

```

如果取出的函数地址表的下标是 0x1C，那么对应的就是最后一个跳转地址，也就是 loc_4444AD。而这个地址中的代码是在告知用户，发送的 IOCTL 是不合法的 IOCTL

```
PAGE:004444AD loc_4444AD:                             ; CODE XREF: DispatchIoCtrl+27↑j
PAGE:004444AD                                         ; DispatchIoCtrl+34↑j
PAGE:004444AD                                         ; DATA XREF: ...
PAGE:004444AD                 push    ecx             ; jumptable 00444098 default case
PAGE:004444AE                 push    offset aInvalidIoctlCo ; "[-] Invalid IOCTL Code: 0x%X\n"
PAGE:004444B3                 push    3               ; Level
PAGE:004444B5                 push    DPFLTR_IHVDRIVER_ID ; ComponentId
PAGE:004444B7                 call    ds:DbgPrintEx
PAGE:004444BD                 add     esp, 10h
PAGE:004444C0                 mov     ebx, STATUS_INVALID_DEVICE_REQUEST
PAGE:004444C5
PAGE:004444C5 loc_4444C5:                             ; CODE XREF: DispatchIoCtrl+13↑j
PAGE:004444C5                                         ; DispatchIoCtrl+66↑j
PAGE:004444C5                 and     [edi+_IRP.IoStatus.Information], 0
PAGE:004444C9                 xor     dl, dl          ; PriorityBoost
PAGE:004444CB                 mov     ecx, edi        ; Irp
PAGE:004444CD                 mov     [edi+_IRP.IoStatus.anonymous_0.Status], ebx
PAGE:004444D0                 call    ds:IofCompleteRequest
PAGE:004444D6                 pop     edi
PAGE:004444D7                 pop     esi
PAGE:004444D8                 mov     eax, ebx
PAGE:004444DA                 pop     ebx
PAGE:004444DB                 pop     ebp
PAGE:004444DC                 retn    8
PAGE:004444DC DispatchIoCtrl  endp

```

这就可以知道，要触发不同的漏洞，IOCTL 是从 0x222003 开始，每次都要增加 4，最多可以增加 0x1B 次。  

[【公告】【iPhone 13 大奖等你拿】看雪. 众安 2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

最后于 1 天前 被 1900 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#Windows](forum-150-1-160.htm)

上传的附件：

*   [HEVD.rar](javascript:void(0)) （1.40MB，2 次下载）
*   [tools.rar](javascript:void(0)) （212.25kb，2 次下载）