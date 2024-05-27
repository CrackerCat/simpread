> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281892.htm)

> 一种对 QUIC 协议的抓包方案（拿某知名 APP 练手）

前言
==

   自从转到了安全岗后，Android 这一块的东西也很少碰了，最近感觉工作挺无聊的，想重新涉足这一领域，看看能不能跳个槽什么的！刚好前段时间有个朋友提到某音现在抓包很难，说是因为采用了什么 QUIC 协议，想让我帮忙看看有没有解决方法。所以借此契机就打算拿这个来重新入门了。

    有太长时间没关注过 Android 这块的对抗技术了，某音在这方面一定也是在持续迭代，为了保险起见不被各种对抗技术劝退，所以选了一个它的同门 APP 某茄小说，根据以往的经验，它们两个所用的网络框架和底层网络支持库都是一样的，所以从理论上来讲搞定了某茄小说就等于搞定了某音。

    根据从网上查到的一些公开怎么应对 GQUIC 协议抓包的方法，大多都是在想办法去绕过这个协议。如，修改路由规则来阻止 UDP 上 443 端口的流量使其被动地降级到 HTTPS，或者逆向分析调整 QUIC 配置使其直接降级到 HTTPS。还有一些其他方法，如 Dump SSL 中的 KeyLog，然后再使用 Wireshark 解密。这个方法用来抓取 HTTPS 特别好用，对于 QUIC 来说则只支持 IETF 标准下的 QUIC，本例中的 APP 使用的是早期的 GQUIC，它有着一套自己的加密方式 QUIC Crypto，所以并不适用。

    总的来说这些方法都更注重于实用性偏向于技巧性，虽对于学习来说也不无裨益，但我更倾向于去理解协议，从协议层去解决这个问题。

1. 先来一个简单的协议介绍
==============

      QUIC，音同'quick'，最初由 Google 提出，是一种将 HTTP/2 的多路复用与头部压缩机制封装到数据包中，通过 UDP 进行传输的协议，仅应用于 HTTP 之上。后续随着 HTTP/2 的逐步完善，QUIC 逐步演变为独立协议，并被 IETF 规范化。IETF 提出了两个概念，即 HTTP Over QUIC 和 QUIC 传输层协议，使得 QUIC 可以被用于其他应用层协议之上。自此，QUIC 就分为了 Google 版本和 IETF 版本，前者被称之为 GQUIC，后者被称之为 IQUIC。现在所常说的 QUIC 一般都指后者，HTTP/3 协议也是基于 IQUIC 的。

    在安全性方面，GQUIC 与 IQUIC 采用了两个完全不同的方式，前者使用了自己的一套加密方式 QUIC Crypto，后者还是延续使用了 TLS1.3.

