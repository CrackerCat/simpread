> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271766.htm)

> [原创]Android APP 漏洞之战（8）——插件化漏洞和解压缩漏洞详解

Android APP 漏洞之战（8）——插件动态加载和解压缩漏洞详解
===================================

目录

*   Android APP 漏洞之战（8）——插件动态加载和解压缩漏洞详解
*            [一、前言](#一、前言)
*            [二、基础知识](#二、基础知识)
*                    [1.Dex 文件基本结构](#1.dex文件基本结构)
*                            （1）Dex Header
*                            [（2）索引区](#（2）索引区)
*                            [（3）数据区](#（3）数据区)
*                    2. Zip 文件结构
*                            [（1）源文件数据存储区](#（1）源文件数据存储区)
*                                    <1> file header
*                                    <2>file data
*                                    <3>data descriptor
*                            [（2）中心目录区](#（2）中心目录区)
*                            [（3）中心目录结束标识](#（3）中心目录结束标识)
*                    3.Android APK 签名机制
*                            [（1）应用签名方案类型](#（1）应用签名方案类型)
*                                    <1> 签名方案 v1
*                                    <2> 签名方案 v2
*                                    <3> 签名方案 v3
*                                    <4> 三种签名的比较和校验时机
*                    [4.Android 动态加载](#4.android动态加载)
*            [三、插件化和解压缩安全场景和分类](#三、插件化和解压缩安全场景和分类)
*                    [1. 插件化漏洞的安全场景](#1.插件化漏洞的安全场景)
*                    [2. 插件化漏洞的分类](#2.插件化漏洞的分类)
*                    [3. 解压缩漏洞的安全场景](#3.解压缩漏洞的安全场景)
*                    [4. 签名机制和解压缩漏洞分类](#4.签名机制和解压缩漏洞分类)
*            [四、插件化漏洞原理分析和复现](#四、插件化漏洞原理分析和复现)
*                    [1. 动态加载漏洞](#1.动态加载漏洞)
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）案例 1——动态加载](#（2）案例1——动态加载)
*                            [（3）安全防护](#（3）安全防护)
*                    [2. 签名检验绕过漏洞](#2.签名检验绕过漏洞)
*                            [（1）原理分析](#（1）原理分析)
*                            [（2）案例 2——java 层签名绕过](#（2）案例2——java层签名绕过)
*                            [（3）案例 3——so 层签名绕过](#（3）案例3——so层签名绕过)
*                            [（4）案例 4——在线签名绕过](#（4）案例4——在线签名绕过)
*                            [（5）安全防护](#（5）安全防护)
*            [五、Zip 解压缩漏洞分析和复现](#五、zip解压缩漏洞分析和复现)
*                    [1. 原理分析](#1.原理分析)
*                    [2. 漏洞复现](#2.漏洞复现)
*                    [3. 安全防护](#3.安全防护)
*            [六、Janus 漏洞分析和复现](#六、janus漏洞分析和复现)
*                    [1. 原理分析](#1.原理分析)
*                    [2. 漏洞复现](#2.漏洞复现)
*                    [3. 安全防护](#3.安全防护)
*            [七、实验总结](#七、实验总结)
*            [八、参考文献](#八、参考文献)

[](#一、前言)一、前言
-------------

最近一直处于忙碌的状态，花了很长一段时间，抽出碎片时间才将这篇帖子写完，本文结合上文的动态加载文章一起学习，本文主要讲述 Android 中存在的插件化漏洞、签名机制漏洞、解压缩漏洞等，并对一些经典的漏洞进行了复现，本文的相关实验文件由于太多，后面都会上传到知识星球

 

本文第二节主要讲述 Dex 文件结构、Zip 文件结构、Android 签名机制

 

本文第三节主要讲述插件化漏洞和解压缩漏洞的安全场景

 

本文第四节主要对插件化漏洞进行讲述

 

本文第五节主要对解压缩漏洞进行讲述

 

本文第六节主要对 Janus 漏洞原理进行讲述

[](#二、基础知识)二、基础知识
-----------------

### 1.Dex 文件基本结构

dex 文件是 anroid 虚拟机 Dalik 运行的一种文件，包含应用程序的全部操作指令以及运行时数据，下面我们看下. class 文件和. dex 文件的区别：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_MMYUPGTB4JH5QFU.png)

 

我们可以发现 dex 文件将原来每个文件都有的共有信息合成一体，从而减少了 class 的冗余

 

下面我们进一步详细看 dex 文件结构

 

![](https://bbs.pediy.com/upload/attach/202203/905443_QYJ34G4FRXU49QU.png)

 

我们可以发现 dex 文件主要由 3 大部分组成，分别是：`文件头、索引区、数据区`。其中索引区主要包括字符串、类型、方法、域、方法的索引。数据区主要包括类的定义、数据区、链路数据区

 

![](https://bbs.pediy.com/upload/attach/202203/905443_P6S8BVPP4ZGHA4C.png)

 

上面我们可以看出 Dex 文件由许多部分组成，其中 Dex Header 最为重要，因为 Dex 的其他组成部分，都需要通过 Dex Header 中的索引才能找到

#### （1）Dex Header

dex 文件头一般固定为 0x70 个字节大小，包括标志、版本号、校验码、sha-1 签名以及其他一些方法、类的数量和偏移地址等信息。如图所示：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_C3AATRQS97DFD4D.png)

 

![](https://bbs.pediy.com/upload/attach/202203/905443_GN5B2NYWNP7HS8Z.png)

 

结合上面的两张图进行对照，下面我们进一步详细的描述 dex 文件的结构

 

![](https://bbs.pediy.com/upload/attach/202203/905443_BGHQUBEAFPPYVZH.png)

 

![](https://bbs.pediy.com/upload/attach/202203/905443_Z9P8V29XR35FHR5.png)

#### [](#（2）索引区)（2）索引区

dex 文件索引区主要是对一些字符串、类型、方法、域、方法的索引，方法可以查找到对应的数据位置

 

![](https://bbs.pediy.com/upload/attach/202203/905443_SVFEZWM8T8CYFKM.png)

 

![](https://bbs.pediy.com/upload/attach/202203/905443_UJ7WWCCQDD27WVS.png)

#### [](#（3）数据区)（3）数据区

![](https://bbs.pediy.com/upload/attach/202203/905443_3Z8YMJTZYHEQJ25.png)

 

数据区一般包括类的定义区、数据区、链接数据区。类的定义区一般存放 dex 文件中一些类对象的声明，数据区则存放代码原数据，链接数据区一般提供从索引区到数据区的链接映射关系

### 2. Zip 文件结构

zip 文件是比较常见的压缩文件，我们先来看一下 zip 文件的基本结构图：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_HC9PEC9KB3JR8C5.png)

 

通过图中我们可以看出，zip 文件一般分为三个部分：源文件数据存储区、中心目录区、中心目录结束标识

#### [](#（1）源文件数据存储区)（1）源文件数据存储区

记录着压缩的所有文件的内容信息，其数据组织结构是每个文件都由 local file header、file data、data descriptor 三部分组成

##### <1> file header

用于标识文件的开始，文件结构如下：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_HSK3ZWC8GK8D4C4.png)

##### <2>file data

主要存放相应的压缩文件的源数据

##### <3>data descriptor

一般用于标识该文件压缩结束，该结构只有在相应的 header 中通用标记字段的第３位设为１时才会出现，紧接在压缩文件源数据后。这个数据描述符只用在不能对输出的 ZIP 文件进行检索时使用。例如：在一个不能检索的驱动器（如：磁带机上）上的 ZIP 文件中。如果是磁盘上的 ZIP 文件一般没有这个数据描述符。

 

![](https://bbs.pediy.com/upload/attach/202203/905443_FCDYD7U9V8ETUM7.png)

#### [](#（2）中心目录区)（2）中心目录区

对于待压缩的目录而言，每一个子目录对应一个压缩目录源数据，记录该目录的描述信息。压缩包中所有目录源数据连续存储在整个归档包的最后，这样便于向包中追加新的文件。头部的结构如下：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_RBP4NJR6ZB48UVX.png)

#### [](#（3）中心目录结束标识)（3）中心目录结束标识

目录结束标识存在于整个归档包的结尾，用于标记压缩的目录数据的结束，结构如下：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_MNVKGJZUANUQKN8.png)

### 3.Android APK 签名机制

应用签名主要是避免外部恶意解压、破解或者反编译修改内容，签名的本质是：

```
认证：Android 平台上运行的每个应用都必须有开发者的签名。在安装应用时，软件包管理器会验证 APK 是否已经过适当签名，安装程序会拒绝没有获得签名就尝试安装应用
验证完整性：软件包管理器在安装应用前会验证应用摘要，如果破解者修改了 apk 里的内容，那么摘要就不再匹配，验证失败

```

![](https://bbs.pediy.com/upload/attach/202203/905443_5GPHMQRRE5P3MCY.png)

#### [](#（1）应用签名方案类型)（1）应用签名方案类型

截止到 Android12，Android 支持三种应用签名方案：

```
v1:基于jar签名
v2:提高验证性能&覆盖范围（Android 7.0 Nougat引入）
v3:支持密钥轮换（Android 9.0 Pie引入）

```

为了提高兼容性，必须按照 v1,v2,v3 的先后顺序采用签名方案，低版本平台会忽略高版本的签名方案在 APK 中添加额外数据，具体流程图如下：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_S7RB5GZ3QGUPCB9.png)

##### <1> 签名方案 v1

最基本的签名方案，是基于 Jar 的签名

 

v1 签名后会增加 META-INF 文件夹，其中会有如下三个文件：

<table><thead><tr><th>文件</th><th>描述</th></tr></thead><tbody><tr><td><strong>「MANIFEST.MF」</strong></td><td>记录「apk 中每一个文件对应的摘要」（除了 META-INF 文件夹）</td></tr><tr><td><strong>「*.SF」</strong></td><td>记录「MANIFEST.MF 文件的摘要」和「MANIFEST.MF 中每个数据块的摘要」</td></tr><tr><td><strong>「*.RSA」</strong></td><td>包含了「*.SF 文件的签名」和「包含公钥的开发者证书」</td></tr></tbody></table>

 

v1 签名流程：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_SFE89PRMRFZ48KN.png)

```
（1）计算每个文件的 SHA-1 摘要，进行 BASE64 编码后写入摘要文件，即 MANIFEST.MF 文件；
（2）计算整个 MANIFEST.MF 文件的 SHA-1 摘要，进行 BASE64 编码后写入签名文件，即*.SF 文件；
（3）计算 MANIFEST.MF 文件中每一块摘要的 SHA-1 摘要，进行 BASE64 编码后写入 签名文件，即*.SF 文件；
（4）计算整个 *.SF 文件的数字签名（先摘要再私钥加密）；
（5）将数字签名和 X.509 开发者数字证书（公钥）写入 *.RSA 文件；

```

验证流程：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_HPBGVQF2DXNT9KJ.png)

 

主要包括验证签名、校验完整性两个步骤：

 

步骤 1：验证签名步骤

```
（1）取出*.RSA 中包含的开发者证书，并校验其合法性
（2）用证书中的公钥解密*.RSA中包含的签名
（3）用证书中的公钥计算*.SF的签名
（4）对比（2）和（3）的签名是否一致

```

步骤 2：验证完整性

```
（1）检查 APK 中包含的所有文件，对应的摘要值与 MANIFEST.MF 文件中记录的值一致
（2）使用证书文件（RSA 文件）检验签名文件（SF 文件）没有被修改过
（3）使用签名文件（SF 文件）检验 MF 文件没有被修改过

```

上面任何一个步骤验证失败，则整个 APK 验证失败

 

问题：

```
覆盖范围不足：Zip 文件中部分内容不在验证范围，例如 META-INF 文件夹；
验证性能差：验证程序必须解压所有压缩的条目，这需要花费更多时间和内存
存在Janus漏洞：恶意开发人员可以通过Janus漏洞去绕过Android 的v1签名验证机制

```

##### <2> 签名方案 v2

Android7.0 中开始引入了 APK 签名方案 v2，一种全文件签名方案，该方案能够发现对 APK 的受保护部分进行所有更改，相比 v1 来说校验速度更快，覆盖的范围也更广。但是考虑到版本兼容的问题，所以一般常见了 v1+v2 的混合签名模式

 

我们由上文知道 Zip 文件主体分为：`源文件数据存储区、中心目录区、中心目录结束标识`。EoCD 中记录了中央目录的起始位置，在`源文件数据存储区`和`中心目录区`插入其他数据不会影响 Zip 的解压

 

因此 v2 签名后会在`源文件数据存储区`和`中心目录区`插入 APK 签名分块（APK Signing Block）

 

如下图所示。从左到右边，我们定义为区块 1~4

 

![](https://bbs.pediy.com/upload/attach/202203/905443_5MMDPDDY287B3RH.png)

 

v2 签名块（APK Signing Block）本身又主要分成三部分:

```
SignerData（签名者数据）：主要包括签名者的证书，整个APK完整性校验hash，以及一些必要信息
Signature（签名）：开发者对SignerData部分数据的签名数据
PublicKey（公钥）：用于验签的公钥数据

```

**签名流程：**

 

​ 相比 v1 签名方案，v2 签名方案不再以文件为单位计算摘要，而是以 1MB 为单位将文件拆分为多个连续的快（chunk），每个分区的最后一个快可能会小于 1MB。v2 签名流程如下：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_QUVKGZ6XZEKM6CQ.png)

```
（1）对区块 1、3、4，按照 1MB 大小分割为多个块（chunk）
（2）计算每个块的摘要
（3）计算（2）中所有摘要的签名
（4）添加X.509开发者数字证书（公钥）

```

**验证流程：**

 

![](https://bbs.pediy.com/upload/attach/202203/905443_QC8K74BJE3RFVTP.png)

 

因为 v2 签名机制是在 Android 7.0 上版本才支持，因此对于 Android 7.0 以及以上版本，在安装过程中，如果 v2 签名块，则必须走 v2 签名机制，不能绕过。否则降级走 v1 签名机制

 

v1 和 v2 签名机制是可以同时存在的，其中对于 v1 和 v2 版本同时存在的时候，v1 版本的 META_INF 的 `.SF` 文件属性当中有一个 `X-Android-APK-Signed` 属性：

```
X-Android-APK-Signed: 2

```

v2 签名本身的验证过程：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_T2TJ25PJAJ2NRYZ.png)

```
（1）利用PublicKey解密Signature，得到SignerData的hash明文
（2）计算SignerData的hash值
（3）两个值进行比较，如果相同则认为APK没有被修改过，解析出SignerData中的证书。否则安装失败
（4）如果是第一次安装，直接将证书保存在应用信息中
（5）如果是更新安装，即设备中原来存在这个应用，验证之前的证书是否与本次解析的证书相同。若相同，则安装成功，否则失败

```

##### <3> 签名方案 v3

Android 9.0 中引入了新的签名方式 v3，v3 签名在 v2 的基础上，仍然采用检查整个压缩包的校验方式。不同的是在签名部分增可以添加新的证书（Attr 块）。在这个新块中，会记录我们之前的签名信息以及新的签名信息， 支持密钥轮换，即以密钥转轮的方案，来做签名的替换和升级。这意味着，只要旧签名证书在手，应用能够在 APK 更新过程中更改其签名密钥。

 

v3 签名新增的新块（attr）存储了所有的签名信息，由更小的 Level 块，以链表的形式存储。

 

**签名流程：**

 

v3 版本签名块也分成同样的三部分，与 v2 不同的是在 SignerData 部分，v3 新增了 attr 块，其中是由更小的 level 块组成。每个 level 块中可以存储一个证书信息。前一个 level 块证书验证下一个 level 证书，以此类推。最后一个 level 块的证书，要符合 SignerData 中本身的证书，即用来签名整个 APK 的公钥所属于的证书。从 v2 到 v3 的过渡：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_YPHEZQ7M5QKX9E5.png)

 

**签名校验：**

 

Android 的签名方案的升级都需要确保向下兼容。因此，在引入 v3 方案后会根据 APK 签名方案，v3 -> v2 -> v1 依次尝试验证 APK。而较旧的平台会忽略 v3 签名并尝试 v2 签名，最后才去验证 v1 签名。如下图所示：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_J26D6J8QBT3ZZB2.png)

 

注意：对于覆盖安装的情况，签名校验只支持升级而不支持降级。即一个使用 V1 签名的 Apk，可以使用 V2 签名的 Apk 进行覆盖安装，反之则不允许

 

v3 签名自身的校验：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_K8EM2WQ4KX36GBH.png)

```
（1）利用PublicKey解密Signature，得到SignerData的hash明文
（2）计算SignerData的hash值
（3）两个值进行比较，如果相同则认为APK没有被修改过，解析出SignerData中的证书。否则安装失败
（4）逐个解析出level块证书并验证，并保存为这个应用的历史证书
（5）如果是第一次安装，直接将证书与历史证书一并保存在应用信息中
（6）如果是更新安装，验证之前的证书与历史证书，是否与本次解析的证书或者历史证书中存在相同的证书，其中任意一个证书符合即可安装

```

##### <4> 三种签名的比较和校验时机

v2、v3 的比较如下图所示：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_RCQVT3THPE2U7YZ.png)

```
v1签名方案：基于 Jar 的签名方案，但存在的问题：完整性覆盖范围不足 & 验证性能差
v2签名方案：通过条目内容区、中央目录区之间插入APK 签名分块（APK Signing Block）对v1签名进行了优化
v3签名方案：支持密钥轮换，新增的新块（attr）存储了所有的签名信息，对v2签名进行了优化

```

验证签名的时机主要要了解 Android 安装应用的方式：

```
系统应用安装：开机时完成，没有安装界面
网络下载的应用安装：通过市场应用完成，没有安装界面
ADB工具安装：没有安装界面
第三方应用安装：通过packageinstall.apk应用安装，有安装界面

```

但是其实无论通过哪种方式安装都要通过 PackageManagerService 来完成安装的主要工作，最终在 PMS 中会去验证签名信息，如 v3 验证方式一样

### 4.Android 动态加载

Android 动态加载总会涉及到插件化、热部署、热修复等，这里我在网上查阅资料后，给大家总结了下动态加载的场景使用和分类：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_R5KH6REK7ESFVNC.png)

 

动态加载，就是程序运行时，可以加载外部的可执行文件并运行，这样使得我们可以不用安装 apk 就可以更新应用，针对一些 SDK 项目，可以加快 app 新版本的覆盖率、快速修复线上 bug。这里运行时是指应用冷启动并开始工作后，外部可以是 SD 卡，可以是 data 目录，也可以是 jniLib 目录，这些可执行文件是没有随着应用一起编译的的

 

动态加载的特点：

```
（1）app在运行的时候，可以通过加载一些本身不存在的文件，来实现一定功能，这种经常应用在app更新的过程中
（2）可执行文件是可以替换的，更换静态资源不属于动态加载
（3）动态加载的核心思想就是动态调用外部的dex文件，Android Apk自带的dex是程序入口，所有功能可以直接从服务器中下载dex来完成

```

Android 动态加载按照工作机制不同，可以分为`虚拟机层动态加载`和`Native层动态加载`两大类

 

这里由于本文主要讲解动态加载方面漏洞，所以对热更新、热修复等原理就不深究了，大家感兴趣可以下去查阅相关资料，动态加载原理详细可以参考我上一篇帖子：[Android 加壳脱壳学习（1）——动态加载和类加载机制详解](https://bbs.pediy.com/thread-271538.htm)

[](#三、插件化和解压缩安全场景和分类)三、插件化和解压缩安全场景和分类
-------------------------------------

### 1. 插件化漏洞的安全场景

前文我们知道了 Android 的动态加载机制和签名机制，Android 插件化机制具有模块解耦性，可以动态升级按序加载，而且当下很多 APP 都使用了热部署、热修复、插件化等技术都采用了动态加载技术，这样可以实现 APP 的快速更新，但是也带来一定的安全隐患，使得很多恶意软件能熬过安全检测，来动态加载代码。而执行加载绕过执行漏洞一般与 Android 的签名机制密不可分，所以上文我们也很详细的讲解了 Android 的签名机制。

### 2. 插件化漏洞的分类

很多 APP 通过动态加载一些 dex 或 so 文件，但是考虑到存在动态加载的安全性问题，往往会对加载的文件进行签名校验机制，因此我们可以将插件化漏洞分为两类：动态加载漏洞和签名校验绕过漏洞

 

![](https://bbs.pediy.com/upload/attach/202203/905443_C5EXX8HTWBEH8CY.png)

### 3. 解压缩漏洞的安全场景

Android 中经常会涉及到解压缩问题，比如动态加载机制，可能下载了 apk/zip 文件，然后在本地做解压工作，还有就是一些资源在本地占用 apk 包的太大，就也打包成 zip 放到服务端，使用的时候再下发。Android 在解压 zip 文件，使用的是 ZipInputStream 和 ZipEntry 类，代码比较简单，但是 ZipEntry.getName 的方法存在的漏洞就是返回的是文件名，并没有对特殊字符处理，linux 中`../`可以命令文件但是这个可以进行穿越上层目录，就会带来一定的安全隐患

### 4. 签名机制和解压缩漏洞分类

我们这里列举两个典型的漏洞：如下所示：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_9MST92RTZVBQUVD.png)

[](#四、插件化漏洞原理分析和复现)四、插件化漏洞原理分析和复现
---------------------------------

### 1. 动态加载漏洞

#### [](#（1）原理分析)（1）原理分析

Android 系统提供类加载器 DexClassLoader，可以在运行时动态加载执行包含的 JAR 或 APK 文件内的 DEX 文件，这样可能导致所加载的 Dex 文件被恶意应用替换或代码注入，如果不对 Dex 文件进行签名校验，就可能导致加载的是恶意代码，这样就会进一步造成严重危害

#### [](#（2）案例1——动态加载)（2）案例 1——动态加载

案例准备：

```
原apk
加载dex

```

我们先编写一个测试类文件, 然后生成 dex 文件，这里我们在 dex 文件中只加入字符串信息，我们源 apk 并未加入签名校验机制

 

![](https://bbs.pediy.com/upload/attach/202203/905443_HDNBMYUCH3VNM9Q.png)

 

我们先将 dex 文件放到模拟器的 sdcard / 下

 

![](https://bbs.pediy.com/upload/attach/202203/905443_KWY3TPGY8HXHDSD.png)

 

我们新建一个程序，然后编写主程序的代码，并授权 sd 读取权限

```
Context appContext = this.getApplication();
testDexClassLoader(appContext,"/sdcard/classes.dex");

```

然后我们编写类加载器代码

```
private void testDexClassLoader(Context context, String dexfilepath) {
        //构建文件路径：/data/data/com.emaxple.test02/app_opt_dex，存放优化后的dex,lib库
        File optfile = context.getDir("opt_dex",0);
        File libfile = context.getDir("lib_dex",0);
 
        ClassLoader parentclassloader = MainActivity.class.getClassLoader();
        ClassLoader tmpclassloader = context.getClassLoader();
    //可以为DexClassLoader指定父类加载器
        DexClassLoader dexClassLoader = new DexClassLoader(dexfilepath,optfile.getAbsolutePath(),libfile.getAbsolutePath(),parentclassloader);
 
        Class clazz = null;
        try {
            clazz = dexClassLoader.loadClass("com.example.test.TestClass");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        if(clazz!=null){
            try {
                Method testFuncMethod = clazz.getDeclaredMethod("test02");
                Object obj = clazz.newInstance();
                testFuncMethod.invoke(obj);
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        }
 
    }

```

效果显示：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_CZA78G2ZW9KKYN3.png)

 

这里说明加载成功了，如果我们这里写的是一段恶意代码，这样就会进行攻击，造成破坏

#### [](#（3）安全防护)（3）安全防护

我们上文的动态加载漏洞，是因为源 APK 并未对加载的 dex 文件进行签名校验，从而导致容易导入恶意代码，当然从 Android 4.4 后加入了**对 JAR/DEX 存放目录文件的 user_id 和动态加载 JAR/DEX 的进程的 user_id 是否一致的判断，如果不一致将抛出异常导致加载失败**，这样就很好的可以防范替换加载的 dex 文件，进行恶意注入

 

解决方案：

```
（1）将动态加载的DEX/APK文件放置在APK内部或应用私有目录中
（2）使用加密网络协议https进行下载加载的并将其放置在应用私有目录中
（3）对加载的Dex文件进行完整性校验和签名校验

```

### 2. 签名检验绕过漏洞

#### [](#（1）原理分析)（1）原理分析

我们知道一般对 APK 的验证，主要使用的是签名校验或者 MD5 校验，使用校验的方式较多。而签名校验一般是处理 APK 中动态加载或防止二次重打包的问题。

 

我们可以将 APK 中的签名检验机制进一步进行分类：

```
java层的签名校验：
原理：这种是开发者在APK java层中加入了签名校验代码，然后通过校验加入文件的MD5值或者SHA1值来对文件进行校验
解决方案：一般这种情况，我们通过定位到APK中的签名代码段，然后进行hook 篡改或者进行修改后重打包就可以进行绕过
so层的签名校验：
原理：由于java可解释语言的原因，所以后来开发者又将签名代码放入so层，从而增加逆向工作的难度
解决方案：这种情况，同样可以使用IDA或GDB进行动态调试确定到签名代码段，然后使用hook 注入技术或静态修改来进行绕过
在线签名校验：
原理：由于前两种方式都是静态校验的方式，这样的安全性仍然较低，后来更多厂商通过服务器在线进行验证，将签名密钥发送然后在so层或java层中进行校验
解决方案：这种情况，我们要使用抓包软件对服务器发送的数据包进行抓取，在成功获取正确密钥后，再去hook对应的签名代码段，从而就可以实现绕过

```

这一部分完整性保护大家可以详细的参考看雪陌殇大佬的帖子 [Android 应用完整性保护总结](https://bbs.pediy.com/thread-250990.htm)

#### [](#（2）案例2——java层签名绕过)（2）案例 2——java 层签名绕过

案例：书旗小说. apk

 

我们发现书旗小说在进行重新签名后，再次安装会报错，首先我们 AndroidKiller 解析 APP：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_38G8Y68AB388JGY.png)

 

然后我们开始进行定位，这里我们使用常见的定位点：**signature、killProcess、PageManager**

```
signature、killProcess、PageManager 一般是签名代码的关键函数，所以当我们发现这三个函数同时出现，很大程度代表了签名点

```

我们这里搜索 signature 或 killProcess，我们找到了签名三兄弟：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_X5S7ZFW86K3RNSF.png)

 

![](https://bbs.pediy.com/upload/attach/202203/905443_YQBFZY4P2UQZQNJ.png)

 

分析签名的逻辑，修改后回编译，再安装显示成功

 

![](https://bbs.pediy.com/upload/attach/202203/905443_H6YD9KDSKZ7WDGR.png)

#### [](#（3）案例3——so层签名绕过)（3）案例 3——so 层签名绕过

因为 so 层和 java 层签名绕过原理相近，只是 so 层是分析汇编代码，java 层分析 Smali 源码，这里我们参考一个博主的案例，我列举一下

 

首先我们根据 NDK 注册定位到 so 层的入口点，去查找 JNI_Onload 函数，然后开始去查找上面的三兄弟

 

![](https://bbs.pediy.com/upload/attach/202203/905443_WRURZDRAUVSAF9G.png)

 

![](https://bbs.pediy.com/upload/attach/202203/905443_75CG3EEC2349GHQ.png)

 

这里我们就很好的定位到了代码段，后续就是分析逻辑，修改对应的校验点即可

 

![](https://bbs.pediy.com/upload/attach/202203/905443_82F6SP4983K8YNE.png)

 

最后就会发现签名成功的绕过

#### [](#（4）案例4——在线签名绕过)（4）案例 4——在线签名绕过

在线签名校验主要是抓取校验部分的数据包，然后去查找 cookie 中的 public_key，或者签名 Signature 值，通过分析数据包后再定位到相应的代码段将值回传到相应的代码段即可

 

这里有一个案例大家可以参考 [Android APP 漏洞之战（6）——HTTP/HTTPs 通信漏洞详解](https://bbs.pediy.com/thread-270634.htm#%EF%BC%881%EF%BC%89%E6%BC%8F%E6%B4%9E%E6%A1%88%E4%BE%8B) 中酷狗直播的漏洞实现，这里就是通过在线修改了 MD5 值，然后使得程序在升级过程中绕过了升级校验，从而成功的注入了恶意病毒

 

![](https://bbs.pediy.com/upload/attach/202203/905443_5KSQNK6HABG4PW6.png)

#### [](#（5）安全防护)（5）安全防护

```
（1）java层和so层都可以进一步混淆，来防止字符定位的方法
（2）可以使用反调试技术，来防止动态调试进行定位的方法
（3）可以采用对frida和xposed的检测，来进行防止hook注入
（4）可以尽量采用在线签名，加密传输报文：
客户端将本地程序信息上传到服务端，服务端返回一段校验代码。客户端动态执行代码，返回校验结果
在登陆接口将登录信息在NDK层进行加密，用签名信息进行加密，在登陆接口实现中，进行解密，如果失败不允许登陆

```

[](#五、zip解压缩漏洞分析和复现)五、Zip 解压缩漏洞分析和复现
------------------------------------

### 1. 原理分析

因为 Linux 系统中`../`代表向上级目录跳转，攻击者可以通过构造相应的 Zip 文件，利用多个'../'从而改变 zip 包中某个文件的存放位置，费用该替换掉应用原有的文件，完成目录穿越。这样严重可能会导致任意代码执行漏洞，危害应用用户的设备安全和信息安全

 

`Java` 代码在解压 `zip` 文件时，会使用到 `ZipEntry` 类的 `getName()` 方法，如果 `zip` 文件中包含 `../` 的字符串，该方法返回值会原样返回。如果没有过滤掉 `getName()` 返回值中的 `../` 字符串，继续解压缩操作，就会在其他目录中创建解压的文件

### 2. 漏洞复现

样本：

```
海豚浏览器 V11.4.18
攻击的so文件：libdolphin.so
Poc攻击代码

```

我们打开海豚浏览器，并用 Fiddler 去监控海豚浏览器，Fiddler 的配置大家可以参考我之前博客

 

![](https://bbs.pediy.com/upload/attach/202203/905443_UU4G2T5TF6ZA4AU.png)

 

这里我们可以通过抓包去发现主题下载的申请链接，然后我们将主题下载下来，然后解包查看结构，这里我们重命名为 zip 文件

 

![](https://bbs.pediy.com/upload/attach/202203/905443_UD8HG86QC4UVASB.png)

 

我们可以发现下载下来的三个资源文件，这也说明海豚浏览器的主题本质是一个 zip 包

 

那么我们如何实现 zip 目录穿越了，我们是不是可以尝试去构建一个这样的 zip 包，去替换浏览器的下载包，并重命令去文件名，使得替换浏览器中的关键文件，这里我们就尝试去替换浏览器中的`libdolphin.so`文件。我们先查看该文件的位置：

 

![](https://bbs.pediy.com/upload/attach/202203/905443_3FRYZQKVJEZG2W3.png)

 

此时我们知道了`libdolphin.so`文件的存放位置，目录为：`/data/data/com.dolphin.browser.express.web/files`，这样我们只需要将我们制作的`libdolphin.so`去替换原文件即可

 

我们编写一个`libdolphin.so`文件

 

![](https://bbs.pediy.com/upload/attach/202203/905443_JDWUK4XFQBV7CGX.png)

 

然后我们将生成的 so 文件重新命名`libdolphin.so`文件，接下来我们再使用我们的 Poc 代码更改名称：

```
import zipfile
 
if __name__ == '__main__':
    ZipPath = '../../../../../data/data/com.dolphin.browser.express.web/files/libdolphin.so'
    zp = zipfile.ZipFile('/root/Desktop/zipAttack/attack.zip','w')
    zp.write('/root/Desktop/zipAttack/libdolphin.so',ZipPath)

```

此时我们就成功的构造了我们的攻击文件`attack.zip`

 

![](https://bbs.pediy.com/upload/attach/202203/905443_WE9C2A8RZV27PSC.png)

 

然后我们只需要对海豚浏览器下载主题的包进行劫持替换即可

 

![](https://bbs.pediy.com/upload/attach/202203/905443_Z4J3HE2MZ8D7FMY.png)

 

然后我们再次点击手机下载相应主题，发现主题是成功的下载，但是并没有替换成功

 

经过验证，我们发现首先正常命名的 so 文件是可以正常的和主题一起下载成功的

 

![](https://bbs.pediy.com/upload/attach/202203/905443_6MUNT4GHQT9Q9Y8.png)

 

然后我们验证，Android 中直接重命令文件`../../libdolphin.so`是可以直接回到上级目录的

 

所以综上是因为我测试的 Android6.0 已经打了补丁，在进行解压的时候对`../`这种情况进行了过滤，这样就导致不能进行成功的穿越

 

当然这里我们主要是理解 zip 穿越的原理，这样就可以在很多地方利用这个原理存在的漏洞了

### 3. 安全防护

```
对重要的 zip 压缩包文件进行数字签名校验，校验通过才进行解压
检查 zip 压缩包中使用 ZipEntry.getName() 获取的文件名中是否包含 ../ 或者 .. 字符
更换 zip 解压方式，不使用 ZipEntry.getName() 的方式，使用 ZipInputStream 替代

```

Google 的修复意见：

```
InputStream is = new InputStream(untrustedFileName);
ZipInputStream zis = new ZipInputStream(new BufferedInputStream(is));
while((ZipEntry ze = zis.getNextEntry()) != null) {
  File f = new File(DIR, ze.getName());
  String canonicalPath = f.getCanonicalPath();
  if (!canonicalPath.startsWith(DIR)) {
    // SecurityException
  }
  // Finish unzipping…
}

```

[](#六、janus漏洞分析和复现)六、Janus 漏洞分析和复现
----------------------------------

我们上面已经介绍了签名相关的漏洞、和 Zip 相关的漏洞，下面我们拿 2017 年的典型漏洞 Janus 漏洞进行说明，这个漏洞结合了签名和 Zip、dex 的原理

### 1. 原理分析

相信 Janus 漏洞原理大家已经十分熟悉了，作为 2017 年比较重大的 Android 漏洞，已经有不少的人对其进行了研究和复现，本节只是初步记录下 Janus 漏洞的学习过程和复现思路（Janus 只针对 v1 签名，v2 签名就无效了）

 

Android ART 虚拟机在加载并执行一个文件时，会首先判断这个文件的类型。如果这个文件是一个 Dex 文件，则按 Dex 的格式加载执行，如果是一个 APK 文件，则先抽取 APK 中的 dex 文件，然后再执行。而判断的依据是通过文件的头部魔术字（Magic Code）来判断。如果文件的头部魔术字是 “dex” 则判定该文件为 Dex 文件，如果文件头部的魔术字是 “PK” 则判定该文件为 Apk 文件

 

然而 Android 在安装一个 APK 时会对 APK 进行签名校验，但却直接默认该 APK 就是一个 Zip 文件（并不检查文件头的魔术字），而 ZIP 格式的文件一般都是从尾部先读取，因此只要 ZIP 文件尾部的数据结构没有被破坏，并且在读取过程中只要没有碰到非正常的数据，那么整个读取就不会有任何问题

 

因此，Android 在加载执行代码时，**只认文件头，而安装签名时只认文件尾**

 

这样我们构造一个 APK**，从其头部看是一个 Dex 文件，从其尾部看，是一个 APK 文件**，就可以实施攻击。因此 Janus 漏洞便是将原 APK 中的 classes.dex 抽取出来，改造或替换成攻击者想要执行的 dex，并将这个 dex 和原 APK 文件拼起来，合成一个文件

 

当然我们在构造 apk 时，还需要修改 dex 文件的字段和 zip 文件的字段：

```
dex文件修改DexHeader中的file_size，将其调整为合并后文件的大小
zip文件修改尾部Zip,修正[end of central directory record]中[central directory]的偏移和[central directory]中各[local file header]的偏移

```

漏洞攻击步骤：

```
1. 从设备上取出目标应用的APK文件，并构造用于攻击的DEX文件；
2. 将攻击DEX文件与原APK文件简单拼接为一个新的文件；
3. 修复这个合并后的新文件的ZIP格式部分和DEX格式部分，修复原理如图1所示，需要修复文件格式中的关键偏移值和数据长度值；
最后，将修复后的文件，重命名为APK文件，覆盖安装设备上的原应用即可

```

### 2. 漏洞复现

实验样本：

```
app-release.apk v1签名的初始样本
classes.dex 修改后的dex文件
out.apk 拼接后的apk文件
janus.py 漏洞拼接代码

```

首先，我们通过 Android Studio 编写 apk 文件，并通过 v1 签名生成

 

![](https://bbs.pediy.com/upload/attach/202203/905443_YHFR33BJNE3RK65.png)

 

我们在 Bulid--->Generate Signed APK 中选择通过 v1 签名来生成 apk 文件

 

![](https://bbs.pediy.com/upload/attach/202203/905443_9AK24HWCPXSJYFB.png)

 

我们便得到了 app-release.apk 文件，我们再通过 AndroidKiller 去修改源文件的代码，然后重新打包

 

![](https://bbs.pediy.com/upload/attach/202203/905443_B48W7ZSMVDMPN6Q.png)

 

然后我们提取出生成的 apk 文件中的 classes.dex 文件

 

我们是使用 [Janus.py python2 版本](https://github.com/V-E-O/PoC/blob/master/CVE-2017-13156/janus.py) 和 [java 版本](https://github.com/xyzAsian/Janus-CVE-2017-13156)，这里我们使用 Python 版本

 

![](https://bbs.pediy.com/upload/attach/202203/905443_2T3DCRFPWH5KX6W.png)

 

我们就得到拼接的 out.apk，我们只需要将这个 apk 去覆盖原始的 apk 即可

 

问题：

 

在尝试了几台 Android 6.0 的机子后，并未成功复现漏洞，最后推断很大程度是 Android 系统打了补丁，所以要复现成功可能只能在未打补丁的系统上才行，不过整体来说是一次很好的学习经历

### 3. 安全防护

```
Android7.0后采用了v2签名机制可以有效的抵制Janus漏洞
现在大部分的手机系统已经打上了Janus漏洞的补丁

```

[](#七、实验总结)七、实验总结
-----------------

本文对插件化和解压缩漏洞进行了详细的讲解和漏洞复现，在漏洞复现的过程中，我们发现一个漏洞复现的环境十分重要，因为很多时候曾经的一些典型漏洞都被打了补丁，很难在当下情况复现，当然我们应该更加注重漏洞的原理，从而进行学习，本文可能还存在一些不足之处就请各位大佬指教了。

 

本文的相关实验文件存放在知识星球中，本系列的实验文件后面也会逐一上传到知识星球。

[](#八、参考文献)八、参考文献
-----------------

dex 文件结构：

```
https://juejin.cn/post/6844903847647772686
https://www.jianshu.com/p/b79c729f326b
https://www.jianshu.com/p/f7f0a712ddfe

```

zip 文件结构：

```
http://blog.sina.com.cn/s/blog_4c3591bd0100zzm6.html
https://thismj.cn/2019/02/14/qian-xi-zip-ge-shi/

```

Android APK 签名机制：

```
https://jishuin.proginn.com/p/763bfbd56b8b
https://www.jianshu.com/p/286d2b372334
https://xuanxuanblingbling.github.io/ctf/android/2018/12/30/signature/

```

插件化漏洞原理：

```
https://www.cnblogs.com/goodhacker/p/5152952.html
https://wooyun.js.org/drops/APK%E7%AD%BE%E5%90%8D%E6%A0%A1%E9%AA%8C%E7%BB%95%E8%BF%87.html
https://www.freebuf.com/articles/network/273466.html
https://www.jianshu.com/p/14719d3a508f
https://fiissh.tech/2021/android-fix-zip-path-traversal-vulnerability.html

```

Janus 漏洞原理：

```
https://bbs.pediy.com/thread-223539.htm
https://github.com/tea9/CVE-2017-13156-Janus
https://cert.360.cn/warning/detail?id=d5a609929388cfd84c7e9aa8fb943265

```

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 1 天前 被随风而行 aa 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#漏洞相关](forum-161-1-123.htm) [#源码分析](forum-161-1-127.htm)