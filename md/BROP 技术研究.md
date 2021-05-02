> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Old4dKS2aDp1TETTn0WzoQ)

本文作者  Strawberry @ QAX A-TEAM

![](https://mmbiz.qpic.cn/mmbiz_gif/EkibxOB3fs4icwQQAZE6MBepadE7zAutkviaEmicgZWqGCPAvRDxD3EhVvrLJQckeqTGqC7Hmc08MTUxXeaMq5pVXw/640?wx_fmt=gif)

Blind Return-oriented programming（BROP）是 2014 年 IEEE Symposium on Security and Privacy 中的一篇论文提出的漏洞利用思路，其可以在没有源码和二进制程序的情况下针对栈溢出漏洞进行自动化攻击。该思路假设服务器程序在崩溃后使用 fork 重启，在这种情况下，即使程序开启 ASLR、NX 以及 Canary 保护也可以利用成功。

![](https://mmbiz.qpic.cn/mmbiz_gif/EkibxOB3fs4icwQQAZE6MBepadE7zAutkviaEmicgZWqGCPAvRDxD3EhVvrLJQckeqTGqC7Hmc08MTUxXeaMq5pVXw/640?wx_fmt=gif)

声明：本篇文章由 Strawberry @ QAX A-TEAM 原创，仅用于技术研究，不恰当使用会造成危害，严禁违法使用 ，否则后果自负。

QAX A-TEAM

**BROP 简介：**  

**![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)**

Blind Return-oriented programming（BROP）是 2014 年 IEEE Symposium on Security and Privacy 中的一篇论文提出的漏洞利用思路，其可以在没有源码和二进制程序的情况下针对栈溢出漏洞进行自动化攻击。该思路假设服务器程序在崩溃后使用 fork 重启，在这种情况下，即使程序开启 ASLR、NX 以及 Canary 保护也可以利用成功。

ROP 是一种通过不断返回程序中原有指令来绕过 NX 保护机制从而控制程序执行流程的技术。BROP 最终也是利用了这项技术，但在整个利用过程中处于一种 Blind 的状态，通过不断向目标程序发送猜测数据，并根据程序的响应数据来确定可用的 gadgets 以及关键函数的导入表位置，最终根据确定的信息发起漏洞利用。下面将以 CVE-2013-2028（假设我们已经知道如何触发这个漏洞）为例，介绍 BROP 技术。

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)  

**暴破 Canary:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)

Canary 是进行漏洞利用的第一道坎，通常，Canary 在栈中的位置如下图所示，位于局部变量之后，RBP 和返回地址之前。这样如果发生栈溢出，就会先覆盖 Canary，然后才能覆盖到 RBP 和返回地址。程序在函数返回时先检查 Canary 的值是否改变可在一定程序上阻止通过覆盖返回地址劫持 RIP 的这种利用方式。                       

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwSuchaQPMFZNA4JCY65jFTlgaukbZB3G6Kr2xVwyx0qEnM5DDpPbpoA/640?wx_fmt=png)

由于 Nginx 守护进程在 Nginx 工作进程崩溃之后会重新启动一个新的工作进程，这个新进程的 Canary 并不会改变（只要守护进程还在），因而我们尝试暴破 Canary。暴破的步骤如下：

1、确定 Canary 偏移：确定刚好可以覆盖到 Canary 的发送数据长度

2、逐字节对 Canary 进行暴破：确定 Canary 的值

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwthxRbJxibg1N4AOZf6Ktt008s1QHwDZickUh9icaMLrRj7BUT8sWXqT6g/640?wx_fmt=png)

首先通过溢出一步一步确定 Canary 的位置，如果溢出的数据覆盖到 Canary，则该程序会在函数返回时 Canary 校验失败然后退出，也就不会有数据传回来，我们可以通过发送数据然后判断有没有成功 recv 数据来确定有没有覆盖到 Canary。首先可以在溢出数据的基础上不断增加 8 字节数据，知道没有接收到数据（目标程序崩溃），那 Canary 的起始地址一定在这 8 个字节对应的地址处。然后可以逐字节判断 Canary 的位置。

确定位置之后就可以逐字节暴破 Canary 了。逐字节暴破的原理也很简单，首先从最低位（一般为 0）开始暴破, 每个位置（字节）都有 0–255 这 256 种可能。直到这个位置的填充了正确的字节，才能成功 recv 到数据，否则程序就会 crash。通过这一特性，可以暴破出 Canary，暴破的结果如上图所示。同样，还可以利用这种思路暴破出 RBP 和返回地址。

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)

