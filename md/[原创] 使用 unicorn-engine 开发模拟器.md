> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266869.htm)

> [原创] 使用 unicorn-engine 开发模拟器

使用 unicorn-engine 开发模拟器
=======================

什么是 unicorn 引擎
--------------

Unicorn 是基于 qemu 开发的一个 CPU 模拟器，支持常见的各种指令集，能在各种系统上运行。

 

GITHUB 项目地址：https://github.com/unicorn-engine/unicorn

 

官网地址：https://www.unicorn-engine.org/

 

一个中文 API 文档：https://github.com/kabeor/Micro-Unicorn-Engine-API-Documentation

 

它只是一个 CPU 模拟器，所以它的 API 非常简洁，它提供了各种编程语言的绑定，你可以选择喜欢编程语言进行开发，被加载到 unicorn 中执行的程序对内存的每一次读写，每一条指令的执行都在你的掌控之中，并且被 unicorn 加载运行的程序对这一切是完全无感知的

准备工作
----

写这篇文章的目的是帮助理解我的开源模拟器 [vmrp](https://github.com/zengming00/vmrp) 的工作原理，因此这里介绍的是 windows 下用 c 语言开发 arm 模拟器，网络上其它的 unicorn 文章也仅仅只描述了简单的应用，对于一些关键的核心问题并没有给出答案。

 

此文列举的几个案例都是为了学习单一知识点而精心设计的简单案例，刻意省去了堆栈操作，因此和实战是不一样的，在最后会介绍到堆栈操作，因此想要应用到实战，必需学习完此文的所有内容。

 

使用到的 GCC 编译器：  
https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win64/Personal%20Builds/mingw-builds/8.1.0/threads-posix/sjlj/x86_64-8.1.0-release-posix-sjlj-rt_v6-rev0.7z

 

使用的 unicorn 下载地址：  
https://github.com/unicorn-engine/unicorn/releases/download/1.0.2/unicorn-1.0.2-win32.zip

 

需要特别注意的是必需使用 1.0.2 以上版本的 unicorn，因为我在使用 unicorn 开发模拟器时 unicorn 还处于 1.0.1 版本，1.0.1 版本存在一个奇怪的 BUG 导致模拟器出现程序跑飞的情况，在 1.0.2 版本发布后再次尝试才终于成功。

 

另外使用 unicorn 一定会涉及到汇编和反汇编知识，如果有这方面的知识理解起来会更容易

helloworld
----------

高级编程语言会有类、对象等概念，这些概念是为了方便人脑的思考，总结出来的一种编程习惯，CPU 只知道如何处理最简单的指令，无论程序有多复杂编译后给 CPU 执行的都是最简单的一条条指令，通常一个 CPU 指令集也就几十条指令而已，计算机所有的功能都是通过这几十条指令完成的，汇编语言其实就是每一条指令的直接使用和一些约定的编程规范

 

你可以简单的把一条指令理解为编程语言中的一个函数

 

让我们通过几条指令来做一个加法运算

```
mov r0,1       // 将数字1送入寄存器r0
mov r1,2       // 将数字2送入寄存器r1
add r2,r0,r1   // 将r0+r1的运算结果送入寄存器r2

```

这里推荐一个学习 arm 汇编的工具：https://github.com/linouxis9/ARMStrong

 

为了方便演示，简化代码，下面的代码假设每一步都是成功的，所以没有错误处理的部分，实际应用时应该对 unicorn api 的返回值做判断

```
#include #include #include "./unicorn-1.0.2-win32/include/unicorn/unicorn.h"
 
// 指令数据在内存中的地址，你可以放在内存地址范围内的任何一个地方，
// 但是每个CPU架构都有一些特殊的内存地址区间是有特殊作用的，
// 并且unicorn要求地址必需4k对齐，因此这里我用的是0x8000
#define ADDRESS 0x8000
 
int main() {
    uc_engine *uc;
    uint32_t r2;
 
    // 汇编代码           指令
    // mov r0,1        0xE3A00001
    // mov r1,2        0xE3A01002
    // add r2,r0,r1    0xE0802001
    uint32_t code[] = {0xE3A00001, 0xE3A01002, 0xE0802001};
 
    // 将unicorn初始化为arm架构的arm指令集模式
    uc_open(UC_ARCH_ARM, UC_MODE_ARM, &uc);
 
    // 申请一块内存空间，由于unicorn要求地址必需4k对齐，所以这里申请了4K的内存，权限为全部权限（可读、可写、可执行）
    uc_mem_map(uc, ADDRESS, 1024 * 4, UC_PROT_ALL);
 
    // 将指令数据写入到模拟器内存
    uc_mem_write(uc, ADDRESS, code, sizeof(code));
 
    // 让模拟器从指定的地址开始运行，到指定的地址停止运行
    uc_emu_start(uc, ADDRESS, ADDRESS + sizeof(code), 0, 0);
 
    // 从模拟器中读取r2寄存器的值
    uc_reg_read(uc, UC_ARM_REG_R2, &r2);
    printf("r2 = %d\n", r2);
    uc_close(uc);
    return 0;
} 
```

编译方法：

```
gcc -g -Wall -m32 -o main 1.c ./unicorn-1.0.2-win32/unicorn.dll

```

注意运行时要把 unicorn.dll 和 main.exe 放在一起

```
$ ./main.exe
r2 = 3

```

可以看到，只需几个 api 调用就可以将 unicorn 运用起来，事实上在我的模拟器中使用到的 unicorn api 也只比上面这个例子多几个而已，unicorn 真的是一个非常强大又极其简单的工具

大小端字节序
------

大小端字节序是指一个大于 1 字节的数据在内存中的存放顺序，如果不搞清楚这个问题可能会在以后的编程中遇到难解的问题

 

大端模式，是指数据的高字节保存在内存的低地址中，而数据的低字节保存在内存的高地址中，这样的存储模式有点儿类似于把数据当作字符串顺序处理: 地址由小向大增加，而数据从高位往低位放; 这和我们的阅读习惯一致。

 

小端模式，是指数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中，这种存储模式将地址的高低和数据位权有效地结合起来，高地址部分权值高，低地址部分权值低。

 

可以看到，我在代码里存放指令数据里是这样写的

```
uint32_t code[] = {0xE3A00001, 0xE3A01002, 0xE0802001};

```

同样的代码编译为大端字节序时在内存中是这样的

```
E3 A0 00 01    E3 A0 10 02    E0 80 20 01

```

而编译为小端字节序时在内存中是这样的

```
01 00 A0 E3    02 10 A0 E3    01 20 80 E0

```

字节序的问题只会影响我们对内存的特殊读写，通常 ARM 都是小端模式，x86 也是小端

ARM 模式和 THUMB 模式
----------------

ARM 模式每条指令占用 4 个字节，而 THUMB 模式每条指令只占用 2 个字节，两种模式的指令是可以同时出现在同一个可执行文件中的，主要原因是使用 thumb 模式编译的文件会比 arm 模式减少大约 30% 的体积，在嵌入式设备中是很可观的，缺点自然是效率不如 arm 模式高。

 

两种模式的代码之间通常是由带'x'后辍的跳转指令自动切换 arm 与 thumb，直接混合在一起是无法运行的

函数
--

无论高级语言有多强大的语法，汇编层面最多只到函数，函数本质就是能够复用的一堆指令，因此和上面第一个例子并没有多大的区别，不同的地方在于一个函数必需要按照严格的约定编写，这样才能让其它人调用你的函数，或者你调用别人写的函数，如果不按约定随便乱来，那么互相调用将必定出问题。

 

这样的约定规则叫做 ABI(Application Binary Interface)，由于这套规则比较复杂，我只举例其中最简单最常用的规则:

*   函数的参数小于等于 4 个，那么按顺序依次放入 r0-r3 寄存器  
    void fn(a,b,c,d){}  
    参数 a 的值放入 r0 寄存器，参数 d 的值放入 r3 寄存器
    
*   如果函数有返回值，那么放在 r0 寄存器
    
*   函数内部如果使用了额外的寄存器，那么在使用前应该备份，使用后应该恢复原本的值
    

以一个简单的函数举例：

```
int add(int a, int b) {
    return a + b;
}

```

编译后是这样的：

```
add r0,r0,r1  // 完成a+b的操作并设置返回值
bx lr         // 返回（跳转到lr寄存器指向的地址）

```

函数调用必定涉及跳转指令，ARM 指令中'B'开头的指令是管跳转的，带上不同的后辍又有不同的功能，在函数调用中最常见的是 bx 和 blx 指令，其中'x'后辍指的是自动切换 ARM 模式和 thumb 模式，'l'后辍是指跳转前将下一条指令的地址保存到 lr 寄存器中，函数的跳转正是由 blx 和 bx lr 配合完成的

函数参数传递
------

在上面提到函数的参数小于等于 4 个时，参数是按顺序依次放入 r0-r3 寄存器，返回值也是放入 r0 寄存器

 

arm 寄存器是 32 位的（arm64 是 64 位），无论传递的是 char 还是 int 或者 float 类型，只要小于等于 32 位都是直接占用一整个寄存器

 

那 64 位类型是怎么传递的呢？答案是用两个寄存器，由于这里只介绍最简单最常见的参数传递，所以这样的参数传递规则我们跳过

 

重点是指针类型，通过指针类型我们可以传递任何的数据，指针表示的是一个内存地址，因此无论你是 char 指针还是 int 指针它都是 32 位的（与寄存器相同），所以指针作为参数传递时也是直接占用一整个寄存器

 

除基本数据类型外还有结构和数组，这两者实际上都是内存块，都可以通过指针访问到，因此用指针类型就能传递

 

有基本数据类型和指针就足够我们进行数据的交换了，它们都满足直接占用一整个寄存器的条件，因此函数调用最简单的规则就是使用 r0-r3 这四个寄存器，C 语言编程时如果也是按这种规则设计函数，那么它的效率也是最高的

 

超出简单规则之外的参数传递方式需要深入去研究，关键词是 ATPCS 和 AAPCS

位置无关代码
------

与位置相关的代码指的是将内存地址写死在了程序里面，这样的程序在任何一个设备上都要求在相同的内存地址才能够正确执行，这样做的好处是程序的运行效率更高。

 

相反，位置无关代码指的是一段代码无论加载到内存中的什么位置它都能够正常执行，技术原理就是使用相对位置，或者是采用一个寄存器来表示一个基础地址，其它任何操作都基于这个给定的基地址，通常这个寄存器是 r9，这样做的好处是能够实现代码的动态加载。

*   在 gcc 编译时加入 - fpic 选项即可生成位置无关代码

那么，要使用相对位置肯定就得知道自己在什么位置，怎么做呢？当然是程序指针寄存器，在 ARM 中叫 PC

 

由于 ARM 体系结构采用了多级流水线技术， 对于 ARM 指令集而言， PC 总是指向当前指令的下两条指令的地址，即 PC 的值为当前指令的地址值加 8 个字节。

```
0x1000 mov r0,pc   // 执行后r0的值并不是当前地址，而是0x1008
0x1004 mov r1,1
0x1008 mov r1,2

```

函数案例
----

这是我精心设计的一个小片段，实现的是如下的功能，通过这个案例能很好的理解位置无关代码与函数调用

```
int add(int a, int b) {
    return a + b;
}
add(add(11,22),33);

```

```
      地址       指令
1  / 0x8000  mov r2,pc      // r2得到当前地址+8的值0x8008
  |  0x8004  add r3,r2,8    // 由r2的值加上程序固定的偏移量就能得到add函数的相对地址
  |  0x8008  add r4,r2,16   // 同理，计算得到一个能绕过add函数的相对地址
   \ 0x800C  bx r4          // 绕过add函数，直接去到0x8018
 
2  / 0x8010  add r0,r0,r1   // add 函数
   \ 0x8014  bx lr          // 返回到调用处的下一条指令
 
3  / 0x8018  mov r0,11      // 给add函数传参数a
  |  0x801C  mov r1,22      // 给add函数传参数b
   \ 0x8020  blx r3         // 调用add函数
 
4  / 0x8024  mov r1,33      // 因为add返回值是通过r0传回的，因此第二次调用时只需传参数b
   \ 0x8028  blx r3         // 再次调用add函数
 
5    0x802C  mov r0,r0      // 这句相当于什么都没做

```

执行顺序是 1=>3=>2=>4=>2=>5，其中第 1 步正是位置无关代码的关键实现原理

```
#define ADDRESS 0x8000
int main() {
    uc_engine *uc;
    uint32_t r0;
 
    // 地址     汇编代码           指令
    // 0x8000  mov r2,pc         0xE1A0200F
    // 0x8004  add r3,r2,8       0xE2823008
    // 0x8008  add r4,r2,16      0xE2824010
    // 0x800C  bx r4             0xE12FFF14
    // 0x8010  add r0,r0,r1      0xE0800001
    // 0x8014  bx lr             0xE12FFF1E
    // 0x8018  mov r0,11         0xE3A0000B
    // 0x801C  mov r1,22         0xE3A01016
    // 0x8020  blx r3            0xE12FFF33
    // 0x8024  mov r1,33         0xE3A01021
    // 0x8028  blx r3            0xE12FFF33
    // 0x802C  mov r0,r0         0xE1A00000
    uint32_t code[] = {0xE1A0200F, 0xE2823008, 0xE2824010, 0xE12FFF14, 0xE0800001, 0xE12FFF1E, 0xE3A0000B, 0xE3A01016, 0xE12FFF33, 0xE3A01021, 0xE12FFF33, 0xE1A00000};
 
    uc_open(UC_ARCH_ARM, UC_MODE_ARM, &uc);
    uc_mem_map(uc, ADDRESS, 1024 * 4, UC_PROT_ALL);
    uc_mem_write(uc, ADDRESS, code, sizeof(code));
    uc_emu_start(uc, ADDRESS, ADDRESS + sizeof(code), 0, 0);
    uc_reg_read(uc, UC_ARM_REG_R0, &r0);
    printf("r0 = %d\n", r0);
    uc_close(uc);
    return 0;
}

```

运行后得到结果 r0 = 66

使用 unicorn 调用函数
---------------

使用 unicorn 的目的当然是为了能够调与目标程序进行通信，既然是通信，那么肯定是双向的，你可以调用目标程序，那反过来目标程序也一定要能够调用你，我们先来看怎么调用目标程序里面的函数

 

在上个例子中我们知道在 0x8010 地址处有一个 add 函数，现在我们来封装它，使它成为一个本地函数供我们的代码使用

```
#define ADDRESS 0x8000
int32_t add(uc_engine *uc, int32_t a, int32_t b) {
    uint32_t r0, lr;
 
    // 传参数
    uc_reg_write(uc, UC_ARM_REG_R0, &a);
    uc_reg_write(uc, UC_ARM_REG_R1, &b);
 
    // 根据函数的调用机制，要求我们必需设置一个返回点，这个返回点正是函数执行完毕的标志
    // 由于函数内部会执行到内存中的什么位置我们是不确定的（在我们这个例子中我们当然知道它会执行到哪里）
    // 并且在uc_emu_start()中也有一个停止点，这个停止点非常强硬，如果pc指针到达这个地址程序就会立刻终止
    // 因此这个地址必需是一个目标函数永远不可能执行到的点，而且这个地址又必需是在已映射的内存范围内
    // 在我这个例子中add函数永远不可能执行到ADDRESS地址，所以我将停止点设置成了ADDRESS，因此当add函数内部经由bx lr返回后
    // pc指针将会到达uc_emu_start()设置的停止点，模拟器才能停止运行，回到我们的代码
    lr = ADDRESS;
    uc_reg_write(uc, UC_ARM_REG_LR, &lr);
 
    // 在unicorn 1.0.2之前uc_emu_start()在特殊情况下不会在pc==stopAddr时立即停止
    uc_emu_start(uc, 0x8010, lr, 0, 0); 
 
    uc_reg_read(uc, UC_ARM_REG_R0, &r0); // 获取返回值
    return r0;
}
 
int main() {
    uc_engine *uc;
    uint32_t code[] = {0xE1A0200F, 0xE2823008, 0xE2824010, 0xE12FFF14, 0xE0800001, 0xE12FFF1E, 0xE3A0000B, 0xE3A01016, 0xE12FFF33, 0xE3A01021, 0xE12FFF33, 0xE1A00000};
 
    uc_open(UC_ARCH_ARM, UC_MODE_ARM, &uc);
    uc_mem_map(uc, ADDRESS, 1024 * 4, UC_PROT_ALL);
    uc_mem_write(uc, ADDRESS, code, sizeof(code));
    printf("%d\n", add(uc, 24, 37));  // 输出61
    printf("%d\n", add(uc, 86, 753)); // 输出839  
    uc_close(uc);
    return 0;
}

```

拦截函数调用
------

上一个案例讲述了由我们调用目标函数的方法，实际上目标函数内部是有可能会有一些系统调用的，系统调用正是双向通信中对方调用我们，该怎么做呢？

 

还是以之前的函数案例来研究，我们发现在 0x8020 和 0x8028 处都调用了 add 函数，这个函数的地址我们是知道的，如果能实现把 add 函数调用替换成调用我们另外实现的一个 add 函数，那不就相当于目标程序调用了系统函数？

 

CPU 是按照 PC 寄存器指向的地址来执行代码的，因此当 PC 寄存器指向 add 函数的地址时，就是我们下手的时机

 

这里需要增加一个新的 unicorn API 调用 uc_hook_add()，通过这个 API 实现控制目标程序的各种行为，比如内存读写、指令执行，以及获取 unicorn 本身的运行状态

```
#define ADDRESS 0x8000
 
void add(uc_engine *uc) {
    int32_t a, b, ret;
    uint32_t lr;
 
    // 获取参数值
    uc_reg_read(uc, UC_ARM_REG_R0, &a);
    uc_reg_read(uc, UC_ARM_REG_R1, &b);
 
    ret = a + b + 1;
 
    // 设置返回值
    uc_reg_write(uc, UC_ARM_REG_R0, &ret);
 
    // 模拟实现bx lr的功能
    uc_reg_read(uc, UC_ARM_REG_LR, &lr);
    uc_reg_write(uc, UC_ARM_REG_PC, &lr);
}
 
void hook(uc_engine *uc, uint64_t address, uint32_t size, void *user_data) {
    if (0x8010 == (uint32_t)address) {  // 当模拟器内执行到add函数地址时，进入我们的add函数进行处理
        add(uc);
    }
}
 
int main() {
    uc_engine *uc;
    uc_hook hh;
    uint32_t r0;
 
    uint32_t code[] = {0xE1A0200F, 0xE2823008, 0xE2824010, 0xE12FFF14, 0xE0800001, 0xE12FFF1E, 0xE3A0000B, 0xE3A01016, 0xE12FFF33, 0xE3A01021, 0xE12FFF33, 0xE1A00000};
 
    uc_open(UC_ARCH_ARM, UC_MODE_ARM, &uc);
    uc_mem_map(uc, ADDRESS, 1024 * 4, UC_PROT_ALL);
    uc_mem_write(uc, ADDRESS, code, sizeof(code));
 
    // 这里我在整个代码地址范围内加上单条指令的hook，每次执行这个地址范围内的指令前都会回调我们的hook函数
    // 如果你可以很明确的知道在哪个地址范围内需要hook，设置一个准确的地址范围能提升程序的运行效率
    uc_hook_add(uc, &hh, UC_HOOK_CODE, hook, NULL, ADDRESS, ADDRESS + sizeof(code));
 
    uc_emu_start(uc, ADDRESS, ADDRESS + sizeof(code), 0, 0);
 
    uc_reg_read(uc, UC_ARM_REG_R0, &r0);
    printf("r0 = %d\n", r0);
    uc_close(uc);
    return 0;
}

```

由于我们实现的 add 函数每次都会额外 + 1，所以原本的结果应该是 66 现在变成了 68

 

之前：(11+22)+33=66

 

之后：(11+22+1)+33+1=68

可变参数的函数调用
---------

最常见的可变参数函数是 printf()，如果模拟器内的代码需要调用这样的函数我们实现起来是比较复杂的，一种比较简单的方式是写一个转换函数编译成机器码加载进模拟器，当需要调用这种函数时跳转到转换函数去执行，转换函数将参数处理后转换成固定参数的函数调用，以此实现简化处理

 

我在遇到这个问题时采用的就是这种策略，将 https://github.com/mpaland/printf 编译后加载进模拟器内，这样无论 printf() 的用法有多复杂，我只需要实现 putchar() 就能实现字符串输出功能

栈内存
---

在前面的函数案例中因为是故意设计的简单函数，因此并没有涉及到栈的操作，在实际应用中一个功能稍微复杂点的函数都一定会用到栈内存，什么是栈内存呢？

 

栈是在 CPU 硬件级别上直接支持的一种数据结构和算法，通常至少由 push/pop 指令和栈指针寄存器组成，实现一种简单的内存管理机制，在编程时函数内的临时变量正是由这一机制自动分配和回收内存。

 

栈在使用时的表现类似于往一个桶里面放东西，后面放进去的东西总是最先拿出来，在程序中 push a, push b 后第一次 pop 拿出来的是 b 的数据，第二次 pop 拿出来的才是 a 的数据，无论是 push 还是 pop 指令都是往一个内存地址读写数据，因此需要有一个寄存器来保存这个内存地址，这个寄存器就是栈指针寄存器 sp，每条 push 和 pop 指令都会自动修改 sp 寄存器的值，因此初始化一个栈空间实际上就是设置 sp 寄存器的初始值。

 

栈的实现方式不同又可以分为不同的类型，在 push/pop 执行前 sp 指向的内存有没有存放数据，以此可以分为两种：

*   空栈 - 先放数据再调指针
*   满栈 - 先调指针再放数据

然后在 push/pop 执行后 sp 指向的地址是增加还是减少，又可以分为递增 / 递减两种，由此可以组合出四种栈类型：

*   满递减 FD
*   空递减 ED
*   满递增 FA
*   满递增 FA

必需要确认程序中使用的是哪一种栈类型，合理的设置 SP 寄存器初始化栈内存程序才能正确运行，在我的模拟器中使用的是满递减类型：

```
#define ADDRESS         0x8000
#define TOTAL_MEMORY    1024 * 4
#define STACK_ADDRESS   (ADDRESS+TOTAL_MEMORY)  // 栈的开始地址放在代码块之后（可以是任意未使用的内存空间）
#define STACK_SIZE      1024 * 4   // 栈内存的大小根据需要进行调整
 
uc_mem_map(uc, ADDRESS, TOTAL_MEM, UC_PROT_ALL);
uc_mem_write(uc, ADDRESS, code, sizeof(code));
 
// 设置栈
uc_mem_map(uc, STACK_ADDRESS, STACK_SIZE, UC_PROT_ALL);
uint32_t value = STACK_ADDRESS + STACK_SIZE;  // 由于是满递减类型，因此sp初始化为栈空间的最末尾
uc_reg_write(uc, UC_ARM_REG_SP, &value);
 
// ... 之后就可以进行函数调用了

```

其中栈内存的大小是可以任意调整的，决定了函数中局部变量最多能使用多少内存，通常如果局部变量少那么几 k 内存也够

内存管理 (堆内存)
----------

内存管理是一个重点知识，因为在 C 语言中 malloc() 和 free() 的地位非常高，我们与模拟器中的函数通信时像 int 之类的基本数据类型是可以直接传递的，因为它们都是值复制形式传递，但是一旦出现字符串、数组、结构体就无能为力了，因为它们传递的都是指针，实际的数据要求我们先放入一块内存中，我们可以通过 uc_mem_map() 再申请一块内存，然后用 uc_mem_write() 把数据写进去，这样做的确可以实现预期的效果，但是有两个弊端：

1.  uc_mem_map() 要求内存必需 4k 对齐，也就是说哪怕你只需要几个字节它也至少会分配 4k 内存。
2.  uc_mem_map() 和 uc_mem_write() 都必需手动管理内存地址和长度，使用起来极其不方便。

unicorn 给我们提供了另一个分配内存的方式：

```
uc_err uc_mem_map_ptr(uc_engine *uc, uint64_t address, size_t size, uint32_t perms, void *ptr);

```

这个 api 与 uc_mem_map() 相比只增加了最后的 ptr 参数，这使我们可以直接将一块存有数据的内存映射到模拟器中，省去了 uc_mem_write() 的操作，似乎这个方法要比之前的要方便，但实际上它同样要求 4k 对齐和手动管理内存地址。

 

在 arm 和 x86 都是通过内存地址访问内存，我们编程操作内存的方式是完全相同的，唯一的区别可能就是大小端字节序的问题，如果字节序不同，写一个转换程序转换一下就行了，在我们这个模拟器里无论是本机还是模拟器都是小端模式，因此两者在内存中的数据是完全相同的。

```
#define ADDRESS            0x8000
#define TOTAL_MEMORY       1024 * 1024 * 4
 
uint8_t *mem = malloc(TOTAL_MEMORY);   // 模拟器的全部内存
uc_mem_map_ptr(uc, ADDRESS, TOTAL_MEMORY, UC_PROT_ALL, mem);

```

采用这种方式给模拟器初始化内存后，我们对 mem 的读写会直接影响到模拟器里面运行的程序，同时模拟器里面的程序对内存的读写也会立刻影响到本机代码，也就是说模拟器里外都是同一片内存。

 

这里面唯一需要注意的是内存地址的区别，在模拟器里面这块内存的首地址是我们设置的 0x8000，而在模拟器外面它是由系统分配的一个地址，因此如果直接拿模拟器返回的地址当指针用的话一定是会出错的，把地址直接传递给模拟器在里面也是无法使用的，所以在与模拟器内部通信时必需要经过一次地址转换：

```
// 模拟器内部地址转换成本地指针
void *toPtr(uint32_t addr) {
    return mem + (addr - ADDRESS);
}
 
// 本地指针转换成模拟器内部地址
uint32_t toAddr(void *ptr) {
    return ((uint8_t *)ptr - mem) + ADDRESS;
}

```

前面说了 uc_mem_map_ptr() 仍然有许多限制，因此要想方便的使用 malloc() 和 free() 必需要自己实现 malloc() 和 free()，下面是一个超小的内存管理器代码：

```
typedef struct {
    uint32 next;
    uint32 len;
} LG_mem_free_t;
 
uint32 LG_mem_min;
uint32 LG_mem_top;
LG_mem_free_t LG_mem_free;
char *LG_mem_base;
uint32 LG_mem_len;
char *Origin_LG_mem_base;
uint32 Origin_LG_mem_len;
char *LG_mem_end;
uint32 LG_mem_left;
 
#define realLGmemSize(x) (((x) + 7) & (0xfffffff8))
 
// 初始化内存管理器
// baseAddress:  托管的内存的首地址，是一个模拟器内的地址
// len:          内存的总长度
void initMemoryManager(uint32 baseAddress, uint32 len) {
    printf("initMemoryManager: baseAddress:0x%X len: 0x%X\n", baseAddress, len);
    Origin_LG_mem_base = toPtr(baseAddress);
    Origin_LG_mem_len = len;
 
    LG_mem_base = (char *)((uint32)(Origin_LG_mem_base + 3) & (~3));
    LG_mem_len = (Origin_LG_mem_len - (LG_mem_base - Origin_LG_mem_base)) & (~3);
    LG_mem_end = LG_mem_base + LG_mem_len;
    LG_mem_free.next = 0;
    LG_mem_free.len = 0;
    ((LG_mem_free_t *)LG_mem_base)->next = LG_mem_len;
    ((LG_mem_free_t *)LG_mem_base)->len = LG_mem_len;
    LG_mem_left = LG_mem_len;
#ifdef MEM_DEBUG
    LG_mem_min = LG_mem_len;
    LG_mem_top = 0;
#endif
}
 
void *my_malloc(uint32 len) {
    LG_mem_free_t *previous, *nextfree, *l;
    void *ret;
 
    len = (uint32)realLGmemSize(len);
    if (len >= LG_mem_left) {
        printf("my_malloc no memory\n");
        goto err;
    }
    if (!len) {
        printf("my_malloc invalid memory request");
        goto err;
    }
    if (LG_mem_base + LG_mem_free.next > LG_mem_end) {
        printf("my_malloc corrupted memory");
        goto err;
    }
    previous = &LG_mem_free;
    nextfree = (LG_mem_free_t *)(LG_mem_base + previous->next);
    while ((char *)nextfree < LG_mem_end) {
        if (nextfree->len == len) {
            previous->next = nextfree->next;
            LG_mem_left -= len;
#ifdef MEM_DEBUG
            if (LG_mem_left < LG_mem_min)
                LG_mem_min = LG_mem_left;
            if (LG_mem_top < previous->next)
                LG_mem_top = previous->next;
#endif
            ret = (void *)nextfree;
            goto end;
        }
        if (nextfree->len > len) {
            l = (LG_mem_free_t *)((char *)nextfree + len);
            l->next = nextfree->next;
            l->len = (uint32)(nextfree->len - len);
            previous->next += len;
            LG_mem_left -= len;
#ifdef MEM_DEBUG
            if (LG_mem_left < LG_mem_min)
                LG_mem_min = LG_mem_left;
            if (LG_mem_top < previous->next)
                LG_mem_top = previous->next;
#endif
            ret = (void *)nextfree;
            goto end;
        }
        previous = nextfree;
        nextfree = (LG_mem_free_t *)(LG_mem_base + nextfree->next);
    }
    printf("my_malloc no memory\n");
err:
    return 0;
end:
    return ret;
}
 
void my_free(void *p, uint32 len) {
    LG_mem_free_t *free, *n;
    len = (uint32)realLGmemSize(len);
#ifdef MEM_DEBUG
    if (!len || !p || (char *)p < LG_mem_base || (char *)p >= LG_mem_end || (char *)p + len > LG_mem_end || (char *)p + len <= LG_mem_base) {
        printf("my_free invalid\n");
        printf("p=%d,l=%d,base=%d,LG_mem_end=%d\n", (int32)p, len, (int32)LG_mem_base, (int32)LG_mem_end);
        return;
    }
#endif
    free = &LG_mem_free;
    n = (LG_mem_free_t *)(LG_mem_base + free->next);
    while (((char *)n < LG_mem_end) && ((void *)n < p)) {
        free = n;
        n = (LG_mem_free_t *)(LG_mem_base + n->next);
    }
#ifdef MEM_DEBUG
    if (p == (void *)free || p == (void *)n) {
        printf("my_free:already free\n");
        return;
    }
#endif
    if ((free != &LG_mem_free) && ((char *)free + free->len == p)) {
        free->len += len;
    } else {
        free->next = (uint32)((char *)p - LG_mem_base);
        free = (LG_mem_free_t *)p;
        free->next = (uint32)((char *)n - LG_mem_base);
        free->len = len;
    }
    if (((char *)n < LG_mem_end) && ((char *)p + len == (char *)n)) {
        free->next = n->next;
        free->len += n->len;
    }
    LG_mem_left += len;
}

```

由于上面的 my_free() 在释放内存时要求传入释放内存的长度，与 c 语言的 free() 用法不同，因此还需要增加两个包装后的函数：

```
void *my_mallocExt(uint32 len) {
    uint32 *p;
    if (len == 0) {
        return NULL;
    }
    p = my_malloc(len + sizeof(uint32));
    if (p) {
        *p = len;
        return (void *)(p + 1);
    }
    return p;
}
 
void my_freeExt(void *p) {
    if (p) {
        uint32 *t = (uint32 *)p - 1;
        my_free(t, *t + sizeof(uint32));
    }
}

```

my_mallocExt() 和 my_freeExt() 直接替换系统的 malloc() 和 free() 进行内存管理，通过这种方式获得的内存只需要通过 **地址转换函数** 处理一下就可以在模拟器内外自由使用了。

 

一个常见的用法是传递字符串：

```
uint32_t copyStrToEmu(char *str) {
    if (!str) return 0;
    uint32_t len = strlen(str) + 1;
    void *p = my_mallocExt(len);
    memcpy(p, str, len);
    return toAddr(p);
}
 
// str 将是一个模拟器内的地址，直接传递给模拟器
uint32_t str = copyStrToEmu("test.txt");
 
// str2 将是一个本地指针，可以直接使用
char *str2 = toPtr(str);
printf("%s\n", str2);

```

结束
--

以上就是我在使用 Unicorn 用到的最核心技术，如果对实际应用的真实案例感兴趣可以研究下我的开源模拟器 [vmrp](https://github.com/zengming00/vmrp)

 

2021/04/05 23:11 中国标准时间

 

原文地址 https://github.com/zengming00/unicorn_engine_tutorial

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#调试逆向](forum-4-1-1.htm) [#系统底层](forum-4-1-2.htm) [#VM 保护](forum-4-1-4.htm) [#其他内容](forum-4-1-10.htm)