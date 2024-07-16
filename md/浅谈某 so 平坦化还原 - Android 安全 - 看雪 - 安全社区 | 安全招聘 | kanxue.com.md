> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282514.htm)

> 浅谈某 so 平坦化还原

前言
==

现在的平坦化混淆越来越刺激了。如下图中目标函数 CFG 很复杂。但是不慌，Binary Ninja 的 [Workflows](https://docs.binary.ninja/dev/workflows.html) 功能可以实现通过修改生成的 IR 指令实现修改反编译器结果。本文中不涉及 Workflows 编写，具体编写参考官方文档。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_T56PTHFAD5XP6N2.webp)

使用 Binary Ninja 版本如下图。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_FZ3H25TQUFWRBP9.webp)

分析
--

在序言块中将原栈变量升级为堆变量，导致后续无法对变量进行交叉引用分析。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_QEB87FVDZTATEGQ.webp)

通过对主分发器的判断值交叉引用发现 case 值并不是直接的常量，而是通过当前 case 值加一个常量得到。导致不能直接还原 switch 的 case 结构，只能通过遍历形式计算全部 case 值。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_SP76UEQ3MPN3FA8.webp)

在真实块中也使用当前 case 值计算作为常量混淆。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_UGECW8DMJREK2NV.webp)

以上三点是该混淆特殊的处理，剩下的就和标准的 OLLVM 差不多了。

还原
--

首先需要一个遍历机制，找出真实块之间的连接关系。由于编译器公共子表达式消除，导致多个真实块尾部指令被优化为一个块。在重构控制流时需要将复用块 dump 一份连接到目标真实块尾部，保持数据流操作完整。例如下图中的块存在多个前驱块。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_NRZNKCBV9Y4RK9W.webp)

### **识别控制变量**

在上文中提到该混淆将原栈变量全部升级为堆变量，该操作的确增加了分析难度。但同样也暴露了当前的栈变量都有可能是平坦化混淆所创建的。在 Binary Ninja 的 Medium IR 下遍历序言块中全部的 MediumLevelILSetVar 指令，且定义值为常量。下图中 x21 为控制量。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_7USC4WT2EH7WPA8.webp)  
遍历获取目标变量

```
entry_block = self.m_func.basic_blocks[0]
for _ in entry_block:
    if isinstance(_, bn.MediumLevelILSetVar) and isinstance(_.src, bn.MediumLevelILConst):
        self.constant_value[_.dest] = _.src.value.value

```

但常量定义并不是仅有一个，例如下图中在目标函数的序言块上就存在两个栈上的常量定义。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_R4QYDHE99S5U5JC.webp)

根据控制变量的预期操作，可以建立以下判断。

1、该变量用于 switch 判断跳转，但在编译器优化下 switch 转换为由 if 构建二分法查找，所以该变量将被很多 if 使用。

2、使用目标变量的 if 指令中操作数一定由目标变量和一个常量值组成。

根据以上规则建立判断，即可找出控制变量。找出被 if 指令使用最多的一个变量，该值即为 switch 的控制变量。

遍历采用污点传播规则，通过 du 链传播到 if 指令为止。

```
effect: List[Any] = []
passed = set()
constant_target = [var]
u_list = [var]
 
while len(u_list) != 0:
    current = u_list.pop()
    if current in passed:
        continue
    passed.add(current)
    used = self.m_func.get_var_uses(current)
    for _ in used:
        if isinstance(_, bn.MediumLevelILSetVar):
            if is_constant(_, constant_target):
                constant_target.append(_.dest)
                u_list.append(_.dest)
                effect.append(_)
        elif isinstance(_, bn.MediumLevelILIf):
            if is_constant(_, constant_target):
                effect.append(_)
        elif isinstance(_, bn.MediumLevelILStore):
            effect.append(_)
        else:
            bn.log_warn("Unexpected instruction")
            break

```

