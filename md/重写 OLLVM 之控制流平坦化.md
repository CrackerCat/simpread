> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/345843635)

OLLVM 的控制流平坦化是逆向老哥最讨厌的混淆了，因为一旦使用就代表着 IDA 的 decompiler 彻底报废了。所以了解并在[自己的项目中](https://link.zhihu.com/?target=https%3A//github.com/chenx6/baby_obfuscator)重写控制流平坦化挺重要的。

什么是控制流平坦化
---------

### 名词解释

### BasicBlock

代码块，以跳转语句结尾的一段代码。

### 语言描述控制流平坦化的实现

在 OLLVM 中，Pass 先实现一个永真循环，然后再在这个循环中放入 switch 语句，将代码中除了开始块的所有 BasicBlock 放入这个 switch 语句的不同 case 中，通过修改 switch 的条件，来实现 BasicBlock 之间的跳转。

### 可行的修复方法

在 [利用符号执行去除控制流平坦化](https://zhuanlan.zhihu.com/write#refs) 讲解了恢复控制流平坦化的方法，并且在 [cq674350529/deflat](https://zhuanlan.zhihu.com/write#refs) 使用了 angr 进行实现。

这里引用腾讯 SRC 的思路概括

1.  函数的开始地址为序言的地址
2.  序言的后继为主分发器
3.  后继为主分发器的块为预处理器
4.  后继为预处理器的块为真实块
5.  无后继的块为 retn 块
6.  剩下的为无用块

怎么实现
----

在开始前，先展示一个普通的，带有 `if` 语句的程序的控制流图。

```
+--------------------------------------+
   | entry:                               |
   |  ...                                 |
   |  compare instruction                 |
   |  br %cmp_res, label %if, label %else |
   +-------------------+-----------+------+
                       |           |
         +-------------+           |
         v                         v
+--------+-------+        +--------+-------+
| if:            |        | else:          |
|  br label %end |        |  br label %end |
+--------+-------+        +--------+-------+
         |                         |
         +-----------+-------------+
                     v
                  +--+---+
                  | end: |
                  |  ... |
                  +------+

```

首先，先判断函数的 BasicBlock 数量，如果只有一个 BasicBlock，那么就不用进行混淆了。然后将函数中原有的的 BasicBlock 存放到容器中备用，并排除第一个 BasicBlock，因为第一个 BasicBlock 要做特殊处理。

```
bool runOnFunction(Function &F) override {
  // Only one BB in this Function
  if (F.size() <= 1) {
    return false;
  }

  // Insert All BB into originBB
  SmallVector<BasicBlock *, 0> originBB;
  for (BasicBlock &bb : F) {
    originBB.emplace_back(&bb);
    if (isa<InvokeInst>(bb.getTerminator())) {
      return false;
    }
  }

  // Remove first BB
  originBB.erase(originBB.begin());
}


```

然后，将第一个 BasicBlock 和他结尾的 BranchInst 进行分割，并将第一个 BasicBlock 和后面的 BB 断绝关系。

```
// If firstBB's terminator is BranchInst, then split into two blocks
BasicBlock *firstBB = &*F.begin();
if (BranchInst *br = dyn_cast<BranchInst>(firstBB->getTerminator())) {
  BasicBlock::iterator iter = firstBB->end();
  if (firstBB->size() > 1) {
    --iter;
  }
  BasicBlock *tempBB = firstBB->splitBasicBlock(--iter);
  originBB.insert(originBB.begin(), tempBB);
}

// Remove firstBB
firstBB->getTerminator()->eraseFromParent();


```

经过处理后程序控制流图如下。

```
+--------------------------------------+
   | entry:                               |
   |  ...                                 |
   +--------------------------------------+

   +--------------------------------------+
   | tempBB:                              |
   |  compare instruction                 |
   |  br %cmp_res, label %if, label %else |
   +-------------------+-----------+------+
                       |           |
         +-------------+           |
         v                         v
+--------+-------+        +--------+-------+
| if:            |        | else:          |
|  br label %end |        |  br label %end |
+--------+-------+        +--------+-------+
         |                         |
         +-----------+-------------+
                     v
                  +--+---+
                  | end: |
                  |  ... |
                  +------+

```

接下来就是创建主循环和 switch 语句。先在第一个 BB 处创建 switch 使用的条件变量，然后创建循环的开头 BB，循环结束 BB，switch 的 default 块，最后将他们用 Br 相连起来。

```
// Create main loop
BasicBlock *loopEntry = BasicBlock::Create(F.getContext(), "Entry", &F);
BasicBlock *loopEnd = BasicBlock::Create(F.getContext(), "End", &F);
BasicBlock *swDefault = BasicBlock::Create(F.getContext(), "Default", &F);
// Create switch variable
IRBuilder<> entryBuilder(firstBB, firstBB->end());
AllocaInst *swPtr = entryBuilder.CreateAlloca(entryBuilder.getInt32Ty());
StoreInst *storeRng =
    entryBuilder.CreateStore(entryBuilder.getInt32(rng()), swPtr);
entryBuilder.CreateBr(loopEntry);
// Create switch statement
IRBuilder<> swBuilder(loopEntry);
LoadInst *swVar = swBuilder.CreateLoad(swPtr);
SwitchInst *swInst = swBuilder.CreateSwitch(swVar, swDefault, 0);
BranchInst *dfTerminator = BranchInst::Create(loopEntry, swDefault);
BranchInst *toLoopEnd = BranchInst::Create(loopEntry, loopEnd);


```

经过处理后的程序控制流图如下

```
   +--------------------------------------+            +--------------------------------------+
   | entry:                               |            | tempBB:                              |
   |  ...                                 |            |  compare instruction                 |
   +-------------------+------------------+            |  br %cmp_res, label %if, label %else |
                       |                               +-------------------+-----------+------+
                       v v---------------------+                           |           |
      +----------------+-----------------+     |             +-------------+           |
      | Entry:                           |     |             v                         v
      |  switch i32 %al, label %Default  |     |    +--------+-------+        +--------+-------+
      +----------------+-----------------+     |    | if:            |        | else:          |
                       |                       |    |  br label %end |        |  br label %end |
       v---------------+                       |    +--------+-------+        +--------+-------+
+----------------+                             |             |                         |
| Default:       |                             |             +-----------+-------------+
|  br label %End |                             |                         v
+------+---------+                             |                      +--+---+
       |                                       |                      | end: |
       +-----------------v                     |                      |  ... |
                 +------------------+          |                      +------+
                 | End:             |          |
                 |  br label %Entry |          |
                 +--------+---------+          |
                          |                    |
                          |                    |
                          +--------------------+

```

然后就是重头戏了：将 BB 们放入 switch 的 case 中。首先先将循环的结尾移动到 BB 后，然后再放入 case 中。然后再判断 BB 的结尾的语句有多少个继承块，如果为 0 个的话，说明是返回语句，那么就不需要管；如果是 1 个的话，说明是无条件跳转语句，那么就计算与其相连的下一个块的 case 值，并更新 switch 的条件变量的值；如果为 2 个的话，说明是一个条件跳转，则根据条件语句 SelectInst 决定下一个块执行的位置; 如果是其他情况，则保持该 BB 不变。最后更新下初始的 switch 条件变量的值，保证第一个块的执行。

```
// Put all BB into switch Instruction
for (BasicBlock *bb : originBB) {
  bb->moveBefore(loopEnd);
  swInst->addCase(swBuilder.getInt32(rng()), bb);
}

// Recalculate switch Instruction
for (BasicBlock *bb : originBB) {
  switch (bb->getTerminator()->getNumSuccessors()) {
  case 0:
    // No terminator
    break;
  case 1: {
    // Terminator is a non-condition jump
    Instruction *terminator = bb->getTerminator();
    BasicBlock *sucessor = terminator->getSuccessor(0);
    // Find sucessor's case condition
    ConstantInt *caseNum = swInst->findCaseDest(sucessor);
    if (caseNum == nullptr) {
      caseNum = swBuilder.getInt32(rng());
    }
    // Connect this BB to sucessor
    IRBuilder<> caseBuilder(bb, bb->end());
    caseBuilder.CreateStore(caseNum, swPtr);
    caseBuilder.CreateBr(loopEnd);
    terminator->eraseFromParent();
  } break;
  case 2: {
    // Terminator is a condition jump
    Instruction *terminator = bb->getTerminator();
    ConstantInt *trueCaseNum =
        swInst->findCaseDest(terminator->getSuccessor(0));
    ConstantInt *falseCaseNum =
        swInst->findCaseDest(terminator->getSuccessor(1));
    if (trueCaseNum == nullptr) {
      trueCaseNum = swBuilder.getInt32(rng());
    }
    if (falseCaseNum == nullptr) {
      falseCaseNum = swBuilder.getInt32(rng());
    }
    IRBuilder<> caseBuilder(bb, bb->end());
    if (BranchInst *endBr = dyn_cast<BranchInst>(bb->getTerminator())) {
      // Select the next BB to be executed
      Value *selectInst = caseBuilder.CreateSelect(
          endBr->getCondition(), trueCaseNum, falseCaseNum);
      caseBuilder.CreateStore(selectInst, swPtr);
      caseBuilder.CreateBr(loopEnd);
      terminator->eraseFromParent();
    }
  } break;
  }
}

// Set swVar's origin value, let the first BB executed first
ConstantInt *caseCond = swInst->findCaseDest(*originBB.begin());
storeRng->setOperand(0, caseCond);


```

> 图请参考 [Obfuscator-llvm 源码分析](https://zhuanlan.zhihu.com/write#refs)，用 ASCII 画图有点麻烦...

总结
--

这个控制流平坦化的代码数量也不多，而且程序的逻辑在理清后很容易理解，所以文章的篇幅很短。

refs
----

[chenx6/baby_obfuscator](https://link.zhihu.com/?target=https%3A//github.com/chenx6/baby_obfuscator)

[Obfuscator-llvm 源码分析](https://link.zhihu.com/?target=https%3A//sq.163yun.com/blog/article/175307579596922880)

[obfuscator-llvm/obfuscator](https://link.zhihu.com/?target=https%3A//github.com/obfuscator-llvm/obfuscator)

[利用符号执行去除控制流平坦化](https://link.zhihu.com/?target=https%3A//security.tencent.com/index.php/blog/msg/112)

[cq674350529/deflat](https://link.zhihu.com/?target=https%3A//github.com/cq674350529/deflat)