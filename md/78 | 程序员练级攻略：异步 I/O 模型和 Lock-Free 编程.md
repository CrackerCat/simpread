> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [time.geekbang.org](https://time.geekbang.org/column/article/9851)

> 左耳朵耗子的机器学习入门 “心法” 及优质学习资源推荐。

异步 I/O 模型是我个人觉得所有程序员都必需要学习的一门技术或是编程方法，这其中的设计模式或是解决方法可以借鉴到分布式架构上来。再说一遍，学习这些模型，是非常非常重要的，你千万要认真学习。

史蒂文斯（Stevens）在《UNIX 网络编程》一书 6.2 I/O Models 中介绍了五种 I/O 模型。

阻塞 I/O

非阻塞 I/O

I/O 的多路复用（select 和 poll）

信号驱动的 I/O（SIGIO）

异步 I/O（POSIX 的 aio_functions）

然后，在前面我们也阅读过了 - C10K Problem 。相信你对 I/O 模型也有了一定的了解。 这里，我们需要更为深入地学习 I/O 模型，尤其是其中的异步 I/O 模型。

接下来，我们需要了解一下各种异步 I/O 的实现和设计方式。

我简单总结一下，基本上来说，异步 I/O 模型的发展技术是： select -> poll -> epoll -> aio -> libevent -> libuv。Unix/Linux 用了好几十年走过这些技术的变迁，然而，都不如 Windows I/O Completion Port 设计得好（免责声明：这个观点纯属个人观点。相信你仔细研究这些 I/O 模型后，你会有自己的判断）。

看过这些各种异步 I/O 模式的实现以后，相信你会看到一个编程模式——Reactor 模式。下面是这个模式的相关文章（读这三篇就够了）。

然后是几篇有意思的延伸阅读文章。

Lock-Free - 无锁技术越来越被开发人员重视，因为锁对于性能的影响实在是太大了，所以如果想开发出一个高性能的程序，你就非常有必要学习 Lock-Free 的编程方式。

关于无锁的数据结构，有几篇教程你可以看一下。

接下来，读一下以下两篇论文 。

最后，有几个博客你要订阅一下。

1024cores - 德米特里 · 伐由科夫（Dmitry Vyukov）的和 lock-free 编程相关的网站。

Preshing on Programming - 加拿大程序员杰夫 · 普莱辛（Jeff Preshing）的技术博客，主要关注 C++ 和 Python 两门编程语言。他用 C++11 实现了类的反射机制，用 C++ 编写了 3D 小游戏 Hop Out，还为该游戏编写了一个游戏引擎。他还讨论了很多 C++ 的用法，比如 C++14 推荐的代码写法、新增的某些语言构造等，和 Python 很相似。阅读这个技术博客上的内容能够深深感受到博主对编程世界的崇敬和痴迷。

Sutter’s Mill - 赫布 · 萨特（Herb Sutter）是一位杰出的 C++ 专家，曾担任 ISO C++ 标准委员会秘书和召集人超过 10 年。他的博客有关于 C++ 语言标准最新进展的信息，其中也有他的演讲视频。博客中还讨论了其他技术和 C++ 的差异，如 C# 和 JavaScript，它们的性能特点、怎样避免引入性能方面的缺陷等。

Mechanical Sympathy - 博主是马丁 · 汤普森（Martin Thompson），他是一名英国的技术极客，探索现代硬件的功能，并提供开发、培训、性能调优和咨询服务。他的博客主题是 Hardware and software working together in harmony，里面探讨了如何设计和编写软件使得它在硬件上能高性能地运行。非常值得一看。

接下来，是一些编程相关的一些 C/C++ 的类库，这样你就不用从头再造轮子了（对于 Java 的，请参看 JDK 里的 Concurrent 开头的一系列的类）。

Folly - Facebook 的开源库（它对 MPMC 队列做了一个很好的实现）。

MPMCQueue - 一个用 C++11 编写的有边界的 “多生产者 - 多消费者” 无锁队列。

SPSCQueue - 一个有边界的 “单生产者 - 单消费者” 的无等待、无锁的队列。

Userspace RCU - liburcu 是一个用户空间的 RCU（Read-copy-update，读 - 拷贝 - 更新）库。

libcds - 一个并发数据结构的 C++ 库。

liblfds - 一个用 C 语言编写的可移植、无许可证、无锁的数据结构库。

例如，在 IBM Blue Gene/Q 上有 64 个线程，我们观察到使用 Blue Gene/Q 硬件事务内存（HTM）的中值加速比为 1.4 倍，使用软件事务内存（STM）的中值加速比为 4.1 倍。什么限制了这些 TM 基准的性能？在本论文中，作者认为问题在于用于编写它们的编程模型和数据结构上，只要使用合适的模型和数据结构，程序的性能可以有 10 多倍的提升。

关于压缩的内容。为了避免枯燥，主要推荐下面这两篇实践性很强的文章。

Hints for Computer System Design ，计算机设计的忠告，这是 ACM 图灵奖得主 Butler Lampson 在 Xerox PARC 工作时的一篇论文。这篇论文简明扼要地总结了他在做系统设计时的一些想法，非常值得一读。（用他的话来说，“Studying the design and implementation of a number of computer has led to some general hints for system design. They are described here and illustrated by many examples, ranging from hardware such as the Alto and the Dorado to application programs such as Bravo and Star“。）

注明一下，Jim Gray 是关系型数据库领域的大师。因在数据库和事务处理研究和实现方面的开创性贡献而获得 1998 年图灵奖。美国科学院、工程院两院院士，ACM 和 IEEE 两会会士。他 25 岁成为加州大学伯克利分校计算机科学学院第一位博士。在 IBM 工作期间参与和主持了 IMS、System R、SQL／DS、DB2 等项目的开发。后任职于微软研究院，主要关注应用数据库技术来处理各学科的海量信息。

好了，总结一下今天的内容。异步 I/O 模型是我个人觉得所有程序员都必需要学习的一门技术或是编程方法，这其中的设计模式或是解决方法可以借鉴到分布式架构上来。而且我认为，学习这些模型非常重要，你千万要认真学习。

接下来是 Lock-Free 方面的内容，由于锁对于性能的影响实在是太大了，所以它越来越被开发人员所重视。如果想开发出一个高性能的程序，你非常有必要学习 Lock-Free 的编程方式。随后，我给出系统底层方面的其它一些重要知识，如 64 位编程、提高 OpenSSL 的执行性能、压缩、SSD 硬盘性能测试等。最后介绍了几篇我认为对学习和巩固这些知识非常有帮助的论文，都很经典，推荐你务必看看。

下面是《程序员练级攻略》系列文章的目录。