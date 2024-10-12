> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [wrlus.com](https://wrlus.com/android-security-learning/android-system-security-intro/)

> 恶意软件这个概念从 PC 安全时代就开始被提及，只不过 PC 安全时代的恶意软件多用于窃取数据、劫持勒索甚至只是炫耀技术。由于手机承载内容的不同，到移动时代的恶意软件有着不同的目标。

Android 系统架构和攻击面
----------------

### Android 系统架构（安全视角）

*   第三方应用：设备启用后，用户自行下载和安装的应用，只能通过系统框架来访问绝大多数系统资源。
*   系统应用：设备出厂预安装的应用，可能具有 system_app 权限、平台签名权限、特许权限。当然也可能没有特殊的权限，这种本质上和第三方应用区别不大。
*   系统框架：特指系统中的一个特殊进程 system_server，它的权限比 system_app 要高，主要功能是维护应用执行所需的环境，以及向应用提供各种系统功能的接口，这些接口以系统服务 (System Service) 的形式提供。
*   Native 库：泛指使用 C/C++ 语言开发的用户态程序，包括被 system_server 进程加载的一些 Native 库、Native 原生进程。负责管理系统底层的功能，例如系统服务、存储、各类附加芯片等，这些进程也会以系统服务的形式提供一些接口。但是出于安全起见，应用一般不允许直接访问到这些进程，而必须经过系统框架中的代理接口来进行。
*   Runtime：ART 虚拟机，在 Android 4.4 之前是 Davilk 虚拟机，这部分代码运行在应用进程中。
*   Linux 内核：Android 使用了一系列定制版的 Linux 内核分支，用于支持一些 Android 特有的功能，例如 Binder 通信机制、ashasm 共享内存等。同时也限制了很多桌面 Linux 发行版内核中的功能和攻击面。
*   TEE：依托于 ARM TrustZone 机制实现的小型子系统，独立于 Android 系统运行，专门用于实现高安全性的功能，典型的应用是锁屏密码、指纹和人脸解锁、密钥存储、DRM 版权保护等。
*   SE：Android 厂商阵营为了模仿 Apple 安全隔区 (Secure Element) 而实现的安全组件，运行在独立于主 SoC 芯片的单独安全芯片上，并运行简单的操作系统。典型实现有 Google Pixel 的 Titan M2、三星 Galaxy S20 开始搭载的安全芯片、华为 Mate 40 Pro 和 Mate 60 系列的海思安全芯片。

### Android 安全模型和边界

*   Linux UID：每个应用使用一个 UID，利用 Linux 内核的 DAC 机制进行隔离，保证了不同应用之间无法互相访问其文件，应用也无法直接访问系统进程的文件，也提供了一套可靠的权限检查方式，实际上 Android 中所有的 Permission 检查都依赖于 Linux UID 进行。
*   SELinux (SEAndroid)：Linux UID 有个最大问题就是，一旦攻击者提权到 system uid 或 root uid，就会拥有巨大的权限，为了解决这个问题 Google 在 Android 5.0 开始引入了成熟的 SELinux 机制，并且做了适当的精简以适合移动端环境，可以精细化限制 Linux 文件访问、Socket 访问、Binder 调用、Property 访问。但是要注意 SELinux 无法实现应用之间的隔离，它仅把应用分为了少数几种类型。

### Android 系统安全攻击面

#### 传统攻击面

*   mediaserver：Android 5.0 之前，mediaserver 是安全研究员的乐园，因为这类进程的输入是易于 Fuzzing 的文件输入，所以自然成为研究员的首选目标。但从 Android 5.0 引入 SELinux 以及 Android 7.0 开始对 mediaserver 相关进程进行拆分隔离之后，这类攻击面就已经难以造成巨大的危害。
*   bluetooth：在 mediaserver 漏洞少了之后，从 2017 年开始安全研究员们又开始专注于研究 bluetooth 漏洞，毕竟这可以从远程攻入手机，还无需用户交互，同时 com.android.bluetooth 这个进程到现在也没有像 mediaserver 那样进行严格的隔离，所以到现在而言 AOSP 漏洞中的 RCE 漏洞几乎都是 bluetooth 漏洞和 NFC 漏洞。
*   Linux 驱动：虽说 Android 经过了良好设计，基本不允许应用直接访问驱动，但是例如 GPU 驱动和 Binder 驱动是例外，GPU 主要是为了高效的图形渲染，而 Binder 驱动是跨进程调用的基础，所以那些喜欢 Linux 内核研究的安全研究员，就会选择这类攻击面作为目标。

#### 专有攻击面

*   应用组件漏洞：通过应用组件的漏洞攻击其他系统应用，最简单的就是各类组件未鉴权导致跨进程访问的问题，最经典的莫过于 Intent 重定向 + FileProvider 导致任意文件读写的利用方式。
*   系统服务接口：系统框架提供了很多给应用的 Binder IPC 接口，如果没有正确的权限检查，可能导致信息泄露或越权操作。
*   反序列化漏洞：可以说是 Android 专有的最精彩的漏洞利用方式，最终可以实现 system_app 任意文件读写，或接续其它漏洞继续攻击。

黑灰产对抗
-----

### 恶意软件

恶意软件这个概念从 PC 安全时代就开始被提及，只不过 PC 安全时代的恶意软件多用于窃取数据、劫持勒索甚至只是炫耀技术。由于手机承载内容的不同，到移动时代的恶意软件有着不同的目标。

#### 移动端恶意软件的类型

*   点播吸费：利用发送付费短信的权限，点播运营商服务实现吸费，由于 Android 6.0 之前没有动态权限授予机制，所以无需用户同意即可实现。
*   广告推广：随着运营商扣费项目的规范，通过直接吸费变得困难之后，黑产便更多地转向投放广告获取收益，和普通广告不同的是通过系统植入的广告软件会在手机使用过程中进行强制弹出。
*   窃取隐私：利用通讯录等权限，收集用户隐私数据特别是通讯录，可用于广告推广或其他违法用途。

#### 移动端恶意软件分发渠道

*   浏览器：从浏览器安装应用是最常规也是最简单的方式。近年来由于厂商对未知应用来源安装应用添加了很多强交互，所以从浏览器安装应用也变得困难，但对于老年以及其它不懂手机的群体来说，还是被诱导的可能。
*   应用市场：实际上国内大部分安卓应用市场的审核并不严格，特别是使用各种黑科技的应用，更是没有办法进行检测，所以有不少恶意软件就直接上传到应用市场，这样用户会更容易下载到。
*   快应用：虽然快应用本身的能力很有限，但是快应用作为一个跨端容器可以很方便地嵌入到浏览器广告甚至其它软件中，没有太多的监管同时也很容易引流到厂商的应用市场进行下载，配合应用市场上的恶意应用进行分发。

#### 后台弹窗技术

后台弹窗是 2022 年以来黑灰产使用的较新技术，Android 恶意软件可以在不进入前台的情况下弹出 Activity，从而实现弹出广告，绕过了 Android 10 以来后台 Activity 启动限制的安全设计。配合获取前台应用信息的漏洞，还可以实现在其他应用在前台时精准弹出广告，提高引流效果，AOSP 中曾出现过的经典漏洞有：

*   CVE-2020-0500：InputMethodManagerService 中不安全的 PendingIntent
*   CVE-2021-39758：VirtualDisplay 豁免漏洞
*   CVE-2022-20197：system_server Parcel 缓存复用漏洞。因为通过点击 Push 通知启动应用属于后台弹出 Activity，所用 PendingIntent 被系统附加了后台启动 Token 到 Parcel 中，而由于 Parcel 缓存机制导致最终利用 AlarmManager 功能弹窗时，之前携带后台启动 Token 的 Parcel 被当前 PendingIntent 复用导致绕过。  
    厂商出现的经典漏洞有：
*   system uid 存在可见窗口：AlarmManager 和 JobService 启动 Activity 时都会满足 system_server 的进程 bindService 到恶意软件的豁免条件，因为厂商在 system uid 的进程中实现手势导航导致被系统认为具有可见窗口，所以满足有可见窗口的 uid 通过 bindService 绑定的豁免条件，导致漏洞出现。

### 刷机解锁

刷机在 PC 时代可能不是一个问题，这可能仅仅指的是在 “电脑城” 这种地方找人帮忙重装系统，但是在移动时代刷机有了新的含义。而解锁则是移动时代新出现的概念。

#### 刷机解锁的目标

*   手机 root：无论是数码爱好者还是安全研究人员都会有的一个合理需求，早期的 KingRoot 这类软件和 iOS 越狱原理类似，都是利用内核漏洞提升到 root 权限，后来的 root 方案基本依赖于 bootloader 解锁。实际上 Google 官方和很多海外品牌（三星、索尼等）是不排斥 bootloader 解锁，只是要确保在用户授权的情况下就允许自由刷写非官方签名的镜像。然而大部分国产品牌因为盲目对标苹果，便剥夺了用户自由刷写任意镜像的权利，导致黑灰产会寻找各类 bootloader 或 BootROM 的漏洞实现自由刷写非官方签名的镜像和 root。
*   灌装 / 定制 ROM：早期 Android 没有安全启动机制的时候，线下手机销售门店会和黑灰产合作推广应用获利，这些应用会直接注入到手机 ROM 包中，用户在未 root 的情况下无法卸载。Android 安全启动普及之后这条路基本行不通。
*   手机洗白（手撕）：在 iOS 推出 “查找” 功能允许即使刷机后也能通过 Apple ID 阻止激活的安全特性之后，Android 厂商纷纷效仿，然而因为技术能力有限导致频频出现绕过的情况，这种手法称为手机洗白。因为主要的目的是刷写被盗的手机并允许二手出售，所以俗称“洗白”，也成为“手撕激活锁”。
*   保资料解锁：和手机洗白有点类似，不同点是手机洗白是无所谓用户数据的，只需要让手机可以重新使用即可，同时最好可以绕过云端的激活锁逻辑以防止刷机之后再次被锁定。而保资料解锁的关键就是不能丢失用户资料，但是没有绕过云端激活锁逻辑的需求。这种技术会被用于公安取证，允许公安机关在犯罪嫌疑人不配合提供手机锁屏密码的情况下，通过保资料解锁实现提取手机中的数据资料。

#### 刷机解锁利用的技术

*   recovery 漏洞：OTA 升级流程中是 recovery 程序中写入升级包到各个正常系统分区完成升级，在数码圈俗称 “卡刷”（因早期 Android 手机内部存储很小，需把升级包放入外置 SD 卡中且无需连接电脑而得名），实际上除了直接解锁 bootloader 写入第三方 recovery 之外，官方的 recovery 程序也可能存在漏洞。例如最典型的就是升级包检查不严格，或处理升级包的过程中存在溢出问题等。
*   启动链漏洞：Android 启动链整体而言比传统 PC 要复杂，传统 PC 只有 UEFI 负责第一层引导，然后便直接加载 Windows 内核。Android 启动链以华为海思为例，在 Android 验证启动 2.0（AVB 2.0）之前还包含 BootROM、xloader、bootloader 这些阶段（通过这些阶段写入分区镜像在数码圈俗称 “线刷”，因为需要使用 USB 线连接电脑操作而得名）。实际上前面的组件是为了对标 iOS 的安全启动功能而实现，这些模块中引入的漏洞也会导致问题。高通的启动链黑灰产和安全研究人员主要集中于 EDL 模式的研究（9008 模式，俗称救砖），高通 EDL 对应于华为海思和苹果的 BootROM 部分，而 MTK 也有类似的模块也是黑灰产和安全研究人员研究的重点。
*   设备加密漏洞：主要用于实现保资料解锁，部分设备加密的密钥只通过锁屏密码生成，知道算法之后就可以直接进行离线爆破，更安全的设备会使用芯片密钥（相同芯片的机型共享一个密钥）和设备独立密钥（即一机一授权）再加上锁屏密码生成，这类设备就具有更高的安全性，必须窃取到被芯片保护的密钥才能完成保资料解锁。
*   业务逻辑漏洞：有些时候实现手机洗白的攻击只需要通过一些上层业务逻辑漏洞实现，比如在 Android 开箱体验（OOBE）程序中的逻辑漏洞，典型的有通过紧急呼叫、Talkback 实现逻辑绕过进入桌面。还有的是利用激活锁云端逻辑的漏洞进行攻击，例如锁屏密码解锁激活锁功能，这些都是黑灰产和安全研究人员喜欢研究的部分。

参考
--

*   [https://source.android.com/docs/security](https://source.android.com/docs/security)
*   [https://www.blackhat.com/docs/us-16/materials/us-16-Kralevich-The-Art-Of-Defense-How-Vulnerabilities-Help-Shape-Security-Features-And-Mitigations-In-Android.pdf](https://www.blackhat.com/docs/us-16/materials/us-16-Kralevich-The-Art-Of-Defense-How-Vulnerabilities-Help-Shape-Security-Features-And-Mitigations-In-Android.pdf)
*   [https://i.blackhat.com/EU-22/Wednesday-Briefings/EU-22-Ke-Android-Parcels-Introducing-Android-Safer-Parcel.pdf](https://i.blackhat.com/EU-22/Wednesday-Briefings/EU-22-Ke-Android-Parcels-Introducing-Android-Safer-Parcel.pdf)
*   [https://wrlus.com/android-security/android-app-protect/](https://wrlus.com/android-security/android-app-protect/)
*   [https://mp.weixin.qq.com/s/hGx8FNjI37trnZehY839bQ](https://mp.weixin.qq.com/s/hGx8FNjI37trnZehY839bQ)