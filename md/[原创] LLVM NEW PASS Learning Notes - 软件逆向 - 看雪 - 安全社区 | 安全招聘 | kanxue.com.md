> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285854.htm)

> [原创] LLVM NEW PASS Learning Notes

一直对代码混淆比较感兴趣，过去的半年刚好有个机会能让我学习并进行了一定的实践，希望将其中的相关经验分享给师傅们，第一篇是对 LLVM NEW Pass 的学习归纳，后续会分享一些混淆的魔改思路与实现思路

> 实验环境：Ubuntu 24.04 LLVM 17  
> 项目地址：https://github.com/zzzcccyyyggg/Kotoamatsukami

LLVM 通过一系列的 passes 来优化 IR 文件，不同的 Pass 作用在 IR 文件不同粒度上，如 Function，Module。而 Pass 中的操作分为两种，Transformation 与 Analysis ，分别用于转化 IR 文件与收集 IR 文件中的信息，而一系列的 pass 则合集则被称为 pass pipeline。

LLVM COMPILE AND INSTALL
------------------------

```
git clone --depth 1 -b release/17.x https://github.com/llvm/llvm-project.git
```

编译相关步骤可参考：https://llvm.org/docs/CMake.html

我的编译命令

```
cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;lld" -DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64" -DCMAKE_BUILD_TYPE=Release -DLLVM_INCLUDE_TESTS=OFF -DLLVM_ENABLE_RTTI=ON -DCMAKE_INSTALL_PREFIX=./build/ ../llvm-project/llvm
ninja -j4
```

编译完成后使用以下命令完成安装，会将编译产物安装到`DCMAKE_INSTALL_PREFIX`指定的文件夹

```
ninja install
```

这里最好安装一下，不然没有`LLVMConfig.cmake`此文件，无法在`CMakeLists.txt`中使用`find_package()`

The LLVM pass manager
---------------------

为了提升框架的灵活程度，LLVM 将源文件 -> 可执行文件的步骤打碎，分为不同的 Passes

