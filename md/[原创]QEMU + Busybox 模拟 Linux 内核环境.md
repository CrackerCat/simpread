> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-264632.htm)

> [原创]QEMU + Busybox 模拟 Linux 内核环境

前言
--

最近转 Linux 平台，开始深入 Linux 内核相关，总结一下进行 Linux 内核环境模拟流程。结合 Linux 的内核源码一起，效果会比较好。

准备环境
----

### 主机环境

Ubuntu 18.04

 

Linux ubuntu 5.4.0-58-generic #64~18.04.1-Ubuntu SMP Wed Dec 9 17:11:11 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

### 需要使用的软件

使用主流的 qemu+busybox 进行模拟，底层的模拟实现软件内部完成，可以将重心放在内核调试上，避免在环境上浪费过多时间。qemu 模拟器原生即支持 gdb 调试器，所以可以方便地使用 gdb 的强大功能对操作系统进行调试。

1.  首先安装 qemu，依次执行以下命令：
    
    ```
    sudo apt-get install qemu
    sudo apt-get install qemu-system
    sudo apt-get install qemu-user-static
    
    ```
    
    这里不建议使用源码编译的方式进行安装，个人建议是节省时间在核心工作上，工具越快搭建好越能提升效率。源码编译涉及到编译器和主机环境各异性的问题，中间可能出现各种情况，浪费时间。（注意，安装好后，无法直接 qemu 无法运行，需要使用`qemu-system-i386, qemu-system-x86_64, qemu-system-arm`这种格式的命令进行运行。如果嫌麻烦，可以设置软链接。）
    
2.  安装 busybox，直接 busybox 的 github 上拖源码下来即可。在实际进行文件系统制作的时候再进行其他操作。
    
3.  最后是下载想进行编译的 Linux 内核源码，这里给出一个各个版本的 [Linux 内核源码集合](http://ftp.sjtu.edu.cn/sites/ftp.kernel.org/pub/linux/kernel/)。
    

编译调试版内核
-------

### 编译正常流程

首先对 Linux 内核进行编译：

```
cd linux-3.18.6
make menuconfig
make bzImage

```

注意，这里在进入`menuconfig`后，需要开启内核参数`CONFIG_DEBUG_INFO`和`CONFIG_GDB_SCRIPTS`。gdb 提供了 python 接口进行功能扩展，内核基于 python 接口实现了一系列辅助脚本来简化内核的调试过程。

```
Kernel hacking  --->
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        [*]   Provide GDB scripts for kernel debuggin

```

### 编译可能遇到的问题

执行 make bzImage 时遇到的问题：

1.  `fatal error: linux/compiler-gcc7.h: No such file or directory`
    
    提示缺少 compiler-gcc7.h 这个文件，是由于内核版本较低和 gcc 版本不匹配造成的有三种解决方法：
    
    ```
    1.在内核文件夹中include/linux目录下找到compiler-gcc4.h文件，不同内核版本可能不一样，也有可能是compiler-gcc3.h,将它重命名为compiler-gcc7.h。然后重新编译一下就好了。
     
    2.在新的内核源码中拷贝一个compiler-gcc7.h，将它拷贝到内核文件夹include/linux目录下，重新编译即可。
     
    3.重装一个版本低一点的gcc。
    
    ```
    
2.  `fatal error: asm/types.h: No such file or directory`
    
    linux 添加到 asm-generic 的软链接: `ln -s /usr/include/asm-generic asm`
    

制作 initramfs 根文件系统
------------------

Linux 启动阶段，boot loader 加载完内核文件 vmlinuz 之后，便开始挂载磁盘根文件系统。挂载操作需要磁盘驱动，所以挂载前要先加载驱动。但是驱动位于`/lib/modules`，不挂载磁盘就访问不到，形成了一个死循环。`initramfs`根文件系统就可以解决这个问题，其中包含必要的设备驱动和工具，boot loader 会加载 initramfs 到内存中，内核将其挂载到根目录，然后运行`/init`初始化脚本，去挂载真正的磁盘根文件系统。

### 编译 busybox

首先需要注意，busybox 默认编译的文件系统是和主机 OS 一样的位数，也就是 Ubuntu 是 x86 的，编译出的文件系统就是 x86 的，如果 Ubuntu 是 x64 的，编译出的文件系统是 x64 的。要保持前面编译的 Linux 内核和文件系统的位数一样。

```
cd busybox-1.32.0
make menuconfig
make -j 20
make install

```

进入 menu 后，修改参数如下：

 

![](https://bbs.pediy.com/upload/attach/202012/779730_QQ6U88Z6JA4H5DG.jpg)

 

其次，修改为静态链接：

```
Settings  --->
    [*] Build static binary (no shared libs)

```

然后再执行 make 和 install 操作。

### 创建 initramfs

编译成功后，会生成`_install`目录，其内容如下：

```
$ ls _install
bin  linuxrc  sbin  usr

```

依次执行如下命令：

```
mkdir initramfs
cd initramfs
cp ../_install/* -rf ./
mkdir dev proc sys
sudo cp -a /dev/{null, console, tty, tty1, tty2, tty3, tty4} dev/
rm linuxrc
vim init
chmod a+x init

```

其中`init`文件的内容如下：

```
#!/bin/busybox sh        
mount -t proc none /proc 
mount -t sysfs none /sys 
 
exec /sbin/init

```

在创建的 initramfs 中包含 busybox 可执行程序、必须的设备文件、启动脚本`init`，且`init`只挂载了虚拟文件系统`procfs`和`sysfs`，没有挂载磁盘根文件系统，所有操作都在内存中进行，不会落地。

 

最后打包 initramfs：

```
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz

```

启动内核
----

```
qemu-system-i386 -s -kernel /path/to/bzImage -initrd initramfs.cpio.gz -nographic -append "console=ttyS0"

```

参数说明：

*   `-s`是`-gdb tcp::1234`缩写，监听 1234 端口，在 gdb 中通过`target remote localhost:1234`连接；
*   `-kernel`指定编译好的内核；
*   `-initrd`指定 initramfs;
*   `-nographic`取消图形输出窗口；
*   `append "console=ttyS0"`将输出重定向到 console，将会显示在标准输出 stdio。

启动后的根目录，就是 initramfs 中包含的内容：

 

![](https://bbs.pediy.com/upload/attach/202012/779730_WZYVEMSG5ZYWR6W.jpg)

 

至此，一个简单的内核就算编译完成了，可以挂 gdb 进行调试了。

参考文献
----

https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/

个人博客
----

一些非大量干货的内容会发在自己 blog 上，争取在看雪发的都是干货比较多的内容。  
http://www.v4ler1an.com/

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)