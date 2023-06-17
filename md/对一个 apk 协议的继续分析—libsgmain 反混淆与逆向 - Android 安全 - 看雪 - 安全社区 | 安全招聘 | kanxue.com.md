> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277665.htm)

> 对一个 apk 协议的继续分析—libsgmain 反混淆与逆向

对一个 apk 协议的继续分析—libsgmain 反混淆与逆向

43 分钟前 116

### 对一个 apk 协议的继续分析—libsgmain 反混淆与逆向

 [![](http://passport.kanxue.com/upload/avatar/108/802108.png?1574223973)](user-home-802108.htm) [0x 指纹](user-home-802108.htm) ![](https://bbs.kanxue.com/view/img/rank/13.png) 4  ![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif) 43 分钟前  116

前言
==

事情是这样的，两三年前在论坛看到一篇帖子[《对一个 apk 的协议分析》](https://bbs.kanxue.com/thread-258114.htm)，作者`yezheyu`要分析的签名算法是 apk 调用阿里安全组件`libsgmainso-5.4.56.so`中安全签名类`SecuritySignature`的`sign`方法，虽然最后没有还原算法，但`yezheyu`把十分详尽的分析过程写在了帖子中，提供了十分多的参考价值。

 

当时也试着动手分析了下，不过功力水平不够，`JNI_OnLoad`函数的混淆都没办法能给还原下，分析了一段时间就给搁置了。几年过去了，现在各方面水平尤其是汇编分析能力相比以前熟练精进了不少，又重新试着分析下，总的来说还算比较顺利地分析出来了，没有卡着或者在哪个环节花了很多的时间。

复盘总结
====

当然，能分析出来也有两点原因。

 

一来是这几年关于`sgmain`的分析的文章也出现了好几篇了，`krash`前辈在[《使用时间无关调试技术 (Timeless Debugging) 高效分析混淆代码》](https://bbs.kanxue.com/thread-273055.htm) 帖子中写到：

> “以前选择阿里作为自己的研究对象，一是他们的混淆因为有足够的强度和代表性；二是阿里较其他厂要开放些，只进行技术交流分享，不恶意分析应用业务逻辑的文章一般都能正常发出。”

 

得益于阿里安全的包容和大家对技术精益求精的追求和无私的分享精神，给像我这样的后来人提供了很多学习机会和资源。虽然几篇文章都只是技术交流分享不会涉及特定应用业务逻辑，但一来能给自己壮胆去挑战高峰，二来一些思路、技术方法或者分析关键点确实提几句就够了。

 

`我是小三`前辈在[《Anti-Bot 安全 SDK SGAVMP 浅析》](https://bbs.kanxue.com/thread-265017.htm) 帖子中的末尾也写到：

> “缺点：缺少一定的灵活性，比如`bycode`隐藏在图片中，当整个执行逻辑被成功分析清楚后难做即时补就措施，黑产特征在随时变化，本地的特征只要改下名字就可以过掉了。”

 

从某个角度来说，这对安全开发者可能也是一个警示，分析的人总是前仆后继，永不会停下脚步，如果安全产品不能够迅速地调整变化，安全逻辑被分析出来也只是时间问题。攻防形势瞬息万变，双方策略的有效性总只会是一时，而对于安全防守一方来说，调整似乎总会伴随着很多问题，周期也会更长，是一个值得思考的问题。

 

二来是因为分析的`libsgmain`版本是`5.4.56`，版本比较早了。`angelToms`前辈在[《逆向学习 sgvmp 篇》](https://bbs.kanxue.com/thread-259829.htm) 中写到：

> “`sgmain`、`sgavmp`、`sgsecuritybody`等前身是`百川sdk`下的无线保镖，最早由聚安全开发，同时对外对内都提供安全能力，对外提供低版本的`5.x`，对内提供更具安全能力的`6.x`版本；`5.x`版本不具备`avmp`、`litevm`功能，也不具备其他插件的能力。”

 

分析完发现这个版本的`sgmain`保护的话，几乎只有对汇编的乱序混淆，没有上`vmp`，而混淆去除后的分析可以说十分容易了，各种加解密压缩解压算法、从资源中解密出密钥、签名验证等过程，在调试、`hook`和`trace`的分析下毫无抵抗力，可见代码算法的攻防对抗上，作为防守方的安全开发还是要上`vmp`啊: )，不然根本经不住分析。

 

不由得想起来`evilpan`前辈在[《如何破解一个 Python 虚拟机壳并拿走 12300 元 ETH》](https://evilpan.com/2020/10/11/protected-python/)一文中写到的：

> “虚拟机加固 (`VMP`) 是当今很常见的一种代码保护方案，不管是`X86`机器码 (汇编)，安卓的`DEX`字节码还是`Python`字节码，其本质上是从处理器中抢活干，自身在用户空间实现代码执行的状态机，有的还自己实现一套中间指令集。正如伟人所说——世上本没有 `VMP`，对抗得深了，自然就成了`VMP`。”

 

还有就是`sgmain`这个版本的保护在项目工程实践上，不知道是不是开发和安全两部分是分离的，针对逆向分析存在一些缺陷。因为各种加解密压缩解压算法都会跳到一个函数中，里面有个`switch`根据模式来选择分发进行哪种算法，对于分析者来说，意味着只要`hook`这个函数的出入口，所有的加解密输入输出，都能够清晰地获取打印出来，几乎全部的安全保护逻辑都给暴露出来了。

有趣的事情
=====

在我把`yezheyu`帖子中样本的`sgin`分析完成后，因为我和他的分析方式完全不同，我又重新看了一遍他的帖子，发现了一个十分有意思的事情，其实`yezheyu`已经完成了几乎`90%`的分析工作了，如果对那个加密算法十分熟悉的话，这个进度其实是`99%`。。。

 

`yezheyu`在帖子里面最后贴的伪代码有两句是某个加密算法很明显的特征，如果能注意到或者看出来这个，再稍作尝试，这个`sign`算法就能分析出来了，因为这个算法的密钥`yezheyu`也分析到了并且就写在帖子里面。

 

俗话说，行百里者半九十，掘井九仞犹为弃井，机缘巧合之下，我也算是切身见证了一个鲜活的案例，令人唏嘘。

 

不禁想起来四哥`scz`在《留给 Burp 破解爱好者的话》 中留的几个开放式问题中的两个便是：

> (d) 如何在 class 中识别 BASE64、SHA1、SHA256、DES/S 盒、RSA (n,e)，不要只想着 JD-GUI 反编译，想想更困难的场景
> 
> (e) 在没有 DES Key 的情况下，如何用 DES Working Key 完成加解密？已知 DES Decrypt Working Key，你能否反推出 DES Encrypt Working Key？”。

 

对逆向工程来说，密码学基本功还是要稳扎稳打的，各种标准加密烂熟于心，不然一不留神，就会错过很多。

分析思路及工具
=======

**本文只涉及技术层面的交流与分享，更多的是普适的分析思路与方法，评论区和私信请不要做一些徒劳的请求，不会有回复的。**

 

整个分析过程的话，一是反混淆，二是调试 trace 定位加密算法。

 

我使用的工具有：`ida`、`miasm`、`unidbg`。`Miasm`和`unidbg`我之前没用过，在别人的分享中看到用的挺多，这次分析有了想法，从下载编译到看`demo`尝试，到使用解决问题，都没花多少时间，感悟是工具之类的其实都是次要的，重要的是面对问题有没有解决的思路想法，以及项目工程、汇编分析和调试的基本功是否扎实。如果你连自己编译`miasm`和`unidbg`、阅读各自的文档和`example code`的能力都没有，何谈去做更多的事情呢。

 

像`yezheyu`的话，几乎是只使用`IDA`一路逆向撕到了最终的加密算法部分，可以看出逆向基本功十分扎实稳固，后面翻到他在[另一篇帖子](https://bbs.kanxue.com/thread-255088.htm)里面分享过总结的`ARM`汇编指令和`ELF`文件的庞大思维导图，便可知道为何他的调试功底如此好了，可惜密码学功底稍微差了些，离成功只有一步之遥了。

反混淆
===

反混淆的话`yezheyu`没有详细写，给出了参考的一篇文章的链接，[《详细分析一款移动端浏览器安全性》](https://www.anquanke.com/post/id/179080)，这篇文章对`libsgmain`的整个分析过程可以说也十分完备充分了。

 

还有就是`krash`前辈在[《阿里 2015 第二届安全挑战赛第三题题解》](https://bbs.kanxue.com/thread-260507.htm) 的分析了，对混淆的分析和去混淆展示可以说是极其详细了，同时我也是从`krash`前辈这里学到了`miasm`在去混淆中的妙用。

 

如果这两篇文章都看着很吃力的话，可能就需要反思下自己的基础是不是不够牢固了: )

 

`Miasm`符号执行混淆的汇编代码片段，可以搞清楚这段汇编干了什么，是跳转了、是加载了一个常量了、还是只是一段无意义的垃圾片段，然后使用`idapython`脚本几乎可以修复所有混淆的函数，流畅反编译。

 

`krash`前辈没有放`miasm`的使用代码，我来提供一段吧，一个分发块处的混淆，执行结果就是跳转，另一个是函数开头的一段垃圾代码混淆，执行结果相当于`nop`+ 跳转。

```
from miasm.core.locationdb import LocationDB
from miasm.ir.symbexec import SymbolicExecutionEngine
from binascii import unhexlify
from miasm.analysis.machine import Machine
from miasm.analysis.binary import Container
 
def symbolic_execute(asm):
    loc_db = LocationDB()
    c = Container.from_string(asm, loc_db)
    print(c)
    machine = Machine("arml")
    mdis = machine.dis_engine(c.bin_stream, loc_db=loc_db)
    asmcfg = mdis.dis_multiblock(0)
    for block in asmcfg.blocks:
        print(block)
 
    lifter = machine.lifter_model_call(loc_db)
    ircfg = lifter.new_ircfg_from_asmcfg(asmcfg)
    sb = SymbolicExecutionEngine(lifter, machine.mn.regs.regs_init)
    sb.run_at(ircfg, 0)
    print("R0: " + str(sb.symbols[machine.mn.regs.R0]))
    print("R1: " + str(sb.symbols[machine.mn.regs.R1]))
    print("LR: " + str(sb.symbols[machine.mn.regs.LR]))
    print("PC: " + str(sb.symbols[machine.mn.regs.PC]))
 
def symbolic_execute_t(asm):
    loc_db = LocationDB()
    c = Container.from_string(asm, loc_db)
    print(c)
    machine = Machine("armtl")
    mdis = machine.dis_engine(c.bin_stream, loc_db=loc_db)
    asmcfg = mdis.dis_multiblock(0)
    for block in asmcfg.blocks:
        print(block)
    lifter = machine.lifter_model_call(loc_db)
    ircfg = lifter.new_ircfg_from_asmcfg(asmcfg)
    sb = SymbolicExecutionEngine(lifter, machine.mn.regs.regs_init)
    sb.run_at(ircfg, 0)
    print("R0: " + str(sb.symbols[machine.mn.regs.R0]))
    print("R1: " + str(sb.symbols[machine.mn.regs.R1]))
    print("R2: " + str(sb.symbols[machine.mn.regs.R2]))
    print("R3: " + str(sb.symbols[machine.mn.regs.R3]))
    print("R4: " + str(sb.symbols[machine.mn.regs.R4]))
    print("R5: " + str(sb.symbols[machine.mn.regs.R5]))
    print("R6: " + str(sb.symbols[machine.mn.regs.R6]))
    print("LR: " + str(sb.symbols[machine.mn.regs.LR]))
    print("PC: " + str(sb.symbols[machine.mn.regs.PC]))
 
dispatcher_code = unhexlify(
    "03002DE90E10A0E1A110A0E10001A0E1040080E28110A0E1001091E701E08EE

```

执行结果包含对应的汇编代码和执行完后寄存器的状态，从而可以判断混淆代码干了什么。

```
======dispatcher_code=======
 loc_0
STMFD      SP!, {R0, R1}
MOV        R1, LR
MOV        R1, R1 LSR 0x1
MOV        R0, R0 LSL 0x2
ADD        R0, R0, 0x4
MOV        R1, R1 LSL 0x1
LDR        R1, [R1, R0]
ADD        LR, LR, R1
LDMFD      SP!, {R0, R1}
LDR        R0, [SP, 0x8]
STR        LR, [SP, 0x8]
MOV        LR, R0
LDMFD      SP!, {R0, R1, PC}
R0: @32[SP_init]
R1: @32[SP_init + 0x4]
LR: @32[SP_init + 0x8]
PC: LR_init + @32[(LR_init & 0xFFFFFFFE) + (R0_init << 0x2) + 0x4]
======junk code=======
 loc_0
PUSH       {R0-R6, LR}
PUSH       {R0-R6, LR}
MOVS       R6, 0x4
MOVS       R1, 0x2
MOV        R0, SP
ADDS       R0, 0x10
MOVS       R6, 0x2
ADDS       R0, 0x8
ADDS       R1, R0, 0x4
MOV        SP, R1
ADD        R6, PC, 0x18
ADDS       R6, R6, 0x1
MOVS       R1, 0x2
ADDS       R6, 0x28
ADDS       R6, R6, R1
STR        R6, [R0, 0x24]
POP        {R6}
LSLS       R0, R0, 0x0
POP        {R0-R6, PC}
R0: R0_init
R1: R1_init
R2: R2_init
R3: R3_init
R4: R4_init
R5: R5_init
R6: R6_init
LR: LR_init
PC: 0x5B 
```

然后使用`idapython`写脚本进行`patch`修复就好了，详细可以参考`krash`前辈提供的修复脚本。

调试 trace
========

去完混淆后就是调试分析了，调试方面的话我自己更喜欢的方式，是能把执行过程给抽离出来，一次执行`trace`，反复稳定调试鞭尸，没错我说的就是微软的`TTD`（`Time Travel Debugging`），如果对这种调试感兴趣的话可以看看四哥`scz`和`krash`前辈的文章：[《MSDN 系列 (46)--WinDbg Preview TTD 入门》](http://scz.617.cn:8/windows/202201251528.txt) 和 [《使用时间无关调试技术 (Timeless Debugging) 高效分析混淆代码 》](https://bbs.kanxue.com/thread-259829.htm)。

 

安卓方面的话目前好像还没有公开的类似`TTD`的工具，如果有谁知道的话可以评论区说一下告诉我。然后我选择了一个替代方案是使用`unidbg`代码项目来跑出正确的`sign`结果，然后在此基础上进行调试和`trace`。

 

`Unidbg`项目看了下没啥好的文档，不过网上关于 unidbg 使用的文章还是挺多的，一搜大把，自己也是看别人怎么用再自己尝试，就不细说怎么用了。

 

感觉对`unidbg`报错的处理还是比较靠经验的，也没有什么好的全面的文档，碰到错误只能东搜西搜，我就因为没有开启多线程的支持被坑了一下，搜了很久的报错，发现是需要开启多线程的支持。 `unidbg`项目仓库的`issue`里有个问 so 多线程有办法支持吗，回复里说 “支持，请参考 `src/test/java/com/github/unidbg/android/ThreadTest.java`”。

 

怎么定位的加密算法呢？可以从`trace`结果入手，`trace`会记录指令涉及的寄存器中的值，那么直接在`trace`中搜加密结果的部分值，从而能够定位相关函数，再不断地回溯分析。甚至是可以直接在`trace`中搜索分析过程怀疑的标准加密的常量值，有时候也是能够事半功倍的。

最后
==

好了，就写到这吧，再详细也没啥必要了，无非是把上面的提到的几篇文章中说的再重复一遍。

 

再次感谢广大技术爱好者的无私分享精神，给像我这样的后辈提供了很多学习资源，这也是我写此篇文章的重要原因之一。

 

对我个人而言，其中还要尤其感谢`scz`、`evilpan`、`krash`、`我是小三`、`angelToms`几位前辈，没事时候总会反复翻他们的文章琢磨看，不仅是学其中技术知识，也是学他们的心态，觉得他们的水平已经技进乎艺了，但仍然保留着无私分享精神，极其可贵。

 

最后重提一遍，**本文只涉及技术层面的交流与分享，更多的是普适的分析思路与方法，评论区和私信请不要做一些徒劳的请求，不会有回复的。**

  

[VMProtect 分析与还原](https://www.kanxue.com/book-section_list-87.htm)

最后于 10 分钟前 被 0x 指纹编辑 ，原因： [#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#协议分析](forum-161-1-120.htm) [#混淆加固](forum-161-1-121.htm)