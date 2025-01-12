> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285212.htm)

> [原创]app 加固分析狗尾续貂之 dex vmp 还原

前言
==

oacia 大佬的 [APP 加固 dex 解密流程分析](https://bbs.kanxue.com/thread-280609.htm)，已经基本把它翻个底朝天了，但众所周知要彻底攻破该加固，仍有 dex vmp 这最后一道堡垒横在面前。在此之前拜读过爱吃菠菜大佬的[某 DEX_VMP 安全分析与还原](https://bbs.kanxue.com/thread-270799.htm)以及 Thehepta 大佬的 [vmp 入门 (一)：android dex vmp 还原和安全性论述](https://bbs.kanxue.com/thread-281427.htm)，了解到 dex vmp 的原理是对 dalvik 指令进行了加密和置换，这里参照大佬们的方法对 oacia 大佬的样本复现了一遍，记录一下历程。

获取 trace
========

既然是 vmp，按照 vmp 常规的分析流程，必然要先拿到 trace，以这次的经历来看，拿到 trace 就已经成功了一半了。参考了很多大佬的文章，大致有以下几种方法：ida trace、unidbg/AndroidNativeEmu、frida stalker、frida qbdi、Dobby Instrument、unicorn 虚拟 CPU。

ida trace
---------

开始打算用 maiyao 的 [ida trace 脚本](https://github.com/maiyao1988/IDAScripts/blob/master/ida_trace.py)，但遇到以下问题：  
1.ida 附加上后每次执行到 linker 加载壳 so 加载到一半就退出了，单步发现是跑到 rtld_db_dlactivity 里的 BRK 指令会直接退出，但加载其他 so 时并没有这个问题（why?)。这里手动把它跳过了。  
2.trace 了一下 JNI_Onload，结果发现每当进入 mutex_lock 就会卡死在那无限循环…… 这个没能解决，只好弃用。

unidbg/AndroidNativeEmu
-----------------------

补 java 环境有点累，尤其是补到后面发现可能会有很多代理类，没坚持补下去……

frida stalker
-------------

从 oacia 那篇文章来的肯定少不了尝试一下 stalker，结果发现和 ida 相似的问题——每当进入 mutex_lock 就会卡死在那无限循环。

frida qbdi
----------

尝试用 yang 的 [frida-qbdi-tracer](https://github.com/lasting-yang/frida-qbdi-tracer)，但每当 trace 到一些跳转的时候就会崩溃，查阅资料发现 qbdi 好像确实存在这样的 bug。

Dobby Instrument（未尝试）
---------------------

这个我是后面才看到的，感觉将来可以尝试一下，详见 KerryS 的[指令级工具 Dobby 源码阅读](https://bbs.kanxue.com/thread-273487.htm)。

unicorn 虚拟 CPU（未尝试）
-------------------

爱吃菠菜那篇文章的中的方法，没完全搞明白，感觉大意应该是把 unicorn 作为一个 so 文件注入到 app 中，用 frida hook 住目标函数，当发生调用时把环境补给 unicorn 来 trace。

小结
--

经过不断尝试，最终还是使用 frida stalker，通过 hook mutex_lock 函数，在 OnEnter 中跳过 stalker，再在 OnLeave 中恢复 stalker 的方法，在经过无数次崩溃之后，终于拿到了 dex vmp 的 trace……

指令流分析
=====

拿到 trace 指令流后，就可以着手分析了。可以看到，onCreate 函数被注册到的地方，使用 DebugSymbol.fromAddress 没有打印出任何符号，应该是一块动态申请的内存区域：  
![](https://bbs.kanxue.com/upload/tmp/941380_QQQHT9PW9JKK59R.webp)  
函数并不大，里面也基本全是位置无关代码，应该只是一个跳板，很快，便来到了主 elf 中的函数 sub_137978。

跳转表修复（未完成）
----------

sub_137978 这个函数就大了，该函数内的指令流数量占了全部的 99%，然而在 ida F5 中却不那么好分析，因为它全程用跳转表实现：  
![](https://bbs.kanxue.com/upload/tmp/941380_2C2HKKWEU7RMDSB.webp)  
![](https://bbs.kanxue.com/upload/tmp/941380_779YJPD2X296D68.webp)  
![](https://bbs.kanxue.com/upload/attach/202501/941380_V5PQSRZ3XJ24VTW.webp)  
![](https://bbs.kanxue.com/upload/tmp/941380_PAVVGDNQD63PVXH.webp)  
ida 倒是具备跳转表修复的功能，在 Edit->other->specify switch idiom 中，可以参考 [https://github.com/ChinaNuke/ChinaNuke.github.io/blob/main/2021/08/Specify-switch-statement-in-IDA-Pro/index.html](https://github.com/ChinaNuke/ChinaNuke.github.io/blob/main/2021/08/Specify-switch-statement-in-IDA-Pro/index.html) （原博文已挂，这里的备份需要下载下来看）  
我按照它的讲解修复了一下，不过效果也不是太好，估计是没搞清 start of switch idiom 和 input register。  
也尝试过用数据流分析来回溯跳转表，然后转化为 bcond addr1、b addr2 来修复，不过因为可能会涉及指令重排的问题，而且有的块的前驱没有分析出来会导致数据流中断，懒得再用迭代了…… 这里直接从 trace 中找到这些 BR X8 的后继，然后在 ida 中用 idc.add_cref(ea, next_ea, idc.fl_JF) 添加上引用，姑且在汇编分析了。

函数调用分析
------

不过，这么多指令应该从哪入手呢？先跟一下它的函数调用吧。在 trace 中搜索 BL，先是几个比较短的函数，然后进入到一个有意思的函数 sub_13B1D8：  
![](https://bbs.kanxue.com/upload/tmp/941380_76264BTDA6BANEY.webp)  
看到这么大这么整齐的一个结构体，而之前又没有看到什么对 a2 的初始化，难道？我尝试将 a2 改为 JNIEnv*，竟然整齐地对上了！再退回到 result 初始化的地方，看到 result 结构体的大小为 0x728，和 JNIEnv 的大小 0x748 如此接近，那么很有可能是一个打乱顺序的 JNIEnv。既然这么巧，先把这个结构体创建出来：  
![](https://bbs.kanxue.com/upload/tmp/941380_QY8MK6UP52KRDMJ.webp)  
继续向下分析函数调用，在经过一系列短函数后，来到了函数 sub_144EF0。  
这个函数指令流数量占了 90% 以上，而其中再没有占比特别高的函数了，大概它就是虚拟机入口了，给它命名为 vm_entry_sub_144EF0，接下来主要研究它。

虚拟机分析
-----

和 sub_137978 一样，vm_entry_sub_144EF0 同样是全程跳转表，怎么办呢？  
根据上文提到的爱吃菠菜以及 Thehepta 的文章, 可知 dex vmp 中会大量调用 JNI 函数，所以从它们入手是一个不错的选择。

### 从 JNI 调用入手

先看这条：  
FindClass(androidx/appcompat/app/AppCompatActivity)libjiagu_64.so!0x19e654  
我们来到 0x19e654 这个地址，将参数设为重排的 JNIEnv，果然，这里确实是调用 FindClass，这也印证了前面对 JNI 的猜测。查看其调用链为 vm_entry_sub_144EF0->sub_160D7C->sub_19E508。  
先看一下 sub_160D7C：  
![](https://bbs.kanxue.com/upload/tmp/941380_2BSQ4X2P5XP5HQ9.webp)  
有很多不同类型的 CallXXMethod，这些显然是在解释函数调用指令，将该函数命名为 call_method_sub_160D7C。  
从 call_method_sub_160D7C 调用处向上回溯，我发现它经历了一系列的 cmp w0, xx 的比较和跳转，很像是在根据操作码来选择 handler 的流程。搜索一下  
w0 的来源，发现是来自 sub_15FC34 的返回值，其输入一部分是来自同一基本块内的 sub_16CE0C，这两个函数都总共被执行了 17 次，均匀地分布在整个 vm_entry_sub_144EF0 的指令流中。  
继续向上回溯，来到 loc_153FC8，查看其引用，发现有多达 254 处，而联想到 dalvik opcode 为 1 字节，256 种，因此这里多半就是虚拟机的 dispatcher 了。  
再看一下 oacia 的未加固样本的 OnCreate 函数指令：  
![](https://bbs.kanxue.com/upload/tmp/941380_QHAK2KSM4F3A7CA.webp)  
数一下，也是刚好 17 条。因此，sub_15FC34 和 sub_16CE0C 应该就是解析字节码的函数了。

### 解密字节码

sub_16CE0C 中多次用到异或运算，像是在解密字节码：  
![](https://bbs.kanxue.com/upload/tmp/941380_CSP39CUTP3ZNSX8.webp)  
而 sub_15FC34，如前所述其返回值是 handler 所对应的 w0，不过没仔细研究其生成过程，像是从文件读取的：  
![](https://bbs.kanxue.com/upload/tmp/941380_YUZSSHYR4CNGKX7.webp)  
回到 call_method_sub_160D7C 继续分析，又发现了两个和 sub_16CE0C 极其相似的函数，不过分别是从 offset=2 和 4 处开始解密，查看其返回值，分别为 0x154b(5451) 和 0x21，根据 dalvik 的函数调用指令 invoke 的格式，其 offset=2 处的对应的是函数 index，我们拿 010editor 来看一下 method 5451：  
![](https://bbs.kanxue.com/upload/tmp/941380_Z6WSDQPW7D2EEMJ.webp)  
恰好就是其实际调用的函数！  
我们把这几个解密字节码的函数的所有返回值按序排列一下，和原字节码进行对比：  
![](https://bbs.kanxue.com/upload/tmp/941380_UWGYXBUFH46EGHD.webp)  
可以看到，除了作为操作码的首字节及部分操作数外，其余均相同。而这些操作数不同的地方，其实也仅仅是因为 dex 内容发生改变而导致的 index 的不同，实际指向仍然是相同的。

### 还原操作码

那么接下来就是还原操作码苦力活了，不过也有迹可循，至少根据 jni 调用可以把 method、field 相关的指令还原，在资源中搜索操作数可以看看是不是对资源的引用，也可以像破解凯撒密码那样利用不同指令的统计特征及上下文相关性来推断。也有一些取巧的方法，比如自己构造一个容纳了尽量多类型指令的 onCreate 函数进行加固作为对照，即使它指令对照表发生改变，也可以根据其对应 handler 相似性来推断。

总结
==

综上所述，onCreate 函数的关键流程包括：重排 JNI、解密操作码、选择 handler、解密操作数，其中操作码由于进行了置换，还需要通过各种方法进一步还原成 dalvik 操作码。

后记
==

整个数字加固研究下来，涉及到的大致包括 ffi 调用、frida 反调试、elf 动态解密加载、自实现 linker 解析与重定位、dex 动态解密加载、dex vmp。但说实话，我花的时间最多的，虽然是 dex vmp 部分，但确是在如何获取完整的 trace 上，之后的分析反而没有花太多时间，可能因为像解密字节码关键函数没有用跳转表混淆掉吧，仍然可以用 ida f5 分析，而且操作数没有做其他的变换，另外就是有大佬们经典文章的加持。  
不过仍有一些问题悬而未决，比如字节码的来源，又是如何被加载到内存中的？interpreter_wrap_int64_t、interpreter_wrap_float 等这几个函数是什么原理？ida 调试为什么单单壳 so 在运行到 rtld_db_dlactivity 里 BRK 时会退出而其他 so 不会？以后有机会再回过头来研究吧（逃）

参考文献
====

[APP 加固 dex 解密流程分析](https://bbs.kanxue.com/thread-280609.htm)，oacia  
[ida trace 脚本](https://github.com/maiyao1988/IDAScripts/blob/master/ida_trace.py)，my1988  
[frida-qbdi-tracer](https://github.com/lasting-yang/frida-qbdi-tracer)，Imyang  
[某 DEX_VMP 安全分析与还原](https://bbs.kanxue.com/thread-270799.htm)，爱吃菠菜  
[vmp 入门 (一)：android dex vmp 还原和安全性论述](https://bbs.kanxue.com/thread-281427.htm)，Thehepta  
[Dalvik 解释器源码到 VMP 分析](https://bbs.kanxue.com/thread-226214.htm)，glider  
[从 0 开发你自己的 Trace 后端分析工具: (1) 概述](https://bbs.kanxue.com/thread-281555.htm)，FANGG3  
[指令级工具 Dobby 源码阅读](https://bbs.kanxue.com/thread-273487.htm)，KerryS

[[注意]APP 应用上架合规检测服务，协助应用顺利上架！](https://www.kanxue.com/manxi.htm)

[#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm)