> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/m6yLK4bbvUE5PiCZyfFP6w)

**一、前言**

  

     最近在 **lineageOS** 官网下载**小米 5s plus** 手机的刷机包刷机开机之后连接 **wifi**。**wifi** 图标显示叉号提示 “**网络连接受限**”。但是实际上能上网。如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433ib7qaSYPialcA7jm6om8Yut5HZymWHcrLQBp3VPpUMQCx3gEvqQT4qAbKyf88VsDX1giaMETVdNPPA/640?wx_fmt=png)

        刚出现这个叉号还以为手机不能上网，怀疑刷机包有问题。然后下载了魔趣 **10.0** 的刷机包验证一下，刷进去之后连接 **wifi** 没有显示任何符号。通过网上搜索查看相关资料，找到了原因和解决方法。  

**二、原因分析**

      在安卓 5.0 系统之后引入了网络验证机制。当你连上网络后，移动网络与 WIFI 网络都会去连接到 **google** 的服务器上然后给目标产生 **204** 响应的服务器发送给一个请求，如果服务器返回的是状态码为 **204** 的响应，那么就被认为网络可以访问；如果返回的是其他状态码，将被视为网络访问需要登录操作；如果没有响应的话，就被认为是网络不可访问。所以手机出现叉号表示系统判定网络不可用。在以上刷机包中 **lineageOs** 刷机包由于比较接近原生并且国外团队维护的，所以网络验证机制的服务器是国外的，国内用户当然是不能访问的了。测试魔趣的可以，那是因为魔趣是国内团队在维护，魔趣可能关闭了网络验证机制或者修改了 **204** 响应服务器为国内的了。  

**三、解决方法**

    可以通过以下三种方法解决。  

**1. 修改 204 验证服务器**  

 手机 adb 连接电脑，使用如下命令设置修改 **204 验证**服务器。

```
adb shell settings put global captive_portal_https_url  https://captive.v2ex.co/generate_204

```

    以上 **204** 验证服务器网上找的，验证可以用。设置以后飞行模式以下。效果如下:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433ib7qaSYPialcA7jm6om8YutZ9q7FF2luF9D5CaTGv001k4fXJPcYo69kwfxIKVaR1jQXRWibzHrCFg/640?wx_fmt=png)

**2. 关闭 Captive Portal**

      执行如下命令关闭 **Captive Portral 204** 服务器验证。命令如下所示:  

```
adb shell settings put global captive_portal_mode 0

```

**3. 源码中关闭或者设置 204 验证服务器彻底解决**

      对于系统开发而言，最直接的方式就是系统源码里面添加代码。彻底解决。在以后重新刷包或者恢复出厂设置以后，再也不会出现感叹号或者叉号的提示。源码中添加如下代码，可以实现手机启动的时候关闭验证或者修改验证服务器。

     **（1)、找到 ConnectivityService 类**

              **ConnectivityService** 类源码路径位于:  

```
frameworks\base\services\core\java\com\android\server\ConnectivityService.java

```

     **（2）、在 **ConnectivityService** 构造函数中添加关闭验证或者修改服务器地址的功能代码**

```
  public ConnectivityService(Context context, INetworkManagementService netManager,
            INetworkStatsService statsService, INetworkPolicyManager policyManager) {
        this(context, netManager, statsService, policyManager,
            getDnsResolver(), new IpConnectivityLog(), NetdService.getInstance());
    }
    @VisibleForTesting
    protected ConnectivityService(Context context, INetworkManagementService netManager,
            INetworkStatsService statsService, INetworkPolicyManager policyManager,
            IDnsResolver dnsresolver, IpConnectivityLog logger, INetd netd) {
            ...
              //关闭验证
             Settings.Global.putInt(mContext.getContentResolver(),Settings.Global.CAPTIVE_PORTAL_MODE,0);
             //修改验证服务器地址
             //Settings.Global.putString(mContext.getContentResolver(),Settings.CAPTIVE_PORTAL_HTTPS_URL,"https://captive.v2ex.co/generate_204");
           ...
    }

```

**如果你对安卓相关的开发学习感兴趣:**

       可加作者的 QQ 群（1017017661), 本群专注安卓方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)