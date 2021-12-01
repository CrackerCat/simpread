> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270543.htm)

> [原创] 某视频 app(V15.7-V18.4) 的学习记录

**[原创] 某视频 app(V15.7-V18.4) 的学习记录**
===================================

        之前发过一个帖子:    [某视频 app(V15.7) 及 web 分析记录](https://bbs.pediy.com/thread-269480.htm)（[https://bbs.pediy.com/thread-269480.htm）](https://bbs.pediy.com/thread-269480.htm），不知不觉版本都到18+了，记录了下学习过程，技术交流用。).

        不知不觉版本都到 18 + 了，记录下学习过程，技术交流用。  

一、软硬件环境：

抖音 android (应用宝渠道)

IDA 7.5

Frida 14.2.2

Gda3.86

JEB

jadx-gui

unidbg

LineageOs 17.1 (android 10)

小米 8

二、流水账 (直接按时间顺序，有坑写坑)

**1.** 截包相关    

    这个看见有人说有个版本开始不能截了，我这边一直都是换证书的，没感觉有影响，估计我下的是盗版，碰到再看了。

2. **之前看到 so 都是 32 位的，后来都换成 64 位的了。**

#### 3. 网络请求相关

          之前就看到有使用 Cronet 模块，想看下实现过程，下过源码（[https://github.com/hanpfei/chromium-net](https://github.com/hanpfei/chromium-net)），一个是发现跟 app 使用的并不完全相同，

          再一个还要自己编译模块，就想到直接使用抖音的模块。

          先新建一个自己的测试 app，接着的工作就是搬代码了，直接导出所有的反编译代码，结合源码，首先就是这个包：

```
    com.ttnet.org.chromium.net

```

        ![](https://bbs.pediy.com/upload/attach/202112/44250_8MFTJRQUGSDN4BU.jpg)

        这里插下搬代码中的一些问题：

        首先就是反编译代码，几个工具（jeb ,jadx, GDA）各有优劣，要结合使用, jadx 得到的代码可读性更好，但是很多函数识别不出来，比如这种：

![](https://bbs.pediy.com/upload/attach/202112/44250_4275W95926EUUVX.jpg)

GDA 擅长处理疑难杂症，其它 2 个识别不了的代码，它这个都能识别，不过可读性不好，基本不能直接编译通过，需要看清楚逻辑后修复。  

再就是反编译代码中变量顺序问题，也会影响编译（初始化的变量放在后面了）：

![](https://bbs.pediy.com/upload/attach/202112/44250_HX5P3RQJWEGQQAD.jpg)

代码修复中有些类型的转换，比如 Bundle 类型变量为 null 的，可能被还原成了 int i=0; 然后条件判断的时候，这个也会出现编译错误。  

![](https://bbs.pediy.com/upload/attach/202112/44250_YH85HQ7ES5S7MHZ.jpg)

再就是 Map.Entry 这种，需要先用 object 遍历，再转换类型。

```
if (!StringsKt.contains$default((CharSequence) name, (CharSequence) "__MACOSX", false, 2, (Object) null)) {

```

kotlin 相关也要修改  

```
StringsKt.contains((CharSequence) str, (CharSequence) "?", false))

```

还有下面这种继承的

```
public final class FrontMethodFragment$onCreateFailed$1 extends Lambda implements Function0 { 
```

```
Lambda要带上Lambda 
```

参考这个 [cannot be inherited with different type arguments_PhilsphyPrgram 的博客 -CSDN 博客](https://blog.csdn.net/PhilsphyPrgram/article/details/118651271)

还有一些逻辑上的，少了 break 之类，就不是很好发现了，这个只有遇到后调试修复了。

最后还有麻烦的就是缺少异常处理，java 编译是强制要求处理异常的，找过资料，想编译时候忽略异常处理，发现是 JAVA 规范强制的，屏蔽不了，加这个也花了不少时间。

总之，3 个工具要结合使用，怎么方便怎么来。

搬过来后，直接调用测试：  

```
                // 初始化引擎
                CronetEngine.Builder myBuilder = new CronetEngine.Builder(getApplicationContext());
                CronetEngine cronetEngine = myBuilder.build();
 
                // 创建请求线程
                Executor executor = Executors.newSingleThreadExecutor();
 
                // 创建UrlRequest
                strUrl="http://10.0.0.217/about";
                UrlRequest.Builder requestBuilder = cronetEngine.newUrlRequestBuilder(
                        strUrl, new MyUrlRequestCallback(), executor);
                requestBuilder.addHeader("testHeader","testValue");
                UrlRequest request = requestBuilder.build();
 
                // 发起请求
                request.start();

```

```
class MyUrlRequestCallback extends UrlRequest.Callback {
    private static final String TAG = "ttttt MyUrlRequestCallback";
 
    @Override
    public void onRedirectReceived(UrlRequest request, UrlResponseInfo info, String newLocationUrl) {
        android.util.Log.i(TAG, "onRedirectReceived method called.");
        // You should call the request.followRedirect() method to continue
        // processing the request.
        request.followRedirect();
    }
 
 
    @Override
    public void onResponseStarted(UrlRequest request, UrlResponseInfo info) {
        //这个函数只会调用一次
        android.util.Log.i(TAG, "onResponseStarted method called.");
        // You should call the request.read() method before the request can be
        // further processed. The following instruction provides a ByteBuffer object
        // with a capacity of 102400 bytes to the read() method.
        request.read(ByteBuffer.allocateDirect(102400));
    }
 
    @Override
    public void onFailed(UrlRequest urlRequest, UrlResponseInfo urlResponseInfo, CronetException cronetException) {
 
    }
 
    @Override
    public void onReadCompleted(UrlRequest request, UrlResponseInfo info, ByteBuffer byteBuffer) {
        //这个会调用多次
        android.util.Log.i(TAG, "onReadCompleted method called.");
        // You should keep reading the request until there's no more data.
        request.read(ByteBuffer.allocateDirect(102400));
    }
 
    @Override
    public void onSucceeded(UrlRequest request, UrlResponseInfo info, String str) {
        android.util.Log.i(TAG, "onSucceeded method called.");
    }
}

```

测试发现本地服务器正常收到请求了，测试 app 也能正常收到回调：

![](https://bbs.pediy.com/upload/attach/202112/44250_8QGCCHE6AFBHRRS.jpg)

不过请求地址改成本地 https 的时候，发现崩溃，根据日志定位报错线程： 

![](https://bbs.pediy.com/upload/attach/202112/44250_VCY8KPWQ3BE6Q6N.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_5PTD7JMR8SDC6EX.jpg)

发现是 throw new UnsupportedOperationException("Method not decompiled 造成的，那就是 jadx 代码不全的问题，这个参考 GDA 补全。

不过这里也看出是证书校验问题，本地的证书不通过。

_// Certificate is not trusted due to non-trusted root of the certificate  
// chain.  
_static int _NO_TRUSTED_ROOT_ = -2;

这里又熟悉了下 https，查了下资料：

[HTTPS 请求的整个过程的详细分析_研究生生活、学习记录 -CSDN 博客_https 过程](https://blog.csdn.net/seujava_er/article/details/90018326)

最后，由于非对称加密的公钥可以在网络中传输，如何保证公钥传送到给正确的一方，这个时候使用了证书来验证。证书不是保证公钥的安全性，而是验证正确的交互方。

![](https://bbs.pediy.com/upload/attach/202112/44250_CCCQ98WXAY5GZGW.jpg)

从这个流程图来看，访问本地网站就是验证证书无效了。![](https://bbs.pediy.com/upload/attach/202112/44250_SXXGE7TZNTWYR48.jpg)

本地 web 服务器的 file.crt 文件：

![](https://bbs.pediy.com/upload/attach/202112/44250_5BD8J3EFHBQZDMR.jpg)

解密后：

<Buffer 30 82 02 6d 30 82 01 d6 02 09 00 b9 de ff e9 ef 4c 9e 70 30 0d 06 09 2a 86 48 86 f7 0d 01 01 05 05 00 30 7a 31 0b 30 09 06 03 55 04 06 13 02 43 4e 31 ... 575 more bytes>

Len：625

正好上面截图对应测试程序调试中的 certChain。

搞清楚这个过程后，后面再进行干涉就有切入点了。

到这里后，虽然可以调用 Cronet 了，但是发现跟抖音自己的接口调用路径其实是不同的，比如某个接口的堆栈：

![](https://bbs.pediy.com/upload/attach/202112/44250_D7YV4TNGRZ2CFRW.jpg)

明显不是上面用的调用方式，那接着就是根据这个补代码了。

![](https://bbs.pediy.com/upload/attach/202112/44250_8DP6GVWAEQKYU6Y.jpg)

熟悉了上面这种注释语法

![](https://bbs.pediy.com/upload/attach/202112/44250_NZCPU9RA8BRN7A3.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_GSX9VUJ3PMHTF7T.jpg)

最后找到下面这个：

![](https://bbs.pediy.com/upload/attach/202112/44250_S2JGKUC6DHXWQY2.jpg)

其实之前也看到过这个 url，但是关联不到怎么调用的，现在看是通过代理类方式使用的（$Proxy84），这里是这个功能的类定义文件，

单纯静态看确实不好看明白，通过动态调试才搞清楚整个过程。

看见有鸿蒙相关：

![](https://bbs.pediy.com/upload/attach/202112/44250_AA7P8TRCBWHQTCF.jpg)

这里遇到个自己挖的坑，因为用了混淆的包名，导致 hash 这里跳过了，返回 null 了，会导致创建 proxy 类不成功：

```
private  T getStaticServiceImplReal(Class cls) {
     PatchProxyResult proxy = PatchProxy.proxy(new Object[]{cls}, this, changeQuickRedirect, false, 95671);
     if (proxy.isSupported) {
         return (T) proxy.result;
     }
  
     int iHashCode=cls.getName().hashCode();
     switch (iHashCode) { 
```

![](https://bbs.pediy.com/upload/attach/202112/44250_SU7AKSJKBJWD2CZ.jpg)

补完各种代码后（中间确实各种报错，各种补文件，最后备份的时候，发现有 3000 + 文件，不过有的模块，比如 wx 相关的基本拷过来就能用了，zfb 的交叉太多了，花了不少时间修复，感觉这样下去，可以编译一个抖音出来了），使用下面方式调用接口：  

```
FeedActionApi.f71204b.diggItem("7018000130007633191", "7018000130007633191", 1, 0).get();

```

运行调试：

![](https://bbs.pediy.com/upload/attach/202112/44250_USHZQQHTZNEJEJE.jpg)

看起来点赞操作的类是创建成功了，跟 hook 看到的堆栈类似了：

这里查了下代理类的资料：

[Java 动态代理之 InvocationHandler 最简单的入门教程 - 简书 (jianshu.com)](https://www.jianshu.com/p/e575bba365f8)

![](https://bbs.pediy.com/upload/attach/202112/44250_VR4TD4X2RQWHPGR.jpg)

创建代理类相关：

![](https://bbs.pediy.com/upload/attach/202112/44250_A5QFVV5JGYXYNUR.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_BZQC2J8CN2WXWQB.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_CTMBRT6UJ73ZGY9.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_W746Y926WJPA4TR.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_BGU9GCDVXM9ED6W.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_3GBKGC4YAJS3VSH.jpg)

这个选择不同的网络模式，模拟器默认走第二种

```
        if (C9859c.m21219a()) {
            jSONObject.put("netClientType", "CronetClient");
        } else {
            jSONObject.put("netClientType", "TTOkhttp3Client");
        }

```

到这里后，终于在测试服务器收到请求了：

![](https://bbs.pediy.com/upload/attach/202112/44250_ZJRSS3TUN849894.jpg)

到这里的时候，发现极速版更新到 18.2 了

Cront 相关 Native 函数引入有变化:

![](https://bbs.pediy.com/upload/attach/202112/44250_UXCHJQXUGDU6QY9.jpg)

输出 jni 函数的时候，发现有 SPDY 相关的：

![](https://bbs.pediy.com/upload/attach/202112/44250_E3Q2GNN6TBJTHJN.jpg)

顺便查了下 SPDY 相关资料：

[HTTP 的前世今生：一次性搞懂 HTTP、HTTPS、SPDY、HTT_请求 (sohu.com)](https://www.sohu.com/a/275505518_497161)

![](https://bbs.pediy.com/upload/attach/202112/44250_RT7S2HRKWSSEEFX.jpg)

![](https://bbs.pediy.com/upload/attach/202112/44250_YTGVUHUBS2BEP58.jpg)

后面看来要看下这个了。

在 16.6 版本都测试程序上替换上 18.2 版本都 so，测试报错：

![](https://bbs.pediy.com/upload/attach/202112/44250_A4CKT4XU7PVWVJH.jpg)

看起来 native 函数引入混淆了：

![](https://bbs.pediy.com/upload/attach/202112/44250_2SZHW5V95HSQ3MM.jpg)

包装了一层：

![](https://bbs.pediy.com/upload/attach/202112/44250_MPS2DVNPETJ5NYR.jpg)

对应修改后，就可以正常编译运行了。

看到检查 root 相关代码：

![](https://bbs.pediy.com/upload/attach/202112/44250_5MAQ7BACAG7VFGW.jpg)

算起来这次搬代码，最难的是开始时候，加进来的代码牵扯其它引用，一堆编译错误，加入引用，可能又会引入新的引用依赖，很容易耐心磨没了，这个时候就需要权衡取舍，不能把分支展得太开，还好坚持下来了，慢慢框架搭起来后，很多就是正向工作了。

学习总结：  

1.  学习了 JAV 代理类使用，大厂设计模式。
    
2.  熟悉了 Cronet 模块。
    

感觉主要工作都是正向的，正向逆向不分家，有了正向的工作，逆向切入点也会更多。

[【公告】【iPhone 13、ipad、iWatch】11 月 15 日中午 12：00，看雪 · 众安 2021 KCTF 秋季赛 正式开赛【攻击篇】！！！文末有惊喜～](https://bbs.pediy.com/thread-270218.htm)

[#逆向分析](forum-161-1-118.htm)