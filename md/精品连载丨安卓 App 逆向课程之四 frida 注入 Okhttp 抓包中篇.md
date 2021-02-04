> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/PICqN6K_LFGHkjyiXkPzUw)

本篇文章接上篇。

### 2. Okhttp3 自吐抓包

我们将一次请求的 request 大致结构罗列如下。  

• 请求方法 GET、POST、PUT、DELETE、HEAD 等 •URL• 使用的协议版本 HTTP/1/1.1/2• 多个请求 Header• 回车、换行符 • 请求 Body 数据

如果通过 Hook 的方式实现另类的 “抓包”，我们的需求是保留 URL，请求 Body，以及 headers。至于协议版本等可有可无。目前国内应用使用的 HTTP 协议版本大多数都是 HTTP/1.1，HTTP/2 虽然出来有段时间了，但国内使用的还比较少，这些字段对我们无帮助。

#### 2.1 Hook request 的时机

先不考虑 response 如何获取，单纯想一下如何 hook request，即使不深入了解 Okhttp 框架，通过断点调试一步步走，大概也能找到几个看着不错的点。

#### 2.1.1 requests 构建过程

```
// 构造request
Request request = new Request.Builder()
    .url(url) // Hook url()方法，此处只能得到单纯url，不妥
    .header(key, value) // Hook header()方法，一个request会多次调用此方法，会造成干扰，不妥。
    .header(key, value)
    ...
    .build();// Hook build()方法，一个request只可能调用一次，且它是构造完成时调用，不会遗漏信息。

```

DEBUG 断点调试

DEBUG url()

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0XvNI9bO8OGfK03jw9Tqdc1bTLtZtQPzvM2bew5vbsLLCMyblM5Mgvw/640?wx_fmt=png)

可以得到 url，其余均获取不到。

DEBUG header()

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0HpqAOG2WXkXg1l1dQlVTS8hfT58VCQjRnh5Gsg77zNYts50iavA0lGw/640?wx_fmt=png)

大家会发现，header 会被调用数次，尽管我们并没有配置 header，但 Okhttp 会检测并帮我们填补许多 header 字段，因此这个点也行不通。

诸如此类，DEBUG build() 方法，会发现每次请求，build 方法都会被调用两次，同样不适合。

上述问题是由 Okhttp 的拦截器机制造成，后续会有篇幅讲解，除此之外，构建一个 request 并不代表发送一次请求，这不是一个概念，因此我们要继续往下找。

查看 OkHttpClient.newCall(request) 方法源码

```
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

```

直接 Hook newCall 方法行不行，当然是可以的。

直接上 frida 脚本

```
Java.perform(function () {
    var OkHttpClient = Java.use("okhttp3.OkHttpClient");
    OkHttpClient.newCall.implementation = function (request) {
        var result = this.newCall(request);
        console.log(request.toString());
        return result；
    };
});

```

启动`Frida server`

```
C:\Users\Lenovo>adb shell
root@OXF-AN10:/ # cd /data/local/tmp
root@OXF-AN10:/data/local/tmp # ./frida-server

```

Hook 结果如下：

```
C:\Users\Lenovo>frida -U com.r0ysue.learnokhttp explore -l C:\Users\Lenovo\Desktop\抓包\teach\1.js
     ____
    / _  |   Frida 12.8.14 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
[OXF AN10::com.r0ysue.learnokhttp]-> 
Request{method=GET, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111, tags={}}
Request{method=GET, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111, tags={}}
Request{method=GET, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111, tags={}}
Request{method=GET, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111, tags={}}
Request{method=GET, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111, tags={}}

```

实际上，Hook `request`的问题远没有完美解决，此`Hook`点同样可能遗漏或多出部分请求，因为存在 "call" 后没有发出实际请求的情况。

`newCall(Request)`方法调用了`RealCall.newRealCall()`方法：

```
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
// Safely publish the Call instance to the EventListener.
RealCall call = new RealCall(client, originalRequest, forWebSocket);
call.eventListener = client.eventListenerFactory().create(call);
return call;
}

```

在 RealCall.newRealCall() 中，创建了一个新的 RealCall 对象，RealCall 对象是 Okhttp3.Call 接口的一个实现，也是 Okhttp3 中 Call 的唯一实现。它表示一个等待执行的请求，它只能被执行一次，但实际上，到这一步，请求依然可以被取消。因此只有 Hook 了 execute() 和 enqueue(new Callback()) 才能真正保证每个从 Okhttp 出去的请求都能被 Hook 到，不多也不少。

除此之外，割裂开请求和相应，分开去找 Hook 点的想法是有问题的，只能看到 request，但无法同时看到返回的相应，无法对爬虫工程师或协议分析过程产生有益的帮助。

