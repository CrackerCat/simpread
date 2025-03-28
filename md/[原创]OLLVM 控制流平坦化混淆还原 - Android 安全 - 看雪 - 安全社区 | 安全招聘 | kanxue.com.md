> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286151-1.htm)

> [原创]OLLVM 控制流平坦化混淆还原

### 背景

最近在研究 ollvm 反混淆，刚好遇到此样本，借此文章对 ollvm fla 控制流平坦化进行一个反混淆分析，顺便分享下 idapython 相关 api 的知识。

### 分析目标

娜迦加固 libgeiri.so

函数: init_proc

### 工具

ida7.7

pycharm: 编写 idapython 代码, 需要从 ida 安装目录下导入相关的库

![](https://bbs.kanxue.com/upload/attach/202503/947335_BFAPNWJ85J68HY4.webp)

### ollvm fla 介绍

下面是一个标准的平坦化 cfg 图, 经过 ollvm fla 混淆后几行的代码代码最后会膨胀至上百行。这里每个矩形图都叫做基本块，在这个图里我们主要关心序言块、return 块和真实块。只有这三类的基本块才是我们需要的。其他的都是 ollvm 混淆加上的，这里统称虚假块。

![](https://bbs.kanxue.com/upload/attach/202503/947335_H69982EJTEKV49S.webp)

基本块: 由多行指令组成，一般最后一条指令以跳转指令或 ret 结尾

主分发器: 类似 switch-case 结构，通过传入的状态变量，并决定跳转至目标块。

预处理器: 我们可以抽象认为汇集在预处理器的基本块均为真实块

以下可以帮我们了解块与块之间的关系

1.  函数的开始地址为序言（Prologue）的地址
2.  序言的后继为主分发器（Main dispatcher）
3.  后继为主分发器的块为预处理器（Predispatcher）
4.  后继为预处理器的块为真实块（Relevant blocks）
5.  无后继的块为 retn 块
6.  剩下的为无用块与子分发器（Sub dispatchers）

这里我画个图概述下什么是前驱和后继，这两个名词也是用得比较多的。

注意看箭头指向，可以看到下面的执行流程是由 **A** 到 **B** 再到 **C**, 那么这里 **B** 就是中间块

因为 **A** 是 **B** 的上一个基本块那么 **A** 就是 **B** 的前驱

因为 **C** 是 **B** 的下一个基本块, 那么 **C** 就是 **B** 的后继

![](https://bbs.kanxue.com/upload/attach/202503/947335_3D35R3YHUNRK2HR.webp)

### 反混淆

上面简要说了下 ollvm 混淆的相关知识，那么我们应该如何去反混淆呢。这里只需要记住一个核心观念：找真实块 (序言块、汇集到预处理器的所有块、return 块)。找到所有真实块后，还需要做的是理清块与块之间的链接关系，最后需要做的工作就是根据块与块之间的关系，修改块的跳转指令 patch 到对应的块。这样就实现了平坦化的反混淆。

### 分析

ida 打开 so，进入到 init_proc 函数

![](https://bbs.kanxue.com/upload/attach/202503/947335_W4CA4YAT7RNX3T5.webp)

F5 进入伪代码

![](https://bbs.kanxue.com/upload/attach/202503/947335_TFCKG6K86FCYWTS.webp)

流程混乱，无法直观的去分析。

根据上面反混淆的理念，我们来理一理这个混淆的相关结构

序言块: 0x43058

![](https://bbs.kanxue.com/upload/attach/202503/947335_PQ4XQUCC6FJ5XFT.webp)

主分发器: 0x43120

![](https://bbs.kanxue.com/upload/attach/202503/947335_J3X3MET2K9XTP64.webp)

通过这个主分发器块，调用 block.preds() 函数, 我们可以得到主分发器块的所有前驱块, 也就是真实块

return 块：0x43500

![](https://bbs.kanxue.com/upload/attach/202503/947335_F6DBJXWA7BQSFNJ.webp)

到此, **序言块**，**汇集到主分发器的所有块**,**return** 块这三大类的所有块我们都找到了，那我们就可以反混淆了吗。当然不是，我们只是收集到了所有块，但是还不知道块与块之间的跳转关系，所以我们还需要理清块跳转关系。

![](https://bbs.kanxue.com/upload/attach/202503/947335_N2DFHZ8SFRYV2R7.webp)

我们先点击主分发器的地址，可以看到 ida 识别到所有引用该地址的基本块，这里提一嘴，"后继为预处理器的块为真实块" 等上文 6 点理论都是广义上的，我们可以那样去认为，但是还是会有个别的情况与这 6 点相驳。

![](https://bbs.kanxue.com/upload/attach/202503/947335_QSTXKUFNH39NCQX.webp)

比如这个基本块它的后继也是主分发器 (这里的函数没有预处理器，而是真实块直接连接主分发器), 难道它就是真实块吗，当然不是，这里它知识重置了状态变量 w8，以供下一个 switch 跳转, 从汇编中可以看出其中并没有任何的真实块逻辑。

![](https://bbs.kanxue.com/upload/attach/202503/947335_H7EVCPR7X66GHG3.webp)

接着看这里的 B.NE loc_43120

w8,w9 不相等走向主分发器，否则走向下面的基本块, 我们也可以看到下一个基本块的最后一条指令也是直接跳转到主分发器的, 所以总结出 B.NE 这个连接到主分发器的块不是真实块，只是一个重置 switch 跳转状态变量的块。所以我们只需要关注汇集到主分发的块的同时, 最后一条指令还得是 B 指令。

好了, 主分发器的前驱块分析完了，我们接着去理清跳转关系

这里我以 0x43490 基本块为参照物, 逻辑都是一样的。

```
loc_43490
ADRP            X8, #off_97FB8@PAGE
LDR             X8, [X8,#off_97FB8@PAGEOFF]
LDR             W8, [X8]
STR             W8, [X19,#0x20]
MOV             W8, #0x39649A15
B               loc_43120
```

这里的 MOV W8, #0x39649A15 命令是重置状态变量, 当我们循环到 switch 的时候, 会匹配当前 w8 的值, 对应上值时就跳转到相应的位置, 所以可知 0x39649A15 是当前基本块的后继。但是这个后继我们目前只知道索引，我们是不是还得知道索引所代表的基本块是哪个啊

![](https://bbs.kanxue.com/upload/attach/202503/947335_7AWDH4Z95576V53.webp)

点击 0x39649A15, 得到这个基本块

```
MOV             W9, #0x39649A15
CMP             W8, W9
B.NE            loc_43120
```

W8, W9 不相等是跳转到 loc_43120 主分发器, 这个我们不用关心，我们去看另一个走向得到如下图示

![](https://bbs.kanxue.com/upload/attach/202503/947335_UDFDASBFH4RT8YW.webp)

这个基本块的地址是 0x431D8，所以我们是不是得到了 0x39649A15 索引指向 0x431D8 基本块的地址了, 得到 0x43490->0x39649A15,

0x39649A15->0x431D8, 换算可得 0x43490->0x431D8

![](https://bbs.kanxue.com/upload/attach/202503/947335_5RN9FNVSK5356Y8.webp)

简单画个图，以 0x43490 为参照, 后继上面说了，这里我们看前驱。可以看到它的前驱存储的索引是 0x3455F111, 然后查找 0x3455F111 被谁引用, 发现 0x43314 的后继索引是 0x3455F111。跟着这个现象我们可以得知真实块自身存储的索引是后继, 前驱存储的索引是此基本块的前驱。

接下来开始写脚本去混淆。

### 脚本处理

##### 获取主分发器

```
def findDispatchers(func_start,num = 10):
    func = idaapi.get_func(func_start)
    blocks = idaapi.FlowChart(func)
    pachers = []
    for block in blocks:
        preds = block.preds()
        preds_list = list(preds)
        if len(preds_list) > num:
            pachers.append(block)
    return pachers
```

拿到主分发器块的地址, 结果正确

![](https://bbs.kanxue.com/upload/attach/202503/947335_HYD8539TZG9HH4U.webp)

##### 获取汇聚到主分发器的所有块

可以借此查看真实块的相关特征

```
def findLoopEntryBlockAllPreds(loop_end_ea):
    block = getBlockByAddress(loop_end_ea)
    for pred in block.preds():
        ea = idc.prev_head(pred.end_ea)
        print("主分发器前驱基本块:", hex(ea), idc.GetDisasm(ea))
```

##### 存储所有真实块

![](https://bbs.kanxue.com/upload/attach/202503/947335_ADF4KAC9ZYT7JJF.webp)

这里先判断真实块的最后一条指令操作符是 B, 并且操作数也必须是主分发器地址 0x43120。然后取每个真实块 B 指令的上一条指令, 根据 MOV 和 MOVK 的不同特征去匹配后继索引。

```
MOV             W8, #0xA6FB
STR             W0, [X19,#0x24]
MOVK            W8, #0x7986,LSL#16
```

简要说下这种汇编, 它的意思是把 0x7986 左移 16 位, 并且保持低位数据不变, 这里低位是上述的 w8 = 0xA6F8，1 个字节 2 个字符, 1 个字节 8 个比特位, 所以一个字符占 4 个比特位, 左移 16 位后变为 0x79860000, 再和低位和并就是 0x7986A6FB。算术运算符就是 w8 =(0x7986<<16) |0xA6FB。

![](https://bbs.kanxue.com/upload/attach/202503/947335_58PP6TQ6S4445QM.webp)

CSEL W8, W24, W8, EQ

注意这种的真实块, 命令 CSEL，这里是带分支的真实块

真实块的连接要么是**顺序连接**只有一个后继

要么就是**分支连接**, 有两个后继, 也就是 if else。

这里样本 CSEL 都是有 4 个操作数的, EQ 是操作符, 当 EQ 条件满足时把 w24 赋值给 w8, 否则把 w8 赋值给 w8。总之不管它的操作符如何变，条件为真时, Z 标志位为 1，都是取第 2 个操作数 (w24), 为假取第 3 个操作数 (w8)

这里我就直接把相关的索引添加进去了, 数量少就没去匹配特征了，数量多的话建议特征处理

![](https://bbs.kanxue.com/upload/attach/202503/947335_DX65MNJM7KTVKX7.webp)

![](https://bbs.kanxue.com/upload/attach/202503/947335_ZZD7P6STS4JYE97.webp)

粘贴到 ida 里运行, 得到所有真实块的前驱索引和后继索引

![](https://bbs.kanxue.com/upload/attach/202503/947335_MGZ5RQVVKFSCQ3W.webp)

##### 验证真实块跳转关系

这里有个疑问就是, 怎么保证这些块的连接是不是都正确呢。针对这个我也写了个函数，验证这个对象里的连接是否都是正确

![](https://bbs.kanxue.com/upload/attach/202503/947335_BU87QDWSXUJCJAB.webp)

![](https://bbs.kanxue.com/upload/attach/202503/947335_4D3FP8HPWVYG2DJ.webp)

得到上述的关系调用链, 以函数地址开头, 以 return 块结尾, 所有存储的真实块调用链执行成功无异常报错，如果其中执行失败会打印以下信息, 数组的最后一个元素 0x434ac 里存储的 0xbd9fbba 索引没有找到后继块，针对这个报错可以去 ida 查看对应的信息, 或者直接向 stamp 对象里补 0x434ac 相关的后继信息。有点类似 unidbg 的味道, 缺啥补啥。

![](https://bbs.kanxue.com/upload/attach/202503/947335_FRN2CTWFM6Q79DK.webp)

##### 重建块连接

最后就是调用此函数去重建块连接

```
def rebuildControlFlow(state_map):
    for block in state_map:
        block_ea = int(block,16)#需要把字符串转成int
        # 获取真实块保存的前驱、后继链接块索引
        value = state_map[block]
        # 查找块尾的跳转指令
        endEa = getBlockByAddress(block_ea).end_ea
 
        last_insn_ea = idc.prev_head(endEa)
        if idc.print_insn_mnem(last_insn_ea) == "B":
            # 如果是无条件跳转(B)
            if len(value) == 2:
                succ_index = value[0] #当前真实块的后继块索引
                if succ_index == None: #return块没有后继,过滤掉它
                    continue
                jmp_addr = getSuccBlockAddrFromMap(state_map,succ_index) #获取后继索引对应的真实块地址
                patchBranch(last_insn_ea, jmp_addr)
 
            # 如果是条件跳转（CSEL）
            elif len(value) == 3:
                succ_0 = value[0] #后继块的索引值
                jmp_addr_0 = getSuccBlockAddrFromMap(state_map, succ_0) #后继块的地址
                patchBranch(last_insn_ea, jmp_addr_0,1)
 
                succ_1 = value[1]   #后继块的索引值
                jmp_addr_1 = getSuccBlockAddrFromMap(state_map, succ_1)
                patchBranch(last_insn_ea, jmp_addr_1)
        if idc.print_insn_mnem(last_insn_ea) == "MOV":
            succ_index = value[0]  # 当前真实块的后继块索引
            # if succ_index == None:  # return块没有后继,过滤掉它
            #     continue
            jmp_addr = getSuccBlockAddrFromMap(state_map, succ_index)  # 获取后继索引对应的真实块地址
            patchBranch(last_insn_ea, jmp_addr)
```

所有流程走完了，现在去 ida 执行脚本

![](https://bbs.kanxue.com/upload/attach/202503/947335_JVVWEXJAN8V56RN.webp)

可以看到反混淆后的代码已经能直观看到运行整个代码的运行逻辑了。

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [libgeiri.so](javascript:void(0);) （2.59MB，49 次下载）
*   [ollvm_geiri.py](javascript:void(0);) （13.80kb，67 次下载）