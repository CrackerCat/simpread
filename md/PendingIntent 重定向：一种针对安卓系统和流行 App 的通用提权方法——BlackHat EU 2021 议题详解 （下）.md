> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/ifrErL88_8wN36WT8QbLvg)

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDhP6iadDnww2c7bicXCnskmlOB7JpRy5h8RBTlXMhdl6AwqZms1W9KotA/640?wx_fmt=png)

以用户隐私安全为中心，用责任兑付信任，OPPO 成立子午互联网安全实验室（ZIWU Cyber Security Lab）。实验室以 “保护用户的安全与隐私，为品牌注入安全基因” 为使命，持续关注并发力于业务安全、红蓝对抗、IoT 安全、Android 安全、数据和隐私保护等领域。

本篇文章源自 OPPO 子午互联网安全实验室。

**1 不安全 PendingIntent 的通用利用方法**

**1.1 不安全 PendingIntent 的特征**

至此，我们已经解决了本议题的第一个问题，经过研究表明，Android 系统中使用的 PendingIntent 大都 可以被三方 App 获取。

获取方式包括 bind SliceProvider、监听通知、连接媒体浏览器服务或者 bind 容纳 窗口小部件的 AppWidgetsProvider。

于是，引入议题研究的第二个关键问题: 如果这些 PendigIntent 不安全，如何利用才能造成安全危害?

首先，我们需要辨别什么样的 PendingIntent 是不安全的。前面描述的公开漏洞案例，均为劫持 base Intent 为空 Intent 的广播 PendingIntent，说明如下 empty base Intent 构建的 PendingIntent 确定存在安全问题。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDXF4sfpe4zfrjNo35Sicl92QNme0p5QnrYibaS7XSVRH2eQ8fA95icczRw/640?wx_fmt=png)

Android 12 之前的开发者文档也对 base Intent 为隐式 Intent 的 PendingIntent 提出了安全警告，但却没 有明确告知到底存在何种危害。而且在 AOSP 代码和流行 App 当中，如下的代码模式广泛存在。这不禁让 我们思索， Implicit base Intent 构建的 PendingIntent 是否真正存在问题? 唯有找到一种确定的针对这 种 PendingIntent 的漏洞利用方法，才能真正证明安全问题的存在。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDqXBuJIo5wIjibYjbfGJ7hBRORsibdPJNFjl4sUIVRoc8KsNyRSFiax4UA/640?wx_fmt=png)

**1.2 深入 Intent fillIn 改写机制**

寻找利用方法之前，需要深入探索 PendingIntent 的改写机制，这决定了其他 App 获得 PendingIntent 以 后，如何对 base Intent 进行改写。这个机制由 Intent.fillIn 函数提供：

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDeefQSNoBZl3u7NLjUJHrQNzBiacRZjia7kUibZZ84L1dTib8KQtKf0miazg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDJYAPXUOZFZ6D4H9GZanq0Mvxq0gO83uCHPTxZdbtcr7VotxFY8lOfg/640?wx_fmt=png)

在上述代码中，this 对象指向当前 Intent，other 为其他 Intent。如果当前 Intent 中的成员变量为空，则可 以被 other 中相应的成员变量覆盖。比较特殊的是 Intent 中的 component 和 selector 成员，即使当前 Intent 中的 component 和 selector 为空，也不能被 other 所改写，除非 PendingIntent

设置了 FILL_IN_COMPONENT

或者 FILL_IN_SELECTOR 标志。

**1.3 PendingIntent 重定向攻击**

因此在获取 PendingIntent 之后，其 base Intent 的 action、category、data、clidpdata、package、flag、extra 等成员都是有可能改写的，而 component 和 selector 无法改写，如图所示。特别地，对于 base Intent 为隐式 Intent 的这种情况，action 已经被设置了，因此也无法被改写，攻击者无法如前面安 卓系统 broadcastAnyWhere 漏洞那样，通过劫持 PendingIntent、在 base Intent 中重新添加 action，隐式打开一个受保护的组件。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDTpX91dia7ZMlGZcsRmMB2BzEZyjhyb6fryReXCkygawFn535ibdCoybg/640?wx_fmt=png)

图 Intent 成员

这里就来到了问题解决的关键点，由于 package 可以指定，回想到以前在 Intent Bridge 漏洞中的利用方 法，我们可以通过设置 intent 中的 flag 来巧妙地解决这个问题。Intent 提供了有关临时授权的标志:

