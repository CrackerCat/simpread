> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277304.htm)

> [原创]ollvm 分析及反混淆

[原创]ollvm 分析及反混淆

2023-5-21 22:26 18192

### [原创]ollvm 分析及反混淆

2023-5-21 22:26

样本  

=====

百度加固 libbaiduprotect.so

说明  

=====

1、本文是对特定样本 ollvm 的分析，提供一种反混淆的方法思路，包含详细的分析过程和针对该样本的反混淆脚本，不包含通用的 ollvm 反混淆脚本。  

2、本文仅分析 init_array 中的第一个函数 sub_88060 ，其余函数的反混淆可按照本文的思路自行处理。

目标
==

对 bcf 和 fla 进行还原

效果  

=====

原始代码
----

![](https://bbs.kanxue.com/upload/attach/202305/745332_H2UPN8ADDMPP8A5.jpg)

去除 bcf 的效果
----------

![](https://bbs.kanxue.com/upload/attach/202305/745332_7Q2RXSRETJNUWT6.jpg)

去除 fla 的效果
----------

![](https://bbs.kanxue.com/upload/attach/202305/745332_2YBKADWECXCYWZ3.jpg)
---------------------------------------------------------------------------

详细分析
====

bcf 分析
------

ida 查看函数的伪代码，包含大量的 _((dword_C0118 * (dword_C0118 - 1)) & 1) == 0 || dword_C0120 < 10_ 判断。  

通过简单计算可知，该判断条件的结果永远为 _false_ 。

所以我们要想办法把该判断识别出来，并把无效跳转给 _nop_ 掉。

![](https://bbs.kanxue.com/upload/attach/202305/745332_366EST4BVHGRP7E.jpg)

转到汇编窗口，找到该条件的汇编指令，

通过分析可知，最后的 _B.EQ_ 永远不会跳转，所以把该指令 _nop_ 即可。

![](https://bbs.kanxue.com/upload/attach/202305/745332_NBXFCVHGRWZFF42.jpg)

先简单写个脚本尝试一下，按顺序匹配指令。

```
import idc
 
nop = 0xD503201F
 
 
def find_bcf(start, end):
    ea = start
    # 01
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'ADRP' or val_1 != 0xC0000:
        return start + 4
 
    # 02
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'ADRP' or val_1 != 0xC0000:
        return start + 4
 
    # 03
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'MOV' or val_1 != 1:
        return start + 4
 
    # 04
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_2 = idc.get_operand_value(ea, 2)
    if mnem != 'ADD' or val_2 != 0x118:
        return start + 4
 
    # 05
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'LDR':
        return start + 4
 
    # 06
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'SUBS':
        return start + 4
 
    # 07
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'MUL':
        return start + 4
 
    # 08
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'MOV' or val_1 != 1:
        return start + 4
 
    # 09
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_2 = idc.get_operand_value(ea, 2)
    if mnem != 'ADD' or val_2 != 0x120:
        return start + 4
 
    # 10
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'LDR':
        return start + 4
 
    # 11
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'AND':
        return start + 4
 
    # 12
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'CMP' or val_1 != 0:
        return start + 4
 
    # 13
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    op_1 = idc.print_operand(ea, 1)
    if mnem != 'CSET' or op_1 != 'EQ':
        return start + 4
 
    # 14
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'CMP' or val_1 != 10:
        return start + 4
 
    # 15
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    op_1 = idc.print_operand(ea, 1)
    if mnem != 'CSET' or op_1 != 'LT':
        return start + 4
 
    # 16
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'ORR':
        return start + 4
 
    # 17
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    val_1 = idc.get_operand_value(ea, 1)
    if mnem != 'CMP' or val_1 != 0:
        return start + 4
 
    # 18
    ea += 4
    if ea >= end:
        return ea
    mnem = idc.print_insn_mnem(ea)
    if mnem != 'B.EQ':
        return start + 4
 
    print(hex(ea))
    idc.patch_dword(ea, nop)
 
    return ea + 4
 
 
def patch_bcf_func(func_start):
    func_end = idc.find_func_end(func_start)
    if func_end == idc.BADADDR:
        return
 
    ea = func_start
    while ea < func_end:
        ea = find_bcf(ea, func_end)
 
 
patch_bcf_func(0x88060)

```

脚本执行后，发现只处理了一部分，

找到未识别的地方，查看汇编指令，发现这些指令并不全在一起，中间可能插入其他指令。

然后发现，未识别的这个地方最后 8 条指令是和前面分析的一致，而且前面的指令是取值，核心的判断逻辑是后面这 8 条指令，  

于是直接把脚本中多余的指令判断全删除，只保留 _AND_ 开始的指令。

![](https://bbs.kanxue.com/upload/attach/202305/745332_RXVU3QC4UEUMW7V.jpg)

ida 重新加载 so**（**上一次脚本把部分跳转指令 nop 掉了，直接执行会导致本次 nop 掉正常的跳转**）**

删除部分指令判断后，重新执行脚本，发现任然有部分未被处理。

分析后发现，最后几条指令之间也可能插入有其他指令。

![](https://bbs.kanxue.com/upload/attach/202305/745332_FT72KW9CNAS68YS.jpg)

再次修改脚本，在每两条指令中间，都加上判断。

```
import idc
 
nop = 0xD503201F
 
 
def find_bcf(start, end):
    ea = start
 
    # # 11
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        ea += 4
        if mnem == 'AND':
            break
 
    # 12
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        val_1 = idc.get_operand_value(ea, 1)
        ea += 4
        if mnem == 'CMP' and val_1 == 0:
            break
 
    # 13
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        op_1 = idc.print_operand(ea, 1)
        ea += 4
        if mnem == 'CSET' and op_1 == 'EQ':
            break
 
    # 14
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        val_1 = idc.get_operand_value(ea, 1)
        ea += 4
        if mnem == 'CMP' and val_1 == 10:
            break
 
    # 15
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        op_1 = idc.print_operand(ea, 1)
        ea += 4
        if mnem == 'CSET' and op_1 == 'LT':
            break
 
    # 16
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        ea += 4
        if mnem == 'ORR':
            break
 
    # 17
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            return start + 4
        val_1 = idc.get_operand_value(ea, 1)
        ea += 4
        if mnem == 'CMP' and val_1 == 0:
            break
 
    while ea < end:
        mnem = idc.print_insn_mnem(ea)
        if mnem and mnem[0] == 'B':
            if mnem == 'B.EQ':
                print(hex(ea))
                idc.patch_dword(ea, nop)
            break
        ea += 4
 
    return ea
 
 
def patch_bcf_func(func_start):
    func_end = idc.find_func_end(func_start)
    if func_end == idc.BADADDR:
        return
 
    ea = func_start
    while ea < func_end:
        ea = find_bcf(ea, func_end)
 
 
patch_bcf_func(0x88060)

```

ida 重新加载 so**（**上一次脚本把部分跳转指令 nop 掉了，直接执行会导致本次 nop 掉正常的跳转**）**

执行修改后的脚本，查看伪代码，判断条件全都被清除了。

![](https://bbs.kanxue.com/upload/attach/202305/745332_WBSHNQ3M7GAHGFQ.jpg)

另一种 bcf 处理方法
------------

此处再提供一种简单的处理方法，ida 生成的伪代码之所以这么多垃圾代码，是因为 bcf 所引用的内存属性包含可写属性，所以我们可以通过将其引用的内存地址变为不可写，这样 ida 就会自动进行优化。

通过查看代码可知，bcf 所引用的地址为 _dword_C0118_ 和 _dword_C0118 _ ，这两个地址都属于. bss 段

![](https://bbs.kanxue.com/upload/attach/202305/745332_QT7R674P94R6AGW.jpg)

.bss 段的内存属性为可读可写，为了不影响 ida 对. bss 中其他变量的分析，把这两个地址单独放在一个段。

发现这两个变量在. bss 段的末尾，而. prgend 段在. bss 后面，并且. prgend 没有内容，

因此可直接修改这两个段的大小，然后把. prgend 段属性改为不可写。

![](https://bbs.kanxue.com/upload/attach/202305/745332_HUW43HQJHXVV4QB.jpg)

手动设置比较麻烦，直接通过脚本设置一下。

```
import idc
 
 
def modify_segment():
    idc.set_segment_bounds(0xC0118, idc.get_segm_start(0xC0118), 0xC0118, idc.SEGMOD_SILENT)
    idc.set_segment_bounds(0xC0148, 0xC0118, 0xC0149, idc.SEGMOD_SILENT)
    idc.set_segm_attr(0xC0148, idc.SEGATTR_PERM, 4)
    idc.patch_dword(0xC0118, 0)
    idc.patch_dword(0xC0120, 0)
 
 
modify_segment()

```

执行脚本修改段属性后，效果和直接 nop 跳转一样  

![](https://bbs.kanxue.com/upload/attach/202305/745332_NR98P5Z9XRFKAVZ.jpg)

fla 分析![](https://bbs.kanxue.com/plugin/chao_editor/rich_text/themes/default/images/spacer.gif)
-----------------------------------------------------------------------------------------------

查看上文处理后的伪代码，包含 while 和 switch 组合，将原本连续的代码分割成了多块，

我们的目标是去除该结构，将所有代码连接起来。

通过分析可知，每个代码块会指定下一个代码块的索引，

所以我们要想办法，把每个代码块设置的索引识别出来，并直接跳转到下一代码块

![](https://bbs.kanxue.com/upload/attach/202305/745332_TDTWGEJ9QY3FBTU.jpg)

通过查看流程图可知，所有分支执行完后都会跳转到 def_88A9C（0x90ED8）再次进行分发，

![](https://bbs.kanxue.com/upload/attach/202305/745332_ST6M8D2B9QHSZC9.jpg)  

由流程图可知，下一个代码块的索引存在 X28 指向的地址，

![](https://bbs.kanxue.com/upload/attach/202305/745332_8HW45876Y9SSSPW.jpg)

先简单写个脚本，看能否将索引和跳转地址识别出来

先查找所有跳转到默认地址的指令

然后倒回去查找 str 和 mov 指令

mov 指令的参数即为下一个代码块的索引

```
import idc
 
 
def find_fla(ea, end, jpt_addr, def_addr):
    # 01
    addr = ea
    while addr < end:
        mnem = idc.print_insn_mnem(addr)
        val_0 = idc.get_operand_value(addr, 0)
        if mnem == 'B' and val_0 == def_addr:
            B_addr = addr
            break
        addr += 4
    else:
        return addr
 
        #  STR             W10, [X28]
    while ea <= addr:
        mnem = idc.print_insn_mnem(addr)
        STR_R = idc.print_operand(addr, 0)
        op_1 = idc.print_operand(addr, 1)
        if mnem == 'STR' and op_1 == '[X28]':
            break
        addr -= 4
 
    # MOV             W10, #0x12
    while ea <= addr:
        mnem = idc.print_insn_mnem(addr)
        op_0 = idc.print_operand(addr, 0)
        val_1 = idc.get_operand_value(addr, 1)
        if mnem == 'MOV' and op_0 == STR_R:
            to_addr = hex((jpt_addr + idc.get_wide_dword(jpt_addr + (val_1 - 1) * 4)) & 0xffffffff)
            print(hex(B_addr), val_1, to_addr)
            break
        addr -= 4
 
    return B_addr
 
 
def patch_fla_func(func_start, jpt_addr, def_addr):
    func_end = idc.find_func_end(func_start)
    if func_end == idc.BADADDR:
        return
 
    ea = func_start
    while ea < func_end:
        ea = find_fla(ea, func_end, jpt_addr, def_addr) + 4
 
 
patch_fla_func(0x88060, 0x958F4, 0x90ed8)

```

执行脚本后，识别到 29 个代码块，但是该函数实际有 31 个块

通过对比后发现，索引为 8 和 0x18 的块未识别  

分析汇编指令后，发现是未通过 X28 直接赋值导致的，

在 str 指令之后又通过 mov 进行赋值

![](https://bbs.kanxue.com/upload/attach/202305/745332_T6464MGADDYZQPN.jpg)

修改脚本，查找 str 指令的时候，不再固定为 X28

再次执行脚本，31 个代码块全部识别

```
r_28 = 28
while ea <= addr:
    mnem = idc.print_insn_mnem(addr)
    STR_R = idc.print_operand(addr, 0)
    op_1 = idc.print_operand(addr, 1)
    if mnem == 'STR' and op_1 == f'[X{r_28}]':
        break
    if mnem == 'MOV' and STR_R == f'X{r_28}':
        r_28 = op_1[1:]
    addr -= 4

```

向脚本添加 patch 指令，每个代码块结束的时候，直接跳到下一代码块

```
def patch_fla(jpt_addr, index, B_addr):
    to_addr = hex((jpt_addr + idc.get_wide_dword(jpt_addr + (index - 1) * 4)) & 0xffffffff)
    print(hex(B_addr), index, to_addr)
 
    encoding, count = ks.asm(f'b {to_addr}', B_addr)
    if not count:
        print('ks.asm err')
    else:
        for i in range(4):
            idc.patch_byte(B_addr + i, encoding[i])

```

执行脚本后，查看伪代码，发现修复不完全，

分析后发现是因为指向第一个代码块的索引，在 while 和 switch 组合之前初始化，并且没有跳转到默认地址，而是直接跳到分发器。  

![](https://bbs.kanxue.com/upload/attach/202305/745332_RCBR7U3S9HC8AXM.jpg)

修复初始化的时候直接跳转到第一个代码块

再次执行脚本，发现已经没有 while 和 switch 结构了，但是发现最后有一个死循环。

![](https://bbs.kanxue.com/upload/attach/202305/745332_Z7GJR4R5SF68CD6.jpg)

分析后发现是因为索引为 6 的代码块，有两个分支，脚本修复的时候，只识别出 17 这一个

于是这两个分支，在 if 判断后直接跳转到对应的代码块

![](https://bbs.kanxue.com/upload/attach/202305/745332_EC8ED9FTE332A2F.jpg)

修复所有异常的 fla 处理代码

```
import keystone
 
import idc
 
ks = keystone.Ks(keystone.KS_ARCH_ARM64, keystone.KS_MODE_LITTLE_ENDIAN)
 
 
def patch_fla(jpt_addr, index, B_addr):
    to_addr = hex((jpt_addr + idc.get_wide_dword(jpt_addr + (index - 1) * 4)) & 0xffffffff)
    print(hex(B_addr), index, to_addr)
 
    encoding, count = ks.asm(f'b {to_addr}', B_addr)
    if not count:
        print('ks.asm err')
    else:
        for i in range(4):
            idc.patch_byte(B_addr + i, encoding[i])
 
 
def find_fla(ea, end, jpt_addr, def_addr):
    # 01
    addr = ea
    while addr < end:
        mnem = idc.print_insn_mnem(addr)
        val_0 = idc.get_operand_value(addr, 0)
        if mnem == 'B' and val_0 == def_addr:
            B_addr = addr
            break
        addr += 4
    else:
        return addr
 
    #  STR             W10, [X28]
    # MOV             X28, X12
    r_28 = 28
    while ea <= addr:
        mnem = idc.print_insn_mnem(addr)
        STR_R = idc.print_operand(addr, 0)
        op_1 = idc.print_operand(addr, 1)
        if mnem == 'STR' and op_1 == f'[X{r_28}]':
            break
        if mnem == 'MOV' and STR_R == f'X{r_28}':
            r_28 = op_1[1:]
        addr -= 4
 
    # MOV             W10, #0x12
    while ea <= addr:
        mnem = idc.print_insn_mnem(addr)
        op_0 = idc.print_operand(addr, 0)
        val_1 = idc.get_operand_value(addr, 1)
        if mnem == 'MOV' and op_0 == STR_R:
            patch_fla(jpt_addr, val_1, B_addr)
            break
        addr -= 4
 
    return B_addr
 
 
def patch_fla_func(func_start, jpt_addr, def_addr):
    func_end = idc.find_func_end(func_start)
    if func_end == idc.BADADDR:
        return
 
    ea = func_start
    while ea < func_end:
        ea = find_fla(ea, func_end, jpt_addr, def_addr) + 4
 
 
jpt_addr = 0x958F4
def_addr = 0x90ed8
patch_fla_func(0x88060, jpt_addr, def_addr)
patch_fla(jpt_addr, 13, 0x88838)
patch_fla(jpt_addr, 29, 0x89D80)
patch_fla(jpt_addr, 17, 0x89FCC)

```

执行脚本后，查看伪代码，所有的分支都成功处理

![](https://bbs.kanxue.com/upload/attach/202305/745332_GE83GW3XDUHVFYS.jpg)

  

[[培训] 科锐逆向工程师培训 48 期预科班将于 2023 年 10 月 13 日 正式开班](https://bbs.kanxue.com/thread-51839.htm)

最后于 2023-6-6 08:44 被卧勒个槽编辑 ，原因： [#逆向分析](forum-161-1-118.htm) [#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [libbaiduprotect.so](javascript:void(0)) （698.16kb，80 次下载）
*   [libbaiduprotect.so.i64](javascript:void(0)) （6.13MB，60 次下载）
*   [deollvm.py](javascript:void(0)) （10.95kb，70 次下载）