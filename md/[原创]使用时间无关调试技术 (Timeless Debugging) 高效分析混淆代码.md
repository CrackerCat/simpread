> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273055.htm)

> [原创]使用时间无关调试技术 (Timeless Debugging) 高效分析混淆代码

同志们，我又来搞阿里的混淆了。开个小玩笑，不能老搞阿里。以前选择阿里作为自己的研究对象，一是他们的混淆因为有足够的强度和代表性；二是阿里较其他厂要开放些，只进行技术交流分享，不恶意分析应用业务逻辑的文章一般都能正常发出。不过它作为我常驻的测试用例，后面还是有点戏份。

 

代码混淆是个令很多人头疼不已的问题，一个简单的流程平坦混淆就能急剧降低我们逆向分析效率，一个间断跳转混淆，就能废掉 IDA 等静态分析工具的武功，VM 防护更是另大多数选手束手无策。各个大厂，安全厂商都有自己的混淆器，阿里甚至有几套 VM 防护。各种混淆方案，加上编译器优化使得我们很难写出通用的反混淆算法。

 

能否有一个通用的，不需进行反混淆就能直接分析我们关注的应用逻辑的方法呢？我将会在本文分享我的尝试。

 

我这里使用的技术称为时间无关调试。我理解的时间无关调试就是记录程序执行过程中的寄存器，和内存变化，使用记录的 trace 离线调试分析的过程，简而言之记录 trace，分析 trace。最初的 qira(https://github.com/geohot/qira)，微软的 TTD(Time Travel Debugging)，mozilla 的 record-replay debugging(https://github.com/rr-debugger/rr) 他们本质都是一样的，感兴趣的朋友可以自行搜索相关资料。

 

我的时间无关调试器分为两部分，trace 记录器和 trace 分析器。记录器的相关信息可以在我上个帖子找到，分析器也就是我们的调试界面。trace 分析功能和调试界面最近刚完工，经过简单的测试后就有点儿想迫不及待的想试试它的威力了。这次选用的样本是看雪论坛 2021 年 11 月份 3w 班的一道题目，题目具体信息见 (https://bbs.pediy.com/thread-270220.htm)。

 

先秀工具。

工具展示
====

指令流视图
-----

![](https://bbs.pediy.com/upload/attach/202205/240967_ZPJHZBX5MJKDF27.png)

寄存器视图
-----

![](https://bbs.pediy.com/upload/attach/202205/240967_2DBZES523VSGETE.png)

 

与实时调试器类似的寄存器试图。

内存视图
----

![](https://bbs.pediy.com/upload/attach/202205/240967_BTHEZ74V2QJSAJ9.png)

 

没错，我们可以像实时调试器一样浏览任意程序点，任意地址的内存内容。有朋友可能会注意到，为什么内存会有 "??" 显示？这是因为这块内存在 trace 的过程中未被访问，我只在内存被使用（读写）的时候才会记录其内容。在其中双击任意已知内容，可以在指令流视图中跳转到他的定义，一般为写入它的指令。

交叉引用视图
------

![](https://bbs.pediy.com/upload/attach/202205/240967_T4UQ42K9EPK4HZY.png)

 

交叉引用视图显示当前指令的寄存器，内存交叉引用。"<-" 是使用定义链，表示某条指令定义的值会被当前指令使用。"->" 是定义使用链，表示当前指令定义的值的使用者。内存交叉引用显示当前指令读写的内存地址，  
对于读内存会在其子树显示写入指令的编号，相反的，写内存地址则会显示它的使用者。在这例子中，"r 4 0000007ff3e99d98" 表示当前指令会读取 "0000007ff3e99d98" 处四字节内容，"00010679 w 4 0000007ff3e99d98" 表示它是由编号为 "00010679" 的指令写入。

 

以上四个基本功能是不是已经可以极大提升逆向效率了？我们还有可以让生活更美好的功能。

字符串参考
-----

![](https://bbs.pediy.com/upload/attach/202205/240967_EC6V8UJ9TEZ64F9.png)

 

杀手级功能！字符串引用作用无需多言。我会分析 trace 时内存出现过的所有字符串，直接秒杀所有字符串加密防护。

污点追踪
----

另外一个杀手锏！正向污点追踪 (Forward Taint Analysis) 能标记出受输入影响的相关指令，而逆向 (Backward Taint Analysis) 污点追踪可帮我们自动回溯变量来源和相关的计算过程。在逆向分析过程中，在对感觉兴趣的寄存器或者内存进行标记之后，使用正向污点追踪可以自动的帮我们找到他们的处理过程。

 

假设在调试中，发现了某条感兴趣的指令，比如说下图的，"03186869 str，x1, [x24, x0, LSL #0x3]" 时，现在想知道 "x1" 后续如何被处理的，该如何入手？可以使用上面介绍的内存引用，找到他的使用者，然后继续分析它的使用。也可以对 x1 进行一次正向的污点追踪分析，污点引擎能自动筛选出受 x1 影响的指令。

 

![](https://bbs.pediy.com/upload/attach/202205/240967_77ZVYN3WJJV7MSN.png)

 

可以看到，在污点分析结果中，x1 会先在内存中来回倒了几次，最后会在 free 中使用，应该是释放 malloc 返回的一个指针。

 

如果想知道另外一个寄存器 x0 的来源呢？同样可以使用交叉引用和逆向污点追踪。逆向污点追踪分析结果如下：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_R9PB6T6AVYXU3EB.png)

 

很显然，x0 依赖 memcpy 复制的一块内存，是 02262601 处 ubfx 执行结果。

 

有朋友能看出这是个 VM 吗？这个例子是我在阿里 10101 命令 trace 中随手取的。在分析 x1 的后续使用过程中，污点追踪引擎自动屏蔽了 VM 解释器取指，解码，执行等各种细节，直接分析出会处理 x1 相关指令。而 x0(=8) 则是 VM 解释器的寄存器参数，对 VM 实现感兴趣可以研究它的来源，解码等过程。

 

以为就是我的时间无关调试器的核心功能。仅依赖于 trace 指令，理论上对任意代码防护均适用。接下来介绍依赖函数分析相关的功能。

调用栈
---

使用调用栈我们可以快速跳转到上层调用者，考察调用参数。

控制流图
----

这是所有功能里面实现难点最多，同时也是最有趣的一部分。为什么要自己恢复和绘制 CFG 呢？自己恢复 CFG 的首要原因就是对抗间接跳转混淆。静态分析间接跳转的跳转目标需要很高的技巧，常见的方法有

*   unicron 模拟执行
*   动态符号执行（DSE）或者静态符号执行（SSE）
*   值集分析（VSA）

而从 trace 中重建 CFG 则简单很多。另外一个原因是代码动态修改和映射。像代码加壳，不使用调试器，我们一般只能使用静态工具分析壳代码；一些防护会动态 mmap 代码，执行之后 unmap，如果我们支持 trace 重建 CFG 的话就能免去调试，dump 内存等步骤，支持对他们进行分析。事实上重建 CFG 也有些难点，有些情况甚至比静态更难处理。考虑如下两种情形：

*   inline hook 之后导致原函数 CFG 变化。
*   代码加壳。如果壳代码跟原始代码运行在相同的地址空间，那么也会出现同一地址不同时间点运行的指令不一样的情况。

在实现的时候我已经考虑这些情况，但还未找到好的处理办法。

 

最难的一部分来了，绘制 CFG。我认为传统的层次布局（Sugiyama Layout）用于调试有以下缺陷：

*   无法反应调试进度。很多时候我们想知道函数规模，当前已经调试了多少代码，还有多久能执行到函数返回，都需要参考伪码，而伪码经过反编译器优化之后可能又跟汇编对应不上。
*   调试中难以得知是当前否位于循环内，跳转是否会进入或者跳出循环。
*   调试大函数（基本块数量较多）。

为了更好的支持调试，我这里参考了 Ghidra 的实现，目标是为了实现一种 “伪” 源码调试效果。

 

Ghidra 的 CFG 很有特色，他们称之为 Decompiler Layout，即使用反编译器提供的代码块顺序和缩进信息对 CFG 布局，这种布局的结果自然而然的接近反编译后的源码，符合我们的需求。  
通用的反编译器结构化算法目标一般是生成更结构化的代码，更少的 goto 语句。Ghidra 的结构化算法就会选择一些不是强连通分量的块作为循环成员，将函数提前返回等手段减少 goto 数量。我目的是为了对 CFG 布局，我觉得应该：

*   强连通分量应该绘制在一起，这样可以很快识别图中的连通分量。
*   函数返回块应该固定于最底层，这样就很容易识别出函数是否会返回，定位到返回块。传统层次布局算法为了布局美观而减少边的长度，这时算法会尽可能将块往上（函数入口方向）摆放，这会导致返回块位置不确定。对函数返回值进行溯源是一个非常常见的逆向需求，能快速定位到返回块有助于我们回溯函数返回值。另外一个优点就是能反映调试进度，在函数只有一个出口的情况下，这种实现函数绝不会从 CFG 中间返回，调试的整体过程也一定是向最底层的块行进。
*   CFG 中的边也应该尽可能的向下跳转，避免和循环的回边混淆。

因此，CFG 布局使用的是我自己的结构化算法。该算法不关心块内指令的语义，而是会直接假定两路跳转都是 if-else 跳转，两路以上的跳转是 switch-case 跳转来进行结构化分析。

 

下面是我跟 Ghidra 绘制的简单对比：

 

ghidra 布局：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_QTKY29VPEEV7YNT.png)

 

我的布局：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_D7TETRN7KM8HX3B.png)

 

阿里间接跳转混淆还原：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_EF3YVSA8U5PPKGY.png)

 

我还实现了一个与上面结构化布局配套的块导航图，应该算是一个小创新，独一无二的功能吧。它看起来是这样的：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_4XJQD7YH8AKAVYX.png)

 

它可以：

*   可视化调试进度。
*   方便的进行块跳转。
*   识别图中循环头（高亮的黄色块）。
*   评估函数规模、循环数量、复杂度等。块密度越大，表明函数规模越大；越 “花” 则表明函数越复杂，这是因为我使用不同颜色绘制不同循环，不同颜色不同缩进等级。

至此，调试器的功能已完全介绍完毕。

 

下面开始我们的实战。样本 libnative-lib.so 基址：0x750bd42000。

实战
--

首先进行第一步，也是最困难的一步，记录算法的执行过程。KanxueSign 函数的 trace 约 42w 条指令，记录文件大小 6.42MB，在我的 pixel 3 上 trace 耗时小于 500 毫秒。我的时间无关调试器是按 “亿” 级别的规模设计的，42w 对我而言个是非常微小的 trace，当前我测试过最大的 trace 约 9000w 条（1.12GB），阿里 10101 命令的 trace 有 1100w 条（139MB）。

 

点击 check 按钮之后，logcat 中会有如下打印，我们目标就是还原计算 output 的算法。

```
KANXUE  : input: nisaiiA5fyk8Raylj8HZ7inewAY5pfXGbB6M output: 9b9da9c850fd4564563b15267f6586c71d008ca3677c19e9d0201df29abec0b4019500be012b0204012b0195012b009e009e019501110071022a00c30156012b0190014b02e201a900c30071023e023e02a2022a02d9019000be015602d900ca00ca014a00be0211016b0297014a023a00c9011c02df01110086016b011c02b100a7014b02d902df00d4017301d10147009b009b019500bd012b011c01a900c3012b009e0156ahuLa65wbNaSn6iLc34KmhzxnxqMa0==
```

输入输出都是字符串，在字符串视图中先搜下输入，内存中出现过两次，经过简单分析之后，貌似都没有后续使用，所以我们这里选择输出作为切入点开始进行分析。

 

![](https://bbs.pediy.com/upload/attach/202205/240967_53JKPXQYU745UMQ.png)

 

使用输出第一字节 "9b" 进行搜索，找到创建它的位置：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_B8YU89JW7XTDKP5.png)

 

在执行 “00180674 75f67c41b0 strb w8, [x0, x9, LSL]” 指令后，内存 0x7fec60ebd9 中将会首次完整出现 "9b9da9c850fd456456"。

 

顺藤摸瓜，在内存视图中选中第一个字节进行逆向污点追踪，找到它的计算过程。

 

![](https://bbs.pediy.com/upload/attach/202205/240967_5JYYJH6TWY6KKEC.png)

 

上方的 “__vfprintf” 函数表明输出可能是执行格式化打印后的结果。将代码执行到“00163000”，查看调用栈：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_CEMV3C7ZRPERQ8W.png)

 

可以看到程序会使用 sprintf 格式化字符串。

 

将程序执行到此处，

 

![](https://bbs.pediy.com/upload/attach/202205/240967_XYH7D5AKMGJUEJU.png)

 

此时 x2 的值正是 "9b"。注意此时内存 "000000750bd9eb04" 仍显示为未知，原因是程序运行到该处时，我们还未曾记录到对它的访问，在 sprintf 调用返回之后，  
重新查看就可以看到它的内容是 “%02x”。

 

![](https://bbs.pediy.com/upload/attach/202205/240967_TG78FW4XCVRG5A3.png)

 

在指令流中追踪 x2 定义：

 

发现 x2 是 “00162718” 处的 ldrb 指令读取 0000007ff3e99fe0 的一个字节内容。

 

![](https://bbs.pediy.com/upload/attach/202205/240967_KWMFAPW3H8BJY73.png)

 

在 CFG 视图中使用 “alt + 单击” 尝试选择 “750bd55400” 所在基本块所属循环，发现能成功，并且循环只有一个出口。

 

考察循环退出条件：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_4YQ4H8MV5ZUQ2Q5.png)

 

很明显 x23 是个计数器，循环会在执行 32 次后退出。

 

将程序执行到 “00162718”：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_JWSAXQ6PTDAWEUR.png)

 

不难得出，该循环把 0000007fec60ecf0 处，32 字节内存内容输出为 16 进制，正好对上 output 的前 32 字节。

 

接下来继续研究 “0000007fec60ecf0” 内存数据来源，它必然是由保存在内存某处的输入经过计算而来。  
为了找到这输入，我们只需要不断对依赖的 ldr 指令的目标寄存器进行逆向污点追踪即可。

 

发现以下路径能回溯到常量：

```
# 00161267 750bd5dc3c ldr w13, [x8]
  # 00161018 750bd62a84 ldr w8, [x10]
    # 00044254 750bd62a84 ldr w8, [x10] (loop)
      # 00011917 750bd5d238 ldr d1, [x13, #0x290]
```

![](https://bbs.pediy.com/upload/attach/202205/240967_XZY5A8WXMK8TGFM.png)

 

经过搜索，发现这是个用于计算 sha256 常量。此时发现 x1 指向一个字符串（"1810ab738de1810a9cf720"），难道是计算它的 sha256？经过验证，发现并不是。

 

看来有必要研究下 sha256 的算法过程。为了找到 sha256 相关例程，对 0000007fec60ed10 中的 sha256 ctx 进行正向污点追踪，

 

![](https://bbs.pediy.com/upload/attach/202205/240967_N9DDGCJ6SBC5764.png)

 

发现首次使用 sha256 ctx 的函数是 0001d1e8，运行到此处发现 x0 指向 sha256 ctx，此时 x1 是个字符串（"mdml=>kod89mdml=e?:knl\\\\\\\\\\\\\\\\\\\\\"），难道最终的 hash 是它的？  
经过验证，也不是。难道不是标准的 sha256？为了研究 sha256 的运行细节，我在随便网上找了一份 sha256 代码 [https://github.com/System-Glitch/SHA256.git](https://github.com/System-Glitch/SHA256.git)，在 [Main loop](https://github.com/System-Glitch/SHA256/blob/30960208691da7a69f550e3e1f0f09f76b18cdd1/src/SHA256.cpp#L80) 每轮迭代前后和 transform 函数返回前打印 ctx。

 

发现使用相同参数运行标准 sha256 transform 之后的 ctx 与 0001d1e8 完全一致，所以 0001d1e8 是标准的 sha256 transform。

 

使用指令执行历史功能，发现这个函数会执行 5 次。

 

![](https://bbs.pediy.com/upload/attach/202205/240967_FZWG92YGA4Y2XYY.gif)

 

其中第二次会使用新的 ctx_b 对如下数据进行转换：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_YNSBSVDRNQRKW4K.png)

 

第三次，四次使用 ctx_b 对 apk 路径进行转换

 

最后一次使用最初的 ctx 转换 ctx_b hash 进行转换。

 

最后一次执行后的 ctx 即是我们最终的 hash，只是字节序有差异。

 

对第二次调用的输入首字节 “07” 进行逆向污点追踪：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_WRHA9H4NQBRUN6A.png)

 

分析后发现他是经过首次运行的参数（"mdml=>kod89mdml=e?:knl"）原地转换而来。  
计算过程如下：

```
*v28 = (~(_BYTE)v29 & 0xEA | v29 & 0x15) ^ 0x80;
```

化简后就是

```
*v28 ^= 0x6a;
```

向上考察污点追踪结果即可得到 "mdml=>kod89mdml=e?:knl" 的来源，发现它依赖参数 "1810ab738de1810a9cf720"，

 

![](https://bbs.pediy.com/upload/attach/202205/240967_XUFNT5CVBVZ3GNK.png)

 

逐字节通过：

```
v27 = (~(unsigned __int8)*v26 & 0xB5 | *v26 & 0x4A) ^ 0xE9;
```

也就是

```
v27 ^= 0x5c;
```

运算得到。

 

最后我们只要弄清参数 "1810ab738de1810a9cf720" 的来源，output 第一个部分的分析就大功告成了。

 

接下来的操作大家应该都知道，污点追踪，通过栈视图跳转到调用者：

 

![](https://bbs.pediy.com/upload/attach/202205/240967_HFJCW4VXXQ8NTTP.png)

 

"1810ab738de1810a9cf720" = sprintf("%08lx%08lx", 0x000001810ab738de, 0x000001810a9cf720)

 

继续分析 x2 来源，

 

![](https://bbs.pediy.com/upload/attach/202205/240967_3VWQC2DJR5A48QZ.png)

 

考察调用者参数的操作可以得到:

 

0x000001810ab738de 是使用 GetStaticLongField 获取的 startTime

 

0x000001810a9cf720 是使用 GetStaticLongField 获取的 firstInstallTime

 

综上，output 第一部分的计算如下：

```
# part 1
s0 = "{:08x}{:08x}".format(start_time, first_install_time)
s0 = (s0 + (64 - len(s0)) * chr(0)).encode() 
 
sha_a = hashlib.sha256()
s1 = bytes([c ^ 0x5c for c in s0])
sha_a.update(s1)
 
sha_b = hashlib.sha256()
s2 = bytes([c ^ 0x6a for c in s1])
sha_b.update(s2)
sha_b.update(package_code_path.encode())
 
sha_a.update(sha_b.digest())
part1 = sha_a.hexdigest()
```

同样的分析套路可以分析出第二，三部分的算法，这里略过冗长的具体分析过程，直接给出最终算法。

 

part 2:

 

使用 randomLong 查表转换 packageCodePath 而来

```
# part 2
part2 = ''
for c in package_code_path:
    part2 += '{:04x}'.format(dword_5C008[random_long % 5 + ord(c)])
```

part 3:

```
s0 = "{:08x}{:08x}".format(start_time, first_install_time)
part3 = base64.b64encode(s0.encode())
std_base64_chars = b'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
ollvm11_base64_chars = b'0123456789_-abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
part3 = part3.translate(bytes.maketrans(std_base64_chars, ollvm11_base64_chars))
```

总结
--

上文展示了使用时间无关调试技术对算法进行一次完整逆向还原的过程。此样本最大的弱点就是使用了标准的 sha256，在识别出它使用的是标准算法后，节省了我们大量分析混淆代码和实现等价算法的时间，这部分计算只需要使用相同的输入调用标准算法即可。在识别算法时，使用时间无关调试器可以方便在函数级，指令级与标准算法流程之间进行比较，使用正向污点追踪甚至可以直接比较处理输入的完整过程。污点追踪，这个听起来就很高级的技术，在自动化反混淆论文中经常可以看到它的身影，但是貌似很少看到过用它，特别是反向污点追踪进行辅助逆向分析文章，它自动化追踪和溯源能力能将我们从枯燥的单步调试，人肉回溯解脱出来。有了污点引擎的加持，逆向回溯指令操作参数来源，然后正向分析其使用的逆向策略实施起来将会更加简单，直接。未来会不会出现污点追踪的对抗，大家可以拭目以待。

参考资料
----

[对 ollvm 的算法进行逆向分析和还原](https://bbs.pediy.com/thread-270529-1.htm)

 

附件包含样本和完整源码。  
游客朋友也可以从我的 github 下载：[reverseme-ollvm11](https://github.com/amimo/pediy-reverseme-ollvm11)

[【看雪培训】《Adroid 高级研修班》2022 年春季班招生中！](https://bbs.pediy.com/thread-271992.htm)

最后于 8 小时前 被 krash 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [ollvm11.py](javascript:void(0)) （5.96kb，8 次下载）
*   [ollvm11.apk](javascript:void(0)) （1.69MB，6 次下载）