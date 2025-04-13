> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286451.htm)

> [原创] 站在巨人的肩膀上：SO 中 ARM_VMP 解释器分析

本人接触 VMP 也有很长时间了，各位大佬的东西也研究的不少，比如 xxvm、小曾、hanbingle、我是小三、krash、金罡等等，我就不一一列举了。其中论详细程度的话莫过于金罡大佬的 https://bbs.kanxue.com/thread-282300.htm，本文有一些注释、命名借鉴于金罡大佬的这篇文章。

最近闲着无聊在家准备探一探大厂的 APP 中 VMP 的防护强度，于是下载了一个最新版本的。事先声明，本文目的在于学习研究，不要用于非法用途，否则后果自负。

本人文笔不太好，再加上分析过程中可能会有些错误，希望大佬批评指正。

1、首先在 VMP 解释器入口分析出 vm_info，即 vm_reg_PC（虚拟指令指针）、VM_REG（虚拟寄存器）等关键信息是通过哪些真实寄存器去保存：

![](https://bbs.kanxue.com/upload/attach/202504/823643_B7MP28TPEGHFSSA.webp) 

vm_info：

X27--pVMMemoryEnd

X6--pVMMemoryEnd-0x140  P_vm_reg_PC(Pcontext-8)

pVMMemoryEnd-0x138--VM_REG

[SP,#0x38]--P_vm_reg_PC

X0--vm_reg_PC

2、根据 vm_reg_PC 分析出虚拟操作码、虚拟操作数：

         ![](https://bbs.kanxue.com/upload/attach/202504/823643_E3BDR476Z5F5XUY.webp)

        ![](https://bbs.kanxue.com/upload/attach/202504/823643_6R8KJ8BS45ZFB2A.webp)

并对其进行解析——

虚拟操作码：

W15--Opcode[5-0:6]

虚拟操作数：

[SP,#0x34]--Opcode>>0x1B Opcode[31-27:5]

[SP,#0x40]--Opcode>>0x1C Opcode[31-28:4]

[SP,#0x44]--Opcode>>0x19 Opcode[31-25:7]

[SP,#0x48]--Opcode>>0xF  Opcode[31-15:17]

[SP,#0x4C]--Opcode>>0x16 Opcode[31-22:10]

[SP,#0x50]--Opcode>>0x8  Opcode[31-8:24]

[SP,#0x54]--Opcode>>0x1A Opcode[31-26:6]

[SP,#0x58]--Opcode>>0xB  Opcode[31-11:21]

[SP,#0x5C]--Opcode>>0x11 Opcode[31-17:15]

[SP,#0x60]--Opcode>>0x15 Opcode[31-21:11]

[SP,#0x64]--Opcode>>0x18 Opcode[31-24:8]

[SP,#0x68]--Opcode>>0x14 Opcode[31-20:12]

[SP,#0x6C]--Opcode>>0xE  Opcode[31-14:18]

[SP,#0x70]--Opcode>>0x10 Opcode[31-16:16]

[SP,#0x74]--Opcode>>0x12 Opcode[31-18:14]

[SP,#0x78]--Opcode>>0x9  Opcode[31-9:23]

[SP,#0x7C]--Opcode>>0xD  Opcode[31-13:19]

W3--Opcode>>0x1E   Opcode[31-30:2]

W7--Opcode>>0x1D   Opcode[31-29:3]

W13--Opcode>>0xC   Opcode[31-12:20]

W19--Opcode>>0x2   Opcode[31-2:30]

W20--Opcode>>0x3   Opcode[31-3:29]

W21--Opcode>>0x4   Opcode[31-4:28]

W22--Opcode>>0x17   Opcode[31-23:9]

W23--Opcode>>0x1   Opcode[31-1:30]

W24--Opcode>>0x6   Opcode[31-6:26]

W25--Opcode>>0x5   Opcode[31-5:27]

W26--Opcode>>0xA   Opcode[31-10:22]

W28--Opcode>>0x7   Opcode[31-7:25]

W30--Opcode>>0x13   Opcode[31-19:13]

W0--Opcode<<0x7

W1--Opcode<<0x3

W4--Opcode<<0x9

W9--Opcode<<0x2

W10--Opcode<<0x1

W11--Opcode<<0x6

W14--Opcode<<0x4

W16--Opcode<<0x5

W17--Opcode<<0x8

3、根据虚拟操作码，找到对应的 vm_Handler；在此版本中对 vm_Handler 进行了地址混淆，并抛弃之前版本的跳转表结构（容易被发现），改用二叉树结构。

多次进行二叉树形式的递归判断：

![](https://bbs.kanxue.com/upload/attach/202504/823643_B6VZX8JY8AHUJNA.webp) 

多次用到地址混淆：

![](https://bbs.kanxue.com/upload/attach/202504/823643_Q4DXVSPW4JZ27EM.webp) 

通过对上述二叉树结构的分析，直到分析到每个根节点，梳理出每个 vm_Handler 对应的地址（部分地址）：

W15==0x0--0x4F37C  

W15==0x1--0x                    

W15==0x2--0x506D0

W15==0x3--0x

W15==0x4--0x4FDD8

W15==0x5--0x50EE8

W15==0x6--0x51C74

W15==0x7--0x

W15==0x8--0x4F968

W15==0x9--0x50ADC

W15==0xA--0x51800

W15==0xB--0x502AC

W15==0xC--0x

W15==0xD--0x51260

W15==0xE--0x51F48

W15==0xF--0x4F720

W15==0x10--0x508D0

W15==0x11--0x4FFEC

W15==0x12--0x

W15==0x13--0x

W15==0x14-0x52460

W15==0x15-0x510DC

W15==0x16--0x

W15==0x17--0x4FBE4

W15==0x18--0x50C9C

W15==0x19--0x51A38  

W15==0x1A--0x50454

W15==0x1B--0x

W15==0x1C--0x

W15==0x1D--0x514D8

W15==0x1E--0x5210C

W15==0x1F--0x4F570  

W15==0x20--0x507DC

W15==0x21--0x

W15==0x22--0x50FDC

W15==0x23--0x51D64

W15==0x24--0x4FABC

W15==0x25--0x

W15==0x26--0x

W15==0x27--0x50BB8

W15==0x28--0x

W15==0x29--0x

W15==0x2A--0x

W15==0x2B--0x5191C                  

W15==0x2C--0x50348                        

W15==0x2D--0x51394                        

W15==0x2E--0x5200C

W15==0x2F--0x4F858

W15==0x30--0x52368                      

W15==0x31--0x509D4

W15==0x32--0x

W15==0x33-0x5012C

W15==0x34--0x

W15==0x35--0x51200

W15==0x36--0x51E50                      

W15==0x37--0x4FCDC                        

W15==0x38--0x50E04

W15==0x39--0x

W15==0x3A--0x

W15==0x3B--0x                      

W15==0x3C--0x51B28

W15==0x3D--0x505D0                      

W15==0x3E--0x51630                        

W15==0x3F--0x52238

4、在每个 vm_Handler 的地址进行详细分析，分析出对应的虚拟指令

4.1、以 VM_ORR 为例：

![](https://bbs.kanxue.com/upload/attach/202504/823643_QCC9TSHVC8A655A.webp) 

操作数不像老版本那样取固定的位了，新版本每个 vm_Handler 的操作数都会取不同的位进行 “|”。例如目的操作数 W10，W10--Opcode[26]|Opcode[15]|Opcode[21]|Opcode[31]|Opcode[29]：

在 0x50CA4：AND W11, W3, #2，W3 的值为 Opcode>>0x1E，即 Opcode[31-30:2]——因为 W3 的值为右移后的结果，所以 W3 的值加上（左移则减对应数）1（代码中#2 为第 1 位），得到 W11 为 Opcode 的 31 位，并且 Opcode[31] 在最后 5 位操作数（“|” 操作）的组合中在第 1 位，表示为 W11--Opcode[31]--1。

只有对金罡大佬的脚本简单改一下即可：

def get_openand1_ORR(opcode):

    return (((opcode & 0x4000000)>>26)<<4 | ((opcode & 0x8000)>>15)<<3 | ((opcode & 0x200000)>>21)<<2 | ((opcode & 0x80000000)>>31)<<1 | ((opcode & 0x20000000)>>29) )  

def get_openand2_ORR(opcode):

    return (((opcode & 0x40000000)>>30)<<4 | ((opcode & 0x20000)>>17)<<3  | ((opcode & 0x1000)>>12)<<2 | ((opcode & 0x8000000)>>27)<<1 | ((opcode & 0x100)>>8) )

def get_imm16_ORR(opcode):

    result = (((opcode & 0x100000)>>20)<<15 | ((opcode & 0x200)>>9)<<14 | ((opcode & 0x800)>>11)<<13 | ((opcode & 0x400)>>10)<<12 | ((opcode & 0x2000)>>13)<<11 | ((opcode & 0x80)>>7)<<10 | ((opcode & 0x40000)>>18)<<9 | ((opcode & 0x40)>>6)<<8 | ((opcode & 0x80000)>>19)<<7 | ((opcode & 0x1000000)>>24)<<6 | ((opcode & 0x10000)>>16)<<5 | ((opcode & 0x2000000)>>25)<<4 | ((opcode & 0x400000)>>22)<<3 | ((opcode & 0x10000000)>>28)<<2 | ((opcode & 0x800000)>>23)<<1 | ((opcode & 0x4000)>>14) )

    return result

                case 24:

                    dcode_insns_status = 1

                    imm16 = get_imm16_ORR(opcode)

                    Xt_ORR = get_openand1_ORR(opcode)

                    Xn_ORR = get_openand2_ORR(opcode)

                    print(f"{pc:08X}  {opcode:08X}  {opcode & 0x3F:02d}(0x{opcode & 0x3F:02X})  {'-'*8}\tORR\t{get_regsiter_name(Xt_ORR)}, {get_regsiter_name(Xn_ORR)}, #{hex(imm16)}")  

4.2、以 VM_B.GT 为例：

![](https://bbs.kanxue.com/upload/attach/202504/823643_4VYJG7ZRFSUYUP2.webp) 

在 Handler 入口还是一些对操作数的组合，之后来到地址：

![](https://bbs.kanxue.com/upload/attach/202504/823643_RXGN5NRDRFXY7JW.webp) 

在地址 0x52540，得到虚拟 PC 的地址，保存到 X6。在 0x5254C，对 W10 进行 *4 操作。之后还是对操作码的类似与二叉树形式的判断，来到地址 0x52600 处：

![](https://bbs.kanxue.com/upload/attach/202504/823643_4WFTN4HCWAVCPGN.webp) 

在地址 0x52600，得到虚拟 PC。在地址 0x52614，得到新的虚拟 PC，X8 = vm_reg_PC + W10 * 4 + 4。之后还是对操作码的类似与二叉树形式的判断，来到地址 0x5262C 处：

对 X9--VM_REG[W8] 进行判断：

若大于向下执行——

![](https://bbs.kanxue.com/upload/attach/202504/823643_YCVPRFA4NUUBQ9Y.webp) 

 ![](https://bbs.kanxue.com/upload/attach/202504/823643_V5J7QNQQS6TEC8J.webp)

若小于等于跳转到地址 loc_52660——

![](https://bbs.kanxue.com/upload/attach/202504/823643_P624XZDNB5XQB27.webp) 

![](https://bbs.kanxue.com/upload/attach/202504/823643_KPG8JF5275537U4.webp)

在地址 52AA4 处，PC=PC+8

两个分支在地址 52AAC 处交汇：

![](https://bbs.kanxue.com/upload/attach/202504/823643_MTA7QHZ8SV6GDKS.webp) 

在地址 52AB8 处，会把新的虚拟 PC 赋值给 [X27,#-0x18]，（X27 为 pVMMemoryEnd，参照金罡大佬文章中的命名）。

上述对于 X9--VM_REG[W8] 的判断后的两个分支的走向，会导致地址 52AE4 处 [X27,#-0x20] 的值为 1（小于等于）或者 2（大于）。接着向后分析至 switch_end（虚拟 PC 更新并开始重新分发）处：

![](https://bbs.kanxue.com/upload/attach/202504/823643_V8NMWTHZQ8FZNF2.webp) 

若 branch_control_flow==2（X9--VM_REG[W8]>0），会进入 mainDispatcher，接着向后分析：

![](https://bbs.kanxue.com/upload/attach/202504/823643_AT9358RCF67FKUB.webp) 

在地址 4F1B0 处，开始取指（此时已开始 PC＋4 的逻辑了），在地址 4F1F0 处判断，若

branch_control_flow==2，执行地址 4F1FC 处代码，将 branch_control_flow 设置为 3。在 PC＋4 的逻辑执行后，将又会进入地址 54F48 代码，因为此时 branch_control_flow==3，会执行下述代码：

![](https://bbs.kanxue.com/upload/attach/202504/823643_DT5ASCJSACCVUD4.webp) 

将 branch_control_flow 重新置为 0，并在 [X27,#-0x18] 中取出 PC（B.GT 后 offset 的值），

继续向后执行：

![](https://bbs.kanxue.com/upload/attach/202504/823643_SY5NM2QCPG62Y83.webp) 

在地址 54FEC 处，*(_QWORD *)P_vm_reg_PC = vm_reg_PC。继续向后执行，会重新来到取指处（在 B.GT 后的 offset 处取指，即后续开始执行 offset 处逻辑）：

![](https://bbs.kanxue.com/upload/attach/202504/823643_WY6SV2W8JUBBMNH.webp) 

若 branch_control_flow==1（X9--VM_REG[W8]<=0），会进入地址 54F7C：

![](https://bbs.kanxue.com/upload/attach/202504/823643_JARESKDMDWWFM6D.webp) 

依然会将 branch_control_flow 重新置为 0，并在 [X27,#-0x18] 中取出 PC（此时 PC 为 PC=PC+8，即 B.GT 后第二条指令），继续向后执行：

![](https://bbs.kanxue.com/upload/attach/202504/823643_3FRE5ZH3R9J56Z6.webp) 

在地址 54FEC 处，*(_QWORD *)P_vm_reg_PC = vm_reg_PC。继续向后执行，会重新来到取指处（在 B.GT 后第二条指令，即后续开始执行 B.GT 后第二条指令处逻辑）：

![](https://bbs.kanxue.com/upload/attach/202504/823643_MFVSTTXP28XXHA4.webp) 

综上：可梳理处逻辑

![](https://bbs.kanxue.com/upload/attach/202504/823643_3CXGJUGWSUJSFYV.webp) 

4.3、以 VM_BLR 为例：

在 Handler 入口处不再赘述，直接到地址 55D5C 处：

![](https://bbs.kanxue.com/upload/attach/202504/823643_TAWW6MDQKT64XSP.webp) 

向后执行到地址 55DCC 处：

![](https://bbs.kanxue.com/upload/attach/202504/823643_BXW2V6YP6Q8CX7X.webp) 

在地址 55DCC 逻辑与 VM_B.GT 逻辑相似，在地址 55E08 处会把 branch_control_flow 设置为 2。向后执行到地址 55E14 处，设置虚拟 LR 为 vm_reg_PC + 8。向后执行来到地址 4F194 处：

![](https://bbs.kanxue.com/upload/attach/202504/823643_DMBK3TQFTBPNYVJ.webp) 

此处会将虚拟 PC 与真实的 LR 进行比较（老版本是将 PC 与 0 进行比较）。在 VM 解释器入口处会把 [SP,#0x20] 设置为外部函数的跳板地址，把 [SP,#0x28] 设置为真实的 LR；并设置 [SP] 及 [SP,#0x10] 的位置为虚拟寄存器 X4、X5：

  
      ![](https://bbs.kanxue.com/upload/attach/202504/823643_FAS3A5XU24Q28JY.webp)

![](https://bbs.kanxue.com/upload/attach/202504/823643_HMYCCK7HCPDCQDC.webp)

![](https://bbs.kanxue.com/upload/attach/202504/823643_DSW3J7EQ6QHG88K.webp)![](https://bbs.kanxue.com/upload/attach/202504/QH8P6GE44PU8VVH.jpg) 

 ![](https://bbs.kanxue.com/upload/attach/202504/823643_DWAFB3VTHNU99C6.webp)

 ![](https://bbs.kanxue.com/upload/attach/202504/823643_QY8GZJ46YJX94ZE.webp)

![](https://bbs.kanxue.com/upload/attach/202504/823643_KU9CU7J5QUPA694.webp)

![](https://bbs.kanxue.com/upload/attach/202504/823643_APRF9UQ5SY6KBYA.webp)

若在地址 4F19C 处满足条件，将会进入地址 54DC8 处，进行外部函数的调用：

![](https://bbs.kanxue.com/upload/attach/202504/823643_KKCF879KKM534J2.webp) 

总结：此版本较老版本的进步在于：

1、抛弃 vm_Handler 跳转表结构（容易被发现），改用二叉树结构。

2、每个 vm_Handler 中操作数都会取不同的位进行 “|”。

3、在解释器中加入了地址混淆，不能 F5。

个人觉得需要改进的地方（可以参照 X86 的 VMProtect）：

1、取指令的地址不要固定——在 X86 的 VMProtect 中会在每个 vm_Handler 中进行取指（在此 APP 版本中的地址固定为 0x4F1B0）。

2、加入解密因子的思想——在 X86 的 VMProtect 中每次取操作码或操作数都会先通过解密因子进行解密，之后再更新解密因子（在此 APP 版本中通过计算器就能将整个虚拟 opcode 进行拆解）。

3、每个 vm_Handler 间动态进行寻址，即在这个 vm_Handler 执行时，算出下一个 vm_Handler 的地址（在此 APP 版本中，每次执行完 vm_Handle 都会跳转到固定地址进行 PC+4 操作）。

4、在 VM 解释器的中对 vm_info 的设置加入混淆代码，即不要一眼就能看出 vm_reg_PC（虚拟指令指针）、VM_REG（虚拟寄存器）等关键信息是通过哪些真实寄存器去保存（在此 APP 版本中，VM 解释器的入口几个复制操作就是在设置 vm_info）。

5、对标志位以及跳转指令的模拟加大难度，例如 X86 的 VMProtect：

 ![](https://bbs.kanxue.com/upload/attach/202504/823643_2CRTPBBR7MKR93Z.webp)

![](https://bbs.kanxue.com/upload/attach/202504/823643_QG5VSDJGQQGBAZQ.webp) 

![](https://bbs.kanxue.com/upload/attach/202504/823643_2YPAQD67X799QQ6.webp) 

6、加入寄存器轮换的概念，将保存虚拟指令指针、虚拟寄存器等关键信息的真实寄存器在执行多个 vm_Handler 后，对真实寄存器进行轮换（在此 APP 版本中一直没换过）。

为了让大家有个更清晰的概念，后续还会再写一篇 X86_VMP 解释器分析, 准备以 VMProtect3.8 为例（个人觉得 VMProtect3.9 的引擎又回到了 VMProtect2.X 的水平，过于简单），还会在附件中附上通过 unicorn 的模拟执行分析出来的 vm_Handler 的源码（VS 工程），执行结果如下：

![](https://bbs.kanxue.com/upload/attach/202504/823643_E8XDCNGGMV7YPP5.webp)

最后，希望厂商采纳上述建议对 VMP 解释器进行改进，保护我们用户的安全，让我们大家一起为安全行业的进步共同努力吧！！！

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#逆向分析](forum-161-1-118.htm)