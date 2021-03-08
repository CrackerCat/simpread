> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484844&idx=2&sn=48b6e2a2d782d3d9e42049edc83f58bd&chksm=ce0754e9f970ddfffd8fc555d09441ca5705af786ba99ef1ae5135b680efddc72dd475eaa276&scene=178&cur_album_id=1681422395766538242#rd)

**一、前言**

      在上一篇文章[安卓 10 源码开发定制 (27) 开发内置自定义 Launcher 之开发自己的 Launcher](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484820&idx=2&sn=8606eef31e0ca63801eedfc9e8501b10&chksm=ce0754d1f970ddc7578d22d0813cfd1918f1995618164cd6e12de8bc51e32f5302456fa50a2c&scene=21#wechat_redirect) 已经实现了 **Android Studio** 创建了一个 **Launcher** 的项目，并可以像普通 **App** 一样安装到手机设备并切换作为当前的启动器。本章将介绍移除系统默认的 **Launcher** 模块，使用自定义的 **Launcher** 来替换系统默认的 **Launcher**。      

  

**二、内置自定义 Launcher**

    **（1）、编译自定义的 Launcher 为 MyHome.apk**

  在 **Android Studio** 中配置编译 **App** 的 **build.gradle** 文件中添加如下内容，生成指定名称的 **Apk**。如下所示:

```
    android.applicationVariants.all {
        variant ->
            variant.outputs.all {
                outputFileName = "MyHome.apk"
            }
    }

```

         添加完成之后执行 **build** 生成 **apk**。如下图所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430Qo0NkS9QPJSt1uXqQichsa1tfzt7LaOTrG0B1Pia0cExkAicMtWkl2hC71aiaDibfTS7K8W4Ot5jLmRw/640?wx_fmt=png)

      **(2)、内置 MyHome.apk 到系统**

 关于如何内置 Apk 到安卓系统可以参考如下文章:

 **[玩转 Android10 源码开发定制 (八) 内置 Apk 到系统](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483855&idx=1&sn=06138a9db04ba2a761f8dd9495ecd56a&chksm=ce07508af970d99c1ab1718ea55e5e668029ab68ff438cef2076349faccdb5fbffe14c97cfa3&scene=21#wechat_redirect)**

    **（3）、移除默认 Launcher 模块编译**  
            在上一文章中已经分析到当前安卓源码中默认 **Launcher** 的源码目录位于路径 **packages\apps\Trebuchet**。在该目录下找到模块的 **Android.mk** 配置内容如下:

```
include $(CLEAR_VARS)
LOCAL_USE_AAPT2 := true
LOCAL_MODULE_TAGS := optional
LOCAL_STATIC_ANDROID_LIBRARIES := Launcher3QuickStepLib
LOCAL_PROGUARD_ENABLED := disabled
ifneq (,$(wildcard frameworks/base))
  LOCAL_PRIVATE_PLATFORM_APIS := true
else
  LOCAL_SDK_VERSION := system_current
  LOCAL_MIN_SDK_VERSION := 26
endif
#  这个地方指定生成模块名
LOCAL_PACKAGE_NAME := TrebuchetQuickStep
LOCAL_PRIVILEGED_MODULE := true
LOCAL_PRODUCT_MODULE := true
# 这个地方表示编译的时候覆盖这些模块为TrebuchetQuickStep模块进行编译
# 
LOCAL_OVERRIDES_PACKAGES := Home Launcher2 Launcher3 Launcher3QuickStep
# //...省略
include $(BUILD_PACKAGE)

```

     在 **Android** 系统中，系统主要模块的编译链配置在源码根目录 **build** 目录下配置。所以根据以上的 **Android.mk** 配置，只需要找到存在 "**TrebuchetQuickStep、Home、Launcher2、Launcher3、Launcher3QuickStep**" 任何一个模块的编译链配置，移除掉就行。通过 **build** 目录搜索，在文件路径 **build\make\target\product\handheld_product.mk** 找到了存在 **Launcher3QuickStep** 模块的编译配置链。如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430Qo0NkS9QPJSt1uXqQichsalzMhHVZGX27lU1ZMQHt6scU84uEGVXWEC4VJXWF7r48FZtYjDiaqphg/640?wx_fmt=png)

将以上 **Launcher3QuickStep** 模块从编译链中移除掉。然后 **out** 目录中删除 **TrebuchetQuickStep** 模块生成的缓存。然后编译刷机包验证测试。

**如果你对安卓系统相关的开发学习感兴趣:**

       可加作者的 QQ 群（1017017661), 本群专注安卓系统方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)