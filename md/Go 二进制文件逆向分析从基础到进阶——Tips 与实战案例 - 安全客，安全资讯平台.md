> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/218674)

> 书接前文，本文介绍 Go 语言二进制文件逆向的几个 Tips，最后用实际案例演示一个 Go 二进制文件逆向分析的进阶玩法——还原复杂数据结构的 Go 语言定义。

[![](https://p5.ssl.qhimg.com/t01a10c24a01079c7d3.jpg)](https://p5.ssl.qhimg.com/t01a10c24a01079c7d3.jpg)

> 书接前文，本文介绍 Go 语言二进制文件逆向的几个 Tips，最后用实际案例演示一个 Go 二进制文件逆向分析的进阶玩法——还原复杂数据结构的 Go 语言定义。

**传送门**：

1.  [Go 二进制文件逆向分析从基础到进阶——综述](https://www.anquanke.com/post/id/214940)
2.  [Go 二进制文件逆向分析从基础到进阶——MetaInfo、函数符号和源码文件路径列表](https://www.anquanke.com/post/id/215419)
3.  [Go 二进制文件逆向分析从基础到进阶——数据类型](https://www.anquanke.com/post/id/215820)
4.  [Go 二进制文件逆向分析从基础到进阶——itab 和 strings](https://www.anquanke.com/post/id/218377)

11. 逆向分析 Tips
-------------

### 11.1 函数的参数与返回值

本系列第一篇《[Go 二进制文件逆向分析从基础到进阶——综述](https://www.anquanke.com/post/id/214940)》中提到过，Go 语言有自己独特的调用约定和栈管理机制，使 C/C++ 二进制文件逆向分析的经验在这里力不从心：Go 语言用的是 [continue stack 栈管理机制](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.5.html) ，并且 Go 语言函数中 callee 的栈空间由上层调用函数 caller 来维护，callee 的参数、返回值都由 caller 在栈中预留空间，**传入参数和返回值都是通过栈空间里的内存**，就难以直观看出哪个是参数、哪个是返回值。详见 [The Go low-level calling convention on x86-64](https://dr-knz.net/go-calling-convention-x86-64.html) 。Go 中函数内存空间布局如下：

[![](https://p4.ssl.qhimg.com/t0142b0c3faed3a5c78.png)](https://p4.ssl.qhimg.com/t0142b0c3faed3a5c78.png)

那么，在 IDAPro 中打开某个函数的反汇编代码看一看，函数头部一堆 **args** 中哪些是传入的参数，哪些又是返回的值呢？

一个简单粗暴的办法，是先去函数末尾找有没有返回值，确定了返回值，剩下的 **args** 就是需要传入的参数。比如下面这个函数：

[![](https://p4.ssl.qhimg.com/t019b00d75037e1e308.png)](https://p4.ssl.qhimg.com/t019b00d75037e1e308.png)

可以看到，函数头中显示共有 6 个 **arg_*** 的变量，而函数尾部显示最后的 **arg_20** 和 **arg_28** 分别被传入两个值以后，函数返回。据此可以确定最后两个 **arg** 就是函数的返回值，而前面 4 个 **arg** 就是函数需要传入的参数。其实，用 IDAPro 打开 Go 二进制文件中的函数来分析，如果函数有返回值的话，返回值总是最后面的 **arg**，即距离栈底较近的位置。

### 11.2 入口函数与初始化函数

通常，在 IDAPro 中打开一个可执行二进制文件，IDAPro 的自动化分析过程结束后，会自动跳转到一个入口函数的位置，等待我们分析。对于 Go 二进制文件来说，64bit PE 文件通常会自动跳转到 `_rt0_amd64_windows` 函数：

[![](https://p0.ssl.qhimg.com/t0144ae89823e07f2bc.png)](https://p0.ssl.qhimg.com/t0144ae89823e07f2bc.png)

64bit ELF 文件，通常会自动跳转到 `_rt0_amd64_linux` 函数：

[![](https://p0.ssl.qhimg.com/t01024077e337fb1f09.png)](https://p0.ssl.qhimg.com/t01024077e337fb1f09.png)

但是，千万不要上来就从这样的函数开始分析，因为它们只不过是 Go runtime 的入口函数而已。**真正的 Go 语言程序逻辑的入口，其实是 `main.main()` 函数**。

那么，上手就开始分析 `main.main()` 函数就一定 OK 了吗？并不是。因为 Go 语言规范中还有个特殊的函数，自己实现之后会在 `main()` 函数之前就执行—— **init()** 函数。

**init()** 函数一般用来初始化一些全局设置，详细作用如下：

*   全局变量初始化
*   检查和修复程序状态
*   运行前注册，例如 decoder，parser 的注册
*   运行只需计算一次的模块，像 sync.once 的作用，或全局数据库连接句柄的初始化等等。

**init()** 函数的特性有以下几个：

*   init 函数先于 main 函数自动执行，不能被其他函数调用；
*   init 函数没有输入参数、返回值；
*   每个包可以有多个 init 函数；
*   **包的每个源文件也可以有多个 init 函数**，这点比较特殊；
*   不同包的 init 函数按照包导入的依赖关系决定该初始化函数的执行顺序；
*   init 函数在 main 函数执行之前，由 runtime 来调用。

举个栗子，在《[DDG 的新征程——自研 P2P 协议构建混合 P2P 网络](https://blog.netlab.360.com/ddg-upgrade-to-new-p2p-hybrid-model/)》一文中，可以看到 DDG 木马样本里，就通过多个 **init()** 函数实现了不同的初始化设置：

1.  在函数 `ddgs_common_init()` 中，DDG 基于**自定义的 Alphabet** 创建了一个**全局的 Base64 编解码句柄**：[![](https://blog.netlab.360.com/content/images/2020/04/b64Enc-1.png)](https://blog.netlab.360.com/content/images/2020/04/b64Enc-1.png)
2.  在函数 `ddgs_global_init()` 中，DDG 还有另外一项关键全局变量的初始化工作：创建一对全局 **ed25519** 密钥，用以在后续存取 BoltDB 中数据时对数据进行签名和校验；
3.  在全局初始化函数 `ddgs_global_init()` 中，DDG 调用了一个函数 `ddgs_global_decodePasswords` ，在这个函数中解密并校验内置的弱口令字典，这些弱口令将在后续传播阶段被用来暴破 SSH 服务；
4.  在函数 `ddgs_xnet_init_0()` 中，DDG 解析了内置的 **xhub** 和 **xSeeds** 数据，还对 **xhub** 数据用 **ed25519** 算法校验是否被伪造。

由此可见，上手分析一个 Go 二进制文件时，应该先看它里面有没有实现 `init()` 函数，然后再分析它的 `main()` 函数。但是，也不能就只看 `main.init()` 函数，因为上文说过了，一个 Go 项目中可能用到了不同的 Package，每个 Package 甚至每个源码文件都可以实现自己的 `init()` 函数，而这些被导入的 Package 中的 `init()` 函数，会在 `main.init()` 之前被调用执行。当然，并不是所有的 Package 中的 `init()` 函数都需要分析，需要分析的一般是指二进制文件里导入的**私有 Package** 中的 `init()` 函数。

再举个栗子，最新的 DDG v5031 样本中，Malware 作者自己实现的 `init()` 函数如下：

```
ddgs_cmd_init
ddgs_common_init
ddgs_common_init_0
ddgs_common_init_0_func1
ddgs_global_init
ddgs_slave_jobs_init
ddgs_slave_jobs_init_0
ddgs_slave_lib__lockfile_init
ddgs_slave_lib__lockfile_init_func1
ddgs_slave_lib_init
ddgs_xnet_init
ddgs_xnet_init_0


```

这些函数都会在 `main.init()` 函数之前执行，当然也会早于 `main.main()` 函数执行。由于它们都是被 Go runtime 来调用执行的，在样本的主逻辑中并不会被调用。所以务必在分析样本主逻辑之前，先分析它们。

### 11.3 GC write barrier

Golang 内部有比较高效的垃圾回收（GC）机制，这个机制在逆向分析 Golang 二进制文件的过程中，会经常遇到。Golang 中的内存写操作，会先通知 GC，在 GC 中为目的内存做标记。通知 GC 的方式，是在内存写操作之前检查 `runtime.writeBarrier.enbaled` 标志位，如果设定了这个标志位，就会通过 `runtime.gcWriteBarrier` 来进行内存写操作。

反映到 IDAPro 中，就可以经常看到类似这样的 CFG（Control Flow Graph）：

[![](https://p3.ssl.qhimg.com/t011a5a20bf685edf08.png)](https://p3.ssl.qhimg.com/t011a5a20bf685edf08.png)

上图的流程，是把 `runtime.makeslice()` 生成的 slice (**var_78**) 的地址，写入 `[rax+20h]` 中。在 64bit Go 二进制文件中，调用 `runtime.gcWriteBarrier()` 时，总是把目标地址放入 `rdi` ，把源地址 (值) 放入 `rax`。

逆向分析的时候遇到这种代码块，知道它只是 runtime GC Barrier 相关的常规操作，就不会再困惑了。

### 11.4 runtime.growslice()

在 Go 语言中，**切片 (Slice)** 是一种特殊的数据类型。Go 中数组类型 (Array) 的长度不可改变，在特定场景中这样的类型就不太适用，所以 Go 就以 “动态数组” 的概念提供了一个类似数组，但可灵活伸缩的数据结构——切片。每当一个切片类型的数据在进行 **Append** 操作时如果需要动态增长（扩容），就会在底层由 runtime 调用 **runtime.growslice()** 来调整切片的内部空间。

**runtime.growslice()** 的函数声明如下：

```
func growslice(et *_type, old slice, cap int) slice


```

即该函数会在调整旧 Slice 内部元素之后，返回一个基于旧 Slice 扩容后的新 Slice。这种操作在 Go 二进制文件中，只要用到 Slice 类型数据的地方，就有可能碰到。下图是一个比较典型的情形：

[![](https://p4.ssl.qhimg.com/t01c4e5b3335729d121.png)](https://p4.ssl.qhimg.com/t01c4e5b3335729d121.png)

在逆向分析过程中，遇到这种带 `runtime.growslice()` 的代码块，知道这是在 Slice 容量不足的情况下进行的扩容操作即可。这块代码不一定会执行，粗暴一些甚至可以直接忽略，直接去看这块代码前后的代码，这块代码前后的代码，才是业务逻辑中真实操作 Slice 中元素的代码。

### 11.5 调试

本系列大部分内容讲的都是静态分析相关的知识点，但静态分析不是万能的。有些时候碰上复杂的代码逻辑，静态分析可能会卡在某个关键点上，比如自定义加密逻辑中的密钥换算逻辑无法理解透彻，需要做一下动态调试，用调试器中跑出来的真实结果来验证自己静态分析得出的结论。甚至有的师傅喜欢调试甚于静态分析，拿到二进制文件就想直接扔到调试器中跑一下看看。

网上搜 Go debug，绝大部分资料跟 **[dlv](https://github.com/go-delve/delve)** 调试工具和 **[GDB 的 Go 调试插件](https://golang.org/doc/gdb)**有关。但是这两个工具的使用有一个前提：需要有调试符号甚至 Go 项目的源码。它们是给常规的 Go 语言开发者使用的。然而我们要分析的 Go 恶意软件，绝大部分是没有调试符号的，strip 处理的很干净。

没有符号就不能调试了吗？不存在的，有困难也要上。好在前面讲的静态分析的姿势，可以让我们通过静态分析就掌握足够的信息，把没符号的 Go 二进制文件扔调试器里，也可以调。

静态分析的姿势掌握之后，Go 二进制文件调试面临的第一个、甚至是唯一一个问题，就是**断点**。

常规用 C/C++ 编写的二进制文件，里面没有 runtime，我们可以方便地加载到调试器中，让调试器在 `main()` 函数处断下来。而 Go 二进制文件做不到这样。以 ELF 文件为例，对于个别旧版本的 Go 二进制文件，我们可以加载到 GDB 中，直接 用以下命令断下：

```
> b main
> run


```

此时，断下来的位置可不是 Go 中的 `main.main()` 函数，而是 cgo 的入口函数 `__libc_start_main` ，断下之后，就可以根据 ELF 加载的地址，在 Go 程序 package main 的入口 main 函数**地址处**下断点。

对于较新版本的 Go 二进制文件，上述做法就不起作用了，因为新的 Go 自己实现的构建工具链不再用 cgo 那套东西了。这种情况只能直接在想要断下来的地址处直接给地址下断点。

另外需要注意的是，Go 语言 runtime 内部通过协程来实现高并发。在调试器中可以看到代码在不同的函数中跳来跳去，所以断点务必要打准，不然很容易就被 Go runtime 的协程调度给搞蒙了。

12. 进阶案例实战演示
------------

对于 Go 二进制文件的逆向分析，恢复函数符号、解析字符串，本系列五篇文章看下来对比衡量一下，只能算是基础姿势；对于上面列出来的几个 Tips，以及一些其他没列出的细节，也只能算是 Go 二进制文件一些特有的小知识点；而涉及加解密、内存执行、复杂协议分析等方面，其他类型的二进制文件中也会涉及，只能算是普适性的进阶逆向内容。

但是，进阶的逆向分析姿势中，Go 二进制文件还有一个独特的方面：**复杂数据结构解析**。

根据前文的内容可以知道，Go 语言中可以通过 **Struct** 来定义复杂的数据结构，通常可以跟一个 JSON 数据结构相互转换，即它可以定义一个结构复杂的 JSON 数据。而 Go 还允许为自定义数据类型绑定一些方法，使 Struct 这个数据类型实际上拥有了面向对象概念中的 **类** 的特性，这就更进一步增加了数据类型的复杂程度。

好在，这些复杂数据类型的定义，都包含在了 Go 二进制文件中。我们可以通过本系列第三篇《[Go 二进制文件逆向分析从基础到进阶——数据类型](https://www.anquanke.com/post/id/215820)》中的内容来将这些复杂数据类型的定义解析、还原出来。

下文就结合 **[go_parser](https://github.com/0xjiayu/go_parser)** ，以 DDG 样本中的一个最新版关键配置数据 **slave configure** 的解析，来演示一下这种逆向过程。

### 12.1 DDG 中的 Slave Configure

DDG 是一个专注于扫描控制 SSH 、 Redis 数据库 和 OrientDB 数据库服务器，并攫取服务器算力挖矿（门罗币）的僵尸网络，后来的版本升级中，DDG 还加入了对 Supervisord 和 Nexus 的漏洞利用来传播自身。从 2019 年 11 月，DDG 又[大幅更新一波](https://blog.netlab.360.com/ddg-upgrade-to-new-p2p-hybrid-model/)，新增自研 P2P 协议，把自己打造成了一个 P2P 结构的挖矿僵尸网络。

DDG 在 v3010 版本的时候，**[新增了一份结构复杂的配置文件](https://blog.netlab.360.com/ddg-mining-botnet-jin-qi-huo-dong-fen-xi/)**：**slave**。这份配置文件由 **[msgpack](https://msgpack.org/)** 编码 (可以简单粗暴理解为压缩版的 JSON 数据通用序列化编码方案)，由 Bot 向 C2 发一个请求后 C2 返回给 Bot。这份配置文件中指定了矿机程序的下载地址、另存为 Path 和 MD5；还指定了要启用的传播模块以及相关配置属性，比如是要内网传播还是公网传播、要爆破的 SSH 服务的端口列表，以及拿下肉鸡后要执行的恶意 Shell 脚本的下载链接等等。最后，还专门为这份配置数据加了一个签名，以防被别人篡改。这份配置文件有自己的版本号，其内部结构至今已经更改多次，在最新的 DDG v5031 版本中仍然在用。

这份配置文件的旧版，我曾在开源旧版 DDG Botnet Tracker 时，把它解码后传到了 Github。限于篇幅，此处仅贴出 URL：

[https://github.com/0xjiayu/DDGBotnetTracker/blob/master/v2/data/slave.json](https://github.com/0xjiayu/DDGBotnetTracker/blob/master/v2/data/slave.json)

### 12.2 逆向分析

对 **slave** 配置文件的操作，都集中在 DDG 样本的 **main.pingpong()** 函数中。这个函数有一个参数是 C&C 的地址，它首先向 `http://<C&C>/slave` 发一个 POST 请求 (HTTP Request Header 的设置此处略去)，然后等待 C&C 回复：

[![](https://p4.ssl.qhimg.com/t01e3488a42cc05ac0a.png)](https://p4.ssl.qhimg.com/t01e3488a42cc05ac0a.png)

收到 C&C 的响应之后，该函数继续执行以下逻辑：

[![](https://p1.ssl.qhimg.com/t01f0c61e2c6fbc06e7.png)](https://p1.ssl.qhimg.com/t01f0c61e2c6fbc06e7.png)

上图左边和右上方的代码块，显示该函数先用 **common.SignData** 这个 Struct 类型来对 C&C 返回的数据执行解码操作。然后右下方的代码块又用 **main.Payload** 这个 Struct 类型，对前面解码出来的部分数据进行**第二层 msgpack 解码**操作。

经过 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析，可以看到 **common.SignData** 这个类型的定义：

[![](https://p1.ssl.qhimg.com/t01e819abd626d49bfe.png)](https://p1.ssl.qhimg.com/t01e819abd626d49bfe.png)

可以看到它包含两个字段：`[]uint8(即 []byte)` 类型的 **Signature**，和 `[]byte` 类型的 **Data**。Go 语言定义表示如下：

```
package common
type SignData struct{
    Signature []byte
    Data      []byte
}


```

其实，下面进行第二层 **msgpack** 解码的，正式这个 **Data** 字段的内容。即 Slave 第一层编码是用来校验配置数据的 Signature，和被编码过的配置数据的字节切片，后面真正的配置数据还要再经过一层 **msgpack** 解码。第二层解码真正的配置数据 **main.Payload** ，是在函数 `ddgs_common__SignData_Decode()` 中进行的，该函数很简单，先校验数据，后用 **msgpack** 解码（此处略去校验过程）：

[![](https://p1.ssl.qhimg.com/t013a095f7c5fb96441.png)](https://p1.ssl.qhimg.com/t013a095f7c5fb96441.png)

而这个 **main.Payload** Struct 类型的定义，正是我在 **[go_parser](https://github.com/0xjiayu/go_parser)** 开源项目 README 中展示的那个效果图：

[![](https://p4.ssl.qhimg.com/t018d88777b2faefb07.png)](https://p4.ssl.qhimg.com/t018d88777b2faefb07.png)

当然，上面效果图只是 **main.Payload** Struct 定义的一部分内容，通过 **[go_parser](https://github.com/0xjiayu/go_parser)** 的解析，我们可以很方便地在 IDAPro 中，从该数据类型定义的起始位置开始，通过**双击跳转**到特定位置的操作，一步一步深入查看每一层、每一个字段的定义。完整的结构我梳理如下：

[![](https://p5.ssl.qhimg.com/t01a1656454700f7dff.png)](https://p5.ssl.qhimg.com/t01a1656454700f7dff.png)

### 12.3 尝试解码数据

看到上面梳理出来的数据结构定义图，先不要急着用 Go 语言去把它写出来并解码数据。前方有坑，出手需谨慎。不妨先大概看一下这个 **Payload** 里到底是什么数据，这份数据的结构能不能与上面的结构图对的上。

上文说过，**Payload** 数据又加了一层 **msgpack** 编码，如果要用 Go 代码来反序列化解码数据的话，必须要逆向、恢复出一个准确的数据结构定义，拿这个准确定数据结构定义去解码数据。这在 Go 语言二进制文件逆向的数据反序列化解码、解密方面，是一个常见的难题（参考 Go 语言中 JSON 数据的操作姿势）。

好在 **msgpack** 是一个**通用**的序列化编码方案，它提供了很多种编程语言的 API。所以同一份 **msgpack** 序列化编码过的数据，用 Go 语言可解，用其他语言也可以解。而用 Python 这种动态类型的高级语言来解的话，有一个好处是**不需要提前知道数据的结构定义**。下面我们先看一下如何用 Python 解码这份数据。

> **NOTE:**
> 
> 最新的 slave configure 原始数据我已上传到 Github：
> 
> [https://github.com/0xjiayu/DDGBotnetTracker/blob/master/v2/data/slave_latest.raw](https://github.com/0xjiayu/DDGBotnetTracker/blob/master/v2/data/slave_latest.raw)

```
In [1]: import msgpack

In [2]: import hexdump

In [3]: f = open("slave_latest.raw", "rb")

In [4]: fcntnt = f.read()

In [5]: f.close()

In [6]: signdata = msgpack.unpackb(fcntnt, use_list=True, raw=True)

In [7]: hexdump.hexdump(signdata[b'Signature'])
00000000: 5B E6 C3 7C B7 61 98 01  42 35 BB 38 33 7B DC B4  [..|.a..B5.83{..
00000010: 11 9E B8 31 E7 DD C0 5B  C7 EE 20 A4 B7 60 2C A1  ...1...[.. ..`,.
00000020: EE 99 78 5E 58 BD 06 57  3A BB 6F 3D CA 1A FA 15  ..x^X..W:.o=....
00000030: 3B 30 B5 26 28 99 BB 72  67 11 47 12 07 F3 2F CD  ;0.&(..rg.G.../.
00000040: 08 AE 67 09 C3 26 14 8B  47 96 20 76 87 E8 4A 16  ..g..&..G. v..J.
00000050: 4A 25 1F 68 0D EA D6 97  BD 07 A3 19 6D C4 8B AC  J%.h........m...
00000060: B2 71 21 F5 BC 7D 1D EB  93 0A 62 C3 6B 6D C8 89  .q!..}....b.km..
00000070: 9E BD A2 47 C9 08 44 8A  02 FA 06 1C 3F 7D 6A C7  ...G..D.....?}j.
00000080: D6 EA 15 20 41 C1 C7 B8  A5 E8 57 B3 89 4B 5C 9F  ... A.....W..K\.
00000090: 30 CA 32 B8 3F EE EA AC  D3 DB EB 98 3C A4 9A 76  0.2.?.......<..v
000000A0: CE F8 CF D2 CE 3D 4D 94  6C 84 8B D8 60 3A 32 2C  .....=M.l...`:2,
000000B0: F7 03 E0 52 CC B0 64 A2  FB 8F 12 11 96 01 10 78  ...R..d........x
000000C0: 08 FD 28 62 37 AB 6F 5E  F6 39 B2 53 51 8F FB 8E  ..(b7.o^.9.SQ...
000000D0: 98 9B CA FA 93 8A 9F F9  08 DE A9 D6 A6 7E 9E 10  .............~..
000000E0: B2 29 32 D7 5E 54 AC FB  59 0C C7 C0 41 E4 33 16  .)2.^T..Y...A.3.
000000F0: C3 C7 9F 63 D6 47 13 AC  A0 4D F2 02 44 DC E5 D7  ...c.G...M..D...

In [8]: msgpack.unpackb(signdata[b'Data'], use_list=True)
Out[8]:
{'CfgVer': 72,
 'Config': {'Interval': '60s'},
 'Miner': [{'Exe': '/usr/bin/joseph',
   'Md5': '39dffed3c43ef9b15a6aa2f4d1ab4b4e',
   'Url': '/static/xmrig0906'},
  {'Exe': '/tmp/joseph',
   'Md5': '39dffed3c43ef9b15a6aa2f4d1ab4b4e',
   'Url': '/static/xmrig0906'}],
 'Cmd': {'AALocalSSH': {'Id': 3053,
   'Version': 5031,
   'UrlPath': '/static/5031/ddgs',
   'Timeout': '2m'},
  'AAssh': {'Id': 2174,
   'Version': 5031,
   'UrlPath': '/static/5031/ddgs',
   'NThreads': 100,
   'Duration': '480h',
   'IPDuration': '12h',
   'GenLan': True,
   'GenAAA': False,
   'Timeout': '1m',
   'Ports': [22, 1987, 2222, 22222, 12222]}}}


```

### 12.4 还原复杂数据结构的 Go 语言定义

如果单纯是为了解码这份配置数据，其实到上一步就可以达到目的了，没有必要非用 Go 实现一个解码程序。但如果是为了写一套 Botnet 追踪程序来追踪 DDG，那还是要用 Go 来写程序把这份数据解码的。

对比上面 **msgpack** for Python 的解码结果，和根据 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析的结果从 IDAPro 中梳理出来的数据结构定义，可以发现他们之间有一些出入：

1.  真实的数据结构，层次并没有从 IDAPro 中梳理出来的数据结构定义的那么多，比如数据结构定义显示 `AAssh` 下面有 3 个字段：Base/AAOptions/UrlPath，而真实的数据中结构更扁平，取消了 Base 和 AAOptions 这一层，并把它们里面的字段直接全都融合到了 `AAssh` 这一层；真实数据中也没有 `XRun` 这一层结构……
2.  并不是 IDAPro 中定义的每个字段，都在真实的数据中存在的，比如真实解码出来的数据中 `Cmd.AALocalSSH` 里的字段就比 `Cmd.AAssh` 里的字段数少了几个；
3.  甚至具体到字段的类型也有冲突：数据结构定义显示，`Cmd.Timeout` 字段和`Cmd.AAOptions` 中的 `IPDuration` 和 `Duration` 字段都是 `Cmd.timeout` 类型，而 `Cmd.timeout` 类型底层是 `time.Duration` 类型，在标准库 `package time` 的文档中，**[`time.Duration` 的定义](https://golang.org/pkg/time/?m=all#Duration)**如下：
    
    ```
    type Duration int64
    const (
        minDuration Duration = -1 << 63
        maxDuration Duration = 1<<63 - 1
    )
    
    
    ```
    
    可以看到它们的底层数据的类型都应该是 **int64**，而真实解码出来的数据中，这三个字段的值都是字符串类型：
    
    ```
    'Duration': '480h'
    'IPDuration': '12h'
    'Timeout': '2m'
    
    
    ```
    

第一个问题，可以用 Go 语言中 Struct 类型的特性来解释。Struct 支持嵌套定义，嵌套定义一个复杂 Struct 类型的时候，支持匿名字段，也可以叫做**内嵌**，即不为这个字段指定 Field Name，而直接指定一个类型名。这样以来，该类型中如果包含多个字段，就可以把该类型中的字段直接内嵌到上层 Struct 中。详情参考： 《**[Go 语言匿名字段和内嵌结构体](https://doc.yonyoucloud.com/doc/wiki/project/the-way-to-go/10.5.html)**》 。

第二个问题，就得归功于 **msgpack** 的特性了。在 Go 语言中，对于需要 **msgpack** 编解码的 Struct，可以在其第一行用以下方式注明，该 Struct 中的字段可能不会全部被设置：

```
_msgpack struct{} `msgpack:",omitempty"`


```

这种操作跟 Go 中操作 JSON 数据类似。另外，**对于类型不明或者可变的字段，类型也可以直接指定为空接口**。

第三个问题，就需要我们写代码的时候把那几个字段强行设置为 string 类型了，而不是内嵌 **time.Duration** 的 timeout 类型。然后用 **[time.ParseDuration(string)](https://golang.org/pkg/time/?m=all#ParseDuration)** 方法来解析字符串类型的 Duration 值。

综合一下，可以用以下方式来定义这份数据的类型结构：

```
type SignData struct {
    Signature []byte
    Data      []byte
}

type Paylaod struct {
    CfgVer int
    Miners []miner  `msgpack:"Miner"`
    Cmds   CmdTable `msgpack:"Cmd"`
}

type IntervalConf struct {
    Interval string
}

type CmdTable struct {
    _msgpack      struct{} `msgpack:",omitempty"`
    AALocalSSH    AAsshSt
    AAredis       AAredisSt
    AAssh         AAsshSt
    AAnexus       AAnexusSt
    AAsupervisord AAsupervisordSt
}

type miner struct {
    XRun
}

type XRun struct {
    Exe string
    Url string
    Md5 string
}

type AAsshSt struct {
    _msgpack struct{} `msgpack:",omitempty"`
    BaseOptions
    AAOptions
    UrlPath string
}

type AAredisSt struct {
    _msgpack struct{} `msgpack:",omitempty"`
    BaseOptions
    AAOptions
    ShellUrl string
}

type AAnexusSt struct {
    _msgpack struct{} `msgpack:",omitempty"`
    BaseOptions
    AAOptions
    ShellUrl string
}

type AAsupervisordSt struct {
    _msgpack struct{} `msgpack:",omitempty"`
    BaseOptions
    AAOptions
    ShellUrl string
}

type AALocalSSHSt struct {
    _msgpack struct{} `msgpack:",omitempty"`
    BaseOptions
    AAOptions
    UrlPath  string
    ShellUrl string
}

type BaseOptions struct {
    _msgpack struct{} `msgpack:",omitempty"`
    Id       int
    Version  int
    Timeout  string
    Result   result
    StartAt  int64
    handler  *func()
}

type AAOptions struct {
    _msgpack   struct{} `msgpack:",omitempty"`
    NThreads   int
    Duration   string
    IPDuration string
    GenLan     bool
    GenAAA     bool
    Ports      []int
}

type result struct {
    Map sync.Map
}


```

完整的代码已上传 Github：

[https://github.com/0xjiayu/DDGBotnetTracker/blob/master/v2/tools/slave_dec.go](https://github.com/0xjiayu/DDGBotnetTracker/blob/master/v2/tools/slave_dec.go)

运行上述 Go 程序，结果如下：

```
➜ go run slave_dec.go
Slave Config Signature:
------------------------------------------------------------------------------
00000000  5b e6 c3 7c b7 61 98 01  42 35 bb 38 33 7b dc b4  |[..|.a..B5.83{..|
00000010  11 9e b8 31 e7 dd c0 5b  c7 ee 20 a4 b7 60 2c a1  |...1...[.. ..`,.|
00000020  ee 99 78 5e 58 bd 06 57  3a bb 6f 3d ca 1a fa 15  |..x^X..W:.o=....|
00000030  3b 30 b5 26 28 99 bb 72  67 11 47 12 07 f3 2f cd  |;0.&(..rg.G.../.|
00000040  08 ae 67 09 c3 26 14 8b  47 96 20 76 87 e8 4a 16  |..g..&..G. v..J.|
00000050  4a 25 1f 68 0d ea d6 97  bd 07 a3 19 6d c4 8b ac  |J%.h........m...|
00000060  b2 71 21 f5 bc 7d 1d eb  93 0a 62 c3 6b 6d c8 89  |.q!..}....b.km..|
00000070  9e bd a2 47 c9 08 44 8a  02 fa 06 1c 3f 7d 6a c7  |...G..D.....?}j.|
00000080  d6 ea 15 20 41 c1 c7 b8  a5 e8 57 b3 89 4b 5c 9f  |... A.....W..K\.|
00000090  30 ca 32 b8 3f ee ea ac  d3 db eb 98 3c a4 9a 76  |0.2.?.......<..v|
000000a0  ce f8 cf d2 ce 3d 4d 94  6c 84 8b d8 60 3a 32 2c  |.....=M.l...`:2,|
000000b0  f7 03 e0 52 cc b0 64 a2  fb 8f 12 11 96 01 10 78  |...R..d........x|
000000c0  08 fd 28 62 37 ab 6f 5e  f6 39 b2 53 51 8f fb 8e  |..(b7.o^.9.SQ...|
000000d0  98 9b ca fa 93 8a 9f f9  08 de a9 d6 a6 7e 9e 10  |.............~..|
000000e0  b2 29 32 d7 5e 54 ac fb  59 0c c7 c0 41 e4 33 16  |.)2.^T..Y...A.3.|
000000f0  c3 c7 9f 63 d6 47 13 ac  a0 4d f2 02 44 dc e5 d7  |...c.G...M..D...|

Slave Config Raw Payload Data:
------------------------------------------------------------------------------
00000000  84 a6 43 66 67 56 65 72  48 a6 43 6f 6e 66 69 67  |..CfgVerH.Config|
00000010  81 a8 49 6e 74 65 72 76  61 6c a3 36 30 73 a5 4d  |..Interval.60s.M|
00000020  69 6e 65 72 92 83 a3 45  78 65 af 2f 75 73 72 2f  |iner...Exe./usr/|
00000030  62 69 6e 2f 6a 6f 73 65  70 68 a3 4d 64 35 d9 20  |bin/joseph.Md5. |
00000040  33 39 64 66 66 65 64 33  63 34 33 65 66 39 62 31  |39dffed3c43ef9b1|
00000050  35 61 36 61 61 32 66 34  64 31 61 62 34 62 34 65  |5a6aa2f4d1ab4b4e|
00000060  a3 55 72 6c b1 2f 73 74  61 74 69 63 2f 78 6d 72  |.Url./static/xmr|
00000070  69 67 30 39 30 36 83 a3  45 78 65 ab 2f 74 6d 70  |ig0906..Exe./tmp|
00000080  2f 6a 6f 73 65 70 68 a3  4d 64 35 d9 20 33 39 64  |/joseph.Md5. 39d|
00000090  66 66 65 64 33 63 34 33  65 66 39 62 31 35 61 36  |ffed3c43ef9b15a6|
000000a0  61 61 32 66 34 64 31 61  62 34 62 34 65 a3 55 72  |aa2f4d1ab4b4e.Ur|
000000b0  6c b1 2f 73 74 61 74 69  63 2f 78 6d 72 69 67 30  |l./static/xmrig0|
000000c0  39 30 36 a3 43 6d 64 82  aa 41 41 4c 6f 63 61 6c  |906.Cmd..AALocal|
000000d0  53 53 48 84 a2 49 64 cd  0b ed a7 56 65 72 73 69  |SSH..Id....Versi|
000000e0  6f 6e cd 13 a7 a7 55 72  6c 50 61 74 68 b1 2f 73  |on....UrlPath./s|
000000f0  74 61 74 69 63 2f 35 30  33 31 2f 64 64 67 73 a7  |tatic/5031/ddgs.|
00000100  54 69 6d 65 6f 75 74 a2  32 6d a5 41 41 73 73 68  |Timeout.2m.AAssh|
00000110  8a a2 49 64 cd 08 7e a7  56 65 72 73 69 6f 6e cd  |..Id..~.Version.|
00000120  13 a7 a7 55 72 6c 50 61  74 68 b1 2f 73 74 61 74  |...UrlPath./stat|
00000130  69 63 2f 35 30 33 31 2f  64 64 67 73 a8 4e 54 68  |ic/5031/ddgs.NTh|
00000140  72 65 61 64 73 64 a8 44  75 72 61 74 69 6f 6e a4  |readsd.Duration.|
00000150  34 38 30 68 aa 49 50 44  75 72 61 74 69 6f 6e a3  |480h.IPDuration.|
00000160  31 32 68 a6 47 65 6e 4c  61 6e c3 a6 47 65 6e 41  |12h.GenLan..GenA|
00000170  41 41 c2 a7 54 69 6d 65  6f 75 74 a2 31 6d a5 50  |AA..Timeout.1m.P|
00000180  6f 72 74 73 95 16 cd 07  c3 cd 08 ae cd 56 ce cd  |orts.........V..|
00000190  2f be                                             |/.|

Payload Config Plain Text:
------------------------------------------------------------------------------
{
    "CfgVer": 72,
    "Miners": [
        {
            "Exe": "/usr/bin/joseph",
            "Url": "/static/xmrig0906",
            "Md5": "39dffed3c43ef9b15a6aa2f4d1ab4b4e"
        },
        {
            "Exe": "/tmp/joseph",
            "Url": "/static/xmrig0906",
            "Md5": "39dffed3c43ef9b15a6aa2f4d1ab4b4e"
        }
    ],
    "Cmds": {
        "AALocalSSH": {
            "Id": 3053,
            "Version": 5031,
            "Timeout": "2m",
            "Result": {
                "Map": {}
            },
            "StartAt": 0,
            "NThreads": 0,
            "Duration": "",
            "IPDuration": "",
            "GenLan": false,
            "GenAAA": false,
            "Ports": null,
            "UrlPath": "/static/5031/ddgs"
        },
        "AAredis": {
            "Id": 0,
            "Version": 0,
            "Timeout": "",
            "Result": {
                "Map": {}
            },
            "StartAt": 0,
            "NThreads": 0,
            "Duration": "",
            "IPDuration": "",
            "GenLan": false,
            "GenAAA": false,
            "Ports": null,
            "ShellUrl": ""
        },
        "AAssh": {
            "Id": 2174,
            "Version": 5031,
            "Timeout": "1m",
            "Result": {
                "Map": {}
            },
            "StartAt": 0,
            "NThreads": 100,
            "Duration": "480h",
            "IPDuration": "12h",
            "GenLan": true,
            "GenAAA": false,
            "Ports": [
                22,
                1987,
                2222,
                22222,
                12222
            ],
            "UrlPath": "/static/5031/ddgs"
        },
        "AAnexus": {
            "Id": 0,
            "Version": 0,
            "Timeout": "",
            "Result": {
                "Map": {}
            },
            "StartAt": 0,
            "NThreads": 0,
            "Duration": "",
            "IPDuration": "",
            "GenLan": false,
            "GenAAA": false,
            "Ports": null,
            "ShellUrl": ""
        },
        "AAsupervisord": {
            "Id": 0,
            "Version": 0,
            "Timeout": "",
            "Result": {
                "Map": {}
            },
            "StartAt": 0,
            "NThreads": 0,
            "Duration": "",
            "IPDuration": "",
            "GenLan": false,
            "GenAAA": false,
            "Ports": null,
            "ShellUrl": ""
        }
    }
}


```

定义一个复杂的数据结构并将其序列化编码，然后通过逆向分析来还原真实的数据结构定义，算是一个比较有难度的工作了。要想准确还原数据结构定义，除了按照 **[go_parser](https://github.com/0xjiayu/go_parser)** 的解析结果，把数据结构的定义从 IDAPro 中梳理出来，在编解码这方面就有了额外的坑需要趟。

13. 总结
------

Go 二进制逆向的基础到进阶姿势，从函数符号到字符串，从数据类型定义到 itab，还涉及底层的 **firstmoduledata** 和 **pclntab** 结构…… 到这里算是系统地介绍了一遍。通过解析 Go runtime 中包含的信息来辅助逆向分析工作的路子，到这里也捋的差不多了。懂得原理之后，再借助 **[go_parser](https://github.com/0xjiayu/go_parser)** 这类趁手的工具，可以熟练应对日常见到的绝大部分 Go 二进制文件。

然而，这就结束了吗？？？？并没有。

这其实只是个开始。Go 编写的恶意软件对抗的序幕刚刚拉开没多久，日后 Go 编写的恶意软件越来越多，对抗的手段也会多种多样，到那个时候，本系列文章介绍的内容，都只能算是基础姿势了。

对抗无止境。

参考资料：
-----

1.  [https://github.com/0xjiayu/go_parser](https://github.com/0xjiayu/go_parser)
2.  [https://blog.netlab.360.com/ddg-upgrade-to-new-p2p-hybrid-model/](https://blog.netlab.360.com/ddg-upgrade-to-new-p2p-hybrid-model/)
3.  [https://blog.netlab.360.com/old-botnets-never-die-and-ddg-refuse-to-fade-away/](https://blog.netlab.360.com/old-botnets-never-die-and-ddg-refuse-to-fade-away/)
4.  [https://blog.netlab.360.com/ddg-mining-botnet-jin-qi-huo-dong-fen-xi/](https://blog.netlab.360.com/ddg-mining-botnet-jin-qi-huo-dong-fen-xi/)
5.  [https://zhuanlan.zhihu.com/p/34211611](https://zhuanlan.zhihu.com/p/34211611)
6.  [https://blog.netlab.360.com/ddg-upgrade-to-new-p2p-hybrid-model/](https://blog.netlab.360.com/ddg-upgrade-to-new-p2p-hybrid-model/)
7.  [https://msgpack.org/](https://msgpack.org/)
8.  [https://github.com/0xjiayu/DDGBotnetTracker](https://github.com/0xjiayu/DDGBotnetTracker)
9.  [https://doc.yonyoucloud.com/doc/wiki/project/the-way-to-go/10.5.html](https://doc.yonyoucloud.com/doc/wiki/project/the-way-to-go/10.5.html)