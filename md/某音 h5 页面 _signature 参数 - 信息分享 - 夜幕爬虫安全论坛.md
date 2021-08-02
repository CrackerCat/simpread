> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.nightteam.cn](https://bbs.nightteam.cn/thread-407.htm)

> 某音 h5 页面 _signature 参数

* * *

闲来无事看了下某音 h5 页面的  _signature 参数，发现和以前维护的 tiktok 很相似，所以花了两个小时研究了下。留在手里无用，分享给正在研究学习的朋友。本文方法与头条系的大多数站点加密通用.

目标：

```
https://www.iesdouyin.com/share/user/102777167489

```

```
https://www.iesdouyin.com/web/api/v2/aweme/post/?user_id=102777167489&sec_uid=&count=21&max_cursor=0&aid=1128&_signature=xyuuZhAemZEv8YwbDzYVO8crrn&dytk=373c0c83cf5a69b82a5264f3482103d9


```

解决方案：

本文提供一种 nodejs jsdom 生成的方法.

过程

1，定位_signature 参数位置

很简单就搜到了，所以不啰嗦其他技巧了

![](https://bbs.nightteam.cn/upload/attach/202004/1616_J8SZJUERJNCNF7P.png)

2，打个断点定位, 点击_bytedAcrawler.sign 进入, 就看到 vm 中的方法了, 复制出来, 以此为基础修改.

![](https://bbs.nightteam.cn/upload/attach/202004/1616_TZ9JHT99264EGJN.png)

3，先构造个 jsdom & 将 window 对象参数绑定到 global 对象 & 设置个 ua & 首页获取 tac 参数, 这些都是网上教程都有的操作, 我就不细说了.

![](https://bbs.nightteam.cn/upload/attach/202004/1616_EJXWGKS4XHXRCZT.png)

![](https://bbs.nightteam.cn/upload/attach/202004/1616_4JYH9DU8ENNRSHZ.png)

![](https://bbs.nightteam.cn/upload/attach/202004/1616_8TWB4MPBZYXXRBA.png)

![](https://bbs.nightteam.cn/upload/attach/202004/1616_3P8GFTJ74ZGDTVK.png)

该部分可搜索  q$sign  , 找到。

```
[object Object]

```

4，构造完成，运行

![](https://bbs.nightteam.cn/upload/attach/202004/1616_ZQJ586GU2D78ZG9.png)

![](https://bbs.nightteam.cn/upload/attach/202004/1616_NTEJ6EZ3ERAXP7U.png)

5，通过。

后记：

    通过了？？？ 我陷入了沉思，那我后面辛辛苦苦对 cavans 绘制指纹的还原是为了什么呢。。。竟然不检查指纹，完全和 tiktok 不一样! 那就不继续往下分享了, 以后检测了指纹的话再写吧。 删除多余逻辑后的源码放到下面了，回复下再领哦。

您好，本帖含有特定内容，请回复后再查看。