> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-962068-1-1.html)

> [md] 转自：https://xz.aliyun.com/t/5170## 前言 > WebAssembly（缩写为 * Wasm*）是基于堆栈的虚拟机的二进制指令格式。

![](https://avatar.52pojie.cn/data/avatar/000/60/80/14_avatar_middle.jpg)默小白

转自：[https://xz.aliyun.com/t/5170](https://xz.aliyun.com/t/5170)

前言
--

> WebAssembly（缩写为 _Wasm_）是基于堆栈的虚拟机的二进制指令格式。Wasm 被设计为可编程 C / C ++ / Rust 等高级语言的可移植目标，可在 Web 上部署客户端和服务器应用程序。

随着 wasm 的逐渐流行，在最近的 ctf 比赛中出现了很多 wasm 类型的逆向题。没接触过 wasm 的人会比较苦手。即使对 wasm 有一定的了解，由于 wasm 的汇编语言可读性十分差，逆向起来会非常的痛苦。本文以刚刚结束的 defcon quals 中的一道题和其他一些题为例，介绍一种 wasm 的优化分析方法，初步探究 wasm 逆向。

wasm 汇编简介
---------

wasm 是基于堆栈的虚拟机的二进制指令格式。与 x86 架构的汇编又很大的区别。反倒更加类似 python 的 opcode

下面分别是 C++，wat，以及由用 g++ 编译的 x86 assembly 代码，让我们首先对 wasm 汇编有个初步印象。

在 [WebAssembly Explorer](https://mbebenita.github.io/WasmExplorer/) 可以在线编译 wasm

C++ 代码：

```
int testFunction(int* input, int length) {
  int sum = 0;
  for (int i = 0; i < length; ++i) {
    sum += input[i];
  }
  return sum;
}

```

wat 代码：

```
(module
 (table 0 anyfunc)
 (memory $0 1)
 (export "memory" (memory $0))
 (export "_Z12testFunctionPii" (func $_Z12testFunctionPii))
 (func $_Z12testFunctionPii (; 0 ;) (param $0 i32) (param $1 i32) (result i32)
  (local $2 i32)
  (set_local $2
   (i32.const 0)
  )
  (block $label$0
   (br_if $label$0
    (i32.lt_s
     (get_local $1)
     (i32.const 1)
    )
   )
   (loop $label$1
    (set_local $2
     (i32.add
      (i32.load
       (get_local $0)
      )
      (get_local $2)
     )
    )
    (set_local $0
     (i32.add
      (get_local $0)
      (i32.const 4)
     )
    )
    (br_if $label$1
     (tee_local $1
      (i32.add
       (get_local $1)
       (i32.const -1)
      )
     )
    )
   )
  )
  (get_local $2)
 )
)

```

使用 gcc 编译的 x86 汇编：

```
push    rbp
         mov     rbp, rsp
         mov     [rbp-18h], rdi
         mov     [rbp-1Ch], esi
         mov     dword ptr [rbp-8], 0
         mov     dword ptr [rbp-4], 0
         mov     eax, [rbp-4]
         cmp     eax, [rbp-1Ch]
         jge     short loc_400766
         mov     eax, [rbp-4]
         cdqe
         lea     rdx, ds:0[rax*4]
         mov     rax, [rbp-18h]
         add     rax, rdx
         mov     eax, [rax]
         add     [rbp-8], eax
         add     dword ptr [rbp-4], 1
         jmp     short loc_01
loc_01:
         mov     eax, [rbp-8]
         pop     rbp
         retn

```

可以看到，不同于 x86 使用的寄存器加栈的架构，wasm 主要基于栈。局部变量被保存在 local 内，全局变量保存在 global 中，对变量的操作则使用 get_local 入栈，操作完再使用 set_local 设置局部变量。

接下来我们以 DEF CON CTF 2019 Quals 中的 wasm 为例，讲解一种 wasm 的逆向分析方法

拿到网址后 f12 把 wasm 下载下来，得到 wasm.wasm

反汇编
---

就像 x86 汇编，wat 的每一句机器码都能唯一的对应一句 wasm 汇编语句。

幸运的是，ida 自带 webassembly 的处理器模块。wasm 文件可以直接被反汇编。

或是使用 [wasm2wat](https://github.com/WebAssembly/wabt)，将生成一份文本格式的. wat 文件。

```
$ ./wasm2wat wasm.wasm -o wasm.wat

```

然而没有人愿意对着 "低级" 的汇编代码分析，我们更情愿看 "高级" 一点的语言。

有关 wat 文本格式的文章：[理解 WebAssembly 文本格式](https://developer.mozilla.org/zh-CN/docs/WebAssembly/Understanding_the_text_format)

反编译
---

ida 当然没有反编译模块。（事实上，正常情况下 ida 只支持 x86 和 arm 架构的反编译）

好在，我们可以用 [wasm2c](https://github.com/WebAssembly/wabt/tree/master/wasm2c) 来生反编译

[这里](https://github.com/WebAssembly/wabt)是 wabt 的项目仓库，可以基本实现 wasm，wat，c 之间的相互转换

编译好后，用如下指令得到 c 代码：

```
$ ./wasm2c wasm.wasm -o wasm.c

```

得到 wasm.c 和 wasm.h

然而此工具得到的 c 代码并不尽如人意，光是行数已经很吓人了，这道 nodb 反编译出来的 c 文件由 12000 + 行！

而且代码几乎和 wat 内容没啥区别

随便截取其中的几行，似乎只是把局部变量与全局变量换了个名字，省略了出入栈的操作。

```
//...
  u32 i0, i1, i2;
  i0 = g12;
  l64 = i0;
  i0 = p1;
  l45 = i0;
  i0 = p0;
  l55 = i0;
  i0 = l45;
  i1 = l55;
  i0 ^= i1;
  l56 = i0;
  i0 = l56;
  i1 = 3u;
  i0 &= i1;
  l57 = i0;
  i0 = l57;
  i1 = 0u;

//...

```

优化
--

这种代码显然是无法分析的，我们需要优化

这里提供一种比较简单的优化方式：用 gcc 编译后在用 ida 反编译

将之前反编译出来的 wasm.c，wasm.h，以及 wabt 项目内的 wasm-rt.h，wasm-rt-impl.c，wasm-rt-impl.h 三个文件放到同一个文件夹。

直接 gcc wasm.c 会报错，因为很多 wasm 的函数没有具体的实现。但是我们可以只编译不链接，我们关心的只是程序本身的逻辑，不需要真正编译出能运行的 elf 来。

```
$ gcc -c wasm.c -o wasm.o

```

得到的还未连接的 elf 文件 wasm.o

现在可以丢进 ida 来分析了，比之前的 wasm.c 友好很多。

![](https://xzfile.aliyuncs.com/media/upload/picture/20190515140326-279ec3e2-76d7-1.jpeg)

尽管和原始的代码差别较大，但好歹可以开始分析了。

常量
--

搜索字符串是每个逆向手接触一个程序的第一步

对于 wasm，所有的字符串会被存放在二进制文件的末尾，以此能获取一些关键的信息。

![](https://xzfile.aliyuncs.com/media/upload/picture/20190515140442-54a20750-76d7-1.jpg)

同时也能在 wasm.c 中看到这些字符串的定义

![](https://xzfile.aliyuncs.com/media/upload/picture/20190515140533-7312312e-76d7-1.png)

直接对字符串查找引用是找不到引用的，因为并不是直接对地址的引用，想找到这些字符串会困难一些。

然而开头的常量比较可疑，比赛的 flag 格式是 OOO{}，这里开头又有连续三个一样的字节，让人联想到异或或者只是一个偏移。抱着试试看的态度减了一下，直接出了 flag。。。

除了这种偷鸡的做法，细心看看代码也能看出一点端倪。

在 f24 函数中，似乎从 0x400 开始的地方获取了个字节，然后返回了这个字节减 66，刚好是跟 flag 的差距。

```
if ( g12 >= g13 )
    Z_envZ_abortStackOverflowZ_vi(80LL);
  v1 = i64_load(Z_envZ_memory, 0x400LL);
  i64_store(Z_envZ_memory, v13, v1);
  v2 = i64_load(Z_envZ_memory, 0x408LL);
  i64_store(Z_envZ_memory, v13 + 8, v2);
  v3 = i64_load(Z_envZ_memory, 0x410LL);
  i64_store(Z_envZ_memory, v13 + 16, v3);
  v4 = i64_load(Z_envZ_memory, 0x418LL);
  i64_store(Z_envZ_memory, v13 + 24, v4);
  v5 = i64_load(Z_envZ_memory, 0x420LL);
  i64_store(Z_envZ_memory, v13 + 32, v5);
  v6 = i64_load(Z_envZ_memory, 0x428LL);
  i64_store(Z_envZ_memory, v13 + 40, v6);
  v7 = i64_load(Z_envZ_memory, 0x430LL);
  i64_store(Z_envZ_memory, v13 + 48, v7);
  v8 = i64_load(Z_envZ_memory, 0x438LL);
  i64_store(Z_envZ_memory, v13 + 56, v8);
  v9 = i32_load(Z_envZ_memory, 0x440LL);
  i32_store(Z_envZ_memory, v13 + 64, v9);
  v10 = i32_load8_s(Z_envZ_memory, 0x444LL);
  i32_store8(Z_envZ_memory, v13 + 68, v10);
  v11 = i32_load8_s(Z_envZ_memory, a1 + v13);
  g12 = v13;
  --wasm_rt_call_stack_depth;
  return (unsigned __int8)(v11 - 66);

```

主逻辑在_authenticate，其中有两处比较可疑:

```
v1 = i32_load(Z_envZ_memory, 0x4D0LL);
  i32_store(Z_envZ_memory, (unsigned int)(v18 + 28), v1);
  v2 = i32_load8_s(Z_envZ_memory, 0x4D4LL);

```

还有循环结尾的：

```
if ( v11 == 69 )
    {
      g12 = v18;
      v13 = 0x4D5;
    }
    else
    {
      g12 = v18;
      v13 = 0x4DD;

```

如果 0x400 指向常量的开头，0x4d5 刚好对应 success，0x4DD 刚好对应 failure。这肯定不是巧合。事实上，0x400 的偏移都对应常量开头，在之前做的另外一些题中也能看到类似的结构。

如此一来，我们便能轻易的从密文中猜出 flag

另外两个例子
------

### simple wasm

这是去年上交大运维赛的一道 web 题，能得到一个 wasm。

直接在文件为查找字符串：

![](https://xzfile.aliyuncs.com/media/upload/picture/20190515140554-7fe1a970-76d7-1.jpg)

看到了 base64 表和密文，解码出来是

```
iodj~44h393d5fh4;e:9h6i598f798;gd<4hf?

```

看看 check 函数，有个地方可疑：

```
while ( v14 != 38 )
  {
    v4 = v14++;
    v5 = i32_load8_s(Z_envZ_memory, v4 + a1);
    f797(v21, (unsigned __int8)(v5 + 3));
  }

```

38 就是长度，下面有个 + 3，也就是凯撒密码。

减一下就能得到 flag

```
flag{11e060a2ce18b76e3f265c4658da91ec}

```

### where_u_are

这是前阵子国赛初赛中的一道 wasm 题。

![](https://xzfile.aliyuncs.com/media/upload/picture/20190515140616-8cdb1be8-76d7-1.jpg)

先找字符串，确定 main 函数位置

注意 f23 中几个函数调用的参数：

```
for ( i = 0; i < 1; ++i )
  {
    i32_store(Z_envZ_memory, (unsigned int)v11, v9);
    f99(0x1170u, (char *)(unsigned int)v11, (unsigned int)v11);
    v4 = f23(v9, (char *)(unsigned int)v11, v3);
    v7 = f24((int *)v4, (unsigned int)v11, v5);
    f64_load(Z_envZ_memory, v7 + 8);
    f64_load(Z_envZ_memory, v7);
    f64_store(Z_envZ_memory, v10);
    f64_store(Z_envZ_memory, (unsigned int)((_DWORD)v11 + 16));
    f98(0x1173u, v10, v10);
  }
  f64_load(Z_envZ_memory, v7);
  if ( 0.0 - (double)25 >= 1.0 || (f64_load(Z_envZ_memory, v7 + 8), 1.0 - (double)175 >= 1.0) )
  {
    f98(0x1186u, (unsigned int)((_DWORD)v11 + 32), (unsigned int)((_DWORD)v11 + 32));
    g10 = (signed int)v11;
  }
  else
  {
    f98(0x117Cu, (unsigned int)((_DWORD)v11 + 24), (unsigned int)((_DWORD)v11 + 24));
    g10 = (signed int)v11;
  }

```

注意 f99, f98 的参数：0x1170, 0x1173, 0x1168, 0x117C

刚好对应之前的字符串常量之间的偏移，可以断定这是 scanf 和 printf

f23 中对输入进行了一些操作：

```
for ( i = 0; i < (unsigned int)strlen(a1, (__int64)a2, v3); ++i )
  {
    for ( j = 4; ; --j )
    {
      v3 = (unsigned int)j;
      if ( j >= 0 == 0 )
        break;
      ++v11;
      v4 = i + a1;
      v5 = i32_load8_s(Z_envZ_memory, v4);
      v7 = f22(v5, v4, v6);
      a2 = (char *)(unsigned int)(4 * (v11 - 1) + 0x11E0);
      i32_store(Z_envZ_memory, (__int64)a2, (v7 >> (j % 5 & 0x1F)) & 1);
    }
  }

```

再看看 f22

```
for ( i = 0; ; ++i )
  {
    if ( i >= 32 )
    {
      v6 = 6;
      goto LABEL_11;
    }
    v4 = i;
    if ( a1 == (char)i32_load8_s(Z_envZ_memory, (unsigned int)(i + 0x400)) )
      break;
  }

```

似乎是从 0x400 的偏移找某个值，而且范围是 32？

找找 0x400 的偏移（跟之前一样，0x400 也是字符串常量的开头？）

在数据开头，看到了一个长度为 32 的表

```
.rodata:000000000006BF00 data_segment_data_0 db '0123456789bcdefghjkmnpqrstuvwxyz'
.rodata:000000000006BF00                                         ; DATA XREF: init_memory+14↑o
.rodata:000000000006BF20                 db    2

```

联想一下之前的查表，这应该是我们的输入范围。

回到 f23，看到循环内有右移 j%5 再 & 1 得操作，可能是分离每一位

加密逻辑在 f24 内：

```
v11 = -180.0;
  v12 = 180.0;
  v13 = -90.0;
  v14 = 90.0;
  for ( i = 0; i < 50; ++i )
  {
    v6 = i32_load(Z_envZ_memory, (unsigned int)(4 * i + (_DWORD)a1));
    if ( (i + 1) % 2 == 1 )
      i32_store(Z_envZ_memory, (unsigned int)(4 * ((i + 1) / 2) + v7), v6);
    else
      i32_store(Z_envZ_memory, (unsigned int)(4 * (i / 2) + v8), v6);
  }
  for ( j = 0; j < 20; ++j )
  {
    if ( (unsigned int)i32_load(Z_envZ_memory, (unsigned int)(4 * j + v7)) == 1 )
    {
      v11 = v10;
      v10 = (v10 + v12) / 2.0;
    }
    else if ( (unsigned int)i32_load(Z_envZ_memory, (unsigned int)(4 * j + v7)) == 0 )
    {
      v12 = v10;
      v10 = (v11 + v10) / 2.0;
    }
    if ( (unsigned int)i32_load(Z_envZ_memory, (unsigned int)(4 * j + v8)) == 1 )
    {
      v13 = v9;
      v9 = (v9 + v14) / 2.0;
    }
    else if ( (unsigned int)i32_load(Z_envZ_memory, (unsigned int)(4 * j + v8)) == 0 )
    {
      v14 = v9;
      v9 = (v13 + v9) / 2.0;
    }
  }

```

是不是很像二分查找？

猜测下逻辑，把输入按 32 个字符的 table 映射到一个 0-31 的数字，转二进制后，根据奇数位和偶数位分成两组，根据每一位为 1 或 0 决定二分查找的方向，两组分别在 [-180,180] 和[-90,90]区间内二分查找，找到结果为 175 和 25 时结果正确，注意精度要足够。

小结
--

初步分析了 wasm 静态分析的方法。wasm 作为一种新的指令格式，相关工具并不齐全。总体上还有很大的进步空间。

关于 wasm 的动态调试过程在网上可以查到很多教程，这里就不多做分析了。用 chrome，Firefox 等浏览器可以轻松实现 wasm 的动态调试。在动态调试的过程中能验证静态分析的一些猜想。

![](https://avatar.52pojie.cn/data/avatar/000/55/89/01_avatar_middle.jpg)gunxsword 还是第一次听说这东西, 长见识了! ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 啦啦呜 收藏 慢慢学习