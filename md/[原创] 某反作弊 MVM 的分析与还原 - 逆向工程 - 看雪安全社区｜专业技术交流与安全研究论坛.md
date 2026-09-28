> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293085.htm#msg_header_h1_26)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

前段时间分析一份 Android 侧反作弊样本，其中相当一部分检测逻辑没有直接编译在 native 代码里，而是被放进一套自定义 VM 中执行。

最开始拿到的主要是两个东西：

```
libanogs.so
mrpcs_a_v.data
```

`libanogs.so` 是 stripped ARM64 ELF。

脚本实际进入的 VM 执行函数位于：

```
sub_3F80CC
```

这个函数外围又套了比较重的 OLLVM flattening，直接 F5 基本只能看到大量状态计算、间接跳转和被拆碎的 basic block。

而 `mrpcs_a_v.data` 外层还是 ZIP，解包以后才是实际送入 MRPCS/VMRPCS loader 的数据。

最初我的目标只是：

> 把 `sub_3F80CC` 看懂，知道脚本里面到底执行了什么。

但分析到后面，整个工作逐渐变成了一套完整工具链：

```
MRPCS / VMRPCS
      ↓
Parser
      ↓
VM Decoder
      ↓
Disassembler
      ↓
Clean VM Emulator
      ↓
CFG / IR
      ↓
C-like Decompiler
```

最终这份样本中的 VM 代码已经可以批量恢复成结构化 C-like 结果。

本文只讨论离线样本中的 VM 格式、解释器和反编译过程，不讨论对线上目标的绕过。

`mrpcs_a_v.data` 外层是一个 ZIP。

解包后只有：

```
unzipmrpcs.data
```

大小：

```
227488 bytes
0x378A0
```

内层开头类似：

```
f8 8d 24 4d 2c 37 28 2a 39 29 ...
```

上层 dispatcher 对头部存在一层编码处理。

当时先根据 loader 逻辑算出：

```
key = payload[1] + payload[3]
    = 0x8D + 0x4D
    = 0xDA
```

再结合长度低字节处理头部，可以得到：

```
56 4D 52 50 43 53
 V  M  R  P  C  S
```

也就是：

```
VMRPCS
```

后续直接对主体做相应解码，也能看到大量明文字符串，例如：

```
libUE4.so
libanogs.so
libanort.so

/proc/self/maps
/data/adb
/data/adb/ksud

pthread_mutex_trylock
pthread_mutex_unlock
strstr
sscanf
dl_iterate_phdr
...
```

这里后面有一个很重要的修正：

**payload 层的 `0xDA` 编码，和 VM 解释器内部的 runtime opcode mapping 不是一回事。**

一开始这两个概念很容易混在一起，后面分析 opcode 初始化函数以后才彻底拆开。

继续跟 loader，会发现 VMRPCS 内部还有一层 section directory。

每个目录项固定 10 字节：

```
struct SectionEntry
{
    uint16_t type;
    uint32_t offset;
    uint32_t size;
};
```

即：

```
u16 type
u32 offset
u32 size
```

loader 会检查：

```
offset + size <= payload_size
```

然后按照 `type` 把不同 section 装载到不同区域。

当前样本中比较关键的是：

```
type 1:
    offset = 0x41D6
    size   = 0x33698
           = 210584 bytes

type 2:
    offset = 0x28
    size   = 0x41AE
           = 16814 bytes
```

Type1 就是真正的 VM bytecode。

Type2 主要是数据和字符串。

因此真正应该交给 VM decoder 的范围是：

```
decoded_payload[
    0x41D6 :
    0x41D6 + 0x33698
]
```

而不是整个文件。

这个结论非常重要。

如果连 code section 都没有切对，后面做 opcode 统计、函数发现、CFG 都会混入大量数据区内容。

VM 主函数开头：

```
3F80CC  SUB  SP, SP, #0x1F0
...
3F8108  MOV  X28, X0
```

很快就能看到典型的间接分发：

```
3F8118  LDR W20, [X28,#0x38]!
...
3F812C  LDR X8, [X28,#-8]!
3F813C  LDR W11, [X8]
3F8148  CMP W11,#8
...
3F8154  LDR X9, [X21,W9,UXTW#3]
3F8158  BR  X9
```

第一眼很容易认为：

```
table → BR X9
```

就是 VM opcode dispatcher。

但继续跟下去会发现并不是。

这条链本质上更像：

```
opaque arithmetic
      ↓
OLLVM state/index
      ↓
0x535030 附近的跳转表
      ↓
BR X9
```

也就是说：

```
0x535030
```

附近的表首先是 **OLLVM control-flow dispatcher table**，不是 VM opcode table。

这是整个分析过程中第一个比较容易踩的坑。

我的做法后来变成：

不要试图一开始把整个 `sub_3F80CC` 恢复成漂亮 CFG。

只找这些东西：

```
谁读 bytecode
谁读 PC
谁访问 register file
谁解析 descriptor
谁修改 next PC
```

在一些真实语义块中，可以看到：

```
LDR   X8, [ ... ]
LDR   X8, [X8]
LDR   X9, [X8,#8]
ADD   X11,X9,X27

LDRB  W12,[X11,#1]
LDRB  W8, [X11,#2]
LDRB  W11,[X11,#3]
```

可以直接抽象成：

```
insn = code_base + pc;

descriptor = insn[1];
arg1       = insn[2];
arg2       = insn[3];
```

随后又能看到大量 handler 最终：

```
ADD W23, W23, #4
```

或者：

```
ADD W23, W23, #5
ADD W23, W23, #7
ADD W23, W23, #0xB
```

最后统一走到：

```
403BD8:
    LDR X8,[X25]
    MOV W9,W23
    STR X9,[X8,#0x130]
```

于是这里的关系逐渐清楚：

```
X27 ≈ current PC
W23 ≈ next PC
state + 0x130 ≈ VM PC
```

后面 loader/header 的数据流又进一步验证了：

```
state + 0x130 = entry cursor / PC
state + 0x138 = code limit / size
```

这比先把几万行 OLLVM CFG 重建一遍有效得多。

handler 中可以反复看到：

```
ADD X11, X9, X11, LSL #3
LDRB W11, [X11,#0x10]
```

以及：

```
ADD  X8, X9, X8, LSL #3
STRB W10, [X8,#0x10]
```

其它路径还有：

```
STRH W10, [X8,#0x10]
STR  W10, [X8,#0x10]
STR  X10, [X8,#0x10]
```

所以 register addressing 非常明显：

```
reg[i] = state_base + 0x10 + i * 8;
```

即：

```
8 bytes / register slot
```

这是一个 64 位 register VM。

但这里后来在 Clean VM 验证阶段还发现了一个很重要的细节：

> 窄写入保留高位。

比如：

```
MOV.d
```

只会修改 register slot 的低 32 位，不会像：

```
regs[x] = (uint32_t)value;
```

那样顺便清掉高 32 位。

第一版 emulator 没处理这个细节时，某些 HOSTCALL service ID 会突然变成巨大的异常数值。

修正 narrow-write semantics 后，这些错误消失。

这个细节反过来也验证了 descriptor width 的分析是正确的。

`0x535030` 那张 dispatcher table 还有一个问题：

直接看 ELF 文件时，很多内容似乎都是 0。

如果只看静态字节，很容易怀疑这个表是不是运行时动态生成的。

继续看 relocation 后才发现：

这些地址由：

```
R_AARCH64_RELATIVE
```

在 ELF load 时填入。

于是可以直接从 relocation table 恢复：

```
OLLVM index
     ↓
real basic block
```

也就是说并不需要跑游戏，也不需要先动态 trace。

静态解析 relocation 就可以把大量：

```
CSEL
...
LDR Xn, [table,index]
BR  Xn
```

还原成真实控制流目标。

第一版就恢复出了 68 个 logical opcode 对应的真实 handler entry。

这一点对后面的效率影响很大：

**不需要完整解决 OLLVM，只需要把 VM semantic block 从 OLLVM state machine 里剥出来。**

一开始建立出的表类似：

```
OP00 -> 0x403958
OP01 -> 0x4039B8
OP02 -> 0x403A48
OP03 -> 0x403B0C

OP04 -> 0x3F820C
OP05 -> 0x3F82D0
OP06 -> 0x3F8458
OP07 -> 0x3F8720
...
```

但很快出现了矛盾：

某些 opcode 从真实脚本行为看，和 handler entry 附近看到的 ARM64 指令对不上。

最后才发现 VM 的控制流其实是：

```
opcode handler entry
        ↓
descriptor 二级分发
        ↓
共享 OLLVM block
        ↓
真正 semantic block
```

所以：

> 不能因为 handler 入口附近出现某条 AArch64 指令，就给整个 opcode 定性。

真正应该分析的是：

```
(opcode, descriptor)
```

组合。

继续跟 `0x396D30` 附近的初始化逻辑，最终恢复出 96 字节 runtime opcode map。

计算过程可以整理成：

```
mode = program[0x174]
seed = program[0x170]

t = ((mode | 0xA0) ^ seed) & 0xFF

if mode < 3 or t == 0:
    k = 0
else:
    k = t

opcode_map[i] = i ^ k
```

因此：

```
raw opcode = logical slot ^ k
```

反过来：

```
logical slot = raw opcode ^ k
```

当前样本对应：

```
k = 0
```

所以这份脚本里：

```
raw opcode == logical opcode
```

只是恰好成立。

这里也正式把两个概念分开了：

```
VMRPCS payload 的 0xDA 编码
```

和：

```
VM runtime opcode XOR key k
```

不是同一层东西。

这个 VM 的基本指令布局可以简化成：

```
+0  opcode
+1  descriptor
+2  operand A
+3  operand B / immediate...
```

descriptor 决定：

```
register / immediate
src width
dst width
operand mode
```

主要宽度：

```
64 bit
```

immediate 常见长度：

```
8 bytes
```

因此常见指令总长度为：

```
11 bytes
```

这和前面 handler 中不断看到的：

```
ADD nextPC,#4
ADD nextPC,#5
ADD nextPC,#7
ADD nextPC,#0xB
```

正好对应。

也就是说 PC 行为、decoder 和 descriptor 三层能够互相验证。

我觉得这部分反而值得单独写出来。

早期分析 `OP07` 时，在某些共享 basic block 里看到：

```
LDRSB
LDRB
STRB
```

当时很自然地判断：

```
OP07 ≈ indirect LOAD/STORE
```

甚至第一版 ISA 表里也是这么写的。

但后面把 descriptor 二级分发理清，再重新沿 `sub_3F80CC` 的实际 handler 路径追了一遍，发现这个结论是错的。

最终确认：

```
OP07 = SMUL_EXT
```

语义类似：

```
dst = truncate_to_dst_width(
    dst * sign_extend(src, src_width)
);
```

descriptor 关系：

```
low = 5
    → register source

mid
    → src signed width
      8 / 16 / 32 / 64

hi
    → destination/result width
      8 / 16 / 32 / 64
```

而且它只接受 register-source 模式。

也就是说早期看到的访存指令，只是共享 OLLVM/descriptor block 中的数据装载，不代表这个 opcode 的最终高层语义。

这个修正让我后面给 opcode 命名时都坚持：

```
native handler
+
descriptor
+
真实 bytecode
+
Clean VM
```

至少多层交叉验证以后再定。

`sub_3F80CC` 有：

```
96 opcode slots
```

但最终确认这个 build 真正实现的只有：

```
69 / 96
```

剩下：

属于未实现 / 保留 slot。

恢复出的 family 包括：

```
MOV

ADD
SUB
MUL
DIV
REM

AND
OR
XOR
SHIFT

CMP
SETcc

JMP
Jcc

CALL
CALL_REG
RET

PUSH_ARG

ENTER_FRAME
LEAVE_FRAME

HOSTCALL
CALL_NATIVE
```

浮点部分也不是摆设，可以看到：

```
FADD
FSUB
FMUL
FDIV
FCMP

FMIN
FMAX

FCVTZU
FCVTZS
UCVTF
SCVTF
...
```

条件位除了整数：

```
EQ
HI
HS
LO
LS
NE
GT
GE
LT
LE
```

还包括：

```
FUEQ
FOEQ
FOGT
FOGE
FOLT
FOLE
FUNE
FONE
FORD
FUNO
```

也就是说它是一套相对完整的通用 register VM，不只是为了执行几个简单检查临时拼出来的解释器。

只看 ISA 还不够。

大量实际行为通过 HOSTCALL 完成。

最终大致可以分三类：

```
低编号 service
    → Linux syscall

0x3xx
    → libanogs internal service

>= 0x500
    → dynamic provider registry
```

其中一些已经可以直接命名：

```
0x36E  memset
0x36F  memcpy_checked
0x372  strlen
0x38C  CALL_NATIVE_N
0x38D  RESOLVE_SYMBOL
0x390  mincore

0x51C  malloc
0x51E  free
0x524  fopen
0x529  fclose
0x569  fgets
```

还有：

```
0x35C  REGISTER_VM_CALLBACK
0x359  VM_SLOT_GET
0x35A  VM_SLOT_SET
```

序列化部分：

```
0x367  SERIALIZER_SET_MODE
0x301  SERIALIZER_APPEND
0x302  SERIALIZER_FLUSH
```

`0x301` 的类型参数最终也可以解成：

```
1 = u8
2 = u16
3 = u32
4 = u64
5 = cstring
```

其中多字节整数按照 big-endian 写入内部 buffer。

于是原来：

```
PUSH_ARG r3
PUSH_ARG 3
HOSTCALL 0x301
```

可以直接提升为：

```
serializer.append_u32(value);
```

而：

```
"libc.so"
"strstr"
HOSTCALL 0x38D
```

则可以恢复成：

```
resolve_symbol("libc.so", "strstr");
```

到这里代码可读性才真正开始产生质变。

把 ISA 表写出来以后，我没有直接认为 VM 已经逆完。

因为静态分析最怕一种情况：

> 每条指令单独看都 “很像对的”，但组合执行以后其实是错的。

所以后面重新写了一套完全不依赖 `sub_3F80CC` 的 Clean VM Emulator。

自己的 VM state 大致包含：

```
struct VMState
{
    uint64_t regs[...];

    uint64_t pc;
    uint64_t flags;

    VMStack stack;
    VMCallStack calls;

    ...
};
```

然后实现：

```
integer ALU
float ALU
load/store
flags
Jcc
CALL/RET
frame
HOSTCALL stub
```

HOSTCALL 默认在分析沙箱里模拟，不让脚本真正执行外部检测动作。

这样原始：

```
sub_3F80CC
```

就从唯一执行引擎变成了：

```
oracle / reference implementation
```

一个例子就是前面提到的 narrow write。

第一版 emulator 把：

```
MOV.d
```

实现成整个 64 位 register 覆盖。

结果某些 HOSTCALL service ID 会变成明显不正常的巨大数。

重新回解释器对照后才发现：

```
32-bit write
```

只改低 32 位，高位保持。

修正以后 callback 执行重新正常。

也就是说 Clean VM 不只是为了 “做个演示”，而是真正承担了：

```
静态 ISA 假设
    ↓
真实 bytecode 执行验证
    ↓
发现不一致
    ↓
回原解释器修正
```

这条闭环。

Clean VM 做 callback 批量验证时，曾经出现：

```
31 / 32 callback 可以执行到 RET
```

剩下：

```
0x6C93
```

一直跑到 step limit。

最开始很自然地怀疑：

```
还有一条 opcode 没实现？
CALL_REG 有问题？
HOSTCALL 返回不对？
```

后来直接对 `0x6C93` 做静态 CFG returnability analysis，才发现：

```
它根本没有可达 RET
```

内部有两个闭环。

也就是说这不是一个 “没有跑通的 callback”，而是设计上：

```
non-returning persistent worker
```

于是正确统计应该是：

```
31 / 31 应返回 callback
    → 全部正常 RET

1 个 persistent worker
    → expected non-returning

semantic gap
    → 0

unexpected VM error
    → 0
```

这时候才可以比较有把握地说：

> 针对当前脚本，Clean VM 已经没有已知 opcode semantic gap。

VM 能执行以后，下一个目标是：

> 不看 VM 汇编，直接得到接近 C 的代码。

所以 lifter 不再读取文本形式的 disassembly，而是直接：

```
bytecode
   ↓
decoder
   ↓
basic block
   ↓
CFG
   ↓
VM register symbolic propagation
   ↓
IR
```

同时把：

```
FP
SP
LR
PUSH_ARG
HOSTCALL
```

等虚拟机噪声向高层折叠。

例如 `0x6C94` 原本是 19 条 VM 指令，最终可以压成：

```
free(g_0470);
g_0470 = 0;
return;
```

另一个 `0x6C63` 有 54 条 VM 指令，可以变成：

```
++g_16A0;

serializer.set_mode(3);

serializer.append_u32(g_16A0);
serializer.append_u32(g_1608);
serializer.append_u32(g_160C);
serializer.append_u32(g_1610);
serializer.append_u32(g_1614);

serializer.flush(1);
```

当时还专门拿一个 592 条 VM 指令的 `0x6C6E` 做压力测试。

第一版 lifted block IR 大约 270 行，已经可以自动恢复：

```
tid = gettid();

sprintf(proc_path, "/proc/%d/comm", tid);

fp = fopen(proc_path, "r");

if (fp) {
    fgets(comm, 0x1E, fp);
    fclose(fp);
}

len = strlen(comm);
tail_len = len < 17 ? len : 16;

strncpy(
    comm_tail,
    comm + len - tail_len,
    tail_len
);

eglGetCurrentContext =
    resolve_symbol(
        "libEGL.so",
        "eglGetCurrentContext"
    );
```

这时候已经不是根据字符串 “猜这个 callback 在干什么”，而是真正把 VM 的寄存器数据流提升成了高层表达式。

第一版 IR 仍然会有：

```
loc_xxx:
goto loc_xxx
phi(...)
```

后面继续做：

```
dominator
post-dominator

natural loop

if
if/else

early return
break

while

phi resolution
```

早期还有大约 20 个复杂 helper 需要 fallback。

但继续补通用 CFG pattern 后，最终当前样本可以做到：

```
393 / 393 functions
393 / 393 zero fallback

0 residual phi_rXX
```

也就是说全部 VM 函数都可以进入结构化 C-like 输出。

一开始函数数量没有这么多。

原因之一就是：

如果把：

```
CALL target
```

错误地当作当前函数内部 CFG edge，那么整个 function boundary 会被合并。

后来明确区分：

```
JMP/Jcc target
    → basic block

CALL target
    → function entry
```

再结合标准函数序言扫描整个 Type1。

最终发现：

```
393 个标准 VM function entry
```

并且：

```
393 / 393 独立解码成功
0 function boundary overlap
0 instruction format error
```

Type1 大小：

```
210584 bytes
```

其中函数指令覆盖：

```
210192 bytes
```

剩余：

```
392 bytes
```

而这 392 字节恰好全部是单字节：

393 个函数之间正好有：

```
392 个 separator
```

所以这时才比较有把握地确认：

> Type1 函数边界已经完整切出来了。

早期一直按：

```
32 callbacks
```

统计。

完整扫描 registration 后才发现这个数字不对。

最终是：

```
33 unique callback entries
34 registrations
```

因为：

```
VM entry 0x10479
```

同时注册成：

```
0x6C57
0x6C60
```

所以：

```
33 callback functions
34 task IDs
```

这个例子也说明：

只从某个初始化入口递归调用图，并不能保证找到所有上层任务。

还必须结合：

```
registration
function prologue
indirect call
callback thunk
```

一起看。

普通：

```
CALL imm
```

很好解决。

真正麻烦的是：

```
CALL_REG rX
```

目标可能来自：

```
constant propagation
stack slot
function argument
object field
runtime resolver
```

后面不断加入：

```
SSA
constant propagation
stack-local propagation
caller/callee propagation
```

后，当前样本最终达到：

```
0 unresolved CALL_REG
```

完整静态 VM call graph：

```
906 edges
```

这一步对于后续函数可达性和 AI / 人工分析都很重要，因为只要还有大量 unresolved indirect call，callgraph 就是不完整的。

把 VM register 消掉以后，新的问题变成：

```
*(uint64_t *)(fp + 0x470)
*(uint32_t *)(fp + 0x478)
*(uint64_t *)(fp + 0x480)
```

机器语义没有错。

但人明显能看出：

这些 offset 很可能属于一个 stack object。

于是后面的工作逐渐从：

```
VM devirtualization
```

转向：

```
type recovery
stack object recovery
struct recovery
```

当前已经恢复出多种对象模式，例如：

```
/proc/self/maps parser state

CPU affinity mask

u64 pair
u64 triple

fixed record

mixed-width aggregate

overlay / union

stack buffer
```

当前完整样本里已经有：

```
37 个函数命中 stack object recovery

90 个 stack buffer/object alias

597 处 fp+offset rewrite
```

并继续在局部 aggregate、函数参数和返回类型之间做跨函数传播。

做到 HOSTCALL 和 C-like 层以后，一些 callback 的行为已经不难理解。

比如 `0x6C99`。

它会读取：

```
/proc/self/maps
```

解析：

```
start-end
permissions
pathname
```

并检查若干映射特征，包括：

```
shadowhook-enter
rwxp
anon:LMA
```

后续还会结合：

```
process_vm_readv
```

对映射内存做进一步读取 / 验证，最终把异常 mapping 整理成记录并交给 serializer。

HOSTCALL 没恢复以前，这类代码只能看到：

```
hostcall_0x373(...);
hostcall_0x10e(...);
```

HOSTCALL 恢复以后就可以直接显示：

```
strstr(...);
process_vm_readv(...);
```

因此对于这类 VM，**native bridge 语义恢复和 opcode 恢复同样重要**。

当前完整样本最终结果：

```
VM functions
393 / 393

zero fallback
393 / 393

callbacks

registered tasks

static VM call edges

unresolved CALL_REG
```

最新输出层还有：

```
physical VM rN references

residual phi_rXX
```

也就是说早期大量存在的：

```
r15
r16
r17
```

这类 VM physical register 已经从最终 C-like 展示层清掉。

类型恢复目前还包括：

```
309 个函数推导出参数

174 个函数推导出返回值

182 个指针/字符串类参数

12 个严格经过全调用点一致性验证的
跨函数指针/结构类型传播
```

类型传播这里刻意比较保守。

只有当一个 callee 参数的所有已观察调用点都给出一致强证据时，才提升成：

```
VmU64Pair *
uint8_t *
const VmU64Pair *
```

如果存在未知调用点、子字段地址或类型冲突，就继续保留：

```
uint64_t
void *
```

而不是为了输出好看强猜类型。

做到最后以后，整个工具大概变成：

```
┌──────────────┐
                    │   MRPCS      │
                    └──────┬───────┘
                           │
                           ▼
                  ┌────────────────┐
                  │ VMRPCS Parser  │
                  └───────┬────────┘
                          │
                  section directory
                          │
                          ▼
                  ┌────────────────┐
                  │   VM Backend   │
                  │     mvm_v1     │
                  └───────┬────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
       Decoder         HOSTCALL        ABI/Types
          │
          ▼
     Disassembler
          │
          ▼
     Function Discovery
          │
          ▼
         CFG
          │
          ▼
       VM Lifter
          │
          ▼
     SSA / Dataflow
          │
          ▼
      Structurer
          │
          ▼
   Type / Object Recovery
          │
          ▼
      C-like Output
```

其中 MVM 特有的部分尽量放到 backend：

```
opcode map
descriptor
ISA
section layout
HOSTCALL
VM ABI
callback registration
```

而这些则尽量通用：

```
CFG
dominator
SSA
dataflow
structurer
type propagation
C-like emitter
```

这样以后如果出现同系列但 opcode map、descriptor 或 HOSTCALL 有变化的新版本，只需要加 backend，不用重新写整个反编译器。

分析 VM 时，最重要的通常不是恢复原生函数的漂亮 CFG。

而是先找到：

```
PC
bytecode
opcode
descriptor
register
next PC
```

只要这些还存在稳定的数据流，就可以直接绕着 OLLVM 做定向恢复。

这次：

```
0x535030
```

就是典型例子。

如果一开始沿着这个误判继续做，会把后面的 handler 全部对应错。

`OP07` 是最好的反例。

一开始：

```
看到 LOAD/STORE
→ 判断它是 indirect memory operation
```

后来发现只是 descriptor/shared block。

最终语义其实是：

```
SMUL_EXT
```

所以至少需要：

```
handler CFG
descriptor
真实 bytecode
emulator
```

交叉验证。

如果只做 static reverse，很多结论最终都只能写：

```
high confidence
```

但自己实现一份 clean VM 后，可以直接问：

```
同一份 bytecode
能不能按照我的语义真正跑通？
```

这个验证力度完全不同。

如果：

```
CALL target
```

也被并进当前函数 CFG，那么 function discovery 从一开始就是错的。

393 个函数能够完整恢复，这个基础问题必须先解决。

例如：

```
foo(&triple);
```

并不能单独证明：

```
foo(VmU64Triple *arg);
```

如果另一个调用点是：

```
foo(&triple.second);
```

那么这个类型提升就是错误的。

最终规则宁愿留下：

```
uint64_t arg1;
```

也不会为了一眼看上去像源码而制造错误结构。

做到后期以后，我对目标也有了一点变化。

如果代码最终主要给人阅读，那么当然希望：

```
MappingContext *ctx;
MappingRecord *record;
size_t length;
```

越多越好。

但如果后续分析本身大量交给 AI，那么它真正需要的是：

```
function boundary 正确

CFG 正确

CALL graph 正确

CALL_REG target 正确

HOSTCALL 语义明确

字符串 xref 正确

global read/write 正确

type evidence 可追踪

VM PC ↔ C-like 对应稳定
```

AI 对：

```
local_68
```

这个名字本身并没有人那么敏感。

真正严重的问题反而是：

```
两个不同 SSA value 被错误合并

branch merge 的值被删掉

CALL_REG 目标丢失

struct base 判断错误
```

所以工具后期的目标已经从单纯：

```
MVM → 漂亮 C
```

逐渐变成：

```
MVM
 ↓
结构化程序表示
 ↓
C-like
 ↓
Callgraph / Types / Xrefs
 ↓
AI Analysis Dataset
```

这一点也是为什么后面没有继续为了消灭所有 `local_x` 而强行猜变量名。

最开始面对的是：

```
stripped ARM64 ELF
        +
OLLVM flattening
        +
未知 VMRPCS 格式
        +
未知 opcode
        +
未知 descriptor
        +
未知 HOSTCALL
```

最后得到的是：

```
VMRPCS Parser

Type1/Type2 Section Parser

69/96 Implemented ISA Model

Clean VM Emulator

393 VM Functions

33 Unique Callbacks

34 Registered Tasks

906 Static VM Call Edges

0 Unresolved CALL_REG

0 Residual phi

393/393 Structured C-like Output
```

当前样本已经完整进入 C-like 输出。

整个过程中，我觉得最有意思的并不是最后识别出了多少检测项，而是研究对象本身发生了变化。

最初是：

```
如何看懂 sub_3F80CC？
```

后来是：

```
如何实现这个 VM？
```

再后来变成：

```
如何实现这个 VM 的反编译器？
```

最终原本被：

```
OLLVM + 自定义 VM
```

包起来的一整套程序，又重新变成了普通的：

```
函数
控制流
调用图
类型
结构体
字符串
系统调用
```

从这个角度看，自定义 VM 也没有什么特别神秘的。

比较有效的思路还是：

```
先找到文件边界
      ↓
找到 VM code
      ↓
绕过 OLLVM dispatcher
      ↓
恢复 VM state
      ↓
恢复 PC / register / descriptor
      ↓
建立 ISA
      ↓
自己实现 VM
      ↓
用真实 bytecode 验证
      ↓
函数发现
      ↓
CFG / IR
      ↓
反编译
      ↓
类型与结构恢复
```

简单说就是：

> **先把 VM 当成一颗 CPU 逆，再把 VM bytecode 当成普通程序逆。**

本文先记录到这里。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)