**寻找 Gadgets:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)  

Gadgets 包含以下几种：useful gadgets、stop gadgets 和 crash gadgets。其中 useful gadgets 是我们传统 rop 技术中通常会用到的 gadgets，如 “pop rdi；ret” 等等；stop gadgets 是使程序陷入循环的一种 gadgets，类似于 C 语言中 sleep 函数的功能；crashgadgets 是会使程序崩溃的 gadgets。

我们的目标是寻找 “pop rdi；ret” 之类的 useful gadgets，如果采取逐地址搜索的方式并没有很好的基准去判断 useful gadgets 是否执行，如下图所示，假设我们将返回地址填充为一个 “pop rdi；ret” 的 gadgets 的地址，即使漏洞函数返回时执行了这个 gadgets，最终还是会返回栈上的一个未知的地址（很有可能是不可读的地址），因而我们不能区分本次测试的 Gadgets 是 useful gadgets 还是 crash gadgets。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwj4A9ufAwcoyBsssZrBQo6asOgRgibXFWch4YzdKczrvEE1RJbczMnNQ/640?wx_fmt=png)

因而需要引入 stop gadgets，这样程序在执行完 pop rdi；ret 之后会返回 stop gadget，这样程序就会 hang 住，我们可以通过判断程序是否 hang 来确定当前是否命中了 useful gadgets。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwW31kpJ0VU4KFvtoGtLdjjz3WOicdO66IvlBfWMf8rKxPYukoF8UC1cQ/640?wx_fmt=png)

64 位程序中一定存在以下 gadgets：pop rbx；pop rbp；pop r12；pop r13；pop r14；pop r15；ret。其中 pop r14 的机器码是 41 5e, 5e 是 pop rsi 的机器码；pop r15 的机器码是 41 5f，5f 是 pop rdi 的机器码。因而只要找到了这个通用 gadgets，也就找到了 pop rdi 和 pop rsi。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwuOaibZ5ic8Wqn5ZJs6FulL36jZsPMmxdQzYL3icb2yzCwLrANVutCDw4Q/640?wx_fmt=png)

同理，如果要寻找这种 gadgets，需要在返回地址后填充 6 个 crash addr（如‘AAAAAAAA’），然后在后面铺上 stop gadgets，这样在一定程度上会过滤掉少于 6 个 pop 的 gadgets，后面会讲如何进行验证。

**探测返回地址深度：**

由于我们对程序的栈布局一无所知，我们还需要探测返回地址的偏移（相对于 Canary）。这样我们可以将待测的 useful gadgets 以及 crash addr 精准投放。探测返回地址的原理还是一样的，如果溢出数据没有覆盖到返回地址，程序就不会崩溃，就会接收到来自服务器的数据。如果刚好覆盖到返回地址，连接就会断开，这样我们就能得到返回地址的偏移，最终我们得到返回地址的偏移为 4。

**寻找 stop gadgets：**

当我们将返回地址填充为随机的地址时，通常会出现这种情况：该地址处的代码片段和当前寄存器中的环境不相符，可能会尝试访问一些无法访问的内存，这样程序就会 crash。但是还存在某种特殊的情况，程序在执行某块代码时陷入循环，不会 crash，这时程序就 hang 在那里，既不退出也不发送数据，攻击者能够一直保持连接状态。作者将这种类型的 gadget 称为 stop gadget，stop gadget 的这种特性是我们后续寻找 useful gadget 的基础。

于是，逐地址去寻找 stop gadget，然后真的找到一处，结果程序被 hang 住了，天真的我也被 hang 住了…… 这令人绝望的 RCX，使程序一直在 0x40260c 处自循环，也影响了程序的正常功能，不仅无法继续进行攻击，也无法重新发动攻击。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwLz3ysIj9T4hDnAOh1CBD3GvN0vvmaia6rdtnkicP9kUPkiaaCPFicMzlEQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwmM0yjllrsUzjRic3w2U6KKZBdI8VITW047Rwry8IN27SicxDvLmzcUibw/640?wx_fmt=png)

后来看了源码才发现，他们寻找的是一种组合起来的 hang，就是某个地址处的指令执行完之后返回到程序本身的某个返回地址会出现短暂的 hang，我们可以通过设置 timeout 来根据接收数据是否超时来判断是否命中了 stop gadgets。

