> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271866.htm)

> [原创]angr 符号变量转 LLVM IR

前言
--

 

之前看到过国外的一篇文章。  
关于如何处理虚拟机的，并给出了针对 tigress 虚拟机的攻击方法。  
具体是这样处理的：

*   **装载目标文件**
*   **初始化一些 hook**
*   **对虚拟机函数进行符号执行，获取各种符号变量**
*   **将符号变量转化成 llvm 的 ir**
*   **使用 llvm 编译优化重编译**

后面那作者还研究出来了攻击 VMP 的方法，具体可以参考。  
https://github.com/JonathanSalwan/VMProtect-devirtualization

> 这个方法确实很巧妙，但是使用的 triton，然而其 python 接口基本没有说明。属实劝退。

[](#angr？)angr？
---------------

 

观察下具体的实现，发现是将 triton 处理出的符号变量的 expr 进行遍历，然后将某些语义节点转化为对应的 llvm ir。从而实现了将 expr lift 到 llvm 的 ir。然后就可以享用 llvm 自带的丰富的优化 pass 了。  
由于 triton 的学习成本较大（x，所以我们可以采用更加简单的符号执行工具 angr。同样的 angr 也使用树结构来描述一个符号变量。然后将其转化成 llvm 的 ir。

[](#claripy？)claripy？
---------------------

 

众所周知，angr 的约束求解框架前端是 claripy，执行出来的符号变量被 claripy 表示出来，然后交给后端的约束求解引擎，例如 z3。  
claripy 的表示中有很多种操作，这里列出两种常见又难以处理的操作。

*   **concat，将两个符号变量链接**
*   **extract，提取符号变量中的某几位。**

由于使用 python 编写，所以考虑使用 llvmlite 来构建 llvm 的 ir。安装过程略。  
对于一些其他的算数指令，可以直接将其映射到 llvm 的某个指令上去，而对于上面两个稍微复杂的操作，考虑直接用位运算来表示。  
这里是一张图，表示了 claripy 下的某个符号变量

 

![](https://bbs.pediy.com/upload/tmp/948449_RJB3N23Z6FR6GFZ.png)

### concat

链接 a 和 b 两个变量  
**concat(a,b)**  
**c = (zext(a, size(a) + size(b)) << size(b)) | (zext(b, size(a) + size(b)))**  
将两个符号变量扩展到两个符号变量位数之和，然后将前面的变量左移后面变量的位数，然后或上后面的那个变量即可实现两个符号变量的链接。**该运算在 angr 中表示为 a..b**

### extract

提取 a 变量的第 l 位到第 h 位。  
**extract(a,l,h)**  
**y = (a & bitmask) >> (h - l + 1)**  
先将变量与上一个 bitmask，可以与出对应的数据，然后一位，让数据从低 0 位开始，最后 trunc 截取一下长度。**该运算在 angr 中表示为 a[l:h]**

 

这样就可以做到将所有的操作用 llvm 的 ir 表示出来，接下来要做的就是遍历获取的 expr 的 ast，然后边遍历边构造 llvm 的 ir。出来又在 run 一下 llvm 自带的优化。

 

直接上代码：  
_解释一下，lifter 是一个 visitor，遍历 ast，然后在某个节点应用对应的处理函数，cur 则代表当前节点对应的值 (对应 LLVM 的 Value 概念)。所以先 visit 子节点，调用函数结束的时候 cur 就是子节点对应的 value，然后可以继续构造当前节点的 value。  
当节点是 BVS 或者是 BVV 时停止向下访问，然后分别处理其他未知变量和常量。未知变量设定为 llvm 的函数参数，常量可以直接提取出来，转化成 llvm ir 中的常量_

```
import angr
import claripy
from llvmlite import ir
import llvmlite.binding as llvm
unop_llvm = {
    '__invert__':ir.IRBuilder.not_,
    '__neg__':ir.IRBuilder.neg
}
binop_llvm = {
    '__add__':ir.IRBuilder.add,
    '__floordiv__':ir.IRBuilder.udiv,
    'SDiv':ir.IRBuilder.sdiv,
    '__mul__':ir.IRBuilder.mul,
    '__sub__':ir.IRBuilder.sub,
    '__mod__':ir.IRBuilder.urem,
    'SMod':ir.IRBuilder.srem,
    '__and__':ir.IRBuilder.and_,
    '__or__':ir.IRBuilder.or_,
    '__xor__':ir.IRBuilder.xor,
    '__lshift__':ir.IRBuilder.shl,
    '__rshift__':ir.IRBuilder.ashr,
    'LShR':ir.IRBuilder.lshr
}
signed_op = ['SDiv','SMod']
supported_op = ['Concat','ZeroExt','SignExt','Extract','RotateLeft','RotateRight'] + list(unop_llvm.keys()) + list(binop_llvm.keys())
supported_type = ['BVV','BVS']
class lifter:
    def __init__(self):
        self.expr = None
        self.cur = None
        self.count = 0
        self.value_array = []
        self.builder = None
        self.func = None
        self.args = {}  
        self.node_count = 0
 
    def new_value(self, value, expr):
        assert value.type.width == expr.size()
        n = self.count
        self.value_array.append(value)
        self.count += 1
        return n
 
    def get_value(self, idx):
        return self.value_array[idx]
 
    def _visit_value(self, expr):
        if expr.op == 'BVV':
            self.cur = self.new_value(ir.Constant(ir.IntType(expr.size()), expr.args[0]), expr)
        else:
            self.cur = self.new_value(self.func.args[self.args[expr]], expr)
        pass
 
    def _visit_binop(self, expr):
        left = None
        for a in expr.args:
            self._visit_ast(a)
            if left is None:
                left = self.cur
            else:
                v = self.cur
                lhs = self.get_value(left)
                rhs = self.get_value(v)
                self.cur = self.new_value(binop_llvm[expr.op](self.builder, lhs, rhs, name = "node" + str(self.node_count)), expr)
                left = self.cur
                self.node_count += 1
        pass
 
    def _visit_unop(self, expr):
        self._visit_ast(expr.args[0])
        v0 = self.cur
        self.cur = self.new_value(unop_llvm[expr.op](self.builder, self.get_value(v0), name = "node" + str(self.node_count)), expr)
        self.node_count += 1
        pass
 
    def _visit_concat(self, expr):
        left = None
        for a in expr.args:
            self._visit_ast(a)
            if left is None:
                left = self.cur
            else:
                v = self.cur
                lens = self.get_value(left).type.width + self.get_value(v).type.width
                val0 = self.builder.zext(self.get_value(left), ir.IntType(lens))
                val1 = self.builder.zext(self.get_value(v), ir.IntType(lens))
                self.cur = self.new_value(self.builder.or_(self.builder.shl(val0, ir.Constant(ir.IntType(lens), self.get_value(v).type.width)), val1, name = "node" + str(self.node_count)), expr)
                left = self.cur
                self.node_count += 1
        pass
 
 
    def get_bit_mask(self, low, high):
        mask = 0
        for i in range(low, high + 1):
            mask += 2 ** i
        return mask
 
    def _visit_extract(self, expr):
        high = expr.args[0]
        low = expr.args[1]
        self._visit_ast(expr.args[2])
        v0 = self.cur
        val = self.get_value(v0)
        mask = self.get_bit_mask(low, high)
        self.cur = self.new_value(self.builder.trunc(self.builder.lshr(self.builder.and_(val, ir.Constant(val.type, mask)), ir.Constant(val.type, low)), ir.IntType(high - low + 1), name = "node" + str(self.node_count)), expr)
        self.node_count += 1
        pass
 
    def _visit_zeroext(self, expr):
        length = expr.args[0]
        self._visit_ast(expr.args[1])
        v0 = self.cur
        self.cur = self.new_value(self.builder.zext(self.get_value(v0), ir.IntType(length + expr.args[1].size()), name = "node" + str(self.node_count)), expr)
        self.node_count += 1
        pass
 
    def _visit_signext(self, expr):
        length = expr.args[0]
        self._visit_ast(expr.args[1])
        v0 = self.cur
        self.cur = self.new_value(self.builder.sext(self.get_value(v0), ir.IntType(length + expr.args[1].size()), name = "node" + str(self.node_count)), expr)
        self.node_count += 1
        pass
 
    def _visit_rotateleft(self,expr):
        bit = expr.args[1]
        self._visit_ast(expr.args[0])
        v0 = self.cur
        val = self.get_value(v0)
        width = val.type.width
        self.cur = self.new_value(self.builder.or_(self.builder.lshr(val, ir.Constant(val.type, width - bit)), self.builder.shl(val.type, ir.Constant(val.type, bit)), name = "node" + str(self.node_count)), expr)
        self.node_count += 1
        pass
 
    def _visit_rotateright(self,expr):
        bit = expr.args[1]
        self._visit_ast(expr.args[0])
        v0 = self.cur
        val = self.get_value(v0)
        width = val.type.width
        self.cur = self.new_value(self.builder.or_(self.builder.shl(val, ir.Constant(val.type, width - bit)), self.builder.lshr(val.type, ir.Constant(val.type, bit)), name = "node" + str(self.node_count)), expr)
        self.node_count += 1
        pass
 
    def _visit_op(self, expr):
        if expr.op in binop_llvm.keys():
            self._visit_binop(expr)
        elif expr.op in unop_llvm.keys():
            self._visit_unop(expr)
        else:
            func = getattr(self, '_visit_' + expr.op.lower())
            func(expr)
 
    def _visit_ast(self, expr):
        assert isinstance(expr, claripy.ast.base.Base)
        if expr.op in supported_op:
            self._visit_op(expr)
        elif expr.op in supported_type:
            self._visit_value(expr)
        else:
            raise Exception("unsupported operation!")
 
    def lift(self, expr):
        self.expr = expr
        self.count = 0
        self.value_array = []
        self.args = {}
        c = 0
        for i in expr.leaf_asts():
            if i.op == 'BVS':
                self.args[i] = c
                c += 1
        items = sorted(self.args.items(),key=lambda x:x[1])
        print("Function arguments: ")
        print(items)
        type_list = []
        for i in items:
            type_list.append(ir.IntType(i[0].size()))
        fnty = ir.FunctionType(ir.IntType(expr.size()), tuple(type_list))
        module = ir.Module(name=__file__)
        self.func = ir.Function(module, fnty, )
        block = self.func.append_basic_block()
        self.builder = ir.IRBuilder(block)
        self._visit_ast(expr)
        self.builder.ret(self.get_value(self.cur))
        return str(module)

```

目前并未进行大规模测试，可能存在 bug。小测一波  
这是被测试的函数

 

![](https://bbs.pediy.com/upload/tmp/948449_NX3FTGWCGJV2EVT.png)

 

测试结果

 

![](https://bbs.pediy.com/upload/tmp/948449_HKZ8VZCW97HEZF4.png)

小结
--

*   这里可以想出一个简单 vm 的处理方法，首先我们找到**虚拟机的内存和上下文**，将其设置为**符号变量**，其他内容则初始化为常量，然后开始符号执行，然后**提取并化简**上下文中的符号表达式，然后**重新编译优化**即可重建虚拟机的逻辑。
*   当然这个虚拟机要足够简单，然而虚拟机往往都有跳转语句，符号执行在遇到跳转的时候将变得极其爆炸。生成的表达式也可能极其复杂，llvm 的优化也没有用。所以可以考虑**分而治之**。
*   找到虚拟机的跳转语句的 handler，然后在 angr 中 hook，根据跳转目的地和当前跳转语句地址 (**控制流生成算法**)，能够很好的将虚拟机的 opcode 构造控制流图，然后针对每一个 opcode 的 basicblock 进行符号执行，**将上下文全部符号化**，然后评估 basicblock 执行完毕后的**上下文的变化**，如果发生了改变则可以说明该 basicblock 干了什么。
*   然后**将这些副作用 (表达式) 收集起来**，则可以**代表**这个 basicblock 到底干了什么，最后**处理整个控制流图**，即可实现程序的**重构**。这种自动化的虚拟机代码重构，处理能力还比较弱，也只是笔者的一个想法，以后准备细细研究一波。

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 2 小时前 被 R1mao 编辑 ，原因：

[#VM 保护](forum-4-1-4.htm)