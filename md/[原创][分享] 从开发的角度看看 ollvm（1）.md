> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274996.htm#msg_header_h1_2)

> [原创][分享] 从开发的角度看看 ollvm（1）

环境搭建
====

Ubuntu 20.04  
CLion2022.02  
llvm 9.0.1  
ubuntu 的镜像我这里就不提供了，直接去官网下载就好啦  
clion 我这里有一篇 pojie 文章分享给大家：[CLion_pojie](https://mp.weixin.qq.com/s?__biz=MzUwOTQwNTUzNQ==&mid=2247488592&idx=1&sn=b85332dbd0528fd21d250ecc603727d6&chksm=f913e460ce646d76e8f1f74c7c37ff5cd4ac200b8fa4fcb2c42e87b9b2451f994112b572796b&token=1615723743&lang=zh_CN#rd)  
然后 llvm 大家直接去官网下载就好啦 [llvm 官网](https://releases.llvm.org/download.html#9.0.1)

第一步就要编译 llvm
------------

[编译 llvm](https://www.cnblogs.com/qiuweidong/p/16215647.html)  
（这里我插一句，大家一定不要相信 llvm 官网的鬼话，说的用 windows 上的 visual studio 也可以编译，用他的那个 powershell，呵呵，之后你就会遇到各种各样奇奇怪怪的错误，还是用 linux 系统吧）  
![](https://bbs.pediy.com/upload/tmp/939330_25KNVZUHFFM36JD.png)  
第一步用 cmake 生成相关的构建文件

```
cmake -G Ninja -DCMAKE_BUILD_TYPE=RELEASE -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_ENABLE_PROJECTS="clang"  -DLLVM_OPTIMIZED_TABLEGEN=ON ../llvm

```

> 其中 - G Ninja 参数表示生成 Ninja 系统的构建文件，采用 Ninja 系统会有比较快的编译速度  
> -DCMAKE_BUILD_TYPE=RELEASE 表示生成 Release 版本的 LLVM，这种构建方式会进行优化，并且生成的目标文件体积会更小  
> -DLLVM_TARGETS_TO_BUILD="X86" 表示编译的目标平台是 X86 平台  
> -DLLVM_ENABLE_PROJECTS="clang" 表示我们除了编译 LLVM 以外，还要编译 clang

 

然后运行编译命令 ninja  
这一步会等待很长时间，cpu 基本处于满载状态：  
![](https://bbs.pediy.com/upload/tmp/939330_Q89K54EFSRW8GNH.png)  
按照这样的顺序编译完成的环境中，clang 应该在这个目录下：  
![](https://bbs.pediy.com/upload/tmp/939330_KXN3PNFVVPZV4WM.png)  
这样 clang 的环境就弄好了

然后再弄 CLion
----------

![](https://bbs.pediy.com/upload/tmp/939330_BAZTU352FWUS5EW.png)  
按照我之前发的 pojie 文章安装好 clion 之后直接打开这个 CMakelists 文件，按照 project 的方式打开：  
![](https://bbs.pediy.com/upload/tmp/939330_UPYMMG6EDCWN2B3.png)  
然后导入文件就好啦，然后再 cmake 的设置中加入 debug 和 release：  
![](https://bbs.pediy.com/upload/tmp/939330_3FETH74SR2QD78W.png)  
输入 -G Ninja -DLLVM_ENABLE_PROJECTS="clang"  
就会生成两个文件：  
![](https://bbs.pediy.com/upload/tmp/939330_QZ9FFBFXQJ372EA.png)  
之后要从这两个文件的目录中编译出 so 文件，然后用 opt 命令操作  
然后按照 llvm 官网的 write pass 的文章中，在这个目录下编写 cpp 和 txt 文件：  
![](https://bbs.pediy.com/upload/tmp/939330_N2XHGVKVBEUQRQD.png)  
![](https://bbs.pediy.com/upload/tmp/939330_7JACD5HXEJ2GHFS.png)  
再添加一下：  
![](https://bbs.pediy.com/upload/tmp/939330_8B96YPAZVYFZ32P.png)  
最后用 ninja 编译一下这个工程，建议编译 release 版本，速度快：  
![](https://bbs.pediy.com/upload/tmp/939330_HXFQSVECNCSR2MY.png)  
如果能生成这个 so 文件，恭喜您，环境搭建完成  
**_（windows 的铁粉在 win 上搭环境弄了三天最后还是 g 了！）_**

编写函数加密 pass
===========

修改函数名称
------

我这里贴一下 CLion 中的 cpp 的代码是怎么实现函数加密的吧：

```
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
using namespace llvm;
namespace {
struct EncodeFunctionName : public FunctionPass {
  static char ID;
  EncodeFunctionName() : FunctionPass(ID) {}
  bool runOnFunction(Function &F) override {
    errs() << "Encode Function Name: ";
    if(F.getName().compare("test_hello1") == 0){
      F.setName("kanxue function");
    }
    errs()< X("Encode", "Encode Function Name Pass",
                                          false /* Only looks at CFG */,
                                          false /* Analysis Pass */);
 
static llvm::RegisterStandardPasses Y(
    llvm::PassManagerBuilder::EP_EarlyAsPossible,
    [](const llvm::PassManagerBuilder &Builder,
       llvm::legacy::PassManagerBase &PM) { PM.add(new EncodeFunctionName()); }); 
```

还有对应的 CMakeList 文件：  
![](https://bbs.pediy.com/upload/tmp/939330_VM8P7CBB6YPJW94.png)  
然后使用 ninja 命令，就能生成 so 文件（作为函数加密的 pass）：

```
ninja LLVMEncodeFunctionName

```

在这个路径下会找到生成的 so 文件

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/lib/LLVMEncodeFunctionName.so'

```

然后新建一个文件夹作为测试空间  
将 clang 的路径添加到文件夹的环境变量中：  
![](https://bbs.pediy.com/upload/tmp/939330_T7Y2FBDCBXCRQNC.png)

```
export PATH=/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin:$PATH

```

首先新建一个文件夹，编写 c 文件（作为测试文件），然后生成 ll 文件，  
这里我总结了几个命令：

```
clang -emit-llvm -S hello_clang.c -o hello_clang.ll
lli  hello_clang.ll
llvm-as hello_clang.ll -o hello_clang.bc
llc hello_clang.bc -o hello_clang.s
clang hello_clang.s -o hello_clang_s
llvm-dis hello_clang.bc -o hello_clang_re.ll

```

这个是 hello.c 文件：

```
#include void test_hello2();
 
void test_hello1() {
    printf("test_hello1\r\n");
}
 
 
int main(int argc, char const *argv[])
{
    if (argc > 2) {
        printf("hello kanxue clang!\r\n");
    } else {
        printf("hello pediy clang!\r\n");
    }
    test_hello1();
    test_hello2();
    return 0;
}
 
 
void test_hello2() {
    printf("test_hello2\r\n");
} 
```

然后利用上面的命令生成 ll 文件  
![](https://bbs.pediy.com/upload/tmp/939330_DDA5T6JZMHW5N9A.png)  
到这里说明我们的文件夹的环境配置好了  
然后接下来该使用 opt 了，（opt 就是把刚刚编写的 cpp 文件生成的 so 文件加载编译的一个 llvm 工具）（所以刚刚的 cpp 文件中就应该实现对函数的加密），我这里遇到了一个问题，虽然设置了环境变量，但是 opt 不能用，所以直接从变量路径中拖出来 opt 使用也可以的：  
![](https://bbs.pediy.com/upload/tmp/939330_7WR536RP2WEV36C.png)

 

干脆直接拖吧，这样还省事  
这里用哪个编译出来的 clang 和 opt 可以  
location1

```
'/home/linuxer/llvm/build/bin/'

```

我又在 release 里面编译了一遍，所以之后的路径是  
location2

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/'

```

这样就能生成 bc 文件

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/opt'
-load
 '/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/lib/LLVMEncodeFunctionName.so'
-Encode hello_clang.ll -o hello_clang_encode.bc

```

然后我们就可以发现对应的函数名称发生了变化

 

![](https://bbs.pediy.com/upload/tmp/939330_WUUEV9KYU7RCHXR.png)  
然后使用 clang 编译一下：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 hello_clang_encode.bc  -o hello_clang_encode
然后执行一下：

```

![](https://bbs.pediy.com/upload/tmp/939330_WZHU32U7PS3TFYB.png)  
程序正常执行  
然后用 ida 看看函数加密情况  
![](https://bbs.pediy.com/upload/tmp/939330_432K3NRDWMUJ7KZ.png)  
![](https://bbs.pediy.com/upload/tmp/939330_QWPAZ36UC52GYAN.png)  
发现函数加密成功！

对函数进行 md5 加密
------------

贴一下 cpp 代码：

```
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
using namespace llvm;
namespace {
struct EncodeFunctionName : public FunctionPass {
  static char ID;
  EncodeFunctionName() : FunctionPass(ID) {}
  bool runOnFunction(Function &F) override {
    errs() << "Encode Function Name: "< ";
    if(F.getName().compare("main") == 0){
      llvm::MD5 Hasher;
      llvm::MD5::MD5Result Hash;
      Hasher.update("kanxue_");
      Hasher.update(F.getName());
      Hasher.final(Hash);
 
      SmallString<32> HexString;
      llvm::MD5::stringifyResult(Hash, HexString);
 
      F.setName(HexString);
    }
    errs()< X("Encode", "Encode Function Name Pass",
                                          false /* Only looks at CFG */,
                                          false /* Analysis Pass */);
 
static llvm::RegisterStandardPasses Y(
    llvm::PassManagerBuilder::EP_EarlyAsPossible,
    [](const llvm::PassManagerBuilder &Builder,
       llvm::legacy::PassManagerBase &PM) { PM.add(new EncodeFunctionName()); }); 
```

然后 ninja 编译一下，生成 so 文件  
还是用刚刚的 opt-load 命令：  
![](https://bbs.pediy.com/upload/tmp/939330_PZ8HWTQ6QCB8JB4.png)  
发现 main 函数的名字已经被 md5 加密了  
![](https://bbs.pediy.com/upload/tmp/939330_73DZFG3C2Z8ZHZ9.png)  
这里说一句，刚刚的 cpp 文件把 main 函数的名字给 md5 加密了，这样是不可以的：  
![](https://bbs.pediy.com/upload/tmp/939330_9ANE8W7FW5MGS8X.png)  
所以刚刚的 cpp 文件应该修改这一处的文件  
![](https://bbs.pediy.com/upload/tmp/939330_JZAMBGX8KUBEAWF.png)  
修改完之后就没有问题了，将除了 main 函数以外的所有的函数的名字都进行了 md5 加密  
![](https://bbs.pediy.com/upload/tmp/939330_S7KU6QEE5DH2DK3.png)  
![](https://bbs.pediy.com/upload/tmp/939330_TNRK5K2GMN8VHFG.png)  
然后尝试在源码之外开发一个 pass，因为这样一直使用源码开发很麻烦：[developing-llvm-passes-out-of-source](https://llvm.org/docs/CMake.html#developing-llvm-passes-out-of-source)  
按照官方文档给的结构创建一个工程  
![](https://bbs.pediy.com/upload/tmp/939330_7XKDNW94J9UTGYJ.png)  
创建完工程后将工程导入 clion 中，然后根据报错信息一个个的改就好啦  
这里学习了几个 linux 的命令，纪录一下：  
在某个目录下找文件

```
find . -name "LLVMConfig.cmake"

```

找到该文件的绝对路径：

```
readlink -f ./lib/cmake/llvm/LLVMConfig.cmake

```

我这里贴一下 clion 中的代码和工程目录吧：  
![](https://bbs.pediy.com/upload/tmp/939330_CAJA7TGKKYAXA6P.png)  
CMakeLists.txt

```
add_library( LLVMEncodeFunctionName2 MODULE
        EncodeFunctionName.cpp
 
        )

```

EncodeFunctionName.cpp 和原来一样  
CmakeLists.txt

```
cmake_minimum_required(VERSION 3.23)
 
project(OUTPASS)
 
set(LLVM_DIR /home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/lib/cmake/llvm/)
 
find_package(LLVM REQUIRED CONFIG)
 
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
include(AddLLVM)
 
 
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})
include_directories(${LLVM_INCLUDE_DIRS})
 
add_subdirectory(EncodeFunctionName2)

```

这两个 cmakelists 要根据自己的错误来配置，路径和版本是不一样的  
然后 build 一下，生成 so 文件  
![](https://bbs.pediy.com/upload/tmp/939330_JD6JG65T4923GWF.png)  
然后继续之前的操作：

```
linuxer@ubuntu:~/llvm/study_llvm/01$ '/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/opt'
-load
'/home/linuxer/llvm/study_llvm/02/outPass/cmake-build-release/EncodeFunctionName2/LLVMEncodeFunctionName2.so'  
-encode2
 hello_clang.ll -o hello_clang_encode.bc
EncodeFunctionName22: test_hello1 -> 9c119247aefaa12cdd417eb3d57d5b2a
EncodeFunctionName22: main -> main
EncodeFunctionName22: test_hello2 -> af0aaac8b98b759ace7b9eacbd2238a6

```

这里不知道出了个什么问题，非要让我用 encode2 执行命令  
这样每次输入命令很麻烦所以接下来尝试把 pass 注册到 clang 里面去

把 pass 注册到 clang 里面去
--------------------

首先创建一个. h 的头文件：  
![](https://bbs.pediy.com/upload/tmp/939330_ZKZM5JPK6TPRFVT.png)  
然后在 cpp 文件中实现这个函数，并且导入相对应的头文件：  
![](https://bbs.pediy.com/upload/tmp/939330_DGF8QQFGMXWATSC.png)

 

.a 是静态库文件  
.so 是一个共享的库  
接下来我们要把 pass 编译成一个静态库也就是一个. a 文件  
修改 EncodeFucntionName 文件夹中的两个文件  
![](https://bbs.pediy.com/upload/tmp/939330_UK6FWUVWKMPSJD6.png)  
![](https://bbs.pediy.com/upload/tmp/939330_349R7TQKXGK95NS.png)  
然后修改 ipo 文件夹中的文件  
在 passermanagerbuilder.cpp 里面加入头文件然后文件中加入参数  
![](https://bbs.pediy.com/upload/tmp/939330_VJPT6UV2CFSYWA5.png)  
![](https://bbs.pediy.com/upload/tmp/939330_SRENFBF8AGB6RRV.png)  
在 llvmbuild.txt 文件中加入 EncodeFunctionName  
![](https://bbs.pediy.com/upload/tmp/939330_PYRY8S8JAE4ZVFZ.png)  
最后我贴上 Encode_Function_name.cpp 的代码:

```
#include "llvm/Pass.h"
#include "llvm/IR/Function.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
#include "llvm/Transforms/EncodeFunctionName/EncodeFunctionName.h"
using namespace llvm;
namespace {
struct EncodeFunctionName : public FunctionPass {
  static char ID;
  bool enable_flag;
  EncodeFunctionName() : FunctionPass(ID) {}
  EncodeFunctionName(bool flag) : FunctionPass(ID) {enable_flag = flag;}
  bool runOnFunction(Function &F) override {
    errs() << "Encode Function Name: "< ";
    if(enable_flag){
      if(F.getName().compare("main") != 0){
        llvm::MD5 Hasher;
        llvm::MD5::MD5Result Hash;
        Hasher.update("kanxue_");
        Hasher.update(F.getName());
        Hasher.final(Hash);
 
        SmallString<32> HexString;
        llvm::MD5::stringifyResult(Hash, HexString);
 
        F.setName(HexString);
      }
    }
 
    errs()< X("Encode", "Encode Function Name Pass",
                                          false /* Only looks at CFG */,
                                          false /* Analysis Pass */);
 
static llvm::RegisterStandardPasses Y(
    llvm::PassManagerBuilder::EP_EarlyAsPossible,
    [](const llvm::PassManagerBuilder &Builder,
       llvm::legacy::PassManagerBase &PM) { PM.add(new EncodeFunctionName()); });
 
 
 
Pass* llvm::createEncodeFunctionName(bool flag){return new EncodeFunctionName(flag);} 
```

这样所有的修改就 ok 了  
然后用 ninja 编译一下 release 版本的就可以了  
![](https://bbs.pediy.com/upload/tmp/939330_7K8M96UVPMWMDEW.png)  
成功编译出. a 文件  
然后重新编译一下 clang

 

![](https://bbs.pediy.com/upload/tmp/939330_FU29GJF4ANG4HTS.png)

 

然后用 clang 编译一下

```
linuxer@ubuntu:~/llvm/study_llvm/01$
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang' 
-mllvm
 -encode_function_name hello_clang.ll
 
结果：
Encode Function Name: test_hello1 -> 9c119247aefaa12cdd417eb3d57d5b2a
Encode Function Name: main -> main
Encode Function Name: test_hello2 -> af0aaac8b98b759ace7b9eacbd2238a6

```

（就能达到和之前一样的效果）  
![](https://bbs.pediy.com/upload/tmp/939330_2ZF6KBZE7SD7SYQ.png)  
这里总结一下思路：  
就是把之前的 opt -load xxxx.pass -encode 命令替换成了 clang，也就是我们之前说的将 llvmpass 集成到 clang 里面，也就是说只用 clang 就能实现函数加密，所以我们要先用 ninja 编译一下 clang

将 ollvm 代码移植到 llvm
==================

首先从 git 上下载第四次分支的代码  
然后复制代码 obfuscation 到 llvm/lib/Tranforms 和 llvm/include/lib/Tranforms  
![](https://bbs.pediy.com/upload/tmp/939330_JFWK3RED8HY59G3.png)  
然后对着 ollvm 一个个的改，这里不是重点，就不一一介绍了  
然后再修改代码的过程中，有一个 AESsead 需要注意一下：  
这个是随机生成的种子，如果种子一样的画，就会生成一样的方案  
如果是随机的种子，那么编译出来的方案也是随机的  
然后修改完源码会由于环境版本不匹配问题产生很多错误，一个个的改就好啦 google 上 github 上面都有，[放个链接吧](https://github.com/obfuscator-llvm/obfuscator/pull/76/files)  
![](https://bbs.pediy.com/upload/tmp/939330_KHFZ4JMVG4KY3XX.png)  
这样就是编译成功了  
![](https://bbs.pediy.com/upload/tmp/939330_7JXNWVUTUGY5QMW.png)  
然后再编译一下 clang：

```
linuxer@ubuntu:~/llvm/llvm_source_code/llvm/cmake-build-release$ ninja LLVMObfuscation
[3/3] Linking CXX static library lib/libLLVMObfuscation.a
linuxer@ubuntu:~/llvm/llvm_source_code/llvm/cmake-build-release$ ninja clang

```

这样所有的环境搭建好之后，看看官网上面的例子  
![](https://bbs.pediy.com/upload/tmp/939330_H8E2PH6GZB6AJZ8.png)

接下来正式进入 ollvm
-------------

### 指令替换

```
https://github.com/obfuscator-llvm/obfuscator/wiki/Instructions-Substitution

```

前面三种都是混淆用的  
最后一个选项是自己创建函数选择要不要混淆函数  
看第一个选项再官网中的描述，也就是其他文章中的指令替换  
![](https://bbs.pediy.com/upload/tmp/939330_JBSCHN97XZ7PBBP.png)  
前两种混淆再 ida 中都可以自动的帮我们优化掉，然后加了随机数之后 ida 就不行了

### 指令替换

然后编写 c 文件作为测试用例：

```
#include int main(int argc, char const *argv[])
{
    int n = argc + 8;
    if (n >= 10) {
        printf("hello ollvm:%d\r\n", n);
    } else {
        printf("hello kanxue\r\n");
    }
    return 0;
} 
```

然后用刚刚生成的 clang（里面已经集成及 ollvm 的 pass）编译一下：  
从编译的结果并看不出来什么不同  
![](https://bbs.pediy.com/upload/tmp/939330_WE9FWVBXBHV9PS8.png)  
然后拖入 ida7.5 看看  
![](https://bbs.pediy.com/upload/tmp/939330_FEWG4D29QPYUBYB.png)  
指令只有这一处发生了变化：  
add eax, 8  
变成了：  
sub eax, 8BAA3A92h  
add eax, 8  
add eax, 8BAA3A92h  
如果看伪 c 代码两个的结果是一样的  
然后增加混淆力度看看：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -sub
 -mllvm -sub_loop=6
 hello_ollvm_bcf.c -o hello_ollvm_sub_6

```

拖入 ida 中看看：  
![](https://bbs.pediy.com/upload/tmp/939330_FNP8VM84TYW6K94.png)

### 虚假控制流

贴一下 c 源码：

```
#include int main(int argc, char** argv) {
  int a = atoi(argv[1]);
  if(a == 0)
    return 1;
  else
    return 10;
  return 0;
} 
```

执行一下混淆命令：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -bcf
-mllvm -bcf_loop=6
 hello_ollvm_bcf.c -o hello_ollvm_bcf_6

```

然后拖入 ida 看看  
先看看 bcf_loop=1 的时候：  
![](https://bbs.pediy.com/upload/tmp/939330_Y8GMEJX6RAVF3P3.png)  
他就比源程序混淆出来一个 while 语句（然后我刚刚动态调试的时候发现 x,y 的值都是 0，所以这个 while 分支是不成立的）：  
![](https://bbs.pediy.com/upload/tmp/939330_8TBKPAPCSCDHKD2.png)  
这个时候还能看看，然后加到 3：  
![](https://bbs.pediy.com/upload/tmp/939330_2AGM87KTWNCCWHK.png)  
发现他的混淆力度还是可以的一条 if 语句出来这么多分支  
然后要是动态跟着一路走过去发现，所有的 while 语句都只判断了前半部分，剩下的都没执行  
![](https://bbs.pediy.com/upload/tmp/939330_74723ZDQERQPBT7.png)  
![](https://bbs.pediy.com/upload/tmp/939330_QW7EDAA895MEXFM.png)  
走到了最后函数的返回值是 10，然后返回 linux 虚拟机看看返回值，确实是 10  
![](https://bbs.pediy.com/upload/tmp/939330_TBEZRTVAS3E54DH.png)  
这一大堆不成立的 while 语句就是其他文章中的虚假控制流吧  
我给大家贴一下程序的执行流程吧:  
![](https://bbs.pediy.com/upload/tmp/939330_JJ54DSEYACZQDDT.png)  
![](https://bbs.pediy.com/upload/tmp/939330_WHFMEW8MDS6PM5G.png)  
![](https://bbs.pediy.com/upload/tmp/939330_TQ7EJ6482E9CCA9.png)  
然后一直到这里才是我们程序的原始的分支  
![](https://bbs.pediy.com/upload/tmp/939330_YFSBYPH7YCQZDWS.png)  
然后又是虚假控制流：  
![](https://bbs.pediy.com/upload/tmp/939330_TS63VKJ3Y4ETF8Z.png)  
![](https://bbs.pediy.com/upload/tmp/939330_URSBQXT6P8MRRCB.png)  
一直到这里程序才执行完：  
![](https://bbs.pediy.com/upload/tmp/939330_U3MP2RJ88ZZD7TJ.png)  
混淆到 9 的时候直接出不来了 xswl：  
![](https://bbs.pediy.com/upload/tmp/939330_5KHJEMCSABNVFX2.png)  
然后改成 999：  
![](https://bbs.pediy.com/upload/tmp/939330_HY6ZFZSJ3WFZGEB.png)  
然后如果想看伪 c 代码的话也会耗费很长时间：  
![](https://bbs.pediy.com/upload/tmp/939330_XRXA3D54UJVYY4W.png)  
bcf：  
![](https://bbs.pediy.com/upload/tmp/939330_699NJHMBM5DVF2R.png)  
太 6 了  
然后我们会发现一个规律，就是无论你 bcf_loop 到多少，总会有一段或者两端的 while1 的内容来恶心你：  
![](https://bbs.pediy.com/upload/tmp/939330_JAWZZXH537T89AX.png)  
![](https://bbs.pediy.com/upload/tmp/939330_SDUF8X4PTR7BR8H.png)  
如果要是混淆用到这个手法的话，可以用动态调试的方法来跟着程序执行，不过这样比较慢，应为许多不执行的 while 语句（但是 ida 不这么认为啊）也会跟到程序的执行过程中，所以我想出来了一种跟踪变量的方法，可以正着跟，也可以从函数的返回值出发：  
这里我简单的说一下我的想法，当然网上也有很多别的更好的方法，一会会说一下 unicorn 模拟执行，我这个跟踪变量的方法只能用于比较简单的程序：  
先从 v19-》v22  
![](https://bbs.pediy.com/upload/tmp/939330_27H5YBBQH6RW8ZQ.png)  
然后往上跟，发现了两个值，正好是我们刚刚源程序中写的值 1，10：  
![](https://bbs.pediy.com/upload/tmp/939330_JFKHFZGMV5CXJ26.png)  
然后这样一直网上跟也能发现程序的逻辑，不过这种方法比较恶心，还是学习一下其他大佬的 unicorn 方法吧，还有符号执行方法  
然后这个虚假控制流还可以指定混淆的百分比：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -bcf
 -mllvm -bcf_loop=3
-mllvm -bcf_prob=80
 hello_ollvm_bcf.c -o hello_ollvm_bcf_3_80

```

发现他的混淆力度也是很强的：  
![](https://bbs.pediy.com/upload/tmp/939330_RBZ679VN7AJWPDA.png)

### 控制流平坦化

```
https://github.com/obfuscator-llvm/obfuscator/wiki/Features

```

看官方文档介绍就是将 if-else 分支的基本块，转化成 switch 的基本快  
![](https://bbs.pediy.com/upload/tmp/939330_2ED3JHX5N7F3ESA.png)  
我们这里还是用官网中给的 c 代码：

```
#include int main(int argc, char** argv) {
  int a = atoi(argv[1]);
  if(a == 0)
    return 1;
  else
    return 10;
  return 0;
} 
```

执行一下混淆代码：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -fla
 hello_ollvm_fla.c -o hello_ollvm_fla

```

然后拖入 ida 看看:  
混淆的也比较明显了  
![](https://bbs.pediy.com/upload/tmp/939330_RKG9MPWNR5QE6N2.png)  
ida 中识别的 switch-case 语句不是很准确  
然后生成一下. ll 文件看看程序流程就比较明显了：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -fla -emit-llvm -S
hello_ollvm_fla.c -o hello_ollvm_fla.ll

```

代码如下：

```
; ModuleID = 'hello_ollvm_fla.c'
source_filename = "hello_ollvm_fla.c"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"
 
; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @main(i32, i8**) #0 {
  %3 = alloca i32
  %4 = alloca i32, align 4
  %5 = alloca i32, align 4
  %6 = alloca i8**, align 8
  %7 = alloca i32, align 4
  store i32 0, i32* %4, align 4
  store i32 %0, i32* %5, align 4
  store i8** %1, i8*** %6, align 8
  %8 = load i8**, i8*** %6, align 8
  %9 = getelementptr inbounds i8*, i8** %8, i64 1
  %10 = load i8*, i8** %9, align 8
  %11 = call i32 @atoi(i8* %10) #2
  store i32 %11, i32* %7, align 4
  %12 = load i32, i32* %7, align 4
  store i32 %12, i32* %3
  %13 = alloca i32
  store i32 804133528, i32* %13
  br label %14
 
14:                                               ; preds = %2, %25
  %15 = load i32, i32* %13
  switch i32 %15, label %16 [
    i32 804133528, label %17
    i32 1846244677, label %21
    i32 1517211666, label %22
    i32 -1600483670, label %23
  ]
 
16:                                               ; preds = %14
  br label %25
 
17:                                               ; preds = %14
  %18 = load volatile i32, i32* %3
  %19 = icmp eq i32 %18, 0
  %20 = select i1 %19, i32 1846244677, i32 1517211666
  store i32 %20, i32* %13
  br label %25
 
21:                                               ; preds = %14
  store i32 1, i32* %4, align 4
  store i32 -1600483670, i32* %13
  br label %25
 
22:                                               ; preds = %14
  store i32 10, i32* %4, align 4
  store i32 -1600483670, i32* %13
  br label %25
 
23:                                               ; preds = %14
  %24 = load i32, i32* %4, align 4
  ret i32 %24
 
25:                                               ; preds = %22, %21, %17, %16
  br label %14
}
 
; Function Attrs: nounwind readonly
declare dso_local i32 @atoi(i8*) #1
 
attributes #0 = { noinline nounwind optnone uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { nounwind readonly "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #2 = { nounwind readonly }
 
!llvm.module.flags = !{!0}
!llvm.ident = !{!1}
 
!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 9.0.1 "}

```

然后加一些参数看看：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
-mllvm -fla -mllvm -split
hello_ollvm_fla.c -o hello_ollvm_fla_split

```

![](https://bbs.pediy.com/upload/attach/202211/939330_SMB8SGZ4ZJSQSE5.png)  
然后把混淆力度加到三看看：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang' -mllvm -fla -mllvm -split  -mllvm -split_num=3  hello_ollvm_fla.c -o hello_ollvm_fla_split_3

```

cfg：  
![](https://bbs.pediy.com/upload/attach/202211/939330_NZFEF5U5GQZQBBB.png)  
其实也没有很复杂这样看来  
然后还可以指定函数来进行混淆:

```
#include int foo() __attribute((__annotate__(("fla"))));
int foo() {
   return 2;
}
int main(int argc, char** argv)  __attribute((__annotate__(("nofla"))))   __attribute((__annotate__(("nosub"))))   __attribute((__annotate__(("nobcf"))));
int main(int argc, char** argv) {
  int a = atoi(argv[1]) + foo();
  if(a == 0)
    return 1;
  else
    return 10;
  return 0;
} 
```

然后输入命令：

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -fla
-mllvm -bcf
 -mllvm -sub
 hello_ollvm_bsf.c -o hello_ollvm_bsf

```

看看 cfg：  
我们会发现，main 函数没有被混淆  
![](https://bbs.pediy.com/upload/attach/202211/939330_9Y9X5WXTFNCBHGM.png)  
然后这个 foo 函数被混淆了  
然后融合所有科技来看看：  
可见混淆的力度是很大的，编译过程中都花费了很多时间

```
'/home/linuxer/llvm/llvm_source_code/llvm/cmake-build-release/bin/clang'
 -mllvm -fla
 -mllvm -split -mllvm -split_num=3
  -mllvm -bcf -mllvm -bcf_loop=3
-mllvm -sub -mllvm -sub_loop=3
 hello_ollvm_fla.c -o hello_ollvm_mix

```

![](https://bbs.pediy.com/upload/attach/202211/939330_7NHBTNFXFFBK3DF.png)  
还有 clang 的调试啥的，下一篇文章分享给大家吧，字数太多了，打字都卡了  
最后感谢一下 imyang 大佬的文章

[看雪 2022 KCTF 秋季赛 防守篇规则，征题截止日期 11 月 12 日！（iPhone 14 等你拿！）](https://bbs.pediy.com/thread-274513.htm)

最后于 5 小时前 被以和爲貴编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#程序开发](forum-161-1-124.htm)