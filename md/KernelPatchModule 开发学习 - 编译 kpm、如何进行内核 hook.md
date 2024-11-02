> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1977664-1-1.html)

> 0x0 前言随着安卓逆向研究已经开始进入内核时代，现在内核相关的开发软件和资料也是越来越多。最近发现 apatch 的 kpm 挺好用的，不依赖内核源码，编译也很方便，还有现成的 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)jbczzz _ 本帖最后由 jbczzz 于 2024-11-1 17:07 编辑_  
**0x0 前言**  
随着安卓逆向研究已经开始进入内核时代，现在内核相关的开发软件和资料也是越来越多。最近发现 apatch 的 kpm 挺好用的，不依赖内核源码，编译也很方便，还有现成的几种内核 hook 方式。但是在学习的过程中发现目前相关资料还是比较少，于是想用文章记录一下自己的学习过程。  
**0x1** **简介、****环境**  
什么是 KernelPatchModule？  
官网的简介：一些代码在内核空间运行，类似于 Loadable Kernel Modules（LKM）。此外，KPM 提供在内核空间进行内联 hook、系统调用表 hook 的能力。KPM 是一个 ELF 文件，可由 KernelPatch 在内核空间内加载和运行。  
官方 GitHub 地址  

> https://github.com/bmax121/KernelPatch/tree/dev

  
环境、工具版本：  
Ubuntu 22.04.2 LTS  
gcc-arm-11.2-2022.02-x86_64-aarch64-none-elf.tar  
GNU Make 4.3  
**0x2 编译**  
首先在官网下一份源码  
![](https://attach.52pojie.cn/forum/202411/01/170709q5fhpdiyorwhdleh.png)

**76AY`_TG1JNE9VUUOJE)WBK.png** _(10.76 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjczMjcxMHxmYzk0YTc5YXwxNzMwNTE3MzYwfDIxMzQzMXwxOTc3NjY0&nothumb=yes)

2024-11-1 17:07 上传

  
我们这里只需要关心 kpms 里的例子是怎么写的  
然后是在这下载 GNU tools ，建议使用 aarch64-none-elf，我之前使用 linux 自带的 aarch64-linux-gnu 编译出来无法正常使用。  
https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads/11-2-2022-02 进入 / kpms/demo-hello 这个目录下  
[Shell] _纯文本查看_ _复制代码_

```
export TARGET_COMPILE=$(yourPath)/aarch64-none-elf-
make
```

![](https://attach.52pojie.cn/forum/202411/01/170644t3nu5c52c26355t5.png)

**RVCKBHHEF_MTW[B3ZRM6W2N.png** _(42.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjczMjcwN3w2NTlhOTUzYXwxNzMwNTE3MzYwfDIxMzQzMXwxOTc3NjY0&nothumb=yes)

2024-11-1 17:06 上传

  
输出的. kpm 就是编译出来的模块  
在 apatch 里内核模块 -> 右下角图标 -> 加载 -> 选择 hello.kpm, 加载成功后会显示  
![](https://attach.52pojie.cn/forum/202411/01/170647gybby4cgbb6y85pb.jpg)

**78ead52bb47a76169322725ba031a510.jpg** _(43 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjczMjcwOXxiMjg1OTQ1MHwxNzMwNTE3MzYwfDIxMzQzMXwxOTc3NjY0&nothumb=yes)

2024-11-1 17:06 上传

  
使用 demsg|grep kp 就能看到模块加载的信息  
![](https://attach.52pojie.cn/forum/202411/01/170646uzyj7dlleimsdjd0.png)

**@88D003H`3~)NT}K[OU~NU7.png** _(96.55 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjczMjcwOHxjNmVkZjJlNnwxNzMwNTE3MzYwfDIxMzQzMXwxOTc3NjY0&nothumb=yes)

2024-11-1 17:06 上传

  
**0x3 kpm 代码的基本结构**  
需要关注的是这些地方  
[C++] _纯文本查看_ _复制代码_

```
///< The name of the module, each KPM must has a unique name.
KPM_NAME("kpm-hello-demo");
 
///< The version of the module.
KPM_VERSION("1.0.0");
 
///< The license type.
KPM_LICENSE("GPL v2");
 
///< The author.
KPM_AUTHOR("AKI");
 
///< The description.
KPM_DESCRIPTION("KernelPatch Module AKI Test");
 
#define KPM_INIT(fn) \
    static mod_initcall_t __kpm_initcall_##fn __attribute__((__used__)) __attribute__((__section__(".kpm.init"))) = fn
 
#define KPM_CTL0(fn) \
    static mod_ctl0call_t __kpm_ctlmodule_##fn __attribute__((__used__)) __attribute__((__section__(".kpm.ctl0"))) = fn
 
#define KPM_CTL1(fn) \
    static mod_ctl1call_t __kpm_ctlmodule_##fn __attribute__((__used__)) __attribute__((__section__(".kpm.ctl1"))) = fn
 
#define KPM_EXIT(fn) \
    static mod_exitcall_t __kpm_exitcall_##fn __attribute__((__used__)) __attribute__((__section__(".kpm.exit"))) = fn
 
#endif
KPM_INIT(hello_init);  
KPM_CTL0(hello_control0);
KPM_CTL1(hello_control1);
KPM_EXIT(hello_exit);
```

其中：  
KPM_INIT(hello_init);   //kpm 加载时调用的函数 KPM_CTL0(hello_control0);  // 在 apatch 中传入参数调用的函数  
KPM_CTL1(hello_control1);// 在 apatch 中传入参数调用的函数  
KPM_EXIT(hello_exit); // 卸载 kpm 调用的函数  
**0x4 根据内核符号地址 hook 内核函数**  
以 hook process_vm_rw 为例子，这个函数在 / mm / process_vm_access.c。  
首先需要找到函数原型，可以结合我上一篇还原内核符号的文章和那个查看内核符号的网站使用。  
函数原型：  
[C++] _纯文本查看_ _复制代码_

```
static ssize_t process_vm_rw(pid_t pid,
                             const struct iovec __user *lvec,
                             unsigned long liovcnt,
                             const struct iovec __user *rvec,
                             unsigned long riovcnt,
                             unsigned long flags, int vm_write) //5.4.210内核源码的函数原型
```

想要 hook 这个函数，首先在文件中按照原来的代码构建一个原型：  
[C++] _纯文本查看_ _复制代码_

```
ssize_t (*process_vm_rw)(pid_t pid, const struct iovec __user *lvec, unsigned long liovcnt,
                       const struct iovec __user *rvec, unsigned long riovcnt, unsigned long flags, int vm_write) = 0;
```

使用 kallsyms_lookup_name，在 / kernel/include/kallsyms.h 中，作用是寻找返回所查找的内核函数地址，找不到返回 0。具体详见  https://blog.csdn.net/weixin_45030965/article/details/132497956  
查找内核符号地址：  
[C++] _纯文本查看_ _复制代码_

```
process_vm_rw = (typeof(process_vm_rw))kallsyms_lookup_name("process_vm_rw");
```

进行 inline hook,hook_wrapX, 这个 X 代表的是需要几个参数，里面调用的是 hook_wrap()。before 是调用前执行的函数，after 是调用后执行的函数:  
[C++] _纯文本查看_ _复制代码_

```
hook_wrap8((void *)process_vm_rw, before, after, 0);
```

**0x5 syscallHook**  
根据系统调用号 hook 内核函数，以例子中的__task_pid_nr_ns 这个函数为例，这个内核函数在 / kernel/pid.c  
根据系统调用号进行 hook，分两种 function_pointer_hook 和 inline_hook，他们调用的方式都是类似的  
[C++] _纯文本查看_ _复制代码_

```
#define __NR_openat 56
err = inline_hook_syscalln(__NR_openat, 4, before_openat_0, 0, 0);
err = fp_hook_syscalln(__NR_openat, 4, before_openat_0, 0, 0);
```

关于这个工具 hook syscall 的具体原理流程后续随缘更新，这里就只讲使用。  
**0x6 小结**  
如果想知道具体 kpm 是如何加载运行的话，需要先了解 APatch 是如何运行的，这个现在还在看 kptools 的源码，还是后续随缘更新，感兴趣的可以先看看 https://bbs.kanxue.com/homepage-935696.htm 这个大佬的文章。![](https://avatar.52pojie.cn/data/avatar/000/80/17/56_avatar_middle.jpg)ytfrdfiw 请问一下楼主，android 内核的 hook 一般是什么应用场景，感谢分享。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)jbczzz

> [ytfrdfiw 发表于 2024-11-1 17:39](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=51614687&ptid=1977664)  
> 请问一下楼主，android 内核的 hook 一般是什么应用场景，感谢分享。

比如说对抗设备环境检测、反调试、process_vm_readv 检测之类的，能更方便的监控应用行为，辅助分析 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) chuanyipai 不错，学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) junxin 学到了，感谢楼主 ![](https://avatar.52pojie.cn/data/avatar/000/73/84/40_avatar_middle.jpg) debug_cat [Asm] _纯文本查看_ _复制代码_

```
#define KPM_INIT(fn) \
    static mod_initcall_t __kpm_initcall_##fn __attribute__((__used__)) __attribute__((__section__(".kpm.init"))) = fn
```

在官方的 hello demo 中没有发现上面的代码，这用途是?![](https://avatar.52pojie.cn/images/noavatar_middle.gif)lmx352470462  
谢谢分享![](https://static.52pojie.cn/static/image/smiley/default/48.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) asd885522 学到了，感谢楼主![](https://static.52pojie.cn/static/image/smiley/default/47.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) yan999 学习下，感谢分享