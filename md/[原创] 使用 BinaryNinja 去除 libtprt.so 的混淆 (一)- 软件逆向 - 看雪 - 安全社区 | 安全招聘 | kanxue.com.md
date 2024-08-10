> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282826.htm)

> [原创] 使用 BinaryNinja 去除 libtprt.so 的混淆 (一)

使用 BinaryNinja 去除 libtprt.so 的混淆 (一)
====================================

> 文章中的思路只是个人想法, 并不是最优解, 如有错误还望斧正.  
> 插件代码 github: [detx](https://github.com/EEEEhex/detx) [https://github.com/EEEEhex/detx](https://github.com/EEEEhex/detx)

版本: speedmobile_1.45.0.53757.apk 中的 libtprt.so

文章将分享去除 [寄存器间接跳转] 与[魔改控制流平坦化]混淆的思路, 并编写去混淆插件代码

0. 混淆类型
-------

libtprt.so 中的混淆大体分为三种类型:

*   魔改的控制流平坦化
*   寄存器间接跳转
*   无效循环

以及这三种的穿插混合, 这些混淆要么是获取信息麻烦, 要么是 Patch 起来麻烦, 总之就是很麻烦

本文将先分享去除 [寄存器间接跳转] 混淆的思路, 主要是 Patch 思路

1. 寄存器间接跳转混淆
------------

### 1.1 原理

其实就是跳转地址是计算出来的, 如下图所示:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_R4EDJDXQR62RAVB.png)  
这种混淆就是把原先的逻辑跳转改为了 jmp(var2)  
其中 var2 = mem[var1 (<< num)] + const 这些值其实都是可以确定的, 即:

```
//-----------------
if (Cond)
  jmp(true_addr)
else
  jmp(false_addr)
//-----------------
变为了->
//-----------------
if (Cond)
  var1 = 0;
else
  var1 = 1;
var2 = data_1fd630[var1];
var3 = var2 - 0x7218df2;
jump(var3);
//-----------------
```

通过 cond 设置偏移 var1, 然后从跳转表 data_1fd630 中拿出 var1 偏移处的值, 然后 +/- 一个常量就得到真正的跳转地址了

### 1.2 获取跳转地址思路

我的思路是静态分析 + 模拟执行:

1.  从 BinaryNinja 的 mlil ssa 层面, 可以获取到 jump 变量 var 的指令
2.  然后层层向上找, 找到所有涉及到的 mlil 指令 (就如上图中所有红框中的指令),
3.  然后拿到这些 mlil 指令对应的汇编指令去模拟执行就可以得到跳转寄存器的值.  
    3.1. 其中在汇编层面是通过条件选择指令 (csel, cset, cinc 等) 来改变值导致最终跳转地址的变化(就是上面红框中 x11#1=8 和 x11#2=0x30)  
    3.2. 因此可将条件选择指令改为 mov 等指令直接赋值, 模拟执行两次来分别获取 if 和 else 的真实跳转地址

例如一次混淆涉及到的如下指令:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_DYWM94C7MM48MJE.png)  
具体来说就是:

1.  首先要识别出一次混淆涉及到的所有汇编指令, 就是上图中所有红框的汇编指令
2.  识别出其中可以改变跳转地址的那条指令 (cset/csinc 等), 本次混淆就是 csel x11, x28, x27
3.  将 csel x11, x28, x27 分别改为 "mov x11, x28" 或 "mov x11, x27" 然后模拟执行从而获取两个跳转地址

**问:** 可以直接模拟执行 br 之前的全部指令, 不去识别一次混淆涉及到的指令吗?  
**答:** 可以是可以, 但这样会涉及到非混淆的真实指令, 我感觉处理起这种情况来不比去识别混淆指令简单.

### 1.3 Patch 思路 *

假设现在已经知道了两个跳转地址是多少, 怎么去 Patch 呢?  
我们的 Patch 不能改变了原始的逻辑, 比如说:  
**问:** 可以把 "csel x11, x28, x27" 改为 "b.lt t_addr", 把 "br x12" 改为 "b f_addr" 这样 patch 吗?  
**答:** 不可以, 因为原始逻辑是在 csel 之后还执行了 "0x9cc6c 0x9cc70 0x9cc78" 这些指令, 如果从 0x9cc64 处就改成 "b.lt" 跳转, 那逻辑就不对了, 原逻辑中 br 之前执行的指令就少执行了一部分.  
.  
**问:** 可以上移指令 (因为混淆指令是无效的可以随便覆盖), 然后在末尾插入 "b.lt + b" 指令吗?  
**答:** 不可以, 比如说 0x9cc74 处的指令属于混淆指令, 是无效的, 将其改为:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_DCHUXYJG2GSHW25.png)  
就是把 csel ... br 中间的指令全部上移覆盖上一个指令, 在末尾多出一个指令的空间, 但这样会出现一个问题, 原逻辑中是:

