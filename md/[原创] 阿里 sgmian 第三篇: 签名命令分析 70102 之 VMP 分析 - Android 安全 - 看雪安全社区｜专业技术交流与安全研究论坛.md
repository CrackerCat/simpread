> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293456.htm#msg_header_h3_16)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> **时间:** 2026 年 11 月 08 日 星期日 01:37:35 CST  
> **作者:** 人生导师

哈哈哈哈哈哈哈哈哈哈哈哈，成了，没想到我的方法在阿里身上更加好用，哈哈哈哈哈  
其实也可以说是对阿里四神虚拟机的分析，希望能拿精华吧，简历上就缺一行然后格式就好看了

成了归成了，过程还是得记下来，不然下次换台 VM 又要从头摸。

先把结论摆这儿：淘宝这台 VM 不是我一开始以为的「另一个抖音」。

它是 **direct-threaded 解释器**，状态几乎全在寄存器里，handler 表只有一张；真正难的不是找 handler，是让 IR 在**变长字节码槽、嵌套帧和 native 调用**之间不散架。

按之前的叫法，淘宝这台我叫它**四神**。它和京东、快手、得物一样，属于「handler 内部直接算」的那一类，不是六神那种 OLLVM Pass 级 VMP。

```
Java doCommandNative(70102, Object[])
  └ jni_doCommandNative 0x5A2FC
     └ vm_invoke(7,1,2,flag=1) 0x56F70        ← 命令调度器，不是虚拟机
        └ cmd70102_handler 0xD002C            ← 参数封送 / 结果组装
           └ vmrun 0xD3410
              └ vm_invoke(7,1,2,flag=0)       ← script 表 miss，落到 hook
                 └ g_vm_hook_fn 0x1B4DC0
                    └ 0x1B0C50 → 0x1B05C4 → 0x160300 → 0x18F6CC
                       └ 0x170D1C            ← 真虚拟机入口


```

虚拟机本身只有一页纸的入口：

```
0x170d58  mov  x26, x2        ; module
0x170d5c  mov  x23, x5        ; handler 表
0x170d60  mov  x27, x1        ; VM ctx
0x170d64  mov  x25, x4        ; 整数 vreg = ctx+0x40
0x170d68  mov  x28, x0        ; 函数对象
0x170d6c  add  x24, x4, #0x100 ; 浮点文件 = ctx+0x140
0x170d70  ldr  x22, [x26, #8] ; 字节码基址
0x170d74  mov  x21, x22
0x170d78  add  x21, x21, x3, lsl #4 ; 当前槽 = base + entry*0x10
0x170d7c  ldrh w0, [x21]
0x170d80  ldr  x1, [x23, x0, lsl #3]
0x170d84  str  x21, [x27, #8]
0x170d88  br   x1


```

<table><thead><tr><th>项</th><th>值</th></tr></thead><tbody><tr><td>样本</td><td>淘宝 10.68.0 / <code>libsgmainso-6.8.260901.so</code>（2.9 MB）</td></tr><tr><td>trace</td><td>19,179,756 条指令；code.log 2.2 GB、rw.log 3.2 GB</td></tr><tr><td>VM 进入 / 退出</td><td>161 / 161，最大深度 <strong>3</strong></td></tr><tr><td>handler 表</td><td>827 项（<code>malloc(0x19D8)</code>）；本次命中 <strong>214</strong> 个 opcode</td></tr><tr><td>切段</td><td>690,815 段；去掉 399 条非 VM 误命中后 dispatch = <strong>690,813</strong></td></tr><tr><td>走过槽数</td><td>24,129</td></tr><tr><td>取指形态</td><td><code>#0x10</code> 472,143 / <code>#0x20</code> 105,340 / <code>[x21]</code> 89,681 / <code>#0x30</code> 23,649</td></tr><tr><td>handler 分类</td><td>纯直线 133 / 块内条件 81（合计 214）；带 bl 13（含 4 个复合）</td></tr><tr><td>帧协议</td><td>prologue 8,845 / return 8,845 / call 3,781 / exit 161</td></tr><tr><td>vm_invoke</td><td>373 次 = 340 handler + 33 hook</td></tr><tr><td>native 检查点</td><td>5,857；<code>sym_gaps=0</code>（符号闭环）</td></tr><tr><td>IR 产出</td><td>669,183 条 VM 指令（vline）；1,198,903 行 IR + native</td></tr></tbody></table>

命令层的参数封送、返回值组装在 2.70102 那篇已经铺过，这篇只讲 VM 本身：**它怎么跑，怎么从 trace 还原到 IR**。

我一开始把阿里想得太复杂了：一看到 `vm_invoke` 这种名字，下意识以为它是虚拟机入口，结果它只是个**命令调度器**——一张表、查表、调用。真正执行字节码的是它下游的 `0x170D1C`，一个十几条指令就能讲完的 direct-threaded 解释器。

这台 VM 和抖音七神最大的区别有两条：

1.  **vPC 是寄存器 `x21`，不是 ctx 里的内存槽。** 七神要反向切片找写 `ctx+0xC` 的 STORE，这边 `x21` 的 trace 写列直接就是下一槽地址，省掉一整层推导。
2.  **分发尾巴是固定的四条。** `ldrh` 取指 → `ldr` 查表 → `str x21,[x27,#8]` 簿记 → `br`。切段不需要理解 handler 内部，认这四条就够了。

代价是**字节码槽是变长的**：`0x10 / 0x20 / 0x30` 都有，还有调用 / 跳转后直接 `[x21]` 取指的形态。你要是假设「一条 VM 指令占一个 0x10 槽」，IR 的 vIP 会全错。

Java 侧 `doCommandNative(70102, Object[])` 进来之后，先到 `jni_doCommandNative`（0x5A2FC），然后：

```
jni_doCommandNative 0x5A2FC
 └ vm_invoke(7,1,2, flag=1)          0x56F70   ← native 表
    └ vm_resolve_command             0x567A8
       └ cmd70102_handler            0xD002C   ← 只做参数封送 / 结果组装
          └ vmrun                    0xD3410
             └ vm_invoke(7,1,2, flag=0)        ← script 表 miss
                └ g_vm_hook_fn 0x1B4DC0
                   └ 0x1B0C50 → 0x1B05C4 → 0x160300
                      └ 0x18F6CC → 0x170D1C   ← VM 内核


```

这里最容易误判的是 `vm_invoke`。它是**重入**的：

*   native handler 用它（`(7,1,2, flag=1)`）；
*   VM 字节码也用它（`(1, …)`）去回调库函数、打点、注册。

命令号第一段区分世界：`a=7` 是业务命令，`a=1` 是库命令。所以看到 `vm_invoke` 不能直接把它当虚拟机，它只是 L1 调度层；真正的 VM 是 `g_vm_hook_fn` 下游那个拿 `x21/x23/x25/x27` 的解释器。

`0x170D1C` 的参数约定是这样的：

<table><thead><tr><th>寄存器</th><th>含义</th></tr></thead><tbody><tr><td><code>x0</code></td><td>函数对象</td></tr><tr><td><code>x1</code></td><td>VM ctx</td></tr><tr><td><code>x2</code></td><td>module</td></tr><tr><td><code>x3</code></td><td>入口槽号</td></tr><tr><td><code>x4</code></td><td>整数 vreg 文件基址（<code>ctx+0x40</code>）</td></tr><tr><td><code>x5</code></td><td>handler 表</td></tr></tbody></table>

入口先把它们钉死：

```
0x170d58  mov  x26, x2
0x170d5c  mov  x23, x5
0x170d60  mov  x27, x1
0x170d64  mov  x25, x4
0x170d68  mov  x28, x0
0x170d6c  add  x24, x4, #0x100   ; ctx+0x140
0x170d70  ldr  x22, [x26, #8]    ; 字节码基址
0x170d74  mov  x21, x22
0x170d78  add  x21, x21, x3, lsl #4
0x170d7c  ldrh w0, [x21]
0x170d80  ldr  x1, [x23, x0, lsl #3]
0x170d84  str  x21, [x27, #8]
0x170d88  br   x1


```

`x3` 是**入口槽号**，`x21 = x22 + x3*0x10`。这条式子很重要：它说明字节码槽的步长是 `0x10`，而且调用 VM 函数时传的是「函数入口相对字节码基址的槽号」，不是绝对地址。

ctx 的布局（按 `x27` 量）：

