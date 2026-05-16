> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291228.htm)

> [原创] 爱加密 v4 加固逆向：从 Frida 失效到完全离线解密脱壳

一、前言  

=======

由于太懒好久没写文章了，为了提高下语言能力，并且前一段时间刚好弄了一个用爱加密（ijiami）v4 商业加固的 APK，做了一次完整的静态分析。常规思路是先用 Frida 脱壳再反编译，结果在这个 App 上接连碰壁——Frida 注入 3 秒必死。

使用 objection+frida-dexdump 方案可以进行脱壳 (实测可行)，但是拿到的 dex 文件中的内容比较少，不过这个并不是重点，重点是想了解这里面到底发生了什么，带着猎奇的心理探索一下。

失败本身不是问题，问题是搞清楚为什么失败，以及如何绕过。这篇文章记录了从最初的 Frida 尝试，到最终实现完全不依赖动态执行的离线 DEX 提取的完整过程，包括踩过的坑和得出每个结论的依据。

环境：小米 Android 12（arm64-v8a），KernelSU，IDA Pro 9.0，Python 3.9(够用)。

二、加固方案识别
========

解压 APK 后看到的文件结构：

classes.dex          13 KB   ← 异常小

assets/ijiami.dat    14 MB   ← 高熵载荷

assets/ijiami.ajm     6.2 MB  ← magic: "indl01"

lib/arm64-v8a/libexec.so      622 KB

lib/arm64-v8a/libexecmain.so  153 KB

classes.dex 只有 13KB，反编译一看只有 4 个混淆类：

s/h/e/l/l/A   → Application 代理，attachBaseContext 入口

s/h/e/l/l/C   → ClassLoader 管理

s/h/e/l/l/N   → Native 加载 / 环境检测

s/h/e/l/l/S   → 签名校验

libexec.so 内有字符串 "ijiami SecLLVM compiler 1.7.4.20"，爱加密 v4 无疑。业务代码全在加密的 "ijiami.dat" 里。

三、Frida 脱壳：三连败
==============

尝试一：frida-dexdump

frida-dexdump -n com.szzc -d

静默退出，无任何输出。原因：App 的进程名是应用名（中文），不是包名，"get_all_process()" 匹配失败。改成 spawn 模式同样无效，fork 前后 PID 不一致导致进程名仍然匹配不上。

尝试二：自定义 Frida 脚本 + spawn

写了含 TracerPID bypass 的注入脚本，spawn 启动 App。结果："Java.perform" 回调从未触发，进程在 resume 后约 3 秒内自杀。

关键发现："libexec.so" 在 "JNI_OnLoad" 极早期就执行 Frida 检测，此时 Frida JS runtime 都还没初始化完，注入完全来不及。

尝试三：重命名 frida-server

cp /data/local/tmp/frida-server /data/local/tmp/debugd

/data/local/tmp/debugd &

App 存活时间延长到了 30 秒以上（进程名检测绕过成功），但 attach 阶段仍然死亡。

抓 logcat 观察：死亡前有 "strstr" 调用扫描 "/proc/self/maps"，找到 "frida-agent.so" 的路径字符串后触发 "kill(getpid(), SIGKILL)"。

三次失败的共同根因：ijiami 的 Frida 检测在 JNI_OnLoad 内极早期触发，spawn 模式根本来不及注入任何 bypass。

四、转换思路：root 直读内存
================

既然注入不进去，换个角度：不注入，让 App 自己正常运行完成解密，然后我们去读它的内存。

原理：ijiami 解密完成后，DEX 以明文驻留在 "[anon:dalvik-DEX data]" 内存区段。有 root 权限就能直读 "/proc/pid/mem"，完全不需要注入任何东西。

1. 正常启动 App

adb shell monkey -p com.szzc -c android.intent.category.LAUNCHER 1

2. 等 20 秒（ijiami 解密需要时间）

3. 找 dalvik-DEX data 区段