**·**FLAG_GRANT_READ_URI_PERMISSION:Intent 携带此标志时，Intent 的接收者将获得 Intent 所携 带 data URI 以及 clipdata URI 中的读权限

**·**FLAG_GRANT_WRITE_URI_PERMISSION:Intent 携带此标志时，Intent 的接收者将获得 Intent 所携 带 data URI 以及 clipdata URI 中的写权限

简言之，恶意 App 对 PendingIntent 进行了指向恶意 App 自己的重定向，通过对 PendingIntent base Intent 的部分修改 (修改包名、授权标志和 data/clipdata)，使其以受害 App 的权限打开恶意 App 自身， 这样恶意 App 在被打开的瞬间即获得对受害 App 私有数据的读写权限。具体的利用方法如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVD3UjMs6ILs4plwMUAIe28UiacWoIUFH8oONL49YfVnACHG1uEbwyFDcg/640?wx_fmt=png)

图 PendingIntent 重定向攻击

步骤如下:

1、受害 App 通过 getActivity 构建 PendingIntent，在通知、SliceProvider、窗口小部件中使用，假定其 base Intent 为隐式 Intent;

2、攻击 App 通过前面探讨的各种渠道获取受害 App 的 PendingIntent;

3、攻击 App 修改 PendingIntent 中的 base Intent，由于是隐式 Intent，因此 action、component 和 selector 都不能修改。但可以做如下修改:

  · 修改 data 或者 clidpdata，使其 URI 指向受害 App 的私有 ContentProvider; 

  · 修改 package，指向攻击 App;

 · 添加 FLAG_GRANT_READ_URI_PERMISSION 和 FLAG_GRANT_WRITE_URI_PERMISSION 标志。

同时攻击 App 声明一个 Activity 支持隐式启动，其 Intent-filter 与 base Intent 中的 action 一致。

4、攻击 App 调用 PendingIntent.send;

5、由于这个 PendingIntent 代表了受害 App 的身份和权限，因此将以受害 App 的名义发送修改后的 base Intent，打开攻击 App 的 Activity;

6、在攻击 App Activity 被打开的瞬间，即被授权访问 base Intent 中携带的 URI，也就获得了对受害 App 私有 ContentProvider 的读写权限。

上面受害 App 的私有 ContentProvider，需要携带属性 grantUriPermission=true ，不限于受害 App 自 己的 ContentProvider，也包括受害 App 有权限访问的 ContentProvider。手机上一个常⻅的具有 grantUriPermission=true 属性的 ContentProvider 就是代表通讯录的 Contacts Provider，只要受 害 App 具有 READ_CONTACTS 权限，出现这样一个 PendingIntent 漏洞后将导致通讯录泄露。

这样，我们通过上述 6 个步骤，就可以成功实现对隐式 Intent 构建 PendingIntent 的漏洞利用，读写受害 App 的私有数据，这也就解决了本研究提出的第二个关键问题: 通过隐式 Intent 构建的 PendingIntent 可遭受通用的重定向提权攻击，也是不安全的。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDK88yJ7eD8tB4Q5nFcGAQru7uKFaGzIhHibdiaxXJCDcdWDeKusXO9t1w/640?wx_fmt=png)

由于这里使用了 grantUri 的技巧，因此并不适用于 broadcast PendingIntent，因为广播接收器是不可以 被 grantUri 的。另外，从 Android 5.0 以后，Service 不能隐式启动，因此也很难看到 base Intent 为隐式 Intent 的 service PendingIntent。所以，这里的 PendingIntent 重定向攻击主要适用于 Activity PendingIntent。

**2 安卓系统中的真实案例**

令人惊讶的是，在 Android 12 之前的 AOSP 代码以及流行 App 中，隐式 Intent 构建的 ActivityPendingIntent 广泛存在，以下是我们发现的典型案例，可能导致手机的敏感信息泄露，甚至以受害 app 的权限执行任意代码。这些漏洞案例均已被厂商所修复。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDcIUAkyYMibdDgEJiay2N9pf88zwJIibsIq74HRfRHKjk3Wicu2HoDax3qw/640?wx_fmt=png)

图 不安全 PendingIntent 典型案例

**2.1 CVE-2020-0188**

不安全的 PendingIntent 存在于 AOSP SettingsSliceProvider 中，

一旦 SettingsSliceProvider 被 blind,

