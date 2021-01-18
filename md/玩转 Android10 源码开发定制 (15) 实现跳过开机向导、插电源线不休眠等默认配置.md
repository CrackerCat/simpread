> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/gmF5FdlQc4MoMiHi8rXpjw)

一、前言
----

在刷机玩机过程中, 常常遇到刷机之后烦人的开机引导设置。特别是有强迫症的人，多希望开机之后就跳转到主界面。经过研究了一下，可以通过修改安卓源码中的默认设置跳过开机引导，此外还有很多其他功能，比如是否打开蓝牙、锁屏等等功能都可以通过默认配置进行修改。

二、安卓系统默认配置设置介绍
--------------

安卓源码中默认属性配置存放路径如下:

```
frameworks/base/packages/SettingsProvider/res/values/defaults.xml


```

该文件中有很多系统初始化的配置信息，以下列举几个:

```
<!--默认是否打开蓝牙-->
 <bool >true</bool>
<!--默认是打开安装非应用市场的app-->
<bool >false</bool>
<!--默认是否开启包验证-->
<bool >true</bool>
<!--默认是否开启数据线连接电源情况下不休眠-->
<bool >false</bool>
<!--本文的关键属性======默认是否开启跳过开机向导-->
<bool >false</bool>


```

从以上属性看 defaults.xml 中绝大多数属性的值要么 false，要么 true，修改起来非常方便。

三、修改默认属性实战
----------

我们将以上列举的属性值 true 改为 false,false 改为 true。如下:

```
<!--关闭蓝牙-->
 <bool >false</bool>
<!--允许-->
<bool >true</bool>
<!--关闭包验证-->
<bool >true</bool>
<!--连接电源情况下不休眠-->
<bool >true</bool>
<!--本文的关键属性======开启跳过开机向导-->
<bool >true</bool>


```

修改以上属性完成之后编译。双清手机刷机，可以看到修改的属性生效，比如开机之后直接进入主界面了。

[**上一篇**] [玩转 Android10 源码开发定制 (14) 修改安卓源码手机永不休眠](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484090&idx=1&sn=85c10fa5435e82565df37a26f664b911&scene=21#wechat_redirect)

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)

扫一扫关注公众号