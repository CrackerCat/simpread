> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271697.htm)

> [翻译] 三星 Android OS 加密漏洞分析：IV 重用攻击与降级攻击

Abstract
--------

基于 ARM 的 Android 智能手机依靠 TrustZone 硬件支持可信执行环境 (TEE) 来实现安全敏感功能。 TEE 与 Android 并行运行一个独立的、隔离的 TrustZone 操作系统 (TZOS)。 TZOS 中的加密功能的实现留给了设备供应商，他们创建了专有的无文档化设计

 

这项工作揭示了三星 Galaxy S8、S9、S10、S20 和 S21 旗舰设备中 Android 硬件支持的密钥库的加密设计和实现。本研究通过逆向工程、对密码设计和代码结构的详细描述，来揭示了其设计缺陷。

 

本文提出了一种针对 AES GCM 的 IV 重用攻击（IV Reuse Attack，允许攻击者提取受硬件保护的 key material）以及一种降级攻击（Downgrade Attack）。即使是最新的三星设备也容易受到 IV 重用攻击。此外，还展示了对最新设备的有效密钥提取攻击（Key Extraction Attack），以及对 TrustZone 和远程服务器之间的两个更高级别加密协议的攻击影响：一个有效的 FIDO2 WebAuthn 登录绕过和谷歌安全密钥导入（Secure Key Import ）的破坏。

**0x01 Introduction**
---------------------

除了在许多和各种日常活动中的使用，智能手机越来越多地用于许多安全关键任务，例如保护敏感数据（消息、图像、文件）、加密密钥管理、FIDO2 网络身份验证、数字版权管理（DRM）、移动支付服务（例如，三星支付）和企业身份。

 

智能手机变得越来越复杂，并呈现出越来越大的攻击面。结果是它们已成为恶意软件和恶意攻击者的主要目标。已经有许多公共漏洞允许攻击者提升 Android 操作系统中的权限，以 root 甚至操作系统内核的身份获得执行。理想情况下，此类攻击不应危及设备的安全关键任务。

 

可信执行环境 (TEE) 主要用于现代移动设备，为可信应用程序 (TA) 的执行提供隔离环境，可以安全地执行安全关键任务。他们有一个相对较小的代码库和有限的 API。相比之下，丰富的执行环境 (REE)，如 Android 操作系统，不能被完全审计和信任（由于复杂性）。隔离 TEE 可与 REE 一起使用，以实现安全敏感功能。这使得攻击者更难破坏这些功能，因为攻击面显着减少并且仅限于与 TEE 的通信。

 

换句话说，TEE 的目标是抵御来自完全受损的 REE 的攻击，包括来自具有内核或根功能的特权对手的攻击。 ARM 是移动和嵌入式市场中使用最广泛的处理器 ，它通过 ARM TrustZone 提供 TEE 硬件支持。 TrustZone 将设备分为两个执行环境：

 

1）运行 “Normal World” 操作系统的非安全 REE；

 

2）运行 “Secure World” 操作系统的安全 TEE。

 

REE 和 TEE 使用单独的资源（例如，内存、外围设备），并且硬件强制执行 Secure World 的保护。在大多数移动设备中，Android 操作系统运行不安全的 Normal World。至于安全 Secure World，还有更多的选择。即使在三星设备中，也至少使用了三种不同的 TrustZone 操作系统 (TZOS)。

 

Android Keystore 通过三星等供应商实施的硬件抽象层 (HAL) 提供硬件支持的加密密钥管理服务。 Keystore 向 Android 应用程序公开 API，包括加密密钥生成、安全密钥存储和密钥使用（例如，加密或签名操作）。三星通过称为 Keymaster TA 的受信任应用程序 (TA) 实施 HAL，该应用程序在 TrustZone 中运行。Keymaster TA 使用硬件外围设备（包括加密图形引擎）在 Secure World 中执行加密操作。

 

Keymaster TA 的安全密钥存储使用 blob：这些是存储在 REE 文件系统上的 “wrapped”（加密）密钥。密钥的“wrapping”、“unwrapping” 和使用是在 Keymaster TA 内使用设备唯一的硬件 AES 密钥完成的。只有 Keymaster TA 才能访问 key material；Normal World 应该只看到不透明的键 blob。尽管严格验证和测试这种加密设计是至关重要的，但现实世界的信任区域实现在文献中得到的关注相对较少。这主要是因为大多数设备供应商没有提供其 TZOS 和专有 TA 的详细文档，并且几乎没有分享有关如何保护敏感数据的信息。本研究对几代三星 Keymaster TA 的完整密码设计和 API 进行了逆向工程，并提出了以下问题：

 

即使在 Normal World 受到威胁时，基于硬件的加密密钥保护是否仍然安全？这种保护的密码设计如何影响依赖其安全性的各种协议的安全性？

 