在返回的 Slice 中将携带一个不做任何操作的 noOpIntentPendingIntent:

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDzz7kbW6xQ1mqwXZMPPeib5bJhaF6icY5Q8f1cHGKE2ktl65lkxlc5EXQ/640?wx_fmt=png)

攻击 App 通过 bind SettingsSliceProvider 获取 PendingIntent，修改 base Intent 并以 Settings 的权限发送，等待自己的 Activity 被打开，就可以实现 Settings 某些私有 Content Provider 的读写。如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVD9hfhUZIKSqw9Sicgvsoogicr5xAVK5pCCSDgeGV0p54THicKWb7PsJp7w/640?wx_fmt=png)

图 CVE-202-0188 POC

**2.2 CVE-2020-0389**

不安全的 PendingIntent 存在于

AOSP SystemUI RecordingService 当中，为用户录屏保存成功后发送的通知所使用。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDuRr2DeTzBLm5NCCN3xDiaDGQicHaPWYy9zkFAl5icepbAQRYQZFjibJycQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDfD9KB82NHcS2Y3jf6nia8CAZcwI0J8U4zIbGhia4Mklibh85vo6LWIrcQ/640?wx_fmt=png)

恶意 App 可以实现一个 NotificiationListener

Service，修改 base Intent，将其 clipdata 指向 ContactsProvider :

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDcrBb8iaaDHRsZvYrOQZLdDKbE5GYFSZKAKs77LGsSNiaia937GbZQQJpA/640?wx_fmt=png)

由于 SystemUI 具有 READ_CONTACTS 权限，因此恶意 App 被打开时，即可成功读取通讯录。

**2.3 A-166126300**

不安全的 PendingIntent 存在 AOSP BluetoothMediaBrowserService 中

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDdOwWTibRYH9K52fCQLiciaibxkogAobJd9biclFrH9icEuwbsdYHI4PYxicibA/640?wx_fmt=png)

恶意 App 可以连接 BluetoothMediaBrowserService ，通过 MediaBrowserCompat.ConnectionCallback 获取 PendingIntent。  

由于 BluetoothMediaBrowserService 存在于具有通讯录权限 Bluetooth 应用中，因此通过 PendingIntent 重定向攻击可读取通讯录

**2.4 某流行 App**

某具有通讯录权限的流行 App 实现了窗口小部件，用户点击窗口小部件的按钮实现跳转，但这个跳转是通过隐式 Intent 构建 PendingIntent 实现的。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDU8Hlu3NrZoxEaoWv876NIQQoQVTZZL1JiaZ9n4MyzEEyG1NOEcpnqeQ/640?wx_fmt=png)

对窗口小部件所属的 AppWidgetProvider 进行 bind，通过反射逐次获取 RemoteViews->mActions->mResponse->mPendingIntent ，最终可以拿到上述不安全的 PendingIntent，进而如法炮制，读取通讯录。

**2.5 CVE-2020-0294**

这些不安全的 PendingIntent 存在安卓系统服务中，对于某些 bind 服务，系统提供了 PendingIntent，跳转到服务的管理界面。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVD2UP8pzA3NSmwnkE1VMWYg6ibGN7aKF5xUicGBAuzr3driaXJ3OkVc4qfg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDCZiaFLhP2sQJ3tFRGTCtuyvWuoa11SUIHSyEIlYkkjAq6XQ8SQibPyFg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDzAh2qIQss4tl8TbT6QhvAZxdicGKBcu1Bic7VbXBcTbUgJq0Zx2XgicRQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDK7kJChNWtficEZexKAdGnToGe1o9yl97GJKTlqu1aib9fvRAbMQSrbKw/640?wx_fmt=png)

这些 PendingIntent 可以直接通过系统 API ActivityManager.getRunningServiceControlPanel 获得，后面进行 PendingIntent 重定向，读取 Settings 中的保护 ContentProvider。

**2.6 危害**

上述多个案例均可造成通讯录这类个人敏感信息泄露，但实际上，由于 PendingIntent 重定向攻击还具有写数据的能力，因此可能造成更大的危害。

例如，很多 App 都具有热更新功能，一般将 dex/jar/apk/so 等文件放在自己的私有目录中，如果这些私有目录可以被 grantUriPermission=true 的 ContentProvider 所引用，就可以利用 PendingIntent 重定向攻击去改写热更新文件，将攻击者自己的代码注入到其中，实现以受害 App 的权限执行任意代码。

