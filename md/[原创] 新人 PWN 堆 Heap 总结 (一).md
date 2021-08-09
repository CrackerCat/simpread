> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268826.htm)

> [原创] 新人 PWN 堆 Heap 总结 (一)

开个堆系列，先声明一下，本人堆学的不是太好，如果有错误，烦请各位师傅指出，谢谢！

一、main_arena 总概
===============

1.arena: 堆内存本身

(1) 主线程的 main_arena: 由 sbrk 函数创建。

①最开始调用 sbrk 函数创建大小为 (128 KB + chunk_size) align 4KB 的空间作为 heap

②当不够用时，用 sbrk 或者 mmap 增加 heap 大小。

(2) 其它线程的 per thread arena: 由 mmap 创建。

①最开始调用 mmap 映射一块大小为 HEAP_MAX_SIZE（32 位系统上默认为 1MB，64 位系统上默认为 64MB）的空间作为 sub-heap。

②当不够用时，会调用 mmap 映射一块新的 sub-heap，也就是增加 top chunk 的大小，每次 heap 增加的值都会对齐到 4KB。

2.malloc_state: 管理 arena 的一个结构，包括堆状态信息，bins 链表等等

(1)main_arena 对应的 malloc_state 存储在 glibc 的全局变量中

(如果可以泄露 malloc_state 结构体的地址，那么就可以泄露 glibc 的地址)

(2)per thread arena 对应的 malloc_state 存储在各自本身的 arena

3.bins: 用链表结构管理空闲的 chunk 块，通过 free 释放进入的 chunk 块 (垃圾桶)

4.chunks: 一般意义上的堆内存块

```
#注释头
 
struct malloc_state{
mutex_t mutex;//(相当于多线程的互斥锁)
int flags;//(记录分配器的一些标志，bit0 用于标识分配区是否包含至少一个 fast bin chunk，bit1 用于标识分配区是否能返回连续的虚拟地址空间。)
mfastbinptr fastbinsY[NFASTBINS];//(一个数组，里面的元素是各个不同大小的fastbins的首地址)
mchunkptr top;//(top chunk的首地址)
mchunkptr last_remainder;//(某些情况下切割剩下来的堆块)
mchunkptr bins[NBINS*2-2];
.......................................................
unsigned int binmap[BINMAPSIZE];//(以bit为单位的数组，共128bit，16个字节，4个int，由于bin的数量为128，对于这里面的128bit，为0表该bin没用有空闲块，为1表有空闲块。通过四个int的大小可以找出不同index的bin中是否有空闲块。这个在某些时候会用到。)
......//后面还有，不太重要
}

```

▲内存中的堆情况：

全局变量 glibc:main_arena = struct malloc_state:

<table><tbody><tr><td width="72">mutes</td><td width="58">&nbsp;bin</td><td width="50">&nbsp;.....</td><td width="68">&nbsp;top</td><td width="112">&nbsp;lastremainder</td></tr></tbody></table><table><tbody><tr><td width="81">Allocated chunk</td><td width="73">Allocated chunk</td><td width="58"><p>Free</p><p>chunk1</p></td><td width="69">Allocated chunk</td><td width="57"><p>Free</p><p>chunk2</p></td><td width="76">Allocated chunk</td><td width="51"><p>Top</p><p>chunk</p></td></tr></tbody></table>

低地址 ------------------------------------------------------------- ------------------> 高地址

由 sbrk 创建的 main_arena

(1) 可以把 bin 也当作一个 chunk，不同 Bins 管理结构不同，有单向链表管理和双向链表管理。

(2)top 里的 bk 保存 Topchunk 的首地址。

(bk 和 fd 只用于 Bins 链表中，allocated_chunk 中是属于用户可以使用的内容)

二、chunk 结构：
===========

1. 在使用中的 allocated_chunk 和未被使用的 free_chunk：

(1)allocated_chunk：