但在 Binary Ninja 的 Medium IR 下并不是标准的三地址形式 IR。如下图中第一条指令是 MediumLevelILSetVar 指令，该指令的 src 为加法指令，加法指令 inline 到 SetVar 中，没有独立为一个条加法指令。所以在上文遍历实际上是包含了全部来源于指定变量的全部 MediumLevelILSetVar 和 MediumLevelILIf 指令。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_Y5EFTXAMXPSPWBH.webp)

注意在常量判断中需要去掉 MediumLevelILAddressOf 指令。

```
def is_constant(code: bn.MediumLevelILInstruction, expected: List[bn.Variable]):
    tmp = list(code.traverse(Deobfuscation.__foreach))
    tmp.reverse()
    for _ in tmp:
        # 移除stack操作
        if isinstance(_, bn.MediumLevelILAddressOf):
            return False
        if isinstance(_, bn.MediumLevelILVar) and _.var not in expected:
            d_list.add(_.src)
            return False
    return True

```

但在实际处理时又有一个新的问题。在下图中 if 指令的操作数没有直接满足 2 条件，常量被提取成一个独立的定义指令，没有 inline 到 if 中。这时先额外记录该 if 指令，等待其他 if 指令处理完成后。额外检查该指令的操作数是否来源于一个常量值。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_MDM7P466XXCMF4H.webp)

额外处理该类型的 if 指令。

```
while len(d_list) != 0:
    current = d_list.pop()
    used = self.m_func.get_var_definitions(current)
    matched = True
    for _ in used:
        if not isinstance(_, bn.MediumLevelILSetVar):
            matched = False
            break
        if not is_constant(_.src, constant_target):
            matched = False
            break
    if not matched:
        continue
    for _ in used:
        effect.append(_)
    for _ in self.m_func.get_var_uses(current):
        effect.append(_)

```

找到控制变量后需要根据该值寻找真实块之间的连接情况。

### 遍历连接块

通过模拟执行在上文中寻找控制变量时得到的相关指令，即可得到真实块连接路径。

在开始遍历前需要先确定入口点，也就是第一个执行的 if 指令。在以往的经验中有序言块的后继不一定是分发器的 if，也可能是个真实块。所以需要另一种方式寻找入口 if 指令。

下图为 IDA 的 CFG，红框为主分发器也就是需要寻找的入口 if 指令所在的块，其后的皆为子分发器。这里需要运用图论中[支配](https://oi-wiki.org/graph/dominator-tree/)的相关理论，在下图的情况下：**树形结构中，一个点的支配点越少，该点的支配域越大**。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_Q9335RFRRXJDQEA.webp)

在树形结构下，根节点一定支配整个树，所以其拥有最大的支配域。根据上文中遍历得到的相关指令中找出主分发器块。

```
first_if: Optional[bn.MediumLevelILIf] = None
min_dominator_size = 0xFFFFFFFF
for _ in self.effect:
    if not isinstance(_, bn.MediumLevelILIf):
        continue
    dominator_size = len(_.il_basic_block.dominators)
    if dominator_size < min_dominator_size:
        min_dominator_size = dominator_size
        first_if = _
return first_if

```

在找出主分发器后开始设计块遍历代码。入口点是从主分发器开始向后遍历，避免序言到主分发器之间存在真实块。

将当前 case 值和来源块进入循环。

```
search_values = [(self.constant_value[self.dispatcher_value], self.main_dispatcher)]
 
passed = set()
while len(search_values) != 0:
    current, start_block = search_values.pop()
 
    if current in passed:
        continue
    passed.add(current)
 
    values: Dict[Any, int] = {self.dispatcher_value: current}
    work_list = [self.main_dispatcher]

```

遍历时首先从当前块中找出相关指令位置。当该块没有控制变量的相关指令时或结尾的跳转指令与控制变量无关时认为该块为真实块，记录该块对应的 case 值。否则根据 if 结果继续遍历后继块。

这里会存在一种特殊情况，子分发器重定义了控制变量后直接指向了主分发器。这种情况也把子分发器视为真实块。

```
while len(work_list) != 0:
    target_block: bn.MediumLevelILBasicBlock = work_list.pop()
    i = -1
    for _ in target_block:
        if _ not in self.effect:
            break
        i = self.effect.index(_)
        break
    if i == -1:
        resolve(target_block)
        break
    may_block = self.effect[i].il_basic_block
    if may_block[may_block.instruction_count - 1] not in self.effect:
        resolve(may_block)
        break
    terminal = self.emulate(i, values)
    if not isinstance(terminal, bn.MediumLevelILIf):
        err("Something gone wrong!")
    cond = values[terminal.condition]
    next_block = self.m_func.get_basic_block_at(terminal.true if cond else terminal.false)
    if next_block == self.main_dispatcher:
        # 子分发器直接指向顶级分发器，把子分发器视为真实块
        resolve(target_block)
    else:
        work_list.append(next_block)

```

#### 常量处理

由于常量混淆导致当前块中常量值都来源于控制变量计算得到，所以在已经计算出当前块的 case 值时即可计算出当前块中全部常量值。在块存在复用情况下不同的 case 值到达该块后会存在不同的值，由于当前没有处理块复用的情况，不能直接替换表达式为常量。记录当前 case 下对应的值。一直遍历到主分发器为止。

```
work_list = [current_block]
 
fix_values = {}
while len(work_list) != 0:
    current = work_list.pop()
    if current == self.main_dispatcher:
        continue
 
    for _ in current:
        if _ not in self.effect:
            continue
        if isinstance(_, bn.MediumLevelILSetVar):
            self.emulate(self.effect.index(_), values, 1)
            fix_values[_.dest] = values[_.dest]
        else:
            bn.log_warn("Skip " + str(_))
    for _ in current.outgoing_edges:
        work_list.append(_.target)

```

#### 后继处理

由于常量混淆导致无法直接获取后继块的 case 值，必须要根据当 case 值计算。分为以下两种情况：

1、当前块到后继为 goto，仅有一个后继 case 值。

2、没有 case 值。

3、当前块为 if 条件分支，存在两个后继 case 值。

1 和 2 处理很简单，这里不再赘述。

在 3 情况如下图，在图中需要把左右分支块当作真实块处理。同时需要记录 case 值是 true 或 false。

![](https://bbs.kanxue.com/upload/attach/202407/1001027_CXZGBM63M9JEMW3.webp)

### 重建 CFG

最麻烦的部分。下文为在 Binary Ninja 中替换新后继的一个简单示例。该示例中 xxx 为目标指令 index 值，inst 为被替换的指令。而且 **Binary Ninja 的 Workflows 中不支持在任意位置插入指令**。如果在 Dump 块时先创建一个 goto 指令占位，则需要额外一次的 replace_expr，将 goto 指向正确的后继块。

```
label = MediumLevelILLabel()
label.operand = xxx
m.replace_expr(inst, m.goto(label))

```

所以本文采用根据真实块之间的依赖关系，从后向前处理。

首先需要先标记全部复用块。遍历全部真实块之间的路径，找出真实块的后继中多次被使用的块。

```
used = {}
for _ in self.patch_target:
    blocks = []
    work_list = [self.patch_target[_].dst]
    while len(work_list) != 0:
        current = work_list.pop()
        if current == self.main_dispatcher:
            continue
        blocks.append(current)
        if current in used:
            used[current] += 1
        else:
            used[current] = 1
        for _ in current.outgoing_edges:
            work_list.append(_.target)
 
for block, num in used.items():
    if num <= 1:
        continue
    self.reuse.append(block.start)

```

处理环时将回边的最后一段路径先切断，等待其他路径连接完成后再处理。

使用拓扑排序进行处理块的连接。在重建 CFG 时存在以下几种情况：

1、A 块到 B 块都不是复用块，且仅使用 goto 连接。

2、A 块到 B 块中任意一个或都为复用块，且仅使用 goto 连接。

3、A 块为 if 分支块连接 B 块与 C 块（这里存在特殊情况，在本文选取的目标函数中 if 块均为复用块，由于常量混淆导致 if 块可以根据当前 case 不同跳转到其他分支）。

第一种情况：在 A 块后继块中或 A 块自身中指向主分发器的 goto 指令替换指向真实块。

第二种情况：细分为两种情况，后继块为复用块时，前驱块为复用块时。

由于本文是反向连接块，假设 A->B，B 为复用块，dump B 后记为 B1，记录 B1 的指令范围与原块的映射关系。因为后续在 B 的作为前驱块时，后继块需要连接到 B1 上的。所以在后继为复用块时，仅寻找是否已经 dump 该块。

```
def get_block(self, block: bn.MediumLevelILBasicBlock, new_block: bool = False) -> int:
    """生成block id
 
    :param block: 原MediumLevelILBasicBlock
    :param new_block: 强制新建块
    :return: block id
    """
    if not new_block and block.start in self.s2i_mapper:
        return self.s2i_mapper[block.start]
    if block.start not in self.reuse:
        target = block.start, block.end - 1
    else:
        target = self.dump_block(block)
    current = self.current_id
    self.i2r_mapper[current] = target
    self.s2i_mapper[block.start] = current
    self.current_id += 1
    return current

```

当 A 为复用块时，需要强制新建 dump 块。因为假设 C 为 A 的前驱块，当处理 C 时 A 需要被 dump，但在 A dump 后已经无法找到 B 了。B 已经在上轮中处理过了。所以在反向处理时，前驱块如果为复用块是强制 dump，后继块仅使用已经创建的块。

```
def get_end_block(self, block: bn.MediumLevelILBasicBlock) -> int:
    """获取从指定块到主分发器之间最后一条跳转指令所在的块
 
    :param block:
    :return: block id
    """
    wait = [(-1, block)]
    block_id = -1
    while len(wait) != 0:
        src_id, tmp = wait.pop()
        if tmp == self.main_dispatcher:
            if block_id == -1:
                block_id = self.get_block(self.main_dispatcher)
            break
        block_id = self.get_block(tmp, True)
        if src_id != -1:
            target_end = self.i2r_mapper[src_id][1]
            new_label = bn.MediumLevelILLabel()
            new_label.operand = self.i2r_mapper[block_id][0]
            self.func.replace_expr(self.func[target_end], self.func.goto(new_label))
        for _ in tmp.outgoing_edges:
            wait.append((block_id, _.target))
    return block_id

```

第三种情况：dump 分支包含的全部块

在上文中中断了环，在全部的 goto 指令处理完成后，需要额外修复环的连接。

在处理指令后需要调用 [finalize](https://api.binary.ninja/binaryninja.mediumlevelil-module.html#binaryninja.mediumlevelil.MediumLevelILFunction.finalize) 重新计算基本块，调 用 [generate_ssa_form](https://api.binary.ninja/binaryninja.mediumlevelil-module.html#binaryninja.mediumlevelil.MediumLevelILFunction.generate_ssa_form) 重新生成 SSA 指令，High IR 是通过 Medium IR 的 SSA 生成的，在调用 generate_ssa_form 才会计算 High 和伪 C 代码。

最终效果
----

![](https://bbs.kanxue.com/upload/attach/202407/1001027_6ZPQYWEVVE7KRVB.webp)

[[培训] 科锐软件逆向 50 期预科班报名即将截止，速来！！！ 50 期正式班报名火爆招生中！！！](https://mp.weixin.qq.com/s/HFghXQRTiTlk6oRKGotpHA)

最后于 10 小时前 被 Moezero 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)