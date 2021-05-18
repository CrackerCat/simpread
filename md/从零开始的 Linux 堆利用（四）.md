> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1430344-1-1.html)  ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒 写到第四篇了，也不知道有没有人看，再贴一下自己博客的链接 https://hack1s.fun/  
这次是介绍到堆中 Unlink 时可能出现的问题  
这种 Unlink 的技巧是比较久远的年代提出的，当时还没有 NX、ASLR 这样的安全机制，我们这次也用到旧版本的 glibc  

Unsorted Bin
------------

与 Top Chunk 相邻的 chunk，在 Free 之后如果不属于 fastbin，就会直接合并到 Top Chunk 中

![](https://static.hack1s.fun/images/2021/03/23/image-20210323151654862.png)

fastbin 最大是可以放 0x80，这边申请两个大小为 0x90 的 chunk，之后 free 与 top chunk 相邻的那个

![](https://static.hack1s.fun/images/2021/03/23/image-20210323151755991.png)

可以看到这是直接将空间收回了；

下面我们再执行两个 malloc，重新申请 b 的空间，以及再申请一个新的 0x20 大小的空间；

这时堆中的内容是下面这样

![](https://static.hack1s.fun/images/2021/03/23/image-20210323152024792.png)

这时如果将 a 的空间释放，a 就被链接到了 unsorted bins 上面

![](https://static.hack1s.fun/images/2021/03/23/image-20210323152331967.png)

这里发生变化的主要有三个地方；

*   被释放的块 a 的用户空间中的两个字，变成了两个指针（绿色框）
*   与 a 靠近的块 b，指示位`previous_inuse`，值从原本的 91 变成了 90（蓝色框）
*   与 a 靠近的块 b，原本空着的一个字变成了`previous_size`，也是 90（红色框）

unsortedbin 是一个双向链表的结构，因此 a 中这两个指针分别是 backward 和 forward；

目前只有 a 一项，所以是同一个值，这个值就是`main_arena`中的 top 这个字段；

![](https://static.hack1s.fun/images/2021/03/23/image-20210323154000475.png)

我们继续执行，把 b 也 free 了，这时并不会再 unsortedbin 中加两项，而是会把 ab 两个块合并在一起

![](https://static.hack1s.fun/images/2021/03/23/image-20210323154354084.png)

可以看到直接是修改了块 a 的大小，变成了 120，最后一个 chunk 的`prev_size`也是变成了 120

free 掉一个 fastbin 是对周边的 chunk 没有影响的，但是 free 常规的 chunk，是对周围的值有影响的；

Unsafe Unlink
-------------

前面一部分介绍了 unsortedbin，我们可以注意到它是通过一个链表实现的；

![](https://static.hack1s.fun/images/2021/03/23/image-20210323165239257.png)

链表大概就是这样的一个结构；

fd、bk 分别是双链表的指针，指向前后的块；

如果想要拆下来其中的一个块，只需要把它前一个块的 fd 改成它自身的 fd，把它后一个块的 bk 改成它自身的 bk 就可以；

![](https://static.hack1s.fun/images/2021/03/23/image-20210323170813482.png)

在低版本的 GLIBC 中没有对这个过程进行检查，并且是通过宏来实现的；

并且由于 fd、bk 这两个指针本身的位置是一个正常块的 user data 部分，所以这里是存在伪造的可能的；

### 示例程序

下面就用一个例子来看一下；

![](https://static.hack1s.fun/images/2021/03/23/image-20210323190151013.png)

程序和之前一样是一个菜单形式

1 是 malloc 申请空间，2 是可以输入对应空间要填充的内容，3 可以 free 掉申请的 chunk

这个程序的 malloc 申请的 chunk 限制了大小在 120~1000 字节之间；

0x78 是 120 比特，也就是说正好是 fastbin 放不下的 chunk

另外还有一点就是，这个漏洞 unsafe_unlink 提出的时候还没有 NX，这个程序为了讲最基本的原理也关闭了 NX

![](https://static.hack1s.fun/images/2021/03/27/image-20210327110922700.png)

### 任意地址写

首先是程序的漏洞所在，就是没有进行输出内容长度的判断，申请 a、b 后输入很多内容回溢出到下一个 chunk

这个类似于之前的 House_of_force，但是这个程序限制了 malloc 的次数，所以没办法像 House_of_force 一样利用；

![](https://static.hack1s.fun/images/2021/03/29/image-20210329155553056.png)

那么我们还是先设法获取一个任意地址写的能力；

我们这次一共只能申请两次 chunk

在前面的 demo 程序中也了解到了，远离 top chunk 的块 a 被 free 之后会加入到 unsortedbin

并且会出现这几个修改

*   还在使用中的块 b，size filed 中 prev_inuse 位置零；
*   被释放的块 a，user data 最后一个字变成块 a 的大小，即 prev_size 字段；
*   被释放的块 a，user data 的前两个字节变成两个指针，分别是 fd 和 bk；

这时`free(b)`会把两个块合并，实际上执行的操作是

1.  查看 b 的 prev_inuse，发现位 0，前一个块已经被释放；
2.  查看 b 的 prev_size，找到块 a 的起始地址；
3.  把 a 从 unsortedbin list 上卸载下来，即按照 fd、bk 去修改前后块的地址

那么我们一步一步来，首先申请 a、b 两个 chunk

编辑 chunk_a 的数据，把 b 的 prev_inuse 位清 0

```
a = malloc(0x88)
b = malloc(0x88)

edit(a,b'a'*0x88+p64(0x90))

```

可以看到这时 a 的最后一个字已经属于 b 了

![](https://static.hack1s.fun/images/2021/03/29/image-20210329161824457.png)

下面把`prev_size`这个字段也伪造了，只需要修改刚才的`edit`那句

```
edit(a,b'a'*0x80+p64(0x90)*2)

```

![](https://static.hack1s.fun/images/2021/03/29/image-20210329162148950.png)

接下来我们需要填入两个指针，修改`chunk_a`的 fd 和 bk 两个字段；

在 unlink 的时候实际上是

```
this.fd->bk = this.bk;
this.bk->fd = this.fd;

```

这样两个写入操作，如果我们想要用第一个语句来写

那么`fd+0x18`是最终写入的地址，`bk`是写入的内容；

我们先试着直接利用这个写入堆中的数据

由于测试的时候都关闭了 ASLR，直接是用的固定的地址

我们输入的 payload 直接就拿这个固定的地址了

```
edit(chunk_A, p64(0x555555757090)+p64(0x5555557570b0)+b"a"*0x70+2*p64(0x90))
free(chunk_B)

```

![](https://static.hack1s.fun/images/2021/03/29/image-20210329163725241.png)

执行完毕之后，用 vis 可以看到 A、B 一起都被 free 掉了

并且可以看到我们输入的两个值都写到内存里了，`fd+0x18`写入了`70b0`，`bk+0x10`写入了`7090`

说明确实是可以写的；

这可以说是一个有限制的任意地址写；

限制在于要写的数据也会被当成地址来做一个解析，并且会尝试往那个地址写内容；

### 命令执行

接下来尝试将这个转换为一个命令执行的漏洞

如果我们直接在`__free_hook`写入`system`的话，会导致 system 之后的内容也被写入一些内容

这样由于尝试在不可写的地方写入导致出错；

不过由于这个程序是 2000s 时的程序，NX 没有开启，因此我们可以直接把 shellcode 写到内存里面

之后把`__free_hook`覆盖成堆里面的地址

```
edit(chunk_A, p64(fd) + p64(bk) + shellcode + p8(0)*(0x70-len(shellcode)) + p64(0x90)*2

```

修改一下要 edit 的内容

shellcode 的话由于`bk+0x10`会被写入为`fd`的值，shellcode 中需要空出来一小段区域，这个可以通过汇编加一个小 jmp 实现

```
shellcode = asm("jmp shellcode;"+ "nop;"*0x20+ shellcraft.execve("/bin/sh"))

```

这里的 nop 至少要把`bk+0x10`这里的 8 个字节都空出来，最小值的话应该是`0x18-len("jmp shellcode")`

`jmp shellcode`汇编代码是两个字节，所以这里至少要是 0x16

![](https://static.hack1s.fun/images/2021/03/31/image-20210401101944829.png)

可以看到图中红框是 shellcode 的内容，绿框是指向 shellcode 地址的一个指针，即 bk

下面执行`free(b)`

![](https://static.hack1s.fun/images/2021/03/31/image-20210401102202252.png)

看到堆中 a 和 b 都已经被释放掉了，并且`bk+0x10`的位置已经变成了`fd`即`__free_hook`的值

使用 u 可以将`__free_hook`中的内容当作 code 打印出来，可以看到已经是我们 shellcode 的内容了

![](https://static.hack1s.fun/images/2021/03/31/image-20210401102941350.png)

这时继续执行，再调用一次 free 就会执行我们的 shellcode

最终返回了一个 shell

![](https://static.hack1s.fun/images/2021/03/31/image-20210401102517211.png)

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[unsafe_unlink.zip](forum.php?mod=attachment&aid=MjI3ODk5MnwwODA0YjMwYXwxNjIxMzAyOTc2fDIxMzQzMXwxNDMwMzQ0)

9.44 KB, 下载次数: 4, 下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/data/avatar/001/51/39/58_avatar_middle.jpg)

> [呆毛王与咖喱棒 发表于 2021-4-30 12:29](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38318593&ptid=1430344)  
> 就随便找的一个，主题不太重要，主要是多写 &#129315;

想要你那种的 我自己的用的 emlog 很简陋 求你了大佬 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) UNKNNOW

> [UNKNNOW 发表于 2021-4-30 08:45](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38312058&ptid=1430344)  
> 有人看有人看 大佬你博客用的是 wordpress 的哪个主题呀

就随便找的一个，主题不太重要，主要是多写 &#129315; ![](https://avatar.52pojie.cn/data/avatar/001/63/60/37_avatar_middle.jpg) 呆毛王与咖喱棒 感谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/44/72/73_avatar_middle.jpg) 666666666 ![](https://avatar.52pojie.cn/data/avatar/001/62/36/23_avatar_middle.jpg) 鹏总不惯病 ![](https://static.52pojie.cn/static/image/smiley/default/42.gif)![](https://static.52pojie.cn/static/image/smiley/default/42.gif)我啥也不是，啥也不会 ![](https://avatar.52pojie.cn/data/avatar/001/55/22/34_avatar_middle.jpg) eonezhang 学无止境，感谢分享！！！![](https://avatar.52pojie.cn/data/avatar/001/15/96/37_avatar_middle.jpg)LENY77777 膜拜，这种好帖 ![](https://avatar.52pojie.cn/data/avatar/001/15/96/37_avatar_middle.jpg) kmzwyong12 分析的很经典啊！！！！![](https://avatar.52pojie.cn/data/avatar/000/97/45/64_avatar_middle.jpg)cqzhaodaxio 不明觉厉！膜拜大佬！![](https://avatar.52pojie.cn/data/avatar/001/55/29/57_avatar_middle.jpg)cqzhaodaxio 最近正在学习 linux，感谢分享 666 ![](https://avatar.52pojie.cn/data/avatar/000/22/14/40_avatar_middle.jpg) 灵魂守卫 学无止境，感谢分享！！