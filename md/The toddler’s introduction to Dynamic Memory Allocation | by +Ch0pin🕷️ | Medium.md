> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [valsamaras.medium.com](https://valsamaras.medium.com/the-toddlers-introduction-to-dynamic-memory-allocation-300f312cd2db)

> Heap vulnerabilities have dominated the interest of the security research community for quite long ti......

Heap vulnerabilities have dominated the interest of the security research community for quite long time due to their potential of finding innovative exploitation ways. Starting in 2001 with the [Vudo Malloc Tricks](http://phrack.org/issues/57/8.html) and [Once Upon A free()](http://phrack.org/issues/57/9.html) followed by the [Advanced Doug Lea’s malloc exploits](http://phrack.org/issues/61/6.html), the [Malloc Maleficarum](https://dl.packetstormsecurity.net/papers/attack/MallocMaleficarum.txt) and so on, the list of heap-type exploitation techniques is long and the concepts seem hard to grasp.

Finding such techniques is often considered a form of dark art and the reader who dives into these areas without a solid background often gets lost, then frustrated …and finally quits. Although that it was tempting to maintain this convoluted-dark-artistic spirit in this post, I finally decided to give a lighter tone, in the context of my repulsion to those people in our area who intentionally try to complicate concepts that can be explained in a more simplistic way. This post will try to shed some light in the heap wilderness and reply to questions that usually you are embarrassed to ask.

Lets start with some basic definitions:

> **Memory allocation:** is the process of setting aside sections of memory in a program to be used to store variables, and instances of structures and classes [1].

There are two types of memory allocation:

> **Static Allocation:**

![](https://miro.medium.com/max/1380/1*3ReuMnV_jmDBl94ROvj9Bg.png)

[https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html](https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html)

**Characteristics:** Allocation is permanent and takes place before the program execution, uses a memory space called **stack,** no memory reusability.

> **Dynamic Allocation:**

![](https://miro.medium.com/max/1400/1*yRYmDdLqYaCxI2TJ_LFDFg.png)

[https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html](https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html)

**Characteristics:** Allocation is happening during runtime and only when the program unit (where the variable declaration takes place) is activated, uses a memory space called **heap,** memory can be reused and freed when not required.

> **The Heap:** A sub-region of a program’s memory space where “plots” can be bind or released dynamically. Each of these plots can be reference using a unique arithmetic address, assigned to special variables called pointers.

![](https://miro.medium.com/max/1400/1*EZOgk5nO--VJkddiZlH-xw.png)

**ptr** is a variable which is saved @**0x006FFA64** and its value is **0x009B54EB** which is a memory address in the heap region.

A real life example
-------------------

Imagine the heap as a huge parking lot where cars are coming and going out. Alice, who works there, has to keep track of the free places to direct the incomers to the right slot. Additionally as some cars may occupy more than one slots, she has to use the available space in a way that maximises the profit.

![](https://miro.medium.com/max/1400/1*_jUfJL6UT_xKCuRbfDUCcQ.png)

It is obvious to assume that maintaining a full record of which slots are allocated or not is a basic requirement to route the cars to the right directions and avoid the so called **fragmentation:**

> **Fragmentation** is an unwanted problem where the memory blocks cannot be allocated to the processes due to their small size and the blocks remain unused. It can also be understood as when the processes are loaded and removed from the memory they create free space or hole in the memory and these small blocks cannot be allocated to new upcoming processes and results in inefficient use of memory [2].

Similarly, heap allocation requires such a record of what memory is allocated and what is not. Additionally a memory allocator will perform maintenance tasks such as defragmenting memory by moving allocated memory around, or garbage collecting — identifying at runtime when memory is no longer in scope and deallocating it.

During the program initialisation the [**mmap**](https://man7.org/linux/man-pages/man2/mmap.2.html) and [**brk**](https://man7.org/linux/man-pages/man2/brk.2.html) system calls are used to set up the program’s memory. The **heap** region is delimited by the **start_brk** and **brk** of the [**mm_struct**](https://github.com/torvalds/linux/blob/master/include/linux/mm_types.h)**:**

![](https://miro.medium.com/max/1400/1*V65JGYUcmzyzGYyjvLLEKQ.png)

*   When [ASLR](http://en.wikipedia.org/wiki/Address_space_layout_randomization) is turned off, start_brk and brk would point to end of data/bss segment ([end_data](http://lxr.free-electrons.com/source/include/linux/mm_types.h?v=3.8#L364)).
*   When ASLR is turned on, start_brk and brk would be equal to end of data/bss segment (end_data) plus random brk offset.

![](https://miro.medium.com/max/1400/1*2KKqMaL1OH-uc4jWugrzCQ.png)

[https://sploitfun.wordpress.com/2015/02/11/syscalls-used-by-malloc/](https://sploitfun.wordpress.com/2015/02/11/syscalls-used-by-malloc/)

The following C library functions can be used by the process to request and release dynamic memory [3]:

*   `malloc(size)`:Requests `size` bytes of dynamic memory; if the allocation succeeds, it returns the linear address of the first memory location.
*   `calloc(n,size)`:Requests an array consisting of `n` elements of size `size`; if the allocation succeeds, it initialises the array components to 0 and returns the linear address of the first element.
*   `free(addr)`:Releases the memory region allocated by `malloc( )` or `calloc( )` that has an initial address of `addr`.
*   `brk(addr)`:Modifies the size of the heap directly; the `addr` parameter specifies the new value of `current->mm->brk`, and the return value is the new ending address of the memory region.
*   `sbrk(incr)`:Similar to `brk( )`, except that the `incr` parameter specifies the increment or decrement of the heap size in bytes.

![](https://miro.medium.com/max/1400/1*f5vovRPhiul2_6WREt35Jg.png)

sbrk, brk in action

Allocating memory in an efficient manner is critical thus many [implementations](https://github.com/emeryberger/Malloc-Implementations/tree/master/allocators) have been used from time to time to tackle this issue:

*   [Doug Lea](https://en.wikipedia.org/wiki/Doug_Lea) has developed the [public domain](https://en.wikipedia.org/wiki/Public_domain) **dlmalloc** (“Doug Lea’s Malloc”) as a general-purpose allocator, starting in 1987.
*   The [GNU C library](https://en.wikipedia.org/wiki/GNU_C_library) (glibc) is derived from Wolfram Gloger’s **ptmalloc** (“pthreads malloc”), a fork of dlmalloc with threading-related improvements.
*   Since [FreeBSD](https://en.wikipedia.org/wiki/FreeBSD) 7.0 and [NetBSD](https://en.wikipedia.org/wiki/NetBSD) 5.0, the old `malloc` implementation (phkmalloc) was replaced by [**jemalloc**](http://jemalloc.net/) (used in Android), written by Jason Evans
*   [OpenBSD](https://en.wikipedia.org/wiki/OpenBSD)’s implementation of the `malloc` function that makes use of [mmap](https://en.wikipedia.org/wiki/Mmap).

Other implementations include hoard malloc, mimalloc, tcmalloc DFWMalloc and so on… In the upcoming articles we are going to focus on the ptmalloc implementations and explore some heap exploitation techniques.

[1] [https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html](https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html)

[2] [https://afteracademy.com/blog/what-is-fragmentation-and-what-are-its-types](https://afteracademy.com/blog/what-is-fragmentation-and-what-are-its-types)

[3] [https://www.oreilly.com/library/view/understanding-the-linux/0596002130/ch08s06.html](https://www.oreilly.com/library/view/understanding-the-linux/0596002130/ch08s06.html)

[4] [https://en.wikipedia.org/wiki/C_dynamic_memory_allocation](https://en.wikipedia.org/wiki/C_dynamic_memory_allocation)

[5] [https://github.com/emeryberger/Malloc-Implementations/tree/master/allocators](https://github.com/emeryberger/Malloc-Implementations/tree/master/allocators)

[https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html](https://www.cs.uah.edu/~rcoleman/Common/C_Reference/MemoryAlloc.html)