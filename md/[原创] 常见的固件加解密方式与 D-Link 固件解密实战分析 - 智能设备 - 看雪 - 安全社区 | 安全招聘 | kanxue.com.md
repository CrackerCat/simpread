> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281043.htm)

> [原创] 常见的固件加解密方式与 D-Link 固件解密实战分析

[原创] 常见的固件加解密方式与 D-Link 固件解密实战分析

1 天前 838

### [原创] 常见的固件加解密方式与 D-Link 固件解密实战分析

 [![](http://passport.kanxue.com/upload/avatar/693/964693.png?1701522539)](user-home-964693.htm) [Arahat0](user-home-964693.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif) 1 天前  838

**常见固件的加解密方式与 D-Link 固件解密实战分析**
===============================

前言
--

当我们需要进行固件分析时，首先要做的就是获取固件，而获取固件无外乎就是从官网获取固件、通过流量拦截获取固件、使用编程器从闪存中读取固件以及通过串口调试提取固件。提取到固件之后，下一步就是对固件的进行分析，分析的对象主要是固件的内核与文件系统，包括 Web 应用、协议、核心控制程序等。

但是现在大多数厂商对了保证自己家产品安全，防止被他人攻击，就会对固件进行加密处理，可能是使用 AES、DES、SM4 等复杂的加密方式，也可能是使用 XOR、ROT 等简单的加密方式，加解密的程序一般放于 Boot loader、内核或者文件系统中。这种情况下我们就不能直接使用 `binwalk` 等工具提取，需要先对加密的固件进行解密。

而厂商对固件的加密一般以下面三种情况为主：

常见的固件加解密方式
----------

### 1、固件出厂未加密，后续发布包含解密方案的未加密版本，最后发布加密版本

设备固件在出厂时未加密（假设此时的版本是 v1.0），也未包含任何解密释序。厂商后续会发布一个包含解密程序的 v1.1 版本的未加密固件，最后再发布一个含有解密程序的 v1.2 版本的加密固件。此时，我们可以从固件 v1.1 中获取解密程序，用它来解密 v1.2 版本的固件，然后进行固件提取。

![](https://bbs.kanxue.com/upload/attach/202403/964693_2FP3KFFNGNMVYSU.webp)

### **2、固件出厂加密，后续发布包含新版解密方案的未加密固件，最后发布新版加密版本**

厂商直接在设备固件的原始版本中进行了加密，但是厂商决定更改加密方案并发布一个未加密的 v1.2 版本的新固件作为过渡，其中包含了新版本的解密程序。

![](https://bbs.kanxue.com/upload/attach/202403/964693_FFECYVTN335PFE2.webp)

在更新固件版本之前，需要先看新固件版本的发布通告，这个通告会指示用户在将固件升级到最新版本之前，需要先升级到固件的一个中间版本，而这个中间版本就是这个未加密的固件版本。通过这个中间版本的固件进行升级，最终可获取新版本加密固件的解密程序。

### 3、**固件出厂加密，后续发布包含新版解密方案的加密固件，最后发布新版加密版本**

从网上下载的设备固件在原始版本中进行了加密，厂商决定更改加密方案并发布一个带新版解密程序的中间版迭代加密固件，但是由于对初始版本的固件就进行了加密，因此很难获得解密程序。

![](https://bbs.kanxue.com/upload/attach/202403/964693_U7NCEPGC9BAYVTY.webp)

此时，想对加密后的固件进行解密会比较困难。针对这种情况，一种思路是购买设备并使用 `JTAG`、`UART` 调试等方法进入 `Linux Shell` 或者 `Uboot Shell`，直接从设备硬件中提取固件的文件系统。然后就是对固件进行更深层次的分析，看看如何能够对加密的固件进行逆向分析，得到加密逻辑，最后破解。

对加密的 D-Link 固件进行解密
------------------

无设备情况下的通用思路如下

### 准备工作

这里以 D-Link DIR-822-US 系列路由器 3.15B02 版本的固件为例进行分析。

该固件可以在[官网](https://support.dlink.com/productinfo.aspx?m=dir-822-us)中下载得到

```
wget https://support.dlink.com/resource/PRODUCTS/DIR-822-US/REVC/DIR-822_REVC_FIRMWARE_v3.15B02.zip

```

下载完后，我们如果用 binwalk 去分析固件会发现报告是空白

![](https://bbs.kanxue.com/upload/attach/202403/964693_5GWG5BAQMGN5SRM.webp)

我们这个时候可以用 `binwalk -E` 命令来查看固件的熵值（查看熵值是一种确认给定的字节序列是否压缩或加密的有效手段。熵值越大，意味着字节序列有可能是加密的或者是压缩过的）

![](https://bbs.kanxue.com/upload/attach/202403/964693_KNZE6KFBMV9JQN4.webp)

这里显示熵值几乎都是 1，这意味着这个固件的各个部分都进行了加密

幸运的是，我们发现对应版本固件的发布说明中提到了`The firmware v3.15 must be upgraded from the transitional version of firmware v303WWb04_middle.`

![](https://bbs.kanxue.com/upload/attach/202403/964693_V2U6Q7V4FVCZR7P.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/964693_KB3BVYC2DGMXU5Y.webp)

### 利用过渡版本解密固件

结合之前固件的三种加密更新方式，这个 `firmware v303WWb04_middle` 很可能就含有解密程序的中间过渡用未加密固件。所以我们就可以想办法下载这个固件并找到解密方案，从而解密 `DIR822C1_FW315WWb02` 固件。  
![](https://bbs.kanxue.com/upload/attach/202403/964693_2GQ99H4QT3PA9CK.webp)

这个固件我们可以去 [D-Link 搭建的 FTP 服务器里下载](https://www.dlink.com/uk/en/support/faq/network-storage-and-backup/nas/dns-series/how-do-i-setup-the-built-in-ftp-server)（嫌麻烦可以在附件下载）

该中间过渡用的版本的固件没有加密，可以 `binwalk -ME` 直接提取

![](https://bbs.kanxue.com/upload/attach/202403/964693_R2YXWT8BXVBFWTH.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/964693_J4UHNAJPZHFNSFY.webp)

因为加密固件是由该未加密固件升级而来，所以我们可以在 `squashfs-root` 文件夹内搜索`update`、`firmware`、`upgrade`、`download`等关键的字符串

```
grep -rnw './' -e 'update\|firmware\|upgrade\|download'

```

![](https://bbs.kanxue.com/upload/attach/202403/964693_UHR2G5FWA3U2KYW.webp)

最终可以在 `/etc/templates/hnap/StartFirmwareDownload.php` 文件中找到线索，在浏览器中访问该文件就会执行下载固件的操作，这里有一行注释为 `// fw encimg` ，对应意思就是 `firmware`、`encryption`、`image`

```
// fw encimg
    setattr("/runtime/tmpdevdata/image_sign" ,"get","cat /etc/config/image_sign");
    $image_sign = query("/runtime/tmpdevdata/image_sign");
    fwrite("a", $ShellPath, "encimg -d -i ".$fw_path." -s ".$image_sign." > /dev/console \n");
    del("/runtime/tmpdevdata");

```

前两行代码是用于获取固件映像签名的命令，使用 `setattr` 函数将属性 `/runtime/tmpdevdata/image_sign` 设置为 `cat /etc/config/image_sign` ，再使用 `query` 函数从属性 `/runtime/tmpdevdata/image_sign` 中获取固件映像签名，并将其存储在变量 `$image_sign` 中。

![](https://bbs.kanxue.com/upload/attach/202403/964693_52XEUVBJSZ6B8DC.webp)

即变量 `$image_sign`  将被设置为 `wrgac43s_dlink.2015_dir822c1`

第三行代码是用于执行固件映像解密操作，使用 `fwrite` 函数将命令字符串 `encimg -d -i ".$fw_path." -s ".$image_sign." > /dev/console` 写入文件 `$ShellPath`。后续会执行这个 `shell`

这个 `encimg` 程序位于 `/usr/sbin`

![](https://bbs.kanxue.com/upload/attach/202403/964693_5GVGEQ89ZXAFX6Y.webp)

然后使用 `readelf` 命令可以知道该程序是 32 位大端 MIPS 架构  
![](https://bbs.kanxue.com/upload/attach/202403/964693_254XJ2F29DJDEQD.webp)

现在使用 qemu 用户模式进行模拟，并加上 `encimg -d -i ".$fw_path." -s ".$image_sign.` 等参数

`.$fw_path.` 就是需要解密的加密固件的路径，即 D-Link DIR-822-US 系列路由器 3.15B02 版本固件的路径

`.$image_sign.` 即 `wrgac43s_dlink.2015_dir822c1`

将 `qemu-mips-static` 与加密固件 `DIR822C1_FW315WWb02.bin` 复制到 `squashfs-root` 目录中用于运行 `encimg`

对固件进行解密：

```
sudo chroot . ./qemu-mips-static ./usr/sbin/encimg -d -i ./DIR822C1_FW315WWb02.bin -s wrgac43s_dlink.2015_dir822c1

```

![](https://bbs.kanxue.com/upload/attach/202403/964693_4XZ74V6B39HK85T.webp)

此时再用 `binwalk` 去查看 `DIR822C1_FW315WWb02.bin` ，不仅可以看到文件信息，还发现熵值也变了  
![](https://bbs.kanxue.com/upload/attach/202403/964693_GQTAQ84EGB8TNV8.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/964693_3WXC4GUU55M25NU.webp)

### 提取解密后的固件

此时就可以使用 `binwalk -Me` 成功提取固件系统文件  
![](https://bbs.kanxue.com/upload/attach/202403/964693_UZFHN4UMK49BG33.webp)

![](https://bbs.kanxue.com/upload/attach/202403/964693_TDT6UF36BNA4YEG.webp)

成功提取出加密固件！！！

参考书籍：《物联网安全漏洞挖掘实战》

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 1 天前 被 Arahat0 编辑 ，原因： [#固件分析](forum-128-1-170.htm) [#家用设备](forum-128-1-173.htm)

上传的附件：

*   [DIR822C1_FW303WWb04_i4sa_middle.bin](javascript:void(0)) （6.45MB，2 次下载）