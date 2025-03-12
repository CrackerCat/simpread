> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285968.htm)

> [原创] 从源码视角分析 Arkari 间接跳转混淆

前言
--

本文仅对 Arkari 用于混淆的 pass 的源码分析和讲解，没有提供去除混淆的方案，请需要去混淆的师傅自己想出解决方案，希望分析能够帮到各位师傅  
文章大部分对源码的解释都直接以注释的形式写在代码中

> PS: 本文所有分析均基于开源项目 Arkari：[Arkari](elink@4a5K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6w2L8$3#2A6e0h3!0W2i4K6u0r3b7i4u0C8j5i4u0A6) LLVM19.x 架构为 ARM64

Arkari 支持的混淆特性
--------------

*   混淆过程间相关
*   间接跳转, 并加密跳转目标 (`-mllvm -irobf-indbr`)
*   间接函数调用, 并加密目标函数地址 (`-mllvm -irobf-icall`)
*   间接全局变量引用, 并加密变量地址 (`-mllvm -irobf-indgv`)
*   字符串 (c string) 加密功能(`-mllvm -irobf-cse`)
*   过程相关控制流平坦混淆 (`-mllvm -irobf-cff`)
*   整数常量加密 (`-mllvm -irobf-cie`)
*   浮点常量加密 (`-mllvm -irobf-cfe`)

编译 Arkari 源码
------------

```
git clone https://github.com/KomiMoe/Arkari.git
cd Arkari
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm
make -j12
```

间接跳转介绍
------

在 ARM64 架构中，间接跳转混淆是一种通过**破坏静态控制流可读性**的代码保护技术，其核心机制是将程序中的**直接跳转（如`B`/`BL`指令）替换为通过通用寄存器（如 X17）间接跳转**的模式，常见形式为`BR X8`等。由于 IDA 等静态反编译工具无法确定寄存器中的值，导致程序原始控制流在静态分析下被截断，使得静态分析失效。

源码分析
----

间接跳转混淆的源码在编译好之后的`Arkari/llvm/lib/Transforms/Obfuscation`路径下的`IndirectBranch.cpp`文件

```
bool runOnFunction(Function &Fn) override {
 
    const auto opt = ArgsOptions->toObfuscate(ArgsOptions->indBrOpt(), &Fn);
 
    if (!opt.isEnabled()) {
      return false;
    }
 
    if (Fn.empty() || Fn.hasLinkOnceLinkage() || Fn.getSection() == ".text.startup") {
      return false;
    }
 
    LLVMContext &Ctx = Fn.getContext();
 
    // Init member fields
    BBNumbering.clear();
    BBTargets.clear();
 
    // llvm cannot split critical edge from IndirectBrInst
    SplitAllCriticalEdges(Fn, CriticalEdgeSplittingOptions(nullptr, nullptr));
    NumberBasicBlock(Fn);
 
    if (BBNumbering.empty()) {
      return false;
    }
 
    uint64_t V = RandomEngine.get_uint64_t();
    uint64_t XV = RandomEngine.get_uint64_t();
    IntegerType* intType = Type::getInt32Ty(Ctx);
    if (pointerSize == 8) {
      intType = Type::getInt64Ty(Ctx);
    }
    ConstantInt *EncKey = ConstantInt::get(intType, V, false);
    ConstantInt *EncKey1 = ConstantInt::get(intType, -V, false);
    ConstantInt *Zero = ConstantInt::get(intType, 0);
 
    GlobalVariable *GXorKey = nullptr;
    GlobalVariable *DestBBs = nullptr;
    GlobalVariable *XorKeys = nullptr;
 
    if (opt.level() == 0) {
      DestBBs = getIndirectTargets0(Fn, EncKey1);
    } else if (opt.level() == 1 || opt.level() == 2) {
      ConstantInt *CXK = ConstantInt::get(intType, XV, false);
      GXorKey = new GlobalVariable(*Fn.getParent(), CXK->getType(), false, GlobalValue::LinkageTypes::PrivateLinkage,
        CXK, Fn.getName() + "_IBrXorKey");
      appendToCompilerUsed(*Fn.getParent(), {GXorKey});
      if (opt.level() == 1) {
        DestBBs = getIndirectTargets1(Fn, EncKey1, CXK);
      } else {
        DestBBs = getIndirectTargets2(Fn, EncKey1, CXK);
      }
    } else {
      auto [fst, snd] = getIndirectTargets3(Fn, EncKey1);
      DestBBs = fst;
      XorKeys = snd;
    }
 
    for (auto &BB : Fn) {
      auto *BI = dyn_cast(BB.getTerminator());
      if (BI && BI->isConditional()) {
        IRBuilder<> IRB(BI);
 
        Value *Cond = BI->getCondition();
        Value *Idx;
        Value *TIdx, *FIdx;
 
        TIdx = ConstantInt::get(intType, BBNumbering[BI->getSuccessor(0)]);
        FIdx = ConstantInt::get(intType, BBNumbering[BI->getSuccessor(1)]);
        Idx = IRB.CreateSelect(Cond, TIdx, FIdx);
 
        Value *GEP = IRB.CreateGEP(
          DestBBs->getValueType(), DestBBs,
            {Zero, Idx});
        Value *EncDestAddr = IRB.CreateLoad(
            GEP->getType(),
            GEP,
            "EncDestAddr");
        // -EncKey = X - FuncSecret
        Value *DecKey = EncKey;
 
        if (GXorKey) {
          LoadInst *XorKey = IRB.CreateLoad(GXorKey->getValueType(), GXorKey);
 
          if (opt.level() == 1) {
            DecKey = IRB.CreateXor(EncKey1, XorKey);
            DecKey = IRB.CreateNeg(DecKey);
          } else if (opt.level() == 2) {
            DecKey = IRB.CreateXor(EncKey1, IRB.CreateMul(XorKey, Idx));
            DecKey = IRB.CreateNeg(DecKey);
          }
        }
 
        if (XorKeys) {
          Value *XorKeysGEP = IRB.CreateGEP(XorKeys->getValueType(), XorKeys, {Zero, Idx});
 
          Value *XorKey = IRB.CreateLoad(intType, XorKeysGEP);
 
          XorKey = IRB.CreateNeg(XorKey);
          XorKey = IRB.CreateXor(XorKey, EncKey1);
          XorKey = IRB.CreateNeg(XorKey);
 
          DecKey = IRB.CreateXor(EncKey1, IRB.CreateMul(XorKey, Idx));
          DecKey = IRB.CreateNeg(DecKey);
        }
 
        Value *DestAddr = IRB.CreateGEP(
          Type::getInt8Ty(Ctx),
            EncDestAddr, DecKey);
 
        IndirectBrInst *IBI = IndirectBrInst::Create(DestAddr, 2);
        IBI->addDestination(BI->getSuccessor(0));
        IBI->addDestination(BI->getSuccessor(1));
        ReplaceInstWithInst(BI, IBI);
      }
    }
    return true;
  } 
```

这是`IndirectBranch.cpp`的`runOnFunction`函数，在 OLLVM 中 runOnFunction 函数是 **pass** 执行的**入口点**

### Step 1: 随机打乱基本块

在函数中函数首先初始化了两个数组：

*   `BBNumbering` : 存储基本块到随机化编号的映射，用于后续计算跳转索引
*   `BBTargets` : 临时存储所有唯一条件分支目标块
    
    > 这里需要提一下，OLLVM 中一个函数对象 (Function) 是由基本块 (BasicBlock) 组成，而基本块由每一条指令 (Instruction) 组成
    

接着调用`SplitAllCriticalEdges`函数，分割关键边，原理是在有多个前驱和后继的边之间**插入新基本块**

再调用`NumberBasicBlock`函数，这个函数将所有**条件分支目标基本块**分配随机化编号，打乱程序原有的顺序执行流程

```
void NumberBasicBlock(Function &F) {
    for (auto &BB : F) {  // 遍历所有基本块
      if (auto *BI = dyn_cast(BB.getTerminator())) {  // 获取基本块的最后一条跳转指令
        if (BI->isConditional()) {    // 判断是否是条件跳转
          unsigned N = BI->getNumSuccessors();   // N为分支数
          for (unsigned I = 0; I < N; I++) {
            BasicBlock *Succ = BI->getSuccessor(I);   // 获取后继基本块
            if (BBNumbering.count(Succ) == 0) {    // 检查Succ是否已存在于map中
              BBTargets.push_back(Succ);  
              BBNumbering[Succ] = 0;       // 添加到BBTargets并初始化编号为0
            }
          }
        }
      }
    }
 
    long seed = RandomEngine.get_uint32_t();      // 随机数种子
    std::default_random_engine e(seed);
    std::shuffle(BBTargets.begin(), BBTargets.end(), e);      // 用随机数打乱
 
    unsigned N = 0;
    for (auto BB:BBTargets) {
      BBNumbering[BB] = N++;   // 打乱的基本块重新编号
    }
} 
```

代码的注释中已经写的很详细，总结一下就是：**寻找条件跳转的后继基本块 -> 随机数打乱基本块 -> 基本块重编号**

### Step 2: 初始化加密密钥

#### **1. 密钥生成**

```
// 生成 64 位随机密钥
uint64_t V = RandomEngine.get_uint64_t();
uint64_t XV = RandomEngine.get_uint64_t();
 
// 根据指针大小选择整数类型
IntegerType* intType = Type::getInt32Ty(Ctx);
if (pointerSize == 8) {
  intType = Type::getInt64Ty(Ctx);
}
 
// 构造加密常数（核心密钥）
ConstantInt *EncKey = ConstantInt::get(intType, V, false);      // 加密密钥
ConstantInt *EncKey1 = ConstantInt::get(intType, -V, false);   // 解密密钥
ConstantInt *Zero = ConstantInt::get(intType, 0);              // 零值常量
```

#### **2. 全局变量声明**

```
GlobalVariable *GXorKey = nullptr;   // 一级异或密钥存储
GlobalVariable *DestBBs = nullptr;   // 加密跳转表
GlobalVariable *XorKeys = nullptr;   // 动态异或密钥池
```

<table><thead><tr><th>变量名</th><th>类型</th><th>作用</th></tr></thead><tbody><tr><td><code>GXorKey</code></td><td><code>GlobalVariable*</code></td><td>存储全局异或密钥（用于后续 Level 1 or 2 混淆）</td></tr><tr><td><code>DestBBs</code></td><td><code>GlobalVariable*</code></td><td>存储加密后的跳转地址数组（核心跳转表）</td></tr><tr><td><code>XorKeys</code></td><td><code>GlobalVariable*</code></td><td>存储动态生成的异或密钥数组（后续 Level 3 使用）</td></tr></tbody></table>

### step 3: 根据 opt 选择混淆强度

这里引用作者的话来解释：

> 可以使用下列几种方法之一单独控制某个混淆 Pass 的强度  
> (Win64-19.1.0-rc3-obf1.5.1-rc5 or later)
> 
> 如果不指定强度则默认强度为 0，annotate 的优先级永远高于命令行参数
> 
> 可用的 Pass:
> 
> *   `icall` (强度范围: 0-3)
> *   `indbr` (强度范围: 0-3)
> *   `indgv` (强度范围: 0-3)
> *   `cie` (强度范围: 0-3)
> *   `cfe` (强度范围: 0-3)
> 
> 1. 通过 **annotate** 对特定函数指定混淆强度：
> 
> `^flag=1` 表示当前函数设置某功能强度等级 (此处为 1)
> 
> ```
> //^icall=表示指定icall的强度
> //+icall表示当前函数启用icall混淆, 如果你在命令行中启用了icall则无需添加+icall
>  
> [[clang::annotate("+icall ^icall=3")]]
> int main() {
>     std::cout << "HelloWorld" << std::endl;
>     return 0;
> }
> ```
> 
> 2. 通过命令行参数指定特定混淆 Pass 的强度
> 
> Eg. 间接函数调用, 并加密目标函数地址, 强度设置为 3(`-mllvm -irobf-icall -mllvm -level-icall=3`)

#### **控制逻辑**

```
if (opt.level() == 0) {
  DestBBs = getIndirectTargets0(Fn, EncKey1);        //level 0 的处理逻辑
} else if (opt.level() == 1 || opt.level() == 2) {                //level 1 和 2 共同的逻辑
  ConstantInt *CXK = ConstantInt::get(intType, XV, false);
  GXorKey = new GlobalVariable(                        //生成用于XOR的key的全局变量
      *Fn.getParent(),
      CXK->getType(),
      false,
      GlobalValue::LinkageTypes::PrivateLinkage,
      CXK,
      Fn.getName() + "_IBrXorKey"
    );
  appendToCompilerUsed(*Fn.getParent(), {GXorKey});
  if (opt.level() == 1) {
    DestBBs = getIndirectTargets1(Fn, EncKey1, CXK);   //level 1 方案
  } else {
    DestBBs = getIndirectTargets2(Fn, EncKey1, CXK);   //level 2 方案
  }
} else {
  auto [fst, snd] = getIndirectTargets3(Fn, EncKey1);  //level 3 方案
  DestBBs = fst;
  XorKeys = snd;
}
```

#### **Level 0**

```
GlobalVariable *getIndirectTargets0(Function &F, ConstantInt *EncKey) const {
  std::string GVName(F.getName().str() + "_IndirectBrTargets");
  GlobalVariable *GV = F.getParent()->getNamedGlobal(GVName);    //获取对应函数的加密地址的全局变量
  if (GV)
    return GV;   //如果已经生成过全局变量，则直接返回，防止重复生成
 
  std::vector Elements;  //存储加密后的基本块地址
 
  //遍历需要加密的基本块
  for (const auto BB:BBTargets) {
    // 获取基本块地址并转换为通用指针类型
    Constant *CE = ConstantExpr::getBitCast(
          BlockAddress::get(BB),
          PointerType::getUnqual(F.getContext())
      );
    // 加密基本块地址
    CE = ConstantExpr::getGetElementPtr(
        Type::getInt8Ty(F.getContext()),
        CE,       //原始地址
        EncKey    //加密偏移
    );
    Elements.push_back(CE);
  }
 
  // 定义数组类型
  ArrayType *ATy = ArrayType::get(
    PointerType::getUnqual(F.getContext()),
    Elements.size()                         // 数组长度
  );
  // 将加密地址封装为常量数组
  Constant *CA = ConstantArray::get(ATy, Elements);
  // 创建全局变量
  GV = new GlobalVariable(
    *F.getParent(),                 // 所属模块
    ATy,                            // 数组类型
    false,                          // 是否常量
    GlobalValue::LinkageTypes::PrivateLinkage, // 链接属性
    CA,                             // 初始值
    GVName                          // 变量名
  );
  // 防止优化器删除该全局变量
  appendToCompilerUsed(*F.getParent(), {GV});
 
  return GV;
} 
```

总结下来`level 0`的处理就是生成一个**加密地址的全局变量数组 (GV)**, 可以用一个公式来概括：`加密地址 = 原始地址 + EncKey`  
需要注意的是，EncKey 是负数，所以代码上看起来是`加密地址 = 原始地址 - EncKey`

#### **Level 1**

```
GlobalVariable *getIndirectTargets1(Function &F, ConstantInt *AddKey, ConstantInt *XorKey) const {
  std::string GVName(F.getName().str() + "_IndirectBrTargets1");
  GlobalVariable *GV = F.getParent()->getNamedGlobal(GVName);
  if (GV)
    return GV;
 
  // encrypt branch targets
  std::vector Elements;
  for (const auto BB:BBTargets) {
    Constant *CE = ConstantExpr::getBitCast(
        BlockAddress::get(BB),
        PointerType::getUnqual(F.getContext())
      );
    CE = ConstantExpr::getGetElementPtr(
        Type::getInt8Ty(F.getContext()),
        CE,
        ConstantExpr::getXor(AddKey, XorKey)     //不同点
    );
    Elements.push_back(CE);
  }
 
  ArrayType *ATy =ArrayType::get(
        PointerType::getUnqual(F.getContext()),
        Elements.size()
    );
  Constant *CA = ConstantArray::get(ATy, ArrayRef(Elements));
  GV = new GlobalVariable(
      *F.getParent(),
      ATy,
      false,
      GlobalValue::LinkageTypes::PrivateLinkage,
      CA,
      GVName
  );
  appendToCompilerUsed(*F.getParent(), {GV});
  return GV;
} 
```

`level 1`主要的修改点就是`ConstantExpr::getXor(AddKey, XorKey)`这里，换成公式就是`加密地址 = 原始地址 + (AddKey ^ XorKey)`

#### **Level 2**

```
GlobalVariable *getIndirectTargets2(Function &F, ConstantInt *AddKey, ConstantInt *XorKey) {
  std::string GVName(F.getName().str() + "_IndirectBrTargets2");
  GlobalVariable *GV = F.getParent()->getNamedGlobal(GVName);
  if (GV)
    return GV;
 
  auto& Ctx = F.getContext();
  IntegerType *intType = Type::getInt32Ty(Ctx);
  if (pointerSize == 8) {
    intType = Type::getInt64Ty(Ctx);
  }
  // encrypt branch targets
  std::vector Elements;
  for (auto BB:BBTargets) {
    Constant *CE = ConstantExpr::getBitCast(
        BlockAddress::get(BB),
        PointerType::getUnqual(F.getContext())
    );
    CE = ConstantExpr::getGetElementPtr(
        Type::getInt8Ty(F.getContext()),
        CE,
        ConstantExpr::getXor(            //主要修改点
            AddKey,
            ConstantExpr::getMul(
                XorKey,
                ConstantInt::get(intType, BBNumbering[BB], false)
              )
          )       
      );           
    Elements.push_back(CE);
  }
 
  ArrayType *ATy =ArrayType::get(PointerType::getUnqual(F.getContext()), Elements.size());
  Constant *CA = ConstantArray::get(ATy, ArrayRef(Elements));
  GV = new GlobalVariable(*F.getParent(), ATy, false, GlobalValue::LinkageTypes::PrivateLinkage,
    CA, GVName);
  appendToCompilerUsed(*F.getParent(), {GV});
  return GV;
} 
```

同`level 1`, 修改点在代码中已标出，公式:`加密地址 = 原始地址 + [AddKey ^ (XorKey * Idx)]` 稍微解释一下，level 0 和 level 1 的 key 都是**固定**的，而 level 2 的 key 和**基本块的编号挂钩**，所以每个基本块对应的解密 key 都不一样

#### **Level 3**

```
std::pair getIndirectTargets3(Function &F, ConstantInt *AddKey) {
  std::string GVNameAdd(F.getName().str() + "_IndirectBrTargets3");
  std::string GVNameXor(F.getName().str() + "_IndirectBr3_Xor");
  GlobalVariable *GVAdd = F.getParent()->getNamedGlobal(GVNameAdd);
  GlobalVariable *GVXor = F.getParent()->getNamedGlobal(GVNameXor);   //新的XorKey数组全局变量
 
  if (GVAdd && GVXor)
    return std::make_pair(GVAdd, GVXor);
 
  auto& Ctx = F.getContext();
  IntegerType *intType = Type::getInt32Ty(Ctx);
  if (pointerSize == 8) {
    intType = Type::getInt64Ty(Ctx);
  }
 
  // encrypt branch targets
  std::vector Elements;
  std::vector XorKeys;
  for (auto BB:BBTargets) {
    uint64_t V = RandomEngine.get_uint64_t();
    Constant *XorKey = ConstantInt::get(intType, V, false);
    Constant *CE = ConstantExpr::getBitCast(
        BlockAddress::get(BB),
        PointerType::getUnqual(F.getContext())
    );
    // 加密地址计算
    // 与level 2相似，但是采用动态密钥
    CE = ConstantExpr::getGetElementPtr(
        Type::getInt8Ty(F.getContext()),
        CE,
        ConstantExpr::getXor(
            AddKey,
            ConstantExpr::getMul(
                XorKey,
                ConstantInt::get(intType, BBNumbering[BB], false)
              )
           )
      );
    //特别之处：使用密钥对原基本块地址加密之后，通过取反和异或等运算混淆密钥，使得原密钥更难被计算
    XorKey = ConstantExpr::getNeg(XorKey);            //XorKey = ~Xorkey
    XorKey = ConstantExpr::getXor(XorKey, AddKey);    //XorKey ^= Addkey
    XorKey = ConstantExpr::getNeg(XorKey);            //Xorkey = ~XorKey
    XorKeys.push_back(XorKey);
    Elements.push_back(CE);
  }
 
  ArrayType *ATy =ArrayType::get(
        PointerType::getUnqual(F.getContext()),
        Elements.size()
    );
  Constant *CA = ConstantArray::get(
      ATy,
      ArrayRef(Elements)
  );
  GVAdd = new GlobalVariable(
      *F.getParent(),
      ATy,
      false,
      GlobalValue::LinkageTypes::PrivateLinkage,
      CA,
      GVNameAdd
  );
  appendToCompilerUsed(*F.getParent(), {GVAdd});
 
  ArrayType *XTy = ArrayType::get(intType, XorKeys.size());
  Constant *CX = ConstantArray::get(XTy, XorKeys);
  GVXor = new GlobalVariable(   //把混淆后的密钥放到新的全局变量中
      *F.getParent(),
      XTy,
      false,
      GlobalValue::LinkageTypes::PrivateLinkage,
      CX,
      GVNameXor
  );
  appendToCompilerUsed(*F.getParent(), {GVXor});
 
  return std::make_pair(GVAdd, GVXor);
} 
```

`level 3` 的加密公式可以这么表示: `加密地址 = 原始地址 + [EncKey1 ^ ( (XorKeys[Idx] ^ EncKey1) * Idx )]`  
`level 3`在`level 2`的基础上增加了对**密钥的混淆**。**完全随机化的动态密钥** + **双数组隔离** + **混淆**的设计，使得原来硬编码的密钥更加安全，在后续解密基本块地址的时候再对混淆的密钥进行还原，**这也是 Arkari 对间接跳转混淆的创新点所在**

### step 4: 解密真实块地址并插入间接跳转指令

```
for (auto &BB : Fn) {
      auto *BI = dyn_cast(BB.getTerminator());
      if (BI && BI->isConditional()) {            //判断基本块结尾是否是条件跳转指令
        IRBuilder<> IRB(BI);                      //获取IR代码的构造器
 
        Value *Cond = BI->getCondition();         //获取跳转的条件
        Value *Idx;
        Value *TIdx, *FIdx;
 
        TIdx = ConstantInt::get(intType, BBNumbering[BI->getSuccessor(0)]);    //获取使得条件为真的后继块的编号
        FIdx = ConstantInt::get(intType, BBNumbering[BI->getSuccessor(1)]);    //获取使得条件为假的后继块的编号
        Idx = IRB.CreateSelect(Cond, TIdx, FIdx);         //构造CSEL指令
 
        Value *GEP = IRB.CreateGEP(DestBBs->getValueType(), DestBBs,{Zero, Idx});//计算数组元素地址(DestBBs(Idx))
        Value *EncDestAddr = IRB.CreateLoad(     //取出加密后的目标地址
            GEP->getType(),
            GEP,
            "EncDestAddr");  
        // -EncKey = X - FuncSecret
        Value *DecKey = EncKey;
 
        if (GXorKey) {
          LoadInst *XorKey = IRB.CreateLoad(GXorKey->getValueType(), GXorKey);
 
          if (opt.level() == 1) {         //生成level 1对应的解密密钥
            DecKey = IRB.CreateXor(EncKey1, XorKey);
            DecKey = IRB.CreateNeg(DecKey);
          } else if (opt.level() == 2) {  //生成level 2对应的解密密钥
            DecKey = IRB.CreateXor(EncKey1, IRB.CreateMul(XorKey, Idx));
            DecKey = IRB.CreateNeg(DecKey);
          }
        }
 
        if (XorKeys) {   //生成level 3对应的解密密钥
          Value *XorKeysGEP = IRB.CreateGEP(XorKeys->getValueType(), XorKeys, {Zero, Idx});
 
          Value *XorKey = IRB.CreateLoad(intType, XorKeysGEP);
 
          XorKey = IRB.CreateNeg(XorKey);
          XorKey = IRB.CreateXor(XorKey, EncKey1);
          XorKey = IRB.CreateNeg(XorKey);
 
          DecKey = IRB.CreateXor(EncKey1, IRB.CreateMul(XorKey, Idx));
          DecKey = IRB.CreateNeg(DecKey);
        }
 
        Value *DestAddr = IRB.CreateGEP(
          Type::getInt8Ty(Ctx),
            EncDestAddr, DecKey);       //计算目标基本块的地址
 
        IndirectBrInst *IBI = IndirectBrInst::Create(DestAddr, 2);   //生成BR指令
        IBI->addDestination(BI->getSuccessor(0));    //预先添加可能跳转的所有目标
        IBI->addDestination(BI->getSuccessor(1));    //这里即当前块的两个后继(因为是条件跳转)
        ReplaceInstWithInst(BI, IBI);
      }
    } 
```

大概的过程就是如下所示 (AI 生成)：  
![](https://bbs.kanxue.com/upload/attach/202503/1009684_BEYDMUB8FXYKTUU.png)

总结
--

Hakari 的间接跳转混淆增加了不同的混淆强度，尤其是 Level 3 的混淆，对密钥做了进一步混淆，使得通过原始计算密钥来恢复函数的控制流变得更加困难，如果想更加完美的解决混淆，用 Angr 强制程序走不同分支也许会更加合适

[[招生] 科锐逆向工程师培训 (2025 年 3 月 11 日实地，远程教学同时开班, 第 52 期)！](https://bbs.kanxue.com/thread-51839.htm)

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm)