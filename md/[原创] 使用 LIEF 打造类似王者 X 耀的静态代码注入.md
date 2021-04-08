> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266762.htm)

使用 lief 工具写一个类似于腾讯 so 保护这样一个东西。本篇记录 lief 工具的使用和 TX 加固的原理，不会对 ELF 文件格式进行记录。

[](#一、工具介绍)一、工具介绍
-----------------

首先，lief 是一个文件解析工具，可以解析 ELF、PE、DEX 等，用途比较广泛。

 

github 地址：https://github.com/lief-project/LIEF

 

我使用的是 python 版，在使用过程中经常出现毛病，所以并不建议使用 python 版。最好是不怕麻烦自己写一个，就可以避免很多不稳定导致的 fix 代码。

二、TX SO 加密介绍
------------

首先，王者荣耀采用的是 il2cpp，比较关键的 so 有 3 个，libil2cpp.so、libGameCore.so、libtprt.so。其中 libtprt.so 为保护 so。

 

王者荣耀加载被保护 so（il2cpp 或 GameCore）时，会优先加载链接库。所以，整个流程 unity 启动后，会优先加载 libtprt.so。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_MPHRXJ9M3MTHUWJ.png)

 

libtprt.so 对自身的 tptext 节进行了加密，在加载过程中执行 init_array 进行解密，跟踪 mprotect，可以看到解密代码：

 

![](https://bbs.pediy.com/upload/attach/202103/833877_8YQHZSZFREFEP4R.png)

 

![](https://bbs.pediy.com/upload/attach/202103/833877_4HBW6DVY2GPB65X.png)

 

使用 lief 快速解密：

```
tprt = lief.parse(name)
tptext_section = tprt.get_section(".tptext").content
 
print(len(tptext_section))
offset = tprt.get_section(".tptext").offset
out = b""
for i in range(len(tptext_section)):
    out += (tptext_section[i] ^ 0xb8).to_bytes(1,byteorder='little')
 
 
print(len(out))
tpp = tprt_bin[:offset]+out+tprt_bin[offset+len(tptext_section):]
with open("tpp.so", 'wb') as fp:
    fp.write(tpp)

```

![](https://bbs.pediy.com/upload/attach/202103/833877_NMTGQ79VGMJST3J.png)

 

tptext 节里的函数主要用于初始化各类保护，会被 jni_onload 调用。现在 tprt 已经被成功加载，现在就需要解密 libil2cpp.so

 

由于 libGameCore.so 小一点，分析起来方便，所以后面就不分析 il2cpp 了。先观察 ELF 结构，可以发现它的 LOAD 段格外多。

 

个人分析他比原生多了 3 个 LOAD 段，新增的第一个 LOAD 段（04），里面是. dynamic 和 .init_array。很好理解，因为强行添加了一个依赖库和 init_array，所以需要生成新的. dynamic，至于 init_array 个人猜测是添加字符串混淆之类的东西。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_AF77K2YW2GQDVP8.png)

 

新增的第二个 LOAD 段（05），是新的. rel.plt，用 ida 分析，可以明显看出有增加项，且增加项在新增的第三个 LOAD 段（06）：

 

![](https://bbs.pediy.com/upload/attach/202103/833877_PSPXGF4J46FMPBX.png)

 

跳转到 0x228cfcc，可以看到这是一个 got 表，并且它的 plt 表在新增的第二个 LOAD 段（05）。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_5EYTCBPQVUBK8KZ.png)

 

![](https://bbs.pediy.com/upload/attach/202103/833877_PEWXYBNQ6YR4XCH.png)

 

![](https://bbs.pediy.com/upload/attach/202103/833877_NUVYT86NZFUQD3U.png)

 

对添加的 LOAD 段有一定了解了，再看一下修改后的. dynamic 有什么变化：

 

![](https://bbs.pediy.com/upload/attach/202103/833877_RN842J8P4MWWF9X.png)

 

需要关注的，我已经添加了红色方框。大部分只是修改偏移，指向新的节（JMPREL、INIT_ARRAY），值得注意的是 DT_INIT，这个是 init 段，比 init_array 更早执行。更适合用来解密。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_94R6T8SYWS7EVTE.png)

 

通过取 off_1cc 的地址 - off_1cc 的值（地址为 base+0x1cc，值是自己设置的，为 1cc）所以算出来是当前 so 的基址，然后调用 sub_2289b58 并且传入基址。

 

sub_2289b58 就是解密函数，网上对它的分析很多了，总的就是调用 g_tprt_pfn_array（“.text”,base,3）对当前的 text 段进行解密。（“.text” 这个字符串，在新增的第三个 LOAD（06）中，意味着是写死的）

 

![](https://bbs.pediy.com/upload/attach/202103/833877_3R4CW5HRV9KJ8A9.png)

 

所以总结一下：

 

MTP 对 SO 新增了 3 个 LOAD 段，第一个 LOAD 新增. dynamic、.init_array，第二个 LOAD 段是映射新的. rel.plt、.plt、.text 节，第三个 LOAD 段用来映射. got、.data 节。

 

MTP 在 program header 之后添加了一段汇编，在 init 时执行，获取当前 SO 的基址，传入解密函数，并调用 libtprt.so 的导出函数 g_tprt_pfn_array 进行 text 节解密。

[](#三、代码实现)三、代码实现
-----------------

由于一次写完大概率会很乱思路不够清晰，所以分几步写，方便理解，也好记录。

### [](#步骤一：添加init段，并调用任意函数)步骤一：添加 INIT 段，并调用任意函数

首先，生成一个 SO，很简单，有一个 PrintLog 函数没有调用，所以 LOG 只有一条：

 

![](https://bbs.pediy.com/upload/attach/202103/833877_FHSPFTEMCYX5WUR.png)

 

![](https://bbs.pediy.com/upload/attach/202103/833877_SXHRYK86K7P78FP.png)

 

执行以下代码：

```
"""
  添加init段，并且调用ADD_FUNC_NAME并传入一个参数（当前So的基址）
"""
INIT_PROC_SIZE = 0x2c
INIT_PROC_CONTENT = [0xC0,0x46,0xFF,0xB5,0x83,0xB0,0x6D,0x46,0x00,0xA3,0x14,0x3B,0x19,0x1C,0x1F,0x68,0xC9,0x1B,0x29,0x60,0x5E,0x68,0x03,0xD0,0x76,0x18,0x6E,0x60,0x28,0x68,0xB0,0x47,0x03,0xB0,0xFF,0xBD]# 腾讯代码里扣出来的嘿嘿
 
def add_init_proc():
 
    binary = lief.parse(TARGET_BIN)
    # 检测.dynamic节的空位是否足够,如果小于3个就要拓展dynamic节内容
    free_dynamic_entry = 0
    for entry in binary.dynamic_entries:
        if entry.tag == lief.ELF.DYNAMIC_TAGS.NULL:
            free_dynamic_entry += 1
    print("free dynamic entry num:", free_dynamic_entry)
    assert free_dynamic_entry > 3
 
    #获取位于segment header 后的 offset，用于添加init_proc
    init_proc_offset = binary.header.program_header_offset + \
                       binary.header.program_header_size * (binary.header.numberof_segments + 2 )
    print("init_proc offset:", hex(init_proc_offset))
 
 
    # 添加init入口
    if not binary.has(ELF.DYNAMIC_TAGS.INIT):
        # 先用0占位,直接写入偏移，lief工具会有点问题
        # 如果不出问题 binary.add(ELF.DynamicEntry(ELF.DYNAMIC_TAGS.INIT,init_proc_offset + 8))
        binary.add(ELF.DynamicEntry(ELF.DYNAMIC_TAGS.INIT,0))
 
    else:
        init_entry = binary.get(ELF.DYNAMIC_TAGS.INIT)
        print("[x] binary has init_proc:", init_entry)
        exit(1)
 
    binary.write("libnative-lib.so")
 
    # 手动修复 DynamicEntry 中的 value
    outbin = lief.parse("libnative-lib.so")
    out_dynamic = outbin.get_section(".dynamic")
    ADD_FUNC = outbin.get_symbol(ADD_FUNC_NAME).value
 
    num = 0
    for entry in outbin.dynamic_entries:
        if entry.tag == ELF.DYNAMIC_TAGS.INIT:
            break
        num+=1
 
    init_entry_offset =  out_dynamic.offset + (num * out_dynamic.entry_size)
    print(hex(init_entry_offset))
 
    patch_file("libnative-lib.so",init_entry_offset+4,struct.pack("
```

![](https://bbs.pediy.com/upload/attach/202103/833877_SKQ8DJ222SEACXA.png)

 

![](https://bbs.pediy.com/upload/attach/202103/833877_236673NWQVB4CX3.png)

 

**可以看到函数 PrintLog 优先于 JNI 函数执行，并且传入的参数为基址。**

### [](#步骤二：向so中添加可执行代码)步骤二：向 so 中添加可执行代码

[关于重定位](https://bbs.pediy.com/thread-221821.htm)

 

需要记住的几个总结：

 

rel.plt ->got -> plt -> extern 表（导入函数）

 

rel.plt ->got -> text（导出）

```
//ELF.h 查看rel.plt的格式，r_offset 指向got表的地址，r_info高8位为类型，后24位为
 
// Relocation entry, without explicit addend.
 struct Elf32_Rel {
   Elf32_Addr r_offset; // Location (file byte offset, or program virtual addr)
   Elf32_Word r_info;   // Symbol table index and type of relocation to apply
 
   // These accessors and mutators correspond to the ELF32_R_SYM, ELF32_R_TYPE,
   // and ELF32_R_INFO macros defined in the ELF specification:
   Elf32_Word getSymbol() const { return (r_info >> 8); }
   unsigned char getType() const { return (unsigned char)(r_info & 0x0ff); }
   void setSymbol(Elf32_Word s) { setSymbolAndType(s, getType()); }
   void setType(unsigned char t) { setSymbolAndType(getSymbol(), t); }
   void setSymbolAndType(Elf32_Word s, unsigned char t) {
     r_info = (s << 8) + t;
   }
 };
 
 
 // Symbol table entries for ELF32.
 struct Elf32_Sym {
   Elf32_Word st_name;     // Symbol name (index into string table)
   Elf32_Addr st_value;    // Value or address associated with the symbol
   Elf32_Word st_size;     // Size of the symbol
   unsigned char st_info;  // Symbol's type and binding attributes
   unsigned char st_other; // Must be zero; reserved
   Elf32_Half st_shndx;    // Which section (header table index) it's defined in
 
   // These accessors and mutators correspond to the ELF32_ST_BIND,
   // ELF32_ST_TYPE, and ELF32_ST_INFO macros defined in the ELF specification:
   unsigned char getBinding() const { return st_info >> 4; }
   unsigned char getType() const { return st_info & 0x0f; }
   void setBinding(unsigned char b) { setBindingAndType(b, getType()); }
   void setType(unsigned char t) { setBindingAndType(getBinding(), t); }
   void setBindingAndType(unsigned char b, unsigned char t) {
     st_info = (b << 4) + (t & 0x0f);
   }
 };

```

编译一个 so，里面的示范代码：

 

![](https://bbs.pediy.com/upload/attach/202103/833877_TCMWHA54C5UN764.png)

 

十分简单，就是一个打印 log，将编译好的 so 拖入 ida，可以看到，虽然我只写了一个函数，实际上 so 运行时还需要许多其他的函数。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_BRA8NMDWUGEPKKF.png)

 

执行以下代码：

```
def add_symbol():
    binary = lief.parse(TARGET_BIN)
    add_bin = lief.parse("libadd.so")
    add_got = add_bin.get_section(".got")
    add_data = add_bin.get_section(".data")
    add_plt = add_bin.get_section(".plt")
    add_text = add_bin.get_section(".text")
 
 
 
    before_add_load_num = 0
    for i in binary.segments:
        if i.type == ELF.SEGMENT_TYPES.LOAD:
            before_add_load_num+=1
    print(before_add_load_num)
 
 
    """
    添加2个load段，用于将add中内容添加进target
    """
    # 第一个可读可执行,用来映射新的.plt节 + .text节
    add_RE_seg = lief.ELF.Segment()
    add_RE_seg.alignment = 0x1000
    add_RE_seg.type = ELF.SEGMENT_TYPES.LOAD
    add_RE_seg.add(ELF.SEGMENT_FLAGS.X)
    add_RE_seg.add(ELF.SEGMENT_FLAGS.R)
    add_RE_seg.content = add_plt.content + add_text.content
    print("add_RE_seg.content :", add_RE_seg.content)
    binary.add(add_RE_seg)
 
 
    #第二个load段 可读可写 添加 .got .data
    add_RW_seg = lief.ELF.Segment()
    add_RW_seg.alignment = 0x1000
    add_RW_seg.type = ELF.SEGMENT_TYPES.LOAD
    add_RW_seg.add(ELF.SEGMENT_FLAGS.W)
    add_RW_seg.add(ELF.SEGMENT_FLAGS.R)
    add_RW_seg.content = add_got.content + add_data.content
    binary.add(add_RW_seg)
    print(add_got.content)
 
 
 
    addbin_relplt = add_bin.pltgot_relocations
 
    """
        添加addbin中的relplt，并且向dynsym添加对应的symbol。
        此操作改动了dynstr、dynsym、rel.plt。
        本来针对原节拓展即可，但lief工具新增了3个load段进行加载新的内容，所以还修改了dynamic
 
    """
    add_sym_value = list()
    for add_entry in addbin_relplt:
        if binary.has_symbol(add_entry.symbol.name):
            sym = binary.get_symbol(add_entry.symbol.name)
        else:
            sym = binary.add_dynamic_symbol(add_entry.symbol)#后面修复
            add_sym_value.append(add_entry.symbol.value - add_text.virtual_address)
            print(hex(add_entry.symbol.value),hex(add_entry.symbol.value - add_text.virtual_address))
 
            # 此处lief工具又有问题，写入后value变了，坑爹货
            #工具不出问题，此处减掉add中text的虚拟地址，加上intermediate中的新增的text虚拟地址就行了
 
            add_reloc = ELF.Relocation()
            add_reloc.type = add_entry.type
            add_reloc.symbol = sym
            add_reloc.address = add_entry.address - add_got.virtual_address
            add_reloc.purpose = ELF.RELOCATION_PURPOSES.PLTGOT
            add_reloc = binary.add_pltgot_relocation(add_reloc)
            # print("add_reloc - ", add_reloc)
 
 
    binary.write("intermediate.so")
    #辣鸡工具，会导致偏移出问题，所以不得不进行手工修复
    inter = lief.parse("intermediate.so")
    #获取前面添加的两个load段的虚拟地址
    add_RE_seg_virtual_address = 0
    add_RW_seg_virtual_address = 0
 
    after_add_load_num = 0
    for i in binary.segments:
        if i.type == ELF.SEGMENT_TYPES.LOAD:
            after_add_load_num += 1
            if after_add_load_num > before_add_load_num and after_add_load_num <= before_add_load_num +2:
                if i.has(ELF.SEGMENT_FLAGS.X):
                    add_RE_seg_virtual_address = i.virtual_address
                if i.has(ELF.SEGMENT_FLAGS.W):
                    add_RW_seg_virtual_address = i.virtual_address
 
    print(hex(add_RE_seg_virtual_address))
    print(hex(add_RW_seg_virtual_address))
 
 
    #修复dynsym中新增的symbol，将Elf32_Sym->st_value 指向text段即可，0不修改是指向import func
    new_dynsym_content = inter.get_section(".dynsym").content
    add_dynsym_start = len(new_dynsym_content) - len(add_sym_value)*16
    print("add_dynsym_start:",add_dynsym_start)
 
    modify_dynsym_content = []
    inx = 0
    for entry_content in [new_dynsym_content[i:i + 16] for i in range(add_dynsym_start, len(new_dynsym_content), 16)]:
        entry = DynSymEntry.parse_from_content(entry_content)
        if(entry.sym_value != 0):
            print(hex(entry.sym_value))
            entry.sym_value =add_sym_value[inx] +add_RE_seg_virtual_address +len(add_plt.content)
            print(hex(entry.sym_value))
        inx += 1
        modify_dynsym_content += entry.content
    patch_file("intermediate.so", inter.get_section(".dynsym").offset + add_dynsym_start, modify_dynsym_content)
 
    #修复.rel.plt中新增的rel项，指向新增的第二个load段中加载的add.so中的got表
    modify_rel_content = []
    relplt = binary.get_section(".rel.plt")
    add_rel_start = binary.get_section(".rel.plt").size - len(add_sym_value)*8
    print("add_rel_start :", hex(add_rel_start))
 
    add_entry_ndx = 0
    for rel_content in [relplt.content[i:i + 8] for i in range(add_rel_start, len(relplt.content), 8)]:
        rel = RelEntry.parse_from_content(rel_content)
        if(rel.offset != 0):
            print("offset :", hex(rel.offset))
            rel.offset = rel.offset + add_RW_seg_virtual_address
            print("offset :", hex(rel.offset))
        modify_rel_content += rel.content
        add_entry_ndx += 1
    patch_file("intermediate.so", inter.get_section(".rel.plt").offset + add_rel_start, modify_rel_content)

```

将 intermediate.so 改名为 libnative-lib.so, 在 apk 里能成功执行。不报错就是胜利！

 

ida 打开 intermediate.so 可以看到相关的函数，虽然把 plt 解析成了函数，不过不影响。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_9GM8TYAF2UXHREA.png)

 

再对比 libadd.so，可以看到，函数内容是一致的：

 

![](https://bbs.pediy.com/upload/attach/202103/833877_7FCKTXDWJQW758E.png)

 

![](https://bbs.pediy.com/upload/attach/202103/833877_DTYYSJE8NSP5BWJ.png)

 

**成功运行不报错~**

### [](#结合步骤一和步骤二，让添加的decrypt函数执行起来)结合步骤一和步骤二，让添加的 decrypt 函数执行起来

有两种方式，TX 是让 plt-》got 表之间的偏移不变，就不需要修改 plt 表。

 

如果修改了 plt、got 之间的偏移，就需要通过修改字节码来实现正常运行。

 

首先可以看到 plt 表有三条指令，获取当前地址，添加偏移，跳转。

 

![](https://bbs.pediy.com/upload/attach/202103/833877_NN228YMT9FYTB2J.png)

 

第一个红框，由于是 0 不好理解，我换成 0x01. 那么就是取当前地址 + 0x0100000

 

第二个红框，0x62，向 r12 添加 0x62000

 

第三个红框，0xf00c，通过 a8028 - a801c = c 。所以低 12 位为添加的偏移 -》0x00c、0x1fc

 

由于其他代码与之前的相同，就没必要重复粘贴了。将新增的 plt 表修复相关代码粘贴出来。

```
# 修复plt表的相关代码
def get_offset(inaddr):
    high = inaddr // 0x0100000
    mid = (inaddr & 0xfffff)//0x1000
    low = (inaddr & 0xfff)
    return high,mid,low
 
#需要fix plt 表,调用外部函数时需要 通过plt表进行跳转
    add_plt_size = add_plt.size
    print("add_plt_size:",add_plt_size)
    PLT_TABLE_HEAD_LEN = 0x14
    need_fix_plt_content = add_RE_seg.content[PLT_TABLE_HEAD_LEN:add_plt_size]
    print(need_fix_plt_content)
 
    print(hex(add_RW_seg_virtual_address - add_RE_seg_virtual_address))
 
 
    inx = 0
    modify_plt_content = []
    for plt_entry in [add_RE_seg.content[i:i+12] for i in range(PLT_TABLE_HEAD_LEN,add_plt_size,12)]:
        #+8是因为ADR取地址是取PC的值
        got2plt_offset = got_address_list[inx] - (inx*12+add_RE_seg_virtual_address+8 +PLT_TABLE_HEAD_LEN )
        inx += 1
        print("got -> plt offset :", hex(got2plt_offset))
        print(plt_entry)
        h,m,l = get_offset(got2plt_offset)
        plt_entry[0] = h
        plt_entry[4] = m
        plt_entry[8] = l & 0xff
        plt_entry[9] = (plt_entry[9]&0xf0) + (l >> 8)
        print("fix entry",plt_entry,hex(plt_entry[9]))
        modify_plt_content += plt_entry
    patch_file("intermediate.so", add_RE_seg_offset + PLT_TABLE_HEAD_LEN, modify_plt_content)

```

修复 plt 表后，执行 so，就可以看到我们注入的 decrypt 函数优先于 JNI 函数执行了。为什么需要修复 plt 表呢？是因为 decrypt 中使用了 android_log 这个系统函数（在正常解密函数中无法避免使用系统函数）

 

![](https://bbs.pediy.com/upload/attach/202103/833877_38MH43MBC8FNUY7.png)

 

到此，只需要将 libadd.so 中 decrypt 函数替换成真正的解密代码就行了。甚至是简单异或都可以实现 so 加密。

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)