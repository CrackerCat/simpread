> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/JDQA4W_Ylutp03NvShceDw)

网上已经有很多阐述 ftrace 原理和使用方法的文章，本文不会面面俱到的介绍所有涉及的原理和方法，只会聚焦在阐述 ftrace 的 event tracing 机制，以及如何利用该机制（包括其他一些方法配合）去跟踪某个进程的调度信息。本文的分析基于处理器为 ARM64 的多核手机平台，linux 版本为 msm-4.19，为了行文方便，我们还是先回顾下 ftrace 的基本概念。

**一、 ftrace 基本原理**
==================

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqO9ofuTZe4g4t6y2NjibzQYbGfbib0QTpHKyRJK1yURSF0wa4vAddgvYog/640?wx_fmt=png)

图 1-1 ftrace 框架

一句话总结：各类 tracer 往 ftrace 主框架注册，不同的 trace 则在不同的 probe 点把信息通过 probe 函数给送到 ring buffer 中，再由暴露在用户态 debufs 实现相关控制。

对不同 tracer 来说

1）需要实现 probe 点（需要跟踪的代码侦测点），有的 probe 点需要静态代码实现，有的 probe 点借助编译器在运行时动态替换，event tracing 属于前者；

2）还要实现具体的 probe 函数，把需要记录的信息送到 ring buffer 中；

3）增加 debugfs 相关的文件，实现信息的解析和控制。

而 ring buffer 和 debugfs 的通用部分的管理由框架负责。

**二、ftrace event tracing 原理介绍**
===============================

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOad1RZlibLAGE7O9iaJcicZR4XXzib8777sAMVSfeZjOXgJU1UDuJCMOFDQ/640?wx_fmt=png)

图 2-1 ftrace event tracing 原理

其它一些 trace 工具也具有相似的工作流程，需要一个 probe 点，一个 probe 函数，一种保存 log 信息的机制，还有用于户进行交互的接口（分别对应图中 A、B、C、D 点）。它们发现可以利用 function trace 的后两部分，也就是环形缓存的管理和用户交互接口部分的代码，只需要实现自己的 probe 点和 probe 函数就可以了。于是在 function trace 进入内核 mainline 后，kprobe 也借用了该机制。

**1.  tracepoint 的实现原理**
------------------------

tracepoint 是早就存在于内核当中的一种 trace 工具，具体原理不再详细展开，示例代码大家可参照内核源码 sample/tracepoints / 相关内容。需要使用小于 3.9 版本的内核来看。原因是 linux 不期望开发人员直接使用 tracepoint，而应该使用更高层的 event tracing，但是有两个宏（DECLARE_TRACE 和 DEFINE_TRACE）会被 trace event 使用，需要看下。

### （1） DECLARE_TRACE

该宏在 <linux/tracepoint.h> 被定义。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqO9bDl4m8icUKvWqxuQuwhpBMcbNJyqU6rzCEA8YmiaYpSjJY7OLsY9yeg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOnN9mrpn11qTPzfzH4QAwhJQQRqibpyliayeB81ZzvvtcxaiaRUicztcxIA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOESxFqyBHTMp1DK1mXtgVFiaFsyCD0a7rDIRgAEI50Dhl9bTp63Yv3XA/640?wx_fmt=png)

trace_##name 最终调用__DO_TRACE 宏将信息送到 ring buffer 中。

reg/unregister_trace_subsys_event 负责注册或注销 probe 函数，probe 函数的原型由 TP_PROTO 宏给出。例如：wakeup 相关 tracer 则是调用了 register_trace_sched_wakeup。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOLwlK2U64qPMQckjKHpNRYqwXVEfkOXBykOJ9eqTdRIRECboKDcIp4g/640?wx_fmt=png)

使用 tracepoint 需要自己实现 probe 函数，event tracing 通过一套通用的机制省去了内核模块 / 驱动自己实现 probe 函数的麻烦。

