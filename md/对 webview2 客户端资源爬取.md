> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1921927-1-1.html)

> 某款软件代码和资源的提取，软件是基于 webview2 的，网上没有看到 webview2 相关逆向的教程，所以发个贴记录一下。

![](https://avatar.52pojie.cn/data/avatar/001/63/83/85_avatar_middle.jpg)少年持剑 _ 本帖最后由 少年持剑 于 2024-5-8 23:58 编辑_  
某款软件代码和资源的提取，软件是基于 webview2 的，网上没有看到 webview2 相关逆向的教程，所以发个贴记录一下。  
**简单分析**  
软件安装后只有一个 exe 文件，软件打开后界面风格很像一个网页  
![](https://attach.52pojie.cn/forum/202405/08/233807ejhw3l7yhgn5ljf7.png)

**}%M`A7[JV6543K){Q5AK8A2.png** _(58.07 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY1OHxiZjA4YTEyOXwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:38 上传

  
X64dbg 附加软件后，发现它加载了一个 edge 的 dll  
![](https://attach.52pojie.cn/forum/202405/08/233843j5k1ki244b5opf5z.png)

**ZPAOP7BMTUBS~L]UOPC~L9W.png** _(87.34 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY1OXw5MmViYWEzMHwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:38 上传

  
搜索后发现是使用 webview2 后会使用的一个 EmbeddedBrowserWebView.dll。  
**WebView2**  
WebView2 控件使用微软的 Edge 作为渲染引擎，WebView2 允许你在本地 App 里面嵌入 web 相关的技术（例如 HTML，CSS 和 JavaScript）。  
既然是 web 就要看能不能打开开发者工具了，进入微软 webview2 开发者文档，可以看到提供了一个 [**OpenDevToolsWindow**](https://learn.microsoft.com/zh-cn/microsoft-edge/webview2/how-to/debug-devtools?tabs=win32cpp) 函数来打开开发者工具  
![](https://attach.52pojie.cn/forum/202405/08/233950xax6ulyvgxgsyfft.png)

**7~8%`)IA`MEC7](5ZML4J{4.png** _(50.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY2MHwyNzA0MWVlMXwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:39 上传

  
[IDA](https://www.52pojie.cn/thread-1874203-1-1.html) 打开 EmbeddedBrowserWebView.dll ，加载符号文件，查看 OpenDevToolsWindow，可以看到它是 embedded_browser_webview_current 下的一个虚函数  
![](https://attach.52pojie.cn/forum/202405/08/234031shzhmml9bkzukq5h.png)

**J@I@UD3QOB{5~BM}HSL2B42.png** _(50.18 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY2MXxjNTUxMGFjMXwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:40 上传

  
查看微软提供的 webview2 的代码示例 [Win32_GettingStarted](https://github.com/MicrosoftEdge/WebView2Samples/blob/main/GettingStartedGuides/Win32_GettingStarted/HelloWebView.cpp)，可以看到它调用了一个 Navigate 函数来访问页面  
![](https://attach.52pojie.cn/forum/202405/08/234210mldff9dlt6dk2l68.png)

**EG)XJ3101Q2LELI(9`(HGIJ.png** _(35.69 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY2M3w5MGVlZTExNnwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

sam

2024-5-8 23:42 上传

  
  
Navigate 函数也是 embedded_browser_webview_current 下的一个虚函数，这里我通过修改虚表中 Navigate 的地址指向,  
EmbeddedBrowserWebView.dll text 段末尾空闲位置调用 OpenDevToolsWindow  
  
![](https://attach.52pojie.cn/forum/202405/08/234251yw8qabiqzziqv1wd.png)

**MOX17G`B0DN]EZZ8PBRP%TN.png** _(27.91 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY2NHw2YTdlNTM4ZnwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:42 上传

  
X64dbg 打上补丁 ，替换 dll  
这时候开发者工具就会弹出来了。  
![](https://attach.52pojie.cn/forum/202405/08/234519hbsn2m4rt2g0nn4r.png)

**U6(DN@L{QX2R8S089G9RK}F.png** _(49.65 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY2NXwzOWIzNTk3ZHwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:45 上传

  
代码和资源提取  
在开发者工具的源代码 tab 导出所有的 js 文件和其它资源文件  
![](https://attach.52pojie.cn/forum/202405/08/234705v1xz2ekkvnkwcvfc.png)

**R(KS@FS4B_[$R@G(_D067SQ.png** _(54.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY2N3xiMDZkYzQ0MHwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:47 上传

  
并记下它的虚拟域名  
**环境搭建**  
在之前的 Win32_GettingStarted gitclone 下来，使用 SetVirtualHostNameToFolderMapping 设置虚拟域名并 Navigate 调用  
![](https://attach.52pojie.cn/forum/202405/08/234906y74ii4lh56cd65lc.png)

**QV4E1P[$~(@QRJIZINCQ6ZO.png** _(67.77 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY3MHw4MGMzYzJhMXwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

2024-5-8 23:49 上传

  
编译运行后开发者工具中会有几个报错。根据工具报错的提示，到正常软件的开发者工具获取正常环境就行了。  
![](https://attach.52pojie.cn/forum/202405/08/235841zvdkkr4gqvz2qgvo.png)

**}))M)GVQKL}QMJ6[}7GL$H8.png** _(194.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY5NDY3MnxjY2VhNmY5ZXwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D&nothumb=yes)

tihuan

2024-5-8 23:58 上传

  
与开发者的工具的本地覆盖相比，提取出代码后 js 更容易修改和稳定，而且 c++ 提供的操作也会更灵活。  
EmbeddedBrowserWebView.dll 文件位于 **C:\Program Files (x86)\Microsoft\EdgeWebView\Application\124.0.2478.80\EBWebView\x64**  目录下，不同版本应该不通用  
  
  
 ![](https://static.52pojie.cn/static/image/filetype/zip.gif) [EmbeddedBrowserWebView.7z](forum.php?mod=attachment&aid=MjY5NDY3MXxjYjk0MTAzOXwxNzE1MjIwMDUxfDB8MTkyMTkyNw%3D%3D) _(1.58 MB, 下载次数: 8, 售价: 2 CB 吾爱币)_ 2024-5-8 23:51 上传 点击文件名下载附件  
修改后  
售价: 2 CB 吾爱币  [[记录]](forum.php?mod=misc&action=viewattachpayments&aid=2694671)  
下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/data/avatar/001/37/21/93_avatar_middle.jpg)我是不会改名的 实际上只需要添加环境变量就行了，官方文档有写的，只不过藏得很深  
set WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS="--auto-open-devtools-for-tabs"  
这里的文档里面有写  
[https://learn.microsoft.com/en-u ... -dotnet-1.0.1774.30](https://learn.microsoft.com/en-us/dotnet/api/microsoft.web.webview2.core.corewebview2environment.createasync?view=webview2-dotnet-1.0.1774.30)  
playwright 里面也有  
[https://playwright.nodejs.cn/docs/webview2#google_vignette](https://playwright.nodejs.cn/docs/webview2#google_vignette)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)qq465881818 我一般抓包![](https://static.52pojie.cn/static/image/smiley/default/lol.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) eric1tian 学到了，赞一个 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) houdongen  
学到了，赞一个 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) nitian0963 学习，感谢楼主 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) uliaxs0n 感谢楼主，学习了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Xieweiping 学到了 楼主辛苦 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) vilenyuu 学到了 辛苦楼主 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) HuskyHappy 学无止境，谢谢楼主