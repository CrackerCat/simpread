> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2106217-1-1.html)

> [md]iv8 是基于 V8 引擎的高性能 Python 原生扩展，在 C++ 层实现浏览器 API，提供高可控、高保真的 BOM/DOM/CSSOM 模拟，内置 API 调用链监控与 Chrome D......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)521105 _ 本帖最后由 521105 于 2026-5-6 14:17 编辑_  

iv8 是基于 V8 引擎的高性能 Python 原生扩展，在 C++ 层实现浏览器 API，提供高可控、高保真的 BOM/DOM/CSSOM 模拟，内置 API 调用链监控与 Chrome DevTools 远程调试，可在 Python 中直接运行依赖 Web 环境的 JavaScript，无需启动浏览器。

项目地址：[github.com/HanZzzzz000/iv8](https://github.com/HanZzzzz000/iv8)  
PyPI：[pypi.org/project/iv8](https://pypi.org/project/iv8/)  
安装：`pip install iv8`

abogus、h5st、`__zp_stoken__`、瑞数 6，仓库 `examples/` 里都给了使用示例。觉得有用，欢迎到 GitHub 点个 Star。

> 本文仅供学习研究、安全测试与合法自动化场景使用，不构成攻击或未授权访问指引。完整免责声明见文末。

* * *

### 一、先说为什么要造这个轮子

补环境这事，

最常见的是 **node 补环境**：大多数 jsdom 打底，自己再用 Proxy / `Object.defineProperty` 一通修补。这条路理论上什么检测点都能补，难点在成本 —— 像 `document.all` 的 `[[IsHTMLDDA]]` 内部 slot、跨 Context 对象身份、`Object.keys(window)` 的枚举顺序，每一处都能写出来，但要写到对方任何角度都识别不出来，每个站还得单独调一套补丁，零零散散既不通用、维护起来也累。况且 Python 项目里再背一个 Node 进程做异语言桥接，本身就不优雅。

另一条路是 **自动化**，Puppeteer / Playwright / DrissionPage 直接开真浏览器。问题也明显：1000 并发就要 1000 个 Chrome 实例，内存先爆；CDP 协议又粗又慢，连接动不动还断；想批量跑就得堆机器、堆 worker，整体成本高。

**iv8 走的是第三条路：把 V8 拉出来，在 V8 之上用 C++ 做一套浏览器运行时。** 它不是 JS Proxy 模拟，而是在 C++ 层实现 `HTMLDivElement`、`document.cookie`、`navigator.webdriver`、`crypto.subtle.digest` 这些 JS 能观察到的浏览器对象、访问器、原型链与关键边界语义。底层尽量沿着 Chromium / Blink 的绑定模型走，内部参考 `ScriptWrappable` / `WrapperTypeInfo` 的对象组织方式。

直白讲：以前补环境是在 JS 层 "打补丁"，iv8 是在 C++ 层 "建宿主"。

* * *

### 二、iv8 长什么样

![](https://attach.52pojie.cn/forum/202605/05/165359r2qmt668c2g6hcht.png)

**system_architecture.png** _(206.8 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg0OTM2MXwyOGVjMjE2MXwxNzc4MzIxMDAwfDIxMzQzMXwyMTA2MjE3&nothumb=yes)

system_architecture

2026-5-5 16:53 上传

> 图注：本文几张架构 / 时序 / 网络流程图来自 iv8 最新版内部文档，用来解释整体设计方向。社区版保留核心运行模型，但真实网络栈、布局 / 动画等能力仍以本文 “社区版能干什么” 章节为准。

一句话：

```
import iv8

with iv8.JSContext() as ctx:
    print(ctx.eval("navigator.userAgent"))
    print(ctx.eval("navigator.webdriver"))
    print(ctx.eval("typeof document.all"))

```

输出是这样的：

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...Chrome/124.0.0.0 Safari/537.36
False
undefined

```

`document.all` 那一行不是字符串拼接出来的，是 V8 层做了 `[[IsHTMLDDA]]` 的语义 —— `typeof document.all === 'undefined'` 但 `document.all !== undefined`。纯 JS 层模拟不出这种 "语言层 + 引擎层联动" 的语义，只能上 native addon；而 iv8 直接在 V8 层做掉，JS 看到的就是真实的 `[[IsHTMLDDA]]` 行为。

每个 `JSContext` 内部独占一个 V8 Isolate，挂载完整的 window 域，覆盖 70+ HTML 元素接口、25+ CSS 规则类型、80+ 事件类型、30+ WebGL 扩展，以及完整的 SubtleCrypto（AES-GCM / RSA-OAEP / ECDH / ECDSA / HMAC / HKDF / PBKDF2）。`iv8.JSContext.get_defaults()` 一打印出来 200 多行，全是可以通过 `environment` 参数覆盖的指纹字段。

* * *

### 三、30 秒上手

跑一个最简单的页面加载：

```
import iv8

with iv8.JSContext() as ctx:
    ctx.eval("""
        window.__iv8__.page.load({
            baseURL: 'https://example.com/',
            html: '<html><head><script>window.x = navigator.userAgent.length;</script></head><body><div>Hello</div></body></html>'
        });
    """)
    print(ctx.eval("window.x"))                                       # UA 长度
    print(ctx.eval("document.getElementById('app').textContent"))     # "Hello"
    print(ctx.eval("document.URL"))                                   # 'https://example.com/'

```

`page.load` 是流式解析 + 脚本执行 + 事件派发，不是 `innerHTML` 那种把字符串塞进去就完事。HTML 解析到 `<script>` 会暂停，把脚本拉出来执行（外联脚本从离线 bundle 匹配），然后继续解析、派发 `DOMContentLoaded` / `load`、同步 `document.URL` 和 `location.href`。

时序对齐细到 Chrome 的内部 feature。比如 Chrome 111+ 默认启用的 `kTimedHTMLParserBudget`：HTML parser 不再只按 token 数让出执行权，而是引入默认 10ms 的 elapsed-time budget；解析过程中如果被 parser-blocking 脚本拖过预算，parser 会 yield，让 `setTimeout(fn, 0)` 这种到期宏任务有机会先跑一轮再续解析。iv8 会根据 UA 解析 Chrome 主版本号，按这个版本边界决定是否启用类似行为 —— UA 写 Chrome 110 和 Chrome 124，同一段页面的脚本执行顺序可能就不一样。

![](https://attach.52pojie.cn/forum/202605/05/170021g00sg9zsbfd0km2t.png)

**Key Event Timing.png** _(146.35 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg0OTM2NHwwZTBmMzY0Y3wxNzc4MzIxMDAwfDIxMzQzMXwyMTA2MjE3&nothumb=yes)

Key Event Timing

2026-5-5 17:00 上传

> 图注：该图来自最新版事件时序图，用来说明 iv8 对页面加载阶段的设计目标；社区版已实现核心 `page.load` / 事件循环 / 逻辑时间能力。

如果只需要把 HTML 解析成 DOM 树（比如做数据提取），可以走另一条路 —— `document.documentElement.innerHTML = "..."`，开销低很多，但脚本不会执行、生命周期事件也不派发。两条路都能用，看场景。

* * *

### 四、社区版能干什么

社区版的重点不是把完整 Chrome 搬进 Python，而是把 “跑风控 JS” 最常见、最影响成败的几块先做稳：环境画像、可调试性、API 访问链路、可控时间、函数伪装，以及离线资源模型。

#### 4.1 指纹配置：200+ 字段，一份 profile 切换

不传 `environment` 也能跑，内置一套 Chrome / Windows 桌面端的兜底指纹。要切换画像直接传字典，没传到的字段保留默认值：

```
with iv8.JSContext(environment={
    "navigator": {
        "userAgent": "Mozilla/5.0 ... Chrome/124.0.0.0 Safari/537.36",
        "platform": "Win32",
        "language": "zh-CN",
        "languages": ["zh-CN", "en-US"],
        "hardwareConcurrency": 8,
        "deviceMemory": 8,
    },
    "screen": {"width": 1920, "height": 1080},
    "webgl": {
        "vendor": "Google Inc. (NVIDIA)",
        "renderer": "ANGLE (NVIDIA, NVIDIA GeForce RTX 3060)",
    },
    "location": {"href": "https://example.com/page"},
}) as ctx:
    ...

```

时区 / 权限 / Canvas 噪声等行为类配置走 `config` 参数，跟指纹解耦：

```
iv8.JSContext(config={"timezone": "Asia/Shanghai"})

```

#### 4.2 DevTools 调试：能看见，才能补准

`mode="debug"` 启用监控能力，`with_devtools(port=9229)` 拉起 Chrome DevTools Protocol 服务。然后用 Chrome 打开 `devtools://devtools/bundled/inspector.html?ws=localhost:9229`，就能 attach 到 iv8 里的 JSContext：

```
with iv8.JSContext(mode='debug').with_devtools(port=9229, watch_apis=["navigator.userAgent"]) as ctx:
    ctx.eval("vdebugger;")              # 在 Chrome DevTools 中暂停
    ctx.eval("navigator.userAgent;")     # 在 Chrome DevTools 中暂停

```

几个细节：

*   **`debugger;` 被禁了**。很多反爬 JS 喜欢丢 `setInterval(() => debugger, 100)` 这种死循环搞反调试，iv8 在 V8 层把原生 `debugger;` 改成空操作。需要断点时用 `vdebugger;`，行为跟原生 debugger 一致。
    
*   **`console` → `vconsole`**。`with_devtools(enable_console=False)` 之后，标准 `console` 不再转发到 DevTools Console；如果你需要在 DevTools 里看自己的调试输出，用 `vconsole.log()`。
    
*   **`watch_apis`**：把关心的 API 列进去，被读写时自动断点。在重度混淆的代码里追 “谁动了 `navigator.userAgent` / `document.cookie`” 比手工找断点稳得多。
    

![](https://attach.52pojie.cn/forum/202605/05/165747eiifei2r8288kfez.png)

**devtools.png** _(251.37 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg0OTM2M3xmNGNkMGNlYXwxNzc4MzIxMDAwfDIxMzQzMXwyMTA2MjE3&nothumb=yes)

devtools

2026-5-5 16:57 上传

#### 4.3 API 访问调用监控：先知道对方在探什么

debug 模式下，iv8 会记录浏览器 API 的读 / 写 / 方法调用 / 构造调用，还会监控常见反射入口，比如 `Object.keys`、`Object.getOwnPropertyDescriptor`、`Reflect.ownKeys`、`Function.prototype.toString`、`JSON.stringify` 等。

这对补环境很实用：目标 JS 不报错、不提示，它只是在某个分支里默默判断 “环境不对”。API 监控能把这条链路打出来：它先读了哪个字段、枚举了哪个对象、调用了哪个原生函数，后面才知道该补指纹、补枚举顺序，还是补时序。

#### 4.4 逻辑时间与事件循环：不等待真实时钟

事件循环按宏任务 / 微任务两阶段推进：`setTimeout`、`setInterval`、`requestAnimationFrame`、XHR 回调属于宏任务，Promise 回调、`queueMicrotask`、MutationObserver 属于微任务。

框架的重点是 **逻辑时间主动推进**。`time_mode="logical"`（默认）下，`__iv8__.eventLoop.advance(250)` 表示把虚拟时钟推进 250ms，并按事件循环规则处理到期任务；这不是 “加速真实时间”，而是**不需要等待真实墙钟自然流逝**：

```
with iv8.JSContext(time_mode="logical") as ctx:
    ctx.eval("""
        var log = [];
        setTimeout(() => log.push('macro-100'), 100);
        setTimeout(() => log.push('macro-200'), 200);
        Promise.resolve().then(() => log.push('micro'));
    """)
    ctx.eval("window.__iv8__.eventLoop.advance(250)")     # 推进 250ms
    print(ctx.eval("log"))
    # ['micro', 'macro-100', 'macro-200']

```

如果目标逻辑真的依赖真实耗时，比如 POW 或时间差校验，可以切到 `time_mode="system"`，让 JS 可见时间跟系统时间锚定。更多 API 用法参考项目 README。

#### 4.5 wrapNative：临时补丁也要像原生函数

`wrapNative` 把任意 JS 函数伪装成 `[native code]`，`Function.prototype.toString.call(fn)` 也假不了：

```
ctx.eval("""
    var myFunc = window.__iv8__.wrapNative(function(x) { return x * 2; }, 'myFunc');
""")
print(ctx.eval("myFunc.toString()"))   # function myFunc() { [native code] }

```

它适合做一些局部 polyfill 或目标站点临时补丁，比如 `MessageChannel` 这类目标 SDK 只用到很小一部分语义的对象。普通 JS 函数一旦被 `Function.prototype.toString` 打开就露馅；`wrapNative` 至少能把这类临时函数伪装成浏览器原生函数，降低补丁本身的可观测差异。

它也可以用来替换某些 JS 入口做业务侧补丁，比如把自定义函数挂回 `Object.keys`、`MessageChannel` 这类位置时，至少让 `toString()` 看起来仍然像原生函数。社区版暂时不宣传 C++ 层原地 hook 能力，后续这块和最新版完全对齐后再单独展开。

* * *

### 五、实战：两个代表性目标

> 开发过程中精力有限，仅挑选部分有代表性的站点进行过测试。示例脚本可在 GitHub 仓库 `examples/` 目录查看。请在合法授权场景下复现，相关法律边界见[文末免责声明](#十免责声明)。如有站点方认为某条示例不合适，可到仓库提 issue，会及时删除。

#### 5.1 瑞数 6

1.  用 `requests` 发第一次请求，拿 HTML + JS
2.  iv8 里 `page.load` 加载 HTML，内联资源喂进 `resources`
3.  等事件循环跑完，从 `document.cookie` 读到种好的 cookie
4.  在 iv8 里发起 XHR（这一步不真发请求，只是触发瑞数对 XHR 的 hook）
5.  从 `__iv8__.netLog.entries` 读最后一条记录，里面的 `url` 是被瑞数加过后缀的真实 URL，`cookieHeader` 是最终 cookie
6.  Python 侧用这个 URL + cookie 走 `requests` 真实请求

核心代码三十多行（节选自 `examples/海关.py`）：

```
with iv8.JSContext() as ctx:
    resp1 = requests.get(page_url, headers=headers)
    js_code = requests.get(js_url, ...).text

    ctx.expose({
        "baseURL": page_url, "html": resp1.text,
        "headers": [[k, v] for k, v in resp1.raw.headers.items()],
        "resources": {js_url: js_code},
    }, "s1")
    ctx.eval("window.__iv8__.page.load(window.__iv8__.data.s1)")
    ctx.eval("window.__iv8__.eventLoop.sleep(100)")

    cookies_str = ctx.eval(
        "window.__iv8__.netLog.entries[window.__iv8__.netLog.entries.length - 1].cookieHeader"
    )

    ...
    # 生成带后缀URL
    ctx.eval(f"""
        var xhr = new XMLHttpRequest();
        xhr.open('POST', '{url}');
        xhr.send('{body_str}');
    """)

    entry = ctx.eval("window.__iv8__.netLog.entries[window.__iv8__.netLog.entries.length - 1]")
    api_url = entry['url']           # 带签名后缀
    final_cookie = entry['cookieHeader']

    response = requests.post(api_url, json=data, headers={**headers, "Cookie": final_cookie})

```

> 社区版对于瑞数 6.5 版本，如某信营业厅需要结合日志针对性的做些补充

#### 5.2 `__zp_stoken__`

`examples/zp_stoken.py`。接口第一次返回 `code=37`，带回 `seed / name / ts`，需要拉一段 JS 计算出 `__zp_stoken__` 才能正常请求。

签名计算就一行：

```
token = ctx.eval(f"encodeURIComponent((new window.ABC).z({seed}, {ts}));")

```

最后 `session.cookies.set("__zp_stoken__", token)` 重发请求，`code=0`，返回职位列表。

* * *

### 六、性能数据

实测环境 Intel Core i7-14700 / Windows 10 / Python 3.11：

<table><thead><tr><th>维度</th><th>指标</th><th>数据</th></tr></thead><tbody><tr><td><strong>速度</strong></td><td>JSContext 创建 + eval + 销毁</td><td>~3.3 ms / 次</td></tr><tr><td></td><td>简单 eval 吞吐（<code>1+1</code>）</td><td>~950,000 ops/s</td></tr><tr><td></td><td>浏览器 API 调用（navigator / DOM / crypto）</td><td>340,000 – 570,000 ops/s</td></tr><tr><td></td><td>维基百科 JavaScript 条目（440 KB）</td><td>~7 ms / 页（含 Context 创建 + 销毁 ~11.5 ms，串行 ~86 页 / s）</td></tr><tr><td><strong>内存</strong></td><td>首次加载（<code>import iv8</code> + 首个 Context）</td><td>+15 MB</td></tr><tr><td></td><td>单轮峰值增量</td><td>~9 MB</td></tr><tr><td></td><td>100 轮长跑累计漂移</td><td>+2 MB</td></tr><tr><td><strong>多线程</strong></td><td>加速比（2 / 4 / 8 线程）</td><td>1.86x / 3.26x / 4.71x</td></tr></tbody></table>

几个值得说的点：

*   **JSContext 创建 ~3.3ms**。意味着可以 "一个请求一个 Context"，不必复用，每次干净环境。复用 Context 的节流收益其实不大，反而要担心副作用累积。
*   **8 线程 4.71x 加速**。V8 执行期释放 GIL，多线程已经接近多进程效果，不需要进程池 + 序列化通信的复杂度。
*   **100 轮长跑漂移 +2MB**。这是用 `v8::Global<>` 强引用 + 显式释放换来的，没引入 `cppgc` 那套复杂度，长期跑批量任务很关键。

* * *

### 七、最新版还在路上的东西

社区版主打 "够用、稳、能上 prod"。最新版不会靠堆 API 名字凑清单，主要攻的是社区版暂时不适合免费放出来的几块硬能力：

*   **真实网络栈**：基于 Chromium net 模块（QUIC / HTTP3 / WebSocket / proxy chain）的深度裁剪，不是 cronet 封装。JS 侧可以直接发起真实网络请求，TLS 指纹、ALPN 协商、HTTP/2 / HTTP/3 行为与 Chromium net 栈保持一致。  
    ![](https://attach.52pojie.cn/forum/202605/05/165631l9xnnx9sq8sclxl1.png)
    
    **1.png** _(172.17 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjg0OTM2MnxlZmQ5NThmNHwxNzc4MzIxMDAwfDIxMzQzMXwyMTA2MjE3&nothumb=yes)
    
    network
    
    2026-5-5 16:56 上传
    
*   **布局与几何闭环**：级联、继承、盒模型布局、虚拟字体度量、`getBoundingClientRect()` / `offsetWidth` / `getClientRects()` 等几何读数保持内部一致。重点不是做渲染，而是解决滑块 / 拼图验证码里常见的 "读几何 → 走分支 → 校验输入"；更完整的类 Chromium 布局树模型也在继续推进中。
    
*   **Worker 多 Isolate 并行**：每个 Worker 独占一个 v8 Isolate 跑在独立线程上，不再是单 Isolate 模拟的伪并发。
    
*   **可观测语义闭环**：反爬检测不只看 “有没有这个 API”，还会看属性描述符、枚举顺序、异常类型、任务调度、回调批处理时机这些边界行为。最新版会继续把这些 JS 可观测细节收敛到同一套 Chromium 语义里，让目标脚本从反射、异常、时序多个角度看过去都更一致。
    
*   **热路径性能与内存治理**：补环境不是单次跑通就完事，真正上量时瓶颈在 CSS selector 匹配、属性枚举、样式同步、Wrapper 生命周期这些热路径。最新版会继续做缓存、惰性同步和对象生命周期治理，让大批量 Context / 多线程任务跑得更久、更稳、更省内存。
    

内部验证场景里，最新版已经跑通过阿里 v2 / v3 / 234（含 _rand）以及保利威视频下载解密这类更贴近真实生产链路的目标。最新版的方向不是做几个 demo，而是继续把 iv8 往 “能承接复杂站点完整执行链路” 的运行时推进。

* * *

### 八、安装、跑起来

```
pip install iv8

```

支持 Python 3.8 – 3.14，Windows x64 / Linux x64（manylinux 标准）。

仓库：[github.com/HanZzzzz000/iv8](https://github.com/HanZzzzz000/iv8)  
PyPI：[pypi.org/project/iv8](https://pypi.org/project/iv8/)

本文涉及的所有实战脚本在 `examples/` 目录下，clone 之后直接 `python examples/zp_stoken.py` 就能跑（部分脚本依赖额外的 JS 文件，看脚本顶部的注释）。

* * *

### 九、写在最后

补环境这事走到今天，已经不是 "塞几个 navigator 字段就完事" 的阶段了。现代风控系统从指纹一致性、跨 Context 身份、堆栈资源名、API 时序到几何度量，每一层都在打。JS 层的方案不是不能用，是覆盖面已经追不上了。

iv8 的思路其实很朴素 —— 既然 Chromium 已经把 "浏览器宿主" 这个抽象做得很完整，那就在 V8 之上把 Chromium 在 Web 平台层的工作复刻一遍，但只做补环境真正需要的部分。不做渲染、不做 GPU 合成、不做 IPC，把省下来的复杂度全部砸在 "JS 看见的浏览器表面" 上。

社区版以编译产物形式免费提供，允许个人学习与研究。**近期内不会开放源码** —— 反爬场景里源码一暴露，对方下检测点的成本几乎归零（特征字符串、内部结构、函数签名全成了抓手）。编译产物多一道逆向门槛，对补环境项目来说算是必要的对抗护城河，不是封闭姿态。等社区版自身的对抗能力进一步沉淀，会重新评估开源策略。

实际站点跑通了，欢迎提 issue 分享；跑不通，更欢迎反馈 —— 真实目标的回馈是 iv8 迭代最重要的输入。

```
pip install iv8

```

去试试看。

* * *

### 十、免责声明

iv8 仅供学习研究、安全测试与合法的自动化场景使用。

*   本项目及示例代码用于演示 iv8 的 API 设计与补环境思路，**不构成对任何网站的攻击、绕过身份验证或未授权访问的指引或工具**。
    
*   使用者应自行评估使用场景的合规性，遵守目标网站的服务条款、`robots.txt` 与所在地区的法律法规。**作者对任何滥用行为不承担责任**。
    
*   涉及绕过身份验证、抓取受保护的个人信息、对明确禁止采集的接口进行大规模请求等场景，可能触及《网络安全法》《数据安全法》《个人信息保护法》及刑法第 285、286 条等法律法规，请审慎评估，自负其责。
    
*   如果你是文中所涉站点或 SDK 的所有者，认为示例代码涉及不当披露或侵权，请到 GitHub 仓库提 issue，将在评估后及时删除相关内容。
    
*   请优先在自己拥有授权、或目标方明确同意的场景下使用 iv8。
    
*   本项目及示例代码用于演示 iv8 的 API 设计与补环境思路，**不构成对任何网站的攻击、绕过身份验证或未授权访问的指引或工具**。
    
*   使用者应自行评估使用场景的合规性，遵守目标网站的服务条款、`robots.txt` 与所在地区的法律法规。**作者对任何滥用行为不承担责任**。
    
*   涉及绕过身份验证、抓取受保护的个人信息、对明确禁止采集的接口进行大规模请求等场景，可能触及《网络安全法》《数据安全法》《个人信息保护法》及刑法第 285、286 条等法律法规，请审慎评估，自负其责。
    
*   如果你是文中所涉站点或 SDK 的所有者，认为示例代码涉及不当披露或侵权，请到 GitHub 仓库提 issue，将在评估后及时删除相关内容。
    
*   请优先在自己拥有授权、或目标方明确同意的场景下使用 iv8。![](https://avatar.52pojie.cn/data/avatar/000/93/53/52_avatar_middle.jpg)xixicoco cdp 会经常断吗？还有好像有个 rust 的 ai 无头浏览器 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) billpeace 非常感谢  
测试了 * 海 * 关 *$.py，是因为 跑了好几次？  
<div>  
                <b> 建议等待 </b>  
                1-3 小时后再试  
            </div>  
            <div>  
                <b> 合规访问 </b>  
                使用正常浏览器  
            </div>![](https://avatar.52pojie.cn/images/noavatar_middle.gif)521105

> [xixicoco 发表于 2026-5-7 11:58](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55235026&ptid=2106217)  
> cdp 会经常断吗？还有好像有个 rust 的 ai 无头浏览器

目前是没遇到过 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 521105

> [billpeace 发表于 2026-5-7 18:30](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55236989&ptid=2106217)  
> 非常感谢  
> 测试了 * 海 * 关 *$.py，是因为 跑了好几次？

这是海关更新了，请求头加了个参数，重新拉示例代码就行，我推上去了 ![](https://avatar.52pojie.cn/data/avatar/001/84/79/65_avatar_middle.jpg) Aerfa9527 等我试试，和我手动补那个比快点。。。。。。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)billpeace

> Aerfa9527 发表于 2026-5-8 10:07  
> 等我试试，和我手动补那个比快点。。。。。。

等您结果  
手动补  怎么能补全呢？ 怎么核查 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) billpeace

> [521105 发表于 2026-5-8 02:50](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55238035&ptid=2106217)  
> 这是海关更新了，请求头加了个参数，重新拉示例代码就行，我推上去了

感谢 感谢 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) billpeace

> [521105 发表于 2026-5-8 02:50](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55238035&ptid=2106217)  
> 这是海关更新了，请求头加了个参数，重新拉示例代码就行，我推上去了

能问下，  两种写法 是一样的效果？ 还是有区别的？  
1.  ctx.expose({  
        "baseURL": page_url, "html": resp1.text,  
        "headers": [[k, v] for k, v in resp1.raw.headers.items()],  
        "resources": {js_url: js_code},  
    }, "s1")  
    ctx.eval("window.__iv8__.page.load(window.__iv8__.data.s1)")  
    ctx.eval("window.__iv8__.eventLoop.sleep(100)")    
2.  # 1. 执行第一段 js  
    ctx.eval(inline_scripts[1])  
    # 2. 执行外部 JS  
    ctx.eval(js_response.text, name=environment['location']['origin']+ js_path)  
    # 3. 执行最后一个脚本（动态函数名：_$gO() 等）  
    ctx.eval(inline_scripts[-1])![](https://avatar.52pojie.cn/images/noavatar_middle.gif)521105

> [billpeace 发表于 2026-5-8 14:43](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55240831&ptid=2106217)  
> 能问下，  两种写法 是一样的效果？ 还是有区别的？  
> 1.  ctx.expose({

- `expose`：把 Python 数据放到 JS/V8 层，后续 JS 可以从隐藏数据区读取。  
- `page.load`：按页面快照走 DOM/HTML 加载流程，会解析 HTML，并自动处理页面里的 `<script>`，外链 JS 需要能从 `resources` 命中。  
- `sleep(100)`：推进 iv8 的事件循环 / 逻辑时间，处理 timer、microtask、异步任务等。  
两种方式对于瑞数来说，没太大区别，瑞数不检测这块。第一种流程更规范，第二种执行效率高（没有推进事件，比如 DOMContentLoaded/load）