> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268572.htm)

> [原创][PwnMonkey] 云丁鹿客门锁 BLE 通信的分析

本文内容较长，所以将目录整理在最前面，用于方便各位读者查阅，目录如下：

1.  简介
2.  Apk 分析  
    2.1 开锁流程分析  
    2.2 Challenge 的处理  
    2.3 app 开锁流程小结
3.  固件分析  
    3.1 调试接口与调试器  
    3.2 通过调试接口提取固件  
    3.3 通过调试接口调试固件
4.  bootloader 和 FreeRTOS 的分析  
    4.1 bootloader 分析  
    4.2 FreeRTOS 分析
5.  小结

下面就开始正文：

1. 简介
=====

本篇我们与各位读者分享一些关于 Loock Touch 智能门锁的研究内容，该门锁由云丁科技公司出品，云丁科技是一家专注于研发和生产智能家居安全产品的公司，旗下有两大品牌，分别是针对家用市场的 “鹿客” 和针对公寓市场的“云丁”。Loock Touch 是鹿客品牌的主打产品之一，其架构与云丁品牌的 D2、D3 智能门锁很相似，最新的鹿客旗舰产品是 Loock Touch 2 Pro，其硬件架构与此前所有产品均不同，变化较大。  
由于胖猴实验室一直和云丁科技有良好的合作关系，所以接下来的几篇的分析和讨论中，我们只对分析方法和研究过程做讨论，而不涉及任何有关漏洞的细节，这与此前的关于海康萤石智能网关的文章类似。

2.App 分析
========

2.1 开锁流程分析
----------

