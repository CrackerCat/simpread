> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.natsume324.top](https://blog.natsume324.top/archives/kirin-bootrom-usb-download-mode)

> 一. BootROM 简介 BootROM 指的是固化在芯片内部且一般不可修改的启动代码。

BootROM 指的是固化在芯片内部且一般不可修改的启动代码。它通常存储在不可擦写或仅可一次性编程的物理介质上。当系统上电或复位时，CPU 的程序计数器被硬件指向 BootROM 所在的地址，开始执行其中的指令。BootROM 负责执行设备的最早期启动操作，加载并执行下一阶段的引导程序（如 麒麟 xloader，高通 PBL）。

麒麟 BootROM 的主要特点：
-----------------

运行于 `LPMCU`（低功耗微控制单元）上，该单元基于 `Cortex-M3` 处理器。  
主要职责是加载 `xloader`（引导加载程序的第一阶段）；  
提供 **USB 下载模式** (USB Download Mode)-HUAWEI USB COM 1.0，用于**临时加载固件到内存中**。类似**高通 EDL**(Emergency Download Mode)-Qualcomm HS-USB QDLoader 9008  
并采用 `AES CTR` 加密和签名验证来保证加载固件的完整性。

图解麒麟芯片安全启动链加载过程：

![](https://blog.natsume324.top/upload/BootROM%20Chain-YibS.jpg)

启动流程：
-----

└ 按下电源键  
└ PMIC 触发  
└ =LPMCU= LPMCU 开始执行 BootROM 代码  
└ =LPMCU= BootROM 从闪存加载 xloader 或 加载 USB 下载模式  
└ =LPMCU= xloader 初始化 DDR 内存和主 CPU  
└ =LPMCU= `xloader` 加载 `fastboot`（对于 ≥990 型号还包括 `bl2`）  
└ =LPMCU= xloader 释放主 CPU 并运行 fastboot（或 bl2）代码  
└ =ACPU= fastboot 在 EL3 权限运行（<990 型号），加载多个固件（包括安全域）  
└ =ACPU= 最终加载内核并将控制权移交内核  
└ =ACPU= 安卓系统启动  
└ =ACPU= 内核初始化调制解调器加载（由 `teeos` 执行）

物理内存映射（近似范围）  
1. `0x00000000-0x00010000` **bootrom**  
2. `0x00022000-0x00050000` **xloader**  
3. `0x60000000-0x60010000` **uce**（依型号而定）  
4. `0x10000000-0x20000000` **DDR** 内存分区视图

华为针对 BootROM 漏洞通过 OTA 更新推送修复补丁，写入`efuse`熔断设备，以永久禁用 USB 下载模式。

此补丁在 EMUI11 高版本推送，受影响的设备为升级过鸿蒙的 Kirin980 及以后设备  
**熔断为硬件级，无法软件级修复**

![](https://blog.natsume324.top/upload/10k.jpg)  
注意，并非无法进入，而是进入了无法正常被主机设备识别连接，解决方法就是如上电路图接入一个 **10k** 电阻，即可正常被识别

具体接法如下:

![](https://blog.natsume324.top/upload/cable.jpg)

网上也有卖的，菊花工程线或工程头

并且由于已被熔断的设备硬件需要新驱动才可正常工作，如果降级到熔断前的旧系统，旧系统的驱动将无法正常工作，导致 **usb2.0** 和**华为快充**无法正常作用，充电功率仅 **2.5w**，但可用 pd 快充达到 **18w** 左右，如果设备支持 usb3.0 的话 usb3.0 不受影响

使用上面的方法接入 10k 电阻后也可以使 usb2.0 作用，但速度非常之慢，似乎比 usb1.0 还慢（

这部分驱动存储在内核当中，所以如果有带新驱动的对应版本的内核，刷入即可恢复 usb2.0 与快充，不同型号不一定通用，目前已知荣耀 20 系列有官方后编译的 EMUI9.1 系统和 Mate30 系列有后编译的 EMUI10.0 系统可恢复。

USB 下载模式由 BootROM 提供并加载，允许从 USB 端口加载下一阶段引导程序。

进入 USB 下载模式的两种方式：
-----------------

1. 软件触发：  
如果 xloader 分区损坏或签名验证失败，则 BootROM 自动进入 USB 下载模式  
可通过修改分区表或 xloader 分区来触发。  
例如通过 root 权限或通过解锁 FB Lock 的 Fastboot 擦除`xloader`分区  
抑或通过`HUAWEI Upgrade Mode`(华为更新模式)，写入修改过的分区表或 xloader

2. Test Point 触发：  
也就是常说的短接  
物理 Test Point 是 PCB 上的一个 GPIO 引脚，Kirin 设备为单个引脚, 对地短接 (如接到屏蔽罩) 即可进入 USB 下载模式。如设备已被熔断，请接入 10k 电阻

进入后将在设备管理器的端口处出现 HUAWEI USB COM 1.0

![](https://blog.natsume324.top/upload/1.0.jpg)

如果显示 USB SER 则需要打上驱动

![](https://blog.natsume324.top/upload/ser.jpg)

驱动链接：[1.0 驱动. exe](https://blog.natsume324.top/upload/1.0%E9%A9%B1%E5%8A%A8.exe)  
自行下载即可

USB 下载模式通信协议：
-------------

USB 下载模式 采用 `Xmodem` 协议进行数据传输，Xmodem 协议实现了一个状态机，处理四种不同类型的数据包，称为块 (`chunk`)，并通过一个单字节的结果进行响应（0xAA：ACK 相应，0x55：NAK 响应，0x07：地址 / 大小错误）

  
以下为各块简介：

**头块 (Head chunk)**：包含加载镜像的地址和大小等。  
**数据块 (Data chunk)**：包含需要加载的实际镜像数据段，每个段最大 1024 字节，并且带有一个序列号计数器。  
**尾块 (Tail chunk)**：用于结束传输数据块并继续验证镜像的签名。  
**查询块 (Inquiry chunk)**：用于向引导加载程序（bootloader）查询状态值。

各块规范均采用 16 进制：  
头块 (Head chunk)：指令 (0xFE) | 序列号 (0x00) | 按位取反序列号 (0xFF) | 文件类型 (0x01/0x02) | 文件长度 (通常为 uint 4 字节无符号 32 位整数) | 加载地址 (通常为 uint 4 字节无符号 32 位整数) | CRC 校验和 (通常为 ushort 2 字节无符号 16 位整数)  
例：FE 00 FF 01 00 02 BB 2C 00 02 20 00 0B 54

数据块 (Data chunk)：指令 (0xDA) | 序列号按位与 0xFF | 按位取反序列号按位与 0xFF | 数据 (最大 1024 字节) | CRC 校验和 (通常为 ushort 2 字节无符号 16 位整数)  
其中 1024 字节数据即要发送的文件镜像数据段，将文件拆分成多个 1024 字节的数据块依次发送  
例：DA 01 FE + 数据... +CRC 校验和

尾块 (Tail chunk)：指令 (0xED) | 序列号按位与 0xFF | 按位取反序列号按位与 0xFF | CRC 校验和 (通常为 ushort 2 字节无符号 16 位整数)  
在所有数据块发送完成后发送  
例：最后一个数据块为 DA 24 DB 00  
则尾块为 ED 25 DA 71

  
查询块 (Inquiry chunk)：指令 (0xCD) | 序列号按位与 0xFF | 按位取反序列号按位与 0xFF | CRC 校验和 (通常为 ushort 2 字节无符号 16 位整数)  

通过反编译 xloader 镜像可知对查询块的处理代码如下：

```
  if ( v11 == 205 )
  {
    a1[636] -= v4;
    v12 = sub_2FC1C("[USBI]inquire frame\n");
    *((_BYTE *)a1 + 984) = sub_3BB14(v12);
    sub_314F8(a1, v5, 1);
    sub_2FC1C("[USBI]response default\n");
    return 0;
  }

```

205 为十进制数字，转换为 16 进制正好为 CD，当 xloader 接受到查询块时默认返回 1 字节响应  
可通过漏洞修改此部分代码然后使用此块将设备内部解密后数据返回至主机

**  
"按位取反" 和 "按位与" 涉及位运算，不懂自行了解**

  
USB 下载模式在加载 xloader 后便启动到了 xloader，后续镜像的加载由 xloader 接管，加载 fastboot，bl2(≥990) 后将启动到 fastboot

所以 USB 下载模式并没有像高通 EDL 一样有真正意义上的刷写功能，**实际刷写功能在 fastboot 完成**。  
USB 下载模式加载的镜像都是临时加载到内存的，不会写入到闪存；但在 fastboot 中刷写分区是会刷入到闪存中的。  

  
另外，华为更新固件. APP 文件中并不包含 uce 分区，如图

![](https://blog.natsume324.top/upload/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-06-06%20133154.png)

xloader 过后并不是 uce 分区，为什么呢？这是因为在`华为更新固件.APP文件`中将 xloader 和 uce 分区合并到了一起

通过 16 进制查看提取出的 xloader 分区镜像，可发现在末尾还包含一个头部证书，这就是 uce 的头部证书，如图

![](https://blog.natsume324.top/upload/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-06-06%20133511.png)

于是将末尾的 uce 部分剪切到新文件，就是 uce 镜像了，再将. APP 固件中的 fastboot 镜像提取，便组成了可以在 USB 下载模式加载的三个完整镜像了

注意：.APP 更新固件内的 fastboot 并未写入`FB Lock`解锁状态，如果直接在 USB 下载模式加载后进入的 fastboot，FB Lock 是锁定的，而工程固件内的 bootloaderimage 文件夹中也包含用于 USB 下载模式加载的几个完整镜像，同 cpu 的 xloader，uce，fastboot，bl2 可以通用。并且华为早期工程固件内的 fastboot 的状态是写入`FB Lock`解锁状态的，所以加载后 BL 状态就已经是`Unlocked`了  
自麒麟 980 开始，工程固件内的 fastboot 也是非解锁状态，需要手动解锁，具体下篇再讲。

在 Github 上有一些开源项目可用于向 USB 下载模式通信发送镜像  
[https://github.com/penn5/hisi-idt](https://github.com/penn5/hisi-idt)

![](https://blog.natsume324.top/upload/example.png)

如图对 Kirin970 设备操作，文件来自工程固件

auto 将自动检测端口号  
只提供端口号则进入交互模式连续发送镜像

此文内容均来自于 TASZK 实验室的文章和 checkm30 文档及个人研究理解  
请勿用于非法行为，所有非法行为与本人无关  
本文同步于酷安和个人博客  
下期更新华为引导加载程序原理