> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Kl1hxnFu4fCTuCk36r9Z9w)

一、安卓源码中 aidl 文件介绍
-----------------

在 Android 系统源码中经常看到很多. aidl 后缀的文件。这些 aidl 文件主要是 binder 进程间通信的接口描述语言。在源码编译的时候才好会通过工具成基于该 .aidl 文件的 IBinder 接口, 并最终打包到系统的 jar 包中。所以在源码中搜索不到具体的 java 实现代码。为了查看 aidl 文件的 java 实现代码，可以有如下两种方式:

**(1)** 通过手机设备提取系统 jar 包。目前高版本的安卓系统中直接提取 jar 包是获取不到具体的实现代码。由于安卓系统编译发布的时候开启了编译优化，会将 dex 转为 oat 文件。所以提取设备的时候就需要提取 jar 包对应的 oat 文件，并用专门的工具将 oat 转化为 dex 文件。然后反编译工具查看。

**(2)** 在源码中编译 userdebug、eng 版本的系统 编译版本 userdebug、eng 版本情况下，jar 包中包含了完整的 dex 文件，可以使用反编译工具查看。

以下以 lineageOs 安卓 10 系统源码编译环境，查看 **ILocationManager.aidl** 文件的实现代码为例说明。

二、jadx 查看 aidl 文件 java 实现代码
---------------------------

**（1）、安卓源码中编译一次 usedebug 版的系统**

**（2）、在源码编译输出目录找到 framework.jar**

**ILocationManager.aidl** 编译之后会打包到 framework.jar 中。安卓源码中 **ILocationManager.aidl** 的文件路径如下:

```
frameworks/base/location/java/android/location/ILocationManager.aidl


```

编译之后 framework.jar 路径如下:

```
out/target/product/oneplus3/system/framework/framework.jar


```

**(3)、打开 jadx 反编译工具，将 framework.jar 拖到工具中**

framework.jar 反编译成功之后, 在工具中打开 android/location/ILocationManager 就可以看到 ILocationManager.aidl 在安卓 10 中的具体实现代码了。如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430xh6QQUaQuJWFc2QXKyPg8xmD9sMy2VllfRCStTLRIAa9R8IiawzrcQ8qT3pgjjkcwq7qia85pYGlw/640?wx_fmt=png)在这里插入图片描述

如果需要下载反编译工具 jadx，可以关注微信公众号 "QDROID88888" 发送消息 "003" 获取下载地址。

[上一篇] [安卓 10 源码开发定制 (21)GPS 定位研究 (1)LocationManager 对象获取流程](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484289&idx=1&sn=75b888e61363e48c0721733771d92a04&chksm=ce0752c4f970dbd22c57c62ebe62ec0cbd941c12b62f4bc95b7fffca90911fa0653d9c757050&scene=21#wechat_redirect)

**安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、刷机交流等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收文章更新。**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430xh6QQUaQuJWFc2QXKyPg8BaTbKm2DTzeDYKTcKbdMeTQ535CCQF9vsbBX38JEUbCw8L76OOI5lQ/640?wx_fmt=png)在这里插入图片描述