> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270634.htm)

> [原创]Android APP 漏洞之战（6）——HTTP/HTTPs 通信漏洞详解

Android 漏洞之战（6）——http/https 通信安全漏洞详解
====================================

目录

*   [Android 漏洞之战（6）——http/https 通信安全漏洞详解](#android漏洞之战（6）——http/https通信安全漏洞详解)
*            [一、前言](#一、前言)
*            [二、基础知识](#二、基础知识)
*                    [1. 加密算法](#1.加密算法)
*                            [（1）对称加密](#（1）对称加密)
*                            [（2）非对称加密](#（2）非对称加密)
*                    [2. 信息安全问题](#2.信息安全问题)
*                            [（1）信息的保密性](#（1）信息的保密性)
*                            [（2）信息的完整性（数字签名）](#（2）信息的完整性（数字签名）)
*                            [（3）身份识别（数字证书）](#（3）身份识别（数字证书）)
*                    [3.Http/Https 详解](#3.http/https详解)
*                            [（1）TLS/SSL](#（1）tls/ssl)
*                            [（2）HTTPs 的单向认证和双向认证](#（2）https的单向认证和双向认证)
*                                    <1> 单向认证
*                                    <2> 双向认证
*                    4.Android Http 网络开发
*                            [（1）HttpURLConnection](#（1）httpurlconnection)
*                            [（2）okhttp3](#（2）okhttp3)
*                            [（3）各自证书的校验方式](#（3）各自证书的校验方式)
*            [三、HTTP/HTTPs 漏洞分析和复现](#三、http/https漏洞分析和复现)
*                    [1. 漏洞的安全种类和危害](#1.漏洞的安全种类和危害)
*                    [2.HTTP 明文传输漏洞](#2.http明文传输漏洞)
*                            [（1）漏洞案例](#（1）漏洞案例)
*                                    <1> 酷我音乐 APP 存在逻辑缺陷漏洞
*                                    <2> 上海任意门科技有限公司 Soul APP 存在信息泄露漏洞
*                                    <3> 酷狗直播存在逻辑缺陷漏洞（hash 验证）
*                            [（2）漏洞防护](#（2）漏洞防护)
*                    [3.HTTPs 密文传输漏洞](#3.https密文传输漏洞)
*                            [（1）漏洞案例](#（1）漏洞案例)
*                                    <1> 忽略 SSL 证书校验漏洞
*                                    <2> 忽略域名校验漏洞
*                                    <3> 作业帮存在 https 升级劫持漏洞
*                            [（2）漏洞防护](#（2）漏洞防护)
*            四、Android 绕过 https 的 SSL Pining
*                    1.SSL pinning
*                    2.Android 7.0 后破解 https 的 ssl pinning
*                    [3. 证书绕过原理](#3.证书绕过原理)
*                    [4. 工具使用](#4.工具使用)
*                            [（1）Xposed+JustTrustMe](#（1）xposed+justtrustme)
*                            [（2）Frida 脚本](#（2）frida脚本)
*            [五、实验总结](#五、实验总结)
*            [六、参考文献](#六、参考文献)

[](#一、前言)一、前言
-------------

本文主要介绍 Android http/https 方面的安全漏洞问题，并会从原理并结合案例来逐一讲解，本文一部分参考网络上一些博客，并在相应部分给出链接

 

本文第二节主要讲述 Android http/https 相关的基础知识

 

本文第三节为漏洞原理解析和漏洞复现

 

本文第四节为 Android https 转包漏洞介绍

[](#二、基础知识)二、基础知识
-----------------

### 1. 加密算法

#### [](#（1）对称加密)（1）对称加密

对称加密算法是双方都持有相同的密钥进行通信，加密速度很快

 

![](https://bbs.pediy.com/upload/attach/202112/905443_ZY4W66BVMZX9BGM.png)

 

特点：

```
a.加密和解密都是用同一个秘钥
b.加密、解密效率高
c.秘钥被窃取，容易造成数据不安全
常见的对称加密算法有DES、3DES、AES等，这里我们就不深入讲解了

```

缺点：

 

上面的对称加密模型最大的问题就是，对称加密模型需要一个安全的信道来传输对称密钥，但是如果真的存在一个真正安全的信道，那直接用这个信道来传输数据就可以了，这就有点矛盾了

#### [](#（2）非对称加密)（2）非对称加密

非对称加密，是为了解决对称加密中的安全问题而诞生，含有一对密钥：公钥和私钥，发送方用公钥进行加密，公钥可以被公开，接收方用私钥进行解密，私钥不可公开

 

![](https://bbs.pediy.com/upload/attach/202112/905443_2NXNM9WJ2QHEHMU.png)

 

特点：

```
a.用公钥加密用私钥解密
b.加密、解密相对于对称加密效率更低，但是比对称加密更安全
c.公钥可能被中间人伪造，造成数据不安全
常见的非对称加密算法有RSA、DSA等

```

缺点：

 

如何保证加密的是接收方的公钥，如何安全的传输公钥

### 2. 信息安全问题

我们在传输数据的过程中往往着眼于三个方面的安全问题：

```
（1）信息的保密性
（2）信息的完整性
（3）身份识别

```

#### [](#（1）信息的保密性)（1）信息的保密性

我们一般会使用各种加密算法对我们传输的数据信息进行加密，即使用上面的对称加密和非对称加密来完成，但无论是对称加密还是非对称加密都存在一个共同的安全问题：`密钥如何传递，而且提高传输速率`，一般公用的方法是采用`对称加密+非对称加密结合`，即双方都在使用对称加密进行传输，但是会存在密钥不能保证安全性的问题，此时我们使用公钥对对称密钥进行加密，然后接收方使用私钥对对称密钥进行解密，这样就可以解决这个问题

 

![](https://bbs.pediy.com/upload/attach/202112/905443_SCEV4D3ZG75J3JT.png)

#### [](#（2）信息的完整性（数字签名）)（2）信息的完整性（数字签名）

数据在传输的过程中，我们的信息可能被第三方劫持篡改，所以我们要保证信息的完整性，一般通过使用散列函数如 SHA1，MD5 将传输内容 hash 依次获得 hash 值，即摘要。客户端使用服务端的公钥对摘要和信息内容进行加密，传输给服务端，服务端使用私钥进行解密，然后用相同的 hash 算法对原始内容进行 hash，然后与摘要值对比，如果一直，说明信息是完整的

 

![](https://bbs.pediy.com/upload/attach/202112/905443_JJTNEHW834H4AZQ.png)

 

举例：

```
Android APP应用一般就具有签名验证机制，以防止恶意攻击者在对APP进行逆向之后，重打包，一般来说Android APP的签名机制分为3类：
（1）java本地验证，在java代码中有hash函数验证，我们通常搜索signature定位到目标代码段，直接删除或hook该代码段即可
（2）so本地验证，为了加强逆向难度，很多公司会将APP验证写在so层，这一般我们通过IDA动态调试，获取代码段然后NOP即可
（3）网络服务器验证，一般来说这种进行网络hash验证，一般这种通过抓包，但有一些加密后变很难处理了

```

#### [](#（3）身份识别（数字证书）)（3）身份识别（数字证书）

我们在信息传输过程中，通常要验证信息的发送方的身份，我们将发送端的公钥发送给接收端，发送端通过把自己的内容使用私钥加密然后发送给接收端，接收端只能使用发送端的公钥加密，自然就验证了发送端的身份

 

![](https://bbs.pediy.com/upload/attach/202112/905443_MDCKYFMAEV8RACA.png)

 

数字证书：

 

但是上述过程中存在一个问题，在传输的过程中，客户端如何获得服务器的公钥呢？当服务器分发给客户端，如果一开始服务端发送的公钥到客户端就被第三方劫持，然后第三方自己伪造一对密钥，将公钥发送给客户端，当服务端发生数据给客户端的时候，中间人就将信息劫持，用一开始劫持的公钥进行解密，然后将自己的私钥将数据发送给客户端，而客户端收到后使用公钥解密，这个过程中中间人是透明的，就可以获取信息了

 

![](https://bbs.pediy.com/upload/attach/202112/905443_VZ4479AU5Q9A6J9.png)

 

为了防止这种中间人攻击，数字证书就出现了，其实是基于上面所说的私钥加密数据，公钥解密来验证其身份

 

数字证书是由权威的 CA（Certificate Authority）机构给服务端进行颁发，CA 机构通过服务端提供的相关信息生成证书，证书内容包含了持有人的相关信息，服务器的公钥，签署者签名信息（数字签名）等，最重要的是公钥在数字证书中

 

数字证书由权威的 CA（Certificate Authority）机构给服务端进行颁发，CA 机构通过服务端提供的相关信息生成证书，证书内容包含了持有人的相关信息，服务器的公钥，签署者签名信息（数字签名）等，最重要的是`公钥在数字证书`中

```
(1)数字证书是如何保证公钥来自于请求的服务器呢？
数字证书上由持有人的相关信息，通过这点可以确定其不是一个中间人
(2)如何保证数字证书为真呢？
一个证书中含有三个部分:"证书内容，散列算法，加密密文"，证书内容会被散列算法hash计算出hash值，然后使用CA机构提供的私钥进行RSA加密

```

![](https://bbs.pediy.com/upload/attach/202112/905443_N7DKDAV6UN7658N.png)

 

客户端完成验证过程：

```
当客户端发起请求是，服务端将该数字证书发送到客户端，客户端通过CA机构提供的公钥对加密密文来进行解密获得散列值（数字签名），同时将证书内容使用相同的散列算法进行Hash得到另一个散列值，比对两个散列值，如果两者相等则说明证书没问题

```

![](https://bbs.pediy.com/upload/attach/202112/905443_PBQ6SP24YDGM559.png)

 

一些常见的证书分类：

```
X.509#DER 二进制格式证书，常用后缀.cer .crt
X.509#PEM 文本格式证书，常用后缀.pem
有的证书内容是只包含公钥（服务器的公钥），如.crt、.cer、.pem
有的证书既包含公钥又包含私钥（服务器的私钥），如.pfx、.p12

```

为了保证证书的一致性，国际电信联盟设计了一套专门针对证书格式的标准 X.509，其核心提供了一种描述证书的格式

 

![](https://bbs.pediy.com/upload/attach/202112/905443_TE82QCCR9G69DHV.png)

### 3.Http/Https 详解

#### （1）TLS/SSL

http: 超文本传输协议，采用明文的方式去传输数据，经过我们上文的分析，在这个过程中很容易导致中间人攻击，因此为了进一步增强数据传输的安全性，开始出现 https，而在此之前我们就需要了解一下 TLS/SSL

```
SSL：（Secure Socket Layer，安全套接字层），位于可靠的面向连接的网络层协议和应用层协议之间的一种协议层。SSL通过互相认证、使用数字签名确保完整性、使用加密确保私密性，以实现客户端和服务器之间的安全通讯
TLS：(Transport Layer Security，传输层安全协议)，用于两个应用程序之间提供保密性和数据完整性

```

我们先看一下 SSL 和 TLS 的区别：

```
简而言之，TLS只是SSL后来迭代的版本而已，在1994年，NetScape设计了SSL协议，1999年，互联网标准化组织ISOC接替NetScape公司，发布了SSL的升级版TLS 1.0版，因此可以理解为TLS 1.0 = SSL 3.1，只是SSL后来的的版本而已

```

SSL 协议即用到了对称加密也用到了非对称加密 (公钥加密)，在建立传输链路时，SSL 首先对对称加密的密钥使用公钥进行非对称加密，链路建立好之后，SSL 对传输内容使用对称加密

```
1.对称加密
速度高，可加密内容较大，用来加密会话过程中的消息
 
2.公钥加密
加密速度较慢，但能提供更好的身份认证技术，用来加密对称加密的密钥

```

因此，HTTPs = HTTP + TLS/SSL

 

![](https://bbs.pediy.com/upload/attach/202112/905443_Q7QC3VYRBAHJCN7.png)

#### [](#（2）https的单向认证和双向认证)（2）HTTPs 的单向认证和双向认证

##### <1> 单向认证

Https 在建立 Socket 连接之前，需要进行握手，具体流程：

 

![](https://bbs.pediy.com/upload/attach/202112/905443_D5JZTT2NY4UURDK.png)

```
1.客户端向服务端发送SSL协议版本号、加密算法种类、随机数等信息。
2.服务端给客户端返回SSL协议版本号、加密算法种类、随机数等信息，同时也返回服务器端的证书，即公钥证书
3.客户端使用服务端返回的信息验证服务器的合法性，包括：
    (1)证书是否过期
    (2)发型服务器证书的CA是否可靠
    (3)返回的公钥是否能正确解开返回证书中的数字签名
    (4)服务器证书上的域名是否和服务器的实际域名相匹配、验证通过后，将继续进行通信，否则，终止通信
4.客户端向服务端发送自己所能支持的对称加密方案，供服务器端进行选择
5.服务器端在客户端提供的加密方案中选择加密程度最高的加密方式。
6.服务器将选择好的加密方案通过明文方式返回给客户端
7.客户端接收服务端返回的加密方式后，使用该加密方式生成产生随机码，用作通信过程中对称加密的密钥，使用服务端返回的公钥进行加密，将加密后的随机码发送至服务器
8.服务器收到客户端返回的加密信息后，使用自己的私钥进行解密，获取对称加密密钥。在接下来的会话中，服务器和客户端将会使用该密码进行对称加密，保证通信过程中信息的安全

```

##### <2> 双向认证

双向认证和单向认证原理基本差不多，只是除了客户端需要认证服务端以外，增加了服务端对客户端的认证，具体过程如下：

 

![](https://bbs.pediy.com/upload/attach/202112/905443_TQTDSBYK4KGWZKU.png)

```
1.客户端向服务端发送SSL协议版本号、加密算法种类、随机数等信息。
2.服务端给客户端返回SSL协议版本号、加密算法种类、随机数等信息，同时也返回服务器端的证书，即公钥证书
3.客户端使用服务端返回的信息验证服务器的合法性，包括：
    (1)证书是否过期
    (2)发型服务器证书的CA是否可靠
    (3)返回的公钥是否能正确解开返回证书中的数字签名
    (4)服务器证书上的域名是否和服务器的实际域名相匹配、验证通过后，将继续进行通信，否则，终止通信
4.服务端要求客户端发送客户端的证书，客户端会将自己的证书发送至服务端
5.验证客户端的证书，通过验证后，会获得客户端的公钥
6.客户端向服务端发送自己所能支持的对称加密方案，供服务器端进行选择
7.服务器端在客户端提供的加密方案中选择加密程度最高的加密方式
8.将加密方案通过使用之前获取到的公钥进行加密，返回给客户端
9.客户端收到服务端返回的加密方案密文后，使用自己的私钥进行解密，获取具体加密方式，而后，产生该加密方式的随机码，用作加密过程中的密钥，使用之前从服务端证书中获取到的公钥进行加密后，发送给服务端
10.服务端收到客户端发送的消息后，使用自己的私钥进行解密，获取对称加密的密钥，在接下来的会话中，服务器和客户端将会使用该密码进行对称加密，保证通信过程中信息的安全。

```

### 4.Android Http 网络开发

我们要学习 Https 通信漏洞挖掘，首先就需要掌握基本的 Android http 网络开发，因为开发和逆向漏洞总是相互相成的，Android 的 HTTP 的网络通信框架一般包括两类：第一类是原生的 Android 网络 HTTP 通信库，原生网路通信库主要通过 HttpURLConnection 以及 HttpClient 两个类完成，但是 Android6.0 后，Andriod 中的 SDK 就去掉了 HttpClient 的支持，Android 9 后，Android 就直接取消了 HttpClient 的支持，但是由于网络通信的操作涉及异步、多线程和效率的问题，HttpURLConnection 中并未对这些操作进行完整的封装，就出现第二类网络通信框架——第三方 HTTP(s) 的网络请求框架，一般为：okhttp、Volley 等，这里我们只介绍当下使用比较广泛的框架

 

![](https://bbs.pediy.com/upload/attach/202112/905443_H6F9YA24E6UM224.png)

#### [](#（1）httpurlconnection)（1）HttpURLConnection

获取 HttpURLConnection 实例，通过 openConnection() 获取

```
URL url = new URL("http://www,baidu.com");
HttpURLConnection connection = (HttpURLConnection) url.openConnection();

```

设置 HTTP 请求使用的方法，`GET`表示希望从服务器那里获取数据，`POST`表示提交数据给服务器

```
connection.setRequestMethod("GET");
或者
connection.setRequestMethod("POST");

```

再就是一些自由定制，如设置连接超时、读取超时的毫秒数、服务器的一些消息头等

```
connection.setRequestProperty("token","wwanghai");//设置请求参数
connection.setConnectTimeout(8000);//设置连接超时时间
connection.setReadTimeout(8000);//设置接收超时时间

```

调用 getInputStream() 方法获取服务器返回的输入流

```
InputStream in = connection.getInputStream();

```

我们可以用字节数组保存读取的数据

```
int bufferSize = 1024;
byte[] buffer = new byte[bufferSize];
StringBuffer sb = new StringBuffer();
while (in.read(buffer)!=-1){
     sb.append(new String(buffer));
     }

```

最后调用 disconnect() 方法将 HTTP 连接关闭掉

```
connection.disconnect();

```

#### [](#（2）okhttp3)（2）okhttp3

okHttp 的项目主页地址是：[okHttp](https://github.com/square/okhttp)

 

我们需要在项目中添加依赖，编辑 app/build.gradle 文件

```
implementation("com.squareup.okhttp3:okhttp:3.12.0")

```

首先创建一个 OkHttpClient 的实例

```
OkHttpClient client = new OkHttpClient();
或者加一些设置
OkHttpClient client = new OkHttpClient.Builder()
                                        .readTimeout(5, TimeUnit.SECONDS)  //设置读超时
                                        .writeTimeout(5, TimeUnit.SECONDS)  //设置写超时
                                        .connectTimeout(15,TimeUnit.SECONDS) //设置连接超时
                                        .retryOnConnectionFailure(true) //是否自动重连
                                        .build();

```

如果要发起 HTTP 请求，就需要创建一个 Request 对象

```
GET:
    Request request = new Request.Builder().build(); //这是一个空的对象
    实际使用中
    Request request = new Request.Builder()
                                  .url("http://www,baidu.com")
                                  .header("token","wanghai")
                                  .build();
POST：
    //需要先构建一个RequestBody来存放待提交的参数，然后再传入Request
    RequestBody requestBody = new FormBody.Builder()
                                            .add("username","damin")
                                            .add("password","123456")
                                            .build();
    Request request = new Request.Builder()
                                  .url("http://www,baidu.com")
                                  .post(requestBody)
                                  .build();

```

调用 OkHttpClient 的 newCall() 方法来创建一个 Call 对象，并调用 execute() 方法来发送请求并获取服务器返回的数据，response 对象就是服务器返回的数据

```
Response response = client.newCall(request).execute();
String responseData = response.body().string();

```

还可以使用异步方式来获取数据，Android 中大部分都使用异步方式来获取数据，通过 enqueue（）函数产生一次真实的网络请求，通过 onResponse（）函数进行回调

```
client.newCall(request).enqueue(new Callback() {
               @Override
               public void onFailure(Call call, IOException e) {
                   call.cancel();
               }
 
               @Override
               public void onResponse(Call call, Response response) throws IOException {
                   String responseData = response.body().string();
               }
           });

```

#### [](#（3）各自证书的校验方式)（3）各自证书的校验方式

```
（1）根据 app 内置证书 KeyStore 生成 TrustManager 验证
（2）自定义 SSLSocketFactory(org.apache.http.conn.ssl.SSLSocketFactory)实现 TrustManager 验证策略(httpClient)
（3）自定义SSLSocketFactory(javax.net.ssl.SSLSocketFactory)实现TrustManager 验证策略(HttpsURLConnection,OkHttp3)
（4）自定义的 HostnameVerifier 和 X509TrustManager 实现验证
（5）第三方库中的验证，如 OkHttp3 中的 CertificatePinner(证书锁定)
（6）WebView 加载 Https 页面时证书校验出错，停止加载

```

下面是比较常见的实现 https 类的各自证书校验方式：

 

![](https://bbs.pediy.com/upload/attach/202112/905443_VRDJKTB2QDRSNQ6.png)

 

下面是证书验证的一些关系示意图，参考链接：[证书关系](https://docs.oracle.com/javase/6/docs/technotes/guides/security/jsse/JSSERefGuide.html)

 

![](https://bbs.pediy.com/upload/attach/202112/905443_9G2XZ9J7CNQRDM4.png)

 

上图中要进行 SSL 会话，必须先建立一个 SSLSocket 对象，而 SSLSocket 对象是通过 SSLSocketFactory 来管理的，SSLSocketFactory 对象则依赖于 SSLContext ，SSLContext 对象的初始化需要 keyManager、TrustManager 和 SecureRandom。TrustManager 对象是我们后文比较关心的，因为正是 TrustManager 负责证书的校验，对网站进行认证，要想确保数据不被中间人抓包分析，就需要实现这个类进行验证，以保障数据的安全性

 

在整个过程中 TrustManager 类专门负责校验证书，可以改写 TrustManager 类，实现对证书对校验或让它不要对证书做校验

三、HTTP/HTTPs 漏洞分析和复现
--------------------

### 1. 漏洞的安全种类和危害

Andoid 的网络通信中一般采用 http 明文传输，或使用 SSL/TLS 协议的 https 密文传输，对于 http 明文传输来说自然会导致很多漏洞，例如信息泄露漏洞，升级劫持漏洞，验证码口令泄露漏洞等等，而使用 https 传输的明文，也存在大量的 HTTPs 证书不校验漏洞，中间人攻击漏洞等等

 

![](https://bbs.pediy.com/upload/attach/202112/905443_Z5CQYSQEWEXYEP5.png)

### 2.HTTP 明文传输漏洞

#### [](#（1）漏洞案例)（1）漏洞案例

##### <1> 酷我音乐 APP 存在逻辑缺陷漏洞

[酷我音乐 APP 存在逻辑缺陷漏洞](https://www.cnvd.org.cn/flaw/show/CNVD-2021-45684)

 

漏洞原理：

```
酷我音乐APP采用http明文传输，攻击者可以通过利用该漏洞利用代理工具篡改数据包来升级校验，从而导致APP升级过程中恶意软件注入攻击

```

漏洞复现：

 

我们点击检测新版本，可以抓取对应的响应请求，其中下面的是下载请求

 

![](https://bbs.pediy.com/upload/attach/202112/905443_6KW96YXS22HVWTW.png)

 

然后，我们可以发现程序下载完成后，显示正常的升级界面

 

![](https://bbs.pediy.com/upload/attach/202112/905443_J98G328JND35KBG.png)

 

我们可以知道这条请求就是程序的下载请求，对应的就是下载的 apk，我们尝试劫持这条请求，将 apk 替换成我们的恶意锁机程序

 

![](https://bbs.pediy.com/upload/attach/202112/905443_GEFPVG2P74RSHH2.png)

 

下劫持响应请求断点，可以让我们在请求响应前劫持

 

![](https://bbs.pediy.com/upload/attach/202112/905443_ECQF6979VDWT3Q2.png)

 

通过 HFS 文件管理服务器，来模拟请求的服务器

 

![](https://bbs.pediy.com/upload/attach/202112/905443_353NBAWAFHS25JE.png)

 

注意路径应与 apk 下载请求 url 保持一致，域名设置为我们本机的 ip 地址

 

重新安装，开始升级

 

![](https://bbs.pediy.com/upload/attach/202112/905443_SYYV864XAEFDKNA.png)

 

然后升级手机被恶意软件劫持

 

![](https://bbs.pediy.com/upload/attach/202112/905443_3HYS3764EEKE2T6.png)

 

![](https://bbs.pediy.com/upload/attach/202112/905443_9C5SMYHZ6ZYRS97.png)

##### <2> 上海任意门科技有限公司 Soul APP 存在信息泄露漏洞

[上海任意门科技有限公司 Soul APP 存在信息泄露漏洞](https://www.cnvd.org.cn/user/myreport/4827046)

 

信息泄露原理很简单就是利用 http 明文传输，导致一些账户信息、登录信息的泄露，具体大家可以拿一个 http 传输的样本去测试，然后自己去查看一些信息问题

##### <3> 酷狗直播存在逻辑缺陷漏洞（hash 验证）

[酷狗直播存在逻辑缺陷漏洞（hash 验证）](https://www.cnvd.org.cn/flaw/show/4050696)

 

考虑到 http 明文传输的危害后，一些厂商开始加入 hash 验证，这也是我们前面讲述过的验证 Android 应用的完整性，因为每一个 Android APP 仅拥有唯一的 hash 值，但是这种我们可以在升级时，同时去替换相应的 hash 值来达到升级劫持漏洞的过程

 

![](https://bbs.pediy.com/upload/attach/202112/905443_4BTW8BP94YHKBFT.png)

 

我们可以发现在一些厂商的报文中会包含 hash 值验证，所以如果我们直接去注入恶意程序，我们的应用是安装不上去，但是我们对对应的报文进行替换 hash 值，替换成我们对于的恶意程序的 hash 值，我们就可以成功的复现上述升级劫持漏洞的过程

#### [](#（2）漏洞防护)（2）漏洞防护

通过上面分析，我们发现上述的漏洞都是因为厂家的 APP 在传输过程中采用了明文传输导致的，因此防护措施：

```
（1）HTTPs加密传输
（2）本地hash验证

```

### 3.HTTPs 密文传输漏洞

#### [](#（1）漏洞案例)（1）漏洞案例

##### <1> 忽略 SSL 证书校验漏洞

漏洞原理：

 

在自定义实现 X509TrustManager 时，checkServerTrusted 中没有检查证书是否可信，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害。由于客户端没有校验服务端的证书，因此攻击者就能与通讯的两端分别创建独立的联系，并交换其所收到的数据，使通讯的两端认为他们正在通过一个私密的连接与对方直接对话，但事实上整个会话都被攻击者完全控制。在中间人攻击中，攻击者可以拦截通讯双方的通话并插入新的内容

 

目标程序代码：

```
public class MyX509TrustManager implements X509TrustManager { 
// 检查客户端证书 
public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
    //没有校验的话，就代表接收任意的证书
} 
 
// 检查服务器端证书 
public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException { 
    //没有校验的话，就代表接收任意的证书
} 
 
// 返回受信任的X509证书数组 
public X509Certificate[] getAcceptedIssuers() { 
return null; 
} 
}

```

在重写 WebViewClient 的 onReceivedSslError 方法时，调用 proceed 忽略证书验证错误信息继续加载页面，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害

 

目标程序代码：

```
mywebview.setWebViewClient(new WebviewClient(){ 
 
@Override 
public void onReceivedError(WebView view,int errorCode,String description,String falingUrl){ 
 
    // TODO Auto generated method stub 
    super.onReceivedError(view,errorCode,description,fallingUrl) ; 
} 
 
@Override 
public void onReceivedSslError(WebView view,SslErrorHandler handler,SslError error) 
 
    // T0D0 Auto-generated method stub 
    handler.proceed( );  //不对证书进行处理
}; 
});

```

**案例一：京东金融 MITM 漏洞**

 

漏洞原理：

```
京东金融Ver 2.8.0由于证书校验有缺陷，导致https中间人攻击，攻击者直接可以获取到会话中敏感数据的加密秘钥，另外由于APP没有做应用加固或混淆，因此可以轻松分析出解密算法，利用获取到的key解密敏感数据

```

登录后捕获的数据：

 

![](https://bbs.pediy.com/upload/attach/202112/905443_X52FMQD7AY4DWRG.png)

 

![](https://bbs.pediy.com/upload/attach/202112/905443_6H4HPNQGXKZR293.png)

 

安全防护：

```
（1）建议自定义实现X509TrustManager时，checkServerTrusted中对服务器信息进行严格校验。 
（2）针对自定义TrustManager,检查checkServerTrusted()函数是否为空实现。 
（3）建议不要重写TrustManager 和HostnameVerifier,使用系统默认的。 
（4）在重写WebViewClient的onReceivedSslError方法时，避免调用proceed忽略证书验证。 
（5）禁止使用proceed()函数忽略证书错误，应该抛给系统进行安全警告

```

例如，我们在相应的地方加上校验：

 

![](https://bbs.pediy.com/upload/attach/202112/905443_UC5V7ZK9GAS2JBZ.png)

 

![](https://bbs.pediy.com/upload/attach/202112/905443_R47Z7JDY63E7YEC.png)

 

![](https://bbs.pediy.com/upload/attach/202112/905443_DWQ84T5A74B882X.png)

##### <2> 忽略域名校验漏洞

在自定义实现 HostnameVerifier 时，没有在 verify 中进行严格证书校验，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害

 

目标代码：

```
HostnameVerifier hv = new HostnameVerifier (){ 
    @Override 
    public boolean verify(String hostname,SSLSession session){ 
    return true; 
    } 
};

```

在 setHostnameVerifier 方法中使用 ALLOW_ALL_HOSTNAME _VERIFIER, 信任所有 Hostname, 导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害

 

目标代码：

```
private HttpClient getNewHttpClient() { 
try { 
    KeyStore trustStore = KeyStore.getInstance(KeyStore 
    .getDefaultType()); 
    trustStore.load(null, null); 
    SSLSocketFactory sf = new SSLSocketFactory(trustStore); 
    sf.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);  //这里信任了所以的hostname，导致可能存在中间人攻击
    HttpParams params = new BasicHttpParams(); 
    HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1); 
    HttpProtocolParams.setContentCharset(params, HTTP.UTF_8); 
    SchemeRegistry registry = new SchemeRegistry(); 
    registry.register(new Scheme("http", PlainSocketFactory 
    .getSocketFactory(), 80)); 
    registry.register(new Scheme("https", sf, 443)); 
    ClientConnectionManager ccm = new ThreadSafeClientConnManager( 
    params, registry); 
    return new DefaultHttpClient(ccm, params); 
} catch (Exception e) { 
return new DefaultHttpClient(); 
} 
}

```

**案例二：[WPS 存在信息泄漏漏洞](https://www.cnvd.org.cn/flaw/show/4069106)**

 

WPS 采用 HTTPs 进行通信但是由于证书校验问题，可以被获取到敏感信息，从而导致信息泄漏漏洞，这里和上面一致，就不再重新演示了

 

![](https://bbs.pediy.com/upload/attach/202112/905443_R9J9B4SYAH5ZXX3.png)

 

安全防护：

```
（1）在自定义实现HostnameVerifier时，在verify中对Hostname进行严格校验
（2）建议setHostnameVerifier方法中使用STRICT_HOSTNAME_VERIFIER进行严格证书校验，避免使用ALLOW_ALL_HOSTNAME_VERIFIER

```

##### <3> 作业帮存在 https 升级劫持漏洞

大家都知道 https 是采用加密方式来进行通信，一般来说除非证书的设置方面存在漏洞，否则很难直接去截获报文信息，但是在我挖掘漏洞的过程中，发现一个新的思路，可能这是很多厂商比较懒的原因，直接升级 https 后，传输的报文数据还是原来的数据，所以我们可以选择采用 http 旧版本的 APP，抓取明文信息，修改后，使用于新版的信息，也可以导致劫持的漏洞

 

当然这个过程中也需要解决接收方证书信任的问题，还需要模拟 https 的请求方式，这里可以使用 [stunnel 配置](https://bbs.pediy.com/thread-268459.htm)，这里主要提供一种思路，其他操作步骤和上述一直，就不再重复演示了

#### [](#（2）漏洞防护)（2）漏洞防护

针对于 Android https 的开发过程中常见的安全缺陷：

```
1)在自定义实现X509TrustManager时，checkServerTrusted中没有检查证书是否可信，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害。
2)在重写WebViewClient的onReceivedSslError方法时，调用proceed忽略证书验证错误信息继续加载页面，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害。
3)在自定义实现HostnameVerifier时，没有在verify中进行严格证书校验，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害。
4)在setHostnameVerifier方法中使用ALLOW_ALL_HOSTNAME_VERIFIER，信任所有Hostname，导致通信过程中可能存在中间人攻击，造成敏感数据劫持危害。

```

四、Android 绕过 https 的 SSL Pining
-------------------------------

我们在对 Android APP 抓包时，经常会出现 HTTPS 报文通过 MITM 代理后不被信任的问题，有些 https 在设置好证书后，会出现：

```
unknown
加密的乱码
报错无法抓包

```

这是因为对方的 https 采用了 SSL pinning

### 1.SSL pinning

SSL pining = 证书绑定 = SSL 证书绑定

 

表示对方的 app 只允许承认自己特定的证书，这导致 MITM 的证书不被识别，不运行，从而导致 MITM 无法解密看到 https 的明文数据

### 2.Android 7.0 后破解 https 的 ssl pinning

Android7.0 后，系统做了改动：

```
APP 默认不信任用户域的证书。之前把MITM的ssl证书，安装到 受信任的凭据 -> 用户 就没用了，因为不受信任了。只信任（安装到）系统域的证书

```

因此这导致我们使用如 Fiddler、Charles 等抓包软件导入证书后，仍然不能在捕获 https 的密文，甚至无法解析请求

 

解决思路：

```
（1）让系统信任Charles的ssl证书
    改自己的app的配置，允许https抓包，这就需要有app的源码
    把证书放到受系统信任的系统证书中去。前提是手机已root
（2）绕开https不去校验
    使用基于Xposed等框架的JustTrustMe、基于Frida框架的r0capyure等

```

### 3. 证书绕过原理

我们基于上文提出第二种思路，详细解析当下的 JustTrustMe 为代表的证书绕过原理

 

通过前面我们了解到，证书验证中到关键是 TrustManager，而绕过证书验证就需要从它入手。xpsoed 上证书校验的绕过插件就是这么干的，目前比较流行的两款基于 xposed 的绕过证书验证的模块有两款 JustTrustMe 和 SSLkiller，针对 HttpsURLConnection，OkHttp 框架各自的证书校验函数

 

这两款工具通过 hook 这些关键函数，或替换 TrustManager(信任所有证书) 或令其验证函数直接失效 (函数替换，不做任何校验)，以达到绕过的目的

 

绕过证书的实现原理图，下图参考博客[安卓 https 证书校验和绕过](https://juejin.cn/post/6992844908788711438)：

 

![](https://bbs.pediy.com/upload/attach/202112/905443_GQTKJ7GVBKAX8FB.png)

### 4. 工具使用

#### （1）Xposed+JustTrustMe

[JustTrustMe 下载地址](https://github.com/Fuzion24/JustTrustMe)

 

使用步骤十分简单，就在手机上安装 xposed 框架，具体安装参考前文帖子，然后将 JustTrustMe 模块安装就可以使用了

 

![](https://bbs.pediy.com/upload/attach/202112/905443_5N6SW2AWNKXQKU4.png)

#### [](#（2）frida脚本)（2）Frida 脚本

下面是两种比较火的 frida 抓包脚本

 

[frida-android-unpinning-ssl](https://codeshare.frida.re/@masbog/frida-android-unpinning-ssl/)

 

[r0capture](https://github.com/r0ysue/r0capture)

 

使用步骤：

 

开启 frida_server 注入脚本就可以了，具体可以参考博客网址

[](#五、实验总结)五、实验总结
-----------------

本文从 Android Http/Https 通信过程出发，讲述了 Android Http/Https 通信漏洞产生的原因，也拿了很多的漏洞复现实例来进行一一说明，最后还简单介绍了当下对 https 转包的处理和原因，当然这部分东西很多还需要进一步深入的研究，本文可能还未归纳完全所有的情况，就请大家指正了

[](#六、参考文献)六、参考文献
-----------------

Android http/https 原理解析：

```
第一行代码
Frida 逆向和抓包实战
https://zhuanlan.zhihu.com/p/330393659
https://xiaoyue26.github.io/2018/09/26/2018-09/%E9%80%9A%E4%BF%97%E7%90%86%E8%A7%A3SSL-TLS%E5%8D%8F%E8%AE%AE%E5%8C%BA%E5%88%AB%E4%B8%8E%E5%8E%9F%E7%90%86/

```

Android https 漏洞挖掘：

```
https://www.geek-share.com/detail/2727192403.html
https://www.cxyzjd.com/article/u010982507/85258477
https://www.jianshu.com/p/84df0a40127c

```

Android 证书绕过：

```
https://juejin.cn/post/6992844908788711438
https://www.jianshu.com/p/34912804bf08
https://www.panaihua.com/android-catchhttp/

```

[【公告】“雪花” 创作激励计划，3 月 1 日正式开启！](https://bbs.pediy.com/thread-271637.htm)

[#基础理论](forum-161-1-117.htm) [#漏洞相关](forum-161-1-123.htm) [#源码分析](forum-161-1-127.htm) [#其他](forum-161-1-129.htm)