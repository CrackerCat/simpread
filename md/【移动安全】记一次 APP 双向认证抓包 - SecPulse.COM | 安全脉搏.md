> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.secpulse.com](https://www.secpulse.com/archives/117194.html)

前言
--

随着移动应用的发展和推广，APP 应用安全越来越受到重视。APP 中各类防抓包的机制的出现，让测试无法正常进行分析。  
这篇文章算是总结一下我最近遇到的一款抓不到包的 APP，给大家提供一个双向证书认证应该如何解决的思路。

判断证书双向认证
--------

刚拿到此 app 时候常规方法一把梭，发现只要一开启手机代理，却提示网络异常，通过观察 burpsuite 的记录发现，只有请求包而没有响应包。  
直觉告诉我应该是使用 SSL Pinning 防止中间人拦截攻击，然后我开启了 ssl-kill-switch2 后发现该 APP 所有的响应包返回 400 No required SSL certificate was sent 的报错信息。  
[![](https://image.3001.net/images/20191029/1572363150_5db85b8eaddab.jpg!small)](https://www.secpulse.com/wp-admin/post.php?post=117212&action=edit)

根据报错提示，搜了一下发现该错误是指服务器端启用了证书双向认证。

> 当服务器启用了证书双向认证之后，除了客户端去验证服务器端的证书外，服务器也同时需要验证客户端的证书，也就是会要求客户端提供自己的证书，如果没有通过验证，则会拒绝连接，如果通过验证，服务器获得用户的公钥。

该 app 直接封装了客户端的证书，相比于单项认证，无非就是多了一个服务器端验证客户端证书的过程，而在以往的用代理工具如 burp 这类工具，抓取 https 的包时，除了浏览器获取的是代理工具的证书外，默认是不发送证书给服务器端的。burp 在抓取 https 报文的过程中也提供了双向认证的证书发送，但是是使用了 burp 提供的证书文件，也就是 CA 证书。app 的服务端不认证这个 burp 提供的 CA 证书，那么我们就需要拿到匹配的证书，以其对服务端进行匹配。

突破思路
----

确定该 APP 是证书双向认证，那么 APP 客户端一定会存一个证书文件。通过对该 APP 解压并进入 payload 目录，发现只有一个. p12 结尾的证书文件。  
![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363205_5db85bc54f792.jpg)

尝试点开发现需要安装密码。  
![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363239_5db85be74641b1.jpg)

app 解密的代码逻辑
-----------

客户端发送数据包以后，需要去从 app 中读取这个证书文件，密码是以硬编码形式放在了代码中，利用这个代码中的密码字段去解密证书文件，从中读取以后，再进行解密并回传给服务器端进行确认。由此推断，寻找证书名称应该就可以拿到安装密码。

获取安装证书密码
--------

首先对其 APP 进行砸壳，完成后我们解压缩然后使用 IDA 加载二进制文件。  
然后在 String 窗口搜索证书的名称 client，搜索后进入对应的类。  
![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363291_5db85c1b1d4a31.jpg)

通过跟踪发现了该证书密钥，如下：  
![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363313_5db85c311d7961.jpg)

测试使用该密钥发现可以成功安装该证书：

![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363333_5db85c45002981.jpg)

burp 添加客户端证书
------------

host 填写 app 服务端的主域名。  
![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363361_5db85c610655d1.jpg)

随后选择 app 客户端内的 client.p12 证书文件，并输入安装密码。

![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363513_5db85cf9ac4c71.jpg)

证书成功导入，勾选即可使用。

![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363419_5db85c9b5b79e1.jpg)

ok~ 发现可以正常抓包，如下。  
![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2019/10/1572363403_5db85c8b83d521.jpg)

参考文章, 感谢各位大佬的倾情奉献

> [https://se8s0n.github.io/2018/09/11/HTTP 系列 (五)/](https://se8s0n.github.io/2018/09/11/HTTP%E7%B3%BB%E5%88%97(%E4%BA%94)/)  
> [https://xz.aliyun.com/t/6551#toc-14](https://xz.aliyun.com/t/6551#toc-14)  
> [https://juejin.im/post/5c9cbf1df265da60f6731f0a](https://juejin.im/post/5c9cbf1df265da60f6731f0a)  
> [https://www.secpulse.com/archives/54027.html](https://www.secpulse.com/archives/54027.html)

**本文作者：[TideSec](https://www.secpulse.com/archives/newpage/author?author_id=26366)**

**本文为安全脉搏专栏作者发布，转载请注明：**[**https://www.secpulse.com/archives/117194.html**](https://www.secpulse.com/archives/117194.html)