```
0x9cc60 cmp w12, w23 
....... 改变跳转寄存器x12
0x9cc7c br x12
```

这样 Patch 之后就变成了:

```
0x9cc60 cmp w12, w23
....... ............
0x9cc70 cmp w12, w15
....... ............
0x9cc78 b.lt 满足条件地址
0x9cc7c b 不满足条件地址
```

条件判断被覆盖了, 原本逻辑是判断的 "cmp w12, w23" 这样一改变成判断 "cmp w12, w15" 了  
.  
那要怎么 Patch? 我的思路如下:

```
1. 一次混淆!至少!涉及以下7个指令(中间穿插着其他逻辑的指令): 
  mov     w10, #0x60 
  ... 
  mov     w11, #0x58 
  ... 
  cmp     w7, w22 
  ... 
  csel    x23, x11, x10, lt 
  ... 
  ldr     x25, [x12, x23] 
  ... 
  add     x7, x25, x13 
  ... 
  br      x7 
2. 改为如下:
  mov     w10, #0x60      <- 可以nop掉 不nop也不影响结果 
  ... 
  mov     w11, #0x58      
  ...
  nop                     <-  cmp     w7, w22 [cmp语句要最后统一nop 因为会可能有多个逻辑共用同一个cmp] 
  ... 
  nop                     <-  csel    x23, x11, x10, lt 
  ... 
  nop                     <-  其他涉及到的指令 
  ... 
  cmp     w7, w22         <-  ldr     x25, [x12, x23] 
  b.lt    ...             <-  add     x7, x25, x13 
  b       ...             <-  br      x7 
  大多只有第一次混淆的时候这些混淆指令会穿插在一起, 之后基本都是ldr+add+br一个整体了 
```

就是 **cmp 下沉**, 将 "cmp + b.cc + b" 放到一起, 这样就不会因为其他指令的 cmp 导致条件被覆盖了  
**问:** 这样下沉如果 cmp w7, w22 中的 w7 和 w22 被之前的指令改变了怎么办?  
**答:** 事实证明是不会的, 我一开始的思路是不移动 cmp 而是在 cmp 之后保存 nzcv 标志位到例如 w10 中, 然后 b.cc 之前再恢复标志位, 结果发现有没有保存 nzcv 都一样.  
其实这个 so 中的函数都是在控制流平坦化之上又加了一层寄存器间接跳转, 所以这些 cmp 指令其实是控制流平坦化的分发指令, 这些值 (w7,w22 之类的) 在进入分发逻辑之前就确定好了, 是不会被改变的.

2. 编写插件代码
---------

> 代码逻辑分为: ①模拟执行 ②信息获取 ③Patch 逻辑 三部分

### 2.1 模拟执行代码

采用 unicorn 框架, 具体请查看 emulate.py 中的 "Emulator" "FuncEmulate" "DeJmpRegEmulate" 三个类, 其实就是给 unicorn 封装了一层.  
修改 条件选择指令 时要根据不同的类型进行修改:

```
#如果是csinc指令, 不满足条件应该改为add x24, x1, #1 | csinc是条件不满足则xd=xm+1, cinc是条件满足则xd=xn+1
if ((insn_token[0] == 'csinc' ) and (index == 1)) or ((insn_token[0] == 'cinc') and (index == 0)):
    if value == 'xzr':#如果是xzr寄存器就不能用add, 相当于赋值为了1
        mov_opcode = bv.arch.assemble(f"mov {cond_set_reg}, #1", condition_insn_addr)
    else:
        mov_opcode = bv.arch.assemble(f"add {cond_set_reg}, {value}, #1", condition_insn_addr)
elif (insn_token[0] == 'csinv') and (index == 1):
    mov_opcode = bv.arch.assemble(f"mvn {cond_set_reg}, {value}", condition_insn_addr) #按位取反
elif (insn_token[0] == 'sneg') and (index == 1):
    mov_opcode = bv.arch.assemble(f"neg {cond_set_reg}, {value}", condition_insn_addr) #取负值
else:
    mov_opcode = bv.arch.assemble(f"mov {cond_set_reg}, {value}", condition_insn_addr) #汇编mov x4, x9
```

### 2.2 信息获取代码

