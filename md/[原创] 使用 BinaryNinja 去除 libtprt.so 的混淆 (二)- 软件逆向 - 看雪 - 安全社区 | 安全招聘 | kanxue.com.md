> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282918.htm)

> [原创] 使用 BinaryNinja 去除 libtprt.so 的混淆 (二)

使用 BinaryNinja 去除 libtprt.so 的混淆 (二)
====================================

> 文章中的思路只是个人想法, 并不是最优解, 如有错误还望斧正.  
> 插件代码 github: [detx](https://github.com/EEEEhex/detx) [https://github.com/EEEEhex/detx](https://github.com/EEEEhex/detx)

版本: speedmobile_1.45.0.53757.apk 中的 libtprt.so

本文将分享去除 [魔改的控制流平坦化] 混淆的思路

1. 魔改的控制流平坦化
------------

### 1.1 原理

我们知道标准的控制流平坦化就是把各个 basicblock 放到了一个 switch 中, 然后通过改变 switch(var) 中判断的这个变量 var 来分发到其他基本块中,  
那么当一个 case 基本块结束, 一定会往 var 中写入一个新值, 来让下一轮分发运行到另一个 case 基本块中:

```
uint32_t var = 0x1234;
while (1)
{
    switch(var)
    {
    case 0x1234:
        //逻辑1....
        var = 0x2345;
        break;
    case 0x2345:
        //逻辑2...
        if (v1 > 7)
            var = 0x3456;
        var = 0x4567;
        break;
    case 0x3456:
        //逻辑N...
        var = 0x4567;
        break;
    //...
    case 0x4567:
        exit(0);
        break;
    }
}
```

这个逻辑可以抽象为:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_CSUSCKZTMUB763Q.png)  
标准的一定包括: entry 块 (进行分发逻辑之前的第一个块), loopEntry 块 (分发循环开始的块), 分发块, 真实块 (源代码逻辑), Ret 块 (跳出函数的块), loopEnd 块 (分发循环结束再次进入 loopEntry 的块)  
.  
因为 ollvm 源代码 pass 里就是这么写的, 但 libtprt 修改了细节  
.  
libtprt 里面的平坦化去除了 loopEnd 块, 且存在很多编译优化的情况, 比如分发块和真实块共用, 真实块共用同一个 swtich 变量赋值指令, 判断提前等等, 而且在一个函数内有多个控制流平坦化 (平行或嵌套):

```
//平行的控制流平坦化 [1]
int32_t x9 = 0x4ba39ac1;
while (true)
{
    if (x9 == 0xde219aba)
        break;
     
    if (x9 == 0x4ba39ac1)
        x9 = 0x2e3a7efe;
     
    if (x9 == 0x2e3a7efe)
    {
        s_2 = s_1;
        x9 = -0x21de6546;
    }
}
 
int32_t x9_1;
//判断提前
if ((arg2 & 1) != 0)
    x9_1 = -0x25a2a29c;
else
    x9_1 = 0x64c05daa;
 
//第二个控制流平坦化 [2]
while (true)
{
    int32_t x9_2 = 0x412e838c;
    while (true)
    {
        if (x9_2 < 0x172ae48c)
        {
            //真实块逻辑....
        }
 
        if (x9_2 == ...)
        {
            x9_2 = x9_1; //判断提前
        }
 
        if (x9_2 >= ...)
        {
            int32_t x9_3 = ...;
            while (true)
            {
                switch (x9_3)
                ...//嵌套的控制流平坦化 [3]
            }
        }
    }
}
```

魔改的控制流平坦化不再是一个完全的 switch 结构体了, 其可能是以下 CFG 图:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_54WDJ5G8PB4VJES.png)

1.2 去混淆思路
---------

但无论怎么改, 怎么编译优化, 平坦化本质上就是通过设置一个值, 然后分发这个值, 然后看这个值会到达哪个真实块:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_D6MZDS92Y38VN4F.png)  
真实块里会重新设置一个值, 然后重新进入分发逻辑, 到达后继的真实块.  
.  
那么我的思路是:

