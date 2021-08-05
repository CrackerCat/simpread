> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1488919-1-1.html)

> [md]## 前言上一篇中介绍了 IDA Python 去混淆的基本思路, 本篇将以腾讯免费御安全为例进行脚本反混淆.(以 x86_32 指令为例)## 思路首先分析出控制块的特征, 将可能是控制 ... 基于 ID......

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Shocker _ 本帖最后由 Shocker 于 2021-8-5 14:24 编辑_  

前言
--

上一篇中介绍了 IDA Python 去混淆的基本思路, 本篇将以腾讯免费御安全为例进行脚本反混淆.(以 x86_32 指令为例)

思路
--

首先分析出控制块的特征, 将可能是控制块的块去掉, 之后连接所有真实的块.

实战
--

Jadx 打开样本, 发现加载了 libshellx-super.2019.so 文件

![](https://attach.52pojie.cn//forum/202108/05/095913g8gupy2yj8yp4h14.jpg?l)  
IDA 加载 libshellx-super.2019.so, 发现 JNI_OnLoad 被混淆

![](https://attach.52pojie.cn//forum/202108/05/100617yelau2qu0z2krz2v.jpg?l)  
发现是个魔改的 OLLVM.

### 找出疑似控制块的代码

1.  ![](https://attach.52pojie.cn//forum/202108/05/101402mb2nfufa80qsdiuu.jpg?l)
2.  ![](https://attach.52pojie.cn//forum/202108/05/142032yo5f5oenoceexx0e.jpg?l)
3.  ![](https://attach.52pojie.cn//forum/202108/05/102346l1i9lhby930ghi9z.jpg?l)
4.  ![](https://attach.52pojie.cn//forum/202108/05/102158n65lrbq8f95qx86n.jpg?l)

可以分析出, 上面四种类型的块与真实逻辑毫无关系

### 去混淆脚本

对疑似控制块的块不下断点, 其余块结尾下断点

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
from idaapi import *
import keypatch

patcher=keypatch.Keypatch_Asm()

fun_offset=0x25D0 #函数地址

class MyDbgHook(DBG_Hooks):
    """ Own debug hook class that implementd the callback functions """

    def dbg_process_start(self, pid, tid, ea, name, base, size):
        print("Process started, pid=%d tid=%d  % (pid, tid, name))
        self.dbg_process_attach(pid, tid, ea, name, base, size)

    def dbg_process_exit(self, pid, tid, ea, code):
        print("Process exited pid=%d tid=%d ea=0x%x code=%d" % (pid, tid, ea, code))
        for sub_dict in self.related_dict:
            if len(self.related_dict[sub_dict])==1:
              for sub_sub_dict_key in self.related_dict[sub_dict]:
                  if idc.print_insn_mnem(sub_dict).startswith('j'):
                        disasm='jmp'+' '+hex(self.related_dict[sub_dict][sub_sub_dict_key].pop())
                        patcher.patch_code(sub_dict,disasm,patcher.syntax,True,False)

    def dbg_process_attach(self, pid, tid, ea, name, base, size):
        print("Process attach pid=%d tid=%d ea=0x%x  % (pid, tid, ea, name, base, size))
        self.pre_blcok=None
        self.ZF_flag=None
        self.related_dict=dict()
        self.block_addr_dict=dict()
        so_base=idaapi.get_imagebase()
        self.f_blocks = idaapi.FlowChart(idaapi.get_func(so_base+fun_offset), flags=idaapi.FC_PREDS)
        for block in self.f_blocks:
            start=block.start_ea
            end=idc.prev_head(block.end_ea)
            if (idc.print_insn_mnem(start)=='cmp' and idc.print_insn_mnem(idc.next_head(start)).startswith('j')) or \
                (idc.print_insn_mnem(start)=='mov' and idc.print_insn_mnem(idc.next_head(start))=='cmp' and idc.print_insn_mnem(idc.next_head(idc.next_head(start))).startswith('j')) or \
                idc.print_insn_mnem(start)=='jmp' or \
                idc.print_insn_mnem(start)=='nop' or \
                idc.print_insn_mnem(start).startswith('cmov'):
                continue
            add_bpt(end,0,BPT_SOFT)
            while start<block.end_ea:
                if idc.print_insn_mnem(start).startswith('ret'):
                    add_bpt(start,0,BPT_SOFT)
                    break
                start=idc.next_head(start)

    def dbg_bpt(self, tid, ea):
        print ("Break point at 0x%x pid=%d" % (ea, tid))

        if not self.block_addr_dict:
            so_base=idaapi.get_imagebase()
            blocks=idaapi.FlowChart(idaapi.get_func(so_base+fun_offset), flags=idaapi.FC_PREDS)
            for block in blocks:
                start=block.start_ea
                end=idc.prev_head(block.end_ea)
                self.block_addr_dict[end]=start

        if not self.pre_blcok==None:
            if self.pre_blcok in self.related_dict:
                ori_dict=self.related_dict[self.pre_blcok]
                if self.ZF_flag in ori_dict:
                    sub_set=ori_dict[self.ZF_flag]
                    sub_set.add(self.block_addr_dict[ea])
                else:
                    sub_set=set()
                    sub_set.add(self.block_addr_dict[ea])
                    ori_dict[self.ZF_flag]=sub_set
            else:
                # 不存在
                sub_set=set()
                sub_set.add(self.block_addr_dict[ea])
                sub_dict={self.ZF_flag:sub_set}
                self.related_dict.update({self.pre_blcok:sub_dict})

        self.pre_blcok=ea
        self.ZF_flag = get_reg_value("ZF")

        if idc.print_insn_mnem(ea).startswith('ret'):
            return 0
        else:
            idaapi.continue_process()
        # return values:
        #   -1 - to display a breakpoint warning dialog
        #        if the process is suspended.
        #    0 - to never display a breakpoint warning dialog.
        #    1 - to always display a breakpoint warning dialog.
        return 0

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

记录所有真实块的下一个真实块.  
我这里用一个 dict 嵌套 dict 来记录,  
大致是这样  
{0x2638:{0:xxxxx,1:xxxxx}} 这里的 0 和 1 是我对 ZF 标志的判断, xxxxx 是下一个真实块的首地址.  
得到所有真实块的流程后, 就可以进行跳转的修复

```
def dbg_process_exit(self, pid, tid, ea, code):
        print("Process exited pid=%d tid=%d ea=0x%x code=%d" % (pid, tid, ea, code))
        for sub_dict in self.related_dict:
            if len(self.related_dict[sub_dict])==1:
              for sub_sub_dict_key in self.related_dict[sub_dict]:
                  if idc.print_insn_mnem(sub_dict).startswith('j'):
                        disasm='jmp'+' '+hex(self.related_dict[sub_dict][sub_sub_dict_key].pop())
                        patcher.patch_code(sub_dict,disasm,patcher.syntax,True,False)

```

我这里仅仅只是对真实块中只有一个后继真实块.  
且结尾为 jxx 的块做了修复

### IDA 动态调试

1.  打开模拟器, 将调试文件传进模拟器, 执行
    
    ```
    adb shell /data/local/tmp/android_x86_server
    
    ```
    
2.  端口转发
    
    ```
    adb forward tcp:23946 tcp:23946
    
    ```
    
3.  打开 ddms
    
4.  调试模式启动 app
    
    ```
    adb shell am start -D -n com.shocker.jiagutest/.MainActivity
    
    ```
    
5.  IDA 先载入脚本, 后附加到 app 上  
    ![](https://attach.52pojie.cn//forum/202108/05/110142dvfs62zzss0s4s7e.jpg?l)
    
6.  jdb 方式启动
    
    ```
    jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=8700
    
    ```
    
7.  F9 运行直到运行至 retn, 继续 F9
    
8.  退出程序![](https://attach.52pojie.cn//forum/202108/05/110839mvpzuqq5xobz5pq2.jpg?l)
    
9.  将 Patch 后的 so 拖进 apk 对应的目录中, 重装 app(因为 patch 后有些块会连在一起, 所以需要再次执行脚本)
    
10.  删除所有断点, 或者重新载入 IDA, 执行一遍脚本.
    
11.  运行 ida-nop 脚本, 将没执行的块全部 nop.
    
    结果
    --
    
    去混淆后的结果  
    ![](https://attach.52pojie.cn//forum/202108/05/120334fszsa18ls8i1s1yq.jpg?l)  
    F5 后的结果  
    ![](https://attach.52pojie.cn//forum/202108/05/120505i4ttxkjpwi4spfxj.jpg?l)
    

相关示例代码见  
[https://github.com/PShocker/de-ollvm/](https://github.com/PShocker/de-ollvm/)