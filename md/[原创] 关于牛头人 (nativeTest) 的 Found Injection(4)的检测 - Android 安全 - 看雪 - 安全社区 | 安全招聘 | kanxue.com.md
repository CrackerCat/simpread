> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286580.htm)

> [原创] 关于牛头人 (nativeTest) 的 Found Injection(4)的检测

原由
--

上一篇 [关于 NativeBridge 注入的检测和隐藏](https://bbs.kanxue.com/thread-286536.htm) NativeBridge 注入后， 牛头人还是会显示 "**Found Injection(4)**"  
这不得行， 先测试：

1.  禁用`riru`和`LSPosed`, 正常
2.  随意`resetprop ro.dalvik.vm.native.bridge xx`后 am restart, `Found Injection(4)`
3.  `resetprop`为 0 后 正常  
    得出，检测逻辑应该和上一篇文章一样，只是内存中还有一个特征驻留了。

解决
--

从头阅读源码，果然找到一个点。[AndroidRuntime::addOption](elink@c47K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0M7#2)9J5k6h3q4F1k6s2u0G2K9h3c8Q4x3X3g2U0L8$3#2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6K6N6i4m8W2M7Y4m8J5L8$3A6W2j5%4c8Q4x3V1k6E0j5h3W2F1i4K6u0r3i4K6u0n7i4K6u0r3L8h3q4A6L8W2)9K6b7h3k6J5j5h3#2W2N6$3!0J5K9%4y4Q4x3V1k6T1j5i4y4W2i4K6u0r3j5$3!0J5k6g2)9J5c8X3A6F1K9g2)9J5c8V1q4F1k6s2u0G2K9h3c8d9N6h3&6@1K9h3#2W2i4K6u0W2j5%4m8H3i4K6y4n7k6s2u0U0i4K6y4p5z5o6u0S2k6r3p5$3y4e0l9K6j5e0R3I4j5h3j5%4k6h3g2W2k6o6t1&6x3U0c8S2x3X3b7J5k6o6V1@1x3U0x3%4y4h3j5$3j5K6S2U0x3W2)9K6b7X3I4Q4x3@1b7#2x3o6q4Q4x3@1k6Z5L8q4)9K6c8s2A6Z5i4K6u0V1j5$3^5`.)  
![](https://bbs.kanxue.com/upload/attach/202504/98159_7JNCC3BEJVDYRHH.webp)  
[Vector<JavaVMOption> mOptions;](elink@21fK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0M7#2)9J5k6h3q4F1k6s2u0G2K9h3c8Q4x3X3g2U0L8$3#2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6K6N6i4m8W2M7Y4m8J5L8$3A6W2j5%4c8Q4x3V1k6E0j5h3W2F1i4K6u0r3i4K6u0n7i4K6u0r3L8h3q4A6L8W2)9K6b7h3k6J5j5h3#2W2N6$3!0J5K9%4y4Q4x3V1k6T1j5i4y4W2i4K6u0r3j5$3!0J5k6g2)9J5c8X3A6F1K9g2)9J5c8X3W2F1j5$3I4#2k6r3g2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6g2X3M7Y4g2F1N6r3W2E0k6g2)9J5c8V1q4F1k6s2u0G2K9h3c8d9N6h3&6@1K9h3#2W2i4K6u0W2K9q4)9K6b7X3c8J5j5#2)9K6c8r3y4V1j5e0l9@1x3e0t1K6y4e0j5^5x3r3u0W2x3h3j5H3z5h3f1I4x3K6f1%4x3r3t1&6x3U0S2V1y4X3x3#2y4e0u0S2k6U0t1@1x3r3c8Q4x3@1u0D9i4K6y4p5x3e0x3@1i4K6y4r3K9r3I4Q4x3@1c8*7K9q4)9J5k6r3y4F1)  
![](https://bbs.kanxue.com/upload/attach/202504/98159_EUE9PY2EKZB9N4P.webp)  
看一下属性的内存位置：  
![](https://bbs.kanxue.com/upload/attach/202504/98159_JHUYB2WTJS2VK5V.webp)  
在 AndroidRumtime 的指针 + 8(point length) 的位置  
AndroidRuntime 的指针可以通过`AndroidRuntime::getRuntime`获取  
写个小`jni`测试一下：  
![](https://bbs.kanxue.com/upload/attach/202504/98159_MV9HB9TBP8FYWCW.webp)  
logcat  
![](https://bbs.kanxue.com/upload/attach/202504/98159_TDUEYASHV5GAH8J.webp)  
可以看出， `Vecrot`的结构还在，但值的指针大部分都是野指针，牛头人应该是通过`size`判断，因为如果加了`NativeBridge`后会多一个参数。  
指针知道了，那这次直接上 CE， 上传好 ·ceserver· 后， 执行`su -c ceserver_arm64`, 电脑连接上  
![](https://bbs.kanxue.com/upload/attach/202504/98159_UKGC24UF7ABKDU9.webp)  
不管了，清零：  
![](https://bbs.kanxue.com/upload/attach/202504/98159_8489JSXUDKVZSS9.webp)  
启动后奔溃，应该是`mOptions->array()`不能为 nullprt, 指向一个空白区域，正好下面有一块：  
![](https://bbs.kanxue.com/upload/attach/202504/98159_XCKVBYBE5ZQQHB9.webp)  
再次测试  
![](https://bbs.kanxue.com/upload/attach/202504/98159_G9CFSQECAJVY8K4.webp)  
难道不是？？  
其实不然，牛头人应该是同时检测了`webview_zygote`  
![](https://bbs.kanxue.com/upload/attach/202504/98159_KFWPH9EFQ3Y7YEX.webp)  
用同样方式改一下：  
![](https://bbs.kanxue.com/upload/attach/202504/98159_HDE7BGD46CAM3P5.webp)  
好了再改一上篇说的`NativeBridgeError`， `found Injection(4)`没了

### 结语

牛头人不愧是 native 的测试，都是内存检测, `Futile Hide(8)`是我用的一个调试模块导致，正式应该环境不会出现， 但`Abnomal Environment`应该解决一下，而且安装`LSPlosed`还是会出现`Found Inject`，等有空搞。睡......

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 2025-4-23 01:26 被无味编辑 ，原因： 格式

[#逆向分析](forum-161-1-118.htm) [#其他](forum-161-1-129.htm)