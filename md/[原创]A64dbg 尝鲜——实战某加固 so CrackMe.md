> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266882.htm)

目录

*   [前言](#前言)
*            [Q&A](#q&a)
*            [关于 A64dbg 的 ADCpp 脚本系统](#关于a64dbg的adcpp脚本系统)
*   [360 旧版加固的 vm 与反调试](#360旧版加固的vm与反调试)
*   [so 文件分析](#so文件分析)
*            [A64dbg-ADCpp 测试](#a64dbg-adcpp测试)
*                    [启动 lidadbg-server](#启动lidadbg-server)
*                    [设置断点](#设置断点)
*                    [ADCpp 脚本系统主动调用解密函数 ooxx](#adcpp脚本系统主动调用解密函数ooxx)
*                            [方式一](#方式一)
*                            [方式二](#方式二)
*                    [效果](#效果)
*                    [Python 脚本 Dump 内存中已解密 so 文件](#python脚本dump内存中已解密so文件)
*                    [效果验证](#效果验证)
*   [后记](#后记)

前言
==

Q&A
---

Q：为什么写这篇文章？

 

A：因为一方面，目前还没有使用 A64dbg 工具进行实战并附有详细分析和使用的文章，此文算是新品开箱的尝鲜文章，分享给各位看雪用户。另一方面，作为安卓逆向爱好者，拿到新的逆向工具还是比较激动的，在进行了不少尝试后，还是希望分享给大家一些有用的东西的。

 

Q：文章所涉及的功能，是 A64dbg 不同于其他逆向工具的全部了吗？

 

A：当然不是，涉及的只是我个人最感兴趣的功能——即 A64dbg 的 UnicornVM 调试模式下的 ADCpp 脚本系统。

 

Q：看完文章后去下载 A64dbg 可以复现这篇文章吗？

 

A：可能不行，我是拿到了 A64dbg 的测试授权，在授权情况下拥有完整功能后才进行的测试，未授权的话可能会有些限制，但我并没有具体地测试此文使用到的 A64dbg 功能被限制到了什么程度，如果感兴趣的话可自行尝试。

 

Q：文中写的内容好像用 unidbg 也可以做到，有什么区别吗？

 

A：unidbg 很强大，但 A64dbg 的 UnicornVM 模式并不同于 unidbg，怎么简洁明了地区别理解它们呢，那就是 unidbg 需要模拟 linux 系统和 java 环境，而 A64dbg 的 UnicornVM 模式并没有模拟这些，是在设备机中启动了 lidadbg-server 然后客户端调试器进行对接，也就是说是运行在真实的环境中的。

关于 A64dbg 的 ADCpp 脚本系统
----------------------

最近上手了 A64dbg 的 unicornVM 模式的下调试，分析样本为 [so 文件动态加解密的 CrackMe](https://bbs.pediy.com/thread-266546.htm#msg_header_h3_1) 一文中进行了 360 加固的 crackme。

 

先要说明的是，在了解 A64dbg 之后，首先吸引到我的是 A64dbg 的 ADCpp 脚本系统，先看一下介绍。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_6Z54PEKFUHPXSE4.jpg)

> IDA 有 IDC 这样的类 C 脚本系统，Frida 嫁接了一层 JavaScript 脚本系统。虽然用起来还行，但始终不能让人满意。因为与 Native 打交道，就应该像 C/C++ 那样直白，毕竟操作 void * 才是 Native 的精华。受限于 C/C++ 是编译型的静态语言，想要实现像 Python/JavaScript 那样的便捷性着实不易。
> 
> LLDB 的表达式倒是可以使用 C/C++ 语句，但是高度依赖类型系统，否则难以写出有效的 C/C++ 表达式。即便如此，还是太弱了，不能写出复杂的完整 C/C++ 函数。
> 
> 但是，这一切即将成为过去时，我们将在 A64Dbg v1.6 专业版中引入 UnicornVM 的时候同时引入 ADCpp 这个 C/C++ 脚本系统，它是把 C/C++ 定义为了编译型动态语言，使用解释执行的方式运行代码。C/C++ 最终编译为机器相关的 arch.adc 字节码，由 UnicornVM 直接加载执行。不需要手动指定头文件、不需要手动指定链接库文件、不需要手动加载动态库，你只需要 code，剩下的就是交给 ADCpp 了。
> 
> 你还可以在 A64Dbg 端定义数据接收的 Python3 函数，然后在 C/C++ 端发送给它，类似于 Frida JavaScript 与 Python 的交互，提供 ADCpp 接口可以主动调用内存模块中的函数，或者是只要有一个指针即可。
> 
> ——[A64Dbg-ADCpp 脚本系统简介](https://mp.weixin.qq.com/s/ShV5-V4XDiyb4mfvVkiAHA)

 

就是说通过 ADCpp 脚本系统你可以实现使用 C/C++ 代码来完成对内存中模块函数的注入 hook，或者是进行主动调用，甚至只要有一个指针地址就可以实现，并且不管是系统空间还是用户空间。

 

基于对此功能的好奇，我进行了简单的测试，尝试使用 A64dbg 主动调用 socrackme 的解密函数，随后完成 dump 解密的 so 文件，以测试 A64dbg 在安卓逆向中内存操作的实战效果，这里将测试结果分享给大家。

360 旧版加固的 vm 与反调试
=================

在开始之前，先来说下 [so 文件动态加解密的 CrackMe](https://bbs.pediy.com/thread-266546.htm#msg_header_h3_1) 一文所做的分析，作者是看雪用户 genliese 。genliese 在文章开头说的很清楚，因为 360 加固的反调试所以 so 文件采用全静态分析，这是全文的基调。

 

关于反调试，笔者简单分析了下此版本的 360 加固，发现是典型的旧 360 壳，判断标志是 libjiagu.so 中存在`__fun_a_18`函数，而反调试就在里面的 case 当中，在论坛上几年前就已经有相关的分析文章了，最早的应该是[某数字公司 VMP 脱壳简记](https://bbs.pediy.com/thread-223528.htm)。

 

`__fun_a_18`函数的流程图看上去像是被加了 ollvm 控制流平坦化，但其实是一个基于堆栈的 vm，找了很多资料，国内应该没有详细分析此 vm 的文章，但是论坛里有一篇翻译老外分析 360 加固的文章（[老外挑战 360 加固 -- 实战分析（很详细）](https://bbs.pediy.com/thread-225561.htm) ），里面有详细说这个 vm，并且在 github 有相关还原代码。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_M5UVWNWH8HFWZ76.png)

 

众所周知，旧版 360 加固的学习价值很高，是入门者练手分析很好的素材，这里我整理了些相关文章，列出来方便大家学习：

*   [360 加固保分析](https://bbs.pediy.com/thread-260049.htm)
    
*   [某数字公司 VMP 脱壳简记](https://bbs.pediy.com/thread-223528.htm)
    
*   [根据”so 劫持” 过 360 加固详细分析](https://bbs.pediy.com/thread-223796.htm)
    
*   [老外挑战 360 加固 -- 实战分析（很详细）](https://bbs.pediy.com/thread-225561.htm)
    
*   [vm_emulator.py](https://github.com/fvrmatteo/DMNP/blob/5cc5a4dea1e8ece24f15dd4370134e19a45cbee2/vm_emulator.py#L350)
    
*   [360 加固之 onCreate 函数还原并重打包](https://bbs.pediy.com/thread-223223.htm)
    
*   [某壳分析学习过程](https://bbs.pediy.com/thread-224708.htm)
    
*   [某 vmp 壳原理分析笔记](https://bbs.pediy.com/thread-225798.htm)
    

现在回过头来说下反调试，旧版 360 加固主要有时间反调试、rtld_db_dlactivity 反调试、traceid 反调试和 IDA 端口反调试。

 

其中 rtld_db_dlactivity 反调试和 traceid 反调试是针对 ptrace 方式调试的反调试，在使用 A64dbg 的 UnicornVM 模式调试的测试过程中，笔者并没有发现触发反调试，说明 A64dbg-UnicornVM 模式的调试机制并不依赖于 ptrace，也就是能直接无视掉此壳的反调试。

 

如果大家想尝试使用 IDA 进行调试分析，过反调试的话可以参考这篇文章 [360 加固保分析](https://bbs.pediy.com/thread-260049.htm)。

so 文件分析
=======

文章中 genliese 使用 FART 定制 ROM 进行了脱壳，笔者进了测试，这个版本的 360 壳并没有进行指令抽取及 vmp，使用 dex 整体 dump 即可脱壳。

 

dex 的代码逻辑很简单，输入传进 native 层的 test 函数进行验证。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_DGMWW9A3APQAGQX.png)

 

首先 libnative-lib.so 的. init_array 中的函数进行了字符串解密，

 

![](https://bbs.pediy.com/upload/attach/202104/802108_G8WA6FDHYBA86X3.png)

 

我们跳过，直接看下 JNI_OnLoad 函数，可以看到字符串还没有被解密出来，以及 java 层的 test 方法应该是关联到了 native 层的 ooxx 函数。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_6RQVHCM23EXUUWZ.png)

 

我们来看 ooxx 函数。

```
.text:00008DC4                 EXPORT ooxx
.text:00008DC4 ooxx                                    ; CODE XREF: j_ooxx+8↑j
.text:00008DC4                                         ; DATA XREF: LOAD:000007C0↑o ...
.text:00008DC4
.text:00008DC4 var_2C          = -0x2C
.text:00008DC4 var_28          = -0x28
.text:00008DC4 var_24          = -0x24
.text:00008DC4 var_18          = -0x18
.text:00008DC4 var_14          = -0x14
.text:00008DC4 var_10          = -0x10
.text:00008DC4
.text:00008DC4 ; __unwind {
.text:00008DC4                 PUSH            {R4-R7,LR}
.text:00008DC6                 ADD             R7, SP, #0xC
.text:00008DC8                 SUB             SP, SP, #0x24
.text:00008DCA                 MOV             R3, R2
.text:00008DCC                 MOV             R12, R1
.text:00008DCE                 MOV             LR, R0
.text:00008DD0                 STR             R0, [SP,#0x30+var_10]
.text:00008DD2                 STR             R1, [SP,#0x30+var_14]
.text:00008DD4                 STR             R2, [SP,#0x30+var_18]
.text:00008DD6                 STR             R3, [SP,#0x30+var_24]
.text:00008DD8                 STR.W           R12, [SP,#0x30+var_28]
.text:00008DDC                 STR.W           LR, [SP,#0x30+var_2C]
.text:00008DE0                 BL              sub_8930
.text:00008DE4                 MOV             R0, R0
.text:00008DE6                 MOV             R0, R0
.text:00008DE8                 MOV             R0, R0
.text:00008DEA                 MOV             R0, R0
.text:00008DEC                 MOV             R0, R0
.text:00008DEE                 MOV             R0, R0
.text:00008DF0                 MOV             R0, R0
.text:00008DF2                 MOV             R0, R0
.text:00008DF4                 MOV             R0, R0
.text:00008DF6                 MOV             R0, R0
.text:00008DF8                 MOV             R0, R0
.text:00008DFA                 MOV             R0, R0
.text:00008DFC                 MOV             R0, R0
.text:00008DFE                 MOV             R0, R0
.text:00008E00                 BX              R0
.text:00008E00 ; End of function ooxx
.text:00008E00
.text:00008E00 ; ---------------------------------------------------------------------------
.text:00008E02                 DCW 0x4502
.text:00008E04                 DCD 0x41064304, 0x4D0A4F08, 0x490E4B0C, 0x55125710, 0x51165314
.text:00008E04                 DCD 0x5D1A5F18, 0x591E5B1C, 0x65226720, 0x61266324, 0x6D2A6F28
.text:00008E2C                 DCD 0x692E6B2C
.text:00008E30                 DCD 0x75327730
.text:00008E34                 DCD 0x71367334
.text:00008E38                 DCD 0x7D3A7F38, 0x793E7B3C
.text:00008E40                 DCD 0x5420740
.text:00008E44                 DCD 0x1460344
.text:00008E48                 DCD 0xD4A0F48

```

可以看到初始化后跳到了 sub_8930，而 sub_8930 函数中，我们看到了十分明显的内存指令解密标志`mprotect`以及`cacheflush`函数。

 

至于如何具体进行的内存指令解密操作，[so 文件动态加解密的 CrackMe](https://bbs.pediy.com/thread-266546.htm#msg_header_h3_1) 一文中十分详细的讲解了，这里我就不多说了，此 crackme 是一个很好的指令解密学习素材，大家感兴趣的话可以动手分析下。可以猜到，sub_8930 进行内存指令解密后，便是开始执行真正的代码了，就是对输入进行校验。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_W7D8H6SGP3VHQBT.png)

 

到这一步可以知道，想要拿到 flag 就需要知道被加密的代码是什么。genliese 采取的方式是使用 fridad 动态 dump 内存中的加密指令的解密密钥，然后再使用密钥静态解密文件中的指令加密部分，然后 IDA 即可反编译出解密后的`ooxx`函数，以及不得不称赞的是 genliese 的代码写的很漂亮，让人看的很舒服。

A64dbg-ADCpp 测试
---------------

好了，下面就开始进入此文的关键的部分，对 A64dbg 的 ADCpp 脚本系统的内存操作功能进行测试，这里测试的主要是 ADCpp 脚本系统的主动调用和内存 dump 特性，我们的测试思路如下所述。

 

尽管我们可以在指令解密完后设下断点，然后输入 flag 点击触发断点随后 dump 内存。但是我们并不这样做，而是选择让程序保持运行，使用 ADCpp 脚本系统主动调用指令解密函数`ooxx`，然后让断点停在指令解密完后，随后进行 dump so 文件。

 

也就是说程序一方面正在运行等待我们的输入，光标还在闪动，而另一方面，我们已经悄无声息地开辟了一条新的战线，执行了内存里的函数，并停在了我们设下的断点，然后进行了 dump 操作。

 

至于为什么在解密完的时刻设下断点，让调试器在这里停在这里，因为`ooxx`函数解密出来的指令在对输入进行校验后，还会将`ooxx`再加密回去。

### 启动 lidadbg-server

我们在设备机里面启动`lidadbg-server`后，然后打开 A64dbg，Options->Preferences 里客户端进行设置。

 

填好 Remote Android 栏下的 ip 和 port，接着填好 adb 的路径，以及最重要的是 Default Platform 选择 Remote UnicornVM Android，这样才能进行 UnicornVM 模式下的调试。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_6ZNSGFVT9VDGHKR.png)

 

然后运行 Apk，点击 File->attach，然后搜索包名 attach 即可。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_F7ZH5DDH9QSTEZ2.png)

### 设置断点

第一次加载会比较慢，等待加载好后，点击`Symbols`窗口，左下角搜索模块名称`libnative-lib.so`，右下角搜索函数`ooxx`，双击。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_2EKGQ49ARP86STT.png)

 

进入汇编窗口，可以看到跳到了`sub_8930`函数，双击。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_7N6UBMP5RE2Y6F7.png)

 

然后找到指令后解密完后，调用`cacheflush`的地方，设下断点。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_3PNA3XZBHVFBJHE.png)

### ADCpp 脚本系统主动调用解密函数 ooxx

现在断点已经设置好了，按照我们的计划，要开始进行主动调用了。

 

ADCpp 脚本运行有两种方式，一种是直接载入脚本，另一种则是通过最下方的命令行窗口，这里我们都尝试一下。

 

然后主动调用函数也有两种方式，一种适用于有符号的函数，我们通过`extern "C" void ooxx();`就可以引用 A64dbg 解析好的有符号的函数，可以直接调用；另一种则是知道函数在内存中的地址后，我们将绝对地址转化为函数指针然后进行调用，这时候要注意汇编是否在 thumb 模式下。

#### 方式一

方式一，从解析出来的模块符号表中找到`ooxx`函数，然后进行调用。

 

具体操作是将如下代码保存文件，后缀名为`.cc`，然后 File->ADCpp Script，找到保存的文件，双击。

```
extern "C" void ooxx();
 
void adc_main_thread() {
    ooxx();
}

```

#### 方式二

比如你知道了`ooxx`函数在内存的起始地址为`0x859EDC4`，`thumb`模式下进行`|1`，然后将这个绝对地址转换为函数指针，即可进行调用。

 

具体操作是在最下面的 Command 窗口，选择`ADCpp`模式，输入`((void(*)())(0xD859EDC4 | 1))();`，然后回车。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_RUBFN35WSAARKJK.png)

### 效果

![](https://bbs.pediy.com/upload/attach/202104/802108_XNPXXBQSG7ZKSAZ.png)

 

最后它们都会在`ooxx`里设置的断点停下来。

### Python 脚本 Dump 内存中已解密 so 文件

在这时候内存中的 so 文件已经完成了解密，我们 dump 下来即可，我们该如何 dump 呢？还记文章开头那张图的右上角是什么吗，我们来看一下。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_D7SA88PBN5AHD2Q.png)

 

没错，我们是可以使用 python3 来处理 ADCpp 脚本系统的返回结果的，实际上，ADCpp 脚本系统也提供给 python 丰富的接口去做很多事情，诸如操作内存、断点等，如果感兴趣的话下载 A64dbg 后可以查看`A64dbg/python3/adp.py`文件。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_FU5DRB9KYYWG5T2.png)

 

这里我们只需要 dump 内存，使用 readMemory 一个接口就足够了。

```
def readMemory(addr, size):
    """
    read memory at addr with size within the page, the result is a bytes object.
    """
    return api_proc_result('readMemory', (addr, size))

```

然后我们要确定 so 文件在内存中的分布。

 

首先我们执行 ADCpp 的 commd，即`printf("pid: %d",getpid())`，获取到 pid，这里使用到了 ADCpp 的内置接口`getpid()`。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_XRDCZXCTWEDW2QW.png)

 

然后切换 TarShell 模式，这里就相当于 android 设备的 shell 了。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_R782RRMGPUTQK7J.png)

 

然后执行`cat /proc/10138/maps | grep libnative-lib.so`，查看应用在内存中的分布。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_R4FZZ97JKJ3DX67.png)

 

我们可以看到 so 文件在内存中分布在两块区域中，`0xd861d000-0xd8636000`和`0xd8637000-0xd863a000`。

 

接着我们写好 python 脚本，使用`readMemory`接口对 so 文件进行 dump。

```
start1 = 0xd861d000
end1 = 0xd8636000
 
start2 = 0xd8637000
end2 = 0xd863a000
 
 
def dump(start,end):
    page_bytes = b''
    for i in range((end-start) // 0x1000 ):
        page_bytes = page_bytes + readMemory(start+i*0x1000, 0x1000)
    return page_bytes
 
so_bytes = b''
so_bytes = so_bytes + dump(start1,end1)
so_bytes = so_bytes + dump(start2,end2)
 
with open("D:\\so_dump.so",'wb') as f:
    f.write(so_bytes)
    f.close()

```

接着点击 Files->Python script，然后找到保存的 python 脚本，点击即可。

 

此时装载到内存中的 so 文件就已经被 dump 下来了，我们使用二进制对比工具进行下对比。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_TDUQV9KJX8X4FAP.png)

 

可以看到中间的是解密部分不同，以及内存中 dump 下来的 so 文件缺少 section 表，如果感兴趣的话可以手动修复。

### 效果验证

然后就到了验证效果的时刻了，我们把 dump 下来的 so 文件拖进 IDA，IDA 会提醒 section 表解析错误，我们略过继续打开，然后搜索`ooxx`函数，可以看到`ooxx`已经被解密了。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_D8G5EBCPZ472DTH.png)

 

好了，到现在我们的测试已经按预想中的完成了。

后记
==

怎么评价 A64dbg？

 

整体评价目前我不太好说，原因是目前我还只是尝试了使用 A64dbg 的 UnicornVM 模式调试下的 ADCpp 脚本系统，进行了简单的安卓逆向测试。

 

但是如果单是讲 ADCpp 脚本系统的话，我只能说 A64dbg 给我带来了焕然一新的 so 层逆向体验，在测试另一个加固 apk 时候，只写了几行 C 代码进行主动调用字符串解密函数，就输出了所有加密字符串，这无疑给静态分析带来了极大的便利，是使用 frida 操作 native 层所不能及的。当然也可能是我的水平火候不够，不能站在一个更高的角度上去对比分析。

 

以及目前我只是测试了主动调用和内存操作的特性，使用 C/C++ 进行 so 注入 hook 还在探索中，但是这几句话你能想到什么呢？我想到了可以直接把安卓源码中的原生头文件包含进来，然后使用 ADCpp 脚本系统去注入 hook，只需要少量并且简洁的 C/C++ 代码就能实现一个脱壳机插件。

 

当然这只是个猜想，因为目前 A64dbg 还不太稳定成熟，不依赖 ptrace 的断点机制多少有些不稳定，以及 A64dbg 仍然缺少大量实战的洗礼，很多 bug 都在埋伏隐藏中，只有在实战中短兵相接才会暴露出来。

 

但是我还是很期待有天 A64dbg 能成熟稳定到支撑起一个脱壳机插件的运行，也希望到时候我有足够强的实力水平来驾驭这个工具，因为我也是在不断使用 A64dbg 进行尝试时想明白了一个道理，工具永远是次要的，最重要的一直都是使用工具的人。

 

论坛[一种通过后端编译优化脱混淆壳的方法](https://bbs.pediy.com/thread-260626.htm)帖子下有个评论让我印象一直很深刻，在这里分享给大家。

 

![](https://bbs.pediy.com/upload/attach/202104/802108_YYV7A9WERAN7872.png)

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)

最后于 22 小时前 被 0x 指纹编辑 ，原因：

上传的附件：

*   [so_crackme.apk](javascript:void(0)) （2.44MB，8 次下载）