> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1917339-1-1.html)

> [md]# ebpf 在 Android 安全上的应用：ebpf 的一些基础知识 (上篇)## 一、ebpf 介绍 **eBPF 是一项革命性的技术，起源于 Linux 内核，它可以在特权上下文中（如操作系 .......

![](https://avatar.52pojie.cn/data/avatar/001/20/09/26_avatar_middle.jpg)windy_ll

ebpf 在 Android 安全上的应用：ebpf 的一些基础知识 (上篇)
=======================================

一、ebpf 介绍
---------

**eBPF 是一项革命性的技术，起源于 Linux 内核，它可以在特权上下文中（如操作系统内核）运行沙盒程序。它用于安全有效地扩展内核的功能，而无需通过更改内核源代码或加载内核模块的方式来实现。（PS：介绍来源于 [https://ebpf.io/zh-cn/what-is-ebpf/](https://ebpf.io/zh-cn/what-is-ebpf/)）**

**对比 kernel hook，ebpf 最大的优点在于安全和可移植性，在 ebpf 载入之前，需要经过验证器的验证，能够保证内核不会因为 ebpf 程序而出现崩溃，可移植性体现在多版本支持，屏蔽掉了底层的细节，能最大程度保证开发者将重心放在程序的逻辑性上；同样的，ebpf 最大的缺点也体现在了为了保证安全的验证器上，例如循环次数有限制等，导致一些明明可以很简洁的操作在 ebpf 中编程时必须要使用很蠢的方法间接实现（ps：对 kernel hook 感兴趣的可以参考一下我之前的一篇文章 [https://www.52pojie.cn/thread-1672531-1-1.html](https://www.52pojie.cn/thread-1672531-1-1.html)）**

* * *

二、运行环境
------

**OS：Android 模拟器 pixel 6 API level 33 x86_64**

**kernel：5.15.41**

* * *

三、开发工具链
-------

**ebpf 常见的开发工具有如下一些：**

*   `bcc`：BCC 是一个框架，它允许用户编写 python 程序，并将 eBPF 程序嵌入其中。但是 bcc 想将 bcc 运行在 android 上时配置环境时相对麻烦，当然，环境配置好开发难度相比其他工具更低，同时，网上的资料相比其他工具也更多
*   `libbpf`：libbpf 是一个基于 C 的库，包含一个 BPF 加载程序，该加载程序获取已编译的 BPF 目标文件并准备它们并将其加载到 Linux 内核中。 libbpf 承担了加载、验证 BPF 程序并将其附加到各种内核挂钩的繁重工作，使 BPF 应用程序开发人员能够只关注 BPF 程序的正确性和性能。官方链接：[https://github.com/libbpf/libbpf](https://github.com/libbpf/libbpf)
*   `cilium`：cilium 是一个纯 Go 库，提供用于加载、编译和调试 eBPF 程序的实用程序。官方链接：[https://github.com/cilium/ebpf](https://github.com/cilium/ebpf)
*   `Android mk`：谷歌提供的 android 原生 ebpf 支撑，官方链接：[https://source.android.google.cn/docs/core/architecture/kernel/bpf?hl=zh-cn](https://source.android.google.cn/docs/core/architecture/kernel/bpf?hl=zh-cn)
    
    **本系列文章均选择使用`cilium`，经过对比，`bcc`配置环境过于麻烦，不方便快速移植到其他设备上；`libbpf`和`cilium`对比起来，在内核层代码都是`c`写的，区别不大，但是在用户层代码上，`go`还是比`c`更方便编写；至于使用`android mk`的方式，其实最开始选用的是该方案，毕竟是`Android`的原生支持，不论是在数据结构上面还是在函数上面支持度相比较前面几个工具都是最优选择，缺点就是占用资源过大，性能不好的机器编译时长不是一般的长**
    

* * *

四、ebpf 中的数据传输
-------------

**ebpf 中内核和用户层之间的数据传输常用的框架有两种，分别是`perf`和`ringbuffer`，前者是从`kernel module`而来的，而后者是专门为`ebpf`定制的，体验性更好，所有一般都使用后者**  

**在内核层，常规用法为首先使用`bpf_ringbuf_reserve`申请一个`buffer`，然后调用`bpf_ringbuf_submit`提交数据到缓冲区，更详细的可以参考文档 [https://www.kernel.org/doc/html/next/bpf/ringbuf.html](https://www.kernel.org/doc/html/next/bpf/ringbuf.html)**

* * *

五、ebpf 中的常见函数
-------------

*   `bpf_printk`: ebpf 内核层打印函数，用法和`printf`一致，该函数输出到了`/sys/kernel/tracing/trace_pipe`文件中 (PS: 有些系统是 / sys/kernel/debug/tracing/trace_pipe)，值得注意的是，要开启打印，需要将`/sys/kernel/tracing/tracing_on`的值置为 1
*   `bpf_probe_read_user_str`: 从用户空间读取字符串
*   `bpf_probe_read`: 从内核空间读取内存, 以上函数用法都可以参考 [https://man7.org/linux/man-pages/man7/bpf-helpers.7.html](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html)

* * *

六、vmlinux.h
-----------

**`vmlinux.h`是啥？`vmlinux.h`是由工具生成而来的，包含了该机器内核所有的数据结构，有了这个头文件，就避免了我们去官网上查询相应的数据结构，还能避免不同版本之间带来的数据结构变动的问题**

**通常我们使用`bpftool`去生成，命令为`bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h`**

**`bpftool`的`github`链接为 [https://github.com/libbpf/bpftool](https://github.com/libbpf/bpftool)**

* * *

七、ebpf 常见的事件类型
--------------

### 7.1 kprobe

**`kprobe`可以简单理解为在内核插桩，目前有两种形式，分别是`kprobe`和`kretprobe`，前者是在函数开始处插桩，后者则是在函数返回之前插桩，使用举例如下：**

**内核层：**

```
//go:build ignore

#include "vmlinux.h"

char __license[] SEC("license") = "GPL";

struct file_data {
    u32 uid;
    u8 filename[256];
};

struct event {
    struct file_data file;
};

struct {
    __uint(type,BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries,1 << 24);
} events SEC(".maps");

const struct event *unused __attribute__((unused));

SEC("kprobe/do_sys_openat2")
int kprobe_openat(struct pt_regs *ctx)
{
    u32 uid;
    struct event *openat2data;
    char *fp = (char *)(ctx->si);

    uid = bpf_get_current_uid_gid();

    openat2data = bpf_ringbuf_reserve(&events,sizeof(struct event),0);
    if(!openat2data)
    {
        return 0;
    }
    long res = bpf_probe_read_user_str(&openat2data->file.filename,256,fp);
    bpf_printk("uid: %d, filename: %s",uid,openat2data->file.filename);
    openat2data->file.uid = uid;
    bpf_ringbuf_submit(openat2data,0);

    return 0;
}

```

* * *

**用户层：**

```
package main

import (
    "log"
    "os"
    "os/signal"
    "syscall"
    "errors"
    "bytes"
    "encoding/binary"
    "fmt"

    //"github.com/cilium/ebpf"
    "github.com/cilium/ebpf/link"
    "github.com/cilium/ebpf/rlimit"
    "github.com/cilium/ebpf/ringbuf"
    "golang.org/x/sys/unix"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -tags "linux" -type event --target=amd64 bpf blog.c -- -I./headers

func main() {
    stopper := make(chan os.Signal,1)
    signal.Notify(stopper,os.Interrupt,syscall.SIGTERM)

    if err := rlimit.RemoveMemlock(); err != nil {
        log.Fatal(err);
    }

    objs := bpfObjects{}
    if err := loadBpfObjects(&objs,nil); err != nil {
        log.Fatal(err);
    }
    defer objs.Close()

    se, err := link.Kprobe("do_sys_openat2",objs.KprobeOpenat,nil)
    if err != nil {
        log.Fatal(err)
    }
    defer se.Close()

    rd, err := ringbuf.NewReader(objs.Events)
    if err != nil {
        log.Fatal(err)
    }
    defer rd.Close()

    go func() {
        <-stopper

        if err := rd.Close(); err != nil {
            log.Fatal(err)
        }
    }()

    log.Println("Waiting for Data")

    var event bpfEvent

    for {
        record, err := rd.Read()
        if err != nil {
            if errors.Is(err,ringbuf.ErrClosed) {
                log.Println("Received signal, exiting...")
                return
            }
            log.Fatal(err)
            continue
        }
        if err := binary.Read(bytes.NewBuffer(record.RawSample),binary.LittleEndian,&event); err != nil {
            log.Fatal(err)
            continue
        }
        fmt.Printf("[%+v]: filename -> %s\n",event.File.Uid,unix.ByteSliceToString(event.File.Filename[:]))
    }
}

```

**编译：先`go generate`，然后`go build`即可**

**效果图如下：**

![](https://attach.52pojie.cn/forum/202404/24/183839yirraqhrr5a9aqdz.png)

**1.png** _(93.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5MTczNnwxZGExNGFiN3wxNzE0MDA2NTIwfDIxMzQzMXwxOTE3MzM5&nothumb=yes)

2024-4-24 18:38 上传

![](https://attach.52pojie.cn/forum/202404/24/184015kzbir9w8airrogb8.png)

**2.png** _(30.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5MTczN3wzY2Y1MmE4NnwxNzE0MDA2NTIwfDIxMzQzMXwxOTE3MzM5&nothumb=yes)

2024-4-24 18:40 上传

![](https://attach.52pojie.cn/forum/202404/24/184026dxrg0bgvdcnlbvfb.png)

**3.png** _(103.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5MTczOHw2NjFlY2MzNXwxNzE0MDA2NTIwfDIxMzQzMXwxOTE3MzM5&nothumb=yes)

2024-4-24 18:40 上传

**至于`kretprobe`，和`kprobe`区别不大，这里不在举例说明**

### 7.2 tracepoint

**`tracepoint`可以理解为是在源码中预埋的 hook 点位，相比较`kprobe`，稳定性被大大增强，当然缺点也很明显，那就是数量有限，没办法自定义，查看所有`tracepoint`可在`/sys/kernel/tracing/events/`目录下找到所有可追踪的事件 (PS: 有些机器可能是在`/sys/kernel/debug/tracing/events/`下)，事件的格式信息在相应的事件目录下的`format`文件中**

![](https://attach.52pojie.cn/forum/202404/24/184036iisl26yic3hc7whg.png)

**4.png** _(82.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5MTczOXw1OTZjYjQ2OHwxNzE0MDA2NTIwfDIxMzQzMXwxOTE3MzM5&nothumb=yes)

2024-4-24 18:40 上传

![](https://attach.52pojie.cn/forum/202404/24/184047j1bks23uzwx16zxg.png)

**5.png** _(47.65 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5MTc0MHxlYjAzZGFlZXwxNzE0MDA2NTIwfDIxMzQzMXwxOTE3MzM5&nothumb=yes)

2024-4-24 18:40 上传

**内核层：**

```
//go:build ignore

#include "vmlinux.h"

char __license[] SEC("license") = "GPL";

struct sys_enter_args {
   unsigned short common_type;
   unsigned char common_flags;
   unsigned char common_preempt_count;
   int common_pid;

   long id;
   unsigned long args[6];
};

SEC("tracepoint/raw_syscalls/sys_enter")
int trace_sys_enter(struct sys_enter_args *args)
{
    u32 syscall_nr;

    syscall_nr = args->id;

    bpf_printk("syscall_nr: %d",syscall_nr);

    return 0;
}

```

**`bpf_printk`函数打印的结果在`/sys/kernel/tracing/trace_pipe`文件中 (PS: 有些机型在`/sys/kernel/debug/tracing/trace_pipe`文件中，下同，下面的不在重复解释)，观看`bpf_printk`函数结果需要先将`/sys/kernel/tracing/tracing_on`文件中的值置为`1`**

**用户层：**

```
package main

import (
    "log"
    "time"

    "github.com/cilium/ebpf/link"
    "github.com/cilium/ebpf/rlimit"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go --target=amd64 bpf blog.c -- -I./headers

func main() {

    if err := rlimit.RemoveMemlock(); err != nil {
        log.Fatal(err)
    }

    objs := bpfObjects{}
    if err := loadBpfObjects(&objs, nil); err != nil {
        log.Fatalf("loading objects: %v", err)
    }
    defer objs.Close()

    kp, err := link.Tracepoint("raw_syscalls","sys_enter",objs.TraceSysEnter,nil)
        if err != nil {
            log.Fatal(err)
        }
        defer kp.Close()

    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    log.Println("Waiting for events..")

    for range ticker.C {
        log.Printf("get rule\n")
        }
}

```

**效果图如下：**

![](https://attach.52pojie.cn/forum/202404/24/184101vpp1frpk8zi1pakc.png)

**6.png** _(160.87 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5MTc0MXw3OTM0ZTFlNnwxNzE0MDA2NTIwfDIxMzQzMXwxOTE3MzM5&nothumb=yes)

2024-4-24 18:41 上传

### 7.3 其他事件类型

**`ebpf`还有其他事件类型，例如`socket`、`sockops`、`tc`、`xdp`等等，但这些更多与流量控制息息相关，跟我们在移动安全上的关联性不是很大，这里不在举例说明，当然还有`uprobe`事件类型，这个是用户层插桩的，但用户层插桩更推荐`frida`这些框架，而且`uprobe`在`linux`使用体验感还好，在`Android`端使用去插桩`APP`过于麻烦了。**

* * *

八、一些使用技巧
--------

### 8.1 将数据从用户空间传输到内核空间

**在`cilium`中，`ringbuffer`并不支持将数据从用户空间传递到内核空间，只支持将数据从内核空间发送到用户空间，在新的数据传输框架`BPF_MAP_TYPE_USER_RINGBUF`支持将数据从用户空间传输到内核空间，但是遗憾的是，`cilium`暂不支持该框架**

**在我们需要传输一些过滤条件或者动态的全局配置到内核层去过滤的时候需要怎么做喃？可以考虑监控特定的文件名、特定的命令等来获取数据，当然这种方式仅时候传递数据量不大的情况**

### 8.2 获取 UID

**UID 是啥，UID 是 android 中 uid 用于标识一个应用程序，uid 在应用安装时被分配，并且在应用存在于手机上期间，都不会改变，可以理解为 app 的唯一身份标识，在 ebpf 中，可以用来过滤指定 app 的数据**

**`ebpf`可以使用`bpf_get_current_uid_gid`函数来获取 UID，该函数返回值为`u32`类型**

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 这两天刚好在学 ebpf mark 一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Corax 谢谢分享～![](https://avatar.52pojie.cn/images/noavatar_middle.gif)jifL88 看来是时候卷内核和 ebpf 了。  赞 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) hjw01  
围观大佬