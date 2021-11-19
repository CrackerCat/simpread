> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1546336-1-1.html)

> [md]# 一、什么是 PPL？PPL 全称，Protected Process Light ，是从 windows8.1 开始引入的一种安全机制，他能保护关键进程不被恶意代码入侵、篡改、和利用。

![](https://avatar.52pojie.cn/data/avatar/000/19/56/71_avatar_middle.jpg)xjun _ 本帖最后由 xjun 于 2021-11-18 18:16 编辑_  

一、什么是 PPL？
==========

PPL 全称，Protected Process Light ，是从 windows8.1 开始引入的一种安全机制，他能保护关键进程不被恶意代码入侵、篡改、和利用。换句话说就是系统给进程分了个级别，受 PPL 保护的进程权限更高，低级别的不能打开高级别的进程。当进程签名具有（受保护进程轻型验证 (1.3.6.1.4.1.311.10.3.22)），系统则会加载此保护。

![](https://attach.52pojie.cn/forum/202111/17/180345v32p4dwzpyirwpow.jpg)

**1.jpg** _(82.84 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwMXxlZjlkMjMzM3wxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

二、PPL 对我们有什么影响？
===============

PPL 几乎把 windows 系统的关键进程都给保护起来了，你不能像以前那样拿到个 DEBUG 权限令牌就能任意的读取进程虚拟内存、远程注入代码、结束进程、调试、拷贝描述符、复制句柄、等操作。像我们常用的 mimikatz 它从 lsass.exe 读不到 windows 的凭据了，无法从 csrss.exe 复制句柄去干其他事情了，那么我们在渗透测试、红蓝对抗等业务场景中常常受阻。

注：默认 lsass 是没有开启 PPL 保护的，需要手动开启，也不知道微软为啥要这么做。

![](https://attach.52pojie.cn/forum/202111/17/180347ymwu3i83vii3im43.png)

**2.png** _(22.34 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwMnwwNGQyZGY3OXwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

三、procexp 驱动分析
==============

因为从事安全相关工作，经常使用到 procexp 这个软件，它能够读取所有服务进程包括带 PPL 进程的信息，故对此好奇，便分析了一波。

通过分析得出，它调用了驱动的一个打开进程 API 功能，如下图

![](https://attach.52pojie.cn/forum/202111/17/180349dlm7sn29q1maz1sg.png)

**3.png** _(71.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwM3xiZjFiNmI1N3wxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

为啥驱动调用 ZwOpenProcess 就能绕过这么多限制，拿到权限，这跟 ring3 层直接调用 ZwOpenProcess 调用过来的有何区别？

因为 ZwOpenProcess 在驱动层它会把 PreviousMode 设置成了 KernelMode，而 ring3 调 ZwOpenProcess 通过 syscall 来到内核的 PreviousMode 还是 UserMode。

![](https://attach.52pojie.cn/forum/202111/17/180351vjhi1124qifd7f1f.jpg)

**4.jpg** _(22.94 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwNHxhOWE3MmZiY3wxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

![](https://attach.52pojie.cn/forum/202111/17/180353mxqjovk1ktprvtk1.jpg)

**5.jpg** _(16.59 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwNXw0MjdlMTMyNXwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

四、突破 PPL 限制代码利用
===============

OK，原理一切清楚之后我们来写利用，从 procexp 创建的设备权限来看，我们进程还必须是管理员权限才能打开此设备，这个时候加个 bypass UAC 就能一套带走了，嘻... 嘻嘻嘻！！！

![](https://attach.52pojie.cn/forum/202111/17/180355lz1sgwgw2x2x2xd1.jpg)

**6.jpg** _(12.17 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwNnw5MzA4NjRjMnwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

首先定义 设备名和控制码

![](https://attach.52pojie.cn/forum/202111/17/180357w56hz4rk3hksf3ks.jpg)

**7.jpg** _(20.37 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwN3w0Y2ZiNzQ5NHwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

实现代码

![](https://attach.52pojie.cn/forum/202111/17/180359j9ns5x7zmnnssanq.jpg)

**8.jpg** _(39.95 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwOHxmNDI0YWJkNnwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:03 上传

测试代码和结果：

![](https://attach.52pojie.cn/forum/202111/17/180401ij6dgfh6zdjthtzz.jpg)

**9.jpg** _(117.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjM0ODUwOXwxM2NiNmM3YXwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D&nothumb=yes)

2021-11-17 18:04 上传

参考：

[https://www.crowdstrike.com/blog/evolution-protected-processes-part-1-pass-hash-mitigations-windows-81/](https://www.crowdstrike.com/blog/evolution-protected-processes-part-1-pass-hash-mitigations-windows-81/)

[https://support.kaspersky.com/13905](https://support.kaspersky.com/13905)

[https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection](https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection)

  
  
 ![](https://static.52pojie.cn/static/image/filetype/zip.gif) [poc.zip](forum.php?mod=attachment&aid=MjM0ODUxMHxkNDY5NmRhZXwxNjM3MzI1NzcyfDB8MTU0NjMzNg%3D%3D) _(130.76 KB, 下载次数: 22)_ 2021-11-18 18:16 上传 点击文件名下载附件  
下载积分: 吾爱币 -1 CB  
测试方法：  
1. 手动加载 procexp 驱动 (带微软签名)  
2. 编译 poc 程序，默认打开 csrss.exe, 注意 64 位驱动编译 64 位程序  
3. 使用 bypass uac 程序启动它。![](https://avatar.52pojie.cn/data/avatar/001/11/65/78_avatar_middle.jpg)Dyingchen 看不懂 根本看不懂 不过还是得说一句 xjun 师傅 nb![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)jjr12138 xjun 师傅 new bee![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)mofa005 写的挺好的，就是我看不懂。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)yushui69 受益匪浅，感谢分享 ![](https://avatar.52pojie.cn/data/avatar/000/99/27/39_avatar_middle.jpg) pizazzboy 还是老大 NB，给你点赞！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)hongshen233 看不懂 根本看不懂 不过还是得说一句 xjun 师傅 nb。加油。我会一直支持你 ![](https://avatar.52pojie.cn/data/avatar/000/80/17/56_avatar_middle.jpg) ytfrdfiw 感谢分享。