![](https://bbs.pediy.com/upload/attach/202203/782560_2QVNTC44ZU6WNU5.png)

 

研究员于 2021 年 5 月向三星移动安全报告了对 S9 的 IV 重用攻击。2021 年 8 月，三星为该问题分配了高严重性的 CVE-2021-25444，并发布了一个补丁程序，通过删除添加自定义 IV 的选项来防止恶意 IV 重用 来自 API。 三星修补的设备列表包括：S9、J3 Top、J7 Top、J7 Duo、TabS4、Tab-A-S-Lite、A6 Plus、A9S。

 

在 2021 年 7 月报告了针对 S10、S20 和 S21 的降级攻击。2021 年 10 月，三星为降级攻击分配了高严重性的 CVE-2021-25490，并修补了搭载 Android P OS 或更高版本的机型，包括 S10、S20、 和 S21。 该补丁完全删除了旧的密钥 blob 实现。

**0x02 Background**
-------------------

### A. AES GCM

高级加密标准 (AES) 是使用最广泛的对称分组密码。Galois 计数器模式 (GCM) 是一种用于块密码的操作模式，可提供经过身份验证的加密。 AES-GCM 是一种流密码，它在内部使用 AES-CTR（计数器模式）和 Galois 消息验证码 (GMAC)。

 

像每个流密码一样，AES-GCM 容易受到初始化向量 (IV) 重用攻击。当使用相同密钥加密时重用 IV 时，生成的密钥流是相同的。在这种情况下，对一种明文的了解会立即揭示另一种明文。此外，在 AES-GCM 中，Joux 展示了如何利用 IV 重用来破坏身份验证并创建新的有效消息。

### B. ARM TrustZone

ARM TrustZone 技术添加了一个额外的虚拟处理器模式，称为 “Secure World”，以补充“Normal World”。这两种模式是分开的，可以使用“Secure Monitor”（运行在最高 EL3 执行级别）或“World Shared Memory” 的内存映射进行通信。这种分离允许实现 TEE，因为受损的 Normal World 将无法访问 Secure World 的内存。下图显示了 TrustZone 架构中每个异常级别的组件。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_3ZZYH3B623JG72W.png)

 

由于 TZOS 的实现留给供应商，因此各个供应商有多种实现方式，包括：

 

• Qualcomm 的 Qualcomm 安全执行环境 (QSEE)：用于 Google Pixel 设备和三星 Galaxy 设备的 Snap Dragon 型号。

 

• Trustonic 的 Kinibi：用于 S10 之前的三星 Galaxy 设备的旧 Exynos 型号。

 

• 华为的 TrustedCore (TC)。

 

• 三星的 TEEGRIS（用于三星 Galaxy 设备的较新 Exynos 型号）。

### C. 可信应用

可信应用程序 (TA) 是在 TEE 中运行并向 Android 客户端应用程序公开安全服务的程序。应用程序可以打开与 TA 的会话并在会话中调用命令。接收到命令后，TA 解析输入的命令，执行所需的处理并将响应发送回客户端。控制通过专用的 SMC（安全监视器调用）指令传递给 TA，TA 和 Normal World 应用程序通常使用称为 World Shared Memory 的共享内存缓冲区交换参数和输出。由于执行 SMC 需要 EL1 权限，Android 内核中的设备驱动程序处理与 TA 的通信并为 Normal World 应用程序公开一个 API。

### D. Android 硬件支持的密钥库

Android Keystore 允许 Normal World 应用程序执行加密操作，同时保护加密密钥不被提取或未经授权使用。在采用 TrustZone 技术的设备上，Keystore 利用 TEE 执行加密操作。这种硬件支持的密钥库实现称为 Keymaster TA。 Keystore 的主要功能是：密钥生成和导入、非对称加密 / 解密 / 签名 / 验证、对称加密 / 解密和对称 MAC 的生成 / 验证。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_RMR7XHUWRQ7SPNC.png)

 

Keymaster TA 可以在 TrustZone 内生成和使用密钥，因此可以防止任何 Normal World 的攻击者。然而，Keymaster TA 依赖于 Normal World 应用程序来存储密钥。为了保护 key material，密钥在 TrustZone 内使用硬件衍生密钥进行加密或 “wrapping”。然后将加密的密钥“blob” 传递到 Normal World 进行存储。上图显示了使用 Keymaster TA 的 Android 应用程序的简化概述。一般流程如下：

 

1）Android 应用程序请求生成一个新的密钥。 Keymaster TA 生成一个新密钥，并使用硬件衍生密钥对其进行加密。生成的加密密钥 Blob B 将传递回 Android 应用程序。

 

2）Android 应用程序将 blob B 保存在 Normal World 的文件系统中。

 

3）WebAuthn 等一些协议要求 Android 应用程序必须证明特定密钥是由 Keymaster TA 安全生成的。在这些情况下，应用程序可以将加密的 blob B 传递给 Key master TA 并要求它生成证明证书。

 

4）应用程序可以要求 Keymaster TA 代表它执行加密操作。例如，要对消息进行签名，应用程序将加密的 blob B 和要签名的消息传递给 Keymaster TA。 Keymaster TA 解密 blob 以恢复签名密钥。然后它使用密钥对消息进行签名并将生成的签名返回给 Normal World 中的应用程序。

 

Keystore 使用以下措施保护密钥不被提取：

 

1）应用程序内存中不存在 key material，因此损害应用程序不会导致 key material 提取。

 

