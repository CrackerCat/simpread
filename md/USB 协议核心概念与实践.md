> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/ipQD4PWP6EPydmxD6vWbMA)

USB，全称是 Universal Serial Bus[1]，即通用串行总线，既是一个针对电缆和连接器的工业标准，也指代其中使用的连接协议。本文不会过多介绍标准中的细节，而是从软件工程师的角度出发，介绍一些重要的基本概念，以及实际的主机和从机应用。最后作为实际案例，从 USB 协议实现的角度分析了`checkm8`漏洞的成因。

USB 101
=======

首先要明确的一点，USB 协议是以主机为中心的 (Host Centric)，也就是说只有主机端向设备端请求数据后，设备端才能向主机发送数据。从数据的角度来看，开发者最直接接触的就是端点 (Endpoint)，端点可以看做是数据收发的管道。

当主机给设备发送数据时，通常流程是:

• 调用用户层 API，如 `libusb_bulk_transfer`• 对内核的 USB 驱动执行对应系统调用，添加发送队列如 `ioctl(IOCTL_USBFS_SUBMITURB)`• 内核驱动中通过 HCI 接口向 USB 设备发送请求 + 数据 • 数据发送到设备端的 Controller -> HCI -> Host

设备给主机发送请求也是类似，只不过由于是主机中心，发送的数据会保存在缓存中，等待主机发送 IN TOKEN 之后才真正发送到主机。在介绍数据发送流程之前，我们先来看下描述符。

描述符
---

所有的 USB 设备端设备，都使用一系列层级的描述符 (Descriptors) 来向主机描述自身信息。这些描述符包括:

•**Device Descriptors**: 设备描述 •**Configuration Descriptors**: 配置描述 •**Interface Descriptors**: 接口描述 •**Endpoint Descriptors**: 端点描述 •**String Descriptors**: 字符串描述

它们之间的层级结构关系如下:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/3eicVGzibzClBCrnIckExlbOoaR3Hud7RHEzTZoHaiazdtGdZpnUsv0MhGuJ0IIZVcv8Xqv4PVuFlLU3nxoQhETbw/640?wx_fmt=png)des.png

每种描述符都有对应的数据结构，定义在标准中的第九章，俗称 ch9。下面以 Linux 内核的实现为例来简要介绍各个描述符，主要参考头文件 **include/uapi/linux/usb/ch9.h**。

### 设备描述

每个 USB 设备只能有一个设备描述 (Device Descriptor)，该描述符中包括了设备的 USB 版本、厂商、产品 ID 以及包含的配置描述符个数等信息，如下所示:

```
/* USB_DT_DEVICE: Device descriptor */
struct usb_device_descriptor {
    __u8  bLength; // 18 字节
    __u8  bDescriptorType; // 0x01
    __le16 bcdUSB; // 设备所依从的 USB 版本号
    __u8  bDeviceClass; // 设备类型
    __u8  bDeviceSubClass; // 设备子类型
    __u8  bDeviceProtocol; // 设备协议
    __u8  bMaxPacketSize0; // ep0 的最大包长度，有效值为 8，6，32，64
    __le16 idVendor; // 厂商号
    __le16 idProduct; // 产品号
    __le16 bcdDevice; // 设备版本号
    __u8  iManufacturer; // 产商字名称
    __u8  iProduct; // 产品名称
    __u8  iSerialNumber; // 序列号
    __u8  bNumConfigurations; // 配置描述符的个数
} __attribute__ ((packed));
#define USB_DT_DEVICE_SIZE        18

```

每个字段的含义都写在注释中了，其中有几点值得一提。

• 设备类型、子类型和协议码，是由 USB 组织定义的；• 产商号也是由 USB 组织定义的，但是产品号可以由厂商自行定义；• 厂商、产品和序列号分别只有 1 字节，表示在字符串描述符中的索引；

> BCD: binary­ coded decimal

### 配置描述

