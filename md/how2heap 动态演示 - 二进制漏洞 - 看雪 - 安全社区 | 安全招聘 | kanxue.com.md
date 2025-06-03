> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287061.htm)

> how2heap 动态演示

how2heap 动态演示

发表于: 7 小时前 44

[举报](javascript:void(0);)

### how2heap 动态演示

 [![](http://passport.kanxue.com/upload/avatar/036/815036.png?1531462774)](user-home-815036.htm) [jmpcall](user-home-815036.htm) ![](https://bbs.kanxue.com/view/img/rank/11.png) 3  ![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 7 小时前  44

本文演示了 [how2heap 仓库](elink@b80K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6K6K9r3g2D9L8s2m8Z5K9i4y4Z5i4K6u0r3K9r3!0%4x3X3S2W2j5i4l9`.)里面各个程序的执行流程。

学习 how2heap 之前，需要先熟悉 ptmalloc 逻辑，推荐先看一下《Glibc 内存管理 --ptmalloc2 源代码分析》，另外有部分概念，我在[深度理解 glibc 内存分配](https://bbs.kanxue.com/thread-271331.htm)这篇文章，更加细节的介绍过。

演示程序仅仅用于描述攻击的基本步骤，其中执行的每一步操作，在实际攻击中，都要通过漏洞程序的交互（本地 / 远程）逻辑触发:

(1) 交互可以访问并显示的范围通常有限，甚至依赖信息泄漏漏洞；

(2) 交互可以向程序写入的范围通常有限，甚至依赖写溢出漏洞；

(3) 交互通常不能轻易控制分配、释放的时机，以及分配内存的大小。

不过，只有透彻理解最基本的原型，才能将极其复杂的真实场景，转换成它们。

就跟下棋一样，不懂招数，看到的整盘棋都是零散的，反之每个阶段围绕一个战略目标，就可以分而治之，只需要应付每个较小范围里的变化。

1. 链表拦截
=======

1.1. unsorted_bin_attack (glibc-2.27)
-------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_DXZ5AJD8P2WFF4E.webp)

过程说明：

将 chunk(p) 释放到 unsorted_bin，然后利用漏洞 (比如 UAF 漏洞，因为 chunk(p) 当前为释放状态)，修改 chunk(p)->bk，使其指向 fake_chunk，就将 fake_chunk 也链入 unsorted bin 了。

细节: 

chunk(p) 被 malloc(0x410) 分配到了:

(1) chunk(p) 没有移入其它 bin，所以 chunk(p)->fd 保持不变；

(2) 停止遍历 unsorted_bin，从而 fake_chunk 仍然链在 unsorted_bin，并且 fake_chunk->fd，指向了 unsorted_bin，unsorted_bin->fd，仍然指向 chunk(p)。

     ![](https://bbs.kanxue.com/upload/attach/202506/815036_GXRSV4TU5TJUGD2.webp)

1.2. unsorted_bin_into_stack (glibc-2.23)
-----------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_S4H3YA63SW384Y9.webp)

过程说明：

相比 unsorted_bin_attack.c，在执行 34 行之前，将 chunk(victim)->size 改掉了，使其不满足 malloc(0x100) 的要求，使其移入 smallbin[0]，从而继续往后遍历，最终分配到 fake_chunk。

作用:

(1) 使目标位置可以被 malloc() 分配到；

(2) 可以在目标位置，写入 unsorted_bin 地址。

1.3. house_of_lore (glibc-2.34)
-------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_JB2RHKYETES6VMH.webp)

过程说明：

将 victim_chunk 释放到 smallbin，然后修改 victim_chunk->bk，使其指向 fake_chunkA，就将 fake_chunkA->fake_chunkB->...->fake_chunk6，链入 smallbin 了。

细节：

glibc-2.26 开始，新增了 tcache 功能，相比 unsorted_bin_attack，第 120 行执行 malloc(0x100)，分配到所需 chunk 后，会继续遍历 smallbin，将后续 chunk 移入 tcache，直到 tcache 填满为止，第 123 行 malloc(0x100)，优先从 tcache 分配到 fake_chunk4。  

1.4. large_bin_attack (glibc-2.34)
----------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_FGPEY5GN4AFBKSW.webp)

过程说明：

将 chunk(p1) 释放到 largebin，然后修改 chunk(p1)->bk，使其指向 fake_chunk（&target-4），就将 fake_chunk，链入 largebin 了。再触发 chunk(p2) 移入 largebin，添加到它们之间时（largebin 中的 chunk 会按大小排序），fake_chunk->fd_nextsize，就会被写为 chunk(p2)。

glibc 相关代码：

![](https://bbs.kanxue.com/upload/attach/202506/815036_SRJV4J7HH3PFHDG.webp)

1.5. tcache_poisoning (glibc-2.34)
----------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_B67GSZG6MG24EC3.webp)

过程说明：

将 chunk(b) 释放到 tcache，然后修改 chunk(b)->fd，使其指向 fake_chunk，就将 fake_chunk，链入 tcache 了。

细节：

(1) tcache 中链接的是 tcache_entry 对象，根据 & chunk->fd 位置强转得到，所以 entry->next 与 chunk->fd 位置对应。

    ![](https://bbs.kanxue.com/upload/attach/202506/815036_ZS8KNV8R9JDWFE7.webp)

(2) glibc-2.32 开始，新增了加密指针功能，tcache 以及 fastbin 链表中，chunk->fd 不再直接指向下一个节点的地址，而是 chunk->fd>>12 ^ 下个节点地址。  

1.6. fastbin_dup_into_stack (glibc-2.23)
----------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_K9XF3V7GE69WFUV.webp)

过程说明：

将 chunk(a) 释放到 fastbin，然后修改 chunk(a)->fd，使其指向 fake_chunk，就将 fake_chunk，链入 fastbin 了。

细节：

演示程序为了更接近真实的攻击过程，事先模拟了 double free 场景。

向 fastbin 添加 chunk 之前，只会检查链表头是否为要添加的 chunk，不会检查整个链表（否则会影响 free() 的效率），所以在两次 free(a) 之间，执行一次 free(b)，就可以绕过 glibc 检测。

![](https://bbs.kanxue.com/upload/attach/202506/815036_GP8VXNA7PB6R6UT.webp)

1.7. house_of_storm (glibc-2.27)
--------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_393ZRS2DEUFCXYA.webp)

先看 138 行之后的执行过程：

(1) 142~154: 填满 tcache[21] (个人认为这一步没什么必要，因为后面没有需要避免 chunk 释放进 tcache[21] 的地方)；

(2) 159~169: 将 large_chunk，释放到 largebin[2]，将 unsorted_chunk，释放到 unsorted_bin；

(3) 204:           修改 unsorted_chunk->bk，将 fake_chunk 链入 unsorted_bin，其实就是利用 unsorted_bin_attack，用于最后执行 calloc() 时，可以分配到 fake_chunk；

(4) 207, 245: 修改 large_chunk->bk、large_chunk->bk_nextsize，其实就是利用 large_bin_attack，使得 unsorted_chunk 移入 large_bin[2] 时，添加到 large_chunk 和 fake_chunk 之间，使得 unsorted_chunk->bk_nextsize->fd_nextsize，指向 unsorted_chunk；

(5) 278:           执行 calloc(alloc_size)，从 unsorted_bin 分配 fake_chunk。

malloc() 优先从 tcache 分配，但 calloc() 不会：

![](https://bbs.kanxue.com/upload/attach/202506/815036_J5WGSQZVQBFSPTC.webp)![](https://bbs.kanxue.com/upload/attach/202506/815036_H4JUY5X3U676WX9.webp)

回头再头 138 行之前的执行过程：

73~138:  为 large_chunk->bk_nextsize 指向，计算微调偏移，使得 0x173，正好写入 & fake_chunk->size 处，保证 fake_chunk->size 满足 calloc(alloc_size) 的分配大小。

2. tcache & fastbin，知多少
=======================

2.1. decrypt_safe_linking (glibc-2.34)
--------------------------------------

glibc-2.32 开始，tcache 以及 fastbin 链表中，chunk->fd 不再直接指向下一个节点的地址，而是 chunk->fd>>12 ^ 下个节点地址，而如果 node 和 nextnode 在同一个 4K page，仅通过加密指针本身，就可以还原出 nextnode 地址。

![](https://bbs.kanxue.com/upload/attach/202506/815036_ERVCMH4ARKUFXCF.webp)

证明：  

node:          A15,A14,A13,A12,A11,A10,A9,A8,A7,A6,A5,A4,A3,A2,A1,A0

nextnode: B15,B14,B13,B12,B11,B10,B9,B8,B7,B6,B5,B4,B3,B2,B1,B0

加密指针：C15,C14,C13,C12,C11,C10,C9,C8,C7,C6,C5,C4,C3,C2,C1,C0

由于 node 和 nextnode 只有低 12 位不相等，所以 node 也可以表示为：

node:          B15,B14,B13,B12,B11,B10,B9,B8,B7,B6,B5,B4,B3,A2,A1,A0

由于，加密指针 = node>>12 ^ nextnode

=> (1) B15,B14,B13 = C15,C14,C13

=> (2) C12,C11,C10 = B15,B14,B13 ^ B12,B11,B10 => B12,B11,B10 = B15,B14,B13 ^ C12,C11,C10，同理可求：B9,B8,B7、B6,B5,B4、B3,B2,B1、B0。

2.2. tcache_stashing_unlink_attack (glibc-2.34)
-----------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_WFMM3GG2W6CND6A.webp)

过程说明：

先触发 chunk 从 smallbin 移入 tcache，再从 tcache 分配这个 chunk，就可以绕过 chunk->bk->fd == chunk 检查。

原因：

(1) 从 tcache 分配 chunk，不检查 chunk->bk->fd == chunk；

(2) 从 smallbin 分配 chunk，会取出首个 chunk，并检查 chunk->bk->fd == chunk，而如果 tcache 未满，会将剩余 chunk 移入 tcache，并且不做检查。

![](https://bbs.kanxue.com/upload/attach/202506/815036_MWUU2BVDVR435VK.webp)

2.3. fastbin_reverse_into_tcache (glibc-2.34)
---------------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_BJYGJFT2WJFET4X.webp)

fastbin FILO，tcache FILO，fastbin 中的 chunks 移入 tcache 后，顺序会颠倒:

最先从 fastbin 移出的 chunk，也会最先移入 tcache，那么在 tcache 中，就会最后才能被移出。

2.4. fastbin_dup (glibc-2.34)
-----------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_XMK45WK7D45WYSU.webp)

fastbin_dup 用于绕过 double free 检测，fastbin_dup_into_stack，就已经展示过。

2.5. fastbin_dup_consolidate (glibc-2.23)
-----------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_MBMMGRMC5BP4XKM.webp)

也是用于绕过 double free 检测，原理都是先将相同的 chunk，挤出 fastbin 链表头，只不过这里是挤到 unsorted_bin。

3. 扩大控制范围
=========

3.1. house_of_spirit (glibc-2.23)
---------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_G2AWFVGJ8YDMEMD.webp)

作用：

真实攻击场景，如果目标位置不能直接控制写入，但其周围内容可控，并且周围地址的释放、分配也可控，就可以尝试将目标位置分配给程序后，再通过交互控制它的内容。

细节：

释放 fake_chunk 前，先要设置地址相邻的 next_chunk->size，保证它属于 (2*SIZE_SZ, av->system_mem) 区间，绕过 free()内部的检查。

![](https://bbs.kanxue.com/upload/attach/202506/815036_TTPZXA99A39MEY6.webp)

3.2. tcache_house_of_spirit (glibc-2.34)
----------------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_J8RUTWKGWH3J3AB.webp)

高版本 glibc，增加了 tcache 功能，优先级高于 fastbin，并且 chunk 释放到 tcache，不用检查 next_chunk->size。

4. 修改 chunk->size
=================

4.1. prev_chunk 错位
------------------

### 4.1.1. unsafe_unlink (glibc-2.34)

![](https://bbs.kanxue.com/upload/attach/202506/815036_NFJ73ZBYFTMY8Z3.webp)

过程说明：

1. 在 chunk0_ptr 位置，构造一个 fake_chunk，并保证：

    fake_chunk->fd->bk = &chunk0_ptr

    fake_chunk->bk->fd = &chunk0_ptr

    使得 unlink(fake_chunk) 时，不会向一个不可写的位置写入内容（unlink() 最终会在 & chunk0_ptr 处，写入 & chunk0_ptr-3）。

2. 利用漏洞，向 chunk1_ptr->prev_size 处，写入 0x420

    free(chunk1_ptr) 时，就会认为 fake_chunk 是它地址相邻的 prev_chunk，并且状态为 free，从而触发合并，而合并前先会 unlink(fake_chunk)。

### 4.1.2. poison_null_byte (glibc-2.34)

![](https://bbs.kanxue.com/upload/attach/202506/815036_7MAYK34SZGDBGXD.webp)

过程说明：

(1) 通过控制分配、释放操作，间接实现内容控制，设置了 chunk(prev)->fd_nextsize、bk_nextsize，以及 chunk(b)->fd；

(2) 剩余部分同 unsafe_unlink。

padding 计算说明：

![](https://bbs.kanxue.com/upload/attach/202506/815036_VGG6KRQAAFQTTXX.webp)

![](https://bbs.kanxue.com/upload/attach/202506/815036_FRZCJZM2GE6JHZ6.webp)

4.2. next_chunk 错位
------------------

### 4.2.1. overlapping_chunks (glibc-2.34)

![](https://bbs.kanxue.com/upload/attach/202506/815036_67UJR8NESAMQ3B5.webp)

过程说明：

利用漏洞，将 chunk(p2)->size 改大，然后释放 chunk(p2)，用改大后的值分配，就可以通过分配状态的 chunk(p4)，向 free 状态的 chunk(p3) 区域写入数据。

### 4.2.2. overlapping_chunks_2 (glibc-2.23)

![](https://bbs.kanxue.com/upload/attach/202506/815036_YNG6PMMQN9ZMDD6.webp)

过程说明：

利用漏洞，将 chunk(p2)->size 改大，使 chunk(p4) 成为它地址相邻的 next_chunk，此时分别释放 chunk(p4)、chunk(p2)，就会触发它们合并。

### 4.2.3. house_of_botcake (glibc-2.34)

![](https://bbs.kanxue.com/upload/attach/202506/815036_2UXYQCJSBEVY9W3.webp)

chunk(a) 一方面已经释放，与 chunk(prev) 合并，另一方面，仍为分配状态被程序使用，等效于 UAF 漏洞。

### 4.2.4. house_of_einherjar (glibc-2.34)

house_of_botcake + tcache_poisoning：

![](https://bbs.kanxue.com/upload/attach/202506/815036_XNRV5N7QSE26NEV.webp)

4.3. 缩小 top_chunk->size
-----------------------

### 4.3.1. house_of_orange (glibc-2.23)

![](https://bbs.kanxue.com/upload/attach/202506/815036_37D5AHYWCGFAAFU.webp)

过程说明：

75,76:       利用漏洞，将 top_chunk->size 改为较小的值；

122:           再去分配一个大于 top_chunk->size，小于 32M 的区间（避免直接使用 mmap()），就会触发 top_chunk 扩展（brk = (char*)(MORECORE(size))）。另外，随着 top_chunk->size 的改小，old_end 也会变小，就会误导 malloc() 执行_int_free(old_top)，将 old_top 释放到 unsorted_bin（通常在扩展非主分配区的 top_chunk 时，才会这样）；

![](https://bbs.kanxue.com/upload/attach/202506/815036_W55J9GH4G7QJF6E.webp)

![](https://bbs.kanxue.com/upload/attach/202506/815036_T36756DU7YT4XUB.webp)

158:           此时，由于 top->fd = unsorted_bin，给它加上 0x9a8 偏移，就可以得到_IO_list_all 变量地址；

![](https://bbs.kanxue.com/upload/attach/202506/815036_QDC44MN2H7CUT2W.webp)

175:           修改 top->bk，准备 unsorted_bin_attack；

182:           设置 system() 参数；

214:           设置 top_chunk->size = 0x61，使 top_chunk 从 unsorted_bin 移出后，移入的是 smallbin[4]；

225~153: 伪造_IO_FILE_plus 对象；

![](https://bbs.kanxue.com/upload/attach/202506/815036_PZZ2RYMEAPMYFN8.webp)

![](https://bbs.kanxue.com/upload/attach/202506/815036_YF6FK6X2XA29KSB.webp)

257:           从 unsorted_bin 遍历到 fake_chunk 时，发现其 size<MINSIZE，触发 abort()，进一步触发_IO_flush_all_lockp()。此时前面构造的 fp，已经位于_IO_list_all 链表，并且符合__overflow() 执行条件，而__overflow 已经在前面被设置为 winner()，winner() 内部又执行了 system() 函数，并且将 fp 指向的 "/bin/bash" 传参给它。

![](https://bbs.kanxue.com/upload/attach/202506/815036_8PUJXD93ZVS7NKC.webp)

![](https://bbs.kanxue.com/upload/attach/202506/815036_EY8V3642SKENS8K.webp)

### 4.3.2. house_of_tangerine (glibc-2.39)

![](https://bbs.kanxue.com/upload/attach/202506/815036_82V6HYTS9W87YCA.webp)

过程说明：

(1) 和 house_of_orange 一样，也是触发 top_chunk 释放，不过是释放到 tcache，并且释放了 2 次，为了后续满足 tcache count 检查；

![](https://bbs.kanxue.com/upload/attach/202506/815036_6QY74ZT495MPQB4.webp)

(2) 再通过 tcache_poisoning，将 fake_chunk 添加到 tcache，后续就可以分配到了。

4.4. house_of_force (glibc-2.27)
--------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_2UKAAYYFNF4R8RF.webp)

利用漏洞，改写 top_chunk->size 为最大值，然后触发一次超级大的内存分配，就可以让 top_chunk"绕一圈"，退回到低地址位置。如果再能精确设计分配大小，就可以控制 top_chunk 正好落在 bss_var 变量所在位置，后续就可以把该位置，分配给程序。

5. 偷梁换柱  

==========

5.1. arena 转移  

----------------

### 5.1.1. house_of_gods (glibc-2.24)

详细介绍：[https://github.com/Milo-D/house-of-gods/blob/master/rev2/HOUSE_OF_GODS.TXT](elink@020K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6y4K9h3I4G2i4K6u0V1c8q4)9J5c8X3S2G2N6i4y4W2i4K6u0V1L8$3k6Q4x3X3c8Y4L8$3c8K6i4K6u0r3j5X3I4G2j5W2)9J5c8X3#2S2M7%4c8W2M7W2)9J5c8Y4u0W2N6U0u0Q4x3V1k6t1e0#2g2e0c8g2)9#2k6V1!0r3i4K6g2X3c8@1!0p5f1#2)9J5k6g2c8j5g2l9`.`.)

![](https://bbs.kanxue.com/upload/attach/202506/815036_DEBSKU3J25QQQPP.webp)

112,119:  将 SC 释放到 unsorted_bin，获取 unsorted_bin 地址；

130:           触发 SC 移入 smallbin[7]，使 binmap 第 9 位置 1（binat(9)），即 0x200，后续作为 binmap_chunk->size。另外，单线程程序，只有一个 arena，所以 main_arena 的 next 指向自己，后续作为 binmap_chunk->bk；

155~171: 再利用 unsorted_bin_attack，将 binmap_chunk 添加到 unsorted_bin（binmap_chunk 相对 unsorted_bin，偏移 0x7f8）；

199:          设置 FAST40_CHUNK->bk = INIM_CHUNK；

204,209:  &main_arena 为 binmap_chunk->bk，设置其 size、bk（2*SIZE_SZ <size <= av->system_mem）；

223:           遍历 unsorted_bin，直到取出 binmap_chunk；

272:           将 & narenas-0x10 视为 chunk，链入 unsortd_bin；

281:           设置 main_arena->system_mem = 0xffffffffffffffff；

296:           遍历 unsorted_bin，直到取出 INTM_CHUNK，使得 unsorted_bin 地址，写入 narenas 变量，使其大于 1；

314:           binmap_chunk 已经分配给程序了，可以利用交互，在其 bk 位置（同时也是 main_arena->next 位置），写入 fake_arena 地址；

324, 331:  触发 2 次 reused_arena()，使 thread_arena = fake_arena；

341~361: 后续的分配，就会使用 thread_arena 指向的假冒分配区。

### 5.1.2. house_of_mind_fastbin (glibc-2.34)

建议先了解一下，非主分配区相关结构：[https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/](elink@6b2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6K6M7r3I4G2K9i4c8X3N6h3&6Q4x3X3g2%4L8%4u0V1M7s2u0W2M7%4y4Q4x3X3g2U0L8$3#2Q4x3V1j5J5x3o6p5#2i4K6u0r3x3o6u0Q4x3V1j5I4x3q4)9J5c8Y4g2F1k6r3g2J5M7%4c8S2L8X3c8A6L8X3N6Q4x3X3c8Y4L8r3W2T1j5#2)9J5k6r3#2S2L8r3I4G2j5#2)9J5c8R3`.`.)

![](https://bbs.kanxue.com/upload/attach/202506/815036_Q2JPZXNGFSZ8A8N.webp)

过程说明：

122~144: 构造 fake_arena、fake_heap_info 对象；

146~162: 在高于 fake_heap_info，且离它不远的位置，分配 fastbin_chunk；

167~173: 填满 tcache，使后续释放 fastbin_chunk 时，移入 fastbinsY[4]；

191:          设置 fake_heap_info->arena；

208,230: 修改 chunk_ptr->size 标志位，欺骗 libc，认为 chunk_ptr 是非主分配区分配的，就会向下 64M 对齐找 sub_heap，进一步找到所属 arena，最终释放到 fake_arena 的 fastbin。

5.2. house_of_roman (glibc-2.23)
--------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_6J68P5NUTZ372PX.webp)

过程说明：

122~143: 构造一个 size=0x71，并且 fd 指向 bins 区域的 chunk（先满足 fd=&smallbin[7]，再通过切割，满足 size=0x71，因为直接分配、释放 0x70 大小的 chunk，会链入 fastbin 链表，不能控制 fd 指向 bins 区域）；

149~201: victim_chunk->fd = main_arena_chunk (top_chunk 起始位置 + 0x70+0x90)

246~252: 控制 main_arena_chunk->fd 指向，使其 size=0x7f（__memalign_hook 变量下方是 severity_list 变量，值为 0x00007f...，并且当中空余 8 字节，值为 0）；

![](https://bbs.kanxue.com/upload/attach/202506/815036_JWA9WHD63XX632S.webp)

![](https://bbs.kanxue.com/upload/attach/202506/815036_58H6FGU8W5URXZP.webp)

260~266: 将__malloc_hook_adjust 指向的 chunk 分配给程序，用于后续控制__malloc_hook 的值；

312~335: 利用 unsorted_bin_attack，控制__malloc_hook = unsorted_bin；

357~383: 通过交互，覆盖__malloc_hook 低字节，将其修改为 system() 函数地址；

392:           执行 malloc()，触发__malloc_hook，即 system() 执行。

5.3. house_of_water (glibc-2.36)
--------------------------------

![](https://bbs.kanxue.com/upload/attach/202506/815036_8JQSX26TDSW2VJY.webp)

过程说明：

58:             程序首次执行 malloc()，会触发 init()，先分配一个 tcache；

![](https://bbs.kanxue.com/upload/attach/202506/815036_8A8JHKQJW3JA9WT.webp)

63:             free(fake_size_lsb)，会将 fake_size_lsb 添加到 tcache->entries[60]，且 tcache->counts[60] ++；

64:             free(fake_size_msb)，会将 fake_size_lsb 添加到 tcache->entries[61]，且 tcache->counts[61] ++；

68:             由于 tcache 是从 top_chunk 起始位置分配的，按页对齐，所以 fake_size_lsb 低 12 位清零，就会到达 tcache 位置；

211, 225: 再将 chunk(unsorted_start) 释放到 tcache->entries[1]（释放过程中，会将 fd 位置设置为 tcache_key，所以需要恢复）；

240~253: 再将 chunk(unsorted_end) 释放到 tcache->entries[0]（释放过程中，会将 fd 位置设置为 tcache_key，所以需要恢复）。此时，就伪造出来一个 fake_chunk，并且 fd、bk 都已设置妥当，以及满足：2*SIZE_SZ < fake_chunk->size <= av->system_mem；

267~343: 再将 start、middle、end 释放到 unsorted bin，修改 start->fd、end->bk，即可将 middle 替换为 fake_chunk，后续可以分配给程序。

  

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#漏洞利用](forum-150-1-154.htm) [#缓冲区溢出](forum-150-1-156.htm) [#UAF](forum-150-1-158.htm) [#Linux](forum-150-1-161.htm)