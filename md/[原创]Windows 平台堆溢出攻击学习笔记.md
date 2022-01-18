> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271213.htm)

> [原创]Windows 平台堆溢出攻击学习笔记

一. 堆的作用  

==========

栈是分配局部变量和存储函数调用参数的主要场所，系统在创建每个线程时会自动为其创建栈。对于 C/C++ 这样的编程语言，编译器在编译阶段会生成合适的代码来从栈上分配和释放空间，不需要编写任何的代码。但是，栈空间（尤其内核态）的容量相对较小，为了防止溢出，不适合在栈上分配特别大的内存区。其次，由于栈帧通常是随着函数的调用和返回而创建和消除的，因此分配在栈上的变量只在函数内有效，这使栈只适合分配局部变量，不适合分配需要较长生存期的全局变量和对象。  

堆克服了栈的以上局限性，是程序申请和使用内存空间的另一种途径。应用程序通过内存分配函数 (malloc 或 HeapAlloc) 或 new 操作符获得的内存空间都来自堆。从操作系统角度看，堆是系统的内存管理功能向应用软件提供服务的一种方式。通过堆，内存管理器将一块较大内存委托给堆管理器来管理。堆管理器将大块的内存分割成不同大小的很多个小块来满足应用程序的需要。应用程序的内存需求通常是频繁而且零散的，如果把这些请求都直接传递给位于内核中的内存管理器，那么必然会影响系统的性能。有了堆管理器，内存管理器就只需要处理大规模的分配请求。这样做不仅可以减轻内存管理器的负担，而且可以大大缩短应用程序申请内存分配所需的时间，提高程序的运行速度。从这个意义上说，堆管理器就好像经营内存零售业务的中间商，它从内存管理器那里批发大块内存，然后零售给应用程序的各个模块。

> **总结：**  
> 
> **一个应用程序在执行过程中，会经常需要申请一块内存来使用。如果每次都进入到内核中，通过内存管理器来分配内存，就会给系统带来比较大的开销。为了减少这种开销，Windows 系统提供了堆管理器。堆管理器会先进入内核中，申请一块比较大的内存存放在用户空间中，以供应用程序使用。当应用程序分配内存的时候，首先会检查要分配的内存大小是否符合要求（不要太大）。如果符合要求，就会从堆管理器中已经分配好的较大的内存中取出一块较小的内存块供程序使用。这样，应用程序申请小内存块的时候，就不需要频繁地进入到内核中去申请，降低了性能开销。如果应用程序申请的内存比较大，那么就会进入内核，通过内存管理器来分配内存**

二. 堆的创建和销毁  

=============

1. 进程的默认堆  

------------

Windows 系统在创建一个新的进程时，在加载器函数执行进程的用户态初始化阶段，会调用 RtlCreateHeap 函数为新的进程创建第一个堆，称为进程的默认堆，有时候简称进程堆。PEB 中的以下字段用来描述进程堆的信息：  

```
kd> dt _PEB
ntdll!_PEB
   +0x018 ProcessHeap      : Ptr32 Void
   +0x078 HeapSegmentReserve : Uint4B
   +0x07c HeapSegmentCommit : Uint4B

```

<table border="1"><tbody><tr><td valign="top"><strong>成员</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">ProcessHeap</td><td valign="top">进程堆的句柄</td></tr><tr><td valign="top">HeapSegmentReserve</td><td valign="top">堆的默认保留大小，字节数，1MB(0x100000)</td></tr><tr><td valign="top">HeapSegmentCommit</td><td valign="top">堆的默认提交大小，其默认值为两个内存页大小；x86 系统中普通内存页的大小为 4KB，因此是 0x2000，即 8KB</td></tr></tbody></table>

Windows 提供以下的函数来获得进程堆句柄，该函数只是简单地找到 PEB 结构，然后读出 ProcessHeap 字段的值。  

```
HANDLE WINAPI GetProcessHeap(void);

```

2. 创建默认堆
--------

