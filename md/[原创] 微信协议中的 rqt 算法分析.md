> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268087.htm)

> [原创] 微信协议中的 rqt 算法分析

在微信的登录请求中，有个 rqt（Reliability Qualification Test）算法，在微信风控中扮演着重要角色，根据这个算法，可以对登录微信的环境可靠性进行判断，做为是否是外挂的重要依据。当然，除了这个算法，24 字段里面的那么多子字段也是风控的依据。7.0.x 版本的 rqt 比较简单，8.0.x 版本的 rqt 就比较复杂了，不过也是可以搞定的。

一、7.0.x 版本的 rqt 算法
==================

通过登录请求，找到 rqt 算法的函数

 

![](https://bbs.pediy.com/upload/attach/202106/860390_XVHW9SPQ3N6UPWQ.png)

 

将输入数据改成字母 "ABC"，求它的 rqt 值

 

函数最后有值

 

![](https://bbs.pediy.com/upload/attach/202106/860390_WJSBGSK3NXNBAZK.png)

 

先求 "ABC" 的 md5 值

 

![](https://bbs.pediy.com/upload/attach/202106/860390_QJ9TU5RNZQZY594.png)

 

这里对输入数据填充 0x36

 

![](https://bbs.pediy.com/upload/attach/202106/860390_D5ZU89VH98YC3BW.png)

 

这里对数据填充了 0x5c

 

![](https://bbs.pediy.com/upload/attach/202106/860390_WQ8P55ARHF96W6V.png)

 

所以可以看出采用了 HMAC 算法。

 

下图为 SHA1 的常量，所以整个是 HMAC-SHA1 算法。

 

![](https://bbs.pediy.com/upload/attach/202106/860390_9WSJ23NXFBCPQXE.png)

 

最后，还需要经过跟 0x83 的一些运算，才得到最终结果

 

![](https://bbs.pediy.com/upload/attach/202106/860390_NPKDJSWEMHN7V43.png)

二、8.0.x 版本的 rqt 算法
==================

8.0.x 的 rqt 算法，入口函数与 7.0.x 一样，也是先求 md5 值，再对 md5 进行处理。

 

接着进行魔改后的 SHA * 算法。

 

再接着，跟 0x85 来了一些运算。

 

最后，使用了类似 7.0.x 的处理，才得到最终结果。

 

最终结果，7.0.x 的 rqt 值是以 0x21 开头，8.0.x 的 rqt 值是以 0x42 开头。

 

因为对于如下式子

```
(r1 << 5 | key & 0x1f) << 24

```

7.0.x 版本，r1=1, key=1，所以

```
1 << 5 | 1 & 0x1f) << 24 = 0x21000000

```

而对于 8.0.x，r1 =2, key=2，所以

```
2 << 5 | 2 & 0x1f) << 24 = 0x42000000

```

可以看出来，算法难度比 7.0.x 版本高多了。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)