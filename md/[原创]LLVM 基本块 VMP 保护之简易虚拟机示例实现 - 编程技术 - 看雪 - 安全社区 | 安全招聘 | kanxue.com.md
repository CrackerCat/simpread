> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287259.htm)

> [原创]LLVM 基本块 VMP 保护之简易虚拟机示例实现

大家好，我是武汉科锐逆向 CR49 期的学员，这篇文章是为我最近开发的基于 LLVM 的混淆框架中的基本块 VMP 保护所写的验证 Demo 的解析，希望能够帮助到需要开发虚拟机保护的朋友们。

我们这次写的是个简单指令集的堆栈式虚拟机，堆栈式虚拟机相对于寄存器虚拟机虽然效率不及寄存器式虚拟机但是指令集可以足够简单对逆向还原能够造成较大的困扰，同时能够较好的产生混淆指令，所以我们选择堆栈式虚拟机，接下来我们谈谈虚拟机的结构。  
首先对于虚拟机结构设计在看雪中已经有很多文章阐述过常见的虚拟机结构，通常都是`Interpreter + opcode + handleTable + VMContext`的形式如下图所示

![](https://bbs.kanxue.com/upload/attach/202506/983513_HGGNA4PKJ87ZEG5.webp)

整个流程就是模拟 CPU 的行为进行。  
虚拟机执行循环:

指令集我们可以根据需要虚拟化的目标对象来进行虚拟化指令集设计，比如如果你仅仅想保护一段核心的算法其实只需要模拟出常见的计算指令即可，如果你想要保护整个函数则需要模拟出整个指令集，我们作为演示程序仅做出一些简单指令的模拟，**且对于指令集由于具有 Handle 表的存在所以我们可以将指令的 Opcode 设计成线性递增的这样可以直接通过下标从 Handle 表中获取对应的 Handle.**

我们暂且模拟上面没有退格的指令，在上述的虚拟机结构体系中每一个指令都代表着一个 Handle 该 Handle 存在一个全局数组中被称为 Handle 表。如下所示

此 handle 表的顺序是随机的，这样我们就可以通过随机重排 Handle 表来保证 HandleOpcode 的随机这样就导致攻击者无法通过一次逆向解决整个保护系统，我们同时还可以对 Handle 进行膨胀混淆，通过**虚假控制流** **随机控制流** **膨胀混淆** 以阻扰攻击者对 Handle 进行特征工程，这样能够避免低级攻击者的脚本行为。在这个基础上由于此 Demo 是用于我的框架之中的所以我还为他加了一个单字节密钥，此密钥将会作用于每一个基本块，在虚拟机解释执行的时候将会自动利用该密钥解密基本块中的 VMOpcode 当然这里可以选用其他更好的加密方案这篇文章的重点不在这里所以就选择了最简单的方案。如下

VMContext 是虚拟机的环境控制块，该结构主要负责保证寄存器环境的正确性，其中需要保存常见的寄存器环境如下，在 VMContext 中我们有一个`status`成员该成员用于标识虚拟机是否可以返回。

在这里我们可以将指令集中的寄存器编号设置为 `00 04 08 0c 10`这样的序列因为这样我们在访问寄存器的时候可以通过转换`Ctx` `Int*`通过下标来获取到寄存器的值。如下所示

然后我们还需要用堆申请一个空间用作虚拟机栈

最重要的就是 Handle 的设计，因为 Handle 是正确模拟指令操作，以及直接操作 VMContext 对象的函数，我们需要根据堆栈式虚拟机的指令特性正确写出 Handle。如下

要想要 Handle 能被正确的执行我们还需要一个解释器，该解释器模仿 CPU 的行为，即取`指令->译码->执行` 在我们的保护程序中还需要对 Opcode 进行解密。

实话说解释器其实并没有太多的技术难点，对于虚拟机保护而言更重要的是如何保证原函数的 API 调用以及跳转语句的翻译和返回才是真正的难点。

因为这个还是个 Demo 程序所以我们进入虚拟机，还不是那么方便因为正常的虚拟机需要在编译期或者插入的时候计算出解释器到被保护的函数的距离然后在函数开头进行跳转，在函数的结尾跳回来，但是我们全程在 VS 中演示且因为我懒，所以导致没有正确的调用栈，不过这也不是什么难事，我们只需要在写 VMCode 时候，手动加上上一层函数的栈底即可。

最后我们可以看下运行的效果  
![](https://bbs.kanxue.com/upload/attach/202506/983513_CGC9PQR58KD6R9U.webp)  
换一个参数  
![](https://bbs.kanxue.com/upload/attach/202506/983513_8AXTGV79AYFYSZP.webp)  
对于完整的虚拟保护兼容性是最大的问题，但是如果我们能够想办法将虚拟机保护的粒度减少到基本块也能避开一部分的兼容性问题，当然基于 LLVM IR 来设计开发虚拟化保护也能让你少做部分工作。  
贴出全部源码大家可以自己复现并且在这基础上增强虚拟机，并且尝试进行 PIC(大概就是 Malloc 可能需要替换成 VirtualAlloc 然后用 PEB 通过 APIHash 拿到指针后调用即可) 然后植入到汇编中去, 谢谢大家的阅读，如果有什么不对的地方也烦请各位在评论区指出。

`00``(参数指示位)` `000000` `Handle idx`

`应该存在一个表用于随机映射handle`

`00` `vPop Reg32 弹出到寄存器`

`01` `vPush Reg32 压入寄存器`

`02` `vPush Imm32 压入立即数`

`03` `vAdd 加法`

`04` `vSub 减法`

`05` `vDiv 除法`

`06` `vMul 乘法`

`07` `vRSH Imm32右移`

`08` `vLSH Imm32左移`

`09` `vAnd 与运算`

`0A` `vOr  或运算`

`0B` `vXor 异或运算`

`0C` `vNot 非运算`

`0D` `vRet 返回指令`

 `0E` `vNop 空指令`

 `0F` `vJmp 跳转指令`

 `10` `vJZ` `0``跳转`

 `11` `vJB stack1 > stack2 则跳转`

 `12` `vRead Imm32 读取``4``字节到虚拟栈上`

 `13` `vRead Reg 读取``4``字节到虚拟栈上`

`0E` `vRead 读取栈顶地址处的``4``字节到栈顶取代处栈顶地址`

`00``(参数指示位)` `000000` `Handle idx`

`应该存在一个表用于随机映射handle`

`00` `vPop Reg32 弹出到寄存器`

`01` `vPush Reg32 压入寄存器`

`02` `vPush Imm32 压入立即数`

`03` `vAdd 加法`

`04` `vSub 减法`

`05` `vDiv 除法`

`06` `vMul 乘法`

`07` `vRSH Imm32右移`

`08` `vLSH Imm32左移`

`09` `vAnd 与运算`

`0A` `vOr  或运算`

`0B` `vXor 异或运算`

`0C` `vNot 非运算`

`0D` `vRet 返回指令`

 `0E` `vNop 空指令`

 `0F` `vJmp 跳转指令`

 `10` `vJZ` `0``跳转`

 `11` `vJB stack1 > stack2 则跳转`

 `12` `vRead Imm32 读取``4``字节到虚拟栈上`

 `13` `vRead Reg 读取``4``字节到虚拟栈上`

`0E` `vRead 读取栈顶地址处的``4``字节到栈顶取代处栈顶地址`

`/``*` `HANDLE` `*``/`

`typedef void (``*``HANDLE)(VMContext& ctx);`

`HANDLE g_handle[]` `=`

`{`

 `&vPopReg32,`

 `&vPushReg32,`

 `&vPushImm32,`

 `&vAdd,`

 `&vSub,`

 `&vDiv,`

 `&vMul,`

 `&vRSH,`

 `&vLSH,`

 `&vAnd,`

 `&vOr,`

 `&vXor,`

 `&vNot,`

 `&vRet,`

 `&vRead`

`};`

`/``*` `HANDLE` `*``/`

`typedef void (``*``HANDLE)(VMContext& ctx);`

`HANDLE g_handle[]` `=`

`{`

 `&vPopReg32,`

 `&vPushReg32,`

 `&vPushImm32,`

 `&vAdd,`

 `&vSub,`

 `&vDiv,`

 `&vMul,`

 `&vRSH,`

 `&vLSH,`

 `&vAnd,`

 `&vOr,`

 `&vXor,`

 `&vNot,`

 `&vRet,`

 `&vRead`

`};`

`unsigned char DecrypyOpcode(unsigned char opcode)`

`{`

 `return` `szXorkey ^ opcode;`

`}`

`unsigned char DecrypyOpcode(unsigned char opcode)`

`{`

 `return` `szXorkey ^ opcode;`

`}`

`/``*` `x86堆栈环境` `*``/`

`typedef struct _VMContext`

`{`

 `/``*`

 `eax` `00`

 `ebx` `04`

 `ecx` `08`

 `edx` `0c`

 `ebp` `10`

 `esp` `14`

 `edi` `18`

 `esi` `1c`

 `eflags` `20`

 `*``/`

 `unsigned` `int` `eax;`

 `unsigned` `int` `ebx;`

 `unsigned` `int` `ecx;`

 `unsigned` `int` `edx;`

 `unsigned` `int``*` `ebp;`

 `unsigned` `int``*` `esp;`

 `unsigned` `int``*` `vEsp;`

 `unsigned` `int` `edi;`

 `unsigned` `int` `esi;`

 `unsigned` `int` `eflags;`

 `unsigned char``*` `opcode;`

 `int` `state;`

`}VMContext,` `*``PVMContext;`

`/``*` `x86堆栈环境` `*``/`

`typedef struct _VMContext`

`{`

 `/``*`

 `eax` `00`

 `ebx` `04`

 `ecx` `08`

 `edx` `0c`

 `ebp` `10`

 `esp` `14`

 `edi` `18`

 `esi` `1c`

 `eflags` `20`

 `*``/`

 `unsigned` `int` `eax;`

 `unsigned` `int` `ebx;`

 `unsigned` `int` `ecx;`

 `unsigned` `int` `edx;`

 `unsigned` `int``*` `ebp;`

 `unsigned` `int``*` `esp;`

 `unsigned` `int``*` `vEsp;`

 `unsigned` `int` `edi;`

 `unsigned` `int` `esi;`

 `unsigned` `int` `eflags;`

 `unsigned char``*` `opcode;`

 `int` `state;`

`}VMContext,` `*``PVMContext;`

`void vPopReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `((``int``*``)(&ctx))[reg` `/` `4``]` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `return``;`

`}`

`void vPopReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `((``int``*``)(&ctx))[reg` `/` `4``]` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `return``;`

`}`

`void VmInit(VMContext``*` `ctx, unsigned char``*` `opcode)`

`{`

 `__asm {`

 `mov eax, ctx`

 `mov dword ptr [eax` `+` `0``], eax`

 `mov dword ptr [eax` `+` `4``], ebx`

 `mov dword ptr [eax` `+` `8``], ecx`

 `mov dword ptr [eax` `+` `0xc``], edx`

 `mov dword ptr [eax` `+` `0x10``], ebp`

 `mov dword ptr [eax` `+` `0x14``], esp`

 `mov dword ptr [eax` `+` `0x18``], edi` `/``/` `直接保存，无需中转`

 `mov dword ptr [eax` `+` `0x1C``], esi` `/``/` `直接保存，无需中转`

 `/``/` `保存EFLAGS`

 `pushfd`

 `pop dword ptr ctx.eflags` `/``/` `直接弹出到内存`

 `}`

 `ctx``-``>state` `=` `0``;`

 `/``*` `赋值OPCODE的EIP` `*``/`

 `ctx``-``>opcode` `=` `opcode;`

 `/``*` `申请栈空间设置栈指针` `*``/`

 `ctx``-``>vEsp` `=` `(unsigned` `int``*``)malloc(STACK_SIZE);`

 `ctx``-``>vEsp` `=` `(unsigned` `int``*``)((unsigned char``*``)ctx``-``>vEsp` `+` `STACK_SIZE);`

 `g_orgStack` `=` `ctx``-``>vEsp;`

 `return``;`

`}`

`void VmInit(VMContext``*` `ctx, unsigned char``*` `opcode)`

`{`

 `__asm {`

 `mov eax, ctx`

 `mov dword ptr [eax` `+` `0``], eax`

 `mov dword ptr [eax` `+` `4``], ebx`

 `mov dword ptr [eax` `+` `8``], ecx`

 `mov dword ptr [eax` `+` `0xc``], edx`

 `mov dword ptr [eax` `+` `0x10``], ebp`

 `mov dword ptr [eax` `+` `0x14``], esp`

 `mov dword ptr [eax` `+` `0x18``], edi` `/``/` `直接保存，无需中转`

 `mov dword ptr [eax` `+` `0x1C``], esi` `/``/` `直接保存，无需中转`

 `/``/` `保存EFLAGS`

 `pushfd`

 `pop dword ptr ctx.eflags` `/``/` `直接弹出到内存`

 `}`

 `ctx``-``>state` `=` `0``;`

 `/``*` `赋值OPCODE的EIP` `*``/`

 `ctx``-``>opcode` `=` `opcode;`

 `/``*` `申请栈空间设置栈指针` `*``/`

 `ctx``-``>vEsp` `=` `(unsigned` `int``*``)malloc(STACK_SIZE);`

 `ctx``-``>vEsp` `=` `(unsigned` `int``*``)((unsigned char``*``)ctx``-``>vEsp` `+` `STACK_SIZE);`

 `g_orgStack` `=` `ctx``-``>vEsp;`

 `return``;`

`}`

`void vPopReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `((``int``*``)(&ctx))[reg` `/` `4``]` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `return``;`

`}`

`void vPushReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `ctx.vEsp``-``-``;`

 `*``(ctx.vEsp)` `=` `((unsigned` `int``*``)(&ctx))[reg` `/` `4``];`

 `return``;`

`}`

`void vPushImm32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `ctx.vEsp``-``-``;`

 `*``ctx.vEsp` `=` `Imm32;`

`}`

`void vAdd(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `+` `num2;`

 `return``;`

`}`

`void vSub(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `-` `num2;`

 `return``;`

`}`

`void vDiv(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `/` `num2;`

 `return``;`

`}`

`void vMul(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `*` `num2;`

 `return``;`

`}`

`void vRSH(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `*``ctx.vEsp >>``=` `Imm32;`

`}`

`void vLSH(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `*``ctx.vEsp <<``=` `Imm32;`

`}`

`void vAnd(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 & num2;`

 `return``;`

`}`

`void vOr(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 | num2;`

 `return``;`

`}`

`void vXor(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 ^ num2;`

 `return``;`

`}`

`void vNot(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `*``ctx.vEsp` `=` `~(``*``ctx.vEsp);`

 `return``;`

`}`

`void vRet(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `ctx.state` `=` `0xff``;`

 `return``;`

`}`

`void vRead(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `int``*` `addr` `=` `(``int``*``)(``*``ctx.vEsp);`

 `*``ctx.vEsp` `=` `*``addr;`

 `return``;`

`}`

`void vPopReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `((``int``*``)(&ctx))[reg` `/` `4``]` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `return``;`

`}`

`void vPushReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `ctx.vEsp``-``-``;`

 `*``(ctx.vEsp)` `=` `((unsigned` `int``*``)(&ctx))[reg` `/` `4``];`

 `return``;`