*   __DO_TRACE
    

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOCepGorBDgBsJNrj6icqVpbNL4hdn8qiagugmEGoMpyXX0xvHiccBXj5OQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOI2OqA7dNUONhRYGRwfYloKG3os4GiadF0dINyVichGBfW5xTP1MJfsUg/640?wx_fmt=png)

该 (it_func_ptr)->func 和 (tp)->funcs 在 enable event 的时候会被赋值，下文会有展开描述。

### （2） DEFINE_TRACE

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqO8STHQe4v1icmPZKhuRicMy1Yhh1DccKtiaYyU9MfXRXpYV35XibGyMUiagA/640?wx_fmt=png)

定义了__tpstrtab_##name 字符串变量和 struct tracepoint __tracepoint_##name。当然二者都放在了链接脚本专门的 section 里（详见 include/asm-generic/vmlinux.lds.h）。编译后可查看 system.map 去查看具体的变量位置。

**2. event tracing 的实现原理**
--------------------------

event tracing 可以理解为 tracepoint 加 function trace 的接口组合而成，提供了接口给内核开发者，让其快速的定义信息保存的格式以及如何打印出来。具体原理可参照 samples/trace_events / 下的例子。

### （1）TRACE_EVENT

该宏可以实现 probe 点、probe 函数、event 的格式，以及 event 打印出来的格式。

第 1 次通过包含 include/linux/tracepoint.h，将其展开成 DECLARE_TRACE。

第 2 次通过 include/trace/trace_events.h，将其展开成 DECLARE_EVENT_CLASS，而在该头文件中 DECLARE_EVENT_CLASS 会被多次展开。

第 3 次通过包含 include/trace/define_trace.h，将其展开成 DEFINE_TRACE

其中，include/linux / 和 include/trace/define_trace.h 必须显示被使用者显式包含。

总结下：

*   trace_##name，是放置 probe 点的函数实现，如果该 event enable 则调用注册进来的 trace_event_raw_event_##call
    
*   trace_raw_output_##call，是输出信息的函数实现（cat trace 时输出）
    
*   trace_event_raw_event_##call，是 probe 函数
    

### （2）trace_##name

细节参见上文宏展开结果。

### （3）trace_raw_output_##call

该函数在 cat trace 文件节点的时候会被调用。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOb2HeyIa33IUlyT6P0Y6icOP95jnopNDicfRM2kt0VgOaPLZnTwsK1Rqg/640?wx_fmt=png)

### （4）trace_event_raw_event_##call

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOkKyZQ5R4micg4x2GYEJdibMKNicPWh0J2PEMxrBeXy2joAfyMnvOf11Qw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqO01nAiaCq8k2f89f0sgqNPuUs3vlzoTdHPFUWX3qETB9AY14pxiclIicsA/640?wx_fmt=png)

综上，通过上述宏展开，得到：

**3. 关键流程梳理**
-------------

### （1）定义 event

使用 TRACE_EVENT 宏定义 event 的原型、信息打印格式等。参见上文描述。

### （2）记录 event

代码中静态埋点，调用 trace_##name，放置 probe 点。参见上文描述。

### （3）使能 event

假设 tracefs 已经提前挂载好（可以通过 cat /proc/mounts | grep tracefs 命令查看）

tracefs /sys/kernel/debug/tracing tracefs rw,seclabel,relatime 0 0

对应命令：

echo 1 > /sys/kernel/debug/tracing/events/<one_trace_event>/enable

对应关键代码流程：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOlEosKSPgA7dPhmzlA7rdktY2mibeblcJpI4sVlvo3VBUQ6LYTQick8bg/640?wx_fmt=png)

将上文描述的展开后的宏里面的 trace_event_raw_event_##call 注册到 struct tracepoint_func 中进去

### （4）输出 event

对应命令：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOlmwvcPZN1pbz2U3bnA9t7AyWV9pYAtJu79NpOgLn7h81zFmEbOd7og/640?wx_fmt=png)