2）Keystore 可以提供访问控制，以各种方式限制密钥的使用，例如限制密钥的用途（例如，仅加密）、设置到期日期、速率限制或要求用户认证（例如，通过生物特征认证 / 解锁屏幕）。

 

3）硬件绑定：支持的设备可以将密钥绑定到安全硬件（即 TEE），因此它们不能在该设备上的安全硬件之外使用。

 

为了支持多个 TZOS 实现，Android Keystore 使用 HAL。每个 TZOS 都为需要硬件支持的 Android 服务实现 HAL，通常是通过一个 TA 来执行所需的操作。三星的 Keymaster TA 就是这种情况，它用于实现 Android 硬件支持的密钥库，是研究的主要重点。

### E. Android 密钥认证

Keystore Key Attestation 允许远程方通过让 Keymaster TA 生成根证书为 Google（或三星，用于 KNOX 证明等应用程序）的证书链来验证密钥是否在安全硬件中生成。这允许远程方信任在 TrustZone 中生成的公钥而不信任 Normal World，尽管与 Keymaster TA 的所有通信都经过 Normal World。安全密钥导入和 FIDO2 WebAuthn 正是为此目的使用密钥证明（使用 Google 的根证书）。

### F. 攻击模型

在本文中假设攻击者可以完全破坏 Normal World，例如，具有 root 甚至内核权限的攻击者。此外，假设攻击者能够在不设置由 Keymaster 证明的引导加载程序 fuse 的情况下破坏 Normal World（例如，引导加载程序解锁，三星的保修位）。攻击者旨在破坏受信任 Normal World 保护的数据，例如 Keystore 硬件保护密钥或依赖远程证明的更高级别协议（例如，克隆 FIDO2 令牌或通过窃取安全导入的密钥）。TrustZone 和 Keymaster TA 提供的加密密钥的硬件保护和隔离应该可以防止任何密钥泄露。

 

请注意，攻击不需要在 Android 内核中实际运行代码。攻击只需要在 EL0（Android 用户模式）中执行代码，并具有足够的权限来读取密钥 blob，以及适当的 SELinux 权限来与 TZOS 驱动程序通信。例如，Android Keystore 用户模式守护进程 / HAL 中的漏洞可能就足够了。或者，可以将根恶意软件或修补 Keymaster HAL 的供应链攻击用于这两种攻击。

 

在实验中使用了三星 Galaxy S9、S10 和 S21 设备，这些设备使用 Magisk 进行根植。请注意，当植根设备时，设置了三星 KNOX Warranty fuse。这不会影响攻击，因为不针对任何 KNOX 功能。

0x03 **Dissecting the Keymaster TA**
------------------------------------

### A. Keymaster TA 家族的调查

重点介绍三星对 Keymaster TA 及其名为 TEEGRIS 的新 TZOS 的实施。与其他 TZOS 和 TA 一样，三星的实现是特定于供应商的，并且是一个专有的闭源系统，几乎没有文档可用。为了理解 Keymaster TA 实现的加密图形设计，使用 Ghidra 静态分析了 3 个 TZOS（TEEGRIS、Kinibi 和 QSEE）中固件和 TA 的二进制文件。此外使用三星开源网站为的设备下载 Android 内核源代码。

 

总体而言，评估了 2018 年和 2021 年间发布的 Exynos 和 Snapdragon 型号的 S8、S9、S10、S20 和 S21（包括 S9+、Note9、S10+、S20+ 等变体）的 26 个固件。发现 Keymaster TA 的代码库在这些固件（S8 除外）中极为相似，即使跨 TZOS（Kinibi/QSEE/TEEGRIS）也是如此。

 

在分析中注意到 S8 中的 Keymaster TA 与较新型号中的显着差异。S9 中引入的一项特定代码更改使其和所有较新的模型都容易受到攻击。尽管 S20 和 S21 型号包含更安全的 Strongbox Keymaster 功能（使用专用的防篡改硬件安全模块），但它们仍然容易受到攻击，因为它们共享相同的易受攻击的密码设计和 API。由于安全社区对 Kinibi 和 QSEE 进行了更深入的研究，因此在讨论细节时，除非另有说明，否则将参考 TEEGRIS。

### B. Keymaster HAL

Keymaster HAL 是 Android 与特定于供应商的 Keymaster 实现之间的接口。 Android 文档为 Keymaster HAL 的实现者提供了参考指南。它在 Android 用户模式下实现，并使用内核驱动程序和 Normal World 共享内存缓冲区与 Keymaster TA 通信（下图）。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_XE9VNNUF7YWET2J.png)

 

Android 为 Keymaster 应该实现的功能提供了一个开源 API。此 API 包括许多功能，例如密钥生成、密钥导入和加密操作，例如使用存储在加密 blob 中的密钥进行加密 / 解密 / 签名 / 验证。

 

TEEGRIS 中的 Keymaster HAL API 在许多使用 TEEGRIS 内核驱动程序的未记录共享对象中实现。为了探索 Keymaster TA，对 Keymaster HAL 进行了逆向工程，并实现了自己的 Keymaster 客户端——一个 Android 进程，无需任何输入验证或过滤即可将自定义请求发送到 Keymaster TA。

### C. 密钥 Blob 加密

研究的主要焦点是如何解密 / 加密密钥 blob。基于对专有 Keymaster TA 的分析和逆向工程，现在将在高层次上描述该过程。Keymaster TA 对 blob 内的 key material 进行加密。这可以保护 key material 不被提取，同时允许其由 Normal World 应用程序存储。正如在研究中发现的那样，每个 blob 都包含一个加密部分，其中包括 key material 和各种参数。该 blob 还包含一个明文部分，其中包含解密所需的信息，例如加密中使用的 IV 和 AAD。

 

Keymaster TA 所依赖的加密基础是永久的、硬件的、设备唯一的 256 位 AES 密钥，称为根加密密钥 (REK)。此密钥仅存在于安全硬件加密引擎中——Keymaster TA 无法直接访问它。每个密钥 blob 都使用从 REK 派生的自己的硬件派生密钥 (HDK) 进行加密。 Blob 的 HDK 是使用密钥导出函数 (KDF) 导出的，该函数将 REK 与特定于 Blob 的盐值混合。 KDF 本身由 Keymaster TA 通过 TZOS 内部的 ioctl API 访问。用于推导 HDK 的盐值被计算为几个值串联的 SHA-256 摘要。正如接下来讨论的那样，三星的盐值生成方法在设备型号之间演变。

### D. 密钥 Blob 的 KDF 版本

美国国家资讯安全保证联盟 (NIAP) 创建了移动设备基础保护配置文件 (MDFPP)，其中包括设备的核心安全要求。盐值派生中使用的常量字符串（在所有关键 blob 类型中）表明三星符合认证，而且三星设备确实通过了 MDFPP CC 认证。

 

在分析中确定了三种不同的 blob salt 衍生版本，称之为 v15、v20-s9 和 v20-s10。版本之间的差异在于字符串中使用的值，这些值经过哈希处理以生成用于密钥派生的盐值。所有版本都可以包含称为应用程序 ID 和应用程序数据的可选值。这些值由 Normal World 设置。下图显示了三个 blob 版本中的每一个的示例字符串。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_PSFDQQ6RYEY56TM.png)

 

第一个 blob 密钥版本，称之为 v15，是 Galaxy S8 中使用的版本。 v15 密钥 blob 中的盐值仅取决于应用程序 ID 和 Normal World 设置的数据（以及常量字符串）。 Galaxy S9 推出了一个新版本，称之为 v20-s9。这个版本向 SHA-256 摘要添加了两个新值，称之为 root_of_trust 和 integrity_flags。这可能是由于新的 MDFPP 法规要求派生密钥绑定到设备的完整性。

 

root_of_trust 是 “定义有关设备状态的关键信息的值的集合”。 Keymaster TA 根据设备的完整性状态计算完整性标志（正常设备应该为 0，有根设备为 7）。在分析中，注意到虽然 S9 默认使用新的 salt 版本 v20-s9，但它仍然包含实现旧 v15 blob 版本的代码。 Keymaster TA API 向 Normal World 公开了使用这个旧版本的选项。由于无法在 Normal World 中找到此选项的任何用途，因此认为这是从未使用过的潜在代码。

 

在 S10、S20 和 S21 设备上发现了一个修改后的 KDF，它与 v20-s9 非常相似：它使用完全相同的字符串、信任根和完整性标志，还有一个重要的补充：在 v20-s10 中，盐值也包括在 TrustZone 内生成的新的 per-blob 16 字节随机值 hek_randomness。与 S9 类似，还发现了实现 v15 并暴露于 Normal World 的潜在代码。

0x04 **Attacking the Keymaster TA**
-----------------------------------

本节描述了两种针对 Keymaster TA 的攻击，它们允许提取受硬件保护的 key material。下表总结了检查过的三星 Galaxy 设备及其对 IV 重复使用的敏感性。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_WVUPAUSKWPWBUHB.png)

### A. 对 v15 和 v20-s9 Blob 的 IV 重用攻击

用于加密密钥 blob (HDK) 的包装密钥是使用 Keymaster TA 计算的盐值导出的。在 v15 和 v20-s9 blob 中，salt 是一个确定性函数，仅依赖于 Normal World 客户端完全控制的应用程序 ID 和应用程序数据（以及常量字符串）。这意味着对于给定的应用程序，所有密钥 blob 都将使用相同的密钥进行加密。由于 blob 在 AES-GCM 操作模式下进行加密，因此生成的加密方案的安全性取决于其 IV 值永远不会被重用。

 

令人惊讶的是，Android 客户端允许在生成或导入密钥时设置 IV。只需将攻击者选择的 IV 作为关键参数的一部分，Keymaster TA 会使用它而不是随机 IV。由于 Normal World 还控制着应用程序 ID 和应用程序数据，这意味着攻击者可以强制 Keymaster TA 重用之前用于加密其他 v15 或 v20-s9 blob 的相同密钥和 IV。由于 AES-GCM 是一种流密码，攻击者现在可以从密钥 blob 中恢复受硬件保护的密钥。

 

给定一个包含未知密钥 KA 的密钥 blob BA，攻击者可以导入另一个具有已知（足够长）密钥 KB 的密钥 blob BB，该密钥 KB 使用相同的 IV 和相同的盐值进行加密。为此，攻击者首先从密钥 blob BA 中提取 IV，并将此 IV 和相同的应用程序 ID 和数据传递给 Keymaster TA 的导入密钥函数。已知 KB 将使用与加密 KA 相同的 blob 加密密钥 HDK（因为 salt 仅取决于应用程序 ID 和数据）和相同的 IV（通过构造）加密。与任何流密码加密一样，现在可以使用对 KB 的知识以及密文 BA 和 BB 来恢复 KA。将具有给定 IV 的密钥 HDK 创建的密钥流表示为 E(HDK,IV)，然后：

 

![](https://bbs.pediy.com/upload/attach/202203/782560_H2NCYTAGYK6NECU.png)

 

在 Galaxy S9 上创建的所有关键 blob 都是易受攻击的，因为其默认 blob 版本是易受攻击的 v20-s9。然而，在 S10、S20 和 S21 设备上，默认版本是 v20-s10，其 salt 由 hek_randomness 字段随机化，因此每个 blob 都使用唯一派生的加密密钥进行加密。虽然攻击者仍然可以导致 IV 重用，但这种重用是不可利用的。注意到，虽然 S8 只能创建 v15 版本的 blob，但它的 Keymaster TA 忽略了密钥生成和密钥导入功能中的 IV 参数，即它不允许设置 IV，因此不容易受到本研究的任何攻击。

 

为了演示 IV 重用攻击，在 Galaxy S9（所有密钥 blob）、S10 和 S21（通过强制创建 v15 密钥 blob）上实现了它，并且能够恢复安全生成的 AES、RSA 和 ECDSA 来自加密 blob 的密钥。

### B. 降级攻击

在 S10 及更高型号上，默认 blob 版本为 v20-s10。然而，发现了允许普通世界通过简单地传递具有特定值的 “加密版本” 参数来请求创建 v15 blob 的潜在代码的存在。尽管 Keymaster HAL API 通常不传递此参数，但特权攻击者可以利用它的存在来强制所有新 blob 为 v15 版本。

 

这种降级攻击使更新的设备，包括 Galaxy S10、S20 和 S21 容易受到 IV 重用攻击（在 Normal World 被破坏之后）。在后文将展示可以利用它来攻击安全关键协议，例如 Google 的 Secure Key Import 和 FIDO2 WebAuthn。

 

为了证明本研究发现，编写了一个 GDB 脚本，它在 Normal World 中拦截对 Keymaster HAL 的调用。挂钩了密钥生成函数，并在飞行中修改了传递的参数，以添加加密版本参数，值为 0x f 表示 v15。这导致 Keymaster TA 始终生成 v15 密钥 blob，并允许使用 IV 重用攻击来恢复加密密钥。恶意软件作者可以通过其他方式实现类似的效果，例如始终返回预先计算的 v15 密钥 blob，或者可能修补 Keymaster HAL，使其始终降级到 v15 blob。

### C. v15 Blob 的持久性

根据 Keymaster API，当 Keymaster 设备更新或 Normal World OS 升级到更新版本时，密钥 blob 会变得 “old”。当密钥“old” 时，API 函数会返回一个特殊的错误代码，指示密钥必须 “upgraded”。密钥主 API 公开 upgradeKey 方法，该方法解包（解密）密钥，检查密钥参数中的操作系统版本并将它们与当前操作系统版本进行比较。如果当前操作系统版本更高，它会通过再次包装（加密）密钥来“upgraded” 密钥（并将当前操作系统版本添加到密钥参数列表中）。

 

但是，发现新 blob 包装中使用的密钥参数与旧密钥中的相同。由于降级攻击创建的任何密钥 blob 都包含加密版本参数，因此 “升级” 此类降级的 v15 密钥 blob 将产生一个新的但仍然易受攻击的 v15 blob。根据三星的说法，这是预期的行为，并且由于 S10 和更新的设备将 v20-s10 作为默认版本，因此不应该有在野 v15 密钥。然而，upgradeKey 函数的这种行为允许降级通过固件更新持续存在。

0x05 **Implications of the Attacks**
------------------------------------

### A. 认证和确认绕过

Keymaster TA 可用于强制限制使用加密密钥，以防止在未经用户同意或不知情的情况下滥用密钥（例如，通过控制 Normal World 的攻击者）。例如，可以限制密钥用于特定用途，例如仅用于签名。 此外，应用程序可以创建 “身份验证绑定” 密钥，要求用户最近通过身份验证才能使用该密钥，例如，通过指定自用户上次输入密码后的超时时间，或要求生物特征提示身份验证。

 

Keymaster TA 在尝试使用 blob 时会强制执行使用和身份验证要求。这可以防止攻击者在未经用户同意的情况下使用设备上的密钥——例如，如果用户未经身份验证，安全硬件可以拒绝使用密钥。类似地，受保护的确认仅在用户提供对要签名的数据的确认（通过受信任的 UI 界面）时才允许使用签名密钥。 Google 讨论了受保护确认的一些用例，包括医疗应用程序，例如用于注射胰岛素的 Big foot Biomedical，以及企业应用程序，例如包括 Duo Mobile 的双因素身份验证。

 

但是，这些限制的安全性是基于无法从 TrustZone 内部提取受保护密钥的假设。攻击者可以使用本文攻击来提取密钥并完全绕过任何限制。例如，他们可以在不知道用户密码的情况下使用 “authentication-bound”，或者在没有用户交互或同意的情况下签署支付交易。此外，即使他们不再有权访问设备，他们也可以继续使用这些密钥。如果攻击者只能在有限的时间内物理访问设备，这将特别有用。

### B. 从安全密钥导入中提取密钥

Keymaster TA 支持安全密钥导入，它允许应用程序将现有密钥安全地配置到密钥库中。该协议的目标是允许服务器安全地与 Android 设备共享密钥，同时防止密钥被截获或从设备中提取，即使设备受到损害。密钥由服务器加密，仅在安全硬件内部解密。因此，它绑定到设备，允许它在各种场景中使用，例如 SSH/RDP、DRM、安全支付等。例如，Google Pay 使用安全密钥导入来配置其一些密钥。

 

安全密钥导入允许服务器安全地将一些 key material K 发送到设备。协议的简化版本和攻击如下图所示。安全密钥导入的工作原理如下：

 

1）应用程序请求 Keymaster TA 生成一个 RSA 私钥 - 公钥对 (Pub,Priv)，并接收到一个包含 Priv 的加密密钥 blob BRSA。

 

2）应用程序还向 Keymaster TA 请求证明证书 Cert，以验证 BRSA 是在安全硬件内部生成的。 Cert 使用专用私钥进行非对称签名，该私钥只能在 TrustZone 内部访问，其对应的公钥由 Google 签名。

 

3）应用程序将证书发送到服务器。验证 Cert 后，服务器使用 Pub 对 K 进行加密，生成 C = EncPub(K)，并将 C 发送给设备。

 

4）为了完成导入过程，应用程序将 C 和 BRSA 传递给 Keymaster TA。 Keymaster TA 解密 BRSA，并使用 Priv 解密 C 并恢复 K。然后它向应用程序返回一个包含 K 的加密密钥 blob BK。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_NVH39U4XADN52PR.png)

 

虽然 Secure Key Import 保护传输中的密钥，但在导入后，它们在密钥 blob 中与其他导入的密钥一样被加密。因此，可以恢复 key material（如本文的攻击）的攻击者可以解密安全导入的密钥并破坏使用安全密钥导入的应用程序的安全性。

 

和以前一样，任何导入 Galaxy S9 的密钥 K 都可以使用对加密 blob BK 的 IV 重用攻击来提取。但是，与常规密钥导入和密钥生成功能不同，安全密钥导入 API 调用不允许指定密钥 blob 的版本，使其能够抵御降级攻击。为了能够在较新的 S10、S20 和 S21 上破解安全密钥导入，需要使用不同的方法：不是从 BK 中提取密钥，而是针对 BRSA 执行降级和 IV 重用攻击，并使用恢复的私钥 Priv 解密 C 并恢复传输中的加密密钥，如上图所示。

 

由于 BRSA 是在 TrustZone 中安全生成的 - 作为 v15 blob - Keymaster TA 功能将很高兴地证明其有效性并生成有效的证书，这能够继续安全导入过程。当从服务器接收到加密的密钥 C 时，可以使用 Priv 执行与 Keymaster 提取导入的密钥 K 相同的解密过程。在 Galaxy S10 和 S21 的概念验证中演示了这种攻击，方法是对包装密钥 BRSA 执行降级攻击。拦截生成 RSA 密钥的请求，并对其进行修改以将生成的 RSA 密钥包装在具有 v15 版本的 blob BRSA 中。然后继续使用 IV 重用攻击来恢复 Priv。

### C. 绕过 FIDO2 WebAuthn

FIDO2 WebAuthn 是 W3C 和 FIDO 的规范，它允许创建和使用公钥加密来注册和验证网站而不是密码。 身份验证密钥可以在称为 “platform authenticator” 的内部安全元件（例如，TrustZone、可信平台模块（TPM））或称为 “roaming authenticator” 的外部安全元件（例如，Yubikey 和 Solo）。 这种安全元素旨在提供两个主要的安全保证：

 

1）攻击者不应该能够从安全元件中提取密钥：这意味着只能使用安全元素进行身份验证，并且无法克隆。

 

2）身份验证可能需要用户在场：例如，可以要求用户按下按钮或使用生物特征扫描进行身份验证。

 

Android 设备可以使用由 Android 硬件支持的密钥库来提供类似的安全保证，使用 TrustZone 而不是专用的外部硬件安全元件。 WebAuthn 包括两个主要阶段（见下图）：

 

1）注册**：**设备创建密钥对并向 Web 服务器发送证明。如果证明通过验证，则服务器将公钥与用户相关联。

 

2）断言（Assertion）：当用户尝试登录时，服务器向设备发送挑战，设备需要用户在场和身份验证（例如，生物识别提示），并且在收到用户同意后，设备使用私钥签署挑战。如果服务器验证签名（使用公钥） - 用户已登录。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_HKBZC5J38W256RE.png)

 

