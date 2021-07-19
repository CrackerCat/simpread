> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268489.htm)

> [原创]VMProtect 分析（三）

[VmProtect 分析（一）](https://bbs.pediy.com/thread-268377.htm)  

=============================================================

[VmProtect 分析（二）](https://bbs.pediy.com/thread-268435.htm)
==========================================================

Tls 回调函数（下）
===========

参考上节的跟踪记录 vm_tls.txt，可以看到第 117 行和第 290 行的 VmCALL 将代码分成 3 块，标记为 Chunk1 - Chunk3，我们先看下 VmCALL 的实现，再分别分析这 3 块代码。

VmCALL
------

<table valign="top"><tbody><tr><td><p>shrd rsi,rbx,cl</p></td><td><br></td></tr><tr><td><p>mov esi,dword ptr ss:[rbp]</p></td><td><p>Esi = 栈顶 DWORD</p></td></tr><tr><td><p>clc</p></td><td><br></td></tr><tr><td><p>stc</p></td><td><br></td></tr><tr><td><p>add rbp,8</p></td><td><br></td></tr><tr><td><p>jmp &nbsp; vmp_userdebugger.7FF69E7403F7</p></td><td><br></td></tr><tr><td><p>bsf ax,di</p></td><td><br></td></tr><tr><td><p>sar r12d,cl</p></td><td><br></td></tr><tr><td><p>inc r12d</p></td><td><br></td></tr><tr><td><p>shl ch,5</p></td><td><br></td></tr><tr><td><p>lea r12,qword ptr ds:[7FF69E74065C]</p></td><td><p>R12 固定取 7FF69E74065C</p></td></tr><tr><td><p>rcl al,7</p></td><td><br></td></tr><tr><td><p>not bh</p></td><td><br></td></tr><tr><td><p>mov rax,100000000</p></td><td><br></td></tr><tr><td><p>rcr rbx,10</p></td><td><br></td></tr><tr><td><p>add rsi,rax</p></td><td><p>Rsi += 100000000</p></td></tr><tr><td><p>xchg al,bh</p></td><td><br></td></tr><tr><td><p>bsr bx,dx</p></td><td><br></td></tr><tr><td><p>shld bx,sp,cl</p></td><td><br></td></tr><tr><td><p>rcr ebx,cl</p></td><td><br></td></tr><tr><td><p>mov rbx,rsi</p></td><td><p>Rbx = Rsi</p></td></tr><tr><td><p>movsx cx,cl</p></td><td><br></td></tr><tr><td><p>add rsi,qword ptr ss:[rbp]</p></td><td><p>Rsi += 栈顶 QWORD, 注意此处未调整 RBP 栈指针</p></td></tr></tbody></table>

可以看到，VmCALL 取栈中 DWORD 作为基数计算 RBX 和 RSI，我们第一篇分析过，RSI 指向字节码缓冲区，RBX 为解密 Seed，也就是说每个 Chunk 都有自己的 RSI 和 RBX。

Chunk1
------

在继续分析 Chunk 之前，可先参考下节 Nor Gate 说明，其对用到的运算的 Nor 变换做了详细说明，下面的分析不在赘述。

[Anakin] VmPOP V_98                 ;V_98 = $HandlerBase

[Anakin] VmPUSH FFFFFFFF9F5A5C32

[Anakin] VmADD

[Anakin] VmPOP V_40

[Anakin] VmPOP V_B8

[Anakin] VmPOP V_28

[Anakin] VmPOP V_18

[Anakin] VmPOP V_00

[Anakin] VmPOP V_78

[Anakin] VmPOP V_A0

[Anakin] VmPOP V_90

[Anakin] VmPOP V_40

[Anakin] VmPOP V_20

[Anakin] VmPOP V_68

[Anakin] VmPOP V_50

[Anakin] VmPOP V_58

[Anakin] VmPOP V_30

[Anakin] VmPOP V_B0

[Anakin] VmPOP V_38

[Anakin] VmPOP V_48

[Anakin] VmPOP V_70

[Anakin] VmPOP V_88

[Anakin] VmPOP V_10

[Anakin] VmPOP V_A8

[Anakin] VmPUSH 0000000064765E24    ; 压栈分支 1 标识

[Anakin] VmPUSHB8 00

[Anakin] VmPUSH 000000014018B3E7

[Anakin] VmPUSH V_98

[Anakin] VmADD

[Anakin] VmPOP V_08

[Anakin] VmREADB                    ;b = BYTE:[000000014018B3E7 + $HandlerBase]

[Anakin] VmSBP

[Anakin] VmREADB

[Anakin] VmNOTANDB                  ;b = ~b

[Anakin] VmPOP V_60         

[Anakin] VmADDB                     ;b = 00 + b

[Anakin] VmPOP V_10                 ;V_10 = eflags

[Anakin] VmSBP

[Anakin] VmREADB

[Anakin] VmNOTANDB                  ;b = ~b

[Anakin] VmPOP V_80                 ;V_80 = eflags

[Anakin] VmPOPW8 V_60               ;V_60 = b  

[Anakin] VmPUSH V_10

[Anakin] VmPUSH V_10

[Anakin] VmNOTAND                   ;d1 = NOTAND(V_10, V_10)            => d1 = ~V_10

[Anakin] VmPOP V_60

[Anakin] VmPUSH FFFFFFFFFFFFF7EA

[Anakin] VmNOTAND                   ;d1 = NOTAND(d1, FFFFF7EA)          => d1 = Nor(~V_10, ~00000815) = V_10 & 00000815

[Anakin] VmPOP V_08

[Anakin] VmPUSH V_80                

[Anakin] VmPUSH V_80

[Anakin] VmNOTAND                   ;d2 = NOTAND(V_80, V_80)            => d2 = ~V_80

[Anakin] VmPOP V_60

[Anakin] VmPUSH 0000000000000815

[Anakin] VmNOTAND                   ;d2 = NOTAND(d2, 00000815)          => d2 = Nor(~V_80, ~FFFFF7EA) = V_80 & FFFFF7EA

[Anakin] VmPOP V_08

[Anakin] VmADD

[Anakin] VmPOP V_08

[Anakin] VmPOP V_70                 ;V_70 = d1 + d2                     => V_70 = EFLAGS(BYTE:[000000014018B3E7 + $HandlerBase] - 0)

[Anakin] VmPUSH 0000000064766651    ; 压栈分支 2 标识

[Anakin] VmSBP                      ; 压栈栈顶指针，用于后文选择分支

[Anakin] VmPUSHB8 03

[Anakin] VmPUSHD 000000BF

[Anakin] VmPUSH V_70

[Anakin] VmNOTAND                   ;q = CDQ(NOTAND(V_70, 000000BF))    => ZF == 0 ? 0b1000000 : 0 

[Anakin] VmPOP V_68

[Anakin] VmSHR                      ;q = SHR(q, 3)                      => ZF == 0 ? 8 : 0 

[Anakin] VmPOP V_08

[Anakin] VmADD                      ;q += SavedRBP (上文压栈的栈顶指针，选择分支)

[Anakin] VmPOP V_08

[Anakin] VmREADQ

[Anakin] VmPOP V_A8                 ;V_A8 = QWORD:[q]（取分支标识）

[Anakin] VmPOP V_68

[Anakin] VmPOP V_08

[Anakin] VmPUSH V_A8

[Anakin] VmPOPD V_A8                ;V_A8 = CQD(V_A8)

[Anakin] VmPUSHD V_A8               

[Anakin] VmSBP

[Anakin] VmREADD

[Anakin] VmNOTANDD                  ;d1 = NOTAND(V_A8, V_A8)

[Anakin] VmPOP V_08

[Anakin] VmPUSHD DB91AA8C

[Anakin] VmNOTANDD                  ;d1 = NOTAND(d1, DB91AA8C)          => d1 = Nor(~V_A8, ~246E5573)

[Anakin] VmPOP V_68

[Anakin] VmPUSHD 246E5573

[Anakin] VmPUSHD V_A8

[Anakin] VmNOTANDD                  ;d2 = NOTAND(V_A8, 246E5573)        => d2 = Nor(V_A8, 246E5573)

[Anakin] VmPOP V_60

[Anakin] VmNOTANDD

[Anakin] VmPOP V_60

[Anakin] VmPOP V_08                 ;V_08 = NOTAND(d2, d1)              => V_08 = Nor(d1, d2) = V_A8 ^ 246E5573 （分支标识解密）

[Anakin] VmPUSH V_18

[Anakin] VmPUSH V_98

[Anakin] VmPUSH V_60

[Anakin] VmPUSH V_00

[Anakin] VmPUSH V_88

[Anakin] VmPUSH V_50

[Anakin] VmPUSH V_30

[Anakin] VmPUSH V_B0

[Anakin] VmPUSH V_20

[Anakin] VmPUSH V_28

[Anakin] VmPUSH V_38

[Anakin] VmPUSH V_78

[Anakin] VmPUSH V_A0

[Anakin] VmPUSH V_90

[Anakin] VmPUSH V_58

[Anakin] VmPUSH V_48

[Anakin] VmPUSH V_40

[Anakin] VmPUSH V_18

[Anakin] VmPUSH V_70

[Anakin] VmPUSH V_B8

[Anakin] VmPUSH 0000000060A5A3CE

[Anakin] VmADD

[Anakin] VmPOP V_60

[Anakin] VmPUSH V_98

[Anakin] VmPUSH V_08                ; 压栈解码后的分支标识                           

[Anakin] VmCALL                      ; 调用选择分支

等价逻辑：

If (*(BYTE*)(000000014018B3E7 + $HandlerBase) != 0)

{

// 未执行

VmCALL 40180B57

}

Else

{

// 即 Chunk2

VmCALL 40183322

}

Chunk2
------

[Anakin] VmPOP V_90                     ;V_90 = $HandlerBase

[Anakin] VmPUSH FFFFFFFF9F5A5C32

[Anakin] VmADD

[Anakin] VmPOP V_20

[Anakin] VmPOP V_00

[Anakin] VmPOP V_70

[Anakin] VmPOP V_80

[Anakin] VmPOP V_60

[Anakin] VmPOP V_98

[Anakin] VmPOP V_38

[Anakin] VmPOP V_48

[Anakin] VmPOP V_28

[Anakin] VmPOP V_18

[Anakin] VmPOP V_30

[Anakin] VmPOP V_10

[Anakin] VmPOP V_88

[Anakin] VmPOP V_08

[Anakin] VmPOP V_A8

[Anakin] VmPOP V_40

[Anakin] VmPOP V_20

[Anakin] VmPOP V_68

[Anakin] VmPOPD V_78                    ;V_78 = eflags

[Anakin] VmPUSHD V_78

[Anakin] VmPUSHD V_78

[Anakin] VmNOTANDD

[Anakin] VmPOP V_B0

[Anakin] VmPUSHD DB91AA8C

[Anakin] VmNOTANDD

[Anakin] VmPOP V_B8

[Anakin] VmPUSHD 246E5573

[Anakin] VmPUSHD V_78

[Anakin] VmNOTANDD

[Anakin] VmPOP V_50

[Anakin] VmNOTANDD

[Anakin] VmPOP V_B0

[Anakin] VmPOP V_A0                     ;V_A0 = V_78 ^ 246E5573

[Anakin] VmPOP V_58

[Anakin] VmPOP V_B8

[Anakin] VmPUSH V_70

[Anakin] VmPUSH V_88

[Anakin] VmPUSH V_08

[Anakin] VmPUSH V_48

[Anakin] VmPUSH V_98

[Anakin] VmPUSH V_80

[Anakin] VmPUSH V_A8

[Anakin] VmPUSH 000000000CABFA9E        ;PUSH Branch1

[Anakin] VmPUSH 000000014018B3E7

[Anakin] VmPUSH V_90

[Anakin] VmADD

[Anakin] VmPOP V_50                     ;PUSH (V_90 + 000000014018B3E7)

[Anakin] VmPUSH 0000000140000000

[Anakin] VmPUSH V_90

[Anakin] VmADD

[Anakin] VmPOP V_58

[Anakin] VmPOP V_50                     ;V_50 = V_90 + 0000000140000000         => V_50 = PIMAGE_DOS_HEADER

[Anakin] VmPUSH V_50

[Anakin] VmPUSHD 0000003C

[Anakin] VmADD

[Anakin] VmPOP V_58                     ;PUSH (V_50 + 0000003C)

[Anakin] VmREADD

[Anakin] VmPOPD V_88                    ;V_88 = DWORD:[BP]                      => V_88 = PIMAGE_DOS_HEADER->e_lfanew

[Anakin] VmPUSH 0000000000000000

[Anakin] VmPOPD V_8C

[Anakin] VmPUSH V_88

[Anakin] VmPUSH V_50

[Anakin] VmADD

[Anakin] VmPOP V_A8                     ;PUSH (V_50 + V_88)                     => PUSH PIMAGE_NT_HEADERS64

[Anakin] VmSBP

[Anakin] VmREADQ        

[Anakin] VmPOP V_B0   

[Anakin] VmPUSHD 00000028               

[Anakin] VmADD

[Anakin] VmPOP V_A8                     ;PUSH (PIMAGE_NT_HEADERS64 + 00000028)  => PUSH PIMAGE_NT_HEADERS64->AddressOfEntryPoint

[Anakin] VmREADD

[Anakin] VmPOPD V_B0                    ;V_B0 = AddressOfEntryPoint

[Anakin] VmPUSH 0000000000000000

[Anakin] VmPOPD V_B4                    ;V_B4 = 0

[Anakin] VmPUSH V_50

[Anakin] VmPUSH V_B0

[Anakin] VmADD

[Anakin] VmPOP V_A8

[Anakin] VmPOP V_A8                     ;V_A8 = V_B0 + V_50

[Anakin] VmPUSHB8 cc

[Anakin] VmPUSH V_A8

[Anakin] VmREADB                        ;b = BYTE:[V_A8], 判断程序入口点地址第一个字节是不是‘0xCC’

[Anakin] VmSBP                          ; 判断逻辑参考 Chunk1 及 Nor Gate

[Anakin] VmREADB

[Anakin] VmNOTANDB

[Anakin] VmPOP V_58                     

[Anakin] VmADDB

[Anakin] VmPOP V_58

[Anakin] VmSBP

[Anakin] VmREADB

[Anakin] VmNOTANDB

[Anakin] VmPOP V_B8

[Anakin] VmPOPW8 V_70

[Anakin] VmPUSH V_58

[Anakin] VmSBP

[Anakin] VmREADQ

[Anakin] VmNOTAND

[Anakin] VmPOP V_88

[Anakin] VmPUSH FFFFFFFFFFFFF7EA

[Anakin] VmNOTAND

[Anakin] VmPOP V_B0

[Anakin] VmPUSH V_B8

[Anakin] VmPUSH V_B8

[Anakin] VmNOTAND

[Anakin] VmPOP V_70

[Anakin] VmPUSH 0000000000000815

[Anakin] VmNOTAND

[Anakin] VmPOP V_88

[Anakin] VmADD

[Anakin] VmPOP V_88

[Anakin] VmPOP V_70

[Anakin] VmPOP V_88

[Anakin] VmPUSH 000000000CABFDC1        ;PUSH Branch2

[Anakin] VmSBP

[Anakin] VmPUSHB8 03

[Anakin] VmPUSHD 000000BF

[Anakin] VmPUSH V_70

[Anakin] VmNOTAND

[Anakin] VmPOP V_A0

[Anakin] VmSHR

[Anakin] VmPOP V_B0

[Anakin] VmADD

[Anakin] VmPOP V_B0

[Anakin] VmREADQ

[Anakin] VmPOP V_58

[Anakin] VmPOP V_B0

[Anakin] VmPOP V_08

[Anakin] VmPUSH V_58

[Anakin] VmPOPD V_58

[Anakin] VmPUSHD V_58

[Anakin] VmSBP

[Anakin] VmREADD

[Anakin] VmNOTANDD

[Anakin] VmPOP V_A0

[Anakin] VmPUSHD B34CBE36

[Anakin] VmNOTANDD

[Anakin] VmPOP V_78

[Anakin] VmPUSHD 4CB341C9       

[Anakin] VmPUSHD V_58

[Anakin] VmNOTANDD

[Anakin] VmPOP V_B0

[Anakin] VmNOTANDD

[Anakin] VmPOP V_B0

[Anakin] VmPOP V_08                     ;V_08 = $Branch ^ 4CB341C9

[Anakin] VmPUSH V_50

[Anakin] VmPUSH V_08

[Anakin] VmPUSH V_80

[Anakin] VmPUSH V_B0

[Anakin] VmPUSH V_40

[Anakin] VmPUSH V_88

[Anakin] VmPUSH V_38

[Anakin] VmPUSH V_98

[Anakin] VmPUSH V_18

[Anakin] VmPUSH V_68

[Anakin] VmPUSH V_28

[Anakin] VmPUSH V_30

[Anakin] VmPUSH V_48

[Anakin] VmPUSH V_60

[Anakin] VmPUSH V_10

[Anakin] VmPUSH V_A8

[Anakin] VmPUSH V_20

[Anakin] VmPUSH V_50

[Anakin] VmPUSH V_70

[Anakin] VmPUSH V_00

[Anakin] VmPUSH 0000000060A5A3CE

[Anakin] VmADD

[Anakin] VmPOP V_B0

[Anakin] VmPUSH V_90

[Anakin] VmPUSH V_08                     ; 压栈选择的分支

[Anakin] VmCALL

等价逻辑：

If (*(BYTE*)($ImageBase + AddressOfEntryPoint) != 0xCC)

{

// 即 Chunk3

VmCALL 4018BB57

}

Else

{

// 虽然调试器设置默认在入口地址处下 int3 断点，但是我们的脚本启动时，会把所有断点禁用，因此并没有走 Else 分支。

VmCALL 4018BC08

}

Chunk3
------

[Anakin] VmPOP V_A8

[Anakin] VmPUSH FFFFFFFF9F5A5C32

[Anakin] VmADD

[Anakin] VmPOP V_10

[Anakin] VmPOP V_10

[Anakin] VmPOP V_30

[Anakin] VmPOP V_28

[Anakin] VmPOP V_08

[Anakin] VmPOP V_B8

[Anakin] VmPOP V_60

[Anakin] VmPOP V_88

[Anakin] VmPOP V_40

[Anakin] VmPOP V_70

[Anakin] VmPOP V_A0

[Anakin] VmPOP V_B0

[Anakin] VmPOP V_18

[Anakin] VmPOP V_48

[Anakin] VmPOP V_00

[Anakin] VmPOP V_80

[Anakin] VmPOP V_90

[Anakin] VmSBP

[Anakin] VmREADD

[Anakin] VmPOPD V_38

[Anakin] VmSBP

[Anakin] VmREADD

[Anakin] VmNOTANDD

[Anakin] VmPOP V_68

[Anakin] VmPUSHD B34CBE36

[Anakin] VmNOTANDD

[Anakin] VmPOP V_50

[Anakin] VmPUSHD V_38

[Anakin] VmPUSHD 4CB341C9

[Anakin] VmNOTANDD

[Anakin] VmPOP V_98

[Anakin] VmNOTANDD

[Anakin] VmPOP V_20

[Anakin] VmPOP V_20

[Anakin] VmPOP V_68

[Anakin] VmPOP V_98

[Anakin] VmPOP V_78

[Anakin] VmPOP V_50

[Anakin] VmPOP V_58

[Anakin] VmPOP V_78

[Anakin] VmPOP V_28

[Anakin] VmPOP V_40

[Anakin] VmPOP V_80

[Anakin] VmPOP V_68

[Anakin] VmPUSH V_68

[Anakin] VmSBP

[Anakin] VmREADQ

[Anakin] VmNOTAND

[Anakin] VmPOP V_48

[Anakin] VmPUSH 00000000000008FF

[Anakin] VmNOTAND

[Anakin] VmPOP V_B8

[Anakin] VmPOPFQ

[Anakin] VmPUSH V_08

[Anakin] VmPUSH V_20

[Anakin] VmPUSH V_78

[Anakin] VmPUSH V_70

[Anakin] VmPUSH V_40

[Anakin] VmPUSH V_50

[Anakin] VmPUSH V_00

[Anakin] VmPUSH V_90

[Anakin] VmPUSH V_68

[Anakin] VmPUSH V_80

[Anakin] VmPUSH V_88

[Anakin] VmPUSH V_28

[Anakin] VmPUSH V_A0

[Anakin] VmPUSH V_18

[Anakin] VmPUSH V_B0

[Anakin] VmPUSH V_58

[Anakin] VmPUSH V_60

[Anakin] VmPUSH V_18

[Anakin] VmPUSH V_A8

[Anakin] VmRet

没有特别需要关注的信息，处理寄存器，函数执行完毕，返回调用处。

综上，Tls 的执行逻辑为：

If (*(BYTE*)(000000014018B3E7 + $HandlerBase) != 0)

{

// 未执行

VmCALL 40180B57

}

Else

{

If (*(BYTE*)($ImageBase + AddressOfEntryPoint) != 0xCC)

{

Return

}

Else

{

// 虽然调试器设置默认在入口地址处下 int3 断点，但是我们的脚本运行时，会把所有断点禁用 (line 15)，因此并没有走 Else 分支。

//PS：  这个分支会在 $HandlerBase + 000000014018B3E8 地址处写一个字节‘0x01’，然后返回。

//            此处暂略，后文分析反调试时再谈。

VmCALL 4018BC08

}

}

Nor Gate
========

基本单元：或非门（Nor）
-------------

两个输入位皆为 0 时输出 1，其它情况输出 0.

<table valign="top"><tbody><tr><td width="0"><p>输入 1</p></td><td width="0"><p>输入 2</p></td><td width="0"><p>Result</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td></tr></tbody></table>PS: VMP 实现的 NOTAND 操作使用了 Not 和 And 操作，有些文档称之为'与非门'，但是从逻辑语义上来说，其实现的是'或非'操作（见上表），此处遵从语义将其称之为或非门（Nor）。

取反（~）
-----

[Anakin] VmREADB                    ;b = BYTE:[000000014018B3E7 + $HandlerBase]

[Anakin] VmSBP

[Anakin] VmREADB

[Anakin] VmNOTANDB                  ;b = NOTAND(b, b)

取反计算~v 实现如下：

Result = Nor(v, v)

<table valign="top"><tbody><tr><td width="0"><p>输入 1</p></td><td width="0"><p>输入 2</p></td><td width="0"><p>Result</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td></tr></tbody></table>

与（&）
----

[Anakin] VmPUSH V_10

[Anakin] VmPUSH V_10

[Anakin] VmNOTAND                   ;d1 = NOTAND(V_10, V_10)            => d1 = ~V_10

[Anakin] VmPOP V_60

[Anakin] VmPUSH FFFFFFFFFFFFF7EA

[Anakin] VmNOTAND                   ;d1 = NOTAND(d1, FFFFF7EA)          => d1 = Nor(~V_10, ~00000815) = V_10 & 00000815

与计算 v1&v2 实现如下：

D1 = ~v1

D2 = ~v2

Result = Nor(D1, D2)

<table valign="top"><tbody><tr><td width="0"><p>输入 1</p></td><td width="0"><p>输入 2</p></td><td width="0"><p>D1</p></td><td width="0"><p>D2</p></td><td width="0"><p>Result</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td></tr></tbody></table>

异或（^）
-----

[Anakin] VmPUSHD V_A8               ;V_A8 = CQD(V_A8)

[Anakin] VmSBP

[Anakin] VmREADD

[Anakin] VmNOTANDD                  ;d1 = NOTAND(V_A8, V_A8)            => d1 = ~V_A8

[Anakin] VmPOP V_08

[Anakin] VmPUSHD DB91AA8C

[Anakin] VmNOTANDD                  ;d1 = NOTAND(d1, DB91AA8C)          => d1 = Nor(~V_A8, ~246E5573)

[Anakin] VmPOP V_68

[Anakin] VmPUSHD 246E5573

[Anakin] VmPUSHD V_A8

[Anakin] VmNOTANDD                  ;d2 = NOTAND(V_A8, 246E5573)        => d2 = Nor(V_A8, 246E5573)

[Anakin] VmPOP V_60

[Anakin] VmNOTANDD

[Anakin] VmPOP V_60

[Anakin] VmPOP V_08                 ;V_08 = NOTAND(d2, d1)              => V_08 = Nor(d1, d2) = V_A8 ^ 246E5573

异或计算 v1^v2 实现如下：

D1 = Nor(~v1, ~v2) = v1 & v2

D2 = Nor(v1, v2)

Result = Nor(D1, D2)

<table valign="top"><tbody><tr><td width="0"><p>输入 1</p></td><td width="0"><p>输入 2</p></td><td width="0"><p>D1</p></td><td width="0"><p>D2</p></td><td width="0"><p>Result</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td></tr><tr><td width="0"><p>0</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td><td width="0"><p>1</p></td></tr><tr><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>1</p></td><td width="0"><p>0</p></td><td width="0"><p>0</p></td></tr></tbody></table>

减法（-）
-----

[Anakin] VmREADB                    ;b = BYTE:[000000014018B3E7 + $HandlerBase]

[Anakin] VmSBP

[Anakin] VmREADB

[Anakin] VmNOTANDB                  ;b = NOTAND(b, b) = ~b

[Anakin] VmPOP V_60         

[Anakin] VmADDB                     ;b = 00 + b

[Anakin] VmPOP V_10                 ;V_10 = eflags

[Anakin] VmSBP

[Anakin] VmREADB

[Anakin] VmNOTANDB                  ;b = NOTAND(b, b) = ~b

[Anakin] VmPOP V_80                 ;V_80 = eflags

[Anakin] VmPOPW8 V_60               ;V_60 = b = BYTE:[000000014018B3E7 + $HandlerBase] - 0 

反码实现减法运算 v1-v2 如下：

D1 = ~v1

D2 = D1 + v2

Result  = ~D2， 即 Result  = ~(~v1 + v2)

此处不做推导，看几个实例：

<table valign="top"><tbody><tr><td><br></td><td><p>v1</p></td><td><p>v2</p></td><td><p>D1 &nbsp; = ~v1</p></td><td><p>D2 = D1 + v2</p></td><td><p>Result =~D2 &nbsp;</p></td></tr><tr><td><p>正正</p></td><td><p>10</p></td><td><p>8</p></td><td><p>-11</p></td><td><p>-3</p></td><td><p>2</p></td></tr><tr><td><p>正负</p></td><td><p>10</p></td><td><p>-8</p></td><td><p>-11</p></td><td><p>-19</p></td><td><p>18</p></td></tr><tr><td><p>负正</p></td><td><p>-10</p></td><td><p>8</p></td><td><p>9</p></td><td><p>17</p></td><td><p>-18</p></td></tr><tr><td><p>负负</p></td><td><p>-10</p></td><td><p>-8</p></td><td><p>9</p></td><td><p>1</p></td><td><p>-2</p></td></tr></tbody></table>

 再看下对 eflags 的处理：

[Anakin] VmPUSH V_10

[Anakin] VmPUSH V_10

[Anakin] VmNOTAND                   ;d1 = NOTAND(V_10, V_10)            => d1 = ~V_10

[Anakin] VmPOP V_60

[Anakin] VmPUSH FFFFFFFFFFFFF7EA

[Anakin] VmNOTAND                   ;d1 = NOTAND(d1, FFFFF7EA)          => d1 = Nor(~V_10, ~00000815) = V_10 & 00000815

[Anakin] VmPOP V_08

[Anakin] VmPUSH V_80                

[Anakin] VmPUSH V_80

[Anakin] VmNOTAND                   ;d2 = NOTAND(V_80, V_80)            => d2 = ~V_80

[Anakin] VmPOP V_60

[Anakin] VmPUSH 0000000000000815

[Anakin] VmNOTAND                   ;d2 = NOTAND(d2, 00000815)          => d2 = Nor(~V_80, ~FFFFF7EA) = V_80 & FFFFF7EA

[Anakin] VmPOP V_08

[Anakin] VmADD

[Anakin] VmPOP V_08

[Anakin] VmPOP V_70                 ;V_70 = d1 + d2

其中 FFFFF7EA = ~00000815， 00000815 = 0b100000010101。

eflags 定义如下：

![](https://bbs.pediy.com/upload/attach/202107/906381_QF5KVURTK2GEBMD.jpg)

V_10 和 V_80 皆为 eflags, 可以看到 v_70 由 V_10 的 CF, PF, AF 及 OF 位 +(or) V_80 的其它位 (ZF, SF 等) 得到。

V_10 由 VmADDB 置位，最后指令为 Add, 受影响标志位为 OF, SF, ZF, AF, CF, PF；

V_80 由 VmNOTANDB 置位，最后指令为 And, 受影响标志位为 OF(0), CF(0), SF, ZF, PF。

简单考虑最常用到的 SF 和 ZF，可以看到这两个标志位是**可以正确反映运算结果**的。

不等（!=）
------

[Anakin] VmPUSHB8 03

[Anakin] VmPUSHD 000000BF

[Anakin] VmPUSH V_70

[Anakin] VmNOTAND                   ;q = CDQ(NOTAND(V_70, 000000BF))

[Anakin] VmPOP V_68

[Anakin] VmSHR                      ;q = SHR(q, 3) = ZF == 0 ? 8 : 0

[Anakin] VmPOP V_08

[Anakin] VmADD                      ;q += SavedRBP

[Anakin] VmPOP V_08

[Anakin] VmREADQ

[Anakin] VmPOP V_A8                 ;V_A8 = QWORD:[q]

不等判断需结合上文的'减法'分析，代码中 V_70 为 eflag(v1 - v2);

像 And 操作取'1'位一样，Nor 操作可以取'0'位，上述代码 Nor(V_70, 000000BF)，其中 000000BF = 0b10111111。可以看到当 ZF 标志位为 0 时（!=, 即两数相减结果不为 0 时），返回 0b1000000，否则返回 0。

结合之后的 SHR 及取栈数据代码, 可以进一步猜想 SHR 3 是经过优化的代码，如下：

<table valign="top"><tbody><tr><td><p>优化前</p></td><td><p>Bool b = Nor(Eflags(v1 - v2),&nbsp;000000BF) &gt;&gt; 6;</p><p>Qword offset = b &lt;&lt; 3;</p></td></tr><tr><td><p>优化后</p></td><td><p>Qword offset = Nor(Eflags(v1 - v2),&nbsp;000000BF) &gt;&gt; 3;</p></td></tr></tbody></table>计算 v1 != v2 得实现如下：

(Nor(Eflags(v1 - v2), 000000BF) >> 6)  == 1。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)