每种不同的配置描述 (Configuration Descriptor) 中分别指定了 USB 设备所支持的配置，如功率等信息；一个 USB 设备可以包含多个配置，但同一时间只能有一个配置是激活状态。实际上大部分的 USB 设备都只包含一个配置描述符。

```
/* USB_DT_CONFIG: Configuration descriptor information.
 *
 * USB_DT_OTHER_SPEED_CONFIG is the same descriptor, except that the
 * descriptor type is different.  Highspeed-capable devices can look
 * different depending on what speed they're currently running.  Only
 * devices with a USB_DT_DEVICE_QUALIFIER have any OTHER_SPEED_CONFIG
 * descriptors.
 */
struct usb_config_descriptor {
    __u8  bLength; // 9
    __u8  bDescriptorType; // 0x02
    __le16 wTotalLength; // 返回数据的总长度
    __u8  bNumInterfaces; // 接口描述符的个数
    __u8  bConfigurationValue; // 当前配置描述符的值 (用来选择该配置)
    __u8  iConfiguration; // 该配置的字符串信息 (在字符串描述符中的索引)
    __u8  bmAttributes; // 属性信息
    __u8  bMaxPower; // 最大功耗，以 2mA 为单位
} __attribute__ ((packed));
#define USB_DT_CONFIG_SIZE        9

```

当主设备读取配置描述的时候，从设备会返回该配置下所有的其他描述符，如接口、端点和字符串描述符，因此需要 **wTotalLength** 来表示返回数据的总长度。

**bmAttributes** 指定了该配置的电源参数信息，D6 表示是否为自电源驱动；D5 表示是否支持远程唤醒；D7 在 USB1.0 中曾用于表示是否为总线供电的设备，但是在 USB2.0 中被 **bMaxPower** 字段取代了，该字段表示设备从总线上消耗的电压最大值，以 2mA 为单位，因此最大电流大约是 `0xff * 2mA = 510mA`。

### 接口描述

一个配置下有多个接口，可以看成是一组相似功能的端点的集合，每个接口描述符的结构如下:

```
/* USB_DT_INTERFACE: Interface descriptor */
struct usb_interface_descriptor {
    __u8  bLength;
    __u8  bDescriptorType; // 0x04
    __u8  bInterfaceNumber; // 接口序号
    __u8  bAlternateSetting;
    __u8  bNumEndpoints;
    __u8  bInterfaceClass;
    __u8  bInterfaceSubClass;
    __u8  bInterfaceProtocol;
    __u8  iInterface; // 接口的字符串描述，同上
} __attribute__ ((packed));
#define USB_DT_INTERFACE_SIZE        9

```

其中接口类型、子类型和协议与前面遇到的类似，都是由 USB 组织定义的。在 Linux 内核中，每个接口封装成一个高层级的功能，即逻辑链接 (Logical Connection)，例如对 USB 摄像头而言，接口可以分为视频流、音频流和键盘(摄像头上的控制按键) 等。

还有值得一提的是 **bAlternateSetting**，每个 USB 接口都可以有不同的参数设置，例如对于音频接口可以有不同的带宽设置。实际上 Alternate Settings 就是用来控制周期性的端点参数的，比如 isochronous endpoint。

### 端点描述

端点描述符用来描述除了零端点 (ep0) 之外的其他端点，零端点总是被假定为控制端点，并且在开始请求任意描述符之前就已经被配置好了。端点(Endpoint)，可以认为是一个单向数据信道的抽象，因此端点描述符中包括传输的速率和带宽等信息，如下所示:

```
/* USB_DT_ENDPOINT: Endpoint descriptor */
struct usb_endpoint_descriptor {
    __u8  bLength;
    __u8  bDescriptorType; // 0x05
    __u8  bEndpointAddress; // 端点地址
    __u8  bmAttributes;    // 端点属性
    __le16 wMaxPacketSize; // 该端点收发的最大包大小
    __u8  bInterval; // 轮询间隔，只对 Isochronous 和 interrupt 传输类型的端点有效 (见下)
    /* NOTE:  these two are _only_ in audio endpoints. */
    /* use USB_DT_ENDPOINT*_SIZE in bLength, not sizeof. */
    __u8  bRefresh;
    __u8  bSynchAddress;
} __attribute__ ((packed));
#define USB_DT_ENDPOINT_SIZE        7
#define USB_DT_ENDPOINT_AUDIO_SIZE    9    /* Audio extension */

```

**bEndpointAddress** 8 位数据分别代表:

•Bit 0-3: 端点号 •Bit 4-6: 保留，值为 0•Bit 7: 数据方向，0 为 OUT，1 为 IN

**bmAttributes** 8 位数据分别代表:

•Bit 0-1: 传输类型 •00: Control•01: Isochronous•10: Bulk•11: Interrupt•Bit 2-7: 对非 Isochronous 端点来说是保留位，对 Isochronous 端点而言表示 Synchronisation Type 和 Usage Type，不赘述；

每种端点类型对应一种传输类型，详见后文。

### 字符串描述

字符串描述符 (String Descriptor) 中包含了可选的可读字符串信息，如果没提供，则前文所述的字符串索引应该都设置为 0，字符串表结构如下:

```
/* USB_DT_STRING: String descriptor */
struct usb_string_descriptor {
    __u8  bLength;
    __u8  bDescriptorType; // 0x03
    __le16 wData[1];        /* UTF-16LE encoded */
} __attribute__ ((packed));
/* note that "string" zero is special, it holds language codes that
 * the device supports, not Unicode characters.
 */

```

字符串表中的字符都以 Unicode[2] 格式编码，并且可以支持多种语言。0 号字符串表较为特殊，其中 wData 包含一组所支持的语言代码，每个语言码为 2 字节，例如 0x0409 表示英文。

传输
--

不像 RS-232 和其他类似的串口协议，USB 实际上由多层协议构造而成，不过大部分底层的协议都在 Controller 端上的硬件或者固件进行处理了，最终开发者所要关心的只有上层协议。

### USB Packet

在 HCI 之下，实际传输的数据包称为 Packet，每次上层 USB 传输都会涉及到 2-3 次底层的 Packet 传输，分别是:

•Token Packet: 总是由主机发起，指示一次新的传输或者事件 •In: 告诉 USB 设备，主机我想要读点信息 •Out: 告诉 USB 设备，主机我想要写点信息 •Setup: 用于开始 Control Transfer•Data Packet: 可选，表示传输的数据，可以是主机发送到设备，也可以是设备发送到主机 •Data0•Data1•Status Packet: 状态包，用于响应传输，以及提供纠错功能 •Handshake Packets: ACK/NAK/STALL•Start of Frame Packets

### Transfer

基于这些底层包，USB 协议定义了四种不同的传输类型，分别对应上节中的四种端点类型，分别是:

**Control Transfers**: 主要用来发送状态和命令，比如用来请求设备、配置等描述以及选择和设置指定的描述符。只有控制端点是双向的。

**Interrupt Transfers**: 由于 USB 协议是主机主导的，设备端的中断信息需要被及时响应，就要用到中断传输，其提供了有保证的延迟以及错误检测和重传功能。中断传输通常是非周期性的，并且传输过程保留部分带宽，常用于时间敏感的数据，比如键盘、鼠标等 HID 设备。

**Isochronous Transfers**: 等时传输，如其名字所言，该类传输是连续和周期性的，通常包含时间敏感的信息，比如音频或视频流。因此这类传输不保证到达，即没有 ACK 响应。

**Bulk Transfers**: 用于传输大块的突发数据 (小块也可以)，不保留带宽。提供了错误校验(CRC16) 和重传机制来保证传输数据的完整性。块传输只支持高速 / 全速模式。