除了系统为每个进程创建的默认堆，应用程序也可以通过以下函数来创建其他堆，这样的堆只能被发起调用的进程访问，通常称为私有堆。  

```
HANDLE WINAPI HeapCreate(
  __in  DWORD flOptions,
  __in  SIZE_T dwInitialSize,
  __in  SIZE_T dwMaximumSize);

```

<table border="1"><tbody><tr><td valign="top"><strong>参数</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">flOptions</td><td valign="top"><p>该参数可以是如下标志中的 0 个或多个：</p><ul><li><p>HEAP_GENERATE_EXCEPTIONS(0x00000004)，通过异常来报告失败情况，如果没有该标志则通过返回 NULL 报告错误</p></li><li><p>HEAP_CREATE_ENABLE_EXECUTE(0X00040000)，允许执行堆中内存块上的代码</p></li><li><p>HEAP_NO_SERIALIZE(0x00000001)，当堆函数访问这个堆时，不需要进行串行化控制（加锁）。指定这一标志可以提高堆操作函数的速度，但应该在确保不会有多个线程操作同一个堆时才这样做，通常在将某个堆分配给某个线程专用时这么做。也可以在每次调用堆函数时指定该标志，告诉堆管理器 u 需要堆那次调用进行串行化控制<br></p></li></ul></td></tr><tr><td valign="top">dwInitialSize</td><td valign="top">用来指定堆的初始提交大小</td></tr><tr><td valign="top">dwMaximumSize</td><td valign="top">用来指定堆空间的最大值（保留大小），如果为 0，则创建的堆可以自动增加。尽管可以使用任意大小的整数作为 dwInitialSize 和 dwMaximumSize 参数，但是系统会自动将其取整为大于该值的临近页边界（即页大小的整数倍）</td></tr></tbody></table>

HeapCreate 内部主要调用 RtlCreateHeap 函数，因此私有堆和默认堆并没有本质的差异，只是创建的用途不同。RtlCreateHeap 内部会调用 ZwAllocateMemory 系统服务从内存管理器申请内存空间，初始化用于维护堆的数据结构，最后将堆句柄记录到进程的 PEB 结构中，确切地说是 PEB 结构地堆列表中。  

3. 堆列表  

---------

每个进程的 PEB 结构以列表的形式记录了当前进程的所有堆句柄，包括进程的默认堆。以下是 PEB 结构中，用来记录这些堆句柄的字段：  

```
kd> dt _PEB
ntdll!_PEB
   +0x018 ProcessHeap      : Ptr32 Void
   +0x088 NumberOfHeaps    : Uint4B
   +0x08c MaximumNumberOfHeaps : Uint4B
   +0x090 ProcessHeaps     : Ptr32 Ptr32 Void

```

<table border="1"><tbody><tr><td valign="top"><strong>成员</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">ProcessHeap</td><td valign="top">进程默认堆句柄</td></tr><tr><td valign="top">NumberOfHeaps</td><td valign="top">记录堆的总数</td></tr><tr><td valign="top">MaximumNumberOfHeaps</td><td valign="top">指定 ProcessHeaps 数组最大个数，当 NumberOfHeaps 达到该值的大小，那么堆管理器就会增大 MaximumNumberOfHeaps 的值，并重新分配 ProcessHeaps 数组</td></tr><tr><td valign="top">ProcessHeaps</td><td valign="top">记录每个堆的句柄，是一个数组，这个数组可以容纳的句柄数记录在 MaximumNumberOfHeaps 中</td></tr></tbody></table>

> 和其他的句柄不同的是，堆句柄的值实际上就是这个堆的起始地址。和其他函数创建的对象保存在内核空间中不同，应用程序创建的堆是在用户空间保存的，因此应用程序可以直接通过该地址来操作堆，而不用担心操作失误造成蓝屏错误。所以，此时的句柄值就可以直接是堆的起始地址  

4. 销毁堆  

---------

应用程序可以调用以下函数来销毁进程的私有堆。该函数内部主要调用 NTDLL 中的 RtlDestoryHeap 函数。后者会从 PEB 的堆列表中将要销毁的堆句柄移除，然后调用 NtFreeVirtualMemory 向内存管理器归还内存  

