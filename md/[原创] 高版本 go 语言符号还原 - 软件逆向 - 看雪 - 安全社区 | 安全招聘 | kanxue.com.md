> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277492.htm)

> [原创] 高版本 go 语言符号还原

实例版本
====

2022.6.8: go version go1.18.3 windows/amd64  
高版本适用 1.20

还原过程
====

搜索 go.build  
![](https://bbs.kanxue.com/upload/attach/202306/880848_ZB8N46JM9SGEPYV.png)  
![](https://bbs.kanxue.com/upload/attach/202306/880848_3F9MEFEC2EHWH9G.png)  
查找交叉引用  
![](https://bbs.kanxue.com/upload/attach/202306/880848_QY4VCXF2EJSQSN8.png)

 

go.build 在新版本中一定位于 函数名称的第一个。

 

![](https://bbs.kanxue.com/upload/attach/202306/880848_WTHPRDZ7Z4XM9BR.png)

 

https://go.dev/src/runtime/symtab.go  
阅读源码，获取 moduledata 结构

```
// pcHeader holds data used by the pclntab lookups.
type pcHeader struct {
    magic          uint32  // 0xFFFFFFF0
    pad1, pad2     uint8   // 0,0
    minLC          uint8   // min instruction size
    ptrSize        uint8   // size of a ptr in bytes
    nfunc          int     // number of functions in the module
    nfiles         uint    // number of entries in the file tab
    textStart      uintptr // base for function entry PC offsets in this module, equal to moduledata.text
    funcnameOffset uintptr // offset to the funcnametab variable from pcHeader
    cuOffset       uintptr // offset to the cutab variable from pcHeader
    filetabOffset  uintptr // offset to the filetab variable from pcHeader
    pctabOffset    uintptr // offset to the pctab variable from pcHeader
    pclnOffset     uintptr // offset to the pclntab variable from pcHeader
}
 
 
 
type moduledata struct {
    pcHeader     *pcHeader
    funcnametab  []byte
    cutab        []uint32
    filetab      []byte
    pctab        []byte
    pclntable    []byte
    ftab         []functab
    findfunctab  uintptr
    minpc, maxpc uintptr
 
    text, etext           uintptr
    noptrdata, enoptrdata uintptr
    data, edata           uintptr
    bss, ebss             uintptr
    noptrbss, enoptrbss   uintptr
    end, gcdata, gcbss    uintptr
    types, etypes         uintptr
    rodata                uintptr
    gofunc                uintptr // go.func.*
 
    textsectmap []textsect
    typelinks   []int32 // offsets from types
    itablinks   []*itab
 
    ptab []ptabEntry
 
    pluginpath string
    pkghashes  []modulehash
 
    modulename   string
    modulehashes []modulehash
 
    hasmain uint8 // 1 if module contains the main function, 0 otherwise
 
    gcdatamask, gcbssmask bitvector
 
    typemap map[typeOff]*_type // offset to *_rtype in previous module
 
    bad bool // module failed to load and should be ignored
 
    next *moduledata
}
```

pclntable 一般等于 ftab， 参照上图， ftab 与 pclntable 填充的是 pclntable 的值。

 

funcnametab 填充的是 函数名称  
![](https://bbs.kanxue.com/upload/attach/202306/880848_3S84PVBUBD6CKU2.png)  
filetab 填充的是 文件名称  
![](https://bbs.kanxue.com/upload/attach/202306/880848_XV4GF6FH8NF9YEZ.png)

 

与函数名称相关的是 ftab 和 pclntable  
![](https://bbs.kanxue.com/upload/attach/202306/880848_7KFZ6NR67228VV2.png)

 

适用于下面的结构体

```
type functab struct {
    entryoff uint32 // relative to runtime.text
    funcoff  uint32
}
```

![](https://bbs.kanxue.com/upload/attach/202306/880848_9CZ8E88GAS8CAKU.png)  
entryoff 为 以代码段为起始位置 的偏移。表示该函数实际的位置。 代码段在 windows 为 text

 

funcoff 为 以 pclntable 为起始位置的偏移。

```
//src\runtime\symtab.go
type functab struct {
    entry   uintptr
    funcoff uintptr
}
 
type funcInfo struct {
    *_func
    datap *moduledata
}
 
//src\runtime\runtime2.go
type _func struct {
    entry   uintptr // start pc
    nameoff int32   // function name
 
    args        int32  // in/out args size
    deferreturn uint32 // offset of start of a deferreturn call instruction from entry, if any.
 
    pcsp      uint32
    pcfile    uint32
    pcln      uint32
    npcdata   uint32
    cuOffset  uint32  // runtime.cutab offset of this function's CU
    funcID    funcID  // set for certain special runtime functions
    _         [2]byte // pad
    nfuncdata uint8   // must be last
}
```

对应着一个 funcInfo 结构体, 里面包含一个 _func 类型，该类型中有我们想要的信息。

 

意思就是: func 的 _func = pclntable + funcOff  
通过上图的信息计算:

```
hex(0x504320 + 0x2c20) = '0x506f40'
hex(0x504320 + 0x2c48) = '0x506f68'
```

发现刚好可以对得上信息  
![](https://bbs.kanxue.com/upload/attach/202306/880848_5K7JRHP8AUA2G6U.png)

 

第一个函数 go.build

 

![](https://bbs.kanxue.com/upload/attach/202306/880848_SKMERMXUEH9RU88.png)  
第二个函数 internal_cpu_Initialize  
![](https://bbs.kanxue.com/upload/attach/202306/880848_PUH85DJAPBPGRCN.png)

输出脚本
====

知道了这些就可以编写简单的脚本来还原 go 符号名了

 

ida python 脚本

```
import idc
from idc import *
import ida_nalt
 
moduledata_addr = 0x05289C0
pcHeader_addr = idc.get_qword(moduledata_addr)
 
if idc.get_wide_dword(pcHeader_addr) != 0x0FFFFFFF0:
    print(idc.get_wide_dword(pcHeader_addr))
    print("错误，并不是一个正确的go文件")
 
funcnametable_addr = idc.get_qword(moduledata_addr + 8)
filetab_addr = idc.get_qword(moduledata_addr + 8 + ((8*3) * 2))
pclntable_addr = idc.get_qword(moduledata_addr + 8 + ((8*3) * 4))
pclntable_size = idc.get_qword(moduledata_addr + 8 + ((8*3) * 4) + (8 * 4))
set_name(moduledata_addr, "firstmoduledata")
set_name(funcnametable_addr, "funcnametable")
set_name(filetab_addr, "filetab")
set_name(pclntable_addr, "pclntable")
 
print(pclntable_size)
 
def readString(addr):
    ea = addr
 
    res = ''
    cur_ea_db = get_db_byte(ea)
    while  cur_ea_db != 0 and cur_ea_db != 0xff:
        res += chr(cur_ea_db)
        ea += 1
        cur_ea_db = get_db_byte(ea)
    return res
 
def relaxName(name):
    # 将函数名称改成ida 支持的字符串
    #print(name)
    if type(name) != str:
        name = name.decode()
    name = name.replace('.', '_').replace("<-", '_chan_left_').replace('*', '_ptr_').replace('-', '_').replace(';','').replace('"', '').replace('\\', '')
    name = name.replace('(', '').replace(')', '').replace('/', '_').replace(' ', '_').replace(',', 'comma').replace('{','').replace('}', '').replace('[', '').replace(']', '')
    return name
 
cur_addr = 0
for i in range(pclntable_size):
 
    # 获取函数信息表
    cur_addr = pclntable_addr + (i * 8)
 
    # 获取函数入口偏移
    funcentryOff = get_wide_dword(cur_addr)
    funcoff = get_wide_dword(cur_addr + 4)
 
    funcInfo_addr = pclntable_addr + funcoff
 
    funcentry_addr = get_wide_dword(funcInfo_addr)
    funnameoff = get_wide_dword(funcInfo_addr + 4)
    funname_addr = funcnametable_addr + funnameoff
    funname = readString(funname_addr)
 
    # 真实函数地址
    truefuncname = relaxName(funname)
    truefuncentry = ida_nalt.get_imagebase() + 0x1000 + funcentryOff
 
    print(hex(truefuncentry), hex(funcoff), hex(funcInfo_addr),hex(funcentry_addr), hex(funnameoff),hex(funname_addr) ,funname)
    # 改名
    set_name(truefuncentry, truefuncname)
 
 
 
 
#print(hex(cur_addr))
```

其中 moduledata_addr 需要手动填充。

 

还原效果

 

![](https://bbs.kanxue.com/upload/attach/202306/880848_SRPE6HE8EZA7485.png)

参考链接
====

[https://github.com/Abyss-emmm/goparse](https://bbs.kanxue.com/target-KnvrJgogDg_2FZ8lHXYUMcR3f9JpwzIeDs7Jz8sKWl9WoatloJrSGVvTfg8VY_3D.htm)  
[跳跳糖](https://bbs.kanxue.com/target-DiSd96aFFSTdeN9Jd3SlRkbcy8pLMWTC1op6pN_2Bk1wiSeXql.htm)

[[招生] 系统 0day 安全 - IOT 设备漏洞挖掘（第 6 期）！](https://www.kanxue.com/book-section_list-7.htm)

最后于 2023-6-5 17:07 被 mingyuexc 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm)