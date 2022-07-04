> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273533.htm)

> [原创]Code Virtualizer 逆向工程浅析

论坛前最大字数限制，发不出去，只好将不紧要的部分略了，就这样吧。有意细究的，去拖未删减的 TXT 原文看。

《Code Virtualizer 逆向工程浅析》

```
创建: 2022-06-30 16:56更新: 2022-07-03 11:07http://scz.617.cn:8/misc/202206301656.txt☆ 背景介绍☆ Code Virtualizer 2.2.2.0    1) CV SDK    2) cvtest.c    3) 编译    4) CV虚拟化☆ TTD☆ CV虚拟机框架概览    1) 从cv_entry[0]到cv_entry[4]    2) 定位cv_entry[1]    3) 在cv_entry[4]处获取关键信息    4) 定位func_array[]    5) 确定func_array[]元素个数    6) 推断VM_CONTEXT结构    7) 从VM_DATA复制数据到VM_CONTEXT    8) 定位cv_entry[5]☆ CV虚拟机逆向工程经验    1) VM_CONTEXT.dispatch_data    2) 跟踪func_array[i]    3) VM_CONTEXT.efl    4) VM_CONTEXT.rsp    5) deref操作    6) 库函数调用    7) 分析func_array[i]        7.1) 复杂func_array[i]的简单示例    8) 静态定位func_array[i]出口        8.1) 全连接图的非递归广度优先遍历    9) 寻找流程分叉点   10) 反向执行寻找pushfq   11) cv_entry[3]的函数化☆ 后记
```

☆ 背景介绍

"Code Virtualizer" 的资料不多，可能与它不如 VMP 被广泛使用有关，OSForensics 9 用了 CV。若非现实世界有实用软件用 CV 保护，鬼才有兴趣对之进行逆向工程。之前没有接触过 CV，用 TTD 调试 OSF 时被绕得七荤八素，后来无意中确认 OSF 用 CV 保护。上网搜了些 CV 资料，都比较老，适用于 1.3.8 或更早期版本，与 OSF 所用 CV 版本差别较大。还有一点，老资料出现在 32 位时代，现在是 64 位时代。

CV 将 CFG 扁平化，实际上没有调用栈回溯一说。CV 处处是间接转移，主要是 jmp 寄存器这种形式，其次是将目标地址压入栈中，靠 ret 转移，这样一来，IDA 中几乎没有显式交叉引用。敏感字符串是可以混淆存放的，这条路也断了。

本文分享一些 CV 逆向工程经验，基于网上能公开下到的 CV 2.x。OSF 所用 CV 是何版本，我不知道，但实测发现本文的经验大多也适用于 OSF 逆向工程。我只关心 C 语言，其他语言不在本文范畴。

☆ Code Virtualizer 2.2.2.0

1) CV SDK

公开能下到的只有 CV 2.x，以此为研究对象。压缩包展开后主要关注这些文件和目录

(略)

2) cvtest.c

没有实际意义，只是示例，要点如下

```
#include "VirtualizerSDK.h"/* * 链接时需要，下面那对宏会转换成对该库中函数的调用，用以占坑 */#pragma comment( lib, "VirtualizerSDK64.lib" )/* * 将需要保护的代码片段置于这对宏中间 */VIRTUALIZER_EAGLE_BLACK_START/* * 被保护代码片段位于两个宏中间 */...VIRTUALIZER_EAGLE_BLACK_END
```

"EAGLE_BLACK" 是 CV 虚拟机的一种，看上去保护强度最高。SDK 自带的 vc_example 用的是 "TIGER_WHITE"，保护强度很低。

3) 编译

(略)

4) CV 虚拟化

执行 Virtualizer.exe

(略)

缺省 LastSectionName 可能是其他值，比如 ".vlizer"。cvtest_p.exe 有 2 个 ".scz"。

同一个 cvtest.exe，每次 CV 虚拟化生成的 cvtest_p.exe 都不一样。

前面简介了 CV SDK 的使用，最好是自己整一个 cvtest.c，生成 cvtest_p.exe，对后者进行逆向工程，积累经验后再去对付现实世界的例子，比如 OSF。

☆ TTD

CV 虚拟化本身不会增加反调试检查，调试 cvtest_p.exe 时不需要反 "反调试"。OSF 有反调试，但 OSF 没有考虑到 TTD 技术的出现，其反调试措施没有针对 TTD 录制。

即便不考虑反 "反调试"，对 CV 保护过的代码进行逆向工程时，条件允许的情况下，强烈建议 TTD 录制。若对 CV 有过经验积累，再动用 TTD，能极大地抵消 CV 保护。

☆ CV 虚拟机框架概览

1) 从 cv_entry[0] 到 cv_entry[4]

VIRTUALIZER_EAGLE_BLACK_START 那一对宏在编译后化身为两个 call

```
/* * VIRTUALIZER_EAGLE_BLACK_START */000000014000108F FF 15 03 10 00 00           call    cs:VirtualizerSDK64_151/* * 被保护代码片段位于两个call中间 */.../* * VIRTUALIZER_EAGLE_BLACK_END */0000000140001152 FF 15 48 0F 00 00           call    cs:VirtualizerSDK64_551
```

