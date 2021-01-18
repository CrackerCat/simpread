> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/yX_i4U8qKXuXYiZYrEr59Q)

  

一、前言
----

       在 Android 系统中, 平时我们开发安装的普通 app 由于权限限制，不能访问系统的一些资源和功能。比如你不能在普通 app 中去杀掉其他应用、开发飞行模式、设置屏幕超时、改变调试模式等等。在系统定制过程中, 如果想要自己开发的 app 有更多的超能力，需要将自己的 app 提升到系统权限。拥有了系统权限后 app 就会和系统的 app"设置" 一样，拥有超能力做很多控制系统相关的事情。

二、开发具有系统权限的 App
---------------

使用 Android Studio 创建工程，然后在 AndroidManifest.xml 文件中添加如下配置:

```
android:sharedUserId="android.uid.system"


```

如下是我个人的一个配置情况:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430FXfZPWYTCmzicFvibB561v54w5xoVVOKQzvceicPP6zj0HyzrWr73h0siadjLSxBksrQccfyJZK08zA/640?wx_fmt=png)

配置好之后就开发所需要的功能，打包成 apk。然后内置到手机。内置 apk 到手机系统参考:

[玩转 Android10 源码开发定制 (八) 内置 Apk 到系统](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483855&idx=1&sn=06138a9db04ba2a761f8dd9495ecd56a&scene=21#wechat_redirect)

三、开发内置过程中的一些注意事项
----------------

**1.** app 工程配置 "android:sharedUserId="android.uid.system"” 之后是不能直接安装到手机测试的，可以先注释掉再安装测试一下在配置打包 apk。

**2.** 内置的时候 Android.mk 中需要配置 LOCAL_CERTIFICATE 签名方式为 platform，不然内置之后 app 不是 system 权限运行的。以下是我的一个配置情况

```
# ///ADD START
# ///ADD END
# 设置当前工作路径
LOCAL_PATH:= $(call my-dir)

# 清除变量值
include $(CLEAR_VARS)
# 生成的模块名称
LOCAL_MODULE := SecurityManager

# 生成的模块类型
LOCAL_MODULE_CLASS := APPS
# 生成的模块后缀名,此处为apk
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
# 设置模块tag，tags取值可以为:user debug eng tests optional
# optional表示全平台编译
LOCAL_MODULE_TAGS := optional

LOCAL_BUILT_MODULE_STEM := package.apk

# 设置源文件
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
# 这个地方非常重要，需要配置为platform平台签名方式
LOCAL_CERTIFICATE := platform
# 此处表示预编译方式
include $(BUILD_PREBUILT)



```

上一篇[玩转 Android10 源码开发定制 (16)LineageOS 中编译 user 模式的系统](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484135&idx=1&sn=9cb4e9288b910fa0cfcdbb62cca058fc&scene=21#wechat_redirect)

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)

扫一扫关注公众号