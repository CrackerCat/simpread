> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269537.htm)

> AFL 二三事 -- 源码分析 3

AFL 二三事 -- 源码分析 3

目录

*            [前言](#前言)
*            AFL 的 fuzzer —— afl-fuzz.c
*                    1. 、概述
*                    [2、核心源码分析](#2、核心源码分析)
*                            1. 初始配置
*                                    1.1 第一个 while 循环
*                                    1.2 setup_signal_handlers 函数
*                                    1.3 check_asan_opts 函数
*                                    1.4 fix_up_sync 函数
*                                    1.5 save_cmdline 函数
*                                    1.6 check_if_tty 函数
*                                    1.7 几个 CPU 检查相关的函数
*                                    1.8 **setup_shm** 函数
*                                    1.9 setup_dirs_fds 函数
*                                    1.10 read_testcases 函数
*                                    1.11 add_to_queue 函数
*                                    1.12 pivot_inputs 函数
*                                    1. 13 find_timeout 函数
*                                    1.14 detect_file_args
*                                    1.15 check_binary 函数
*                            2. 第一遍 fuzz
*                                    2.1 检查
*                                    2.2 perform_dry_run 函数
*                                    2.3 calibrate_case 函数
*                                    2.4 init_forkserve 函数
*                                    2.5 run_target 函数
*                                    2.6 update_bitmap_score 函数
*                            3. 主循环
*                                    3.1 主循环之前
*                                            3.1.1 cull_queue 函数
*                                            3.1.2 show_init_stats 函数
*                                            3.1.3 find_start_position 函数
*                                            3.1.4 write_stats_file 函数
*                                            3.1.5 save_auto 函数
*                                    3.2 主循环
*                                    3.3 主循环后
*                                            3.3.1 fuzz_one 函数
*                                            3.3.2 sync_fuzzers 函数
*            [四、总结](#四、总结)
*            [五、参考文献](#五、参考文献)

前言
--

本文为《AFL 二三事》-- 源码分析系列的第三篇，主要阅读 AFL 的 fuzzer 部分的源码，学习 AFL 的 fuzz 核心。

 

**当别人都要快的时候，你要慢下来。**

AFL 的 fuzzer —— afl-fuzz.c
--------------------------

### 1. 、概述

AFL 中最重要的部分便是 fuzzer 的实现部分——`afl_fuzz.c` ，其主要作用是通过不断变异测试用例来影响程序的执行路径。该文件代码量在 8000 行左右，处于篇幅原因，我们不会对每一个函数进行源码级分析，而是按照功能划分，介绍其中的核心函数。该文件属于 AFL 整个项目的核心中的核心，强烈建议通读该文件。

 

在介绍源码的同时，会穿插 AFL 的整体运行过程和设计思路，辅助理解源码的设计思路。

 

在功能上，可以总体上分为 3 部分：

1.  初始配置：进行 fuzz 环境配置相关工作
2.  fuzz 执行：fuzz 的主循环过程
3.  变异策略：测试用例的变异过程和方式

我们将按照以上 3 个功能对其中的关键函数和流程进行分析。

### [](#2、核心源码分析)2、核心源码分析

#### 1. 初始配置

##### 1.1 第一个 while 循环

```
while ((opt = getopt(argc, argv, "+i:o:f:m:b:t:T:dnCB:S:M:x:QV")) > 0)
  ... ...

```

该循环主要通过 `getopt` 获取各种环境配置、选项参数等。

##### 1.2 setup_signal_handlers 函数

调用 `sigaction` ，注册信号处理函数，设置信号句柄。具体的信号内容如下：

<table><thead><tr><th>信号</th><th>作用</th></tr></thead><tbody><tr><td>SIGHUP/SIGINT/SIGTERM</td><td>处理各种 “stop” 情况</td></tr><tr><td>SIGALRM</td><td>处理超时的情况</td></tr><tr><td>SIGWINCH</td><td>处理窗口大小</td></tr><tr><td>SIGUSER1</td><td>用户自定义信号，这里定义为 skip request</td></tr><tr><td>SIGSTP/SIGPIPE</td><td>不是很重要的一些信号，可以不用关心</td></tr></tbody></table>

##### 1.3 check_asan_opts 函数

读取环境变量 `ASAN_OPTIONS` 和 `MSAN_OPTIONS`，做一些必要性检查。

##### 1.4 fix_up_sync 函数

如果通过 `-M`或者`-S`指定了 `sync_id`，则更新 `out_dir` 和 `sync_dir` 的值：设置 `sync_dir` 的值为 `out_dir`，设置 `out_dir` 的值为`out_dir/sync_id`。

##### 1.5 save_cmdline 函数

copy 当前命令行参数，保存。

##### 1.6 check_if_tty 函数

检查是否在 tty 终端上面运行：读取环境变量 `AFL_NO_UI` ，如果存在，设置 `not_on_tty` 为 1，并返回；通过 `ioctl` 读取 window size，如果报错为 `ENOTTY`，表示当前不在一个 tty 终端运行，设置 `not_on_tty`。

##### 1.7 几个 CPU 检查相关的函数

*   `static void get_core_count(void)get_core_count()`：获取核心数量
*   `check_crash_handling()`：确保核心转储不会进入程序
*   `check_cpu_governor()`：检查 CPU 管理者

##### 1.8 **setup_shm** 函数

该函数用于设置共享内存和 `virgin_bits`，属于比较重要的函数，这里我们结合源码来解析一下：

```
/* Configure shared memory and virgin_bits. This is called at startup. */
 
EXP_ST void setup_shm(void) {
 
  u8* shm_str;
 
  if (!in_bitmap) memset(virgin_bits, 255, MAP_SIZE);
  // 如果 in_bitmap 为空，调用 memset 初始化数组 virgin_bits[MAP_SIZE] 的每个元素的值为 ‘255’。
 
  memset(virgin_tmout, 255, MAP_SIZE); // 调用 memset 初始化数组 virgin_tmout[MAP_SIZE] 的每个元素的值为 ‘255’。
  memset(virgin_crash, 255, MAP_SIZE); // 调用 memset 初始化数组 virgin_crash[MAP_SIZE] 的每个元素的值为 ‘255’。
 
  shm_id = shmget(IPC_PRIVATE, MAP_SIZE, IPC_CREAT | IPC_EXCL | 0600);
  // 调用 shmget 函数分配一块共享内存，并将返回的共享内存标识符保存到 shm_id
 
  if (shm_id < 0) PFATAL("shmget() failed");
 
  atexit(remove_shm); // 注册 atexit handler 为 remove_shm
 
  shm_str = alloc_printf("%d", shm_id); // 创建字符串 shm_str
 
  /* If somebody is asking us to fuzz instrumented binaries in dumb mode,
     we don't want them to detect instrumentation, since we won't be sending
     fork server commands. This should be replaced with better auto-detection
     later on, perhaps? */
 
  if (!dumb_mode) setenv(SHM_ENV_VAR, shm_str, 1);
  // 如果不是dumb_mode，设置环境变量 SHM_ENV_VAR 的值为 shm_str
 
  ck_free(shm_str);
 
  trace_bits = shmat(shm_id, NULL, 0);
  // 设置 trace_bits 并初始化为0
 
  if (trace_bits == (void *)-1) PFATAL("shmat() failed");
 
}

```

这里通过 `trace_bits` 和 `virgin_bits` 两个 bitmap 来分别记录当前的 tuple 信息及整体 tuple 信息，其中 `trace_bits` 位于共享内存上，便于进行进程间通信。通过 `virgin_tmout` 和 `virgin_crash` 两个 bitmap 来记录 fuzz 过程中出现的所有目标程序超时以及崩溃的 tuple 信息。

##### 1.9 setup_dirs_fds 函数

该函数用于准备输出文件夹和文件描述符，结合源码进行解析：

```
EXP_ST void setup_dirs_fds(void) {
 
  u8* tmp;
  s32 fd;
 
  ACTF("Setting up output directories...");
 
  if (sync_id && mkdir(sync_dir, 0700) && errno != EEXIST)
      PFATAL("Unable to create '%s'", sync_dir);
  /* 如果sync_id，且创建sync_dir文件夹并设置权限为0700，如果报错单errno不是 EEXIST ，抛出异常 */
 
  if (mkdir(out_dir, 0700)) { // 创建out_dir， 权限为0700
 
    if (errno != EEXIST) PFATAL("Unable to create '%s'", out_dir);
 
    maybe_delete_out_dir();
 
  } else {
 
    if (in_place_resume) // 创建成功
      FATAL("Resume attempted but old output directory not found");
 
    out_dir_fd = open(out_dir, O_RDONLY); // 以只读模式打开，返回fd：out_dir_fd
 
#ifndef __sun
 
    if (out_dir_fd < 0 || flock(out_dir_fd, LOCK_EX | LOCK_NB))
      PFATAL("Unable to flock() output directory.");
 
#endif /* !__sun */
 
  }
 
  /* Queue directory for any starting & discovered paths. */
 
  tmp = alloc_printf("%s/queue", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp); 
  // 创建 out_dir/queue 文件夹，权限为0700
 
  ck_free(tmp);
 
  /* Top-level directory for queue metadata used for session
     resume and related tasks. */
 
  tmp = alloc_printf("%s/queue/.state/", out_dir);
 
  // 创建 out_dir/queue/.state 文件夹，用于保存session resume 和相关tasks的队列元数据。
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* Directory for flagging queue entries that went through
     deterministic fuzzing in the past. */
 
  tmp = alloc_printf("%s/queue/.state/deterministic_done/", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* Directory with the auto-selected dictionary entries. */
 
  tmp = alloc_printf("%s/queue/.state/auto_extras/", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* The set of paths currently deemed redundant. */
 
  tmp = alloc_printf("%s/queue/.state/redundant_edges/", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* The set of paths showing variable behavior. */
 
  tmp = alloc_printf("%s/queue/.state/variable_behavior/", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* Sync directory for keeping track of cooperating fuzzers. */
 
  if (sync_id) {
 
    tmp = alloc_printf("%s/.synced/", out_dir);
 
    if (mkdir(tmp, 0700) && (!in_place_resume || errno != EEXIST))
      PFATAL("Unable to create '%s'", tmp);
 
    ck_free(tmp);
 
  }
 
  /* All recorded crashes. */
 
  tmp = alloc_printf("%s/crashes", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* All recorded hangs. */
 
  tmp = alloc_printf("%s/hangs", out_dir);
  if (mkdir(tmp, 0700)) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  /* Generally useful file descriptors. */
 
  dev_null_fd = open("/dev/null", O_RDWR);
  if (dev_null_fd < 0) PFATAL("Unable to open /dev/null");
 
  dev_urandom_fd = open("/dev/urandom", O_RDONLY);
  if (dev_urandom_fd < 0) PFATAL("Unable to open /dev/urandom");
 
  /* Gnuplot output file. */
 
  tmp = alloc_printf("%s/plot_data", out_dir);
  fd = open(tmp, O_WRONLY | O_CREAT | O_EXCL, 0600);
  if (fd < 0) PFATAL("Unable to create '%s'", tmp);
  ck_free(tmp);
 
  plot_file = fdopen(fd, "w");
  if (!plot_file) PFATAL("fdopen() failed");
 
  fprintf(plot_file, "# unix_time, cycles_done, cur_path, paths_total, "
                     "pending_total, pending_favs, map_size, unique_crashes, "
                     "unique_hangs, max_depth, execs_per_sec\n");
                     /* ignore errors */

```

该函数的源码中，开发者对关键位置均做了清楚的注释，很容易理解，不做过多解释。

##### 1.10 read_testcases 函数

该函数会将 `in_dir` 目录下的测试用例扫描到 `queue` 中，并且区分该文件是否为经过确定性变异的 input，如果是的话跳过，以节省时间。  
调用函数 `add_to_queue()` 将测试用例排成 queue 队列。该函数会在启动时进行调用。

##### 1.11 add_to_queue 函数

该函数主要用于将新的 test case 添加到队列，初始化 `fname` 文件名称，增加`cur_depth` 深度，增加 `queued_paths` 测试用例数量等。

 

首先，`queue_entry` 结构体定义如下：

```
struct queue_entry {
 
  u8* fname;                          /* File name for the test case      */
  u32 len;                            /* Input length                     */
 
  u8  cal_failed,                     /* Calibration failed?              */
      trim_done,                      /* Trimmed?                         */
      was_fuzzed,                     /* Had any fuzzing done yet?        */
      passed_det,                     /* Deterministic stages passed?     */
      has_new_cov,                    /* Triggers new coverage?           */
      var_behavior,                   /* Variable behavior?               */
      favored,                        /* Currently favored?               */
      fs_redundant;                   /* Marked as redundant in the fs?   */
 
  u32 bitmap_size,                    /* Number of bits set in bitmap     */
      exec_cksum;                     /* Checksum of the execution trace  */
 
  u64 exec_us,                        /* Execution time (us)              */
      handicap,                       /* Number of queue cycles behind    */
      depth;                          /* Path depth                       */
 
  u8* trace_mini;                     /* Trace bytes, if kept             */
  u32 tc_ref;                         /* Trace bytes ref count            */
 
  struct queue_entry *next,           /* Next element, if any             */
                     *next_100;       /* 100 elements ahead               */
 
};

```

然后在函数内部进行的相关操作如下：

```
/* Append new test case to the queue. */
 
static void add_to_queue(u8* fname, u32 len, u8 passed_det) {
 
  struct queue_entry* q = ck_alloc(sizeof(struct queue_entry));
  // 通过ck_alloc分配一个 queue_entry 结构体，并进行初始化
 
  q->fname        = fname;
  q->len          = len;
  q->depth        = cur_depth + 1;
  q->passed_det   = passed_det;
 
  if (q->depth > max_depth) max_depth = q->depth;
 
  if (queue_top) {
 
    queue_top->next = q;
    queue_top = q;
 
  } else q_prev100 = queue = queue_top = q;
 
  queued_paths++; // queue计数器加1
  pending_not_fuzzed++; // 待fuzz的样例计数器加1
 
  cycles_wo_finds = 0;
 
  /* Set next_100 pointer for every 100th element (index 0, 100, etc) to allow faster iteration. */
  if ((queued_paths - 1) % 100 == 0 && queued_paths > 1) {
 
    q_prev100->next_100 = q;
    q_prev100 = q;
 
  }
 
  last_path_time = get_cur_time();
 
}

```

##### 1.12 pivot_inputs 函数

在输出目录中为输入测试用例创建硬链接。

##### 1. 13 find_timeout 函数

变量 `timeout_given` 没有被设置时，会调用到该函数。该函数主要是在没有指定 `-t` 选项进行 resuming session 时，避免一次次地自动调整超时时间。

##### 1.14 detect_file_args

识别参数中是否有 “@@”，如果有，则替换为 `out_dir/.cur_input` ，没有则返回：

```
/* Detect @@ in args. */
 
EXP_ST void detect_file_args(char** argv) {
 
  u32 i = 0;
  u8* cwd = getcwd(NULL, 0);
 
  if (!cwd) PFATAL("getcwd() failed");
 
  while (argv[i]) {
 
    u8* aa_loc = strstr(argv[i], "@@"); // 查找@@
 
    if (aa_loc) {
 
      u8 *aa_subst, *n_arg;
 
      /* If we don't have a file name chosen yet, use a safe default. */
 
      if (!out_file)
        out_file = alloc_printf("%s/.cur_input", out_dir);
 
      /* Be sure that we're always using fully-qualified paths. */
 
      if (out_file[0] == '/') aa_subst = out_file;
      else aa_subst = alloc_printf("%s/%s", cwd, out_file);
 
      /* Construct a replacement argv value. */
 
      *aa_loc = 0;
      n_arg = alloc_printf("%s%s%s", argv[i], aa_subst, aa_loc + 2);
      argv[i] = n_arg;
      *aa_loc = '@';
 
      if (out_file[0] != '/') ck_free(aa_subst);
 
    }
    i++;
  }
  free(cwd); /* not tracked */
 
}

```

##### 1.15 check_binary 函数

检查指定路径要执行的程序是否存在，是否为 shell 脚本，同时检查 elf 文件头是否合法及程序是否被插桩。

#### 2. 第一遍 fuzz

##### 2.1 检查

调用 `get_cur_time()` 函数获取开始时间，检查是否处于 `qemu_mode`。

##### 2.2 perform_dry_run 函数

该函数是 AFL 中的一个关键函数，它会执行 `input` 文件夹下的预先准备的所有测试用例，生成初始化的 queue 和 bitmap，只对初始输入执行一次。函数控制流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_ZKDYB9DDX9WG9BB.jpg)

 

下面将结合函数源码进行解析（删除部分非关键代码）：

```
/* Perform dry run of all test cases to confirm that the app is working as
   expected. This is done only for the initial inputs, and only once. */
 
static void perform_dry_run(char** argv) {
 
  struct queue_entry* q = queue; // 创建queue_entry结构体
  u32 cal_failures = 0;
  u8* skip_crashes = getenv("AFL_SKIP_CRASHES"); // 读取环境变量 AFL_SKIP_CRASHES
 
  while (q) { // 遍历队列
 
    u8* use_mem;
    u8  res;
    s32 fd;
 
    u8* fn = strrchr(q->fname, '/') + 1;
 
    ACTF("Attempting dry run with '%s'...", fn);
 
    fd = open(q->fname, O_RDONLY);
    if (fd < 0) PFATAL("Unable to open '%s'", q->fname);
 
    use_mem = ck_alloc_nozero(q->len);
 
    if (read(fd, use_mem, q->len) != q->len)
      FATAL("Short read from '%s'", q->fname); // 打开q->fname，读取到分配的内存中
 
    close(fd);
 
    res = calibrate_case(argv, q, use_mem, 0, 1); // 调用函数calibrate_case校准测试用例
    ck_free(use_mem);
 
    if (stop_soon) return;
 
    if (res == crash_mode || res == FAULT_NOBITS)
      SAYF(cGRA "    len = %u, map size = %u, exec speed = %llu us\n" cRST,
           q->len, q->bitmap_size, q->exec_us);
 
    switch (res) { // 判断res的值
 
      case FAULT_NONE:
 
        if (q == queue) check_map_coverage(); // 如果为头结点，调用check_map_coverage评估覆盖率
 
        if (crash_mode) FATAL("Test case '%s' does *NOT* crash", fn); // 抛出异常
 
        break;
 
      case FAULT_TMOUT:
 
        if (timeout_given) { // 指定了 -t 选项
 
          /* The -t nn+ syntax in the command line sets timeout_given to '2' and
             instructs afl-fuzz to tolerate but skip queue entries that time
             out. */
 
          if (timeout_given > 1) {
            WARNF("Test case results in a timeout (skipping)");
            q->cal_failed = CAL_CHANCES;
            cal_failures++;
            break;
          }
 
          SAYF(... ...);
 
          FATAL("Test case '%s' results in a timeout", fn);
 
        } else {
 
          SAYF(... ...);
 
          FATAL("Test case '%s' results in a timeout", fn);
 
        }
 
      case FAULT_CRASH: 
 
        if (crash_mode) break;
 
        if (skip_crashes) {
          WARNF("Test case results in a crash (skipping)");
          q->cal_failed = CAL_CHANCES;
          cal_failures++;
          break;
        }
 
        if (mem_limit) { // 建议增加内存
 
          SAYF(... ...);
        } else {
 
          SAYF(... ...);
 
        }
 
        FATAL("Test case '%s' results in a crash", fn);
 
      case FAULT_ERROR:
 
        FATAL("Unable to execute target application ('%s')", argv[0]);
 
      case FAULT_NOINST: // 测试用例运行没有路径信息
 
        FATAL("No instrumentation detected");
 
      case FAULT_NOBITS:  // 没有出现新路径，判定为无效路径
 
        useless_at_start++;
 
        if (!in_bitmap && !shuffle_queue)
          WARNF("No new instrumentation output, test case may be useless.");
 
        break;
 
    }
 
    if (q->var_behavior) WARNF("Instrumentation output varies across runs.");
 
    q = q->next; // 读取下一个queue
 
  }
 
  if (cal_failures) {
 
    if (cal_failures == queued_paths)
      FATAL("All test cases time out%s, giving up!",
            skip_crashes ? " or crash" : "");
 
    WARNF("Skipped %u test cases (%0.02f%%) due to timeouts%s.", cal_failures,
          ((double)cal_failures) * 100 / queued_paths,
          skip_crashes ? " or crashes" : "");
 
    if (cal_failures * 5 > queued_paths)
      WARNF(cLRD "High percentage of rejected test cases, check settings!");
 
  }
 
  OKF("All test cases processed.");
 
}

```

总结以上流程：

1.  进入 `while` 循环，遍历 `input` 队列，从队列中取出 `q->fname`，读取文件内容到分配的内存中，然后关闭文件；
2.  调用 `calibrate_case` 函数校准该测试用例；
3.  根据校准的返回值 `res` ，判断错误类型；
4.  打印错误信息，退出。

##### 2.3 calibrate_case 函数

该函数同样为 AFL 的一个关键函数，用于新测试用例的校准，在处理输入目录时执行，以便在早期就发现有问题的测试用例，并且在发现新路径时，评估新发现的测试用例的是否可变。该函数在 `perform_dry_run`，`save_if_interesting`，`fuzz_one`，`pilot_fuzzing`，`core_fuzzing`函数中均有调用。该函数主要用途是初始化并启动 fork server，多次运行测试用例，并用 `update_bitmap_score` 进行初始的 byte 排序。

 

函数控制流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_MBE655SNXYUZVN6.jpg)

 

结合源码进行解读如下：

```
/* Calibrate a new test case. This is done when processing the input directory
   to warn about flaky or otherwise problematic test cases early on; and when
   new paths are discovered to detect variable behavior and so on. */
 
static u8 calibrate_case(char** argv, struct queue_entry* q, u8* use_mem,
                         u32 handicap, u8 from_queue) {
 
  static u8 first_trace[MAP_SIZE]; // 创建 firts_trace[MAP_SIZE]
 
  u8  fault = 0, new_bits = 0, var_detected = 0, hnb = 0,
      first_run = (q->exec_cksum == 0); // 获取执行追踪结果，判断case是否为第一次运行，若为0则表示第一次运行，来自input文件夹
 
  u64 start_us, stop_us;
 
  s32 old_sc = stage_cur, old_sm = stage_max;
  u32 use_tmout = exec_tmout;
  u8* old_sn = stage_name; // 保存原有 stage_cur, stage_max, stage_name
 
  /* Be a bit more generous about timeouts when resuming sessions, or when
     trying to calibrate already-added finds. This helps avoid trouble due
     to intermittent latency. */
 
  if (!from_queue || resuming_fuzz)
    // 如果from_queue为0（表示case不是来自queue）或者resuming_fuzz为1（表示处于resuming sessions）
    use_tmout = MAX(exec_tmout + CAL_TMOUT_ADD,
                    exec_tmout * CAL_TMOUT_PERC / 100); // 提升 use_tmout 的值
 
  q->cal_failed++;
 
  stage_name = "calibration"; // 设置 stage_name
  stage_max  = fast_cal ? 3 : CAL_CYCLES; // 设置 stage_max，新测试用例的校准周期数
 
  /* Make sure the fork server is up before we do anything, and let's not
     count its spin-up time toward binary calibration. */
 
  if (dumb_mode != 1 && !no_fork server && !forksrv_pid)
    init_fork server(argv); // 没有运行在dumb_mode，没有禁用fork server，切forksrv_pid为0时，启动fork server
 
  if (q->exec_cksum) { // 判断是否为新case（如果这个queue不是来自input文件夹）
 
    memcpy(first_trace, trace_bits, MAP_SIZE);
    hnb = has_new_bits(virgin_bits);
    if (hnb > new_bits) new_bits = hnb;
 
  }
 
  start_us = get_cur_time_us();
 
  for (stage_cur = 0; stage_cur < stage_max; stage_cur++) { // 开始执行 calibration stage，总计执行 stage_max 轮
 
    u32 cksum;
 
    if (!first_run && !(stage_cur % stats_update_freq)) show_stats(); // queue不是来自input，第一轮calibration stage执行结束，刷新一次展示界面
 
    write_to_testcase(use_mem, q->len);
 
    fault = run_target(argv, use_tmout);
 
    /* stop_soon is set by the handler for Ctrl+C. When it's pressed,
       we want to bail out quickly. */
 
    if (stop_soon || fault != crash_mode) goto abort_calibration;
 
 
    if (!dumb_mode && !stage_cur && !count_bytes(trace_bits)) {
      // 如果 calibration stage第一次运行，且不在dumb_mode，共享内存中没有任何路径
      fault = FAULT_NOINST;
      goto abort_calibration;
    }
 
    cksum = hash32(trace_bits, MAP_SIZE, HASH_CONST);
 
    if (q->exec_cksum != cksum) {
 
      hnb = has_new_bits(virgin_bits);
      if (hnb > new_bits) new_bits = hnb;
 
      if (q->exec_cksum) { // 不等于exec_cksum，表示第一次运行，或在相同参数下，每次执行，cksum不同，表示是一个路径可变的queue
 
        u32 i;
 
        for (i = 0; i < MAP_SIZE; i++) {
 
          if (!var_bytes[i] && first_trace[i] != trace_bits[i]) {
                    // 从0到MAP_SIZE进行遍历， first_trace[i] != trace_bits[i]，表示发现了可变queue
            var_bytes[i] = 1;
            stage_max    = CAL_CYCLES_LONG;
 
          }
 
        }
 
        var_detected = 1;
 
      } else {
 
        q->exec_cksum = cksum; // q->exec_cksum=0，表示第一次执行queue，则设置计算出来的本次执行的cksum
        memcpy(first_trace, trace_bits, MAP_SIZE);
 
      }
 
    }
 
  }
 
  stop_us = get_cur_time_us();
 
  total_cal_us     += stop_us - start_us;  // 保存所有轮次的总执行时间
  total_cal_cycles += stage_max; // 保存总轮次
 
  /* OK, let's collect some stats about the performance of this test case.
     This is used for fuzzing air time calculations in calculate_score(). */
 
  q->exec_us     = (stop_us - start_us) / stage_max; // 单次执行时间的平均值
  q->bitmap_size = count_bytes(trace_bits); // 最后一次执行所覆盖的路径数
  q->handicap    = handicap;
  q->cal_failed  = 0;
 
  total_bitmap_size += q->bitmap_size; // 加上queue所覆盖的路径数
  total_bitmap_entries++;
 
  update_bitmap_score(q);
 
  /* If this case didn't result in new output from the instrumentation, tell
     parent. This is a non-critical problem, but something to warn the user
     about. */
 
  if (!dumb_mode && first_run && !fault && !new_bits) fault = FAULT_NOBITS;
 
abort_calibration:
 
  if (new_bits == 2 && !q->has_new_cov) {
    q->has_new_cov = 1;
    queued_with_cov++;
  }
 
  /* Mark variable paths. */
 
  if (var_detected) { // queue是可变路径
 
    var_byte_count = count_bytes(var_bytes);
 
    if (!q->var_behavior) {
      mark_as_variable(q);
      queued_variable++;
    }
 
  }
 
  // 恢复之前的stage值
  stage_name = old_sn;
  stage_cur  = old_sc;
  stage_max  = old_sm;
 
  if (!first_run) show_stats();
 
  return fault;
 
}

```

总结以上过程如下：

1.  进行参数设置，包括当前阶段 `stage_cur`，阶段名称 `stage_name`，新比特 `new_bit 等初始化;
2.  参数 `from_queue`，判断 case 是否在队列中，且是否处于 resuming session， 以此设置时间延迟。testcase 参数 `q->cal_failed` 加 1， 是否校准失败参数加 1；
3.  判断是否已经启动 fork server ，调用函数 `init_fork server()` ；
4.  拷贝 `trace_bits` 到 `first_trace` ，调用 `get_cur_time_us()` 获取开始时间 `start_us`；
5.  进入 loop 循环，该 loop 循环多次执行 testcase，循环次数为 8 次或者 3 次；
6.  调用 `write_to_testcase` 将修改后的数据写入文件进行测试。如果 `use_stdin` 被清除，取消旧文件链接并创建一个新文件。否则，缩短`prog_in_fd` ；
7.  调用 `run_target` 通知 fork server 可以开始 fork 并 fuzz；
8.  调用 `hash32` 校验此次运行的 `trace_bits`，检查是否出现新的情况；
9.  将本次运行的出现 `trace_bits` 哈希和本次 testcase 的 `q->exec_cksum`对比。如果发现不同，则调用 `has_new_bits`函数和总表`virgin_bits` 对比；
10.  判断 `q->exec_cksum` 是否为 0，不为 0 说明不是第一次执行。后面运行如果和前面第一次 `trace_bits` 结果不同，则需要多运行几次；
11.  loop 循环结束；
12.  收集一些关于测试用例性能的统计数据。比如执行时间延迟，校准错误，bitmap 大小等等；
13.  调用 `update_bitmap_score()` 函数对测试用例的每个 byte 进行排序，用一个 `top_rate[]` 维护最佳入口；
14.  如果没有从检测中得到 `new_bit`，则告诉父进程，这是一个无关紧要的问题，但是需要提醒用户。  
    总结：calibratecase 函数到此为止，该函数主要用途是 init_fork server；将 testcase 运行多次；用 update_bitmap_score 进行初始的 byte 排序。

##### 2.4 init_forkserve 函数

AFL 的 fork server 机制避免了多次执行 `execve()` 函数的多次调用，只需要调用一次然后通过管道发送命令即可。该函数主要用于启动 APP 和它的 fork server。函数整体控制流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_JPCNP5VMMBS6P9G.jpg)

 

结合源码梳理一下函数流程：

```
EXP_ST void init_fork server(char** argv) {
 
  static struct itimerval it;
  int st_pipe[2], ctl_pipe[2];
  int status;
  s32 rlen;
 
  ACTF("Spinning up the fork server...");
 
  if (pipe(st_pipe) || pipe(ctl_pipe)) PFATAL("pipe() failed");
  // 检查 st_pipe 和ctl_pipe，在父子进程间进行管道通信，一个用于传递状态，一个用于传递命令
 
  forksrv_pid = fork();
  // fork进程出一个子进程
  // 如果fork成功，则现在有父子两个进程
  // 此时的父进程为fuzzer，子进程则为目标程序进程，也是将来的fork server
 
  if (forksrv_pid < 0) PFATAL("fork() failed"); // fork失败
 
  // 子进程和父进程都会向下执行，通过pid来使父子进程执行不同的代码
 
  if (!forksrv_pid) { // 子进程执行
 
    struct rlimit r;
 
    /* 中间省略针对OpenBSD的特殊处理 */
 
    ... ...
 
    /* Isolate the process and configure standard descriptors. If out_file is
       specified, stdin is /dev/null; otherwise, out_fd is cloned instead. */
 
    // 创建守护进程
    setsid();
 
    // 重定向文件描述符1和2到dev_null_fd
    dup2(dev_null_fd, 1);
    dup2(dev_null_fd, 2);
 
    // 如果指定了out_file，则文件描述符0重定向到dev_null_fd，否则重定向到out_fd
    if (out_file) {
 
      dup2(dev_null_fd, 0);
 
    } else {
 
      dup2(out_fd, 0);
      close(out_fd);
 
    }
 
    /* Set up control and status pipes, close the unneeded original fds. */
    // 设置控制和状态管道，关闭不需要的一些文件描述符
 
    if (dup2(ctl_pipe[0], FORKSRV_FD) < 0) PFATAL("dup2() failed");
    if (dup2(st_pipe[1], FORKSRV_FD + 1) < 0) PFATAL("dup2() failed");
 
    close(ctl_pipe[0]);
    close(ctl_pipe[1]);
    close(st_pipe[0]);
    close(st_pipe[1]);
 
    close(out_dir_fd);
    close(dev_null_fd);
    close(dev_urandom_fd);
    close(fileno(plot_file));
 
    /* This should improve performance a bit, since it stops the linker from
       doing extra work post-fork(). */
 
    // 如果没有设置延迟绑定，则进行设置，不使用缺省模式
    if (!getenv("LD_BIND_LAZY")) setenv("LD_BIND_NOW", "1", 0);
 
    /* Set sane defaults for ASAN if nothing else specified. */
 
    // 设置环境变量ASAN_OPTIONS，配置ASAN相关
    setenv("ASAN_OPTIONS", "abort_on_error=1:"
                           "detect_leaks=0:"
                           "symbolize=0:"
                           "allocator_may_return_null=1", 0);
 
    /* MSAN is tricky, because it doesn't support abort_on_error=1 at this
       point. So, we do this in a very hacky way. */
 
    // MSAN相关
    setenv("MSAN_OPTIONS", "exit_code=" STRINGIFY(MSAN_ERROR) ":"
                           "symbolize=0:"
                           "abort_on_error=1:"
                           "allocator_may_return_null=1:"
                           "msan_track_origins=0", 0);
 
        /* 带参数执行目标程序，报错才返回
             execv()会替换原有进程空间为目标程序，所以后续执行的都是目标程序。
             第一个目标程序会进入__afl_maybe_log里的__afl_fork_wait_loop，并充当fork server。
             在整个过程中，每次要fuzz一次目标程序，都会从这个fork server再fork出来一个子进程去fuzz。
             因此可以看作是三段式：fuzzer -> fork server -> target子进程
        */
    execv(target_path, argv);
 
    /* Use a distinctive bitmap signature to tell the parent about execv()
       falling through. */
 
    // 告诉父进程执行失败，结束子进程
    *(u32*)trace_bits = EXEC_FAIL_SIG;
    exit(0);
 
  }
 
  /* Close the unneeded endpoints. */
 
  close(ctl_pipe[0]);
  close(st_pipe[1]);
 
  fsrv_ctl_fd = ctl_pipe[1]; // 父进程只能发送命令
  fsrv_st_fd  = st_pipe[0];  // 父进程只能读取状态
 
  /* Wait for the fork server to come up, but don't wait too long. */
    // 在一定时间内等待fork server启动
  it.it_value.tv_sec = ((exec_tmout * FORK_WAIT_MULT) / 1000);
  it.it_value.tv_usec = ((exec_tmout * FORK_WAIT_MULT) % 1000) * 1000;
 
  setitimer(ITIMER_REAL, &it, NULL);
 
  rlen = read(fsrv_st_fd, &status, 4); // 从管道里读取4字节数据到status
 
  it.it_value.tv_sec = 0;
  it.it_value.tv_usec = 0;
 
  setitimer(ITIMER_REAL, &it, NULL);
 
  /* If we have a four-byte "hello" message from the server, we're all set.
     Otherwise, try to figure out what went wrong. */
 
  if (rlen == 4) { // 以读取的结果判断fork server是否成功启动
    OKF("All right - fork server is up.");
    return;
  }
 
  // 子进程启动失败的异常处理相关
  if (child_timed_out)
    FATAL("Timeout while initializing fork server (adjusting -t may help)");
 
  if (waitpid(forksrv_pid, &status, 0) <= 0)
    PFATAL("waitpid() failed");
 
   ... ...
 
}

```

我们结合 fuzzer 对该函数的调用来梳理完整的流程如下：

 

启动目标程序进程后，目标程序会运行一个 fork server，fuzzer 自身并不负责 fork 子进程，而是通过管道与 fork server 通信，由 fork server 来完成 fork 以及继续执行目标程序的操作。

 

![](https://bbs.pediy.com/upload/attach/202109/779730_HNKV49C35TQ7YG5.jpg)

 

对于 fuzzer 和目标程序之间的通信状态我们可以通过下图来梳理：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_WAUT5W2VZAJ3WNU.jpg)

 

结合前面的插桩部分一起来看：

 

首先，`afl-fuzz` 会创建两个管道：状态管道和控制管道，然后执行目标程序。此时的目标程序的 `main()` 函数已经被插桩，程序控制流进入 `__afl_maybe_log` 中。如果 fuzz 是第一次执行，则此时的程序就成了 fork server 们之后的目标程序都由该 fork server 通过 fork 生成子进程来运行。fuzz 进行过程中，fork server 会一直执行 fork 操作，并将子进程的结束状态通过状态管道传递给 `afl-fuzz`。

 

（对于 fork server 的具体操作，在前面插桩部分时已经根据源码进行了说明，可以回顾一下。）

##### 2.5 run_target 函数

该函数主要执行目标应用程序，并进行超时监控，返回状态信息，被调用的程序会更新 `trace_bits[]` 。

 

结合源码进行解释：

```
static u8 run_target(char** argv, u32 timeout) {
 
  static struct itimerval it;
  static u32 prev_timed_out = 0;
  static u64 exec_ms = 0;
 
  int status = 0;
  u32 tb4;
 
  child_timed_out = 0;
 
  /* After this memset, trace_bits[] are effectively volatile, so we
     must prevent any earlier operations from venturing into that
     territory. */
 
  memset(trace_bits, 0, MAP_SIZE); // 将trace_bits全部置0，清空共享内存
  MEM_BARRIER();
 
  /* If we're running in "dumb" mode, we can't rely on the fork server
     logic compiled into the target program, so we will just keep calling
     execve(). There is a bit of code duplication between here and
     init_fork server(), but c'est la vie. */
 
  if (dumb_mode == 1 || no_fork server) { // 如果是dumb_mode模式且没有fork server
 
    child_pid = fork(); // 直接fork出一个子进程
 
    if (child_pid < 0) PFATAL("fork() failed");
 
    if (!child_pid) {
 
     ... ...
 
      /* Isolate the process and configure standard descriptors. If out_file is
         specified, stdin is /dev/null; otherwise, out_fd is cloned instead. */
 
      setsid();
 
      dup2(dev_null_fd, 1);
      dup2(dev_null_fd, 2);
 
      if (out_file) {
 
        dup2(dev_null_fd, 0);
 
      } else {
 
        dup2(out_fd, 0);
        close(out_fd);
 
      }
 
      /* On Linux, would be faster to use O_CLOEXEC. Maybe TODO. */
 
      close(dev_null_fd);
      close(out_dir_fd);
      close(dev_urandom_fd);
      close(fileno(plot_file));
 
      /* Set sane defaults for ASAN if nothing else specified. */
 
      setenv("ASAN_OPTIONS", "abort_on_error=1:"
                             "detect_leaks=0:"
                             "symbolize=0:"
                             "allocator_may_return_null=1", 0);
 
      setenv("MSAN_OPTIONS", "exit_code=" STRINGIFY(MSAN_ERROR) ":"
                             "symbolize=0:"
                             "msan_track_origins=0", 0);
 
      execv(target_path, argv); // 让子进程execv执行目标程序
 
      /* Use a distinctive bitmap value to tell the parent about execv()
         falling through. */
 
      *(u32*)trace_bits = EXEC_FAIL_SIG; // execv执行失败，写入 EXEC_FAIL_SIG
      exit(0);
 
    }
 
  } else {
 
    s32 res;
 
    /* In non-dumb mode, we have the fork server up and running, so simply
       tell it to have at it, and then read back PID. */
 
    // 如果并不是处在dumb_mode模式，说明fork server已经启动了，我们只需要进行
    // 控制管道的写和状态管道的读即可
    if ((res = write(fsrv_ctl_fd, &prev_timed_out, 4)) != 4) {
 
      if (stop_soon) return 0;
      RPFATAL(res, "Unable to request new process from fork server (OOM?)");
 
    }
 
    if ((res = read(fsrv_st_fd, &child_pid, 4)) != 4) {
 
      if (stop_soon) return 0;
      RPFATAL(res, "Unable to request new process from fork server (OOM?)");
 
    }
 
    if (child_pid <= 0) FATAL("Fork server is misbehaving (OOM?)");
 
  }
 
  /* Configure timeout, as requested by user, then wait for child to terminate. */
 
 
// 配置超时，等待子进程结束
  it.it_value.tv_sec = (timeout / 1000);
  it.it_value.tv_usec = (timeout % 1000) * 1000;
 
  setitimer(ITIMER_REAL, &it, NULL);
 
  /* The SIGALRM handler simply kills the child_pid and sets child_timed_out. */
 
  if (dumb_mode == 1 || no_fork server) {
 
    if (waitpid(child_pid, &status, 0) <= 0) PFATAL("waitpid() failed");
 
  } else {
 
    s32 res;
 
    if ((res = read(fsrv_st_fd, &status, 4)) != 4) {
 
      if (stop_soon) return 0;
      RPFATAL(res, "Unable to communicate with fork server (OOM?)");
 
    }
 
  }
 
  if (!WIFSTOPPED(status)) child_pid = 0;
 
  getitimer(ITIMER_REAL, &it);
  exec_ms = (u64) timeout - (it.it_value.tv_sec * 1000 +
                             it.it_value.tv_usec / 1000); // 计算执行时间
 
  it.it_value.tv_sec = 0;
  it.it_value.tv_usec = 0;
 
  setitimer(ITIMER_REAL, &it, NULL);
 
  total_execs++;
 
  /* Any subsequent operations on trace_bits must not be moved by the
     compiler below this point. Past this location, trace_bits[] behave
     very normally and do not have to be treated as volatile. */
 
  MEM_BARRIER();
 
  tb4 = *(u32*)trace_bits;
 
  // 分别执行64和32位下的classify_counts，设置trace_bits所在的mem
#ifdef WORD_SIZE_64
  classify_counts((u64*)trace_bits);
#else
  classify_counts((u32*)trace_bits);
#endif /* ^WORD_SIZE_64 */
 
  prev_timed_out = child_timed_out;
 
  /* Report outcome to caller. */
 
  if (WIFSIGNALED(status) && !stop_soon) {
 
    kill_signal = WTERMSIG(status);
 
    if (child_timed_out && kill_signal == SIGKILL) return FAULT_TMOUT;
 
    return FAULT_CRASH;
 
  }
 
  /* A somewhat nasty hack for MSAN, which doesn't support abort_on_error and
     must use a special exit code. */
 
  if (uses_asan && WEXITSTATUS(status) == MSAN_ERROR) {
    kill_signal = 0;
    return FAULT_CRASH;
  }
 
  if ((dumb_mode == 1 || no_fork server) && tb4 == EXEC_FAIL_SIG)
    return FAULT_ERROR;
 
  /* It makes sense to account for the slowest units only if the testcase was run
  under the user defined timeout. */
  if (!(timeout > exec_tmout) && (slowest_exec_ms < exec_ms)) {
    slowest_exec_ms = exec_ms;
  }
 
  return FAULT_NONE;
 
}

```

##### 2.6 update_bitmap_score 函数

当我们发现一个新路径时，需要判断发现的新路径是否更 “favorable”，也就是是否包含最小的路径集合能遍历到所有 bitmap 中的位，并在之后的 fuzz 过程中聚焦在这些路径上。

 

以上过程的第一步是为 bitmap 中的每个字节维护一个 `top_rated[]` 的列表，这里会计算究竟哪些位置是更 “合适” 的，该函数主要实现该过程。

 

函数的控制流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_8MEFWMMRJX7S2QJ.jpg)

 

结合源码进行解释：

```
static void update_bitmap_score(struct queue_entry* q) {
 
  u32 i;
  u64 fav_factor = q->exec_us * q->len;
  // 首先计算case的fav_factor，计算方法是执行时间和样例大小的乘积
 
  /* For every byte set in trace_bits[], see if there is a previous winner,
     and how it compares to us. */
 
  for (i = 0; i < MAP_SIZE; i++) // 遍历trace_bits数组
 
    if (trace_bits[i]) { // 不为0，表示已经被覆盖到的路径
 
       if (top_rated[i]) { // 检查top_rated是否存在
 
         /* Faster-executing or smaller test cases are favored. */
 
         if (fav_factor > top_rated[i]->exec_us * top_rated[i]->len) continue; // 判断哪个计算结果更小
         // 如果top_rated[i]的更小，则代表它的更优，不做处理，继续遍历下一个路径；
         // 如果q的更小，就执行以下代码：
 
         /* Looks like we're going to win. Decrease ref count for the
            previous winner, discard its trace_bits[] if necessary. */
 
         if (!--top_rated[i]->tc_ref) {
           ck_free(top_rated[i]->trace_mini);
           top_rated[i]->trace_mini = 0;
         }
       }
       /* Insert ourselves as the new winner. */
 
       top_rated[i] = q; // 设置为当前case
       q->tc_ref++;
 
       if (!q->trace_mini) { // 为空
         q->trace_mini = ck_alloc(MAP_SIZE >> 3);
         minimize_bits(q->trace_mini, trace_bits);
       }
 
       score_changed = 1;
 
     }
 
}

```

#### 3. 主循环

##### 3.1 主循环之前

###### 3.1.1 cull_queue 函数

 

在前面讨论的关于 case 的 `top_rated` 的计算中，还有一个机制是检查所有的 `top_rated[]` 条目，然后顺序获取之前没有遇到过的 byte 的对比分数低的 “获胜者” 进行标记，标记至少会维持到下一次运行之前。在所有的 fuzz 步骤中，“favorable”的条目会获得更多的执行时间。

 

函数的控制流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_39HN95B9JY9HV24.jpg)

 

结合源码解析如下：

```
static void cull_queue(void) {
 
  struct queue_entry* q;
  static u8 temp_v[MAP_SIZE >> 3];
  u32 i;
 
  if (dumb_mode || !score_changed) return;  // 如果处于dumb模式或者score没有发生变化（top_rated没有发生变化），直接返回
 
  score_changed = 0;
 
  memset(temp_v, 255, MAP_SIZE >> 3);
  // 设置temp_v大小为MAP_SIZE>>3，初始化为0xff，全1，表示还没有被覆盖到，为0表示被覆盖到了。
 
  queued_favored  = 0;
  pending_favored = 0;
 
  q = queue;
 
  while (q) { // 进行队列遍历
    q->favored = 0; // 所有元素的favored均设置为0
    q = q->next;
  }
 
  /* Let's see if anything in the bitmap isn't captured in temp_v.
     If yes, and if it has a top_rated[] contender, let's use it. */
 
  // i从0到MAP_SIZE进行迭代，筛选出一组队列条目，它们可以覆盖到所有现在已经覆盖的路径
  for (i = 0; i < MAP_SIZE; i++)
    if (top_rated[i] && (temp_v[i >> 3] & (1 << (i & 7)))) {
 
      u32 j = MAP_SIZE >> 3;
 
      /* Remove all bits belonging to the current entry from temp_v. */
 
      // 从temp_v中，移除所有属于当前current-entry的byte，也就是这个testcase触发了多少path就给tempv标记上
      while (j--)
        if (top_rated[i]->trace_mini[j])
          temp_v[j] &= ~top_rated[i]->trace_mini[j];
 
      top_rated[i]->favored = 1;
      queued_favored++;
 
      if (!top_rated[i]->was_fuzzed) pending_favored++;
 
    }
 
  q = queue;
 
  while (q) { // 遍历队列，不是favored的case（冗余的测试用例）被标记成redundant_edges
    mark_as_redundant(q, !q->favored); // 位置在/queue/.state/redundent_edges中
    q = q->next;
  }
 
}

```

这里根据网上公开的一个例子来理解该过程：

 

现假设有如下 tuple 和 seed 信息：

*   **tuple**: t0, t1, t2, t3, t4
    
*   **seed**: s0, s1, s2
    
*   初始化 `temp_v = [1,1,1,1,1]`
*   s1 可覆盖 t2, t3，s2 覆盖 t0, t1, t4，并且 top_rated[0] = s2，top_rated[2]=s1

将按照如下过程进行筛选和判断：

1.  首先判断 temp_v[0]=1，说明 t0 没有被覆盖；
2.  top_rated[0] 存在 (s2) -> 判断 s2 可以覆盖的范围 -> `trace_mini = [1,1,0,0,1]`；
3.  更新 `temp_v=[0,0,1,1,0]`， 标记 s2 为 "favored"；
4.  继续判断 temp_v[1]=0，说明 t1 此时已经被覆盖，跳过；
5.  继续判断 temp_v[2]=1，说明 t2 没有被覆盖；
6.  top_rated[2] 存在 (s1) -> 判断 s1 可以覆盖的范围 -> `trace_mini=[0,0,1,1,0]`；
7.  更新 `temp_v=[0,0,0,0,0]`，标记 s1 为 "favored"；
8.  此时所有 tuple 都被覆盖，具备 "favored'标记的为 s1, s2，过程结束。

###### 3.1.2 show_init_stats 函数

 

进入主循环前的准备工作使用的函数之一，主要作用为在处理输入目录的末尾显示统计信息，警告信息以及硬编码的常量；

###### 3.1.3 find_start_position 函数

 

进入主循环前的准备工作使用的函数之一，主要作用为在 resume 时，尝试查找要开始的队列的位置。

###### 3.1.4 write_stats_file 函数

 

也是准备工作函数之一，主要作用为更新统计信息文件以进行无人值守的监视。

###### 3.1.5 save_auto 函数

 

该函数主要保存自动生成的 extras。

##### 3.2 主循环

这里是 seed 变异的主循环处理过程，我们将结合流程图和源码进行详细解读。

 

主循环的控制流程图如下（将 while 部分单独设置为了一个函数，只看循环部分即可）：

 

![](https://bbs.pediy.com/upload/attach/202109/779730_A7X44DUFE5BR49N.jpg)

 

主循环源码：

```
while (1) {
 
  u8 skipped_fuzz;
 
  cull_queue(); // 调用cull_queue进行队列精简
 
  if (!queue_cur) { // 如果queue_cure为空（所有queue都被执行完一轮）
 
    queue_cycle++; // 计数器，所有queue执行的轮数
    current_entry     = 0;
    cur_skipped_paths = 0;
    queue_cur         = queue; // 准备开始新一轮fuzz
 
    while (seek_to) { // 如果seek_to不为空
      current_entry++;
      seek_to--;
      queue_cur = queue_cur->next; // 从seek_to指定的queue项开始执行
    }
 
    show_stats(); // 刷新展示界面
 
    if (not_on_tty) {
      ACTF("Entering queue cycle %llu.", queue_cycle);
      fflush(stdout);
    }
 
    /* If we had a full queue cycle with no new finds, try
         recombination strategies next. */
 
    if (queued_paths == prev_queued) { // 如果一轮执行后queue中的case数与执行前一样，表示没有发现新的case
 
      if (use_splicing) cycles_wo_finds++; else use_splicing = 1; // 是否使用splice进行case变异
 
    } else cycles_wo_finds = 0;
 
    prev_queued = queued_paths;
 
    if (sync_id && queue_cycle == 1 && getenv("AFL_IMPORT_FIRST"))
      sync_fuzzers(use_argv);
 
  }
 
  skipped_fuzz = fuzz_one(use_argv); // 对queue_cur进行一次测试
 
  if (!stop_soon && sync_id && !skipped_fuzz) {
   // 如果skipped_fuzz为0且存在sync_id，表示要进行一次sync
 
    if (!(sync_interval_cnt++ % SYNC_INTERVAL))
      sync_fuzzers(use_argv);
 
  }
  if (!stop_soon && exit_1) stop_soon = 2;
 
  if (stop_soon) break;
 
  queue_cur = queue_cur->next;
  current_entry++;
 
}

```

总结以上内容，该处该过程整体如下：

1.  判断 `queue_cur` 是否为空，如果是则表示已经完成队列遍历，初始化相关参数，重新开始一轮；
2.  找到 queue 入口的 case，直接跳到该 case；
3.  如果一整个队列循环都没新发现，尝试重组策略；
4.  调用关键函数 `fuzz_one()` 对该 case 进行 fuzz；
5.  上面的变异完成后，AFL 会对文件队列的下一个进行变异处理。当队列中的全部文件都变异测试后，就完成了一个”cycle”，这个就是 AFL 状态栏右上角的”cycles done”。而正如 cycle 的意思所说，整个队列又会从第一个文件开始，再次进行变异，不过与第一次变异不同的是，这一次就不需要再进行 “deterministic fuzzing” 了。如果用户不停止 AFL，seed 文件将会一遍遍的变异下去。

##### 3.3 主循环后

###### 3.3.1 fuzz_one 函数

 

该函数源码在 1000 多行，出于篇幅原因，我们简要介绍函数的功能。但强烈建议通读该函数源码，

 

函数主要是从 queue 中取出 entry 进行 fuzz，成功返回 0，跳过或退出的话返回 1。

 

整体过程：

1.  根据是否有 `pending_favored` 和`queue_cur`的情况，按照概率进行跳过；有`pending_favored`, 对于已经 fuzz 过的或者 non-favored 的有 99% 的概率跳过；无 pending_favored，95% 跳过 fuzzed&non-favored，75% 跳过 not fuzzed&non-favored，不跳过 favored；
2.  假如当前项有校准错误，并且校准错误次数小于 3 次，那么就用 calibrate_case 进行测试；
3.  如果测试用例没有修剪过，那么调用函数 trim_case 对测试用例进行修剪；
4.  修剪完毕之后，使用 calculate_score 对每个测试用例进行打分；
5.  如果该 queue 已经完成 deterministic 阶段，则直接跳到 havoc 阶段；
6.  deterministic 阶段变异 4 个 stage，变异过程中会多次调用函数 common_fuzz_stuff 函数，保存 interesting 的种子：
    *   bitflip，按位翻转，1 变为 0，0 变为 1
    *   arithmetic，整数加 / 减算术运算
    *   interest，把一些特殊内容替换到原文件中
    *   dictionary，把自动生成或用户提供的 token 替换 / 插入到原文件中
    *   havoc，中文意思是 “大破坏”，此阶段会对原文件进行大量变异。
    *   splice，中文意思是 “绞接”，此阶段会将两个文件拼接起来得到一个新的文件。
7.  该轮完成。

这里涉及到 AFL 中的变异策略，不在本次的讨论中，感兴趣的小伙伴可以结合源码自行进行研究。

###### 3.3.2 sync_fuzzers 函数

 

该函数的主要作用是进行 queue 同步，先读取有哪些 fuzzer 文件夹，然后读取其他 fuzzer 文件夹下的 queue 文件夹中的测试用例，然后以此执行。如果在执行过程中，发现这些测试用例可以触发新路径，则将测试用例保存到自己的 queue 文件夹中，并将最后一个同步的测试用例的 id 写入到 `.synced/fuzzer文件夹名` 文件中，避免重复运行。

[](#四、总结)四、总结
-------------

分析完源码，可以感受到，AFL 遵循的基本原则是简单有效，没有进行过多的复杂的优化，能够针对 fuzz 领域的痛点，对症下药，拒绝花里胡哨，给出切实可行的解决方案，在漏洞挖掘领域的意义的确非同凡响。后期的很多先进的 fuzz 工具基本沿用了 AFL 的思路，甚至目前为止已基本围绕 AFL 建立了 “生态圈”，涉及到多个平台、多种漏洞挖掘对象，对于安全研究员来说实属利器，值得从事 fuzz 相关工作的研究员下足功夫去体会 AFL 的精髓所在。

 

考虑到篇幅限制，我们没有对 AFL 中的变异策略进行源码说明，实属遗憾。如果有机会，将新开文章详细介绍 AFL 的变异策略和源码分析。

[](#五、参考文献)五、参考文献
-----------------

1.  http://lcamtuf.coredump.cx/afl/
2.  https://eternalsakura13.com/2020/08/23/afl/
3.  https://bbs.pediy.com/thread-265936.htm
4.  https://bbs.pediy.com/thread-249912.htm#msg_header_h3_3

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 1 天前 被有毒编辑 ，原因：

[#自动化挖掘](forum-150-1-155.htm) [#Fuzz](forum-150-1-157.htm) [#Linux](forum-150-1-161.htm)