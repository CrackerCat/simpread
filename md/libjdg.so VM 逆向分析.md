> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2110046-1-1.html)

> [md]> ** 作者：** 人生导师 > ** 日期：** 2025 年 5 月 28 日结论先说：京东 `libjdg.so` 内部有一套寄存器 VM，保护签名算法和反调试逻辑。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)rsds0duck

> **作者：** 人生导师  
> **日期：** 2025 年 5 月 28 日

结论先说：京东 `libjdg.so` 内部有一套寄存器 VM，保护签名算法和反调试逻辑。最终还原结果——`b4 = Base64(RC4(a11 XOR timestamp, zlib.compress(json_plaintext)))`，标准算法组合没有魔改。这篇从 VM 入口怎么找、上下文初始化、dispatch 机制、handler 逆向、反汇编器编写，一直写到签名算法完整还原。整个 VM 框架代码是干净的（没混淆），真正的保护在解释器的花指令和私有字节码上。

环境：ARM64，Frida trace 真机执行日志，IDA Pro 静态辅助。

* * *

### 找 VM 入口

京东的混淆跟抖音不一样。抖音是 OLLVM 控制流平坦化，你可以硬追；京东是花指令，把函数拆成三块：

```
初始化构造 → 一个短小的汇编块，初始化后 BR 跳转到函数体
函数体     → 执行逻辑，完事后 BR 跳转到返回
函数返回   → 通用收尾模板

```

同一批函数（比如开一样大小栈的）共享初始化块和返回块，然后互相跳转。效果就是 IDA 的 CFG 直接报废，伪 C 看不了。

去花这块我感觉没必要，难度 8/10，但跟 trace 走就行了。

那怎么找入口？我的思路是：**总不能全部都是花指令吧，怎么也得有一些干净的函数**。然后我就在完整 trace 里搜，找到了一个能看伪 C 的函数。找到以后用 IDA 反向引用追上层调用，不管调用者是不是被花出来的函数，一层层往上追到最顶点，就找到了 VM 入口。

最后确认方法更朴实——直接在日志里搜 `vm_interpreter` 的地址（0xE1708），搜到 4 次调用，对应 4 个 VM 程序。

* * *

### VM 整体架构

找到入口以后，整个执行流程就清楚了：

```
vm_ctx_init() → vm_stack_push() × N → vm_interpreter() → vm_ctx_destroy()

```

三个 VM 程序共用这套框架：

<table><thead><tr><th>属性</th><th>vm_exec_sign_basic</th><th>vm_exec_sign_jni</th><th>vm_exec_anti_frida</th></tr></thead><tbody><tr><td>地址</td><td>0x875C0</td><td>0x876F4</td><td>0x87834</td></tr><tr><td>参数数量</td><td>8</td><td>9</td><td>10</td></tr><tr><td>字节码大小</td><td>4206B</td><td>10266B</td><td>6654B</td></tr><tr><td>数据段大小</td><td>54B</td><td>179B</td><td>476B</td></tr><tr><td>handler 数量</td><td>16</td><td>50</td><td>65</td></tr></tbody></table>

调用模式完全一致：

```
unsigned int vm_exec_XXX(args...) {
    vm_ctx *ctx = vm_ctx_init(ip_slot, stack_size, code_size,
                              b64_bytecode, data_size, b64_data);
    if (!ctx) return 9;

    if (vm_stack_push(ctx, arg1) || vm_stack_push(ctx, arg2) || ...) {
        result = 11;  // 压栈失败
    } else {
        result = vm_interpreter(handler_table, opcode_count, ctx);
    }

    vm_ctx_destroy(ctx);
    return result;
}

```

以 `vm_exec_sign_basic` 为例，IDA 里长这样：

```
ctx_init = vm_ctx_init(
    8,                    // ip_slot
    312,                  // stack_size
    0x106Eu,              // code_size (4206 字节)
    aPt09puiaeqbzah,      // base64 编码的字节码
    0x36u,                // data_size (54 字节)
    "PT09PQAAAAAlc18lcwAwADMuMAAlbGxkAGIxAGIyAGIzAGI0AGI1AGI3AGI2AAAAAAAAAAAA"
);

```

* * *

### vm_ctx_init：上下文初始化

先看参数怎么传进来的。ARM64 调用约定 x0-x7 是参数，trace 直接告诉你值：

```
mov w26, w1 → w1=0x138  (stack_size = 312)
mov w20, w0 → w0=0x8    (ip_slot = 8)
mov w21, w2 → w2=0x106e (code_size = 4206)
mov x22, x3 → x3=0x749f877d80 (b64_bytecode 指针)
mov w23, w4 → w4=0x36   (data_size = 54)
mov x24, x5 → x5=0x749f879370 (b64_data 指针)

```

这些 mov 是编译器把参数保存到 callee-saved 寄存器，trace 直接给了值，不需要猜。

函数内部做的事情：

```
vm_ctx* vm_ctx_init(int ip_slot, int stack_size, uint32_t code_size,
                    uint8_t *b64_bytecode, uint32_t data_size, uint8_t *b64_data) {
    vm_ctx *ctx = calloc(1, 0x38);  // 分配 56 字节的上下文结构体
    if (!ctx) return NULL;

    // 分配三块缓冲区，每块多给 256 字节余量
    ctx->code_buf_capacity = code_size + 256;
    ctx->code_buf = calloc(1, ctx->code_buf_capacity);

    ctx->data_buf_capacity = data_size + 256;
    ctx->data_buf = calloc(1, ctx->data_buf_capacity);

    ctx->stack_capacity = stack_size + 256;
    ctx->stack_buf = calloc(1, ctx->stack_capacity);

    // base64 解码并填充
    uint8_t *dec = base64_decode(b64_data);
    memcpy(ctx->data_buf, dec, data_size); free(dec);

    dec = base64_decode(b64_bytecode);
    memcpy(ctx->code_buf, dec, code_size); free(dec);

    // 设置帧基址和栈顶
    ctx->frame_base = ctx->stack_top = ctx->stack_buf + 8 * ip_slot;
    return ctx;
}

```

calloc(1, 0x38) 分配 56 字节，返回值就是整个 VM 上下文。分配失败直接全剧终，但我们知道肯定成功。

怎么确定各字段的含义？看 trace 里的实际操作：

1.  0x116e = 0x106e + 0x100，而 0x106e 是参数 code_size → 这个字段是 code_buf_capacity
2.  紧接着 calloc(1, 0x116e) 的返回值存到 [x19, #0x20] → 这是 code_buf
3.  后面 base64_decode(b64_bytecode) 的结果被 memcpy 到这块内存 → 确认是字节码缓冲区

* * *

### vm_ctx 结构体

根据 trace 中寄存器状态和内存读写，构造出完整结构体：

```
struct vm_ctx {
    uint32_t data_buf_capacity;   // +0x00
    uint32_t stack_capacity;      // +0x04
    uint32_t code_buf_capacity;   // +0x08
    uint32_t _pad;                // +0x0C (对齐填充，trace 验证后续未使用)
    void    *data_buf;            // +0x10: 常量池
    void    *stack_buf;           // +0x18: 栈缓冲区基址
    void    *code_buf;            // +0x20: 字节码
    void    *frame_base;          // +0x28: 寄存器文件基址 (固定不动)
    void    *stack_top;           // +0x30: 栈顶指针 (push 时递减)
};

```

总计 56 字节，占满分配的内存。padding 的判定依据：看 trace 里寄存器是 w1（32 位）还是 x1（64 位），以及后续是否有读写操作。我在 trace 里验证了这段内存后续始终未被使用，所以判定为填充。

* * *

### 内存布局

```
stack_buf (低地址)                    ← 下溢边界
  │
  │  [push 的参数, 向下增长]
  │     arg7  (frame_base - 0x40)    ← 第 8 个 push
  │     ...
  │     arg0  (frame_base - 0x08)    ← 第 1 个 push
  │
  ├── frame_base                     ← 寄存器寻址锚点 (= stack_buf + 8*ip_slot)
  │
  │  [局部变量, 向上使用]
  │     local_0  (frame_base + 0x08)
  │     local_1  (frame_base + 0x10)
  │     ...
  │
  └── stack_buf + stack_capacity     ← 缓冲区结束

```

这里有个坑——我一开始不知道 `frame_base` 和 `stack_top` 是什么，只看初始化的话它们是同一个值。得看后续 handler 怎么使用才能确定语义。

* * *

### 字段语义推断

#### code_buf：字节码

从 trace seq 13dde-13df2（解释器入口）：

```
ldr x8, [ctx, #0x20]     ← 加载 code_buf 指针
ldrh w23, [x8, #4]       ← 读 header 中的 opcode_key = 0x42
ldrh w9, [x8, #8]        ← 读第一条指令的 raw opcode = 0x273
eor w9, w9, w23          ← XOR 解密: 0x273 ^ 0x42 = 0x231 (真实 opcode)
ldrh w24, [x8, #6]       ← 读 operand_key = 0x11
add x21, x8, #0xa        ← x21 = code_buf + 10 = 第一条指令的操作数起始
br x9                    ← 跳转到 handler

```

使用方式：从头部读解密 key → 顺序取指 → 解密 opcode → 查跳转表 → 执行 handler → handler 完成后 IP 前进到下一条。这就是 CPU 读取指令的方式，所以叫 code。

#### data_buf：常量池

从 trace seq 1534d（handler 0xEB3E0）：

```
add x8, x27, x8    ← x27=data_buf, x8=偏移量(0x8)
str x8, [x26, x9]  ← 把 data_buf+offset 的地址存到寄存器文件

```

使用方式：指令中编码一个偏移量 → handler 计算 data_buf + offset → 得到指向常量的指针 → 存入 VM 寄存器供后续使用。这就是程序访问 .rodata/.data 段的方式，所以叫 data。

#### frame_base 和 stack_top

这两个字段的语义是通过分析 `vm_stack_push` 才确定的。一开始我被传统名词卡住了——"ip" 和 "sp" 这两个名字让我往指令指针和栈指针上想，但实际上这个 VM 没有把 PC 存在 ctx 里（PC 只存在于 native 寄存器 x21 中）。

想通以后就清楚了：

*   `frame_base`：寄存器文件基址，初始化后固定不动，解释器加载到 x26，正偏移是局部变量，负偏移是参数
*   `stack_top`：栈顶指针，push 时递减，是唯一在初始化后被修改的 ctx 字段

* * *

### vm_stack_push：压入参数

看 0xED52C 这个函数的完整逻辑（从 trace 直接读出）：

```
ldr x9, [x0, #0x30]   ← 读取 ctx->stack_top (当前值 = 0x764b4382b0)
ldr x10, [x0, #0x18]  ← 读取 ctx->stack_buf (基址 = 0x764b438270)
sub x9, x9, #8        ← sp -= 8 (新 sp = 0x764b4382a8)
cmp x9, x10           ← 新 sp < 基址? (溢出检查)
b.hs                   → 不跳转则返回错误
str x9, [x8, #0x30]   ← 写回 ctx->stack_top = 新值
str x1, [x9]          ← *sp = 参数值 (push 的数据)
ret

```

翻译成 C：

```
int vm_stack_push(vm_ctx *ctx, int64_t value) {
    int64_t *new_top = (int64_t*)ctx->stack_top - 1;  // stack_top -= 8
    if (new_top < ctx->stack_buf)
        return 11;  // 栈溢出
    ctx->stack_top = new_top;
    *new_top = value;
    return 0;
}

```

从连续调用看 sp 的变化：

```
第 1 次 push: sp = 0x764b4382b0 → 0x764b4382a8 (-8)
第 2 次 push: sp = 0x764b4382a8 → 0x764b4382a0 (-8)
第 3 次 push: sp = 0x764b4382a0 → 0x764b438298 (-8)

```

三个判定依据：

1.  **指针递减写入** — 每次 sp 减 8，数据写到新 sp 位置，经典 push 操作
2.  **边界检查是下溢检查** — cmp new_sp, base_addr; b.hs，检查 sp 是否低于 buffer 起始地址，普通数组不会这样检查
3.  **frame_base 和 stack_top 初始化到同一位置** — push 的参数在 stack_top 下方增长，而 frame_base 标记了 "参数区和局部变量区的分界线"，这是寄存器 VM 的典型栈帧设计

* * *

### 混淆策略总结

<table><thead><tr><th>层级</th><th>混淆方式</th><th>效果</th></tr></thead><tbody><tr><td>VM 框架代码</td><td><strong>无混淆</strong></td><td>vm_ctx_init、base64_decode 等均为干净代码</td></tr><tr><td>VM 解释器</td><td><strong>花指令</strong></td><td>IDA 无法反编译，CFG 重建失败</td></tr><tr><td>Bytecode</td><td><strong>私有指令集 + 变长指令</strong></td><td>静态分析无法直接读取算法逻辑</td></tr><tr><td>Opcode 编码</td><td><strong>固定 XOR</strong></td><td>opcode ^ 0x42, operand ^ 0x11，key 明文存于 header</td></tr></tbody></table>

真正的保护不在 XOR 加密（那只是薄纱），而在于：解释器花指令让你看不懂 handler 逻辑、私有 opcode 让你解密了也读不懂、变长指令让你不知道边界在哪、间接跳转让静态分析追不了控制流。

绕过方法：Frida trace 记录实际执行路径，花指令不会被执行。

* * *

### 解释器 Dispatch：Threaded Interpretation

前置初始化搞完了，接下来看 `vm_interpreter`（0xE1708）怎么执行字节码。

这个函数本身被花指令搞得 IDA 完全报废，但 trace 里每一条都是真实执行的，花指令不会被执行。从 trace seq 0x13DD3 开始还原：

```
; 函数序言
stp x28, x27, [sp, #-0x60]!     ; 开栈帧，保存 callee-saved
...
; 读 bytecode header
ldr x8, [x2, #0x20]             ; x8 = code_buf
ldrh w23, [x8, #4]              ; w23 = opcode_key = 0x42
ldrh w9, [x8, #8]               ; w9 = first_opcode(加密) = 0x273
eor w9, w9, w23                  ; 解密: 0x273 ^ 0x42 = 0x231
cmp w10, #0x23a                  ; 边界检查 (最多 571 种 opcode)
; 查跳转表
ldr x9, [x20, x9, lsl #3]       ; jump_table[0x231] = 0xB5B4
; 加载上下文
ldrh w24, [x8, #6]              ; w24 = operand_key = 0x11
ldr x26, [x2, #0x28]            ; x26 = 寄存器文件基址
ldr x27, [x2, #0x10]            ; x27 = data_buf
add x21, x8, #0xa               ; x21 = IP (跳过 10 字节 header)
; 跳转
add x9, x28, x9                 ; handler_addr = code_base + offset
br x9                            ; → alloca handler

```

关键发现：这是 **threaded interpretation**。每个 handler 执行完自身逻辑后，不返回主循环，而是直接 dispatch 下一条指令：

```
; handler 尾部 (以 alloca 为例)
ldrh w8, [x21, #8]              ; 读当前指令末尾 2 字节 = 下一条 opcode(加密)
eor w8, w8, w23                  ; XOR 解密
ldr x8, [x20, x8, lsl #3]       ; 查跳转表
add x21, x21, #0xa              ; IP += 10 (当前指令长度)
add x8, x28, x8                 ; handler_addr = code_base + offset
br x8                            ; 跳转到下一个 handler

```

没有中央 dispatch loop，handler 之间直接跳。这意味着你在 IDA 里看不到任何函数边界——整个解释器就是一坨互相 BR 的代码块。

寄存器约定（全程不变）：

<table><thead><tr><th>寄存器</th><th>用途</th></tr></thead><tbody><tr><td>x20</td><td>jump_table 基址</td></tr><tr><td>x21</td><td>bytecode IP（每条指令更新）</td></tr><tr><td>w23</td><td>opcode_key = 0x42</td></tr><tr><td>w24</td><td>operand_key = 0x11</td></tr><tr><td>x25</td><td>外部函数表指针 (off_11A7F0)</td></tr><tr><td>x26</td><td>寄存器文件基址 (frame_base)</td></tr><tr><td>x27</td><td>data_buf 常量池基址</td></tr><tr><td>x28</td><td>code_base (0xE2788)</td></tr></tbody></table>

* * *

### Bytecode 编码：固定 XOR 薄纱

Bytecode 的 "加密" 是固定 XOR，key 明文存在 header 里。所有 VM 程序共用同一对 key。

```
Header (10 bytes):
  [0:4]  magic "====" (0x3D3D3D3D)
  [4:6]  opcode_key = 0x0042
  [6:8]  operand_key = 0x0011
  [8:10] first_opcode (加密)

```

解密规则：

*   opcode: `real = encoded ^ 0x42`
*   operand (寄存器偏移): `real = encoded ^ 0x11`
*   立即数 / 大小字段: 不加密，直接读

这里有个坑——opcode 不在当前指令体内，而是由前一条指令的尾部携带。每条指令的最后 2 字节是下一条的加密 opcode。第一条的 opcode 来自 header[8:10]。

```
Header: [magic:4][opcode_key:2][operand_key:2][first_opcode_enc:2]
Insn 0: [operands (L-2 bytes)][next_opcode_enc:2]
Insn 1: [operands (L-2 bytes)][next_opcode_enc:2]
...

```

指令是变长的，长度由 opcode 类型决定。从 handler 内部的 `ADD X21, X21, #N` 可以确定每种 opcode 的指令长度。

为什么说这不是真正的加密：key 自包含在 header 里，固定 XOR 没有任何密码学强度，没有完整性校验，所有程序共用 key。真正的保护在于：你解密了 bytecode 也读不懂——私有 opcode 语义不公开，变长指令边界不确定，handler 逻辑被花指令保护。

* * *

### 写反汇编器：从跳转表到完整 Disasm

知道了编码规则，下一步是写反汇编器。但有个前提——你得知道每个 opcode 对应哪个 handler、指令多长。

#### 提取跳转表

跳转表在 SO 偏移 0x7F0C0，共 571 条，每条 8 字节。用 Frida 在运行时 dump：

```
// dump_jump_table.js (简化)
var base = Module.findBaseAddress("libjdg.so");
var table = base.add(0x7F0C0);
for (var i = 0; i < 571; i++) {
    var offset = table.add(i * 8).readS64();
    var handler = 0xE2788 + offset;
    // 从 handler 内部找 ADD X21, X21, #N 确定指令长度
}

```

571 个 opcode 映射到 530 个 unique handler。指令长度分布：

<table><thead><tr><th>长度</th><th>数量</th><th>说明</th></tr></thead><tbody><tr><td>6B</td><td>75</td><td>双操作数 (load/store/mov)</td></tr><tr><td>8B</td><td>174</td><td>三操作数 (算术运算)</td></tr><tr><td>10B</td><td>49</td><td>带 32-bit 立即数 (alloca/cmp)</td></tr><tr><td>12B</td><td>52</td><td>带 64-bit 立即数 (ld_imm64/ld_data)</td></tr><tr><td>14B</td><td>66</td><td>带 64-bit 立即数 + 额外操作数 (add_imm/cmp_eq)</td></tr><tr><td>16-24B</td><td>31</td><td>复杂指令</td></tr><tr><td>var</td><td>80</td><td>分支 / 调用 (长度由参数数量决定)</td></tr></tbody></table>

#### Handler 逆向

每个 handler 的逆向方法：在 IDA 里看汇编（花指令只在 dispatch 入口，handler 内部大多是干净的），结合 trace 验证。

以 `store_ptr`（0xEB208）为例：

```
ldrh w8, [x21]          ; op0 (加密)
ldrh w9, [x21, #2]      ; op1 (加密)
eor w8, w8, w24          ; 解密 op0 → 目标地址寄存器
eor w9, w9, w24          ; 解密 op1 → 源值寄存器
sxth x8, w8             ; 符号扩展
sxth x9, w9
ldr x10, [x26, x8]      ; x10 = reg[op0] (目标地址)
ldr x11, [x26, x9]      ; x11 = reg[op1] (源值)
str x11, [x10]           ; *reg[op0] = reg[op1]
; dispatch next...

```

语义：`*reg[dst] = reg[src]`，6 字节指令。

用这个方法逐个逆向，sign_basic 用到 20 个 handler，sign_jni 额外多 34 个（总共 53 个 unique handler）。全部确认语义后，反汇编器就能输出可读的伪代码了。

#### 反汇编器架构

```
# bytecode_decoder.py 核心逻辑
def disassemble(raw_bytes, handler_registry):
    opcode_key = struct.unpack_from('<H', raw_bytes, 4)[0]
    operand_key = struct.unpack_from('<H', raw_bytes, 6)[0]
    current_opcode = struct.unpack_from('<H', raw_bytes, 8)[0] ^ opcode_key
    pos = 0x0A

    while pos < len(raw_bytes):
        handler = handler_registry[current_opcode]
        length = handler.insn_length
        body = raw_bytes[pos:pos+length]
        # 调用 handler 的 decode 方法输出可读文本
        text = handler.decode(body, operand_key)
        # 尾部 2 字节 → 下一条 opcode
        next_enc = struct.unpack_from('<H', body, length-2)[0]
        current_opcode = next_enc ^ opcode_key
        pos += length

```

每个 handler 注册自己的解码逻辑。最终 sign_basic 输出 463 条指令、sign_jni 输出 905 条指令，0 未知 opcode。

* * *

### sign_basic：编排层

sign_basic 是最外层的 VM 程序，它不做加密计算，只负责编排。反汇编后看到的逻辑：

1.  从输入结构体取字段（functionId、UUID、时间戳等）
2.  格式化时间戳 → b7
3.  调用 sign_jni（通过 `call_native(7)`）→ 得到 b4
4.  调用 anti_frida（通过 `call_native(9)`）→ 反调试检测
5.  内部计算 b5、b6（HMAC-SHA256，handler 内联实现）
6.  组装最终 JSON 输出

b5/b6 的计算不走 native 函数表，而是由 VM handler 内部直接实现 SHA256 和 MD5：

```
b5 = hex(HMAC-SHA256(hmac_key, MD5(signing_string)))
b6 = hex(HMAC-SHA256(hmac_key, MD5(signing_string + b5)))

```

这些加密原语作为独立 handler 存在于解释器内部，不经过 FFI 调用桥。

* * *

### sign_jni：核心签名算法还原

sign_jni 是真正干活的。10266 字节 bytecode，905 条指令，53 个 handler。反汇编输出直接可读，整个算法分 8 个阶段。

#### 输入参数

```
arg1: 输出缓冲区指针 (ret_ptr)
arg2: 输出长度指针 (ret_len_ptr)
arg3: 签名上下文结构体 (含 functionId 等)
arg4: 时间戳 (8字节 LE, 毫秒)
arg5: 原子变量指针 (计数器)
arg6: JNIEnv*
arg7: 设备信息结构体
arg8: (未使用)
arg9: 配置字符串 (含 a11 字段)

```

#### Phase 1: 参数准备

```
string_c_str(arg3 + 0x30)        → functionId
json_node_create(functionId)     → v35 (失败 → error 0x17C1)
atomic_load_u32(arg5, 5)         → v10 (计数器)
get_sdk_version()                → sdk
if (sdk >= 24 && sdk <= 26):
    v5 = 0x1000000
else:
    v5 = 0x2000000

```

SDK 版本影响后续标志位，24-26 对应 Android 7.0-8.0。

#### Phase 2: JNI 环境探测 + 随机数混淆

这段是整个函数里最绕的部分。它尝试通过 JNI 调用验证运行环境是否真实：

```
if (arg6 != NULL && !vector_empty(arg6)):
    jni_FindClass("java/lang/String")    → data[0x38]
    if (FindClass 成功):
        jni_GetMethodID("getBytes", "(Ljava/lang/String;)[B")
        jni_CallObjectMethodA(...)
        if (异常):
            jni_ExceptionClear()
            // 生成随机数
            random() % (0xFF - 0x0F) + 0x0F + 1
            左移 8 位 → OR 到 v34

```

根据不同路径（FindClass 成功 / 失败、GetMethodID 成功 / 失败、是否有异常），走不同的随机数生成分支，最终都是往 v34 里 OR 一个随机值。

最后：

```
v34 = (i32)v34
v34 ^= arg4 (时间戳)
v37 = json_node_create_f64((double)v34)   → e3 字段

```

这个 e3 字段看起来是时间戳混淆值——用 JNI 环境探测结果的随机数 XOR 时间戳，转成浮点数存入 JSON。如果你在模拟器里跑（没有真实 JNI 环境），这个值的生成路径会不同，服务端可能据此判断环境真实性。

#### Phase 3: 字符串拼接 — 构建 e2 字段

```
memset(v11, 0, 128)
sprintf(v11, "%d", v10)                  → 格式化计数器

string_write_fd(v84, arg3+0x18)          → 协议版本
string_append("§§")                      → data[0x31] 分隔符
string_append("0")                       → data[0x0E]
string_append("§§")
string_write_fd(v3, *arg7+0x00)          → 包名
string_append("|")                       → data[0x36]
string_write_fd(v3, *arg7+0x48)          → 签名 hash
string_append("|")
string_write_fd(v3, *arg7+0x18)          → app 版本
string_append("§§")
string_write_fd(v3, *v50+0x00)           → 设备信息
string_append("§§")

```

最终 e2 格式：`"1.0§§0§§com.jingdong.app.mall|E0D1A70367...|15.2.70§§设备信息§§..."`

这里的 data[0x31]、data[0x36] 等是 data_buf 里的常量字符串。`§§` 是分隔符，`|` 是子字段分隔符。

#### Phase 4: JSON 构建

```
json_tree_create()
json_node_add(tree, "e1", v35)           → functionId 的 c_str
json_node_add(tree, "e2", v36)           → 拼接后的设备信息
json_node_add(tree, "e3", v37)           → 时间戳混淆值 (f64)
json_node_add(tree, "e5", v38)           → token (arg3+0x48 或空)

```

注意跳过了 e4。后面还有条件分支，根据 `atomic_check_bit0` 决定是否添加额外字段（e4/e6/e7/e8 等），这些字段跟 arg3 结构体的 +0x70、+0x88、+0x60、+0x68 偏移有关。

```
json_serialize(tree) → JSON 字符串 (约 745 字节)
json_tree_free(tree)

```

#### Phase 5: zlib 压缩

```
string_length(json_str)          → len = 745
compressBound(len + 1)           → bound = 764
malloc(bound)                    → v81 (压缩缓冲区)
memset(v81, 0, bound)
compress(v81, &v76, json_cstr, len)  → zlib 压缩
// 压缩后约 526 字节

```

标准 zlib compress，没有魔改。

#### Phase 6: RC4 密钥派生

```
string_from_cstr(v65, "a11")                     → data[0x81]
string_split_get(*arg9, v82, v65)                → 从配置字符串提取 a11 字段
string_c_str(v82)                                → "4243db92b492307870de03ba0bf47aad7e14339e"
hex_to_bytes(a11_hex, &key_len)                  → 20 字节原始 key

```

然后是 XOR 密钥派生：

```
// v69 = key_len (20)
// v70 = 8 (时间戳字节数)
// v71 = arg4 (时间戳 LE 指针)
for (int i = 0; i < key_len; i++) {
    derived_key[i] = a11_raw[i] ^ timestamp_le[i % 8];
}

```

a11 是 APK 版本级常量（v15.2.70 对应 `4243db92b492307870de03ba0bf47aad7e14339e`），每次调用用当前毫秒时间戳 XOR 派生出 RC4 key。

这里有个设计——密钥派生用的是时间戳的 8 字节 little-endian 表示，循环 XOR 到 20 字节的 a11 mask 上。解密端只要知道时间戳就能还原 key。

#### Phase 7: RC4 加密

```
rc4_init(sbox, derived_key, 20)              → KSA (S-box 初始化)
rc4_crypt(sbox, compressed, output, len)     → PRGA (加密)

```

IDA 里确认了 RC4 实现：

*   KSA 在 sub_79480：标准 `S[i]=i` 初始化 + key mixing
*   PRGA 在 sub_795B4：标准 `i++, j+=S[i], swap, XOR`
*   S-box 258 字节（256 + 2 字节 i/j 计数器）

标准 RC4，没有魔改。

#### Phase 8: Base64 编码 + 输出

```
base64_encode_alloc(ciphertext, cipher_len, &result)  → b4 字符串
// 写回结果
*ret_ptr = result
*ret_len_ptr = result_len
// 设置 error_code = 0
// 清理临时字符串
halt

```

* * *

### 完整调用链

```
Java: Bridge.main(101, objArr)
  └→ JNI native (0x922A8)
      └→ sign_basic_dispatch (0x86FEC)
          └→ vm_exec_sign_basic (0x875C0) [编排层, 463 条指令]
              ├→ 构建 JSON (b1~b7 字段名)
              ├→ 格式化时间戳 → b7
              ├→ call_native(7) → sign_jni_dispatch
              │   └→ vm_exec_sign_jni (0x876F4) [核心, 905 条指令]
              │       ├→ 拼接 e2 字符串 (设备信息)
              │       ├→ JNI 环境探测 + 随机数混淆 → e3
              │       ├→ 构建 JSON {e1, e2, e3, e5, ...}
              │       ├→ zlib compress
              │       ├→ 派生 RC4 key (a11 XOR timestamp)
              │       ├→ RC4 加密
              │       └→ Base64 编码 → b4
              ├→ call_native(9) → anti_frida_dispatch
              │   └→ vm_exec_anti_frida (0x87834) [检测]
              ├→ 内部计算 b5 = HMAC-SHA256(key, MD5(sign_str))
              ├→ 内部计算 b6 = HMAC-SHA256(key, MD5(sign_str + b5))
              └→ 组装最终 JSON 输出

```

* * *

### Python 还原

```
import zlib, struct, base64, json

A11_MASK = bytes.fromhex("4243db92b492307870de03ba0bf47aad7e14339e")

def rc4(key: bytes, data: bytes) -> bytes:
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) & 0xFF
        S[i], S[j] = S[j], S[i]
    i = j = 0
    out = bytearray(len(data))
    for k in range(len(data)):
        i = (i + 1) & 0xFF
        j = (j + S[i]) & 0xFF
        S[i], S[j] = S[j], S[i]
        out[k] = data[k] ^ S[(S[i] + S[j]) & 0xFF]
    return bytes(out)

def derive_rc4_key(timestamp_ms: int) -> bytes:
    ts_bytes = struct.pack("<Q", timestamp_ms)
    return bytes([A11_MASK[i] ^ ts_bytes[i % 8] for i in range(20)])

def decrypt_b4(b4: str, timestamp_ms: int) -> str:
    ciphertext = base64.b64decode(b4)
    key = derive_rc4_key(timestamp_ms)
    compressed = rc4(key, ciphertext)
    return zlib.decompress(compressed).decode()

```

用这个脚本，给定 b4 密文和对应的时间戳，就能解密出原始 JSON 明文。验证通过。

* * *

### 密钥材料总结

<table><thead><tr><th>名称</th><th>长度</th><th>值 (v15.2.70)</th><th>来源</th></tr></thead><tbody><tr><td>a11 mask</td><td>20B</td><td><code>4243db92b492307870de03ba0bf47aad7e14339e</code></td><td>arg9 配置, APK 版本级常量</td></tr><tr><td>timestamp</td><td>8B</td><td>毫秒时间戳 LE</td><td>arg4, 每次调用不同</td></tr><tr><td>RC4 key</td><td>20B</td><td><code>a11[i] ^ ts_le[i%8]</code></td><td>动态派生</td></tr><tr><td>hmac_key</td><td>36B</td><td>APK 版本级常量</td><td>sign_basic 内部 handler</td></tr></tbody></table>

* * *

### VM 混淆评估

<table><thead><tr><th>保护层</th><th>技术</th><th>效果</th></tr></thead><tbody><tr><td>指令集</td><td>私有 bytecode + 变长指令 + XOR 编码</td><td>IDA 完全无法识别</td></tr><tr><td>解释器</td><td>花指令 + threaded dispatch</td><td>IDA 反编译失败</td></tr><tr><td>加密原语</td><td>RC4/SHA256/MD5 内联在 handler 中</td><td>无标准库调用可 hook</td></tr><tr><td>间接跳转</td><td>全局指针表分发 (off_121A70 等)</td><td>静态分析断链</td></tr><tr><td>Anti-unidbg</td><td>FindClass("java/lang/String") 小写检测</td><td>单点绕过</td></tr><tr><td>Anti-Frida</td><td>/proc/self/task 线程名检测</td><td>可绕过</td></tr></tbody></table>

**弱点**：标准算法无魔改，密钥明文在内存中，trace 可完整记录执行路径。XOR 编码是薄纱，跳转表可运行时 dump，handler 逻辑虽然有花指令但内部大多干净。

* * *

### 结论

整个分析路径：

1.  找入口 — trace 搜干净函数，反向引用追到 VM 入口
2.  逆上下文 — vm_ctx_init 是干净代码，直接读
3.  逆 dispatch — trace 还原 threaded interpretation 机制
4.  提跳转表 — Frida dump 571 条 opcode→handler 映射
5.  逆 handler — IDA 看汇编 + trace 验证，逐个确认语义
6.  写反汇编器 — handler 注册表架构，按需添加解码器
7.  读反汇编输出 — 905 条指令直接可读，还原算法

最终结论：**b4 = Base64(RC4(a11 XOR timestamp, zlib.compress(json_plaintext)))**

标准算法组合，没有魔改。保护强度全靠 VM 壳——你不把 handler 逆完就读不懂 bytecode，但一旦逆完了，算法本身没有任何难度。

anti_frida 那块没细看，不影响签名还原。等有精力再分析。

* * *

### 回头看

这个 VM 从第一次碰到到最终还原，中间隔了大半年。第一次看的时候连 handler 是什么都不知道，trace 拿到了也不知道该怎么用，纯纯的看天书。

现在回头看，京东这套 VM 的设计其实很 "工程化"——框架代码不混淆方便自己维护，花指令只加在解释器入口，handler 内部逻辑干净，算法用标准库不魔改。它的安全性完全建立在 "你看不懂我的指令集" 这一层上。一旦你把 handler 逆完、反汇编器写出来，剩下的就是读伪代码，跟读一个普通 C 函数没区别。

真正耗时间的不是算法本身，是前面那些基础设施——搞清楚 dispatch 怎么跳、指令边界在哪、操作数怎么解密、寄存器文件怎么寻址。这些东西每一个单独拿出来都不难，但组合在一起就是一堵墙。你得一块砖一块砖拆，没有捷径。

工具链方面，Frida trace 是核心——花指令再多，执行路径是确定的。IDA 负责看 handler 内部汇编，trace 负责验证。两个配合着用，比单独用任何一个都快。

下一个目标大概是快手或者美团的 VM 也可能继续去还原抖音的 VM，看看别家的设计思路有什么不同。京东这套算是入门级——标准算法、固定 XOR、无自修改代码。真正硬的应该是带运行时 key 更新、handler 动态生成、或者算法本身魔改的那种。到时候再说。

本文仅用于安全研究与技术学习交流，不提供任何可直接用于绕过商业产品保护的完整工具或服务。文中涉及的分析对象版本已过时，所有密钥材料均为历史版本数据。请勿将本文内容用于未经授权的商业用途，因使用本文信息产生的一切法律后果由使用者自行承担