这是 cvtest.exe 中的效果，cvtest.exe 只是从 C 编译成 PE，尚未进行 CV 虚拟化处理。151、551 这种数字无关紧要，要点是它们成对出现。

Virtualizer.exe 靠这两个 call 识别出待保护代码片段，对之 CV 虚拟化，将 11KB 的 cvtest.exe 膨胀成 1649KB 的 cvtest_p.exe，这是加了多少垃圾代码？

CV 虚拟化时将 "call VirtualizerSDK64_151" 就地转成 jmp，这是 cvtest_p.exe 中的效果

```
/* * VIRTUALIZER_EAGLE_BLACK_START * * 为叙述方便，此处定义成cv_entry[0] */000000014000108F E9 32 BB 19 00              jmp     cv_entry_1_14019CBC6/* * 中间的字节流是啥我也不知道，反正不是原来的代码 */.../* * VIRTUALIZER_EAGLE_BLACK_END * * 将"call VirtualizerSDK64_551"就地转成类似nop的填充指令，模式不固定 * * 为叙述方便，此处定义成cv_exit[0] */0000000140001152 88 C9                       mov     cl, cl0000000140001154 88 C9                       mov     cl, cl0000000140001156 88 C9                       mov     cl, cl
```

cv_entry[0] 还在. text 中，但 jmp 的目标地址 cv_entry[1] 已离开. text，进入. scz。

```
000000014019CBC6                         cv_entry_1_14019CBC6/* * 为叙述方便，此处定义成cv_entry[1] */000000014019CBC6 9C                          pushfq.../* * 从pushfq到jmp，无任何分支转移指令，二者就是块首、块尾 * * 为叙述方便，此处定义成cv_entry[2] */000000014019CD1C E9 18 DD FE FF              jmp     cv_entry_3_14018AA39
```

cv_entry[1] 的特点是 pushfq，cv_entry[1]、cv_entry[2] 之间无任何分支转移指令，二者就是块首、块尾，在 IDA 中用图形模式查看，非常明显，这是第二个特点。

```
/* * 位于第一个".scz"中，Ctrl-S确认 */000000014018AA39                         cv_entry_3_14018AA39/* * 用到自定位技巧，shellcode常用套路 * * 为叙述方便，此处定义成cv_entry[3] */000000014018AA39 E8 00 00 00 00              call    $+5.../* * 为叙述方便，此处定义成cv_entry[4] */000000014018BE28 FF 20                       jmp     qword ptr [rax]
```

cv_entry[3] 的特点是 "call $+5"，一种自定位技巧，shellcode 常用套路。在 IDA 中 Alt-B 搜索字节流 "E8 00 00 00 00"，找出所有 "call $+5"，基本上都是 cv_entry[3]。

cv_entry[4] 的特点是 "jmp [rax]"。

即使在 C 代码中只使用了一对 VIRTUALIZER_START/VIRTUALIZER_END，cvtest_p.exe 仍有可能出现多个 cv_entry[3]，为什么？因为只要进入 CV 虚拟机一次，就会有一个 cv_entry[3] 等着经过，从 CV 虚拟机中调用外部库函数时，会临时离开 CV 虚拟机，执行完外部库函数，重新回到 CV 虚拟机。在这些进出 CV 虚拟机过程中，自然出现多个 cv_entry[3]，有些进出流程可能共用一个 cv_entry[3]，有些可能用自己的 cv_entry[3]。

cv_entry_3_14018AA39 可以 p 操作成函数，图形化查看时非常复杂，但把握住前述入口与出口特点，搞几次后就能轻松定位。

IDA 可能缺省未将 cv_entry[1] 与 cv_entry[3] 识别成函数，我的事后复盘经验是，一定将它们 p 成函数，以降低静态分析难度，IDA 的图形模式只能看函数。

CV 虚拟机官方没有 cv_entry[0]、cv_entry[4] 这些概念，这是为了叙述方便自己给的定义。回顾一下流程框架

```
/* * cv_entry[0] */jmp     cv_entry[1]-------------------/* * cv_entry[1] */pushfq.../* * cv_entry[2] */jmp     cv_entry[3]-------------------/* * cv_entry[3] */call    $+5.../* * cv_entry[4] */jmp     [rax]
```

逻辑上 cv_entry[0] 在 CV 虚拟机外，一般在. text 中，这个不绝对。之后 cv_entry[1] 至 cv_entry[4] 全部在 CV 虚拟机中，一般在. vlizer 中。

2) 定位 cv_entry[1]

已知从 cv_entry[0] 转向 cv_entry[1]，会从. text 转向. scz(本例中的名字)，可以用 x64dbg 调试，对. scz 设内存访问断点，以此快速定位 cv_entry[1]。这段话假设目标程序比较复杂，现在还在. text 中，静态分析一时半会儿找不到 cv_entry[0]。若肉眼就能发现 cv_entry[0]，则无需前述技巧。

定位 cv_entry[1] 之后，静态分析就能定位定位 cv_entry[2] 到 cv_entry[4]。

3) 在 cv_entry[4] 处获取关键信息

假设调试器停在 cv_entry[4]

