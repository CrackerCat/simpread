> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1452575-1-1.html)

> 参考文档：1、发一个可编译，可替换的 hookport 代码网址：https://bbs.pediy.com/thread-157472.htm2、腾讯管家攻防驱动分析 - TsFltMgr 网址：https:/......

 ![](https://avatar.52pojie.cn/data/avatar/000/25/89/14_avatar_middle.jpg) 鱼无论次 _ 本帖最后由 鱼无论次 于 2021-6-3 14:30 编辑_  
**参考文档：**  
**1、发一个可编译，可替换的 hookport 代码  
网址：[https://bbs.pediy.com/thread-157472.htm](https://bbs.pediy.com/thread-157472.htm)  
2、腾讯管家攻防驱动分析 - TsFltMgr  
网址：[https://www.jianshu.com/p/718dd8a1dd27](https://www.jianshu.com/p/718dd8a1dd27)  
3、总结一把，较为精确判断 SCM 加载**  
**网址：[https://bbs.pediy.com/thread-135988.htm](https://bbs.pediy.com/thread-135988.htm)  
**  
**资料代码：**  
**[https://github.com/WINGS2709/360Safe](https://github.com/WINGS2709/360Safe)  
**  
**环境：**  
编译器：VS2013 + WDK8.1  
版本：Win7（32 位）  
中断：sxe ld XXXXX.sys + lmvm XXXXX  
**驱动使用：**  
先加载 HookPort 再加载 SelfProtection  
**前言：**  
1、HookPort 看 achillis 分析即可，新旧版本区别在于新增两个地方：INT 4 中断、过滤函数的优化（感兴趣的单独处理，不感兴趣的通用 Hook 框架）    
2、SelfProtection 实现对应的 Fake 函数模板然后再返回给 HookPort 执行    
3、SelfProtection 绝大部分原理翻看 MJ0011 和 V 校帖子基本有解答  
4、简单介绍下 HookPort 和 SelfProtection 流程，具体实现看附件    
5、有错误欢迎大家告诉我    
6、建议用 OneNote 打开附件    
7、项目仅供参考，代码自己都快忘记了    
8、代码 ShaodwSSDT 部分有 BUG，SSDT 只分析了感兴趣的部分。很多嫌麻烦就没看了    
**原理介绍：**  
1、Hook 前后对比图  
![](https://attach.52pojie.cn/forum/202106/03/105821j813kyyio2c8aci6.png)

**1.png** _(74.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODk2NXw0NjMxNmEwYnwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 10:58 上传

  
2、HookPort 的工作流程  
HookPort 负责构造一份空白的 Hook 模板（不负责编写对应的 Fake 函数，导出给 SalfProtectionX 使用）  
可以理解为老板（HookPort）小弟（SelfProtectionX）  
HookPort 负责循环执行 SalfProtectionX 返回的 Fake 函数  
![](https://attach.52pojie.cn/forum/202106/03/105854txd89dxnp2p9inrx.png)

**2.png** _(26.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODk2NnxjMDNhYWRjZXwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 10:58 上传

  
理论上我们是可以有无数个 SelfProtectionX，但是大数字最大限制 16 个  
Hook 模板结构如下（单向链表结构）：  
![](https://attach.52pojie.cn/forum/202106/03/105923cbp5pxq51q6l1a66.png)

**3.png** _(24.17 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODk2OHw3ZTZiMDZhNHwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 10:59 上传

  
举个例子：  
假设我们一共有 SelfProtection1、SelfProtection2 两个驱动设置了对应的 Fake_CraeteProcess 函数  
![](https://attach.52pojie.cn/forum/202106/03/105951kit5sz5tz5us5d1y.png)

**4.png** _(16.76 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODk2OXw0YTUxNWViNHwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 10:59 上传

  
原始 CreateProcess->KiFastCallEntry->Filter_CreateProcess 代 {过}{滤} 理函数 ->HookPort_DoFilter  
循环将链表中所有 Fake 函数取出来并执行，直到链表下一个为零终止  
必须全部所有 Fake 函数合法返回才算正确，其中一个返回错误都算错误  
![](https://attach.52pojie.cn/forum/202106/03/134624jvqkwmuzldg9dbgg.png)

**1.png** _(49.75 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5OTAxOXxjMjE3ZGRlYXwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 13:46 上传

  
**测试例子**：  
例如我们要拦截 SCM 加载（Fake_ZwAlpcSendWaitReceivePort + Fake_ZwLoadDriver）  
Safe_Initialize_SetFilterSwitchFunction：设置对应的 Fake 函数  
Safe_Initialize_SetFilterRule：设置对应的 Fake 函数开关  
![](https://attach.52pojie.cn/forum/202106/03/135648vo7c242kkvvzlpur.png)

**图片. png** _(22.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5OTAyMHxjYjE2ZTg5YXwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 13:56 上传

  
大致原理：  
Fake_ZwAlpcSendWaitReceivePort 获取真实的 PID（这个技术好像是 360 在 2011 年左右提出来的理论）  
![](https://attach.52pojie.cn/forum/202106/03/135900gssoye9qcb74lqss.png)

**图片. png** _(18.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5OTAyMXxmMTc2NDRlMXwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 13:59 上传

  
然后 Fake_ZwLoadDriver 函数打印出来  
![](https://attach.52pojie.cn/forum/202106/03/135946g5k4k4z2znz13ka3.png)

**图片. png** _(24.37 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5OTAyMnxiYTgyOTk5Y3wxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 13:59 上传

  
设置完毕，我们先加载 HookPort 然后加载 SelfProtection，我们随意加载一个驱动。  
![](https://attach.52pojie.cn/forum/202106/03/140619jkkddldxjvrdffdx.png)

**图片. png** _(446.91 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5OTAyNHw2YmRhYTU1ZnwxNjIyNzEwMzIwfDIxMzQzMXwxNDUyNTc1&nothumb=yes)

2021-6-3 14:06 上传![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)efujin 不明觉厉，从头到尾的字我都认识，可是为什么看完啥都不懂![](https://static.52pojie.cn/static/image/smiley/default/8.gif) ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) xiaohong0827 向技术员致敬！![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)Hmily [@鱼无论次](https://www.52pojie.cn/home.php?mod=space&uid=258914) 标题改改？![](https://avatar.52pojie.cn/data/avatar/000/25/89/14_avatar_middle.jpg)

> [Hmily 发表于 2021-6-3 11:28](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38792155&ptid=1452575)  
> @鱼无论次 标题改改？

改好了，要把所有 X60 的代码里的字眼删了吗？![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)鱼无论次 很厉害，赞一个 ![](https://avatar.52pojie.cn/data/avatar/000/93/53/52_avatar_middle.jpg) ZLJ13697750126 不是 360 吗？？？![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)xixicoco

> [鱼无论次 发表于 2021-6-3 11:43](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38792395&ptid=1452575)  
> 改好了，要把所有 X60 的代码里的字眼删了吗？

这个无所谓，360 也没事，主要是完善下标题描述，具体做什么的。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Hmily 这东西感觉好强![](https://static.52pojie.cn/static/image/smiley/laohu/laohu1.gif) ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 还是没有搞懂