在之前的文章中，我们分析了果加智能门锁，链接为 [https://bbs.pediy.com/thread-259530.htm](https://bbs.pediy.com/thread-259530.htm)，在本次分析中我们依旧以 BLE 开锁为分析切入点。云丁鹿客的 Loock Touch 门锁同样可以通过手机 BLE 直接开锁，那么我们就从 app 的开锁流程开始分析。  
应用市场中下载到的云丁鹿客智能门锁的配套 app 加了梆梆企业版的壳，脱壳之后可以看到该 app 打印出来的日志（脱壳不在本系列文章的讨论范畴中，分析 iOS 版也是可以的）。我们操作手机开启门锁后，app 的日志中有这样几条记录吸引到了我们。  
![](https://bbs.pediy.com/upload/attach/202107/613694_AKT98JAUWHGMG2V.jpg)  
图 2-1 app 开锁时打印的日志  
根据上图中 3 个红框中的记录内容，我们可以推测开锁过程是基于 Challenge/Response 模式进行认证的，具体过程如下：  
（1）app 与门锁建立 BLE 连接后，当使用 app 开启门锁时，app 会先下发一条通知指令，告知门锁准备进入开锁流程；  
（2）随后门锁会向手机发送一组数字，称之为 Challenge；  
（3）app 对 Challenge 进行处理生成 Response 并发送给门锁，门锁验证 Response 正确时，就会开锁。  
我们对比了多次开锁时的 app 日志，发现每次开锁时第（1）步发送的通知指令仅有一个字节内容不同，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_9C3XBVTJV5J6UTP.jpg)  
图 2-2 通知指令对比  
上图中，我们对比了 3 条通知指令，发现仅有第 7 个字节不同，在 app 封装数据包的代码段中，可以确定这个字节的含义是 seqID，即数据包的序列号。因此我们可以推测这一条通知指令的作用类似于 TCP 协议中的 SYN 数据包，仅起到握手的作用，而门锁对手机的认证过程，主要是第（2）和第（3）步，那么接下来我们就分析一下 app 是如何处理 Challenge 并生成 Response 的。

2.2 Challenge 的处理
-----------------

### 2.2.1 Challenge 处理流程分析

通过在代码中搜索日志内容，我们可以确定，app 收到 Challenge 数据后，会由下图中的方法进行处理。  
![](https://bbs.pediy.com/upload/attach/202107/613694_RWSUAM2C6SGK3BJ.jpg)  
图 2-3 Challenge 数据处理  
上图中， handleCommand 方法用于处理 BLE 返回数据，门锁返回的 Challenge 就作为参数传给了方法 sendUnlockCommandNewProtocol，从参数和方法名称来看，这就是处理 Challenge 数据、生成 Response 并发送回门锁的方法。  
进一步查看代码，我们发现在 sendUnlockCommandNewProtocol 方法中，最终调用了下图中的 encryptBleKeyNewProtocol 方法对 Challenge 数据进行处理，该方法返回后的数据，添加包头后会被直接发送给门锁。  
![](https://bbs.pediy.com/upload/attach/202107/613694_36CDQFK7ZRWHJXV.jpg)  
图 2-4 对 Challenge 数据进行加密  
上图中，参数 arg13 即为 Challenge 数据，可以看到该方法中使用了 AES EBC 加密，密钥是参数 arg9，加密完成后使用密文又异或了一组固定数据。  
经过以上分析我们可以推测，开锁流程中，门锁对手机的认证依赖于图 2-3 中的 AES EBC 算法，即双方使用相同的密钥对 Challenge 进行加密和解密处理，仅当手机拥有正确的 AES 密钥才能通过认证开启门锁。那么开锁流程的安全性，就取决于门锁与手机使用的密钥是否安全了。

### 2.2.2 Challenge 加密密钥的获取

图 2-3 中，加密密钥是参数 arg9，通过对 arg9 的回溯可以确定，对 Challenge 进行加密的密钥通过 resetBleToken 方法获取，如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_FTFSVJSBPKXW6DS.jpg)  
图 2-5 加密密钥的获取  
resetBleToken 方法中发出了一个 http 请求，收到返回的 BleToken 并进行处理后，会对一些变量进行赋值。图中的 mBleKey 是 BleKeyInfo 类型的变量，其 token 成员变量，就是 2.2.1 章节中 Challenge 数据的加密密钥。下文中，我们将以 BleKey 来代称这个加密密钥。  
由以上分析可以知道，BleKey 是 app 从服务器上请求到的，这里就有两种可能：（1）每次 app 与门锁建立通信时，都会请求一个 Key 并下发给门锁；（2）当 app 与门锁建立通信时，如果没有可用的 BleKey，才会向服务器发送请求。  
为了验证以上推测，我们使用手机多次重复开启门锁，这样使手机和门锁反复建立通信，这时我们在 app 日志和抓取到的数据包中，都没有发现这个请求以及相关内容，可以判断 app 应该是本地保存了 BleKey，而不需要从服务器获得了。  
我们可以通过清空 app 缓存的方式，删除本地存储的 BleKey，删除后 app 开锁时的日志发生了一些变化，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_G8FND4RETCT5HSV.jpg)  
图 2-6 删除 app 的蓝牙钥匙后 app 开启门锁时的日志  
对比图 2-6 和图 2-1 中的日志内容，可以看到删除 BleKey 后，在开启门锁时先执行了 resetBleToken，这和我们的分析吻合。这里我们注意到，resetBleToken 之后，还有另外一条日志 sendBleKeyCommandNewProtocol，这里是向门锁发送 BleKey 的方法，下面我们先继续手机获取 BleKey 这一过程的分析，手机向门锁下发 BleKey 的过程将在 2.2.4 节中分析。  
与此同时，我们在 fiddler 中也抓到的对应的 http 请求，如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_A6SBEE9K42XP35J.jpg)  
图 2-7 reset_token 请求  
app 收到返回的 BleToken 后，会使用 AES CBC 算法，对上图中红框内的 secret 和 token 进行解密，相关代码如下图所示:  
![](https://bbs.pediy.com/upload/attach/202107/613694_GZV74GWUC7U7QCD.png)  
图 2-8 reset_token 返回值的处理  
此时可以确定，app 从服务器获取到的 BleKey 同样被 AES CBC 加密算法保存。通过图 2-8 可以看到，其密钥由方法 getCryptSecret 返回，与方法名称对应，这个密钥我们就称之为 CryptSecret，下一步我们就来分析 CryptSecret 的来源。

### 2.2.3 CryptSecret 的获取

BleKey 解密时的密钥通过 getCryptSecret 方法获得。那么我们可以从这个方法入手，相关代码的搜索结果如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_BZQQQN8UBYJMGW8.jpg)  
图 2-9 获取 CryptSecret 的代码流程  
上图中，getCryptSecret 方法返回的是 this.mCryptSecret，而这个变量是通过 setCryptSecret 赋值的，setCryptSecret 方法则是在 getSecret 方法中被调用。在 getSecret 方法中，可以看到 CryptSecret 是从云丁鹿客的服务器获取到的，其 url 为”api/v1/crypt_secret”。同样的，我们在 fiddler 里也可以抓到 crypt_secret 数据包，如下图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_3NGGFDYWEV9CR5N.jpg)  
图 2-10 crypt_secret 请求及其响应  
由于这里的信息有些敏感，所以我们做了打码处理。

### 2.2.4 向门锁下发 BleKey

2.2.2 节中说到 sendBleKeyCommandNewProtocol 函数负责向门锁下发 BleKey，该函数如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_ZP9M44S8WP8EF97.jpg)  
图 2-11 手机向门锁下发 BleKey 的函数  
结合上图与图 2-6 的日志和图 2-7 中 reset_token 请求的返回数据包，可以发现，手机向门锁下发 BleKey 的流程非常简单，就是对 reset_token 返回数据包中的 totalData 进行 base64 解码，解码后的二进制数据直接通过 BLE 通信发送给了门锁，门锁对 totalData 的处理我们将在后续文章中分析。

2.3 app 开锁流程小结
--------------

经过第二章的分析，我们可以将 Loock Touch 门锁的开锁流程总结如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_P89UA78BJ3ZJMC3.jpg)  
图 2-12 开锁流程图  
上图中，开锁流程可以分为两个部分：  
（1）虚线部分表示，当 app 没有存储可用的 BleKey 时，会向服务器请求 CryptKey 和 BleToken 两组数据，以 CryptSecret 为密钥，对 BleToken 中的数据进行 AES CBC 解密后，就可以得到 BleKey。  
（2）实线部分则是当 app 中有可用的 BleKey 时，会以 BleKey 为密钥对 Challenge 进行 AES EBC 加密，密文异或一组固定数据后，作为 Response 的主要内容并发送给门锁，当 Response 校验成功后才会开启门锁。

3. 固件分析
=======

上文中，我们着重关注的是手机端是如何处理和发送 BLE 消息的，那么接下来就看看门锁这边针对 BLE 消息的处理流程。

3.1 调试接口与调试器
------------

在此前的文章中，我们已经介绍了通过 gdbserver 调试海康萤石固件程序的方法，链接为 [https://bbs.pediy.com/thread-261679.htm](https://bbs.pediy.com/thread-261679.htm)，通常情况下，gdbserver 是运行于 Linux 系统环境中的调试器程序，通过使用 Linux 系统提供的调试接口以完成调试工作，并不需要太多硬件上的辅助。而在某些场景中，例如，没有 Linux 操作系统时，gdbserver 等工具就无法满足我们的调试需求了，此时需要直接使用 MCU 提供的硬件调试接口和功能，以完成调试工作。通过阅读 MCU 的芯片手册，可以找到 MCU 提供的调试方式、调试引脚等详细信息。  
为了使用 MCU 的调试接口，我们还需要与之对应的硬件调试器。与 IDA、GDB 等调试器软件类似，通过硬件调试器，我们也可以完全掌握被调试程序的执行状况。一般情况下，硬件调试器是通过 USB 接口连接到 PC 主机的，而硬件调试器与 MCU 之间的通信接口则有许多种。不同的 MCU，支持的接口也不尽相同，具体信息需要查阅各芯片的手册。  
本篇文章要研究的 Loock Touch 门锁使用的 MCU 型号为 EFM32GG280F512，通过翻阅芯片手册，可以找到其调试接口信息如下图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_QKSJKJVKSA8VUSD.png)  
图 3-1 EFM32 调试接口  
上图中的调试接口名称为 SWD 接口，常见的使用 SWD 接口的硬件调试器有 JLink、STLink 等。  
我们目前使用的调试器 SEGGER JLink（下文简称为 JLink）提供了对 JTAG 和 SWD 两种调试接口的支持，具体使用时，只需要连接到对应的引脚即可，JLink 中两种接口的引脚如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_WGTRSJ42RUSW6BX.jpg)  
图 3-2 SEGGER JLink 的引脚定义

