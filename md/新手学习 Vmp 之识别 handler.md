> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1814843-1-1.html)

> [md]## 新手学习 Vmp 之利用 ghidra 识别 handler 最近也是在研究 vmp，记录一下过程，没什么技术含量，仅提供一种思路作为参考。

![](https://avatar.52pojie.cn/data/avatar/000/48/36/37_avatar_middle.jpg)fjqisba _ 本帖最后由 fjqisba 于 2023-7-29 16:49 编辑_  

新手学习 Vmp 之利用 ghidra 识别 handler
------------------------------

最近也是在研究 vmp，记录一下过程，没什么技术含量，仅提供一种思路作为参考。

这里还是以 vmp2 为例，为什么研究 vmp2 而不是 vmp3 呢？一方面我认为是要循序渐进，在掌握了 vmp2 的基础上，再去分析 vmp3 应该是能够容易不少的，另一方面也不排除有软件依旧会采用 vmp2 进行加密。

之前分析 vmp 用 unicorn 动态跟踪生成了一个动态流程图，可以参考上一篇帖子 https://www.52pojie.cn/thread-1798622-1-1.html

vmp2 在画出控制流程图后架构基本一目了然，通过一些简易的规则很快就能识别出 handler 块、handler 分发块这些东东，这里就不谈了。

对于识别出 handler，我认为关键点在于能对 handler 中的无用指令进行**反混淆**，在提炼出核心指令后，就好处理了。这里我并没有使用符号执行 (主要是没发现好用的 C++ 符号执行库额)，而是采用另一款开源的 ghidra 反编译库。

Ghidra 设计了一套中间指令，能够将汇编指令转换为中间码，并对其进行 SSA、死代码消除等一系列化简操作，这和反混淆本质上是一样的，我们只需对其稍加进行改造，像 vmp 这种乱七八糟的二进制指令，Ghidra 也能处理。

例如 vmp handler 有一个 vAdd4 指令，原始汇编如下:

```
004D4DD9    66:F7D0         not ax 
004D4DDC    66:0FBDC1       bsr ax,cx
004D4DE0    8B45 00         mov eax,dword ptr ss:[ebp]
004D4DE3    60              pushad
004D4DE4    66:F7C4 C21F    test sp,0x1FC2
004D4DE9    83EC E0         sub esp,-0x20
004D4DF2    0145 04         add dword ptr ss:[ebp+0x4],eax
004D4DF5    60              pushad
004D4DF6    9C              pushfd
004D4DF7    885C24 04       mov byte ptr ss:[esp+0x4],bl
004D4DFB    9C              pushfd
004D4DFC    8F4424 20       pop dword ptr ss:[esp+0x20]
004D4E00    882424          mov byte ptr ss:[esp],ah
004D4E03    FF7424 20       push dword ptr ss:[esp+0x20]
004D4E07    8F45 00         pop dword ptr ss:[ebp]
004D4E0A    66:C74424 10 A5 mov word ptr ss:[esp+0x10],0x71A5
004D4E11    60              pushad
004D4E12    9C              pushfd
004D4E13    C70424 2BFA35ED mov dword ptr ss:[esp],0xED35FA2B
004D4E1A    8D6424 48       lea esp,dword ptr ss:[esp+0x48]

```

Ghidra 反编译后生成如下 Pcode:

```
0x004d4de0:1:        u0x00007a00(0x00000002:1) = *(ram,EBP(i))
0x004d4df2:162:        u0x00001d00(0x00000007:162) = EBP(i) + #0x1(*#0x4)
0x004d4df2:2f:        u0x00007a00(0x00000007:2f) = *(ram,u0x00001d00(0x00000007:162))
0x004d4df2:30:        CF(0x00000007:30) = CARRY4(u0x00007a00(0x00000007:2f),u0x00007a00(0x00000002:1))
0x004d4df2:31:        u0x00007a00(0x00000007:31) = *(ram,u0x00001d00(0x00000007:162))
0x004d4df2:32:        OF(0x00000007:32) = SCARRY4(u0x00007a00(0x00000007:31),u0x00007a00(0x00000002:1))
0x004d4df2:33:        u0x00007a00(0x00000007:33) = *(ram,u0x00001d00(0x00000007:162))
0x004d4df2:34:        u0x00007a00(0x00000007:34) = u0x00007a00(0x00000007:33) + u0x00007a00(0x00000002:1)
0x004d4df2:35:        *(ram,u0x00001d00(0x00000007:162)) = u0x00007a00(0x00000007:34)
0x004d4df2:36:        u0x00007a00(0x00000007:36) = *(ram,u0x00001d00(0x00000007:162))
0x004d4df2:163:        u0x10000080(0x00000007:163) = (cast) u0x00007a00(0x00000007:36)
0x004d4df2:37:        SF(0x00000007:37) = u0x10000080(0x00000007:163) < #0x0
0x004d4df2:38:        u0x00007a00(0x00000007:38) = *(ram,u0x00001d00(0x00000007:162))
0x004d4df2:39:        ZF(0x00000007:39) = u0x00007a00(0x00000007:38) == #0x0
0x004d4df2:3a:        u0x00007a00(0x00000007:3a) = *(ram,u0x00001d00(0x00000007:162))
0x004d4df2:3b:        u0x0000dc80(0x00000007:3b) = u0x00007a00(0x00000007:3a) & #0xff
0x004d4df2:3c:        u0x0000dd00:1(0x00000007:3c) = POPCOUNT(u0x0000dc80(0x00000007:3b))
0x004d4df2:3d:        u0x0000dd80:1(0x00000007:3d) = u0x0000dd00:1(0x00000007:3c) & #0x1:1
0x004d4df2:3e:        PF(0x00000007:3e) = u0x0000dd80:1(0x00000007:3d) == #0x0:1
0x004d4dfb:94:        u0x0000b400:1(0x0000000b:94) = NT(i) & #0x1:1
0x004d4dfb:95:        u0x0000b480(0x0000000b:95) = ZEXT14(u0x0000b400:1(0x0000000b:94))
0x004d4dfb:96:        u0x0000b500(0x0000000b:96) = u0x0000b480(0x0000000b:95) * #0x4000
0x004d4dfb:98:        u0x0000b600(0x0000000b:98) = ZEXT14(OF(0x00000007:32))
0x004d4dfb:99:        u0x0000b680(0x0000000b:99) = u0x0000b600(0x0000000b:98) * #0x800
0x004d4dfb:9a:        u0x0000b700(0x0000000b:9a) = u0x0000b500(0x0000000b:96) | u0x0000b680(0x0000000b:99)
0x004d4dfb:9f:        u0x0000b980:1(0x0000000b:9f) = IF(i) & #0x1:1
0x004d4dfb:a0:        u0x0000ba00(0x0000000b:a0) = ZEXT14(u0x0000b980:1(0x0000000b:9f))
0x004d4dfb:a1:        u0x0000ba80(0x0000000b:a1) = u0x0000ba00(0x0000000b:a0) * #0x200
0x004d4dfb:a2:        u0x0000bb00(0x0000000b:a2) = u0x0000b700(0x0000000b:9a) | u0x0000ba80(0x0000000b:a1)
0x004d4dfb:a3:        u0x0000bb80:1(0x0000000b:a3) = TF(i) & #0x1:1
0x004d4dfb:a4:        u0x0000bc00(0x0000000b:a4) = ZEXT14(u0x0000bb80:1(0x0000000b:a3))
0x004d4dfb:a5:        u0x0000bc80(0x0000000b:a5) = u0x0000bc00(0x0000000b:a4) * #0x100
0x004d4dfb:a6:        u0x0000bd00(0x0000000b:a6) = u0x0000bb00(0x0000000b:a2) | u0x0000bc80(0x0000000b:a5)
0x004d4dfb:a8:        u0x0000be00(0x0000000b:a8) = ZEXT14(SF(0x00000007:37))
0x004d4dfb:a9:        u0x0000be80(0x0000000b:a9) = u0x0000be00(0x0000000b:a8) * #0x80
0x004d4dfb:aa:        u0x0000bf00(0x0000000b:aa) = u0x0000bd00(0x0000000b:a6) | u0x0000be80(0x0000000b:a9)
0x004d4dfb:ac:        u0x0000c000(0x0000000b:ac) = ZEXT14(ZF(0x00000007:39))
0x004d4dfb:ad:        u0x0000c080(0x0000000b:ad) = u0x0000c000(0x0000000b:ac) * #0x40
0x004d4dfb:ae:        u0x0000c100(0x0000000b:ae) = u0x0000bf00(0x0000000b:aa) | u0x0000c080(0x0000000b:ad)
0x004d4dfb:af:        u0x0000c180:1(0x0000000b:af) = AF(i) & #0x1:1
0x004d4dfb:b0:        u0x0000c200(0x0000000b:b0) = ZEXT14(u0x0000c180:1(0x0000000b:af))
0x004d4dfb:b1:        u0x0000c280(0x0000000b:b1) = u0x0000c200(0x0000000b:b0) * #0x10
0x004d4dfb:b2:        u0x0000c300(0x0000000b:b2) = u0x0000c100(0x0000000b:ae) | u0x0000c280(0x0000000b:b1)
0x004d4dfb:b4:        u0x0000c400(0x0000000b:b4) = ZEXT14(PF(0x00000007:3e))
0x004d4dfb:b5:        u0x0000c480(0x0000000b:b5) = u0x0000c400(0x0000000b:b4) * #0x4
0x004d4dfb:b6:        u0x0000c500(0x0000000b:b6) = u0x0000c300(0x0000000b:b2) | u0x0000c480(0x0000000b:b5)
0x004d4dfb:b8:        u0x0000c600(0x0000000b:b8) = ZEXT14(CF(0x00000007:30))
0x004d4dfb:ba:        eflags(0x0000000b:ba) = u0x0000c500(0x0000000b:b6) | u0x0000c600(0x0000000b:b8)
0x004d4dfb:bb:        u0x0000c780:1(0x0000000b:bb) = ID(i) & #0x1:1
0x004d4dfb:bc:        u0x0000c800(0x0000000b:bc) = ZEXT14(u0x0000c780:1(0x0000000b:bb))
0x004d4dfb:bd:        u0x0000c880(0x0000000b:bd) = u0x0000c800(0x0000000b:bc) * #0x200000
0x004d4dfb:be:        u0x0000c900(0x0000000b:be) = eflags(0x0000000b:ba) | u0x0000c880(0x0000000b:bd)
0x004d4dfb:bf:        u0x0000c980:1(0x0000000b:bf) = VIP(i) & #0x1:1
0x004d4dfb:c0:        u0x0000ca00(0x0000000b:c0) = ZEXT14(u0x0000c980:1(0x0000000b:bf))
0x004d4dfb:c1:        u0x0000ca80(0x0000000b:c1) = u0x0000ca00(0x0000000b:c0) * #0x100000
0x004d4dfb:c2:        u0x0000cb00(0x0000000b:c2) = u0x0000c900(0x0000000b:be) | u0x0000ca80(0x0000000b:c1)
0x004d4dfb:c3:        u0x0000cb80:1(0x0000000b:c3) = VIF(i) & #0x1:1
0x004d4dfb:c4:        u0x0000cc00(0x0000000b:c4) = ZEXT14(u0x0000cb80:1(0x0000000b:c3))
0x004d4dfb:c5:        u0x0000cc80(0x0000000b:c5) = u0x0000cc00(0x0000000b:c4) * #0x80000
0x004d4dfb:c6:        u0x0000cd00(0x0000000b:c6) = u0x0000cb00(0x0000000b:c2) | u0x0000cc80(0x0000000b:c5)
0x004d4dfb:c7:        u0x0000cd80:1(0x0000000b:c7) = AC(i) & #0x1:1
0x004d4dfb:c8:        u0x0000ce00(0x0000000b:c8) = ZEXT14(u0x0000cd80:1(0x0000000b:c7))
0x004d4dfb:c9:        u0x0000ce80(0x0000000b:c9) = u0x0000ce00(0x0000000b:c8) * #0x40000
0x004d4dfb:ca:        eflags(0x0000000b:ca) = u0x0000cd00(0x0000000b:c6) | u0x0000ce80(0x0000000b:c9)
0x004d4e07:d9:        *(ram,EBP(i)) = eflags(0x0000000b:ca)

```

可以说是几乎很完美地去除了无效的指令了额，看不懂没关系，我们直接提出上面 Pcode 中的出现的地址和其对应的指令:

```
004D4DE0    8B45 00         mov eax,dword ptr ss:[ebp]
004D4DF2    0145 04         add dword ptr ss:[ebp+0x4],eax
004D4DFB    9C              pushfd
004D4E07    8F45 00         pop dword ptr ss:[ebp]

```

这样会清楚一些，针对上面生成的结果，那么识别 Handler 就有两种识别方案，复杂一点的是编写 Pcode 规则进行识别，通过遍历 Pcode 来追踪值之间的关系再进行特征匹配；追求简单点就直接对汇编指令进行规则识别，不过缺点就是部分关键汇编指令也可能被 Ghidra 优化掉，会丢失一些准确度，但是应该也是能满足大部分场景了。

这里就介绍一下采取简单方案大致的识别步骤，我们将化简的指令再进行细分，例如

mov xxx,dword ptr ss:[ebp]，这类指令可标记为 ReadVmStack0_4，后面的两个数字，一个是代表 ebp 的寄存器偏移，一个是代表大小。

add dword ptr ss:[ebp+0x4],xxx，这类指令可标记为 Add_Ebp4_4

由于我们做标记的指令都是化简之后的指令，已经是关键指令了，因此当出现这两个标记的时候，即可直接认为该 vmp handler 为 vAdd4![](https://avatar.52pojie.cn/images/noavatar_middle.gif)bamboo52 学到了！！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)moyanblade112 学到了！![](https://avatar.52pojie.cn/data/avatar/000/66/98/22_avatar_middle.jpg)梁茵 感谢楼主分享，对于小白的我来说可以增长一下知识![](https://static.52pojie.cn/static/image/smiley/default/17.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) sir2008 感谢楼主分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif)wuyedebingyu 最近也在研究。。学到了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) yaowenshuo123 学到新知识了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) tianyu925 可以可以 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) XDev136 学习了！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)nj2004 学习高手的经验！感谢分享