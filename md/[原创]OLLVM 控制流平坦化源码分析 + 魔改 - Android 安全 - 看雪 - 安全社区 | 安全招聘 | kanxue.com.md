> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278213.htm)

> [原创]OLLVM 控制流平坦化源码分析 + 魔改

目录

*   [前置知识](#前置知识)
*   [Flattening::flatten(Function *f)](#flatteningflattenfunction-f)
*   [源代码](#源代码)
*   [目前代码](#目前代码)
*   [创建 switch](#创建switch)
*   [目前代码：](#目前代码：)
*   [创建 case](#创建case)
*   [目前代码](#目前代码-1)
*   [枚举更改各个 case block 块](#枚举更改各个case-block块)
*            [return block](#return-block)
*            [非条件跳转 block](#非条件跳转block)
*            [条件跳转 block](#条件跳转-block)
*   [目前代码](#目前代码-2)
*   [魔改平坦化](#魔改平坦化)
*   [尾声](#尾声)

前置知识
====

Module 是指模块，Function 模块下的函数，BasicBlock 函数下的基本块，Instruction 基本块下的 IR 指令

Flattening::flatten(Function *f)
================================

```
for (Function::iterator i = f->begin(); i != f->end(); ++i) {
   BasicBlock *tmp = &*i;
   origBB.push_back(tmp);
 
   BasicBlock *bb = &*i;
   if (isa(bb->getTerminator())) {
     return false;
   }
 } 
```

把函数分成很多个基本块，并且 push 到 vector 类型的 origBB 中。  
判断里面基本块是否大于 1，不大于 1 的话就没有意义去进行混淆：

```
if (origBB.size() <= 1) {
   return false;
 }

```

需要把 vertor 里面的第一个基本块即入口基本块单独拿出来进行处理：对入口基本块进行判断，如果是无条件跳转则不进行任何处理，否则需要找到最后一条指令，将整个 if 结构给 split，split 之后两个块之间会自动添加跳转指令，然后就可以把原来的 split 后的 if 结构给它扔进要处理的基本块列表。

```
origBB.erase(origBB.begin());
 
// Get a pointer on the first BB
Function::iterator tmp = f->begin(); //++tmp;
BasicBlock *insert = &*tmp;
 
// If main begin with an if
BranchInst *br = NULL;
if (isa(insert->getTerminator())) {
  br = cast(insert->getTerminator());
}
 
if ((br != NULL && br->isConditional()) ||
    insert->getTerminator()->getNumSuccessors() > 1) {
  BasicBlock::iterator i = insert->end();
  --i;
 
  if (insert->size() > 1) {
    --i;
  }
 
  BasicBlock *tmpBB = insert->splitBasicBlock(i, "first");
  origBB.insert(origBB.begin(), tmpBB);
} 
```

如果是条件跳转的话这里是把上面自动添加那个跳转指令给删除，如果不是的话，那么也是需要把它删除，因为跳转点目标还不能确定：

```
// Remove jump
  insert->getTerminator()->eraseFromParent();

```

源代码
===

```
%retval = alloca i32, align 4
 %argc.addr = alloca i32, align 4
 %argv.addr = alloca i8**, align 8
 %a = alloca i32, align 4
 store i32 0, i32* %retval, align 4
 store i32 %argc, i32* %argc.addr, align 4
 store i8** %argv, i8*** %argv.addr, align 8
 %0 = load i8**, i8*** %argv.addr, align 8
 %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
 %1 = load i8*, i8** %arrayidx, align 8
 %call = call i32 @atoi(i8* %1) #3
 store i32 %call, i32* %a, align 4
 %2 = load i32, i32* %a, align 4
 br label %NodeBlock8

```

目前代码
====

```
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32

```

创建一个 switchvar 变量，然后去获取一个随机整数创建 store 指令塞给 switchvar 中

```
switchVar =
     new AllocaInst(Type::getInt32Ty(f->getContext()), 0, "switchVar", insert);
 new StoreInst(
     ConstantInt::get(Type::getInt32Ty(f->getContext()),
                      llvm::cryptoutils->scramble32(0, scrambling_key)),
     switchVar, insert);

```

也就是在 switchvar 添了如下这一行：

```
store i32 157301900, i32* %switchVar

```

创建 switch
=========

创建两个 block，其它的基本块插入它们之间

```
loopEntry = BasicBlock::Create(f->getContext(), "loopEntry", f, insert);
loopEnd = BasicBlock::Create(f->getContext(), "loopEnd", f, insert);

```

如下：

```
loopEntry:                                     
  
loopEnd:                                         

```

目标基本块里面啥内容也没有。

在 loopEntry 里面新建一个 load 指令，并且把 switchVar：

```
load = new LoadInst(switchVar, "switchVar", loopEntry);

```

目前 loopentry 指令如下：

```
loopEntry:                                        ; preds = %entry, %loopEnd
  %switchVar10 = load i32, i32* %switchVar

```

把 insert 插入到 loopEntry 之前，这里的 insert 就是 entry 基本块，再创建两个跳转指令，从 insert（即第一个基本块）跳转到 loopEntry；从 loopend 跳转到 loopEntry：

```
// Move first BB on top
insert->moveBefore(loopEntry);
BranchInst::Create(loopEntry, insert);
// loopEnd jump to loopEntry
BranchInst::Create(loopEntry, loopEnd);

```

这里结束后，entry 模块就完整了，如下：

```
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
  store i32 157301900, i32* %switchVar
  br label %loopEntry

```

而 loopend 模块也有了一条指令（其实也是完整了）：

```
loopEnd:                                       
br label %loopEntry

```

紧接着创建一个基本块，然后在基本块里面创建一个跳转指令，从 switchDefault 跳转到 loopend 中：

```
BasicBlock *swDefault =
     BasicBlock::Create(f->getContext(), "switchDefault", f, loopEnd);
 BranchInst::Create(loopEnd, swDefault);

```

多了一个 switchDefault 基本块，指令如下：

```
switchDefault:                                    ; preds = %loopEntry
  br label %loopEnd

```

创建一个 switch 指令，位置是在 loopentry 基本块下，且创建了 0 个 case，然后设置了条件为 load，就上面的 load。

```
switchI = SwitchInst::Create(&*f->begin(), swDefault, 0, loopEntry);
 switchI->setCondition(load);

```

把 entry 最后一行跳转指令删除后再创建了一个跳转指令，从 entry 跳转到 loopentry：

```
f->begin()->getTerminator()->eraseFromParent();
  BranchInst::Create(loopEntry, &*f->begin());

```

```
for (std::vector::iterator b = origBB.begin();
     b != origBB.end(); ++b) {
  BasicBlock *i = *b;
  ConstantInt *numCase = NULL;
 
  // Move the BB inside the switch (only visual, no code logic)
  i->moveBefore(loopEnd);
 
  // Add case to switch
  numCase = cast(ConstantInt::get(
      switchI->getCondition()->getType(),
      llvm::cryptoutils->scramble32(switchI->getNumCases(), scrambling_key)));
  switchI->addCase(numCase, i);
} 
```

目前代码：
=====

```
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
  store i32 157301900, i32* %switchVar
  br label %loopEntry
 
loopEntry:    
%switchVar10 = load i32, i32* %switchVar
  switch i32 %switchVar10, label %switchDefault [
  ]
 
switchDefault:                                    ; preds = %loopEntry
  br label %loopEnd
 
loopEnd:                                       
br label %loopEntry

```

创建 case
=======

```
for (std::vector::iterator b = origBB.begin();
     b != origBB.end(); ++b) {
  BasicBlock *i = *b;
  ConstantInt *numCase = NULL;
 
  // Move the BB inside the switch (only visual, no code logic)
  i->moveBefore(loopEnd);
 
  // Add case to switch
  numCase = cast(ConstantInt::get(
      switchI->getCondition()->getType(),
      llvm::cryptoutils->scramble32(switchI->getNumCases(), scrambling_key)));
  switchI->addCase(numCase, i);
} 
```

这里的 i 就是指剩下的那些 case 分支代码基本块，i->moveBefore(loopEnd)，把某个代码基本块置于 loopend 之前。比如某个基本块是这样：

```
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  br i1 %Pivot9, label %LeafBlock, label %NodeBlock

```

然后下面的这些代码就是创建一个 numcase，就是 case 分支里面的 case 值，这个值它是随机生成的，种子的话是 Entry.cpp 里面的那个 AesSeed 值，如果确定 AesSeed 的话，那么这里随机生成的 case 每次都是固定的。  
switchI->addCase(numCase, i); 紧接着在 switch 里面增加一个 case 值，跳转到 NodeBlock8 里面。  
目前 switch 执行完一次后，loopentry 基本 bolck 块如下：

```
loopEntry:    
%switchVar10 = load i32, i32* %switchVar
  switch i32 %switchVar10, label %switchDefault [
   i32 157301900, label %NodeBlock8
  ]

```

当循环执行结束后：

目前代码
====

```
entry:
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  br label %NodeBlock8
 
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  br i1 %Pivot9, label %LeafBlock, label %NodeBlock
 
NodeBlock:                                        ; preds = %NodeBlock8
  %Pivot = icmp slt i32 %2, 3
  br i1 %Pivot, label %sw.bb2, label %LeafBlock6
 
LeafBlock6:                                       ; preds = %NodeBlock
  %SwitchLeaf7 = icmp eq i32 %2, 3
  br i1 %SwitchLeaf7, label %sw.bb4, label %NewDefault
 
LeafBlock:                                        ; preds = %NodeBlock8
  %SwitchLeaf = icmp eq i32 %2, 1
  br i1 %SwitchLeaf, label %sw.bb, label %NewDefault
 
sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  br label %sw.epilog
 
sw.bb2:                                           ; preds = %NodeBlock
  %call3 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i64 0, i64 0))
  br label %sw.epilog
 
sw.bb4:                                           ; preds = %LeafBlock6
  %call5 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.2, i64 0, i64 0))
  br label %sw.epilog
 
NewDefault:                                       ; preds = %LeafBlock6, %LeafBlock
  br label %sw.default
 
sw.default:                                       ; preds = %NewDefault
  br label %sw.epilog
 
sw.epilog:                                        ; preds = %sw.default, %sw.bb4, %sw.bb2, %sw.bb
  %3 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %3, 0
  br i1 %cmp, label %if.then, label %if.else
 
if.then:                                          ; preds = %sw.epilog
  store i32 1, i32* %retval, align 4
  br label %return
 
if.else:                                          ; preds = %sw.epilog
  store i32 10, i32* %retval, align 4
  br label %return
 
return:                                           ; preds = %if.else, %if.then
  %4 = load i32, i32* %retval, align 4
  ret i32 %4
   
loopEnd:                                          ; preds = %if.else, %if.then, %sw.epilog, %sw.default, %NewDefault, %sw.bb4, %sw.bb2, %sw.bb, %LeafBlock, %LeafBlock6, %NodeBlock, %NodeBlock8, %switchDefault
  br label %loopEntry
 
}

```

枚举更改各个 case block 块
===================

return block
------------

```
// Ret BB
   if (i->getTerminator()->getNumSuccessors() == 0) {
     continue;
   }

```

getNumSuccessors 是获取后续 BB 的个数，Ret BB 后继 BB 为 0 个（判断分支），直接 continue

非条件跳转 block
-----------

```
// If it's a non-conditional jump
   if (i->getTerminator()->getNumSuccessors() == 1) {
     // Get successor and delete terminator
     BasicBlock *succ = i->getTerminator()->getSuccessor(0);
     i->getTerminator()->eraseFromParent();
 
     // Get next case
     numCase = switchI->findCaseDest(succ);
 
     // If next case == default case (switchDefault)
     if (numCase == NULL) {
       numCase = cast(
           ConstantInt::get(switchI->getCondition()->getType(),
                            llvm::cryptoutils->scramble32(
                                switchI->getNumCases() - 1, scrambling_key)));
     }
 
     // Update switchVar and jump to the end of loop
     new StoreInst(numCase, load->getPointerOperand(), i);
     BranchInst::Create(loopEnd, i);
     continue;
   } 
```

如果后面只有一个分支的话，那么先判断分支是否能够找到，不为 null 后先去根据原来条件去创建一个 store 指令，然后创建一个跳转指令跳转到 loopend，再把原来跳转指令抹去。  
原来：

```
sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  br label %sw.epilog

```

改变后：

```
sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  store i32 387774014, i32* %switchVar
  br label %loopEnd

```

条件跳转 block
----------

```
if (i->getTerminator()->getNumSuccessors() == 2) {
    // Get next cases
    ConstantInt *numCaseTrue =
        switchI->findCaseDest(i->getTerminator()->getSuccessor(0));
    ConstantInt *numCaseFalse =
        switchI->findCaseDest(i->getTerminator()->getSuccessor(1));
 
    // Check if next case == default case (switchDefault)
    if (numCaseTrue == NULL) {
      numCaseTrue = cast(
          ConstantInt::get(switchI->getCondition()->getType(),
                           llvm::cryptoutils->scramble32(
                               switchI->getNumCases() - 1, scrambling_key)));
    }
 
    if (numCaseFalse == NULL) {
      numCaseFalse = cast(
          ConstantInt::get(switchI->getCondition()->getType(),
                           llvm::cryptoutils->scramble32(
                               switchI->getNumCases() - 1, scrambling_key)));
    }
 
    // Create a SelectInst
    BranchInst *br = cast(i->getTerminator());
    SelectInst *sel =
        SelectInst::Create(br->getCondition(), numCaseTrue, numCaseFalse, "",
                           i->getTerminator());
 
    // Erase terminator
    i->getTerminator()->eraseFromParent();
 
    // Update switchVar and jump to the end of loop
    new StoreInst(sel, load->getPointerOperand(), i);
    BranchInst::Create(loopEnd, i);
    continue;
  } 
```

首先会把两个跳转分支都取出来，先判断两个分支是否都能够找到，如果都不为 null 的话，那么取出原来的跳转指令，根据 br 的两个分支条件，去创建一个 SelectInst 然后再删除原来指令，创建一个 store 指令，再去创建一个跳转指令跳转到 loopend。  
原来：

```
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  br i1 %Pivot9, label %LeafBlock, label %NodeBlock

```

改变后：

```
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  %3 = select i1 %Pivot9, i32 -1519555718, i32 241816174
  store i32 %3, i32* %switchVar
  br label %loopEnd

```

目前代码
====

```
define dso_local i32 @main(i32 %argc, i8** %argv) #0 {
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
  store i32 157301900, i32* %switchVar
  br label %loopEntry
 
loopEntry:                                        ; preds = %entry, %loopEnd
  %switchVar10 = load i32, i32* %switchVar
  switch i32 %switchVar10, label %switchDefault [
    i32 157301900, label %NodeBlock8
    i32 241816174, label %NodeBlock
    i32 1003739776, label %LeafBlock6
    i32 -1519555718, label %LeafBlock
    i32 -749093422, label %sw.bb
    i32 1599617141, label %sw.bb2
    i32 1815329037, label %sw.bb4
    i32 1738940479, label %NewDefault
    i32 -282945350, label %sw.default
    i32 387774014, label %sw.epilog
    i32 1681741611, label %if.then
    i32 347219667, label %if.else
    i32 -618048859, label %return
  ]
 
switchDefault:                                    ; preds = %loopEntry
  br label %loopEnd
 
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  %3 = select i1 %Pivot9, i32 -1519555718, i32 241816174
  store i32 %3, i32* %switchVar
  br label %loopEnd
 
NodeBlock:                                        ; preds = %NodeBlock8
  %Pivot = icmp slt i32 %2, 3
  %4 = select i1 %Pivot, i32 1599617141, i32 1003739776
  store i32 %4, i32* %switchVar
  br label %loopEnd
 
LeafBlock6:                                       ; preds = %NodeBlock
  %SwitchLeaf7 = icmp eq i32 %2, 3
  %5 = select i1 %SwitchLeaf7, i32 1815329037, i32 1738940479
  store i32 %5, i32* %switchVar
  br label %loopEnd
 
LeafBlock:                                        ; preds = %NodeBlock8
  %SwitchLeaf = icmp eq i32 %2, 1
  %6 = select i1 %SwitchLeaf, i32 -749093422, i32 1738940479
  store i32 %6, i32* %switchVar
  br label %loopEnd
 
sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  store i32 387774014, i32* %switchVar
  br label %loopEnd
 
sw.bb2:                                           ; preds = %NodeBlock
  %call3 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i64 0, i64 0))
  br label %sw.epilog
 
sw.bb4:                                           ; preds = %LeafBlock6
  %call5 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.2, i64 0, i64 0))
  store i32 387774014, i32* %switchVar
  br label %loopEnd
 
NewDefault:                                       ; preds = %LeafBlock6, %LeafBlock
  br label %sw.default
 
sw.default:                                       ; preds = %NewDefault
  store i32 387774014, i32* %switchVar
  br label %loopEnd
 
sw.epilog:                                        ; preds = %sw.default, %sw.bb4, %sw.bb2, %sw.bb
  %3 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %3, 0
  %8 = select i1 %cmp, i32 1681741611, i32 347219667
  store i32 %8, i32* %switchVar
  br label %loopEnd
 
if.then:                                          ; preds = %sw.epilog
  store i32 1, i32* %retval, align 4
  store i32 -618048859, i32* %switchVar
  br label %loopEnd
 
if.else:                                          ; preds = %sw.epilog
  store i32 10, i32* %retval, align 4
  store i32 -618048859, i32* %switchVar
  br label %loopEnd
 
return:                                           ; preds = %if.else, %if.then
  %4 = load i32, i32* %retval, align 4
  ret i32 %4
 
loopEnd:                                          ; preds = %if.else, %if.then, %sw.epilog, %sw.default, %NewDefault, %sw.bb4, %sw.bb2, %sw.bb, %LeafBlock, %LeafBlock6, %NodeBlock, %NodeBlock8, %switchDefault
  br label %loopEntry
}

```

附带一句，进行控制流平坦化之前刚开始会把 switch 语句给它全部进行改为 if else 目的主要是为了进行多次平坦化做准备（进行平坦化时里面是可以填次数）

魔改平坦化
=====

正常流程就是首先进入 entry，entry 会给一个 case 常量值，然后进入 loopentry，loopentry 根据这个常量值进行 switch 分跳转到 case 常量对应的基本块，基本块执行完又会赋值一个 case 常量值，跳转到 loopend，loopend 又会跳转到 loopentry 进行下一次分发。（感觉特征点的话就是 entry 和 loopend 都会跳转到 loopentry）。

大部分的方法都是基于定位 loopend 基本块，这种简单的 IR 结构编译成汇编代码也是这个样子，很容易定位到 loopend 基本块。在这里，我们的魔改方法就直接在 loopend 基本块后面加一个新的 switchInst，然后跳转到基本块，相当于从开始的 switch 转移到后面来了。即抹除原来的跳转，插入 loadInst 加载 swichvar，再用于 swichInst 跳转。

尾声
==

网上很多很多现成资料，我这个也是自己看了再总结的，如果哪里不对的话，还请多多指教。

[议题征集启动！看雪 · 第七届安全开发者峰会 10 月 23 日上海](https://bbs.kanxue.com/thread-276136.htm)

最后于 2 小时前 被寻梦之璐编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#源码框架](forum-161-1-127.htm)