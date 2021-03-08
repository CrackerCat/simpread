> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484820&idx=2&sn=8606eef31e0ca63801eedfc9e8501b10&chksm=ce0754d1f970ddc7578d22d0813cfd1918f1995618164cd6e12de8bc51e32f5302456fa50a2c&scene=178&cur_album_id=1681422395766538242#rd)

**一、前言**

     **Launcher** 是安卓系统中的桌面启动器，安卓系统的桌面 UI 统称为 **Launcher**。手机在和用户交互的过程中，**Launcher** 为用户提供了各种 **App** 启动的入口。有了 **Launcher** 就可以对 **App** 进行很多操作，比如清理缓存、卸载、权限设置等等。由于系统自带的 **Launcher** 都是功能比较标准的，有时候想通过定制个性化的 **Launcher** 来替换系统的 **Launcher** 来实现一些扩展功能。通过 **Launcher** 可以实现很多特殊的功能。列举如下几个:

*   禁止卸载 **App**
    
*   控制 **App** 是否隐藏 / 显示、启动 / 停止等 (这个功能可以开发出管控手机，防止孩子沉迷游戏)
    
*   控制 **App** 安装将智能机变老年机
    
    以下将以 **lineageOs** 安卓 **10** 系统、**oneplus3** 设备演示说明。  
    

**二、定位查看当前系统的 Launcher 源码**

      在开发自己的 **Launcher** 之前先找到系统 **Launcher** 源码存放地方，后续好屏蔽编译。一般情况下安卓系统的关键 **App** 都存放在 **packages/apps/** 目录下，比如 **Settings**(设置)、**Contacts**(通讯录)、**Camera**(相机) 等。可以按照如下方式查看源码中的 **Launcher** 的源码所在。

  **(1)、当前源码编译刷机之后查看当前 Launcher 顶层 Activity 信息**  

        可以使用 **uiautomatorviewerStrong** 工具获取当前顶层 **Activity** 信息。关于 **uiautomatorviewerStrong** 工具可以参考文章:

[uiautomatorviewer 增强版工具 uiautomatorviewerStrong 使用介绍](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484702&idx=1&sn=a6772a13a3425d3caebc4b34875e60e7&chksm=ce07545bf970dd4d01d09128a9c02fa2f9b24722cd5f3d92913af6aba161bbd7bd20ed33f267&scene=21#wechat_redirect)  

 使用 **uiautomatorviewerStrong** 获取的信息如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0TF5aFBT0NUVLo1U4ZS0MZp9ozlkmnBkoLNVu6ARl5TsA8CdbtEJiciaQ/640?wx_fmt=png)

        也可以通过终端执行 **adb** 命令查看。以下是手机执行情况:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0PMorudwWR6Az7bCH6SWyeG59pgZHQ9oAnhvqic1kp7v2ibIt4Nw0CVCQ/640?wx_fmt=png)

          从以上分析可以知道当前我编译的手机系统的 **Launcher** 的包名为:**com.android.launcher3**。

   **（2）、查看 Launcher 安装的路径名称**  

           执行如下命令查看当前 **Launcher** 的安装路径，可以通过安装路径获取到编译的时候的模块名称。命令如下:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0mL3icTfKebYspeCW1pZnjALevtp0KoLib27Qew9wnyeWduCCNGNnTUKg/640?wx_fmt=png)

          以上输出路径 Launcher 安装路径为:

```
/system/product/priv-app/TrebuchetQuickStep/TrebuchetQuickStep.apk

```

 根据路径可以获取到在安卓源码中编译的模块名称为 **TrebuchetQuickStep**。在源码中通过如下命令定位 **Launcher** 的源码目录。如下图所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0Ahmt6mmTLDcN8CCeHSwNz9GayEVgAOTjTJtOWQ4iaMwiaBfAwD4tA4gQ/640?wx_fmt=png)

    **mgrep** 是安卓源码中提供在 Android.mk 文件执行搜索操作的命令。以上命令搜索可知 **Launcher** 的源码目录为:

```
packages/apps/Trebuchet

```

**三、开源 Launcher 下载编译**

     为了站在巨人的肩膀上快速开发自己的 **Launcher**。在 **github** 上找了一个开源的 **Launcher** 项目 **Lawnchair**。关于从 **github** 快速下载 **Lawnchair** 的方法可以参考文章:

[[经验分享]github 上如何快速下载需要的项目仓库](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484783&idx=2&sn=4f4dff6df7676eb00f6f94569b18ce10&chksm=ce07542af970dd3c5b18105df22ef3461fc96ec8285d161bb452a8fa4313125380fd5f482c79&scene=21#wechat_redirect)。

      **Lawnchair** 项目下载解压之后使用 **Android Studio** 导入工程。导入工程需要很长时间才导入完成。导入之后编译遇到一些坑解决方法记录如下:

*     **编译过程中 gitpack.io 连接失败**
    

      将 **Lawnchair** 工程根目录下 **build.gradle** 中的 "**https://jitpack.io**"改成"**https://www.jitpack.io**"。

*   **关闭签名验证提示**
    

     在源码文件路径:

```
Lawnchair\lawnchair\src\ch\deletescape\lawnchair\LawnchairLauncher.kt

```

中将方法 **verifySignature** 永远返回 **true**。如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0icaQ5WUdMp4m0ZceQV2RjLOqzicrvnlgiaaVqpiajHxBQ9XaUgdcXbqg2w/640?wx_fmt=png)

   以上修改完成之后直接编译运行安装到手机即可。

**四、效果展示**

   **图一按 Home 键弹出 Launcher 选择器**:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0lLBvTTv0N6MnXj6tdXxtwmUtd3sahmzN9Q1l3049iaBYjsJRXMlHRdA/640?wx_fmt=png)

**图二系统自带的 Launcher 界面:**

![图片](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0Y8wF31QWFrIqGDES7nDe9jbqCwicSiaI9ibDbVCFMic2icFib0dlDoPFRMtg/640?wx_fmt=png)

**图三 Lawnchair 启动器效果:**

![图片](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432enOzccPksbibTmEaItThe0CQvzMOvmficDLvicPKZUBJojCiapTiaibHBPAKicFRgvSBjOq9d90j31a3Uw/640?wx_fmt=png)

**如果你对安卓系统相关的开发学习感兴趣:**

       可加作者的 QQ 群（1017017661), 本群专注安卓系统方面的技术，欢迎加群技术交流。  

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** | 获取更多文章信息**** 

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)