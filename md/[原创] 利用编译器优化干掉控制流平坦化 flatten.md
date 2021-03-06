> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266323.htm)

上一篇在这里 [https://bbs.pediy.com/thread-265335.htm 利用编译器优化干掉虚假控制流](https://bbs.pediy.com/thread-265335.htm利用编译器优化干掉虚假控制流)  

上一篇我们通过优化去除了类似 x * (x + 1) % 2 == 0 的虚假控制流，这里的混淆去除 bcf 之后如下图，很明显的平坦化混淆

图上已经比较清晰了，黄色圈住的块是进入平坦化，离开平坦化 return，选择控制流的 switch 混淆块，

蓝色的两块如果执行到的话就是真实块，其余基本都是 case 混淆块![](https://bbs.pediy.com/upload/attach/202103/877073_DYXADTRSQE3TZXT.jpg)  

具体算法过程如下，这里参考了大佬的思路，看到大佬又发了一篇 vmp 的，下面必须抽空继续学习

一种通过后端编译优化脱混淆壳的方法 [https://bbs.pediy.com/thread-260626.htm](https://bbs.pediy.com/thread-260626.htm)  

0. 把 ir 中所有 switch 指令转换为 cmp+br 指令

1. 遍历所有 block，如果某个 block 的 Predecessor >6, 可能是混淆节点，也就是 switch 块，标记为 ob

2. 获取 ob 的 Successor 后续块的 Terminator 指令，如果是条件跳转 cmp+br，getOperand(0) 就得到 switchvar

3. 根据 switchvar 这个 Value 类的实例，遍历所有使用这个值的 Users ，挑出其中指令为 cmp 且 switchvar 为 op0 的 Users，标记为 case 分发块

4. 紧随 case 分发块之后一直到 ob 混淆块或者 return 的为真实块。这里大概有这几种情况

4.1 赋值 -> 多个 case 分发块 ->1 个或多个真实块 ->ob 块 -> 赋值 -> 下一次循环 这种是包含了真实块的有用执行路径  

4.2 赋值 -> 多个 case 分发块 -> 赋值 ->ob 块 -> 下一次循环  这种是只改变 switchvar 的无用路径

4.3 赋值 -> 多个 case 分发块 ->ob 块 -> 下一次循环 这种在 ob 块的 phi 节点上没有赋值，是永远不可能执行的路径，在分析 ob 块的 phi 节点时可以直接删掉  

5. 根据所有分发块建立二叉树，根节点为 ob 节点，往下遍历 block 寻找结尾 cmp+br 指令，左节点为 true，右为 false，直到没有 br 指令或者 cmp 不为 switchvar，就到达真实块开头，它正对着的混淆 phi 节点前驱块为真实块结尾

6. 从混淆 phi 节点开始，根据 switchvar 的 Use 往上追踪所有赋值，遇到 phi 继续追踪直到定值，建立混淆 phi 节点前驱块与 switchvar 赋值一一对应，没有 switchvar 赋值的 phi 路径肯定不会执行

7. 模拟执行，第一次 switchvar 的值在进入 switch 时赋予，也就是 ob 块的后继块的 phi 左值，其余的在到达 ob 混淆节点后由 phi 指令赋予，最后一次必然到达 return 指令，理清楚整个路径的顺序

8. 把真实块通过 br 连接起来，优化，这个函数其实只有一个真实块

把 ir 中所有 switch 指令转换为 cmp+br 指令
===============================

opt  -lowerswitch

runOnFunction 函数
================

```
    bool runOnFunction(Function &F) override {
      Function *tmp=&F;
      BasicBlock *ob=NULL;
      bool flatOrNot=hasFlatten(*tmp,&ob);
      // errs() << "混淆block:" << *ob << '\n';
      if(flatOrNot){
        Instruction *var=getSwitchVar(*ob);
        //errs() << "getSwitchVar:" << *var << '\n';
        match(*tmp,*var);
      }
      return false;
    }

```

判断是否有平坦化并获得 switch 混淆块
======================

遍历所有 block，如果某个 block 的 Predecessor >6, 可能是混淆节点，也就是 switch 块，标记为 ob  

```
    bool hasFlatten(Function &F,BasicBlock **ob){
      Function *tmp=&F;
      bool flat=false;
      for (Function::iterator bb = tmp->begin(); bb != tmp->end(); ++bb) {
          BasicBlock *tmp=&*bb;        
          int numPre=0;
          pred_iterator PI = pred_begin(tmp), E = pred_end(tmp);
          for (;PI != E; ++PI) {
            numPre+=1;
          }
          if(numPre>=6){
            flat=true;
            static BasicBlock *obb=&*tmp;
            *ob=&*tmp; 
            // errs() << "混淆blockterm:" << *obb<< '\n';
          }
      }
      return flat;
    }

```

获得 switchvar
============

获取 ob 混淆块的 Successor 后续块的 Terminator 指令，如果是条件跳转 cmp+br，getOperand(0) 就得到 switchvar

```
    Instruction* getSwitchVar(BasicBlock &ob){
        BasicBlock *tmp=&ob;
        BasicBlock *succ = tmp->getTerminator()->getSuccessor(0);
        // errs() << "混淆block succe:" << *succ << '\n';   
        BranchInst *br = NULL;
        br = cast(succ->getTerminator());  
        if(br->isConditional()){  //收集所有case和phi控制点
            ICmpInst *ins=NULL;
            ins = cast(br->getOperand(0));
            Instruction *op0=NULL;
            op0 = cast(ins->getOperand(0));//op0就是switchvar
          return op0;
        }
        else{
          return NULL;
        }
 
    } 
```

3. 获得 case 分发块
==============

根据 switchvar 这个 Value 类的实例，遍历所有使用这个值的 Users ，挑出其中指令为 cmp 且 switchvar 为 op0 的 Users，标记为 case 分发块

```
      for (User *U : op0->users()) {//这里保存所有switch控制块
        if (Instruction *Inst = dyn_cast(U)) {
          // errs() << Inst->getParent()->getName()<< "\n";
          if(ICmpInst *Inst = dyn_cast(U)){
            inst.push_back(Inst);
            blk.push_back(Inst->getParent());
            num.push_back(dyn_cast(Inst->getOperand(1)));
            predicate.push_back(Inst->getPredicate());
            }
          else if(PHINode *Inst = dyn_cast(U)){
            phi=Inst;
            // errs() <<"phi:"<< phi->getParent()<< '\n';
            }
          }
        } 
```

4. 获得真实块  

===========

紧随 case 分发块之后一直到 ob 混淆块或者 return 的为真实块

```
      for(int i=0;i(&(**PI));
            vector::iterator it=find(blk.begin(), blk.end(), *PI);
            bool switch_=(*PI!=dyn_cast(phi->getParent()));
            if(switch_&&(it == blk.end())){
               blk2.push_back(*PI);
            }
            // if(it == blk.end()){
            //   if(*PI!=dyn_cast(phi->getParent())){
            //     blk2.push_back(*PI);
            //   }
            // }
          }
        } 
```

5. 获得混淆 phi 节点前驱块与 switchvar 赋值的一一对应
====================================

遍历 ob 块 phi 节点获取所有 switchvar，这里有一些 retdec 识别错误的 ConstantExpr 我们把它替换成 Constant

```
vectorphinum;
        vectorblk3;
        int a=phi->getNumIncomingValues();
        for(int i=0;igetNumIncomingValues();i++){//这里根据phi节点确定每条路径的SwitchVar值
          // blk3.push_back(phi->getIncomingBlock(i));
          Value* a=phi->getIncomingValue(i);
          // errs() << *a<< '\n';
          int j=0;
          while((!isa(a))&&(j<5)){
            j+=1;
            a=cast(a)->getOperand(1);
          }
          if(dyn_cast(a)){
            phinum.push_back(dyn_cast(a));
            blk3.push_back(phi->getIncomingBlock(i));
            }
        }
          for(int i=0;igetName()<< '\n';
            if(dyn_cast(phinum[i])){
              ConstantExpr* exp=cast(phinum[i]);
              ConstantInt *b=(ConstantInt *)ConstantInt::get(exp->getType(),0);
              Constant* cons=exp->getWithOperandReplaced(0,b);
              // errs() << "getAsInstruction:" <<*cons<< '\n';
              phinum[i]=cons;
            }
            // errs() << "num:" <<*phinum[i]<< '\n';
          } 
```

6. 模拟执行
=======

本来想学着用 klee 的，破单位网速太差一直下一半断掉，没办法只能自己试着写  

blk 向量存储了所有判断状态变量 switchvar 的的 block，blk2 向量存储了所有 ob 混淆 block 的前驱块，excute 向量存储了所有执行过的 block，

很明显每次执行首先获得 switchvar，然后沿着路径根据 blk 的 br 一直执行到 blk2 也就是 ob 混淆 block 的前驱块，

通过 ob 混淆块的 phi 节点更改 switchvar，执行下一条路径

这么执行几次后，最后一条路径到达 return，混淆结束，是一个单进单出的大循环

遍历得到所有路径之后，直接连接所有的真实块，把 blk 和 ob 都忽略掉，就可以干掉平坦化了  

```
      Constant* x1=dyn_cast(dyn_cast(op0)->getIncomingValue(0));
      BasicBlock* blknow=(phi->getParent())->getSingleSuccessor();
      BasicBlock* rtn;
      // errs() << "blknow:" <<*blknow<< '\n';
      APInt a1=x1->getUniqueInteger();
      bool branch;int pre;
      int i=1;
      errs() << "switch_var:" <excute;
    loop:
        ICmpInst* inst1=cast(blknow->getTerminator()->getOperand(0));   
        // errs() << "inst1:" <<*inst1<< '\n';
        APInt b;     
        //b=dyn_cast(inst1->getOperand(1))->getUniqueInteger(); 
        if(dyn_cast(inst1->getOperand(1))){
          ConstantExpr* exp=cast(inst1->getOperand(1));
          ConstantInt *bb=(ConstantInt *)ConstantInt::get(exp->getType(),0);
          Constant* cons=exp->getWithOperandReplaced(0,bb);
          b=dyn_cast(cons)->getUniqueInteger();
        }
        else{
          b=dyn_cast(inst1->getOperand(1))->getUniqueInteger(); 
        }
        pre=inst1->getPredicate();
        APInt rlt=a1-b;      
        int c=rlt.getSExtValue();
        succ_iterator SI = succ_begin(blknow);
        BasicBlock *PI = *SI;
        BasicBlock *E=*(++SI);
        // errs() << "PI:" <<*PI<< '\n';
        // errs() << "E:" <<*E<< '\n';
        if(c>0){          
          if(pre==33||pre==34||pre==35||pre==38||pre==39){branch=true;}
          else{branch=false;}  
        }
        else if(c==0){
          if(pre==32||pre==35||pre==37||pre==39||pre==41){branch=true;}
          else{branch=false;}  
        }
        else if(c<0){
          if(pre==33||pre==36||pre==37||pre==40||pre==41){branch=true;}
          else{branch=false;} 
        }
         
        // errs() << "branch:" <getName()<< '\n';
        vector::iterator it=find(blk.begin(), blk.end(), blknow);
        vector::iterator itt=find(blk3.begin(), blk3.end(), blknow);
        if((it != blk.end())&&(itt == blk3.end())){
          excute.push_back(blknow);
          goto loop;
        }
        else{
          if(isa(blknow->getTerminator())){
            errs() << "return_get:" <getTerminator()<< '\n';
            rtn=blknow;
            goto stop;
          }
          errs() << "block_end:" <getName()<< '\n';
          vector::iterator it2=find(blk3.begin(), blk3.end(), blknow);
          int pos=std::distance(blk3.begin(), it2);
          errs() << "phi_into:" <getUniqueInteger();
          errs() << "switch_var:" <getParent())->getSingleSuccessor();
          goto loop;
          // a1=phinum[i]->getUniqueInteger();
        }
        stop:    
          errs() << "cycle_count:" <getName()<< '\n'; 
```

执行结果：  

switch_var:-333430598  
block_pass:dec_label_pc_1f4f4  
block_pass:NodeBlock  
block_pass:LeafBlock  
block_end:LeafBlock  
phi_into:7  
switch_var:-531767870  
block_pass:dec_label_pc_1f376  
block_pass:NodeBlock8  
block_pass:LeafBlock6  
block_end:LeafBlock6  
phi_into:6  
switch_var:-1326600704  
block_pass:dec_label_pc_1f376  
block_pass:dec_label_pc_1f37a  
block_end:dec_label_pc_1f37a  
phi_into:3  
switch_var:281842383  
block_pass:dec_label_pc_1f4f4  
block_pass:NodeBlock  
block_pass:LeafBlock1  
block_end:LeafBlock1  
phi_into:5  
switch_var:868024320  
block_pass:dec_label_pc_1f4f4  
block_pass:dec_label_pc_1f4fa  
block_pass:dec_label_pc_1f3fe  
block_end:dec_label_pc_1f3fe  
phi_into:0  
switch_var:818644147  
block_pass:dec_label_pc_1f4f4  
block_pass:dec_label_pc_1f4fa  
block_pass:NodeBlock15  
block_pass:LeafBlock11  
block_pass:dec_label_pc_1f514  
return_get:0x22098a8  
cycle_count:6  
cycle_end:dec_label_pc_1f514  
excute block:dec_label_pc_1f4f4  
excute block:NodeBlock  
excute block:dec_label_pc_1f376  
excute block:NodeBlock8  
excute block:dec_label_pc_1f376  
excute block:dec_label_pc_1f4f4  
excute block:NodeBlock  
excute block:dec_label_pc_1f4f4  
excute block:dec_label_pc_1f4fa  
excute block:dec_label_pc_1f4f4  
excute block:dec_label_pc_1f4fa  
excute block:NodeBlock15  
excute block:LeafBlock11

这个函数除了 return 只有 dec_label_pc_1f3fe 这个 block 是真实块

7. 连接真实块  

===========

然后，从 excute 中过滤出所有真实块存入 excute_real

```
        vectorexcute_real;
        for(int i=0;igetName()<< '\n';
          vector::iterator it=find(blk_real.begin(), blk_real.end(),excute[i]);
          if(it != blk.end()){
            excute_real.push_back(excute[i]);
          }
        }
        excute_real.push_back(rtn);
        for(int i=0;i
```

最后直接把 excute_real 里的 block 用 br 连接起来，再用 opt -sccp -ipsccp -simplifycfg -adce 优化就可以还原了，这个函数比较简单，优化完只剩 1 个块了

对比一下刚开始的 cfg

![](https://bbs.pediy.com/upload/attach/202103/877073_QXGRXJSUB88A2D9.jpg)

附录
==

1. 获取后续块数量

```
          pred_iterator PI = pred_begin(tmp), E = pred_end(tmp);
          for (;PI != E; ++PI) {
            numPre+=1;
          }

```

2.phi 指令遍历

```
        int a=phi->getNumIncomingValues();
        for(int i=0;igetNumIncomingValues();i++){//这里根据phi节点确定每条路径的SwitchVar值
          Value* a=phi->getIncomingValue(i);
          // errs() << *a<< '\n';
          int j=0;
          while((!isa(a))&&(j<5)){
            j+=1;
            a=cast(a)->getOperand(1);
          }
          if(dyn_cast(a)){
            phinum.push_back(dyn_cast(a));
            blk3.push_back(phi->getIncomingBlock(i));
            }
        } 
```

3. 替换识别错误的 ConstantExpr 为 Constant，把那个错误识别的 globalvalue 替换为 0 就可以了

```
            if(dyn_cast(phinum[i])){
              ConstantExpr* exp=cast(phinum[i]);
              ConstantInt *b=(ConstantInt *)ConstantInt::get(exp->getType(),0);
              Constant* cons=exp->getWithOperandReplaced(0,b);
              // errs() << "getAsInstruction:" <<*cons<< '\n';
              phinum[i]=cons;
            } 
```

4.Constant 总结

ConstantExpr 常量表达式，在 Constants.h 里查看它的相关函数，介绍在这里 [https://llvm.org/docs/LangRef.html#constant-expressions](https://llvm.org/docs/LangRef.html#constant-expressions，)  

长的像一个 Instruction，可以通过 getAsInstruction 转换为 Instruction 就可以 getOperand 和 opCode 了

```
          ConstantExpr* exp=cast(inst1->getOperand(1));//把value转换为常量表达式
          ConstantInt *bb=(ConstantInt *)ConstantInt::get(exp->getType(),0);//声明一个值为0的常量
          Constant* cons=exp->getWithOperandReplaced(0,bb);//把常量表达式ConstantExpr第一个operand替换为0
          
          APInt a1=dyn_cast(cons)->getUniqueInteger()->getUniqueInteger();//获得Constant的整数值存储为APInt类型，不能直接为int类型 
```

5. 整体代码

```
#include "llvm/ADT/Statistic.h"
#include "llvm/IR/Function.h"
#include "llvm/Pass.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Intrinsics.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/Instruction.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Operator.h"
#include "llvm/IR/Intrinsics.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/Instructions.h"
#include "llvm/Transforms/Utils/Local.h" 
#include "llvm/Transforms/Utils/BasicBlockUtils.h"//辣鸡注释少个s，浪费我2个小时
#include "llvm/IR/BasicBlock.h"
#include "llvm/ADT/Statistic.h"
#include "llvm/Transforms/IPO.h"
#include "llvm/IR/Module.h"
#include "llvm/Support/CommandLine.h"
#include "llvm/Support/Debug.h"
#include "llvm/IR/CFG.h"
#include "llvm/ADT/PostOrderIterator.h"
#include "llvm/IR/InstrTypes.h"
#include "llvm/IR/Metadata.h"
#include  #include  #include  using namespace std;
using namespace llvm;
 
#define DEBUG_TYPE "dflat"
 
STATISTIC(dflat, "Counts number of functions greeted");
 
namespace {
  // dflat - The first implementation, without getAnalysisUsage.
  struct dflat : public FunctionPass {
    static char ID; // Pass identification, replacement for typeid
    dflat(): FunctionPass(ID) {}
 
    bool runOnFunction(Function &F) override {
      Function *tmp=&F;
      BasicBlock *ob=NULL;
      bool flatOrNot=hasFlatten(*tmp,&ob);
      // errs() << "混淆block:" << *ob << '\n';
      if(flatOrNot){
        Instruction *var=getSwitchVar(*ob);
        // errs() << "getSwitchVar:" << *var << '\n';
        match(*tmp,*var);
      }
      return false;
    }
    bool hasFlatten(Function &F,BasicBlock **ob){
      Function *tmp=&F;
      bool flat=false;
      for (Function::iterator bb = tmp->begin(); bb != tmp->end(); ++bb) {
          BasicBlock *tmp=&*bb;        
          int numPre=0;
          pred_iterator PI = pred_begin(tmp), E = pred_end(tmp);
          for (;PI != E; ++PI) {
            numPre+=1;
          }
          if(numPre>=6){
            flat=true;
            static BasicBlock *obb=&*tmp;
            *ob=&*tmp; 
            // errs() << "混淆blockterm:" << *obb<< '\n';
          }
      }
      return flat;
    }
 
    Instruction* getSwitchVar(BasicBlock &ob){
        BasicBlock *tmp=&ob;
        BasicBlock *succ = tmp->getTerminator()->getSuccessor(0);
        // errs() << "混淆block succe:" << *succ << '\n';   
        BranchInst *br = NULL;
        br = cast(succ->getTerminator());  
        if(br->isConditional()){  //收集所有case和phi控制点
            ICmpInst *ins=NULL;
            ins = cast(br->getOperand(0));
            Instruction *op0=NULL;
            op0 = cast(ins->getOperand(0));//op0就是switchvar
          return op0;
        }
        else{
          return NULL;
        }
 
    }
    void match(Function &F,Instruction& op){
      Function *tmp=&F;
      Instruction *op0=&op;
      vectorinst;
      vectorblk;
      vectorblk2;//真实块起点
      vectornum;
      vectorpredicate;
      PHINode* phi;
      for (User *U : op0->users()) {//这里保存所有switch控制块
        if (Instruction *Inst = dyn_cast(U)) {
          // errs() << Inst->getParent()->getName()<< "\n";
          if(ICmpInst *Inst = dyn_cast(U)){
            inst.push_back(Inst);
            blk.push_back(Inst->getParent());
            num.push_back(dyn_cast(Inst->getOperand(1)));
            predicate.push_back(Inst->getPredicate());
            }
          else if(PHINode *Inst = dyn_cast(U)){
            phi=Inst;
            // errs() <<"phi:"<< phi->getParent()<< '\n';
            }
          }
        }
          // for(int i=0;igetTerminator())->getOperand(0)<< '\n';
          //   errs() << "num:" <<*num[i]<< '\n';
          //   errs() << "predicate:" <(&(**PI));
            vector::iterator it=find(blk.begin(), blk.end(), *PI);
            bool switch_=(*PI!=dyn_cast(phi->getParent()));
            if(switch_&&(it == blk.end())){
               blk2.push_back(*PI);
            }
          }
        }
        vectorblk_real;
        for(int i=0;i1){
            blk_real.push_back(blk2[i]);}
          }        
          // for(int i=0;iphinum;
        vectorblk3;
        for(int i=0;igetNumIncomingValues();i++){//这里根据phi节点确定每条路径的SwitchVar值
          // blk3.push_back(phi->getIncomingBlock(i));
          Value* a=phi->getIncomingValue(i);
          // errs() << *a<< '\n';
          int j=0;
          while((!isa(a))&&(j<5)){
            j+=1;
            a=cast(a)->getOperand(1);
          }
          if(dyn_cast(a)){
            phinum.push_back(dyn_cast(a));
            blk3.push_back(phi->getIncomingBlock(i));
            }
        }
          for(int i=0;igetName()<< '\n';
            if(dyn_cast(phinum[i])){
              ConstantExpr* exp=cast(phinum[i]);
              ConstantInt *b=(ConstantInt *)ConstantInt::get(exp->getType(),0);
              Constant* cons=exp->getWithOperandReplaced(0,b);
              // errs() << "getAsInstruction:" <<*cons<< '\n';
              phinum[i]=cons;
            }
            // errs() << "num:" <<*phinum[i]<< '\n';
          }  
//开始  
      Constant* x1=dyn_cast(dyn_cast(op0)->getIncomingValue(0));
      BasicBlock* blknow=(phi->getParent())->getSingleSuccessor();
      BasicBlock* rtn;
      // errs() << "blknow:" <<*blknow<< '\n';
      APInt a1=x1->getUniqueInteger();
      bool branch;int pre;
      int i=1;
      errs() << "switch_var:" <excute;
    loop:
        ICmpInst* inst1=cast(blknow->getTerminator()->getOperand(0));   
        // errs() << "inst1:" <<*inst1<< '\n';
        APInt b;     
        //b=dyn_cast(inst1->getOperand(1))->getUniqueInteger(); 
        if(dyn_cast(inst1->getOperand(1))){
          ConstantExpr* exp=cast(inst1->getOperand(1));
          ConstantInt *bb=(ConstantInt *)ConstantInt::get(exp->getType(),0);
          Constant* cons=exp->getWithOperandReplaced(0,bb);
          b=dyn_cast(cons)->getUniqueInteger();
        }
        else{
          b=dyn_cast(inst1->getOperand(1))->getUniqueInteger(); 
        }
        pre=inst1->getPredicate();
        APInt rlt=a1-b;      
        int c=rlt.getSExtValue();
        succ_iterator SI = succ_begin(blknow);
        BasicBlock *PI = *SI;
        BasicBlock *E=*(++SI);
        if(c>0){ //这里enum Predicate定义在llvm/InstrTypes.h，不同版本可能不一样     
          if(pre==33||pre==34||pre==35||pre==38||pre==39){branch=true;}
          else{branch=false;}  
        }
        else if(c==0){
          if(pre==32||pre==35||pre==37||pre==39||pre==41){branch=true;}
          else{branch=false;}  
        }
        else if(c<0){
          if(pre==33||pre==36||pre==37||pre==40||pre==41){branch=true;}
          else{branch=false;} 
        }        
        // errs() << "branch:" <getName()<< '\n';
        vector::iterator it=find(blk.begin(), blk.end(), blknow);
        vector::iterator itt=find(blk3.begin(), blk3.end(), blknow);
        if((it != blk.end())&&(itt == blk3.end())){
          excute.push_back(blknow);
          goto loop;
        }
        else{
          if(isa(blknow->getTerminator())){
            errs() << "return_get:" <getTerminator()<< '\n';
            rtn=blknow;
            goto stop;
          }
          errs() << "block_end:" <getName()<< '\n';
          vector::iterator it2=find(blk3.begin(), blk3.end(), blknow);
          int pos=std::distance(blk3.begin(), it2);
          errs() << "phi_into:" <getUniqueInteger();
          errs() << "switch_var:" <getParent())->getSingleSuccessor();
          goto loop;
        }
        stop:    
          errs() << "cycle_count:" <getName()<< '\n';
//结束
        vectorexcute_real;
        for(int i=0;igetName()<< '\n';
          vector::iterator it=find(blk_real.begin(), blk_real.end(),excute[i]);
          if(it != blk.end()){
            excute_real.push_back(excute[i]);
          }
        }
        excute_real.push_back(rtn); 
        for(int i=0;i X("dflat", "dflat Pass"); 
```

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)