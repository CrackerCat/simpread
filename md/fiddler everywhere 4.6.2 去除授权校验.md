> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1837902-1-1.html)

> [md]# 前言官方上了安全措施，对大量文件进行 hash 校验，所以搞起来些许有些麻烦了。

![](https://avatar.52pojie.cn/data/avatar/000/42/71/99_avatar_middle.jpg)cqc520

前言
==

官方上了安全措施，对大量文件进行 hash 校验，所以搞起来些许有些麻烦了。

这篇文章的实践是在 Windows 下进行的。

在过程中涉及到一些断点调试，根据变量值判断当前程序所处状态来寻找关键修改点；

因此，在 Linux，Mac 下面操作起来可能不是很方便。

不过，我的步骤都有截图，我在每个修改后面都标了 step，你可以根据其中的 16 进制来查找替换。

另外，我不清楚 DLL 是否通用，通用的话直接复制就行了。

0 前置信息
======

0.1 大致思路
--------

1.  修改文件，使得一些篡改检测失效
2.  搭建本地简易服务器伪造响应

0.2 本文常用 IL 对应十六进制
------------------

<table><thead><tr><th>IL 指令</th><th>十六进制</th></tr></thead><tbody><tr><td>ret</td><td>0x2A</td></tr><tr><td>ldc.i4.0</td><td>0x16</td></tr><tr><td>ldc.i4.1</td><td>0x17</td></tr><tr><td>ldc.i4.2</td><td>0x18</td></tr><tr><td>nop</td><td>0x00</td></tr></tbody></table>

1 安装启动
======

1.  下载 v4.6.2 并安装
2.  默认安装在 C 盘，但是会存在操作权限问题不方便后面修改文件
3.  移动文件到自定义的目录
4.  打开软件
    
    ![](https://attach.52pojie.cn/forum/202309/24/182942qndyy464nn7z25dd.png)
    
    **1-1.png** _(67.48 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTI5M3w2MWYwNzE4NnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/1-1
    
    2023-9-24 18:29 上传
    

2 尝试开启控制台
=========

查看安装文件夹的内容，可以很确定这是一个基于 Electron 的应用，那么它就能够开启控制台。

开启控制台后，我们能通过 F5 即时刷新页面，免去修改过程中重开的麻烦。

使用 VS Code 打开软件目录

![](https://attach.52pojie.cn/forum/202309/24/183005ntwgovoroewrogrv.png)

**2-1.png** _(620.06 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTI5NHxmZTRjOWY1NHwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/2-1

2023-9-24 18:30 上传

在 `./resources/app/out` 文件夹中搜索内容 `BrowserWindow(`

在每个结果后面加上 `xx.webContents.openDevTools();`, 次数 xx 要替换为具体变量

在 VS Code 控制台输入内容回车 `& '.\Fiddler Everywhere.exe'`, 尝试启动软件；

没有反应，重试也不行；

那就撤销修改再启动，欸，可以。

那说明它自带的启动器有做文件修改校验，得想办法跳过。

3 替换 Electron
=============

由于软件是基于 Electron 的，那么我们可以试试用 Electron 官方的软件包去启动。

3.1 下载 Electron
---------------

1.  打开地址：[Electron Release](https://github.com/electron/electron/releases)
2.  选一个顺眼的下载，我选当前较新的 `v26.2.2`

3.2 替换 Electron
---------------

1.  删除原有的文件，除了 `resources` 文件夹
2.  将下载的 Electron 中的文件赋值过去，除了 `resources` 文件夹
3.  控制台输入 `& '.\electron.exe'` 回车，欸启动了。
    
    ![](https://attach.52pojie.cn/forum/202309/24/183111qvlxx1vomovxaxn1.png)
    
    **2-1.png** _(139.1 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTI5NXw2ZTZkZjlmY3wxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/3/2-1
    
    2023-9-24 18:31 上传
    
4.  此时去重试第二步，你会发现，闪退了，这时候就到第 4 步了。

4 移除 main.js 的校验
================

4.1 定位错误位置
----------

1.  查看控制台输出，可以发现由一个报错 `Unable to start server process, closed with code:252`
    
    ![](https://attach.52pojie.cn/forum/202309/24/183149wslas2ppjxlc6gxl.png)
    
    **1-1.png** _(544.98 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTI5Nnw0NzE3Y2FhYXwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/4/1-1
    
    2023-9-24 18:31 上传
    
2.  搜索该字符串并局部格式化代码
    
3.  可以看出，该处启动了一个外部程序，但是这个程序执行失败了
    
4.  加一个日志打印看看输出
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==) ![](https://attach.52pojie.cn/forum/202309/24/183213m6v66ycrv6p1yqi8.png) 
    
    **1-2.png** _(338.9 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTI5N3wyNmFiZDQxNnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/4/1-2
    
    2023-9-24 18:32 上传
    
5.  输出了程序路径以及执行参数：`resources\app\out\WebServer\Fiddler.WebUi.exe --port=16225 --logDirectory=Path\To\Logs --verboseLogging=false --logMaxSize=4194304`
    
6.  拼接并执行，表现与日志记录一致
    
    ```
      PS D:\Software\Fiddler Everywhere\resources\app\out\WebServer> ./Fiddler.WebUi.exe --port=16225 "--logDirectory=C:\Users\msojocs\AppData\Roaming\Fiddler Everywhere\Logs" --verboseLogging=false --logMaxSize=4194304
      Error while calculating application port! Contact Support
      PS D:\Software\Fiddler Everywhere\resources\app\out\WebServer> $LastExitCode 
      252
    
    ```
    
    4.2 找出 main.js 校验关键点
    --------------------
    
7.  dnSpy 启动！
    
8.  填写路径参数
    
    ![](https://attach.52pojie.cn/forum/202309/24/183221ghijk7zjfj5wuzgy.png)
    
    **2-1.png** _(29.01 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTI5OHw4OGI4MzI2YXwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/4/2-1
    
    2023-9-24 18:32 上传
    
9.  通过断点调试，可以找到图中的一个 hash 比较代码，我们需要把 flag 置成总是正确的
    
    ![](https://attach.52pojie.cn/forum/202309/24/183244zaq0rvbincr8fuqc.png)
    
    **2-2.png** _(16.61 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTI5OXxkN2JjNTUxMHwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/4/2-2
    
    2023-9-24 18:32 上传
    
10.  正常情况下，Equals 返回为 true，取反就是 false，也就是说 flag 总是 false 是需要达到的结果;
    
    IL 代码中使用 0 与 Equals 比较，Equals 取值有 0 和 1，要让比较结果总是 false，就把固定值 0 换成 2;
    
    即 `ldc.i4.0` 换成 `ldc.i4.2`。
    
11.  你可以尝试再 dnSpy 中修改代码保存，这是行不通的，会报错，所以只能修改二进制代码。
    

4.3 解除 main.js 校验
-----------------

直接在 VS Code 里面搜索十六进制，直到是唯一值（0x16FE013A），然后把 0x16 改成 0x18。(step1, Fiddler.WebUi.dll)

前面有个十六进制地址，但那是包含偏移量的，所以与实际有差别。你可以直接打开一个新的这个 dll，那么地址回事正确的，但是不如搜索来的快。

![](https://attach.52pojie.cn/forum/202309/24/183314b0icjsa961yid1g9.png)

**2-3.png** _(48.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMwMHw3Njc5NjRmOXwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/4/2-3

2023-9-24 18:33 上传

改完之后，替换原文件，执行命令。

你会发现错误变了，原来它还对自身进行修改检测。

```
Unhandled exception. System.TypeInitializationException: The type initializer for '<Module>' threw an exception.
 ---> System.Exception: Fiddler.WebUi is tampered.
   at Fiddler.WebUi.Identifiers.AlgoSchemaID.ReflectProcessor()
   at StopAnnotation()
   at .cctor()
   --- End of inner exception stack trace ---

```

4.4 定位 WebUi 的监测点并跳过检测
----------------------

粗略浏览上一步报错的方法 `Fiddler.WebUi.Identifiers.AlgoSchemaID.ReflectProcessor`，就是检查有没有被篡改，篡改抛出异常，否则直接返回。

那直接返回就行了，把开头的 `2B 05` 修改为 `2A 00`。(step2, Fiddler.WebUi.dll)

最后，启动，OK。

![](https://attach.52pojie.cn/forum/202309/24/183330ef4bc5ti3dio9dcc.png)

**2-4.png** _(78.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMwMXwwZmQzYzNlM3wxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/4/2-4

2023-9-24 18:33 上传

5 伪造服务器
=======

5.1 试着伪造一个请求数据
--------------

控制台按下 F5，可以看到一个 `versions` 请求，先伪造它试试

![](https://attach.52pojie.cn/forum/202309/24/183356e8usxzup9pu0cr98.png)

**1-1.png** _(157.84 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMwMnwzYmMyZjc1ZnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/5/1-1

2023-9-24 18:33 上传

使用 js 创建一个简单的服务器

```
(() => {
  const http = require('http')
  const path = require('path')
  const fs = require('fs')
  http.createServer((req, res) => {
    res.setHeader('Content-Type', 'application/json; charset=utf-8')
    const fullPath = req.url
    const url = new URL(fullPath, 'http://127.0.0.1:5678')
    console.log(url.pathname)
    let data = ''
    if (url != null) {
      const loc = path.resolve(__dirname, `./file/${url.pathname}`)
      if (fs.existsSync(loc + '.json')) { // 在后面加上.json后缀，存在就用这个
        data = fs.readFileSync(loc + '.json').toString()
      }
      else if (fs.existsSync(loc)) { // 直接使用原始路径
        data = fs.readFileSync(loc).toString()
      }
    }

    res.end(data)
  }).listen(5678)
})();

```

之后请求地址会由 `https://api.getfiddler.com/versions` 变成 `http://127.0.0.1:5678/api.getfiddler.com/versions`

搜索 `https://api.getfiddler.com` 替换成 `http://127.0.0.1:5678/api.getfiddler.com`

为 versions 地址创建对应文件并把刚刚的响应体复制进去；按下 F5，没有什么意外发生。

5.2 移除 UI 层的签名验证
----------------

虽然没有什么异常发生，但是查看日志会发现有报错（日志在 `Help->Open Logs Folder`）：

> [Error] [Angular] Not able to fetch the minimum supported FE version. Error: Cannot find signature!

搜素相关文本，可以看到一个 `verifyResponse` 的验证方法在检查签名；

搜索 `verifyResponse`，发现只有两个结果；那么很明显一个实现，一个调用，把调用删了就行。

![](https://attach.52pojie.cn/forum/202309/24/183415aq3ryrmqe0y609qr.png)

**2-1.png** _(134.8 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMwM3w5MTViMWRkNnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/5/2-1

2023-9-24 18:34 上传

再按下 F5，发现日志开始报错。

重启软件无法启动，跟之前情况一样，应该是检测到 main.xxx.js 被修改了。

5.3 移除 mian.xxx.js 的检测
----------------------

使用 dnSpy 调试，与之前一致，把 `ldc.i4.0` 换成 `ldc.i4.2`。(step3, Fiddler.WebUi.dll)

![](https://attach.52pojie.cn/forum/202309/24/183421hbbhe663cz8zdodt.png)

**3-1.png** _(46.96 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMwNHw3MDczMTI3NXwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/5/3-1

2023-9-24 18:34 上传

启动 `Fiddler.WebUi.exe`，成功。

启动 `electron.exe` 发现自动退出登录了。

重新登录账户，又会自动退出，再日志中看到如下信息：

看来在 dll 中也有对之前出现的签名进行验证。

```
[Error] [NETCore] Unexpected backend error.
FiddlerBackend.Contracts.ValidationException: Unable to verify response signature
   at FiddlerBackendSDK.Core.Http.Client.SignedResponseHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at Microsoft.Extensions.Http.Logging.LoggingScopeHttpMessageHandler.<SendAsync>g__Core|5_0(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpClient.<SendAsync>g__Core|83_0(HttpRequestMessage request, HttpCompletionOption completionOption, CancellationTokenSource cts, Boolean disposeCts, CancellationTokenSource pendingRequestsCts, CancellationToken originalCancellationToken)
   at FiddlerBackendSDK.Core.Http.Client.AuthenticatedHttpClient.<>c__DisplayClass10_0`1.ConnectionDescriptorListener.MoveNext()

```

5.4 移除 dll 中的响应数据签名验证
---------------------

由于这是在使用中发生的检测，所以此处要使用 “附加到进程” 的方式进行调试。（你可能需要打开 dnSpy 选项中的“调试从进程内存加载的文件”）

1.  在 `FiddlerBackend.Contracts.ValidationException` 的开头下断点
2.  重新登录，可以看见参数内容与日志内容吻合。
    
    ![](https://attach.52pojie.cn/forum/202309/24/183445l2oq0yajzyo2ngwo.png)
    
    **4-1.png** _(42.11 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTMwNXxkM2U3ZGY2MnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/5/4-1
    
    2023-9-24 18:34 上传
    
3.  点击上一个堆栈，可以发现检测到一个字符串为空，大概率是那个签名。
    
    ![](https://attach.52pojie.cn/forum/202309/24/183458jh8fsagzhzhfsdsc.png)
    
    **4-2.png** _(65.67 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTMwNnw1ODcyNTA5MnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/5/4-2
    
    2023-9-24 18:34 上传
    
4.  此处不对判空做修改，因为跳过判空，它肯定要去计算签名是否正确，这样改动有点麻烦，最好改动是像上面一样的微小改动。
    
    所以，我们补一个签名上去，在它判断签名是否正确的 if 分支那里做修改。
    
5.  在服务器的 `res.setHeader` 前面或后面添加以下代码，然后重启服务器。这是随便拦截的一个签名，格式一定是正确的。
    
    `res.setHeader('Signature', 'SignedHeaders=content-type, Signature=AAAAWzBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABNsAzGwa7Q3iTZFqv3xYHemw/qxkwk0sIC/usJVi7713VJv0B1JbfuiDXxHfScNyyQjkuaHKtwbn5qUeHjFwpGbEwT7g2t3hdiBTpJ+406wmIST7bK+eY/HU283penaNN9dDWv/ndsvDHCEcckxvSb7XwFBcdy0/Nq3RC9FKAPug')`
    
6.  重新登录，触发了异常断点显示 “响应数据被篡改”。
    
    ![](https://attach.52pojie.cn/forum/202309/24/183515kmjwtq9fmkft6tfx.png)
    
    **4-3.png** _(32.65 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTMwN3xmM2YxOTAwZXwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/5/4-3
    
    2023-9-24 18:35 上传
    
7.  去调用堆栈的上一个看看，发现外面有个 if 分支，看名字以及异常内容应该就是计算响应体是否篡改的。
    
    ![](https://attach.52pojie.cn/forum/202309/24/183536lgo2zislg52eylm3.png)
    
    **4-4.png** _(46.64 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTMwOHw1NWRiYzg3M3wxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/5/4-4
    
    2023-9-24 18:35 上传
    
8.  点进去看看，可以看到返回的是 `ecdsa.VerifyData` 的结果，那直接返回 1 就得了，不管它的计算结果。
    
    ![](https://attach.52pojie.cn/forum/202309/24/183544t08hzw8bpj0bqyhr.png)
    
    **4-5.png** _(24.16 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTMwOXxjY2RjMWE4ZHwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/5/4-5
    
    2023-9-24 18:35 上传
    
9.  返回 1，就把 `0x11 0x03` 修改成 `0x17 0x00` (step1, FiddlerBackendSDK.dll)
    
    ![](https://attach.52pojie.cn/forum/202309/24/183611lfclrzmb7bfvfw37.png)
    
    **4-6.png** _(13.52 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjY0NTMxMHxlMjBjNWJkY3wxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)
    
    pic/5/4-6
    
    2023-9-24 18:36 上传
    
10.  重启 electron，不出意外，报错了。
    
    ```
    [NETCore] Error checking offline license: System.TypeInitializationException: The type initializer for '<Module>' threw an exception.
    ---> System.Exception: FiddlerBackendSDK is tampered.
      at ivwASk28iRuKml2oWx.cFY8uLIGhyRee236nI.OJS7J8FnH()
      at .cctor()
      --- End of inner exception stack trace ---
      at Fiddler.WebUi.Services.Backend.BackendService.GetOfflineLicense()
      at System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start[TStateMachine](TStateMachine& stateMachine)
      at Fiddler.WebUi.Services.Backend.BackendService.GetOfflineLicense()
      at Fiddler.WebUi.Hubs.FiddlerHub.CheckOfflineLicense()
    
    ```
    

5.5 移除 dll 中的自身验证
-----------------

打开 `ivwASk28iRuKml2oWx.cFY8uLIGhyRee236nI.OJS7J8FnH`，粗略查看代码，也是一堆 Hash 计算验证。

找到关键点：

![](https://attach.52pojie.cn/forum/202309/24/183628bu8qutwu77bu8ah7.png)

**5-1.png** _(10.8 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMxMXwzZmQ4MTg4MXwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/5/5-1

2023-9-24 18:36 上传

故技重施，把 0 改成 2：(step2, FiddlerBackendSDK.dll)

![](https://attach.52pojie.cn/forum/202309/24/183646nmt29oz921awocg3.png)

**5-2.png** _(16.28 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMxMnwwODBmOTQ1OHwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/5/5-2

2023-9-24 18:36 上传

重启 electron，ok 没有严重报错，剩下就是补全响应体了。

5.5 响应体数据获取
-----------

这个你要一个个复现是很麻烦的，我是注册了一个新账户；

然后把请求记录全记录下来，塞在服务器里面，然后只要修改以下过期时间就能用了。

很 OK 啊。

![](https://attach.52pojie.cn/forum/202309/24/183706oou55aygfffm233f.png)

**6-1.png** _(242.69 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY0NTMxM3xlYjVmMmM4MnwxNjk1NzA5MTg0fDIxMzQzMXwxODM3OTAy&nothumb=yes)

pic/5/6-1

2023-9-24 18:37 上传

6 最后
====

可以把服务器代码嵌在 main.js 开头，这样开启软件的时候就会自动启动欺骗服务器。

部分相关文件已上传至 github: [https://github.com/msojocs/fiddler-everywhere-enhance](https://github.com/msojocs/fiddler-everywhere-enhance)

还有部分请自行修改。

![](https://avatar.52pojie.cn/data/avatar/000/07/12/20_avatar_middle.jpg)tanzhiwei 楼主应该是搜索. BrowserWindow(  
然后在他后面加上当前这个对象. webContents.openDevTools()  
比如：  
this.noProxyWin || (this.noProxyWin = new mn.BrowserWindow({  
        height: 400,  
        show: !1,  
        width: 400  
}), await this.noProxyWin.webContents.session.setProxy({  
        proxyRules: null,  
        mode: "direct",  
        pacScript: null,  
        proxyBypassRules: null  
}))  
// 加上下面这句  
noProxyWin.webContents.openDevTools();,![](https://avatar.52pojie.cn/images/noavatar_middle.gif)MegatronKing

> [xiaoniu88 发表于 2023-9-25 21:47](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48086990&ptid=1837902)  
> ui 有点小问题， 工具哪里的打开窗口会闪白框

flutter 官方还不支持多窗口模式，使用的第三方库实现，问题就是打开慢一点，背景闪一下。闪一下的问题可以解决，但是也麻烦，我目前没精力处理这个边角问题了，马上要做移动端 app，还有一大堆的需求和用户反馈，哎，这个只好放一放了，先将就下吧。![](https://avatar.52pojie.cn/data/avatar/000/19/82/97_avatar_middle.jpg)szwangbin001 这个是不是一定要登录才能用的，内网登录不了账号。![](https://avatar.52pojie.cn/data/avatar/000/42/71/99_avatar_middle.jpg)cqc520

> [szwangbin001 发表于 2023-9-24 19:20](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48076618&ptid=1837902)  
> 这个是不是一定要登录才能用的，内网登录不了账号。

公司就去买正版。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)DaveBoy 我现在还用的 4.0.1，主要我不知道 github 上 4.6.2 那几个要怎么用，要说直接覆盖的话，step1、step2 这种文件看着又像是逆向过程的中间文件。不像 401windos 那几个文件直接一覆盖，改下 main 的去更新就行了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) qqycra 这个最好用了，就是 x 很麻烦，还要求登录。我都不用它了，用 fiddler classic 和 花瓶。![](https://avatar.52pojie.cn/data/avatar/000/42/71/99_avatar_middle.jpg)cqc520

> [qqycra 发表于 2023-9-24 20:59](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48077275&ptid=1837902)  
> 这个最好用了，就是 x 很麻烦，还要求登录。我都不用它了，用 fiddler classic 和 花瓶。

现在伪造服务器响应数据就是做了本地化；  
登录那一步也能做本地化，但是对我来说可有可无，文章里面就没弄。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)qqycra

> [cqc520 发表于 2023-9-24 21:53](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48077565&ptid=1837902)  
> 现在伪造服务器响应数据就是做了本地化；  
> 登录那一步也能做本地化，但是对我来说可有可无，文章里面就没 ...

步骤多了复杂了，把想白嫖的都吓走了。大佬你文章适合想学习的人。![](https://avatar.52pojie.cn/data/avatar/000/72/12/65_avatar_middle.jpg)无闻无问 厉害啊，大佬，谢谢分享![](https://avatar.52pojie.cn/data/avatar/000/23/12/19_avatar_middle.jpg)孤狼微博 谢谢大佬分享, 越来越麻烦觉得我该放弃了 ![](https://avatar.52pojie.cn/data/avatar/002/01/49/93_avatar_middle.jpg) moruye 厉害厉害，刚好捣鼓捣鼓