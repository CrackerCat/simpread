> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268651.htm)

> 短视频最新版通用 quic 协议解决方案

短视频最新版通用 quic 协议解决方案
====================

看到很多人在说短视频新版 app 抓不到包，这里接提供解决方案。  
由于最新版的两款短视频都使用了 quic 协议，这就导致爬虫小伙伴在抓包的过程遇到不能抓包的问题，这里提供他们 quic 协议所有版本的通用解决方案，使他们不使用 quic 协议，直接通过 Charles 抓包。  
由于不能透露 app 名字，那就按活跃度说吧。

### quic 协议是什么

QUIC（Quick UDP Internet Connection）是谷歌制定的一种互联网传输层协议，它基于 UDP 传输层协议，同时兼具 TCP、TLS、HTTP/2 等协议的可靠性与安全性，可以有效减少连接与传输延迟，更好地应对当前传输层与应用层的挑战。

### 最火的短视频 app

对于该 app 禁止 quic 相关的 so 加载即可，记得没错的话该 so 是`lib**cronet.so`, 通过 ida 也可以看到相关函数名。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d825c410dbb842a8b24b91735dd11314~tplv-k3u1fbpfcp-zoom-1.image)  
该部分的 xposed 代码如下:

```
XposedHelpers.findAndHookMethod("org.chromium.CronetClient", lpparam.classLoader, "tryCreateCronetEngine", Context.class, boolean.class, boolean.class, boolean.class, boolean.class, String.class, Executor.class, boolean.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                Util.xposedLog("CronetClient disable tryCreateCronetEngine");
                param.setResult(null);
            }
        });

```

这样就可以继续抓包奔放了。

### 排名第二的短视频 app

该 app 使用 quic 协议比较早，之前由于不同版本混淆，导致 hook 代码要更新，后来通过看了一部分 Cronet 网络库的资料，找到该通用型的解决方案，该 app 的加载的 so 是`libconnection**.so`，记得没错的话。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/022c7c27d3064510a8e5e3af5df1ad65~tplv-k3u1fbpfcp-zoom-1.image)  
该部分的 xposed 代码如下:

```
    XposedHelpers.findAndHookMethod("com.**.aegon.Aegon", lpparam.classLoader, "nativeUpdateConfig", String.class, String.class, new XC_MethodHook() {
            @Override
            protected void
    beforeHookedMethod(MethodHookParam param) throws Throwable {
param.args[0] = "{\"enable_quic\": false, \"preconnect_num_streams\": 2, \"quic_idle_timeout_sec\": 180, \"quic_use_bbr\": true, \"altsvc_broken_time_max\": 600, \"altsvc_broken_time_base\": 60, \"proxy_host_blacklist\": []}";
            }
        });

```

最后 + 用 postern 转发到抓包工具。  
下载地址：postern 下载

### 看雪上的另外一种

使用`iptables` 禁止掉 udp 的 53 端口，因为 quic 使用的 udp 发包，53 端口又主要用于域名解析，所以禁止掉后，无法正常通讯，就会自动降级到 http。

小结
--

现在的 quic 协议抓包基本是通过 quic 无法正常使用，迫使 app 使用 http 进行抓包，希望能通过分析 cronet 相关源码进行突破，直接抓取 quic 的相关数据包。  
希望哥哥们关注小公众号，稍后建群可以讨论。  
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa299664b0e44831b1216fe1d35a979b~tplv-k3u1fbpfcp-watermark.image)

参考文章
----

https://segmentfault.com/a/1190000039827785

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm) [#其他](forum-161-1-129.htm)