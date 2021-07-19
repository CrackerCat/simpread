> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268485.htm)

> [原创] 某生鲜电商 sign 签名算法分析

磨刀霍霍
====

环境预备
----

### 测试手机

nexus x5 android 6.0

 

(android7.0 以上版本抓包工具默认抓不到 https 请求，因为 7.0 以上只信任系统级别证书，而 charles 证书是安装到用户级目录的。

 

解决方式：可将 charles 证书升级为系统证书，即安装证书到系统证书目录下。

 

具体操作可参考连接：https://www.pianshen.com/article/97291182754/ )

### PC

macbookpro

 

13-inch 4-core 16G-RAM macOS 10.15.5

所需工具
----

### charles - 网络抓包

![](https://bbs.pediy.com/upload/attach/202107/928237_NWN9RC6SEM9MRH8.jpg)

 

下载地址：https://www.charlesproxy.com/

 

(前提：手机和电脑均安装好 charles 证书)

 

证书安装及支持抓包 https 设置指引请参考： https://blog.csdn.net/victory0943/article/details/106332095/

### postman - 接口调试工具

![](https://bbs.pediy.com/upload/attach/202107/928237_YH8X7XARHHVJ9V4.jpg)

 

下载地址：https://www.postman.com/

 

支持导入 cURL，便捷高效，导入操作如下图

 

![](https://bbs.pediy.com/upload/attach/202107/928237_NV5JYU8P9C646HQ.jpg)

### RE 文件管理器 -android 文件导出工具 (需要 root 权限)

![](https://bbs.pediy.com/upload/attach/202107/928237_33MG278NWPQAN4F.jpg)

 

下载地址：https://m-k73-com.sm-tc.cn/c/m.k73.com/mipw/574951.html

### Apk Messenger - 查壳工具

加壳（加固）后的 apk 直接反编译是看不到真实源代码的，需要脱壳后再反编译。

 

使用 APK Messenger 可以很方便地判断是否加了壳

### AndroidCrackTool For Mac - 反编译 apk

https://github.com/Jermic/Android-Crack-Tool

 

mac 下 Android 逆向神器，实用工具集

 

AndroidCrackTool 集成了 Android 开发中常见的一些编译 / 反编译工具, 方便用户对 Apk 进行逆向分析, 提供 Apk 信息查看功能. 目前主要功能包括 (详细使用方法见使用说明):

*   反编译 APK
*   重建 APK
*   签名 APK
*   优化 APK
*   DEX2JAR（APK2JAR）
*   JDGUI
*   提取 DEX
*   提取 XML
*   Class to smail
*   Apk 信息查看
*   Unicode 转换

初窥门径
====

抓包分析
----

### 获取门店接口分析

首先手机配置好代理，打开 APP，用 Charles 抓一下包。

 

![](https://bbs.pediy.com/upload/attach/202107/928237_WJKH6R39MYUK44J.jpg)

一叶障目
====

多次请求获取门店的接口后，复制 cURL 进行对比分析：

 

这里介绍一个文本比对的在线工具，方便观察异同 https://qqe2.com/word/diff

 

![](https://bbs.pediy.com/upload/attach/202107/928237_PTSZCCEF5DRJRYD.jpg)

 

对比分析得出：

 

变化值：

```
longitude和latitude： 经纬度传参，多次定位后肯定取值有所不同
timestamp：时间戳毋庸置疑
signature：顾名思义加密签名

```

相同值：

```
token：多次调用发现token取值是固定的

```

火眼金睛
====

### 加密参数确定

postman 中导入该请求，signature 参数勾选去掉后进行请求，发现返回认证过期无法正常获取数据，说明 signature 为加密参数

 

![](https://bbs.pediy.com/upload/attach/202107/928237_MEVX89VH5FFMUPE.jpg)

 

同时发现 header 中有个 timestamp 时间戳字段，根据逆向经验，请求中同时出现加密和时间戳，一般时间戳都会参与加密运算的，这样后端才会用时间戳结合加密逻辑进行加密认证。

 

因此我们再做个试验，只传 signature 不传 timestamp 进行请求，发现依然无法正确响应，证明我们的猜想的正确的。

 

![](https://bbs.pediy.com/upload/attach/202107/928237_2C22K5Q3RZQ7WUH.jpg)

### 破解目标确定

signature 的加密逻辑

 

且 signature 是有 32 位的数字加小写字母组成的，猜测应该是 MD5 加密

螳臂当车
====

apk 查壳
------

查壳工具很多，这里我使用的是 APK Messenger，打开后，直接将 apk 包拖入界面，幸运地发现该 apk 没有加壳

长枪直入
====

### apk 反编译

本次使用了 Android Crack Tool 这个工具，其实用 jadx、jd-gui 等都可以，平时多换着用用，就知道各个工具的优缺点了~

1.  利用 Android Crack Tool 提取 classes.dex 文件

![](https://bbs.pediy.com/upload/attach/202107/928237_4462EK83NNDGJJX.jpg)

1.  将得到 classes.dex 文件转为 jar

![](https://bbs.pediy.com/upload/attach/202107/928237_XX928YK3A4GHWUJ.jpg)

1.  通过 jd-gui 打开生成的 jar 文件，得到了反编译的源码

![](https://bbs.pediy.com/upload/attach/202107/928237_XUPKTNMJQFYZWQH.jpg)

 

![](https://bbs.pediy.com/upload/attach/202107/928237_B2FT3HY35473EFE.jpg)

真假猴王
====

### 加密逻辑静态分析

**由于加密参数为 signature，在 jd-gui 中全局搜索 “signature”**

 

![](https://bbs.pediy.com/upload/attach/202107/928237_DD63S5RE5H9BWSF.jpg)

 

经过简单的浏览排除后，发现 GlobalHttpHandlerImpl 这个类中的相关代码最为相似，定位到关键代码处：

 

![](https://bbs.pediy.com/upload/attach/202107/928237_KSUR7JAG9F29EE7.jpg)

 

其实这段逻辑很容易看懂：

 

最后一行代码 localObject2 转为 String 后赋值给了 signature，说明 localObject2 是 md5 的加密结果，即倒数第二行 y.a() 方法是 MD5 加密方法，传入 localObject2 经过加密后又赋值给了 localObject2；

 

localObject2 往上追溯，它是一个字符串的拼接，先拼接了一串固定盐值 "CE0BFD14562B68D6964536097A3D3E0C",

 

其次拼接了 str 和 localObject1，那么这两个是什么呢？其实不用向前追溯，我们换个思路往下看，你会发现

 

str 赋值给了 timestamp，localObject1 赋值给了 token，一切都有了答案~

 

一段伪代码进行加密逻辑总结：

```
signature = MD5("CE0BFD14562B68D6964536097A3D3E0C" + timestamp + token)

```

取接口中的 token 和时间戳进行加密验证下，结果是一致的，手工~

 

![](https://bbs.pediy.com/upload/attach/202107/928237_ZEE79MC7XFXH6WK.jpg)

浮沙之上
====

心得：

1.  逆向需要耐心也需要大胆的猜想去不断尝试，同时需要寻求巧妙的验证方式。本例的分析向上和向下的追溯均有，灵活应对
2.  逆向工作会用到的很多好用的工具，平时注意多收集一些好用的工具或博文以事半功倍，本文所用到的工具和相关扩展知识点均贴出了链接，方便读者收藏~
3.  本文旨在分享一些逆向技巧和思路，本文所举 case 相关敏感已打码略去，读者不可利用本文所述内容进行非法商业获取利益，若执意带来的法律责任由读者自行承担。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#逆向分析](forum-161-1-118.htm)