*   一定会有一个分发变量, 所有往这个分发变量里写入值的, 都是真实块
*   所有 if 中判断了分发变量的, 都是分发块
*   没有后继的就是 Ret 块

然后就是正常的去平坦化的思路:

1.  获取所有真实块, 并拿到真实块的后继值
2.  通过设置分发变量为后继值, 然后模拟执行分发逻辑, 看会到达哪个真实块
3.  那么此块就是真正的后继块, 最后进行 Patch

关键点在于: ①怎么获取这分发变量 ②怎么拿到所有写入了分发变量的块 ③怎么拿到真实块的后继值 (要设置的分发变量的值)  
幸运的是, 这些问题通过 BinaryNinja 的 mlil ssa 层面都可以很轻松的解决

### 1.2.1 获取分发变量

这个需要自己获取, 我是通过获取鼠标当前行 (鼠标要点击 loopEntry 块) 的 if 语句的条件中的变量, 比如下面:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_NESBRXXTSKZMM84.png)  
用户需要找到分发开始块, 然后获取到 x8#2, 同时也能获取到 loopEntry

### 1.2.2 获取赋值块

我将写入了分发变量的块称为'赋值块' (注意: 赋值块不一定包含了全部的真实块):  
![](https://bbs.kanxue.com/upload/attach/202408/901761_GRGSP329YTM7EGR.png)  
怎么获取呢, 可以发现, 在 loopEntry 块中的 if 语句上有一个'x8#2 = ϕ(x8#1, x8#2, x8#5, x8#6, x8#9, x8#14)'.  
.  
这里面的 x8#1, x8#5... x8#14 都是被写入的分发变量, 可以通过 def_site 拿到赋值语句, 就是图上的 "x8#1 = 0x703c1e1e","x8#6 = 0x6110cf13" 等, 赋值语句所在的块就是'赋值块', 用 il_basicblock.source_block 获取.

### 1.2.3 获取真实块的后继值

怎么拿到一个真实块设置的分发变量的值呢?  
如果是没有条件的话, 就赋一个值的那种, 其实通过 1.2.2 就获取到了, 但如果是有条件的话:

```
if (arg3 == arg2)
    x8 = 0x2de2ab44;
else
    x8 = -0x26983ee;
```

在 mlil ssa 层面是:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_ECB79ZFVZDENV85.png)  
也就是 1.2.2 获取到的是'x8#9 = ϕ(x8#7, x8#8)'这条语句, 然后通过 x8#7/x8#8 的 def_site 一样可以获取到两个后继值, 然后通过'il_basic_block.incoming_edges[0].type == BranchType.TrueBranch'来拿到哪个后继值是满足条件时设置的, 哪个后继值是不满足条件时设置的.

### 1.2.4 模拟执行分发逻辑

首先通过 1.2.1 拿到了 loopEntry 块的地址, 但在从 loopEntry 开始执行分发逻辑之前, 需要进行分发比较值的初始化, 因为可以看到下图中, cmp 语句的寄存器其实在进入分发逻辑之前就赋值好了, 所以在模拟执行分发逻辑前需要进行分发比较值的初始化:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_JTUW7PK6YY36RG6.png)  
我的做法是拿到分发逻辑之前的所有块, 都当作 init 块, 然后模拟执行.  
当然也可以从 llil 层面, 拿到 if 条件中的寄存器被写入的语句, 然后该语句所在的块就是 init 块 (其实应该是这种写法比较合理)

1.3 嵌套平坦化
---------

