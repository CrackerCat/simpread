> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1841174-1-1.html)

> 补环境过某多多 anti_content 参数 , 吾爱破解 - LCG - LSG | 安卓破解 | 病毒分析 | www.52pojie.cn

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)shenjingwa1989 _ 本帖最后由 shenjingwa1989 于 2023-10-6 23:13 编辑_  
今天的目标是通过补环境的方式来获取某多多 anti_content 参数的生成算法。  
目标网站：aHR0cHM6Ly9waWZhLnBpbmR1b2R1by5jb20v  
目标网站有一个获取推荐商品信息的请求：  
xxx/pifa/recommend/queryRecommendGoods，即查询平台推荐商品信息，  
通过 https://curlconverter.com/，直接生成 python 代码如下：  
[Python] _纯文本查看_ _复制代码_

```
import requests
cookies = {
    '_nano_fp': 'XpEbn5TqXqUyXqXbXC_ce~224ukCxf1u1XfO3s2M',
    'VISITOR_PASS_ID': 'xIq8lYpSf9Jly_cbSFX5YBQvE7aRBi7HoIpLsJkcXrRGLcgOnC9_ydjPGvzfs5xhqcE0_eiQ2uBZQ3mYnpOqz_wy_N8oE6Cw3MqIUF84iW4_f75cfe5acb',
    'webp': 'true',
    'api_uid': 'rBUUQ2UdhyOBIyLGedC+Ag==',
    '_bee': 'jIEH60E6sqwJIevKJ0siLLUdXsS0lK6b',
    '_f77': '6801b4df-cbfd-4907-b5a7-980cb144a304',
    '_a42': '08114944-4f0c-4244-9ad1-858185e42d2a',
    'rckk': 'jIEH60E6sqwJIevKJ0siLLUdXsS0lK6b',
    'ru1k': '6801b4df-cbfd-4907-b5a7-980cb144a304',
    'ru2k': '08114944-4f0c-4244-9ad1-858185e42d2a',
}
 
headers = {
    'authority': 'pifa.pinduoduo.com',
    'accept': '*/*',
    'accept-language': 'zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7',
    'anti-content': '0aqAfx5e-wCE6_qUXbSt_USccG7GNVX_iTIqXxGtmy6PIJUOsxi2hoXR7gl0Wapl0YU5XjnGEycYPanYTjnG4Yn07dnY_Ynp4yHYXq9q44Ca_sZkoP_ymptYXK4JEKNJOq1Yfqmw6jpQTXs574CSWMFxTKRPT1B2MSBjCK99pXpkbdtXxnn2mA4CKYdNnTUrNa_Wvn5mYj5XadnDbNT7xNuVeYXrsTsiipUr6xn2AAXSoYmv6j09YXi9x2d4g3X_qYtVBKKyPCSmgJtXyJxq5saYTGZqfqTRlpTb5YOhV00N6YYhS19o0GORlP_UaLE_ShNaop4QnXP432tm8Q56tWTM9xGN8pQaMfJZRqNtfNfRgf-fAX8x-Lz4WXpG5999gCReM3ihFSkO',
    'cache-control': 'no-cache',
    'content-type': 'application/json',
    # 'cookie': '_nano_fp=XpEbn5TqXqUyXqXbXC_ce~224ukCxf1u1XfO3s2M; VISITOR_PASS_ID=xIq8lYpSf9Jly_cbSFX5YBQvE7aRBi7HoIpLsJkcXrRGLcgOnC9_ydjPGvzfs5xhqcE0_eiQ2uBZQ3mYnpOqz_wy_N8oE6Cw3MqIUF84iW4_f75cfe5acb; webp=true; api_uid=rBUUQ2UdhyOBIyLGedC+Ag==; _bee=jIEH60E6sqwJIevKJ0siLLUdXsS0lK6b; _f77=6801b4df-cbfd-4907-b5a7-980cb144a304; _a42=08114944-4f0c-4244-9ad1-858185e42d2a; rckk=jIEH60E6sqwJIevKJ0siLLUdXsS0lK6b; ru1k=6801b4df-cbfd-4907-b5a7-980cb144a304; ru2k=08114944-4f0c-4244-9ad1-858185e42d2a',
    'origin': 'https://pifa.pinduoduo.com',
    'pragma': 'no-cache',
    'referer': 'https://pifa.pinduoduo.com/',
    'sec-ch-ua': '"Google Chrome";v="117", "Not;A=Brand";v="8", "Chromium";v="117"',
    'sec-ch-ua-mobile': '?0',
    'sec-ch-ua-platform': '"Windows"',
    'sec-fetch-dest': 'empty',
    'sec-fetch-mode': 'cors',
    'sec-fetch-site': 'same-origin',
    'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36',
}
 
json_data = {
    'page': 2,
    'pageSize': 20,
    'queryApi': '',
}
 
response = requests.post(
    'https://pifa.pinduoduo.com/pifa/recommend/queryRecommendGoods',
    cookies=cookies,
    headers=headers,
    json=json_data,
)
 
print(response.text)

```

