> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283413.htm)

> [翻译] 使用 eBPF 编写系统调用跟踪器

_原文链接：[https://sh4dy.com/2024/08/03/beetracer/](https://sh4dy.com/2024/08/03/beetracer/)_

**sh4dy**

2024-08-03

[eBPF](https://sh4dy.com/tags/eBPF/), [linux](https://sh4dy.com/tags/linux/), [low-level](https://sh4dy.com/tags/low-level/)

**Pre-Requisites**

前置知识
----

System calls, eBPF, C, basics of low-level programming.  
系统调用，eBPF，C 语言，底层编程基础。

**Introduction**

引言
--

eBPF (Extended Berkeley Packet Filter) is a technology that allows users to run custom programs within the kernel. BPF / or cBPF (classic BPF), the predecessor of eBPF provided a simple and efficient way to filter packets based on predefined rules. eBPF programs offer enhanced safety, portability, and maintainability as compared to kernel modules. There are several high-level methods available for working with eBPF programs, such as [Cilium’s go library](https://github.com/cilium/ebpf), [bpftrace](https://github.com/bpftrace/bpftrace), [libbpf](https://github.com/libbpf/libbpf), etc.  
eBPF（扩展伯克利包过滤器）是一项允许用户在内核中运行自定义程序的技术。BPF 或 cBPF（经典 BPF），作为 eBPF 的前身，提供了一种基于预定义规则过滤数据包的简单高效方式。相较于内核模块，eBPF 程序在安全性、可移植性和可维护性方面均有显著提升。针对 eBPF 程序的操作，有多种高级方法可供选择，如 [Cilium 的 Go 库](https://github.com/cilium/ebpf)、[bpftrace](https://github.com/bpftrace/bpftrace)、[libbpf](https://github.com/libbpf/libbpf) 等。

*   `Note`: This post requires the reader to have a basic understanding of `eBPF`. If you’re not familiar with it, [this post](https://ebpf.io/what-is-ebpf/) by `ebpf.io` is a great read.  
    `注意`：本文要求读者具备`eBPF`的基础知识。如果您对此不熟悉，[这篇文章](https://ebpf.io/what-is-ebpf/)由`ebpf.io`撰写，值得一读。

**Objectives**

目标
--

You must already be familiar with the famous tool `strace`. We’ll be developing something similar to that using eBPF. For example,  
您一定已经熟悉著名的工具`strace`，我们将使用 eBPF 开发一个类似的功能。例如，

```
./beetrace /bin/ls
```

**Concepts**

概念
--

Before we start writing our tool, we need to familiarize ourselves with some key concepts.  
在我们开始编写工具之前，我们需要熟悉一些关键概念。

1.`Tracepoints`:

They are instrumentation points placed in various parts of the Linux kernel code. They provide a way to hook into specific events or code paths within the kernel without modifying the kernel source code. The events available of tracing can be found at `/sys/kernel/debug/tracing/events`.  
`跟踪点`：它们是放置在 Linux 内核代码各处的检测点。它们提供了一种在不修改内核源代码的情况下，挂钩到内核中特定事件或代码路径的方法。可用于跟踪的事件可以在 `/sys/kernel/debug/tracing/events` 找到。

2.The `SEC` macro:

It creates a new section with the name as the name of the tracepoint within the target ELF. For example, `SEC(tracepoint/raw_syscalls/sys_enter)` creates a new section with this name. The sections can be viewed using readelf.  
`SEC` 宏：它创建一个新节，节名即为目标 ELF 中跟踪点的名称。例如， `SEC(tracepoint/raw_syscalls/sys_enter)` 创建一个具有此名称的新节。可以使用 readelf 查看这些节。

```
readelf -s --wide somefile.o
```

3.`Maps`:

They are shared data structures that can be accessed from both eBPF programs and applications running in the userspace.  
`映射`：它们是共享的数据结构，可以从 eBPF 程序和运行在用户空间的应用程序中访问。

**Writing the eBPF programs**

编写 eBPF 程序
----------

We won’t be writing a comprehensive tool for tracing all the system calls due to the vast number of system calls present in the Linux kernel. Instead, we’ll focus on tracing a few common system calls. To achieve this, we’ll write two types of programs: eBPF programs and a loader (which loads the BPF objects into the kernel and attaches them).  
由于 Linux 内核中系统调用数量庞大，我们不会编写一个全面追踪所有系统调用的工具。相反，我们将专注于追踪一些常见的系统调用。为此，我们将编写两种类型的程序：eBPF 程序和一个加载器（负责将 BPF 对象加载到内核并附加它们）。

Let’s start by creating a few data structures to set things up.  
让我们从创建几个数据结构开始，为后续工作做好准备。

```
// controller.h
 
// SYS_ENTER : for retrieving system call arguments
// SYS_EXIT : for retrieving the return values of syscalls
 
typedef enum
{
    SYS_ENTER,
    SYS_EXIT
} event_mode;
 
struct inner_syscall_info
{
    union
    {
        struct
        {
            // For SYS_ENTER mode
            char name[32];
            int num_args;
            long syscall_nr;
            void *args[MAX_ARGS];
        };
        long retval; // For SYS_EXIT mode
    };
    event_mode mode;
};
 
struct default_syscall_info{
    char name[32];
    int num_args;
};
 
// Array for storing the name and argument count of system calls
const struct default_syscall_info syscalls[MAX_SYSCALL_NR] = {
    [SYS_fork] = {"fork", 0},
    [SYS_alarm] = {"alarm", 1},
    [SYS_brk] = {"brk", 1},
    [SYS_close] = {"close", 1},
    [SYS_exit] = {"exit", 1},
    [SYS_exit_group] = {"exit_group", 1},
    [SYS_set_tid_address] = {"set_tid_address", 1},
    [SYS_set_robust_list] = {"set_robust_list", 1},
    [SYS_access] = {"access", 2},
    [SYS_arch_prctl] = {"arch_prctl", 2},
    [SYS_kill] = {"kill", 2},
    [SYS_listen] = {"listen", 2},
    [SYS_munmap] = {"sys_munmap", 2},
    [SYS_open] = {"open", 2},
    [SYS_stat] = {"stat", 2},
    [SYS_fstat] = {"fstat", 2},
    [SYS_lstat] = {"lstat", 2},
    [SYS_accept] = {"accept", 3},
    [SYS_connect] = {"connect", 3},
    [SYS_execve] = {"execve", 3},
    [SYS_ioctl] = {"ioctl", 3},
    [SYS_getrandom] = {"getrandom", 3},
    [SYS_lseek] = {"lseek", 3},
    [SYS_poll] = {"poll", 3},
    [SYS_read] = {"read", 3},
    [SYS_write] = {"write", 3},
    [SYS_mprotect] = {"mprotect", 3},
    [SYS_openat] = {"openat", 3},
    [SYS_socket] = {"socket", 3},
    [SYS_newfstatat] = {"newfstatat", 4},
    [SYS_pread64] = {"pread64", 4},
    [SYS_prlimit64] = {"prlimit64", 4},
    [SYS_rseq] = {"rseq", 4},
    [SYS_sendfile] = {"sendfile", 4},
    [SYS_socketpair] = {"socketpair", 4},
    [SYS_mmap] = {"mmap", 6},
    [SYS_recvfrom] = {"recvfrom", 6},
    [SYS_sendto] = {"sendto", 6},
};
```

The loader will read the path of the ELF file to be traced, which will be provided by the user as a command line argument. Then, the loader will spawn a child process and use `execve` to run the program specified in the command line argument.  
加载器将读取要跟踪的 ELF 文件的路径，该路径将由用户作为命令行参数提供。然后，加载器将生成一个子进程，并使用`execve`来运行命令行参数中指定的程序。

The parent process will handle all the necessary setup for loading and attaching the eBPF programs. It also performs the crucial task of sending the child process’s ID to the eBPF program via the BPF hashmap.  
父进程将处理加载和附加 eBPF 程序所需的所有必要设置。它还执行关键任务，即通过 BPF 哈希映射将子进程的 ID 发送给 eBPF 程序。

```
// loader.c
 
int main(int argc, char **argv)
{
  if (argc < 2)
  {
    fatal_error("Usage: ./beetrace ");
  }
 
  const char *file_path = argv[1];
 
  pid_t pid = fork();
  if (pid == 0)
  {
    // Child process
    int fd = open("/dev/null", O_WRONLY);
    if(fd==-1){
        // error
    }
    dup2(fd, 1); // disable stdout for the child process
    sleep(2); // wait for the parent process to do the required setup for tracing
    execve(file_path, NULL, NULL);
  }
  else{
    // Parent process
  }
} 
```

To trace system calls, we need to write eBPF programs that are triggered by the `tracepoint/raw_syscalls/sys_enter` and `tracepoint/raw_syscalls/sys_exit` tracepoints. These tracepoints provide access to the system call number and arguments. For a given system call, the `tracepoint/raw_syscalls/sys_enter` tracepoint is always triggered before the `tracepoint/raw_syscalls/sys_exit` tracepoint. We can use the former to retrieve the system call arguments and the latter to obtain the return value. Additionally, we will use eBPF maps to share information between the user-space program and our eBPF programs. Specifically, we will use two types of eBPF maps: hashmaps and ring buffers.  
要追踪系统调用，我们需要编写由 `tracepoint/raw_syscalls/sys_enter` 和 `tracepoint/raw_syscalls/sys_exit` 跟踪点触发的 eBPF 程序。这些跟踪点提供了访问系统调用号和参数的途径。对于特定的系统调用， `tracepoint/raw_syscalls/sys_enter` 跟踪点总是在 `tracepoint/raw_syscalls/sys_exit` 跟踪点之前触发。我们可以利用前者获取系统调用参数，后者则用于获取返回值。此外，我们将使用 eBPF 映射在用户空间程序与 eBPF 程序之间共享信息。具体来说，我们将使用两种类型的 eBPF 映射：哈希表和环形缓冲区。

```
// controller.c
 
// Hashmap
struct
{
  __uint(type, BPF_MAP_TYPE_HASH);
  __uint(key_size, 10);
  __uint(value_size, 4);
  __uint(max_entries, 256 * 1024);
} pid_hashmap SEC(".maps");
 
// Ring buffer
struct
{
  __uint(type, BPF_MAP_TYPE_RINGBUF);
  __uint(max_entries, 256 * 1024);
} syscall_info_buffer SEC(".maps");
```

Having defined the maps, we’re ready to write the programs. Let’s start by writing the program for the tracepoint `tracepoint/raw_syscalls/sys_enter`.  
定义了映射后，我们准备编写程序。首先从编写跟踪点 `tracepoint/raw_syscalls/sys_enter` 的程序开始。

```
// loader.c
 
SEC("tracepoint/raw_syscalls/sys_enter")
int detect_syscall_enter(struct trace_event_raw_sys_enter *ctx)
{
  // Retrieve the system call number
  long syscall_nr = ctx->id;
  const char *key = "child_pid";
  int target_pid;
 
  // Reading the process id of the child process in userland
  void *value = bpf_map_lookup_elem(&pid_hashmap, key);
  void *args[MAX_ARGS];
 
  if (value)
  {
    target_pid = *(int *)value;
 
    // PID of the process that executed the current system call
    pid_t pid = bpf_get_current_pid_tgid() & 0xffffffff;
    if (pid == target_pid && syscall_nr >= 0 && syscall_nr < MAX_SYSCALL_NR)
    {
 
      int idx = syscall_nr;
      // Reserve space in the ring buffer
      struct inner_syscall_info *info = bpf_ringbuf_reserve(&syscall_info_buffer, sizeof(struct inner_syscall_info), 0);
      if (!info)
      {
        bpf_printk("bpf_ringbuf_reserve failed");
        return 1;
      }
 
      // Copy the syscall name into info->name
      bpf_probe_read_kernel_str(info->name, sizeof(syscalls[syscall_nr].name), syscalls[syscall_nr].name);
      for (int i = 0; i < MAX_ARGS; i++)
      {
        info->args[i] = (void *)BPF_CORE_READ(ctx, args[i]);
      }
      info->num_args = syscalls[syscall_nr].num_args;
      info->syscall_nr = syscall_nr;
      info->mode = SYS_ENTER;
      // Insert into ring buffer
      bpf_ringbuf_submit(info, 0);
    }
  }
  return 0;
}
```

Similarly, we can write the program for reading the return value and sending it to userland.  
同样地，我们可以编写程序来读取返回值并将其发送给用户空间。

```
// controller.c
 
SEC("tracepoint/raw_syscalls/sys_exit")
int detect_syscall_exit(struct trace_event_raw_sys_exit *ctx)
{
  const char *key = "child_pid";
  void *value = bpf_map_lookup_elem(&pid_hashmap, key);
  pid_t pid, target_pid;
 
  if (value)
  {
    pid = bpf_get_current_pid_tgid() & 0xffffffff;
    target_pid = *(pid_t *)value;
    if (pid == target_pid)
    {
      struct inner_syscall_info *info = bpf_ringbuf_reserve(&syscall_info_buffer, sizeof(struct inner_syscall_info), 0);
      if (!info)
      {
        bpf_printk("bpf_ringbuf_reserve failed");
        return 1;
      }
      info->mode = SYS_EXIT;
      info->retval = ctx->ret;
      bpf_ringbuf_submit(info, 0);
    }
  }
  return 0;
}
```

Let’s now finalize the functionality for the parent process in the loader program. Before doing that, we need to understand how some key functions work.  
现在让我们最终确定加载程序中父进程的功能。在此之前，我们需要了解一些关键函数的工作原理。

1.`bpf_object__open`:

Creates a bpf_object by opening the BPF ELF object file pointed to by the passed path and loading it into memory.  
通过打开由传递路径指向的 BPF ELF 对象文件并将其加载到内存中，创建一个 bpf_object。

```
LIBBPF_API struct bpf_object *bpf_object__open(const char *path);
```

2.`bpf_object__load`:

Loads BPF object into kernel.  
将 BPF 对象加载到内核中。

```
LIBBPF_API int bpf_object__load(struct bpf_object *obj);
```

3.`bpf_object__find_program_by_name`:

Returns a pointer to a valid BPF program.  
返回指向有效 BPF 程序的指针。

```
LIBBPF_API struct bpf_program *bpf_object__find_program_by_name(const struct bpf_object *obj,const char *name);
```

4.`bpf_program__attach`:

Function for attaching a BPF program based on auto-detection of program type, attach type, and extra paremeters, where applicable.  
根据程序类型、附加类型及适用情况下的额外参数自动检测，用于附加 BPF 程序的函数。

```
LIBBPF_API struct bpf_link *bpf_program__attach(const struct bpf_program *prog);
```

5.`bpf_map__update_elem`:

Allows to insert or update value in BPF map that corresponds to provided key.  
允许在 BPF 映射中插入或更新与所提供键对应的值。

```
LIBBPF_API int bpf_map__update_elem(const struct bpf_map *map,const void *key, size_t key_sz, const void *value, size_t value_sz, __u64 flags);
```

6.`bpf_object__find_map_fd_by_name`:

Given a BPF map name, it returns a file descriptor to it.  
给定一个 BPF 映射名称，它返回其文件描述符。

```
LIBBPF_API int bpf_object__find_map_fd_by_name(const struct bpf_object *obj, const char *name);
```

7.`ring_buffer__new`:

Returns a pointer to the ring buffer.  
返回指向环形缓冲区的指针。

```
LIBBPF_API struct ring_buffer *ring_buffer__new(int map_fd, ring_buffer_sample_fn sample_cb, void *ctx, const struct ring_buffer_opts *opts);
```

The second argument must be a function which can be used for handling the data received from the ring buffer.  
第二个参数必须是一个函数，用于处理从环形缓冲区接收到的数据。

```
bool initialized = false;
 
static int syscall_logger(void *ctx, void *data, size_t len)
{
  struct inner_syscall_info *info = (struct inner_syscall_info *)data;
  if (!info)
  {
    return -1;
  }
 
  if (info->mode == SYS_ENTER)
  {
    initialized = true;
    printf("%s(", info->name);
    for (int i = 0; i < info->num_args; i++)
    {
      printf("%p,", info->args[i]);
    }
    printf("\b) = ");
  }
  else if (info->mode == SYS_EXIT)
  {
    if (initialized)
    {
      printf("0x%lx\n", info->retval);
    }
  }
  return 0;
}
```

It prints the name and arguments of the system calls.  
它打印系统调用的名称和参数。

8.`ring_buffer__consume`:

It processes the available events in the ring buffer.  
它处理环形缓冲区中的可用事件。

```
LIBBPF_API int ring_buffer__consume(struct ring_buffer *rb);
```

We now have everything needed to write the loader.  
我们现在拥有编写加载器所需的一切。

```
// loader.c
#include #include "controller.h"
#include #include #include #include #include void fatal_error(const char *message)
{
  puts(message);
  exit(1);
}
 
bool initialized = false;
 
static int syscall_logger(void *ctx, void *data, size_t len)
{
  struct inner_syscall_info *info = (struct inner_syscall_info *)data;
  if (!info)
  {
    return -1;
  }
 
  if (info->mode == SYS_ENTER)
  {
    initialized = true;
    printf("%s(", info->name);
    for (int i = 0; i < info->num_args; i++)
    {
      printf("%p,", info->args[i]);
    }
    printf("\b) = ");
  }
  else if (info->mode == SYS_EXIT)
  {
    if (initialized)
    {
      printf("0x%lx\n", info->retval);
    }
  }
  return 0;
}
 
int main(int argc, char **argv)
{
  int status;
  struct bpf_object *obj;
  struct bpf_program *enter_prog, *exit_prog;
  struct bpf_map *syscall_map;
  const char *obj_name = "controller.o";
  const char *map_name = "pid_hashmap";
  const char *enter_prog_name = "detect_syscall_enter";
  const char *exit_prog_name = "detect_syscall_exit";
  const char *syscall_info_bufname = "syscall_info_buffer";
 
  if (argc < 2)
  {
    fatal_error("Usage: ./beetrace ");
  }
  const char *file_path = argv[1];
 
  pid_t pid = fork();
  if (pid == 0)
  {
    int fd = open("/dev/null", O_WRONLY);
    if(fd==-1){
      fatal_error("failed to open /dev/null");
    }
    dup2(fd, 1);
    sleep(2);
    execve(file_path, NULL, NULL);
  }
  else
  {
    printf("Spawned child process with a PID of %d\n", pid);
    obj = bpf_object__open(obj_name);
    if (!obj)
    {
      fatal_error("failed to open the BPF object");
    }
    if (bpf_object__load(obj))
    {
      fatal_error("failed to load the BPF object into kernel");
    }
 
    enter_prog = bpf_object__find_program_by_name(obj, enter_prog_name);
    exit_prog = bpf_object__find_program_by_name(obj, exit_prog_name);
 
    if (!enter_prog || !exit_prog)
    {
      fatal_error("failed to find the BPF program");
    }
    if (!bpf_program__attach(enter_prog) || !bpf_program__attach(exit_prog))
    {
      fatal_error("failed to attach the BPF program");
    }
    syscall_map = bpf_object__find_map_by_name(obj, map_name);
    if (!syscall_map)
    {
      fatal_error("failed to find the BPF map");
    }
    const char *key = "child_pid";
    int err = bpf_map__update_elem(syscall_map, key, 10, (void *)&pid, sizeof(pid_t), 0);
    if (err)
    {
      printf("%d", err);
      fatal_error("failed to insert child pid into the ring buffer");
    }
 
    int rbFd = bpf_object__find_map_fd_by_name(obj, syscall_info_bufname);
 
    struct ring_buffer *rbuffer = ring_buffer__new(rbFd, syscall_logger, NULL, NULL);
 
    if (!rbuffer)
    {
      fatal_error("failed to allocate ring buffer");
    }
 
    if (wait(&status) == -1)
    {
      fatal_error("failed to wait for the child process");
    }
 
    while (1)
    {
      int e = ring_buffer__consume(rbuffer);
      if (!e)
      {
        break;
      }
      sleep(1);
    }
  }
  return 0;
} 
```

And, here are the eBPF programs. The C code will be compiled into a single object file.  
以下是 eBPF 程序。C 代码将被编译成一个单独的目标文件。

```
// controller.c
 
#include "vmlinux.h"
#include #include #include #include "controller.h"
 
struct
{
  __uint(type, BPF_MAP_TYPE_HASH);
  __uint(key_size, 10);
  __uint(value_size, 4);
  __uint(max_entries, 256 * 1024);
} pid_hashmap SEC(".maps");
 
struct
{
  __uint(type, BPF_MAP_TYPE_RINGBUF);
  __uint(max_entries, 256 * 1024);
} syscall_info_buffer SEC(".maps");
 
 
SEC("tracepoint/raw_syscalls/sys_enter")
int detect_syscall_enter(struct trace_event_raw_sys_enter *ctx)
{
  // Retrieve the system call number
  long syscall_nr = ctx->id;
  const char *key = "child_pid";
  int target_pid;
 
  // Reading the process id of the child process in userland
  void *value = bpf_map_lookup_elem(&pid_hashmap, key);
  void *args[MAX_ARGS];
 
  if (value)
  {
    target_pid = *(int *)value;
 
    // PID of the process that executed the current system call
    pid_t pid = bpf_get_current_pid_tgid() & 0xffffffff;
    if (pid == target_pid && syscall_nr >= 0 && syscall_nr < MAX_SYSCALL_NR)
    {
 
      int idx = syscall_nr;
      // Reserve space in the ring buffer
      struct inner_syscall_info *info = bpf_ringbuf_reserve(&syscall_info_buffer, sizeof(struct inner_syscall_info), 0);
      if (!info)
      {
        bpf_printk("bpf_ringbuf_reserve failed");
        return 1;
      }
 
      // Copy the syscall name into info->name
      bpf_probe_read_kernel_str(info->name, sizeof(syscalls[syscall_nr].name), syscalls[syscall_nr].name);
      for (int i = 0; i < MAX_ARGS; i++)
      {
        info->args[i] = (void *)BPF_CORE_READ(ctx, args[i]);
      }
      info->num_args = syscalls[syscall_nr].num_args;
      info->syscall_nr = syscall_nr;
      info->mode = SYS_ENTER;
      // Insert into ring buffer
      bpf_ringbuf_submit(info, 0);
    }
  }
  return 0;
}
 
SEC("tracepoint/raw_syscalls/sys_exit")
int detect_syscall_exit(struct trace_event_raw_sys_exit *ctx)
{
  const char *key = "child_pid";
  void *value = bpf_map_lookup_elem(&pid_hashmap, key);
  pid_t pid, target_pid;
 
  if (value)
  {
    pid = bpf_get_current_pid_tgid() & 0xffffffff;
    target_pid = *(pid_t *)value;
    if (pid == target_pid)
    {
      struct inner_syscall_info *info = bpf_ringbuf_reserve(&syscall_info_buffer, sizeof(struct inner_syscall_info), 0);
      if (!info)
      {
        bpf_printk("bpf_ringbuf_reserve failed");
        return 1;
      }
      info->mode = SYS_EXIT;
      info->retval = ctx->ret;
      bpf_ringbuf_submit(info, 0);
    }
  }
  return 0;
}
 
char LICENSE[] SEC("license") = "GPL"; 
```

Before compiling, we can create a test program which will be traced by our tool.  
在编译之前，我们可以创建一个测试程序，该程序将由我们的工具进行跟踪。

```
#include int main(){
    puts("tracer in action");
    return 0;
} 
```

The following Makefile can be used to compile all the stuff.  
以下 Makefile 可用于编译所有内容。

```
compile:
    clang -O2 -g -Wall -I/usr/include -I/usr/include/bpf -o beetrace loader.c -lbpf
    clang -O2 -g -target bpf -c controller.c -o controller.o
```

Now let’s execute the loader with root privileges.  
现在让我们以 root 权限执行加载器。

```
sudo ./beetrace ./test
```

![](https://bbs.kanxue.com/upload/attach/202409/823361_KB73THW4MR264VC.png)

The entire code can be found in [this](https://github.com/0xSh4dy/bee_tracer) GitHub repository.  
整个代码可以在[这个](https://github.com/0xSh4dy/bee_tracer) GitHub 仓库中找到。

**References**

参考
--

[https://ebpf.io/](https://ebpf.io/)

[https://github.com/libbpf/libbpf](https://github.com/libbpf/libbpf)

HN 评论
-----

_评论链接：[https://news.ycombinator.com/item?id=41156985](https://news.ycombinator.com/item?id=41156985)_

**@khuey：**  
This example doesn't really gain much by using eBPF. The tracepoint machinery and perf_event_open is perfectly capable of recording pids and registers at syscall entry/exit (via PERF_SAMPLE_TID and PERF_SAMPLE_REGS_USER) into a ring buffer. `perf trace` does that today and it can be a useful replacement for strace in situations where strace disturbs the program's timing too much or otherwise cannot be used (e.g. you want to strace something that's already being ptraced by another process).  
通过使用 eBPF，这个示例并没有真正获得太多好处。跟踪点机制和 perf_event_open 完全能够在系统调用入口 / 出口（通过 PERF_SAMPLE_TID 和 PERF_SAMPLE_REGS_USER）将 pid 和寄存器记录到环形缓冲区中。今天，“perf trace” 就做到了这一点，并且在 strace 过多干扰程序时序或无法使用的情况下（例如，您想要 strace 已被另一个进程跟踪的内容），它可以成为 strace 的有用替代品。

Where eBPF is powerful is that it allows you to extend the tracepoint ability to grab more complicated system call arguments. The first register argument to open(2) for instance, is a pointer to the filename to open. Merely reporting the registers is largely useless, the tracer needs to chase the pointer and report the string too. An eBPF filter can be used to recognize that the tracepoint is at an open(2) and to read the userspace memory that the relevant register points to. This enables a "full" strace replacement without using ptrace at all. There's some ongoing work to add this capability to `perf trace`.  
eBPF 的强大之处在于它允许您扩展跟踪点能力以获取更复杂的系统调用参数。例如，open(2) 的第一个寄存器参数是指向要打开的文件名的指针。仅仅报告寄存器基本上是无用的，跟踪器还需要追踪指针并报告字符串。 eBPF 过滤器可用于识别跟踪点位于 open(2) 并读取相关寄存器指向的用户空间内存。这可以实现 “完整”strace 替换，而无需使用 ptrace。目前正在开展一些工作，将此功能添加到“perf trace” 中。

**@T3OU-736:**  
There is an additional aspect to this, I think - `stace` has a hell (order of 100x AFAIK) of an impact on performance of the process being traced. Aside from obvious, this leads to things like hiding race conditions.  
我认为还有一个额外的方面 - `stace` 对被跟踪进程的性能有极大的影响（据我所知大约是 100 倍）。除了明显的问题之外，这还会导致隐藏竞争条件等问题。

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

[#系统内核](forum-41-1-131.htm)