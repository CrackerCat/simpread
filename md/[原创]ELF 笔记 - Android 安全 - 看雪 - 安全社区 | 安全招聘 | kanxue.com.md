> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287319.htm)

> [原创]ELF 笔记

elf 文件格式 (分析的 64 位, 32 位有的地方宽度不一样)
----------------------------------

我怕自己讲的不清楚, 有的地方难免啰嗦, 请您谅解。笔者也是小白一个, 如有错误, 还请您费些口舌指出, 感谢。

### 为什么学

笔者也是看了这篇文章才决定重新细致的学一遍 elf: https://bbs.kanxue.com/thread-287182.htm, 当我们在逆向一个 elf 时, 首先脑海里要有整个文件的布局, 随着不断分析, 这个布局越来越清晰, 可能这才是正确的路线？

### 一些补充概念 (遇到不懂的概念, 没准这里会解你燃眉之急)

*   大端序、小端序: 地址从左到右是由低到高, 而如果把我们平时用的阿拉伯数字放进来用地址衡量, 那么正好与地址高低的顺序相反, 高位在低地址, 低位在高地址, 我想有些强迫症看到这个结果已经别扭死了, 遂分为两派: 大端序、小端序, 大端序是保持我们平时用阿拉伯数字的习惯, 高位在低地址, 低位在高地址; 小端序则在低位放低地址, 高位放高地址。值得一提的是操作的单位是字节, 如一个十六进制的阿拉伯数字 0x6E0, 大端序: `06 E0`, 小端序: `E0 06`。在 elf 中是小端序, CPU 可直接从低地址读取数据并逐步向高地址处理, 无需调整字节位置。
*   相对虚拟地址 (RVA)、进程虚拟地址 (VA)、文件偏移地址 (FOA): 进程虚拟地址 = 进程的基址 (可能会存在随机基址) + 相对虚拟地址; 文件偏移地址 = 进程虚拟地址 - 所在段的起始的进程虚拟地址 + 所在段的起始的文件偏移地址
*   每个程序有自己的独立进程 (内存) 空间: 每个进程在启动时, 操作系统会利用分页机制为其分配一个​​独立的虚拟进程 (内存) 空间​​。达到只用一份真实的物理内存, 却有无数份进程虚拟内存空间的效果。进程虚拟地址也可以通过分页机制转化为内存中真实的物理地址。
*   mmap: 用于将文件或设备直接映射到进程的虚拟地址空间, 参数解析:
    
    ```
    #include void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset); 
    ```
    
    *   addr: 建议映射起始地址, 通常设为 NULL(由内核自动分配), 非 NULL 时**必须页对齐** (通常是 4kb)
        
    *   length: 映射区大小 (字节), 内核会自动按页大小向上取整后在映射 (假设页大小为 4kb, length 传入的 5, 那么仍会映射 4kb 的空间, 因为内核是以页为粒度管理内存的)
        
    *   prot: 内存保护标志, PROT_READ(可读)、PROT_WRITE(可写)、PROT_EXEC(可执行)
        
    *   flags: 映射类型, MAP_SHARED(共享)、MAP_PRIVATE(私有)、MAP_ANONYMOUS(匿名)、MAP_FIXED(强制要求映射必须从 addr 参数指定的地址开始)
        
    *   fd: 文件描述符 (匿名映射时设为 -1)
        
    *   offset: 文件偏移地址量, **必须页对齐**
        
    *   **注意**: 有这样一种情况 `mmap` 可能超出你的预料: (举例) 文件大小为 5 字节, 按页对齐向上取整为 0x1000 字节, 但是设置的 `length` 为 0x2000(一个大于 0x1000 的数) 字节, 会成功分配, 但在这 0x2000 字节大小的内存空间, 前 0x1000 组成为: 原本文件的 5 字节 + 用 0 填充到 0x1000 字节, 之后的部分 (这里就是剩下的 0x1000 字节) 并不会给予分页, 只会占据这部分进程的虚拟内存, 不在允许再被分配, 而当你访问这段内存时会发生错误。
        
