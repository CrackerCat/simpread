> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282681.htm)

> [原创] 某视频平台关键 so vm 解释器还原

0x00 前言
=======

> 曾经的曾经，我们还是懵懂无知 de 看待这个世界，直到被世界鞭挞的体无完肤，才对世界有一点点理解。

将 metasec_xx.so 拖入 ida，ida 很自信的帮我们把 so 分析完毕，查看区段信息，ok，没有压缩或加密，可以直接静态分析了 (ida server 总是断，用 unidbg 调试没有 ida 直观，lldb/gdb 又太古老了，不如使用 idapython 调 unicorn)，找到入口，开始分析。

![](https://bbs.kanxue.com/upload/attach/202407/851856_8UJSGVPA7AX72V6.png)

0x01 开始分析
=========

跟踪关键算法从 java 层调用到 native 层之后，最终会调用一个很大的函数

![](https://bbs.kanxue.com/upload/attach/202407/851856_EGNQKUDCVCNVC5J.png)

F5 状态下

![](https://bbs.kanxue.com/upload/attach/202407/851856_C3J3DZJ8TCQAS6D.png)

可以确定该函数是 vm 的入口，并且有混淆，使 ida 反编译失败，看看汇编

![](https://bbs.kanxue.com/upload/attach/202407/851856_XK2AF45QEBD9PPN.png)

ok，不是 ollvm 的混淆。

0x02 混淆及去混淆
===========

分析一下这段混淆的含义

```
MOV             W9, #9        ; 全新的变量赋值
STR             X9, [SP,#0xA0+var_58] ; 将这个写入到堆栈中
LDR             X9, [SP,#0xA0+var_58] ; 又取出
MOV             W15, #0x11    ; 全新的变量赋值
STR             X15, [SP,#0xA0+var_58] ; 写入又取出
LDR             X15, [SP,#0xA0+var_58]
ADD             X0, X9, #1    ; x0=x9+1=10
ADD             X3, X9, #2    ; x3=x9+2=11
MUL             X9, X0, X9    ; x9=x0*x9=90
MOV             X7, #0xAAAAAAAAAAAAAAAA ; 全新的变量赋值
ADD             X0, X15, #1   ; x0=x15+1=0x12
MUL             X9, X9, X3    ; x9=x3*x9=11*90=(x9*(x9+1)*(x9+2))=0x3de
MOVK            X7, #0xAAAB   ; x7=0xAAAAAAAAAAAAAAAB
ADD             X3, X15, #2   ; x3=x15+2=0x13
MUL             X15, X0, X15  ; x15=x15*x0=0x11*0x12
UMULH           X0, X9, X7    ; x0=(x9*x7)>>48=0x294
MUL             X15, X15, X3  ; x15=x15*x3=0x11*0x13=(x15*(x15+1)*(x15+2))
LSR             X3, X0, #1    ; x3=x0/2
LSL             X3, X3, #1    ; x3=x3*2=x0
ADD             X0, X3, X0,LSR#1 ; X0 = X3 + (X0 >> 1)=x0+x0/2=1.5*x0=0x3de（应该是用了数学技巧，矩阵转换？）
UMULH           X3, X15, X7
SUB             X9, X9, X0    ; x9=x9-x0=0
LSR             X0, X3, #1
LSL             X0, X0, #1    ; x0=x3
ADD             X0, X0, X3,LSR#1 ; X0 = X0 + (X3 >> 1)=x15
ADRP            X3, #loc_4A85C@PAGE ; 地址0
SUB             X15, X15, X0  ; x15=x15-x0=0
ADRP            X0, #loc_4ACF8@PAGE ; 地址1
ADD             X3, X3, #loc_4A85C@PAGEOFF
ADD             X0, X0, #loc_4ACF8@PAGEOFF
ADD             X9, X9, X3    ; x9=地址0
ADD             X15, X15, X0  ; x15=地址1
CMP             W14, #3       ; 获取bytecode的控制码
CSEL            X9, X9, X15, CC ; 根据上面的比较判断是去地址0，还是地址1
BR              X9

```

可以看到在写地址前，和比较前的一些运算完全没用，关键的只是后面地址和判断条件，那么去混淆的时候只要保存这块就行。

先手动去混淆

```
CMP             W14, #3
B.CC            loc_4A85C
B               loc_4ACF8
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP
NOP

```

再看看后面是不是都是这样的

![](https://bbs.kanxue.com/upload/attach/202407/851856_6XSMA25UAGDG2M5.png)

可以确定，该类混淆是通过在跳转指令中将跳转地址转为计算值，比较后写入到寄存器中实现跳转，ida 默认没有自动给你分析这个结果，导致反编译失败 (中间的多余计算也是一个干扰)。

根据正确的跳转方式，保留关键的寄存器，其他和跳转无关的全部 nop(全是无意义的指令 (ps: 其实有些也不是))

写去混淆脚本

```
import idaapi
import idc
from unicorn import *
from unicorn.arm64_const import *
from keystone import *
 
BASE_reg=0x81
class txxxxk:
    def __init__(self,address,size) -> None:
        self.start_addre=address
        self.size=size
        self.mu = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
        self.mu.mem_map(address&0xfff000,0x20000000)
        data = idaapi.get_bytes(address,size)
        self.mu.mem_write(address,data)
        self.mu.reg_write(UC_ARM64_REG_SP,0x11000000)
        self.mu.hook_add(UC_HOOK_CODE, self.hook_code)
        self.cmp_reg_num=0
        self.no_nop=[]
        self.br_remake=[]
        self.br_reg=[]
        self.b_addr1=0
        self.b_addr2=0
        self.cmp_seg=""
        self.ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
         
    def hook_code(self,mu, address, size, user_data):
        print("%x"%address)
        insn=idaapi.insn_t()
        idaapi.decode_insn(insn,address)
        dism=idc.generate_disasm_line(address,0)
        if insn.itype==idaapi.ARM_cmp:
            self.cmp_reg_num=insn.Op1.reg-BASE_reg
            self.no_nop.append(address)
        if insn.itype == idaapi.ARM_csel:
            self.b_addr1=self.mu.reg_read(UC_ARM64_REG_X0+insn.Op2.reg-BASE_reg)
            print(insn.Op3.reg-BASE_reg)
            self.b_addr2=self.mu.reg_read(UC_ARM64_REG_X0+insn.Op3.reg-BASE_reg)
            self.br_reg.append(insn.Op2.reg-BASE_reg)
            self.br_reg.append(insn.Op3.reg-BASE_reg)
            print("跳转地址 %x"%self.b_addr1)
            print("跳转地址 %x"%self.b_addr2)
            self.br_remake.append(address)
            self.no_nop.append(address)
            self.cmp_seg=dism.split(",")[-1].split(" ")[-1]
        if insn.itype==idaapi.ARM_br:
            self.br_remake.append(address)
            self.no_nop.append(address)
 
    def start(self):
        try:
            self.mu.emu_start(self.start_addre,self.start_addre+self.size)
        except UcError as e:
            if e.errno==UC_ERR_EXCEPTION:
                print("go on")
            else:
                print(e)
                print("ESP = %x" %self.mu.reg_read(UC_ARM64_REG_SP))
                return
        self.check_reg()
        print("no_nop list ")
        print(self.no_nop)
        print("br_reg list ")
        print(self.br_reg)
        print("br list")
        print(self.br_remake)
        self.nop()
        self.change_ida_byte()
 
    def check_reg(self):
        i=self.size
        nop_list=[]
        while(i>=0):
            insn=idaapi.insn_t()
            idaapi.decode_insn(insn,self.start_addre+i)
 
            flag=False
            for op in insn.ops:
                if op.reg!=0 and (op.reg-BASE_reg) in self.br_reg:
                    flag=True
            for op in insn.ops:
                if flag:
                    if op.reg!=0 and (op.reg-BASE_reg) not in self.br_reg and op.reg!=0xa1:
                        self.br_reg.append(op.reg-BASE_reg)
                    print("%x 参与计算的其他寄存器 %d"%(self.start_addre+i,op.reg-BASE_reg))
                    nop_list.append(self.start_addre+i)
            i-=4
        for no_nop_i in self.no_nop:
            if no_nop_i in nop_list:
                nop_list.remove(no_nop_i)
        j=0
        while(j
```

去混淆后，ida 再次反编译查看

![](https://bbs.kanxue.com/upload/attach/202407/851856_VNF9JUAXCMB45RN.png)

可以正常解析了，接下来我们要做的就简单了只需要对这个函数进行逐 byte 解析并记录其过程即可。

0x03 VM 解释流程
============

前面说过，该函数是一个 VM 解释的函数，VM 里执行的二进制代码我把他统称为 vm_opcode，这个 vm 里的 vm_opcode 大概长这样

![](https://bbs.kanxue.com/upload/attach/202407/851856_TZGSFQPKR85ZDAH.png)

前面去混淆的代码就是在对这段数据进行一个翻译执行。动静态执行分析一下，解释流程大概是这样的

![](https://bbs.kanxue.com/upload/attach/202407/851856_BFXDTD6V759K8E6.png)

关于如何分析 vm 和 vm 的结构可以看看看雪的月板第一的文章 `https://bbs.kanxue.com/thread-282300.htm`

0x04 还原成 C 代码
=============

在上面的文章中，大佬已将伪代码一一列出，最后的做法是在这个基础上纯看代码静态分析，在他的基础上做了点改进

`将伪代码用函数表示，并用真实的堆栈和真实的c语句将其打印，对于跳转的指令在前面加上c的跳转指示表，然后通过gcc编译，将编译的结果放入到ida中进行反编译，这样做的目的是在编译时会对编译语句进行编译优化，去除一下冗余和重复计算，这样之后再用ida的反编译优化，再进行了一次的优化，使得其读起来没那么复杂`

```
#include #include typedef void (*PFN_CALLSTUB)(uint64_t, void *);
 
void foo_000DA700(uint64_t *p_args, uint64_t *g_vars, uint64_t *p_funcs, PFN_CALLSTUB callstub) {
  uint64_t r0 = 0;
  uint64_t r1 = 0;
  uint64_t r2 = 0;
  uint64_t r3 = 0;
  uint64_t r4 = (uint64_t)p_args;
  uint64_t r5 = (uint64_t)g_vars;
  uint64_t r6 = (uint64_t)p_funcs;
  uint64_t r7 = (uint64_t)callstub;
  uint64_t r8 = 0;
  uint64_t r9 = 0;
  uint64_t r10 = 0;
  uint64_t r11 = 0;
  uint64_t r12 = 0;
  uint64_t r13 = 0;
  uint64_t r14 = 0;
  uint64_t r15 = 0;
  uint64_t r16 = 0;
  uint64_t r17 = 0;
  uint64_t r18 = 0;
  uint64_t r19 = 0;
  uint64_t r20 = 0;
  uint64_t r21 = 0;
  uint64_t r22 = 0;
  uint64_t r23 = 0;
  uint64_t r24 = 0;
  uint64_t r25 = 0;
  uint64_t r26 = 0;
  uint64_t r27 = 0;
  uint64_t r28 = 0;
  uint64_t r30 = 0;
  uint64_t r31 = 0;
 
  uint64_t field_120 = 0;
  uint64_t field_128 = 0;
 
  uint8_t stack_buffer[0x420];
  uint64_t r29 = (uint64_t)&stack_buffer[sizeof(stack_buffer)];
 
L_000DA700: r29 = r29 + -0x420;
L_000DA704: *(uint64_t *)(r29+0x418) = r31;
L_000DA708: *(uint64_t *)(r29+0x410) = r30;
L_000DA70C: *(uint64_t *)(r29+0x408) = r23;
L_000DA710: *(uint64_t *)(r29+0x400) = r22;
L_000DA714: *(uint64_t *)(r29+0x3f8) = r21;
L_000DA718: *(uint64_t *)(r29+0x3f0) = r20;
L_000DA71C: *(uint64_t *)(r29+0x3e8) = r19;
L_000DA720: *(uint64_t *)(r29+0x3e0) = r18;
L_000DA724: *(uint64_t *)(r29+0x3d8) = r17;
L_000DA728: *(uint64_t *)(r29+0x3d0) = r16;
L_000DA72C: r16 = r0 | r7;
L_000DA730: r1 = 0xee572c;
L_000DA734: *(uint64_t *)(r29+0x50) = r1;
L_000DA738: r1 = 0xee5744; 
```

大概样子

![](https://bbs.kanxue.com/upload/attach/202407/851856_NAQFCBMKUBADJUT.png)

这样之后就比纯看来的好多了

0x05 四神算法还原
===========

暂时没做，只做了 vm 解释器的还原，后面看看有没有需求，然后做做看。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 18 小时前 被 zhnnmsl 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)