3.2 通过调试接口提取固件
--------------

JLink 调试器可以通过调试接口读写芯片内置的 Flash，从而获取完整的固件。并不是所有芯片都可以通过调试接口提取固件，很多设备在发行版中启用了读保护（Readout Protection）机制，使得我们无法通过调试接口读到 Flash 内容，所幸本文中的设备并没有启用该机制。  
接下来，我们就寻找并尝试用 JLink 连接芯片的调试接口。

### 3.2.1 连接调试接口

在门锁的电路板上，主控 MCU 一旁有一排过孔，我们已经在将排针焊接到这排过孔上了，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_YE7VZFZK647G2ZC.jpg)  
图 3-3 MCU 及其部分接口引脚  
图中，左侧红框内就是我们焊上的排针，电路板上标注了排针对应的 MCU 引脚名称。经过万用表测量，可以确定红框中的 IO 和 CLK 引脚就是芯片的 SWDIO 和 SWCLK 两个引脚，我们在图 2-1 中提到这两个引脚是 SWD 调试接口的一部分，那么我们将这几个引脚按照如下图的方式连接到 JLink 调试器。  
![](https://bbs.pediy.com/upload/attach/202107/613694_DUFPV2HKZ4KP5UW.jpg)  
图 3-4 调试接口与 JLink 的连接  
上图左侧是图 3-1 中的排针，右侧是 SEGGER JLink 的引脚。连接完成后，将 JLink 连接到电脑的 USB 口，并打开 JLink Commander 命令行工具（JLink 的相关工具和使用手册等都可以在 SEGGER 的官网下载到：https://www.segger.com/downloads/jlink/），执行下图中的命令：  
![](https://bbs.pediy.com/upload/attach/202107/613694_RQYYUFSH9CZPFBX.png)  
图 3-5 使用 JLink 连接 SWD 接口  
下面逐一解释上图中 6 个红框的内容：  
（1）命令行工具开启时，会首先对 JLink 的状态进行检测（SEGGER 官网的 JLink，无论价格还是到货时间都不理想，所以我们选择了某宝，从这里打印的 JLink 固件信息来看，某宝的似乎是盗版产品）；  
（2）输入 connect 指令，控制 JLink 开始通过调试接口连接 MCU；  
（3）输入？指令选择要调试的芯片型号，在弹出的对话框中，我们按照芯片的厂商和型号，选择如下图的选项；  
![](https://bbs.pediy.com/upload/attach/202107/613694_3FNGBA7PNMSKDXT.jpg)  
图 3-6 芯片型号选择  
（4）输入 s 指令，选择调试接口的类型为 SWD；  
（5）选择接口的通信速率，一般保持默认即可；  
（6）当出现第 6 个红框中的内容时，说明 JLink 已经通过 SWD 接口连接到了 MCU，接下来我们就可以读取固件或者进行调试了。

### 3.2.2 提取固件

上文提到，调试器可以读写芯片的内置 Flash，而固件代码就存储在芯片内置 Flash 中，所以我们只需要将 Flash 中代码区域的数据读出并保存下来，就相当于拿到了固件。按照我们分析果加门锁时的思路，翻阅芯片手册，可以确定代码存储在内存地址为 0x0~0x100000 的区域。我们通过 JLink Commander 命令行工具的 savebin 命令（该命令的使用方法可以去 JLink 的手册中查阅）将这一区域中的二进制数据保存下来。  
![](https://bbs.pediy.com/upload/attach/202107/613694_5936XDSV4A2FPZY.jpg)  
图 3-7 读取固件的二进制数据  
上图中，我们拿到了固件代码，并将其命名为 LoockEfm32Fw.bin。

### 3.2.3 解析固件

在拿到固件之后，我们需要解析固件并对其进行逆向分析。通过查阅手册可以知道，EFM32GG280 的核心是 ARM Cortex-M3，和果加门锁用到的 STM32L0 同属 Cortex-M 系列，所以我们可以按照果加门锁相关篇章中的方法配置 IDA 并载入固件。载入之后，继续沿用果加门锁分析时的思路，寻找 Reset 中断的中断服务程序，即偏移 0x4 地址处的数据，跳转之后按 “c” 键，IDA 就会自动开始解析代码，如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_HJ2NNTSUGRV2SCN.jpg)  
图 3-8 从 Reset 中断入手解析代码  
上文提到，代码存储在内存地址为 0x0~0x100000 的区域，所以图 3-6 中固件的基址为 0x0。此前，我们在分析果加门锁时，将固件载入 IDA 时基址是 0x08000000，因为在果加的案例中，代码存储区域的起始地址是 0x08000000。至此，我们成功提取并解析了 Loock Touch 门锁的固件。

3.3 通过调试接口调试固件
--------------

### 3.3.1 SEGGER JLink 的调试功能

在进行调试之前，先简单介绍一下如何使用 SEGGER JLink（下文以 JLink 代称）的调试功能。上一篇文章中，我们在提取固件时使用了 JLink 的命令行工具，这个命令行工具也包含了诸如设置断点、暂停 CPU 核心、读取 / 写入寄存器或指定位置的内存、单步调试等动态调试所需要的指令，具体可以查阅 SEGGER 的 Wiki（https://wiki.segger.com/J-Link_Commander）。可以选择 openOCD 配合 JLink 进行调试，但我们在本篇中暂不介绍 openOCD，而是选择另一款调试工具。  
在实际调试过程中，命令行工具通常需要搭配 IDA 使用，用起来非常繁琐，且信息展示不够直观。SEGGER 提供了一个图形界面工具——Ozone，可以在一定程度上解决以上问题，关于 Ozone 的介绍可以参考 SEGGER 的官方网站（https://www.segger.com/products/development-tools/ozone-j-link-debugger/），软件以及使用手册的下载页面也可以在这里找到。下载后的安装一路 next 就可以了，想必大家都很熟练（滑稽）。  
安装完成后，可以通过 “File->New->New Project Wizard” 选项来创建新的项目，创建时需要选择待调试芯片的型号、调试接口类型、通信速率等信息。继续，点击 “Debug->Start Debug Session->Attach to Running Program” 连接到待调试设备，注意整个调试过程中需要保持电脑、调试器和设备的接通状态，如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_DU5ZRYF827KZ8V2.jpg)  
图 3-9 连接到设备  
连接到待调试设备后，Ozone 可以像 IDA 一样展示多个 subview，这样就可以同步观察很多信息了，下图是笔者某次调试的界面。  
![](https://bbs.pediy.com/upload/attach/202107/613694_E6MD5HJPXP8KMKW.jpg)  
图 3-10 Ozone 调试界面  
如上图所示，我们在调试时可以一次性看到寄存器的数据、多个内存区域的数据以及反汇编后的代码，下方的控制台区域可以执行一些指令或预先编写好的脚本，除此之外，左上角的断点设置区域，可以看到断点除了 Location 以外，还有 Type 和 Extra 两个属性，通过阅读用户手册可以确定这两个属性是用于设置断点类型的，如执行断点，读写断点、TRACE 断点等，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_89TQDKKS4BBARTV.jpg)  
图 3-11 Ozone 中断点的属性  
Ozone 还提供了很多强大的功能，在后文中，我们用到的时候就会逐一介绍这些功能。

### 3.3.2 通过 SWD 接口对门锁进行调试

翻阅门锁 MCU 的芯片手册可以看到，芯片提供了 AES 处理模块，该模块的内存映射如下图所示。  
![](https://bbs.pediy.com/upload/attach/202107/613694_5G65HJ776HDDE3N.jpg)  
图 3-12 EFM32 中的内存映射  
显然，我们假设鹿客门锁使用芯片的 AES 模块进行加解密操作，而不是自写 AES 算法，那么必然需要访问 0x400E0000~0x400E0400 这片区域中的内存地址，那么我们只要在设置适当的读写断点，然后等待手机与门锁进行通信时触发断点就可以了。  
在断点设置区域右键，选择 Set Data Breakpoint，会弹出如下图所示的窗口:  
![](https://bbs.pediy.com/upload/attach/202107/613694_TGPFNPVMXYGY8PK.jpg)  
图 3-13 设置数据端点  
上图中的设置，表示当 CPU 向 0x400E00XX 地址写入数据时触发断点。  
断点设置完成之后，在手机上点击开锁，由于我们设置的数据断点是监控一片内存区域，所以开锁过程中会多次被触发，后可以看到如下图左侧的一小段代码，右侧是执行到 0xED0C 地址时的寄存器数据。  
![](https://bbs.pediy.com/upload/attach/202107/613694_6XD4E98DA2DZ57Y.jpg)  
图 3-14 写入待解密数据的断点  
上图中，0x400E001C 地址（R0 + 28）是 AES_DATA 寄存器，这一小段代码所处的函数向 AES_DATA 寄存器写数据，应该就是 AES 的处理函数（下文以 AESFunc 代称）。回溯调用栈，可以找到调用 AESFunc 的外层函数，以及 AESFunc 函数的起始地址，进而使用 IDA 的 F5 功能对 AESFunc 函数进行分析，如下图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_BJMBPHRCQ9X93AK.jpg)  
图 3-15 AESFunc 函数的伪代码  
结合芯片手册，由伪代码很容易能判断出 AESFunc 各个参数的作用，在本系列第一篇文章中，我们已经知道了开锁时的解密密钥就是 BleKey，而门锁 BleKey 的获取过程，是由服务器下发一组数据 totalData 至手机 app，app 没有进行任何处理，直接通过 BLE 通信转发给了门锁。那么，接下来看一看门锁是如何处理 totalData 的。  
首先我们需要看一下，totalData 的内容是什么，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_Q8DGD6J9ZURVK68.jpg)  
图 3-16 totalData 字段及其 base64 解码数据  
可以看到 totalData 是一串二进制数据经过 base64 编码后的结果，红框之前的部分可以视作数据的 header，包含消息头、数据包序号、校验等内容， header 之后的 body 部分，即红、蓝、黑框中的数据，这三组数据的结构是相同的，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_TZR45XSAM4XEVW8.jpg)  
图 3-17 totalData 中 payload 用到的数据结构  
红框中 data_type 为 0x03 的数据，其 data_content=0x5FFFECAE，该部分与 AES 根密钥密文有关。totalData 的 body 部分还有两组数据，其具体作用不再详细说明，说太多有些不妥。  
接着，我们在 AESFunc 的入口下一个断点，可以看到 BleKey 的值如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_E3J5JPE3NYVVQ76.jpg)  
图 3-18 内存中的 BleKey  
从图 3-16 和图 3-18 中看不到 BleKey 和 totalData 之间的联系，想必 totalData 和 BleKey 之间还是经过了某些解密或解码转换，我们需要继续寻找门锁对 totalData 的处理过程。保持 AESFunc 函数的断点，在调试过程中可以发现 totalData 也是经过 AESFunc 函数进行解密的。此时，回溯调用栈即可找到如下图所示的关键代码。  
![](https://bbs.pediy.com/upload/attach/202107/613694_VRQEE8FTCSTM8YX.jpg)  
图 3-19 totalData 数据的处理  
上图中的 AESEntry 函数为 AESFunc 的封装，这部分就是由 totalData 生成 BleKey 的核心代码，该流程可以整理为下图。  
![](https://bbs.pediy.com/upload/attach/202107/613694_2E4P6W47DZBDYZA.jpg)  
图 3-20 BleKey 获取流程图  
在这个流程中，如果没有根密钥就无法解密 BleKey，通过调试可以找到根密钥在 Flash 中的存储地址是 0x7E09C。

4. bootloader 和 FreeRTOS 的分析
============================

在第三章中，我们直接用 SWD 调试器定位到了关键代码，这种分析方法固然好，但却跳过了很多有价值的地方。在第四章中，我们回过头来从另一个角度分析这个设备，即从设备上电开始的第一条代码开始分析，一点一点理解整个固件的内容。

4.1 bootloader 分析
-----------------

bootloader 是设备上电之后首先运行的一段程序，它会完成必要的初始化操作，并引导操作系统的加载。在我们拿到设备固件，并顺利通过 IDA 载入固件之后，可以看到解析出的内容仅是完整固件的一小部分，而更多的固件内容还是未解析状态。这被解析出来的一小部分，其实就是设备的 bootloader 部分代码。  
为帮助我们分析这个云丁鹿客的智能门锁，我们可以尝试接通设备的串口。串口的位置就在 SWD 接口的旁边，可以参考云丁鹿客智能门锁系列第二篇中的图 3-1。此时，给设备上电，是可以看到串口有字符串输出的，如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_JXP2MWZYMAKQ2P2.jpg)  
图 4-1 启动时的串口输出  
上图中，前几条字符串是可以直接在 IDA 中搜索到引用的，而剩下的字符串则未找到引用地址，如下图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_2DNBJTBGYXRTGXY.jpg)  
图 4-2 字符串引用  
![](https://bbs.pediy.com/upload/attach/202107/613694_RKU6SS8FVCGDWJE.jpg)  
图 4-3 字符串未引用  
造成上图中现象的原因是，IDA 仅仅解析了 bootloader 部分代码，这些字符串的引用代码刚好是 bootloader 部分输出的；其余字符串是由主程序输出的，IDA 未能解析出主程序而导致这些字符串在代码中未被引用。上图中，程序最终进入 sub_C44 函数，如下图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_52EAXMERED5FEHZ.jpg)  
图 4-4 sub_C44 函数  
上图函数的功能很简单，即将 0x4100 存储到向量表偏移寄存器（0xE000ED08 地址），也就是将 0x4100 地址的数据设置为新的中断向量表，随后调用 sub_F0 函数。  
![](https://bbs.pediy.com/upload/attach/202107/613694_5JHQXVTWKDZN4VN.png)  
图 4-5 sub_F0 函数  
结合图 4-3 和图 4-4，bootloader 最后会把 0x2001B860（0x4100 地址处的值）设置为新的栈指针，并跳转到 0x49ED（0x4104 地址处的值）。从这之后 bootloader 就将 MCU 的控制权交给了主程序。  
综上，设备上电后会首先执行 bootloader，bootloader 会在 Flash 中搜索、校验固件，校验完成后，会设置新的中断向量表并跳转至主程序的入口地址。

4.2 FreeRTOS 分析
---------------

通过固件中的字符串可以推测出设备使用了 FreeRTOS 操作系统，该系统是一种很常见的开源实时操作系统，其官方主页是：https://www.freertos.org。在使用 FreeRTOS 系统的设备中，固件主程序是 FreeRTOS 系统内核代码和应用逻辑代码相互杂糅在一起的一个二进制文件，所以我们需要先设法将二者区分开来，否则就容易出现山总分析 MFC42 一样的错误。  
FreeRTOS 操作系统提供了任务管理的功能，而设备的逻辑功能就是由其中一个或多个任务（task）来实现的，所以应用逻辑代码必然在这些 task 的实现代码中。FreeRTOS 系统创建任务的 API 如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_9Z8AC3V7DHHC3ZC.png)  
图 4-6 FreeRTOS 任务创建 API  
上图中，函数的第一个参数是该任务对应的函数指针，第二个参数是任务的名称，只要找到这个 API 的调用处，通过它的第一个参数就可以定位任务函数了。  
由于 FreeRTOS 是开源的操作，我们可以使用源码编译生成一个带符号表的程序，进而利用 IDA 的 bindiff 插件对比识别固件中的 FreeRTOS 操作系统 API。此处的 bindiff 插件是一款 IDA 插件，用于比较两个 idb (IDA database) 文件中的相似函数，常见于漏洞的补丁分析，其官方网址是 https://www.zynamics.com/bindiff.html。按照官网的安装说明，下载相应版本的 bindiff 安装程序，完成安装即可使用。  
为了提高 bindiff 插件对比的准确率，我们需要尽量选择与云丁鹿客近似的 FreeRTOS 版本，编译环境和配置也尽量靠近云丁鹿客的智能门锁。有趣的是，在我们搜索可用的源码过程中，意外地发现 github 上有一份代码与我们正在逆向分析的固件极其相似，链接是 https://github.com/RunningChild/efm32_freertos_app，后发现该仓库的拥有者就是云丁鹿客的工作人员，这份代码是把关键应用逻辑代码全部删掉之后剩下的 FreeRTOS 框架。  
顺利编译该项目之后即可使用 bindiff 插件做比较，具体编译方法和过程会在以后的文章中专门做整理和介绍，本篇番外中暂不做过多介绍，部分对比结果如下图所示：  
![](https://bbs.pediy.com/upload/attach/202107/613694_3PSTE4ZTNQVUFYE.jpg)  
图 4-7 bdiff 部分结果  
上图中我们编译的固件中 xTaskGenericCreate 函数和原版固件中 sub_34A50 函数相似度非常高，可以判断 sub_34A50 函数就是 FreeRTOS 中创建任务的 API 接口，那么我们在 IDA 中查找 sub_34A50 函数的引用，某一处引用如下图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_UN2TWAYF4KBBB4G.jpg)  
图 4-8 sub_34A50 的调用  
上图中，sub_34A50 函数的参数和图 4-6 中 xTaskCreate 的参数是吻合的；还可以判断出位于 0x1B0A9 地址处的函数负责的应该是 BLE 消息的接收和处理。在 0x1B0A8 地址处下一个断点开始调试，当手机 app 里点击开门按钮后就会触发断点，可以沿此线索继续往后分析。  
由此我们可以确定 sub_34A50 函数就是固件用来创建任务的 API，接下来只需要写一个简单的 IDApython 脚本，即可将固件中的所有任务都整理出来，并使 IDA 以这些任务的函数地址创建函数，脚本如图：  
![](https://bbs.pediy.com/upload/attach/202107/613694_2JQNXYJH7DM6AQR.jpg)  
图 4-9 IDA 解析任务入口函数的脚本  
上图中的脚本运行结果如下：  
![](https://bbs.pediy.com/upload/attach/202107/613694_RUKPS85ADP6RAMY.jpg)  
图 4-10 脚本的结果输出  
注意，脚本功能依靠 sub_34A50 函数的交叉引用来实现，存在一些未被 IDA 识别的引用点，因此可能会漏掉一些 task。至此，我们就可以随意分析云丁鹿客智能门锁中我们感兴趣的功能点了。

5. 小结
=====

到这里，关于云丁鹿客智能门锁的分析也结束了。在这篇文章中，我们通过逆向手机 apk 了解门锁蓝牙功能的通信数据中的 AES 加密保护；然后分享了如何使用 SWD 接口提取固件并进一步通过对门锁固件的调试，来分析门锁获取 BleKey 的过程；最后，又着重分析了设备启动后的 bootloader 代码以及 FreeRTOS 操作系统。作为一款开源的实时操作系统，FreeRTOS 着实拥有不少的用户量，虽然分析起来要比嵌入式 Linux 操作系统的程序要麻烦一些，但仍然建议各位读者对其有一点简单的认识。希望本篇文章能给各位读者带来一些收获，如若有什么想商量或者讨论的，可以随时联系我们胖猴微信：PwnMonkey。在后续文章中，我们还会继续分享其他的研究案例，敬请期待。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 21 小时前 被胡一米编辑 ，原因：

[#安全研究](forum-128-1-167.htm) [#家用设备](forum-128-1-173.htm)