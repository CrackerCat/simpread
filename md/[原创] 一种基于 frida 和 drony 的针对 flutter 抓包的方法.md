> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-261941.htm)

> [原创] 一种基于 frida 和 drony 的针对 flutter 抓包的方法

**1、使用 frida 解除 flutter 证书验证**
------------------------------

参考：https://www.jianshu.com/p/ada10d2976f2

Flutter 是 Google 使用 Dart 语言开发的移动应用开发框架，使用一套 Dart 代码就能快速构建高性能、高保真的 iOS 和 Android 应用程序。

由于 Dart 使用 Mozilla 的 NSS 库生成并编译自己的 Keystore，导致我们就不能通过将代理 CA 添加到系统 CA 存储来绕过 SSL 验证。为了解决这个问题，就必需要研究 libflutter.so。

当向 Burp 发送 HTTPS 流量时，Flutter 应用程序会抛出一个错误，可以将其作为起点进行溯源：

> E/flutter (10371): [ERROR:flutter/runtime/dart_isolate.cc(805)] Unhandled exception:  
> 
>  E/flutter (10371): HandshakeException: Handshake error in client (OS Error: 
> 
>  E/flutter (10371):  NO_START_LINE(pem_lib.c:631)
> 
>  E/flutter (10371):  PEM routines(by_file.c:146)
> 
>  E/flutter (10371):  NO_START_LINE(pem_lib.c:631)
> 
>  E/flutter (10371):  PEM routines(by_file.c:146)
> 
>  E/flutter (10371):  CERTIFICATE_VERIFY_FAILED: self signed certificate in certificate chain(handshake.cc:352))
> 
>  E/flutter (10371): #0      _rootHandleUncaughtError. (dart:async/zone.dart:1112:29)
> 
>  E/flutter (10371): #1      _microtaskLoop (dart:async/schedule_microtask.dart:41:21)
> 
>  E/flutter (10371): #2      _startMicrotaskLoop (dart:async/schedule_microtask.dart:50:5)
> 
>  E/flutter (10371): #3      _runPendingImmediateCallback (dart:isolate-patch/isolate_patch.dart:116:13)
> 
>  E/flutter (10371): #4      _RawReceivePortImpl._handleMessage (dart:isolate-patch/isolate_patch.dart:173:5)

该错误显示了触发错误的位置：handshake.cc:352，代码如下所示。

> if (ret == ssl_verify_invalid) {
> 
>     OPENSSL_PUT_ERROR(SSL, SSL_R_CERTIFICATE_VERIFY_FAILED);
> 
>     ssl_send_alert(ssl, SSL3_AL_FATAL, alert);
> 
>   }

这是 ssl_verify_peer_cert 函数的一部分，该函数返回 ssl_verify_result_t 的枚举，枚举定义在 ssl.h 的第 2290 行：

> enum ssl_verify_result_t BORINGSSL_ENUM_INT {
> 
>   ssl_verify_ok,
> 
>   ssl_verify_invalid,
> 
>   ssl_verify_retry,
> 
> };

经过试验，将 ssl_verify_peer_cert 的返回值更改为 ssl_verify_ok (=0) 的话仍会因为上面的 ssl_send_alert() 函数调用而失败，因此需要找另一个 hook 点。handshake.cc 的代码段上方有一段验证证书链的方法：

> ret = ssl->ctx->x509_method->session_verify_cert_chain(
> 
>               hs->new_session.get(), hs, &alert)
> 
>               ? ssl_verify_ok
> 
>               : ssl_verify_invalid;

session_verify_cert_chain 函数定义在 ssl_x509.cc，此函数返回布尔值类型，并且没有像 ssl_verify_peer_cert 函数那样的问题，可以作为 hook 的目标。在该方法里可以看到有两个字符串可以辅助定位方法，如图 1-1 所示。