从 1.1 节里的代码可以看到, 其实在一个函数中是存在多个平坦化的, 可能两个平坦化是平行的, 也可能在一个平坦化的 if 中又有一个平坦化.  
无论是平行还是嵌套, 一样可以通过 1.2 节的思路去除, 但是会少一个真实块的地址:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_XRAAYGTMJ3DD2N3.png)  
如上图中所示, 如果仅通过 1.2.2 把分发变量里写入值的当作真实块, 就是图上黄框的块.  
那么模拟执行的时候, 当执行到'if (x8_19 == 0x70d4e113) break;'时, 就暂停不了了, 因为没有遇到黄框块 (因为实际上要遇到蓝框块), 实际上当执行到这里逻辑时, 是要从内层平坦化跳出来, 所以需要把图上蓝框的块(这个块可能是任何块也可能是外层平坦化的分发块) 也当作真实块, 这样当模拟执行时遇到此块就会暂停返回.  
.  
我在代码中并没有自动去搜索出口块, 需要用户输入, 当然也是很好分辨的, 内层平坦化毕竟是一个循环, 所有的出口后继都是 loopEntry 块, 唯有一个后继不是 loopEntry 块的, 那就是出口块.

1.4 编译优化
--------

> 实际上编译优化的情况比较多, 需要特殊分析

### 1.4.1 判断提前

当分支判断并不是在真实块中进行的, 而且提前到了循环外怎么办?  
![](https://bbs.kanxue.com/upload/attach/202408/901761_39MJZQEX3SUMJYQ.png)  
可能有人会想, 在 mlil ssa 层面无非就是多了一层赋值呗, 一层一层往上找一样能找到后继值.  
确实是这样没错, 但问题是拿到该块对应的后继块后, 怎么 Patch?:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_YF72QFKXGJZF5MF.png)  
如果想把他下沉放到真实块里, 那 "cmp + b.cc + b" 放哪里?, **况且逻辑上也不能放到真实块里**, 比如这个条件判断的是 "cmp x1, #0", 如果在对应的真实块执行之前, x1 被改变了怎么办, 那逻辑就完成不正确了.  
我的思路是:

1.  判断改为 cset
2.  条件传递

```
mov w20, w1
...
tst w20, #0x1 //相当于cmp了
...
csel w9, w20, w12, ne  {0xda5d5d64}  {0x64c05daa}
...
str w9, [sp, #0xc {var_b4}]
...
cmp wX, w10 //分发逻辑
...
//------真实块开始----
ldr w9, [sp, #0xc {var_b4}] //就是x9 = x20, 值由上面的csel w9, w20确定
b 0xafcdc
//------真实块结束----
```

改为:

```
mov w20, w1
...
tst w20, #0x1
...
cset w9, ne  {1}  {0}  //csel改为cset
...
str w9, [sp, #0xc {var_b4}]
...
cmp wX, w10
...
ldr w9, [sp, #0xc {var_b4}] //此时x9 = 0或1
b 随机找一个分发块(因为分发快是无用块 可以随便Patch)
|
-> cmp w9, #0x1  //if (x9 == 1) 条件传递
b.ne 满足条件地址
b 不满足条件地址
(如果分发块放不下三条指令就接着找或拆分)
```

什么意思呢, 用伪代码表示就是这样:

```
//---原逻辑-------------
if (arg2 == 0)
    x28 = 0x3208a470;
else
    x28 = -0x3059f83a;
//...
x8 = x28;
//---------------------
改为==>
//---------------------
if (arg2 == 0)
    x28 = 1;
else
    x28 = 0;
 
if (x28 == 1)
{
    //0x3208a470对应的真实块
}
else
{
    //0x3059f83a对应的真实块
}
```

原先的判断逻辑不变, 只是把 x28 用 cset 设置成了 0 或 1, 不再是后继值了, 然后在真实块里判断 x28 是 0 还是 1.  
问题是这么改真实块还是需要**额外容纳**三条指令 (因为这样改真实块的原指令就不能动了), 那还是放不下, 所以就需要拿分发块(无用块) 去 patch:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_W6M82HJAHSYNY3H.png)

### 1.4.2 共用分发变量赋值语句

就是两个真实块的后继值是一样的, 比如:

```
//真实块1...
x8 = 0x124897684
 
//真实块2
x8 = 0x124897684
```

