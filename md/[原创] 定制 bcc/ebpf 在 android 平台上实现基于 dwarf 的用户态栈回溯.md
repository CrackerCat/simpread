> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274546.htm)

> [原创] 定制 bcc/ebpf 在 android 平台上实现基于 dwarf 的用户态栈回溯

ebpf 是个非常强大的内核级跟踪机制，不仅可以用于性能分析，在逆向分析中也是非常强大的工具，对此介绍性的文章可以参照 evilpan 大佬的 [Linux 内核监控在 Android 攻防中的应用](https://bbs.pediy.com/thread-271043.htm#msg_header_h1_4)一文。

 

而 bcc 就是其中最著名的上层封装框架，本文就是提供一种定制 bcc 源码并在 android 平台上实现基于 dwarf 的用户态栈回溯的方案，并讨论其中涉及的代码原理及机制。

 

首先引出如下问题: bcc/ebpf 提供了什么形式的栈回溯? 这种栈回溯有什么缺点它是基于什么原理? 而基于 dwarf 的栈回溯有什么优点?

 

本文使用的环境是 pixel 3XL, linux kernel 4.9，aosp 10。由于 ebpf 是个发展迅速的技术，越新的内核功能越丰富，4.9 内核其实算是比较旧的内核了。

 

本文采用的 bcc 代码为目前最新的 (2022 年 8 月 11 日 release) v0.25.0 版本。

一. bcc 环境:
----------

bcc 的编译运行环境采用的是 debootstrap 自制 debian 10 arm64 镜像, 并且使用 adeb 工具将镜像 push 到手机中，在手机中通过 chroot 方式运行一个轻量级容器化环境, 在这个环境中安装编译所需软件。

 

编译 ebpf 程序需要 kheader, 如果没有会报错:  
`Unable to find kernel headers. Try rebuilding kernel with CONFIG_IKHEADERS=m (module) or installing the kernel development package for your running kernel version.`  
官方推荐在编译内核的时候加上 CONFIG_IKHEADERS 选项，但是这个选项在 4.9 内核上还没有。其实在官方提供的内核 android-msm-crosshatch-4.9 编译完以后会出生 kheader 文件:  
`out/android-msm-pixel-4.9/dist/kernel-headers.tar.gz`  
将这个文件 push 到手机上, 然后解压至 / tmp/kheaders-`uname -r` 目录即可, 因为 bcc 程序会读取这个目录做为 kheaders, 具体的逻辑在 loader.cc 调用的 get_proc_kheaders 函数中。

二. bcc 提供的用户态栈回溯:
-----------------

ebpf 可以很方便的打印出系统调用参数, 如果能同时打印出堆栈信息那么对排查问题和逆向分析来说就非常有用了, 比如得知是哪个代码路径打开了某个文件, 哪个 so 建立了网络连接, 连接到哪个地址, 端口号多少等等。

 

其实 bcc 程序 trace.py 提供了选项 - U 用于打印用户态堆栈。

 

先准备一个小程序，在 libnative-lib.so 中有如下代码:

```
void func_a(int arg) {
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "func_a: %d", arg);
    func_b("test");
}
 
void func_b(std::string arg) {
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "func_b: %s", arg.c_str());
    func_c(10.0f);
}
 
void func_c(float arg) {
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "func_c: %f", arg);
    func_d(true);
}
 
void func_d(bool arg) {
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "func_d: %d", arg);
    func_e(10.0f);
}
 
void func_e(double arg) {
    int fd;
    fd = open("/data/local/tmp/test", O_RDONLY);
    __android_log_print(ANDROID_LOG_INFO, LOG_TAG, "args is %f, fd is : %d", arg, fd);
    close(fd);
}

```

程序很简单，就是调用 pthread_create 创建一个线程并且从 func_a 一路调用到 func_e，在 func_e 中打开一个文件`/data/local/tmp/test`。

 

调用 trace.py 跟踪该程序 (uid 10121) 的 do_sys_open:  
`python3 trace --uid 10121 'do_sys_open "%s", arg2@user' -U --address -v`

 

可以看到能打印出一些栈调用信息，但是结果不是很理想，比如:

```
5791   5791   com.mypack.test   do_sys_open      b'/proc/self/task/12421/comm'
              74f7854388 __openat+0x8 [libc.so]
              7465f59d28 [unknown]
              7fd4be9740 [unknown]
 
5791    5843    test_unwind     do_sys_open      b'/data/local/tmp/test'
              7e1b223388 __openat+0x8 [libc.so]
              7d8a34ec58 [unknown] [base.apk]
              7d8a34ec00 [unknown] [base.apk]
              7d8a34eac0 [unknown] [base.apk]
              7d8a34ea14 [unknown] [base.apk]
              7d8a34edb8 [unknown] [base.apk]
              7e1b237b44 __pthread_start(void*)+0x28 [libc.so]
              7e1b1da2d4 __start_thread+0x44 [libc.so]

```

栈回溯其实包括两部分: 获取调用堆栈信息以及符号化  
第一个打印堆栈大部分信息都缺失了。  
第二个打印是 func_a 函数发起的打开文件 / data/local/tmp/test 的堆栈, 堆栈信息似乎是好的, 但是符号化却出了问题。可以看到符号信息全是 unknown, 且调用位置是位于 base.apk 中, 并没有显示出 libnative-lib.so, 且堆栈前面的地址是运行地址, 如果能显示出在 so 中的偏移地址就好了, 这样才有利于使用 ida 分析。  
原因在哪呢? 这就得去源码里找答案了。

三. bcc 程序运行原理
-------------

先来看一下 Brendan Gregg 网站的框架图:  
![](https://bbs.pediy.com/upload/attach/202209/607812_GM7P2NZMSKA6D89.png)  
bcc 提供的是 python 语言的前端, python 会和后边的 c/c++ 语言的后端进行通信, 调用 llvm 编译生成 ebpf 程序字节码, 调用 bpf 系统调用经过验证器验证以后会经过 jit 存放编译后的可执行代码, ebpf 程序想得到执行还需要挂载到内核 trace/event 上, 事件发生时 ebpf 程序得到执行, ebpf 程序可以通过 bpf map 或者 perf_output 和用户空间通信, ebpf 程序执行完以后用户空间读取生成的数据, 因此用户空间是异步读取。

 

其中 ebpf 所使用的 perf_output 是本文实现基于 dwarf 的用户态栈回溯的关键所在。

 

来看 bcc 文档`docs/tutorial_bcc_python_developer.md`中的`Lesson 7. hello_perf_output.py`这个程序:

```
from bcc import BPF
 
# define BPF program
prog = """
#include // define output data structure in C
struct data_t {
    u32 pid;
    u64 ts;
    char comm[TASK_COMM_LEN];
};
BPF_PERF_OUTPUT(events);
 
int hello(struct pt_regs *ctx) {
    struct data_t data = {};
 
    data.pid = bpf_get_current_pid_tgid();
    data.ts = bpf_ktime_get_ns();
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
 
    events.perf_submit(ctx, &data, sizeof(data));
 
    return 0;
}
"""
 
# load BPF program
b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_)
 
# header
print("%-18s %-16s %-6s %s" % ("TIME(s)", "COMM", "PID", "MESSAGE"))
 
# process event
start = 0
def print_event(cpu, data, size):
    global start
    event = b["events"].event(data)
    if start == 0:
            start = event.ts
    time_s = (float(event.ts - start)) / 1000000000
    print("%-18.9f %-16s %-6d %s" % (time_s, event.comm, event.pid,
        "Hello, perf_output!"))
 
# loop with callback to print_event
b["events"].open_perf_buffer(print_event)
while 1:
    b.perf_buffer_poll() 
```

这个程序的作用是跟踪 clone 系统调用, 并且打印出发生调用的时间, 调用者的进程名和 pid。

 

注意 prog 变量括起来的字符串代码是 c 语言代码, 它之后会被 llvm 编译, 除此之外全是 python 代码. 其中 c 语言的函数 hello 是最终会在内核中执行的 ebpf 程序, 它会以 kprobes 形式挂载到系统调用函数 sys_clone。

 

prog 变量中除了 hello 函数的其他代码属于帮助性代码, 用于将 ebpf 输出和 python 代码获取的输入数据结构统一起来, 这些代码并不会被包含在 ebpf 最终加载的程序中去  
比如上面的`BPF_PERF_OUTPUT(events)`就定义了用于在 ebpf 程序中输出的 perf 缓冲区名称, python 程序就可以使用 b["events"] 关联同一个缓冲区, 并从中获取结构化的数据 struct data_t。

 

ebpf 程序需要包括相应的头文件, 这是为了调用 clang 的时候能编译通过, 也是为了保证 ebpf 可以加载到 api 一致的内核中去。

 

其实上面 hello 函数中的 events.perf_submit(ctx, &data, sizeof(data)) 并不是一个合法的 ebpf 程序, ebpf 执行的字节码是严格受限的:

1.  只有 512 字节大小的栈空间
2.  不能随意调用内核函数, 只能调用预定义好的 bpf helper 函数或者 tail call 其他的 bpf 程序
3.  程序最大长度是有限的
4.  禁止循环

所有这些限制都是为了保证系统的安全性与稳定性, 因为 bpf 程序在内核中执行, 一旦有问题会导致整个系统崩溃。

 

其实最终`events.perf_submit(ctx, &data, sizeof(data))`会被替换成对应的 bpf helper 函数, 这是通过调用 clang 修改编译后程序的 ast 来实现的。

### 分析一下程序:

`b = BPF(text=prog):`  
可以在调用它的时候添加 debug 参数:  
`b = BPF(text=prog,debug=DEBUG_PREPROCESSOR|DEBUG_SOURCE),`  
可以打印出预处理后的程序代码与显示出反编译后的 ebpf 程序, 加上以后再运行 hello_perf_output.py 程序可以看到程序预处理以后添加了一些宏定义, 前面添加上:

```
#if defined(BPF_LICENSE)
#error BPF_LICENSE cannot be specified through cflags
#endif
#if !defined(CONFIG_CC_STACKPROTECTOR)
#if defined(CONFIG_CC_STACKPROTECTOR_AUTO) \
    || defined(CONFIG_CC_STACKPROTECTOR_REGULAR) \
    || defined(CONFIG_CC_STACKPROTECTOR_STRONG)
#define CONFIG_CC_STACKPROTECTOR
#endif
#endif
#define bpf_probe_read_kernel bpf_probe_read
#define bpf_probe_read_kernel_str bpf_probe_read_str
#define bpf_probe_read_user bpf_probe_read
#define bpf_probe_read_user_str bpf_probe_read_str

```

最后添加上:  
`#include <bcc/footer.h>`

 

hello 函数添加了`__attribute__((section(".bpf.fn.hello")))`指定输出 section 为`.bpf.fn.hello`。

 

而`events.perf_submit(ctx, &data, sizeof(data)`) 则修改成了 bpf helper 函数:`bpf_perf_event_output(ctx, bpf_pseudo_fd(1, -1), CUR_CPU_IDENTIFIER, &data, sizeof(data));`  
只不过其中的参数 bpf_pseudo_fd(1, -1) 还是个伪参数, 后面还会做进一步的处理。

 

继续回到`b = BPF(text=prog)`函数, 它所做的事情主要如下:

1.  编译 bpf 程序
2.  加载 bpf 程序
3.  attach bpf 程序

### 1. 编译 bpf 程序:

python 程序中表示 bpf 程序的为 BPF 类, c/c++ 中表示 bfp 程序的类为`ebpf::BPFModule`, 两者的关联依靠于 python 中的 ctypes, 这是类似于 jni 的一种机制 : python 中的 BPF 类加载 libbcc.so.0 共享库并且调用其中的`bpf_module_create_c_from_string()`函数从而创建出 ebpf::BPFModule 对象。

 

ebpf::BPFModule 的构造函数中会初始化 llvm 运行环境, 为编译 ebpf 程序做好准备:

```
  initialize_rw_engine();
  LLVMInitializeBPFTarget();
  LLVMInitializeBPFTargetMC();
  LLVMInitializeBPFTargetInfo();
  LLVMInitializeBPFAsmPrinter();
#if LLVM_MAJOR_VERSION >= 6
  LLVMInitializeBPFAsmParser();
  if (flags & DEBUG_SOURCE)
    LLVMInitializeBPFDisassembler();
#endif
  LLVMLinkInMCJIT(); /* call empty function to force linking of MCJIT */

```

接下来会调用 ebpf::BPFModule 的 load_string() 函数执行编译操作, 它所做的事情可总结如下:

1.  编译 ebpf 程序需要 kheaders 文件, 寻找 kheaders 文件所处路径, 一般 linux 发行版中位于`/lib/modules`目录, 如果配置了`CONFIG_IKHEADERS=m`的内核, kheaders 文件路径为`/proc/kheaders.tar.xz`. 否则就在 / tmp/kheaders-`uname -r` 目录下寻找。
2.  下面开始拼接出 clang 的编译选项, 位于 flags_cstr 变量中, 由于指定了 - emit-llvm 选项, 因此编译出来产物为 llvm IR 文件. 注意一些编译选项使用的文件是虚拟文件:-`include /virtual/include/bcc/bpf_workaround.h -include /virtual/include/bcc/helpers.h`, 编译的虚拟目标是 / virtual/main.c. 编译选项中使用 - I 参数指定了内核的 kheaders 目录。
3.  初始化 clang 编译库, 进行预处理操作, 首先调用`clang::CompilerInvocation的getPreprocessorOpts().addRemappedFile()`函数进行 remapping 操作, 将上面涉及到的虚拟文件映射成真实的文件, 其中 / virtual/main.c 映射为内存中源代码的表示 llvm::MemoryBuffer 对象。
4.  接着进行三轮基于抽象语法树 ast 处理, 分别是:`TracepointFrontendAction,BFrontendAction,EmitLLVMOnlyAction`, 这种处理可以进行源代码的转换, 其中 BFrontendAction 相当复杂, 处理的类型很多, 理解它需要对 clang 相关 api 和 ast 很熟悉才行, 不过可以直接将处理后的源代码打印出来只看对目前程序的处理即可. BFrontendAction 有一步重要的处理:`Catch the map declarations and open the fd's`。
5.  经过上面的处理以后就会生成 LLVM 的 IR 表示 llvm::Module 对象, 接着调用 finalize() 函数, 针对 IR 进行 bpf 程序的编译, 并且执行 PassManager 将涉及到的函数都变成内联函数. 编译出的指令存放在 prog_func_info_ 对象中, 其中 FuncInfo 代表着每个编译出来的函数, 函数执行指令在内存中地址为 FuncInfo 对象的 start_指针。
6.  调用 load_maps 函数通过 bpf 系统调用创建出 bpf maps。

来重点看一下第 6 条的`load_maps`,`hello_perf_output.py`程序有一行代码 : `BPF_PERF_OUTPUT(events);`  
`BPF_PERF_OUTPUT`是一个宏, 定义在文件`src/cc/export/helpers.h`文件中,`BPF_PERF_OUTPUT(events)`其实等价于:

```
struct events_table_t {
  int key;
  u32 leaf;
  /* map.perf_submit(ctx, data, data_size) */
  int (*perf_submit) (void *, void *, u32);
  int (*perf_submit_skb) (void *, u32, void *, u32);
  u32 max_entries;
};
__attribute__((section("maps/perf_output")))
struct events_table_t events = { .max_entries = 0 }

```

`__attribute__((section("maps/perf_output")))`这个编译属性指定的 section 为`maps/perf_output`, 上面的编译过程中 BFrontendAction 扫描到了这个编译属性以后会生成一个生成 map 的 fake id, 并存放在 fake_fd_map_ 成员变量中.  
这个 map 的类型为`BPF_MAP_TYPE_PERF_EVENT_ARRAY`,key 大小对应于`events_table_t`结构的 key 类型 int,value 大小对应于 events_table_t 结构的 leaf 类型 u32,max_entries 大小为 cpu 的个数。

 

当执行到上面第 6 步的 load_maps 函数时, 该函数会遍历`fake_fd_map_`, 调用 bpf 系统调用, 简化版代码可以认为是这样调用的:

```
union bpf_attr attr = {
    .map_type    = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    .key_size    = sizeof(int),
    .value_size  = sizeof(u32),
    .max_entries = get_possible_cpus().size(),
    .map_name = "events",
 
};
int bpf_map_fd = bpf(BPF_MAP_CREATE, &attr, sizeof(attr));

```

总结一下上面的过程:

1.  编译 bpf 程序, 编译产物存放在 BPFModule 类的 prog_func_info_ 成员变量中
2.  因为有了这一行代码: BPF_PERF_OUTPUT(events); 就会调用 bpf 系统调用创建出一个 bpf maps, 类型为 BPF_MAP_TYPE_PERF_EVENT_ARRAY

### 2. 加载 bpf 程序:

这一步就是调用 bpf 系统调用, cmd 为 BPF_PROG_LOAD, 将上面编译的 ebpf 程序加载到内核中。

### 3. attach bpf 程序:

加载完 ebpf 程序以后, 还得将它以 kprobes 形式挂载到系统调用函数 sys_clone, 这种 attach 操作依赖于内核中已经存在机制: perf 或者 ftrace。

 

bcc 会优先使用 perf 的系统调用`perf_event_open`进行 attach, 如果它失败了则使用 ftrace 进行 attach。

 

使用`perf_event_open`进行 attach 的时候是这么调用的:

```
perf_event_attr attr;
attr.sample_period = 1;
attr.wakeup_events = 1;
attr.config |= (0 << 32);
attr.config2 = 0;
attr.size = sizeof(attr);
attr.type = type;   //cat /sys/bus/event_source/devices/kprobe/type
attr.config1 = ptr_to_u64((void *)"sys_clone");
int pfd = perf_event_open(&attr,-1,0,-1,PERF_FLAG_FD_CLOEXEC)
ioctl(pfd, PERF_EVENT_IOC_SET_BPF, progfd);//progfd为bpf(BPF_PROG_LOAD)后的返回值

```

这是第一次遇到 perf_event_open 这个系统调用, 本文实现的 dwarf 的用户态栈回溯正是基于这个系统调用。

 

如果失败了则使用 ftrace 进行 attach:  
将 attach 字符串写入文件`/sys/kernel/debug/tracing/kprobe_events`即可。

#### bpf 程序的执行与数据获取:

`hello_perf_output.py`程序的 hello 函数中的`` `events.perf_submit(ctx, &data, sizeof(data));``  
已经被替换成了如下函数:  
`bpf_perf_event_output(ctx, bpf_pseudo_fd(1, -1), CUR_CPU_IDENTIFIER, &data, sizeof(data));`

 

`bpf_perf_event_output`第二个参数类型为`struct bpf_map *map`, 而上面为`bpf_pseudo_fd(1, -1)`, 可以看到生成的 bpf 程序的字节码:  
`; bpf_perf_event_output(ctx, bpf_pseudo_fd(1, -1), CUR_CPU_IDENTIFIER, &data, sizeof(data)); // Line 35 14: 18 12 00 00 ff ff ff ff 00 00 00 00 00 00 00 00 ld_pseudo r2, 1, 4294967295`  
其实上`BPFModule::load_maps`函数中会对涉及到`bpf_pseudo_fd(1, -1)`形式的指令进行改写, 将这个指令的立即数改写成 events 所对应的 bpf maps 文件描述符, 也就是上面的 bpf_map_fd。

 

为什么`bpf_perf_event_output`第二个参数是`struct bpf_map *map`, 但是调用它的时候传递的却是文件描述符呢, 事实上在加载 bpf 程序的时候内核的 verifier.c 会调用`replace_map_fd_with_map_ptr`将 bpf_map_fd 映射为 bpf_map *。

 

bpf_map 其实算是面向对象中的基类, map 可以有很多种类型, 比如`BPF_MAP_TYPE_PERF_EVENT_ARRAY,BPF_MAP_TYPE_HASH,BPF_MAP_TYPE_PERCPU_ARRAY`, 对于`BPF_MAP_TYPE_PERF_EVENT_ARRAY`类型的 bpf maps 来说, 内核中的实际数据结构为 bpf_array, 它包含着 bpf_map 对象。

 

上面虽然调用了`int bpf_map_fd = bpf(BPF_MAP_CREATE, &attr, sizeof(attr));`创建出了一个`BPF_MAP_TYPE_PERF_EVENT_ARRAY`类型的 bpf maps, 但是这个 maps 并没有关联实际的 perf 存储, 因此还无法往其中写入数据, 需要用户空间调用`perf_event_open()`函数, 将 perf 和 bpf maps 关联才行, 我们在代码中看这是如何做的:  
看一下 python 程序中的这一句:  
`b["events"].open_perf_buffer(print_event)`  
它做的事情如下:

1.  根据 "events" 名称找到 events 所对应的 bpf_map_fd, 创建出 PerfEventArray 对象并且调用它的 open_perf_buffer 函数, print_event 是用于接收数据的回调。
2.  PerfEventArray 对象的 open_perf_buffer 函数会针对每个在线的 cpu 都调用 perf_event_open 系统调用创建针对该 cpu 的 perf_event:
    
    ```
    struct perf_event_attr attr = {};
    attr.config = 10;//PERF_COUNT_SW_BPF_OUTPUT;
    attr.type = PERF_TYPE_SOFTWARE;
    attr.sample_type = PERF_SAMPLE_RAW;
    attr.sample_period = 1;
    attr.wakeup_events = opts->wakeup_events;
    pfd = syscall(__NR_perf_event_open, &attr, pid, cpu, -1, PERF_FLAG_FD_CLOEXEC);
    
    ```
    
    可以看到 perf_event_open 是个相当复杂的系统调用, 有很多的参数配置。
3.  调用 mmap 将 pfd 映射到当前用户空间, 用于和内核通信, 读取内核发送的 perf 记录。
4.  ioctl(pfd, PERF_EVENT_IOC_ENABLE, 0) 启用事件。
5.  回到 table.py 的_open_perf_buffer 函数中调用:`self[self.Key(cpu)] = self.Leaf(fd)` 将所有的 perf_event 与 bpf maps 关联. 这一步的原理其实是调用了`bpf(BPF_MAP_UPDATE_ELEM)`更新 bpf maps 中的键值对, 键为 cpu 的下标, 值为 perf_event_open 系统调用的返回值。

bpf maps 和 perf_event 关联以后, 当 clone 系统调用被调用时, bpf 程序会调用 helper 函数`bpf_perf_event_output`, 往 bpf maps 所关联的 perf_event 中写数据, bcc 会通过读取之前 mmap 的 perf 内存获取这些数据, 调用 python 端的回调函数 print_event 从而打印出 ebpf 程序发送的数据。

 

下面来看一下`perf_event_open`系统调用,`perf_event_open`的配置是相当复杂的, 返回的信息也是五花八门。  
一开始这只是一个性能监测所使用的系统调用, 现代的 cpu 都有被称为`PMU(performance monitoring unit )`的硬件模块, PMU 有几个硬件计数器, 用于给诸如以下的事件计数: cpu 运转了多少个时钟周期, 执行了多少指令以及有多少缓存丢失等等。  
Linux 内核将这些硬件计数器包装成了硬件`perf events`, 同时也有和硬件无关的软件 perf events, 后来软件 perf events 越来越多, 比如和 bpf 相关的软件 perf events 为`PERF_COUNT_SW_BPF_OUTPUT`。

 

perf 事件有两种模式: 计数和采样, 配置了 sample_period 就是采样模式:

```
static inline bool is_sampling_event(struct perf_event *event)
{
    return event->attr.sample_period != 0;
}

```

在采样模式下每达到一定的采样数量会将事件相关信息写入内核和用户空间共享的环形缓冲区中, 而用户空间则通过 mmap 来读取这些数据。  
上面可以看到不仅 attach ebpf 程序的时候可以通过`perf_event_open`(被称为 perf_kprobe PMU),ebpf 程序和用户空间程序进行通信也可以使用它, 而且效率很高, 也是 bcc 推荐的通信方式。

四. trace.py 的栈打印
----------------

trace.py 会自动根据传入的参数帮我们生成一个 bpf 程序:

```
#include #include /* For TASK_COMM_LEN */
 
 
struct probe_do_sys_open_1_data_t
{
 
 
        u32 tgid;
        u32 pid;
        char comm[TASK_COMM_LEN];
        char v0[80];
 
 
       int user_stack_id;
        u32 uid;
};
 
BPF_PERF_OUTPUT(probe_do_sys_open_1_events);
BPF_STACK_TRACE(probe_do_sys_open_1_stacks, 1024);
 
int probe_do_sys_open_1(struct pt_regs *ctx)
{
        u64 __pid_tgid = bpf_get_current_pid_tgid();
        u32 __tgid = __pid_tgid >> 32;
        u32 __pid = __pid_tgid; // implicit cast to u32 for bottom half
        u32 __uid = bpf_get_current_uid_gid();
 
        if (__tgid == 22803) { return 0; }
 
 
        if (__uid != 10121) { return 0; }
 
 
 
 
        if (!(1)) return 0;
 
        struct probe_do_sys_open_1_data_t __data = {0};
 
 
        __data.tgid = __tgid;
        __data.pid = __pid;
        __data.uid = __uid;
        bpf_get_current_comm(&__data.comm, sizeof(__data.comm));
 
        if (PT_REGS_PARM2(ctx) != 0) {
                void *__tmp = (void *)PT_REGS_PARM2(ctx);
                bpf_probe_read_user(&__data.v0, sizeof(__data.v0), __tmp);
        }
 
 
        __data.user_stack_id = probe_do_sys_open_1_stacks.get_stackid(
          ctx, BPF_F_USER_STACK
        );
        probe_do_sys_open_1_events.perf_submit(ctx, &__data, sizeof(__data));
        return 0;
} 
```

经过处理以后

```
__data.user_stack_id = probe_do_sys_open_1_stacks.get_stackid(ctx, BPF_F_USER_STACK);
probe_do_sys_open_1_events.perf_submit(ctx, &__data, sizeof(__data));

```

变为:

```
__data.user_stack_id = bpf_get_stackid(ctx,bpf_pseudo_fd(1, -2),  BPF_F_USER_STACK);
bpf_perf_event_output(ctx, bpf_pseudo_fd(1, -1), CUR_CPU_IDENTIFIER, &__data, sizeof(__data));

```

其中`bpf_perf_event_output`和`hello_perf_output.py`的流程一样就不再赘述了, 这里还多了一个`bpf_get_stackid()`函数, 它也是一个 bpf helper 函数。

 

来看一下上面的程序的这一句:  
`BPF_STACK_TRACE(probe_do_sys_open_1_stacks, 1024);`  
`BPF_STACK_TRACE`这个宏同样的定义`在src/cc/export/helpers.h`文件中, 它被扩展为:

```
struct probe_do_sys_open_1_stacks_table_t {
  int key;
  struct bpf_stacktrace leaf;
  struct bpf_stacktrace * (*lookup) (int *);
  struct bpf_stacktrace * (*lookup_or_init) (int *, struct bpf_stacktrace *);
  struct bpf_stacktrace * (*lookup_or_try_init) (int *, struct bpf_stacktrace *);
  int (*update) (int *, struct bpf_stacktrace *);
  int (*insert) (int *, struct bpf_stacktrace *);
  int (*delete) (int *);
  void (*call) (void *, int index);
  void (*increment) (int, ...);
  void (*atomic_increment) (_key_inttype, ...);
  int (*get_stackid) (void *, u64);
  void * (*sk_storage_get) (void *, void *, int);
  int (*sk_storage_delete) (void *);
  void * (*inode_storage_get) (void *, void *, int);
  int (*inode_storage_delete) (void *);
  void * (*task_storage_get) (void *, void *, int);
  int (*task_storage_delete) (void *);
  u32 max_entries;
  int flags;
};
__attribute__((section("maps/stacktrace")))
struct probe_do_sys_open_1_stacks_table_t probe_do_sys_open_1_stacks = { .flags = (0), .max_entries = (roundup_pow_of_two(1024)) };
 
struct ____btf_map_probe_do_sys_open_1_stacks {           
    int key;               
    struct bpf_stacktrace value;               
};                       
struct ____btf_map_probe_do_sys_open_1_stacks   
__attribute__ ((section(".maps.probe_do_sys_open_1_stacks"), used))   
____btf_map_probe_do_sys_open_1_stacks = { }

```

当 BFrontendAction 在处理 ast 的时候扫描到编译属性`__attribute__((section("maps/stacktrace")))`时, 就会创建出类型为`BPF_MAP_TYPE_STACK_TRACE`的 bpf maps, 这个 maps 的 key 大小为`probe_do_sys_open_1_stacks_table_t`结构的`key`大小`int`,`value`大小为`probe_do_sys_open_1_stacks_table_t`结构的`leaf`大小`struct bpf_stacktrace`。

 

struct bpf_stacktrace 结构定义为:

```
struct bpf_stacktrace {
  u64 ip[BPF_MAX_STACK_DEPTH];
};

```

可以看到它其实就是个 u64 数组, 事实上每一个 u64 都对应着一个调用栈的 pc 指针, bcc 就是靠这个数据来进行栈回溯的。

 

当 ebpf 程序执行的时候, 它会通过`bpf_perf_event_output()`往用户空间输出`bpf_get_stackid()`的返回值`long stack_id`, 用户空间程序拿到这个`stack_id`以后把它作为`bpf maps`中的 key, 调用`bpf(BPF_MAP_LOOKUP_ELEM)`得到类型为`struct bpf_stacktrace`结构的调用栈数据, 然后 bcc 就可以按照自己的方式进行符号化了。具体的代码在 table.py 的 class StackTrace 类中。  
打印调用栈的逻辑为:

```
for addr in stack:
                stackstr += '        '
                if Probe.print_address:
                    stackstr += ("%16x " % addr)
                symstr = bpf.sym(addr, tgid, show_module=True, show_offset=True)
                stackstr += ('%s\n' % (symstr.decode('utf-8')))

```

其中 addr 就是通过 stack_id 调用`bpf(BPF_MAP_LOOKUP_ELEM)`得到的地址列表。

 

`bpf_get_stackid()`是怎么实现的? 直接给出结论: 通过基于 fp 的栈回溯来实现的, 感兴趣的话可以查看一下内核的`perf_callchain_user`函数, 这里就不细述了。

 

到这里, 上面使用 trace.py 打印出程序的堆栈信息缺失的原因就揭晓了, 基于 fp 的栈回溯虽然比较快, 但是编译时如果指定了`"-fomit-frame-pointer"`就不会生成 fp 指针, 堆栈信息就会缺失. 如果用 bcc 分析某些大厂 app, 很多调用除了第一行以外其他全是 unknown。

 

而对于`func_a`调用的堆栈信息符号化缺失是因为 bcc 它的符号化是读取`/proc/pid/maps`并解析其中的 so 的 elf 符号表来实现的, 由于 zipalign 的原因：[https://developer.android.com/studio/command-line/zipalign?hl=zh-cn](https://developer.android.com/studio/command-line/zipalign?hl=zh-cn)  
如果 apk 中所有未归档文件相对于文件开头是对齐的, 那么 so 文件就不会被解压至 / data/app 目录下, 而是直接在 apk 文件中进行 mmap。  
bcc 对 apk 的支持不好, 所以无法正确处理这种情况. 而且 bcc 读取`/proc/pid/maps`只读取一次, 除非整个可执行文件被替换不然 bcc 不会重新读取`/proc/pid/maps`, 这也就导致了无法更新到后加载的 so 的情况。

五. 问题的解决
--------

在 bcc 的 issue 列表中其实已经有人提了这样的问题:  
[https://github.com/iovisor/bcc/issues/3515](https://github.com/iovisor/bcc/issues/3515)

 

上面提到了一个工具`simpleperf`  
trace.py 无法打印的, simpleperf 却可以, 这是因为 simpleperf 是基于 dwarf 的. eh_frame 节进行栈回溯的, 就算 so 编译的时候指定了`"-fomit-frame-pointer"`也能正确打印出堆栈信息. 那么如果将 simpleperf 的栈回溯机制和 bcc 结合不就可以解决问题了吗?

 

先来看一下`simpleperf`的使用:  
simpleperf 是 android 提供的性能分析工具, 它也是通过调用`perf_event_open`来实现的, 假设我们想追踪 4665 这个进程的 sys_enter_clone 系统调用. 我们在手机上这么执行持续 10 秒采样:

```
$ simpleperf record -p 4665 --call-graph dwarf -e syscalls:sys_enter_clone --duration 10  -o /data/local/tmp/perf.data

```

然后将`/data/local/tmp/perf.data`拷贝至`aosp/system/extras/simpleperf/scripts`目录下, 执行`report_sample.py`即可看到期间的所有`sys_enter_clone`调用以及堆栈。

 

虽然`simpleperf`有时用来做逆向分析的备用工具来使用也凑合, 但是它最大的缺陷是无法打印出系统调用的参数, 更无法采用一些 hack 的技巧来修改系统调用参数, 而 ebpf 是可以直接执行定制化代码的, 因此如果 bcc 可以结合 simpleperf 的栈回溯机制将是完美的组合。

 

从 bcc 的官方开发人员的回复来看, 短时间内应该不会有基于 dwarf 的官方实现. 而且安卓运行环境比较特殊, 感觉 bcc 更多的是面向 linux pc 领域, 只能自己来动手了。

 

首先我们需要理解一下`simpleperf`的原理, 对于上面的命令来说,`simpleperf`其实是构建了以下的 perf_event_attr:

```
struct perf_event_attr attr;
attr.size = sizeof(perf_event_attr);
attr.type = PERF_TYPE_TRACEPOINT;
attr.config = id; //cat /sys/kernel/debug/tracing/events/syscalls/sys_enter_openat/id
attr.disabled = 0;
attr.read_format = PERF_FORMAT_TOTAL_TIME_ENABLED | PERF_FORMAT_TOTAL_TIME_RUNNING | PERF_FORMAT_ID;
attr.sample_type |= PERF_SAMPLE_IP | PERF_SAMPLE_TID | PERF_SAMPLE_TIME | PERF_SAMPLE_PERIOD | PERF_SAMPLE_CPU | PERF_SAMPLE_ID | PERF_SAMPLE_RAW;
attr.sample_type |= PERF_SAMPLE_CALLCHAIN | PERF_SAMPLE_REGS_USER | PERF_SAMPLE_STACK_USER;
attr.sample_type &= ~PERF_SAMPLE_BRANCH_STACK;     
attr.exclude_callchain_user = 1;
attr.sample_regs_user = ((1ULL << PERF_REG_ARM64_MAX) - 1);
attr.sample_stack_user = 65528;
attr.inherit = 0;
attr.sample_id_all = 1;     //SampleIdAll                           
attr.freq = 0;
attr.sample_period = DEFAULT_SAMPLE_PERIOD_FOR_TRACEPOINT_EVENT; //1
attr.mmap = 1;
attr.comm = 1;

```

可以看到 attr.sample_type 指定的类型非常多, 其中最重要的类型为`PERF_SAMPLE_REGS_USER | PERF_SAMPLE_STACK_USER`

 

看一下 perf 对 dwarf 栈回溯支持的提交文件:  
[https://lwn.net/Articles/507753/](https://lwn.net/Articles/507753/)  
[[https://lore.kernel.org/all/1343391834-10851-7-git-send-email-jolsa@redhat.com/](https://lore.kernel.org/all/1343391834-10851-7-git-send-email-jolsa@redhat.com/)]

1.  `PERF_SAMPLE_REGS_USER`用于指示内核将用户空间发生事件时 (这里为系统调用) 的寄存器信息以`PERF_RECORD_SAMPLE`记录类型传送给用户空间。
    
2.  `PERF_SAMPLE_STACK_USER`用于指示内核将用户空间发生事件时 (这里为系统调用) 的栈空间片段数据以`PERF_RECORD_SAMPLE`记录类型传送给用户空间。
    

`simpleperf`有了这两个信息以后再加上一个目标进程的 maps 信息即可指示`libunwindstack`库进行 dwarf 的栈回溯:

```
UnwindMaps& cached_map = cached_maps_[thread.pid];
cached_map.UpdateMaps(*thread.maps);
std::shared_ptr stack_memory(
    new unwindstack::MemoryOfflineBuffer(reinterpret_cast(stack),
                                         stack_addr, stack_addr + stack_size));
std::unique_ptr unwind_regs(GetBacktraceRegs(regs));
if (!unwind_regs) {
  return false;
}
unwindstack::Unwinder unwinder(MAX_UNWINDING_FRAMES, &cached_map, unwind_regs.get(),
                               stack_memory);
unwinder.SetResolveNames(false);
unwinder.Unwind(); 
```

libunwindstack 库是谷歌自己开发的栈回溯库, 它提供了强大的功能:  
支持 ARM32，ARM64，X86，X86_64，MIPS，MIPS_64, 可以关联 ART 虚拟机 OAT，Jit 调试产生的 native 栈等。

 

总结起来就是`simpleperf`拿到目标进程的寄存器信息, 栈空间片段数据以及 maps 信息即可调用`libunwindstack`库进行 dwarf 的栈回溯。

 

`libunwindstack`库基于 gtest 写了一些测试用例位于`system/core/libunwindstack/tests`目录下, 阅读这些测试用例可以很好的理解 libunwindstack 库的 api, 可以执行如下命令来运行测试用例:

```
$ atest -bt libunwindstack_test

```

既然知道`simpleperf的`原理, 那么如果 bcc 程序也可以得到寄存器信息和栈空间片段数据, 不就一样可以调用`libunwindstack`库进行 dwarf 的栈回溯了吗?  
事实上还需要解决以下问题:

1.  如何和现有的 bcc 框架集成
2.  有没有现有的 bpf helper 函数可以实现这一目标
3.  bcc 运行环境是 debian, 如何访问 android 环境下的 libunwindstack 库

bcc 的开发者在 issue 列表中提议也许可以通过`bpf_probe_read_user`调用获取用户栈数据, 但是由于 ebpf 的执行栈只有 512 字节, 获取了用户栈数据需要保存在栈上的变量中, 而很多程序的调用栈远远不止 512 字节, 因此我认为这种方式不太好。

 

**我这里的解决方案为:**  
通过观察`bpf_perf_event_output`函数的实现可以发现, 在它调用的`perf_output_sample`函数中只要 sample_type 的`PERF_SAMPLE_REGS_USER`位被设置, 就会发送寄存器数据, 只要 sample_type 的`PERF_SAMPLE_STACK_USER`位被设置, 就会发送用户空间栈数据。  
因此只需在 bcc 程序中调用`perf_event_open`的时候设置相应的标记即可。不过由于 bcc 程序没有针对这两种类型数据做处理, 因此需要修改读取函数的逻辑。  
得到相应的数据以后, 可以在 android 端启动一个进程专门获取 bcc 端发送的数据, 这个进程然后调用 libunwindstack 库进行栈回溯, 将结果再回传给 bcc, 两者通信可以采用 unix domain socket。

 

为了控制何时打印堆栈，何时不打印堆栈，如果 perf 变量以`_with_stack`结尾则打印出堆栈信息：  
`BPF_PERF_OUTPUT(events_with_stack);`  
在 libbpf.c 文件的`bpf_open_perf_buffer_opts`函数中判断如果需要打印堆栈，则修改 perf 调用参数:

```
if (opts->unwind_call_stack == 1) {
    attr.sample_type |= PERF_SAMPLE_STACK_USER | PERF_SAMPLE_REGS_USER;
    attr.sample_stack_user = 16384;  // MAX=65528
    attr.sample_regs_user = ((1ULL << 33) - 1);
    attr.size = sizeof(struct perf_event_attr);
    reader->is_unwind_call_stack = true;
  }

```

接着在 perf_reader.c 的`parse_sw`函数中解析获取到的寄存器信息与栈数据:

```
if (reader->is_unwind_call_stack) {
   int pid = *(int *)raw->data;
   int write_size = ((uint8_t *)data + size) - ptr;
   print_frame_info(pid, ptr, write_size);
 }

```

perf 的 PERF_RECORD_SAMPLE 类型的记录是可变结构，对于这里来说它的结构为:

```
struct {
    struct perf_event_header header;
    u32    size;        /* if PERF_SAMPLE_RAW */
    char   data[size];  /* if PERF_SAMPLE_RAW */
    u64    abi;         /* if PERF_SAMPLE_REGS_USER */
    u64    regs[weight(mask)];
                        /* if PERF_SAMPLE_REGS_USER */
    u64    size;        /* if PERF_SAMPLE_STACK_USER */
    char   data[size];  /* if PERF_SAMPLE_STACK_USER */
    u64    dyn_size;    /* if PERF_SAMPLE_STACK_USER &&
                           size != 0 */
};

```

本来是想在记录中请求 PERF_SAMPLE_TID 信息用于获取目标进程的 / proc/pid/maps 文件，但得到的数据总是不正确的，看了一个内核代码，应该和`perf_prepare_sample`函数并没有处理 PERF_SAMPLE_TID 相关的逻辑有关，不过从 ebpf 程序中可以直接得到 pid,pid 的值为:  
`bpf_get_current_pid_tgid() >> 32;`

 

获取到的信息全部通过 unix domaian socket 发送给远端 android 端守护进程，由守护进程解析出信息:

```
static void print_frame_info(int pid, uint8_t *ptr, int write_size) {
  int fd = socket(PF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0);
  if (fd == -1) {
    fprintf(stderr, "cannot socket()!\n");
  }
  const char *socket_path = "/dev/socket/uw";
  struct sockaddr_un addr = {.sun_family = AF_UNIX};
  strncpy(addr.sun_path, socket_path, sizeof(addr.sun_path));
 
  int ret = connect(fd, (struct sockaddr *)(&addr), sizeof(addr));
  if (ret != 0) {
    fprintf(stderr, "connect() to %s failed: %s\n", socket_path,
            strerror(errno));
  } else {
    WriteFully(fd, &pid, 4);
    write_size += 4;
    WriteFully(fd, &write_size, 4);
 
    if (!WriteFully(fd, ptr, write_size)) {
      // fprintf(stderr, "All data were send.\n");
      fprintf(stderr, "prepare to write %d to socket\n", write_size);
      close(fd);
      return;
    }
    int frameinfo_size;
    if (ReadFully(fd, &frameinfo_size, 4)) {
      unsigned char frame_info[frameinfo_size + 1];
      if (ReadFully(fd, frame_info, frameinfo_size)) {
        frame_info[frameinfo_size] = '\0';
        printf("===================================>Frame:\n");
        printf("%s\n", frame_info);
      } else {
        fprintf(stderr, "Read frame_info from socket error.\n");
      }
    } else {
      fprintf(stderr, "Read frameinfo_size from socket error. \n");
    }
 
    close(fd);
  }
}

```

Android 端守护进程获取数据以后组装起来调用`libunwindstack`打印:

```
bool OfflineUnwinder::UnwindCallChain(DataBuff &buff, std::string &frame_info) {
    RegSet regs(buff.mRegs.abi, buff.mRegs.reg_mask, buff.mRegs.regs);
    uint64_t sp_reg_value;
    if (!regs.GetSpRegValue(&sp_reg_value)) {
        std::cerr << "can't get sp reg value" << std::endl;
        return false;
    } else {
#ifdef DEBUG
        std::cout << "sp_reg_value: 0x" << std::hex << sp_reg_value << std::endl;
#endif
    }
    uint64_t stack_addr = sp_reg_value;
    const char *stack = buff.mStack.data;
    size_t stack_size = buff.mStack.dyn_size;
 
    std::shared_ptr stack_memory(
            new unwindstack::MemoryOfflineBuffer(reinterpret_cast(stack),
                                                 stack_addr, stack_addr + stack_size));
    std::unique_ptr unwind_regs(GetBacktraceRegs(regs));
    if (!unwind_regs) {
        return false;
    }
    std::unique_ptr maps;
    std::string data;
    std::string proc_map_file = "/proc/" + std::to_string(buff.mTargetPid) + "/maps";
    android::base::ReadFileToString(proc_map_file, &data);
    maps.reset(new unwindstack::BufferMaps(data.c_str()));
    maps->Parse();
    unwindstack::Unwinder unwinder(512, maps.get(), unwind_regs.get(), stack_memory);
    // unwinder.SetResolveNames(false);
    unwinder.Unwind();
    frame_info = DumpFrames(unwinder);
    return true;
} 
```

然后 android 端守护进程会将得到的打印栈信息通过 socket 传送给 bcc。

 

程序好了，试一下效果吧，先看看测试程序 (已经指定了`-fomit-frame-pointer`) 的堆栈打印:  
![](https://bbs.pediy.com/upload/attach/202209/607812_3KQBGC9NSVSP826.png)  
可以看到，非常的完美。

 

再来看一下修改 bcc 的`tcpconnect.py`对某大厂 app 分析的结果:  
将`tcpconnect.py`中的`ipv4_events`修改为  
`ipv4_events_with_stack`,`ipv6_events`修改为`ipv6_events_with_stack`表示需要打印堆栈。  
然后 app 看看效果:  
`python3 tcpconnect.py -u 10112`

 

可以看到打印出的堆栈信息:  
![](https://bbs.pediy.com/upload/attach/202209/607812_ZKRAH2Q9P82SNJ9.png)

 

不仅能显示出堆栈偏移量，而且能打印出连接的 ip 地址和端口号:-)

 

那么可以将 so 中的. eh_frame 节移除掉吗，这个节会在运行时加载，c++ 语言中的异常是靠它来实现的，如果移除掉，涉及到 C++ 异常代码的时候程序直接就崩溃了，而且 app 的 crash 收集也靠它，所以基于. eh_frame 节的栈回溯目前来是个比较可靠的方案。

[[2022 夏季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

最后于 10 小时前 被飞翔的猫咪编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#源码框架](forum-161-1-127.htm)