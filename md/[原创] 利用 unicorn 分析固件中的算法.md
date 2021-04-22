> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267153.htm)

前言
==

近年来，越来越多安全研究员开始使用 QEMU 以及 unicorn 这类虚拟化技术对固件进行模拟执行，甚至是 FUZZ 测试。模拟执行在嵌入式固件分析中应用越来越广泛，为了让大家能了解这一技术的使用方法，本文从实战出发，利用 unicorn 框架分析某个设备的加密算法。

分析固件
====

本文的目的想要分析并调用该固件的一个魔改的 MD5。  
![](https://bbs.pediy.com/upload/attach/202104/651413_WNR77DN7PHP8H3V.png)  
我们简单地对比了该算法和标准 md5 的区别，发现大量算法常数被修改了。

MD5Init 对比
----------

标准算法：  
![](https://bbs.pediy.com/upload/attach/202104/651413_GWMNZPA8D254SYX.png)  
修改后的：  
![](https://bbs.pediy.com/upload/attach/202104/651413_8ZJZYVCCYXRRYWC.png)

MD5Step 对比
----------

标准算法：  
![](https://bbs.pediy.com/upload/attach/202104/651413_45VQTAGTR7J4AVE.png)  
修改后的：  
![](https://bbs.pediy.com/upload/attach/202104/651413_CNSCR73VJS7VGET.png)

分析思路
----

1.  对照标准算法实现，分析魔改的地方，然后对标准算法进行修改。  
    缺点：分析时间长，修改的地方多的话还容易出错。特别是遇上二进制代码混淆，分析起来更加麻烦。
2.  无需花大量时间分析算法，直接复制 IDA 反编译的代码，重新编译即可。  
    缺点：反编译的伪代码不一定准确，如果代码函数较多，需要复制和整理的函数也比较多，特别是变量类型这块，也要进行修复。  
    3．只需要分析函数的参数，使用模拟执行技术，对关键算法进行模拟执行。

模拟执行
====

分析函数参数
------

标准的 MD5 一般有三个函数，分别如下所示：  
![](https://bbs.pediy.com/upload/attach/202104/651413_7BEMPFQZPQWWK2Z.png)  
为了模拟执行，我们需要在 IDA 找到这三个函数的地址，如下所示：  
![](https://bbs.pediy.com/upload/attach/202104/651413_RC8FZ7YDAM38527.png)  
用法如下，这里就不多介绍了  
![](https://bbs.pediy.com/upload/attach/202104/651413_KMEK35FA99SNDB7.png)

模拟环境初始化
-------

该固件是 MIPS 大端架构系统，初始化一些加载地址，栈地址之类的全局变量  
![](https://bbs.pediy.com/upload/attach/202104/651413_PWEBF9WNMBAD8T8.png)  
解析 ELF 文件，把固件的代码段读取到模拟器中：  
![](https://bbs.pediy.com/upload/attach/202104/651413_4J8B76C3RX2MMXF.png)  
分配栈空间，以及变量空间，用于存放 md5_ctx 以及输入的变量字符串  
![](https://bbs.pediy.com/upload/attach/202104/651413_Q4D44E5HVPA9AAH.png)

调用 MD5 函数
---------

首先，我们为了让模拟器调用完每个函数之后能够停止运行，必须将返回地址设置为一个指定的地址，当 callback 检测到运行到该地址，立刻停止下来了：  
![](https://bbs.pediy.com/upload/attach/202104/651413_CBYJ3778Z7MPQ8F.png)  
分别调用 3 个函数：  
![](https://bbs.pediy.com/upload/attach/202104/651413_FX5X3NSGP4YWVCD.png)  
通过代码可以知道，最终的 md5 值在 MD5Context 偏移为 88 的地方：  
![](https://bbs.pediy.com/upload/attach/202104/651413_96SH6KZ7CQD8EK5.png)  
所以在调用完成之后直接把 MD5_CTX 偏移为 88 的数据读取出来即为 MD5 运算结果：  
![](https://bbs.pediy.com/upload/attach/202104/651413_UQJ87SGB45BWFBF.png)

运行结果
----

当输入为 12345678 得到下面的 MD5 值  
![](https://bbs.pediy.com/upload/attach/202104/651413_XCTZ4XY439AKKXQ.png)  
所有代码如下：

```
from unicorn import *
from capstone import *
from unicorn.mips_const import *
from elftools.elf.elffile import ELFFile
from elftools.elf.segments import Segment
import ctypes
import binascii
import hexdump
filepath='fw'
 
load_base=0
stack_base=0
stack_size=0x20000
var_base=load_base+stack_size
var_size=0x10000
stop_stub_addr=0x30000
stop_stub_size=0x10000
 
 
emu = Uc(UC_ARCH_MIPS,UC_MODE_MIPS32 + UC_MODE_BIG_ENDIAN)
 
def disasm(bytecode,addr):
    md=Cs(CS_ARCH_MIPS,CS_MODE_MIPS32+ CS_MODE_BIG_ENDIAN)
    for asm in md.disasm(bytecode,addr):
        return '%s\t%s'%(asm.mnemonic,asm.op_str)
def align(addr, size, growl):
    UC_MEM_ALIGN = 0x1000
    to = ctypes.c_uint64(UC_MEM_ALIGN).value
    mask = ctypes.c_uint64(0xFFFFFFFFFFFFFFFF).value ^ ctypes.c_uint64(to - 1).value
    right = addr + size
    right = (right + to - 1) & mask
    addr &= mask
    size = right - addr
    if growl:
        size = (size + to - 1) & mask
    return addr, size
def hook_code(uc, address, size, user_data):
    bytecode=emu.mem_read(address,size)
    print(" 0x%x :%s"%(address,disasm(bytecode,address)))
    if address==stop_stub_addr:
        emu.emu_stop()
 
 
#init var
 
def my_md5(key):
 
    with open(filepath, 'rb') as elffile:
        elf=ELFFile(elffile)
        load_segments = [x for x in elf.iter_segments() if x.header.p_type == 'PT_LOAD']
 
        for segment in load_segments:
            prot = UC_PROT_ALL
            print('mem_map: addr=0x%x  size=0x%x'%(segment.header.p_vaddr,segment.header.p_memsz))
 
            addr,size=align(load_base + segment.header.p_vaddr,segment.header.p_memsz,True)
            emu.mem_map(addr, size, prot)
            emu.mem_write(addr, segment.data())
 
    emu.mem_map(stack_base, stack_size, UC_PROT_ALL)
    emu.mem_map(var_base, var_size, UC_PROT_ALL)
 
    md5_ctx=var_base
    psw=var_base+0x5000
 
    emu.mem_write(psw,key)
 
 
    emu.mem_map(stop_stub_addr, stop_stub_size, UC_PROT_ALL)
    emu.reg_write(UC_MIPS_REG_A0, md5_ctx)
    emu.reg_write(UC_MIPS_REG_RA,stop_stub_addr)
    emu.reg_write(UC_MIPS_REG_SP,stack_base+stack_size)
 
    my_MD5Init_addr=0x0041FAA8
    my_MD5Update_addr=0x0041FAE4
    my_MD5Final_addr=0x0041FC18
 
    #MD5Init
    code=emu.mem_read(my_MD5Init_addr, 8)
    emu.hook_add(UC_HOOK_CODE, hook_code)
    emu.emu_start(my_MD5Init_addr, my_MD5Init_addr + 0x1000)
 
    #MD5Update
    emu.reg_write(UC_MIPS_REG_A0, md5_ctx)
    emu.reg_write(UC_MIPS_REG_A1, psw)
    emu.reg_write(UC_MIPS_REG_A2, len(key))
    emu.reg_write(UC_MIPS_REG_SP,stack_base+stack_size)
    emu.reg_write(UC_MIPS_REG_RA,stop_stub_addr)
    emu.emu_start(my_MD5Update_addr, my_MD5Update_addr + 0x1000)
    #MD5Final
 
    emu.reg_write(UC_MIPS_REG_A0, md5_ctx)
    emu.reg_write(UC_MIPS_REG_SP,stack_base+stack_size)
    emu.reg_write(UC_MIPS_REG_RA,stop_stub_addr)
    emu.emu_start(my_MD5Final_addr, my_MD5Final_addr + 0x1000)
 
    return emu.mem_read(md5_ctx+88,16)
 
if __name__=="__main__":
    key=b'12345678'
    hexdump.hexdump(my_md5(key))

```

总结
==

本文通过 unicorn 将固件中魔改的 md5 算法成功进行模拟执行，并输出了正确的值，说明使用 unicorn 对固件中的算法进行分析是非常有效的。这将会对使用了混淆的算法特别有用，逆向研究人员只需要分析关键函数以及参数，让 unicorn 执行模拟即可，当然还有些不足，比如有些 CPU 指令支持不完全，运行效率等问题。

参考文章
====

https://bbs.pediy.com/thread-253868.htm

[[公告]5 月 14 日腾讯安全零信任发展趋势论坛重磅开幕！邀您一起从 “零” 开始，共建信任！！](https://zta.insecworld.com/?utm_campaign=MJTG&utm_source=KX&utm_medium=WZLJ)