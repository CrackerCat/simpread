> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271574.htm)

> [原创] 线性 MBA 混淆的 LLVM Pass 实现

混合布尔算术（Mixed Boolean-Arithmetic, MBA）是 2007 年提出的一种混淆算法，这种算法由算数运算（例如 ADD、SUB、MUL）和布尔运算（例如 AND、OR、NOT）的混合使用组成。针对 MBA 的去混淆研究在长时间内都处于起步阶段，直到去年的一篇顶会论文 [MBA-Blast: Unveiling and Simplifying Mixed Boolean-Arithmetic Obfuscation](https://www.usenix.org/conference/usenixsecurity21/presentation/liu-binbin) 才提出了比较完美的去混淆方案。  
MBA 表达式有**线性 MBA 表达式**和**多项式 MBA 表达式**两种形式，本文讲解如何用 LLVM Pass 实现线性 MBA 表达式混淆。  
**本文涉及少量线性代数和数论的知识，并且本文使用的定理不会提供证明，证明请参考原论文。**  
原论文：[Information Hiding in Software with Mixed Boolean-Arithmetic Transforms](https://link.springer.com/chapter/10.1007/978-3-540-77535-5_5)

0x00. 基本定义
==========

Boolean-arithmetic algebra (BA-algebra), BA[n] 是一个代数系统，其定义如下：  
![](https://bbs.pediy.com/upload/attach/202202/910514_8EN864Y9QDCT4DZ.png)  
其中 Bn 表示所有 n 位二进制数的集合（其中 n 为正整数），后面的所有运算都是建立在集合 Bn 上的运算，上标有 s 运算表示有符号数的运算，否则为无符号数。  
多项式 MBA 的形式如下，其中 ai 是常量，ei,j(x1,...,xt) 表示由变量 x1,...,xt 组成的按位运算表达式：  
![](https://bbs.pediy.com/upload/attach/202202/910514_BBMW7ZA4PSHUG46.png)  
线性 MBA 表达式是多项式 MBA 表达式的简化版，其形式如下：  
![](https://bbs.pediy.com/upload/attach/202202/910514_CJ328ZC84P7WZJV.png)  
举个例子，第一个式子是由四个变量组成的多项式 MBA 表达式，第二个式子是由两个变量组成的线性 MBA 表达式：  
![](https://bbs.pediy.com/upload/attach/202202/910514_HMKTSHXR4P7E7YU.png)

0x01. 线性 MBA 恒等式构造
==================

看雪貌似不太支持数学公式，所以这部分我就直接贴图了：  
![](https://bbs.pediy.com/upload/attach/202202/910514_5BTD8PTMQDC5B6E.png)  
实际上布尔表达式可以随意选取，最后都能得到一个为 0 的恒等式（前提是至少有一个系数 aj 不为 0）。  
如果要对`x+y`进行替换，那么就在 MBA 恒等式两边加上`x+y`，得到`x+y=...`的恒等式，所以可以用右边复杂的 MBA 表达式对左边的`x+y`进行替换，以达到混淆的目的。

0x02. LLVM Pass 实现
==================

源码：[LinearMBAObfuscation.cpp](https://github.com/bluesadi/Pluto-Obfuscator/blob/main/llvm/lib/Transforms/Obfuscation/LinearMBAObfuscation.cpp) 以及 [MBAUtils.cpp](https://github.com/bluesadi/Pluto-Obfuscator/blob/main/llvm/lib/Transforms/Obfuscation/MBAUtils.cpp)  
也欢迎大家关注我的项目：[Pluto-Obfuscator](https://github.com/bluesadi/Pluto-Obfuscator)  
基本思路：

1.  随机选取`termsNumber`个布尔表达式构造真值表矩阵（参考上面的矩阵 A）
2.  令 AY=0，将矩阵方程转换为线性方程组，用 z3-solver 求出解向量 Y
3.  解向量 Y 即为 MBA 恒等式的参数，将参数与对应的布尔表达式相乘，得到项，将每一项相加，合并同类项，得到等于 0 的线性 MBA 恒等式
4.  以`x + y`的混淆为例，在恒等式左右两边同时加上`x + y`，得到`x + y`的 MBA 表达式替换。

1. 真值表
------

构造 MBA 恒等式的基础是真值表。以两个变量的 MBA 表达式为例，两个变量总共有四种可能的取值（x=0,y=0;x=0,y=1;x=1,y=0;x=1,y=1），可以转换为包含四个元素的列向量，例如`x & y`的真值表可以转换为`{0, 0, 0, 1}`。

```
int8_t truthTable[15][4] = {
    {0, 0, 0, 1},   // x & y
    {0, 0, 1, 0},   // x & ~y
    {0, 0, 1, 1},   // x
    {0, 1, 0, 0},   // ~x & y
    {0, 1, 0, 1},   // y
    {0, 1, 1, 0},   // x ^ y
    {0, 1, 1, 1},   // x | y
    {1, 0, 0, 0},   // ~(x | y)
    {1, 0, 0, 1},   // ~(x ^ y)
    {1, 0, 1, 0},   // ~y
    {1, 0, 1, 1},   // x | ~y
    {1, 1, 0, 0},   // ~x
    {1, 1, 0, 1},   // ~x | y
    {1, 1, 1, 0},   // ~(x & y)
    {1, 1, 1, 1},   // -1
};

```

2. 选取布尔表达式
----------

在预先设定的 15 个布尔表达式里随机选取`termsNumber`个来构造真值表矩阵。

```
int64_t* llvm::generateLinearMBA(int termsNumber){
    int* exprSelector = new int[termsNumber];
    while(true){
        context c;
        vector params;
        solver s(c);
        for(int i = 0;i < termsNumber;i ++){
            string paramName = formatv("a{0:d}", i);
            params.push_back(c.int_const(paramName.c_str()));
        }
        for(int i = 0;i < termsNumber;i ++){
            exprSelector[i] = rand() % 15;
        } 
```

3. 方程求解
-------

将矩阵方程转换为线性方程组，然后用 z3 求解。并且解不能为 0 向量：

```
for(int i = 0;i < 4;i ++){
    expr equ = c.int_val(0);
    for(int j = 0;j < termsNumber;j ++){
        equ = equ + params[j] * truthTable[exprSelector[j]][i];
    }
    s.add(equ == 0);
}
expr notZeroCond = c.bool_val(false);
// a1 != 0 || a2 != 0 || ... || an != 0
for(int i = 0;i < termsNumber;i ++){
    notZeroCond = notZeroCond || (params[i] != 0);
}
s.add(notZeroCond);
if(s.check() != sat){
    continue;
}
model m = s.get_model();

```

4. 合并同类项
--------

将布尔表达式相同的项合并（最多可能有 15 个项，对应 15 个布尔表达式）。并且合并之后不能所有的项都是 0：

```
        int64_t *terms = new int64_t[15];
        fill_n(terms, 15, 0);
        for(int i = 0;i < termsNumber;i ++){
            terms[exprSelector[i]] += m.eval(params[i]).as_int64();
        }
        // reject if all params are 0
        bool all_zero = true;
        for(int i = 0;i < 15;i ++){
            if(terms[i] != 0) all_zero = false;
        }
        if(all_zero){
            delete[] terms;
            continue;
        }
        return terms;
    }
}

```

5. 替换
-----

替换的原理非常简单，如果要替换的式子是`x + y`，那么在生成的 MBA 恒等式左右两边同时加上`x + y`即可。将存储了 MBA 表达式参数的`terms[2]`和`terms[4]`（对应 x 和 y）加上 1 即可。其他的式子如`x - y`、`x ^ y`等也是同理：

```
void LinearMBAObfuscation::substituteAdd(BinaryOperator *BI){
    int64_t *terms = generateLinearMBA(TermsNumber);
    terms[2] += 1;
    terms[4] += 1;
    Value *mbaExpr = insertLinearMBA(terms, BI);
    BI->replaceAllUsesWith(mbaExpr);
}

```

0x03. 混淆效果
==========

对 AES 进行混淆，以下是 SubBytes 函数的混淆效果：  
![](https://bbs.pediy.com/upload/attach/202202/910514_JEAXVK6NDRFP7PY.png)

0x04. 改进思路
==========

z3 求解线性方程组的速度比较慢，所以如果在混淆的时候临时去生成 MBA 恒等式，那么当项目比较大的时候会耗费很多时间（尤其是加密相关的程序，涉及大量运算）。所以可以事先生成一些 MBA 恒等式，并保存在文件里，混淆的时候从文件取用即可。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#软件保护](forum-4-1-3.htm)