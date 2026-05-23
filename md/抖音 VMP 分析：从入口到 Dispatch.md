> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2109082-1-1.html)

> [md]# 抖音 VMP 分析：从入口到 Dispatch> ** 作者：** 人生导师 > ** 日期：** 2026 年 5 月 22 日 > ** 版本：** 抖音 38.1.0> **so 加密文件：**......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)rsds0duck

抖音 VMP 分析：从入口到 Dispatch
-----------------------

> **作者：** 人生导师  
> **日期：** 2026 年 5 月 22 日  
> **版本：** 抖音 38.1.0  
> **so 加密文件：**`libmetasec_ml.so`

最近在看抖音的签名保护，发现核心逻辑藏在一个自研的 VMP 里。这篇算是我的分析笔记，记录了从找到 VM 入口到搞清楚 dispatch loop 的过程。水平有限，如果有分析错误的地方欢迎指正。

* * *

### 一、找到 VM 入口

#### 方法：trace + IDA 逐行对照

trace 是执行记录，格式长这样：

```
行号  : 绝对地址  [偏移]  "指令"  寄存器变化
923   : 0x71ee4bd618  [0x2a6618]  "sub sp, sp, #0xa0" (r)sp=0x725beaea40 (w)sp=0x725beae9a0

```

一份 trace 最少五百万行汇编，最多一千一百万行。我手上有三份不同请求的 trace，目前在分析最短的那份（五百万行），另外还有一份七百万行的留着交叉验证。

追法很朴素：从入口开始，一条一条看地址偏移的末位。ARM64 指令固定 4 字节，正常顺序执行的话偏移末位是 `0 → 4 → 8 → C → 0 → 4 → 8 → C` 循环递增。一旦这个节奏断了——比如从 `0x2a6618` 突然跳到 `0x2ab240`——就说明有跳转发生了，这时候去 IDA 里看这条指令是什么：`b`、`bl`、`blr`、还是条件跳转。

但是也要看全，不能只盯末位。有时候两段不连续的代码正好拼在一起，末位看着是连续的，实际已经跳了。所以每次节奏 "看起来正常" 的时候也得瞄一眼完整偏移有没有突变。

遇到 `bl`（函数调用）就记录子函数的入口和返回点，确定它的范围。遇到 `blr`（间接调用）就更要注意，因为目标地址在寄存器里，可能跳到完全不同的地方。

#### 包裹函数和真函数

从 7 神入口 `0x2A6E38` 往里追，很快遇到一个 `bl`：

```
91f  : 0x71ee4bf434  [0x2a8434]  "bl #0x71ee4c2238"   // 包裹函数
922  : 0x71ee4c2240  [0x2ab240]  "b #0x71ee4bd618"    // 跳到真函数开头

```

这里的结构是：入口先调用一个包裹函数（wrapper），包裹函数里面再 `b` 跳转到真正干活的函数。`b` 不是 `bl`，不压返回地址，说明这是尾调用——包裹函数本身不做什么事，就是个跳板。

#### VM 入口定位

继续往下追，找到了关键调用：

```
c4d0  : 0x71ee4c4544  [0x2ad544]  "blr x8" (r)x8=0x71ee4c5a90
c4d1  : 0x71ee4c5a90  [0x2aea90]  "sub sp, sp, #0x30"  // VM 入口
2e0e3 : 0x71ee4c4548  [0x2ad548]  "ldp x29, x30, [sp, #0x20]"  // 返回后继续

```

`blr x8` 间接调用，跳到 `0x2AEA90`。从行号看，`c4d1` 到 `2e0e3`（返回点），中间执行了差不多 **18 万行**指令。这就是 VM 的主体了。

对比三份 trace：两份里这个 VM 入口内部自己循环了 11 次，一份循环了 10 次。说明 VM 不是一条直线跑完的，内部有 dispatch loop，根据输入数据的不同，循环次数会变。

但是有个好消息：VM 入口只被外部调用了一次。不用担心别的地方还会再次进入 VM、重新设置初始状态。分析范围是确定的。

#### VM 前的初始化段

既然 VM 入口只进一次，那进入之前的那段代码就是初始化逻辑——准备 VM 需要的上下文、参数、状态。

确定范围：

```
c465  : 0x71ee4b46cc  [0x29d6cc]  "stp x29, x30, [sp, #-0x60]!"  // 初始化开始
c4c4  : 0x71ee4b4780  [0x29d780]  "bl #0x71ee4c4518"              // 调用 VM

```

从行号 `c465`（50279）到 `c4c4`（50372），大概 **93 行**指令。这段就是进入 VM 前的全部准备工作。

另外还有一个封装函数 `0x2AA36C`，它是从 7 神入口到 VM 的桥梁。

#### 阶段小结

<table><thead><tr><th>内容</th><th>偏移</th><th>状态</th></tr></thead><tbody><tr><td>7 神入口</td><td><code>0x2A6E38</code></td><td>已定位</td></tr><tr><td>包裹函数 → 真函数</td><td><code>0x2A8434</code> → <code>0x2A6618</code></td><td>已确认，hook 验证</td></tr><tr><td>VM 前初始化段</td><td><code>0x29D6CC</code> ~ <code>0x29D780</code></td><td>已确定范围</td></tr><tr><td>VM 入口</td><td><code>0x2AEA90</code></td><td>已定位，内部循环 10~11 次</td></tr></tbody></table>

* * *

### 二、VM 前置初始化与架构猜想

#### EnterVM：7 神到 VM 的桥梁

从 7 神入口到 VM 之间的封装函数，地址 `0x2AA36C`：

```
__int64 EnterVM(parser_obj@X0, cookie_map@X2, flag@W3, output@X8)
{
  if (!*(*(parser_obj+16)+48))    // VM 未启用，直接返回空
    return empty_output;
  if (!*(cookie_map+24))          // cookie_map 没数据，走 fallback
    return fallback;

  v10 = sub_2AA968(stack_buf);    // 在栈上构造临时缓冲区
  obj = *(*(sub_15F060(v10)) + 304);  // 取 vtable+304 处的对象
  val = sub_25BE84(...);          // 计算一个值
  *(obj+8) |= 0x80;              // 设置标志位
  sub_282144(obj, bitwise_op(val, dword_3D5DF0));  // 位运算处理

  // 关键：通过虚函数表调用 vm_entry
  vtable = *(*(parser_obj+16));
  func = vtable[0x60/8];         // 第 12 个虚函数槽位
  func(*(parser_obj+16), v10, cookie_map, flag);
}

```

逻辑很直白：检查两个前置条件（VM 启用 + cookie 有数据），然后通过虚函数表间接调用 `vm_entry`。

从 trace 里确认的参数：

<table><thead><tr><th>参数</th><th>寄存器</th><th>值</th><th>含义</th></tr></thead><tbody><tr><td>parser_obj</td><td>X0</td><td><code>0x71ee5fdb58</code></td><td>解析器对象</td></tr><tr><td>cookie_map</td><td>X2</td><td><code>0x725beaee40</code></td><td>cookie 映射结构</td></tr><tr><td>flag</td><td>W3</td><td><code>0x171</code></td><td>标志位</td></tr><tr><td>output</td><td>X8</td><td><code>0x725beaed90</code></td><td>输出缓冲区</td></tr></tbody></table>

输出就是那堆签名 header：X-Perseus、X-Medusa、X-Argus、X-Helios、X-Ladon、X-Khronos、X-Gorgon。

#### vm_entry：初始化 VM 上下文

`EnterVM` 通过虚函数表跳到 `vm_entry`（`0x29D6CC`），这就是那 93 行初始化段：

```
__int64 vm_entry(parser_obj@X0, input_buf@X1, flag@W3, output_buf@X8)
{
  // 1. 确保字节码表只初始化一次
  pthread_once(&vm_once_flag, vm_bytecode_init);

  // 2. 在栈上分配 ~25KB 的 VM 上下文
  ctx = vm_ctx_init(stack);

  // 3. 把外部参数映射到 VM 寄存器
  vm_ctx_set_reg(ctx, 8,  output_buf);   // 输出缓冲区
  vm_ctx_set_reg(ctx, 9,  parser_obj);   // parser 内部对象
  vm_ctx_set_reg(ctx, 10, input_buf);    // 输入数据
  vm_ctx_set_reg11_cookie_map(ctx);      // cookie_map
  vm_ctx_set_reg(ctx, 12, flag);         // 标志位 0x171

  // 4. 启动 VM
  vm_run(vm_bytecode_program, ctx);
}

```

干净利落，五步走完。重点是第 3 步——外部世界的参数通过 `vm_ctx_set_reg` 映射到 VM 的虚拟寄存器里，VM 内部只通过寄存器槽位来访问这些数据。

#### VM Context 结构体

`vm_ctx_init`（`0x2AD554`）的实现：

```
__int64 vm_ctx_init(__int64 ctx) {
  *(uint32_t*)(ctx + 0) = 0;                 // PC/状态字清零
  *(uint64_t*)(ctx + 8) = 0;                 // 栈帧指针清零
  memset(ctx + 0x6050, 0, 0x280);            // 80 个通用寄存器清零
  *(uint64_t*)(ctx + 0x6058) = ctx + 0x6010; // reg_ptr 指向寄存器基址
  return ctx;
}

```

整个 VM 上下文大约 25KB，布局如下：

```
偏移        大小        用途
────────────────────────────────────────────────
0x0000      4 bytes    PC / 状态字
0x0008      8 bytes    栈帧指针（嵌套调用用）
0x0010      ~24KB      VM 执行栈（字节码运行时的临时数据区）
0x6010      0x40       保留寄存器 slot 0-7（不清零，VM 内部管理）
0x6050      0x280      通用寄存器 slot 8-79（80 个 8 字节槽位，初始化为 0）
0x6058      8 bytes    reg_ptr → ctx + 0x6010

```

寄存器存取公式：`*(ctx + slot*8 + 0x6050) = value`

slot 0-7 没被 memset 清零，说明是 VM 内部的特殊寄存器（PC、SP、条件标志之类的），由 dispatch loop 自己管理。

##### 初始寄存器分配

<table><thead><tr><th>Slot</th><th>偏移</th><th>值</th><th>含义</th></tr></thead><tbody><tr><td>8</td><td>0x6090</td><td><code>0x725beaed90</code></td><td>output 缓冲区</td></tr><tr><td>9</td><td>0x6098</td><td><code>0x737662c110</code></td><td>parser 内部对象</td></tr><tr><td>10</td><td>0x60A0</td><td><code>0x725beae790</code></td><td>输入数据缓冲区</td></tr><tr><td>11</td><td>0x60A8</td><td><code>0x725beaee40</code></td><td>cookie_map</td></tr><tr><td>12</td><td>0x60B0</td><td><code>0x171</code></td><td>flag 标志位</td></tr></tbody></table>

#### vm_bytecode_init：字节码加载与 handler 注册

`pthread_once` 保证这个函数只跑一次。它干的事情：

##### 1. 构造 handler 表

在栈上构造三张表：

*   **CF handler 表**：85 个条目（CF0 ~ CF54，十六进制），每个是一个 native 函数指针
*   **G handler 表**：9 个条目（G0 ~ G8），全局 / 内置操作码
*   **入口点描述表**：3 个条目，描述 VM 程序的命名入口

##### 2. 构建 VM 程序

```
program = sub_2AD3E0(
  &unk_387D30,   // 加密字节码数据（.rodata 段）
  0x30CD,        // 数据大小：12493 字节
  temp_buf,      // 临时缓冲区
  0,             // 未使用
  cf_handlers,   // 85 个 CF handler
  0x55,          // CF 数量
  g_handlers,    // 9 个 G handler
  9,             // G 数量
  entry_descs,   // 入口点描述表
  3              // 入口点数量
);

```

内部流程（`sub_2CFBC8`）：

**第一步：XOR 解密字节码**

字节码存在 `.rodata` 段 `0x387D30` 处，12493 字节，XOR 加密。解密 key 的选取：

```
int index = bytecode_size % entry_count;  // 0x30CD % 3 = 1
uint8_t key = entry_table[index].xor_key; // entry[1].xor_key = 0xF3
// 然后整块 XOR（NEON 加速，每次 32 字节）
for (i = 0; i < size; i++)
    bytecode_data[i] ^= 0xF3;

```

入口点描述表的结构：

<table><thead><tr><th>条目</th><th>type</th><th>XOR key</th><th>参数数量</th></tr></thead><tbody><tr><td>entry[0]</td><td>0x52 'R'</td><td>0x1B</td><td>6</td></tr><tr><td>entry[1]</td><td>0x52 'R'</td><td><strong>0xF3</strong></td><td>4</td></tr><tr><td>entry[2]</td><td>0x52 'R'</td><td>0xF5</td><td>2</td></tr></tbody></table>

**第二步：校验解密结果**（可能是 CRC/hash，确保字节码完整）

**第三步：解析字节码**（原始字节流 → 指令数组 + 常量池 + 函数表 + 字符串表）

**第四步：注册 handler**（把 85 个 CF + 9 个 G 的函数指针绑定到程序对象）

##### 3. 查找命名入口点

```
qword_3E64F8    = sub_2AD420(program, "F0");  // 备用入口
vm_bytecode_program = sub_2AD420(program, "F1");  // 主入口

```

程序对象内部有个 **名称 → 字节码偏移** 的哈希表。`vm_entry` 用的是 F1 入口，F0 目前没看到被调用。

#### vm_run：启动执行

`vm_run`（`0x2AD518`）本身很薄：

```
__int64 vm_run(__int64 entry_point, __int64 vm_ctx) {
  exec_mode = *(entry_point + 56);           // 取执行模式索引
  exec_func = off_367F08[exec_mode];         // 从函数指针表选执行器
  call_frame[1] = entry_point;
  call_frame[0] = &vm_ctx;
  return exec_func(call_frame);
}

```

`off_367F08` 是一个三元素的函数指针表：

<table><thead><tr><th>索引</th><th>地址</th><th>功能</th></tr></thead><tbody><tr><td>0</td><td><code>0x2AE9AC</code></td><td>简单执行（直接调用入口点函数）</td></tr><tr><td>1</td><td><code>0x2AE9E0</code></td><td>带栈帧的执行（设置帧头后进 dispatch loop）</td></tr><tr><td>2</td><td><code>0x2AEA90</code></td><td><strong>vm_execute_function</strong>：完整的 VM 函数执行器</td></tr></tbody></table>

实际走的是索引 2，也就是 `vm_execute_function`。

#### vm_execute_function：设置栈帧并进入 dispatch loop

```
__int64 vm_execute_function(__int64 **call_frame) {
  entry_point = call_frame[1];
  vm_ctx = **call_frame;

  stack_size = entry_point[1];              // 从入口点取栈帧大小（0x1d0）
  prev_frame = *(vm_ctx + 8);              // 保存当前栈帧（嵌套调用恢复用）
  new_stack_top = *(vm_ctx + 0x6058) - stack_size;  // 计算新栈顶

  // 栈溢出检查
  if (new_stack_top < vm_ctx + 16) crash();

  // 构造栈帧头：{PC偏移=0x154b, 栈大小=0x1d0}
  frame_header = {*entry_point, stack_size};
  *(vm_ctx + 8) = &frame_header;           // 设置新栈帧

  // 进入 dispatch loop
  vm_dispatch_loop(entry_point + 8, vm_ctx);

  // 恢复上一层栈帧
  *(vm_ctx + 8) = *(*(vm_ctx + 8) + 8);
}

```

关键信息：字节码从偏移 `0x154b` 开始执行，VM 栈帧大小 `0x1d0`（464 字节）。

#### 完整执行流程图

```
EnterVM (0x2AA36C)
│  检查 VM 启用 + cookie 有数据
│  通过虚函数表调用 ↓
│
vm_entry (0x29D6CC)
│  pthread_once → vm_bytecode_init
│  vm_ctx_init: 分配 25KB 上下文
│  设置 VM 寄存器 slot 8-12
│  vm_run(vm_bytecode_program, ctx)
│       │
│       ↓
│  vm_run (0x2AD518)
│       │  取 exec_mode → 选择执行器
│       │  exec_mode=2 → vm_execute_function
│       ↓
│  vm_execute_function (0x2AEA90)
│       │  设置栈帧: PC=0x154b, stack_size=0x1d0
│       │  栈溢出检查
│       ↓
│  vm_dispatch_loop (0x2AF5CC)
│       │  threaded dispatch: 每个 handler 直接跳下一个
│       │  opcode handler: 纯运算（ROR/DIV/LOAD/FADD/...）
│       │  CF handler: 调用 native 函数（85 个）
│       │  G handler: 全局操作（9 个）
│       │  循环 10~11 次（取决于输入）
│       ↓
│  返回签名结果
↓
输出: X-Perseus, X-Medusa, X-Argus, X-Helios, X-Ladon, X-Khronos, X-Gorgon

```

#### 架构猜想

到这里，VM 的骨架已经清楚了：

1.  **这是一个寄存器机**，不是栈机。有 80 个通用寄存器、独立的浮点区和条件标志区。
2.  **Threaded dispatch**：不走 switch-case，每个 handler 末尾直接跳下一个。这让静态分析很难追踪控制流，因为 IDA 看不到 handler 之间的调用关系。
3.  **字节码 XOR 加密**：key 从入口点描述表里取，用 `size % count` 选索引。简单但够用——防止直接 dump 字节码做模式匹配。
4.  **CF handler 是突破口**：VM 内部的纯运算很难追，但 CF handler 是 VM 和外部世界的接口。85 个 CF handler 里肯定包含了 MD5、SHA、AES 之类的密码学原语，把它们逐个识别出来就能还原算法骨架。
5.  **F0 和 F1 两个入口**：目前只用了 F1，F0 可能是另一种签名模式或者调试入口。

#### vm_bytecode_init 的全局变量

<table><thead><tr><th>地址</th><th>名称</th><th>用途</th></tr></thead><tbody><tr><td><code>0x3E64F0</code></td><td><code>vm_once_flag</code></td><td>pthread_once 标志</td></tr><tr><td><code>0x3E6500</code></td><td><code>vm_bytecode_program</code></td><td>F1 入口点（主入口）</td></tr><tr><td><code>0x3E64F8</code></td><td><code>qword_3E64F8</code></td><td>F0 入口点（备用）</td></tr><tr><td><code>0x3D5DF0</code></td><td><code>dword_3D5DF0</code></td><td>EnterVM 位运算常量</td></tr></tbody></table>

* * *

### 三、正式开始的 VM 分析

前面看了初始化和架构猜想，现在该来搞实际的 VM 了。那个跳转表快 800 个 handler，再加上 94 个 CF 和 G handler，接近 900 个。

#### Dispatch Loop：VM 的心脏

```
; void __fastcall vm_dispatch_loop(__int64 instruction_stream, _DWORD *vm_ctx)
vm_dispatch_loop:
STP             X29, X30, [SP,#-0x60]!
STP             X28, X27, [SP,#0x10]
STP             X26, X25, [SP,#0x20]
STP             X24, X23, [SP,#0x30]
STP             X22, X21, [SP,#0x40]
STP             X20, X19, [SP,#0x50]
MOV             X29, SP
MOV             X20, X0          ; X20 = instruction_stream (字节码指令流)
LDR             X0, [X0]         ; X0 = *instruction_stream = 第一条指令的元数据指针
MOV             W8, #0x6050
ADRL            X24, opcode_handler_table  ; X24 = 跳转表基址
ADD             X23, X1, X8      ; X23 = vm_ctx + 0x6050 = 整数寄存器数组基址
LDR             X8, [X0,#0x28]   ; X8 = 第一条指令的 opcode 索引
MOV             W9, #0x61D0
ADD             X25, X1, X9      ; X25 = vm_ctx + 0x61D0 = 浮点寄存器区
MOV             W9, #0x6150
MOV             X19, X1          ; X19 = vm_ctx 基址
STR             WZR, [X1]        ; *(vm_ctx) = 0，PC 归零
LDR             X8, [X24,X8,LSL#3]  ; X8 = handler_table[first_opcode]
ADD             X26, X1, X9      ; X26 = vm_ctx + 0x6150 = 条件标志区
BR              X8               ; 跳到第一个 handler，开始执行

```

IDA 把下面的 handler 全识别为独立函数，但实际上不是函数，而是一个正常执行的流程。这就是 **threaded dispatch**——每个 handler 执行完直接 `BR` 跳到下一个 handler，永远不回到这个入口点。整个 VM 执行期间，这些寄存器全程保持不变：

<table><thead><tr><th>寄存器</th><th>值</th><th>用途</th></tr></thead><tbody><tr><td>X19</td><td>vm_ctx</td><td>VM 上下文基址，<code>*(X19)</code> = PC</td></tr><tr><td>X20</td><td>instruction_stream</td><td>字节码指令流指针</td></tr><tr><td>X23</td><td>vm_ctx + 0x6050</td><td>整数寄存器数组（通过 <code>X23 + idx*8</code> 读写）</td></tr><tr><td>X24</td><td>opcode_handler_table</td><td>handler 跳转表</td></tr><tr><td>X25</td><td>vm_ctx + 0x61D0</td><td>浮点寄存器区</td></tr><tr><td>X26</td><td>vm_ctx + 0x6150</td><td>条件标志区</td></tr></tbody></table>

记住这张表，后面所有 handler 都靠这些寄存器工作。

#### 指令元数据结构

每条 VM 指令不只是一个 opcode + 操作数，它还有一个 48 字节的元数据结构：

```
偏移      内容
────────────────────────────
0x00      4字节操作数（编码了寄存器索引和立即数）
0x08      数据指针（某些指令用来存嵌入的地址）
0x28      下一条指令的 opcode 索引（dispatch 用）

```

所以你会在每个 handler 里看到这个寻址公式：

```
SMADDL X0, W8, W10, X9    ; X0 = PC_new * 48 + metadata_base

```

`PC * 48 + base` = 对应指令的元数据地址。48 就是那个到处出现的 `#0x30`。

#### 4 字节操作数编码

每条指令的核心操作数是 4 字节，但**编码格式不统一**，不同类型的指令拆法不一样：

```
31        16 15      8 7       0
┌──────────┬─────────┬─────────┐
│  HIWORD  │  BYTE1  │  BYTE0  │
└──────────┴─────────┴─────────┘

```

常见的拆法：

<table><thead><tr><th>指令类型</th><th>BYTE0</th><th>BYTE1</th><th>HIWORD</th></tr></thead><tbody><tr><td>STORE</td><td>base 寄存器</td><td>value 寄存器</td><td>有符号偏移</td></tr><tr><td>ADD_IMM</td><td>src 寄存器</td><td>dst 寄存器</td><td>有符号立即数</td></tr><tr><td>MOV_REG</td><td>src 寄存器</td><td>(unused)</td><td>dst 寄存器在 BYTE2</td></tr><tr><td>LOAD_INDIRECT</td><td>(unused)</td><td>dst 寄存器</td><td>有符号偏移</td></tr></tbody></table>

ARM64 里对应的解码指令：

*   `AND X, X, #0xFF` → 取 BYTE0
*   `UBFX X, X, #8, #8` → 取 BYTE1
*   `SBFX X, X, #0x10, #0x10` → 取 HIWORD 并符号扩展（偏移可以是负数）
*   `LSR X, X, #0x10` → 取 HIWORD 不符号扩展

#### Handler 通用模板

看了几个 handler 之后，发现它们全是同一个模板：

```
1. 取操作数    → 从 X0 或 [X0,#offset] 拿 4 字节编码
2. 解码        → AND/UBFX/SBFX 拆出寄存器索引和立即数
3. PC++        → STR W_new, [X19]（和运算无关，纯取指消费）
4. 读寄存器    → LDR X, [X23, Xidx, LSL#3]
5. 执行        → 一条核心操作（STR/ADD/LDR/...）
6. 算下一条地址 → SMADDL X0, PC, #48, base
7. 取 opcode   → LDR X, [X0, #0x28]
8. 查表        → LDR X, [X24, X, LSL#3]
9. 跳转        → BR X

```

区别只在第 5 步换了什么运算。一旦你把这个模板内化了，后面看新 handler 就是秒懂。

#### 实例分析：vm_op_store（单条 STORE）

地址 `0x2B05F4`，最简单的 handler，只做一件事：把寄存器的值写到内存。

```
vm_op_store:
LDR W8, [X0]              ; 取 4 字节操作数
MOV W13, #0x30            ; 48
LDR W9, [X19]             ; PC
LDR X12, [X20]            ; metadata_base
AND X10, X8, #0xFF        ; X10 = BYTE0 = base 寄存器索引
UBFX X11, X8, #8, #8     ; X11 = BYTE1 = value 寄存器索引
ADD W9, W9, #1            ; PC + 1
SBFX X8, X8, #0x10, #0x10; X8 = 有符号偏移
LDR X10, [X23,X10,LSL#3] ; X10 = regs[base] 的值
NOP
SMADDL X0, W9, W13, X12  ; 下一条指令元数据地址
LDR X11, [X23,X11,LSL#3] ; X11 = regs[value] 的值
STR W9, [X19]             ; PC = PC + 1
STR X11, [X8,X10]         ; ★ *(regs[base] + offset) = regs[value]
LDR X8, [X0,#0x28]        ; 下一个 opcode 索引
LDR X8, [X24,X8,LSL#3]   ; handler 地址
BR X8                     ; 跳

```

伪代码：

```
*(regs[base] + (int16_t)offset) = regs[value];
PC++;

```

就这么简单。注意 `SBFX` 做了符号扩展——偏移可以是负数，比如访问结构体前面的字段。

#### 实例分析：vm_op_load_indirect_imm（间接加载）

地址 `0x2B137C`，稍微不一样——数据源不是寄存器，而是指令元数据里嵌入的指针。

```
vm_op_load_indirect_imm:
LDR W8, [X19]             ; PC
MOV W10, #0x30            ; 48
LDR X9, [X20]             ; metadata_base
LDR W11, [X0]             ; 4 字节操作数
ADD W8, W8, #1            ; PC + 1
SMADDL X9, W8, W10, X9   ; 下一条指令元数据地址
LDR X10, [X0,#8]          ; ★ X10 = 元数据偏移 8 处的指针（数据源地址）
LSR X13, X11, #0x10       ; X13 = HIWORD（偏移量，待符号扩展）
UBFX X11, X11, #8, #8    ; X11 = BYTE1 = 目标寄存器索引
MOV X0, X9               ; 更新 X0
STR W8, [X19]             ; PC++
LDR X10, [X10]            ; ★ X10 = *指针 = 解引用拿到实际值
LDR X12, [X9,#0x28]       ; 下一个 opcode 索引
ADD X10, X10, W13,SXTH   ; X10 = 值 + sign_extend_16(偏移)
LDR X12, [X24,X12,LSL#3] ; handler 地址
STR X10, [X23,X11,LSL#3] ; ★ regs[dst] = 结果
BR X12                    ; 跳

```

伪代码：

```
regs[dst] = *insn.metadata_ptr + (int16_t)offset;
PC++;

```

这条指令的特殊之处：它从 `[X0, #8]` 取了一个指针，解引用后加偏移，结果存到目标寄存器。数据源是编译时就确定的地址，硬编码在字节码元数据里。

#### 实例分析：复合指令 vm_op_store_store_mov

地址 `0x2C06F8`，一次执行 3 条微操作，PC += 4。

```
vm_op_store_store_mov:
LDP X13, X8, [X0,#8]      ; X13=insn_slots[1], X8=insn_slots[2]
; ... 解码第一条 STORE ...
STR X11, [X8,X10]         ; ★ 微操作1: *(regs[base] + offset) = regs[value]

; ... 解码第二条 STORE ...
STR X11, [X8,X10]         ; ★ 微操作2: *(regs[base] + offset) = regs[value]

; ... 解码第三条 MOV_REG ...
AND X11, X8, #0xFF        ; src 寄存器索引
UBFX X8, X8, #0x10, #8   ; dst 寄存器索引（注意：BYTE2 不是 BYTE1！）
LDR X11, [X23,X11,LSL#3] ; regs[src]
STR X11, [X23,X8,LSL#3]  ; ★ 微操作3: regs[dst] = regs[src]
BR X10

```

伪代码：

```
*(regs[base1] + offset1) = regs[val1];   // STORE
*(regs[base2] + offset2) = regs[val2];   // STORE
regs[dst] = regs[src];                    // MOV_REG
PC += 4;

```

复合指令就是把高频出现的操作组合打包成一个 handler，减少 dispatch 开销。这三步组合起来的典型场景：往结构体连续写两个字段，然后把指针复制一份。

注意 MOV_REG 的编码格式和 STORE 不一样——它用 BYTE0 和 BYTE2 作为两个寄存器索引，跳过了 BYTE1。**VM 的指令编码不是统一格式**，不同操作类型对 4 字节的拆法不一样。

* * *

### 关于 VMP 的一些感悟

分析到这里，说实话 VMP 也没那么玄乎。VM 本质上就是个解释器，它必须规规矩矩地走 fetch-decode-execute 循环。再怎么混淆，底层逻辑不可能乱来——寄存器得有固定的存取方式，指令编码得有确定的格式，handler 之间的跳转得有可预测的机制。

这个 VM 的设计其实很教科书：ctx 结构体布局固定、寄存器存取就一个公式 `slot*8+0x6050`、每个 handler 末尾统一从 `metadata[0x28]` 取下一个 opcode 索引然后 `BR`。一旦你把这几个约定摸出来，所有 handler 都是同一个模板套出来的。

VMP 的 "难" 从来不在单个组件的复杂度，而在于：

1.  **体力活**——85 个 CF handler + 800 个 opcode handler，一个个看
2.  **间接性**——静态看不到执行顺序，必须靠 trace
3.  **规模**——五百万行 trace 里找有意义的模式

但结构本身确实没什么玄学。这大概率也不是 LLVM pass 自动生成的，LLVM pass 生成不出来这么细节的汇编——VM 引擎本身是正常 C++ 写的，用常规编译器编译。签名算法才是被编译成字节码跑在 VM 上的。说白了就是个自研语言虚拟机，和 JVM、Lua VM 是同一类东西，只不过目的是保护代码而不是跨平台。

* * *

### 当前进度

<table><thead><tr><th>内容</th><th>状态</th></tr></thead><tbody><tr><td>dispatch loop 结构</td><td>已理解，threaded dispatch</td></tr><tr><td>全局寄存器约定</td><td>已确认（X19/X20/X23/X24/X25/X26）</td></tr><tr><td>指令元数据结构</td><td>48 字节，操作数 + 数据指针 + next_opcode</td></tr><tr><td>操作数编码</td><td>非统一格式，不同指令拆法不同</td></tr><tr><td>vm_op_store</td><td>已分析</td></tr><tr><td>vm_op_load_indirect_imm</td><td>已分析</td></tr><tr><td>vm_op_add_imm_store_store</td><td>已分析（复合）</td></tr><tr><td>vm_op_store_store_mov</td><td>已分析（复合）</td></tr><tr><td>剩余 ~800 个 handler</td><td>慢慢磨...</td></tr></tbody></table>

下一步就是从 trace 里提取实际的 handler 调用序列，看签名算法到底走了哪些 opcode、调了哪些 CF。不用把 800 个全分析完，只要把 trace 里实际执行到的那些搞清楚就行。

* * *

以上就是目前的进度，整体来说还在摸索阶段，很多地方的理解可能不够准确。后续如果发现之前的结论有问题会回来修正。如果有大佬看出哪里分析得不对，欢迎留言指出，感谢。

![](https://avatar.52pojie.cn/data/avatar/000/16/93/30_avatar_middle.jpg)smile1110 前排支持 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 先赞后看![](https://static.52pojie.cn/static/image/smiley/default/42.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Manson9527 有用![](https://static.52pojie.cn/static/image/smiley/default/42.gif)受教了