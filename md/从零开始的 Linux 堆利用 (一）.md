> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1399142-1-1.html)  ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒  

感觉栈相关简单的漏洞基本原理学的差不多了，准备学一学堆相关的；

这个主要是学习 Linux Heap Exploitation 时的笔记，具体的课可以去 Udemy 上看，感觉讲的蛮不错的；

然后内容都是自己的博客，原文在 [https://hack1s.fun/](https://hack1s.fun/)，欢迎大家去看

introduction
------------

### Glibc

`ldd`是 list dymic dependencies，可以显示出二进制程序运行时需要加载的动态链接库

libc 是 linux 中最基本的动态链接库，绝大多数程序都需要用到 libc，如果删除 libc 的链接，关机都会关不掉；

### malloc

堆是程序在执行中可以使用 malloc 向内核请求一段连续的内存空间；

I/O、文件读写等等都是通过堆来实现的；

### 堆与 malloc

首先需要对堆有一个基本的理解，堆通过 malloc 分配 chunk，通过 free 来释放 chunk

首先是一个 demo 例子用于理解堆

在 pwndbg 中执行

```
set context-sections code

```

这样可以让之后每一次显示 context 只显示源代码部分

![](https://static.hack1s.fun/images/2021/02/17/image-20210217212638040.png)

这个程序主要做的事情就是调用了几次 malloc，之后 return

`vmmap`可以显示出进程当前的内存空间，在这一句 malloc 还没有执行的时候，程序内存空间不存在堆的区域

![](https://static.hack1s.fun/images/2021/02/17/image-20210217212950029.png)

当第一次 malloc 执行之后，查看 vmmap 会发现多出来了一个堆的空间

![](https://static.hack1s.fun/images/2021/02/17/image-20210217213700269.png)

在 pwndbg 中的命令`vis_heap_chunk`简写为`vis`可以查看堆的 chunk 分布

![](https://static.hack1s.fun/images/2021/02/17/image-20210217214128093.png)

我们虽然是执行了`malloc(9)`，申请了 9 的空间，但是实际上给了我们 3*8=24 字节的空间（蓝色部分的第一个 8 字节是头部，不是用户可以用的）

也就是说`malloc(9)`分配给了 24 字节的 user data 以及 8 个字节的 meta data，这个 chunk 一共占了 32 字节；

`malloc`分配的最小 chunk 就是这样 0x20 的大小，即 24 字节的 user data 和 8 个字节的 meta data

即使执行的是`malloc(0)`、`malloc(1)`仍然会分配一个 0x20 大小的 chunk

![](https://static.hack1s.fun/images/2021/02/17/image-20210217214720913.png)

图中几个 chunk 分别是`malloc(9)`、`malloc(1)`、`malloc(0)`、`malloc(24)`分配的；

可以看到实际上内容都是一样的占了 0x20 字节；

但是也可以注意到，其中 meta data 部分并不是存储了 0x20，而是 0x21；

这是因为 meta data 这里两个字段，一是 chunk size，表示整个 chunk（包含 user 和 meta 两部分）的大小，另外由于 chunk 分配时是按照 16 字节对齐的，最低位就可以用来表示其他信息；这个字段就是 previous_inuse，用来表示这一个 chunk 相邻的前一个 chunk 是否是在使用的状态，如果是就为 1，否则为 0；

下面如果继续执行`malloc(25)`会分配一个 0x30 的空间

![](https://static.hack1s.fun/images/2021/02/17/image-20210217215150684.png)

虽然最后 16 个字节只用到了 1 个字节，但还是按照 16 字节对齐进行分配的。

最后就是 Top chunk，可以看到在我们自己申请的 Chunk 之后有一个 Top Chunk 的 meta data；

并且随着一次次的申请新的堆空间，这个 Top Chunk 的大小会发生变化。

这是因为内核在分配堆的内存空间时是创建一块大的 Top Chunk，每一次用户执行的 malloc 就是压缩 top chunk 分配给用户，直到 Top Chunk 的空间不足以分配，就会再向下拓展 Top Chunk

![](https://static.hack1s.fun/images/2021/02/17/image-20210217221525951.png)

在 Top Chunk 中有一个值得注意的地方是，在 Glibc 的很多版本中，Top Chunk 的 Size 字段都是没有完整性检查的，这就是 The House of Force 的基本原理

在 2005 年，第一次出现了一篇名为 The Malloc Maleficarum 的论文，其中写了 5 种堆利用的技巧；

1.  Houses of Prime
2.  Houses of Mind
3.  Houses of Force
4.  Houses of Lore
5.  Houses of Spirit

从此之后的堆利用技巧也因此都叫 "house of  XX" 这样的形式

House of Force
--------------

### 原理

house of force 的原理就是前面提到的，没有对 Top Chunk 的 size 字段进行完整性检查；

这导致在分配了一个比较小的 chunk 后，如果输入的内容大于 chunk 的大小，进而溢出到 top chunk 的 size 字段，就可以伪造控制 top chunk 的大小；

之后再一次使用`malloc`分配 chunk，可以达到一个任意地址写的效果，运用得当也可以实现 RCE 的效果；

### 漏洞程序本身

程序本身是一个类似 CTF 中堆题的结构

![](https://static.hack1s.fun/images/2021/02/26/image-20210226160340641.png)

为了方便学习漏洞本身，程序运行前输出了`puts`的地址以及`heap`开始的地址

选项 1 是`malloc`，之后可以输入要申请的大小以及输入的内容

例如上面申请了 24，但是输入了 24 个`a`以及 7 个`b`最后和一个`\n`

这是我们`<C-c>`后回到 pwndbg，可以用 vis 看到现在的堆

![](https://static.hack1s.fun/images/2021/02/26/image-20210226160534387.png)

可以看到由于溢出了 7 个字节的 b 和 1 个字节的`\x0a`，top chunk 处的 size 已经被覆盖了。

在 GDB 中使用`vmmap libc`可以查看程序调用的 libc 信息

这个程序由于增加了`Runpath`，连接的是特定的 libc

![](https://static.hack1s.fun/images/2021/02/26/image-20210226193254435.png)

这里使用的是没有开启 tcache 的程序，但是实际上 house of force 是可以在 tcache 存在的 libc 使用的；

这里使用这样没有开启 tcache 的 libc 是为了在还没有学过 tcache 机制的情况下就可以了解如何使用这个漏洞利用方式

### 任意地址写

程序的第二个选项可以输出一个变量 target

正常来说这个变量的值是一串 X

![](https://static.hack1s.fun/images/2021/02/26/image-20210226162944191.png)

在 pwndbg 中可以使用`dq &target`以四个字为单位查看这个变量附近的值

![](https://static.hack1s.fun/images/2021/02/26/image-20210226163057701.png)

`dq`的全称是`dump qwords`，另外也有类似的`dw`、`dd`、`db`

堆起始地址是`0x603000`，但是可以看到这里其实 target 的位置是在堆的上方的`0x602010`

使用 malloc 只能继续往高地址申请空间，没有办法摸到 target

所以我们需要溢出`top chunk`

这边由于虚拟机的环境有写问题，换了一台虚拟机

首先申请 24 的空间，然后输入`b"Y"*24+b"\xff"*8`

这样可以把 top chunk 覆盖为`0xffffffff`，在 python 脚本里面用 gdb 调试

这里给出的脚本中有几个函数是可以直接辅助在 VIM 中运行的，在 vim 输入

```
:!./% GDB

```

这个功能的实现是通过这一块代码

```
gs = '''
continue
'''

def start():
  if args.GDB:
    return gdb.debug(elf.path,gdbscript=gs)
  else:
    return process(elf.path)

```

相当于直接启动 GDB 附加这个程序，使用`vis`看到

![](https://static.hack1s.fun/images/2021/03/04/image-20210304153132039.png)

top chunk 已经是全 f 了

第二步就是申请一个特别大的 chunk，正好到 target 前面一点点的位置；

这个程序前面输出了 heap 的地址，在 pwntools 中读取之后，计算差值

需要分配的是`(0xffffffff-0x603000)+0x602010-0x20-0x20`这么大的内容

malloc 之后用 vis 查看

![](https://static.hack1s.fun/images/2021/03/04/image-20210304155216574.png)

可以看到这时正好在 0x602010 上方

这之后再 malloc 申请内存覆盖的就是 target 的地方，再申请 20 的空间，在里面随便输入一些内容

发现 vis 后 0x602010 处的值就已经不再是 XXXXXX 了

![](https://static.hack1s.fun/images/2021/03/04/image-20210304155430523.png)

在菜单里面输出 target 发现值也变了

![](https://static.hack1s.fun/images/2021/03/04/image-20210304155406422.png)

这就实现了一个任意地址写的效果

### 任意代码执行

通过一个任意地址写转换成代码执行的利用有这样几个思路：

1.  修改栈，但是这个程序中栈采用了 ASLR;
2.  修改 Binary 段，修改 PLT 中的项或修改`fini_array`，程序中的每一个函数在退出时会运行这个`fini_array`中的，但是这个程序开启了 full-RELRO，在 binary 加载完成之后原本的二进制区段会变成只读，无法对其进行修改；
3.  修改堆，但是这个程序中除了我们自己的数据，没有影响控制流的数据，所以没用；
4.  修改 libc，`__exit_funcs`、`tls_dtors_list`这两个指针会在特定情况下调用，比较类似于 PLT，但是都被指针完整性保护，并且在这个程序中没有可以触发的地方，所以难以实现；
5.  修改`__malloc_hook`，在 GLIBC 中的数据段，修改`__malloc_hook`可以使程序在调用 malloc 时调用这里被修改的函数；

这里面修改`__malloc_hook`是可行的，我们首先将 top chunk 溢出为全 f

之后申请一个空间，从堆中目前 top chunk 所在的位置到 libc 的`__malloc_hook`这么长；

由于`libc`的区段在堆的下面，不需要像获取任意地址写那样滚一圈内存空间了；

```
distance = libc.sym.__malloc_hook - 0x20 - (heap + 0x20)

```

调试的时候使用`:!./% GDB NOASLR`暂时关掉 ASLR

分配之后查看`__malloc_hook`的位置

![](https://static.hack1s.fun/images/2021/03/08/image-20210308212201801.png)

由于分配时减了 0x20，这里看一下`__malloc_hook - 2`

![](https://static.hack1s.fun/images/2021/03/08/image-20210308212444377.png)

运行`top_chunk`看到 top_chunk 的位置就在这上方

![](https://static.hack1s.fun/images/2021/03/08/image-20210308212604069.png)

所以我们接下来再用 malloc 申请一段空间，将`__malloc_hook`这里的函数指针修改为指向 system 函数的地址

```
libc.sym['system']

```

这时用`p __malloc_hook`看一下可以发现这个函数指针已经变成了`__ libc_system`

![](https://static.hack1s.fun/images/2021/03/08/image-20210308213641372.png)

下面就是想办法执行`system("/bin/bash")`

由于 system 的参数是一个指向字符串的指针，我们可以在前面几轮 malloc 填充数据时就直接填充`/bin/bash`在这里填写当时分配出来的地址

![](https://static.hack1s.fun/images/2021/03/08/image-20210308214212027.png)

比如把第二轮的 malloc 中填充的字符串改成`/bin/bash`

这样最后调用时要填写的字符串地址就是最初的`heap+0x30`

这次再执行就不需要后面的`GDB NOASLR`了，直接运行就可以拿到 shell

![](https://static.hack1s.fun/images/2021/03/08/image-20210308214407682.png)

总结一下，这里的 house_of_force 只对 2.28 以下的 GLIBC 有效，再新的 GLIBC 就增加了 top chunk 的完整性保护了；

  

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[house_of_force.zip](forum.php?mod=attachment&aid=MjI0Mjc5NHxiNjdmOWZiY3wxNjIxMzAyOTY5fDIxMzQzMXwxMzk5MTQy)

567.13 KB, 下载次数: 14, 下载积分: 吾爱币 -1 CB

二进制程序和脚本模板

 ![](https://avatar.52pojie.cn/data/avatar/001/64/18/32_avatar_middle.jpg) Linux 我是一直在虚拟机里面 用的   哈哈  水滴 pin  WiFi   学习了 ![](https://avatar.52pojie.cn/data/avatar/001/67/47/30_avatar_middle.jpg) wxyj2599 感谢楼主！最近在学 linux，有的时候有的地方比较迷，需要各大地方爬楼找，挺累的。感谢楼主！![](https://avatar.52pojie.cn/data/avatar/001/43/18/31_avatar_middle.jpg)Nightingale521 我很赞同！![](https://avatar.52pojie.cn/data/avatar/001/45/73/92_avatar_middle.jpg)莎莎啦啦 感谢分享，学习了 ![](https://avatar.52pojie.cn/data/avatar/001/63/92/77_avatar_middle.jpg) syz17213 感谢楼主分享![](https://static.52pojie.cn/static/image/smiley/default/17.gif) ![](https://avatar.52pojie.cn/data/avatar/001/65/97/68_avatar_middle.jpg) bullsh1tlie 学习了，感谢分享。![](https://avatar.52pojie.cn/data/avatar/001/66/61/11_avatar_middle.jpg)hugeljh 我想学栈，，现在还没到堆的程度 ![](https://avatar.52pojie.cn/data/avatar/000/14/50/86_avatar_middle.jpg) muscipular2021 GOOD GOOD GOOD。非常牛。收下了 ![](https://avatar.52pojie.cn/data/avatar/001/49/68/42_avatar_middle.jpg) tan567421 支持楼主，很好的教程，感谢分享。 ![](https://avatar.52pojie.cn/data/avatar/001/63/97/30_avatar_middle.jpg) 感谢楼主分享 ![](https://avatar.52pojie.cn/data/avatar/001/25/59/07_avatar_middle.jpg) jncsw 支持，感谢，学习了。