`}`

`void vPushImm32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `ctx.vEsp``-``-``;`

 `*``ctx.vEsp` `=` `Imm32;`

`}`

`void vAdd(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `+` `num2;`

 `return``;`

`}`

`void vSub(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `-` `num2;`

 `return``;`

`}`

`void vDiv(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `/` `num2;`

 `return``;`

`}`

`void vMul(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `*` `num2;`

 `return``;`

`}`

`void vRSH(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `*``ctx.vEsp >>``=` `Imm32;`

`}`

`void vLSH(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `*``ctx.vEsp <<``=` `Imm32;`

`}`

`void vAnd(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 & num2;`

 `return``;`

`}`

`void vOr(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 | num2;`

 `return``;`

`}`

`void vXor(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 ^ num2;`

 `return``;`

`}`

`void vNot(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `*``ctx.vEsp` `=` `~(``*``ctx.vEsp);`

 `return``;`

`}`

`void vRet(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `ctx.state` `=` `0xff``;`

 `return``;`

`}`

`void vRead(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `int``*` `addr` `=` `(``int``*``)(``*``ctx.vEsp);`

 `*``ctx.vEsp` `=` `*``addr;`

 `return``;`

`}`

`int` `VmEntry(unsigned char``*` `opcode)`

`{`

 `VMContext ctx{``0``};`

 `/``/``VmInit(&ctx, opcode);`

 `unsigned` `int``*` `caller_ebp;`

 `__asm {`

 `mov eax, [ebp]` `/``/` `获取add函数的ebp`

 `mov caller_ebp, eax`

 `}`

 `VmInit(&ctx, opcode);`

 `/``/` `手动设置正确的ebp`

 `ctx.ebp` `=` `caller_ebp;`

 `while` `(true)`

 `{`

 `if` `(ctx.state` `=``=` `0xff``)`

 `break``;`

 `unsigned char op` `=` `DecrypyOpcode(GetOpcode(ctx));`

 `g_handle[op](ctx);`

 `}`

 `return` `ctx.eax;`

`}`

`int` `VmEntry(unsigned char``*` `opcode)`

`{`

 `VMContext ctx{``0``};`

 `/``/``VmInit(&ctx, opcode);`

 `unsigned` `int``*` `caller_ebp;`

 `__asm {`

 `mov eax, [ebp]` `/``/` `获取add函数的ebp`

 `mov caller_ebp, eax`

 `}`

 `VmInit(&ctx, opcode);`

 `/``/` `手动设置正确的ebp`

 `ctx.ebp` `=` `caller_ebp;`

 `while` `(true)`

 `{`

 `if` `(ctx.state` `=``=` `0xff``)`

 `break``;`

 `unsigned char op` `=` `DecrypyOpcode(GetOpcode(ctx));`

 `g_handle[op](ctx);`

 `}`

 `return` `ctx.eax;`

`}`

`/``/` `加法字节码`

`unsigned char g_opAdd[]` `=`

`{`

 `/``/` `获取参数``1` `(a)：ebp` `+` `0x08`

 `0x78``,` `0x10``,` `/``/` `vPush ebp`

 `0x7B``,` `0x08``,` `0x00``,` `0x00``,` `0x00``,` `/``/` `vPush` `0x08`

 `0x7A``,` `/``/` `vAdd (计算ebp` `+` `0x08``)`

 `0x77``,` `/``/` `vRead (读取参数``1``)`

 `/``/` `获取参数``2` `(b)：ebp` `+` `0x0C` 

 `0x78``,` `0x10``,` `/``/` `vPush ebp`

 `0x7B``,` `0x0C``,` `0x00``,` `0x00``,` `0x00``,` `/``/` `vPush` `0x0C`

 `0x7A``,` `/``/` `vAdd (计算ebp` `+` `0x0C``)`

 `0x77``,` `/``/` `vRead (读取参数``2``)`

 `/``/` `执行加法`

 `0x7A``,` `/``/` `vAdd (参数``1` `+` `参数``2``)`

 `/``/` `返回结果`

 `0x79``,` `0x00``,` `/``/` `vPop eax (结果保存到eax)`

 `0x74`                           `/``/` `vRet`

`};`

`/``*` `加法调用如下` `*``/`

`__declspec(naked)` `int` `add(``int` `a,` `int` `b)`

`{`

 `__asm {`

 `/``/` `建立栈帧`

 `push ebp`

 `mov ebp, esp`

 `/``/` `保存寄存器`

 `push eax`

 `push ebx`

 `push ecx`

 `push edx`

 `/``/` `调用VM`

 `push offset g_opAdd`

 `call VmEntry`

 `add esp,` `4`

 `/``/` `恢复寄存器 (eax是返回值，不恢复)`

 `pop edx`

 `pop ecx`

 `pop ebx`

 `add esp,` `4`  `/``/` `跳过原eax`

 `/``/` `清理栈帧`

 `mov esp, ebp`

 `pop ebp`

 `ret`

 `}`

`}`

`/``/` `加法字节码`

`unsigned char g_opAdd[]` `=`

`{`

 `/``/` `获取参数``1` `(a)：ebp` `+` `0x08`

 `0x78``,` `0x10``,` `/``/` `vPush ebp`

 `0x7B``,` `0x08``,` `0x00``,` `0x00``,` `0x00``,` `/``/` `vPush` `0x08`

 `0x7A``,` `/``/` `vAdd (计算ebp` `+` `0x08``)`

 `0x77``,` `/``/` `vRead (读取参数``1``)`

 `/``/` `获取参数``2` `(b)：ebp` `+` `0x0C` 

 `0x78``,` `0x10``,` `/``/` `vPush ebp`

 `0x7B``,` `0x0C``,` `0x00``,` `0x00``,` `0x00``,` `/``/` `vPush` `0x0C`

 `0x7A``,` `/``/` `vAdd (计算ebp` `+` `0x0C``)`

 `0x77``,` `/``/` `vRead (读取参数``2``)`

 `/``/` `执行加法`

 `0x7A``,` `/``/` `vAdd (参数``1` `+` `参数``2``)`

 `/``/` `返回结果`

 `0x79``,` `0x00``,` `/``/` `vPop eax (结果保存到eax)`

 `0x74`                           `/``/` `vRet`

`};`

`/``*` `加法调用如下` `*``/`

`__declspec(naked)` `int` `add(``int` `a,` `int` `b)`

`{`

 `__asm {`

 `/``/` `建立栈帧`

 `push ebp`

 `mov ebp, esp`

 `/``/` `保存寄存器`

 `push eax`

 `push ebx`

 `push ecx`

 `push edx`

 `/``/` `调用VM`

 `push offset g_opAdd`

 `call VmEntry`

 `add esp,` `4`

 `/``/` `恢复寄存器 (eax是返回值，不恢复)`

 `pop edx`

 `pop ecx`

 `pop ebx`

 `add esp,` `4`  `/``/` `跳过原eax`

 `/``/` `清理栈帧`

 `mov esp, ebp`

 `pop ebp`

 `ret`

 `}`

`}`

`void``*` `g_orgStack` `=` `nullptr;`

`/``*` `x86堆栈环境` `*``/`

`typedef struct _VMContext`

`{`

 `/``*`

 `eax` `00`

 `ebx` `04`

 `ecx` `08`

 `edx` `0c`

 `ebp` `10`

 `esp` `14`

 `edi` `18`

 `esi` `1c`

 `eflags` `20`

 `*``/`

 `unsigned` `int` `eax;`

 `unsigned` `int` `ebx;`

 `unsigned` `int` `ecx;`

 `unsigned` `int` `edx;`

 `unsigned` `int``*` `ebp;`

 `unsigned` `int``*` `esp;`

 `unsigned` `int``*` `vEsp;`

 `unsigned` `int` `edi;`

 `unsigned` `int` `esi;`

 `unsigned` `int` `eflags;`

 `unsigned char``*` `opcode;`

 `int` `state;`

`}VMContext,` `*``PVMContext;`

`/``*` `HANDLE` `*``/`

`typedef void (``*``HANDLE)(VMContext& ctx);`

`/``*`

`*`  

 `00``(参数指示位)` `000000` `Handle idx`

 `应该存在一个表用于随机映射handle`

 `00` `vPop Reg32 弹出到寄存器`

 `01` `vPush Reg32 压入寄存器`

 `02` `vPush Imm32 压入立即数`

 `03` `vAdd 加法`

 `04` `vSub 减法`

 `05` `vDiv 除法`

 `06` `vMul 乘法`

 `07` `vRSH Imm32右移`

 `08` `vLSH Imm32左移`

 `09` `vAnd 与运算`

 `0A` `vOr  或运算`

 `0B` `vXor 异或运算`

 `0C` `vNot 非运算`

 `0D` `vRet 返回指令`

 `0E` `vNop 空指令`

 `0F` `vJmp 跳转指令`

 `10` `vJZ` `0``跳转`

 `11` `vJB stack1 > stack2 则跳转`

 `12` `vRead Imm32 读取``4``字节到虚拟栈上`

 `13` `vRead Reg 读取``4``字节到虚拟栈上`

 `0E` `vRead 读取栈顶地址处的``4``字节到栈顶取代处栈顶地址`

`*``/`

`/``*`

 `堆栈式虚拟机`

`*``/`

`/``*`

 `ebp`

 `ret`

 `ebp`

 `ret`

 `2`

 `1`

`*``/`

`unsigned char szXorkey` `=` `0x79``;`

`/``/` `加法字节码`

`unsigned char g_opAdd[]` `=`

`{`

 `/``/` `获取参数``1` `(a)：ebp` `+` `0x08`

 `0x78``,` `0x10``,` `/``/` `vPush ebp`

 `0x7B``,` `0x08``,` `0x00``,` `0x00``,` `0x00``,` `/``/` `vPush` `0x08`

 `0x7A``,` `/``/` `vAdd (计算ebp` `+` `0x08``)`

 `0x77``,` `/``/` `vRead (读取参数``1``)`

 `/``/` `获取参数``2` `(b)：ebp` `+` `0x0C` 

 `0x78``,` `0x10``,` `/``/` `vPush ebp`

 `0x7B``,` `0x0C``,` `0x00``,` `0x00``,` `0x00``,` `/``/` `vPush` `0x0C`

 `0x7A``,` `/``/` `vAdd (计算ebp` `+` `0x0C``)`

 `0x77``,` `/``/` `vRead (读取参数``2``)`

 `/``/` `执行加法`

 `0x7A``,` `/``/` `vAdd (参数``1` `+` `参数``2``)`

 `/``/` `返回结果`

 `0x79``,` `0x00``,` `/``/` `vPop eax (结果保存到eax)`

 `0x74`                           `/``/` `vRet`

`};`

`void vPopReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `((``int``*``)(&ctx))[reg` `/` `4``]` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `return``;`

`}`

`void vPushReg32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned char reg` `=` `*``ctx.opcode``+``+``;`

 `ctx.vEsp``-``-``;`

 `*``(ctx.vEsp)` `=` `((unsigned` `int``*``)(&ctx))[reg` `/` `4``];`

 `return``;`

`}`

`void vPushImm32(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `ctx.vEsp``-``-``;`

 `*``ctx.vEsp` `=` `Imm32;`

`}`

`void vAdd(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `+` `num2;`

 `return``;`

`}`

`void vSub(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `-` `num2;`

 `return``;`

`}`

`void vDiv(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `/` `num2;`

 `return``;`

`}`

`void vMul(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1` `*` `num2;`

 `return``;`

`}`

`void vRSH(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `*``ctx.vEsp >>``=` `Imm32;`

`}`

`void vLSH(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `Imm32` `=` `*``(unsigned` `int``*``)ctx.opcode;`

 `ctx.opcode` `+``=` `4``;`

 `*``ctx.vEsp <<``=` `Imm32;`

`}`

`void vAnd(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 & num2;`

 `return``;`

`}`

`void vOr(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 | num2;`

 `return``;`

`}`

`void vXor(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `unsigned` `int` `num1` `=` `*``ctx.vEsp;`

 `ctx.vEsp``+``+``;`

 `unsigned` `int` `num2` `=` `*``ctx.vEsp;`

 `*``ctx.vEsp` `=` `num1 ^ num2;`

 `return``;`

`}`

`void vNot(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `*``ctx.vEsp` `=` `~(``*``ctx.vEsp);`

 `return``;`

`}`

`void vRet(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `ctx.state` `=` `0xff``;`

 `return``;`

`}`

`void vRead(VMContext& ctx)`

`{`

 `unsigned char op` `=` `*``ctx.opcode``+``+``;`

 `int``*` `addr` `=` `(``int``*``)(``*``ctx.vEsp);`

 `*``ctx.vEsp` `=` `*``addr;`

 `return``;`

`}`

`/``*`

`00` `vPop Reg32 弹出到寄存器`

 `01` `vPush Reg32 压入寄存器`

 `02` `vPush Imm32 压入立即数`

 `03` `vAdd 加法`

 `04` `vSub 减法`

 `05` `vDiv 除法`

 `06` `vMul 乘法`

 `07` `vRSH Imm32右移`

 `08` `vLSH Imm32左移`

 `09` `vAnd 与运算`

 `0A` `vOr  或运算`

 `0B` `vXor 异或运算`

 `0C` `vNot 非运算`

 `0D` `vRet 返回指令`

`*``/`

`HANDLE g_handle[]` `=`

`{`

 `&vPopReg32,`

 `&vPushReg32,`

 `&vPushImm32,`

 `&vAdd,`

 `&vSub,`

 `&vDiv,`

 `&vMul,`

 `&vRSH,`

 `&vLSH,`

 `&vAnd,`

 `&vOr,`

 `&vXor,`

 `&vNot,`

 `&vRet,`

 `&vRead`

`};`

`unsigned char DecrypyOpcode(unsigned char opcode)`

`{`

 `return` `szXorkey ^ opcode;`

`}`

`void VmInit(VMContext``*` `ctx, unsigned char``*` `opcode)`

`{`

 `__asm {`

 `mov eax, ctx`

 `mov dword ptr [eax` `+` `0``], eax`

 `mov dword ptr [eax` `+` `4``], ebx`

 `mov dword ptr [eax` `+` `8``], ecx`

 `mov dword ptr [eax` `+` `0xc``], edx`

 `mov dword ptr [eax` `+` `0x10``], ebp`

 `mov dword ptr [eax` `+` `0x14``], esp`

 `mov dword ptr [eax` `+` `0x18``], edi` `/``/` `直接保存，无需中转`

 `mov dword ptr [eax` `+` `0x1C``], esi` `/``/` `直接保存，无需中转`

 `/``/` `保存EFLAGS`

 `pushfd`

 `pop dword ptr ctx.eflags` `/``/` `直接弹出到内存`

 `}`

 `ctx``-``>state` `=` `0``;`

 `/``*` `赋值OPCODE的EIP` `*``/`

 `ctx``-``>opcode` `=` `opcode;`

 `/``*` `申请栈空间设置栈指针` `*``/`

 `ctx``-``>vEsp` `=` `(unsigned` `int``*``)malloc(STACK_SIZE);`

 `ctx``-``>vEsp` `=` `(unsigned` `int``*``)((unsigned char``*``)ctx``-``>vEsp` `+` `STACK_SIZE);`

 `g_orgStack` `=` `ctx``-``>vEsp;`

 `return``;`

`}`

`unsigned char GetOpcode(VMContext& ctx)`

`{`

 `unsigned char szRet` `=` `*``(ctx.opcode);`

 `return` `szRet;`

`}`

`int` `VmEntry(unsigned char``*` `opcode)`

`{`

 `VMContext ctx{``0``};`

 `/``/``VmInit(&ctx, opcode);`

 `unsigned` `int``*` `caller_ebp;`

 `__asm {`

 `mov eax, [ebp]` `/``/` `获取add函数的ebp`

 `mov caller_ebp, eax`

 `}`

 `VmInit(&ctx, opcode);`

 `/``/` `手动设置正确的ebp`

 `ctx.ebp` `=` `caller_ebp;`

 `while` `(true)`

 `{`

 `if` `(ctx.state` `=``=` `0xff``)`

 `break``;`

 `unsigned char op` `=` `DecrypyOpcode(GetOpcode(ctx));`

 `g_handle[op](ctx);`

 `}`

 `return` `ctx.eax;`

`}`

`__declspec(naked)` `int` `add(``int` `a,` `int` `b)`

`{`

 `__asm {`

 `/``/` `建立栈帧`

 `push ebp`

 `mov ebp, esp`

 `/``/` `保存寄存器`

 `push eax`

 `push ebx`

 `push ecx`

 `push edx`

 `/``/` `调用VM`

 `push offset g_opAdd`

 `call VmEntry`

 `add esp,` `4`

 `/``/` `恢复寄存器 (eax是返回值，不恢复)`

 `pop edx`

 `pop ecx`

 `pop ebx`

 `add esp,` `4`  `/``/` `跳过原eax`

 `/``/` `清理栈帧`

 `mov esp, ebp`

 `pop ebp`

 `ret`

 `}`

`}`

`int` `main(``int` `argc, char``*` `argv[], char``*` `envp[])`

`{`

 `int` `a` `=` `add(``1``,` `7``);`

 `printf(``"%d"``, a);`

 `return` `0``;`

`}`

`void``*` `g_orgStack` `=` `nullptr;`

`/``*` `x86堆栈环境` `*``/`

`typedef struct _VMContext`

`{`

 `/``*`

 `eax` `00`

 `ebx` `04`

 `ecx` `08`

 `edx` `0c`

 `ebp` `10`

 `esp` `14`

 `edi` `18`

 `esi` `1c`

 `eflags` `20`

 `*``/`

 `unsigned` `int` `eax;`

 `unsigned` `int` `ebx;`

 `unsigned` `int` `ecx;`

 `unsigned` `int` `edx;`

 `unsigned` `int``*` `ebp;`

 `unsigned` `int``*` `esp;`

 `unsigned` `int``*` `vEsp;`

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)