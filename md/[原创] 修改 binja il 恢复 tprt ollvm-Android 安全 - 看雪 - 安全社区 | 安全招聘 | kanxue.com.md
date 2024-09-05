> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283288.htm#msg_header_h2_2)

> [原创] 修改 binja il 恢复 tprt ollvm

预热 (记录的碎碎念)
-----------

项目地址 : [https://github.com/zhuzhu-Top/deobf](https://github.com/zhuzhu-Top/deobf) (配合源码食用更佳)

代码没有经过大量的测试, 基本只覆盖了 0x9cbe0 函数

但是方法是不变的, 代码也没有整理, 无关的代码可以忽略

### 参考文章 (放到开头就是先建议阅读再参考此文)

使用 BinaryNinja 去除 libtprt.so 的混淆 (一) [核心参考]  
[https://bbs.kanxue.com/thread-282826.htm](https://bbs.kanxue.com/thread-282826.htm)

使用 BinaryNinja 去除 libtprt.so 的混淆 (二) [核心参考]  
[https://bbs.kanxue.com/thread-282918.htm](https://bbs.kanxue.com/thread-282918.htm)

[1]ollvm 控制流混淆的 angr 还原的原理

> Angr Control Flow Deobfuscation [https://research.openanalysis.net/angr/symbolic%20execution/deobfuscation/research/2022/03/26/angr_notes.html](https://research.openanalysis.net/angr/symbolic%20execution/deobfuscation/research/2022/03/26/angr_notes.html)

[2] 对本文没用, 只是讲解了 angr 的 reaching definition engine

> A reaching definition engine for binary analysis built-in in angr [https://degrigis.github.io/posts/angr_rd/](https://degrigis.github.io/posts/angr_rd/)

上面文章中提及了一个人写的污点分析实现上面文章最终实现的相同的功能 ([https://github.com/badnack/angr_taint_engine), 还没有阅读这个代码 (以后看污点分析的时候可以尝试看一下)](https://github.com/badnack/angr_taint_engine),%E8%BF%98%E6%B2%A1%E6%9C%89%E9%98%85%E8%AF%BB%E8%BF%99%E4%B8%AA%E4%BB%A3%E7%A0%81(%E4%BB%A5%E5%90%8E%E7%9C%8B%E6%B1%A1%E7%82%B9%E5%88%86%E6%9E%90%E7%9A%84%E6%97%B6%E5%80%99%E5%8F%AF%E4%BB%A5%E5%B0%9D%E8%AF%95%E7%9C%8B%E4%B8%80%E4%B8%8B))

[3] 污点分析 原理讲解 以及如何实现污点分析

> [https://github.com/firmianay/CTF-All-In-One/blob/master/doc/5.5_taint_analysis.md](https://github.com/firmianay/CTF-All-In-One/blob/master/doc/5.5_taint_analysis.md)

[4] 里面讲了还原 数据流混淆 (没看完, 没耐心了)

> Greybox Program Synthesis: A New Approach to Attack Dataflow Obfuscation [https://www.youtube.com/watch?v=eJo74i7nxtk](https://www.youtube.com/watch?v=eJo74i7nxtk)

视频配套代码 [https://github.com/quarkslab/qsynthesis](https://github.com/quarkslab/qsynthesis)

[5] 可以参考的还原 ollvm 的代码, 感觉不是很好,(里面用跳转到分发器的块就是真实块, 我觉得不是很好, 代码作为参考还是不错的)

> [https://github.com/cq674350529/deflat](https://github.com/cq674350529/deflat)

[6] 忘记什么时候找到的 Deobfuscate OLLVM Bogus Control Flow with angr

> [https://github.com/bluesadi/debogus/tree/main](https://github.com/bluesadi/debogus/tree/main)  
> 里面有一段代码

```
proj.hook(next_func_addr, angr.SIM_PROCEDURES["stubs"]["ReturnUnconstrained"](), replace=True)
```

这个是在 angr 符号执行的时候遇到 函数调用, 不是继续分析函数内部, 而是直接返回无约束的返回值  
可以用下面代码替换

```
state.options.add(angr.options.CALLLESS)
```

直接在 nagr state 上 添加 这个 option 更便捷

[7] 利用 angr 符号执行去除虚假控制流 (学习了里面去除虚假控制流 以及 angr 的用法)

> [https://bbs.kanxue.com/thread-266005.htm#msg_header_h1_1](https://bbs.kanxue.com/thread-266005.htm#msg_header_h1_1)

[8] Angr 使用技巧速通笔记 (参考了 angr 的用法)

> [https://bbs.kanxue.com/thread-276834.htm](https://bbs.kanxue.com/thread-276834.htm)

> [https://bbs.kanxue.com/thread-276860.htm#msg_header_h2_0](https://bbs.kanxue.com/thread-276860.htm#msg_header_h2_0)

正文
--

我也是把还原的过程分成两步

一: 还原间接跳转

二: 还原真实块的后继块

为什么这么分呢? 因为如果能完成第一步, 第二步可以说是水到渠成 (后面会详细解释原因), 每一个真实块后面只可能两种情况, 要么直接跳转到下一个真实块 (包括直接返回), 要么就是直接根据条件, 跳转到 true 或者 false 分支, 在常规情况下还原这个步骤是比较难的 (要 patch 汇编, 我懒狗 + 菜狗, 不想这么操作)

但是, 当我了解到 binary ninja (后面都用 binja) 的 workflow 可以直接修改 il (中间表示) 的时候, patch 的问题就变得没有难度, 修改 il 想加几行就加几行, 没有 patch 汇编那种限制 (0xEEEE 大佬的代码 我根本没读完, 我这种菜鸟, 看了 patch 汇编, 一个头两个大)

结束闲扯

### 还原间接跳转

在 0xEEEE 大佬的文章中 用到了 "cmp 下沉" , 这个思想非常关键 (建议反复阅读来理解), 我主要是也是 "cmp 下沉", 我用这个方法的主要是为了简化 il 的代码, 来一段代码解释一下 (本文主要围绕 9cbe0 做出解释):

![](https://bbs.kanxue.com/upload/attach/202409/878476_D9PUQDJ86GPJ2C3.png)  
这里根据 w12 和 w23 作为判断条件给 x11 赋值

```
59 @ 0009cc68  x13 = [x20 + x11].q
60 @ 0009cc6c  if (cond:1) then 61 else 63
```

紧接着就是借助 x11 的值计算 x13

```
66 @ 0009cc74  x12 = x13 + x21
```

x13 算出 x12

```
73 @ 0009cc7c  jump(x12 => 74 @ &BN_CODE_start_0x9cc80_size_0x98, 77 @ 0x9cc8c)
```

最终得出 x12 的值  
观察计算过程和整个函数得知 除了最上面的 x11 根据条件得到的结果不同外, 中间计算过程, 是完全一样样了, 中间计算过程的值是不会改变的 (x20,x21), 我们就可以推出 br x12 的跳转目标主要受 x11 的值的影响, x11 主要是 0x9cc64 处的判断 决定

再来看一个真实块前面的一个块

![](https://bbs.kanxue.com/upload/attach/202409/878476_F4D8CS3XZGXEA95.png)

0x9ccc8 开头的地址就是一个真实块 , 关键上面那个块, 根据 jump(x14) 来决定是否能够跳进 下面的真实块, 决定 x14 的关键又是 x9, x9 的值又是由于 0x9cc58 的比较来决定的  
最终我们就可以把代码替换一下, 让 能否跳进 真实块的 br reg 变成 , if(条件) 真实块 else 返回分发器

也就是把

```
86 @ 0009cca0  jump(x14 => 73 @ 0x9cc7c, 94 @ 0x9ccc8)
```

变成

```
if (w12 s< w22){
    真实块
}else{
    返回重新分发(这个原来失败跳转到哪里,修改之后还是跳转到原来的地方)
}
```

> 这里我做了两个关键步骤 1. 比较条件下移, 2jump reg 变成条件跳转 (后续会用 修改 binja il 的方式实现)

这么做的好处是, 我们可以去掉中间繁琐的跳转目标地址计算过程, x12 的值在分发到目标真实块之前是不会改变的, w22 的值也是固定的

中间跳转的计算过程我们就不需要了, 全部都可以 nop 掉, 具体要 nop 哪些地址, 可以借助 binja il 的 ssa, 可以快速的找到变量的定义位置

![](https://bbs.kanxue.com/upload/attach/202409/878476_BHNMWFTKEH5623W.png)

从图上面的 advanced il forms 里面找到 Medium Level Il (SSA Form)

鼠标点击 x14_4#7 就会告诉我们这个变量定义于 0x9cc90 的位置 (自行尝试别的变量)

这个过程转换成代码就是 拿到这个变量, 然后获取变量的 def_site , 然后一路遍历, 就可以拿到这一串计算过程中得到的中间变量, 全部 nop 掉

下面是源码中如何查找的过程

```
"""
找到 中间计算过程中所有 相关用到的变量
"""
def find_all_relative_var(jump_var: SSAVariable ,info):
    def_il = jump_var.def_site
    for var in def_il.vars_read:
        if info.get("var_read") is None:
            info["var_read"] = [var]
        else:
            info["var_read"].append(var)
        find_all_relative_var(var,info)
```

把上面修改的过程全部完成, 我们就得到了下面的代码

![](https://bbs.kanxue.com/upload/attach/202409/878476_HZVVA6ZFGMJ7E9F.png)

可以看到中间那一大坨分发全部 nop 了, 下面的 br reg 也改成了条件跳转

既然进入真实块之前的内容我们的都处理好了, 下一步就是如何知道 真实块 的 后继块

### 还原真实块的后继块

> 这部分没看 0xEEEE 大佬怎么做的, 大佬代码里面很多都是关于怎么 patch 的, 头大, 看不下去

```
0009cbe0  {
0009cc18      int64_t x8 = 0;
0009cc50  label_9cc50:
0009cc50      int32_t i = -0x5ce3d627;
0009cc50     
0009cc88      while (true)
0009cc88      {
0009cc88          int64_t var_68;
0009cc88         
0009cc88          if (i == 0xa31c29d9)
0009cc88          {
0009ccb0              var_68 = x8;
0009ccb0             
0009ccc0              if (var_68 < *(uint64_t*)((char*)arg2 + 0x10))
0009ccc0                  i = 0x41d706b8;
0009ccc0              else
0009ccc0                  i = 0x5856f79f;
0009cc88          }
0009cc88          else
0009cc88          {
0009cc7c              while (i >= 0x41d706b8)  // zhuzhu_com
0009cc7c              {
0009cca0                  if (i == 0x41d706b8)
0009cca0                  {
0009ccdc                      sub_9ceb8((arg1 + ((uint64_t)*(uint32_t*)(*(uint64_t*)((char*)arg2 + 8) + (var_68 << 2)))));
0009ccf0                      x8 = (var_68 + 1);
0009ccf4                      goto label_9cc50;
0009cca0                  }
0009cca0                 
0009ccac                  if (i == 0x5856f79f)
0009cd14                      return;
0009cc7c              }  // zhuzhu_com
0009cc88          }
0009cc88      }
0009cbe0  }
```

有了前面的基础, 我们得到了这样的伪代码,  
分析 0009ccc0 代码的意图, 根据判断条件设置 i 的值为 0x41d706b8

```
0009cca0                  if (i == 0x41d706b8)
0009cca0                  {
0009ccdc                      sub_9ceb8((arg1 + ((uint64_t)*(uint32_t*)(*(uint64_t*)((char*)arg2 + 8) + (var_68 << 2)))));
0009ccf0                      x8 = (var_68 + 1);
0009ccf4                      goto label_9cc50;
0009cca0                  }
```

i 的值为 0x5856f79f

```
0009ccac                  if (i == 0x5856f79f)
0009cd14                      return;
```

这个代码目的就变得显而易见了, 我们只需要在设置 i 的值的代码变成对应的代码, 我们就实现了这个控制流的恢复

变成这样

```
0009ccc0              if (var_68 < *(uint64_t*)((char*)arg2 + 0x10))
0009ccdc                      sub_9ceb8((arg1 + ((uint64_t)*(uint32_t*)(*(uint64_t*)((char*)arg2 + 8) + (var_68 << 2)))));
0009ccf0                      x8 = (var_68 + 1);
0009ccf4                      goto label_9cc50;
0009ccc0              else
0009cd14                      return;
```

这样就可以接近代码要表达的意思, 如果汇编的 patch 来实现, 用脚趾头想一下都麻烦, 有了 binja, 情况就完全不一样了

```
def if_expr(self, operand: ExpressionIndex, t: LowLevelILLabel, f: LowLevelILLabel) -> ExpressionIndex:
    """
    ``if_expr`` returns the ``if`` expression which depending on condition ``operand`` jumps to the LowLevelILLabel
    ``t`` when the condition expression ``operand`` is non-zero and ``f`` when it's zero.
 
    :param ExpressionIndex operand: comparison expression to evaluate.
    :param LowLevelILLabel t: Label for the true branch
    :param LowLevelILLabel f: Label for the false branch
    :return: the ExpressionIndex for the if expression
    :rtype: ExpressionIndex
    """
    return ExpressionIndex(core.BNLowLevelILIf(self.handle, operand, t.handle, f.handle))
```

这是 binja if il 的创建, t: LowLevelILLabel 是对应的 true 的 inst_id,f: LowLevelILLabel 是 false 的

```
73 @ 0009cc7c  if (w12 s< w23)
```

inst_id 就是在界面里面看到的, 比如这个 73, 修改 il, 我们如果知道 inst_id, 想要 il 跳到哪就跳哪里

既然已经知道了怎么修改 il, 还差两个最核心的问题

一 : 我怎么知道哪个 if 是要修改的?(真实代码里面肯定也有 if)

> 在汇编里面是找的 csel 之类的指令, 如果是给 跳转变量赋值就是了, 在 il 里面, 完全可以判断 if 的 成功之后的块和 失败之后的块是否对 跳转变量做了赋值, 如果赋值了, 我们就需要对这个 if 行修改

二 : 怎么知道跳转目标块的 inst_id?(修改 if 的时候需要)

> 根据上面手动还原 if 代码的过程, 省略了两个问题, 认为了 i= 0x5856f79f 就是 对应 return,i = 0x41d706b8 就是对应

```
0009ccdc                      sub_9ceb8((arg1 + ((uint64_t)*(uint32_t*)(*(uint64_t*)((char*)arg2 + 8) + (var_68 << 2)))));
0009ccf0                      x8 = (var_68 + 1);
0009ccf4                      goto label_9cc50;
```

因此需要建立一个这样的字典, 来 记录跳转的目标地址

```
{
    "0x41d706b8":0009ccc8
    "0x5856f79f":0x9ccf8
}
```

但是我实际搞出来是这样的

```
"642172":{ # 0x9CC7C
    "cmp_addr":642148,
    "true_addr":642176,
    "false_addr":642188
  },
  "642184":{
    "cmp_addr":642168,
    "true_addr":642224,
    "false_addr":642172
  },
  "642220":{ # 0x9CCAC
    "cmp_addr":642140,
    "true_addr":642296,
    "false_addr":642172
  },
```

> 具体的参考 我开源项目的 data.json 文件, 里面是生成好的, workflow 修改 il 的时候需要的所有数据  
> 0x9CCAC 就是需要修改 br reg 为 if 的地址 (作为字典的 key*),cmp_addr 就是需要被下移的比较的条件, true_addr 和 false_addr 当然就是 成功和失败后的地址

要连接所有真实的块, 就要 解决从 哪里来 和 到哪里去 的问题, 有了前面内容就得出, 如果想要跳转到下一个块, 就会给跳转变量赋值到 下一个快对应的一个很大的数, 那怎么找到这个对应的块? cmp 下移之后, 出现的这样的代码

```
0009ccac                  if (i == 0x5856f79f)
0009cd14                      return;
```

只需要遍历找到所有 if, 然后判断是否是 var == num 的情况, 这种就可以做以一个记录, 下次别的地方跳转的时候, 就知道那个 数字对应的块的地址了

#### cmp_addr (解决从哪里来的问题)

这个是根据 br reg 的对应的变量一路 def_site 找到最上面就是一个 if

#### true_addr & false_addr (解决到哪里去的问题)

这两个地址, 我这里给出两种获取方法,

1.  借助 binja 的 parse_expression  
    parse_expression 可以执行一些表达式  
    例如:

```
>>> bv.parse_expression("1+1")
>>> bv.parse_expression("3+1")
```

那就可以借用这个来计算 跳转的目标地址

```
87 @ 0009cca4  x14 = [x20 + (x10 << 3)].q
88 @ 0009cca8  x14 = x14 + x21
89 @ 0009ccac  jump(x14 => 73 @ 0x9cc7c, 107 @ 0x9ccf8)
```

0009cca4 处 的指令 x14 = [x20 + (x10 << 3)].q

除了 x10 变动外, 别的都是不变的  
表达式值就可以变成

```
[0x1ddc80 + (x10 << 3)].q
```

再根据 il 得知 ,x10 = 2,7  
就可以得到两个表达式

```
[0x1ddc80 + (2 << 3)].q
[0x1ddc80 + (2 << 7)].q
```

分别用 parse_expression 执行 可以得到两个 x14 的值

```
>>> bv.parse_expression("[0x1ddc80 + (2 << 3)].q")
>>> bv.parse_expression("[0x1ddc80 + (2 << 7)].q")
```

我就是通过这样的迭代, 获取到了 br reg 修改成条件跳转之后 (if) 可以跳转到哪个是 true 对应的块和 false 对应的块, 真实块之后会再次设置跳转变量的值, 跳转到下一个块, 前面已经提及了, 跳转变量被设置的值对应的块的查找方法 就是找 if(br_var = 0xxxxx), 这个就是对应的块

1.  符号执行获取执行流

在参考文章中, 别人是直接 通过 simulation_manager 去 step 遍历拿到的, 解释下原文中两个主要步骤  
找打分发器的 state

```
import angr
 
proj = angr.Project("/tmp/pandora_dump_SCY.bin", load_options={'auto_load_libs': False})
 
def get_dispatcher_state(function, dispatcher):
    state = proj.factory.call_state(addr=function)
 
    # Ignore function calls
    # https://github.com/angr/angr/issues/723
    state.options.add(angr.options.CALLLESS)
 
    simgr = proj.factory.simulation_manager(state)
 
    # Find the dispatcher
    while True:
        simgr.step()
        assert len(simgr.active) == 1
        state = simgr.active[0]
        if state.addr == dispatcher:
            return state.copy()
 
addr_main = 0x7FF6C4B066F0
addr_dispatcher = 0x7ff6c4b067f0
dispatcher_state = get_dispatcher_state(function=addr_main, dispatcher=addr_dispatcher)
print(f"Dispatcher state: {dispatcher_state}")
initial_state = dispatcher_state.solver.eval_one(dispatcher_state.regs.eax)
print(f"Initial eax: {hex(initial_state)}")
```

这里是通过 step 然后一直找到 state.addr == dispatcher,

但是我当前样本不适用这种方法, 在 step 到分发器的时候, simulation_manager 就已经停在了分发器虾下面的一个块了  
文章中需要 分发器的 state, 是为了下一次可以直接改变 跳转变量的值, 然后符号执行, 就会自己找到对应的块去了

```
def get_dispatcher_state(function):
    state = project.factory.call_state(addr=function)
    # Ignore function calls
    # https://github.com/angr/angr/issues/723
    state.options.add(angr.options.CALLLESS)
    simgr = project.factory.simulation_manager(state)
    # Find the dispatcher
    # while True:
    #     simgr.step()
    #     assert len(simgr.active) == 1
    #     state = simgr.active[0]
        # if state.addr == dispatcher:
        # return state.copy()
    # 直接使用上面的 strp无法停在分发器的开始位置,所以不能按照文章上的写
    simgr.explore(find=dispatcher_addr)
    if simgr.found:
        _init = simgr.found[0]
        return _init.copy()
    else:
        raise Exception("未找到分发器")
```

这个是我的修改版, 直接 用 explore 停在分发器的第一行汇编出, 一定是第一行汇编的地址 (不然后面找后继块的时候是找不到的)

继续解释文章代码

```
def find_successors(state_value, dispatcher):
    state = dispatcher_state.copy()
    state.regs.eax = state.solver.BVV(state_value, 32)
    simgr = proj.factory.simulation_manager(state)
    while True:
        print(f"eax: {simgr.active[0].regs.eax}")
        print(f"Stepping: {simgr.active} ...")
        simgr.step()
        # TODO: the block before the dispatcher is wrong, we need the first non-dispatcher block
        if len(simgr.active) == 0:
            return state, []
        assert len(simgr.active) == 1
 
        state2 = simgr.active[0]
        print(f"  Only found a single sucessor: {hex(state2.addr)}")
        if state2.addr == dispatcher:
            print(f"  Dispatcher, eax: {state2.regs.eax}")
            # TODO: figure out where these potential values are set
            solutions = state2.solver.eval_upto(state2.regs.eax, 2)  # TODO: might need more potential states
            return state, solutions
        elif state2.addr == 0x7ff6c4b070ea:  # HACK: int3 here, no idea how to properly handle it
            return state, []
        state = state2
 
from queue import Queue
 
# state_value => real basic block state
states = {}
 
q = Queue()
q.put(initial_state)
 
while not q.empty():
    state_value = q.get()
    # Skip visited states
    if state_value in states:
        continue
    bb_state, successors = find_successors(state_value, addr_dispatcher)
    print(f"{hex(state_value)} {bb_state} => {[hex(n) for n in successors]}")
    print()
    states[state_value] = bb_state, successors
    for state_value in successors:
        q.put(state_value)
 
dot = "digraph CFG {\n"
for state_value in states.keys():
    _, succ = states[state_value]
    for s in succ:
        dot += f"\"{hex(state_value)}\" -> \"{hex(s)}\"\n"
dot += "}"
print(dot)
```

构造了一个 Queue, 然后遍历去找后继, state_value 就是 跳转变量被设置的那个很大的数字, 找到的条件是, 再一次 step 到 分发器, 因为在当前真实块到下一个块的过程中, 在 跳转变量被设置之后, 都不会被改变

> 这种做法可能存在问题, 如果真实块结束之后不是里面跳转到分发快的话, 那个保存的 state 就是错误的了

关键代码

```
solutions = state2.solver.eval_upto(state2.regs.eax, 2)
```

真实块跳转到分发器重新分发的时候, 用 solver 计算得到最多两个 solutions, 在进入到到 step 之前,

state = dispatcher_state.copy()

state.regs.eax = state.solver.BVV(state_value, 32)

初始状态是 分发器的状态, 然后符号化 跳转变量, 这样最后再次符号执行到分发器, 就可以求解得到最多两种中新的 跳转变量的值了

这里给出我的代码

```
def find_successors(state_value, dispatcher,debug = False):
    state = dispatcher_state.copy()
    setattr(state.regs,state_register_name,state.solver.BVV(state_value, 64))
    simgr = project.factory.simulation_manager(state)
    # 必须先step一下 不然就直接结束了
    new_sm = simgr.step()
    back_state = new_sm.active[0]
 
    if debug and state_value == 0xf3498be1:
        print(hex(state.addr)," -> ",hex(back_state.addr))
        while True:
            simgr.step()
            find_state = simgr.active[0]
            print(hex(simgr.active[0].addr))
            pass
    new_sm.explore(find=dispatcher)
    if new_sm.found:
        # [可能1] 直接跳转到下一个块
        # [可能2] 这个block下面有两个后继可以选择
        # 所以 eval_upto 第二个参数是 2
        found_state = new_sm.found[0]
        solutions = found_state.solver.eval_upto(getattr(found_state.regs,state_register_name), 2)
        return found_state, solutions
    else:
        # 不跳转到分发器 可能是 ret
        def find_ret(_state: angr.SimState):
            return _state.history.jumpkind == JumpKind.Ret
 
        _ret_sm = project.factory.simulation_manager(back_state)
        _ret_sm.explore(find=find_ret)
        if _ret_sm.found:
            found_state = _ret_sm.found[0]
            return found_state, []
        else:
            raise Exception("未处理异常,可能不是ret的情况")
```

只是把 step 换成了 explore , 因为当前样本无法直接停在 分发器

得到这样的输出

```
跳转变量的值    对应的块地址  真实块结束后 下一个块对应的 跳转变量的值
0x9ff58171  ==> 0x9d088     ['0xf3498be1', '0xb74983fc']
0xf3498be1  ==> 0x9d164     ['0x9d23c004', '0x33138a0a']
0xb74983fc  ==> 0x49d274    [ret]
0x9d23c004  ==> 0x9d11c     ['0x33138a0a', '0x948df64d']
0x33138a0a  ==> 0x9d0f0     ['0x2671310f', '0xc07e49a3']
0x948df64d  ==> 0x9cfd4     ['0x67817258', '0xc07e49a3']
0x2671310f  ==> 0x9d204     ['0xc07e49a3', '0xee0fcd5f']
0xc07e49a3  ==> 0x9cf74     ['0x9ff58171']
0x67817258  ==> 0x9d248     ['0xc07e49a3']
0xee0fcd5f  ==> 0x9d038     ['0xc07e49a3', '0x5f1f066d']
0x5f1f066d  ==> 0x9d24c     ['0xc07e49a3']
```

有了这些信息, 就可以 生成一个跟 binja 得到的 data.json 一样的数据文件, 随之 修改 il, 我代码里面只有 binja parse_expression 那种实现的全部代码, 符号执行我只是 进行了 可行性的探索

binja 生成 data.json 在 项目的 header_less.py

符号执行代码在项目的 symbolic_.py

补充内容
----

大佬告诉我 污点分析 也可以干掉 ollvm, 我学了一下, 但是没什么主意去搞 (不知道怎么写, 没参考)  
在线 查看 triton 生成的 dot 图  
[https://dreampuf.github.io/GraphvizOnline](https://dreampuf.github.io/GraphvizOnline)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#脱壳反混淆](forum-161-1-122.htm)