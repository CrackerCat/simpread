> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266527.htm)

问题原由
====

函数抽取壳是当前最为流行的 DEX 加壳方式之一，这种加壳方式的主要流程包含两个步骤：一、将 DEX 中需要保护的函数指令置空（即抽取函数体）；二、在应用启动的过程中，HOOK 类的加载过程，比如 ClassLinker::LoadMethod 函数，然后及时回填指令。

笔者在实现抽取壳的过程中遇到了一个问题，即在步骤二回填指令之前，需要先调用 mprotect 将目标内存设置为 “可写”，但在初次尝试过程中一直调用失败，于是有了今天这篇文章。

本文探讨的主要内容是 mprotect 调用失败的根本原因，以及在加壳实现中的解决方案，通过本文的阐述，一方面能够帮助遇到同类问题的小伙伴解决心中的疑惑，另一方面能够给大家提供可落地的实现方案。

调用 mprotect 修改内存失败的现象
=====================

以下代码块截取自自定义 LoadMethod 函数，其目标是将目标函数指令所在内存页的属性修改为可写——通过 mprotect 函数的参数 “PROT_WRITE” 指定，实际结果是 mprotect 调用失败了，返回”-1“，errno 为”13“。

```
int pagesize = sysconf(_SC_PAGESIZE);
int protectsize = pagesize;
byte *code_item_start = static_cast(code_item_addr) + 16;
void *protectaddr = (void*) ((int) code_item_start - ((int) code_item_start % pagesize) - pagesize);
LOGD("process:%d,enter loadmethod:protectaddr:%p,protectsize:%d", getpid(), protectaddr, protectsize);
int result = mprotect(protectaddr, protectsize,  PROT_WRITE);
LOGD("mprotect return 0: %d, errno: %d", result, errno); 
```

”13“号 errno 的符号为 EACCES，查看 linux 手册可知是权限问题。手册中给出一个可能的场景，即如果使用 mmap 映射一个以” 只读 “模式打开的文件，然后使用 mprotect 尝试修改内存属性为可写，就会返回 EACCES 错误。

```
EACCES The memory cannot be given the specified access.  This can
              happen, for example, if you mmap(2) a file to which you
              have read-only access, then ask mprotect() to mark it
              PROT_WRITE.

```

接下来我们将沿着这个可能的场景，首先验证 DEX 文件是否以只读模式打开，然后再进行下一步分析。

mprotect 调用失败的原因分析
==================

使用 strace 跟踪应用的系统调用，验证了 DEX 文件的打开模式为只读模式——"O_RDONLY"，然后通过 mmap2 将 DEX 文件映射进内存，内存属性为只读的私有映射。

```
[pid 13190] openat(AT_FDCWD, "/storage/emulated/0/payload.dex", O_RDONLY|O_LARGEFILE) = 49
mmap2(NULL, 2121728, PROT_READ, MAP_PRIVATE, 49, 0) = 0xcef7a000

```

为了进一步证实并彻底理清背后的逻辑，我研究了下 mprotect 的设计文档 [1]。mprotect 是用户空间 PAX 的一部分，它的核心目标是缓解可利用内存漏洞被利用的情况，所以我理解 mprotect 实际上就是 “memory protect”，它的主要目的是从安全的角度保护内存：

> The goal of MPROTECT is to help prevent the introduction of new executable
> 
>    code into the task's address space. This is accomplished by restricting the
> 
>    mmap() and mprotect() interfaces.

mprotect 通过内存属性控制内存的访问权限，其中安全状态良好的属性组合包括如下几种：

> VM_WRITE
> 
> VM_MAYWRITE
> 
> VM_WRITE | VM_MAYWRITE
> 
> VM_EXEC
> 
> VM_MAYEXEC
> 
> VM_EXEC | VM_MAYEXEC

即内存要么是 “可写” 的，要么是 “可执行” 的，“可写”与 “可执行” 必须互斥，这样才能阻断 “写入并执行” 的内存攻击。

理解了 mprotect 的设计理念之后，我们再回到本文所遇到的问题本身：为什么以只读方式打开的 DEX 文件映射到内存之后，无法使用 mprotect 修改为 “可写” 内存？