对应关键代码流程：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMeXGltZc6PicpD6DYMeCcZKyA6gT0995neUiaibZNNZzCQtCpC2Y3hLmkJYW5EpA9z80JliapXXfLnwQ/640?wx_fmt=png)

print_trace_fmt 函数代码截图：  

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOEUM6zCSF76qbptFB41PrCePUsrZ6TQiad2Cb5oKC4LosSeKZibxm6IvQ/640?wx_fmt=png)

funcs->trace，参见上文被赋值成 trace_raw_output_##call 了。

**三、使用 ftrace event tracing 跟踪特定进程调度原理**
========================================

本章我们主要介绍手机平台实现精准跟踪特定进程调度使用的一些技巧。因为我们最终还是要借助 systrace 这个 ftrace 前端，所以简单介绍 ftrace 相关 tag，包括 Java 层和 Native 层如何使用 ftrace 接口。同时我们要简单研究下调度相关的 trace event 以及 filter 功能，这是解决实际问题的关键。

**1. Android systrace tag list**
--------------------------------

使用 atrace –list 即可查看所有支持的 tag list。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOPAMYwNXT85sohaRnibKggMRzg6Kswjf3UQNYF90ibHs24jcu1scv5icBw/640?wx_fmt=png)

**2.  用户态如何使用 ftrace**
----------------------

有时候我们还要把用户态信息和内核信息同步起来，可以通过 trace_marker 实现。

### （1）使用 trace_marker

Systrace 利用 ftrace 提供了如下两个接口供用户程序调用：

*   Java 层：Trace.traceBegin(tag, name)/Trace.traceEnd(tag)
    
*   Native 层 ：ATRACE_BEGIN(name)/ATRACE_END()
    

当然，也可以根据特定需求，直接利用 systrace 现成的 tag（即还可以使用 systrace 设置 tag 和手动设置 sched 调度的 event filfter 搭配起来用）。如果想关闭用户态的信息打印也是可以的。

### （2）开关 trace_marker

查看：

cat options/markers

打开：

echo 1 > options/markers

关闭：

echo 0 > options/markers

**3. Android Linux 状态**
-----------------------

### （1）Android 状态转移

![](https://mmbiz.qpic.cn/mmbiz_jpg/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqO08VQeUzuJfp1Dfofdfsl6Ftq4NOy8zNDib7Mic7R3d02Qv9LwLSYmfSQ/640?wx_fmt=jpeg)

图 3-1 Android 进程状态转移

1）Runnable -> Running：就绪态的进程获得了 CPU 的时间片，进入运行态；

2）Running -> Runnable: 运行态的进程在时间片用完后，必须出让 CPU，进入就绪态；

3）Running -> Blocked：当进程请求资源的使用权 (如外设) 或等待事件发生 (如 I/O 完成) 时，由运行态转换为阻塞态；

4）Blocked -> Runnable：当进程已经获取所需资源的使用权或者等待事件已完成时，中断处理程序必须把相应进程的状态由阻塞态转为就绪态。

### （2）Linux 原生状态信息

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOghSZRWqHt0ykNJTRSH3vHSllTaMoxnGB1q6gojcE18IWdKKsOsG4dA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOjbVEmj7HaKic2eM6zkea2yPPW86Qf50ibbDU9Hk7kzuUZTzI5YKyhiaXw/640?wx_fmt=png)

**4. 调度相关 events 分析**
---------------------

可以通过 cat available_events | grep sched: 命令或者 ls events/sched / 去查看内核已经实现调度相关的 events。结合上文的进程状态分析，我们从中挑选 events 如下：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjMeXGltZc6PicpD6DYMeCcZKZ60WRhztz1IO8ibhKsmGtDq4PpticMHmiak6ZVW4Tuu4YY4fiaSCrcsK5Q/640?wx_fmt=png)

