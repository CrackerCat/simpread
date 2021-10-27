> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/212768)

> 本文讲讲我个人重写虚假控制流的思路，以及对其他项目中相同功能的理解。

[![](https://p0.ssl.qhimg.com/t01ddd667736897a5aa.png)](https://p0.ssl.qhimg.com/t01ddd667736897a5aa.png)

由于我个人觉得 OLLVM 的代码年久失修，所以就新建了个项目，开始学习并重写 OLLVM 。相对于 OLLVM 中的代码，我个人认为我的实现更加的精简易读一些。

下面讲讲我个人重写虚假控制流的思路，以及对其他项目中相同功能的理解。

什么是虚假控制流 (Bogus control flow)
-----------------------------

在讲解虚假控制流时，我们先解释一些名词，然后看下流程表示就可以知道大概了。

### 名词解释

#### 不透明谓词 (Opaque predicate)

这里引用知乎上大佬的回答：[利用不透明谓词混淆代码的原理是什么？ – 张建涛的回答 – 知乎](https://www.zhihu.com/question/46259412/answer/199689652)

> 不透明谓词是指一个表达式，他的值在执行到某处时，对程序员而言必然是已知的，但是由于某种原因，编译器或者说静态分析器无法推断出这个值，只能在运行时确定。

#### 中间表示 (IR)

在将代码转换成汇编和机器码之前，编译器中的一种代码表示形式，可以简单的理解为具有跨平台性和其他特性的汇编，正确解释请看编译原理。

### 虚假控制流的流程表示

用 OLLVM 项目中的注释部分就可以清晰表示出什么是虚假控制流。可以看到，相对于原程序中的单个代码块，混淆后的程序中加入了一个由不透明谓词组成的条件语句，由于混淆器知道他的值，所以可以保证被混淆程序的正确性，执行被混淆程序原本的代码，而编译器和反编译器都无法对这个表达式进行求值，只能保留此谓词，达到干扰静态分析的目的。

```
 Before :
                        entry
                          |
                    ______v______
                   |   Original  |
                   |_____________|
                          |
                          v
                       return

 After :
                        entry
                          |
                      ____v_____
                     |condition*| (false)
                     |__________|----+
                    (true)|          |
                          |          |
                    ______v______    |
               +-->|   Original* |   |
               |   |_____________| (true)
               |   (false)|    !-----------> return
               |    ______v______    |
               |   |   Altered   |<--!
               |   |_____________|
               |__________|


```

怎么实现
----

### 别的项目是怎么实现的

在 OLLVM 中，使用了 `y > 10 || x * (x + 1) % 2 == 0` 这个不透明谓词，学过数学的都知道，`x * (x + 1) % 2 == 0` 是个永真式。也就是说，混淆器已经知道了 `y > 10 || x * (x + 1) % 2 == 0` 这个式子的值，所以可以将其放在条件语句中，控制代码的走向；而编译器和反编译器都认为这个表达式需要进行运行后才能求值，会保留这段代码，进而干扰到反编译器。

在 [HikariObfuscator](https://github.com/HikariObfuscator/Core) 这个项目中使用 IRBuilder 生成一箩筐的整数运算，由于 LLVM 的 IRBuilder 会折叠常量，混淆器就能知道了之前生成的一箩筐的整数运算的最终结果，而编译器和反编译器都不知道生成的表达式的值，所以就获得了一个崭新的不透明谓词。

当然，上面的两个混淆都有较为简单的破解方法，就是将不透明谓词中的变量修改成常量，并且设置一个初始值，这时候 Hex-rays 等反编译器会对不透明谓词进行求值，如果不透明谓词的结果可知，那么这个表达式将会被优化。

### 具体实现

这里用的是旧的 LLVM Pass 格式，所以 Pass 的主要逻辑都在 `runOnFunction` 这个函数中。程序先收集函数中所有的 BasicBlock 和 alloca 指令，然后再随机挑选 BasicBlock 加上虚假控制流。这个 Pass 通过被选中的 BasicBlock 生成虚假的 BasicBlock，然后再使用不透明谓词将这两个 BasicBlock 粘在一起。

```
  virtual bool runOnFunction(Function &F) {
    
    SmallVector<BasicBlock *, 0> targetBasicBlocks;
    for (BasicBlock &BB : F) {
      targetBasicBlocks.emplace_back(&BB);
    }
    
    findAllocInst(F.getEntryBlock());
    
    for (BasicBlock *BB : targetBasicBlocks) {
      if (rng() % 100 >= ObfProbRate) {
        continue;
      }
      BasicBlock *bogusBB = geneBogusFlow(BB, &F);
      addBogusFlow(BB, bogusBB, &F);
    }
    return true;
  }


```

在虚假的 BasicBlock 生成中，先使用 `CloneBasicBlock` 复制 BasicBlock。

```
  
  
  
  BasicBlock *geneBogusFlow(BasicBlock *targetBB, Function *F) {
    ValueToValueMapTy VMap;
    const Twine &Name = "bogusBlock";
    BasicBlock *bogusBB = CloneBasicBlock(targetBB, VMap, Name, F);


```

然后再修复虚假的 BasicBlock，例如修复 phi 节点和编译时生成的 metadata。

```
    
    BasicBlock::iterator ji = targetBB->begin();
    for (Instruction &i : *bogusBB) {
      
      for (Use &opi : i.operands()) {
        
        Value *v = MapValue(opi, VMap, RF_None);
        if (v != nullptr) {
          opi = v;
        }
      }
      
      if (PHINode *pn = dyn_cast<PHINode>(&i)) {
        for (unsigned int j = 0, e = pn->getNumIncomingValues(); j != e; ++j) {
          Value *v = MapValue(pn->getIncomingBlock(j), VMap, RF_None);
          if (v != nullptr) {
            pn->setIncomingBlock(j, cast<BasicBlock>(v));
          }
        }
      }
      
      SmallVector<std::pair<unsigned, MDNode *>, 4> MDs;
      i.getAllMetadata(MDs);
      
      i.setDebugLoc(ji->getDebugLoc());
      ji++;
    }


```

最后是将虚假的 BasicBlock 中的操作数进行随机改变，以干扰静态分析。

```
    
    for (Instruction &i : *bogusBB) {
      if (!i.isBinaryOp()) {
        continue;
      }
      unsigned int opcode = i.getOpcode();
      unsigned int numOperands = i.getNumOperands();
      if (find(integerOp, opcode) != integerOp.end()) {
        i.setOperand(0, i.getOperand(rng() % numOperands));
      } else if (find(floatOp, opcode) != floatOp.end()) {
        i.setOperand(0, i.getOperand(rng() % numOperands));
      }
    }
    return bogusBB;
  }


```

接下来是将原本的 BasicBlock 和 虚假的 BasicBlock 用不透明谓词粘在一起。首先挑出本 Basicblock 中第一个不是 Phi、Dbg、Lifetime 的指令的地址，然后调用 splitBasicBlock 函数将一个 Basicblock 按照前面获得的地址一分为二。接下来将两个 BasicBlock 与父节点分离。

```
  
  void addBogusFlow(BasicBlock *targetBB, BasicBlock *bogusBB, Function *F) {
    
    BasicBlock *targetBodyBB;
    if (targetBB->getFirstNonPHIOrDbgOrLifetime()) {
      targetBodyBB =
          targetBB->splitBasicBlock(targetBB->getFirstNonPHIOrDbgOrLifetime());
    } else {
      targetBodyBB = targetBB->splitBasicBlock(targetBB->begin());
    }

    
    bogusBB->getTerminator()->eraseFromParent();
    targetBB->getTerminator()->eraseFromParent();


```

最后通过 IRBuilder 创建不透明谓词 `(x * (x + 1) % 2 == 0)`，将两个 BasicBlock 按照一定的逻辑相连（生成代码的逻辑则是在开头的[虚假控制流的流程表示](#%E8%99%9A%E5%81%87%E6%8E%A7%E5%88%B6%E6%B5%81%E7%9A%84%E6%B5%81%E7%A8%8B%E8%A1%A8%E7%A4%BA)）。这个不透明谓词的变量 `x` 选择的是之前在 `runOnFunction` 收集的 alloca 指令，即在高级语言中的各种变量，这样能更强的干扰反编译器（虽然还是可能被符号执行优化掉）。

```
    
    
    IRBuilder<> bogusCondBuilder(targetBB);
    Value *loadInst;
    if (allocaInsts.size() != 0) {
      loadInst =
          bogusCondBuilder.CreateLoad(allocaInsts[rng() % allocaInsts.size()]);
    } else {
      loadInst = bogusCondBuilder.getInt32(1);
    }
    Value *addInst =
        bogusCondBuilder.CreateAdd(loadInst, bogusCondBuilder.getInt32(1));
    Value *mulInst = bogusCondBuilder.CreateMul(loadInst, addInst);
    Value *remainInst =
        bogusCondBuilder.CreateSRem(mulInst, bogusCondBuilder.getInt32(2));
    Value *trueCond =
        bogusCondBuilder.CreateICmpEQ(remainInst, bogusCondBuilder.getInt32(0));
    bogusCondBuilder.CreateCondBr(trueCond, targetBodyBB, bogusBB);
    BranchInst::Create(targetBodyBB, bogusBB);

    
    BasicBlock *targetBodyEndBB =
        targetBodyBB->splitBasicBlock(--targetBodyBB->end());
    
    targetBodyBB->getTerminator()->eraseFromParent();
    
    IRBuilder<> endCondBuilder(targetBodyBB, targetBodyBB->end());
    Value *endCond =
        endCondBuilder.CreateICmpEQ(remainInst, bogusCondBuilder.getInt32(0));
    endCondBuilder.CreateCondBr(endCond, targetBodyEndBB, bogusBB);
  }


```

基本的代码就是上面所讲的了。

效果演示
----

原程序反编译结果，和原代码基本一样：

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int j; 
  int i; 
  unsigned __int64 n; 
  char s[32]; 
  const char **v8; 
  int v9; 
  int v10; 

  v10 = 0;
  v9 = argc;
  v8 = argv;
  __isoc99_scanf("%30s", s, envp);
  n = strlen(s);
  for ( i = 0; i < n; ++i )
  {
    s[i] ^= 0x2Au;
    s[i] += 6;
  }
  for ( j = 1; j < n; ++j )
    s[j - 1] -= s[j];
  if ( !strncmp(s, &encrypt_data, n) )
    printf("Correct!");
  return 0;
}


```

程序混淆后反编译结果，可以看到混淆器成功的添加了不透明谓词，且程序可以正常运行：

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; 
  int v5; 
  int v6; 
  int v7; 
  unsigned __int64 n; 
  char s[32]; 
  const char **v10; 
  int v11; 
  int v12; 

  v12 = 0;
  v11 = argc;
  v10 = argv;
  __isoc99_scanf("%30s", s, envp);
  n = strlen(s);
  v7 = 0;
  while ( 1 )
  {
    while ( (v5 + 1) * v5 % 2 )
      ;
    if ( v7 >= n )
      break;
    v4 = (v11 + 1) * v11 % 2;
    if ( v4 )
      goto LABEL_20;
    while ( 1 )
    {
      s[v7] ^= 0x2Au;
      s[v7] += 6;
      if ( !v4 )
        break;
LABEL_20:
      s[v7] = 0;
      s[v7] += 6;
    }
    if ( (v11 + 1) * v11 % 2 )
      goto LABEL_21;
    while ( 1 )
    {
      ++v7;
      if ( !((v11 + 1) * v11 % 2) )
        break;
LABEL_21:
      v7 = 2;
    }
  }
  do
    v6 = 1;
  while ( (v7 + 1) * v7 % 2 );
  while ( 2 )
  {
    if ( v6 < n )
    {
      s[v6 - 1] -= s[v6];
      if ( !((v11 + 1) * v11 % 2) )
      {
LABEL_12:
        ++v6;
        if ( !((v11 + 1) * v11 % 2) )
          continue;
      }
      v6 = 2;
      goto LABEL_12;
    }
    break;
  }
  if ( !strncmp(s, &encrypt_data, n) )
  {
    if ( !((v6 + 1) * v6 % 2) )
      goto LABEL_16;
    do
    {
      printf("Correct!");
LABEL_16:
      printf("Correct!");
    }
    while ( (v6 + 1) * v6 % 2 );
  }
  while ( (v6 + 1) * v6 % 2 )
    ;
  return 0;
}


```

总结
--

相比与 OLLVM，重写的代码中省略了大部分虚假 BasicBlock 中的随机生成代码的部分，并改写了虚假谓词生成的逻辑，在 OLLVM 中，先是先生成了 (1.0 == 1.0) 来进行占位，然后再在后面替换前面的永真式，我个人觉得没必要，直接生成不透明谓词不就好了吗… 所以代码看起来比较精简（在本文撰写时，带注释的代码行数为 180 行左右）。

refs
----

[个人项目地址](https://github.com/chenx6/baby_obfuscator)

[Obfuscator-llvm 源码分析](https://sq.163yun.com/blog/article/175307579596922880)

[HikariObfuscator/Core](https://github.com/HikariObfuscator/Core)

[obfuscator-llvm/obfuscator](https://github.com/obfuscator-llvm/obfuscator)