> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484033&idx=1&sn=20bd2ce390d418a90ce3f87d2ccf0368&chksm=ce0753c4f970dad27462339b6fddc9d818cd04d4921c8e0d6accb7042b24d839537af5f8027d&scene=21#wechat_redirect)

一、**准备工程源码**  

      从以下网址下载 Riru 工程，下载地址:

```
https://github.com/RikkaApps/Riru

```

    从以下网址下载 Edxposed 工程源码，下载地址:  

```
https://github.com/ElderDrivers/EdXposed

```

   Riru 和 Edxposed 工程源码下载好之后，用 android studio 打开，等待加载完成。  

**二、工程源码编译**

**1.Riru 工程源码编译**

   Riru 工程编译有两种方式。一种是用 gradlew 命令编译; 另一种是使用 android studio 中提供的图形化菜单 task 构建。

 **终端 gradlew 命令编译**

  如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnLXIiaUuBDt09TGkZTGFrdpYwhm0d7NYibsQOaMEB3pLg6Egc3V32pLKRw/640?wx_fmt=png)

从以上可以看出实际使用如下命令:

‍‍

```
gradlew edxp-core:zipSandhookRelease

```

*   **android studio task 方式编译**
    
    如下图所示:
    
    ![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnLb6WiacIiaAl844Q4H1s25N92UziaG00QO28NicOnPia1crhc4emKg69ETUQ/640?wx_fmt=png)
    
    编译成功之后，如下所示:
    
    ![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnL9CRxVbSiag700ia5b5jiapfywCCD6W4iaBLvBzjCtwdX53ssrrLcmiaSP0A/640?wx_fmt=png)
    

**2.Edxpose****d **工程源码编译****  

 **Edxposed** 工程编译有两种方式。一种是用 **gradlew** 命令编译; 另一种是使用 **android studio** 中提供的图形化菜单 **task** 构建。

*    **终端 gradlew 命令编译**
    
      如下图所示:
    

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnLOYQ3Dxic08gqUNy4kwAHrKHzC0OgctKQP2fH0KPLfWGdromsIkqLAGg/640?wx_fmt=png)

从以上可以看出实际使用如下命令:

```
gradlew edxp-core:zipSandhookRelease

```

*   **android studio task 方式编译**
    
    如下图所示:
    

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnLxibWZibFZelIWFwxFXyysCdETW2KRPe0UUuulQib3AS6UrLOd830V2vTw/640?wx_fmt=png)

编译成功之后，刷机的文件存放如下所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnLAyuLKCyia5NwMapMTmbqGUlbY5nWF98JId1BAuEFbyVGpJg6VFtTt9Q/640?wx_fmt=png)

**三. 安装 Riru 和 Edxposed**

可以参考如下文章:  

[Edxposed 学习研究 (一) 手把手教你安装 Edxposed](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484016&idx=1&sn=a2c2bc275c1c7a17a1c1fd2988923001&chksm=ce075335f970da23ff69f81cae607ade411afbeb11d5aaaf703a037b982a55f162e55c028ae4&scene=21#wechat_redirect)  

**专注安卓系统开发定制、安卓 ndk 开发、安卓应用安全开发和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等研究。关注公众号第一时间接收更新文章。**  

**微信搜索公众号 "QDOID88888" 关注或者扫以下二维码关注![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431ma5Kj7G1XZaF7vFcQJAnLHJhjbvHCwtXuvN1zjrvoMCdKqkR3ZA9RGDXx6ScWjJDWP4KcnUaUdw/640?wx_fmt=png):**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)