#### 2.2 Okhttp 拦截器

拦截器是 Okhttp 中重要的一个概念，Okhttp 通过 Interceptor 来完成监控管理、重写和重试请求。Okhttp 本身存在五大拦截器，每个网络罗请求，不管是 GET 还是 PUT/POST 或者其他，都必须经过这五大拦截器。拦截器可以对 request 做出一定修改，同时对返回的 Response 做出一定修改，因此 Interceptor 是一个绝佳的 Hook 点，可以同时打印输出请求和相应。

为了帮助大家理解 Interceptor 机制，我们从三个方面去窥其一角。

#### 2.2.1 拦截器整体概览

拦截器可以对 request 做出修改，在数据返回时，再对 response 做出修改，这种说法可能会让人不知所云，引用刘望舒的演示：

![](https://mmbiz.qpic.cn/mmbiz_jpg/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB07CzusSEe9Nd6LqPXgfSAGRGibaWORjDHaCic2KIyNBOibJmoibGgt6515g/640?wx_fmt=jpeg)

引用 yuashuai[1] 所举的例子，他表述的生动且贴切。

> 老张有很多干面条，但是他想吃汤面，可是自己又不会做，但是碰巧村里大郎会做，于是老张拿一包干面条让大郎做成了汤面。但是老张发现他做面不好吃，盐都没放，连个青菜叶子都没有。
> 
> 这时候老张正好碰到隔壁老王，老王说了这东西我也会做，比他做的好吃多了。于是老张又拿着一包干面条给了老王，老王说老张你等着，我马上回家给你做，做好了就给你送过去。但是老王回家并没有做，而是去家里拿了一包盐，然后去找隔壁老李了，原来老王并不会做面，但是他知道隔壁老李会做，而且做得比较好吃。于是他把干面条和盐都交给了老李。老李对老王说你回去等着吧，做好了马上给你送过去。可是老李同样不会做，但是他知道村里的大郎会做，这时老李首先回厨房拿了两个生菜叶子，然后带着老王给的干面条和盐去找大郎了，对大郎说，生菜叶子，盐，面条都给你了，你快给我做一碗面。大郎对老李说好嘞，3 分钟就好了，3 分钟后，老李拿着做好了的放了盐和生菜叶子的一碗面回去了。本来打算直接给老王，但是一想，自己放了两个生菜叶子，不吃点这个面吃不是有点亏，于是老李偷偷了吃了几根面。然后老李去找老王说你的面做好了并把面交给了老王。老王一看这面只有两个青菜叶子，营养是不是不够呀！于是老王又买了半斤熟牛肉，切切放了进去。然后老王去找老张说你的面做好了，还说道这么大一碗你也吃不完吧，让小张也吃点。最后老张吃着老王送来的红烧牛肉面感动的肉牛满面。
> 
> **这里的干面条就可以看做一个最原始的 request，到老王哪里被加了点盐，到老李哪里被加了生菜叶子，于是大郎才能把这个 request 做成放了盐和生菜叶子的 response，这个 response 回到老李那里又被啃了几口，到老王那又被放了点牛肉。于是最后回到老张哪里收到的 response 就是被啃了几口并且加了牛肉的 response。这样整个链条是不是就清楚了！**

在 Okhttp 代码中，由 getResponseWithInterceptorChain 方法展示这个过程：

```
Response getResponseWithInterceptorChain() throws IOException {
    // 空的拦截器容器
    List<Interceptor> interceptors = new ArrayList<>();
    // 添加用户自定义的应用拦截器集合(可能0，1或多个)
    interceptors.addAll(client.interceptors());
    // 添加retryAndFollowUpInterceptor拦截器，该拦截器用于取消、失败重试、重定向
    interceptors.add(retryAndFollowUpInterceptor);
    // 添加BridgeInterceptor拦截器，对于Request而言，该拦截器把用户请求转换为 HTTP 请求；对于Resposne，把 HTTP 响应转换为用户友好的响应
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 添加CacheInterceptor拦截器，该拦截器读写缓存、根据策略决定是否使用
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //该拦截器实现和服务器建立连接
    interceptors.add(new ConnectInterceptor(client));
    //如果不是WebSocket请求，添加用户自定义的网络拦截器集合(可能0，1或多个)
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    // 添加真正发起网络请求的Interceptor
    interceptors.add(new CallServerInterceptor(forWebSocket));
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
    return chain.proceed(originalRequest);
  }
}

```

参照下图理解整体理解拦截器链：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0TiaBgPjknqQfRXNliaMWPOgtQ5gDE8Zvo1HevFZLpy8ic7sADYDTMJb1Q/640?wx_fmt=png)

#### 2.2.2 动手添加用户拦截器

新建 LoggingInterceptor 类，实现 Interceptor 接口，这代表它是一个拦截器，接下来实现 intercept 方法，我们的拦截器会打印 URL 和请求 headers，完整代码如下。

LoggingInterceptor.java

```
package com.r0ysue.learnokhttp;
import android.util.Log;
import java.io.IOException;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;
class LoggingInterceptor implements Interceptor {
    // TAG即为日志打印时的标签
    private static String TAG = "learnokhttp";
    @Override public Response intercept(Interceptor.Chain chain) throws IOException {
        Request request = chain.request();
        Log.i(TAG, "请求URL："+String.valueOf(request.url())+"\n");
        Log.i(TAG, "请求headers："+"\n"+String.valueOf(request.headers())+"\n");
        Response response = chain.proceed(request);
        return response;
    }
}

```

接下来需要将我们自定义的 LoggingInterceptor 拦截器添加到拦截器链中，这部分需要顺着 1.3.1 中的 OkhttpClient 对象继续讲。用户自定义的拦截器就是在此处添加。

用户拦截器有两种

• 应用拦截器 Application Interceptors• 网络拦截器 Network Interceptors

具体区别可以看官网 [2], 对于我们的需求而言，两者都可，但网络拦截器更好，具体原因有心的读者可以去对照一下。看一下相应代码的修改：

在 example.java 中

```
// 新建一个Okhttp客户端
// OkHttpClient client = new OkHttpClient();
// 新建一个拦截器
OkHttpClient client = new OkHttpClient.Builder()
        .addNetworkInterceptor(new LoggingInterceptor())
        .build();

```

大家会发现，OKhttpclient 的创建方式改变了，这里讲一下 Okhttpclient 三种创建方式，之所以存在这三种创建方式，和 Okhttpclient 本身的原则息息相关。

创建方法一

```
OkHttpClient client = new OkHttpClient();

```

其内部即默认配置大量参数

```
public OkHttpClient() {
   this(new Builder());
 }
public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      //...
      //...
      //...
    }
#此即Okhttp默认配置
OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    this.protocols = builder.protocols;
    this.connectionSpecs = builder.connectionSpecs;
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    this.eventListenerFactory = builder.eventListenerFactory;
    this.proxySelector = builder.proxySelector;
    //...
    //...
    //...
  }

```

Okhttp 框架帮我们默认所有配置，因此无法自定义添加用户拦截器。

创建方法二

```
OkHttpClient client = new OkHttpClient.Builder()
    .addNetworkInterceptor(new LoggingInterceptor())
    .build();

```

此即采用建造者 (Builder) 模式，可以自定义配置所有参数，我们在这儿添加了一个拦截器。

我们使用方式二验证一下拦截器是否生效，然后再讲解为什么需要第三种创建方式。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0DKs2bJ69VHGZzMBiaIZPoOovGbasAAFfia8kYXU3MB08JqvR7S4kp0yQ/640?wx_fmt=png)

输出内容有三行，第一行是 URL，第二行是 headers，需要注意，第三行的 response 并非由我们拦截器中打印出来，而是之前代码的作用，之所以不在拦截器中顺带打印 response，是因为在此时，response 对象的打印稍有不便，我们在后续提它。

接下来讨论第三种 client 创建方式，它会在原先的 client 基础上创建一个新的 okhttp 客户端。

创建方式三

```
// 此为原先的client
OkHttpClient client = new OkHttpClient();
// 基于原先的client创建新的client
OkHttpClient newClient = client.newBuilder()
    .addNetworkInterceptor(new LoggingInterceptor())
    .build();

```

newclient 和 client 共享连接池、线程池，且 newclient 继承 client 原先的配置。为什么我们需要它呢？

———通过这种方式，我们可以配置出一个用于处理某一类特定需求的 client。

打个比方，一个新闻客户端，主要提供如下三个接口：

•/xxx/xxx/news ——浏览新闻 •/xxx/xxx/comments ——查看评论 •/xxx/xxx/login ——登录

可以将 Okhttpclient 想象成网络通信工厂，根据需求不断生产这三类请求，但登录接口由于涉及到用户信息，我们需要额外的防护，最好通过一个拦截器给 request 增加一个 sign 验证。在这种情况下，我们就可以使用第三种方式后创建 loginClient, 此工厂会为每个请求加上 sign 验证。那为什么不重新创建一个全新的 client？一是因为每个 App 的网络通信都有许多默认配置，比如 host，固定的 sign，或者延迟等待时间等等。如果重新创建一个 client，同样需要设置和添加这些配置，比较繁琐。二是为了提升性能，减少内存消耗。通过 newBuilder 方式创建的新 client，和原 client 共享连接池、线程池等 “基础设施”。

你可能会有疑问，为什么要在意这一点消耗呢？从我们的 DEMO 代码可以发现，我们每次点击 “发送请求” 按钮，都会创建一个新的 client，既然每点击一次都创建一个新的客户端，何必在意 newbuilder 省下来的那点性能呢？

其实不然，在演示 DEMO 时，我们忽略了性能的问题，其实 Okhttpclient 应该被设置为单例模式，即 App 全局只使用一个共享的 OkHttpClient 实例，将所有的网络请求都通过这个实例处理。因为每个 OkHttpClient 实例都有自己的连接池和线程池，重用这个实例能降低延时，减少内存消耗，而重复创建新实例则会浪费资源。

换而言之，我们的 DEMO 每次点击都创建一个新的实例，相当于每个生产车间只生产一个布娃娃，因此会造成极大的浪费，甚至会造成内存的溢出。

Okhttp 官方并没有在框架中强制 OkhttpClient 全局单例 (可能是出于让开发者更灵活和自由的缘故)，但强烈建议非必要的情况下，全局共享一个 OkHttpclient(网络访问框架一般都需要单例模式)。

接下来我们使用 Objection 来验证，DEMO 中是否存在滥创 Okhttpclient 的现象。(显而易见，我们单纯熟悉一下 Objection 操作)。首先关闭并重新打开 App，不做任何操作，创造一个干净的环境。

objection 是一个基于 Frida 开发的命令行工具，它可以很方便的 Hook Java 函数和类，并输出参数，调用栈，返回值。只需一行命令就可以完成 Hook。Objection 的简单使用可以看这篇 [3]。

开启 server，安装 objection 后，使用 Objection Hook 我们的 DEMO app：

```
C:\Users\Lenovo>objection -g com.r0ysue.learnokhttp explore

```

正常情况展示如下：

```
C:\Users\Lenovo>objection -g com.r0ysue.learnokhttp explore
A newer version of objection is available!
You have v1.8.4 and v1.9.2 is ready for download.
Upgrade with: pip3 install objection --upgrade
For more information, please see: https://github.com/sensepost/objection/wiki/Updating
Using USB device `OXF AN10`
Agent injected and responds ok!
     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.8.4
     Runtime Mobile Exploration
        by: @leonjza from @sensepost
[tab] for command suggestions
com.r0ysue.learnokhttp on (HUAWEI: 5.1.1) [usb] #

```

使用 Objection 搜索堆中 okhttp3.OkHttpClient 实例，命令如下：

```
android heap search instances okhttp3.OkHttpClient

```

```
com.r0ysue.learnokhttp on (HUAWEI: 5.1.1) [usb] # android heap search instances okhttp3.OkHttpClient
Class instance enumeration complete for [32mokhttp3.OkHttpClient[39m

```

结果为空，点击数次 “发送请求” 按钮后，再次尝试

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0IxCIo4ndyoxKNQibmgrx0CLvh2muOtFhIP32JNv7eT1ByF0ce8etbTQ/640?wx_fmt=png)

每次 click 后，堆中就会多出一个新的 client 实例，这是极大的浪费，除此之外，如果读者显示结果有误差，堆中实例并无增加，添加 --fresh 参数重试。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0JJicoOat0rh0qxGywVIC0y0FRd34rKelcvKT7hZmtfuJ2J5NDgoYxIw/640?wx_fmt=png)

#### 2.2.3 BridgeInterceptor 拦截器讲解

以`BridgeInterceptor`为例，截取一段代码：

```
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }
    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

```

如果我们并不对 header 中的相应字段做设置，Bridge 拦截器会为其添加一些默认值，从之前的抓包对比也可以看出，当我们没有添加 user-Agent、Host，Accept-Encoding 等字段时，Okhttp 会为我们自动添加这些信息。

本篇文章先到这里，下一篇我们会介绍另外两个拦截器—— Yang Okhttp 拦截器和天外飞仙拦截器的实现，敬请期待。

### References

`[1]` yuashuai: _https://blog.csdn.net/qq_16445551/article/details/79008433_  
`[2]` 官网: _https://square.github.io/okhttp/interceptors/#application-interceptors_  
`[3]` 这篇: _https://www.anquanke.com/post/id/197657_

怎么样？如果大家对安卓逆向感兴趣，想学到更多的知识，或者想与肉丝姐进一步交流的话，欢迎加入肉丝姐的星球来学习。

这里我跟肉丝姐还申请到了专属的半价（原价 50 元）优惠，一杯咖啡的钱大家就能学到更多关于安卓逆向的知识，感兴趣的朋友来扫码加入吧。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7G3vcLPYLZnBlHySsX5WyXkqpNQYPicAbXVfy2lNKict2cmqqWF8SwoibTiacd9eQiczvuHLEPIwI7gSiaA/640?wx_fmt=png)