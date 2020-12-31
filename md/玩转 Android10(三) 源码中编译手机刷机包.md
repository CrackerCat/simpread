> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484211&idx=1&sn=64269a05b89df9eb57ffb4faa875684d&chksm=cebb467df9cccf6b6b104bf54f8939e2dd160a5046565e4cdf6988113f2d41649c5415a683fc&scene=21#wechat_redirect)

 **说明: 如未特别说明, 本文及以后文章都以手机 oneplus3 为测试设备。**

**一、获取手机设备依赖代码**  

 1. 执行命令下载设备所需配置文件

      源码下载之后，进入源码根目录，如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430HpkFIRvrbTB68PwHwicZh5c8UGG3kNcfosSyl4Ajb7ZaKdmIEB60ga86od4C4Nf7GyeYJdmbWIxg/640?wx_fmt=png)

    在终端源码根目录，输入以下命令完成 oneplus3 手机设备配置文件下载和内核源码下载:  

```
source build/envsetup.sh
breakfast oneplus3

```

如下图所示:  

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh52FeRNuicHiajPgbr4pozxGBhsiaoYRUgTyMf7d940CwkmxquKPumkbNvQ/640?wx_fmt=jpeg)

2. 拉取设备配置文件命令执行完成之后，将手机接入 ubuntu 虚拟机。然后进入源码设备配置文件目录执行提取手机厂商二进制文件脚本 extract-files.sh。如下图所示:  

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5TdojXicYFvzgHylrImbRkicibPPTLTCpibCZzUGNU9lIq4XRvDk6JBtypA/640?wx_fmt=jpeg)

**二、编译设备镜像**

    手机设备编译配置源码以及厂商二进制文件都准备好之后，接下来准备编译设备镜像了。

   在源码根目录分别执行如下命令，完成设备源码编译工作:

```
source build/envsetup.sh
breakfast  oneplus3
brunch oneplus3

```

     执行以上命令之后，耐心等待源码编译完成。完成之后 手机刷机包镜像保存在目录 / home/qiang/lineageAndroid10/out/target/product/oneplus3 中。

**三、刷机测试**  

     刷机包文件编译好之后，可以参照[玩转 Android10(二) 刷机操作之 Recovery 刷机演示](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483690&idx=3&sn=547e5269ede412973a03ba3090848bf9&chksm=ce07506ff970d979f223f29a1b52788d463f2e5bfe443fb3f696f123a3fd73df82738191ba54&scene=21#wechat_redirect)中的方法刷入手机。

**四、可能遇到的问题**  

     1. 执行 breakfast oneplus3 之后, 下载的设备依赖包不完整。

          解决方法: 在设备源码目录中找到 lineage.dependencies 文件，根据文件配置，缺什么就手工下载解压到指定目录。如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5RBey7VBnoopc4clXdamcueRq8BfMQUDUEh7AMGOABMo90dJWPZODOw/640?wx_fmt=jpeg)

     2. 执行 extract-files.sh 脚本 adb 无权限提取手机中的文件。

     解决方法: 下载 lineageOs 官方编译的刷机包，主动给手机刷一次 lineageOs 系统。因为 lineageOs 刷了之后 adb 默认有 root 权限，所以可以 adb 提取到手机设备厂商二进制文件。

     关注我的公众号，第一时间获取文章更新。

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF1oKCbibC1QUXfVLiczlfuiacZpHlgYF8czR6K856p6okMY6HR4TfFNiciaL01RvwH8H3fPpHxNZItOupw/640?wx_fmt=jpeg)