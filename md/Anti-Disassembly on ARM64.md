> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270673.htm)

> Anti-Disassembly on ARM64

Anti-Disassembly on ARM64  

在 ARM64 平台，使用内联汇编对抗反汇编器的技巧。

先来一段 DCB/DCW/DWD/DCQ
--------------------

[ARM 文档如是说](https://developer.arm.com/documentation/dui0473/m/directives-reference/dcb?lang=en)

> The `DCB` directive allocates one or more bytes of memory, and defines the initial runtime contents of the memory.

DCB(DCW/DCD/DCQ 同理) 伪指令开辟一个字节或者多个字节的内存，并且定义了内存的初始值。

B = byte,W= word  (2bytes),D = dword(4bytes),Q = qword(8bytes)

并且 ARM 很贴心的给了一个正常示例:

```
C_string   DCB  "C_string",0

```

很通俗的理解就是 DCQ 是汇编语言里为了方便 (字符串) 常量定义和赋值的指令。既然是数据定义，那么指令出现在可读可写的数据段才更合理。正常人和正常的编译器是不会把 DCQ 放到只读的代码段的，但是不怎么正常的逆向 (安全) 工程师会这样做。既然我们知道 DCQ 是一段数据，那么我们就可以用内联汇编来构造一段 DCQ。直接用. long 后边跟上任意四字节，我这里就写了两个`12345678`和`0x12345678`, 写`0xdeadbeef`也可以。用 Xcode new 一个 demo，并把下边的代码贴到 demo 里。

```
static __attribute__((always_inline)) void DCQ_Demo() {
#ifdef __arm64__
    __asm__(
            ".long 0x12345678\n"
            ".long 12345678\n" 
            );
#endif
}

```

clean，build，product 扔到 IDA 里。熟悉的 DCQ 来了，这两个. long 把后边的解析直接带跑偏了，IDA 不解析了。

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202112/828976_4JX5ZWMTMGVWNHF.jpeg)

再来看看动态的 Xcode，第一个. long `0x12345678`被解析成了正常的汇编代码。第二个`12345678`无法识别，因此注释了`unknow opcode`。先别着急往下看，大家可以猜想一下放开断点之后会发生什么？

