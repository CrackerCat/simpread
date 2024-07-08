> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282412.htm)

> [原创] 去除反混淆后程序中的冗余汇编代码

一、前言
====

在之前去混淆的过程中，发现去混淆的程序中，伪代码与原程序伪代码相差无几，然而如果仔细观察汇编代码的话，可以发现存在较多的冗余代码，即混淆过程中产生的垃圾代码。那么如何去除这些冗余汇编指令呢？

二、基于 Liveness Analysis 的想法
==========================

先给出我们需要解决一个例子（这是一个虚假控制流混淆中的部分代码，其中标`*`的为冗余汇编指令）：

```
  mov     rax, [rbp+var_10]
  movsxd  rcx, [rbp+var_14]
  movsx   eax, byte ptr [rax+rcx]
  add     eax, [rbp+var_18]
  mov     [rbp+var_18], eax
* mov     rax, offset x
* mov     eax, [rax]
* mov     rcx, offset y
* mov     ecx, [rcx]
* mov     edx, eax
* sub     edx, 1
* imul    eax, edx
* and     eax, 1
* cmp     eax, 0
* setz    al
* cmp     ecx, 0Ah
* setl    cl 
* or      al, cl    
* test    al, 1      
* jnz     loc_401235


```

最开始的想法是从第 20 行的汇编指令开始，跟踪与该指令以及后续指令相关的指令。如何跟踪呢？简单想就是跟踪这些指令使用到的寄存器。

2.1 Liveness Analysis
---------------------

`Liveness Analysis`是一种数据流分析技术，用于确定程序中的每个变量在程序的不同点上是否 “活跃”。“活跃” 的意思是变量在某个程序点后将被使用，且在这之前没有被重新定义。这项技术在编译器优化、寄存器分配和死代码消除中起着关键作用。

