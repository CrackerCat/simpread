> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286780.htm)

> [原创]VMProtect3.5.1 脱壳临床指南

**a. 前言**
=========

  随着 VMProtect 版本的不断迭代，高版本的 VMProtect 脱壳变得极为困难。然而，在各大安全论坛上，仍有不少大牛发布的帖子深入探讨了 VMProtect 壳的特性，这些零星的讨论逐渐汇聚成详实的资料，供大家查阅与交流。在此基础上，我进一步补充一些内容，旨在抛砖引玉，如有不当之处，还望雅正。如果在阅读过程中有新的思路或想法，欢迎跟帖讨论，共同探索。以下将以 VMProtect 3.5.1 版本作为分析对象，选择该版本的原因在于其完整源码可供参考。接下来，我将分享我的探索过程。

#### **vmp 源码编译**

  VMProtect 3.5.1 完整源码于 2023 年 12 月 7 日泄露，此前流出的版本缺少两个核心文件：intel.cc 和 processor.cc。编译 VMProtect 3.5.1 源码时可能会遇到各种问题。看雪论坛上有几篇关于编译 VMProtect3.5.1 源码的帖子，内容详尽，几乎是手把手教学。如在编译过程中遇到问题，可以参考这几篇帖子！

#### **约定**

为便于后续叙述，现作如下约定：

**vm** 代指 **vm 虚拟机**，**vmp** 代指 **VMProtect**，**reg**、**reg1**~**reg3** 表示**通用寄存器**。

**numb、numb1、numb2** 或 **number** 表示立即数。**exp** 表示一个表达式。

**一个操作**指由**若干短跳**和**一个长跳**组成的指令流，其中短跳为 jmp+imm(立即数)，长跳为 jmp+reg(寄存器) 或 ret 指令，具体可参考示例 a-00-00。

