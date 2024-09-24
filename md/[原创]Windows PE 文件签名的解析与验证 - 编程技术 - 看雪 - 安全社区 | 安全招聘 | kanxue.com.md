> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283584.htm)

> [原创]Windows PE 文件签名的解析与验证

为了方便在 mac 和 Linux 下解析与验证 PE 签名，我开发了一个跨平台工具 [https://github.com/0xlane/pe-sign](https://github.com/0xlane/pe-sign)。安装和使用方法可以参考项目中的 [README_zh.md](https://github.com/0xlane/pe-sign/blob/main/README_zh.md)。本文分享的是开发该工具之前做的一些签名研究内容。

导出签名数据
------

具体的 PE 格式可以参考 [MSDN](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)。

签名证书的位置在 Certificate Table 里（也称为 Security Directory）：

![](https://bbs.kanxue.com/upload/attach/202409/860174_C9FMWX84VTY7KEV.webp)

可以从 Optional Header 的 Data Directories 里找到 Security Directory 的文件偏移：

![](https://bbs.kanxue.com/upload/attach/202409/860174_6U2DP9TD8KH3QSY.webp)

如图表示，ProcessHacker.exe 文件的签名数据在 0x1A0400 位置，长度 0x3A20。

导航到这个偏移位置即可看到这里就是签名数据：

![](https://bbs.kanxue.com/upload/attach/202409/860174_KYPDE2TFTE4HXSM.webp)

参考 [MSDN](https://learn.microsoft.com/en-us/windows/win32/api/wintrust/ns-wintrust-win_certificate)， 签名数据的结构如下：

```
typedef struct _WIN_CERTIFICATE {
  DWORD dwLength;
  WORD  wRevision;
  WORD  wCertificateType;
  BYTE  bCertificate[ANYSIZE_ARRAY];
} WIN_CERTIFICATE, *LPWIN_CERTIFICATE;
```

所以，bCertificate 是实际的证书内容，wCertificateType 表示签名证书类型，根据这个字段可知支持三种证书类型：PKCS #1、PKCS #7、X509，我看到过的文件都是使用 PKCS #7 签名。

找到 Security Directory 偏移之后，跳过前面的 8 字节就是实际的 PKCS #7 证书内容，DER 格式，代码示意：

```
fn extract_pkcs7_from_pe(file: &PathBuf) -> Result, Box> {
    // ...
    let image = VecPE::from_disk_file(file.to_str().unwrap())?;
    let security_directory =
        image.get_data_directory(exe::ImageDirectoryEntry::Security)?;
    let signature_data =
        exe::Buffer::offset_to_ptr(&image, security_directory.virtual_address.into())?; // security_data_directory rva is equivalent to file offset
 
    Ok(unsafe {
        let vec = std::slice::from_raw_parts(signature_data, security_directory.size as usize).to_vec();    // cloned
        vec.into_iter().skip(8).collect()   // _WIN_CERTIFICATE->bCertificate
    })
} 
```

使用项目中的 pe-sign 工具可以直接导出：

```
pe-sign extract > pkcs7.cer 
```

使用 `--pem` 参数可以将 DER 格式转换为 PEM 格式：

```
pe-sign extract --pem > pkcs7.pem 
```

使用 openssl 解析证书
---------------

解析导出的证书，如果是 PEM 格式，`-inform DER` 改为 `-inform PEM`：

```
openssl pkcs7 -inform DER -in .\pkcs7.cer -print_certs -text -noout
```

这里有个小疑问，从属性里看 ProcessHacker.exe 文件有两个签名，一个是 sha1, 另一个是 sha256：

![](https://bbs.kanxue.com/upload/attach/202409/860174_3HHBPQUPFAU5QSH.webp)

openssl 打印的证书信息只有 sha1 证书的，使用 `-print` 参数打印 pkcs7 结构也可以看到是有 sha256 证书内容的但是没正确解析。

看了下微步沙箱也只解析到了 1 个签名：[https://s.threatbook.com/report/file/bd2c2cf0631d881ed382817afcce2b093f4e412ffb170a719e2762f250abfea4](https://s.threatbook.com/report/file/bd2c2cf0631d881ed382817afcce2b093f4e412ffb170a719e2762f250abfea4)

解析内嵌证书
------

经过一番观察，发现 sha256 这个证书是在 sha1 证书属性里内嵌的，并不是平级：

![](https://bbs.kanxue.com/upload/attach/202409/860174_ZARGRT5CMRUFDX4.webp)

然后就搜了一个这个属性名 `1.3.6.1.4.1.311.2.4.1` 有什么特殊的地方，从 [MSDN](https://learn.microsoft.com/en-us/previous-versions/hh968145(v=vs.85)) 可知，这个值表示的是 `szOID_NESTED_SIGNATURE` 内容（实际就是一个 pkcs#7 格式证书），ChatGPT 是这么解释这个属性的：

> szOID_NESTED_SIGNATURE 是一个表示嵌套签名的对象标识符（OID），其对应的 OID 是 1.3.6.1.4.1.311.2.4.1。在 PKCS7 或 CMS（Cryptographic Message Syntax）中，嵌套签名允许在签名数据中嵌套另一个签名数据块。这种机制用于实现多层次的签名或加密操作。

使用 openssl 的 asn1parse 命令可以找出嵌套签名的偏移位置：

```
PS C:\dev\windows_pe_signature_research> openssl asn1parse -i -inform DER -in \pkcs7.cer | Select-String -Context 1,3 1.3.6.1.4.1.311.2.4.1
 
   7638:d=6  hl=4 l=7223 cons:       SEQUENCE
>  7642:d=7  hl=2 l=  10 prim:        OBJECT            :1.3.6.1.4.1.311.2.4.1
   7654:d=7  hl=4 l=7207 cons:        SET
   7658:d=8  hl=4 l=7203 cons:         SEQUENCE
   7662:d=9  hl=2 l=   9 prim:          OBJECT            :pkcs7-signedData
```

SET 后面开始就是嵌入数据，7658 就是嵌套数据开始的文件偏移，hl 表示头大小，l 表示数据大小，所以总的嵌套数据大小为 4+7203=7207。使用 powershell 提取这个嵌套签名：

```
$offset = 7658
$size = 7207
$fileStream = [System.IO.File]::OpenRead(".\pkcs7.cer")
$buffer = New-Object byte[] $size
$fileStream.Seek($offset, [System.IO.SeekOrigin]::Begin)
$fileStream.Read($buffer, 0, $size)
$fileStream.Close()
[System.IO.File]::WriteAllBytes(".\pkcs7_embed.cer", $buffer)
```

再次使用 openssl 就可以解析出这个嵌套签名证书：

```
PS C:\dev\windows_pe_signature_research> openssl pkcs7 -inform der -in .\pkcs7_embed.cer -print_certs -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            04:0c:b4:1e:4f:b3:70:c4:5c:43:44:76:51:62:58:2f
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=DigiCert Inc, OU=www.digicert.com, CN=DigiCert SHA2 High Assurance Code Signing CA
        Validity
            Not Before: Oct 30 00:00:00 2013 GMT
            Not After : Jan  4 12:00:00 2017 GMT
        Subject: C=AU, ST=New South Wales, L=Sydney, O=Wen Jia Liu, CN=Wen Jia Liu
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:97:a8:e8:af:59:2a:05:f3:d5:0e:36:66:eb:89:
                    95:52:1a:3b:dd:41:12:63:b2:81:b9:f4:d0:cb:3d:
                    df:7d:f5:9b:f1:55:35:c0:9b:0a:ae:39:f6:ed:d8:
                    da:58:dd:ab:0e:ca:ce:b2:de:de:f0:fd:9d:6d:77:
                    1e:9d:05:ae:51:e6:02:27:8d:a4:c2:43:2f:2b:07:
                    cf:04:0b:00:a0:46:5d:61:13:f6:a3:b6:ab:bd:04:
                    b3:e4:b6:a2:9e:bd:94:d6:95:cf:28:bb:d9:5f:dc:
                    fb:06:2c:52:00:3d:63:6c:64:f8:68:ca:02:5a:1f:
                    25:b8:1c:d5:af:6e:bb:11:61:c0:f5:72:97:32:c1:
                    66:af:41:b8:7b:59:b0:da:e5:5b:9b:25:db:56:b4:
                    44:fc:52:5f:44:40:3b:5f:b0:02:37:53:d1:9f:96:
                    a5:a0:a5:47:87:19:c8:3d:a6:5b:91:05:01:b1:d4:
                    00:96:14:31:80:04:8a:e0:a6:a3:a5:32:31:92:37:
                    1a:93:85:da:b1:e9:79:ec:1a:bb:a6:1a:34:c7:70:
                    80:2d:8a:d6:89:38:d3:8c:54:ae:6e:86:3d:3a:c5:
                    49:d6:72:7b:b7:94:b6:6b:ee:f0:d0:70:11:c2:f0:
                    a2:5d:d8:87:5c:47:a4:7e:8e:36:29:d5:64:cf:49:
                    79:85
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                67:9D:0F:20:09:0C:CC:8A:3A:E5:82:46:72:62:FC:F1:CC:90:E5:40
            X509v3 Subject Key Identifier:
                13:86:9A:3B:EF:68:31:53:BC:6B:18:9B:34:C6:FF:0A:8B:4D:68:28
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                Code Signing
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:http://crl3.digicert.com/sha2-ha-cs-g1.crl
                Full Name:
                  URI:http://crl4.digicert.com/sha2-ha-cs-g1.crl
            X509v3 Certificate Policies:
                Policy: 2.16.840.1.114412.3.1
                  CPS: https://www.digicert.com/CPS
                Policy: 2.23.140.1.4.1
            Authority Information Access:
                OCSP - URI:http://ocsp.digicert.com
                CA Issuers - URI:http://cacerts.digicert.com/DigiCertSHA2HighAssuranceCodeSigningCA.crt
            X509v3 Basic Constraints: critical
                CA:FALSE
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        91:87:ac:23:c2:84:07:cb:c7:6a:c1:f8:1a:6c:78:3b:21:9f:
        c2:48:1b:08:e8:f5:e3:41:f7:eb:e2:d9:41:b1:48:e6:1b:ff:
        55:ab:79:c1:a2:d8:16:7a:2d:d2:94:f5:32:c8:00:5a:5f:dd:
        f2:f8:b2:19:86:47:fb:e7:aa:a7:16:e6:ff:0a:c3:37:f9:64:
        c0:5b:51:64:ef:8a:23:c4:7a:d0:8f:d7:37:b8:70:dd:35:6f:
        19:06:4e:a5:cd:ea:0a:ef:a4:f2:2a:1c:b3:49:f8:a6:89:ac:
        a5:67:f3:8b:b3:de:01:23:20:4f:9f:f5:56:0c:59:ec:e4:01:
        32:3f:2c:e5:84:be:e0:ea:5b:b7:39:31:26:ff:32:7f:eb:15:
        cc:82:d0:b0:16:f8:fc:6f:1d:c8:b2:1b:9c:85:68:27:7d:45:
        b0:e0:7a:7c:dd:26:f4:9a:d4:7d:0f:a6:ac:04:c1:48:65:1c:
        ef:49:33:0b:d2:c7:28:95:a7:26:51:09:cf:1a:f0:d2:ca:74:
        93:92:8c:e3:44:6e:24:5b:57:9f:b2:cc:df:01:e0:3d:b4:7f:
        41:cc:97:d6:ea:3d:1f:42:3d:fd:2b:75:82:0c:43:50:e9:d8:
        ac:d1:d9:e6:e4:08:97:05:75:29:55:be:2d:89:7d:ab:16:5e:
        be:32:e0:b5
 
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            0b:7e:10:90:3c:38:49:0f:fa:2f:67:9a:87:a1:a7:b9
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=DigiCert Inc, OU=www.digicert.com, CN=DigiCert High Assurance EV Root CA
        Validity
            Not Before: Oct 22 12:00:00 2013 GMT
            Not After : Oct 22 12:00:00 2028 GMT
        Subject: C=US, O=DigiCert Inc, OU=www.digicert.com, CN=DigiCert SHA2 High Assurance Code Signing CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:b4:4a:5e:7d:07:0f:41:de:c4:f5:76:16:36:bd:
                    71:ff:cf:3f:4f:73:4b:9c:d1:0d:fe:4a:cb:57:58:
                    5e:85:16:dd:02:15:54:99:f0:8f:3c:2f:4d:02:78:
                    10:68:c8:d8:35:4b:3f:c1:f7:67:ce:98:1c:ae:33:
                    b9:2d:1d:a4:0a:54:93:c4:85:a2:df:35:b1:f5:f1:
                    3c:a7:b3:34:fb:5d:48:c9:46:c9:62:44:bc:48:99:
                    eb:28:49:53:c3:3d:8f:c0:0e:de:35:98:e9:62:51:
                    df:3d:6b:40:61:ee:04:41:da:cf:a7:5c:56:96:d1:
                    f9:4c:b7:44:84:87:98:69:e5:82:b9:13:e6:55:bf:
                    c8:92:70:92:0a:31:6f:7f:8b:32:ab:cf:6b:5a:9f:
                    62:c4:3e:ee:be:ed:59:a4:53:7f:0b:f1:52:88:8a:
                    7b:0a:67:24:cb:90:cd:ec:d2:4d:34:4c:b0:e1:b5:
                    9f:9c:c6:f6:6f:2c:cd:e6:ca:53:74:01:9f:67:35:
                    de:38:49:2d:ce:ed:39:44:82:19:79:4e:1a:b2:b5:
                    fb:bb:78:f0:49:66:a7:cf:fa:5c:96:75:92:8b:1a:
                    72:d9:ff:50:92:53:cc:3e:c2:43:32:09:1a:86:13:
                    69:3c:fb:81:32:33:32:64:75:73:28:26:1d:08:30:
                    3b:07
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 Extended Key Usage:
                Code Signing
            Authority Information Access:
                OCSP - URI:http://ocsp.digicert.com
                CA Issuers - URI:http://cacerts.digicert.com/DigiCertHighAssuranceEVRootCA.crt
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:http://crl4.digicert.com/DigiCertHighAssuranceEVRootCA.crl
                Full Name:
                  URI:http://crl3.digicert.com/DigiCertHighAssuranceEVRootCA.crl
            X509v3 Certificate Policies:
                Policy: 2.16.840.1.114412.0.2.4
                  CPS: https://www.digicert.com/CPS
                Policy: 2.16.840.1.114412.3
            X509v3 Subject Key Identifier:
                67:9D:0F:20:09:0C:CC:8A:3A:E5:82:46:72:62:FC:F1:CC:90:E5:40
            X509v3 Authority Key Identifier:
                B1:3E:C3:69:03:F8:BF:47:01:D4:98:26:1A:08:02:EF:63:64:2B:C3
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        6a:0e:ff:7e:13:7c:06:a5:4b:c0:2e:8c:f9:53:64:09:e2:ba:
        58:91:30:50:ec:cc:9f:e1:d3:a8:2f:48:46:36:18:29:d0:78:
        28:5f:98:56:40:0f:1e:ba:bd:b1:3b:87:5c:dc:5b:d8:20:0d:
        ed:1a:16:4d:d5:11:24:21:4b:f1:27:69:90:13:eb:11:a1:01:
        da:fd:b5:4e:79:59:75:bd:38:2a:6a:c3:f6:8e:41:2b:8a:a2:
        8b:d7:2c:51:51:d9:9c:a0:c8:e3:4e:ba:6c:a8:47:d2:4e:d1:
        68:1f:8c:02:57:3b:b3:29:6a:8e:6a:20:2a:b9:f2:00:62:64:
        ba:c8:e9:00:f9:cc:a4:d4:ba:9a:35:d8:af:2c:65:6c:16:7c:
        58:21:de:4a:30:d0:fa:eb:24:5d:06:c9:9d:16:b7:ad:4a:45:
        d3:25:e2:0c:f0:40:aa:5c:4d:ac:7e:cd:06:82:b9:76:46:69:
        08:d8:32:b6:82:fe:e3:a9:58:34:43:1b:8e:67:67:97:3f:68:
        31:16:36:38:95:3e:87:f7:c7:c3:af:9d:7a:77:19:d9:de:93:
        b5:fd:6e:2b:fc:94:f9:3d:b7:4c:12:35:2c:30:be:e8:8d:9e:
        05:70:9a:48:13:f4:8c:d6:e7:1e:ac:38:e7:a8:f3:ad:0c:b7:
        7a:ec:67:ed
```

### 导出内嵌证书

代码解析的话，需要按 ASN.1 结构手动解析，因为这里找的内容算是个固定的，只需要定位 `1.3.6.1.4.1.311.2.4.1` 所在位置就可以。所以就需要研究一下 OID 在证书内是以怎么方式存储的，根据之前 openssl 的 asn1parse 命令的结果可以知道偏移位置 7642 + 2 处为 `1.3.6.1.4.1.311.2.4.1`，长度 10：

```
>  7642:d=7  hl=2 l=  10 prim:        OBJECT            :1.3.6.1.4.1.311.2.4.1
```

用下面的代码读取出来为 `2b 06 01 04 01 82 37 02 04 01`：

```
$offset = 7644
$size = 10
$fileStream = [System.IO.File]::OpenRead(".\pkcs7.cer")
$buffer = New-Object byte[] $size
$fileStream.Seek($offset, [System.IO.SeekOrigin]::Begin)
$fileStream.Read($buffer, 0, $size)
$fileStream.Close()
Write-Host $(($buffer | ForEach-Object { "{0:x2}" -f $_ }) -join " ")
```

可以发现有些数字是可以直接能和 `1.3.6.1.4.1.311.2.4.1` 对应上的：

```
2b . 06 . 01 . 04 . 01 . 82 37 . 02 . 04 01
```

不难看出，这里的 `2b` 就是表示的 `1.3`，`82 37` 表示的 `311`。但是 `13` 的十六进制表示为 `D`、`311` 的十六进制表示为 `137`，所以这里并不是直接的 10-16 转换关系。查了一些 OID 的资料，在 [这篇文章](https://letsencrypt.org/zh-cn/docs/a-warm-welcome-to-asn1-and-der/#object-identifier-%E7%9A%84%E7%BC%96%E7%A0%81) 中找到了一些关于 OID 编码方式的描述：

> OID 的实质就是一串整数， 而且至少由两个整数组成。 第一个数必须是 0、1、2 三者之一， 如果是 0 或 1，则第二个数必须小于 40。 因此，前两个数 X 和 Y 可以直接用 40×X+Y 来表示，不会产生歧义。
> 
> 以 2.999.3 的编码为例，首先要将前两个数合并成 1079（即 40×2+999），得到 1079.3。
> 
> 完成合并后再用 Base 128 编码，左侧为高位字节。 也就是说，每个字节的最高位设为 1，但最后一个字节最高位设为 0，表示一个整数到此结束。各字节的其余七位从高到低依次相连表示数值。 例如数字 3 就用一个字节 0x03 表示， 而 129 则需要两个字节 0x81 0x01。 每个数字都如此转换成字节后，拼接在一起就形成了 OID 的编码。
> 
> 无论是在 BER 还是 DER 中，OID 都必须用最短的方式编码。 所以其中每个数字编码时开头都不能出现 0x80 字节。

所以只有开头的 `1.3` 编码方式和后面不一样，后面都是 Base 128 编码。`1.3` 需要先按公式变成 `40x1+3 = 43`，再转换为十六进制就是 `2b`。反着推就是 `2b` 的十进制表示为 `43`，需要先确定第一位，因为只能是 0、1、2。

0 和 1 时第二个数不能超过 40，所以可以知道第一位：

*   是 0 时，最终得到的十进制数字在 [0, 40) 之间
*   是 1 时，最终得到的十进制数字在 [40, 80) 之间
*   是 2 时，最终得到的十进制数字在 [80, 255) 之间（我感觉开头两位长度是固定的 1 字节，所以超不过 255，不可能出现上面文章说的 1079 的情况）

所以 `43` 在 [40, 80) 之间，第一位应该是 `1`，第二位就是 `43 - 40x1 = 3`。

后面每个数字都由 Base 128 编码方式存储，即数字在 128 (0x80) 以内（不包含 128）不需要转换直接存储，所以在上面能看到有些个位数的数字能直接对应上。128 之后就需要转成二进制从右向左每 7 位拆分一次，例如 `311` 的二进制表示为 `100110111`，每 7 位拆分 1 次变成：

```
10  0110111
```

前面补 0 变 8 位：

```
00000010  00110111
```

除最后 1 个字节（8 位）不需要动，前面每个字节最高位变成 1：

```
10000010  00110111
```

最后变成 十六进制看就是 `82 37`。

现在对 `2b 06 01 04 01 82 37 02 04 01` 整体反向解析一下，开头两位前面说过了，`2b` 表示 `1.3`，后面的数字需要 1 个接 1 个字节换成二进制形式地看：

*   先看 `06` 二进制表示为 `00000110`，开头是 0，所以直接转换为 `6`，`01 04 01` 同理表示的 `1.4.1`
*   到 `82` 二进制形式为 `10000010`，开头是 1 表示还有后续，`37` 二进制形式为 `00110111`，开头是 0 表示当前数字结束了，`82 37` 是一个整体，整体换成二进制 `10000010 00110111`，去掉每个字节的最高位连起来就是 `0000010 0110111`，即 `311`
*   后面的 `02 04 01` 都不超过 128，所以和前面一样是直接存储的不需要转换，表示的 `2.4.1`

合起来最终得到 `1.3.6.1.4.1.311.2.4.1`。

搞明白这个之后，可以直接通过 `2b 06 01 04 01 82 37 02 04 01` 定位到后 1 位再向后偏移 4 字节跳过 SET 块就是 `SEQUENCE`，此处便是内嵌证书的开头位置，一般直接读取到结尾就可以，严谨一点的话需要解析一下头部位置：

```
30 82 1C 23
```

第 1 个字节是标签，30 表示 `SEQUENCE` 不用管，第二个字节最高位被设置为 1 的时候代表长编码，为 0 表示短编码，短编码时该字节就表示内容的长度，长编码时该字节表示存储长度的字节大小，`0x82` 最高位为 1，所以是长编码形式，去掉最高位的 1 就是 `0x2`，表示后面有 2 个字节用来表示 `SEQUENCE` 内容的长度，即 `0x1c23`，加上头的长度，这里是 `4`，内嵌证书长度就是 `0x1c27`。

代码实现：

```
fn extract_pkcs7_embedded<'a>(cert_bin: &'a [u8]) -> Option<&'a [u8]> {
    match cert_bin.windows(EMBEDDED_SIGNATURE_OID.len()).position(|w| w == EMBEDDED_SIGNATURE_OID) {
        Some(pos) => {
            let sequence_pos = pos + EMBEDDED_SIGNATURE_OID.len() + 4;
            let mut header_len = 1;
 
            let size = if cert_bin[sequence_pos + 1] >= 0b10000000 {
                // 长编码
                let len = (cert_bin[sequence_pos + 1] & 0b01111111) as usize;
                header_len += 1 + len;
                let size_bin = &mut cert_bin[sequence_pos + 1 + 1 .. sequence_pos + 1 + 1 + len].to_vec();
                size_bin.reverse();     // big endian to little endian
                size_bin.resize(8, 0x0);    // align to usize
                usize::from_le_bytes(unsafe { *(size_bin.as_ptr() as *const [u8; 8]) })
            } else {
                // 短编码
                header_len += 1;
                cert_bin[sequence_pos + 1] as usize
            };
 
            Some(&cert_bin[sequence_pos.. sequence_pos + header_len + size])
        },
        None => None
    }
}
```

现在可以直接使用项目中的 pe-sign 工具的 `--embed` 参数直接导出：

```
pe-sign extract --pem --embed > pkcs7.pem 
```

也可以直接传递到 openssl 解析：

```
pe-sign extract --pem --embed | openssl pkcs7 -inform pem -noout -text -print_certs 
```

验证证书
----

PKCS #7 证书的验证可以直接调用 [`PKCS7_verify`](https://docs.openssl.org/1.0.2/man3/PKCS7_verify/)，该函数原型：

```
int PKCS7_verify(PKCS7 *p7, STACK_OF(X509) *certs, X509_STORE *store, BIO *indata, BIO *out, int flags);
```

根据参数可以知道验证证书需要准备的东西：

*   `p7`: 证书内容
*   `certs`: 可选参数，签名者证书，在 PE 提取的证书里一般有签名者，可以不传
*   `store`: 可选参数，用于链路验证的受信任证书存储，需要验证证书链所有证书的有效性，所以需要传，可以导入 curl 提供的 [cacert.pem](https://curl.se/docs/caextract.html) 文件作为受信任的 CA 证书
*   `indata`: 被签名的原始数据内容，可空
*   `out`: 验证通过后，输出签名的数据，可空
*   `flags`: 一些控制验证过程的标志，没有特殊需求就是 0

里面比较特殊的是 `indata` 参数，根据资料显示如果证书为 detached 证书，即签名独立证书，此时证书内不包含被签名的原始数据内容，需要调用 `PKCS7_verify` 时传入 `indata` 参数，否则可以为空。

这样的话，PE 中提取的证书内必然包含被签名的原始数据，所以看似不需要传入 `indata`，使用这个函数对应的 openssl 命令快速验证一下：

```
PS C:\dev\windows_pe_signature_research>.\pe-sign.exe extract .\ProcessHacker.exe --pem | openssl smime -verify -inform PEM -CAfile .\cacert.pem -purpose any --no_check_time
Verification failure
64390000:error:80000002:system library:file_open:No such file or directory:providers\implementations\storemgmt\file_store.c:263:calling stat(C:\Program Files\Common Files\SSL/certs)
64390000:error:1608010C:STORE routines:inner_loader_fetch:unsupported:crypto\store\store_meth.c:360:No store loader found. For standard store loaders you need at least one of the default or base providers available. Did you forget to load them? Info: Global default library context, Scheme (C : 0), Properties ()
64390000:error:10800065:PKCS7 routines:PKCS7_signatureVerify:digest failure:crypto\pkcs7\pk7_doit.c:1084:
64390000:error:10800069:PKCS7 routines:PKCS7_verify:signature failure:crypto\pkcs7\pk7_smime.c:342: 
```

验证失败了，导致失败的报错信息是 `digest failure` 和 `signature failure`，看起来是签名和原始数据内容并不匹配。大概查了一下，PKCS #7 SignedData 结构用 ASN.1 描述为：

```
SignedData ::= SEQUENCE {
  version Version,
  digestAlgorithms DigestAlgorithmIdentifiers,
  contentInfo ContentInfo,
  certificates [0] IMPLICIT ExtendedCertificatesAndCertificates OPTIONAL,
  Crls [1] IMPLICIT CertificateRevocationLists OPTIONAL,
  signerInfos SignerInfos
}

DigestAlgorithmIdentifiers ::=
  SET OF DigestAlgorithmIdentifier

ContentInfo ::= SEQUENCE {
  contentType ContentType,
  content [0] EXPLICIT ANY DEFINED BY contentType OPTIONAL
}

ContentType ::= OBJECT IDENTIFIER

SignerInfos ::= SET OF SignerInfo

SignerInfo ::= SEQUENCE {
  version Version,
  issuerAndSerialNumber IssuerAndSerialNumber,
  digestAlgorithm DigestAlgorithmIdentifier,
  authenticatedAttributes [0] IMPLICIT Attributes OPTIONAL,
  digestEncryptionAlgorithm DigestEncryptionAlgorithmIdentifier,
  encryptedDigest EncryptedDigest,
  unauthenticatedAttributes [1] IMPLICIT Attributes OPTIONAL
}
IssuerAndSerialNumber ::= SEQUENCE {
  issuer Name,
  serialNumber CertificateSerialNumber
}
EncryptedDigest ::= OCTET STRING
```

`ContentInfo` 中保存的原始数据，`encryptedDigest` 中是消息摘要，验签时需要使用签名者的 publickey 解开 `encryptedDigest` 得到一个 hash，然后如果存在 `authenticatedAttributes`，里面能再提取出一个消息摘要，这个消息摘要的 hash 如果和 `encryptedDigest` 中的一致说明可信，验证 `authenticatedAttributes` 可信后，从 `authenticatedAttributes` 中又可以得到一个 hash，这个 hash 是 `ContentInfo` 中原始内容的 hash，最终通过该 hash 验证原始数据内容是否可信。假如没有 `authenticatedAttributes`，`encryptedDigest` 的消息摘要 hash 就是 `ContentInfor` 原始内容的 hash。

调用 `PKCS7_verify` 时，会自动进行上面的验证，但是当解析 `ContentInfo->content` 的时候，根据 `ContentInfo->contentType` 类型解析，只有类型为 `data`、`ASN1_OCTET_STRING` 才能被正确解析到 content（参考源码位置 [pk7_doit.c#L48](https://github.com/openssl/openssl/blob/f98e49b326fe1fda5efadc10e7905b09a394591c/crypto/pkcs7/pk7_doit.c#L48)）。

根据 [Authenticode_PE.docx](http://download.microsoft.com/download/9/c/5/9c5b2167-8017-4bae-9fde-d599bac8184a/Authenticode_PE.docx) 中的描述，Authenticode 相当于 PE 文件的 hash，保存在 `ContentInfo->content` 中，`ContentInfo->contentType` 必须是 `SPC_INDIRECT_DATA_OBJID (1.3.6.1.4.1.311.2.1.4)`，所以 `PKCS7_verify` 并不能解析出 content 内容，需要自行解析后通过 `indata` 参数传入。

以下是 `SpcIndirectDataContent` 的 ASN.1 定义：

```
SpcIndirectDataContent ::= SEQUENCE {
    data                    SpcAttributeTypeAndOptionalValue,
    messageDigest           DigestInfo
} --#public—

SpcAttributeTypeAndOptionalValue ::= SEQUENCE {
    type                    ObjectID,
    value                   [0] EXPLICIT ANY OPTIONAL
}

DigestInfo ::= SEQUENCE {
    digestAlgorithm     AlgorithmIdentifier,
    digest              OCTETSTRING
}

AlgorithmIdentifier    ::=    SEQUENCE {
    algorithm           ObjectID,
    parameters          [0] EXPLICIT ANY OPTIONAL
}
```

对应的 asn1parse 解析结果：

```
39:d=3  hl=2 l=  76 cons:    SEQUENCE         
41:d=4  hl=2 l=  10 prim:     OBJECT            :1.3.6.1.4.1.311.2.1.4                          // contentType = SPC_INDIRECT_DATA_OBJID
53:d=4  hl=2 l=  62 cons:     cont [ 0 ]       
55:d=5  hl=2 l=  60 cons:      SEQUENCE                                                         // SpcIndirectDataContent
57:d=6  hl=2 l=  23 cons:       SEQUENCE         
59:d=7  hl=2 l=  10 prim:        OBJECT            :1.3.6.1.4.1.311.2.1.15
71:d=7  hl=2 l=   9 cons:        SEQUENCE         
73:d=8  hl=2 l=   1 prim:         BIT STRING       
76:d=8  hl=2 l=   4 cons:         cont [ 0 ]       
78:d=9  hl=2 l=   2 cons:          cont [ 2 ]       
80:d=10 hl=2 l=   0 prim:           cont [ 0 ]       
82:d=6  hl=2 l=  33 cons:       SEQUENCE         
84:d=7  hl=2 l=   9 cons:        SEQUENCE         
86:d=8  hl=2 l=   5 prim:         OBJECT            :sha1
93:d=8  hl=2 l=   0 prim:         NULL             
95:d=7  hl=2 l=  20 prim:        OCTET STRING      [HEX DUMP]:9253A6F72EE0E3970D5457E0F061FDB40B484F18
```

下面是 pe-sign 工具中关于提取 `indata` 的代码：

```
let indata = unsafe {
    let x = pkcs7.as_ptr();
    let signed_data = (*x).d.sign;
    let other = (*(*signed_data).contents).d.other;
    let authenticode_seq =
        Asn1StringRef::from_ptr((*other).value.sequence).as_slice();
 
    // seq value
    if authenticode_seq[1] >= 0b10000000 {
        // 长编码
        let len = (authenticode_seq[1] & 0b01111111) as usize;
        authenticode_seq[1 + 1 + len..].to_vec()
    } else {
        // 短编码
        authenticode_seq[1 + 1..].to_vec()
    }
};
```

解析证书中的一些关键信息
------------

由于 openssl 解析出的证书信息只是一些证书通用的信息，有些关键字段还需要单独解析一下：

*   signingTime: 签名时间
*   authenticode: PE 部分内容的哈希，验证证书有效之后，还需要验证 authenticode

### 签名时间解析

签名时间大多数时候存储在副署签名里，少数直接在认证属性里就可以找到，在副署签名里的时候有两种情况：

第一种情况，当 unauth_attrs 直接存在副署（countersignature）属性时，该内容为 `PKCS7_SIGNER_INFO` 结构，表示副署签名者信息，该信息作为副署（countersignature）签名展示：

![](https://bbs.kanxue.com/upload/attach/202409/860174_HZHK4PD8AF4W9RM.webp)

第二种情况，不存在副署（countersignature）属性，寻找 unauth_attrs 中的 `1.3.6.1.4.1.311.3.3.1` 属性作为副署签名，该内容为 Timestamping signature：

![](https://bbs.kanxue.com/upload/attach/202409/860174_GHBBFKFBF8ZBHSY.webp)

该属性内容被作为副署（countersignature）签名展示：

![](https://bbs.kanxue.com/upload/attach/202409/860174_J3EA93ABY5SG87T.webp)

签名时间一般保存在副署签名中名为 signingTime 的 auth_attrs 里：

![](https://bbs.kanxue.com/upload/attach/202409/860174_PWNSWGU7GQME3NK.webp)

如果 signingTime 属性不存在（[案例](https://s.threatbook.com/report/file/3177a0554250e2092443b00057f3919efd6d544e243444b70eb30f1f7dd9f1d1)），签名时间保存在 `contentInfo->content` 中。

把案例文件的 `1.3.6.1.4.1.311.3.3.1` 属性内容导出：

```
pe-sign.exe extract 3177a0554250e2092443b00057f3919efd6d544e243444b70eb30f1f7dd9f1d1 -o ts331_sign.cer
$offset = 4243
$size = 6016
$fileStream = [System.IO.File]::OpenRead(".\ts331_sign.cer")
$buffer = New-Object byte[] $size
$fileStream.Seek($offset, [System.IO.SeekOrigin]::Begin)
$fileStream.Read($buffer, 0, $size)
$fileStream.Close()
[System.IO.File]::WriteAllBytes(".\ts331_attr.cer", $buffer)
openssl cms -cmsout -inform DER -in .\ts331_attr.cer -text -noout -print
```

从结果中可以看到，`contentInfo->content` 内容类型为 `id-smime-ct-TSTInfo`，里面也是一段 ASN.1 序列化数据：

```
CMS_ContentInfo:
  contentType: pkcs7-signedData (1.2.840.113549.1.7.2)
  d.signedData:
    version: 3
    digestAlgorithms:
        algorithm: sha256 (2.16.840.1.101.3.4.2.1)
        parameter: NULL
    encapContentInfo:
      eContentType: id-smime-ct-TSTInfo (1.2.840.113549.1.9.16.1.4)
      eContent:
        0000 - 30 82 01 39 02 01 01 06-0a 2b 06 01 04 01 84   0..9.....+.....
        000f - 59 0a 03 01 30 31 30 0d-06 09 60 86 48 01 65   Y...010...`.H.e
        001e - 03 04 02 01 05 00 04 20-fb e4 fd ce 67 6d c5   ....... ....gm.
        002d - 75 54 88 67 d0 48 b6 04-b4 e4 33 2d c2 1c 89   uT.g.H....3-...
        003c - 0a 70 80 cc 1e 28 a6 a2-f7 25 02 06 66 95 56   .p...(...%..f.V
        004b - 6e bc 39 18 13 32 30 32-34 30 38 30 31 30 30   n.9..2024080100
        005a - 34 36 34 34 2e 36 31 37-5a 30 04 80 02 01 f4   4644.617Z0.....
        0069 - a0 81 d1 a4 81 ce 30 81-cb 31 0b 30 09 06 03   ......0..1.0...
        0078 - 55 04 06 13 02 55 53 31-13 30 11 06 03 55 04   U....US1.0...U.
        0087 - 08 13 0a 57 61 73 68 69-6e 67 74 6f 6e 31 10   ...Washington1.
        0096 - 30 0e 06 03 55 04 07 13-07 52 65 64 6d 6f 6e   0...U....Redmon
        00a5 - 64 31 1e 30 1c 06 03 55-04 0a 13 15 4d 69 63   d1.0...U....Mic
        00b4 - 72 6f 73 6f 66 74 20 43-6f 72 70 6f 72 61 74   rosoft Corporat
        00c3 - 69 6f 6e 31 25 30 23 06-03 55 04 0b 13 1c 4d   ion1%0#..U....M
        00d2 - 69 63 72 6f 73 6f 66 74-20 41 6d 65 72 69 63   icrosoft Americ
        00e1 - 61 20 4f 70 65 72 61 74-69 6f 6e 73 31 27 30   a Operations1'0
        00f0 - 25 06 03 55 04 0b 13 1e-6e 53 68 69 65 6c 64   %..U....nShield
        00ff - 20 54 53 53 20 45 53 4e-3a 41 34 30 30 2d 30    TSS ESN:A400-0
        010e - 35 45 30 2d 44 39 34 37-31 25 30 23 06 03 55   5E0-D9471%0#..U
        011d - 04 03 13 1c 4d 69 63 72-6f 73 6f 66 74 20 54   ....Microsoft T
        012c - 69 6d 65 2d 53 74 61 6d-70 20 53 65 72 76 69   ime-Stamp Servi
        013b - 63 65                                          ce
```

从右侧可以看出，`20240801004644.617Z` 就是签名时间。程序中需要解析 TSTInfo 结构：

```
TSTInfo ::= SEQUENCE {
    version        INTEGER { v1(1) },
    policy         TSAPolicyID,
    messageImprint MessageImprint,
        -- MUST have the same value as the similar field in
        -- TimeStampReq serialNumber INTEGER,
        -- Time Stamps users MUST be ready to accommodate integers
        -- up to 160 bits.
    serialNumber   SerialNumber,
    genTime        GeneralizedTime,
    accuracy       Accuracy     OPTIONAL,
    ordering       BOOLEAN DEFAULT FALSE,
    nonce          INTEGER      OPTIONAL,
        -- MUST be present if the similar field was present
        -- in TimeStampReq. In that case it MUST have the same value.
    tsa [0]        GeneralName  OPTIONAL,
    extensions [1] IMPLICIT Extensions OPTIONAL
}
```

提取出 `contentInfo->content`，并通过 openssl 解析 asn1 结构：

```
PS C:\dev\windows_pe_signature_research> openssl cms -verify -inform DER -in .\ts331_attr.cer -noverify -out content.bin -binary
PS C:\dev\windows_pe_signature_research> openssl asn1parse -i -inform DER -in .\content.bin
    0:d=0  hl=4 l= 313 cons: SEQUENCE
    4:d=1  hl=2 l=   1 prim:  INTEGER           :01
    7:d=1  hl=2 l=  10 prim:  OBJECT            :1.3.6.1.4.1.601.10.3.1
   19:d=1  hl=2 l=  49 cons:  SEQUENCE
   21:d=2  hl=2 l=  13 cons:   SEQUENCE
   23:d=3  hl=2 l=   9 prim:    OBJECT            :sha256
   34:d=3  hl=2 l=   0 prim:    NULL
   36:d=2  hl=2 l=  32 prim:   OCTET STRING      [HEX DUMP]:FBE4FDCE676DC575548867D048B604B4E4332DC21C890A7080CC1E28A6A2F725
   70:d=1  hl=2 l=   6 prim:  INTEGER           :6695566EBC39
   78:d=1  hl=2 l=  19 prim:  GENERALIZEDTIME   :20240801004644.617Z
   99:d=1  hl=2 l=   4 cons:  SEQUENCE
  101:d=2  hl=2 l=   2 prim:   cont [ 0 ]
  105:d=1  hl=3 l= 209 cons:  cont [ 0 ]
  108:d=2  hl=3 l= 206 cons:   cont [ 4 ]
  111:d=3  hl=3 l= 203 cons:    SEQUENCE
  114:d=4  hl=2 l=  11 cons:     SET
  116:d=5  hl=2 l=   9 cons:      SEQUENCE
  118:d=6  hl=2 l=   3 prim:       OBJECT            :countryName
  123:d=6  hl=2 l=   2 prim:       PRINTABLESTRING   :US
  127:d=4  hl=2 l=  19 cons:     SET
  129:d=5  hl=2 l=  17 cons:      SEQUENCE
  131:d=6  hl=2 l=   3 prim:       OBJECT            :stateOrProvinceName
  136:d=6  hl=2 l=  10 prim:       PRINTABLESTRING   :Washington
  148:d=4  hl=2 l=  16 cons:     SET
  150:d=5  hl=2 l=  14 cons:      SEQUENCE
  152:d=6  hl=2 l=   3 prim:       OBJECT            :localityName
  157:d=6  hl=2 l=   7 prim:       PRINTABLESTRING   :Redmond
  166:d=4  hl=2 l=  30 cons:     SET
  168:d=5  hl=2 l=  28 cons:      SEQUENCE
  170:d=6  hl=2 l=   3 prim:       OBJECT            :organizationName
  175:d=6  hl=2 l=  21 prim:       PRINTABLESTRING   :Microsoft Corporation
  198:d=4  hl=2 l=  37 cons:     SET
  200:d=5  hl=2 l=  35 cons:      SEQUENCE
  202:d=6  hl=2 l=   3 prim:       OBJECT            :organizationalUnitName
  207:d=6  hl=2 l=  28 prim:       PRINTABLESTRING   :Microsoft America Operations
  237:d=4  hl=2 l=  39 cons:     SET
  239:d=5  hl=2 l=  37 cons:      SEQUENCE
  241:d=6  hl=2 l=   3 prim:       OBJECT            :organizationalUnitName
  246:d=6  hl=2 l=  30 prim:       PRINTABLESTRING   :nShield TSS ESN:A400-05E0-D947
  278:d=4  hl=2 l=  37 cons:     SET
  280:d=5  hl=2 l=  35 cons:      SEQUENCE
  282:d=6  hl=2 l=   3 prim:       OBJECT            :commonName
  287:d=6  hl=2 l=  28 prim:       PRINTABLESTRING   :Microsoft Time-Stamp Service
```

没有找到能快速解析 TSTInfo 结构的方法，所以代码里直接通过搜索第一个 `GENERALIZEDTIME` 作为签名时间。在 ASN.1 序列化数据中 0x18 表示 `GENERALIZEDTIME`，后跟一个字节表示长度，因为 `GENERALIZEDTIME` 时间格式大多数时候为 `YYYYMMDDHHMMSSZ`、`YYYYMMDDHHMMSS.sssZ`，其他情况遇到再改代码吧，`s` 可以减少不一定是 3 个，所以长度在 15 - 19 这个范围，找 `18 [15, 19]` 序列即可。

### authenticode 解析

Authenticode 信息在 `indata` 里，需要根据 `SpcIndirectDataContent` 结构解析：

```
SpcIndirectDataContent ::= SEQUENCE {
    data                    SpcAttributeTypeAndOptionalValue,
    messageDigest           DigestInfo
} --#public—

SpcAttributeTypeAndOptionalValue ::= SEQUENCE {
    type                    ObjectID,
    value                   [0] EXPLICIT ANY OPTIONAL
}

DigestInfo ::= SEQUENCE {
    digestAlgorithm     AlgorithmIdentifier,
    digest              OCTETSTRING
}

AlgorithmIdentifier    ::=    SEQUENCE {
    algorithm           ObjectID,
    parameters          [0] EXPLICIT ANY OPTIONAL
}
```

`SpcIndirectDataContent->messageDigest->digest` 就是 authenticode，indata 数据开头的 sequence 表示的是 `SpcIndirectDataContent->data`，跳过这个 sequence 就是 `messageDigest`。

提取代码：

```
fn extract_authtiencode(cert_bin: &[u8]) -> Option<(String, String)> {
    unsafe {
        let asn1_type =
            d2i_ASN1_TYPE(null_mut(), &mut cert_bin.as_ptr(), cert_bin.len() as _).as_ref()?;
        if asn1_type.type_ == V_ASN1_SEQUENCE {
            let data_seq = Asn1StringRef::from_ptr(asn1_type.value.sequence);
            let data_len = data_seq.len();
            if data_len > 0 && cert_bin.len() > data_len {
                // 跳过 SpcIndirectDataContent->data 得到 SpcIndirectDataContent->messageDigest
                let message_digest_bin = &cert_bin[data_len..];
                // 跳过 seq header
                let message_digest_bin = &message_digest_bin[2..];
                // 解析 digestAlgorithm
                let digest_algo_obj_bin_len = message_digest_bin[3] as usize;
                let digest_algo_obj_bin = &message_digest_bin[2..2 + 2 + digest_algo_obj_bin_len];
                let asn1_type = d2i_ASN1_TYPE(
                    null_mut(),
                    &mut digest_algo_obj_bin.as_ptr(),
                    digest_algo_obj_bin.len() as _,
                )
                .as_ref()?;
                if asn1_type.type_ == V_ASN1_OBJECT {
                    let digest_algo_obj = Asn1ObjectRef::from_ptr(asn1_type.value.object);
                    let digest_algo_str = digest_algo_obj.to_string();
                    // 跳过 SpcIndirectDataContent->messageDigest->digestAlgorithm 得到 SpcIndirectDataContent->messageDigest -> digest
                    let digest_algo_seq_len = message_digest_bin[1] as usize;
                    let digest_octet_str_bin = &message_digest_bin[2 + digest_algo_seq_len..];
                    let asn1_type = d2i_ASN1_TYPE(
                        null_mut(),
                        &mut digest_octet_str_bin.as_ptr(),
                        digest_octet_str_bin.len() as _,
                    )
                    .as_ref()?;
                    if asn1_type.type_ == V_ASN1_OCTET_STRING {
                        let digest_octet_str =
                            Asn1OctetStringRef::from_ptr(asn1_type.value.octet_string);
                        let authenticode = to_hex_str(digest_octet_str.as_slice());
                        return Some((digest_algo_str, authenticode));
                    }
                }
            }
        }
    }
 
    None
}
```

通过 pe-sign 工具验证证书并打印证书信息：

```
PS C:\dev\windows_pe_signature_research> pe-sign help verify
Check the digital signature of a PE file for validity
 
Usage: pe-sign.exe verify [OPTIONS] Arguments:
   Options:
      --no-check-time   Ignore certificate validity time
      --ca-file Trusted certificates file [default: cacert.pem]
  -h, --help            Print help 
```

authenticode 计算
---------------

[Authenticode_PE.docx](http://download.microsoft.com/download/9/c/5/9c5b2167-8017-4bae-9fde-d599bac8184a/Authenticode_PE.docx) 中有详细的 authenticode 计算步骤：

1.  加载 PE 文件到内存
2.  初始化 hash 算法上下文
3.  hash checknum 字段之前的内容
4.  跳过 checknum 字段，4 字节
5.  hash checknum 字段之后到 Certificate Table Directory 之前
6.  获取 Certificate Table Directory 大小
7.  跳过 Certificate Table Directory，hash Certificate Table Directory 之后到 header 结尾
8.  创建一个计数器 SUM_OF_BYTES_HASHED, 设置计数器为 optionalHeader->SizeOfHeaders 字段值
9.  构建一个由 section header 组成的数组
10.  根据 PointerToRawData 字段对 section header 数组递增排序
11.  根据排序后的 section header，hash section 内容
12.  添加 section header 的 SizeOfRawData 值到 SUM_OF_BYTES_HASHED
13.  重复 11、12 步遍历所有 section
14.  创建一个 FILE_SIZE 变量表示文件大小，如果 FILE_SIZE 比 SUM_OF_BYTES_HASHED 大，表示还有额外的数据需要 hash. 额外数据在 SUM_OF_BYTES_HASHED 偏移处开始，长度为：  
    (File Size) – ((Size of AttributeCertificateTable) + SUM_OF_BYTES_HASHED)
15.  完成 hash 算法上下文

可以通过 pe-sign 工具计算 authenticode，支持 sha1、sha256、md5 三种算法：

```
PS C:\dev\windows_pe_signature_research> pe-sign help calc
Calculate the authticode digest of a PE file
 
Usage: pe-sign.exe calc [OPTIONS] Arguments:
   Options:
  -a, --algorithm Hash algorithm [default: sha256] [possible values: sha1, sha256, md5]
  -h, --help                   Print help 
```

使用 pe-sign 工具验证文件签名时，status 如果是 `InvalidSignature` 表示证书中的 authenticode 与实际不符，文件被篡改：

```
PS C:\dev\windows_pe_signature_research> pe-sign verify .\3177a0554250e2092443b00057f3919efd6d544e243444b70eb30f1f7dd9f1d1"
Certificate:
    CN: Microsoft Corporation
    Status: InvalidSignature
    Version: V2
    SN: 33000003af30400e4ca34d05410000000003af
    Fingerprint: c2048fb509f1c37a8c3e9ec6648118458aa01780
    Algorithm: sha256WithRSAEncryption
    ValidityPeriod: 2023-11-17 03:09:00 +08:00 - 2024-11-15 03:09:00 +08:00
    SigningTime: 2024-08-01 08:46:44.617 +08:00
    Authenticode: 60d5dabe345c33681964cc0ae9f4bfebc754869f4d0c918d12f9c5ecce26296c
==============================================================
PS C:\dev\windows_pe_signature_research> pe-sign calc .\3177a0554250e2092443b00057f3919efd6d544e243444b70eb30f1f7dd9f1d1"
ab19518250c085de397e582c33f4bb911a193ac500aa7952d318faae41a477c0
```

其他 tips
-------

### RSA 公钥解密

大多数工具只提供了公钥验签，不提供公钥解密功能。可以使用类似 [https://www.lddgo.net/en/encrypt/rsa](https://www.lddgo.net/en/encrypt/rsa) 的在线解密工具，对签名数据解密。

### 导入系统中内置的可信任根证书

打开 `cerlm.msc` 证书管理窗口，导航到 “受信任的根证书颁发机构 -> 证书”，全选后导出为 p7b。

然后通过 openssl 可以转换 p7b 证书为 pem：

```
openssl pkcs7 -inform DER -in win_ca.cer -print_certs -outform PEM > win_ca.pem
```

不过这种方式，不知道什么原因导出的证书有时候并不全，可能有缺失。

所以改用 python 的 wincertstore 库枚举系统根证书然后导出为 PEM：

```
import wincertstore
 
wincerts = []
 
with wincertstore.CertSystemStore("ROOT") as store:
     for cert in store.itercerts(usage=wincertstore.CODE_SIGNING):
         if cert.cert_type == 'CERTIFICATE':
            tmp = cert.get_pem()
            if tmp not in wincerts:
              wincerts.append(tmp)
 
with open('c:\\cacerts.pem', 'r') as fp:
  fp.write("\n".join(wincerts))
```

[[课程]Linux pwn 探索篇！](https://www.kanxue.com/book-section_list-172.htm)

最后于 20 小时前 被 techliu 编辑 ，原因：

[#基础知识](forum-41-1-130.htm) [#开源分享](forum-41-1-134.htm) [#开发技巧](forum-41-1-135.htm)