适用于 Android 的 FIDO2 WebAuthn 实现使用硬件支持的 Android 密钥库在注册期间生成密钥和验证密钥，并在登录时执行需要用户确认的断言（使用密钥 blob 签名）。当使用硬件支持的密钥库时，WebAuthn 应该能够承受 Normal World 的破坏。与安全密钥导入中所做的类似，（Pub,Priv）密钥对由 TrustZone 内的 Keymaster 生成，并且应用程序仅接收包含密钥对的加密密钥 blob BAUT H。应用程序将证明证书（包括公钥）发送到 FIDO 服务器。当需要身份验证时，应用程序将服务器的质询和 BAUT H 传递给 Keymaster TA，要求它使用 Priv 对其进行签名。

 

可以使用攻击来提取用于身份验证的私钥，并违反预期的安全保证。这可能允许攻击者克隆 “platform authenticator” 并允许来自其他设备的证明，而无需进一步访问目标智能手机。此外，可以在没有用户在场或同意的情况下使用恢复的私钥对网站进行身份验证。此攻击类似于之前针对安全密钥导入的攻击：在 Galaxy S9 上，使用 IV 重用攻击从 BAUT H 密钥 blob 中提取 Priv；在 Galaxy S10、S20 和 S21 上，必须首先执行降级攻击。协议的简化版本和攻击如上图所示。在高层次上，out 攻击的工作原理如下：

 

1）当设备注册到网站（例如，Pay pal.com）时，攻击者使用降级攻击。他们拦截生成密钥的请求并对其进行修改以强制 blob BAUT H 中生成的密钥使用 v15 KDF 方法。

 

2）攻击者使用 IV 重用攻击来提取密钥块的 key material。

 

3）攻击者现在可以通过使用私钥签署断言挑战来静默地对网站进行身份验证——无需用户确认。

 

使用 StrongKey FIDO 示例 Android 原生应用程序和 FIDO 客户端演示了对三星 Galaxy S10 的攻击。StrongKey 系统有两个组件：运行 FIDO(R) Certified StrongKey FIDO Server 的 Linux 服务器和客户端应用程序，在例子中是示范的 Android 应用程序。将应用程序安装在设备上而不做任何修改，并使用它来注册和验证 StrongKey 的演示服务器。当用户注册到网站时，Android StrongKey 应用程序请求通过 Keymaster HAL 生成密钥。使用降级攻击，对函数 nwd_generate_key 的调用被调试器拦截，生成的 blob 被强制为 v15 blob BAUT H。然后对这个 blob 使用 IV Reuse 攻击并恢复其私钥 Priv。

 

为了验证攻击，根据证明证书中的公钥验证了恢复的私钥 Priv。使用此私钥，攻击者能够伪造签名并绕过断言阶段。为了完成演示，创建了示例应用程序的替代修改版本，它使用恢复的私钥 KP 而不是使用 Keystore 对网站的挑战进行签名——见下图：

 

1）图 a 显示了如何将调试器附加到 Keymaster HAL 进程；

 

2）图 b 显示了降级攻击的 GDB 输出；

 

3）图 c 显示在未修改的应用程序中成功注册了 FIDO 服务器；

 

4）图 d 显示攻击者使用替代应用程序成功地进行了身份验证；

 

5）图 e 和图 f 显示了在替代应用程序中重新认证以批准交易的示例。

 

