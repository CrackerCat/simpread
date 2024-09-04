> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283255.htm)

> [原创] 逆向进入内核时代之 APatch 源码学习 (02. 手术刀医生 kptools 及如何阅读关键内核代码)

前一章节我们了解如何编译内核及如何进行模拟调试, 除了完成了模拟环境的搭建, 其中有个重要的环节就是使用 kptools 进行内核 Patch.

*   kptools 就像一个拿手术刀的医生, 小心解剖着内核文件, 往身体里面塞入了 kpimg 和 kpatch 这两个外挂器官, 同时心脏搭桥 (hook), 血液循环, 让外挂器官寄生在内核里面, 这神乎奇迹的刀法, 上次大约还是在周星驰的 007 里面见过.
    
*   可想而知, 这个医生绝对需要对内核文件了如指掌, 才能完成这样绝妙的工作. 而我们作为一个见习医生, 也需要对内核文件略知一二才行:
    

这个小结, 看完后需要掌握知识点:
-----------------

1.  vmlinux.lds 是什么, 对应的作用是什么?
2.  内核入口文件 Head.S 是什么?
3.  System.map 是什么, 对应的作用?

在 Android 真机上面使用 kptools 进行 patch:

```
cd /data/data/me.bmax.apatch/patch 或者 run-as me.bmax.apatch
./kptools -p -i Image -S a12345678 -k kpimg -o Image2 -K kpatch
```

当然在 linux 环境下也可以使用 kptools-linux 直接进行 patch, 我当时在手机上面折腾的, 绕了个远路, 怎么说呢, 有得有失.

00.kptools 做了哪些事情呢?
-------------------

