> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-261585.htm)

> 本文从栈回溯的原理开始，到 arm ehabi 的回溯方式，再到 elf 文件中的 unwind 信息，最后实现一款 ida 里实时进行 arm 栈回溯的插件，覆盖了现代 arm 栈回溯的全部内容，希望能给大家带来帮助。

 

作品链接：[https://github.com/LeadroyaL/IDA_ARM_Unwind](https://github.com/LeadroyaL/IDA_ARM_Unwind)

 

pyelftools 的 commit 链接：[https://github.com/eliben/pyelftools/commit/ee0facee32ae5fc91709c93f9a57a9a7683a3315](https://github.com/eliben/pyelftools/commit/ee0facee32ae5fc91709c93f9a57a9a7683a3315)

[](#第一部分：背景，不耐烦可跳过)第一部分：背景，不耐烦可跳过
=================================

起因
==

故事要从几个月前的一个 arm crash 说起，把 crash 交给新来的小朋友看，他说 IDA 里显示的栈回溯和 logcat 里显示的栈回溯是不一致的，问我为什么。

 

我说一直是 logcat 里看的，是正确的；那么 ida 里只有一层栈回溯肯定是错的，但我却解释不来原因，于是有了本文和一系列研究。

平时讨论的函数调用栈结构
============

从小老师就教育我们，函数开头一般是这三句话，用于保存堆栈，开辟新的栈空间：

```
push ebp
mov ebp, esp
sub esp, 0x100

```

在这种设定下，栈回溯变得非常简单，ebp 就是栈帧，ebp 附近是上一个栈帧，再附近是返回地址。网上相关的文章一搜一大把，这里就不多讲了，找一张网图凑合一下。

 

![](https://bbs.pediy.com/upload/attach/202008/735539_CWXBUAZMXAW4W6U.jpg)

arm 的栈结构
========

我们随便找个 `/system/lib/libc.so`，再随便编译一个 so，随便找几个函数看一下，发现和 x86 的不大一样。

```
.text:000233C0                 PUSH.W          {R4-R8,LR}
.text:000233C4                 SUB             SP, SP, #8
.text:000233C6                 LDR             R4, [SP,#0x20+arg_8]
.text:000233C8                 MOV             R5, R1
...
.text:00023460                 MOV             R0, R4
.text:00023462                 ADD             SP, SP, #8
.text:00023464                 POP.W           {R4-R8,PC}

```

```
.text:00023A94                 PUSH.W          {R4-R9,LR}
.text:00023A98                 SUB             SP, SP, #4
.text:00023A9A                 MOV             R8, R1
.text:00023A9C                 MOV             R5, R0
...
.text:00023B12                 ADD             SP, SP, #4
.text:00023B14                 POP.W           {R4-R9,PC}

```

```
.text:0003477C                 PUSH            {R7,LR}
.text:0003477E                 MOV             R7, SP
.text:00034780                 SUB             SP, SP, #0x28
.text:00034782                 LDR             R2, =(__stack_chk_guard_ptr - 0x34788)
...
.text:000347CE                 MOVS            R0, #0
.text:000347D0                 ADD             SP, SP, #0x28
.text:000347D2                 POP             {R7,PC}

```

```
.text:000138C4                 PUSH            {R4,R5,R7,LR}
.text:000138C6                 ADD             R7, SP, #8
.text:000138C8                 SUB             SP, SP, #0x20
.text:000138CA                 LDR             R4, =(__stack_chk_guard_ptr - 0x138D0)
...
.text:000138F4                 ADDEQ           SP, SP, #0x20
.text:000138F6                 POPEQ           {R4,R5,R7,PC}

```

观察这几组汇编，前两段 sp 的内容并没有被保存到任意一个寄存器里，但它可以被正确栈回溯，暗示栈回溯信息不在这段汇编里；后两段，把 `sp` 放到 `r7` 里，把 `sp+8` 放到 `r7` 里，有点像栈帧的感觉，并且函数内也没有覆盖掉 `r7` 的内容，有点 `x86` 的感觉。

 

查阅资料，随着时代发展，arm 有两种 unwind 方式：

1.  一种是古老的，和 x86 类似的（目前没有找到样例，可能在某种编译选项下存在），使用专用的 `fp` 寄存器保存原先的 `sp`，`fp` 在函数内禁止被改写，thumb 模式下使用 `r7` 作为 `fp`，arm 模式下使用 `r11` 作为 `fp`。
2.  另一种是流行的， arm 特有的（目前绝大部分都使用这种方式），遵从 eabi 里的 ehabi 标准，即 exception handler abi，定义了一套专属的标准。简而言之就是对每个函数分配自己专用的字节码，解释执行，从而实现栈回溯。

使用`readelf -u`可以查看，字节码长这样：

```
0x9a8c <__cxa_end_cleanup_impl>: @0x14f28
  Compact model index: 1
  0x97      vsp = r7
  0x41      vsp = vsp - 8
  0x84 0x0b pop {r4, r5, r7, r14}
  0xb0      finish
  0xb0      finish

```

arm ehabi
=========

讲了这么多，终于引出本系列的重点：arm ehabi。

 

官方文档，复杂但权威：[https://developer.arm.com/documentation/ihi0038/b/](https://developer.arm.com/documentation/ihi0038/b/)  
看雪有篇不错的文档：[原创 andorid native 栈回溯原理分析与思考](https://bbs.pediy.com/thread-216447.htm)

*   使用 `readelf -u` 可以打印相关信息，也可以使用`pyelftools`里的 `readelf.py -au` 打印出来（而且这个功能是我写的）。
*   千万不要使用 `llvm-readelf -u`，因为它有 bug，只支持 `.o` 文件。

名词解释

<table><thead><tr><th>名词</th><th>解释</th></tr></thead><tbody><tr><td>stack unwind</td><td>意思就是栈回溯。</td></tr><tr><td>abi</td><td>（application binary interface）二进制应用接口，相当于标准和规范</td></tr><tr><td>arm eabi</td><td>arm 很多规范的合集，包括 AADWARF、AAELF 等，也包括 CLIBABI、CPPABI、【EHABI】</td></tr><tr><td>arm ehabi</td><td>arm 的 exception handler abi</td></tr><tr><td>exception handler</td><td>异常处理，既包括 crash 时的栈回溯，也包括 c++ 里的异常处理。</td></tr><tr><td>arm exception handler index table</td><td>指 <code>.ARM.exidx</code>，存放函数 offset、简单的 handler 的数据、复杂 handler 的索引。</td></tr><tr><td>arm exception handler table</td><td>指 <code>.ARM.extab</code>，存放复杂 handler 的数据。</td></tr></tbody></table>

 

和平时逆向相关的，有两部分内容，有个大致认知就行：

1.  数据存放。在 ELF 文件里肯定存放了信息，指导如何进行栈回溯，`.ARM.exidx` 和 `.ARM.extab` 就是做这件事情的；
2.  数据使用。每个函数 unwind 时需要解析 `exception handler table`，需要解释执行字节码，这个功能有时会由操作系统完成（例如 crash 的时候），有时会由应用程序自己完成（例如使用写代码主动进行栈回溯）。

总结
==

本文讲的是背景，没什么技术细节，第二篇讲文件格式。

第二部分：ELF 文件中的栈回溯信息。
===================

> 本系列文章共三篇。本文是第二篇，讲 ELF 文件如何存放和使用 arm ehabi。

ELF .ARM.exidx 和 .ARM.extab 的位置
===============================

Section 角度：很久以前，`readelf -S` 时候一直不理解这两个 section 是做什么的，占空间，放的不是汇编，IDA 打开，里面也是一团意义不明的 data，总觉得没什么用。

```
  ~ readelf -S libnative-lib.so
There are 25 section headers, starting at offset 0x1a130:
 
Section Headers：
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .note.android.ide NOTE            00000134 000134 000098 00   A  0   0  4
  [ 2] .note.gnu.build-i NOTE            000001cc 0001cc 000024 00   A  0   0  4
  .......
  [14] .ARM.exidx        ARM_EXIDX       00013eb8 013eb8 000e80 08  AL 13   0  4
  [15] .ARM.extab        PROGBITS        00014d38 014d38 001014 00   A  0   0  4

```

Segment 角度：`readelf -l` 时候，`EXIDX` 就是 `.ARM.exidx`，感觉还是有点用的，但仍然意义不明；不能直接确认 `.ARM.extab` 的位置。

```
readelf -l libnative-lib.so
 
Elf file type is DYN (Shared object file)
Entry point 0x0
There are 8 program headers, starting at offset 52
 
Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00000034 0x00000034 0x00100 0x00100 R   0x4
  LOAD           0x000000 0x00000000 0x00000000 0x1802e 0x1802e R E 0x1000
  LOAD           0x018570 0x00019570 0x00019570 0x01aa0 0x01cb9 RW  0x1000
  DYNAMIC        0x019c94 0x0001ac94 0x0001ac94 0x00110 0x00110 RW  0x4
  NOTE           0x000134 0x00000134 0x00000134 0x000bc 0x000bc R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
  EXIDX          0x013eb8 0x00013eb8 0x00013eb8 0x00e80 0x00e80 R   0x4
  GNU_RELRO      0x018570 0x00019570 0x00019570 0x01a90 0x01a90 RW  0x4

```

好了不废话了，本文只针对 `shared_library` 和 `executable` 的 ehabi 解析，不支持 `relocatable`；因为 `relocatable` 拥有大量的 `.ARM.exidx` section 和重定位，有点复杂。

 

本文参考：

*   官方文档，发现看不懂的就去读文档：[https://developer.arm.com/documentation/ihi0038/b/](https://developer.arm.com/documentation/ihi0038/b/)
*   llvm-readelf 的实现：[https://github.com/llvm/llvm-project/blob/master/llvm/tools/llvm-readobj/ARMEHABIPrinter.h](https://github.com/llvm/llvm-project/blob/master/llvm/tools/llvm-readobj/ARMEHABIPrinter.h)
*   binutils-readelf 的实现 [https://github.com/bminor/binutils-gdb/blob/master/binutils/readelf.c](https://github.com/bminor/binutils-gdb/blob/master/binutils/readelf.c)
*   看雪有篇不错的文档：[原创 andorid native 栈回溯原理分析与思考](https://bbs.pediy.com/thread-216447.htm)
*   网上挺火的外国人的文档 《Stack frame unwinding on ARM》 （Ken Werner）（可以在《andorid native 栈回溯原理分析与思考》的附件里下载到）

.ARM.exidx 结构
=============

这个 section 连续存放 Entry，视为一个 Entry 数组。先要处理大小端问题，处理好后，每个 Entry 由两个 uint16 组成，相当于如下 struct：

```
struct EHEntry {
  uint32_t Offset;
  uint32_t World1;
};

```

`EHEntry.Offset` 意义是函数起始偏移。最高 bit 一定是 0，结合当前偏移（当前 pc ）进行使用 `prel31` 解码，得到 uint64_t。

 

格式为：

```
| 31----24 | 23----16 | 15-----8 | 7------0 |
| 0XXXXXXX | XXXXXXXX | XXXXXXXX | XXXXXXXX |

```

`EHEntry.Word1` 有三种情况：

*   `EHEntry.Word1 == 1` ，表示 CannotUnwind
    
    ```
    | 00000000 | 00000000 | 00000000 | 00000001 |
    
    ```
    
*   最高 bit 为 1，则`[31:24]`必须为 `0b10000000`（其实是 personality 为 0，属于 inline compact model），余下 X、Y、Z， 3 个 byte 表示字节码。
    
    ```
    | 31----24 | 23----16 | 15-----8 | 7------0 |
    | 10000000 | XXXXXXXX | YYYYYYYY | ZZZZZZZZ |
    
    ```
    
*   最高 bit 为 0，则整个 Word1 使用 `prel31` 解码，得到 uint64_t，指向真正的数据。必然会落在 `.ARM.extab` 里。
    
    ```
    | 31----24 | 23----16 | 15-----8 | 7------0 |
    | 0XXXXXXX | XXXXXXXX | XXXXXXXX | XXXXXXXX |
    
    ```
    

prel31 解码
=========

这个东西就是个计算方式，根据当前的绝对偏移 (uint32)，与当前的内容(uint32) 进行运算，求出绝对偏移(uint64)。

 

对于 ELF 文件，我们可以假想它基址为 0，从而实现解析；

 

对于内存中的 ELF 片段，可以通过这个计算，根据当前位置寻找到附近的另一个位置，从而避免重定位；

 

下图代码中，Address 表示当前内容，Place 表示绝对偏移。

```
static uint64_t PREL31(uint32_t Address, uint32_t Place) {
    uint64_t Location = Address & 0x7fffffff;
    if (Location & 0x04000000)
        Location |= (uint64_t) ~0x7fffffff;
    return Location + Place;
}

```

.ARM.extab 结构
=============

`.ARM.extab` 作为 `.ARM.exidx` 的附属存在，存放数据，但无法直接找到每段数据的入口。入口需要借助上文 `Entry.Word1`，当 `Entry.Word1` 的最高 bit 为 0 时，`prel31`解码后一定会指向 `.ARM.extab` 的内容，这就是入口。

 

名词解释：`personality` ——特性，可能没有中文概念。

 

先读出第一个 `uint32_t`，进行初步解析，再根据情况进行进一步解析，有以下的情况：

*   最高 bit 为 0，表示 `generic personality`，使用 `prel31` 解码，使用指向的函数进行 unwind。
    
    ```
    | 31----24 | 23----16 | 15-----8 | 7------0 |
    | 0XXXXXXX | XXXXXXXX | XXXXXXXX | XXXXXXXX |
    
    ```
    
*   最高 bit 为 1，表示 `arm compact personality`，`[31:28]`必须为 `0b1000`，`[27:24]` 有且仅有有 0、1、2 三种情况。
    
    *   0: inline compact model，X、Y、Z， 3 个 byte 表示字节码。
        
        ```
        | 31----24 | 23----16 | 15-----8 | 7------0 |
        | 10000000 | XXXXXXXX | YYYYYYYY | ZZZZZZZZ |
        
        ```
        
    *   1 或者 2: `[23:16]` 表示 more_word(uint_8)，表示剩余字节码个数，后面的都是字节码
        
        ```
        | 31----24 | 23----16 | 15-----8 | 7------0 |
        | 10000001 | MOREWORD | ........ | ........ |
        | 10000010 | MOREWORD | ........ | ........ |
        
        ```
        

字节码的反汇编
=======

根据上文，我们可以得到每一处 Entry 及其 unwind 方式。我们关心的是使用字节码进行 unwind （即 personality 为 0、1、2）的情况，经过解析可以得到 `uint8_t[N]` 的字节码 ，解析方式在 "Table 4, ARM-defined frame-unwinding instructions" 文件章节。

 

参考 `llvm-readelf` 的实现，它的可读性最好，代码在 [https://github.com/llvm/llvm-project/blob/master/llvm/tools/llvm-readobj/ARMEHABIPrinter.h](https://github.com/llvm/llvm-project/blob/master/llvm/tools/llvm-readobj/ARMEHABIPrinter.h) ，其中有大量`OpcodeDecoder::Decode_XXXXX` 函数可以抄。

 

纯体力活，没什么好说的，官方表格如下：

<table><thead><tr><th>Instruction</th><th>Explanation</th></tr></thead><tbody><tr><td>00xxxxxx</td><td>vsp = vsp + (xxxxxx &lt;&lt; 2) + 4. Covers range 0x04-0x100 inclusive</td></tr><tr><td>01xxxxxx</td><td>vsp = vsp – (xxxxxx &lt;&lt; 2) - 4. Covers range 0x04-0x100 inclusive</td></tr><tr><td>10000000 00000000</td><td>Refuse to unwind (for example, out of a cleanup) (see remark a)</td></tr><tr><td>1000iiii iiiiiiii (i not a ll 0)</td><td>Pop up to 12 integer registers under masks {r15-r12}, {r11-r4} (see remark b)</td></tr><tr><td>1001nnnn (nnnn != 13,15)</td><td>Set vsp = <code>r[nnnn]</code></td></tr><tr><td>10011101</td><td>Reserved as prefix for ARM register to register moves</td></tr><tr><td>10011111</td><td>Reserved as prefix for Intel Wireless MMX register to register moves</td></tr><tr><td>10100nnn</td><td>Pop r4-<code>r[4+nnn]</code></td></tr><tr><td>10101nnn</td><td>Pop r4-<code>r[4+nnn]</code>, r14</td></tr><tr><td>10110000</td><td>Finish (see remark c)</td></tr><tr><td>10110001 00000000</td><td>Spare (see remark f)</td></tr><tr><td>10110001 0000iiii (i not all 0)</td><td>Pop integer registers under mask {r3, r2, r1, r0}</td></tr><tr><td>10110001 xxxxyyyy</td><td>Spare (xxxx != 0000)</td></tr><tr><td>10110010 uleb128</td><td>vsp = vsp + 0x204+ (uleb128 &lt;&lt; 2) (for vsp increments of 0x104-0x200, use 00xxxxxx twice)</td></tr><tr><td>10110011 sssscccc</td><td>Pop VFP double-precision registers <code>D[ssss]-D[ssss+cccc]</code> saved (as if) by FSTMFDX (see remark d)</td></tr><tr><td>101101nn</td><td>Spare (was Pop FPA)</td></tr><tr><td>10111nnn</td><td>Pop VFP double-precision registers <code>D[8]-D[8+nnn]</code> saved (as if) by FSTMFDX (seeremark d)</td></tr><tr><td>11000nnn (nnn != 6,7)</td><td>Intel Wireless MMX pop <code>wR[10]-wR[10+nnn]</code></td></tr><tr><td>11000110 sssscccc</td><td>Intel Wireless MMX pop <code>wR[ssss]-wR[ssss+cccc]</code> (see remark e)</td></tr><tr><td>11000111 00000000</td><td>Spare</td></tr><tr><td>11000111 0000iiii</td><td>Intel Wireless MMX pop wCGR registers under mask {wCGR3,2,1,0}</td></tr><tr><td>11000111 xxxxyyyy</td><td>Spare (xxxx != 0000)</td></tr><tr><td>11001000 sssscccc</td><td>Pop VFP double precision registers <code>D[16+ssss]-D[16+ssss+cccc]</code> saved (as if) by VPUSH (see remarks d,e)</td></tr><tr><td>11001001 sssscccc</td><td>Pop VFP double precision registers <code>D[ssss]-D[ssss+cccc]</code> saved (as if) by VPUSH (see remark d)</td></tr><tr><td>11001yyy</td><td>Spare (yyy != 000, 001)</td></tr><tr><td>11010nnn</td><td>Pop VFP double-precision registers <code>D[8]-D[8+nnn]</code> saved (as if) by VPUSH (seeremark d)</td></tr><tr><td>11xxxyyy</td><td>Spare (xxx != 000, 001, 010)</td></tr></tbody></table>

实战：用 python 写一个 ehabi 的 parser
==============================

很遗憾，pyelftools 并未实现解析 arm ehabi 的功能，

 

我为什么要写解析的功能？一方面因为 pyelftools 平时经常用，想为它做一些贡献；另一方面，我计划写一个 ida-arm-unwind 的插件，缺乏一个 python 库帮我完成解析，在 pyelftools 上补充功能是最合适的。

 

pull reqeust：[https://github.com/eliben/pyelftools/pull/328](https://github.com/eliben/pyelftools/pull/328)

 

merge commit：[https://github.com/eliben/pyelftools/commit/ee0facee32ae5fc91709c93f9a57a9a7683a3315](https://github.com/eliben/pyelftools/commit/ee0facee32ae5fc91709c93f9a57a9a7683a3315)

 

花了不少时间，写了将近 1000 行代码，实现得也比较优雅，大概有如下的功能：

*   加了 `has_ehabi_infos` 和 `get_ehabi_infos` 两个 API，返回 `List[EHABIInfo]`。
*   添加 `class EHABIInfo`，提供 `num_entry` 和 `get_entry(i)` 两个 API，返回 `EHABIEntry`。
*   添加 `class EHABIEntry` 及其子类，描述每个 unwind 条目，描述函数偏移和字节码，也可以反汇编

看一下效果：

```
  pyelftools git:(master)  scripts/readelf.py -au test/testfiles_for_unittests/arm_exidx_test.so | head -n 20
 
Unwind section '.ARM.exidx' at offset 0x639d8 contains 1933 entries
 
Entry 0:
    Function offset 0x34610: @0x69544
    Compact model index: 1
    0x97 ; vsp = r7
    0x41 ; vsp = vsp - 8
    0x84 0x0d ; pop {r4, r6, r7, lr}
    0xb0 ; finish
    0xb0 ; finish
 
Entry 1:
    Function offset 0x34640: @0x6bef4
    Compact model index: 1
    0x97 ; vsp = r7
    0x41 ; vsp = vsp - 8
    0x84 0x0b ; pop {r4, r5, r7, lr}
    0xb0 ; finish
    0xb0 ; finish

```

总结
==

本文讲了 ELF 里 arm ehabi 的存放和使用。

[](#第三部分，ida-arm-unwind-plugin的开发)第三部分，ida-arm-unwind-plugin 的开发
================================================================

> 本系列文章共三篇。本文是第三篇，使用已有的知识，实现 arm stack unwind，给本系列完美地画上句号。

ida arm unwind plugin
=====================

IDA 里有一个不常用的功能，叫打印栈回溯，使用的是常见的 ebp/esp 栈帧技术，没有对 ARM 进行适配，导致调试安卓 so 时完完全全是错的。本文编写一个 ida 插件，正确展示实时的 arm 栈回溯。

### https://github.com/leadroyal/ida_arm_unwind ，欢迎 star。"href="# 代码在：[https://github.com/leadroyal/ida_arm_unwind](https://github.com/leadroyal/ida_arm_unwind) ，欢迎 star。"> 代码在：[https://github.com/LeadroyaL/IDA_ARM_Unwind](https://github.com/LeadroyaL/IDA_ARM_Unwind) ，欢迎 star。

设计思路：

1.  总目标，对当前断点进行栈回溯，得到 pc 序列，进一步可以得到 crash-log 一样的栈回溯展示
2.  检查运行环境，需要是 ARM 架构，需要处于调试中的状态
3.  将当前 pc 加入序列
4.  初始化`VRSStatus`状态，将各个寄存器的值设置正确，为 unwind 做准备
5.  递归 unwind
    1.  根据当前 pc 找到当前 ELF 头部的大概位置，解析头部的一些字节，使用 pyelftools 得到第一个 PT_LOAD
    2.  `.ARM.exidx` 和 `.ARM.extab` 一定在第一个 `PT_LAOD` 里，使用 pyelftools 解析它们的数据
    3.  根据当前 pc、`List[EHABIEntry]`，找到对应的 EHABIEntry
    4.  解释执行对应的字节码，得到最终状态
    5.  最终状态的 LR 就是返回地址，判断 ARM 还是 THUMB，再决定是 -2 还是 -4，修正为上一条指令的地址
    6.  将计算好的 pc 加入序列
6.  使用 pc 序列，寻找对应的 module、funcion，计算相对偏移，构造为 `List[Frame]`  
    7: 绑定快捷键，画 GUI，抄的 `https://github.com/ChiChou/IDA-ObjCExplorer` 的代码

效果图，个人感觉还是非常好用的哈：

 

![](https://bbs.pediy.com/upload/attach/202008/735539_DEMZVCFYCN824SD.png)  
![](https://bbs.pediy.com/upload/attach/202008/735539_JPNNHUSWQ7U3B8G.png)

 

具体用途看仓库里的 readme 哈。

总结
==

三篇文章，虽然是按照开发的时间顺序写的，但发生顺序其实是反的，这个流程拖得挺长：

1.  先发现 IDA 失效，于是准备写个插件；
2.  写插件，研究栈回溯，发现 ARM 的栈回溯跟人不一样 （参考 [原创 andorid native 栈回溯原理分析与思考](https://bbs.pediy.com/thread-216447.htm) ）
3.  pyelftools 不提供 arm ehabi 的解析，于是自己实现数据解析（参考 [https://github.com/llvm/llvm-project/blob/master/llvm/tools/llvm-readobj/ARMEHABIPrinter.h](https://github.com/llvm/llvm-project/blob/master/llvm/tools/llvm-readobj/ARMEHABIPrinter.h) ）
4.  libunwind 接入成本太高，于是自己用 python 写字节码的解释执行和 unwind（参考 [https://github.com/llvm/llvm-project/blob/master/libunwind/src/Unwind-EHABI.cpp](https://github.com/llvm/llvm-project/blob/master/libunwind/src/Unwind-EHABI.cpp) ）

把大量的规范从 c 移植为 python，整个下来花了我大量时间，但对 unwind 本身有了非常深刻的理解，希望 `elftools.ehabi` 和 `ida-arm-unwind-plugin` 这两个轮子能为行业做贡献吧。

 

作品链接：[https://github.com/LeadroyaL/IDA_ARM_Unwind](https://github.com/LeadroyaL/IDA_ARM_Unwind)

 

pyelftools 的 commit 链接：[https://github.com/eliben/pyelftools/commit/ee0facee32ae5fc91709c93f9a57a9a7683a3315](https://github.com/eliben/pyelftools/commit/ee0facee32ae5fc91709c93f9a57a9a7683a3315)

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

最后于 2020-8-23 09:31 被 LeadroyaL 编辑 ，原因： 错别字