![](https://bbs.pediy.com/upload/attach/202009/767367_ZX46UNVT7PED88Q.jpg)

图 1-1  所需 hook 函数中有辨识度较高字符串

之后在 ida 中的 strings 可以找到并定位函数为 sub_3A5564，过程如图 1-2 到 1-4 所示。

![](https://bbs.pediy.com/upload/attach/202009/767367_BENBA5PWYT6FMHF.jpg)

图 1-2  stirngs 搜索到目标字符串

![](https://bbs.pediy.com/upload/attach/202009/767367_77RE7X2JMS8AV5M.jpg)

图 1-3  查看调用函数

![](https://bbs.pediy.com/upload/attach/202009/767367_X78VGJ29W5JVAPP.jpg)

图 1-4  确定本函数为目标函数

![](https://bbs.pediy.com/upload/attach/202009/767367_P63Z42RECBY9XXB.jpg)

图 1-5  利用前 10 字节定位函数

后面可以在 frida 中编写脚本，使用函数前 10 字节定位，在运行时将返回函数改为 true 即可绕过证书链检查实现抓包，示例如下。

> function hook_ssl_verify_result(address)
> 
> {
> 
>   Interceptor.attach(address, {
> 
>     onEnter: function(args) {
> 
>       send("Disabling SSL validation")
> 
>     },
> 
>     onLeave: function(retval)
> 
>     {
> 
>       send("Retval:" + retval)
> 
>       retval.replace(0x1);
> 
>     }
> 
>   });
> 
> }
> 
> function disablePinning()
> 
> {
> 
>  var m = Process.findModuleByName("libflutter.so"); 
> 
>  var pattern = "2d e9 f0 4f a3 b0 81 46 50 20 10 70"
> 
>  var res = Memory.scan(m.base, m.size, pattern, {
> 
>   onMatch: function(address, size){
> 
>       send('[+] ssl_verify_result found at:' + address.toString());
> 
>       // Add 0x01 because it's a THUMB function
> 
>       hook_ssl_verify_result(address.add(0x01));
> 
>     }, 
> 
>   onError: function(reason){
> 
>       send('[!] There was an error scanning memory');
> 
>     },
> 
>     onComplete: function()
> 
>     {
> 
>       send("All done")
> 
>     }
> 
>   });
> 
> }

之所以 address.add(0x01) 是因为看到这么个说明：在 32 位 ARM 上, 对于 ARM 函数, 此地址的最低有效位必须设置为 0, 对于 Thumb 函数, 此地址必须设置为 1。

针对 64 位 flutter.so 同样可以搜索 ssl_client 来定位函数，并通过函数前面的一串字节进行定位，过程如图 1-6 到图 1-9。

![](https://bbs.pediy.com/upload/attach/202009/767367_FG6TCZGXZMQJ2NY.jpg)

图 1-6  strings 搜索 ssl_client

![](https://bbs.pediy.com/upload/attach/202009/767367_XV3P39PMZY3NUZ7.jpg)

图 1-7  查找 ssl_client 的交叉引用

![](https://bbs.pediy.com/upload/attach/202009/767367_5PTG3ZCSBE6H835.jpg)

图 1-8  找到 ssl_client 的引用位置

![](https://bbs.pediy.com/upload/attach/202009/767367_PZVSA5JSTBJW37M.jpg)

图 1-9  通过函数头部字节定位

针对 64 位 flutter.so 的 hook 代码示例如下，地址不再需要 + 1。

> function hook_ssl_verify_result(address) {
> 
>     Interceptor.attach(address, {
> 
>             onEnter: function(args) {
> 
>                 console.log("Disabling SSL validation")
> 
>             },
> 
>             onLeave: function(retval) {
> 
>                 console.log("Retval:" + retval);
> 
>                 retval.replace(0x1);
> 
>             }
> 
>         }); 
> 
> }
> 
> function hookFlutter() {
> 
>     var m = Process.findModuleByName("libflutter.so");
> 
>     var pattern = "FF 03 05 D1 FD 7B 0F A9 9A E3 05 94 08 0A 80 52 48 00 00 39 16 54 40 F9 56 07 00 B4 C8 02 40 F9 08 07 00 B4";
> 
>     var res = Memory.scan(m.base, m.size, pattern, {
> 
>             onMatch: function(address, size){
> 
>                 console.log('[+] ssl_verify_result found at:' + address.toString());
> 
>                 hook_ssl_verify_result(address); 
> 
>             },
> 
>             onError: function(reason){
> 
>                 console.log('[!] There was an error scanning memory');
> 
>             },
> 
>             onComplete: function() {
> 
>                 console.log("All done")
> 
>             }
> 
>         });
> 
> }

执行后即可使用 packet capture 进行抓包，如图 1-10 所示。但是使用代理还有问题，需要使用 drony 来配合代理进行抓包。

![](https://bbs.pediy.com/upload/attach/202009/767367_TZSQ4UQFMYYTJQR.jpg)

图 1-10  通过函数头部字节定位

**2、使用 drony 与 fiddler 联合抓包**
-----------------------------

参考：https://zhuanlan.zhihu.com/p/139645460

drony 是一个十分方便的 VPN 软件，drony 会在你的手机上创建一个 VPN，将手机上的所有流量都重定向到 drony 自身，这样 drony 就可以管理所有手机上的网络流量，甚至可以对手机上不同 APP 的流量进行单独配置。

进入首页后左滑进行设置，页面如图 1-11 到图 1-13 所示。

![](https://bbs.pediy.com/upload/attach/202009/767367_8TBQ3U7E4B9TGSM.jpg)

图 1-11  主页面

![](https://bbs.pediy.com/upload/attach/202009/767367_FHVCBZCN7NDSBJN.jpg)

图 1-12  设置页面

![](https://bbs.pediy.com/upload/attach/202009/767367_6ZFNTMK9F82DTW5.jpg)

图 1-13  设置页面

设置到代理后回到主页面点击 off 启动，即可在 fiddler 查看到抓取的内容，如图 1-14 所示。

![](https://bbs.pediy.com/upload/attach/202009/767367_6YDS3YK8PEKJJM5.jpg)

图 1-14  抓包页面

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 2020-9-9 16:55 被 beimingyouyu 编辑 ，原因：