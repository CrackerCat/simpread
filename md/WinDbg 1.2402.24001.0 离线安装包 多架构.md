> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1949311-1-1.html)

> [md]# WinDbg 1.2402.24001.0 离线安装包 多架构![WinDbg1.png](https://s2.loli.net/2024/07/29/eUPIXGo6KL4BnCJ.pn......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)skrets

WinDbg 1.2402.24001.0 离线安装包 多架构
===============================

![](https://s2.loli.net/2024/07/29/eUPIXGo6KL4BnCJ.png)  
![](https://s2.loli.net/2024/07/29/Brz7u92cdteiW3V.png)

目前 (**2024/7/29**) 的最新版，从商店里抓出来的**原版**安装包

为 `msix` 格式，可以直接**离线**用 Microsoft Store 安装，也可以直接把后缀改成 `zip` 解压即可运行

分为 `x64`, `x86`, `arm64` 三个架构，需要自取

下载链接
----

123pan: [https://www.123pan.com/s/SQKtVv-x8R53.html](https://www.123pan.com/s/SQKtVv-x8R53.html)

baidu: [https://pan.baidu.com/s/1aePD4VeGj1OMkmg0h4s-lw?pwd=1111](https://pan.baidu.com/s/1aePD4VeGj1OMkmg0h4s-lw?pwd=1111)

祝使用愉快～😀

![](https://avatar.52pojie.cn/data/avatar/000/43/30/91_avatar_middle.jpg)scz

> [Kement 发表于 2024-7-30 10:56](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50949436&ptid=1949311)  
> 这个可以调试. Net 吗？

我给个官网下载链接

```
1.2402.24001.0
https://windbg.download.prss.microsoft.com/dbazure/prod/1-2402-24001-0/windbg.msixbundle

```

资源管理器中右键 windbg.msixbundle，有个安装，点击安装即可。但这个操作实际依赖 "App Installer"，没有 Microsoft Store 的 Windows 此法不通。可用 PowerShell 安装、卸载:

```
Add-AppxPackage -Path "<path>\windbg.msixbundle"

```

更进一步，很多用 windbgx 的都有 Portable 的需求，安装后再复制太 low 了，事实上 7-Zip 直接析取，释放到任意位置，就是 Portable windbgx，可用于无 Store 的 Windows。

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 这个可以调试. Net 吗？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Kement 很及时，正在查系统蓝屏的原因，百度说可以用 windbg，结果这就来了 ![](https://avatar.52pojie.cn/data/avatar/001/90/55/78_avatar_middle.jpg) windindind

> [windindind 发表于 2024-7-30 17:23](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50953884&ptid=1949311)  
> 很及时，正在查系统蓝屏的原因，百度说可以用 windbg，结果这就来了

还有创建内存转储文件也可以用这个软件重现 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) collinchen1218 不错  很强大的工具![](https://avatar.52pojie.cn/images/noavatar_middle.gif)justwz 自从用了 x32dbg，基本上就没用这个软件了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) SoftCracker 好东西点个赞哦 ![](https://avatar.52pojie.cn/data/avatar/001/62/02/67_avatar_middle.jpg) by3721 感谢，直接安装成功