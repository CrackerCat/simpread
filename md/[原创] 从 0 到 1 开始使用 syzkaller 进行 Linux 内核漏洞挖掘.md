> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265405.htm#msg_header_h1_4)

尝试从 0 到 1 开始使用 syzkaller 进行 Linux 内核漏洞挖掘
========================================

目录

*   [尝试从 0 到 1 开始使用 syzkaller 进行 Linux 内核漏洞挖掘](#尝试从0到1开始使用syzkaller进行linux内核漏洞挖掘)
*   [环境搭建过程（吐血踩坑）](#环境搭建过程（吐血踩坑）)
*                    [编译 syzkaller](#编译syzkaller)
*                    [拉取 linux 主线源码](#拉取linux主线源码)
*                    [Image](#image)
*                    [过程总结（重点）](#过程总结（重点）)
*   Syzkaller Overview
*   Syscall descriptions
*   [尝试使用 Syzkaller 捕捉一个简单的内核堆溢出](#尝试使用syzkaller捕捉一个简单的内核堆溢出)
*            [编译有漏洞的驱动到内核中](#编译有漏洞的驱动到内核中)
*            [添加对应的 syzkaller 规则](#添加对应的syzkaller规则)
*            [效果](#效果)
*   [参考](#参考)

[](#环境搭建过程（吐血踩坑）)环境搭建过程（吐血踩坑）
=============================

**觉得前面踩坑的过程繁琐可以直接去看过程总结**  
整个环境搭建的过程踩了很多坑。有不少是网上没提到的，于是我详细记录了一下，希望能帮到以后踩坑的同学。

### 编译 syzkaller

这里我使用了一个**完全全新的 Ubuntu18.04** 来搭建环境。

 

内存分配 40G。

 

安装基本的软件

```
sudo apt-get install debootstrap
sudo apt install qemu-kvm
sudo apt-get install subversion
sudo apt-get install git
sudo apt-get install make
sudo apt-get install qemu
sudo apt install libssl-dev libelf-dev
sudo apt-get install flex bison libc6-dev libc6-dev-i386 linux-libc-dev linux-libc-dev:i386 libgmp3-dev libmpfr-dev libmpc-dev
apt-get install g++
apt-get install build-essential
apt install golang-go
apt install gcc

```

注意，此时我的 gcc 版本是 `gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)`

 

接下来尝试

```
go get -u -d github.com/google/syzkaller/prog

```

然而这一步慢的离谱。。我查了一下 - u -d 的选项，似乎跟直接 git clone 没什么区别。

 

于是我换了个方式，直接使用 git clone 拉取源码.

```
git clone https://github.com/google/syzkaller /usr/local/golang/src/github.com/google/syzkaller

```

然后也慢的离谱。

 

于是我直接在宿主机上下载了 syzkaller.zip，然后复制到虚拟机中直接 unzip 解压。

 

然后直接 make，发现报错：

 

![](https://bbs.pediy.com/upload/attach/202101/876323_BU9NHQ9U2ZGRJJQ.jpg)

 

使用 `dmesg | egrep -i -B100 'killed process'` 查看：

 

发现触发了 OOM-killer ，内存不够。

 

![](https://bbs.pediy.com/upload/attach/202101/876323_SE2C29PYE2HHE2R.jpg)

 

按照网上的方法建立了一个 swap 分区：

 

https://studygolang.com/articles/11781?fr=sidebar

 

重新编译又报错：

```
vm/vmimpl/merger.go:69:12: undefined: bytes.ReplaceAll

```

似乎是因为 golang 版本太低。（1.10）

 

于是使用 wget 下载新版本的 golang。进行如下操作：

```
wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz
tar -xf go1.14.2.linux-amd64.tar.gz
mv go goroot
mkdir gopath
export GOPATH=/root/gopath
export GOROOT=/root/goroot
export PATH=$GOPATH/bin:$PATH
export PATH=$GOROOT/bin:$PATH #把这几行加到.bashrc中

```

root@ubuntu:~/gopath/src/github.com/google/syzkaller# **go version**  
**go version go1.14.2 linux/amd64**

 

接下来把 syzkaller.zip 在：`/root/gopath/src/github.com/google/` 下解压。

 

make！

 

又报 OOM。

 

于是我查看了一下，似乎是因为同时编译多个文件造成的？

 

于是我尝试先单独编译一下第一个文件：

```
GOOS=linux GOARCH=amd64 go build "-ldflags=-s -w -X github.com/google/syzkaller/prog.GitRevision= -X 'github.com/google/syzkaller/prog.gitRevisionDate='" -o ./bin/syz-manager github.com/google/syzkaller/syz-manager

```

然后在 `./bin/` 下看到了编译好的 `syz-manager`

 

于是在进行 make。

 

似乎是成功了。.git 产生的 fatal 不用理他。

 

![](https://bbs.pediy.com/upload/attach/202101/876323_QWETWC8EFTACAAH.jpg)

 

在 `./bin/` 下我们看到了如下文件：

 

![](https://bbs.pediy.com/upload/attach/202101/876323_7Z8UEWYW95GH88Z.jpg)

 

应该是编译好了。大概说一下这几个 syz-* 都是干嘛的：

 

总结一下我此时的环境：

```
Ubuntu18.04，内核5.4.0-42-generic
 
 
gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)
 
 
go version go1.14.2 linux/amd64

```

### 拉取 linux 主线源码

克隆主线 linux 代码，使用

```
https://mirrors.tuna.tsinghua.edu.cn/git/linux.git

```

如果太慢可以或者直接去： https://gitee.com/mirrors/linux/repository/archive/master.zip

 

下载解压好后。

 

运行如下命令：

```
1.首先：
make CC="/usr/bin/gcc" defconfig
make CC="/usr/bin/gcc" kvmconfig    （这个选项没设置成功？）
 
2.在.config文件中添加
CONFIG_KCOV=y
CONFIG_DEBUG_INFO=y
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_CONFIGFS_FS=y
CONFIG_SECURITYFS=y
 
3.然后：
make CC="/usr/bin/gcc" olddefconfig
make CC="/usr/bin/gcc" -j64

```

编译完就生成了内核，不过这次不同于我们之前 make bzImage。这次应该是驱动也编译了。

### Image

```
sudo apt-get install debootstrap

```

然后建立一个 image 文件夹，cd 进去

```
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
 
chmod +x create-image.sh
 
./create-image.sh

```

但我这里 wget 老是失败，于是我直接在这里：https://github.com/google/syzkaller/blob/master/tools/create-image.sh

 

copy 了一份下来运行。

 

实际上就是先获取 arch 信息，然后使用 **debootstrap** 来构建基本的文件系统。

 

这一步也很慢，不过我寻思文件系统应该可以换成 CTF 题的 rootfs.img，毕竟只要内核对了就行。

 

漫长的等待后：

```
root@ubuntu:~/source/linux/image# ls
chroot  create-image.sh  stretch.id_rsa  stretch.id_rsa.pub  stretch.img

```

可以看到出现了：**stretch.id_rsa、stretch.id_rsa.pub、stretch.img** 这几个文件。

 

接下来安装 qemu 环境：

```
sudo apt-get install qemu-system-x86

```

一般 kvm 什么的都是自带的。不过如果是 vmware 可能需要开启一下**虚拟化引擎** 里的 `虚拟化 Intel VT-x/EPT 或 AMD-V/RVI(V)`

 

就在虚拟机的设置界面中。

 

boot.sh 如下:

```
qemu-system-x86_64 \
 -kernel /root/source/linux/arch/x86/boot/bzImage \
 -append "console=ttyS0 root=/dev/sda debug earlyprintk=serial slub_debug=QUZ"\
 -hda ./stretch.img \
 -net user,hostfwd=tcp::10021-:22 -net nic \
 -enable-kvm \
 -nographic \
 -m 256M \
 -smp 2 \
 -pidfile vm.pid \
 2>&1 | tee vm.log

```

启动起来后显示：

 

root@syzkaller:~#

 

然后 poweroff 掉。

 

回到 root@ubuntu:~/gopath/src/github.com/google/syzkaller#

 

新建 my.cfg 文件

 

然后尝试运行 syz-manger 发现报错：

```
2021/01/16 20:57:54 bad syz-manager build: build with make, run bin/syz-manager

```

猜测是 .git ？的问题，因为我在源码中发现了这样一行：

```
if prog.GitRevision == "" {
        log.Fatalf("bad syz-manager build: build with make, run bin/syz-manager")
    }

```

于是尝试 `git init` 一下，重新编译。

 

![](https://bbs.pediy.com/upload/attach/202101/876323_CV2WQJKYFK4XF7Z.jpg)

 

这次启动起来了。但是 http panic 了。

 

定位问题：

```
2021/01/17 01:41:29 http: panic serving 127.0.0.1:56498: runtime error: slice bounds out of range [:8] with length 4
goroutine 53 [running]:
net/http.(*conn).serve.func1(0xc00087e780)
    /root/goroot/src/net/http/server.go:1772 +0x139
panic(0x10f68c0, 0xc0004f74a0)
    /root/goroot/src/runtime/panic.go:975 +0x3e3
main.(*Manager).collectStats(0xc000100820, 0x0, 0x0, 0x0)
    /root/gopath/src/github.com/google/syzkaller/syz-manager/html.go:111 +0x106f

```

切片越界了。本来只有 length=4，但是切到了 8。

 

看一下这部分的代码:

 

![](https://bbs.pediy.com/upload/attach/202101/876323_YKPSUJQ22M7PQZT.jpg)

 

这里似乎是一个获取版本的操作。尝试了一下把这个地方从 8 改成 4 就可以跑了。但是显示如下

 

![](https://bbs.pediy.com/upload/attach/202101/876323_6ACWSMMQN46ZY4H.jpg)

 

首先看这个 revision 这里，确实是一个长度为 4 的字符串，所以切片越界了。

 

但是底下都显示的 0，没跑起来啊。。。

 

于是我重新在宿主机里挂了一下 git 的代理：

```
git config --global http.proxy socks5://192.168.80.1:7890
git config --global https.proxy socks5://192.168.80.1:7890
git config --global http.sslVerify false

```

然后绑定到远程的 git 仓库。pull 下来，处理冲突。

```
git pull origin master

```

然后重新 make。启动。

 

然而还是这样子。（泪）

 

突然发现 qemu 启动的时候似乎在这里有一些问题，Kernel File Systems mount 不上去：

```
[FAILED] Failed to mount /sys/kernel/config.
See 'systemctl status sys-kernel-config.mount' for details.
[DEPEND] Dependency failed for Local File Systems.
[DEPEND] Dependency failed for Mark the need to relabel after reboot.
[  OK  ] Reached target Timers.
[    2.878286] EXT4-fs (sda): re-mounted. Opts: (null). Quota mode: none.
[  OK  ] Closed Syslog Socket.
[  OK  ] Started Emergency Shell.
[  OK  ] Reached target Emergency Mode.
[  OK  ] Reached target Login Prompts.
[  OK  ] Started Load Kernel Modules.
[FAILED] Failed to start Remount Root and Kernel File Systems.

```

于是查找解决方案：

 

https://github.com/google/syzkaller/issues/760

 

按照上面的，删掉那些注释。

 

重新编译内核生成 bzImage

 

成功！！！

 

![](https://bbs.pediy.com/upload/attach/202101/876323_US32WJRUCRSKM2T.jpg)

### [](#过程总结（重点）)过程总结（重点）

1. 网上很多说是必须 gcc 8 的版本，实际并不需要，直接 apt 安装 gcc 即可。但是 golang 版本最好是最新的。当然你可以先 apt install golang-go，然后手动升级到最新版本就行。

 

2.go get -u -d 效果跟直接 git clone 一样的。如果 git clone 太慢可以挂一下代理。

 

3. 如果你是直接下载的. zip，记得先 git init 一下，然后绑定到远程仓库。

 

4. **注意编译内核的时候要添加那些选项，并且要把下面那些选项对应的注释删掉！！（比如：# CONFIG_KCOV is not set）要不然会被 rewrite 掉。。**

Syzkaller Overview
==================

官方给出了一个如下的 overview 图片。

 

![](https://bbs.pediy.com/upload/attach/202101/876323_RJJ5GV45J8E76HS.png)

*   **syz-manager** 是起了一个监控者的作用。他启动多个 VM instance（也就是黄色框），同时通过远程过程调用，在 VM 中启动 **syz-fuzzer**
*   **syz-fuzzer** 在 VM 内部运行。syz-fuzzer 负责引导整个 fuzz 的过程（input generation, mutation, minimization, etc.）。并且通过 RPC 将那些引起了新的覆盖（coverage）的输入回送到 **syz-manager**。**syz-fuzzer** 也负责启动短暂的 **syz-executor** 进程。
*   每个 **syz-executor** 进程都负责执行单个输入（一系列的 syscall）。**syz-executor** 接受从 **syz-fuzzer** 生成的 input，然后执行，最后回送结果。这部分的设计要尽可能的简化。不干扰 fuzz 过程。用 C ++ 编写，编译为静态二进制文件并使用共享内存进行通信。

Syscall descriptions
====================

Syzkaller 采用一套自己的系统调用的描述 or 声明（syzlang），来操纵 fuzz 的系统调用序列。

 

官方给出的语法如下：https://github.com/google/syzkaller/blob/master/docs/syscall_descriptions_syntax.md

 

这里有已经声明好的：https://github.com/google/syzkaller/blob/master/sys/linux/sys.txt

```
syscallname "(" [arg ["," arg]*] ")" [type] ["(" attribute* ")"]
arg = argname type
argname = identifier
type = typename [ "[" type-options "]" ]
typename = "const" | "intN" | "intptr" | "flags" | "array" | "ptr" |
       "string" | "strconst" | "filename" | "len" |
       "bytesize" | "bytesizeN" | "bitsize" | "vma" | "proc"
type-options = [type-opt ["," type-opt]]

```

具体的过程是：

 

使用 **syz-extract** 得到常量和值一一对应的. const 文件（例如 / sys/linux/tty.txt 被转换为 sys/linux/tty_amd64.const），然后使用 **syz-sysgen** 编译 AST(Abstract Syntax Tree，抽象语法树) 和常量值，并返回包含生成的 prog 对象的 Prog（根据系统调用模板和第一步中生成的 const 文件使用 syz-sysgen 生成 syzkaller 用的 go 代码）。**syz-sysgen** 具体又分为下面 4 步。  
    1.**assignSyscallNumbers**：分配系统调用号，检测不受支持的系统调用并丢弃  
    2.**patchConsts**：将 AST 中的常量 patch 成对应的值  
    3.**check**：对 AST 进行语义检查  
    4.**genSyscalls**：从 AST 生成 prog 对象

 

**以上是先知上的 [houjingyi](https://xz.aliyun.com/u/4572) 大师傅分析总结的。对应的源码分析的文章链接如下，我这个初学者就不献丑了。。**

 

[内核漏洞挖掘技术系列 (4)——syzkaller(2)](https://xz.aliyun.com/t/5098)

尝试使用 Syzkaller 捕捉一个简单的内核堆溢出
===========================

这部分主要参考 ：

1.  [bsauce](https://www.jianshu.com/u/a12c5b882be2)
    
2.  https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/tree/master/docs/harbian_qa/fuzz_testing
    
3.  [使用 Syzkaller&QEMU 捕捉内核堆溢出 Demo](http://embedsec.systems/zh/gnulinux-security/2017/06/05/syzkaller-demo.html)
    

整体过程大概如下：

编译有漏洞的驱动到内核中
------------

test.c 中有一个堆溢出的 demo。我们将他编译然后 insmod 上去。

```
static ssize_t proc_write (struct file *proc_file, const char __user *proc_user, size_t n, loff_t *loff)
{
    char *c = kmalloc(512, GFP_KERNEL);
 
    copy_from_user(c, proc_user, 4096);        //堆溢出
    printk(DEBUG_FLAG":into write!\n");
    return 0;
}

```

当我们想要尝试编译内核模块的时候，涉及到一个 linux header 的问题。（比如说我在 5.4.0-62-generic 的系统下编译 5.11.0-rc3 的驱动）

 

我的解决方案是：

```
在/lib/modules/ 下mkdir一个5.11.0-rc3/
然后cd 5.11.0-rc3/
mkdir kernel
ln -s /root/source/linux ./source
ln -s /root/source/linux ./build

```

然后 make。这里发现了一个问题。

 

源码中有这样几行：

```
static struct file_operations a = {
                                .open = proc_open,
                                .read = proc_read,
                                .write = proc_write,
};
 
 
static int __init mod_init(void)
{
    ......
    const struct file_operations *proc_fops = &a;
    ......
 
    test_entry = proc_create(MY_DEV_NAME, S_IRUGO|S_IWUGO, NULL, proc_fops);
    ......
}

```

但是编译的时候会报：

```
/root/fuzz/test.c:27:66: error: passing argument 4 of ‘proc_create’ from incompatible pointer type [-Werror=incompatible-pointer-types]
     test_entry = proc_create(MY_DEV_NAME, S_IRUGO|S_IWUGO, NULL, proc_fops);
                                                                  ^~~~~~~~~
In file included from /root/fuzz/test.c:3:0:
./include/linux/proc_fs.h:109:24: note: expected ‘const struct proc_ops *’ but argument is of type ‘const struct file_operations *’
 struct proc_dir_entry *proc_create(const char *name, umode_t mode, struct proc_dir_entry *parent, const struct proc_ops *proc_ops);

```

这个版本下：proc_create 的最后一个参数要求是 **const struct proc_ops *** 而不是旧的 const struct file_operations *。

 

解决方案参照：https://stackoverflow.com/questions/64931555/how-to-fix-error-passing-argument-4-of-proc-create-from-incompatible-pointer

 

最终我对 test.c 改动如下：

```
......
static struct proc_ops a = {                        //修改
                                .proc_open = proc_open,   
                                .proc_read = proc_read,
                                .proc_write = proc_write,
};
 
static int __init mod_init(void)
{
    struct proc_dir_entry *test_entry;
    //const struct file_operations *proc_fops = &a;    //不要这里
    const struct proc_ops *proc_fops = &a;            //修改
    printk(DEBUG_FLAG":proc init start!\n");
 
    test_entry = proc_create(MY_DEV_NAME, S_IRUGO|S_IWUGO, NULL, proc_fops);
    if(!test_entry)
       printk(DEBUG_FLAG":there is somethings wrong!\n");
 
    printk(DEBUG_FLAG":proc init over!\n");
    return 0;
}
......
module_init(mod_init);
MODULE_LICENSE("GPL");        //添加LICENSE

```

Makefile 如下：

```
CONFIG_MODULE_SIG=n
 
obj-m += test.o
 
EXTRA_CFLAGS += -fno-stack-protector -no-pie
all:
    make -C /lib/modules/5.11.0-rc3/build M=$(PWD) modules
clean:
    rm -rf *.ko
    rm -rf *.mod
    rm -rf *.mod.*
    rm -rf *.o
    rm .tmp_versions/test.mod
    rm .test.ko.cmd
    rm .test.mod.o.cmd
    rm .test.o.cmd
    rm Module.symvers
    rm modules.order

```

此时可以正常通过编译，生成 test.ko。说明可以正常编译了。

 

（**这种不用自己找 linux header，然后手动创建的方式屡试不爽 orz，很适合调试的时候错版本加载一些内核驱动，之前我调试内核 cve 的时候也用的这种方式**）

 

接下来我们把 test.c cp 到 / linux/drivers/char / 下，然后 vim /linux/drivers/char/Kconfig，添加如下：

```
config TEST_MODULE
        tristate "Heap Overflow Test"       
        default y                                           
        help
          This file is to test a buffer overflow.

```

然后 make -j64 编译。中途看了一眼，刚好发现了：

 

![](https://bbs.pediy.com/upload/attach/202101/876323_5CP4GAHJYE7ABYX.jpg)

 

然后启动：

```
root@syzkaller:/proc# ls | grep test
test

```

已经有了。

添加对应的 syzkaller 规则
------------------

1.

 

在 **syzkaller/sys/linux/** 下新建

 

对应的 **proc_operation.txt** 如下：

```
include open$proc(file ptr[in, string["/proc/test"]], flags flags[proc_open_flags], mode flags[proc_open_mode]) fd
read$proc(fd fd, buf buffer[out], count len[buf])
write$proc(fd fd, buf buffer[in], count len[buf])
close$proc(fd fd)
 
proc_open_flags = O_RDONLY, O_WRONLY, O_RDWR, O_APPEND, FASYNC, O_CLOEXEC, O_CREAT, O_DIRECT, O_DIRECTORY, O_EXCL, O_LARGEFILE, O_NOATIME, O_NOCTTY, O_NOFOLLOW, O_NONBLOCK, O_PATH, O_SYNC, O_TRUNC, __O_TMPFILE
proc_open_mode = S_IRUSR, S_IWUSR, S_IXUSR, S_IRGRP, S_IWGRP, S_IXGRP, S_IROTH, S_IWOTH, S_IXOTH 
```

这里我 **引用 bsauce 师傅** 的对于调用规则的讲解：

 

**调用规则**：$ 号前的 syscallname 是系统调用名，$ 号后的 type 是指特定类型的系统调用。如上文的 open$proc 指的就是 open 这个类调用中 proc 这个具体的调用，这个名字是由规则编写者确定的，具体行为靠的是后面的参数去确定。 参数的格式如下： ArgumentName ArgumentType[Limit] ArgumentName 是指参数名，ArgumentType 指的是参数类型，例如上述例子有 string、flags 等类型。[ ] 号中的内容就是具体的类型的值，不指定的时候由 syzkaller 自动生成，若要指定须在后文指定，以上文为例：

 

​ mode flags[proc_open_mode]

 

​ proc_open_mode = ...

 

​ 因为我们给的例子是通过 / proc/test 这个内核接口的写操作来触发堆溢出，因此我们需要控制的参数是 open 函数中的 file 参数为 “/proc/test” 即可，其他操作参考 sys.txt 即可。

 

2.

 

编译 syz-extract 和 syz-sysgen

```
make bin/syz-extract
make bin/syz-sysgen

```

接下来我们使用 **syz-extract** 生成 .const 文件：

```
bin/syz-extract -os linux -sourcedir "/root/source/linux" -arch amd64 proc_operation.txt

```

生成了 proc_operation.txt.const 内容如下：

```
# Code generated by syz-sysgen. DO NOT EDIT.
arches = amd64
FASYNC = amd64:8192
O_APPEND = amd64:1024
O_CLOEXEC = amd64:524288
O_CREAT = amd64:64
O_DIRECT = amd64:16384
O_DIRECTORY = amd64:65536
O_EXCL = amd64:128
O_LARGEFILE = amd64:32768
O_NOATIME = amd64:262144
O_NOCTTY = amd64:256
O_NOFOLLOW = amd64:131072
O_NONBLOCK = amd64:2048
O_PATH = amd64:2097152
O_RDONLY = amd64:0
O_RDWR = amd64:2
O_SYNC = amd64:1052672
O_TRUNC = amd64:512
O_WRONLY = amd64:1
S_IRGRP = amd64:32
S_IROTH = amd64:4
S_IRUSR = amd64:256
S_IWGRP = amd64:16
S_IWOTH = amd64:2
S_IWUSR = amd64:128
S_IXGRP = amd64:8
S_IXOTH = amd64:1
S_IXUSR = amd64:64
__NR_close = amd64:3
__NR_open = amd64:2
__NR_read = amd64:0
__NR_write = amd64:1
__O_TMPFILE = amd64:4194304

```

重新运行

```
root@ubuntu:~/gopath/src/github.com/google/syzkaller# bin/syz-sysgen

```

然后：

```
make clean
make all

```

重新编译 syzkaller。

 

最后我们修改 my.cfg

 

加上：

```
"enable_syscalls": [
                "open$proc",
                "read$proc",
                "write$proc",
                "close$proc"
],

```

最后

```
scp -P 10021 -i ./stretch.id_rsa -r /root/gopath/src/github.com/google/syzkaller/bin root@127.0.0.1:/root/bin

```

boot 起来虚拟机将其拷贝到虚拟机的 / root/bin 下。

效果
--

启动：`bin/syz-manager -config my.cfg -vv 10`  
一段时间后出现：

 

但不知为什么我这里是空指针未引用 orz。。不过确实是跑出来了 /proc/test 的洞。

 

![](https://bbs.pediy.com/upload/attach/202101/876323_4FCYM4SQMFWJTMC.png)  
还有这种的：  
![](https://bbs.pediy.com/upload/attach/202101/876323_6URC3ZPWDHRQB9G.png)

 

一段时间后：  
![](https://bbs.pediy.com/upload/attach/202101/876323_APUT8PWG8C8AR6D.png)  
产生了一个 c 报告。可以直接在里面看对应的产生漏洞 c 代码（syzkaller 生成的）

参考
==

[编译 delve 时报错 \"../../pkg/proc/native/proc_linux.go:170:16: undefined: strings.ReplaceAll\" 如何处理？](https://www.e-learn.cn/topic/3777256)

 

[解决 golang 编译项目时出现 signal: killed](https://studygolang.com/articles/11781?fr=sidebar)

 

[Setup: Ubuntu host, QEMU vm, x86-64 kernel](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md)

 

[ubuntu 系统 debootstrap 的使用](https://blog.csdn.net/whatday/article/details/88290434)

 

[Git clone 走 ss 代理](https://www.jianshu.com/p/adf7cca269ac)

 

[tools/create-image.sh: image does not boot in qemu](https://github.com/google/syzkaller/issues/760)

 

[在 Ubuntu 16.04.6 LTS 上升级 Go 到最新版 1.12.5 实录](https://blog.csdn.net/tao_627/article/details/90602733)

 

[Syzkaller](https://github.com/google/syzkaller)

 

[内核漏洞挖掘技术系列 (4)——syzkaller(2)](https://xz.aliyun.com/t/5098)

 

[【漏洞挖掘】使用 Syzkaller&QEMU 捕捉内核堆溢出 Demo](https://www.jianshu.com/p/790b733f80a2?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

 

[如何将一个驱动编译进内核](https://blog.csdn.net/qq_33487044/article/details/81949703)

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 3 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)

最后于 11 小时前 被 ScUpax0s 编辑 ，原因： test.c 为我修改好的文件

上传的附件：

*   [test.c](javascript:void(0)) （1.76kb，0 次下载）