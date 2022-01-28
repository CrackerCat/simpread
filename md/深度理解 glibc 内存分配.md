> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271331.htm)

> 深度理解 glibc 内存分配

  如果对 glibc 内存管理的逻辑比较了解，在学习各种堆漏洞利用的方法时，就会更顺畅一些，而且 glibc 内存管理的代码，本身也很值得看一看，好几次决定写点总结，都是咬咬牙又放弃了，现在，我觉得已经不能再拖了。

1. 什么是分配 / 释放？
--------------

  在刚入门 C 语言编程时就知道：为了使用一块堆内存，要先分配。随着接触编程一段时间后，慢慢开始意识到：分配，就是底层代码把正在使用的内存区域记录下来，从而保证程序始终知道哪些空间是正在使用的，哪些空间当前是空闲的，进一步保证每次获取的内存，不会与正在使用的内存重叠。

*   **分配释放初级理解**  
    ![](https://bbs.pediy.com/upload/tmp/815036_APXUSQW6VQEAHAK.png)
*   **分配释放的本质**
    
    ```
    // 32位系统
    *(char *)0x9d08008 = 0;
    
    ```
    
    执行：
    
    ```
    jmpcall@ubuntu:test$ ./a.out
    Segmentation fault
    
    ```
    
    进程刚启动时，空闲空间是很大的，很容易自己找到还未使用内存的地址，比如 0x9d08008(<0xc0000000，所以为用户空间的地址，用户态无法访问内核空间稍后另外说明)，但是执行时，却段错误了。  
    这个段错误是怎么来的呢？  
    猜想：系统给的。这样猜想就太笼统了，“系统”是什么，怎么 “给的”，都没有具体的信息。“系统” 如果是底层的代码逻辑 (glibc 或者内核)，那如果底层代码本身执行这样的指令，就是允许的吗，还是说“系统” 是个什么别的东西？  
    反汇编一下：
    
    ```
    int main()
    {
     80483c3:    55                       push   %ebp
     80483c4:    89 e5                    mov    %esp,%ebp
            *(char *)0x9d08008 = 0;
     80483c6:    b8 08 80 d0 09           mov    $0x9d08008,%eax
     80483cb:    c6 00 00                 movb   $0x0,(%eax)
            return 0;
     80483ce:    b8 00 00 00 00           mov    $0x0,%eax
    }
    
    ```
    
    需要明白一点的是，即使以上是应用程序的代码，它的每一条指令，也是直接由 CPU 执行的，所以 0x9d08008 这个地址无法访问，是 CPU 在硬件层判断的，并不是 glibc 或者内核在软件层进行判断的。就是说，分配归根结底是为了让 CPU 硬件层 “承认” 哪些内存空间可用，换句话说，底层代码对已使用空间的记录，其实是给硬件层使用的(所以记录在哪，怎么记录，就必须按照 CPU 手册中的规范来做)。这样，以上代码绕开分配函数，直接使用 0x9d08008 这个地址，为什么产生段错误，就很好理解了：即使 0x9d08008 始终不与已使用的内存空间冲突，也会由于没记录到已分配空间，而不被 CPU 硬件层“承认”。所以，**内存分配的本质，是按照硬件规范，对已使用空间进行记录**。
*   **分配 / 释放机制建立过程**  
    刚开机执行的第一条指令，就可以调用分配接口吗？  
    很多硬件的功能都是可以切换的，比如沙发，也可以铺开变成一张床，而 CPU 同样身为一种硬件，它的功能和状态也是可以调节的 (通过给一些寄存器的特定位加电)。比如，它可以切换为实模式或保护模式，保护模式下又可以调节到不同的权限级别(低权限状态下，相当于将 CPU 的部分功能“锁” 起来了，执行不了特权指令)。CPU 刚加电时，正是默认处于实模式(这是硬件设计保证的)，内核启动的初级阶段，自然就是在 CPU 实模式下执行，指令中的地址，会被 CPU 直接加到地址总线上，也就是说，这个阶段，使用内存是不需要经过分配这个步骤的，甚至有些是人为安排的固定地址。  
    内核在实模式运行阶段，会为进入保护模块做好准备 (如同软件开发中“初始化” 的概念)，因为切换到保护模式的瞬间，CPU 马上就会按照另外一套硬件逻辑，对软件的指令进行处理，这时再执行访问内存的指令，就不再像实模式那样直截了当了，而是必须先根据 CR3 寄存器，找到目录表、页表，查找跟指令包含地址对应的物理地址，并且通过一些检查后，才最终进行内存访问，所以目录表、页表的基础内容，以及 CR3 寄存器的指向，都要在实模式运行阶段设置好。  
    进入保护模式后，指令中包含的地址，就是虚拟地址了，类似于数学符号 (不过和数学符号也是有区别的，哪怕某个复杂的数学推导过程将α、β、γ、A-Z 都用完了，也还有 A0、A1、A10000.. 可以用，而虚拟地址的个数是有限的，比如对于 32 位系统，虚拟地址 = 指针值，也就是一个 32 位的值，所以只有 2^32 个)，虚拟地址只是起到指代物理地址的作用，执行时具体由哪个物理地址“扮演” 它，就可以通过软件层面进行设计了，换句话说就是，CPU 保护模式的这种硬件特性，给了内核可以在底层进行地址映射的机会，从而让物理地址对上层程序透明。
*   **总结**  
    ① 早期 CPU 只支持模式，它直接 “承认” 指令中包含的地址，分配 / 释放机制只能给程序编写带来点便利，但起不到防止内存冲突的作用，因为即使记录了哪些空间正在被别的任务，甚至是系统使用，但只要某个任务有 bug，甚至是故意的，仍然可以对这些空间进行随意访问和修改。  
    ② CPU 进入保护模式后，只会 “承认” 已经记录映射关系的地址，**映射过程由硬件执行，映射关系由软件设置**，这其实是硬件层向软件层提出了一种要求，也正是由于这种要求，内核才有机会限制用户态代码访问任意内存，将 CPU 的充分使用权，掌握在自己手里，从而保证系统的稳定性。

2. 内存管理的环节
----------

  刚才已经说了，CPU 的权限是支持调节的，最高权限时，可以执行所有指令，包括将权限调低，但低权限向高权限调节，就需要接受额外的条件 (否则权限如果可以随意调高，那么权限设计的意义就没了)，那就是必须穿过“门” 进行提升，而在穿过 “门” 的同时，也要求执行一次代码跳转 (这种要求，以及“门” 的设计，也是硬件层实现的)，而内核代码是在 CPU 刚上电时就开始执行了，相比用户程序，内核具有设置这个跳转地址的先机，所以当然会将其设置为内核代码的地址。这样，内核代码就总能在最高权限执行，而用户代码总是在低权限执行，同时又有路径从用户态 (低 CPU 权限下执行用户代码的状态) 回到内核态 (最高 CPU 权限下执行内核代码的状态)。很多特权指令，只在 CPU 高权限状态时有效，比如 CR3 寄存器的读 / 写指令，从而“强制” 用户程序使用内核接口，对进程本身的虚拟空间和占用的物理内存，进行管理。

*   **内核与 glibc**  
    ![](https://bbs.pediy.com/upload/tmp/815036_DKKUVT7ACZBJUN3.png)  
    CPU 保护模式下，指令中包含的地址是虚拟地址，最终能访问到哪个物理地址，完全由底层的内核代码决定，而内核对 3G~4G 虚拟空间的映射，只会记录一份，对 0~3G 虚拟空间的映射，为每个进程单独记录一份。所有进程访问 3G~4G 虚拟空间，会访问到相同的物理内存，因此也是所有进程的公共空间，为了防止被某个进程任意读写，内核在映射 3G~4G 范围的地址时，同时也会向硬件标记，只有最高权限才能访问，从而使进程希望访问内核空间时，必须切换到内核态，由内核代码代为执行，正是这个原因，3G~4G 虚拟空间被称为 “内核空间”；不同进程访问 0~3G 虚拟空间，访问到的都是内核只为自己映射的物理内存，称为 “用户空间”。
    *   内核中的内存管理
        *   物理内存
        *   虚拟空间 -- 所有进程公共的内核空间
        *   虚拟空间 -- 各个进程私有的用户空间
    *   glibc 中的内存管理
        *   当前进程用户空间中已分配的部分 (内核侧)
        *   还未分配给应用程序的部分 (业务侧)
*   **glibc 内存管理目的**  
    为什么要在内核管理的外面，加一层 glibc 管理？  
    反正肯定不是因为开发内核的大佬都是菜鸟，能力已经发挥到极限了，只能提供 brk()、mmap()/munmap()这几个简陋的接口给用户程序使用，而是因为内核处于最底层，不能对上层应用场景的具体要求做过多的假设。比如除了 ptmalloc，还有 google 开发的 TCMalloc 等其它内存管理库，它们对内存利用率、分配性能、最佳适合场景的权衡都不一样，所以内核如果在最底层偏向任何一方，就会在上层损失灵活性，因此内核的接口就只能提供最基础、最原始功能。内核是按照段或页为单位进行内存管理的，其中，段式内存管理目前几乎已经淘汰了，因此可以认为内核的接口，必须按页进行分配 / 释放 (这是由 CPU 的硬件特性决定的，这样设计的好处是，记录 1 对虚拟页面地址和物理页面地址的映射关系，就为 4K 个地址都建立了映射)，所以，如果用户程序直接使用系统调用，哪怕只分配 1 个字节，也会得到 4K 大小的内存，如果希望尽可能的利用剩下的(4K-1) 空间，需要用户程序自己实现逻辑进行控制，可以认为 glibc 正是为用户程序提供一套这样的接口，glibc 按 chunk 为单位对内存进行再次划分，最小 chunk 只有 16 字节。  
    另外，glibc 会将应用程序已经释放的内存，准确说，只是业务层已经不需要的内存，仍然维持在用户态，当业务层再次需要内存时，再将它们返回给业务层，相比每次直接使用系统调用分配 / 释放，可以避免用户态和内核态之间频繁切换，从而提高程序的性能。可以将 glibc 理解为业务层与内核之间的缓存，当缓存达到一定阈值后，仍然会通过系统调用最终释放 (它的释放时机始终存在，跟由内存泄漏是两回事)，业务层释放的内存不管是内核 “保管”，还是 glibc“保管”，从逻辑意义上来讲，这块内存肯定不会跟正使用内存冲突了 (否则就是业务层本身有 bug)，所以再次返回给业务层，也是没有问题的。

3. glibc 内存管理的实现
----------------

  这块内容是参考《Glibc 内存管理 --Ptmalloc2 源代码分析. pdf》学习的，这份 pdf 文件总共 129 页，分析过程非常详细、深刻，为了防止遗忘，我整理了一个主要过程出来，并且为有些部分添加了配图和个人的理解。

*   **进程空间布局**  
    ![](https://bbs.pediy.com/upload/tmp/815036_XYGZK6WQVKACVGZ.png)  
    32 位高版本内核采用的内存布局，是 mmap 区域的最高地址固定，向下增长，而 64 位内存布局和 32 位经典内存布局是同一种布局，都是 mmap 区域的最低地址固定，向上增长，更细节的内容可以参考：[https://www.iteye.com/blog/mqzhuang-901602](https://www.iteye.com/blog/mqzhuang-901602)。  
    需要知道的是，内核就是从 heap 区域和 mmap 区域，分配虚拟空间给用户程序，其中，heap 底部位置始终固定不动，从 heap 区域分配就是抬高 heap 顶部的位置，因此只可能存在一个 heap 已分配空间，而从 mmap 区域分配，可以取 mmap 区域中任意一块空闲的区间，因此可以存在多个 mmap 已分配空间，并且每个分配空间的起始和结束位置，是固定不动的。另外，heap 和 mmap 区域的分配 / 释放接口 (系统调用) 分别为 skb()、mmap()/munmap()，参数中表示分配位置或分配大小的值，必须按页对齐。
*   **Arena**  
    (下图引用于：[https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/))  
    ![](https://bbs.pediy.com/upload/tmp/815036_WKKZQFNWUMBQA9E.png)  
    glibc 介于内核与业务层之间，所以它的任务就是，通过内核分配 Arena，再从 Arena 上切割 Chunk，提供给业务层，并对所有的 Arena 和 Chunk 进行管理。
    *   Main Arena  
        Main Arena 对应 heap 已分配区域，所以有且只有一个，并且它的 Top Chunk 顶部位置，是随着 heap 已分配区域的顶部变化的。其中的 Allocated Chunk，表示正在被业务层使用的空间，Free Chunk 表示空闲空间，比如业务层使用完释放掉的空间，除此之外还有其它信息需要记录，都是通过一个 malloc_state 结构的全局变量进行管理的。
    *   Thread Arena  
        Thread Arena 对应一个或多个 mmap 已分配区域，和 Main Arena 不同，Thread Arena 可以有多个，每个 Thread Arena 也可以包含多个 mmap 已分配区域，并且它们的 Top Chunk 顶端都不会变动。另外，Thread Arena 的 malloc_state 管理结构，直接嵌套在它包含的首个 mmap 已分配区域中，并且其中的每个 mmap 已分配区域的开始位置，都会有一个 heap_info 结构 (这是因为 mmap 分配区域需要一点额外的管理信息，比如这块区域中有多少部分已经映射为可读写区域了，以及，当 Thread Arena 包含多个 mmap 已分配区域时，还需要有链表结点，将它们链在一起)。
        *   问题 1：为什么需要 Thread Arena？
        *   问题 2：Thread Arena 可以扩充，为什么还需要多个 Thread Arena？  
            问题 1(Main Arena 也是可以扩充的)和问题 2 实际上是一样的，都可以转换为：为什么需要多个 Arena？原因其实很简单，Thread Arena 是为了避免或减小多线程之间的竞争，额外创建的 Arena，这也是 “Thread Arena” 这个名称的来历，所以 Thread Arena 和 Main Arena 在使用时，是平等的，主线程可以用 Thread Arena，子线程也可以用 Main Arena，千万不能望文生义的认为，主线程只能使用 Main Arena，或者 Main Arena 只提供给主线程专用。
    *   malloc_state 结构
        
        ```
        struct malloc_state {
          /* Serialize access.  */
          mutex_t mutex;
          /* Flags (formerly in max_fast).  */
          int flags;
        #if THREAD_STATS
          /* Statistics for locking.  Only used if THREAD_STATS is defined.  */
          long stat_lock_direct, stat_lock_loop, stat_lock_wait;
        #endif
          /* Fastbins */
          mfastbinptr      fastbinsY[NFASTBINS];
          /* Base of the topmost chunk -- not otherwise kept in a bin */
          mchunkptr        top;
          /* The remainder from the most recent split of a small request */
          mchunkptr        last_remainder;
          /* Normal bins packed as described above */
          mchunkptr        bins[NBINS * 2 - 2];
          /* Bitmap of bins */
          unsigned int     binmap[BINMAPSIZE];
          /* Linked list */
          struct malloc_state *next;
        #ifdef PER_THREAD
          /* Linked list for free arenas.  */
          struct malloc_state *next_free;
        #endif
          /* Memory allocated from the system in this arena.  */
          INTERNAL_SIZE_T system_mem;
          INTERNAL_SIZE_T max_system_mem;
        };
        
        ```
        
        malloc_state 结构用于记录 Arena 管理信息：
        *   mutex：防止不同线程同时使用同一个 Arena；
        *   top：描述 Arena 的 Top Chunk 信息；
        *   next：将进程的所有 Arena 链成一条链表；
        *   next_free：将当前未被任何线程使用的 Arena 链成一条链表；
        *   system_mem：Arena 中已分配给业务层的空间大小；
        *   max_system_mem：Arena 最多能分配给业务层的空间大小，max_system_mem-system_mem 即为 Arena 中空闲空间的大小；
        *   其余主要成员，通过后续内容一一介绍。
    *   heap_info 结构
        
        ```
        typedef struct _heap_info {
          mstate ar_ptr; /* Arena for this heap. */
          struct _heap_info *prev; /* Previous heap. */
          size_t size;   /* Current size in bytes. */
          size_t mprotect_size;    /* Size in bytes that has been mprotected
                       PROT_READ|PROT_WRITE.  */
          /* Make sure the following data is properly aligned, particularly
             that sizeof (heap_info) + 2 * SIZE_SZ is a multiple of
             MALLOC_ALIGNMENT. */
          char pad[-6 * SIZE_SZ & MALLOC_ALIGN_MASK];
        } heap_info;
        
        ```
        
        *   ar_ptr：所属的 Arena；
        *   prev：与同一 Arena 中的其它 heap 链在一起；
        *   size：heap 大小；
        *   mprotect_size：还未设置可读写属性的区域大小；
        *   pad：使 heap_info 对象按 MALLOC_ALIGNMENT 对齐 (其实是为了让紧挨着的 Chunk 起始地址按 MALLOC_ALIGNMENT 对齐)，由于 pad 以上的成员大小加起来已经满足对齐要求，所以 pad 数组的大小为 0 即可 (不管 32 位还是 64 位环境下，-6 * SIZE_SZ & MALLOC_ALIGN_MASK 算出来都是 0，这里还没有理解为什么要写的这么奇怪)。
*   **Chunk**  
    说到这里，已经很容易明白，glibc 与内核之间，是按 Arena(对于 Thread Arena，是其中包含的 heap) 为单位分配 / 释放，glibc 与业务层之间，是按 Chunk 为单位分配 / 释放。
    *   每个 Chunk 头部，包含一个 malloc_chunk 管理结构
        
        ```
        struct malloc_chunk {
          INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
          INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */
          struct malloc_chunk* fd;         /* double links -- used only if free. */
          struct malloc_chunk* bk;
          /* Only used for large blocks: pointer to next larger size.  */
          struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
          struct malloc_chunk* bk_nextsize;
        };
        
        ```
        
        *   prev_size：紧挨着当前 Chunk 的上个 Chunk 的大小；
        *   size：最低 3 位为属性位 (稍后说明)，属性位清零即为当前 Chunk 的大小 (包括管理信息部分，即以下结构说明图中橙色、蓝色、灰色空间的大小)；
        *   fd：当前 Chunk 如果是 Free Chunk，会与其它 Free Chunk 链在一起，fd 表示链在当前 Chunk 前面的 Free Chunk；
        *   bk：当前 Chunk 如果是 Free Chunk，会与其它 Free Chunk 链在一起，bk 表示链在当前 Chunk 后面的 Free Chunk；
        *   fd_nextsize：后面说明；
        *   bk_nextsize：后面说明。
    *   结构说明  
        ![](https://bbs.pediy.com/upload/tmp/815036_HZUGHRCMHQ3QDT7.png)
        *   空间复用
            *   这个结构并不是从当前 Chunk 的头部开始，而是从上个 Chunk 的最后 4 个字节处开始。原因：只有在释放当前 Chunk，并且上个 Chunk 为 Free Chunk，从而需要合并当前 Chunk 和上个 Chunk 的时候，才需要根据 prev_size 找上个 Chunk，而既然上个 Chunk 这时为 Free Chunk，所以自然可以使用上个 Chunk 自己的空间记录 prev_size，这样当前 Chunk 就可以少花 4 字节记录空间，进一步而言，业务层就可以从当前 Chunk 中多使用 4 个字节。
            *   fd 和 bk 只在当前 Chunk 是 Free Chunk 时有意义，当前 Chunk 如果为 Allocated Chunk，fd 和 bk 所在的空间，会作为 user data(业务层使用空间) 的一部分。原因：fd,bk 只为了方便搜索和组织不同大小的 Free Chunk(如果不考虑性能，直接从 Arena 中也能找出所有的 Free Chunk)，所以也只有 Free Chunk 才需要 fd,bk，而既然已经是 Free Chunk 了，自然可以使用 user data 记录 fd,bk。也正是由于 Free Chunk 必须记录 size,fd,bk，以及为下个 Chunk 提供记录 prev_size 的空间，所以 Chunk 最小大小为 16 字节。
            *   fd_nextsize 和 bk_nextsize，只在当前 Chunk 是 Free Chunk，并且大小≥512 字节时有意义。首先，和 fd,bk 一样，Free Chunk 的 fd_nextsize 和 bk_nextsize，复用了 Allocated Chunk 的 user data 空间；另外，chunk 的大小是按 8 字节对齐的，所以 [16,512) 字节区间的 Chunk，只有 62 种大小，每种大小都有一一对应的链表头，而≥512 的 Free Chunk，不同大小是要链在同一链表中的，fd_nextsize 和 bk_nextsize 是为了提高插入排序的性能，相同大小的多个 Free Chunk，只用比较一次。  
                ![](https://bbs.pediy.com/upload/tmp/815036_5CYUCMY8YRAGUU3.png)
    *   标志位
        *   P：表示上个 Chunk 是否空闲
            *   当前 Chunk 本身是否空闲，可以通过当前 Chunk 的 size，找到下一个 Chunk，进而根据下一个 Chunk 的 P 标志位进行判断；
            *   P 标志用于判断 prev_size 是否有意义 (如果 P 标志用于表示当前 Chunk 自己是否空闲，就矛盾了：想知道 prev_size 是否有意义，必须先用 prev_size 计算上个 Chunk 的位置，进而才能获取 P 标志位的值)。
        *   M：表示是否**直接从 mmap 区域分配**  
            这个标志容易理解错，虽然 Thread Arena 是从 mmap 区域分配的，但 Thread Arena 上的 Chunk，并不设置 M 标志位，所以 M 标志位的含义，其实可以换个说法，即表示是否从 Arena 分配。相应的，M=1 时，释放必须通过 munmap() 直接归还到 mmap 区域。
        *   A：表示是否从 Main Arena 分配
*   **Bins**  
    _为避免混淆，在此声明一下：通过我在学习过程中看到的一些文档，都认为 Bins = unsorted bin + small bins + large bins，而 Fast bins 不属于 Bins，bin 也是指 Bins 中的某个 bin，而跟 Fast bins 无关。_  
    ![](https://bbs.pediy.com/upload/attach/202201/815036_PT8URRBH5EEYX6R.png)  
    Fast bins 和 Bins 其实就是两个数组，数组中的元素都是用于悬挂 Free Chunk 的链表头。Fast bins 用于存放大小为 16~80 字节的 Free Chunk(默认为 16~64 字节，业务层可以通过 mallopt() 函数设置 Fast bins 有效部分的大小)，small bins 用于存放大小为 16~504 字节的 Free Chunk，large bins 用于存放≥512 字节的 Free Chunk。
    *   bin 结构  
        ![](https://bbs.pediy.com/upload/tmp/815036_EB46YG8R3A6RU3Y.png)  
        Fast bins 中都是单向链表头，而 bin 都是双向链表头，需要说一下的是，glibc 代码对 bin 的使用方式有点特别，比如使用 bin1 的时候，它是通过 bin_at(1)，将 bin0+bin1 视为一个 malloc_chunk 结构，然后通过这个结构，使用 bin1 中的 fd,bk 成员。
    *   large bins  
        bins 中第一个 bin 不使用、第二个 bin 作为 unsorted bin，剩余部分从整体上看划分成了 7 份，分别包含 64、32、16、8、4、2、1 个 bin，前面 6 份都包含多个 bin，相邻两个 bin 允许存放的最小 Free Chunk 的公差分别为 8、8^2、8^3、8^4、8^5、8^6，由于 Chunk 大小最小也相差 8 字节，所以第 1 份，即 small bins 中的每个 bin，存放的 Free Chunk，大小肯定都是相等的，而后面 6 份，即 large bins 中的每个 bin，存放的 Free Chunk，大小都是一个区间。
        
        ```
        #define largebin_index_32(sz)                                                \
        (((((unsigned long)(sz)) >>  6) <= 38)?  56 + (((unsigned long)(sz)) >>  6): \
         ((((unsigned long)(sz)) >>  9) <= 20)?  91 + (((unsigned long)(sz)) >>  9): \
         ((((unsigned long)(sz)) >> 12) <= 10)? 110 + (((unsigned long)(sz)) >> 12): \
         ((((unsigned long)(sz)) >> 15) <=  4)? 119 + (((unsigned long)(sz)) >> 15): \
         ((((unsigned long)(sz)) >> 18) <=  2)? 124 + (((unsigned long)(sz)) >> 18): \
                        126)
        
        ```
        
        通过上述规律，再来看 largebin_index_32() 这个宏的逻辑，就不至于摸不清东南西北了。既然执行到这个宏，说明 sz 肯定是属于 large bins 范围的，宏里面就是要进一步确定，sz 属于 6 个部分中的哪一个，然后根据相应的公差最终计算所属 bin 的下标值。
        *   右移 6、9、12、15、18，分别表示除以不同部分的公差 8^2、8^3、8^4、8^5、8^6；
        *   small bins 中有 8 个 8^2，large bins 第 1 部分有 32 个 8^2，所以除以 8^2 如果达到 40，那必须在第 2 部分的范围，但代码中是 38，可能是由于除法计算会损失 1，下标也要减 1；
        *   large bins 的下标从 64 开始，但是 small bins 有 8 个 8^2，所以 large bins 第 1 部分范围中的大小，除以 8^2，至少是 8，所以是基于 56 相加。
    *   “Bins 缓存”  
        Fast bins 可以理解为 small bins 部分区间的缓存 (16~64/80)，并且是 “一级缓存”，unsorted bin 可以理解为整个 small bins 和 large bins 的缓存，并且是 “二级缓存”。
*   **malloc()、free() 流程**  
    详细过程，建议参考这里直接借个图，引用于：[https://www.cnblogs.com/unr4v31/p/14446412.html](https://www.cnblogs.com/unr4v31/p/14446412.html)  
    ![](https://bbs.pediy.com/upload/attach/202201/815036_7GFYRRD729SJVZV.png)

参考
--

《Glibc 内存管理 --ptmalloc2 源代码分析》博客系列：[https://www.iteye.com/blog/user/mqzhuang](https://www.iteye.com/blog/user/mqzhuang)

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#缓冲区溢出](forum-150-1-156.htm) [#UAF](forum-150-1-158.htm) [#Linux](forum-150-1-161.htm)