> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1492187-1-1.html)

> [md]## 前言本文是对上一篇文章 [基于 IDA Python 的 OLLVM 反混淆 (二) 实战腾讯御安全 (模拟器)](https://www.52pojie.cn/thread-1488919-1-1.h......

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Shocker _ 本帖最后由 Shocker 于 2021-8-11 14:34 编辑_  

前言
--

本文是对上一篇文章  
[基于 IDA Python 的 OLLVM 反混淆 (二) 实战腾讯御安全 (模拟器)](https://www.52pojie.cn/thread-1488919-1-1.html)  
的补充.

思路
--

1.  合并代码块
2.  修复 call 的偏移地址
3.  去除残留的属于控制块的代码

实现过程
----

实验环境: ida7.5 + 雷电模拟器 (64 位)4

由上篇可知  
原始指令最终被化简成了许多的代码块, 且大多数代码块的结尾以 jmp 结束  
![](https://attach.52pojie.cn/forum/202108/11/140226zks547glm555spy4.jpg)

**liuchengtu1.jpg** _(12.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyMTYyN3w3YzA4NTJjZXwxNjI4NjczMjkxfDIxMzQzMXwxNDkyMTg3&nothumb=yes)

2021-8-11 14:02 上传

我们就可以将 jmp 指令去除, 将代码块合并在一起

### 代码块合并

```
#combine_block.py
import keypatch
from idaapi import *
import capstone
import struct

combine_blocks={}
ea_blcok_map={}
codes_map={}

def nop_block(block):
    nop_code=0x90
    for i in range(block.end_ea-block.start_ea):
        idc.patch_byte(block.start_ea+i,nop_code)

so_base=idaapi.get_imagebase()
fun_offset=0x25D0
f_blocks = idaapi.FlowChart(idaapi.get_func(so_base+fun_offset), flags=idaapi.FC_PREDS)
for block in f_blocks:
    if block.start_ea==block.end_ea:
        continue
    ea_blcok_map[block.start_ea]=block
    codes_map[block.start_ea]=get_bytes(block.start_ea,block.end_ea-block.start_ea)
    block_end=idc.prev_head(block.end_ea)
    if idc.print_insn_mnem(block_end).startswith('j'):
        next_block=get_operand_value(block_end,0)
        combine_blocks[block.start_ea]=next_block
    else:
        combine_blocks[block.start_ea]=block.end_ea
    # 将所有块nop后再将原始指令连接    
    nop_block(block)

first_block=so_base+fun_offset
wirte_offect=0

while True:
    if wirte_offect==0:
        block=ea_blcok_map[first_block]
    else:
        block=ea_blcok_map[next_block]

    codes=codes_map[block.start_ea]
    md=capstone.Cs(capstone.CS_ARCH_X86,capstone.CS_MODE_32)
    for code in md.disasm(codes,block.start_ea):
        if code.mnemonic=='jmp' or code.mnemonic.startswith('cmov') or code.mnemonic=='nop': #排除这些指令
            continue
        block_bytes=bytes(code.bytes)
        if code.mnemonic=='call':
            if code.op_str.startswith('0x'):
                called_addr=int(code.op_str,16)
                fix_addr=called_addr-fun_offset-wirte_offect-5
                fix_bytes=struct.pack('i',fix_addr)
                block_bytes=bytes(code.bytes[0:1])+fix_bytes
        print('combine_block:0x%x'%block.start_ea)
        patch_bytes(first_block+wirte_offect,block_bytes)
        wirte_offect=wirte_offect+len(block_bytes)

    if block.start_ea in combine_blocks:     
        next_block=combine_blocks[block.start_ea]
        if not next_block in ea_blcok_map:
            break
    else:
        break

    # print('0x%x,0x%x'%(key,combine_blocks[key]))


```

合并后的流程图  
![](https://attach.52pojie.cn/forum/202108/11/140309eh7gdffnn66gf44f.jpg)

**liuchengtu2.jpg** _(2.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyMTYyOHw5YzJjMDRlY3wxNjI4NjczMjkxfDIxMzQzMXwxNDkyMTg3&nothumb=yes)

2021-8-11 14:03 上传

可以看到, 还有许多属于控制块的指令没有去除  
![](https://attach.52pojie.cn/forum/202108/11/140330mtbt9cyst9700yy9.jpg)

**code1.jpg** _(230.06 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyMTYyOXw3N2I2YjA0NXwxNjI4NjczMjkxfDIxMzQzMXwxNDkyMTg3&nothumb=yes)

2021-8-11 14:03 上传

### 去除混淆指令

去除混淆指令原理基于  
[x86 反混淆 IDA Python 脚本 (一)](https://www.52pojie.cn/thread-1491068-1-1.html)  
反复运行该脚本直到无新指令被 patch, 手动删除一些无效的内存指令  
最后执行 combine_block.py 脚本去除所有 nop 指令  
去混淆的代码片段如下所示  
![](https://attach.52pojie.cn/forum/202108/11/140344q4yicljj141tyy84.jpg)

**code2.jpg** _(203.66 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyMTYzMHw2MTg5MGQ0MHwxNjI4NjczMjkxfDIxMzQzMXwxNDkyMTg3&nothumb=yes)

2021-8-11 14:03 上传

相关示例代码见  
[https://github.com/PShocker/de-ollvm/](https://github.com/PShocker/de-ollvm/)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)wushaoye 学习学习 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) S11ence 刚好在学习 ollvm