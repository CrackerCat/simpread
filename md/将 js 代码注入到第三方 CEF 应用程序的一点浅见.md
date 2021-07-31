> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268570.htm)

> 将 js 代码注入到第三方 CEF 应用程序的一点浅见

高考后的第一次发帖 ^-^

前言
==

`CEF`是`Chromium Embedded Framework`的缩写，即 “Chromium 嵌入式框架”，采用 c++ 编写，地位类似于 Electron，是 web 开发应用程序的重要框架，被许多软件包括微信，网易云，生死狙击等采用，是一款十分优秀的嵌入式框架

 

由于一些原因，需要把 js 注入到某款使用 CEF 的应用中，通过 js 代码与应用 web 进行一些交互，于是乎研究了一下 CEF 这个框架，并分享注入 js 代码到第三方 CEF 应用的一些经验

CEF 应用典型特征
==========

文件目录特征很明显

```
cef.pak
cef_100_percent.pak
cef_200_percent.pak
libcef.dll

```

着手点
===

既然 CEF 功能核心全部都在 libcef.dll 里面，那么自然首选也是从这个 dll 入手

 

网上简单搜索 CEF 的教程后发现，CEF 的框架在基础 libcef.dll 的导出函数的基础上，增加了一层 c++ 类的实现，实现了 c2cpp 接口的对接

 

观察官方给出的 demo 发现，创建浏览器对象，最终会调用`libcef.dll!cef_browser_host_create_browser`来创建出浏览器

 

而一旦得到浏览器对象，我们就可以根据 CEF 框架中的头文件，非常方便的执行 js 代码了

 

所以很自然想到把`libcef.dll!cef_browser_host_create_browser`给 hook 掉，然后从中间偷出 `cef_browser_t*`对象

```
int cef_browser_host_create_browser(
    const cef_window_info_t* windowInfo,
    struct _cef_client_t* client,
    const cef_string_t* url,
    struct _cef_browser_settings_t* settings,
    struct _cef_request_context_t* request_context);

```

然而新的问题马上出现，`cef_browser_host_create_browser` 是使用异步的方法创建浏览器，因此`cef_browser_t*`对象只会在创建成功的回调函数中才能得到，于是我准备设法从`_cef_client_t* client`这个参数中得到回调地址，然后 inline hook 掉回调函数（直接改回调地址失败了，估计是地址不可写）

 

很不幸的是，虽然获取回调地址很顺利，但是不知什么原因，回调函数始终没有得到调用，因此此路不通，只能另辟蹊径

 

此时完全没有头绪，根本不知道该如何下手了，毕竟`cef_browser_t*`得不到，想执行 js 代码根本不可能，无聊之余打开了 ApiMonitor，想看看能否从应用的`libcef.dll`的调用情况中得到一点启发

 

ApiMonitor 监视自定义 dll 的调用情况也非常方便，设置监视的函数为`libcef.dll`中的全部函数  
![](https://bbs.pediy.com/upload/attach/202107/751959_ATA7RJDWQQ8YCUD.png)

 

然后启动应用，由于浏览器内核特性，应用自然也是多进程的，一个个查看后，其中一个进程调用的 api 引起我的兴趣

 

![](https://bbs.pediy.com/upload/attach/202107/751959_YWDJMT5E8N3J5VU.png)  
![](https://bbs.pediy.com/upload/attach/202107/751959_6ZVA49P8PHNRP7S.png)

 

`cef_v8context_get_current_context`这个函数的调用瞬间将思路打开  
看到 V8，那这个函数传参或者返回值多半和 Javascript 分不开了，再去 CEF 的源码里找找参数或者返回值的含义

```
///
// Returns the current (top) context object in the V8 context stack.
///
CEF_EXPORT cef_v8context_t* cef_v8context_get_current_context();

```

惊喜的发现居然没有参数，直接反汇当前 V8 引擎上下文中最前的 context

 

由于 browser 是与 V8js 引擎绑定的，所以也可以通过返回值`cef_v8context_t*`得到 browser 对象了

```
cef_browser_t* browser = NULL;
cef_v8context_t* my_cef_v8context_get_current_context() {
    cef_v8context_t* js_context = ori_cef_v8context_get_current_context();
    Print("my_cef_v8context_get_current_context js_context: %X!!\n", js_context);
    browser = js_context->get_browser(js_context);
    //Print("[!!!] browser = %X\n", browser);
    _cef_frame_t* frame = browser->get_main_frame(browser);
    //Print("[!!!] frame = %X\n", frame);
    CefString script = payload;
    CefString url = xorstr("app/wd").crypt_get();
    frame->execute_java_script(frame, script.GetStruct(), url.GetStruct(), 0);
    return js_context;
}

```

接下来的事也就水到渠成了，按照网上教程的方法，得到 browser 过后一切变得简单明了起来

 

而且写完后我发现这样的写法还有个好处，就是可以考虑到页面刷新的情况，因为浏览器一旦页面刷新，重启 js 虚拟环境，那么浏览器又会主动调取`cef_v8context_get_current_context`, 因此只要我们 hook 这个函数过后，所有的页面，不管是否刷新，我们都能够注入自己想要的 js 代码

总结
==

在时间有限，没有足够时间去完整系统学习一个框架的时候，例如这里的 CEF 框架，我们依然想快速的实现目标，就必须要开拓思路，不要始终拘泥于网上别人教授的正常 / 官方方法，有时另辟蹊径反而可以带来意料之外的惊喜

 

研究 CEF 的前大半部分时间，我始终困在网上一些无比老旧，东搬西凑，复制粘贴的教程中，边抱怨作者的脑瘫，边复制着脑瘫的代码，最后还实现不了脑瘫的功能，最后干脆还是只能放弃，进行独立思考，将成果分享给大家

参考
==

https://blog.csdn.net/gaga392464782/article/details/49490101?spm=1001.2014.3001.5502

 

https://zhuanlan.zhihu.com/p/29077686

 

https://stackoverflow.com/questions/54398446/how-do-i-add-a-access-control-allow-origin-to-a-requested-resource-with-a-cust

 

https://blog.csdn.net/gaga392464782/article/details/49490101?spm=1001.2014.3001.5502

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#基础知识](forum-41-1-130.htm) [#HOOK / 注入](forum-41-1-133.htm)