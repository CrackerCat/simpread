> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270426.htm)

> [原创]【符号执行】Ubuntu20.04 下 KLEE 安装教程 + KLEE 论文阅读笔记 +.md

论文《KLEE: Unassisted and Automatic Generation of High-Coverage  
Tests for Complex Systems Programs》的阅读笔记如下，安装教程在后面。

2.1 KLEE 的用法
------------

KLEE 的使用：用户用公开的 GNU C 的 LLVM 编译器去编译成字节码

```
llvm-gcc --emit-llvm -c tr.c -o tr.bc

```

然后 KLEE 运行字节码文件。

```
klee --max-time 2 --sym-args 1 10 10
--sym-files 2 2000 --max-fail 1 tr.bc

```

max-time：测试的分钟数  
--sym-args 1 10 10：表示使用 0 至 3 个命令行参数，第一个参数是一字符长，剩下两个是 10 字符长。  
--sym-files 2 2000：表示使用标准输入和一个文件，每个数据包含 2000 字节的符号数据。  
--max-fail 1：选项表示每个程序路径最多一个系统调用失败

```
示例程序
1 : void expand(char *arg, unsigned char *buffer) { 8
2 : int i, ac; 9
3 : while (*arg) { 10*
4 : if (*arg == ’\\’) { 11*
5 : arg++;
6 : i = ac = 0;
7 : if (*arg >= ’0’ && *arg <= ’7’) {
8 : do {
9 : ac = (ac << 3) + *arg++ − ’0’;
10: i++;
11: } while (i<4 && *arg>=’0’ && *arg<=’7’);
12: *buffer++ = ac;
13: } else if (*arg != ’\0’)
14: *buffer++ = *arg++;
15: } else if (*arg == ’[’) { 12*
16: arg++; 13
17: i = *arg++; 14
18: if (*arg++ != ’-’) { 15!
19: *buffer++ = ’[’;
20: arg −= 2;
21: continue;
22: }
23: ac = *arg++;
24: while (i <= ac) *buffer++ = i++;
25: arg++; /* Skip ’]’ */
26: } else
27: *buffer++ = *arg++;
28: }
29: }
30: . . .
31: int main(int argc, char* argv[ ]) { 1
32: int index = 1; 2
33: if (argc > 1 && argv[index][0] == ’-’) { 3*
34: . . . 4
35: } 5
36: . . . 6
37: expand(argv[index++], index); 7
38: . . .
39: }

```

第 18 行有缓冲区溢出，

2.2 KLEE 符号执行的具体过程。
-------------------

1. 命令行运行

```
klee --max-time 2 --sym-args 1 10 10
--sym-files 2 2000 --max-fail 1 tr.bc

```

2. 首先会遇到第 33 行的分支，会把 argc>1 和 argc<=1 约束用 STP 约束求解器求解，然后发现两个约束都可以，然后会生成两个执行路径。  
3. 有多个执行路径的话，KLEE 要选择一条路径线去执行。（详细的算法会在 3.4 节讲）这里我们假设会往到达 bug 的路径执行，然后后续会添加关于 5 个 argv 的约束（分别在代码的 33，3，4，15 行）。  
4. 在代码 17，18 行中，arg 自增了两次，导致了越界。  
5. 测试用例

```
[ "" ""

```

会触这个错误。这是三个字符串。

3.The KLEE Architecture
=======================

KLEE 是一个基于之前符号执行系统 EXE 的更完整的一个工作。

3.4 state scheduling
--------------------

KLEE 会在每条指令执行时，通过交替使用两个启发式算法选择一个 state。

### Random Path Selection

这个方法存有一棵二叉树，叶子节点是当前的 state, 内部节点是之前执行的路径。state 会在各个分支中随机选择出来。因此，当到达一个分支点时，每个子树中的状态集被选择的概率相同，而不管它们的子树的大小如何。  
这个方法偏爱在分支中的较高的 state。因为这些 states 对其符号输入的约束较少，因此有更大的自由来获取未覆盖的代码。第二，可以避免饥饿（当程序进入包含符号约束的循环时，会有路径 fork 爆炸）。

Coverage-Optimized Search
=========================

覆盖 - 优化搜索试图选择可能在不久的将来覆盖新代码的状态。它使用启发式方法计算每个状态的权值，然后根据这些权值随机选择一个状态。目前，这些启发式方法考虑了到未公开指令的最小距离、状态的调用堆栈以及该状态最近是否覆盖了新代码。

 

KLEE 以循环的方式使用每种策略。虽然这可以增加一个特别有效的策略来实现高覆盖率的时间，但它可以防止个别策略陷入困境的情况。此外，由于策略从同一个状态池中选择，交错它们可以提高整体有效性。

在虚拟机 Ubuntu20.04 中安装 KLEE
=========================

### 安装一些必备工具

```
sudo apt-get install build-essential curl libcap-dev git cmake libncurses5-dev python-minimal python-pip unzip libtcmalloc-minimal4 libgoogle-perftools-dev libsqlite3-dev doxygen

```

```
sudo apt-get install python3 python3-pip gcc-multilib g++-multilib

```

安装了 pip3 后发现用 Pip3 去安装东西出现找不到 Pip3 的问题，先找到 Pip3 的安装路径。

```
bigeast@ubuntu:~$ locate pip3
/usr/local/lib/python3.6/dist-packages/bin/pip3
/usr/local/lib/python3.6/dist-packages/bin/pip3.6
/usr/share/man/man1/pip3.1.gz

```

然后创建软链接到 bin 目录下

```
sudo ln -s /usr/local/lib/python3.6/dist-packages/bin/pip3.6 /usr/local/bin/pip3

```

即可使用 pip3 来安装 ellvm

```
pip3 install lit tabulate wllvm

```

```
sudo apt-get install clang-9 llvm-9 llvm-9-dev llvm-9-tools

```

### 安装 Z3 约束求解器

```
git clone https://github.com/Z3Prover/z3.git
cd z3/
python scripts/mk_make.py
sudo make install

```

### （可选）构建 uClibc 和 POSIX 环境模型（macOS 不支持）、

默认情况下，KLEE 适用于封闭程序（不使用任何外部代码的程序，例如 C 库函数）。但是，如果您想使用 KLEE 来运行实际程序，您将需要启用 KLEE POSIX 运行时，它构建在 uClibc C 库之上。

```
bigeast@ubuntu:~$ git clone https://github.com/klee/klee-uclibc.git
bigeast@ubuntu:~$ cd klee-uclibc
bigeast@ubuntu:~/klee-uclibc$ ./configure --make-llvm-lib
INFO:Disabling assertions
INFO:Configuring for Debug build
INFO:Configuring for LLVM bitcode archive
INFO:Using llvm-config at...None
ERROR:llvm-config cannot be found

```

报错，把命令改成如下手动配置路径即可。

```
./configure --make-llvm-lib --with-llvm-config /usr/bin/llvm-config-9

```

### （可选）获取 Google 测试源：

对于单元测试，我们使用 Google 测试库。如果要运行单元测试，则需要执行此步骤，并 - DENABLE_UNIT_TESTS=ON 在步骤 8 中配置 KLEE 时传递给 CMake。

 

我们现在依赖一个版本，1.7.0 所以获取它的来源。

```
curl -OL https://github.com/google/googletest/archive/release-1.7.0.zip
unzip release-1.7.0.zip

```

### 安装 gcc9

```
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-9 g++-9

```

在 klee-2.1 目录下

```
bigeast@ubuntu:~/klee-2.1$ sudo LLVM_VERSION=9 SANITIZER_BUILD= BASE=/usr/include/c++/9 REQUIRES_RTTI=1 DISABLE_ASSERTIONS=1 ENABLE_DEBUG=0 ENABLE_OPTIMIZED=1 ./scripts/build/build.sh libcxx

```

报错如下

```
Detected OS: linux
OS=linux
Detected distribution: ubuntu
DISTRIBUTION=ubuntu
Detected version: 18.04
DISTRIBUTION_VER=18.04
Component sanitizer not found.
Component sanitizer not found.
Already installed llvm
/usr/bin/llvm-config-9
Already installed clang
CMake Error at /usr/share/cmake-3.10/Modules/CMakeDetermineCCompiler.cmake:48 (message):
  Could not find compiler set in environment variable CC:
 
  wllvm.
Call Stack (most recent call first):
  CMakeLists.txt:49 (project)
 
 
CMake Error: CMAKE_C_COMPILER not set, after EnableLanguage
CMake Error: CMAKE_CXX_COMPILER not set, after EnableLanguage
CMake Error: CMAKE_ASM_COMPILER not set, after EnableLanguage
-- Configuring incomplete, errors occurred!
See also "/usr/include/c++/9/libc++-build-9/CMakeFiles/CMakeOutput.log".
make: *** No rule to make target 'cxx'.  Stop.

```

