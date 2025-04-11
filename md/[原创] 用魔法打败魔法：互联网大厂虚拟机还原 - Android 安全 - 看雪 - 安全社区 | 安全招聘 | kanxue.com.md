> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286441.htm#msg_header_h3_14)

> [原创] 用魔法打败魔法：互联网大厂虚拟机还原

前言
==

### 为什么还是虚拟机

有人就要问了，怎么又是虚拟机的文章，我觉得其他领域基本上都有人分析过了再次写重复的文章意义不大，虚拟机保护分析的门槛较高且文章明显偏少于是就选择了它。

虚拟机保护到目前为依然是很多人迈不过去的一个坎，但对于现在逆向工程来说它不是唯一到达目的的捷径，很多时候可以跳过虚拟机的分析对目标 app 的指令和数据流回溯分析，例如最近流行的各种 trace。另外即使能够分析和还原它虚拟机也只占用工程工作量中的少量时间，这个时间比例可能就 5%-10% 左右吧。  
每个人的分析方法思路、使用的工具、对逆向工程的理解都有各不相同，文中只是将我个人的一些分析方法思路分享出来，可能方法不是最好的，或许在有些人眼中觉得你的方法太笨了，每个人都有技术盲点的地方，有交流才会有进步欢迎留言指正我的不足。

随着最近几年对抗的升级现阶段大厂基本上都是虚拟机了，分析的工作量明显上升，一个人对抗大厂多个团队已经显得力不从心，单兵作战早已经成为过去式。

逆向工程从来就不是一个简单的事情，在逆向分析过程中很长时间就是精神上的折磨，时间久了会慢慢的适应这种精神折磨状态，逆向工程概括了方方面面的东西：经验、知识面、理解和认知、天赋，好像这些和我沾边的都不多，有时在逆向分析时会一根筋的钻进死胡同。

环境
--

`App版本`：versionCode='1200280405' versionName='12.28.405'

`手机系统`：android 11/pixel 2 xl

`PC`: macOS 15.3

分析篇
===

代码混淆乱序
------

以函数 vm_interpreter 为例，函数的开始和结束边界清晰，代码块没有穿插到其他函数或位置中，说明混淆或乱序只是在函数的开始和结束地址范围内进行。

*   函数开始  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_U9JBGKMG3U3G2YF.webp)
*   函数结束  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_2852R3H2SN4WZUV.webp)

### 基本块分割

在分析时发现很代码块 (指多连续的基本块) 的终止符使用_**无条件跳转**_连接，通常正常编译生成无条件跳转可能出现在 if then end、if else end、break、continue、switch case、switch case end、loop entry、loop、loop exit 的位置比较多，正常的代码不会出现如此多的无条件跳转，这说明基本块从中间被分割后被随机打乱后使用无跳转条件跳转进行连接。

### 指令分发基本块复制

看来 app 加密开发者还是很懂逆向的，一般来说 handler 执行完成会回到取指的基本块位置，这里将分发取指的基本块进行了复制作为 handler 的后续终止符进行连接，防止一个位置下断或 hook 拿取 opcode 数据。  
但是也不排除是函数 inline 造成多个分发位置。

### 间接跳转

去掉间接跳转的目的是方便在 IDA 中查看分析代码的结构，读取跳转常量数组将真实的地址连接。

```
import idautils
import idc
import idaapi
from keystone import *  # pip3 install keystone-engine
def get_insn_const(addr):
    op_val = None
    if idc.print_insn_mnem(addr) in ['MOV', 'LDR']:
        op_val = idc.get_operand_value(addr, 1)
        if op_val > 0x1000:  # 可能是间接引用
            op_val = idc.get_wide_dword(op_val)
    else:
        raise Exception(f"error ops const: {addr}")
    return op_val
def get_patch_data(addr):
    addr_list = []
    for bl_insn_addr in idautils.XrefsTo(addr):
        bl_insn_addr = bl_insn_addr.frm
        # print(f'L1 {hex(bl_insn_addr)}:')
        for xref_addr_l2 in idautils.XrefsTo(bl_insn_addr):               
            # print(f'\tL2 {hex(xref_addr_l2.frm)}:')
            index = get_insn_const(xref_addr_l2.frm - 4)
            const_table_start = bl_insn_addr + 4
            offset = idaapi.get_dword(const_table_start + index * 4)
            link_target = const_table_start + offset
            addr_list.append({"bl_insn_addr": bl_insn_addr, "patch_addr": xref_addr_l2.frm, "index": index,
                        "offset": offset, "link_target": link_target})
    return addr_list
def print_patch_data(patch_data):
    for item in patch_data:
        print(
            f"bl_insn_addr: {item["bl_insn_addr"]:#x}, patch_addr: {item["patch_addr"]:#x}, index: {item["index"]}, offset: {item["offset"]:#x}, link_target: {item["link_target"]:#x}")
def patch_insns(patch_data):
    index = 0
    for item in patch_data:
        ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)
        asm = f'B {item["link_target"]:#x}'
        print(f'patch addr {item["patch_addr"]:#x}: {asm}')
        encoding, count = ks.asm(asm, as_bytes=True, addr=item["patch_addr"])
        print(encoding)
        for i in range(4):
            idc.patch_byte(item["patch_addr"] + i, encoding[i])
        index += 1
        # if index == 1:
        #     break       
def start():
    modify_x30_func_address = 0x25D00
    patch_data = get_patch_data(modify_x30_func_address)
    print_patch_data(patch_data)
    patch_insns(patch_data)
start()
```

### 调用关系图

![](https://bbs.kanxue.com/upload/attach/202504/271698_BEG894WY6GFS8G6.webp)

### 快速定位虚拟机位置

载入 IDA 分析完成后打开函数视图，函数列表最上方找到 Length 表头点击按函数大小排序，找到第二大小的函数就来到了虚拟机解释器的位置。

*   vm_interpreter  
    vm_interpreter(0x138518) 是虚拟机解析执行的核心，它包含了取指、解码、执行 handler 三个基本的完整流程。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_JKJ62RHGNACPW79.webp)  
    从 cfg 图看比较复杂，其中有很多的 switch 和间接跳转没有识别出来出现在流图中，实际复杂程度比想象中的要复杂得多。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_QMFZXKMPSJ3KUJY.webp)
*   vm_read  
    双击函数列表中的地址 vm_interpreter(0x138518) 来到 vm_interpreter 函数起始的位置，交叉引用来到上级调用函数 **vm_read**。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_599Q4UUYEGZN6W4.webp)
*   vm_entry  
    找到函数 vm_read 起始位置 0x131680，再次交叉引用找到上级调用函数 **vm_entry**。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_MBSGQQ2M2ZBC84G.webp)

### 函数分析

找到 vm_entry 起始地址 0x1313F0，这时可以看到有 31 个引用地址这意味着有 31 个函数被虚拟化执行，找到 2 号引用地址，它是 app 启动后首次调用的虚拟机代码，从这里开始逐步展开分析。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_JPJVEK724JHUMV3.webp)  
双击 2 号交叉引用来到函数 0x6884C，x0-x2 对应函数原型的 3 个参数 bytecode、bc_size、external，其他剩余的参数寄存器是变参部分：寄存器 x3-x7、Q0-Q7、堆栈。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_Z54DX9M4SBPSATV.webp)

#### void vm_entry(void * bytecode, int bc_size, void** external, ...)

描述：准备参数和返回值对象  
`bytecode`: 字节码指针  
`bc_size`: 字节码大小  
`external`: 外部指针数组，它可能包含外部函数地址和全局变量指针。  
在函数的开始将所有可能是变参的寄存器入栈、分配返回值对象内存、将变参参数转成指针数组 pVA、最后 vm_ready 调用结束后从返回值对象中取出返回值并赋值给真实寄存器 x0。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_JRYX4E99XGYBRPS.webp)

#### int vm_ready(void *ret_val, void *pBytecode, int bcSize, void **external, void **pVA)

描述：构建或准备 bytecode 对应的 VMPState 对象，bytecode 映射一个 VMPState。  
`pRetVal`: 返回值  
`regCount`: 虚拟寄存器数量  
`pRegister`: 虚拟寄存器  
`typeCount`: 类型表数据  
`pTypeList`: 类型表  
`insCount`: 虚拟机指令数量  
`pInstructons`: 虚拟机指令  
`pBranchs`: 分支表

> 解析器比较复杂，只介绍简单的 handler 分析过程，在分析时有五个非常重要的数据在需要时刻清楚放在什么位置：** 虚拟机 pc**、* _指令对象_ *、**上下文对象**、**类型表对象**、**分支表对象 (遇到分支指令时关注)**，不然在分析时非常容易迷茫。  
> ![](https://bbs.kanxue.com/upload/attach/202504/271698_BYMCUCEMTMBUYQS.webp)

**getVMPObject 函数**

获取全局虚拟机缓存对象，返回值是一个 std::list

**getVMPState**

根据入参 pBytecode 尝试从缓存中查找应对的 VMPState 对象，如果 VMPState 没有被缓存则跳到 start_build 解析 bytecode 生成 VMPState 对象，相关分析参考 [bytecode 首次解码](##bytecode%E9%A6%96%E6%AC%A1%E8%A7%A3%E7%A0%81)，否则开始创建上下文对象。

**创建一个上下文对象**  
![](https://bbs.kanxue.com/upload/attach/202504/271698_X2C6WTS46BKFAKK.webp)  
**初始化寄存器**

将所有寄存器填充为 0  
![](https://bbs.kanxue.com/upload/attach/202504/271698_FZK9E3JYYPH8PYZ.webp)

**复制寄存器数据**

在首次解码构建时有一部分的寄存器会被赋上初值，寄存器中的初值是加密前原生汇编代码中的常量是不能被修改的，因此需要复制寄存器对象。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_UVUWVMY9SJ4TJSA.webp)

**复制入参到虚拟机寄存器**

如果不存在参数则准备执行 vm_interpreter，否则将传入的变参依次将参数复制到虚拟寄存器 V0,V1,V2,...。

> 数据结构 VMPState 对应着加密前的函数信息，其中包括参数数量、参数的类型和返回值

**准备执行**

在进入解释器前会把 VMPState 的数据结构成员作为参数传入后，准备执行虚拟机解释器了。

![](https://bbs.kanxue.com/upload/attach/202504/271698_44P4MK86XAE8XGE.webp)

#### int vm_interpreter()

`void *pRetVal`: 返回值

`int regCount`: 虚拟寄存器数量

`void *pRegister`: 虚拟机寄存器

`int typeCount`: 类型表数量

`void **pTypeList`: 类型表

`int insCount`: 虚拟机指令数量

`int16_t **pInstructons`: 虚拟机指令

`int16_t **pBranchs:` 分支表

**首次读取主操作码**

在进入解释器后这里只执行一次，由于**指令分发基本块**被分别复制到 13 个 handler 尾部原因，因此每个 handler 都有自己独立的主操作码分发器，在 handler 执行完成后会接着读取下一条虚拟机指令的主操作码然后进入下一个 handler，具体原因可以参数上面的章节[指令分发基本块复制](###%E6%8C%87%E4%BB%A4%E5%88%86%E5%8F%91%E5%9F%BA%E6%9C%AC%E5%9D%97%E5%A4%8D%E5%88%B6)。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_WRQE8U5QTVHBHZS.webp)

分发指令时关注对象：

`虚拟pc`: **0x0**

`指令对象`：**X24**

读取主操作数时需要关注是指针对象和 pc，这里刚刚开始执行 pc 为 0，只要关注指令对象在 x24 寄存器就行了，值得要注意的是不是的主操作码分发器中指令对象有可能在其他真实寄存器中。

> 13A5C0 MOV W19, #1  
> 在指令中读取了相对偏移 0 的主操作码后，通常这个指令相对偏移 0 用不上了，因此会有一个指针指向当前指令字节码的下一个字节也就是偏移 1，在很多的主操作码分发器中经常就是这样做的，因此在进入 handler 后看到的都是从偏移 1 开始读取指令中的字节码。提示：在**大多数分发器中**使用 W19 来指向偏移 1 的。

**主操作码 switch 分发表**

switch 表中包含了 13 个 handler，图中未命名暗蓝色 loc 开头的是填充虚假无用的地址并非真实 handler 起到一定的迷惑作用。

![](https://bbs.kanxue.com/upload/attach/202504/271698_7PVKW4MC5YE57WE.webp)  
![](https://bbs.kanxue.com/upload/attach/202504/271698_KC2HHPTGZGHCZSW.webp)

透过下面一张图可以了解虚拟机所有指令集和第二操作码的情况，有第二操作码意味着会有再次 switch 子分发的情况。

![](https://bbs.kanxue.com/upload/attach/202504/271698_YBGUMKK63MCAQRZ.webp)

### 指令集分析

在此只介绍一些简单的指令分析带领大家入门，复杂的指令会让文章变得更加杂乱无章。

#### MOV 指令

> MOV 指令格式 (6): [opcode, op2, dtype, stype, sreg, dreg]  
> opcode: 主操作码  
> op2: 第二操作码  
> dtype: 目标操作数类型  
> stype: 源操作数类型  
> sreg: 源寄存器  
> dreg: 目标寄存器

`汇编语法`：

> MOV dreg, sreg

有时 MOV 可能并不是直接将原寄存器中的数据传到目标寄存器，可能还有会其他操作：零扩展、带符号扩展、截断等等，这是什么还有 op2 第二个操作码的原因。

> 提示：IDA PRO 中的注释 op 和 pBytecode 是同一个指针变量两者等价，原因是我懒得一个一个的去回改注释了。

> 按重要程度依次分别是：**虚拟机 pc**、**指令对象**、**上下文对象**、**类型表对象**、**分支表对象** (遇到分支指令时关注)

`虚拟机pc(当前指令偏移op/pInsn)`: 当前指令 MOV 来源于 13 个 handler 其中之一尾部的主操作码分发，因此 pc 指针的当前指令偏移 0 已经不再使用，W19 已经在主分发器中修正为当前指令偏移 1 的位置，只要关注 W19 即可。  
`指令对象(pReg)`: 真实寄存器 x24 指向它。

`前驱节点`： w25=0。

x24 指向 pInstructions 起始位置指针，w19 指向当前指令字节码中的偏移 1。  
**地址 0x139400**: 取出 op2 第二个操作码到 w8 并判断 op2 是否合法。

![](https://bbs.kanxue.com/upload/attach/202504/271698_8W99XNAJ6RJ3BSP.webp)

> ADD W9, W19, #1  
> ADD W10, W19, #2  
> ADD W11, W19, #3  
> ADD W12, W19, #4  
> 计算出当前指令字节的偏移，然后分别将读取偏移 2、3、4、5 的字节数据。
> 
> w19 = 当前指令偏移 op
> 
> x24 = 指令对象：它是一个 16 位的数组使用 w19 计算好的偏移索引来访问  
> x21 = 类型对象 (pType)：同样它是一个指针数组，同样使用索引来访问  
> x28 = 寄存器对象 (pReg)：它也是一个数组，不同的 VMPState 元素数量不同，使用索引来访问，元素是 Register 内存大小 0x18，使用 base+index*0x18

![](https://bbs.kanxue.com/upload/attach/202504/271698_293QC8EUTUK2KMZ.webp)

当指令执行到 **0x13C49C** 位置时开始分发 op2，此时的数据状态如下：  
`目标操作数类型`: x2=pType[op[2]]，从当前指令偏移 2 读取索引值，然后使用索引获取类型指针  
`源操作数类型`: x8=pType[op[3]]，从当前指令偏移 3 读取索引值，使用索引获取类型指针  
`源寄存器`: x26=pReg[op[4]]，从当前指令偏移 4 读取索引值，再从使用索引获取寄存器指针  
`目标寄存器`: x20=pReg[op[5]]，从当前指令偏移 5 读取索引值，再从使用索引获取寄存器指针

> 0x13C48C ADD W22, W19, #5  
> `下一个pc`: 此时 W19 是指向偏移 1 的位置，加上 5 后 W22 指向下一条指令起始位置即偏移 0，由此得知当前指令长度是 6 个字节。

取出源寄存器的值并存放到目标寄存器

![](https://bbs.kanxue.com/upload/attach/202504/271698_SWJ4GQPBJK3ATW3.webp)

> 0x13C4BC: 是 MOV handler 尾部的添加的主操作码分发器。

MOV handler 主操作码分发器:

x24 是指向指令对象的，使用索引 w22(new pc) 获取下一条指令的主操作码。

此时 W19 指向上一指令偏移 1 的位置并且上一条指令的长度是 6 个字节，在执行 W19, W19, #6 之后 W19 正好指向下一条指令偏移 1 的位置。注：加上长度是 6 个字节，是因为 此分析器是 MOV 专用的只有 MOV 指令执行完成后才可到达此位置。

![](https://bbs.kanxue.com/upload/attach/202504/271698_UE948MTZ679ADJS.webp)

#### CMP 指令

> CMP 指令格式 (6): [opcode, type, sreg1, sreg2, creg, op2]  
> opcode: 主操作码  
> type: reg1, reg2 的操作数类型  
> sreg1: 源寄存器 1  
> sreg2: 源寄存器 2  
> creg: 目标条件码寄存器  
> op2: 第二操作码  
> 指令说明: 通常 cmp 和 jcc、cmp 和 csel 成对出现

`汇编语法`：

> CMP.

在分析前再次提示一下重要的对象，handler 分析是主要围绕这五个对象的数据展开分析的。

> 按重要程度依次分别是：虚拟机 pc、指令对象、上下文对象、类型表对象、分支表对象 (遇到分支指令时关注)

`虚拟机pc`: 当前 CMP 指令来源于 13 个 handler 其中之一尾部的主操作码分发，此时已经完成分发到当前指令，pc 偏移 0 已经不再使用，在主分发器中 W19 已经修正为当前指令偏移 1 的位置，只要关注 W19 即可。  
`指令对象`: 真实寄存器 x24 指向它。

W19 指向当前指令偏移 1 的位置，加上 4 后 W8 指向偏移 5，根据上面的指令格式偏移 5 是读取 op2。  
**ADD W8, W19, #4**  
**LDR W8, [X24,W8,UXTW#2]**  
备份 W25 到 W26 因为后面要使用 W25 寄存器了，这里的 W25 是指向前驱基本块，在后面执行 PHI 时会用到前驱基本块数据。  
**MOV W26, W25**  
检查 op2 是否合法  
**CMP W8, #0x28**

![](https://bbs.kanxue.com/upload/attach/202504/271698_5FVT83TG38JSCQP.webp)

准备分发第二个操作数，第二个操作码就是比较条件的种类：EQ、NE、GT、GE、LT、LE 等等。  
`上下文对象`: X28

`类型表对象`: X21

获取操作数类型 **type**:  
**LDR W23, [X24,W19,UXTW#2]**  
**LDR X25, [X21,X23,LSL#3]****  
获取第一个源寄存器 **sreg1** 指针，pRegsBase+Index+0x18(寄存器元素长度):  
**ADD W9, W19, #1**  
**MOV W12, #0x18**  
__LDR W9, [X24,W9,UXTW#2] *_  
*_MADD X22, X9, X12, X28*_  
获取第二个源寄存器_ _sreg2* _指针  
*_ADD W10, W19, #2*_  
*_LDR W10, [X24,W10,UXTW#2]*___  
**MADD X9, X11, X12, X28**  
取目标操作数 **creg** 条件码寄存器指针  
**ADD W11, W19, #3**  
**LDR W11, [X24,W11,UXTW#2]**  
**MADD X9, X11, X12, X28**

![](https://bbs.kanxue.com/upload/attach/202504/271698_ZU2HCQS8WFYJC32.webp)

当指令执行到地址`0x13BB70`，指令中的操作数已经全部取出，类型对象、指令对象已经不再使用，只需关心 type、sreg1、sreg2、dreg 所在寄存器即可：  
type: X25  
sreg1: X22  
seg2: X27  
creg: X9

第二个操作数是 CMP 条件码的种类，条件码中还包括浮点数的比较，在此以常用的条件码 CMP.EQ 进行分析。

![](https://bbs.kanxue.com/upload/attach/202504/271698_USFG96X74VU7673.webp)

首先获取比较操作数的大小，再从寄存器指针 sreg1 获取操作数大小的值，以 4 个字节大小为例：

![](https://bbs.kanxue.com/upload/attach/202504/271698_PEEUFMRYTAZ44UH.webp)

取 sreg1 的值：  
此时发现使用的汇编指令是 LDRSW，的确是取出 4 个字节的数据。

`sreg1_val`: X20

![](https://bbs.kanxue.com/upload/attach/202504/271698_FBHPH6X723DCNPS.webp)

取 sreg2 操作数大小和寄存器的值:  
`sreg2_val: X8

![](https://bbs.kanxue.com/upload/attach/202504/271698_SPW4A5H2S87D77Y.webp)  
![](https://bbs.kanxue.com/upload/attach/202504/271698_YYBJJEW77FMJ6SN.webp)

比较 sreg1 和 sreg2 获取 EQ 条件码  
![](https://bbs.kanxue.com/upload/attach/202504/271698_M57XZX4MTN6WKCH.webp)

更新下一条指令 pc 并检查是否超过 macPC，w19 指向当前指令偏移 1 的位置，**ADD W8, W19, #5** 并指向下一个指令的偏移 0 位置，表示当前指令 CMP 长度是 6。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_646U3HPCA4S9BTX.webp)

CMP 尾部主操作码分发器：  
依旧是读取主操作码、更新 w19 指针到偏移 1 位置，最后 **BR X8** 进入下一个 handler。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_FUKTUVNWY67MDX2.webp)

#### LDR 指令

> LDR 指令 (4): [opcode, type, mreg, dreg]  
> opcode: 主操作码  
> type: 操作数类型  
> mreg: 源寄存器内存, 没有偏移量立即数  
> dreg: 目标寄存器  
> 指令说明: LDR dreg, [mreg], LDR 指令是 mem--->reg 到目标寄存器，即寄存器内存的值到寄存器中

`汇编语法`：

> LDR dreg, [mreg]

> 关键对象：虚拟机 pc、指令对象、上下文对象、类型表对象、分支表对象 (遇到分支指令时关注)  
> 虚拟机 pc: W19  
> 指令对象: X24  
> 上下文对象: X28  
> 类型表对象: X21

获取操作数类型`type`:

> 0x13A1AC LDR W10, [X24,W19,UXTW#2]  
> 0x13A1C0 LDR X1, [X21,X10,LSL#3]

获取目标操作数寄存器`dreg`：

> 0x13A1B0 ADD W8, W19, #1  
> 0x13A1B8 LDR W8, [X24,W8,UXTW#2]  
> 0x13A1C4 MOV W10, #0x18  
> 0x13A1C8 MADD X2, X8, X10, X28

![](https://bbs.kanxue.com/upload/attach/202504/271698_MV289EXV3G4E2KA.webp)

**LDR_With_Type 函数分析**：

函数原型：void LDR_With_Type(void *dreg, void *type, void *mReg);

参数：x0=dreg, x1=type, x2=mreg  
判断操作数类型是否 char _指针类型，如果是 char_ 指针则拷贝指针，否则取出 mreg 地址中的值赋值给 dreg。  
获取类型长度：

> 0x13FF60 BLR X8

类型长度不能大于 8 个字节:

> 0x13FF68 CMP W8, #7

获取 mreg 中的指针:

> LDR X8, [X20]

依据类型长度 1/2/4/8 字节，然后再次从获取指针中的值：

> 0x13FF88 LDRSB W8, [X8]  
> 0x13FFA4 LDRSH W8, [X8]  
> 0x13FFB0 LDR W8, [X8]  
> 0x13FF94 LDR X8, [X8]

最后将值赋值给 dreg：

> 0x13FF98 STR X8, [X19]

或

> 0x13FFB4 STR W8, [X19]

![](https://bbs.kanxue.com/upload/attach/202504/271698_GSHN5J24ZCDTSYY.webp)

bytecode 首次解码
-------------

> 解码的目的就是将 bytecode 反序列到 VMPState 对象。

### VBR 编码

bytecode 使用 Variable Bitrate（VBR）编码格式，这个编码很多领域都在使用例如音频和视频，有兴趣的也可以 AI 了解。  
虚拟机字节码采用了 6 位的编码格式，与 protobuf 的 Varints 编码格式有些类似，只不过 protobuf 使用了 8 字节，这里使用了 6 个字节。  
protobuf Varints 编码相关链接：[5.1、Varints 编码 (变⻓的类型才使⽤)](elink@5bcK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0L8r3!0#2k6q4)9J5k6i4c8W2L8X3y4W2L8Y4c8Q4x3X3g2U0L8$3#2Q4x3V1k6V1k6i4k6W2L8r3!0H3k6i4u0Q4x3V1k6S2M7Y4c8A6j5$3I4W2i4K6u0r3x3U0b7$3x3K6j5#2x3b7`.`.)  
虚拟机 6 位的 VBR 编码：  
数字 5 的二进制 6 位编码是这样的，以 0 位开始最高位 5 是 0 说明没有后续字节位数据，解码后的数值就是 5

> 000101

前 6 位最高位是 1，代表还有后续的字节位数据。

> 100101 001111

在解码组合位数据时先取 0-4 位有效数据低 5 位: 00101，第 5 位为 1 说明还有后续的字节位数据，然后再取下一个 6 位字节数 001111，第 5 位为 0 说明没有后续的字节位数据了，有效数据位是低 5 位：01111，解码组合后的数值为 0x1E5(0b111100101)。注意：后 6 位的在组合时要放在高位。

> 01111 00101 ---> 111100101  
> 如果后位最高位为 1 就再取六位数据，如此反复直到最高位为 0 为止。  
> 使用脚本解码 VBA，默认位数 6bit

脚本实现 decode：

```
class BytecodeDecoder():
    def __init__(self, bytecode: bytes, extern_address: list, filename):
        self.extern_address = extern_address
        self.bytecode_bits = bitarray(endian='little')
        self.bytecode_bits.frombytes(bytecode)
        self.bytecode = bytecode
        self.bit_index = 0
        self.filename = filename
 
    def decode(self, nBit=6):
        num = 0
        index = 0
        bit_num = 0
        exit = False
        while index + nBit <= len(self.bytecode_bits):
            # 读取 nBit 位
            chunk = self.bytecode_bits[self.bit_index:self.bit_index + nBit]
            high_bit = chunk[-1]  # 最高位
            low_bits = ba2int(chunk[:-1])  # 低 5 位转换为整数
            if high_bit == 1:
                num = num | (low_bits << bit_num)  # 左移 5 位并按位或合并
            else:
                num = num | (low_bits << bit_num)
                exit = True
            self.bit_index += nBit  # 移动索引
            if exit:
                return num
            bit_num += 5
        raise Exception("bytecode decode error")
```

### bytecode 解码流程

![](https://bbs.kanxue.com/upload/attach/202504/271698_26TYM48W995J5D6.webp)

#### bytecode 解码流程书签

在 IDA PRO 使用 ctrl+m 打开收藏的书签，没有快捷键的打开菜单 View->Open subviews->bookmarks 打开，[step

**读取第一个字节码**

字节码是 64 位长度为单位进行处理的，而 VBR 是以 6 个字节单位处理数据，当前 64 位中有效字节码数据不足 6 位时，会读取下一个 64 位数据并从低位开始取字节位，将不足的有效位补到当前字节位直到满 6 位为止。在解码的同时还在堆栈中维护了表示当前 VBR 解码状态的数据，分别是指向字节码的指针：pBytecode，当前剩余的 64 数据：remain_bytecode，remain_bytecode 的剩余位数： remain_bit。  
从 bytecode 读取 64 位的数据：  
![](https://bbs.kanxue.com/upload/attach/202504/271698_PSCAV74WEBJ3MZE.webp)

1.  计算出寄存器数量  
    bytecode 最开始位置保存了寄存器数量，读取数量并创建上下文对象。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_WUKHSFG8DSX8EXS.webp)  
    初始化并填充数据  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_HUWWY4AV7ZW2UFJ.webp)  
    解码脚本：
    
    ```
    def decode_register_count(self):
      regs_count = self.decode()
      return regs_count
    ```
    
2.  解码需要初始化寄存器
    
    *   初始化寄存器的数量  
        解码出需要初始化寄存器的数量，然后申请了一个堆内存用于存放寄存器索引。  
        ![](https://bbs.kanxue.com/upload/attach/202504/271698_F9XJ23DY84P2DD7.webp)
        
    *   解码需要初始化寄存器表  
        在解码完寄存器数量后，紧接着字节流后面数据是需要初始化的寄存器表。  
        ![](https://bbs.kanxue.com/upload/attach/202504/271698_DSV9EE9CMTD589D.webp)
        

解码脚本：

```
def decode_register_initial_value(self):
  regs_initial_count = self.decode()
  regs_initial = []
  for i in range(0, regs_initial_count):
    reg_num = self.decode()
    # init_regs_num.append((insn, f'{insn:#x}'))
    regs_initial.append(reg_num)
    return regs_initial
```

1.  设置外部地址列表到寄存器  
    这里的外部地址的寄存器索引数据与第 2 步的初始寄存器数据不共用。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_H837659SXV2J5B9.webp)  
    解码脚本：
    
    ```
    def set_registers_extern_address(self,
                                     registers: list[Register],
                                     extern_address: list[int]):
      extern: ExternInstructions = ExternInstructions()
      addr_list_count = self.decode()
      for i in range(0, addr_list_count):
        reg_idx = self.decode()  # 初始化的寄存器索引
        addr_list_idx = self.decode()  # 获取地址列表索引
        addr = self.read_extern_address(extern_address, addr_list_idx)
        registers[reg_idx].value = addr
        extern.targetRegs.append(reg_idx)
        extern.externalAddress.append(addr)
        # print(
        #     f"extern register[{reg_idx}] = {addr:#x}")
        return extern
    ```
    
2.  解码类型对象表  
    虚拟机解释执行时所需的类型数据均来自此表。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_DSK8X48WZ4AW3RT.webp)  
    解码脚本：
    
    ```
    def decode_types(self):
            # 4.解码类型表
            types_count = self.decode()
            types = [None] * types_count
            for i in range(0, types_count):
                type = self.decode()
                # print(f'type[{i}]: {type:#x}')
                match(type):
                    case 0x3 | 0x10 | 0x12:
                        raise Exception("error")
                    case 0x5 | 0xc | 0x13:
                        struct_0xC = 0
                        struct_0xD = self.decode()
                        struct_0xE = 1
                        member_count = self.decode()
                        # 4.1设置结构体成员类型
                        members = []
                        for j in range(0, member_count):
                            member_type_index = self.decode()
                            t_membetr = types[member_type_index]
                            if t_membetr is None:
                                types[member_type_index] = t_membetr = StructType(0, [], "")
                            members.append(t_membetr)
                        # 4.2 获取结构体类型名
                        type_name = []
                        type_name_size = self.decode()
                        for j in range(0, type_name_size):
                            c = self.decode()
                            type_name.append(c)
     
                        name = "".join(chr(c) for c in type_name)
                        struct_type = StructType(member_count, members, name)
                        struct_type.init()
                        types[i] = struct_type
                    case 0x1:
                        types[i] = VMPType()
                    case 0x6:
                        nbit = self.decode()
                        types[i] = IntegerType(nbit)
                    case 0x9:
                        element_count = self.decode()
                        element_type_idx = self.decode()
                        element_type = types[element_type_idx]
                        if element_type is None:
                            # 创建数组元素为结构类型...
                            raise Exception("error")
                        types[i] = ArrayType(element_count, element_type)
                    case 0x7:
                        ptr_type_index = self.decode()
                        ptr_type = types[ptr_type_index]
                        if ptr_type is None:
                            ptr_type = StructType(0, [], "")
                            types[ptr_type_index] = ptr_type
                        types[i] = PointerType(ptr_type)
                    case 0xb:
                        types[i] = FloatType()
                    case 0x14:
                        flag = self.decode()
                        return_value_type = self.decode()
                        argument_count = self.decode()
     
                        return_value = types[return_value_type]
                        arguments = []
                        if return_value is None:
                            return_value = None
                            raise Exception("返回值类型 error")
                        for j in range(0, argument_count):
                            arg_type_idx = self.decode()
                            arg_type = types[arg_type_idx]
                            if arg_type is not None:
                                arguments.append(arg_type)
                            else:
                                # 构建结构类型
                                raise Exception("error")
     
                        types[i] = FunctionType(
                            return_value, argument_count, arguments, flag)
                    case 0x15:
                        types[i] = DoubleType()
                    case _:
                        input(f"未知的类型:{type:#x}\n")
            return types
    ```
    
3.  为寄存器设置初值  
    这里的寄存器列表来源第 2 步解码的数据，这里不仅会为寄存器设置初始值，还会有其他指令的操作，目标寄存器是一个 “静态或只读的”，解释器的执行不会改目标寄存的值。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_A8K9RPXVPXSP2EF.webp)  
    解码脚本：
    
    ```
    def set_registers_inial_value(self, registers: list[Register], types: list[int],
                                regs_initial: list[int]):
        init_instructions: InialInstructions = InialInstructions()
        count = self.decode()
        for i in range(0, count):
            init_type = self.decode()           
            reg_idx = regs_initial[i]
            # print(f'init reg type: {init_type:#x}')
            # 记录初始化指令
            init_instructions.opcodes.append(init_type)
            init_instructions.dRegs.append(reg_idx)           
            match(init_type):   # ADD.MO
                case 0 | 0xb | 0x17:
                    type_idx = self.decode()
                    deep = self.decode()
                    breg = self.decode()
                    member_offset_table = []
                    for j in range(1, deep):
                        member_offset_table.append(self.decode())
                    # 记录初始化指令
                    init_instructions.imm.append(member_offset_table)
                    init_instructions.type.append(type_idx)
                    init_instructions.sRegs.append(breg)
                case 7: # MOV REG, IMM
                    imm = self.decode()
                    registers[reg_idx].value = imm
                    # 记录初始化指令
                    init_instructions.imm.append(imm)
                    init_instructions.type.append(None)
                    init_instructions.sRegs.append(None)
                    # print(f'init register[{reg_idx}] = {imm:#x}')
                case 8:# MOV REG, ???
                    type_index = self.decode()
                    t = types[type_index]
                    if t.tag != 0xe:
                        registers[reg_idx].value = 0
                    else:
                        raise Exception("error")
                    # 记录初始化指令
                    init_instructions.type.append(type_index)
                    init_instructions.imm.append(None)
                    init_instructions.sRegs.append(None)
                case 0x15:  # MOV REG.T, REG@FuncPtr
                    while self.decode() & 0x20:
                        pass
                    type_idx = self.decode()
                    t = types[type_idx]
                    sreg = self.decode()
                    registers[reg_idx].value = registers[sreg].value
                    registers[reg_idx].type = t
                    # 记录初始化指令
                    init_instructions.type.append(type_idx)
                    init_instructions.imm.append(None)
                    init_instructions.sRegs.append(sreg)
                case _:
                    input(f"未知的初始化寄存器类型:{init_type:#x}\n")
        return init_instructions
    ```
    
4.  获取入口函数对象  
    从第 4 步类型对象表获取入口函数类型信息，这里的入口函数是指原生代码未虚拟化前的函数信息。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_MZHU2G2GMFPC9GF.webp)  
    解码脚本：
    
    ```
    def get_function_type(self, types: list[int]):
      func_type_idx = self.decode()
      return types[func_type_idx].type
    ```
    
5.  解码虚拟机指令  
    虚拟机机解释器执行的指令。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_AH7FMT25QXF87PH.webp)  
    解码脚本：
    
    ```
    def decode_bytecode(self):
      vm_insn_count = self.decode()
      vm_instructions = []
      for i in range(0, vm_insn_count):
        insn = self.decode()
        vm_instructions.append(insn)
        return vm_instructions
    ```
    
6.  创建分支表  
    分支表是跳转的目标地址，BSEL.PHI、J、JCC、SWITCH 指令等会从此表中获取目标地址。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_3XSAKSC79QF7JXM.webp)  
    解码脚本：
    
    ```
    def decode_branches(self):
      branches_count = self.decode()
      branches = []
      for i in range(0, branches_count):
        offset = self.decode()
        branches.append(offset)
        return branches
    ```
    

**创建 VMPState 对象**

在解码完 bytecode 后，入口函数类型、虚拟寄存器数量、类型表数量、指令数量、上下文对象、类型表对象、指令表对象、分支表关键数据已经获取，接下来该创建 VMPState 对象了。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_K9WD7Q9Q9WPB4FQ.webp)  
并将解码完成后的数据放入此对象并将 VMPState 添加到全局缓存对象，防止下次执行此 bytecode 时重复构建。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_CWHT54X42V2RM34.webp)

### 虚拟机数据结构

在逆向分析时和虚拟机执行时这个数据结构十分重要了解好这份数据分析不迷路，尤其是解释执行时经常要获这些数据结构：指令、类型、寄存器、分支等。

#### 虚拟机状态对象

```
struct VMPState {
  0x0:  Type* entryFunction;
  0x8:  int regCount;
  0xC:  int typeCount;
  0x10: int insCount;
  0x18  Context* context;
  0x20: Type** pTypeList;
  0x28: int16_t** pInstructions;
  0x30: Branches* pBranches;
}
```

#### 类型对象

类型对象用于描述类型数据，并不会包含类型的值。

*   类型签标
    
    ```
    enum TypeTag {
      VoidType = 0,
      FloatType = 2,
      DoubleType = 3,
      InteterType = 0xb,
      FunctionType = 0xc,
      StructType = 0xd,
      ArrayType = 0xe,
      PointerType = 0xf,
      VertorType = 0x10
    }
    ```
    
*   所有类型的基类  
    `内存长度`: 0x10
    
    ```
    struct Type {
    +0x0:    void* vtable;
    +0x8:    TypeTag tag;
    +0xc:    union {
                  int nbit;    // IntegerType时使用
                                  bool field1;                               
              }
    +0xd:    bool isInit;  // StructType使用，是否计算结构体内存长度
    +0xe:    bool field3;
    }
    ```
    
*   空类型  
    `内存长度`: 0x10
    
    ```
    struct VoidType : public Type {
    }
    ```
    
*   浮点类型  
    `内存长度`: 0x10  
    使用 tag 来区别数据类型，float 长度 32 位，double 长度 64 位。
    
    ```
    struct FloatType : public Type {
    }
    ```
    
    ```
    struct DoubleType : public Type {
    }
    ```
    
*   整形  
    `内存长度`: 0x10  
    `nbit`成员用于描述 1 位 / 8 位 / 16 位 / 32 位 / 64 位。
    
    ```
    struct IntegerType : public Type {
    }
    ```
    
*   函数类型  
    `内存长度`: 0x28
    
    ```
    struct FunctionType : public Type {
    +0x10:  Type*   returnValueType;
    +0x18:  int.       argumentCount;
    +0x20: type**   arguments;
    }
    ```
    
*   结构体类型  
    `内存长度`: 0x38
    
    ```
    struct StructType : public Type {
      0x10: int32   memberCount;
      0x18: type**  members;
      0x20: int32   memorySize;
      0x28: int32*  memberOffsetTable;
      0x30: char*  typeName;
    }
    ```
    
*   数组类型  
    `内存长度`: 0x18
    
    ```
    struct ArrayType{
      0xc: int32   count;
      0x10: Type*  element;
    }
    ```
    
*   指针类型  
    `内存长度`: 0x18
    
    ```
    struct PointerType{
      0x10: Type*  pointee;
    }
    ```
    
    #### 上下文对象
    
    `内存长度`: 0x18
    
    ```
      struct Register {
            0x0: int64 value;
            0x8: Type* t;
        0x10: bool isBuffer; // 指示value是否内存管理函数分配: cmalloc/malloc
    }
    ```
    
    `内存长度`: 内存长度不确定，依据 count 的数值决定大小, size=count * sizeof(Register)
    
    ```
    struct Context {
          0x0: int count;
        0x8: Register regs[];
    }
    ```
    
    #### 分支表
    
    `内存长度`: 内存长度不确定，首次解码有长度数据。
    
    ```
    struct Branchs {
    0x0: int16_t br[];
    }
    ```
    
    #### 指令对象
    
    `内存长度`: 内存长度不确定，VMPState->insCount 描述了指令数量。
    
    ```
    struct Instructions {
    0x0: int16_t ins[];
    }
    ```
    

指令解码格式
------

> 注：第一次解码出 VMPState 核心数据结构，第二次是虚拟执行时的指令解码

最外层的 [] 相当于 python 的列表，里层的 [] 表示数据是可选的，()通常是成对数据。

<table><thead><tr><th>指令名称</th><th>opcode</th><th>长度</th><th>格式</th><th>说明</th></tr></thead><tbody><tr><td>ARITH</td><td>0x1</td><td>6</td><td>[opcode, op2, type, sreg1, sreg2, dreg]</td><td></td></tr><tr><td>MOV</td><td>0x2</td><td>6</td><td>[opcode, op2, dtype, stype, sreg, dreg]</td><td></td></tr><tr><td>CALLOC</td><td>0x6</td><td>5</td><td>[opcode, a2_type, a1_type, a1_sreg, dreg]</td><td></td></tr><tr><td>STR</td><td>0xa</td><td>4</td><td>[opcode, stype, sreg, mreg]</td><td></td></tr><tr><td>CMP</td><td>0xc</td><td>6</td><td>[opcode, type, sreg1, sreg2, creg, op2]</td><td></td></tr><tr><td>PHI</td><td>0xe</td><td>变长 (3+)</td><td>[opcode, dreg, count, [(reg, branch), (reg, branch), (...)]]</td><td></td></tr><tr><td>CALL</td><td>0xf</td><td>变长 (5+)</td><td>[opcode, ftype, [count], tag, [retreg], ereg, none, [areg, ...]]</td><td></td></tr><tr><td>RET</td><td>0x13</td><td>变长 (2+)</td><td>[opcode, type, [retreg]]</td><td></td></tr><tr><td>SWITCH</td><td>0x16</td><td>变长 (5+)</td><td>[opcode, treg, type, defbranch, count, [(casereg, branch), ...]]</td><td></td></tr><tr><td>J/JCC</td><td>0x1d</td><td>3 或 6</td><td>[opcode, brtype, tbranch,[creg, ctype, fbranch]]</td><td></td></tr><tr><td>CSEC</td><td>0x1e</td><td>7</td><td>[opcode, ntype, creg, type, treg, freg, dreg]</td><td></td></tr><tr><td>GEP</td><td>0x28</td><td>变长 (5+)</td><td>[opcode, dreg, none, type, count, breg, [index, ...]]</td><td></td></tr><tr><td>LDR</td><td>0x2a</td><td>4</td><td>[opcode, type, mreg, dreg]</td><td></td></tr></tbody></table>

还原篇
===

虚拟机还原一共分两篇，还原到汇编大多数情况只要能够理解其中逻辑就已经足够了，还原到原生代码是将还原做到极致。直接还原汇编是最接近虚拟机指令的语言是最容易的也是最不容易出错的，同样它也是虚拟机指令的映射。

还原到汇编
-----

解码的后的字节码还是一堆数据，为了方便理解数据需要将数据转成汇编代码方便阅读，对于虚拟机指令和原生架构相似度比较高的可以借用原生架构的汇编代码还原，对于一些虚拟机有自己指令集的，通常需要结合虚拟机指令集的特点自定义了一套与虚拟机语意相近的汇编语言，只要将字节码转成语义相近的汇编语言即可，这里的汇编语言参考了 mips、risc-v、smali、Binary Ninja 中间语言等一些语法。

*   操作数长度  
    通用寄存器和浮点寄存数量不固定
    
    <table><thead><tr><th>寄存器</th><th>1 位</th><th>8 位</th><th>16 位</th><th>32 位</th><th>64 位</th></tr></thead><tbody><tr><td>通用寄存器</td><td>V0.b</td><td>V1.b</td><td>V2.h</td><td>V3.d</td><td>V4</td></tr><tr><td>浮点寄存器</td><td></td><td></td><td>H0(半精度)</td><td>S0(单精度)</td><td>D0(双精度)</td></tr></tbody></table>
*   操作数标识符说明
    
    <table><thead><tr><th>标识</th><th>说明</th><th>备注</th></tr></thead><tbody><tr><td>op</td><td>主要操作码</td><td></td></tr><tr><td>op2</td><td>第二个操作码</td><td></td></tr><tr><td>dreg</td><td>目标寄存器</td><td></td></tr><tr><td>sreg</td><td>源寄存器</td><td>只有一个源寄存器时的命名</td></tr><tr><td>sreg1</td><td>第一个源寄存器</td><td></td></tr><tr><td>sreg2</td><td>第二个源寄存器</td><td></td></tr><tr><td>type</td><td>操作数的类型</td><td>表示操作数的长度和数据类型</td></tr><tr><td>stype</td><td>源操作数类型</td><td></td></tr><tr><td>dtype</td><td>目标操作类型</td><td></td></tr><tr><td>a1_type</td><td>第一个参数类型</td><td>只适用于 ALLOC 指令</td></tr><tr><td>a2_type</td><td>第二个参数类型</td><td>只适用于 ALLOC 指令</td></tr><tr><td>a1_sreg</td><td>第一个参数的寄存器</td><td>只适用于 ALLOC 指令</td></tr><tr><td>mreg</td><td>内存操作数的寄存器</td><td>内存访问指令 STR/LDR</td></tr><tr><td>creg</td><td>条件码寄存器</td><td>CMP 指令</td></tr><tr><td>count</td><td>元素数量</td><td>PHI 指令 [val, lab] 的数量、GEP 表示 [index] 的数量、SWITCH 表示 case 的元素数量</td></tr><tr><td>branch</td><td>分支</td><td>表未基本块的起始标签</td></tr><tr><td>ftype</td><td>函数类型</td><td>只适用于 CALL 指令, 类型中描述了函数的返回值、参数数量、参数类型</td></tr><tr><td>retreg</td><td>返回值寄存器</td><td>只适用于 CALL 指令, 保存函数的返回值的寄存器</td></tr><tr><td>ereg</td><td>调用目标寄存器</td><td>只适用于 CALL 指令, 保存了需要调用的目标地址。例如 call 0x1234、call reg</td></tr><tr><td>areg</td><td>参数寄存器</td><td>只适用于 CALL 指令</td></tr><tr><td>treg</td><td>用于比较的目标寄存器</td><td>只适用于 SWITCH 指令</td></tr><tr><td>defbranch</td><td>默认分支基本块标签</td><td>只适用于 SWITCH 指令</td></tr><tr><td>casereg</td><td>SWITCH 的 case 常量</td><td>只适用于 SWITCH 指令, casereg 寄存器保存了 case 立即数</td></tr><tr><td>branch</td><td>SWITCH 的代码基本块标签</td><td>只适用于 SWITCH 指令</td></tr><tr><td>brtype</td><td>表示分支指令的类型</td><td>值 0 时无跳转指令，值 1 时有条件跳转</td></tr><tr><td>tbranch</td><td>真值分支</td><td>只用于 J/JCC 指令</td></tr><tr><td>fbranch</td><td>假值分支</td><td>只用于 J/JCC 指令</td></tr><tr><td>treg</td><td>真值寄存器</td><td></td></tr><tr><td>freg</td><td>假值寄存器</td><td>只用于 CSEL 指令</td></tr><tr><td>breg</td><td>基址寄存器</td><td>只用于 GEP 指令</td></tr><tr><td>index</td><td>元素索引寄存器</td><td>只用于 GEP 指令, index 保存了访问元素成员的值</td></tr></tbody></table>
*   指令集
    

1.  算术指令
    
    <table><thead><tr><th>助记符</th><th>语法</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>XOR</td><td>XOR V0, V1, V2</td><td>MOV dreg, sreg1, sreg2</td><td></td></tr><tr><td>SUB</td><td>SUB V0.b, V1.b, V2.b</td><td>MOV dreg, sreg1, sreg2</td><td></td></tr><tr><td>UDIV</td><td>UDIV V0.h, V1.h, V2.h</td><td>UDIV dreg, sreg1, sreg2</td><td></td></tr><tr><td>ADD</td><td>ADD V0.d, V1.d, V2.d</td><td>ADD dreg, sreg1, sreg2</td><td></td></tr><tr><td>OR</td><td>OR V0, V1, V2</td><td>OR dreg, sreg1, sreg2</td><td></td></tr><tr><td>SMOD</td><td>SMOD V0.b, V1.b, V2.b</td><td>SMOD dreg, sreg1, sreg2</td><td></td></tr><tr><td>SDIV</td><td>SDIV V0.h, V1.h, V2.h</td><td>SDIV dreg, sreg1, sreg2</td><td></td></tr><tr><td>UMOD</td><td>UMOD V0.d, V1.d, V2.d</td><td>UMOD dreg, sreg1, sreg2</td><td></td></tr><tr><td>ASR</td><td>ASR V0, V1, V2</td><td>ASR dreg, sreg1, sreg2</td><td></td></tr><tr><td>LSL</td><td>LSL V0.b, V1.b, V2.b</td><td>LSL dreg, sreg1, sreg2</td><td></td></tr></tbody></table>
2.  MOV 指令
    
    <table><thead><tr><th>助记符</th><th>语法</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>MOV</td><td>XOR V0, V1</td><td>MOV dreg, sreg</td><td>数据移动</td></tr><tr><td>MOV.T</td><td>MOV.T V0, V1.b</td><td>MOV.T dreg, sreg</td><td>将 V1 操作截断为 8 位移动到 V0</td></tr><tr><td>MOV.Z</td><td>MOV.Z V0, V1.d</td><td>MOV.Z dreg, sreg</td><td>将 32 位操作数 V1.d 零扩展到 64 位操作数 V0</td></tr><tr><td>MOV.S</td><td>MOV.S V0, V1.d</td><td>MOV.S dreg, sreg</td><td>将 32 位操作数 V1.d 带符号扩展到 64 位操作数 V0</td></tr></tbody></table>
3.  MOV 指令辅助操作
    
    <table><thead><tr><th>操作符</th><th>描述</th></tr></thead><tbody><tr><td>T</td><td>数据截断</td></tr><tr><td>Z</td><td>零拓展</td></tr><tr><td>S</td><td>符号拓展</td></tr></tbody></table>
4.  CALLOC 指令
    
    内存分配指令，向堆申请内存
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>CALLOC</td><td>CALLOC (0x1, 0x20), V21</td><td>CALLOC (num, size), RetVal</td><td>功能与 libc 中的 calloc 一致</td></tr></tbody></table>
5.  内存访问指令
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>STR</td><td>STR V0.b, [V3]</td><td>STR dreg, [mreg]</td><td>与 arm 指令相同</td></tr><tr><td>LDR</td><td>LDR V5, [V80]</td><td>LDR dreg, [mreg]</td><td>与 arm 指令相同</td></tr></tbody></table>
6.  比较指令
    
    FCMP 指令用到的很少暂不列入
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>CMP.EQ</td><td>CMP.EQ V3.b, V0, V3</td><td>CMP.EQ creg, sreg1, sreg2</td><td>比较并将结果 EQ 条件码存入 creg</td></tr><tr><td>CMP.NE</td><td>CMP.NE V3.b, V0, V3</td><td>CMP.NE creg, sreg1, sreg2</td><td>比较并将结果 NE 条件码存入 creg</td></tr><tr><td>CMP.CC</td><td>CMP.CC V3.b, V0, V3</td><td>CMP.CC creg, sreg1, sreg2</td><td>比较并将结果 CC 条件码存入 creg</td></tr><tr><td>CMP.LT</td><td>CMP.LT V3.b, V0, V3</td><td>CMP.LT creg, sreg1, sreg2</td><td>比较并将结果 LT 条件码存入 creg</td></tr></tbody></table>
7.  分支数据选择指令
    
    指令说明：BSEL 指令检查当前指令来自哪个前驱分支 BranchIndex, 将匹配到的分支中的 reg 赋值给 dreg, 通常它是循环的开始位置 (循环第一条指令) 类似 for 循环的 init 语句块和 inc 自增块。
    
    详情参考 llvm ir 中的 phi 指令。
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>BSEL.PHI</td><td>BSEL.PHI V8, [(V0, #0x3), ...]</td><td>BSEL.PHI dreg, [(reg, branch), ...]</td><td>() 是成对出现的, 一般二对数据｜</td></tr></tbody></table>
8.  调用子程序指令
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>CALL</td><td>CALL V3, [V0, V8, V21,...], V33</td><td>CALL dreg, [areg, areg,...], retreg</td><td>调用子程序, 参数和返回值是可选的</td></tr></tbody></table>
9.  分支指令
    
    基本块的后继指令
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>RET</td><td>RET V8</td><td>RET [retreg]</td><td>返回子程序, 返回值寄存器 retreg 是可选的</td></tr><tr><td>SWITCH</td><td>SWITCH V3, #0x1234, [(V2, #0x5678)]</td><td>SWITCH treg, #defbranch, [(reg, #branch), ...]</td><td>可以理解为 c 语言中的 switch 语句</td></tr><tr><td>J</td><td>J 0x1234</td><td>J address</td><td>无条件跳转指令</td></tr><tr><td>JCC</td><td>JCC&lt;V8.b&gt; 0x1234, 0x5678</td><td>JCC</td><td>有条件跳转, 当条件码等于 1 时为真</td></tr></tbody></table>
10.  条件选择指令
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>CSEL</td><td>CSEL&lt;V3.b&gt; V6, V7, V9</td><td>CSEL</td><td>arm64 中的 CSEL 指令类似</td></tr></tbody></table>
11.  取元素指针指令
    
    GEP 是 getelementptr 指令的缩写，详细可以 llvm ir 中的 getelementptr 指令
    
    <table><thead><tr><th>助记符</th><th>语法格式</th><th>操作数格式</th><th>说明</th></tr></thead><tbody><tr><td>GEP</td><td>GEP V2, V3, [V8, V9, V20]</td><td>GEP dreg, breg,[index, ...]</td><td></td></tr></tbody></table>

### 汇编

经过前面的自行设计的汇编代码和了解[指令解码格式](##%E6%8C%87%E4%BB%A4%E8%A7%A3%E7%A0%81%E6%A0%BC%E5%BC%8F)后，现在尝试解析指令数据将汇编指令打印出来，从代码中看出它并没有像传统的原生的汇编一样有堆栈指针寄存器，因为它是一个基于寄存器虚拟机。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_KG48MZSRPGKEB45.webp)

还原到原生
-----

目前还原到原生代码一共尝试了二种方案，第一种是将虚拟机指令构建到 LLVM IR，再使用 clang 编译器编译`.ll`文件生成原生代码，目前来看这是最佳方案。第二种是构建反编译器的 il 中间语言还原到伪 C 代码，目前只是进行了初步尝试并没有写完，有兴趣的朋友可以进行尝试。

### LLVMLIte

llvmlite 实现了 llvm ir 大多数的功能对我来说还原已经足够用了，不喜欢的可以更换官方 llvm 或者其他的库。

安装：

```
pip install llvmlite
```

开源仓库：[llvmlite](elink@c64K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6F1N6h3#2T1j5g2)9J5c8X3I4D9N6X3#2D9K9i4c8W2)

文档：[llvmlite 文档](elink@180K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6D9L8s2k6E0L8r3W2@1k6g2)9J5k6i4u0W2j5h3c8@1K9r3g2V1L8$3y4K6i4K6u0W2K9h3!0Q4x3V1k6W2L8W2)9J5c8X3I4S2N6r3g2K6N6q4)9J5c8X3W2F1k6r3g2^5i4K6u0W2K9s2c8E0L8l9`.`.)

### 构建 LLVM IR

虚拟在初始化外部指针、寄存器常量赋值时寄存器的类型信息丢失，导致在后面构建 IR 时做了很多类型和数据长度的转换操作，在反汇编角度来说数据类型只要认 1/2/4/8 字节数据、指针等等就行，没有类型信息反而更好编写。

#### 简单的 IR 操作基础知识：

还原的汇编有非常多的寄存器，不同的原生函数在被虚拟化后寄存器的数量还不一样，这些寄存器在阅读时可以当成变量来理解。在高级语言编码中定义一个变量完全不考虑寄存器的问题，因为编译器为我们自动分配了寄存器，同样 IR 语言中也不用考虑变量分配寄存器的问题。

**定义变量**：

C/C++ 定义一个局部变量是这样的，指定一个类型和变量名并未赋初值。

C/C++ 声明变量：

```
int num;
```

IR 声明变量:

```
t_int32 = ir.IntType(32)
ptr_num = builder.alloca(t, )
```

高级语言中 int num; 编译器默认是在堆栈分配变量，而在 IR 中 builder.alloca 也是在堆栈分配变量，与高级语言不同的是返回值是一个指针 int * ptr_num，想要把值拿出来使用必须使用 builder.load 把值取出来，这和高级语言中的指针取值运算类型 * ptr_num 类似。

**加法运算**：

c/c++；

```
int a = 111;
int b = 222;
int c = a + b;
```

IR:

```
t = ir.IntType(32)
# int a = 111;
ptr_a = builder.alloca(t, ) # 定义变量a，返回一个堆栈的指针引用
builder.store(ir.Constant(ir.IntType(32), 111), ptr_a)
# int b = 222;
ptr_b = builder.alloca(t, ) # 定义变量b，返回一个堆栈的指针引用
builder.store(ir.Constant(ir.IntType(32), 111), ptr_b)
# int c = a + b;
a_value = builder.load(ptr_a) # 从堆栈中把变量a的值取出来
b_value = builder.load(ptr_a) # 从堆栈中把变量a的值取出来
result = builder.add(a_value, b_value) # 将a和b的值进行加法运算，返回的结果是一个值不是堆栈指针。
ptr_c = builder.alloca(t, ) # 定义变量c，返回一个堆栈的指针引用
builder.store(result, ptr_c) # c = a + b;
```

简单的 3 行高级语言代码在 IR 中有不少行的逻辑，有点麻烦笨拙的感觉

**数值扩展**：

在 IR 中左值和右值类型或类型长度不至时是不能够直接参与运算的，需要进行转换后才能够使用。

c/c++：

```
signed char c = 0xf8;
signed int n = c;
```

IR：

```
t_i8 = ir.IntType(32)
t_i32 = ir.IntType(32)
# signed char c = 0xf8;
ptr_c = builder.alloca(t_i8, )
builder.store(ir.Constant(ir.IntType(32), 0xf8), ptr_c)
# signed int n = c;
ptr_n = builder.alloca(t_i32, )
builder.store(ir.Constant(ir.IntType(32), 0xf8), ptr_n)
sext_type = ir.IntType(32) # 定义一个需要扩展的目标类型int
c_value = builder.load(ptr_c) # 从栈栈中取出c的值
cast_value = builder.sext(c_value, sext_type) # 传入需要扩展的值和扩展的目标类型，注意返回值是值类型
builder.store(cast_value, ptr_n) # 将转换后的值放入变量c
```

**定义函数：**

想要向函数写入指令，首要声明**函数类型**并指定参数的类型和返回值类型，然后再定义函数对象，要知道函数对象承载了基本块对象而基本块承载了指令对象，只要向函数添加指令至少一个基本块之后才能向基本块写入指令，对于有多个基本块时要使用 “指令指针” 移动到该基本块，再向该基本块定入指令。

IR 指令指针移动到向基本块的未尾：

```
builder.position_at_end(curr_ir_basic_block)
```

IR 指令指针移动到向基本块的开始：

```
builder.position_at_start(curr_ir_basic_block)
```

**常用的 IR 指令**：

加减乘除:

```
builder.add(lhs, rhs)
builder.sub(lhs, rhs)
builder.mul(lhs, rhs)
builder.sdiv(lhs, rhs)
builder.udiv(lhs, rhs)
```

带符号取模操作：

```
temp = builder.sdiv(lhs, rhs)
result = builder.sub(lhs, builder.mul(temp, rhs))
```

指针类型之间的转换:

```
builder.bitcast(ptr, target_type)
```

整形转指针：

```
builder.inttoptr(val, target_ptr_type)
```

指针转整形:

```
builder.ptrtoint(val, target_int_type)
```

比较：

```
builder.icmp_signed("==", lhs, rhs) # lhs和rhs是左值和右值，整形比较的不要使用堆批指针
builder.icmp_signed("!=", lhs, rhs)
builder.icmp_signed(">", lhs, rhs)
builder.icmp_signed(">=", lhs, rhs)
icmp_unsigned("==", lhs, rhs)
icmp_unsigned("!=", lhs, rhs)
icmp_unsigned("<", lhs, rhs)
icmp_unsigned("<=", lhs, rhs))
```

无条件跳转：

```
builder.branch(target_basic_block)
```

有条件跳转：

```
builder.cbranch(builder.load(ptr_cond), true_br, false_br)
```

返回指令：

```
builder.ret_void()
```

```
ret_val = builder.load(ptr_ret_val)
builder.ret(ret_val)
```

#### 还原流程

我的还原方法不一定很好都多都是临时有想法加进去的，对 IR 非常熟悉的完全可以按照自己的想法去实现。

1.  解码虚拟机指令字节数据
    
    由于指令是不定变长的，这里把指令字节数据放到一张列表中，也方便在查找基本块时将指令加入到基本块中。
    
    ```
    vmState: VMState = BytecodeDecoder.build_vmp_state(
      filename, offset, size, extern_list)
    instructions = vmState.deocde_instruction()
    ```
    
2.  创建基本块
    
    这里的基本块是还原后的虚拟机基本块，基本块的终止符指令有：ret、switch、j、jcc 指令，从第一条指令开始扫描这些指令，并记录下这些指令的真假和多路跳转目标地址，这些跳转地址是基本块的起始地址，当遇到终止符指令结束基本块并把基本块的信息保存到基本块列表中。
    
    ```
    def get_jump_targets(insn):
        jump_targets = []
        match insn[0]:
            case 0x16:  # switch
                opcode, treg, type, defbranch, element_count, switch_branches = insn
                # [deftar, casetar, casetar, ...]
                jump_targets.append(defbranch)
                jump_targets.extend(switch_branches)
            case 0x1d:  # jcc
                truebr = insn[2]
                jump_targets.append(truebr)
        return jump_targets
    def is_successors_insn(insn):
        match insn[0]: # 指令的opcode
            case 0x13 | 0x16 | 0x1d: # ret: 0x13 | switch: 0x16 |  jmp/jcc: 0x1d
                return True
            case _:
                return False
    def find_basic_blocks(instructions):
        """识别基本块"""
        jump_targets = set()
        basic_blocks = []
        current_block = []
        # 先找出所有跳转目标
        for insn in instructions:
            targets = get_jump_targets(insn)
            if targets:
                jump_targets.update(targets)
        pc = start = end = 0
        for insn in instructions:
            insn_len = get_insn_length(insn)
            print(f"{pc:#x}, {insn_len}")
            # 如果当前指令是跳转目标，开始新块
            if pc in jump_targets and current_block:
                basic_block_info = {"start_hex": f"{start:#x}", "end_hex": f"{pc + insn_len:#x}",
                                    "start": start, "end": pc + insn_len,
                                    "basic_block": current_block}
                basic_blocks.append(basic_block_info)
                current_block = []
                start = pc + insn_len
                print(f"{'-' * 50}")
            current_block.append(insn)
            # 检查是否是基本块终止指令
            if is_successors_insn(insn):
                basic_block_info = {"start_hex": f"{start:#x}", "end_hex": f"{pc + insn_len:#x}",
                                    "start": start, "end": pc + insn_len,
                                    "basic_block": current_block}
                basic_blocks.append(basic_block_info)
                current_block = []
                start = pc + insn_len
                print(f"{'-' * 50}")
            end += insn_len
            pc += insn_len
        # 添加最后一个块（如果有）
        if current_block:
            basic_block_info = {"start_hex": f"{start:#x}", "end_hex": f"{pc + len(insn):#x}",
                                "start": start, "end": pc + len(insn),
                                 "basic_block": current_block}
            basic_blocks.append(basic_block_info)
        return basic_blocks
    ```
    
3.  初始化模块  
    创建 IR 模块并初始化一些基本参数，一个模块包含多个函数，一个函数包含多个基本块，一个基本块包含多条指令，而指令中包含操作码和一个或多个操作数。IR 模块是一个顶级模块，只要有模块对象就可以拿任何想要的数据。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_E37TPH3EDPSN5N5.webp)  
    创建模块并设置要编译的架构和目标平台：
    
    ```
    module = ir.Module(f"{filename}_{size:#x}_{extern_list:#x}")
    module.triple = "aarch64-unknown-linux-gnu" # 目标架构为linux aarch64
    ```
    
4.  声明外部函数声明
    
    告诉它函数中会调用 calloc 需要调用，从 dump 的汇编代码来看，CALLOC 的申请的内存大小基本上都小于 0x100，其实对于比较小的数据可以把 CALLOC 的内存转移到堆栈中，我并没有这么做原因太懒了写完还要去验证代码。
    
    ```
    size = 0xa8
    t_array = ir.ArrayType(ir.IntType(8), size)
    buffer = builder.alloca(t_array, )
    ```
    
    声明一个函数类型填好参数和返回值的类型，然后创建一个函数对象指令外部符号名称 "calloc"。
    
    ```
    def declare_external_calloc(module: ir.Module):
        declare_calloc = ir.FunctionType(ir.PointerType(ir.IntType(8)), [ir.IntType(32), ir.IntType(32)])
        fn_calloc = ir.Function(module, declare_calloc, "calloc")
        fn_calloc.args[0].name = "num"
        fn_calloc.args[1].name = "size"
        return fn_calloc
    ```
    
5.  定义一个入口函数
    
    虚拟机所有的指令将会使用 IR 接口向函数写入指令。
    
    ```
    def ini_entry_function(vmState: VMState, module: ir.Module, alloca_regs: dict):
        entry_func_define = create_type(vmState.entry_function)
        entry_func = ir.Function(module, entry_func_define, "entry_func")
        for i in range(vmState.entry_function.argumentCount):
            name = "A" + str(i)
            entry_func.args[i].name = name
            alloca_regs[name] = entry_func.args[i]
        # entry_func.append_basic_block("entry")
        return entry_func
    ```
    
6.  为入口函数创建所有的基本块
    
    调用 func.append_basic_block 为函数添加基本块，此时所有的基本块还未写入指令，因此它们是空的。
    
    ```
    def create_all_basic_blocks(func: ir.Function, basic_blocks):
        for bb in basic_blocks:
            func.append_basic_block(f"bb_{bb["start"]:x}")
    ```
    
    在 create_all_basic_blocks 函数返回后，在调试控制台输入 print(entry_func) 回车后打印该函数已经写入的所有 IR 信息。
    
    ```
    define void @"entry_func"(i8* %"A0", i32 %"A1", i8* %"A2")
    {
    bb_0:
    bb_32d:
    bb_340:
    bb_342:
    }
    ```
    
7.  为模块创建一个 IR 构建器
    
    在函数添加一个新的基本块 func.append_basic_block() 时，基本块对象成员中会有与之关联的上层的函数对象：parent，同样函数对象成员会一个上层的模块对象，因此创建 IR 构建器参数是一个基本块，很容易就可以关联到为模块创建一个构建器。
    
    ```
    builder = ir.IRBuilder(entry_func.blocks[0])
    ```
    
8.  初始化外部指针
    
    在上面一章还原的汇编代码中会将外部的指针设置到寄存器中，因此要为这些变量 (虚拟寄存器) 设置初值。之前有提到过
    
    外部地址和初始化寄存这二个步骤是没有类型信息的，这里暂时设置类型为 32 位的整形常量。
    
    ```
    def set_extern_regs(vmState: VMState, builder: ir.IRBuilder, alloca_regs):
        for i in range(0, len(vmState.extern.targetRegs)):
            v = get_register_ptr(alloca_regs, builder, vmState.extern.targetRegs[i], ir.IntType(32))
            builder.store(ir.Constant(ir.IntType(32), vmState.extern.externalAddress[i]), v)
    ```
    
9.  初始化寄存器
    
    为寄存器设置初始化值，这个初始化的值是一个常量，常量在后面指令执行时到结束都不会改变寄存器的值。前面说过这里常量来自未加密前原生汇编中的常量例如: MOV X3, #0x88，#0x88 就是一个常量它的值是不会发生改变的，生成虚拟化代码后可能就是 MOV V20, #0x88，初始化寄存不止有 MOV 指令还有会其他指令对目标寄存进行初始化。
    
    目前只添加了遇到的指令：
    
    ```
    def set_inial_regs(vmState: VMState, builder: ir.IRBuilder, alloca_regs):
        inial = vmState.inial
        types = vmState.types
        for i in range(0, len(inial.opcodes)):
            match inial.opcodes[i]:
                case 0 | 0xb | 0x17:
                    ptr_breg = get_register_ptr(alloca_regs, builder, inial.sRegs[i])
                    if isinstance(ptr_breg, ir.Argument):
                        val_breg = ptr_breg
                    else:
                        t = types[inial.type[i]]
                        if t.tag != 0xf:
                            ir_type = ir.PointerType(create_type(t))
                        else:
                            ir_type = create_type(t)
                        val_breg = builder.bitcast(ptr_breg, ir_type)
                    index_table = []
                    for idx in inial.imm[i]:
                        index_table.append(ir.Constant(ir.IntType(32), idx))
                    ele_ptr = builder.gep(val_breg, index_table)
                    builder.store(ele_ptr, get_register_ptr(alloca_regs, builder, inial.dRegs[i], ele_ptr.type))
                case 7:
                    v = get_register_ptr(alloca_regs, builder, inial.dRegs[i], ir.IntType(32))
                    builder.store(ir.Constant(ir.IntType(32), inial.imm[i]), v)
                case 8:
                    t = types[inial.type[i]]
                    if t.tag != 0xe:
                        v = get_register_ptr(alloca_regs, builder, inial.dRegs[i], ir.IntType(32))
                        builder.store(ir.Constant(ir.IntType(32), 0), v)
                    else:
                        raise Exception("error")
                case 0x15:
                    print(
                        f"{'-' * 6:<}    {'-' * 5}registers const{'-' * 6}    MOV\tV{inial.dRegs[i]}.T@{Decoder.get_type_annotation(types[inial.type[i]])}, V{inial.sRegs[i]}")
                case _:
                    raise Exception("error")
    ```
    
10.  预分配  
    这个是为了生成的代码好看，先让 IR 代码开始的位置分配堆栈变量先把堆栈坑给占了，编译器编译后自动计算这个坑的内存大小，例如：sub sp, sp, #0x240，#0x240 就大小就是编译器在生成的函数时就为我们计算出的坑大小。如果不在函数开头不预分配会发生什么样的情况呢，在代码生成的中间部分会临时修改堆栈的指针分配堆栈的内存，这个频率会非常的多，这会大大降低了汇编代码的可读性，尽管反编译器生成伪 C 代码的优化会把它掉，后面的章节会有校验虚拟机汇编和生成的原生汇编逻辑是否一致，通过对比来验证我们生成 IR 代码是否有问题。  
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_54QZ5ZES46Y623Q.webp)  
    遍历虚拟机指令先把指令中的目标寄存先分配了：
    
    ```
    def get_register_ptr(alloca_regs, builder: ir.IRBuilder, reg, t: ir.Type = ir.IntType(64)):
        name = f"V{reg}"
        arg_name = f"A{reg}"
        if alloca_regs.get(arg_name, None) is None:
            if alloca_regs.get(name, None) is None:
                alloca_regs[name] = builder.alloca(t, name=name)    # 这分配堆栈变量
            return alloca_regs[name]
        else:
            return alloca_regs[arg_name]
    def preallocation_registers(basic_blocks, types: list[Type], alloca_regs, builder, vmState: VMState):
        pc = 0
        for bb in basic_blocks:
            for insn in bb["basic_block"]:
                print(f"prealloc regs addr: {pc:#x}")
                opcode = insn[0]
                match (opcode):
                    case 0x1:   # ARITH
                        t = types[insn[2]]
                        sreg1 = insn[3]
                        sreg2 = insn[4]
                        dreg = insn[5]
                        get_register_ptr(alloca_regs, builder, dreg, create_type(t))
                    case 0x2:   # MOV
                        op2 = insn[1]
                        dtype = types[insn[2]]
                        stype = types[insn[3]]
                        sreg = insn[4]
                        dreg = insn[5]
                        match op2:
                            case 0 | 5 | 0xA | 0xC:
                                get_register_ptr(alloca_regs, builder, dreg, create_type(stype))
                            case _:
                                get_register_ptr(alloca_regs, builder, dreg, create_type(dtype))
                    case 0x6:   # CALLOC
                        a2_type = types[insn[1]]
                        a1_type = types[insn[2]]
                        sreg = insn[3]
                        dreg = insn[4]
     
                        t2_type = create_type(a2_type)
                        get_register_ptr(alloca_regs, builder, dreg, ir.PointerType(t2_type))
                        # get_register_ptr(alloca_regs, builder, dreg, t2_type)
                    case 0xa:   # STR
                        type = types[insn[1]]
                        sreg = insn[2]
                        mreg = insn[3]
                        # get_register_ptr(alloca_regs, builder, sreg, create_type(type))
                        get_register_ptr(alloca_regs, builder, mreg, create_type(PointerType(type)))
                    case 0xc:  # CMP
                        dreg = insn[4]
                        get_register_ptr(alloca_regs, builder, dreg, ir.IntType(1))
                    # case 0xe:  # BSEL.PHI
                    case 0xf:  # CALL
                        func_define: FunctionType = types[insn[1]]
                        if func_define.returnValueType.tag != 0:
                            ret_reg = insn[3]
                            t_ret_val = create_type(func_define.returnValueType)
                            get_register_ptr(alloca_regs, builder, ret_reg, t_ret_val)
                    case 0x13:  # RET
                        ret_type = types[insn[1]]
                        if ret_type.tag != 0:
                            ret_reg = insn[2]
                            get_register_ptr(alloca_regs, builder, ret_reg, create_type(ret_type))
                    case 0x16:  # SWITCH
                        raise Exception("not implemented")
                    case 0x1d:  # JCC
                        cond = insn[1]
                        jmpTrue = insn[2]
                        if cond != 0:
                            flagReg = insn[3]
                            t = types[insn[4]]
                            get_register_ptr(alloca_regs, builder, flagReg, create_type(t))
                    case 0x1e:  # CSEL
                        dreg = insn[6]
                        type = types[insn[3]]
                        get_register_ptr(alloca_regs, builder, dreg, create_type(type))
                    case 0x28:  # GEP
                        dreg = insn[1]
                        t = types[insn[3]]
                        # for idx in insn[6]:
                        #     if not vmState.is_static_reg(idx):
                        #         ptr_reg = get_register_ptr(alloca_regs, builder, idx)
                        #         gep_idx = builder.load(ptr_reg)
                        # get_register_ptr(alloca_regs, builder, dreg, create_type(t))
                    case 0x2a:  # LDR
                        t = types[insn[1]]
                        dreg = insn[3]
                        get_register_ptr(alloca_regs, builder, dreg, create_type(t))
                pc += get_insn_length(insn)
    ```
    
11.  获取虚拟机指令中的所有 PHI 指令和 PHI 参数信息  
    从所有基本块一个一个查找 PHI 指令，找到后先把 PHI 的 result 值在堆栈中先分配占坑预分配堆栈，然后从 phi 指令参 数中取出前驱基本块和前驱基本块对应的值。
    
    ```
    def get_phi_nodes(state: VMState, alloca_regs, builder: ir.IRBuilder, func: ir.Function, basic_blocks, ):
        phi_nodes = []
        for bb in basic_blocks:
            bb_name = f"bb_{bb["start"]:x}"
            ir_basic_block = get_ir_basic_block_by_name(func, bb_name)
            for insn in bb["basic_block"]:
                match insn[0]:
                    case 0xe:
                        # 初始化phi结果(目标)寄存器
                        phi_store_reg = insn[1]
                        builder.store(ir.Constant(ir.IntType(64), 0), get_register_ptr(alloca_regs, builder, phi_store_reg, ir.IntType(64)))
                        # 获取phi信息
                        phi_node_info = []
                        for node in insn[3]:
                            val = node[0]
                            if not state.is_static_reg(val):
                                # 当值是变量寄存器则分配一个堆栈变量并赋值为0
                                ptr_val = get_register_ptr(alloca_regs, builder, val, ir.IntType(64))
                                builder.store(ir.Constant(ir.IntType(64), 0), ptr_val)
                            ir_bb = get_basic_block(func, node[1])
                            phi_node_info.append([val, ir_bb])
                        phi_nodes.append({"phi_basic_block": ir_basic_block, "phi_store_reg": phi_store_reg, "phi_node_info": phi_node_info})
        return phi_nodes
    ```
    
12.  为所有基本块中的指令添加 IR 指令  
    经过前面所有的准备步骤后，终于可以添加 IR 指令了，前面说过了要添加指令必须要让 builder 的指令指向基本块的一个位置，在基本块迭代的开头位置首先指定要写入指令的基本块。  
    基本块经过前面的寄存器初始化后已经写入了一些指令了，现在要指令指针移动到基本块的尾部 builder.position_at_end(curr_ir_basic_block)。
    
    ```
    for bb in basic_blocks:
        bb_name = f"bb_{bb["start"]:x}"
        curr_ir_basic_block = get_ir_basic_block_by_name(entry_func, bb_name)   # 当前指令所在的基本块
        builder.position_at_end(curr_ir_basic_block)
    ```
    
    从基本块中取出指令开始遍历虚拟机指令生成 IR 代码，为了方面阅读下面只是框架的部分代码：
    
    ```
    for bb in basic_blocks:
        bb_name = f"bb_{bb["start"]:x}"
        curr_ir_basic_block = get_ir_basic_block_by_name(entry_func, bb_name)   # 当前指令所在的基本块
        builder.position_at_end(curr_ir_basic_block)    # 构建器指针移动到新的基本块未尾
        for insn in bb["basic_block"]:
            print(f"addr: {pc:#x}")
            opcode = insn[0]
            match (opcode):
                case 0x1:   # ARITH
                    op2 = insn[1]
                    t = types[insn[2]]
                    sreg1 = insn[3]
                    sreg2 = insn[4]
                    dreg = insn[5]
                    match op2:
                        case 0: # XOR
                          # TODO IR
                        case 0x1: # SUB
                          # TODO IR
                        case 0x2: # LSR
                          # TODO IR
                        case 0x3: # UDIV
                          # TODO IR
                        case 0x4: # ADD
                          # TODO IR
                        case 0x5:   # OR
                          # TODO IR
                        case 0x6: # SMOD
                          # TODO IR
                        case 0x7: # SIDV
                          # TODO IR
                        case 0x8: # UMOD
                        case 0xA: # ASR
                          # TODO IR
                        case 0xC: # LSL
                          # TODO IR
                        case _:
                            # TODO Handler Not Impl
                case 0x2:   # MOV
                    op2 = insn[1]
                    dtype = types[insn[2]]
                    stype = types[insn[3]]
                    sreg = insn[4]
                    dreg = insn[5]
                    match op2:
                      case 0 | 5 | 0xA | 0xC: # 保持两个操作操作数类型一致性
                        # TODO IR MOV dreg, sreg
                      case 0x1:   # 扩展
                        # TODO IR  MOV dreg, sreg.8
                      case 0x7:   # 截取
                        # TODO IR  MOV dreg.d, sreg.8
                      case 0x9:
                        # TODO IR
                      case _:
                          # TODO Handler Not Impl
                case 0x6:   # CALLOC
                    a2_type = types[insn[1]]
                    a1_type = types[insn[2]]
                    sreg = insn[3]
                    dreg = insn[4]
                    # TODO call calloc(num, size)
                case 0xa:   # STR
                    type = types[insn[1]]
                    sreg = insn[2]
                    mreg = insn[3]
                    # TODO STR sreg, [mreg]
                case 0xc:  # CMP
                    t = types[insn[1]]
                    reg1 = insn[2]
                    reg2 = insn[3]
                    dreg = insn[4]
                    op2 = insn[5]
                    match op2:
                      case 0x20:  # CMP_EQ
                            # TODO
                      case 0x21:  # CMQ_NE
                          raise Exception(f"CMP.NE not impl")                           
                      case 0x24:  # CMP_CC
                          raise Exception(f"CMP.CC not impl")                           
                      case 0x28:  # CMP_LT
                          raise Exception(f"CMP.LT not impl")                           
                      case _:
                          input(f"未识别的CMP指令op2={op2:#x}")
                case 0xe:  # BSEL.PHI
                    phi_basic_blocks.append(curr_ir_basic_block)    # 记录phi指令所在的基本块
                case 0xf:  # CALL
                    # TODO
                case 0x13:  # RET
                    ret_type = types[insn[1]]
                    if ret_type.tag == 0:
                        # TODO 没有返回值时
                    else:
                        # TODO 有返回值时
                case 0x16:  # SWITCH
                    raise Exception("not implemented")
                case 0x1d:  # JCC
                      # ------ 查找当前节点是否PHI中的参数前驱节点 ------
                      # 到了终止符指令了，在写入终止符前遍历PHI节点的前驱节点参数label是否在当前基本块，如果找到则
                    # 1.保存当前基本块信息，
                    # 2. 并取出PHI参数节点变量
                     
                    # ------ 然后才开始处理跳转指令 ------
                    cond = insn[1]
                    jmpTrue = insn[2]
                    if cond == 0:  # jmp
                        # TODO IR 连接后续基本块
                    else:  # j.cond
                        flagReg = insn[3]
                        t = types[insn[4]]
                        jmpFalse = insn[5]
                        # TODO IR 连接真和假块
                case 0x1e:  # CSEL
                    ntype = insn[pc + 1]
                    if ntype.tag == 0x10:
                        raise Exception("error")
                    creg = insn[pc + 2]
                    type = insn[pc + 3]
                    treg = insn[pc + 4]
                    freg = insn[pc + 5]
                    dreg = insn[pc + 6]
                    # TODO IR
                case 0x28:  # GEP
                    dreg = insn[pc + 1]
                    none = insn[pc + 2]
                    type = insn[pc + 3]
                    count = insn[pc + 4]
                    breg = insn[pc + 5]
                    # TODO IR
                case 0x2a:  # LDR
                    type = insn[pc + 1]
                    mreg = insn[pc + 2]
                    dreg = insn[pc + 3]
                    # TODO IR
    ```
    
    在写入完所有指令后开始处理 PHI 指令，PHI 指令必须是基本块的第一条指令位置，因此将 IR 指令指针移动到基本块最前方的位置，然后再添加 PHI 组合参数 (变量，基本块 < label>)，最后保存 PHI 结果变量。  
    量。
    
    ```
    # 最后处理phi指令
     if phi_basic_blocks:
         for bb in phi_basic_blocks:
             for phi_node in phi_nodes:
                 if phi_node["phi_basic_block"] == bb:
                     builder.position_at_start(bb)   # 基本块的最前面插入phi指令
                     phi = builder.phi(int64_type, f"phi_var_{bb.name}")
                     for node in phi_node["phi_node_info"]:
                         val = node[0]
                         phi.add_incoming(val, node[1])
                     dreg_ptr = get_register_ptr(alloca_regs, builder, phi_node["phi_store_reg"], int64_type)
                     builder.store(phi, dreg_ptr)
    ```
    
    **最后打印 IR 保存**:
    
    ```
    print(module)
    ```
    
    ![](https://bbs.kanxue.com/upload/attach/202504/271698_A92J99E5GS2VVDW.webp)
    

#### 编译 IR

为了验证编译 IR 生成的原生代码编译不被编译器优化删减，添加参数禁用优化 - O0，没有定义 main 函数编译为动态库：-shared。

```
clang -O0 -shared devmp_0x168B60.ll -o devmp_0x168B60_O0.o
```

**反汇编校对逻辑**

IDA PRO 载入 devmp_0x168B60_O0.o，查看还原的原生代码和虚拟机汇编逻辑是否一致。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_4K93QBZSAYMNAGK.webp)  
**禁用优化的伪 C 代码**  
![](https://bbs.kanxue.com/upload/attach/202504/271698_MAV3NC352GGSU7M.webp)**优化编译**

优化参数调整为 O1 生成目标代码

```
clang -O1 -shared devmp_0x168B60.ll -o devmp_0x168B60_O0.o
```

打开优化查看原生汇编，发现代码被精减了好多。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_G34QUYEANGFFJN5.webp)  
F5 查看优化编译后的伪 C 代码逻辑清晰了。  
![](https://bbs.kanxue.com/upload/attach/202504/271698_K9866K97UP69NPR.webp)

### 构建 Binary Ninja IL

但它不乏也是一种还原思路，目前做了尝试使用 Binary Ninja 构建 IL 进行还原只做了少部分几条指令没有时间做下去了，从还原的几条指令效果来看，这个方法是行得通的，但没有还原到 llvm IR 效果那么好，llvm 编译器优化做的非常到位，甚至有的时候能够将非常多的指令精简到难以想像的结果，精简后代码量少了非常方便于进行阅读分析指令。有兴趣的可以参考附加的文件`DecodeBNIL.py`

虚拟机和 llvm bitcode 虚拟机
=====================

IR 中有一个指令 getelementprt 和虚拟机中的一条指令逻辑非常像，之前这条指令名叫 ADD.MO，MO 是 member offset 的简写，查找 llvm 相关代码发现虚拟机和 llvm bitcode 有非常大的关系，发现 bitcode 中的指令、[字节码的解析](elink@a58K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6D9L8s2k6E0i4K6u0r3L8r3I4$3L8g2)9J5k6s2m8J5L8$3A6W2j5%4c8Q4x3V1k6T1L8r3!0T1i4K6u0r3L8h3q4A6L8W2)9J5c8X3I4D9N6X3#2Q4x3V1k6A6L8X3y4D9N6h3c8W2i4K6u0r3L8r3I4$3L8g2)9J5c8V1u0A6N6s2y4@1M7X3g2S2L8g2)9J5c8V1u0A6N6s2y4@1M7X3g2S2L8g2u0W2j5h3c8W2M7W2)9J5k6h3S2Q4x3U0y4x3x3U0t1^5)、[解释器](elink@aebK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6D9L8s2k6E0i4K6u0r3L8r3I4$3L8g2)9J5k6s2m8J5L8$3A6W2j5%4c8Q4x3V1k6T1L8r3!0T1i4K6u0r3L8h3q4A6L8W2)9J5c8X3I4D9N6X3#2Q4x3V1k6D9K9h3u0Q4x3V1k6q4P5r3g2U0N6i4c8A6L8$3&6q4L8X3N6A6L8X3g2Q4x3V1k6u0L8Y4c8W2M7Y4m8J5k6i4c8W2M7W2)9J5c8V1g2^5k6h3y4#2N6r3W2G2L8W2)9J5k6h3y4H3M7l9`.`.)等等两者的逻辑和虚拟机非常的相似，想必大家已经猜了它是由什么改造而来的吧，文章写到止已经非常庞大了不做过多介绍了，有兴趣的可以阅读 llvm bitcode 的相关代码。

总结
==

*   判断是否虚拟机  
    单从 cfg 控制流图中是否很难判断出来，目前我没有快速的方法去判断，虚拟机保护的目的是隐藏真实的代码执行，如果想要确定虚拟机或混淆或者还是混淆中包含虚拟机，在确定是否虚拟机之前要提前了解混淆的原理和特征去排除纯混淆代码。  
    虚拟机在执行时有取指、解码、执行的 handler 三个步骤，三个步骤之间有时还会有 switch 分发表的连接 (刻意隐藏的除外)，一个完整的虚拟机保护 handler 会有完整的指令集模拟支持，这意味着 hanler 数量会非常的多：数据移动 MOV 类、算术运算加减乘除、逻辑运算与或非取反、调用子程序 (外部函数)、内存访问等等，执行完 handler 会返回到取指令的位置，根据虚拟机的一些特性去综合判断，通常都是要分析一部分代码的逻辑才能确认，如果发现此类指令的模拟基本上可以确认是虚拟机了。总之来说需要分析经验的积累，简单的可能需要 1-3 天，复杂的可以要 1-3 周才能确定。
    
*   关于分析时间
    
    很多人都喜欢问这个分析了多久，的确这是一个非常重要的时间考量。对于简单的虚拟机分析在 1-2 周的工作日，国内大厂的一般在 2-4 周工作日左右，还有一些国外的世界级国际大厂，对于国际大厂他们做的非常好强度是非常高的则需要时间 2-4 个月，按这个时间成本来算的话已经达到了强不可催的目的了。
    
    > 之前的短视频虚拟分析和还原脚本大约 3.5 周的工作日时间，合计 18 天左右，另外加上写文章 1.5 周的工作日，总耗时约 5 周的工作日，文中分析的 so 是一个未经混淆并且字节码未加密的虚拟机，实际上在其他位置 so 中的虚拟机是一套东西，只不过混淆有所加强字节码被加密了，分析难度虽说有加强但也不是非常大。
    
    > 此 app 的虚拟机分析和汇编还原脚本合计总耗时大约 4 周的工作日，IR 编写大约 1.5 周的工作日，总耗时接近 6 周。
    
*   关于还原的理论  
    对于任何虚拟机指令接近原生指令的可以借用原生汇编指令还原到汇编，而对于虚拟机拥有自定义指令集的，理论来说都可以先还原到中间语言然后再还原到原生汇编。
    

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 26 分钟前 被金罡编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)