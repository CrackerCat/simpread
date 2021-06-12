> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268054.htm)

> [原创]Uni-App 逆向分析

Uni-App 逆向分析
------------

一般 App 数据加密分为 Java 层跟 So 层，但对于 H5App 来讲的话，它是界面里嵌入一个 WebView，在 WebView 里显示网页，在网页里面用 JS 加密，在通过 Java 和 JS 交互，用 Java 提交数据或者直接用网页提交数据两种方式

### 判断 App 是否为 H5App

可以利用 uiautomatorviewer.bat 来查看界面信息

 

![](https://bbs.pediy.com/upload/attach/202106/918604_CQUPK3VUT387WR9.png)

### JS 代码存放位置

大多数 H5App 的 JS 文件都存放在 assets 目录，少部分会存在 res 目录，存放在这两个地方获取比较方便，放在别的地方比如 classes.dex 的话获取比较麻烦，并且对 classes.dex 还要做一个处理

 

还有一个点就是 JS 文件是可以加密的，只要在加载到 WebView 之前进行一个解密就行

### WebView 相关的几个关键词

```
setWebContentsDebuggingEnabled    // 是否允许调试
// shouldInterceptRequest、WebResourceResponse
public WebResourceResponse shouldInterceptRequest(WebView p0,String p1)    // 拦截器 jadx会反编译不出来建议用jeb或者gda 里面可以对js文件做一些处理
WebResourceResponse shouldInterceptRequest(WebView webView, WebResourceRequest webResourceRequest)
WebResourceResponse shouldInterceptRequest(WebView webView, String str)

```

### 远程调试 WebView

远程调试需要满足三个条件:

1.  **Chrome WebView** 版本要大于手机端 **WebView** 版本 (chrome 更新到最新版就能解决)
2.  **inspect** 打开空白 则需要下载离线包
3.  如果连 **WebView** 下的一些信息都没显示出来则代表你没有调试权限 `setWebContentsDebuggingEnabled` 这个的值必须为`true`才有调试权限, App 发布一般都会把这个值设置成假不让你调试, 直接 Hook 掉就行

![](https://bbs.pediy.com/upload/attach/202106/918604_U4A8MG5UMP9NDVS.png)

 

`setWebContentsDebuggingEnabled` 三个检测点:

 

![](https://bbs.pediy.com/upload/attach/202106/918604_QYAKFM5Z9U626EW.png)

1.  chrome 浏览器调试 地址 **chrome://inspect**
2.  **Devices** 下两个选项都勾选上
3.  浏览器版本 > 手机端 **WebView** 版本
4.  点击 **WebView** 下方的 **inspect** 开始调试

PS: 由于手机端 **WebView** 版本不一致, 所以第一次运行需要从谷歌站点下载一系列的离线包, 否者打开 **DevTools** 就是空白界面 (科学上网会自动下载)

 

[Android 远程调试 WebView 的方法](https://zhuanlan.zhihu.com/p/95740830)

 

[Android 设备 WebView 远程调试](https://blog.csdn.net/dong123dddd/article/details/70217248)

### 逆向分析

#### 1. **Network** 能抓到数据 (网页发包)

**inspect** 开始调试 , 切换到 **Network** 选项卡抓取数据 如果有抓取到数据也为网页发包 否则就是 Java 层发包 这个 app 是网页发包的

 

![](https://bbs.pediy.com/upload/attach/202106/918604_6VCJAZZPQMPP4KM.png)

 

抓到的数据包 函数调用栈 接下来就是 Js 逆向分析 断点调试一顿分析就完事了

```
// 提交的数据
appid=quickdogrestful&mobile=139xxxxxxxx&password=a12345678&device_id=863254033385807%2C863254033385815&device_info=MI%206%20Xiaomi&app_version=1.0.3.4&nonce=a×tamp=1623410323.775&sign=573080b9042317ee8b30ab6411c9b56e

```

![](https://bbs.pediy.com/upload/attach/202106/918604_D9QRFFZ5RRFAET6.png)

 

![](https://bbs.pediy.com/upload/attach/202106/918604_SGKUM6WPY887B7E.png)

#### 2. **Network** 无法抓到数据 (不是网页发包)

这种大概率就是 JS 加密 然后 Java 发包的 就只能找参数静态分析了

```
// 抓包数据
POST //api/user/login.do HTTP/1.1
user-agent: Mozilla/5.0 (Linux; Android 9; MI 6 Build/PKQ1.190118.001; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/74.0.3729.136 Mobile Safari/537.36 uni-app Html5Plus/1.0 (Immersed/24.0)
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Content-Length: 138
Host: app168.zhongjianlepai.com
Connection: Keep-Alive
Accept-Encoding: gzip
Cookie: aliyungf_tc=b05125519e32b58949efe39018fbd902f0c0c052f4429ff19bd0ba4172416ff6; JSESSIONID=A8B840C693E48003233EE93EED504656; eiis-sessionid=3f17f8ce-4735-4670-ad2f-6036da767e52
 
phoneNum=139xxxxxxxx&pwd=a12345678&isTakecookie=false&appVersion=1.2.2×tamps=1623410775656&signature=21549323b5116195020eee9808de314b

```

解压 App 一顿分析 **app-services.js** 定位关键函数

 

![](https://bbs.pediy.com/upload/attach/202106/918604_KRFNUM9FJ25T5BV.png)

 

![](https://bbs.pediy.com/upload/attach/202106/918604_E4MC3KHXCV5C3JY.png)

 

修改为下面代码保存 Js 文件 然后覆盖进 apk 文件里 功能 ->APK 签名 -> 安装 再抓包

 

跟之前的 URL 对比 就可以查看我们提交的参数

```
r.login = function(t, e, s) {
 t["appVersion"] = a.default.APP_VERSION,
 i.default.request("/api/user/login.do?"+ JSON.stringify(t) + JSON.stringify(e) + JSON.stringify(s) , "POST", t, e, s)}

```

![](https://bbs.pediy.com/upload/attach/202106/918604_RVQVWGR5FQAC89S.png)

 

![](https://bbs.pediy.com/upload/attach/202106/918604_VHJYWGXB263T77M.png)

 

![](https://bbs.pediy.com/upload/attach/202106/918604_EAS9GWU8PE5DE83.png)

```
POST //api/user/login.do?{%22phoneNum%22:%22139xxxxxxxx%22,%22pwd%22:%22a12345678%22,%22isTakecookie%22:false,%22showLoading%22:true,%22loaddingText%22:%22%E7%99%BB%E9%99%86%E4%B8%AD...%22,%22appVersion%22:%221.2.2%22}undefinedundefined HTTP/1.1
 
t  = {"phoneNum":"139xxxxxxxx","pwd":"a12345678","isTakecookie":false,"showLoading":true,"loaddingText":"登陆中...","appVersion":"1.2.2"}
 
e,s = undefined

```

这串数据提交还没有包含 `signature` 那就继续查找下一个关键点 全局搜索`signature` 查找关键函数 最后抓包对比

```
// 传进去的参数
{"phoneNum":"139xxxxxxxx","pwd":"a12345678","isTakecookie":false,"showLoading":true,"loaddingText":"登陆中...","appVersion":"1.2.2"}
 
// 经过排序加salt 放入n.default(i)处理MD5
appKey=syc0049ec3d91b9028&appVersion=1.2.2&phoneNum=139xxxxxxxx&pwd=a12345678×tamps=1623410775656&key=2e4dea7ba1dc4d1991cbcdd7756e548e
 
MD5：21549323b5116195020eee9808de314b
 
// 上面的抓包数据做对比
phoneNum=139xxxxxxxx&pwd=a12345678&isTakecookie=false&appVersion=1.2.2×tamps=1623410775656&signature=21549323b5116195020eee9808de314b

```

![](https://bbs.pediy.com/upload/attach/202106/918604_6XFBD34AJHY6HC5.png)

 

![](https://bbs.pediy.com/upload/attach/202106/918604_2HDBENESS4KXZAH.png)

#### uni-app 核心请求

关键词 uni.request

 

![](https://bbs.pediy.com/upload/attach/202106/918604_6NCGH7Z7HBQSBYS.png)

 

最后总结一下: **Network** 能抓包的话可以动态调式 分析起来方便一点 不然的话就只能 静态分析 + 猜测 修改 H5 代码 签名 App 抓包来分析

 

第一次发帖 记录一下自己的学习过程，有不对的地方还请大佬指正

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 16 小时前 被 mb_yzrdfloq 编辑 ，原因：