```
BOOL WINAPI HeapDestroy(__in  HANDLE hHeap);

```

三. 分配和释放堆块  

=============

当应用程序调用堆管理器的分配函数向堆管理器申请内存时，堆管理器会从自己维护的内存区中分割除一个满足用户指定大小的内存块，然后把这个块中允许用户访问部分的起始地址返回给应用程序，堆管理器把这样的块叫一个 Chunk，也就是 "堆块"。应用程序用完一个堆块后，应该调用堆管理器的释放函数归还堆块。  

1. 分配堆块
-------

在 Windows 系统中，从堆中分配空间的最直接方法就是调用 HeapAlloc 函数，不过该函数只是 RtlAllocateHeap 函数的别名  

```
#define HeapAlloc RtlAllocateHeap

```

RtlAllocateHeap 函数定义如下：  

```
PVOID
  RtlAllocateHeap( 
    IN PVOID  HeapHandle,
    IN ULONG  Flags,
    IN SIZE_T  Size
    );

```

<table border="1"><tbody><tr><td valign="top"><strong>参数</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">HeapHandle</td><td valign="top">要从中分配空间的内存堆的句柄</td></tr><tr><td valign="top">Flags</td><td valign="top"><p>该标志位可以是以下的组合：</p><ul><li><p>HEAP_GENERATE_EXCEPTIONS(0x00000004)：使用异常来报告失败情况，如果没有此标志，则使用 NULL 返回值来报告错误。异常代码可能是 STATUS_ACCESS_VIOLATION 或 STATUS_NO_MEMORY</p></li><li><p>HEAP_ZERO_MEMORY(0x00000008)：将所分配的内存区初始化为 0</p></li><li><p>HEAP_NO_SERIALIZE(0x00000001)：不需要堆盖茨分配实施串行控制（加锁）。如果希望堆该堆的所有分配调用都不需要串行化控制，那么可以创建堆时指定 HEAP_NO_SEROA;OZE 选项。对于进程堆，调用 HeapAlloc 时用于不应该指定 HEAP_NO_SERIALIZE 标志，因为系统代码可能随时会调用堆函数访问进程堆<br></p></li></ul></td></tr><tr><td valign="top">Size</td><td valign="top">所需内存块的字节数</td></tr></tbody></table>

可以使用 HeapReAlloc 来改变一个堆中分配的内存块的大小，该函数是 RtlReAlloc 函数的别名

```
#define HeapReAlloc RtlReAllocateHeap

```

HeapReAlloc 函数定义如下：  

```
LPVOID WINAPI HeapReAlloc(
  __in  HANDLE hHeap,
  __in  DWORD dwFlags,
  __in  LPVOID lpMem,
  __in  SIZE_T dwBytes);

```

<table border="1"><tbody><tr><td valign="top"><strong>参数</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">hHeap</td><td valign="top">堆句柄</td></tr><tr><td valign="top">dwFlags</td><td valign="top">除了可以指定上述三种标志位，还可以指定 HEAP_REALLOC_IN_PLACE_ONLY(0x00000010)，含义是不改变原来内存块的位置</td></tr><tr><td valign="top">lpMem</td><td valign="top">指向以前分配的内存块</td></tr><tr><td valign="top">dwBytes</td><td valign="top">重新分配的大小</td></tr></tbody></table>

2. 释放堆块  

----------

通过 HeapFree 释放堆块，该函数是 RtlFreeHeap 的别名

```
#define HeapFree RtlFreeHeap

```

RtlFreeHeap 函数定义如下：

```
BOOLEAN
  RtlFreeHeap( 
    IN PVOID  HeapHandle,
    IN ULONG  Flags,
    IN PVOID  HeapBase);

```

