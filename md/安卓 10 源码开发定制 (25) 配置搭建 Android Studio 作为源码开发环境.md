> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484552&idx=1&sn=9c1c134b102f8acba7249539379a8201&chksm=ce0755cdf970dcdb15cb6a29685cf9d8d88cb72235ba9ac5552c7d9481bb1bf74e5e87e418d5&scene=178&cur_album_id=1681422395766538242#rd)

**前言**

  

      之前一直都是用 **SourceInsight** 工具来阅读修改 **Android** 源码。**SourceInsight** 阅读源码比较不错，但是一直没找到方法配置代码提示功能。平时开发 **Android app** 或者 **jni** 的时候体验到了 **Android Studio** 强大的代码智能提示，所以就想通过参考网上的一些教程亲自配置使用 **Android Studio** 作为源码阅读开发环境。

**软硬件环境**

    ** 配置基础环境信息如下所示:**

<table><tbody><tr><td width="179.66666666666666" valign="top"><strong>&nbsp; &nbsp;环境清单</strong><br></td><td width="296.6666666666667" valign="top"><strong>参数信息</strong><br></td></tr><tr><td width="179.66666666666666" valign="top"><section><strong>&nbsp; &nbsp;电脑操作系统</strong><br></section></td><td width="297.6666666666667" valign="top">Windows&nbsp;10<br></td></tr><tr><td valign="top" colspan="1" rowspan="1" width="178.66666666666666"><strong>&nbsp; &nbsp;处理器</strong><br></td><td valign="top" colspan="1" rowspan="1" width="299.6666666666667"><p>&nbsp;&nbsp;i5</p></td></tr><tr><td valign="top" colspan="1" rowspan="1" width="178.66666666666666">&nbsp; &nbsp;<strong>内存</strong><br></td><td valign="top" colspan="1" rowspan="1" width="301.6666666666667">32GB<br></td></tr><tr><td valign="top" colspan="1" rowspan="1" width="178.66666666666666"><strong>&nbsp;&nbsp;虚拟机环境</strong><br></td><td valign="top" colspan="1" rowspan="1" width="302.6666666666667">VMware Workstation 15 Player+Ubuntu&nbsp;20(虚拟机硬盘&gt;=230G）</td></tr><tr><td valign="top" colspan="1" rowspan="1" width="178.66666666666666"><strong>Android Studio 版本</strong><br></td><td valign="top" colspan="1" rowspan="1" width="303.6666666666667">4.1.1<br></td></tr><tr><td valign="top" colspan="1" rowspan="1"><strong>&nbsp; &nbsp; 安卓源码版本</strong><br></td><td valign="top" colspan="1" rowspan="1">安卓 10 版本的 lineageOs 源码&nbsp;<br></td></tr></tbody></table>

  **说明:**

*   **现在的安卓系统源码下载编译需要很大的系统硬盘，如果创建虚拟机的时候建议大于 200G 以上**
    
*   **由于本人采用的是虚拟机方式，对内存要求很高, 之前配置 24G 都感觉吃力，后面又升级为 32G 的内存才感觉勉强能双系统工作。**  
    

**详细配置教程**

1.  **先编译一次源码**  
    
2.   **生成 IDE 相关文件**
    
     在源码根目录执行如下命令，生成 Android Studio 导入源码工程需要的文件。如下所示:
    
    ```
    qiang@ubuntu:~/lineageOs$ source build/envsetup.sh 
    qiang@ubuntu:~/lineageOs$ mmm development/tools/idegen/
    qiang@ubuntu:~/lineageOs$ ./development/tools/idegen/idegen.sh
    
    ```
    
    执行以上命令之后，会在源码根目录生成如下文件。如下图所:
    
    ![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDCGN1zUTzJT5nXyZJVUpQFyhM58tEHzP0RaABZSGLp5FVpgQ5ianwvzA/640?wx_fmt=png)
    

**3.Android Studio 导入源码**  

      由于我 **Android Studio** 是运行在 **Windows** 系统端，需要导入虚拟机 **Ubuntu** 系统中的安卓源码。所以需要将虚拟机源码共享出来供 **Windows** 系统访问。如何共享虚拟机里面的安卓源码供 **Windows** 系统访问，可以参考文章:

[玩转 Android10(四) 源码开发环境搭建。](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483826&idx=1&sn=c3398f87832db6550fc8c1123cefff84&chksm=ce0750f7f970d9e186174e001b52622238aca6bc7f982b1a2e1eab1ec1904441e35489da5208&scene=21#wechat_redirect)  

    **（1）、通过 Android Studio 打开源码根目录中的 "android.ipr" 文件完成源码导入。导入有点慢，需要耐心等待。如下图所示:**

 **图 1：**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDFRvqpsVkdKLugJRbHNIuSg37mtjxRdTA1IYiazLCrwQ3OsSnVocZRhA/640?wx_fmt=png)

**图 2：**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDJuKgkN9ECDUuttibT2vWx2eiadvVGZzawsUkHdAwSDC97hzNSicuIX44Q/640?wx_fmt=png)

**（2）、添加 / 移除源码工程中的模块**  

   Android Studio 首次导入完成之后, 如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFD8HxPNpGEmeOYR8Sdib9Sh4GWojia7SSxDG7TNJDUcMR6NyLYMh7sQ0bw/640?wx_fmt=png)

       由于安卓源码中的模块太多，默认会导入很多模块。源码根目录中的 "**android.iml**"文件记录了导入的模块配置信息。该文件中"**excludeFolder**" 节点表示该模块不导入。如下所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDLic9D8B7ib7IF9WeUZkxX4yoaWlNfzS8fjotIVtgiaYyCkHeU18IoG4vQ/640?wx_fmt=png)

    可以通过 **Android Studio** 中的工程模块管理进行模块多余的模块异常，提升 **Android Studio** 的运行速度。如下操作所示:  

**图 1：**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDN03YbM9QdnNFibYvWWoQHse8GcB2G3yZ9Q1Mvy1Gwuyic8vyibsW67DlQ/640?wx_fmt=png)

**图 2：**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDKnrowTQ93AntvoVM1jyOWtSeQibHCqJ6gMQ6mnlZK6OngwCURujcDRg/640?wx_fmt=png)

**图 3 配置成功之后:**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDytozuotg5ic7YSsqVg1o7B6SITrLfRWpNyNOHGXUawHtGtldlPou2ZA/640?wx_fmt=png)

**图 4 编写代码验证智能提示功能:**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432um0MZby1KCEOD88JzQmFDKiagyHK6zHwox0HKKelY8HZibEqO6KozIlEEWFibPA1iaJM5icxXEskjic8A/640?wx_fmt=png)

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** | 查看更多相关开发知识****  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)