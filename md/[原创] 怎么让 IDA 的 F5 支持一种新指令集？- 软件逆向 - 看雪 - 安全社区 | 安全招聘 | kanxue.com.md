> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281568.htm)

> [原创] 怎么让 IDA 的 F5 支持一种新指令集？

引言
==

在逆向工程领域，IDA Pro 是一款广受赞誉的反汇编和调试工具，它支持多种主流的指令集，为开发者和安全研究人员提供了强大的分析能力。然而，一些特殊的指令集，如 VMP（Virtual Machine Protection）指令集，可能并不在 IDA 的支持列表中。目前，随着攻防对抗技术的发展，许多自定义的 VM 指令集被开发并应用在真实项目，迫切需要一种方法，扩展 IDA 的反编译器。

本文目的是介绍一种让 IDA 反编译支持新指令集的简单方法，能够一定程度上缓解新指令集无反编译器的困境。该方法是并非笔者原创，而是笔者对许多前辈的文章和材料做出的一个总结，起到一个抛砖引玉的作用，旨在帮助想在 IDA 中实现未知指令集反编译的朋友。

⚠️ 本文方法限定的指令集范围是一种 IDA 未知的指令集，**读者应当具备该指令集的完整知识，至于如何获取该指令集的完整知识（对于 VM 来说就是 VM 指令集的逆向）的过程不在本文的讨论范围！**

已有方法
====

IDA 处理器模块 与 Ghidra 插件
---------------------

IDA Pro 虽然支持用户开发特定架构的处理器模块，却不能使用其反编译功能，并且编写特定架构的处理器模块过程复杂，参考资料少。

Ghidra 是一款优秀的开源反编译器，用户通过插件的形式添加新指令的反编译器，例如有一个插件实现了 Ghidra 中反编译 WASM 模块 [1]，然而，编写 Ghidra 插件仍然是一项复杂的工作，参考资料少，并且不在本文限定的 “IDA” 范围。

wasm 反编译
--------

WebAssembly （WASM）是一种最近比较流行的底层指令集，主要运行在浏览器环境，也有部分运行在终端或嵌入式设备。一般，开发者使用编译型语言，例如 C/C++、Rust、Go 开发 WASM 上层程序，再使用 LLVM 将其编译为 WASM 模块。因此，WASM 的 IDA 反编译方法是一个很有代表性的方法。

IDA Pro 8.4 仍然没有加入对 WASM 的反编译支持，然而有资料表明，IDA Pro 能够反编译 WASM 模块 [2][3]。目前让 IDA Pro 支持 WASM 反编译的方法是使用 wasm2c [4] 程序，将 WASM 模块转换成等价的 C 语言低级表示形式，再使用 GCC/CLANG 编译该代码，最后使用 IDA 分析最终产物。

![](https://bbs.kanxue.com/upload/attach/202404/617255_7WQJCCN9QMCY6P6.webp)

为什么不直接阅读 wasm2c 的结果呢？wasm2c 确实能够将 wasm 模块转换成等价的 C 语言形式，然而这种形式是其 wasm 汇编的等价表示形式，可读性极差。因此，需要使用 C 语言编译器将其优化再编译成一种 IDA 能够识别的指令集，最后使用 IDA 逆向分析最终产物。

如下图就是用该方法反编译的一个效果图。

![](https://bbs.kanxue.com/upload/attach/202404/617255_DBQZ9FYNTZBKB6T.webp)

上图是一个 wasm 模块中 strlen 函数的反编译结果，strlen 所需的循环结构清晰可见，这种方法确实实现了让 IDA 反编译一种未知指令集的效果。然而，美中不足的是，wasm2c 并没有将内存访问 load 和 store 以一种原生 C 语言形式表示，这导致内存访问在 IDA 反编译结果中以函数形式呈现，这可能是由于 wasm2c 需要保证转换结果的正确性，然而，对于反编译来说，可以采用更加激进的内存访问行为，这将有利于提升 wasm IDA 反编译效果。

方法
==

笔者受到已有方法启发后，认为为了给新指令集快速实现一个 IDA 中的反编译器，可以先将未知指令集转换成一种 IDA 能够识别的指令集，例如 x86 或 ARM。然而直接转换成 x86 或 ARM 仍然有太多细节处理，因此，可以采取先将指令集，以函数单位，转换成等价的 C 语言表示形式的方法来完成未知指令集向已知指令集的转换。

具体举例来说，如下图所示的例子，转换器输入未知指令集的汇编指令，将其转换成等价 C 语言低级层形式。

![](https://bbs.kanxue.com/upload/attach/202404/617255_W5FMW7VJZTUQUER.webp)

对于寄存器，可以映射成 C 语言中的局部变量或全局变量，对于内存模型可以映射成 C 语言中的 memory 数组访问或指针访问，对于赋值指令、运算指令可以简单转换成 C 表达式，对于内存访问指令，可以转换成 C 语言的数组访问或指针访问的形式，对于标志寄存器，可以定义一组相关的宏来根据结果修改标志寄存器（C 编译器的优化功能会将未使用的标志寄存器相关的代码作为 dead code 删除！）。

具体如何映射，还需要根据特定的指令集来设计，在编写转换器的时候，尽量保证转换的正确性，剩下的反编译效果，则交给 C 编译优化和 IDA 内部的反编译优化器来解决

实践例子
====

笔者选择了 4 个比较有代表性的 VM 指令集例子，另外 wasm 的 IDA 逆向方法本身也是一个很好的例子，因为 wasm 的 IDA 反编译方法代表了 real world 大型复杂指令集。而笔者选择的指令集更加偏向于比赛场景，与 real world 中的指令集有一定程度差距，但是适合快速上手实践，同时笔者也希望读者能够在更多 real world 指令集上尝试这种方法。

例子 1 直接转换成 x64 汇编
-----------------

这个例子的指令集来自 QWB S5 vmnote ，笔者将该 VM 指令集直接转换成 x64 汇编，使用 IDA 反编译转换结果，最后成功发现程序中的漏洞。

该 VM 指令集除了基础指令以外，还有一些特殊指令，例如 `getchar` 从标准输入读取一个字符，`putchar` 向标准输出输出一个字符，`putInt` 向标准输出输出一个整数，这些指令都是 x64 指令集没有的。

<table><thead><tr><th>opcode</th><th>伪代码</th><th>替代指令</th></tr></thead><tbody><tr><td>0x3</td><td>getchar reg</td><td>mov reg, [0xbbccdd]</td></tr><tr><td>0x17 / 0</td><td>putchar reg</td><td>mov [r11+0xaa], reg</td></tr><tr><td>0x17 / 1</td><td>putInt reg</td><td>mov [r11+0xaa2], reg</td></tr></tbody></table>

笔者的主要思想是将这些 x64 中不存在的功能性指令，替换成 IDA 能够正常识别且处理数据流的指令集。在反编译结果中，可以看到对内存 0xbbccdd 的访问，从而看出来 getchar 的值赋值给了哪个变量或者表达式。

另外，还需要对调用约定进行一些转换，使 IDA 能够以 x64 的调用阅读处理函数调用。具体来说，vm 指令的寄存器经过映射后，使用 rdi 寄存器来传递返回值，然而，x64 调用一般使用 rax 传递返回值。只需要添加一条简单的 mov 指令即可完成转换。

```
func:
_0x21: leave
_0x22: mov rax, rdi
ret

_0x23: call func
mov rdi, rax


```

最后，在 IDA 中查看效果如下图，笔者将 getchar 指令映射成对 0xbbccdd 的内存访问，正如预期效果一样，IDA 正确分析出了 getchar 指令的数据流向，有了 vm 反编译器，我们团队迅速发现了 vm 程序内部的漏洞。（以下截图均为 VM 未知指令集 IDA 反编译）

![](https://bbs.kanxue.com/upload/attach/202404/617255_7NEHKUNKP8R63MY.webp)  
![](https://bbs.kanxue.com/upload/attach/202404/617255_WYXD2C2W3DV325R.webp)

笔者实现该转换器的代码大概用时 3 小时，代码行数仅有三百行左右，并且代码都是重复片段，因此，这种转换方法确实能够在短时间内使 IDA 反编译未知指令集。

具体的实践代码见 [https://github.com/P4nda0s/qwb_vmnote_recompiler](https://github.com/P4nda0s/qwb_vmnote_recompiler)

例子 2 栈机反编译
----------

这个实践例子来源于祥云杯 machine，这个例子独特在于，该 vm 是一个栈机（类似于 Java），栈机没有寄存器的概念，然而转换成 C 后再编译成二进制用 IDA 逆向，效果仍然不错。

笔者直接将该例子的 vm 未知指令集转换成 C 语言等价形式。

对于 machine vm 的逆向工程内容不再赘述，有能力的读者可以自行尝试逆向。

由于篇幅有限，笔者仅展示部分指令的转换过程。对于栈机，在转换过程中需要维护一个栈（Stack），栈是一种先进后出的线性数据结构，而栈机没有寄存器，它的操作数就是存放在栈里面。

具体来说，对于一条指令机指令 Add，它先从栈顶出栈两次，取出两个操作数 a、b，执行 a+b 运算，最后将结果压入栈。

Add 转换成 C 语言的过程如下 (提示：match 是 Python 的新语法)

```
match self.instructions[pc]:
    case Add(addr):
        a = stack.pop()
        b = stack.pop()
        code.append("v{}={}+{};".format(variable, a, b))

```

完整代码见附件 case2。

最后，这个样例实现的反编译效果如下（节选）

```
int __cdecl main()
{
  // ...... 变量定义 略
  putc(73, stdout);
  putc(110, stdout);
  putc(112, stdout);
  putc(117, stdout);
  putc(116, stdout);
  putc(58, stdout);
  putc(32, stdout);
  v3 = getc(stdin);
  // ....
  v4 = getc(stdin) + (v3 << 8);
  v5 = getc(stdin) + (v4 << 8);
  v6 = getc(stdin) + (v5 << 8);
  v7 = getc(stdin) + (v6 << 8);
  // ....
  Check(v10 ^ '734f1698');
  Check(v19 ^ 0x5606035104535A0ALL);
  // ....
  Check(v28 ^ v36 ^ 0x451505A50075C58LL);
  // ....
   putc(82, stdout);
   putc(105, stdout);
   putc(103, stdout);
   putc(104, stdout);
}

```

从上面这段代码看出，IDA 识别出了输出字符和输入字符的数据流向，并且能够正确构造表达式。

例子 3 控制流与正确性
------------

这个例子来自 Google CTF 2021，笔者实现的转换器还原了控制流，同时还保证了程序转换后的正确性，转换后的程序甚至可以使用 angr 做符号执行，完成复杂的约束求解。

转换后的片段节选

```
_1: R = ~R;
_2: Z = 1;
_3: R = R + Z;
_4: R = R + Z;
_5: if(!R) goto  _38; else goto _6;
_6: R = R + Z;
_7: if(!R) goto _59; else goto _8;
_8: R = R + Z;
_9: if(!R) goto _59; else goto _10;
_10: bug();
_11: goto end;
_12: X = 1;
_13: Y = 0;
_14: if(!X) goto _22; else goto _15;

```

这个例子中，笔者将 vm 指令集的内存地址转换为 C 语言中的标号，以此来实现条件跳转。

具体完整细节见附件 [https://panda0s.top/2021/07/19/Google-CTF-2021/#CPP](https://panda0s.top/2021/07/19/Google-CTF-2021/#CPP)

例子 4 复杂的表达式
-----------

这个例子由三叶草技术小组 SYJ 完成，笔者在这里对他的内容做一个简单总结，并提供原文链接。

原文链接：[https://bbs.kanxue.com/thread-269591.htm](https://bbs.kanxue.com/thread-269591.htm)

这个例子主要处理了标号和跳转指令，并实现了对一种类 TEA 算法 的 VM 实现反编译。

IDA 对 VM 指令的反编译效果如下（节选）

```
int __cdecl main()
{
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x7B7CC140;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x2105B8C9;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0xFEA4FB66;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x7D1AE6C0;
  a0 += (a1 + ((a1 >> 5) ^ (16 * a1))) ^ 0x2775177E;
  a1 += (a0 + ((a0 >> 5) ^ (16 * a0))) ^ 0x925FBD3;
  // .....

```

总结
==

本文在前人工作的基础上，总结了一种快速实现在 IDA Pro 中反编译未知指令集的方法，该方法适用于 real world 的 wasm 指令集逆向。同时也在一些小众指令集上取得成功，实现了 “小语种” 的 IDA 反编译器。该方法能够在最短时间内编写一个未知指令集的反编译，能够缓解未知指令没有反编译器的窘境。尽管如此，该方法仍然有许多不足待改进，例如笔者还没有在 VMP 商业保护以及流行于 Android 应用程序加固领域的 Java 虚拟化保护上测试该方法，笔者认为，该方法仅适合用于对未知指令集的初步探索逆向，若要开发一个成熟的反编译器，仍然需要根据实际情况，编写 Ghidra 反编译插件或者从零开发反编译（例如脚本类字节码 Python、Lua 等）。

参考
==

1.  ghidra-wasm-plugin. [https://github.com/nneonneo/ghidra-wasm-plugin](https://github.com/nneonneo/ghidra-wasm-plugin))
2.  关于 wasm 逆向的实战方法. [https://xz.aliyun.com/t/13474](https://xz.aliyun.com/t/13474)
3.  WebAssembly Reverse. [https://panda0s.top/2021/05/14/WebAssembly-Reverse/](https://panda0s.top/2021/05/14/WebAssembly-Reverse/)
4.  wasm2c-readme. [https://github.com/WebAssembly/wabt/blob/main/wasm2c/README.md](https://github.com/WebAssembly/wabt/blob/main/wasm2c/README.md)

题外话
===

如果各位老板有学习 IDA 基础操作的需求，给大家推荐我在看雪的 IDA 基础课程  
[https://www.kanxue.com/book-section_list-156.htm](https://www.kanxue.com/book-section_list-156.htm)

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 2024-4-29 13:36 被无名侠编辑 ，原因：

[#VM 保护](forum-4-1-4.htm)

上传的附件：

*   [recompile.zip](javascript:void(0)) （782.90kb，18 次下载）