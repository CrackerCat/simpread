> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271082.htm)

> [原创]Android 风险环境检测

本文中提到的检测方式，都是笔者自己总结，现有公开资料都未有涉及，对于已有或者已知的检测方式，不在本文讨论范围内。

双开和多开检测
=======

背景
--

Android 5.0 以上强制开启 SELinux，每个 app 都会分配 SELinux domain，三方 app 都是 untrusted_app，但随着版本升高，untrusted_app 也会在后面加版本。  
如 untrusted_app_25、untrusted_app_27 等。

检测思路
----

使用 SELinux domain 进行检测，若 app 运行在双开 app 的环境下，domain 使用的是双开 app 的，而该 app 实际的 domain 和双开不一致则可检测出来。需要满足下面条件，不能 100% 检测，但若是检测出，则一样是有问题的

前提条件
----

需要双开或多开软件和检测 app 的 targetsdkversion 不一样，并且在当前手机系统上 targetsdkversion 生成的 domain 不一样才可以。

技术原理
----

本文不详细介绍 domain 的生成过程，只说下大概思路：

1.  首先根据签名信息和系统中的 mac_permissions.xml 文件比较，获取 seinfo 信息，三方 app 的 seinfo 格式为：default:targetSdkVersion=XX, 代码在 SELinuxMMAC.java
2.  然后通过 seinfo 和 seapp_contexts 中的配置信息匹配，计算出 domain。

如下面的是 Android12 关于三方 app 的配置，只关注 minTargetSdkVersion 和 domain 字段，若当前 app targetsdkversion >= minTargetSdkVersion，会分配后面的 domain，minTargetSdkVersion 顺序从高到低匹配。

 

若当前 app 的 targetsdkversion 是 30，匹配到第一条规则，则 domain 就是 untrusted_app

 

若 targetsdkversion 是 27，会匹配到最后一条，则 domain 是 untrusted_app_27

```
... ...
user=_app minTargetSdkVersion=30 domain=untrusted_app type=app_data_file levelFrom=all
user=_app minTargetSdkVersion=29 domain=untrusted_app_29 type=app_data_file levelFrom=all
user=_app minTargetSdkVersion=28 domain=untrusted_app_27 type=app_data_file levelFrom=all
user=_app minTargetSdkVersion=26 domain=untrusted_app_27 type=app_data_file levelFrom=user
... ...

```

### 不同 Android 版本和 minTargetSdkVersion 生成 domain 的表格

<table><thead><tr><th></th><th>8.0(26)</th><th>8.1(27)</th><th>9(28)</th><th>10(29)</th><th>11(30)</th><th>12(31)</th></tr></thead><tbody><tr><td>&gt;=30</td><td></td><td></td><td></td><td></td><td>untrusted_app</td><td>untrusted_app</td></tr><tr><td>&gt;=29</td><td></td><td></td><td></td><td>untrusted_app</td><td>untrusted_app_29</td><td>untrusted_app_29</td></tr><tr><td>&gt;=28</td><td></td><td></td><td>untrusted_app</td><td>untrusted_app_27</td><td>untrusted_app_27</td><td>untrusted_app_27</td></tr><tr><td>&gt;=26</td><td>untrusted_app</td><td>untrusted_app</td><td>untrusted_app_27</td><td>untrusted_app_27</td><td>untrusted_app_27</td><td>untrusted_app_27</td></tr><tr><td>&lt; 26</td><td>untrusted_app_25</td><td>untrusted_app_25</td><td>untrusted_app_25</td><td>untrusted_app_25</td><td>untrusted_app_25</td><td>untrusted_app_25</td></tr></tbody></table>

 

横坐标为 Android 版本，纵坐标为 minTargetSdkVersion，中间即为系统生成 domain 的值

测试
--

在 Android11 的手机上使用 双开助手 - 微多开分身 这款双开 app 测试，该 app 的 targetsdkversion 为 26，检测 app 的 targetsdkversion 为 31，根据上面表格可知，双开 app 的 domain 是 untrusted_app_27，而检测 app 的 domain 为 untrusted_app，domain 不一致则检测出有使用双开，如下图：

 

![](https://bbs.pediy.com/upload/attach/202201/403246_TRADMQMAXKC2TM2.png)

EdXposed 和 LSPosed 检测
=====================

Xposed 最高支持到 Androd8，若要使用 Xposed 框架，就需要使用 EdXposed 和 LSPosed，这两个框架需要搭配 magisk 来使用，并且需要使用 magisk 插件 [Riru](https://github.com/RikkaApps/Riru)。

检测思路
----

该类 hook 框架都需要注入到目标进程中，Riru 提供了一种能力，可以在目标进程中隐藏注入的 so 文件，检测思路就是在隐藏的内存中搜索注入框架的特征。

技术原理
----

Riru 隐藏的代码在 [这里](https://github.com/RikkaApps/Riru/blob/master/riru/src/main/cpp/hide/hide.cpp)，思路：

1.  先备份隐藏的内存，首先申请内存，将要隐藏的内存拷贝到申请的内存里
2.  释放需要隐藏的内存
3.  然后重新申请内存，并且指定的地址为刚才释放的隐藏内存
4.  最后将最开始备份的内存拷贝回去

这样的效果是读取内存列表文件 /proc/[target pid]/maps 时隐藏的内存后面没有文件名，像下面这样，是匿名内存

```
7f063d8000-7f06770000 ---p 00000000 00:00 0
7f06770000-7f06772000 rw-p 00000000 00:00 0
7f06772000-7f073d8000 ---p 00000000 00:00 0
7f073d8000-7f073d9000 ---p 00000000 00:00 0

```

但是上述的隐藏不会修改内存的权限，一般匿名内存都是可读可写，有可执行权限的情况比较少，所以我们只需要对匿名内存中有可执行权限的进行搜索，这样就避免对整个内存进行搜索。

 

还有内存中存在可读、可写和可执行权限的内存段，则这种情况一定是有风险的。特别是在 Android 高版本上。这违背了 W^X 原则，可写和可执行不能同时存在。双开 app 大多数存在可写和可执行的 libc.so 和 linker 等情况。

测试
--

![](https://bbs.pediy.com/upload/attach/202201/403246_AYR7WB2WH4BFTDW.png)

 

通过这种方式，EdXposed 和 LSPosed 都是可以检测出来的。

总结
==

对于双开检测，针对系统和目标 app 都是高版本，则比较有效，一般双开或多开 app 的 targetsdkversion 都较低，若升高可能某些 API 不可用或者适配代价较大。

 

对于 EdXposed 和 LSPosed 目前只是粗略的对相关 so 进行检测，以后看下是否可以直接检测出 hook 的类和方法。

 

最后给出代码： [https://github.com/liwugang/RiskEnvDetection](https://github.com/liwugang/RiskEnvDetection)

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#其他](forum-161-1-129.htm)