从这张图中可以看到他们之间的区别  
![](https://bbs.kanxue.com/upload/attach/202405/848410_P8VZTH2JFSHQVUC.webp)  
    本文中所提到的 APP 使用的是 GQUIC Q043 协议，所以下文也只会针对这个版本进行分析。

2. 怎么在移动设备上抓取 GQUIC 的流量
=======================

   QUIC 在网络流量中表现形式为 UDP 流量，因此传统的 HTTP 抓包工具无法捕获它，所以需要使用能够直接抓取网卡流量的工具，如 tcpdump 和 Wireshark。要在 Android 设备上捕获网卡流量, 有两种主要方式:

1.  在本地搭建一个类似于软路由的服务, 通过 VPN 连接后, 即可从软路由上进行抓包。
2.  在 Android 设备上安装 tcpdump, 从而直接进行抓包。可以通过 [Termux](https://github.com/termux/termux-app) 来安装。

我所使用的是 waydroid，它用了容器技术使得可以在 Linux 系统上直接运行原生的 Android 环境，因此 Linux 宿主机也可以随意的访问和操作 Android 环境的资源。

    在这里帮张银奎老师的幽兰打一个广告，幽兰是一款 ARM64 的笔记本，其中内置了 waydroid，JTAG 调试接口和 EBPF 等。在里面可以自定义 ARM 的 Android 镜像，用 gdb 调试 Android 中的进程，访问 Android 文件系统，直接与 ARM 处理器通信等，总之让你体验站在上帝的视角看 Android，嘎嘎好用！

    感兴趣的朋友可以了解一下 [Nano Code](https://www.nanocode.cn/#/yl/)。

![](https://bbs.kanxue.com/upload/attach/202405/848410_RYQUX2P6GU2DD5R.webp)

    流量已经能抓到了，下面来看看流量中是个什么东西。

3. GQUIC 的数据包结构
===============

3.1. 整体结构
---------

      GQUIC 中的数据包分为常规数据包和特殊数据包，特殊数据包用于版本协商和连接重置，常规数据包则用于正常的数据传输，因此本文主要分析常规数据包。一个完整的 GQUIC 常规数据包由一个可变长度的数据包头部和一个或多个可变长度的数据帧构成, 头部携带控制信息, 帧负责承载应用数据，整体分层结构如下图所示。

![](https://bbs.kanxue.com/upload/attach/202405/848410_BD774EE97UX2SMQ.webp)

                                                            （抽象图）

    在常规数据包中，除用于建立初始连接的握手包 (Client Hello 和 Server REJ) 外, 所有数据包自第一帧起均均会被加密，对握手包则是进行 HASH 认证。

![](https://bbs.kanxue.com/upload/attach/202405/848410_4YG7KZDGUS2G5MA.webp)

                                                         （未被加密的 CHLO 包）

![](https://bbs.kanxue.com/upload/attach/202405/848410_85SV6XRRKQV9EB9.webp)

                                                         （加密后的常规数据包）

3.2. 数据包剖析
----------

### 3.2.1. 包头部分

    GQUIC 数据包头部包含了几个控制字段，其长度在 1-51 字节之间，具体格式为：

```
     0        1        2        3        4            8
+--------+--------+--------+--------+--------+---    ---+
| Public |    Connection ID (64)    ...                 | ->
|Flags(8)|      (optional)                              |
+--------+--------+--------+--------+--------+---    ---+
     9       10       11        12  
+--------+--------+--------+--------+
|      QUIC Version (32)            | ->
|         (optional)                |                          
+--------+--------+--------+--------+
 
    13       14       15        16      17       18       ...      44
+--------+--------+--------+--------+--------+--------+--------+--------+
|                        Diversification Nonce                          | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+
    45      46       47        48       49       50
+--------+--------+--------+--------+--------+--------+
|           Packet Number (8, 16, 32, or 48)          |
|                  (variable length)                  |
+--------+--------+--------+--------+--------+--------+
 
 51       ...
+--------+--------+--------+--------+--------+--------+
|                     XXX Frame                       |
|                  (variable length)                  |
+--------+--------+--------+--------+--------+--------+
 
51 + prev frame len
+--------+--------+--------+--------+--------+--------+
|                     XXX Frame                       |
|                 optional (variable length)          |
+--------+--------+--------+--------+--------+--------+

```

    Public Flags 字段是一组控制标识，共 8 个字节，具体含义如下：

        0x1：如果该数据包是由客户端发起的，则表示包头中包含了一个 4 字节长度的版本号信息。如果该数据包是由服务端发起的，则表示这是一个特殊的版本协商包。

        0x2：表示这是一个特殊的连接重置包。

        0x4：表示包头中包含了一个 32 字节长度的随机数 Diversification Nonce，这个标志只在由服务端发起的数据包中有效。

        0x8：表示数据包中包含了一个 8 字节长度的 Connetion ID， 由客户端发起的数据包中会携带。

        0x30：表示包头中包含了一个 6 字节长度的包编号。

        0x20： 表示包头中包含了一个 4 字节长度的包编号。

        0x10：表示包头中包含了一个 2 字节长度的包编号。

        0x00：表示包头中包含了一个 1 字节长度的包编号。

        0x40：保留。

        0x80：保留。必须为 0。

    Diversification Nonce 字段是一个随机数，用于密钥的计算，通常出现在 server hello 数据包中。Packet Number 字段表示数据包的序号，从 1 开始依次递增，主要用于 ACK 机制，但不能保证应用数据的流式顺序。其他字段的含义可以通过其名称直接理解，因此不再赘述。

### 3.2.2. 帧部分

     GQUIC 包含了多种类型的数据帧。例如，Stream Frame 用于承载客户端和服务器之间的应用数据；ACK Frame 和 Stop Waiting Frame 用于数据包的可靠传输；Padding Frame 用于满足协议的对齐要求等。但这里的重点是怎么获取客户端与服务端之间传输的应用数据，因此也只会对 Stream Frame 进行分析。若有兴趣了解更多，可参考 Google 放出的官方文档 [QUIC Wire Layout Specification - Google 文档](https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit#heading=h.o9jvitkc5d2g)。

    Stream Frame，指的就是数据流。它是一个独立的数据传输通道，每个流都有一个唯一的 Stream ID。多条流可以同时进行数据传输而互不干扰，因此一个连接中可以存在多个流，也就是多个数据通道，这正是 GQUIC 多路复用机制的实现。其格式如下：

```
  SLEN用于表示Stream ID字段的长度，OLEN用于表示offset字段的长度，
  DLEN用于表示Data Length字段的长度。
  0           1       …                SLEN
+--------+--------+--------+--------+--------+
|Type (8)| Stream ID (8, 16, 24, or 32 bits) |
|        |    (Variable length SLEN bytes)   |
+--------+--------+--------+--------+--------+
 
  SLEN+1  SLEN+2     …                                         SLEN+OLEN  
+--------+--------+--------+--------+--------+--------+--------+--------+
|   Offset (0, 16, 24, 32, 40, 48, 56, or 64 bits) (variable length)    |
|                    (Variable length: OLEN  bytes)                     |
+--------+--------+--------+--------+--------+--------+--------+--------+
 
  SLEN+OLEN+1   SLEN+OLEN+2
+-------------+-------------+
| Data length (0 or 16 bits)|
|  Optional(maybe 0 bytes)  |
+------------+--------------+
 
 SLEN+OLEN+1+DLEND
+-------------------+-------------+
|          Data Block             |
|      (Variable length)          |
+-------------------+-------------+

```

    Type 字段是一个长度为 1 字节的值，包含各种标志，为方便描述所以这里表示为 “1fdooossB”。其中最高位必须为 1，f 位表示 FIN 位，意味着发送结束；d 位表示是否存在 Data Length 字段。接下来的三个 ooo 位表示 Offset 字段的长度，最后两个 ss 位表示 Stream ID 字段的长度。

    Offset 字段是一个可变长度的值，它用于表示这个帧中所携带的数据块在整个数据流中的偏移量。因为基于 UDP 的协议都会存在乱序这个现象，所以 GQUIC 使用了 Stream ID + Offset 的机制来确认一个流中每个数据块的顺序，接收方再根据这个顺序来重新组装数据。

    Data Lenght 字段是一个可选字段，若存在该字段，则表示该帧中所携带的数据块的长度。若不存在该字段，则表示这是最后一个帧，且该数据包中后续所有的内容都是这个帧所携带的数据块。

    Data Block 就是帧所携带的数据，主要分为三种类型：HandShakeMessage、HTTP/2 Frame 和 HttpBody。HandShakeMessage 用于传递双方的握手消息，HTTP/2 Frame 用于传递双方的 HTTP 头部数据，HttpBody 则用于传递双方的 HTTP 主体数据，以下是详细的介绍。

### 3.2.3. HandShakeMessage

   HandshakeMessage 用于客户端和服务器在握手阶段的通信，共有四种类型：由客户端发起的 InChoate CHLO 和 Complete CHLO，以及由服务器发起的 Rejection 和 SHLO。它们通过一个 tag 字段进行区分，如下：

  ![](https://bbs.kanxue.com/upload/attach/202405/848410_N2HGGYWKQZJRHHS.webp)

    InChoate CHLO 表示这是一个不完整的 CHLO 包，它的目的是希望服务器能够提供自身的配置信息，它可包含以下字段：

*   **SNI**（可选）：用于解析 DNS 的域名。
*   **STK**（可选）：服务器先前提供的令牌。
*   **PDMD**：客户端所接受的证书验证算法。
*   **CCS**（可选）：本地通用证书的 Hash 值。
*   **CCRT**（可选）：缓存的服务器证书的 Hash 值。
*   **VER**：所使用的协议版本。
*   **XLCT**：客户端期望服务器使用的叶证书的哈希值。

    Rejection 是服务器在收到客户端发起的 InChoate CHLO 后作出的响应消息，其中包含了服务器自身的配置信息。这些配置信息将用于后续的密钥协商，它可包含以下字段：

*   **SCFG**（可选）：服务器的配置信息，它包含了一个或多个配置项，每个配置项都包含以下内容：
    
    *   AEAD： 服务器所支持的数据加密算法。
        
    *   SCID：该配置的 ID。
        
    *   PUBS：服务器的公钥。
        
    *   KEXS：服务器所支持的密钥交换算法。
        
    *   Orbit：历史遗留。
        
    *   EXPY：该配置项的过期时间。
        
    *   VER：该配置项所支持的版本。
        
*   **STK**（可选）：一个不透明的字节串，客户端需要在下一个 CHLO 包中携带这个值。
    
*   **SNO**（可选）：服务端生成的随机数，客户端需要在下一个 CHLO 包中携带这个值。
    
*   **STTL**（可选）：服务器配置的有效时长（以秒为单位）。
    
*   **CRT**（可选）：服务器其的证书链。
    
*   **PROF**（可选）：用于对服务器证书的验证。
    

   Complete CHLO 表示客户端和服务器已经完成了所有通信细节的协商，并开始进行密钥交换，它相较于 InChoate CHLO 包新增了以下字段：

*    **SCID**：客户端所使用的服务器配置项 ID。
    
*   **AEAD**：客户端所选择的数据加密算法。
    
*   **KEXS**：客户端所选择的密钥交换算法。
    
*   **NONC**：客户端生成的随机数。
    
*   **SNO**（可选）：服务端生成的随机数。
    
*   **PUBS**：客户端的公钥。
    

    SHLO 包是一个加密消息，表示服务器和客户端已经成功了完成初始密钥的协商，并成功建立了连接。相比于 Rejection 包，SHLO 包新增了一个 PUBS 字段，该字段是一个临时公钥。通过这个临时公钥，双方可以在已协商的初始密钥基础上计算出会话密钥，至此整个握手过程完毕。

### 3.2.4. HTTP/2 Frame

    在 GQUIC 协议中，HTTP 头部数据会被压缩传输，这一部分用的是 HTTP/2 中的机制，所以称为 HTTP2/Frame。（有些资料说它使用的不是 HTTP/2，而是 HTTP/2 的前身 SPDY，但 Google 在它自己的官方文档中都是用 HTTP/2 描述的）

    由于 GQUIC 把 HTTP/2 和 QUiC 传输层揉到了一起，这使得 HTTP2 中绝大部分对数据流的管理工作都被 GQUIC 传输层接管了。因此在该协议中，它只使用了 HTTP/2 中的 HEAD_STREAM 类型来处理 HTTP 头部的压缩数据，所以这里只会涉及到 HEAD_STREAM。

HEAD_STREAM 的结构可表示如下：

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+

```

*   **Length** : 帧主体长度。
*   **Type** ： 类型。
*   **Flags** ： 特定的 Type，有一组特定的 flag，以便对 type 做更多约定。
*   **R** : 保留 (1bit)。语义未设置并且必须在发送的时候设置为 0 。
*   **Stream Identifier** : 流标识符。

### 3.2.5. HttpBody

    HttpBody 指的是原始的 HTTP 主体内容数据，没有特定格式。这部分数据可能会被压缩，具体取决于 HTTP 头部中的 Content-Encoding 字段。

### 3.2.6. GQUIC 数据包概括

    至此，有关 GQUIC 中数据包相关的内容大概已经讲完了，相信大家在脑子中都有了一个初步的概念，现在就用一张图来总结一下 GUIC 的数据包格式，如下：

![](https://bbs.kanxue.com/upload/attach/202405/848410_NUWP63GXR3XDJJ3.webp)

4. GQUIC 的通信过程
==============

   如果把通信过程划分为三个生命周期阶段，可以将其概括为建立连接阶段、数据传输阶段和连接终止阶段。在建立连接阶段，双方通过握手交换信息来初始化连接；数据传输阶段主要涉及通过一个或多个数据流传递应用数据。最后，在连接终止阶段，双方协商关闭连接并释放所有相关资源。

4.1. 建立连接阶段
-----------

    在 GQUIC 中该阶段服务器和客户端会使用固定的 Stream ID 为 1 的数据流来交换握手消息（HandShakeMessage），当流建立后，客户端先检查是否存在已缓存的服务器配置信息，如果有则直接进入 0-RTT 阶段，否则进入 1-RTT 阶段，流程大概如下：  
![](https://bbs.kanxue.com/upload/attach/202405/848410_JFGZWZ9YWAK5CQH.webp)  
    详细过程如下：

1.  客户端首先发送一个不完整的 CHLO 包，请求服务器提供加密通信所需的配置信息。
    
2.  服务器核查当前使用的协议版本。如果该版本不被支持，便进入版本协商过程；如果支持，则发送一个包含加密配置信息的 REJ 包。
    
3.  客户端接收 REJ 消息后，根据提供的加密配置选取合适的密钥交换算法，并利用服务器的公钥计算出初始密钥。随后，客户端再发送一个完整的 CHLO 包，其中包含客户端所选用的密钥交换算法，数据加密算法和客户端公钥等。
    
4.  客户端用初始密钥加密并发送 Request 包。
    
5.  服务器使用客户端的公钥计算出初始密钥，再以初始密钥为基础计算出一个临时的公私钥对，服务器使用这个私钥计算出最终的会话密钥。然后，服务器使用初始密钥对 Request 解密，如果能解开则发送一个用初始密钥加密的 SHLO 包，SHLO 包中包含了用于生成会话密钥的公钥。 最后服务器再响应一个用会话密钥加密的 Response。
    
6.  客户端收到 SHLO 包后，使用初始密钥解密以提取临时公钥，并使用该公钥生成会话密钥，再使用会话密钥解密 Response。
    
7.  双方均切换至会话密钥，握手阶段结束，后续开始应用数据的传输。
    

4.2. 数据传输阶段
-----------

   数据传输阶段顾名思义就是双方进行应用数据传输，在 GQUIC 中 它使用了不同的双向流分别传递 HTTP 头部数据和 HTTP 主体数据，其中用于传递 HTTP 头部数据的流 ID 固定为 3，而用于传递 HTTP 主体数据的流 ID 则来自于 HTTP/2 Frame 中的流 ID。在同一个连接上，头部数据和主体数据的传递是同时进行的，如下图所示：  
![](https://bbs.kanxue.com/upload/attach/202405/848410_BPY4936AHJ7AAA6.webp)

    从图上来看，其实 GQUIC 中的多路复用机制并不完全，因为它使用了一个固定的流来传递头部数据，所以少在头部数据的传输过程中还是会存在队头阻塞的问题。

### 4.3. 连接终止阶段

    对本文来说不重要，略。

对于 GQUIC 协议的分析就到先到此为止了，是时候该考虑怎么对数据包进行解密，以及怎么从数据包中提取 HTTP 报文的问题了。

5. 怎么解密 GQUIC 数据包
=================

     GQUIC Crypto 使用了前向安全机制，即双方先计算出一个临时的初始密钥，用于对握手阶段的信息进行加密。后续再基于这个初始密钥，进一步计算出会话密钥，用于加密后续的通信数据。这样做的优点就在于，即使服务器的私钥后续被泄露，也无法解密之前已经完成的通信记录。所以想要解密 GQUIC 数据包，则必须也按照这个步骤先计算出初始密钥，再计算出会话密钥。至于初始密钥和会话密钥的计算过程，Google 在其官方文档中已经罗列出了很详细的步骤，所以这里我直接将它抄录如下：

    Key material is generated by an HMAC-based key derivation function (HKDF) with hash function  SHA-256.  HKDF (specified in [RFC 5869](https://tools.ietf.org/html/rfc5869)) uses the approved two-step key derivation procedure specified in [NIST SP 800-56C](http://csrc.nist.gov/publications/nistpubs/800-56C/SP-800-56C.pdf).

Step 1: HKDF-Extract

The output of the key agreement (32 bytes in the case of Curve25519 and P-256) is the premaster secret, which is the input keying material (IKM) for the HKDF-Extract function. The salt input is the client nonce followed by the server nonce (if any).  HKDF-Extract outputs a pseudorandom key (PRK), which is the master secret. The master secret is 32 bytes long if SHA-256 is used.

Step 2: HKDF-Expand

The PRK input is the master secret. The info input (context and application specific information) is the concatenation of the following data:

1.  The label “QUIC key expansion”
    
2.  An 0x00 byte
    
3.  The GUID of the connection from the packet layer.
    
4.  The client hello message
    
5.  The server config message
    
6.  The DER encoded contents of the leaf certificate
    

Key material is assigned in this order:

1.  Client write key.
    
2.  Server write key.
    
3.  Client write IV.
    
4.  Server write IV.
    

If any primitive requires less than a whole number of bytes of key material, then the remainder of the last byte is discarded.

When the forward-secret keys are derived, the same inputs are used except that info uses the label “QUIC forward secure key expansion”.

When the server’s initial keys are derived, they must be diversified to ensure that the server is able to provide entropy into the HKDF.

Step 1: HKDF-Extract

The concatenation of the server write key plus the server write IV from the found round is the input keying material (IKM) for the HKDF-Extract function. The salt input is the diversification nonce.  HKDF-Extract outputs a pseudorandom key (PRK), which is the diversification secret. The diversification secret is 32 bytes long if SHA-256 is used.

Step 2: HKDF-Expand

The PRK input is the diversification secret. The info input (context and application specific information) is the label "QUIC key diversification".

Key material is assigned in this order:

1.  Server write key.
    
2.  Server write IV.
    

   反推上述步骤可见，想要解密 GQUIC 数据包至少需要四个材料：预主密钥（premaster secret）、CHLO 握手数据包，REJ 握手数据包，和随机数 (Diversification Nonce)。其中后三者都可以通过抓包工具直接获得，但是，预主密钥的生成依赖于各自的私钥和对方的公钥，公钥信息可以从握手数据包中提取，私钥信息则是保密的。因此，解密 GQUIC 数据包的关键实际上就是怎么去获取私钥。当然，获取服务器的私钥不太现实，但是获取客户端的私钥还是有一定操作空间的。

    目前商用的 QUIC 协议大多都是基于已开源的项目，因此在分析采用 QUIC 协议的客户端时，可以先确定其使用的是哪个开源项目后再去分析该项目的开源代码，根据从代码中提取到的特征去逆向分析客户端，使其能直接定位到生成私钥的地方，然后再通过 Hook 获取。至于怎么通过特征去逆向分析，那办法就多了，最简单的莫过于通过字符串去定位，总之对于有经验的人来说就是小 case。

    以某茄 APP 为例，它使用了 Google 的 cronet 库 [GitHub - google/quiche](https://github.com/google/quiche)， 所以通过分析 cronet 的开源代码后通过从代码中提取到的字符串”Key exchange failure“特征，成功地定位到了生成私钥的地方。  
![](https://bbs.kanxue.com/upload/attach/202405/848410_Y5MNC62RY3B2U4V.webp)

    然后再使用 Frida hook，代码如下

```
function OnCreatePrivateKey(imgBase)
{
    var cornetBase = Module.findBaseAddress(imgBase);
    if (cornetBase == null)
    {
        console.log("can't found the cornet\n");
        return;
    }
    var onCreatePrivateKeyFun =  cornetBase.add(0x1E524C);
    if (onCreatePrivateKeyFun == null)
    {
        console.log("bad onCreatePrivateKeyFun");
        return;
    }
    if (onCreatePrivateKeyFun.readU64().toString(16) != "97fff8b8910163e8")
    {
        console.log(`bad feature${ onCreatePrivateKeyFun.readU64().toString(16)}`);
        return;
    }
 
    Interceptor.attach(onCreatePrivateKeyFun, {
        onEnter: function(args)
        {
            var private_key_ptr = new NativePointer(this.context.x0);
            var len = this.context.x1.toUInt32();
            if (len != 32)
            {
                console.log("bad private key");
            }
            else
            {
                var private_key = private_key_ptr.readByteArray(32);
                var private_key_hex = Array.prototype.map.call(new Uint8Array(private_key), x => ('00' +
                x.toString(16)).slice(-2)).join('');
                 
                console.log(`hit OnCreatePrivateKey and the key is ${private_key_hex} the len is ${len}`)
            }
             
        }
    })
    console.log("success found onCreatePrivateKeyFun")
}

```

    现在已经能获取到私钥了，然后再根据上述的步骤去生成密钥就可以对数据包解密了，不会也没关系，后面有开源代码。

6. 怎么解密使用了 0-RTT 握手的数据包
=======================

    在上一节中提到想要解密数据包，则需要预主密钥（premaster secret）、CHLO 握手数据包，REJ 握手数据包和多样随机数的参与。但若客户端采用了 0-RTT 握手方式的话，服务端是不会返回 REJ 数据包的。那么，该如何解决这个问题呢？很简单，在 CHLO 包中有一个名为 SNI 的字段，该字段是服务器的域名，所以当使用一个支持 GQUIC 的客户端，并通过相同版本的协议访问该域名时，服务器就会返回一个 REJ 包。由于 REJ 包中的配置信息是固定的，故可以直接拿来使用。

    还有一个问题就是，SNI 其实是一个可选的字段，也就是说可能会出现没有这个字段的场景。如果真出现这种情况的话，也不用慌，可以先试一下通过 IP 能不能访问到服务器，如果可以的话用 IP 代替域名也能达到同样的效果，但有的服务器禁止直接通过 IP 访问，所以对于这种情况就只能逆向分析了。通常来说，SNI 这个字段都是有的，因为它会涉及 CDN 加速的问题，而且稍微大一点的 APP 都会配置 CDN 加速。

7. 实战
=====

    上面讲了很多东西， 现在要做的就是怎么将这些东西全部给用起来，我在 Google 开源库 [GitHub - google/quiche](https://github.com/google/quiche) 的基础上实现了这些功能，并成功地应用到了某茄 APP 上，效果如下：  
![](https://bbs.kanxue.com/upload/attach/202405/848410_H6KBSWXU5AV8Y9R.webp)

    至于某音的话朋友说和这个一样，不过我没尝试，不想去折腾它的反调试了。

    至于具体的实现，这里就不讲了，不然篇幅实在太大，我会把代码上传到 Github 上，感兴趣的自己去研究吧。

8. The End
==========

     源码地址在[这里](https://github.com/aazhuliang/GQUIC_043_decrypt)（只是一个 Demo），目前只能应用在 GQUIC 协议的 Q043 版本上，如果大家想基于这个源码去扩展的话，则强烈推荐看完 [Google 的官方文档]([QUIC, a multiplexed transport over UDP](https://www.chromium.org/quic/))，然后再根据文档去读活代码。

    最后，这里只讲了怎么解密 GQUIC 的流量，如果是 IQUIC 的话则有更简单的方式，就是文中一开始提到的 Dump SSL 库中的 KeyLog，然后直接使用 Wireshark 解密。这个方法用来解密 HTTPS 也百试不爽，下次在再专门写一篇文章来说吧。

    注意：文中所开源的代码只用作学习与交流，切勿用作其它非法用途。

    码字不易，转载请注明出处。

[阿里云助力开发者！2 核 2G 3M 带宽不限流量！6.18 限时价，开 发者可享 99 元 / 年，续费同价！](https://click.aliyun.com/m/1000393730/)

最后于 1 天前 被执着的追求编辑 ，原因：

[#协议分析](forum-161-1-120.htm)