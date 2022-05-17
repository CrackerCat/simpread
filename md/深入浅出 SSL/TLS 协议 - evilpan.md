> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [evilpan.com](https://evilpan.com/2022/05/15/tls-basics/)

> 有没有那么一个人，几乎每天都在你身边，但某天发生一些事情后你会突然发现，自己完全不了解对方。

有没有那么一个人，几乎每天都在你身边，但某天发生一些事情后你会突然发现，自己完全不了解对方。对于笔者而言，这个人就是 TLS，虽然每天都会用到，却并不十分清楚其中的猫腻。因此在碰壁多次后，终于决定认真学习一下 TLS，同时还是奉行 Learning by Teaching 的原则，因此也就有了这篇稍显啰嗦的文章。

关于 SSL/TLS 和 HTTPS 这些基本概念也就不多废话了，相信看笔者文章的朋友都有一定基础，因此本文的重点主要放在其中的握手流程。下文分别以目前最为常用的 TLS1.2 和 TLS1.3 协议分别进行分析，涉及到对称加密和椭圆曲线加密的相关介绍可以参考之前的一些文章，比如:

*   [对称加密与攻击案例分析](https://evilpan.com/2019/06/02/crypto-attacks/)
*   [椭圆曲线加密与 NSA 后门考古](https://evilpan.com/2020/05/17/ec-crypto/)
*   [Elliptic Curve Cryptography: a gentle introduction](https://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/) (强烈推荐这篇)

在深入握手细节之前我们可以先抓一个 HTTPS 的包去对整个握手流程有个大致感受，如下所示:

 [![](https://img-blog.csdnimg.cn/cb79a45a21be427d938d11316eedff57.png)](https://img-blog.csdnimg.cn/cb79a45a21be427d938d11316eedff57.png "TLS1.2") TLS1.2

其中绿色部分是为了便于理解已经解密了的 http-over-tls 明文。从上面的数据包中我们可以得到一些初步印象:

*   从握手开始，到发送第一个数据包之前，客户端与服务端之间经历了 5 次传输；
*   每次传输中带有一个或者多个`数据包`；

这里的`数据包`称为 **TLS Record**，每个 Record 都是典型的 TLV (Type-Length-Value) 结构，而其中 Value 的具体类型又与 Record 的类型有关。

Record 的结构如下:

```
struct {
    uint8 major;
    uint8 minor;
} ProtocolVersion;

enum {
    change_cipher_spec(20), alert(21), handshake(22),
    application_data(23), (255)
} ContentType;

struct {
    ContentType type;
    ProtocolVersion version;
    uint16 length;
    opaque fragment[TLSPlaintext.length];
} TLSPlaintext;


```

在实际的网络传输中，一个 Record 可以分割为多个 Fragment，每个 Fragment 的大小最多不超过 `2^14 = 16384`，而 Record 的 length 字段为 `uint16`，最大可以到 65535 字节。之所以这么设计是因为对于加密的数据，解密方需要收到整个 Record 才能开始解密，太大会影响解密性能，从而增加交互延时。

从上述定义可以看到 Record 的前两字节是用于定义协议版本，但是从上图我们发现 TLS 1.2 对应的版本为 0303，这乍看起来有点别扭，但其实是历史发展的结果。历史上 TLS 由 SSL 进化而来，通常也统称为 SSL/TLS，因此版本对应关系分别是:

*   SSL 3.0 -> 0300
*   TLS 1.0 -> 0301
*   TLS 1.1 -> 0302
*   TLS 1.2 -> 0303
*   TLS 1.3 -> 0304
*   …

只需要记住这个版本号即可，并不是很重要。后文也会将 SSL、TLS、SSL/TLS 混用，如无特殊说明，都是指代当前分析的 TLS 协议。

另外值得一提的是 Record 类型，在 TLS 1.2 中主要有 4 个，基本上都出现在上述 Wireshark 截图里了。其中在实际传输明文 (HTTP) 数据之前的大部分 Record 都是 `handshake` 类型的子类型，但 `change_cipher_spec` 却是一个单独的 Record 类型，并不属于 `handshake`，这点后面也会提到。

下面，我们就来开始分析 TLS 握手的具体实现。

注: 下文中所使用到的公、私钥以及相关工具都来源于以下的仓库，并且相关介绍也参考了其中的流程，因此感兴趣的可以直接通过原文去进行学习:

*   [The Illustrated TLS 1.2 Connection: Every byte explained](https://tls12.ulfheim.net/)
*   [The Illustrated TLS 1.2 Connection - Github](https://github.com/syncsynchalt/illustrated-tls)

在 TLS 握手中，总是以客户端的 ClientHello 为起始，就像 TCP 握手总是以 SYN 为起始一样，告诉服务器我们想建立一个 TLS 链接。在 ClientHello 请求的结构如下:

```
struct {
    ProtocolVersion client_version;
    Random random;
    SessionID session_id;
    CipherSuite cipher_suites<2..2^16-2>;
    CompressionMethod compression_methods<1..2^8-1>;
    select (extensions_present) {
        case false:
            struct {};
        case true:
            Extension extensions<0..2^16-1>;
    };
} ClientHello;


```

`client_version` 指客户端版本，值为 `0x0303`，表示 TLS 1.2，前面已经说过。值得一提的是该值与 Record 中的版本不一定一致，后者由于兼容性的原因通常会设置为一个较旧的版本 (比如 TLS 1.0)，服务端应当以 ClientHello 中指定的版本为准。

`random` 是客户端本地生成的 **32 字节** 随机数，在 [RFC5246](https://datatracker.ietf.org/doc/html/rfc5246) 中提到随机数的前四字节应该是客户端的本地时间戳，但后来发现这样会存在[针对客户端或者服务端的设备指纹标记](https://tools.ietf.org/html/draft-mathewson-no-gmtunixtime-00)，因此已经不建议使用时间戳了。为方便后续计算，这里将该随机数固定为:

```
00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f


```

`session_id` 主要用于恢复加密链接，需要客户端和服务端同时支持。由于秘钥协商的过程中涉及到很多费时的操作，对于短链接而言将之前协商好的加密通道恢复可以大大减少运算资源。如果服务器支持恢复会话，那么后续可以直接进入加密通信，否则还是需要进行完整的握手协商。该字段的长度是可变的，占 1 字节，也就是说数据部分最多可以长达 255 字节。

`cipher_suites` 表示客户端所支持的加密套件，带有 2 字节长度字段，每个加密套件用 2 字节表示，且优先级高的排在前面。使用 `openssl` 可以查看实现的加密套件列表，如下所示:

```
$ openssl ciphers -V | column -t
0x13,0x02  -  TLS_AES_256_GCM_SHA384         TLSv1.3  Kx=any       Au=any    Enc=AESGCM(256)             Mac=AEAD
0x13,0x03  -  TLS_CHACHA20_POLY1305_SHA256   TLSv1.3  Kx=any       Au=any    Enc=CHACHA20/POLY1305(256)  Mac=AEAD
0x13,0x01  -  TLS_AES_128_GCM_SHA256         TLSv1.3  Kx=any       Au=any    Enc=AESGCM(128)             Mac=AEAD
0xC0,0x2C  -  ECDHE-ECDSA-AES256-GCM-SHA384  TLSv1.2  Kx=ECDH      Au=ECDSA  Enc=AESGCM(256)             Mac=AEAD
0xC0,0x30  -  ECDHE-RSA-AES256-GCM-SHA384    TLSv1.2  Kx=ECDH      Au=RSA    Enc=AESGCM(256)             Mac=AEAD
0x00,0x9F  -  DHE-RSA-AES256-GCM-SHA384      TLSv1.2  Kx=DH        Au=RSA    Enc=AESGCM(256)             Mac=AEAD
...


```

每个加密套件包含一个秘钥交换算法、一个认证算法、一个对称加密算法和一个用于完整性校验的 MAC 算法。例如后文中协商出的加密套件 `0xC0,0x13` 表示 `ECDHE-RSA-AES128-SHA`。

`compression_methods` 表示客户端所支持的一系列压缩算法。数据需要先压缩后加密，因为加密后的数据通常很难压缩。但是压缩的数据在加密中会受到类似 [CRIME](https://en.wikipedia.org/wiki/CRIME) 攻击的影响，所以当前几乎所有的实现都默认禁用了压缩，而且在新版本的 TLS 中也将压缩的特性移除了。

`extensions` 是 TLS 具备拓展性的一个功能，用于告知服务端一些额外的信息或者启用某些新的特性。比如客户端所支持的 TLS 版本列表，支持的椭圆曲线种类，支持的签名算法等。其中一个常用的拓展为 SNI(Server Name Indication)，包含了服务器的明文域名，主要用于后端服务器的反向代理。

服务器在收到 ClientHello 后必须回应 ServerHello 进行下一步握手协商，否则就需要回复 Alert 类型的 Record 告诉客户端中断握手并关闭连接。

ServerHello 的类型和 ClientHello 基本一致，差别在于回应的 `cipher_suite` 和 `compression_method` 是选择后的固定值，而不是列表。另外需要注意的是服务端返回的 ServerHello 中的拓展必须是客户端中所提供的拓展。假设服务端返回的 `cipher_suite` 为 0x0c13 (ECDHE-RSA-AES128-SHA)，返回的随机数为:

```
70 71 72 73 74 75 76 77 78 79 7a 7b 7c 7d 7e 7f 80 81 82 83 84 85 86 87 88 89 8a 8b 8c 8d 8e 8f


```

随后服务端接着返回自身的证书信息，X509 格式，使用 ASN.1 DER 编码。通常服务器会返回多个证书，因为当前域名往往不是由根证书直接签名的，而是使用由于根证书所签名的次级证书去签发具体域名的证书。

如果使用了多级证书，那么返回的证书列表中第一个必须是对应域名的证书，而后每个证书都是前一个证书的 issuer，且最后一个证书是由系统中某个根证书签发的，注意根证书本身并不会一起返回。

以 baidu.com 为例，实际返回的证书列表如下:

```
$ openssl s_client -connect baidu.com:443
CONNECTED(00000006)
...
---
Certificate chain
 0 s:/C=CN/ST=Beijing/O=BeiJing Baidu Netcom Science Technology Co., Ltd/CN=www.baidu.cn
   i:/C=US/O=DigiCert Inc/CN=DigiCert Secure Site Pro CN CA G3
 1 s:/C=US/O=DigiCert Inc/CN=DigiCert Secure Site Pro CN CA G3
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root CA
---


```

实际返回了两个证书，最后一个证书由 `DigiCert Global Root CA` 签发。如果服务端需要校验客户端证书的话，随后会发送一个 `Certificate Request` 请求，然后客户端返回对应的 `Client Certificate` 进行一轮额外的信息交换，当然这一步是可选的。

服务器会在 server Certificate 消息之后发送 Server Key Exchange 消息，提供用于 ECDH 秘钥交换的信息。

关于 ECDH 的原理可以阅读文章开头的参考文章，简单来说，ECDH 可以在通信媒介不可信的情况下安全地完成秘钥交换。假设 A、B 双方的公私钥分别是 PA、SA，PB、SB，那么有:

双方只需要知道对方的公钥，可以在不暴露私钥的情况下实现信息的交换，防止中间人攻击，所交换的信息就是后续使用的对称加密秘钥。

更进一步，为了避免未来私钥泄露导致以前的通信被解密，通常交换时并不直接使用原始公私钥，而是一个随机生成的新公私钥对，只需要用原始私钥进行认证。这种交换方式也称为 ECDHE，其中 `E` 表示 `Ephemeral`，而这种做法所带来的称为 Forward Security，即前向安全。

令服务端端选择的椭圆曲线为 `x25519`，那么首先需要生成一个临时私钥，长度为 32 字节，只要随机生成即可，这里假设为:

```
909192939495969798999a9b9c9d9e9fa0a1a2a3a4a5a6a7a8a9aaabacadaeaf


```

那么对应的公钥则是:

```
9fd7ad6dcff4298dd3f96d5b1b2af910a0535b1488d7f8fabb349a982880b615


```

可以通过 openssl 计算得出:

```
$ openssl pkey -noout -text < server-ephemeral-private.key

X25519 Private-Key:
priv:
    90:91:92:93:94:95:96:97:98:99:9a:9b:9c:9d:9e:
    9f:a0:a1:a2:a3:a4:a5:a6:a7:a8:a9:aa:ab:ac:ad:
    ae:af
pub:
    9f:d7:ad:6d:cf:f4:29:8d:d3:f9:6d:5b:1b:2a:f9:
    10:a0:53:5b:14:88:d7:f8:fa:bb:34:9a:98:28:80:
    b6:15


```

Server Key Exchange 的消息格式如下所示:

```
struct {
select (KeyExchangeAlgorithm) {
    case dh_anon:
        ServerDHParams params;
    case dhe_dss:
    case dhe_rsa:
        ServerDHParams params;
        digitally-signed struct {
            opaque client_random[32];
            opaque server_random[32];
            ServerDHParams params;
        } signed_params;
    case rsa:
    case dh_dss:
    case dh_rsa:
        struct {} ;
        /* message is omitted for rsa, dh_dss, and dh_rsa */
    /* may be extended, e.g., for ECDH -- see [TLSECC] */
};
} ServerKeyExchange;


```

不同的加密套件有不同的格式，由于我们的是 `dhe_rsa`，因此在消息中应当包含椭圆曲线参数，以及 `ClientRandom+ServerRandom+参数` 的签名信息。使用原始私钥来计算最终的签名信息:

```
$ cat client_random >> /tmp/compute
$ cat server_random >> /tmp/compute
# 03 -> named_curve; 001d -> curve x25519
$ echo -en '\x03\x00\x1d' >> /tmp/compute
$ echo -en '\x20' >> /tmp/compute # 临时私钥长度 32 字节
$ cat server-ephemeral-public.key >> /tmp/compute # 上述生成的临时公钥
$ openssl dgst -sign server.key -sha256 /tmp/compute | hexdump
0000000 04 02 b6 61 f7 c1 91 ee 59 be 45 37 66 39 bd c3
... snip ...
00000f0 7d 87 dc 33 18 64 35 71 22 6c 4d d2 c2 ac 41 fb


```

服务端表示自己这一边的握手已经完成，接下来就等待客户端计算相关握手信息了。

客户端收到 ClientKeyExchange 后，得知服务器选择了 x25519 曲线，那么也类似的生成响应临时秘钥。随机生成 32 字节私钥:

```
202122232425262728292a2b2c2d2e2f303132333435363738393a3b3c3d3e3f


```

对应的公钥计算方法与 Server Key Exchange 中介绍的一样:

```
358072d6365880d1aeea329adf9121383851ed21a28e3b75e965d0d2cd166254


```

Client Key Exchange 的数据格式如下:

```
struct {
    select (KeyExchangeAlgorithm) {
        case rsa:
            EncryptedPreMasterSecret;
        case dhe_dss:
        case dhe_rsa:
        case dh_dss:
        case dh_rsa:
        case dh_anon:
            ClientDiffieHellmanPublic;
    } exchange_keys;
} ClientKeyExchange;


```

其格式相对简单，对于我们选择的加密套件而言只需要包含临时生成的 ECDH 公钥。注意此处与 Server Key Exchange 不同，并没有对客户端的公钥进行签名，也就是说可以被中间人进行替换。不过协议设计的时候已经考虑到了这一点，因为此时双方已经有足够的信息去协商秘钥并且进行验证了，通过后文的计算过程也可以确认这一点。

该数据包告诉服务器客户端已经计算好了共享秘钥，并且后续客户端发送给服务器的数据都将使用共享秘钥进行加密。在下一个版本的 TLS 中该数据包类型将会被移除，因为加密数据是可以通过数据类型推断的。

那么，客户端是如何计算出共享秘钥的呢？目前客户端所已知的数据为:

*   client_random
*   server_random
*   server-ephemeral-public.key
*   client-ephemeral-private.key

首先根据前文对 ECDH 的介绍，通过对方的公钥和自己的私钥，可以计算出一个共同秘钥，这里称之为 `PMS(Pre-Master-Secret)`，具体计算方法可以参考 [curve25519-mult.c](https://tls12.ulfheim.net/files/curve25519-mult.c):

```
$ gcc -o curve25519-mult curve25519-mult.c
$ ./curve25519-mult client-ephemeral-private.key \
                    server-ephemeral-public.key | hexdump

0000000 df 4a 29 1b aa 1e b7 cf a6 93 4b 29 b4 74 ba ad
0000010 26 97 e2 9f 1f 92 0d cc 77 c8 a0 a0 88 44 76 24


```

实际上服务端计算出的共享秘钥也是一样的:

```
$ ./curve25519-mult server-ephemeral-private.key \
                    client-ephemeral-public.key | hexdump

0000000 df 4a 29 1b aa 1e b7 cf a6 93 4b 29 b4 74 ba ad
0000010 26 97 e2 9f 1f 92 0d cc 77 c8 a0 a0 88 44 76 24


```

该共享秘钥计算过程只涉及自身私钥和对方的公钥，为了进一步将共享秘钥关联当当前会话中，需要为其加入双方的随机数，当然不能直接相加，需要增加随机性，因此使用到了一个伪随机函数，称为 PRF(pseudorandom function)。其计算方式如下:

```
seed = "master secret" + client_random + server_random
a0 = seed
a1 = HMAC-SHA256(key=PreMasterSecret, data=a0)
a2 = HMAC-SHA256(key=PreMasterSecret, data=a1)
p1 = HMAC-SHA256(key=PreMasterSecret, data=a1 + seed)
p2 = HMAC-SHA256(key=PreMasterSecret, data=a2 + seed)
MasterSecret = p1[all 32 bytes] + p2[first 16 bytes]


```

所得到的的 48 字节拓展秘钥称为主密钥 (Master Secret)，其值为:

```
916abf9da55973e13614ae0a3f5d3f37b023ba129aee02cc9134338127cd7049781c8e19fc1eb2a7387ac06ae237344c


```

最后，我们需要将该主密钥进行拓展 (至任意长度)，并将结果的不同部分分别用作不同秘钥，如下所示:

```
seed = "key expansion" + server_random + client_random
a0 = seed
a1 = HMAC-SHA256(key=MasterSecret, data=a0)
a2 = HMAC-SHA256(key=MasterSecret, data=a1)
a3 = HMAC-SHA256(key=MasterSecret, data=a2)
a4 = ...
p1 = HMAC-SHA256(key=MasterSecret, data=a1 + seed)
p2 = HMAC-SHA256(key=MasterSecret, data=a2 + seed)
p3 = HMAC-SHA256(key=MasterSecret, data=a3 + seed)
p4 = ...
p = p1 + p2 + p3 + p4 ...
client write mac key = [first 20 bytes of p]
server write mac key = [next 20 bytes of p]
client write key = [next 16 bytes of p]
server write key = [next 16 bytes of p]
client write IV = [next 16 bytes of p]
server write IV = [next 16 bytes of p]


```

最终秘钥分成了 6 个部分，分别是客户端和服务端的 MAC 秘钥、数据加密秘钥和初始向量。这里涉及到几个有趣的问题，比如:

*   为什么客户端和服务端要使用不同的数据加密秘钥？
*   为什么客户端和服务端要使用不同的 MAC 秘钥？
*   为什么要单独指定 IV？

根据 RFC5246 中的介绍，使用不同的 MAC 秘钥是为了防止来自一方的数据被注入到另一方中；对于使用流密钥加密的情况，客户端和服务端使用不同的秘钥也能防止秘钥重用攻击。

在 TLS 1.0 中的 CBC 使用了前一部分 Record 的数据作为 IV 导致了选择明文攻击 (chosen plaintext attack)，因此这在新版本中的 TLS 协议明确指定了 IV 的生成方法。注意这个 IV 只有部分需要额外指定 IV 的 AEAD 算法会用到。

总而言之，通过 ECDHE 秘钥交换，客户端计算出了下述秘钥:

```
client MAC key: 1b7d117c7d5f690bc263cae8ef60af0f1878acc2
server MAC key: 2ad8bdd8c601a617126f63540eb20906f781fad2
client write key: f656d037b173ef3e11169f27231a84b6
server write key: 752a18e7a9fcb7cbcdd8f98dd8f769eb
client write IV: a0d2550c9238eebfef5c32251abb67d6
server write IV: 434528db4937d540d393135e06a11bb8


```

> PS: ChangeCipherSpec 消息并不是 Handshake(22) 消息的子结构，而是一个单独的消息，与 Handshake 并列，类型为 20。这是为了解决握手期间管道阻塞的问题。

[](#client-handshake-finished)Client Handshake Finished
-------------------------------------------------------

该数据包告诉服务器，客户端的握手流程也已经完成。同时，还携带了一部分加密数据，所加密的内容称为 **Verify Data**，用以验证握手成功且没有被中间人修改过。

**Verify Data** 的内容是该消息之前的所有握手包的 HASH 经过 HMAC 计算出来的一个 12 字节数据，其计算方法为:

```
seed = "client finished" + SHA256(all handshake messages)
a0 = seed
a1 = HMAC-SHA256(key=MasterSecret, data=a0)
p1 = HMAC-SHA256(key=MasterSecret, data=a1 + seed)
verify_data = p1[first 12 bytes]


```

在示例数据包中，`verify_data` 值为 `cf919626f1360c536aaad73a`，使用 `client_write_key` 进行加密，服务端收到后使用对应的秘钥进行解密，所使用加解密算法由之前协商的加密套件决定，这里是 `aes-128-cbc`。

服务端收到上述加密后的数据为 (Record Body):

```
404142434445464748494a4b4c4d4e4f
227bc9ba81ef30f2a8a78ff1df50844d
5804b7eeb2e214c32b6892aca3db7b78
077fdd90067c516bacb3ba90dedf720f


```

为了进行验证，服务端使用相同的方式计算出共享秘钥 Pre Master Secret，由 ECDH 的特性以及前文的计算可以看到，服务端和客户端计算出的 PMS 是相同的，因衍生出来的对称加密秘钥、IV、MAC 秘钥也是相同的。

故，服务端收到加密数据后，可以使用协商出来的 client_write_key 对其进行解密:

```
hexdata=227bc9ba81ef30f2a8a78ff1df50844d5804b7eeb2e214c32b6892aca3db7b78077fdd90067c516bacb3ba90dedf720f
# client write key
hexkey=f656d037b173ef3e11169f27231a84b6
# record iv，保存在加密数据之前
hexiv=404142434445464748494a4b4c4d4e4f

$ echo -n $hexdata | xxd -r -p | openssl enc -d -nopad -aes-128-cbc -K $hexkey -iv $hexiv | rax2 -S
1400000ccf919626f1360c536aaad73a
a5a03d233056e4ac6eba7fd9e5317fac
2db5b70e0b0b0b0b0b0b0b0b0b0b0b0b


```

值得注意的是这里使用的 key 是协商的 `client_write_key`，但 IV 并不是 `client_write_iv`，而是一个随机生成的针对当前 Record 的 IV，并且附加到加密数据的前方。

在解密后的数据中， `1400000c` 是 Record 的子协议头部，对应 Handshake/Finish，长度 0x0c 即 12 字节，数据正好是前面计算出的 `verify_data` 的值，即 `cf919626f1360c536aaad73a`。

末尾还有 32 字节的数据，是使用 client mac key 计算的签名，用于确保所接收数据的完整性，计算方法为:

```
### from https://tools.ietf.org/html/rfc2246#section-6.2.3.1
$ sequence='0000000000000000'
$ rechdr='16 03 03'
$ datalen='00 10'
$ data='14 00 00 0c cf 91 96 26 f1 36 0c 53 6a aa d7 3a'
### client MAC key
$ mackey=1b7d117c7d5f690bc263cae8ef60af0f1878acc2
$ echo $sequence $rechdr $datalen $data | xxd -r -p \
  | openssl dgst -sha1 -mac HMAC -macopt hexkey:$mackey

a5a03d233056e4ac6eba7fd9e5317fac2db5b70e


```

再使用 `PKCS#5` 填充至 32 字节即为末尾添加的数据。服务端通过将客户端发送的 verify_data 与自身计算的值进行比对，可确保整个握手流程的完整性；使用 HMAC 校验当前数据可以保证消息没有被中间人篡改。

在这些校验都完成后，服务端给客户端返回 Change Cipher Spec 消息，告知客户端接下来发送的数据都将经过协商秘钥进行加密。当然，和前文一样，其实这条消息是多余的，在 TLS 1.3 中已经被移除了。

[](#server-handshake-finished)Server Handshake Finished
-------------------------------------------------------

此时，服务端已经完成了握手的所有流程，并且也确认这个握手流程没有被中间人篡改，但是客户端还不知道啊！因此，类似于 Client Handshake Finished，服务端也要发送一个加密并验签的数据给客户端，让客户端进行验证并确认整个握手流程的正确性。

发送的数据格式和 Client Finished 几乎一样，只有几点小差异。比如数据使用 `server_write_key` 进行加密 (而不是 client_write_key)，HMAC key 也是类似。另外计算 `verify_data` 与前者相比还多了一个 `Client Finished` 消息，毕竟协议中说的是用于验证 “当前消息前的所有握手消息”。

客户端收到 Server Finished 后，同样进行解密并校验 HMAC，如果确认无误就可以开始发送应用数据了。

> 注: ChangeCipherSpec、Alert 以及其他非 Handshake 的消息都不参与 verify_data 的计算，另外根据 RFC 的规定，HelloRequest 请求也不参与哈希计算。

Application Data 是一个单独类型的 Record (type=23)，准确来说已经不属于握手阶段了，不过这里还是提一下。

该消息格式中主要是使用协商秘钥加密的应用数据，客户端发送的数据使用 client write key 进行加密，服务端返回的数据使用 server write key 进行加密，并且`明文`数据末尾还加了 HMAC 校验数据，使用对应的 MAC key 进行签名，加解密和签名过程和 Client/Server Finished 消息的过程一致。因此每条应用数据都可以保证机密性和完整性。

在新版本的 Wireshark 中，TLS 流量 可以通过指定 keylog 文件进行解密，许多应用如 curl、Chrome 都可以通过指定 `SSLKEYLOGFILE` 环境变量指定 keylog 的位置，例如:

```
$ SSLKEYLOGFILE=~/SSLKEYLOGFILE.txt curl https://example.com
$ cat ~/SSLKEYLOGFILE.txt
CLIENT_RANDOM 54BA544490B18D32B4725E40493A0839CA64AACE14A3E119C0658C49E12A998C EA6B59F06F9E2C219535A02C31FEDBE60053E235A91FC36701B2871477C18E99CE6F2F33144EB6FC28031BB7D51BEF00


```

keylog 文件的格式为 [NSS Key Log Format](https://firefox-source-docs.mozilla.org/security/nss/legacy/key_log_format/index.html)，上述输出是针对 TLS 1.2 的会话秘钥，第一部分为客户端随机数，用于与具体的 TLS 流进行对应；第二部分则是 48 字节的 **Pre Master Secret** 值。根据我们前面的分析，通过该秘钥可以获取到双方的对称加密秘钥，因此就可以解密对应的 TLS 数据了。

由于 TLS 1.3 是在 TLS 1.2 的基础上优化而来的，因此对于与上节实现相同的部分就不再详细介绍了，而只关注其中不同的部分。

总体来看，TLS 1.3 与 TLS 1.2 相比，较大的差异有下面这些:

*   去除了一大堆过时的对称加密算法，只留下较为安全的 AEAD (Authenticated Encryption with Associated Data) 算法；加密套件 (cipher suite) 的概念被修改为单独的认证、秘钥交换算法以及秘钥拓展和 MAC 用到的哈希算法；
*   去除了静态 RSA 和秘钥交换算法套件，使目前所有基于公钥的交换算法都能保证前向安全；
*   引入了 0-RTT(round-trip time) 的模式，减少握手的消息往返次数；
*   `ServerHello` 之后所有的握手消息都进行了加密；
*   修改了秘钥拓展算法，称为 HKDF (HMAC-based Extract-and-Expand Key Derivation Function)；
*   废弃了 TLS 1.2 中的协议版本协商方法，改为使用 Extension 实现；
*   TLS 1.2 中的会话恢复功能现在采用了新的 PSK 交换实现；
*   ……

下面就以一个完整的握手流程去分析这之中的差异，所使用的秘钥和数据包可以在下面的链接中找到:

*   [The Illustrated TLS 1.3 Connection: Every byte explained](https://tls13.ulfheim.net/)
*   [The Illustrated TLS 1.3 Connection - Github](https://github.com/syncsynchalt/illustrated-tls13)

与 TLS 1.2 一样，握手总是以 Client 发送 Hello 请求开始。但正如本节开头所说，TLS 握手时的协议协商不再使用 Handshake/Hello 中的 version 字段，虽然是 1.3 版本，但请求中 version 还是指定 1.2 版本，这是因为有许多 web 中间件在设计时候会忽略不认识的 TLS 版本号，因此为了兼容性，版本号依旧保持不变。实际协商 TLS 版本是使用的是 `Supported Versions` 拓展实现的。

`client_random` 是客户端生成的随机数，这里是:

```
00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f


```

`session_id` 字段在此前的版本中该字段被用于恢复 TLS 会话，不过在 TLS 1.3 中会话恢复使用了一种更为灵活的 PSK 秘钥交换方式，因此这个字段在 TLS 1.3 中是没有实际作用的。

在 Client Hello 消息中，有一个重要的拓展，即 `Key Share`，用于与服务器交换秘钥。前文说到在 TLS 1.3 中，Server Hello 之后的所有消息都是加密的，为了双方能够正确加解密数据，因此在 Client Hello 中通过该拓展告诉服务端自己的公钥以及秘钥交换算法。

这里客户端还是指定了 x25519 椭圆曲线加密，并且生成一个临时私钥:

```
202122232425262728292a2b2c2d2e2f303132333435363738393a3b3c3d3e3f


```

对应的公钥计算方法前文已经说过，这里再重复一遍:

```
$ openssl pkey -noout -text < client-ephemeral-private.key

X25519 Private-Key:
priv:
    20:21:22:23:24:25:26:27:28:29:2a:2b:2c:2d:2e:
    2f:30:31:32:33:34:35:36:37:38:39:3a:3b:3c:3d:
    3e:3f
pub:
    35:80:72:d6:36:58:80:d1:ae:ea:32:9a:df:91:21:
    38:38:51:ed:21:a2:8e:3b:75:e9:65:d0:d2:cd:16:
    62:54


```

随后，该公钥就随着 Client Hello 发送给了服务端。

服务端根据客户端提供的选项，选择一个好自己支持的 TLS 版本以及加密套件，这里选的是 `TLS_AES_256_GCM_SHA384`， 返回的 `server_random` 为:

```
70 71 72 73 74 75 76 77 78 79 7a 7b 7c 7d 7e 7f 80 81 82 83 84 85 86 87 88 89 8a 8b 8c 8d 8e 8f


```

由于涉及到了秘钥交换，服务端在收到请求后也需要先生成一对临时公私钥，所生成的私钥为:

```
909192939495969798999a9b9c9d9e9fa0a1a2a3a4a5a6a7a8a9aaabacadaeaf


```

对应的公钥是:

```
9fd7ad6dcff4298dd3f96d5b1b2af910a0535b1488d7f8fabb349a982880b615


```

`Key Share` Extension 中返回的即为上述公钥。如果还记得上文的 ECDH 秘钥交换方法，这里就可以很容易计算出两端的共享秘钥:

```
$ ./curve25519-mult server-ephemeral-private.key \
                    client-ephemeral-public.key | hexdump

0000000 df 4a 29 1b aa 1e b7 cf a6 93 4b 29 b4 74 ba ad
0000010 26 97 e2 9f 1f 92 0d cc 77 c8 a0 a0 88 44 76 24


```

该秘钥用于生成后续握手包所需的秘钥，使用 HKDF 函数进行生成，如下所示:

```
early_secret = HKDF-Extract(salt: 00, key: 00...)
empty_hash = SHA384("")
derived_secret = HKDF-Expand-Label(key: early_secret, label: "derived", ctx: empty_hash, len: 48)
handshake_secret = HKDF-Extract(salt: derived_secret, key: shared_secret)
client_secret = HKDF-Expand-Label(key: handshake_secret, label: "c hs traffic", ctx: hello_hash, len: 48)
server_secret = HKDF-Expand-Label(key: handshake_secret, label: "s hs traffic", ctx: hello_hash, len: 48)
client_handshake_key = HKDF-Expand-Label(key: client_secret, label: "key", ctx: "", len: 32)
server_handshake_key = HKDF-Expand-Label(key: server_secret, label: "key", ctx: "", len: 32)
client_handshake_iv = HKDF-Expand-Label(key: client_secret, label: "iv", ctx: "", len: 12)
server_handshake_iv = HKDF-Expand-Label(key: server_secret, label: "iv", ctx: "", len: 12)


```

得到以下秘钥:

*   **handshake secret**: bdbbe8757494bef20de932598294ea65b5e6bf6dc5c02a960a2de2eaa9b07c929078d2caa0936231c38d1725f179d299
*   **server handshake traffic secret**: 23323da031634b241dd37d61032b62a4f450584d1f7f47983ba2f7cc0cdcc39a68f481f2b019f9403a3051908a5d1622.
*   **client handshake traffic secret**: db89d2d6df0e84fed74a2288f8fd4d0959f790ff23946cdf4c26d85e51bebd42ae184501972f8d30c4a3e4a3693d0ef0.
*   **server handshake key**: 9f13575ce3f8cfc1df64a77ceaffe89700b492ad31b4fab01c4792be1b266b7f
*   **server handshake IV**: 9563bc8b590f671f488d2da3
*   **client handshake key**: 1135b4826a9a70257e5a391ad93093dfd7c4214812f493b3e3daae1eb2b1ac69
*   **client handshake IV**: 4256d2e0e88babdd05eb2f27

客户端也可以计算出同样的秘钥值。

在计算完共享秘钥后，后续的流量将使用上述秘钥进行加密，因此对于 TLS 1.2 的情况服务端会先返回一个 ChangeCipherSpec，在 TLS 1.3 中可不必多此一举，不过在兼容模式下为了防止某些中间件抽风还是会多这么一步。

我们这里直接看加密的数据，服务端一般会先返回一个 Encrypted Extensions 类型的 Record 消息，该消息加密后存放在 Record(type=0x17)，即 Application Data 的 Body 部分，同时 (加密后数据的) 末尾还添加了 16 字节的 **Auth Tag**，这是 AEAD 算法用来校验加密消息完整性的数据。

数据使用 `AES-256-GCM` 进行加密和校验，解密代码可以参考 [aes_256_gcm_decrypt.c](https://tls13.ulfheim.net/files/aes_256_gcm_decrypt.c)，使用 server hanshake key/iv 进行解密的示例如下所示:

```
# server handshake key
$ key=9f13575ce3f8cfc1df64a77ceaffe89700b492ad31b4fab01c4792be1b266b7f
# server handshake iv
$ iv=9563bc8b590f671f488d2da3
### from this record
$ recdata=1703030017
$ authtag=9ddef56f2468b90adfa25101ab0344ae
$ recordnum=0
### may need to add -I and -L flags for include and lib dirs
$ cc -o aes_256_gcm_decrypt aes_256_gcm_decrypt.c -lssl -lcrypto
$ echo "6b e0 2f 9d a7 c2 dc" | xxd -r -p > /tmp/msg1
$ cat /tmp/msg1 \
  | ./aes_256_gcm_decrypt $iv $recordnum $key $recdata $authtag \
  | hexdump -C

00000000  08 00 00 02 00 00 16                              |.......|


```

这里解密后的拓展长度为空。一般与握手无关的额外拓展都会放在这里返回，这是为了能够尽可能地减少握手阶段的明文传输。

> 注: 如无特殊说明，后续的握手包都是使用相同方法进行加密的，并且只针对解密后的原数据进行分析。

使用 server handshake key/iv 进行加密。解密后的数据与 TLS 1.2 的证书响应相同，因此不再赘述。

前文 Hello 阶段进行 ECDHE 秘钥交换的时候其实有个问题，即双方只交换了公钥，却没有认证这个秘钥，因此如果存在网络劫持，就可能被中间人进行攻击，那加密似乎也只是加了个寂寞。

但无需担心，这点早已在计划之中。虽然之前没有进行认证，但可以后面补上。Server Certificate Verify 就是这个作用。该消息将服务端证书的私钥与之前生成的临时公钥进行绑定，准确来说是使用证书的私钥对其进行签名，并将签名算法与结果返回给客户端。由于客户端可以认证证书的有消息，就间接地证实了之前所交换的秘钥的真实性。

```
struct {
    SignatureScheme algorithm;
    opaque signature<0..2^16-1>;
} CertificateVerify;


```

同理，如果服务端需要验证客户端的真实性，那么在提供客户端证书后也同样发送一个 Certificate Verify 消息即可。

[](#server-handshake-finished-1)Server Handshake Finished
---------------------------------------------------------

至此服务端所需要发送的握手包已经发送完毕了，因此最后发送一个 Finished 数据给客户端并等待对方的握手完成。在 Finished 数据中，消息体的内容和 TLS 1.2 类似，是通过此前所有的握手数据计算得到的 `verify_data`，并使用 HMAC 进行认证，进一步确保此前的消息没有经过中间人修改。

计算方法如下:

```
finished_key = HKDF-Expand-Label(key: server_secret, label: "finished", ctx: "", len: 32)
finished_hash = SHA384(Client Hello ... Server Cert Verify)
verify_data = HMAC-SHA384(key: finished_key, msg: finished_hash)


```

server_secret 是指前文中协商得到的 **server handshake traffic secret**。

同时，服务端使用前面协商得到的 **handshake secret** 加上前面所有握手包的哈希重新计算出一个应用秘钥，用于加密实际的应用数据。计算方法如下:

```
empty_hash = SHA384("")
derived_secret = HKDF-Expand-Label(key: handshake_secret, label: "derived", ctx: empty_hash, len: 48)
master_secret = HKDF-Extract(salt: derived_secret, key: 00...)
client_secret = HKDF-Expand-Label(key: master_secret, label: "c ap traffic", ctx: handshake_hash, len: 48)
server_secret = HKDF-Expand-Label(key: master_secret, label: "s ap traffic", ctx: handshake_hash, len: 48)
client_application_key = HKDF-Expand-Label(key: client_secret, label: "key", ctx: "", len: 32)
server_application_key = HKDF-Expand-Label(key: server_secret, label: "key", ctx: "", len: 32)
client_application_iv = HKDF-Expand-Label(key: client_secret, label: "iv", ctx: "", len: 12)
server_application_iv = HKDF-Expand-Label(key: server_secret, label: "iv", ctx: "", len: 12)


```

之所以重新计算而不是使用 handshake key 是为了防止针对某些加密套件可能存在的选择密文攻击。 最终得到:

*   **server application key**: 01f78623f17e3edcc09e944027ba3218d57c8e0db93cd3ac419309274700ac27
*   **server application IV**: 196a750b0c5049c0cc51a541
*   **client application key**: de2f4c7672723a692319873e5c227606691a32d1c59d8b9f51dbb9352e9ca9cc
*   **client application IV**: bb007956f474b25de902432f

相当于 TLS 1.2 中的 client/server write key/IV。

[](#client-handshake-finished-1)Client Handshake Finished
---------------------------------------------------------

由于双方的 handshake secret 相同，那么由此派生出来的 application key/iv 必然也是相同的。

客户端在收到 Server Finished 之后会使用对应服务器证书对数据进行校验，确认无误后进行可选的 ChangeCipherSpec 将加密并签名的 verify_data 在 Finished 请求中发送给服务器。

```
# client handshake traffic secret
finished_key = HKDF-Expand-Label(key: client_secret, label: "finished", ctx: "", len: 32)
finished_hash = SHA384(Client Hello ... Server Finished)
verify_data = HMAC-SHA384(key: finished_key, msg: finished_hash)


```

可以这么理解，Server Finished 用来让客户端确认服务端没有被中间人攻击，而 Client Finished 则用来让服务端确认客户端没有被中间人攻击。双向认证之后则可以保证 TLS 握手的真实性和完整性，成功建立加密信道。

这一步通常是可选的。服务端在握手完成后会发送若干个 ticket 给客户端，可以理解为 web 中的 cookie。客户端在后续如果需要重新发起握手，可以带上这个 ticket，用于恢复当前的 TLS 会话。从上面的握手流程可见 TLS 握手需要涉及许多计算和网络请求，如果能够恢复会话，将极大地降低云服务器资源和网络延时。

ticket 消息的格式如下:

```
struct {
    uint32 ticket_lifetime;
    uint32 ticket_age_add;
    opaque ticket_nonce<0..255>;
    opaque ticket<1..2^16-1>;
    Extension extensions<0..2^16-2>;
} NewSessionTicket;


```

包含有效期、随机数等信息。其中 `ticket` 字段对于客户端是透明的，但对于服务端而言需要是有效的会话凭据，可通过该数据恢复之前的 TLS 会话。

由于 ticket 是一次性的，综合考虑时间和空间成本，一般服务端都会返回两个 ticket 给客户端。由于是服务端返回的数据，因此使用 server application key/iv 进行加密。

随后客户端发送的数据加密方式与 handshake 过程的加密类似，区别仅在于应用数据的加密使用的是 client application key/iv，服务端发送给客户端的数据使用 server application key/iv。

前文说到 Wireshark 支持导入 keylog 解密 TLS 1.2 流量，其实对于 TLS 1.3 也可以，但由于后者在 Server Hello 之后的握手包都经过了加密，因此要解密 TLS 1.3 数据流还分别需要握手阶段的秘钥。

以本节所涉及到的握手包为例，其 keylog 内容如下:

```
$ openssl s_client -keylogfile keylog.txt -connect example.com:443
$ cat keylog.txt
SERVER_HANDSHAKE_TRAFFIC_SECRET 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 23323da031634b241dd37d61032b62a4f450584d1f7f47983ba2f7cc0cdcc39a68f481f2b019f9403a3051908a5d1622
CLIENT_HANDSHAKE_TRAFFIC_SECRET 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f db89d2d6df0e84fed74a2288f8fd4d0959f790ff23946cdf4c26d85e51bebd42ae184501972f8d30c4a3e4a3693d0ef0
EXPORTER_SECRET 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 5da16dd8325dd8279e4535363384d9ad0dbe370538fc3ad74e53d533b77ac35ee072d56c90871344e6857ccb2efc9e14
SERVER_TRAFFIC_SECRET_0 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 86c967fd7747a36a0685b4ed8d0e6b4c02b4ddaf3cd294aa44e9f6b0183bf911e89a189ba5dfd71fccffb5cc164901f8
CLIENT_TRAFFIC_SECRET_0 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 9e47af27cb60d818a9ea7d233cb5ed4cc525fcd74614fb24b0ee59acb8e5aa7ff8d88b89792114208fec291a6fa96bad


```

同一个 TLS 流中的 RANDOM 值相同，且都是 `client_random`，使用 Label 来区分不同阶段的秘钥。值得一提的是 macOS 中的 LibreSSL 还不支持 `-keylogfile` 选项，因此上面的命令是在 Linux 中进行测试的。

至此，TLS 1.2 和 TLS 1.3 的常规握手流程基本上也分析完了，当然在这其中还存在一些减少 RTT 的优化，篇幅原因就留到以后分析 mmtls 的时候再说了。

还记得前文中介绍 Server Certificate 的时候，服务端返回一直到除了根证书以外的的证书链，那么从最后一个证书是如何定位到根证书的呢？一般认为是通过 issuer 字段，但 issuer 字段只是一段文字 (ASN.1)，很可能会重复，这样岂不是有歧义？。其实分析 Android 的验证流程就可以知道。

了解 Android 应用安全的朋友应该都知道系统的根证书存放在 `/system/etc/security/cacerts/` 之中，里面的证书都是 `xxxx.0` 格式，这是个传统的 PEM 格式证书，但名字比较特殊，使用 `openssl x509 -subject_hash_old` 生成。

通过阅读 AOSP 相关代码会发现，这个文件名的前半部分也正是证书的 subject 字段的 MD5，后半部分的 `.0` 则是为了防止哈希碰撞预留的计数字段，如果有重复就加一，相关代码如下:

```
// frameworks/base/core/java/android/security/net/config/DirectoryCertificateSource.java
private Set<X509Certificate> findCerts(X500Principal subj, CertSelector selector) {
    String hash = getHash(subj);
    Set<X509Certificate> certs = null;
    for (int index = 0; index >= 0; index++) {
        String fileName = hash + "." + index;
        if (!new File(mDir, fileName).exists()) {
            break;
        }
        if (isCertMarkedAsRemoved(fileName)) {
            continue;
        }
        X509Certificate cert = readCertificate(fileName);
        if (cert == null) {
            continue;
        }
        if (!subj.equals(cert.getSubjectX500Principal())) {
            continue;
        }
        if (selector.match(cert)) {
            if (certs == null) {
                certs = new ArraySet<X509Certificate>();
            }
            certs.add(cert);
        }
    }
    return certs != null ? certs : Collections.<X509Certificate>emptySet();
}


```

我们也可以通过以下 `Python` 脚本去进行手动计算:

```
from cryptography import x509
import hashlib
import struct

with open("01419da9.0", "rb") as f:
    pem = f.read()
cert = x509.load_pem_x509_certificate(pem)
d = hashlib.md5(cert.subject.public_bytes()).digest()
out = struct.unpack("<I", d[:4])[0]
print("hash: %08x" % out)
assert(out == 0x01419da9)


```

SSL Pinning 又称为 Certificate Pinning，通常翻译为证书绑定。但个人感觉这个翻译其实不太准确，因为绑定关系通常是双向的，而 Pinning 操作实质上只是客户端应用限制了本身所信任的 CA 范围，类似于图钉将某个证书固定到对应域名上。

在 Android 应用中可以通过 [NSC(Network Security Config)](https://developer.android.com/training/articles/security-config#CertificatePinning) 来设置证书绑定规则，例如:

`res/xml/network_security_config.xml`:

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">evilpan.com</domain>
        <pin-set expiration="2022-01-01">
            <pin digest="SHA-256">7HIpactkIAq2Y49orFOOQKurWxmmSFZhBCoQYcRhJ3Y=</pin>
            <!-- backup pin -->
            <pin digest="SHA-256">fwza0LRMXouZHRC8Ei+4PyuldPDcf3UKgO/04cDM1oE=</pin>
        </pin-set>
    </domain-config>
</network-security-config>


```

由于证书有效期的问题，客户端应用如果无法及时升级，可能会导致服务端证书更新后无法正常进行 HTTPS 通信，因此 Google 是[不推荐应用使用证书绑定](https://developer.android.com/training/articles/security-ssl#Pinning)的。如果一定要用的话，最好提供一个备用的可控 pin(比如自签名的证书)，用以保障通信的可用性。

当然不使用 NSC 也是可以实现证书绑定。因为本质上只是对服务端证书的单独比对，将 TLS 握手过程中获取的服务端证书或者其中的公钥与本地保存的值进行额外的校验。这个校验过程可以放在 Java 层，也可以放在 native 层 (比如抖音)，但这都是具体实现问题了。

本文主要介绍了 TLS 1.2 和 TLS 1.3 的常规握手过程，分别参考了 [RFC5246](https://datatracker.ietf.org/doc/html/rfc5246) 和 [RFC8446](https://datatracker.ietf.org/doc/html/rfc8446)。虽然有很多细节还没有涉及到，但总的来说对 TLS 的理解又加深了一点，也算没浪费这个周末。后续有时间的话会总结一些在客户端安全研究时的网络分析方法，以及一些关于 TLS 的有趣特性，Stay Tune!

*   [The Illustrated TLS 1.2 Connection](https://tls12.ulfheim.net/)
*   [The Illustrated TLS 1.3 Connection](https://tls13.ulfheim.net/)
*   [TLS 协议分析 与 现代加密通信协议设计](https://blog.helong.info/blog/2015/09/07/tls-protocol-analysis-and-crypto-protocol-design/)
*   [WeMobileDev: 基于 TLS1.3 的微信安全通信协议 mmtls 介绍](https://github.com/liuqun/article/blob/master/%E5%9F%BA%E4%BA%8ETLS1.3%E7%9A%84%E5%BE%AE%E4%BF%A1%E5%AE%89%E5%85%A8%E9%80%9A%E4%BF%A1%E5%8D%8F%E8%AE%AEmmtls%E4%BB%8B%E7%BB%8D.md)

> **版权声明**: 自由转载 - 非商用 - 非衍生 - 保持署名 ([CC 4.0 BY-SA](https://creativecommons.org/licenses/by-nc/4.0/))  
> **原文地址**: [https://evilpan.com/2022/05/15/tls-basics/](https://evilpan.com/2022/05/15/tls-basics/)  
> **微信订阅**: **『[有价值炮灰](http://t.evilpan.com/qrcode.jpg)』**  
> – _TO BE CONTINUED_.