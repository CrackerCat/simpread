> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288865.htm)

> 教你玩转 ida 反编译（修改 ida microcode）

前言
==

看到了 GhHei 大佬的文章 [OLLVM 扁平化还原—更优雅的解法：IDA Hex-Rays Microcode](https://bbs.kanxue.com/thread-288691.htm)，迫不及待得尝试了一下，然后想对一些变种的 OLLVM 进行处理，由于大佬使用的方法没法添加新的分支，所以自己琢磨了一周，发现网上关于 ida microcode 的文章极少。这里查阅了 [D810 插件](elink@908K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6B7L8%4W2V1L8#2)9J5c8X3b7^5x3e0l9`.)和 [hrtng 插件](elink@a73K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6w2j5i4y4H3k6i4u0K6K9%4W2x3j5h3u0Q4x3V1k6Z5M7Y4c8F1k6H3`.`.)的源码才找到添加新分支的办法，最后也是写出了处理 OLLVM 的脚本。

修改反编译的两种方法
==========

直接获取 Pseudocode-A 中的 mba
------------------------

通过 find_widget 函数查找 Pseudocode-A 界面，这里拿到的是 TWidget 再使用 get_widget_vdui 拿到 vdui_t，从 vdui_t 中可以拿到 mba。

```
def find_pseudocode_view ( func_ea, auto_open=False ):
    widget  = idaapi.find_widget( "Pseudocode-A" )
    if widget and idaapi.get_widget_type( widget ) == idaapi.BWN_PSEUDOCODE :
        vu  = idaapi.get_widget_vdui( widget )
        if vu and vu.cfunc.entry_ea == func_ea :
            return vu
 
    if auto_open :
        idaapi.open_pseudocode( func_ea, idaapi.OPF_REUSE )
        return find_pseudocode_view( func_ea, False )
 
    return None
 
vu: ida_hexrays.vdui_t= find_pseudocode_view(0x113CC, True)
mba: ida_hexrays.mba_t = vu.cfunc.mba

```

这种方法拿到的 mba.maturity（成熟度） 是 8，已经是经过了所有优化之后的情况，没有办法添加新的块和删除块。

使用 ida hook 拦截中间的优化过程
---------------------

ida python 里面有个 Hexrays_Hooks 类，这个类里面可以拦截优化过程进度。根据类中的函数分别有以下进度：  
● stkpnts：SP change points have been calculated.（SP 变化点已计算。）（mba maturity: 0）  
● prolog：Prolog analysis has been finished.（前缀分析已完成。）（mba maturity: 0）  
● microcode：Microcode has been generated.（微码已生成。）（mba maturity: 0）  
● preoptimized：Microcode has been preoptimized.（微码已预先优化。）（mba maturity: 1）  
● locopt：Basic block level optimization has been finished.（基本块级优化已完成。）（mba maturity: 2）  
● calls_done：All calls have been analyzed.（所有调用都已分析。）（mba maturity: 3）  
● prealloc：Local variables: preallocation step begins.（局部变量：预分配步骤开始。）（mba maturity: 5、6）  
● glbopt：Global optimization has been finished. If microcode is modified, MERR_LOOP must be returned. It will cause a complete restart of the optimization.（全局优化已完成。如果微代码被修改，必须返回 MERR_LOOP。这将导致优化完全重启。）（mba maturity: 6）

```
class HexraysDecompilationHook(Hexrays_Hooks):
    def __init__(self):
        super().__init__()
 
    def prolog(self, mba: mbl_array_t, fc, reachable_blocks, decomp_flags) -> "int":
        print(f"prolog mba maturity: {mba.maturity}")
        return 0
 
    def glbopt(self, mba: mbl_array_t) -> "int":
        print(f"glbopt mba maturity: {mba.maturity}")
        return MERR_OK
     
    def locopt(self, mba: mba_t) -> int:
        print(f"locopt mba maturity: {mba.maturity}")
        return 0
 
    def prealloc(self, mba: mba_t) -> int:
        print(f"prealloc mba maturity: {mba.maturity}")
        return 0
    def calls_done(self, mba: mba_t):
        print(f"calls_done mba maturity: {mba.maturity}")
        return 0
     
    def microcode(self, mba: mba_t):
        print(f"microcode mba maturity: {mba.maturity}")
        return 0
 
    def stkpnts(self, mba: mba_t, _sps:ida_frame.stkpnts_t):
        print(f"stkpnts mba maturity: {mba.maturity}")
        return 0
 
    def preoptimized(self, mba: mba_t):
        print(f"preoptimized mba maturity: {mba.maturity}")
        return 0
     
testHook = HexraysDecompilationHook()
print(testHook.hook())

```

使用 hook 会在 F5 的时候都会执行，需要注意多个 hook 同一个优化进度可能会导致冲突。

在块中处理指令
=======

在 ida mblock 中的指令是以双向链表进行连接的，mblock.head 和 mblock.tail 分别指向第一条指令和最后一条指令，指令使用的类型为 minsn_t，以下为分析：  
● minsn_t.opcode：操作码，如 goto、mov、jz 等  
● minsn_t.d：目标操作数，可能是变量或寄存器  
● minsn_t.l：左操作数，可能是变量、寄存器、常数、目标块  
● minsn_t.r：右操作数，可能是变量或寄存器  
举个例子：

```
mov    #1.8, x8.8

```

mov 是 opcode，#1.8 是左操作数，x8.8 是右操作数

```
goto   @34

```

goto 是 opcode，@34 是左操作数（块编号）

```
sub    x0.8{34}, %var_90.8{3}, x8.8

```

sub 是 opcode，x0.8{34} 是左操作数，%var_90.8{3} 是右操作数，x8.8 是目标操作数  
这些操作数使用的类是 mop_t：  
● mop_t.t：操作数类型，是个 int，后面会有说明  
● mop_t.size：操作数大小，一般是 1、2、4、8 或者 NOSIZE  
● mop_t.r：微码寄存器，类型为 mreg_t  
● mop_t.nnn：一个整数常量，类型为 mnumber_t  
● mop_t.s：栈上变量的引用，类型为 stkvar_ref_t  
● mop_t.b：一般用于 goto 之后的 block_id  
● mop_t.l：局部变量的引用，类型为 lvar_ref_t  
这里的操作数类型的宏定义如下：  
● mop_r：寄存器，在 maturity 为 MMAT_LVARS 之前使用，也就是说在 MMAT_LVARS 之前使用的是 mop_t.r，在其之后使用 mop_t.l  
● mop_n：立即数，对应 mop_t.nnn  
● mop_b：基本块（mblock），对应 mop_t.b  
● mop_l：局部变量，对应 mop_t.l  
● mop_S：局部栈变量，对应 mop_t.s

修改指令
----

举个例子：

```
2.0 ; 1WAY-BLOCK 2 FAKE INBOUNDS: 1 OUTBOUNDS: 4 [START=1143C END=11440] MINREFS: STK=C0/ARG=2C0, MAXBSP: 0
2.0 ; DEF: w9.4
2.0 ; DNU: w9.4
2.0 ; VALRANGES: x20.8:==0
2.0 mov    #0x24279280.4, w9.4     ; 1143C split4 u=           d=w9.4
2.1 goto   @4                      ; 1143C u=
2.1

```

这是第 2 个块（成熟度为 MMAT_GLBOPT3），可以通过 mba.get_mblock(2) 拿到该块的数据，假如我要修改 #0x24279280.4 这个立即数，首先使用 mblock.head 拿到第一条指令数据，然后这个立即数是左操作数那他对应的就是 minsn.l.nnn，在 mnumber_t 里面成员 value 可以拿到具体数值，也就是说 minsn.l.nnn.value 就是 0x24279280，此时可以直接给 minsn.l.nnn.value 赋值即可改变。代码如下：

```
from ida_hexrays import *
def change_minsn(mba: mba_t):
    mblock:mblock_t = mba.get_mblock(2)
    minsn:minsn_t = mblock.head
    minsn.l.nnn.value = 1
    mba.verify(True)
 
class HexraysDecompilationHook(Hexrays_Hooks):
    def __init__(self):
        super().__init__()
 
    def glbopt(self, mba: mbl_array_t) -> "int":
        print(f"glbopt mba maturity: {mba.maturity}")
        change_minsn(mba)
        return MERR_OK
         
testHook = HexraysDecompilationHook()
print(testHook.hook())

```

改变前的反编译效果：  
![](https://bbs.kanxue.com/upload/attach/202510/979192_8M45DHVKN2VZ9VZ.webp)  
加载脚本后，改变后的反编译效果  
![](https://bbs.kanxue.com/upload/attach/202510/979192_QDNPTDCMAWDFCZ9.webp)

新增指令
----

同样使用上面的例子，现在目标是在两行指令之间添加一行指令对 w9.4 里面的值再次处理，新增一个减法指令，代码如下：

```
from ida_hexrays import *
def add_new_minsn(mba: mba_t):
    mblock:mblock_t = mba.get_mblock(2)
     
    # 新建左操作数，使用第一条指令的目标寄存器
    new_mop_l = mop_t()
    new_mop_l.t = mop_r
    new_mop_l.r = mblock.head.d.r
    new_mop_l.size = mblock.head.d.size
 
    # 新建右操作数，新建一个常数
    new_mop_r = mop_t()
    new_mop_r.t = mop_n
    new_mop_r.nnn = mnumber_t(0x24279280)
    new_mop_r.size = mblock.head.l.size
 
    # 新建目标操作数，使用第一条指令寄存器
    new_mop_d = mop_t()
    new_mop_d.t = mop_r
    new_mop_d.r = mblock.head.d.r
    new_mop_d.size = mblock.head.d.size
 
    # 新建指令，设定操作码和所有操作数
    new_minsn = minsn_t(mblock.tail.ea)
    new_minsn.opcode = m_sub
    new_minsn.d = new_mop_d
    new_minsn.l = new_mop_l
    new_minsn.r = new_mop_r
 
    # 在参数二指令的后面添加新指令
    mblock.insert_into_block(new_minsn, mblock.head)
    # 用于打印mblock内容
    vp = vd_printer_t()
    mblock._print(vp)
    mba.verify(True)
 
class HexraysDecompilationHook(Hexrays_Hooks):
    def __init__(self):
        super().__init__()
 
    def glbopt(self, mba: mbl_array_t) -> "int":
        print(f"glbopt mba maturity: {mba.maturity}")
        change_minsn(mba)
        return MERR_OK
         
testHook = HexraysDecompilationHook()
print(testHook.hook())

```

mblock 打印出来的内容如下：

```
2.0 mov    #0x24279280.4, w9.4     ; 1143C split4 u=           d=w9.4
2.1 sub    w9.4, #0x24279280.4, w9.4 ; 1143C u=w9.4       d=w9.4
2.2 goto   @4                      ; 1143C u= 

```

再查看反编译后的内容：  
![](https://bbs.kanxue.com/upload/attach/202510/979192_PKA2VT4PCQ5EHHT.webp)

修改或新增跳转指令
---------

修改或者新增跳转指令，不仅是对指令操作，还有对 mblock 的处理，包括 mblock 的上下文关系还有 mblock 的属性等。重新观察一下上面的例子：

```
2.0 ; 1WAY-BLOCK 2 FAKE INBOUNDS: 1 OUTBOUNDS: 4 [START=1143C END=11440] MINREFS: STK=C0/ARG=2C0, MAXBSP: 0
2.0 ; DEF: w9.4
2.0 ; DNU: w9.4
2.0 ; VALRANGES: x20.8:==0
2.0 mov    #0x24279280.4, w9.4     ; 1143C split4 u=           d=w9.4
2.1 goto   @4                      ; 1143C u=
2.1

```

在第一行里有 1WAY-BLOCK、INBOUNDS: 1、OUTBOUNDS: 4 这三个项是我们需要注意的，其中 INBOUNDS 代表会有什么块跳转到本块，OUTBOUNDS 代表本块会跳转到什么块。那么 1WAY-BLOCK 是什么呢？我们来看下一个例子：

```
6.0 ; 2WAY-BLOCK 6 INBOUNDS: 5 OUTBOUNDS: 7 42 [START=11494 END=114A4] MINREFS: STK=C0/ARG=2C0, MAXBSP: 0
6.0 ; USE: w8.4
6.0 ; VALRANGES: w8.4:(==D61A7D2|==24279280), %0xC.4:(==24279280|==D6015831)
6.0 jz     w8.4, #0xD61A7D2.4, @42 ; 114A0 inverted_jx u=w8.4
6.0

```

可以发现，当 OUTBOUNDS 有 2 个的时候他是 2WAY-BLOCK，这里代表两个分支。

### 修改 goto

从简单一点的入手，我们可以修改 mblock2 的 goto 让他直接跳转到结尾，代码如下：

```
def modify_edge(mba: mba_t, cur_block_id: int, new_block_id: int = 0, old_block_id: int = 0):
    cur_block: mblock_t = mba.get_mblock(cur_block_id)
    new_block: mblock_t = mba.get_mblock(new_block_id)
    old_block: mblock_t = mba.get_mblock(old_block_id)
 
    cur_block.succset._del(old_block_id)
    old_block.predset._del(cur_block_id)
 
    cur_block.succset.push_back(new_block_id)
    new_block.predset.push_back(cur_block_id)
     
def change_goto(mba: mba_t):
    mblock = mba.get_mblock(2)
    minsn:minsn_t = mblock.tail
    minsn.l.b = 69
    modify_edge(mba, cur_block_id=2, old_block_id=4, new_block_id=69)
    mba.verify(True)

```

mblock.succset 是后继块，mblock.predset 是前驱块，这个关联需要手动处理。  
改完之后，这个 if else 直接到最后了。  
![](https://bbs.kanxue.com/upload/attach/202510/979192_S54G3QE2MGW6QWN.webp)

### 修改条件跳转

这会还是使用 mblock2，把 goto 改成 jz 跳转，这里注意需要添加一个分支和修改 1WAY-BLOCK，代码如下：

```
def change_jz(mba: mba_t):
    mblock:mblock_t = mba.get_mblock(2)
    minsn:minsn_t = mblock.tail
 
    new_mop_l = mop_t()
    new_mop_l.t = mop_r
    new_mop_l.r = mblock.head.d.r
    new_mop_l.size = mblock.head.d.size
 
    new_mop_r = mop_t()
    new_mop_r.t = mop_n
    new_mop_r.nnn = mnumber_t(0x24279280)
    new_mop_r.size = mblock.head.l.size
 
    new_mop_d = mop_t()
    new_mop_d.t = mop_b
    new_mop_d.b = 69
    new_mop_d.size = NOSIZE
 
    minsn.opcode = m_jz
    minsn.l = new_mop_l
    minsn.r = new_mop_r
    minsn.d = new_mop_d
 
    modify_edge(mba, cur_block_id=2, new_block_id=3, old_block_id=4)
    modify_edge(mba, cur_block_id=2, new_block_id=69)
    mblock.type = BLT_2WAY
    mba.verify(True)

```

mblock.type 实际上就是 1WAY-BLOCK，这里设置为 BLT_2WAY 也就是 2WAY-BLOCK  
![](https://bbs.kanxue.com/upload/attach/202510/979192_7H8EJGFW56TSJGF.webp)  
**特别需要注意的是，OUTBOUNDS 是有顺序的，例如 jz 的话第一个 id 是下一个 mblock，第二个才是 jz 跳转过去的 mblock**

新增块（mblock）
===========

观察 ida 的微码可以发现，每个块的结尾都是一个跳转指令，那么如果我们想要添加一个分支判断也就是 jz 加 goto，那么我们需要新增一个块给 goto 使用。那么刚刚已经修改了 mblock2 的 goto 为 jz，这次新增一个 mblock3 写一个 goto 作为一个分支。代码如下：

```
def add_new_goto(mba: mba_t, cur_block_id: int, new_block_id: int, minsn_ea: int = None):
    new_mop = mop_t()
    new_mop.t = mop_b
    new_mop.b = new_block_id
    new_mop.size = NOSIZE
 
    cur_block: mblock_t = mba.get_mblock(cur_block_id)
    if minsn_ea:
        new_goto = minsn_t(minsn_ea)
    else:
        new_goto = minsn_t(cur_block.tail.ea)
    new_goto.opcode = m_goto
    new_goto.l = new_mop
 
    if cur_block.tail:
        cur_block.insert_into_block(new_goto, cur_block.tail)
    else:
        cur_block.head = new_goto
        cur_block.tail = new_goto
         
def add_new_block(mba:mba_t):
    mblock:mblock_t = mba.get_mblock(2)
    new_mblock:mblock_t = mba.insert_block(mblock.serial + 1)
    add_new_goto(mba, cur_block_id=new_mblock.serial, new_block_id=5, minsn_ea=mblock.tail.ea)
    modify_edge(mba, cur_block_id=new_mblock.serial, new_block_id=5)
    # 这里调整上一个块的后继块
    modify_edge(mba, cur_block_id=mblock.serial, new_block_id=3, old_block_id=3 + 1)
    # 调整顺序所以需要删除再push
    modify_edge(mba, cur_block_id=mblock.serial, new_block_id=70, old_block_id=70)
    new_mblock.make_lists_ready()
    new_mblock.type = BLT_1WAY
    new_mblock.flags |= MBL_GOTO
    # 设定地址
    new_mblock.start = 0x123456
    new_mblock.end = 0x123458
    mba.verify(True)
 
class HexraysDecompilationHook(Hexrays_Hooks):
    def __init__(self):
        super().__init__()
        self.mba_change = False
 
    def glbopt(self, mba: mbl_array_t) -> "int":
        print(f"glbopt mba maturity: {mba.maturity}")
        if self.mba_change == False:
            change_jz(mba)
            add_new_block(mba)
            self.mba_change = True
            return MERR_LOOP
        else:
            return MERR_OK

```

新增块有几个需要注意的点：

1.  插入块后，其他块的前驱后继连接都会自行改变，不用手动修改。
2.  新的块里面没有 head 和 tail，新增指令时不能使用 insert_into_block 而是直接赋给 head 和 tail。
3.  需要设定 mblock 的起始地址和结束地址，这里地址不一定是真实地址，但是结束地址必须大于起始地址。
4.  最重要的是在 hook 的时候，如果增删块的情况下需要返回 MERR_LOOP 使其重新优化，否则会报错 50346。（你问我这个数字为什么记这么清楚，我只能说就是修这个报错修了两天）

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#工具脚本](forum-161-1-128.htm)