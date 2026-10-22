> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293244.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 设备：理论任何高通手机  
> 关键词：RKP、KeyMint、CDI_Leaf、keybox、QSEE、MINK IPC

玩 keybox 的都知道两条路：**采集**（把设备的 RKP seed / CDI_Leaf 从 TEE 里抠出来，然后离线无限量签）和**代签**（让设备自己把证书申请走一遍）。前者一劳永逸，后者需要真机配合。本文记录的是第三条路——**不取种子、不走系统正常 RKP 流程，直接把 KeyMint TA 当成签名 oracle**，让 TA 用真 CDI_Leaf 给我们的软件公钥签出 CSR，再提交 Google 拿到完整 attestation 证书链，最终产出可用的 `keybox.xml`。

整件事的起点其实是一次失败的攻坚：我们最初想按老思路把 CDI_Leaf 种子采出来，把 9008、PBL、firehose、fastboot、Ed25519 nonce 全部试了一遍，撞了一路墙。墙撞完之后换了个方向看，发现根本不需要破任何内存防线。

设备（KeyMint 2.0+）出厂不烧 attestation keybox，而是运行时用 RKP（Remote Key Provisioning）协议向 Google 申请：

```
TEE 生成密钥对 → 公钥放进 CSR(AuthenticatedRequest)
→ CSR 用 CDI_Leaf 签名（DICE 证书链背书到熔丝）
→ Google 验链 → 签发 X.509 证书链
```

CDI_Leaf 就是那 32 字节的 Ed25519 种子。在这台设备上，它由 `KDF(SHK, "RKP PRIVATE KEY SEED")` 派生，运行时固定在 KeyMint TA 的全局区 `base+0x67761`，其公钥在 `base+0x67741`。拿到它就能离线无限量出 keybox，这就是 "seed 采集"。

CSR 的 `keysToSign` 里每个公钥，理论上必须是 **TEE 内部生成的**。保证这点的机制是：每个 key 要附带一个用设备唯一 RKP HMAC key 计算的 MAC——NS 世界没有这把 key，所以塞不进自己生成的 key。这是整条信任链里唯一防止 "软件私钥混进 TEE 签名队列" 的关卡。

后文会看到这层关卡在这版 TA 上形同虚设。

先说结论：**这台机器的 PBL 已经修掉了 CVE-2026-25262 依赖的加载器缺陷**，9008 路线在本机不成立。以下是实测验签地图（Sahara v3，COM10，授权 devprg `xbl_s_devprg_ns.melf`）：

<table><thead><tr><th>攻击面</th><th>实测结果</th></tr></thead><tbody><tr><td>elf0(RV32 stub) 数据段修改</td><td>status=0 通过，但该 stub 从不执行（entry hang 补丁无影响）</td></tr><tr><td>elf0 header/phdr 修改</td><td>NAK 0x30，拉数据前先验</td></tr><tr><td>elf1(TME fw)/elf2/elf3/elf4 数据段修改</td><td>全部 NAK 0x22（拉完后验）</td></tr><tr><td>elf1-elf4 phdr 重定向</td><td>全部 NAK 0x21/0x25（前置验）</td></tr><tr><td>Sahara MEMORY_DEBUG hello</td><td>NAK 0x1b</td></tr><tr><td>firehose peek / poke</td><td>NAK（<code>Peek is disabled on secure boot devices</code> 硬门控）</td></tr><tr><td>firehose configure / read / getstorageinfo</td><td>ACK（分区读可用）</td></tr></tbody></table>

fastboot 侧 33 条命令穷尽，无内存读写原语；Lenovo 解锁是 32B 挑战 + RSA-2048 服务器 token，离线不可伪造。Ed25519 nonce 复用测试：10 条不同 challenge 的 CSR，R 值零复用，实现规范。

唯一有意义的负结果之外收获：分区备份全套、CSR 里提取出了设备真实的 CDI_Leaf 公钥（`e63af59f...`，通过验证 SignedData 签名者确认）。

既然秘密只需要 "以 CSR 签名的形式出来一次"，那真正要研究的对象就不是内存隔离，而是 **TA 的命令处理逻辑**。

工具链：`keymaster.img`（KeyMint TA，ELF64 AArch64，与 live 分区字节一致）全量线性反汇编 + 字符串 xref，还原了 96 条命令的分发表和 CSR 全流程：

```
GENERATE_CSR_V2(0x3122)
  → generateCsrHandlerV2(0x4734c)
  → generateBcc(0x42484)
  → derive_device_private_key(0x3f7b4)   ; CDI_Leaf 派生点
```

审计结论（对后面很关键）：

*   CSR 输入校验其实很严：challenge ≤64B 硬校验（`0x47590: cmp x1,#0x41; b.hs err`）、memcpy 全部 min 钳制、CBOR 游标跟踪。没有直接的 OOB 读。
*   但有一个被忽略的问题：**MAC 校验函数 `parseMacTags`(0x4297c) 只被 CSR v1(0x311e) 和 CSR_V2_FIXED(0x3125) 调用，0x3122 的普通路径根本没接**。也就是说 V2 这条路上，"key 必须是 TEE 生成" 这个检查根本不存在。

设备有 KernelSU root（SELinux Enforcing，ksu 域）。没有 `/dev/qseecom`，但有 `/dev/smcinvoke`：