![](https://zzzcccimage1.oss-cn-beijing.aliyuncs.com/img/LLVM%E6%B5%81%E7%A8%8B%E5%9B%BE.drawio.png)

而控制这些 Passes 正确运行的，就是 Pass Manager，而 Pass 往往按照其工作范围分为以下几类：

### Module Pass

Module Pass 以整个模块为输入。它在给定模块上执行其工作，可以用于模块内的过程间操作。由于它处理的是整个模块，模块 pass 可以访问和修改模块中的所有函数和全局变量。这使得它特别适合那些需要跨多个函数进行分析或优化的任务。

### Call Graph Pass

Call Graph Pass 操作的是调用图的强连通分量（SCCs）。调用图是一个图结构，其中节点表示函数，边表示函数之间的调用关系。SCC 是图中所有节点互相可达的最大子图。调用图 pass 从底向上遍历这些组件，即从被调用函数开始，逐步向上处理调用它们的函数。

### Function Pass

Function Pass 以单个函数为输入，只对该函数进行操作。它在函数的级别上执行分析或转换，不涉及其他函数。这使得函数 pass 简单且高效，因为它只需要处理一个函数的代码，而无需考虑模块中其他部分的状态。

### Loop Pass

Loop Pass 作用于函数内的循环。它只处理特定循环的内容，进行针对性的优化和转换。  
如开头所提到，在 LLVM 中，除了用于改变 IR 代码的`Transformation Pass`外 ，还存在着`Analysis Pass`用于分析`IR`得到其`CFG`等相关分析结果 而这些分析结果需要在 Pass 之间传递，但是当 Pass 对代码进行改动时，原有的分析结果可能会发生改变，这就需要我们决定是否保存相关分析结果，或是重新分析。所以编写我们的新 Pass 的时候，需要注意的分析结果的保存与丢弃，尽量减少重复分析。

![](https://zzzcccimage1.oss-cn-beijing.aliyuncs.com/img/image-20250304171323792.png)

参考上图，pass 以流水线方式执行。例如，如果有几个函数 pass 应按顺序执行，pass 管理器会在第一个函数上运行每个函数 pass，然后在第二个函数上运行所有函数 pass，依此类推。这种方法的基本思想是通过在有限的数据集（一个 IR 函数）上执行转换，然后移动到下一个有限的数据集，来改善缓存行为。

Implementing a new pass
-----------------------

那么如何实现一个 New Pass，首先需要将我们的 Pass 注册到 PassManager 中，详情参考代码：https://github.com/zzzcccyyyggg/Kotoamatsukami/blob/llvm-17-plugins/src/PassPlugin.cpp

```
#include "llvm/Passes/PassPlugin.h"
#include "AntiDebugPass.h"
#include "BogusControlFlow.h"
#include "Flatten.h"
#include "GVEncrypt.h"
#include "IndirectBranch.h"
#include "IndirectCall.h"
#include "SplitBasicBlock.h"
#include "Substitution.h"
#include "include/AddJunkCodePass.h"
#include "include/Branch2Call.h"
#include "include/Branch2Call_32.h"
#include "include/ForObsPass.h"
#include "include/Loopen.hpp"
#include "llvm/Passes/PassBuilder.h"
using namespace llvm;
 
llvm::PassPluginLibraryInfo getKotoamatsukamiPluginInfo()
{
    return {
        LLVM_PLUGIN_API_VERSION, "Kotoamatsukami", LLVM_VERSION_STRING,
        [](PassBuilder& PB) {
            // first way to use the pass
            PB.registerPipelineParsingCallback(
                [](StringRef Name, ModulePassManager& MPM, ArrayRef) {
                    if (Name == "gv-encrypt"){
                        MPM.addPass(GVEncrypt());
                        return true;
                    }
                    else if (Name == "split-basic-block") {
                        MPM.addPass(SplitBasicBlock());
                        return true;
                    } else if ……
                    return false;
                });
            // second way to use the pass
            PB.registerPipelineStartEPCallback(
                [](ModulePassManager& MPM, OptimizationLevel Level) {
                    MPM.addPass(AntiDebugPass());
                    MPM.addPass(SplitBasicBlock());
                    MPM.addPass(GVEncrypt());
                    MPM.addPass(BogusControlFlow());
                    MPM.addPass(AddJunkCodePass());
                    MPM.addPass(Loopen());
                    MPM.addPass(ForObsPass());
                    MPM.addPass(Branch2Call_32());
                    MPM.addPass(Branch2Call());
                    MPM.addPass(IndirectCall());
                    MPM.addPass(IndirectBranch());
                    MPM.addPass(Flatten());
                    MPM.addPass(Substitution());
                });
        }
    };
}
 
__attribute__((visibility("default"))) extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo()
{
    return getKotoamatsukamiPluginInfo();
} 
```

可以看到以上代码有两处的`MPM.addPass`，其分别对应着两种对 Pass 的调用方式，其中第一种方式是通过 opt 主动调用对应的 pass，使用示例如下

```
opt-17 --load-pass-plugin=SoPath input_file --passes=passname -S -o output_file
```

而第二种方式则是将 Pass 插入到 Clang、opt 等工具提供的 Pipeline 中的某些节点，当这些 Pipeline 被调用时，我们的 Pass 就会在 Pipeline 的对应位置被调用，使用示例如下

```
clang -fpass-plugin=SoPath input_file -O0 -o output_file
```

```
opt-17 --load-pass-plugin=SoPath input_file -passes='default' -S -o output_file 
```

How to make a plugin out of the source code
-------------------------------------------

而调用 Pass 有两种方式，一种是将其编进 LLVM 源码树，另一种是将其编译为 Pass Plugin，在需要使用的时候加载对应的 Plugin，我这里采用的是源码树外构建 Pass Plugin 的方式，个人感觉比较方便，源码树内构建的话可以参考 https://github.com/bluesadi/Pluto

参考 CMakeLists.txt 如下

```
cmake_minimum_required(VERSION 3.10)
project(Kotoamatsukami)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g")
set(LLVM_DIR "/home/zzzccc/llvm-17/llvm-project/build/lib/cmake/llvm")
find_package(LLVM REQUIRED CONFIG)
message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")
include(FetchContent)
FetchContent_Declare(
    json
    SOURCE_DIR /home/zzzccc/cxzz/Kotoamatsukami/lib/json
)
FetchContent_MakeAvailable(json)
message(${LLVM_INCLUDE_DIRS})
include_directories(${LLVM_INCLUDE_DIRS})
include_directories("/home/zzzccc/cxzz/Kotoamatsukami/src/include")
include_directories("/home/zzzccc/cxzz/Kotoamatsukami/lib/json/include")
include_directories("/home/zzzccc/cxzz/Kotoamatsukami/lib/json/single_include/nlohmann")

link_directories(${LLVM_LIBRARY_DIRS})
message(STATUS "WORKING_DIRECTORY: ${WORKING_DIRECTORY}")
include(AddLLVM)
include(HandleLLVMOptions)

add_definitions("${LLVM_DEFINITIONS}")

add_llvm_pass_plugin(Kotoamatsukami 
    src/PassPlugin.cpp
    src/pass/AddJunkCodePass.cpp
    src/pass/Branch2Call.cpp
    src/pass/Branch2Call_32.cpp
    src/pass/ForObsPass.cpp
    src/pass/Loopen.cpp
    src/pass/AntiDebugPass.cpp
    src/pass/SplitBasicBlock.cpp
    src/pass/IndirectBranch.cpp
    src/pass/IndirectCall.cpp
    src/pass/BogusControlFlow.cpp
    src/pass/Substitution.cpp
    src/pass/Flatten.cpp
    src/pass/GVEncrypt.cpp
    src/utils/utils.cpp
    src/utils/config.cpp
    src/utils/TaintAnalysis.cpp
    )

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -rdynamic")
target_link_libraries(Kotoamatsukami nlohmann_json::nlohmann_json)
```

这里`CMakeLists`搞了好一阵子，主要是研究了下`find_package`的用法，参考以下两篇文章：https://blog.csdn.net/qq_39466755/article/details/130912344 、https://zhuanlan.zhihu.com/p/50829542

这里首先明确`find_package`有两种模式，如下

> `module模式`  
> 在这个模式下会查找一个名为 find.cmake 的文件，首先去 CMAKE_MODULE_PATH 指定的路径下去查找，然后去 cmake 安装提供的查找模块中查找（安装 cmake 时生成的一些 cmake 文件）。找到之后会检查版本，生成一些需要的信息。
> 
> `config模式`  
> 在这个模式下会查找一个名为 - config.cmake（<小写包名>-config.cmake）或者 Config.cmake 的文件，如果指定了版本信息也会搜索名为 - config-version.cmake 或者 ConfigVersion.cmake 的文件。

而 LLVM 编译安装后会在 / install/lib/llvm 下存在相应的`LLVMConfig.cmake`，而其主要搜寻路径如下

> ```
> _DIR
> CMAKE_PREFIX_PATH
> CMAKE_FRAMEWORK_PATH
> CMAKE_APPBUNDLE_PATH
> PATH 
> ```

故只需要设置 LLVM_DIR 并将 find_package 改为搜寻 config 模式即可

```
set(LLVM_DIR "/home/zzzccc/llvm-project/build/lib/cmake/llvm")
find_package(LLVM REQUIRED CONFIG)
```

编译成功后即可使用 clang 或 opt 调用

参考
--

《Learn_LLVM_17_A_beginner's_guide_to_learning_LLVM_2nd》

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#软件保护](forum-4-1-3.htm)