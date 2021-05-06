> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-259317-1.htm)

[](#linux它是一款开源的内核系统。本人也非常喜欢嵌入式linux系统，特别是它的内核源码，书写的风格，都非常讨我心欢。这个驱动是之前业余的时候写的对于新手来说，还是有学习价值的。)Linux 它是一款开源的内核系统。本人也非常喜欢嵌入式 Linux 系统，特别是它的内核源码，书写的风格，都非常讨我心欢。这个驱动是之前业余的时候写的对于新手来说，还是有学习价值的。
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

![](https://bbs.pediy.com/upload/attach/202104/892833_FREFW2T45EAQUPB.png)  
![](https://bbs.pediy.com/upload/attach/202104/892833_CEYYJR8PVEXXF5M.jpg)

##### [](#下面将对源码进行简单的讲解。)下面将对源码进行简单的讲解。

[](#首先是隐藏内核驱动模块。)首先是隐藏内核驱动模块。
-----------------------------

```
list_del_init(&__this_module.list);
kobject_del(&THIS_MODULE->mkobj.kobj);

```

list_del_init 是将自身驱动模块从驱动列表（lsmod）中抹掉  
kobject_del 是将自己从 / sys/class/xxxxxx 中抹掉

[](#接下来是打开进程接口。)接下来是打开进程接口。
---------------------------

在 Linux 内核里，不区分进程与线程。统一按照线程来看待。那么每个线程都有一个对应的 pid_t、pid_、task_struct。  
他们之间的关系是这样的：  
pid_t <–> struct pid_  
nr 为进程 pid 数值

```
#include pid_t pid_vnr(struct pid *pid)
{
    return pid_nr_ns(pid, current->nsproxy->pid_ns);
}
EXPORT_SYMBOL_GPL(pid_vnr);
 
struct pid *find_pid_ns(int nr, struct pid_namespace *ns);
EXPORT_SYMBOL_GPL(find_pid_ns);
 
struct pid *find_vpid(int nr)
{
    return find_pid_ns(nr, current->nsproxy->pid_ns);
}
EXPORT_SYMBOL_GPL(find_vpid);
 
struct pid *find_get_pid(int nr)
{
    struct pid *pid;
 
    rcu_read_lock();
    pid = get_pid(find_vpid(nr));
    rcu_read_unlock();
 
    return pid;
}
EXPORT_SYMBOL_GPL(find_get_pid);
 
void put_pid(struct pid *pid);
EXPORT_SYMBOL_GPL(put_pid);
 
 
struct pid * –> struct task_struct *
 
 
#include struct pid *get_task_pid(sturct task_struct *task, enum pid_type);
EXPORT_SYMBOL_GPL(get_task_pid);
 
struct task_struct *pid_task(struct pid *pid, enum pid_type);
EXPORT_SYMBOL(pid_task);
 
struct task_struct *get_pid_task(struct pid *pid, enum pid_type)
{
    struct task_struct *result;
    rcu_read_lock();
    result = pid_task(pid, type);
    if (result)
        get_task_struct(result);
    rcu_read_unlock();
    return result;
}
EXPORT_SYMBOL(get_pid_task);
 
#include #define get_task_struct(tsk) do { atomic_inc(&(tsk)->usage); } while (0)
static inline void put_task_struct(struct task_struct *t)
{
    if (atomic_dec_and_test(&t->usage))
        __put_task_struct(t);
}
 
void __put_task_struct(struct task_struct *t);
EXPORT_SYMBOL_GPL(__put_task_struct); 
```

看完以上的逻辑。大家是不是柳暗花明又一村，心里开朗了许多，他们之间是可以相互转换的。通过进程 pid_t 可以拿到 pid_，通过 pid_ 可以拿到 task_struct。又可以反过来通过 task_struct 拿到进程 pid。

然后是关闭进程接口
---------

驱动源码是使用 put_pid 将进程 pid * 的使用次数减去 1

 

在 Linux 内核源码 / kernel/pid.c 下可以看到

```
void put_pid(struct pid *pid)
{
    struct pid_namespace *ns;
 
    if (!pid)
        return;
 
    ns = pid->numbers[pid->level].ns;
    if ((atomic_read(&pid->count) == 1) ||
         atomic_dec_and_test(&pid->count)) {
        kmem_cache_free(ns->pid_cachep, pid);
        put_pid_ns(ns);
    }
}
EXPORT_SYMBOL_GPL(put_pid);

```

[](#读进程内存、写进程内存接口)读进程内存、写进程内存接口
===============================

这里采用读、写物理内存的思路，因为这样简洁明了，不需要理会其他反调试干扰。  
首先根据 pid * 用 get_pid_task 取出 task_struct。再用 get_task_mm 取出 mm_struct 结构。因为这个结构包含了进程的内存信息。首先检查内存是否可读 if (vma->vm_flags & VM_READ)  
如果可读。那么开始计算物理内存地址位置。由于 Linux 内核默认开启 MMU 机制，所以只能以页为单位计算物理内存地址。计算物理内存地址的方法有很多，这里提供三种思路：  
第一种是使用 get_user_pages，

### [](#第二种是直接使用pagemap文件，)第二种是直接使用 pagemap 文件，

### [](#第三种是纯手写算法（将pgd、pud、pmd和pte拼接在一起）)第三种是纯手写算法（将 pgd、pud、pmd 和 pte 拼接在一起）

### [](#我个人推荐使用第三种方法，也是我正在使用的方法，速度极快而且不触碰任何的进程文件，被调试进程无任何感知。)我个人推荐使用第三种方法，也是我正在使用的方法，速度极快而且不触碰任何的进程文件，被调试进程无任何感知。

### [](#不推荐使用第一种get_user_pages方法，因为此方法会触发反调试机制，导致调试失败。)不推荐使用第一种 get_user_pages 方法，因为此方法会触发反调试机制，导致调试失败。

知道了物理内存地址后，读、写物理内存地址，其实 Linux 内核里面也有演示，即 drivers/char/mem.c。写的非常详细。最后还要注意 MMU 机制的离散内存，即 buffer 不连续问题，通俗的说就是不要一下子读太多，读到另一页去了，要分开页来读

获取进程内存块列表
=========

这个接口就很简单了，通过 task_struct 取出 mm_struct，接下来在 mm_struct 中遍历取出 vma。详情可以参考代码 fs\proc\task_mmu.c

```
struct mm_struct {
    struct vm_area_struct * mmap;        /* list of VMAs */
    struct rb_root mm_rb;
    struct vm_area_struct * mmap_cache;    /* last find_vma result */
#ifdef CONFIG_MMU
    unsigned long (*get_unmapped_area) (struct file *filp,
                unsigned long addr, unsigned long len,
                unsigned long pgoff, unsigned long flags);
    void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
#endif
    unsigned long mmap_base;        /* base of mmap area */
    unsigned long mmap_legacy_base;         /* base of mmap area in bottom-up allocations */
    unsigned long task_size;        /* size of task vm space */
    unsigned long cached_hole_size;     /* if non-zero, the largest hole below free_area_cache */
    unsigned long free_area_cache;        /* first hole of size cached_hole_size or larger */
    unsigned long highest_vm_end;        /* highest vma end address */
    pgd_t * pgd;
    atomic_t mm_users;            /* How many users with user space? */
    atomic_t mm_count;            /* How many references to "struct mm_struct" (users count as 1) */
    int map_count;                /* number of VMAs */
 
    spinlock_t page_table_lock;        /* Protects page tables and some counters */
    struct rw_semaphore mmap_sem;
 
    struct list_head mmlist;        /* List of maybe swapped mm's.    These are globally strung
                         * together off init_mm.mmlist, and are protected
                         * by mmlist_lock
                         */
 
 
    unsigned long hiwater_rss;    /* High-watermark of RSS usage */
    unsigned long hiwater_vm;    /* High-water virtual memory usage */
 
    unsigned long total_vm;        /* Total pages mapped */
    unsigned long locked_vm;    /* Pages that have PG_mlocked set */
    unsigned long pinned_vm;    /* Refcount permanently increased */
    unsigned long shared_vm;    /* Shared pages (files) */
    unsigned long exec_vm;        /* VM_EXEC & ~VM_WRITE */
    unsigned long stack_vm;        /* VM_GROWSUP/DOWN */
    unsigned long def_flags;
    unsigned long nr_ptes;        /* Page table pages */
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;
 
    unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

```

获取进程命令行
=======

mm_struct 结构体里面有个 arg_start 变量，储存的地址值即是进程命令行。  
但这里有个要注意的地方，经过我多台设备测试发现，并不是每个 Linux 内核系统的 arg_start 变量偏移值是一样的，这样子就会非常危险，一旦读错就会死机，而且原因还不好查找。

[](#这里我使用了一些玄学的技巧，驱动可以自适应地在不同的linux内核中自行修正arg_start变量的偏移值，从而读取到正确的值，不会死机。)这里我使用了一些玄学的技巧，驱动可以自适应地在不同的 Linux 内核中自行修正 arg_start 变量的偏移值，从而读取到正确的值，不会死机。
---------------------------------------------------------------------------------------------------------------------------------------------------

获取进程 PID 列表
===========

驱动里写入了两种方法，第一种是遍历 / proc/pid 目录，第二种是遍历 task_struct 结构体。这里有个要注意的地方，经过我多台设备测试发现，并不是每个 Linux 内核系统的 task_struct 结构体里的 tasks 变量偏移值是一样的，但具体玄学修正方法我还没时间进行编写，待有空再补充。

获取进程物理内存大小
==========

读取 task_struct 结构体里的 mm_struct，再读取 rss_stat 就会有进程的物理内存占用大小，这个来源与 / proc/pid/status 里的源码编写。这里同样需要玄学技巧修正变量的偏移值，具体方法我已编写在内。

[](#获取进程权限、设置进程权限为root)获取进程权限、设置进程权限为 ROOT
==========================================

在取得 task_struct 进程结构后，观察头文件可以发现里面有两个变量值，一个是 real_cred，另一个是 cred，其实很简单，将两个 cred 里面的 uid、gid、euid、egid、fsuid、fsgid 修改成 0 即可

```
const struct cred __rcu *real_cred;
const struct cred __rcu *cred;

```

real_cred 指向主体和真实客体证书，cred 指向有效客体证书。通常情况下，cred 和 real_cred 指向相同的证书，但是 cred 可以被临时改变  
同样需要注意，每个 Linux 内核的 cred 结构变量偏移值并不是一样的，读错会死机，同理，我也使用了玄学的技巧，驱动能智能修正 Linux 内核变量的偏移值，能准确的识别出每个 Linux 内核版本里 real_cred 与 cred 的正确偏移位置  
![](https://bbs.pediy.com/upload/tmp/892833_JF2VGK6DEF2SPKZ.png)  
![](https://bbs.pediy.com/upload/tmp/892833_U7RAEH3G4RK62D9.png)

#### [](#开源地址（含使用demo）)开源地址（含使用 demo）

[Github 跳转](https://github.com/abcz316/rwProcMem33)

[](#总结：)总结：
===========

首先，编译此源码需要一定的技巧，再者，手机厂商本身已设置多重障碍用来阻止第三方驱动的加载，如果您需要加载此驱动，则需要将内核中的一些限制给去除。（其实这些验证都可以用 IDA 暴力 Patch 之~）。

[](#提醒：)提醒：
===========

**本源码不针对任何程序，仅供交流、学习、调试 Linux 内核程序的用途，禁止用于任何非法用途，调试器是一把双刃剑，希望大家能营造一个良好的 Linux 内核 Rootkit 技术的交流环境。**

[](#后续：)后续：
===========

后面即将开源：  
**“不需要源码，强制暴力修改手机 Linux 内核、去除加载内核驱动的所有验证”**  
**“不需要源码，强制加载启动 ko 驱动文件、跨 Linux 内核版本、跨设备启动 ko 驱动模块文件”**  
**“不需要源码，Linux 内核进程万能调试器 - 可过所有的反调试机制 - 含硬件断点：Linux 天下调，让天下没有难调试的 Linux 进程！”**  
**“不需要源码，突破 Linux 内核 Elf 结构限制、将 ollvm 混淆加入到 ko 驱动文件中，增加驱动逆向难度”**

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 2021-4-3 01:42 被 abcz316 编辑 ，原因： 补充图片