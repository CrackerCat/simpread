> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/)

> 从 PR 中学习如何修改 flashrom 读取国产 flash 前段时间分析了一个摄像头，在提取固件的时候遇到了一些麻烦。

[](#从PR中学习如何修改-flashrom-读取国产-flash "从PR中学习如何修改 flashrom 读取国产 flash")从 PR 中学习如何修改 flashrom 读取国产 flash
----------------------------------------------------------------------------------------------------

前段时间分析了一个摄像头，在提取固件的时候遇到了一些麻烦。flashrom 不能识别、读取 Flash，鉴于当前的环境越来越多的产品使用国产方案，传统开源工具对国内的芯片支持相对滞后。这种情况下就需要掌握对不常见芯片的拓展支持能力。

### [](#直接提取固件失败 "直接提取固件失败")直接提取固件失败

Flash 是 XMC(长江存储) 的 XM25QH128A，大小为 16MB，官网上没找到[数据手册](https://www.semiee.com/file/XMC/XMC-XM25QH128A.pdf)，在半导小芯上找到了。下图是芯片的引脚说明，引脚采用了标准布局。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210222225432301.png)

FT232H 与 Flash 的连接表。

<table><thead><tr><th>FT232H</th><th>SPI FLASH</th></tr></thead><tbody><tr><td>AD0(SCK)</td><td>SCK</td></tr><tr><td>AD1(MOSI)</td><td>MOSI</td></tr><tr><td>AD2(MISO)</td><td>MISO</td></tr><tr><td>AD3(/CS)</td><td>/CS</td></tr><tr><td>VCC 3.3V</td><td>VCC 3.3V(+/HOLD or /RESET,/WP</td></tr><tr><td>GND</td><td>GND</td></tr></tbody></table>

使用 flashrom 提取 FLASH 失败，提示 “unknown SPI chip (REMS)” 未知的 SPI 芯片。

```
root@kali:~
flashrom v1.2 on Linux 5.10.0-kali3-amd64 (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
Found Generic flash chip "unknown SPI chip (REMS)" (0 kB, SPI) on ft2232_spi.
===
This flash part has status NOT WORKING for operations: PROBE READ ERASE WRITE
The test status of this chip may have been updated in the latest development
version of flashrom. If you are running the latest development version,
please email a report to flashrom@flashrom.org if any of the above operations
work correctly for you with this flash chip. Please include the flashrom log
file for all operations you tested (see the man page for details), and mention
which mainboard or programmer you tested in the subject line.
Thanks for your help!
Read is not working on this chip. Aborting.
```

查看 flashrom 帮助信息发现最新的 V1.2 release 版本还不支持长江存储的 Flash。 但 就 1 月份 Github 上已经有人 [PR](https://github.com/flashrom/flashrom/commit/32f4cb4ffa2854354f00e5facc9ccb8c9beafd61) 了代码，对部分长江存储芯片做了支持。其中有 XM25QH128C，应该也能读取 XM25QH128A 吧。没有去看手册，盲猜一把。

```
XM25QU64C
XM25QU128C
XM25QU256C
XM25QH64C
XM25QH128C
XM25QH256C
```

### [](#手动编译-flashrom "手动编译 flashrom")手动编译 flashrom

刚开始编译的时候，没有安装 libftdi 库，报错提示 “未知的编程器”。

```
flashrom Error: Unknown programmer "ft2232_spi:type=232H"
```

后面去看了一下编译说明，才发现问题。

```
* pciutils+libpci (if you want support for mainboard or PCI device flashing)
* libusb (if you want FT2232, Dediprog or USB-Blaster support)
* libftdi (if you want FT2232 or USB-Blaster support)
```

是我草率了，以上来就 make，然后根据报错安装依赖。以后还是需要看一眼说明再来。并不是所有的缺少依赖都会提醒（报错），有些是可选依赖，按需配置。

```
apt-get install libftdi-dev libpci-dev libusb-dev
```

安装好依赖之后，使用 Make 命令编译后就能识别 FT232H 了。

```
root@kali:/opt/flashrom-master
flashrom  on Linux 5.10.0-kali3-amd64 (x86_64)
flashrom is free software, get the source code at https://flashrom.org
```

手动编译了 flashrom，再次尝试读取仍旧失败。于是，就只能修改源码，添加对这款芯片的支持。

### [](#分析-pull-requests-中的内容 "分析 pull requests 中的内容")分析 pull requests 中的内容

提取 SPI Flash 中的固件对 flashrom 工具比较依赖，但遇到不支持的芯片的时候就很尴尬。趁这个机会，跟着 Commit 来了解一下怎么添加对新设备的支持。为什么是跟着别人的 `pull requests`学习呢，直接阅读完整源码花费的时间很长，但看别人提交的对某芯片的支持就只需要花费很短的时间，然后模仿新增的代码编写自己需要的内容，降低学习成本。接下来看看 [flashchips.c: Add support for XMC new SPI flash types](https://github.com/flashrom/flashrom/commit/32f4cb4ffa2854354f00e5facc9ccb8c9beafd61) 为支持 XMC 的部分芯片添加了哪些代码。

在 PR 中对 XMC 的 6 个型号的芯片做了支持，新增代码也不多。接下来看看是怎么对 **XM25QH128C** 添加支持的。

首先在 [flashchips.h](https://github.com/flashrom/flashrom/commit/32f4cb4ffa2854354f00e5facc9ccb8c9beafd61#diff-f98a7e2ed1def86dec092724417b7c43c5adb183ea3c81b7e5fd905be0b8bb68) 对各 Flash 头信息进行了声明。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223175253389.png)

这些值是怎么确定的，回到 [芯片手册](https://www.semiee.com/file/XMC/XMC-XM25QH128C.pdf) 中。芯片手册的第七章，7.1.1 表格 “制造商和设备标识” 表格可以回答。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223175922908.png)

0x20 是制造商信息。0x4018 是设备的标识信息。

首先添加便是制造商 ID，既然原来已经存在 0x20 的制造商 ID，这里就只需要标注一下，XMC 也需要用到这个值。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223180143750.png)

然后是设备信息，**XM25QH128C** 的 ID 为 0x4018。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223180654733.png)

头文件中只定义了这两个值，其他需要添加的代码在 [flashchips.c](https://github.com/flashrom/flashrom/commit/32f4cb4ffa2854354f00e5facc9ccb8c9beafd61#diff-dc874507fb2d47cc63e00b1408078a636033c9ae4298597204ead3102e60ef19) 中。在这个文件中为 Flash 进行了详细的描述。

```
{
    .vendor		= "XMC",
    .name		= "XM25QH128C",
    .bustype	= BUS_SPI,
    .manufacture_id	= ST_ID,
    .model_id	= XMC_XM25QH128C,
    .total_size	= 16384,
    .page_size	= 256,
    .feature_bits	= FEATURE_WRSR_WREN | FEATURE_OTP | FEATURE_QPI,
    .tested		= TEST_UNTESTED,
    .probe		= probe_spi_rdid,
    .probe_timing	= TIMING_ZERO,
    .block_erasers	=
    {
        {
            .eraseblocks = { {4 * 1024, 4096} },
            .block_erase = spi_block_erase_20,
        }, {
            .eraseblocks = { {32 * 1024, 512} },
            .block_erase = spi_block_erase_52,
        }, {
            .eraseblocks = { {64 * 1024, 256} },
            .block_erase = spi_block_erase_d8,
        }, {
            .eraseblocks = { {16 * 1024 * 1024, 1} },
            .block_erase = spi_block_erase_60,
        }, {
            .eraseblocks = { {16 * 1024 * 1024, 1} },
            .block_erase = spi_block_erase_c7,
        }
    },
    .printlock	= spi_prettyprint_status_register_plain,
    .unlock		= spi_disable_blockprotect,
    .write		= spi_chip_write_256,
    .read		= spi_chip_read,
    .voltage	= {2700, 3600},
},
```

从以上的代码中，可以看到供应商、芯片名称、总线类型、制造商编号、模块编号、总容量、页大小、特性位、读写方式、电压等。这些信息都能在手册中找到。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223183121407.png)

flashchips 结构体详细定义如下。

```
/*
* .vendor		= Vendor name                                    供应商名称
* .name		= Chip name                                          芯片名
* .bustype		= Supported flash bus types (Parallel, LPC...)   总线类型
* .manufacture_id	= Manufacturer chip ID                       制造商芯片ID
* .model_id		= Model chip ID                                  芯片ID
* .total_size		= Total size in (binary) kbytes              总容量
* .page_size		= Page or eraseblock(?) size in bytes        页大小
* .tested		= Test status                                    测试状态
* .probe		= Probe function                                 探测函数 
* .probe_timing	= Probe function delay                           探测函数延时
* .block_erasers[]	= Array of erase layouts and erase functions 
* {
*	.eraseblocks[]	= Array of { blocksize, blockcount }         擦除块 块大小 块数量
*	.block_erase	= Block erase function                       擦除函数
* }
* .printlock		= Chip lock status function                  芯片锁状态函数
* .unlock		= Chip unlock function                           芯片解锁函数
* .write		= Chip write function                            芯片写函数
* .read		= Chip read function                                 芯片读函数
* .voltage		= Voltage range in millivolt                     电压范围
*/
```

### [](#添加对-XMC-的支持 "添加对 XMC 的支持")添加对 XMC 的支持

由于 commit 中没有 XM25QH128A，但有 XM25QH128C，版本不同或许能通用。下面来分析现有代码能不能操作。

从 [芯片手册](https://www.semiee.com/file/XMC/XMC-XM25QH128A.pdf) 中发现还是有些差异的。制造商 ID 还是 0x20，但设备 ID 为 0x7018。正是由于这个值不匹配，所以 flashrom 无法识别芯片。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223184740360.png)

现在可以根据[芯片手册](https://www.semiee.com/file/XMC/XMC-XM25QH128A.pdf)对 XM25QH128A 添加支持，新增的代码不多，只需要向 flashchips.h 和 flashchips.c 添加一些内容。

在 flashchips.h 中，定义芯片名称`XMC_XM25QH128A`及设备 ID`0x7018`。

```
#define XMC_XM25QH128A		0x7018
```

在 flashchips.c 中，向 flashchips 结构体添加 XM25QH128A 描述。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210307104857874.png)

设置供应商为`XMC`, 名称为 `XM25QH128A`, 总线类型为 SPI `BUS_SPI`, 制造商 ID `ST_ID`，总容量 16384KB，页大小 256，特性位 `Write Enable (WREN)`、·`OTP`、`QPI`，电压区间从 2700mV 到 3600mV 等。

```
{ 
    .vendor     = "XMC",
    .name       = "XM25QH128A",
    .bustype    = BUS_SPI,
    .manufacture_id = ST_ID,
    .model_id   = XMC_XM25QH128A,
    .total_size = 16384,
    .page_size  = 256,
    .feature_bits   = FEATURE_WRSR_WREN | FEATURE_OTP | FEATURE_QPI,
    .tested     = TEST_UNTESTED,
    .probe      = probe_spi_rdid,
    .probe_timing   = TIMING_ZERO,
    .block_erasers  =
    {     
        {     
            .eraseblocks = { {4 * 1024, 4096} },
            .block_erase = spi_block_erase_20,
        }, {  
            .eraseblocks = { {32 * 1024, 512} },
            .block_erase = spi_block_erase_52,
        }, {  
            .eraseblocks = { {64 * 1024, 256} },
            .block_erase = spi_block_erase_d8,
        }, {  
            .eraseblocks = { {16 * 1024 * 1024, 1} }, 
            .block_erase = spi_block_erase_60,
        }, {  
            .eraseblocks = { {16 * 1024 * 1024, 1} }, 
            .block_erase = spi_block_erase_c7,
        }     
    },    
    .printlock  = spi_prettyprint_status_register_plain,
    .unlock     = spi_disable_blockprotect,
    .write      = spi_chip_write_256,
    .read       = spi_chip_read,
    .voltage    = {2700, 3600},
},
```

### [](#验证 "验证")验证

对修改后的 flashrom 源码进行编译，然后再次去读取固件。不一会儿，固件读取完成。但还需要验证固件是否正确读取。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223204738839.png)

binwalk 分析提取出来的固件，识别出了其中存在 Squashfs、JFFS2 文件系统等，表明提取固件成功。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210223205213327.png)

最后，向 flashrom 提交了 [PR](https://github.com/flashrom/flashrom/pull/193)，方便其他人使用。

![](https://delikely.github.io/2021/03/20/add-support-to-flashrom-for-XM25Q/image-20210307103548616.png)

### [](#总结 "总结")总结

IOT 设备与方案众多，不时会遇到现有工具不支持的情况，此时需要自己添加支持。这次发现从 PR 中分析并模仿是个比较省时的方法。

首发于：[ChaMd5 安全团队 - 微信公众号](https://mp.weixin.qq.com/s/kifu_p4eOfy1kuSfLMrXMw)