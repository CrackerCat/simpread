> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271867.htm)

> [原创] 基于 llvm 的变量轮转混淆 pass 实现

前言
--

之前有了解过一点点 VMP 的虚拟机，其中存在一种混淆机制，就是在执行之后将虚拟机的寄存器轮转，这样每个指令的寄存器就不一样了，极大的增加了数据流分析的难度。可能理解有点问题，但是这种思想也可以应用于源码级混淆。所以我就基于 LLVM 实现了一种变量轮转的混淆方式，发现能够能够阻碍数据流分析，迷惑看代码的人。  
![](https://bbs.pediy.com/upload/attach/202203/948449_F5B3J3WXJWFFHDH.png)

设计
--

在 LLVM 中，函数内部的变量都是通过 allocainst 进行分配的，其分配的结果是一个指针类型，在 IDA 中对应的就是栈空间。  
![](https://bbs.pediy.com/upload/attach/202203/948449_3QDCDXN849YMNA4.png)  
在这里，由于 LLVM 的分配类型并不相同，不能像虚拟寄存器那样方便轮转。所以先想个办法将这些分配的 allocainst 统一一下。

*   由于 allocainst 返回的是一个指针，代表的该变量存储的位置。所以我们可以一开始就将所有的 allocainst 收集起来。
*   统一用一个 allocainst 分配一个巨大的 uchar 类型的 buffer 空间，用于这些 allocainst 分配。然后计算每一个 alloca 指令相对于这个 buffer 空间的偏移地址。
*   将引用这些原来 allocainst 的指令的操作数替换成，分配 buffer + 偏倚地址的指针。到这里实现将所有的 allocainst 统一起来。
*   然后就是构造变量轮转函数了，这个直接用 llvm 的 irbuilder 即可。
*   接下来就是变量轮转，由于空间本身不是环形空间，轮转的位数也有限制，必须一整个一整个变量的轮转。且要保证每个基本块进入时相对于原来没有发生移位。

具体代码实现
------

### 构造轮转函数

具体算法实现很简单，给一个数组指针，数组长度，移位位数，然后移位就可以了。  
![](https://bbs.pediy.com/upload/attach/202203/948449_UXH6YJYMM7KBS5P.png)

> c 代码实现如下

```
Function *VariableRotation::createRotateFunc(Module *m,PointerType *ptrType,Twine &name)
{
    std::vector params;
    params.push_back(ptrType);
    params.push_back(Type::getInt32Ty(m->getContext()));
    params.push_back(Type::getInt32Ty(m->getContext()));
    Type *rawType=ptrType->getPointerElementType();
    FunctionType *funcType=FunctionType::get(Type::getVoidTy(m->getContext()),params,false);
    Function *func=Function::Create(funcType,GlobalValue::PrivateLinkage,name,m);
    BasicBlock *entry1=BasicBlock::Create(m->getContext(),"entry1",func);
    BasicBlock *cmp1=BasicBlock::Create(m->getContext(),"cmp1",func);
    BasicBlock *entry2=BasicBlock::Create(m->getContext(),"entry2",func);
    BasicBlock *cmp2=BasicBlock::Create(m->getContext(),"cmp2",func);
    BasicBlock *shift=BasicBlock::Create(m->getContext(),"shift",func);
    BasicBlock *end2=BasicBlock::Create(m->getContext(),"end2",func);
    BasicBlock *end1=BasicBlock::Create(m->getContext(),"end1",func);
    Function::arg_iterator iter=func->arg_begin();
    Value *ptr=iter;
    Value *len=++iter;
    Value *times=++iter;
    IRBuilder<> irb(entry1);
    Value *zero=ConstantInt::get(irb.getInt32Ty(),0);
    Value *one=ConstantInt::get(irb.getInt32Ty(),1);
    AllocaInst *i=irb.CreateAlloca(irb.getInt32Ty());
    AllocaInst *j=irb.CreateAlloca(irb.getInt32Ty());
    AllocaInst *tmp=irb.CreateAlloca(rawType);
    AllocaInst *array=irb.CreateAlloca(ptrType);
    irb.CreateStore(zero,j);
    irb.CreateStore(ptr,array);
    irb.CreateBr(cmp1);
 
    irb.SetInsertPoint(cmp1);
    irb.CreateCondBr(irb.CreateICmpSLT(irb.CreateLoad(j),times),entry2,end1);
 
    irb.SetInsertPoint(entry2);
    irb.CreateStore(zero,i);
    irb.CreateStore(irb.CreateLoad(irb.CreateInBoundsGEP(irb.CreateLoad(array),{zero})),tmp);
    irb.CreateBr(cmp2);
 
    irb.SetInsertPoint(cmp2);
    irb.CreateCondBr(irb.CreateICmpSLT(irb.CreateLoad(i),irb.CreateSub(len,one)),shift,end2);
 
    irb.SetInsertPoint(shift);
    Value *ival=irb.CreateLoad(i);
    Value *arr=irb.CreateLoad(array);
    irb.CreateStore(irb.CreateLoad(irb.CreateInBoundsGEP(arr,{irb.CreateAdd(ival,one)})),irb.CreateInBoundsGEP(rawType,arr,{ival}));
    irb.CreateStore(irb.CreateAdd(ival,one),i);
    irb.CreateBr(cmp2);
 
    irb.SetInsertPoint(end2);
    irb.CreateStore(irb.CreateAdd(irb.CreateLoad(j),one),j);
    irb.CreateStore(irb.CreateLoad(tmp),irb.CreateInBoundsGEP(irb.CreateLoad(array),{irb.CreateSub(len,one)}));
    irb.CreateBr(cmp1);
 
    irb.SetInsertPoint(end1);
    irb.CreateRetVoid();
    return func;
} 
```

这里并不是重点，熟悉 llvm 的 ir 的很快就能写出来。

### 处理函数的 AllocaInst

*   函数第一个参数代表着要处理的 llvm 函数，第二个代表着统计出的函数的 allocainst 集合，第三个代表之前生成的移位函数。  
    ![](https://bbs.pediy.com/upload/attach/202203/948449_H65TT7R8HWGSZTQ.png)
*   然后开始处理，先遍历 AllocaInst，然后使用 getTypeAllocSize 分别获取该指令分配的空间的大小，用于计算 value_map，其实就是表示这 AllocaInst 到 buffer 的偏移地址的一个映射。  
    ![](https://bbs.pediy.com/upload/attach/202203/948449_GV5CSD696WEBD9F.png)
*   计算完成各个变量的偏移地址后，同时也得到了 buffer 应该有的大小，保存下来，并且使用 AllocaInst 分配空间。  
    ![](https://bbs.pediy.com/upload/attach/202203/948449_YATD3GBRT6ANTKX.png)
*   开始遍历每个基本块。  
    ![](https://bbs.pediy.com/upload/attach/202203/948449_2ERSUUQUXXQXF6C.png)
*   对于每个基本块，开始进行处理每个引用了原来变量的指令，替换成 GEP buffer 生成的指针，并将指针类型转换成原 allocainst 的指针类型。然后在基本块内部维护一个 int，保存到当前指令的时候，已经轮转了多少位了，然后通过随机数进行插入轮转函数，更新 int。  
    ![](https://bbs.pediy.com/upload/attach/202203/948449_NC7FPQ7WY532NGH.png)
*   处理完基本块之后，查看轮转了多少位，如果不是 0 的话则需要修正一下，让轮转的位数为 0(在取模意义下)  
    ![](https://bbs.pediy.com/upload/attach/202203/948449_N8C44NZXT5H5FZH.png)

### 完整代码

```
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/CFG.h"
#include "llvm/ADT/MapVector.h"
#include "llvm/ADT/SetVector.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/Utils/Cloning.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
#include "assert.h"
#include #include #include #include #include using namespace llvm;
namespace
{
    struct VariableRotation : public ModulePass
    {
        static char ID;
 
           VariableRotation() : ModulePass(ID) {}
        Function *createRotateFunc(Module *m,PointerType *ptrType,Twine &name);
        void processFunction(Function &f,SetVector &shift);
        bool processVars(Function &f,SetVector &vars,Function *rotateFunc);
        bool isUsedByInst(Instruction *inst,Value *var)
        {
            for(Value *ops:inst->operands())
                if(ops==var)
                    return true;
            return false;
        }
 
        bool runOnModule(Module &m) override
        {
            Twine fname1=Twine("shiftFuncitonI8");
            Function *shiftFunc=createRotateFunc(&m,Type::getInt8PtrTy(m.getContext()),fname1);
            SetVector shifts;
            shifts.insert(shiftFunc);
            for(Function &f:m)
            {
                if(&f==shiftFunc)
                    continue;
                if(f.hasExactDefinition())
                    processFunction(f,shifts);
            }
            return true;
        }
      };
}
 
char VariableRotation::ID=0;
static RegisterPass X("varobfu", "varobfu");
bool VariableRotation::processVars(Function &f,SetVector &vars,Function *rotateFunc)
{
    if(vars.size()<2)
        return false;
    IRBuilder<> irb(&*f.getEntryBlock().getFirstInsertionPt());
    Value *zero=ConstantInt::get(irb.getInt32Ty(),0);
    DataLayout data=f.getParent()->getDataLayout();
    int space=0;
    SetVector value_map;
    printf("function: %s\n",f.getName());
    for(int i=0;igetAllocatedType());
    }
    ArrayType *arrayType=ArrayType::get(irb.getInt8Ty(),space);
    AllocaInst *array=irb.CreateAlloca(arrayType);
    int prob=30;
    for(BasicBlock &bb:f)
    {
        int offset=0;
        BasicBlock::iterator iter=bb.getFirstInsertionPt();
        if(&bb==&f.getEntryBlock())
            iter++;
        while(iter!=bb.end())
        {
            Instruction *inst=&*iter;
            irb.SetInsertPoint(inst);
            for(int i=0;igetType());
                    int c=0;
                    for(Value *ops:inst->operands())
                    {
                        if(ops==vars[i])
                            inst->setOperand(c,cast);
                        c++;
                    }
                    break;
                }
            iter++;
        }
        if(offset!=0)
        {
            irb.SetInsertPoint(bb.getTerminator());
            irb.CreateCall(FunctionCallee(rotateFunc),{irb.CreateGEP(array,{zero,zero}),ConstantInt::get(irb.getInt32Ty(),space),ConstantInt::get(irb.getInt32Ty(),(space-value_map[offset])%space)});
        }
 
    }
    return true;
}
void VariableRotation::processFunction(Function &f,SetVector &shift)
{
    SetVector list;
    for(BasicBlock &bb:f)
        for(Instruction &instr:bb)
            if(isa(instr))
            {
                AllocaInst *a=(AllocaInst*)&instr;
                list.insert(a);
            }
    if(processVars(f,list,shift[0]))
    {
        for(AllocaInst *a:list)
            a->eraseFromParent();
    }
}
Function *VariableRotation::createRotateFunc(Module *m,PointerType *ptrType,Twine &name)
{
    std::vector params;
    params.push_back(ptrType);
    params.push_back(Type::getInt32Ty(m->getContext()));
    params.push_back(Type::getInt32Ty(m->getContext()));
    Type *rawType=ptrType->getPointerElementType();
    FunctionType *funcType=FunctionType::get(Type::getVoidTy(m->getContext()),params,false);
    Function *func=Function::Create(funcType,GlobalValue::PrivateLinkage,name,m);
    BasicBlock *entry1=BasicBlock::Create(m->getContext(),"entry1",func);
    BasicBlock *cmp1=BasicBlock::Create(m->getContext(),"cmp1",func);
    BasicBlock *entry2=BasicBlock::Create(m->getContext(),"entry2",func);
    BasicBlock *cmp2=BasicBlock::Create(m->getContext(),"cmp2",func);
    BasicBlock *shift=BasicBlock::Create(m->getContext(),"shift",func);
    BasicBlock *end2=BasicBlock::Create(m->getContext(),"end2",func);
    BasicBlock *end1=BasicBlock::Create(m->getContext(),"end1",func);
    Function::arg_iterator iter=func->arg_begin();
    Value *ptr=iter;
    Value *len=++iter;
    Value *times=++iter;
    IRBuilder<> irb(entry1);
    Value *zero=ConstantInt::get(irb.getInt32Ty(),0);
    Value *one=ConstantInt::get(irb.getInt32Ty(),1);
    AllocaInst *i=irb.CreateAlloca(irb.getInt32Ty());
    AllocaInst *j=irb.CreateAlloca(irb.getInt32Ty());
    AllocaInst *tmp=irb.CreateAlloca(rawType);
    AllocaInst *array=irb.CreateAlloca(ptrType);
    irb.CreateStore(zero,j);
    irb.CreateStore(ptr,array);
    irb.CreateBr(cmp1);
 
    irb.SetInsertPoint(cmp1);
    irb.CreateCondBr(irb.CreateICmpSLT(irb.CreateLoad(j),times),entry2,end1);
 
    irb.SetInsertPoint(entry2);
    irb.CreateStore(zero,i);
    irb.CreateStore(irb.CreateLoad(irb.CreateInBoundsGEP(irb.CreateLoad(array),{zero})),tmp);
    irb.CreateBr(cmp2);
 
    irb.SetInsertPoint(cmp2);
    irb.CreateCondBr(irb.CreateICmpSLT(irb.CreateLoad(i),irb.CreateSub(len,one)),shift,end2);
 
    irb.SetInsertPoint(shift);
    Value *ival=irb.CreateLoad(i);
    Value *arr=irb.CreateLoad(array);
    irb.CreateStore(irb.CreateLoad(irb.CreateInBoundsGEP(arr,{irb.CreateAdd(ival,one)})),irb.CreateInBoundsGEP(rawType,arr,{ival}));
    irb.CreateStore(irb.CreateAdd(ival,one),i);
    irb.CreateBr(cmp2);
 
    irb.SetInsertPoint(end2);
    irb.CreateStore(irb.CreateAdd(irb.CreateLoad(j),one),j);
    irb.CreateStore(irb.CreateLoad(tmp),irb.CreateInBoundsGEP(irb.CreateLoad(array),{irb.CreateSub(len,one)}));
    irb.CreateBr(cmp1);
 
    irb.SetInsertPoint(end1);
    irb.CreateRetVoid();
    return func;
}
static RegisterStandardPasses Y(PassManagerBuilder::EP_EarlyAsPossible,
  [](const PassManagerBuilder &Builder, legacy::PassManagerBase &PM) {
    PM.add(new VariableRotation());
  }); 
```

效果
--

编译成 so，并应用于 ll 文件。

```
clang `llvm-config --cxxflags` -Wl,-znodelete -fno-rtti -fPIC -shared VarObfu.cpp -o Var.so `llvm-config --ldflags`
clang -S -emit-llvm test.cpp -o test.ll
opt -load Var.so -var test.ll -S -o test_out.ll
llvm-as test_out.ll
clang test_out.ll -o test

```

IDA 打开，可以看到 f5 代码中的变量的意义已经完全不一样了，如果忽略轮转函数，则代码完全没有意义，而想要恢复，则需要脑子一点点去演算变量地址才行。  
![](https://bbs.pediy.com/upload/attach/202203/948449_8A85DB9SMCGB88N.png)

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 59 分钟前 被 R1mao 编辑 ，原因：

[#软件保护](forum-4-1-3.htm)