*   PLT: 过程链接表, 位于. plt 节, 包含​​跳转代码​​(非数据表), 用于实现延迟绑定 (Lazy Binding), 首次调用外部函数时, 触发 _dl_runtime_resolve 解析符号地址并回填 GOT<table><thead><tr><th>表项</th><th>用途</th></tr></thead><tbody><tr><td>PLT[0]</td><td>公共桩代码, 调用 _dl_runtime_resolve</td></tr><tr><td>​​PLT[1..n]</td><td>每个函数独立的桩代码 (如 puts@plt), 首次调用时跳转至解析流程</td></tr></tbody></table>
*   GOT: 全局偏移表, 位于. got(变量) 和. got.plt(函数), 存储​​外部符号的实际地址​​, 分为两部分:
    *   .got: 存储全局变量地址 (启动时填充)
    *   .got.plt: 存储函数地址 (首次调用时填充)<table><thead><tr><th>表项</th><th>用途</th></tr></thead><tbody><tr><td>GOT[0]</td><td>.dynamic 段地址 (动态链接信息)</td></tr><tr><td>GOT[1]</td><td>link_map 结构地址 (已加载库的元数据)</td></tr><tr><td>GOT[2]</td><td>_dl_runtime_resolve 函数地址 (符号解析入口)</td></tr><tr><td>GOT[3..n]</td><td>存储函数实际地址 (如 printf@got.plt)</td></tr></tbody></table>
*   program_header_table: 程序头表, 定义了​​运行时如何将文件映射到内存​​, 包含多个 program header 条目, 可执行文件必须包含程序头表, 否则无法运行；而可重定位文件 (如. o) 通常不包含该表
*   section_header_table: 节区表, ​​静态分析工具 (如 readelf、objdump) 的参考依据​​, 用于描述文件的节 (Section) 信息(如. text、.data、.rel.dyn 等)。这些信息在​​运行时不被动态链接器使用​​, 甚至可被剥离以减小文件体积

### 整体梳理 (只写了我觉得有用的, 一些没列出的字段, 可能写什么都不会有影响, 结合 010editor 食用)

#### elf_header

elf_header 是 elf 文件最开始的结构, 让我们来看看里面包含了哪些信息:

*   file_identification 魔数, 在文件的最开始, 固定为 `7F 45 4C 46`, 表明这是一个 ELF 文件
*   e_type: 文件类型, 可执行文件或 .so 文件都是 `03 00` 即 ET_DYN(3)
*   e_machine: 目标机器架构
*   e_entry_START_ADDRESS: 程序入口点的相对虚拟地址
*   e_phoff_PROGRAM_HEADER_OFFSET_IN_FILE: 下一个结构 (program_header_table) 的文件偏移地址(当解析下一个结构时, 获取到那个结构的偏移, 不然找不到, 这好像是一句废话......)。
*   e_phentsize_PROGRAM_HEADER_ENTRY_SIZE_IN_FILE: 下一个结构 (program_header_table) 中每一项多大, `program_header_table` 是一个数组
*   e_phnum_NUMBER_OF_PROGRAM_HEADER_ENTRIES: 下一个结构 `program_header_table` 数组长度

#### program_header_table

`program_header_table`(程序头表) 是一个数组, 每一项都描述了一段内存和文件的对应关系, 看看 `program_header_table` 中每一项的属性:

*   p_type: 段类型, 只有属性为 `PT_LOAD(1)` 的段会被直接加载到内存中, 常见类型包括:
    *   PT_NULL(0): 无效条目, 用于标记结束或占位。
    *   PT_LOAD(1): 可加载段​​(代码 / 数据), **需映射到内存**。
    *   PT_DYNAMIC(2): 动态链接信息 (如依赖库列表、重定位表)。
    *   PT_INTERP(3): ​​解释器路径​​(如 /lib/ld-linux.so.2), 用于动态加载。
    *   PT_NOTE(4): 辅助信息 (如调试信息)。
    *   PT_PHDR(6): `program_header_table`自身的位置和大小​​。
    *   PT_TLS(7): 线程局部存储 (Thread-Local Storage) 段。
    *   PT_GNU_RELRO(0x6474e552): ​​将动态重定位后的内存区域设为只读, 重定位完成后, PT_GNU_RELRO 标记的区域会被设置为只读权限, 阻止后续写入操作, 防范攻击者篡改函数指针或全局变量。
