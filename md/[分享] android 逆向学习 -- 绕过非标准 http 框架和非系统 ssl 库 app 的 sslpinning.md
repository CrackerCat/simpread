> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268014.htm)

> [分享] android 逆向学习 -- 绕过非标准 http 框架和非系统 ssl 库 app 的 sslpinning

随意在安卓应用市场上下载了一个电子书 app，发现还是有点东西的：  
Charles 首次抓包报错如下：  
![](https://bbs.pediy.com/upload/attach/202106/848410_HYR3FBS4J6BRHAF.png)  
第一反应就是有 sslpinning，感觉挺简单的，但是几乎找了所有公开的 unsslpinning 脚本都无济于事， dump 证书也如此。

 

对 apk 解包时发现里面有 okhttp3，以为使用了 okhttp3 库，但 hook 后发现并不如此。

 

既然 okhttp3 hook 不到，就找了更深层次点的函数 SSLOutputStream 的 write, 奇怪的是也没发现有调用，突然意识到事情可能没那么简单了！。

 

对于这种情况，貌似就只能多对一些底层或者基础的涉及网络的函数进行 hook，终于在 java.net.url 函数打开了突破口  
![](https://bbs.pediy.com/upload/attach/202106/848410_UXBXPPTCDSMC8T4.png)  
看这个调用栈大概能看出，这个 app 因该是自己根据 okttp3 魔改了一个自己的框架，看后面的文件名感觉有点熟悉。  
![](https://bbs.pediy.com/upload/attach/202106/848410_6QGK9NYH2XMZ8N2.png)  
这不就是 okhttp 的拦截器吗？  
Ok 直接看源码  
![](https://bbs.pediy.com/upload/attach/202106/848410_YAUN5SGUXEV63JB.png)  
这个地方能看出来跟 okhttp 还是有区别的，决定 hook 下打印出所有的拦截器类。  
![](https://bbs.pediy.com/upload/attach/202106/848410_SMNPVCMVR92RPGV.png)

 

在 okhttp3 中，tls 连接的部分在倒数第二个拦截器中，但是在本 app 上并没有这样做，所以重要分析了最后一个拦截器 callServerInterceptor，具体如下：  
看这个 excutecall 函数很关键，继续往下跟，其实 hook 这个地方就已经能拿到请求体和响应内容了，但是以学习为目的的话还是要搞明白它是怎么做的。

 

这个 executel 最终会调到 com.ttnet.org.chromium.net.impl.CronetUrlRequest$1 中，然后就进入了 native 层。  
![](https://bbs.pediy.com/upload/attach/202106/848410_P53HNGVA8HU99W6.png)  
至此在 java 层也没发现在哪里有对证书的操作，所以有充分的理由相信他在 native 层做了校验。

 

尝试 hook 了下 libssl.so 中的 SSL_write 函数，居然也没发现有调用，惊出了一生冷汗，难不成是用了自己的 ssl 库？尝试搜了以下 app 已加载的 so 库，果不其然：  
![](https://bbs.pediy.com/upload/attach/202106/848410_3XFXBMXNK7AM53D.png)  
查了下这个 boringssl 是 google 开源的 openssl 分支，于是尝试 hook 了下 boringssl 中的 SSL_write 函数，果然有调用，所以可以基本确定它使用了自己的 ssl 库。

 

根据 SSL_write 的调用栈，判断该函数的调用来自 libsscronet.so，于是在该库中搜索判断是否有调用 boringssl 中涉及证书验证的函数。

 

![](https://bbs.pediy.com/upload/attach/202106/848410_HGJCHP5RDE44KQJ.png)

 

上面的不管，只看导入函数，有两个函数比较可疑 SSL_CTX_set_custom_verify 和 SSL_CTX_set_reverify_on_resume，于是下载了一份 boringssl 的源码，分别看了下这两个函数，果然是用来做证书校验的，并且都有调用。  
![](https://bbs.pediy.com/upload/attach/202106/848410_AU8BXG2ZURVY33A.png)  
该函数的原型为：  
![](https://bbs.pediy.com/upload/attach/202106/848410_M6D2TUSF5E9EPP7.png)  
第二个参数为校验模式：  
![](https://bbs.pediy.com/upload/attach/202106/848410_HZVKUAPB96YUCC7.png)  
第三个参数为回调函数。  
回调函数的返回值用来确认证书验证是否成功，具体如下：  
![](https://bbs.pediy.com/upload/attach/202106/848410_8B2R64FYUTGM7KB.png)  
Hook 之：  
![](https://bbs.pediy.com/upload/attach/202106/848410_E646APQ7FWUBC59.png)  
![](https://bbs.pediy.com/upload/attach/202106/848410_G2KK7R74S7JHD87.png)  
![](https://bbs.pediy.com/upload/attach/202106/848410_W4S47PTSNEDYMH3.png)  
这里有个问题，不能直接替换他原本的回调函数，必须要调用一次，不然会出问题，具体是什么原因懒得去研究。  
验证：  
![](https://bbs.pediy.com/upload/attach/202106/848410_D8PDWAPYV7943PH.png)  
既然这个函数可以 unpinning, 另一个函数现在就懒得去看了，有空在研究下。

 

后来大概查了下，这个 app 是今日头条旗下的，难怪如此。接下来去干签名算法了。。。。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)