这里以控制传输 (Control Transfers) 为例，来看看底层 Packet 如何组成一次完整的传输。控制传输实际上又可能最多包含三个阶段，每个阶段在应用层可以看成是一次 “USB 传输” (在 Wireshark 中占一行)，分别是:

•Setup Stage: 主机发送到设备的请求，包含三次底层数据传输 1.Setup Token Packet: 指定地址和端点号 (应为 0)2.Data0 Packet: 请求数据，假设是 8 字节的 `Device Descriptor Request`3.ACK Handshake Packet: 设备的响应， 不允许用 STALL 或者 NAK 来响应 Setup Packet•Data Stage: 可选阶段，包含一个或者多个 IN/OUT 传输，以 IN 为例，也包含三次传输 1.IN Token Packet: 表示主机端要从设备端读数据 2.Datax Packet: 如果上面 Setup Stage 是 `Device Descriptor Request`， 这里返回 `Device Descriptor Response` (的前 8 字节，然后再根据实际长度再 IN 一次)。3.ACK/STALL/NAK Status Packet•Status Stage: 报告本次请求的状态，底层也是三次传输，但是和方向有关:• 如果在 Data Stage 发送的是 IN Token，则该阶段包括:1.OUT Token2.Data0 ZLP(zero length packet): 主机发送长度为 0 的数据 3.ACK/NACK/STALL: 设备返回给主机 • 如果在 Data Stage 发送的是 OUT Token，则该阶段包括:1.IN Token2.Data0 ZLP: 设备发送给主机，表示正常完成，否则发送 NACK/STALL3.ACK: 如果是 ZLP，主机响应设备，双向确认

每个阶段的数据都有自己的格式，例如 Setup Stage 的 Request，即 Data0 部分发送的 8 字节数据结构如下:

```
struct usb_ctrlrequest {
    __u8 bRequestType; // 对应 USB 协议中的 bmRequestType，包含请求的方向、类型和指定接受者
    __u8 bRequest; // 决定所要执行的请求
    __le16 wValue; // 请求参数
    __le16 wIndex; // 同上
    __le16 wLength; // 如果请求包含 Data Stage，则指定数据的长度
} __attribute__ ((packed));

```

下面是一些标准请求的示例:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/3eicVGzibzClBCrnIckExlbOoaR3Hud7RHkJFiacKj6IrjGWytL1Pokz2qGanYyLxuNAw7kO4GjV2k95m4baBrJeg/640?wx_fmt=png)

> ref: https://www.beyondlogic.org/usbnutshell/usb6.shtml

虽然 HCI 之下传输的数据包大部分情况下对应用开发者透明，但是了解底层协议发生了什么也有助于加深我们对 USB 的理解，后文中介绍 checkm8 漏洞时候就用到了相关知识。

主机端
===

在主机端能做的事情相对有限，主要是分析和使用对应的 USB 设备。

抓包分析
----

使用 wireshark 可以分析 USB 流量，根据上面介绍的描述符字段以及 USB 传输过程进行对照，可以加深我们对 USB 协议的理解。如下是对某个安卓设备的 Device Descriptor Response 响应:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/3eicVGzibzClBCrnIckExlbOoaR3Hud7RH19KX8DGbwppo92cKzxjicDBrYCtkic9KIicjb0QAcvhDCaGQvJmF58mng/640?wx_fmt=png)device.png

也就是所谓安卓变砖恢复时经常用到的高通 9008 模式。说个题外话，最近对于高通芯片 BootROM 的研究发现了一些有趣的东西，后面可能会另外分享，Stay Tune！

应用开发
----

对于应用开发者而言，通常是使用封装好的库，早期只有 libusb，后来更新了 libusb1.0，早期的版本变成 libusb0.1，然后又有了 OpenUSB[3] 和其他的 USB 库。但不管用哪个库，调用的流程都是大同小异的。以 Python 的封装 pyusb 为例，官方给的示例如下:

```
import usb.core
import usb.util
# find our device
dev = usb.core.find(idVendor=0xfffe, idProduct=0x0001)
# was it found?
if dev is None:
    raise ValueError('Device not found')
# set the active configuration. With no arguments, the first
# configuration will be the active one
dev.set_configuration()
# get an endpoint instance
cfg = dev.get_active_configuration()
intf = cfg[(0,0)]
ep = usb.util.find_descriptor(
    intf,
    # match the first OUT endpoint
    custom_match = \
    lambda e: \
        usb.util.endpoint_direction(e.bEndpointAddress) == \
        usb.util.ENDPOINT_OUT)
assert ep is not None
# write the data
ep.write('test')

```

总的来说分为几步，

1. 根据设备描述符查找到指定的设备 2. 获取该设备的配置描述符，选择并激活其中一个 3. 在指定的配置中查找接口和端点描述符 4. 使用端点描述符进行数据传输

如果不清楚 USB 的工作原理，会觉得上面代码的调用流程很奇怪，往 USB 上读写数据需要那么复杂吗？但正是因为 USB 协议的高度拓展性，才得以支持这么多种类的外设，从而流行至今。

设备端
===

对于想要开发设备端 USB 功能的开发者而言，使用最广泛的要数**树莓派 Zero** 了，毕竟这是树莓派系列中唯一支持 USB OTG 的型号。网上已经有很多资料教我们如何将树莓派 Zero 配置成 USB 键盘、打印机、网卡等 USB 设备的教程。当然使用其他硬件也是可以的，配置自定义的 USB 设备端可以让我们做很多有趣的事情，比如网卡中间人或者 Bad USB 这种近源渗透方式。后文中我们会使用 Zero 进行简单测试。

一些相关的配置资料可以参考:

•https://github.com/RoganDawes/P4wnP1•Using RPi Zero as a Keyboard[4]

内核驱动
----

在介绍应用之间，我们先看看内核的实现。还是以 Linux 内核为例，具体来说，我们想了解如何通过添加内核模块的方式实现一个新的自定义 USB 设备。俗话说得好，添加 Linux 驱动的最好方式是参看现有的驱动，毕竟当前内核中大部分都是驱动代码。

因为 Linux 内核既能运行在主机端，也能运行在设备端，因此设备端的 USB 驱动有个不同的名字: **gadget** driver。对于不同设备，也提供不同的内核接口，即 Host-Side API 和 Gadget API。既然我们是想实现自己的设备，就需要从 gadget 驱动入手。

`g_zero.ko` 就是这么一个驱动，代码在 drivers/usb/gadget/legacy/zero.c。该驱动实现了一个简单的 USB 设备，包含 2 个配置描述，各包含 1 个功能，分别是 sink 和 loopback，前者接收数据并返回 0，后者接收数据并原样返回:

•drivers/usb/gadget/function/f_sourcesink.c•drivers/usb/gadget/function/f_loopback.c

代码量不多，感兴趣的自行 RTFSC。另外值得一提的是，对于运行于 USB device 端的系统而言，内核中至少有三个层级处理 USB 协议，可能用户层还有更多。gadget API 属于三层的中间层。至底向上，三层分别是:

1.USB Controller Driver: 这是软件的最底层，通过寄存器、FIFO、DMA、IRQ 等其他手段直接和硬件打交道，通常称为 `UDC` (USB Device Controller) Driver。2.Gadget Driver: 作为承上启下的部分，通过调用抽象的 UDC 驱动接口，底层实现了硬件无关的 USB function。主要用于实现前面提到的 USB 功能，包括处理 setup packet (ep0)、返回各类描述符、处理各类修改配置情况、处理各类 USB 事件以及 IN/OUT 的传输等等。3.Upper Level: 通过 Gadget Driver 抽象的接口，实现基于 USB 协议的上层应用，比如 USB 网卡、声卡、文件存储、HID 设备等。

关于 Linux USB 子系统的详细设计结构，可以参考源码中的文档: Linux USB API[5]，以及其他一些资料，如下所示:

