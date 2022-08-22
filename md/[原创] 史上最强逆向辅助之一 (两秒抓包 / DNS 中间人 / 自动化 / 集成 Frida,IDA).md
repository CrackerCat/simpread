> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273629.htm)

> [原创] 史上最强逆向辅助之一 (两秒抓包 / DNS 中间人 / 自动化 / 集成 Frida,IDA)

> 这是一个用于安卓逆向及自动化 (群控) 的辅助框架，它以编程化的接口大幅减少你的手动操作。

*   还在用 adb shell input am pm.. ?
*   还在每次都要 /data/local/tmp/frida-server & ?
*   还在安装 `ProxyDroid`、`httpcanary` 甚至更加底层的抓包方法吗？
*   还在为抓不到包发愁吗？
*   又或，还在用 adb 吗？
*   想永久脱离 usb 吗？

> 本框架的目标就是让你以最简单的方式完成耗时或者需要处理各种异常情况的任务，除了 `sslpining`，让你没有抓不到的包，零依赖无需 `xposed` 等类似框架，仅需 `root` 即可，而且，它足够稳定 (7*24)。绝不是写着玩玩的质量。

远程控制界面
------

![](https://bbs.pediy.com/upload/attach/202207/762254_22GUMGU2TYGS4A6.gif)

快速进行中间人
-------

![](https://bbs.pediy.com/upload/attach/202207/762254_24CG8DDV2VV8Z5Y.gif)

[](#当然，一键给国外app来个中间人也行)当然，一键给国外 APP 来个中间人也行
-------------------------------------------

![](https://bbs.pediy.com/upload/attach/202207/762254_T7DM2R3VW3F9TTG.gif)

自动化操作
-----

![](https://bbs.pediy.com/upload/attach/202207/762254_VPK8BBUDF6PRFFZ.gif)

DNS 中间人
-------

![](https://bbs.pediy.com/upload/attach/202207/762254_PQDFKX6VB6B2872.png)

> 因为客户端库整体使用 Python 编写，在继续之前，你可能需要掌握 《Python 编程 从入门到实践》 前十章的内容。但是如果仅仅使用中间人，那么无所谓。

*   第 2 章　变量和简单数据类型
*   第 3 章　列表简介
*   第 4 章　操作列表
*   第 5 章　if 语句
*   第 6 章　字典
*   第 7 章　用户输入和 while 循环
*   第 8 章　函数
*   第 9 章　类
*   第 10 章　文件和异常

除此之外，你可能还需要略微了解 shell 命令及脚本，熟悉 adb。

 

**项目地址：**[github.com/rev1si0n/lamda](https://github.com/rev1si0n/lamda)

 

请注意积极更新，这目前并不是接口稳定的框架，不时会添加删除接口。  
这不是你所见过的自动化软件，它没有那种华丽的中心后台界面，它只提供代码接口给你。

[](#举个熟悉的例子：)举个熟悉的例子：
---------------------

当你还在 adb push 证书到设备，还要在 ProxyDroid 或者系统设置里设置代理，然后还要担心遇到某种：应用可以选择不使用系统代理，charles fiddler 配置不熟悉 的情况时。

 

你可能需要花费 2 分钟来完成各项设置，而使用它可以在 2 秒内完成：

```
python3 startmitm.py 192.168[设备IP]
```

仅仅一条命令而已就为你完成了所有操作。

 

如果你有其他需求，例如在代码里动态设置（你可以查看 startmitm.py 的源码）

```
profile = GproxyProfile()
profile.type = GproxyType.HTTP_CONNECT
 
profile.host = "代理服务器地址"
profile.port = 代理服务器端口
# 启动代理
d.start_gproxy(profile)
```

不止这对你来说是否足够简单？当然，它不止这么多配置，你可以查看文档了解。  
这一切仅仅需要你在已 root 的设备上运行服务端而已。

手机不在身边，但是想调试 / 控制怎么办
--------------------

你只需要有一台公网服务器，配置好转发服务后，在启动时编写如下配置放到指定目录（具体请看文档）。

```
fwd.host=服务器地址
fwd.port=6009
fwd.rport=2022
fwd.token=yespassword
fwd.protocol=tcp
fwd.enable=true
```

好了，现在你可以在服务器上愉快的调试或者控制世界各地只要有网的手机了！

它绝不止上面描述的这些能力
-------------

*   支持安卓 6.0 - 12。
*   支持通信加密
*   支持标准游戏模拟器，AVD 及真机，(arm/arm64/x86/x86_64）全架构
*   内置 OpenVPN 可实现全局 / 非全局的 vpn
*   内置 http/socks5 代理，可实现设置系统级别 / 单个应用级别的代理
*   支持 OpenVPN 与代理共存
*   可通过证书接口轻松设置系统证书，配合 http/socks5 代理实现 mitm 中间人
*   UI 自动化，通过接口轻松实现自动化操作
*   设备状态 / 资源消耗读取
*   大文件上传下载
*   接口锁
*   唤起 APP 任意 Activity
*   前后台运行脚本，授予撤销 APP 权限等
*   系统属性 / 设置读取修改
*   WIFI 相关功能
*   滑动轨迹录制重放功能
*   内置 frida, IDA 7.5 server 等工具
*   类 sekiro 的内部 API 接口暴露功能
*   可使用 ssh 登录设备
*   支持 crontab 定时任务
*   内置 Python3.9 及部分常用模块
*   内网穿透，简单配置加上公网服务器即可控制任何地方的设备
*   远程控制（web）
*   远程调试

> 完整的功能及接口文档介绍请查看 [github.com/rev1si0n/lamda](https://github.com/rev1si0n/lamda) 后期可能会在 bili 发布完整功能的使用教程及视频并在此更新章节。

**有使用问题可以加群。U1TR0N**
--------------------

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

最后于 2022-7-27 00:22 被 RiDiN 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)