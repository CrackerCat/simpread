> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/rrrfff/article/details/78073392)

        Android 阵营新出机型的 cpu 基本都是 64 位了，虽然可以向下兼容 armeabi-v7a，但是使用 32 位的 so 毕竟不能充分发挥 64 位 cpu 的潜力，所以以后 arm64-v8a 用的会越来越多。但是整个安卓生态圈似乎还没有**开源发布**的 ARM64 内联 HOOK 方案，所以自己动手写了个，姑且取名 [And64InlineHook](https://github.com/rrrfff/And64InlineHook) 吧，需要注意的是仍然是 Alpha 版。

        关于 Inline Hook 的背景知识这里不再阐述，我仅支持了相对简单的函数首指令改写方案，原则上也支持函数任意位置 HOOK(需自行处理堆栈)，不支持短函数，也暂不支持 AArch32 state。

        由于 arm64-v8a 指令集都是固定的 32 位，指令地址也因而是 4 字节对齐，相比变长指令处理起来会相对容易，但是没办法直接操作 PC 寄存器，A64 处理跳转起来会比 A32 和 T32 繁琐。

        首先是计算需要覆盖的指令数，我的方案是计算目标函数和转接函数的距离，然后决定是使用 B 直接转跳 还是 BR 寄存器绝对转跳，其中 B 只会影响一条指令，是比较优的路径，但是它的转跳范围不能超过 0b00001111111111111111111111111100；BR 需要占用 4-5 条指令 (就我目前的方案而言，即把地址直接写在代码里)，因为我们需要设法把地址加载到寄存器，这就需要类似下面的操作，然后为什么是 5 条指令? 因为对于 LOAD 操作我们需要考虑内存对齐，于是可能需要一条额外的 NOP 指令来填充。

```
0x00000000  LDR X, #0x4
0x00000004  BR X
0x00000008  DCD ...
0x0000000C  DCD ...

```

上面 X 是 64 位寄存器，显而易见这会造成寄存器污染，所以我们需要挑一个尽可能不被使用的寄存器，也就不需要去设法保护它，这里我选了 X17，其实 X16 也行。

        接着就是重头戏指令修复。把覆盖掉的指令一条条复制到我们的跳板 _Trampoline_ 里，并对所有含 PC-relative address 操作数的指令进行重定位。至于哪些指令需要被修复，你可以查阅 armv8a 指令手册 (我上传了份从 ARM 官网下载的 pdf，应该是最新修订的，[http://download.csdn.net/download/rrrfff/9992241](http://download.csdn.net/download/rrrfff/9992241)) 或者 在线文档 [http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.100069_0608_00_en/index.html](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.100069_0608_00_en/index.html)。

         大概有以下几类指令，分别由__fix_branch_imm、__fix_cond_comp_test_branch、__fix_loadlit、__fix_pcreladdr 完成具体修复，具体可以去看源码，不过可能会比较乱，当然了这玩意后期基本不需要什么维护，也别想着以后 ARM128 还能用:

```
// branch_imm
b
bl
// compbranch
cbz
cbnz
// condbranch
b.c
// loadlit
ldr
ldr
ldrsw
prfm
// pcreladdr
adr
adrp
// testbranch
tbz
tbnz
// condbranch
beq
bne
bcs
bhs
bcc
blo
bmi
bpl
bvs
bvc
bhi
bls
bge
blt
bgt
ble

```

其中值得一提的大概有几点，一是__fix_loadlit，因为涉及到 LOAD 操作，自然要考虑内存对齐，然而 LOAD 的目的寄存器可能是 32、64 甚至 128 位，所以需要考虑的情形就有 3 种；二是对于 BL 这种带链接跳转，如果转化成 BR 需要先把返回地址写到 LR(X30) 里；此外就是代码中引用的地址可能就在我们覆盖的范围内，这种情况很少，但是还是简单处理了下，对相应地址进行了修复，参考 process_fix_map。