<table><thead><tr><th>偏移</th><th>内容</th></tr></thead><tbody><tr><td><code>+0x00</code></td><td>引擎类型（2 / 4）</td></tr><tr><td><code>+0x04</code></td><td>host 调用标志</td></tr><tr><td><code>+0x08</code></td><td>当前 pc 的簿记槽（<code>str x21,[x27,#8]</code> 写这里）</td></tr><tr><td><code>+0x10/18/20</code></td><td>VM 栈顶 / malloc 基址 / 大小（≥ <code>0x40000</code>）</td></tr><tr><td><code>+0x38</code></td><td>vcpu 自指针</td></tr><tr><td><code>+0x40</code></td><td>整数 vreg[0..31]，32 × 8 B</td></tr><tr><td><code>+0x80</code></td><td>host 参数窗口 = vreg[8..13]</td></tr><tr><td><code>+0xB0</code></td><td>vreg[14] = SP</td></tr><tr><td><code>+0xB8</code></td><td>vreg[15] = &amp;unk_2D1CB0（顶层兼返回哨兵）</td></tr><tr><td><code>+0xC0</code></td><td>vreg[16..31] 保存区</td></tr><tr><td><code>+0x140</code></td><td>浮点文件（<strong>4 B 索引</strong>，<code>lsl #2</code>）</td></tr><tr><td><code>+0x240</code></td><td>module</td></tr><tr><td><code>+0x24C</code></td><td>魔数 <code>0xB8A3</code></td></tr><tr><td><code>+0x250</code></td><td>函数对象</td></tr></tbody></table>

这里有个细节要单独说：`ctx+0x8` 不是 vPC 本体，只是解释器每轮把 `x21` 存一份的**簿记槽**。真正决定下一条指令的是 `x21`。七神那台反过来，vPC 是 `ctx+0xC` 的内存槽（`x23` 指向它），字节码源在 `x20`，当前指令在 `x8`。两台的锚点不能混。

这台 VM 的锚点全是寄存器，入口一次定死，后面不再变：

<table><thead><tr><th>寄存器</th><th>pypcode offset</th><th>作用</th><th>本次运行值</th></tr></thead><tbody><tr><td><code>x28</code></td><td><code>0x40E0</code></td><td>函数对象</td><td><code>0x700190053140</code></td></tr><tr><td><code>x27</code></td><td><code>0x40D8</code></td><td>VM ctx</td><td><code>0x700200056710</code></td></tr><tr><td><code>x26</code></td><td><code>0x40D0</code></td><td>module</td><td><code>0x7000d004e380</code></td></tr><tr><td><code>x25</code></td><td><code>0x40C8</code></td><td>整数 vreg = <code>ctx+0x40</code></td><td><code>0x700200056750</code></td></tr><tr><td><code>x24</code></td><td><code>0x40C0</code></td><td>浮点文件 = <code>ctx+0x140</code></td><td><code>0x700200056850</code></td></tr><tr><td><code>x23</code></td><td><code>0x40B8</code></td><td>handler 表</td><td><code>0x7000d004c7b0</code></td></tr><tr><td><code>x22</code></td><td><code>0x40B0</code></td><td>字节码基址</td><td><code>0x7002111b7500</code></td></tr><tr><td><code>x21</code></td><td><code>0x40A8</code></td><td>当前槽 = <code>base + entry*0x10</code></td><td><code>0x7002111d7ee0</code></td></tr></tbody></table>

**锚点一律按 offset 认，不按名字认。** 同一个 offset 上 `w23`/`x23` 是两个视图（size 4 和 8），按名字认会漏。这个坑在七神那篇已经踩过一次，这边直接按 offset 写进 `lifter.py` 的 `ANCHOR`：

```
ANCHOR = {
    0x40A8: "ip",      
    0x40C8: "regf",    
    0x40B8: "table",   
    0x40C0: "float",   
    0x40D8: "ctx",     
    0x40D0: "module",  
    0x40B0: "bcbase",  
    0x40E0: "fnobj",   
}


```

和抖音那台的对照：

<table><thead><tr><th></th><th>抖音（七神）</th><th>淘宝 70102（四神）</th></tr></thead><tbody><tr><td>handler 表</td><td><code>x25</code></td><td><strong><code>x23</code></strong></td></tr><tr><td>字节码游标</td><td><code>x8</code></td><td><strong><code>x21</code></strong>（寄存器即 vPC）</td></tr><tr><td>整数 vreg</td><td><code>x24</code></td><td><strong><code>x25</code></strong> = <code>ctx+0x40</code></td></tr><tr><td>浮点</td><td><code>x26</code> / <code>x27</code></td><td><strong><code>x24</code></strong> = <code>ctx+0x140</code></td></tr><tr><td>ctx</td><td><code>x19</code></td><td><strong><code>x27</code></strong></td></tr><tr><td>module</td><td>—</td><td><code>x26</code>（<code>[x26+8]</code> = 字节码基址 <code>x22</code>）</td></tr><tr><td>函数对象</td><td>—</td><td><code>x28</code></td></tr><tr><td>dispatch 边界</td><td><code>ldr xN,[x25,…]; br</code></td><td><strong><code>str x21,[x27,#8]</code> + <code>br</code></strong></td></tr><tr><td>槽长</td><td>隐含</td><td><code>ldrh [x21,#0x10/0x20/0x30]!</code></td></tr></tbody></table>

这张表就是移植的全部工作量。`dy_7/VMP` 那套 `parse → matcher → lifter2` 的结构没动，换的是 `ANCHOR / SPACE / CTX_OFF / FILES / IP` 这五个常量。

每个 handler 的最后四条基本是固定的：

```
ldrh wN, [x21, #0x10]!     ; 取下一条 opcode，x21 前移 0x10
ldr  xN, [x23, xN, lsl #3] ; 查 handler 表
str  x21, [x27, #8]        ; 簿记当前 pc
br   xN                    ; 尾跳


```

**切段要认的是 `str x21,[x27,#8]` 紧跟 `br`，不是「br 的寄存器来自 x23 表」。** 原因是有条件跳转 handler 两个分支各自算自己的下一条：

```
0x182308  ldrb  w4, [x21, #4]
0x18230c  ldr   x3, [x25, x4, lsl #3]
0x182310  ldrsw x1, [x21, #8]
0x182314  cmp   w3, w1
0x182318  b.eq  0x182334
0x18231c  ldrsw x5, [x21, #0xc]
0x182320  add   x21, x21, x5, lsl #4   ; 落空支：相对跳
0x182324  ldrh  w8, [x21]
0x182328  ldr   x2, [x23, x8, lsl #3]
0x18232c  str   x21, [x27, #8]
0x182330  br    x2

0x182334  ldrh  w12, [x21, #0x20]!     ; 命中支：跳过 1 槽
0x182338  ldr   x11, [x23, x12, lsl #3]
0x18233c  str   x21, [x27, #8]
0x182340  br    x11


```

两个分支都能查到自己的表项，所以「br 的寄存器来自 `[x23, …]`」这条判据在命中的分支上会漏。按簿记指令认，两个分支都会落在同一个段边界上。

拿一个最干净的 OR handler 看：

```
0x179d00  ldrb  w2, [x21, #6]     ; src1
0x179d04  ldr   x0, [x25, x2, lsl #3]
0x179d08  ldrb  w15, [x21, #7]    ; src2
0x179d0c  ldr   x6, [x25, x15, lsl #3]
0x179d10  orr   x12, x0, x6
0x179d14  ldrb  w14, [x21, #4]    ; dst
0x179d18  str   x12, [x25, x14, lsl #3]
0x179d1c  ldrh  w19, [x21, #0x10]!
0x179d20  ldr   x17, [x23, x19, lsl #3]
0x179d24  str   x21, [x27, #8]
0x179d28  br    x17


```

对应的 IR 一行：

```
0x221  0x179d00  14834  orr  VREG[0x1a], VREG[0x0], VREG[0x18]   <0x2>


```

`0x221` 是 opcode，`0x179d00` 是 handler，`14834` 是本次执行次数，`<0x2>` 是回写后的 vIP。**opcode → handler → 语义**这张表就是这么长出来的。

一条 VM 指令的槽布局：

```
+0x00  u16 opcode
+0x02  .. +0x07   六个操作数（每字节一个寄存器号 / 小立即数）
+0x08  .. +0x0f   立即数 / 偏移
步长 = 0x10 × N


```

`N` 不是常数。分发尾巴里的预增量就是答案：

<table><thead><tr><th>取指形态</th><th>次数</th><th>含义</th></tr></thead><tbody><tr><td><code>ldrh wN,[x21,#0x10]!</code></td><td>472,143</td><td>普通 1 槽指令</td></tr><tr><td><code>ldrh wN,[x21,#0x20]!</code></td><td>105,340</td><td>2 槽指令（条件跳转落空支等）</td></tr><tr><td><code>ldrh wN,[x21]</code></td><td>89,681</td><td>调用 / 跳转后，<code>x21</code> 已指向目标首条，不再前移</td></tr><tr><td><code>ldrh wN,[x21,#0x30]!</code></td><td>23,649</td><td>3 槽指令</td></tr></tbody></table>

这里要纠正 2.5 那篇的一个统计口径：那里还列了 399 条 `ldrh w8,[x21,#0xa]`。我后来查了，这 399 条**全部**来自 `0x1afd20`，那是个普通 native 函数，`x21` 在里面是另一个结构体指针，不是 VM 的 vPC。把它们算进 dispatch 会让总数虚高 399。真正的 dispatch 是：

```
472143 + 105340 + 89681 + 23649 = 690813


```

和切段数 `690,815` 只差 2 条（收尾和跨深度的边界段），口径基本对得上。这个口径我当时没细看，后来对总数才发现，算是给「数字必须自洽」交了一次学费。

四条帧协议指令要单独记：

<table><thead><tr><th>opcode</th><th>handler</th><th>次数</th><th>语义</th></tr></thead><tbody><tr><td><code>0x24C</code></td><td><code>0x186768</code></td><td>8,845</td><td>函数 prologue：<code>SP += s64(pc+8)</code>（帧大小为负，本次 <code>-0x6e0</code>），保存 vreg[16..31]</td></tr><tr><td><code>0x249</code></td><td><code>0x187594</code></td><td>8,845</td><td>函数 return：恢复 vreg，<code>pc = vreg[15]+0x10</code></td></tr><tr><td><code>0x12</code></td><td><code>0x170CD4</code></td><td>3,781</td><td>VM 直接调用：<code>target = pc + s64(pc+8)*0x10</code></td></tr><tr><td><code>0x33A</code></td><td><code>0x170AC4</code></td><td>161</td><td>伪 opcode：退出解释器</td></tr></tbody></table>

prologue 的实际动作（`x25` 是 vreg 基址）：

```
0x186768  ldr  x11, [x25, #0x70]   ; vreg[14] = 旧 SP
0x18676c  ldr  x2,  [x21, #8]      ; 帧大小（负数）
0x186770  add  x16, x11, x2        ; 新 SP
0x186774  str  x16, [x25, #0x70]
0x186778  ldp  x9, x10, [x25, #0x80] ; vreg[16..]
0x18677c  stp  x9, x10, [x11]        ; 存到旧 SP
...
0x1867b8  ldp  x9, x10, [x25, #0x40] ; vreg[8..9]
0x1867bc  stp  x9, x10, [x25, #0xc0] ; -> vreg[24..25]
...
0x1867d0  str  x11, [x25, #0xf0]     ; vreg[30] = 旧 SP（帧基址）
0x1867d4  ldr  x19, [x25, #0x78]     ; vreg[15] = 调用点
0x1867d8  str  x19, [x25, #0xf8]     ; vreg[31] = 返回地址


```

return 反过来：

```
0x187594  ldr  x20, [x25, #0xf0]   ; 帧基址
0x187598  ldp  x9, x10, [x25, #0xc0] ; vreg[24..]
0x18759c  stp  x9, x10, [x25, #0x40] ; -> vreg[8..]
...
0x1875b0  ldr  x14, [x25, #0xf8]   ; vreg[31] = 返回地址
0x1875b4  str  x14, [x25, #0x78]   ; -> vreg[15]
0x1875b8  ldp  x9, x10, [x20]      ; 恢复 vreg[16..31]
...
0x1875f8  str  x20, [x25, #0x70]   ; SP = 帧基址
0x1875fc  ldr  x21, [x25, #0x78]   ; vPC = 返回地址


```

vreg 的角色划分也就清楚了：

```
0..7    临时
8..13   host 参数窗口
14      SP
15      调用点 / 返回哨兵
16..31  被调保存
23      module
24..29  VM 函数入参
30      帧基址
31      返回地址


```

8845 对 prologue/return、3781 次 call、161 次 exit、最大深度 3。**这台 VM 有自己的调用约定，而且调用约定本身就是字节码语义的一部分。** 所以 IR 里 `@PROLOGUE`、`@RET`、`@CALLVM` 不是装饰，是控制流骨架。

`matcher.py` 是流式的，规则只有一条：

> 一次 dispatch 跳转（`br`）收一段；段 = 上一次 dispatch 到这一次 dispatch 之间的完整执行切片，就是一个 handler。

ENTRY(`0x170D1C`) / EXIT(`0x170AC4`) 不是段边界，只是深度标记：执行到 ENTRY 深度 +1，执行到 EXIT 深度 -1。段可以跨 ENTRY/EXIT——复合 handler 里 `blr` 进嵌套 VM 时，外层段不会断，嵌套执行完继续回到外层段。`interpreter.py` 在段尾检查深度，如果 EXIT 把深度打回段首以下，就在 EXIT 处截断，避免把外层帧的原生代码混进来。

实测：

```
段数 = 690,815        唯一 opcode = 214
结束方式 = dispatch / exit
走过槽数 = 24,129


```

214 个 opcode 里，纯直线 133、块内条件分支 81，这两类正好凑满 214；另外有 13 个 handler 带 `bl/blr`，其中 4 个是复合指令：

```
纯直线（无块内分支）        133
有块内条件分支              81
  └─ 其中带 bl/blr          13（4 个复合，其余是 helper/帧协议调用）


```

这里的「分支」是块内条件，不是跨段的跳转。跨段跳转由 `ip` 线表达，下一节说。

整条流水线是：

```
code.log
  └ parse.py        -> 原子 dict（off / pc / asm / read / write / seq）
     └ matcher.py   -> handler 段（690,815 段）
        └ interpreter.py -> 分流：VM 进出 / 帧协议 / 普通 / 复合
           └ lifter.py   -> shadow IR + 值代入 + 折叠
              └ out/handlers.tsv / out/handlers / out/devirt.log


```

这份 trace 是 DuckDroid 用 `--stage 70102 --trace DIR` 产出的：init 在 trace 开始前跑完，arm 在 `doCommandNative` 入口（`libsgmainso+0x5A2FC`），到它的 `ret` 结束；后台线程和 init 命令都不记录。两个实现细节决定了这份 trace 能不能用：

*   `arm_address` 按入口地址 arm，避免后台 guest 线程抢到第一跳；
*   装好 code hook 后对入口区间 `invalidateCode()`，否则 init 里无 hook 缓存的 TB 会跳过入口那几条指令。

复跑对拍用 `TAOBAO_RANDOM_SEED` + `TAOBAO_WALL_TIME_NS` 固定熵流 / 时钟，否则每次熵流和时钟字段都会变，trace 没法逐条对。

#### 7.1 不吃「汇编回汇编」这条路

淘宝这台不用 keystone 回汇编，原因很实际：

*   trace 的 `off` 列就是模块内偏移，实测和 ELF 文件偏移 **1:1**（首个 LOAD 的 `p_offset == p_vaddr == 0`）；
*   直接按 `off` 读 4 字节喂 pypcode，`adrp/bl` 的 PC 相对立即数天然对；
*   keystone 停在 v8.0，淘宝的 handler 里有 v8.1 指令，回汇编会卡。

```
class SoBytes:
    def read(self, off, n=4):
        self.f.seek(off)
        return self.f.read(n)


```

`pypcode` 用的是 `AARCH64:LE:64:v8A`。每译一条指令先插一个 `IMARK`，A64 定长 4 字节，所以 **IMARK 和指令一一对应**，用它当边界切 shadow：

```
IMARK          -> 切到当前指令（读这一行的 read/write 列）
真实 p-code    -> 属于当前指令


```

值一律来自 trace 的 read/write 列，不来自求值：

*   寄存器输入取 read 列（执行前），输出取 write 列（执行后）；
*   中间 `unique` 不填——trace 不记，也没有谁需要它的值。

`x`/`w` 是同一个物理槽的两个视图，必须按 offset 合并，再按 size 分层保留：

```
self.AllRegisters.setdefault(v.offset, {})[v.size] = name


```

不这么干，`x10` 会被 `w10` 吃掉，后面所有值全错。

#### 7.2 前向染色

`taint()` 只做一件事：单趟扫 shadow，给每个 varnode 记「你的值派生自哪条锚点线」。

<table><thead><tr><th>规则</th><th>说明</th></tr></thead><tbody><tr><td>普通 op</td><td>输出继承所有输入的色（并集）</td></tr><tr><td>输入全无色</td><td>输出断链、清色（寄存器被重新赋值）</td></tr><tr><td><code>LOAD</code></td><td>地址有色 ≠ 读出来的值有色——地址的色过 <code>DEREF</code> 换成「值」色落到输出</td></tr><tr><td><code>STORE</code></td><td>不产值，但地址 / 值两路的色扫到时就记下来</td></tr></tbody></table>

`DEREF` 表就是「指针 → 指针指向的值」：

```
DEREF = {"regf": "regval", "table": "hptr", "float": "fregval"}


```

`x21` 线、`x23` 线、`x25` 线、`x24` 线都是同一个函数换个种子。**先用种子把锚点线拉出来，再谈语义**，这是整套方法的核心顺序。

#### 7.3 反向切片

操作数线不是从 handler 头往下推，而是**从写虚拟寄存器文件的 STORE 起，往上游跟 def-use**。

理由和七神那篇一样：一条 VM 指令的结果就落在那几个 STORE 上。往前推是发散的，往回追是收敛的——每个 varnode 只有一处定义。p-code 不是严格 SSA，但块内是顺序语义：**「use 之前最后一次写」就是它的 def**，反向走是确定的，不用解数据流方程组。

```
def _defs(self, shir):
    ...
def _def_before(defs, key, pos):
    


```

根只有四类：

```
bc[+off]      字节码字段
VREG[idx]     整数虚拟寄存器
FREG[idx]     浮点槽
ctx[+delta]   VM 上下文


```

helper 里的槽位是 spill/reload + 别名算出来的，静态锚点认不准。所以 `_ctx_delta()` 沿 def 链往回走，根必须落在：

*   `x27` = normal handler 里一直放着 ctx 的寄存器；
*   `x1` = helper 形态下 ctx 的传入寄存器（复合 handler 先 `mov x1,x27` 再 `bl`）。

路上只允许 COPY、零符号扩展、常量加减、栈上 spill 再 reload。走到别的寄存器 / 算式就认不出，认不出就按裸指针 `*` 输出，**不猜**。这条「宁缺勿错」在 helper 里特别重要，因为 helper 有大量自己的栈帧和自校验状态机，认错了整段 VREG 都会废。

`_img_addr()` 负责另一件事：这个访存地址到底在 VM 镜像里，还是在运行期对象上。

*   根落在 `x21`（字节码游标）→ 镜像访问，折成 `bc[+k]`；
*   根落在 helper 的「记录区指针」实参（靠 `[值+8]` 这个用法认）→ 也当镜像；
*   根落在 `x25`（VREG 文件）→ 运行期对象，**不能折**，折了依赖就丢了。

#### 7.4 折叠

SLEIGH 把 `BFI/UBFX/LSR` 展成 shift/or/and 链，一条指令能膨胀成几十个算符。这些是**解码头**，不是 VM 语义。不折的话，IR 就是一堵墙。

这台用的办法是**追字节道**：算式的第 k 个字节是从哪个内存字节来的。位对齐（整字节的移位 / 掩码）才追得动，追不动就不折。

够不够格折成「读」，三条判据缺一不可：

1.  恰好一条内存来处；
2.  落在第 0 道上；
3.  值也等于它。

两个坑：

*   只比值会认错字节：`W8 & 0xff` 和 `W8 >> 16` 可能恰好同值，所以**先由来历定是哪个字节，值只做确认**；
*   来历相同 ≠ 位置相同：`u16[+8] << 8` 的字节 0 也来自 `+8`，但那是移位不是提取，只能靠「落在哪一道」判。

`csel` 也要特殊处理。A64 的 `csel` 没有 CSELECT op，SLEIGH 展成「CBRANCH + 同一格 unique 在分支前后各定义一次」。反向线性取「use 前最后一次写」只拿到落空那一支，条件整支丢。所以 `_branches()` 把 CBRANCH 夹出来的条件抓回来，`_fold()` 再把条件折成常量的 csel 收成一支。**条件跳转的语义全在条件上，丢了条件等于丢了这条 handler。**

#### 7.5 IR 长什么样

一条 VM 指令一个 vline（可以有多行 IR），列和 code.log 对齐：

```
vline   vIP    off     seq    seq_end  kind    ir


```

举例：

```
1   0x2   0x179d00  45be  45c8  normal  orr   VREG[0x1a], VREG[0x0], VREG[0x18]
1c  0x7   0x17a1f4  494f  4961  normal  VREG[0x3] = resolve_native(module, ctx, off=0x11828)   ; -> 0x7002111b6700
1b  0x5   0x18c870  473b  494e  normal  ip = call_vm(fnobj, ctx, idx=0x1d)   ; -> fn vmimg+0x90040
223 0x0   0x18c870  6b8a  6d48  normal  ip = call_vm(fnobj, ctx, idx=0x0)   ; -> fn vmimg+0xbab0


```

几条渲染规则：

*   虚拟寄存器两套名字：**VREG**（整数，`x25`）、**FREG**（浮点，`x24`）。
*   槽号写实例值（`VREG[0x14]`）——槽号本身就是这条指令的操作数。
*   **解引用不折**：`*(VREG[0x5] + 0xc)` 折了操作就没了。
*   子树里没有读的式子整体算成一个数，负数写成 `-0x4a0`。
*   写 vPC 的那条（寄存器 `x21`，以及 `str x21,[x27,#8]` 这条簿记）不占行，它是 `ip` 线的输入。
*   顺序推进 `ip = ip + 0x10` 是隐含语义，不出行；条件推进、非 0x10 步长、跳转 / 返回才出行。

整套东西拆到最后，其实就是三条线：

```
边界线  x23   handler 表      -> 切段
控制线  x21   vPC / 字节码槽   -> 这条 VM 指令从哪来、下一条去哪
操作数线 x25/x24  vreg/freg   -> 这条 VM 指令算了什么


```

#### 8.1 边界线

`x23` 只做两件事：取值、被 `br`。`ldr xN,[x23,xN,lsl#3]` 的 `xN` 是 opcode，查出来的就是 handler。切段按上一节的尾巴签名，不按 handler 内部。

#### 8.2 控制线

`ip_line()` 取三个东西：

*   **pre**：段内第一次读 `x21` 的 trace 值（本条指令的槽地址）；
*   **post**：段内写 `x21` 的 trace 值（下一条指令的槽地址 / 跳转落点）；
*   **tree**：写 `x21` 的表达式的反向切片。

正常 dispatch 的 tree 是 `ip + 0x10`；条件 / 跳转 / 返回 handler 是 `csel` / `ip + N*0x10` / `VREG[0x1f]`。`str x21,[x27,#8]` 是簿记，不在这条线上，`_is_ip()` 里专门把它排除掉。

`_slot_index()` 把绝对地址归一成槽号：

*   优先按 VM 函数入口归一：prologue 执行时的 `x21` 就是函数入口，之后 `(x21 - func_base) >> 4`，每个函数从 0 起；
*   没有函数基址时退回 `x22` 镜像槽号；
*   跨函数调用入口记成被调函数的槽 0，与它的 prologue 对齐。

这样 vIP 跨 run、跨线程稳定，不会因为 ctx 地址变了就全乱。

#### 8.3 操作数线

`operation()` 扫所有 STORE：

*   写 VREG 文件 → `VREG[idx] = …`
*   写内存 → `*(…) = …`（store 指令，操作数在 VREG 指向的对象上）
*   写 vPC 的那条 → 跳过，走 ip 线
*   落不到锚点上的 → 跳过（helper 自己的栈帧、自校验状态机）

种子是**所有 STORE**，不是「写 VREG 的 STORE」。这一点在七神那篇被打过脸：VM 的 store 指令一个字节都不落 VREG 文件，种子太窄会整类被过滤掉，看起来像「这条 handler 没算东西」。这边从一开始就放宽了。

全量 19,179,756 条 trace 跑一遍：

```
段数             690,815
唯一 opcode      214
有 IR 的 vline   669,183
IR + native 行   1,198,903
空段             21,632   （纯顺序控制，无数据、无显式分支）
native 检查点    5,857
vm_invoke        373 = 340 handler + 33 hook


```

`669,183` 和 `690,815` 差的那 `21,632` 段，就是「只有 `ip+0x10` 隐含推进、没有数据写、没有显式分支」的段。它们不是丢了，是本来就没有可渲染的语义。

随便截一段真实 IR（`out/handlers`）：

```
0   0x209e  0x170d1c  4577  4592  normal  mov   ip, 0x7002111d7ee0
1   0x2     0x179d00  45be  45c8  normal  orr   VREG[0x1a], VREG[0x0], VREG[0x18]
2   0x3     0x18d6a0  45c9  45d4  normal  ldr   VREG[0x1b], [VREG[0x18], VREG[0x0]]
3   0x5     0x17e224  45d5  45dd  normal  add   ip, ip, 0x20
5   0x8     0x180ec0  45e7  45ed  normal  mov   VREG[0x18], 0x0
7   0xb     0x17f670  45f7  4600  normal  ldr   VREG[0x1a], [VREG[0x1a], 0x8]
9   0xe     0x17c760  460d  4622  normal  orr   t0, VREG[0x1a], 0x8
9   0xe     0x17c760  460d  4622  normal  and   VREG[0x1a], VREG[0x1a], t0
9   0xe     0x17c760  460d  4622  normal  ldr   VREG[0x1d], [VREG[0x1a], 0x8]
9   0xe     0x17c760  460d  4622  normal  ldr   VREG[0x1c], [VREG[0x1a], 0x10]
9   0xe     0x17c760  460d  4622  normal  add   ip, ip, 0x20
c   0x91    0x182308  4638  4642  normal  add   ip, ip, 0x810
1b  0x5     0x18c870  473b  494e  normal  ip = call_vm(fnobj, ctx, idx=0x1d)   ; -> fn vmimg+0x90040
1c  0x7     0x17a1f4  494f  4961  normal  VREG[0x3] = resolve_native(module, ctx, off=0x11828)   ; -> 0x7002111b6700


```

这屏 IR 里规律会自己浮出来：

*   `add ip, ip, 0x20` / `0x810` 是变长槽和跳转；
*   `ldr VREG[x], [VREG[y], off]` 是对象字段访问；
*   `VREG[x] = resolve_native(...)` 是拿方法描述符；
*   `ip = call_vm(...)` 是 VM→VM 调用。

把 `pre -> post` 串起来，就是一台 VM 的执行序列：`ip+0x10` 是顺序、跳变是分支、回跳是循环。CFG 不用单独造，从这列就能长出来。

13 个带 `bl/blr` 的 handler 里，有 4 个是真正的复合指令，内部会再进 VM：

<table><thead><tr><th>handler</th><th>次数</th><th>内部调用</th></tr></thead><tbody><tr><td><code>0x18C870</code></td><td>4,414</td><td><code>bl 0x161474</code>（<code>call_vm</code>）</td></tr><tr><td><code>0x18F2DC</code></td><td>4,836</td><td><code>bl 0x161474</code> + 先做一次 <code>orr/and</code></td></tr><tr><td><code>0x16FA70</code></td><td>1,099</td><td><code>bl 0x1618DC</code>（<code>call_vm2</code>）</td></tr><tr><td><code>0x173E3C</code></td><td>37</td><td><code>bl 0x160B48</code></td></tr></tbody></table>

这里的「次数」是 handler 执行次数，不是嵌套 VM 次数。真正进嵌套 VM 看 `@VM_ENTER`（161 次），别把 4,836 当成 4,836 次重入。

`0x18C870` 是最干净的样本：

```
0x18c870  str  x21, [x25, #0x78]   ; vreg[15] = 当前 vPC（长返回用）
0x18c874  mov  x0, x28             ; fnobj
0x18c878  mov  x1, x27             ; ctx
0x18c87c  ldr  x2, [x21, #8]       ; idx
0x18c880  bl   0x161474            ; call_vm
0x18c884  cmp  x0, #0
0x18c888  b.eq 0x18c8a0            ; 0 = 长返回，恢复 vreg[15]
0x18c88c  mov  x21, x0             ; 否则进入被调 VM 函数
0x18c890  ldrh w3, [x21]
0x18c894  ldr  x5, [x23, x3, lsl #3]
0x18c898  str  x21, [x27, #8]
0x18c89c  br   x5

0x18c8a0  ldr  x6, [x25, #0x78]
0x18c8a4  mov  x21, x6
0x18c8a8  ldrh w8, [x21, #0x10]!
0x18c8ac  ldr  x11, [x23, x8, lsl #3]


```

`call_vm` 的返回值是**被调 VM 函数的字节码入口**；返回 0 表示长返回（被调函数已经把返回地址塞回 `vreg[15]`）。所以 `x21 = x0` 是「跳进 VM 函数」，不是「函数返回」。`0x18F2DC` 多做了 `orr/and`，`0x16FA70` 用 `call_vm2` 走运行期方法对象，`0x173E3C` 走另一条 helper。

这四个之外，还有 `resolve_native` / `lookup` 两条不调用函数的「机制调用」：

```
_CALL_NAMES = {
    "0x16e2dc": "resolve_native",
    "0x161474": "call_vm",
    "0x1618dc": "call_vm2",
    "0x16e27c": "lookup",
}


```

*   `resolve_native(module, ctx, off)`：返回 `(ctx->[0x240])->[0x1A0] + off`，是方法表描述符指针，不调用函数；
*   `lookup(module, ctx, idx)`：返回 `((ctx->[0x240])->[0xB8] + idx*0x18)->[8]`，是真实 native fnptr，也不调用函数；
*   `call_vm(fnobj, ctx, idx)`：`idx` 是描述符表索引（步长 `0x18`），表项 `entry_type=0` 走 VM 函数，`1` 走原生 `call_desc`；
*   `call_vm2(fnobj, ctx, obj)`：`obj` 是运行期算出的方法对象指针，动态派发。

`vm_invoke` 是另一条固定流程，不用看字节码：

```
vm_invoke(a, b, c, flag, ctx, status)
  table   = flag ? g_vm_native_table : g_vm_script_table
  handler = vm_resolve_command(table, a, b, c, 0, &handler)
  rc ? g_vm_hook_fn(&desc, &st)   0x570D4  blr x8
     : handler(ctx, status)       0x57164  blr x8
  return                          0x57174  ldr x0,[sp,#var_88]


```

本 trace 373 次 = 340 handler + 33 hook，两个出口严格一一对应。`hook` 落点 `0x1B4DC0`，就是 VM 的重入入口。

native 调用的 ABI 也固定：

```
call_type = 2
实参 VREG[8..13,16] -> x0..x6
返回写 VREG[8]


```

`native_checkpoints.tsv` 有 5,857 个检查点，每个都记录 `x0..x6` 的符号表达式、callee 真正消费的实参上的 Java 依赖、额外实参。当前 `sym_gaps=0`。**native 屏障的意思是：只锁 ABI，不进去；每次 native 调用后 mem 缓存作废，避免把 callee 内部的写当成 VM 的写。**

前面一直在讲 VM 怎么跑，但 VM 并不认识 Java 对象。它看到的只是一块 216 字节的 `sgm_cmd_args`。从 `Object[] args` 到 `VREG[0x10]`，中间有三层手：

1.  `jni_doCommandNative`：把 `id` 拆成 `(a,b,c)`，把 `JNIEnv*` 和 `Object[]` 塞进 `ctx_jni`；
2.  `cmd70102_handler`：用 JNI getter 把 `Object[]` 摊平成 `sgm_cmd_args`；
3.  `vmrun`：再包一层 `{&args, &s}`，交给 VM。

这一节按这三层拆。

#### 11.1 Java 侧：12 个 Object

本 trace 的 Java 入参（`hook_a.log`）：

<table><thead><tr><th>下标</th><th>Java 类型</th><th>值</th><th>落到</th></tr></thead><tbody><tr><td>0</td><td>String</td><td><code>"&lt;appKey&gt;"</code></td><td><code>str0</code></td></tr><tr><td>1</td><td>String</td><td>185 字符，<code>at88…</code></td><td><code>str1</code></td></tr><tr><td>2</td><td>Boolean</td><td><code>false</code></td><td><code>bool2</code></td></tr><tr><td>3</td><td>Integer</td><td><code>0</code></td><td><code>int3</code></td></tr><tr><td>4</td><td>String</td><td><code>"mtop.taobao.afc.reductionandlogin"</code></td><td><code>str4</code></td></tr><tr><td>5</td><td>String</td><td><code>"pageId=&amp;page</code></td><td><code>str5</code></td></tr><tr><td>6</td><td>null</td><td>—</td><td><code>str6</code>（len=0）</td></tr><tr><td>7</td><td>null</td><td>—</td><td><code>str7</code>（NULL）</td></tr><tr><td>8</td><td>null</td><td>—</td><td><code>str8</code>（NULL）</td></tr><tr><td>9</td><td>String</td><td><code>"r_2"</code></td><td><code>str9</code></td></tr><tr><td>10</td><td>Integer</td><td><code>0</code></td><td><code>bizId</code></td></tr><tr><td>11</td><td>Integer</td><td><code>0</code></td><td><code>flag</code>（VM 里会变成 8）</td></tr></tbody></table>

Java 侧签名是 `doCommandNative(int id, Object[] args)`，所有元素都是 boxed object。native 侧不能直接当 `int`/`bool` 用，必须走 JNI 取。

#### 11.2 JNI 入口：先把 id 拆成 (a,b,c)

`jni_doCommandNative`（0x5A2FC）的头几行：

```
v12 = args;                 
v7  = code / 10000;         
v8  = ((5243 * (code % 10000)) >> 19)
    + ((unsigned)(5243 * (code % 10000)) >> 31); 
v9  = code % 100;           
v5  = vm_invoke(code/10000, v8, code%100, 1, &v7, &v6);


```

这里要纠正一个说法：`b` 不是 hash，是 `(id % 10000) / 100`。反编译里的 `(5243 * x) >> 19` 是「除以 100」的倒数乘法，`5243 / 2^19 ≈ 1/100`。所以 70102 拆成 `(7,1,2)`，70703 拆成 `(7,7,3)`，不是随机 hash。

`ctx_jni` 是栈上的 0x20 字节：

```
+0x00  a            (int)
+0x04  b            (int)
+0x08  c            (int)
+0x0C  0
+0x10  JNIEnv*
+0x18  jobjectArray args


```

`flag=1` 表示走 native 表；`status` 是另一个 `int`，失败时 `throw_sec_exception()`。

#### 11.3 第一跳：vm_invoke 查 native 表

`vm_invoke(a,b,c,flag,ctx,status)`（0x56F70）：

```
flag ? g_vm_native_table : g_vm_script_table
  -> vm_resolve_command(table, a,b,c,0,&handler)
  -> found ? handler(ctx,status) : g_vm_hook_fn(...)


```

`(7,1,2,1)` 在 native 表里查到 `mod+0xD002C`，也就是 `cmd70102_handler`。注意这一步还不是 VM：它只是命令 handler，和 `(1,32,12)` 那些库命令共用同一张调度器。

#### 11.4 cmd70102_handler：把 Object[] 摊平成 sgm_cmd_args

`cmd70102_handler` 在栈上开了一个 `sgm_cmd_args args`（216 字节），先把 `args.str0.len` 起的 0xD0 字节清零，再给几个短字符串预置栈缓冲：

```
args.str0.s = s;      args.str0.cap = 50;    
args.str1.s = v15;    args.str1.cap = 500;   
args.str4.s = v14;    args.str4.cap = 150;   
args.str5.s = v13;    args.str5.cap = 500;   
args.str6.s = v12;    args.str6.cap = 30;    
args.str9.s = v11;    args.str9.cap = 30;    


```

`str7` / `str8` 不预置（`s=NULL, cap=0`），只有非 null 且长度达到或超过 cap 时 getter 才 `malloc`。

然后按 Java 下标取：

```
jni_get_string_arg(ctx->env, ctx->args, 0, &args.str0);
args.bool2 = jni_get_bool_arg(ctx->env, ctx->args, 2) & 1;
args.int3  = jni_get_int_arg(ctx->env, ctx->args, 3);
jni_get_string_arg(ctx->env, ctx->args, 1, &args.str1);
jni_get_string_arg(ctx->env, ctx->args, 4, &args.str4);
jni_get_string_arg(ctx->env, ctx->args, 5, &args.str5);
jni_get_string_arg(ctx->env, ctx->args, 6, &args.str6);
jni_get_string_arg(ctx->env, ctx->args, 7, &args.str7);
jni_get_string_arg(ctx->env, ctx->args, 8, &args.str8);
jni_get_string_arg(ctx->env, ctx->args, 9, &args.str9);
args.bizId = jni_get_int_arg(ctx->env, ctx->args, 10);
args.flag  = jni_get_int_arg(ctx->env, ctx->args, 11);


```

字符串 getter 的动作（`jni_get_string_arg` 0x154744）：

```
obj = GetObjectArrayElement(env, args, idx);
if (!obj) return 0;                  
utf = GetStringUTFChars(env, obj, 0);
len = strlen(utf);
if (out->cap <= len) {
    out->s = malloc(len + 1);        
    out->heap = 1;
}
memcpy(out->s, utf, len);
out->s[len] = 0;
out->len = len;
ReleaseStringUTFChars(env, obj, utf);
DeleteLocalRef(env, obj);


```

`jni_get_bool_arg` 是 `GetObjectArrayElement` → `Boolean.booleanValue` → `&1`；`jni_get_int_arg` 是 `GetObjectArrayElement` → `Integer.intValue`。都是「先取元素，再 unbox」，不是直接读 `jintArray`。

`sgm_cstr` 和 `sgm_cmd_args` 的布局：

```
sgm_cstr (0x18):
  +0x00  char* s
  +0x08  int len
  +0x0C  int cap
  +0x10  int heap       // 1 = malloc 过，handler 退出时 free

sgm_cmd_args (0xD8):
  +0x00 str0  +0x18 str1  +0x30 bool2  +0x38 int3
  +0x40 str4  +0x58 str5  +0x70 str6  +0x88 str7
  +0xA0 str8  +0xB8 str9  +0xD0 bizId  +0xD4 flag


```

本 trace 的实际值：

<table><thead><tr><th>字段</th><th>偏移</th><th>len</th><th>cap</th><th>值</th></tr></thead><tbody><tr><td><code>str0</code></td><td>+0x00</td><td>8</td><td>50</td><td><code>"&lt;appKey&gt;"</code></td></tr><tr><td><code>str1</code></td><td>+0x18</td><td>185</td><td>500</td><td><code>"at88…"</code></td></tr><tr><td><code>bool2</code></td><td>+0x30</td><td>—</td><td>—</td><td>0</td></tr><tr><td><code>int3</code></td><td>+0x38</td><td>—</td><td>—</td><td>0</td></tr><tr><td><code>str4</code></td><td>+0x40</td><td>33</td><td>150</td><td><code>"mtop.taobao.afc.reductionandlogi"</code></td></tr><tr><td><code>str5</code></td><td>+0x58</td><td>47</td><td>500</td><td><code>"pageId=&amp;pageName=…"</code></td></tr><tr><td><code>str6</code></td><td>+0x70</td><td>0</td><td>30</td><td>空</td></tr><tr><td><code>str7</code></td><td>+0x88</td><td>0</td><td>0</td><td>NULL</td></tr><tr><td><code>str8</code></td><td>+0xA0</td><td>0</td><td>0</td><td>NULL</td></tr><tr><td><code>str9</code></td><td>+0xB8</td><td>3</td><td>30</td><td><code>"r_2"</code></td></tr><tr><td><code>bizId</code></td><td>+0xD0</td><td>—</td><td>—</td><td>0</td></tr><tr><td><code>flag</code></td><td>+0xD4</td><td>—</td><td>—</td><td>0 → 8（<code>vmrun</code> 置 bit3）</td></tr></tbody></table>

这里有两个细节：

*   **flag 不是 Java 直接给的 8。** Java `args[11]` 是 0，`cmd70102_handler` 也存 0；`vmrun` 在配置开关返回 1 时做 `args->flag |= 8`，VM 才看到 8。所以文件头里写 `flag=8`，根在 `vmrun`，不在 Java。
*   **所有字符串都在栈上。** str0 8<50、str1 185<500、str4 33<150、str9 3<30，都没有触发 `malloc`，`heap=0`。`args` 和这些缓冲都在 `cmd70102_handler` 的栈帧里；VM 是在 `vmrun` 里同步跑的，等 `vmrun` 返回，指针仍然有效，但 handler 只会 free 那些 `heap=1` 的。

#### 11.5 vmrun：再包一层 {&args, &s}

`vmrun(args, out, status)`（0xD3410）先做两个检查：

```
if (!args->str0.s || !args->str0.len || !*args->str0.s) {
    *status = 0x961; return 0;
}

ret = runtime->invoke(1, 32, 12, 0, {name:"<config>"}, &st);
if (ret == 1) args->flag |= 8;


```

然后把 `args` 和 8 个输出槽打包：

```
v64[0] = &args;   
v64[1] = &s;      
runtime->invoke(7, 1, 2, 0, v64, status);


```

`flag=0` 走 script 表；表里没有 `(7,1,2)`，于是落到 `g_vm_hook_fn`（0x1B4DC0）→ `0x1B0C50` → VM 解释器。

**注意这里换了一个 ctx。** JNI 入口的 `ctx_jni` 是 `{a,b,c,0,env,Object[]}`；`vmrun` 传给 VM 的 `ctx` 是 `{&sgm_cmd_args, &out_slots}`。VM 的 `ctx[0]` 不是 JNIEnv，是参数结构；`ctx[1]` 是输出槽。两套 ctx 不要混。

#### 11.6 VM 侧：VREG[0x10] 就是 args 指针

VM 入口 `0x170D1C` 拿到的 `x1` 是 `v64`。它把 `x1` 放进 `x27`（ctx），之后某条 VM 指令执行：

```
vline 3ca:  VREG[0x10] = [ctx]      // ctx[0] = &sgm_cmd_args


```

从这以后，VM 里的 Java 参数就是 `VREG[0x10]` 指向的那块 216 字节。字段访问全是固定偏移：

<table><thead><tr><th>字段</th><th>偏移</th><th>首次使用 handler</th><th>vline</th><th>IR</th></tr></thead><tbody><tr><td>str0.s</td><td>+0x00</td><td>0x17b6c0</td><td>3086b</td><td><code>ldr VREG[0x1a], [VREG[0x10], VREG[0x0]]</code></td></tr><tr><td>str0.len</td><td>+0x08</td><td>0x17eac8</td><td>3086d</td><td><code>ldr VREG[0x1a], [VREG[0x10], 0x8]</code></td></tr><tr><td>str1.s</td><td>+0x18</td><td>0x17c3a0</td><td>30868</td><td><code>orr/and</code> 后 <code>ldr VREG[0x8], [VREG[0x10], 0x18]</code></td></tr><tr><td>str1.len</td><td>+0x20</td><td>0x17eac8</td><td>30869</td><td><code>ldr VREG[0x1a], [VREG[0x10], 0x20]</code></td></tr><tr><td>bool2</td><td>+0x30</td><td>0x17f670</td><td>68f22</td><td><code>ldr VREG[0x2], [VREG[0x10], 0x30]</code></td></tr><tr><td>int3</td><td>+0x38</td><td>0x17eac8</td><td>30866</td><td><code>ldr VREG[0x1a], [VREG[0x10], 0x38]</code></td></tr><tr><td>str4.s</td><td>+0x40</td><td>0x17f670</td><td>3cc</td><td><code>ldr VREG[0x18], [VREG[0x10], 0x40]</code></td></tr><tr><td>str4.len</td><td>+0x48</td><td>0x17eac8</td><td>3d0</td><td><code>ldr VREG[0x1a], [VREG[0x10], 0x48]</code></td></tr><tr><td>str7.s</td><td>+0x88</td><td>0x17c3a0</td><td>86ceb</td><td><code>ldr VREG[0x18], [VREG[0x19], 0x88]</code></td></tr><tr><td>str8.s</td><td>+0xA0</td><td>0x17f670</td><td>3c792</td><td><code>ldr VREG[0x8], [VREG[0x10], 0xA0]</code></td></tr><tr><td>str9.s</td><td>+0xB8</td><td>0x17f670</td><td>74777</td><td><code>ldr VREG[0xa], [VREG[0x18], 0xB8]</code></td></tr><tr><td>str9.len</td><td>+0xC0</td><td>0x17eac8</td><td>780e3</td><td><code>ldr VREG[0x9], [VREG[0x18], 0xC0]</code></td></tr><tr><td>bizId</td><td>+0xD0</td><td>0x17eac8</td><td>3086f</td><td><code>ldr VREG[0x1c], [VREG[0x10], 0xD0]</code></td></tr><tr><td>flag</td><td>+0xD4</td><td>0x17eac8</td><td>3cd</td><td><code>ldr VREG[0x1d], [VREG[0x10], 0xD4]</code></td></tr></tbody></table>

这张表就是「Java 参数在 VM 里到底从哪进」的完整清单。几个观察：

*   VM 把 `+0x18` 直接当 `str1.s` 指针读，不区分「结构体嵌套」；`sgm_cmd_args` 在 native 侧已经摊平，VM 只认偏移。
*   `str9.s` 走的是 `VREG[0x18]`，不是 `VREG[0x10]`——VM 在中间把 args 指针抄到了另一个槽；这不妨碍溯源，只要符号执行知道 `VREG[0x18]` 也来自 `ctx[0]`。
*   `bool2` / `int3` / `bizId` / `flag` 是 32 位字段，VM 用 `ldr W` 直接读，不需要再解 Java 对象。
*   这 14 个点里没有一个是「回调 Java 取字段」；JNI 的活全在 `cmd70102_handler` 一次做完。

#### 11.7 返回：HashMap 是另一条路

`vmrun` 把输出槽 `strdup` 成 `out[0..7]`，`cmd70102_handler` 建一个 `java.util.HashMap`，按字段名 `put` 进去，`header_map_to_java` 用 `NewObject` + `NewStringUTF` + `CallObjectMethod` 返回 Java 对象。输入是 `Object[]` → `sgm_cmd_args`，输出是 `out[]` → `HashMap`；两条路在 `cmd70102_handler` 里交汇，但方向相反。

#### 11.8 符号执行：检查值流是否「完整」和「对称」

`tools/sym_check.py` 不是还原器，它是**值流闭包检查**：把 VM IR 当成单路径程序，符号化执行一遍，看每个值有没有来源、每个 native 实参能不能对上 VM 槽。

模型：

*   状态是 `vreg 槽 -> Sym`；临时量只在 handler 内有效。
*   `Sym` 分五类：常量 / Java 输入 `('i', name)` / 表达式 / 未知 `('u', origin)` / 基址 `('b', ctx|module|fnobj|bc|ip|args)`。
*   帧协议按实测：prologue 把 8..0xD 搬到 0x18..0x1D，return 反向搬回，0x10..0x1E 保存 / 恢复，0..7 算 callee-clobber；`0xE` 是帧指针（每帧一个 id），不是第 7 个实参。
*   根：在 `--ctx-seed 3b5:0x16` 指定的 vline 之后，把该槽种成 `ctx`；`ctx[0]` → `&args`；`args + off` 按 `LAYOUT` 认成 `str0.s` / `str0.len` / …。
*   指针来源：表达式节点同时带 `bases` 集合，`and/orr` 掩码之后仍然认得出 `args`/`ctx`，所以 VM 里的指针算术不会把根丢掉。
*   native 检查点：每个 `abi_calls` 调用点按 `call_type=2` 取槽 8..13,16 的符号，记成 `x0..x6`；不进入 native，只锁 ABI。
*   native 屏障：每次 native 调用后清空 `mem` 缓存（native 可能改写 spill 区）。

核心就两个函数，一读一写（简化版），正好对应「完整」和「对称」：

```
def read_slot(self, slot):
    s = self.state.get(slot)
    if s is None:
        self.gap(f"VREG[0x{slot:x}]")   
        return None
    self.defs[slot]["uses"] += 1
    return s

def write_slot(self, slot, sym):
    old = self.defs.get(slot)
    if old is not None and old["uses"] == 0:
        self.orphans_total += 1          
    self.defs[slot] = {"vline": self.vline, "seq": self.seq, "ir": self.cur_ir, "uses": 0}
    self.state[slot] = sym


```

`read_slot` 发现槽里没有值，就记一条 `gap`；`write_slot` 发现旧 def 一次都没被读过就被覆盖，就记一条 `orphan`。这两个计数器就是 `sym_gaps` / `sym_orphans` 的来源。

它输出四样：

<table><thead><tr><th>文件</th><th>含义</th><th>本次</th></tr></thead><tbody><tr><td><code>sym_gaps.tsv</code></td><td>用了但没定义（「凭空多值」）</td><td><strong>0 条</strong></td></tr><tr><td><code>sym_orphans.tsv</code></td><td>被覆盖且从未被读的写（dead-write）</td><td>92,676 条（只留前 20,000）</td></tr><tr><td><code>native_checkpoints.tsv</code></td><td>每个 native 调用点的 <code>x0..x6</code> 符号 + Java 依赖</td><td><strong>5,857</strong></td></tr><tr><td><code>sym_summary.txt</code></td><td>计数 + 模型假设</td><td>—</td></tr></tbody></table>

「完整」看 `sym_gaps=0`：没有哪个操作数是从空气里来的。每个读都能追到常量、Java 输入、ctx/module 基址、或某个 def；追不到就是 gap。

「对称」看两头：

1.  **def/use 对称**：每个写最好有读。`sym_orphans` 是反过来的一侧——写了没人读。92,676 条不是 bug：VM 里大量临时槽、spill 槽、被后续覆盖的中间值，本来就不会被读。所以 `sym_orphans` 只作参考，不是失败判据。
2.  **VM/native 对称**：VM 槽里的符号值，必须能对上 native 调用点的实际 `x0..x6`。`native_checkpoints.tsv` 把符号表达式和 `abi_calls.tsv` 的实测值放在一起；`java_inputs` 是 callee 真正消费的实参上带的 Java 字段（48 条），`extra_inputs` 是 callee 不用的额外实参（13 条）。这一步就是把「VM 里的符号」和「native 的真实 ABI」对齐。

`provenance` 精确字段核对 14 对 / 0 不对，是这套检查的抽样对账：挑了 14 个直接 load 的字段，符号来源和实测值逐一对，全部对上。

要说清楚它的边界：这是**单路径**检查，按 trace 顺序跑，不枚举分支；native 只锁 ABI，不进内部；`heap` 载荷和外部返回值记成 `?unknown(origin)`，不算缺值。它能回答「这次执行的值流有没有断、VM 槽和 native 实参对不对得上」，不能回答「任意输入下语义是否等价」。但对我们这种「先保证这次 trace 的骨架是完整的」的目标，够了。

#### 11.9 out/handlers 文件头

下面是 `out/handlers` 的头部原文（`;` 开头是注释）。参数映射、VM 内使用点、native ABI、符号检查都在这。三处小改：`b` 那一行按反编译改成 `(id%10000)/100`（原文件写的是 hash）；第一行那句指向还原用途的话删了，免得又把这篇写成还原入口；appKey 和 str1 的 token 前缀打码成 `<appKey>` / `at88…`。其余照原文。

```
;====================================================================================================
; Java 参数映射（trace: 70102）
;====================================================================================================
; Java  doCommandNative(70102, Object[] args)
;  └ jni_doCommandNative 0x5A2FC
;      a=id/10000=7, b=(id%10000)/100=1, c=id%100=2, flag=1
;      ctx_jni = {a@+0, b@+4, c@+8, 0@+0xc, JNIEnv@+0x10, Object[]@+0x18}
;  └ vm_invoke(7,1,2,flag=1,&ctx_jni) 0x56F70 -> native 表 handler mod+0xd002c
;      cmd70102_handler 0xD002C：jni_get_{string,bool,int}_arg 把 Java args 搬进 sgm_cmd_args
;  └ vmrun 0xD3410
;  └ vm_invoke(7,1,2,flag=0,{&args,&s}) 0x56F70 -> script 表未命中 -> g_vm_hook_fn mod+0x1b4dc0
;      fn_call1 0x1B4DC0 -> 0x1B0C50 -> VM 解释器
;  └ VM: ctx[0]=&sgm_cmd_args=0x403ff850, ctx[1]=&out_slots=0x403ff590
;
; sgm_cmd_args @0x403ff850   (sgm_cstr: s@+0, len@+8, cap@+0xc, heap@+0x10)
;   +0x00 str0  len=8 cap=50 "<appKey>"   (Java args[0])
;   +0x18 str1  len=185 cap=500 "at88…"   (Java args[1])
;   +0x40 str4  len=33 cap=150 "mtop.taobao.afc.reductionandlogi"   (Java args[4])
;   +0x58 str5  len=47 cap=500 -   (Java args[5])
;   +0x70 str6  len=0 cap=30 -   (Java args[6])
;   +0x88 str7  len=0 cap=0 NULL   (Java args[7])
;   +0xa0 str8  len=0 cap=0 NULL   (Java args[8])
;   +0xb8 str9  len=3 cap=30 "r_2"   (Java args[9])
;   +0x30 bool2  = 0   (Java args[2])
;   +0x38 int3   = 0   (Java args[3])
;   +0xd0 bizId  = 0   (Java args[10])
;   +0xd4 flag   = 8   (Java args[11])
;
; VM 内参数使用点（VREG[0x10] = &args，见 vline 3ca）:
;   +0x00 str0.s    0x17b6c0  vline 3086b  ldr   VREG[0x1a], [VREG[0x10], VREG[0x0]]
;   +0x08 str0.len  0x17eac8  vline 3086d  ldr   VREG[0x1a], [VREG[0x10], 0x8]
;   +0x18 str1.s    0x17c3a0  vline 30868  orr   t0, VREG[0x10], 0x18 | and   VREG[0x10], VREG[0x10], t0 | ldr   VREG[0x8], [VREG[0x10], 0x18] | add   ip, ip, 0x30
;   +0x20 str1.len  0x17eac8  vline 30869  ldr   VREG[0x1a], [VREG[0x10], 0x20]
;   +0x30 bool2     0x17f670  vline 68f22  ldr   VREG[0x2], [VREG[0x10], 0x30]
;   +0x38 int3      0x17eac8  vline 30866  ldr   VREG[0x1a], [VREG[0x10], 0x38]
;   +0x40 str4.s    0x17f670  vline 3cc    ldr   VREG[0x18], [VREG[0x10], 0x40]
;   +0x48 str4.len  0x17eac8  vline 3d0    ldr   VREG[0x1a], [VREG[0x10], 0x48]
;   +0x88 str7.s    0x17c3a0  vline 86ceb  orr   t0, VREG[0x19], 0x88 | and   VREG[0x19], VREG[0x19], t0 | ldr   VREG[0x18], [VREG[0x19], 0x88] | add   ip, ip, 0x70
;   +0xa0 str8.s    0x17f670  vline 3c792  ldr   VREG[0x8], [VREG[0x10], 0xa0]
;   +0xb8 str9.s    0x17f670  vline 74777  ldr   VREG[0xa], [VREG[0x18], 0xb8]
;   +0xc0 str9.len  0x17eac8  vline 780e3  ldr   VREG[0x9], [VREG[0x18], 0xc0]
;   +0xd0 bizId     0x17eac8  vline 3086f  ldr   VREG[0x1c], [VREG[0x10], 0xd0]
;   +0xd4 flag      0x17eac8  vline 3cd    ldr   VREG[0x1d], [VREG[0x10], 0xd4]
;
; native 调用 ABI: call_type=2 -> 实参 VREG[8..13,16] -> x0..x6, 返回写 VREG[8]
;====================================================================================================
;====================================================================================================
; 符号闭环检查（tools/sym_check.py -> out/）
;   用而无定义（凭空多值）: gaps=0
;   native 检查点: 5857
;   其中 callee 实际消费的实参带 Java 字段依赖: 48
;   只有额外实参（callee 不用）带 Java 字段: 13
;   provenance 精确字段核对: 14 对 / 0 不对
;   => native 调用屏障已生效（每次 native 调用后 mem 缓存作废），
;      直接 load 的 provenance 全部核对通过
;   注意：used= 是数据依赖（如 malloc 的 size 由 str1 的某个堆字段算出），
;         不是指针恒等；派生值要做数值验证需要堆内容。
;   模型: 单路径；ctx 根；槽 0xe=帧指针(每帧唯一 id)；marker 按 [seq] 延迟；
;         native 只锁 ABI；帧 prologue 8..0xd -> 0x18..0x1d
;   文件: out/sym_gaps.tsv / native_checkpoints.tsv / sym_summary.txt
;====================================================================================================
; 0x170d1c @VM_ENTER   [4577]
0	0x209e	0x170d1c	4577	4592	normal	mov   ip, 0x7002111d7ee0
; 0x186768 @PROLOGUE -0x6e0 base=0x7002111d7ee0   [4593]
1	0x2	0x179d00	45be	45c8	normal	orr   VREG[0x1a], VREG[0x0], VREG[0x18]
2	0x3	0x18d6a0	45c9	45d4	normal	ldr   VREG[0x1b], [VREG[0x18], VREG[0x0]]


```

文件头里 `flag = 8 (Java args[11])` 是简写：Java 给的是 0，8 是 `vmrun` 的配置开关置的，见 11.4。再往后就是逐行 IR（`vline / vIP / off / seq / kind / ir`），第 9 节已经截过一段。

*   **锚点按 offset 认，不按名字认。** `w23`/`x23` 同槽，按名字认会漏。
*   **`ctx+0x8` 是簿记，不是 vPC。** 真正控制流是 `x21`；把 `str x21,[x27,#8]` 当 vPC 写点，切段会全错。
*   **槽长是变的。** `0x10/0x20/0x30` 都有，调用 / 跳转后还有 `[x21]` 取指。假设「一条指令一个槽」，vIP 必错。
*   **浮点文件是 4 B 索引。** `lsl #2`；`lifter.py` 里 `SCALE["FREG"]` 当时写成了 8，属于已知瑕疵。好在这条 trace 只碰 `FREG[0]/[1]`，而且 FREG 只做数据、不做基址，没影响主链；下次要修。
*   **399 条 `[x21,#0xa]` 不是 dispatch。** 全部来自 `0x1afd20` 这个普通 native 函数，`x21` 在那里是另一个结构体指针。
*   **种子要放宽到所有 STORE。** 只认「写 VREG」会漏掉 VM 的 store 指令（它写 `*VREG[ptr]+off`）。
*   **`vline` / `seq` 都是十六进制**，`seq == code.log` 行号。
*   **libc 的写不进 rw.log。** memcpy/strlen/JNI 写的缓冲，要么跟源，要么盯 x0/x1/x2。
*   **单 trace 只能证明「这次跑了什么」。** 214 个 opcode 是这次命中的，不是全部 827 个；常量 vs 数据要换一组 Java 参数再跑一条才能分清。

这台 VM 的「骨架」已经能看了，但离「还原成伪 C」还差几步：

1.  **613 个没跑到的 handler。** 827 项表，214 个命中。换输入、换分支，就会有这次一次都没碰过的 handler。
2.  **CFG。** 现在有每条 VM 指令的 `pre -> post`，但还没把 669,183 条 vline 串成块和边。入口是现成的：第 N 段的 post 对第 N+1 段的 pre，顺序是 `ip+0x10`，对不上就是跳转，回到前面出现过的槽就是循环。
3.  **FREG 的 4 B 索引。** 修 `SCALE`，顺便把 `ldr d`/`ldr s` 重叠两槽的情况补上。

顺序还是那句：**先出块，再出 CFG，伪 C 排最后。** 反了做不出来。

从最初接触这个 SDK，到逐渐摸清它的运行时结构、虚拟机调度机制和算法执行流程，整个过程比预想中复杂，也收获了不少意料之外的东西。  
曾经觉得非常复杂的东西，在真正投入时间、建立模型、逐步验证之后，也开始变得清晰。  
比起最终还原了多少函数、多少命令，我更看重这次研究带来的分析经验。、  
未知的东西总会存在，今天是 SGMAIN，明天也可能是另一套完全陌生的执行体系。  
继续研究，继续积累。  
他日卧龙终得雨，今朝放鹤且冲天。  
不过是些许风霜罢了。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)

最后于 11 小时前 被一只鸭子编辑 ，原因： 格式调整