安装 wllvm 解决这个问题

```
bigeast@ubuntu:~/klee-2.1$ sudo -H python3 -m pip install wllvm

```

再次执行

```
bigeast@ubuntu:~/klee-2.1$ sudo LLVM_VERSION=9 SANITIZER_BUILD= BASE=/usr/include/c++/9 REQUIRES_RTTI=1 DISABLE_ASSERTIONS=1 ENABLE_DEBUG=0 ENABLE_OPTIMIZED=1 ./scripts/build/build.sh libcxx

```

```
mkdir build
cd build
cmake \
-DENABLE_POSIX_RUNTIME=ON \
-DENABLE_KLEE_UCLIBC=ON \
-DENABLE_KLEE_LIBCXX=ON \
-DKLEE_LIBCXX_DIR=/usr/include/c++/9/libc++-install-90/ \
-DKLEE_LIBCXX_INCLUDE_DIR=/usr/include/c++/9/libc++-install-90/include/c++/v1/ \
-DENABLE_KLEE_EH_CXX=ON \
-DKLEE_LIBCXXABI_SRC_DIR=/usr/include/c++/9/llvm-90/libcxxabi/ \
-DKLEE_UCLIBC_PATH=/home/bigeast/klee-uclibc/ \
-DENABLE_UNIT_TESTS=ON \
-DENABLE_SYSTEM_TESTS=ON \
-DGTEST_SRC_DIR=/home/bigeast/googletest-release-1.7.0/  \
-DLLVM_CONFIG_BINARY=/usr/bin/llvm-config-9 \
-DLLVMCC=/usr/bin/clang-9 \
-DLLVMCXX=/usr/bin/clang++-9 \
-DCMAKE_C_COMPILER=clang  \
-DCMAKE_CXX_COMPILER=clang++  \
..

```

报错如下：

```
bigeast@ubuntu:~/klee-2.1/build$ cmake -DENABLE_POSIX_RUNTIME=ON -DENABLE_KLEE_UCLIBC=ON -DENABLE_KLEE_LIBCXX=ON -DKLEE_LIBCXX_DIR=/usr/include/c++/9/libc++-install-90/ -DKLEE_LIBCXX_INCLUDE_DIR=/usr/include/c++/9/libc++-install-90/include/c++/v1/ -DENABLE_KLEE_EH_CXX=ON -DKLEE_LIBCXXABI_SRC_DIR=/usr/include/c++/9/llvm-90/libcxxabi/ -DKLEE_UCLIBC_PATH=/home/bigeast/klee-uclibc/ -DENABLE_UNIT_TESTS=ON -DENABLE_SYSTEM_TESTS=ON -DGTEST_SRC_DIR=/home/bigeast/googletest-release-1.7.0/  -DLLVM_CONFIG_BINARY=/usr/bin/llvm-config-9 -DLLVMCC=/usr/bin/clang-9 -DLLVMCXX=/usr/bin/clang++-9 -DCMAKE_C_COMPILER=clang  -DCMAKE_CXX_COMPILER=clang++  ..
-- The CXX compiler identification is unknown
-- The C compiler identification is unknown
CMake Error at CMakeLists.txt:43 (project):
  The CMAKE_CXX_COMPILER:
 
    clang++
 
  is not a full path and was not found in the PATH.
 
  Tell CMake where to find the compiler by setting either the environment
  variable "CXX" or the CMake cache entry CMAKE_CXX_COMPILER to the full path
  to the compiler, or to the compiler name if it is in the PATH.
 
 
CMake Error at cmake/cxx_flags_override.cmake:24 (message):
  Overrides not set for compiler
Call Stack (most recent call first):
  /usr/share/cmake-3.10/Modules/CMakeCXXInformation.cmake:89 (include)
  CMakeLists.txt:43 (project)
 
 
CMake Error at CMakeLists.txt:43 (project):
  The CMAKE_C_COMPILER:
 
    clang
 
  is not a full path and was not found in the PATH.
 
  Tell CMake where to find the compiler by setting either the environment
  variable "CC" or the CMake cache entry CMAKE_C_COMPILER to the full path to
  the compiler, or to the compiler name if it is in the PATH.
 
 
-- Configuring incomplete, errors occurred!
See also "/home/bigeast/klee-2.1/build/CMakeFiles/CMakeOutput.log".
See also "/home/bigeast/klee-2.1/build/CMakeFiles/CMakeError.log".

```