![](https://bbs.kanxue.com/upload/attach/202407/985561_M5JX8UE6FK3AZGW.webp)

本文章中主要用到了里面的`def`和`use`集合的想法。

*   `def`集合：当前汇编指令显式和隐式改变的寄存器、内存的集合。
*   `use`集合：当前汇编指令显式和隐式使用的寄存器、内存的集合。

2.2 改进 Liveness Analysis 以适配
----------------------------

然而从图中看，似乎`Liveness Analysis`技术没有考虑到标志寄存器的改变和使用。对于`Liveness Analysis`技术能否解决我所预设的问题，我也不太清楚。因此在此基础上，进行如下改进：

*   引入标志寄存器来跟踪相关指令，这对于一些条件指令的跟踪是有帮助的。
*   对于内存操作数（例如`[rbp+var_10]`），还应提取其表达式中的寄存器，存入`use`集合中。因为我认为既要跟踪内存地址是怎么来的，也要跟踪内存地址对应的值是怎么来的。
*   对于指令中操作数既不是寄存器，也不是内存的（例如`var_4`、全局变量`x`），应认为它们是常量，即不加入`use`集合中。
*   由于指令中出现的寄存器名可能是同一寄存器的不同位数区分（例如`rax, eax, ax, ah, al`），统一将它们转换成最大位数的寄存器名。

对刚才那个例子进行`def`和`use`集合的分析，结果如下：

```
  mov     rax, [rbp+var_10]              def={rax}       use={[rbp+var_10], rbp}         
  movsxd  rcx, [rbp+var_14]              def={rcx}       use={[rbp+var_14], rbp}
  movsx   eax, byte ptr [rax+rcx]        def={rax}       use={[rax+rcx], rax, rcx}
  add     eax, [rbp+var_18]              def={rax}       use={rax, [rbp+var_18], rbp} 
  mov     [rbp+var_18], eax              def={[rbp+var_18]}       use={rax, rbp}//理应认为[rbp+var_18]的寻址用到了rbp
* mov     rax, offset x                  def={rax}       use={}//认为x是具体值，不加入到use集合中
* mov     eax, [rax]                     def={rax}       use={rax, [rax]}//既要知道内存地址对应的值从哪来的，也要知道内存地址从哪来的
* mov     rcx, offset y                  def={rcx}       use={}//认为y是具体值，不加入到use集合中
* mov     ecx, [rcx]                     def={rcx}       use={rcx, [rcx]}
* mov     edx, eax                       def={rdx}       use={rax}
* sub     edx, 1                         def={rdx}       use={rdx}
* imul    eax, edx                       def={rax}       use={rax, rdx}
* and     eax, 1                         def={rax}       use={rax}
* cmp     eax, 0                         def={rflags}     use={rax}
* setz    al                             def={rax}       use={rflags}
* cmp     ecx, 0Ah                       def={rflags}     use={rcx}
* setl    cl                             def={rcx}       use={rflags}
* or      al, cl                         def={rax}       use={rax, rcx}
* test    al, 1                          def={rflags}     use={rax}
* jnz     loc_401235                     def={}          use={rflags}


```

具体跟踪的初步想法是：

1.  给定某一指令 I ，初始化跟踪集合 trace=useI​ （指令 I 的 use 集合），初始化相关指令列表 relevantInsns=[I]。
2.  往上追溯每条指令，如果 trace∩defIi​​=∅ ，那么认为指令 Ii​ 是相关指令，将指令 Ii​ 加入到 relevantInsns 中， 同时在 trace 集合删除交集中的元素，并将指令 Ii​ 的 use 集合中的元素加入进来，即进行如下运算：trace=(trace−(trace∩defIi​​))∪useIi​​ 。如果 trace∩defIi​​=∅ ，那么认为指令 Ii​ 是不相关指令，不进行任何操作。
3.  在追溯的过程中，如果 trace=∅ 或者待跟踪指令集为空，则认为跟踪结束。

感觉说的有些混乱，就那上面这个例子解释一下吧。首先我们假设跟踪的指令为`jnz loc_401235`，后续步骤如下：

```
初始化: trace = {rflags}	relevantInsns = [20]	(为了简洁，用行号代替指令)	
往上分析:
19: trace ∩ use = {rflags},  是相关指令。   更新 trace = {rax}, relevantInsns = [19, 20]
18: trace ∩ use = {rax},     是相关指令。   更新 trace = {rax, rcx}, relevantInsns = [18, 19, 20]
17: trace ∩ use = {rcx},     是相关指令。   更新 trace = {rax, rflags}, relevantInsns = [17, 18, 19, 20]
16: trace ∩ use = {rflags},  是相关指令。   更新 trace = {rax, rcx}, relevantInsns = [16, 18, 19, 20]
15: trace ∩ use = {rax},     是相关指令。   更新 trace = {rcx, rflags}, relevantInsns = [15, 16, 18, 19, 20]
14: trace ∩ use = {rflags},  是相关指令。   更新 trace = {rax, rcx}, relevantInsns = [14, 15, 16, 18, 19, 20]
13: trace ∩ use = {rax},     是相关指令。   更新 trace = {rax, rcx}, relevantInsns = [13, 14, 15, 16, 18, 19, 20]
12: trace ∩ use = {rax},     是相关指令。   更新 trace = {rax, rcx, rdx}, relevantInsns = [12, 13, 14, 15, 16, 18, 19, 20]
11: trace ∩ use = {rdx},     是相关指令。   更新 trace = {rax, rcx, rdx}, relevantInsns = [11, 12, 13, 14, 15, 16, 18, 19, 20]
10: trace ∩ use = {rdx},     是相关指令。   更新 trace = {rax, rcx}, relevantInsns = [10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
9: trace ∩ use = {rcx},      是相关指令。   更新 trace = {rax, rcx, [rcx]}, relevantInsns = [9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
8: trace ∩ use = {rcx},      是相关指令。   更新 trace = {rax, [rcx]}, relevantInsns = [8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
7: trace ∩ use = {rax},      是相关指令。   更新 trace = {rax, [rax], [rcx]}, relevantInsns = [7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
6: trace ∩ use = {rax},      是相关指令。   更新 trace = {[rax], [rcx]}, relevantInsns = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
5: trace ∩ use = {},         不是相关指令。     trace = {[rax], [rcx]}, relevantInsns = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
4: trace ∩ use = {},         不是相关指令。     trace = {[rax], [rcx]}, relevantInsns = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
3: trace ∩ use = {},         不是相关指令。  	 trace = {[rax], [rcx]}, relevantInsns = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
2: trace ∩ use = {},         不是相关指令。   	 trace = {[rax], [rcx]}, relevantInsns = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
1: trace ∩ use = {},         不是相关指令。  	 trace = {[rax], [rcx]}, relevantInsns = [6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 19, 20]
待分析指令集为空，结束跟踪


```

对于上述例子，我们的想法成功的追踪到了所有冗余指令。然而当存在`cmovcc`指令时，我们的想法并不是很理想。

2.3 cmovcc 指令所带来的问题
-------------------

以下是控制流平坦化混淆的垃圾汇编指令（其中标`*`的为冗余汇编指令）。

```
* mov edx, [rbp+var_4]			def={rdx} 			use={[rbp+var_4], rbp}
* mov eax, 4A...h 				def={rdx} 			use={}
* mov ecx, 32..h				def={rcx} 			use={}
* cmp edx, 2					def={rflags} 		use={rdx}
* cmovnz eax, ecx				def={rax} 			use={rcx,rflags}	或者	def={}		use={rflags}
* mov [rbp+var_1c], eax  		def={[rbp+var_1c]} 	use={rax, rbp}
* jmp loc_401418 				def={} 				use={}


```

由于`cmovnz`存在执行和不执行的情况，因此需要分两种情况讨论：

*   当标志寄存器满足`cmovnz`的条件时，执行`cmovnz`指令，此时`def={rax}`, `use={rcx,rflags}`。
*   当标志寄存器不满足`cmovnz`的条件时，不执行`cmovnz`指令，此时`def={}`, `use={rflags}`。

单独分析以上任何一种情况，其结果都不能覆盖所有的冗余指令，但其两者情况的并集能够覆盖所有冗余指令。因此在向上溯源时，需要注意是否时`cmovcc`指令，如果是，则需要保存当前状态，探索其中一条路径，当该路径探索完毕后，再探索另一种路径。此情况可通过递归实现。

三、代码实现
======

接下来主要展示代码中重要的部分，完整代码请参考 [https://github.com/gal2xy/AssembleCodeTracer。](https://github.com/gal2xy/AssembleCodeTracer%E3%80%82)

3.1 汇编指令字典的建立
-------------

对于刚才所展示的例子中，我们可以直观的看出指令的`def`和`use`集合，但是计算机不行。这是我所面临的第一个麻烦，解决这个问题，后续分析才能进行下去。为此，我建立了一个简单的汇编指令字典，部分示例如下：

```
'mov': {
        'insns': ['mov', 'movsx', 'movzx', 'movsxd'], # 类似指令的集合, 由于一些指令的def和use集合可能是相同的，因此这么做是为了避免冗余
        'has_types': False,                           # 该指令格式是否有多个，即操作数个数是否不固定
        'def_exp_index': [0],                         # 显式def，即在指令中出现的，存储对应索引值
        'use_exp_index': [1],                         # 显式use，即在指令中出现的，存储对应索引值
        'def_imp_reg': [],                            # 隐式def，即在指令中未出现的但是又被该指令更改了值的
        'use_imp_reg': [],                            # 隐式use，即在指令中未出现的但是又被该指令使用了的
}
 
'imul': {
    'insns': ['imul'],
    'has_types': True,
    'types': {
        # 指令中的操作数个数为1个
        1: {
            'def_exp_index': [],
            'use_exp_index': [0],
            'def_imp_reg': ['rax', 'rflags'],
            'use_imp_reg': ['rax'],
        },
        # 指令中的操作数个数为2个
        2: {
            'def_exp_index': [0],
            'use_exp_index': [0, 1],
            'def_imp_reg': ['rflags'],
            'use_imp_reg': [],
        },
        # 指令中的操作数个数为3个
        3: {
            'def_exp_index': [0],
            'use_exp_index': [0, 1, 2],
            'def_imp_reg': ['rflags'],
            'use_imp_reg': [],
        }
    },
}

```

3.2 解析指令中的操作数
-------------

由于汇编指令中的操作数有多种类型，且对于不同类型的操作数，我们需要进行不同的操作。

*   寄存器类型可以直接放入`def`和`use`集合中。
*   内存不仅可以直接放入`def`和`use`集合中，而且还需要解析其表达式中的寄存器，这些寄存器也要放入`def`和`use`集合中。
*   立即数则不应放入`def`和`use`集合中，应认为它是某一寄存器跟踪结束的标志。

```
def parseOperand(operand: str):
 
    operand = operand.strip()
 
    # 判断是否为寄存器.后一个捕获r8~r15相关寄存器
    if re.match(r'^[er]?[abcds][xlhpi]$', operand) or re.match(r'(r[89]|r1[0-5])[dwb]?', operand):
        return 'register'
    # 判断是否为内存操作数
    elif re.match(r'^\[.*\]$', operand):
        return 'memory'
    # 判断是否为立即数
    elif re.match(r'(?:0[xX])?[0-9A-Fa-f]+h?', operand):
        return 'immediate'
    return 'unknown'

```

3.3 从内存表达式中提取寄存器
----------------

由于指令中的内存可能是一个表达式的情形，例如`[2*rax+rbx-1]`，我们可以直观的看出，它肯定使用了`rax`、`rbx`寄存器，因此我们需要实现一个函数来提取内存表达式中的寄存器。

```
def findRegInMemOp(operand: str):
 
    pattern = r'\b([er]?([abcds][xlhpi]|bp|sp)|r[89]|r1[0-5])\b'
 
    registers = set([])
    # 查找内存操作数中的所有寄存器
    registersInOperand = re.findall(pattern, operand)
    print(f'find registers in memory operand: {registersInOperand}')# [('rbp', 'bp')]
 
    for registersTuple in registersInOperand:
        for register in registersTuple:
            fullName = findFullRegisterName(register)
            if fullName is not None:
                registers.add(fullName)
            else:
                print(f'Warning!!! Unknow register: {register}, may be you need define it in registers_db')
 
    print(f'memory operand: {operand}, extract registers: {registers}')
    return registers

```

3.4 获取指令中的 def 和 use 集合
-----------------------

这一步就需要借助到 3.1 所定义的汇编指令字典。整个函数的功能为：

*   获取`def`集合：获取当前指令的`def_exp_index`键值，根据索引值列表找到对应的具体操作数，并根据操作数的类型进行不同操作。之后再获取当前指令的`def_imp_reg`键值，将隐式定义的寄存器加入到`def`集合中。
*   获取`use`集合：同理类似。

```
def getDefAndUseSet(insnsDic: dict, operands: List):
 
    defSet = set([])
    useSet = set([])
    # 先找显式定义的
    # 遍历需要加入到def集合中的操作数索引
    for pos in insnsDic['def_exp_index']:
        # 根据操作数的类型决定是否添加到def集合中
        operandType = parseOperand(operands[pos])
        if operandType == 'register':  # 寄存器直接加入
            defSet.add(operands[pos])
        elif operandType == 'memory':  # 内存操作数
            defSet.add(operands[pos])
            # 内存操作数借助的寄存器放入useSet中
            registers = findRegInMemOp(operands[pos])
            useSet.update(registers)
        else:  # 未知类型，例如label标签
            continue
    # 再找隐式定义的
    for register in insnsDic['def_imp_reg']:
        defSet.add(register)
 
    # useSet同理
    for pos in insnsDic['use_exp_index']:
        # 根据操作数的类型决定是否添加到use集合中
        operandType = parseOperand(operands[pos])
        if operandType == 'register':  # 寄存器直接加入
            useSet.add(operands[pos])
        elif operandType == 'memory':  # 内存操作数，则读取表达式中的寄存器并加入
            useSet.add(operands[pos])
            registers = findRegInMemOp(operands[pos])
            useSet.update(registers)
        else:  # 未知类型，例如label标签
            continue
 
    for register in insnsDic['use_imp_reg']:
        useSet.add(register)
 
    # 由于寄存器存在高低位之分，所以转成最大的寄存器的名称
    newDefSet = set([])
    for register in defSet:
        fullName = findFullRegisterName(register)
        if fullName is not None:
            newDefSet.add(fullName)
        else:# 其他类型的操作数，例如[rbp+var_4], rflags
            newDefSet.add(register)
     
    # 将寄存器名全部转成最大位数的寄存器名
    newUseSet = set([])
    for register in useSet:
        fullName = findFullRegisterName(register)
        if fullName is not None:
            newUseSet.add(fullName)
        else:
            newUseSet.add(register)
 
    return newDefSet, newUseSet

```

3.5 跟踪相关汇编指令
------------

最终算法如下：

1.  给定待跟踪的指令`I`，以及所需跟踪的指令范围，初始化 traceSet=useSetI​，relevantInsPos={I}，startPos=posI​，其中 posI​为指令 I 在 Insns 中的索引值。
2.  开始向上探索（即 startPos−−），对于每一条指令 Ii​，i∈[0,posI​)：
    1.  如果 traceSet∩defSetIi​​=∅ ，则指令 Ii​ 为相关指令，更新 tarceSet=(traceSet−defSetIi​​)∪useSetIi​​ 以及 Ii​→relevantInsPos。（对于`cmovcc`指令，需先进行如下操作：定义 newTarceSet=traceSet∪{rflags} ，递归调用当前算法，将返回结果当前 relevantInsPos 合并)
    2.  如果 traceSet∩defSetIi​​=∅ ，则指令 Ii​ 为非相关指令，不进行任何操作。
3.  如果 traceSet=∅或者 startPos<0， 则跟踪结束，返回 relevantInsPos ，否则继续进行步骤 2。

```
# 跟踪既定汇编指令的相关汇编指令
def traceInstructionsInBlock(instructions: List, defAndUseSet: List[Tuple[Set, Set]], startPos: int, traceSet=None):
    # trace = trace - def + use 集合运算
    if traceSet is None:
        traceSet = defAndUseSet[startPos][1]# trace = use
 
    relevantInsPos = set([startPos])
    print(f'traceSet: {traceSet}')
 
    currPos = startPos - 1
    while currPos >= 0 or traceSet is None:# 结束标志: tarce集合为空, 或者分析到达基本块首部
        currDefSet = defAndUseSet[currPos][0]
        currUseSet = defAndUseSet[currPos][1]
        print(f'instructions: {instructions[currPos]}, currDefSet = {currDefSet}, currUseSet = {currUseSet}')
         
        # 求 traceSet 和 currDefSet 的交集，如果不为空，则当前指令是相关指令
        intersectionSet = traceSet & currDefSet
        print(f'交集: {intersectionSet}')
        if len(intersectionSet) != 0:# 是相关指令
            if instructions[currPos].startswith('cmov'): # cmov指令特殊，应分两种情况进行讨论
                # 此情况是条件不成立，cmov不执行。
                # 不减交集, 而是直接并use={rflags}
                newTraceSet = traceSet | set(['rflags'])
                # 在该情况的基础上进行探索
                anotherRelevantInsPos = traceInstructionsInBlock(instructions, defAndUseSet, currPos, traceSet=newTraceSet)
                relevantInsPos.update(anotherRelevantInsPos)
             
            # 以下是cmov执行的情况, 以及除cmov指令外的其他指令
            # 做差集, 减去被定义的操作数
            traceSet = traceSet - intersectionSet
            # 做并集，加入后续需要跟踪的操作数
            traceSet = traceSet | currUseSet
            print(f'traceSet: {traceSet}')
 
            # 添加当前指令到相关指令集中
            relevantInsPos.add(currPos)
 
        currPos -= 1
 
    return relevantInsPos

```

四、不足之处
======

当真实汇编指令与垃圾汇编指令构成了标志寄存器的前者定义后者使用的关系时，此算法会造成误判。具体例子如下所示。（其中标`*`的为冗余汇编指令，其余为真实代码）

```
  mov     rax, [rbp + var_10] 
  movsx   edx, byte ptr [rax + 3]
* mov     eax, 7A39EAC0h
* mov     ecx, 9BEE23F6h
  cmp     edx, 65h 
* cmovl   eax, ecx
* mov      [rbp + var_1C], eax
* jmp      loc_401418


```

由于`cmovl`指令依赖标志寄存器，因此会将第 5 行的`cmp`指令纳入到相关指令集合中（然而第 5 行的`cmp`指令是真实代码），以至于后续将其余真实代码也囊括进来，最终造成了误判！这个问题似乎解决不了，所以我认为我这个想法到头来还是失败的。

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

[#其他](forum-161-1-129.htm)