<table border="1"><tbody><tr><td valign="top"><strong>参数</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">HeapHandle</td><td valign="top">内存堆的句柄</td></tr><tr><td valign="top">Flags</td><td valign="top">可以包含 HEAP_NO_SERIALIZE(0x00000001) 标志，用来告诉堆管理器不需要堆该次调用进行防止并发的串行化操作以提高速度</td></tr><tr><td valign="top">HeapBase</td><td valign="top">用来指定要释放内存块的地址，也就是使用 HeapAlloc 函数申请内存时所得到的返回值</td></tr></tbody></table>

四. 堆的结构  

==========

Windows 通过 HEAP 结构来记录和维护堆的管理信息，该结构位于堆的开始处。当使用 HeapCreate 函数创建堆时，该函数返回的堆句柄，也就是创建的堆的地址，首先保存的就是该结构。HEAP 结构的定义如下：

```
kd> dt _HEAP
ntdll!_HEAP
   +0x000 Entry            : _HEAP_ENTRY
   +0x008 Signature        : Uint4B
   +0x00c Flags            : Uint4B
   +0x010 ForceFlags       : Uint4B
   +0x014 VirtualMemoryThreshold : Uint4B
   +0x018 SegmentReserve   : Uint4B
   +0x01c SegmentCommit    : Uint4B
   +0x020 DeCommitFreeBlockThreshold : Uint4B
   +0x024 DeCommitTotalFreeThreshold : Uint4B
   +0x028 TotalFreeSize    : Uint4B
   +0x02c MaximumAllocationSize : Uint4B
   +0x030 ProcessHeapsListIndex : Uint2B
   +0x032 HeaderValidateLength : Uint2B
   +0x034 HeaderValidateCopy : Ptr32 Void
   +0x038 NextAvailableTagIndex : Uint2B
   +0x03a MaximumTagIndex  : Uint2B
   +0x03c TagEntries       : Ptr32 _HEAP_TAG_ENTRY
   +0x040 UCRSegments      : Ptr32 _HEAP_UCR_SEGMENT
   +0x044 UnusedUnCommittedRanges : Ptr32 _HEAP_UNCOMMMTTED_RANGE
   +0x048 AlignRound       : Uint4B
   +0x04c AlignMask        : Uint4B
   +0x050 VirtualAllocdBlocks : _LIST_ENTRY
   +0x058 Segments         : [64] Ptr32 _HEAP_SEGMENT
   +0x158 u                : __unnamed
   +0x168 u2               : __unnamed
   +0x16a AllocatorBackTraceIndex : Uint2B
   +0x16c NonDedicatedListLength : Uint4B
   +0x170 LargeBlocksIndex : Ptr32 Void
   +0x174 PseudoTagEntries : Ptr32 _HEAP_PSEUDO_TAG_ENTRY
   +0x178 FreeLists        : [128] _LIST_ENTRY
   +0x578 LockVariable     : Ptr32 _HEAP_LOCK
   +0x57c CommitRoutine    : Ptr32     long 
   +0x580 FrontEndHeap     : Ptr32 Void
   +0x584 FrontHeapLockCount : Uint2B
   +0x586 FrontEndHeapType : UChar
   +0x587 LastSegmentIndex : UChar

```

其中偏移 0x178 的 FreeLists 是一个包含 128 个元素的双向链表数组，用来记录堆中空闲堆块链表的表头。当有新的分配请求时，堆管理器会遍历这个链表寻找可以满足请求大小的最接近堆块。如果找到了，便将这个块分配出去；否则，便要考虑为这次请求提交新的内存页和简历新的堆块。当释放一个堆块的时候，除非这个堆块满足解除提交的条件，要直接释放给内存管理器，大多数情况下对其修改属性并加入空闲链表中。  

FreeLists 双向链表结构如下图所示，可以看到，从一号链表 (FreeLists[1]) 开始，链表所指向的堆块的大小从 8 开始，随着索引的增加，每次增加八个字节，最大的堆块的 1016 个字节。大于等于 1024 字节的堆块，则按顺序从小到大链入零号链表 (FreeLists[0]) 中。  