原因是要用 clang-9 和 clang++-9

```
bigeast@ubuntu:~/klee-2.1/build$ cmake -DENABLE_POSIX_RUNTIME=ON -DENABLE_KLEE_UCLIBC=ON -DENABLE_KLEE_LIBCXX=ON -DKLEE_LIBCXX_DIR=/usr/include/c++/9/libc++-install-90/ -DKLEE_LIBCXX_INCLUDE_DIR=/usr/include/c++/9/libc++-install-90/include/c++/v1/ -DENABLE_KLEE_EH_CXX=ON -DKLEE_LIBCXXABI_SRC_DIR=/usr/include/c++/9/llvm-90/libcxxabi/ -DKLEE_UCLIBC_PATH=/home/bigeast/klee-uclibc/ -DENABLE_UNIT_TESTS=ON -DENABLE_SYSTEM_TESTS=ON -DGTEST_SRC_DIR=/home/bigeast/googletest-release-1.7.0/  -DLLVM_CONFIG_BINARY=/usr/bin/llvm-config-9 -DLLVMCC=/usr/bin/clang-9 -DLLVMCXX=/usr/bin/clang++-9 -DCMAKE_C_COMPILER=clang-9  -DCMAKE_CXX_COMPILER=clang++-9  ..
-- The CXX compiler identification is Clang 9.0.0
-- The C compiler identification is Clang 9.0.0
-- Check for working CXX compiler: /usr/bin/clang++-9
-- Check for working CXX compiler: /usr/bin/clang++-9 -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Check for working C compiler: /usr/bin/clang-9
-- Check for working C compiler: /usr/bin/clang-9 -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- KLEE version 2.1
-- CMake generator: Unix Makefiles
-- CMAKE_BUILD_TYPE is not set. Setting default
-- The available build types are: Debug;Release;RelWithDebInfo;MinSizeRel
-- Build type: RelWithDebInfo
-- KLEE assertions enabled
-- LLVM_CONFIG_BINARY: /usr/bin/llvm-config-9
-- LLVM_PACKAGE_VERSION: "9.0.0"
-- LLVM_VERSION_MAJOR: "9"
-- LLVM_VERSION_MINOR: "0"
-- LLVM_VERSION_PATCH: "0"
-- LLVM_DEFINITIONS: "-D_GNU_SOURCE;-D__STDC_CONSTANT_MACROS;-D__STDC_FORMAT_MACROS;-D__STDC_LIMIT_MACROS"
-- LLVM_ENABLE_ASSERTIONS: "ON"
-- LLVM_ENABLE_EH: "OFF"
-- LLVM_ENABLE_RTTI: "ON"
-- LLVM_INCLUDE_DIRS: "/usr/lib/llvm-9/include"
-- LLVM_LIBRARY_DIRS: "/usr/lib/llvm-9/lib"
-- LLVM_TOOLS_BINARY_DIR: "/usr/lib/llvm-9/bin"
-- LLVM_ENABLE_VISIBILITY_INLINES_HIDDEN: "OFF"
-- TARGET_TRIPLE: "x86_64-pc-linux-gnu"
-- LLVM_BUILD_MAIN_SRC_DIR: "/usr/lib/llvm-9/build/"
-- Looking for bitcode compilers
-- Found /usr/bin/clang-9
-- Found /usr/bin/clang++-9
-- Testing bitcode compiler /usr/bin/clang-9
-- Compile success
-- Checking compatibility with LLVM 9.0.0
-- "/usr/bin/clang-9" is compatible
-- Testing bitcode compiler /usr/bin/clang++-9
-- Compile success
-- Checking compatibility with LLVM 9.0.0
-- "/usr/bin/clang++-9" is compatible
-- LLVMCC: /usr/bin/clang-9
-- LLVMCXX: /usr/bin/clang++-9
-- Performing Test HAS__Wall_CXX
-- Performing Test HAS__Wall_CXX - Success
-- C++ compiler supports -Wall
-- Performing Test HAS__Wextra_CXX
-- Performing Test HAS__Wextra_CXX - Success
-- C++ compiler supports -Wextra
-- Performing Test HAS__Wno_unused_parameter_CXX
-- Performing Test HAS__Wno_unused_parameter_CXX - Success
-- C++ compiler supports -Wno-unused-parameter
-- Performing Test HAS__Wall_C
-- Performing Test HAS__Wall_C - Success
-- C compiler supports -Wall
-- Performing Test HAS__Wextra_C
-- Performing Test HAS__Wextra_C - Success
-- C compiler supports -Wextra
-- Performing Test HAS__Wno_unused_parameter_C
-- Performing Test HAS__Wno_unused_parameter_C - Success
-- C compiler supports -Wno-unused-parameter
-- Not treating compiler warnings as errors
-- Could NOT find STP (missing: STP_DIR)
-- STP solver support disabled
-- Found Z3 libraries: "/usr/lib/libz3.so"
-- Found Z3 include path: "/usr/include"
-- Found Z3: /usr/include 
-- Z3 solver support enabled
-- Found Z3
-- Checking prototype Z3_get_error_msg for HAVE_Z3_GET_ERROR_MSG_NEEDS_CONTEXT - True
-- Z3_get_error_msg requires context
-- metaSMT solver support disabled
-- Performing Test HAS__fno_exceptions
-- Performing Test HAS__fno_exceptions - Success
-- C++ compiler supports -fno-exceptions
-- Found ZLIB: /usr/lib/x86_64-linux-gnu/libz.so (found version "1.2.11")
-- Zlib support enabled
-- TCMalloc support enabled
-- Looking for C++ include gperftools/malloc_extension.h
-- Looking for C++ include gperftools/malloc_extension.h - not found
CMake Error at CMakeLists.txt:419 (message):
  Can't find "gperftools/malloc_extension.h"
 
 
-- Configuring incomplete, errors occurred!
See also "/home/bigeast/klee-2.1/build/CMakeFiles/CMakeOutput.log".
See also "/home/bigeast/klee-2.1/build/CMakeFiles/CMakeError.log".

```

