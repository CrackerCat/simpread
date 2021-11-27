> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1551294-1-1.html)

> [md]## House of LoreHouse of Lore 主要是在有 UAF 漏洞的情况下，通过修改 smallbins 的 bk 实现在任意位置申请 smallbins 的利用方法。

![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg)呆毛王与咖喱棒

House of Lore
-------------

House of Lore 主要是在有 UAF 漏洞的情况下，通过修改 smallbins 的 bk 实现在任意位置申请 smallbins 的利用方法。

不过类似的思想同样也可以用在 unsortedbin 以及 largebin 上

### 实践

直接来看一下目标程序

![](https://static.hack1s.fun/images/2021/11/18/image-20211118162523024.png)

程序运行起来输出了 libc 和堆的地址，输入一个 username 之后进入常规的堆菜单

<img src="[https://static.hack1s.fun/images/2021/11/18/image-20211118164315417.png](https://static.hack1s.fun/images/2021/11/18/image-20211118164315417.png)"alt="image-20211118164315417" />

可以申请的 chunk 大小要在 0x400 以内，但是实际上最大可以申请 0x428 的空间

存在的问题主要是 malloc 的内容在 free 掉之后仍然可以使用 edit 来写

也就是存在 UAF 的问题

那么接下来就根据我们想要 edit 的 bins 类型进行分类，实现将伪造的 chunk 加入到对应的 bins 链表中，并申请到对应的空间实现任意地址写的效果。

### Unsortedbin

根据我们已经学过的知识，在我们申请一个大小处于 unsortedbin 的 chunk，并将其 free 掉之后

利用 UAF 的漏洞修改 unsortedbin 的 bk，就可以利用 [unsortebin attack](https://hack1s.fun/2021/11/05/heaplab-unsorted-bin/) 实现任意地址写

但是这里我们和 [unsortedbin attack](https://hack1s.fun/2021/11/05/heaplab-unsorted-bin/) 不同的地方在于

unsortedbin attack 中，我们关注的是 unsortedbin ulink 下来时执行的操作，并不关注这个 chunk 是否会被申请

但是这里我们确确实实是想要将这个 chunk 申请下来的

在内存中，想要修改的 target 在输入的 username 附近

![](https://static.hack1s.fun/images/2021/11/19/image-20211119153449914.png)

和前面类似的技巧，通过修改 username，将其伪造为一个合适的大小

```
username = p64（0） + p64(0xb1)
chunk_A = malloc(0x98)
chunk_B = malloc(0x88)
free(chunk_A)
edit(chunk_A, p64(0)+p64(elf.sym.user))

```

这部分代码运行可以看到 unsortedbin 的 bk 被修改为了 user 结构体的地址

![](https://static.hack1s.fun/images/2021/11/19/image-20211119160140678.png)

这之后继续 malloc 一个 0xa0 的 chunk

这时 malloc 首先会检查大小为 0xa0 的 smallbin，发现里面为空之后会检查 unsortedbin

(从 main_arena 倒着搜索)

正好发现 unsortedbin 里面的 chunk 大小为 0xa0，所以分配成功，同时写值

分配成功后可以看到 main_arena 中的 unsortedbin bk 指向了 user 结构体

![](https://static.hack1s.fun/images/2021/11/19/image-20211119163406551.png)

这时如果再申请一个 chunk，就会尝试从 user 结构体位置申请

我们之前写了一个大小为 0xb0，那么就申请一个大小为 0xb0 的 chunk

![](https://static.hack1s.fun/images/2021/11/19/image-20211119160648846.png)

但是在运行是出现了报错，这是因为我们伪造的这个 unsortedbin chunk 的 bk 是 0

![](https://static.hack1s.fun/images/2021/11/19/image-20211119164048752.png)

在 unsortedbin unlink 下来的时候`p->bk->fd=fd`会发生这样的写操作，也就是 unsortedbin attack 中的副作用

而在这里我们伪造的 unsortedbin chunk 的 bk 是一个不可写的值，因此在尝试向 0x0 这附近的地址写内容时会出错；

需要改进的地方是将这个地方填上一个合理的地址，还是直接改为 user 结构体的地址

```
username = p64（0） + p64(0xb1) + p64(0) + p64(elf.sym.user)
chunk_A = malloc(0x98)
chunk_B = malloc(0x88)
free(chunk_A)
edit(chunk_A, p64(0)+p64(elf.sym.user))
malloc(0x98)
chunk_C = malloc(0xa8)

```

正常执行，这样我们就申请到了 target 前方的 chunk

![](https://static.hack1s.fun/images/2021/11/19/image-20211119164814770.png)

最后执行一句`edit(chunk_C, p64(0)*4+b"Much Win")`

就可以将目标字符串修改了

![](https://static.hack1s.fun/images/2021/11/19/image-20211119164928830.png)

总结一下，unsortedbin 的 house of lore 有点类似之前 fastbin dup 中从任意位置申请来一个伪造的 chunk，这里面还有几个特点

*   可以利用内存中的 flag 位，并且 unused 位被置 1 的情况下也可以成功；
*   因为 fake chunk 的 unsortedbin 可以修改为一个可写的地址，可以进一步利用这个地址继续申请大小合适的 unsortedbin
*   在 glibc 2.29 及更新的版本中无法使用，因为 unsortedbin attack 被修复了

### smallbins

这一节利用 house of lore 将 fake chunk 加入到 smallbins 的链表中

smallbins 的结构与 unsortedbin 很类似

*   循环双链表
*   先进先出，即插入到 smallbin 时插入到头部，搜索时从尾部搜索

在前面 unsortedbin 的脚本基础上修改

首先需要在`free`掉 chunk_A 之后增加一句`malloc`

这里申请的大小需要大于 chunk_A

如果申请的大小正好等于 chunk_A，就会像前一节一样直接分配下这个 unsortedbin 的空间；

如果申请的大小小于 chunk_A，则会发生 remaindering，也无法将 chunk 链接到 smallbin 上

```
username = p64（0） + p64(0xb1) + p64(0) + p64(elf.sym.user)
chunk_A = malloc(0x98)
chunk_B = malloc(0x88)
free(chunk_A)

malloc(0xa8)

```

![](https://static.hack1s.fun/images/2021/11/19/image-20211119193903380.png)

可以看到这样 chunk_A 就被放进 smallbins 中了

接下来我们也使用同样的方式修改 bk 的指针为 user 的地址

```
edit(chunk_A,p64(0)+p64(elf.sym.user))
malloc(0x98)
chunk_C = malloc(0x98)

```

接下来我们如果想要申请到目标地址作为 chunk，还需要两次`malloc`

一次申请下来的是原本 chunk_A，另一次则是目标位置

但是这样运行时报错了，切换到对应的源码

![](https://static.hack1s.fun/images/2021/11/19/image-20211119195003327.png)

这个错误信息看起来很奇怪，因为提示的是 fastbin 的错误，切换到`malloc_printerr`的栈帧，可以看到实际上传进来的错误信息是`malloc(): smallbin double linke d list corrupted`

![](https://static.hack1s.fun/images/2021/11/19/image-20211119195254417.png)

查看这相关的源码，实际上我们出错的地方是这里的检查

这里的检查实际是`vicitm->bk->fd!=victim`

在我们的例子中是顺着 chunk_A 的 bk 找到 fake chunk，而 fake chunk 的 fd 并没有指向 chunk_A，因此这里的检查没有通过

修改一下 fake chunk 的 fd，让它指向 chunk_A

```
username = p64（0） + p64(0) + p64(heap) + p64(elf.sym.user)

```

这样修改 fake chunk 之后倒数第二次的`malloc`成功通过了验证

查看 smallbins

![](https://static.hack1s.fun/images/2021/11/19/image-20211119203215519.png)

同时由于 unlink 的副作用，可以看到 fake chunk 的 fd 变成了 smallbins 在 arena 中的地址

这导致在最后一次 malloc 时出了错误

所以我们需要保证这一次申请也满足`p->bk->fd==p`

```
username = p64(elf.sym.user) + p64(0) + p64(heap) + p64(elf.sym.user-0x10)

```

这样构造出来的结构可以用这两张图概括

首先是申请的 chunk_A，我们构造的数据从 chunk_A 的 bk 指向 user 结构

将 user 结构的 fd 构造为 chunk_A 的地址，这样通过了第一次的 malloc

![](https://static.hack1s.fun/images/2021/11/24/image-20211124205005658.png)

第二次则是从 user 开始的这个 chunk，它的 bk 指向 0x602000 的 chunk，这个 chunk 的 fd 处（正好是 user 的第一个字）构造成 0x602010

这样就通过了第二次 malloc 的检查

![](https://static.hack1s.fun/images/2021/11/24/image-20211124205818804.png)

这样都通过之后，最后一次申请的这个 chunk 进行编辑，就可以修改目标字符串的值

![](https://static.hack1s.fun/images/2021/11/24/image-20211124210121042.png)

这就完成了将伪造的 chunk 加入 smallbin 实现任意地址写的效果

总结一下，smallbin 的伪造方法相比 unsortedbin 主要需要多做一个 unlink 时的检查绕过；

### largebin

首先简单介绍一下 Largebin

largebin 一共有 64 个，其中有 63 个时在使用的

一个重要的不同是一个 largiebin 存储的不是一个大小的 chunk，而是一系列大小的 chunk

大小从 0x400 开始，每一个都是存储一系列的 chunk 大小

![](https://static.hack1s.fun/images/2021/11/25/image-20211125151348290.png)

并且这里有一个特点，不同的 largebin 支持的 chunk 大小范围不是相同的

负责越大空间的 largebin 支持的大小范围越大。

例如 0x400 的 largebin 支持的是 0x400-0x430 大小的 chunk

而 0x5000 的 largebin 则支持的是 0x5000-0x5ff0 这么大范围的 chunk

并且 largebin 是按照大小倒序排列的，维持有序性是使用到 skip list

![](https://static.hack1s.fun/images/2021/11/25/image-20211125160002099.png)

首先不看这张图上的`fd_size`、`bk_size`，这样的链表结构就和 unsortedbin list 很类似，fd 指向下一个 chunk，bk 指向上一个 chunk，这样构成了一个双向循环链表

skip list 则是在 fd、bk 之后又增加了两个指针，即图中的`fd_size`和`bk_size`

这样个字段并不像 fd、bk 那样挨着指向下一个 chunk，而是指向下一个不同大小的 chunk

例如第一个 0x430 的 chunk 该字段指向了第一个 0x410 大小的 chunk

接下来我们就先申请一下 largebin 看一看

首先得到两个大小 0x3f8 的 chunk

为了防止 free 的时候两个 chunk 合并，还需要增加两次填充的申请

这时 chunk_A 和 chunk_B 都会被放到 unsortedbin 中，为了让他们 sort 到 largebin

需要再申请一个更大的 chunk

```
chunk_A = malloc(0x3f8)
malloc(0x18)
chunk_B = malloc(0x3f8)
malloc(0x18)

free(chunk_A)
free(chunk_B)

malloc(0x408)

```

在 GDB 中 break 看一下

![](https://static.hack1s.fun/images/2021/11/25/image-20211125164018278.png)

可以看到 chunk_A 的`fd_nextsize`和`bk_nextsize`都是指向了自己

而 chunk_B 的`fd_nextsize`和`bk_nextsize`都是空

这里因为 chunk_A 和 chunk_B 都是在 0x400 的 largebin 中，并且两者大小相同，所以只有 chunk_A 在 skip_list 中

接下来利用 UAF 对 chunk_A 的数据进行修改，设法让在 user 处伪造的 chunk 被 link 到 larginbin 的链表上

![](https://static.hack1s.fun/images/2021/11/25/image-20211125182843576.png)

在从 largebin 中 unlink 下数据时会进行图中这几个判断

```
chunksize_nomask(victim)==chunksize_nomask(victim->fd)

```

这里的`chunksize_nomask`是不经过字节位处理比较 size 字段，也就是说伪造的 chunk 要和 chunk_A 在 size 字段完全一致

另外最后执行的是一个 unlink 操作，这里的 unlink 不是 unsafe 的宏版本，而是 safe unlink 的函数版本

相关的绕过方式我们在之前的 [Safe Unlink](https://hack1s.fun/2021/04/25/heaplab-safe_unlink/) 中已经学习过了

需要满足的是`p->fd->bk==p`以及`p->bk->fd==p`两个条件

我们都设为其本身

```
user = p64(0)+p64(0x401)+p64(elf.sym.user)+p64(elf.sym.user)
edit(chunk_A,p64(elf.sym.user))

chunk_C = malloc(0x3f8)

edit(chunk_C,p64(0)*4+b"Much Win")

```

这样构造最终就可以改写 target 的值

总结一下最后一次的 malloc 过程

*   收到请求申请的大小为 0x3f8 后去检查了 0x400 大小的 largebin
    
*   顺着 skip_list 的`bk_nextsize`找到 0x400 大小的 chunk_A
    
*   之后顺着 chunk_A 的`fd`字段找到我们构造的 fake_chunk
    
*   发现 fake_chunk 的`fd_nextsize`、`bk_nextsize`都为 Null，因此选择将`fake_chunk`分配出去
    
*   做分配前的检查，unlink 的检查，都通过后分配 fake_chunk 给用户
    

这样我们就实现了将伪造的 chunk link 到 largebin 中并实现分配的效果

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[house_of_lore.zip](forum.php?mod=attachment&aid=MjM1MTUyNnxkYzZmNjFhOHwxNjM4MDA1MjE0fDB8MTU1MTI5NA%3D%3D)

2021-11-25 19:31 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

6.29 KB, 下载次数: 1, 下载积分: 吾爱币 -1 CB

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)southerlywindly 我很赞同![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)金冰 感谢分享！！！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)jianyugege 感谢大佬分享！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)fengbu401 谢谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) bradyCC 学习了，感谢 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) gaoliying 感谢楼主的分享！很实用！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)buzhidao 学习，感谢 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) cbhblb20211111  
学习，感谢![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)小熙家的大厨 感谢楼主的分享！很实用！