*   `libQSEEComAPI.so` 在本平台是 QSEECOMPAT shim，架在 MINK IPC 上，`start_app` 能用但 `send_cmd` 被对象模型拒绝；
*   真正能用的方式 = 照抄 HAL：dlopen `libkeymasterdeviceutils.so`，调 `KeymasterUtils::KeymasterUtils(TEE)` + `send_cmd`（MINK invoke op=0, counts=0x11，普通 malloc buffer），TA app name = `keymaster64`；
*   线格式（TA 入口反汇编实证）：请求 `LE32 cmd || payload`，响应 `LE32 retcode || LE32 len || payload`。

自己写了 `kmcli`（无 NDK，clang 直接链接设备上拉的 bionic `libc.so/libdl.so`），支持单发和批处理。基准测试 `0x200 GET_VERSION` 通过，`0x311f GET_ROOT_OF_TRUST` 通过。

顺手实测了两个附产物：

*   `0x219 GENERATE_FRS_AND_UDS`：**直接返回 32 字节 SHK 派生秘密（FRS），provisioning 门没拦**。审计时以为量产机会拒（-0x2711），实测完全开放。
*   `0x3126`：不是任意 MAC oracle，是 P-256 密钥对生成 + MAC 后端。

`0x3122` 请求是一个 CBOR map，**恰好两个字段**（多一个就 rc=-1）：

<table><thead><tr><th>label</th><th>内容</th></tr></thead><tbody><tr><td>4</td><td>challenge: bstr，≤64B</td></tr><tr><td>57(0x39)</td><td>keysToSign: bstr，里面是 CBOR 的 <code>[COSE_Key]</code> 数组</td></tr></tbody></table>

响应是 `{61: bstr(AuthenticatedRequest)}`，需要解包。COSE_Key 用标准 P-256 形式 `{1:2, 3:-7, -1:1, -2:x, -3:y}`。

```
[PC]  生成 P-256 密钥对（私钥自己留）
[PC]  fetchEekChain → Google 发 challenge（防重放）
[PC]  组装 {4: challenge, 57: bstr([我们的COSE_Key])}
[设备] kmcli 3122 → TA 不验 MAC，把我们公钥编进 CsrPayload
        用真 CDI_Leaf 签 SignedData，返回完整 CSR
[PC]  解包 {61: bstr} → POST signCertificates?challenge=...
[PC]  Google 返回 4 级证书链 → build_keybox_xml(私钥, 链) → keybox.xml
```

踩过的三个坑（都是 Google 400 逼出来的）：

1.  `Expected an array, got MAP` 第一次：keysToSign 必须是**数组**（`[COSE_Key]`），裸 map 不行——其实真正原因是下一条；
2.  真正的包装问题：TA 响应是 `{61: bstr}`，直接把整个 map 发给 Google 就报这个错，解包后秒过；
3.  官方 CDDL（`generateCertificateRequestV2.cddl`）里 AuthenticatedRequest 第二个元素是 `UdsCerts`（map，可空），**V2 根本没有 ProtectedData 字段**，不需要自己包 EEK。

Google 生产服务器（`remoteprovisioning.googleapis.com`）返回 HTTP 200 + 4 级链：

```
CN=<公钥哈希>,O=TEE  →  CN=Droid CA3  →  CN=Droid CA2  →  Key Attestation CA1
```

验证：叶子证书公钥 == 我们的私钥公钥 ✓；链签名逐级 VALID ✓；`keybox.xml` 与私钥配套。连跑多张全部成功。

顺带确认：证书链里**没有任何机型字段**（叶子 CN 就是公钥哈希），`fetchEekChain` 的 fingerprint 换 Pixel 也照签，DeviceInfo 由 TA 内部 provision 决定、进不了最终证书。所以 "改机型" 在这套产物里是个伪需求，真正控制 attestation 机型的是 TrickyStore 那一层。

Google 的证书语义只证明一件事：**签发时，这个公钥出现在一条由真机 CDI_Leaf 签名的 CSR 里**。

*   DICE 链签名 → 证明 CSR 来自真机 TEE（我们伪造不了，也没伪造）；
*   签名里的公钥是不是 TEE 生成的 → **只能由 TA 的 MAC 校验来担保**；
*   这版 TA 没在 V2 路径上做这个担保 → NS 世界随便什么 key 都能混进签名队列。

于是 "attestation key 必须 TEE 生成" 退化成一句空话：私钥在我们手里，但协议层面无法区分。这就是整条 oracle 路线的全部本质——**不攻破内存隔离，只利用命令语义缺口**。

*   **成立条件**：能直连 TA 命令面（本机靠 root）+ 这版 QTI KeyMint TA 固件 + Google 认可的 DICE 链。非高通平台命令格式不同，需重新逆。
*   **零副作用**：CSR 生成是只读操作，不动设备状态，不动 bootloader 锁态，不用 9008。
*   **会失效的场景**：TA 把 MAC 校验接到 V2 路径上（固件修复），或 Google 服务端改策略。种子采集路线（把 CDI_Leaf 真正抠出来）依然是更彻底的解，只是在本机所有提取通道都被堵死的情况下，oracle 是性价比最高的解。

*   `kmcli`：root 直连 KeyMint TA 的命令行客户端（单发 / 批处理 / 响应落盘）
*   `rkp_oracle.py`：一键 keybox pipeline（fetchEekChain → kmcli → signCertificates → keybox.xml）
*   TA 静态分析库：96 命令表、CSR 全流程反汇编、seed 定位（`base+0x67761`）
*   9008 验签地图与 Sahara/firehose 探针组

_【声明：本文章只用于学习，请勿非法使用，任何违法事情与本人无关】_

[冰与火的战歌：Windows 内核攻防实战高级班！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)

最后于 5 天前 被西瓜帅编辑 ，原因：