![](https://bbs.kanxue.com/upload/attach/202505/877885_VZWV7DAE98755DJ.png)

约定 **vm 栈顶寄存器**为 **VMesp**，vm 栈顶的存取操作通常会以这样的形式出现：[esp+reg]。

约定 **vm 栈底寄存器**为 **VMebp**，**VmDataPtrReg** 为指向 vm 数据区的指针寄存器，**VmJmpReg** 为长跳 jmp 指令后面跟随的寄存器。**vmMagicReg** 为魔数寄存器，在一个操作中该寄存器和 **VmDataPtrReg** 取出来的数值相互运算，最终确定下一个长跳地址。

注：vm 栈底寄存器不限于 ebp，可为其他寄存器，但 vm 栈顶寄存器一定是 esp。在切换 vm 环境时，VMebp、VmDataPtrReg 及 VmJmpReg 均可能发生变化。

**b. 指令还原**
===========

00. 简单示例
--------

  VMProtect 提供两种代码保护方式：一是通过 SDK 直接保护代码；二是将程序拖入 VMP 界面，通过添加函数地址进行保护。使用 SDK 的方式更为便捷和灵活。

以下是使用 sdk 示例：

```
#include #include"VMProtectSDK.h"
 
__declspec(naked) int naked_function(int a)
{
    // 保护开始位置
    VMProtectBegin("naked_function");
    _asm {
        add ecx, 1
        ret
    }
    //保护结束位置
    VMProtectEnd();
} 
```

第二种保护方式：

![](https://bbs.kanxue.com/upload/attach/202505/877885_XN7QKJ3UZ46PAGJ.png)

此外，使用 SDK 保护代码时，VMProtect 会在代码开头插入额外代码。分析虚拟机（VM）时，过滤这些额外代码的方法如下：

① 首先，定位到被保护函数的地址，然后跳转至 vmp 的节区。

![](https://bbs.kanxue.com/upload/attach/202505/877885_WX96A2QYT8PTD9W.png)

② 在当前区域搜索特征码：68 ?? ?? ?? ?? E8 ?? ?? ?? ??

![](https://bbs.kanxue.com/upload/attach/202505/877885_EWG4WN2D3CWTJ67.png)

③ 从搜索的结果显示，**0x00419C26** 地址为代码保护的实际入口，分析应从此处开始。

![](https://bbs.kanxue.com/upload/attach/202505/877885_HEHDNR72K9ZC89Z.png)

-------------------------

没过滤前（即从 **0x41AC92** 开始分析），还原的代码：

![](https://bbs.kanxue.com/upload/attach/202505/877885_74PQRPHZHWKRWU6.png)

-------------------------

过滤后（即在 **0x00419C26** 开始分析），还原的代码：

![](https://bbs.kanxue.com/upload/attach/202505/877885_BMESVN8Y57Z42NA.png)

01. 框架搭建
--------

### 构建堆栈副本

  进入虚拟机（VM）环境后，即可构建堆栈副本。首先记录 esp 的位置，随后记录通用寄存器和标志寄存器压入栈区的位置。以下示例 b-01-00 为进入虚拟机环境的一个操作。

![](https://bbs.kanxue.com/upload/attach/202505/877885_ZXRTHZAUVFPB7XN.png)

  当真实寄存器的值被压入 vm 栈底后，再通过从 vm 栈底取值并放置到 vm 栈顶 [esp+xxx] 的操作，与 [esp+xxx] 位置形成了一个唯一的映射关系，见下图。

![](https://bbs.kanxue.com/upload/attach/202505/877885_2Y8R33AQW2MM9YS.png)

最终创建的虚拟机（VM）栈区副本信息，其可视化效果如下图所示：

![](https://bbs.kanxue.com/upload/attach/202505/877885_XUQZ8RD6ZRRYYSN.png)

### 特征收集

  根据前述对 “一个操作” 的定义，通过提前扫描即可获取一个操作。在分析过程中，你将会发现不同操作的功能各异：一些操作从 vm 栈顶 [esp+xxx] 取值并放入 vm 栈底，而一些操作可能直接从 vm 栈底取值进行运算等。示例 b-01-01 展示了一个退出 vm 虚拟机环境的一个操作。

![](https://bbs.kanxue.com/upload/attach/202505/877885_JDVG4E4Q57Z8RNT.png)

抽取有用的信息整理如下：

```
mov esp,ebp                                  
pop ebp                                      
pop eax                                      
pop edx                                      
pop ebx                                      
popfd                                        
pop esi                                      
pop ecx                                                                       
pop edi                                      
```

如何确认上述操作属于退出 vm 虚拟机环境的特征？

答：通过提前扫描，将一个操作的所有指令收集至缓存（如 buff），然后使用 strstr(buff, "mov esp,") 在缓存中搜索子串。若搜索结果不为空，则表明该操作符合退出 vm 虚拟机环境的特征。特征匹配成功后，再通过单步异常回调函数来逐一处理这些指令。

以下总结了一个操作的主要特征：

```
// 特征01：vm栈底取值，放入vm栈顶[esp+xxx]中
// mov reg1,dword ptr [VMebp]
// mov reg3,byte ptr [VmDataPtrReg]
// mov dword ptr [esp+reg3],reg1
///
 
 
// 特征02：vm栈底取值，然后放回栈底
// mov reg1,dword ptr [VMebp]
// mov reg3,dword ptr [reg1]
// mov dword ptr [VMebp],reg3
///
 
 
// 特征03：vm栈底取值，进行运算
// mov reg1,dword ptr [VMebp]
// mov reg3,dword ptr [VMebp+4]
// not reg1;
// not reg3;
// and/or/add/shr/shl reg1,reg3
// mov dword ptr [VMebp+4],reg1
// pushfd
// pop dword ptr [VMebp]
///
（注：特征03的运算种类其实很庞杂，并且存在一些变体。此处仅列举了一般情况，建议读者自己总结和归纳。）
 
 
// 特征04：vm栈顶[esp+xxx]取值，放入vm栈底
// movzx reg1,byte ptr [VmDataPtrReg]
// mov reg3,dword ptr [esp+reg1]
// mov dword ptr [VMebp],reg3
///
 
 
// 特征05：数据区取值，放入vm栈底
// mov reg1,dword ptr [VmDataPtrReg]
// mov dword ptr [VMebp],reg1
///
 
 
// 特征06：退出vm虚拟机环境
// mov esp, reg
///
 
 
// 特征07：切换vm环境
// mov vmMagicReg, reg
///
 
 
// 特征08：切换vm环境2      （注：特征08是特征07的变体）
// lea VmJmpReg, dword ptr ds:[number]
///
 
 
// 特征09：栈底取值，赋值给VMebp本身   
// mov VMebp,dword ptr [VMebp]
///
注1：特征09用于调整堆栈，包括提升（对应的指令还原为sub esp,numb）或恢复（对应的指令还原为add esp,numb）。
注2：或用于call指令带参情况下恢复堆栈平衡。
//-----------------------------------------------------------------------------//
```

### 优化规则

为了方便描述，在特征 03 运算时获取的有效指令，如:

```
not reg1
not reg2
or reg1,reg2
```

可以简写为 or (not)reg1, (not)reg2, 类似地

```
not reg1
not reg2
and reg1,reg2
```

可以简写为 and (not)reg1, (not)reg2。

  当然，在特征 03 这个运算过程中，中间数据是通过抽象语法树（AST）来存储的。AST 并非复杂概念，本质上是一棵二叉树，其中内部节点表示运算符，叶子节点表示运算对象。用二叉树来表示表达式非常方便，例如表达式 or (not)reg1, (not)reg2 ，可表示为如下的二叉树结构：

```
       or    （根节点）
     /    \
  not      not   （内部节点）        
  /          \
reg1         reg2    （叶子节点）
```

注：该框架将所有 token 类型（如寄存器 token、数字 token 等）都封装为表达式，以实现统一管理。

  现在，假设 vm 栈顶 [esp+0x20] 位置映射的寄存器为 reg1，并从该位置取两次值并放入 vm 栈底进行运算，得到 or (not)reg1, (not)reg1。显然，or (not)reg1, (not)reg1 运算后结果不变，仍为 (not)reg1。因此，or (not)reg1, (not)reg1 可直接优化为 (not)reg1。以下是总结的优化规则：

```
①：(or/and (not)reg, (not)reg)         => (not)reg
②：(not (and (not)reg1, (not)reg2))    => or reg1,reg2
③：(not (or (not)reg1, (not)reg2))     => and reg1,reg2
④：(not)((not)reg) => reg
⑤: 消除死指令: [1] mov reg,reg
              [2] push reg
                  pop reg
 
⑥：(not)(add (not)reg1, reg2)    => cmp reg1, reg2 或者 sub reg1, reg2
⑦：(or/and (not)exp, (not)exp)   => (not)exp  (注：exp是表达式）
⑧：(and/or/add numb1, numb2)    直接计算     （注：numb1、numb2表示数字）
⑨: 寄存器折叠:  mov reg2,reg3; mov reg1,reg2    => mov reg1,reg3
                  
```

    
  上述优化规则并非全部在特征 03 中运算时执行。例如，优化规则⑥实际上是一个指令还原的匹配规则，其前式在运算后会被放入 vm 栈顶 [esp+xxx] 的某个位置。这一操作对应特征 01，因此可将优化规则⑥融入处理特征 01 的函数中。

### 虚拟寄存器

  这里规定如下：若从 [VmDataPtrReg] 区域获取立即数（如 numb1），并将其放入 vm 栈顶 [esp+0x10] 位置（假设），若该位置为空，则需为此位置创建虚拟寄存器，并生成一条还原指令 mov VirtualReg001, numb1，其中 VirtualReg001 为该位置新创建的虚拟寄存器。下面将通过示例 b-01-03 来说明虚拟寄存器的创建及消除过程。

示例 b-01-03 代码清单:

```
#include #include"VMProtectSDK.h"
 
// 需要vmp保护的函数
__declspec(naked) int naked_function(int a)
{
    VMProtectBegin("naked_function");
    // immediate test01
    _asm {
        mov ecx,0x333
        ret
    }
    VMProtectEnd();
}
 
int main()
{
    int x = naked_function(0x666);
    if (x == 0x777) {
        std::cout << "succeed！！！ " << std::endl;
    }  else  {
        std::cout << "failed！！！ " << std::endl;
    }
     
    getchar();
     return 0;
} 
```

在 naked_function 函数被虚拟机保护后，现在指令还原的效果如下：

![](https://bbs.kanxue.com/upload/attach/202505/877885_2PWQHU2VAYSUEWK.png)

  从以上的日志信息可知，存在一条 mov ecx, VirtualReg001 的还原语句。在退出 vm 虚拟机环境时，pop reg1 指令隐含的语义动作等价于 mov reg1, reg2/imm/VirtualRegXXX。当 reg2 与 reg1 相同时，可以使用优化规则⑤消除该指令，即移除 mov reg1, reg2。对于虚拟寄存器而言，仅当最终出现 mov reg, VirtualRegXXX 这种形式时，VirtualRegXXX 才被视为有解。例如，上面 VirtualReg001 对应指令为 ecx，因此可以用 ecx 替换 VirtualReg001 出现过的地方。

  这一简化过程可通过脚本或插件自动替换实现，或通过后续所述的二次扫描处理。以下是通过二次扫描后，指令还原的效果：

![](https://bbs.kanxue.com/upload/attach/202505/877885_QSWXWTEFP725G73.png)

### 分支

vm 虚拟机在处理 JCC 指令时，会将判断逻辑拆分为多个操作。例如 jl 指令：

```
JL 跳转的条件是 SF ≠ OF，即溢出标志和符号标志不同时为0。
虚拟机（VM）将分别判断 SF 和 OF 标志位（如下所示），并根据标志位信息决定是否跳转。
(not)(or(not)0x80,(not)eflags) => and 0x80,eflags
(not)(or(not)0x800,(not)eflags) => and 0x800,eflags
```

  要判断是否是 jl 指令，只需要在特征 01 中判断来自 vm 栈底来的表达式是否是为 "and 0x80,eflags"。遇到 JCC 指令时，需保存寄存器和堆栈副本的信息。解析至返回地址后，检查是否仍有未处理的分支。若有，则恢复之前保存的寄存器和堆栈副本，并切换至另一个分支，依此逐一解析所有指令。需要注意的是，强行切换分支，可能会引发内存访问异常等问题，此时无需惊慌。因为提前扫描确认了该操作的特征类型，异常发生时可直接设置 EIP 跳过异常地址。

  为统一处理 JCC 指令的程序流程，设定规则为：第一次不跳转，第二次跳转。此设计的优势在于，还原后的指令存放的数据结构是动态数组，第二次跳时，还原的指令可直接附加到数组末尾，而无需重新去构建控制流图（CFG）。

  以下示例 b-01-04 展示了测试 jl 指令的代码清单，以及第一次扫描后指令还原的效果。

![](https://bbs.kanxue.com/upload/attach/202505/877885_3T69RY6G84UT7DY.png)

二次扫描后，指令还原的效果：

![](https://bbs.kanxue.com/upload/attach/202505/877885_6A8U7G9JN8APD7D.png)

### 循环

  循环与分支的区别在于循环至少包含一条回边，但两者的处理步骤一致。唯一不同之处是需额外添加标记：在解析过程中，为每条还原指令附上当前 VmDataPtrReg 值的标记。获取还原指令后，与之前还原的指令对比，若 VmDataPtrReg 值相同，则表明循环已找到落脚点，并在该指令前插入一个标签，然后再次检查是否仍有未处理的分支。若无未处理分支，则指令还原流程结束。至此，整个指令还原框架搭建完成。

  示例 b-01-05 展示的是测试循环的代码清单，以及第一次扫描后指令还原的效果。

![](https://bbs.kanxue.com/upload/attach/202505/877885_DRYMYVYY7VWPPQ3.png)

二次扫描后，指令还原的效果：

![](https://bbs.kanxue.com/upload/attach/202505/877885_JTVWBN3YXFFW4XN.png)

02. 疑难问题
--------

### 二次扫描

  二次扫描，具体是指重新调试目标程序，与第一次扫描过程相同，唯一区别在于利用了第一次扫描时收集的信息。现在我们来讨论二次扫描的必要性。先看示例 b-02-00，该示例展示了测试 imul 指令的代码清单，以及第一次扫描后指令还原的效果。

![](https://bbs.kanxue.com/upload/attach/202505/877885_ZCAJ9EW98YYWU28.png)

注意观察，在还原 imul 指令的过程中，多出了一条 **mov edx, 469670** 指令。

**成因分析**：

  imul 指令在虚拟机（VM）中会先计算第二个和第三个操作数，中间结果被置于栈顶 [esp+0x1C]（假定），该地址实际上隐式映射到 edx 寄存器（此映射关系仅在退出 vm 环境时，解析 pop edx 指令才会明确），然而，原先[esp+0x14] 位置映射的 edx 寄存器关系未被清除，导致一个数值被放入 [esp+0x14] 位置时，触发了一次 mov 赋值操作。

**二次扫描需解决的问题**：

  在还原 imul 指令的过程中，需提前识别 [esp+0x1C] 位置映射到 edx 寄存器，并且及时清除 edx 寄存器在 [esp+xxx] 中多余的映射关系。因此，在第一次扫描时，信息收集显得尤为重要。

**第一次扫描时，收集信息的具体步骤**：

  为每个置于栈顶 [esp+xxx] 的表达式分配唯一编号。在特征 06 退出虚拟机环境时，遇到 pop reg1 指令，若还原指令非 mov reg1, reg1，则记录三项数据：第一操作数的寄存器号数、第二操作数的编号及运算符类型（即 mov）。

下面为二次扫描后指令还原的效果，可见冗余（或错误）的指令已被消除：

![](https://bbs.kanxue.com/upload/attach/202505/877885_EP67SK3HFG3E3Y7.png)

注：由示例 b-02-00 可以看到，虚拟寄存器也可以用来处理一些特定类型的操作数。

### 恒等变换

在恒等变换中，最典型的指令就是 **xor** 指令。在数理逻辑中，有

```
A⊕B = (A∧¬B)∨(¬A∧B)
     = ((A∧¬B)∨¬A)∧((A∧¬B)∨B)
     = (¬B∨¬A)∧(A∨B)
```

下面示例 b-02-01 展示了 **xor** 指令测试，及指令还原后的效果。

![](https://bbs.kanxue.com/upload/attach/202505/877885_9S22W9RQBBBWRS2.png)

从打印的日志信息上看，可以观察到 xor ecx, edx 指令还原的整个信息流。

现在整理如下：

```
or (not)edx,(not)edx            => (not)edx            下条指令的第二个操作数
or (not)ecx,(not)((not)edx)     => or (not)ecx,edx    第五条指令第二个操作数
or (not)ecx,(not)ecx            => (not)ecx            下条指令的第一个操作数
or (not)((not)ecx),(not)edx     => or ecx,(not)edx    第五条指令第一个操作数
or (not)(or ecx,(not)edx),(not)(or (not)ecx,edx)    => or (and (not)ecx,edx),(and ecx,(not)edx)
```

表达式 or (and (not)ecx, edx), (and ecx, (not)edx) 对应逻辑公式为 (A∧¬B)∨(¬A∧B)，因此指令最终还原为 xor ecx, edx。

### 待定指令

  若不同指令在还原时使用了相同匹配规则，导致无法区分，则称为待定指令。示例 b-02-02 展示了 cmp 和 sub 指令的测试情况，仔细观察右侧框住的日志信息，可以看到 cmp 和 sub 指令在还原时使用了相同的匹配规则。

![](https://bbs.kanxue.com/upload/attach/202505/877885_HQ2UC7VAW39HDY8.png)

cmp/sub reg1,reg2 对应的匹配规则，现整理如下：

```
or (not)reg1, (not)reg1    => (not)reg1    下条指令的第一个操作数
add (not)reg1, reg2                        下条指令的操作数
or (not)(add (not)reg1, reg2 ),(not)(add (not)reg1, reg2 )    => (not)(add (not)reg1, reg2)
 
取反操作有：(not)reg1 = 1-reg1，因此
(not)(add (not)reg1, reg2) = 1-((1-reg1)+reg2) = reg1-reg2，
即对应的是优化规则⑥：(not)(add (not)reg1,reg2) => sub/cmp reg1, reg2
```

如何消除 cmp/sub 的待定状态？

答：方法很简单。上述结果会存储至 vm 栈顶 [esp+xxx] 的某个位置。每次操作 [esp+xxx] 时，需判断其中的表达式是否为待定状态。若为待定状态，则读取 [esp+xxx] 时对应 sub 指令，而写入 [esp+xxx] 或不操作 [esp+xxx] 时对应 cmp 指令。

此外，待定指令还包括 test/and reg1, reg2。其对应的匹配规则，整理如下：

```
or (not)ebx, (not)edx   下条指令的操作数
or (not)(or (not)ebx, (not)edx), (not)(or (not)ebx, (not)edx)   => and reg1, reg2
因此匹配规则为：and reg1, reg2  => test/and reg1, reg2
```

### CMOVcc

  CMOVcc 指令在保护过程中会被转换为 JCC 指令。初次还原后，观察还原代码可发现 CMOVcc 指令的痕迹较为明显。尤其在第二次扫描后，更为明显。示例 b-02-03 展示了 cmovc 指令测试的代码清单，及第一次扫描后的指令还原效果。

![](https://bbs.kanxue.com/upload/attach/202505/877885_87MFC5UWBUJY8J6.png)

第二次扫描后，指令还原的状态：

![](https://bbs.kanxue.com/upload/attach/202505/877885_Q2MMYUZ5E5KMPJ9.png)

显然，两次扫描相比，第二次扫描更有利于 CMOVcc 指令的还原。下面简要描述 CMOVcc 指令还原的算法：

 **①** 首先，检查标签 label1 的上一条指令，若非 ret 指令，则跳转至⑥。

 **②** 初始化两个指针，记为指针 01 和指针 02，分别指向 jc label1 和标签 label1 的下一条指令。

 **③** 若两指针指向的指令相同，则判断是否为最后两条指令；若是，则跳转至⑤；否则，两个指针分别移至下一条指令，返回③。若指令不同，跳转至④。

 **④** 首次发现不同指令，将指针 02 指向的指令保存为 (mov reg1, reg2)，指针移至下一条指令，返回③。否则跳转至⑥。

 **⑤** 匹配成功，还原为 cmovc reg1, reg2 指令。

 **⑥** 匹配失败，直接返回。

----------------------

执行上述 CMOVcc 还原算法后，最终指令还原的效果如下：

![](https://bbs.kanxue.com/upload/attach/202505/877885_5NTKYSD42PXX46M.png)

### 幽灵寄存器 esp

  在第一次扫描时，无法实时获取 esp 的值，根源在于放入 vm 栈底的表达式存在二义性：该表达式究竟用于实际压栈 （即 push xxx）还是用于运算。为解决此问题，仍需依赖二次扫描。第一次扫描后，仅需标记涉及堆栈变化的指令并记录其数值，未发生堆栈变化的指令标记为 0。第二次扫描时，为每条指令加上第一次扫描记录的数值，即可实时获取 esp 的正确位置。示例 b-02-04 展示了 esp 指令测试时的代码清单，及第一次扫描后的指令还原效果。

![](https://bbs.kanxue.com/upload/attach/202505/877885_S8ABGMRAF3E3QAA.png)

可以看到，lea eax, dword ptr[esp-0x40] 在第一次扫描时未能还原。但经过二次扫描后，该指令已成功还原：

![](https://bbs.kanxue.com/upload/attach/202505/877885_KYY5DTHSGR5KYVS.png)

### 寄存器逃逸问题

  在测试过程中，可能遇到以下问题：创建的虚拟寄存器被使用，但在退出虚拟机环境时并未映射到真实寄存器，此类问题统称为寄存器逃逸问题。请参考以下示例。

示例 b-02-05 代码清单：

```
#include #include int g_canary = 0; // 暗桩
 
// 要保护的函数
bool check_isvalid(int val)
{
    int tmpVal = g_canary + val;
    if (tmpVal % 3) {
        g_canary++;
        return false;
    }
    return true;
}
 
int main()
{
    // 初始化随机数生成器
    srand(time(NULL));
 
    // 生成一个能被3整除的随机数
    int random_num = rand();
    g_canary = random_num - (random_num % 3); // 调整到最近的3的倍数
 
    // 输入整数
    int num;
    std::cout << "Enter an integer: ";
    std::cin >> num;
 
    // 检查输入值
    bool ret = check_isvalid(num);
    if (ret == true) {
        printf("注册码输入正确!!!\n");
    } else {
        printf("注册码输入错误！！！\n");
    }
    system("pause");
    return 0;
} 
```

check_isvalid 函数的原汇编代码如下：

```
push ebp
mov ebp,esp
sub esp,0xCC
push ebx
push esi
push edi
lea edi,dword ptr ss:[ebp-0xC]
mov ecx,0x3
mov eax,0xCCCCCCCC
rep stosd
mov ecx, call test_comprehensive.10113AC
nop
mov eax,dword ptr ds:[]
add eax,dword ptr ss:[ebp+0x8]
mov dword ptr ss:[ebp-0x8],eax
mov eax,dword ptr ss:[ebp-0x8]
cdq
mov ecx,0x3
idiv ecx
test edx,edx
je test_comprehensive.10123A1
 
mov eax,dword ptr ds:[]
add eax,0x1
mov dword ptr ds:[],eax
xor al,al
jmp test_comprehensive.10123A3
 
mov al,0x1
 
pop edi
pop esi
pop ebx
add esp,0xCC
cmp ebp,esp
call test_comprehensive.10112B2
mov esp,ebp
pop ebp
ret 
```

第一次扫描后，指令还原如下：

![](https://bbs.kanxue.com/upload/attach/202505/877885_7FHYJSRNGWY978C.png)

第二次扫描后，指令还原如下：

![](https://bbs.kanxue.com/upload/attach/202505/877885_FSYA39JBRKMAYMA.png)

  第二次扫描后，标注的问题 1 便是寄存器逃逸问题：VirtualReg005 未映射至任何真实寄存器。VMProtect 会利用局部优化技术，通过分析小范围内寄存器的使用或定值情况，判断是否允许寄存器逃逸。

  为解决此问题，需定位最后使用 VirtualReg005 的指令，并检查其下一条指令是否存在不活跃寄存器。例如，在下一条指令 mov eax, dword ptr [FFFFFFF8+ebp] 中，eax 为不活跃状态，因此可直接用 eax 替换 VirtualReg005。

但若出现以下情况，应如何处理？

```
... ; 省略
mov dword ptr [FFFFFFF8+ebp], VirtualReg005
add eax, dword ptr [FFFFFFF8+ebp]
... ; 省略
 
当把 mov 指令换成为 add 后，eax就为活跃寄存器了。将指令 add eax, dword ptr [FFFFFFF8+ebp] 拆解为四元式：
tmp0 = FFFFFFF8 + ebp
tmp1 = *tmp0
eax = eax + tmp1
可见，eax 在赋值前已被使用，处于活跃状态。
```

  若最后使用 VirtualReg005 的指令的下一条指令中寄存器均为活跃，则需继续检查下下一条指令，依此类推，直至找到不活跃的寄存器。VirtualReg005 的替换规则为：寻找最近的不活跃寄存器进行替换。

### 其他问题

  在测试过程中，可能遇到指令还原后出现错位的问题。此问题易于解决：每次从 vm 栈顶 [esp+xxx] 取出表达式放入 vm 栈底时，为每个表达式分配一个唯一的递增值。这样在运算时，若两个表达式组合形成一条新指令，则以唯一递增值较小的表达式作为该指令的序列依据。

  此外，当堆栈未按 4 字节对齐时，需额外准备一个堆栈副本。因此，在操作 vm 栈底数据时，需附加判断栈底是否按 4 字节对齐，以确定使用哪个堆栈副本。

03. 框架优化（讨论）
------------

### **慎用正则匹配**

  正则匹配在处理多字符时，搜索子串的开销较大，速度较慢。因此，在提前扫描以识别操作特征时，不宜使用正则匹配。例如，在示例 b-01-01 中，通过 strstr 在缓存中搜索 "mov esp," 子串来匹配特征 06。然而，在处理单条指令的复杂场景时，正则匹配则可以简化算法。例如，在特征 03 中，匹配指令 “and/or/add/shr/shl reg1, reg3” 时，使用正则匹配就比较合适。

### **采用并行方案**

  在被 vmp 保护的**原函数**中，通常会存在大量条件跳转 JCC 指令。一种可行的优化策略是在解析过程中遇到 JCC 指令时，创建新的线程（假设线程池资源充足）来并行处理跳转分支，随后将各分支收集的信息进行归并整理。

  这种并行化方法在多核处理器环境下，理论上能显著提升分析效率，特别是对于具有复杂执行路径的虚拟化代码。但是实现起来有很大的挑战性，包括跨线程的同步控制机制、执行状态的分布式管理，以及分析结果的动态聚合等复杂问题。我尚未进行实际测试。对此感兴趣的读者可以在自己的项目中尝试实现，并评估其效果。

### **指令模拟执行**

  由于该框架采用动态还原机制并依赖单步回调函数，每次指令执行都会导致环切换（Ring 3 → Ring 0 → Ring 3），这构成了单步调试性能开销的主要来源。如果采用 Unicorn Engine（CPU 模拟器）来执行指令，理论上可以避免用户态与内核态之间的频繁切换，从而提高执行效率。

  然而，模拟器执行指令时会引入额外的指令翻译和调度开销，这些开销可能会抵消掉环切换减少带来的性能优势。我尚未进行实际测试。有兴趣的读者可以在自己的项目中实际测试这两种方案，比较它们在不同条件下的性能表现。

04. 结语
------

  如遇到较复杂的问题时，可先标记放在一旁，切忌扬汤止沸。随着对 VMProtect 的深入理解，之前的问题往往能迎刃而解。善用二次扫描，当问题难以解决时，可尝试通过二次扫描来处理。由于篇幅限制，无法详尽呈现所有细节，但指令还原的核心要点及疑难问题均已涵盖。对于未涉及的内容，建议参考其他帖子或教程。

c. IAT 修复
=========

先看一个只勾选了输入表保护（见下图）的程序示例。

![](https://bbs.kanxue.com/upload/attach/202505/877885_K4DQYMUG989X9C9.png)

00. 寻找 OEP
----------

调试时，让程序停在壳子入口处，并在 [esp-0x04] 处，设置硬件访问断点。

![](https://bbs.kanxue.com/upload/attach/202505/877885_6MKNAGQXBWH8KDB.png)

按 F9 运行程序，直至首次停在非 VMProtect 的节区（如下图），即为原始入口点（OEP）。

![](https://bbs.kanxue.com/upload/attach/202505/877885_46HNYMHYVCE77AT.png)

01. 修复方法
--------

随意定位一个地址被加密的 call 指令，如图 c-01-00。

![](https://bbs.kanxue.com/upload/attach/202505/877885_7FFCAXV89D6JFSJ.png)

  从以上图中看到，跳板函数同样受到虚拟机保护，直到最后的长跳指令才解密目标 API 地址。因此，程序执行到长跳指令时，记录 API 地址是一种有效方法。基于此，修复思路已初步形成，以下为简要算法描述：

 **①** 遍历代码区，只收集跳转至 "vmp0" 节区的 call 指令，记录 call 指令地址及跳转地址，并放到一个结构数组中，记为 ImpStructArray。

 **②** 从 ImpStructArray 的首个元素开始，将 eip 设置为 call 指令地址，利用单步异常回调函数，遇到长跳指令则记录 API 地址。随后，将 eip 设置为 ImpStructArray 下一元素的 call 指令地址，依次处理所有元素。

 **③** 通过步骤 ① 和 ② 收集 IAT 表信息，随后利用 "vmp0" 节区重建跳转表和 IAT 表，最后将 IAT 表地址回填至对应的 call 指令处，以完成修复。

----------------------

  需要注意，若程序存在增量链接，在判断 call 跳转地址时，目标地址可能仍位于代码区。此时，需进一步检查该跳转地址的第一条指令是否为 jmp 指令。若是，则将 jmp 的跳转地址存入 ImpStructArray 中。此外，在设置 eip 时，需记录 esp 的值，并在程序到达长跳时与当前 esp 值比较。若差值为 -4，则回填时需将地址向低位偏移一个字节。

在执行上述算法后，图 c-00-00 中 call 指令的跳转地址已被修复：

![](https://bbs.kanxue.com/upload/attach/202505/877885_8PUGCUCDXGTB2HZ.png)

最后，再使用 Scylla 插件，在 Dump 保存后，然后点击 Fix Dump，选择刚才 Dump 的文件，完成最终修复。

![](https://bbs.kanxue.com/upload/attach/202505/877885_YHJDXRUDM6DZFQ2.png)

d. 过反调试
=======

00. 绕过思路和步骤
-----------

  VMProtect 3.5.1 版本的反调试功能位于文件目录 vmprotect\runtime\loader.cc 中，具体实现在 loader.cc 的 SetupImage 函数内。该函数涵盖了 Options 配置的所有功能，包括反调试的 Debugger 子选项，详见下图。

![](https://bbs.kanxue.com/upload/attach/202505/877885_SJHEKRAZ6NFFSWD.png)

  在仅启用反调试功能的情况下，SetupImage 函数针对反调试功能，重点关注五个关键区域（见下图）。这些反调试机制的原理将在下一节的反调试测试中详细阐述。

![](https://bbs.kanxue.com/upload/attach/202505/877885_YXVJ7TTSZFASKA6.png)

![](https://bbs.kanxue.com/upload/attach/202505/877885_5NZ4WSTRDQREA8M.png)

### **绕过思路**

  绕过 peb->BeingDebugged 标志位相对简单，但对于 NtQueryInformationProcess 和 NtSetInformationThread 两个函数来说，局部变量 sc_query_information_process 和 sc_set_information_thread 非常关键。如果这两个变量有值，虚拟机（VM）则直接调用系统号；如果为 0，则将通过 LoaderGetProcAddress 函数先获取系统 API 地址，再调用系统 API。在 Windows 11 中，sc_query_information_process 和 sc_set_information_thread 的值分别为 0x03008019 和 0x0200800d。如果要直接调用系统 API，必须将 sc_query_information_process 和 sc_set_information_thread 设置为 0。

  此外，LoaderGetProcAddress 函数也很重要，因为它是查找系统 API 地址的关键函数。如果在调用 LoaderGetProcAddress 函数时，直接将第二个参数设置为 0，返回值必为 0，反调试将可以直接绕过。同样的方法也适用于绕过 CloseHandle。

### **具体步骤**

① 首先，确定 LoaderGetProcAddress 函数的地址。

  为确定 LoaderGetProcAddress 函数在 vm 虚拟机中的位置，需在适当位置进行操作，如下图所示。在 PEB->OSBuildNumber 的内存位置设置硬件访问断点，断点触发后，通过单步调试回调函数进行跟踪。当检测到特征码 06（即退出 VM 环境的标志）时，长跳即为调用 LoaderGetProcAddress 函数的地址。

![](https://bbs.kanxue.com/upload/attach/202505/877885_HDDXXZ7R95NYD6K.png)

② 在 peb->BeingDebugged 的内存位置设置硬件访问断点，断点触发后，将其值修改为 0。

③ 此时，观察右下角 vm 堆栈区域（见下图），将 sc_query_information_process 和 sc_set_information_thread 局部变量的值清零。

![](https://bbs.kanxue.com/upload/attach/202505/877885_QWC577DJV3JBV9Z.png)

④ 随后，在 LoaderGetProcAddress 地址处设置硬件执行断点，按 F9 运行，程序将断下三次，分别对应查询 NtQueryInformationProcess、NtSetInformationThread 和 CloseHandle 函数的地址。每次断下时，将 [esp+4]（即 LoaderGetProcAddress 的第二个参数）的值设为 0。在第三次断下后，禁用所有硬件断点。

⑤ 按 F9 继续运行，程序将在单步异常处中断。点击 “确定” 后，再次按 F9 运行，如下图所示，已成功绕过了反调试机制。

![](https://bbs.kanxue.com/upload/attach/202505/877885_S4TPXH3CKBYJE63.png)

 注：定位 PEB 结构体相对简单。在调试程序时，进入系统入口后，通过命令行执行 mov eax, fs:[0x30]，即可将 PEB 结构体的地址存储到 EAX 寄存器中。随后，再对 BeingDebugged（偏移 +0x2）和 OSBuildNumber（偏移 +0xac）字段进行操作就很容易了。

**01. 反调试测试**
-------------

以下是从 VMProtect 3.5.1 中提取的反调试相关代码，并做了详细的注释，可直接复制用于测试运行：

```
#include #if defined(_WIN32) || defined(_WIN64) // windows平台
#include #include #include #pragma comment(lib, "ntdll.lib")
 
#define STATE_SUCCUSS 0     // 成功状态
 
#define ThreadHideFromDebugger (THREADINFOCLASS)17
#define ProcessDebugPort (PROCESSINFOCLASS)0x7
#define ProcessDebugObjectHandle (PROCESSINFOCLASS)0x1e
#define ProcessDebugFlags (PROCESSINFOCLASS)0x1f                           
#define ProcessDefaultHardErrorMode (PROCESSINFOCLASS)0x0c
#define ProcessInstrumentationCallback (PROCESSINFOCLASS)40
#define SystemModuleInformation (SYSTEM_INFORMATION_CLASS)11
#define SystemKernelDebuggerInformation (SYSTEM_INFORMATION_CLASS)35
 
 
typedef struct _SYSTEM_KERNEL_DEBUGGER_INFORMATION
{
    BOOLEAN DebuggerEnabled;
    BOOLEAN DebuggerNotPresent;
} SYSTEM_KERNEL_DEBUGGER_INFORMATION;
 
 
using namespace std;
bool detect_debugger(); // 检测调试器
 
 
// 程序入口
int main()
{
    if (detect_debugger() == true) {
        cout << "Debugger detected!" << endl;
    }
    else {
        cout << "Not debugged!" << endl;
    }
    std::cin.get(); // 暂停
    return 0;
}
 
 
// 获取API地址
void* GetSystemApi(const char* lpLibFileName, const char* funcName)
{
    // 加载 DLL
    HMODULE hModule = LoadLibraryA(lpLibFileName);
    if (hModule == NULL) {
        return NULL;
    }
 
    // 获取函数地址
    FARPROC Addr = GetProcAddress(hModule, funcName);
    if (Addr == NULL) {
        return NULL;
    }
    return (void*)Addr;
}
 
 
// 获取Teb
void* GetTebAddress()
{
#ifdef _WIN64
    return (void*)__readgsqword(0x30);
#else
    return (void*)__readfsdword(0x18);
#endif
}
 
 
// 获取Peb
void* GetPebAddress()
{
#ifdef _WIN64
    return (void*)__readgsqword(0x60);
#else
    return (void*)__readfsdword(0x30);
#endif
 
}
 
 
 
// 检测peb的BeingDebugged字段是否有值
bool isBeingDebugged_peb()
{
    PEB* peb = (PEB*)GetPebAddress();
    if (peb->BeingDebugged != 0) {
        return true;
    }
    return false;
}
 
 
bool check_NtQueryInformationProcess()
{
    ULONG_PTR status = 0;
    HANDLE ProcHandle;
    ULONG returnLength = 0;
    ULONG_PTR isDebuggerPresent = 0;
    ProcHandle = GetCurrentProcess();
 
    ///=========== 1. 检测ProcessDebugPort ============///
    status = NtQueryInformationProcess(
        ProcHandle,                     // 进程句柄
        ProcessDebugPort,               // 要检索的进程信息类型，ProcessDebugPort：调试器端口号
        &isDebuggerPresent,             // 如果当前被调试，isDebuggerPresent的值为-1
        sizeof(ULONG_PTR),              // 缓冲区大小
        NULL                            // 实际返回进程信息的大小
    );
 
    cout << "返回状态：" << hex << status << endl;
    cout << "isDebuggerPresent:" << hex << isDebuggerPresent << endl;
    if (status == STATE_SUCCUSS && isDebuggerPresent != 0) {
        return true;
    }
 
 
 
    ///========== 2. 检测ProcessDebugObjectHandle ========///
    /**
    * 注意：ProcessDebugObjectHandle作为参数时，如果没有被调试，需要记录两个状态值，
    *      status的值为0xc0000353（64位下是0xFFFFFFFFc0000353),isDebuggerPresent值必须为0
    *
    */
    status = NtQueryInformationProcess(
        ProcHandle,                                 // 进程句柄
        ProcessDebugObjectHandle,                   // 调试句柄
        &isDebuggerPresent,                         // 如果当前被调试，isDebuggerPresent的值不为0
        sizeof(ULONG_PTR),                          // 缓冲区大小
        NULL                                        // 实际返回进程信息的大小
    );
 
    cout << "返回状态：" << hex << status << endl;
    cout << "isDebuggerPresent:" << hex << isDebuggerPresent << endl;
    if (status == STATE_SUCCUSS && isDebuggerPresent != 0) {
        return true;
    }
 
 
 
 
    ///========= 3. 检测ProcessDebugFlags ==========///
#ifdef _WIN64
 
#else
    // 在win11上测试，32位程序能正常运行，64程序总是返回0xFFFFFFFFC0000004的状态码
    status = NtQueryInformationProcess(
        ProcHandle,                     // 进程句柄
        ProcessDebugFlags,              // 调试标志
        &isDebuggerPresent,             // 如果当前被调试，isDebuggerPresent的值为0
        sizeof(ULONG_PTR),              // 缓冲区大小
        &returnLength                   // 实际返回进程信息的大小
    );
 
    if (status == STATE_SUCCUSS && isDebuggerPresent == 0) {
        return true;
    }
    cout << "返回状态：" << hex << status << endl;
    cout << "isDebuggerPresent:" << hex << isDebuggerPresent << endl;
#endif // _WIN64
 
    return false;
}
 
 
 
bool check_NtQuerySystemInformation()
{
    // 注意：此方法仅仅只显示当前系统是否处于调试模式！
    SYSTEM_KERNEL_DEBUGGER_INFORMATION info;
    NTSTATUS status = NtQuerySystemInformation(
        SystemKernelDebuggerInformation,
        &info,
        sizeof(info),
        NULL
    );
 
    if (NT_SUCCESS(status) && info.DebuggerEnabled && !info.DebuggerNotPresent) {
        return true;
    }
 
    return false;
}
 
 
 
bool check_NtSetInformationThread()
{
    // check ThreadHideFromDebugger
    // 说明：① 防止附加。如果附加到调试器，会造成程序退出。
    //      ② 如果该程序正在被调试，且在调用此函数之后，设置断点，也会造成程序崩溃退出。
    //
 
    NTSTATUS status =
        NtSetInformationThread(GetCurrentThread(), ThreadHideFromDebugger, NULL, 0);
    int textVal = 0;
    while (true)
    {
        printf("%d ", textVal);
        Sleep(1000);
        if (textVal++ == 200) {
            break;
        }
    };
 
    return false;
}
 
 
bool check_CloseHandle()
{
    //
    // 注： 只要是无效的句柄就可以，这里使用0xDEADC0DE。如果存在调试器，调用CloseHandle时
    //     会产生一个EXCEPTION_INVALID_HANDLE(0xC0000008)异常,并被__except捕获。如果
    //     没有调试器，CloseHandle会执行失败并返回false。
    //
    __try {
        if (CloseHandle(HANDLE(INT_PTR(0xDEADC0DE)))) {
            return true;
        }
        cout << "CloseHandle failed to execute." << endl;
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        return true;
    }
 
    return false;
}
 
 
 
bool check_writeeflags()
{
 
    //
    // 注： 在调试该程序时，直接运行会在此处触发一个 0x80000004 异常（即单步异常）。按下 F8 键可以逐步执行代码，
    //     但最终会检测到调试器的存在。如果按 F9 让程序继续运行，系统的异常处理流程将接管此异常，并执行 __except
    //     块的内容。在此过程中，程序会检查是否存在硬件断点。如果检测到硬件断点，则表明当前程序正在被调试。
    //
    //     需要注意，在触发单步异常之前，必须禁用所有硬件断点。因为在异常发生时，上下文信息就被保存到了 CONTEXT
    //     结构体中。如果在异常发生后才禁用硬件断点，调试器仍然会被检测到。同理，在异常发生后设置硬件断点，并不会
    //     影响到 CONTEXT 结构体中的内容。
    //
    //
 
    size_t drx;
    uint64_t val;
    CONTEXT* ctx;
    __try {
        __writeeflags(__readeflags() | 0x100); // 设置单步异常
        val = __rdtsc(); // 这条指令可有可无
        __nop();
        return true;
    }
    __except (ctx = (GetExceptionInformation())->ContextRecord,
        drx = (ctx->ContextFlags & CONTEXT_DEBUG_REGISTERS) ? ctx->Dr0 | ctx->Dr1 | ctx->Dr2 | ctx->Dr3 : 0,
        EXCEPTION_EXECUTE_HANDLER) {
        if (drx) {
            return true;
        }
    }
 
    return false;
}
 
 
// ======= 检测调试器 =========== //
bool detect_debugger()
{
 
    // 1.检测peb的BeingDebugged字段是否有值
   /* if (isBeingDebugged_peb() == true) {
        return true;
    }*/
 
 
    // 2.调用NtQueryInformationProcess函数
    /*if (check_NtQueryInformationProcess() == true) {
        return true;
    }*/
 
 
    // 3.调用NtQuerySystemInformation函数
    /*if (check_NtQuerySystemInformation() == true) {
        return true;
    }*/
 
 
    // 4. 检测CC断点
    // 检测特定API的头一个字节是否是0xCC
 
 
    // 5. 调用NtSetInformationThread函数
    //check_NtSetInformationThread();
 
 
    // 6. 调用CloseHandle
    //if (check_CloseHandle() == true) {
    //    return true;
    //}
 
 
    // 7. 设置单步异常
    if (check_writeeflags() == true) {
        return true;
    }
 
    return false;
}
 
#endif 
```

e. 总结
=====

  从以上分析可以看出，VMProtect 3.5.1 版本防护机制已极为变态，尤其是其指令虚拟化保护措施，使得指令还原变得异常困难。因此，我相信，只要 VMProtect 持续更新，其安全性便能得到充分保障，无需担忧程序被破解或分析。分析 vmp 需要消耗大量的时间和资源成本，在成本与收益极度不对称的现实下，几乎无人愿意投入巨大精力去破解受虚拟机保护的程序。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 4 分钟前 被舒默哦编辑 ，原因： 更正

上传的附件：

*   [栈区副本. zip](javascript:void(0);) （9.15kb，0 次下载）