根据 mprotect 设计文档的阐述，mprotect 主要通过 VM_MAYWRITE 控制内存是否可被设置为 “可写”，该属性的设置时机在 mmap 调用之时：

> VM_WRITE | VM_MAYWRITE or VM_MAYWRITE if PROT_WRITE was requested at
> 
> mmap() time

mmap 首先将所有可能的属性标致置位，然后再进行合法性检查：

kernel/msm/+/refs/heads/android-msm-vega-4.4-oreo-daydream/mm/mmap.c

```
/* Do simple checking here so the lower-level routines won't have
 * to. we assume access permissions have been handled by the open
 * of the memory object, so we don't do any here.
 */
vm_flags |= calc_vm_prot_bits(prot) | calc_vm_flag_bits(flags) |
mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;

```

如果文件打开时未设置 “可写” 属性，则清除 “VM_MAYWRITE” 属性。

kernel/msm/+/refs/heads/android-msm-vega-4.4-oreo-daydream/mm/mmap.c

```
if (!(file->f_mode & FMODE_WRITE))
 
vm_flags &= ~(VM_MAYWRITE | VM_SHARED);

```

最后 mprotect 会对相关属性进行检查，如果 VM_MAYWRITE 没有被设置，则不可通过 mprotect 设置内存的写属性，返回 EACCES 错误标识：

kernel/msm/+/refs/heads/android-msm-vega-4.4-oreo-daydream/mm/mprotect.c

```
/* newflags >> 4 shift VM_MAY% in place of VM_% */
if ((newflags & ~(newflags >> 4)) & (VM_READ | VM_WRITE | VM_EXEC)) {
error = -EACCES;
goto out;
}

```

通过 strace 日志可以证实 mmap DEX 文件到内存的过程中并没有设置 VM_MAYWRITE 和 VM_WRITE，所以直接使用 mprotect 设置内存为 “可写” 的行为会被拒绝。

```
mmap2(NULL, 2121728, PROT_READ, MAP_PRIVATE, 49, 0) = 0xcef7a000

```

综上，mprotect 修改内存为可写的整个逻辑如下：

系统以只读模式打开 DEX 文件，所以 mmap 在映射文件时清除了 VM_MAYWRITE 标志，导致接下来在调用 mprotect 修改内存为可写的过程中，mprotect 检测目标内存未设置 VM_MAYWRITE 标志，返回 EACCES 错误代码。

两种可行的解决方案
=========

在研究清楚原因之后，我们再来聊聊可能的解决方案。我这里给出两种经过验证的思路：

1）hook openat 函数，设置文件打开时的属性为可读写——O_RDWR；

```
if(strstr(pathname,"payload")){
        LOGD("[myopenat] path: %s, flags: %d", (char*)pathname, flags);
        flags &= (~O_RDONLY);
        flags |= O_RDWR;
    }

```

2）hook mmap 函数，或者在 mmap 之前修改传入 mmap 的标签，直接将内存属性修改为 “可写”。这里我们以后面一种思路为例，HOOK MemMap::MapFileAtAddress 函数，在调用 mmap 映射文件之前修改 prot 参数：

art/runtime/mem_map.cc

```
void* myMapFileAtAddr(int expected_ptr, int byte_count, int prot, int flags, int fd, int start, int low_4gb, int reuse, char *filename, int error_msg){
    if(strstr(filename, "payload"))
    {
        LOGD("[myMapFileAtAddr] file name contains 'payload': %s, prot: %d, flags: %d, fd: %d", filename, prot, flags, fd);
        prot |= PROT_WRITE;
    }
    void* res = oriMapFileAtAddr(expected_ptr, byte_count, prot, flags, fd, start, low_4gb, reuse, filename, error_msg);
    return res;
}

```

小结
==

网络上很多关于抽取壳实现的教程都没有提过 mprotect 的问题，默认 mprotect 修改内存是成功的，这可能是因为大多数人都是通过模拟器进行实验。然而，如果我们要做线上的加壳产品，面向生产环境进行开发的话，mprotect 调用失败的问题大概率会遇到，希望本文能有所帮助。

参考：

[1].mprotect 设计文档：https[:][/][/]pax[.]grsecurity[.]net[/]docs[/]mprotect[.]txt

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 14 小时前 被 kxliping 编辑 ，原因：