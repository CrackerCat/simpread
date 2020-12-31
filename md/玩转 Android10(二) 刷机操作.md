> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484206&idx=1&sn=c7ed81fcee32ced399e089f1d4cbb9fb&chksm=cebb4660f9cccf7668c37179b0cb7e6bd5e29455b20e4449ba3be22f27f481d367d6327cd48a&scene=21#wechat_redirect)

一、刷机方式概览
========

     本文只讨论的刷机模式只针对 lineageOs 编译的系统进行刷机。Android 刷机方式主要分为线刷方式和 recovery 模式刷机。  
_**线刷方式**_:  
    该方式需要手机处于 fastboot 模式。Fastboot 是一种底层刷机模式，该模式下可以刷入基带、bootloader 以及 Android 系统的各个分区镜像。如果手机已经无法开机了，也可以通过进入 fastboot 模式进行线刷救活。  
_**recovery 模式刷机**_:  
    该方式需要刷入第三方 recovery，需要手机进入 recovery 模式。recovery 模式下可以进行系统备份或升级、恢复出厂设置等操作。

二、刷机相关常用命令说明
============

_**1.adb 命令相关**_  
（1）adb 显示当前连接电脑的手机设备列表

> adb devices

(2) 手机进入 bootloader 模式

> adb reboot bootloader

(3) 手机进入 recovery 模式

> adb reboot recovery

(4) 使用 adb sideload 模式安装刷机包, 需要配合 recovery 使用

> adb sideload xx.zip

_**2.fastboot 命令相关**_  
(1) 设备解锁操作

> fastboot  flashing  unlock

(2) 刷入 boot 分区. 如果修改了 kernel 代码, 刷该分区生效。  

> fastboot  flash  boot  boot.img

(3) 刷入 recovery 分区. 现在常用刷入第三方 recovery 来进行刷机操作，比如使用 twrp。

> fastboot  flash  recovery  recovery.img

（4）刷入 system 分区, 系统定制过程中，绝大部分都被编译到 system.img 中  

> fastboot  flash  system  system.img

(5) 刷入 bootloader  

> fastboot  flash  bootloader  bootloader.img    

(6) 直接刷入进行启动 recovery 模式

> fastboot boot <recovery_filename>.img

(6) 设备上锁

> fastboot  flashing lock

三、刷机演示准备
========

    接下来，将演示线刷方式和 recovery 模式两种刷机。

_**线刷方式**_:  

    由于线刷方式需要官方对应 Android 系统版本的工厂镜像底包, 很多型号的手机镜像底包有点难找或者没有，所以这个方式的刷机主要用在 google 系列手机，比如 nexus 5、pixel 2 等等。本文将采用 pixel 2 手机演示。

_**recovery 模式刷机**_:  
    本文将采用 oneplus3 手机，刷入第三方 twrp recovery 进行卡刷演示。

关注下面的公众号，获取更多相关文章:  

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF3FUeiaXU7G3N3DpMgKhWnsYJo371deoRZibcibTtz5xL6GphibDxAbBIHyxn5nTk2zHIsFGPxhTdlQ5w/640?wx_fmt=jpeg)