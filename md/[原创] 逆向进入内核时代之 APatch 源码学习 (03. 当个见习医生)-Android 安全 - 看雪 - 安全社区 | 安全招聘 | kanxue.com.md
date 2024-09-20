> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283541.htm)

> [原创] 逆向进入内核时代之 APatch 源码学习 (03. 当个见习医生)

话不多说, 咱们续接上回. 前面一个章节我们了解到 kptools 对内核的第一条指令进行了 inlinehook, 修改了跳转目标地址.

KP 医生是如何篡改的呢?
-------------

前一章节我们了解到 KernelPatchQEMU/arch/arm64/kernel/head.S 前几行代码的含义.

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
```

KP 医生做的事情非常简单, 就是梳理经络, 把上面的指令包装为一个数据结构, 按需读取解析就好了.

```
//KernelPatch/tools/image.c
typedef struct
{
    union _entry
    {
        // #ifdef CONFIG_EFI
        struct _efi
        {
            uint8_t mz[4]; // "MZ" signature required by UEFI.
            uint32_t b_insn; // branch to kernel start, magic
        } efi;
        // #else
        struct _nefi
        {
            uint32_t b_insn; // branch to kernel start, magic
            uint32_t reserved0;
        } nefi;
        // #endif
    } hdr;
 
    uint64_t kernel_offset;  // Image load load_offset from start of RAM, little-endian
    uint64_t kernel_size_le; // Effective size of kernel image, little-endian
    uint64_t kernel_flag_le; // Informative flags, little-endian
 
    uint64_t reserved0;
    uint64_t reserved1;
    uint64_t reserved2;
 
    char magic[4]; // Magic number "ARM\x64"
 
    union _pe
    {
        // #ifdef CONFIG_EFI
        uint64_t pe_offset; // Offset to the PE header.
        // #else
        uint64_t npe_reserved;
        // #endif
    } pe;
} arm64_hdr_t;
```

对应文件解析在 get_kernel_info 这个函数, 其中最重要的就是记录下 B 跳转偏移因为后面还需跳转回来呢. 这里我们需要对 B 指令有详细的理解, 不然有些常数很难理解.

```
/**
读取kenrel header信息
*/
int32_t get_kernel_info(kernel_info_t *kinfo, const char *img, int32_t imglen)
{
    //....
    b_primary_entry_insn = u32le(b_primary_entry_insn);
    if ((b_primary_entry_insn & 0xFC000000) != 0x14000000) {
        tools_loge_exit("kernel primary entry: %x\n", b_primary_entry_insn);
    } else {
        //因为第一条指令是b 跳转指令, 我们需要了解下b指令的基础含义是什么,才能看懂0x03ffffff的用意
        uint32_t imm = (b_primary_entry_insn & 0x03ffffff) << 2;       
        kinfo->primary_entry_offset = imm + b_stext_insn_offset;
    }
}
```

理解下 0x03ffffff 和 <<2
--------------------

ARM64 指令集（也称为 AArch64）是 ARM 架构在 64 位模式下的指令集。B 指令是其中的一条分支指令，用于无条件跳转到指定的地址。以下是对 B 指令的详细解释：  
语法

```
B 
```

功能  
B 指令的主要功能是将程序计数器（PC）跳转到指令中指定的地址或标签。跳转是无条件的，也就是说，不需要任何条件判断。  
操作码格式  
ARM64 指令是固定长度的 32 位指令，其中 B 指令的操作码格式如下：

```
31  30 29 28 27 26 | 25 24 23 22 21 20 19 18 17 16 15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
    op             |                imm26
```

这里补充说明下, 因为有很多人疑问为啥需要 <<2:  
Q: 为什么 <<2(2024 年 5 月 31 日)  
A: 如果对这个有疑惑, 那么说明我们对 B 指令的理解还不够仔细. 其实文档里面说的很明白.  
b 指令里面的偏移是怎么计算的呢?  
offset = (targetadder - pcadder - 8) / 4  
// 减 8，指令流水造成  
// 除 4，因为指令定长，存储指令个数差，而不是地址差  
大家可以模拟调试下...

page_shift 识别
-------------

```
kinfo->load_offset = u64le(khdr->kernel_offset);
   kinfo->kernel_size = u64le(khdr->kernel_size_le);
 
   uint8_t flag = u64le(khdr->kernel_flag_le) & 0x0f;
   kinfo->is_be = flag & 0x01;
 
   if (kinfo->is_be) tools_loge_exit("kernel unexpected arm64 big endian img\n");
 
   switch ((flag & 0b0110) >> 1) {
   case 2: // 16k
       kinfo->page_shift = 14;
       break;
   case 3: // 64k
       kinfo->page_shift = 16;
       break;
   case 1: // 4k
   default:
       kinfo->page_shift = 12;
   }
```

内核在编译时会配置 page_size 大小, 这个大小也会通过宏定义配置到内核头信息里面.

要解答 page_shift, 我们需要知晓 kernel_flag_le. kernel_flag_le 从源码中来的, 我们就得到源码中去寻求答案.  
![](https://bbs.kanxue.com/upload/attach/202409/935696_4BPT676NBPJ4BVQ.webp)

其中定义为:

```
.... 3               | 2,       1          |     0
__HEAD_FLAG_PHYS_BASE|__HEAD_FLAG_PAGE_SIZE|__HEAD_FLAG_BE
```

有了对上上面的理解, 就能看懂下面的代码了.

```
switch ((flag & 0b0110) >> 1) {
    case 2: // 16k
        kinfo->page_shift = 14;
        break;
    case 3: // 64k
        kinfo->page_shift = 16;
        break;
    case 1: // 4k
    default:
        kinfo->page_shift = 12;
}
```

IDA 分析 B 跳转前后差异 (仙人指路)
----------------------

![](https://bbs.kanxue.com/upload/attach/202409/935696_A44JUX8H9BZB557.webp)

![](https://bbs.kanxue.com/upload/attach/202409/935696_JCVHYF8UM6CS255.webp)

通过 IDA 分析工具我们发现, 在内核启动之初, 会优先执行下面的代码.

```
; https://github.com/nzcv/blu7t/blob/master/kernel/arch/arm64/kernel/head.S
.section .entry.text, "ax"
.global setup_entry
.type entry, %function
setup_entry:
    // x0 = physical address to the FDT blob.
    // Preserve the arguments passed by the bootloader in x0 .. x3
    mov x9, sp
    adrp x11, stack
    add x11, x11, :lo12:stack
    add x11, x11, STACK_SIZE
    mov sp, x11
    stp x9, x10, [sp, -16]!
    b setup
```

也给我们提供了一个如何阅读内核代码提供了线索. 对于任何一个不清楚的细节, 可以调试观察寄存器的状态变化.

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#基础理论](forum-161-1-117.htm) [#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm)