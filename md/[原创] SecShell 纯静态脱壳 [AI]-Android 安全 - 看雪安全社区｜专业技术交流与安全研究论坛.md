> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293299.htm#msg_header_h2_6)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 全程零动态执行，所有算法均从二进制静态还原并用 Python 离线复现。使用 GLM5.3 最高分析，gpt6.1sol-high 纠正错误。

<table><thead><tr><th>项</th><th>值</th></tr></thead><tbody><tr><td>APK</td><td>143MB，<code>classes.dex</code> 单文件 41.5MB（正常业务 dex 不会这么大——第一个疑点）</td></tr><tr><td>包名</td><td><code>com.ixyzh.question</code>（versionName 1.9.4，minSdk 23 / targetSdk 30）</td></tr><tr><td>签名</td><td>仅 JAR v1（自签名证书），无 v2/v3 签名块</td></tr><tr><td>壳特征</td><td><code>lib/*/libSecShell.so</code> 全 ABI + <code>application="com.SecShell.SecShell.AW"</code> + <code>appComponentFactory="com.SecShell.SecShell.AP"</code></td></tr><tr><td>顶象私有签名</td><td><code>assets/meta-data/{manifest.mf, rsa.pub, rsa.sig}</code>（区别于 APK v1 签名的完整性清单）</td></tr><tr><td>环境对抗</td><td><code>assets/na.czl</code>：59 条模拟器检测路径（<code>/dev/socket/qemu</code>、<code>/dev/qemu_pipe</code>、<code>/goldfish</code>… 每条独立单字节 XOR）</td></tr></tbody></table>

壳 stub 是一个**合法的 DEX**（ART 能直接加载，负责拉起 libSecShell.so），载荷藏在整个文件尾部：

```
classes.dex (0x27A570C = 41,572,108 B)
┌──────────────────────────────────────────────────┐
│ DEX Header (合法)                                 │
│   magic     = "dex\n035\0"                        │
│   file_size = 0x27A570C   ← 声明整个文件(含载荷)   │
│   map_off   = 0xA561C    ← 壳元数据到此为止        │
├──────────────────────────────────────────────────┤
│ 壳 stub 元数据 (string_ids / class_defs / ...)    │
│   仅 264 个类: SecShell 壳类 + androidx 子集       │
├────────────── 0xA56EC ───────────────────────────┤
│ 载荷容器: [count=1][总长][名长=8]["dexdata0"]      │
├────────────── 0xA5700 ───────────────────────────┤
│ blob = [尺寸前缀 4B][载荷区域 ...]                 │
├────────────── 0x27A5700 ─────────────────────────┤
│ 尾部 12B: [未知u32][回指0xA56EC]["fdex"]          │
└────────────── 0x27A570C (EOF) ───────────────────┘
```

关键观察：`file_size` 覆盖全文件而 `map_off` 只到 `0xA561C`——DEX 规范允许 map 之后的 "填充区"，壳就把 40MB 载荷塞在这里，对 ART 完全不可见。

`readelf -d` / `nm -D` 对这个 so 全部失效（无符号、无导入）。原因是它有**两张动态表**：

```
Program Headers (本样本实际值):
  LOAD    R    vaddr 0x00000    p_offset 0x00000    filesz 0x119B20   ← 元数据明文+密文代码
  LOAD    RW   vaddr 0x12A170   p_offset 0x11A170    filesz 0xA118
  DYNAMIC      vaddr 0x12B0B8   p_offset 0x125AE0    ← ★ 同一段头两个地址
  LOAD    R+X  vaddr 0x14A078   p_offset 0x125078    filesz 0xA54     ← 明文引导桩

动态表定位:
  readelf / 常规解析器 → 跟 p_offset (0x125AE0) → 8 项+NULL 即止 = 诱饵 ✗
  bionic linker       → 跟 p_vaddr  (0x12B0B8) → 映射回文件 0x11B0B8 → 36 项 = 真表 ✓
```

linker 加载 so 时用 `p_vaddr` 经 LOAD 段映射定位动态表，从不用 `p_offset`——按 ELF 正常执行逻辑分析，这个反静态手法即可穿透：

```
def v2o(v, loads):                      # vaddr → file offset
    for va, off, fsz in loads:
        if va <= v < va + fsz: return off + (v - va)

# RW LOAD: vaddr 0x12A170 ↔ file 0x11A170
# 真动态表 = file[0x11B0B8:]
# → DT_INIT = 0x14A098, INIT_ARRAY = [0x169A8, 0x176AC]
```

真表还给出一个自洽性证据：引导桩所在 RX 段（`0x14A078`，仅 0xA54 字节）是全 so 唯一的明文代码。

```
0x14A078 (RX LOAD 段头):
  +0x00  qword 0xA28           桩代码大小
  +0x08  qword 0
  +0x10  qword 0x14A078        自身 vaddr (运行时算 slide 用)
  +0x18  qword 0x14CA8         ★ SRC = 压缩代码地址
  +0x20  ARM64 代码 ...         ← DT_INIT = 0x14A098 正好指向这里

META 参数区 (桩代码之后, 即 0x14AAA0, 全 u32):
  +0x0C = 0x14CA8   (SRC)
  +0x24 = 0x14CA8   (解压目标 = 原地覆盖)
  +0x1C = 0x104E78  (解压后尺寸)
  +0x28 = 0x891E5   (压缩流长度)
  +0x20 = 0         (尾部 XOR 层密钥 K, 0=禁用)
  +0x14 = 0         (绝对地址补丁数 n, 0=禁用)
```

桩执行流程（linker 在重定位完成后、任何 JNI 调用前调用 DT_INIT）：

```
// 1. 把 SRC 处 0x891E5 字节拷到匿名 RWX 页（防止解压时覆写压缩流自身）
// 2. mprotect(RWX) 于 [0x14CA8, 0x119B20)
// 3. LZ解压(scratch → 0x14CA8)        ← 算法见 1.3
// 4. 尾部区域 u32 XOR K               ← 本样本 K=0 禁用
// 5. 绝对地址补丁 n 条                 ← 本样本 n=0 禁用
// 6. __clear_cache + munmap(scratch)
```

另有一处反篡改：桩内联标记字符串 `"__b_a_n_g_c_l_e__check1234567_"`（借用 bangcle 风格命名）， memcmp 三段自校验，失败则陷入 `/proc/self/maps` 扫描死循环。

**解压区间的自洽性**：`[0x14CA8, 0x119B20)` 恰好等于第一个 PT_LOAD 的终点，也恰好等于 JMPREL 重定位表的终点——壳只加密代码区，链接器需要的动态符号 / 重定位元数据全部保留明文（否则 so 无法加载）。

非标准 aPLib/NRV，是自定义 LZSS 变体。核心特征：

*   位流 MSB-first；位寄存器低 7 位耗尽时重载 `(byte<<1)|1`——bit0 恒为 1 哨兵，保证 "寄存器非空" 判定不会提前
*   `1` = 字面量；`0` 后跟 gamma 编码值 g：
    *   `g==2` → rep-match（沿用上次偏移），长度种子 = 1 bit
    *   否则 `v = src_byte + (g<<8) - 0x300`；`v==0xFFFFFFFF` 为终止符；偏移 = `(v>>1)+1`
*   长度 gamma：结果非 0 即结束，否则内层循环累加后 `+2`
*   拷贝 **W6+1** 字节（偏移 > 0x500 再 +1），支持重叠拷贝（RLE 语义）
*   终止校验双向精确：输出长度 == 期望尺寸 且 输入恰好耗尽

完整 Python 复现：

```
class LZDepacker:
    def __init__(self, src):
        self.src, self.pos, self.buf = src, 0, 0

    def _bit(self):
        if (self.buf & 0x7F) != 0:
            self.buf = (self.buf << 1) & 0xFFFFFFFF
        else:  # 位耗尽: 重载 (byte<<1)|1, bit0 为哨兵
            self.buf = ((self.src[self.pos] << 1) | 1) & 0xFFFFFFFF
            self.pos += 1
        return (self.buf >> 8) & 1

    def _gamma_off(self):
        m = 1
        while True:
            b = self._bit(); m = ((m << 1) + b) & 0xFFFFFFFF
            if self._bit(): return m                    # 停止位=1
            b = self._bit(); m = (((m + 0x7FFFFFFF) << 1) + b) & 0xFFFFFFFF

    def run(self):
        out, W5 = bytearray(), 1                        # 偏移寄存器初值 = 1
        while True:
            if self._bit():                             # 1 = 字面量
                out.append(self.src[self.pos]); self.pos += 1; continue
            g = self._gamma_off()
            if g == 2:                                  # rep-match: 沿用上次偏移
                w6 = self._bit()
            else:
                w6 = (g << 8) & 0xFFFFFFFF
                byte = self.src[self.pos]; self.pos += 1
                w5 = (byte + w6 - 0x300) & 0xFFFFFFFF
                if w5 == 0xFFFFFFFF: return out         # 终止符
                w6 = 0 if (byte & 1) else 1             # 长度种子 = !(byte&1)
                W5 = ((w5 >> 1) + 1) & 0xFFFFFFFF       # 新偏移
            while True:                                 # 长度 gamma
                b = self._bit(); w6 = ((w6 << 1) + b) & 0xFFFFFFFF
                if w6 != 0: break
                w6 = 1
                while True:
                    b = self._bit(); w6 = ((w6 << 1) + b) & 0xFFFFFFFF
                    if self._bit(): break
                w6 = (w6 + 2) & 0xFFFFFFFF
                break
            n = w6 + (2 if W5 > 0x500 else 1)           # 拷贝 W6+1 字节(大偏移再+1)
            st = len(out) - W5
            for i in range(n): out.append(out[st + i])
```

so 静态解密完整流程（读 ELF → 定位真 DT_INIT → 取桩头 / META → 解压回写）：

```
def unpack_so(data: bytes) -> bytes:
    f = bytearray(data)
    phoff, = struct.unpack_from('<Q', f, 0x20)
    esz, n = struct.unpack_from('<HH', f, 54)
    loads, init = [], None
    for i in range(n):
        t, fl, off, va, pa, fsz, msz, al = struct.unpack_from('<IIQQQQQQ', f, phoff + i*esz)
        if t == 1: loads.append((va, off, fsz))
        if t == 2:                                    # ★ 真动态表按 p_vaddr 定位
            p = v2o(va, loads); entries = {}
            while True:
                tag, val = struct.unpack_from('<QQ', f, p); p += 16
                if tag == 0: break
                entries.setdefault(tag, val)
            init = entries.get(12)                    # DT_INIT
    self_ptr, = struct.unpack_from('<Q', f, v2o(init - 0x10, loads))  # 桩头[自身vaddr]
    src_va,   = struct.unpack_from('<Q', f, v2o(init - 0x08, loads))  # 桩头[SRC]
    stub_size, = struct.unpack_from('<Q', f, v2o(self_ptr, loads))
    mo = v2o(self_ptr + stub_size, loads)             # META 紧跟桩代码
    M = lambda o: struct.unpack_from('<I', f, mo + o)[0]
    dest, size, srclen = M(0x24), M(0x1C), M(0x28)
    dep = LZDepacker(bytes(f[src_va:src_va + srclen]))
    out = dep.run()
    assert len(out) == size and dep.pos == srclen     # 双向精确校验
    f[dest:dest + size] = out
    return bytes(f)
```

**验证**：同一解压器直接套 x86_64 版 libSecShell.so 一次成功（该版本 `DT_INIT=0x163098`，解压 0x12DCE0 字节，尺寸与消耗同样精确匹配）——算法跨 ABI 正确的强证据。armeabi-v7a 为 32 位 ELF，结构解析需另适配。

解密产物回灌 IDA 后：736 个函数、386 个 dynsym 符号恢复，全部下述分析基于明文态。

解密后的 so 被 FSM 状态机混淆支配：控制流全靠 `BR Xn` 跳转表分发。全局 696 个 BR 中 **62 个没有任何出边**——每个黑洞背后藏着整段 case 代码。跳转表形态统一：

```
LDRH W0, [Xtbl, Widx, UXTW#1]     ; u16 表项
ADR  X1, <base>
ADD  X0, X1, W0, SXTH#2           ; target = base + (int16)表项 * 4
BR   X0
```

修复要点：

*   表指针解析**必须按符号名**（`ADD Xt, Xt, #name@PAGEOFF` → `get_name_ea(name)`）；`get_operand_value` 对 `@PAGEOFF` 操作数只返回页内偏移，会得到错误表地址
*   ADRP 与配套 ADD 可能相隔 20+ 条指令（被栈上字符串构造代码穿插），需向前扫描
*   对每个目标地址 `add_cref` + `create_insn`，让 IDA 重新分析

修复 25 个 BR 后 **745 条 case 边重见天日**（函数 722→736），装载链才变得可读。

沿装载链（`JNI_OnLoad → H.attach → 容器装载器 sub_33F78`）定位到解密函数，结构：

```
decrypt(offset, buf, size):
    head = (offset <= 0x1FFFF) ? min(0x20000 - offset, size) : 0;
    if (head > 0):
        RC4(key16) 解密 buf[0:head]     // 标准 RC4, S 初值=identity
        若 offset > 0: 先空跑 offset 轮 PRGA   // 支持分块调用的流续接
    if (0xAC层开关启用 && 剩余 > 0):    // 全局配置开关
        for (i = head; i < size; i++) buf[i] ^= 0xAC;   // NEON 向量化 + 标量尾巴
```

汇编里 KSA 的 i=0 迭代被编译器剥离到循环外（`j = 0 + S[0] + key[0]` 预计算为 `j = key[0]`），从 FSM 跳转表入口追踪汇编容易误读为自定义变体——**用 Hex-Rays 伪代码归一化后再核对，并以公开测试向量自检**：

```
def rc4_ksa(key):
    S = list(range(256)); j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) & 0xFF
        S[i], S[j] = S[j], S[i]
    return S

def prga(S, data):
    S = list(S); i = j = 0; out = bytearray()
    for b in data:
        i = (i + 1) & 0xFF; t = S[i]; j = (j + t) & 0xFF
        S[i], S[j] = S[j], S[i]
        out.append(b ^ S[(S[i] + t) & 0xFF])
    return bytes(out)

# 公开 KAT 向量自检 —— 还原任何标准算法后先做这一步
assert prga(rc4_ksa(b"Key"), b"Plaintext").hex() == "bbf316e8d940af0ad3"
```

密钥派生链（每跳均有反编译 + 原生执行双重证据）：

```
JNI_OnLoad case9:
  FindClass("com/SecShell/SecShell/H")
  → GetStaticFieldID("PKGNAME", "Ljava/lang/String;")   ← 静态字段, stub dex 里明文可见
  → GetStaticObjectField → GetStringUTFChars → strcpy(全局槽)

初始化器 → 密钥跳板(0x7EF18) → 密钥生成器(sub_76E00):
    key16 = BASE16 ⊕ PKGNAME 前 16 字节 UTF-8
    BASE16 = 66 97 6C E8 6D 46 38 B0 09 5A A5 D7 0F CB 9A A0   (16 字节常量, 栈构造)
```

```
BASE16 = bytes.fromhex('66976ce86d4638b0095aa5d70fcb9aa0')

def derive_key(package: bytes) -> bytes:
    key = bytearray(BASE16)
    for i, b in enumerate(package[:16]):
        key[i] ^= b
    return bytes(key)

derive_key(b"com.ixyzh.question")
# = 05f801c6043e41ca6174d4a26ab8eec9
```

`sub_76E00` 内部另有一套 MD5（`sub_74A04`，标准实现：初始向量 0x67452301… + 旋转移位集 32−{7,12,17,22,…} 双重验证）加 4096 字节 per-app 盐（标签 `"g_rkeyumFile0001"`），但在主路径中被最终输出覆盖，不参与密钥。

密钥公式的验证方式（推荐工程做法）：用 Unicorn 映射解密态 so（含重定位）+ malloc/strlen 桩 + 防篡改哨兵同值，**原生执行真实 ARM64 指令**——空串输入得 key=BASE16 原样、5 字节串仅前 5 字节异或，公式确证；再对 0x7D440 变换做原生 ↔ Python 逐字节对照（7 组用例，含 0x20000 边界与 0xAC 开关）。

数据源先澄清：装载器通过 JNI 反射 `java/lang/Class.dexCache.dexFile` 取 ART 已加载的 DexFile 原生结构（begin@+8 / size@+16）——输入就是内存中的完整 classes.dex。导航逻辑：

```
sub_73360(dex, size, &out_ptr, &out_len):
  assert(dex[0..2] == "dex")                  // 壳 stub 魔数
  tail = dex + u32@(dex + 0x20)               // file_size 字段 → 文件尾
  assert(tail[-4..] == "fdex")                // ★ 尾部魔数
  cont = dex + u32@(tail - 8)                 // ★ 回指偏移 → 0xA56EC 容器头
  count = u32@cont                            // 条目数
  // 条目: [总长 u32][名长 u32][段名 ASCII]
  // 输出: 载荷区域 = 条目+8+名长+4, 长度 = 该处 LE u32
```

对应到本样本：容器头 `0xA56EC` 处为 `[count=1][0x2700014][8]["dexdata0"]`，输出区域 = **blob+4**、长度 = `LE_u32@blob[0:4]` = 0x2700000。

**易错点**：blob（0xA5700 起）前 4 字节 `00 00 70 02` 是 LE 尺寸前缀，真正的载荷区域从 **blob+4（文件偏移 0xA5704）** 开始——解密的流对齐起点在这里，差 4 字节则全盘皆错。

```
载荷区域 (0x2700000 B):
0x00000  ┌─────────────────────────────────────────┐
         │ ... 头区(配置数据) ...                    │
0x00FC8  │ 目录: 7 × {BE u32 offset, BE u32 size}   │ ← 目录在 4096 边界顶部
0x01000  ├─────────────────────────────────────────┤
         │ dex[0]  前 0x20000 B = RC4 密文           │ ← 密文以 77 ea 1f 31 01 a8 41 69
         │         其余部分     = 明文               │   开头(即 "dex\n035\0" 的密文形态)
0x721000 ├─────────────────────────────────────────┤
         │ dex[1]  头窗口加密, 其余明文               │
         ├─────────────────────────────────────────┤
         │ ... dex[2] .. dex[6] 同构 ...             │
0x26F5600└─────────────────────────────────────────┘  各 dex 起点均为 0x1000 对齐

目录条目 (BE):
  [0] off=0x000000 size=0x71F748    [4] off=0x1FF5000 size=0x47519C
  [1] off=0x720000 size=0x7AB3C8    [5] off=0x246B000 size=0x248DE0
  [2] off=0xECC000 size=0x7D1E50    [6] off=0x26B4000 size=0x4A600
  [3] off=0x169E000 size=0x9564F4
```

对每个 dex 调用 `decrypt(0, dex, size)`：head = min(0x20000, size) 走 RC4。**其余部分保持明文**（本样本 0xAC 层开关关闭）——两个直接的字节级证据：

*   各 dex 的 `map_list`（尾部）在 blob 里是明文（`11 00 00 00…` 合法条目结构）
*   0x20000 边界处是明文升序 u32 数组（string_ids 表）

这同时解释了对整个 blob 做熵扫描时看到的 "高熵散布块"：不是区域加密，而是**各 dex 边界处的加密头窗口**。

解密后每个 dex 按 DEX 规范校验并修复签名（SHA-1 / Adler-32 壳运行时会重写这两个字段，离线脱壳同样重写以通过解析器校验）：

```
DEX header (0x70 B) 关键字段         解密后期望值
+0x00  magic                        "dex\n035\0"
+0x08  checksum (Adler-32)          ← 需按明文重写
+0x0C  signature (SHA-1, 20B)       ← 需按明文重写
+0x20  file_size                    == 目录条目 size（天然自校验）
+0x24  header_size                  0x70
+0x28  endian_tag                   0x12345678
+0x34  map_off                      < file_size, 指向合法 map_list
+0x38  string_ids_size              数万量级
+0x3C  string_ids_off               0x70
```

```
#!/usr/bin/env python3
"""顶象 SecShell 纯静态脱壳器
用法: python3 unpack.py <apk> <包名> <输出目录>"""
import struct, sys, os, zipfile, hashlib, zlib

BASE16 = bytes.fromhex('66976ce86d4638b0095aa5d70fcb9aa0')

def derive_key(pkg: bytes) -> bytes:
    key = bytearray(BASE16)
    for i, b in enumerate(pkg[:16]): key[i] ^= b
    return bytes(key)

def rc4_ksa(key):
    S = list(range(256)); j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) & 0xFF
        S[i], S[j] = S[j], S[i]
    return S

def prga(S, data):
    S = list(S); i = j = 0; out = bytearray()
    for b in data:
        i = (i + 1) & 0xFF; t = S[i]; j = (j + t) & 0xFF
        S[i], S[j] = S[j], S[i]
        out.append(b ^ S[(S[i] + t) & 0xFF])
    return bytes(out)

def main(apk_path, package, outdir):
    key = derive_key(package.encode())
    dex = zipfile.ZipFile(apk_path).read('classes.dex')

    # ---- 第 1 层: fdex 尾部导航 ----
    tail = struct.unpack_from('<I', dex, 0x20)[0]           # file_size 字段
    assert dex[tail-4:tail] == b'fdex', '无 fdex 尾标记'
    cont = struct.unpack_from('<I', dex, tail - 8)[0]       # 回指容器头
    count = struct.unpack_from('<I', dex, cont)[0]
    body, o, regions = dex[cont:], 4, []
    for _ in range(count):                                  # 条目:[总长][名长][段名]
        total, namelen = struct.unpack_from('<II', body, o)
        rp = o + 8 + namelen                                # 跳过条目头+段名
        rlen = struct.unpack_from('<I', body, rp)[0]        # 区域尺寸前缀
        regions.append((body[o+8:o+8+namelen], cont + rp + 4, rlen))
        o += total + 8

    # ---- 第 2 层: 4096 目录 + 逐 dex 解密 ----
    os.makedirs(outdir, exist_ok=True)
    for name, roff, rlen in regions:
        region = dex[roff:roff + rlen]
        n = 1
        while True:                                         # 目录条目数(向后扩展)
            a, b = struct.unpack_from('>II', region, 0x1000 - 8*(n+1))
            if 0x1000 + a + b > rlen: break
            n += 1
        entries = [struct.unpack_from('>II', region, 0x1000 - 8*n + 8*i) for i in range(n)]
        for i, (eo, es) in enumerate(entries):
            buf = bytearray(region[0x1000+eo : 0x1000+eo+es])
            head = min(0x20000, es)                         # 仅头部窗口 RC4
            buf[:head] = prga(rc4_ksa(key), bytes(buf[:head]))
            assert buf[:4] == b'dex\n', '魔数校验失败'
            buf[0x0C:0x20] = hashlib.sha1(bytes(buf[0x20:])).digest()      # signature
            buf[0x08:0x0C] = struct.pack('<I', zlib.adler32(bytes(buf[0x0C:])) & 0xFFFFFFFF)
            path = f'{outdir}/classes_{name.decode()}_{i+1}.dex'
            open(path, 'wb').write(bytes(buf))
            print(f'{path}  {es:#x} bytes')

if __name__ == '__main__':
    main(sys.argv[1], sys.argv[2], sys.argv[3])
```

运行结果（本样本）：

```
classes_dexdata0_1.dex  0x71f748 bytes   # androidx/support 库
classes_dexdata0_2.dex  0x7ab3c8 bytes   # kotlin/三方库
classes_dexdata0_3.dex  0x7d1e50 bytes   # ★ 业务包 309 个类（App.java 等）
classes_dexdata0_4.dex  0x9564f4 bytes
classes_dexdata0_5.dex  0x47519c bytes
classes_dexdata0_6.dex  0x248de0 bytes
classes_dexdata0_7.dex  0x4a600 bytes
```

全部 7 个 dex：map_list 17-18 条目有效、file_size 精确匹配、jadx 完整反编译。

```
APK classes.dex (41.5MB)
├─ 前段: 壳 stub（明文）: AW/AP/CP/H 壳类
└─ 尾部: [u32][回指]["fdex"] → dexdata0 容器 → 7 × RC4 头窗口加密 DEX

libSecShell.so:
├─ 自解密: 诱饵 .dynamic + DT_INIT 引导桩 + LZ 解压(代码段)
├─ 初始化: SDK 探测 / root 检测 / dlsym 表 / 密钥初始化(BASE16⊕PKGNAME)
├─ 装载链: JNI_OnLoad → H.attach → fdex 导航 → 4096 容器 → 逐 dex 解密
│           → 修复 checksum/file_size/尾标记 → DefineClass
├─ ART hook: Art50/Art90 DexFile 构造桩 + string_ids 掉包(RC4) + 方法级解密
└─ 对抗: na.czl 59 条模拟器路径 / maps 扫描 / 内联自校验标记
```

```
0x14A078  自解密引导桩          0x73360  fdex 容器导航器
0x14CA8   LZ 压缩代码数据       0x7D440  载荷解密器(RC4+0xAC)
0x169A8   INIT_ARRAY[0] SDK探测  0x7EF18  密钥生成跳板
0x1A084   初始化器(dlsym/建表)   0x76E00  密钥生成器(key=BASE16⊕str)
0x330D0   H.attach dex装载循环   0x74A04  标准MD5压缩函数
0x321E4   dex修复+DefineClass    0x131550 per-app 4096B盐
0x33F78   容器装载+解密调度      0x6D7D0  string_ids RC4("1234"标记dex用)
0x7364C   DexFile原生结构读取
```

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)