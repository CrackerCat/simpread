> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280626.htm)

> [原创] 对 explorer.exe 检测 360 安全卫士的分析

[原创] 对 explorer.exe 检测 360 安全卫士的分析

2 小时前 256

[举报](javascript:void(0);)

### [原创] 对 explorer.exe 检测 360 安全卫士的分析

 [![](http://passport.kanxue.com/upload/avatar/739/978739.png?1708658064)](user-home-978739.htm) [lidowx](user-home-978739.htm) ![](https://bbs.kanxue.com/view/img/rank/5.png) 1  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 2 小时前  256

背景:
---

微软最近的 KB5034203 补丁包中 explorer.exe 包含这样一段代码, 这两天传来传去, 各种说法都有, 我也很好奇, 但是一直没等到具体的技术细节分析, 自己分析一下.

![](https://bbs.kanxue.com/upload/attach/202402/978739_C2MJ44RVBC2JCZY.png)

正文
--

**explorer.exe 启动时会加载 twinui.pcshell.dll 并执行其中的`ShellFeedsAvailability::WriteFeedsAvailabilityToRegistry`, 这个函数很重要, 如果它没有执行, 那么 IsHijackingProcessRunning() 那些代码都不会执行到:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_PEE4NEBF73S6TZG.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_NFD3WTD4TF8M2NP.png)

**此函数内部会调用`ShellFeedsAvailability::CheckFeedsServerAvailability()`从微软服务器下载 json 配置文件并决定相关功能开关. 首先组装配置文件 URL:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_DPJQQ6KVZJ5NWWC.png)

**URL:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_G566VTRYJ58BRB2.png)

**随后 PingFeedsEnablementEndpoint() 函数用这个 URL 请求 json 配置:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_CWED9BADSGB5S9Q.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_VBRUY3R9E5SAQH6.png)

**获取到的 json 内容:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_B9C2XNHYQ33AQSE.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_J4XG6ZVNZ843H5S.png)

**再将获取到的 json 传递到 IsFeedsEnabledAtEndPoint, 这个函数会解析 json 配置文件的一个重要字段`reclaimEnabled`, 如果为 false, 那么后面检测 360 的那段逻辑也不会执行, 而如果是 true, 则将注册表 Campaign 设置为 "feeds-reclaim":**

![](https://bbs.kanxue.com/upload/attach/202402/978739_KVZS3NUQYVT4ZAE.png)

**然后返回到最初的 WriteFeedsAvailabilityToRegistry, 将注册表 CampaignState 写为 2, 再给 Shell_TrayWnd 发一个自定义消息, WParam 为 12:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_3E82YRT7Z3NKGHX.png)

**这个消息会在 explorer.exe 的`Feeds::FeedsDynamicContent::ProcessMessage`中处理, 主要是启动一个 ID 为 6 的 timer, 默认 300 秒 (或 10 秒后) 运行, 运行一次后就 killtimer:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_NMH6YP8FKG43VUH.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_RRWH8ESU9KMNG7S.png)

**`Feeds::FeedsDynamicContent::OnTimer`中判断 TimerID 为 6 且 reclaim feature 开启就会调用 ShellFeedsCampaignHelper::RunFeedsCampaign::**

![](https://bbs.kanxue.com/upload/attach/202402/978739_RQUS9MCJWXR7X4T.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_CG7J6M8AD3WCKWX.png)

**RunFeedsCampaign 会检查前面那两个注册表键值, 如果设置了并且值符合要求, 就会执行`ShellFeedsCampaignHelper::CheckCampaignAvailability`:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_HVCER7MVGH2KZHJ.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_5E5MBXQDZA46ZP8.png)

**CheckCampaignAvailability 函数里一系列环境判断, 但会优先查看 feeds 是否被隐藏, 如果没有被隐藏, 后续的检测, 包括对 360 进程的检测都不会执行:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_YBPX2MWU6Q2HM8R.png)

**同时这个函数里也有一些其他判断, 比如:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_HPZN8W4CJMHEH3G.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_JSGEWVWM5BZ8N5F.png)

**如果所有条件都成立, 则执行 CampaignAction_Reclaim:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_PBRQMNY2TQBWPPH.png)

