> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-259651.htm)

> [原创] 对 QQ 手游授权登录的一点分析

1. 前言
=====

最近无事玩王者，发现某些租号平台可以直接通过自身的 APP 打开腾讯的游戏进行登录，于是对这一登录过程做了简单的分析

2.QQ 打开游戏的简单分析
--------------

![](https://bbs.pediy.com/upload/attach/202005/258186_W5XSMUX7ZD98X2H.png)

 

发现不管是 IOS 还是 Android 都可以在 QQ 里面的游戏中心直接打开腾讯相关的游戏，然后通过抓包发现 QQ 打开游戏的时候会访问的 URL 请求地址，在 JEB 里面对这个地址进行搜索找到相关代码  
![](https://bbs.pediy.com/upload/attach/202005/258186_V5ZKYB62SMWJWWH.png)  
伪造王者荣耀包名，使用 QQ 打开伪造的王者荣耀 APP 对参数进行截取，对比代码分析发现，登录所使用的参数 atoken ptoken 等参数是可以通过官方 SDK 获取的  
![](https://bbs.pediy.com/upload/attach/202005/258186_283WVD6KEEBZVSR.png)

 

对 IOS 也通过伪造包名发现登录参数的格式是这样的  
tencentlaunch1104466820://startapp?atoken=CB6CE0E7C75849189FFD0D692576C67E&openid=24B0EFBBE7FD07941C6BD452CD6E9E32&ptoken=1D044410A43724E932DEB12CB31A5190&platform=qq_m&current_uin=24B0EFBBE7FD07941C6BD452CD6E9E32&launchfrom=sq_gamecenter

### 3. 使用 QQ SDK 获取登录参数测试 APP 打开

使用 QQ SDK 获取的登录参数，发现打开的王者荣耀 APP 账号并没有登录上去，所以怀疑，通过 QQ SDK 获取的参数有问题，想到腾讯手游里面是可以直接通过 QQ 网页登录游戏的，于是我就在伪造的包名的 APP 里面通过调用 SDK 发现这个登录请求是带了 APPID 的  
https://ssl.ptlogin2.qq.com/check?pt_tea=2&uin=392469882&appid=716027609&ptlang=2052&regmaster=&pt_uistyle=35&r=0.5246853942298563  
![](https://bbs.pediy.com/upload/attach/202005/258186_TNNPTGJ52DWNYRV.png)  
![](https://bbs.pediy.com/upload/attach/202005/258186_NVKKNMRDX9PH7XV.png)  
然后使用返回的参数测试登录王者荣耀 APP 成功

 

https://xui.ptlogin2.qq.com/cgi-bin/xlogin?appid=716027609&pt_3rd_aid=1104466820&daid=381&pt_skey_valid=0&style=35&s_url=http%3A%2F%2Fconnect.qq.com&refer_cgi=m_authorize&ucheck=1&fall_to_wv=1&status_os=13.3.1&redirect_uri=auth%3A%2F%2Fwww.qq.com&client_id=1104466820&response_type=token&scope=get_user_info%2Cget_simple_userinfo%2Cadd_t&sdkp=i&sdkv=3.3.8_lite&state=test&status_machine=iPhone9%2C1&switch=1&traceid=NjM1ODQ2RjUtMEUwMC00RjgyLTkxNTktQjU5OEJFQTQwQzA0_1587467997

 

然后又打开了这个地址发现就是个 JS 的 QQ 的网页登录  
![](https://bbs.pediy.com/upload/attach/202005/258186_2HHEXPH76GSGSWA.png)  
然后通过网上搜索发现这个网页的 QQ 登录分析文章挺多的，我这里随便找了个分析的连接，感兴趣的朋友可以自行学习 https://blog.hidove.cn/post/713

#### 4. 后记

作为一个约不到妹子程序员，整个 5.20 都在孤单中度过，于是就无聊的整理了一篇文章

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)