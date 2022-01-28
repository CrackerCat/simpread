> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271316.htm)

> malloc 源码分析

学了这么久堆漏洞了，我想应该把`glibc`的`malloc`和`free`源码解析写一下了，希望能帮助一下刚上路的师傅，同时也巩固一下自身知识。

内存分配
----

我们平时写程序的时候，某些变量可能需要在开始就分配内存，这些内存是不可避免的。那么这些内存就是静态分配的，当程序编译完成之后，它就确定了占用这么多的内存。但是有时候，实际问题的规模没有预期那么大，我们不一定需要很大的内存，如果每次都按最大考虑那么就有很大一部分内存是被浪费的，这就是静态分配内存的弊端，虽然咱打 acm 的时候都是静态分配的，但是这没啥，因为每个问题不要超过它的总内存上限问题就不大 (狗头。但是在内存不足的年代，如果都这样使用静态分配内存的方式，那么计算机的效率会被拖垮很多，所以就有动态分配内存的概念了。

 

`glibc`采用`ptmalloc`的管理方式去分配内存。

ptmalloc2 的分配策略
---------------

那么动态分配内存要怎么去分配呢？如果我们需要占用除了我程序本身占用的内存以外的一块内存，那程序指定是没权限用的，得先向操作系统申请这一块内存的使用权。而操作系统没那么闲，分配几个字节的内存都要它去管，操作系统管理都是按页式的管理。而一页的内存是`0x1000B`，如果每一次申请我都向操作系统申请，每一次归还都直接归还给操作系统那么必定会增大操作系统的负担。因此分配内存的时候可以按照一个策略去分配，分配一定得尽量避免过多地使用系统调用，归还的时候可以等到程序结束时一并归还，这样的话操作系统的负担就大大下降了。

 

`ptmalloc2`的分配方式会在你第一次`malloc`的时候向操作系统申请`0x21000B(132KB)`的内存，然后后续分配就不会向操作系统申请内存，只有用完了的时候才会再次申请内存。

 

操作系统的问题解决了之后我们再来看看`glibc`怎么处理具体的分配细节。分配的时候我一定是切出一块特定大小才是最优的策略的，比如程序`malloc(4)`，那我接切个 4 字节的内存给它用，`malloc(1)`那就给它一字节去使用。然而现实没有那么理想，因为如果我切下来的块用户程序完全可写的话，那么我怎么区分这个内存块是否被使用呢？然后内存块的分界线又如何界定呢？所以分割内存块的时候不可避免地要在内存块中额外开出一部分区域用于管理。那么可以在每个分配的内存块加上一个`int`数据作为此内存块的`size`，64 位的操作系统可以使用`long long`。同理，为了管理方便，`glibc`在分配`chunk`的时候也并不是分配这么多就只能写这么多的。它也不想闲到去管 1 字节 2 字节这样的内存块。而且如果有这样的内存块，那么在分配指针的时候内存没办法对齐会出现很多麻烦的事。所以在分配内存块的时候，有一个`SIZE_SZ`，一次分配的内存必定是`SIZE_SZ*2`的整倍数，`SIZE_SZ`在 32 位操作系统下的值是`4`，64 位的值是`8`。为了方便，以下把内存块统一叫`chunk`。

 

以 32 位操作系统为例，size 的值必定为 8 的整数倍，二进制角度下看来，低三位永远是 0，这样有点浪费了内存，因此规定`size`的低三位不作为实际的`chunk`大小，而是标志位。三个标志位从高位到低位分别是：

1.  `NON_MAIN_ARENA`: 是否为主分配，0 表示是主分配，权值为 4
2.  `IS_MMAPPED`: 表示内存是否为`mmap`获得，0 表示不是，权值为 2
3.  `PREV_INUSE`: 表示前面一个内存块是否被使用，0 表示不被使用，权值为 1

在 64 位操作系统中，多出一个标志位，但是这个标志位无任何意义，可能后续会赋予别的意义，但是它一样不影响`chunk`的大小。

 

在看 malloc 源码的时候可以看到一个宏定义。

```
#define SIZE_BITS (PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
/* Get size, ignoring use bits */
#define chunksize(p)         ((p)->size & ~(SIZE_BITS))

```

那么就可以看到`chunksize`在取实际`size`的时候与了一个`0xfffffff8`，忽略了最低三位，64 位操作系统则会忽略最低四位。

 

以下例子为 64 位操作系统

 

`chunk`最小的大小为`0x20`，为什么没有`0x10`大小的`chunk`呢，这么看来`size`占了`8`字节还能有 8 字节给用户去写似乎没问题。大不了我超过`8B`再分配`0x20`大小的内存嘛，这个疑问先放一下，我们来看看这样的策略它还有没有什么问题。

 

如果一个`chunk`被确定释放了，那么该以什么方式去管理。你会想到前面有一个`prev_inuse`位可以确定一个堆块是否被释放，你会想到改下一个`chunk`的标志位就可以了，但是如果这个内存块再次被需要呢，难道去遍历每一个`chunk`，一来要看`size`符不符合，二来还要看它有没有被使用，这样时间开销太大了。因为空闲的`chunk`可能散落在内存各个角落，管理零碎内存最好的办法就是链表。链表还得有表头，这个表头其实就是我们的`main_arena`中的`bin`。因此`chunk`上还得有一块内存是指针，指针又占了`8`个字节。

 

但是你可能想到，指针它只在块被释放的时候有用啊，`0x10`的块，一个`size`，一个指针，被分配的时候用指针作为数据域，被释放的时候指针用于链式管理。这样就解决了，这样也的确没问题。但是看看它这样的分配策略还有没有问题？如果我多次分配`chunk`很小的块，`free`之后它们便只能用于分配这么大的内存了。如果不加另一种策略组织起来，导致内存碎片越来越多，就容易耗尽系统内存。

 

那么就有`ptmalloc`的又一个策略：尽量合并物理相邻的`free_chunk`。咱们前面一直提到切割内存块，合并内存块就是切割的一个逆过程。在合并的时候我可能前面会有`free`的内存块，后面也会有`free`的内存块。那么我怎么在只知道我自身信息的情况下准确找到前后的`chunk`具体在哪呢。

 

想找到后面的很容易，我知道我自己所在的位置（指针），也知道我的`size`，那么相加就可以找到后面的`size`了。那么我如何找前面的`size`在什么位置呢？所以就不得不再开辟一个内存来存前一个`chunk`的信息了。通过`prev_inuse`位我很容易得知前一个`chunk`有没有被`free`，但是我并不知道前一个`chunk`的大小啊。所以在一个`chunk`的结构体，在 size 之前还会有一个`prev_size`。与前面那个指针同理，我只有在前一个块被`free`需要合并的时候才会想看看它在哪，他要是都还在用我都没必要去使用这个`prev_size`字段了。但是要注意，这个`prev_size`是服务于上一个`chunk`的。所以一个 chunk 的结构体就有`0x10`个不得不分配的字节，而且自己还不能用。因此`0x10`的`chunk`就没有意义了。所以源码中也会找到这样的定义：

```
#define MINSIZE 4*SIZE_SZ

```

说了这么多了，`ptmalloc`的策略大致总结一下就是：

1.  一次系统调用会分配大块内存
    
2.  程序结束后统一归还内存给操作系统
    
3.  方便管理，内存分配尽量对齐，也就是所谓的 size 为某某整倍数
    
4.  尽量分配最小能满足的内存块
    
5.  链式管理空闲空间，适当的时候合并物理相邻的`chunk`
    

而且根据以上分析我们可以得出一些关于`chunk`的结构体。

```
struct chunk{
    size_t prev_size;
    size_t size;
    chunk *fd;
    chunk *bk;//因为链式管理还有可能是双向链表
}

```

至此，我们大致就明白了`ptmalloc`的分配方式。

ptmalloc2 的具体分配策略
-----------------

前面我们讲到了，对于空闲块使用了链式管理方式。但是对于不同大小的`chunk`，它又有细分。这里先给一个概念：`bin`，字面意义垃圾桶，用于装`free_chunk`的垃圾桶，在这里可以理解为链表表头。

 

以下均以`glibc 2.23`版本解析

### fast bin

对于`size`较小的`free_chunk`，我们认为它很快就会被再次用到，因此在`free` `0x20~0x80`大小的`chunk`时，我们会把它扔进`fast bin`里面，字面意义，里面存的`free_chunk`很快会被再次用到。`fast bin` 管理`free_chunk`采用单链表方式，并且符合后进先出（`FILO`）的原则，比如以下程序段

```
x=malloc(0x10);
y=malloc(0x10);
free(x);
free(y);
z=malloc(0x10);
w=malloc(0x10);

```

那么 z 会得到 y 的指针，w 会得到 x 的指针。

 

并且`fast bin`的`chunk`的之后的`chunk` `prev_inuse`位永远为 1。也就是说它永远被视为在使用中，但是通常这个使用中是用于检测参不参与物理相邻`chunk`的合并，所以不会参与物理相邻的`chunk`的合并，也不会被切割。它的匹配规则就是，定量匹配。比如我想要一个`0x30`的`chunk`，没有就是没有，没有我就找其它的，不会说`0x40`好像还挺合适就拿了，不会。

 

`fast bin`一共有`10`个，`main_arena`结构体中，用`fastbinsY`来存储每一个`fast bin`的链表头部，32 位系统中，`fast bin`，从 0x10 开始到`0x40`，有 7 种`fast bin`，64 位系统从`0x20`开始到`0x80`，也是七种`fast bin`。单个`fast bin`链表上的`chunk`大小一定严格相等。

 

一定情况下可以修改`global_max_fast`的值来调整`fast bin`的个数，64 位系统下这个值通常为`0x80`，代表小于等于`0x80`的`chunk`都为`fast bin`。

 

其余的链表头部都在`bin`数组当中。并且由于只有`fast bin`是单链表结构，其余`bin`都是双向链表结构，`bin`会成对出现。

### unsorted bin

对于非`fast bin`大小的`chunk`，被释放时会首先进入`unsorted bin`。`unsorted bin`在特定的时候会进入`small bin` 和 `large bin`。

 

非`fast bin`的`bin`都是用一对`bin`指针来描述的，这两个`bins`也要看成一个`chunk`，然后初始它们的`fd`和`bk`都指向自身的`prev_size`那个位置。比如`main_arena+104`这个地方是`bin`数组的第一个，然后呢`main_arena+104`和`main_arena+112`分别就是`unsorted bin`的头部，它们本身虽然不是`chunk`，但是要理解它们的初始状态还是得看成一个`chunk`。所以`main_arena+104`和`main_arena+112`的初始值就是`main_arena+88`。如图：

 

![](https://bbs.pediy.com/upload/attach/202201/919002_CCDKX9SW8FTTBH5.jpg)

 

设置这一个`bin`的主要目的是扮演一个缓存层的角色以加快分配和释放的操作，链表中`chunk`大小不一定相等且无序排列。

 

当需要检查`unsorted bin`的时候，会遍历整个链表，寻找第一个能满足的`chunk`大小切割。如果切割后的大小不足`2*SIZE_SZ`，则不会切割，而是将整个堆块返回给用户使用。

### small bin

一共有`62`个，从最小的`chunk`开始，公差为`SIZE_SZ*2`，双链表管理。它的特点也是跟 fast bin 一样，单条链表`chunk`大小相等，但是它会参与合并，切割。先进先出（`FIFO`）的策略。它表示的范围就是`4*SIZE_SZ~126*SIZE_SZ`

### large bin

`large bin`与`small bin`不一样，`large bin`表示的是一个范围。一共有`63`个 (假设下标`0~62`)，从`small bin`最小不能表示的`chunk`开始，大到无穷。

 

它表示的范围类似一个等差数列。

<table><thead><tr><th>起下标</th><th>止下标</th><th>公差</th></tr></thead><tbody><tr><td>0</td><td>31</td><td>16*SIZE_SZ</td></tr><tr><td>32</td><td>47</td><td>32*SIZE_SZ</td></tr><tr><td>48</td><td>55</td><td>64*SIZE_SZ</td></tr><tr><td>56</td><td>59</td><td>128*SIZE_SZ</td></tr><tr><td>60</td><td>61</td><td>256*SIZE_SZ</td></tr><tr><td>62</td><td>62</td><td>∞</td></tr></tbody></table>

 

最小的`large bin`是`small bin`的最小不能表示的大小。

 

所以`large bin`从`128*SIZE_SZ`开始。那么下标为`0`的`large bin`表示的范围就是`128*SIZE_SZ~144*SIZE_SZ`(左闭右开)，同理下标为 1 的`large bin`表示的范围就是`144*SIZE_SZ~160*SIZE_SZ`，以此类推，等到`32`的时候就在原来的基础上加`32*SIZE_SZ`作为右开区间

 

它会以二维双向链表进行维护，对于`bin`中所有的`chunk`，相同大小的`chunk`用`fd`和`bk`指针相连，对于不同大小的`chunk`，采用`fd_nextsize`和`bk_nextsize`指针连接。并且沿着`fd_nextsize`指针，`chunk`大小递增。

### top_chunk

我们之前说过，第一次`malloc`的时候，操作系统会给我们`0x21000B`的内存，它是作为一个`top_chunk`存在的，可以把`top_chunk`看成`heap`的边界。`top_chunk`的地址会被记录在 main_arena+88 的位置。`gdb`中通过`p/x main_arena`的命令也可以查看`main_arena` 的具体结构。

### 分配流程

首先用户`malloc`请求一个内存，先将请求的内存大小转换成`chunk`的大小，通过以下宏定义转换。

```
#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)

```

大概逻辑就是寻找一个最小能满足的`chunksize`作为`chunk`大小。

 

什么是最小能满足呢，我们看看一个`size=0x20`的`chunk`能有多少区域给用户写：`0x20`字节分别为`prev_size`，`size`，`fd`和`bk`，`prev_size`和`size`都不允许写，但是我们可以写`fd`和`bk`，以及下一个块的`prev_size`，前面我们也说过，当这个块没有被`free`的时候，它的`fd`,`bk`以及下一个`chunk`的`prevsize`位都是可以给用户任意写数据的，所以`size=0x20`，我们可以写的数据段为`0x18`。最小能满足就是说，当我请求的内存小于等于`0x18`的时候，我给你`size=0x20`的`chunk`，一旦多了就继续加`0x10`，也就是`2*SIZE_SZ`。这里用了其它宏定义去描述它我们尚且不管，如果用一个函数来实现它的话大概就是这样。

```
size_t request2size(size_t req){
    chunk_size=SIZE_SZ*4;
    while(chunk_size
```

所以在分配的时候我们尽量选择`0x18`,`0x28`这样刚刚好的数值，这样更容易发生溢出，哪怕溢出一个字节，也能够加以利用。

 

那么算出了它的`chunk_size`之后呢，我们先会判断这个`chunk_size`是否`<=global_max_fast`，也就是是否在`fast bin`范围内。如果在则优先寻找能匹配的`fast bin`，如果该`size`的`fast bin`为空则会寻找`small bin`，`small bin`会寻找特定`size`的`chunk`返回。如果`small bin`也为空，或者找不到能满足的那就会去`large bin`中寻找，同样是最小能满足，找到之后返回或者切割之后返回。还找不到就会去`unsorted bin`，`unsorted bin`则会找第一个能满足的`chunk`并返回或者切割之后返回，`unsorted bin` 中每遍历一个不满足要求的`unsorted bin`就会把该`unsorted bin`加到合适的 small bin 或者`large bin`当中。如果切割之后剩余的部分 <`MINSIZE`，那么则不会切割整个返回。

 

如果还是找不到，那么就会切割`top_chunk`。如果`top_chunk`都不能满足请求的大小，则会`free` `top_chunk`并再一次向操作系统申请新的`top_chunk`，这次申请同样还是申请一个`0x21000B`的`top_chunk`，通常情况下旧的`top_chunk`和新申请的`top_chunk`物理相邻，那么如果`free` 旧的`top_chunk`进入了一个非`fast bin`的链当中，就会被新的`top_chunk`合并。

 

如果一次申请的内存超过`0x200000B`，那么就不会在 heap 段上分配内存，将会使用`mmap`在`libc`的`data`段分配内存。通常利用就是每次分配给分配地址，分配`size`没限制那就`malloc`一个很大的内存就可以直接泄露`libc`的地址。

 

分配方式到此就讲完了。

malloc 源码分析
-----------

接下来我们直接解读一下`malloc`的源码。

### __libc_malloc 源码分析

```
/*------------------------ Public wrappers. --------------------------------*/
 
void *
__libc_malloc (size_t bytes)
{
  mstate ar_ptr;
  void *victim;
 
  void *(*hook) (size_t, const void *)
    = atomic_forced_read (__malloc_hook);
  if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));
 
  arena_get (ar_ptr, bytes);
 
  victim = _int_malloc (ar_ptr, bytes);
  /* Retry with another arena only if we were able to find a usable arena
     before.  */
  if (!victim && ar_ptr != NULL)
    {
      LIBC_PROBE (memory_malloc_retry, 1, bytes);
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }
 
  if (ar_ptr != NULL)
    (void) mutex_unlock (&ar_ptr->mutex);
 
  assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
          ar_ptr == arena_for_chunk (mem2chunk (victim)));
  return victim;
}

```

malloc 实际上会直接调用这里的`__libc_malloc`函数，然后`__libc_malloc`也只不过是一层包装而已，实际上大部分的逻辑都是调用`_int_malloc`函数完成的，那么先来分析外面。

 

第一段

```
void *(*hook) (size_t, const void *)
    = atomic_forced_read (__malloc_hook);
if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));

```

定义了一个`hook`函数指针，如果`hook!=NULL`则直接调用`hook`指向的内容。`hook`是为了方便开发者调试的一个东西，比如我自己写了一个`malloc`函数想测试它的性能如何，那么我在这里直接让`__malloc_hook=my_malloc`就可以直接调用我自己写的 malloc 函数了。但是同时它也是最容易被劫持的，刚开始我们很多题目都是靠写`__malloc_hook`为一个`onegadget`，然后调用`malloc`去`getshell`的。在`2.34`版本中，`__malloc_hook`同其它大部分的`hook`都被删除了。

 

第二段

```
arena_get (ar_ptr, bytes);
victim = _int_malloc (ar_ptr, bytes);

```

通过`arena_get`获得一个分配区，`arena_get`是个宏定义，定义在`arena.c`中，

```
#define arena_get(ptr, size) do { \
      arena_lookup (ptr);                           \
      arena_lock (ptr, size);                         \
  } while (0)

```

`arena_lookup`定义如下，也是获取分配器指针。

```
#define arena_lookup(ptr) do { \
      void *vptr = NULL;                              \
      ptr = (mstate) tsd_getspecific (arena_key, vptr);             \
  } while (0)

```

然后加锁，没了，获取分配器指针这一段不是我们主要要分析的，也就不过多去解析了。

 

第三段

```
/* Retry with another arena only if we were able to find a usable arena
     before.  */
  if (!victim && ar_ptr != NULL)
    {
      LIBC_PROBE (memory_malloc_retry, 1, bytes);
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }
  if (ar_ptr != NULL)
      (void) mutex_unlock (&ar_ptr->mutex);
  assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
          ar_ptr == arena_for_chunk (mem2chunk (victim)));
  return victim;

```

它本身注释也写清楚了，在能够找到一个可用的`arena`之前尝试寻找另外一个`arena`，我这英文比较飘还请亲见谅。如果`arena`找到了但是`_int_malloc`居然返回 0 了，那么就重新寻找另一个分配器再次调用一次`_int_malloc`。完了之后呢，要给`arena`解锁，然后返回得到的`chunk`指针。

### _int_malloc 源码分析

由于比较长，为了摆脱水长度的嫌疑就不给看总代码了，需要的自己找`glibc`的源码就好了，下面我一段一段分析。

### [](#第一段：main_arena初始化)第一段：main_arena 初始化

```
INTERNAL_SIZE_T nb;               /* normalized request size */
unsigned int idx;                 /* associated bin index */
mbinptr bin;                      /* associated bin */
 
mchunkptr victim;                 /* inspected/selected chunk */
INTERNAL_SIZE_T size;             /* its size */
int victim_index;                 /* its bin index */
 
mchunkptr remainder;              /* remainder from a split */
unsigned long remainder_size;     /* its size */
 
unsigned int block;               /* bit map traverser */
unsigned int bit;                 /* bit map traverser */
unsigned int map;                 /* current word of binmap */
 
mchunkptr fwd;                    /* misc temp for linking */
mchunkptr bck;                    /* misc temp for linking */
 
const char *errstr = NULL; 
checked_request2size (bytes, nb);
 
/* There are no usable arenas.  Fall back to sysmalloc to get a chunk from
     mmap.  */
if (__glibc_unlikely (av == NULL))
{
    void *p = sysmalloc (nb, av);
    if (p != NULL)
        alloc_perturb (p, bytes);
    return p;
}

```

变量定义就不用看了，源码中也都标注出来了，这里最主要就是把用户请求的`bytes`转换成最小能满足的`chunk size`，然后它的变量名应该是`nb`，这个`nb`应该是`nbytes`的缩写。

```
#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)
 
/*  Same, except also perform argument check */
 
#define checked_request2size(req, sz)                             \
  if (REQUEST_OUT_OF_RANGE (req)) {                          \
      __set_errno (ENOMEM);                              \
      return 0;                                      \
    }                                          \
  (sz) = request2size (req);

```

这里原来也给注释了，这俩宏定义就是一样的，只不过做一个参数 check。

 

这里还要注意一下那些宏定义：

```
__glibc_unlikely(exp)表示exp很可能为假。
__glibc_likely(exp)表示exp很可能为真。
__builtin_expect(exp,value)表示exp==value大概率成立

```

这三个宏定义在源码中经常能看到，其实它不会改编程序逻辑，只是告诉编译器这个很可能为某个值，就把否的情况作为跳转，真的情况就顺序运行下去，减少程序的跳转，一定程度上可以优化程序运行速度。或者还有一个简单粗暴的办法，你把这三个字符全都去了，不影响代码逻辑。

 

那么这一段的逻辑就是，如果在分配的时候`arena`为空，那就调用`sys_malloc`系统调用去请求一个`chunk`，然后`memset`这个`chunk`的数据段。

```
/* ------------------ Testing support ----------------------------------*/
 
static int perturb_byte;
static void alloc_perturb (char *p, size_t n)
{
  if (__glibc_unlikely (perturb_byte))
    memset (p, perturb_byte ^ 0xff, n);
}

```

通常情况下`perturb_byte`为假，差不多意思就是如果你没有特殊设置，那么`data`段全为 0 字节，实际情况也确实是这样的。

### 第二段：fast bin 的处理

```
#define get_max_fast() global_max_fast
#define fastbin_index(sz) \
  ((((unsigned int) (sz)) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)
#define fastbin(ar_ptr, idx) ((ar_ptr)->fastbinsY[idx])
 
if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
{
    idx = fastbin_index (nb);
    mfastbinptr *fb = &fastbin (av, idx);
    mchunkptr pp = *fb;
    do
    {
        victim = pp;
        if (victim == NULL)
            break;
    }
    while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim))
           != victim);
    if (victim != 0)
    {
        if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))//在malloc的时候检查了fastbin的size发现不对
        {
            errstr = "malloc(): memory corruption (fast)";
            errout:
            malloc_printerr (check_action, errstr, chunk2mem (victim), av);
            return NULL;
        }
        check_remalloced_chunk (av, victim, nb);
        void *p = chunk2mem (victim);
        alloc_perturb (p, bytes);
        return p;
    }
}

```

这里嘛就会判断，你申请的这个`nb`是否`<=global_max_fast`，如果成立那么就会先在`fast bin`中寻找能满足的`chunk`，并且一定是完全匹配。它先找到`av->fastbinY[idx]`观察是否为 0，如果不为 0 则说明该`size`的`fast bin`有`chunk`，那么就做以下动作：

 

取出`av->fastbinY[idx]`给`victim`

 

链表中删除这个`victim`，然后重新接回去。

 

中间有一个`check`，就是判断所给`chunk`的`fastbinY`链上的`size`是否＝我需要的`size`，如果不相等那么直接报错退出。

 

末尾也有一个`check`

```
/*
   Properties of chunks recycled from fastbins
 */
 
static void
do_check_remalloced_chunk (mstate av, mchunkptr p, INTERNAL_SIZE_T s)
{
  INTERNAL_SIZE_T sz = p->size & ~(PREV_INUSE | NON_MAIN_ARENA);
 
  if (!chunk_is_mmapped (p))
    {
      assert (av == arena_for_chunk (p));
      if (chunk_non_main_arena (p))
        assert (av != &main_arena);
      else
        assert (av == &main_arena);
    }
 
  do_check_inuse_chunk (av, p);
 
  /* Legal size ... */
  assert ((sz & MALLOC_ALIGN_MASK) == 0);
  assert ((unsigned long) (sz) >= MINSIZE);
  /* ... and alignment */
  assert (aligned_OK (chunk2mem (p)));
  /* chunk is less than MINSIZE more than request */
  assert ((long) (sz) - (long) (s) >= 0);
  assert ((long) (sz) - (long) (s + MINSIZE) < 0);
}

```

就是`check`各个标志位，一般不会被触发，所以可以理解为`fast bin`在分配的时候只有这一个`check`，就是那个`chunk`的`size`一定是等于我申请的`size`的，过了就把这个`chunk`的指针返回，`check`没过报错，如果根本都没取到`fast bin`，那么就进行下面的逻辑了。

### 第三段：small bin 的处理

```
#define NBINS             128
#define NSMALLBINS         64
#define SMALLBIN_WIDTH    MALLOC_ALIGNMENT
#define SMALLBIN_CORRECTION (MALLOC_ALIGNMENT > 2 * SIZE_SZ)
#define MIN_LARGE_SIZE    ((NSMALLBINS - SMALLBIN_CORRECTION) * SMALLBIN_WIDTH)
 
#define in_smallbin_range(sz)  \
  ((unsigned long) (sz) < (unsigned long) MIN_LARGE_SIZE)
 
#define smallbin_index(sz) \
  ((SMALLBIN_WIDTH == 16 ? (((unsigned) (sz)) >> 4) : (((unsigned) (sz)) >> 3))\
   + SMALLBIN_CORRECTION)
#define bin_at(m, i) \
  (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2]))                  \
             - offsetof (struct malloc_chunk, fd))
#define first(b)     ((b)->fd)
#define last(b)      ((b)->bk)
 
    /*
     If a small request, check regular bin.  Since these "smallbins"
     hold one size each, no searching within bins is necessary.
     (For a large request, we need to wait until unsorted chunks are
     processed to find best fit. But for small ones, fits are exact
     anyway, so we can check now, which is faster.)
   */
 
  if (in_smallbin_range (nb))
    {
      idx = smallbin_index (nb);
      bin = bin_at (av, idx);
 
      if ((victim = last (bin)) != bin)
        {
          if (victim == 0) /* initialization check */
            malloc_consolidate (av);
          else
            {
              bck = victim->bk;
    if (__glibc_unlikely (bck->fd != victim))
                {
                  errstr = "malloc(): smallbin double linked list corrupted";
                  goto errout;
                }
              set_inuse_bit_at_offset (victim, nb);
              bin->bk = bck;
              bck->fd = bin;
 
              if (av != &main_arena)
                victim->size |= NON_MAIN_ARENA;
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
        }
    }

```

首先判断它在不在`small bin`的范围内，然后取出这个`size`的`small bin`的最后一个`chunk`。它添加是在头部添加的，因此是符合先进先出的，嗯。然后需要判断，如果最后一个 chunk!= 自身的话，两个情况：要么没初始化`arena`，那就初始化，要么它有一个合法的块。如果它指向自身那就没必要做过多的判断了，没有这个大小的`small bin`。

 

这里是调用了`malloc_consolidate`函数去初始话这个`arena`分配器，该函数逻辑如下，不重点解读。

```
static void malloc_consolidate(mstate av)
{
  mfastbinptr*    fb;                 /* current fastbin being consolidated */
  mfastbinptr*    maxfb;              /* last fastbin (for loop control) */
  mchunkptr       p;                  /* current chunk being consolidated */
  mchunkptr       nextp;              /* next chunk to consolidate */
  mchunkptr       unsorted_bin;       /* bin header */
  mchunkptr       first_unsorted;     /* chunk to link to */
 
  /* These have same use as in free() */
  mchunkptr       nextchunk;
  INTERNAL_SIZE_T size;
  INTERNAL_SIZE_T nextsize;
  INTERNAL_SIZE_T prevsize;
  int             nextinuse;
  mchunkptr       bck;
  mchunkptr       fwd;
 
  /*
    If max_fast is 0, we know that av hasn't
    yet been initialized, in which case do so below
  */
 
  if (get_max_fast () != 0) {
    clear_fastchunks(av);
 
    unsorted_bin = unsorted_chunks(av);
 
    /*
      Remove each chunk from fast bin and consolidate it, placing it
      then in unsorted bin. Among other reasons for doing this,
      placing in unsorted bin avoids needing to calculate actual bins
      until malloc is sure that chunks aren't immediately going to be
      reused anyway.
    */mlined version of consolidation code in free() *
 
    maxfb = &fastbin (av, NFASTBINS - 1);
    fb = &fastbin (av, 0);
    do {
      p = atomic_exchange_acq (fb, 0);
      if (p != 0) {
    do {
      check_inuse_chunk(av, p);
      nextp = p->fd;
 
      /* Slightly streamlined version of consolidation code in free() */
      size = p->size & ~(PREV_INUSE|NON_MAIN_ARENA);
      nextchunk = chunk_at_offset(p, size);
      nextsize = chunksize(nextchunk);
 
      if (!prev_inuse(p)) {
        prevsize = p->prev_size;
        size += prevsize;
        p = chunk_at_offset(p, -((long) prevsize));
        unlink(av, p, bck, fwd);
      }
 
      if (nextchunk != av->top) {
        nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
 
        if (!nextinuse) {
          size += nextsize;
          unlink(av, nextchunk, bck, fwd);
        } else
          clear_inuse_bit_at_offset(nextchunk, 0);
 
        first_unsorted = unsorted_bin->fd;
        unsorted_bin->fd = p;
        first_unsorted->bk = p;
 
        if (!in_smallbin_range (size)) {
          p->fd_nextsize = NULL;
          p->bk_nextsize = NULL;
        }
 
        set_head(p, size | PREV_INUSE);
        p->bk = unsorted_bin;
        p->fd = first_unsorted;
        set_foot(p, size);
      }
 
      else {
        size += nextsize;
        set_head(p, size | PREV_INUSE);
        av->top = p;
      }
 
    } while ( (p = nextp) != 0);
 
      }
    } while (fb++ != maxfb);
  }
  else {
    malloc_init_state(av);
    check_malloc_state(av);
  }
}

```

大致意思就是清空所有`arena`的`chunk`，可以看到大的`if`是判断`global_max_fast`是否为 0，为 0 则初始化，调用`malloc_init_state`和`check_malloc_state`函数初始化堆。否则把所有的`fast bin` 取出来，先清除它们的标志位，然后扔到`unsorted bin`中尝试向前合并或者向后合并。

 

这个呢，不太能运行到，因为`victim==0`的时候，必还没初始化，没初始化到这里就要初始化，初始化了之后`victim`又不可能`=0`了，所以这里可以理解为就是初始化`arena`的。

```
#define set_inuse_bit_at_offset(p, s)                          \
  (((mchunkptr) (((char *) (p)) + (s)))->size |= PREV_INUSE)
#define chunk2mem(p)   ((void*)((char*)(p) + 2*SIZE_SZ))
bck = victim->bk;
if (__glibc_unlikely (bck->fd != victim))
{
    errstr = "malloc(): smallbin double linked list corrupted";
    goto errout;
}
set_inuse_bit_at_offset (victim, nb);
bin->bk = bck;
bck->fd = bin;
 
if (av != &main_arena)
victim->size |= NON_MAIN_ARENA;
check_malloced_chunk (av, victim, nb);
void *p = chunk2mem (victim);
alloc_perturb (p, bytes);
return p;

```

这里判断了一下`victim->bk->fd==victim`。也就是当前这个堆块后一个堆块的`fd`指针是否指向`victim`，如果不等说明链表被破坏了，那么就报错退出。

 

然后`set_inuse_bit_at_offset`，这个也不难理解，因为现在这个`small bin`被取出来了要使用了，所以我得设置后一个块的`prev_inuse`为 1 证明它不是空闲堆块了。然后就是进行`unlink`操作，对链表熟悉的同学应该看得懂。如果我要删除`victim`元素那应该怎么写逻辑？

```
victim->fd->bk=victim->bk;
victime->bk->fd=victim->fd;

```

在这里呢，我们取链的最后一个`chunk`，也就是`bin->bk=victim`所以`victim->fd=bin`

 

然后前面有一个赋值就是`bck=victim->bk`。带进上面的式子就得到了源码里面这样的写法。

 

然后下面设置`main_arena`标志位，一波同样的`check`，然后返回内存指针。也就是这里的`chunk2mem`，我们这里用的`chunk`指针，但是其实我们要返回的应该是`chunk`中数据域的指针，所以这里用了这样的宏定义做替换。

 

然后就是清除`data`数据，但是这个一般不会被执行，前面也分析过了，然后返回。这是`small bin`找到对应的`chunk`的逻辑，如果`small bin`还没找到那么接下来应该要去找`large bin`了，那么我们接着往下读。

### [](#第四段：分配largebin时的操作)第四段：分配 largebin 时的操作

那么如果没有在 small bin 的范围内呢。

```
else
{
    idx = largebin_index (nb);
    if (have_fastchunks (av))
        malloc_consolidate (av);
}

```

这一步比较耐人寻味。

 

先获取`large bin`的`index`，然后如果`fast bin`不为空，调用`malloc_consolidate`。这一步是什么意思呢？我们前面分析过`malloc_consolidate`，如果没有初始化，那么初始化，如果初始化了，那么合并所有的`fast bin`。但是这里，都已经有`fast bin`存在了，那么堆指定已经初始化了，所以这里执行的逻辑基本只能是合并所有`fast chunk`。为什么要在搜索`large bin`的时候合并所有`fast bin`呢？因为`large bin`的匹配方式是最小能满足，然后切割。

 

考虑这样一种情况：

 

如果一个`0x20`的`fast bin`和 0x500 的`large bin`物理相邻。此时我要申请一个`0x510`的`large bin`，如果此时`fast bin`被合并了，那么我就能找到一个`0x520`的`large bin`并把它返回给用户。如果我不做这一步，那么我找不到`0x510`大小的`large bin`，我就被迫只能切割`top_chunk`了，这样子就浪费了很大的一块内存。

 

那么这个会不会有多此一举的时候呢，也是会的，还是刚刚那种情况，假如我申请`0x500`的`chunk`。这样子合并之后又会被切割，那么这样子，之前的合并就显得多次一举了，但是它只是浪费了一部分时间开销，内存分配上还是执行上面的逻辑比较占优势。所以这一步可以理解为空间上的优化，但是牺牲了小部分时间。看不来的话可以多看看上面举得例子。

### 第五段：large bin 和 unsorted bin 的相爱相杀

这里开始逻辑都混合起来了，不仅有`large bin`，unsorted bin，切割`top_chunk`，还有系统调用重新分配`top_chunk`。

#### 第 1 小块

```
for (;; )
    {
      int iters = 0;
      while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks (av))
        {
          bck = victim->bk;
          if (__builtin_expect (victim->size <= 2 * SIZE_SZ, 0)
              || __builtin_expect (victim->size > av->system_mem, 0))
            malloc_printerr (check_action, "malloc(): memory corruption",
                             chunk2mem (victim), av);
          size = chunksize (victim);
 
          /*
             If a small request, try to use last remainder if it is the
             only chunk in unsorted bin.  This helps promote locality for
             runs of consecutive small requests. This is the only
             exception to best-fit, and applies only when there is
             no exact fit for a small chunk.
           */
           ......
         }
    ......
    }

```

首先外面套了一个`while(1)`，然后里面有一个`while`循环，判断内容就是取得最后一个`unsorted chunk`是否与这个`bin`相等，这里大概就是开始遍历`unsorted chunk`了。

 

然后这里又有一个`check`。`victim->size <= 2 * SIZE_SZ`就是说`chunk`的`size`小于等于`0x10`，`victim->size > av->system_mem`就是说我一个块的`size`居然比我系统调用申请来的内存都多，那这肯定不合理啊，所以任意满足一个就会报错了。

#### 第二小块

```
if (in_smallbin_range (nb) &&
bck == unsorted_chunks (av) &&
victim == av->last_remainder &&
(unsigned long) (size) > (unsigned long) (nb + MINSIZE))
{
    /* split and reattach remainder */
    remainder_size = size - nb;
    remainder = chunk_at_offset (victim, nb);
    unsorted_chunks (av)->bk = unsorted_chunks (av)->fd = remainder;
    av->last_remainder = remainder;
    remainder->bk = remainder->fd = unsorted_chunks (av);
    if (!in_smallbin_range (remainder_size))
    {
        remainder->fd_nextsize = NULL;
        remainder->bk_nextsize = NULL;
    }
 
    set_head (victim, nb | PREV_INUSE |
    (av != &main_arena ? NON_MAIN_ARENA : 0));
    set_head (remainder, remainder_size | PREV_INUSE);
    set_foot (remainder, remainder_size);
 
    check_malloced_chunk (av, victim, nb);
    void *p = chunk2mem (victim);
    alloc_perturb (p, bytes);
    return p;
}

```

这里有四个条件：

1.  `in_smallbin_range (nb)`：申请`small bin`范围内的`chunk`
2.  `bck == unsorted_chunks (av)`：`bck=victim->bk=unsorted_chunks(av)->bk->bk`，也就是说`unsorted_chunks (av)->bk->bk=unsorted_chunks (av)`，翻译一下差不多就是`unsorted bin中`只有一个`chunk`。
3.  `victim == av->last_remainder`：就是说这个 chunk 刚好是最近被分割过的剩余部分。
4.  `(unsigned long) (size) > (unsigned long) (nb + MINSIZE))`：保证我找到的这个`chunksize` > 需要的最小块 +`MINSIZE`。为了保证等会我切割出`nb size`之后剩余的`chunk`能 >`MINSIZE`，这里我也不知道为什么不能等于，可能解读哪里有误吧，如果您知道请帮我勘误一下，谢谢了。

然后接下来就是切割`victim`，切割出一块刚刚好大小的`chunk`给用户，切割出来的`chunk`作为新的`av->last_remainder`，注意如果切割剩余的`chunk size`不符合`small bin`的大小，则`fd_nextsize`和`bk_nextisze`会被清空，因为剩余的的`chunk`会被放到`unsorted bin`当中。

 

然后设置`victim`的`size`为`nb|PREV_INUSE`，然后判断是否为主分配加上标记。

 

然后把 remainder 的`prev_inuse`位设置为 1，因为前一个块已经被拿走使用了，所以这个`prev_inuse`要设置为 1。

 

然后因为`remainder`的`size`发生了改表，所以下一个`chunk`的`prev_size`也要相应地改变。

 

剩下的前面类似的都讲过就不赘述了。

#### 第三小块

```
/* remove from unsorted list */
unsorted_chunks (av)->bk = bck;
bck->fd = unsorted_chunks (av);
 
/* Take now instead of binning if exact fit */
 
if (size == nb)
{
    set_inuse_bit_at_offset (victim, size);
    if (av != &main_arena)
        victim->size |= NON_MAIN_ARENA;
    check_malloced_chunk (av, victim, nb);
    void *p = chunk2mem (victim);
    alloc_perturb (p, bytes);
    return p;
}
 
/* place chunk in bin */
 
if (in_smallbin_range (size))
{
    victim_index = smallbin_index (size);
    bck = bin_at (av, victim_index);
    fwd = bck->fd;
}

```

否则会先取出这最后一个`chunk`，把它移除`unsorted bin`。如果取出的这个`size`刚好等于这个`nb`，那就说明这个块一定是最合适的，直接把它返回了，不要迟疑。如果并不是最合适呢，那么先会判断一下它是否属于`small bin`，属于则执行以下的逻辑，把`bck`对应`bin`的`bk`，`fwd`为对应`bin`的`fd`，也就是找到那一对`bin`，`fwd`在前，`bck`在后。就没了，预计等会就要用这些指针把`chunk`链进去了。

#### 第四小块

```
else
{
    victim_index = largebin_index (size);
    bck = bin_at (av, victim_index);
    fwd = bck->fd;
 
    /* maintain large bins in sorted order */
    if (fwd != bck)
    {
        /* Or with inuse bit to speed comparisons */
        size |= PREV_INUSE;
        /* if smaller than smallest, bypass loop below */
        assert ((bck->bk->size & NON_MAIN_ARENA) == 0);
        if ((unsigned long) (size) < (unsigned long) (bck->bk->size))
        {
            fwd = bck;
            bck = bck->bk;
 
            victim->fd_nextsize = fwd->fd;
            victim->bk_nextsize = fwd->fd->bk_nextsize;
            fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
        }
        else
        {
            assert ((fwd->size & NON_MAIN_ARENA) == 0);
            while ((unsigned long) size < fwd->size)
            {
                fwd = fwd->fd_nextsize;
                assert ((fwd->size & NON_MAIN_ARENA) == 0);
            }
 
            if ((unsigned long) size == (unsigned long) fwd->size)
                /* Always insert in the second position.  */
                fwd = fwd->fd;
            else
            {
                victim->fd_nextsize = fwd;
                victim->bk_nextsize = fwd->bk_nextsize;
                fwd->bk_nextsize = victim;
                victim->bk_nextsize->fd_nextsize = victim;
            }
            bck = fwd->bk;
        }
    }
    else
        victim->fd_nextsize = victim->bk_nextsize = victim;
}

```

如果不是`small bin`，那就得进`large bin`了，要进`large bin`。这里要知道，`large bin`可是所有`bin`当中最复杂的`bin`了，一个`chunk`四个指针，一对`bin`管理一个二维双向链表，`fd`,`bk`指针与相同大小的`chunk`连接，`fd_nextsize`和`bk_nextsize`与不同大小的`chunk`连接。

 

然后呢，虽然`fd`和`bk`是连接相同大小的`chunk`，但是那一对 bin 还是相当于是`fd`和`bk`字段。除了表头以外，其余的不同大小的 chunk 都是靠`fd_nextsize`和`bk_nextsize`的。并且沿着`bk_nextsize`，`chunksize`递增。也就是说`av->bin[index]->bk`是第一个`chunk`，并且`size` 最小，然后通过`bk_nextsize`字段一直连接到`av->bin[index]->fd`，反向同理。还有一点需要注意：`large bin`所在的`chunk`并不与`chunk`双向连接。

 

这里给出一张`large bin`的结构图，看看能不能帮助理解一下

 

![](https://bbs.pediy.com/upload/attach/202201/919002_M5XRCJBV3UQURFS.jpg)

 

那么这里的`bck`指的是`bin`所在的`chunk`，`fwd`指的是最大的这个`chunk`。

 

`bck->bk`指的就是图上的 n 号`chunk`，也是这个`large bin`中最小的那个`chunk`，如果拿出来的`unsorted bin`它比最小的`chunk`还要小，那就已经可以确定插入在哪了，就不用做下面的循环再看看它在哪了。然后就是一个链表的插入操作，这里要注意的是，`bin`所在的`chunk`只有`fd`和 bk 指针，而其它`chunk`都是`fd_nextsize`和`bk_nextsize`连接的。我们只需要先在最大块和最小块之间插入，然后把`bin->bk`指向`victim`即可。

 

那么我们大概自己写一下操作看看与源码是否一致。首先不考虑 bin，只考虑链表的情况下，我们先找到最大块和最小块。

```
fwd=bin->fd;
bck=bin->bk;
victim->bk_nextsize=bck;
victim->fd_nextsize=fwd;
fwd->bk_nextsize=bck->fd_nextsize=victim;

```

跟上面大致一样，只不过它这里`fwd`的值是那个`large bin`的`chunk`，直接通过`fd`指针也能直接找到最大的`chunk`。所以我后面的主要代码应该把`fwd`改成`fwd->fd`就跟上面一模一样了。

 

如果不是，那就接着往`bk_nextsize`这个指针上面找，找到大于等于的`chunk`为止。然后如果等于，就只需要用`fd`和`bk`指针与相等大小的`chunk`相连，如果没有相等，就得在`fd_nextsize`和`bk_nextsize`方向上插入，然后`fd`和`bk`都默认指向自己。这个我就不演试了，跟前面那个基本是一样的。

#### 第五小块

```
mark_bin (av, victim_index);
victim->bk = bck;
victim->fd = fwd;
fwd->bk = victim;
bck->fd = victim;
#define MAX_ITERS       10000
if (++iters >= MAX_ITERS)
    break;

```

这个就很简单了，就是一个插入操作，前面既然已经找到了插入的位置，这里一气呵成直接解决了。然后这里还有一个遍历`unsorted bin`的最大值，一次最多遍历`10000`个`unsorted bin`，这个也可以理解，如果我一次产生了很多的`unsorted bin`，然后我一次`malloc`，那边一直在循环搞这个`unsorted bin`，迟迟就没分配内存回来所以这里设定一个最大值。

 

到了这里，对`unsorted bin`的遍历就结束了。

#### 第六小块

```
if (!in_smallbin_range (nb))
{
    bin = bin_at (av, idx);
 
    /* skip scan if empty or largest chunk is too small */
    if ((victim = first (bin)) != bin &&
        (unsigned long) (victim->size) >= (unsigned long) (nb))
    {
        victim = victim->bk_nextsize;
        while (((unsigned long) (size = chunksize (victim)) <
                (unsigned long) (nb)))
            victim = victim->bk_nextsize;
 
        /* Avoid removing the first entry for a size so that the skip
                 list does not have to be rerouted.  */
        if (victim != last (bin) && victim->size == victim->fd->size)
            victim = victim->fd;
 
        remainder_size = size - nb;
        unlink (av, victim, bck, fwd);
 
        /* Exhaust */
        if (remainder_size < MINSIZE)
        {
            set_inuse_bit_at_offset (victim, size);
            if (av != &main_arena)
                victim->size |= NON_MAIN_ARENA;
        }
        /* Split */
        else
        {
            remainder = chunk_at_offset (victim, nb);
            /* We cannot assume the unsorted list is empty and therefore
                     have to perform a complete insert here.  */
            bck = unsorted_chunks (av);
            fwd = bck->fd;
            if (__glibc_unlikely (fwd->bk != bck))
            {
                errstr = "malloc(): cor
                    rupted unsorted chunks";
                goto errout;
            }
            remainder->bk = bck;
            remainder->fd = fwd;
            bck->fd = remainder;
            fwd->bk = remainder;
            if (!in_smallbin_range (remainder_size))
            {
                remainder->fd_nextsize = NULL;
                remainder->bk_nextsize = NULL;
            }
            set_head (victim, nb | PREV_INUSE |
                      (av != &main_arena ? NON_MAIN_ARENA : 0));
            set_head (remainder, remainder_size | PREV_INUSE);
            set_foot (remainder, remainder_size);
        }
        check_malloced_chunk (av, victim, nb);
        void *p = chunk2mem (victim);
        alloc_perturb (p, bytes);
        return p;
    }
}

```

这边就看这个最小能满足的`nb`是否在`small bin`的范围内。不在则执行，其实如果在的话，那前面有一个`small bin`的范围判断，如果`small bin`范围那，`idx`就是`small bin`，不在则是`large bin`的`idx`。`small bin`之前已经判断过一遍了，并且判断策略也跟之前不一样，所以这里加一个`!in_small_bin_range`的判断还是很有必要的。

 

来看下面的 if 语句，两个条件。

1.  `(victim = first (bin)) != bin`：这个 bin 里面有`chunk`，并使`victim=bin->fd`
2.  `(unsigned long) (victim->size) >= (unsigned long) (nb)`：找到目标 chunk 的 size 要大于等于这个最小能满足的 size nb。

同时满足那么就可能要取这一块 chunk 来分配了，正如注释所说，如果 bin 为空或者最大的 chunk 还是比较小，那就跳过这个逻辑。然后`victim = victim->bk_nextsize`，这里`victim`是最大块，最大块的`bk_nextsize`就是最小块，这里应该也是尽量寻找最小能满足的块。正如循环所描述，如果`victim`的`chunk size`比我所需的最小能满足的`chunk size` `nb`还小，那就去寻找比他大的，因为是递增，所以能保证在`chunk`当中我一定会找到一个最小能满足的`chunk`。

 

这里解释一下两个最小能满足的意思：

 

首先`nb`是指用户需要的最小能满足的块的`size`，比如我只需要 1 个字节，但是我最小的`chunk size`都是`0x20`了，`0x20`的`chunk`就是对用户最小能满足的`chunk size`了。

 

如果能找到`size=nb`的块，当然是最好不过了，但是现实往往不会那么顺利，比如我只有一个`0x30`的块，如果我只有`0x30`而没有`0x20`的块，那么`0x30`就是我所有`free`块当中的最小能满足，其实这里`nb`应该叫最优能满足，但是我还是习惯这么叫了 hhh。

 

然后呢找到这个之后就`unlink`这个块，把它从链中删除，拿出来之后进行一个判断，如果切割之后的块小于 MINSIZE，那就不切割了，直接把它物理相邻的下一个快`prev_inuse`位设 1，这个块就直接返回给用户了。否则就是切割，设置各种东西，这个前面有差不多的代码，我们主要看看剩下的块去哪里了，很明显，重新链入`unsorted bin`了。后面有一个`check`，如果`unsorted bin->fd->bk!=unsorted bin`，那么报错退出。这里需要注意，它只检测了`unsorted bin->fd->bk`是否等于那个`unsorted bin`，对于堆块来说我就是只检测了`bk`指针，这意味着`fd`指针如果修改为任意值不会在这里被检测到，这是一个利用小技巧，也只有你读过源码后才能好好理解这个`unsorted bin attack`了。然后如果剩余大小不在`small bin`范围内把`nextsize`指针全部清空，其它就是正常返回了。如果被切割的剩下`chunk`不在`small bin`范围内，就会清空它的`fd_nextsize`和`bk_nextsize`。因为它要回到`unsorted bin`里面，这两个字段就没什么用了，就会被清空。

#### 第七小块

```
++idx;
bin = bin_at (av, idx);
block = idx2block (idx);
map = av->binmap[block];
bit = idx2bit (idx);

```

我们来讲一讲`arena`的`binmap`结构，这个用于快速检索一个`bin`是否为空，每一个`bit`表示对应的`bin`中是否存在空闲`chunk`，虽然不知道为什么前面没有用到。这一段就是说，如果`large bin`搜索完了都没有找到合适的`chunk`，那么就去下一个`idx`里面寻找，这很合理。然后一共有 4 个 int，每个`int`32 位表示一块`map`，一共表示`128`位。

#### 第八小块

```
for(;;)
{
    /* Skip rest of block if there are no more set bits in this block.  */
    if (bit > map || bit == 0)
    {
        do
        {
            if (++block >= BINMAPSIZE) /* out of bins */
                goto use_top;
        }
        while ((map = av->binmap[block]) == 0);
 
        bin = bin_at (av, (block << BINMAPSHIFT));
        bit = 1;
    }
    ........
}

```

我们来看看两个条件

1.  `bit>map`：如果这个位的权值都比它整个的`map`都大了，说明`map`上那个`bit`的权值必定为 0
2.  `bit==0`：如果这个`bit`都是 0 说明这个`index`也不对。

满足其一就看看别的`index`。

 

然后如果说`map==0`，说明这整个`block`都没有空闲块，就直接跳过，不为 0 则退出去执行下面的操作，如果超过了`block`的总数，那就说明`unsorted bin`和`large bin`中也没有合适的`chunk`，那我们就切割`top_chunk`了，这里用了一个`goto`跳转，我们后面分析。

#### 第九小块

```
/* Advance to bin with set bit. There must be one. */
while ((bit & map) == 0)
{
    bin = next_bin (bin);
    bit <<= 1;
    assert (bit != 0);
}
 
/* Inspect the bin. It is likely to be non-empty */
victim = last (bin);
 
/*  If a false alarm (empty bin), clear the bit. */
if (victim == bin)
{
    av->binmap[block] = map &= ~bit; /* Write through */
    bin = next_bin (bin);
    bit <<= 1;
}

```

此时我已经找到了一个合适的`block`，然后就是看`block`的各个位了。从低位开始，如果检查到`map`那一位对应为 0 就找下一位，我们前面提到 bk 为`large bin`的最小块，所以先考虑它，当然不能说`map`里面说这里有它就有，我还得自己判断一下这个`bin`里面是不是真的有，如果没有 (`bin->bk==bin`)，那么我就要及时把标志位清除然后`bit<<1`去寻找下一个`index`。

#### 最后一小块

```
else
{
    size = chunksize (victim);
 
    /*  We know the first chunk in this bin is big enough to use. */
    assert ((unsigned long) (size) >= (unsigned long) (nb));
 
    remainder_size = size - nb;
 
    /* unlink */
    unlink (av, victim, bck, fwd);
 
    /* Exhaust */
    if (remainder_size < MINSIZE)
    {
        set_inuse_bit_at_offset (victim, size);
        if (av != &main_arena)
        victim->size |= NON_MAIN_ARENA;
    }
 
    /* Split */
    else
    {
        remainder = chunk_at_offset (victim, nb);
 
        /* We cannot assume the unsorted list is empty and therefore
        have to perform a complete insert here.  */
        bck = unsorted_chunks (av);
        fwd = bck->fd;
        if (__glibc_unlikely (fwd->bk != bck))
        {
            errstr = "malloc(): corrupted unsorted chunks 2";
            goto errout;
        }
        remainder->bk = bck;
        remainder->fd = fwd;
        bck->fd = remainder;
        fwd->bk = remainder;
 
        /* advertise as last remainder */
        if (in_smallbin_range (nb))
        av->last_remainder = remainder;
        if (!in_smallbin_range (remainder_size))
        {
            remainder->fd_nextsize = NULL;
            remainder->bk_nextsize = NULL;
        }
        set_head (victim, nb | PREV_INUSE |
        (av != &main_arena ? NON_MAIN_ARENA : 0));
        set_head (remainder, remainder_size | PREV_INUSE);
        set_foot (remainder, remainder_size);
    }
    check_malloced_chunk (av, victim, nb);
    void *p = chunk2mem (victim);
    alloc_perturb (p, bytes);
    return p;
}

```

如果它确实有`chunk`呢？然后其实它还是跟前面一样的，在`large bin`中找到`chunk`的处理方式，`unlink`，切割，判断，设置标志位，切割后及时更新`last_remainder`，这里就是一个`large bin`的遍历。

 

还要讲一下的就是这个`check`，依旧是对`unsorted bin`的一个`check`，判断第一个`unsorted chunk`的`bk`指针是否指向`unsorted bin`的位置。这里需要把割剩下的`chunk`重新放回`unsorted bin`。至此整个`unsorted bin`和`large bin`的分配就讲完了。

### [](#第七段：切割top_chunk)第七段：切割 top_chunk

```
use_top:
    /*
             If large enough, split off the chunk bordering the end of memory
             (held in av->top). Note that this is in accord with the best-fit
             search rule.  In effect, av->top is treated as larger (and thus
             less well fitting) than any other available chunk since it can
             be extended to be as large as necessary (up to system
             limitations).
 
             We require that av->top always exists (i.e., has size >=
             MINSIZE) after initialization, so if it would otherwise be
             exhausted by current request, it is replenished. (The main
             reason for ensuring it exists is that we may need MINSIZE space
             to put in fenceposts in sysmalloc.)
           */
 
    victim = av->top;
    size = chunksize (victim);
 
    if ((unsigned long) (size) >= (unsigned long) (nb + MINSIZE))
    {
        remainder_size = size - nb;
        remainder = chunk_at_offset (victim, nb);
        av->top = remainder;
        set_head (victim, nb | PREV_INUSE |
                  (av != &main_arena ? NON_MAIN_ARENA : 0));
        set_head (remainder, remainder_size | PREV_INUSE);
 
        check_malloced_chunk (av, victim, nb);
        void *p = chunk2mem (victim);
        alloc_perturb (p, bytes);
        return p;
    }
 
    /* When we are using atomic ops to free fast chunks we can get
             here for all block sizes.  */
    else if (have_fastchunks (av))
    {
 
        else if (have_fastchunks (av))
        {
            malloc_consolidate (av);
            /* restore original bin index */
            if (in_smallbin_range (nb))
                idx = smallbin_index (nb);
            else
                idx = largebin_index (nb);
        }
 
        /*
                 Otherwise, relay to handle system-dependent cases
               */
        else
        {
            void *p = sysmalloc (nb, av);
            if (p != NULL)
                alloc_perturb (p, bytes);
            return p;
        }
    }
}

```

这一步比较简单，就是说先从`av->top`拿到`top_chunk`的地址。判断大小尝试切割，如果不能切割，它也不会尽量去麻烦操作系统，先调用`malloc_consolidate`去合并所有的`fast bin`里面的`chunk`。然后合并之后接着步入之前的循环，重新找一次`small bin` `large bin` `unsorted bin`，因为现在可能已经有合适的`chunk`了对吧。

 

然后如果还是没有合适的呢？就会进入这里的`else`，调用`sysmalloc`去分配内存，一次还是分配`0x21000`的`chunk`作为新的`top_chunk`，原来的`top_chunk`将会被`free`，一般来说如果你没有改过`top_chunk`的`size`，那么新的和旧的`top_chunk`将会是物理相邻，如果`free` 的`top_chunk`不在`fast bin`范围内，那就会和新的`top_chunk`发生合并。那么这一整个`malloc`源码就解读完了，我们来做一下总结。

### 总结

1.  检查是否设置了`malloc_hook`，若设置了则跳转进入`malloc_hook`，若未设置则获取当前的分配区，进入`int_malloc`函数。
    
2.  如果当前的分配区为空，则调用`sysmalloc`分配空间，返回指向新`chunk`的指针，否则进入下一步。
    
3.  若用户申请的大小在`fast bin`的范围内，则考虑寻找对应`size`的`fast bin chunk`，判断该`size`的`fast bin`是否为空，不为空则取出第一个`chunk`返回，否则进入下一步。
    
4.  如果用户申请的大小符合`small bin`的范围，则在相应大小的链表中寻找`chunk`，若`small bin`未初始化，则调用`malloc_consolidate`初始化分配器，然后继续下面的步骤，否则寻找对应的`small bin`的链表，如果该`size` 的`small bin`不为空则取出返回，否则继续下面的步骤。如果申请的不在`small bin`的范围那么调用`malloc_consolidate`去合并所有`fast bin`并继续下面的步骤。
    
5.  用户申请的大小符合`large bin`或`small bin`链表为空，开始处理`unsorted bin`链表中的`chunk`。在`unsorted bin`链表中查找符合大小的`chunk`，若用户申请的大小为`small bin`，`unsorted bin`中只有一块 chunk 并指向`last_remainder`，且`chunk size`的大小大于`size+MINSIZE`，则对当前的`chunk`进行分割，更新分配器中的`last_remainder`，切出的`chunk`返回给用户，剩余的`chunk`回`unsorted bin`。否则进入下一步。
    
6.  将当前的`unsorted bin`中的`chunk`取下，若其`size`恰好为用户申请的`size`，则将`chunk`返回给用户。否则进入下一步
    
7.  获取当前`chunk size`所对应的 bins 数组中的头指针。（`large bin`需要保证从大到小的顺序，因此需要遍历）将其插入到对应的链表中。如果处理的 chunk 的数量大于`MAX_ITERS`则不在处理。进入下一步。
    
8.  如果用户申请的空间的大小符合`large bin`的范围或者对应的 small bin 链表为空且`unsorted bin`链表中没有符合大小的`chunk`，则在对应的`large bin`链表中查找符合条件的`chunk`（即其大小要大于用户申请的`size`）。若找到相应的`chunk`则对`chunk`进行拆分，返回符合要求的`chunk`（无法拆分时整块返回）。否则进入下一步。
    
9.  根据`binmap`找到表示更大`size`的`large bin`链表，若其中存在空闲的`chunk`，则将`chunk`拆分之后返回符合要求的部分，并更新`last_remainder`。否则进入下一步。
    
10.  若`top_chunk`的大小大于用户申请的空间的大小，则将`top_chunk`拆分，返回符合用户要求的`chunk`，并更新`last_remainder`，否则进入下一步。
    
11.  若`fast bin`不为空，则调用`malloc_consolidate`合并`fast bin`，重新回到第四步再次从`small bin`搜索。否则进入下一步。
    
12.  调用`sysmalloc`分配空间，`free top chunk`返回指向新`chunk`的指针。
    
13.  若`_int_malloc`函数返回的`chunk`指针为空，且当前分配区指针不为空，则再次尝试`_int_malloc`
    
14.  对`chunk`指针进行检查，主要检查`chunk`是否为`mmap`，且位于当前的分配区内。
    
    free 源码分析
    ---------
    

free 的源码分析先咕一会，主要是吧，熬夜写这玩意受不了。。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 22 小时前 被 xi@0ji233 编辑 ，原因： 格式问题

[#专题](forum-171-1-186.htm)