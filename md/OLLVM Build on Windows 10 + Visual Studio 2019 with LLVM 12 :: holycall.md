> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [holycall.tistory.com](https://holycall.tistory.com/364)

> Copy OLLVM source code to LLVM 12 Get LLVM 12 source code from https://github.com/llvm/llvm-project .......

Copy OLLVM source code to LLVM 12
---------------------------------

Get LLVM 12 source code from [https://github.com/llvm/llvm-project](https://github.com/llvm/llvm-project) .

Get OLLVM source code built with LLVM 10 from [https://github.com/isrc-cas/flounder](https://github.com/isrc-cas/flounder) .

Copy OLLVM source code to LLVM 12 code.

```
copy /r C:\Dev\ollvm\reference\flounder\llvm\include\llvm\Transforms\Obfuscation C:\Dev\ollvm\llvm-project\llvm\include\llvm\Transforms
copy /r C:\Dev\ollvm\reference\flounder\llvm\lib\Transforms\Obfuscation C:\Dev\ollvm\llvm-project\llvm\lib\Transforms

```

Modify CmakeLists.txt
---------------------

Add the following at line 13 of llvm-project\llvm\lib\Transforms\CMakeLists.txt

```
add_subdirectory(Obfuscation)

```

llvm-project\llvm\lib\Transforms\IPO\CMakeLists.txt

Add the following line at line 73

```
Obfuscation

```

Modify source code. 
--------------------

### PassManagerBuilder.cpp

llvm-project\llvm\lib\Transforms\IPO\PassManagerBuilder.cpp

Add the following at line 51

```
#include "llvm/Transforms/Obfuscation/BogusControlFlow.h" 
#include "llvm/Transforms/Obfuscation/Flattening.h" 
#include "llvm/Transforms/Obfuscation/Split.h" 
#include "llvm/Transforms/Obfuscation/Substitution.h" 
#include "llvm/Transforms/Obfuscation/CryptoUtils.h" 
#include "llvm/Transforms/Obfuscation/StringObfuscation.h"

```

Add the following at line 93

```
// Flags for obfuscation 
static cl::opt<std::string> Seed("seed", cl::init(""), 
                                    cl::desc("seed for the random")); 
static cl::opt<std::string> AesSeed("aesSeed", cl::init(""), 
                                    cl::desc("seed for the AES-CTR PRNG")); 
static cl::opt<bool> StringObf("sobf", cl::init(false), 
                                  cl::desc("Enable the string obfuscation"));   //tofix 
static cl::opt<bool> Flattening("fla", cl::init(false),              //tofix 
                                cl::desc("Enable the flattening pass")); 
static cl::opt<bool> BogusControlFlow("bcf", cl::init(false), 
                                      cl::desc("Enable bogus control flow")); 
static cl::opt<bool> Substitution("sub", cl::init(false), 
                                  cl::desc("Enable instruction substitutions")); 
static cl::opt<bool> Split("split", cl::init(false), 
                           cl::desc("Enable basic block splitting")); 
// Flags for obfuscation

```

Add the following at line 567

```
  //obfuscation related pass 
  MPM.add(createSplitBasicBlockPass(Split)); 
  MPM.add(createBogusPass(BogusControlFlow)); 
  MPM.add(createFlatteningPass(Flattening)); 
  MPM.add(createStringObfuscationPass(StringObf)); 
  MPM.add(createSubstitutionPass(Substitution));

```

### StringObfuscation.cpp

llvm-project\llvm\lib\Transforms\Obfuscation\StringObfuscation.cpp

Modify CallSite.h to AbstractCallSite.h. at line 10

```
#include "llvm/IR/AbstractCallSite.h"

```

Insert the follwing at line 15

```
#include "llvm/IR/Instructions.h"

```

Replace MayAlign to Align at line 15, 175, 183, 184, 191.

```
ptr_19->setAlignment(Align(8));
...
LoadInst* int8_20 = new LoadInst(ptr_arrayidx->getType()->getArrayElementType(), ptr_arrayidx, "", false, label_for_body);
int8_20->setAlignment(Align(1));
...
void_21->setAlignment(Align(1));

```

### Substitution.cpp

Modify Substitution::addDoubleNeg function at line 215. BinaryOperator → UnaryOperator

```
// Implementation of a = -(-b + (-c)) 
void Substitution::addDoubleNeg(BinaryOperator *bo) { 
  BinaryOperator *op, *op2 = NULL; 
  UnaryOperator *op3, *op4; 
  if (bo->getOpcode() == Instruction::Add) { 
    op = BinaryOperator::CreateNeg(bo->getOperand(0), "", bo); 
    op2 = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo); 
    op = BinaryOperator::Create(Instruction::Add, op, op2, "", bo); 
    op = BinaryOperator::CreateNeg(op, "", bo); 
    bo->replaceAllUsesWith(op); 
    // Check signed wrap 
    //op->setHasNoSignedWrap(bo->hasNoSignedWrap()); 
    //op->setHasNoUnsignedWrap(bo->hasNoUnsignedWrap()); 
  } else { 
    op3 = UnaryOperator::CreateFNeg(bo->getOperand(0), "", bo); 
    op4 = UnaryOperator::CreateFNeg(bo->getOperand(1), "", bo); 
    op = BinaryOperator::Create(Instruction::FAdd, op3, op4, "", bo); 
    op3 = UnaryOperator::CreateFNeg(op, "", bo); 
    bo->replaceAllUsesWith(op3); 
  }   
}

```

Modify Substitution::subNeg function at line 299. BinaryOperator → UnaryOperator

```
// Implementation of a = b + (-c) 
void Substitution::subNeg(BinaryOperator *bo) { 
  BinaryOperator *op = NULL;   
  if (bo->getOpcode() == Instruction::Sub) { 
    op = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo); 
    op = BinaryOperator::Create(Instruction::Add, bo->getOperand(0), op, "", bo); 
    // Check signed wrap 
    //op->setHasNoSignedWrap(bo->hasNoSignedWrap()); 
    //op->setHasNoUnsignedWrap(bo->hasNoUnsignedWrap()); 
  } else { 
    auto op1 = UnaryOperator::CreateFNeg(bo->getOperand(1), "", bo); 
    op = BinaryOperator::Create(Instruction::FAdd, bo->getOperand(0), op1, "", bo); 
  } 
  bo->replaceAllUsesWith(op); 
}

```

### BogusControlFlow.cpp

C:\Dev\ollvm\llvm-project\llvm\lib\Transforms\Obfuscation\BogusControlFlow.cpp 

Add the following at line 380

```
UnaryOperator *op2;

```

Modify at line 420

case 1: op2 = UnaryOperator::CreateFNeg(i->getOperand(0),*var,&*i);

```
case 1: op2 = UnaryOperator::CreateFNeg(i->getOperand(0),*var,&*i);

```

Modify at line 569-570

```
opX = new LoadInst (x->getType()->getElementType(), (Value *)x, "", (*i));
opY = new LoadInst (x->getType()->getElementType(), (Value *)y, "", (*i));


```

The constructor LoadInst in LLVM 12 requires a llvm::Type object as the first argument. 

### InitializePasses.h

C:\Dev\ollvm\llvm-project\llvm\include\llvm\InitializePasses.h 

Add the following at 453

```
void initializeFlatteningPass(PassRegistry&);

```

### Modify Flattening.cpp

C:\Dev\ollvm\llvm-project\llvm\lib\Transforms\Obfuscation\Flattening.cpp 수정

Add the following at line 17

```
#include "llvm/InitializePasses.h"

```

Modify line 123. 

```
load = new LoadInst(switchVar->getType()->getElementType(), switchVar, "switchVar", loopEntry);

```

cmake
-----

```
cd llvm-project
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS="clang;libcxx" -G "Visual Studio 16 2019" -A x64 -Thost=x64 ..\llvm

```

Build
-----

Open LLVM.sln and build clang.