•https://bootlin.com/doc/legacy/linux-usb/linux-usb.pdf•https://static.lwn.net/images/pdf/LDD3/ch13.pdf•https://elinux.org/images/5/5e/Opasiak.pdf

GadgetFS/ConfigFS
-----------------

参考现有的 Linux 驱动，依葫芦画瓢可以很容易实现一个自定义的 USB Gadget。但是这样存在一些问题，如果我想实现一个八声道的麦克风，还要重新写一遍驱动、编译、安装，明明内核中麦克风的功能已经有了，复制粘贴就显得很不优雅。

那么，有没有什么办法可以方便组合和复用现有的 gadget function 呢？在 Linux 3.11 中，引入了 USB Gadget ConfigFS，提供了用户态的 API 来方便创建新的 USB 设备，并可以组合复用现有内核中的驱动。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/3eicVGzibzClBCrnIckExlbOoaR3Hud7RHOaMXz8Fq0Hp4dcE8nVnbJw8cKz9UoDjicJJECDLL0ADw2waY6IxYrgA/640?wx_fmt=png)gfs.png

前文提到的基于树莓派 Zero 实现的各类 USB 设备，大部分都是基于 Gadget ConfigFS 接口实现的。基于 configfs 创建 USB gadget 的步骤一般如下:

```
CONFIGFS_HOME=/sys/kernel/config/usb_gadget
# 1. 新建一个 gadget，并写入实际的设备描述
mkdir $CONFIGFS_HOME/mydev # 创建设备目录后，该目录下自动创建并初始化了一个设备模板
cd $CONFIGFS_HOME/mydev
echo 0x0100 > bcdDevice # Version 1.0.0
echo 0x0200 > bcdUSB # USB 2.0
echo 0x00 > bDeviceClass
echo 0x00 > bDeviceProtocol
echo 0x40 > bMaxPacketSize0
echo 0x0104 > idProduct # Multifunction Composite Gadget
echo 0x1d6b > idVendor # Linux Foundation
# 2. 新建一个配置，并写入实际的配置描述
mkdir configs/c.1 # 创建一个配置实例: <config name>.<config number>
cd configs/c.1
echo 0x01 > MaxPower
echo 0x80 > bmAttributes
# 3. 新建一个接口(function)，或者将已有接口链接到当前配置下
cd $CONFIGFS_HOME/mydev
mkdir functions/hid.usb0 # 创建一个 function 实例: <function type>.<instance name>
echo 1 > functions/hid.usb0/protocol
echo 8 > functions/hid.usb0/report_length # 8-byte reports
echo 1 > functions/hid.usb0/subclass
ln -s functions/hid.usb0 configs/c.1
# 4. 将当前 USB 设备绑定到 UDC 驱动中
echo ls /sys/class/udc > $CONFIGFS_HOME/mydev/UDC

```

这样就实现了一个最简单的 USB gadget，当然要完整实现的话还可以添加字符串描述，以及增加各个端点的功能。使用 configfs 实现一个 USB 键盘的示例可以参考网上其他文章，比如 Using RPi Zero as a Keyboard[6]，或者 Github 上的开源项目，比如 P4wnP1[7]。

有些人觉得 ConfigFS 配置起来很繁琐，所以开发了一些函数库 (如 libusbgx) 来通过调用创建 gadget；有人觉得通过函数操作也还是繁琐，就创建了一些工具 (如 gt[8]) 来通过处理一个类似于 libconfig 的配置文件直接创建 gadget，不过笔者用得不多。

FunctionFS
----------

FunctionFS 最初是对 GadgetFS 的重写，用于支持实现用户态的 gadget function，并组合到现有设备中。这里说的 FunctionFS 实际上是新版基于 ConfigFS 的 GadgetFS 拓展。在上一节中说到创建设备 gadget 的第四步就是给对应的 configuration 添加 function，格式为 **function_type.instance_name**，type 对应一个已有的内核驱动，比如上节中是 `hid`。

