> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278229.htm)

> [原创]OLLVM 虚假控制流源码分析

[原创]OLLVM 虚假控制流源码分析

8 小时前 330

### [原创]OLLVM 虚假控制流源码分析

 [![](http://passport.kanxue.com/upload/avatar/635/917635.png?1690713154)](user-home-917635.htm) [寻梦之璐](user-home-917635.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif) 8 小时前  330

目录

*   [runOnFunction 函数](#runonfunction函数)
*   [bogus 函数](#bogus函数)
*   [目前源码：](#目前源码：)
*   [addBogusFlow 函数 1](#addbogusflow函数1)
*   [createAlteredBasicBlock 函数](#createalteredbasicblock函数)
*            [原基本块：](#原基本块：)
*            [copy 的基本块：](#copy的基本块：)
*   [addBogusFlow 函数 2](#addbogusflow函数2)
*   [尾声](#尾声)

runOnFunction 函数
================

```
if (ObfTimes <= 0) {
        errs()<<"BogusControlFlow application number -bcf_loop=x must be x > 0";
        return false;
      }
if ( !((ObfProbRate > 0) && (ObfProbRate <= 100)) ) {
        errs()<<"BogusControlFlow application basic blocks percentage -bcf_prob=x must be 0 < x <= 100";
        return false;
      }

```

```
const int defaultObfRate = 30, defaultObfTime = 1;
 
static cl::opt ObfProbRate("bcf_prob", cl::desc("Choose the probability [%] each basic blocks will be obfuscated by the -bcf pass"), cl::value_desc("probability rate"), cl::init(defaultObfRate), cl::Optional);
 
static cl::opt ObfTimes("bcf_loop", cl::desc("Choose how many time the -bcf pass loop on a function"), cl::value_desc("number of times"), cl::init(defaultObfTime), cl::Optional); 
```

这里的 ObfTimes 对应过来的默认值就是 1，对函数进行混淆的次数，ObfProbRate 就是 30，对基本块进行混淆的概率，分别是 opt 时传入的参数。

```
// check for compatible
  for (BasicBlock &bb : F.getBasicBlockList()) {
    if (isa(bb.getTerminator())) {
      return false;
    }
  } 
```

枚举这个函数的所有基本块，如果这个函数后面基本块中含有 invoke（调用了某个函数），它就不执行了，就退出这个基本块。

```
if(toObfuscate(flag,&F,"bcf")) {
        bogus(F);
        doF(*F.getParent());
        return true;
      }

```

判断有没有 bcf 也就是虚假控制流，有的话就进入。

bogus 函数
========

```
if(ObfProbRate < 0 || ObfProbRate > 100){
       DEBUG_WITH_TYPE("opt", errs() << "bcf: Incorrect value,"
           << " probability rate set to default value: "
           << defaultObfRate <<" \n");
       ObfProbRate = defaultObfRate;
     }
if(ObfTimes <= 0){
       DEBUG_WITH_TYPE("opt", errs() << "bcf: Incorrect value,"
           << " must be greater than 1. Set to default: "
           << defaultObfTime <<" \n");
       ObfTimes = defaultObfTime;
     }

```

首先进行判断这个次数和概率，是否符合条件，不符合的话会进行设置默认值。  
紧接着就是一个大型的 do while 循环里面包含着的代码：

```
std::list basicBlocks;
 for (Function::iterator i=F.begin();i!=F.end();++i) {
   basicBlocks.push_back(&*i);
 } 
```

把所有的基本块放在 basicblock list 里面。  
获取一个随机值，如果符合的话就进入:

```
if((int)llvm::cryptoutils->get_range(100) <= ObfProbRate){
             DEBUG_WITH_TYPE("opt", errs() << "bcf: Block "
                 << NumBasicBlocks <<" selected. \n");
             hasBeenModified = true;
             ++NumModifiedBasicBlocks;
             NumAddedBasicBlocks += 3;
             FinalNumBasicBlocks += 3;
             // Add bogus flow to the given Basic Block (see description)
             BasicBlock *basicBlock = basicBlocks.front();
             addBogusFlow(basicBlock, F);
           }else{
             DEBUG_WITH_TYPE("opt", errs() << "bcf: Block "
                 << NumBasicBlocks <<" not selected.\n");
           }

```

每个基本块都有 ObfProbRate 的概率被混淆，即基本块调用了 addBogusFlow 函数。  
这个函数的作用就是对指定函数的每个基本块以 ObfProbRate 的概率去进行调用函数混淆。

目前源码：
=====

```
define dso_local i32 @main(i32 %argc, i8** %argv) #0 {
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
  %call = call i32 @atoi(i8* %1) #2
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %2, 0
  br i1 %cmp, label %if.then, label %if.else
 
if.then:                                          ; preds = %entry
  store i32 1, i32* %retval, align 4
  br label %return
 
if.else:                                          ; preds = %entry
  store i32 10, i32* %retval, align 4
  br label %return
 
return:                                           ; preds = %if.else, %if.then
  %3 = load i32, i32* %retval, align 4
  ret i32 %3
}

```

addBogusFlow 函数 1
=================

```
Instruction *i1 = &*basicBlock->begin();
     if(basicBlock->getFirstNonPHIOrDbgOrLifetime())
       i1 = basicBlock->getFirstNonPHIOrDbgOrLifetime();
     Twine *var;
     var = new Twine("originalBB");
     BasicBlock *originalBB = basicBlock->splitBasicBlock(i1, *var);

```

执行完后这里的 basicBlock 指令是 br label %originalBB，而 originalBB 目前代码块如下：

```
originalBB：
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
  %call = call i32 @atoi(i8* %1) #2
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %2, 0
  br i1 %cmp, label %if.then, label %if.else

```

而之前的 entry 就是目前 basicblock：

```
entry：
br label %originalBB

```

紧接着：

```
Twine * var3 = new Twine("alteredBB");
     BasicBlock *alteredBB = createAlteredBasicBlock(originalBB, *var3, &F);

```

createAlteredBasicBlock 会把这个 originalBB 进行克隆

createAlteredBasicBlock 函数
==========================

```
virtual BasicBlock* createAlteredBasicBlock(BasicBlock * basicBlock,
    const Twine &  Name = "gen", Function * F = 0){
  // Useful to remap the informations concerning instructions.
  ValueToValueMapTy VMap;
  BasicBlock * alteredBB = llvm::CloneBasicBlock (basicBlock, VMap, Name, F);
 
 
    // Remap attached metadata.
    SmallVector, 4> MDs;
    i->getAllMetadata(MDs);
    // important for compiling with DWARF, using option -g.
    i->setDebugLoc(ji->getDebugLoc());
    ji++;
  } // The instructions' informations are now all correct 
```

这里的代码的话主要解决两个问题，就 cloneBasicBlock 函数进行的克隆并不是完全的克隆，第一它不会对操作数进行替换，比如：

```
orig:
  %a = ...
  %b = fadd %a, ...
  
clone:
  %a.clone = ...
  %b.clone = fadd %a, ... ; Note that this references the old %a and
not %a.clone!

```

在 clone 出来的基本块中，fadd 指令的操作数不是 %a.clone，而是 a%，所以之后要通过 VMap 对所有操作数进行映射，使其恢复正常：

```
BasicBlock::iterator ji = basicBlock->begin();
for (BasicBlock::iterator i = alteredBB->begin(), e = alteredBB->end() ; i != e; ++i){
  // Loop over the operands of the instruction
  for(User::op_iterator opi = i->op_begin (), ope = i->op_end(); opi != ope; ++opi){
    // get the value for the operand
    Value *v = MapValue(*opi, VMap,  RF_NoModuleLevelChanges, 0);
    if (v != 0){
      *opi = v;
    }
  }

```

第二，它不会对 PHI Node 进行任何处理，PHI Node 的前驱块仍是原始基本块的前驱块，但是新克隆出来的基本块没有任何前驱块，所以要对 PHI Node 的前驱块进行 remap：

```
// Remap phi nodes' incoming blocks.
        if (PHINode *pn = dyn_cast(i)) {
          for (unsigned j = 0, e = pn->getNumIncomingValues(); j != e; ++j) {
            Value *v = MapValue(pn->getIncomingBlock(j), VMap, RF_None, 0);
            if (v != 0){
              pn->setIncomingBlock(j, cast(v));
            }
          }
        } 
```

往基本块里面添加一些没用的赋值指令，或者修改 cmp 的条件，binaryop 大概指的是 add，mul，cmp 这类运算指令

```
for (BasicBlock::iterator i = alteredBB->begin(), e = alteredBB->end() ; i != e; ++i){
       // in the case we find binary operator, we modify slightly this part by randomly
       // insert some instructions
       if(i->isBinaryOp()){ // binary instructions
         unsigned opcode = i->getOpcode();
         BinaryOperator *op, *op1 = NULL;
         Twine *var = new Twine("_");
         // treat differently float or int
         // Binary int
         if(opcode == Instruction::Add || opcode == Instruction::Sub ||
             opcode == Instruction::Mul || opcode == Instruction::UDiv ||
             opcode == Instruction::SDiv || opcode == Instruction::URem ||
             opcode == Instruction::SRem || opcode == Instruction::Shl ||
             opcode == Instruction::LShr || opcode == Instruction::AShr ||
             opcode == Instruction::And || opcode == Instruction::Or ||
             opcode == Instruction::Xor){
           for(int random = (int)llvm::cryptoutils->get_range(10); random < 10; ++random){
             switch(llvm::cryptoutils->get_range(4)){ // to improve
               case 0: //do nothing
                 break;
               case 1: op = BinaryOperator::CreateNeg(i->getOperand(0),*var,&*i);
                       op1 = BinaryOperator::Create(Instruction::Add,op,
                           i->getOperand(1),"gen",&*i);
                       break;
               case 2: op1 = BinaryOperator::Create(Instruction::Sub,
                           i->getOperand(0),
                           i->getOperand(1),*var,&*i);
                       op = BinaryOperator::Create(Instruction::Mul,op1,
                           i->getOperand(1),"gen",&*i);
                       break;
               case 3: op = BinaryOperator::Create(Instruction::Shl,
                           i->getOperand(0),
                           i->getOperand(1),*var,&*i);
                       break;
             }
           }
         }
         // Binary float
         if(opcode == Instruction::FAdd || opcode == Instruction::FSub ||
             opcode == Instruction::FMul || opcode == Instruction::FDiv ||
             opcode == Instruction::FRem){
           for(int random = (int)llvm::cryptoutils->get_range(10); random < 10; ++random){
             switch(llvm::cryptoutils->get_range(3)){ // can be improved
               case 0: //do nothing
                 break;
               case 1: op = BinaryOperator::CreateFNeg(i->getOperand(0),*var,&*i);
                       op1 = BinaryOperator::Create(Instruction::FAdd,op,
                           i->getOperand(1),"gen",&*i);
                       break;
               case 2: op = BinaryOperator::Create(Instruction::FSub,
                           i->getOperand(0),
                           i->getOperand(1),*var,&*i);
                       op1 = BinaryOperator::Create(Instruction::FMul,op,
                           i->getOperand(1),"gen",&*i);
                       break;
             }
           }
         }
         if(opcode == Instruction::ICmp){ // Condition (with int)
           ICmpInst *currentI = (ICmpInst*)(&i);
           switch(llvm::cryptoutils->get_range(3)){ // must be improved
             case 0: //do nothing
               break;
             case 1: currentI->swapOperands();
                     break;
             case 2: // randomly change the predicate
                     switch(llvm::cryptoutils->get_range(10)){
                       case 0: currentI->setPredicate(ICmpInst::ICMP_EQ);
                               break; // equal
                       case 1: currentI->setPredicate(ICmpInst::ICMP_NE);
                               break; // not equal
                       case 2: currentI->setPredicate(ICmpInst::ICMP_UGT);
                               break; // unsigned greater than
                       case 3: currentI->setPredicate(ICmpInst::ICMP_UGE);
                               break; // unsigned greater or equal
                       case 4: currentI->setPredicate(ICmpInst::ICMP_ULT);
                               break; // unsigned less than
                       case 5: currentI->setPredicate(ICmpInst::ICMP_ULE);
                               break; // unsigned less or equal
                       case 6: currentI->setPredicate(ICmpInst::ICMP_SGT);
                               break; // signed greater than
                       case 7: currentI->setPredicate(ICmpInst::ICMP_SGE);
                               break; // signed greater or equal
                       case 8: currentI->setPredicate(ICmpInst::ICMP_SLT);
                               break; // signed less than
                       case 9: currentI->setPredicate(ICmpInst::ICMP_SLE);
                               break; // signed less or equal
                     }
                     break;
           }
 
         }
         if(opcode == Instruction::FCmp){ // Conditions (with float)
           FCmpInst *currentI = (FCmpInst*)(&i);
           switch(llvm::cryptoutils->get_range(3)){ // must be improved
             case 0: //do nothing
               break;
             case 1: currentI->swapOperands();
                     break;
             case 2: // randomly change the predicate
                     switch(llvm::cryptoutils->get_range(10)){
                       case 0: currentI->setPredicate(FCmpInst::FCMP_OEQ);
                               break; // ordered and equal
                       case 1: currentI->setPredicate(FCmpInst::FCMP_ONE);
                               break; // ordered and operands are unequal
                       case 2: currentI->setPredicate(FCmpInst::FCMP_UGT);
                               break; // unordered or greater than
                       case 3: currentI->setPredicate(FCmpInst::FCMP_UGE);
                               break; // unordered, or greater than, or equal
                       case 4: currentI->setPredicate(FCmpInst::FCMP_ULT);
                               break; // unordered or less than
                       case 5: currentI->setPredicate(FCmpInst::FCMP_ULE);
                               break; // unordered, or less than, or equal
                       case 6: currentI->setPredicate(FCmpInst::FCMP_OGT);
                               break; // ordered and greater than
                       case 7: currentI->setPredicate(FCmpInst::FCMP_OGE);
                               break; // ordered and greater than or equal
                       case 8: currentI->setPredicate(FCmpInst::FCMP_OLT);
                               break; // ordered and less than
                       case 9: currentI->setPredicate(FCmpInst::FCMP_OLE);
                               break; // ordered or less than, or equal
                     }
                     break;
           }
         }
       }
     }

```

原基本块：
-----

```
originalBB：
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
  %call = call i32 @atoi(i8* %1) #2
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %2, 0
  br i1 %cmp, label %if.then, label %if.else

```

copy 的基本块：
----------

```
originalBBalteredBB:                              ; preds = %originalBB, %entry
  %retvalalteredBB = alloca i32, align 4
  %argc.addralteredBB = alloca i32, align 4
  %argv.addralteredBB = alloca i8**, align 8
  %aalteredBB = alloca i32, align 4
  store i32 0, i32* %retvalalteredBB, align 4
  store i32 %argc, i32* %argc.addralteredBB, align 4
  store i8** %argv, i8*** %argv.addralteredBB, align 8
  %70 = load i8**, i8*** %argv.addralteredBB, align 8
  %arrayidxalteredBB = getelementptr inbounds i8*, i8** %70, i64 1
  %71 = load i8*, i8** %arrayidxalteredBB, align 8
  %callalteredBB = call i32 @atoi(i8* %71) #3
  store i32 %callalteredBB, i32* %aalteredBB, align 4
  %72 = load i32, i32* %aalteredBB, align 4
  %cmpalteredBB = icmp eq i32 %72, 0
  br i1 %cmpalteredBB,label %if.then,label %if.else

```

copy 后变量名字进行改动

addBogusFlow 函数 2
=================

```
alteredBB->getTerminator()->eraseFromParent();
basicBlock->getTerminator()->eraseFromParent();

```

这里的话是指把 entry 里面的跳转和拷贝出来的 block 块最后一个跳转也删掉：

```
entry：
 
originalBBalteredBB:                              ; preds = %originalBB, %entry
  %retvalalteredBB = alloca i32, align 4
  %argc.addralteredBB = alloca i32, align 4
  %argv.addralteredBB = alloca i8**, align 8
  %aalteredBB = alloca i32, align 4
  store i32 0, i32* %retvalalteredBB, align 4
  store i32 %argc, i32* %argc.addralteredBB, align 4
  store i8** %argv, i8*** %argv.addralteredBB, align 8
  %70 = load i8**, i8*** %argv.addralteredBB, align 8
  %arrayidxalteredBB = getelementptr inbounds i8*, i8** %70, i64 1
  %71 = load i8*, i8** %arrayidxalteredBB, align 8
  %callalteredBB = call i32 @atoi(i8* %71) #3
  store i32 %callalteredBB, i32* %aalteredBB, align 4
  %72 = load i32, i32* %aalteredBB, align 4
  %cmpalteredBB = icmp eq i32 %72, 0

```

```
  Value * LHS = ConstantFP::get(Type::getFloatTy(F.getContext()), 1.0);
Value * RHS = ConstantFP::get(Type::getFloatTy(F.getContext()), 1.0);
Twine * var4 = new Twine("condition");
FCmpInst * condition = new FCmpInst(*basicBlock, FCmpInst::FCMP_TRUE , LHS, RHS, *var4);

```

在 entry 里面生成两个浮点数，并进行两个浮点数的比较跳转指令：

```
entry:
    %condition=fcmp true float 1.00000e+00,1.00000e+00
    br i1 %7, label %originalBB, label %originalBBalteredBB

```

fcmp 后面条件为 true，它只会一直跳转为前者 originalBB。

```
BranchInst::Create(originalBB, alteredBB, (Value *)condition, basicBlock);

```

在 originalBBalteredBB 生成一个跳转指令，跳转到 originalBB:

```
originalBBalteredBB:                              ; preds = %originalBB, %entry
  %retvalalteredBB = alloca i32, align 4
  %argc.addralteredBB = alloca i32, align 4
  %argv.addralteredBB = alloca i8**, align 8
  %aalteredBB = alloca i32, align 4
  store i32 0, i32* %retvalalteredBB, align 4
  store i32 %argc, i32* %argc.addralteredBB, align 4
  store i8** %argv, i8*** %argv.addralteredBB, align 8
  %70 = load i8**, i8*** %argv.addralteredBB, align 8
  %arrayidxalteredBB = getelementptr inbounds i8*, i8** %70, i64 1
  %71 = load i8*, i8** %arrayidxalteredBB, align 8
  %callalteredBB = call i32 @atoi(i8* %71) #3
  store i32 %callalteredBB, i32* %aalteredBB, align 4
  %72 = load i32, i32* %aalteredBB, align 4
  %cmpalteredBB = icmp eq i32 %72, 0
   br label %originalBB

```

```
BasicBlock::iterator i = originalBB->end();
// Split at this point (we only want the terminator in the second part)
Twine * var5 = new Twine("originalBBpart2");
BasicBlock * originalBBpart2 = originalBB->splitBasicBlock(--i , *var5);

```

查找到 originBB 最后一条指令进行 split，然后创建一个 originalBBpart2 基本块

```
originalBBpart2：
  br i1 %cmp, label %if.then, label %if.else

```

切割后 originBB 最后就变成了无条件的跳转：

```
originalBB：
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
  %call = call i32 @atoi(i8* %1) #2
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %2, 0
  br label %originalBBpart2

```

紧接着把 originBB 最后一行给删掉，创建一个 fcmp 的条件跳转：

```
originalBB->getTerminator()->eraseFromParent();
Twine * var6 = new Twine("condition2");
 FCmpInst * condition2 = new FCmpInst(*originalBB, CmpInst::FCMP_TRUE , LHS, RHS, *var6);
 BranchInst::Create(originalBBpart2, alteredBB, (Value *)condition2, originalBB);

```

如下：

```
originalBB：
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
  %call = call i32 @atoi(i8* %1) #2
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %2, 0
  %condition2=fcmp true float 1.00000e+00,1.00000e+00
  br i1 %condition2, label %originalBBpart2, label %originalBBalteredBB

```

这个函数主要就是创建了 entry 和 originalBB 代码块的最后两行浮点数比较的跳转

尾声
==

网上很多很多现成资料，我这个也是自己看了再总结的，如果哪里不对的话，还请多多指教。

  

[看雪 ·2023 KCTF 年度赛即将来袭！ [防守方] 规则发布，征题截止 08 月 25 日](https://bbs.kanxue.com/thread-276642.htm)

最后于 8 小时前 被寻梦之璐编辑 ，原因： [#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#源码框架](forum-161-1-127.htm)