```
rax=000000014018120e rbx=0000000000000360 rcx=0000000140000000
rdx=0000000000180eae rsi=5555555555555555 rdi=6666666666666666
rip=000000014018be28 rsp=000000000014fe08 rbp=00000001400ab9d6
 r8=8888888888888888  r9=9999999999999999 r10=aaaaaaaaaaaaaaaa
r11=bbbbbbbbbbbbbbbb r12=cccccccccccccccc r13=dddddddddddddddd
r14=eeeeeeeeeeeeeeee r15=ffffffffffffffff
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
```

在 cv_entry[4] 处有

```
rsp 0x14fe08    VM_DATArbp 0x1400ab9d6 VM_CONTEXTrax 0x14018120e &func_array[0x6c]rbx 0x360       0x360/8=0x6crcx 0x140000000 ImageBaserdx 0x180eae    VM_CONTEXT.func_array 0x140180eae
```

在 cv_entry[4] 处可以找到 VM_CONTEXT，这是 CV 虚拟机的核心组件之一，后面再说。

(略)

从 cv_entry[0] 到 cv_entry[4] 真正干的大事就是将 cv_entry[0] 处各寄存器压栈，一堆眼花缭乱的操作都是为了掩盖这个事实。最初我还老老实实在 TTD 调试中一步步跟，后来意识到它的意图后，采用污点追踪的思想快速定位 cv_entry[4] 处栈中诸数据。

前文所用术语都是自己瞎写的，结合上下文对得上就成。

4) 定位 func_array[]

func_array[] 就是老资料里说的 handler[]，CV 虚拟化将每一条位于保护区的汇编指令转换成许多个 func_array[i] 组合。

在 cv_entry[4] 处有多种办法定位 func_array[]，比如

