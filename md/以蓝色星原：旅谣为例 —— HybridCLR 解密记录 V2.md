> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2060145-1-1.html)

> [md]# 以蓝色星原：旅谣为例 —— HybridCLR 解密记录 V2 在 BW 试玩过蓝色星原 ~~(蓝色原神)~~ 之后，一直想找包解，浑身刺挠（包体流出后，发现 AB 包只是单纯的 ...

![](https://avatar.52pojie.cn/data/avatar/001/71/99/28_avatar_middle.jpg)DNLINYJ _ 本帖最后由 DNLINYJ 于 2025-9-14 16:51 编辑_  

以蓝色星原：旅谣为例 —— HybridCLR 解密记录 V2
-------------------------------

在 BW 试玩过蓝色星原 ~(蓝色原神)~ 之后，一直想找包解，浑身刺挠（

包体流出后，发现 AB 包只是单纯的 UnityCN 特色解密（无聊

但是在解密 `global-metadata.dat` 时，发现被加密了，解密完后头 magic 是 `CODEPHIL`，一眼代码哲学出品

翻文件的时候，发现几个 `CDPH` 开头的文件，同时又有 .NET 的 Metadata 段 magic + `HybridCLR` 的存在，估计这玩意是新的热加载 DLL，故写了这篇文章来记录 ¯\_(ツ)_/¯

> 补: 这篇文章所涉及的加解密与代码哲学新推出与 `HybridCLR` 共同使用的 `Obfuz` 混淆器相关

### 1. 分析 `global-metadata.dat` 解密

在 IDA 中搜索 `CODEPHIL` 或者 `global-metadata.dat`，前者对应 `DecryptMetaData`，后者对应 `il2cpp::vm::GlobalMetadata::Initialize`

![](https://attach.52pojie.cn/forum/202509/14/164559fa0ii29v6kw1pape.png)

**image-1.png** _(63.26 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1MHw4NzFiZmZhYXwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:45 上传

![](https://attach.52pojie.cn/forum/202509/14/164602urohyhhjhhdpeedh.png)

**image-2.png** _(61.73 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1MXw3N2JiOTU1ZnwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:46 上传

`DecryptMetaData` 核心流程

*   清空输出指针：*a2 = 0。
*   校验头魔术 0x1357FEDA（不匹配则不解密，函数最后仍返回 1，输出保持 0）。
*   以 64 字节为块对 [a1+0x148, a1+0x148+total_size) 原地解密；最后一块按实际剩余长度。
*   解密完成后检查 memcmp(a1+0x148, "CODEPHIL", 8)：
    *   若不相等（memcmp != 0）返回 0。
    *   若相等：设置 *a2 = a1 + 0x150，然后返回 1。

分析完 `DecryptMetaData`，可以将 `global-metadata.dat` 构造成结构体如下

```
struct globalMetadata {
    int encryptMagic; // 0x1357FEDA
    int fileLength;
    char keys[256];
    char opcodes[64];
    decryptedGlobalMetadata* data;
};

struct decryptedGlobalMetadata {
    char magic[8]; // CODEPHIL
    void* data;
}

```

对于 `DecryptBlock` 的分析，见后文内容 =w=

### 2. 分析 HybridCLR 对 HotPatch DLL 的加载

在[官方文档](https://www.hybridclr.cn/docs/beginner/quickstart#%E5%8A%A0%E8%BD%BD%E7%83%AD%E6%9B%B4%E6%96%B0%E7%A8%8B%E5%BA%8F%E9%9B%86)中，直接使用了 `Assembly.Load` 来加载 HotPatch 的 DLL

![](https://attach.52pojie.cn/forum/202509/14/164557nn4qsjo8nqne0187.png)

**image.png** _(82.72 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA0OXw4ZmIzYmVmYnwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:45 上传

其自定义后的 IL2CPP 通过下面的调用顺序对原始二进制进行加载

```
int32_t RuntimeApi::LoadMetadataForAOTAssembly(Il2CppArray* dllBytes, int32_t mode)
                                    |
                                    ↓
LoadImageErrorCode Assembly::LoadMetadataForAOTAssembly(const void* dllBytes, uint32_t dllSize, HomologousImageMode mode)
                                    |
                                    ↓
LoadImageErrorCode AOTHomologousImage::Load(const byte* imageData, size_t length)
                                    |
                                    ↓
LoadImageErrorCode RawImageBase::Load(const void* rawImageData, size_t length)

```

所以对自定义 HotPatch DLL 的加载和解密会在 `RawImageBase::Load` 里面

### 3. HotPatch DLL 的加载和解密分析

在 IDA 中定位到 `RawImageBase::Load` (AzurPromilia CBT1 GameAssembly.dll RVA 0x95F380)

![](https://attach.52pojie.cn/forum/202509/14/164604s5aaqzl97rxuqaa1.png)

**image-3.png** _(106.34 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1Mnw3NTRhNzE5OXwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:46 上传

#### 3.1 LoadCDPHHeader

原版 HybridCLR 中, `LoadCDPHHeader` 的位置实际是 `LoadCLIHeader`，所以 `LoadCDPHHeader` 在读取自定义结构的时候，也会读取 `CLI Header`

![](https://attach.52pojie.cn/forum/202509/14/164606gdcdb0caadrzs2xh.png)

**image-4.png** _(65.05 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1M3w4NWIyZTlmM3wxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:46 上传

LoadCDPHHeader 具体流程如下 (伪代码已经经过 LLM 处理)

*   读取 magic、版本 和 格式编号
    
    ```
    // 1. 头与版本
    // * `this->_imageLength` 在 `this+0x10`，先判 `>= 0x100`。
    // * 检查魔术字：`memcmp(this->_imageData, "CDPH", 4) == 0`。
    // * 版本：`*(uint32*)(this->_imageData + 4) == 1`，否则返回 `9`。
    // 2. “密钥块”与固定起始偏移
    // * `_cdphKey = this->_imageData + 8` 保存到 `this+0x0B40`。
    // * `*(uint32*)_cdphKey <= 1`，否则返回 `10`。
    
    if (self->_imageLength < 0x100 || memcmp(self->_imageData, "CDPH", 4) != 0) return 1;
    if (*(uint32_t*)(self->_imageData + 4) != 1) return 9;
    
    self->_cdphKey = self->_imageData + 8;
    if (*(uint32_t*)self->_cdphKey > 1) return 10;
    
    ```
    
*   读取解密用的 8 段 opcodes
    

```
// * 数据读取起点固定为 `data_offset = 0x110`。
// 3. 读取 8 段数据 → `instructions[0..7]`
// * 对 8 个“向量”循环（对象内连续 8 个 `std::vector<uint8_t>`，见下文布局）：
//   * `len = *(uint32*)(image + off)`，数据体起点 `body = off + 4`，终点 `end = body + len`。`end > imageLength` 则失败。
//   * 关键点：它把**目标向量的 `last`（+8）临时写成 `imageBase + len`**，然后用一个特殊拷贝函数完成复制：
//     * `sub_180A520C0(destBegin, srcBegin, endPtr)` 的**语义是拷贝 `endPtr - srcBegin` 个字节**到 `destBegin`。
//     * 于是当 `srcBegin = image + body`、`endPtr = image + len` 时，效果正好是拷贝 `len` 字节。
//   * 如果容量不足，会走两条“扩容”分支之一（`j_psub_1809CABB0 / 1809CAB90`），并把 `end`（+0x10）更新为某个“容量终点”，注意这个 `end` 在实现里还会被 `& 0x7FFF...` 屏蔽最高位（汇编里 `and rax, 7FFFFFFFFFFFFFFFh`）。
//   * 推进 `off = align4(end)`。

    size_t off = 0x110;
    // instructions[0..7]
    for (int k = 0; k < 8; ++k) {
        uint32_t len  = *(uint32_t*)(self->_imageData + off);
        size_t   body = off + 4;
        size_t   end  = body + len;
        if (end > self->_imageLength) return 1;

        auto& vec = self->instructions[k]; // MSVC vector: first,last,end

        // 设 last = imageBase + len（用于 CopySpan 的 end - src 公式）
        vec.last = self->_imageData + len;

        // 如需扩容（比较的是 vec.end 屏蔽符号位后的值）
        if (vec.last > (vec.end & ((void*)~(1ULL<<63)))) {
            if ((int64_t)vec.end < 0) {
                void* newFirst = j_psub_1809CAB90(vec.last, /*grow*/1);
                CopySpan(newFirst, vec.first, vec.last); // 复制旧内容：vec.last - vec.first
                vec.end   = vec.last;
                vec.first = newFirst;
            } else {
                vec.first = j_psub_1809CABB0(vec.first, vec.last, /*grow*/1);
                vec.end   = vec.last;
            }
        }

        // 从镜像拷贝：len = (imageBase + len) - (imageBase + body)
        CopySpan(vec.first, self->_imageData + body, self->_imageData + len);

        off = (end + 3) & ~3ULL;
    }

```

*   构造特定 Table 以检验密钥合法性

```
// 4. 密钥校验
// * 读当前 `off` 处 16 字节到 `output`。
// * 用两组常量 + 计数器生成 0x100 字节工作表（那组 SSE 指令）。
// * `DecryptBlock(buf=table, size=0x100, key=_cdphKey+8, inout=output, n16=0x10)`；
// * 要求 `output == "Hello, HybridCLR"`，否则返回 `1`。

    // 密钥校验
    uint8_t output[16];
    memcpy(output, self->_imageData + off, 16);
    uint8_t table[0x100];
    BuildTableSSE(table); // 汇编里那段 SSE
    DecryptBlock(/*buf=*/table, /*size=*/0x100,
                 /*key=*/self->_cdphKey + 8,
                 /*inout=*/output, /*n16=*/0x10);
    if (memcmp(output, "Hello, HybridCLR", 16)) return 1;

```

*   读取 CLI Header

```
    // 5) 入口与元数据
    *entrypointToken = *(uint32_t*)&self->_imageData[off + 0x10];
    *metadataAddress = *(uint32_t*)&self->_imageData[off + 0x14];
    *metadataSize    = *(uint32_t*)&self->_imageData[off + 0x18];

    // 6) 节表
    uint32_t nsec = *(uint32_t*)&self->_imageData[off + 0x1C];
    const uint8_t* p = &self->_imageData[off + 0x28];    // 实际循环里从 +0x28 之前取 RVA

    for (uint32_t i = 0; i < nsec; i++) {
        // 反编译行为：va_delta = *(p - 8) /*RVA*/ - *(p) /*fileOffset*/
        uint32_t rva       = *(uint32_t*)(p - 8);
        uint32_t fileOff   = *(uint32_t*)(p + 0);
        uint32_t size      = *(uint32_t*)(p + 4);

        section_entry se;
        se.start   = fileOff;
        se.end     = fileOff + size;
        se.va_delta= (int)rva - (int)fileOff;

        self->_sections.push_back(se);
        p += 16;
    }

```

#### 3.2 LoadStreamHeaders / LoadTables

参照 HybirdCLR 原版代码，这部分读取没有做自定义化处理

[LoadImageErrorCode RawImageBase::LoadStreamHeaders(uint32_t metadataRva, uint32_t metadataSize)](https://github.com/focus-creative-games/hybridclr/blob/main/hybridclr/metadata/RawImageBase.cpp#L52)

[LoadImageErrorCode RawImageBase::LoadTables()](https://github.com/focus-creative-games/hybridclr/blob/main/hybridclr/metadata/RawImageBase.cpp#L200)

#### 3.3 PostLoadStreams (解密 #1)

`PostLoadStreams` 在 HybirdCLR 原版代码没有给出定义，在这里它负责将 .NET 元数据的 streams 进行解密操作

其中的 `instructions` 和 `_cdphKey` 正是在 `LoadCDPHHeader` 读取的

> 注: 伪代码中的 `_cdphKey + 8` 是我定义结构体时写错了 无伤大雅 ovo

![](https://attach.52pojie.cn/forum/202509/14/164608mgt56thygbv666qy.png)

**image-5.png** _(78.39 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1NHw3NzFhYjQxMnwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:46 上传

该函数内 `instructions (Opcodes)` 和 streams 的对应关系如下

<table><thead><tr><th>Opcodes 序号</th><th>streams</th><th>说明</th></tr></thead><tbody><tr><td>1</td><td>#Strings</td><td>标识 / 名字池</td></tr><tr><td>2</td><td>#Blob</td><td>签名等二进制数据池（方法签名、字段签名、属性签名…）</td></tr><tr><td>3</td><td>#US</td><td>User String（C# 源里写的 <code>@"..."</code>/ 普通字符串常量）</td></tr><tr><td>5</td><td>#~ / #-</td><td><strong>表流（Tables Stream）</strong> #~ 为优化格式，#- 多用于编辑 / 调试</td></tr></tbody></table>

#### 3.4 PostLoadTables

这段 `PostLoadTables` 很 “vector 内存管理味儿”，作用就是：  
**把一个按 “表 2 行数（rowNum）” 计的辅助数组（字节数组）扩到足够大，并把新扩出来的部分清零，最后把 size 设为 rowNum。**

换句话说：在读取完元数据表后，给 “表 2（索引 2，对应 ECMA-335 的 TypeDef 表）” 准备一块 `rowNum` 字节的工作区，用来存标记 / 状态之类的一维字节表。

#### 3.5 #US 流解密 sub_1806FF780 (解密 #2)

`sub_1806FF780` 的唯一引用存在于 `hybridclr::managed_cdpe_vtbl`，为虚表第五个元素，如下

![](https://attach.52pojie.cn/forum/202509/14/164610rfznjiola5fj905z.png)

**image-6.png** _(51.38 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1NXwyNDc3NGI4ZXwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:46 上传

`sub_1806FF780` (`DecryptUS`) 可以概括为：**从 `#US`（用户字符串）流里按 ECMA-335 的 “压缩无符号整数” 规则读出一个长度前缀，取出后续那段加密的 UTF-16 字节串，用 `instructions[4]` + `_cdphKey` 解密，然后构造成托管字符串返回**。长度小于 0x1000 用栈上缓冲区，反之堆上 malloc 一块。

##### 伪代码（等价）

```
Il2CppString* ReadUserString(managed_cdpe* self, uint32_t offset) {
    const uint8_t* p = &self->_streamUS.data[offset];

    // decode ECMA-335 compressed uint
    uint32_t len, adv;
    if (p[0] < 0x80)        { len = p[0];                adv = 1; }
    else if (p[0] < 0xC0)   { len = ((p[0]&0x3F)<<8)|p[1]; adv = 2; }
    else if (p[0] < 0xE0)   { len = ((p[0]&0x1F)<<24)|(p[1]<<16)|(p[2]<<8)|p[3]; adv = 4; }
    else { Hybrid::Log("bad metadata data. ReadEncodeLength fail"); /* 继续走，len 未定义 */ }

    if (len == 0) return System_Security_Principal_WindowsIdentity__GetTokenName_0_0();

    const uint8_t* src = p + adv;
    uint8_t* buf;
    bool heap = (len >= 0x1000);
    if (heap) buf = (uint8_t*)malloc(len);
    else      buf = stack_buf_0x1000;

    memmove_opt(buf, src, len); // sub_180A520C0

    DecryptData(self->instructions[4].instructions,
                self->instructions[4].count,
                self->_cdphKey,
                buf, len, 0x10);

    Il2CppString* s = il2cpp_string_new_utf16_0((const char16_t*)buf, (len - 1) >> 1);

    if (heap) free(buf);
    return s;
}

```

#### 3.6 解密 TypeDef 表 sub_1807082D0 (解密 #3)

`sub_1806FF780` 的唯一引用存在于 `hybridclr::managed_cdpe_vtbl`，为虚表第八个元素

结论一句话：**`sub_1807082D0` 读取并（按需）解密** TypeDef 表**（表索引 2）中第 `a3` 行，把该行的 6 个列值取出写到 `a2[0..5]` 并返回 `a2`。**  
它用一个 “已解密标记” 字节数组避免重复解密；列宽（2/4 字节）和列内偏移从 `a1->_tableRowMetas[2].a` 的模式描述里取。

##### 做了什么（按顺序）

1.  **一次性解密该行（惰性）**
    
    *   以 `a3-1` 为 0 基行号，在 `a1->unknown[12]` 指向的字节数组里查看该行是否已解密：
        
        ```
        if (!flags[a3-1]) { flags[a3-1] = 1; DecryptBlock(..., row_ptr, row_size); }
        
        ```
        
    *   `row_ptr = &a1->_tables[2].data[row_size * (a3-1)]`
    *   解密材料：`instructions[6].instructions / .count` + `_cdphKey`，长度为整行 `rowMetaDataSize`。
2.  **按 “列模式” 读取 6 列**
    
    *   列模式基址：`meta = a1->_tableRowMetas[2].a`。
    *   每列的描述是**成对字段**：`[width32, offset16]`，在 `meta` 上按 8 字节一组排布：
        
        ```
        第0列: *(u32*)(meta+0)  宽度=2或4；  *(u16*)(meta+4)  偏移
        第1列: *(u32*)(meta+8)          ;  *(u16*)(meta+12)
        第2列: *(u32*)(meta+16)         ;  *(u16*)(meta+20)
        第3列: *(u32*)(meta+24)         ;  *(u16*)(meta+28)
        第4列: *(u32*)(meta+32)         ;  *(u16*)(meta+36)
        第5列: *(u32*)(meta+40)         ;  *(u16*)(meta+44)
        
        ```
        
    *   读取逻辑：如果该列 `width == 2` 就读 `uint16`，否则读 `uint32`；写入 `a2[i]`（扩展为 32 位保存）。
3.  **返回 `a2`**。
    

> 表 2 正好有 6 列，和 ECMA-335 的 **TypeDef** 一致：  
> `Flags`, `TypeName`, `TypeNamespace`, `Extends`(TypeDefOrRef coded index), `FieldList`, `MethodList`。  
> 其中不同列宽（2/4）由堆大小 / 表行数决定（字符串堆大用 4 字节，索引 / 编码索引随目标表行数变化）。

* * *

##### 等价伪代码

```
DWORD* ReadTypeDefRow(managed_cdpe* self, DWORD* out6, int row1based)
{
    const uint64_t flags_base = self->unknown[12];   // 每行1字节“已解密”标记
    const uint32_t rowSize    = self->_tables[2].rowMetaDataSize;
    uint8_t* rowPtr = &self->_tables[2].data[rowSize * (row1based - 1)];

    // 惰性解密
    uint8_t* decFlag = (uint8_t*)(flags_base + (uint32_t)(row1based - 1));
    if (!*decFlag) {
        *decFlag = 1;
        DecryptBlock(
            (char*)self->instructions[6].instructions,
            self->instructions[6].count,
            (char*)self->_cdphKey,
            (char*)rowPtr,
            rowSize);
    }

    uint64_t meta = self->_tableRowMetas[2].a;

    auto RD = [&](int col) -> uint32_t {
        uint32_t width  = *(uint32_t*)(meta + col*8 + 0);
        uint16_t offset = *(uint16_t*)(meta + col*8 + 4);
        return (width == 2)
             ? *(uint16_t*)(rowPtr + offset)
             : *(uint32_t*)(rowPtr + offset);
    };

    out6[0] = RD(0);
    out6[1] = RD(1);
    out6[2] = RD(2);
    out6[3] = RD(3);
    out6[4] = RD(4);
    out6[5] = RD(5);
    return out6;
}

```

* * *

##### 关键信息与推断

*   **`unknown[12]`**：上一函数里我们已确定它被初始化为**长度 = TypeDef 行数**的一维字节数组，作用就是 **“该行是否已解密”** 的标志位。
*   **`_tableRowMetas[2].a`**：是 “表 2” 每列的宽度与偏移描述，按照 6×`{u32 width; u16 offset}` 组成（每组步长 8 字节）。
*   **列宽 2/4 的含义**：完全符合 ECMA-335 的可变宽索引规则（小堆 / 大堆，或编码索引跨表行数变化）。
*   **参数**：`a3` 是 **1 基**的行号；`a2` 必须至少可容纳 6×`DWORD`。
*   **线程安全**：无锁的惰性解密 + 标志位，**多线程并发读取同一行有竞态**（两方可能同时解密 / 写标记）。
*   **越界检查**：本函数**不检查** `a3` 是否 ≤ 行数，也不检查行内偏移 / 宽度是否合法，安全性依赖上游元数据与模式生成的正确性。

#### 3.7 IL Code 解密 sub_1806FAF70 (解密 #4)

`sub_1806FAF70` 为虚表 `hybridclr::managed_cdpe_vtbl` 第六个元素，也是最后一个解密函数，其伪代码如下

![](https://attach.52pojie.cn/forum/202509/14/164612jm202z78jhk7yhfj.png)

**image-7.png** _(100.42 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA1NnxiMTA4MDZhMnwxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:46 上传

*   `sub_1806FAF70` 在通过一串门闸校验（`sub_18070C160`/`sub_18070B380` 比对）后，直接对传入的 `IL Code` 做解密，本函数用 **`instructions[7]`**，随后进入 `sub_18070F190(...)` 做进一步解析 / 分发，更像 “把已解密的 **IL** 交给后续解释 / 加载”。
*   状态推进：`sub_180716F90(a1->unknown, 1)` 像是 “阶段标记 = 已解密”，符合方法体加载流程中的一步。

#### 3.8 总结

通过上面的分析，我们可以得到在 `CDPH Header` 中读取的 `Opcodes` 与实际解密的关系如下表

<table><thead><tr><th>Opcodes 序号</th><th>解密对象</th><th>解密块大小</th><th>说明</th></tr></thead><tbody><tr><td>0</td><td>无</td><td>无</td><td>保留</td></tr><tr><td>1</td><td>#Strings</td><td>0x100 (256)</td><td>解密 Metadata 段的 #Strings 整体</td></tr><tr><td>2</td><td>#Blob</td><td>0x100 (256)</td><td>解密 Metadata 段的 #Blob 整体</td></tr><tr><td>3</td><td>#US</td><td>0x100 (256)</td><td>解密 Metadata 段的 #US 整体</td></tr><tr><td>4</td><td>#US</td><td>0x10 (16)</td><td>解密 Metadata 段的 #US 中的字符串</td></tr><tr><td>5</td><td>#~</td><td>0x100 (256)</td><td>解密 Metadata 段的 <strong>表流（Tables Stream）</strong> 整体</td></tr><tr><td>6</td><td>TypeDef</td><td>typeDefRowMetaDataSize</td><td>解密 <strong>表流（Tables Stream）</strong> 中的 TypeDef 表（表索引 2）</td></tr><tr><td>7</td><td>IL Codes</td><td>0x10 (16)</td><td>解密方法体的 IL Codes</td></tr></tbody></table>

### 4. 解密算法分析 (`DecryptData`/`DecryptBlock`)

总结一句话：这是个 **“指令驱动的字节块变换器”**。  
`DecryptData` 负责把整段数据按固定块长切片，然后对每个块调用一次 `DecryptBlock`；  
`DecryptBlock` 把 `opcode`（一串字节程序）逐条解释执行到 `output` 这个块上，每条指令用到 `key[0..255]` 某个字节和一些常量，对块内**某个索引**的字节做 `XOR / 加减 / 右旋 / 交换 / 组合更新`。

* * *

把真实语义写清楚：

*   `DecryptData(opcodes, opcodes_len, key, data, data_len, block_len)`
    
    *   `opcodes/opcodes_len`：指令序列及长度（之前叫 `buf/buf_size`，容易误解）
    *   `key`：256 字节左右的密钥 / 表（索引到 0..255）
    *   `data/data_len`：要变换（解密 / 加密）的字节串
    *   `block_len`：每次处理的块大小（见你代码里多为 `0x10`）
*   `DecryptBlock(opcodes, opcodes_len, key, block, n)`
    
    *   `block[0..n-1]` 就地被修改

* * *

#### DecryptData 的逻辑（块处理器）

```
void DecryptData(const u8* opcodes, size_t opcodes_len,
                 const u8* key,
                 u8* data, size_t data_len,
                 size_t block_len)
{
    if (!opcodes_len) return;         // 没“程序”就什么都不做
    size_t off = 0;
    while (off < data_len) {
        size_t n = block_len;
        if (data_len - off < block_len) n = data_len - off;   // 最后一个不满块
        DecryptBlock(opcodes, opcodes_len, key, data + off, (u32)n);
        off += block_len;
    }
}

```

要点：**无链路 / 无状态**，每块独立按同一串 `opcodes` 变换（更像 “每块应用同一个程序”，而非标准分组密码的 CBC 等模式）。

* * *

#### DecryptBlock 的 “虚拟机” 指令集（概览）

每条指令是一个 `int8_t opcode = opcodes[i]`。`n = output_size`。函数里所有索引都写成 `IDX = CONST % n` 或 `IDX = (key[K] ± CONST) % n`，保证落在块内。

指令类型大致分为 5 类（在你的代码里分别对应大量 `case`）：

1.  **XOR**
    
    ```
    out[ C1 % n ] ^= key[K] ^ C2;
    
    ```
    
    例：`case 2, 11, 12, 25, 42, 45, 46, 47, 48, ...`
    
2.  **加 / 减（模 256）**
    
    ```
    out[ C1 % n ] = out[ C1 % n ] - key[K] ± C2;  // 或纯 -= key[K] / += 常量
    
    ```
    
    例：`case 1, 3, 4, 5, 8, 17, 21, 23, 28, 33(纯 -= key[65]) ...`
    
3.  **按位循环右移（单字节 ROR）**
    
    ```
    out[ C1 % n ] = ROR8(out[ C1 % n ], (key[K] + delta) & 7);
    
    ```
    
    例：`case 0, 7, 9, 10, 13, 20, 29, 31, 34, 35, 36, 39, ...`
    
    > `__ROR1__` 即对 8 位数做循环右移；`& 7` 保证 0..7 位数。
    
4.  **交换两位置字节**（`LABEL_261` 路径）
    
    ```
    i1 = C1 % n;
    i2 = (key[K] ± C2) % n;
    swap(out[i1], out[i2]);
    
    ```
    
    例：`case 19, 22, 27, 43, 44, 55, 62, 63, 75, 77, 86, 90, 94, 99, 105, 112, 122, 126, -123, -121, -120, -110, -99, -94, -84, -76, -74, -71, -69, -67, -51, -50, -48, -34, -21, -18, -14, -10, -6 ...`
    
5.  **组合更新：先减 1 再影响另一位**（`LABEL_23` → `LABEL_262` 路径）
    
    ```
    i1 = C1 % n;
    i2 = (key[K] ± C2) % n;
    t  = out[i2] - 1;
    out[i1] -= t;     // out[i1] = out[i1] - (out[i2] - 1)
    out[i2]  = t;     // 同时把 out[i2] 自减 1
    
    ```
    
    例：`case 18, 24, 26, 30, 32, 40, 58, 64, 70, 79, 81, 87, 89, 93, 97, 103, 106, 107, 110, 113, 116, 117, 119, 121, 123, 124, 126, -126, -110, -106, -102, -98, -97, -93, -92, -91, -88, -86, -80, -72, -63, -62, -40, -39, -34, -28, -26, -24, -20, -19, -17, -5 ...`
    

> 由于 `opcode` 是 **有符号 8 位**，所以 `128..255` 的字节在 `switch` 中以 `-128..-1` 出现（你看到那些负数的 `case`）。

除此之外，每条指令的**常量**（如 `0xB330B7AF` 等）在运行时都会取模 `n=output_size` 成为块内索引。很多常量会在小块（例如 16 字节）时**映射为同一个下标**，从而**多次叠加**到同一字节上。

* * *

#### 代表性几条指令（逐条翻译）

*   `case 0`  
    `out[ 0xB330B7AF % n ] = ROR8(out[idx], (key[182] + 1) & 7);`
    
*   `case 2`  
    `out[ 0x8CC11BA1 % n ] ^= key[224] ^ 0x23;`
    
*   `case 19`（交换）  
    `i1 = 0x02BF6A70 % n; i2 = (key[115] - 1335899030) % n; swap(out[i1], out[i2]);`
    
*   `case 18`（组合更新）  
    `i1 = 0x8AAEBD71 % n; i2 = (key[112] + 259923827) % n; t = out[i2]-1; out[i1]-=t; out[i2]=t;`
    
*   `case 61`  
    `out[0x3BDE5592 % n] = ROR8(out[idx], (key[13] - 4) & 7);`
    

……（其余同理，都是以上 5 大模板的实例）

* * *

#### 重要性质与安全性提示

*   **块独立、指令重放**：每个块独立变换，且用同一套 `opcodes`；**无随机化 / 反馈**。
*   **操作线性 / 弱非线性**：`XOR/加减/旋转/交换` 这类操作对攻击者非常友好（尤其是短块、常量取模后重复击中同一索引时）。
*   **密钥索引固定**：每条指令使用 `key[固定位置]`；只要拿到 `opcodes` 和 `key`，就完全可逆。
*   **实现方便**：按上面的 5 个模板就能写出可运行的参考实现 / 解密器。

### 5. 总结

首先感谢我的朋友 @66hh ( [52pojie - zbby](https://www.52pojie.cn/home.php?mod=space&uid=1280903) | [Github - 66hh](https://github.com/66hh) )，在对解密逻辑的分析以及解密器的编写中都做出了重大贡献，如果没有他的协助，该文章是不可能出现的 ovo

其次，在文章开头说过，本文所分析的解密是由代码哲学所开发的 `Obfuz` 所生成的，根据该项目的 ReadMe，解密用的 VM，即上文所分析的 `DecryptBlock`，是随机生成的，故该文章无法保证该解密方法适用于所有使用该混淆器的游戏

最后，来一张 dnSpy 解析解密后的 HotPatch DLL 作为结尾吧 ovo

> 解密器不公开 别私信问我了 (

![](https://attach.52pojie.cn/forum/202509/14/164555cz2jcnrjpnotcwpr.png)

**image-8.png** _(275.64 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgwMzA0OHwzMTMyMmM4M3wxNzU3OTk1NjM4fDIxMzQzMXwyMDYwMTQ1&nothumb=yes)

2025-9-14 16:45 上传![](https://avatar.52pojie.cn/data/avatar/000/93/53/52_avatar_middle.jpg)xixicoco code 哲学，分析很牛掰 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) BrutusScipio 解包看资源吗？不过这是否已实际构成法律风险![](https://avatar.52pojie.cn/images/noavatar_middle.gif)不再吃药惹 加油，大佬就是大佬 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zyzoicq 学习个思路，也不知道 干什么用啊![](https://avatar.52pojie.cn/images/noavatar_middle.gif)哀默余生

> [zyzoicq 发表于 2025-9-15 09:15](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=53863028&ptid=2060145)  
> 学习个思路，也不知道 干什么用啊

可以当内鬼，做外挂，做私服等 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) yunxin0yu 太强啦！！！！    逆向大佬！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)IcePlume 学一下蓝色原神的解密思路 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) jackyawang 那些外挂就是这么来的么?![](https://avatar.52pojie.cn/images/noavatar_middle.gif)UFOfeifei  
学一下蓝色原神的解密思路