安装 libtcmalloc-minimal4 libgoogle-perftools-dev

```
sudo apt-get install libtcmalloc-minimal4 libgoogle-perftools-dev

```

再执行命令

```
bigeast@ubuntu:~/klee-2.1/build$ cmake \
> -DENABLE_POSIX_RUNTIME=ON \
> -DENABLE_KLEE_UCLIBC=ON \
> -DENABLE_KLEE_LIBCXX=ON \
> -DKLEE_LIBCXX_DIR=/usr/include/c++/9/libc++-install-90/ \
> -DKLEE_LIBCXX_INCLUDE_DIR=/usr/include/c++/9/libc++-install-90/include/c++/v1/ \
> -DENABLE_KLEE_EH_CXX=ON \
> -DKLEE_LIBCXXABI_SRC_DIR=/usr/include/c++/9/llvm-90/libcxxabi/ \
> -DKLEE_UCLIBC_PATH=/home/bigeast/klee-uclibc/ \
> -DENABLE_UNIT_TESTS=ON \
> -DENABLE_SYSTEM_TESTS=ON \
> -DGTEST_SRC_DIR=/home/bigeast/googletest-release-1.7.0/  \
> -DLLVM_CONFIG_BINARY=/usr/bin/llvm-config-9 \
> -DLLVMCC=/usr/bin/clang-9 \
> -DLLVMCXX=/usr/bin/clang++-9 \
> -DCMAKE_C_COMPILER=clang-9  \
> -DCMAKE_CXX_COMPILER=clang++-9  \
> ..
-- KLEE version 2.1
-- CMake generator: Unix Makefiles
-- Build type: RelWithDebInfo
-- KLEE assertions enabled
-- LLVM_CONFIG_BINARY: /usr/bin/llvm-config-9
-- LLVM_PACKAGE_VERSION: "9.0.0"
-- LLVM_VERSION_MAJOR: "9"
-- LLVM_VERSION_MINOR: "0"
-- LLVM_VERSION_PATCH: "0"
-- LLVM_DEFINITIONS: "-D_GNU_SOURCE;-D__STDC_CONSTANT_MACROS;-D__STDC_FORMAT_MACROS;-D__STDC_LIMIT_MACROS"
-- LLVM_ENABLE_ASSERTIONS: "ON"
-- LLVM_ENABLE_EH: "OFF"
-- LLVM_ENABLE_RTTI: "ON"
-- LLVM_INCLUDE_DIRS: "/usr/lib/llvm-9/include"
-- LLVM_LIBRARY_DIRS: "/usr/lib/llvm-9/lib"
-- LLVM_TOOLS_BINARY_DIR: "/usr/lib/llvm-9/bin"
-- LLVM_ENABLE_VISIBILITY_INLINES_HIDDEN: "OFF"
-- TARGET_TRIPLE: "x86_64-pc-linux-gnu"
-- LLVM_BUILD_MAIN_SRC_DIR: "/usr/lib/llvm-9/build/"
-- Looking for bitcode compilers
-- Found /usr/bin/clang-9
-- Found /usr/bin/clang++-9
-- Testing bitcode compiler /usr/bin/clang-9
-- Compile success
-- Checking compatibility with LLVM 9.0.0
-- "/usr/bin/clang-9" is compatible
-- Testing bitcode compiler /usr/bin/clang++-9
-- Compile success
-- Checking compatibility with LLVM 9.0.0
-- "/usr/bin/clang++-9" is compatible
-- LLVMCC: /usr/bin/clang-9
-- LLVMCXX: /usr/bin/clang++-9
-- C++ compiler supports -Wall
-- C++ compiler supports -Wextra
-- C++ compiler supports -Wno-unused-parameter
-- C compiler supports -Wall
-- C compiler supports -Wextra
-- C compiler supports -Wno-unused-parameter
-- Not treating compiler warnings as errors
-- Could NOT find STP (missing: STP_DIR)
-- STP solver support disabled
-- Found Z3 libraries: "/usr/lib/libz3.so"
-- Found Z3 include path: "/usr/include"
-- Z3 solver support enabled
-- Found Z3
-- Z3_get_error_msg requires context
-- metaSMT solver support disabled
-- C++ compiler supports -fno-exceptions
-- Zlib support enabled
-- TCMalloc support disabled
-- Could NOT find SQLITE3 (missing: SQLITE3_LIBRARY SQLITE3_INCLUDE_DIR)
CMake Error at CMakeLists.txt:434 (message):
  SQLite3 not found, please install
 
 
-- Configuring incomplete, errors occurred!
See also "/home/bigeast/klee-2.1/build/CMakeFiles/CMakeOutput.log".
See also "/home/bigeast/klee-2.1/build/CMakeFiles/CMakeError.log".

```

