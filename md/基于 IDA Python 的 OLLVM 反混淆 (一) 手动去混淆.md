> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1488350-1-1.html)

> [md]## 前言本文介绍一种根据 IDA Python 动态调试的 OLLVM 反混淆方法 (基于 x86_32 指令)## 思路根据之前大佬们给出的 ollvm 流程图, 可知 #### 控制流平坦化![](https:......

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Shocker _ 本帖最后由 Shocker 于 2021-8-4 17:03 编辑_  

前言
--

本文介绍一种根据 IDA Python 动态调试的 OLLVM 反混淆方法 (基于 x86_32 指令)

思路
--

根据之前大佬们给出的 ollvm 流程图, 可知

#### 控制流平坦化

![](https://attach.52pojie.cn/forum/202107/29/092921xgcts8n9i2ms4nrz.png)  
OLLVM 的真实逻辑在

*   序言
*   真实块
*   retn 块

那么只要在这些块的头部下断点, 记录下之前断点的位置, 最后将块中真实块的跳转的地址修改为下一个真实的块的地址即可.

#### 对所有真实块下断点

![](https://attach.52pojie.cn//forum/202108/03/150347hl80akl1a01k7aan.png?l)

#### 注意

1. 一个真实块的后继真实块可能连接着多个真实块, 也可能只有一个真实块  
![](https://attach.52pojie.cn//forum/202108/03/180445c6vhu8f8ewwrpp45.jpg?l)  
如上图所示, 这个块中将一个立即数放入栈中的一个变量里, 且该块的后继块中并无 cmov 指令, 意味着该块的后继真实块**只有一个**.

2. 若该块或其后继块中包含 cmov 指令  
![](https://attach.52pojie.cn//forum/202108/03/181851kb665pqzq691qbbq.jpg?l)  
意味着该块的后继将会有**两个真实块**, 在此例中, 该块的下一个真实块由第一个条指令 cmp 决定.  
若 [ebp+var_8] 的值小于 5, 它的下一个真实块由 ecx(0D8AAAD2D)决定, 反之由 eax(140A249C)决定.

### 找出所有真实块的后继块

用 IDA Python 脚本记录所有的断点运行的地址

```
from idaapi import *

class MyDbgHook(DBG_Hooks):
    """ Own debug hook class that implementd the callback functions """

    def dbg_bpt(self, tid, ea):
        print ("Break point at 0x%x pid=%d" % (ea, tid))
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

将脚本载入 IDA 并启动调试  
![](https://attach.52pojie.cn//forum/202108/04/120925y8wj86s6bfo3shh6.jpg?l)

F9 直到程序结束.

![](https://attach.52pojie.cn//forum/202108/04/121523pulfr98r015bswwf.jpg?l)

就可以得到所有真实块的下一个真实块.

**0x401600 -> 0x401677**  
**0x401677 (该块包含 cmov 指令) -> 0x401690,0x4016cc**  
**0x401690 -> 0x4016b3**  
**0x4016b3 -> 0x401677**

### 连接所有真实块

使用 IDA 的 Keypatch 插件就可以对代码进行 patch  
![](https://attach.52pojie.cn//forum/202108/04/165829fc9t9icyqp793t92.jpg?l)  
需要注意

*   若真实块只有一个后继的真实块, 则可以将块中末尾的跳转地址改成下一真实块的首地址.
*   若真实块中包含两个后的继真实块, 则需要根据 cmov 的类型对跳转指令进行修改.  
    例如
    
    ```
    cmovl xxx,xxx
    
    ```
    
    则末尾的跳转指令需要改成
    
    ```
    jl xxxxxxxxx
    jmp xxxxxxxxx
    
    ```
    
    结果
    --
    
    将无关的块进行 nop 后, 结果如下  
    ![](https://attach.52pojie.cn//forum/202108/04/140507wofx2x8fz62zf62r.jpg?l)
    

IDA 去混淆后 F5 的代码

![](https://attach.52pojie.cn//forum/202108/04/140559in8zsmvrniiiq0d1.jpg?l)

混淆前 F5 的代码

![](https://attach.52pojie.cn//forum/202108/04/141007p8h7ac000oa8jjz8.jpg?l)

萌新第一次发帖, 如有不足之处请大家多等指教...

本文所用示例  
链接：[https://pan.baidu.com/s/15k9KQcHChFqMgMt8mFD6sA](https://pan.baidu.com/s/15k9KQcHChFqMgMt8mFD6sA)  
提取码：mxnk![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)GuiXiaoQi 虽然不懂，但是还是要抢一楼![](https://avatar.52pojie.cn/data/avatar/000/87/41/54_avatar_middle.jpg)芽衣 是不是发错地方了？这个是 pc 的吧![](https://static.52pojie.cn/static/image/smiley/laohu/laohu34.gif)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)你就是我的阳光 看指令怎么是 x86 的![](https://static.52pojie.cn/static/image/smiley/default/3.gif)