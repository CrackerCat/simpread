> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [weishu.me](https://weishu.me/2022/06/12/eBPF-on-Android/)

> 若干年前，我还在做 Android 客户端性能优化的时候，读到了 OpenResty 作者章亦春老师的文章：动态追踪技术漫谈，当时被深深地震撼到了，原来通过使用各种高级的调试技术，解决问题竟然可以做到如此精准而优雅。

若干年前，我还在做 Android 客户端性能优化的时候，读到了 OpenResty 作者章亦春老师的文章：[动态追踪技术漫谈](https://blog.openresty.com.cn/cn/dynamic-tracing/)，当时被深深地震撼到了，原来通过使用各种高级的调试技术，解决问题竟然可以做到如此精准而优雅。

然而当我真正要解决 Android 系统上应用程序的性能问题时，才发现理想很丰满现实很骨感——手头趁手的工具几乎没有。文章中提到的内核态追踪技术 SystemTap / DTrace 在 Android 系统上压根不存在，用户态的追踪技术开销大到可怕：TraceView 开启后程序性能直接下降十倍不止，Systrace 当时功能半残废，使用起来还需要自己插桩；simpleperf 能使，但就是有点 simple…… 到最后，真正要解决问题的时候，还是靠经验、二分法和 inline hook；为了定位 Android 虚拟机的性能问题，我甚至还自己造了个 [ART HOOK 的轮子](https://github.com/tiann/epic)。

然而，时间来到 2022 年，世界已焕然一新：eBPF 这种革命性的技术改变了一切。

eBPF 是什么？
---------

> eBPF is a revolutionary technology with origins in the Linux kernel that can run sandboxed programs in a privileged context such as the operating system kernel. It is used to safely and efficiently extend the capabilities of the kernel without requiring to change kernel source code or load kernel modules.

简单来说，[eBPF](https://ebpf.io/) 是一个运行在 Linux 内核里面的虚拟机组件，它可以在无需改变内核代码或者加载内核模块的情况下，安全而又高效地拓展内核的功能。

[![](https://blog.dimenspace.com/mweb/16549650615106.jpg)](https://blog.dimenspace.com/mweb/16549650615106.jpg)

eBPF 的前身是 BPF(Berkeley Packet Filter) 技术，它原始的含义是 extended BPF。BPF 是一种古老的技术，最早可以追溯到 1992 年，它是为捕捉和过滤网络数据包而设计的，鼎鼎大名的抓包软件 Wireshark 就是基于它实现的。然而，经过若干年的发展，eBPF 早已脱胎换骨，成为 Linux 内核可观测技术事实上的标准。

在 eBPF 之前，给内核拓展功能非常麻烦。由于内核是硬件资源的管理者，其对安全性和稳定性的要求非常高，所以它的功能迭代周期相比应用程序慢得多。如果想要在内核中添加新功能，要么就直接修改内核代码，要么通过内核模块（LKM）实现。修改源码的话，自己维护成本高，进主干发布周期长，改完替换内核还有风险；用内核模块的话，由于 Linux 内核不提供稳定的 API，其维护成本会很高，内核模块在面临不同的内核版本时会面临巨大的困难。

eBPF 通过在内核中实现一个轻量级虚拟机，可以动态地加载和运行自定义的程序，通过这个自定义的程序可以轻松地拓展内核的功能。虚拟机在运行程序之前会进行校检，可以确保其不会导致 panic（回想一下内核模块，一不小心系统分分钟重启给你看..）另外，在 BTF 的加持下，eBPF 程序可以在跨内核版本上做兼容（内核模块再次泪目…

扯了这么多，有人会说，这玩意不就是一个运行在内核里面的解释器嘛。。就像运行在浏览器里面的 Javascript 引擎一样，有什么好稀罕。这个类比好像没毛病，但由于它运行在内核里，Linux 上独此一家，因此得万千宠爱于一身。因为，Linux 内核它真的太需要一个虚拟机了…

为什么是现在？
-------

eBPF 技术最早在 Linux 3.15 被引进，从时间上来看已经不算是一种新技术了，而且它在服务端的应用已经有很长的历史，为何到现在咱才想起它呢？

这玩意它首先得益于云原生技术的蓬勃发展，不过跟咱要讲的 Android 没啥关系，暂且不表。对 Android 系统来说，GKI(General Kernel Image) 的出现，让 eBPF 登上了历史舞台。

[GKI——通用内核镜像](https://source.android.com/devices/architecture/kernel/generic-kernel-image)，是 Google 为了解决 Android 碎片化而提出的一种技术。

在 GKI 之前，Android 内核的碎片化非常严重，不同的设备制造商、不同的设备型号，甚至不同的设备版本，其运行的内核代码都不一样；这就导致在内核里面添加功能维护成本巨高，进而使得内核升级几无可能（回想一下以前的 Android 设备，其内核版本出厂之后除了安全补丁，基本上被锁死了）

而 eBPF 作为内核的一个功能，它的启用需要依赖特定的内核编译配置，如果某个内核没有开启某编译选项，eBPF 可能就用不了；在 Android 内核碎片化的年代，想要搞到能完整支持 eBPF 的设备，那是挺难的；如果想用 eBPF 这个功能，只有自己去改内核代码然后自己编译，然而你会遇到各种设备驱动问题，比如触屏失灵，Wifi 不工作等等等等。。当你好不容易在自己设备上折腾好，想要在别的设备上运行时，哦豁，它不支持 BTF，你的 eBPF 程序跑不了。。

GKI 通过统一核心内核，把其他功能（如 SoC，ODM 等提供的）从内核剥离并提供稳定接口（KMI），一举解决了碎片化问题。并且，Google 强制要求，Android 12 以上版本的设备，出厂必须使用 GKI 内核。

[![](https://blog.dimenspace.com/mweb/16549602328227.jpg)](https://blog.dimenspace.com/mweb/16549602328227.jpg)

更重要的是，GKI 内核的编译选项，完整支持 eBPF 的几乎所有功能！！也就是说，你拿到一个 GKI 的设备，无需自己编译内核代码，它必然支持 eBPF；你在一个 GKI 设备上编写 的 eBPF 程序，可以轻松地拓展到其他的 GKI 设备！

[![](https://blog.dimenspace.com/mweb/16549649154392.jpg)](https://blog.dimenspace.com/mweb/16549649154392.jpg)

所以现在这个时间点，在 Android 12 已经发布，Android 13 已经 beta 的情况下，是时候开始学习和使用 eBPF 了。

eBPF 能干什么？
----------

长篇大论了这么多，把 eBPF 吹的那么神，这种所谓的革命性的技术能拿来干啥？官方文档这么说：

> eBPF 在现代数据中心和云原生环境中提供高性能的网络和负载均衡，以低开销提取细粒度的安全可观察性数据，帮助应用程序开发人员追踪应用程序，为性能故障排除提供洞察力，预防应用程序和容器运行时的安全执行等等。eBPF 的可能性是无穷的，它所释放的创新才刚刚开始。

那么，在 Android 系统上，它能干什么？按照官方的说法，我们可以拿它来动态追踪应用程序、解决性能问题；可能还不够具体，我举几个例子。

### 系统调用监控

我们知道，应用程序与内核打交道的方式就是系统调用，我们在分析应用程序的时候，监控系统调用是一种非常有效的方式。在 eBPF 之前，我们通常有如下方法：

1.  基于 ptrace 技术，如 strace。ptrace 技术可以在用户空间拦截系统调用，功能非常强大，但它最大的问题是，由于它太强大，很多应用程序会拒绝在 ptrace 环境下运行；一旦你 ptrace 它，它自己就闪退给你看。我们需要用各种技术去绕过这种检测，这些绕过方法基本上不通用，而且成本比较高。
2.  inline hook 技术。这种技术通过 hook libc 中的函数以及在内存中搜索系统调用实现对系统调用的监控；它的问题在于需要注入目标进程，并且搜索内存可能会漏掉某些调用。
3.  seccomp 技术。seccomp 也可以监控系统调用，它的问题在于，它需要在应用程序中注入代码，另外，改变了 seccomp context 后，用户空间会遗留特征，也是会被检测的，虽然目前为止关心它的应用不多。
4.  自己编译内核，或者内核模块。这种方法不易被检测，不过编译内核可能会遇到各种阻碍，比如内核源码没公开，或者虽然公开了但不是最版本，或者内核和驱动对不上导致手机功能不正常；另外，编译内核无法通用，一个手机上能使，换个手机就不一定有条件了。

然而，如果我们使用 eBPF，那事情简直不要太简单，写几行脚本就能实现，甚至有的还是现成的，比如 `opensnoop` 和 `execsnoop` 工具，以下是对某加固程序的 `opensnoop` 运行截图：

[![](https://blog.dimenspace.com/mweb/16549625026785.jpg)](https://blog.dimenspace.com/mweb/16549625026785.jpg)

### 应用程序插桩

用 eBPF 可以很方便地编写各种 portable 的 hook 程序；它可以通过内核的 kprobe/uprobe/tracepoints/USDT 来动态监视甚至修改系统的状态。其中，kprobe 可以在内核的几乎任意地方注入代码, 不只包括函数的入口和出口, 也可以是函数内部的某个偏移地址；uprobe 是 kprobe 的用户空间版本，它可以对你的应用程序注入代码。比如说，我曾想观察 Android 应用对线程的创建，为此，我通过 plt hook 技术 hook 了 libc.so 中的 pthread_create 方法；而现在用 eBPF 的话，通过 bpftrace，我都不需要编写额外的代码，一行就搞定了：

```
bpftrace -e 'BEGIN { printf("%-10s %-6s %-16s %s\n", "TIME(ms)", "PID", "COMM", "FUNC");} uprobe:/apex/com.android.runtime/lib64/bionic/libc.so:pthread_create{ printf("%-10u %-6d %-16s %s\n", elapsed /1000000, pid, comm, usym(arg2));}'
```

### 性能问题分析

由于 eBPF 可以对内核中几乎所有函数插桩，如果我们在内核的某些关键链路加入观测逻辑，那我们就可以监控特定组件的性能；比如说，我们发现应用程序启动过慢，怀疑是有大量的文件读写积累耗时。传统的方法比如微信的 Mars 组件，通过 hook libc 中的 io 函数做监测，而如果使用 eBPF，我们可以直接在内核中监控 vfs，开销更低并且更准确，而且还简单.. 因为别人已经写好了，一个命令完事；如果我们想监控系统的 io 和网络模块，同样一个命令就能告诉你个大概。。

### 抓包

抓包也是一个很常见的需求，通常情况下，我们有这么些方式：

1.  在局域网的电脑上建代理，然后设置手机的 WiFi 通过电脑的代理上网，进而在电脑上抓包。
2.  通过 Xposed 插件或者 Frida 来 HOOK 特定的网络请求方法。
3.  在手机上建立 VPN，通过 VPN 进行抓包。这种方法比较通用，适合普通人使用。

以上无论何种方式，都可能被应用程序检测到，并且，通常情况下 https 的包抓起来没有那么容易。而如果我们用 eBPF，可以直接在内核中对网络模块插桩，无需绕过 https 证书直接抓包；事实上，有大佬已经写好了：[eCapture](https://github.com/ehids/ecapture)（虽然目前还不支持 Android，但理论上是可行的。

结语
--

当然，eBPF 的功能还远不止这么多，正如官方所述，它释放了无穷的可能性，我们可以拭目以待。

另外，虽然现在 eBPF 已经很成熟了，但是让它在 Android 系统上运行的资料并不多，接下来的文章我将介绍如何在 Android 系统中使用 eBPF 以及我们具体如何使用 eBPF，敬请关注～