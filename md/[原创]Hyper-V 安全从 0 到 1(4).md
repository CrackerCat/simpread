> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-222657.htm)

书接上文。。。

______________________________________________________________________________________________

4.3. Hypervisor 层

         本节的内容主要有两点，一方面是虚拟机执行了 vmcall 指令之后发生的事情，另一方面是虚拟机和宿主机之间的内存映射关系。下面我们就这两点进行讨论。

         在开始介绍之前，我们需要一份 Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 3B: System Programming Guide, Part 2 文档，这份文档可以直接从 Intel 官方下载，这个文档是 Intel VT 技术的详细介绍与使用。

vmcall 指令的处理

         经过上面小节的介绍，我们发现虚拟机内核可以通过执行 vmcall 指令将通知发送给 Hypervisor 层，下面我们来介绍在执行 vmcall 之后的处理。

         通过查阅 Intel 文档，我们得知在发生 VM-Exit 事件之后，使用 vmread 读取 0x4402 字段的数据可以将 VM-Exit 事件产生的原因代码取出来，并且 VM-Exit 原因代码为 0x12 代表 VM-Exit 发生的原因为执行 vmcall 指令。知道了这些，我们便可以通过搜索 IDA 中的关键字来寻找读取 0x4402 字段的代码，IDA 中结果如下。

```
Address                Function             Instruction                       
-------                --------             -----------                       
.text:FFFFF80000219629 sub_FFFFF800002194F0                 mov     eax, 4402h
.text:FFFFF8000029F434 sub_FFFFF8000029F418                 mov     eax, 4402h
.text:FFFFF800002B6387 sub_FFFFF800002B5F68                 mov     eax, 4402h

```

         然后在 WinDbg 中分别对上面三个地址设置断点，然后继续运行系统。注意，设置断点需要在调试 hypervisor 的 WinDbg 窗口中操作。最终发现只有第一个断点被访问，逆向之，部分代码如下。

```
.text:FFFFF80000219629 ; --------------------------------------------------------
.text:FFFFF80000219629 loc_FFFFF80000219629:; CODE XREF: sub_FFFFF800002194F0+126
.text:FFFFF80000219629                 mov     eax, 4402h
.text:FFFFF8000021962E                 vmread  rcx, rax
.text:FFFFF80000219631                 mov     [rbp+57h+var_A8], rcx
.text:FFFFF80000219635
.text:FFFFF80000219635 loc_FFFFF80000219635:; CODE XREF: sub_FFFFF800002194F0+137
.text:FFFFF80000219635                 movzx   edx, cx
.text:FFFFF80000219638                 mov     r8, 0EC0191DFFDF400h
.text:FFFFF80000219642                 xor     edx, r15d
.text:FFFFF80000219645                 mov     dword ptr [rdi+4], 25h
.text:FFFFF8000021964C                 bt      r8, rdx
.text:FFFFF80000219650                 jnb     short loc_FFFFF8000021967F
.text:FFFFF80000219652                 mov     eax, cs:dword_FFFFF80000624F70
.text:FFFFF80000219658                 test    al, 1
.text:FFFFF8000021965A                 jz      short loc_FFFFF8000021966E
.text:FFFFF8000021965C                 mov     rax, gs:17010h
.text:FFFFF80000219665                 mov     r8d, [rax+2C8h]
.text:FFFFF8000021966C                 jmp     short loc_FFFFF8000021967B
.text:FFFFF8000021966E ; --------------------------------------------------------
.text:FFFFF80000219D00 ; --------------------------------------------------------
.text:FFFFF80000219D00 loc_FFFFF80000219D00:; CODE XREF: sub_FFFFF800002194F0+3AD
.text:FFFFF80000219D00                 cmp     edx, 12h
.text:FFFFF80000219D03                 jnz     loc_FFFFF80000219E37
.text:FFFFF80000219D09                 mov     rax, [rdi+30h]
.text:FFFFF80000219D0D                 mov     dword ptr [rax+130h], 6
.text:FFFFF80000219D17                 mov     rax, [rdi+88h]
.text:FFFFF80000219D1E                 cmp     dword ptr [rax+0CF8h], 3
.text:FFFFF80000219D25                 jnz     short loc_FFFFF80000219D55
.text:FFFFF80000219D27                 mov     r10b, 1
.text:FFFFF80000219D2A                 mov     [rdi+18h], r10b
.text:FFFFF80000219D2E                 mov     rax, [rdi+88h]
.text:FFFFF80000219D35                 mov     rcx, [rax+28h]
.text:FFFFF80000219D39                 mov     rax, [rcx+20h]
.text:FFFFF80000219D3D                 mov     [rdi+10h], rax
.text:FFFFF80000219D41                 xor     edx, edx
.text:FFFFF80000219D43                 mov     rcx, rdi
.text:FFFFF80000219D46                 call    sub_FFFFF800002BC17C
.text:FFFFF80000219D4B                 test    al, al
.text:FFFFF80000219D4D                 jz      loc_FFFFF80000219DEF
.text:FFFFF80000219D53                 jmp     short loc_FFFFF80000219D5C
.text:FFFFF80000219D55 ; ------------- ------------------------------------------

```

         通过上面的代码可知，代码在. text:FFFFF8000021962E 处读取出 VM-Exit 的原因，然后之后通过判断原因代码的值进行操作。代码位置. text:FFFFF80000219D00 处如果 VM-Exit 原因代码为 0x12(vmcall 造成的 VM-Exit)，便开始运行对应 VM-Exit 原因代码的操作。之后，便是一些分发操作和数据处理，然后触发 Windows 宿主机内核对应的中断处理程序并交给 Windows 宿主机内核继续进行处理。

宿主机与虚拟机内存映射

         Hyper-V 中的宿主机和虚拟机之间的内存映射是通过 EPT(Extended-Page Table) 实现的，要弄清楚宿主机和虚拟机的内存映射关系首先要弄清楚 Hyper-V 是如何使用 EPT 的。

         通过查阅 Intel 文档，得知通过调用 vmwrite 指令读取 0x201A 字段获得的值为 EPT Pointer(full)，那么我们通过使用上面的办法，在 IDA 中搜索”201AH” 看看哪里使用了这个值。

```
Address                Function             Instruction       
-------                --------             -----------       
.text:FFFFF800002A0B35 sub_FFFFF800002A0AD8 mov     ecx, 201Ah
.text:FFFFF800002A2D40 sub_FFFFF800002A2ACC mov     ecx, 201Ah
.text:FFFFF800002AB0E7 sub_FFFFF800002AAF78 mov     eax, 201Ah
.text:FFFFF800002B61AC sub_FFFFF800002B5F68 sub     ebx, 201Ah

```

         和上面一样，对前三个地址设备断点，然后继续运行代码，会发现没有一个断点被访问，这时，我们需要开启或者重启虚拟机才能触发断点，因为 EPT 初始化是在虚拟机启动过程中执行的。重启虚拟机，发现只有. text:FFFFF800002A0B35 地址被访问了，代码如下。