**调试前我就已经纯手动关了 feeds(任务栏右键 -> 资讯 -> 关闭), 函数逻辑及执行前任务栏右下角状态:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_MKY6K24W7WV68X3.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_C9YQC9QVP34V7AD.png)

**CampaignAction_Reclaim 执行后:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_DHQTWNCKAVDVTXM.png)

**也就是说 CampaignAction_Reclaim 函数主要就是把被隐藏的 feeds 组件重新展示出来 (当然前提是满足一系列条件).**

**此外 explorer.exe 中新增了一些代码, 会将 feeds 组件的某些注册表参数值做 "加密"(这个也是由 feature 开关决定是否要执行):**

![](https://bbs.kanxue.com/upload/attach/202402/978739_YQHBAE343TE3SMQ.png)

**"加密" 相关逻辑:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_WBCZQM6R75J9Y3Y.png)

**导入的 HashData 函数的具体哈希逻辑没兴趣看:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_9MFNPTR7HKR5ZFH.png)

* * *

**整体就是判断一系列条件是否全满足 (其中一个必要条件是存在 360 进程), 且 feeds 组件是隐藏状态且超过一定时间, 那么就再次给它展示出来. (据说 360 并不会主动隐藏微软的 feeds 组件, 是 360 系统优化中的其中一个功能, 需要用户手动操作才会隐藏 feeds, 这个我没实际验证)**

* * *

仅客观技术分析, 以上都是静态代码逻辑能明确看到, 并且在调试器实际验证观察过的 (除了注册表值加密那部分, 那部分我不关心, 所以并没有让那段代码跑起来实际观察验证, 加密逻辑很简单, 静态看就差不多了).
--------------------------------------------------------------------------------------------------------

**至于网上所说的是微软为了避免跟 360 冲突崩溃 / 主动隐藏自身广告 / 兼容 360 搜索窗口错位, 暂时没有看到明确相关的逻辑 (但也不是说完全不可能, 也许只是我暂时没看到, 因为时间有限, 要对两个版本的多个二进制文件所有差异做 100% 全覆盖分析太费事了, 我主要想弄明白的主要还是 explorer.exe"检测到 360 后到底会做什么".)**

* * *

调试环境
----

**原始镜像:**

zh-cn_windows_10_business_editions_version_22h2_updated_oct_2023_x64_dvd_eb713365.iso

**补丁包:**

windows10.0-kb5034203-x64_14f2cba156944cea66379d78c305f5aa5a6517e7.msu

(这个补丁包里 explorer 等组件版本大多是 3996, 这是目前在微软官方网站能找到的首个包含 IsHijackingProcessRunning 的版本)

**注:**

上面提到的 feature 开关 (带 feature_xxxx_private_IsEnabled 的), 和 json 里的那个字段是独立的, 如果想调试复现的话, 要注意把相关 feature 开关也打开 (调试器里把相关函数头改为 mov eax,1; ret).

* * *

**另外借 "操作系统检测三方软件" 这个话题, 再额外发一段 windows 内核中的逻辑:**

![](https://bbs.kanxue.com/upload/attach/202402/978739_8UHEBVX52WU83EC.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_QXQFVWNZARFAFZ2.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_F3AWWNFC34JKEZ5.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_NE572BCCD98UJR9.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_PBE6VYK7XWVD48Q.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_6SZA9EYPG9HSU9S.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_VSH3HRAMKNW88A5.png)

![](https://bbs.kanxue.com/upload/attach/202402/978739_BMYVZMJRRAHACHY.png)

这是几年前调一个驱动问题的时候无意中发现的, 在最新 Win11 上依旧存在, 但这个我也没有实际安装卡巴斯基调试验证, 但是有点技术背景的应该都能看出个大概: 微软 hook 了卡巴斯基驱动的 ZwQueryValueKey, 让卡巴斯基读取 UseVTHardware 的时候返回 0. 至于目的, 可能是因为卡巴斯基要用 VT, win 自己也要用 VT, 或者卡巴斯基的 VT 开了之后有其他问题. 具体只有当时写代码的人或者看原始 commit 信息才知道了.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

发卡巴斯基这段并不是为了吐槽微软, 只是恰巧想起来了而已, 实际我是理解这种特殊处理的.
--------------------------------------------

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

最后于 2 小时前 被 lidowx 编辑 ，原因： 补充 [#调试逆向](forum-4-1-1.htm)