> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-261349.htm)

本文介绍了 ELF 的基本结构和内存加载的原理，并用具体案例来分析如何通过 ELF 特性实现 HIDS bypass、加固 / 脱壳以及辅助进行 binary fuzzing。

 

目录

*   [前言](#前言)
*   ELF 101
*            [大局观](#大局观)
*            ELF Header
*            Section Header
*            Program Header
*            [小结](#小结)
*   [程序加载](#程序加载)
*            [内核空间](#内核空间)
*            [用户空间](#用户空间)
*   [实际案例](#实际案例)
*            Interpreter Hack
*            [加固 / 脱壳](#加固/脱壳)
*            Binary Fuzzing
*   [总结](#总结)
*   [参考链接](#参考链接)

前言
==

作为一个安全研究人员，ELF 可以说是一个必须了解的格式，因为这关系到程序的编译、链接、封装、加载、动态执行等方方面面。有人就说了，这不就是一种文件格式而已嘛，最多按照 SPEC 实现一遍也就会了，难道还能复杂过 FLV/MP4？曾经我也是这么认为的，直到我在日常工作时遇到了下面的错误:

```
$ r2 a.out
Segmentation fault

```

作为一个开源爱好者，我的 [radare2](https://github.com/radareorg/radare2) 经常是用 master 分支编译的，经过在 github 中搜索，发现 radare 对于 ELF 的处理还有不少同类的问题，比如 [issue#17300](https://github.com/radareorg/radare2/issues/17300) 以及 [issue#17379](https://github.com/radareorg/radare2/issues/17379)，这还只是近一个月内的两个 open issue，历史问题更是数不胜数。

 

总不能说 radare 的开发者不了解 ELF 吧？事实上他们都是软件开发和逆向工程界的专家。不止 radare，其实 IDA 和其他反编译工具也曾出现过各类 [ELF 相关的 bug](https://ioactive.com/wp-content/uploads/2018/05/IOActive_ELF_Parsing_with_Melkor-1.pdf)。

 

说了那么多，只是为了引出一个观点: ELF 既简单也复杂，值得我们去深入了解。网上已经有了很多介绍 ELF 的文章，因此本文不会花太多篇幅在 SPEC 的复制粘贴上，而是结合实际案例和应用场景去进行说明。

ELF 101
=======

ELF 的全称是 **Executable and Linking Format**，这个名字相当关键，包含了 ELF 所需要支持的两个功能——执行和链接。不管是 ELF，还是 Windows 的`PE`，抑或是 MacOS 的`Mach-O`，其根本目的都是为了能让处理器正确执行我们所编写的代码。

大局观
---

在上古时期，给 CPU 运行代码也不用那么复杂，什么代码段数据段，直接把编译好的机器码一把梭烧到中断内存空间，PC 直接跳过来就执行了。但随着时代变化，大家总不能一直写汇编了，即便编译器很给力，也会涉及到多人协作、资源复用等问题。这时候就需要一种可拓展 (Portable) 的文件标准，一方面让开发者 (编译器 / 链接器) 能够高效协作，另一方面也需要系统能够正确、安全地将文件加载到对应内存中去执行，这就是 ELF 的使命。

 

![](https://bbs.pediy.com/upload/attach/202008/844554_FV5D8AW37MUCV33.png)

 

从大局上看，ELF 文件主要分为 3 个部分:

*   ELF Header
*   Section Header Table
*   Program Header Table

其中，**ELF Header** 是文件头，包含了固定长度的文件信息；**Section Header Table** 则包含了**链接时**所需要用到的信息；**Program Header Table** 中包含了**运行时**加载程序所需要的信息，后面会进行分别介绍。

ELF Header
----------

ELF 头部的定义在 [elf/elf.h](https://github.com/bminor/glibc/blob/glibc-2.27/elf/elf.h) 中 (以 glibc-2.27 为例)，使用 POD 结构体表示，内存可使用结构体的字段一一映射，头部表示如下:

```
#define EI_NIDENT (16)
 
typedef struct
{
  unsigned char    e_ident[EI_NIDENT];    /* Magic number and other info */
  Elf32_Half    e_type;            /* Object file type */
  Elf32_Half    e_machine;        /* Architecture */
  Elf32_Word    e_version;        /* Object file version */
  Elf32_Addr    e_entry;        /* Entry point virtual address */
  Elf32_Off    e_phoff;        /* Program header table file offset */
  Elf32_Off    e_shoff;        /* Section header table file offset */
  Elf32_Word    e_flags;        /* Processor-specific flags */
  Elf32_Half    e_ehsize;        /* ELF header size in bytes */
  Elf32_Half    e_phentsize;        /* Program header table entry size */
  Elf32_Half    e_phnum;        /* Program header table entry count */
  Elf32_Half    e_shentsize;        /* Section header table entry size */
  Elf32_Half    e_shnum;        /* Section header table entry count */
  Elf32_Half    e_shstrndx;        /* Section header string table index */
} Elf32_Ehdr;

```

注释都很清楚了，挑一些比较重要的来说。其中`e_type`表示 ELF 文件的类型，有以下几种:

*   ET_NONE: 未知类型
*   ET_REL: 可重定向类型 (relocatable)，通常是我们编译的`*.o`文件
*   ET_EXEC: 可执行类型 (executable)，静态编译的可执行文件
*   ET_DYN: 共享对象 (shared object)，动态编译的可执行文件或者动态库`*.so`
*   ET_CORE: coredump 文件

`e_entry`是程序的入口虚拟地址，注意不是 main 函数的地址，而是`.text`段的首地址`_start`。当然这也要求程序本身非 PIE(`-no-pie`) 编译的且 ASLR 关闭的情况下，对于非`ET_EXEC`类型通常并不是实际的虚拟地址值。

 

其他的字段大多数是指定 Section Header(`e_sh`) 和 Program Header(`e_ph`) 的信息。Section/Program Header Table 本身可以看做是数组结构，ELF 头中的信息指定对应 Table 数组的位置、长度、元素大小信息。最后一个 _e_shstrndx_ 表示的是 section table 中的第 _e_shstrndx_ 项元素，保存了所有 section table 名称的字符串信息。

Section Header
--------------

上节说了 section header table 是一个数组结构，这个数组的位置在 _e_shoff_ 处，共有 _e_shnum_ 个元素 (即 section)，每个元素的大小为 _e_shentsize_ 字节。每个元素的结构如下:

```
typedef struct
{
  Elf32_Word    sh_name;        /* Section name (string tbl index) */
  Elf32_Word    sh_type;        /* Section type */
  Elf32_Word    sh_flags;        /* Section flags */
  Elf32_Addr    sh_addr;        /* Section virtual addr at execution */
  Elf32_Off    sh_offset;        /* Section file offset */
  Elf32_Word    sh_size;        /* Section size in bytes */
  Elf32_Word    sh_link;        /* Link to another section */
  Elf32_Word    sh_info;        /* Additional section information */
  Elf32_Word    sh_addralign;        /* Section alignment */
  Elf32_Word    sh_entsize;        /* Entry size if section holds table */
} Elf32_Shdr;

```

其中`sh_name`是该 section 的名称，用一个 word 表示其在字符表中的偏移，字符串表 (.shstrtab) 就是上面说到的第 _e_shstrndx_ 个元素。ELF 文件中经常使用这种偏移表示方式，可以方便组织不同区段之间的引用。

 

`sh_type`表示本 section 的类型，SPEC 中定义了几十个类型，列举其中一些如下:

*   SHT_NULL: 表示该 section 无效，通常第 0 个 section 为该类型
*   SHT_PROGBITS: 表示该 section 包含由程序决定的内容，如`.text`、`.data`、`.plt`、`.got`
*   SHT_SYMTAB/SHT_DYNSYM: 表示该 section 中包含符号表，如`.symtab`、`.dynsym`
*   SHT_DYNAMIC: 表示该 section 中包含动态链接阶段所需要的信息
*   SHT_STRTAB: 表示该 section 中包含字符串信息，如`.strtab`、`.shstrtab`
*   SHT_REL/SHT_RELA: 包含重定向项信息
*   ...

虽然每个 section header 的大小一样 (e_shentsize 字节)，但不同类型的 section 有不同的内容，内容部分由这几个字段表示:

*   sh_offset: 内容起始地址相对于文件开头的偏移
*   sh_size: 内容的大小
*   sh_entsize: 有的内容是也是一个数组，这个字段就表示数组的元素大小

与运行时信息相关的字段为:

*   sh_addr: 如果该 section 需要在运行时加载到虚拟内存中，该字段就是对应 section 内容 (第一个字节) 的虚拟地址
*   sh_addralign: 内容地址的对齐，如果有的话需要满足`sh_addr % sh_addralign = 0`
*   sh_flags: 表示所映射内容的权限，可根据`SHF_WRITE/ALLOC/EXECINSTR`进行组合

另外两个字段`sh_link`和`sh_info`的含义根据 section 类型的不同而不同，如下表所示:

 

![](https://bbs.pediy.com/upload/attach/202008/844554_XHZJRQ8CAFEQDJK.png)

 

至于不同类型的 section，有的是保存符号表，有的是保存字符串，这也是 ELF 表现出拓展性和复杂性的地方，因此需要在遇到具体问题的时候查看文档去进行具体分析。

Program Header
--------------

program header table 用来保存程序加载到内存中所需要的信息，使用段 (segment) 来表示。与 section header table 类似，同样是数组结构。数组的位置在偏移 _e_phoff_ 处，每个元素 (segment header) 的大小为 _e_phentsize_，共有 _e_phnum_ 个元素。单个 segment header 的结构如下:

```
typedef struct
{
  Elf32_Word    p_type;            /* Segment type */
  Elf32_Off    p_offset;          /* Segment file offset */
  Elf32_Addr    p_vaddr;        /* Segment virtual address */
  Elf32_Addr    p_paddr;        /* Segment physical address */
  Elf32_Word    p_filesz;        /* Segment size in file */
  Elf32_Word    p_memsz;        /* Segment size in memory */
  Elf32_Word    p_flags;        /* Segment flags */
  Elf32_Word    p_align;        /* Segment alignment */
} Elf32_Phdr;

```

既然 program header 的作用是提供用于初始化程序进程的段信息，那么下面这些字段就是很直观的:

*   p_offset: 该 segment 的数据在文件中的偏移地址 (相对文件头)
*   p_vaddr: segment 数据应该加载到进程的虚拟地址
*   p_paddr: segment 数据应该加载到进程的物理地址 (如果对应系统使用的是物理地址)
*   p_filesz: 该 segment 数据在文件中的大小
*   p_memsz: 该 segment 数据在进程内存中的大小。注意需要满足`p_memsz>=p_filesz`，多出的部分初始化为 0，通常作为`.bss`段内容
*   p_flags: 进程中该 segment 的权限 (R/W/X)
*   p_align: 该 segment 数据的对齐，2 的整数次幂。即要求`p_offset % p_align = p_vaddr`。

剩下的`p_type`字段，表示该 program segment 的类型，主要有以下几种:

*   PT_NULL: 表示该段未使用
*   PT_LOAD: Loadable Segment，将文件中的 segment 内容映射到进程内存中对应的地址上。值得一提的是 SPEC 中说在 program header 中的多个 PT_LOAD 地址是按照虚拟地址递增排序的。
*   PT_DYNAMIC: 动态链接中用到的段，通常是 RW 映射，因为需要由`interpreter`(ld.so) 修复对应的的入口
*   PT_INTERP: 包含 interpreter 的路径，见下文
*   PT_HDR: 表示 program header table 本身。如果有这个 segment 的话，必须要在所有可加载的 segment 之前，并且在文件中不能出现超过**一次**。
*   ...

在不同的操作系统中还可能有一些拓展的类型，比如`PT_GNU_STACK`、`PT_GNU_RELRO`等，不一而足。

小结
--

至此，ELF 文件中相关的字段已经介绍完毕，主要组成也就是 Section Header Table 和 Program Header Table 两部分，整体框架相当简洁。而 ELF 中体现拓展性的地方则是在 Section 和 Segment 的类型上 (s_type 和 p_type)，这两个字段的类型都是`ElfN_Word`，在 32 位系统下大小为 4 字节，也就是说最多可以支持高达`2^32 - 1`种不同的类型！除了上面介绍的常见类型，不同操作系统或者厂商还能定义自己的类型去实现更多复杂的功能。

程序加载
====

在新版的 ELF 标准文档中，将 ELF 的介绍分成了三部分，第一部分介绍 ELF 文件本身的结构，第二部分是处理器相关的内容，第三部分是操作系统相关的内容。ELF 的加载实际上是与操作系统相关的，不过大部分情况下我们都是在 GNU/Linux 环境中运行，因此就以此为例介绍程序的加载流程。

 

Linux 中分为用户态和内核态，执行 ELF 文件在用户态的表现就是执行 **execve** 系统调用，随后陷入内核进行处理。

内核空间
----

内核空间对 execve 的处理其实可以单独用一篇文章去介绍，其中涉及到进程的创建、文件资源的处理以及进程权限的设置等等。我们这里主要关注其中 ELF 处理相关的部分即可，实际上内核可以识别多种类型的可执行文件，ELF 的处理代码主要在 [fs/binfmt_elf.c](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c) 中的`load_elf_binary`函数中。

 

对于 ELF 而言，Linux 内核所关心的只有 Program Header 部分，甚至大部分情况下只关心三种类型的 Header，即`PT_LOAD`、`PT_INTERP`和`PT_GNU_STACK`。以 3.18 内核为例，load_elf_binary 主要有下面操作:

1.  对 ELF 文件做一些基本检查，保证`e_phentsize = sizeof(struct elf_phdr)`并且`e_phnum`的个数在一定范围内；
2.  循环查看每一项 program header，如果有 PT_INTERP 则使用`open_exec`加载进来，并替换原程序的`bprm->buf`;
3.  根据`PT_GNU_STACK`段中的 flag 设置栈是否可执行；
4.  使用`flush_old_exec`来更新当前可执行文件的所有引用；
5.  使用`setup_new_exec`设置新的可执行文件在内核中的状态；
6.  `setup_arg_pages`在栈上设置程序调用参数的内存页；
7.  循环每一项`PT_LOAD`类型的段，`elf_map`映射到对应内存页中，初始化 BSS；
8.  如果存在 interpreter，将入口 (elf_entry) 设置为 interpreter 的函数入口，否则设置为原 ELF 的入口地址；
9.  `install_exec_creds(bprm)`设置进程权限等信息；
10.  `create_elf_tables`添加需要的信息到程序的栈中，比如 **ELF auxiliary vector**；
11.  设置`current->mm`对应的字段；

从内核的处理流程上来看，如果是静态链接的程序，实际上内核返回用户空间执行的就是该程序的入口地址代码；如果是动态链接的程序，内核返回用户空间执行的则是 interpreter 的代码，并由其加载实际的 ELF 程序去执行。

 

为什么要这么做呢？如果把动态链接相关的代码也放到内核中，就会导致内核执行功能过多，内核的理念一直是能不在内核中执行的就不在内核中处理，以避免出现问题时难以更新而且影响系统整体的稳定性。事实上内核中对 ELF 文件结构的支持是相当有限的，只能读取并理解部分的字段。

用户空间
----

内核返回用户空间后，对于静态链接的程序是直接执行，没什么好说的。而对于动态链接的程序，实际是执行 interpreter 的代码。ELF 的 interpreter 作为一个段，自然是编译链接的时候加进去的，因此和编译使用的工具链有关。对于 Linux 系统而言，使用的一般是 GCC 工具链，而 interpreter 的实现，代码就在 glibc 的 [elf/rtld.c](https://github.com/bminor/glibc/blob/glibc-2.27/elf/rtld.c) 中。

 

interpreter 又称为 dynamic linker，以 glibc2.27 为例，它的大致功能如下:

*   将实际要执行的 ELF 程序中的内存段加载到当前进程空间中；
*   将动态库的内存段加载到当前进程空间中；
*   对 ELF 程序和动态库进行重定向操作 (relocation)；
*   调用动态库的初始化函数 (如_.preinit_array, .init, .init_array_)；
*   将控制流传递给目标 ELF 程序，让其看起来自己是直接启动的；

其中参与动态加载和重定向所需要的重要部分就是 Program Header Table 中`PT_DYNAMIC`类型的 Segment。前面我们提到在 Section Header 中也有一部分参与动态链接的 section，即`.dynamic`。我在自己解析动态链接文件的时候发现，实际上 `.dynamic` section 中的数据，和`PT_DYNAMIC`中的数据指向的是文件中的**同一个地方**，即这两个 entry 的 s_offset 和 p_offset 是相同。每个元素的类型如下:

```
typedef struct
{
  Elf32_Sword    d_tag;            /* Dynamic entry type */
  union
    {
      Elf32_Word d_val;            /* Integer value */
      Elf32_Addr d_ptr;            /* Address value */
    } d_un;
} Elf32_Dyn;

```

d_tag 表示实际类型，并且 d_un 和 d_tag 相关，可能说是很有拓展性了:) 同样的，标准中定义了几十个 d_tag 类型，比较常用的几个如下:

*   DT_NULL: 表示_DYNAMIC 的结尾
*   DT_NEEDED: d_val 保存了一个到字符串表头的偏移，指定的字符串表示该 ELF 所依赖的动态库名称
*   DT_STRTAB: d_ptr 指定了地址保存了符号、动态库名称以及其他用到的字符串
*   DT_STRSZ: 字符串表的大小
*   DT_SYMTAB: 指定地址保存了符号表
*   DT_INIT/DT_FINI: 指定初始化函数和结束函数的地址
*   DT_RPATH: 指定动态库搜索目录
*   DT_SONAME: Shared Object Name，指定当前动态库的名字 ([logical name](https://en.wikipedia.org/wiki/Soname))
*   ...

其中有部分的类型可以和 Section 中的`SHT_xxx`类型进行类比，完整的列表可以参考 ELF 标准中的 _Book III: Operating System Specific_ 一节。

 

在 interpreter 根据`DT_NEEDED`加载完所有需要的动态库后，就实现了完整进程虚拟内存映像的布局。在寻找某个动态符号时，interpreter 会使用**广度优先**的方式去进行搜索，即先在当前 ELF 符号表中找，然后再从当前 ELF 的`DT_NEEDED`动态库中找，再然后从动态库中的`DT_NEEDED`里查找。

 

因为动态库本身是位置无关的 (PIE)，支持被加载到内存中的随机位置，因此为了程序中用到的符号可以被正确引用，需要对其进行重定向操作，指向对应符号的真实地址。这部分我在之前写的关于 [GOT,PLT 和动态链接](https://evilpan.com/2018/04/09/about-got-plt/)的文章中已经详细介绍过了，因此不再赘述，感兴趣的朋友可以参考该文章。

实际案例
====

有人也许会问，我看你 bibi 了这么多，有什么实际意义吗？呵呵，本节就来分享几个我认为比较有用的应用场景。

Interpreter Hack
----------------

在渗透测试中，红队小伙伴们经常能拿到目标的后台 shell 权限，但是遇到一些部署了 HIDS 的大企业，很可能在执行恶意程序的时候被拦截，或者甚至触发监测异常直接被蓝队拔网线。这里不考虑具体的 HIDS 产品，假设现在面对两种场景:

1.  目标环境的可写磁盘直接 mount 为 **noexec**，无法执行代码
2.  目标环境内核监控任何非系统路径的程序的执行都会直接告警

不管什么样的环境，我相信老红队都有办法去绕过，这里我们运用上面学到的 ELF 知识，其实有一种更为简单的解法，即利用 interpreter。示例如下:

```
$ cat hello.c
#include int main() {
    return puts("hello!");
}
$ gcc hello.c -o hello
$ ./hello
hello!
 
$ chmod -x hello
$ ./hello
bash: ./hello: Permission denied
$ /lib64/ld-linux-x86-64.so.2 ./hello
hello!
$ strace /lib64/ld-linux-x86-64.so.2 ./hello 2>&1 | grep exec
execve("/lib64/ld-linux-x86-64.so.2", ["/lib64/ld-linux-x86-64.so.2", "./hello"], 0x7fff1206f208 /* 9 vars */) = 0 
```

`/lib64/ld-linux-x86-64.so.2`本身应该是内核调用执行的，但我们这里可以直接进行调用。这样一方面可以在没有执行权限的情况下执行任意代码，另一方面也可以在一定程度上避免内核对 execve 的异常监控。

 

利用 (滥用)interpreter 我们还可以做其他有趣的事情，比如通过修改指定 ELF 文件的 interpreter 为我们自己的可执行文件，可让内核在处理目标 ELF 时将控制器交给我们的 interpreter，这可以通过直接修改字符串表或者使用一些工具如 [patchelf](https://github.com/NixOS/patchelf) 来轻松实现。

 

对于恶意软件分析的场景，很多安全研究人员看到 ELF 就喜欢用 [ldd](https://man7.org/linux/man-pages/man1/ldd.1.html) 去看看有什么依赖库，一般 ldd 脚本实际上是调用系统默认的 [ld.so](https://man7.org/linux/man-pages/man8/ld.so.8.html) 并通过环境变量来打印信息，不过对于某些 glibc 实现 (如 glibc2.27 之前的 ld.so)，会调用 ELF 指定的 interpreter 运行，从而存在非预期命令执行的风险。

 

当然还有更多其他的思路可以进行拓展，这就需要大家发挥脑洞了。

加固 / 脱壳
-------

与逆向分析比较相关的就是符号表，一个有符号的程序在逆向时基本上和读源码差不多。因此对于想保护应用程序的开发者而言，最简单的防护方法就是去除符号表，一个简单的 **strip** 命令就可实现。strip 删除的主要是 Section 中的信息，因为这不影响程序的执行。去除前后进行 diff 对比可看到删除的 section 主要有下面这些:

```
$ diff 0 1
1c1
< There are 35 section headers, starting at offset 0x1fdc:
---
> There are 28 section headers, starting at offset 0x1144:
32,39c32
<   [27] .debug_aranges    PROGBITS        00000000 00104d 000020 00      0   0  1
<   [28] .debug_info       PROGBITS        00000000 00106d 000350 00      0   0  1
<   [29] .debug_abbrev     PROGBITS        00000000 0013bd 000100 00      0   0  1
<   [30] .debug_line       PROGBITS        00000000 0014bd 0000cd 00      0   0  1
<   [31] .debug_str        PROGBITS        00000000 00158a 000293 01  MS  0   0  1
<   [32] .symtab           SYMTAB          00000000 001820 000480 10     33  49  4
<   [33] .strtab           STRTAB          00000000 001ca0 0001f4 00      0   0  1
<   [34] .shstrtab         STRTAB          00000000 001e94 000145 00      0   0  1
---
>   [27] .shstrtab         STRTAB          00000000 00104d 0000f5 00      0   0  1

```

其中**.symtab** 是符号表，**.strtab** 是符号表中用到的字符串。

 

仅仅去掉符号感觉还不够，熟悉汇编的人放到反编译工具中还是可以慢慢还原程序逻辑。通过前面的分析我们知道，ELF 执行需要的只是 Program Header 中的几个段，Section Header 实际上是不需要的，只不过在运行时动态链接过程会引用到部分关联的区域。大部分反编译工具，如 IDA、Ghidra 等，处理 ELF 是需要某些 section 信息来构建程序视图的，所以我们可以通过构造一个损坏 Section Table 或者 ELF Header 令这些反编译工具出错，从而干扰逆向人员。

 

当然，这个方法并不总是奏效，逆向人员可以通过动态调试把程序 dump 出来并对运行视图进行还原。一个典型的例子是 Android 中的 JNI 动态库，有的安全人员对这些 so 文件进行了加密处理，并且在`.init/.initarray`这些动态库初始化函数中进行动态解密。破解这种加固方法的策略就是将其从内存中复制出来并进行重建，重建的过程可根据 segment 对 section 进行还原，因为 segment 和 section 之间共享了许多内存空间，例如:

```
$ readelf -l main1
...
 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-id
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .dynamic .got

```

在`Section to Segment mapping`中可以看到这些段的内容是跟对应 section 的内容重叠的，虽然一个 segment 可能对应多个 section，但是可以根据内存的读写属性、内存特征以及对应段的一般顺序进行区分。

 

如果程序中有比较详细的日志函数，我们还可以通过反编译工具的脚本拓展去修改`.symtab/.strtab`段来批量还原 ELF 文件的符号，从而高效地辅助动态调试。

Binary Fuzzing
--------------

考虑这么一种场景，我们在分析某个 IoT 设备时发现了一个定制的 ELF 网络程序，类似于 httpd，其中有个静态函数负责处理输入数据。现在想要单独对这个函数进行 fuzz 应该怎么做？直接从网络请求中进行变异是一种方法，但是网络请求的效率太低，而且触达该函数的程序逻辑也可能太长。

 

既然我们已经了解了 ELF，那就可以有更好的办法将该函数抽取出来进行独立调用。在介绍 ELF 类型的时候其实有提到，可执行文件可以有两种类型，即可执行类型 (`ET_EXEC`) 和共享对象 (`ET_DYN`)，一个动态链接的可执行程序默认是共享对象类型的:

```
$ gcc hello.c -o hello
$ readelf -h hello | grep Type
  Type:  DYN (Shared object file)

```

而动态库 (.so) 本身也是共享对象类型，他们之间的本质区别在于前者链接了 libc 并且定义了 main 函数。对于动态库，我们可以通过`dlopen/dlsym`获取对应的符号进行调用，因此对于上面的场景，一个解决方式就是修改目标 ELF 文件，并且将对应的静态函数导出添加到 dynamic section 中，并修复对应的 ELF 头。

 

这个思想其实很早就已经有人实现了，比如 [lief 的 bin2lib](https://lief.quarkslab.com/doc/latest/tutorials/08_elf_bin2lib.html)。通过该方法，我们就能将目标程序任意的函数抽取出来执行，比如 hugsy 就用这个方式复现了 Exim 中的溢出漏洞 (CVE-2018-6789)，详见 [Fuzzing arbitrary functions in ELF binaries](https://blahcat.github.io/2018/03/11/fuzzing-arbitrary-functions-in-elf-binaries/)([中文翻译](https://www.anquanke.com/post/id/100801))。

总结
==

本文主要介绍了 32 位环境下 ELF 文件的格式和布局，然后从内核空间和用户空间两个方向分析了 ELF 程序的加载过程，最后列举了几个依赖于 ELF 文件特性的案例进行具体分析，包括 dynamic linker 的滥用、程序加固和反加固以及在二进制 fuzzing 中的应用。

 

ELF 文件本身并不复杂，只有三个关键部分，只不过在 section 和 segment 的类型上保留了极大的拓展性。操作系统可以根据自己的需求在不同字段上实现和拓展自己的功能，比如 Linux 中通过 dymamic 类型实现动态加载。但这不是必须的，例如在 Android 中就通过 ELF 格式封装了特有的`.odex`、 `.oat`文件来保存优化后的 dex。另外对于 64 位环境，大部分字段含义都是类似的，只是字段大小稍有变化 (Elf32->Elf64)，并不影响文中的结论。

参考链接
====

*   [Linux Foundation Referenced Specifications](https://refspecs.linuxfoundation.org/)
*   [Executable and Linkable Format (ELF)](http://www.cs.yale.edu/homes/aspnes/pinewiki/attachments/ELF(20)format/ELF_format.pdf)
*   [Tool Interface Standard (TIS) Executable and Linking Format (ELF) Specification Version 1.2](https://refspecs.linuxfoundation.org/elf/elf.pdf)
*   [elf(5) - format of Executable and Linking Format (ELF) files](https://man7.org/linux/man-pages/man5/elf.5.html)
*   [How programs get run: ELF binaries](https://lwn.net/Articles/631631/)
*   [深入了解 GOT,PLT 和动态链接](https://evilpan.com/2018/04/09/about-got-plt/)

 

博客地址: [https://evilpan.com/](https://evilpan.com/) 排版带侧边目标，PC 阅读体验更佳。  
![](https://bbs.pediy.com/upload/attach/202008/844554_R99VG5JREW5M4C6.png)

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)