这些 trace events 足够我们分析线程级别的调度信息。

**5.  event 的 filter 功能**
-------------------------

为实现问题进程的跟踪，我们需要使用 event 的 filter 功能。

### （1）event filter 语法简单介绍

field-name relation-operatior value     

*   对于数字域，可以使用操作符 ==, !=, <, <=,>, >=, &
    
*   对于字符串域，可以使用 ==, !=, ~。目前字符串只支持完全匹配，且最多可以组合 16 个条件。”~” 支持通配符 (*,?)。
    

例子：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqON5OFVLaaje7BEz6NzRkDfzSVpZyxoebiaUwWib1ibaN1dx3fByeibw2rng/640?wx_fmt=png)

### （2）event filter format

可以先通过该 event 目录下的 format 查看 event 消息格式。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjM0D7oTfDHnlQRic8rxnAibqOiavbGC66lYHSL2VSIMV5tAyKrB1zd5Y4E2Q3icgH3w686svN07CnSKvA/640?wx_fmt=png)

我们依次分析根据上文所有需要跟踪的 events，可以根据 comm（包括 pre_comm，next_comm）和 PID 等设置过滤条件。

剩下就是把相关 events enable 就可以了，细节本文就不展开了。

**四、实际例子**
==========

**1. 实际问题**
-----------

我们在实际的项目中曾经遇到过低概率问题，那个问题是相机切换模式时预览卡顿并出现闪退。我们第一反应肯定是直接打开 systrace 去抓取 trace 回来，但面临个问题：

1）问题低概率出现，复现一次不容易，需要一次复现就抓到问题现场；

2）抓取 log 量还是过大（即使只用 “sched” 和 “camera”），问题场景对负载敏感；

3）问题复现后，需要手动停止，这块手动操作要到 10 几秒到 1 分钟不等，需要抓取 log 量尽可能短，以便抓到出问题时的现场，不然 ring buffer 被覆盖会导致前功尽弃。

**2. 问题分析**
-----------

问题关键是要让 ftrace 记录时间尽可能长，很自然能想到的措施：

1）增大 ftrace buffer；

2）减少 ftrace 记录的内容，同时要保证足够问题分析。

虽然该问题概率低，但好消息是只要压测时间足够久，还是能复现出来。通过分析规律，我们发现每次相机切换时，都有 AlgoInterface::init 出现，重点怀疑该对应的进程调度异常，经确认该方法会对应 “APSRoutine” 的进程异常。

所以解决问题的关键就变成：

1）利用 ftrace event tracing 的过滤功能，跟踪 “APSRoutine” 进程，具体的 events 的 filter 设置参见上一章内容；

2）要知道对应的用户态操作，所以必须要打开 trace_marker 开关（但 trace_marker 不支持过滤功能，这点比较遗憾，不然可以让记录的内容更少）；

3）在 Java 层 crash 的地方加 hook，crash 时抓取 ftrace。

所以做了这样满足上述条件的小工具。后续我们还可以将其完善，做成支持抓取任意 Java/Native 层特定进程调度相关 log 的功能。

这样我们就把 ftrace event tracing 的原理及一个基于 ftrace 的小工具解决实际问题的实践介绍完了，由于时间和个人水平有限，难免有出错的地方，如有还请大家帮忙指出。

参考材料
====

[1] 进程状态的切换，袁辉辉
---------------

[2] ftrace 中 eventtracing 的实现原理，普侯
----------------------------------

[3] 史上最长的宏定义，Kernel Exploring
-----------------------------

[4] https://www.kernel.org/doc/html/latest/trace/events.html
------------------------------------------------------------

![](https://mmbiz.qpic.cn/mmbiz_gif/d4hoYJlxOjM9WWBsVsUpiaGmAPiaTAJIsM8YMTErJrQy9vichOzuhB2BNSdKrKyQ0eOC0lYRVrPMLUJOvEOGto4Mg/640?wx_fmt=gif)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程