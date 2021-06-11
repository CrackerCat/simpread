> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/214940)

> 本系列文章将简单介绍 Go 语言二进制文件逆向姿势的发展历史，以及几个典型的恶意程序家族。

[![](https://p5.ssl.qhimg.com/t01a10c24a01079c7d3.jpg)](https://p5.ssl.qhimg.com/t01a10c24a01079c7d3.jpg)

1. 概述
-----

Go 语言是一个比较新的强类型静态语言，2009 年由 Google 发布，在 2012 年才发布首个稳定版。Go 语言靠 Goroutine 和 channel、wait group、select、context 以及 sync 等辅助机制实现的 CSP 并发模型，以简洁高效编写高并发程序为亮点，很快就在全球范围内吸引大量开发者使用其编写各种程序。现在 Go 已成为云原生领域的首选开发语言。

由于 Go 语言上手快，编码效率高，程序运行效率也高，而且很方便跨平台交叉编译，也吸引了恶意软件开发者的注意。渐渐地，安全厂商开始捕获到越来越多的 [Go 语言编写的恶意软件](https://unit42.paloaltonetworks.com/the-gopher-in-the-room-analysis-of-golang-malware-in-the-wild/) ，这些恶意软件横跨 Windows、Linux 和 Mac 三大平台。

然而，Go 语言的编译工具链会全静态链接构建二进制文件，把标准库函数和第三方 package 全部做了静态编译，再加上 Go 二进制文件中还打包进去了 runtime 和 GC(Garbage Collection，垃圾回收) 模块代码，所以即使做了 strip 处理 ( `go build -ldflags "-s -w"` )，生成的二进制文件体积仍然很大。在反汇编工具中打开 Go 语言二进制文件，可以看到里面包含动辄几千个函数。再加上 Go 语言的独特的函数调用约定、栈结构和多返回值机制，使得对 Go 二进制文件的分析，无论是静态逆向还是动态调式分析，都比分析普通的二进制程序要困难很多。

不过，好消息是安全社区还是找到了针对性的方法，让安全分析人员对 Go 语言二进制文件的逆向变得更加轻松。最开始有人尝试过针对函数库做符号 Signature 来导入反汇编工具中，还原一部分二进制文件中的函数符号。后来有人研究出 Go 语言二进制文件内包含大量的运行时所需的符号和类型信息，以及字符串在 Go 二进制文件中独特的用法，然后开发出了针对性的符号甚至类型信息恢复工具。至此，对 Go 语言二进制文件的逆向分析工作，就变得轻松异常了。

本系列文章将简单介绍 Go 语言二进制文件逆向姿势的发展历史，以及几个典型的恶意程序家族。然后详细介绍基于二进制文件内置的符号、类型信息来逆向分析 Go 语言二进制文件的技术原理和工具，最后以实际的分析案例演示前面介绍的工具和方法。至于还有人研究出[基于符号执行和形式化定理证明](http://home.in.tum.de/~engelke/pubs/1709-ma.pdf)的高深技术，来恢复 Go 语言二进制文件的符号和类型信息的姿势，不在本文讨论范围之内。~因为鄙人也没研究明白~

2. 典型的恶意程序
----------

早在 2012 年，Symantec(现已被博通收购) 就曝光了一个 Go 语言编写的 Windows 平台上的恶意软件：**[Encriyoko](https://community.broadcom.com/symantecenterprise/communities/community-home/librarydocuments/viewdocument?DocumentKey=7a3cd022-0705-43fb-8c11-181ec86b2c74&CommunityKey=1ecf5f55-9545-44d6-b0f4-4e4a7f5f5e68&tab=librarydocuments)** ，这是鄙人能查到的最早的 Go 编写的恶意软件。当时，这个恶意软件在业内并没引起多大注意。

到了 2016 年 8 月，Go 编写的两个恶意软件被俄罗斯网络安全公司 Dr.Web 曝光，在业内吸引了很多注意：**[Linux.Lady](https://vms.drweb.com/virus/?i=8400823&lng=en)** 和 **[Linux.Rex](https://vms.drweb.com/virus/?_is=1&i=8436299&lng=en)** 。前者后来发展成臭名昭著的 **[DDGMiner](https://blog.netlab.360.com/tag/ddg/)** ，后者则是**史上第一个 Go 编写的 P2P Botnet**(基于 [DHT](https://en.wikipedia.org/wiki/Distributed_hash_table) )。从公开的信息来看，正是从这时开始，业内的安全研究人员开始对 Go 二进制文件的逆向分析进行初步探索。

2017 年，TrendMicro 曝光了一个[大型的黑客团伙 BlackTech](https://blog.trendmicro.com/trendlabs-security-intelligence/following-trail-blacktech-cyber-espionage-campaigns/) ，他们用到的一个核心的数据窃取工具 **DRIGO**，即是用 Go 语言编写。2019 年 [ESET 发布一篇报告](https://www.welivesecurity.com/2019/09/24/no-summer-vacations-zebrocy/)，分析了 APT28 组织用到的知名后门工具 **Zebrocy** ，也有了 Go 语言版本。这也说明 Go 语言编写的木马越来越成熟，Go 语言开始被大型黑客组织纳入编程工具箱。

再往后，Go 语言编写的恶意软件就呈泛滥的态势了。2020 年初，鹅厂还曝光过一个功能比较复杂的跨平台恶意挖矿木马 **[SysupdataMiner](https://s.tencent.com/research/report/904.html)** ，也是由 Go 语言编写。就在最近， Guardicore 刚爆光了一个 [Go 编写的功能复杂的 P2P Botnet: **FritzFrog**](https://www.guardicore.com/2020/08/fritzfrog-p2p-botnet-infects-ssh-servers/) 。

3. 已有研究与工具
----------

前文说过，Go 语言二进制文件有它自己的特殊性，使得它分析起来跟普通的二进制文件不太一样。主要有以下三个方面：

1.  Go 语言内置一些复杂的数据类型，并支持类型的组合与方法绑定，这些复杂数据类型在汇编层面有独特的表示方式和用法。比如 Go 二进制文件中的 string 数据不是传统的以 `0x00` 结尾的 C-String，而是用 (StartAddress, Length) 两个元素表示一个 string 数据；比如一个 slice 数据要由 (StartAddress, Length, Capacity) 三个元素表示。这样的话，在汇编代码中看，给一个函数传一个 string 类型的参数，其实要传两个值；给一个函数传一个 slice 类型的参数，其实要传 3 个值。返回值同理；[![](https://p4.ssl.qhimg.com/t0142b0c3faed3a5c78.png)](https://p4.ssl.qhimg.com/t0142b0c3faed3a5c78.png)
2.  独特的调用约定和栈管理机制，使 C/C++ 二进制文件逆向分析的经验在这里力不从心：Go 语言用的是 [continue stack 栈管理机制](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.5.html) ，并且 Go 语言函数中 callee 的栈空间由 caller 来维护，callee 的参数、返回值都由 caller 在栈中预留空间，就难以直观看出哪个是参数、哪个是返回值。详见 [The Go low-level calling convention on x86-64](https://dr-knz.net/go-calling-convention-x86-64.html) ；
3.  全静态链接构建，里面函数的数量动辄大几千，如果没有调试信息和符号，想静态逆向分析其中的特定功能点，如大海捞针，很容易迷失在函数的海洋中；动态调试难度更大，其独特的 Goroutine 调度机制再加上海量的函数，很容易调飞。

由于恶意软件大都是被 strip 处理过，已经去除了二进制文件里的调试信息和函数符号，所以 Go 二进制文件的逆向分析技术的探索，前期主要围绕着函数符号的恢复来展开。

最早是有人尝试过为函数符号做 Signature ，然后把 Signature 导入到反汇编工具里的做法。这是一个 Hard Way 的笨办法，比较原始，但挺实用。r2con 2016 上 zl0wram 的议题《[Reversing Linux Malware](https://www.youtube.com/watch?v=PRLOlY4IKeA&feature=youtu.be)》中就演示过这种做法：

[![](https://p0.ssl.qhimg.com/t01c287625c3a5bcdb0.png)](https://p0.ssl.qhimg.com/t01c287625c3a5bcdb0.png)

进一步，大家发现了隐藏在 Go 二进制文件种 **pclntab** 结构中的函数名信息，并没有被 strip 掉，而且可以通过辅助脚本在反汇编工具里将其恢复。比如 RedNaga 的 **@timstrazz** 写了一篇 **[Reversing GO binaries like a pro](https://rednaga.io/2016/09/21/reversing_go_binaries_like_a_pro/)** ，详细讲述了如何从被 strip 的 Go 二进制文件中恢复函数符号以及解析函数中用到的字符串，让 IDAPro 逆向 Go 二进制文件变得更轻松。[@timstrazz](https://github.com/timstrazz "@timstrazz") 还开源了他写的 IDA 脚本 **[golang_loader_assist](https://github.com/strazzere/golang_loader_assist)** 。

值得一提的是， **golang_loader_assist** 诞生时，旧版的 IDAPro 对 Go 二进制中的函数识别效果并不是很好，很多函数体识别不全，导致 IDAPro 的自动分析能分析出的函数量有限。所以 **golang_loader_assist** 实现了一种依靠 Go 语言中 **连续栈 (Continue Stack)** 维护的机制来解析、标注函数体的功能。具体做法是靠汇编代码的特征来找出 `runtime_morestack` 和 `runtime_morestack_noctxt` 函数，然后在 IDAPro 种遍历对这两个函数交叉引用的位置来找出函数体。这是一个无奈之举，IDAPro v7.x 的版本对 Go 二进制文件中函数体的自动解析功能加强了很多，绝大部分函数都可以被识别出来，无需自己费劲去识别、解析函数体。

再进一步，有安全研究员发现除了可以从 **pclntab** 结构中解析、恢复函数符号，Go 二进制文件中还有大量的类型、方法定义的信息，也可以解析出来。这样就可以大大方便对 Go 二进制文件中复杂数据结构的逆向分析。两个代表工具：

1.  用于 radare2 的 **[r2_go_helper](https://github.com/zlowram/radare2-scripts/tree/master/go_helpers)** ，由上面提到的 zl0wram 在 r2con 2016 上发布；
2.  用于 IDAPro 的 **[GoUtils](https://gitlab.com/zaytsevgu/goutils)** ，以及基于 **[GoUtils 2.0](https://gitlab.com/zaytsevgu/GoUtils2.0)** 开发的更强的 **[IDAGolangHelper](https://github.com/sibears/IDAGolangHelper)** 。

前文提到的 **[Reversing GO binaries like a pro](https://rednaga.io/2016/09/21/reversing_go_binaries_like_a_pro/)** 可能是业内最火的介绍 Go 二进制逆向的文章，但最火的工具可能还是 **IDAGolangHelper** ：

[![](https://p0.ssl.qhimg.com/t018db0e0f0c475d0eb.png)](https://p0.ssl.qhimg.com/t018db0e0f0c475d0eb.png)

**IDAGolangHelper** 对 Go 的不同版本做了更精细化处理，而且第一次在 Go 二进制文件解析中引入 **moduledata** 这个数据结构。而且提供一个 GUI 界面给用户提供丰富的操作选项，用户体验更胜一筹。

不过 **IDAGolangHelper** 的缺点也非常明显：

1.  支持的 Golang 版本略旧。目前最高支持 Go 1.10，而最新的 Go 1.15 已经发布了。 Go 1.2 之后这些版本之间的差异并不大，所以这也不是个太大的问题；
2.  太久不更新，目前在 IDAPro v7.x 上已经无法顺利执行，这个问题比较严重；
3.  其内部有个独特的做法：把 Go 语言各种数据类型的底层实现，在 IDAPro 中定义成了相应的 ida_struct。这样一来，即使可以顺利在 IDAPro 中解析出各种数据类型信息，展示出来的效果并不是很直观，需要查看相应的 struct 定义才能理解类型信息中各字段的意义，而且不方便跳转操作。窃以为这种体验并不好：[![](https://p5.ssl.qhimg.com/t014478c2f7f9def547.png)](https://p5.ssl.qhimg.com/t014478c2f7f9def547.png)

更进一步，2019 年 10 月份，JEB 官方博客发表一篇文章 《**[Analyzing Golang Executables](https://www.pnfsoftware.com/blog/analyzing-golang-executables/)**》，并发布一个 JEB 专用的 Go 二进制文件解析插件 **[jeb-golang-analyzer](https://github.com/pnfsoftware/jeb-golang-analyzer)** 。这是一个功能比前面几个工具更加完善的 Go 二进制文件解析工具，除了解析前面提到的函数名、字符串和数据类型信息，还会解析 [Duff’s device](https://en.wikipedia.org/wiki/Duff's_device) 、Source File Path list、GOROOT 以及 Interface Table 等信息。甚至会把每个 pkg 中定义的特定数据类型分门别类地列出来，比如解析某 Go 二进制文件中的部分类型信息：

```
> PACKAGE: net/http:

    > struct http.Request (5 fields):
        - string Method (offset:0)
        - *url.URL URL (offset:10)
        - string Proto (offset:18)
        - int ProtoMajor (offset:28)
        - int ProtoMinor (offset:30)

> PACKAGE: net/url:

    > struct url.URL (9 fields):
        - string Scheme (offset:0)
        - string Opaque (offset:10)
        - *url.Userinfo User (offset:20)
        - string Host (offset:28)
        - string Path (offset:38)
        - string RawPath (offset:48)
        - bool ForceQuery (offset:58)
        - string RawQuery (offset:60)
        - string Fragment (offset:70)

    > struct url.Userinfo (3 fields):
        - string username (offset:0)
        - string password (offset:10)
        - bool passwordSet (offset:20)


```

**jeb-golang-analyzer** 也有一些问题：对 strings 和 string pointers 的解析并不到位，虽然支持多种 CPU 架构类型 (x86/ARM/MIPS) 的字符串解析，但是 Go 二进制文件中字符串的操作方式有多种，该工具覆盖不全。另外，该工具内部定位 **pclntab** 的功能实现，基于 Section Name 查找和靠 Magic Number 暴力搜索来结合的方式，还是可能存在误判的可能性，一旦发生误判，找不到 **pclntab** 结构，至少会导致无法解析函数名的后果。最后，这个工具只能用于 JEB，而对于用惯了 IDAPro 的人来说，JEB 插件的解析功能虽强大，但在 JEB 中展示出来的效果并不是很好，而且 JEB 略卡顿，操作体验不是很好。

另外，还有一个非典型的 Go 二进制文件解析工具：基于 **[GoRE](https://go-re.tk/gore/)** 的 **[redress](https://go-re.tk/redress/)**。 GoRE 是一个 Go 语言编写的 Go 二进制文件解析库，**redress** 是基于这个库来实现的 Go 二进制文件解析的 **命令行工具**。redress 的强大之处，可以**结构化**打印 Go 二进制文件中各种详细信息，比如打印 Go 二进制文件中的一些 Interface 定义：

```
$ redress -interface pplauncher
type error interface {
    Error() string
}

type interface {} interface{}

type route.Addr interface {
    Family() int
}

type route.Message interface {
    Sys() []route.Sys
}

type route.Sys interface {
    SysType() int
}

type route.binaryByteOrder interface {
    PutUint16([]uint8, uint16)
    PutUint32([]uint8, uint32)
    Uint16([]uint8) uint16
    Uint32([]uint8) uint32
    Uint64([]uint8) uint64
}


```

或者打印 Go 二进制文件中的一些 Struct 定义以及绑定的方法定义：

```
$ redress -struct -method pplauncher
type main.asset struct{
    bytes []uint8
    info os.FileInfo
}

type main.bindataFileInfo struct{
    name string
    size int64
    mode uint32
    modTime time.Time
}
func (main.bindataFileInfo) IsDir() bool
func (main.bindataFileInfo) ModTime() time.Time
func (main.bindataFileInfo) Mode() uint32
func (main.bindataFileInfo) Name() string
func (main.bindataFileInfo) Size() int64
func (main.bindataFileInfo) Sys() interface {}

type main.bintree struct{
    Func func() (*main.asset, error)
    Children map[string]*main.bintree
}


```

这些能力，都是上面列举的反汇编工具的插件难以实现的。另外，用 Go 语言来实现这种工具有天然的优势：可以复用 Go 语言开源的底层代码中解析各种基础数据结构的能力。比如可以借鉴 [src/debug/gosym/pclntab.go](https://golang.org/src/debug/gosym/pclntab.go) 中的代码来解析 **pclntab** 结构，可以借鉴 [src/runtime/symtab.go](https://golang.org/src/runtime/symtab.go) 中的代码来解析 **moduledata** 结构，以及借鉴 [src/reflect/type.go](https://golang.org/src/reflect/type.go) 中的代码来解析各种数据类型的信息。

**redress** 是一个接近极致的工具，它把逆向分析需要的信息尽可能地都解析到，并以友好的方式展示出来。但是它只是个命令行工具，跟反汇编工具的插件相比并不是很方便。另外，它目前还有个除 **jeb-golang-analyzer** 之外以上工具都有的缺点：限于内部实现的机制，**无法解析 `buildmode=pie` 模式编译出来的二进制文件**。用 redress 解析一个 **[PIE(Position Independent Executable)](https://docs.google.com/document/d/1nr-TQHw_er6GOQRsF6T43GGhFDelrAP0NqSS_00RgZQ/edit?pli=1#heading=h.nidcdnrtrn3n)** 二进制文件，报错如下：

[![](https://p3.ssl.qhimg.com/t01e7f28259df6c3bf1.png)](https://p3.ssl.qhimg.com/t01e7f28259df6c3bf1.png)

最后，是鄙人开发的一个 IDAPro 插件： **[go_parser](https://github.com/0xjiayu/go_parser)** ，该工具除了拥有以上各工具的绝大部分功能 (strings 解析暂时只支持 x86 架构的二进制文件，这一点不如 **jeb-golang-analyzer** 支持的丰富)，还**支持对 PIE 二进制文件的解析**。另外会把解析结果以更友好、更方便进一步操作的方式在 IDAPro 中展示。以 DDG 样本中一个复杂的结构体类型为例，解析结果如下：

[![](https://p4.ssl.qhimg.com/t01d00058530284de96.png)](https://p4.ssl.qhimg.com/t01d00058530284de96.png)

4. 原理初探
-------

前文盘点了关于 Go 二进制文件解析的已有研究，原理层面都是一句带过。可能很多师傅看了会有两点疑惑：

1.  为什么 Go 二进制文件中会有这么多无法被 strip 掉的符号和类型信息？
2.  具体有哪些可以解析并辅助逆向分析的信息？

第一个问题，一句话解释就是，Go 二进制文件里打包进去了 **runtime** 和 **GC** 模块，还有独特的 **Type Reflection**(类型反射) 和 **Stack Trace** 机制，都需要用到这些信息。参考前文 **redress** 报错的配图，redress 本身也是 Go 语言编写，其报错时打出来的栈回溯信息，除了参数以及参数地址，还包含 pkg 路径、函数信息、类型信息、源码文件路径、以及在源码文件中的行数。

至于内置于 Go 二进制文件中的类型信息，主要为 Go 语言中的 Type Reflection 和类型转换服务。Go 语言内置的数据类型如下：

[![](https://p3.ssl.qhimg.com/t010f01af0e76e97c60.png)](https://p3.ssl.qhimg.com/t010f01af0e76e97c60.png)

而这些类型的底层实现，其实都基于一个底层的结构定义扩展而来：

[![](https://p3.ssl.qhimg.com/t01ec2c57eccba40a41.png)](https://p3.ssl.qhimg.com/t01ec2c57eccba40a41.png)

再加上 Go 允许为数据类型绑定方法，这样就可以定义更复杂的类型和数据结构。而这些类型在进行类型断言和反射时，都需要对这些底层结构进行解析。

第二个问题，对于 Go 二进制文件中，可以解析并对逆向分析分析有帮助的信息，我做了个列表，详情如下：

[![](https://p1.ssl.qhimg.com/t01f134e86cd71f543d.png)](https://p1.ssl.qhimg.com/t01f134e86cd71f543d.png)

后文会以这张图为大纲，以 **[go_parser](https://github.com/0xjiayu/go_parser)** 为例，详细讲解如何查找、解析并有组织地展示出这些信息，尽最大可能提升 Go 二进制文件逆向分析的效率。

且看下回分解。