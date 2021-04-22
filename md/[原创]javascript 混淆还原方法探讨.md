> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-263243.htm)

对于 web 爬虫经验丰富的人来说这个并不算有难度的事情，但是对于熟悉安卓不熟悉 web 的开发者来说有些费劲，现在很多 APP 中 webview 使用率越来越多并且混淆也越来越强，毕竟脚本语言无法编译，只能通过伪编译或者混淆，今天分享一下一种案例来举例如何还原一个混淆 js。

准备工具：charles，nginx，chrome，python

步骤：

      用抓包工具抓到相关网页地址，然后打开 chrome 浏览器点击 F12 键打卡开发者工具转化成模拟移动设备模式，然后访问。这是浏览器会加载所需的 js 代码和其他资源文件，

![](https://bbs.pediy.com/upload/attach/202011/807809_DAKPNPZ8GYFWPVQ.jpg)

点击 Source 能看到所有的资源，到相应资源鼠标右键 Save As 就可以保存了。

![](https://bbs.pediy.com/upload/attach/202011/807809_6YYCVXWAXUDFUH3.jpg)

保存的 html 和 js 代码放到 nginx 的 html 目录下，启动 ngix，然后访问 127.0.0.1:8089, 端口在 nginx.conf 里自行配置。

![](https://bbs.pediy.com/upload/attach/202011/807809_DK8E2X53FG9ADW5.jpg)

如果 js 是压缩的就解压，把代码各行的显示这样下断点容易。  

![](https://bbs.pediy.com/upload/attach/202011/807809_EY85SUNVBMD54CK.jpg)

这个是一个抽出来的的验证码的 webview

![](https://bbs.pediy.com/upload/attach/202011/807809_ZVCSBDJVC4Z53S3.jpg)

一看能看出来通过_0x2ef9 函数隐藏了一部分 api 对象或者字符串函数名等，我们先用正则表达找到所有_0x2ef9 函数的调用，然后参数全部列出来去重，然后 js 里调用 console 返回值。

![](https://bbs.pediy.com/upload/attach/202011/807809_2U25R3TV8KKUSJQ.jpg)

![](https://bbs.pediy.com/upload/attach/202011/807809_725VAXQBNP7NVNF.jpg)

这样就差不多走了 80% 的路程了。然后写一个脚本查找所有_0x2ef9 函数替换解密出来的值就还原了隐藏的内容了，下一步就修复 js 了，这一步可能费点时间，不过会 js 的人基本上能确定那个是 js 的 api，那个是字符串，不会的需要复习一下。

![](https://bbs.pediy.com/upload/attach/202011/807809_6RH3R3CYRNJQAZS.jpg)

![](https://bbs.pediy.com/upload/attach/202011/807809_4ETBJ52WFD9PDSS.jpg)

替换后的部分代码，觉得有希望了。

![](https://bbs.pediy.com/upload/attach/202011/807809_7FBGUNRQYX2HS56.jpg)

这是 js 混淆还原的一种解决办法，还有很多可以大家一起探讨，基本上搞过前端或者 web 的都会。

[[公告]5 月 14 日腾讯安全零信任发展趋势论坛重磅开幕！邀您一起从 “零” 开始，共建信任！！](https://zta.insecworld.com/?utm_campaign=MJTG&utm_source=KX&utm_medium=WZLJ)