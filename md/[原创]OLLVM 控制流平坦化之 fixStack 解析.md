> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268789.htm)

> [原创]OLLVM 控制流平坦化之 fixStack 解析

在我之前的文章 [基于 LLVM Pass 实现控制流平坦化](https://bbs.pediy.com/thread-266082.htm) 中漏掉了对控制流平坦化里一个很重要的函数`fixStack`的解释。后来我发现`fixStack`这个函数中蕴含的一些原理也是自己在编写 Pass 时需要面对的问题，也是初学 LLVM 最容易迷惑的一个地方，还是决定补充说明一下。

 

首先贴上我重构之后的`fixStack`函数，该函数较 OLLVM 中的`fixStack`函数有所区别，大家自行比较：

```
void llvm::fixStack(Function &F) {
    vector origPHI;
    vector origReg;
    do{
        origPHI.clear();
        origReg.clear();
        BasicBlock &entryBB = F.getEntryBlock();
        for(BasicBlock &BB : F){
            for(Instruction &I : BB){
                if(PHINode *PN = dyn_cast(&I)){
                    origPHI.push_back(PN);
                }else if(!(isa(&I) && I.getParent() == &entryBB)
                    && I.isUsedOutsideOfBlock(&BB)){
                    origReg.push_back(&I);
                }
            }
        }
        for(PHINode *PN : origPHI){
            DemotePHIToStack(PN, entryBB.getTerminator());
        }
        for(Instruction *I : origReg){
            DemoteRegToStack(*I, entryBB.getTerminator());
        }
    }while(!origPHI.empty() || !origReg.empty());
} 
```

从整体上看这个函数处理了两类指令：

*   第一类是 PHI 指令
*   第二类是逃逸变量

什么是 PHI 指令呢，对 LLVM 有所研究的话应该都比较熟悉，控制流平坦化会打乱基本块之间的前后关系，PHI 指令又是基于前驱块的指令，前驱块被打乱了 PHI 指令肯定会出现错误，因此我们要对 PHI 指令进行修复。

 

修复的方法是调用`DemotePHIToStack`这个函数。Demote 是降级的意思，`DemoteToStack`就是用 alloca, store, load 三类内存访问指令来代替之前的 PHI 指令。为什么是 Stack 呢，因为 LLVM IR 中的 alloca 指令是在栈中进行分配的，这点不同于 C 语言中的`malloc`函数。将所有 PHI 指令用三类内存访问指令表示，并将所有 alloca 指令都放在入口块，PHI 指令的错误就被修复了。

 

再来看看第二类情况，逃逸变量指的是在当前基本块中被定义，并且在其他的基本块中被引用了的变量。还是因为控制流平坦化会打乱基本块之间的前后关系，这类逃逸变量在编译（LLVM IR-> 目标平台机器代码）时会出现分不清定义和引用顺序的问题，因此也需要进行修复。

 

修复的方法是调用`DemoteRegToStack`这个函数。Reg 是什么意思呢，这里的 Reg 是寄存器 Register 的缩写，LLVM IR 中所有本地变量都称做 “虚拟寄存器”，所以这里是 DemoteReg。

```
isa(&I) && I.getParent() == &entryBB 
```

这段代码用来判断当前指令是否是 alloca 指令，并且是否位于函数入口块，该类指令不算是逃逸变量，所以不做处理（因为修复逃逸变量就是靠入口块的 alloca 指令和 store, load 指令）。

```
I.isUsedOutsideOfBlock(&BB)

```

这段代码则是判断当前指令是否在除当前基本块以外的基本块被使用，成立则为逃逸变量，需要调用`DemoteRegToStack`函数进行处理。

 

比较 OLLVM 中的`fixStack`函数和我在上面贴的`fixStack`函数后就会发现，OLLVM 中的`fixStack`函数还引用了一个`valueEscape`函数：

```
bool valueEscapes(Instruction *Inst) {
  BasicBlock *BB = Inst->getParent();
  for (Value::use_iterator UI = Inst->use_begin(), E = Inst->use_end(); UI != E;
       ++UI) {
    Instruction *I = cast(*UI);
    if (I->getParent() != BB || isa(I)) {
      return true;
    }
  }
  return false;
}
...
 
        if (!(isa(j) && j->getParent() == bbEntry) &&
            (valueEscapes(&*j) || j->isUsedOutsideOfBlock(&*i))) {
 
... 
```

`valueEscapes`函数是用来判断指令使用的操作数中是否包含逃逸变量以及 PHINode，但事实上修复 PHI 指令和逃逸变量的`DemotePHIToStack`以及`DemoteRegToStack`两个函数在修复当前指令的同时还会修复所有引用了当前指令的指令（详见 llvm/lib/Transforms/Utils/DemoteRegToStack.cpp 源代码），因此我觉得`valueEscape`函数有点多余，不知道我的想法是否正确，如果有误欢迎大家指正。

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#软件保护](forum-4-1-3.htm)