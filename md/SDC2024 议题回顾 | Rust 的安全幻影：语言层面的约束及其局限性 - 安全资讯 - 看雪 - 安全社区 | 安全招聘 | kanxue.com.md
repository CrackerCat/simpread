> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284170.htm)

> SDC2024 议题回顾 | Rust 的安全幻影：语言层面的约束及其局限性

Rust 语言的安全机制源自于其背后的约束，Rust 约束不仅是语言设计的核心，也是其安全性的重要保障。然而，这些约束并非无懈可击。在现实场景中， Rust 的所有权、借用和生命周期在某些特定情况下可能导致安全隐患。

一起来回顾下陈浩在 SDC2024 上发表的议题演讲：《Rust 的安全幻影：语言层面的约束及其局限性》

![](https://bbs.kanxue.com/upload/attach/202410/236762_XDP7A6PXTAMAWGS.jpg)

【陈浩：奇安信天工实验室安全研究员】

一、Rust 介绍

Rust 语言自其发布以来就备受人们关注，作为一门现代的系统级编程语言，Rust 在安全性方面引起了人们的极大兴趣。它与其他语言相比，引入了一系列创新的安全特性，旨在帮助开发者编写更可靠、更安全的软件。在这个基础上，许多大厂开始纷纷在自己的项目中引入 Rust，比如`Cloudflare`的`pingora`，Rust 版的 git -- `gitxoide`，连微软都提到要将自家的 win32k 模块用 rust 重写，足以见得其火爆程度。

之所以人们对 Rust 那么充满兴趣，除了其强大的语法规则之外，Rust 提供了一系列的安全保障机制也让人非常感兴趣，其主要集中在以下几个方面：

*   内存安全：Rust 通过使用所有权系统和检查器等机制，解决了内存安全问题。它在编译时进行严格的借用规则检查，确保不会出现数据竞争、空指针解引用和缓冲区溢出等常见的内存错误。
    
*   线程安全：Rust 的并发模型使得编写线程安全的代码变得更加容易。它通过所有权和借用的机制，确保在编译时避免了数据竞争和并发问题，从而减少了运行时错误的潜在风险。
    
*   抽象层安全检测：Rust 提供了强大的抽象能力，使得开发者能够编写更加安全和可维护的代码。通过诸如模式匹配、类型系统、trait 和泛型等特性，Rust 鼓励使用安全抽象来减少错误和提高代码的可读性。
    

Rust 强大的编译器管会接管很多工作，从而尽可能的减少各种内存错误的诞生。

二、Rust 不会出现漏洞吗？

在 Rust 的各类机制下，开发人员在编译阶段被迫做了相当多的检查工作。同时在 Rust 的抽象机制下，整体的开发流程得到了规范，理论上应该是很难再出现漏洞了。然而，安全本质其实是人，错误本质上是由人们的错误认知引发的。即便是在 Rust 的保护之下，人们也是有可能犯错，从而导致新的问题的出现。对于这种场景，我们可以用一种宏观的方法论来概括，那就是认知偏差。这里可以用一个图来大致描述一下这个认知偏差：

![](https://bbs.kanxue.com/upload/attach/202410/236762_BKZSYKHTW25JBYF.jpg)

换句话说，在使用 Rust 开发中，人们认为 Rust 能够提供的防护和 Rust 实际上提供的防护，这两者存在一定的差异。具体来说，可以有一下几种场景：

*   Rust 检查时，能否防护过较为底层的操作状态？
    
*   Rust 自身特性是否会引入问题？
    
*   Rust 能否检查出作为 mod 或者 API 被其他人调用时，也能完全保护调用安全吗？
    

为了能够更好的了解认知差异，接下来我们就介绍几种比较典型的 Rust 下容易出现的漏洞。

三、案例一：对操作系统行为的认知错误

在进行开发过程中，Rust 通常会需要与操作系统底层进行交互。然而在这些操作过程中，本质上是对底层的 API 或者对底层操作系统的操作，此时考察的是开发者对于操作系统的理解。而 Rust 编译器的防护机制并无法直接作用于这些底层的操作系统对象，从而会导致错误的发生。

一种常见的认知偏差就是默认操作系统提供的特性，比如说接下来要提到的特殊字符过滤规则。

1、BatBadBut（CVE-2024-24576）

在 2024 年 4 月，安全研究员 RyotaK 公开了一种他发现现有大部分高级语言中常见的漏洞类型，取名为`BatBadBut`，其含义为 batch 文件虽然糟糕，但不是最糟糕的。

> batch files and bad, but not the worst

_在 Windows 下，想要执行 bat 文件就必须要启动一个 cmd.exe，所以执行的时候通常会变成_`cmd.exe /c test.bat`。

每个高级语言在 Windows 平台下需要创建新的进程的时候，最终都会调用 Windows 的 API`CreateProcess`。为了防止命令注入，它们大多数会对参数进行一定的限制，然而 Windows 平台下的 CreateProcess 存在一定的特殊行为，使得一些常见的过滤手段依然能够被绕过。作者给了一个 nodejs 的例子，在 nodejs 中，当进行进程创建的时候，通常是这样做的：

![](https://bbs.kanxue.com/upload/attach/202410/236762_CB8UT2J8K8DXRZG.jpg)

这种做法通常是没问题的，此时由`CreateProcess`创建的进程为`echo`，参数为后续的两个参数。同时，这个调用过程中伴随的如下的过滤函数，会将`"`过滤成`\"`

![](https://bbs.kanxue.com/upload/attach/202410/236762_GSQBRY2BB6WM6MP.jpg)

此时，上述的指令会形成如下的指令：

![](https://bbs.kanxue.com/upload/attach/202410/236762_JHTZQ6VJXJQYE5G.jpg)

然而，当遇到如下代码的时候，情况会发生变化：

![](https://bbs.kanxue.com/upload/attach/202410/236762_ZT4KBSE5GGNXREJ.jpg)

因为 Windows 并没有办法直接的启动一个 bat 文件，所以实际上启动的时候，Windows 执行的实际逻辑变成了：

![](https://bbs.kanxue.com/upload/attach/202410/236762_N6NGMZYR5XQHNFA.jpg)

而实际上，在 Windows 中的`\`并非是我们理解的那种能将所有符号进行转义，转义字符。其只能转义`\`本身，类似于作为路径的时候，以及转义换行符。所以，上述的命令实际上等价于：

![](https://bbs.kanxue.com/upload/attach/202410/236762_B6GNUP7H2M3NZ23.jpg)此时命令解析模式如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_D3Q8XJU7P22X4JK.jpg)

可见依然发生了命令注入。实际上，如果想要在 Windows 下进行我们常规理解下的命令转换，要使用`^`符号，例如将上述指令修改成如下的形式，即可防止命令注入：

![](https://bbs.kanxue.com/upload/attach/202410/236762_3V99KECB37ZZEJV.jpg)

作者给出了他测试过的受到影响的语言：

*   Erlange
    
*   Go
    
*   Haskell
    
*   Java
    
*   Node.js
    
*   PHP
    
*   Python
    
*   Ruby
    
*   Rust
    

这些语言的内置`Execute`或者`Command`函数都或多或少会受到影响。

2、Rust CVE-2024-24576

Rust 也有这样的问题，所以进行了紧急修复，但是 Rust 一开始似乎是意识到了`.bat`的特异性行为，还给出了相关处理函数：

![](https://bbs.kanxue.com/upload/attach/202410/236762_7FJ8GVK4UJVKQA3.jpg)

然而它在处理的过程中，并未对双引号正确处理，而是同样使用了`\`：

![](https://bbs.kanxue.com/upload/attach/202410/236762_5SQT2PQNWMGFRNT.jpg)

在这边，错误的使用了`\\`作为过滤字符，所以同样导致了问题的出现。

这里参考[网上流传的 poc](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN)：

![](https://bbs.kanxue.com/upload/attach/202410/236762_W7M3EX6WJ7W2ZUD.jpg)

当我们传入`"&calc.exe`时候就能弹出计算器，此时观察命令行的参数可以看到如下的内容：

![](https://bbs.kanxue.com/upload/attach/202410/236762_ZP63Y7UDKWJWU5V.jpg)

Rust 给出的[修复在这边](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN)。经分析，可以知道主要是引入了函数`append_bat_arg`，在对各种字符串做了过滤之后，假设遇到双引号，则再次插入另一个，从而阻止绕过的发生：

![](https://bbs.kanxue.com/upload/attach/202410/236762_DCMS84U2CWUKEEW.jpg)

3、认知错误分析

实际上，这个漏洞本身和 Rust 关联不大，但是我们仍然可以用这个认知模型对这个漏洞进行分析：

*   开发人员认知：Windows 中，`\`与 Linux 下含义相同
    
*   实际运行环境：Windows 中的`^`与 Linux 下`\`语义相同
    

这种对于操作系统的认知差异导致了这个问题的出现。

四

案例二：对特性的认知错误

1、内容重排序问题

在[之前的文章中](http://mp.weixin.qq.com/s?__biz=Mzk0OTU2ODQ4Mw==&mid=2247485705&idx=1&sn=b10b583e944939d6c3a8e2b739817749&chksm=c3571f85f4209693b322eae9ec8308ac89a895674d1e067155eb0ef633a3ffdb75a8a7c11085&scene=21#wechat_redirect)提到过，Rust 的结构体的变量顺序可能会由于内存对齐问题进行重排序，这边简单复习一下，假设存在结构体：

![](https://bbs.kanxue.com/upload/attach/202410/236762_QUBV25GFHN5GCHQ.jpg)

上述结构体如果在 C 里面写的话，可以写作如下的形式：

![](https://bbs.kanxue.com/upload/tmp/236762_XT27SXNB3ZK3WXQ.jpg)

此时，这个结构体的大小是什么呢？

实际上，假设我们打印结构体的大小和偏移，会得到这个答案：

![](https://bbs.kanxue.com/upload/attach/202410/236762_CX7VFPW6F6SJKWR.jpg)

因为结构体对齐的时候，会遵顼三个原则：

*   第一个成员的起始地址为 0
    
*   每个成员的首地址为自身大小的整数倍
    
*   总大小为成员大小的整数倍
    

由于 b 的起始地址必须是 4 对齐，所以所有的变量都被迫进行了 4 字节对齐，从而形成了这个状态。

![](https://bbs.kanxue.com/upload/attach/202410/236762_6H5T5CUN84JK2WG.jpg)

那么这个结构体在 Rust 中的内存排布是如何的呢？

![](https://bbs.kanxue.com/upload/attach/202410/236762_35M7G9577AATCNX.jpg)

如果我们尝试打印他们的偏移的话，可以得到如下的结果：

![](https://bbs.kanxue.com/upload/attach/202410/236762_MXNSNHXN4QF9GYN.jpg)

从 IDA 中看，也可看到类似的结果：

![](https://bbs.kanxue.com/upload/attach/202410/236762_UAXEMG6URBSD4JW.jpg)

可以看到 field2 和 field3 的偏移发生了变化，其原因源自于之前提到过的对齐特性，Rust 会尽可能的缩小结构体大小，会因此调换结构体成员的变量顺序，从而保证结构体尽可能地小。

![](https://bbs.kanxue.com/upload/attach/202410/236762_RKV5ZTHAMND3HZX.jpg)

2、repr

然而在实际开发中，我们有时候不需要编译器对我们的结构体进行操作，此时可以通过声明 #[repr(C)] 关键字，强行让结构体排序不发生变化。

同时，我们可以看到，Rust 还是尽可能地保证了结构体在 4/8 字节上的对齐，然而在某些场景中，我们可能希望结构体能够尽可能地小，此时可以声明 #[repr(packed)] 来强行要求结构体不要保留 padding。这两种做法都是非常常见的。

3、RUSTSEC-2024-0346

这个漏洞出现在 Rust 下的一个 [zerovec](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN) 模块中，这个模块特点为零拷贝，本质上是对现有对象进行引用以及一些序列化相关的操作。

[根据文档](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN)，zerovec 的底层核心是一个叫做`ULE`的特征辅助实现的，其实现大致如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_7N8JXF36W6XNWXK.jpg)

这个特征要求利用 unsafe 的函数直接的获取对应的字节流是否有效。在这基础上，如果我们将要包含的数据大小是不定长的，则需要实现`VarULE`这一个特性：

![](https://bbs.kanxue.com/upload/attach/202410/236762_56GD29UNS8J2TND.jpg)

这个函数会要求对底层的数据进行一些操作从而完成进行数据拷贝，所以底层可以理解会存在【序列化】的过程。

该漏洞[提供的 POC](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN) 如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_AJBB3J3SQAGYTHH.jpg)

上述这段代码逻辑基本上就是做了一个时间格式化，然而在底层却触发了断言，导致了崩溃。经过开发者定位，最终确定有问题的数据结构如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_RQQ8F8KM999PR6D.jpg)

在这段代码中, 成员`dates_to_eras`对应的`ZeroVec`所关联的结构体就是 Tuple`(EraStartDate, TinyStr16)`，这两个结构体定义如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_XMYM9QV66PZNBRZ.jpg)

可以看到，`EraStartDate`为 8 字节，而`TinyStr16`为 16 字节。而在`ZeroVec`中，其使用了宏来处理这种情况：

![](https://bbs.kanxue.com/upload/attach/202410/236762_K4GH86Y836F4QBA.jpg)

可以看到，当遇到`tuple`类型的时候，`ZeroVec`会使用`packed`关键字将其封装，从而保证数据占用空间的大小不会太大。由于是 tuple，所以这里的结构体中不存在对应的实际成员，使用的时候只能使用`self.0`或者`self.1`来操作对应的`EraStartDate`和`TinyStr16`。然而，这里的 tuple 却并不如我们看到的那样排序，当加上`packed`关键字后，其会发生一个重排序的过程，例如下面代码：

![](https://bbs.kanxue.com/upload/attach/202410/236762_ZWEABCCFTB6UPB9.jpg)

实际运行起来的时候，我们会得到如下的结果：

![](https://bbs.kanxue.com/upload/attach/202410/236762_WFKYCK7NVXU3SEQ.jpg)

此时可以发现，`EraStartDate`和`TinyStr16`的顺序发生了颠倒。那么此时在使用`tuple`来操作对象的时候，原先位于`0`位置的`EraStartDate`就变成了`TinyStr16`，从而造成了漏洞的产生。

4、修复策略与认知差异分析

实际上，这个漏洞的修复非常简单，是需要将声明改成`#[repr(C,packed)]`即可强迫 Rust 使用 C 语言的内存排序对其进行严格的顺序声明，从而阻止这类漏洞的产生。

如果从认知差异的角度触发，这个漏洞其实就是一种非常典型的认知差异，表现为对语言特性理解的差异：

*   开发人员认知：Rust 中，`packed`字段未提及重排序，所以不会发生重排序问题
    
*   实际运行环境：Rust 中，`packed`字段不会决定排序，所以可能发生重排序
    

实际上，现在去 Rust 官网阅读文档也会发现，`packed`关键字明确提到了可能会发生重排序。然而开发者们在开发的过程中，依然可能存在记忆混淆等认知错误的问题，从而导致漏洞本身的出现。

五、案例三：对生命周期的错误认知

1、CVE-2024-27284

这个漏洞是 [Casandra-rs](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN) 中的问题。

由于没有 PoC，所以只能推测漏洞点的大致位置。

这个库是一个分布式数据库的 Rust 封装实现。由于数据库操作不可避免的要与数据库操作，所以有大量底层数据操作，因此引入了`unsafe`关键字，并且有很多迭代器存在，这个过程中就会导致漏洞的出现。

漏洞的关键点在于迭代器的误用，分析 patch 可以看到这样的逻辑：

![](https://bbs.kanxue.com/upload/attach/202410/236762_TR6867HRX6EU4UC.jpg)

这里有几个比较明显的修复特征：

*   Iterator 切换成了 LendingIterator
    
*   Item 定义有了微妙的变化，但是始终和`Raw<'a>`相关
    
*   `get_row`的返回值由`Row<'a>`换成了`Row`
    

我们这里可以全部过一遍这些修复点：首先是这里的`LendingIterator`，其实是其内部实现的一个特殊的数据结构：

![](https://bbs.kanxue.com/upload/attach/202410/236762_GQR6FHYWP3TMQGF.jpg)

这样声明后，迭代器的生命周期就会被强制与其`Item`指向的对象生命周期保持一致。

其次，这里提到了`Raw<'a>`定义如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_WDK9U9M2VD8Q7ZF.jpg)

可以看到，原先的`Row`关联的生命周期为`CassResult`，而新的`Row`关联的生命周期为`_Row`。

然后，这个`get_row`所操作的是一个`unsafe`对象，这个对象来自 cpp 部分：

![](https://bbs.kanxue.com/upload/attach/202410/236762_RNUEAE3DNXEKJPK.jpg)

`get_row`会返回一个`Row`对象，这个`Row`对象来自于`ResultIterator`这个结构体中定义的`Row`对象，此时我们可以知道，结构体关系如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_DQ8QEQ79AGH7VEM.jpg)

此时可以得出一个结论：

> `ResultIterator`和`Row`公用一段内存空间

同时，根据之前修改的代码，可以观察到这里修改：

![](https://bbs.kanxue.com/upload/attach/202410/236762_JSZAV9XXQRDH95T.jpg)

这两个迭代器虽然都关联了`Row<'a>`，但是这个对象的定义同样发生了变化：

![](https://bbs.kanxue.com/upload/attach/202410/236762_KVWQH5XFBGV6GZP.jpg)

原先的`Row<'a>`声明的时候，与`CassResult`关联，这个`CassResult`即为我们调用数据库查询功能后，能够得到的类型，而新版本的`Row<'a>`则是与`_Row`关联，这个`_Row`就是前文提到过的`Row`指针。同时，`next`函数就是迭代器在迭代过程中会自动调用获取下一个迭代对象的指针，如果我们罗列一下之前提到过的所有函数获得对象的关系，是这样的：

*   `get_result`能够获取`CassResult`
    
*   `CassResult.iter()` 能够获取 `ResultIterator`
    
*   `ResultIterator`在递归过程中，通过`next`获取`Row`
    

然而我们在修复前，`CassResult`和`Row`的生命周期一致，但是`ResultIterator`和`Row`对象未强制要求生命周期一致， 此时漏洞触发的原因就呼之欲出：

> 由于未强制关联`Row`与`ResultIterator`，而`ResultIterator`和`Row`共用一套内存，当`ResultIterator`被销毁，`Row`未被销毁的场合，就会引发漏洞

总结一下，PoC 形式如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_N235G65D5G9HKCF.jpg)

此时，由于`get_result`获取的`CassResult`未被销毁，此时对应的`Row<'a>`也就是`tmp_row`不会被 Rust 认为超出生命周期，然而此时的`result.iter()`获取的`ResultIterator`已经被销毁了，最终导致了 UAF 的产生。

我们根据上述的模型，写了一个类似的 POC，形式如下：

![](https://bbs.kanxue.com/upload/attach/202410/236762_M3SHY353ABERGJE.jpg)

此时能够成功的触发一个 UAF 问题

![](https://bbs.kanxue.com/upload/attach/202410/236762_2S2N8FMB9DM9BW7.jpg)

2、认知错误总结

从认知差异的角度触发，这个漏洞其实是一种基于逻辑错误而导致的内存问题。其虽然与`unsafe`关键字关联，但是实际上它从设计层面就出现了问题，概括来讲就是：

*   开发人员认知：`Row`与`Iterator`存放在同一内存中，两者可在同一时刻释放，不会发生内存问题；`Row`生命周期与`CassResult`关联，从而保证两者生命周期长度一致，防止内存泄漏；
    
*   实际运行环境：`Row`与`Iterator`可能不在同一个声明域中使用，可能存在`Iterator`提前释放的场景。
    

可以猜到，在开发的时候，开发者应该着重考虑了内存泄露的问题，并且假定迭代器创建的对象会被用户拷贝，抑或是保留在指定的生命周期中，然而实际开发过程中，错误的生命周期声明会导致检查的失效，从而导致 UAF 问题的出现。

六、总结

Rust 虽然是一个相当安全的语言，但是其安全范围是有限的，问题尤其会在人们错误的理解 Rust 提供的安全能力这种认知错误场合中出现。

根据我们前文的漏洞，可以总结出以下几种脱离了 Rust 防护机制的情形：

*   漏洞与操作系统底层关联，Rust 编译器无法感知
    
*   Rust 本身的特性导致的问题出现
    
*   开发者错误声明 Rust 生命周期的场合
    

通过对此类边界的观测，能够更加容易发现漏洞点，同样也能借此观测软件的防护情况， 加强软件防护。

七、参考链接

*   [The Rust Security Advisory Database](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN)
    
*   [BatBadBut: You can't securely execute commands on Windows](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN)
    
*   [Rust 下的二进制漏洞 CVE-2024-27284 分析](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458579888&idx=1&sn=6ee6350ecda2740e1d51c63b9fc1288c&chksm=b18dc33a86fa4a2c4252ae8a5dd9f7349fd6d565b31624b58ca9c70c88e111492c2a2022df2d&token=2024854501&lang=zh_CN)
    

PPT 及回放视频

峰会议题 PPT 及回放视频已上传至【看雪课程】：_https://www.kanxue.com/book-leaflet-195.htm_

_![](https://bbs.kanxue.com/upload/attach/202410/236762_MH3PE7JMAEPUP28.jpg)_

已购票的参会人员可免费获取，主办方已通过短信将 “兑换码” 发至手机，按提示兑换即可~

[[注意] 传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)