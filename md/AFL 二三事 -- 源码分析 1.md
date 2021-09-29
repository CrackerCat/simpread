> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269534.htm)

> AFL 二三事 -- 源码分析 1

AFL 二三事 —— 源码分析 1
=================

目录

*   AFL 二三事 —— 源码分析 1
*            [前言](#前言)
*            [宏观](#宏观)
*            一、AFL 的 gcc —— afl-gcc.c
*                    1. 概述
*                    2. 源码
*                            1. 关键变量
*                            2. main 函数
*                            3. find_as 函数
*                            4. edit_params 函数
*            二、AFL 的源码插桩 —— afl-as.c
*                    1. 概述
*                    2. 源码
*                            1. 关键变量
*                            2. main 函数
*                            3. add_instrumentation 函数
*                            4. edit_params 函数
*                    3. instrumentation trampoline 和 main_payload
*                            1. trampoline_fmt_64/32
*                            2. __afl_maybe_log
*                            3. __afl_setup
*                            4. __afl_setup_first
*                            5. __afl_forkserver
*                            6. __afl_fork_wait_loop
*                            7. __afl_fork_resume
*                            8. __afl_store
*            [三、总结](#三、总结)
*            [参考文献：](#参考文献：)

前言
--

深入分析 AFL 源码，对理解 AFL 的设计理念和其中用到的技巧有着巨大的帮助，对于后期进行定制化 Fuzzer 开发也具有深刻的指导意义。所以，阅读 AFL 源码是学习 AFL 必不可少的一个关键步骤。

 

考虑到 AFL 源码规模，源码分析部分将分为几期进行。

 

**当别人都要快的时候，你要慢下来。**

宏观
--

首先在宏观上看一下 AFL 的源码结构：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_8QXGCAXH83KUXSV.jpg)

 

主要的代码在 `afl-fuzz.c` 文件中，然后是几个独立功能的实现代码，`llvm_mode` 和 `qemu_mode` 的代码量大致相当，所以分析的重点应该还是在 AFL 的根目录下的几个核心功能的实现上，尤其是 `afl-fuzz.c`，属于核心中的重点。

 

各个模块的主要功能和作用：

*   **插桩模块**
    
    1.  `afl-as.h, afl-as.c, afl-gcc.c`：一般插桩模式，针对源码插桩，编译器可以使用 gcc， clang；
    2.  `llvm_mode`：llvm 插桩模式，针对源码插桩，编译器使用 clang；
    3.  `qemu_mode`：qemu 插桩模式，针对二进制文件插桩。
*   fuzzer 模块
    
    `afl-fuzz.c`：fuzzer 实现的核心代码，AFL 的主体。
    
*   其他辅助模块
    
    1.  `afl-analyze`：对测试用例进行分析，通过分析给定的用例，确定是否可以发现用例中有意义的字段；
    2.  `afl-plot`：生成测试任务的状态图；
    3.  `afl-tmin`：对测试用例进行最小化；
    4.  `afl-cmin`：对语料库进行精简操作；
    5.  `afl-showmap`：对单个测试用例进行执行路径跟踪；
    6.  `afl-whatsup`：各并行例程 fuzzing 结果统计；
    7.  `afl-gotcpu`：查看当前 CPU 状态。
*   部分头文件说明
    
    1.  `alloc-inl.h`：定义带检测功能的内存分配和释放操作；
    2.  `config.h`：定义配置信息；
    3.  `debug.h`：与提示信息相关的宏定义；
    4.  `hash.h`：哈希函数的实现定义；
    5.  `types.h`：部分类型及宏的定义。

一、AFL 的 gcc —— afl-gcc.c
------------------------

### 1. 概述

`afl-gcc` 是 GCC 或 clang 的一个 wrapper（封装），常规的使用方法是在调用 `./configure` 时通过 `CC` 将路径传递给 `afl-gcc` 或 `afl-clang`。（对于 C++ 代码，使用 `CXX` 并将其指向 `afl-g++` / `afl-clang++`。）`afl-clang`, `afl-clang++`， `afl-g++` 均为指向 `afl-gcc` 的一个符号链接。

 

`afl-gcc` 的主要作用是实现对于关键节点的代码插桩，属于汇编级，从而记录程序执行路径之类的关键信息，对程序的运行情况进行反馈。

### 2. 源码

#### 1. 关键变量

在开始函数代码分析前，首先要明确几个关键变量：

```
static u8*  as_path;                /* Path to the AFL 'as' wrapper，AFL的as的路径      */
static u8** cc_params;              /* Parameters passed to the real CC，CC实际使用的编译器参数 */
static u32  cc_par_cnt = 1;         /* Param count, including argv0 ，参数计数 */
static u8   be_quiet,               /* Quiet mode，静默模式      */
            clang_mode;             /* Invoked as afl-clang*? ，是否使用afl-clang*模式 */
 
# 数据类型说明
# typedef uint8_t  u8;
# typedef uint16_t u16;
# typedef uint32_t u32;

```

#### 2. main 函数

main 函数全部逻辑如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_WK6NXBU63ESFT5B.jpg)

 

其中主要有如下三个函数的调用：

*   `find_as(argv[0])` ：查找使用的汇编器
*   `edit_params(argc, argv)`：处理传入的编译参数，将确定好的参数放入 `cc_params[]` 数组
*   调用 `execvp(cc_params[0], (cahr**)cc_params)` 执行 `afl-gcc`

![](https://bbs.pediy.com/upload/attach/202109/779730_KMXHBXPMNTPCUEX.jpg)

 

这里添加了部分代码打印出传入的参数 arg[0] - arg[7] ，其中一部分是我们指定的参数，另外一部分是自动添加的编译选项（之前的[原理](https://www.v4ler1an.com/afl%E4%BA%8C%E4%B8%89%E4%BA%8B2/)文章的插桩部分有简单介绍）。

#### 3. find_as 函数

函数的核心作用：寻找 `afl-as`

> / _Try to find our "fake" GNU assembler in AFL_PATH or at the location derived from argv[0]. If that fails, abort._ /

 

函数内部大概的流程如下（软件自动生成，控制流程图存在误差，但关键逻辑没有问题）：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_V9BWQ8W4PTN8XXC.jpg)

1.  首先检查环境变量 `AFL_PATH` ，如果存在直接赋值给 `afl_path` ，然后检查 `afl_path/as` 文件是否可以访问，如果可以，`as_path = afl_path`。
2.  如果不存在环境变量 `AFL_PATH` ，检查 `argv[0]` （如 “/Users/v4ler1an/AFL/afl-gcc”）中是否存在 "/" ，如果存在则取最后 “/” 前面的字符串作为 `dir`，然后检查 `dir/afl-as` 是否可以访问，如果可以，将 `as_path = dir` 。
3.  以上两种方式都失败，抛出异常。

#### 4. edit_params 函数

核心作用：将 `argv` 拷贝到 `u8 **cc_params`，然后进行相应的处理。

> / _Copy argv to cc_params, making the necessary edits._ /

 

函数内部的大概流程如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_H7WCZRTR9BR4AYG.jpg)

1.  调用 `ch_alloc()` 为 `cc_params` 分配大小为 `(argc + 128) * 8` 的内存（u8 的类型为 1byte 无符号整数）
    
2.  检查 `argv[0]` 中是否存在`/`，如果不存在则 `name = argv[0]`，如果存在则一直找到最后一个`/`，并将其后面的字符串赋值给 `name`
    
3.  对比 `name`和固定字符串`afl-clang`：
    
    1.  若相同，设置`clang_mode = 1`，设置环境变量`CLANG_ENV_VAR`为 1
        
        1.  对比`name`和固定字符串`afl-clang++`:：
            1.  若相同，则获取环境变量`AFL_CXX`的值，如果存在，则将该值赋值给`cc_params[0]`，否则将`afl-clang++`赋值给`cc_params[0]`。这里的`cc_params`为保存编译参数的数组；
            2.  若不相同，则获取环境变量`AFL_CC`的值，如果存在，则将该值赋值给`cc_params[0]`，否则将`afl-clang`赋值给`cc_params[0]`。
    2.  如果不相同，并且是 Apple 平台，会进入 `#ifdef __APPLE__`。在 Apple 平台下，开始对 `name` 进行对比，并通过 `cc_params[0] = getenv("")` 对`cc_params[0]`进行赋值；如果是非 Apple 平台，对比 `name` 和 固定字符串`afl-g++`（此处忽略对 Java 环境的处理过程）：
        
        1.  若相同，则获取环境变量`AFL_CXX`的值，如果存在，则将该值赋值给`cc_params[0]`，否则将`g++`赋值给`cc_params[0]`；
            
        2.  若不相同，则获取环境变量`AFL_CC`的值，如果存在，则将该值赋值给`cc_params[0]`，否则将`gcc`赋值给`cc_params[0]`。
            
4.  进入 while 循环，遍历从`argv[1]`开始的`argv`参数：
    
    *   如果扫描到 `-B` ，`-B`选项用于设置编译器的搜索路径，直接跳过。（因为在这之前已经处理过`as_path`了）；
        
    *   如果扫描到 `-integrated-as`，跳过；
        
    *   如果扫描到 `-pipe`，跳过；
        
    *   如果扫描到 `-fsanitize=address` 和 `-fsanitize=memory` 告诉 gcc 检查内存访问的错误，比如数组越界之类，设置 `asan_set = 1；`
        
    *   如果扫描到 `FORTIFY_SOURCE` ，设置 `fortify_set = 1` 。`FORTIFY_SOURCE` 主要进行缓冲区溢出问题的检查，检查的常见函数有`memcpy, mempcpy, memmove, memset, strcpy, stpcpy, strncpy, strcat, strncat, sprintf, vsprintf, snprintf, gets` 等；
        
    *   对 `cc_params` 进行赋值：`cc_params[cc_par_cnt++] = cur;`
        
5.  跳出 `while` 循环，设置其他参数：
    
    1.  取出前面计算出的 `as_path` ，设置 `-B as_path` ；
6.  如果为 `clang_mode` ，则设置`-no-integrated-as`；
    1.  如果存在环境变量 `AFL_HARDEN`，则设置`-fstack-protector-all`。且如果没有设置 `fortify_set` ，追加 `-D_FORTIFY_SOURCE=2` ；
7.  sanitizer 相关，通过多个 if 进行判断：
    
    *   如果 `asan_set` 在前面被设置为 1，则设置环境变量 `AFL_USE_ASAN` 为 1；
        *   如果 `asan_set` 不为 1 且，存在 `AFL_USE_ASAN` 环境变量，则设置`-U_FORTIFY_SOURCE -fsanitize=address`；
    *   如果不存在 `AFL_USE_ASAN` 环境变量，但存在 `AFL_USE_MSAN` 环境变量，则设置`-fsanitize=memory`（不能同时指定`AFL_USE_ASAN`或者`AFL_USE_MSAN`，也不能同时指定 `AFL_USE_MSAN` 和 `AFL_HARDEN`，因为这样运行时速度过慢；
        
        *   如果不存在 `AFL_DONT_OPTIMIZE` 环境变量，则设置`-g -O3 -funroll-loops -D__AFL_COMPILER=1 -DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION=1`；
        *   如果存在 `AFL_NO_BUILTIN` 环境变量，则表示允许进行优化，设置`-fno-builtin-strcmp -fno-builtin-strncmp -fno-builtin-strcasecmp -fno-builtin-strncasecmp -fno-builtin-memcmp -fno-builtin-strstr -fno-builtin-strcasestr`。
8.  最后补充`cc_params[cc_par_cnt] = NULL;`，`cc_params` 参数数组编辑完成。

二、AFL 的源码插桩 —— afl-as.c
-----------------------

### 1. 概述

`afl-gcc` 是 GNU as 的一个 wrapper（封装），唯一目的是预处理由 GCC/clang 生成的汇编文件，并注入包含在 `afl-as.h` 中的插桩代码。 使用 `afl-gcc / afl-clang` 编译程序时，工具链会自动调用它。该 wapper 的目标并不是为了实现向 `.s` 或 `asm` 代码块中插入手写的代码。

 

`experiment/clang_asm_normalize/` 中可以找到可能允许 clang 用户进行手动插入自定义代码的解决方案，GCC 并不能实现该功能。

### 2. 源码

#### 1. 关键变量

在开始函数代码分析前，首先要明确几个关键变量：

```
static u8** as_params;          /* Parameters passed to the real 'as'，传递给as的参数   */
 
static u8*  input_file;         /* Originally specified input file ，输入文件     */
static u8*  modified_file;      /* Instrumented file for the real 'as'，as进行插桩处理的文件  */
 
static u8   be_quiet,           /* Quiet mode (no stderr output) ，静默模式，没有标准输出       */
            clang_mode,         /* Running in clang mode?    是否运行在clang模式           */
            pass_thru,          /* Just pass data through?   只通过数据           */
            just_version,       /* Just show version?        只显示版本   */
            sanitizer;          /* Using ASAN / MSAN         是否使用ASAN/MSAN           */
 
static u32  inst_ratio = 100,   /* Instrumentation probability (%)  插桩覆盖率    */
            as_par_cnt = 1;     /* Number of params to 'as'    传递给as的参数数量初始值         */

```

注：如果在参数中没有指明 `--m32` 或 `--m64` ，则默认使用在编译时使用的选项。

#### 2. main 函数

main 函数全部逻辑如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_XVS2U66V77WGBEA.jpg)

1.  首先获取环境变量 `AFL_INST_RATIO` ，赋值给 `inst_ratio_str`，该环境变量主要控制检测每个分支的概率，取值为 0 到 100%，设置为 0 时则只检测函数入口的跳转，而不会检测函数分支的跳转；
2.  通过 `gettimeofday(&tv,&tz);`获取时区和时间，然后设置 `srandom()` 的随机种子 `rand_seed = tv.tv_sec ^ tv.tv_usec ^ getpid();`
3.  调用 `edit_params(argc, argv)` 函数进行参数处理；
4.  检测 `inst_ratio_str` 的值是否合法范围内，并设置环境变量 `AFL_LOOP_ENV_VAR`；
5.  读取环境变量`` `AFL_USE_ASAN``和`AFL_USE_MSAN`的值，如果其中有一个为 1，则设置`sanitizer`为 1，且将`inst_ratio`除 3。这是因为在进行 ASAN 的编译时，AFL 无法识别出 ASAN 特定的分支，导致插入很多无意义的桩代码，所以直接暴力地将插桩概率 / 3；
6.  调用 `add_instrumentation()` 函数，这是实际的插桩函数；
7.  fork 一个子进程来执行 `execvp(as_params[0], (char**)as_params);`。这里采用的是 fork 一个子进程的方式来执行插桩。这其实是因为我们的 `execvp` 执行的时候，会用 `as_params[0]` 来完全替换掉当前进程空间中的程序，如果不通过子进程来执行实际的 `as`，那么后续就无法在执行完实际的 as 之后，还能 unlink 掉 modified_file；
8.  调用 `waitpid(pid, &status, 0)` 等待子进程执行结束；
9.  读取环境变量 `AFL_KEEP_ASSEMBLY` 的值，如果没有设置这个环境变量，就 unlink 掉 `modified_file`(已插完桩的文件)。设置该环境变量主要是为了防止 `afl-as` 删掉插桩后的汇编文件，设置为 1 则会保留插桩后的汇编文件。

可以通过在 main 函数中添加如下代码来打印实际执行的参数：

```
print("\n");
 
for (int i = 0; i < sizeof(as_params); i++){
  peinrf("as_params[%d]:%s\n", i, as_params[i]);
 
}

```

![](https://bbs.pediy.com/upload/attach/202109/779730_QCTJBRB3YKRG2G6.jpg)

 

在插桩完成后，会生成 `.s` 文件，内容如下（具体的文件位置与设置的环境变量相关）：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_DH2YTFCHGCXGWVU.jpg)

#### 3. add_instrumentation 函数

`add_instrumentation` 函数负责处理输入文件，生成 `modified_file` ，将 `instrumentation` 插入所有适当的位置。其整体控制流程如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_6PEXKCHGGH8VNQ9.jpg)

 

整体逻辑看上去有点复杂，但是关键内容并不算很多。在 main 函数中调用完 `edit_params()` 函数完成 `as_params` 参数数组的处理后，进入到该函数。

1.  判断 `input_file` 是否为空，如果不为空则尝试打开文件获取 fd 赋值给 `inf`，失败则抛出异常；`input_file` 为空则 `inf` 设置为标准输入；
    
2.  打开 `modified_file` ，获取 fd 赋值给 `outfd`，失败返回异常；进一步验证该文件是否可写，不可写返回异常；
    
3.  `while` 循环读取 `inf` 指向文件的每一行到 `line` 数组，每行最多 `MAX_LINE = 8192`个字节（含末尾的‘\0’），从`line`数组里将读取到的内容写入到 `outf` 指向的文件，然后进入到真正的插桩逻辑。这里需要注意的是，插桩只向 `.text` 段插入，：
    
    1.  首先跳过标签、宏、注释；
        
    2.  这里结合部分关键代码进行解释。需要注意的是，变量 `instr_ok` 本质上是一个 flag，用于表示是否位于`.text`段。变量设置为 1，表示位于 `.text` 中，如果不为 1，则表示不再。于是，如果`instr_ok` 为 1，就会在分支处执行插桩逻辑，否则就不插桩。
        
        1.  首先判断读入的行是否以‘\t’ 开头，本质上是在匹配`.s`文件中声明的段，然后判断`line[1]`是否为`.`：
            
            ```
            if (line[0] == '\t' && line[1] == '.') {
             
                  /* OpenBSD puts jump tables directly inline with the code, which is
                     a bit annoying. They use a specific format of p2align directives
                     around them, so we use that as a signal. */
             
                  if (!clang_mode && instr_ok && !strncmp(line + 2, "p2align ", 8) &&
                      isdigit(line[10]) && line[11] == '\n') skip_next_label = 1;
             
                  if (!strncmp(line + 2, "text\n", 5) ||
                      !strncmp(line + 2, "section\t.text", 13) ||
                      !strncmp(line + 2, "section\t__TEXT,__text", 21) ||
                      !strncmp(line + 2, "section __TEXT,__text", 21)) {
                    instr_ok = 1;
                    continue;
                  }
             
                  if (!strncmp(line + 2, "section\t", 8) ||
                      !strncmp(line + 2, "section ", 8) ||
                      !strncmp(line + 2, "bss\n", 4) ||
                      !strncmp(line + 2, "data\n", 5)) {
                    instr_ok = 0;
                    continue;
                  }
             
                }
            
            ```
            
            1.  '\t'开头，且`line[1]=='.'`，检查是否为 `p2align` 指令，如果是，则设置 `skip_next_label = 1`；
            2.  尝试匹配 `"text\n"` `"section\t.text"` `"section\t__TEXT,__text"` `"section __TEXT,__text"` 其中任意一个，匹配成功， 设置 `instr_ok = 1`， 表示位于 `.text` 段中，`continue` 跳出，进行下一次遍历；
            3.  尝试匹配`"section\t"` `"section "` `"bss\n"` `"data\n"` 其中任意一个，匹配成功，设置 `instr_ok = 0`，表位于其他段中，`continue` 跳出，进行下一次遍历；
        2.  接下来通过几个 `if` 判断，来设置一些标志信息，包括 `off-flavor assembly`，`Intel/AT&T`的块处理方式、`ad-hoc __asm__`块的处理方式等；
            
            ```
            /* Detect off-flavor assembly (rare, happens in gdb). When this is
               encountered, we set skip_csect until the opposite directive is
               seen, and we do not instrument. */
             
            if (strstr(line, ".code")) {
             
              if (strstr(line, ".code32")) skip_csect = use_64bit;
              if (strstr(line, ".code64")) skip_csect = !use_64bit;
             
            }
             
            /* Detect syntax changes, as could happen with hand-written assembly.
               Skip Intel blocks, resume instrumentation when back to AT&T. */
             
            if (strstr(line, ".intel_syntax")) skip_intel = 1;
            if (strstr(line, ".att_syntax")) skip_intel = 0;
             
            /* Detect and skip ad-hoc __asm__ blocks, likewise skipping them. */
             
            if (line[0] == '#' || line[1] == '#') {
             
              if (strstr(line, "#APP")) skip_app = 1;
              if (strstr(line, "#NO_APP")) skip_app = 0;
             
            }
            
            ```
            
        3.  AFL 在插桩时重点关注的内容包括：`^main, ^.L0, ^.LBB0_0, ^\tjnz foo` （_main 函数， gcc 和 clang 下的分支标记，条件跳转分支标记），这些内容通常标志了程序的流程变化，因此 AFL 会重点在这些位置进行插桩：
            
            对于形如`\tj[^m].`格式的指令，即条件跳转指令，且`R(100)`产生的随机数小于插桩密度`inst_ratio`，直接使用`fprintf`将`trampoline_fmt_64`(插桩部分的指令) 写入 `outf` 指向的文件，写入大小为小于 `MAP_SIZE`的随机数——`R(MAP_SIZE)`
            
            ，然后插桩计数`ins_lines`加一，`continue` 跳出，进行下一次遍历；
            
            ```
            /* If we're in the right mood for instrumenting, check for function
               names or conditional labels. This is a bit messy, but in essence,
               we want to catch:
             
                 ^main:      - function entry point (always instrumented)
                 ^.L0:       - GCC branch label
                 ^.LBB0_0:   - clang branch label (but only in clang mode)
                 ^\tjnz foo  - conditional branches
             
               ...but not:
             
                 ^# BB#0:    - clang comments
                 ^ # BB#0:   - ditto
                 ^.Ltmp0:    - clang non-branch labels
                 ^.LC0       - GCC non-branch labels
                 ^.LBB0_0:   - ditto (when in GCC mode)
                 ^\tjmp foo  - non-conditional jumps
             
               Additionally, clang and GCC on MacOS X follow a different convention
               with no leading dots on labels, hence the weird maze of #ifdefs
               later on.
             
             */
             
            if (skip_intel || skip_app || skip_csect || !instr_ok ||
                line[0] == '#' || line[0] == ' ') continue;
             
            /* Conditional branch instruction (jnz, etc). We append the instrumentation
               right after the branch (to instrument the not-taken path) and at the
               branch destination label (handled later on). */
             
            if (line[0] == '\t') {
             
              if (line[1] == 'j' && line[2] != 'm' && R(100) < inst_ratio) {
             
                fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32,
                        R(MAP_SIZE));
             
                ins_lines++;
             
              }
             
              continue;
             
            }
            
            ```
            
        4.  对于 label 的相关评估，有一些 label 可能是一些分支的目的地，需要自己的评判
            
            首先检查该行中是否存在`:`，然后检查是否以`.`开始
            
            1.  如果以`.`开始，则代表想要插桩`^.L0:`或者 `^.LBB0_0:`这样的 branch label，即 style jump destination
                
                1.  检查 `line[2]`是否为数字 或者 如果是在 clang_mode 下，比较从`line[1]`开始的三个字节是否为`LBB.`，前述所得结果和`R(100) < inst_ratio)`相与。如果结果为真，则设置`instrument_next = 1`；
            2.  否则代表这是一个 function，插桩`^func:`，function entry point，直接设置`instrument_next = 1`（defer mode）。
            
            ```
                /* Label of some sort. This may be a branch destination, but we need to
                   tread carefully and account for several different formatting
                   conventions. */
             
            #ifdef __APPLE__
             
                /* Apple: L: */
             
                if ((colon_pos = strstr(line, ":"))) {
             
                  if (line[0] == 'L' && isdigit(*(colon_pos - 1))) {
             
            #else
             
                /* Everybody else: .L: */
             
                if (strstr(line, ":")) {
             
                  if (line[0] == '.') {
             
            #endif /* __APPLE__ */
             
                    /* .L0: or LBB0_0: style jump destination */
             
            #ifdef __APPLE__
             
                    /* Apple: L / LBB */
             
                    if ((isdigit(line[1]) || (clang_mode && !strncmp(line, "LBB", 3)))
                        && R(100) < inst_ratio) {
             
            #else
             
                    /* Apple: .L / .LBB */
             
                    if ((isdigit(line[2]) || (clang_mode && !strncmp(line + 1, "LBB", 3)))
                        && R(100) < inst_ratio) {
             
            #endif /* __APPLE__ */
             
                      /* An optimization is possible here by adding the code only if the
                         label is mentioned in the code in contexts other than call / jmp.
                         That said, this complicates the code by requiring two-pass
                         processing (messy with stdin), and results in a speed gain
                         typically under 10%, because compilers are generally pretty good
                         about not generating spurious intra-function jumps.
             
                         We use deferred output chiefly to avoid disrupting
                         .Lfunc_begin0-style exception handling calculations (a problem on
                         MacOS X). */
             
                      if (!skip_next_label) instrument_next = 1; else skip_next_label = 0;
             
                    }
             
                  } else {
             
                    /* Function label (always instrumented, deferred mode). */
             
                    instrument_next = 1;
             
                  }
                }
              } 
            ```
            
        5.  上述过程完成后，来到 `while` 循环的下一个循环，在 `while` 的开头，可以看到对以 defered mode 进行插桩的位置进行了真正的插桩处理：
            
            ```
            if (!pass_thru && !skip_intel && !skip_app && !skip_csect && instr_ok &&
                instrument_next && line[0] == '\t' && isalpha(line[1])) {
             
              fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32,
                      R(MAP_SIZE));
             
              instrument_next = 0;
              ins_lines++;
             
            }
            
            ```
            
            这里对 `instr_ok, instrument_next` 变量进行了检验是否为 1，而且进一步校验是否位于 `.text` 段中，且设置了 defered mode 进行插桩，则就进行插桩操作，写入 `trampoline_fmt_64/32` 。
            

至此，插桩函数 `add_instrumentation` 的主要逻辑已梳理完成。

#### 4. edit_params 函数

`edit_params`，该函数主要是设置变量 `as_params` 的值，以及 `use_64bit/modified_file` 的值， 其整体控制流程如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_T9FVPCSEVNTXUCR.jpg)

1.  获取环境变量 `TMPDIR` 和 `AFL_AS`;
    
2.  对于 `__APPLE_` 宏， 如果当前在 `clang_mode` 且没有设置 `AFL_AS` 环境变量，会设置 `use_clang_mode = 1`，并设置 `afl-as` 为 `AFL_CC/AFL_CXX/clang`中的一种；
    
3.  设置 `tmp_dir` ，尝试获取的环境变量依次为 `TEMP, TMP`，如果都失败，则直接设置为 `/tmp`；
    
4.  调用 `ck_alloc()` 函数为 `as_params` 参数数组分配内存，大小为 (argc + 32) * 8；
    
5.  设置 `afl-as` 路径：`as_params[0] = afl_as ? afl_as : (u8*)"as";`
    
6.  设置 `as_params[argc] = 0;` ，as_par_cnt 初始值为 1；
    
7.  遍历从 `argv[1]` 到 `argv[argc-1]` 之前的每个 argv：
    
    1.  如果存在字符串 `--64`， 则设置 `use_64bit = 1` ；如果存在字符串 `--32` ，则设置 `use_64bit = 0`。对于`__APPLE__` ，如果存在`-arch x86_64`，设置 `use_64bit=1`，并跳过`-q`和`-Q`选项；
    2.  `as_params[as_par_cnt++] = argv[i]`，设置 as_params 的值为 argv 对应的参数值
8.  开始设置其他参数：
    
    1.  对于 `__APPLE__`，如果设置了 `use_clang_as`，则追加 `-c -x assembler`；
        
    2.  设置 `input_file` 变量：`input_file = argv[argc - 1];`，把最后一个参数的值作为 `input_file`；
        
        1.  如果 `input_file` 的首字符为`-`：
            
            1.  如果后续为 `-version`，则 `just_version = 1`, `modified_file = input_file`，然后跳转到`wrap_things_up`。这里就只是做`version`的查询；
            2.  如果后续不为 `-version`，抛出异常；
        2.  如果 `input_file` 首字符不为`-`，比较 `input_file` 和 `tmp_dir`、`/var/tmp` 、`/tmp/`的前 `strlen(tmp_dir)/9/5`个字节是否相同，如果不相同，就设置 `pass_thru` 为 1；
            
    3.  设置 `modified_file`：`modified_file = alloc_printf("%s/.afl-%u-%u.s", tmp_dir, getpid(),
        
        ```
        (u32)time(NULL));`，即为`tmp_dir/afl-pid-tim.s` 格式的字符串
        
        ```
        
        1.  设置`as_params[as_par_cnt++] = modified_file`，`as_params[as_par_cnt] = NULL;`。

### 3. instrumentation trampoline 和 main_payload

`trampoline` 的含义是 “蹦床”，直译过来就是 “插桩蹦床”。个人感觉直接使用英文更能表达出其代表的真实含义和作用，可以简单理解为桩代码。

#### 1. trampoline_fmt_64/32

根据前面内容知道，在 64 位环境下，AFL 会插入 `trampoline_fmt_64` 到文件中，在 32 位环境下，AFL 会插入`trampoline_fmt_32` 到文件中。`trampoline_fmt_64/32`定义在 `afl-as.h` 头文件中：

```
static const u8* trampoline_fmt_32 =
 
  "\n"
  "/* --- AFL TRAMPOLINE (32-BIT) --- */\n"
  "\n"
  ".align 4\n"
  "\n"
  "leal -16(%%esp), %%esp\n"
  "movl %%edi,  0(%%esp)\n"
  "movl %%edx,  4(%%esp)\n"
  "movl %%ecx,  8(%%esp)\n"
  "movl %%eax, 12(%%esp)\n"
  "movl $0x%08x, %%ecx\n"    // 向ecx中存入识别代码块的随机桩代码id
  "call __afl_maybe_log\n"   // 调用 __afl_maybe_log 函数
  "movl 12(%%esp), %%eax\n"
  "movl  8(%%esp), %%ecx\n"
  "movl  4(%%esp), %%edx\n"
  "movl  0(%%esp), %%edi\n"
  "leal 16(%%esp), %%esp\n"
  "\n"
  "/* --- END --- */\n"
  "\n";
 
static const u8* trampoline_fmt_64 =
 
  "\n"
  "/* --- AFL TRAMPOLINE (64-BIT) --- */\n"
  "\n"
  ".align 4\n"
  "\n"
  "leaq -(128+24)(%%rsp), %%rsp\n"
  "movq %%rdx,  0(%%rsp)\n"
  "movq %%rcx,  8(%%rsp)\n"
  "movq %%rax, 16(%%rsp)\n"
  "movq $0x%08x, %%rcx\n"  // 64位下使用的寄存器为rcx
  "call __afl_maybe_log\n" // 调用 __afl_maybe_log 函数
  "movq 16(%%rsp), %%rax\n"
  "movq  8(%%rsp), %%rcx\n"
  "movq  0(%%rsp), %%rdx\n"
  "leaq (128+24)(%%rsp), %%rsp\n"
  "\n"
  "/* --- END --- */\n"
  "\n";

```

上面列出的插桩代码与我们在 `.s` 文件和 IDA 逆向中看到的插桩代码是一样的：

 

`.s` 文件中的桩代码：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_3RGZYQCMHHE6Z3U.jpg)

 

IDA 逆向中显示的桩代码：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_9MMMF8SM85QBWWF.jpg)

 

上述代码执行的主要功能包括：

*   保存 `rdx`、 `rcx` 、`rax` 寄存器
*   将 `rcx` 的值设置为 `fprintf()` 函数将要打印的变量内容
*   调用 `__afl_maybe_log` 函数
*   恢复寄存器

在以上的功能中， `__afl_maybe_log` 才是核心内容。

 

从 `__afl_maybe_log` 函数开始，后续的处理流程大致如下 (图片来自 ScUpax0s 师傅)：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_GPYAW7Q2MW85DPG.jpg)

 

首先对上面流程中涉及到的几个 bss 段的变量进行简单说明（以 64 位为例，从`main_payload_64`中提取）：

```
.AFL_VARS:
 
  .comm   __afl_area_ptr, 8
  .comm   __afl_prev_loc, 8
  .comm   __afl_fork_pid, 4
  .comm   __afl_temp, 4
  .comm   __afl_setup_failure, 1
  .comm    __afl_global_area_ptr, 8, 8

```

*   `__afl_area_ptr`：共享内存地址；
*   `__afl_prev_loc`：上一个插桩位置（id 为 R(100) 随机数的值）；
*   `__afl_fork_pid`：由 fork 产生的子进程的 pid；
*   `__afl_temp`：缓冲区；
*   `__afl_setup_failure`：标志位，如果置位则直接退出；
*   `__afl_global_area_ptr`：全局指针。

**说明**

 

以下介绍的指令段均来自于 `main_payload_64` 。

#### 2. __afl_maybe_log

```
__afl_maybe_log:   /* 源码删除无关内容后 */
 
  lahf
  seto  %al
 
  /* Check if SHM region is already mapped. */
 
  movq  __afl_area_ptr(%rip), %rdx
  testq %rdx, %rdx
  je    __afl_setup

```

首先，使用 `lahf` 指令（加载状态标志位到`AH`）将 EFLAGS 寄存器的低八位复制到 `AH`，被复制的标志位包括：符号标志位（SF）、零标志位（ZF）、辅助进位标志位（AF）、奇偶标志位（PF）和进位标志位（CF），使用该指令可以方便地将标志位副本保存在变量中；

 

然后，使用 `seto` 指令溢出置位；

 

接下来检查共享内存是否进行了设置，判断 `__afl_area_ptr` 是否为 NULL：

*   如果为 NULL，跳转到 `__afl_setup` 函数进行设置；
*   如果不为 NULL，继续进行。

#### 3. __afl_setup

```
__afl_setup:
 
        /* Do not retry setup is we had previous failues. */
        cmpb $0, __afl_setup_failure(%rip)
        jne __afl_return
 
        /* Check out if we have a global pointer on file. */
        movq __afl_global_area_ptr(%rip), %rdx
        testq %rdx, %rdx
        je __afl_setup_first
 
        movq %rdx, __afl_area_ptr(%rip)
        jmp  __afl_store

```

该部分的主要作用为初始化 `__afl_area_ptr` ，且只在运行到第一个桩时进行本次初始化。

 

首先，如果 `__afl_setup_failure` 不为 0，直接跳转到 `__afl_return` 返回；

 

然后，检查 `__afl_global_area_ptr` 文件指针是否为 NULL：

*   如果为 NULL，跳转到 `__afl_setup_first` 进行接下来的工作；
*   如果不为 NULL，将 `__afl_global_area_ptr` 的值赋给 `__afl_area_ptr`，然后跳转到 `__afl_store` 。

#### 4. __afl_setup_first

```
__afl_setup_first:
 
  /* Save everything that is not yet saved and that may be touched by
     getenv() and several other libcalls we'll be relying on. */
 
  leaq -352(%rsp), %rsp
 
  movq %rax,   0(%rsp)
  movq %rcx,   8(%rsp)
  movq %rdi,  16(%rsp)
  movq %rsi,  32(%rsp)
  movq %r8,   40(%rsp)
  movq %r9,   48(%rsp)
  movq %r10,  56(%rsp)
  movq %r11,  64(%rsp)
 
  movq %xmm0,  96(%rsp)
  movq %xmm1,  112(%rsp)
  movq %xmm2,  128(%rsp)
  movq %xmm3,  144(%rsp)
  movq %xmm4,  160(%rsp)
  movq %xmm5,  176(%rsp)
  movq %xmm6,  192(%rsp)
  movq %xmm7,  208(%rsp)
  movq %xmm8,  224(%rsp)
  movq %xmm9,  240(%rsp)
  movq %xmm10, 256(%rsp)
  movq %xmm11, 272(%rsp)
  movq %xmm12, 288(%rsp)
  movq %xmm13, 304(%rsp)
  movq %xmm14, 320(%rsp)
  movq %xmm15, 336(%rsp)
 
  /* Map SHM, jumping to __afl_setup_abort if something goes wrong. */
 
  /* The 64-bit ABI requires 16-byte stack alignment. We'll keep the
     original stack ptr in the callee-saved r12. */
 
  pushq %r12
  movq  %rsp, %r12
  subq  $16, %rsp
  andq  $0xfffffffffffffff0, %rsp
 
  leaq .AFL_SHM_ENV(%rip), %rdi
call _getenv
 
  testq %rax, %rax
  je    __afl_setup_abort
 
  movq  %rax, %rdi
call _atoi
 
  xorq %rdx, %rdx   /* shmat flags    */
  xorq %rsi, %rsi   /* requested addr */
  movq %rax, %rdi   /* SHM ID         */
call _shmat
 
  cmpq $-1, %rax
  je   __afl_setup_abort
 
  /* Store the address of the SHM region. */
 
  movq %rax, %rdx
  movq %rax, __afl_area_ptr(%rip)
 
  movq %rax, __afl_global_area_ptr(%rip)
  movq %rax, %rdx

```

首先，保存所有寄存器的值，包括 `xmm` 寄存器组；

 

然后，进行 `rsp` 的对齐；

 

然后，获取环境变量 `__AFL_SHM_ID`，该环境变量保存的是共享内存的 ID：

*   如果获取失败，跳转到 `__afl_setup_abort` ；
*   如果获取成功，调用 `_shmat` ，启用对共享内存的访问，启用失败跳转到 `__afl_setup_abort`。

接下来，将 `_shmat` 返回的共享内存地址存储在 `__afl_area_ptr` 和 `__afl_global_area_ptr` 变量中。

 

后面即开始运行 `__afl_forkserver`。

#### 5. __afl_forkserver

```
__afl_forkserver:
 
     /* Enter the fork server mode to avoid the overhead of execve() calls. We
     push rdx (area ptr) twice to keep stack alignment neat. */
 
  pushq %rdx
  pushq %rdx
 
  /* Phone home and tell the parent that we're OK. (Note that signals with
     no SA_RESTART will mess it up). If this fails, assume that the fd is
     closed because we were execve()d from an instrumented binary, or because
     the parent doesn't want to use the fork server. */
 
  movq $4, %rdx               /* length    */
  leaq __afl_temp(%rip), %rsi /* data      */
  movq $" STRINGIFY((FORKSRV_FD + 1)) ", %rdi       /* file desc */
CALL_L64("write")
 
  cmpq $4, %rax
  jne  __afl_fork_resume

```

这一段实现的主要功能是向 `FORKSRV_FD+1` （也就是 198+1）号描述符（即状态管道）中写 `__afl_temp` 中的 4 个字节，告诉 fork server （将在后续的文章中进行详细解释）已经成功启动。

#### 6. __afl_fork_wait_loop

```
__afl_fork_wait_loop:
 
  /* Wait for parent by reading from the pipe. Abort if read fails. */
 
  movq $4, %rdx               /* length    */
  leaq __afl_temp(%rip), %rsi /* data      */
  movq $" STRINGIFY(FORKSRV_FD) ", %rdi            /* file desc */
CALL_L64("read")
  cmpq $4, %rax
  jne  __afl_die
 
  /* Once woken up, create a clone of our process. This is an excellent use
     case for syscall(__NR_clone, 0, CLONE_PARENT), but glibc boneheadedly
     caches getpid() results and offers no way to update the value, breaking
     abort(), raise(), and a bunch of other things :-( */
 
CALL_L64("fork")
  cmpq $0, %rax
  jl   __afl_die
  je   __afl_fork_resume
 
  /* In parent process: write PID to pipe, then wait for child. */
 
  movl %eax, __afl_fork_pid(%rip)
 
  movq $4, %rdx                   /* length    */
  leaq __afl_fork_pid(%rip), %rsi /* data      */
  movq $" STRINGIFY((FORKSRV_FD + 1)) ", %rdi             /* file desc */
CALL_L64("write")
 
  movq $0, %rdx                   /* no flags  */
  leaq __afl_temp(%rip), %rsi     /* status    */
  movq __afl_fork_pid(%rip), %rdi /* PID       */
CALL_L64("waitpid")
  cmpq $0, %rax
  jle  __afl_die
 
  /* Relay wait status to pipe, then loop back. */
 
  movq $4, %rdx               /* length    */
  leaq __afl_temp(%rip), %rsi /* data      */
  movq $" STRINGIFY((FORKSRV_FD + 1)) ", %rdi         /* file desc */
CALL_L64("write")
 
  jmp  __afl_fork_wait_loop

```

1.  等待 fuzzer 通过控制管道发送过来的命令，读入到 `__afl_temp` 中：
    *   读取失败，跳转到 `__afl_die` ，结束循环；
    *   读取成功，继续；
2.  fork 一个子进程，子进程执行 `__afl_fork_resume`；
3.  将子进程的 pid 赋给 `__afl_fork_pid`，并写到状态管道中通知父进程；
4.  等待子进程执行完成，写入状态管道告知 fuzzer；
5.  重新执行下一轮 `__afl_fork_wait_loop` 。

#### 7. __afl_fork_resume

```
__afl_fork_resume:
 
/* In child process: close fds, resume execution. */
 
  movq $" STRINGIFY(FORKSRV_FD) ", %rdi
CALL_L64("close")
 
  movq $(" STRINGIFY(FORKSRV_FD) " + 1), %rdi
CALL_L64("close")
 
  popq %rdx
  popq %rdx
 
  movq %r12, %rsp
  popq %r12
 
  movq  0(%rsp), %rax
  movq  8(%rsp), %rcx
  movq 16(%rsp), %rdi
  movq 32(%rsp), %rsi
  movq 40(%rsp), %r8
  movq 48(%rsp), %r9
  movq 56(%rsp), %r10
  movq 64(%rsp), %r11
 
  movq  96(%rsp), %xmm0
  movq 112(%rsp), %xmm1
  movq 128(%rsp), %xmm2
  movq 144(%rsp), %xmm3
  movq 160(%rsp), %xmm4
  movq 176(%rsp), %xmm5
  movq 192(%rsp), %xmm6
  movq 208(%rsp), %xmm7
  movq 224(%rsp), %xmm8
  movq 240(%rsp), %xmm9
  movq 256(%rsp), %xmm10
  movq 272(%rsp), %xmm11
  movq 288(%rsp), %xmm12
  movq 304(%rsp), %xmm13
  movq 320(%rsp), %xmm14
  movq 336(%rsp), %xmm15
 
  leaq 352(%rsp), %rsp
 
  jmp  __afl_store

```

1.  关闭子进程中的 fd；
2.  恢复子进程的寄存器状态；
3.  跳转到 `__afl_store` 执行。

#### 8. __afl_store

```
__afl_store:
 
  /* Calculate and store hit for the code location specified in rcx. */
 
  xorq __afl_prev_loc(%rip), %rcx
  xorq %rcx, __afl_prev_loc(%rip)
  shrq $1, __afl_prev_loc(%rip)
 
  incb (%rdx, %rcx, 1)

```

我们直接看反编译的代码：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_B56D3HHPNVNTGP4.jpg)

 

这里第一步的异或中的 `a4` ，其实是调用 `__afl_maybe_log` 时传入 的参数：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_URYVNBM7APY98YD.jpg)

 

再往上追溯到插桩代码：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_Z6UNUUGWU6PG3PR.jpg)

 

可以看到传入 `rcx` 的，实际上就是用于标记当前桩的随机 id， 而 `_afl_prev_loc` 其实是上一个桩的随机 id。

 

经过两次异或之后，再将 `_afl_prev_loc` 右移一位作为新的 `_afl_prev_loc`，最后再共享内存中存储当前插桩位置的地方计数加一。

[](#三、总结)三、总结
-------------

本文综合分析了 AFL 中的 gcc 部分和插桩部分的源代码，由衷佩服 AFL 设计开发者的巧妙思路和高超的开发技巧，不愧是开启了 fuzzing 新时代的、影响力巨大的 fuzz 工具。

[](#参考文献：)参考文献：
---------------

衷心感谢乐于分享的师傅们，能让我站在巨人的肩膀上。

1.  https://eternalsakura13.com/2020/08/23/afl/
2.  https://bbs.pediy.com/thread-265936.htm

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

最后于 1 天前 被有毒编辑 ，原因：

[#自动化挖掘](forum-150-1-155.htm) [#Fuzz](forum-150-1-157.htm) [#Linux](forum-150-1-161.htm)