**目标是通过扣代码 + 框架补环境方式获取 anti-content 参数的生成方式**  
**1、通过搜索方式获知 anti-content 的赋值地方**  
![](https://attach.52pojie.cn/forum/202310/06/230616kw93ipx3lh9wwozl.png)

**002.png** _(60.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzI0NHxjZTJkZDU5MnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

  
分别在 2 处打上断点，发现第 2 处被断下来了：  
![](https://attach.52pojie.cn/forum/202310/06/230614uwshsghgfeh00mct.png)

**003.png** _(226.27 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzI0M3xmMWU0MzUxNHwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

  
**通过分析代码可以知道 e 的值是通过一个 js 的异步 Promise 函数，在 resolve 方法中返回： ![](https://attach.52pojie.cn/forum/202310/06/230611m1rmuzp8sbvdzb01.png)

**004.png** _(177.76 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzI0MnxiNzE1NDk5NHwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

****跟踪 l(e.rawFetch, x) 方法，也是一个 Promise 的异步方法，传入一个 getServerTime，然后执行 B 里面的逻辑， ![](https://attach.52pojie.cn/forum/202310/06/230607dy1bmmnyqvjx0qz6.png)

**005.png** _(19.52 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzI0MXxiZWYwYmRjOHwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

****控制台打印 B，并进入 B：**  
![](https://attach.52pojie.cn/forum/202310/06/230605ouvmp6t6pgit2iuy.png)

**006-1.png** _(4.08 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzI0MHxkMTQ2Y2IyYnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

  
![](https://attach.52pojie.cn/forum/202310/06/230603wiiqw9ay9n7yigc7.png)

**006-2.png** _(12.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzOXwwYjk3NzIwNnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

  
**这里返回：****[JavaScript] _纯文本查看_ _复制代码_

```
new (n(284))({
            serverTime: e
        }).messagePack()
这里的e是前面传入的ServerTime

```** **控制台执行，发现其实这里就是生成 anti_content 参数的地方：**  
![](https://attach.52pojie.cn/forum/202310/06/230601z2f9sq02ii9jiqqq.png)

**007.png** _(34 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzOHxkYjBmYmQwNnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:06 上传

  
**观察参数 n**  
![](https://attach.52pojie.cn/forum/202310/06/230559f2j7z883q4j84gqz.png)

**008-1.png** _(2.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzN3xmYmE0ZTVmZnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:05 上传

  
![](https://attach.52pojie.cn/forum/202310/06/230556v55lgz0uuj7l5eb8.png)

**008-2.png** _(8.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzNnw0ZjJlM2ZiYXwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:05 上传

  
[color=rgba(0, 0, 0, 0.9)] 发现是一个是典型的 webpack 基本结构，  
![](https://attach.52pojie.cn/forum/202310/06/230554j7uxn7x1x4e3x36g.png)

**009.png** _(59 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzNXxiMzNmODc3ZHwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:05 上传

  
[color=rgba(0, 0, 0, 0.9)] 我们只要把整个加载器代码，和要加载的代码找到即可。并把加载器 + 模块代码贴入补环境框架的的 input.js 文件内容  
input.js  
![](https://attach.52pojie.cn/forum/202310/06/230552zjtu9kks88dq8s2r.png)

**010.png** _(37.11 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzNHwyNjM2MTY1MnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:05 上传

  
我们已经找到生成 anti_content 参数的位置，但是直接执行以上代码会报错。原因是什么吗？因为缺失浏览器环境对象。原来的代码可以复制到浏览器直接运行，因为浏览器内置了很多对象，如 window、document、history、screen、localStorage 等等。  
**2、通过补环境框架快速完成环境补充**  
我们可以通过我们之前已经补好的简单版补环境框架来协助我们完成补环境这部分工作。在 input.js 里面是我们要调试的网站代码，另外我们要新装一个 userVar.js 文件用于完成一些网站页面变量初始化：  
[JavaScript] _纯文本查看_ _复制代码_

```
// 网页变量初始化
!function(){
    location.protocol = 'https:';
    location.hostname = 'pifa.pinduoduo.com';
    location.href = 'https://pifa.pinduoduo.com/';
    document.referrer = "https://pifa.pinduoduo.com/";
    document.cookie = "'webp=true; api_uid=rBUUnmUftm8KFSM5G7BkAg==; _nano_fp=XpEbn5d8nqgqn5X8lT_c733lBUlh97cIqwG8knZL"
    localStorage.setItem("_nano_fp","XpEbn5d8nqgqn5X8lT_c733lBUlh97cIqwG8knZL")
}();

```

**如何知道要初始化这些值，为啥是这些值？**我们可以通过开启补环境框架代 {过}{滤} 理，通过设置 config.js 里面的开关：  
ldvm.config.proxy =true; [backcolor=rgba(0, 0, 0, 0.03)]// 是否开启代 {过}{滤} 理 后面通过代 {过}{滤} 理调试我们的代码，[backcolor=rgba(0, 0, 0, 0.03)] **我们可以知道网站加密代码获取或者设置了什么浏览器对象的什么值，我们通过把下面的值一个个补回和浏览器一致即可**，因为前期我已经补过 window、localStorage、document、Plugin、PluginArray、MineType、Location 等对象，这次只要把 History 等对象和一些其他的属性补齐即可： [backcolor=rgba(0, 0, 0, 0.03)][JavaScript] _纯文本查看_ _复制代码_

```
{has|obj:[plugin_ID(1)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[plugin_ID(1)] -> prop:[Symbol(data)]}
{has|obj:[plugin_ID(1)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(1)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(1)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[mimeTypeArray_ID(3)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[mimeTypeArray_ID(3)] -> prop:[Symbol(data)]}
0--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[0]}
{getOwnPropertyDescriptor|obj:[mimeTypeArray_ID(3)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[mimeTypeArray_ID(3)] -> prop:[0]}
1--{设置|obj:[mimeTypeArray_ID(3)] -> 属性:[0],属性类型:[[object MimeType]]}
{defineProperty|obj:[mimeTypeArray_ID(3)] -> prop:[application/pdf]}
{has|obj:[mimeTypeArray_ID(3)] -> prop:[Symbol(data)], result:[true]}
{getOwnPropertyDescriptor|obj:[plugin_ID(1)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(1)] -> prop:[0]}
2--{设置|obj:[plugin_ID(1)] -> 属性:[0],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(1)] -> prop:[application/pdf]}
3--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[1]}
4--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
5--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
{getOwnPropertyDescriptor|obj:[mimeTypeArray_ID(3)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[mimeTypeArray_ID(3)] -> prop:[1]}
6--{设置|obj:[mimeTypeArray_ID(3)] -> 属性:[1],属性类型:[[object MimeType]]}
{defineProperty|obj:[mimeTypeArray_ID(3)] -> prop:[text/pdf]}
{has|obj:[mimeTypeArray_ID(3)] -> prop:[Symbol(data)], result:[true]}
{getOwnPropertyDescriptor|obj:[plugin_ID(1)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(1)] -> prop:[1]}
7--{设置|obj:[plugin_ID(1)] -> 属性:[1],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(1)] -> prop:[text/pdf]}
{has|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)]}
8--{获取|obj:[pluginArray_ID(5)] -> 属性:[length],返回值:[0]}
{getOwnPropertyDescriptor|obj:[pluginArray_ID(5)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[0]}
{getPrototypeOf|obj:[plugin_ID(1)]}
9--{设置|obj:[pluginArray_ID(5)] -> 属性:[0],属性类型:[[object Plugin]]}
10--{获取|obj:[plugin_ID(1)] -> 属性:[name],返回值:[PDF Viewer]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[PDF Viewer]}
{has|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(6)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[plugin_ID(6)] -> prop:[Symbol(data)]}
{has|obj:[plugin_ID(6)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(6)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(6)] -> prop:[Symbol(data)], result:[true]}
11--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
12--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
13--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
14--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
15--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(6)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(6)] -> prop:[0]}
16--{设置|obj:[plugin_ID(6)] -> 属性:[0],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(6)] -> prop:[application/pdf]}
17--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
18--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
19--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
20--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
21--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(6)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(6)] -> prop:[1]}
22--{设置|obj:[plugin_ID(6)] -> 属性:[1],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(6)] -> prop:[text/pdf]}
23--{获取|obj:[pluginArray_ID(5)] -> 属性:[length],返回值:[1]}
{getOwnPropertyDescriptor|obj:[pluginArray_ID(5)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[1]}
{getPrototypeOf|obj:[plugin_ID(6)]}
24--{设置|obj:[pluginArray_ID(5)] -> 属性:[1],属性类型:[[object Plugin]]}
25--{获取|obj:[plugin_ID(6)] -> 属性:[name],返回值:[Chrome PDF Viewer]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[Chrome PDF Viewer]}
{has|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(9)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[plugin_ID(9)] -> prop:[Symbol(data)]}
{has|obj:[plugin_ID(9)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(9)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(9)] -> prop:[Symbol(data)], result:[true]}
26--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
27--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
28--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
29--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
30--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(9)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(9)] -> prop:[0]}
31--{设置|obj:[plugin_ID(9)] -> 属性:[0],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(9)] -> prop:[application/pdf]}
32--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
33--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
34--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
35--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
36--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(9)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(9)] -> prop:[1]}
37--{设置|obj:[plugin_ID(9)] -> 属性:[1],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(9)] -> prop:[text/pdf]}
38--{获取|obj:[pluginArray_ID(5)] -> 属性:[length],返回值:[2]}
{getOwnPropertyDescriptor|obj:[pluginArray_ID(5)] -> prop:[2],type:[[object Undefined]]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[2]}
{getPrototypeOf|obj:[plugin_ID(9)]}
39--{设置|obj:[pluginArray_ID(5)] -> 属性:[2],属性类型:[[object Plugin]]}
40--{获取|obj:[plugin_ID(9)] -> 属性:[name],返回值:[Chromium PDF Viewer]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[Chromium PDF Viewer]}
{has|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(12)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[plugin_ID(12)] -> prop:[Symbol(data)]}
{has|obj:[plugin_ID(12)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(12)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(12)] -> prop:[Symbol(data)], result:[true]}
41--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
42--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
43--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
44--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
45--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(12)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(12)] -> prop:[0]}
46--{设置|obj:[plugin_ID(12)] -> 属性:[0],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(12)] -> prop:[application/pdf]}
47--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
48--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
49--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
50--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
51--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(12)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(12)] -> prop:[1]}
52--{设置|obj:[plugin_ID(12)] -> 属性:[1],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(12)] -> prop:[text/pdf]}
53--{获取|obj:[pluginArray_ID(5)] -> 属性:[length],返回值:[3]}
{getOwnPropertyDescriptor|obj:[pluginArray_ID(5)] -> prop:[3],type:[[object Undefined]]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[3]}
{getPrototypeOf|obj:[plugin_ID(12)]}
54--{设置|obj:[pluginArray_ID(5)] -> 属性:[3],属性类型:[[object Plugin]]}
55--{获取|obj:[plugin_ID(12)] -> 属性:[name],返回值:[Microsoft Edge PDF Viewer]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[Microsoft Edge PDF Viewer]}
{has|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(15)] -> prop:[Symbol(data)], result:[false]}
{defineProperty|obj:[plugin_ID(15)] -> prop:[Symbol(data)]}
{has|obj:[plugin_ID(15)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(15)] -> prop:[Symbol(data)], result:[true]}
{has|obj:[plugin_ID(15)] -> prop:[Symbol(data)], result:[true]}
56--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
57--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
58--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
59--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
60--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(15)] -> prop:[0],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(15)] -> prop:[0]}
61--{设置|obj:[plugin_ID(15)] -> 属性:[0],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(15)] -> prop:[application/pdf]}
62--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[length],返回值:[2]}
63--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[0],返回值类型:[[object MimeType]]}
64--{获取|obj:[mimeType_ID(2)] -> 属性:[type],返回值:[application/pdf]}
65--{获取|obj:[mimeTypeArray_ID(3)] -> 属性:[1],返回值类型:[[object MimeType]]}
66--{获取|obj:[mimeType_ID(4)] -> 属性:[type],返回值:[text/pdf]}
{getOwnPropertyDescriptor|obj:[plugin_ID(15)] -> prop:[1],type:[[object Undefined]]}
{defineProperty|obj:[plugin_ID(15)] -> prop:[1]}
67--{设置|obj:[plugin_ID(15)] -> 属性:[1],属性类型:[[object MimeType]]}
{defineProperty|obj:[plugin_ID(15)] -> prop:[text/pdf]}
68--{获取|obj:[pluginArray_ID(5)] -> 属性:[length],返回值:[4]}
{getOwnPropertyDescriptor|obj:[pluginArray_ID(5)] -> prop:[4],type:[[object Undefined]]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[4]}
{getPrototypeOf|obj:[plugin_ID(15)]}
69--{设置|obj:[pluginArray_ID(5)] -> 属性:[4],属性类型:[[object Plugin]]}
70--{获取|obj:[plugin_ID(15)] -> 属性:[name],返回值:[WebKit built-in PDF]}
{defineProperty|obj:[pluginArray_ID(5)] -> prop:[WebKit built-in PDF]}
{has|obj:[pluginArray_ID(5)] -> prop:[Symbol(data)], result:[true]}
284
85
71--{获取|obj:[window] -> 属性:[navigator],返回值类型:[[object Navigator]]}
72--{获取|obj:[window] -> 属性:[Date],返回值类型:[[object Function]]}
73--{获取|obj:[window] -> 属性:[Math],返回值类型:[[object Math]]}
74--{获取|obj:[window] -> 属性:[history],返回值类型:[[object History]]}
{getPrototypeOf|obj:[document]}
75--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
{has|obj:[document] -> prop:[ontouchstart], result:[false]}
76--{获取|obj:[window.Date] -> 属性:[now],返回值类型:[[object Function]]}
{apply|function:[window.Date.now], argumentsList: [],result:[1696588668748]}
77--{获取|obj:[window.Date] -> 属性:[now],返回值类型:[[object Function]]}
{apply|function:[window.Date.now], argumentsList: [],result:[1696588668768]}
78--{获取|obj:[window] -> 属性:[screen],返回值类型:[[object Screen]]}
79--{获取|obj:[window.screen] -> 属性:[availWidth],返回值:[1920]}
80--{获取|obj:[window] -> 属性:[screen],返回值类型:[[object Screen]]}
81--{获取|obj:[window.screen] -> 属性:[availHeight],返回值:[1040]}
82--{获取|obj:[window] -> 属性:[outerHeight],返回值:[1040]}
83--{获取|obj:[window] -> 属性:[outerWidth],返回值:[1920]}
84--{获取|obj:[window] -> 属性:[outerHeight],返回值:[1040]}
85--{获取|obj:[window] -> 属性:[outerWidth],返回值:[1920]}
86--{获取|obj:[window] -> 属性:[callPhantom],返回值:[undefined]}
87--{获取|obj:[window] -> 属性:[_phantom],返回值:[undefined]}
88--{获取|obj:[window] -> 属性:[Buffer],返回值:[undefined]}
89--{获取|obj:[window] -> 属性:[emit],返回值:[undefined]}
90--{获取|obj:[window] -> 属性:[spawn],返回值:[undefined]}
91--{获取|obj:[window.navigator] -> 属性:[webdriver],返回值:[false]}
92--{获取|obj:[window] -> 属性:[domAutomation],返回值:[undefined]}
93--{获取|obj:[window] -> 属性:[domAutomationController],返回值:[undefined]}
{getPrototypeOf|obj:[pluginArray_ID(5)]}
94--{获取|obj:[window.navigator] -> 属性:[plugins],返回值类型:[[object PluginArray]]}
{getPrototypeOf|obj:[pluginArray_ID(5)]}
95--{获取|obj:[window.navigator] -> 属性:[plugins],返回值类型:[[object PluginArray]]}
96--{获取|obj:[pluginArray_ID(5)] -> 属性:[length],返回值:[5]}
97--{获取|obj:[window.navigator] -> 属性:[languages],返回值类型:[[object Array]]}
98--{获取|obj:[window] -> 属性:[vendor],返回值:[undefined]}
99--{获取|obj:[window] -> 属性:[Modernizr],返回值:[undefined]}
100--{获取|obj:[window] -> 属性:[chrome],返回值类型:[[object Object]]}
{has|obj:[window.navigator] -> prop:[webdriver], result:[true]}
101--{获取|obj:[window.navigator] -> 属性:[hasOwnProperty],返回值类型:[[object Function]]}
{getOwnPropertyDescriptor|obj:[window.navigator] -> prop:[webdriver],type:[[object Undefined]]}
{apply|function:[window.navigator.hasOwnProperty], argumentsList: [webdriver],result:[false]}
102--{获取|obj:[window.history] -> 属性:[back],返回值类型:[[object Function]]}
103--{获取|obj:[window.history] -> 属性:[back],返回值类型:[[object Function]]}
104--{获取|obj:[window.history.back] -> 属性:[toString],返回值类型:[[object Function]]}
105--{获取|obj:[window.history.back] -> 属性:[Symbol()],返回值:[function back() { [native code] }]}
{apply|function:[window.history.back.toString], argumentsList: [],result:[function back() { [native code] }]}
{getPrototypeOf|obj:[document]}
106--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
107--{获取|obj:[document] -> 属性:[getElementById],返回值类型:[[object Function]]}
108--{获取|obj:[document.getElementById] -> 属性:[toString],返回值类型:[[object Function]]}
109--{获取|obj:[document.getElementById] -> 属性:[Symbol()],返回值:[function getElementById() { [native code] }]}
{apply|function:[window.history.back.toString], argumentsList: [],result:[function getElementById() { [native code] }]}
{getPrototypeOf|obj:[location]}
110--{获取|obj:[window] -> 属性:[location],返回值类型:[[object Location]]}
111--{获取|obj:[location] -> 属性:[href],返回值:[https://pifa.pinduoduo.com/]}
{getPrototypeOf|obj:[location]}
112--{获取|obj:[window] -> 属性:[location],返回值类型:[[object Location]]}
113--{获取|obj:[location] -> 属性:[href],返回值:[https://pifa.pinduoduo.com/]}
114--{获取|obj:[window] -> 属性:[DeviceOrientationEvent],返回值类型:[[object Function]]}
115--{获取|obj:[window] -> 属性:[DeviceMotionEvent],返回值类型:[[object Function]]}
116--{获取|obj:[window.navigator] -> 属性:[userAgent],返回值:[Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36]}
{getPrototypeOf|obj:[document]}
117--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
118--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; ]}
{getPrototypeOf|obj:[document]}
119--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
120--{设置|obj:[document] -> 属性:[cookie],属性值:[_nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; expires=Mon, 20 Feb 2051 10:37:48 GMT; path=/]}
{getPrototypeOf|obj:[localStorage]}
121--{获取|obj:[window] -> 属性:[localStorage],返回值类型:[[object Storage]]}
122--{获取|obj:[localStorage] -> 属性:[getItem],返回值类型:[[object Function]]}
{has|obj:[localStorage] -> prop:[_nano_fp], result:[true]}
123--{获取|obj:[localStorage] -> 属性:[_nano_fp],返回值:[XpEbn5d8nqgqn5X8lT_c733lBUlh97cIqwG8knZL]}
{apply|function:[localStorage.getItem], argumentsList: [_nano_fp],result:[XpEbn5d8nqgqn5X8lT_c733lBUlh97cIqwG8knZL]}
{getPrototypeOf|obj:[document]}
124--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
125--{获取|obj:[document] -> 属性:[referrer],返回值:[https://pifa.pinduoduo.com/]}
{getPrototypeOf|obj:[document]}
126--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
127--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; _nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; ]}
{getPrototypeOf|obj:[document]}
128--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
129--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; _nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; ]}
{getPrototypeOf|obj:[document]}
130--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
131--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; _nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; ]}
{getPrototypeOf|obj:[document]}
132--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
133--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; _nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; ]}
{getPrototypeOf|obj:[document]}
134--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
135--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; _nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; ]}
{getPrototypeOf|obj:[document]}
136--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
137--{获取|obj:[document] -> 属性:[cookie],返回值:['webp=true; _nano_fp=XpEbn5dJl0Ejl0EYlT_cUsnO~T1Wo7~xPnvO_st4; ]}
{getPrototypeOf|obj:[location]}
138--{获取|obj:[window] -> 属性:[location],返回值类型:[[object Location]]}
139--{获取|obj:[location] -> 属性:[href],返回值:[https://pifa.pinduoduo.com/]}
{getPrototypeOf|obj:[location]}
140--{获取|obj:[window] -> 属性:[location],返回值类型:[[object Location]]}
141--{获取|obj:[location] -> 属性:[port],返回值:[undefined]}
142--{获取|obj:[window] -> 属性:[opr],返回值:[undefined]}
143--{获取|obj:[window] -> 属性:[opera],返回值:[undefined]}
144--{获取|obj:[window.navigator] -> 属性:[userAgent],返回值:[Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36]}
145--{获取|obj:[window.navigator] -> 属性:[userAgent],返回值:[Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36]}
146--{获取|obj:[window] -> 属性:[HTMLElement],返回值类型:[[object Function]]}
147--{获取|obj:[window.HTMLElement] -> 属性:[toString],返回值类型:[[object Function]]}
148--{获取|obj:[window.HTMLElement] -> 属性:[Symbol()],返回值:[function HTMLElement() { [native code] }]}
{apply|function:[window.history.back.toString], argumentsList: [],result:[function HTMLElement() { [native code] }]}
149--{获取|obj:[window] -> 属性:[safari],返回值:[undefined]}
{getPrototypeOf|obj:[document]}
150--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
{getPrototypeOf|obj:[document]}
151--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
152--{获取|obj:[document] -> 属性:[documentMode],返回值:[undefined]}
153--{获取|obj:[window] -> 属性:[StyleMedia],返回值:[undefined]}
154--{获取|obj:[window] -> 属性:[navigate],返回值:[undefined]}
155--{获取|obj:[window] -> 属性:[chrome],返回值类型:[[object Object]]}
156--{获取|obj:[window] -> 属性:[chrome],返回值类型:[[object Object]]}
157--{获取|obj:[window.chrome] -> 属性:[webstore],返回值:[undefined]}
158--{获取|obj:[window] -> 属性:[chrome],返回值类型:[[object Object]]}
159--{获取|obj:[window.chrome] -> 属性:[runtime],返回值:[undefined]}
{getPrototypeOf|obj:[document]}
160--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
161--{获取|obj:[document] -> 属性:[addEventListener],返回值类型:[[object Function]]}
{apply|function:[document.addEventListener], argumentsList: [mousedown,[object Object],true],result:[undefined]}
{getPrototypeOf|obj:[document]}
162--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
163--{获取|obj:[document] -> 属性:[addEventListener],返回值类型:[[object Function]]}
{apply|function:[document.addEventListener], argumentsList: [mousemove,[object Object],true],result:[undefined]}
{getPrototypeOf|obj:[document]}
164--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
165--{获取|obj:[document] -> 属性:[addEventListener],返回值类型:[[object Function]]}
{apply|function:[document.addEventListener], argumentsList: [click,[object Object],true],result:[undefined]}
{getPrototypeOf|obj:[document]}
166--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
167--{获取|obj:[document] -> 属性:[addEventListener],返回值类型:[[object Function]]}
{apply|function:[document.addEventListener], argumentsList: [scroll,[object Object],true],result:[undefined]}
168--{获取|obj:[window.Date] -> 属性:[now],返回值类型:[[object Function]]}
{apply|function:[window.Date.now], argumentsList: [],result:[1696588669004]}
169--{获取|obj:[window.Math] -> 属性:[random],返回值类型:[[object Function]]}
{apply|function:[window.Math.random], argumentsList: [],result:[0.8347512154120387]}
170--{获取|obj:[window.Math] -> 属性:[pow],返回值类型:[[object Function]]}
{apply|function:[window.Math.pow], argumentsList: [2,52],result:[4503599627370496]}
171--{获取|obj:[window.Math] -> 属性:[random],返回值类型:[[object Function]]}
{apply|function:[window.Math.random], argumentsList: [],result:[0.9820538049982175]}
172--{获取|obj:[window.Math] -> 属性:[pow],返回值类型:[[object Function]]}
{apply|function:[window.Math.pow], argumentsList: [2,30],result:[1073741824]}
{getPrototypeOf|obj:[document]}
173--{获取|obj:[window] -> 属性:[document],返回值类型:[[object HTMLDocument]]}
{ownKeys|obj:[document],type:[[object Array]]}
174--{获取|obj:[window.Date] -> 属性:[now],返回值类型:[[object Function]]}
{apply|function:[window.Date.now], argumentsList: [],result:[1696588669049]}
result: 0aqAfx5e-wCE6_qUXbSt_USccG7GNVX_iTIqXxGtmy6PIJUOsxi2hoXR7gl0Wapl0YU5XjnGEycYPanYTjnG4Yn07dnY_Ynp4yHYXq9q44Ca_sZkoP_ymptYXK4JEKNJOq1Yfqmw6jpQTXs574CSWMFxTKRPT1B2MSBjCK99pXpkbdtXxnn2mA4CKYdNnTUrNa_Wvn5mYj5XadnDbNT7xNuVeYXrsTsiipUr6xn2AAXSoYmv6j09YXi9x2d4g3X_qYtVBKKyPCSmgJtXyJxq5saYTGZqfqTRlpTb5YOhV00N6YYhS19o0GORlP_UaLE_ShNaop4QnXP432tm8Q56tWTM9xGN8pQaMfJZRqNtfNfRgf-fAX8x-Lz4WXpG5999gCReM3ihFSkO

```

[我们可以看到其实网站本身至少获取或者设置 174 次浏览器对象的属性或方法。网站也可能对这些浏览器对象进行检测。  
整个过程大概只要 1-2 个小时可以补好。其实效率还是比一步步去解混淆，单步 debug 要高效得多。[backcolor=rgba(0, 0, 0, 0.03)] 最后生成的 anti_content，我们带入前面生成的 python 代码，可以成功返回数据： ![](https://attach.52pojie.cn/forum/202310/06/230550skcxzo2j622ibv6y.png)

**011.png** _(242.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzM3w3ZDAwNjJmMXwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:05 上传

这里如果我们不传 anti_cotent 或者如果 anti_cotent 传的不对会返回错误： ![](https://attach.52pojie.cn/forum/202310/06/230546l1dwpnww8lwaapll.png)

**012.png** _(43.72 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIzMnxlMjczMjI1OHwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 23:05 上传

以上就是通过环境框架获取某多多网站 anti_content 参数的介绍。因为公众号字数有限，整段代码贴出来上万行，有兴趣要源码的可以和我私聊。或者关注飞翔逆向公众号。  
[001.png](forum.php?mod=attachment&aid=MjY0NzIyMXw1MjVmZjU2ZnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes) _(60.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIyMXw1MjVmZjU2ZnwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 22:57 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202310/06/225723tr5hyu9vraezycyu.png) [003.png](forum.php?mod=attachment&aid=MjY0NzIyMnxhOWQ5Y2UxMXwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes) _(226.27 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIyMnxhOWQ5Y2UxMXwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 22:58 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202310/06/225820iwmz7pcnzyv1ppzm.png) [image.png](forum.php?mod=attachment&aid=MjY0NzIwN3w5ZGYzZDUyYXwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes) _(60.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NzIwN3w5ZGYzZDUyYXwxNjk2NjU1NDQ2fDIxMzQzMXwxODQxMTc0&nothumb=yes)

2023-10-6 22:39 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202310/06/223942vtvl56gs6ul5lhub.png) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif)CNASA 天下商家苦拼多多久矣 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) hannoch  
天下商家苦拼多多久矣 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) a382662643 某多多能导出订单信息就更好了