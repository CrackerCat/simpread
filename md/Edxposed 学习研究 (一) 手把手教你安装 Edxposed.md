> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484016&idx=1&sn=a2c2bc275c1c7a17a1c1fd2988923001&chksm=ce075335f970da23ff69f81cae607ade411afbeb11d5aaaf703a037b982a55f162e55c028ae4&scene=21#wechat_redirect)

一、前期准备  

---------

**测试手机设备准备环境:**

```
手机型号:oneplus3
系统版本:Android 10
手机提前刷入twrp recovery镜像。


```

关于如何刷 twrp 可以参考如下文章: [玩转 Android10 源码开发定制 (二) 刷机操作之 Recovery 刷机演示](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483690&idx=3&sn=547e5269ede412973a03ba3090848bf9&scene=21#wechat_redirect)

**Magisk 刷机包准备:**

Magisk 下载地址如下:

```
https://github.com/topjohnwu/Magisk/releases


```

此处我选择 Magisk v21.1 版本, 下载地址:

```
https://github.com/topjohnwu/Magisk/releases/tag/v21.1


```

**riru-core 刷机包准备:**

此处下载 v21 版本的 riru-core，下载地址:

```
https://github.com/yangzhaofeng/riru-core


```

说明: riru-core v22 之前的版本使用替换系统库 libmemtrack.so 的方式实现注入 zygote;v22 及以后采用设置 ro.dalvik.vm.native.bridge=libriruloader.so 的方式实现注入 zygote。

**Edxposed 刷机包准备:**

Edxposed 下载地址如下:

```
https://github.com/ElderDrivers/EdXposed/releases


```

此处我选择 Edxposed v0.4.6.4 sandhook 版本, 下载地址:

```
https://github.com/ElderDrivers/EdXposed/releases/tag/v0.4.6.4


```

说明: 由于最新版本的 Edxposed 的要求 Magisk v21+,Riru v23+。所有我下载低版本的才能匹配上我的 Magisk 版本和 riru-core。

**EdxposedManager 安装包准备:**

下载地址如下:

```
https://github.com/ElderDrivers/EdXposedManager/releases


```

二、安装 Magisk
-----------

1.  **将 Magisk、riru-core、Edxposed-sandhook 包放到外置**
    

参考命令如下

```
C:\Users\Qiang>adb push E:\studyspace\Android10\magisk\Magisk-v21.1.zip /sdcard/Magisk-v21.1.zip
E:\studyspace\Android10\magisk\Magisk... 138.1 MB/s (6135789 bytes in 0.042s)

C:\Users\Qiang>adb push E:\studyspace\Android10\magisk\riru-core-v21.zip /sdcard/riru-core-v21.zip
E:\studyspace\Android10\magisk\riru-c.... 696.5 MB/s (521036 bytes in 0.001s)

C:\Users\Qiang>adb push E:\studyspace\Android10\magisk\EdXposed-SandHook-v0.4.6.4.4563.-release.zip  /sdcard/EdXposed-SandHook.zip
E:\studyspace\Android10\magisk\EdXpos...1160.9 MB/s (3092528 bytes in 0.003s)



```

2.  **手机进入 twrp Recovery**
    

可以用如下 adb 命令进入 twrp recovery:

```
adb reboot recovery


```

（1）、**进入 recovery 之后的主界面如下:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYjOPEUYpibLrTYgm6kZdohqZiaZEdT40gZMD4Y1fP42N3wmG4EXPCKgSw/640?wx_fmt=jpeg)

(2)、**在 recovery 主界面依次操作 "Wipe->Advanced Wipe", 然后选中除 "Internal Storage" 以外的选项完成后点击 "Wipe" 完成刷机之前的清理操作。如下图所示:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYibEaeHGRbQy9aFlQMcyHdV26iczkH77e1tYllmhoojgGGiaBFQKaNJgicA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYWwZ6Z2MpmmkEdXHfXZ2icLfs09y01D0Sz0ydE8HNcM9xpcsZOar992Q/640?wx_fmt=jpeg)

(3)、**回到 recovery 主界面之后, 点击 "Install" 进去选中 Magisk-v21.1.zip 包刷机，刷完之后重启手机。如下所示:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYMB1C3tcI6pwMqJLobe6RgrbMQKEX6NrBicsPm1eVLCxyl9c6k1mhYvA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYbK89ibP7snZFWzB3FepPhwlHBmByAbHicHUat1pCFKEicJtJFicCHJKmfg/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYxWUkHmNfHe4SiaKpOodwSCJSPvdbZKv1vo8NHnZzvwAOYkZa8ic1KSQA/640?wx_fmt=jpeg)

三、在 Magsik 中安装 riru 和 Edxposed  

---------------------------------

Magisk 安装好重启手机之后，会默认安装一个 "Magisk Manager" 的 App。打开该 app，进入主界面，准备安卓 riru-core 和 edxpsed-sandhook。详细安装过程如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYjno0ibQicth7PSWwSNKO0lmgeQ5mOa6ibIkRoKpD7EicksbsiaPYwq64uww/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUY1ic6MJnQuwYic7niaFzsThyHEsFAtJpz5OERSMEwZMzXSXOsVtklPNXMg/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYN1ib6AkIN4icyZTjiaWA0CCsv55iav9JIMG7xcSXymhBwRSO1vPxOnVCKw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYQn0ic7wqh7wVACBT50uZVoQ0XlGXNKoSSpxw4FereUuHGMnwkQx1n1w/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYS0NqQlGrVwSKcBVSyK0xpBcVwYKIN1CUuP9b6JWpO8pSpFPeLv6k5w/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYFYcdSXOLm5eyzMcqqMNZ4L3TU7Nk4kjFeI7bvXPzU2eibOylkAKbtPA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYp5BTDJicwYoGBJS4cORZmYn7bbGB3YQrymblPDUdECdSuYweZl8sM9A/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYDrGLxZC8D9YtIPBKOIGn6KzA1powELiawyLia49IPfjicRWhgiaqvsjEkg/640?wx_fmt=jpeg)

  
安装完 riru-core 和 Edxposed 重启手机之后，安装 EdXposedManager app，打开验证是否成功。成功如下所示:![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYH9Z2JZ2rGqtxW7scgibhReicuDrOfJwhiayj1n3ZYLTWLoCS0boKFHCxA/640?wx_fmt=jpeg)

**专注安卓系统开发定制、安卓 ndk 开发、安卓应用安全开发和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳、安卓行为分析沙箱、安卓 app 检测等等。关注公众号第一时间接收更新文章。**

**微信搜索公众号 "QDOID88888" 关注或者扫以下二维码关注:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430j2yXQZGT25Gsdbfs72CUYdIn4BjK4Ke09tarLFlbAuRc7WPPZaLITfnVvnvjdfTPOwicGytNYtFw/640?wx_fmt=jpeg)