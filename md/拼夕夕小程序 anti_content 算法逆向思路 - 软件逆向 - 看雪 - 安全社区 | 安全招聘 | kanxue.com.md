> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283125.htm)

> 拼夕夕小程序 anti_content 算法逆向思路

拼夕夕 anti_content 算法网页版的 anti_content 算法很多逆向大佬发表过文章，唯独小程序版 anti_content 算法网络上比较少见。在逆向算法这块本人菜鸟一个，通过分析核心 JS 文件发现，比起网页版的 anti_content 算法相对是比较简单的，比如没有做过多的混淆, 由于微信小程序对 API 的限制，需要补的环境也少很多。本人不做爬虫，所以没有去分析拼夕夕风控相关的，比如获取商品已售罄的提示，这个和账号和其他风控相关不在讨论范围，只分享 anti_content 算法逆向思路，请勿使用于非法用途！

一、反编译拼夕夕小程序源码

我比较喜欢分析 PC 微信小程序，提取相对比较简单。反编译的工具有很多，个人比较喜欢 unveilr，无论什么工具反编译出来的源码不能直接调用。会提示各种错误，按照错误提示，去掉相关出错的地方代码就行了。使用微信开发者工具调用源码，AppID 选择测试号，后端服务选择不使用云服务。调试使用真机调用，如果直接调试出来的参数和真机不一样，获取到的 anti_content 算法是无法使用的，过不了服务器检测，区别是调用环境 anti_content 长度比较短，真机 anti_content 长度比较长。

![](https://bbs.kanxue.com/upload/tmp/1002050_KVR4GNK5ZWG7GMB.webp)

![](https://bbs.kanxue.com/upload/tmp/1002050_2YQT336CKBA6S4P.webp)

二、JS 文件逆向思路

anti_content 算法在 lightMiniApp.js 里面

在 node 环境里面不兼容微信小程序 api, 比如 wx.getSystemInfo，这些 API 的作用是获取小程序一些环境参数，需要手工补上环境参数。其实 anti_content 算法用到的环境参数不多，由于涉及到细节的东西就不好文字说明，比如其中有一个参数是 version，代表的是环境系统版本，比如 PC 微信端就是就是微信的版本号。

参数就几个，相信各位慢慢分析就能找到的。这些参数都需要手工补上去，生成的 anti_content 才可以用。

抓包商品详情接口发现，head 头中有个参数 rfp 也参与了 anti_content 算法，rfp 算法可以参考我另外写的分析文章。

测试发现拼夕夕对时间这个参数很重视的，很多地方都加入了时间参数，这个也是个坑，如果时间参数不对生成的算法也是过不了。

下面放几张图，供大家参考。喜欢小程序逆向算法的可以和我多多交流。

1. 没有补环境生成的短 anti_content

![](https://bbs.kanxue.com/upload/tmp/1002050_BHYNHD56T4FMGCK.webp)

2. 补好环境生成可以通过测试的 anti_content

![](https://bbs.kanxue.com/upload/tmp/1002050_FNG8Y8RTNS2TTSK.webp)