```
.text:FFFFF800002A0AD8 sub_FFFFF800002A0AD8 proc near  
.text:FFFFF800002A0AD8                 sub     rsp, 28h
.text:FFFFF800002A0ADC                 mov     r9, rcx
.text:FFFFF800002A0ADF                 cmp     [rcx+0A18h], edx
.text:FFFFF800002A0AE5                 jz      loc_FFFFF800002A0B6C
.text:FFFFF800002A0AEB                 mov     eax, edx
.text:FFFFF800002A0AED                 imul    r8, rax, 0E0h
.text:FFFFF800002A0AF4                 mov     [rcx+0A18h], edx
.text:FFFFF800002A0AFA                 add     r8, [rcx+0A20h]
.text:FFFFF800002A0B01                 mov     ecx, cs:dword_FFFFF80000624F70
.text:FFFFF800002A0B07                 mov     rax, [r8+4D8h]
.text:FFFFF800002A0B0E                 and     rax, 0FFFFFFFFFFFFFFDEh
.text:FFFFF800002A0B12                 or      rax, 1Eh
.text:FFFFF800002A0B16                 test    cl, 1
.text:FFFFF800002A0B19                 jz      short loc_FFFFF800002A0B35
.text:FFFFF800002A0B1B                 mov     rcx, gs:17010h
.text:FFFFF800002A0B24                 btr     dword ptr [rcx+338h], 9
.text:FFFFF800002A0B2C                 mov     [rcx+270h], rax
.text:FFFFF800002A0B33                 jmp     short loc_FFFFF800002A0B3D
.text:FFFFF800002A0B35 ; --------------------------------------------------------
.text:FFFFF800002A0B35 loc_FFFFF800002A0B35:; CODE XREF: sub_FFFFF800002A0AD8+41
.text:FFFFF800002A0B35                 mov     ecx, 201Ah
.text:FFFFF800002A0B3A                 vmwrite rcx, rax
.text:FFFFF800002A0B3D loc_FFFFF800002A0B3D:; CODE XREF: sub_FFFFF800002A0AD8+5B
.text:FFFFF800002A0B3D                 mov     edx, gs:8
.text:FFFFF800002A0B45                 mov     eax, edx
.text:FFFFF800002A0B47                 shr     eax, 6
.text:FFFFF800002A0B4A                 and     edx, 3Fh
.text:FFFFF800002A0B4D                 mov     rcx, [r8+rax*8+4E0h]
.text:FFFFF800002A0B55                 mov     eax, edx
.text:FFFFF800002A0B57                 bt      rcx, rax
.text:FFFFF800002A0B5B                 jb      short loc_FFFFF800002A0B6C
.text:FFFFF800002A0B5D                 xor     r8d, r8d
.text:FFFFF800002A0B60                 mov     rcx, r9
.text:FFFFF800002A0B63                 lea     edx, [r8+3]
.text:FFFFF800002A0B67                 call    sub_FFFFF800002A08E8
.text:FFFFF800002A0B6C loc_FFFFF800002A0B6C:; CODE XREF: sub_FFFFF800002A0AD8+D
.text:FFFFF800002A0B6C                        ; sub_FFFFF800002A0AD8+83
.text:FFFFF800002A0B6C                 add     rsp, 28h
.text:FFFFF800002A0B70                 retn
.text:FFFFF800002A0B70 sub_FFFFF800002A0AD8 endp
.text:FFFFF800002A0B70 ; --------------------------------------------------------

```

        我们通过 WinDbg 调试查看通过 vmwrite 指令在 0x201A 字段写入了什么数据。

```
Breakpoint 4 hit
hv+0x2a0b35:
fffff800`078ceb35 b91a200000      mov     ecx,201Ah
0: kd> r rax
rax=00000001ea40901e

```

         我们在 Linux 内核中写入一些数据，并且把这些数据的地址转换为物理地址并打印出来。在内核日志中显示如下。

```
$dmesg –w|grep Hello
[+]phy address of string:”AAAAAAAAAAAA......” 0x96eee000

```

         我们在写了一个 Linux 内核模块，主要内容为将一个字符串的物理地址打印到内核 log 中，将它加载进内核后运行上面代码便出现了” AAAAAAAAAAAA......” 字符串在虚拟机中的物理地址。

         下面我们在 WinDbg 中通过调试来确定虚拟机物理地址和宿主机物理地址的关系。

```
hv+0x23b7a0:
fffff800`078697a0 cc              int     3
1: kd> !vtop 1ea40901e 0x96eee000
Amd64VtoP: Virt 00000000`96eee000, pagedir 00000001`ea409000
Amd64VtoP: PML4E 00000001`ea409000
Amd64VtoP: PDPE 00000001`ea408010
Amd64VtoP: PDE 00000001`4da3a5b8
Amd64VtoP: PTE 00000001`4daf3770
Amd64VtoP: Mapped phys 00000001`972ee000
Virtual address 96eee000 translates to physical address 1972ee000.
1: kd> !db 1972ee000
#1972ee000 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee010 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee020 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee030 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee040 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee050 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee060 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA
#1972ee070 41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41 AAAAAAAAAAAAAAAA

```

         在 hv+0x2a0b35 处断下时 rax 值为 EPTP，把它作为! vtop 指令的 dirbase 参数，将虚拟机中的物理地址作为! vtop 中的 VirtualAddress 参数，便可得到虚拟机物理地址对应宿主机中的物理地址，上面调试过程中，这个地址为 0x1972ee000。通过查看物理地址内存命令! db 0x1972ee000，可以看到这段内存和虚拟机中的内容是一样的。

         我们以后便可使用上面的方法将虚拟机中的物理地址映射到宿主机物理中。

  

4.4. Windows 内核层

         在介绍这部分内容之前，这里先要声明 Windows 操作系统的版本，笔者的被调试机的 Windows 版本为 Windows10 企业版 Build 14393.rs1_release.160715-1616，不同的 Windows 操作系统版本在逆向和调试时可能代码和偏移会有所不同，所以这里我们以我的被调试机为准，下面的逆向和调试过程中，都使用的是这个版本的 Windows 系统。

         每当虚拟机发来数据时，hypervisor 层都会触发注册在 Windows 内核中的中断，WinDbg 运行! idt 命令的部分结果如下。

```
1: kd> !idt
 
Dumping IDT: ffffc680d1969c70
 
......
30: fffff803c45d88c0 nt!KiHvInterrupt
31: fffff803c45d8c30 nt!KiVmbusInterrupt0
32: fffff803c45d8f90 nt!KiVmbusInterrupt1
33: fffff803c45d92f0 nt!KiVmbusInterrupt2
34: fffff803c45d9650 nt!KiVmbusInterrupt3
......