在寻找 stop gadgets 的时候可以在程序. plt 段（.plt 段通常在. text 段地址前）里面找，这样在找 stop gadgets 的同时会找到非常有用的 PLT 项。Procedure Linkage Table（PLT，过程链接表）用于延迟绑定。

下面简单介绍一下原理，首先 PLT 项是 16 字节对齐的，每个函数的 PLT 项的第一条指令都是 jmp [func_got]，func_got 表示该函数的 GOT 表项，如果函数没有绑定（也就是没有执行过），其 func_got 中的内容为 PLT+0x6，即程序会跳转到该函数 PLT 表项的 0x6 偏移处继续执行，将函数的导入索引压栈，最终调用_dl_runtime_resolve_avx 函数将函数的实际地址写入 GOT 表并执行该函数。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwptxicgKUNAJRFdvtUo8bcmSKFvHl2gu1ZrGVibmwSSeILVbxEJWyqxag/640?wx_fmt=png)

这样当函数再次被调用时，会直接查询 GOT 表跳转到该函数处执行，如下图所示。因此我们可以得出结论，就是无论从某函数 PLT 表项的首地址处开始执行，或是从该函数函数 PLT 表项 0x6 偏移处开始执行，该函数都会成功被调用。这可以作为后续验证 PLT 项的一条判断规则。  

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwkiaJCcdRBFavhZvuCJwmAnWO2UIX9DZl9t9ncrOBRUcgycz0hHtYkwg/640?wx_fmt=png)

下面，我们开始寻找一个合适的 PLT 项作为 stopgadgets。我们在 Canary 后面开始填充待测数据，需要填充足够的深度使其可以紧挨返回地址，这样程序在执行这段组合代码时会挂住，这个深度也是需要逐步探测的。如果某个地址测试成功，我们可以将该地址偏移 6 的地址加入测试，如果依然测试成功，则证明找到了 PLT 项。  

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwHccF9RhB6t17m1ThLHhpQjKhKxIDWbWiaM8MsmrZxzosuATibcC1I9qg/640?wx_fmt=png)

我们从 0x400000 开始, 搜索范围为 0x5000（大概是 PLT 的范围），搜索间距为 0x300。如上图所示，第一次探测到的深度为 7。相对于返回地址的偏移是 3，最后由于最后 8 个字节 (偏移 7 处) 一定要填充为 stop gadgets 和后续的返回地址组合触发 hang，所以只剩下两个 Qword 的空间来存放 crash addr，寻找 6 个 pop 的 gadgets 至少需要 6*8 个字节的填充空间，因而我们需要逐步增加搜索深度。  

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pw0uusSnLPiaVSFQ8dXPBOQuralm8TrHibysEsLSdds6ISEZLrRebKvecA/640?wx_fmt=png)

最终找到了 0x402760 处的 plt，需要填充的深度为 43。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwKwm3wb4MyVS0SZqsibpACibXRSYzzEby9Dpo2kYI5nf7CWnfd3Enm9EA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwZylKS0Bm2KVhaNibq5K5vuqAdZlXuUsUyy9pKQgMEoDv6vDyXZ1XdhQ/640?wx_fmt=png)

**寻找 Useful Gadget：**  

找到可以用作参考的 stop gadget 后，我们就可以寻找 useful gadget 了，即寻找位于通用 gadgets 里的 pop rdi 和 pop rsi。首先再来欣赏一次通用 gadgets，该 gadgets 的起始地址为 0x430ddb，结束地址为 0x430de5，整个 gadgets 的字节数是 11。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwjgibicNnosVMa3hgGer1GTbYjo3EicqEUd91LDL3acBXFARvE0ADblhqA/640?wx_fmt=png)

为了节省时间，我们从 0x430000 处开始每隔 16 个字节进行一次测试。在实际测试中，可能不会精准命中 pop rbx 那个地址，因而我们先粗略获取包含 pop；ret 的 gadgets，这个 gadgets 可能是通用 gadgets 的一部分，然后再做进一步辨别。  

