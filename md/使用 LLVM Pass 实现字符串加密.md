> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/104735336)

这个项目可以作为我看了这么久 LLVM 的 Docs 的一个小总结吧。这个项目主要就是给函数使用的常量字符串进行加密，在程序被静态分析的时候干扰分析。当然找到思路后，这个混淆还是很容易解开的。

代码已上传到 Github [chenx6/baby_obfuscator](https://link.zhihu.com/?target=https%3A//github.com/chenx6/baby_obfuscator)

> 吐槽下，网上的文章质量参差不齐，写的让人不知所云的，真的恐怖。所以还是面向 Stackoverflow 和官方文档编程吧...

为了写 LLVM Pass ，首先得看看 [LLVM Programmers' Manual](https://link.zhihu.com/?target=https%3A//llvm.org/docs/ProgrammersManual.html)，里面讲了许多代码样例，API 讲解和类的层次结构。当然这只是基础，具体的使用得看使用 doxygen 生成的文档。

当然也得对 LLVM IR 也得有一定了解 [https://llvm.org/docs/LangRef.html](https://link.zhihu.com/?target=https%3A//llvm.org/docs/LangRef.html)

Pass 思路概述
---------

首先是找到字符串，其次是找到用这个字符串的函数，在这个函数被调用前，先调用解密函数进行解密。最后加密原本字符串。

开发环境准备
------

编译环境请参考[上一篇文章](https://zhuanlan.zhihu.com/p/104295455)，只是写个 Pass 而已，不要再从整个 LLVM 项目开始编译了好吗...

这里使用的是 VSCode + WSL 搭建开发环境，在项目文件夹的 ".vscode/c_cpp_properties.json" 加上对应的 "includePath" 就有智能提示了。

```
{
    "includePath": [
        "${workspaceFolder}/**",
        "/usr/include/llvm-8",
        "/usr/include/llvm-c-8"
    ]
}

```

由于使用 CMake 来管理编译依赖，所以给 VSCode 加上 CMake 插件，可以实现小部分 CMake GUI 的功能。

代码讲解
----

### 基本框架

首先是 include 的头文件，项目里用了 LLVM 自己的 `SmallVector`，还有 IR 下面的 `Function`, `GlobalVariable`, `IRBuilder`, `Instructions`，还有些 Pass 必备的一些头文件，`raw_ostream` 用来调试输出。

```
#include "llvm/ADT/SmallVector.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/GlobalVariable.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/InstrTypes.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Pass.h"
#include "llvm/Support/FormatVariadic.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
#include <map>
#include <vector>


```

然后是 LLVM Pass 的基本框架。这里实现的是 Module Pass，所以这个 Pass 继承于 `ModulePass`。最后两行代码是用于注册 Pass，注册之后在 `opt -load libpass.so -help` 就能找到这个 Pass 了。

加密函数只用了简单的 xor 来进行加密。需要注意的是不要把字符串的最后一位 '\0' 给 xor 了。

```
using namespace llvm;

namespace {
/// encrypt strings with xor.
/// \param s input string for encrypt.
void encrypt(std::string &s) {
  for (int i = 0; i < s.length() - 1; i++) {
    s[i] ^= 42;
  }
}

/// A pass for obfuscating const string in modules.
struct ObfuscatePass : public ModulePass {
  static char ID;
  ObfuscatePass() : ModulePass(ID) {}

  virtual bool runOnModule(Module &M) {
    return true;
  }
};
} // namespace

char ObfuscatePass::ID = 0;
static RegisterPass<ObfuscatePass> X("obfstr", "obfuscate string");


```

### `runOnModule` 函数

然后开始讲解具体的代码。下面的代码是在寻找全局变量，并且找到他的使用者。

首先通过 `GlobalVariable GVar->users()` 来找到使用者。如果使用者不是单独的指令，而是类似 `i8* getelementptr ...` 这样的语句，则寻找这个语句的使用者。如果发现这个语句只有 `CallInst` 类型的使用者，且只有一个函数使用了这个常量，则对字符串进行加密，并去除对应字符串的常量属性。

```
virtual bool runOnModule(Module &M) {
    for (GlobalValue &GV : M.globals()) {
      GlobalVariable *GVar = dyn_cast<GlobalVariable>(&GV);
      if (GVar == nullptr) {
        continue;
      }
      std::vector<std::pair<Instruction *, User *>> Target;
      bool hasExceptCallInst = false;
      // Find all user and encrypt const value, then insert decrypt function.
      for (User *Usr : GVar->users()) {
        // Get instruction.
        Instruction *Inst = dyn_cast<Instruction>(Usr);
        if (Inst == nullptr) {
          // If Usr is not an instruction, like i8* getelementptr...
          // Dig deeper to find Instruction.
          for (User *DirecUsr : Usr->users()) {
            Inst = dyn_cast<Instruction>(DirecUsr);
            if (Inst == nullptr) {
              continue;
            }
            if (!isa<CallInst>(Inst)) {
              hasExceptCallInst = true;
              Target.clear();
            } else {
              Target.emplace_back(std::pair<Instruction *, User *>(Inst, Usr));
            }
          }
        }
      }
      if (hasExceptCallInst == false && Target.size() == 1) {
        for (auto &T: Target) {
          obfuscateString(M, T.first, T.second, GVar);
        }
        // Change constant to variable.
        GVar->setConstant(false);
      }
    }
    return true;
  }


```

### `obfuscateString` 函数

函数中主要分为下面三个部分

1.  替换字符串为加密字符串
2.  创建函数声明
3.  创建调用指令

下面是加密字符串常量的语句，这里先将常量转换成 `ConstantDataArray`，用 `getAsString()` 提取字符串，最后用加密后的字符串创建 `Constant` 替换原来字符串。

```
// Encrypt origin string and replace it encrypted string.
ConstantDataArray *GVarArr =
    dyn_cast<ConstantDataArray>(GVar->getInitializer());
if (GVarArr == nullptr) {
    continue;
}
std::string Origin;
if (GVarArr->isString(8)) {
    Origin = GVarArr->getAsString().str();
} else if (GVarArr->isCString()) {
    Origin = GVarArr->getAsCString().str();
}
encrypt(Origin);
Constant *NewConstStr = ConstantDataArray::getString(
    GVarArr->getContext(), StringRef(Origin), false);
GVarArr->replaceAllUsesWith(NewConstStr);


```

然后是使用 `IRBuilder` 创建相关语句和函数调用。首先通过 `getOrInsertFunction` 插入或获取函数，然后使用 `builder.CreateCall();` 语句创建调用解密和加密函数的指令。

由于调用顺序是 解密字符串函数 -> 使用字符串函数 -> 加密字符串函数，加密函数得插入使用指令的后方，而不是和 `IRBuilder` 创建的指令一样待在使用指令的前方。所以这里不使用 `IRBuilder` 创建指令，而是手动创建指令 + 手动插入。

```
// Insert decrypt function above Inst with IRBuilder.
IRBuilder<> builder(Inst);
Type *Int8PtrTy = builder.getInt8PtrTy();
// Create decrypt function in GlobalValue / Get decrypt function.
SmallVector<Type *, 1> FuncArgs = {Int8PtrTy};
FunctionType *FuncType = FunctionType::get(Int8PtrTy, FuncArgs, false);
Constant *DecryptFunc = M.getOrInsertFunction("__decrypt", FuncType);
Constant *EncryptFunc = M.getOrInsertFunction("__encrypt", FuncType);
// Create call instrucions.
SmallVector<Value *, 1> CallArgs = {Usr};
CallInst *DecryptInst = builder.CreateCall(FuncType, DecryptFunc, CallArgs);
CallInst *EncryptInst = CallInst::Create(FuncType, EncryptFunc, CallArgs);
EncryptInst->insertAfter(Inst);


```

### `__encrypt` 和 `__decrypt` 函数

由于只是 xor，所以函数的代码很相似。

```
char *__decrypt(char *encStr) {
  char *curr = encStr;
  while (*curr) {
    *curr ^= 42;
    curr++;
  }
  return encStr;
}

char *__encrypt(char *originStr) {
  char *curr = originStr;
  while (*curr) {
    *curr ^= 42;
    curr++;
  }
  return originStr;
}

```

Pass 的简单使用
----------

首先将需要加密的文件和含有加密函数的文件编译成后缀为 `.ll` 的 LLVM IR 形式，然后使用 `opt` 来对需要加密的文件进行加密，最后则是将含有加密函数的文件和需要加密的文件进行链接，生成二进制文件。

将下面的脚本保存为 "run.sh" 后，就可以使用 `bash run.sh test.c` 来编译程序了。

```
fullname=${1}
basename=${fullname/.c/}
clang-8 -emit-llvm -S ${fullname} -o ${basename}.ll
clang-8 -emit-llvm -S encrypt.c -o encrypt.ll
opt-8 -p \
    -load ../build/obfuscate/libobfuscate.so \
    -obfstr ${basename}.ll \
    -o ${basename}_out.bc
llvm-link-8 encrypt.ll ${basename}_out.bc -o final.bc
clang-8 final.bc -o final.out

```

混淆效果
----

下面是测试程序的代码。

```
#include <stdio.h>

int main() {
  puts("This is a testing string!");
  char ch;
  if ((ch = getchar()) == '6') {
    printf("6666%c\n", ch);
  } else {
    printf("WTF?!\n");
  }
  return 0;
}

```

下面是使用 `clang-8 -emit-llvm -S test.c -o test.ll` 直接编译出的 IR。

```
@.str = private unnamed_addr constant [26 x i8] c"This is a testing string!\00", align 1
@.str.1 = private unnamed_addr constant [8 x i8] c"6666%c\0A\00", align 1
@.str.2 = private unnamed_addr constant [7 x i8] c"WTF?!\0A\00", align 1

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i8, align 1
  store i32 0, i32* %1, align 4
  %3 = call i32 @puts(i8* getelementptr inbounds ([26 x i8], [26 x i8]* @.str, i32 0, i32 0))
  %4 = call i32 @getchar()
  %5 = trunc i32 %4 to i8
  store i8 %5, i8* %2, align 1
  %6 = sext i8 %5 to i32
  %7 = icmp eq i32 %6, 54
  br i1 %7, label %8, label %12

; <label>:8:                                      ; preds = %0
  %9 = load i8, i8* %2, align 1
  %10 = sext i8 %9 to i32
  %11 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([8 x i8], [8 x i8]* @.str.1, i32 0, i32 0), i32 %10)
  br label %14

; <label>:12:                                     ; preds = %0
  %13 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str.2, i32 0, i32 0))
  br label %14

; <label>:14:                                     ; preds = %12, %8
  ret i32 0
}

```

下面是使用 `opt-8 -p -load ../build/obfuscate/libobfuscate.so -obfstr test.ll -o test.bc` 调用 Pass 得到的混淆过的 IR，可以发现字符串已经被混淆成功了。

```
@.str = private unnamed_addr global [26 x i8] c"~BCY\0ACY\0AK\0A^OY^CDM\0AY^XCDM\0B\00", align 1
@.str.1 = private unnamed_addr global [8 x i8] c"\1C\1C\1C\1C\0FI \00", align 1
@.str.2 = private unnamed_addr global [7 x i8] c"}~l\15\0B \00", align 1

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i8, align 1
  store i32 0, i32* %1, align 4
  %3 = call i8* @__decrypt(i8* getelementptr inbounds ([26 x i8], [26 x i8]* @.str, i32 0, i32 0))
  %4 = call i32 @puts(i8* getelementptr inbounds ([26 x i8], [26 x i8]* @.str, i32 0, i32 0))
  %5 = call i8* @__encrypt(i8* getelementptr inbounds ([26 x i8], [26 x i8]* @.str, i32 0, i32 0))
  %6 = call i32 @getchar()
  %7 = trunc i32 %6 to i8
  store i8 %7, i8* %2, align 1
  %8 = sext i8 %7 to i32
  %9 = icmp eq i32 %8, 54
  br i1 %9, label %10, label %16

; <label>:10:                                     ; preds = %0
  %11 = load i8, i8* %2, align 1
  %12 = sext i8 %11 to i32
  %13 = call i8* @__decrypt(i8* getelementptr inbounds ([8 x i8], [8 x i8]* @.str.1, i32 0, i32 0))
  %14 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([8 x i8], [8 x i8]* @.str.1, i32 0, i32 0), i32 %12)
  %15 = call i8* @__encrypt(i8* getelementptr inbounds ([8 x i8], [8 x i8]* @.str.1, i32 0, i32 0))
  br label %20

; <label>:16:                                     ; preds = %0
  %17 = call i8* @__decrypt(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str.2, i32 0, i32 0))
  %18 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str.2, i32 0, i32 0))
  %19 = call i8* @__encrypt(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str.2, i32 0, i32 0))
  br label %20

; <label>:20:                                     ; preds = %16, %10
  ret i32 0
}

```

在 IDA 中也可以看到混淆的效果。

![](https://pic4.zhimg.com/v2-f3ed96cf9947d000f069dddd1eabe9af_r.jpg)

完整代码
----

obfuscate.cpp

```
//===-- obfuscate/obfuscate.cpp - main file of obfuscate_string -*- C++ -*-===//
//
// The main file of the obfusate_string pass.
// LLVM project is under the Apache License v2.0 with LLVM Exceptions.
// See https://llvm.org/LICENSE.txt for license information.
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
//
//===----------------------------------------------------------------------===//
///
/// \file
/// This file contains the module pass class and decrypt function.
///
//===----------------------------------------------------------------------===//
#include "llvm/ADT/SmallVector.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/GlobalVariable.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/InstrTypes.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Pass.h"
#include "llvm/Support/FormatVariadic.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
#include <map>
#include <vector>

using namespace llvm;

namespace {
/// encrypt strings with xor.
/// \param s input string for encrypt.
void encrypt(std::string &s) {
  for (int i = 0; i < s.length() - 1; i++) {
    s[i] ^= 42;
  }
}

/// A pass for obfuscating const string in modules.
struct ObfuscatePass : public ModulePass {
  static char ID;
  ObfuscatePass() : ModulePass(ID) {}

  virtual bool runOnModule(Module &M) {
    for (GlobalValue &GV : M.globals()) {
      GlobalVariable *GVar = dyn_cast<GlobalVariable>(&GV);
      if (GVar == nullptr) {
        continue;
      }
      std::vector<std::pair<Instruction *, User *>> Target;
      bool hasExceptCallInst = false;
      // Find all user and encrypt const value, then insert decrypt function.
      for (User *Usr : GVar->users()) {
        // Get instruction.
        Instruction *Inst = dyn_cast<Instruction>(Usr);
        if (Inst == nullptr) {
          // If Usr is not an instruction, like i8* getelementptr...
          // Dig deeper to find Instruction.
          for (User *DirecUsr : Usr->users()) {
            Inst = dyn_cast<Instruction>(DirecUsr);
            if (Inst == nullptr) {
              continue;
            }
            if (!isa<CallInst>(Inst)) {
              hasExceptCallInst = true;
              Target.clear();
            } else {
              Target.emplace_back(std::pair<Instruction *, User *>(Inst, Usr));
            }
          }
        }
      }
      if (hasExceptCallInst == false && Target.size() == 1) {
        for (auto &T: Target) {
          obfuscateString(M, T.first, T.second, GVar);
        }
        // Change constant to variable.
        GVar->setConstant(false);
      }
    }
    return true;
  }

  /// Obfuscate string and add decrypt function.
  /// \param M Module
  /// \param Inst Instruction
  /// \param Usr User of the \p GlobalVariable
  /// \param GVar The const string
  void obfuscateString(Module &M, Instruction *Inst, Value *Usr,
                       GlobalVariable *GVar) {
    // Encrypt origin string and replace it encrypted string.
    ConstantDataArray *GVarArr =
        dyn_cast<ConstantDataArray>(GVar->getInitializer());
    if (GVarArr == nullptr) {
      return;
    }
    std::string Origin;
    if (GVarArr->isString(8)) {
      Origin = GVarArr->getAsString().str();
    } else if (GVarArr->isCString()) {
      Origin = GVarArr->getAsCString().str();
    }
    encrypt(Origin);
    Constant *NewConstStr = ConstantDataArray::getString(
        GVarArr->getContext(), StringRef(Origin), false);
    GVarArr->replaceAllUsesWith(NewConstStr);

    // Insert decrypt function above Inst with IRBuilder.
    IRBuilder<> builder(Inst);
    Type *Int8PtrTy = builder.getInt8PtrTy();
    // Create decrypt function in GlobalValue / Get decrypt function.
    SmallVector<Type *, 1> FuncArgs = {Int8PtrTy};
    FunctionType *FuncType = FunctionType::get(Int8PtrTy, FuncArgs, false);
    Constant *DecryptFunc = M.getOrInsertFunction("__decrypt", FuncType);
    // Create call instrucions.
    SmallVector<Value *, 1> CallArgs = {Usr};
    CallInst *DecryptInst = builder.CreateCall(FuncType, DecryptFunc, CallArgs);
    Constant *EncryptFunc = M.getOrInsertFunction("__encrypt", FuncType);
    CallInst *EncryptInst = CallInst::Create(FuncType, EncryptFunc, CallArgs);
    EncryptInst->insertAfter(Inst);
  }
};
} // namespace

char ObfuscatePass::ID = 0;
static RegisterPass<ObfuscatePass> X("obfstr", "obfuscate string");


```

encrypt.c

```
char *__decrypt(char *encStr) {
  char *curr = encStr;
  while (*curr) {
    *curr ^= 42;
    curr++;
  }
  return encStr;
}

char *__encrypt(char *originStr) {
  char *curr = originStr;
  while (*curr) {
    *curr ^= 42;
    curr++;
  }
  return originStr;
}

```