```

         在上面的结果中，列出了五个函数。这五个函数用作处理从 hypervisor 层发来的通知，并且最终都会调用 nt! HvlRouteInterrupt 函数。nt! HvlRouteInterrupt 函数是用来将各类的通知分发到不同的函数中，再到其他模块中进行下一步的处理。下面是 nt! HvlRouteInterrupt 函数的原型。

```
HvlRouteInterrupt      HvlRouteInterrupt proc near
HvlRouteInterrupt      arg_0 = dword ptr  8
HvlRouteInterrupt
HvlRouteInterrupt                      mov     [rsp+arg_0], ecx
HvlRouteInterrupt+4                    sub     rsp, 28h
HvlRouteInterrupt+8                    movsxd  rcx, [rsp+28h+arg_0]
HvlRouteInterrupt+D                    lea     rdx, HvlpInterruptCallback
HvlRouteInterrupt+14                   add     rsp, 28h
HvlRouteInterrupt+18                   jmp     qword ptr [rdx+rcx*8]
HvlRouteInterrupt+18   HvlRouteInterrupt endp

```

         由上面的代码中可知，这段函数以 rcx 寄存器中的值为引索在 HvlpInterruptCallback 表中查表，然后调用表中函数。下面我们通过动态调试查看表中的数据，并总结出 rcx 分别对应着上文中哪些中断处理函数。nt! HvlpInterruptCallback 表中的数据如下。

```
0: kd> dps nt!HvlpInterruptCallback  
fffff802`cf620548  fffff80b`2aca29f0 winhvr!WinHvOnInterrupt  
fffff802`cf620550  fffff80b`2ac21360 vmbusr!XPartEnlightenedIsr  
fffff802`cf620558  fffff80b`2ac21360   vmbusr!XPartEnlightenedIsr  
fffff802`cf620560  fffff802`cf3976d0 nt!EmpCheckErrataList  
fffff802`cf620568  fffff802`cf3976d0 nt!EmpCheckErrataList        

```

         通过在函数 nt! HvlRouteInterrupt 下断点，每次查看 rcx 寄存器的值以及栈回溯，总结出如下结论，如表 1-7。

![](https://bbs.pediy.com/upload/attach/201711/624619_k4q7zrguhc176s7.png)

         在笔者的 Windows10 系统中，只使用了上文中 5 个中断处理函 s 数的前三个，剩余两个函数在实际的使用中没有被调用过，并且在 nt!HvlpInterruptCallback 表中偏移 0x18，0x20 为 nt!EmpCheckErrataList 函数，说明后面两个回调函数并没有初始化。即中断处理函数 nt!KiVmbusInterrupt2 和 nt!KiVmbusInterrupt3 虽然被注册但是在笔者的操作系统版本中并没有使用。

         我们继续查看函数 vmbusr!XPartEnlightenedIsr 的内部实现，它的代码如下。

```
XPartEnlightenedIsr      XPartEnlightenedIsr proc near
XPartEnlightenedIsr                      sub     rsp, 28h
XPartEnlightenedIsr+4                    mov     eax, gs:1A4h
XPartEnlightenedIsr+C                    xor     r8d, r8d        ; SystemArgument2
XPartEnlightenedIsr+F                    mov     edx, eax
XPartEnlightenedIsr+11                   lea     eax, [rcx-1]
XPartEnlightenedIsr+14                   movsxd  rcx, eax
XPartEnlightenedIsr+17                   lea     rcx, [rcx+rdx*2]
XPartEnlightenedIsr+1B                   xor     edx, edx        ; SystemArgument1
XPartEnlightenedIsr+1D                   shl     rcx, 6
XPartEnlightenedIsr+21                   add     rcx, cs:P       ; Dpc
XPartEnlightenedIsr+28                   call    cs:__imp_KeInsertQueueDpc
XPartEnlightenedIsr+2E                   cmp     cs:byte_1C0014769, 0
XPartEnlightenedIsr+35                   jnz     short loc_1C000139C
XPartEnlightenedIsr+37
XPartEnlightenedIsr+37   loc_1C0001397:
XPartEnlightenedIsr+37                   add     rsp, 28h
XPartEnlightenedIsr+3B                   retn
XPartEnlightenedIsr+3C   ; ----------------------------------------------------
XPartEnlightenedIsr+3C
XPartEnlightenedIsr+3C   loc_1C000139C:
XPartEnlightenedIsr+3C                   call    cs:__imp_HvlPerformEndOfInterrupt
XPartEnlightenedIsr+42                   jmp     short loc_1C0001397
XPartEnlightenedIsr+42   XPartEnlightenedIsr endp

```

         从上面的代码中可以看出，函数 vmbusr!XPartEnlightenedIsr 通过调用 KeInsertQueueDpc 函数来插入一个 DPC 结构体，它的作用是延迟调用 DPC 结构体中保存的函数地址。下面是 vmbusr!XPartEnlightenedIsr 调用 KeInsertQueueDpc 时的 DPC 结构体内容。

```
0: kd> dt _kdpc @rcx
win32k!_KDPC
   +0x000 TargetInfoAsUlong : 0x113
   +0x000 Type             : 0x13 ''
   +0x001 Importance       : 0x1 ''
   +0x002 Number           : 0
   +0x008 DpcListEntry     : _SINGLE_LIST_ENTRY
   +0x010 ProcessorHistory : 1
   +0x018 DeferredRoutine  : 0xfffff80b`2ac211f0     void  vmbusr!ParentRingInterruptDpc+0
   +0x020 DeferredContext  : 0xfffff80b`2ac344e0 Void
   +0x028 SystemArgument1  : (null) 
   +0x030 SystemArgument2  : (null) 
   +0x038 DpcData          : (null)
 
0: kd> dt _kdpc @rcx
win32k!_KDPC
   +0x000 TargetInfoAsUlong : 0x113
   +0x000 Type             : 0x13 ''
   +0x001 Importance       : 0x1 ''
   +0x002 Number           : 0
   +0x008 DpcListEntry     : _SINGLE_LIST_ENTRY
   +0x010 ProcessorHistory : 1
   +0x018 DeferredRoutine  : 0xfffff80b`2ac25030     void  vmbusr!ParentInterruptDpc+0
   +0x020 DeferredContext  : 0xfffff80b`2ac344e0 Void
   +0x028 SystemArgument1  : (null) 
   +0x030 SystemArgument2  : (null) 
   +0x038 DpcData          : (null)