发现找不到 SQLite3，以下进行安装

```
sudo apt-get install sqlite3 libsqlite3-dev

```

把命令的路径全部检测一遍，发现上面的路径好多都写错了（因为是直接复制的别人的），再执行

```
cmake \
-DENABLE_POSIX_RUNTIME=ON \
-DENABLE_KLEE_UCLIBC=ON \
-DKLEE_LIBCXX_DIR=/usr/include/c++/9/libc++-install-9/  \
-DKLEE_LIBCXX_INCLUDE_DIR=/usr/include/c++/9/libc++-install-9/include/c++/v1/ \
-DENABLE_KLEE_EH_CXX=ON \
-DKLEE_LIBCXXABI_SRC_DIR=/usr/include/c++/9/libc++-9/libcxxabi/ \
-DKLEE_UCLIBC_PATH=/home/bigeast/klee-uclibc/ \
-DENABLE_UNIT_TESTS=ON \
-DENABLE_SYSTEM_TESTS=ON \
-DGTEST_SRC_DIR=/home/bigeast/googletest-release-1.7.0/  \
-DLLVM_CONFIG_BINARY=/usr/bin/llvm-config-9 \
-DLLVMCC=/usr/bin/clang-9 \
-DLLVMCXX=/usr/bin/clang++-9 \
-DCMAKE_C_COMPILER=clang-9  \
-DCMAKE_CXX_COMPILER=clang++-9  \
..

```