简单来说 kptools 以 Image(未压缩内核镜像) 作为输入, 嵌入 kpimg 和 kpatch 到文件尾部. 同时对内核进行 inlinehook 和填充对应启动参数. 这里通过一张图说明 kptools 做了什么  
![](https://bbs.kanxue.com/upload/attach/202409/935696_DP5E4T7VSME93TP.webp)

01. 起步阶段 (先建立感官认知)
------------------

如果我们想知道 kptools 如何做到修改内核执行顺序, 要解答这个问题, 我们需要对内核有一些最基础的了解, 了解啥呢?

1.  先知道 Image 到底是什么? 从底层观察一下?
2.  对 Patch 前后做一些差异对比?

1.1 了解一个文件到底是什么, 最好的方式就是打开文件看看.  
对应工具: [rehex](https://github.com/solemnwarning/rehex)  
对应文件: [KernelImage](https://github.com/nzcv/KernelPatchQEMU/releases/download/dev1.0.7/Image)

![](https://bbs.kanxue.com/upload/attach/202409/935696_STUU4F3UJ3QSVCD.webp)  
从文件本身观察特征有个 ARMd, 我们通过一些网上的资料也可以了解到内核文件本身就是一个可执行文件.

1.2 通过 IDA 打开文件看看

IDA 并不能识别对应架构和文件被加载到内存后对应的虚拟内存布局, 我们需要手动填写, 具体填什么内容呢???  
![](https://bbs.kanxue.com/upload/attach/202409/935696_TUXEVGYD2JD78MG.webp)  
![](https://bbs.kanxue.com/upload/attach/202409/935696_NFDMB45BE9FVY8K.webp)

1.3 Patch 前后文件对比  
![](https://bbs.kanxue.com/upload/attach/202409/935696_PK67RWK8MNNB4JN.webp)

通过文件前后对比, 我们可以最简单的发现 kptools 对前 4 个字节做了对应修改, 这 4 个字节的含义是什么呢? 是不是问号越来越多了??? 他是我们学习的突破口, 三言 2 语也很难说清楚, 后面慢慢解答.

02. 深究细节 (前 4 个字节到底是什么?)
------------------------

内核 (操作系统) 作为程序员 3 专研对象好之一, 依然没有逃离计算机编译原理, 会经历预处理 ->编译 ->链接三阶段. 其中链接阶段可以通过. lds 进行手动编排(细心的朋友会发现 kpimg 也通过 kpimg.lds 进行了编排)

### 2.1 内核文件通过 arch/arm64/kernel/vmlinux.lds 进行布局编排

![](https://bbs.kanxue.com/upload/attach/202409/935696_A9N9RMWE2FREMRZ.webp)

上面的文件我们可以得知两个信息:

1.  内核文件在虚拟内存空间, 起始位置 0xffff000008080000
2.  .head.text 会被编排在起始位置

```
Python 3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> ((((0xffffffffffffffff - (1 << (48)) + 1) + (0)) + (0x08000000))) + 0x00080000
>>> hex(((((0xffffffffffffffff - (1 << (48)) + 1) + (0)) + (0x08000000))) + 0x00080000)
'0xffff000008080000'
```

```
include/linux/init.h
95:#define __HEAD               .section        ".head.text","ax"
```

### 2.2 顺藤摸瓜找到 head.S

![](https://bbs.kanxue.com/upload/attach/202409/935696_S69P5X6927HWP9F.webp)

前面我们知道. head.text 是__HEAD 对应的宏定义, 我们代码搜索就能找到 head.S 文件. 通过代码 + 注释我们可以获得大概的信息, 这部分是 Image header, 被 bootloader 使用. 同时代码通过 CONFIG_EFI 进行控制，UEFI 一般常用于 PC 方法启动，手机并不是配置为 UEFI。

```
    __HEAD
_head:
    /*
     * DO NOT MODIFY. Image header expected by Linux boot-loaders.
     */
#ifdef CONFIG_EFI
    /*
     * This add instruction has no meaningful effect except that
     * its opcode forms the magic "MZ" signature required by UEFI.
     */
    add x13, x18, #0x16
    b   stext
#else
    b   stext               // branch to kernel start, magic
    .long   0               // reserved
#endif
    le64sym _kernel_offset_le       // Image load offset from start of RAM, little-endian
    le64sym _kernel_size_le         // Effective size of kernel image, little-endian
    le64sym _kernel_flags_le        // Informative flags, little-endian
    .quad   0               // reserved
    .quad   0               // reserved
    .quad   0               // reserved
    .ascii  "ARM\x64"           // Magic number
#ifdef CONFIG_EFI
    .long   pe_header - _head       // Offset to the PE header.
 
pe_header:
    __EFI_PE_HEADER
#else
    .long   0               // reserved
#endif
```

_kernel_offset_le，_kernel_size_le， _kernel_flags_le 分别使用小端的方式存储内核启动偏移，内核文件大小. 这里重点讲解_kernel_flags_le

```
#define __HEAD_FLAGS        ((__HEAD_FLAG_BE << 0) |    \
                 (__HEAD_FLAG_PAGE_SIZE << 1) | \
                 (__HEAD_FLAG_PHYS_BASE << 3))
 
/*
 * These will output as part of the Image header, which should be little-endian
 * regardless of the endianness of the kernel. While constant values could be
 * endian swapped in head.S, all are done here for consistency.
 */
#define HEAD_SYMBOLS                        \
    DEFINE_IMAGE_LE64(_kernel_size_le, _end - _text);   \
    DEFINE_IMAGE_LE64(_kernel_offset_le, TEXT_OFFSET);  \
    DEFINE_IMAGE_LE64(_kernel_flags_le, __HEAD_FLAGS);
```

通过宏定义我们可以知晓__HEAD_FLAGS 包含了内核本身使用大端还是小端进行存储， 以及 PAGE_SIZE, PHYS_BASE. 这里 kptools 对 flags 进行了读取分析。这样你就可以映射上阅读对应的代码了。

### 2.3 终极解答前 4 个字节到底是什么?

![](https://bbs.kanxue.com/upload/attach/202409/935696_HJFVJKKW8VZXDTU.webp)

![](https://bbs.kanxue.com/upload/attach/202409/935696_WT2JC4KNK36VYEV.webp)

对的就是一个 b 跳转

让疑惑来得更猛烈些吧?
===========

kptools 这个神医把它桥接到什么地方去了呢? 预知后事如何, 关注我, 收听下会分解.

[[课程]FART 脱壳王！加量不加价！FART 作者讲授！](https://bbs.kanxue.com/thread-281194.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm)