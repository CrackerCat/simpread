> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2109303-1-1.html)

> [md]# 抖音 VMP 分析（四）：指令流的分析上一篇把 dispatch loop 和 handler 模板搞清楚了，按理说下一步应该是从 trace 里提取执行序列。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)rsds0duck

抖音 VMP 分析（四）：指令流的分析
-------------------

上一篇把 dispatch loop 和 handler 模板搞清楚了，按理说下一步应该是从 trace 里提取执行序列。但我发现一个问题：trace 里看到的是 handler 在跑，但我不知道它跑的是哪条字节码指令。handler 从 `[X0]` 取操作数，这个 X0 指向的是运行时的 slot 结构——那 slot 里的数据是从哪来的？

答案当然是字节码文件。但从加密的 `.rodata` 数据到 handler 手里的操作数，中间经过了好几层变换。这篇就是把这条链路完整拆出来：找到字节码、解密它、理解它的格式、搞清楚它怎么变成 handler 能执行的东西。

* * *

### 字节码在哪：从 vm_bytecode_init 顺藤摸瓜

第二篇分析 `vm_bytecode_init`（`0x29CA50`）的时候已经看到了，它调用 `sub_2AD3E0` 构建 VM 程序：

```
program = sub_2AD3E0(
  &unk_387D30,   // 加密字节码
  0x30CD,        // 大小：12493 字节
  ...
);

```

第一个参数 `0x387D30` 就是字节码在 SO 文件里的位置，在 `.rodata` 段。IDA 里点进去看，全是高熵乱码：

```
.rodata:387D30  89 A6 96 76 8F C5 7E DC  F2 67 72 73 73 F3 E4 93
.rodata:387D40  F4 8D 8D 8D 8D 8D 8C 8D  F3 93 F1 8D 8D F2 8C 93

```

显然是加密的。第二篇已经提过解密是 XOR，key 是 `0xF3`。但当时只是一笔带过，这次把完整的解密逻辑拆清楚。

* * *

### 解密：XOR Key 怎么来的

`sub_2AD3E0` 是个薄 wrapper，真正干活的是 `sub_2CFBC8`（`vm_program_build_core`）。

#### Key 的动态派生

解密 key 不是硬编码的，而是从入口点描述表里动态算出来的：

```
// 0x2CFC48 ~ 0x2CFC58
index = bytecode_size % entry_desc_count;   // 12493 % 3 = 1
key = entry_desc_table[index].byte[2];      // entry[1][2] = 0xF3

```

对应的 ARM64：

```
UDIV  X8, X3, X27          ; X3=bytecode_size, X27=entry_count(3)
MOV   W10, #0x18           ; 每个 entry 24 字节
MSUB  X8, X8, X27, X3      ; X8 = size % count
MADD  X8, X8, X10, X28     ; X8 = index*24 + table_base
LDRB  W8, [X8, #2]         ; key = entry[index].byte[2]

```

入口点描述表有 3 个条目，第二篇已经列过：

<table><thead><tr><th>条目</th><th>byte[2] (XOR key)</th></tr></thead><tbody><tr><td>0</td><td>0x1B</td></tr><tr><td>1</td><td><strong>0xF3</strong> ← 当前命中</td></tr><tr><td>2</td><td>0xF5</td></tr></tbody></table>

设计意图很明显：字节码长度变了（比如新版本加了函数），key 就跟着变。简单的版本绑定。

#### NEON 向量化解密

解密不是逐字节 XOR，用的是 ARM NEON 指令集，每次处理 32 字节：

```
; 0x2CFC9C ~ 0x2CFCC4
AND   X10, X3, #0xFFFFFFFFFFFFFFE0  ; 32字节对齐的部分
DUP   V0.16B, W8                    ; V0 = 16 × 0xF3（广播到128-bit向量）

loop:
LDP   Q1, Q2, [X11, #-0x10]        ; 加载 32 字节
SUBS  X12, X12, #0x20              ; 计数器 -= 32
EOR   V1.16B, V1.16B, V0.16B       ; 前 16 字节 XOR
EOR   V2.16B, V2.16B, V0.16B       ; 后 16 字节 XOR
STP   Q1, Q2, [X11, #-0x10]        ; 写回（in-place）
ADD   X11, X11, #0x20
B.NE  loop

```

尾部不足 32 字节的先尝试 8 字节批量，最后剩余的逐字节。性能上无所谓，12KB 的数据怎么解都是瞬间的事，但这说明这个 VM 引擎是认真工程化过的，不是随手糊的。

#### 验证

Python 一行搞定：

```
enc = open('vm_encrypted_bytecode.bin', 'rb').read()
dec = bytes(b ^ 0xF3 for b in enc)
# dec[:8] == b'\x7a\x55\x65\x85\x7c\x36\x8d\x2f'  ← magic header

```

解密后前 8 字节是 `7a 55 65 85 7c 36 8d 2f`，不是标准 WASM 的 `\0asm`，是自定义的 magic。

* * *

### 字节码格式：WASM 的变体

解密后的 12493 字节是一个 TLV（Tag-LEB128Length-Value）编码的二进制格式，结构上模仿了 WebAssembly 的 section 布局。

#### 整体结构

<table><thead><tr><th>文件偏移</th><th>Tag</th><th>Section</th><th>大小</th><th>说明</th></tr></thead><tbody><tr><td>0x00</td><td>—</td><td>Header</td><td>8B</td><td>Magic: <code>7a 55 65 85 7c 36 8d 2f</code></td></tr><tr><td>0x0E</td><td>1</td><td>Type</td><td>148B</td><td>23 个函数签名</td></tr><tr><td>0xA8</td><td>2</td><td>Import</td><td>820B</td><td>97 个导入（CF + G + libc）</td></tr><tr><td>0x3E2</td><td>3</td><td>Function</td><td>3B</td><td>函数索引 → 类型映射</td></tr><tr><td>0x3EB</td><td>6</td><td>Global</td><td>55B</td><td>全局变量定义</td></tr><tr><td>0x428</td><td>7</td><td>Export</td><td>11B</td><td>2 个入口点: F0, F1</td></tr><tr><td>0x439</td><td>12</td><td>DataCount</td><td>1B</td><td>数据段计数</td></tr><tr><td><strong>0x440</strong></td><td><strong>10</strong></td><td><strong>Code</strong></td><td><strong>8007B</strong></td><td><strong>VM 指令主体</strong></td></tr><tr><td>0x238D</td><td>11</td><td>Data</td><td>434B</td><td>字符串常量池</td></tr><tr><td>0x2545</td><td>0</td><td>Custom "linking"</td><td>1133B</td><td>链接信息</td></tr><tr><td>0x29B8</td><td>0</td><td>Custom "reloc.CODE"</td><td>1813B</td><td>代码重定位表</td></tr></tbody></table>

有意思的是，它用了标准 WASM 的 section tag 编号（Type=1, Import=2, Code=10 等），LEB128 编码也是标准的。但指令集完全不是 WASM opcodes——这只是借了个容器格式。

#### Type Section：函数签名

用标准 WASM 类型编码（`0x60` = func, `0x7F` = i32, `0x7E` = i64），定义了 23 种签名：

```
type[0]: (i64, i64, i64, i64, i64, i32, i64) -> ()
type[1]: (i64, i64) -> (i32)
type[2]: (i64) -> (i32)
type[3]: (i64) -> ()
type[4]: () -> (i64)
type[5]: (i64, i64) -> ()
...

```

#### Import Section：97 个导入

分三类：

*   **CF0 ~ CF54**（85 个）：native call handler，就是第二篇说的那 85 个 CF
*   **原子操作**：`__sync_val_compare_and_swap_8`、`__sync_val_compare_and_swap_1`
*   **G0 ~ G8**（9 个）：全局变量访问器

#### Export Section：入口点

```
F0 → function index 88
F1 → function index 89

```

第二篇里 `vm_bytecode_init` 最后调用 `sub_2AD420(program, "F0")` / `"F1"` 查找的就是这两个导出名。

#### Data Section：字符串池

这里透露了 VM 的业务逻辑：

```
"167774bf518c11948aa0784351ccf5a9"   (MD5 hash)
"http", "https"                       (协议)
" %d|%s"                              (格式串)
"ML_DoHttpReqSignIT"                  (签名 API 名)
"X-METASEC-MODE"                      (HTTP Header)
"X-BD-CLIENT-KEY"
"X-BD-KMSV"

```

看到 `ML_DoHttpReqSignIT` 和那些 header 名就确认了——这个字节码就是签名算法的实现。

* * *

### Code Section：从字节到指令

这是最关键的部分。Code section 从文件偏移 `0x440` 开始，8007 字节。

#### 指令编码：固定 4 字节

**关键发现：这不是标准 WASM opcodes。** 标准 WASM 用变长指令编码，这里用的是固定 4 字节（u32 little-endian）。

每个函数体的结构：

```
[body_size: LEB128]      // 函数体总大小
[locals_size: LEB128]    // 局部变量槽数
[num_locals: LEB128]     // 局部变量声明数（均为 0）
[instructions: 4B × N]   // 固定 4 字节指令流

```

#### 两个函数的具体布局

```
func_count = 2

F0:
  body_size  = 0x1543 (5443 bytes)
  locals_size = 0x410 (1040 slots)
  code starts at file offset 0x446
  code size = 5440 bytes = 1360 instructions

F1:
  body_size  = 0x9FF (2559 bytes)
  locals_size = 0x1D0 (464 slots)
  code starts at file offset 0x198B
  code size = 2556 bytes = 639 instructions

总计: 1999 条指令

```

F1 的 `locals_size = 0x1D0`——这就是第二篇里 `vm_execute_function` 取到的栈帧大小 `0x1d0`。对上了。

#### 解析器：sub_327B08

VM 的 code section 解析器在 `sub_327B08`，逻辑很直白：

```
for (int func = 0; func < func_count; func++) {
    body_size = read_leb128(&cursor);
    locals_size = read_leb128(&cursor);
    num_locals = read_leb128(&cursor);

    while (cursor < body_end) {
        uint32_t insn_word = *(uint32_t*)(bytecode_base + cursor);
        cursor += 4;
        emit_to_slot(insn_word);  // 存入 0x0C 字节的紧凑 slot
    }
}

```

注意：**读取阶段不做任何变换**。文件里的原始 4 字节值直接进入紧凑 slot。变换发生在后面。

* * *

### 从文件到执行：三阶段流水线

一条 VM 指令从文件中的 4 字节到最终被 handler 执行，经历三个阶段：

```
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: 解析 (sub_327B08)                                       │
│   bytecode_file[offset] → compact_slot (0x0C bytes)             │
│   直接复制，无变换                                                │
├─────────────────────────────────────────────────────────────────┤
│ Stage 2: 展开 (sub_2D24F4 — 9KB 混淆函数)                        │
│   compact_slot → runtime_slot (0x30 bytes)                      │
│   包含: 位域置换 + handler_index 查表                             │
├─────────────────────────────────────────────────────────────────┤
│ Stage 3: 执行 (dispatch loop @ 0x2AF5CC)                         │
│   runtime_slot.handler_index → handler_table[index] → BR        │
│   handler 从 runtime_slot.original_word 解码操作数                │
└─────────────────────────────────────────────────────────────────┘

```

第三篇分析的 handler 模板——从 `[X0]` 取操作数、从 `[X0, #0x28]` 取 opcode 索引——对应的就是 Stage 3 从 runtime_slot 里读数据。

* * *

### Stage 2：位域置换系统

Stage 2 是整个链路里最恶心的部分。`sub_2D24F4` 是一个 9KB 的混淆函数，负责把 compact_slot 展开成 runtime_slot。核心变换在 `sub_2DF578`：

```
transformed_word = sub_2DF578(descriptor, original_word);

```

#### 三种指令格式

变换由 3 个编码描述符控制，对应三种格式：

<table><thead><tr><th>格式</th><th>位域拆分</th><th>用途</th></tr></thead><tbody><tr><td>R</td><td>6+5+5+5+5+6 = 32 bits</td><td>寄存器操作</td></tr><tr><td>I</td><td>6+5+5+16 = 32 bits</td><td>立即数操作</td></tr><tr><td>J</td><td>6+26 = 32 bits</td><td>跳转</td></tr></tbody></table>

看着眼熟吧？这就是 MIPS 的三种指令格式。R/I/J，经典设计。

#### 上下文相关：状态机

这里有个坑——每条指令用哪个描述符，取决于**前一条指令**的变换结果：

```
descriptor_index = previous_transformed_word % 3;

```

意味着你不能单独解码一条指令，必须从函数开头顺序解码。这是一种简单但有效的反静态分析手段：你不能随便跳到中间某条指令开始反汇编，因为你不知道当前的描述符状态。

#### Handler Index 的确定

变换后的 `transformed_word` 决定了 handler_index。通过实验验证：

*   只用部分字段 → 存在冲突
*   用全部 6 个字段 → **0 冲突**

结论：handler_index 本质上是一个以 transformed_word 为 key 的查找表，嵌入在那个 9KB 混淆函数里。

* * *

### Runtime Slot 布局

最终每条指令在内存中占 0x30（48）字节——这就是第三篇里说的 "48 字节元数据结构"：

```
偏移    大小    内容
0x00    4B     original_word（handler 用来解码操作数的原始指令字）
0x08    8B     数据指针（部分指令使用，指向全局变量/常量）
0x10    8B     数据指针（compound handler 的子指令）
0x28    8B     handler_index（dispatch 查表用）

```

第三篇里 handler 从 `[X0]` 取的 4 字节操作数，就是这里偏移 0x00 的 `original_word`。从 `[X0, #0x28]` 取的 opcode 索引，就是 `handler_index`。全对上了。

* * *

### 验证：字节码文件就是 VM 执行的指令

分析到这一步有个关键问题：我从 SO 文件里提取并解密的字节码，**真的就是 VM 运行时执行的指令吗？** 万一中间还有什么运行时的二次变换呢？

#### 方法

用 unidbg 模拟执行 SO，在 slot 展开完成后 dump 所有 runtime slot 中的 `original_word`，与字节码文件中的对应偏移逐一比对。

写了个 `DumpVmSlots.java`，hook slot 展开过程，提取每条指令的 original_word、transformed_word、handler_index，输出 CSV：

```
func_id,slot_index,original_word,transformed_word,hi6,lo6,handler_index
1,0,0x3001ea80,0x500001bd,20,61,0x27b
1,1,0xc04383bf,0x7421fe30,29,48,0x42
...

```

#### 比对结果

```
import struct, csv

dec = open('vm_decrypted_bytecode.bin', 'rb').read()

with open('vm_slot_dump.csv') as f:
    for row in csv.DictReader(f):
        func_id = int(row['func_id'])
        idx = int(row['slot_index'])
        expected = int(row['original_word'], 16)

        if func_id == 1:  # F0
            actual = struct.unpack_from('<I', dec, 0x446 + idx*4)[0]
        else:             # F1
            actual = struct.unpack_from('<I', dec, 0x198B + idx*4)[0]

        assert actual == expected

# 1999/1999 全部dd匹配，零偏差

```

**1999 条指令，100% 匹配。** 字节码文件中的每一个 4 字节值，都精确对应 VM 运行时 slot 中的 `original_word`。没有运行时二次变换，Stage 1 确实是直接复制。

* * *

### 后续发现：不止两个函数

验证完字节码文件的正确性之后，用 unidbg 做了动态 trace（`TraceVmExec.java`），发现一个重要的事实：**VM 不止 F0 和 F1 两个函数。**

sign 调用涉及数十个 VM 函数，pc_offset 分布在 `0x6`、`0x967`、`0xc3a`、`0x14bd`、`0x1535`、`0x16bd`、`0xc8cd`、`0x863f`、`0x10107`、`0x146e2` 等位置。递归深度最深达 6 层。50000 条指令中有 290 次函数进入、959 次 CALL。

这意味着 12493 字节的字节码文件里不只有 F0 和 F1 的代码——那些更大的 pc_offset（比如 `0x10107`）远超 F0+F1 的范围（F0 1360 条 + F1 639 条 = 1999 条 × 4 字节 = 7996 字节）。要么是 CF handler 内部又触发了新的字节码加载，要么是我对 Code section 的解析还不完整。

这个留到后面再追。

* * *

### 总结

<table><thead><tr><th>步骤</th><th>关键地址 / 文件</th><th>产出</th></tr></thead><tbody><tr><td>定位字节码</td><td><code>0x387D30</code> (SO 偏移)</td><td>12493 字节加密数据</td></tr><tr><td>确定 XOR key</td><td><code>0x2CFC48</code> (key 计算)</td><td>key = 0xF3 (由 size%3 派生)</td></tr><tr><td>解密</td><td><code>0x2CFC9C</code> (NEON loop)</td><td>12493 字节明文</td></tr><tr><td>解析格式</td><td>Magic <code>7a556585...</code></td><td>WASM 变体，TLV 编码</td></tr><tr><td>提取指令</td><td>Code section @ 0x440</td><td>F0: 1360 条, F1: 639 条</td></tr><tr><td>位域置换</td><td>sub_2DF578</td><td>R/I/J 三格式，上下文相关</td></tr><tr><td>验证</td><td>unidbg slot dump</td><td>1999/1999 匹配</td></tr></tbody></table>

整个链路的核心思路：**先动态确认行为（unidbg dump），再静态还原结构（格式解析），最后交叉验证（逐字节比对）**。

下一步是把位域置换和 handler 映射完整实现成反汇编器——把 1999 条 4 字节的 raw word 翻译成人能读的指令。那个 9KB 的混淆函数 `sub_2D24F4` 得硬啃，但至少现在知道它的输入输出是什么了。

追到现在我也是有点懵逼了，从入口追到 dispatch loop 还算有章法，但到了位域置换、上下文状态机、9KB 混淆函数这一层，复杂度确实上了一个台阶。这东西不太适合新手直接硬啃——我自己也是反复对照 unidbg dump 才把链路理顺的。先这样吧，等我把反汇编器完整实现、handler 语义全部还原之后，再写后续的 VM 反汇编方面，算法还原还不知道啥时候能搞定。

![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)Hmily 文章不长，建议合并到之前帖子中，方便阅读和加分，如何？ ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 讲的很好！！！但是我看不懂！![](https://static.52pojie.cn/static/image/smiley/default/17.gif)