如果要使用当前内核中没有的 function 实现自定义的功能，那么内核还提供了一个驱动可以方便在用户态创建接口，该驱动就是 ffs 即 FunctionFS。使用 ffs 的方式也很简单，将上面第三步替换为:

```
cd $CONFIGFS_HOME/mydev
mkdir functions/ffs.usb0
ln -s functions/ffs.usb0 configs/c.1

```

创建一个类型为 ffs，名称为 usb0 的 function，然后挂载到任意目录:

```
cd /mnt
mount usb0 ffs -t functionfs

```

挂载完后，_/mnt/ffs_ 目录下就已经有了一个 ep0 文件，如名字所言正是 USB 设备的零端点，用于收发 Controller Transfer 数据以及各类事件。在该目录中可以创建其他的端点，并使用类似文件读写的操作去实现端点的读写，内核源码中提供了一个用户态应用示例，代码在 tools/usb/ffs-test.c。如果嫌 C 代码写起来复杂，还可以使用 Python 编写 ffs 实现，比如 python-functionfs[9]。

案例分析: checkm8 漏洞
================

checkm8 漏洞就不用过多介绍了，曾经的神洞，影响了一系列苹果设备，存在于 BootROM 中，不可通过软件更新来修复，一度 Make iOS Jailbreak Great Again。当然现在可以通过 SEP 的检查来对该漏洞进行缓解，这是后话。

关于 checkm8 的分析已经有很多了，我们就不再鹦鹉学舌，更多是通过 checkm8 的成因，来从漏洞角度加深对 USB device 开发的理解。

checkm8 漏洞发生在苹果的救砖模式 DFU (Device Firmware Upgrade)，即通过 USB 向苹果设备刷机的协议。该协议是基于 USB 协议的一个拓展，具体来说:

• 基于 USB Control Transfer•bmRequestType[6:5] 为 0x20，即 **Type** 为 Class•bmRequestType[4:0] 为 0x01，即 **Recipient** 为 Interface•bRequest 为 DFU 相关操作，比如 Detach、Download、Upload、GetStatus、Abort 等

DFU 接口初始化的代码片段如下:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/3eicVGzibzClBCrnIckExlbOoaR3Hud7RHDWyElhhxr0XliaR1tQBtU6aNwEHZ3GUQGGFXMf2vEtx0P2PAUiaWp1pw/640?wx_fmt=png)dfu.png

Control Transfer 主要是在 ep0 上传输，因此 ep0 的读写回调中就会根据收到的数据来派发到不同的 handler，对于 DFU 协议的分发伪代码如下:

```
static byte *data_buf;
static size_t data_rcvd;
static size_t data_size;
static struct usb_ctrlrequest setup_request; 
void handle_ctr_transfer_recv(byte *buf, int len, int *p_stage, int is_setup) {
  *p_stage = 0;
  if (!is_setup) {
    handle_data_recv(buf, len, p_stage);
  }
  // handle control request
  memcpy(&setup_request, buf, 8);
  switch(setup_request.bRequestType & 0x60) {
    case STANDARD:
      // ...
    case VENDOR:
      // ...
    case CLASS:
      if (setup_request.bRequestType & 0x1f == INTERFACE) {
        int n = intf_handlers[setup_request.wIndex]->handle_request(&setup_request, &data_buf);
        if (n > 0) {
          data_size = n;
        }
      }
    default:
    // ...
  }
}

```

其中 intf_handlers 是 usb_core_regisger_interface 函数中添加到的的全局函数数组。handle_reuqest 中传入的是一个指针的指针，并在处理函数中复制为 io_buffer 的地址。而开头的 data stage 阶段，内部实现就是将收到的数据拷贝到 data_buf 即 io_buffer 中。

io_buffer 一直是有效的吗？并不尽然，因为 io_buffer 在 DFU 退出阶段会被 free 释放掉，此后 data_buf 仍然持有着无效指针，就构成了一个典型的 UAF 场景，这正是 checkm8 的漏洞所在。至于如何触发以及如何构造利用，可以需要额外的篇幅去进行介绍，感兴趣的朋友可以参考文末的文章。

从 checkm8 漏洞中我们可以看到出现漏洞的根本成因:

• 大量使用全局变量 • 在处理 USB 内部状态机出现异常时，没有充分清除全局变量的值，比如只将 io_buffer 置零而没有将 data_buf 置零 • 在重新进入状态机时，全局变量仍然有残留，导致进入异常状态或者处理异常数据

网上有人评论说这么简单的漏洞为什么没有通过自动化测试发现出来，个人感觉这其实涉及到模糊测试的两大难题:

一是针对 stateful 的数据测试，每增加一种内部状态，测试的分支就成指数级别增长，从而增加了控制流覆盖到目标代码的难度；

二是硬件依赖，要测试这个 USB 状态机，需要 mock 出底层的驱动接口，工作量和写一个新的 USB 驱动差不多，更不用说 DFU 本身还会涉及存储设备的读写，这部分接口是不是也要模拟？

因此这类漏洞的更多是通过代码审计发现出来，不过厂商又执着于 **Security by Obsecurity**，这就导致投入的更多是利益驱动的组织，对个人用户安全而言并不算是件好事。如果 iBoot 开源，那么估计这个漏洞早就被提交给苹果 SRC，成本也就几千欢乐豆的事，也不至于闹出这么大的舆情，甚至以 checkm8 为跳板，把 SEPOS 也撸了个遍。

后记
==

本文是最近对 USB 相关的一些学习记录，虽然文章是从前往后写的，但实际研究却是从后往前做的。即先看到了网上分析 checkm8 的文章，为了复现去写一个 USB 设备，然后再去学习 USB 协议的细节，可以算是个 Leaning By Hacking 的案例吧。个人感觉这种方式前期较为痛苦，但后期将点连成线之后还是挺醍醐灌顶的，也算是一种值得推荐的研究方法。

参考资料
====

•USB in a NutShell[10]•USB and the Real World[11]•pyusb/pyusb[12]•Linux USB API[13]•Kernel USB Gadget Configfs Interface[14]•Technical analysis of the checkm8 exploit[15]

#### 引用链接

`[1]` Universal Serial Bus: _https://en.wikipedia.org/wiki/USB_  
`[2]` Unicode: _http://www.unicode.org/_  
`[3]` OpenUSB: _http://sourceforge.net/p/openusb/wiki/Home/_  
`[4,6]` Using RPi Zero as a Keyboard: _https://www.rmedgar.com/blog/using-rpi-zero-as-keyboard-setup-and-device-definition_  
`[5]` Linux USB API: _https://www.kernel.org/doc/html/v4.18/driver-api/usb/index.html_  
`[7]` P4wnP1: _https://github.com/RoganDawes/P4wnP1_  
`[8]` gt: _https://github.com/kopasiak/gt_  
`[9]` python-functionfs: _https://github.com/vpelletier/python-functionfs_  
`[10]` USB in a NutShell: _https://www.beyondlogic.org/usbnutshell/usb1.shtml_  
`[11]` USB and the Real World: _https://elinux.org/images/a/ae/Ott--usb_and_the_real_world.pdf_  
`[12]` pyusb/pyusb: _https://github.com/pyusb/pyusb/blob/master/docs/tutorial.rst_  
`[13]` Linux USB API: _https://www.kernel.org/doc/html/v4.18/driver-api/usb/index.html_  
`[14]` Kernel USB Gadget Configfs Interface: _https://www.elinux.org/images/e/ef/USB_Gadget_Configfs_API_0.pdf_  
`[15]` Technical analysis of the checkm8 exploit: _https://habr.com/en/company/dsec/blog/472762/_