*   p_flags: 定义段在内存中的访问权限, 通过位掩码组合控制内存页的读 (Read)、写(Write)、执行(Execute) 权限(一般来说, 代码段可读可执行, 数据段可读可写), 除了可加载的段即 `p_type` = `PT_LOAD(1)`, 其他段 `p_flags` 属性是没有作用的, 可能的值:
    *   PF_None(0): 无权限
    *   PF_Exec(1): 可执行
    *   PF_Write(2): 可写
    *   PF_Write_Exec(3): `PF_Exec(1) | PF_Write(2)` 可写可执行
    *   PF_Read(4): 可读
    *   PF_Read_Exec(5): `PF_Read(4) | PF_Exec(1)` 可读可执行
    *   PF_Read_Write(6): `PF_Read(4) | PF_Write(2)` 可读可写
    *   PF_Read_Write_Exec(7): `PF_Read(4) | PF_Write(2) | PF_Exec(1)` 可读可写可执行
*   p_offset_FROM_FILE_BEGIN: 文件偏移地址 (从文件开头计算的偏移量)
*   p_vaddr_VIRTUAL_ADDRESS: 该段起始的相对虚拟地址
*   : 该段在物理内存中的预期加载地址, 在​​无虚拟内存管理 (MMU)​​ 的嵌入式系统或实时操作系统(RTOS) 被使用
*   p_filesz_SEGMENT_FILE_LENGTH: 该段在 ​​ELF 文件​​ 中的实际数据长度 (单位: 字节)
*   p_memsz_SEGMENT_RAM_LENGTH: 该段在 `​​进程内存​​` 中需占用的总空间长度 (单位: 字节)

在 `program_header_table` 数组中, 在所有可加载段 (`p_type` = `PT_LOAD(1)`) 前, 必须要有 `Program Header`(`p_type` = `PT_PHDR(6)`) 和 `Interpreter Path`(`p_type` = `PT_INTERP(3)`) 段:

*   `Program Header`: 在这个段中 `p_paddr_PHYSICAL_ADDRESS` 与 `p_vaddr_VIRTUAL_ADDRESS` 与 `elf_header` 中的 `e_phoff_PROGRAM_HEADER_OFFSET_IN_FILE` 相同
*   `Interpreter Path`: 记录了解析加载这个 elf 文件程序的路径, 根据文件偏移地址 `p_offset_FROM_FILE_BEGIN` 找到字符串, 类似 `/lib64/ld-linux-x86-64.so.2`, 可以修改成你自己的加载器路径。执行流程为: 操作系统创建新进程后, 会根据 `Interpreter Path` 段中记录的路径找到加载器, 并将其将其加载到内存, 这个加载器会读取 elf 文件的中导入表, 将 elf 用到的模块加载到内存......

在加载 `Loadable Segment`(`p_type` = `PT_LOAD(1)`) 的段时, 将段利用 mmap 映射到内存的流程: 根据文件基址 (base)、相对虚拟地址(p_vaddr_VIRTUAL_ADDRESS) 算出进程虚拟地址(base + p_vaddr_VIRTUAL_ADDRESS), 进程虚拟地址向下对页大小取整获得要用 mmap 映射的起始地址 `pbase`, 而 mmap 的第二个参数 `length` 赋值为 `base + p_vaddr_VIRTUAL_ADDRESS - pbase + p_filesz_SEGMENT_FILE_LENGTH`(之后用 `len1` 代替) 进行一次映射, 此时段的数据内容是从 `base + p_vaddr_VIRTUAL_ADDRESS` 开始的, 而 `pbase` 到 `base + p_vaddr_VIRTUAL_ADDRESS` 之前文件映射但​​无有效内容​, 访问可能触发错误。

然后再计算 `base + p_vaddr_VIRTUAL_ADDRESS - pbase + p_memsz_SEGMENT_RAM_LENGTH`(之后用 `len2` 代替), 如果 `len1` 按页大小向上取整的值 (称为 `len1_`) 比 `len2` 按页大小向上取整的值 (称为 `len2_`) 小, 那么此时在进行一次 mmap, 映射的起始地址为 `len1_`, `length` 为 `len2_ - len1_`, 并且 `flags` 要多设置上一个 `MAP_ANONYMOUS` 属性, fd 赋值为 -1, 表示没有实际的文件与之对应。(为什么多进行一次 mmap 可以看看上边 mmap 中注意的内容, 如果只进行一次 mmap, 直接按照 `length` 为 `len2_` 映射会在访问超出 `len1_` 时发生错误)。

