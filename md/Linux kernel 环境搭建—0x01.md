> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1710242-1-1.html)

> [md]# 向内核 pwn 迈进 ## 自行编写驱动首先咱们来介绍以下基础知识 ### 基础知识 ### Loadable Kernel Modules(LKMs) 可加载核心模块 (或直接称为内核模块) 就像运行......

![](https://avatar.52pojie.cn/data/avatar/001/92/50/45_avatar_middle.jpg)peiwithhao

向内核 pwn 迈进
==========

自行编写驱动
------

首先咱们来介绍以下基础知识

### 基础知识

### Loadable Kernel Modules(LKMs)

可加载核心模块 (或直接称为内核模块) 就像运行在内核空间的可执行程序，包括:

*   驱动程序（Device drivers）
*   设备驱动
*   文件系统驱动  
    ...  
    内核扩展模块 (modules)  
    LKMs 的文件格式和用户态的可执行程序相同，Linux 下为 ELF，Windows 下为 exe/dll，mac 下为 MACH-O，因此我们可以用 IDA 等工具来分析内核模块。

模块可以被单独编译，但不能单独运行。它在运行时被链接到内核作为内核的一部分在内核空间运行，这与运行在用户控件的进程不同。

模块通常用来实现一种文件系统、一个驱动程序或者其他内核上层的功能。

* * *

而 Linux 内核之所以提供模块机制，是因为它本身是一个单内核 (monolithic kernel)。单内核的优点是效率高，因为所有的内容都集合在一起，但缺点是可扩展性和可维护性相对较差，模块机制就是为了弥补这一缺陷。（而一般内核 pwn 中的漏洞出自这些模块里面，但是有的师傅也说内核也会有漏洞，这咱们以后再学）  
这里比较常用的指令有以下几个：

*   insmod: 讲指定模块加载到内核中
*   rmmod: 从内核中卸载指定模块
*   lsmod: 列出已经加载的模块
*   modprobe: 添加或删除模块，modprobe 在加载模块时会查找依赖关系
*   dmesg: 输出内核态缓冲区的输出，这里跟用户态不一样，用户态一般输出到屏幕就完事了，内核中是输出到缓冲区。

这里注意指令均需运行在管理员权限下。

* * *

这里还注意一个特殊的函数 ioctl

#### ioctl

直接查看 man 手册

```
NAME
       ioctl - control device

SYNOPSIS
       #include <sys/ioctl.h>

       int ioctl(int fd, unsigned long request, ...);

DESCRIPTION
       The ioctl() system call manipulates the underlying device parameters of special
       files.  In particular, many  operating  characteristics  of  character  special
       files  (e.g., terminals) may be controlled with ioctl() requests.  The argument
       fd must be an open file descriptor.

       The second argument is a device-dependent request code.  The third argument  is
       an  untyped  pointer  to  memory.  It's traditionally char *argp (from the days
       before void * was valid C), and will be so named for this discussion.

       An ioctl() request has encoded in it whether the argument is an in parameter or
       out  parameter, and the size of the argument argp in bytes.  Macros and defines
       used in specifying an ioctl() request are located in the file <sys/ioctl.h>.

```

可以看出 ioctl 也是一个系统调用，用于与设备通信。  
int ioctl(int fd, unsigned long request, ...) 的第一个参数为打开设备 (open) 返回的 文件描述符，第二个参数为用户程序对设备的控制命令，再后边的参数则是一些补充参数，与设备有关。

使用 ioctl 进行通信的原因：

> 操作系统提供了内核访问标准外部设备的系统调用，因为大多数硬件设备只能够在内核空间内直接寻址, 但是当访问非标准硬件设备这些系统调用显得不合适, 有时候用户模式可能需要直接访问设备。  
> 比如，一个系统管理员可能要修改网卡的配置。现代操作系统提供了各种各样设备的支持，有一些设备可能没有被内核设计者考虑到，如此一来提供一个这样的系统调用来使用设备就变得不可能了。  
> 为了解决这个问题，内核被设计成可扩展的，可以加入一个称为设备驱动的模块，驱动的代码允许在内核空间运行而且可以对设备直接寻址。一个 Ioctl 接口是一个独立的系统调用，通过它用户空间可以跟设备驱动沟通。对设备驱动的请求是一个以设备和请求号码为参数的 Ioctl 调用，如此内核就允许用户空间访问设备驱动进而访问设备而不需要了解具体的设备细节，同时也不需要一大堆针对不同设备的系统调用。

### 1. 初级 LKM 模块

```
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init kernel_module_init(void)
{
    printk("<1>Hello the Linux kernel world!\n");
    return 0;
}

static void __exit kernel_module_exit(void)
{
    printk("<1>Good bye the Linux kernel world! See you again!\n");
}

module_init(kernel_module_init);
module_exit(kernel_module_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("peiwithhao");

```

当编写完成后咱们用 make 脚本进行编译链接，脚本如下

```
obj-m += peiwithhao
CURRENT_PATH := $(shell pwd)
LINUX_KERNEL := $(shell uname -r)
LINUX_KERNEL_PATH := /usr/src/linux-headers-$(LINUX_KERNEL)
all:
    make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules
clean:
    make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean


```

这里程序有几个注意点在以下标识

#### 头文件

*   linux/module.h：对于 LKM 而言这是必须包含的一个头文件
*   linux/kernel.h：载入内核相关信息
*   linux/init.h：包含着一些有用的宏

通常情况下，这三个头文件对于内核模块编程都是不可或缺的

#### 入口点 / 出口点

一个内核模块的入口点应当为 module_init()，出口函数应当为 module_exit()，在内核载入 / 卸载内核模块时会缺省调用这两个函数

在这里我们将自定义的两个函数的指针作为参数传入 LKM 入口函数 / 出口函数中，以作为其入口 / 出口函数

#### 其他

**init &** exit：这两个宏用以在函数结束后释放相应的内存  
MODULE_AUTHOR() & MODULE_LICENSE()：声明内核作者与发行所用许可证  
printk()：内核态函数，用以在内核缓冲区写入信息，其中 < 1> 标识着信息的紧急级别（一共有 8 个优先级，0 为最高，相关宏定义于 linux/kernel.h 中，这个大伙可以查查资料，在后面我一般用 printk(KERN_INFO""), 这个跟 < 6 > 好像时一个意思）

由于 make 我也才刚接触，所以这里不做解释免得误导大家。  
这里继续`make`即可，然后咱们使用如下命令

```
insmod hellokernel.ko
lsmod
rmmod hellokernel
dmesg

```

这里由于我之前这个模块过于简单就忘保存了，所以引用一下师傅的图  
![](https://i.loli.net/2021/02/28/vJitGgkFTzcPx8a.png)

### 2. 提供 IO 接口

虽然说我们的新模块成功跑起来了，但是除了在内核缓冲区进行输入输出以外好像就做不了什么了，我们希望我们写的内核模块能够向我们提供更多的功能并能够让用户与其进行交互，以发挥更多的作用  
所以这里我们的首要步骤就是注册设备。而为了与设备进行交互，我们就需要驱动来为咱们隐藏底层做的很多事，就好比我想用打印机，但要是咱们编程的时候既要考虑到用户这边，又要考虑到设备这边的问题，咱们就太累了，幸好在计算机中没有什么是加一层解决不了的，如果不行，那就加两层。  
所以出现了驱动的这一概念，驱动也就帮咱们隐藏了底层实现，咱们调用的时候只需要会 open，read，write 即可。

#### 设备分类

在 Linux 中 I/O 设备分为如下两类：

*   字符设备：在 I/O 传输过程中以字符为单位进行传输的设备，例如键盘、串口等。字符设备按照字符流的方式被有序访问，不能够进行随机读取
*   块设备：在块设备中，信息被存储在固定大小的块中，每个块有着自己的地址，例如硬盘、SD 卡等。用户可以对块设备进行随机访问——从任意位置读取一定长度的数据

#### file_operations 结构体

在注册设备之前，我们需要用到一个结构体——file_operations 来完成对设备的一些相关定义，该结构体定义于 include/linux/fs.h 中，相关源码比较长不在此贴出，在其中定义了大量的函数指针, 这里再之后的代码会体现出来，他也就是定义了 open、read 等系统调用的函数指针。

一个文件应当拥有一个 file_operations 实例，并指定相关系统调用函数指针所指向的自定义函数，在后续进行设备的注册时会使用该结构体

#### 主设备号 & 次设备号

在 Linux 内核中，使用类型 dev_t（unsigned long）来标识一个设备的设备号。  
一个字符的设备号由主设备号与次设备号组成，高字节存储主设备号，低字节存储次设备号：  
主设备号：标识设备类型，使用宏 MAJOR(dev_t dev) 可以获取主设备号  
次设备号：用以区分同类型设备，使用宏 MINOR(dev_t dev) 可以获取次设备号  
Linux 还提供了一个宏 MKDEV(int major, int minor);，用以通过主次设备号生成对应的设备号

#### 设备节点（struct device_node & struct device）

由于 Linux 中所有的设备都以文件的形式进行访问，这些文件存放在 / dev 目录下，一个文件就是一个设备节点，如下图  
![](http://imgsrc.baidu.com/super/pic/item/dcc451da81cb39dbc797769495160924aa183098.jpg)

在 Linux kernel 中使用结构体 device 描述一个设备，该结构体定义于 include/linux/device.h（内核源码路径）中，每个设备在内核中都有着其对应的 device 实例，其中记录着设备的相关信息

在 DTS（Device Tree Source，设备树）中则使用 device_node 结构体表示一个设备

#### 设备类（struct class）

在 Linux kernel 中使用结构体 class 用以表示高层次抽象的设备，该结构体定义于 include/linux/device/class.h 中

每个设备节点实例中都应当包含着一个指向相应设备类实例的指针

设备的注册与注销  
方便起见，我们接下来将会注册一个字符型设备，大致的一个步骤如下：

使用由内核提供的函数`register_chrdev(unsigned int major, const char *name, const struct file_operations *fops)`进行字符型设备注册，该函数定义于 include/linux/fs.h，会将注册成功后的主设备号返回，若失败则会返回一个负值，参数说明如下：  
major：主设备号，若为 0 则由内核分配主设备号  
name：设备名，由用户指定  
fops：该设备的文件操作系统（file_operations 结构体）指针  
使用宏 class_create(owner, name) 创建设备类，该宏定义于 include/linux/device.h 中，其核心调用函数是__class_create(struct module _owner, const char_ name, struct lock_class_key *key)

使用函数`device_create(struct class *cls, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...)`创建设备节点，若成功则最终会在 / dev 目录下生成我们的设备节点文件，各参数说明如下：

cls：该设备的设备类  
parent：该设备的父设备节点，通常情况下应当为某种总线或主机控制器，若该设备为顶级设备则设为 NULL  
devt：该设备的设备号  
drvdata：该驱动的相关信息，若无则填 NULL  
fmt：设备名称  
设备的注销则是逆着上面的进程进行，同样有着相对应的三个函数：`device_destroy(struct class *cls, dev_t devt)`、`class_destroy(struct class *cls)`、`unregister_chrdev(unsigned int major, const char *name)`，用法相似，这里就不一一赘叙了

✳ 需要注意的是若是注册设备的进程中的某一步出错了，我们在退出内核态函数之前应当手动调用注销函数清理原先的相关资源

#### 设备权限

内核模块运行在内核空间，所创建的设备节点只有 root 用户才有权限进行读写，对于其他用户而言便毫无意义，这并不是我们想要的，因此我们需要通过进一步的设置使得所有用户都有权限通过设备节点文件与我们的内核模块进行交互

在内核中使用 inode 结构体表示一个文件，该结构体定义于 include/linux/fs.h 中，其中用以标识权限的是成员 i_mode

而在内核中对于使用 flip_open() 打开的文件，Linux 内核中使用 file 结构体进行描述，该结构体定义于 include/linux/fs.h 中，其中有着指向内核中该文件的 inode 实例的指针，使用 file_inode() 函数可以获得一个 file 结构体中的 inode 结构体指针

那么我们不难想到，若是在内核模块中使用 file_open() 函数打开我们的设备节点文件，随后修改 file 结构体中的 inode 指针指向的 inode 实例的 i_mode 成员，便能够修改该文件的权限,

大伙可以照着敲一敲，逻辑很简单。

```
#include<linux/module.h>        //it have to exist
#include<linux/kernel.h>        //loading the information of kernel
#include<linux/init.h>          //contain some useful define
#include<linux/fs.h>            
#include<linux/device.h>

#define DEVICE_NAME "peiwithhao"
#define DEVICE_PATH "/dev/peiwithhao"
#define CLASS_NAME "P_Wmodule"

static int major_num;
static struct class * module_class = NULL;
static struct device * module_device = NULL;
static struct file * __file = NULL;
struct inode * __inode = NULL;

static struct file_operations PW_module_fo = {                          //descripe the device
    .owner = THIS_MODULE
};

static int __init kernel_module_init(void){
    printk(KERN_INFO "[peiwithhao_TestModule:]Module loaded. Start to register device ...\n");
    major_num = register_chrdev(0,DEVICE_NAME,&PW_module_fo);           //register the major number
    if(major_num <0){
        printk(KERN_INFO "[peiwithhao_TestModule:] Failed to register a major number! \n");
        return major_num;
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Register completed ,major number: %d\n",major_num);

    module_class = class_create(THIS_MODULE,CLASS_NAME);                //create the struct class
    if(IS_ERR(module_class)){
        unregister_chrdev(major_num,DEVICE_NAME);
        printk(KERN_INFO "[peiwithhao_TestModule:] Failed to register class device!\n");
        return PTR_ERR(module_class);
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Class Register complete. \n");

    module_device = device_create(module_class,NULL,MKDEV(major_num,0),NULL,DEVICE_NAME);   //create the device
    if(IS_ERR(module_class)){
        class_destroy(module_class);
        unregister_chrdev(major_num,DEVICE_NAME);
        printk(KERN_INFO "[peiwithhao_TestModule:] Failed to create the device! \n");
        return PTR_ERR(module_class);
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Module Register complete. \n");

    __file = filp_open(DEVICE_PATH, O_RDONLY,0);                        //open the file ,now that device
    if(IS_ERR(__file)){
        device_destroy(module_class,MKDEV(major_num,0));
        class_destroy(module_class);
        unregister_chrdev(major_num,DEVICE_NAME);
        printk(KERN_INFO "[peiwithhao_TestModule:] Unable to change module privilege! \n");
        return PTR_ERR(__file);
    }
    __inode = file_inode(__file);   
    __inode->i_mode |= 0666;             
    filp_close(__file,NULL);
    printk(KERN_INFO "[peiwithhao_TestModule:] Module privilege change complete.... \n");
    return 0;
}
static void __exit kernel_module_exit(void){
    printk(KERN_INFO "[peiwithhao_TestModule:] Start to clean up the module... \n");
    device_destroy(module_class,MKDEV(major_num,0));
    class_destroy(module_class);
    unregister_chrdev(major_num,DEVICE_NAME);
    printk(KERN_INFO "[peiwithhao_TestModule:] Module clean up complete. See you next time! \n");

}
module_init(kernel_module_init);        //in
module_exit(kernel_module_exit);        //out
MODULE_LICENSE("GPL");
MODULE_AUTHOR("dawn");


```

之后就是编译 make，然后测试，在这里咱们注册了一个 device，所以咱们也可以到 / dev 目录底下查看  
dmesg 查看效果  
![](http://imgsrc.baidu.com/super/pic/item/37d12f2eb9389b5054bcf190c035e5dde6116eb4.jpg)

> 这里注意下，为什么都要先 lsmod 再 rmmod 呢，因为咱们在注册的时候会调用 init 那个函数，卸载时会调用 exit，由于需要都看看进程所以一起用来算了

#### 3. 编写系统调用接口

我们编写如下的三个简单的函数使得用户应用程式可以通过 open、close、read、write、ioctl 与其进行交互

在这里我们引入了自旋锁 spinlock_t 类型变量以增加对多线程的支持

需要注意的是 file_operations 结构体中 ioctl 的函数指针应当为 unlocked_ioctl，close 对应的函数指针应当为 release

还有就是内核空间与用户空间之间传递数据应当使用 copy_from_user(void _to, const void_ from, unsigned long n)、copy_to_user(void _to, const void_ from, unsigned long n) 函数，从函数名咱们就可以知道他的妙用。

代码如下，我稍作解释，对了这里分出了头文件，因为文件太长了，我一切跟着师傅走然后慢慢理解  
首先是 p_wmodule.h

```
#include<linux/module.h>        //it have to exist
#include<linux/kernel.h>        //loading the information of kernel
#include<linux/init.h>          //contain some useful define
#include<linux/fs.h>            
#include<linux/device.h>

#define DEVICE_NAME "peiwithhao"
#define DEVICE_PATH "/dev/peiwithhao"
#define CLASS_NAME "p_wmodule"
#define NOT_INIT 0xffffffff
#define READ_ONLY 0x1000
#define ALLOW_WRITE 0x1001
#define BUFFER_RESET 0x1002

static int major_num;
static int p_w_module_mode = READ_ONLY;
static struct class * module_class = NULL;
static struct device * module_device = NULL;
static void * buffer = NULL;
static spinlock_t spin;
static struct file * __file = NULL;
struct inode * __inode = NULL;

static int __init kernel_module_init(void);
static void __exit kernel_module_exit(void);
static int p_w_module_open(struct inode *,struct file *);
static ssize_t p_w_module_read(struct file *,char __user *,size_t,loff_t *);
static ssize_t p_w_module_write(struct file*,const char __user * ,size_t,loff_t *);
static int p_w_module_release(struct inode *, struct file *);
static long p_w_module_ioctl(struct file *,unsigned int,unsigned long);
static long __internal_p_w_module_ioctl(struct file * __file,unsigned int cmd,unsigned long param);

static struct file_operations PW_module_fo = {                          //descripe the device
    .owner = THIS_MODULE,
    .unlocked_ioctl = p_w_module_ioctl,
    .open = p_w_module_open,
    .read = p_w_module_read,
    .write = p_w_module_write,
    .release = p_w_module_release,
};


```

再者之后就是 p_wmodule.c 了

```
#include<linux/module.h>        //it have to exist
#include<linux/kernel.h>        //loading the information of kernel
#include<linux/init.h>          //contain some useful define
#include<linux/fs.h>            
#include<linux/device.h>
#include<linux/slab.h>
#include "p_wmodule.h"

module_init(kernel_module_init);        //in
module_exit(kernel_module_exit);        //out
MODULE_LICENSE("GPL");
MODULE_AUTHOR("dawn");

static int __init kernel_module_init(void){
    spin_lock_init(&spin);
    printk(KERN_INFO "[peiwithhao_TestModule:]Module loaded. Start to register device ...\n");
    major_num = register_chrdev(0,DEVICE_NAME,&PW_module_fo);           //register the major number
    if(major_num <0){
        printk(KERN_INFO "[peiwithhao_TestModule:] Failed to register a major number! \n");
        return major_num;
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Register completed ,major number: %d\n",major_num);

    module_class = class_create(THIS_MODULE,CLASS_NAME);                //create the struct class
    if(IS_ERR(module_class)){
        unregister_chrdev(major_num,DEVICE_NAME);
        printk(KERN_INFO "[peiwithhao_TestModule:] Failed to register class device!\n");
        return PTR_ERR(module_class);
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Class Register complete. \n");

    module_device = device_create(module_class,NULL,MKDEV(major_num,0),NULL,DEVICE_NAME);   //create the device
    if(IS_ERR(module_class)){
        class_destroy(module_class);
        unregister_chrdev(major_num,DEVICE_NAME);
        printk(KERN_INFO "[peiwithhao_TestModule:] Failed to create the device! \n");
        return PTR_ERR(module_class);
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Module Register complete. \n");

    __file = filp_open(DEVICE_PATH, O_RDONLY,0);                        //open the file ,now that device
    if(IS_ERR(__file)){
        device_destroy(module_class,MKDEV(major_num,0));
        class_destroy(module_class);
        unregister_chrdev(major_num,DEVICE_NAME);
        printk(KERN_INFO "[peiwithhao_TestModule:] Unable to change module privilege! \n");
        return PTR_ERR(__file);
    }
    __inode = file_inode(__file);   
    __inode->i_mode |= 0666;             
    filp_close(__file,NULL);
    printk(KERN_INFO "[peiwithhao_TestModule:] Module privilege change complete.... \n");

    return 0;
}
static void __exit kernel_module_exit(void){
    printk(KERN_INFO "[peiwithhao_TestModule:] Start to clean up the module... \n");
    device_destroy(module_class,MKDEV(major_num,0));
    class_destroy(module_class);
    unregister_chrdev(major_num,DEVICE_NAME);
    printk(KERN_INFO "[peiwithhao_TestModule:] Module clean up complete. See you next time! \n");
}

static long p_w_module_ioctl(struct file * __file, unsigned  int cmd , unsigned long param){
    long ret;

    spin_lock(&spin);

    ret = __internal_p_w_module_ioctl(__file , cmd, param);

    spin_unlock(&spin);

    return ret;
}

static long __internal_p_w_module_ioctl(struct file *__file,unsigned int cmd, unsigned long param)
{
    printk(KERN_INFO "[peiwithhao_TestModule:] Received operation code : %d\n",cmd);
    switch(cmd){
        case READ_ONLY:
            if(!buffer){
                printk(KERN_INFO "[peiwithhao_TestModule:] Please reset the buffer at first!\n");
                return -1;
            }
            printk(KERN_INFO "[peiwithhao_TestModule:] Module operation mode reset to READ_ONLY...\n");
            p_w_module_mode = READ_ONLY;
            break;
        case ALLOW_WRITE:
            if(!buffer){
                printk(KERN_INFO "[peiwithhao_TestModule:] Please reset the buffer at first!\n");
                return -1;
            }
            printk(KERN_INFO "[peiwithhao_TestModule:] Module operation mode reset to ALLOW_WRITE..\n");
            p_w_module_mode = ALLOW_WRITE;
            break;
        case BUFFER_RESET:
            if(!buffer){
                buffer = kmalloc(0x500,GFP_ATOMIC);
                if(buffer == NULL){
                    printk(KERN_INFO "[peiwithhao_TestModule:] Unable to initialize the buffer. Kernel malloc error!\n");
                    p_w_module_mode = NOT_INIT;
                    return -1;
                }
            }
            printk(KERN_INFO "[peiwithhao_TestModule:] Buffer reset . Module operation mode reset to READ_ONLY...\n");
            memset(buffer,0,0x500);
            p_w_module_mode = READ_ONLY;
            break;
        case NOT_INIT:
            printk(KERN_INFO "[peiwithhao_TestModule:] Module operation mode reset to NOT_INIT...");
            p_w_module_mode = NOT_INIT;
            kfree(buffer);
            buffer = NULL;
            return 0;
        default:
            printk(KERN_INFO "[peiwithhao_TestModule:] Invalid operation code\n");
            return -1;
        }
    return 0;
}

static int p_w_module_open(struct inode * __inode, struct file * __file){
    spin_lock(&spin);

    if(buffer == NULL){
        buffer = kmalloc(0x500,GFP_ATOMIC);
        if(buffer == NULL){
            printk(KERN_INFO "[peiwithhao_TestModule:] Unable to initialize the buffer. Kernel malloc error!\n");
            p_w_module_mode = NOT_INIT;
            return -1;
        }
        memset(buffer,0,0x500);
        p_w_module_mode = READ_ONLY;
        printk(KERN_INFO "[peiwithhao_TestModule:] Device open,buffer initialized successfully...\n");
    }
    else{
        printk(KERN_INFO "[peiwithhao_TestModule:] Warning: reopen the device may cause unexpected error in kernel!\n");
    }
    spin_unlock(&spin);

    return 0;
}

static int p_w_module_release(struct inode * __inode, struct file * __file){
    spin_lock(&spin);

    if(buffer){
        kfree(buffer);
        buffer = NULL ;
    }
    printk(KERN_INFO "[peiwithhao_TestModule:] Device closed\n");
    spin_unlock(&spin);
    return 0;
}

static ssize_t p_w_module_read(struct file * __file ,char __user * user_buf,size_t size,loff_t *__loff){
    const char * const buf = (char*)buffer;
    int count;

    spin_lock(&spin);

    if(p_w_module_mode == NOT_INIT){
        printk(KERN_INFO "[peiwithhao_TestModule:] Module operation mode reset to NOT_INIT...");
        return -1;
    }
    count = copy_to_user(user_buf,buf,size > 0x500 ? 0x500 :size);
    spin_unlock(&spin);

    return count;
}
static ssize_t p_w_module_write(struct file * __file ,const char __user * user_buf,size_t size,loff_t *__loff){
    const char * const buf = (char*)buffer;
    int count;

    spin_lock(&spin);

    if(p_w_module_mode == NOT_INIT){
        printk(KERN_INFO "[peiwithhao_TestModule:] Module operation mode reset to NOT_INIT...");
        count =  -1;
    }
    else if(p_w_module_mode == READ_ONLY){
        printk(KERN_INFO "[peiwithhao_TestModule:] Unable to write under the mode READ_ONLY");
        count = -1;
    }else
        count = copy_from_user(buf,user_buf,size > 0x500?0x500 : size);

    spin_unlock(&spin);

    return count;

}


```

这里强烈建议大家跟着码一便，代码不是很长，在写的过程中你就可以懂这里的机制了。  
这里我讲解一下，当我们注册了这个设备后，由于咱们再 file_operations 中已经定义了系统调用的函数指针，所以此时也就是调用咱们的实现了，就这么简单，然后这里的 ioctl 就是设置通信的权限等了。  
咱们来编译试试看  
![](http://imgsrc.baidu.com/super/pic/item/8435e5dde71190ef577c831d8b1b9d16fcfa6068.jpg)  
![](http://imgsrc.baidu.com/super/pic/item/c2cec3fdfc039245b233f0cdc294a4c27c1e2569.jpg)

#### 4. 测试一下咱们写的'驱动'

c 代码如下，十分简单

```
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<string.h>
#include<fcntl.h>
#include<sys/ioctl.h>

char * buf = "test for read and write..";

int main(void){
    char ch[0x100];
    int fd = open("/dev/peiwithhao",2);
    int len = strlen(buf);
    ioctl(fd,0x1000,NULL);          //READ_ONLY

    write(fd,buf,len);
    ioctl(fd,0x1001,NULL);          //ALLOW_WRITE
    write(fd,buf,len);
    read(fd,ch,len);
    write(0,ch,len);

    ioctl(fd,0x1002,NULL);          //BUFFER_RESET
    read(fd,ch,len);
    write(0,ch,len);

    close(fd);
    return 0;

}


```

简单链接后执行，  
![](http://imgsrc.baidu.com/super/pic/item/bd315c6034a85edfb973efe10c540923dc547570.jpg)  
大伙可能初看没什么，但是注意这里咱们并没用使用 printf 函数，这里的输出是再内核中将我们输入缓冲区的值再输出出来。  
我们再用 dmesg 看看  
![](http://imgsrc.baidu.com/super/pic/item/35a85edf8db1cb137f9c2dd19854564e93584b7e.jpg)  
大获全胜！！！！

### 总结

经过一晚上的折腾，对于内核编程有了初步的认知，不会像之前那样摸不着头脑，在这里感谢 arttnba3 师傅博客的指点。

> 师傅的隐秘小屋  
> [https://arttnba3.cn/](https://arttnba3.cn/)

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 牛逼，收藏了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 牛逼哄哄 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) GodfatherTT 太牛逼了，收藏 ![](https://avatar.52pojie.cn/data/avatar/000/21/34/31_avatar_middle.jpg) yuyi0 先收藏，慢慢再看