![](https://bbs.pediy.com/upload/tmp/835440_2AF4RHSFMK6GAEQ.jpg)

这些链表所指向的堆块面临着被频繁使用的处境，因此，需要有相关的字段来标识它们的状态。Windows 使用八字节的 HEAP_ENTRY 结构来标识这些堆块的状态，该结构定义如下：  

```
kd> dt _HEAP_ENTRY
ntdll!_HEAP_ENTRY
   +0x000 Size             : Uint2B
   +0x002 PreviousSize     : Uint2B
   +0x004 SmallTagIndex    : UChar
   +0x005 Flags            : UChar
   +0x006 UnusedBytes      : UChar
   +0x007 SegmentIndex     : UChar

```

<table border="1"><tbody><tr><td valign="top"><strong>成员</strong></td><td valign="top"><strong>含义</strong></td></tr><tr><td valign="top">Size</td><td valign="top">堆块的大小，以分配粒度为单位</td></tr><tr><td valign="top">PreviousSize</td><td valign="top">前一个堆块的大小</td></tr><tr><td valign="top">SmallTagIndex</td><td valign="top">用于检查堆溢出的 Cookie</td></tr><tr><td valign="top">Flags</td><td valign="top">标志</td></tr><tr><td valign="top">UnusedBytes</td><td valign="top">为了补齐而多分配的字节数</td></tr><tr><td valign="top">SegmentIndex</td><td valign="top">这个堆块所在段的序号</td></tr></tbody></table>

其中的 Flags 字段代表了堆块的状态，其值由以下这些标志的组合：  

*   HEAP_ENTRY_BUSY(0x1)：该块处于占用状态
    
*   HEAP_ENTRY_EXTRA_PRESENT(0x2)：这个块存在额外描述
    
*   HEAP_ENTRY_FILL_PATTERN(0x4)：使用固定模式填充堆块
    
*   HEAP_ENTRY_VIRTUAL_ALLOC(0x8)：虚拟分配
    
*   HEAP_ENTRY_LAST_ENTRY(0x10)：该段的最后一个块  
    

HEAP_ENTRY 结构位于每个堆块的起始，用于描述相应堆块的状态，紧随该结构之后保存的就是堆块的用户数据。所以，上面的 FreeLists 双向链表所指向的堆块的大小，其实还要算上 8 字节的 HEAP_ENTRY，也就是说 FreeLists[1] 所指堆块的用户可用数据的大小为 0，FreeLists[2] 所指堆块的用户可用数据才是 8。

函数 HeapAlloc 获取的堆块用户数据的地址，将其减去 8 得到的就是该堆块的 HEAP_ENTRY 结构的地址。  

因为当某个堆块处于释放的状态的时候，需要把这个堆块链接到 FreeLists 数组中的某一个链表中，所以堆块中还应该有一个双向链表用来连接。该链表其实就保存在 HEAP_ENTRY 结构随后的八个字节中，也就是说，当堆块处于未被使用的状态的时候，此时位于堆块其实的结构其实是 HEAP_FREE_ENTRY，该结构的定义如下：

```
kd> dt _HEAP_FREE_ENTRY
ntdll!_HEAP_FREE_ENTRY
   +0x000 Size             : Uint2B
   +0x002 PreviousSize     : Uint2B
   +0x000 SubSegmentCode   : Ptr32 Void
   +0x004 SmallTagIndex    : UChar
   +0x005 Flags            : UChar
   +0x006 UnusedBytes      : UChar
   +0x007 SegmentIndex     : UChar
   +0x008 FreeList         : _LIST_ENTRY

```

该结构就是在 HEAP_ENTRY 结构后面增加一个双向链表，该链表用于连接到 FreeLists 数组中，以供程序使用。这也就能解释，为什么 FreeLists 数组里面连接的链表的堆块大小至少是 8 个字节，因为当这个堆块不用的时候，它需要八个字节来连入 FreeLists 数组中去。  

五. 漏洞原理  

==========

1. 漏洞成因  

----------

要理解该漏洞，需要对堆分配的过程有个理解，考虑以下的代码：  

```
#include  #include  int main()
{
    HANDLE hHeap = NULL;
    PVOID h1 = NULL, h2 = NULL, h3 = NULL;
 
    printf("Create\n");
    system("pause");
    hHeap = HeapCreate(NULL, 0x1000, 0x10000);
    h1 = HeapAlloc(hHeap, HEAP_ZERO_MEMORY, 0x8);
    h2 = HeapAlloc(hHeap, HEAP_ZERO_MEMORY, 0x8);
    h3 = HeapAlloc(hHeap, HEAP_ZERO_MEMORY, 0x8);
 
    printf("Free\n");
    HeapFree(hHeap, 0, h2);       // 为了防止合并，释放第二个
 
    printf("Allocate again\n");
    h2 = HeapAlloc(hHeap, HEAP_ZERO_MEMORY, 0x8);
 
    system("pause");
 
    return 0;
} 
```

这里从堆中申请了 3 块八字节的堆块，然后释放第二个堆块，让它连接进 FreeLists[1] 中，然后再从 FreeLists[1] 中将其取出。之所以要这样，是为了防止堆块合并。堆块合并的原理是当你释放该堆块的时候，它会通过 Size 和 PreviousSize 来找到它前一个堆块和后一个堆块，再查看找到的堆块的 Flags 是否处于占用状态，如果不是占用状态，就会发生堆块合并。而此时在释放 h2 的时候，因此 h1 和 h3 都在被使用，所以它检查的时候不会发生堆块的合并。

由于处于调试模式下的堆机制有不同的表现，所以这里通过暂停程序的方式调试程序。当程序处于暂停状态的时候，在将调试器附加到进程上。在 HeapCreate 函数后下断点，此时 eax 保存的就是创建的堆句柄，也就是堆地址，该地址偏移 0x178 处保存的就是 FreeLists 双向链表数组。此时除了 FreeLists[0]，也就是零号链表以外的链表都指向了自己，所以刚创建的堆，只有一个大的堆块挂在了 0 号链表中。

![](https://bbs.pediy.com/upload/attach/202201/835440_T9FJCTPN7DCK5CT.jpg)

而零号链表保存的地址是堆句柄偏移 0x688 处了，该地址处保存的 FreeList 链表连接的位置就是 FreeLists[0] 的 0x003C0178，而它前面的八个字节，则说明了这个堆块的信息，这个时候第五个字节是 0x10，没有 HEAP_ENTRY_BUSY(0x1) 标志，表明了该堆块是空闲堆块，而堆块大小 Size 为 0x130。  

![](https://bbs.pediy.com/upload/attach/202201/835440_ZDXZK7RAASERUNG.jpg)

第一次调用 HeapAlloc 的时候，eax 的保存的堆块地址是 0x003C0688，也就是 FreeLists[0]所指的大堆块，此时偏移 0x688 处的堆块被分配出来初始化为 0，而堆块的大小 (Size) 变成了 0x02，第五个的 Flags 变成的 0x01，也就表明该堆块此时处于被占用状态。

![](https://bbs.pediy.com/upload/attach/202201/835440_UEVP63Z8AURQ8N8.jpg)

大的堆块此时已经被分出了 8 字节，再算上描述两个堆块信息的 HEAP_ENTRY 结构，剩余的堆块就保存在偏移 0x698 处。此时该堆块链接进了 0 号链表中，而堆块的属性中的 Size，也从 0x130 变成了 0x12E(减 0x2)，PreviousSize 则是 0x2，属性为 0x10 表明处于空闲状态。  

![](https://bbs.pediy.com/upload/attach/202201/835440_SHF4ZF82JFZT5KK.jpg)

0 号链表此时连接的也正是剩余的堆块的地址  

![](https://bbs.pediy.com/upload/attach/202201/835440_R9R447A8FG4WDP7.jpg)

余下两次 HeapAlloc 函数的过程和上述类似，最后大堆块会被切出三个小堆块，此时这三个块的 Flags 都是 0x01，处于被占用状态。  

![](https://bbs.pediy.com/upload/attach/202201/835440_ZNJBSQ9PEQ38XGR.jpg)

当执行 HeapFree 的时候，第二个块就会被释放出来，此时第二个块的 Flags 会变成 0x00 的空闲状态，但是因为相邻的两个块都处于占用状态，因此不会发生堆块的合并。那么这个八字节大的堆块就会被链接到 FreeLists[2] 中，该链表的偏移就是 0x188  

![](https://bbs.pediy.com/upload/attach/202201/835440_T4GAKZBS4WZFTFJ.jpg)

此时也可以看到 FreeLists[2] 链表所指的地址就是第二个堆块的可用数据地址  

![](https://bbs.pediy.com/upload/attach/202201/835440_CUZ3Q5B388DVH8N.jpg)

此时再次调用 HeapAlloc 就会将刚才释放的第二个堆块分配出来，第二个堆块的属性和数据再次变成下图所示  

![](https://bbs.pediy.com/upload/attach/202201/835440_UKRTXCBEFDBWUMD.jpg)

而 FreeLists[2] 链表再次链接到自己，因此可以知道这一次的分配就不是从 0 号链表，也就是 FreeLists[0] 中所指的大的堆块中切出一小块使用。而是直接将挂入到 FreeLists[2] 链表中的堆块取出使用。  

![](https://bbs.pediy.com/upload/attach/202201/835440_NPWFENS5ZP78U23.jpg)

从 FreeLists[2] 链表中取出堆块就需要用到第二个堆块中的链表结构，也就是_HEAP_FREE_ENTRY 中的 FreeList。可是，此时第二个堆块是紧跟在第一个堆块后面的，如果用户输入到第一个堆块的数据长度过长，就会导致数据淹没到第二个堆块中，修改了堆块的 FreeList，这样在分配该堆块的时候就会因为链表所指的地址是非法的，导致程序出现错误。  

在上述代码第二次申请堆块前加入如下的代码：

```
printf("Allocate again\n");
char szExp[0x18] = { 0 };
memset(szExp, 'A', 0x18);
memcpy(h1, szExp, sizeof(szExp));
h2 = HeapAlloc(hHeap, HEAP_ZERO_MEMORY, 0x8);

```

再次编译运行，在 memcpy 函数处下断点，可以看到此时第二个堆块的保存是正常的，它的链表指向了 FreeLists[2]  

![](https://bbs.pediy.com/upload/attach/202201/835440_QAEVE5PGKRXJ9TQ.jpg)

而执行完 memcpy 之后，可以看到第二个堆块的数据被淹没了，包括 FreeList 链表也被淹没成了 0x41  

![](https://bbs.pediy.com/upload/attach/202201/835440_QYRVYNZRWDKW42H.jpg)

此时调用 HeapAlloc 申请第二个堆块的时候，就会因为链表被淹没成了 0x41 导致程序崩溃

![](https://bbs.pediy.com/upload/attach/202201/835440_6JPHZE6WFQSDR5J.jpg)

2. 漏洞利用  

----------

想要利用这个漏洞还需要明白程序是怎么将堆块从 FreeLists 数组链表中摘除的，在 RtlAllocateHeap 中，首先会要申请的堆大小进行更改

```
    rounded_size = ROUND_SIZE(size);
    if (rounded_size < HEAP_MIN_DATA_SIZE) rounded_size = HEAP_MIN_DATA_SIZE;

```

其中 HEAP_MIN_DATA_SIZE 被定义为 16

```
#define HEAP_MIN_DATA_SIZE    16

```

然后从空闲链表 FreeLists 中找到可以分配的堆块  

```
   /* Locate a suitable free block */
 
    if (!(pArena = HEAP_FindFreeBlock( heapPtr, rounded_size, &subheap )))
    {
        TRACE("(%p,%08lx,%08lx): returning NULL\n",
                  heap, flags, size  );
        if (!(flags & HEAP_NO_SERIALIZE)) RtlLeaveHeapLock( &heapPtr->critSection );
        if (flags & HEAP_GENERATE_EXCEPTIONS) RtlRaiseStatus( STATUS_NO_MEMORY );
        return NULL;
    }

```

找到以后，就调用 list_remove 从链表中删除找到的堆块

```
   /* Remove the arena from the free list */
 
    list_remove( &pArena->entry );

```

而 list_remove 函数的实现如下：

```
/* remove an element from its list */
__inline static void list_remove( struct list *elem )
{
    elem->next->prev = elem->prev;
    elem->prev->next = elem->next;
}

```

而 elem->prev->next = elem->next 转成反汇编就会得到如下的结果：

```
.text:0040D456                 mov     ecx, [ebp+pList]
.text:0040D459                 mov     edx, [ecx+4]    ; pList->prev赋给edx
.text:0040D45C                 mov     eax, [ebp+pList] 
.text:0040D45F                 mov     ecx, [eax]       ; pList->next赋给eax
.text:0040D461                 mov     [edx], ecx

```

又因为此时可以控制 next 和 prev 的值，因此，这是一个任意地址写任意数据的漏洞。此时可以通过将 next 修改为 shellcode 的地址，而 prev 修改为一个想要攻击的目的地址，就可以实现将 shellcode 的地址写入目的地址。  

此时假设目的地址是栈中保存了返回地址的栈地址，那么上述代码的运行逻辑就会变成如下：

```
.text:0040D456                 mov     ecx, [ebp+pList]
.text:0040D459                 mov     edx, [ecx+4]     ; pList->prev赋给edx，此时edx为保存返回地址的栈地址
.text:0040D45C                 mov     eax, [ebp+pList] 
.text:0040D45F                 mov     ecx, [eax]    ; pList->next保存了shellcode地址，这句话等价于 mov ecx, dwShellCodeAddr
.text:0040D461                 mov     [edx], ecx    ; 等价于 mov [dwRetAddr], ecx 这里的dwRetAddr是保存了返回地址的栈地址

```

可以看到，最终程序会将 shellcode 的地址赋值到保存返回地址的栈地址中。那么，接下来程序继续运行退出函数的时候就会转去执行 shellcode。另外，由于 elem->next->prec = elem->prev 的存在，可以由下反汇编知道，shellcode+4 的地址中的内容会被修改，所以 shellcode 的开始处要进行一定的处理  

```
.text:0040D448                 mov     eax, [ebp+pList]
.text:0040D44B                 mov     ecx, [eax]
.text:0040D44D                 mov     edx, [ebp+pList]
.text:0040D450                 mov     eax, [edx+4]
.text:0040D453                 mov     [ecx+4], eax

```

目标地址的设定可以是多种的，最常用的以下的三种分别会造成不同的攻击：

*   栈帧中的函数返回地址：函数返回时，跳去执行 shellcode
    
*   栈帧中的 SEH 句柄：异常发生时，跳去执行 shellcode
    
*   重要函数调用地址：函数调用时，跳去执行 shellcode  
    

然而，上述的攻击只能在 xp sp2 之前的系统生效。原因是从 sp2 开始，在卸载堆块的时候，函数会首先判断堆块的前向指针和后向指针的完整性，以防止漏洞被利用，此时的 list_remove 函数如下：  

```
__inline static void list_remove( struct list *elem )
{
    if (elem->next->prev == node && elem->prev->next == node)
    {
        elem->next->prev = elem->prev;
        elem->prev->next = elem->next;
    }
}

```

且 HEAP_ENTRY 中的 SmallTagIndex 也被用来检测是否发生堆溢出。从 Windows Vista 及后续的版本中又引入了元数据加密的保护机制，该机制会在块首中的一些重要数据在保存时会与一个 4 字节额随机数进行异或运算，在使用这些数据的时候需要再一次进行异或运算来还原，如此一来就不可以破坏这些数据。

因此在实际利用中就需要找到其他办法来实现漏洞的利用

六. 参考资料  

==========

> *   《0day 安全：软件漏洞分析技术》
>     
> *   《软件调试》（第二版）
>     

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#缓冲区溢出](forum-150-1-156.htm) [#Windows](forum-150-1-160.htm)