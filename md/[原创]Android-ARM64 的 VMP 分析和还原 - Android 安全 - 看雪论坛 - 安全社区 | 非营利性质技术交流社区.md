> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288731.htm)

> [原创]Android-ARM64 的 VMP 分析和还原

一、背景
====

  [[原创]OLLVM 扁平化还原—新角度：状态机](https://bbs.kanxue.com/thread-288598.htm)里面分享了代码保护之一的控制流扁平化的还原思路。控制流扁平化，实际上就是对基本块之间的调用关系进行调整，但基本块的内部保持不动。而更严格的代码保护，是对基本块内部的指令进行保护，而这篇文章分享的是指令级别的代码保护（VMP）对应的分析和还原。

  样本来自金罡大佬的 [https://bbs.kanxue.com/thread-282300.htm](https://bbs.kanxue.com/thread-282300.htm)，里面是对某个大厂 so 的详细分析。金罡大佬是基于汇编直接进行分析，给出了 3 个虚拟指令的分析过程，并且贴了最终的结果。这篇文章更多是聊补阙漏，结合操作系统的知识，试着推测出 VMP 逆向分析的思路和 VMP 正向设计的理念。

  以前分析 PC 的 VMProtect，是用 Ollydbg 跟踪 trace 出汇编指令，然后再写脚本去解析导出的文本内容。这个样本的 so 比较简单，可以在 IDA 上直接 F5 转成伪代码。所以除了基于汇编分析，还结合了 IDA 的伪代码和 Microcode，让分析和还原的过程更加便捷和易懂。  

二、真实 -> 虚拟
==========

  逆向常说的 VMP，一般是指 PC 上的 VMProtect 工具；但更广泛的意思是 Virtual Machine Protection（虚拟机保护）；而更准确的说法应该是 Virtualization-based Protection（基于虚拟化的代码保护）。本质上，VMP 就是把真实代码变成虚拟代码 + 虚拟机器，从而实现代码保护的目的。

```
真实代码 <=> 虚拟代码(VitrualCodes) + 虚拟机器(VitrualMachine)
 
真实代码：call real_function(args)
虚拟保护：call VitrualMachine(VitrualCodes, args)

```

  在样本里面对应：

```
// vPC是虚拟的指令指针，指向虚拟代码(vitrual_codes)
unsigned int *__fastcall VitrualMachine(unsigned int *vPC, uint64_t args, uint64_t a3, uint64_t funcs, __int64 a5)

```

三、真实指令 VS 虚拟指令
==============

  以加法指令做例子，格式如下：

```
ADD   Xd,  Xn, #imm

```

1、真实指令
------

  真实指令的格式：

```
真实指令的硬编码：0x91001120
│31│30-23│22│21-10│9-5│4-0│
│ 1│ 0x22│ 0│0x004│  9│  0│
│sf│  ADD│ 0│  imm│ Xn│ Xd│
 
真实指令的汇编：ADD X0, X9, #4 ; X0 = X9 + 4

```

  可以看出：真实指令的硬编码是把 32bit 切割成多个字段，每个字段代表不同的含义  
  这里只是简单做演示，具体细节可以看 ARM64 的指令编码规则：[https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions?lang=en](elink@7c2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6V1k6i4k6W2L8r3!0H3k6i4u0Q4x3X3g2S2M7X3#2Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8$3y4#2L8h3g2F1N6r3q4@1K9h3!0F1i4K6u0r3k6r3c8A6x3o6j5H3x3W2)9J5c8U0t1H3x3U0g2Q4x3X3b7H3z5g2)9J5c8V1u0S2M7$3g2Q4x3X3c8u0L8Y4y4@1M7Y4g2U0N6r3W2G2L8Y4y4Q4x3@1k6D9j5h3&6Y4i4K6y4p5k6h3^5`.)  

2、虚拟指令
------

  虚拟指令的格式：  
  相对应的：虚拟指令的硬编码也可以对 32bit 重新切割成多个字段，并且赋予不同的含义

```
虚拟指令的硬编码：0x01200115
│31│30-26│25-21│20-16│15-12│11-6│ 5-0│
│ 0│    0│    9│    0│    0│   4│  21│
│ 0│    0│   Xn│   Xd│    0│ imm│vADD│
 
虚拟指令的汇编：vADD vX0, vX9, #4 ; vX0 = vX9 + 4

```

  如果想知道真实指令是怎样通过写代码转化成虚拟指令，可以参考开源的 xVMP：[https://github.com/GANGE666/xVMP](elink@347K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6s2b7f1&6s2c8e0j5$3y4W2)9J5c8Y4S2h3e0g2l9`.)  

四、真实机器 VS 虚拟机器
==============

1、真实机器
------

  ARM64 遵循 RISC（精简指令集）原则：

```
1、指令简单、长度固定（4 字节）
2、每条指令完成一个基本操作
3、尽量通过寄存器进行运算，避免复杂寻址模式

```

  换句话说，真实机器需要提供寄存器，方便真实指令进行计算处理。

```
通用寄存器：X0、X1...X30
复用寄存器：XZR/PC(X31)
程序计数器：PC

```

  此外，真实指令在真实机器上的运行流程如下：

```
1、Fetch（取指）：用 IP/PC 从内存读取指令
2、Decode（译码）：解码的到：操作码与操作数
3、Execute（执行）：执行指令逻辑，如加减、跳转
4、PCUpdate（更新）：指令指针更新，递增或者跳转

```

2、虚拟机器
------

  对应的，虚拟机器也需要提供虚拟寄存器：

```
通用寄存器：X0、X1...X30 -> vX0、vX1...vX30
复用寄存器：XZR/SP(X31) -> vXZR/vSP (vX31)
程序计数器：PC -> vPC

```

  **虚拟代码在虚拟机器上的运行过程**和**真实代码在真实机器上的运行过程**也类似，4 个步骤在虚拟机器里面对应如下：

```
1、Fetch（取指）
0x2CC8: LDR W12, [X0]             ; vInsn = *vPC;
 
2、Decode（译码）
0x2CD4: AND   W10, W12, #0x3F     ; vOpcode
0x2CEC: AND W16, W12, #0x10000000 ; v9 = vInsn & 0x10000000;
0x2D08: AND W15, W12, #0x20000000 ; v8 = vInsn & 0x20000000;
0x2D14: AND W14, W12, #0x40000000 ; v7 = vInsn & 0x40000000;
0x2D2C: AND W13, W12, #0x80000000 ; v6 = vInsn & 0x80000000;
 
3、Execute（执行）
0x2CE8: LDRSW X3, [X27,X10,LSL#2]
0x2D20: ADD X2, X3, X27
0x2D3C: BR X2                     ; switch(vOpcode)
 
4、PCUpdate（更新）
0x3A80: ADD X0, X9, #4            ; vPC = (*p_vPC + 4);
0x3A88: STR X0, [X21]             ; *p_vPC = vPC;

```

五、虚拟机器的运行
=========

  以上是偏理论的讲解，对 VMP 有一个简单的了解之后，接下来就是看看虚拟机器对虚拟代码的具体运行过程：

0、call（调用）
----------

  先从调用虚拟机器开始说起

```
call VitrualMachine(VitrualCodes, args)
 
unsigned int *__fastcall sub_2AC4(__int64 a1, __int64 a2, __int64 a3, __int64 a4)
    return vitrual_machine(dword_B090, args, 0, &off_21D10, v6);

```

（1）虚拟机器 (VitrualMachine): 0x2C18  
  vPC 指向虚拟代码的指针，就是虚拟的指令寄存器（也就是 vIP）

```
unsigned int *__fastcall VitrualMachine(unsigned int *vPC, uint64_t args, ...)

```

（2）虚拟代码 (VitrualCodes): dword_B090  
  这里的虚拟代码是 32bit 的虚拟硬编码

```
0xB090 DCD 0xDFBDFC15, 0x23BF0217, 0x23BE0017, 0x1FB70E17, 0x1FB60C17
0xB0A4 DCD 0x1FB50A17, 0x1FB40817, 0x1FB30617, 0x1FB20417, 0x1FB10217
0xB0B8 DCD 0x1FB00017, 0xE081CB, 0x8B0428, 0x8C0028, 0xC20028
0xB0CC DCD 0xC30228, 0xC50428, 0xC70628, 0xC80828, 0xC90A28, 0x920628
0xB0E4 DCD 0xCA0C28, 0xC60E28, 0x930228, 0x6610D95, 0xBA10217
0xB0F8 DCD 0x2440028, 0x808109CB, 0x2000D8, 0x64B, 0x8000A8C
0xB10C DCD 0x2400017, 0xF001F8B4, 0x84215C07, 0xC1230B, 0x7A40017
0xB120 DCD 0x141230B, 0x3A40E17, 0x121230B, 0x3A40C17, 0x101230B
0xB134 DCD 0x3A40817, 0xE1230B, 0x3A40617, 0xA1230B, 0x3A40017
0xB148 DCD 0x8061B30B, 0x41F30B, 0x140815, 0x1BB40C17, 0x1BA50C15
0xB15C DCD 0x3C021CB, 0x8200C1CB, 0x3AB0A17, 0x8320FACB, 0x3AC0217
0xB170 DCD 0x1BB40A17, 0x1BB50E28, 0x1BB50817, 0x1BA50815, 0x8200C1CB
0xB184 DCD 0x8320FACB, 0x2E021CB, 0x1BA50415, 0x160415, 0x1BB60417
0xB198 DCD 0x8200C1CB, 0x8320FACB, 0x3C021CB, 0x1BA50015, 0x1BB60017
0xB1AC DCD 0x1BB10628, 0x3B10417, 0x8200C1CB, 0x8320FACB, 0x3C021CB
0xB1C0 DCD 0x17B50417, 0x17B40617, 0x17B10817, 0x17B60A17, 0x17B60E17
0xB1D4 DCD 0x1BB40228, 0x17B40C17, 0x3A40028, 0x8200C1CB, 0x8320FACB
0xB1E8 DCD 0x17A50415, 0x6710015, 0x17A50015, 0x17B10017, 0x8200C1CB
0xB1FC DCD 0x8320FACB, 0x3C021CB, 0x7BE0215, 0x17B60228, 0x13B30C17
0xB210 DCD 0x3B70228, 0x13B70A17, 0x13BE0E17, 0x3A40628, 0x8200C1CB
0xB224 DCD 0x8320FACB, 0x13A50A15, 0x13A50415, 0x4010015, 0x13BE0617
0xB238 DCD 0x13B60417, 0x13A10817, 0x3BE0828, 0x8200C1CB, 0x8320FACB
0xB24C DCD 0x3C021CB, 0xFA50E15, 0x6C10015, 0x13B70017, 0xFA10E17
0xB260 DCD 0x13B30217, 0x8200C1CB, 0x8320FACB, 0x3C021CB, 0x10811
0xB274 DCD 0xFB50A17, 0x3B30A28, 0xFB30817, 0xFA10C30, 0x3A40C28
0xB288 DCD 0x8200C1CB, 0x8320FACB, 0xFA50815, 0x2610995, 0x3B30428
0xB29C DCD 0x20411, 0xBA30215, 0xBB30A17, 0xBA20C30, 0xBB40E17
0xB2B0 DCD 0xFB60017, 0xFB10217, 0xFA10417, 0xFA30617, 0xBA10228
0xB2C4 DCD 0xFC21F695, 0xBA10217, 0x3A40E28, 0x8200C1CB, 0x8320FACB
0xB2D8 DCD 0xBA50A15, 0xBA20228, 0x400118, 0x64B, 0x410995, 0x800068C
0xB2F0 DCD 0x2410017, 0x2400017, 0xBB50817, 0xBA50815, 0x7B10028
0xB304 DCD 0x8200C1CB, 0x8320FACB, 0x22021CB, 0xBB30617, 0xBA50615
0xB318 DCD 0x8200C1CB, 0x8320FACB, 0x22021CB, 0xBA50415, 0xBB60417
0xB32C DCD 0x8200C1CB, 0x8320FACB, 0x22021CB, 0x1FB00028, 0x1FB10228
0xB340 DCD 0x1FB20428, 0x1FB30628, 0x1FB40828, 0x1FB50A28, 0x1FB60C28
0xB354 DCD 0x1FB70E28, 0x23BE0028, 0x23BF0228, 0x3E00F8B, 0x23BD0415

```

1、Fetch（取指）
-----------

  X0 是第一个参数 vPC  
  取指就是从 vPC 取出虚拟指令：vInsn

```
0x2CC8: LDR W12, [X0] ; vInsn = *(vPC)

```

2、Decode（译码）
------------

  取出 vInsn 之后，就开始解析翻译  
  具体来说就是对 32 位的数据进行切分  
  译码对应的汇编：

```
0x2CD4 AND   W10, W12, #0x3F        ; W10[5:0] = W12[5:0]
 
0x2CE0 LSR   W11, W12, #0xB         ; W11[20:0] = W12[31:11]
0x2CE4 AND   W14, W11, #2           ; W14[1] = W11[1] = W12[12]
0x2CEC AND   W16, W12, #0x10000000  ; W16[28] = W12[28]
0x2CF0 AND   W2, W11, #4            ; W2[2] = W12[13]
0x2CF4 BFXIL W14, W12, #0x1F, #1    ; W14[0] = W12[31]
0x2CF8 ORR   W14, W14, W2           ; W14[2][1][0] = W12[13][12][31]
0x2CFC AND   W2, W11, #8            ; W2[3] = W12[14]
0x2D00 AND   W10, W11, #0x10        ; W10[4] = W12[15]
0x2D04 LSR   W11, W16, #0x1A        ; W11[2] = W12[28]
0x2D08 AND   W15, W12, #0x20000000  ; W15[29] = W12[29]
0x2D0C ORR   W2, W14, W2            ; W2[3][2][1][0] = W12[14][13][12][31]
0x2D10 BFXIL W11, W12, #0x1A, #2    ; W11[2:0] = W12[28:26]
0x2D14 AND   W14, W12, #0x40000000  ; W14[30] = W12[30]
0x2D18 ORR   W10, W2, W10           ; W10[4][3][2][1][0] = W12[15][14][13][12][31]
0x2D1C ORR   W11, W11, W15,LSR#26   ; W11[3:0] = W12[29:26]
0x2D24 UBFX  W9, W12, #0x15, #5     ; W9[4:0]  = W12[25:21]
0x2D28 UBFX  W8, W12, #0x10, #5     ; W8[4:0]  = W12[20:16] 
0x2D2C AND   W13, W12, #0x80000000  ; W13[31] = W12[31] 
0x2D30 AND   W18, W12, #0x4000000   ; W18[26] = W12[26]
0x2D34 AND   W17, W12, #0x8000000   ; W17[27] = W12[27] 
0x2D38 ORR   W11, W11, W14,LSR#26   ; W11[4:0] = W12[30:26]

```

  寄存器映射关系：

```
W10[5:0] = W12[5:0]
 
W8[4:0]  = W12[20:16]
W9[4:0]  = W12[25:21]
W10[4:0] = W12[15][14][13][12][31]
W11[4:0] = W12[30:26]
 
W13[31] = W12[31] 
W14[30] = W12[30]
W15[29] = W12[29]
W16[28] = W12[28]
W17[27] = W12[27]
W18[26] = W12[26]

```

  切分的示意图如下：

```
┌───┬─────┬─────┬─────┬─────┬────┬───┐
│31 │30-26│25-21│20-16│15-12│11-6│5-0│
└───┴─────┴─────┴─────┴─────┴────┴───┘
  │    │     │     │     │     │   └─► 6位 操作码
  │    │     │     │     │     └─────► 6位 操作数
  │    │     │     │     └───────────► 4位 操作数 Xm w10 加[31]凑5位
  │    │     │     └─────────────────► 5位 操作数 Xt w8
  │    │     └───────────────────────► 5位 操作数 Xn w9
  │    └─────────────────────────────► 5位 操作数 Xa w11
  └──────────────────────────────────► 1位 [15:12][31]凑5位操作数

```

  根据这个切分规则，可以写出对应的切分脚本：

```
def decode_vinsn(vinsn: int):
    op31       = (vinsn >> 31) & 0x1      # bit 31
    op30_26    = (vinsn >> 26) & 0x1F     # 30-26 (5 bits)
    op25_21    = (vinsn >> 21) & 0x1F     # 25-21 (5 bits)
    op20_16    = (vinsn >> 16) & 0x1F     # 20-16 (5 bits)
    op15_12    = (vinsn >> 12) & 0xF      # 15-12 (4 bits)
    op11_6     = (vinsn >> 6)  & 0x3F     # 11-6  (6 bits)
    op5_0      = vinsn & 0x3F             # 5-0   (6 bits)
 
    print(f"│31│30-26│25-21│20-16│15-12│11-6│5-0│")
    print(f"│{op31:2b}│{op30_26:5d}│{op25_21:5d}│{op20_16:5d}│{op15_12:5d}│{op11_6:4d}│{op5_0:3d}│")
 
# 调用：
# 第1条虚拟指令：0xDFBDFC15
decode_vinsn(0xDFBDFC15)
 
输出：
│31│30-26│25-21│20-16│15-12│11-6│5-0│
│ 1│   23│   29│   29│   15│  48│ 21│

```

3、Execute（执行）
-------------

  指令切分得到：操作码和操作数，执行就是根据操作码的含义，对操作数进行处理。换句话说，执行需要对操作码进行分发处理：

```
0x2C80: ADRP X27, #jpt_2D3C@PAGE       ; X27 = jpt_2D3C 跳转表页基址
0x2CA4: ADD X27, X27, #jpt_2D3C@PAGEOFF; X27 = jpt_2D3C 跳转表绝对地址
0x2CD4: AND W10, W12, #0x3F            ; vOpcode = vInsn[5:0] 虚拟操作码
0x2CE8: LDRSW X3, [X27,X10,LSL#2]      ；X3 = *((X27 + (X10<<2))) 相对偏移
0x2D20: ADD X2, X3, X27                ；X2 = X27 + X3 跳转地址
0x2D3C: BR X2                          ; switch(vOpcode)

```

  基于汇编，可以写出跳转表的脚本：

```
def dump_jumptable(symname, count):
    base_ea = ida_name.get_name_ea(idaapi.BADADDR, symname)
    if base_ea == idaapi.BADADDR:
        return
 
    print(f"\n[jumptable]\n{symname}: 0x{base_ea:X}")
    for idx in range(count):
        ea = base_ea + idx * 4
        off = ida_bytes.get_dword(ea)
        if off & 0x80000000:
            off -= 0x100000000
         
        target = base_ea + off
        print(f"case 0x{idx:X} : 0x{target:X}")
 
# 调用
dump_jumptable("jpt_2D3C", 64)
 
# 输出
[jumptable]
jpt_2D3C: 0xA6A0
case 0x0 : 0x2DC0
case 0x1 : 0x3A78
case 0x2 : 0x305C
case 0x3 : 0x3A78
case 0x4 : 0x3A78
case 0x5 : 0x3A78
case 0x6 : 0x3A78
case 0x7 : 0x2ECC
case 0x8 : 0x3A78
case 0x9 : 0x3A78
case 0xA : 0x2DC0
case 0xB : 0x3098
case 0xC : 0x2F80
case 0xD : 0x3A78
case 0xE : 0x30E8
case 0xF : 0x2D40
case 0x10 : 0x3A78
case 0x11 : 0x2F24
case 0x12 : 0x3A78
case 0x13 : 0x2DC0
case 0x14 : 0x2E50
case 0x15 : 0x2F24
case 0x16 : 0x3A78
case 0x17 : 0x2E50
case 0x18 : 0x2DC0
case 0x19 : 0x312C
case 0x1A : 0x2D40
case 0x1B : 0x2E50
case 0x1C : 0x3A78
case 0x1D : 0x2DC0
case 0x1E : 0x2D40
case 0x1F : 0x3A78
case 0x20 : 0x2D40
case 0x21 : 0x2D40
case 0x22 : 0x3A78
case 0x23 : 0x2E50
case 0x24 : 0x2E50
case 0x25 : 0x2F24
case 0x26 : 0x2D40
case 0x27 : 0x3A78
case 0x28 : 0x2D40
case 0x29 : 0x3220
case 0x2A : 0x2D40
case 0x2B : 0x2D40
case 0x2C : 0x3A78
case 0x2D : 0x2D40
case 0x2E : 0x2D40
case 0x2F : 0x3A78
case 0x30 : 0x2E50
case 0x31 : 0x2D40
case 0x32 : 0x2F24
case 0x33 : 0x2ECC
case 0x34 : 0x2ECC
case 0x35 : 0x2D40
case 0x36 : 0x2E50
case 0x37 : 0x2F80
case 0x38 : 0x3254
case 0x39 : 0x2DC0
case 0x3A : 0x2E50
case 0x3B : 0x2DC0
case 0x3C : 0x3A78
case 0x3D : 0x2DC0
case 0x3E : 0x3A78
case 0x3F : 0x2ECC

```

  因为编译器优化会把类似的操作聚合在一起，导致多个 case 可能对应同一个地址：

```
case 0x0 : 0x2DC0
case 0xA : 0x2DC0
case 0x13 : 0x2DC0
case 0x18 : 0x2DC0
case 0x1D : 0x2DC0
case 0x39 : 0x2DC0
case 0x3B : 0x2DC0
case 0x3D : 0x2DC0

```

  在 ida 的伪代码里面就很明显：  
![](https://bbs.kanxue.com/upload/attach/202510/1000535_VZZFT5A5ECV845X.webp)

  虽然基于汇编的脚本可以继续优化得到类似的输出，但更舒服的方式是基于 Microcode，因为 Microcode 已经对跳转表进行解析和统计：

```
jtbl   (v18.1{8} & #0x3F.1), {0,0xA,0x13,0x18,0x1D,0x39,0x3B,0x3D => 9, 2 => 39, 7,0x33,0x34,0x3F => 17, 0xB => 40, 0xC,0x37 => 25, 0xE => 47, 0xF,0x1A,0x1E,0x20,0x21,0x26,0x28,0x2A,0x2B,0x2D,0x2E,0x31,0x35 => 7, 0x11,0x15,0x25,0x32 => 21, 0x14,0x17,0x1B,0x23,0x24,0x30,0x36,0x3A => 15, 0x19 => 52, 0x29 => 62, 0x38 => 68, def => 432} ; 2D3C u=w12.1

```

  基于 Microcode 的脚本：

```
# m_jtbl l, r=mcases
if minsn.opcode == ida_hexrays.m_jtbl :
    # print(f'm_jtbl = 0x{minsn.ea:X}')
    # print(get_mopt_name(minsn.l.t))
    # print(get_mcode_name(minsn.l.d.opcode))
    # print(get_mopt_name(minsn.r.t))
     
    mcases = minsn.r.c
    casesvec = mcases.values
    targetvec = mcases.targets
     
    print(f'{i:3d}: 0x{minsn.ea:X} [switch]')
    for j in range(casesvec.size()):
        cases = casesvec.at(j)
        if cases.size() == 0 : continue
        cases_str = ", ".join(f"{cases.at(k)}" for k in range(cases.size()))
        # cases_str = ", ".join(f"0x{cases.at(k):X}" for k in range(cases.size()))
        target = targetvec.at(j)
        target_mblock = mba.get_mblock(target)
        print(f'{target:3d}: 0x{target_mblock.start:X} <- case : {cases_str}')
 
# 输出
  6: 0x2D3C [switch]
  9: 0x2DC0 <- case : 0, 10, 19, 24, 29, 57, 59, 61
 39: 0x305C <- case : 2
 17: 0x2ECC <- case : 7, 51, 52, 63
 40: 0x3098 <- case : 11
 25: 0x2F80 <- case : 12, 55
 47: 0x30E8 <- case : 14
  7: 0x2D40 <- case : 15, 26, 30, 32, 33, 38, 40, 42, 43, 45, 46, 49, 53
 21: 0x2F24 <- case : 17, 21, 37, 50
 15: 0x2E50 <- case : 20, 23, 27, 35, 36, 48, 54, 58
 52: 0x312C <- case : 25
 62: 0x3220 <- case : 41
 68: 0x3254 <- case : 56

```

  也正因为编译器优化聚合，导致有差异的操作在聚合之后，需要再次进行分发：  
![](https://bbs.kanxue.com/upload/attach/202510/1000535_YBKF8PFX2R8VR5X.webp)

我们是要分析同一条虚拟指令完整的执行过程，对应有 3 个思路：  
  1、如果基于 ida 的伪代码，直接肉眼去跟进，但**函数级别**比较大容易眼花缭乱；  
  2、也可以 trace 得到真实执行的汇编指令列表，然后挨条指令去跟进，但**汇编级别**的工作量又会比较多；  
  3、除此之外，还可以按照**基本块级别**进行分析，工作量就适中了。具体来说就是做一个 Microcode 模拟器，模拟 Microcode 的执行过程，从而得到基本块的跳转关系。具体可以参考 D-810 的 Microcode 模拟器：[https://gitlab.com/eshard/d810/-/blob/master/d810/emulator.py](elink@336K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8D9j5h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6W2M7$3S2S2M7X3c8Q4x3V1k6V1z5o6p5H3i4K6u0r3i4K6u0V1i4K6u0r3j5X3I4G2j5W2)9J5c8X3#2S2M7%4c8W2M7W2)9J5c8X3b7^5x3e0m8Q4x3V1k6W2L8i4g2D9j5i4c8G2M7W2)9J5k6i4m8&6)  

  这里拿第 1 条虚拟指令的虚拟操作码：21(0x15)

```
模拟器输出：
opcode=21 : 21-0x2F24, 22-0x2F54, 38-0x3048
 
分析结果：
1、21-0x2F24
(1)汇编：
; jumptable 0000000000002D3C cases 17,21,37,50
0x2F24 AND W0, W12, #0xF000
 
(2)伪代码：
case 0x15u:
  imm16 = vInsn & 0xF000 ...
 
2、21-0x2F24
(1)汇编：
0x2F54 CMP W11, #0x15
 
(2)伪代码：
if ( vOpcode == 0x25 || vOpcode == 0x15 )
 
3、38-0x3048
(1)汇编：
0x3048 ADD X11, X21, #8        ; X11 = X21 + 8
0x304C LDR X9, [X11,W9,UXTW#3] ; X9 = *(X11 + (W9 << 3))
0x3050 ADD X9, X9, W10,SXTH    ; X9 = X9 + X10 = X9 + imm16
0x3054 STR X9, [X11,W8,UXTW#3] ; *(X11 + (W8 << 3)) = X9
 
等价于：*(X11 + (W8 << 3)) = *(X11 + (W9 << 3)) + imm16
 
(2)伪代码：
*(v12 - 0x130 + 8LL * v29) = *(v12 - 0x130 + 8LL * v28) + v44;

```

  前面提到 ARM 是基于寄存器进行运算，这就意味者：

```
1、这是一条加上立即数的指令
2、v12 - 0x130 + 8LL * v29 对应 虚拟目标寄存器：vXd
3、v12 - 0x130 + 8LL * v28 对应 虚拟源寄存器：vXn

```

  从而可以进一步推测出：

```
1、v12 - 0x130 对应 最开始的虚拟寄存器：vX0
2、v29 = W8[4:0] = W12[20:16]就是虚拟目标寄存器的编号
3、v28 = W9[4:0] = W12[25:21]就是虚拟源寄存器的编号

```

  虚拟操作码对应的格式就是：

```
vADD  vXd, vXn, #imm

```

  代入具体的数据就可以得到第 1 条虚拟指令的含义：

```
│31│30-26│25-21│20-16│15-12│11-6│5-0│
│ 1│   23│   29│   29│   15│  48│ 21│
 
W8[4:0] = W12[20:16] = 29
W9[4:0] = W12[25:21] = 29
 
vADD vX29, vX29, -0x210

```

4、PCUpdate（更新）
--------------

  虚拟指令执行完，接下来就是更新：

```
0x3058: B def_2D3C ; 跳转到更新PC的基本块
 
0x3A78 def_2D3C
0x3A78 LDR X9, [X21]          ; vPCOld = *p_vPC
0x3A80 ADD X0, X9, #4         ; vPC = vPCOld + 4 -> vPC = *p_vPC + 4
0x3A88 STR X0, [X21]          ; *p_vPC = vPC;

```

  可以看出：vPC 通过加 4 进行更新，也就是按顺序执行。但也有可能是跳转执行，甚至是调用外部函数。所以接下来就是处理跳转执行和调用外部函数：

```
v91 = *(v12 - 0x20);   // v12 - 0x20: flag，决定顺序执行还是跳转执行
if ( v91 == 2 )
    if ( !vPC ) return vPC;
    goto fetch;        // 跳到Fetch（取指）
 
if ( v91 == 3 || v91 == 1 )
    vPC = *(v12 - 0x18);        // v12 - 0x18: 跳转或者调用的地址
    *(v12 - 0x20) = 0;          // flag改为0
    if ( vPC == v13 )           // v13: 跳板函数，用来调用外部函数
        (v13)(*v15, *v16);      // 调用外部函数
        vPC = *(v12 - 0x10);    // v12 - 0x10: 调用之后的返回函数
        if ( !vPC ) return vPC;
        goto fetch;             // 跳到Fetch（取指）
    else // vPC ！= v13         // 不是跳板函数
        if ( !vPC ) return vPC;
        goto fetch;             // 跳到Fetch（取指）

```

  PCUpdate（更新）之后，又继续下一轮的 Fetch（取指），有一个细节需要关注一下：

```
# 取指之后，会紧跟着处理一下状态
vInsn = *vPC;          // 1、Fetch（取指）
if ( flag == 2 ) {     // flag == 2 会被改为 flag == 3
    flag = 3;
    *(v12 - 0x20) = 3;
}

```

  这就意味着，某条指令修改 flag == 2，继续执行下一条指令后  
  在下一条指令的开头：因为 flag == 2，修改 flag = 3  
  在下一条指令的结束：因为 flag == 3，进行跳转  

  举个例子，按虚拟代码的顺序，应该是先 00000078，再 0000007C  
  但虚拟机器有了这个特殊的调度流程，实际上效果变成：先 0000007C，再 00000078

```
0000007C 02400017 : STR  X0, [X18, #0x0]
00000078 08000A8C : BR   #0x2A8

```

六、虚拟机器的状态
=========

  在前面提到虚拟机器需要提供虚拟寄存器，方便虚拟指令进行运算。从虚拟机器的运行过程，我们也能得知虚拟机器依赖一些特殊变量充当特殊角色。而虚拟机器的状态 VMState 就是包含这些虚拟寄存器和特殊的变量，为虚拟指令提供可以使用的上下文环境。

  在上面的分析过程中已经知道：

```
v12 - 0x130 : vX0
v12 - 0x20  : flag
v12 - 0x18  : 跳转或者调用的地址
v12 - 0x10  : 调用之后的返回函数

```

  从 vX0 可以推测出其它通用寄存器：

```
0x2C38 SUB  X8, X19, #0x130    ; X19 - 0x130 -> v12 - 0x130 : vX0
0x2C3C STUR XZR, [X19,#-0x38]  ; X19 - 0x38
0x2C40 STR  XZR, [X8]
 
(0x130 - 0x38) / 8 = 0x1F 刚好是31，对应vX0到vX31

```

  结合更多的虚拟指令去完善 VMState，最终得到 VMState 的结构体：

```
00000000 struct VMState // sizeof=0x150
00000000 {
00000000     uint64_t vStack;
00000008     uint64_t vStack1;
00000010     uint64_t _pad0;
00000018     uint64_t vPC;
00000020     uint64_t vX0;
00000028     uint64_t vX1;
00000030     uint64_t vX2;
00000038     uint64_t vX3;
00000040     uint64_t vX4;
00000048     uint64_t vX5;
00000050     uint64_t vX6;
00000058     uint64_t vX7;
00000060     uint64_t vX8;
00000068     uint64_t vX9;
00000070     uint64_t vX10;
00000078     uint64_t vX11;
00000080     uint64_t vX12;
00000088     uint64_t vX13;
00000090     uint64_t vX14;
00000098     uint64_t vX15;
000000A0     uint64_t vX16;
000000A8     uint64_t vX17;
000000B0     uint64_t vX18;
000000B8     uint64_t vX19;
000000C0     uint64_t vX20;
000000C8     uint64_t vX21;
000000D0     uint64_t vX22;
000000D8     uint64_t vX23;
000000E0     uint64_t vX24;
000000E8     uint64_t vX25;
000000F0     uint64_t vX26;
000000F8     uint64_t vX27;
00000100     uint64_t vX28;
00000108     uint64_t vX29;
00000110     uint64_t vX30;
00000118     uint64_t vX31;
00000120     uint64_t _pad1;
00000128     uint64_t _pad2;
00000130     uint64_t vFlag;
00000138     uint64_t vAddrJump;
00000140     uint64_t vAddrReturn;
00000148     uint64_t vAddrBase;
00000150 };

```

  这里细说一下结构体里面的成员变量

```
v12 - 0x20  : flag
flag的偏移是负数：-0x20
 
IDA的结构体不支持负数的偏移
所以要换成-1的元素得到正数的偏移
 
正常是：VMState+flagOffset
也就是：&VMState[0]+flagOffset
修改成：&VMState[-1]+sizeof(VMState)+flagOffset
具体值：sizeof(VMState)是0x150；而flagOffset是-0x20
计算出：0x150 - 0x20 = 0x130 -> 00000130 uint64_t vFlag;

```

  把 v12 改成 VMState *VMState，在 IDA 上面就能更直观看出虚拟指令的含义：

```
第1条虚拟指令的伪代码：
*(v12 - 0x130 + 8LL * v29) = *(v12 - 0x130 + 8LL * v28) + v44;
 
改成VMState变得更直观：
*(&VMState[-1].vX0 + Xd) = *(&VMState[-1].vX0 + Xn) + imm16;

```

七、总结
====

  VMP 虚拟化保护，是把真实代码变成虚拟代码加虚拟机器，然后虚拟机器按自己的规则去运行虚拟代码，实现真实代码的效果。

  至于代码保护，实际上是提高逆向分析的难度，VMP 的正向设计主要的 2 个关键点：

```
1、虚拟指令的格式含义重新定义
2、虚拟机器的运行过程重新调整

```

  对应的，VMP 的逆向分析就需要完成 2 个关键点：

```
1、梳理出虚拟机器的运行过程
2、还原成虚拟指令的格式含义

```

八、题外话
=====

  之前准备对 Android 的 so 进行加固，发现 VMProtect 不支持 Android 的 ARM64。收费方面：国内有几家商用的 so 加固，但价格比较贵。免费方面：有开源的 xVMP，但只是 demo 不够用。如果移植 VMProtect 的虚拟化策略去完善 xVMP，工作量又特别大。

  后来看到大佬对 VMP 的分析文章，一方面文章写得特别详细；另一方面样本是大厂在用的成熟方案，很适合学习之后自己魔改一个完整的 xVMP。

  结果，10 月 1 号，VMProtect 支持 Android 的 ARM64 了...

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm)