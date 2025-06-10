> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287182.htm)

> [原创] 神奇日游保护分析——从 Frida 的启动说起

最近看论坛里很多人在讨论一个日游——jp.gungho.*，特征 so 是 lib__ 6dba__.so。

出于兴趣看了看，发现不是常规的自定义 linker。于是花了三天时间把基本的加固流程和保护原理分析的差不多了，发现其中的 so 完整性检查（可用于 anti frida）和 mmap 模块化调用很有新意，所以整理成文章和大家分享一下。

frida 已经成为逆向爱好者必备的工具，各种 app 对于 frida 的检测也五花八门，甚至还有什么 “某某企业壳 frida 关了也能给检测出来” 这种吓人操作。网上也有大量相关的文章。

可是 frida 启动时做了什么却很少有人分析。通过阅读 frida 源码，可以发现 frida 在启动时留下了大量可供检测的特征。

使用 frida 的第一步是在目标机器上启动 frida-server，即使我们什么 app 都不 hook，只启动一个 server。

frida-server 一启动，就主动注入了 zygote 进程：

![](https://bbs.kanxue.com/upload/attach/202506/872365_RYJAEMTZC2GQXW6.webp)

查看 zygote 进程的内存布局，在启动 frida-server 之前：

![](https://bbs.kanxue.com/upload/attach/202506/872365_MTNM8XYD4DTEM4D.webp)

只启动 frida-server，什么都不 hook：

![](https://bbs.kanxue.com/upload/attach/202506/872365_3VYZ9Z6NCQBJZCG.webp)

可以看到 frida 的 so 已经注入 zygote 了

frida-server 启动注入 zygote 后，立刻自己 hook 了一些 libc 函数

主要有退出相关：exit，_exit，abort

![](https://bbs.kanxue.com/upload/attach/202506/872365_2JFD33G3588REVQ.webp)

fork 相关：

![](https://bbs.kanxue.com/upload/attach/202506/872365_GDMY72TH949CDXS.webp)

和一些文件操作相关的函数。

frida 是使用 inline hook 对函数进行 hook 的，inlinehook 意味着要写内存，所以在 hook 前，frida 会修改目标函数所在页的页属性为 RWX：

![](https://bbs.kanxue.com/upload/attach/202506/872365_KMSH7PNDME9SCDM.webp)

在 maps 文件中，连续内存的不同页属性会被独立列出来：

启动 frida-server 前，zygote maps 文件中 libc.so 的布局：

![](https://bbs.kanxue.com/upload/attach/202506/872365_T45X5K3HS2HV8NN.webp)

启动 frida-server 后：

![](https://bbs.kanxue.com/upload/attach/202506/872365_77T6Y5NMCA724GU.webp)

可以看到 libc.so 的内存中多了很多长度为 0x1000 的 rwx 片段，这些片段正是 frida 自己 hook 那些 libc 基本函数导致的。

*   由于 frida 在退出时并没有还原页属性，所以即使关闭 frida-server，zygote 进程的 libc.so 内存布局仍然遗留有大量 rwx 片段。**这个特性只有机器重启才会还原。**
*   由于 app 都是有 zygote fork 出来的，所以启动 frida-server 之后，**无论是否关闭 frida，新的 app 中 maps 文件的 libc.so 部分也存在大量 rwx 片段。**

这两点就是一些 “企业壳” 所谓在 **frida 就算关了也能检测出来**的基本原理。

打开目标 so，发现只有一个 init 函数，没有 JNI_OnLoad，同时. text 段一大堆 0，很明显是个壳子。

![](https://bbs.kanxue.com/upload/attach/202506/872365_62CBE6WPR7JGS8Q.webp)

于是跟入 init 函数调试看看。

首先通过解密字符串 / proc/self/maps 打开 maps 文件，找到自身 so 的内存地址：

![](https://bbs.kanxue.com/upload/attach/202506/872365_Q6XNVR8U9WYB8NC.webp)

使用的是自实现的 svc 函数 mmap。这个壳没有使用任何 libc 函数，所有的字符串操作：strcpy，strcmp, 内存操作：memcmp 等，都是自己实现，或者直接走 svc 0。

![](https://bbs.kanxue.com/upload/attach/202506/872365_3J62TPMVDCD4FBH.webp)

这无形中增加了逆向的难度。

![](https://bbs.kanxue.com/upload/attach/202506/872365_ZRGYVSXJ9FPM9FS.webp)

不过这个操作没什卵用，更重要的是打开了本地的 lib__ 6dba__.so，然后解析 elf 文件，寻找 type 为 0x80000000 的 section：

![](https://bbs.kanxue.com/upload/attach/202506/872365_T25QD4V9J9QB9HB.webp)

sh_type 为 0x80000000 - 0xffffffff 为用户自定义区间：

![](https://bbs.kanxue.com/upload/attach/202506/872365_ZWA23AYFDUP5SM9.webp)

可以看到这里存放了大量的加密数据：

![](https://bbs.kanxue.com/upload/attach/202506/872365_Z95G3X6YM82TA69.webp)

然后对这段加密数据进行解密，首先解密出元信息，包括模块的大小，初始函数入口偏移等信息：

![](https://bbs.kanxue.com/upload/attach/202506/872365_P2HZ9Q55EKN3TM9.webp)

如 0x257c8 是解密的模块长度，0x5d4 是解密后入口的偏移。

接下来会 mmap 一段 0x257c8 长度的内存，然后将数据解密过去，然后根据偏移跳转到解密出来的代码里执行：

![](https://bbs.kanxue.com/upload/attach/202506/872365_FC6QU4TCXRGQHRD.webp)

注意，这里解密出来的是一个模块而不仅仅是一个函数。里面包含了几十个函数，所以我们需要将其 dump 下来分析。

进入解密的代码块中，首先 mmap 了一块 0x1800 大小的内存，这块内存用来存储后面 mmap 出的模块的信息。

每个模块的基本信息如下：

struct module{  
int id;  
int isrodata;  
void* base;  
int64 size;  
};  
![](https://bbs.kanxue.com/upload/attach/202506/872365_HQSCM9V8X5KKKAQ.webp)

首先插入了两个模块，id 分别是 0xe2 和 0xd0，其中 0xe2 模块的内存基址是 0x709e426000，大小是 0x257c8

![](https://bbs.kanxue.com/upload/attach/202506/872365_EGTMU6J6ECB3EHM.webp)

而 0xe2 模块，其实就是 2.1 中解密出的模块，就是当前模块本身。

这个保护解密出来的模块分为三种类型，最核心的是解释器模块：

解释器模块主要负责循环解密下一个模块。

1. 解释器模块首先检查当前执行的模块 id 是否大于 0xe2, 如果是的话将 id-1 的模块移除（在模块列表里删除，同时将对应的内存清空（memset 为 0）释放（munmap））。

![](https://bbs.kanxue.com/upload/attach/202506/872365_GDBQ652Z3UAHXA6.webp)

这个操作主要的作用是**解释器替换。**

因为解释器模块可能 mmap 解密出新的解释器模块，这样执行新的解释器模块后，会将上一个解释器模块释放掉，用新的解释器模块来解密。

2. 循环解密新的模块

![](https://bbs.kanxue.com/upload/attach/202506/872365_U25QM5PXQNK5U3A.webp)

3. 如果解密出来的模块有初始化函数，调用初始化函数。

主要有两个初始化函数

*   第一个用来将模块列表传递给新的模块。
*   第二个为新模块的逻辑入口。

![](https://bbs.kanxue.com/upload/attach/202506/872365_EBEBKZRNR5PCGXB.webp)

新的模块入口函数执行后，如果返回 1，继续解密下一个模块执行。如果返回 0，会跳出循环，然后清除之前的所有模块，然后调用 svc exit 退出。

![](https://bbs.kanxue.com/upload/attach/202506/872365_FYE67A8P8MQRP9C.webp)

注意，解释器模块是不会返回的，因为上一个解释器已经被清掉了，返回会直接 crash。

而逻辑模块则必须返回 1。如果逻辑模块返回 0，则必然会 svc exit 退出。

逻辑模块主要执行不同的逻辑，有的是安全模块，检查各种环境信息。有的是解密模块。

例如：

*   0x20 模块是内存 hash 检查，如果成功返回 1，执行下一个反调试模块 0x54。如果检查不通过，则会返回 0，然后解释器模块 svc exit。
*   0x9b 模块负责解密还原真实的 so。（注意所有模块都只是壳的一部分）
*   0x98 模块调用原始 so 的 init array。

没有入口函数，解密出来扔在模块列表里，供其他模块使用。

![](https://bbs.kanxue.com/upload/attach/202506/872365_8VYEKVZK887KEVT.webp)

第一个模块 0xe2 由原始 so 的 init 函数解密出来并执行。

然后 0xe2 模块解密了

*   0x8f,0x8e 两个模块。
*   0xe3 模块（解释器）

0xe3 模块解密出了

*   0xe4 模块（解释器）

0xe4 模块解密了

*   0x60 模块（root 检查）
*   0x40 模块（模拟器检测）
*   0x20 模块（hash 校验和 libc 检查）
*   0x54 模块（反调试检查）
*   0xe5 模块（解释器）

0xe5 解密出了

*   0xA4 模块，fork 出了一个子进程。
*   0xe6 模块（解释器）

0xe6 解密出了

*   0x72 模块，意义不明，不是安全模块，不影响
*   0xe7 模块（解释器）

0xe7 模块解密出了

*   0x9b 模块，主要使用多线程进行原始 so 的代码段解密和回填。
*   0xe8 模块（解释器）

0xe8 模块：

解密出了 0x98 模块，该模块执行原始 so 的 init_array 函数。然后返回到 linker 中。

当然还有其他一些模块，这里只列举出比较重要的，一共大概有 20 多个模块。

为什么返回到 linker 中了？因为我们从 init 函数来的，最终所有模块执行完了（如果都成功的话），会返回到 linker 掉用 init 函数的地方。

*   以上所有模块都是 mmap 在内存中，直接调用入口函数执行。如果没有研究清楚，就会觉得在无限次 mmap。
    
*   模块在执行中解释器从 0xe2-0xe3-0xe4-0xe5-0xe6-0xe7-0xe8，一共换了 7 次，所以显得非常复杂。但其实基本逻辑是相同的，研究清楚了很好跟进。
    
*   对于每个模块，只有 plt 函数和代码段，没有 elf 头，动态链接信息，section header 等。所以 dump 下来后需要自己将 dump 下来的代码自己修复为一个 so，然后用 ida 打开分析。否则所有代码在内存中，并且没有符号，很难分析。
    
*   每个模块都自己实现了一套 libc 基本函数，svc 调用和字符串解密函数，所以 hook libc 函数没什么作用。
    

这个壳的安全部分就是

*   0x60 模块（root 检查）
    
*   0x40 模块（模拟器检测）
    
*   0x20 模块（hash 校验和 libc 检查）
    
*   0x54 模块（反调试检查）
    

这四个模块。

在 /proc/mount，/proc/pid/mounts 文件中检查 magisk，同时检查一些 su 文件和路径（/system/bin/su 之类）

主要使用了自定义的 strcmp 函数检查字符串，svc 调用 newfstatat 检查文件是否存在。

![](https://bbs.kanxue.com/upload/attach/202506/872365_R9X5M6YXN2SWJWA.webp)

（在调试过程中解密出的字符串被我改了，为了绕过检测）

主要检查模拟器，通过检测文件路径和包名来判断是否存在对应的模拟器。

其中包名检查主要是构造 / data/user/0 / 包名路径，然后 svc 调用 newfstatat 检查。

![](https://bbs.kanxue.com/upload/attach/202506/872365_9M9PYMTK67DR8MJ.webp)

检查的包：

![](https://bbs.kanxue.com/upload/attach/202506/872365_WKQCBWABC8HHP94.webp)

![](https://bbs.kanxue.com/upload/attach/202506/872365_XD4WAPJBDXPQUC8.webp)

检查的路径：

![](https://bbs.kanxue.com/upload/attach/202506/872365_WUKWUN3XBEUQ8X6.webp)

![](https://bbs.kanxue.com/upload/attach/202506/872365_GQ9NXBMMT5SH4SN.webp)

![](https://bbs.kanxue.com/upload/attach/202506/872365_7N3U6APT6MHD3T5.webp)

包括使用__system_property_get() 函数，获取一些系统信息：

![](https://bbs.kanxue.com/upload/attach/202506/872365_4Q8VT6DGT24JJ66.webp)

包括检查 ro.hardware，检查是否为 goldfish，或者 ranchu

![](https://bbs.kanxue.com/upload/attach/202506/872365_QSMT6Z2VMXDCCPT.webp)

![](https://bbs.kanxue.com/upload/attach/202506/872365_678TRZSC83NNZ9D.webp)

![](https://bbs.kanxue.com/upload/attach/202506/872365_BJKVYEC2BM3XXSQ.webp)

主要检查 so 库的 hash 和 libc

首先通过解密函数解密 / assets/6dba/data1.dat 文件，解密出：

![](https://bbs.kanxue.com/upload/attach/202506/872365_3HP2K8PXPXNUT3Y.webp)

其中分别包含所有 so 的

*   文件大小
*   文件 hash
*   .text 段偏移
*   .text 段大小
*   .text 段 hash
*   so 名

如上图红框所示，分别是 libopenal.so 的文件大小：0x14e518，文件 hash：0xbc53ce75 ，.text 偏移：0x7e30

, .text 大小：0x1b7c8, .text 段 hash：0x176a9502

该模块会打开本地对应的 so，检查文件的 hash。如果当前模块是对应的 so，同时会进行. text 段的 hash 校验。

hash 函数为自定义的，不是常见的 crc32：

![](https://bbs.kanxue.com/upload/attach/202506/872365_RCPZC5XB8CA55WJ.webp)

该加固对 libc 检查的方式非常奇妙，这就用到了开头说的 frida 启动特征。

正常的 libc 在 maps 里的条目是比较少的，通常小于 10 个：

![](https://bbs.kanxue.com/upload/attach/202506/872365_E6BPXVQHG4CUH5K.webp)

但是启动过 frida 的机器，或被 frida hook 的 app，libc 的 maps 里有很多 rwx 碎片：

![](https://bbs.kanxue.com/upload/attach/202506/872365_72VNEBTQBJGY2MX.webp)

该模块首先打开 maps 文件，查找起始地址大于 libc .text 起始，并小于 libc .text 终止的条目个数。

如果大于 9 个，则认为 libc"不正常"，然后返回 0（解释器 svc exit 退出）

![](https://bbs.kanxue.com/upload/attach/202506/872365_GGGWBH5H5FUA6YS.webp)

如果 maps 里条目数正常，该模块会映射一份本地 libc，和内存中的 libc 的代码段用自定义的 memcmp 函数比较：

![](https://bbs.kanxue.com/upload/attach/202506/872365_BF7MCH9S2E673PC.webp)

通常来讲都会比较 libc 代码段的 crc32，但是其实直接 memcmp 也没什么问题。只要内存不一致就说明 libc 被修改了，没必要算 crc.

反调试模块比较常规，就是打开 / proc/pid/status 检查 TracerPid。

![](https://bbs.kanxue.com/upload/attach/202506/872365_NJUZ82A7GPSM6F7.webp)

同时还通过 libart.so 找到了 SetJdwpAllowed 函数地址：

![](https://bbs.kanxue.com/upload/attach/202506/872365_UXHQJNYRCNGFF22.webp)

然后执行了 SetJdwpAllowed(false)，似乎没什么用。

使用多线程解密原始 so：

![](https://bbs.kanxue.com/upload/attach/202506/872365_HMC8ZADA3NEJZ2Q.webp)

解密之后 dump 原始 so 的 rx 段回填，就可以看到原始 so 的代码了。

![](https://bbs.kanxue.com/upload/attach/202506/872365_NYQTY852MF25DSW.webp)

还缺重定位表和 init_array 信息，这两个拿到了就能修复 so 了，关于自定义 linker 部分这里就不细究了。但是应该在 0x9b 模块里是能拿到的。

这个加固将 app 里的每一个 so 都加壳了，每个 so 都要经过这样的模块化层层处理后，最终才会解密执行。

不同的是，除了 lib__ 6dba__.so 自身以外，其他的 so 在解密过程中只执行了 0x20 模块进行 hash 校验和 libc 检查，没有执行其他安全模块。

至于专门针对 frida 的检测，并没有看到。但是只要检查 libc 在 maps 中的个数，就可以完全对抗 frida 了。甚至 frida 启动过一次关了都能检查出来，必须重启机器才行。

对于模块化保护，需要熟悉 elf 结构，自定义 libc 函数，svc 调用，无头无动态链接纯代码还原成 so 静态分析，自定义 linker 等，分析起来难度还是相当大的。

当然，最重要的是耐心，他的模块化执行路径我也是调试了几十次才最终搞清楚。。。

总的来说这是一款有难度的安卓保护，mmap 模块化匿名执行和 maps 检查 antiFrida 都是很新的思路。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 8 小时前 被乐子人编辑 ，原因： 排版