此时在汇编层面会把这个'x8 = 0x124897684'单独拆出来:  
![](https://bbs.kanxue.com/upload/attach/202408/901761_RZMKG3DHD6BU8XS.png)  
此时通过拿往分发变量里写入值的块当真实块就会少那俩块, 所以 1.2.2 节说赋值块不一定包含了全部的真实块, 此时就需要判断当一个赋值块有多个直接前继时, 它的前继也是真实块.  
.  
当然还有其他情况, 比如共用 cmp, 分发块和真实块合并为一个, 但这些都无伤大雅不影响整体逻辑.

2. 编写插件代码
=========

> 具体逻辑请查看 deflat2.py 与 emulate.py

2.1 模拟执行逻辑
----------

具体请查看 emulate.py 中的 "Emulator" "FuncEmulate" "DeflatEmulate" 三个类, 其实就是给 unicorn 封装了一层.

```
for info in assign_bb_infos:
    var_value = info.var_value
    real_sucs_ = []
    if isinstance(var_value, AssignBBInfo.VarValueInfoU): #只有一条赋值语句
        deflat_emu.set_switch_var_value(var_value.u_value)
        suc_addr = deflat_emu.start_until_stop()
        real_sucs_.append(suc_addr)
 
    elif isinstance(var_value, AssignBBInfo.VarValueInfoTF): #有两个赋值语句
        deflat_emu.set_switch_var_value(var_value.t_value)
        suc_addr = deflat_emu.start_until_stop()
        real_sucs_.append(suc_addr)
 
        deflat_emu.set_switch_var_value(var_value.f_value)
        suc_addr = deflat_emu.start_until_stop()
        real_sucs_.append(suc_addr)
    info.real_suc = real_sucs_
```

2.1 获取信息逻辑
----------

具体逻辑都在'def deflat2(bv: BinaryView, func: Function, switch_var_ssa: SSAVariable, extra_real_addr = None, manual_value = None, witch_check = False)'函数中, 比如:

```
#获取获取赋值块 同时获取真实块的信息
switch_vars = loop_phi_var_insn.src #所有用到的var的ssa变量
for svar in switch_vars:
    assign_insn = svar.def_site #给当前ssa变量赋值的指令
    if assign_insn == loop_phi_var_insn:
        continue
    cur_ssa_bb = assign_insn.il_basic_block
    disasm_bb = cur_ssa_bb.source_block
    real_sbb.append(disasm_bb)
    #如果一个赋值块有两个(两个以上的情况还没遇到过)直接前继, 且其前继块不是分发块, 则其前继块也是真实块, 该赋值块是共用的
    if (len(disasm_bb.incoming_edges) == 2): #>2块共用也有可能 遇到再改
        pre_edge1 = disasm_bb.incoming_edges[0]
        pre_edge2 = disasm_bb.incoming_edges[1]
        if (pre_edge1.type == BranchType.UnconditionalBranch) and (pre_edge2.type == BranchType.UnconditionalBranch):
            pre_bb1 = pre_edge1.source
            pre_bb2 = pre_edge2.source
            if (pre_bb1 not in dispatch_sbb) and (pre_bb2 not in dispatch_sbb):
                real_sbb.append(pre_bb1)
                real_sbb.append(pre_bb2)
     
    cur_bb_info = AssignBBInfo()
    cur_bb_info.bb_start = cur_ssa_bb.source_block.start    #赋值块起始地址
    cur_bb_info.set_var_addr = assign_insn.address          #改变var值的指令的地址
    if isinstance(assign_insn, MediumLevelILSetVarSsa): #说明不是分支
        #...
    elif isinstance(assign_insn, MediumLevelILVarPhi) and (len(assign_insn.src) == 2):#x9_2#20 = ϕ(x9_2#18, x9_2#19)
        #...
    assign_bb_infos.append(cur_bb_info)
```

3. 效果
=====

![](https://bbs.kanxue.com/upload/attach/202408/901761_5A5QD9W3W4N6APK.png)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 23 秒前 被 0xEEEE 编辑 ，原因： 修改细节

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm)