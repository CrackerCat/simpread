> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270935.htm)

> [原创] ELF 文件中的各个节区

> 版权声明：本文为 CSDN 博主「ashimida@」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。  
> 原文链接：https://blog.csdn.net/lidan113lidan/article/details/119901186
> 
> 更多内容可关注微信公众号 ![](https://bbs.pediy.com/upload/attach/202112/PQ6AK783G5VSGBX.jpg)    

**ELF 节区信息概述:**

<table width="1076"><tbody><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>节区名</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>节区说明</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p><strong>备注</strong> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.rodata1</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>和 rodata 类似，只存放只读数据</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.comment</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>存放编译器版本信息，如字符串 "GCC:(GNU)4.2.0"</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.debug</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>存放调试信息</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.shstrtab</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>节区头部表名字的字符串表 (Section Header String Table)</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.plt</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>过程链接表 (Procedure Linkage Table), 用来保存长跳转格式的函数调用</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.got</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>全局偏移表 (Global Offset Table)，在地址无关代码中才需要，所有只读段需要修复的位置都间接引用到此表, 因此只读段自身就无需修复，只需修复此 got 表即可.</p><p>.got 表是在编译期间确定，静态链接期间生成的</p><p>而. plt 表是在静态链接期间确定，静态链接期间生成的</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>可执行文件通常不论如何编译都有 got 表，这是因为是否加入 got 表是由编译 (cc1) 期间决定的，而可执行文件默认连接的多个目标文件默认都有 got 表元素.</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.got.plt</p></td><td width="292">实际上其本质是从. got 表中拆除来的一部分，当开启延迟绑定 (Lazy&nbsp;Binding) 时，会将 plt 表中的长跳转 (函数) 的重定位信息单独放到此表中，以满足后续实际的延迟绑定.</td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.symtab</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>(静态链接) 符号表的作用是保存当前<strong>目标文件</strong>中所有对符号的定义和引用.</p><p>*&nbsp;符号表中 UND 的符号不是当前目标文件定义的，也就是对符号的引用</p><p>*&nbsp;符号表中其他非 UND 的符号，全部是定义在当前目标文件中的，也就是对符号的定义</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>默认所有非. L 开头的符号都要输出，.L 开头的不输出</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.strtab</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>静态链接字符串表，其中记录的是静态链接符号表中使用到的字符串，这些字符串仅供静态链接符号表使用，strip 的时候会将. symtab 和. strtab 两个段完全清除.</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>符号表的第一个元素必须是<strong> STN_UNDEF，</strong>其代表一个未定义的符号索引，此符号表项内部所有值都为 0</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.group</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>是用来记录多个节区的相关信息的，比如说代码段引用了数据段，这种信息是为了保证二进制处理时候不会操作错误的，像是 elf 自动生成的，细节见链接</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p><a href="https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblj/index.html" title="">File Format (Linker and Libraries Guide) </a>&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;</p><p>section groups</p></td></tr><tr><td colspan="3" width="639">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>动态链接相关节区 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.interp</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.interp 整个段的内容就是一个字符串，此字符串为系统中动态链接器的路径，如：</p><p>/lib/ld-linux-aarch64.so.1</p><p>linux 的可执行文件加载时会去寻找可执行文件所需的动态链接器</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;</td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.dynamic</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.interp 保存的是动态链接器路径，.dynamic 中保存的是动态链接器用到的基本信息, 如动态链接符号表 (.dynsym)，字符串表 (.dynstr), 重定位表 (.rela.dyn/rela.plt), 依赖的运行时库，库查找路径等</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.rela.dyn</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>记录所有变量的动态链接重定位信息（.rela.plt 记录的是函数），与. rela.plt 一起，是系统中唯二的两张动态链接重定位表。</p><p>.rela.dyn 记录除了. plt 段之外所有段的动态链接重定位信息，若开启了地址无关代码，那么这些信息都应该只与. got 段的地址有关.</p></td><td width="291"><br></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.rela.plt</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>&nbsp; 过程连接表的动态链接重定位表，只要有过程链表，通常就会有此表，因为 plt 导致了绝对跳转，那么所有 plt 表中所有需要动态链接 / 重定位的绝对地址 (可能在. got.plt 或. got 中，依赖于是否开启延迟绑定), 都需要通过. rela.plt 记录&nbsp;&nbsp;</p><p>&nbsp; 此表中记录所有全局函数 (长跳转函数) 的动态链接重定位信息，与. rela.dyn 一起，是系统中唯二的两张动态链接重定位表。</p><p>.rela.plt 实际上记录的是. plt 段的动态链接重定位信息，若未开启 lazy binding, 则这这些信息应该都只与. got 段的地址有关；若开启 lazy binding, 则这些信息应该都只与. got.plt 段的地址有关;</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p><strong>需要动态链接重定位的原因主要是模块有导入符号的存在</strong>, 这些符号在运行时才能确定,<strong> 地址无关代码并不能改变未定符号的本质</strong> (即不影响模块是否需要动态链接重定位),<strong> 但地址无关代码可以让重定位变得简单</strong> (如仅重定位 .got/ .data/ .got.plt)</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.dynsym</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>&nbsp; 动态链接符号表，其格式和. symtab 一样，readelf -s 会尝试同时输出. dynsym 和. symtab，如右图.</p><p>&nbsp; <strong>动态链接符号表是静态链接符号表的子集，其只保留了与动态链接相关的符号信息</strong>，所有模块内部符号则不保留 (因此静态符号表是可以被 strip 的，其只对于目标文件有用).</p><p>&nbsp; 动态链接符号表中未定义的符号 (符号引用), 又称为导入符号 (类似导入表)</p><p>&nbsp; 动态链接符号表中已定义的符号 (符号定义), 又称为导出符号 (类似导出表)</p></td><td width="291">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>全局符号默认是直接导出到动态链接重定位表的</p></td></tr><tr><td width="55">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>.dynstr</p></td><td width="292">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<p>动态链接符号表用到的字符串表，其与静态链接字符串表 (.strtab) 分开的原因应该是. strtab 是可以完全 strip 的</p></td><td width="291"><br></td></tr></tbody></table>

**链接过程图:**

![](https://bbs.pediy.com/upload/attach/202112/490870_ZU6P2H4AWKS3WW4.jpg)

**1.ELF 文件头**

  ELF 的基础文件格式如下图所示:

![](https://bbs.pediy.com/upload/attach/202112/490870_PS3UUDQQ8TXCGVX.jpg)

 由 ELF 文件的第一字节开始的一个 ELF32_Ehdr 结构体是 ELF 头，其中记录 ELF 的基本信息包括:

    * 32/64 位

    * 大 / 小端

    * ELF 版本号

    * 文件类型 (ET_REL/ET_EXE/ET_DYN, 分别对应 relocatable/pde/[pie|dll] 文件)    

    * 体系结构

    * 入口地址

    * 程序头部表偏移，表项大小，表项个数 (**见 2, 程序头部表**)

    * 节区头部表偏移，表项大小，表项个数，对应的字符串表 (**见 1，节区头部表**)

    * ELF 文件头大小

**2. 节区头部表 (Section Heade Table****)、节区头部表字符串表 (shstrtab) 和字符串表(strtab)**

  节区头部表实际上是一个 Elf_Shdr[m]数组，其中的每一个元素 (表项) 记录系统中一个节区的信息，包括如节区名, 类型, flag, 内存 / 文件起始地址, 大小, 对齐等信息, 如:

```
typedef struct elf64_shdr {
  Elf64_Word sh_name;        /* Section name, index in string tbl */
  Elf64_Word sh_type;        /* Type of section */
  Elf64_Xword sh_flags;        /* Miscellaneous section attributes */
  Elf64_Addr sh_addr;        /* Section virtual addr at execution */
  Elf64_Off sh_offset;        /* Section file offset */
  Elf64_Xword sh_size;        /* Size of section in bytes */
  Elf64_Word sh_link;        /* Index of another section */
  Elf64_Word sh_info;        /* Additional section information */
  Elf64_Xword sh_addralign;    /* Section alignment */
  Elf64_Xword sh_entsize;    /* Entry size if section holds table */
} Elf64_Shdr;

```

通过 readelf -S 可读取其内容，如:

```
##这里手动换行，便于观察
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -S x.o
There are 11 section headers, starting at offset 0x2e8:
Section Headers:
  [Nr] Name              Type             Address           Offset        Size              EntSize              Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000     0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040      000000000000002c  0000000000000000  AX       0     0     4
  [ 2] .rela.text        RELA             0000000000000000  00000218     0000000000000078  0000000000000018   I       8     1     8
  [ 3] .data             PROGBITS         0000000000000000  0000006c     0000000000000004  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  00000070     0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .rodata           PROGBITS         0000000000000000  00000070     0000000000000008  0000000000000000   A       0     0     8
  [ 6] .comment          PROGBITS         0000000000000000  00000078     0000000000000031  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000a9     0000000000000000  0000000000000000           0     0     1
  [ 8] .symtab           SYMTAB           0000000000000000  000000b0     0000000000000150  0000000000000018           9    11     8
  [ 9] .strtab           STRTAB           0000000000000000  00000200     0000000000000017  0000000000000000           0     0     1
  [10] .shstrtab         STRTAB           0000000000000000  00000290     0000000000000052  0000000000000000           0     0     1

```

  其中每一行是节区头部表中的一个表项，也代表着此 elf 文件中的一个节区信息 (ELF 文件中定位节区的方式是: ELF 头 => 节区头部表 =>节区首地址及其他信息)

  节区头部表字符串表是给节区头部表专门准备的字符串表，ELF 文件中通常存在两个字符串表:

    * 一个是代码中所有使用到的字符串的表, 名称为. strtab

    * 一个是记录所有节区名的字符串表，名称为. shstrtab

  二者通常是没有关系的，**节区头部表字符串表中只记录节区名称**，如下:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ aarch64-linux-gnu-readelf -p .shstrtab x
 
String dump of section '.shstrtab':
  [     1]  .symtab
  [     9]  .strtab
  [    11]  .shstrtab
  [    1b]  .interp
  [    23]  .note.ABI-tag
  [    31]  .note.gnu.build-id
  [    44]  .gnu.hash
  [    4e]  .dynsym
  [    56]  .dynstr
  [    5e]  .gnu.version
  [    6b]  .gnu.version_r
  [    7a]  .rela.dyn
  [    84]  .rela.plt
  [    8e]  .init
  [    94]  .text
  [    9a]  .fini
  [    a0]  .rodata
  [    a8]  .eh_frame
  [    b2]  .init_array
  [    be]  .fini_array
  [    ca]  .dynamic
  [    d3]  .got
  [    d8]  .got.plt
  [    e1]  .data
  [    e7]  .bss
  [    ec]  .comment

```

  而**字符串表则记录符号表中符号相关的字符串信息**，如:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ aarch64-linux-gnu-readelf -p .strtab x
 
String dump of section '.strtab':
  [     1]  /usr/lib/gcc-cross/aarch64-linux-gnu/7/../../../../aarch64-linux-gnu/lib/../lib/crt1.o
  [    58]  $d
  [    5b]  $x
  [    5e]  /usr/lib/gcc-cross/aarch64-linux-gnu/7/../../../../aarch64-linux-gnu/lib/../lib/crti.o
  [    b5]  call_weak_fn
  [    c2]  /usr/lib/gcc-cross/aarch64-linux-gnu/7/../../../../aarch64-linux-gnu/lib/../lib/crtn.o
  [   119]  crtstuff.c
  [   124]  deregister_tm_clones
  [   139]  __do_global_dtors_aux
  [   14f]  completed.8500
  [   15e]  __do_global_dtors_aux_fini_array_entry
  [   185]  frame_dummy
  [   191]  __frame_dummy_init_array_entry
  [   1b0]  x.c
  [   1b4]  elf-init.oS
  [   1c0]  __FRAME_END__
  [   1ce]  __init_array_end
  [   1df]  _DYNAMIC
  [   1e8]  __init_array_start
  [   1fb]  _GLOBAL_OFFSET_TABLE_
  [   211]  __libc_csu_fini
  [   221]  _ITM_deregisterTMCloneTable
  [   23d]  __bss_start__
  [   24b]  _edata
  [   252]  __bss_end__
  [   25e]  __libc_start_main@@GLIBC_2.17
  [   27c]  __data_start
  [   289]  __gmon_start__
  [   298]  __dso_handle
  [   2a5]  abort@@GLIBC_2.17
  [   2b7]  _IO_stdin_used
  [   2c6]  __libc_csu_init
  [   2d6]  yyy
  [   2da]  __end__
  [   2e2]  __bss_start
  [   2ee]  main
  [   2f3]  __TMC_END__
  [   2ff]  _ITM_registerTMCloneTable
  [   319]  printf@@GLIBC_2.17

```

**3. 程序头部表 (Program Header Table****)**

  程序头部表实际上是一个 Elf_Phdr[n]数组, 其元素 (表项) 个数与节区头部表项个数 (m) 一般是不同的.

  程序头部表关注的是 ELF 文件加载时会有几个不同的属性的段, 关心如何将相同属性的段合并为一个 segment, 以优化时间和空间消耗 (只有 dll/exe 文件拥有程序头部表，relocatable 文件没有程序头部表)

  程序头部表格式如下:

```
typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;        /* Segment file offset */
  Elf64_Addr p_vaddr;        /* Segment virtual address */
  Elf64_Addr p_paddr;        /* Segment physical address */
  Elf64_Xword p_filesz;        /* Segment size in file */
  Elf64_Xword p_memsz;        /* Segment size in memory */
  Elf64_Xword p_align;        /* Segment alignment, file & memory */
} Elf64_Phdr;

```

  通过 readelf -l 可查看程序头部表，如下:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -l x
Elf file type is EXEC (Executable file)
Entry point 0x400660
There are 8 program headers, starting at offset 64
 
Program Headers:
  Type           Offset             VirtAddr           PhysAddr           FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040 0x00000000000001c0 0x00000000000001c0  R      0x8
  INTERP         0x0000000000000200 0x0000000000400200 0x0000000000400200 0x000000000000001b 0x000000000000001b  R      0x1    [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000 0x0000000000000860 0x0000000000000860  R E    0x10000
  LOAD           0x0000000000000de8 0x0000000000410de8 0x0000000000410de8 0x000000000000024c 0x0000000000000258  RW     0x10000
  DYNAMIC        0x0000000000000df8 0x0000000000410df8 0x0000000000410df8 0x00000000000001e0 0x00000000000001e0  RW     0x8
  NOTE           0x000000000000021c 0x000000000040021c 0x000000000040021c 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000000de8 0x0000000000410de8 0x0000000000410de8 0x0000000000000218 0x0000000000000218  R      0x1
 
Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-i
   06     
   07     .init_array .fini_array .dynamic .got

```

  这里可以看到 Elf_Phdr[n]数组只有 8 个元素，每个段 (segment) 包含多少个节应该是根据 section 的地址排序确定的, 其中:

    * INTERP: 此表项的 offset 实际上指向. interp 段的内容，也就是动态链接器的全路径的字符串，如 / lib/ld-linux-aarch64.so.1

    * LOAD: 代表可加载的 segment, 只有此类型的 segment 在运行时会载入内存

    * DYNAMIC: 对于动态链接的二进制，此 segment 记录动态链接器信息段 (.dynamic) 的指针

    * NOTE: 记录辅助信息，如对于 core dump 文件, 此 segment 包含 core dump 文件创建时的进程状态信息，如导致 crash 的 signal,pending & held signals, 进程 / 父进程 UID,nice, 寄存器值 (包括当前 pc) 等.

    * GNU_STACK: 此段并没有任何内容，但其属性用来代表当前进程的 stack 是否可执行.

    * GNU_RELRO: 此段代表重定位之后哪些段需要设置为 RO 的 (若. got.plt 没有在这里，代表其最后不受到 RO 保护，因为延迟绑定，是正常的，解决方法是关闭延迟绑定)

    * GNU_EH_FRAME: 保存栈帧的 unwind 信息, 若存在此段通常指向. eh_fram_hdr(eh = Exception Handling)

    * TLS: 代表线程局部存储的信息

**4..text/.data/.rodata/.bss 节区**

  这四个节区是 ELF 的基本节区，其中:

    * .text 是默认的代码段

    * .data 是默认的 rw 数据段

    * .rodata 是默认的只读数据段

    * .bss 是未初始化或初值为 0 的全局变量所在节区

       .bss 中的数据因为默认值为 0，故不需要在文件中为其分配空间，程序运行时在内存申请空间即可

  这些节区实际上是在编译 (cc1) 的时候就确定下来了, 在直接编译出的汇编代码 (.s) 中可以直接看到如. text/.data 的定义，这就是目标文件中这些节区的由来, as 只是忠实的将汇编代码转换为目标文件中对应的节区, 如:

```
  1     .arch armv8-a                                                                                                                                               
  2     .file   "x.c"
  3     .text
  4     .global x
  5     .data
  6     .align  2
  7     .type   x, %object
  8     .size   x, 4
  9 x:
  ......

```

  而对于链接后的文件 (exe/dll/relocatable) 来说, 还涉及如将多个. o 中的. text 段合并的过程，合并方法记录在默认的链接脚本中，可通过 ld -verbose 来查看, 如:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ ld -verbose
GNU ld (GNU Binutils for Ubuntu) 2.30
......
SECTIONS
{
  PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000)); . = SEGMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;
  .interp         : { *(.interp) }
  ......
  .dynsym         : { *(.dynsym) }
  .dynstr         : { *(.dynstr) }
  ......
  .rela.dyn       :
    {
      ......
      *(.rela.text .rela.text.* .rela.gnu.linkonce.t.*)
      *(.rela.rodata .rela.rodata.* .rela.gnu.linkonce.r.*)
      *(.rela.data .rela.data.* .rela.gnu.linkonce.d.*)
      *(.rela.got)
       ......
    }
  .rela.plt       :
    {
      *(.rela.plt)
      .......
    }
  ......
  .plt            : { *(.plt) *(.iplt) }
  .plt.got        : { *(.plt.got) }
  .plt.sec        : { *(.plt.sec) }
  .text           :
  {
    *(.text.unlikely .text.*_unlikely .text.unlikely.*)
    *(.text.exit .text.exit.*)
    ......
  }
  ......
  PROVIDE (__etext = .);
  PROVIDE (_etext = .);
  PROVIDE (etext = .);
  .rodata         : { *(.rodata .rodata.* .gnu.linkonce.r.*) }
  .rodata1        : { *(.rodata1) }
  ......
  .data.rel.ro : { *(.data.rel.ro.local* .gnu.linkonce.d.rel.ro.local.*) *(.data.rel.ro .data.rel.ro.* .gnu.linkonce.d.rel.ro.*) }
  .dynamic        : { *(.dynamic) }
  .got            : { *(.got) *(.igot) }
  . = DATA_SEGMENT_RELRO_END (SIZEOF (.got.plt) >= 24 ? 24 : 0, .);
  .got.plt        : { *(.got.plt)  *(.igot.plt) }
  .data           :
  {
    *(.data .data.* .gnu.linkonce.d.*)
    SORT(CONSTRUCTORS)
  }
  .data1          : { *(.data1) }
  _edata = .; PROVIDE (edata = .);
  . = .;
  __bss_start = .;
  .bss            :
  {
   *(.dynbss)
   *(.bss .bss.* .gnu.linkonce.b.*)
   *(COMMON)
   ......
  }
  .......
}

```

**5. 符号表 (symtab) 节区**

  符号表中符号的实际上指的是汇编代码或汇编器 (as) 中的对符号的定义。根据 as 手册, 符号是一个中心概念，程序使用符号来命名事物，链接器使用符号来链接，调试器使用符号来调试(as 并没有将目标文文件中的符号以其出现顺序排序), 如:

```
1     .arch armv8-a                                                     
2     .file   "x.c"3     .text
4     .global xx
5     .data
6     .align  2
7     .type   xx, %object
8     .size   xx, 4
9 xx:
10     .word   1
11     .section    .rodata
12     .align  3
13 .LC0:
14     .string "x:%d,%d\n"15     .text
16     .align  2
17     .global main
18     .type   main, %function19 main:
20     stp x29, x30, [sp, -16]!
21     add x29, sp, 0
22     adrp    x0, :got:xx
23     ldr x0, [x0, #:got_lo12:xx] ......

```

  以上代码中 xx/.LC0/main 均为符号，除此之外: 如 file 和每个 section 都会在最终目标文件中输出一个符号, 而以. L 开头的符号则默认不会输出到目标文件 (除非指定 - L) 选项，如:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -s x.o
 
Symbol table '.symtab' contains 15 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS x.c    //文件名符号
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    3 $d    //数据段开始符号
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     7: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    5 $d
     8: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    1 $x    //代码段开始符号
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
    11: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    3 xx
    12: 0000000000000000    56 FUNC    GLOBAL DEFAULT    1 main
    13: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND yyy
    14: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf

```

  所以说 ELF 文件中**符号表的作用实际上是记录汇编代码中出现的符号** (根据 as 选项不同决定. L 开头的符号是否输出)。

**6. 过程链接表 (Procedure Linkage Table,plt 表) 节区, 过程链接表动态重定位表 (.rela.plt) 节区**

  过程链接表是在链接阶段 (ld) 生成的, 其作用是用来链接不同函数之间的调用; 在 cc1 的编译过程中只是指定了符号的调用，并不关心具体是如何调用的，如下：

```
//x.s文件
......
19 func22:
......
27     ldr w2, [x0]
28     adrp    x0, .LC0
29     add x0, x0, :lo12:.LC0
30     bl  printf                    //1)
31     nop
32     ldp x29, x30, [sp], 16
33     ret
......
38 main:
39     stp x29, x30, [sp, -16]!
40     add x29, sp, 0
41     bl  func22                    //2)
42     mov w0, 0
......
 
//目标文件反汇编
Disassembly of section .text:
0000000000000000 :
   0:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
   4:   910003fd        mov     x29, sp
   8:   90000000        adrp    x0, 0  c:   f9400000        ldr     x0, [x0]
  10:   b9400001        ldr     w1, [x0]
  14:   90000000        adrp    x0, 0  18:   f9400000        ldr     x0, [x0]
  1c:   b9400002        ldr     w2, [x0]
  20:   90000000        adrp    x0, 0  24:   91000000        add     x0, x0, #0x0
  28:   94000000        bl      0  // 1)
  2c:   d503201f        nop
  30:   a8c17bfd        ldp     x29, x30, [sp], #16
  34:   d65f03c0        ret
 
0000000000000038 

:
  38:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  3c:   910003fd        mov     x29, sp
  40:   94000000        bl      0  // 2)
  44:   52800000        mov     w0, #0x0                        // #0
  48:   a8c17bfd        ldp     x29, x30, [sp], #16
  4c:   d65f03c0        ret 
```

  在汇编文件和目标文件中，都只是简单的指定了 (函数) 符号调用，并不区分此符号是已定义还是未定义的, 而在 (aarch64 平台) 链接过程中全局函数和局部函数是否会有 PLT 表取决于是否存在对应的引用，以及如何引用，其对应的源码如下:

```
./binutil/gold/aarch64.cc
//此函数负责对局部符号的重定位，STT_GNU_IFUNC 参考https://sourceware.org/glibc/wiki/GNU_IFUNC https://www.airs.com/blog/archives/403
Target_aarch64::Scan::local(...)
{
  bool is_ifunc = lsym.get_st_type() == elfcpp::STT_GNU_IFUNC;
  if (is_ifunc && this->reloc_needs_plt_for_ifunc(object, r_type))
    target->make_local_ifunc_plt_entry(symtab, layout, object, r_sym);
  ......
}
 
//对于全局符号重定位，只要是CALL26/JUMP26都直接做plt_entry
Target_aarch64::Scan::global(......)
{
    case elfcpp::R_AARCH64_TSTBR14:
    case elfcpp::R_AARCH64_CONDBR19:
    case elfcpp::R_AARCH64_JUMP26:
    case elfcpp::R_AARCH64_CALL26:
      {
    if (gsym->final_value_is_known())
      break;
    if (gsym->is_defined() &&
        !gsym->is_from_dynobj() &&
        !gsym->is_preemptible())
      break;
    // Make plt entry for function call.
    target->make_plt_entry(symtab, layout, gsym);
    break;
      }   
    ......
} 
```

  这里没有仔细分析这段源码，但根据 <ELF for the ARM 64-bit Architecture (AArch64)> 手册:

>   一个 PLT 入口代表到一个可执行文件外部的长跳转，通常来说，在静态链接的过程中只知道目标符号的名称，而不知道其地址，这样的位置称为一个导入位置或导入符号    
> 
>   基于 SysV 的 DSOs(Dynamic Shared Objects, 如 for linux) 同样需要从可执行文件导出的函数拥有 PLT 入口. 事实上导出函数被当做导入函数对待，这样在动态链接时其定义才有可能被重写.    
> 
>   静态链接器必须为每个存在的 AARCH64 B/BL 系列指令的重定位指令引用的符号产生一个 PLT 入口，在 linux/sysV DSO 中，如 STB_GLOBAL 符号 + STV_DEFUALT 可见性的就是个候选指令.    

  也就是**大体来说只要是全局函数，且被 B/BL 引用的 (代码中的 CALL26/JUMP26), 基本上都要放到 plt 表** (在 ld 选项中没有看到可以完全关闭 plt 表的编译选项，-static 编译也是默认有 plt 表的).

  链接器为函数创建 plt 表后调用点的代码如下:

```
125 00000000000004a0 :                     //2) 再间接跳转到真正函数函数
126  4a0:   b0000090    adrp    x16, 11000  127  4a4:   f9400211    ldr x17, [x16]
128  4a8:   91000210    add x16, x16, #0x0
129  4ac:   d61f0220    br  x17                        //3) 绝对地址跳转
......
161 0000000000000508 

:
162  508:   a9bf7bfd    stp x29, x30, [sp, #-16]!
163  50c:   910003fd    mov x29, sp
164  510:   97ffffe4    bl  4a0  //1) 跳转到函数的plt表
165  514:   52800000    mov w0, #0x0                   
166  518:   a8c17bfd    ldp x29, x30, [sp], #16
167  51c:   d65f03c0    ret 
```

  当链接器为函数创建对应的 func@plt 代码后：

    * 被调用点 (如这里的 main) 会利用相对寻址调用到 plt 表中的函数(这里的 func22@plt)

    * plt 表中的函数 (func22@plt) 在运行时通过绝对地址寻址 (范围可覆盖整个地址空间) 调用真正的函数(func22)

    * 链接器将上述代码中所有函数的 @plt 代码都放到了一个名为. plt 的节区中，所有函数的 @plt 代码，构成了 ELF 中的. plt 表 (过程链接表).

  由上述代码还可得知，在经过 plt 表处理后的函数，最终的跳转是由绝对地址跳转实现的，对于可变加载地址和动态链接过程来说，此绝对地址在运行时必然是需要被修复的（前者是直接动态链接时重定位修复，后者是动态链接时查找符号修复，二者区别在于前者是已定义符号，后者是未定义符号)，故在链接后的 exe/dll 文件中就有一个. rela.plt 表, 专门用来记录. plt 段相关的动态链接重定位信息。

   根据 [1] 可知, 内核动态链接一共有两个表，分别是. rela.dyn 和. rela.plt, 其中：

    * .rela.plt 记录的是. plt 表相关的动态链接信息

        这个名字很像静态链接重定位表，但实际上是链接阶段生成的 plt 的动态链接重定位表。在链接之前，汇编器是没有生成 plt 节区的，链接后的 dll/exe 文件中通过此节区记录 plt 节区的动态链接信息. 实际上 rela.plt 是保存所有加载时必须重定位的**函数**的信息，其中每一个表项都是一个 ELF64_Rela 结构体

    * .rela.dyn 记录的是 dll/exe 文件中除了 plt 节区外，其他节区的动态链接信息

      .rela.dyn 保存所有加载时必须重定位的变量的信息，其中每一个表项都是一个 ELF64_Rela 结构体

   **ELF 文件中除了. rela.plt 和. rela.dyn 这两个节区是保存函数 / 变量的动态链接重定位表外，其他的. rela.xxx 保存的都是静态链接的重定位信息**.

   * 在未开启地址无关代码的情况下:

     rela.dyn 中需要重定的地址可能出现在任何内存段，如 text/data/bss

     rela.plt 中的重定位地址出现在. got 或. got.plt 段中

   * 在仅开启了地址无关代码 (fPIC/fPIE + -pie/-shared)，未开启延迟绑定(Lazy Binding) 的情况下：

       rela.plt 和 rela.dyn 这两个动态链接重定位表的重定位信息都在. got 段内

   * 在开启了地址无关代码 (fPIC/fPIE + -pie/-shared)，同时开启延迟绑定(Lazy Binding) 的情况下：

       rela.plt 中需重定位的地址均出现在. got.plt 段中

       rela.dyn 中需要重定位的地址信息均出现在. got 段中

**7. 全局偏移表 (got 表) 节区和. got.plt 节**

  全局偏移表 (Global Offset Table, got 表) 的作用是用来构造地址无关代码，地址无关代码按照 ld 的代码理解应该是: 在加载地址不同的情况下，程序中的所有只读数据运行时也不需要修改;

  而具体的做法就是是将代码中所有需要被重定位的位置，都通过间接引用的方式放到. got 表中，运行时只需要修改. got 表中的数据即可实现只读代码不被修改.

  .got 表和. plt 表的相似点是，**二者都是在链接过程中生成的**；不同点在于，默认情况下:

   * .got 表是由编译阶段确定的，链接过程只是按照编译阶段的标定来处理

   * 对哪些函数生成. plt 表是由链接过程来决定的

  如下的汇编代码 (目标文件) 中就显示的标注了使用 got 表：

```
......
18     .type   func22, %function
19 func22:
20     stp x29, x30, [sp, -16]!
21     add x29, sp, 0
22     adrp    x0, :got:xx                                                                                                                                         
23     ldr x0, [x0, #:got_lo12:xx]
24     ldr w1, [x0]
25     adrp    x0, :got:yyy                //aarch64汇编中通过  :got:标注当前符号要放到.got表中
26     ldr x0, [x0, #:got_lo12:yyy]
27     ldr w2, [x0]
......
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -r --wide x.o
Relocation section '.rela.text' at offset 0x280 contains 8 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000008  0000000b00000137 R_AARCH64_ADR_GOT_PAGE 0000000000000000 xx + 0
000000000000000c  0000000b00000138 R_AARCH64_LD64_GOT_LO12_NC 0000000000000000 xx + 0
0000000000000014  0000000d00000137 R_AARCH64_ADR_GOT_PAGE 0000000000000000 yyy + 0   //目标文件通过静态链接重定位类型来代表yyy是got表中的元素
0000000000000018  0000000d00000138 R_AARCH64_LD64_GOT_LO12_NC 0000000000000000 yyy + 0
0000000000000020  0000000600000113 R_AARCH64_ADR_PREL_PG_HI21 0000000000000000 .rodata + 0
0000000000000024  0000000600000115 R_AARCH64_ADD_ABS_LO12_NC 0000000000000000 .rodata + 0
0000000000000028  0000000e0000011b R_AARCH64_CALL26       0000000000000000 printf + 0
0000000000000040  0000000c0000011b R_AARCH64_CALL26       0000000000000000 func22 + 0
 
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -S x.o|grep got
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -S x|grep got
  [20] .got              PROGBITS         0000000000010f98  00000f98                //只有最终的dll/exe文件才有got表

```

  .got.plt 表实际上和. got 表的原理是一样的，只不过是将函数的重定位信息单独的保存到. got.plt 表了，这主要和延迟绑定 (Lazy Binding) 有关:

  * 若 ld 传入了参数支持延迟绑定 (-z lazy)，那么就会将函数(plt) 的动态链接重定位信息放到单独的. got.plt 表中;

  * 若未开启延迟绑定 (-z now)，那么函数(plt) 的动态链接重定位信息也放到. got 表中，.got 表是加载时直接绑定的.

  未开启 Lazy Binding 时的 ELF 文件和运行时重定位实现如下（这里以 pie 文件为例）:

![](https://bbs.pediy.com/upload/attach/202112/490870_GVSWPGV54K58BTG.jpg)

   开启 Lazy Binding 时的 ELF 文件和运行时重定位实现如下：

![](https://bbs.pediy.com/upload/attach/202112/490870_YTNHG4V9YF8KBCZ.jpg)

  注:

    .got.plt 表的前三项分别为:

    1) 内存中. dynamic 段的内存地址

    2) 当前模块 ID

    3) _dl_runtime_resolve 函数地址，在开启延迟绑定时，所有. got.plt 中待重定位的函数绝对地址指针都指向此函数，

       当函数第一次运行时，此函数负责将将当前指针指向真正要调用的正确的函数地址。

8. **动态链接信息 (.dynamic) 表**

    .interp 段中保存了**动态链接器**在系统中的路径，而. dynamic 段则保存了**动态链接器**所需要的基本信息，如依赖的共享对象 (dll)，动态链接符号表位置，动态链接重定位表位置，共享对象的初始化代码等.

    .dynamic 段中存的是 Elf32_Dyn 结构体的数组，其只有两个字段，如下:

```
typedef struct {
  Elf64_Sxword d_tag;        /* entry tag value */
  union {
    Elf64_Xword d_val;
    Elf64_Addr d_ptr;
  } d_un;
} Elf64_Dyn;

```

    其中 d_tag 是类型，d_un 是值，常见的类型如:

    * DT_SYMTAB: d_ptr 记录动态链接符号表 (.dynsym) 的地址偏移

    * DT_STRTAB: d_ptr 记录动态链接字符串表 (.dynstr) 的地址偏移

    * DT_STRSZ: d_val 记录动态链接字符串表 (.dynstr) 的大小

    * DT_HASH: d_ptr 表示动态链接 hash 表 (.hash) 的地址

    * DT_SONAME: 本共享对象的 SO-NAME

    * DT_RPATH: 动态链接共享对象的搜索路径

    * DT_INIT: 初始化代码地址

    * DT_FINIT: 结束代码地址

    * DT_NEED: 当前文件依赖的共享目标文件的文件名

    * DT_REL/DT_RELA: 动态链接重定位表地址

    * DT_RELAENT: 动态链接重定位表项的数目

    .dynamic 段的内容可通过 readelf -d 查看:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -d main
 
Dynamic section at offset 0xd78 contains 28 entries:
  Tag        Type                         Name/Value
0x0000000000000001 (NEEDED)             Shared library: [./y.so]        //依赖库，正常只是库文件名，若编译时驶入的名为 ./y.so，则这里就为./y.so
0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]     //依赖库，基本上所有非static的ELF文件中都要依赖于libc.so这个基础的系统库
0x000000000000000c (INIT)               0x730                           //.init段的文件偏移
0x000000000000000d (FINI)               0x99c                           //.finit段的文件偏移
0x0000000000000019 (INIT_ARRAY)         0x10d68                         //.init_array段的文件偏移
0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)                       //.init_array段的大小
0x000000000000001a (FINI_ARRAY)         0x10d70                         //.fini_array段的文件偏移
0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)                       //.fini_array段的大小
0x000000006ffffef5 (GNU_HASH)           0x260                           //.gnu.hash段的偏移
0x0000000000000005 (STRTAB)             0x488                           //动态链接字符串表(.dynstr)的文件偏移
0x0000000000000006 (SYMTAB)             0x2a8                           //动态链接符号表(.dynsym)的文件偏移
0x000000000000000a (STRSZ)              218 (bytes)                     //动态链接字符串表(.dynstr)的大小
0x000000000000000b (SYMENT)             24 (bytes)                      //动态链接符号表(.dynsym)中每个元素的大小
0x0000000000000015 (DEBUG)              0x0                             //运行时ld.so会将r_debug的运行时地址填充到这里，此信息可用于GDB调试
0x0000000000000003 (PLTGOT)             0x10f78                         //.got.plt段的内存偏移
0x0000000000000002 (PLTRELSZ)           120 (bytes)                     //.rela.plt段的大小
0x0000000000000014 (PLTREL)             RELA                            //.rela.plt段重定位类型是RELA还是REL
0x0000000000000017 (JMPREL)             0x6b8                           //.rela.plt段的地址
0x0000000000000007 (RELA)               0x5b0                           //.rela.dyn段的地址
0x0000000000000008 (RELASZ)             264 (bytes)                     //.rela.dyn段的大小
0x0000000000000009 (RELAENT)            24 (bytes)                      //.rela.dyn段中每个元素的大小
0x000000000000001e (FLAGS)              BIND_NOW                        //额外的flag，如 BIND_NOW, STATIC_TLS, TEXTREL等  当前代码是否为地址无关，是否延迟绑定都可以由这里确定
0x000000006ffffffb (FLAGS_1)            Flags: NOW PIE                  //额外的flag，如是否是地址无关代码
0x000000006ffffffe (VERNEED)            0x590                           //.gnu.version_r段的地址
0x000000006fffffff (VERNEEDNUM)         1                               //needed versions的数量
0x000000006ffffff0 (VERSYM)             0x562                           //.gnu.version段的地址
0x000000006ffffff9 (RELACOUNT)          6                               //.rela.plt段中RELA类型的元素个数？？
0x0000000000000000 (NULL)               0x0                             //结束标记

```

注:

    x86 平台可能还有一个. plt.got 段，但 aarch64 没有，见 [2,3]:

**9. 动态链接符号表 (.dynsym)，动态链接字符串表 (.dynstr)**

  在静态链接的过程中，有一个专门的节区叫做符号表 (.symtab), 其中保存的实际上是此目标文件中符号的定义和引用 (所有 UND 的内容都是符号的引用，其他的都是符号的定义); 

   动态链接的符号表和静态链接的比较相似，其区别在于动态链接的符号表指保存了与动态链接相关的符号，对于模块内部的符号 (如私有变量) 则不保存;

  动态链接的模块通常都同时拥有动态链接符号表 (.dynsym) 和静态链接符号表(.symtab)，**后者通常保存了所有目标文件来的符号，而前者只是后者的子集**。

  .dynsym 同样也需要一些辅助表，.dynstr 记录的就是动态链接符号表用到的字符串表; 动态链接符号表比较好理解，这里重做一个动态链接字符串表应该是为了支持 strip，因为 strip 会完全清除文件中目标文件遗留的静态链接符号表 (.symtab) 和静态链接符号表对应的字符串表(.strtab)，如:

![](https://bbs.pediy.com/upload/attach/202112/490870_T4HRPR525D9ZFTQ.jpg)

   对于动态链接来说, 动态链接符号表中保存的是此文件动态链接中的符号定义和引用；

   * **动态链接符号表中定义的符号，**实际上是给其他模块使用的，又称之为当前文件的**导出符号**.

   * **动态链接符号表中引用的符号**，实际上是需要从其他模块获取的符号，由称之为当前文件的**导入符号**.

  readelf -s 同时显示静态和动态链接的符号表:

```
tangyuan@ubuntu:~/compiler_test/gcc_test/x86/test$ readelf -s y.so|grep -A5  Symbol
Symbol table '.dynsym' contains 19 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000000005c8     0 SECTION LOCAL  DEFAULT    9
     2: 0000000000011018     0 SECTION LOCAL  DEFAULT   20
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
--
Symbol table '.symtab' contains 73 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000190     0 SECTION LOCAL  DEFAULT    1
     2: 00000000000001b8     0 SECTION LOCAL  DEFAULT    2
     3: 0000000000000208     0 SECTION LOCAL  DEFAULT    3

```

**10. 动态链接重定位表 (.rela.plt,.rela.dyn)**

  **需要动态链接重定位的原因主要是模块有导入符号的存在** (即动态链接符号表中的未定义符号, 如上)，在编译时这些导入符号的地址是未知的，在运行时才能确定, 故动态链接需要在运行时修正这些符号, 即动态链接的重定位 (**链接与重定位的区别在于链接包括空间分配，符号决议和重定位**).

  **地址无关代码并不能解决模块需要重定位的本质**（本质是当前模块有未定义符号），故**是否采用 PIC 编译，并不会影响一个模块是否需要动态链接重定位**，但 PIC 可以保证代码段 (只读段) 不需要重定位(通过相对引用将重定位转移到数据段)，而**数据段 (包括. data/.got) 在运行时的重定位还是不可避免的**。

  在动态链接中，只有唯二的动态链接重定位表:

  * .rela.plt: 记录. plt 段对应的. got 或. got.plt(具体是哪个取决于是否开启延迟绑定) 的动态链接重定位表项

    其相当于代表所有全局函数调用的重定位

  * .rela.dyn: 记录非. plt 段的数据引用修正

    除了. plt 之外其他段的动态链接重定位修正都记录在这里，若开了 PIC，那么所有的引用修正都应该位于. got 段和数据段.

**参考资料:**

1. [https://www.cs.stevens.edu/~jschauma/631/elf.html](https://www.cs.stevens.edu/~jschauma/631/elf.html) //elf 文件格式信息, ELF 的更多资料可参考 LSB(Linux Standard Base) 手册

2. [GOT and PLT for pwning. · System Overlord](https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html "                  GOT and PLT for pwning. · System Overlord            ")

3. [matplotlib - .plt .plt.got what is different? - Stack Overflow](https://stackoverflow.com/questions/58076539/plt-plt-got-what-is-different)

[【看雪培训】目录重大更新！《安卓高级研修班》2022 年春季班开始招生！](https://bbs.pediy.com/thread-271992.htm)

最后于 2022-2-19 15:32 被 ashimida 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm)