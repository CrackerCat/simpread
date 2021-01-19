> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.secpulse.com](https://www.secpulse.com/archives/54027.html)

> 本文由 [wadcl](https://www.secpulse.com/archives/author/wadcl) 原创投稿安全脉搏，安全脉搏独家发表本文，如需要转载，请先联系安全脉搏授权；未经授权请勿转载。

以前给一家证券机构做测试，第一次碰到了服务器双向认证的问题，当时双向认证的概念还没有推广开来，所以折腾了很久，虽然当时不知道原理，但是也算是解决了双向认证走代理的问题了。前段时间，碰到了一个 apk，想抓包看看数据，发现用的也是双向认证，所以就折腾了一下。

当服务器启用了双向认证之后，除了客户端去验证服务器端的证书外，服务器也同时需要验证客户端的证书，也就是会要求客户端提供自己的证书，如果没有通过验证，则会拒绝连接，如果通过验证，服务器获得用户的公钥。

正式因为如此，双向认证以便都是企业内部或者证券、银行等这类用户使用，而如何保证证书的合法和保密性，就不可能通过一个公开接口去提供给访问者下载，所以一般都是放入 usb-key 中，或者是提供一个身份认证接口，认证通过后，可以下载安装，但是一般不会如此使用，这样的话，使用者多个电脑都安装的话，其他人也就可以使用了，所以保证唯一性，大部分都会采用 usb-key 的方式，所以也就限制了双向认证的使用，但是这几年手机端应用的推广，和安全的推进，很多企业在 apk 中直接封装了客户端的证书，使得我们想对 app 基于行为的安全检测，无法成功。

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/11.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/11.jpg)

**突破证书限制**
----------

所以相比于单项的认证，其实也就是多了一个服务器端验证客户端证书的过程，而在以往的用代理工具如 burp 和 fiddler 这一类工具，抓取 https 的包时，除了浏览器获取的是代理工具的证书外，默认是不发送证书给服务器端的，而其实代理工具也提供了双向认证的证书发送，如 fiddler 的 ClientCertificate.cer 证书。

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/2.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/2.jpg)

只要提供了客户端的证书，也就实现了双向认证的破解过程。

所以重点在于如何提取出证书来。

**WEB 应用上证书的提取**
----------------

最简单的一种就是，直接安装证书，或者使用时查看证书属性，在双向认证中一般会弹出此框，或者 [usb-key](https://www.secpulse.com/archives/tag/usb-key) 中如果有导入证书的功能的话更好，一般安全系数高的话，是会设置各种门槛阻止你获取证书的。

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/31.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/31.jpg)

点击后直接安装：

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/41.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/41.jpg)

安装完成后，就可以直接导出证书了。

或者直接通过 keytools 工具来生成证书。

**安卓 APP 下的证书**
---------------

在应用中嵌入证书，使得每次请求都读取证书并发送，这样做，证书一般就需要和安卓应用一起打包，甚至放置的 [**trustStore** 信任集](https://www.secpulse.com/archives/tag/trustStore%E4%BF%A1%E4%BB%BB%E9%9B%86)，就需要密码来单独提取和安装证书了。

拿到 apk 包，首先需要解压出来内部的文件：

可以搜索一些证书的后缀文件，例如 cer/p12/pfx 等，一般安卓下的为 bks，也可以先去 assets 或者 res 目录下去找找。

例如我碰到的 apk 就在 assets 目录下存放：

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/91.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/91.jpg)

我们双击 p12 安装一下，提示需要私钥密码：

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/10.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/10.jpg)

用 java 代码模拟双向认证的请求的过程：

```
        // 双向认证证书
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        KeyStore trustStore = KeyStore.getInstance("jks");
        // keyStore是服务端验证客户端的证书，trustStore是客户端的信任证书
        InputStream ksIn = new FileInputStream("E:/Java/jre8/lib/security/re/1.pfx");
        InputStream tsIn = new FileInputStream(new File("E:/Java/jre8/lib/security/re/1"));

        keyStore.load(ksIn, "123456".toCharArray());

        SSLContext sslContext = SSLContexts.custom().loadTrustMaterial(trustStore, new TrustSelfSignedStrategy())
                .loadKeyMaterial(keyStore, "123456".toCharArray()).setSecureRandom(new SecureRandom()).useSSL().build();

        ConnectionSocketFactory pSocketFactory = new PlainConnectionSocketFactory();
        SSLConnectionSocketFactory sslConnectionSocketFactory = new SSLConnectionSocketFactory(sslContext);

        Registry<ConnectionSocketFactory> r = RegistryBuilder.<ConnectionSocketFactory> create()
                .register("http", pSocketFactory).register("https", sslConnectionSocketFactory).build();
        PoolingHttpClientConnectionManager secureConnectionManager = new PoolingHttpClientConnectionManager(r);

        HttpClientBuilder secureHttpBulder = HttpClients.custom().setConnectionManager(secureConnectionManager);
        HttpClient client = secureHttpBulder.build();

        HttpGet httpGet = new HttpGet("https://xxx.com");
        HttpResponse httpResponse1 = client.execute(httpGet);

```

反编译了代码后，发现被加固过，于是想脱壳，用了网上说的[动态脱壳](https://www.secpulse.com/archives/tag/%E5%8A%A8%E6%80%81%E8%84%B1%E5%A3%B3)，不知道是不是水平问题，还是这个方法已经过去式了，反正没有成功，那咋办呢？

想到一个取巧的方法，直接搜历史 app 版本的记录，首先确定那个 app 之后开始时 https 的访问请求，然后在看这个 app 有没有加固过，最终是找到了一个年初的版本，这个版本已经开始使用了 https，但是还没有完美的加固，至于历史版本，官网几乎删除了，不过有很多应用商店，如豌豆荚。(脉搏小编：历史版本这个取巧方法不错）

app 依然有些地方被混淆了，不过无所谓，因为混淆代码一般混淆的都是自己编译的方法和类，向调用证书的函数方法，一般是组件类的，尝试的找了一下，总有一些蛛丝马迹：

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/121.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/121.jpg)

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/131-1024x315.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/131.jpg)

最终还是找到了：

[![](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/1.jpg)](https://secpulseoss.oss-cn-shanghai.aliyuncs.com/wp-content/uploads/2016/12/1.jpg)

如此获取到了秘钥之后，就可以直接导入和生成证书了。

【本文由 [wadcl](https://www.secpulse.com/archives/author/wadcl) 原创投稿安全脉搏，安全脉搏独家发表本文，如需要转载，请先联系安全脉搏授权；未经授权请勿转载。】

**本文作者：[wadcl](https://www.secpulse.com/archives/newpage/author?author_id=3261)**

**本文为安全脉搏专栏作者发布，转载请注明：**[**https://www.secpulse.com/archives/54027.html**](https://www.secpulse.com/archives/54027.html)