```
-- KLEE version 2.1
-- CMake generator: Unix Makefiles
-- Build type: RelWithDebInfo
-- KLEE assertions enabled
-- LLVM_CONFIG_BINARY: /usr/bin/llvm-config-9
-- LLVM_PACKAGE_VERSION: "9.0.0"
-- LLVM_VERSION_MAJOR: "9"
-- LLVM_VERSION_MINOR: "0"
-- LLVM_VERSION_PATCH: "0"
-- LLVM_DEFINITIONS: "-D_GNU_SOURCE;-D__STDC_CONSTANT_MACROS;-D__STDC_FORMAT_MACROS;-D__STDC_LIMIT_MACROS"
-- LLVM_ENABLE_ASSERTIONS: "ON"
-- LLVM_ENABLE_EH: "OFF"
-- LLVM_ENABLE_RTTI: "ON"
-- LLVM_INCLUDE_DIRS: "/usr/lib/llvm-9/include"
-- LLVM_LIBRARY_DIRS: "/usr/lib/llvm-9/lib"
-- LLVM_TOOLS_BINARY_DIR: "/usr/lib/llvm-9/bin"
-- LLVM_ENABLE_VISIBILITY_INLINES_HIDDEN: "OFF"
-- TARGET_TRIPLE: "x86_64-pc-linux-gnu"
-- LLVM_BUILD_MAIN_SRC_DIR: "/usr/lib/llvm-9/build/"
-- Looking for bitcode compilers
-- Found /usr/bin/clang-9
-- Found /usr/bin/clang++-9
-- Testing bitcode compiler /usr/bin/clang-9
-- Compile success
-- Checking compatibility with LLVM 9.0.0
-- "/usr/bin/clang-9" is compatible
-- Testing bitcode compiler /usr/bin/clang++-9
-- Compile success
-- Checking compatibility with LLVM 9.0.0
-- "/usr/bin/clang++-9" is compatible
-- LLVMCC: /usr/bin/clang-9
-- LLVMCXX: /usr/bin/clang++-9
-- C++ compiler supports -Wall
-- C++ compiler supports -Wextra
-- C++ compiler supports -Wno-unused-parameter
-- C compiler supports -Wall
-- C compiler supports -Wextra
-- C compiler supports -Wno-unused-parameter
-- Not treating compiler warnings as errors
-- Could NOT find STP (missing: STP_DIR)
-- STP solver support disabled
-- Found Z3 libraries: "/usr/lib/libz3.so"
-- Found Z3 include path: "/usr/include"
-- Z3 solver support enabled
-- Found Z3
-- Z3_get_error_msg requires context
-- metaSMT solver support disabled
-- C++ compiler supports -fno-exceptions
-- Zlib support enabled
-- TCMalloc support disabled
-- SELinux support disabled
-- Workaround for LLVM PR39177 (affecting LLVM 3.9 - 7.0.0) disabled
-- KLEE_RUNTIME_BUILD_TYPE: Debug+Asserts
-- POSIX runtime enabled
-- klee-uclibc support enabled
-- Found klee-uclibc library: "/home/bigeast/klee-uclibc/lib/libc.a"
-- klee-libcxx support enabled
-- Use libc++ include path: "/usr/include/c++/9/libc++-install-9/include/c++/v1/"
-- Found libc++ library: "/usr/include/c++/9/libc++-install-9/lib/libc++.bca"
-- CMAKE_CXX_FLAGS:  -Wall -Wextra -Wno-unused-parameter
-- KLEE_COMPONENT_EXTRA_INCLUDE_DIRS: '/usr/lib/llvm-9/include;/usr/include;/usr/include'
-- KLEE_COMPONENT_CXX_DEFINES: '-D_GNU_SOURCE;-D__STDC_CONSTANT_MACROS;-D__STDC_FORMAT_MACROS;-D__STDC_LIMIT_MACROS;-DKLEE_UCLIBC_BCA_'
-- KLEE_COMPONENT_CXX_FLAGS: '-fno-exceptions'
-- KLEE_COMPONENT_EXTRA_LIBRARIES: '/usr/lib/x86_64-linux-gnu/libz.so'
-- KLEE_SOLVER_LIBRARIES: '/usr/lib/libz3.so'
-- Testing is enabled
-- Using lit: /home/bigeast/.local/bin/lit
-- Unit tests enabled
-- Google Test: Building from source.
-- GTEST_SRC_DIR: /home/bigeast/googletest-release-1.7.0/
-- Found PythonInterp: /usr/bin/python (found version "2.7.17")
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Looking for pthread_create
-- Looking for pthread_create - not found
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE 
-- GTEST_INCLUDE_DIR: /home/bigeast/googletest-release-1.7.0/include
-- System tests enabled
CMake Deprecation Warning at test/CMakeLists.txt:133 (cmake_policy):
  The OLD behavior for policy CMP0026 will be removed from a future version
  of CMake.
 
  The cmake-policies(7) manual explains that the OLD behaviors of all
  policies are deprecated and that a policy should be set to OLD only under
  specific short-term circumstances.  Projects should be ported to the NEW
  behavior and not rely on setting a policy to OLD.
 
 
-- Could NOT find Doxygen (missing: DOXYGEN_EXECUTABLE)
CMake Warning at docs/CMakeLists.txt:46 (message):
  Doxygen not found.  Can't build Doxygen documentation
 
 
-- Configuring done
-- Generating done
CMake Warning:
  Manually-specified variables were not used by the project:
 
    KLEE_LIBCXXABI_SRC_DIR
 
 
-- Build files have been written to: /home/bigeast/klee-2.1/build

```