```

         从 WinDbg 结果可知，函数 vmbusr!XPartEnlightenedIsr 将 vmbusr!ParentRingInterruptDpc 函数和 vmbusr!ParentInterruptDpc 函数插入到延迟调用队列中。

         到此为止，宿主机收到从 hypervisor 层发来的通知后，进程会经历以上流程运行到 winhvr! WinHvOnInterrupt 函数，vmbusr!ParentInterruptDpc 函数和 vmbusr!ParentRingInterruptDpc 函数中。在这里，winhvr! WinHvOnInterrupt 函数和 vmbusr!ParentInterruptDpc 函数并非用于 Hyper-V 虚拟设备数据传输之间的运行过程，攻击面较小。所以我们主要介绍 vmbusr!ParentRingInterruptDpc 函数之后的流程，由于篇幅有限，读者们可以自行对剩余两个函数进行逆向研究。

         下面我们以网卡设备 (vmswitch.sys) 为例子，介绍数据在 Windows 内核中的传递过程。下面我们先从 vmbusr!ParentRingInterruptDpc 函数开始，它的代码如下。

```
ParentRingInterruptDpc      ParentRingInterruptDpc proc near
ParentRingInterruptDpc
ParentRingInterruptDpc      var_18          = qword ptr -18h
ParentRingInterruptDpc      arg_0           = qword ptr  8
ParentRingInterruptDpc      arg_8           = qword ptr  10h
ParentRingInterruptDpc      arg_10          = qword ptr  18h
ParentRingInterruptDpc      arg_18          = qword ptr  20h
ParentRingInterruptDpc
ParentRingInterruptDpc                mov     [rsp+arg_18], rbx
ParentRingInterruptDpc+5              push    rdi
ParentRingInterruptDpc+6              sub     rsp, 30h
ParentRingInterruptDpc+A              mov     ebx, cs:dword_1C001477C
ParentRingInterruptDpc+10             xor     edi, edi
ParentRingInterruptDpc+12
ParentRingInterruptDpc+12   loc_1C0001202:
ParentRingInterruptDpc+12             mov     [rsp+38h+arg_0], rbp
ParentRingInterruptDpc+17             mov     [rsp+38h+arg_8], rsi
ParentRingInterruptDpc+1C             mov     [rsp+38h+arg_10], r14
ParentRingInterruptDpc+21
ParentRingInterruptDpc+21   loc_1C0001211: 
ParentRingInterruptDpc+21             mov     ecx, ebx
ParentRingInterruptDpc+23             call    cs:__imp_WinHvGetNextQueuedPort
ParentRingInterruptDpc+29             test    eax, eax
ParentRingInterruptDpc+2B             jz      short loc_1C000128F
ParentRingInterruptDpc+2D             lea     rdx, [rsp+38h+var_18]
ParentRingInterruptDpc+32             mov     ecx, eax
ParentRingInterruptDpc+34             call    cs:__imp_WinHvLookupPortId
ParentRingInterruptDpc+3A             mov     rbp, [rsp+38h+var_18]
ParentRingInterruptDpc+3F             xor     r14b, r14b
ParentRingInterruptDpc+42             lea     rcx, [rbp+28h]  ; SpinLock
ParentRingInterruptDpc+46             call    cs:__imp_KeAcquireSpinLockAtDpcLevel
ParentRingInterruptDpc+4C             mov     rcx, [rbp+18h]
ParentRingInterruptDpc+50             test    rcx, rcx
ParentRingInterruptDpc+53             jz      short loc_1C000126D
ParentRingInterruptDpc+55             mov     rax, [rcx]     
ParentRingInterruptDpc+55            ; vmbkmclr!KmclpVmbusIsr
ParentRingInterruptDpc+55            ; vmbkmclr!KmclpVmbusManualIsr
ParentRingInterruptDpc+58             mov     rcx, [rcx+8]    ; _QWORD
ParentRingInterruptDpc+5C             call    cs:__guard_dispatch_icall_fptr
ParentRingInterruptDpc+62             mov     edx, gs:1A4h
ParentRingInterruptDpc+6A             mov     r8d, edx
ParentRingInterruptDpc+6D             mov     rdx, cs:VmbusPerProcessorStats
ParentRingInterruptDpc+74             inc     dword ptr [rdx+r8*8+4]
ParentRingInterruptDpc+79             test    al, al
ParentRingInterruptDpc+7B             jz      short loc_1C00012A9
ParentRingInterruptDpc+7D
ParentRingInterruptDpc+7D   loc_1C000126D: ; CODE XREF: ParentRingInterruptDpc+53
ParentRingInterruptDpc+7D   ; ParentRingInterruptDpc+BF
ParentRingInterruptDpc+7D             lea     rcx, [rbp+28h]  ; SpinLock
ParentRingInterruptDpc+81            call    cs:__imp_KeReleaseSpinLockFromDpcLevel
ParentRingInterruptDpc+87             test    r14b, r14b
ParentRingInterruptDpc+8A             jnz     loc_1C0006682
ParentRingInterruptDpc+90
ParentRingInterruptDpc+90   loc_1C0001280:; CODE XREF: ParentRingInterruptDpc+54A0
ParentRingInterruptDpc+90             inc     edi
ParentRingInterruptDpc+92             cmp     edi, 200h
ParentRingInterruptDpc+98             jb      short loc_1C0001211
ParentRingInterruptDpc+9A             jmp     loc_1C0006695
ParentRingInterruptDpc+9F   ; ---------------------------------------------------
ParentRingInterruptDpc+9F
ParentRingInterruptDpc+9F   loc_1C000128F: ; CODE XREF: ParentRingInterruptDpc+2B
ParentRingInterruptDpc+9F   ; ParentRingInterruptDpc+54CD
ParentRingInterruptDpc+9F             mov     r14, [rsp+38h+arg_10]
ParentRingInterruptDpc+A4             mov     rsi, [rsp+38h+arg_8]
ParentRingInterruptDpc+A9             mov     rbp, [rsp+38h+arg_0]
ParentRingInterruptDpc+AE             mov     rbx, [rsp+38h+arg_18]
ParentRingInterruptDpc+B3             add     rsp, 30h
ParentRingInterruptDpc+B7             pop     rdi
ParentRingInterruptDpc+B8             retn
ParentRingInterruptDpc+B9   ; ---------------------------------------------------
ParentRingInterruptDpc+B9
ParentRingInterruptDpc+B9   loc_1C00012A9: ; CODE XREF: ParentRingInterruptDpc+7B
ParentRingInterruptDpc+B9             inc     dword ptr [rbp+24h]
ParentRingInterruptDpc+BC             mov     r14b, 1
ParentRingInterruptDpc+BF             jmp     short loc_1C000126D
ParentRingInterruptDpc+BF   ParentRingInterruptDpc endp

```

         从 vmbusr!ParentRingInterruptDpc 函数的代码中可以看到，在 ParentRingInterruptDpc+5C 位置有一句跳转到 rax 寄存器的中的值位置的语句。下面通过调试，查看 ParentRingInterruptDpc+55 处 rcx 中地址的内容。

```
0: kd> dps rcx
ffff8c8f`28ceac80  fffff80b`2ac84a60 vmbkmclr!KmclpVmbusManualIsr
ffff8c8f`28ceac88  ffff8c8f`2c2c95c0
ffff8c8f`28ceac90  00000000`00000000
 
0: kd> g;dps rcx
ffff8c8f`2c462c80  fffff80b`2ac82400 vmbkmclr!KmclpVmbusIsr
ffff8c8f`2c462c88  ffff8c8f`2c461010
ffff8c8f`2c462c90  00000000`00000000

```

         上面的结果说明 vmbusr!ParentRingInterruptDpc 函数使用了两个函数分发，分别调用了函数 vmbkmclr!KmclpVmbusManualIsr 和函数 vmbkmclr!KmclpVmbusIsr。这两个函数不同之处在于：vmbkmclr!KmclpVmbusIsr 是用于处理虚拟机通过 monitorpage 方式将数据发送给宿主机的函数；vmbkmclr!KmclpVmbusManualIsr 是用于处理虚拟机通过 hypercall 方法将数据发送给宿主机的函数。从上文中的介绍我们知道，虚拟机中不同的设备使用固定的方式来通知宿主机，所以这里也可以得出结论：网卡 (hv_netvsc)，硬盘(hv_storvsc) 设备会使用 vmbkmclr!KmclpVmbusIsr 函数进行之后的处理，而集成服务 (hv_utils)，键盘(hyperv_keyboard)，鼠标(hid_hyperv)，动态内存(hv_balloon)，视频(hyperv_fb) 设备会使用 vmbkmclr!KmclpVmbusManualIsr 函数进行之后的处理。vmbkmclr!KmclpVmbusManualIsr 这个分支用作准备用户态使用的数据并通知 Windows 用户态程序读写完成，所以这个分支会在下面的 Windows 应用层小节中介绍，本小节主要以 Hyper-V 虚拟网卡设备在宿主机中的驱动 (vmswitch.sys) 为例做 Windows 内核层的介绍，故小节剩余部分主要介绍 vmbkmclr!KmclpVmbusIsr 分支。

         这里也许会有人有个疑问，每当代码运行到 vmbusr!ParentRingInterruptDpc 函数时，我如何知道是虚拟机中的哪个设备发来的通知，如果不知道是什么设备又该如何单一的调试一个虚拟设备，也许其他设备发来的通知一样会被 WinDbg 截获到，会对调试造成一定的困扰。所以，在介绍 vmbkmclr!KmclpVmbusIsr 这条分支之前，我们需要在代码运行到 vmbusr!ParentRingInterruptDpc 函数时判断是哪个虚拟机中的设备发来的通知。

         为了分别不同的虚拟机中设备，笔者先修改了 Linux 内核。在 Linux 内核中，每个虚拟设备都会通过 vmbus_sendpacket 函数将数据发送至宿主机，所以只要找到每个虚拟设备代码中调用 vmbus_sendpakcet 的函数，然后在这个函数中添加一段代码，下面以 hyperv_fb.c 作为例子。

```
static inline int synthvid_send(struct hv_device *hdev,
                struct synthvid_msg *msg)
{
static atomic64_t request_id = ATOMIC64_INIT(0);
int ret;
msg->pipe_hdr.type = PIPE_MSG_DATA;
msg->pipe_hdr.size = msg->vid_hdr.size;
struct vmbus_channel *tmp_chl = hdev->channel;//新加行
  printk(KERN_INFO"[+]hyperv_fb conn_id:0x%lx\n",
tmp_chl->offermsg.connection_id);//新加行
ret = vmbus_sendpacket(hdev->channel, msg,
msg->vid_hdr.size + sizeof(struct pipe_msg_hdr),
atomic64_inc_return(&request_id),
VM_PKT_DATA_INBAND, 0);
if (ret)
pr_err("Unable to send packet via vmbus\n");
return ret;
}

```

         保存文件，然后重新编译内核，重启虚拟机。然后开机选择刚才编译过的内核，进入系统后，输入如下命令。

```
$dmesg –w|grep hyperv_fb
[+]hyperv_fb conn_id:0x10005
[+]hyperv_fb conn_id:0x10005
......

```

         上面的命令运行的结果说明，hyperv_fb 设备的 connection_id 的值为 0x10005。我们在 Linux 驱动源码中添加了将 vmbus_channel 结构体中 offermsg.connection_id 成员变量的值打印出来的功能，是为了通过 connection_id 来确定不同的虚拟设备发送的数据。像上面举的例子一样，如果是 hyperv_fb 驱动发送出去的数据，connection_id 一定是 0x10005。

         既然确定通过 connection_id 来标记不同设备的数据，那么在宿主机驱动中，又该如何通过 connection_id 的值知道是哪个设备发送的数据呢？我们通过下面的调试过程来弄清楚这件事。首先先在函数 ParentRingInterruptDpc+0x55 处设置断点。

```
0: kd> bp vmbusr!ParentRingInterruptDpc+0x55;g;
Breakpoint 0 hit
vmbusr!ParentRingInterruptDpc+0x55:
fffff80b`2ac21245 488b01          mov     rax,qword ptr [rcx]
 
0: kd> dd rbp+38
ffff8c8f`2c234338  00010005 00000000 00010000 00000000
ffff8c8f`2c234348  00000000 00000000 00000000 00000000
ffff8c8f`2c234358  00000000 00000002 00000005 00000000
ffff8c8f`2c234368  00000000 00000000 00000000 00000000
ffff8c8f`2c234378  00000000 00000000 00000000 00000000
ffff8c8f`2c234388  000347e6 00000000 0219000a 656c6946
ffff8c8f`2c234398  5a34ed22 eebe3a1a 2c2344e8 ffff8c8f
ffff8c8f`2c2343a8  00000001 00000000 00000400 00000180

```

         WinDbg 在 ParentRingInterruptDpc+0x55 处成功断下，这时我们查看 rbp+0x38 位置处的数据，会发现地址 0x ffff8c8f2c234338 处的 4 字节数据便是虚拟机驱动中 connection_id 的值。我们可以通过这种办法判断当运行到 ParentRingInterruptDpc 函数时传输着什么虚拟设备的数据，方便之后跟踪调试特定设备的数据传输流程，减小调试的难度。

         言归正传，我们继续介绍 vmbkmclr!KmclpVmbusIsr 函数之后的流程。在笔者的 Linux 虚拟机环境中，虚拟网卡的设备是 hv_netvsc，它发送数据时 vmbus_channel 结构体中 offermsg.connection_id 的值为 0x10008，这个值可能在不同的机器上会不一样。下面我们要跟踪网卡数据在宿主机驱动中的流向，首先先要逆向 vmbkmclr!KmclpVmbusIsr 函数，代码如下。