![](https://bbs.pediy.com/upload/attach/202108/904686_5VYX8DN9UT783UM.jpg)

(2)free_chunk:

![](https://bbs.pediy.com/upload/attach/202108/904686_YDYBRNYF7PDS457.jpg)

2.prev_size：8 字节，保存前一个 chunk 的大小，在 allocatedchunk 中属于用户数据，参考上述的图片，free_chunk 的下一个 chunk 的 pre_size 位为该 free_chunk 的 size。

3.size:8 字节，保存当前 chunk 大小。(free 和 allocated 都有用) 一个 chunk 的 size 以 0x10 递增，以 0x20 为最小 chunk。

(1)malloc(0x01)：会有 0x20 这么大，实际用户可用数据就是 0x18。size=0x21

(2)malloc(0x01-0x18)：仍然 0x20 这么大，实际用户可用数据就是 0x18。size=0x21

(3)malloc(0x18)：会有 0x30 这么大，实际用户可用数据是 0x28。size=0x31

所以 size 这个 8 字节内存的最低 4 位都不会被用到，所以 malloc 管理机制给最低的 3 位搞了个特殊形式标志位，A,M,P，分别代表不同内容。

①A:NON_MAIN_ARENA，代表是否属于非 main_arena，1 表是，0 表否。就是线程的不同。

```
#注释头
 
#define chunk_non_main_arena(p) ((p)->size & NON_MAIN_ARENA)

```

②M:IS_MMAPPED，代表当前 chunk 是否是 mmap 出来的。

```
#注释头
 
#define chunk_is_mmapped(p) ((p)->size & IS_MMAPPED)

```

③P:PREV_INUSE，代表前一个 chunk 是否正在被使用，处于 allocated 还是 free。

```
#注释头
 
#define prev_inuse(p) ((p)->size & PREV_INUSE)

```

(标志位为 1 都代表是，0 都代表否)

三、bins 结构：
==========

1.fastbins: 放在 struct malloc_state 中的 mfastbinptr fastbinsY[NFASTBINS] 数组中。

(1) 归类方式：只使用 fd 位

①bin 的 index 为 1，bins[0],bins[1] 组成一个 bin。

②规定大小的 chunk 归到一类，但个数有限，不同版本不同，同时也可以设置其范围：

M_MXFAST 即为其最大的参数，可以通过 mallopt() 进行设置，但最大只能是 80B。

![](https://bbs.pediy.com/upload/attach/202108/904686_ZVPCWGP4EASM6T2.jpg)

(2) 单向链表：

▲例子：a=malloc(0x10); b=malloc(0x10); c=malloc(0x10); d=malloc(0x10)

FastbinY,d,c,b,a

①free(a) 之后：

```
#注释头
 
fastbinY[0x20]->a;       a.fd=0

```

②free(b) 之后：

```
#注释头
 
fastbinY[0x20]->b;       b.fd=a       a.fd=0

```

③free(c) 之后：

```
#注释头
 
fastbinY[0x20]->c;        c.fd=b       b.fd->a;    a.fd=0

```

④free(d) 之后：

```
#注释头
 
fastbinY[0x20]->d;       d.fd=c       c.fd->b;     b.fd->a;    a.fd=0

```

(3) 后进先出：

①m=malloc(0x10):          m->d

②n=malloc(0x10):           n->c

(4) 不改变 IN_USE 位 (p 位):

如果某个 chunk 进入到 fastbin 中，那么该 chunk 的下一个 chunk 的 IN_USE 位还是为 1，不会改变成 0。

例子：a=malloc(0x10); b=malloc(0x10); c=malloc(0x10);

①free(a) 之后:        b.p=1

②free(b) 之后：     c.p=1;      b.p=1

p 位不会变成 0，如果该 chunk 进入到 fastbins 中。

可以进行 free(0),free(1),free(0)，但是不能直接 free(0) 两次。

(5) 除了 malloc_consolidate 函数会清空 fastbins，其它的操作都不会减少 fastbins 中 chunk 的数量。

2.smallbins: 放在 bins[2]-bins[125] 中，共计 62 组，是一个双向链表。最大 chunk 的大小不超过 1024 字节

(1) 归类方式：

①相同大小的 chunk 归到一类：大小范围 [0x20,0x3f0]，0x20、0x30、....0x3f0。每组 bins 中的 chunk 大小一定。

②每组 bin 中的 chunk 大小有如下关系：Chunk_size=2 * SIZE_SZ * index，index 即为 2-63，下图中 64 即为 largebins 的范围了。(SIZE_SZ 在 32，64 位下分别位 4，8。)

![](https://bbs.pediy.com/upload/attach/202108/904686_8WYPRCY87YTUZRZ.jpg)

(2) 双向链表：

▲例子：a=malloc(0x100); b=malloc(0x100); c=malloc(0x100)

①free(a) 之后:          smallbin,a

```
#注释头
 
smallbin.bk->a;         a.bk->smallbin;      
samllbin.fd->a          a.fd->smallbin;

```

②free(b) 之后:        smallbin,b,a

```
#注释头
 
smallbin.bk->a;       a.bk->b    b.bk->smallbin
smallbin.fd->b;       b.fd->a    a.fd->smallbin

```

③free(c) 之后：     smallbin,c,b,a

```
#注释头
 
smallbin.bk->a;       a.bk->b    b.bk->c    c.bk->smallbin
smallbin.fd->c;       c.fd->b    b.fd->a    a.fd->smallbin

```

（fd,bk 为 bins[n]，bins[n+1]。fd 和 bk 共同构成一个 Binat。）

(3) 先进先出：

①m=malloc(0x100):                m->a

②n=malloc(0x100):                 n->b

(4) 当有空闲 chunk 相邻时，Chunk 会被合并成一个大 chunk，这里指的是物理上的地址相邻，不是链表中相邻。通过判断当前 chunk 的 in_use 位的值来判断前一个 chunk 是否处于 Free，如果是，那么合并。再通过当前 chunk 首地址加上 size 取得下一个 chunk 首地址，再将下一个 chunk 首地址加上它自己的 size，取得下下个 chunk 的首地址，然后判断下下个 chunk 的 in_use 位的值看是否下个 chunk 处于 Free，如果处于 Free，则合并。

3.largebins: 放在 bins[126]-bins[251]，共计 63 组，bin 的 index 为 64-126，最小 chunk 的大小不小于 1024 个字节。

(1) 归类方式：范围归类，例如 bins[126]-bins[127] 中保存 chunk 范围在 [0x400,0x440]。且 chunk 在一个 bin 中按照从大到小排序，便于检索。其它与 smallbins 基本一致。

①范围模式：由以下代码定义范围：

```
#注释头
 
#define largebin_index_64(sz)                                               
  (((((unsigned long) (sz)) >> 6) <= 48) ?  48 + (((unsigned long) (sz)) >> 6) :
   ((((unsigned long) (sz)) >> 9) <= 20) ?  91 + (((unsigned long) (sz)) >> 9) :
   ((((unsigned long) (sz)) >> 12) <= 10) ? 110 + (((unsigned long) (sz)) >> 12) :
   ((((unsigned long) (sz)) >> 15) <= 4) ? 119 + (((unsigned long) (sz)) >> 15) :
   ((((unsigned long) (sz)) >> 18) <= 2) ? 124 + (((unsigned long) (sz)) >> 18) :
   126)

```

②范围具体实例：

```
#注释头
 
size                           index
[0x400 , 0x440)                64
[0x440 , 0x480)                65
[0x480 , 0x4C0)                66
[0x4C0 , 0x500)                67
[0x500 , 0x540)                68
等差 0x40      …
[0xC00 , 0xC40)                96
[0xC40 , 0xE00)                97
[0xE00 , 0x1000)               98
[0x1000 , 0x1200)              99
[0x1200 , 0x1400)              100
[0x1400 , 0x1600)              101
等差 0x200    …
[0x2800 , 0x2A00)              111
[0x2A00 , 0x3000)              112
[0x3000 , 0x4000)              113
[0x4000 , 0x5000)              114
等差 0x1000 …
[0x9000 , 0xA000)              119
[0xA000 , 0x10000)             120
[0x10000 , 0x18000)            121
[0x18000 , 0x20000)            122
[0x20000 , 0x28000)            123
[0x28000 , 0x40000)            124
[0x40000 , 0x80000)            125
[0x80000 , …. )                126

```

(2) 双向链表，但是有两种排列方式：

①取用排列：

首先依据 fd_nextsize，bk_nextsize 两个指针大小找到适合的，然后按照正常的 FIFO 先进先出原则，通过 fd,bk 来排列。

②大小排列：

每个进入 largebin 的 chunk

其 chunk_addr+0x20 处即为其 fd_nextsize 指针，chunk_addr+0x28 处为其 bk_nextsize 指针。

然后通过 fd_nextsize，bk_nextsize 两个指针依据从大至小的顺序排列：

![](https://bbs.pediy.com/upload/attach/202108/904686_WXZRBHWQXBP2ZDS.jpg)

(这张图片我也忘记从哪里整的了... 侵删)

其中 size 顺序为：D>C>B>A，但是释放顺序却不一定是这样的。设置这个的原因是当申请特定大小的堆块时，可以据此来快速查找，提升性能。

(3) 特殊解链：

由于 largebin 中会存在 fd_nextsize 指针和 bk_nextsize 指针，所以通常的 largebin_attack 就是针对这个进行利用的。

借用星阑科技的一张图说明一切：

![](https://bbs.pediy.com/upload/attach/202108/904686_W33HTFGWT73PEKE.jpg)

4.unsortedbins:bins[0] 和 bins[1] 中，bins[0] 为 fd，bins[1] 为 bk，bin 的 index 为 1，双链表结构。

(1) 归类方式：只有一个 bin，存放所有不满足 fastbin，未被整理的 chunk。

(2) 双向链表：

a=malloc(0x100); b=malloc(0x100); c=malloc(0x100)

①free(a) 之后:          unsortedbin,a

```
#注释头
 
unsortedbin.bk->a;  a.bk->unsortedbin;
unsortedbin.fd->a;  a.fd->unsortedbin;

```

②free(b) 之后:        unsortedbin,b,a

```
#注释头
 
unsortedbin.bk->a;  a.bk->b     b.bk->unsortedbin
unsortedbin.fd->b;  b.fd->a     a.fd->unsortedbin

```

③free(c) 之后：     unsortedbin,c,b,a

```
#注释头
 
unsortedbin.bk->a;  a.bk->b     b.bk->c     c.bk->unsortedbin
unsortedbin.fd->c;  c.fd->b     b.fd->a     a.fd->unsortedbin

```

(3) 先进先出：

①m=malloc(0x100):                m->a

②n=malloc(0x100):                 n->b

▲依据 fd 来遍历：

如果 fd 链顺序为 A->B->C

那么解链顺序一定是先解 C，再解 B，再解 A。

5、Tcache 机制：从 libc-2.26 及以后都有: 先进后出，最大为 0x410

(1) 结构：

①2.29 以下，无 key 字段的 tcache，结构大小为 0x240，包括 chunk 头则占据 0x250:

```
#注释头
 
typedef struct tcache_perthread_struct
{
  char counts[TCACHE_MAX_BINS];//0x40
  tcache_entry *entries[TCACHE_MAX_BINS];//0x40
} tcache_perthread_struct;

```

A.counts: 是个数组，总共 64 个字节，对应 tcache 的 64 个 bin，每个字节代表对应 bin 中存在 chunk 的个数，所以每个字节都会小于 8，一般使用

```
#注释头
 
tc_idx = csize2tidx (size);
tcache->counts[tc_idx]

```

来访问索引对应 bin 的 count。

![](https://bbs.pediy.com/upload/attach/202108/904686_JE66B4J9ECMFF4D.jpg)

从 0x55555555b010 至 0x55555555b04f 都是 counts 这个数组的范围。

▲由于使用 tcache 时，不会检查 tcache->counts[tc_idx]的大小是否处在 [0,7] 的范围，所以如果我们可以将对应 bin 的 count 改成 [0,7] 之外的数，这样下回再 free 该 bin 对应大小的 chunk 时，就不会将该 chunk 置入 tcache 中，使得 tcache 不满也能不进入 tcache。

B.entries：是个 bin 指针数组，共 64 个指针数组，每个指针 8 个字节，总计大小 0x200 字节，指针指向对应的 bin 中第一个 chunk 的首地址，这个首地址不是 chunk 头的首地址，而是对应数据的首地址。如果该 bin 为空，则该指针也为空。一般会使用 tcache->entries[tc_idx] != NULL 来判断是否为空。

②2.29 及以上，在 entry 中增加了 key 字段，结构体大小为 0x290，count 由原来的一个字节变为两个字节

```
#注释头
 
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  struct tcache_perthread_struct *key; /* 新增指针 */
} tcache_entry;

```

(2) 类似于一个比较大的 fastbins。总共 64 个 bin。

(3) 归类方式：

相同大小的 chunk 归到一类：大小范围 [0x20,0x410]。每组 bins 中的 chunk 大小一定。且一组 bin 中最多只能有 7 个 chunk，如果 free 某大小 bin 的 chunk 数量超过 7，那么多余的 chunk 会按照没有 tcache 机制来 free。

(4). 单向链表：

▲例子：a=malloc(0x10); b=malloc(0x10); c=malloc(0x10); d=malloc(0x10)

FastbinY,d,c,b,a

①free(a) 之后：tcachebins[0x20]->a;   a.fd=0

②free(b) 之后：tcachebins[0x20]->b;   b.fd=a      a.fd=0

③free(c) 之后：tcachebins[0x20]->c;    c.fd=b       b.fd->a;    a.fd=0

④free(d) 之后：tcachebins[0x20]->d;   d.fd=c       c.fd->b;    b.fd->a;    a.fd=0

★但是这里的 fd 指向的是 chunk 内容地址，而不是其它的 bins 中的 fd 指向的是 chunk 头地址。

(5) 后进先出，与 fastbins 类似。且 tcache 的优先级最高。

(6) 特殊：

①当 tcache 某个 bin 被填满之后，再 free 相同大小的 bin 放到 fastbin 中或者 smallbins 中，之后连续申请 7 个该大小的 chunk，那么 tcache 中的这个 bin 就会被清空。之后再申请该大小的 chunk 就会从 fastbins 或者 smallbins 中找，如果找到了，那么返回该 chunk 的同时，会将该大小的 fastbin 或者 smallbin 中所有的 chunk 都移动到对应大小的 tcache 的 bin 中，直至填满 7 个。(移动时仍旧遵循先进后出的原则，所以移动之后 chunk 顺序会发生颠倒)

②libc-2.26 中存在 tcache poisoning 漏洞，即可以连续 free(chunk) 多次。

假如 chunk0,chunk1，然后 free(chunk0) 两次，这样 tcache bin 中就是：

chunk0.fd ->chunk0，即 chunk0->chunk0

那么第一次申请回 chunk0，修改 fd 为 fakechunk，tcache bin 中就是：

chunk0.fd->fakechunk，即 chunk0->fakechunk

之后再申请回 chunk0，再申请一次就是 fakechunk 了，实现任意地址修改。

★这个漏洞在 libc2.27 中就被修复了。

③从 tcache 中申请 Chunk 的时候不会检查 size 位，不需要构造字节错误。

(7)2.31 下新增 stash 机制：

在 Fastbins 处理过程中新增了一个 Stash 机制，每次从 Fastbins 取 Chunk 的时候会把剩下的 Chunk 全部依次放进对应的 tcache，直到 Fastbins 空或是 tcache 满。

(8)2.32 下新增 fd 异或机制：

会将 fd 异或上某个值。这个具体看其他文章吧，我自己调试大多都是异或 heap_base/0x1000，比较奇怪，具体题目具体分析。

6.Topchunk: 不属于任何 Bin，在 arena 中属于最高地址，没有其它空闲块时，topchunk 就会被用于分配。

7.last_remainder: 当请求 small chunk 大小内存时，如果发生分裂，则剩余的 chunk 保存为 last_remainder，放入 unsortedbin 中。

四、没有 tcache 的 malloc 和 free 流程：
===============================

malloc 流程：
----------

★如果是多线程情况下，那么会先进行分配区的加锁，这里就可能存在条件竞争漏洞。

1. 如果 size 在 fastbins 的范围内，则先在 fastbins 中查找，找到则结束，没找到就去 unsortedbins 中找。

2. 如果 size 不在 fastbins 范围中，而在 smallbins 范围中，则查找 smallbins（在 2.23 下如果发现 smallbin 链表未初始化，则会调用 malloc_consolidate 函数，但是实际情况在申请 chunk 之前都已经初始化过了，所以这个不怎么重要 if (victim == 0) /* initialization check */  malloc_consolidate (av); 而且这个操作从 2.27 开始已经没有了，如果能够让 smallbin 不初始化，或者将 main_arena+0x88 设置为 0），此时若 smallbin 找到结束，没找到去 unsortedbins 中找

3. 如果 size 不在 fastbins，smallbins 范围中，那一定在 largebins 中，那么先调用 malloc_consolidate 函数将所有的 fastbin 中的 chunk 取出来，合并相邻的 freechunk，放到 unsortedbin 中，或者与 topchunk 合并。再去 largebins 中找，找到结束，没找到就去 unsortedbins 中找。

4.unsortedbins 中查找：

(1) 如果 unsortedbin 中只有 last_reaminder，且分配的 size 小于 last_remainder，且要求的 size 范围为 smallbin 的范围，则分裂，将分裂之后的一个合适的 chunk 给用户，剩余的 chunk 变成新的 last_remainder 进入 unsortedbin 中。如果大于 last_remainder，或者分配的 size 范围为 largebin 的范围，则将 last_remainder 整理至对应 bin 中，跳至第 5 步。

(2) 如果 unsortedbin 中不只一个 chunk，则先整理，遍历 unsortedbins。如果遇到精确大小，直接返回给用户，接着整理完。如果一直没遇到过，则该过程中所有遇到的 chunk 都会被整理到对应的 fastbins，smallbins，largebins 中去。

5.unsortedbins 中还是找不到，则：

(1) 若当前 size 最开始判断是处于 smallbins 范围内，则再去 smallbins 找，这回不找精确大小，找最接近略大于 size 的一个固定大小的 chunk 给分裂，将符合 size 的 chunk 返回给用户，剩下的扔给 unsortedbins，作为新的 last_remainder。

(2)若当前 size 最开始判断处于 largebins 范围内，则去 largebins 中找，和步骤 (1) 类似。

(3) 若当前 size 大于 largebins 中最大的 chunk 大小，那么就去 topchunk 来分割使用。

6.topchunk 分割：

(1)topchunk 空间够，则直接分割。

(2)topchunk 空间不够，那么再调用 malloc_consolidate 函数进行整理一下，然后利用 brk 或者 mmap 进行再拓展。

①brk 扩展：当申请的 size 小于 128K 时，使用该扩展。向高地址拓展新的 topchunk，一般加 0x21000，之后从新的 topchunk 再分配，旧的 topchun 进入 unsortedbin 中。

②mmap 扩展：申请的 size 大于等于 mmap 分配阈值 (最开始为 128k) 时，使用该扩展，但是这种扩展申请到的 chunk，在释放时会调用 munmap 函数直接被返回给操作系统，而不会进入 bins 中。所以如果用指针再引用该 chunk 块时，就会造成 segmentation fault 错误。

▲当 ptmalloc munmap chunk 时，如果回收的 chunk 空间大小大于 mmap 分配阈值的当前值，并且小于 DEFAULT_MMAP_THRESHOLD_MAX（32 位系统默认为 512KB，64 位系统默认为 32MB），ptmalloc 会把 mmap 分配阈值调整为当前回收的 chunk 的大小，并将 mmap 收缩阈值（mmap trim threshold）设置为 mmap 分配阈值的 2 倍。这就是 ptmalloc 的对 mmap 分配阈值的动态调整机制，该机制是默认开启的，当然也可以用 mallopt() 关闭该机制

▲如果将 M_MMAP_MAX 设置为 0，ptmalloc 将不会使用 mmap 分配大块内存。

free 流程：
--------

★多线程情况下，free() 函数同样首先需要获取分配区的锁，来保证线程安全。

1. 首先判断该 chunk 是否为 mmaped chunk，如果是，则调用 munmap() 释放 mmaped chunk，解除内存空间映射，该该空间不再有效。同时满足条件则调整 mmap 阈值。

2. 如果 size 位于 fastbins 范围内，直接放到 fastbins 中。

(这里有点争议，在《glibc 内存管理 ptmalloc 源代码分析. pdf》一书中，写到：

如果执行到这一步，说明释放了一个与 top chunk 相邻的 chunk。则无论它有多大，都将它与 top chunk 合并，并更新 top chunk 的大小等信息。

但是实际测试，如果 size 位于 fastbins 范围内，则不管是否与 topchunk 相邻，都直接放到 fastbin 中，测试环境是 libc2.23，内核版本是 Linux version 5.10.13，也可能是某些参数的问题把，具体题目分析就好了。)

3. 如果 size 不在 fastbins 范围内，则进行判断：

(1) 先判断前一个 chunk_before，如果 chunk_before 是 free 状态的，那么就将前一个 chunk 从其对应的 bins 中取出来 (unlink)，然后合并这两个 chunk 和 chunk_before。

由于还没有进入链表结构中，所以这里寻找 chunk_before 地址是通过当前地址减去当前 chunk 的 presize 内容，得到 chunk_before 的地址，从而获取其 in_use 位。

这也是 presize 唯一的用途，所以在堆溢出中，可以只要不进行 free，presize 可以任意修改。

(这里如果 chunk_before 是位于 fastbins 中则没办法合并，因为在 fastbins 中的 in_use 位不会被改变，永远是 1，在判断时始终认为该 chunk 是处于 allocated 状态的)

(2) 再判断后一个 chunk，如果后一个 chunk 是 free 状态，那么如步骤 (1)，合并，之后将合并和的 chunk 放入到 unsortedbins 中去。如果后一个 chunk 就是 topchunk，那么直接将这个 chunk 和 topchunk 合并完事。

之后将合并之后的 chunk 进行判断，(这里也可能不发生合并，但依旧会进行判断) 如果 size 大于 FASTBIN_CONSOLIDATION_THRESHOLD(0x10000)，那么就调用 malloc_consolidate 函数进行整理 fastbins，然后给到 unsortedbins 中，等待 malloc 时进行整理。

▲32 位与 64 位区别，差不多其实，对于 0x8 和 0x10 的变化而已：

![](https://bbs.pediy.com/upload/attach/202108/904686_MMR8E3QKEC9554N.jpg)

▲至于有 tcache 的 malloc 和 free，在各个版本其实都差不太多，除开 2.29 和 2.31、2.32 三个版本的不同检查和机制，都是判断 count 和链表来着，具体结合题目会比较好。  

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 8 小时前 被 PIG-007 编辑 ，原因：