对于 CVE-2020-0188 和 CVE-2020-0294 这类源于系统 uid 的 PendingIntent，由于在 UriGrantsManagerService 当中进行了限制，因此在原生系统中的危害很有限，只能读取特定的几个 Content Provider。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDZ4G6ZGGy8ZMBBLVrHibiagOHhPVwNump2WHgD6HNwWhN9wyXOXvH4icCQ/640?wx_fmt=png)

但是由于 Android 系统的定制化，上述限制可能在 OEM 厂商中被打破，造成更大的危害。

Google 对安卓系统中这类漏洞的修复，起初是将 base Intent 设置为显式 Intent，指定明确的组件。后来均使用 FLAG_IMMUTABLE 修复，当使用这个 flag 时，PendingIntent 的 base Intent 将无法通过 Intent.fillIn 函数改写，例如

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDVFVsZa9dMF1WHzOkRKxyibGQqQORWog86RN2oqbLE6ibIhmxNWeeH4CQ/640?wx_fmt=png)

**3 自动化分析**

基于对不安全 PendingIntent 特征的掌握，我们编写了一个自动化扫描工具 PendingIntentScan，该工具基于 Soot[4] 这一 Java 静态分析框架对 apk 进行数据流静态分析，其体系结构如图所示。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDMbzaDKfCb9Ae5QIwopO2ibMtRsMm65tTDDv5nvgy6Q6mQPutiaCC4ibBQ/640?wx_fmt=png)

图 PendingIntentScan 原理

首先，使用 Soot 将 apk 的字节码转换为 Jimple 形式的 IR，然后搜寻一系列生成 PendingIntent 的 API，并挑选出没有使用 FLAG_IMMUTABLE 的：

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDvXMU8icS54oe7nAGzicUhe8LIAHVl9E8bbayD1Wvq9shgqMiakHAKNiaQQ/640?wx_fmt=png)

然后，通过 Soot 提供的 ForwardFlowAnalysis 对 PendingIntent 的 Intent 参数进行检查，查看是否调用下列函数。如果都没有使用，则认为 PendingIntent 是不安全的：

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDKmenoNan8w8DxHDVRakCv95aJL2ezZcjkFEjGIBkrvjqcicpL8nhtOQ/640?wx_fmt=png)

这个工具目前开源在

https://github.com/h0rd7/PendingIntentScan，可以迅速发现 apk 中存在的不安全 PendingIntent，效果如下。

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDbwb22wxp3Vv1icMQqI52XUGe15XBziayduOVHiaTLScoapdgL5qbcINZg/640?wx_fmt=png)

**4 安卓 12 安全变更**

针对我们的研究成果，Google 安卓安全团队对 AOSP 代码进行了全面排查，几乎修复了所有的不安全 PendingIntent。大部分的修复使用了 PendingIntent.FLAG_IMMUTABLE，小部分的修复将 base Intent 设置为显式 Intent。

而在 Android 12 大版本中，安卓系统对 PendingIntent 的行为进行了重大安全变更，引入了一个新的

flag:PendingIntent.FLAG_MUTABLE，表示 base Intent 可以改写。这与原有的 FLAG_IMMUTABLE 共同描述 PendingIntent 的可变性。

对于 Target S + 的 App，Android 系统要求开发者必须明确指定 PendingIntent 的可变性，FLAG_IMMUTABLE 和 FLAG_MUTABLE 必须使用其一，否则系统会抛出异常。这就要求开发者对自己 PendingIntent 的使用有清晰的理解，知道 PendingIntent 是否会在将来被改写。

Google 也对开发者提出了详细的安全编码建议:

**·** 尽可能使用 FLAG_IMMUTABLE 来生成不可改写的 PendingIntent; 

**·** 如果使用 FLAG_MUTABLE 来生成可改写的 PendingIntent，base Intent 一定要使用显式 Intent，明确指定 Intent 的组件。

同时 AndroidStuido IDE 中也引入了一个新的 lint 检查插件 PendingIntentMutableFlagDetector，用于检查 PendingIntent 是否使用了 FLAG_IMMUTABLE。

**5 结论**

本议题解决了 PendingIntent 的获取问题，明确了不安全 PendingIntent 的特征，提出了有关不安全 PendingIntent 的重定向攻击利用方法，从而揭示了安卓系统和流行 app 有关 PendingIntent 使用的一种通用安全⻛险。Google 针对议题描述的漏洞均已进行了修复，并在 Android 12 中引入了缓解此问题的重大安全变更，对开发者提出了详细的安全编码建议。

开发者在使用 FLAG_IMMUTABLE 构建 PendingIntent 时应格外小心，除了要使用显式 Intent 以外，还要保证 base Intent 其他没有填充的字段不会造成安全影响。例如下面存在问题的代码源于一个真实 app 的案例。这个 PendingIntent 已经设置了显式 Intent，在通知中使用，用于启动内部不导出的 MainActivity

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDoFr3SibAlngdjR32cwogV2NGibZsvN4pLkMeyYW3NrVokqibCRdpibad1w/640?wx_fmt=png)

在 MainActivity 中，

可以对 EXTRA_REDIRECT_INTENT 进行处理，最后调用 startActivity：

![](https://mmbiz.qpic.cn/mmbiz_png/ECThdibSVDxgYa1VhSlMgVVJzgelLDGVDD7YR1cmAauY1YmND0EkA2PaXicticcsERXMY4tOgTbGOhm6NlgD4Ludw/640?wx_fmt=png)

这样，劫持 PendingIntent 仍然可以设置 EXTRA_REDIRECT_INTENT，通过 startActivity 去打开应用的任意保护组件。

因此，每一个没有使用 FLAG_IMMUTABLE 的 PendingIntent 均应该仔细审查，这是我们对开发者的最后安全忠告。

**参考**

[1]http://retme.net/index.php/2014/11/14/broadAnywhere-bug-17356824.html  

[2]https://www.slideshare.net/CanSecWest/csw2016-chaykin-havingfunwithsecuremessengersandandroidwear

[3]https://mp.weixin.qq.com/s/SAhXsCHvAct_2SxCXd2w0Q

[4] http://soot-oss.github.io/soot/

[5]https://developer.android.com/about/versions/12/behavior-changes-12#pending-intent-mutability  

  

**作者简介**

**heeeeen  安全架构师**

毕业于北京航空航天大学，现工作于 OPPO 子午互联网安全实验室，擅长 Android 框架与 APP 漏洞挖掘，多次获得 Google 安全致谢

*** 本文版权归 OPPO 公司所有，如需转载请在后台留言联系**

**最新动态**  

[PendingIntent 重定向：一种针对安卓系统和流行 App 的通用提权方法（上）](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247489227&idx=1&sn=187344e29f10c27b12770d287b958f27&chksm=fa7b1787cd0c9e91306b9d7459c09e7268015a02c37d576af2b904956891ffdfa6029293d366&scene=21#wechat_redirect)
=====================================================================================================================================================================================================================================================================

[技术分享 | OPPO 在深度伪造和检测的探索与实践‍](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247489162&idx=1&sn=ee2ab2ad60ce44802da07f5f236ca976&chksm=fa7b17c6cd0c9ed0696cb863d71e607493b0fb49b7439c72587069847f8e0e2c1774e6588177&scene=21#wechat_redirect)
====================================================================================================================================================================================================================================================

[点击查看 | 2021，你与 OSRC 的亲密度如何？](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247489134&idx=1&sn=34c0a9fb1fd57cb4d6770cfcf2aa30a0&chksm=fa7b1722cd0c9e348882e401e2d8d1fb913d178c27ad125fb53a62991fd5fcf5463d049f8fc9&scene=21#wechat_redirect)  

=======================================================================================================================================================================================================================================================

[【年终奖励】年度排行终揭晓，师傅有话对你说~](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247488991&idx=1&sn=c671871f9b96c6fd2c8654a5eca72c92&chksm=fa7b1493cd0c9d853195cfacd8312f00942d1f8057bc775e0b44aff4b2a17e4048622bf4eb18&scene=21#wechat_redirect)

[技术分享 | 智能护盾安全大脑：移动应用智慧检测能力构建](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247488978&idx=1&sn=34f8169b952be0b0d55af9aa7e4f91fc&chksm=fa7b149ecd0c9d88799e1c992d47b6eb815f1bd6793a5b76dabbe9f5f81cd1d49f3eacad4059&scene=21#wechat_redirect)

[汇聚极客顶尖高手和行业大咖，OPPO 安全 AI 挑战赛暨高峰论坛圆满落幕](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247488834&idx=1&sn=a91cc381a62dc1edd18affcec99cd1fe&chksm=fa7b140ecd0c9d18ca541c55639d985ff6a72a9b66eb0785dc1ae3b8a98046f2f6f99ccaa6d5&scene=21#wechat_redirect)
==============================================================================================================================================================================================================================================================

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)