```
KmclpVmbusIsr      KmclpVmbusIsr   proc near        
KmclpVmbusIsr      var_48          = qword ptr -48h
KmclpVmbusIsr      var_38          = byte ptr -38h
KmclpVmbusIsr      var_30          = dword ptr -30h
KmclpVmbusIsr      var_2C          = dword ptr -2Ch
KmclpVmbusIsr      var_20          = qword ptr -20h
KmclpVmbusIsr      arg_8           = qword ptr  10h
KmclpVmbusIsr      arg_10          = qword ptr  18h
KmclpVmbusIsr
KmclpVmbusIsr                      mov     [rsp+arg_10], rbx
KmclpVmbusIsr+5                    push    rsi
KmclpVmbusIsr+6                    push    rdi
KmclpVmbusIsr+7                    push    r14
KmclpVmbusIsr+9                    sub     rsp, 50h
KmclpVmbusIsr+D                    mov     rax, cs:__security_cookie
KmclpVmbusIsr+14                   xor     rax, rsp
KmclpVmbusIsr+17                   mov     [rsp+68h+var_20], rax
KmclpVmbusIsr+1C                   inc     qword ptr [rcx+5C0h] ; transaction_id加1
KmclpVmbusIsr+23                   lea     rsi, [rcx+40h]
KmclpVmbusIsr+27                   mov     eax, [rsi+8Ch]
KmclpVmbusIsr+2D                   mov     rbx, rcx
KmclpVmbusIsr+30                   add     eax, [rsi+48h]
KmclpVmbusIsr+33                   mov     ecx, [rsi+4Ch]
KmclpVmbusIsr+36   loc_1C0002436:
KmclpVmbusIsr+36                   mov     [rsp+68h+arg_8], rbp
KmclpVmbusIsr+3B                   sub     eax, ecx
KmclpVmbusIsr+3D                   jz      loc_1C00059B2
KmclpVmbusIsr+43                   lea     eax, [rcx+1]
KmclpVmbusIsr+46                   mov     r14b, 1
KmclpVmbusIsr+49                   mov     [rsi+4Ch], eax
KmclpVmbusIsr+4C   loc_1C000244C: ; CODE XREF: KmclpVmbusIsr+35BB
KmclpVmbusIsr+4C                   lea     rcx, [rbx+380h] ; SpinLock
KmclpVmbusIsr+53                   xor     dil, dil
KmclpVmbusIsr+56                   call    cs:__imp_KeAcquireSpinLockAtDpcLevel
KmclpVmbusIsr+5C                   cmp     dword ptr [rbx+388h], 4
KmclpVmbusIsr+63                   jnz     short loc_1C000248D
KmclpVmbusIsr+65                   mov     rax, [rbx+58h]
KmclpVmbusIsr+69                   mov     dil, 1
KmclpVmbusIsr+6C                   mov     dword ptr [rax+8], 1
KmclpVmbusIsr+73                   mov     dword ptr [rbx+388h], 1
KmclpVmbusIsr+7D                   mov     rax, gs:188h
KmclpVmbusIsr+86                   mov     [rbx+3A8h], rax
KmclpVmbusIsr+8D   loc_1C000248D: ; CODE XREF: KmclpVmbusIsr+63
KmclpVmbusIsr+8D                   lea     rcx, [rbx+380h] ; SpinLock
KmclpVmbusIsr+94                   call    cs:__imp_KeReleaseSpinLockFromDpcLevel
KmclpVmbusIsr+9A                   test    dil, dil
KmclpVmbusIsr+9D                   jz      loc_1C0002546
KmclpVmbusIsr+A3                   cmp     qword ptr [rbx+4F0h], 0FFFFFFFFFFFFFFFFh
KmclpVmbusIsr+AB                   jz      loc_1C00059C0
KmclpVmbusIsr+B1                   mov     rax, 0FFFFF78000000320h
KmclpVmbusIsr+BB                   mov     rax, [rax]
KmclpVmbusIsr+BE                   mov     rdx, rax
KmclpVmbusIsr+C1                   sub     rdx, [rbx+4E0h]
KmclpVmbusIsr+C8                   jnz     loc_1C000257B
KmclpVmbusIsr+CE   loc_1C00024CE: ; CODE XREF: KmclpVmbusIsr+1A6
KmclpVmbusIsr+CE                                ; KmclpVmbusIsr+1DA
KmclpVmbusIsr+CE                   mov     rax, [rbx+4F0h]
KmclpVmbusIsr+D5                   cmp     [rbx+4E8h], rax
KmclpVmbusIsr+DC                   jnb     loc_1C00025DF
KmclpVmbusIsr+E2                   lea     rcx, [rsp+68h+var_38]
KmclpVmbusIsr+E7                   call    cs:__imp_KeQueryDpcWatchdogInformation
KmclpVmbusIsr+ED                   test    eax, eax
KmclpVmbusIsr+EF                   js      short loc_1C000250A
KmclpVmbusIsr+F1                   imul    ecx, [rsp+68h+var_30], 1Eh
KmclpVmbusIsr+F6                   mov     eax, 51EB851Fh
KmclpVmbusIsr+FB                   mul     ecx
KmclpVmbusIsr+FD                   shr     edx, 5
KmclpVmbusIsr+100                  cmp     [rsp+68h+var_2C], edx
KmclpVmbusIsr+104                  jb      loc_1C00025DF
KmclpVmbusIsr+10A  loc_1C000250A: ; CODE XREF: KmclpVmbusIsr+EF
KmclpVmbusIsr+10A                  rdtsc
KmclpVmbusIsr+10C                  shl     rdx, 20h
KmclpVmbusIsr+110                  mov     r8d, 1Eh
KmclpVmbusIsr+116                  or      rax, rdx
KmclpVmbusIsr+119                  mov     rcx, rbx
KmclpVmbusIsr+11C                  mov     dl, 2
KmclpVmbusIsr+11E                  mov     rdi, rax
KmclpVmbusIsr+121                  call    InpFillAndProcessQueue
KmclpVmbusIsr+126                  mov     r8d, eax
KmclpVmbusIsr+129                  rdtsc
KmclpVmbusIsr+12B                  shl     rdx, 20h
KmclpVmbusIsr+12F                  or      rax, rdx
KmclpVmbusIsr+132                  sub     rax, rdi
KmclpVmbusIsr+135                  add     [rbx+4E8h], rax
KmclpVmbusIsr+13C  loc_1C000253C: ; CODE XREF: KmclpVmbusIsr+35D3
KmclpVmbusIsr+13C                  mov     dl, 1
KmclpVmbusIsr+13E                  mov     rcx, rbx
KmclpVmbusIsr+141                  call    InpTransitionRunningQueue
KmclpVmbusIsr+146  loc_1C0002546: ; CODE XREF: KmclpVmbusIsr+9D
KmclpVmbusIsr+146                               ; KmclpVmbusIsr+208
KmclpVmbusIsr+146                  mov     edx, [rbx+0C8h]
KmclpVmbusIsr+14C                  mov     rbp, [rsp+68h+arg_8]
KmclpVmbusIsr+151                  test    edx, edx
KmclpVmbusIsr+153                  jnz     sub_1C0005A00
KmclpVmbusIsr+159  loc_1C0002559: 
KmclpVmbusIsr+159                  movzx   eax, r14b
KmclpVmbusIsr+15D                  mov     rcx, [rsp+68h+var_20]
KmclpVmbusIsr+162                  xor     rcx, rsp
KmclpVmbusIsr+165                  call    __security_check_cookie
KmclpVmbusIsr+16A                  mov     rbx, [rsp+68h+arg_10]
KmclpVmbusIsr+172                  add     rsp, 50h
KmclpVmbusIsr+176                  pop     r14
KmclpVmbusIsr+178                  pop     rdi
KmclpVmbusIsr+179                  pop     rsi
KmclpVmbusIsr+17A                  retn

```

         通过 WinDbg 单步跟踪, 我们发现 vmbkmclr!KmclpVmbusIsr 函数会调用函数 vmbkmclr!InpFillAndProcessQueue。继续单步跟踪函数 vmbkmclr!InpFillAndProcessQueue 发现，vmbkmclr!InpFillAndProcessQueue 函数中会直接调用 vmswitch 模块中的函数。在 InpFillAndProcessQueue+16E 处会调用 vmswitch!VmsVmNicPvtKmclProcessPacket 函数，这个函数 vmswitch 模块用于处理发来的数据；在 InpFillAndProcessQueue+2CA 处调用 vmswitch!VmsVmNicPvtKmclProcessingComplete 函数，这个函数也同样用于处理虚拟机发来的数据。这样，便从 VMBus 总线过渡到 vmswitch 之类的虚拟设备驱动中。之后在 vmswitch 驱动中的操作便是解析数据，并对解析后的数据进行下一步操作。

         以上便是 Hyper-V 将虚拟机中数据传递给 Windows 内核中 Hyper-V 驱动组件 (如 vmswitch.sys) 的流程，为了便于理解，我们将以上的流程概括成图 1-33。