![](https://bbs.pediy.com/upload/attach/202112/828976_WDN6YCP8RQUGKHM.jpeg)

第一个 0x12345678 被解析成了正常的汇编代码。实际执行过程中改变了 w24 寄存器的值，由于上下文都没引用到 w24, 所以在这段程序里这行代码没有产生任何负面效果。再来看第二个 12345678 也就是`unknown opcode`。cpu 执行到这行，由于无法识别这段代码，所以直接抛出异常，程序崩溃了。

**小结：DCQ 是一条正常的汇编伪指令，用来声明内存并赋初始值。代码段 (可读可执行，不可写) 的 DCQ 可以用来声明数据。生成的垃圾指令无法被 IDA 正常解析也无法被 xcode 识别执行。结合其他指令可以用来做代码混淆。**

B 指令 + DCQ
----------

第一段我们已经知道了 DCQ 是什么，并且可以用内联汇编构造出 DCQ。但是 DCQ 本质上是一段数据 (指令)，能被正常解析成指令的话，运行时会产生不可预知的效果，不能被解析成指令的话，cpu 直接抛出异常。

如何能构造出 DCQ 又能让程序正常运行呢？ 可以用 B 指令，“跨过” 那两条不能被正常执行的指令。这样 DCQ 迷惑了反汇编器，B 指令又跨过了这些错误的指令。

用内联汇编怎么写 B 指令呢？

[来看一下 ARM 文档 B 指令](https://developer.arm.com/documentation/ddi0596/2021-06/Base-Instructions/B--Branch-?lang=en)

> B
> -
> 
> Branch causes an unconditional branch to a label at a PC-relative offset, with a hint that this is not a subroutine call or return.

蹩脚翻译：

1.  B 是无条件跳转，那啥是有条件？请君自学
    
2.  B 是相对跳转，既然相对了，那么参照物是啥？ PC-relative .
    
3.  B 跳转不是调用子函数，所以没 return。意思是不像 BL, 跳过去会把 LR 变了。
    

**写给菜鸟，大佬跳过：**

PC 是 program counter，程序计数器。每条指令执行完会 + 1，增加一个单位，也就是四字节。

众所周知操作系统加载程序会带上 ASLR, 也就是说每次程序加载地址都不一样，同样一段代码每次执行的 PC 值都不一样。但这并不会影响到 B 这种相对地址跳转。我们只要把相对地址固定好就行。

```
0x100000000 b 0xc # 当前PC = 0x100000000, 所以B的目的地址 = 0x100000000 + 0xc 也就是直接跳到C那里
0x100000004 A
0x100000008 B
0x10000000c C
0x100000010 D
0x100000014 E

```

所以用 B 跨过了那些迷惑反汇编器的指令。下边代码可以被插入到程序的任何地方。因为仅仅是多了一条 b，没有对寄存器的占用, 所以不会对程序逻辑产生任何影响。B 之间的 0x12345678 可以被任意替换，填充的长度也可以被任意替换。B 后边的操作数 = (填充代码的长度 + 1) * 4 。比如下边这条，填充了两条，所以 B 后边的操作数就是 (2 + 1)* 4 = 12 也就是 0xc。

```
static __attribute__((always_inline)) void B_DCQ_Demo() {
#ifdef __arm64__
    __asm__(
            "b 0xc\n"
            ".long 12345678\n"
            ".long 12345678\n"
            );
#endif
}

```

再看 IDA

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202112/828976_5NBHFBR7ADAVE9G.jpeg)

细心的你也许会发现，同样是两条. long , 为啥图一后边的代码都是 DCQ, 而这张图上却仅仅有两条呢？以下是我的猜测。

反汇编器的扫描策略大致可以分为两种：线性扫描和递归下降扫描 (flow-oriented) 。线性扫描当然很好理解，就是逐条的扫。但是线性扫描对指令长度是有要求的，固定长度的扫下来才不会出错 (说实话我觉得对于 ARM64 这种 4 指令固定长度的，线性扫描真的挺合适的) 。 Intel x86 指令是变长的，ARM32 也能切到 16bit 的 thumb 模式，上述平台线性扫描就不适用。就像考试抄袭学霸的答题卡，一旦抄错位一个，后边的就都错了。

所以主流的反汇编器都使用 flow-oriented 的扫描模式。所以我们可以先看一下图二，反汇编进行到 B 指令，程序流产生了分叉，所以 IDA 选择从 B 的目的地址接着扫。最后中间被跨过去的部分就没被扫到，也就比较原始的形式留在那里了。那么图一呢？ 扫到了错误的指令之后，IDA 直接停止了对当前流的扫描，直接返回到上个分叉的地方继续扫，那么上个分叉的地方是哪里？ 如果猜测正确的话，上个分叉的地方是下一个函数。

以上仅为猜测，也不是本文重点，希望不要误导。当然技多不压身，多了解一点反编译原理更有助于我们写出对抗代码。

虚假控制流 + DCQ
-----------

DCQ 迷惑了 IDA 但运行时会让程序崩溃, B 跨过去可以规避问题，但却让迷惑效果打折。除了 B 有没有更好的办法？ 当然有，把 DCQ 放到永远都不会被执行的分支里。也就是所谓的虚假控制流 BogonControlFlow。

先来看一下如何构造虚假控制流。构造虚假控制流是一个非常常用的技巧。有各种各样花式技巧。构造虚假控制流最需要考虑的是，构造出来的谓词不要被聪明的编译器给优化掉 (比如常量折叠之类的)。 举几个常见的例子，x 是整数 (x+1)(x+2) % 2 一定等于 0，同理 (x+1)(x+2)(x+3) % 3 也一定等于 0。把上述一定成立的结论取个反，就成了恒为假的条件。

下面示例用勾股数来构造一个虚假控制流。勾三股四弦五，那咱搞个弦六，够虚假了吧。开方运算 sqrt 在 math.h 里，不会被编译器优化。代码里的变量名比较随便，如果想在生产写这种代码，记得把 abc 换成不容易撞车的变量名。

```
static __attribute__((always_inline)) void BogonControlFlow_DCQ_Demo() {
#ifdef __arm64__
    int a = 3;
    int b = 4;
    int c = 6;
    if (c < sqrt(a*a+b*b)) {
        __asm__(
                ".long 12345678\n"
                ".long 12345678\n"
                );
    }
#endif
}

```

把脏指令扔到了虚假控制流里，迷惑了 IDA, 左侧的函数列表里甚至都看不到 viewDidLoad 的函数名了。右侧的汇编页面也变红了，没法 F5 了。

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202112/828976_U28HGCDA4NFABG8.jpeg)

B + 虚假控制流 + DCQ
---------------

有了虚假控制流和 DCQ 的加持，可以构造出混淆 case 了。在虚假控制流里，可以随意折腾任何指令。这次把 B 指令也加上，直接 B 到脏指令上。虽然 IDA 看起来与上文无异，但是我们可以把这个 case 拓展一下，变成另外一种混淆即堆栈不平衡。

```
static __attribute__((always_inline)) void B_BogonControlFlow_DCQ_Demo() {
#ifdef __arm64__
    int a = 3;
    int b = 4;
    int c = 6;
    if (c < sqrt(a*a+b*b)) {
        __asm__(
                "b 0x4\n"
                ".long 12345678\n"
                );
    }
#endif
}

```

