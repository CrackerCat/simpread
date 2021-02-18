> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/mcxwFAb7ZZf0XpLFQI2wKA)

**前言**

      在系统源码中增加系统服务之后，接下来准备 Android Studio 编写 App 进行测试验证。  

      编写测试 App 之后，请先编译系统刷一次机。不然编写的 app 在其他手机系统无法正常运行。

**开发 App 测试验证  
**

**1. 首先创建测试** **App** **工程 "****GetWifiMacServiceTest****"。**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430YspJVjyFpxpSfJKRfW89vEvw0E88npMZ2yKz1shhwEU3B24GA1HTicJtP7nb17z2D6fUWppeGAcw/640?wx_fmt=png)

**2. 复制源码编译输出目录中的 framework 层的 jar 包到工程中**

   源码编译之后可以在以下路径找到 framework 层生成 jar 包。路径如下:  

> **/home/qiang/lineageOs/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar**

   将 classes.jar 拷贝放到工程中。如下图所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430YspJVjyFpxpSfJKRfW89vZnaUic4qS89SmWqoicWLJOwIaIociadic48JBSCvudG8XbsAOzMF6ib3ScQ/640?wx_fmt=png)

**3. 配置** ****GetWifiMacServiceTest**** **工程使用** **classes.jar** **作为编译** **sdk**

    **GetWifiMacServiceTest** 工程添加 "**classes.jar**" 之后会和安卓 **sdk** 中的 **android.jar 冲突。**为了让工程编译的时候用的是 classes.jar，需要配置进行额外的配置一下。配置流程如下:  

**图 1:**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430YspJVjyFpxpSfJKRfW89vDeyssLPHWSvzEtJAPczMu2zhIXXysJGRe3A2gic3nYBj1iauLWGTCrzQ/640?wx_fmt=png)

**图 2：**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430YspJVjyFpxpSfJKRfW89vgwUVnDibCiacFeOkwvwDoQN2rbe6LOibEXXjic9qOu0dBkGIwgeyuEzeXg/640?wx_fmt=png)

**4. 编写测试代码**

  核心关键代码如下:

```
//直接使用GetWifiMacServiceManager
private static String getWifiMac2(Context context)
{
        GetWifiMacServiceManager getWifiMacServiceManager=(GetWifiMacServiceManager)context.getSystemService(Context.GET_WIFI_MAC_SERVICE);
        return  getWifiMacServiceManager.getWifiMac();
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        log("getWifiMac  "+getWifiMac2(this));
    }

```

**5. 编译工程然后安装 app 到手机测试**

 测试效果如下:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430YspJVjyFpxpSfJKRfW89vDzsvibA56BqcMO4lmr21FpDK8VTIEECSQCNANrG7hE8szzbV4O8lHfw/640?wx_fmt=png)

 _**如果需要** framework 层添加自定义系统服务**的 Android Studio 工程，请到公众号后台资源专区中下载。**_  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)