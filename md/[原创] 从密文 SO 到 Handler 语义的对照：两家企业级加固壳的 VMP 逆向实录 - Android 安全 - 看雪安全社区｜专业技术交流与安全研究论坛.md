> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293216.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> **声明**：仅用于安全技术研究与教学，全部分析在隔离测试设备 / 离线环境中完成。

加固的 VMP 各家做法差异极大，而这个差异直接决定了 "要花多少代价去还原"。  
把两个真实样本并排拆开：

<table><thead><tr><th></th><th><strong>壳 A：SecNeo / Bangcle 企业版</strong></th><th><strong>壳 B：iJiami 企业版</strong></th></tr></thead><tbody><tr><td>分派结构</td><td><strong>中心跳转表</strong>（<code>LDRSW</code> 查表 + <code>BR</code>，16 个 opcode）</td><td><strong>threaded-code 尾调用</strong></td></tr><tr><td>指令集规模</td><td><strong>16 opcode</strong></td><td>归一化表 <strong>256 项双射</strong> + handler 表 <strong>872 项 / 去重 531</strong></td></tr><tr><td>VM 状态</td><td><code>X25</code> 虚拟内存（栈上，初始为加密字节码流）+ <code>W28</code> 状态寄存器 + <code>W19</code> 辅助</td><td>栈上 IP（<code>STACK[0x820]</code>）+ 栈上虚拟寄存器 + 22 个通用寄存器传递状态</td></tr><tr><td>保护对象</td><td><code>cL</code> 的<strong>参数解析 + methodID 分发</strong>（Instruction Stealing）</td><td><strong>壳自身</strong>（业务方法 100% 是标准 Dalvik 字节码）</td></tr><tr><td>还原卡点</td><td><strong>动态侧</strong>：unidbg 在 opcode 10 崩（高位指针 / 上下文失效）</td><td><strong>静态侧</strong>：<code>dispatch</code> 运行时才填，磁盘上为 0</td></tr></tbody></table>

**两条主线结论**：

1.  **同一个词 "VMP" 下，见到了两种完全不同的设计** —— 一个把 VM 当 "数据混淆器"（统一模式 `byte[X25+addr] ^= key`，  
    偷的是**控制流**），另一个把 VM 当 "真正的指令层"（handler 做实际计算，护的是**自己**）。  
    前者要还原 "程序"，后者要还原 "语义"。
2.  **两种典型的卡点，且互为镜像**：[壳 A 卡在模拟环境](#6-%E4%B8%A4%E7%A7%8D%E5%8D%A1%E7%82%B9%E4%B8%8E%E7%A0%B4%E8%A7%A3%E6%80%9D%E8%B7%AF)（unidbg 的指针 / 上下文传递），  
    [壳 B 卡在运行时数据](#6-%E4%B8%A4%E7%A7%8D%E5%8D%A1%E7%82%B9%E4%B8%8E%E7%A0%B4%E8%A7%A3%E6%80%9D%E8%B7%AF)（关键表只在内存里存在）。  
    ⇒ 于是**两边的解法正好可以互换**：A 用 B 的 "带外真机取证"，B 用 A 的 "符号执行模拟器 + trace 对拍"。

> **壳 A**：16 个 opcode 的混淆写已全部提取、跳转表 16/16 逐项验证、`#238` 的解释器路径真机跑通；  
> 但「编号 → Java 方法名」经**命名路六轮、七条路逐条实测后裁决为「运行时观测面已穷尽」**（那层索引不在运行时数据里）。  
> **壳 B**：分派链五跳全部实测闭合、handler 语义覆盖率 **99.9%**（只剩 1 个 `sshll`(SIMD) 形状）；  
> 但 handler 内部的逐指令语义仍未做到。  
> **两次实验都做了，且都如实给了结论**：一次命中（数值级验证）、一次未命中（保留为否决）。

*   **壳 A** 的动态侧跑通了大部分（JNI 调用链、算法回环），最关键的一步 ——  
    **在真实的 `#238` 指令流上跑通 VM**—— 曾被模拟器缺陷挡住，**已用真机带外取证换路跑通**；
*   **壳 B** 的静态侧做得很干净（分派面整张表都挖出来了），但**从 "索引" 到 "handler" 的最后一跳**需要运行时数据。

更巧的是，**两边的卡点性质相反**：

```
壳 A：VM 能跑，但"跑不全" → 卡在环境
壳 B：VM 看得见，但"接不上" → 卡在数据
```

<table><thead><tr><th></th><th>壳 A 样本</th><th>壳 B 样本</th></tr></thead><tbody><tr><td>App（均为脱敏占位）</td><td>某金融 App，<code>com.mobile.demo_bank</code></td><td>某金融 App，<code>com.demo.bank</code></td></tr><tr><td>加固</td><td>某企业版加固（<code>libDexHelper</code> + <code>libdexjni01</code> + <code>libbangcle_risk</code> + <code>libsmkernel</code>）</td><td>某密企业版（iJiami Enterprise）</td></tr><tr><td>VM 所在库</td><td><code>libdexjni01.so</code>（单库）</td><td><code>libexecmain.so</code>（分派层）+ <code>libexec.so</code>（执行体，<strong>跨两库</strong>）</td></tr><tr><td>保护对象</td><td><code>cL</code> 的参数解析 + methodID 分发</td><td>壳自身的 native 代码</td></tr></tbody></table>

**壳 A**

<table><thead><tr><th>素材</th><th>说明</th></tr></thead><tbody><tr><td>壳 A 的 VM 静态分析笔记</td><td>16 opcode 语义表 / W28 状态链符号化 / opcode 10 展开（原文记 160 条，实测 157 条）/ op14-15 / svc</td></tr><tr><td>自写<strong>符号执行模拟器</strong>（逐 opcode 建模）</td><td>X25 bytearray + W28/W19 + 8 个 VM 寄存器</td></tr><tr><td>opcode 10 的反汇编摘录</td><td>二级跳转表与调用序言</td></tr><tr><td>干净脱壳产物（可加载 so + IDA 库）</td><td>103 函数 / 9185 字符串，IDA 可直接打开</td></tr></tbody></table>

**壳 B**

<table><thead><tr><th>素材</th><th>说明</th></tr></thead><tbody><tr><td>解密后的可执行段 dump</td><td>运行时解密出的真代码（匿名可执行段，581,632 B）</td></tr><tr><td>同一段包成的可加载 ELF</td><td>1 个 <code>PT_LOAD</code>：<code>va=0x747674F000</code>、<code>filesz=memsz=0x8E000</code>，供静态分析</td></tr><tr><td><code>r--p</code> 段镜像（36,864 B）</td><td>18 张 handler 表就在里面</td></tr><tr><td>两张表的完整转储</td><td>18 张表的<strong>全部 872 项</strong></td></tr><tr><td>置换表转储</td><td><code>getOpCode</code> 的 <strong>256 项置换表</strong></td></tr><tr><td>两份表的复现脚本</td><td>纯离线扫段 + 判重</td></tr></tbody></table>

<table><thead><tr><th>能力</th><th>壳 A 用到</th><th>壳 B 用到</th></tr></thead><tbody><tr><td>取码</td><td>unidbg 跑通 <code>sub_13F8</code> 全流程 → dump RWX 段</td><td>unidbg 触发自解密 + <code>/proc/&lt;pid&gt;/mem</code> 取匿名段</td></tr><tr><td>反汇编 / 反编译</td><td>IDA（脱壳产物 + 库）</td><td>IDA（解密段包成的 ELF + 库）+ capstone 批量统计</td></tr><tr><td>动态验证</td><td><strong>unidbg JNI 日志</strong>（全链路）+ Java 标准库回环</td><td>自研 ROM 三点打点 + 段取证（带外）</td></tr><tr><td>VM 语义验证</td><td><strong>自写符号执行模拟器 + 真实 trace 对拍</strong>（序列一致；状态值差异已记录）</td><td><strong>真机注入 trace（② 档）</strong>：30/30 与静态模型逐位一致</td></tr></tbody></table>

**（1）脱壳 / 指令抽取类工作**：解决的是 "DEX 拿不回来" ——  
在 ART 里插桩或内存 dump，再把 `code_item` 重新填回去。这类工作的终点在 **Java 层**：拿到能反编译的 dex 就算成功。  
本文不问 "代码在哪"，而问 **"壳的虚拟机长什么样、怎么分派、能还原到哪一步"**—— 两者的技术栈只在 "样本相同" 这一层重叠。

**（2）模拟执行 / 符号执行**：本文壳 A 侧走的就是这条路。  
优点是能 "一步步跑"；代价有两个：① 模拟器的**指针 / 上下文传递**与真机有差异；  
② 模拟器里 "跑得通" **不等于**真机上 "跑得对"。

**（3）动态插桩 / hook**：在有 RASP 的样本上，它们本身就是被针对的对象；因此本文全程用 **"零代码改写" 的定点取数** ——  
只替换一个**数据指针或表项**（`core+0x20`、16 项跳转表、`JNIEnv[215]`），不插桩指令、不改代码页。

**（4）加固壳 VMP 分析**：  
① **两个商用壳并排**；  
② **把分派链跑到运行时**；  
③ **干预式验证**。

> 一句话：本文的定位是 " **结构 + 分派 + 机制** " 三层，并且三层都以**可检验**的形式给出。

`libdexjni01.so` 的 98% 内容是加密数据；明文只有 ~11 KB 的 loader stub。链路是：

```
open("/system/lib64/libdexjni01.so")
 → pread(key, offset=0x2E1E60, 20 B) ← 16 字节 RC4 key 的存放位置
 → 分块 RC4 解密（sub_1184，每块 ≤0x40 字节）
 → 自定义 linker：解析 DYNAMIC / 重定位 GOT（sub_13F8）
 → 调用 JNI_OnLoad（sub_CD0 用哈希查符号表）
```

关键点：**解密目标段不在模块基址上**，而是运行时 `mmap` 出来的 RWX 段 ——  
所以静态在 `module.base+0x13090` 处看到的是密文，必须让 loader 真跑一遍。  
本项目用 unidbg 跑通 `sub_13F8`（期间需注册壳的自定义 `svc 63505` = mmap2 反模拟检查），  
再从正确的基址连续 dump，得到 `payload_clean_full.so`（103 函数）。

```
[LOAD] rx off=0x0 vaddr=0x0 fsz=0x81B00 ← 531 KB 代码
[LOAD] rw off=0x83D38 vaddr=0x87D38 fsz=0x259B4C ← 2.4 MB 数据
entry 0x2D80 · 镜像 0x2E2818
```

`cL` 函数体调用 `loc_10950` —— VM 入口。序言建立 VM 环境后进入主循环：

```
序言
ADRL X9, xmmword_7D492 ; 加密字节码流（虚拟内存初始内容）
MOV W28, #0xE568AAEC ; 状态寄存器初值
ADRL X27, jpt_109E8 ; 主跳转表基址
STUR Q0, [X25,#0x1C] ; 装配虚拟内存 X25

主循环（0x109D4~0x109E8）
UBFX X9, X8, #0, #0x20 ; 取 opcode
CMP W8, #0xF ; 16 个 case
B.HI def_109E8 ; >15 → default
LDRSW X8, [X27, X9, LSL#2] ; 跳转表查表（相对偏移）
ADD X8, X8, X27 ; → 绝对地址
BR X8 ; 分发
```

**入口序言（0x10950 起，实测）**：建栈 `SUB SP,SP,#0x2A0`、把两个入参存到 `[SP,#0x10]`/`[SP,#0x18]`、  
`X21 = X0`（JNIEnv / 对象）、**`X25 = X29 - 0x90`**（虚拟内存的落点），  
随后把 `0x7D492` 处的数据（`ldp q1,q2` + `ldur q0`，共 44 字节）搬进 X25，  
并把**哨兵指针**（`off_107FF0`）快照到 `[SP,#0x20]`（供 opcode 15 校验）。  
最后 `MOV W8,#0xA` —— **VM 的初始 opcode 是 10**，与动态 trace 的首个 opcode 一致。

主跳转表在 `0x112B8`：**16 项，逐项验证全部指回 16 个 opcode 的入口**（相对偏移，`LDRSW + ADD + BR`）。

**壳 A 的 VM 主循环与 16 项跳转表（IDA 反汇编）**

![](https://bbs.kanxue.com/upload/attach/202610/1006543_WF8K874ZTQWGMSB.webp)  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_MJRZR88UTFZHCW5.webp)

三个状态对象：

*   **`X25`：虚拟内存**—— 栈上的缓冲区，初始内容 = `0x7D492` 处的加密数据流；
*   **`W28`：状态寄存器**—— 初值 `0xE568AAEC`，每个 opcode 后更新，形成**状态链**；
*   **`W19`：辅助状态**—— 初值 `0x3DC1`，供 opcode 8/9/10/14 使用。

对整段解释器逐个 case 做**符号化反汇编提取**，得到 16 个 opcode 的语义表 ——  
每个 opcode 的 "X25 混淆写次数" 与 `next` 都是可核的：

<table><thead><tr><th>opcode</th><th>地址</th><th>X25 混淆写</th><th>next</th><th>备注</th></tr></thead><tbody><tr><td>0</td><td>0x109EC</td><td><strong>1</strong> 次</td><td>0xC</td><td>非线性：<code>addr/key</code> 含 <code>W28*W28</code></td></tr><tr><td>1</td><td>0x109D0</td><td><strong>0</strong></td><td><strong>0xF</strong></td><td><strong>纯跳转</strong>（只有一条 <code>MOV W8,#0xF</code>）</td></tr><tr><td>2</td><td>0x10A30</td><td><strong>7</strong> 次</td><td>0xD</td><td>最短的一段批量混淆</td></tr><tr><td>3</td><td>0x10AC8</td><td><strong>4</strong> 次</td><td>0xB</td><td></td></tr><tr><td>4</td><td>0x10B44</td><td><strong>3</strong> 次</td><td><strong>动态</strong></td><td>用 <code>BFI</code> 拼 key；无固定 <code>MOV W8</code></td></tr><tr><td>5</td><td>0x10BAC</td><td>0</td><td>1</td><td>条件分支：读 <code>[SP,#0x28]</code> 决定走向</td></tr><tr><td>6</td><td>0x10C0C</td><td>0</td><td>1</td><td>纯状态更新：<code>W28 = (W28 &amp; K1) ^ K2</code></td></tr><tr><td>7</td><td>0x10C2C</td><td><strong>4</strong> 次</td><td>6</td><td>末尾带 <strong><code>SVC #0</code></strong>（外部调用，返回 −4 时重试）</td></tr><tr><td>8</td><td>0x10CE0</td><td><strong>5</strong> 次</td><td><strong>动态</strong></td><td>混用 <code>W19</code>：<code>MUL W11,W19,#0xDBEC</code></td></tr><tr><td>9</td><td>0x10D90</td><td>0</td><td>2</td><td>改写 <code>X29</code> 栈区 3 个字节（<code>-0x80/-0x87/-0x76</code>）</td></tr><tr><td><strong>10</strong></td><td>0x10DC8</td><td>0</td><td>动态</td><td><strong>157 条指令的 JNI 调用指令</strong></td></tr><tr><td>11</td><td>0x11048</td><td><strong>5</strong> 次</td><td>0xE</td><td>addr/key 用 <code>2*W28</code></td></tr><tr><td>12</td><td>0x110E8</td><td><strong>3</strong> 次</td><td>7</td><td></td></tr><tr><td>13</td><td>0x1116C</td><td><strong>3</strong> 次</td><td>3</td><td><code>MADD</code> 非线性</td></tr><tr><td>14</td><td>0x111D0</td><td><strong>6</strong> 次</td><td>8</td><td>表达式已逐条核对（见下）</td></tr><tr><td>15</td><td>0x1127C</td><td>0</td><td>—</td><td><strong>结束指令</strong>：哨兵校验 → 弹栈 RET</td></tr><tr><td></td><td></td><td><strong>合计 41 次</strong></td><td></td><td></td></tr></tbody></table>

**统一模式**：`byte[X25 + addr] ^= key`，其中 `addr`/`key` 都由 `W28`（+`W19`）经混淆算术派生 ——  
也就是说，**数据在执行过程中被就地解密**。以 opcode 14 为例（6 条，与手工核对逐字一致）：

```
X25[(W28 + 0x8A0A8765) + 0x13] ^= (mem[(W28+0x8A0A8765)+0x13] ^ (W28 + 0x8F))
X25[(W28 ^ 0x75F57882)] ^= (mem[W28^0x75F57882] ^ (W28 + 0xC1))
X25[(W28 + 0x8A0A8765) + 0x18] ^= ((W28 ^ mem[(W28+0x8A0A8765)+0x18]) ^ 0xEA)
X25[(W28 + 0x8A0A8765) + 0x08] ^= ((W28 ^ mem[(W28+0x8A0A8765)+0x08]) ^ 0xCF)
X25[(W28 + 0x8A0A8765)] ^= (mem[W28+0x8A0A8765] ^ (W28 + 0xA1))
X25[(W28 ^ (0x75F57882+0x2B))] ^= ((W28 ^ mem[W28^(0x75F57882+0x2B)]) ^ 0xF0)
W28 ← W28 + 0x5112151D ; 状态链推进
```

**`W28` 状态链只有 4 类更新原语**，这是 "可符号化" 的前提：

<table><thead><tr><th>原语</th><th>性质</th><th>出现在</th></tr></thead><tbody><tr><td><code>ADD/SUB</code>（寄存器或立即数）</td><td><strong>仿射</strong></td><td>op 2/3/7/11/12/14</td></tr><tr><td><code>EOR</code></td><td><strong>仿射</strong></td><td>op 5/6/7/12</td></tr><tr><td><code>LSL</code> / <code>AND</code></td><td>位混淆</td><td>op 4/8/6</td></tr><tr><td><code>MUL</code> / <code>MADD</code></td><td><strong>非线性</strong></td><td>op 0/4/13</td></tr></tbody></table>

> **⚠️ 一处待解的疑点**：`addr` 由 32 位寄存器算出（如 `W28 + 0xAFD109F9`），  
> 在 `W28` 初值 `0xE568AAEC` 下会得到**远大于栈区的偏移**。这说明二选一：  
> 要么这些 case **只在特定的 `W28` 值（状态链推进到某处）时才会被执行**，  
> 要么它们的地址语义需要按某种截断 / 取模理解。**目前静态无法判定**，必须靠动态 trace 的 `W28` 序列来闭合。  
> **混淆写轮给出答案：是 "前者"**—— dispatch trace 实测该流程里 VM **只执行 op1/op5/op10/op15**（三次运行一致），  
> 其余 12 个 opcode（也就是 41 条混淆写所依附的全部）**一次都没被 dispatch**；  
> 也就是说**只有状态链推进到别的 `W28` 序列时才会进入那些 case**（而那条短循环四步的混淆写次数都是 0）。

opcode 10 占 **157 条指令**（实测），是 "偷指令" 真正发生的地方：

```
入口: X21 = X0（JNIEnv/对象）, [SP,#0x18] = X1（参数数组）
10dc8: LDR X8, [X21] ; LDR X8, [X8,#0x558] ; → CallStaticObjectMethodV 槽位
10ddc: BLR X8 ; 第 1 次 JNI 调用
10de8: SUB W23, W0, #1 ; W23 = 返回值−1 = 方法索引
10df8: BLR X8 ; 第 2 次 JNI 调用（CallObjectMethod 槽位 0x568）
10e10: BL sub_11328 ; JNI 调用包装（压栈 + 哨兵校验 + 间接调用）
10e14: BL loc_12BD4 ; ★ methodID 解析：LDR X0,[表, W0*8]
10e18: MOV X24, X0 ; → X24 = VM 寄存器文件基址
10e20: B def_10E64 ; 内嵌二级分发（12 case，W19 混淆算术算目标）
```

即：**opcode 10 = `methodID 查表 → JNIEnv 槽位调用 → 按返回类型（12 case）写 VM 寄存器文件`**。  
`cL(#239)` 里的 `KeyGenerator.getInstance("AES")` 调用链正是经此完成 —— 与动态 JNI 日志一一对应。

**编号探针 ·1（把 "参数面" 直接读出来）**：把 `JNIEnv` 表 `+0x568` 那一项换成探针（零代码改写）后，  
**VM 自己发起的 1345 次调用**（caller 落在 payload 的匿名 `rwxp` 段）里，`x2`（即上面的 `W23 = W0 − 1`）取值是：

<table><thead><tr><th>x2</th><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th></tr></thead><tbody><tr><td>次数</td><td>406</td><td>388</td><td>289</td><td>180</td><td>82</td></tr></tbody></table>

⇒ `W23` 是**参数数组的序号**（与 "逐参数解析"、模拟器里的 `w22/w23` 参数计数完全吻合），  
**不是** "被保护方法的编号"（那是 `#238`/`#239` 的量级）。这也解释了为什么 " 在 VM 层区分 `#238`/`#239`" 注定做不到：  
**VM 层结构上不含这个信息**（见本节后面的实测）。

**编号探针 ·2（同一天，把 "方法编号" 本身也读出来）**：编号探针 ·1 只 hook 了 `+0x568`，  
`+0x558`（第一次调用）与 `loc_12BD4`（methodID 解析）那两格是空的。本轮把 `loc_12BD4` 的**入口**  
改成一条 `b <trampoline>`（**只覆盖 1 条指令，其余原样重放**），记下它的入参 `W0`：

<table><thead><tr><th>项</th><th>实测</th></tr></thead><tbody><tr><td><code>loc_12BD4</code> 的真实语义</td><td><code>jmethodID = (*(payload+0x2E2580))[W0]</code> —— <strong>就是一个 "编号 → methodID" 的查表器</strong></td></tr><tr><td>入参 <code>W0</code>（= <strong>方法编号</strong>）</td><td><strong>63 个不同编号</strong>，范围 <strong><code>0x13d</code> ~ <code>0x1057</code></strong>；同一编号重复出现（<code>0x76b</code> 出现 19 次）⇒ <strong>稳定标识</strong></td></tr><tr><td>methodID 表</td><td>基址 <code>*(payload+0x2E2580)</code> = <strong>索引表</strong>：容量 <code>0x2400</code>（9,216）项、其中 <strong>7,594</strong> 项非零，每项指向一个 <code>malloc(0x28)</code> 的 40 B 私有描述符（<strong>分配块间距 <code>0x30</code>，不是 <code>ArtMethod</code> 大小</strong> ）；表由壳用 <code>sub_12688</code> 从私有流建出，<strong>私有流的有效项数是 4,186</strong> —— 三者不是同一个量</td></tr><tr><td><code>+0x558</code> 槽</td><td>VM 自己发起 <strong>111</strong> 次（<code>+0x568</code> 槽 <strong>601</strong> 次）；<code>x2</code>（参数序号）分布与编号探针 ·1 一致（<code>1..5</code>）</td></tr></tbody></table>

*   编号探针 ·1 问 " 为什么在 VM 层搜不到 `#238`/`#239`"，回答是"**VM 层结构上不含该方法编号** "；
*   本轮直接看到：**编号确实存在**，但它是 `loc_12BD4` 的**瞬时寄存器实参**（`W0`），  
    **从不以 VM 状态形式落盘**—— 所以 "在 VM 寄存器文件 / X25 数据区里搜编号" 注定搜不到，  
    而在 **JNI 侧查表器入口一定拿得到**。

**命名路：六轮实验，把「编号 → Java 方法名」这条锚点逐条走完**。  
一共**七条路（六条运行时 + 一条静态）全部实测排除**，只剩 "行为级对拍"（高成本、非 "名字级"）：

<table><thead><tr><th>#</th><th>路</th><th>手段</th><th>实测结果</th></tr></thead><tbody><tr><td>1</td><td>JNI 反射</td><td><code>dlopen("libart.so")</code> → <code>dlsym</code> → 跨线程取 <code>JavaVM</code> → <code>ToReflectedMethod</code></td><td>❌ 四个坑全撞上：libart 在 APEX ⇒ linker namespace 拒绝、<code>dlsym(RTLD_DEFAULT,…)</code> = NULL、跨线程取 <code>JavaVM</code> ⇒ ART <code>abort()</code>、attach 线程里 <code>ToReflectedMethod</code> 死锁</td></tr><tr><td>2</td><td>描述符 → 真 <code>ArtMethod</code></td><td>静态反汇编 libart 的 <code>ArtMethod::PrettyMethod</code> 拿标准布局，再套回表项</td><td>❌ 表项 <code>+0x00</code> <strong>严格等于表索引</strong>（不是类指针）⇒ 它不是 <code>ArtMethod</code>，是壳私有 48 B 描述符（<code>4+4+8+8+8+8+8</code>）</td></tr><tr><td>3</td><td><code>sub_11328</code> → <code>ArtMethod</code></td><td>换算 <code>op10</code> 的两个 JNI 槽名 + 反汇编 <code>sub_11328</code></td><td>❌ <code>+0x558</code>=<code>GetArrayLength</code>、<code>+0x568</code>=<code>GetObjectArrayElement</code>；<code>sub_11328</code> 只是 <code>CallXxxMethodV</code> 的 <code>va_list</code> 包装 ⇒ 描述符里没有任何字段是 / 指向 <code>ArtMethod</code></td></tr><tr><td>4</td><td>JNI 按名字取 id</td><td>hook <code>FindClass</code>/<code>GetMethodID</code>/<code>GetStaticMethodID</code>/<code>GetFieldID</code>/<code>GetObjectClass</code> 五槽 + 同步记编号</td><td>❌ <strong>622 条 JNI 记录里 payload 段 caller = 0</strong> ⇒ 壳 A 的 VM 不通过 JNI 表按名取 id</td></tr><tr><td>5</td><td>dex 指纹反查</td><td>用「方法体头（<code>code_item</code> 的 <code>regs/ins/outs/tries</code>）+ 数据长度」当指纹查 8 份 payload dex</td><td>❌ 2001 项只 "唯一命中" 171~261 项，且同一 idx 在不同 dex 命中不同方法 ⇒ 纯碰撞、不可用</td></tr><tr><td>6</td><td>解名字池编码</td><td>离线定性 <code>sub_4460</code>（子串查找）/<code>sub_5680</code>（intern），再解池编码</td><td>❌ 池是壳自己的<strong>字符串常量池</strong>，<strong>池里没有编号结构</strong>（连续递增 u32/u16 run = 0）；真机三 hook 探针 <code>POOL 0 / SRCH 0 / MID 273</code> ⇒ 运行期不再访问池</td></tr><tr><td>7</td><td>静态语义</td><td>反编译 4186 个方法体</td><td>❌ 方法体头部是标准 <code>code_item</code>（91.2% 满足 <code>len == 16 + insns_size*2 + tries*8</code>），但指令载荷不是标准 Dalvik（首指令为寄存器类指令的 364 条里，寄存器编号 <strong>100% 越界</strong>）</td></tr></tbody></table>

⇒ 三条结论：

1.  **编号确实存在，但只以 "瞬时寄存器实参" 的形式存在** —— 它是 `loc_12BD4` 的入参 `W0`，**从不落进 VM 状态区**，所以 "在 VM 寄存器文件 / X25 数据区里搜编号" 注定搜不到；
2.  **名字也确实是明文，但和编号是两套坐标** —— 名字在 `payload` 静态区 `0x108000` 起的字符串常量池里（用 ≥0x80 引用码做前后缀复用压缩），池里没有任何编号结构；
3.  ⇒ **"编号 → 方法名" 在运行时数据里不存在**（不是 "没找到"）—— 这条锚点因此**裁决为 "运行时观测面已穷尽"**。

> **边界**：本项是**代码级最小插桩**（只覆盖 1 条指令、其余原样重放），与其它各项的 "零代码改写" 口径不同；  
> 装置：一个 hook `loc_12BD4` 入口的注入体（只覆盖 1 条指令、其余重放）。

基于上述语义实现了 Python 符号执行模拟器（`X25` bytearray + `W28`/`W19` 32-bit + `vm_regs[8]`），  
四步验证全部通过：

1.  **确定性**：同一 op 路径两次运行状态完全一致（`W28` 终值 `0x5552AA27` —— 这是**修正前旧实现**的读数，修正后的值见下面的真机补测表）；
2.  **仿射性质**：`W28 += 0x5112151D` 经两组初值验证为 "初值 + 增量"；
3.  **全 opcode 覆盖**：16 个 opcode 单独运行均无异常；
4.  **真实路径对拍**：以动态 trace 得到的 op 序列 **`10, 5, 1, 15`** 驱动模拟器，完整走完解释器生命周期  
    （解析 methodID=239 → 模拟 JNI → 条件分支 → 数据 XOR → 哨兵校验通过 → 弹栈 RET），  
    与 unidbg trace + JNI 日志闭环。

⇒ **#239 这条路径是 "真闭环" 的**：解释器语义 + 状态链 + 模拟器三者互证。

> **真机补测—— 这条 "闭环" 曾有一个缺口，现已按实测补上**：首次对拍时  
> **只有 opcode 序列一致、状态值并不一致**；按反汇编推出公式并用真机读数校验后，四步状态值**全部对上（4/4）**：
> 
> <table><thead><tr><th>步骤</th><th>真机实测</th><th>修正前模拟器</th><th>修正后</th><th></th></tr></thead><tbody><tr><td>进入 op10</td><td>W28=<code>0xe568aaec</code>、W19=<code>0x3dc1</code></td><td>同</td><td>同</td><td>一致</td></tr><tr><td>进入 op5</td><td>W28=<code>0xe568aaec</code>、W19=<code>0x4d6d</code></td><td>W19=<code>0x7827</code> ✗</td><td>W19=<code>0x4d6d</code></td><td><strong>已修正</strong></td></tr><tr><td>进入 op1</td><td>W28=<code>0x7b022248</code>、W19=<code>0x4d6d</code></td><td>W28=<code>0x5552aa27</code> ✗</td><td>W28=<code>0x7b022248</code></td><td><strong>已修正</strong></td></tr><tr><td>进入 op15</td><td>W28=<code>0x7b022248</code>、W19=<code>0x4d6d</code></td><td>同</td><td>同</td><td>一致</td></tr></tbody></table>
> 
> 两处修正都来自**反汇编**，不是凑数：
> 
> *   `op5`（`0x10BAC`）：`mov w8,#0x9E6A0000` + `movk w8,#0x88A4` + `eor w28,w28,w8`  
>     ⇒ **`W28 ^= 0x9E6A88A4`**；而 `0xE568AAEC ^ 0x9E6A88A4 == 0x7B022248` ✓ 与真机读数**逐位吻合**；
> *   `op1`（`0x109D0`）：实为**主循环分发器**（`mov w8,#0xf; ubfx x9,x8,#0,#0x20; cmp; b.hi; ldrsw; add; br`）  
>     ⇒ 它**不写 X25**、`next = 15`（旧实现臆造了一次 XOR 且返回 `0xC`，两处都错）；

> ✅ **补完：`op10` 之后的 `W19` 也从 "实测值" 升为 "公式"**。  
> op10 是一条**内嵌二级分发**（`x26 = 0x112F8`，一张 12 项表）；真实路径逐跳解开后是：
> 
> ```
> idx9（按返回类型把 X24 的 8 个字段搬进 VM 寄存器文件）
> → idx0 W19 ← 0x176b；w22 ← 0
> → idx6 w8 = ((W19 * 0xC438) >> 0x17) * (-0xA7) + W19 ；W19 ← 0x4e03 - w8
> （若 w22 < w23：→ idx2 再调一次 JNI，W19←0x7827 → idx3（w22 += 1）→ 回到 idx6）
> → idx4 收尾，**不写 W19**（只算 w9 = 0x6b*w20 + 0x25 - W19）
> → idx11 回主循环；循环头 `mov w8,#5` ⇒ **next opcode = 5**（正是真机序列 `10→5→1→15`）
> ```
> 
> 代入 `0x176b` ⇒ `w8 = 0x96` ⇒ **`W19 = 0x4e03 - 0x96 = 0x4d6d`**，与真机读数**逐位吻合**；  
> 且穷举 `w23`（= 参数数 − 1）为 0/1/2/4/8 **都得到同一个 `0x4d6d`**⇒ 它是这条**参数循环结构的不动点**，  
> 与本条路径上的运行时数据无关（`X24` 的内容只影响 VM 寄存器文件与下一 opcode）。  
> 复现：纯离线，把二级跳转表与真机读数对拍。

`#238`（AES/CBC 加密）进入 VM 后，unidbg 在 **opcode 10 的大 case 处崩掉**：

```
opcode 10 入口要求 X21 = JNIEnv、[SP,#0x18] = 参数数组，
但崩时 X21 = 0xe4ffefc9、SP = 0xe4fff700 —— 均为野指针。
```

判断：  
主解释器 → opcode 10 的寄存器 / 栈上下文传递在 unidbg 中失效，  
疑似壳在 opcode 9→10 的寄存器保存 / 恢复中发生高位指针被截断（低 32 位被损坏），  
而非 `longjmp`/ 异常切栈（解密后的 payload 中未见 `setjmp/longjmp/signal` 特征）。

三条旁证交叉确认：

1.  静态字符串：解密产物中唯一的 `AES/` 算法串是 `AES/CBC/PKCS5P…`；
2.  参数映射：Java 侧签名为（明文, key, iv）；
3.  **Java 标准库回环**：`AES/CBC/PKCS5Padding` 等价复现，6 组明文加解密全部 PASS。

**静态能做到哪一步**：16 个 opcode 的**混淆写已经全部提取**（41 条），  
连 opcode 之间的 `next` 也已确定（多数固定、两个动态）。但要让这 41 条**真正解开 X25 的内容**，  
必须先知道路径上每一刻的 `W28` —— 没有动态 trace，`W28` 就是未知的。  
⇒ **壳 A 的 "最后一格" 不是 "看不懂汇编"，而是 "缺一份运行时状态序列"**——  
而这恰好可以用壳 B 项目里已经跑通的 "真机带外取证 + ROM 打点" 补上。

> ✅ `#238` 所在的解释器路径在**真机上跑通了**，且用的是**零代码改写**的办法  
> 只把 16 项跳转表——RWX 段里的**数据**——换成记录 stub。  
> 多轮实测给出的结论：
> 
> <table><thead><tr><th>读到的项</th><th>真机实测</th><th>与静态记录的关系</th></tr></thead><tbody><tr><td>调用序列</td><td><strong>只有 <code>10,5,1,15</code> 一种</strong>（每次调用都如此）</td><td>与 unidbg trace <strong>完全一致</strong></td></tr><tr><td>进入 <code>op=10</code> 的状态</td><td>W28=<code>0xE568AAEC</code>、W19=<code>0x3DC1</code></td><td>与静态记录的初值<strong>逐位吻合</strong></td></tr><tr><td>状态推进</td><td><code>op=5</code> 后 W28 <code>0xE568AAEC→0x7B022248</code>；<code>op=10</code> 后 W19 <code>0x3DC1→0x4D6D</code></td><td>静态只有符号式，这是<strong>首次实测值</strong></td></tr><tr><td><strong>X25 的 44 字节数据区</strong></td><td>全程<strong>恒定</strong>（48 B 区间的取值数 = 1）</td><td>与语义表 "<code>op 1/5/10</code> 都是 <strong>0 次 X25 写</strong> "<strong> 逐条吻合</strong></td></tr></tbody></table>
> 
> ⇒ 四点结论：  
> ① **"卡点在模拟环境而非 VM 语义" 被真机数据直接证实**—— unidbg 崩掉的 opcode 10 在真机上正常执行，  
> 后续 `5,1,15` 完整走到 `op15` 哨兵校验通过；  
> ② 文章缺的 "运行时状态序列" 这一格**补上了**（W28/W19 实测值）；  
> ③ **X25 在这条路径上不被写**—— 所以 "X25 里藏着 VM 字节码程序" 这个说法要修正：VM 的 "程序" 就是  
> 这条**由分发流程决定**的 opcode 序列，X25 只是 44 字节的初始化数据区；  
> ④ 顺带查出**现有模拟器的状态更新与真机不一致**，并已按反汇编 + 实测**修正到 4/4 一致**。
> 
> **关于 "逐条区分 #238 与 #239"**：两次专门采集（v6 读 X25 区 96 B、v7 再读栈帧上部 `X29+0x00..0x2F`）  
> 都**没有**在任何 VM 层可见位置找到 `methodID`（238/239 的 u32/u64 搜索均未命中）。  
> 这反而给出更强的结论：**VM 层对被保护方法完全同构**（序列、W28/W19、X25 全同）——  
> `#238` 与 `#239` 在 VM 层的执行**一模一样**，差异只落在 `op10` 内部处理的 JNI 方法上。  
> 所以 "区分" 在 VM 层既**不必要也不可行**（要抓 methodID 得在 op10 handler 内部插第二个记录点，那就不是 "零代码改写" 了）。  
> ✅ **补：那个 "第二个记录点" 已经插上了**—— 把 methodID 查表器 `loc_12BD4` 的**入口**  
> 换成一条 `b <trampoline>`（**只覆盖 1 条指令、其余原样重放**），就拿到了它的入参 `W0` = **方法编号本身**  
> （63 个不同编号、范围 `0x13d~0x1057`、同一编号稳定重复）。结论因此更精确：  
> **编号不是 "搜不到"，而是 "不以 VM 状态形式存在"**—— 它是 `loc_12BD4` 的瞬时寄存器实参。

**③ 档扰动实验：先预测、再让真机证伪**—— 前两档都是 "观察式一致"，这一档是**干预式**：

<table><thead><tr><th>环节</th><th>内容</th></tr></thead><tbody><tr><td>假设</td><td>16 项跳转表决定 opcode 分发；且 <code>op5</code> 负责 <code>W28 ^= 0x9E6A88A4</code></td></tr><tr><td>扰动</td><td>注入体加 <code>-DREMAP_FROM=5 -DREMAP_TO=1</code> ⇒ <strong>表项 [5] 指向 stub_1</strong>（任何 <code>next=5</code> 都落到 <code>op1</code> handler）——<strong>仍然只改数据表、不改指令</strong></td></tr><tr><td>预测（事先写下）</td><td>① 序列由 <code>10,5,1,15</code> 变 <strong><code>10,1,15</code></strong>；② 第 2 条是 <code>op=1</code> 且 <strong>W28 = <code>0xE568AAEC</code></strong>；③ <code>op=5</code> 记录数 = 0；④ <code>op15</code> 的 W28 也不再推进</td></tr><tr><td>对照</td><td>同源码<strong>不加宏</strong>（对照组 so，md5 与 v7 一致）⇒ 唯一变量就是那一个表项</td></tr><tr><td>实测</td><td>实验版 op 分布 <code>{1:330, 10:330, 15:329}</code>（<strong>无 op5</strong>）、<code>op=1</code>/<code>op=15</code> 的 W28 <strong>全部</strong><code>0xE568AAEC</code>；对照版回到 <code>{1:219, 5:218, 10:221, 15:219}</code></td></tr></tbody></table>

⇒ **4/4 命中**：模型给出了可被推翻的预测，而真机没有推翻它。

** 第二个扰动（更强）：表项 [15] → stub_1。** 假设 `op1` 的 `next = 15` ⇒ 分发链会变成 `op1 → op1 → op1 …`。  
事先预测：VM **永不退出**⇒ ① trace 出现连续大量 `op=1`；② 32,768 条的环形缓冲被打爆（日志出现 `# OVERFLOW`）；  
③ 该调用线程被拖死。实测：**记录 4,845 条里 4,842 条是 `op=1`**（99.9%）、末尾 20 条全为 `op=1`、  
日志出现 **`# OVERFLOW pos=5798855`**（缓冲溢出时的分派计数），App 进程随即消失 ——  
而在同一触发流程下正式版每次调用只有 **4 条**记录。⇒ **预测成立**（相差 5 个数量级）。

③ 档只动过**两个表项**（[5]、[15]）。既然跳转表只有 16 项，那就把**每一项都单独试一遍**——  
这样 "③ 档" 就不再是 "分派层的抽查"，而是**逐条 opcode 的可证伪验证**。

方法：对 `i = 0..15` 各跑一轮，注入体编译时把**表项 [i] 重定向到 `stub_1`**（`-DREMAP_FROM=i -DREMAP_TO=1`），  
每轮**先 `force-stop` 重启 App**（干净起点，上一轮的注入随进程消失），并另有一个**同源码不加宏**的对照组。

**事先写下的预测**（机械、可被推翻）：模型给出的真实路径是 `10,5,1,15` ⇒

*   `i ∈ {5,10,15}`（在路径上、且不是自映射）⇒ **受影响**；
*   `i = 1` ⇒ 也是路径上的 opcode，但 `REMAP_TO=1` 使它**重定向到自身**（`stub_1` 记录 `op=1` 后跳回原 handler[1]）  
    ⇒ 天然是一个**恒等对照**⇒ **不受影响**；
*   其余 12 个 ⇒ **不受影响**。

<table><thead><tr><th>i（重定向的表项）</th><th>预测</th><th>实测（记录数）</th><th>与对照</th><th>判定</th></tr></thead><tbody><tr><td>0, 2, 3, 4, 6, 7, 8, 9, 11, 12, 13, 14（12 个）</td><td>不受影响</td><td>不受影响（388~576 条）</td><td><strong>同形</strong>（op 占比 25/25/25/25、最常见序列 <code>10,5,1,15</code>）</td><td>✅</td></tr><tr><td>1（自映射 ⇒ 恒等对照）</td><td>不受影响</td><td>不受影响（572 条）</td><td>同形</td><td>✅</td></tr><tr><td>5</td><td>受影响</td><td>受影响（381 条）</td><td><strong>不同</strong>：op 占比 <code>{1,10,15}</code> 各 33%、序列变 <strong><code>10,1,15</code></strong>（op5 消失）</td><td>✅</td></tr><tr><td>10（JNI 调用入口）</td><td>受影响</td><td>受影响（<strong>只剩 2 条</strong>：<code>op1</code>,<code>op15</code>）</td><td>不同</td><td>✅</td></tr><tr><td>15（结束指令）</td><td>受影响</td><td>受影响（<strong>0 条</strong>）</td><td>不同</td><td>✅</td></tr></tbody></table>

⇒ **16 / 16 命中**。三条推论：

1.  **"哪些 opcode 在真实路径上" 被独立验证**：12 个路径外表项怎么改都与对照同形；
2.  **路径上的三项各自给出了 "性格证据"**：`op5` 一改 ⇒ `W28` 不再推进（序列少一条）；`op10`（JNI 调用入口）一改 ⇒  
    VM 立刻走不下去（只剩 2 条记录）；`op15`（结束指令）一改 ⇒ **一条记录都没有**（无法正常收尾）；
3.  `i = 1` 的自映射意外提供了一个**组内恒等对照**（改与不改完全同形）—— 也顺带说明 "分派切换点就是这 16 项表项"。

> 环境前置（本轮踩过的坑，值得记）：① 壳 A 样本**同样要先开反检测环境**（`/data/local/tmp/start_bank.sh`：  
> 时间窗 + `setup_stealth` + 重启 zygote）——不跑的话 ptrace attach 会一直挂着；② 注入目标要选**主进程**  
> （`ps` 里 NAME=`com.mobile.bank` 且 **PPID 不在本 App 的 pid 集合内**的那个——壳还有同名子进程 / 隔离进程）；  
> ③ App 冷启动后要等 **~14 s** 再注入（`payload` 还没解密完时注入体会报 "16 项跳转表未找到"）。
> 
> 完整 16 轮原始记录与逐条判定；分析与批量采集脚本均为纯离线。

在动手之前先回答一个问题：**壳 B 的 VMP 到底保护谁？**  
判据是 Dalvik opcode 的合法区间 —— `0x00–0xE2` ∪ `0xFA–0xFF`，  
其中 `0xE3–0xF9` 是 AOSP 保留的 unused 区：**若壳把方法换成了自定义指令集，必然在这里露馅**。

对 5 份业务 / 三方 dex（**合计 210,061 个有代码方法**，按本表求和）实测：

<table><thead><tr><th>dex</th><th>有代码方法</th><th>用到的 opcode</th><th>结论</th></tr></thead><tbody><tr><td>主业务 dex</td><td>45,746</td><td>99 种，最大 <code>0xE2</code></td><td>✅ 全标准</td></tr><tr><td>业务 + 壳检测</td><td>53,861</td><td>140 种，最大 <code>0xE2</code></td><td>✅ 全标准</td></tr><tr><td>壳 RASP</td><td>3,361</td><td>51 种，最大 <code>0xE2</code></td><td>✅ 全标准</td></tr><tr><td>androidx / hutool</td><td>50,495</td><td>137 种，最大 <code>0xFB</code>（标准新指令）</td><td>✅ 全标准</td></tr><tr><td>三方库合集</td><td>56,598</td><td>125 种，最大 <code>0xE2</code></td><td>✅ 全标准</td></tr></tbody></table>

> **复算与口径（补）**：  
> 需要说明的是：工作区现存的 dex 与做这张表时用的那一批存在  
> **±1.6% 的版本差异**（"三方库合集" 现测 56,598、"壳 RASP" 现测 3,361 与表中一致；  
> 另两份现值 50,497 / 45,021 与表中的 50,495 / 45,746 略有出入）。  
> 差异来自 dump 轮次 —— **不影响结论**：严格口径（每个 `code_item` 的首指令）  
> 的**保留区命中在任何一版上都是 0**，即 "业务方法 100% 标准 Dalvik" 这一点稳定成立。

⇒ **业务方法 100% 是标准 Dalvik 字节码，`ijm_vmp` 只覆盖壳自身**——  
这条判据同时决定了 "要不要继续逆它"。

壳 B 的执行体 `libexec.so` 与壳 A 不同：**它不是 UPX**（`PT_LOAD(rx)` 的 `filesz == memsz`），  
而是**代码段整体加密**—— 磁盘上按 4 KiB 扫描，多处 "可解码指令为 0"，`.symtab` 也被破坏。  
真正的代码在**运行时解密到「紧跟 so 之后的匿名可执行段」**：

<table><thead><tr><th>so</th><th>file <code>r-xp</code>（数据 / 未解密）</th><th><strong>匿名 <code>r-xp</code>（解密后代码）</strong></th></tr></thead><tbody><tr><td><code>libexec.so</code></td><td><code>+0x42000</code></td><td><strong><code>+0x8E000</code>（581,632 B）</strong></td></tr><tr><td><code>libijm-emulator.so</code></td><td><code>+0x9000</code></td><td><code>+0x2D000</code>（184,320 B）</td></tr></tbody></table>

真机上（root、无侵入 `dd if=/proc/<pid>/mem`）取出那 581,632 B 后：

*   容错线性扫描得 **145,233 条指令**（注意：必须**逐 4 字节容错扫描**，因为 `disasm(buf,0)` 在第一条失败处就停；  
    且不能用 "可解码指令数" 判代码 —— capstone 把 `0x00000000` 解成 `udf #0`，一个 4 KiB 全零块能虚报 1024 条）；
*   IDA 识别出 **883 个函数**，其中 **254 个含 `BR Xn`（共 811 处）**—— 这就是 VM 的**分派面**。

壳 B 的分派是**跨两个 so** 的。`libexecmain.so`（唯一导出符号 `getOpCode`，0x3a3c，4244 B）负责前半段：

**(a) `getOpCode` 是一张 256 项的双射置换表。**  
它是纯二分决策树（内部 128 个 `ret`、253 个 `cmp` + 126 个 `csel`，`bl`/`blr`/`br`/`adrp` 全为 0），  
因此可以**静态还原**：反汇编后跑一个只认 `mov`/`cmp`/`and`/`sxtb`/`tbnz`/`b.hs`/`bics`/`csel`/`csinc`/`csinv`/`ret`  
的迷你解释器，喂 0x00–0xFF，得到 **256 个 opcode → 256 个互不相同的内部编号**（值域 `-128..127`）。

**壳 B 的 getOpCode：纯二分决策树的 256 项置换表（IDA 反汇编）**  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_H4VXWZ2NCTS284H.webp)

**(b) 它还构造 "553 项分派描述符"。**  
构造函数（`.init_array[0]`）`malloc(0x40)` 出一个描述符挂到全局 `core`：

<table><thead><tr><th>偏移</th><th>内容</th></tr></thead><tbody><tr><td><code>+0x00</code></td><td><code>(累计偏移, 长度)</code> 对表 —— ✅ 结构<strong>已实测确证</strong>（98 项、总长 553、单条 1~53 B、98/98 自洽、静态可得）；⚠️ 它是 **「553 项方法表」的 98 段分段索引 **（<code>len[i]</code> = 第 i 段含多少个 24 B 方法项），<strong>不是 "变长指令的边界表"</strong></td></tr><tr><td><code>+0x08</code></td><td><strong>553 项函数指针表</strong>（加载时由 <code>.rela.dyn</code> 填充），指向 553 个<strong>跳板 thunk</strong>（体长两种：180 B / 184 B）—— <strong>运行时已验证：553/553 全部指向 libexecmain <code>r-xp</code>，间距 0xB8 为主（545/552）、0xB4 少数（7/552）</strong></td></tr><tr><td><code>+0x10</code></td><td><code>&amp;getOpCode</code></td></tr><tr><td><code>+0x18</code></td><td><strong>0 —— 留给外部</strong></td></tr><tr><td><code>+0x20</code></td><td><strong>0 —— 留给外部</strong></td></tr></tbody></table>

跳板只做一件事 —— 按索引转发：

```
__int64 sub_4AD0(__int64 a1, __int64 a2, ...) { // 第 0 个跳板
 va_start(va, a2);
 v2 = *(__int64 (**)(...))(core + 0x20); // 描述符 +0x20（运行时由外部填）
 va_copy(va1, va);
 return v2(a1, 0, a2, va1); // ← 只把索引 0 透传
}
// 下一个：return v2(a1, 1, a2, va1); …… 一直到 552
```

真正的执行体在 `libexec.so` 的解密段里。把它的 `r--p` 段（36,864 B）按 8 字节小端扫一遍，  
找 "值落在解密段范围内的函数指针" 的连续片段，得到 **18 张表、合计 872 项、去重后 531 个 handler 地址**：

<table><thead><tr><th>指标</th><th>值</th></tr></thead><tbody><tr><td>表数 / 项数 / 去重 handler</td><td><strong>18 / 872 / 531</strong></td></tr><tr><td>规模分布（项数 × 张数）</td><td><code>731×1</code>、<code>25×1</code>、<code>10×2</code>、<code>7×12</code>、<code>6×2</code></td></tr><tr><td>最大一张</td><td><code>+0xAF0</code>，<strong>731 项 / 443 个不同函数</strong></td></tr><tr><td>最高频 handler</td><td><code>0x74767CA8AC</code>，出现 <strong>134 次</strong></td></tr><tr><td>频次分布</td><td>只出现 1 次：371 个；≥2 次：160 个</td></tr><tr><td>地址对齐</td><td>全部 <code>mod 4 == 0</code>（不是 16 字节对齐的函数起始）</td></tr><tr><td>相邻间距</td><td>中位 <strong>60 B</strong>、最小 <strong>4 B</strong>（有 "只做一次尾调用" 的极小 handler）</td></tr></tbody></table>

> 真机实测证明 **`thunk_i` = 某个 Java native 方法的实现指针**  
> （`ArtMethod->data_`）⇒ 它内嵌的常量是 " **第 i 个被注册的  
> native 方法**的编号 "。因此本节指令集对照表里"**VM 指令集 / opcode**" 的表述应理解为  
> " **方法编号的分发空间** "。
> 
> 被一并标为 "待重新审视" 的 " **变长指令编码** " 一项，曾据离线证据判为**成立**——  
> 描述符 `+0x00` 指向的表**当时**被读成 "变长指令的边界表"：`(u32 累计偏移, u32 长度)` 对、**98 项**、  
> 相邻项满足 `off[i+1] == off[i] + len[i]`（98/98 自洽）、累计总长 **`0x229` = 553 字节**、  
> 单条长度 **1 ~ 53 B**。

**这 531 个 handler 到底有几种 "形状"？**（纯静态聚类，无真机：  
把寄存器编号归一成 `xR/wR`、立即数归一成 `#I`、条件码归一成 `CC` 后做指纹）

<table><thead><tr><th>聚类口径</th><th>类数</th><th>前 5 类覆盖表项</th><th>前 10 类覆盖表项</th></tr></thead><tbody><tr><td>严格同形（全序列指纹）</td><td><strong>206</strong></td><td>64.4%</td><td>71.9%</td></tr><tr><td>指令类骨架（<code>LOAD/STORE/MOV/ALU/CMP/BR/CALL/VEC</code>）</td><td><strong>124</strong></td><td>64.4%</td><td>73.7%</td></tr><tr><td>骨架 + 相似度 ≥ 0.90</td><td><strong>59</strong></td><td>80.4%</td><td>89.3%</td></tr><tr><td>骨架 + 相似度 ≥ 0.85</td><td><strong>50</strong></td><td>82.7%</td><td>92.8%</td></tr><tr><td>骨架 + 相似度 ≥ 0.80</td><td><strong>33</strong></td><td>89.4%</td><td>95.3%</td></tr><tr><td>骨架 + 相似度 ≥ 0.70</td><td>8</td><td>99.7%</td><td>100%</td></tr></tbody></table>

⇒ **"531 → 十几类" 这个预期过乐观**：只有把相似度放宽到 0.70 才降到个位数（那时已把语义不同的  
handler 混为一类，失去意义）。**合理口径（0.85~0.90）下是 50~59 类**；但**前 10 类就覆盖 89~93% 的表项**——  
既然 371/531（70%）的 handler 只被引用 1 次、最高频那 1 个就占 15.4%，正确的成本模型是  
**"按频次 × 形状排序，先做头部"**，而不是 "先聚类再全做完"。  
长度也高度集中在三段：7~8 条（149 个）、15 条（135 个）、26~27 条（74 个）。

反编译最高频那个 handler，VM 的形态一目了然 —— **threaded-code**：

```
// handler 模板（表项 0x74767CA8AC）
v23 = STACK[0x820] + 2; // 推进 VM 的 IP（+2）
v25 = STACK[0x81C] ^ ((104 - BYTE4(a22)) & 0x5F | (BYTE4(a22) - 105) & 0xA0) ^ 0x5F; // 混淆 XOR 取操作数
LOBYTE(STACK[0x82C]) = v25; // 写进「栈上的虚拟寄存器」
LOBYTE(STACK[0x81C]) = v25;
STACK[0x820] = v23;
return (*(fn **)(x22 + 8 * LOBYTE(STACK[0x82C])))(a1, ..., a17); // ★ 尾调用下一个 handler
```

即：**VM 状态（IP + 虚拟寄存器）放在栈上，22 个通用寄存器把状态直接传给下一个 handler，分派靠尾调用**  
—— 没有中心 dispatch 循环，所以前面那 **811 处 `br`/`blr`** 就是这 443 个 handler 的尾巴。

**壳 B 最高频 handler 的伪代码：VM 状态在栈上 + 尾调用分派（IDA F5 反编译）**  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_AQBJFQ9D2JG9HPV.webp)

#### 3.4.1 `+0x00` 那张表：**结构上是 "98 段 (off,len) 表"，语义是「553 项方法表的分段索引」**

描述符的 `+0x00` 一直记作 `&unk_29150`（"疑为长度表"）。它确实**就在磁盘上**（`.data` 段，不是运行时才填的），  
把它按 `(u32, u32)` 读出来，**98/98 项全部满足 `off[i+1] == off[i] + len[i]`**：

<table><thead><tr><th>量</th><th>值</th></tr></thead><tbody><tr><td>项数</td><td><strong>98</strong>（表大小 784 B，8 B / 项）</td></tr><tr><td>累计总长</td><td><strong><code>0x229</code> = 553 字节</strong></td></tr><tr><td>单条长度</td><td>1 ~ 53 B（1 B×17、2 B×24、3 B×12、…、53 B×1）</td></tr><tr><td>自洽性</td><td><code>off[0] == 0</code> 且逐项衔接，<strong>98/98 无断点</strong></td></tr></tbody></table>

⇒ 当时（早些时候）据此读成 " **变长指令的边界表** "：每条指令的起点与长度，  
于是壳 B 的 "VM 程序" 第一次有了形状 —— **98 条指令、总长 553 字节**。

> 这个语义**后来被更正**了（结构自洽 ≠ 语义，事后再看这一步是明显的过度推断）：  
> 把描述符 `+0x00` 置成**未映射地址**后，SIGSEGV 直接指认了读它的代码（匿名解密段 `+0x50810`）——  
> 那里的用法是**按段遍历一张 24 字节 / 项的方法表**：
> 
> ```
> ldr w10, [表 + i*8 + 4] ; len = 表[i].len
> cmp w10, #1 ; b.lt ; len < 1 ⇒ 整段跳过
> ldr w9, [表 + i*8] ; off = 表[i].off
> ldr x8, [core+0x08] ; ⇒ 553 项 × 24 B 的方法表
> smaddl x24, w10, #0x18, x8 ; 方法表基址 + off*24
> ```
> 
> ⇒ 它是 **「553 项方法表」的 98 段分段索引**：第 `i` 段覆盖方法表的第 `[off[i], off[i]+len[i])` 项。  
> 而 `off[97] + len[97] = 538 + 15 = 553` 恰好闭合 —— 所以 "总长 553" 与 "553 项表" **不是巧合，是同一个 553**。

而它紧挨着的另一半是 `+0x08` 的 **553 项指针表**：`0x28008 + 553 × 8 = 0x29150`（正好接到边界表开始），  
`0x29150 + 98 × 8 = 0x29460`（正好是 `.data` 结束、`.bss` 开始 —— 那儿只有 `core` 变量那 8 字节）。  
**8 + 553×8 + 98×8 恰好等于 `.data` 的大小 `0x1460`**—— 一个字节不多、一个字节不少：

<table><thead><tr><th><code>.data</code> 段内布局</th><th>VA</th><th>大小</th><th>磁盘上</th><th>内容</th></tr></thead><tbody><tr><td>段头</td><td><code>0x28000</code></td><td>8 B</td><td>—</td><td>（对齐 / 未知）</td></tr><tr><td><strong>指针表</strong>（描述符 <code>+0x08</code>）</td><td><code>0x28008</code></td><td>4,424 B = <strong>553 × 8</strong></td><td><strong>全 0</strong></td><td>553 个跳板；运行时才写入</td></tr><tr><td><strong>边界表</strong>（描述符 <code>+0x00</code>）</td><td><code>0x29150</code></td><td>784 B = <strong>98 × 8</strong></td><td><strong>有初值</strong></td><td><code>(累计偏移, 长度)</code> 对</td></tr></tbody></table>

⇒ 这两张表**不是 " 碰巧都在 `.data`"，而是同一套结构的两半**（一张运行时填、一张静态写死），  
被描述符一起交给 VM。

> （指令集对照表里 "变长编码" 那一行已作废：`+0x00` 是**方法表分段索引**，不是指令边界。）

#### 3.4.1.1 ③ 档：把边界表改坏 ⇒ **自检失败**

静态 "看得见" 还不够 —— 于是做了一次**干预实验**（仍是零代码改写：只写数据，注入体只改边界表里的一个字节；  
把 so 推到 App 的 `lib/arm64/` 后 "启动即注入"，注入体内部每 5 ms 轮询 `core` 变量，一填好就动手）：

<table><thead><tr><th>组</th><th>干预</th><th>触发后主进程 pid</th><th><code>FATAL EXCEPTION</code></th><th>隔离进程崩</th><th><code>chromium</code> 日志行</th></tr></thead><tbody><tr><td><strong>A 对照</strong></td><td>不改（只读 + 记录）</td><td>13877 → <strong>13877（不变）</strong></td><td><strong>0</strong></td><td>0</td><td>25</td></tr><tr><td><strong>B 微扰</strong></td><td>表项 [0] 长度 <code>2 → 8</code></td><td>14998 → <strong>14998（不变）</strong></td><td><strong>0</strong></td><td>0</td><td>26</td></tr><tr><td><strong>C 重击</strong></td><td><strong>98 项长度全 → 1</strong>（总长 <code>553 → 98</code>）</td><td>13227 → <strong>13407（重启）</strong>；14371 → <strong>死亡后重启</strong></td><td><strong>1 / 1（2/2 复现）</strong></td><td>1 / 1</td><td>9 / 7</td></tr></tbody></table>

> 末列 = logcat 里 `chromium` 标签的行数（WebView 侧日志）：A/B 组 25~26 行说明进程跑到了 WebView 初始化，  
> C 组只有 9/7 行 ⇒ **崩得更早**。这一列只是 “进程跑到哪一步” 的旁证，与结论无关。

而 C 组的崩溃栈**直接指认了壳的自检**：

```
FATAL EXCEPTION: main
java.lang.RuntimeException: Unable to create application com.demo.fintech.demoApp:
 com.demo.fintech.encrypt.demoSMExecption: 完整性校验-签名校验失败
 at com.demo.fintech.encrypt.demoSMTools.init(demoSMTools.java:4)
 at com.demo.mobile.kits.support.n.y(Support.kt:4)
 at com.demo.fintech.cmk.e.l(CMKManager.kt:9)
 at com.demo.fintech.demoApp.onCreate(demoApp.java:5)
```

⇒ 三条结论：

1.  **预测命中（③ 档）**：破坏边界表 ⇒ 壳自己的**完整性校验失败**⇒ 主进程崩 / 重启（**2/2 复现**），  
    而 "不改" 与 "只改一项"**3/3 干净**；
2.  **剂量 - 反应**：单点破坏（`2 → 8`）**不崩**、整体破坏（`553 → 98`）**必崩**⇒ 这张表**确实参与运行时流程**，  
    且**容忍微小偏差**（与 "按累计偏移取指令" 的模型自洽：错一处只错一条，错全表则整条指令流错位）；  
    这与判定互相印证 —— 壳 B 护的是**它自己**，所以破坏它的 VM 数据，第一反应就是**自检失败**；
3.  附带事实：从进程起来到 `core` 变量填好只要 **75~150 ms**（"启动即注入 + 5 ms 轮询" 正好卡在这个窗口动手）。

> 边界：崩点在自检链的 Java 侧；" **到底哪一条指令错位、错位后走了哪条岔路** " 还没逐条定位；  
> B 组只试了表项 [0] 一处，不能推出 "任意单点改动都被容忍"。

**再往下钻两层：决定崩不崩的既不是「总量」也不是「形态」，而是「具体哪三项」**

最早一轮（连续区间扫档）给出 "≤12 稳 / ≥16 崩"；又一轮（破坏形态 + 重复）把它修成 " **某段连续错位 ≥16 项** "；  
但把破坏**位置**换一下又对不上了 —— 于是有了这张三轮自我修正表：

<table><thead><tr><th>轮次</th><th>当时的结论</th><th>被下一轮推翻的理由</th></tr></thead><tbody><tr><td>1</td><td>≤12 项稳、≥16 项崩（阈值 ≈28~30 B）</td><td>散布 49 项（<strong>−249 B</strong>）<strong>稳</strong></td></tr><tr><td>2</td><td>触发条件 = 某段<strong>连续错位 ≥16 项</strong></td><td><strong>中段</strong>连续 16 项（<code>[40,56)</code>/<code>[20,36)</code>）<strong>稳</strong>，而 <code>[12,16)</code> 只错 <strong>2 B</strong> 就崩</td></tr><tr><td>3</td><td><strong>关键项 = 表项 <code>{14, 65, 83}</code></strong></td><td>——（3/3 重复 + 相邻项对照）</td></tr></tbody></table>

第三轮的做法：① 同样项数**换位置**；② 专挑 "减少量最大" 的**单点**；③ 二分到 4 项；④ 单点定位 + 重复验证。

<table><thead><tr><th>破坏</th><th>总长 553→</th><th>减少</th><th>结果</th><th>破坏</th><th>减少</th><th>结果</th></tr></thead><tbody><tr><td>单点 idx 94</td><td>501</td><td><strong>52</strong></td><td>稳 2/2</td><td>连续 <code>[40,56)</code> 16 项</td><td>39</td><td>稳 2/2</td></tr><tr><td>单点 idx 93 / 75</td><td>521 / 522</td><td>32 / 31</td><td>稳 2/2</td><td>连续 <code>[20,36)</code> 16 项</td><td>50</td><td>稳 2/2</td></tr><tr><td>单点 idx 18 / 70</td><td>529 / 533</td><td>24 / 20</td><td>稳 2/2</td><td>连续 <code>[82,98)</code> 16 项</td><td>145</td><td><strong>崩 2/2</strong></td></tr><tr><td>连续 <code>[74,78)</code></td><td>486</td><td><strong>67</strong></td><td>稳 2/2</td><td>连续 <code>[10,22)</code> 12 项</td><td>48</td><td><strong>崩 2/2</strong></td></tr><tr><td>连续 <code>[12,16)</code></td><td>551</td><td><strong>2</strong></td><td><strong>崩 2/2</strong></td><td>连续 <code>[60,72)</code> 12 项</td><td>56</td><td><strong>崩 2/2</strong></td></tr><tr><td>连续 <code>[64,68)</code></td><td>524</td><td>29</td><td><strong>崩 2/2</strong></td><td>连续 <code>[74,86)</code> 12 项</td><td>84</td><td><strong>崩 2/2</strong></td></tr><tr><td>连续 <code>[82,86)</code></td><td>541</td><td>12</td><td><strong>崩 2/2</strong></td><td>各 4 项</td><td>35 / 4 / 23 / 5 / 12</td><td>稳 2/2</td></tr></tbody></table>

⇒ **错位量完全不是自变量**（−2 B 崩、−67 B 稳、−52 B 稳）。于是往下做**单项**二分：

<table><thead><tr><th>表项</th><th>结果</th><th></th><th>表项</th><th>结果</th></tr></thead><tbody><tr><td>idx <strong>14</strong></td><td><strong>崩 3/3</strong></td><td></td><td>idx 12</td><td>稳 2/2</td></tr><tr><td>idx <strong>65</strong></td><td><strong>崩 3/3</strong></td><td></td><td>idx 67</td><td>稳 2/2</td></tr><tr><td>idx <strong>83</strong></td><td><strong>崩 3/3</strong></td><td></td><td>idx 85 / 13 / 15</td><td>稳（各 2 或 1 次）</td></tr></tbody></table>

**判据也升了级**：`PID_after` 从 "单次查询" 改成 **6 次 × 1 s 的采样序列**（排除 "恰好查在重启窗口" 的假阳性），  
并用 `logcat -b events` 确认观测到的消失是**进程自己死**（`am_proc_died` 没有对应的 `am_kill`）：

```
idx14 #1 : PID_before=18130 PID_after_seq=[18130 18130 18130 18130 EMPTY EMPTY] ← 触发后第 4~5 秒死
idx12 #1 : PID_before=20437 PID_after_seq=[20437 20437 20437 20437 20437 20437] ← 一直活着
```

⇒ **结论：关键项 = 边界表的 `{14, 65, 83}`**。

*   **单项致命**（各 **3/3**），**相邻项不致命**（12 / 13 / 15 / 67 / 85 全稳，其中 15 与 12 原本就是 1 字节，  
    改与不改同形 —— 正好当 "只注入不改" 的对照）；
*   与**错位量**无关、与**连续段长度**也无关（中段连续 16 项稳、`[12,16)` 4 项崩）；
*   **结构上等价的说法**：边界表是 `(累计偏移, 长度)` 表，改第 `i` 项的**长度**不会动它自己的偏移，  
    而是让 `i+1` 起的所有项整体错位 —— 所以结论也可以读成「**错位起点 ∈ {15, 66, 84} 时壳退出**」  
    （改 13 稳 / 改 14 崩 ⇒ 起点 15；改 84 稳 / 改 83 崩 ⇒ 起点 84，两边都吻合）；
*   **全部三轮数据自洽**：把每一组代入 "`破坏 ∩ {14,65,83} ≠ ∅` ⟺ 崩 "，只剩那 14 种破坏形态里的  
    4 个**散布组**（各 **1 次**崩）是例外 —— 它们都不含关键项，  
    且**重复后全部变稳**⇒ 与本轮实测的 "散布组偶发崩率 ≈16%、单次无判别力" 一致，归为**噪声**。

**崩在哪（两轮合起来才完整）**：只要动了边界表，壳的隔离进程就**普遍**报  
`demoSMExecption: 完整性校验-签名校验失败`（Java `FATAL`）——  
"动边界表 = 壳自检失败" 这条成立；但**主进程是否退出**，只取决于动的是不是 `{14,65,83}`，  
且主进程那一条**没有** Java `FATAL`、也没有新 tombstone ⇒ 更像 native 侧的**静默退出**。

#### 3.4.1.2 **抓到崩因**，于是 `{14,65,83}` 不再是谜

把 "改这张表 ⇒ 壳退出" 一路修到 " 关键项 = `{14,65,83}`"，但**为什么是这三项**一直空着。  
两项实验把它补上。

**(a) 先把崩因抓全**（先前几轮只证明了 "进程消失"）。方法：**先起流式 `logcat` 再冷启动 App**  
（不然环形 buffer 会把关键行冲掉 —— 这套设备上踩过的老坑）。跑 `-DPATCH_IDX=14 -DPATCH_VAL=1`：

```
E/com.demo.hkdemom: No implementation found for void com.demo.fintech.activity.SplashActivity.onDestroy()
 (tried Java_com_demo_fintech_activity_SplashActivity_onDestroy and Java_...)
E/AndroidRuntime: FATAL EXCEPTION: main
E/AndroidRuntime: android.util.SuperNotCalledException:
 Activity {com.demo.hkdemomb/...SplashActivity} did not call through to super.onDestroy()
Zygote: Process ... exited due to signal 9 (Killed)
```

⇒ 崩因是 **`SplashActivity.onDestroy()` 的 native 实现缺失**⇒ 覆写里走不到 `super.onDestroy()` ⇒  
`SuperNotCalledException` ⇒ 主进程被 kill。⇒ **改这张表影响的是「native 方法注册」，而不是「VM 指令流」。**

**改边界表 ⇒ SplashActivity.onDestroy() 的 native 实现缺失 ⇒ SuperNotCalledException（原始 logcat）**  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_JQBK2F4J2DNRCRZ.webp)

**(b) 让 "读表的人" 自己暴露**。把 `*(core+0x00)`（这张表）改成**未映射地址**`0xdead00000`  
（只改一个数据字段，零代码改写），进程立刻 SIGSEGV：

```
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xdead00054
 x9 0000000dead00000 ; 正是写进去的非法值
 pc 0000007176ba8810 → 匿名解密段+0x50810
```

**fault addr 里的 `0x54` = `10*8+4`**⇒ 崩在 `ldr w10,[表 + i*8 + 4]`（读 `len` 字段，当时 `i = 10`）。  
**core+0x00 置非法地址 ⇒ SIGSEGV：fault addr 0xdead00054、pc 落在匿名解密段 +0x50810**  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_ENFVZFSGE7B5FMJ.webp)

读它的那段代码就在匿名解密段 `+0x507f8..+0x50864`。

**(c) 于是三项被完全解释**：`len[i]` 改成 1 ⇒ 第 `i` 段只覆盖第一项，**段内其余项被漏掉**（不注册）。  
锚定到索引的方法名套进来：

<table><thead><tr><th>组</th><th>改前 → 改后</th><th>被漏掉的方法</th></tr></thead><tbody><tr><td><strong>组 14</strong><code>[43,45)</code></td><td>2 → 1</td><td><strong>#44 <code>onDestroy()V</code></strong></td></tr><tr><td><strong>组 65</strong><code>[252,260)</code></td><td>8 → 1</td><td>#253 <code>getSwipeBackLayout</code>、#254 <code>onKeyDown</code>、<strong>#255~258 <code>onRestart / onResume / onStart / onStop</code></strong>、#259 <code>scrollToFinishActivity</code></td></tr><tr><td><strong>组 83</strong><code>[399,404)</code></td><td>5 → 1</td><td><strong>#400 <code>onCreate</code>、#401 <code>onPause</code>、#402 <code>onStart</code>、#403 <code>onStop</code></strong></td></tr><tr><td>组 94 <code>[475,528)</code></td><td>53 → 1</td><td>漏 52 项，但<strong>已锚定的 62 个方法名一个都不在其中</strong>⇒ <strong>稳</strong></td></tr></tbody></table>

⇒ **这三段漏掉的恰好是 Activity 生命周期回调的 native 实现**：一漏，框架调到它时就  
`No implementation found`，覆写里走不到 `super.xxx()` ⇒ 崩。  
而组 94 漏了 52 项却**在测试路径上没人调用**⇒ 稳 —— 这也顺带解释了  
"为什么错位 52 B 的组反而稳、错位 2 B 的组反而崩"。

**(d) 再往上追一层：遍历是「按需」的，不是「启动全量」的**。遍历函数的入口是匿名解密段 `+0x507c0`  
（入参 `w2` = 段索引）；全段扫它的调用点只有两个（`+0x50b20` / `+0x50b8c`），  
都在 `+0x509d0` 的一个**循环**里。hook 入口、记录每次的 `w2` 与调用者 `lr`，两次独立运行**逐条一致**：

```
idx = 0xa 0x55 0x56 0x41 0xe 0x47 0x53 0x25 0xd 0x2a 0x2c （只有 11 段，不是 98）
lr = 匿名解密段+0x50b90（调用点 B 的下一条，11/11 完全相同）
```

这 11 段合计覆盖 **31 / 553** 项，而且清一色是 **Activity / Fragment 生命周期回调 +  
`dispatchTouchEvent` / `onKeyDown` + `SQLiteOpenHelper`**（`attachBaseContext`、`onCreate`、`onStart`、  
`onResume`、`onPause`、`onStop`、`onDestroy`、`onKeyDown`、`onConfigurationChanged`、`onDestroyView`…）。

⇒ **三条互相独立的结果对上了**：

*   `{14, 65, 83}` **都在这 11 段里**（`14→✓ 65→✓ 83→✓`），而改它却**稳**的那组所在段 **94 不在列表里**；
*   这与 "`RegisterNatives` 只锚上 **62 / 553 = 11.2%**" 是**同一个画像**——  
    壳只处理**走到过的方法**：**遍历是按需的，不是启动时全量注册**。

> 途中一个坑：定位那个函数时**指纹法误匹配**到匿名解密段 `+0x19a9c`（同形序言在这个段里很常见），  
> 改用 " **段基址 + 固定偏移 + 首字断言** "才准 —— 在" 函数序言高度同形 " 的段里，指纹法不可靠。  
> 为什么只观测到 B 也算清了 —— **A 是死分支**。  
> A 之前那段混淆算术是**不透明谓词**（`r = (w2−0x30) mod 64` → 5 连乘 → 魔数乘 → 旋转 → 与  
> `0x19999998` 比较）：枚举 `r` 的全部 64 种取值、以及 `w2 ∈ [0, 0x100000)` 的 **100 万个值**，  
> **没有一个能走到 A**（`w8` 最大只到 `0x13B52B0C`，远小于阈值）；  
> 而 B 之前的 "循环计数" 恒为 **0**（三个连续整数之积必被 6 整除）⇒ **"每段恰好遍历一次" 是必然，不是采样**。  
> （与 handler 混着 OLLVM 风格混淆链是同一路数。）  
> 边界：`+0x509d0` 自己是被间接调用的，再往上没追到。

> 结论：`{14,65,83}` 的 "关键" **不是这张表的某种神秘属性**，而是**这三段的尾部恰好是生涯周期回调**。  
> 证据：本节的崩溃栈与 tombstone 现场；做法：  
> 一个注入体把 `core+0x00` 置成非法值（只写数据、不改代码），让读它的代码自己崩出来。  
> 边界：`len` 改 0 的情形未单独验证；匿名解密段 `+0x507f8` 那个函数的**调用者**没往上追。

#### 3.4.2 531 个 handler 的**语义族谱**

VM 的状态位置是已知的  
（IP = `STACK[0x820]`、虚拟寄存器 = `STACK[0x81c]/[0x82c]`、描述符槽位 = `[x8,#0x20]` …），  
所以可以 ** 按 "碰了哪些位置 + 怎么收尾"** 给 531 个 handler 分类（每个 handler 取反汇编基本块）：

<table><thead><tr><th>族</th><th>语义</th><th>handler 数</th><th>占表项</th></tr></thead><tbody><tr><td><strong>F1</strong></td><td><strong>IP 取指 / 推进</strong>（读 <code>STACK[0x820]</code>，其中 71 个还做 <code>IP += 1</code>，尾调用）</td><td><strong>279</strong></td><td><strong>63.0%</strong></td></tr><tr><td><strong>F2</strong></td><td><strong>虚拟寄存器</strong>：读 / 算（含写回栈槽后尾调用）</td><td>133</td><td>15.3%</td></tr><tr><td>F0</td><td>其它（短桩 / 转发 / 未分类）</td><td>42</td><td>11.0%</td></tr><tr><td>F6</td><td>子函数调用（<code>bl</code>/<code>blr</code> 辅助例程）</td><td>21</td><td>3.3%</td></tr><tr><td>F7</td><td>直接跳转（段内 / 另一个 handler）</td><td>15</td><td>2.1%</td></tr><tr><td>F3</td><td>原子 / 标志检查（<code>ldar</code>/<code>stlr</code> 等，短基本块）</td><td>17</td><td>1.9%</td></tr><tr><td>F4</td><td>段内绑定写入（<code>adrp</code> 出地址 → <code>str</code> 进全局结构 → <code>ret</code>）</td><td>16</td><td>1.8%</td></tr><tr><td>F5</td><td>栈上暂存 + 算术（操作数准备）</td><td>8</td><td>1.6%</td></tr></tbody></table>

⇒ **头部 4 族覆盖 89.3% 的 handler / 95.1% 的表项**；"531 个逐个定性" 因此可以压成  
**"取指推进 + 虚拟寄存器算术 + 原子检查 + 绑定写入" 四件套**。  
另外两点实测结论：

*   **`blr` 面已归类**：106/531 个 handler 含 `blr`，目标**绝大多数是结构体槽位载入**  
    （`ldr xN,[xM,#imm]` 紧接着 `blr xN`）——典型如 `[x8,#0x20]`、`[x8,#0x88]`，以及 **IP 槽 `[sp,#0x820]`**（25 处）  
    ⇒ 它是**尾调用 / 槽位调用**，不是 JNI 反射；真正的 JNI 面仍在分派层（`JNIEnv` 槽位表）。
*   **"结束条件" 的实测答案是否定的**：整段里 **没有任何 handler 比较 `0x229`（= 方法表项总数 553）**，  
    也没有别的显式哨兵；收尾形态是 `ret` 44 个 / `br xR` 间接尾调用 161 个 / 条件分支 28 个  
    ⇒ 壳 B 的收尾是**尾调用链自然回到宿主**，与壳 A 的 `opcode 15` **显式哨兵校验**不是同一类设计。

#### 3.4.3 第一步：**IP 步长**与长度表的交叉

F1 族（"IP 取指 / 推进"）是最值钱的一族，那它到底**怎么推进 IP**？如果 VM 是变长编码，  
步长就应当是**那条指令的长度**—— 于是有一个可离线验证的交叉断言：  
_" 把每个 handler 里『读 IP 之后的 `add/sub #k`』抽出来，k 的分布应与长度表同形 "_。

实测（531 个 handler）：**读 IP 槽 `[sp,#0x820]` 的正好 279 个**—— 访问模式算出的  
**F1 族 279 个完全一致**（两条互相独立的口径给出同一个数）；其中

<table><thead><tr><th>项</th><th>值</th></tr></thead><tbody><tr><td>可抽出<strong>常数</strong>步长 <code>add/sub #k</code></td><td><strong>99</strong> 个（<code>add</code> 92 / <code>sub</code> 7）</td></tr><tr><td>步长来自<strong>寄存器</strong>（动态长度）</td><td><strong>10</strong> 个（<code>x9</code>×9、<code>x10</code>×1）</td></tr><tr><td>读 IP 但步长不在同一寄存器里</td><td>170 个</td></tr></tbody></table>

<table><thead><tr><th>步长 k</th><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th><th>8</th><th>9</th></tr></thead><tbody><tr><td>handler 数</td><td><strong>74</strong></td><td>1</td><td><strong>15</strong></td><td>1</td><td>4</td><td>3</td><td>1</td></tr></tbody></table>

⇒ 与长度表的长度谱（`1B×17 > 2B×24 > 3B×12 > 4B×6 > 5B×7 …`，取值 1~53）**同形**（都以小值为头部，  
且 handler 侧的步长值是长度表长度值的**子集**：{1,2,3,4,5,8,9} ⊂ {1..10,13,15,20,21,25,32,33,53}）。  
另外发现 **10 个 handler 的步长是寄存器**（`x9`）⇒ 与 "变长编码" 自洽：  
长度既可以是立即数，也可以是**指令流里的字段**。

> 运行时证据显示：那张表其实是  
> **「553 项方法表」的分段索引**（`len[i]` = 第 `i` 段含多少个 24 B 方法项），**不是 "变长指令的长度"**。  
> 因此 "IP 步长分布与长度谱同形" 这条交叉断言**降级为巧合性观察**，不能再当作语义证据；  
> F1 族 "推进 IP" 这个事实（279 个 handler 读 `STACK[0x820]`）本身不受影响，  
> 但 "步长 = 指令长度" 这个**对应关系作废**。
> 
> 边界：这是**静态语义标注 + 一次交叉对比**，**不是**逐条 handler 的干预式验证；  
> 要把 F1/F2 两族的运算语义都做到 ③ 档，需要给 handler **内部的输入槽**造变体并做真机对拍（成本高，未做）；  
> 与它相邻的那一跳（thunk → dispatch 的索引常量）**已经**用同一手法验证通过。  
> （纯离线：把每个 handler 的 IP 步长与边界表对照。）

#### 3.4.4 第二步：**handler 的运算语义**逐类还原

" 每类在**动哪些状态** "，把 F1 族的"IP 步长 " 落到了数。  
"531 个 handler 逐个还原**运算** " 一直被当成**无底洞**（handler 里混着 OLLVM 风格的位运算混淆链）。  
本轮换问法：**不逐条猜，而是先把 "同形状" 的 handler 合并，再用解释器把混淆链跑掉、只看净值。**

做法（纯离线，用一个 AArch64 子集解释器跑具体值 + 扰动差分）：

1.  **形状去重**：把每个 handler 的反汇编基本块（反汇编到第一条控制流转移为止）的**指令序列**当作形状；
2.  **具体值解释**：对每个形状的代表元跑一个 AArch64 子集解释器（只支持 " 寄存器算术 + `sp` 相对访存 "），  
    输入 = 被读的 `[sp,#off]` 槽 + 被读的传入寄存器，输出 = 被写的栈槽 / `x0..x17`；
3.  **扰动差分**：逐个把输入置 `0xA5`、再对单变量枚举 `0~255`，判定依赖，并把结果匹配成简洁表达式。

三条结果：

**(a) 531 个 handler 只有 168 种形状，而且头部极其集中**：

<table><thead><tr><th>#</th><th>形状（指令序列）</th><th>handler 数</th><th>表项引用</th><th>占表项</th></tr></thead><tbody><tr><td>1</td><td><code>mov strb strb ldr ldrb ldrb add b</code></td><td>95</td><td>190</td><td><strong>21.8%</strong></td></tr><tr><td>2</td><td><code>ldrb ldrb mov ldrb mov mov sub sub and and orr eor eor mov b</code></td><td>134</td><td>134</td><td><strong>15.4%</strong></td></tr><tr><td>3</td><td><code>str strb strb ldrb ldrb ldr strb strb str ldrb ldr ldr br</code></td><td>1</td><td><strong>134</strong></td><td><strong>15.4%</strong></td></tr><tr><td>4</td><td><code>mov strb strb ldr ldrb ldrb b</code></td><td>37</td><td>74</td><td>8.5%</td></tr><tr><td>5/6/12</td><td><code>bti mov ret</code> / <code>bti ret</code> / <code>bti mov mov ret</code></td><td>6</td><td>59</td><td>6.8%</td></tr><tr><td>7/8</td><td><code>adrp add ldar cbz</code> / <code>… cbnz</code></td><td>26</td><td>26</td><td>3.0%</td></tr><tr><td>…</td><td>其余 156 种</td><td>207</td><td>225</td><td>25.8%</td></tr></tbody></table>

⇒ **头部 4 种形状就覆盖 61% 的 handler / 66% 的表项**。

**(b) 同形状的 handler 是 "同一模板 + 不同参数"**：形状 1 的 95 个 handler 是**同一段指令、  
只有栈槽偏移不同**；形状 3 更极端 —— **1 个 handler 被 134 个表项共用**。  
所以 "531 个各不相同的 handler" 更准确的说法是**少数模板 × 操作数槽**。

**(c) 解释器把混淆链跑掉后露出净值**。最有代表性的是形状 2（134 个 handler，全表最长混淆链）：

```
ldrb w8,[sp,#0x6c] ; 输入 x
mov w10,#0x68
ldrb w13,[sp,#0x81c] ; 输入 v（另一个槽）
mov w11,#0x5f
mov w12,#-0x60
sub w8,w8,#0x69 ; ┐
sub w9,w10,w9 ; │ 四条位运算
and w9,w9,w11 ; │ 混淆链
and w8,w8,w12 ; │
orr w8,w9,w8 ; ┘
eor w8,w13,w8
eor w8,w8,w11
```

解释器逐条执行 + 单变量枚举 256 次之后，输出只剩一行：

```
x8 = (mem[0x6c] + 0xffffff97 (mod 2^32)) XOR mem[0x81c]
x9 = 同 x8
```

即 **`dst = (byte[slot] − 0x69) ^ byte[0x81c]`**—— `sub/and/orr/eor` 那 7 条**全部抵消**  
（对 `0~255` 全枚举验证）。同形状的 134 个 handler 里，只有 `slot` 这个 "操作数槽" 在变。

同一手法还给出另两类：

*   形状 1/4（**IP 推进**模板）：`x10 = mem[0x820] + 1`（即 `IP += 1`），并把常量 `0x3b` 写回一个栈槽；
*   形状 3（**分派核心**，134 表项共用）：结构标注为  
    `读 IP → 写虚拟寄存器槽 0x81c/0x82c → 查表 next = x19[x8 << 3] → br` ——  
    正是 threaded-dispatch 的 "取指→派发" 一跳。

**形状 3 的静态现场：`0x74767CA8AC` —— 134 项表项共用的分派尾（IDA 反汇编）**  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_QGVGUHFBFZHDKTV.webp)

> 边界：
> 
> 1.  还原是**基本块级**的（跨 handler 的虚拟寄存器堆布局未整体建模）；表达式匹配只覆盖  
>     `in+c` / `in^c` / `~in` / `in` 与两输入 `f(x) XOR y` 五种形式，其余渲染为 `table(...)`/「混合输入」；
> 2.  **后来补上了 "非栈访存" 与 "浮点" 两类建模 ⇒ 覆盖率 58.1% → 99.9%。**  
>     原来一遇到 "非 sp/x29 基址" 就主动止步（报 `indirect mem via x8` 等）；这一轮改成**符号化建模**：
> 
> *   `ldar`/`stlr` 等 acquire/release 访存按 `ldr`/`str` 处理（ARMv8 的内存序语义在单线程模拟下不影响取值）；
> *   未知基址、`[xB, xI]` 查表、`adrp+add` 得来的绝对地址（全局变量）一律**符号化成 "输入"**，  
>     从而直接进既有的 "单变量枚举 256 次 + 差分" 机制；
> *   条件码判定从 `cset` 的操作数改成**文本解析 + 完整 N/Z/C/V 跟踪**（原来只跟踪 Z），  
>     并补符号扩展族（`sxtb/sxth/sxtw/uxtb/uxth/uxtw`）、`mrs`（系统寄存器，符号化）、`udiv/sdiv`、`stp/ldp/stnp/ldnp`。
> *   **又加了浮点寄存器模型**：`scvtf`/`ucvtf`/`fcvtzs`/`fcvtzu`/`fcvt`/`fadd`/`fsub`/`fmul`/`fdiv`/`fcmp`/`fmov`  
>     以及 `ldursb`/`ldursh`/`ldursw` ⇒ 原来那 31 个 "含浮点" 的形状全部跑通；  
>     **⇒ 167/168 种形状（871/872 表项 = 99.9%）已还原**（旧为 24 种形状 / 507 表项 = 58.1%）；  
>     **仅剩 1 个形状（1 个表项）= `sshll`（SIMD 向量指令）**，如实标注；
> 
> 3.  三个 "原本未还原" 的形状现在解出了语义，且与本节模板结论一致：
> 
> *   **形状 3（1 个 handler / 134 表项 = 分派核心）**：`x8 = mem[0x10]` → `x9 = mem[x19 + x8]`  
>     ⇒ **"读操作数 → 查跳转表取下一个 handler"**（就是 threaded-dispatch 的 "取指→派发" 一跳）；
> *   **形状 10 / 11（各 10 表项）**：`mem[0x39c] = mem[x9]`、`x8 = mem[0x81c]`、`x9 = mem[x24 + x8]` ⇒ 同类 "查表 + 写虚拟寄存器"；
> *   **形状 7 / 8（16 / 10 表项）**：`x16 = 0x7476826580`（adrp+add 的绝对地址）→ `x8 = mem[0x7476826580]`  
>     ⇒ **"读某个全局标志位，然后条件分支"**（原来只能标 "结构语义"）；
> *   **无回归**：形状 1（95 handler / 190 表项，IP 推进）的净表达式与旧报告逐字一致（`x10 = mem[0x820] + 1`、`mem[0x53c] = 0x3b`）；
> 
> 4.  这是**静态语义**；**③ 档的干预式验证已经做了**（「③ 档判据升级」）：给 thunk 的内嵌索引常量 "造变体"（改一条 `movk`），  
>     真机实测该表项的 `x1` **逐位变成预测值**（2/2 命中、其余表项一格不动）—— 但那是 handler **前面**那一跳；  
>     handler **内部**运算语义的干预（改输入槽、预测输出）仍未做。
> 
> 没做的原因：三个条件要同时成立 ——  
> ① **靶子要在采集窗口内被执行**：handler 是**按需**触发的（98 段实测只走到 11 段、覆盖 31/553 项），  
> 上一段那次 thunk 常量干预就出现过 "三次冷启动都没捕到表项 43"，最后改用一个 "每次运行都出现" 的靶子；  
> ② **要能定位该 handler 的共享栈帧与它的输入槽偏移**：handler 之间是 `br` 尾调用（不建新帧）、帧还是 JIT 那一层，  
> 而槽偏移随形状变（形状 2 用 `[sp,#0x6c]` + `[sp,#0x81c]`）；  
> ③ **还要在 `br` 前后各取一次值**：尾调用没有返回点，只能在 "下一个 handler 入口" 读寄存器。  
> 而满足这三条的代价是**放弃 "零代码改写"**—— 要写运行期栈槽 / 寄存器，而壳带完整性校验（改边界表即自检失败）；  
> 再加上 handler 地址不是 16 B 对齐、相邻间距最小 **4 B**，入口只能靠 "段基址 + 固定偏移 + 首字断言" 定位。  
> 离线一侧已给出 **167/168 种形状**的净值表达式，且可被反汇编独立复核 ⇒ 这一格留在 "下一层"。

**工具：把 531 个 handler 压成 168 种形状的四个关键函数**——  
下面代码里的 `sim(ins, 输入)` 是那个 **AArch64 子集解释器的入口**（寄存器 / 栈槽 / 符号访存 / 浮点四类状态，  
`step()` 逐指令推进）；完整实现 **784 行**，这里只摘 "机制" 部分：`shape()` / `pattern_tags()` 做形状指纹与结构语义标注，  
`discover()` / `match_expr()` 做具体值模拟 + 扰动差分 → 真值表归约。  
缺了它，本节所有净值就只能靠 "我说它抵消了"。

```
# ① 形状去重：把反汇编归一化成"骨架"，同骨架 ⇒ 同一模板（531 个 handler ⇒ 168 种形状）
import re

def shape(ins):
    out = []
    for i in ins:
        ops = re.sub(r"\b[wx]\d+\b", "R", i.op_str)                       # 寄存器归一
        ops = re.sub(r"#-?0x[0-9a-f]+|#\d+", "#I", ops)                   # 立即数归一
        ops = re.sub(r"\b(eq|ne|cs|cc|hi|ls|ge|lt|gt|le)\b", "CC", ops)   # 条件码归一
        out.append(i.mnemonic + " " + ops.rstrip(","))
    return " ".join(out)

def pattern_tags(ins):
    js = " ".join("%s %s" % (i.mnemonic, i.op_str) for i in ins)
    tags = []
    if "[sp, #0x820]" in js:
        tags.append("读 IP 槽 mem[0x820]")
    for m in re.finditer(r"add\s+x(\d+),\s*x\1,\s*#(\d+)", js):           # IP 推进
        if re.search(r"ldr\s+x%s,\s*[sp, #0x820]" % m.group(1), js):
            tags.append("IP += %s" % m.group(2))
    vs = sorted(set(re.findall(r"strb?\s+w\d+,\s*[sp, #(0x8[0-9a-f]c)]", js)))
    if vs:
        tags.append("写虚拟寄存器槽 " + ",".join(vs))
    m = re.search(r"ldr\s+x(\d+),\s*[x(\d+),\s*x(\d+),\s*lsl #(\d+)]", js)
    if m:
        tags.append("查表 next = x%s[x%s << %s]" % m.group(2, 3, 4))
    tags.append({"br": "尾调用（间接 br）", "b": "尾调用（直接 b）",
                 "ret": "返回 ret"}.get(ins[-1].mnemonic, "子调用 " + ins[-1].mnemonic))
    return tags
```

```
# ② 语义还原：具体值模拟 + 扰动差分（那一层 and/orr/eor 混淆链在这一步被自动抵消）
def discover(ins, sim):
    base = sim(ins, {})
    dep = {}
    for inp in base.inputs:                   # ① 逐个扰动：置 0xA5，看谁跟着变
        s1 = sim(ins, {inp: 0xA5})
        for key in base.outputs:
            if base.get(key) != s1.get(key):
                dep.setdefault(key, set()).add(inp)
    out = {}
    for key, d in dep.items():
        if len(d) == 1:                       # ② 单输入 ⇒ 枚举 256 次，拿真值表
            inp = next(iter(d))
            out[key] = [sim(ins, {inp: v}).get(key) for v in range(256)]
        else:                                 # ③ 多输入 ⇒ 先做 XOR 可分离性检验
            out[key] = ("multi", sorted(d))
    return out

def match_expr(tbl):
    b = tbl[0]
    if all(((v + b) & 0xFFFFFFFF) == t for v, t in zip(range(256), tbl)):
        return "in + %#x (mod 2^32)" % b
    if all(((v ^ b) & 0xFFFFFFFF) == t for v, t in zip(range(256), tbl)):
        return "in ^ %#x" % b
    if all(((~v) & 0xFFFFFFFF) == t for v, t in zip(range(256), tbl)):
        return "~in"
    return None
```

> 真跑一次的输出（`--json handler_tables.json --anon <解密段.bin>`）：`168 shapes, 12 rendered, 0 unknown`；  
> 头部 4 种形状覆盖 **61% 的 handler / 66% 的表项**。形状 2（134 handler / 134 表项）的代表元  
> `0x74767ca854` 反汇编 15 条，解释器给出的净值是  
> `x8 = (in + 0xffffff97 (mod 2^32)) XOR mem[0x81c]`，即 **(mem[0x6c] − 0x69) ^ mem[0x81c]**——  
> 中间那 5 条 `sub/and/orr/eor` 混淆链**被自动抵消**，这就是本节 “静态语义” 的来历；  
> 形状 3（1 个 handler / 134 表项）解出 `x8 = mem[0x10]` → `x9 = mem[x19 + x8]` → `br x9`，  
> 也就是 threaded-dispatch 的 “取指 → 查表 → 派发” 那一跳。

分派链要闭合，需要四跳。**已把第 3 跳在运行时取到**：

<table><thead><tr><th>跳</th><th>载体</th><th>状态</th></tr></thead><tbody><tr><td><code>opcode → 内部编号</code></td><td><code>libexecmain</code> 的 <code>getOpCode</code></td><td>✅ <strong>已还原（256 项双射表）</strong></td></tr><tr><td><code>内部编号 → 描述符索引</code></td><td>553 项函数指针表 + 553 个跳板</td><td>✅ <strong>已实测验证</strong>（553/553 指向 <code>r-xp</code>；间距 0xB8 为主）</td></tr><tr><td><code>索引 → handler</code></td><td>描述符 <code>+0x20</code>（<strong>运行时填入</strong>；<code>+0x18</code> 是另一个跳板，实测<strong>未被分派链使用</strong>）</td><td>✅ <strong>本次已取到</strong></td></tr><tr><td>handler → 语义</td><td>531 个 handler 逐个定性</td><td>✅ <strong>已归类</strong>：按访问模式分 8 族，头部 4 族覆盖 89.3% handler / 95.1% 表项</td></tr></tbody></table>

**运行时实测**（描述符就是构造函数 `malloc(0x40)` 出来的那 0x40 字节）：

<table><thead><tr><th>偏移</th><th>运行时值</th><th>解析</th></tr></thead><tbody><tr><td><code>+0x00</code></td><td><code>libexecmain+0x29150</code></td><td><code>&amp;unk_29150</code> ✓</td></tr><tr><td><code>+0x08</code></td><td><code>libexecmain+0x28008</code></td><td>553 项表 ✓</td></tr><tr><td><code>+0x10</code></td><td><code>libexecmain+0x3A3C</code></td><td><strong>正是 <code>getOpCode</code></strong>（与静态记录的 <code>0x3a3c</code> 逐位吻合）✓</td></tr><tr><td><code>+0x18</code></td><td><strong>匿名解密段 +0x504A4</strong></td><td>磁盘上是 0；运行时是<strong>一个跳板函数</strong></td></tr><tr><td><code>+0x20</code></td><td><strong>匿名解密段 +0x504F4</strong></td><td>同上（与 <code>+0x18</code> 相隔 0x50）</td></tr></tbody></table>

⇒ 与静态结论**完全对上**：`+0x18`/`+0x20` 在磁盘上确实是 0，  
**由 `libexec.so` 在运行时填成 "解密段里的两个跳板函数"**。

两个跳板**几乎逐条相同**：`ldp q1,q0,[x3]` → `stp q1,q0,[sp]` → `bl #…124C` ——  
即把第 3 个参数指向的 32 字节搬到栈上，再**转发给同一个 dispatch 主体**（段内 `+0x5224C`）。  
dispatch 主体里就是描述的那个**同构分派模板**：

```
and w20, w1, #0xffffff ; 取索引（低 24 位）
umaddl x1, w20, #0x18, x8 ; 索引 × 24 → 某项结构体
bl #0x74767A53E8
adrp x20,#0x7476813000 ; add x20,x20,#0xED0 ; ★ 表基址 = rw-p + 0xAED0
cinc w8, w9(=2), ne
ldr x8, [x20, w8, uxtw #3] ; ★ 查表
mov w9,#0x51A3 ; movk w9,#0x430E,lsl#16 ; 常量 0x430E51A3
add x8, x8, x9 ; ★ 表项 + 常量 = 目标
br x8 ; ★ 分派
```

而那张表（`libexec.so` 的 **`rw-p` 段 +0xAED0**）里的值，**静态看是一串无意义的数**：

<table><thead><tr><th>表项</th><th><code>+ 0x430E51A3</code></th><th>落点</th></tr></thead><tbody><tr><td><code>0x74336BC13D</code></td><td><code>0x74767A12E0</code></td><td>匿名解密段 <strong>+0x522E0</strong>（正是 dispatch 主体里 <code>ldr x8,[x19,#0x10]</code> 那条）</td></tr><tr><td><code>0x74336BC165</code></td><td><code>0x74767A1308</code></td><td>+0x52308</td></tr><tr><td><code>0x74336BC22D</code></td><td><code>0x74767A13D0</code></td><td>+0x523D0</td></tr></tbody></table>

⇒ **表项 = 目标地址 − 常量 `0x430E51A3`**。这就是 "静态接不上" 的直接原因：  
**它不是 "值为 0 的表"，而是一张被常量混淆过的表**—— 不知道那个常量，就还原不出目标。

> **一条跨运行稳定性结论**：本次运行时 dump 的匿名段（712,704 B）与既有  
> 两次运行 dump 出的解密段**前 581,632 字节逐字节一致**⇒ 解密段内容跨运行稳定（ASLR 只改地址、  
> 不改内容），因此可以把**运行时地址换算成段内偏移**，在已有的离线 dump 上做静态分析。

**再往下追一层**：dispatch 主体调用的 `0x74767A53E8`（段内 `+0x563E8`）已反汇编 —— 它做的是  
**为一段 VM 程序分配 "解码表"**：

```
malloc(0x80) ; VM 上下文项（+0x68 处保存了「描述符 +0x20」的值）
ldrh w8, [x24] → N ; 从 16 位头读"数量"
malloc(N*8) → +0x08 ; 8 字节/项的表
malloc(N*4) → +0x10 ; 4 字节/项的表
ldr w8, [x8(=+0x10 表), w23, sxtw #2] ; 按索引查 u32 表
```

⇒ 到这里，" **索引 → 结构体 → 解码表 → 分派** " 的骨架已经贯通。

** 最后一块拼图：`core+0x08` 那张 553 项表。** 它在 **libexecmain 的 `rw-p` 段偏移 8**，实测：

<table><thead><tr><th>检查项</th><th>结果</th></tr></thead><tbody><tr><td>表项数</td><td><strong>553</strong></td></tr><tr><td>落点</td><td><strong>553/553 全部指向 libexecmain 的 <code>r-xp</code> 段</strong></td></tr><tr><td>间距</td><td><strong>0xB8 = 184 B 为主（545/552）、0xB4 = 180 B（7/552）</strong>（用全表实测修正旧记 "恒 180 B"）</td></tr><tr><td>表项 [0]</td><td><code>r-xp + 0x4AD0</code> —— 正是举例的 <code>sub_4AD0</code></td></tr></tbody></table>

thunk 前半是**变参序言**（保存 `x0–x7`/`q0–q7` 到栈 + 栈 canary，就是 `va_start` 的实现），  
**尾部唯一差异就是传给 dispatch 的索引**：

```
; thunk[i]（表项[i] 指向；0x4AD0 起，间距 0xB4 / 0xB8 两种）
adrp x10,#0x27000 ; ldr x10,[x10,#0xff8] ; ldr x10,[x10] ; → core（描述符）
ldr x9, [x10, #0x20] ; x9 = *(core+0x20) = dispatch（运行时填入）
mov w1, #i ; ★ 只有这条随 i 变（thunk[0]: mov w1,wzr；thunk[1]: mov w1,#1）
blr x9 ; dispatch(a1, i, a2, va)
```

⇒ **五跳链条**（**注意定性**：其中**第 2 跳是静态推断，且已被本节的干预实验证伪**；第 3 跳由本次实验直接证实 "确实被调用"；其余各跳仍是**静态推断 + 一致性证据**）：

```
Dalvik opcode
 → getOpCode(opcode) = 内部编号 i [libexecmain +0x3A3C，256 项置换表]
 → 553 项表[i] = thunk_i [libexecmain rw-p +0x8，间距 0xB8 为主] ⚠️ 静态推断；**干预改它无效**
 → thunk_i: ldr x9,[core+0x20]; mov w1,#i; blr x9 ✅ 实测：tramp 挂 `core+0x20` 后能记录到 x1=thunk 内嵌常量
 → core+0x20 = 跳板 [libexec.so 匿名段 +0x504F4]
 → 跳板转发 → dispatch 主体 [匿名段 +0x5224C]
 → dispatch 查表（rw-p +0xAED0，表项 = 目标 − 0x430E51A3）→ br → handler 分支
```

⏳→✅ **"运行时走哪条" 这一格也已补上（② 档）**：不追 `br x8` 的 handler 目标，而是**记录分派索引本身**——  
把描述符里的一个**数据指针**`*(core+0x20)` 换成一段汇编 trampoline（**不动任何指令、不做 mprotect**），  
trampoline 记下 `x1 / lr / x2 / x8` 后 `br` 回原跳板，语义等价。真机（Pixel XL，Android 10）跑出 **30 条**记录：

<table><thead><tr><th>实测（真机注入 trace）</th><th>静态模型（表项 → thunk → 内嵌 w1 常量）</th><th>判定</th></tr></thead><tbody><tr><td><code>lr_off=0x6a38</code>，<code>x1=0x100002b</code></td><td>表项 **[43]**→ thunk <code>0x69A4</code>（<code>mov w1,#0x2b; movk w1,#0x100,lsl#16</code>）</td><td><strong>MATCH</strong></td></tr><tr><td><code>lr_off=0x16a14</code>，<code>x1=0x200018f</code>（×12 次）</td><td>表项 **[399]**→ thunk <code>0x16980</code></td><td><strong>MATCH</strong></td></tr><tr><td><code>lr_off=0xbb70</code>，<code>x1=0x100009c</code></td><td>表项 **[156]**→ thunk <code>0xbadc</code></td><td><strong>MATCH</strong></td></tr><tr><td><code>lr_off=0x10070</code>，<code>x1=0x10000fc</code>（×14 次）</td><td>表项 **[252]**→ thunk <code>0xFFDC</code></td><td><strong>MATCH</strong></td></tr></tbody></table>

⇒ **30/30 逐位一致**，"模型预测 == 真机记录" 成立，壳 B 的验证强度由此进入 **② 档**。  
两处副产物值得单独记：

1.  **553 个 thunk 的 "索引常量" 互不相同**（0、1、…、4、`0x1000000|i`、`0x2000000|i` …）——  
    所以一条 `x1` 就能唯一定位表项下标，这是本条证据成立的前提；
2.  **修正一处旧口径**：表项间距**不是**恒定的 `0xB4`(180 B)，实测为 **`0xB8`(184 B) × 545 + `0xB4` × 7**  
    —— 因为带 `movk` 的 thunk 比纯 `mov` 的长 4 字节。

**③ 档干预实验：改了那张 "分派表"，行为却没变 —— 预测未命中**

前两档都是 "看起来一致"，这一档用**干预**去戳它：把 `core+0x08` 指向的 **553 项表**的某一项改成**另一项的值**  
（只写 8 字节数据）：`表项[399] ← 表项[43]`（即让 "索引 399" 指向 `thunk_43`）。

<table><thead><tr><th>环节</th><th>内容</th></tr></thead><tbody><tr><td>预测（事先写下）</td><td>若该表真的被 dispatch 读取，则 "索引 399" 的分派应改走 <code>thunk_43</code> ⇒ trace 的 <code>x1</code> 应从 **<code>0x200018f</code>** 变成 <strong><code>0x100002b</code></strong>（43 的常量），<code>lr</code> 落到 thunk_43 区间</td></tr><tr><td>实测</td><td><strong>19/19 条仍是 <code>x1=0x200018f</code>、<code>lr</code> 仍落在 thunk_399 区间</strong>—— 预测<strong>未命中</strong></td></tr><tr><td>旁证</td><td>注入后读 <code>/proc/&lt;pid&gt;/mem</code> 复核：<code>表项[399] = 0x78f3bc99a4</code>（= thunk_43）<strong>写入仍在、未被覆盖</strong></td></tr></tbody></table>

⇒ **结论（如实记录）**：**"553 项表 [i] = thunk_i" 这一跳在运行时并未被 dispatch 读取**——  
它 "看起来完全像分派表"（553/553 落 `r-xp`、间距规整），  
但**改它并不能改变行为**。` 机制只做到 "记录现象"，机制留了三种可能待验  
（① 初始化时被消费；② 运行期另有一份副本；③ thunk 地址另有来源）。

> **机制**：答案是 **①+② 的组合**——  
> 那张 553 项表是**注册期** "编号 → thunk 地址" 的来源，壳在注册时把地址写进 **`ArtMethod->data_`**；  
> 运行期分派读的是 **`ArtMethod->data_`（也就是那份 "副本"）**，不再读表。三条独立证据：  
> ① `ArtMethod->data_ = thunk_252` 的运行时地址（实测：`+0x18 = 0x78f3b99fdc`）；

> ② 直接改 `ArtMethod->data_` ⇒ **分派一次即崩**、对照存活（③ 档，本节末）；  
> ③ 206 个 native 方法的 `fnPtr` 与 553 项表**直接对位 62 项**⇒ 表项 == 注册进 `data_` 的值。  
> ——所以 "第 2 跳降级为静态推断" 要准确理解：它**不是错的**，只是**只在注册期成立**。
> 
> **`把上面这套机制从"事后解释"升级为"事前预测并命中"。** 改表是**注入到"已完全启动"的进程**（复现步骤原话「需先完全启动再 attach」）⇒ 表早被消费掉了， 所以"改它无效"并不奇怪。新一轮改用**注入体 + 启动即注入**（pid 一出现立刻注入）， 并内置**时机判据**：同时抢装` RegisterNatives`hook —— 只要记到注册调用，就证明我们确实在注册之前动手。 实测 **改表`t=+41ms`、首次注册` t=+727ms`**（差 686 ms）✓。
> 
> <table><thead><tr><th>装置（针对表项 [43]）</th><th><code>[DISP]</code> 分派记录</th></tr></thead><tbody><tr><td><strong>注册前毒化</strong>（表项 [43] ← <code>0xDEADBEEFDEADBEEF</code>）</td><td><strong>0 / 0 / 0</strong>（3 轮）</td></tr><tr><td><strong>对照</strong>（不改表）</td><td><strong>1 / 1 / 1</strong>（本轮 3 轮）＋ <strong>1 × 6</strong>（前一轮 6 轮）</td></tr></tbody></table>
> 
> ⇒ 对照 **9/9** 都能记录到 "索引 43" 的分派，而**注册前把它毒化后这个分派 3/3 不再出现**——  
> 唯一自洽的解释：**壳在注册期读那张表、把值写进方法的 `ArtMethod->data_`**，运行期调用它时用的是被写坏的入口。
> 
> **⇒ 三种可能全部判定**：  
> ① 该表在**初始化（注册）期被消费**✅ **证实**（注册前改 ⇒ 行为按预测改变；注册后改 ⇒ 无效）；  
> ② 运行期另有**一份副本**✅ **证实**（副本就是 `ArtMethod->data_`）；  
> ③ thunk 地址**另有来源**❌ **证伪**（表项 == 注册进 `data_` 的值，62 项对位）。
> 
> 两条 "如实记录的坑"：**(a) **改一个 "在本窗口内不被调用的索引"（如 `399`/`252`）是**测不出差异**的  
> —— 这也解释了最早那两次 "毒化表项" 实验（索引 43 / 399）为什么看着 "毫无影响"（触发流程里观测到的分派几乎只有索引 43）；  
> **(b)**`x1` 是 **thunk 内嵌常量**，"改走另一个 thunk" 与 "本来就调到那个 thunk" 的 `(x1, lr)` **完全一样**  
> ⇒ 判据必须用**毒化**这种二值形式（或比对 `ArtMethod` 本体）。

> 这正是 ③ 档的价值：**干预能暴露静态推断的漏洞**。本节链条里标 "实测" 的第 2 跳因此**降级为 "静态推断"**。

**查到 thunk 的真实入口，并把 ③ 档做成了**

1.  **`core+0x18` 不是那条分派口**：把两个跳板都挂记录（记录里带一个 `src` 标识区分），11 条记录**全部来自 `core+0x20` 那个跳板**⇒ 假设不成立。
2.  **thunk 的调用者在 JIT code cache**：thunk 序言把 `x30` 存在 `[sp,#0x120]`，tramp 里读 `*(sp+0x128)` 即可拿到  
    "谁调用了 thunk"。实测 **`caller = 0x78ff149354`**（5/5 相同），落在  
    `/apex/com.android.runtime/lib64/libart.so` 的 **`rwxp`（JIT code cache）段**⇒  
    **thunk 是被 JIT 编译的代码调用的**，`thunk_i` 本身就是 "Java native 方法被调用" 的入口。  
    ——这也解释了为什么改那张 553 项表无效：**它不在 JIT → thunk 这条链上**。
3.  **篡改实验（③ 档，预测命中）**：在 tramp 里、转发回原跳板之前把索引写成 **`0xDEADBEEF`**。  
    事先预测：dispatch 取到错误 / 越界 handler ⇒ **App 崩溃或异常**。

<table><thead><tr><th>版本</th><th>触发前主进程 pid</th><th>触发后 pid</th><th>结果</th></tr></thead><tbody><tr><td>篡改版 #1 / #2</td><td>9999 / 11906</td><td><strong>10485 / 12359</strong></td><td><strong>崩溃 → 重启</strong>（另有 <code>IsolatedService ... has died</code>）</td></tr><tr><td>对照版 #1 / #2</td><td>11156 / 13014</td><td>11156 / 13014</td><td>存活</td></tr></tbody></table>

⇒ **篡改 2/2 崩、对照 2/2 不崩**；篡改版记录里出现 `x1=0xdeadbeef`作为旁证。  
**壳 B 的验证强度由此推到 ③ 档**（分派层子模型）。  
**再深一层：把机制钉死**

既然 `thunk_i` 是被 JIT 代码调用的，那就顺着 JIT 代码往上读：

1.  **`caller` 附近的 JIT 代码**（`libart.so` 的 `rwxp` = JIT code cache）：  
    `str x0,[sp]` → `mov x0,x19`（x19 = `art::Thread*`）→ `mov x1,x29` → **`bl 0x78ff5acd9c`**→ `blr x16`。
2.  **那个 helper（ART 的 native 入口解析）**：`ldr x20,[x1]`（x1 指向 **ArtMethod**）、`ldr w8,[x20,#4]`、  
    **`tbnz w8,#0x12`**（测 `ACC_FAST_NATIVE`）⇒ 它在 "按 ArtMethod 解析 native 实现地址"。
3.  **拿到 ArtMethod 并读它**（tramp 里 `ldr x15,[x29]; ldr x15,[x15]` = `[JIT_x29]`，JIT 把 x0 存在那儿）：

<table><thead><tr><th>偏移</th><th>值</th><th>解读</th></tr></thead><tbody><tr><td><code>+0x00</code></td><td><code>0x180001011382a418</code></td><td><code>declaring_class_</code>（低 32）+ <code>access_flags_</code>（<strong>含 <code>ACC_NATIVE</code></strong>）</td></tr><tr><td><strong><code>+0x18</code></strong></td><td><strong><code>0x78f3b99fdc</code></strong></td><td><strong><code>data_</code> = <code>thunk_252</code> 的运行时地址</strong>★</td></tr><tr><td><code>+0x20</code></td><td><code>0x78ff1492c0</code></td><td><code>entry_point_from_quick_compiled_code_</code>（JIT 代码入口）</td></tr></tbody></table>

⇒ **整条链闭合**：`Java 方法 → JIT 代码 → bl ART helper（读 ArtMethod 的 data_）→ blr → thunk_i → blr *(core+0x20) → dispatch`。  
**`thunk_i` 就是 "某个 Java native 方法的实现指针"**；而那张 553 项表 + `getOpCode` 是**注册期**用的  
⇒ **运行期不经表**，这正是 "改表无效" 的原因。

**③ 档判据升级：从 "崩不崩" 到 "这个寄存器等不等于这个数"**

前面那条 ③ 档是**二值**的（篡改索引 ⇒ 预测崩 ⇒ 实测崩）。本轮把判据做成**数值级**：

*   离线（逐 4 字节容错反汇编 `libexecmain` 的 `r-xp` 段）：553 项表第 `i` 项的 thunk 里，`+0x78` 是 `mov w1,#lo`、`+0x80` 是 `movk w1,#hi,lsl#16`，  
    这对常量就是**本次调用传下去的表项号**（表项 43 → `0x100002b`，低位正是 43）；
*   干预：把**表项 257** 的 `movk w1,#0x100` 改成 `#0x700`（**校验原编码通过才写**，拒绝盲写）；
*   **事先写下的预测**：trace 里「`lr` 落在该 thunk 内」的记录，实测 `x1` 从 `0x1000101` 变成 **`0x7000101`**；其余表项一格不变。

<table><thead><tr><th>组</th><th>落在该 thunk 的记录</th><th>实测 <code>x1</code></th></tr></thead><tbody><tr><td><strong>实验版</strong>（改一条 <code>movk</code>）</td><td><code>lr_off=+0x10404</code> ×2</td><td><strong><code>0x7000101</code></strong>✅ 命中预测</td></tr><tr><td><strong>对照版</strong>（同源码不 patch）</td><td>同一落点 ×2</td><td><code>0x1000101</code> ✅ 基线</td></tr></tbody></table>

其余 8 个被观察到的表项（256/258/399/401/156/44/301/42/158）**逐条不变**。⇒ 三点结论：  
① 执行期确实走「表项 → thunk → 内嵌常量」这条链 —— 这正是 "改 553 项表无效" 留下的疑问的答案：  
**表只在注册期被消费，thunk 才是执行期真正被调用的那一段**；  
② 内嵌常量就是本次调用的表项号（静态 `0x100<<16|257` 与真机逐位相等，改一个字节即刻改变实测值）；  
③ **只有被改动的那一格变化**—— 靶子与对照在**同一进程、同一次运行**内完成。

> 口径：本条是**代码级**干预（patch 一条指令 + `mprotect`）。  
> 选型（如实记录）：先试表项 43，三次冷启动都没捕到它的分派（只在启动早期某一步出现）⇒ 改用**每次必现**的 257；  
> 也只改**高位**—— 因为改低位有陷阱：43 的 `0x100002b` 改成 `0x100002c` 会与**表项 44** 的常量撞车，产生 "假命中"。  
> `lr_off` 两次运行完全一致（`+0x10404`），路径可复现。

**由机制直接导出的更根本干预（第三步 ③ 档）**：直接改 `ArtMethod->data_`（注入体里用一个编译开关 `-DTAMPER_DATA` 打开这条路）。

<table><thead><tr><th></th><th>注入前主进程 pid</th><th>触发后</th><th>记录条数</th></tr></thead><tbody><tr><td><strong>改 <code>data_</code></strong>（写 <code>0xDEAD0BAD</code>）</td><td>19234</td><td><strong>主进程消失</strong>（只剩重启中的 <code>IsolatedService</code>）</td><td><strong>1 条</strong>（分派一次即崩）</td></tr><tr><td>对照（不改 <code>data_</code>）</td><td>11156 / 13014 / 15466</td><td><strong>pid 均不变、App 存活</strong></td><td>17 条</td></tr></tbody></table>

⇒ 预测 "下次调用该方法即崩" **命中**；相比 "改索引"，这一版干预的是**方法绑定**本身，是分派层上更根部的干预。

**顺带读到的一条 "运行时数据链"**（属下面的 ① 档证据，不是 ② 档）：  
`libexec.so` 的 `r--p + 0x5AB0` → 本 so `rw-p + 0x1220` → …… → 堆上（`[anon:libc_malloc]`）一张  
**24 字节 / 项的结构体表**，项形如 `{ptr, ptr, u64}`。  
两点值得记：① **历史 dump 与本次运行的段内偏移一致（都是 `+0x1220`）**⇒ 该结构跨运行稳定；  
② dispatch 里 `umaddl x1, w20, #0x18, x8` 的 `x8` 正是这张表的基址 —— 也就是说  
**"索引 × 24 结构体" 这一环也在运行时读到了实际值**。

**补充（把这条链与 "VM 程序" 分开）**：壳自己的 **"VM 程序" 另有其物**——  
描述符 `+0x00` 的表给出了 **98 段 / 总长 553** 的分段索引（静态可得；** 语义由 `边界表**： 它索引的是「553 项 × 24 B 的方法表」，不是"指令流"）； 而这里追到的 **24 字节/项结构体表**是 **ART 侧的方法解析结果**（由` JNIEnv`→`[p1+0x2F8]`→ 堆 →`[p2+0x28]`一路取到）， 两者的数据源、生命周期、用途都不同：**一个属于壳（自己写死），一个属于运行时（ART 填）**。 ⇒ 不再把"表项记录里的`(N, count)`"往"VM 程序"上靠；**两者的接合点已取到**（`thunk` 编号 ↔ Java 方法名）。

机制结论是 "`thunk_i` = 某个 Java native 方法的实现指针，**注册期**绑上去的 "。那就**在注册发生的那一刻**把三件套抓下来  
（`name` / `signature` / `fnPtr`）——`RegisterNatives` 正是 `JNIEnv` 函数表的第 **215** 项，**只换这一个函数指针**（仍是零代码改写）：

```
// 注入体片段：把 JNIEnv[215] 指向记录函数，记录后原样转发
void **tbl = *(void ***)env; // JNIEnv 第一个字段 = JNINativeInterface 表
mprotect(page_of(&tbl[215]), 0x1000, PROT_READ | PROT_WRITE);
g_orig = tbl[215];
tbl[215] = (void *)register_hook_probe; // hook：打印 methods[i].{name,signature,fnPtr} 后调用 g_orig
```

**时机是关键**：注册集中在 App 冷启动的头一两秒（注入早了没进程、晚了注册已结束），所以用 " **启动即注入** "——  
`am force-stop` → `am start` → **轮询 `pidof`，进程一出现立刻注入**  
另外两条踩过的坑：① 注入源 so **必须放在 App 的 `lib/arm64/`**（放 `/data/local/tmp` 会被 SELinux 挡住、进程直接死）；  
② 早期没有 `libnativehelper.so` 时 `JNI_GetCreatedJavaVMs` 拿不到，故 so 里备了一条**扫表兜底**（在 libart 的只读段里找  
"连续 200+ 个指向 libart 代码段的指针、且前 4 项为 0" 的数组，即 `JNINativeInterface`）。

**抓到的结果**：

<table><thead><tr><th>量</th><th>值</th></tr></thead><tbody><tr><td><code>RegisterNatives</code> 调用</td><td><strong>79 次</strong>（持续发生：每加载一批类就补注册一批）</td></tr><tr><td>抓到的 native 方法</td><td><strong>206</strong> 个（唯一名 176 个）；名字多为混淆短名（<code>l</code>/<code>r</code>/<code>ra</code>/<code>b2b</code>/<code>m</code>/<code>sa</code>/<code>al</code>…）</td></tr><tr><td>同一进程读出的 553 项表</td><td><strong>553 项、全唯一</strong>；表项 [0] = <code>base + 0x4AD0</code>；间距 <strong><code>0xB8</code>×545 + <code>0xB4</code>×7</strong></td></tr><tr><td><strong><code>fnPtr</code> 命中 553 项表</strong></td><td><strong>62 个</strong></td></tr></tbody></table>

**这 62 个对位长这样**：

<table><thead><tr><th>表项 idx</th><th>方法名</th><th>签名</th></tr></thead><tbody><tr><td>36</td><td><code>attachBaseContext</code></td><td><code>(Landroid/content/Context;)V</code></td></tr><tr><td>213</td><td><code>onCreate</code></td><td><code>(Landroid/os/Bundle;)V</code></td></tr><tr><td>241</td><td><code>finish</code></td><td><code>()V</code></td></tr><tr><td>242</td><td><code>getStaticFilePath</code></td><td><code>(Ljava/lang/String;)Ljava/lang/String;</code></td></tr><tr><td>252</td><td><code>dispatchTouchEvent</code></td><td><code>(Landroid/view/MotionEvent;)Z</code></td></tr><tr><td>254</td><td><code>onKeyDown</code></td><td><code>(ILandroid/view/KeyEvent;)Z</code></td></tr><tr><td>…</td><td>（<code>onResume</code>/<code>onStart</code>/<code>onStop</code>/<code>onDestroyView</code>/<code>setUserVisibleHint</code>… 同一批）</td><td></td></tr></tbody></table>

⇒ 三条结论：

1.  **"553 项表 = 被注册的 native 方法实现" 从推断变成实测**：62 个 `fnPtr` **就在表里**，  
    而它们对应的方法名几乎全是**框架回调 / 生命周期**（`attachBaseContext`/`onCreate`/`onResume`/`dispatchTouchEvent`/`onKeyDown`/`finish`…）  
    ⇒ 这正是 " **壳把 App 自己的方法改成 native**"的那一批（与" 指令抽取壳 "里那种" 补空桩 " 是同一件事的两面）。
2.  **表项索引就是 `thunk` 内嵌的那个 "编号"⇒ "编号空间" 至此有了名字**：  
    `编号 i` ⇄ `553 项表[i]` ⇄ `某个 Java native 方法`。这补齐了 "语义侧" 的锚点。
3.  **注册面 ≠ 跳板面**：206 个被注册方法里只有 62 个的实现指针在表内，其余落在 `libmmkv.so` / `libsqlcipher.so` / 匿名段等  
    （各自库自己的 `JNI_OnLoad`）—— 所以 "553 项表" 只覆盖**壳自己接管的**那一批方法。  
    离线旁证：App 的 4 个 dex 里 `ACC_NATIVE` 声明共 **612** 个  
    ⇒ "553" 是**壳接管的子集**，不是 "App 里所有 native 方法"。

> 复现：注入体→ 推到 App `lib/arm64/` → `ptrace_inject64 <pid> <so>`（启动即注入）  
> → `od -j <libexecmain_base+0x28008> -N 4424 /proc/<pid>/mem` 读表 → 求交集。

**第二次独立运行—— 结论是 "可复现"，不是 "能做大"**：

<table><thead><tr><th>轮</th><th>触发</th><th>抓到的 native 方法</th><th><code>RegisterNatives</code> 调用</th><th>命中表项</th></tr></thead><tbody><tr><td>第 1 轮</td><td>启动 + tab / 滑动 / 返回</td><td>206</td><td>79</td><td><strong>62</strong>（索引 36~535）</td></tr><tr><td>第 2 轮</td><td>启动 + 更多页面 / 更深点击</td><td>206</td><td>79</td><td><strong>62</strong>（索引 36~535）</td></tr><tr><td>合并</td><td>—</td><td>206（去重后）</td><td>158</td><td><strong>62，其中 62 个两轮都命中（100% 重叠）</strong></td></tr></tbody></table>

两点值得单独记下：

1.  **交集完全可复现**：两次**独立运行**（不同进程、不同 ASLR）抓到的方法集合与命中的表项索引**逐项相同**（62/62 重叠），  
    而且 **553 项表两次逐项一致**（表项 [0] 都是 `base+0x4AD0`、间距都是 `0xB8`×545 + `0xB4`×7）  
    ⇒ 交集是**结构事实**，不是采样噪声；ASLR 只改地址、不改内容这一点再次被实测。
2.  **注册面是 "走到过的类"**：再怎么加页面路径，注册数都**稳定收敛在 206 个方法 / 79 次调用**；  
    而 553 项表里只有 **62 个索引（11.2%）被锚上 ⇒ 壳的表是 "可注册面"，实际注册的是本次运行真正走到的类**。  
    要把 553 全锚上，需要把 App 的全部业务路径走一遍 —— 这也解释了为什么 "表大而注册少"。

<table><thead><tr><th>#</th><th>维度</th><th>壳 A（SecNeo）</th><th>壳 B（iJiami）</th><th>看点</th></tr></thead><tbody><tr><td>1</td><td><strong>分派结构</strong></td><td>中心跳转表（<code>jpt_109E8</code> + <code>BR X8</code>）</td><td>threaded-code 尾调用 + 553 项描述符 / 跳板</td><td>两种分派</td></tr><tr><td>2</td><td><strong>指令集规模</strong></td><td><strong>16 opcode</strong></td><td>256 项归一化 + <strong>872 项 handler 表（531 去重）</strong>；形状上：严格同形 <strong>206 类</strong>、骨架 + 相似度 <strong>33~59 类</strong>（前 10 类覆盖 **89~93%** 表项）</td><td>约 30×（但<strong>形状种类</strong>远没那么悬殊）</td></tr><tr><td>3</td><td><strong>VM 状态</strong></td><td><code>X25</code> 虚拟内存 + <code>W28</code> 状态寄存器 + <code>W19</code></td><td>栈上 IP + 栈上虚拟寄存器 + 22 通用寄存器</td><td>数据混淆型 vs 寄存器传递型</td></tr><tr><td>4</td><td><strong>指令语义模式</strong></td><td>统一 <code>byte[X25+addr] ^= key</code></td><td>handler 做真实计算</td><td>混淆数据 vs 混淆控制流</td></tr><tr><td>5</td><td><strong>索引 / 操作数混淆</strong></td><td>W28 四类原语（仿射 / 位混淆 / 非线性）</td><td><code>MUL</code>/<code>MOVK #0xCCCC</code>/<code>EXTR #2</code>/<code>CSEL</code></td><td>是否可符号化</td></tr><tr><td>6</td><td><strong>反篡改</strong></td><td>opcode 15 结束指令含<strong>哨兵校验</strong>（失败走 <code>sub_2C80</code>）</td><td>描述符 <code>+0x18</code>/<code>+0x20</code> 留给外部</td><td>自校验设计取向</td></tr><tr><td>7</td><td><strong>保护对象</strong></td><td><code>cL</code> 参数解析 + methodID 分发（<strong>Instruction Stealing</strong>）</td><td><strong>壳自身</strong></td><td>"偷指令" vs "护自身"</td></tr><tr><td>8</td><td><strong>跨模块</strong></td><td>单 so</td><td>跨 <code>libexecmain</code> + <code>libexec</code></td><td>分派链长度</td></tr><tr><td>9</td><td><strong>取码方式</strong></td><td>unidbg 全流程 → dump RWX 段</td><td>unidbg 触发自解密 + <code>/proc/mem</code> 取匿名段</td><td>同一工具两种用法</td></tr><tr><td>10</td><td><strong>动态验证</strong></td><td>JNI 日志全链路 + Java 回环</td><td>ROM 三点打点 + 段取证</td><td>内省式 vs 带外式</td></tr><tr><td>11</td><td><strong>完成度</strong></td><td>语义层完成 + 真机 trace 跑通 + <strong>③ 档（4/4 命中）</strong></td><td>分派 trace 一致（②）+ <strong>③ 档（篡改索引 ⇒ 2/2 崩溃）</strong>+ 证伪一拍并查清 thunk 由 JIT 调用；handler 语义部分（模板 × 参数；<strong>167/168 形状、99.9% 表项已还原</strong>，仅剩 1 个 <code>sshll</code>(SIMD) 形状）</td><td><strong>两壳均到 ③ 档</strong>（分派层）</td></tr><tr><td>12</td><td><strong>卡点性质</strong></td><td><strong>动态侧</strong></td><td><strong>静态侧</strong></td><td>环境的锅 vs 数据的锅</td></tr></tbody></table>

**两个壳的 VMP 分派结构对比**  
![](https://bbs.kanxue.com/upload/attach/202610/1006543_YYFEBC4J49JVPHV.webp)

**表 4.1b：两个 VM 的指令集对照**

<table><thead><tr><th></th><th>壳 A（16 opcode）</th><th>壳 B（256 项置换 + 531 handler）</th></tr></thead><tbody><tr><td>指令编码</td><td>定长（1 字节 opcode + 内联参数，由 W28 混淆派生）</td><td>⚠️ <strong>早期曾误读为 "变长指令边界"</strong>（<code>+0x00</code> 的 <code>(累计偏移, 长度)</code> 表：98 项 / 总长 553 B / 单条 1~53 B）；<strong>经非法地址注入确认为「553 项方法表的分段索引」，不能作为指令长度证据</strong></td></tr><tr><td>数据面</td><td><code>X25</code> 虚拟内存（字节级 XOR）</td><td>栈上虚拟寄存器（<code>STACK[0x81C]/0x82C]</code>）+ IP（<code>STACK[0x820]</code>）</td></tr><tr><td>控制流</td><td><code>next</code> 字段（表内硬编码，含 "重分发"）</td><td>尾调用下一个 handler</td></tr><tr><td>结束条件</td><td>opcode 15 哨兵校验 + 弹栈 RET</td><td>✅ <strong>已归类</strong>：<strong>无显式哨兵 / 越界比较</strong>（<code>0x229</code> 零命中）⇒ 收尾是 "尾调用链回到宿主"</td></tr><tr><td>JNI 交互</td><td>opcode 10（methodID 查表 → JNIEnv 槽位调用）</td><td>✅ <strong>已归类</strong>：<code>blr</code>（106/531 个 handler）是<strong>尾调用 / 槽位调用</strong>，非 JNI 反射；JNI 面在分派层</td></tr></tbody></table>

1.  **"VM 是什么" 不同**：壳 A 的 VM 是**控制流混淆器**（把真实指令流藏进加密字节码，  
    每条 opcode 只做一次 `X25` 字节改写 + 状态推进）；壳 B 的 VM 是**真正的指令层**  
    （handler 之间有尾调用、状态在寄存器 / 栈之间流动，`531 : 256 ≈ 2:1` 说明同一语义有多个变体入口）。  
    ⇒ 两者的还原目标因此不同：**A 要还原 "程序"（字节码序列），B 要还原 "语义"（handler 含义）**。
2.  **状态模型不同**：A 用**单个状态寄存器 `W28` 串起整条链**（并且是唯一的状态），  
    这让 "符号执行" 非常自然；B 的状态散在**栈上 IP + 虚拟寄存器 + 22 个通用寄存器**里，  
    单点观测看不到全貌，需要多点交叉。
3.  **反篡改的存在感不同**：A 在 VM 内部就埋了哨兵（结束指令校验一个运行时重定位的全局量），  
    说明它预期 "有人会去改它"；B 把 `dispatch` 交给外部填，更像是 "分层设计" 而非 "防改"。

> **归因边界（重要）**：A 与 B 之间**同时变了四件事**—— 厂商、样本、Android 版本、VM 设计，  
> **没有做变量控制**。所以上表只能读作 " **观测到的三个差异** "，**不能**读成  
> "用中心表的设计就一定只护自身" 这类规律 —— 反例完全可能：一个用中心跳转表、却把业务方法也 VM 化的壳。  
> 要谈因果，得在**同厂商的不同样本**或**同一样本的不同版本**上分别复测。

*   **"磁盘上为空、运行时才有"**：A 的哨兵值 `off_107FF0` 与 B 的 `core+0x20` 是同一个坑的两个实例 ——  
    **凡是 "运行时重定位 / 运行时填入" 的量，静态都拿不到**，必须先做一次动态取数；
*   **工具会把 "填充数据" 误判成代码**：B 的磁盘 rx 段按 4 KiB 扫描时 `udf` 占 **34,803 / 65,536（53%）**，  
    随机字节约六成能 "解码" 成合法指令 —— 所以**不能用 "可解码率" 判断一段内存是不是代码**。

**做法**：把 "能静态确定的东西" 全部挖出来 —— 置换表、描述符、跳板、handler 表、代码画像。

**优点**：不需要设备、不需要跑起来，**结果可复现、可交叉验证**（B 的两份表都有脚本能逐字节重现）。

**边界**：只能到 "结构 + 分派面"。**凡是运行时填的表、运行时生成的指针，静态一律拿不到**（B 的第 3 跳）。

**适用判据**：只要目标壳的 VM 是 "表驱动 + 代码可 dump"，静态就能吃下结构层。

**做法**：在 unidbg 里把 loader/VM 真跑一遍，用符号执行模拟器复现 opcode 行为，再与真实 trace 对拍。

**优点**：能直接得到**执行语义**（A 拿到的 16 opcode 语义表 + 状态链 + 模拟器）；  
验证力强 —— "真实路径对拍" 比任何静态推断都硬。

**边界**：  
**受模拟环境限制**。A 的 `#238` 就是被 unidbg 的指针 / 上下文传递问题挡住；  
而且模拟跑通 ≠ 真机跑通——这次真机 trace 还顺带查出**模拟器的状态更新与真机不一致**。

**适用判据**：VM 规模小（十几条 opcode）、状态可符号化（A 的 `W28` 只有 4 类原语）、  
且有稳定的动态入口。

```
① 分派是"中心跳转表"还是"尾调用/threaded"？
 中心表 → 静态可直接枚举 opcode 集合
 threaded → 需要先找到"handler 表"（通常在 r--p 段），再按表项地址当入口分析
 ② 关键表/指针是"磁盘上就有"还是"运行时才填"？
 磁盘上有 → 静态够
 运行时填 → 必须先动态取一次（ROM 打点 / 读 /proc/<pid>/mem）
 ③ opcode 数与状态复杂度：
 十几条 + 单一状态寄存器 → 符号执行模拟器性价比最高
 上百条 + 状态分散 → 先聚类，再抽样验证
```

`#238` 在 unidbg 里崩于 opcode 10，破解方向：

1.  ✅ **换执行环境**：在真机上让壳自己跑 `#238` —— 不读堆，而是  
    **零代码改写**地把 payload 的 16 项跳转表换成记录 stub，拿到 1524 条分派记录与 W28/W19 序列；
2.  **缩小模拟范围**：不追整个 `#238`，只把 opcode 10 的**入口上下文**（`X21`/`SP`）从真机 trace 里取回来，  
    在模拟器里以 "正确上下文" 重放这一段；
3.  **侧证补强**：算法层用宿主语义回环（A 已做，6 组 PASS）——它不能替代 VM 跑通，但能独立支撑算法结论。

`core+0x18/+0x20` 由解密段在运行时写入。**本次走的是第 2 条路（读内存）**，步骤：

1.  stealth 环境（Pixel XL 原包）→ 启动 App → 确认 `libexecmain` / `libexec` 出现在 maps；
2.  从 maps 取 `libexecmain` 基址 ⇒ 读 `base+0x29460` 拿到描述符地址 ⇒ 读描述符的 8 个 qword；
3.  发现 `+0x18`/`+0x20` 指向**匿名解密段里的两个跳板函数**，它们又转发到同一个 dispatch 主体；
4.  在 dispatch 主体里找到**混淆地址表**（`rw-p + 0xAED0`，表项 = 目标 − `0x430E51A3`）。

**踩到的坑（值得记下）**：设备 shell 是 **32 位算术**—— `0x797a404000 + 0x29460` 会被截断成  
`0x7a42d460`，第一次读出来全是 0。**64 位地址必须在 PC 上算好再传**。

> 拿到这张表之后，下一步是与已挖出的 **531 个 handler** 求交集：  
> 大量表项是同一函数的复用，真正 "被用到" 的 handler 集合通常远小于 531。

<table><thead><tr><th>壳 A 的卡点</th><th>用壳 B 的方法解</th></tr></thead><tbody><tr><td>模拟器跑不通</td><td><strong>带外真机取证</strong>（B 的 <code>/proc/mem</code> + 段取证套路）</td></tr><tr><td>缺 "真机侧对照"</td><td>B 的<strong>自研 ROM 打点</strong></td></tr></tbody></table>

<table><thead><tr><th>壳 B 的卡点</th><th>用壳 A 的方法解</th></tr></thead><tbody><tr><td>分派链接不上</td><td>A 的<strong>符号执行模拟器 + 真实 trace 对拍</strong>（先把 handler 语义符号化，再用 trace 验证）</td></tr><tr><td>531 个 handler 太多</td><td>A 的 "<strong> 先归类原语、再逐个确认</strong> "（A 把 <code>W28</code> 更新收敛成 4 类）</td></tr></tbody></table>

> 压成 **6 条判据**。每条都给出**检验方式**、"通过 / 不通过各长什么样" 与**实测值**；  
> **并单独列出已知的适用边界与反例**—— 判据的边界，本身就是结论的一部分。

<table><thead><tr><th>#</th><th>判据</th><th>检验方式</th><th>"通过" 长什么样</th><th>"不通过" 长什么样</th><th>实测</th></tr></thead><tbody><tr><td><strong>J1</strong></td><td>VMP 是否覆盖<strong>业务方法</strong></td><td>统计业务 dex 的 opcode 取值分布，看是否落进 AOSP 保留区 <code>0xE3–0xF9</code></td><td>出现非标准 opcode ⇒ 业务被 VM 化</td><td>全落在 <code>0x00–0xE2</code> ∪ <code>0xFA–0xFF</code> ⇒ <strong>只护壳自身</strong></td><td>5 份 dex / <strong>210,061</strong> 个方法，最大 <code>0xE2</code>（其中一份 <code>0xFB</code> 是标准新指令）⇒ <strong>只护壳自身</strong></td></tr><tr><td><strong>J2</strong></td><td>执行体是否<strong>静态可得</strong></td><td>对磁盘 <code>r-x</code> 段按 4 KiB 扫描，统计 " 非 <code>udf</code> 可解码率 "</td><td>可解码率高 ⇒ 明文</td><td>低（大量 <code>udf</code>）⇒ 密文，必须运行时取</td><td>壳 B 磁盘 <code>r-x</code> 段 <code>udf</code> <strong>34,803 / 65,536（53%）</strong>⇒ 密文；壳 A 的 loader 自解密到 RWX 段</td></tr><tr><td><strong>J3</strong></td><td>分派是<strong>中心表</strong>还是 <strong>threaded</strong></td><td>找主循环：是否存在 " 取表基址 → <code>ldr</code> 表项 → <code>br</code>" 的闭合结构</td><td>有中心表 ⇒ 枚举 opcode 集合即可</td><td>handler 之间<strong>尾调用</strong>、无中心循环 ⇒ threaded</td><td>壳 A：中心表 <strong>16 项</strong>（逐项验证 <strong>16/16</strong>）；壳 B：threaded（<strong>811 处</strong><code>br</code>/<code>blr</code> 就是 handler 尾巴）</td></tr><tr><td><strong>J4</strong></td><td>handler 表是否<strong>静态可读</strong></td><td>扫数据段找 "<strong> 连续</strong>指向代码段的 8 字节指针 "</td><td>命中 ⇒ 表在磁盘上</td><td>0 命中 ⇒ 表在运行时构造，<strong>或采集不完整</strong></td><td>壳 B 命中（<code>r--p</code> 段扫出 <strong>18 张 / 872 项</strong>）；<strong>第三样本 8 个 <code>r--p</code> 段全部 0 命中</strong></td></tr><tr><td><strong>J5</strong></td><td>关键指针是<strong>静态还是运行时</strong></td><td>看磁盘上该字段是否为 0</td><td>非 0 ⇒ 静态可读</td><td><strong>为 0 ⇒ 运行时由别的模块填入</strong>，必须动态取一次</td><td>壳 B 描述符 <code>+0x18</code>/<code>+0x20</code> 磁盘为 <strong>0</strong>⇒ 实测运行时被填成匿名解密段里的两个跳板</td></tr><tr><td><strong>J6</strong></td><td>表是否被<strong>常量混淆</strong></td><td>表项值是否<strong>落在任何已知段之外</strong></td><td>落在段内 ⇒ 直接可读</td><td>不在任何段内 ⇒ 被常量混淆，得先找出那个常量</td><td>壳 B <code>rw-p + 0xAED0</code>：表项 <code>0x74336BC13D</code> 不在任何段内 ⇒ <strong>表项 = 目标 − <code>0x430E51A3</code></strong></td></tr></tbody></table>

**这 6 条的定位**：它们是 **"我们在两个样本上真正用到的判断点"，  
不声称穷尽 VMP 的判断维度 —— 换个样本很可能合并成 4 条、也可能拆出 8 条。  
把它们写成 "可否证的命题" 而不是 "规律"，就是为了让后来者用第三、第四个样本去改它

**可执行版本**：附带的判据脚本 —— 把下面那张 "输入 / 工具 / 输出 / 阈值" 表做成一条能跑的命令，  
喂 so/dex 就打印六条判据的判定与中间量；**缺输入的一律标 NOT-EVALUABLE，不猜**  
（两壳实测：J2 `34,803 / 65,536`、J3 `br = 2 / blr = 0`（只统计该判据输入的那一段，与 "整个解密段 811 处" 不是同一口径）、J4 `18 张 / 872 项`，与正文逐项一致）。

**判据速查表（输入 → 工具 → 输出 → 阈值）**—— 上表是 "命题与实测"，这张表是 " **照着跑要准备什么** "：

<table><thead><tr><th>#</th><th>输入</th><th>工具（用的）</th><th>输出</th><th>阈值 / 判定</th></tr></thead><tbody><tr><td>J1</td><td>业务 dex（<code>classes*.dex</code>）</td><td>任意 dex 解析器（如自写的 dex 扫描脚本）</td><td>opcode 取值直方图</td><td>最大值 ≤ <code>0xE2</code> ⇒ 只护壳自身</td></tr><tr><td>J2</td><td>磁盘 so 的 <code>r-x</code> 段</td><td>capstone 按 4 KiB 对齐解码（失败按 <code>udf</code> 计）</td><td>" 非 <code>udf</code> 可解码率 "</td><td>高（≈ ≥90%）⇒ 明文；低（实测 53%）⇒ 密文</td></tr><tr><td>J3</td><td>解释器区反汇编</td><td>capstone + 手写 "回边 / 入度" 统计</td><td>中心表项数 / <code>br</code> 尾调用处数 / <strong>回边入度直方图</strong></td><td>一个高入度地址 ⇒ 中心表；无 ⇒ threaded；<strong>两个 ⇒ 混合体</strong></td></tr><tr><td>J4</td><td>数据段（<code>r--p</code> / <code>rw-p</code>）字节</td><td>扫 "<strong> 连续</strong>指向代码段的 8 字节指针 "</td><td>命中的表数 / 项数</td><td>命中 ⇒ 静态可读；<strong>0 命中 ⇒ 表在运行时构造，或采集不完整（必须先排除后者）</strong></td></tr><tr><td>J5</td><td>描述符 / 结构体字段</td><td>磁盘画像 <strong>vs</strong> 运行时 <code>/proc/&lt;pid&gt;/mem</code> 对比</td><td>该字段在磁盘上的值</td><td>磁盘非 0 ⇒ 静态；<strong>磁盘 0 ⇒ 运行时由别的模块填入</strong></td></tr><tr><td>J6</td><td>表项值 + <code>/proc/&lt;pid&gt;/maps</code></td><td>逐项判 "是否落在任何已知段内"</td><td>落在段内的比例</td><td>全落段内 ⇒ 可直接读；否则 ⇒ 存在常量偏移（实测 <code>0x430E51A3</code>），要先解出来</td></tr></tbody></table>

> 说明：表中 "阈值" 一列里，只有 J2 是**区间型**（可解码率高低），其余都是**二值 + 一个可复算的数字**；  
> 这样设计是为了让负面结果也能被复核（例如 J4 的 "0 命中" 必须连同 "采集范围" 一起记录）。

```
# J1：业务 dex 有没有被 VM 化 —— 判据线：code_item 首指令的 opcode 是否落进 AOSP 保留区 0xE3~0xF9
import struct, sys
LO, HI = 0xE3, 0xF9
NATIVE, ABSTRACT = 0x100, 0x400
u32 = lambda b, o: struct.unpack_from("<I", b, o)[0]

def uleb(b, o):
    v = s = 0
    while True:
        x = b[o]; o += 1; v |= (x & 0x7F) << s; s += 7
        if not x & 0x80: return v, o

def scan(path):
    b = open(path, "rb").read()
    assert b[:4] in (b"dex\n", b"dey\n"), "不是 dex：%r" % b[:4]
    n_m = n_code = hit = 0
    for i in range(u32(b, 0x60)):
        o = u32(b, u32(b, 0x64) + i * 32 + 24)
        if not o: continue
        sf, o = uleb(b, o); inf, o = uleb(b, o)
        dm, o = uleb(b, o); vm, o = uleb(b, o)
        for _ in range(sf + inf):
            _, o = uleb(b, o); _, o = uleb(b, o)
        for cnt in (dm, vm):
            for _ in range(cnt):
                _, o = uleb(b, o)
                flags, o = uleb(b, o)
                code = 0
                if not flags & (NATIVE | ABSTRACT):
                    code, o = uleb(b, o)
                n_m += 1
                if code:
                    units = u32(b, code + 12)                 # insns_size（16-bit 单位）
                    if units and code + 16 + units * 2 <= len(b):
                        n_code += 1
                        if LO <= (b[code + 16] & 0xFF) <= HI:  # insns 首单元；低字节才是 opcode
                            hit += 1
    return n_m, n_code, hit
```

```
# J2/J3：执行体静态可得吗 + 分派是中心表还是 threaded
#   J2 判据线：按 4 KiB 对齐逐 4 字节解码，统计解成 `udf` 的比例（高=明文，低=密文）
#   J3 判据线：找 "adrp/add + ldr …,lsl #3 + br" 的闭合窗口（有=中心表，无=尾调用链）
import sys
from capstone import Cs, CS_ARCH_ARM64, CS_MODE_LITTLE_ENDIAN

def scan(path, base=0, chunk=4096):
    blob = open(path, "rb").read()
    md = Cs(CS_ARCH_ARM64, CS_MODE_LITTLE_ENDIAN)
    total = undec = udf = br = blr = 0
    recent, cand = [], []
    for c0 in range(0, len(blob), chunk):
        for off in range(c0, min(c0 + chunk, len(blob)) - 3, 4):
            ins = next(md.disasm(blob[off:off + 4], base + off), None)
            total += 1
            if ins is None or ins.mnemonic == "udf":
                udf += (ins is not None); undec += (ins is None); recent = []
                continue
            if ins.mnemonic in ("adrp", "add", "ldr", "ldrsw"):
                recent.append((ins.mnemonic, ins.op_str)); recent = recent[-12:]
            if ins.mnemonic == "br":
                br += 1
                if any(m in ("ldr", "ldrsw") and "lsl #3" in o for m, o in recent) \
                   and any(m == "adrp" for m, o in recent):
                    cand.append(off)              # 一个「取表基址→ldr 表项→br」闭合点
                recent = []
            elif ins.mnemonic == "blr":
                blr += 1; recent = []
    return dict(total=total, undec=undec, udf=udf, br=br, blr=blr, cand=cand)
```

```
# J4：handler 表能不能静态扫出来
#   判据线：数据段里有没有"连续 ≥6 个指向可执行段的 8 字节指针"
#   用法：python j4.py <数据段.bin> <数据段基址> <可执行段.bin> <可执行段基址>
import struct, sys
data_path, dbase, code_path, cbase = sys.argv[1], int(sys.argv[2], 0), sys.argv[3], int(sys.argv[4], 0)
MIN = 6
data = open(data_path, "rb").read()
lo, hi = cbase, cbase + len(open(code_path, "rb").read())
vals = struct.unpack_from("<%dQ" % (len(data) // 8), data, 0)
tables, i = [], 0
while i < len(vals):
    if lo <= vals[i] < hi:                                  # 落在可执行段内 ⇒ 疑似函数指针
        j = i
        while j < len(vals) and lo <= vals[j] < hi: j += 1
        if j - i >= MIN: tables.append((i * 8, j - i, vals[i]))
        i = j
    else:
        i += 1
print("命中 %d 张表，共 %d 项" % (len(tables), sum(n for _, n, _ in tables)))
for off, n, first in tables[:5]:
    print("  +%#06x：%d 项，首项 %#x" % (off, n, first))
print("⇒ %s" % ("静态可读" if tables else "0 命中：表在运行时构造，或采集不完整（按 §7.2-1 必须先排除后者）"))
```

```
# J5：关键指针是静态还是运行时
#   判据线：**磁盘上**该字段是不是 0（是 0 ⇒ 运行时由别的模块填入）
#   用法：python j5.py <文件> <文件偏移>
import struct, sys
path, off = sys.argv[1], int(sys.argv[2], 0)
v = struct.unpack_from("<Q", open(path, "rb").read(), off)[0]
print("%s 偏移 %#x 的 8 字节 = %#x" % (path.split("/")[-1], off, v))
print("⇒ %s" % ("磁盘为 0 ⇒ 运行时由别的模块填入（必须再取一次运行时镜像才能定案）" if v == 0
               else "非 0 ⇒ 静态可读，可直接按这个地址继续追"))
```

```
# J6：表项有没有被常量混淆
#   判据线：表项值是否落在**任何已知段**之外（落在外 ⇒ 存在常量偏移）
#   用法：python j6.py <表.bin> <表在文件内的偏移> <项数> <段1的lo-hi> <段2的lo-hi> ...
#   段表从 /proc/<pid>/maps 直接抄；本文实测例：壳 B 的表项 = 目标 − 0x430E51A3
import struct, sys
path, off, n = sys.argv[1], int(sys.argv[2], 0), int(sys.argv[3], 0)
segs = [tuple(int(x, 0) for x in a.split("-")) for a in sys.argv[4:]]
blob = open(path, "rb").read()[off:]
inside = outside = 0
deltas = []
for i in range(n):
    v = struct.unpack_from("<Q", blob, i * 8)[0]
    if any(lo <= v < hi for lo, hi in segs):
        inside += 1
    else:
        outside += 1
        cand = [v - lo for lo, hi in segs if v > hi]
        if cand:
            deltas.append(min(cand))
print("表项 %d：段内 %d，段外 %d" % (n, inside, outside))
if outside:
    print("候选常量（(段基址 − 表项) 的最小正值，前 5 个）：%s"
          % ", ".join("%#x" % d for d in sorted(set(deltas))[:5]))
    print("⇒ 存在常量混淆：若某个候选值在**全部**段外表项上恒定，它就是那个常量")
else:
    print("⇒ 表项就在段内 ⇒ 没有被常量混淆，可直接读")
```

**① J4 的反例：第三样本 8 个 `r--p` 段全部 0 命中。**  
这**不能**推出 "它不是表驱动 VM"，因为 "0 命中" 至少有三种竞争解释：

1.  它的 handler 表不在 `r--p`（可能在 `rw-p` 或匿名段）；
2.  表是**运行时构造**的（那就落进 J5/J6 的场景）；
3.  **我们的采集不完整**（该样本只有 7 组段 dump，表指向的段可能压根没采到）。

⇒ **方法学结论：负面判据必须先排除 "采集不完整" 这个竞争解释**。  
对这个样本，只能说 " **未观察到** "，不能说" 不存在 "。

**② J1 的边界**：它的 "保留区" 判据只对**在 Dalvik 字节码层做 VM 化**的壳有效。  
若壳在 **native 层**做指令替换（两壳都不是），J1 会给出 "只护壳自身" 的**假阴性**。

**③ J6 的边界**：找出混淆常量，依赖 " **还原后的目标必须落在某个已知段内** " 这一收敛条件。  
若混淆是多级的（例如目标还要再解一次），该条件不成立，只能反过来用动态读表。

**④ J3 的边界**：两壳恰好在两个极端（纯中心表 / 纯尾调用）；  
现实中很可能存在**混合体**（外层中心表 + 内层尾调用）。**混淆写轮把这一格从 "悬而未决" 变成了 "可测判据"**

**J3′：分派层数的判定**

1.  **列出所有 "分派式跳转"**：形如 `ldr xN,[xB,xI,lsl#k]`（或 `ldrh/ldrb`）紧跟 `br xN`，  
    或编译器生成的 case 表 `ldrsw xN,[表+xI,lsl#2]` 紧跟 `br xN`；
2.  **统计比值**：`自带分派式跳转的 handler 数 / handler 总数`；
3.  **按阈值判**：

<table><thead><tr><th>比值</th><th>判定</th><th>依据</th></tr></thead><tbody><tr><td><strong>≤ 2%</strong>（且集中在 1~2 个、<strong>入度极高</strong>的 handler 上）</td><td><strong>单级</strong></td><td>那 1~2 个就是<strong>分派器本身</strong>，其余 handler 只 "回分派器" —— <strong>壳 A 0/16 = 0%</strong>、<strong>壳 B 1/531 = 0.19%</strong></td></tr><tr><td>≥ 10%（且<strong>分散</strong>在多个 handler 上）</td><td><strong>两级 = 混合体</strong></td><td>每个带分派的 handler 都是一个 "内层分派器"</td></tr></tbody></table>

**辅助判据 ①（定 "哪层是外层"）：看表所在段的权限**——  
**只读段（代码段内）的表 = 编译器生成的 case 表 ⇒ 外层**（opcode 级）；  
**运行时可写段 / 匿名段的表 = 壳自建 ⇒ 内层**（VM 指令级）。  
两壳正好落在两侧：壳 A 的 16 项表是**编译期常量**（与解释器同段、从不被改写），  
壳 B 的 **18 张 handler 表**在 `libexec.so` 的 **`r--p` 段**（静态可读）；运行时才填的是**分派入口**（描述符 `+0x18`/`+0x20`）。

**辅助判据 ②：把所有 handler 结尾的 `b <addr>` 的目标地址做直方图**——  
**单级**分派下只会出现**一个**入度 ≈ 全部 handler 的地址（实测：壳 A `0x109d4`、壳 B `0x74767ca8ac`）；  
**混合体**下会出现**两个**高入度地址（外层分派器 + 内层分派器），且它们引用的表分处 "只读 / 可写" 两段。

```
拿到一个未知 VM
├─ ① J1：业务代码是否被 VM 化？（扫 opcode 区间）
│ ├─ 是 → 目标是"还原业务语义"，成本高、价值高
│ └─ 否 → 只护壳自身 ⇒ 目标降为"判定覆盖范围 + 结构还原"（壳 B 情形）
├─ ② J2：代码是否可静态获得？
│ ├─ 是（明文/弱壳）→ 直接静态分析
│ └─ 否（整体加密/自解密）→ 取码：unidbg 触发 + /proc/mem 取段（A/B 都走了这条）
├─ ③ J3：分派形态？
│ ├─ 中心跳转表 → 枚举 opcode 集合 → 静态语义表
│ └─ threaded → 先按 J4 挖 handler 表 → 用表项地址当入口（别信 IDA 自动划分）
└─ ④ J5/J6：关键表是否运行时填 / 被混淆？
 ├─ 都不是 → 结构层可以一次做完
 └─ 是 → 安排一次定点动态取数（ROM 打点 / 读内存），否则链永远差一跳
```

<table><thead><tr><th>档</th><th>判据</th><th>达成情况</th></tr></thead><tbody><tr><td><strong>① 结构自洽</strong></td><td>映射链闭合、地址全落在预期段内、频次分布合理</td><td>壳 B：<code>553/553</code> 落点验证 + <code>531/872</code> 频次 ≈ <strong>达标</strong>（但链条第 2 跳被干预证伪）</td></tr><tr><td><strong>② trace 一致</strong></td><td>模型预测的分派 / handler 序列 == 真机记录</td><td>两壳<strong>均达标</strong>：壳 A <code>#239</code>（模拟器 ↔ trace）；壳 B <strong>已达成</strong>—— 真机注入 trace 30 条记录与 "表项 → thunk → 内嵌 w1 常量" 的静态预测<strong>逐位一致</strong></td></tr><tr><td><strong>③ 行为可证伪</strong></td><td>改模型输入 ⇒ 预测行为变化 ⇒ 真机复现</td><td><strong>两壳都已达成</strong>：壳 A 先扰动一个表项（4 条预测 <strong>4/4 命中</strong>），再把 <strong>16 个表项逐条重定向</strong>（<strong>16/16 命中</strong>）；壳 B <strong>四版干预均命中</strong>—— ①篡改分派索引 <strong>2/2 崩</strong>、对照 2/2 存活；②打坏 <code>ArtMethod-&gt;data_</code>（方法绑定）⇒ <strong>分派一次即崩</strong>、对照存活；③④ <strong>改边界表</strong>⇒ 壳<strong>完整性校验失败</strong>；三轮共 113 次运行把触发条件一路修正到「<strong>关键项 = <code>{14,65,83}</code></strong>」（单项致命 <strong>3/3</strong>、相邻项全稳、与错位量无关）</td></tr></tbody></table>

**做了什么**：拿第三个样本（7 组段 dump）按 J4 的 "检验方式" 直接跑了一遍 ——  
扫它的 **8 个 `r--p` 段**，找 " **连续**指向同批 dump 代码段的 8 字节指针 "。

**结果**：**8 个 `r--p` 段全部 0 命中**；它的 `rw-p` 段里有 **2,431 个指向代码段的重定位**，  
但那是**正常的 ELF 重定位**（同段内、间距不规则），不是 "handler 表"。

**这次 "未命中" 该怎么记**：

<table><thead><tr><th>可能的解释</th><th>是否成立</th><th>判据</th></tr></thead><tbody><tr><td>(a) <strong>J4 被推翻</strong>：存在 "表不在磁盘段里" 的壳</td><td>❌ <strong>不能下这个结论</strong></td><td>该样本只有<strong>段 dump、没有运行时内存镜像</strong>；<code>r--p</code> 段里没有表 **≠** 运行时没有表 —— 按 J4 的 "不通过" 形态，这正落在 "<strong> 或采集不完整</strong> " 那一支</td></tr><tr><td>(b) 该样本属于 <strong>J5 型</strong>（表在运行时由别的模块填）</td><td>✅ 与观测一致</td><td>它的 <code>rw-p</code> 段有大量重定位（是个<strong>动态链接的正常 ELF</strong>）；而两壳的 "运行时填表" 特征（壳 B 描述符 <code>+0x18/+0x20</code> 磁盘为 0）在它身上<strong>没有被观测到</strong>（缺运行时镜像）</td></tr><tr><td>(c) 该样本<strong>本就不是</strong> "<code>r--p</code> 放 handler 表 " 的形态</td><td>✅ 与观测一致</td><td>同上：<strong>"未观察到" ≠ "不存在"</strong></td></tr></tbody></table>

**⇒ 结论**：这次外推只得到一个 " **未观察到** "，**不足以判定 J4 的边界**。  
要让 J4 真伪可判，能同时给出 "磁盘段 + 进程内存" 两套数据。

**按验证强度自我定位**：

*   **壳 A 达 ③ 档** —— 两次扰动、事先写下 4 条预测、真机 **4/4 命中**；随后把 **16 个表项逐条重定向**、逐条预测 ⇒ **16/16 命中**；**41 条混淆写**的干预式尝试**不可达**（patch 全部生效但无可观测差异，dispatch trace 查明该流程 VM 只执行 op1/5/10/15，三次运行一致）。
*   **壳 B 达 ③ 档** —— 分派层：篡改分派索引 ⇒ 真机 **2/2 崩溃**、对照 2/2 存活；改 `ArtMethod->data_` ⇒ **分派一次即崩**、对照存活。数据层：改 `+0x00` 边界表 ⇒ 壳**完整性校验失败**、进程崩 / 重启，三轮共 113 次运行把触发条件修正到「**关键项 = `{14,65,83}`**」（单项致命 **3/3**、相邻项全稳、与错位量无关）。

**两个壳都没做到的事**：

*   壳 A：这个 ③ 档**只覆盖 "分派层 + 状态更新" 这个子模型**，不等于 16 个 opcode 的逐条语义都被干预式验证过；
*   壳 A：**「编号 → Java 方法名」裁决为 "运行时观测面已穷尽"** —— 六轮、七条路逐条实测排除后只剩 "行为级对拍"（高成本、非 "名字级"）；编号只以 `loc_12BD4` 的**瞬时寄存器实参**形式存在、不落在 VM 状态区（这也是 "VM 层对被保护方法完全同构" 的原因），而名字确是明文（壳私有字符串池，编码 `tag = 0xA0 | len`、用 ≥0x80 引用码做前后缀压缩），缺的只是 "编号 ↔ 池项" 这一层索引；
*   壳 B：**handler 内部的运算语义**没有做到逐条干预式验证 —— 已推进到 "模板 × 参数 + 逐形状净值"（167/168 形状、871/872 表项 = 99.9%），但那是**离线还原**，不是真机对拍；
*   壳 B：**分派 trace 只到 "分派了哪个索引"**，没有再追进 handler 内部；**`thunk` 内嵌编号 → Java 方法名**仍缺（`RegisterNatives` 只锚上 62/553 = 11.2%）。

**方法上的两处已知边界**：J4 的负面结果受 "采集完整性" 干扰（只能记 "未观察到"）；J3 的 "混合分派形态" 已有可测判据（**J3′**：自带分派式跳转的 handler 占比 ≤2% ⇒ 单级、≥10% ⇒ 两级，辅以 "表所在段的权限" 与 "回边入度直方图"；两壳实测 0% 与 0.19%，都判为单级）。

**两条可复用的做法**：① **不依赖 linker 的手工符号解析**（`dl_iterate_phdr` → `PT_DYNAMIC` → `DT_GNU_HASH` → 扫 `.dynsym`）—— 两个坑：`d_ptr` 要做一次 "在不在本段内" 的绝对 / 相对修正（否则 `SIGSEGV @0x270`）、符号数必须用 GNU_HASH 的标准算法算，不能用 `(strtab-symtab)/24` 估；② **JNI 函数表的定位**走 `JNI_GetCreatedJavaVMs → env → *env`，且**槽号就是数组下标**。

**（1）三句话**

*   同一个词 "VMP" 下**两种不同的东西**：壳 A 把 VM 当**数据混淆器**（偷控制流），壳 B 把 VM 当**真指令层**（护自己）——  
    名字一样，还原路径完全不同。
*   两种卡点**互为镜像**：壳 A 卡在**环境**（模拟器缺陷）、壳 B 卡在**数据**（关键表运行时才填）；解法可以互换。
*   "还原到哪一步" 要用**可检验的判据**说话：J1~J6 + 验证强度三档；两壳都做到 **③ 档**。

**（2）四条方法论**

*   **零代码改写的定点取数**：只改一个数据指针 / 表项 ⇒ 不会因为 "插桩改变了时序" 而让对照不再干净；
*   **先写预测、再看真机**：**没命中**（改 553 项表无效）比命中更有价值 —— 它当场证伪了一拍推断，  
    并顺带查出 `thunk` 其实由 JIT 代码调用；
*   **结构自洽 ≠ 语义**：`(off,len)` 98/98 自洽只说明 "它是个分段表"，**不说明分的是什么**；  
    读成 "变长指令边界"，直到 把 `core+0x00` 置成非法地址、让**读它的代码自己崩出来**  
    （匿名解密段 `+0x50810`）才定死它是**方法表分段索引**。**"把嫌疑字段变成非法值"** 是一个通用且便宜的手段；
*   **"崩了" 必须重复才算数**：113 次运行，每次都在修正上一轮的规律 ——  
    从 "12~16 项阈值"→"连续错位 ≥16 项"→最终 " **关键项 `{14,65,83}`**"。  
    三次修正里唯一没变的是那条方法论：**二值观测量单次观测没有判别力**（散布组实测偶发崩率 ≈16%）。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)

最后于 17 小时前 被 kudos5566 编辑 ，原因：