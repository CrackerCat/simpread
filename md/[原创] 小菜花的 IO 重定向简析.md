> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271041.htm)

> [原创] 小菜花的 IO 重定向简析

0x0 引言  

=========

*   X：svc 跟 syscall 的区别是啥
    
*   Y：xxxx 唧唧哇哇 oooo
    
*   X：怎么 bypass 内存搜索还是 ptrace
    
*   Y：va 用的内存搜索 + inline 的（目前所看到的源码），隔壁珍惜大佬用的 ptrace 的
    
*   Y：大佬们要不要关注下我公众号，突然灵机一动，想写个 openat，syscall，svc，和 va io svcbypass 处理的文章 [皱眉]
    

    以上纯属是个玩笑。当然了，本文要表述的这部分内容比较基础，大佬级别手下留情，给小菜花留点活路，感激涕零  

0x1 系统调用相关概念  

===============

 我先去 cv 一点概念性的东西  

![](https://bbs.pediy.com/upload/attach/202201/844301_4T2TRFM9AQDKJD3.jpg)

    用户态是怎么进入到内核态呢？这着实让人头疼。实际上常见的 kill 方法 就会进入到内核态，而沟通内核态的方法就是 sys_call 方法。![](https://bbs.pediy.com/upload/attach/202201/844301_6D83G66JEY6CAVQ.jpg)

0x2 跟踪 openat 源码
================

咱们这里以 android8.0 arm64 为例: [http://androidxref.com/8.0.0_r4](http://androidxref.com/8.1.0_r33/)

![](https://bbs.pediy.com/upload/attach/202201/844301_QPA73C5BN5RY9AF.jpg)

可以看到，open 和 openat 的实现都是__openat，那就去__openat 看下  

![](https://bbs.pediy.com/upload/attach/202201/844301_QAYMJM534VD8MEM.jpg)

那就看 64 位吧

![](https://bbs.pediy.com/upload/attach/202201/844301_JR8DE9JTUXD4UJM.jpg)

svc 出现了

0x3 跟踪 syscall 源码
=================

使用方式：

```
pid_t pid = syscall(__NR_getpid);

```

源码：

![](https://bbs.pediy.com/upload/attach/202201/844301_U9PX9D3JBJKACDY.jpg)

svc 出现了

0x4 io 重定向
==========

va：https://github.com/asLody/VirtualApp

ratel：[https://github.com/virjarRatel/ratel-core](https://github.com/virjarRatel/ratel-core)

io 重定向都是上述两个产品的核心功能之一，并且它俩的逻辑差不多哈，因为 ratel 是新开源的，所以直接拿 ratel 来看  

（ps：io 重定向可以干的事很多哈，如 va 的多开原理，ratel 的一机多号原理，bypass 文件检测等场景中，io 重定向都起到了至关重要的作用）

![](https://bbs.pediy.com/upload/attach/202201/844301_T8Z6F47SZGBQRH2.jpg)

函数 hook

![](https://bbs.pediy.com/upload/attach/202201/844301_HTRYPVTYBS6SZZV.jpg)

内存扫描 + hook

![](https://bbs.pediy.com/upload/attach/202201/844301_EEGM2JRMVTT2JZW.jpg)

![](https://bbs.pediy.com/upload/attach/202201/844301_3U2HVNAHRYG6Q8G.jpg)

这部分优雅的写法主要是 IORelocator.h 中几个宏决定的，大家可以好好读读源码，然后学习和 cv 一波

frida 方式见我的另一篇文章：[https://bbs.pediy.com/thread-270144.htm](https://bbs.pediy.com/thread-270144.htm)

0x5 总结
======

*   ```
    int fd = openat(dirfd, "output.log", O_CREAT|O_RDWR|O_TRUNC, 0777);
    
    ```
    
*   ```
    long fd = syscall(__NR_openat, dirfd, "output.log", O_CREAT|O_RDWR|O_TRUNC, 0777);
    
    ```
    

以 上述日常读文件方式 和 android8.0 源码 arm 模式 为例：

*   都要由用户态进入内核态
    
*   进入内核态的逻辑由汇编实现（准备寄存器参数，然后 swi/svc）  
    
*   32 位下是 swi 指令，64 位下是 svc 指令
    
*   openat -> __openat -> __openat.S -> 准备系统调用号，然后 swi/svc（这里可以理解写死调用号形式的汇编实现）
    
*   syscall -> syscall.S -> 准备系统调用号，然后 swi/svc（这里可以理解为传参形式的汇编实现）
    

*   io 重定向的应用场景很多
    

*   ratel hook 了
    

*   以__openat 为代表的文件操作相关函数
    
*   以__openat.S 为代表的具体的文件操作方式的 swi/svc 逻辑（写死调用号形式的汇编实现）
    
*   app so 自实现的具体的文件操作方式的 swi/svc 逻辑（写死调用号形式的汇编实现）（ps：目前 hook 时机在 dlopen 后，目标自实现 svc 汇编写法要和__openat.S 中写法类似，才会生效。哈哈，这部分逻辑我提的 pr，cv 大法好。当然了，hook 的时机、汇编特征、更多调用号的适配都可以继续优化）
    

*   ratel 未 hook
    

*   syscall 这个函数
    
*   syscall.S 的汇编实现
    

*   va，ratel，fakexposed.... 等项目中有很多可以学习和 cv 的地方，公众号发了个人理解的方案流 / 算法流技术晋级路线
    
*   感谢上述的各位巨巨
    

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 1 小时前 被 huaerxiela 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#源码分析](forum-161-1-127.htm)