执行成功，构建 KLEE

```
make
sudo make install

```

在~/klee-2.1/examples/get_sign 路径下执行

```
clang-9 \
-I ../../include \
-emit-llvm -c \
-g \
-O0 -Xclang -disable-O0-optnone \
get_sign.c

```

其中，get_sign.c 源码如下：

```
/*
 * First KLEE tutorial: testing a small function
 */
 
#include "klee/klee.h"
 
int get_sign(int x) {
  if (x == 0)
     return 0;
 
  if (x < 0)
     return -1;
  else
     return 1;
}
 
int main() {
  int a;
  klee_make_symbolic(&a, sizeof(a), "a");
  return get_sign(a);
}

```

使用 KLEE 对 bitcode 进行符号执行

```
bigeast@ubuntu:~/klee-2.1/examples/get_sign$ klee get_sign.bc
KLEE: output directory is "/home/bigeast/klee-2.1/examples/get_sign/klee-out-0"
KLEE: Using Z3 solver backend
 
KLEE: done: total instructions = 33
KLEE: done: completed paths = 3
KLEE: done: generated tests = 3

```

可以看到 KLEE 成功探索了 3 条可能的执行路径，并将每条执行路径所对应的输入写入 klee-last 文件夹的 .ktest 文件中：

```
bigeast@ubuntu:~/klee-2.1/examples/get_sign$ ls klee-last
assembly.ll  messages.txt  run.stats         test000002.ktest  warnings.txt
info         run.istats    test000001.ktest  test000003.ktest

```

最后可以使用 ktest-tool 查看具体的输入值，以下依次查看三条路径的输入值

```
bigeast@ubuntu:~/klee-2.1/examples/get_sign$ ktest-tool klee-last/test000001.ktest
ktest file : 'klee-last/test000001.ktest'
args       : ['get_sign.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 4
object 0: data: b'\x00\x00\x00\x00'
object 0: hex : 0x00000000
object 0: int : 0
object 0: uint: 0
object 0: text: ....
bigeast@ubuntu:~/klee-2.1/examples/get_sign$ ktest-tool klee-last/test000002.ktest
ktest file : 'klee-last/test000002.ktest'
args       : ['get_sign.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 4
object 0: data: b'\xff\x00\x00\x00'
object 0: hex : 0xff000000
object 0: int : 255
object 0: uint: 255
object 0: text: ....
bigeast@ubuntu:~/klee-2.1/examples/get_sign$ ktest-tool klee-last/test000003.ktest
ktest file : 'klee-last/test000003.ktest'
args       : ['get_sign.bc']
num objects: 1
object 0: name: 'a'
object 0: size: 4
object 0: data: b'\x00\x00\x00\x80'
object 0: hex : 0x00000080
object 0: int : -2147483648
object 0: uint: 2147483648
object 0: text: ....

```

分别是大于 0，等于 0，小于 0 的值

 

参考资料：  
https://b33t1e.github.io/2018/01/30/klee%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/  
https://www.anquanke.com/post/id/240038  
https://github.com/klee/klee  
https://jywhy6.zone/2020/12/11/klee-notes/  
https://klee.github.io/build-llvm9/

[【公告】【iPhone 13、ipad、iWatch】11 月 15 日中午 12：00，看雪 · 众安 2021 KCTF 秋季赛 正式开赛【攻击篇】！！！文末有惊喜～](https://bbs.pediy.com/thread-270218.htm)

[#漏洞分析](forum-150-1-153.htm) [#自动化挖掘](forum-150-1-155.htm) [#Fuzz](forum-150-1-157.htm) [#Windows](forum-150-1-160.htm) [#Linux](forum-150-1-161.htm) [#Andorid](forum-150-1-162.htm)