对照着图看可能会好一点:  
![](https://bbs.kanxue.com/upload/attach/202506/1025255_8MFV6H245K79M65.webp)  
也可以对照表格 (有些变量对照着上面的文字看, 写全表达式太长了):

<table><thead><tr><th>地址范围</th><th>内容状态</th><th>访问行为</th></tr></thead><tbody><tr><td><code>[pbase, base + p_vaddr_VIRTUAL_ADDRESS)</code></td><td>文件映射但<strong>无有效内容</strong></td><td>访问可能触发 <code>SIGBUS</code> 错误</td></tr><tr><td><code>[base + p_vaddr_VIRTUAL_ADDRESS, len1 + pbase)</code></td><td><strong>文件内容</strong> (有效数据)</td><td>正常读取文件数据</td></tr><tr><td><code>[len1 + pbase, align_up(pbase + len2, PAGE_SIZE))</code></td><td><strong>匿名映射</strong> (初始化为 0)</td><td>读取到全 0</td></tr></tbody></table>

将 `Loadable Segment` 段内容映射到内存后, 也同时包含了其他段描述的内容, 自此, 我们不需要在将文件的内容向内存映射。此时我们获得任意一个相对虚拟地址都可以直接用 `base + 相对虚拟地址` 得到进程虚拟地址

在 `Dynamic Segment`(`p_type` = `PT_DYNAMIC(2)`) 段的内容已经在加载 `Loadable Segment` 时包含了, 已经加载到了内存, 所以直接获取 `Dynamic Segment` 的 `p_vaddr_VIRTUAL_ADDRESS`, 算出进程虚拟地址就可以访问 `Dynamic Segment` 的内容 (下文称为 `tags`) 了, 这里的 `tags` 是一个结构体数组, 结构体大小为 16 字节, 前 8 字节代表类型, 后 8 字节是相对虚拟地址或者整数值 (32 位 elf 大小不同, 结构相同):

```
typedef struct {
    Elf64_Sxword  d_tag;      /* 8 字节: 动态条目类型标识 */
    union {
        Elf64_Xword d_val;    /* 8 字节: 整数值(如大小、标志等) */
        Elf64_Addr  d_ptr;    /* 8 字节: 相对虚拟地址(指向符号表、字符串表等) */
    } d_un;               
} Elf64_Dyn;
```

`d_tag` 的常见类型有:

<table><thead><tr><th>d_tag 宏名</th><th>值</th><th>d_un 解释</th><th>用途</th></tr></thead><tbody><tr><td><code>DT_NULL</code></td><td><code>0x0</code></td><td>忽略</td><td>动态段结束标记</td></tr><tr><td><code>DT_STRTAB</code></td><td><code>0x5</code></td><td><code>d_ptr</code>: 相对虚拟地址</td><td>字符串表 (<code>.dynstr</code>) 地址</td></tr><tr><td><code>DT_NEEDED</code></td><td><code>0x1</code></td><td><code>d_val</code>: 字符串表偏移</td><td>依赖库名 (如 <code>libc.so.6</code>)</td></tr><tr><td><code>DT_SYMTAB</code></td><td><code>0x6</code></td><td><code>d_ptr</code>: 相对虚拟地址</td><td>符号表 (<code>.dynsym</code>) 地址</td></tr><tr><td><code>DT_PLTRELSZ</code></td><td><code>0x2</code></td><td><code>d_val</code>: 字节大小</td><td>PLT(Procedure Linkage Table) 重定位表总大小</td></tr><tr><td><code>DT_JMPREL</code></td><td><code>0x17</code></td><td><code>d_ptr</code>: 相对虚拟地址</td><td>PLT 重定位表 (<code>.rela.plt</code>) 地址</td></tr><tr><td><code>DT_REL</code></td><td><code>0x11</code></td><td><code>d_ptr</code>: 相对虚拟地址</td><td>重定位表 (<code>.rel.dyn</code>) 地址</td></tr><tr><td><code>DT_GNU_HASH</code></td><td><code>0x6ffffef5</code></td><td><code>d_ptr</code>: 相对虚拟地址</td><td>​​GNU 扩展哈希表 (.gnu.hash) 地址</td></tr></tbody></table>

*   结束, `d_tag` = `DT_NULL`: 标志着 `tags` 结束。
*   字符串表, `d_tag` = `DT_STRTAB`: 字符串表存储了一堆以 '\0' 结尾的字符串。 找到字符串表 (`.dynstr`):
    *   文件中查看: 计算 `d_ptr` 相对虚拟地址对应的文件偏移地址: `d_ptr` - 所在段的起始的相对虚拟地址 + 所在段的起始的文件偏移地址。
    *   内存中查看: 计算 `d_ptr` 相对虚拟地址对应的进程虚拟地址: `base` + `d_ptr` (如果忘了的话可以看看上边 `Loadable Segment` 那里)
    *   在文件尾部的结构 `section_header_table` 中也保存了 `.dynstr` 等的地址, 但是不作数, 可以随意篡改
*   依赖库名, `d_tag` = `DT_NEEDED`: 每个都代表了一个依赖库名。 要结合字符串表 (`.dynstr`) 一起用, `d_val` 也就是这个项的后 8 字节记录的在字符串表 (`.dynstr`) 中的偏移, 要先找到字符串表 (`.dynstr`) 的地址在加上偏移才是对应的依赖库名字符串的开始, 遇到 '\0' 结束。
*   符号表, `d_tag` = `DT_SYMTAB`: 存储了程序在动态链接时需要的符号 (如函数名、变量名) 及其对应地址等信息。找到符号表(`.dynsym`) 与找到字符串表 (`.dynstr`) 三种方式相同, 不在赘述。符号表 (`.dynsym`) 也是一个结构体数组, 结构体如下 (32 位 elf 大小不同, 字段顺序不同, 组成一样):
    
    ```
    typedef struct {
        Elf64_Word  st_name;    // 4 字节
        unsigned char st_info;  // 1 字节
        unsigned char st_other; // 1 字节
        Elf64_Half  st_shndx;   // 2 字节
        Elf64_Addr  st_value;   // 8 字节
        Elf64_Xword st_size;    // 8 字节
    } Elf64_Sym;
    ```
    
    *   st_name: 符号名在字符串表中的偏移
        
    *   st_info: 符号类型和绑定信息
        
        *   高 4 位 (bit 7-4): 绑定类型, 常见取值:
            *   STB_LOCAL(0): 局部符号
            *   STB_GLOBAL(1): 全局符号
            *   STB_WEAK(2): 弱引用
        *   低 4 位 (bit 3-0): 符号类型, 常见取值:
            *   STT_NOTYPE(0): 未指定类型
            *   STT_OBJECT(1): 变量
            *   STT_FUNC(2): 函数
            *   STT_SECTION(3): 段符号
            *   STT_FILE(4): 文件名符号
    *   st_other: 未定义的含义, 通常为 0
        
    *   st_shndx: 符号所属段在 `section_header_table`(段头表) 中的索引
        
    *   st_value:
        
        *   在可执行文件或共享库 (.so) 中: 符号对应变量的相对虚拟地址。
        *   可重定位文件 (.o): 符号在节内的偏移量
    *   st_size: 符号大小 (如函数代码长度或变量占用字节数), 若大小未知或未定义, 值为 0(如未初始化符号)。
        
    *   **导出**: 导出的符号需要满足:
        
        *   ​​全局性: st_info 绑定类型为 STB_GLOBAL 或 STB_WEAK
        *   类型匹配: st_info 符号类型为 STT_FUNC(函数) 或 STT_OBJECT(变量)
        *   已定义: st_shndx != SHN_UNDEF
*   PLT 重定位表总大小, `d_tag` = `DT_PLTRELSZ`: `d_val` 记录了 PLT 重定位表所占字节数, 在解析 PLT 重定位表要先获取它的大小。
*   PLT 重定位表, `d_tag` = `DT_JMPREL`: PLT 重定位表 (.rela.plt) 的作用是支持延迟绑定机制:
    
    *   程序启动时: 外部函数调用会先跳转到 PLT(`section_header_table` 中的 `.plt` 段) 中对应的入口。
    *   PLT 入口的初始行为: 首次调用时, PLT 会跳转到 PLT[0], 这是一段公共代码, 负责调用动态链接器的符号解析函数
    *   符号解析过程: 动态链接器通过。rela.plt 中的重定位信息 (符号索引、类型等), 查找函数的实际地址, 并将其写入 GOT(Global Offset Table)。
    *   后续调用优化: 首次解析后, GOT 中已存储实际地址, 后续调用直接通过 GOT 跳转, 无需再次解析。
    
    ```
    typedef struct {
        Elf64_Addr    r_offset;    // 8字节
        Elf64_Xword   r_info;      // 8字节
        Elf64_Sxword  r_addend;    // 8字节
    } Elf64_Rela;
    ```
    
    *   r_offset: 指定需要被修改的内存位置。该地址是​​待修正值的存储位置​​。
    *   r_info: 高 32 位是符号表索引, 在 .dynsym(动态符号表) 中的索引, 指向 Elf64_Sym 结构体; 低 32 位是重定位类型, 决定如何计算新地址。常见类型 (S = 符号地址, A = r_addend, B = 共享库基址, P = 被修正位置地址):<table><thead><tr><th>类型</th><th>数值</th><th>计算方式</th><th>用途</th></tr></thead><tbody><tr><td><strong>R_X86_64_64</strong></td><td><code>0x1</code></td><td><code>S + A</code></td><td>绝对地址重定位</td></tr><tr><td><strong>R_X86_64_RELATIVE</strong></td><td><code>0x8</code></td><td><code>B + A</code></td><td>基址重定位 (数据段)</td></tr><tr><td><strong>R_X86_64_JUMP_SLOT</strong></td><td><code>0x7</code></td><td><code>S</code></td><td>PLT/GOT 函数解析</td></tr><tr><td><strong>R_X86_64_PC32</strong></td><td><code>0x2</code></td><td><code>S + A - P</code></td><td>相对地址重定位</td></tr></tbody></table>
    *   r_addend: 在计算新地址时作为​​附加常量​​参与运算, 通常用于调整偏移 (如数组访问、结构体成员), 多数函数重定位(如 R_X86_64_JUMP_SLOT) 中为 0
*   重定位表, `d_tag` = `DT_REL`: 存储程序启动时需​​立即修正的全局变量和静态数据​​的重定位信息。与 PLT 重定位表不同, 重定位表在程序启动时解析此表, 直接修改 GOT 或数据段中的地址 (​​无延迟绑定​​)。
    
    ```
    typedef struct {
        Elf64_Addr    r_offset;   // 8字节
        Elf64_Xword   r_info;     // 8字节
    } Elf64_Rel;
    ```
    
    *   r_offset: 需修正的目标地址 (如 GOT 表项地址)
    *   r_info: 高 32 位: 符号在 .dynsym 中的索引；低 32 位: 重定位类型, 常见重定位类型:<table><thead><tr><th>类型</th><th>数值</th><th>计算方式</th><th>用途</th></tr></thead><tbody><tr><td><strong>R_X86_64_RELATIVE</strong></td><td><code>0x8</code></td><td><code>B + A</code></td><td>基址重定位</td></tr><tr><td><strong>R_X86_64_GLOB_DAT</strong></td><td><code>0x6</code></td><td><code>S</code></td><td>全局变量地址修正</td></tr><tr><td><strong>R_X86_64_COPY</strong></td><td><code>0x5</code></td><td>-</td><td>复制符号到数据段</td></tr><tr><td><strong>R_X86_64_32</strong></td><td><code>0x10</code></td><td><code>S + A</code></td><td>绝对地址重定位</td></tr></tbody></table>
*   哈希表, `d_tag` = `DT_GNU_HASH`: 扩展的​​符号哈希表​​, 用于​​加速动态链接时的符号查找​​。相比老版本的 DT_HASH, 它在结构和算法上进行了优化, 尤其适合大型共享库或程序, 被使用在新版中。

### 参考资料

https://space.bilibili.com/37877654/lists/1467282?type=series

一些博客, AI

### 后续工作

注释一遍源码, 魔改 / 实现一个 linker

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 21 小时前 被 nothing233 编辑 ，原因：

[#基础理论](forum-161-1-117.htm)