**问:** 怎么通过代码拿到一次混淆涉及到的全部指令?  
**答:** 我是通过从 mlil ssa 层面, 因为用 ssa 的话, 可以很方便的查找一个变量的被写入的语句, 代码中是通过 def_site.  
比如从 jump(x9_2#5) 开始, 先拿到 x9_2#5 的 def_site, 比如说是 "x9_2#5 = x9_1#4 + 0x3872d170", 然后取出这条语句的等号右边涉及到的变量, 这里是 x9_1#4, 然后拿到 x9_1#4 的 def_site, 比如是 x9_1#4 = [&data_1dd4c0 + x9#3].q @ mem#1, 然后拿 x9#3 的 def_site, 比如是 x9#3 = ϕ(x9#1, x9#2), 最后得到 x9#1 = 0, x9#2 = 0x58. 其实用一个递归就解决了:

```
def get_involve_insns(jmp_insn: MediumLevelILJump):
    def get_right_ssa_var(expr, vars: list):
        if isinstance(expr, SSAVariable):
            vars.append(expr)
            return
        elif isinstance(expr, list):
            for ope in expr:
                if isinstance(ope, SSAVariable):
                    vars.append(ope)
            return
 
        if hasattr(expr, 'operands'):
            for ope in expr.operands:
                get_right_ssa_var(ope, vars)
        return
 
    involve_insns = [] #涉及到的指令
    jmp_var = jmp_insn.dest.var
    var_stack = []
    var_stack.append(jmp_var)
    while len(var_stack) != 0: #拿到一次寄存器间接跳转混淆涉及到的所有指令
        cur_ssa_var = var_stack.pop()
        insn_ = cur_ssa_var.def_site #一条指令 应该是MediumLevelILSetVarSsa或MediumLevelILVarPhi
        if insn_ == None:
            break
 
        if insn_ in involve_insns:
            break #如果拿到的指令已经在之前获取到的指令中了, 说明遇到循环了
        else:
            involve_insns.append(insn_) #添加涉及到的指令
 
        if 'cond' in insn_.dest.name:#遇到cond:20#1 = x8#2 == 0x586b6221这种就不再继续了 要不然有可能遇到phi节点导致死循环
            break
 
        insn_right = insn_.src #这条指令=右边的表达式
        get_right_ssa_var(insn_right, var_stack) #拿到表达式中的变量            
     
    return involve_insns
```

然后通过 mlssa_insn.llils 拿到一条 mlil 指令涉及到的 llil 指令, llil 和汇编指令的地址是基本一一对应的:

```
involve_asm_addrs = [] #涉及到的汇编指令的地址 可能少csx赋值指令 后面补上
for mlssa_insn in involve_insns:
    llil_insns = mlssa_insn.llils
    for insn_ in llil_insns:
        if insn_.address not in involve_asm_addrs:
            involve_asm_addrs.append(insn_.address)
```

实际这样下来可能会缺少指令, 就是那两个设置跳转表偏移量的指令, 比如 "mov w27, #0x30" 和 "mov w28, #0x8"  
那么就通过从当前块开始, 向前继块从后往前搜索指令, 先拿到 csel/cinc 指令的操作寄存器, 然后搜索类型是 "mov", 第一个寄存器是条件选择指令操作寄存器的指令  
具体逻辑请查看 dejmpreg.py

### 2.3 Patch 代码

首先要拿到**从 csel 到 br 之间**的**所有**指令, 当然可以分段获取然后移动构造, 而且分段获取的话还可以应对从后往前跳转的情况 (当前混淆中是没有这种情况的), 只是我懒得写了:)

```
#0. 拿到所有要操作的指令
obf_insns_index = [] #指在csx2br_insns_text中的index
csx2br_insns_text = [] #从csx到br中的所有指令文本 (包含csx不包含br)
#1. 将混淆指令全转为nop, 并删除最后两个nop(一个nop改bcc, 一个nop改cmp)
for i in obf_insns_index:
    csx2br_insns_text[i] = 'nop'
csx2br_insns_text.pop(obf_insns_index[-1])
csx2br_insns_text.pop(obf_insns_index[-2]) #index本身就是从小到大排序的, 所以直接pop不影响
 
#2. 下沉cmp
cmp_txt = bv.get_disassembly(cmp_addr)
csx2br_insns_text.append(cmp_txt)
 
#3. 获取select指令的寄存器 并添加跳转
csx_tokens = (bv.get_disassembly(cond_addr)).split() #获取csel/cset/csinc等的token
csx_cond = csx_tokens[-1] #条件eq/lt等
bcc_cond = 'b.' + csx_cond
bcc_txt = f"{bcc_cond} {hex(tbr_addr)}"
csx2br_insns_text.append(bcc_txt)
b_txt = f"b {hex(fbr_addr)}"
csx2br_insns_text.append(b_txt)
logger.log_info(f"csx2br_insns_text: {csx2br_insns_text}")
```

3. 效果
-----

![](https://bbs.kanxue.com/upload/attach/202408/901761_QVXEB8FG2TAQJVC.png)

[[竞赛]2024 KCTF 大赛征题截止日期 08 月 10 日！](https://bbs.kanxue.com/thread-281194.htm)

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm)

上传的附件：

*   [libtprt.so.zip](javascript:void(0)) （710.58kb，20 次下载）