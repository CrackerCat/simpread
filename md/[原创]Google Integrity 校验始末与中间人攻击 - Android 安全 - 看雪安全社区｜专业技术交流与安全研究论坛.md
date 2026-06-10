> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291541.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

Google Integrity 一直以绝对可信闻名，基于此大量的银行、金融类 APP 都高度依赖 Integrity 服务来验证客户端的请求是否可信。但它真的完全可靠吗？怀揣着这个疑问，我深度分析了 Integrity Token 的生成过程与 Key Attestation 的原理。得出的结论是不可靠。

在官方的文档中，想要获取一个 Token 有两种请求方式。分别为传统与标准请求。

传统请求的示例代码为：

```
import com.google.android.gms.tasks.Task; ...


String nonce = ...


IntegrityManager integrityManager =
    IntegrityManagerFactory.create(getApplicationContext());


Task<IntegrityTokenResponse> integrityTokenResponse =
    integrityManager
        .requestIntegrityToken(
            IntegrityTokenRequest.builder().setNonce(nonce).build());


```

标准请求的示例代码为：

```
import com.google.android.gms.tasks.Task;

// Create an instance of a manager.
StandardIntegrityManager standardIntegrityManager =
    IntegrityManagerFactory.createStandard(applicationContext);

StandardIntegrityTokenProvider integrityTokenProvider;
long cloudProjectNumber = ...;

// Prepare integrity token. Can be called once in a while to keep internal
// state fresh.
standardIntegrityManager.prepareIntegrityToken(
    PrepareIntegrityTokenRequest.builder()
        .setCloudProjectNumber(cloudProjectNumber)
        .build())
    .addOnSuccessListener(tokenProvider -> {
        integrityTokenProvider = tokenProvider;
    })
    .addOnFailureListener(exception -> handleError(exception));


```

这两种请求方式的区别仅仅是标准请求使用了缓存，高频调用的响应速度更快。从业务安全的角度来说是完全一致的。因此，我们以传统请求 API 为例，来

分析完整的生成过程。

#### 客户端分析

首先我们通过 Maven 拿到 Integrity 的 jar 包。然后使用 Jadx 打开定位到 requestIntegrityToken 函数：  
![](https://bbs.kanxue.com/upload/attach/202606/1012164_B69EBUDPH8GVH9R.webp)

发起请求时携带了一个 IntegrityTokenRequest 参数，里面主要包含了获取 Token 的必要参数：nonce、cloudProjectNumber。

进入到 this.a.c 函数：

![](https://bbs.kanxue.com/upload/attach/202606/1012164_XPMUKK4DZ42ZGF5.webp)

*   将 nonce 解码为 byte[]
*   初始化 TaskCompletionSource 回调类
*   初始化 an 执行类（继承自 Runnable ）
*   this.a.u 启动 an 线程

组装 Binder 跨进程调用参数：  
![](https://bbs.kanxue.com/upload/attach/202606/1012164_RGE2S4SU6J8ZJT3.webp)

发起调用：

![](https://bbs.kanxue.com/upload/attach/202606/1012164_57UKUNM9PZFE6C5.webp)

最终由 IIntegrityService 做为服务端处理来自客户端的请求，至此我们正式进入 Google Play 源码分析。

#### 服务端分析

![](https://bbs.kanxue.com/upload/attach/202606/1012164_MNC7B9NWM5SB3C9.webp)  
![](https://bbs.kanxue.com/upload/attach/202606/1012164_BXV7RZBKDNNVJMY.webp)

*   经由 dispatchTransaction 转发到 c 函数
*   由 c 函数具体处理客户端的请求

由于篇幅的关系，我这里不展开 c 以及一系列子函数的调用。我只大概讲一下它的流程：

*   接收客户端参数
*   验证客户端的身份（通过 Binder.getCallingUid() 验证 Uid 是否与包名相等）
*   收集设备信息（ro.build.product、ro.product.brand、ro.vendor.build.fingerprint）
*   收集登录账号信息（用于验证是否具有指定 APP 的安装权限）
*   收集包名信息（通过 PMS 获取版本号、签名 Signature[] apkContentsSigners = signingInfo.hasMultipleSigners() ? signingInfo.getApkContentsSigners() : signingInfo.getSigningCertificateHistory(); 、安装来源标识等……）
*   将包名等信息转发给 GMS DroidGuard 签名
*   发起 Key Attestation 挑战拿到 Tee 证书链（验证设备是否已解锁）
*   将所有信息组装为 Protobuf 数据包发给后端验证
*   后端返回 Integrity Token 回调客户端

```
Request Body：
{
  "type": "IntegrityRequestParameters",
  "deviceIntegritySignals": {
    "type": "DeviceIntegritySignals",
    "droidguardTokenByteString": {
      "type": "ByteString",
      "size": 39032,
      "sha256": "bd5a30d9d722df7e36397b918861aa9084ef9eb54ae31255926e7552e84e4104",
      "asciiPreview": "..'..i........O.].....[.....s.......D.Q..M.w+.t..o...@..{HD..~-..U'....V.Y..z......._.}......DL.",
      "hexPreview": "0a06270ae369a80012c1b0020a064c06f22f2f1e1e68ae3126dba9ac7ae367a7488b1cc40d81ff230e19a6c55d75537e7d3",
      "base64Preview": "CgYnCuNpqAASwbACCgZPrF2VyZrSEFsAAOOyvHOlh8C+TYZbQWkeIyFfTKs5JIvjXQqUIFrFi1qWSY0nXJZIXfHo0tpmUUwpdGWPHySiA"
    },
    "flowName": "pia_attest_e1",
    "keyAttestationMetadata": {
      "type": "bmyq",
      "raw": "# bmyq@5eb8d24",
      "field_b": "2",
      "field_d": "null",
      "field_e": "# bmyn@7c344",
      "field_f": "null",
      "certificateChain": [
        {
          "type": "ByteString",
          "size": 678,
          "sha256": "984470b2b5ceafcdcd705c1a6285bc93ffe420fea2d25d373dcdc3949ac85fda",
          "asciiPreview": "0...0..H........0...*.H.=...091.0...U....TEE1)0'..U... 22317fdd8510cac935893090d9b6e2a00...70010",
          "hexPreview": "308202a230820248a003020102020101300a060828648ce3d020106082a8648ce3d0301070342000413a0b0fdc54e9e1861e41bdc30",
          "base64Preview": "MIICojCCAkigAwIBAgIBATAKBggqhkjOPQQDAjA5MQwwC30Oo4IBWTCCAVUwDgYDVR0PAQH/BAQDAgeAMIIBQQYKKwYBBAHWeQIBEQSCATEw"
        },
        {
          "type": "ByteString",
          "size": 503,
          "sha256": "5e6af0046b0b9faf744df21648abe4e049c1c7e5c0de06b830e6b784187d7e8d",
          "asciiPreview": "0...0..y........&..Y...O.B....M0...*.H.=...091.0...U....TEE1)0'..U... d492ff615155c8a4a60926ded6",
          "hexPreview": "308201f330820179a00302010202101c26a48a5902c1a64f97429fadfbe538393330393064396236653261",
          "base64Preview": "MIIB8zCCAXmgAwIBAgIQHCakilkCwaZPl0KfrfvhTTAKBggqhkjOCWgwDTmGjYzBh"
        },
        {
          "type": "ByteString",
          "size": 920,
          "sha256": "2812ef63dd35e8d8c7cbc6453dbd7961ec5f1e3645d4eae0584dbd383f0ae7ec",
          "asciiPreview": "0...0..|...................G..w.0...*.H........0.1.0...U....f92009e853b6b0450...240202225757Z..3",
          "hexPreview": "308203943082017ca003020102021100941cbcc1e8a41de02e17e847c3b877db300d06092a052b810400220362000408",
          "base64Preview": "MIIDlDCCAXygAwIBAgIRAJQcvMHopB3gLhfoR8O4d9swDQYJKoZIhvcNAQELBQAwGzEZMBcGABGuj"
        },
        {
          "type": "ByteString",
          "size": 1312,
          "sha256": "cedb1cb6dc896ae5ec797348bce9286753c2b38ee71ce0fbe34a9a1248800dfc",
          "asciiPreview": "0...0.............r.....0...*.H........0.1.0...U....f92009e853b6b0450...220320180748Z..420315180",
          "hexPreview": "3082051c30820304a0030201041663abef982f32c77f7531030c9752",
          "base64Preview": "MIIFHDCCAwSgAwIBAgIJAPHBcqaZ6vUdM+T+W8a9nsNL/ggj"
        }
      ],
      "certificateChainCount": 4
    },
    "buildFingerprintMetadata": {
      "type": "bmyh",
      "fingerprint": "Lenovo/TB320FC_PRC/TB320FC:15/AQ3A.240812.002/ZUXOS_1.1.350_250418_PRC:user/release-keys",
      "raw": "# bmyh@8f6a54a8"
    }
  },
  "packageName": "gr.nikolasspyr.integritycheck",
  "versionCode": 22,
  "nonce": "4X4a4MUeHGybbu9GQF9vYBbqYTg==",
  "certificateSha256Digests": [
    "F5UrXPhnBbreh3Q_WjMe_kyYK_tNoNL9XXC_wjXPeeM"
  ],
  "timestampAtRequest": "2026-04-18T13:45:51.190Z",
  "cloudProjectNumber": null,
  "playCoreVersion": {
    "type": "bmyu",
    "major": 1,
    "minor": 4,
    "patch": 0,
    "versionString": "1.4.0"
  },
  "playProtectDetails": {
    "type": "bmyv",
    "raw": "# bmyv@7bcd9",
    "fields": {
      "b": "1",
      "c": "2"
    }
  },
  "appAccessRiskDetailsResponse": {
    "type": "abfk",
    "raw": "AppAccessRiskDetailsResponse{installedAppsSignalData={2=[com.motorola.mobiledesktop, com.zui.pp, com.zui.wifip2p, com.lenovo.penservice, com.lenovo.ue.device, com.lenovo.levoice.trigger, com.zui.safecenter, com.qualcomm.qti.services.systemhelper, com.zui.cores, com.dolby.dolbyvisionservice, vendor.qti.qesdk.sysservice, com.qualcomm.qti.uceShimService, com.qualcomm.qti.dynamicddsservice, com.qti.dpmserviceapp, com.motorola.android.providers.settings], 6=[bin.mt.plus]}, accessibilityAbuseSignalData={}, displayListenerMetadata=DisplayListenerMetadata{isActiveDisplayPresent=false, displayListenerInitialisationTimeDelta=Optional.empty, lastDisplayAddedTimeDelta=Optional[PT-0.001S], displayListenerUsed=2}, signalGenerationLatency=PT0.002342813S, signalGenerationBreakdownTelemetry=SignalGenerationBreakdownTelemetry{requestParametersLatency=RequestParametersLatencyBreakdown{installedPackages=PT0.040158698S, runningAppProcesses=PT0.002319791S, runningServices=PT0.001184271S, activeDisplays=PT0.001730833S, enabledAccessibilityServices=PT0.004136928S, mediaProjectionDebugDump=PT0.001912032S, runningApps=PT0.000196459S, installedPackagesIsRecognized=PT0.000449375S, appOpsToOpEntry=PT0.00093677S, manifestPermissionToPackages=PT0.067272969S, activeDisplayInfo=PT0.000058125S, accessibilityServiceData=PT0.000021302S, runningMediaProjectionTypeServicePackages=PT0.000812031S}, installedAppsSignalLatency=PT0.000189688S, screenCaptureSignalLatency=PT0.00091099S, accessibilityAbuseSignalLatency=PT0.000031927S, screenOverlaySignalLatency=PT0.000257761S, displayListenerMetadataLatency=PT0.001320938S}}"
  },
 "installSourceMetadata": {
        "type": "bmyk",
        "formatted": "1:1,2:1,3:0,4:1,5:[],6:0",
        "raw": {
          "c": "31",
          "d": "1",
          "e": "1",
          "f": "0",
          "g": "1",
          "i": "0"
        }
      },
      "locationTrustMetadata": null,
      "deviceIdentifier": {
        "type": "ByteString",
        "size": 64,
        "sha256": "e47aa4c751f9079c06367ff3814e9bb441c754242e2154929de5067a680a3b4e",
        "ascii": "894ce4844f1b477fa99a4f515de4acaa80121998b19efe3e7faffae6497be163",
        "hex": "383934636534383434663162343737669376265313633",
        "base64": "ODk0Y2U0ODQ0ZjFiNDc3ZmE5="
      }
    }
}

Response Body:
{
  "token": "eyJhbGciOiJBMjU2S1ciLCJlbmMiOiJBMjU2R0NNIn0.ullChctbyhVID6nmtbgtcMqN3fxb2tsNIeLD9baKL6YnkFjbfjGcWSyADMZLeaK1XbmSYMO3OegfoX0CwEQkSDSa3bfwfM9Bo_NTD3b8i_hXCkpdAuPpahcLy_VLh5tvcqwSx7KAFEyg5IxkPPGeIIVehPwi22xBPcq3V5Pe7NOF705Av4nXv1-GgP9t5IPnf3moFbo4lqMJBANi8.35KbgY4kp8sL9jJyRNjE5w",
  "statusCode": "0",
  "extra": "null",
}


```

在收集的所有信息中，最为人津津乐道的便是 Tee 签名的证书链。这是由绝对安全的硬件空间生成的设备描述信息，按理说是绝对可信的。因为，一旦你更改了返回的证书链，你没办法重签。不重签后端就不认这个证书链，直接判定为伪造。你想要重签，就必须拿到私钥，可是私钥放到 Tee 中，不可导出不可读取。因此，形成了完美闭环。

这套完美的防御体系，看似无懈可击。但是当 Tee 私钥泄漏了之后呢？这条防御链条中最重要的一环，将土崩瓦解。

Google Play 发起 Key Attestation 的代码为：

![](https://bbs.kanxue.com/upload/attach/202606/1012164_WBTFJKPHVGKMBZA.webp)

*   首先指定别名、密钥生成算法、挑战值（防重放）等信息构建一个 KeyGenParameterSpec
    
*   通过 KeyPairGenerator 指定 AndroidKeyStore 为具体的处理服务
    
*   generateKeyPair 生成派生密钥对
    
*   getCertificateChain 拿到最终的认证证书链
    
*   发送给服务端验证
    

这些调用的背后实际上发生了什么呢？generateKeyPair 具体的流程如下：  
![](https://bbs.kanxue.com/upload/attach/202606/1012164_ZWWSTGFQ78B3C84.webp)  
这一步调用完以后 ，Tee 返回一个 X.509 证书链。这个证书链就是 Key Attestation 验证的核心。一般情况下至少包含三个证书：

*   Root Certificate（厂商根证书（预存在 Tee），必须交由 Google 备案。否则不能通过验证。）
*   Attestation Certificate（中间证书（预存在 Tee），当前设备的证书。必须经过 Root Private Key 签名。）
*   Leaf Certificate（叶子证书（Tee 动态生成的证书），必须经过 Attestation Private Key 签名。附带大量的设备描述与辅助验证信息）

根据官方文档描述，Leaf Certificat 附带的信息至少包含如下两项：

1.  RootOfTrust（用于校验设备是否已解锁）
2.  AttestationApplicationId（用于校验发起密钥认证的 APP 签名）

拿到证书链之后，便组包发给了 Google 后端进行校验。那么在后端是如何校验的呢？依赖的便是在上文中指定的签名算法：EC（secp256r1），实际上使用的算法是：ECDSA（椭圆曲线数字签名算法）secp256r1 为指定的椭圆曲线参数。算法具体的签名与验证过程：

![](https://bbs.kanxue.com/upload/attach/202606/1012164_NGR5R9W7ZMGM38F.webp)

根据图中的验证过程可知：私钥只签名不参与验证，公钥只验证不参与签名。签名值与公钥分别附加在证书的 Certificate.signatureValue Certificate.TBSCertificate.subjectPublicKeyInfo 中。因为每张证书都由上张证书签名，所以 Google 拿到三张证书依次进行校验：

*   r, s = decode_ecdsa_signature(Leaf.signatureValue); ecdsa_verify(Leaf.TBSCertificate, r, s, Attestation Public Key)
*   r, s = decode_ecdsa_signature(AttestationsignatureValue); ecdsa_verify(Attestation.TBSCertificate, r, s, Root Public Key)

```
def ecdsa_verify(message, r, s, Q):
    
    if r <= 0 or r >= n:
        return False

    if s <= 0 or s >= n:
        return False

    
    e = sha256(message)

    
    w = inverse_mod(s, n)

    
    u1 = (e * w) % n
    u2 = (r * w) % n

    
    P = point_add(
            scalar_mult(u1, G),
            scalar_mult(u2, Q)
        )

    if P is INF:
        return False

    
    x = P.x % n

    return x == r


```

当校验到 Root Certificate 时，由于没有上级证书。这时拿 Root Certificate 去数据库匹配，如果这张 Root Certificate 是由厂商提交过备案的，则证明整条链条可信。接下来再继续校验 AttestationApplicationId、RootOfTrust 等信息。其中 RootOfTrust 是获得 MEETS_DEVICE_INTEGRITY 也就是二绿的关键。这里面携带了设备是否解锁的标识：deviceLocked。并且整个 RootOfTrust 状态是由 Tee 维护的，因此不具备伪造性。

但是当我们拥有一台有 Root 权限并且没有解锁 Bootloader 设备的时候呢？那么这台设备就具备就具备以下两个发起中间人攻击的条件：

*   能通过 Google Integrity 验证（因为 BootLoader 未解锁）
*   能将 Key Attestation 的证书链导出（因为拥有 Root 权限）

设备 A：Y700 - 拥有 Root 权限 - 三绿

设备 B：Pixel6 - 拥有 Root 权限 - 一绿

Google Play 版本均保持一致（过 AttestationApplicationId 签名校验）

<table><thead><tr><th>设备</th><th>是否具有 Root 权限</th><th>是否通过 Google Integrity 验证</th></tr></thead><tbody><tr><td>Y700</td><td>是</td><td>是</td></tr><tr><td>Pixel 6</td><td>是</td><td>否</td></tr></tbody></table>

我们使用如下 Frida 脚本启动 Pixel 的 Google Play：

```
let cache_certificate_chain = []

function sendToY700(challenge) {
    
    
    return certificate_chain
}

function hook() {
    Java.perform(function () {
        const KeyPairGeneratorSpec = Java.use('android.security.keystore.KeyGenParameterSpec$Builder');
        KeyPairGeneratorSpec.setAttestationChallenge.overload('[B').implementation = function (challenge) {
            cache_certificate_chain = sendToY700(challenge)
            return this.setAttestationChallenge(challenge);
        };


        const SecLevel = Java.use('android.security.keystore2.IKeystoreSecurityLevel$Stub$Proxy');
        SecLevel.getCertificateChain.implementation = function () {
            return cache_certificate_chain
        };

    });
}

setImmediate(hook)


```

如下脚本启动 Y700 的 Google Play：

```
function receiveFormPixel6(challenge) {
    let KeyGenParameterSpec keyGenParameterSpecBuild = new KeyGenParameterSpec.Builder("integrity.api.key.alias", 4).setAlgorithmParameterSpec(new ECGenParameterSpec("secp256r1")).setDigests("SHA-512").setAttestationChallenge(challenge).setDevicePropertiesAttestationIncluded(false).build();
    let KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("EC", "AndroidKeyStore");
    keyPairGenerator.initialize(keyGenParameterSpecBuild);
    if (keyPairGenerator.generateKeyPair() == null) {
        throw new IllegalStateException("Failed to create the key pair.");
    }
    let String keystoreAlias = keyGenParameterSpecBuild.getKeystoreAlias();
    return keyStore.getCertificateChain(keystoreAlias);
}


```

然后惊奇的发现：

![](https://bbs.kanxue.com/upload/attach/202606/1012164_EJPZ3ZWNH4SQQQU.webp)

[[培训]《冰与火的战歌：Windows 内核攻防实战》！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)