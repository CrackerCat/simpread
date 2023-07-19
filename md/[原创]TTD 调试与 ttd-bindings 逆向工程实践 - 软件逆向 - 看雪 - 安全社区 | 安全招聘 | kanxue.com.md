> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278069.htm)

> [原创]TTD 调试与 ttd-bindings 逆向工程实践

[原创]TTD 调试与 ttd-bindings 逆向工程实践

16 小时前 1113

### [原创]TTD 调试与 ttd-bindings 逆向工程实践

 [![](http://passport.kanxue.com/upload/avatar/108/802108.png?1574223973)](user-home-802108.htm) [0x 指纹](user-home-802108.htm) ![](https://bbs.kanxue.com/view/img/rank/13.png) 5  ![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif) 16 小时前  1113

前言
==

在上篇文章《对一个 apk 协议的继续分析—libsgmain 反混淆与逆向 [^1](https://bbs.kanxue.com/thread-277665.htm) 》中的调试 trace 一节，我提到了微软的 TTD，即 Time Travel Debugging，还放了沈沉舟 [^2](https://scz.617.cn/windows/202201251528.txt) 和 krash[^3](https://bbs.kanxue.com/thread-273055.htm) 两位前辈的文章。

什么是 TTD 呢，按官网的介绍 [^4](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview) 来说，TTD 是一款用户级进程的 trace 录制工具，录制完成后可以在调试器中向前向后重放，不需要再重新运行程序就能让你的调试器状态回退，并且还能分享你的 trace 文件给别人，从方方面面帮助你更快更轻松找到 bug。

当时我只是通过各种文章了解到 TTD 的相关介绍，但一直没有什么契机和需求，就还没有实操过，甚至 Windbg 也没怎么摸过。直到沈沉舟前辈前段时间在微博上发了一个关于 Windows UWP Calculator 的 puzzle[^5](https://weibo.com/1273725432/N8JeKa7df):

> Win10 Calculator 是 UWP 程序，找出 UI 界面上乘法运算对应的汇编指令所在。要求计算器必须是 UWP，且版本不低于 11.2210.0.0。非 UWP 计算器不考虑，更低版本 UWP 不考虑，可以是 Win11 上的 UWP 计算器，只要版本不低于 11.2210.0.0。

当时一看到这个 puzzle，便想这是个绝佳的学习上手 TTD 的机会啊，随后便跟着沈沉舟前辈的 TTD 入门教程 [^6](https://scz.617.cn/windows/202201251528.txt) 快速上手了新版的 Windbg 和 TTD 调试功能，没花多少时间便把这个 puzzle 解决了。

回答之后沈沉舟前辈又提出了一个升级版的问题 [^7](https://weibo.com/1273725432/N8VYK1DC2)：

> 假设已有 run 文件，如何在几分钟内定位四则运算的汇编指令所在。这类问题的答案最终都将演变成生产力工具。

因为我对沈沉舟前辈一些文章十分熟悉了，一看到这个升级版的问题，便知道解答方向正是《TTD 调试进阶之 ttd-bindings[^8](https://scz.617.cn/windows/202207271620.txt)》一文所讲的 ttd-bindings。

然后就快速上手学习了 ttd-bindings 的使用，做了各种尝试，踩了不少坑，解答了升级版的问题，可以做到一分钟左右定位四则运算的汇编指令所在。

整个过程还是十分宝贵有意义的，这也是我记录此文的主要原因。

一方面我体会到了 TTD 调试的神奇，当用 TTD 调试解决了第一个问题后，调试体验让我回味无穷，我很激动地跟学弟 xxr0ss 分享说 “逆向调试体验飞升了一个档次”。在刚开始沉浸在使用 TTD 反向调试带来的新鲜感中时候，我甚至想起了《百年孤独》中看冰的桥段:)。

不过事后我对 TTD 进行了一番互联网考古才发现，TTD 属于反向调试世界中的一员，而反向调试有着丰富的历史，发展至今已经比较成熟了，所以我这算是少见多怪了。

另一方面这种通过好的问题与挑战来学习的方式让我获益匪浅。虽然我很早就知道有 TTD 这种东西了，但是一直没有动手去尝试下，而沈沉舟前辈的 puzzles 却让我很快行动起来，短时间内便能熟悉 TTD 调试功能以及使用 ttd-bindings 去编程解决问题，可以看出激励作用是十分巨大的。

这种方式也是非常有利于学习的，沈沉舟前辈在《scz's puzzles[^9](https://scz.617.cn/misc/202206221800.txt)》一文中就提到说：“一个好问题可以产生很多意料之外的成果”，而我也是在解决问题过程中产生了十分多意料之外的成果，除了学习到 TTD 调试与 ttd-bindings 的使用，还了解到反向调试的发展历史和不同反向调试器的差异与一些技术实现原理。

TTD 调试
======

搜一搜的话，会发现除了官网文档，中文互联网关于 TTD 调试的内容少得可怜，而关于 ttd-bindings 使用介绍的更是几乎只有沈沉舟前辈的一篇 [^8](https://scz.617.cn/windows/202207271620.txt)（可能因为 ttd-bindings 是去年刚开源的）。。

沈沉舟前辈主要有这几篇：

*   MSDN 系列 (46)--WinDbg Preview TTD 入门 [^6](https://scz.617.cn/windows/202201251528.txt)
*   TTD 调试进阶之 ttd-bindings[^8](https://scz.617.cn/windows/202207271620.txt)
*   TTD 历史回顾 [^14](https://scz.617.cn/windows/202207191506.txt)

把这几篇文章给读一遍，基本上就可以知道 TTD 调试和 ttd-bindings 的来龙去脉和基本使用了。

TTD 官网文档也蛮丰富详细的，先上手简单操作感受一下，再回头仔细读一遍官网上 TTD 部分，也是十分不错的。

我也根据文档概括性简单说一下 TTD 吧。

TTD 功能要求 WinDbg 版本 1.0.13.0 或者更新，进行录制需要管理员权限，可以使用 WinDbg 客户端录制，也可以使用命令行录制，文档有一节专门说 TTD 命令行使用的，命令行爱好者可以看一下。

由于 TTD 录制是侵入式的技术，可能会和别的也使用了侵入式技术的应用有冲突，如跟踪了系统内存调用、阻止内存访问的应用，有反病毒软件和 Microsoft Enhanced Mitigation Experience Toolkit 等。应用了虚拟化框架 electron 的录制可能会出现死锁或者崩溃的情况。

录制完成会生成. run 后缀的 trace 文件，打开 Windbg 选择 Open trace file，输入文件路径，进入后会自动生成优化对跟踪信息访问的 idx 文件，trace 文件和 idx 文件都很大，idx 文件通常是 trace 文件的两倍大，所以要保证有充足的存储空间。trace 文件生成大小没有上限，文档说 WinDbg 可以重放几百 GB 大小的 trace 文件，影响 trace 生成大小的因素有执行的指令数、录制时间和应用内存大小。

不用 idx 文件来调试也是可以的，但不推荐，因为 idx 文件能保证从调试进程读取的内存值是最准确的，并且能够提高各种调试操作的效率。可以手动使用! index -status 命令检查关联 trace 文件的 idx 文件状态，!index -force 命令可以重新生成 idx 文件。如果 trace 文件不能用出现了一些报错，可能就需要重新进行录制了。

文档还提到了一个在调试中可能会碰到的错误信息 “Derailment events”[^10](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-troubleshooting)，即执行脱轨事件。这个其实蛮有意思，涉及到了 TTD 的技术实现原理，TTD 录制并没有完全进行 trace，那样生成的 trace 文件会非常大，还有一部分技术是重执行，录制过程出错或是模拟执行引擎出错，都会导致脱轨事件，后面 TTD 互联网考古部分还会说到这个。

调试的话一般我喜欢把窗口调成这样。

![](https://bbs.kanxue.com/upload/attach/202307/802108_3NA2V2K577GD6DV.png)

左下角是 Timelines 时间线窗口，最左边是 trace 的开始，最后就是结束，时间线可以可视化表示执行过程中的各种事件，如断点、内存读写、call/ret 调用和异常，官网文档介绍得很详细，可以直接去读一遍官网。

TTD 时间点位置表示是用两个十六进制数组成 Major:Minor 这样的形式，Major 是序列编号，对应一个序列事件，Minor 是步数，大致相当于序列事件后的指令数。使用! position 命令可以查看当前所处的时间位置，使用! tt Major:Minor 命令达到相应的时间点位置，!tt 还可以跟着一个十进制数，表示达到整个 trace 时间线的百分比位置。

![](https://bbs.kanxue.com/upload/attach/202307/802108_9TMDW3JXCYGUDEN.png)

左上角是四个正向调试操作，四个反向调试操作，第一次使用反向调试操作的话会有十分新奇的体验。

![](https://bbs.kanxue.com/upload/attach/202307/802108_YYQWMBX2HTAS5UT.png)

导航栏有个 “Time Travel”，主要是一些命令的图形化按钮。Index Trace 对应命令! index -force，未自动生成 idx 文件的话可以用这个手动生成。Events 可以图形化查看模块加载和异常事件对象的属性如所处的时间点，命令行也可以实现。两个 Time travel to start 和 Time travel to end 对应命令! tt 0 和! tt 100，可快速到达开始和结束的地方。Information 可以查看 trace 信息，包括 trace 文件大小，创建时间和线程数量。Timelines 就是打开时间线窗口。

![](https://bbs.kanxue.com/upload/attach/202307/802108_NJRN8AADFKAU476.png)

相比于别的反向调试器，TTD 一个更强大的地方可能是，调试器加载 trace 文件后，会将 trace 过程中各种属性操作或事件生成对应的 TTD 数据模型对象，然后可以使用 LINQ 进行查询，关于 LINQ 放一段官网文档的介绍 [^11](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/)：

> Language-Integrated Query (LINQ) is the name for a set of technologies based on the integration of query capabilities directly into the C# language. Traditionally, queries against data are expressed as simple strings without type checking at compile time or IntelliSense support. Furthermore, you have to learn a different query language for each type of data source: SQL databases, XML documents, various Web services, and so on. With LINQ, a query is a first-class language construct, just like classes, methods, events. You write queries against strongly typed collections of objects by using language keywords and familiar operators. The LINQ family of technologies provides a consistent query experience for objects (LINQ to Objects), relational databases (LINQ to SQL), and XML (LINQ to XML).

这样相当于 TTD 为我们把 trace 过程中各种信息事件整理好了，我们只要按需查询就可以了，别的很多反向调试器远远没有做到这么多，只是在普通调试器基础上增加了录制重放、反向调试和反向断点等功能。

可以在官网文档中看到数据模型对象的种类十分丰富。

![](https://bbs.kanxue.com/upload/attach/202307/802108_9BFG6D74VDD5VZC.png)

举个内存对象（Memory objects）结合实践的例子吧，场景就是我们在堆中搜索到了一个值 0x9420fef2 的地址是 0x149808ac440 ，想找到这个值计算产生的汇编现场，已经完成了 TTD 录制过程。  
![](https://bbs.kanxue.com/upload/attach/202307/802108_EHTHUYKNPE6DSY7.png)

我们使用 LINQ 查询写入或读取 0x149808ac440 地址开始的四个字节的内存对象，就是 @$cursession.TTD.Memory(0x149808ac440,0x149808ac444,"rw")，使用 dx -g 以网格形式显示出来：

![](https://bbs.kanxue.com/upload/attach/202307/802108_NHM76YXAMZ6BTEQ.png)

一共有 0x4a 条，每条都有很多属性，匹配的太多了，我们只需要读写了值 0x9420fef2 的部分，对应的属性是 Value，我们可以使用 Where 从句过滤一下 .Where(m=>m.Value==0x9420fef2)。

![](https://bbs.kanxue.com/upload/attach/202307/802108_TDBTN8ASAWECSEQ.png)

可以看到符合条件的有一条查询，表明在 223E1:B53 时间点位置，有对 0x149808ac440 地址写入四个字节即值 0x9420fef2 的操作，接着我们可以执行! tt 223E1:B53 跳转到操作时间点位置分析汇编，或者直接点击蓝色的时间点，会执行对应的命令，也可以跳转到操作时间点位置。

是不是十分方便，其实在 TTD 和别的反向调试器中 “数据断点 + 反向执行” 也可以实现同样的效果，但是很耗时，远不如这种 TTD 整理好后，通过 LINQ 查询的方式快速与便捷。

TTD 官网文档中对各种数据模型对象的属性和查询方式都有详细的说明介绍 [^12](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-object-model)，值得仔细阅读遍历一遍。

ttd-bindings
============

TTD 的基本情况差不多说完了，然后开始说下 ttd-bindings 吧, 引用一下沈沉舟前辈对 ttd-bindings 的介绍：

> TTD 录制的. run 文件格式未公开，之前只能在 WinDbg Preview 中操作. run，很难对之
> 
> 进行脚本操作，比如，想对 position 进行大小比较，很费劲。
> 
> Bindings for Microsoft WinDBG TTD [https://github.com/commial/ttd-bindings](https://github.com/commial/ttd-bindings)
> 
> 上文作者对 TTDReplay.dll 进行逆向工程，逆出一套编程解析处理. run 文件的 API。没有完整逆向，但基本功能都有，比 windbgx 的 GUI 还多出一些功能。
> 
> C++ bindings 功能较多，Python bindings 功能弱一些，后者无法设置
> 
> CallRetCallback、MemCallback。此外，pyTTD.pyd 用的是 Python 3.10，为了在
> 
> Python 3.9 中用，必须自己编译出 pyTTD.pyd。
> 
> 作者提供了. cpp、py 的使用示例，对于逆向工程人员来说，浅显易懂，可以照猫画虎实现自定义功能。

上手的话，仔细读一遍《TTD 调试进阶之 ttd-bindings》[^8](https://scz.617.cn/windows/202207271620.txt)，再下载项目代码 [^13](https://github.com/commial/ttd-bindings) 看看几个 example 的实现，动手编译一下，跟着写一写就能熟悉使用了。

借助 ttd-bindings 可以通过编程来实现对 trace 文件的自动化操作处理，比如可以基于 trace 文件开发出多样实用的分析工具，项目仓库中就给出了四个工具的实现样例：

![](https://bbs.kanxue.com/upload/attach/202307/802108_YJFHTCJGRERVYM2.png)

以及在调试过程中，有时候我们想进行一些重复连续的处理，比如我们想遍历 trace 每一条指令的执行的同时，对寄存器值进行一些判断，光靠 WinDbg 的 TTD 调试就有些困难了，但这时候就是 ttd-bindings 大显身手的时候了。

不过 ttd-bindings 毕竟是逆向出来的 API，不仅. run 文件格式没有公开，TTD 内部实现原理和细节公开的资料也不多，所以用起来多多少少会踩一些坑。

总的来说 TTD 调试本身已经很好用了，ttd-bindings 则让 TTD 更加好用。

TTD 互联网考古
=========

受沈沉舟前辈《TTD 历史回顾 [^14](https://scz.617.cn/windows/202207191506.txt)》一文启发，我对 TTD 进行了一番互联网考古。发现在互联网历史中人们更习惯说 Reverse Debugging，而非 Time Traveling Debugging,Jakob 在《A new (and old) Reverse Debugger – Microsoft WinDbg[^15](https://jakob.engbloms.se/archives/2649)》一文最后 TTD? Reverse? 一节里写到说目前为止，还是很多工具、项目和公司用的是反向调试。

> It seems that time-traveling debug or time travel debugging is becoming more common as a term (again). Still, most tools and companies and projects still talk about reverse debugging. Undo, RR, gdb, and Simics all use “reverse” to describe the feature. If I look at the history of the field as far as I have been able to trace it, most people call it reverse until the a 2005 paper introduces “time-traveling virtual machines”.
> 
> For more on the history of reverse debuggers, see my three-part series of blogs from 2012 that I have since updated as new products and tools have appeared or been found in the archive of history: history 1, history 2, history 3.

如果在谷歌中搜索 Reverse Debugging，相关内容确实要比搜索 Time Traveling Debugging 多得多。

顺便一提，Jakob 关于反向执行调试的一系列博客 [^16](https://jakob.engbloms.se/?s=Reverse)，是我这次考古中最大的收获，学习思考到了十分多，这里向大家推荐一下。下面这个是《Reverse History》三篇中第一篇 [^17](https://jakob.engbloms.se/archives/1547) 的开头，就很引人入胜。

![](https://bbs.kanxue.com/upload/attach/202307/802108_4GZTJ5UUB29SE7R.png)

我最初除了从沈沉舟前辈这里了解到 TTD 调试，还从 krash 前辈《使用时间无关调试技术 (Timeless Debugging) 高效分析混淆代码 [^3](https://bbs.kanxue.com/thread-273055.htm)》一文中了解到 TTD，他在文中写到：“我理解的时间无关调试就是记录程序执行过程中的寄存器，和内存变化，使用记录的 trace 离线调试分析的过程，简而言之记录 trace，分析 trace。”

![](https://bbs.kanxue.com/upload/attach/202307/802108_BC36TYJXKSAATMG.png)

在我进行一番 TTD 互联网考古后，才发现不完全如此。

一方面，这种基于录制的 Trace-based debugging 只是反向调试的一种，主要体现为调试工作都在 trace 上进行，在《Reverse History Part One[^17](https://jakob.engbloms.se/archives/1547)》文中有段有详细写到：

> Trace-based debugging is based on recording everything a system does into a trace, and once the recording is done, debugger works on the trace rather than using the log to drive an actual system. Reverse debug is implemented by recreating the state of a system by reading the trace, and finding points in the trace where breakpoint conditions are true. This is also known as post-mortem debug, since you debug after the target system has finished executing (typically). A log can be captured by a hardware device, or by software being instrumented to log everything that is going on. The technology can also be used to implement record-replay debugging.

除了 Trace-based 的，还有种是比较少见并且重型的 Full-system-simulation-based 类型的反向调试，比较有名的是 Simics，可以做到什么程度呢 [^18](https://www.windriver.com/blog/back-to-reverse-execution)：

> Simics reverse execution and reverse debugging is a unique and very powerful feature of the simulator. In this blog post and accompanying video, we will look at what exactly it is you can do with reverse execution in Simics. It is not just a matter of running backwards to a breakpoint or stepping instructions (pick up my 2012 S4D article for more on reverse debugging). Reverse execution fundamentally is about undoing arbitrary operations on the target and going back to a prior state from a previous point in time. Furthermore, with Simics, you can go back in time and choose to do something different than you did in the original execution.

可以看到最后一句说到除此之外你可以选择回到过去做一些和原来执行不同的事情。

我在另一款同样 Full-system-simulation-based 类型的反向调试器 Simulics 官网 [^19](http://www.simulics.com/index_en.php) 上找到了一组生动的图可以很贴切地描述这种能力。

![](https://bbs.kanxue.com/upload/attach/202307/802108_EZTHK7MQEQCX4V7.png)

左边是 Trace-based 调试器可以做到的事情，而 Full-system-simulation-based 不仅可以做到左边，还能做到右边的事情，即回到过去做一些不一样的事情。

另一方面，像 TTD 的话，不完全是 Trace-based，还混合了重执行技术，Jakob 在《A new (and old) Reverse Debugger – Microsoft WinDbg[^15](https://jakob.engbloms.se/archives/2649)》文中称为 “a mix of re-execution and trace-based reverse debug”，

> There is mention of the emulator-based approach in the docs – it is found in a note about “derailment”, where the reverse debugger realizes that it has created an inconsistent current state of the program during reverse. From the TTD docs:
> 
> “TTD works by running an emulator inside of the debugger, which executes the instructions of the debugged process in order to replicate the state of that process at every position in the recording. Derailments happen when this emulator observes some sort of discrepancy between the resulting state and information found in the trace file.”

可以看到 TTD 的重执行是由调试器内部的模拟执行负责的，执行过程中会与 trace 文件中的信息对比看是否有差异，有差异的话就是 “derailment” 即脱轨。Jakob 后面说到 TTD 高效录制与没有产生大量 trace 文件的关键，是只记录了不能通过运行代码重构出来的内存值，然后结合重执行技术做到反向调试。

> The key to making this work efficiently and without gigantic trace files is to only record the memory values that cannot be reconstructed by running the code. Thus, we have a mix of re-execution and trace-based reverse debug, where the re-execution engine is used for short sequences of instructions, before getting the state forced back to the trace as needed.

还有个概念是 Keyframes，Jakob 说在 Simics 反向调试中的叫法是 Checkpoints，有着所有线程的完整状态，方便调试器跳转到执行中的某些特定的时间点。

> The “key frames” used in the time concept are more like checkpoints in Simics reverse debugging, in that they represent a complete state across all threads. It makes it easy for the debugger to jump to certain points in the execution and set up a consistent state across the threads there. Neat and useful.

后面我看了下 rr，rr 反向调试器中用的也是 Checkpoints，并且这个东西似乎和反向执行的实现原理有关，在 rr 官网中有段这样的介绍 [^20](https://rr-project.org/)：

> Furthermore, since debugging is the process of tracing effects to their causes, it's much easier if your debugger can execute backwards in time. It's well-known that given a record/replay system which provides restartable checkpoints during replay, you can simulate reverse execution to a particular point in time by restoring the previous checkpoint and executing forwards to the desired point.

说是从检查点向前执行来模拟实现反向执行，不知我在使用 ttd-bingdings 时候，发现 ReplayBackward 向前执行极其慢而 ReplayForward 向后就很快，是不是这个原因。

我在 Jakob 系列博文考古中还发现了很多有意思的事情。

*   《Borland Turbo Debugger – Reverse Execution in 1992[^21](https://jakob.engbloms.se/archives/2788)》，文中提到了很多文档资料，我根据线索下载了一本 92 年的调试器用户手册（TURBO DEBUGGER 3.0 FOR WINDOWS[^22](http://www.bitsavers.org/pdf/borland/turbo_debugger/Turbo_Debugger_3.0_for_Windows_Users_Guide_1991.pdf)），读起来十分新鲜有趣，那时候调试器功能就十分丰富了，有基于 trace 进行的有限反向执行；
    
*   gdb 在 7.0 版本就添加了反向调试功能，但不太好用，开销大很慢，gdb 的反向调试命令被很多反向调试器当作调试的前端，并没有真的用到 gdb 内置反向调试功能。
    
*   在《rr–The Mozilla Reverse Debugger[^23](https://jakob.engbloms.se/archives/2306)》一文评论区出现了文中提到的 rr 首席开发人员 Robert O'Callahan 亲自答疑，《A Replay Debugger from 1995! [^24](https://jakob.engbloms.se/archives/2350)》一文评论区出现了这款调试器的作者亲自讲述这款重放调试器的开发经历。
    

...

还有很多，就不细说了，大家有兴趣也可以去看一看，挖一挖。

对 scz‘s puzzles 的解答
===================

先贴一下我之前对两个问题的回答吧。

问题一的解答：

> 11.2210.0.0 版本，用 TTD 鞭尸找到了 CalcViewModel+0x12cb06: 00007ff9`5eeccb06 4c0fafc5 imul r8,rbp

问题二的解答：

> 关于 #scz's puzzles# 之 Win10 Calculator 升级版问题的解答：
> 
> 学了下 ttd-bindings，踩了些坑，简单实现了一分钟左右定位到加减乘的汇编和除法的结束现场（除法似乎是间接方式实现的，结束现场由不是输入的数值得到的计算结果）。
> 
> 实现方式：
> 
> 先获取到 CalcViewModel.dll 模块的起始地址和大小，接着设置 call、ret 回调函数，回调函数中对触发时的地址进行判断，若属于 CalcViewModel.dll 模块中的地址，则从该时间点往后单步 ReplayForward，每执行一步就对常用寄存器值进行判断（加减乘判断输入的两个值，除法判断结果值），若包含相应的值就打印时间点再到 windbg 里面验证过滤。
> 
> 如何缩短定位时间：
> 
> 1. 遍历 "!tt 0" 到 "!tt 100" 的汇编代码时间花费很长，很多部分和计算无关不需要执行，尝试了挺久，通过设置 call、ret 回调函数找到第一个属于 CalcViewModel.dll 模块中的地址，从这个时间点开始单步 ReplayForward 执行，到达加减乘的汇编现场的时间可以接受。
> 
> 2. 让录制时间尽可能短，如提前打开计算器，先输入好两个数值，然后 attach 录制，快速点击计算操作，出结果后立马停止录制，使用上面的 ttd-bindings 编程思路，可在一分钟内定位到加减乘的汇编和除法的结束现场。
> 
> 踩的坑：
> 
> 1.ReplayBackward 执行极其慢，最初的思路是内存搜索计算结果，查找最早写这个数据的时间点，然后从这个时间点向前执行，但发现 ReplayBackward 真是龟速，ReplayForward 就很快。不过虽然 ReplayForward 快，但是架不住 run 文件的时间长，还是要尽可能地想办法缩小时间范围。
> 
> 2.auto ctxt = ttdcursor.GetContextx86_64()执行后，如果不 free(ctxt)，内存会一点点被耗尽，仓库代码里面没有 free 操作，遍历 trace 文件汇编代码发现内存满了我才意识到这个问题。。。huhuO7O6 说的 “太耗内存” 可能也是没有主动 free ctxt。
> 
> 3. 尝试通过开多线程分段跑不同段的时间线，以加快定位速度，但是发现线程里面调用 ttd-bindings 的函数，程序会直接停止执行，原因不明。。
> 
> 4. 如果 callret 回调函数中的操作和时间点顺序有关，需要生成 run 文件的 idx 文件，不然 callret 回调函数中获取的时间点，打印出来会发现从前到后是乱序的，即不是从 start 到 last。
> 
> 5.ttdcursor.ReplayForward(&replayrez, last, -1)，第三个形参是 - 1 的话，无法触发 call、ret 回调函数，这个坑让我十分困惑，不过没多久我在仓库的 issues 里面发现 yara-ttd 的作者 atxr 提了 issue 描述了这个问题（github.com/commial/ttd-bindings/issues/27）。。。
> 
> 总结：
> 
> TTD 十分好用，ttd-bindings 让 TTD 变得更好用。

puzzles 解答细节
============

问题一
---

问题一解答的话（不用 ttd-bindings），先熟悉下 TTD 的使用，然后跟着《MSDN 系列 (46)--WinDbg Preview TTD 入门 [^2](https://scz.617.cn/windows/202201251528.txt)》一文中的 Calculator 示例部分来就行了。整体思路就是计算完后搜索结果在内存堆上的地址，然后 TTD 调试向前查询追踪数值写入的时间点位置，然后! tt 命令回退时间，分析流程，就这样再不断向前回溯至乘法计算汇编现场，最终会找到 CalcViewModel+0x12cb06: imul r8,rbp 处。

我是事后才发现如果计算结果比较大的话，计算过程会有一些对 r8 的变换处理，如我输入的是 0xabcd*0xdcba = 0x9420fef2，会经过 mov rcx,r8 和 btr exc 1Fh 计算，ecx 中的值就是 0x1420fef2 被写入到内存中，到最后又经过变换变成 0x9420fef2 显示出来。

也就是说 0x9420fef2 被写入内存离乘法汇编现场是有一段距离的，直接查询穿越到 0x9420fef2 的内存写入时间位置，还要继续向前不断回溯找到由 0x1420fef2 变为 0x9420fef2 的地方，再回溯 0x1420fef2 找到乘法现场。如果计算得到的结果没那么大，未被 btr ecx 1Fh 这些指令影响，直接写入到了内存中，应该可以免去这部分过程，很快定位到汇编现场。

问题二
---

问题二的解决思路还是很直观的，就是借助 ttd-bindings 遍历 trace 指令 + 判断通用寄存器值，再从符合条件的时间点检查进行过滤。

首先是正向遍历还是反向遍历，我最初的思路是走反向遍历的路子，需要先到达时间线末尾，内存搜索结果值地址，查询写入时间点位置，从这个位置向前遍历，但最后因为内存搜索不便自动化和 ReplayBackward 向后执行太慢放弃了。

正向遍历的话也是摸索了一段时间，主要是解决 ReplayForward 遍历整个 trace 文件时间过长的问题，不符合要求的几分钟内。摸索过程使用项目的 calltree 工具，进行了下输出打印的简化，发现前面大概有三分之一到四分之一的时间线长度是和 CalcViewModel 模块无关的（默认解答问题一后知道计算在这个模块中），便尝试了使用 callret 回调函数快速到达第一次出现 CalcViewModel 模块中的时间点，然后向后单步遍历执行，判断通用寄存器的值是否符合条件。最后经过测试，这个方式达到计算汇编现场花费的时间是可以接受的，能够控制在一分钟内。

贴一下我基于 example_calltree 修改的简陋累赘的代码，感兴趣的话可以自行研究一下：

```
#include #include #include #include "TTD/utils.h"
#include "TTD/TTD.hpp"
 
#define STEP_COUNT 100000
 
LARGE_INTEGER start, end, freq;
uint64_t calcViewModelStart, calcViewModelSize;
const wchar_t* path;
DWORD64 value1, value2;
 
void logTime() {
    QueryPerformanceCounter(&end);
    std::cout << "    Elapsed time: " << static_cast(end.QuadPart - start.QuadPart) / freq.QuadPart << " seconds." << std::endl;
}
 
bool findValueInRegs(PCONTEXT ctxt, uint64_t val) {
    if (ctxt->Rax == val or ctxt->Rcx == val or ctxt->Rdx == val or ctxt->Rbx == val or ctxt->Rsp == val or ctxt->Rbp == val
        or ctxt->Rsi == val or ctxt->Rdi == val or ctxt->R8 == val or ctxt->R9 == val or ctxt->R10 == val
        or ctxt->R11 == val or ctxt->R12 == val or ctxt->R13 == val or ctxt->R14 == val or ctxt->R15 == val) {
        return 1;
    }
    return 0;
}
 
void findCalcAsm(TTD::Position *startPosition) {
    TTD::ReplayEngine ttdengine = TTD::ReplayEngine();
    int result = ttdengine.Initialize(path);
    if (result == 0) {
        std::cout << "Fail to open the trace";
        exit(-1);
    }
    TTD::Cursor ttdcursor = ttdengine.NewCursor();
    ttdcursor.SetPosition(startPosition);
    TTD::Position* last = ttdengine.GetLastPosition();
    TTD::TTD_Replay_ICursorView_ReplayResult replayrez;
    TTD::TTD_Replay_ICursorView_ReplayResult* ret;
    do {
        ret = ttdcursor.ReplayForward(&replayrez,last, 1);
        auto ctxt = ttdcursor.GetContextx86_64();
        if (findValueInRegs(ctxt, value1) && findValueInRegs(ctxt,value2) ){
            printf("(%llx) [%llx:%llx] Find!\n", ttdcursor.GetProgramCounter(), ttdcursor.GetPosition()->Major, ttdcursor.GetPosition()->Minor);
            logTime();
        }
        free(ctxt);
    } while (ret->unk3 != 0);
    printf("over.\n");
    logTime();
    exit(0);
}
 
void callCallback_tree(unsigned __int64 callback_value, TTD::GuestAddress addr_func, TTD::GuestAddress addr_ret, struct TTD::TTD_Replay_IThreadView* thread_view) {
    uint64_t pc = thread_view->IThreadView->GetProgramCounter(thread_view);
    TTD::Position* position = thread_view->IThreadView->GetPosition(thread_view);
    if ( pc > calcViewModelStart and pc < calcViewModelStart + calcViewModelSize) {
        printf("(%llx) [%llx:%llx] Enter CalcViewModel, start stepping-by-steping to search for calc instructions.\n", addr_func, position->Major, position->Minor);
        logTime();
        findCalcAsm(position);
    }
    return;
}
 
bool containsString(const wchar_t* str, const wchar_t* substr) {
    std::wstring s(str);
    std::wstring sub(substr);
    return s.find(sub) != std::wstring::npos;
}
int wmain(int argc, const wchar_t* argv[])
{
    if (argc < 4)
    {
        wprintf(L"Usage: program.exe \n");
        return 1;
    }
 
    path = argv[1];
    value1 = wcstoull(argv[2], nullptr, 0);
    value2 = wcstoull(argv[3], nullptr, 0);
 
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&start);
 
    TTD::ReplayEngine ttdengine = TTD::ReplayEngine();
    TTD::TTD_Replay_ICursorView_ReplayResult replayrez;
     
    std::cout << "Openning the trace\n";
    int result = ttdengine.Initialize(path);
    if (result == 0) {
        std::wcerr << "Fail to open the trace";
        exit(-1);
    }
 
    const TTD::TTD_Replay_Module* mod_list = ttdengine.GetModuleList();
    for (int i = 0; i < ttdengine.GetModuleCount(); i++) {
        if (containsString(mod_list[i].path, L"CalcViewModel.dll")) {
            calcViewModelStart = mod_list[i].base_addr;
            calcViewModelSize = mod_list[i].imageSize;
            break;
        }
    }
 
    TTD::Cursor ttdcursor = ttdengine.NewCursor();
    TTD::Position* first = ttdengine.GetFirstPosition();
    TTD::Position end = *ttdengine.GetLastPosition();
 
    ttdcursor.SetPosition(first);
    ttdcursor.SetCallReturnCallback((TTD::PROC_CallCallback)callCallback_tree, 0);
 
    TTD::Position LastPosition;
    unsigned long long stepCount;
    unsigned long long totalStepCount = 0;
    for (;;) {
        ttdcursor.ReplayForward(&replayrez, &end, STEP_COUNT);
        stepCount = replayrez.stepCount;
        totalStepCount += stepCount;
 
        if (replayrez.stepCount < STEP_COUNT) {
            ttdcursor.SetPosition(&LastPosition);
            ttdcursor.ReplayForward(&replayrez, &end, stepCount - 1);
            totalStepCount += stepCount - 1;
            break;
        }
        memcpy(&LastPosition, ttdcursor.GetPosition(), sizeof(LastPosition));
    }
    return 0;
} 
```

总结
==

本文讲述了笔者由 scz’s puzzles 的两个 Win10 UWP Calculator 问题展开的 TTD 调试与 ttd-bindings 的逆向工程实践，分享了对 TTD 调试与 ttd-bindings 的学习与理解，还有一些在对 TTD 互联网考古过程中发现的有趣的事情。目前 TTD 调试与 ttd-bindings 实践的资料不算多，希望未来能看到更多相关的分享出现，也十分感谢沈沉舟前辈和 ttd-bindings 项目开源开发者们的无私分享。

引用链接
====

1: [https://bbs.kanxue.com/thread-277665.htm](https://bbs.kanxue.com/thread-277665.htm)  
2: [https://scz.617.cn/windows/202201251528.txt](https://scz.617.cn/windows/202201251528.txt)  
3: [https://bbs.kanxue.com/thread-273055.htm](https://bbs.kanxue.com/thread-273055.htm)  
4: [https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview)  
5: [https://weibo.com/1273725432/N8JeKa7df](https://weibo.com/1273725432/N8JeKa7df)  
6: [https://scz.617.cn/windows/202201251528.txt](https://scz.617.cn/windows/202201251528.txt)  
7: [https://weibo.com/1273725432/N8VYK1DC2](https://weibo.com/1273725432/N8VYK1DC2)  
8: [https://scz.617.cn/windows/202207271620.txt](https://scz.617.cn/windows/202207271620.txt)  
9: [https://scz.617.cn/misc/202206221800.txt](https://scz.617.cn/misc/202206221800.txt)  
10: [https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-troubleshooting](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-troubleshooting)  
11: [https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/)  
12: [https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-object-model](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-object-model)  
13: [https://github.com/commial/ttd-bindings](https://github.com/commial/ttd-bindings)  
14: [https://scz.617.cn/windows/202207191506.txt](https://scz.617.cn/windows/202207191506.txt)  
15: [https://jakob.engbloms.se/archives/2649](https://jakob.engbloms.se/archives/2649)  
16: [https://jakob.engbloms.se/?s=Reverse](https://jakob.engbloms.se/?s=Reverse)  
17: [https://jakob.engbloms.se/archives/1547](https://jakob.engbloms.se/archives/1547)  
18: [https://www.windriver.com/blog/back-to-reverse-execution](https://www.windriver.com/blog/back-to-reverse-execution)  
19: [http://www.simulics.com/index_en.php](http://www.simulics.com/index_en.php)  
20: [https://rr-project.org/](https://rr-project.org/)  
21: [https://jakob.engbloms.se/archives/2788](https://jakob.engbloms.se/archives/2788)  
22: [http://www.bitsavers.org/pdf/borland/turbo_debugger/Turbo_Debugger_3.0_for_Windows_Users_Guide_1991.pdf](http://www.bitsavers.org/pdf/borland/turbo_debugger/Turbo_Debugger_3.0_for_Windows_Users_Guide_1991.pdf)  
23: [https://jakob.engbloms.se/archives/2306](https://jakob.engbloms.se/archives/2306)  
24: [https://jakob.engbloms.se/archives/2350](https://jakob.engbloms.se/archives/2350)

  

[CTF 训练营 - Web 篇](https://www.kanxue.com/book-section_list-105.htm)

最后于 1 小时前 被 0x 指纹编辑 ，原因： [#调试逆向](forum-4-1-1.htm)