> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282305.htm)

> [原创] 自实现一个 LLVM Pass 以及 OLLVM 简单的魔改

前言
==

总结 LLVM，OLLVM 相关知识，自实现一个 Pass，魔改 OLLVM 中的 Pass，加深 OLLVM 的理解。  
环境: LLVM 9.0、CMake:3.16.3

LLVM 介绍及编译
==========

LLVM 介绍
-------

首先介绍下 LLVM 是什么，借用下面的这张图说明一下，  
![](https://bbs.kanxue.com/upload/attach/202406/978849_UWGPDM4BQUT8QE5.webp)  
LLVM 是一套编译器基础设施项目，分为前端后端和中间表示（IR Intermediate Representation）, 从图中可以看到有曾经用过的 Clang，Clang 就是 llvm 用于处理 C 和 C++ 的前端。除了 Clang，LLVM 还有支持其他语言的前端，例如 Rust、Swift、Python 等。这些前端会将对应语言的源代码转换成 LLVM 的中间表示，使得 LLVM 能够处理多种语言的编译需求。后端部分则负责将 LLVM IR 转换成特定目标平台的机器码或汇编代码。  
OLLVM 使用的就是 LLVM IR 来处理源代码，它能够获取到程序的结构和控制流信息，通过对这些信息的处理增强代码混淆的效果。  
LLVM IR 有两种文件格式. ll 和. bc，.ll 文件和 .bc 文件都是 LLVM 中间表示的不同表示形式，.ll 文件是文本形式的可读表示，方便分析和调试；.bc 文件是二进制形式的紧凑表示，用于编译过程中的处理和优化，所以主要看. ll 格式文件内容  
再介绍一些 IR 相关的概念 1.Pass 是用于处理 IR 的关键组成部分，LLVM 中自带的 Pass 主要是 lib/Transforms 中的. cpp 文件 2.IR 结构，Module->Function->Basic Block->Instruction，这些是 IR 的不同层次结构从左到右是一对多的包含关系，而且从命名来看也比较好理解，Module 对应. c 文件内的整个源码，Function 就是函数，Basic Block 是基本块，每个基本块以一个终止指令（例如 ret、br 等）结尾，或者以一个无条件分支（如 br 指令）指向其他基本块，Instruction 就是指令了，看一个 ll 文件内容：

```
define dso_local i32 @main(i32 %argc, i8** %argv) #0 函数 {
...
first:                                            ; preds = %loopEnd
  %.reload = load volatile i32, i32* %.reg2mem 指令
  %cmp = icmp eq i32 %.reload, 2
  %1 = select i1 %cmp, i32 769163483, i32 -1060808858
  store i32 %1, i32* %switchVar
  br label %loopEnd
first就是一个名为first的基本块以br结尾
if.then:                  S                        ; preds = %loopEnd2
  %2 = load i8*, i8** %str, align 8
  %3 = load i8*, i8** @globalString, align 8
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str.5, i64 0, i64 0), i8* %2, i8* %3)
  store i32 -1907967449, i32* %switchVar
  br label %loopEnd2
 
if.else:                                          ; preds = %loopEnd
  %4 = load i32, i32* %argc.addr, align 4
  %cmp1 = icmp eq i32 %4, 3
  %5 = select i1 %cmp1, i32 1950105811, i32 1575411630
  store i32 %5, i32* %switchVar
  br label %loopEnd
...
}

```

LLVM 编译
-------

安装并编译 llvm，这里我选的是比较老的版本 llvm 9.0，因为新版 llvm 更新了很多东西，包括最主要的 PassManager，网上可参考资料比较少，而且只是学习 ollvm 的话学习思路比较重要。

```
https://github.com/llvm/llvm-project/releases

```

从官方的 git 库下载下来，之后编译，有两种一种是 release 版本一种是 debug 版本，普遍的说法是 debug 版本可用来调试自己写的 Pass，在后面自己写 Pass 的时候感觉通过代码里打印的方式调试也还好，所以这里我编译的是 release 版。  
编译没啥注意事项建文件打命令即可，-DLLVM_ENABLE_PROJECTS 这个参数把 clang 编译了，后面要用到，这里还有一个常用的工具 lldb 这个调试 Pass 需要这个。

```
cd到解压的文件夹里
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE=release -DLLVM_ENABLE_PROJECTS="clang" ../llvm

```

要编译几个小时，编译好之后，把编译后的 bin 文件路径加到环境变量里。

在源码外开发一个函数名变量名加密 Pass
=====================

开发 Pass，先简单配置一下编辑器的代码提示，我用的 VsCode，首先在一级目录下建一个文件夹. vscode 这个文件夹下建一个 c_cpp_properties.json 文件，内容为：

```
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "{LLVM解压文件路径}/build_debug/include",
                "{LLVM解压文件路径}/llvm/include"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "c17",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-clang-x64"
        }
    ],
    "version": 4
}

```

准备好开发环境后，开始在源码外开发 Pass，先按看官方文档中的流程建好文件。

```
https://llvm.org/docs/CMake.html#developing-llvm-passes-out-of-source

```

我配置的时候不同的是最外层的 CMakeLists.txt，要根据报错自己加一些配置。

```
cmake_minimum_required(VERSION 3.16)
project(HelloPass)#这个可以随便取
 
# 设置 LLVM 路径
set(LLVM_DIR "{llvm源码路径}/build/lib/cmake/llvm")
find_package(LLVM REQUIRED CONFIG)
 
project(ProjectName)
set(CMAKE_CXX_STANDARD 17)
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
include(AddLLVM) //支持add_llvm_library
 
SET(CMAKE_CXX_FLAGS "-Wall -fno-rtti")
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})
include_directories(${LLVM_INCLUDE_DIRS})
 
add_subdirectory(下级目录名)

```

配置好后，写一个简单的函数名和变量名加密的 Pass，实现方式主要是通过 Module 遍历 Function，getName 获取 Function 名后加密函数名，这里用 MD5 代替加密函数，之后通过 setName 再设置成加密后的函数名。变量名同理通过 Module 即可获取到所有变量名，这里以全局变量为例还是 getName-> 加密 ->setName。  
主体代码：

```
namespace {
  struct EncodeFunctionName2 : public ModulePass {
    static char ID;
    EncodeFunctionName2() : ModulePass(ID) {}
    bool runOnFunction(Function &F) {
      Module *M = F.getParent();
     LLVMContext &context = M->getContext();
      errs() << "EncodeFunctionName: " << F.getName() << " -> ";
       
      if (F.getName().compare("main") != 0&&F.getName().compare("main") != 0) {
 
        llvm::MD5 Hasher;
        llvm::MD5::MD5Result Hash;
        Hasher.update("kanxue_");
        Hasher.update(F.getName());
        Hasher.final(Hash);
 
        SmallString<32> HexString;
        llvm::MD5::stringifyResult(Hash, HexString);
 
        F.setName(HexString);
      }
          errs() << F.getName() << "\r\n";
           return true;//当修改了IR代码的话返回True
    }
     bool runOnModule(Module &M) override {
      llvm::MD5 Hasher;
        llvm::MD5::MD5Result Hash;
 
        for (auto &F : M) {
        runOnFunction(F);
      }
       for (GlobalVariable &GV : M.globals()) {
                StringRef oldName = GV.getName();
                Hasher.update("kanxue_");
                Hasher.update(oldName);
                Hasher.final(Hash);
                SmallString<32> HexString;
                 llvm::MD5::stringifyResult(Hash, HexString);
                errs() << "EncodeVariableName: " << oldName << " -> ";
                // llvm::MD5::stringifyResult(Hash, HexString);
                // std::string newName = encryptName(oldName.str());
                GV.setName(HexString);
                // modified = true;
                errs() << GV.getName() << "\r\n";
            } 
      return true;
    }
 
      
  };
}
char EncodeFunctionName2::ID = 0;
//注册Pass
static RegisterPass X("encode", "Encode Function and Variable Name Pass",
                             false /* Only looks at CFG */,
                             false /* Analysis Pass */); 
```

在加密函数名时要注意跳过系统函数和 main 函数的加密。  
使用 opt 工具生成. ll 文件，opt 在设置了环境变量后就能直接用, 新版 llvm 加 -enable-new-pm=0。  
opt -load "./EncodeFunctionName2/LLVMEncodeFunctionName2.so" --encode -S ../../hello.ll -o ../hello.ll  
![](https://bbs.kanxue.com/upload/attach/202406/978849_6CMB2WRGJ8XYVW3.webp)  
查看写的 Pass 里的参数选项  
opt -load "./EncodeFunctionName2/LLVMEncodeFunctionName2.so" --help |grep encode  
![](https://bbs.kanxue.com/upload/attach/202406/978849_4Q3AVAUP57BEQQV.webp)  
结果：  
![](https://bbs.kanxue.com/upload/attach/202406/978849_2SWDPZ76RRQU2BK.webp)

控制流平坦化分析及魔改
===========

原版的 OllVM 在逆向的对抗中不断发展出了各种各样的反混淆方式，甚至都有了一键去平坦化的脚本，在分析过 OLLVM 的源码之后，我开始尝试对它进行简单的魔改，首先是控制流平坦化的 Pass, 在参考了网上大佬前辈的文章后我准备基于原版的控制流平坦化代码实现两个改进，一是不通过 loopEntry 分发 loopEntry 只作为入口，直接进入 loopEnd, 把分发流程做到 loopEnd 中，二是添加多个 loopEnd，这样还能做到多个分发块。  
先顺着原版的 OLLVM 代码说一下控制流平坦化的实现原理：

```
bool Flattening::runOnFunction(Function &F) {
  Function *tmp = &F;
  // Do we obfuscate
  // if (toObfuscate(flag, tmp, "fla")) {
    if (flatten(tmp)) {
      ++Flattened;
    }
  // }
 
  return false;
}

```

首先运行的是 runOnFunction，因为我是在源码树外开发的 Pass 所以把原版的参数判断代码注释了，主要调用了 faltten 函数

```
vector origBB;
BasicBlock *loopEntry;
BasicBlock *loopEnd;
LoadInst *load;
SwitchInst *switchI;
AllocaInst *switchVar;
 
// SCRAMBLER
char scrambling_key[16];
llvm::cryptoutils->get_bytes(scrambling_key, 16);
// END OF SCRAMBLER
 
// Lower switch
 FunctionPass *lower = createLowerSwitchPass();
 lower->runOnFunction(*f); 
```

这一段是生成用于加密的 key 和运行 LowerSwitch 函数，用于优化 Switch 语句，会把 Switch 语句变成更基础的控制流结构，如一系列的条件分支或跳转指令，说白了就是会改变逻辑把代码结果变得更复杂，这是 LLVM 自带的 Pass，也算是一次混淆了。

```
  // Save all original BB
  for (Function::iterator i = f->begin(); i != f->end(); ++i) {
    BasicBlock *tmp = &*i;
    origBB.push_back(tmp);
 
    BasicBlock *bb = &*i;
    if (isa(bb->getTerminator())) {
      return false;
    }
  }
 
  // Nothing to flatten
  if (origBB.size() <= 1) {
    return false;
  }
 
  // Remove first BB
  origBB.erase(origBB.begin());
 
  // Get a pointer on the first BB
  Function::iterator tmp = f->begin();  //++tmp;
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
// 指令多大于一可能还有cmp指令所以要和跳转指令一起分割
    if (insert->size() > 1) {
      --i;
    }
 
    BasicBlock *tmpBB = insert->splitBasicBlock(i, "first");
    origBB.insert(origBB.begin(), tmpBB);
  }
 
  // Remove jump 移除掉splitBasicBlock函数分割后第一个块跳转到frist块的指令。
  insert->getTerminator()->eraseFromParent(); 
```

这一段主要是保存一份所有原始的代码块，在原始代码块被修改时方便取用，之后把原始代码块中的第一个入口代码块和后面的代码分割开，这样做的目的是为了插入 loopEntry 这个用于分发的代码块，再通过 loopEntry 和后面的代码块连接起来并且把入口块用于跳转的指令独立分割成一个名为 first 的块加入到 loopEntry 分发中，。  
类似这样, entry 作为入口代码块平坦化后会设置 switchVar 为 first 对应的 switchVar，再通过 first 来进入第二个代码块，这样就不能直接看出正常逻辑 entry 之后该执行的真实代码块是哪个了  
![](https://bbs.kanxue.com/upload/attach/202406/978849_694935WYG78744Y.webp)  
![](https://bbs.kanxue.com/upload/attach/202406/978849_DWVNP7HBSB4W4XE.webp)

```
// Create switch variable and set as it 创建switchVar变量并设置一个随机生成的值
switchVar =
    new AllocaInst(Type::getInt32Ty(f->getContext()), 0, "switchVar", insert);
new StoreInst(
    ConstantInt::get(Type::getInt32Ty(f->getContext()),
                     llvm::cryptoutils->scramble32(0, scrambling_key)),
    switchVar, insert);
 
// Create main loop 第三个参数是插入在哪个块之前
loopEntry = BasicBlock::Create(f->getContext(), "loopEntry", f, insert);
loopEnd = BasicBlock::Create(f->getContext(), "loopEnd", f, insert);
 
load = new LoadInst(switchVar, "switchVar", loopEntry);
 
// Move first BB on top
insert->moveBefore(loopEntry);
BranchInst::Create(loopEntry, insert);
 
// loopEnd jump to loopEntry
BranchInst::Create(loopEntry, loopEnd);
 
BasicBlock *swDefault =
    BasicBlock::Create(f->getContext(), "switchDefault", f, loopEnd);
BranchInst::Create(loopEnd, swDefault);
 
// Create switch instruction itself and set condition
switchI = SwitchInst::Create(&*f->begin(), swDefault, 0, loopEntry);
switchI->setCondition(load);
 
// Remove branch jump from 1st BB and make a jump to the while
f->begin()->getTerminator()->eraseFromParent();
 
BranchInst::Create(loopEntry, &*f->begin());
 
// Put all BB in the switch
for (vector::iterator b = origBB.begin(); b != origBB.end();
     ++b) {
  BasicBlock *i = *b;
  ConstantInt *numCase = NULL;
 
  // Move the BB inside the switch (only visual, no code logic)
  i->moveBefore(loopEnd);
 
  // Add case to switch
  //llvm::cryptoutils->scramble32这个函数的第一个参数是加密字段，第二个是key，switchI->getNumCases的值是从0开始递增那这里生成的跳转条件就和llvm::cryptoutils->scramble32(0, scrambling_key)是一样的
  numCase = cast(ConstantInt::get(
      switchI->getCondition()->getType(),
      llvm::cryptoutils->scramble32(switchI->getNumCases(), scrambling_key)));
  switchI->addCase(numCase,i);
} 
```

这一段是用于生成 loopEntry，loopEnd，switchDefault 三个代码块，在 loopEntry 中添加一个 switch 语句，并以 SwitchVar 存储条件值，原始代码块通过赋值这个变量即可完成跳转，并遍历所有原始代码块，以 switch 结构中目前 case 的个数进行加密得到的数字作为 case 的条件，就是 0，1，2，3... 挨个加密所以不会出现重复的 case 条件，再把跳转原始基本块的条件加入到 switch 语句中。

```
// Recalculate switchVar 这里开始遍历所有块去除块中最后的br
for (vector::iterator b = origBB.begin(); b != origBB.end();
     ++b) {
  BasicBlock *i = *b;
  ConstantInt *numCase = NULL;
 
  // Ret BB
  if (i->getTerminator()->getNumSuccessors() == 0) {
    continue;
  }
 
  if (i->getTerminator()->getNumSuccessors() == 1) {//1个的类似if{} 判断是一个block条件为真后的代码是一个代码块
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
 
  // If it's a conditional jump
  if (i->getTerminator()->getNumSuccessors() == 2) { //有两个的类似if(){}else{}
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
}
fixStack(f);
return true; 
```

最后这段代码完成收尾工作，之前已经把 loopEntry 中的 switch 跳转做好了，但是原始代码块的最后的跳转还是原来的逻辑，这一块还没有处理，主要做两点通过原来的跳转指令获取要跳转到的代码块，再通过 switchI->findCaseDest 获取到是对应哪个 Case，把 Case 的条件赋值给 switchVar，这里就可以移除原始跳转指令了，再添加一个跳转 loopEnd 的指令，loopEnd 会跳转到 loopEntry，这样进入 Switch 分发器了，最后修复 PHI 和堆栈，fixStack 这一块和这次魔改没太大关系就不说了。  
接下来开始实现之前说的两个改进，首先是不通过 loopEntry 分发 loopEntry 只作为入口，直接进入 loopEnd, 把分发流程做到 loopEnd 中  
，这里要做的是把 switch 生成到 loopEnd 中，然后把 loopEnd 跳转 loopEntry 的跳转指令删除掉，这样就可以实现 loopEnd 直接分发代码块

```
   LoadInst *load2 = new LoadInst(switchVar, "switchVar", loopEnd);
 BasicBlock *swDefault =
      BasicBlock::Create(f->getContext(), "switchDefault", f, loopEnd);
  BranchInst::Create(loopEnd, swDefault);
SwitchInst *switch2=SwitchInst::Create(load2, loopEnd2, 0, loopEnd);
  //修改查找Case的switch为loopEnd里的switch
  numCase = switch2->findCaseDest(succ)
 
  生成的LR
  loopEnd:                                          ; preds = %for.end, %for.body, %if.else5, %if.else, %first, %loopEnd2, %switchDefault
  %switchVar2 = load i32, i32* %switchVar
  switch i32 %switchVar2, label %loopEnd2 [
    i32 -256601636, label %first
    i32 -1060808858, label %if.else
    i32 1575411630, label %if.else5
    i32 1213754259, label %for.body
    i32 420055895, label %for.end
    i32 -1907967449, label %if.end9
  ]

```

二是添加多个 loopEnd, 这里要改的部分比较多，因为不只一个分发块的情况下每个分发块的平分了所有代码块，这里就要保证当一个代码块进入了这个分发块的时候如果 switchVar 的条件值可以找到对应的 Case，类似下图这样。  
![](https://bbs.kanxue.com/upload/attach/202406/978849_B55WK8ZV73RYEHN.webp)  
和只有一个分发块不同的就是要把 switch 的默认跳转设置成下一个 loopEnd 在最后一个 loopEnd 把默认跳转设置成第一个 loopEnd，这次魔改我用了两个 loopEnd，想多一点可以写成循环。

```
SwitchInst *switch2=SwitchInst::Create(load2, loopEnd2, 0, loopEnd);
SwitchInst *switch3=SwitchInst::Create(load3, loopEnd, 0, loopEnd2);

```

接下来是把所有代码块的跳转平分给两个 loopEnd,

```
int addCase_flag=1;
  size_t count1=0;
  for (vector::iterator b = origBB.begin(); b != origBB.end();
       ++b,count1++) {
    BasicBlock *i = *b;
    ConstantInt *numCase = NULL;
 
    // Move the BB inside the switch (only visual, no code logic)
    i->moveBefore(loopEnd);
        //change
      if(addCase_flag==1){
         // Add case to switch
    numCase = cast(ConstantInt::get(
        switchI->getCondition()->getType(),
        llvm::cryptoutils->scramble32(count1, scrambling_key)));
      switch2->addCase(numCase, i);
      addCase_flag=0;
      }
      else{
         // Add case to switch
    numCase = cast(ConstantInt::get(
        switchI->getCondition()->getType(),
        llvm::cryptoutils->scramble32(count1, scrambling_key)));
        switch3->addCase(numCase, i);
        addCase_flag=1;
      }
      //change
    // switch2->addCase(numCase, i);
  } 
```

以及最后去除原始代码块跳转添加判断跳转 Case 值的修改, 这里用了一个判断，来判断这个分发块中是否包含了 Case 条件值没有就查找下一个分发块。

```
size_t conut2=0;
int a=1;
for (vector::iterator b = origBB.begin(); b != origBB.end();
     ++b,conut2++) {
  BasicBlock *i = *b;
  ConstantInt *numCase = NULL;
 
  // Ret BB
  if (i->getTerminator()->getNumSuccessors() == 0) {
    continue;
  }
 
  // If it's a non-conditional jump
  if (i->getTerminator()->getNumSuccessors() == 1) {
    // Get successor and delete terminator
    BasicBlock *succ = i->getTerminator()->getSuccessor(0);
    i->getTerminator()->eraseFromParent();
 
    // Get next case
    if(switch2->findCaseDest(succ)!=nullptr)
    {numCase = switch2->findCaseDest(succ);}
    else
    {numCase = switch3->findCaseDest(succ);}
 
    // If next case == default case (switchDefault)
    if (numCase == NULL) {
      numCase = cast(
          ConstantInt::get(switchI->getCondition()->getType(),
                           llvm::cryptoutils->scramble32(
                               switchI->getNumCases() - 1, scrambling_key)));
    }
 
    // Update switchVar and jump to the end of loop
    new StoreInst(numCase, load->getPointerOperand(), i);
    
    if(a==1){
    BranchInst::Create(loopEnd, i);
    a=0;
    }
    else{
       BranchInst::Create(loopEnd2, i);
      a=1;
    }
    // BranchInst::Create(loopEnd, i);
    continue;
  }
 
  // If it's a conditional jump
  if (i->getTerminator()->getNumSuccessors() == 2) {
    // Get next cases
    ConstantInt *numCaseTrue =nullptr;
    ConstantInt *numCaseFalse =nullptr;
    if(switch2->findCaseDest(i->getTerminator()->getSuccessor(0))!=nullptr)
    {
          
       numCaseTrue = switch2->findCaseDest(i->getTerminator()->getSuccessor(0));
    }
    else
    {
      numCaseTrue =
        switch3->findCaseDest(i->getTerminator()->getSuccessor(0));
    }
 
    if(switch2->findCaseDest(i->getTerminator()->getSuccessor(1))!=nullptr)
    {
      numCaseFalse =
        switch2->findCaseDest(i->getTerminator()->getSuccessor(1));
    }
    else
    {
  numCaseFalse =
        switch3->findCaseDest(i->getTerminator()->getSuccessor(1));
    }
  
   
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
      if(a==1){
    BranchInst::Create(loopEnd, i);
    a=0;
    }
    else{
       BranchInst::Create(loopEnd2, i);
      a=1;
    }
    continue;
  }
} 
```

看下效果：  
![](https://bbs.kanxue.com/upload/attach/202406/978849_QBTUE53EWNFU4U6.webp)  
可以看到现在有两个 loopEnd 分发块且入度基本相同，并且由于是直接跳转到基本块，流程图里的两个 loopEnd 各会有一条不同的路径返回到不同的块，这样通过 LoopEntry 的前继也获取不到所有的 loopEnd 了。  
不过对于 OLLVM 来说控制流平坦化只是其中的一环，配合 OLLVM 中的其他模块或一些的新型混淆 Pass 才能发挥最大的作用。

参考文章
====

[https://xuanxuanblingbling.github.io/ctf/pwn/2019/12/21/llvm/](https://xuanxuanblingbling.github.io/ctf/pwn/2019/12/21/llvm/)  
[https://leadroyal.cn/p/1072/](https://leadroyal.cn/p/1072/)  
[http://www.qfrost.com/posts/llvm/llvmflattening%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#%E5%A4%8D%E6%9D%82%E5%88%86%E5%8F%91%E8%BF%87%E7%A8%8B](http://www.qfrost.com/posts/llvm/llvmflattening%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#%E5%A4%8D%E6%9D%82%E5%88%86%E5%8F%91%E8%BF%87%E7%A8%8B)  
[https://groups.google.com/g/llvm-dev/c/-ihkMNlDvEQ](https://groups.google.com/g/llvm-dev/c/-ihkMNlDvEQ)

[[培训] 科锐软件逆向 50 期预科班报名即将截止，速来！！！ 50 期正式班报名火爆招生中！！！](https://mp.weixin.qq.com/s/HFghXQRTiTlk6oRKGotpHA)

[#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)