如下（左）图所示，这种布局可以捕获到 pop；ret、pop；pop；ret、pop；pop；pop；ret…… 这种 gadgets, 假设捕获到的 gadgets 属于通用 gadgets，那该通用 gadgets 的第一条指令 pop rbx 一定位于 [gadgets-9，gadgets] 这个区间。由于测试通过的 gadgets 至少为 pop rdi；ret（0x430de4），当然也有可能是 0x430de1，我们由最远的 0x430de4 去推算 pop rbx 的位置，就是 0x430de4-9（0x430ddb）。因而当某个 gadgets 通过第一次测试，我们会在 gadgets-9，gadgets]区间进行第二次测试，如下（中）图所示，如果该区间某个 addr 可通过测试二，说明从该地址处开始执行会进行 6 个 pop 操作，然后 ret，但并不能证明该地址处就是 pop rbx，还需要就进行终极验证。

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pw0cFt9zyyib9MbwbDHsTkW0g3LQJrnG95LAl2BPicmTzspoNcxLiafH3BQ/640?wx_fmt=jpeg)

终极验证的模式是在返回地址后面填充一个 pop 槽，如上（右）图所示，我们本地测试了通用 gadgets 中每个位置（还包括 ret 后一个地址）的程序运行结果，作为通用 gadgets 的特征，如下表所示。

| 

偏移

 | 

0

 | 

1

 | 

2

 | 

3

 | 

4

 | 

5

 | 

6

 | 

7

 | 

8

 | 

9

 | 

10

 | 

11

 |
| 

结果

 | 

Hang

 | 

Hang

 | 

Hang

 | 

False

 | 

Hang

 | 

Hang

 | 

Hang

 | 

Hang

 | 

Hang

 | 

Hang

 | 

False

 | 

False

 |

如果满足上述特征则证明找到了通用 gadgets，因而 pop rdi 在通用 gadgets 偏移 9 处，pop rsi（注意该 gadgets 有两个 pop）在通用 gadgets 偏移 7 处。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pw9WuYHebWKa8Ik890MYked9m9PQp244qakDGcANp6vBj1kYbla2UYjA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)

**寻找 plt 函数项：**

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)目前我们找到了控制前两个参数的 gadgets，某些后续要用到的函数需要三个参数，但 pop rdx；ret 不太好找，作者指了一条明路：借助 strcmp 函数设置 rdx，rdx 保存比较成功的字节数。那就成功将问题转化为寻找 strcmp。  

strcmp 函数用来比较两个字符串是否相等。它的两个参数均为字符串指针，如果某个指针指向的地址为无法访问的地址，程序就会 crash。超过 vsyscall page 时程序会正常返回也是判断 strcmp 函数的一个标准。因而作者采用了以下校验标准来判断该函数是否为 strcmp：

0、设置 good: 0x400000; bad1：3; bad2：5

1、strcmp(bad1, bad2) --> crash

2、strcmp(good, bad2) --> crash

3、strcmp(bad1, good) --> crash

4、strcmp(good, good)  --> hang

5、strcmp(vsyscall + 0x1000 - 1,good)  --> hang

**寻找 strcmp：**

前面介绍过 plt，它对导入函数进行延迟绑定，通过指定函数特定的索引可以将该函数的地址写入其对应的 GOT 表项，并运行该函数，我们关注的是它的第二个功能。假设导入函数中包含 strcmp，也就是程序在某一刻会调用 strcmp，那么在. plt 段一定存在 strcmp 的 PLT 项，我们要做的就是确定该函数的索引。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwxQXDYtSYNtkDq5adcVqMeZpTeJytibZSlsmJblYjz94y1TjCcDclxQg/640?wx_fmt=png)

我们将返回地址覆盖为 pop rdi 的地址，后面依次为第一个参数、pop rsi 的地址、第二个参数、填充 8 字节（由于 pop rsi 后要执行个 pop r15 才能 ret）、plt+0xb、待测索引、stopgadgets。其中 plt+0xb 后跟待测索引相当于程序 push 某个导入函数的索引然后要去执行 jmp 0x4024f0，这样就会执行相应的函数。

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pw0x2ZEEyefLMMgIhPhKfNtCZh6H2kk1DKrwBYqSic7k55RoTUZ438iafA/640?wx_fmt=jpeg)

顺序遍历索引直到找到可以满足 strcmp 函数五条准则的索引。如下图所示，当参数为 vsyscall+0x1000–1 和 good 时 0x1a 索引的函数触发了 crash，因而不是 strcmp；当参数为 3 和 5 时，索引为 0x1b 的函数成功执行了，因而也不是 strcmp，最终找到 strcmp 的索引为 0x1c。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwyTsNhPdGrD75B79kC3Tklia28ZnZ6Xoyiba1V8plAeU5ibeYY2h4ylufA/640?wx_fmt=png)

**寻找 write：**  

找到 strcmp 后就可以尽情设置 rdx 了，这时候就可以去寻找使用 3 个参数的 write 函数，以便后续 dump 程序内存。write 函数可以将 buf 中的 n 个字节写入指定文件描述符中，通常在 ELF 程序的开头会有以下魔值：

7f 45 4c 46 02 01 01 00 00 00 00 00 0000 00 00

如果从 0x400000 处读取一个字符串，那就是 "\x7fELF\x02\x01\x01"，我们可以将这个 buf 指针设置为 0x400000，然后通过调用 strcmp 将 n 设置为 7。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwo1XNpk4YeSIwYrNknQkAfDnYEPSlb3S3Ljba5cKw00UMpsHoxBjUibw/640?wx_fmt=png)

由于不能确定本次通信的文件描述符 fd，我们通过多次 write 的思路，将 0x400000 处的字符串写进不同的 fd，这样如果我们接收到了 "\x7fELF\x02\x01\x01" 字符串就证明找到了 write 函数索引。找到了 write 函数之后再遍历一遍 fd，就可以确定当前 fd。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwwXooVWZRq3ahUb6uAwr1TdQonvjQUsLmlkicf37XT4KviaMribgsQ8UUA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)

**Dump 程序内存：**

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)  

下面为从某个地址起 Dump140 个字节的代码，如果需要 dump 整个 ELF 文件，需要连续调用 dump_addr 函数，获得 dump 的内存并写入本地文件，然后将文件大小作为下次 addr 的偏移。

```
def dump_addr(self, addr):
data = ''
        for i in range(self.depth):
            data += p64(padval)
        for i in range(20):
            data = self.set_rdx(data)
            data += p64(self.rdi)
            data += p64(self.fd)
            data += p64(self.rsi)
            data += p64(addr + (i * 7))
            data += p64(0)
data += p64(self.plt + 0xb)
data += p64(self.write)
        data += p64(death)
        self.get_socket()
        self.send_body(data)
        x = ""
        while True:
            r = self.s.recv(4096)
            if len(r) == 0:
                break
            x += r
        self.close_socket()
        return x

```

下面要插播一段 ELF 文件结构，首先查看程序的段结构，其中. dynstr 段中存放进行动态链接所需的字符串，通常是表示与符号表各项关联的名称的字符串；.dynsym 段中为动态链接符号表；.rela.dyn 是对数据段应用的修正，它所修正的位置位于. got 和数据段；.rela.plt 是对函数引用的修正，它所修正的位置位于. got.plt：

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwIwJB9ggncjWIYjt7dZicSWd6nwUcNibsKUMlB9crX9P5Ax9c4DLBIPcg/640?wx_fmt=png)

首先通过偏移查看. dynstr 段中的内容，嗯…… 果然都是字符串，而且是连续的不需要对齐的字符串。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwf3tEY5TqxpAwviay3fSqWFYibX53Dl6PIqHStpCCQmSgU3qmSficE87PA/640?wx_fmt=png)

下图为. dynsym 段中符号表项的结构，每个符号表项的大小为 24 个字节。其中 str_name 为该项在字符串表中的索引，st_info 中包含了符号类型和绑定属性等信息，可以通过一定的处理得到这些信息，st_value 中可能指定该函数的虚拟地址（也有可能是 0）。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwicWuNdjr90OM2TKgVysqtJZ7pG6aic4wa9CXs29xP0ic0t52UMfUrQqPQ/640?wx_fmt=png)

我们先查看. dynsym 段中的内容，发现. dynsym 段的第一项均为 0，这或许可以作为后续寻找. dynsym 段的依据。我们可以查看第二项以后的 st_name, 分别为 0x23c、0x2b5、0x159、0xe4。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwvhJ8PqRqQgaRciaictNgASyrmDfRcgxRR2eRvEahicIYMShBicwLZzNujg/640?wx_fmt=png)

通过. dynsym 项中的偏移可以得到以下字符串。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwnQItYMsKOkw5k2uNLLco6DpJPJmP1H7oHl0nTtO6aYs56lGLbFqN0g/640?wx_fmt=png)

下面是从 info 中获取符号类型和绑定属性信息的方法，通过将 info 和 0xf 逐位相与，也就是取其低 4 比特的值。比如说. dynsym 中的第一项，其 info 为 0x12，0x12&0xf 为 2，所以其符号类型就是 2。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwicVE3kCLicXaiaicFUavmN7blg3APJPBssuUokjtupWibFV4JkiaNljPZ99w/640?wx_fmt=png)

下面看一下符号类型：STT_NOTYPE(0) 表示未指定符号类型；STT_OBJECT(1) 表示此符号与变量、数组等数据关联；STT_FUNC(2) 表示此符号与函数或其他可执行代码关联，这与 initgroups 项的符号类型为 2 相符。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwROJr19HkIqbXa92OfibvjdV6Wic3c7iaBLTbVcjAiaBxNeIpbejM4Rvt3Q/640?wx_fmt=png)

接下来就是. rela.plt 段了，首先来看一下 ELF64_Rela 结构体，每个结构体有三个成员：offset 表示该函数的 got 表地址；info 里面高 32 位为符号表索引、低 32 位为重定位类型；addend 指定常量加数，用于计算将存储在可重定位字段中的值。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwXX7jmCkpcJf14ofg6JCibPxzRPhuBhZhknvjhicgHoyld0fSlLpVJbxw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwkQ5ptN98CfI4Sq8ZI3xYQ5AAp4jLU3ia7BNhZxyMzpHnw1CXPAAVpog/640?wx_fmt=png)

使用 readelf 解析. rela.plt 段中的内容，可以发现 offset 中的值确实位于. got 段，符号表索引由 1 开始，重定位类型均为 R_X86_64_JUMP_SLOT(7)。  

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwqjhypvHam0tubptmaLWicTzLv7d2tYjUlMBoicSdwoxUWlgUTJSk5D7g/640?wx_fmt=png)

我们来看一下前两个函数的 got 表项，第一个位于 0x695018，它里面存储的地址已经变成 initgroups 函数的地址了，说明这个函数已经被调用过一次。第二项是 chmod 函数，起 got 表项中的地址依然是它的 plt+6 的地址，注意 push 0x1 中的索引 1，和上图中的 2（符号表索引）并不一样。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwxiaZHJ7uyyrib89IZlNutME0yxDCU111iaKlTEEQcuedibleh73wgwlobQ/640?wx_fmt=png)

再来看一眼 PLT 表，确定 PLT 索引是从 0 开始的，其实. rela.plt 段中的符号表索引不是一直递增的，所以如果想获取 PLT 索引表，需要先定位到. dynsym 段，然后找到. dynsym 段，根据每一项的字符串表偏移找到该符号的名称，得到符号表。然后再找到. rela.plt 段，通过每一项中的符号表索引得到其对应的符号名，得到 PLT 索引表。有了这个表就可以轻松找到某导入函数的 PLT 索引，然后使用之前介绍的方法去调用某些函数了。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pw2Kty7lMRJ2rAmkQrOREJLzFoicvVdib8ia8uibH3mPITicSK3ymLWFXjkLQ/640?wx_fmt=png)

**寻找 dynstr：**

通过 strings 命令获取程序中的字符串，发现. dynsym 段前面只有一个字符串。因而如果我们可以找到某个地址，从该地址处开始出现连续字符串，那么我们就找到了. dynstr 段。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwCvuLNvaB0GbykagS4RmwBF60aIcNnqU3YNQvAtPuuGqeKeKG9iajLGw/640?wx_fmt=png)

可以采用简单的有限状态机来定位到. dynstr 段（或正则匹配）：

首先初始化为状态 0，如果没有找到可见字符，就一直保持状态 0，如果发现可见字符就将 len 置 0，进入状态 1；

状态 1：如果发现可见字符就将 len 加 1，如果发现不可见字符，首先判断是否为‘\x00’，如果为‘\x00’，判断 len 是否大于等于 3，如果判断成功，则将 len 置 0，进入状态 2，否则回到状态 0；

状态 2：如果发现可见字符就将 len 加 1，如果 len 的值大于等于 3 就验证成功，如果如果发现不可见字符，则回到状态 0。

由于. dynsym 段第一个字节为‘\x00’，还需要判断测试通过需要将得到的地址减 1。

**寻找 dynsym：**

从布局来看. dynsym 段正好位于. dynstr 段前面，通过分析，我们可以发现. dynsym 段的最后一项为 free，后面没有填充，紧挨着的就是. dynstr 段。因而，我们可以从. dynstr 段头开始向前搜索，直到发现 24 个连续的‘\x00’，就证明我们找到了. dynsym 段。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwUoibJkaeh23JhqedxhNncNtkic4gH6K2R4DRAe5fMkqZTQwspxPmuMlg/640?wx_fmt=png)

找到. dynsym 之后，我们解析符号表，获取偏移、类型和 val 值，然后通过偏移在. dynstr 中获取相应的符号名，并时刻关注符号类型，如果发现符号类型为 1，并且 val 为有效值，则找到一个可写的地址 val，这个后面会用到。然后从 1 开始编号，按顺序写入符号名，得到一个符号表。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwClbyLIVQYHMPpxxd7wxicKpbLVMSmktZt4VQeTYGBoegeDcSicpTSXEw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwBUcVURnwpntxvQWWJKAic6vZoEUujK1spkccFwgwha082TONGibtBewQ/640?wx_fmt=png)

**寻找 rela.plt：**

由于. rela.plt 段在. dynsym 和. dynstr 后面，而且 rela 结构也是 24 字节对齐，所以可以顺序向下读取，直到找到一个地址 x，从 x 处取出 4 字节值（type）为 7，从 x+8 处取出 8 字节值为 0（addend），并且当 x = x + 24i 时，这种关系均成立，那么…….rela.plt 的起始地址为 x-8。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwHlRpSWXgOrDXlyIzEHWGAHibV3sDTiaQMn8cjscgyAoX8VHFK86Et8QA/640?wx_fmt=png)

然后就可以解析重定位表了，获取相应的重定位类型和符号表索引，通过符号表索引获取该项的符号名，并从 0 开始编号。重定位类型可作为一个结束判断条件，当我们获取的重定位类型不再为 7，也就证明重定位表解析完成了。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pw0TL3XZwtnd6mo8KnhwpHGruRPf6jYbDgj5t0ldsWiaxfH6XGq449dvg/640?wx_fmt=png)

然后可以在 PLT 索引表中寻找感兴趣的函数索引，可以在后续利用中使用。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwYLbGCwD7HrgdE7szmssYJrYvWs2aCaLBLHAfpomtsVFVibn7JqiatFcw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)

**漏洞利用之 getshell：**

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)  

由于没有在程序中找 “/bin/sh” 字符串，所以要找一块可写的内存然后使用 read 函数将其写入，前面在解析符号表的时候已经找到可写的地址。“/bin/sh” 字符串的大小为 8 个字节，后面的‘\x00’不能忽略，而我们之前设置的 rdx 为 7，所以需要找到字符串长度超过 8 的某个地址，然后通过 strcmp 设置长度。这种字符串很多，通用 gadgets 里面就有长度大于 8 的字符串。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwn8Zb2VseicByrnMp4kyea1NWEEucblbXflEpgXcdYticqEA3qDzqEVQQ/640?wx_fmt=png)

我们可通过 pop rdi 的地址再次定位到该通用 gadgets 起始地址，然后逐地址设置 rdx，然后将该地址处的字符串通过 write 函数传送过来，通过判断接受到的字符串长度是否大于 8 来确定是否找到这个字符串的地址。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwvFPEjN3uCmXplamzCXDMq4OHYYIgeImPh3FgbcicGW5bb5KWNUnUibvA/640?wx_fmt=png)

然后还是需要使用 dup2 函数将 stdin、stdout、stderr 复制到当前套接字，然后使用 write 函数向回发送一段数据（可以是可写地址的数据）作为 fd 重定向完成的信号，这样我们就可以将 “/bin/sh” 发送过去，同时在 write 函数后面我们需要调用 usleep 函数等待 2 秒，不然 “/bin/sh” 还没发过去 read 执行了，那 “/bin/sh” 就会写失败。然后在后面接上 read，从套接字读取 “/bin/sh” 放到可写地址。然后接上 execve 函数，执行 execve（“/bin/sh”，0，0）返回一个交互式的 shell。

![](https://mmbiz.qpic.cn/mmbiz_png/EkibxOB3fs4ibzw6D3ec4a19A48fbTC4pwdGV1xBPlq9m2TntNjcxfhAxKta1wXJOOwZvWwkj98DfG3VDCtXYEkQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/EkibxOB3fs4icwQQAZE6MBepadE7zAutkvAdjZD0jIFrBDRm9sBJ3KIqPyXib8bxWfuyXiclBGFIjeXC8ZkMZeicO8A/640?wx_fmt=jpeg)