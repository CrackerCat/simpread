> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266201.htm)

虚假控制流的原理和去除方法我在另一篇文章已经讲过了：[利用 angr 符号执行去除虚假控制流](https://bbs.pediy.com/thread-266005.htm)，这篇文章简单记录下我阅读 OLLVM 虚假控制流源码的过程。

 

OLLVM 虚假控制流源码地址：[https://github.com/obfuscator-llvm/obfuscator/blob/llvm-4.0/lib/Transforms/Obfuscation/BogusControlFlow.cpp](https://github.com/obfuscator-llvm/obfuscator/blob/llvm-4.0/lib/Transforms/Obfuscation/BogusControlFlow.cpp)

0x00. runOnFunction 函数
======================

首先从 **runOnFunction** 函数开始：

```
virtual bool runOnFunction(Function &F){
      // Check if the percentage is correct
      if (ObfTimes <= 0) {
        errs()<<"BogusControlFlow application number -bcf_loop=x must be x > 0";
        return false;
      }
 
      // Check if the number of applications is correct
      if ( !((ObfProbRate > 0) && (ObfProbRate <= 100)) ) {
        errs()<<"BogusControlFlow application basic blocks percentage -bcf_prob=x must be 0 < x <= 100";
        return false;
      }

```

**ObfTimes** 是对函数进行混淆的次数，**ObfProbRate** 是对基本块进行混淆的概率，这两个分别是执行 **opt** 时传入的参数：

```
static cl::opt ObfProbRate("bcf_prob", cl::desc("Choose the probability [%] each basic blocks will be obfuscated by the -bcf pass"), cl::value_desc("probability rate"), cl::init(defaultObfRate), cl::Optional);
 
static cl::opt ObfTimes("bcf_loop", cl::desc("Choose how many time the -bcf pass loop on a function"), cl::value_desc("number of times"), cl::init(defaultObfTime), cl::Optional); 
```

**runOnFunction** 函数剩下的内容：

```
  // If fla annotations
  if(toObfuscate(flag,&F,"bcf")) {
    bogus(F);
    doF(*F.getParent());
    return true;
  }
 
  return false;
} // end of runOnFunction()

```

**toObfuscate** 函数的作用是检查这个函数之前有没有进行过虚假控制流混淆，如果有则返回 false 避免重复混淆，还检查了一些其他东西，感觉不是很重要。要重点分析的是 **bogus** 和 **doF** 函数。

0x01. bogus 函数
==============

OLLVM 的作者喜欢在函数开头就声明所有全局变量，先不管它。

```
void bogus(Function &F) {
  // For statistics and debug
  ++NumFunction;
  int NumBasicBlocks = 0;
  bool firstTime = true; // First time we do the loop in this function
  bool hasBeenModified = false;
  DEBUG_WITH_TYPE("opt", errs() << "bcf: Started on function " << F.getName() << "\n");
  DEBUG_WITH_TYPE("opt", errs() << "bcf: Probability rate: "<< ObfProbRate<< "\n");
  if(ObfProbRate < 0 || ObfProbRate > 100){
    DEBUG_WITH_TYPE("opt", errs() << "bcf: Incorrect value,"
        << " probability rate set to default value: "
        << defaultObfRate <<" \n");
    ObfProbRate = defaultObfRate;
  }
  DEBUG_WITH_TYPE("opt", errs() << "bcf: How many times: "<< ObfTimes<< "\n");
  if(ObfTimes <= 0){
    DEBUG_WITH_TYPE("opt", errs() << "bcf: Incorrect value,"
        << " must be greater than 1. Set to default: "
        << defaultObfTime <<" \n");
    ObfTimes = defaultObfTime;
  }
  NumTimesOnFunctions = ObfTimes;

```

进入 **bogus** 函数后首先检查了一遍 **obfProbRate** 和 **obfTimes**（话说之前在 **runOnFunction** 里不就检查过了吗 = =）。

```
NumTimesOnFunctions = ObfTimes;

```

这里的 NumTimesOnFunctions 是调试用的。虚假控制流的代码比控制流平坦化的代码多了几百行，重要的原因就是添加了很多调试语句。为了突出重点，之后的调试语句直接忽略:

```
STATISTIC(NumTimesOnFunctions, "b. Number of times we run on each function");

```

接着是一个超大的循环，直到函数结尾，我们去掉所有调试语句分析，循环总共有两层，第一层的循环开头把所有基本块存入一个 list 中：

```
do{
  // Put all the function's block in a list
  std::list basicBlocks;
  for (Function::iterator i=F.begin();i!=F.end();++i) {
    basicBlocks.push_back(&*i);
  } 
```

接着进入第二层的循环，每个基本块有 **ObfProbRate** 的概率被混淆，即对基本块调用了 **addBogusFlow** 函数：

```
while(!basicBlocks.empty()){
  NumBasicBlocks ++;
  // Basic Blocks' selection
  if((int)llvm::cryptoutils->get_range(100) <= ObfProbRate){
    hasBeenModified = true;
    BasicBlock *basicBlock = basicBlocks.front();
    addBogusFlow(basicBlock, F);
  }
  // remove the block from the list
  basicBlocks.pop_front();
} // end of while(!basicBlocks.empty())

```

后面就都是调试内容了，所以 **bogus** 函数的作用就是对指定函数的每个基本块以 **ObfProbRate** 的概率进行混淆，接下来分析 **addBogusFlow** 函数。

0x02. addBogusFlow 函数第一部分
=========================

首先通过 **getFirstNonPHIOrDbgOrLifetime** 函数和 **splitBasicBlock** 函数把 BasicBlock 分成了两部分，第一部分只包含 **PHI Node** 和一些调试信息，其中 **Twine** 可以看做字符串，即新产生的 BasicBlock 的名称：

```
/* addBogusFlow
 *
 * Add bogus flow to a given basic block, according to the header's description
 */
virtual void addBogusFlow(BasicBlock * basicBlock, Function &F){
 
 
  // Split the block: first part with only the phi nodes and debug info and terminator
  //                  created by splitBasicBlock. (-> No instruction)
  //                  Second part with every instructions from the original block
  // We do this way, so we don't have to adjust all the phi nodes, metadatas and so on
  // for the first block. We have to let the phi nodes in the first part, because they
  // actually are updated in the second part according to them.
  BasicBlock::iterator i1 = basicBlock->begin();
  if(basicBlock->getFirstNonPHIOrDbgOrLifetime())
    i1 = (BasicBlock::iterator)basicBlock->getFirstNonPHIOrDbgOrLifetime();
  Twine *var;
  var = new Twine("originalBB");
  BasicBlock *originalBB = basicBlock->splitBasicBlock(i1, *var);

```

随后调用 **createAlteredBasicBlock** 函数产生了一个新的基本块：

```
// Creating the altered basic block on which the first basicBlock will jump
      Twine * var3 = new Twine("alteredBB");
      BasicBlock *alteredBB = createAlteredBasicBlock(originalBB, *var3, &F);

```

0x03. createAlteredBasicBlock 函数
================================

**createAlteredBasicBlock** 函数首先调用了 **CloneBasicBlock** 函数对传入的基本块进行克隆：

```
virtual BasicBlock* createAlteredBasicBlock(BasicBlock * basicBlock,
    const Twine &  Name = "gen", Function * F = 0){
  // Useful to remap the informations concerning instructions.
  ValueToValueMapTy VMap;
  BasicBlock * alteredBB = llvm::CloneBasicBlock (basicBlock, VMap, Name, F);

```

但是 **CloneBasicBlock** 函数进行的克隆并不是完全的克隆，第一他不会对指令的操作数进行替换，比如：

```
orig:
  %a = ...
  %b = fadd %a, ...
 
clone:
  %a.clone = ...
  %b.clone = fadd %a, ... ; Note that this references the old %a and
not %a.clone!

```

在 clone 出来的基本块中，**fadd** 指令的操作数不是 **%a.clone**，而是 **%a**。

 

所以之后要通过 **VMap** 对所有操作数进行映射，使其恢复正常：

```
// Remap operands.
BasicBlock::iterator ji = basicBlock->begin();
for (BasicBlock::iterator i = alteredBB->begin(), e = alteredBB->end() ; i != e; ++i){
  // Loop over the operands of the instruction
  for(User::op_iterator opi = i->op_begin (), ope = i->op_end(); opi != ope; ++opi){
    // get the value for the operand
    Value *v = MapValue(*opi, VMap,  RF_None, 0);
    if (v != 0){
      *opi = v;
      DEBUG_WITH_TYPE("gen", errs() << "bcf: Value's operand has been setted\n");
    }
  }

```

第二，它不会对 PHI Node 进行任何处理，PHI Node 的前驱块仍然是原始基本块的前驱块，但是新克隆出来的基本块并没有任何前驱块，所以我们要对 PHI Node 的前驱块进行 remap：

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

解释一下什么是 PHI Node，所有 LLVM 指令都使用 **SSA** (Static Single Assignment，静态一次性赋值) 方式表示，即所有变量都只能被赋值一次，这样做主要是便于后期的代码优化。如下图，**%temp** 的值被赋值成 1 后就永远是 1 了：  
![](https://bbs.pediy.com/upload/attach/202102/910514_6RWUREBTATQRJMC.png)  
PHI Node 是一条可以一定程度上绕过 SSA 机制的指令，它可以根据不同的前驱基本块来赋值（有点像三元运算符）。如下图，如果 PHI Node 的前驱基本块是 entry，则将 current_i 赋值为 2，如果是 for_body，则赋值为 %i_plus_one：  
![](https://bbs.pediy.com/upload/attach/202102/910514_HS9JP8AC34VYME5.png)  
在这里我产生了一个疑问：既然 Clone 出来的基本块不存在任何前驱，那 PHI Node 的前继基本块到底被 remap 到了哪呢？  
我翻了一下 LLVM 的源码，发现 VMap 只会保存新旧指令的映射关系：  
![](https://bbs.pediy.com/upload/attach/202102/910514_SQCA3GYRCMP96RS.png)  
然后根据 MapValue 的注释来看应该是映射成了 nullptr：  
![](https://bbs.pediy.com/upload/attach/202102/910514_GFGUTKXX9DVUHBY.png)  
暂且不知道 PHI Node 中的前驱块设为 nullptr 会有什么影响，先继续往下看吧。  
之后是恢复 Metadata 和 Debug 信息，个人感觉是可有可无的，不管它：

```
  // Remap attached metadata.
  SmallVector, 4> MDs;
  i->getAllMetadata(MDs);
  // important for compiling with DWARF, using option -g.
  i->setDebugLoc(ji->getDebugLoc());
  ji++;
 
} // The instructions' informations are now all correct 
```

至此我们就完成了对一个基本块的克隆，之后是往基本块里添加垃圾指令，这里大概的思路就是往基本块里插入一些没用的赋值指令，或者修改 cmp 指令的条件，**BinaryOp** 大概指的是 add、mul、cmp 这类运算指令：

```
  // add random instruction in the middle of the bloc. This part can be improve
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
  return alteredBB;
} // end of createAlteredBasicBlock()

```

总的来说 createAlteredBasicBlock 函数克隆了一个基本块，并且往这个基本块里添加了一些垃圾指令。

0x04. addBogusFlow 函数第二部分
=========================

现在我们有了一个原始基本块 **originalBB**，和一个克隆出的基本块 **alteredBB**，以及一个只包含 PHI Node 和 Debug 信息的基本块 **basicBlock**（就是一开始分离出来的那个），现在的状况是这样：  
![](https://bbs.pediy.com/upload/attach/202102/910514_TTZV7M496J7E9KB.png)  
删除了 basciBlock 和 alteredBB 末尾的跳转，添加了新的分支指令。 **FCmpInst * condition = new FCmpInst(*basicBlock, FCmpInst::FCMP_TRUE , LHS, RHS, *var4)** 是一个永真的条件  
：

```
// Now that all the blocks are created,
// we modify the terminators to adjust the control flow.
 
alteredBB->getTerminator()->eraseFromParent();
basicBlock->getTerminator()->eraseFromParent();
 
// Preparing a condition..
// For now, the condition is an always true comparaison between 2 float
// This will be complicated after the pass (in doFinalization())
Value * LHS = ConstantFP::get(Type::getFloatTy(F.getContext()), 1.0);
Value * RHS = ConstantFP::get(Type::getFloatTy(F.getContext()), 1.0);
 
// The always true condition. End of the first block
Twine * var4 = new Twine("condition");
FCmpInst * condition = new FCmpInst(*basicBlock, FCmpInst::FCMP_TRUE , LHS, RHS, *var4);
 
// Jump to the original basic block if the condition is true or
// to the altered block if false.
BranchInst::Create(originalBB, alteredBB, (Value *)condition, basicBlock);
 
// The altered block loop back on the original one.
BranchInst::Create(originalBB, alteredBB);

```

现在的状况是这样：  
![](https://bbs.pediy.com/upload/attach/202102/910514_PBHHJMVXRYZZV5P.png)  
然后又是一波类似的操作，这里截取的 originalBB 结尾的跳转指令作为一个新的基本块 **originalBBPart2**：

```
  // The end of the originalBB is modified to give the impression that sometimes
  // it continues in the loop, and sometimes it return the desired value
  // (of course it's always true, so it always use the original terminator..
  //  but this will be obfuscated too;) )
 
  // iterate on instruction just before the terminator of the originalBB
  BasicBlock::iterator i = originalBB->end();
 
  // Split at this point (we only want the terminator in the second part)
  Twine * var5 = new Twine("originalBBpart2");
  BasicBlock * originalBBpart2 = originalBB->splitBasicBlock(--i , *var5);
  // the first part go either on the return statement or on the begining
  // of the altered block.. So we erase the terminator created when splitting.
  originalBB->getTerminator()->eraseFromParent();
  // We add at the end a new always true condition
  Twine * var6 = new Twine("condition2");
  FCmpInst * condition2 = new FCmpInst(*originalBB, CmpInst::FCMP_TRUE , LHS, RHS, *var6);
  BranchInst::Create(originalBBpart2, alteredBB, (Value *)condition2, originalBB);
} // end of addBogusFlow()

```

现在的状况是这样，可以看到 alteredBB 实际上是一个不可达的基本块，俗称垃圾块：  
![](https://bbs.pediy.com/upload/attach/202102/910514_8G35Q95KWBCHEJ2.png)  
至此虚假控制流混淆已见雏形。

0x05. doF 函数
============

首先回顾一下 **runOnFunction** 函数最后的代码，**bogus** 函数的代码我已经了解得差不多了，接下来是 **doF** 函数：

```
  // If fla annotations
  if(toObfuscate(flag,&F,"bcf")) {
    bogus(F);
    doF(*F.getParent());
    return true;
  }
 
  return false;
} // end of runOnFunction()

```

从开头的注释我们大概能了解到这个函数的作用，修改所有永真式（比如：**if(true)** 改为 **if((y < 10 || x * (x + 1) % 2 == 0))**），使其变得更加复杂，并且移除所有基本块和指令的名称：

```
/* doFinalization
 *
 * Overwrite FunctionPass method to apply the transformations to the whole module.
 * This part obfuscate all the always true predicates of the module.
 * More precisely, the condition which predicate is FCMP_TRUE.
 * It also remove all the functions' basic blocks' and instructions' names.
 */
bool doF(Module &M){
  // In this part we extract all always-true predicate and replace them with opaque predicate:
  // For this, we declare two global values: x and y, and replace the FCMP_TRUE predicate with
  // (y < 10 || x * (x + 1) % 2 == 0)
  // A better way to obfuscate the predicates would be welcome.
  // In the meantime we will erase the name of the basic blocks, the instructions
  // and the functions.

```

初始化两个全局变量 **x** 和 **y**：

```
//  The global values
Twine * varX = new Twine("x");
Twine * varY = new Twine("y");
Value * x1 =ConstantInt::get(Type::getInt32Ty(M.getContext()), 0, false);
Value * y1 =ConstantInt::get(Type::getInt32Ty(M.getContext()), 0, false);
 
GlobalVariable     * x = new GlobalVariable(M, Type::getInt32Ty(M.getContext()), false,
    GlobalValue::CommonLinkage, (Constant * )x1,
    *varX);
GlobalVariable     * y = new GlobalVariable(M, Type::getInt32Ty(M.getContext()), false,
    GlobalValue::CommonLinkage, (Constant * )y1,
    *varY);

```

遍历模块中所有函数的所有指令，遇到条件为永真式的条件跳转时将比较指令 **FCmpInst** 存储到 **toDelete** 向量中，将跳转指令 **BranchInst** 存储到 **toEdit** 向量中。删除所有基本块和指令名称的代码被注释掉了，不知道为啥：

```
std::vector toEdit, toDelete;
BinaryOperator *op,*op1 = NULL;
LoadInst * opX , * opY;
ICmpInst * condition, * condition2;
// Looking for the conditions and branches to transform
for(Module::iterator mi = M.begin(), me = M.end(); mi != me; ++mi){
  for(Function::iterator fi = mi->begin(), fe = mi->end(); fi != fe; ++fi){
    //fi->setName("");
    TerminatorInst * tbb= fi->getTerminator();
    if(tbb->getOpcode() == Instruction::Br){
      BranchInst * br = (BranchInst *)(tbb);
      if(br->isConditional()){
        FCmpInst * cond = (FCmpInst *)br->getCondition();
        unsigned opcode = cond->getOpcode();
        if(opcode == Instruction::FCmp){
          if (cond->getPredicate() == FCmpInst::FCMP_TRUE){
            toDelete.push_back(cond); // The condition
            toEdit.push_back(tbb);    // The branch using the condition
          }
        }
      }
    }
    /*
    for (BasicBlock::iterator bi = fi->begin(), be = fi->end() ; bi != be; ++bi){
      bi->setName(""); // setting the basic blocks' names
    }
    */
  }
}

```

添加新的条件跳转，删除旧的条件跳转，注释里的条件给错了，从代码来看应该是 **if y <10 || x*(x-1) % 2 == 0**，不过也差不多：

```
// Replacing all the branches we found
for(std::vector::iterator i =toEdit.begin();i!=toEdit.end();++i){
  //if y < 10 || x*(x+1) % 2 == 0
  opX = new LoadInst ((Value *)x, "", (*i));
  opY = new LoadInst ((Value *)y, "", (*i));
 
  op = BinaryOperator::Create(Instruction::Sub, (Value *)opX,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 1,
        false), "", (*i));
  op1 = BinaryOperator::Create(Instruction::Mul, (Value *)opX, op, "", (*i));
  op = BinaryOperator::Create(Instruction::URem, op1,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 2,
        false), "", (*i));
  condition = new ICmpInst((*i), ICmpInst::ICMP_EQ, op,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 0,
        false));
  condition2 = new ICmpInst((*i), ICmpInst::ICMP_SLT, opY,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 10,
        false));
  op1 = BinaryOperator::Create(Instruction::Or, (Value *)condition,
      (Value *)condition2, "", (*i));
 
  BranchInst::Create(((BranchInst*)*i)->getSuccessor(0),
      ((BranchInst*)*i)->getSuccessor(1),(Value *) op1,
      ((BranchInst*)*i)->getParent());
  (*i)->eraseFromParent(); // erase the branch
}

```

另外在 IDA 里这个条件有时候会变成 **if y >= 10 && x*(x-1) % 2 != 0**：  
![](https://bbs.pediy.com/upload/attach/202102/910514_6NWUJ63ZRP6Z98R.png)  
最后是删除之前存储的 **FCmpInst**：

```
// Erase all the associated conditions we found
for(std::vector::iterator i =toDelete.begin();i!=toDelete.end();++i){
  DEBUG_WITH_TYPE("gen", errs() << "bcf: Erase condition instruction:"
      << *((Instruction*)*i)<< "\n");
  (*i)->eraseFromParent();
}

```

到这里 OLLVM 虚假控制流的源码基本分析完毕了。

0x06. 总结
========

在研究虚假控制流的源码之前属实没想到它的代码会比控制流平坦化多出几百行。原因之一是在复制基本块时要处理 PHI Node 和一些 Debug 信息，以及添加垃圾指令（不知道这样做是不是为了防止 IDA 识别重复的基本块）；原因之二是代码中有很多 debug 语句，也占据了半壁江山；最后一点就是作者的代码风格问题了，感觉有点肿胀。

 

总的来说还是很有收获啦！不过在研究过程中还是有一些没弄懂的地方，比如 PHI Node 的 remap 到底是个什么 remap 法，简单的映射成 nullptr 不会有问题吗？然后就是 Lifetime Intrinsics 和 Debug Intrinsics 到底是个啥也没弄明白。希望有大佬能指点迷津。

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)