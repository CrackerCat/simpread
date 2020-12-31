> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484195&idx=1&sn=a6b728fbab31ee74087c2eaca5d58b18&chksm=cebb466df9cccf7ba4da3aadfc915b876344852aa4920feed8f246c751525572a9e869ad7786&scene=21#wechat_redirect)

一、Objection 针对所有支持的平台，提供了下列核心功能 2

（一）Android 特殊功能 2

（二）iOS 特殊功能 2

二、搭建 Objection 测试环境 2

三、使用 Objection 测试分析移动 App3

**一、****Objection 针对所有支持的平台，提供了下列核心功能**

  (1) 修复 iOS 和 Android 应用程序，嵌入了 Frida 实用工具。

  (2) 与文件系统交互，枚举条目以及上传 / 下载的文件。

  (3) 执行各种内存相关任务，例如列举加载的模块以及相关的输出。

  (4) 尝试绕过或模拟越狱 / root 环境。

  (5) 发现加载的类，并列举对应的方法。

  (6) 执行常见 SSL 绑定绕过。

  (7) 针对目标应用程序，从方法调用中动态导出参数。

  (8) 与内联 SQLite 数据库交互，无需下载其他数据库或使用外部工具。

  (9) 执行自定义 Frida 脚本。

  (10) 动态分析在运行代码的情况下，通过跟踪分析相关的内存，如寄存器内容，函数执行结果，内存使用情况等等，分析函数功能，明确代码逻辑。

（一）**Android 特殊功能**

   (1) 枚举应用程序的活动、服务和广播接收器。

   (2) 开启目标应用程序中的任意活动。

   (3) 监控类方法、报告执行活动。

（二）**iOS 特殊功能**

   (1) 导出 iOS 钥匙串，并存储至文件中。 

   (2) 从常见存储中导出数据，例如 NSUserDefaults 以及共享 NSHTTPCookieStorage。

   (3) 将信息以可读形式导出。

   (4) 绕过 TouchID 限制。

   (5) 监控类中的所有方法执行。

   (6) 监控 iOS 剪贴板。

   (7) 在无需外部解析工具的情况下，将已编码的. plist 文件导出为可读形式。

**二、搭建 O****bjection** **测试环境**

python --version

python3 --version（**建议安装 python3 最新版本**）

frida --version

pip install objection

objection version

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDn9LmUGYLiaScaYw11NtoNLPSiaXnqcycGMRzmUtU6KLKruV3mlqB49ERA/640?wx_fmt=png)

**三、使用 O****bjection** **测试分析移动 App**

objection -g com.***.***.cn explore

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnebtFLrKTOwTu9OXfFRY05DVd0EppPVYJnibyGJ2TaVsOic0RJIUABpfg/640?wx_fmt=png)

objection -g com.***.cn explore

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnBZzkqjEqBUfiaOsWvwhtjEUzWrAPbbYpqqg98m5ZRKUD3goIDBzZhGA/640?wx_fmt=png)

**监视方法的参数、调用栈及返回值**

android hooking watch class_method java.lang.StringBuilder.toString --dump-args --dump-backtrace --dump-return 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnwvEv5CicrUN3GGddOclOOYVqPOHSH7hPmiaf6sxQZQekybS0wFTIpjog/640?wx_fmt=png)

**监视目标类下的所有方法的调用情况，以防 so 中有未察觉到的反射调用**

android hooking watch class com.***.mb.common.system.ConnectReceiver

android hooking watch class com.***.cn.splash.DexOptService

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnQEGBClJeKmJprjW91XtHhV2jfibrHuMWVFCyzWj2R10wXY3EcmEWLgg/640?wx_fmt=png)

mv frida-server-12.11.12-android-arm fs121112

chmod 755 fs121112

**通过 objection 动态分析 App，因要分析的 App 会自动断掉 USB 连接，所以在测试机上用运行 frida 服务，并监听 8888 端口，用电脑去连接监听的端口**

./fs121112 -l 0.0.0.0:8888

frida-ps -H 10.22.6.209:8888

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnEXtRflicmAGFq5cqicsXmvTpmpIAjCqOvEOIetRvRlNLGkr7NWibaORQA/640?wx_fmt=png)

**配合加载 Wallbreaker 插件，更方便的搜索查看 Android 内存中的类结构、实例、内部数据等**

objection -N -h 10.22.6.209 -p 8888 -g com.***.cn explore -P /home/gyp/gyp/SecurityAnalysis/objection/plugins

objection -N -h 10.22.6.209 -p 8888 -g com.***.cn explore -P

~/gyp/SecurityAnalysis/objection/plugins/

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnIsaDbpR5E8EszbOVHA1FDWYHKcL5LWwic2NOVOc4CuXOx4EYF3VjXtw/640?wx_fmt=png)

android hooking list services 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnqib2xOfofxXl9jYpQXL3IF6wMEn53zqhJ3N4TwFrQHTdl3ImlazPCYg/640?wx_fmt=png)

android hooking list receivers 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDn96nYKibHmatZVSka2ojJcrGb7TzRqdjSGTQxZqfAgph0rWv3S9z62xQ/640?wx_fmt=png)

**配合加载 Wallbreaker 插件，更方便的搜索查看 Android 内存中的类结构、实例、内部数据等**

plugin wallbreaker classdump com.***.cn.cache.ArtistCacheManager --fullname

plugin wallbreaker classdump com.***.cn.splash.DexOptService --fullname

plugin wallbreaker classdump com.***.mrs.util.MatrixReportBroadcast --fullname

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDne8O3ZbgBjfVs1BE58cCFVZQic5kd6TOV41BP0SBMmnib4qTQJ1xDyDHA/640?wx_fmt=png)

**监视目标类下的所有方法的调用情况，以防 so 中有未察觉到的反射调用****，****通过动静态结合分析了解大致逻辑对应的类，然后 hook 该类**

android hooking watch class com.***.cb.common.system.ConnectReceiver

android hooking watch class com.***.cn.splash.DexOptService

android hooking watch class com.***.mgrs.util.MatrixReportBroadcast

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDn75y7hMOJ6icuxSyqW32IoCMnIegtalccDnBPesWDGclUS49jWyj6V0w/640?wx_fmt=png)

**监视方法的参数、调用栈及返回值**

android hooking watch class_method com.***.mrgs.util.MatrixReportBroadcast.onReceive --dump-args --dump-backtrace --dump-return

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnsgeLmsYhXmAMOL1icA3vDga3VC143eoF3pMGVw5XibqDtSHlUURZdbNQ/640?wx_fmt=png)

android hooking watch class_method com.***.cn.sdk.g.b.d.a --dump-args --dump-backtrace --dump-return

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnwdUVkEHYhgk4Npiad90cAj3TwXtFIVYART1tbYZluTokIKMpibye1jicg/640?wx_fmt=png)

**Android App 弹窗堆栈**

android hooking watch class_method android.app.Dialog.show --dump-args --dump-backtrace --dump-return

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF3BXibU5mTiacSojib2foqArDnYudjRRucwKsI0uWK0IHYFIibE0pf6D3j4e2dUdicJquCs4hicV100zcyw/640?wx_fmt=jpeg)