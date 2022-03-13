> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271853.htm)

> [原创]android so 文件攻防实战 - 某团 libmtguard.so 反混淆

计划是写一个 android 中 so 文件反混淆的系列文章，目前这是第二篇。  
第一篇：[android so 文件攻防实战 - 百度加固免费版 libbaiduprotect.so 反混淆](https://bbs.pediy.com/thread-271388.htm)  
样本在附件。我偷个懒，就不完整分析整个 so 文件了，只分析一下 JNI_OnLoad。打开 IDA，JNI_OnLoad 长这个样子：  
![](https://bbs.pediy.com/upload/attach/202203/734571_7CAPT3YNB8PKSX4.png)  
BEQ 和 BNE 是一对条件相反的跳转，这之前计算出接下来指令的地址 0x3430 赋值给 R4，loc_340C 处的代码通过 POP 指令将 R4 的值给 PC，看 0x3430 处的代码。  
![](https://bbs.pediy.com/upload/attach/202203/734571_2MFB58H8TXKG2ZM.png)  
![](https://bbs.pediy.com/upload/attach/202203/734571_EYFYQM4ZZJHTVQ2.png)  
接下来的代码很多都是这种动态计算跳转地址的。0x3432 处的指令从 0x3438 中取一个 dword 给 R0，跳到 0x3440 然后再跳到 0x407C。0x3444 处开始不是指令而是一个跳转表，索引就是 R0，值就是接下来指令地址的偏移，0x407C 开始的代码就是取这个偏移。所以我们可以写一个脚本 patch，计算出偏移，在 0x3434 处就直接跳过去。  
![](https://bbs.pediy.com/upload/attach/202203/734571_QSGSCXPWHD7A7B4.png)  
同时还会有一些跳转到 0x4094 的指令，像上面这样 0x3464 跳到 0x4094 之后会返回到 0x346C，从 0x3462 到 0x346C 这些指令我都直接 patch 成 NOP 了。  
修复脚本：

```
import keystone
from capstone import *
import idc
import ida_bytes
import subprocess
 
arch = keystone.KS_ARCH_ARM
mode = keystone.KS_MODE_THUMB
ks = keystone.Ks(arch, mode)
md = Cs(CS_ARCH_ARM, CS_MODE_THUMB)
 
def is_BLX_sub407C(ea):
    ldr_addr = ea
    ldr_flags = idc.get_full_flags(ldr_addr)
    if not idc.is_code(ldr_flags):
        return False
 
    if idc.print_insn_mnem(ldr_addr) != 'BLX':
        return False
 
    if idc.print_operand(ldr_addr, 0) != 'sub_407C':
        return False
 
    return True
 
def is_BLX_sub4094(ea):
    ldr_addr = ea
    ldr_flags = idc.get_full_flags(ldr_addr)
    if not idc.is_code(ldr_flags):
        return False
 
    if idc.print_insn_mnem(ldr_addr) != 'BLX':
        return False
 
    if idc.print_operand(ldr_addr, 0) != 'sub_4094':
        return False
 
    if idc.print_insn_mnem(ldr_addr - 2) != 'PUSH':
        return False
 
    if idc.print_insn_mnem(ldr_addr + 8) != 'POP':
        return False
 
    return True
 
def func_patch():
 
    ins_addr = idc.next_head(0)
    while ins_addr != idc.BADADDR:
 
         if is_BLX_sub407C(ins_addr):
            for i in CodeRefsTo(ins_addr, False):
                if idc.get_wide_word(i + 4) == 18112:
                    index = idc.get_wide_word(i + 6)
                    patch_qword(i + 6, 0x46C046C0)
                    idc.create_insn(i + 6)
                else:
                    index = idc.get_wide_word(i + 4)
                    patch_qword(i + 4, 0x46C046C0)
                    idc.create_insn(i + 4)
                print("i:" + hex(i))
                index = index * 4 + ins_addr + 4
                offset = ida_bytes.get_dword(index)
                target = ins_addr + 0x4 + offset
                command = "BL " + hex(target)
                print("command:" + command)
                pi = subprocess.Popen(['D:\\keystone-0.9.2-win64\\kstool.exe', 'thumb', command, \
                hex(i)], shell=True, stdout=subprocess.PIPE)
                output = pi.stdout.read()
                ins = str(output[-15:-4])[2:-1]
                ins = ins.split(" ")
                ins = "0x" + ins[3] + ins[2] + ins[1] + ins[0]
                print("ins:" + ins)
                patch_dword(i, int(ins, 16))
 
         if is_BLX_sub4094(ins_addr):
            patch_dword(ins_addr - 2, 0xbf00)
            patch_dword(ins_addr, 0xbf00)
            patch_dword(ins_addr + 2, 0xbf00)
            patch_dword(ins_addr + 4, 0xbf00)
            patch_dword(ins_addr + 6, 0xbf00)
            patch_dword(ins_addr + 8, 0xbf00)
 
         ins_addr = idc.next_head(ins_addr)
 
func_patch()

```

修复结束之后跟踪指令，0x3434 处的代码跳到了 0x345A，0x345A 处的代码主要逻辑是在 0x34C6 跳到 0x34EC 通过 0x34EC 动态计算地址跳到了 0x3520，0x3524 处的代码跳到了 0x365C，0x366E 处的代码跳到了 0x35A8，0x35A8 是 JNI_OnLoad 的主要逻辑。  
![](https://bbs.pediy.com/upload/attach/202203/734571_VY8JQ4NDYEQTZN8.png)  
如上图所示，0x35A8 基本就干了两件事：FindClass(com/meituan/android/common/mtguard/NBridge) 和 RegisterNatives(com/meituan/android/common/mtguard/NBridge, main, 1)。  
![](https://bbs.pediy.com/upload/attach/202203/734571_CN2FX8H8952CTYS.png)  
sub_3680 里面调用 sub_36B4 通过异或解密了一些字符串，上图我已经把解密之后的结果 patch 进去了。  
当然用 unidbg 是可以直接跑 JNI_OnLoad 的：  
![](https://bbs.pediy.com/upload/attach/202203/734571_93E427YN8B6DZCZ.png)  
不过这个系列文章主要是想讨论一下 so 混淆和反混淆，有精力的话还可以对这个 so 进行完整分析，混淆应该没有啥新的了。

[【公告】“雪花” 创作激励计划，3 月 1 日正式开启！](https://bbs.pediy.com/thread-271637.htm)

[#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [libmtguard.so](javascript:void(0)) （662.75kb，5 次下载）