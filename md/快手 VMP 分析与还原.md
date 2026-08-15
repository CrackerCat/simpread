> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2123217-1-1.html)

> [md]# 快手 VMP 分析与还原 > ** 作者：** 人生导师 > ** 日期：** 2026 年 8 月 14 日 -## 前述在现在看来，VMP 沉重的保护还是最难攻破与维护的内容，并 ...

![](https://avatar.52pojie.cn/data/avatar/002/43/67/04_avatar_middle.jpg)rsds0duck

快手 VMP 分析与还原
------------

> **作者：** 人生导师  
> **日期：** 2026 年 8 月 14 日

* * *

### 前述

在现在看来，VMP 沉重的保护还是最难攻破与维护的内容，并且现在的方法大多数也还是在延续 Unpacking Virtualization Obfuscators（WOOT 2009）的六步法则

1.  逆向 VM 解释器（找 dispatcher/handler）
2.  找虚拟化代码的控制流入口
3.  写一个字节码反汇编器
4.  反汇编出 VM 字节码（得到 IR）
5.  做编译器式优化（去掉 VM 的样板开销）
6.  重新生成可执行代码

当然，这六步不好吗？当然很好但是这对于当今日益复杂与庞大的现代 VMP 来说几乎是不够用的，啃完现在版本，下一个版本 handler 又发生变化了，还需要再去做，这是和厂家进行时间竞赛，也是折磨自己，所以我们要进行一种现代化的方法，从汇编与地址热点来看它们有什么特点，今天的 VMP 案例就是`快手的 __NS_xfalcon`：选择它是因为它小巧、简单适合写文章使用，我 trace 了很多份日志，一共是不超过 40 万行汇编日志，这也在很多大佬的耐力范围内，可以通过字节码关系进行算法的还原，各位也可以使用我的 [Trace](https://github.com/djskncxm/DuckTrace)

### 一、定位 VMP

然后我们就可以使用 Linux 的 shell 命令进行一些小小的筛选了，假设我是不知道这个东西的，但是据我了解到

*   handler 内跳转 handler
*   ollvm 当作分发器进行的 vmp

那么最简单的就是看地址热度了，如果在一块连续的内存区域执行了几万次，那几乎可以确定是分发器 vmp 了，如果它执行的虽然多但是和日志量不怎么匹配那么就需要看跳转是否多了，一般都是 br x 寄存器的形式，我是很推荐各位对自己的或者用的顺手的 trace 做一个小小的解析工具，这样统计数据什么的就非常快  
经过我的小小探索，得到它是链式 VMP 结构，那么就直接开找，直接搜`br x8`之类的，直接搜通用寄存器就行，国内 Android VMP 还没那么牛逼，还是很简单的，拿个 handler 给各位看看

### 二、handler 切割

```
0x1D8C4                 LDRSH           X8, [X26,#8]
LDRSH           X9, [X26]
LDR             X8, [X21,X8]
LDR             X9, [X21,X9]
ADD             X8, X21, X8
STR             X8, [X23,X9]
ADD             X8, X26, #0x10
LDR             X8, [X8]
ADD             X26, X26, #0x18
ADD             X8, X25, X8
0x1D8EC                 BR              X8

```

它设计的还是很有实力的，我直接把分析的结构写出来吧，我是万万没想到它是这样做的

```
# 新 VM 的锚点寄存器 (VEX offset, 用寄存器身份而非具体基址值判定对象)
REG_X21 = _off("x21") # vreg 文件基址
REG_X26 = _off("x26") # bytecode 指针 (vpc)
REG_X25 = _off("x25") # handler 表基址 (dispatch, 不参与数据流)
REG_SP = _off("sp") # vm 栈Q

REG_X23 = _off("x23") # ctx 基址 (可选对象)
REG_X20 = _off("x20") # ctx 结构指针 (可选对象)

_ALIAS = {"lr": "x30", "fp": "x29"} # 这个是我trace的结构，别名什么的

```

扯远了，首先还是告诉各位它的 handler 切割特征，看看是怎么进行拆分的，可以看见结构是`add ... add ... br ... ldr`这样的结构，其中`ldr`是下一个的开始，当然`ldr`是语义不是顶死的汇编指令，理想很丰满现实很骨感啊，tm 的没办法确定它上下结构特征，我想的是写一个特征检测然后 bool 变量控制，但是实际上是没地方能控制，是 True 后一直是了，所以让模型理解我的意图以后它发力了，根据 x25 作为锚点，进行数据流转方向的切割，如果 br 之前是 x25 的操作那么就可以判定这个 br 是 handler 分发点，直接下刀。  
这个方案其实是把我那条死路走通的开关：用一个 "一触发就永久 True 的全局 bool" 是没办法兼顾那么多地方的，改成**寄存器级数据流追踪**，让每个寄存器的状态天然有作用域、在每个分发点自动复位，做法就是维护一张 `last_writer` 表，记录每个寄存器最后一次是被哪条指令写的。判断一个 `br xR` 是不是分发，就看 xR 上次是不是被 `add xR, x25, xR` 这个形态写出来的——x25 是 handler 表基址，`add xR, x25, xR` 干的就是 "字节码偏移量 + 表基址算下一个 handler 地址" 这件事，它写出来的寄存器拿来 br，那必然是链尾分发，没跑的。

```
def match(self):
    last_writer: dict[str, str] = {}
    for inst in self.parser.iter_atomic_objects():
        asm = self._asm(inst)
        is_dispatch = False
        if asm.startswith("br"):
            reg = asm[2:].split(",")[0].strip()
            if reg.startswith("x") and last_writer.get(reg) == f"add{reg},x25,{reg}":
                is_dispatch = True
        if is_dispatch:
            self.cur_handler.append(inst)
            self.handlers.append(self.cur_handler)
            self.cur_handler = []
            last_writer.clear()
        else:
            self.cur_handler.append(inst)
        for r in inst.get("write", {}):
            lr = r.lower()
            if lr.startswith("x") or lr.startswith("w"):
                last_writer[lr] = asm

```

几个点值得单独说一下：

*   **为什么用 last_writer 而不是看相邻两条**：复合 handler 里 `add xR, x25, xR` 和 `br xR` 之间完全可能夹着别的指令（比如中间 `bl` 调 helper 再 `ret` 回来接尾巴），硬套 "br 上面一条必须是 add x25" 的模板，形态一换就漏。写者表相当于给每个 br 配了个 "验明正身"：不管中间隔了几条，我只问 xR 上一次是谁写的、写的形态是不是 `add xR,x25,xR`。
*   **怎么解决 bool 卡死的问题**：之前那个全局 bool 一旦置 True 就永远复位不了。这里每个寄存器的写者是 "后来者覆盖前者" 的，天然有时效性；而且每次切段后 `last_writer.clear()`，状态的作用域被牢牢锁在每段 handler 的尾巴里，不会跨段泄漏。
*   **切段时机**：br 归入上一个 handler 的尾块——链式 VMP 里 br 就是那条 handler 的收尾指令，不属于下一个。
*   **尾巴处理**：trace 收尾没被 br 截断的残余会单独成段，最后 `self.handlers = self.handlers[1:-1]` 把头尾两段扔掉，剩下的就全是干净的 handler 段。

### 三、解剖 handler 与 IR 选型

跑完的效果就是开头那个 raw handler，一段一段切得贼齐：每段入口是 `ldrsh x8, [x26]` 这种取指令，中间是语义操作，末尾 `br x8` 收尾，跟手工分析的完全对得上。

这时候我们有的是一个个的 handler 块，需要使用一个东西了`angr`这个库，也是大名鼎鼎的库相信有的同志已经使用过了，我使用这个东西是想快速的汇编到 IR 的提升以及后续的算法还原，这条路上我踩坑不少的，慢慢来说，首先我用的 P-code 因为它非常死板，这里是真有点，我们在做还原期间就是需要死板的，不变的可以帮助我们对算法进行减负，否则算法需要巨大的精力去维护，P-code 是只吃机器码的，然后我用 keystone 来明文对汇编转化机器码，然后再给 miasm 去处理，miasm 其实已经有自己的 IR 了但是过于活跃了，所以没使用它，然后就是慢慢进行自己的路  
结果发现 angr 直接可以用 P-code(angr 维护的 pypcode)，然后打算直接 keystone+angr 了，但是发现 angr 对 P-code 的支持不如 VEX IR，测了 10 个 handler，每个 handler 进行 10 次 hash 计算，确定是稳定的以后就直接换 VEX IR 了，最后发现 angr 可以直接吃明文汇编，底层自动转化的所以直接 ks 也不需要了然后我们就得到了`angr的project、state`，接下来就到了算法部分了 (快手这个 handler 写的好烂啊，几乎找不到合适讲解的)，下面是我把上面的逻辑让 AI 生成的图，没招了 52 的 MD 不支持图显示，各位还是自己复制到本地看吧

```
graph TD
    subgraph Stage1_Fetch[阶段1: 字节码提取]
        A[字节码指针 X26] -->|LDRSH 取偏移1| B[X8 = 槽位索引1]
        A -->|LDRSH 取偏移2| C[X9 = 槽位索引2]
    end

    subgraph Stage2_VREG_Read[阶段2: 从VREG文件读取值]
        B -->|LDR X8, X21+X8| D[X8 = VREG槽内的偏移值]
        C -->|LDR X9, X21+X9| E[X9 = VREG槽内的偏移值]
        style X21 fill:#f9f,stroke:#333
    end

    subgraph Stage3_Execute[阶段3: 地址计算]
        D -->|ADD X8, X21, X8| F[X8 = 绝对地址对象1]
        style F fill:#ff9,stroke:#333
    end

    subgraph Stage4_Writeback[阶段4: 写回 CTX]
        F -->|STR X8, X23+X9| G[将绝对地址存入 CTX 结构体]
        E -.->|作为偏移基址| G
        style X23 fill:#9cf,stroke:#333
    end

    subgraph Stage5_Dispatch[阶段5: 更新VPC并派发]
        A -->|ADD X26, 0x18| H[X26 指向下一条字节码]
        A -->|LDR X8, X26+0x10| I[X8 = Handler表偏移量]
        I -->|ADD X8, X25, X8| J[计算绝对Handler地址]
        J -->|BR X8| K[跳转到下一指令执行]
        style X25 fill:#ccc,stroke:#333
    end

    Stage1_Fetch --> Stage2_VREG_Read --> Stage3_Execute --> Stage4_Writeback --> Stage5_Dispatch

```

这时候来简单说一下 VEX IR，`add rax, rbx`是原始汇编，虽然是 x86 的但是含义差一样，都是 rbx 加 rax 再复制到 rax 里面

```
IRSB {
    t0:Ity_I64 t1:Ity_I64 t2:Ity_I64 t3:Ity_I64 // 临时变量，为SSA辅助
    00 | ------ IMark(0x0, 3, 0) ------         // 每个IMark都是一条汇编
    01 | t2 = GET:I64(rax)                      // 获取RAX寄存器，VEX中GET会获取寄存器，在架构信息中的偏移
    02 | t1 = GET:I64(rbx)                      // 获取RBX寄存器
    03 | t0 = Add64(t2,t1)                      // 二者临时变量相加
    04 | PUT(cc_op) = 0x0000000000000004        //
    05 | PUT(cc_dep1) = t2
    06 | PUT(cc_dep2) = t1                      // 到这里三行都是为eflages服务的
    07 | PUT(rax) = t0                          // 赋值回RAX寄存器
    NEXT: PUT(rip) = 0x0000000000000003; Ijk_Boring // 下一条汇编执行
}

```

其中我们得到的这一块 handler 汇编转化成 VEX IR 是有很多很多内容的，在实际写东西时候遍历的就是每一行的 VEX IR 字段，然后构造算法，angr 给了一些比较方便使用的方法，期待自行探索

落地的时候有几个 VEX 特有的坑值得说一下：

*   **逐指令提升**：一个 IRSB 到 `bl` 就断了（call 出口），而 CALL 类 handler 的 helper 体要跟进去。所以每条指令单独 `pyvex.lift(4 字节)` 成一个 IRSB，DFG 靠寄存器 offset 跨指令穿起来，`bl/ret` 就不打断数据流了。
*   **IMark / Exit 跳过**：IMark 是地址标记，Exit 是 dispatch 的 `br` 尾巴，都不参与数据流。
*   **标志位模型**：`cmp` 产生 `PUT(cc_op)=3; PUT(cc_dep1); PUT(cc_dep2)`，`csel` 是 `Iex_CCall(arm64g_calculate_condition)` + `ITE` 读 cc。打包的条件码 = `(cond<<4)|cc_op`，比如 csel hi 就是 `(8<<4)|3 = 0x83`，可解码。这就是后面折 MAX/MIN/SLTU 的入口。
*   **寄存器按 offset 键**：`Get`/`Put` 用架构偏移，`w8` 和 `x8` 是同一个 offset，写 w 即定义 x 的 W/X 统一天然成立，不用自己管别名。

### 四、还原

其实我们要做的就是分析出来 Def-Use 链，然后进行活性分析，最终得到一条线

*   复制传播：精简中间值，直接得到不可约表达式
*   常量折叠：能算什么算什么
*   代数简化：消除无意义的数学游戏，取消一些无效执行

经过三步以后，我们就基本得到了一个不可约表达式，然后就可以进行代入实际值了

```
Add64( GET(x26), 0x8 ) // x26 = 0x7fff1000

```

代入这个值，然后再根据 trace 所记录的读写，就可以校验正确性了，随后就可以进行大规模测试了，当然也要自己定义一些自己的 IR

我们切出来的每个 handler 都是 VMP 中的一条指令所做的内容 (符合指令会多做几条)，然后我们根据它目标寄存器，使用不动点迭代的思维，最终肯定会得到一条线，以这条线(DU、活性、不动点) 为核心，再进行其中实际作用于 VMP 内容的分析(CTX、特殊寄存器)，然后得到这个指令所做的内容

理论到这就到头了，落地才是最折磨人的。我最后跑通的链路是这样的：

```
handler 汇编 --keystone--> bytes --pyvex--> VEX IR --build_dfg--> DFG
  --常量传播(propagate_values)--> 锚点过滤(reaches_anchor) --> emit_expr --> Expr 树
  --simplify 化简到不动点--> 语义识别渲染 --> 短 IR

```

#### 4.1 VEX -> DFG

VEX 的每条 statement 都能映射成一个图节点：`WrTmp` 是 temp 的定义、`RdTmp` 是引用、`Put`/`Get` 是寄存器写读、`Store` 是写内存根。寄存器一律按 offset 键，`Get` 的偏移和 `Put` 的偏移对上就是 def-use 边。

```
# DFG 节点 (对齐 lift_pypcode 的 Node)
class Node:
    op        # "INT_ADD"/"LOAD"/"STORE"/"CONST"/"INPUT"...
    ins       # 输入节点 id 列表
    out_off   # 寄存器写的话记 offset
    out_size  # 结果位宽 (字节)
    has_val   # 有没有 trace 具体值
    val       # 具体值

```

两条硬规则：

1.  **每条指令最后一个寄存器写挂 trace 值**（hasVal 标注）——这就是 "代入" 的原料。同时把值下放给本条指令创建的 LOAD 节点，否则 LOAD 自身不知道自己的值（值挂在包它的 `32Uto64` 上）。
2.  跑完一遍**常量传播**（`propagate_values`）：节点输入全是已知常量就算出值。opcode 解码链（`and x12,x9,#0xff` 这种抠字节的）就是在这步塌掉的。

#### 4.2 锚点过滤：哪些保留结构，哪些折叠成常量

这是整个还原的核心决策。锚点寄存器前面给过：**x21 = vreg 文件基址、x26 = vpc、x25 = handler 表、sp = vm 栈、x23/x20 = ctx**。分两类：

*   **x21 / sp / x23 / x20 = VM 状态**：以它们为基址的内存读写是 VM 自己的寄存器文件 / 栈 / 上下文，必须**保留符号**（`VM_REG[n]` / `VM_STACK[off]` / `CTX[off]`）。
*   **x26 = bytecode 指针**：从 bytecode 读出的操作数就是 "解码链"，**折叠成具体值**——这就是 "代入"。一个 handler 的语义应该由字节码操作数的具体值驱动，而不是留着 `BC_U64[0x8]` 这种符号。

实现就是递归判定子树能不能到达某个 VM 状态锚点的 INPUT 节点：

```
def reaches_anchor(ctx, nid):
    """子树是否经任意输入到达 VM 状态锚点 (x21/sp/x23/x20)"""
    n = ctx.nodes[nid]
    if n.op == "INPUT":
        return n.reg_off in _VM_STATE_OFF
    return any(reaches_anchor(ctx, i) for i in n.ins)

```

能到达 → 保留结构；到达不了但有具体值 → 塌成常量。这一刀切下去，`ldr x9, [x26, #0x8]` 读出的操作数立刻变成具体数，而 `ldr w8, [sp, #0x8]` 保持 `VM_STACK[0x8]`。

#### 4.3 内存对象分类

LOAD/STORE 的地址要归类成 VM 对象，识别 `INT_ADD(直连锚点, 偏移)` 形态：

<table><thead><tr><th>地址形态</th><th>对象</th></tr></thead><tbody><tr><td><code>x21 + idx</code></td><td><code>VM_REG[slot]</code></td></tr><tr><td><code>sp + imm</code></td><td><code>VM_STACK[off]</code></td></tr><tr><td><code>x23 + off</code> / <code>x20 + off</code></td><td><code>CTX[off]</code> / <code>CTX_S[off]</code></td></tr><tr><td>从栈 load 出的指针再解引用</td><td><code>*STACK_PTR</code></td></tr><tr><td><code>add R, x21, X</code> 作为值被存</td><td><code>&amp;vreg[...]</code>（绝对地址对象）</td></tr></tbody></table>

#### 4.4 化简到不动点

展开出来的原始表达式是一条很长的线，三步化简到不可约：

*   **常量折叠**：能算什么算什么
*   **恒等消元**：`x+0`→`x`、`x&全1`→`x`、`zext32(low32(x))`→`low32(x)`、`and(and(x,c1),c2)`→`and(x,c1&c2)`
*   **惯用式识别**：ALIGN / sext / MAX / MIN

举几个真实的（这些才是真正给这条 handler 定语义的东西）：

```
# ALIGN: (A+B-1) & -B, B 是字节码操作数的具体值
and32(add32(VM_STACK[0x8], #0x3), #0xfffffffc)   ->  align32 VM_STACK[0x8], #0x4

# byte 符号扩展: orr(and(asr(lsl(X,0x38),0x3f), -0x100), and(X,0xff))
->  sext8 X

# cmp + csel hi (VEX cc 模型: arm64g_calculate_condition + ITE)
->  MAX a, b

```

这里有个小坑：`add64(add64(A, B), 0xffffffff)` 是 64 位加，常量合并后变成 `add64(A, 0x100000003)`，64 位的 `c1+c2 == -1` 关系就断了——因为 0xffffffff 是 32 位的 -1 零扩展。所以 ALIGN 匹配要同时查 32 位掩码关系 `(c1+c2)&0xffffffff == 0xffffffff`。

化简要**迭代到不动点**，一遍不够。比如 `low32(x+c) = low32(x) + (c&0xffffffff)` 分配律要把 64 位对齐链拆成 32 位，拆完还要再折 add 常量，层层触发。

#### 4.5 数值校验

化简完代入具体值，和 trace 记录的写入值对账，`[✓]` 标记：

```
add     VM_STACK[0x8], VM_STACK[0x8], #0x80  // = 0x2e48  [✓]

```

#### 4.6 真实的输出

拿 0x1d218 这个 handler 举例（它在这个 trace 里重复执行了 9 次，每次字节码不同，正好能看代入效果）：

```
mov     VM_REG[0x90], VM_STACK[0x8]                       // = 0x2dc8  [✓]
add     VM_STACK[0x8], VM_STACK[0x8], #0x80               // = 0x2e48  [✓]
MAX     *STACK_PTR[+0x20], *STACK_PTR[+0x20], add32(VM_STACK[0x8], #0x80)  // = 0x2e48  [✓]

align32 VM_REG[0x98], VM_STACK[0x8], #0x4                 // = 0x2e48  [✓]
add     VM_STACK[0x8], align32(VM_STACK[0x8], #0x4), #0x10  // = 0x2e58  [✓]
MAX     *STACK_PTR[+0x20], *STACK_PTR[+0x20], add32(align32(VM_STACK[0x8], #0x4), #0x10)  // = 0x2e58  [✓]

align32 VM_REG[0xa0], VM_STACK[0x8], #0x8                 // = 0x2e58  [✓]
...

```

同一个 handler，字节码操作数一变（#0x80→#0x10→#0x8），vreg 目标槽跟着变（0x90→0x98→0xa0），算出来的值一条一条往下传（0x2dc8→0x2e48→0x2e58），这就是 "代入具体值驱动语义" 的最好证明。这个 handler 干的事本质是：**`vreg[dest] = 栈上运行值对齐；栈上运行值累加一个字节码量；再对栈上指针指向的槽做 max`**——一条复合指令被还原成了三行。

### 五、CALL 类 handler

还有一点，对于 CALL 类的 handler 处理，其中 CALL 类分三种

*   原生函数调用：`strncmp、strncat、memcmp`这类的，直接选择分析其中作用以及返回值的 DU 链，分析后与 trace 进行对校，也可以解析一下 elf 文件的符号表，写上也更展示内容精细程度
*   CALL 里 handler：这就是真正的调用自己的逻辑函数了，这种被切割出来的 handler 会巨长，也是自行分析一下，然后根据 AArch64 的指令集定长，折叠地址即可，原始 trace 可原始输出或者拿 ida 反编译出来的内容
*   杂乱无章的：这类一般就是编译器傻逼了，开始瞎写了，乱插 BL 跳转，这其实在 trace 展开后不会影响什么内容，并且我们还是根据数据流转进行分析的，如果有 BUG 需要自己分析和解决了

落地时 CALL 标注我做得很轻：扫 handler 里的 `bl`/`blr`，**helper 入口地址直接取 bl 后第一条指令的 off**——trace 会跟进 helper 体，所以下一条就是 helper 实际执行的地址。然后尝试用 ELF 的 `.dynsym` 解析符号，内部静态函数解析不到就保留地址。

```
def detect_calls(insts):
    """bl/blr -> [(bl 地址, helper 入口地址)]; 入口 = bl 后第一条指令的 off"""
    calls = []
    for i, inst in enumerate(insts):
        mnem = inst["asm"].strip().lower()
        if mnem.startswith("bl") and i + 1 < len(insts):
            calls.append((inst["off"], int(insts[i+1]["off"], 16)))
    return calls

```

效果：

```
// call @0xe63c
mov     VM_REG[0x70], CTX[0x20e8]  // = 0x2128  [✓]

```

这条的语义：handler 读 vreg 值、算 `ctx + (vreg[A]+B)`、`bl` 调 helper，helper 里 `ldr x0, [x0]; ret` 把 `*(ctx+off)` 返回，存回 vreg。DFG 因为逐指令提升穿过了 `bl`，返回值 x0 的 def-use 直接连到 helper 体里的那条 load，所以还原出的就是 `VM_REG[0x70] = CTX[0x20e8]`——helper 的语义被 DFG 自动 "内联" 了。

复合 CALL handler 一次能标出好几个 helper：

```
// call @0xb97c
// call @0xe114
// call @0x4210
// call @0xe1ac
// call @0xe150
// call @0x220b4

```

### 六、结尾

全量 25558 段跑下来的总账：

```
sink 24321 行 | 可校验 24055 (✓ 24055 / ✗ 0) | 无值可校验 266

```

**凡是有 trace 值能代入校验的，全部正确，零错误。** 266 条无值是因为 trace 没记录表达式里某个 load 的值（vreg / 栈读为主），不是引擎算错。

已实现的语义面：`mov/add/and/orr/eor/sub/mul/divs/lsl/lsr/asr/not/MAX/MIN/align32/sext8/sext16/sext32`，内存对象 `VM_REG/VM_STACK/CTX/CTX_S/*STACK_PTR/&obj[...]`，CALL 标注 `// call @0x...`。

**已知缺口**（给想继续的同志指路）：

1.  **复合分配器 handler**：`TPIDR_EL0` 取 TLS、free-list 弹出这种，helper 里的指针解引用落在锚点区域外，产出 `MEM[0x0]`——这是博客 § 五的 "杂乱无章" 类，超出锚点分析范围。
2.  **计算偏移的 ctx 写**：`ctx[vreg[A]+B] = byte` 这类，偏移是运行时算出来的，现在产出的是具体化偏移（`CTX[0x102de9]`），符号形式 `ctx[vreg[A]+B]` 丢了。要捡回来的话，sink 的地址侧也得走一遍反向污点。
3.  **无值 sink**：trace 没记 load 值的表达式没法对账，只能结构上保证。

这套东西前后踩的坑：`e.op` 带 `Iop_` 前缀、`arm64g_calculate_condition` 是 `Iex_CCall` 不是 Qop（helper 名在 `.cee.name`）、VEX 掩码常量是无符号的（`0xffffffffffffff00` 不是 `-0x100`）、`asr` 求值要算术右移不是逻辑右移。都记在这了，少走点弯路。

代码在 [GitHub: djskncxm/DuckReVM](https://github.com/djskncxm/DuckReVM)（`src/interpreter.py` 是 VEX→DFG→ 短 IR 的主管线，`analyze.py` 是驱动）。  
我感觉还原出来没达到我的预期 (没心劲了)，不过就这样吧，下一步会尝试一下六神或者虾皮那种分发器的 VMP