adb shell su -c "grep'dalvik-DEX'/proc/$(pidof com.szzc)/maps"

输出示例：

7dc1e74000-7dc2250000 r--p 00000000 00:00 0  [anon:dalvik-DEX data]

7dc25a7000-7dc2c44000 r--p 00000000 00:00 0  [anon:dalvik-DEX data]

...（共 9 条）

对每个区段用 "dd" 读取内存，扫描 "dex\n" magic，得到 9 个 DEX 文件，共约 30MB。

这步成功了，但拿到的 DEX 含运行时页对齐填充，大小不精确。更重要的是——我想搞清楚 ijiami 的加密机制，实现仅凭 APK + 缓存文件就能离线还原，不依赖跑 App。

于是开始逆 "libexec.so"。

五、libexec.so：NRV2B 双层打包
=======================

把 "libexec.so" 丢进 IDA，立刻发现 "DT_INIT"（偏移 "0x84BF8"）不是常规业务代码，而是一个解压器。

解压参数表在 "0x84BE4"，5 个 uint32：

[0x84BE4] = 0x00031E24   压缩 block header 基地址

[0x84BE8] = 0x00084BE8   自引用（运行时计算 ASLR slide）

[0x84BEC] = 0x000617B8   页对齐参数

[0x84BF0] = 0x000D7FF0   解压总输出大小（= 0xA8000 字节）

[0x84BF4] = 0x00031E0C   压缩数据起始偏移

ASLR base 计算："slide = runtime_addr(0x84BE8) - 0x84BE8"。

解压流程：

1. "sub_84E0C"：读参数表，准备 src/dst/len

2. "sub_84EE0"：调用 "sub_852CC"（NRV2B 核心）解压两个 block

3. "sub_85220"：修正解压后代码内的 BL 指令偏移

4. "mmap(base+0x30000, 0xA8000, PROT_RW, MAP_ANON|MAP_FIXED)"：用匿名映射覆盖 "0x30000" 处的原始文件映射

5. "blr x4"：跳转到内层代码新的 DT_INIT

这是标准的 "外层 stub 解压内层代码并覆盖自身" 做法，NRV2B 的 get_bit 核心：

-->asm<--

get_bit:

adds w4, w4, w4    ; 左移，MSB → carry（提取的位）

cbz  w4, reload

ret

reload:

ldr  w4, [x0], #4  ; 加载 32-bit LE 字，src += 4

adcs w4, w4, w4    ; 左移 + carry_in；新 carry = 字的 MSB

ret

初始状态 "w4 = 0x80000000"（哨兵，首次调用立即 reload），经典 UCL NRV2B 实现。

-->gap code 问题 <--

内层代码占 vaddr "0x30000"–"0xD8000"（0xA8000 字节），但 IDA 静态分析只能看到 "0x00000"–"0x852DC"，从 "0x852DC" 往后的 0x52D24 字节（"gap code"）在原始文件里是压缩态，IDA 看不到。包括关键的内层解密函数都在这个区域。

获取 gap code 的方法：App 运行时从 "/proc/pid/mem" 读取 "libexec.so" 运行时 base + "0x852DC" 处的数据，保存为 "gap_code.bin"，再在 IDA 用 "load additional binary" 加载到对应 vaddr。

六、ijiami.dat 解密管道：必须搞清楚顺序
=========================

在开始写解密脚本之前，最重要的一步是把调用链搞清楚。这也是整个分析里最容易出错的地方。

通过 IDA 逐函数追踪，最终确认的管道：

ijiami.dat（14 MB，压缩态）

│

 |  sub_68AE8：

├─ skip 40 字节 header

├─ deflate 解压（14MB → 55MB）

├─ sub_6A52C：外层 XOR 去混淆

└─ 结果写入设备：/data/data/<pkg>/files/ia<hash>（55MB 缓存文件）

 | sub_68634（外层 DEX 加载）：

├─ 读取缓存文件 [40:]（55,099,928 字节）

├─ sub_6AC24：内层 S-box 解密

└─ 读尾部索引表 → 按条目提取各 DEX

关键陷阱：我最初以为 "sub_6A52C" 在 "sub_68634" 里调用，反复对 "dec_szzc.bin" 前后各再做一遍外层 XOR 都拿不到正确结果，白耗了不少时间。IDA 反编译确认后才发现它在 "sub_68AE8" 内部、解压之后调用。这个顺序搞错，整个管道就错了。

七、外层 XOR（sub_6A52C）
===================

"sub_6A52C" 的反编译相对干净，算法是纯 XOR，密钥由包名推导：

-->python<--  

import hashlib

EAEC0 = bytes([

   0x37,0x1f,0x01,0x75,0xf1,0xbf,0x01,0x80,

   0x54,0x58,0x66,0x9a,0x4d,0x39,0xda,0xcf,

   0x8b,0xb0,0x43,0xf4,0x06,0xa0,0x66,0x7f,

   0xc4,0xe8,0xbf,0xf9,0x11,0x07,

])

v4  = hashlib.md5(b"com.szzc_IJM_2019").hexdigest().encode()

v40 = bytearray(v4[i+3] ^ EAEC0[i+1] for i in range(27))

作用域只有首尾各约 1024 字节，中间的 ~14.9MB 完全不动：

-->python<--

xpos = [0, 2, 4, 6, 8, 9, 11, 13, 15]

xk   = [v40[1],v40[3],v40[5],v40[7],v40[9],v40[10],v40[13],v40[15],v40[17]]

# 前 1039 字节

for i in range(15):

   buf[i] ^= v40[i+1]

for i in range(64):

   base = 15 + i*16

   for pi, xi in zip(xpos, xk):

       buf[base+pi] ^= xi

# 后 1024 字节

base_last = len(buf) - 1024

for i in range(64):

   base = base_last + i*16

   for pi, xi in zip(xpos, xk):

       buf[base+pi] ^= xi

验证：去混淆后 offset 32 出现字符串 "arsbs.d2x"（"classes.dex" 的字符置换），说明外层 XOR 正确。

ijiami 自定义 ZIP-like 容器

去混淆后的 payload 是 ijiami 私有的 ZIP 变体，与标准 ZIP 的差异：

| 字段           |   标准 ZIP  |         ijiami         |

|----------|-----------|------------|

| EOCD 签名 | `PK\x05\x06` | `PK\x05\x55` |

|   整数宽度  |     2/4 字节     |   3 字节小端   |

主条目名 "arsbs.d2x"（= "classes.dex"），deflate 压缩，14.2MB → 55MB。解压结果即设备上的缓存文件。

八、内层 S-box 解密（sub_6AC24）
========================

算法结构

"sub_6AC24" 含大量 ARM64 NEON 指令，IDA 反编译不直观，但结构可以归纳为三阶段：

输入：dec_szzc.bin[40:]（55,099,928 字节）

Phase 1：前 num_blocks×1024 字节（= 55,099,392 字节）

逐字节 S-box 替换：buf[i] = v65[buf[i]]

Phase 2：剩余 536 字节，分 33 组 ×16 字节

每组在 9 个固定位置 XOR：

pos {0,2,4,6,8,9,11,13,15} ^= {v64[1],v64[3],v64[5],v64[7],v64[9],v64[10],v64[13],v64[15],v64[17]}

Phase 3：最后 8 字节（536 mod 16 = 8），偶数次 XOR 净效果为零

**S-box 构造（v66 → v65 逆表）**：

-->python<--

v66 = bytearray((key_byte + i) & 0xFF for i in range(256))

for i in range(0, 128, 2):

   v66[i], v66[128+i] = v66[128+i], v66[i]   # 偶数下标互换

v65 = bytearray(256)

for i in range(256):

   v65[v66[i]] = i   # 求逆

key_byte 是个问题

S-box 的种子 "key_byte" 来自全局指针 "qword_EF600" 指向的字符串第 4 个字节（"src[3]"）。这个全局指针在运行时由两条路径写入：

- 路径 A：从 "ijm_apkid" 环境变量管道读取（Java 层注入）

- 路径 B："sub_34500" 内 vtable hash 函数计算

静态看不到值。自然想到用包名 MD5："MD5("com.szzc_IJM_2019")[3] = '2' = 0x32"，试了，不对。

暴力搜索：key_byte 只有 256 种。从设备拉取缓存文件，取前 4 字节 "4a cb 5e f0"，这是加密后的 DEX magic。对全部 256 种 key_byte 构造 S-box，检查能否解出 "64 65 78 0a"（"dex\n"）：

-->python<--

for kb in range(256):

   v66 = bytearray((kb+i)&0xFF for i in range(256))

   for i in range(0,128,2): v66[i],v66[128+i] = v66[128+i],v66[i]

   v65 = bytearray(256)

   for i in range(256): v65[v66[i]] = i

   if v65[0x4a]==0x64 and v65[0xcb]==0x65 and v65[0x5e]==0x78 and v65[0xf0]==0x0a:

       print(f"key_byte = 0x{kb:02x}")

唯一解："key_byte = 0x66 = 'f'"。

这也意味着 "qword_EF600" 字符串的第 4 个字节是 'f'，与包名 MD5 无关。ijiami 有意让密钥推导路径与包名解绑，增加静态分析难度。

九、XOR 密钥恢复：利用 DEX 格式不变量
=======================

背景

Phase 1 S-box 覆盖前 55,099,392 字节，9 个 DEX 中的前 8 个（classes.dex 到 classes8.dex）完全落在这个范围内，S-box 解密后即正确。

第 9 个 DEX（classes9.dex）起点在 offset 54,116,064，其最后 460 字节（offset 54,599,532 ~ 54,599,852）落在了 Phase 2 XOR 区域，需要知道全部 9 个 XOR key 字节。

3 个字节（"v64[13]=0xbb, v64[15]=0xe9, v64[17]=0x75"）可由缓存文件名后缀 "aab06c6f624be099"（即 "src[16:32]"）直接推导，剩余 6 个（"v64[1,3,5,7,9,10]"）静态拿不到。

方法：DEX map_list 结构约束

读 classes9.dex header，偏移 52 的 "map_off" 字段 = "983568"。XOR 区域起点 = "55,099,928 - 536 = 55,099,392"，而 classes9.dex 的全局偏移起点是 "54,116,064"，因此 map_list 相对于缓冲区的绝对偏移 = "54,116,064 + 983,568 = 55,099,632"，落在 XOR 区域内（XOR group 15，position 0 处）。

DEX map_list 的格式是完全确定的：

-->c<--

struct map_list {

   uint32_t size;        // 条目数（< 256，高 3 字节必为 0）

   map_item items[];

};

struct map_item {

   uint16_t type;        // item 类型

   uint16_t unused;      // 填充，必为 0

   uint32_t size;        // 条目数量

   uint32_t offset;      // 数据区偏移

};

// items[0].type = 0x0000（HEADER_ITEM，DEX 标准规定第一条必为此）

// items[1].type = 0x0001（STRING_ID_ITEM）

每个不变量唯一确定一个 key 字节，推导如下：

加密字节 raw[2] = map_count 高字节 → 明文 0x00 → v64[3] = raw[2] ^ 0x00 = raw[2]

加密字节 raw[4] = items[0].type 低字节 → 明文 0x00 → v64[5] = raw[4]

加密字节 raw[6] = items[0].unused 低字节 → 明文 0x00 → v64[7] = raw[6]

加密字节 raw[8] = items[0].size 低字节 → 明文 0x01 → v64[9] = raw[8] ^ 0x01

加密字节 raw[9] = items[0].size 次字节 → 明文 0x00 → v64[10] = raw[9]

加密字节 raw[16] = items[1].type 低字节 → 明文 0x01 → v64[1] = raw[16] ^ 0x01

6 个字节全部唯一确定，无枚举。

交叉验证

恢复 6 个 key 后，解密 map_list 并读连续 4 条：

|   条目  |     type    |  count  |       offset       |                 验证              |

|------|--------|-------|------------|--------------------|

| HEADER_ITEM    | 0x0000 |    1    |       0    |                   ✓               |

| STRING_ID_ITEM | 0x0001 | 8,073 |    112.   |         ✓（紧跟 header）   |

| TYPE_ID_ITEM    | 0x0002 | 1,654 | 32,404 |    ✓（= 112 + 8073×4）.   |

| PROTO_ID_ITEM | 0x0003 | 2,205 | 39,020 |   ✓（= 32404 + 1654×4）|

| FIELD_ID_ITEM   | 0x0004 | 3,425 | 65,480 | ✓（= 39020 + 2205×12）|

4 条链式验证全部通过，6 个密钥字节完全确定。

最终全部 9 个 XOR key：

-->python<--

XOR_KEYS = [0x17, 0xdc, 0x36, 0x3f, 0x53, 0x01, 0xbb, 0xe9, 0x75]

#           v64[1]  [3]   [5]   [7]   [9]  [10]  [13]  [15]  [17]

十、索引表与 DEX 提取
=============

解密后的 55MB 缓冲区尾部存储 9-DEX 索引表：

[n-76 : n-4]  9 条 × 8 字节：(size uint32, offset uint32)

[n-4  : n]  uint32 = 9（条目数）

注意字段顺序是 size 在前，offset 在后，不是通常直觉的顺序。我在这里翻车过一次——index 读反后偏移变成了 9 位数的天文数字，"assert dex[0:4] == b'dex\n'" 立刻报错，才发现问题。

修正 DEX checksum 时，SHA-1 必须在 Adler32 之前计算，因为 Adler32 的覆盖范围 "data[12:]" 包含了 SHA-1 签名字段：

-->python<--

import hashlib, zlib, struct

def fix_dex_checksums(dex: bytearray):

   # 1. SHA-1（覆盖 data[32:]）

   sig = hashlib.sha1(bytes(dex[32:])).digest()

   dex[12:32] = sig

   # 2. Adler32（覆盖 data[12:]，此时 sig 已就位）

   cksum = zlib.adler32(bytes(dex[12:])) & 0xFFFFFFFF

   dex[8:12] = struct.pack('<I', cksum)

最终提取结果：

![](https://bbs.kanxue.com/upload/attach/202605/1050618_U58ZN94NAETTDNR.webp)

十一、完整离线解密脚本
===========

把以上所有步骤整合为一个脚本 "decrypt_szzc.py"，输入 "dec_szzc.bin"（从设备拉取的 55MB 缓存文件），输出 9 个 DEX：

![](https://bbs.kanxue.com/upload/attach/202605/1050618_T8FF9CP54GH6YT9.webp)

![](https://bbs.kanxue.com/upload/attach/202605/1050618_SCYJ5QH8557N6N4.webp)

运行结果：

[✓] classes.dex: 9392872 bytes, 7255 classes

[✓] classes2.dex: 8825108 bytes, 7136 classes

...

[✓] classes9.dex: 983788 bytes, 774 classes

十二、总结
=====

ijiami v4 加密方案总览

![](https://bbs.kanxue.com/upload/attach/202605/1050618_DNXWUDTPSAWDJQU.webp)

_免责声明：本文仅用于安全研究和技术交流，请遵守相关法律法规。_

[[培训]《冰与火的战歌：Windows 内核攻防实战》！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)

最后于 17 小时前 被奋斗的小趴菜编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)