![](https://bbs.pediy.com/upload/attach/202203/782560_Z54AW243KEK2PX3.png)

 

在演示中，没有通过 StrongKey 对 Android 示例应用程序进行更改：拦截是在应用程序外部完成的，而替代应用程序（用于登录期间的断言）需要进行最小更改才能使用恢复的密钥。注册和认证是针对 StrongKey 自己的演示服务器完成的。

**0x06 Discussion**
-------------------

### A. 低级加密问题

Keymaster 设计中的一个基本问题是流密码 AES-GCM 的选择。 GCM 的优势，即快速和可并行化，对于 blob 加密似乎不太重要——它总是伴随着缓慢的 I/O 操作；而正如所展示的，它对密钥流重用的敏感性令人担忧。如果设计人员选择保留 AES-GCM 作为构建块，那么使用抗随机误用的 AEAD（例如 AES-GCM-SIV）应该可以防止 IV 重用攻击。

 

IV 重用攻击的根本原因是 Keymaster TA 提供的 API 允许普通世界设置 IV 的值。 Keymaster API 不应允许用户设置 IV，而是始终生成一个随机的 12 字节 IV。现代加密库，如 Tink 和 Google 的 Trusty Keymaster 实现在内部处理 IV 而不将其暴露在 API 中，从而保护用户免受已知陷阱的影响。

 

降级攻击的根本原因是 API 允许用户选择 blob 版本。不应让用户选择可能影响加密 blob 安全性的任何选项，尤其是在用户可能是恶意的情况下。此外，在 Keymaster TA 等安全关键应用程序中存在潜在代码会增加应用程序的攻击面大小，应该避免。正如所展示的，将此类潜在代码暴露给外部 API 会招致攻击者的利用。

 

此外，不应该允许在 blob 中持续降级到 v15。由于已经支持 “升级” 旧加密 blob 的过程，它还应该使用最新的（并且希望是最安全的）加密选项来加密新的 blob。设计合理的 “升级” 可以缓解许多降级攻击。

 

积极的一面是，尽管 API 仍然允许设置 IV，但在 Keymaster TA 中使用内部随机性作为密钥派生过程的一部分使加密过程更加健壮并实际上阻止了攻击。最新的三星设备仍然易受攻击的原因是 blob 版本能被降级为不随机化密钥派生的版本。

### B. 可组合性：证明的差距

在 FIDO2 WebAuthn 和安全密钥导入等协议中，受信任的远程服务器使用密钥证明来验证密钥是在安全硬件中生成的。正如为三星设备和 Busch 等人展示的那样。华为的漏洞证明，TEE 实现可能存在缺陷，这使得攻击者能够破坏密钥。密钥证明的协议步骤应该可以缓解这种情况。问题在于 Keymaster HAL 中定义的证明不承诺用于保护密钥的加密方法。事实上，它甚至不提交 Keymaster TA 的版本号。这种差距意味着接收证明的远程服务器无法设置诸如 “仅接受使用非易受攻击的 KDF 版本保护的密钥的证明” 之类的策略。

 

远程服务器可访问的证明数据包括有关密钥的一般信息 (KeyDescription)、密钥是否受 TEE 或 HSM (SecurityLevel) 保护、密钥属性 (AuthorizationList) 以及有关设备的信息状态（RootOfTrust 和 VerifiedBootState） - 例如引导加载程序被锁定。正如最近的攻击所示，设备可以在不改变引导加载程序状态的情况下被远程入侵。

 

证明证书确实包含一个名为 osPatchLevel 的字段，它可能允许服务器识别易受攻击的设备。然而，正如所展示的，三星最新的 Keymaster 同时支持两种 KDF 方法：不安全的 v15 和安全的 v20-s10，并且 osPatchLevel 字段不指示使用了哪种方法。因此，依赖 Keymaster TA 的软件补丁级别仍可能留下误用的机会。WebAuthn 等协议的安全性取决于它与 Keymaster TA 的实现和密码设计的组合。当前使用供应商特定黑盒设计的方法使得无法分析组合物的安全性。正如所展示的，这为漏洞提供了充足的空间。

### C. 基于 TrustZone 的密钥库标准

在本文中描述的攻击突出了基于 Trustzone 的密钥库的加密图形设计中的问题可能导致的关键漏洞。然而，到目前为止，这些密码设计和协议在学术文献中并没有受到太多关注。这主要是因为当前的生态系统是基于黑盒设计的，不同供应商之间的 API 不一致和碎片化。事实上，发现本文中提出的漏洞需要大量耗时的逆向工程工作。

 

希望本文工作能够激发对 Keystore 安全性的进一步研究，并为 Keymaster HAL 和 TA 制定统一的开放标准。这样的标准可以减少当前阻碍研究人员分析密码设计和协议安全性的障碍。与 TLS 1.3 的标准化过程类似，学术界和工业界之间的合作将允许对整体设计的安全性进行正式分析。这应该包括一个细粒度的威胁模型，该模型将激发故障弹性设计。

 

例如，如果有多个密钥加密选项可用，则证明证书应提供有关用于密钥的加密方法的详细信息。这将允许服务器使用易受攻击的加密方法阻止请求，并减轻类似于降级攻击的攻击。此外，包括完整 API 和密钥加密方案的形式分析可以在标准化过程的早期检测到 IV 重用漏洞等问题。

0x07 **Conclusions**
--------------------

包括三星和高通在内的供应商对其 TZOS 和 TA 的实施和设计一直保持保密。正如本文所展示的，在处理密码系统时存在危险的陷阱。设计和实施细节应由独立研究人员仔细审核和审查，不应依赖逆向工程专有系统的难度。在这项工作中检查了三星 Galaxy S8、S9、S10、S20 和 S21 旗舰设备中 Android 硬件支持的密钥库的加密设计和实现。通过大量的逆向工程工作，可以分析多个 TZOS（TEEGRIS、Kinibi 和 QSEE）中的 Keymaster TA。

 

通过分析发现了严重的密码设计缺陷，确定了一种针对 AES-GCM 的 IV 重用攻击，允许攻击者提取受硬件保护的 key material，以及一种降级攻击，即使是最新的三星设备也容易受到 IV 重用攻击。演示了对最新设备的有效密钥提取攻击。还展示了对 TrustZone 和远程服务器之间的两个更高级别加密协议的攻击的影响：一个有效的 FIDO2 WebAuthn 登录绕过和谷歌安全密钥导入的破坏。最后注意到，本研究对高级加密图形协议的攻击适用于新设备，因为它们与低级密钥加密的可组合性产生了微妙的攻击。此外，使用脆弱的 AES-GCM 流密码进行身份验证的 blob 加密的设计选择值得讨论。这些问题进一步激发了对开放和标准化密码设计的需求。

> 参考链接：https://eprint.iacr.org/2022/208.pdf

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#逆向分析](forum-161-1-118.htm) [#漏洞相关](forum-161-1-123.htm) [#源码分析](forum-161-1-127.htm)