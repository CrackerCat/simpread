> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268108.htm)

> [原创] 使用 unicorn 对 ollvm 字符串进行解密

[](#题目出自3w班12月第一题。对抗字符串混淆)题目出自 3w 班 12 月第一题。对抗字符串混淆
===================================================

样本是一个比较初级的 ollvm 混淆，其字符串解密过程在 init_array 中，所以 so 文件正常加载到内存中后，其加密的字符串就已经解密后写回原始地址了。  
所以我们可以通过 unicorn 加载 so 文件，手动将解密后的字符串写回 so 文件，经过简单的 patch 就可以静态分析了

 

本文参考了 [leadroyal](https://www.leadroyal.cn/?p=968) 与 [smartdone](https://www.anquanke.com/post/id/181051#h3-8) 大佬的文章。

使用 unicorn 对字符串进行解密
-------------------

问题一：

 

偷懒使用 unicorn 直接将整个 so 文件 load 进 memory，读取 data 的数据全是 00，修复后也是错误的，我按照 leadroyal 大佬的脚本修改后是正确的

 

问题二：

 

有时候使用 unicorn map 后，代码看着没有问题，但是跑起来就报错，可能你需要看看你分配的内存是不是不是 1024 的倍数

 

脚本如下：

```
import binascii
import math
 
import ida_bytes
import idaapi
from capstone import *
from elftools.elf.elffile import ELFFile
from elftools.elf.sections import SymbolTableSection
from keystone import *
from unicorn import *
from unicorn.arm_const import *
 
 
class FILFINIT(object):
    def __init__(self, file, mode):
        self.file = file
        self.fd = self.file_init(mode)
        self.elf = ELFFile(self.fd)
        self.file_size = self.file_size()
        self.data_divs = self.get_data_div()
 
    def file_init(self, mode='rb+'):
        fd = open(self.file, mode=mode)
        return fd
 
    def file_close(self):
        self.fd.close()
 
    def file_seek_to(self, seekto):
        self.fd.seek(seekto, 0)
 
    def file_read(self, size=-1):
        binary = self.fd.read(size)
        return binary
 
    def file_write(self, data):
        self.fd.write(data)
 
    def file_size(self):
        import os
        return os.path.getsize(self.file)
 
    def get_data_div(self):
        data_divs = []
        if self.elf:
            symtab = self.elf.get_section_by_name('.dynsym')
            assert isinstance(symtab, SymbolTableSection)
 
            for sym in symtab.iter_symbols():  # sym.name:str
                if sym.name.startswith('.datadiv'):
                    data_divs.append(sym.entry.st_value)
        return data_divs
 
 
class UCINIT(object):
    def __init__(self, file, mode, offset=0x10000, stack_size=0x10000, data_size=0x10000):
        self.cs = Cs(CS_ARCH_ARM, CS_MODE_THUMB)
        self.fileinfo = FILFINIT(file, mode)
        self.data_divs = self.uc_filter()
        self.func_start_and_end = self.get_func_start_and_end()
 
        self.uc = unicorn.Uc(UC_ARCH_ARM, UC_MODE_THUMB)
        self.image_base = 0x10000
        self.offset = offset
        self.stack_size = stack_size
        self.data_size = data_size
        self.image_size = (math.ceil(self.fileinfo.file_size / self.offset) + 1) * self.offset
        self.uc.mem_map(self.image_base, self.image_size)
        self.init_seg()
        # self.uc.mem_write(self.image_base, self.fileinfo.file_read()) # 全文件导入是错误的，不知道为什么
        self.stack_base = self.image_base + self.image_size + self.offset
        self.stack_top = self.stack_base + self.stack_size - 0x1000
        self.uc.mem_map(self.stack_base, self.stack_size)
        self.uc.reg_write(UC_ARM_REG_SP, self.stack_top)
        self.data_base = self.stack_base + self.stack_size + self.offset
        self.uc.mem_map(self.data_base, self.data_size)
 
    def init_seg(self):
        segs = [seg for seg in self.fileinfo.elf.iter_segments() if seg.header.p_type == 'PT_LOAD']
        for seg in segs:
            self.uc.mem_write(self.image_base + seg.header.p_vaddr, seg.data())
 
    def init_reg(self, regs):
        index = 0
        for reg in regs:
            reg_num = 66 + index
            self.uc.reg_write(reg_num, reg)
 
    def uc_run(self, target, end):
        targ = target + self.image_base
        util = self.image_base + end
        self.uc.emu_start(targ, util)
 
    def uc_stop(self):
        self.uc.emu_stop()
        self.fileinfo.file_close()
 
    def register_hook_code(self, htype, callback):
        self.uc.hook_add(htype, callback)
 
    def uc_filter(self, size=0x2):
        data_divs = []
        for data_div in self.fileinfo.data_divs:
            self.fileinfo.file_seek_to(data_div - 1)
            data = self.fileinfo.file_read(size)
            for inis in self.cs.disasm(data, 0):
                if inis.mnemonic == 'bx' and inis.op_str == 'lr':
                    continue
                else:
                    data_divs.append(data_div)
        self.fileinfo.file_seek_to(0)
        return data_divs
 
    def get_func_start_and_end(self):
        data_divs = []
        for data_div in self.data_divs:
            func = idaapi.get_func(data_div)
            func_start_and_end = (data_div, func.end_ea - 0x2)
            data_divs.append(func_start_and_end)
        return data_divs
 
    def get_elf_data(self):
        data_section_header = self.fileinfo.elf.get_section_by_name('.data').header
        print('[+] .data addr {}-{}'.format(hex(data_section_header.sh_addr), hex(data_section_header.sh_size)))
        new_data = self.uc.mem_read(self.image_base + data_section_header.sh_addr, data_section_header.sh_size)
        print('[+] .data binary {}'.format(binascii.hexlify(new_data)))
        return new_data
 
    def patch_so_data(self):
        data_section_header = self.fileinfo.elf.get_section_by_name('.data').header
        data_addr = data_section_header.sh_addr
 
        # self.fileinfo.file_seek_to(data_addr)
        # self.fileinfo.file_write(self.get_elf_data())
        new_data = self.get_elf_data()  # new_data: bytearray
        print('[+] patch so data addr {}\n {}'.format(hex(data_addr), bytes(new_data)))
        ida_bytes.patch_bytes(data_addr, bytes(new_data))
 
    def patch_data_div(self):
        ks = Ks(KS_ARCH_ARM, KS_MODE_THUMB)
        for data_div in self.data_divs:
            binary = ks.asm('bx lr')
            print(bytes(binary[0]))
            # print(struct.pack('B', binary[0]))
            print('[+] patch so text .datadiv {}\n {}'.format(hex(data_div - 1), bytes(binary[0])))
            ida_bytes.patch_bytes(data_div - 1, bytes(binary[0]))
 
 
def hook_code(uc: unicorn.Uc, address, size, user_data):
    cs = Cs(CS_ARCH_ARM, CS_MODE_THUMB)
    me = uc.mem_read(address, size)
    for code in cs.disasm(me, size):
        print("[+] Tracing instructions at 0x%x, instructions size = ix%x, inst: %s %s" % (
            address, size, code.mnemonic, code.op_str))
 
 
def hook_mem_access(uc, access, address, size, value, user_data):
    if access == UC_MEM_WRITE:
        print('[+] Memory is being WRITE at 0x%x, data size = %u, data value = 0x%x' % (address, size, value))
    else:  # READ
        me = uc.mem_read(address, size)
        print('[+] Memory is being READ at 0x%x, data size = %u, data value = 0x%s' % (
            address, size, binascii.hexlify(me)))
 
 
if __name__ == '__main__':
    print('[+] So file parse starting!')
    so = UCINIT('libcrack.so', 'rb')
    # so.get_elf_data()
    for start, end in so.func_start_and_end:
        print('[+] Data div addr {}-{}'.format(hex(start), hex(end)))
        so.init_reg((0, 0, 0))
        # so.register_hook_code(UC_HOOK_CODE, hook_code)
        # so.register_hook_code(UC_HOOK_MEM_READ, hook_mem_access)
        # so.register_hook_code(UC_HOOK_MEM_WRITE, hook_mem_access)
        so.uc_run(start, end)
        so.patch_data_div()
    # so.patch_so_data()
    # so.get_elf_data()
    so.uc_stop()
    print('[+] So file run finished!')

```

uemu 对字符串进行解密 "href="# 使用 [uemu](https://github.com/alexhude/uemu) 对字符串进行解密 "> 使用 [uEmu](https://github.com/alexhude/uEmu) 对字符串进行解密
-----------------------------------------------------------------------------------------------------------------------------------

```
# 下载uEmu
$ git clonehttps://github.com/alexhude/uEmu

```

复制 uEmu.py 到 IDA_Pro_7.5\plugins \ 下，重启 ida

 

找到函数，设置起始与结束断点：

 

![](https://bbs.pediy.com/upload/attach/202106/811222_CPFMSCUUDNTXGJW.png)

 

在需要执行的汇编指令起点，右键 - uEmu-start

 

![](https://bbs.pediy.com/upload/attach/202106/811222_364P8ASBGMHWCEE.png)

 

配置寄存器

 

![](https://bbs.pediy.com/upload/attach/202106/811222_CHNES5NYTDJXKSJ.png)

 

ctrl+shift+alt+s 单步运行

 

![](https://bbs.pediy.com/upload/attach/202106/811222_GWJHQT6KJMWGUVT.png)

 

如果你需要多行执行，单击要执行到某一行，右键 - uEmu-run

 

![](https://bbs.pediy.com/upload/attach/202106/811222_SX27PBKPE663WX8.png)

 

会自动向下执行到你单击的那一行

 

查看内存，比如我们要查看 0x1F0AD 处的内存，看一下解密后的字符串内容：右键 - uEmu-show memory range

 

![](https://bbs.pediy.com/upload/attach/202106/811222_FBMCP4XM7W8EHSG.png)

 

填写地址与长度，注意长度为 10 进制，选择 add

 

![](https://bbs.pediy.com/upload/attach/202106/811222_P8PP86H3F7ADAF7.png)

 

![](https://bbs.pediy.com/upload/attach/202106/811222_HZKPFFJK5KS9V3J.png)

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 11 小时前 被新萌编辑 ，原因： 修改样式

上传的附件：

*   [libcrack.so](javascript:void(0)) （122.64kb，2 次下载）
*   [libcrack.so.idb](javascript:void(0)) （2.32MB，2 次下载）
*   [libcrypt.so](javascript:void(0)) （128.04kb，2 次下载）
*   [obf.so](javascript:void(0)) （14.03kb，2 次下载）