> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-215953.htm)

# 来自高纬的对抗 - 逆向 TinyTool 自制

Author: ThomasKing

Date: 2017.02.24

Weibo/Twitter: ThomasKing2014

Blog: thomasking2014.com

## 一、序

无论是逆向分析还是漏洞利用，我所理解的攻防博弈无非是二者在既定的某一阶段，以高纬的方式进行对抗，并不断地升级纬度。比如，逆向工程人员一般会选择在 Root 的环境下对 App 进行调试分析，其是以 root 的高权限对抗受沙盒限制的低权限；在 arm64 位手机上进行 root / 越狱时，ret2usr 利用技术受到 PXN 机制的约束，厂商从修改硬件特性的高纬度进行对抗，迫使漏洞研究者提高利用技巧。

下文将在 Android 逆向工程方面，分享鄙人早期从维度攻击的角度所编写的小工具。工具本身可能已经不能适应现在的攻防，“授人以鱼不如授人以渔”，希望能够给各位读者带来一些思路，构建自己的分析利器。

## 二、正

### 0x00 自定义 Loader

早期 Android 平台对 SO 的保护采用畸形文件格式和内容加密的方式来对抗静态分析。随着 IDA 以及 F5 插件地不断完善和增多，IDA 已经成为了逆向人员的标配工具。正因如此，IDA 成为了畸形文件格式的对抗目标。畸形方式从减少文件格式信息到构造促使 IDA 加载 crash 的变化正应证了这一点。对此，鄙人研究通过重建 [文件格式信息](http://bbs.pediy.com/thread-192874.htm) 的方式来让 IDA 正常加载。

在完成编写修复重建工具不久之后，鄙人在一次使用 IDA 的加载 bin 文件时，猛然意识到畸形文件格式的对抗目标是 IDA 对 ELF 文件的加载的默认 loader。既然防御的假象和维度仅仅在于默认 loader，那么以自定义的 loader 加载实现高纬攻击，理论是毫无敌手的。

那如何来实现 IDA 自定义 loader 呢？

1. 以 Segment 加载的流程对 ELF 文件进行解析，获取和重建 Section 信息 (参看上面所说贴子)。

2. 把文件信息在 IDA 中进行展示，直接调用对应的 IDAPython 接口

实现加载 bin 文件的 py 代码见文末 github 链接，直接放置于 IDA/loaders 目录即可。由于早期少有 64 位的安卓手机，加载脚本仅支持 arm 32 位格式，有兴趣读者可以改写实现全平台通用。不同 ndk 版本所编译文件中与动态加载无关的 Section 不一定存在，注释相应的重建代码即可。

### 0x01 Kernel Helper

以 APP 分析为例，对于加固过的应用通常会对自身的运行环境进行检测。比如: 检测自身调试状态，监控 proc 文件等。相信各位读者有各种奇淫技巧来绕过，早期鄙人构建 hook 环境来绕过。从维度的角度，再来分析这种对抗。对于 APP 或者 bin 文件而言，其仅运行于受限的环境中，就算 exp 提权后也只是权限的提升和对内核有一定的访问控制权。对于 Android 系统而言，逆向人员不仅能够拿到 root 最高权限，而且还可以修改系统的所有代码。从攻防双方在运行环境的维度来看，“魔” 比” 道 “高了不只三丈，防御方犹如板上鱼肉。而在代码维度，防御方拥有源代码的控制权，攻防处于完全劣势。随着代码混淆和 VMP 技术的运用，防御方这块鱼肉越来越不好 "啃"。

对于基于 linux 的安卓系统而言，进程的运行环境和结构是由内核来提供和维护的。从修改内核的维度来对抗，能达到一些不错的效果。下文将详述在内核态 dump 目标进程内存和系统调用监控。

### 1. 内存 DUMP

对内核添加一些自定义功能时，通常可以采用内核驱动来实现。虽然一部分 Android 手机支持驱动 ko 文件加载，但内核提供的其他工具则不一定已经编译到内核，在后文中可以看到。nexus 系列手机是谷歌官方所支持的，编译刷机都比较方便，推荐使用。

S1. 编译内核

为了让内核支持驱动 ko 文件的加载，在 make memuconfig 配置内核选项时，以下勾选:

[*] Enable loadable module support

次级目录所有选项

编译步骤参看谷歌官方提供的内核编译步骤。

S2. 驱动代码

linux 系统支持多种驱动设备，这里采用最简单的字符设备来实现。与其他操作系统类似，linux 驱动程序也分为入口和出口。在 module\_init 入口中，对字符设备进行初始化，创建 / dev/REHelper 字符设备。文末代码采用传统的方式对字符设备进行注册，也可直接使用 misc 的方式。字符设备的操作方式通过注册 file\_operations 回调实现，其中 ioctl 函数比较灵活，满足实现需求。

定义 command ID:

#define CMD_BASE 0xC0000000

#define DUMP_MEM (CMD_BASE + 1)

#define SET_PID  (CMD_BASE + 2)

构建 dump_request 参数:

struct dump_request{

   pid_t pid; // 目标进程

   unsigned long addr; // 目标进程 dump 起始地址

   ssize_t count; //dump 的字节数

   char __user *buf; // 用户空间存储 buf

};

在 ioctl 中实现分支:

    case DUMP_MEM:

        target_task = find_task_by_vpid(request->pid); // 对于用户态，进程通过进程的 pid 来标示自身；在内核空间，通过 pid 找到对应的进程结构 task_struct

        if(!target_task){

            printk(KERN_INFO "find_task_by_vpid(%d) failed\n", request->pid);

            ret = -ESRCH;

            return ret;

        }

        request->count = mem_read(target_task->mm, request->buf, request->count, request->addr); // 进程的虚拟地址空间同样由内核进程管理，通过 mm_struct 结构组织

mem\_read 其实是对 mem\_rw 函数的封装，mem\_rw 能够读写目标进程，简略流程:

static ssize_t mem_rw(struct mm_struct *mm, char __user *buf,

           size_t count, unsigned long addr, int write)

{

   ssize_t copied;

   char *page;

...

   page = (char *)__get_free_page(GFP_TEMPORARY); // 获取存储数据的临时页面

...

   while (count> 0) {

       int this_len = min_t(int, count, PAGE_SIZE);

 // 将写入数据从用户空间拷贝到内核空间

       if (write && copy_from_user(page, buf, this_len)) {

           copied = -EFAULT;

           break;

       }

// 对目标进程进行读或写操作，具体实现参看内核源码

       this_len = access_remote_vm(mm, addr, page, this_len, write);

// 将获取到的目标进程数据从内核拷贝到用户空间

       if (!write && copy_to_user(buf, page, this_len)) {

           copied = -EFAULT;

           break;

       }

...  

   }

   ...

}

内核驱动部分的 dump 功能实现，接着只需在用户空间访问驱动程序即可。

// 构造 ioctl 参数

request.pid = atoi(argv[1]);

request.addr = 0x40000000;

request.buf = buf;

request.count = 1000;

// 打开内核驱动

int fd = open("/dev/REHelper", O_RDWR);

// 发送读取命令

ioctl(fd, DUMP_MEM, &request);

close(fd);

S3. 测试

文末代码中，dump\_test 为目标进程，dump\_host 通过内核驱动获取目标进程的数据。insmod 和 dump\_host 以 root 权限运行即可。

### 2. 系统调用监控

通常情况下，APP 通过动态链接库 libc.so 间接的进行系统调用，直接在用户态 hook libc.so 的函数即可实现监控。而对于静态编译的 bin 文件和通过 svc 汇编指令实现的系统调用，用户态直接 hook 是不好处理的。道理很简单，系统调用由内核实现，hook 也应该在内核。

linux 系统的系统调用功能统一存在 syscall 表中，syscall 表通常编译放在内核映像的代码段，修改 syscall 表需要修改内核页属性，感兴趣的读者可以找到 linux rootkit 方面的资料。本文对系统调用监控的实现，采用内核从 2.6 支持的 probe 功能来实现，选用的最重要原因是：通用性。在不同 abi 平台通过汇编实现系统调用的读者应该知道，不同 abi 平台的系统调用功能号并不一定相同，这就意味其在 syscall 表中的数组索引是不一致的，还需要额外的判定，实现并不优雅。

linux 内核提供了 kprobe、jprobe 和 kretprobe 三种方式。限于篇幅，仅介绍利用 jprobe 实现系统调用监控。感兴趣的读者可以参看内核 Documentation/kprobes.txt 文档以及 samples 目录下的例子。

S1. 编译选项

为了能够支持 probe 功能，需在上述开启驱动 ko 编译选项的基础上勾选 kprobe 选项。如果没有开启内核驱动选项，是不会有 kprobes(new) 选项的

General setup --->

[*] Kprobes(New)

S2. 驱动代码

以监控 sys_open 系统调用为例。首先，在 module\_init 函数中对调用 register\_jprobes 进行注册。注册信息封装在 struct jprobe 结构中。

static struct jprobe open_probe = {

   .entry          = jsys_open, // 回调函数

   .kp = {

       .symbol_name    = "sys_open", // 系统调用名称

   },

};

由于系统调用为所有进程提供服务，不加入过滤信息会造成监控信息过多。回调函数的声明和被监控系统调用的声明一致。

asmlinkage int jsys_open(const char *pathname, int flags, mode_t mode){

    pid_t current_pid = current_thread_info()->task->tgid;    

    // 从当前上下文中获取进程的 pid

// monitor_pid 初始化 - 1，0 为全局监控。

    if(!monitor_pid || (current_pid == monitor_pid)){

        printk(KERN_INFO "[open] pathname %s, flags: %x, mode: %x\n", 

            pathname, flags, mode);

    }

    jprobe_return();

    return 0;

}

对 monitor_pid 的设置通过驱动的 ioctl 来设置，参数简单直接设置。

case SET_PID:

    monitor_pid = (pid_t) arg;

S3. 测试

文末代码 bin\_wrapper 和 ptrace\_trace 均为静态编译，bin\_wrapper 通过设置监控对 ptrace\_trace 的进行监控。内核 prink 的打印信息通过 cat /proc/kmsg 获取，输出类似如下:

<6>[34728.283575] REHelper device open success!

<6>[34728.285504] Set monitor pid: 3851

<6>[34728.287851] [openat] dirfd: -100, pathname /dev/__properties__, flags: a8000, mode: 0

<6>[34728.289348] [openat] dirfd: -100, pathname /proc/stat, flags: 20000, mode: 0

<6>[34728.291325] [openat] dirfd: -100, pathname /proc/self/status, flags: 20000, mode: 0

<6>[34728.292016] [inotify_add_watch]: fd: 4, pathname: /proc/self/mem, mask: 23

<6>[34729.296569] PTRACE_PEEKDATA: [src]pid = 3851 --> [dst]pid = 3852, addr: 40000000, data: be919e38

## 三、尾

本文介绍了鄙人对攻防的维度思考，以及从维度分析来实现的早期工具的部分介绍。希望能够给各位读者带来一些帮助和思考。限于鄙人水平，难免会有疏漏或者错误之处，敬请各位指出，谢谢。

## 四、附

https://github.com/ThomasKing2014/ReverseTinytoolDemo

======================================

新版不支持 MD 十分尴尬.

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

上传的附件：

*   [逆向 TinyTool 自制. pdf](javascript:void(0)) （278.18kb，60 次下载）