![](https://bbs.pediy.com/upload/attach/201711/624619_zf7hsrxujc8epl1.png)

图 1-33

4.5. Windows 应用层

         这一小节内容为 Hyper-V 把虚拟机中数据传递给 Windows 用户态下的 Hyper-V 组件的流程，我们会以 hyperv_fb 设备为例子，介绍数据从 Linux 虚拟机中的 hyperv_fb 设备到 Hyper-V 用户态组件 vmuidevices.dll 的过程。这个小节名字虽为 Windows 应用层，但实际上很大部分介绍还是在 Windows 内核态下。

         首先我们继续上面小节的部分，介绍 vmbkmclr!KmclpVmbusManualIsr 这个分支之后的流程。vmbkmclr!KmclpVmbusManualIsr 代码如下。

```
KmclpVmbusManualIsr      KmclpVmbusManualIsr proc near       
KmclpVmbusManualIsr
KmclpVmbusManualIsr                      push    rbx
KmclpVmbusManualIsr+2                    sub     rsp, 20h
KmclpVmbusManualIsr+6                    mov     rax, [rcx+710h]
KmclpVmbusManualIsr+D                    mov     rbx, rcx
KmclpVmbusManualIsr+10                   inc     qword ptr [rcx+5C0h]
KmclpVmbusManualIsr+17                   call    cs:__guard_dispatch_icall_fptr
KmclpVmbusManualIsr+17                   ;call   vmbusr!PipeEvtChannelSignalArrived
KmclpVmbusManualIsr+1D                   test    al, al
KmclpVmbusManualIsr+1F                   jz      loc_1C00066FC
KmclpVmbusManualIsr+25                   mov     al, 1
KmclpVmbusManualIsr+27
KmclpVmbusManualIsr+27   loc_1C0004A87:  ; CODE XREF: KmclpVmbusManualIsr+1CA4
KmclpVmbusManualIsr+27                   add     rsp, 20h
KmclpVmbusManualIsr+2B                   pop     rbx
KmclpVmbusManualIsr+2C                   retn
KmclpVmbusManualIsr+2C   KmclpVmbusManualIsr endp

```

         函数 vmbkmclr!KmclpVmbusManualIsr 调用函数 vmbusr!PipeEvtChannelSignalArrived，下面为函数 vmbusr!PipeEvtChannelSignalArrived 的代码。

```
PipeEvtChannelSignalArrived      PipeEvtChannelSignalArrived proc near 
PipeEvtChannelSignalArrived      var_18          = qword ptr -18h
PipeEvtChannelSignalArrived      arg_0           = qword ptr  8
PipeEvtChannelSignalArrived
PipeEvtChannelSignalArrived              mov     [rsp+arg_0], rbx
PipeEvtChannelSignalArrived+5            push    rdi
PipeEvtChannelSignalArrived+6            sub     rsp, 30h
PipeEvtChannelSignalArrived+A            call    cs:__imp_VmbChannelGetPointer 
PipeEvtChannelSignalArrived+A                   ; VmbChannelGetPointer proc near
PipeEvtChannelSignalArrived+A                   ; mov     rax, [rcx+7E8h]
PipeEvtChannelSignalArrived+A                   ; retn
PipeEvtChannelSignalArrived+A                    ; VmbChannelGetPointer endp
PipeEvtChannelSignalArrived+10            mov     rcx, rax        ; SpinLock
PipeEvtChannelSignalArrived+13            mov     rbx, rax
PipeEvtChannelSignalArrived+16        call    cs:__imp_KeAcquireSpinLockAtDpcLevel
PipeEvtChannelSignalArrived+1C            mov     ecx, [rbx+0CCh]
PipeEvtChannelSignalArrived+22            add     ecx, [rbx+88h]
PipeEvtChannelSignalArrived+28            mov     eax, [rbx+8Ch]
PipeEvtChannelSignalArrived+2E            cmp     ecx, eax
PipeEvtChannelSignalArrived+30            jz      short loc_1C0001920
PipeEvtChannelSignalArrived+32            lea     ecx, [rax+1]
PipeEvtChannelSignalArrived+35            mov     dil, 1      
PipeEvtChannelSignalArrived+38            mov     [rbx+8Ch], ecx
PipeEvtChannelSignalArrived+3E            jmp     short loc_1C0001923
PipeEvtChannelSignalArrived+40   ; ----------------------------------------------
PipeEvtChannelSignalArrived+40   loc_1C0001920: 
PipeEvtChannelSignalArrived+40            xor     dil, dil
PipeEvtChannelSignalArrived+43
PipeEvtChannelSignalArrived+43  loc_1C0001923: 
PipeEvtChannelSignalArrived+43            mov     rcx, cs:WPP_GLOBAL_Control
PipeEvtChannelSignalArrived+4A            lea     rax, WPP_GLOBAL_Control
PipeEvtChannelSignalArrived+51            cmp     rcx, rax
PipeEvtChannelSignalArrived+54            jz      short loc_1C0001969
PipeEvtChannelSignalArrived+56            test    dword ptr [rcx+2Ch], 100000h
PipeEvtChannelSignalArrived+5D            jz      short loc_1C0001969
PipeEvtChannelSignalArrived+5F            cmp     byte ptr [rcx+29h], 5
PipeEvtChannelSignalArrived+63            jb      short loc_1C0001969
PipeEvtChannelSignalArrived+65            mov     r8, [rbx+120h]
PipeEvtChannelSignalArrived+6C            mov     edx, 18h
PipeEvtChannelSignalArrived+71            mov     rcx, [rcx+18h]
PipeEvtChannelSignalArrived+75            mov     r9, rbx
PipeEvtChannelSignalArrived+78            mov     [rsp+38h+var_18], r8
PipeEvtChannelSignalArrived+7D                                                      lea     r8, WPP_94815654934f31e3788ac4bb387d0b84_Traceguids
PipeEvtChannelSignalArrived+84            call    WPP_SF_qq
PipeEvtChannelSignalArrived+89   loc_1C0001969:
PipeEvtChannelSignalArrived+89            mov     rcx, rbx        ; SpinLock
PipeEvtChannelSignalArrived+8C            call    PipeProcessDeferredIosAndUnlock
PipeEvtChannelSignalArrived+91            mov     rbx, [rsp+38h+arg_0]
PipeEvtChannelSignalArrived+96            movzx   eax, dil
PipeEvtChannelSignalArrived+9A            add     rsp, 30h
PipeEvtChannelSignalArrived+9E            pop     rdi
PipeEvtChannelSignalArrived+9F            retn
PipeEvtChannelSignalArrived+9F   PipeEvtChannelSignalArrived endp

```

         从上面 vmbusr!PipeEvtChannelSignalArrived 函数的代码可以很简洁的看出，它主要调用了一个函数，即 vmbusr!PipeProcessDeferredIosAndUnlock。我们继续跟进这个函数的内容，它的代码如下。

```
PipeProcessDeferredIosAndUnlock      PipeProcessDeferredIosAndUnlock proc near
PipeProcessDeferredIosAndUnlock      var_28          = qword ptr -28h
PipeProcessDeferredIosAndUnlock      var_18          = qword ptr -18h
PipeProcessDeferredIosAndUnlock      var_10          = qword ptr -10h
PipeProcessDeferredIosAndUnlock      arg_0           = qword ptr  8
PipeProcessDeferredIosAndUnlock
PipeProcessDeferredIosAndUnlock               mov     [rsp+arg_0], rbx
PipeProcessDeferredIosAndUnlock+5             push    rdi
PipeProcessDeferredIosAndUnlock+6             sub     rsp, 40h
PipeProcessDeferredIosAndUnlock+A             mov     rdi, rcx
PipeProcessDeferredIosAndUnlock+D             mov     rcx, cs:WPP_GLOBAL_Control
PipeProcessDeferredIosAndUnlock+14            lea     rax, WPP_GLOBAL_Control
PipeProcessDeferredIosAndUnlock+1B            cmp     rcx, rax
PipeProcessDeferredIosAndUnlock+1E            jz      short loc_1C00087AB
PipeProcessDeferredIosAndUnlock+53   loc_1C00087AB:
PipeProcessDeferredIosAndUnlock+53            lea     rax, [rsp+48h+var_18]
PipeProcessDeferredIosAndUnlock+58            mov     rcx, rdi
PipeProcessDeferredIosAndUnlock+5B            mov     [rsp+48h+var_10], rax
PipeProcessDeferredIosAndUnlock+60            lea     rdx, [rsp+48h+var_18]
PipeProcessDeferredIosAndUnlock+65            lea     rax, [rsp+48h+var_18]
PipeProcessDeferredIosAndUnlock+6A            mov     [rsp+48h+var_18], rax
PipeProcessDeferredIosAndUnlock+6F            call    PipeProcessDeferredReadWrite
PipeProcessDeferredIosAndUnlock+74            mov     rcx, rdi        ; SpinLock
PipeProcessDeferredIosAndUnlock+77            mov     bl, al
PipeProcessDeferredIosAndUnlock+79     call cs:__imp_KeReleaseSpinLockFromDpcLevel
PipeProcessDeferredIosAndUnlock+7F            test    bl, bl
PipeProcessDeferredIosAndUnlock+81            jz      short loc_1C00087E8
PipeProcessDeferredIosAndUnlock+83            mov     rcx, [rdi+100h]
PipeProcessDeferredIosAndUnlock+8A           call cs:__imp_VmbChannelSendInterrupt
PipeProcessDeferredIosAndUnlock+90   loc_1C00087E8: 
PipeProcessDeferredIosAndUnlock+90            lea     rcx, [rsp+48h+var_18]
PipeProcessDeferredIosAndUnlock+95            call    PipeCompleteIrpList
PipeProcessDeferredIosAndUnlock+9A            mov     rbx, [rsp+48h+arg_0]
PipeProcessDeferredIosAndUnlock+9F            add     rsp, 40h
PipeProcessDeferredIosAndUnlock+A3            pop     rdi
PipeProcessDeferredIosAndUnlock+A4            retn
PipeProcessDeferredIosAndUnlock+A4   PipeProcessDeferredIosAndUnlock endp

```

         通过上面的代码，可以看出代码运行到这个函数，会先将从虚拟机传来的数据拿出来，放在驱动的缓冲区中，这个过程调用的函数是 vmbusr!PipeProcessDeferredReadWrite。取完数据之后便调用 vmbusr!PipeCompleteIrpList 函数，vmbusr!PipeCompleteIrpList 函数中会调用函数 IofCompleteRequest，用户态 readfile 函数返回，随后用户态组件处理虚拟机发来的数据。这个过程比较像 Hyper-V 的 Linux 内核驱动里__vmbus_recvpacket 函数，笔者在逆向时确实也受到 Linux 驱动的启发。

         我们以 vmuidevices.dll 组件举例，当 Windows 内核态中 vmbusr!PipeCompleteIrpList 函数调用函数 IofCompleteRequest 之后，vmuidevices.dll 文件中的 vmuidevices!VMBusPipeTransportImpl<VMBusPipeIO,VMBusPipeServerDisposition>::IoOperation 函数中调用的 readfile 函数便会返回，代码继续运行。此时 readfile 的 lpBuffer 参数指向的内存中便是从虚拟机发来的数据，vmuidevices.dll 组件之后会对这些数据进行解析，并根据数据进行不同的操作。

         以上，便是数据从虚拟机发送至宿主机中的用户态组件的过程。和上面小节一样，我们同样用一张图总结，如图 1-34。

![](https://bbs.pediy.com/upload/attach/201711/624619_7gppt6pp518p2kl.png)

图 1-34

______________________________________________________________________________________________

本文如需引用转载请联系本文作者，看雪 ID：ifyou

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

上传的附件：

*   [4.2.pdf](javascript:void(0)) （991.51kb，75 次下载）