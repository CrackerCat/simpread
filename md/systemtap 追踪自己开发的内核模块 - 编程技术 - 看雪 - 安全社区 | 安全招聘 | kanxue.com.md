> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280023.htm)

> systemtap 追踪自己开发的内核模块

systemtap 追踪自己开发的内核模块

41 分钟前 41

### systemtap 追踪自己开发的内核模块

 [![](http://passport.kanxue.com/upload/avatar/036/815036.png?1531462774)](user-home-815036.htm) [jmpcall](user-home-815036.htm) ![](https://bbs.kanxue.com/view/img/rank/11.png) 3  ![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/sun.gif) 41 分钟前  41

**1. 背景**
---------

    看了[动态追踪技术漫谈](https://blog.openresty.com.cn/cn/dynamic-tracing/)这篇文章之后，就想着拿 systemtap 追踪一下自己开发的内核模块，然而，[systemtap 官方文档](https://sourceware.org/systemtap/documentation.html)，对如何追踪内核模块的描述中，只是拿内核源码树中的驱动举例，比如：probe module("ext3").function("*") { }，实际验证确实也很顺利_（我的系统使用的是 xfs 文件驱动，stap 命令执行后，随便 vi 一个文件，就能看到 xfs_iread() 函数被调用了，并列出了参数信息）_：

    ![](https://bbs.kanxue.com/upload/tmp/815036_AW38E6WH7J48SG5.webp)

    然后，我写了一个最简单的驱动，test.c：

```
#include  void test(int n)
{
    printk("%s(), %d: %d\n", __FUNCTION__, __LINE__, n);
}
 
static int __init test_init(void)
{
    test(100);
    printk("%s(), %d\n", __FUNCTION__, __LINE__);
    return 0;
}
 
static void __exit test_exit(void)
{
    test(100);
    printk("%s(), %d\n", __FUNCTION__, __LINE__);
}
 
 
 
module_init(test_init);
module_exit(test_exit);
 
MODULE_LICENSE("GPL"); 
```

    Makefile：

```
ifneq ($(KERNELRELEASE),)
    obj-m := test.o
else
    KERNELDIR ?= /lib/modules/$(shell uname -r)/build
    PWD := $(shell pwd)
 
default:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) modules
clean:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) clean
endif

```

    以及 x.stap：

```
#!/usr/bin/env stap
 
 
probe begin {
  printf("begin\n");
}
 
probe module("test").function("*").call {
  printf("%s -> %s\n", thread_indent(1), probefunc())
}
 
probe module("test").function("*").return {
  printf("%s <- %s\n", thread_indent(-1), probefunc())
}

```

    接下来，编译 test.c-> 加载 test.ko-> 执行 x.stap_（此刻，我还在一个思维陷阱里，以为加载驱动，是 stap 追踪该驱动的前提）_：

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_YRWN2S5Y84FT5H5.webp)

**2. 解决 "stap x.stp" 执行失败**
---------------------------

    我一开始认为，驱动都已经加载了，stap 却还不认识，那问题应该出在 test.ko 的编译上，为了解决这个问题，兜兜绕绕了一大圈。

    首先是百度、google 了一遍，看了前几页的回答，都是说要安装内核 debuginfo，不过我的系统，正好之前已经安装过内核 debuginfo，而且我要追踪的是自己开发的驱动，按道理也不需要依赖内核 debuginfo。

     ![](https://bbs.kanxue.com/upload/attach/202312/815036_AQMHTWPJT2GQB8B.webp)

    然后到一些微信群里询问也无果，可能大佬都很忙，没空理我。

    最后只能自己再翻翻官方文档，开始各种推测与尝试。

    记忆中，之前看过一款动态追踪工具的原理，提到在编译被追踪程序时，gcc 必须添加 - pg 编译选项，这会使 gcc 在每个函数入口，添加 5 条 nop 指令，从而预留 5 个字节，可以在动态追踪时，替换成 "call mcount" 的机器码，才能使在追踪点注入的代码有机会执行。想到这，就在 Makefile 里加了一行：

```
ccflags-y += -pg

```

    结果，问题仍然存在，再回过头搜一下资料，得知 ftrace 才依赖 -pg 编译选项，systemtap 底层依赖的是 kprobe，它可以追踪任何地址处的指令，原理是将追踪地址的第一个字节，替换成 0xCC_（即 "int 3" 指令）_，利用中断机制实现的。

    那么，和 xfs 驱动相比，除了是否已加载和编译选项之外，还有什么区别？

![](https://bbs.kanxue.com/upload/attach/202312/815036_AYSW3F44CNXMPP9.webp)

    官方文档中的这么一句话，虽然只是阐述了一个客观情况，并没有表达，.ko 文件一定要放在 / lib/modules/$(uname -r)/ 目录，才能被追踪的意思，但是 test.ko 和 xfs 驱动相比，目前能想的的区别，也就这个了，所以就侥幸的试了一下，竟然成功了：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_7P8PP2APT328E2E.webp)

    并且，额外的惊喜是，"stat x.stp" 的执行，并不依赖先 "insmod test.ko"，也就是说，test_init() 函数，也可以被追踪。不过想想也是，如果连内核模块加载函数都追踪不了，那 systemtap 还号称什么 " 利器 "。  

3. 未显示函数名称
----------

    本来以为，接下来就可以尽情的畅游了。

    然而，通过 "insmod test.ko" 和 "rmmod test" 分别触发 test_init() 和 test_exit() 执行，发现 stap 的打印内容是这样：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_2NM4UC8FSZXDT8A.webp)

    其中，0xffffffffc037f000 是 test_init() 函数的加载地址，0xffffffffc0876000 是 test_exit() 函数的加载地址，这可以在 test.ko 卸载之前，通过以下 3 种方式证实：

    ① 查看 test.ko 节区的加载地址  

        ![](https://bbs.kanxue.com/upload/attach/202312/815036_UXG37ZJDZAUPRAS.webp)

    ② 查看内核符号表中，属于 test 驱动的符号及其加载地址_（不清楚为什么没看到 "test_init"）_  

        ![](https://bbs.kanxue.com/upload/attach/202312/815036_7SWJ95K5RDKQ9CZ.webp)

    ③ 查看这两个内存地址的内容，与 test_init()/test_exit() 函数反汇编的机器码对比_（这种方式要求系统安装了内核 debuginfo）_  

        test_exit() 函数机器码：

        ![](https://bbs.kanxue.com/upload/attach/202312/815036_ZEBWECPHGC6T36J.webp)

        内存查看：  

        ![](https://bbs.kanxue.com/upload/attach/202312/815036_V9VN59D6MXH3RNC.webp)

    确认打印内容中，"->" 之后是被追踪函数的地址之后，还存在另外 3 个疑问：

    1. stap 打印的为什么是函数地址，而不是函数名称？

        这个可以通过将 x.stp 脚本中的 probefunc()，替换成 ppfunc() 解决，同时也能避免以下问题 3 中的现象：

        ![](https://bbs.kanxue.com/upload/attach/202312/815036_3CYJ6HMU5XNPAJU.webp)

    2. test_init() 和 test_exit() 函数都可以追踪了，test() 函数为什么没被追踪到？  

        第 4 节专门介绍。  

    3. "<-" 与 "->" 后面的地址不同，又代表什么地址？  

        解决问题 2 后，让函数多调用几层，就能看出，"->" 后面是 callee 函数地址，"<-" 后面是 caller 函数地址。

4. 未追踪到 test() 函数
-----------------

    x.stp 脚本明明追踪的是所有函数_（function("*")）_，test() 函数却没被追踪到。  

    首先尝试的是，在 Makefile 中加一行：  

```
ccflags-y += -g

```

    然而，仍然不能追踪 test() 函数。

    执行 "readelf -S test.ko" 可以发现，不加 - g 编译，test.ko 就已经包含 debug_info 节区了，节区数量也不比加了 - g 编译少：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_FKAYUDSE4QG2BZH.webp)

    这时就想着看看，init_test()/test_exit() 函数中，是怎么调用 test() 函数的：

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_Y36P38YZ6729UHV.webp)

    可以看出，调用 test() 的 call 指令，在 test_init() 和 test_exit() 函数中的偏移，都是 0x1e，那么 0x1f 处一定有对应的重定项（这里需要一点链接原理的知识，可以看看我写的 "[32 位 elf 格式中的 10 种重定位类型](https://bbs.kanxue.com/thread-246373.htm) "）：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_86E3UDYARUZ55VJ.webp)

    最终得出，0x1f 处，原本应该填充为 test() 函数地址，但是却被直接填充为 printk() 函数的地址了，所以大致可以推测，可能由于 test() 除了调用一次 printk()，其它什么也没干，编译时就被 gcc 优化成直接调用 printk() 了。

    为了证实想法，在 Makefile 添加一行：  

```
ccflags-y += -O0

```

    再看重定项信息，就是 test() 函数了_（重新编译后，调用 test() 的 call 指令，位于 0x09 偏移）_：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_PBF7DF4HW7Y5BH4.webp)

    并且可以看到 test() 可以被追踪了：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_S3U6YC26KXNYW7B.webp)

5. 无法获取 test() 函数的参数和局部变量值  

    搞到这里，估计大家都不想再有什么妖蛾子了，但是 systemtap 才不管你想不想！  

    加了 - O0 选项编译，可以追踪 test() 函数之后，我又油然而生了一个僭越的想法，便将 x.stp 改了，试图追踪到 test() 函数时，打印一下参数和局部变量值：  

```
probe module("test").function("test").call {
  printf("%s -> %s, %s, %d\n", thread_indent(1), ppfunc(), $$parms, $n)
}

```

    结果却是：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_3WC2XT4FXVE42MB.webp)

    n 的值明显不对，n 等于 100 才对！  

    由于 - O0 给过我惊喜，加上如果优化级别为 0 都有问题，更何况更高的优化级别呢，所以我并没有第一时间想到它会害我，后来无意去掉 "ccflags-y += -O0"，发现获取到了 "n=0x64"，才没再留恋，果断弃了它。  

    不过去掉 "ccflags-y += -O0"，又得回去面对追踪不到 test() 函数的问题，但这不是 systemtap 的问题，而是 gcc 作祟，也可以理解为，test() 函数确实简单到不需要追踪了，所以，只要将 test() 函数改 " 复杂 "，就可以" 解决 " 这个问题：

```
void test(int n)
{
    int m = n/10 + 7;
    printk("%s(), %d: %d, %d\n", __FUNCTION__, __LINE__, n, m);
}

```

    然而，再次被 systemtap 玩耍：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_9FHMGS88PTXY7KD.webp)

    不过还是被我冷静的发现，开始执行 "cat /proc/kallsyms" 时，除了没看到 "test_init"，也没看到 "test"。

    对于 "test_init"，按常理应该和"test_exit"一样被显示才对，至于为什么没显示，我没再深究，但是可以感觉到，肯定和"test" 没被显示是有区别的，因为 test_init() 函数，一直都是可以被追踪的。由此推测，得让 test() 函数，在驱动加载后，也存在于内核符号表才行。

    于是尝试在 test.c 中，导出 test() 函数名称：

```
EXPORT_SYMBOL(test);

```

    最终达到了满意的效果：  

    ![](https://bbs.kanxue.com/upload/attach/202312/815036_AF93Y49Z8AEH57P.webp)

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

[#驱动开发](forum-41-1-132.htm) [#工具脚本](forum-41-1-137.htm)