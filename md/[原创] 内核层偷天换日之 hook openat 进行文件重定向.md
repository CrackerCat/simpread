> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269730.htm)

> [原创] 内核层偷天换日之 hook openat 进行文件重定向

目前很多防护系统做了自己 openat 实现不再依赖于 libc.so, 我们进行一些简单的 hook 技术已经免得比较麻烦, 于是有底层文件重定向的想法. 简单的进行对应的记录.

openat 重定向文件读写的一些问题
-------------------

前段时间, 卓玛星球的强哥, 提出了在内核层进行文件重定向的想法. 但是他告诉我这里面有很多限制. 比如__user, const 等. 于是我就顺着思路撸了一下内核代码, 具体相关情况如下:

1.  __user 和 const 都是类型修饰符而已, 主要目标是进行编译器检查. 但是在运行时是没有这种检查, 因此我们有 N 种方法进行处理
    
2.  文件重定向就是进行文件路径修改, 我们不能直接修改的原因. 得从 arm 内存访问以及代码实现说起.
    

do_sys_open 底层在做什么事情
--------------------

*   asmlinkage long sys_openat(int dfd, const char __user *filename, int flags, umode_t mode);
    *   do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
        *   tmp = getname(filename);
            *   getname_flags(filename, 0, NULL);
                *   step1 检查是否可以重用, 如果可以复用直接返回
                *   audit_reusename(filename);
                    *   遍历 current->audit_context->names_list
                    *   __audit_reusename(name) //fill out filename with info from existing entry
                        *   if (n->name->uptr == uptr) { // 检查 uptr 地址是否一致, 一致的话 refcnt++
                *   step2 embed the struct filename inside the names_cache
                *   len = strncpy_from_user(kname, filename, EMBEDDED_NAME_MAX); // 这里面有一次 filename 的访问, 这里是 EMBEDDED_NAME_MAX
                *   step3 接近 PATH_MAX, 拆分后在插入
                *   len = strncpy_from_user(kname, filename, PATH_MAX); // 这里是 EMBEDDED_NAME_MAX

上面列出所有访问 **user filename 的地方, 到后面进入 long strncpy_from_user(char *dst, const char** user *src, long count),

strncpy_from_user 底层到底在做什么事情
----------------------------

*   strncpy_from_user 前面主要是做一些合法性检查
    *   do_strncpy_from_user(dst, src, count, max);
        *   判断字节对齐
        *   缺页处理
        *   byte_at_a_time:
            *   unsafe_get_user

最开始我想直接修改源码去掉 **user 检查限制, 后面发现我太天真了, 层层限制, 层层都有** user 检查.

 

不过还有个终极手段就是直接改掉__user 修饰符, 思来想去还是没去做这个事情, 因为总感觉自己做的不对. 因此切换了思路, 我遇到的问题肯定别人也遇到过.

简单粗暴 memcopy
------------

A 思路走不通, 那就走 B 思路呗. 我仅仅是想修改内存而已, 我都是有特权级别的人了, 啥不能搞呢? 先写个代码试试看

```
// access userspace in kernel
do
{           
    mm_segment_t fs;
    fs =get_fs();
    set_fs(KERNEL_DS);
    if(strstr(pathname, "a.txt"))
    {
        if(access_ok(VERIFY_WRITE, pathname, len))
        {
            memcpy(pathname+len-5, "b.txt", 5); //受CONFIG_ARM64_PAN配置限制
 
            if(copy_to_user(pathname+len-5, "b.txt", 5)) {
                printk("[i] bingo magic\n");
            } else {
                printk("[err] copy to usr error. %p len=%d\n", pathname, len);
            }
 
            len = strncpy_from_user(bufname, pathname, 255);
            printk("[i] bufname =%s\n", bufname);
 
        }
        else {
            printk("[e] access not ok\n");
        }
    }           
    set_fs(fs);
} while (0);

```

似乎结果并不太好, 内核层崩溃了. 为啥呢? 思来想去查了很多资料, 重要找到了答案, 内核层有一个配置开关 CONFIG_ARM64_PAN

1.  CONFIG_ARM64_PAN 配置选项的功能是阻止内核态直接访问用户地址空间

具体核心代码如下:

```
// arch\arm64\include\asm\uaccess.h
#define __uaccess_disable(alt)                        \
do {                                    \
    if (!uaccess_ttbr0_disable())                    \
        asm(ALTERNATIVE("nop", SET_PSTATE_PAN(1), alt,        \
                CONFIG_ARM64_PAN));            \
} while (0)
 
#define __uaccess_enable(alt)                        \
do {                                    \
    if (!uaccess_ttbr0_enable())                    \
        asm(ALTERNATIVE("nop", SET_PSTATE_PAN(0), alt,        \
                CONFIG_ARM64_PAN));            \
} while (0)

```

修改后对应的风险 (欢迎大佬提供思路)
-------------------

因为我们直接更改了用户层的数据, 这里面就隐含了其他风险, 对应的用户层的路径名直接就变更了. 用户层再拿着这个  
数据进行处理, 就会遇到一些问题.

 

我后面仔细想了想还可以通过修改 libc.so 的方式, 但是很多防护系统都做了字段 openat. 因此我们只能去手动修改对应的 so 了.

 

欢迎各群里有理解 filesystem 系统的大佬提供下对应的意见和想法.

具体源码参考与实现
---------

[https://github.com/yhnu/op7t](https://github.com/yhnu/op7t)

参考链接:
-----

[https://github.com/yhnu/op7t/commit/ec30f893b42aba551c84b7036aaa71f012703f4a](https://github.com/yhnu/op7t/commit/ec30f893b42aba551c84b7036aaa71f012703f4a)

 

[https://developer.ibm.com/articles/l-kernel-memory-access/](https://developer.ibm.com/articles/l-kernel-memory-access/)

 

[http://www.wowotech.net/memory_management/454.html](http://www.wowotech.net/memory_management/454.html)

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

[#系统相关](forum-161-1-126.htm) [#源码分析](forum-161-1-127.htm)