```
> ? @rcx+@rdx> ? @rax-qwo(@rsp+0x88)*8Evaluate expression: 5370285742 = 00000001`40180eae
```

0x140180eae 即 func_array[] 起始地址。

有个取巧的办法定位 func_array[] 起始地址。假设已知 VM_CONTEXT 在 0x1400ab9d6，本例中该结构占 0x174 字节，但该结构大小并不固定，有可能是其他大小。在 IDA 中查看 0x1400ab9d6 处 hexdump，大片的 0，只有一处非零，就是 VM_CONTEXT.func_array 字段所在，静态查看时该值是重定位前的偏移值，加上基址才是内存地址。

IDA 中看 func_array[i]，是重定位之前的偏移值，加上 ImageBase 才是函数地址。应在 IDA 中静态 Patch，人工完成重定位，使得 IDA 分析出更多代码。func_array[] 比较大，很可能没有以 qword 形式展现，一个一个手工加基址 Patch 不现实，写 IDAPython 脚本完成。

5) 确定 func_array[] 元素个数

没有简单办法确定 func_array[] 元素个数。在 IDA 中肉眼识别、逐步逼近当然可以，但不够放心，怕不精确。

有个辅助办法，图形化查看 cv_entry[4]，往低址方向找如下 cmp、test 指令，还是比较容易定位的。

```
/* * rax=0x140180eae VM_CONTEXT.func_array+ImageBase * rdx=0x180eae VM_CONTEXT.func_array */000000014018B9D4 48 39 C2                    cmp     rdx, rax000000014018B9D7 0F 84 72 03 00 00           jz      loc_14018BD4F
```

```
/* * rbx=0x97 func_array[]元素个数 */000000014018BABD 48 85 DB                    test    rbx, rbx000000014018BAC0 0F 84 7D 01 00 00           jz      loc_14018BC43
```

找到 0x14018B9D4、0x14018BABD 这两个地址后，在 TTD 调试中对之设断点，从 cv_entry[3] 处正向执行，断点命中时查看寄存器，注释中写了。不一定 TTD 调试，普通调试就可以，但我一上来就 TTD 录制了，后面的分析都是在反复鞭尸，更方便。

精确知道 func_array[] 元素个数后，写 IDAPython 脚本对之批量 qword 化、加基址。这还不够，应该对每个 func_array[i] 加 "repeatable FUNCTION comment"，比如这种效果

```
0000000140180EAE 4A BB 0A 40 01 00 00 00     dq offset sub_1400ABB4A             ; func_array[0x0]0000000140180EB6 8E BC 0A 40 01 00 00 00     dq offset sub_1400ABC8E             ; func_array[0x1]0000000140180EBE 54 BE 0A 40 01 00 00 00     dq offset sub_1400ABE54             ; func_array[0x2]0000000140180EC6 0E C7 0A 40 01 00 00 00     dq offset sub_1400AC70E             ; func_array[0x3]
```

```
00000001400ABB4A                         ; func_array[0x0]00000001400ABB4A00000001400ABB4A                         sub_1400ABB4A proc00000001400ABB4A E9 3F E9 01 00              jmp     sub_1400CA48E               ; func_array[0x0]00000001400ABB4A                         sub_1400ABB4A endp
```

```
00000001400CA48E                         ; func_array[0x0]00000001400CA48E00000001400CA48E                         sub_1400CA48E proc00000001400CA48E 9C                          pushfq
```

CV 虚拟机很复杂，给每个 func_array[i] 自动加注释，有助于聚焦。

EAGLE_BLACK 虚拟机比 TIGER_WHITE 虚拟机复杂得多，func_array[i] 只是个幌子。0x1400ABB4A 处 jmp 到 0x1400CA48E，后者也不是真正干活的 handler，其实是另一个 cv_entry[1]，后面有另一个 cv_entry[2] 到 cv_entry[4]，最终会去找另一个 func_array_2[j]。不建议初次接触 CV 的人一上来就逆 EAGLE_BLACK 虚拟机，可以拿 TIGER_WHITE 虚拟机练手。当然，前面我都给出提纲挈领的大框架了，再看 EAGLE_BLACK 虚拟机，也不是那么难。

6) 推断 VM_CONTEXT 结构

流程到达 cv_entry[4] 时，rbp 指向 VM_CONTEXT 结构

(略)

本例中该结构占 0x174 字节，但该结构大小并不固定，主要是大片的 0。流程到达 cv_entry[4] 时，VM_CONTEXT 结构部分成员已初始化，包括

```
#pragma pack(1)struct VM_CONTEXT{    /*     * 进入CV虚拟机时设1，离开CV虚拟机时设0，逻辑上相当于(实际有出入)     *     * pusha     * mov busy, 1     * ...     * mov busy, 0     * popa     */    unsigned int        busy;               // +0x0 0x1400ab9d6    ...    /*     * 保存cv_entry[0]时的rbp     */    unsigned long long  orig_rbp;           // +0x96 0x1400aba6c    ...    /*     * 0x140000000     */    unsigned long long  ImageBase;          // +0xd5 0x1400abaab    ...    /*     * 0x19a914+0x140000000=0x14019a914     */    unsigned char *     dispatch_data;      // +0x10a 0x1400abae0    ...    /*     * 0x180eae+0x140000000=0x140180eae (func_array)     */    unsigned long long  func_array;         // +0x12a 0x1400abb00    ...                                            // +0x174 0x1400abb4a};#pragma pack()
```

每个 CV 虚拟机要单独分析 VM_CONTEXT 结构各成员位置，总是在变，就是为了对抗逆向工程，上面只是一种示例。若非高价值目标，不建议与 CV/VMP 这类虚拟机搏斗，浪费生命。

可能过去 VM_CONTEXT 结构总是位于. vlizer 起始位置，现在没这经验规律了，不能假设仍然如此，事实上 OSF 就不服从该规律。此外，VM_CONTEXT 结构之后不能假设紧跟 func_array[]，应该用 VM_CONTEXT.func_array 定位。流程到达 cv_entry[4] 时，VM_CONTEXT.func_array 已是重定位后的地址。

7) 从 VM_DATA 复制数据到 VM_CONTEXT

VM_DATA 是我给压在栈上的各寄存器布局瞎起的结构名字，便于叙述，不必当真。

在 cv_entry[4] 处查看 VM_DATA

```
00000000`0014fe08  88888888`88888888    r8  <=rsp00000000`0014fe10  99999999`99999999    r900000000`0014fe18  aaaaaaaa`aaaaaaaa    r1000000000`0014fe20  bbbbbbbb`bbbbbbbb    r1100000000`0014fe28  cccccccc`cccccccc    r1200000000`0014fe30  dddddddd`dddddddd    r1300000000`0014fe38  eeeeeeee`eeeeeeee    r1400000000`0014fe40  ffffffff`ffffffff    r1500000000`0014fe48  66666666`66666666    rdi00000000`0014fe50  55555555`55555555    rsi00000000`0014fe58  77777777`77777777    rbp00000000`0014fe60  22222222`22222222    rbx00000000`0014fe68  22222222`22222222    rbx_other00000000`0014fe70  44444444`44444444    rdx00000000`0014fe78  33333333`33333333    rcx00000000`0014fe80  11111111`11111111    rax00000000`0014fe88  00000000`00000202    efl00000000`0014fe90  00000000`0000006c    func_array_index00000000`0014fe98  00000000`0019a914    dispatch_data
```

直接对栈中各寄存器值设数据断点

```
ba r1 /1 @rsp "dqs 0x14fe00 l 0n21;db 0x1400ab9d6 l 0x174"
```

每次命中时重新设置上述数据断点，依次命中

```
1400167f5   pop_to_context_n_1400165FC  func_array_2[0x5a]14007d427   pop_to_context_n_14007D27C  func_array_2[0x27e]140079527   pop_to_context_n_14007940A  func_array_2[0x266]...14001e89d   pop_to_context_n_14001E7B3  func_array_2[0x8a]1400141d7   pop_to_context_n_14001413C  func_array_2[0x4f]
```

用这种办法可以知道 VM_CONTEXT.r8 的偏移，还可以找到 pop_to_context__，这种 handler 对应 "pop [addr]"。EAGLE_BLACK 有多种 pop_to_context__，TIGER_WHITE 只有一种，难度相差极大。

8) 定位 cv_entry[5]

从栈中弹栈到 VM_CONTEXT.efl，是最后一个弹栈动作，至此所有栈中寄存器均被弹入 VM_CONTEXT 结构相应成员。假设流程到达 cv_entry[4]，不必费劲地对栈中各寄存器设数据断点，只需要对栈中的 efl 设数据断点即可。

```
ba r1 /1 @rsp+0x80 "dqs 0x14fe00 l 0n21;db 0x1400ab9d6 l 0x174"
```

断点命中时，查看内存中的 VM_CONTEXT 结构

(略)

由于我采用了污点追踪的思想，肉眼就能识别各寄存器在 VM_CONTEXT 结构中的偏移，据此可进一步完善 VM_CONTEXT 结构定义。

cv_entry[5] 是个虚概念，只是为了叙述方便。流程到达 cv_entry[5] 时，VM_CONTEXT 中各寄存器已填写完毕。若在 TTD 调试中，记下断点命中时所在 position 值，方便回滚。

cv_entry[5] 位于 func_array_2[j] 中，j 不固定。func_array_2[j] 没有显著特征，无法通过静态分析定位 cv_entry[5]，只能动态调试定位，这与 cv_entry[4] 不同。

cv_entry[5] 之后的流程才真正对应 "被保护代码片段"，之前的流程都是 CV 虚拟机初始化。若不知道这点，一上来就楞调试，早早陷入 CV 虚拟机的圈套，很容易失焦。

cv_entry[5] 之后也不见得马上对应 "被保护代码片段"，某些 func_array_2[i] 实际对应 nop 操作，看上去又很复杂，nop 操作想插多少有多少，想插哪里插哪里。分析 CV 虚拟机时，还得动其他脑子。

☆ CV 虚拟机逆向工程经验

为叙述方便，本节不区分 func_array[i]、func_array_2[j] 等，概念上它们地位相当。

1) VM_CONTEXT.dispatch_data

VM_CONTEXT.dispatch_data 是个指针，指向 IDA 中静态可见的数据区域。每个 func_array[i] 都会从 VM_CONTEXT 中取 dispatch_data 指针，再从 dispatch_data[] 取  
数据。

dispatch_data[] 是一段字节流，没有固定的结构，没有固定的大小。使用它时，从哪个位置取几个字节上来，完全由当前用它的 func_array[i] 决定，几乎每个 func_array[i] 使用 dispatch_data[] 的方式都不一样，这是对抗逆向工程的手段之一。

以 mov 操作为例，可能 wo(dispatch_data+5) 是一个 16 位偏移，加上 VM_CONTEXT 基址后定位到 VM_CONTEXT.rax 成员；可能 dwo(dispatch_data+0x13) 是虚拟化之前的 mov 指令中的立即数。理论上，找到合适的 dispatch_data[i] 可以暴破 CV 虚拟化过的代码。

每个 func_array[i] 用完当前 dispatch_data[] 后，会更新 VM_CONTEXT.dispatch_data，确切地说，是递增，使之对应 func_array[i+1]。

2) 跟踪 func_array[i]

前面讲过定位 func_array[] 起始地址，现在想知道依次执行了哪些 func_array[i]。

已知在各个 func_array[i] 之间转移时，VM_CONTEXT.dispatch_data 会递增，对之设数据断点，即可跟踪 func_array[i]。前述数据断点命中时，有些 CV 虚拟机可能位于 func_array[i] 的最后一条指令，一般是相对转移指令，这是理想情况。OSF 所用 CV 虚拟机更变态，更新 VM_CONTEXT.dispatch_data 的代码在 func_array[i] 中部，而不是尾部。

3) VM_CONTEXT.efl

虚拟化前 add/sub/xor/cmp/test 等指令在虚拟化后都有各自对应的 func_array[i]。简单的 CV 虚拟机可能 add 指令对应唯一的 func_array[i]，早期 CV 可能就这样，现在不是了，多条 add 指令可能对应不同的 func_array[i]，防止在逆向工程中一次标定多次使用。好不容易标定某 func_array[i] 对应 add 操作，结果下一个 add 操作不过这个 func_array[i]，抓狂。

前述这些指令有个共同点，实际执行时会修改 efl。CV 虚拟化后，它们对应的 func_array[i] 会修改 VM_CONTEXT.efl，可能是这样的片段

```
/* * r13d=op1 * edi=op2 */000000014007DEAD 41 85 FD                    test    r13d, edi000000014007DEB0 9C                          pushfq...000000014007DF4B 5B                          pop     rbx.../* * rbx=efl * r15=VM_CONTEXT.efl */000000014007E003 49 89 1F                    mov     [r15], rbx
```

对 VM_CONTEXT.efl 设数据断点，能加快 func_array[i] 的标定。

上面是理想情况。EAGLE_BLACK 虚拟机比较变态，test 指令修改了 efl，但当前 func_array[i] 不会更新 VM_CONTEXT.efl，它将 efl 存到 tmp 中；然后其他 func_array[i] 不断搬运 tmp，push/pop/mov 操作对应的 func_array[i] 挨个来，无效搬运，很久之后才将源自 tmp 的数据搬运进 VM_CONTEXT.efl。我碰上过 test 操作与最终更新 VM_CONTEXT.efl 的操作相差 619 个 func_array[i]，中间的全是垃圾操作，目的是让你搞不清发生了什么。OSF 所用 CV 虚拟机更新 VM_CONTEXT.efl 时没这么变态，但有其他变态之处。

4) VM_CONTEXT.rsp

push/pop 操作对应的 func_array[i] 可能同步更新 VM_CONTEXT.rsp，对之设数据断点，能加快标定。

5) deref 操作

虚拟化前的指针操作被虚拟化成某种 func_array[i]，且不唯一。

对于汇编指令 "mov rcx,[r15]"，逻辑上相当于 "rcx=*(r15)"，这就是一种 deref 操作，对应某种 func_array[i]。

对于 "mov edx,[rsp+0x48]"，这种会拆分成 "tmp=rsp+0x48"、"edx=*(tmp)"，至少对应两个不同的 func_array[i]。

6) 库函数调用

部分情况通过 ret 进行库函数调用，并不都是。无论如何，从 CV 虚拟机内部调用外部库函数，都涉及离开、重入 CV 虚拟机，该过程必然更新 VM_CONTEXT.busy 字段。

流程到达 cv_entry[4] 时，rbp 指向 VM_CONTEXT 结构，"db @rbp l 0x174"，有个明显的位置是 1，那儿就是 VM_CONTEXT.busy 字段。

IDA 中图形化查看 cv_entry[3]，第 2 个 block 中有一段将 busy 置 1，可在 TTD 调试中辅助确认。

```
000000014018B314 31 C0                       xor     eax, eax/* * rbp=VM_CONTEXT * rbx=0 偏移 * ecx=1 * * 访问VM_CONTEXT.busy */000000014018B316 F0 0F B1 0C 2B              lock cmpxchg [rbx+rbp], ecx000000014018B31B 0F 84 07 00 00 00           jz      loc_14018B328
```

定位 VM_CONTEXT.busy 后，对之设数据断点，找到离开 CV 虚拟机的 func_array[i]，再具体分析。

动态调试 CV 虚拟机时，无法通过库函数入口处的调用栈回溯寻找主调位置，因为不是 call 过来的，要么 ret 过来，要么 jmp 过来。若是 ret 过来的，在库函数入口处 qwo(@rsp-8) 应该等于 rip，据此识别此类情况。若是 jmp 过来的，运气好的话，r 查看寄存器，应该有某个其他寄存器等于 rip。若是 "jmp [rax]" 这种，除非一个个检查，很难一眼识别出来。即使识别出怎么过来的，也无助于寻找主调位置。若是 TTD 调试，直接断在库函数入口，然后 "t-" 就找到主调位置，对付 CV，必须上 TTD。

假设 CV 虚拟机通过 ret 调用外部库函数，在 ret 指令处，qwo(@rsp) 即库函数地址，一般 qwo(@rsp+8) 是库函数重入 CV 虚拟机的点，这取决于库函数怎么维持栈平衡并返回。可提前在 "重入 CV 虚拟机的点" 设断，很可能是另一个 cv_entry[0] 或 cv_entry[1]。

7) 分析 func_array[i]

IDA 中将 func_array[i] 整成函数，图形化查看，快速确定函数入口、出口。

写 IDAPython 脚本批量处理 OSF 的 func_array[i] 时，碰上很多 p 操作失败的情形，一般是 64 位操作数前缀所致，比如

```
/* * 失败情形 */000000014C46CABF 49                          db  49h000000014C46CAC0 81 CE 04 00 00 00           or      esi, 4000000014C46CAC6 49 C7 C2 00 04 00 00        mov     r10, 400h
```

```
/* * 成功情形 */000000014C46CABF 49 81 CE 04 00 00 00        or      r14, 4000000014C46CAC6 49 C7 C2 00 04 00 00        mov     r10, 400h
```

在 IDA 中先 p 一下，会提示

```
000000014C46CABF: The function has undefined instruction/data at the specified address.
```

跳至 0x14C46CABF，选中其后指令一起先 u 后 c，再回到函数入口 p 即可。

TTD 调试，在函数入口、出口分别执行如下命令，并用 BC 对比结果

```
r;dqs <stack> l 0n21;db <VM_CONTEXT> l 0x174
```

最好是 VM_DATA 附近的值。用 BC 对比，快速找出发生变化的数据，比如 push/pop 会导致 rsp 变化，push 会导致栈中数据变化，func_array[i] 可能同步修改 VM_CONTEXT.rsp，某个 pop 可能更新 VM_CONTEXT.rcx，等等。基于变化的数据，很可能直接猜中 func_array[i] 大概在干什么。TTD 调试，在 func_array[i] 出口处对发生变化的数据设数据断点，反向执行，找出其变化的逻辑。

基本上每个 func_array[i] 都含有干扰静态分析的代码，比如这种

```
mov rcx, [rax]xor rcx, 0x13141314sub rcx, 0x51201314mov [rsi], rcx          // 保存中间值...mov rdx, [rdi]          // 取出中间值，rdi等于之前的rsiadd rdx, 0x51201314xor rdx, 0x13141314
```

一堆垃圾代码互相夹杂着，实际做的是 "mov rdx,[rax]"。这是个最简情形，OSF 中 func_array[i] 更复杂，xor/add/sub 的 op2 不再是常量，而是 [reg]，reg 的值也是各种混淆、反混淆得来。不管怎么复杂，混淆与反混淆总是对称出现，用 TTD 调试相对更容易发现规律。

EAGLE_BLACK 虚拟机的 func_array[i] 功能很单一，OSF 所用 CV 虚拟机在这方面非常变态，会将 imul/add/sub/mov/deref 等操作组合到某个 func_array[i] 中，无法对单个 func_array[i] 标定唯一功能，这是对抗逆向工程的手段之一。

7.1) 复杂 func_array[i] 的简单示例

本来我尽量避免在本文中展现大段汇编代码，但有时为了产生感性认识，不上例子就差点意思。下面是 OSF 中某个复杂 func_array[i]，是个几合一功能的，其中一个功能是 add 操作，具体到本例，是 "add rcx,rax"。如此复杂的逻辑，整下来就干了这么一件简单的事，妥妥地对抗逆向工程。

(略)

为了突出要点，上述代码已做了极大精简，看上去仍很复杂。我是按执行顺序从上到下展示汇编指令，若细看，会发现指令地址并非单向递增的。若非借助 TTD 调试，很难分析透。

分析并标定 func_array[i] 功能是 CV 逆向工程中最繁琐的部分，相当枯燥。借助 TTD 调试，只要耗下去，肯定能分析清楚，就是性价比太低。

OSF 有个 func_array_2[0x653]，这么多 handler，一个个分析过去，会死人的。

即使成功标定了所有 func_array[i] 的所有功能，意义也很有限。几合一的功能函数，断在入口时几乎无法确定后续走哪个功能流程，不是简单的 switch 逻辑。

8) 静态定位 func_array[i] 出口

某些 CV 虚拟机的 func_array[i] 相对简单，IDA 缺省能识别函数出口。OSF 所用 CV 对单个 func_array[i] 做了大量 block 切分操作，就是两三条指令一个 block，然后 jmp 到下一个 block，各 block 之间非物理连续，一会儿前一会儿后的。这种在 IDA 中用图形模式看还可以，但图形模式只能看函数，若代码片段不属于函数，就得设法 p 出函数来，还得确保 p 出来的函数确实包含完整的代码。block 切分使得 IDA 识别函数时包含完整代码的能力下降，可以 "Append function tail" 人工添加 block 到指定函数中。

OSF 中 func_array[i] 大多极其复杂，IDA 缺省无法将之 p 成函数，很容易找到函数入口，但肉眼极难找到函数出口。可以写 IDAPython 脚本从函数入口开始进行类似 "全连接图非递归广度优先遍历" 的操作，寻找间接跳转或 ret 指令，以此定位 func_array[i] 出口。可用同样的技术从函数入口开始自动 c 操作，直至出口，再自动 p，因为 OSF 中大量代码未被 IDA 识别，缺省以数据形式展现。

有 2 个出口的 func_array[i] 一般对应 jxx 操作。

8.1) 全连接图的非递归广度优先遍历

(略)

9) 寻找流程分叉点

已知流程会过 [addr_a,addr_b] 区间，我想在此区间单步执行每一条指令，每次单步后想执行一些命令，比如检查相关寄存器值并据此做出不同动作。

该需求与 ta/pa 命令无关，这两个命令无法在每次单步后执行指定命令。也与 wt 命令无关。

编辑 tcmd.txt 如下

```
.if(@rip==@$t1){}.else{r $t0,rip;r $t0=@$t0+1;t "$$< tcmd.txt"}
```

在 addr_a 处执行如下命令

```
t "r $t0=0;r $t1=<addr_b>;$$< tcmd.txt"
```

效果是，从 addr_a 单步执行至 addr_b，每次都执行 "r $t0,rip;r $t0=@$t0+1"，输出中会看到一堆 "$t0=… rip=…"。

这只是示例，根据原始需求调整 tcmd.txt 的内容，比如当 rax 等于特征值时停止单步执行，修改每次单步时所执行的命令，等等。这是土法 "Run Trace" 功能。

配合. logopen/.logclose，对指定范围内被执行的指令进行定制化记录，当流程因 in 不同而不同时，对两次 log 进行 BC 比较，快速找出分叉点。根据 [addr_a,addr_b] 的具体情况，将 t 换成 p，避免失焦。

我用类似技术快速定位了 jnz 操作对应的 func_array[i] 的分叉点。当时看到的代码片段是这样的

```
/* * esi=0x202 源自VM_CONTEXT.efl * * 0x40是ZF */000000014006BEE5 81 E6 40 00 00 00           and     esi, 40h000000014006BEEB 0F 85 31 00 00 00           jnz     loc_14006BF22
```

参看

```
https://en.wikipedia.org/wiki/FLAGS_register
```

10) 反向执行寻找 pushfq

调试 OSF 对试用期过期天数反向溯源时，有次通过数据断点反向找到 0x14BCAF348 处的代码。在 IDA 中图形化查看其所在 func_array[i]，TTD 调试对比过入口、出口处相关数据区，注意到 VM_CONTEXT.efl 有变，合理猜测该函数可能提供 add 或 sub 操作。但该函数相当复杂，IDA 中静态查看，很难找到原始的 add 或 sub 操作。

基于已积累的经验，add 或 sub 操作之后必有 pushfq，从 0x14BCAF348 处反向执行，找到 pushfq，其低址方向的指令就会揭示究竟是 add 还是 sub，或是其他什么操作。

```
/* * func_array[0x45] * * sub操作，会更新VM_CONTEXT.efl(0x14bbf9a67) */000000014BCA98C7                         imul_add_sub_mov_n_14BCA98C7.../* * r8d=0x278d00 (30天对应的秒数) * r13d=0x3d862 (已过去的秒数) * * 0x278d00-0x3d862=0x23b49e (以秒为单位的过期时间) */000000014BCB0222 45 29 E8                    sub     r8d, r13d000000014BCB0225 E9 AA EC FF FF              jmp     loc_14BCAEED4000000014BCAEED4 9C                          pushfq.../* * r13d=0x23b49e (以秒为单位的过期时间) * r12=0x14bbf99bd VM_CONTEXT.rcx * * t- "r $t0=0;$$< tcmd_1.txt" * * 用上述命令反向定位pushfq指令，更快的办法是 * * ba w1 /1 @rsp;g- */000000014BCAF348 45 89 2C 24                 mov     [r12], r13d
```

撰写本文时，意识到反向寻找 pushfq 最简办法是，TTD 调试，在 0x14BCAF348 处执行

```
ba w1 /1 @rsp;g-
```

当时不知哪里想岔了，用一个笨办法。编辑 tcmd_1.txt 如下

```
.if(by(@rip)==0x9c){}.else{r $t0,rip;r $t0=@$t0+1;t- "$$< tcmd_1.txt"}
```

在 0x14BCAF348 处执行如下命令

```
t- "r $t0=0;$$< tcmd_1.txt"
```

笨办法也能成功反向定位 pushfq。单就前例而言，不推荐笨办法，但 tcmd_1.txt 可以定制修改以满足其他需求，这是一种非凡的、普适的调试技巧。

若非试图分享 CV 逆向工程经验，我不会进行二次总结、三次总结，也不会对总结过的经验复审，从而不会意识到自己用了个笨办法，可能相当长时间里陷入笨办法的思维定势。这正是分享、交流的意义，是反复总结的意义，是文档化的意义。

半个月前 bluerust 因故跟我感慨，年岁大了，文档化太重要。当时我就斥责他，你看，我在你们旁边耳提面命了多年，你们就是不好好听、不好好实践，仗着自己智商高、记忆力强肆无忌惮，老了后悔了吧。

11) cv_entry[3] 的函数化

CV 虚拟化过的代码会多次离开、重入 CV 虚拟机，并非只有一组 cv_entry[0] 到 cv_entry[5]。有些重入 CV 虚拟机的流程可能有各自的 cv_entry[1]，但共用一个 cv_entry[3]，OSF 就出现了这种情况。此时 IDA 缺省无法将 cv_entry[3] 函数化，手工 p 会失败，其主要原因是多个 cv_entry[1] 已经函数化，IDA 将它们共用的 cv_entry[3] 纳入它们的函数范畴。

cv_entry[1] 是否函数化的重要性远比不上将 cv_entry[3] 函数化，宁可牺牲前者，也得成全后者，有很多重要调试点位于 cv_entry[3] 与 cv_entry[4] 之间。

可以写 IDAPython 脚本，实现 OSFDeleteFunc()，遍历指定地址的反向交叉引用，对所有反向交叉引用点所在函数进行删除函数的操作。有的被共用的 cv_entry[3]，其反向交叉引用有几十个，一个个手工删除函数不现实。删干净后再在 cv_entry[3] 处 p，一般都会成功。待 cv_entry[3] 函数化成功之后，再将其反向交叉引用点所在代码片段重新函数化，这个随意，无所谓。

```
def OSFDeleteFunc ( ea, delete=False ) :    for x in idautils.XrefsTo( ea, 0 ) :        if ida_xref.fl_JN == x.type :            func    = ida_funcs.get_func( x.frm )            if func is not None :                print( hex( func.start_ea ) )                if delete :                    ida_funcs.del_func( func.start_ea )
```

☆ 后记

分析 CV 虚拟化时容易像无头苍蝇一样乱转，较好的方式是，给自己定几个比较明确的目标，比如

```
a) 调用外部库函数时如何组织、传递形参b) 调用外部库函数时字符串形参是否涉及反混淆c) 调用外部库函数时如何离开、重入CV虚拟机d) 寻找test/cmp这类操作对应的func_array[i]e) 寻找jxx指令对应的func_array[i]
```

带着这些具体问题去分析 CV 虚拟化，搞清楚后在暴破场景能派上用场。CV 虚拟机虽然每次都变形，但总有一些套路是不变的。

本文给出的各种技巧与思路是普适的、提纲挈领的，刻意避免陷入只见树木不见森林的境地。我是按照给从未接触过 CV 逆向工程的新手进行可操作式科普来撰写本文的，有意上手者，对照本文进行一次 CV 逆向工程实战，可快速入门。

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

[#调试逆向](forum-4-1-1.htm) [#VM 保护](forum-4-1-4.htm)