> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1496575-1-1.html)

> [md]## 前言本文以萌新的角度来分析 VMP 壳, 不涉及到 VM 的任何概念, 算是 VMP 的入门贴.#### 分析环境: windows 10+IDA 7.5+VS 2019+VMProtect_Ultimat......

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Shocker

前言
--

本文以萌新的角度来分析 VMP 壳, 不涉及到 VM 的任何概念, 算是 VMP 的入门贴.

#### 分析环境:

windows 10+IDA 7.5+VS 2019+  
VMProtect_Ultimate_v3.5.0_x32_Build_1213_Retail_Licensed

环境搭建
----

声明裸函数 nake_main, 这个函数就是要加 vm 的函数

```
__declspec(naked) void nake_main() {
    _asm mov eax, 0x12345678;
    _asm ret;
}
int main()
{
    nake_main();
}

```

函数逻辑很简单, 就是给 eax 赋值 0x12345678 然后返回.  
VS2019 编译,  
加壳  
![](https://attach.52pojie.cn/forum/202108/19/112608l7k7zwp7pe337j3j.jpg)

**vmp0.jpg** _(53.47 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ1OXw2NzFjYjNkOHwxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:26 上传

可以看到, 整个函数已经被 vm 了.  
![](https://attach.52pojie.cn/forum/202108/19/112744tki360kb58jnc5j8.jpg)

**4.jpg** _(60.69 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ2MHwyNWY1OGMwNnwxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:27 上传

### 记录指令

用 IDA trace 脚本跟踪函数进虚拟机直到出虚拟机所执行的指令, 并将指令的二进制记录到文件中. **注意下断点**  
![](https://attach.52pojie.cn/forum/202108/19/112915leriqqgq1382qtni.jpg)

**script.jpg** _(58.09 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ2MXxkYzJhY2ZhN3wxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:29 上传

![](https://attach.52pojie.cn/forum/202108/19/112930n51sf7r8788v1117.jpg)

**trace.jpg** _(498.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ2Mnw4Mjc0MzlmNHwxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:29 上传

  
**注意要替换所有的 call 指令为 call +5, 把 retn 指令替换为 lea esp,[esp+4].**

```
#---------------------------------------------------------------------
# Debug notification hook test
#
# This script start the executable and steps through the first five
# instructions. Each instruction is disassembled after execution.
#
# Original Author: Gergely Erdelyi <gergely.erdelyi@d-dome.net>
#
# Maintained By: IDAPython Team
#
#---------------------------------------------------------------------
import idc
from idaapi import *
import binascii
import struct

from capstone import *
md=Cs(CS_ARCH_X86,CS_MODE_32)

class MyDbgHook(DBG_Hooks):
    """ Own debug hook class that implementd the callback functions """

    def dbg_process_start(self, pid, tid, ea, name, base, size):
        print("Process started, pid=%d tid=%d  % (pid, tid, name))
        self.asmfile=open('dump.asm','wb') #保存记录指令的文件
        self.record_count=0

    def dbg_process_exit(self, pid, tid, ea, code):
        print("Process exited pid=%d tid=%d ea=0x%x code=%d" % (pid, tid, ea, code))
        if not self.asmfile ==None:
            self.asmfile.close()

    # def dbg_library_unload(self, pid, tid, ea, info):
    #     print("Library unloaded: pid=%d tid=%d ea=0x%x info=%s" % (pid, tid, ea, info))
    #     return 0

    # def dbg_process_attach(self, pid, tid, ea, name, base, size):
    #     print("Process attach pid=%d tid=%d ea=0x%x  % (pid, tid, ea, name, base, size))

    # def dbg_process_detach(self, pid, tid, ea):
    #     print("Process detached, pid=%d tid=%d ea=0x%x" % (pid, tid, ea))
    #     return 0

    # def dbg_library_load(self, pid, tid, ea, name, base, size):
    #     print ("Library loaded: pid=%d tid=%d  % (pid, tid, name, base))

    def dbg_bpt(self, tid, ea):
        print("0x%x     %s" % (ea, GetDisasm(ea)))
        codelen=get_item_size(ea)
        self.record_count=self.record_count+codelen
        b=get_bytes(ea,codelen)
        self.asmfile.write(b)
        self.asmfile.flush()
        # return values:
        #   -1 - to display a breakpoint warning dialog
        #        if the process is suspended.
        #    0 - to never display a breakpoint warning dialog.
        #    1 - to always display a breakpoint warning dialog.
        return 0

    # def dbg_suspend_process(self):
    #     print ("Process suspended")

    # def dbg_exception(self, pid, tid, ea, exc_code, exc_can_cont, exc_ea, exc_info):
    #     print("Exception: pid=%d tid=%d ea=0x%x exc_code=0x%x can_continue=%d exc_ea=0x%x exc_info=%s" % (
    #         pid, tid, ea, exc_code & idaapi.BADADDR, exc_can_cont, exc_ea, exc_info))
    #     # return values:
    #     #   -1 - to display an exception warning dialog
    #     #        if the process is suspended.
    #     #   0  - to never display an exception warning dialog.
    #     #   1  - to always display an exception warning dialog.
    #     return 0

    def dbg_trace(self, tid, ea):
        print("0x%x     %s" % (ea, GetDisasm(ea)))
        if idc.print_insn_mnem(ea).startswith('j'): #不记录所有的跳转指令
            return 0

        if idc.print_insn_mnem(ea) == 'retn':#把retn 替换为lea esp,[esp+4]
            code=b'\x8D\x64\x24\x04' #lea esp,[esp+4]
            self.asmfile.write(code)
            self.asmfile.flush()
            self.record_count=self.record_count+len(code)
            return 0 
        if idc.print_insn_mnem(ea) == 'call':#把call 替换为call +5
            fix_addr=0
            mnemonic=struct.pack('B',idc.get_wide_byte(ea))
            op=struct.pack('i',fix_addr)
            call_asm=mnemonic+op
            self.asmfile.write(call_asm)
            self.asmfile.flush()
            self.record_count=self.record_count+get_item_size(ea)
            return 0 
        for addr in range(ea,idc.next_head(ea)):
            b=struct.pack('B',idc.get_wide_byte(addr))
            self.asmfile.write(b)
            self.asmfile.flush()
        self.record_count=self.record_count+get_item_size(ea)

        # eip = get_reg_value("EIP")
        # print("0x%x %s" % (eip, GetDisasm(eip)))
        # print("Trace tid=%d ea=0x%x" % (tid, ea))
        # return values:
        #   1  - do not log this trace event;
        #   0  - log it
        return 0

    # def dbg_step_into(self):
    #     eip = get_reg_value("EIP")
    #     print("0x%x %s" % (eip, GetDisasm(eip)))

    # def dbg_run_to(self, pid, tid=0, ea=0):
    #     print ("Runto: tid=%d" % tid)
    #     idaapi.continue_process()

    # def dbg_step_over(self):
    #     eip = get_reg_value("EIP")
    #     print("0x%x %s" % (eip, GetDisasm(eip)))
    #     self.steps += 1
    #     if self.steps >= 5:
    #         request_exit_process()
    #     else:
    #         request_step_over()

# Remove an existing debug hook
try:
    if debughook:
        print("Removing previous hook ...")
        debughook.unhook()
except:
    pass

# Install the debug hook
debughook = MyDbgHook()
debughook.hook()
debughook.steps = 0

# Stop at the entry point
ep = get_inf_attr(INF_START_IP)
request_run_to(ep)

# Step one instruction
request_step_over()

# Start debugging
run_requests()

```

将记录后的指令文件用 CFF_Explorer 添加到原文件里  
![](https://attach.52pojie.cn/forum/202108/19/113034le666otkendcp1t1.jpg)

**cff.jpg** _(388.92 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ2M3w1NTE4OTk5ZHwxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:30 上传

IDA 打开找到对应地址, 运行去混淆脚本  
[x86 反混淆 IDA Python 脚本 (一)](https://www.52pojie.cn/thread-1491068-1-1.html)  
将混淆指令去除后再合并, 即可看到 vm 的代码  
![](https://attach.52pojie.cn/forum/202108/19/113351rvixrtdjfs2s22ay.jpg)

**vvmp.jpg** _(61.09 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ2NXw1NGRmMmMzNXwxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:33 上传

分析指令
----

```
seg005:0048D000                 push    8FF9032Bh
seg005:0048D005                 call    $+5
seg005:0048D00A                 push    eax
seg005:0048D00B                 push    edx
seg005:0048D00C                 pushf
seg005:0048D00D                 push    edi
seg005:0048D00E                 push    esi
seg005:0048D00F                 push    ebp
seg005:0048D010                 push    ecx
seg005:0048D011                 push    ebx
seg005:0048D012                 mov     edx, 0
seg005:0048D017                 push    edx             ; 保存寄存器
seg005:0048D018                 mov     edi, [esp+28h]  ; [esp+28h]就是0x8FF9032B,即进入虚拟机时push值,给edi
seg005:0048D01C                 ror     edi, 3
seg005:0048D01F                 add     edi, 4F581DEFh
seg005:0048D025                 ror     edi, 1
seg005:0048D027                 bswap   edi
seg005:0048D029                 rol     edi, 2
seg005:0048D02C                 add     edi, 55C970B6h
seg005:0048D032                 add     edi, edx        ; 对edi经过一系列运算,最后edi指向一片内存区域
seg005:0048D034                 mov     esi, esp        ; esi指向了保存所有寄存器的位置,指向上面push edx的edx的值
seg005:0048D036                 sub     esp, 0C0h       ; 将esp减去C0,为后面的mov [esp+ecx],eax,保留空间
seg005:0048D03C                 mov     ebx, edi
seg005:0048D03E                 mov     eax, 0
seg005:0048D043                 sub     ebx, eax        ; 这个ebx在后面每一个代码段里都有用
seg005:0048D045                 lea     ebp, loc_46ACC5 ; 这里ebp仅作跳转作用,即用来找到下一个要跳转的地址

```

这是进入 vm 的起始代码, 这里主要是用 push 将进入虚拟机时原始的 **eax,edx,eflags,edi,esi,ebp,ecx,ebx** 保存到栈里, 之后根据 push 的值 (0x8FF9032B) 做一系列运算给 edi, 使 edi 指向一片内存区域.  
![](https://attach.52pojie.cn/forum/202108/19/113128kp2luuz77lql7qtl.jpg)

**edi.jpg** _(43.37 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMyNDQ2NHw4NzM2ZmJmYXwxNjI5MzY0MzEwfDB8MTQ5NjU3NQ%3D%3D&nothumb=yes)

2021-8-19 11:31 上传

之后对 ebx 一系列操作,**ebx 在之后的代码段都会做一次更新操作.**

```
seg005:0048D04B                 mov     eax, [edi]      ; 从edi的内存读取4字节
seg005:0048D04D                 lea     edi, [edi+4]    ; edi+4
seg005:0048D053                 xor     eax, ebx
seg005:0048D055                 dec     eax
seg005:0048D056                 neg     eax
seg005:0048D058                 dec     eax
seg005:0048D059                 bswap   eax
seg005:0048D05B                 xor     ebx, eax        ; 对ebx操作
seg005:0048D05D                 add     ebp, eax
seg005:0048D05F                 push    ebp
seg005:0048D060                 lea     esp, [esp+4]    ; 这里本来是retn,这一段代码只对ebx做了操作

```

这个代码段更新了本轮 ebx

```
seg005:0048D064                 mov     eax, [esi]      ; esi指向原始保存了所有寄存器的位置
seg005:0048D066                 add     esi, 4          ; esi指向下一个原始push的位置
seg005:0048D06C                 movzx   ecx, byte ptr [edi]
seg005:0048D06F                 lea     edi, [edi+1]    ; edi+1
seg005:0048D075                 xor     cl, bl
seg005:0048D077                 dec     cl
seg005:0048D079                 ror     cl, 1
seg005:0048D07B                 sub     cl, 5Dh ; ']'
seg005:0048D07E                 not     cl
seg005:0048D080                 xor     bl, cl          ; 更新ebx
seg005:0048D082                 mov     [esp+ecx], eax  ; 将原来保存的寄存器搬运到[esp+ecx]里,ecx是由ebx和[edi]解密出来
seg005:0048D085                 mov     ecx, [edi]      ; 从这里到结束,其作用是对ebx又进行了一系列操作
seg005:0048D087                 add     edi, 4          ; edi+4
seg005:0048D08D                 xor     ecx, ebx
seg005:0048D08F                 ror     ecx, 1
seg005:0048D091                 lea     ecx, [ecx-43A06B23h]
seg005:0048D097                 neg     ecx
seg005:0048D099                 xor     ecx, 46ED52DEh
seg005:0048D09F                 xor     ebx, ecx        ; 更新ebx
seg005:0048D0A1                 add     ebp, ecx
seg005:0048D0A3                 push    ebp
seg005:0048D0A4                 lea     esp, [esp+4]    ; 相当于retn结束

```

这一段代码的作用是把原始 push 的寄存器保存到 [esp+ecx] 里, ecx 的值取决于 edi 和上一个代码段里的 ebx. 然后**更新本轮的 ebx.**  
之后该代码段一直循环直到 **esi 的值等于未进虚拟机的 esp 的值 (保存完所有原始寄存器) 结束.**

```
seg005:0048D332                 mov     ecx, [edi]      ; esi等于未进入虚拟机的esp,说明原始寄存器已经全部被保存了
seg005:0048D334                 lea     edi, [edi+4]    ; edi+4
seg005:0048D33A                 xor     ecx, ebx
seg005:0048D33C                 sub     ecx, 607554FFh
seg005:0048D342                 not     ecx
seg005:0048D344                 neg     ecx
seg005:0048D346                 inc     ecx             ; ecx=0x12345678,这个值是根据[edi]和上一轮的ebx解密出来
seg005:0048D347                 xor     ebx, ecx
seg005:0048D349                 sub     esi, 4
seg005:0048D34F                 mov     [esi], ecx      ; 结束 ecx=0x12345678

```

这段代码解密出操作数 0x12345678 放到 [esi] 里, 刚好覆盖了第一个 push 的值.

```
seg005:0048D3B3                 movzx   eax, byte ptr [edi] ; 根据[edi]的值去读取[esp+eax]的值
seg005:0048D3B6                 add     edi, 1
seg005:0048D3BC                 xor     al, bl
seg005:0048D3BE                 inc     al
seg005:0048D3C0                 not     al
seg005:0048D3C2                 add     al, 14h
seg005:0048D3C4                 xor     al, 4Eh
seg005:0048D3C6                 xor     bl, al
seg005:0048D3C8                 mov     ecx, [esp+eax]  ; 从[esp+eax]读取寄存器
seg005:0048D3CB                 sub     esi, 4          ; esi减去4
seg005:0048D3D1                 mov     [esi], ecx      ; 放到esi中,其实是把[esp+eax]保存到原始栈中
seg005:0048D3D3                 mov     eax, [edi]
seg005:0048D3D5                 add     edi, 4
seg005:0048D3DB                 xor     eax, ebx
seg005:0048D3DD                 add     eax, 7DC64D28h
seg005:0048D3E2                 neg     eax
seg005:0048D3E4                 bswap   eax
seg005:0048D3E6                 not     eax
seg005:0048D3E8                 xor     ebx, eax        ; 更新ebx
seg005:0048D3EA                 add     ebp, eax
seg005:0048D3EC                 push    ebp
seg005:0048D3ED                 lea     esp, [esp+4]    ; retn

```

这个代码段将之前存储到 [esp+eax] 的值全部搬运到原始栈上. 为虚拟机退出做准备.

```
seg005:0048D5A0                 mov     esp, esi        ; 最后将esi给esp
seg005:0048D5A2                 pop     ebx
seg005:0048D5A3                 pop     ecx
seg005:0048D5A4                 pop     ebp
seg005:0048D5A5                 pop     esi
seg005:0048D5A6                 pop     edi
seg005:0048D5A7                 popf
seg005:0048D5A8                 pop     edx
seg005:0048D5A9                 pop     eax             ; 退出虚拟机
seg005:0048D5AA                 retn

```

退出虚拟机.

结论
--

*   进入虚拟机时 push 的值最终会被解密出指向一片内存区域, 并根据该值将原始寄存器保存到不同的 [esp+eax] 位置和更新 ebx 值.
*   操作数的解密需要依赖上一轮的 ebx 值.
*   操作数的解密的过程在完成了所有寄存器的保存后执行.
*   虚拟机退出前会把 [esp+eax] 的值保存到 [esi] 里, 最后通过 pop 返回给寄存器

相关示例代码见  
[https://github.com/PShocker/vmp3_test](https://github.com/PShocker/vmp3_test)

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Domado 看见 vmp，直接拖入垃圾桶，哈哈