B + 虚假控制流 + 堆栈不平衡
-----------------

```
static __attribute__((always_inline)) void B_BogonControlFlow_ADD_SP() {
#ifdef __arm64__
    int a = 3;
    int b = 4;
    int c = 6;
    if (c < sqrt(a*a+b*b)) {
        __asm__(
                "b 0x4\n"
                "add sp,sp,#0x100\n"
                "add sp,sp,#0x100\n"
                );
    }
#endif
}

```

程序运行在栈上，栈从上往下生长 (满递减，高地址向低地址生长。表述不同，其实都一个意思)。所以开辟空间就是减 sub sp sp 0x1234, 回收空间就是加 add sp sp 0x1234. 开辟和回收的空间一定相等。如果不相等会怎样？ 上边在虚假控制流里把 sp 加了一些，所以 IDA 分析的时候，直接导致了堆栈不平衡，没法 F5 了。

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202112/828976_DPDFVX6HMR2XYN3.jpeg)

用 BR 实现间接跳转
-----------

核心思想：把要跳转的地址藏到 BR 后边的寄存器里。因为 IDA 是静态反汇编器。反汇编过程中不会计算

[先看官方解释](https://developer.arm.com/documentation/ddi0596/2021-06/Base-Instructions/BR--Branch-to-Register-?lang=en)

> BR
> --
> 
> Branch to Register branches unconditionally to an address in a register, with a hint that this is not a subroutine return.

直接上代码，把要跳转的地址藏到寄存器里。静态分析无法获取寄存器的运行时的值，所以会让分析停下来。

最关键的是，如何能在 br 之前获取到紧接着 br 的一条地址。同样先用地址无关的 ADR 指令，把紧接着 br 指令的地址算出来，并把地址 “藏” 到 x8 寄存器里，直接用 br 跳过去。这样就实现了最简单的间接跳转。

```
static __attribute__((always_inline)) void BR_2_X8() {
#ifdef __arm64__
    __asm__(
            "mov x8,#0x1\n"
            "adr x9, #0x10\n"
            "mul x8, x9, x8\n"
            ".long 0x12345678\n"
            "br x8\n"
            );
#endif
}

```

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202112/828976_8Z3WR8MU94R4CRY.jpeg)

RET TO SELF
-----------

这是一个比较有趣的技巧。我把它命名成 ret to self。前文已经说过 IDA 是面向流的扫描方式，所以如果程序里如果不出现任何流 (也就是不出现任何跳转指令 B，BR,BL,BLR 等)。那么 IDA 会一直线性扫描到函数结尾。换句话说，我们构造一种 case，让 IDA 线性的扫描到 ret 以为函数已经结束。

直接看代码吧。第一条，用 adr 计算出了紧跟着 ret 指令后一条的 pc 地址。第二条，把这个地址放到 x30 寄存器里。为什么要这么做？

```
static __attribute__((always_inline)) void RET_2_SELF() {
#ifdef __arm64__
    __asm__(
            "adr x8,#0xc\n"
            "mov x30,x8\n"
            "ret\n"
            );
#endif
}

```

来看一下 RET 指令

> RET
> ---
> 
> Return from subroutine branches unconditionally to an address in a register, with a hint that this is a subroutine return.

在 a 函数里调用了 b，b 在 return 的时候发生了什么？ 当然是返回到 a 函数的调用处的下一条。调用处下一条的地址存在哪里？当然是 LR 寄存器里。LR 寄存器是什么？当然是 x30 了。

所以 ret 指令有一条 “等价” 写法 ==> `mov pc lr`。

再看上面的代码就很明显了，ret 之后实际是跳到了自己后面继续执行。所以叫 ret to self 没毛病把。

再看 IDA，成功被骗。IDA 没扫到任何流，线性的撞到了 ret 上，所以以为函数已经结束了。F5 之后得到一个空函数。

![](https://bbs.pediy.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)![](https://bbs.pediy.com/upload/attach/202112/828976_J37WYEWH7KX9CXZ.jpeg)

最后
==

谢谢收看。点个赞 / star。

[https://github.com/AppleReer/Anti-Disassembly-On-Arm64](https://github.com/AppleReer/Anti-Disassembly-On-Arm64)

上述代码仅为基本型的 Demo 演示。可以灵活组合使用。切勿直接复制粘贴在生产环境使用。寄存器污染了，程序崩溃了，是会被开除的！

[【公告】看雪团队招聘 CTF 安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 1 天前 被 mull 编辑 ，原因：

[#逆向分析](forum-166-1-189.htm) [#基础理论](forum-166-1-188.htm) [#程序开发](forum-166-1-191.htm) [#工具脚本](forum-166-1-195.htm)