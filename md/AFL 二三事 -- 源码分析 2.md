> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269536.htm)

> AFL 二三事 -- 源码分析 2

AFL 二三事 -- 源码分析 2
=================

目录

*   AFL 二三事 -- 源码分析 2
*            [前言](#前言)
*            一、LLVM 前置知识
*            二、 AFL 的 afl-clang-fast
*                    1. 概述
*                    2. 源码
*                            1. afl-clang-fast.c
*                                    1. main 函数
*                                    2. find_obj 函数
*                                    3. edit_params 函数
*                            2. afl-llvm-pass.so.cc
*                                    1. pass 注册
*                                    2. runOnModule 函数
*                                    3. afl-llvm-rt.o.c
*                                            1. deferred instrumentation
*                                            2. persistent mode
*                                            3. trace-pc-guard mode
*            [总结](#总结)
*            [参考链接](#参考链接)

前言
--

本文为《AFL 二三事》-- 源码分析系列的第二篇，主要阅读 AFL 的另外一种插桩方式 ——`llvm`模式。这主要是因为通过 `afl-gcc` 的方式进行插桩，在效率和性能上已经不能完美应对现代复杂的软件程序。随着 llvm 的成熟发展，AFL 提供了更好的插桩方式 `afl-clang-fast`，通过 llvm pass 来实现插桩，从而提升性能。

 

**当别人都要快的时候，你要慢下来。**

一、LLVM 前置知识
-----------

LLVM 主要为了解决编译时多种多样的前端和后端导致编译环境复杂、苛刻的问题，其核心为设计了一个称为 `LLVM IR` 的中间表示，并以库的形式提供一些列接口，以提供诸如操作 IR 、生成目标平台代码等等后端的功能。其整体架构如下所示：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_GXSEXM24ZC32HRF.jpg)

 

不同的前端和后端使用统一的中间代码`LLVM InterMediate Representation(LLVM IR)`，其结果就是如果需要支持一门新的编程语言，只需要实现一个新的前端；如果需要支持一款新的硬件设备，只需要实现一个新的后端；优化阶段为通用阶段，针对统一的 LLVM IR ，与新的编程语言和硬件设备无关。

 

GCC 的前后端耦合在一起，没有进行分离，所以 GCC 为了支持一门新的编程语言或一个新的硬件设备，需要重新开发前端到后端的完整过程。

 

Clang 是 LLVM 项目的一个子项目，它是 LLVM 架构下的 C/C++/Objective-C 的编译器，是 LLVM 前端的一部分。相较于 GCC，具备编译速度快、占用内存少、模块化设计、诊断信息可读性强、设计清晰简单等优点。

 

最终从源码到机器码的流程如下（以 Clang 做编译器为例）：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_WNCMH89CW5PFWMW.jpg)

 

（LLVM Pass 是一些中间过程处理 IR 的可以用户自定义的内容，可以用来遍历、修改 IR 以达到插桩、优化、静态分析等目的。）

 

代码首先由编译器前端 clang 处理后得到中间代码 IR，然后经过各 LLVM Pass 进行优化和转换，最终交给编译器后端生成机器码。

二、 AFL 的 afl-clang-fast
-----------------------

### 1. 概述

AFL 的 `llvm_mode` 可以实现编译器级别的插桩，可以替代 `afl-gcc` 或 `afl-clang` 使用的比较 “粗暴” 的汇编级别的重写的方法，且具备如下几个优势：

1.  编译器可以进行很多优化以提升效率；
2.  可以实现 CPU 无关，可以在非 x86 架构上进行 fuzz；
3.  可以更好地处理多线程目标。

在 AFL 的 `llvm_mode` 文件夹下包含 3 个文件： `afl-clang-fast.c` ，`afl-llvm-pass.so.cc`， `afl-llvm-rt.o.c`。

 

`afl-llvm-rt.o.c` 文件主要是重写了 `afl-as.h` 文件中的 `main_payload` 部分，方便调用；

 

`afl-llvm-pass.so.cc` 文件主要是当通过 `afl-clang-fast` 调用 clang 时，这个 pass 被插入到 LLVM 中，告诉编译器添加与 `` `afl-as.h`` 中大致等效的代码；

 

`afl-clang-fast.c` 文件本质上是 clang 的 wrapper，最终调用的还是 clang 。但是与 `afl-gcc` 一样，会进行一些参数处理。

 

`llvm_mode` 的插桩思路就是通过编写 pass 来实现信息记录，对每个基本块都插入探针，具体代码在 `afl-llvm-pass.so.cc` 文件中，初始化和 forkserver 操作通过链接完成。

### 2. 源码

#### 1. afl-clang-fast.c

##### 1. main 函数

`main` 函数的全部逻辑如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_YQJWD6C8S2EJ3NE.jpg)

 

主要是对 `find_obj(), edit_params(), execvp()` 函数的调用，

 

其中主要有以下三个函数的调用：

*   `find_obj(argv[0])`：查找运行时 library
*   `edit_params(argc, argv)`：处理传入的编译参数，将确定好的参数放入 `cc_params[]` 数组
*   `execvp(cc_params[0], (cahr**)cc_params)`：替换进程空间，传递参数，执行要调用的 clang

这里后两个函数的作用与 `afl-gcc.c` 中的作用基本相同，只是对参数的处理过程存在不同，不同的主要是 `find_obj()` 函数。

##### 2. find_obj 函数

`find_obj()`函数的控制流逻辑如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_SEXZVM7V4U4GU2E.jpg)

*   首先，读取环境变量 `AFL_PATH` 的值：
    *   如果读取成功，确认 `AFL_PATH/afl-llvm-rt.o` 是否可以访问；如果可以访问，设置该目录为 `obj_path` ，然后直接返回；
    *   如果读取失败，检查 `arg0` 中是否存在 `/` 字符，如果存在，则判断最后一个 `/` 前面的路径为 AFL 的根目录；然后读取`afl-llvm-rt.o`文件，成功读取，设置该目录为 `obj_path` ，然后直接返回。
*   如果上面两种方式都失败，到`/usr/local/lib/afl` 目录下查找是否存在 `afl-llvm-rt.o` ，如果存在，则设置为 `obj_path` 并直接返回（之所以向该路径下寻找，是因为默认的 AFL 的 MakeFile 在编译时，会定义一个名为`AFL_PATH`的宏，该宏会指向该路径）；
*   如果以上全部失败，抛出异常提示找不到 `afl-llvm-rt.o` 文件或 `afl-llvm-pass.so` 文件，并要求设置 `AFL_PATH` 环境变量 。

函数的主要功能是在寻找 AFL 的路径以找到 `afl-llvm-rt.o` 文件，该文件即为要用到的运行时库。

##### 3. edit_params 函数

该函数的主要作用仍然为编辑参数数组，其控制流程如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_HDTPGV2G8RZJ263.png)

*   首先，判断执行的是否为 `afl-clang-fast++` ：
    
    *   如果是，设置 `cc_params[0]` 为环境变量 `AFL_CXX`；如果环境变量为空，则设置为 `clang++` ；
    *   如果不是，设置 `cc_params[0]` 为环境变量 `AFL_CC`；如果环境变量为空，则设置为 `clang` ；
*   判断是否定义了 `USE_TRACE_PC` 宏，如果有，添加 `-fsanitize-coverage=trace-pc-guard -mllvm(only Android) -sanitizer-coverage-block-threshold=0(only Android)` 选项到参数数组；如果没有，依次将 `-Xclang -load -Xclang obj_path/afl-llvm-pass.so -Qunused-arguments` 选项添加到参数数组；（这里涉及到 llvm_mode 使用的 2 种插桩方式：默认使用的是传统模式，使用 `afl-llvm-pass.so` 注入来进行插桩，这种方式较为稳定；另外一种是处于实验阶段的方式——`trace-pc-guard` 模式，对于该模式的详细介绍可以参考 [llvm 相关文档——tracing-pcs-with-guards](https://clang.llvm.org/docs/SanitizerCoverage.html#tracing-pcs-with-guards)）
    
*   遍历传递给 `afl-clang-fast` 的参数，进行一定的检查和设置，并添加到 `cc_params` 数组：
    
    *   如果存在 `-m32` 或 `armv7a-linux-androideabi` ，设置 `bit_mode` 为 32；
    *   如果存在 `-m64` ，设置 `bit_mode` 为 64；
    *   如果存在 `-x` ，设置 `x_set` 为 1；
    *   如果存在 `-fsanitize=address` 或 `-fsanitize=memory`，设置 `asan_set` 为 1；
    *   如果存在 `-Wl,-z,defs` 或 `-Wl,--no-undefined`，则直接 pass 掉。
*   检查环境变量是否设置了 `AFL_HARDEN`：
    
    *   如果有，添加 `-fstack-protector-all` 选项；
    *   如果有且没有设置 `FORTIFY_SOURCE` ，添加 `-D_FORTIFY_SOURCE=2` 选项；
*   检查参数中是否存在 `-fsanitize=memory`，即 `asan_set` 为 0：
    
    *   如果没有，尝试读取环境变量 `AFL_USE_ASAN`，如果存在，添加 `-U_FORTIFY_SOURCE -fsanitize=address`；
    *   接下来对环境变量`AFL_USE_MSAN`的处理方式与 `AFL_USE_ASAN` 类似，添加的选项为 `-U_FORTIFY_SOURCE -fsanitize=memory`；
*   检查是否定义了 `USE_TRACE_PC` 宏，如果存在定义，检查是否存在环境变量 `AFL_INST_RATIO`，如果存在，抛出异常`AFL_INST_RATIO` 无法在 trace-pc 时使用；
    
*   检查环境变量 `AFL_NO_BUILTIN` ，如果没有设置，添加 `-g -O3 -funroll-loops`；
*   检查环境变量 `AFL_NO_BUILTIN`，如果进行了设置，添加 `-fno-builtin-strcmp -fno-builtin-strncmp -fno-builtin-strcasecmp -fno-builtin-strcasecmp -fno-builtin-memcmp`；
*   添加参数 `-D__AFL_HAVE_MANUAL_CONTROL=1 -D__AFL_COMPILER=1 -DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION=1`；
*   定义了两个宏 `__AFL_LOOP(), __AFL_INIT()`；
*   检查是否设置了 `x_set`， 如果有添加 `-x none`；
*   检查是否设置了宏 `__ANDORID__` ，如果没有，判断 `bit_mode` 的值：
    *   如果为 0，即没有`-m32`和`-m64`，添加 `obj_path/afl-llvm-rt.o` ；
    *   如果为 32，添加 `obj_path/afl-llvm-rt-32.o` ；
    *   如果为 64，添加 `obj_path/afl-llvm-rt-64.o` 。

#### 2. afl-llvm-pass.so.cc

`afl-llvm-pass.so.cc` 文件实现了 LLVM-mode 下的一个插桩 LLVM Pass。

 

本文不过多关心如何实现一个 LLVM Pass，重点分析该 pass 的实现逻辑。

 

该文件只有一个 Transform pass：`AFLCoverage`，继承自 `ModulePass`，实现了一个 `runOnModule` 函数，这也是我们需要重点分析的函数。

```
namespace {
 
  class AFLCoverage : public ModulePass {
 
    public:
 
      static char ID;
      AFLCoverage() : ModulePass(ID) { }
 
      bool runOnModule(Module &M) override;
 
      // StringRef getPassName() const override {
      //  return "American Fuzzy Lop Instrumentation";
      // }
 
  };
 
}

```

##### 1. pass 注册

对 pass 进行注册的部分源码如下：

```
static void registerAFLPass(const PassManagerBuilder &,
                            legacy::PassManagerBase &PM) {
 
  PM.add(new AFLCoverage());
 
}
 
 
static RegisterStandardPasses RegisterAFLPass(
    PassManagerBuilder::EP_ModuleOptimizerEarly, registerAFLPass);
 
static RegisterStandardPasses RegisterAFLPass0(
    PassManagerBuilder::EP_EnabledOnOptLevel0, registerAFLPass);

```

其核心功能为向 PassManager 注册新的 pass，每个 pass 相互独立。

 

对于 pass 注册的细节部分请读者自行研究 llvm 的相关内容。

##### 2. runOnModule 函数

该函数为该文件中的关键函数，其控制流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_5VD6MZF4S3YMZCH.jpg)

*   首先，通过 `getContext()` 来获取 `LLVMContext` ，获取进程上下文：
    
    ```
    LLVMContext &C = M.getContext();
     
    IntegerType *Int8Ty  = IntegerType::getInt8Ty(C);
    IntegerType *Int32Ty = IntegerType::getInt32Ty(C);
    
    ```
    
*   设置插桩密度：读取环境变量 `AFL_INST_RATIO` ，并赋值给 `inst_ratio`，其值默认为 100，范围为 1～100，该值表示插桩概率；
    
*   获取只想共享内存 shm 的指针以及上一个基本块的随机 ID：
    
    ```
    GlobalVariable *AFLMapPtr =
      new GlobalVariable(M, PointerType::get(Int8Ty, 0), false,
                         GlobalValue::ExternalLinkage, 0, "__afl_area_ptr");
     
    GlobalVariable *AFLPrevLoc = new GlobalVariable(
      M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_loc",
      0, GlobalVariable::GeneralDynamicTLSModel, 0, false);
    
    ```
    
*   进入插桩过程：
    
    *   通过 `for` 循环遍历每个 BB（基本块），寻找 BB 中适合插入桩代码的位置，然后通过初始化 `IRBuilder` 实例执行插入；
        
        ```
        BasicBlock::iterator IP = BB.getFirstInsertionPt();
              IRBuilder<> IRB(&(*IP));
        
        ```
        
    *   随机创建当前 BB 的 ID，然后插入 load 指令，获取前一个 BB 的 ID；
        
        ```
        if (AFL_R(100) >= inst_ratio) continue; // 如果大于插桩密度，进行随机插桩
         
        /* Make up cur_loc */
         
        unsigned int cur_loc = AFL_R(MAP_SIZE);
         
        ConstantInt *CurLoc = ConstantInt::get(Int32Ty, cur_loc);  // 随机创建当前基本块ID
         
        /* Load prev_loc */
         
        LoadInst *PrevLoc = IRB.CreateLoad(AFLPrevLoc);
        PrevLoc->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
        Value *PrevLocCasted = IRB.CreateZExt(PrevLoc, IRB.getInt32Ty()); // 获取上一个基本块的随机ID
        
        ```
        
    *   插入 load 指令，获取共享内存的地址，并调用 `CreateGEP` 函数获取共享内存中指定 index 的地址；
        
        ```
        /* Load SHM pointer */
         
        LoadInst *MapPtr = IRB.CreateLoad(AFLMapPtr);
        MapPtr->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
        Value *MapPtrIdx =
          IRB.CreateGEP(MapPtr, IRB.CreateXor(PrevLocCasted, CurLoc));
        
        ```
        
    *   插入 load 指令，获取对应 index 地址的值；插入 add 指令加一，然后创建 store 指令写入新值，并更新共享内存；
        
        ```
        /* Update bitmap */
         
        LoadInst *Counter = IRB.CreateLoad(MapPtrIdx);
        Counter->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
        Value *Incr = IRB.CreateAdd(Counter, ConstantInt::get(Int8Ty, 1));
        IRB.CreateStore(Incr, MapPtrIdx)
                ->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
        
        ```
        
    *   右移 `cur_loc` ，插入 store 指令，更新 `__afl_prev_loc`；
        
        ```
        /* Set prev_loc to cur_loc >> 1 */
         
        StoreInst *Store =
          IRB.CreateStore(ConstantInt::get(Int32Ty, cur_loc >> 1), AFLPrevLoc);
        Store->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
        
        ```
        
    *   最后对插桩计数加 1；
        
    *   扫描下一个 BB，根据设置是否为 quiet 模式等，并判断 `inst_blocks` 是否为 0，如果为 0 则说明没有进行插桩；
        
        ```
        if (!be_quiet) {
         
          if (!inst_blocks) WARNF("No instrumentation targets found.");
          else OKF("Instrumented %u locations (%s mode, ratio %u%%).",
                   inst_blocks, getenv("AFL_HARDEN") ? "hardened" :
                   ((getenv("AFL_USE_ASAN") || getenv("AFL_USE_MSAN")) ?
                    "ASAN/MSAN" : "non-hardened"), inst_ratio);
        }
        
        ```
        

整个插桩过程较为清晰，没有冗余动作和代码。

##### 3. afl-llvm-rt.o.c

该文件主要实现了 llvm_mode 的 3 个特殊功能：`deferred instrumentation, persistent mode,trace-pc-guard mode`。

###### 1. deferred instrumentation

 

AFL 会尝试通过只执行一次目标二进制文件来提升性能，在 `main()` 之前暂停程序，然后克隆 “主” 进程获得一个稳定的可进行持续 fuzz 的目标。简言之，避免目标二进制文件的多次、重复的完整运行，而是采取了一种类似快照的机制。

 

虽然这种机制可以减少程序运行在操作系统、链接器和 libc 级别的消耗，但是在面对大型配置文件的解析时，优势并不明显。

 

在这种情况下，可以将 `forkserver` 的初始化放在大部分初始化工作完成之后、二进制文件解析之前来进行，这在某些情况下可以提升 10 倍以上的性能。我们把这种方式称为 LLVM 模式下的 `deferred instrumentation`。

 

首先，在代码中找到可以进行延迟克隆的合适位置。 这需要_极端_小心地完成，以避免破坏二进制文件。 特别是，如果您在以下情况下选择一个位置，程序可能会出现故障：

 

首先，在代码中寻找可以进行延迟克隆的合适的、不会破坏原二进制文件的位置，然后添加如下代码：

```
#ifdef __AFL_HAVE_MANUAL_CONTROL
    __AFL_INIT();
#endif

```

以上代码插入，在 `afl-clang-fast.c` 文件中有说明：

```
  cc_params[cc_par_cnt++] = "-D__AFL_INIT()="
    "do { static volatile char *_A __attribute__((used)); "
    " _A = (char*)\"" DEFER_SIG "\"; "
#ifdef __APPLE__
    "__attribute__((visibility(\"default\"))) "
    "void _I(void) __asm__(\"___afl_manual_init\"); "
#else
    "__attribute__((visibility(\"default\"))) "
    "void _I(void) __asm__(\"__afl_manual_init\"); "
#endif /* ^__APPLE__ */

```

`__afl_manual_init()` 函数实现如下：

```
/* This one can be called from user code when deferred forkserver mode
    is enabled. */
 
void __afl_manual_init(void) {
 
  static u8 init_done;
 
  if (!init_done) {
 
    __afl_map_shm();
    __afl_start_forkserver();
    init_done = 1;
 
  }
 
}

```

首先，判断是否进行了初始化，没有则调用 `__afl_map_shm()` 函数进行共享内存初始化。 `__afl_map_shm()` 函数如下：

```
/* SHM setup. */
 
static void __afl_map_shm(void) {
 
  u8 *id_str = getenv(SHM_ENV_VAR); // 读取环境变量 SHM_ENV_VAR 获取id
 
  if (id_str) { // 成功读取id
 
    u32 shm_id = atoi(id_str);
 
    __afl_area_ptr = shmat(shm_id, NULL, 0); // 获取shm地址，赋给 __afl_area_ptr
 
    /* Whooooops. */
 
    if (__afl_area_ptr == (void *)-1) _exit(1);  // 异常则退出
 
    /* Write something into the bitmap so that even with low AFL_INST_RATIO,
       our parent doesn't give up on us. */
 
    __afl_area_ptr[0] = 1; // 进行设置
 
  }
 
}

```

然后，调用 `__afl_start_forkserver()` 函数开始执行 forkserver：

```
/* Fork server logic. */
 
static void __afl_start_forkserver(void) {
 
  static u8 tmp[4];
  s32 child_pid;
 
  u8  child_stopped = 0;
 
  /* Phone home and tell the parent that we're OK. If parent isn't there,
     assume we're not running in forkserver mode and just execute program. */
 
  if (write(FORKSRV_FD + 1, tmp, 4) != 4) return; // 写入4字节到状态管道，通知 fuzzer已准备完成
 
  while (1) {
 
    u32 was_killed;
    int status;
 
    /* Wait for parent by reading from the pipe. Abort if read fails. */
 
    if (read(FORKSRV_FD, &was_killed, 4) != 4) _exit(1);
 
    /* If we stopped the child in persistent mode, but there was a race
       condition and afl-fuzz already issued SIGKILL, write off the old
       process. */
 
      // 处于persistent mode且子进程已被killed
    if (child_stopped && was_killed) {
      child_stopped = 0;
      if (waitpid(child_pid, &status, 0) < 0) _exit(1);
    }
 
    if (!child_stopped) {
 
      /* Once woken up, create a clone of our process. */
 
      child_pid = fork(); // 重新fork
      if (child_pid < 0) _exit(1);
 
      /* In child process: close fds, resume execution. */
 
      if (!child_pid) {
 
        close(FORKSRV_FD); // 关闭fd，
        close(FORKSRV_FD + 1);
        return;
 
      }
 
    } else {
 
      /* Special handling for persistent mode: if the child is alive but
         currently stopped, simply restart it with SIGCONT. */
 
      // 子进程只是暂停，则进行重启
      kill(child_pid, SIGCONT);
      child_stopped = 0;
 
    }
 
    /* In parent process: write PID to pipe, then wait for child. */
 
    if (write(FORKSRV_FD + 1, &child_pid, 4) != 4) _exit(1);
 
    if (waitpid(child_pid, &status, is_persistent ? WUNTRACED : 0) < 0)
      _exit(1);
 
    /* In persistent mode, the child stops itself with SIGSTOP to indicate
       a successful run. In this case, we want to wake it up without forking
       again. */
 
    if (WIFSTOPPED(status)) child_stopped = 1;
 
    /* Relay wait status to pipe, then loop back. */
 
    if (write(FORKSRV_FD + 1, &status, 4) != 4) _exit(1);
 
  }
 
}

```

上述逻辑可以概括如下：

*   首先，设置 `child_stopped = 0`，写入 4 字节到状态管道，通知 fuzzer 已准备完成；
    
*   进入 `while` ，开启 fuzz 循环：
    
    *   调用 `read` 从控制管道读取 4 字节，判断子进程是否超时。如果管道内读取失败，发生阻塞，读取成功则表示 AFL 指示 forkserver 执行 fuzz；
    *   如果 `child_stopped` 为 0，则 fork 出一个子进程执行 fuzz，关闭和控制管道和状态管道相关的 fd，跳出 fuzz 循环；
        
    *   如果 `child_stopped` 为 1，在 `persistent mode` 下进行的特殊处理，此时子进程还活着，只是被暂停了，可以通过`kill(child_pid, SIGCONT)`来简单的重启，然后设置`child_stopped`为 0；
        
    *   forkserver 向状态管道 `FORKSRV_FD + 1` 写入子进程的 pid，然后等待子进程结束；
    *   `WIFSTOPPED(status)` 宏确定返回值是否对应于一个暂停子进程，因为在 `persistent mode` 里子进程会通过 `SIGSTOP` 信号来暂停自己，并以此指示运行成功，我们需要通过 `SIGCONT`信号来唤醒子进程继续执行，不需要再进行一次 fuzz，设置`child_stopped`为 1；
    *   子进程结束后，向状态管道 `FORKSRV_FD + 1` 写入 4 个字节，通知 AFL 本次执行结束。

###### 2. persistent mode

 

`persistent mode` 并没有通过 fork 子进程的方式来执行 fuzz。一些库中提供的 API 是无状态的，或者可以在处理不同输入文件之间进行重置，恢复到之前的状态。执行此类重置时，可以使用一个长期存活的进程来测试多个用例，以这种方式来减少重复的 `fork()` 调用和操作系统的开销。不得不说，这种思路真的很优秀。

 

一个基础的框架大概如下：

```
while (__AFL_LOOP(1000)) {
 
  /* Read input data. */
  /* Call library code to be fuzzed. */
  /* Reset state. */
 
}
 
/* Exit normally */

```

设置一个 `while` 循环，并指定循环次数。在每次循环内，首先读取数据，然后调用想 fuzz 的库代码，然后重置状态，继续循环。（本质上也是一种快照。）

 

对于循环次数的设置，循环次数控制了 AFL 从头重新启动过程之前的最大迭代次数，较小的循环次数可以降低内存泄漏类故障的影响，官方建议的数值为 1000。（循环次数设置过高可能出现较多意料之外的问题，并不建议设置过高。）

 

一个 `persistent mode` 的样例程序如下：

```
#include #include #include #include #include /* Main entry point. */
 
int main(int argc, char** argv) {
 
  char buf[100]; /* Example-only buffer, you'd replace it with other global or
                    local variables appropriate for your use case. */
 
  while (__AFL_LOOP(1000)) {
 
    /*** PLACEHOLDER CODE ***/
 
    /* STEP 1: 初始化所有变量 */
 
    memset(buf, 0, 100);
 
    /* STEP 2: 读取输入数据，从文件读入时需要先关闭旧的fd然后重新打开文件*/
 
    read(0, buf, 100);
 
    /* STEP 3: 调用待fuzz的code*/
 
    if (buf[0] == 'f') {
      printf("one\n");
      if (buf[1] == 'o') {
        printf("two\n");
        if (buf[2] == 'o') {
          printf("three\n");
          if (buf[3] == '!') {
            printf("four\n");
            abort();
          }
        }
      }
    }
 
    /*** END PLACEHOLDER CODE ***/
 
  }
 
  /* 循环结束，正常结束。AFL会重启进程，并清理内存、剩余fd等 */
 
  return 0;
 
} 
```

宏定义 `__AFL_LOOP` 内部调用 `__afl_persistent_loop` 函数：

```
  cc_params[cc_par_cnt++] = "-D__AFL_LOOP(_A)="
    "({ static volatile char *_B __attribute__((used)); "
    " _B = (char*)\"" PERSIST_SIG "\"; "
#ifdef __APPLE__
    "__attribute__((visibility(\"default\"))) "
    "int _L(unsigned int) __asm__(\"___afl_persistent_loop\"); "
#else
    "__attribute__((visibility(\"default\"))) "
    "int _L(unsigned int) __asm__(\"__afl_persistent_loop\"); "
#endif /* ^__APPLE__ */
    "_L(_A); })";

```

`__afl_persistent_loop(unsigned int max_cnt)` 的逻辑如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_MPXCKYQ45JT8E9Y.jpg)

 

结合源码梳理一下其逻辑：

```
/* A simplified persistent mode handler, used as explained in README.llvm. */
 
int __afl_persistent_loop(unsigned int max_cnt) {
 
  static u8  first_pass = 1;
  static u32 cycle_cnt;
 
  if (first_pass) {
 
    if (is_persistent) {
 
      memset(__afl_area_ptr, 0, MAP_SIZE);
      __afl_area_ptr[0] = 1;
      __afl_prev_loc = 0;
    }
 
    cycle_cnt  = max_cnt;
    first_pass = 0;
    return 1;
 
  }
 
  if (is_persistent) {
 
    if (--cycle_cnt) {
 
      raise(SIGSTOP);
 
      __afl_area_ptr[0] = 1;
      __afl_prev_loc = 0;
 
      return 1;
 
    } else {
 
      __afl_area_ptr = __afl_area_initial;
 
    }
 
  }
 
  return 0;
 
}

```

*   首先判读是否为第一次执行循环，如果是第一次：
    
    *   如果 `is_persistent` 为 1，清空 `__afl_area_ptr`，设置 `__afl_area_ptr[0]` 为 1，`__afl_prev_loc` 为 0；
    *   设置 `cycle_cnt` 的值为传入的 `max_cnt` 参数，然后设置 `first_pass=0` 表示初次循环结束，返回 1；
*   如果不是第一次执行循环，在 persistent mode 下，且 `--cycle_cnt` 大于 1：
    
    *   发出信号 `SIGSTOP` 让当前进程暂停
    *   设置 `__afl_area_ptr[0]` 为 1，`__afl_prev_loc` 为 0，然后直接返回 1
        
    *   如果 `cycle_cnt` 为 0，设置`__afl_area_ptr`指向数组 `__afl_area_initial`。
        
*   最后返回 0
    

重新总结一下上面的逻辑：

*   第一次执行 loop 循环，进行初始化，然后返回 1，此时满足 `while(__AFL_LOOP(1000)`， 于是执行一次 fuzz，计数器 cnt 减 1，抛出 SIGSTOP 信号暂停子进程；
*   第二次执行 loop 循环，恢复之前暂停的子进程继续执行，并设置 `child_stopped` 为 0。此时相当于重新执行了一次程序，重新对 `__afl_prev_loc` 进行设置，随后返回 1，再次进入 `while(_AFL_LOOP(1000))` ，执行一次 fuzz，计数器 cnt 减 1，抛出 SIGSTOP 信号暂停子进程；
*   第 1000 次执行，计数器 cnt 此时为 0，不再暂停子进程，令 `__afl_area_ptr` 指向无关数组 `__afl_area_initial` ，随后子进程结束。

###### 3. trace-pc-guard mode

 

该功能的使用需要设置宏 `AFL_TRACE_PC=1` ，然后再执行 `afl-clang-fast` 时传入参数 `-fsanitize-coverage=trace-pc-guard` 。

 

该功能的主要特点是会在每个 edge 插入桩代码，函数 `__sanitizer_cov_trace_pc_guard` 会在每个 edge 进行调用，该函数利用函数参数 `guard` 指针所指向的 `uint32` 值来确定共享内存上所对应的地址：

```
void __sanitizer_cov_trace_pc_guard(uint32_t* guard) {
  __afl_area_ptr[*guard]++;
}

```

`guard` 的初始化位于函数 `__sanitizer_cov_trace_pc_guard_init` 中：

```
void __sanitizer_cov_trace_pc_guard_init(uint32_t* start, uint32_t* stop) {
 
  u32 inst_ratio = 100;
  u8* x;
 
  if (start == stop || *start) return;
 
  x = getenv("AFL_INST_RATIO");
  if (x) inst_ratio = atoi(x);
 
  if (!inst_ratio || inst_ratio > 100) {
    fprintf(stderr, "[-] ERROR: Invalid AFL_INST_RATIO (must be 1-100).\n");
    abort();
  }
 
  *(start++) = R(MAP_SIZE - 1) + 1;
 
  while (start < stop) { // 这里如果计算stop-start，就是程序里总计的edge数
 
    if (R(100) < inst_ratio) *start = R(MAP_SIZE - 1) + 1;
    else *start = 0;
 
    start++;
 
  }
 
}

```

总结
--

其实在这里最主要的还是 `persistent mode` 中的逻辑，其他的两个模式对于实际的 fuzz 工作已经或尚且没有太大的指导意义。但是研究一下其源码实现，可以发现 AFL 的设计和开发真的很巧妙，其中很多思路值得开发人员和安全人员学习和借鉴。

参考链接
----

1.  https://eternalsakura13.com/2020/08/23/afl/
2.  https://bbs.pediy.com/thread-266025.htm

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 1 天前 被有毒编辑 ，原因：

[#自动化挖掘](forum-150-1-155.htm) [#Fuzz](forum-150-1-157.htm) [#Linux](forum-150-1-161.htm)