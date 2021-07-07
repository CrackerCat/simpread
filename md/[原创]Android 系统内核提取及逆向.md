> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266625.htm)

> [原创]Android 系统内核提取及逆向

android 系统内核提取及逆向
=================

前言
--

该部分主要是涉及到 Android 设备的内核提取与逆向分析相关的知识点，通常是用于对内核对底层文件的分析。原本计划是从提取到逆向分析再到 POC 编写一条龙服务，能力有限，漏洞分析和 POC 编写就留到以后再分享。第一次发帖，请多多关照，能力有限，大神们轻点喷。

第一节 boot.img 的结构
----------------

在介绍 boot 文件之前，简单阐述一下 android 系统的分区

 

比较重要的几个分区如下：

 

1、/boot 分区：该分区主要包含 android kernel 镜像和 ramdisk（一种将 RAM 模拟为硬盘的技术，提高访问速度）。

 

2、/system 分区：该分区主要存放 Android 框架及其相关配置，包含系统预装的 app。

 

3、/recovery 分区：该分区主要是备份的分区

 

4、/data 用户数据的存储区域

```
/data/app/com.xxxx/ 以包名存放应用安装文件，包括base.apk /lib
/data/data/com.xxxx/ 存放应用数据，包括sp、db等
/data/dalvik-cache 以包名存放优化过的应用dex文件

```

5、/cache 分区：Android 系统缓存区域，保存系统最常访问的数据和应用程序。

 

6、/misc 分区：此分区包含一些系统功能设置开关和数据，比如 USB 设置。

 

7、sdcard 分区：外置存储分区

 

8、/vendor 分区：厂商定制的分区，厂商的某些系统升级可以通过这个分区来实现。

### boot 文件

boot.img 就是 android 系统的 Linux 内核主要的镜像文件，在该文件中大致包含 boot header，kernel，ramdisk。

 

boot.img 文件跳过 2K 的文件头之后，包含两个 gz 压缩包，一个是 boot.img-kernel.gz Linux 内核，一个是 boot.img-ramdisk.cpio.gz，然后加上 ramdisk 文件。

 

大致的结构图如下

 

![](https://bbs.pediy.com/upload/attach/202103/849527_BB97KK7ZN7U8ZXD.png)

 

值得注意的是我们的 boot.img 文件在针对 kernel 是有不同压缩算法来进行压缩的，在后面的实战环节中是有相关的案例。

第二节 提取内核及分离
-----------

### 提取

提取内核可以从升级包中提取，也可以选择从 Android 手机中提取内核，android 手机中提取内核的方法百度一大堆，这里就不展开。

### 分离

分离内核通常会使用到 binwalk、imgtool、或者其他集成工具，建议分离在 Linux 环境中进行。

 

使用 binwalk 查看相关的结构

 

![](https://bbs.pediy.com/upload/attach/202103/849527_FZZ32HHFFGUYUG2.png)

 

一个 boot.img 文件在前面的章节中介绍过，会大致有三个结构。而我们的内核文件则会使用压缩算法进行压缩，我们需要分离出这三个部分的文件，我们可以使用 imgtool 或者 perl 脚本进行分离。

### 我们先来看 perl 脚本集合工具

如下图：

 

![](https://bbs.pediy.com/upload/attach/202103/849527_C3UD42NH5AFCV33.png)

 

boot_info 是用来查看 boot.img 信息的脚本文件。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_TXTY9W733DSE3XA.png)

 

可以查看到镜像文件的大小还有其他的一些基础信息。

 

split_boot 脚本

 

该脚本的功能就是分解 boot.img 或者 recovery.img 文件。

 

解压后的文件目录如下：

```
boot/ramdisk
boot.img-kernel
boot.img-ramdisk.cpio.gz

```

其中 unpack_ramdisk 和 repack_ramdisk 是对 ramdisk 进行解包和打包的工具脚本。

 

在我们对 boot.img 文件提取后还需要查看生成的 boot.img-kernel 文件的内容，大部分提取出来的都会存在压缩算法，gzip、lz4 等等压缩，我们再进行解压才能够得到我们用于内核分析。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_54EES7TVJC78A9A.png)

 

可以查看到的使用了 lz4 压缩算法，那我们可以直接使用 lz4 工具进行解压。

```
lz4 -d filename

```

### imgtool 工具解包

该工具需要自己编译一下

 

工具链接：[https://github.com/kerastinell/imgtool](https://github.com/kerastinell/imgtool)

 

编译之后就可以使用工具进行解包

 

解包之后的文件

 

![](https://bbs.pediy.com/upload/attach/202103/849527_FV6TNC65EHP84BB.png)

 

与上一个工具的解包后的文件是相似的，我们依然需要查看相应内核文件的压缩算法，依然需要使用解压工具进行解压即可。

### 解包实例分析

#### 实例 1

固件：nexus6p 手机中脱的内核镜像

 

在拿到一个 img 文件我们可以使用 file 命令去查看一个基础信息

 

![](https://bbs.pediy.com/upload/attach/202103/849527_NDWD7XVQ6P6UFRY.png)

 

使用 boot_info 工具也可以查看更加详细的信息。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_5ZA52CCHHM7JJFP.png)

 

基础信息对我们提取内核并没有多大帮助，如果后期设计到修改内核文件，重新打包 boot.img 这些信息对我们来讲就是有用。

 

接下来我们可以使用 binwalk 工具查看一下文件类型

 

![](https://bbs.pediy.com/upload/attach/202103/849527_AUFZCJY7T86X3HR.png)

 

可以看出 boot 中含有两个使用 gzip 压缩的文件，在前面介绍 boot.img 文件结构的时我们就讲过含有三个压缩包文件。

 

我们可以尝试使用暴力提取的方法，直接使用 binwalk 提取我们的内核文件。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_NBZ5MHQDCEP8TNG.png)

 

会生成四个文件（实际上是三个），我们依次查看文件类型。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_2V35TAGKFCHTE6E.png)

 

第一个文件就是我们的内核文件，第二个是我们的

 

第二种办法使用比较官方的解法，也是比较正式的解法来剥离我们的内核。

 

该方法会使用到上面提供工具进行解包

 

![](https://bbs.pediy.com/upload/attach/202103/849527_B2ARPWQWRYX5T3B.png)

 

![](https://bbs.pediy.com/upload/attach/202103/849527_VZ6KAWCEZ8YC8FG.png)

 

解出来的三个文件实际上是很官方的三个文件，在这里验证了我们前面的理论知识。

 

我们识别一下我们的 kernel 文件。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_4AKDTCTGDQACSAV.png)

 

这就是一个使用 gzip 压缩的文件，那我们直接解压我们的文件就可以拿到我们的内核文件。这样的形式直接使用 binwalk 当然可以暴力拉出 kernel 文件，但是后面的例子就不一定可以暴力拉出 kernel 文件。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_SSABYH5WVQEV8CS.png)

 

解压后可以拿到同样的文件。

#### 实例 2

固件: pixel-kernel

 

第二个固件我们查看信息会发现采用了 LZ4 压缩

 

![](https://bbs.pediy.com/upload/attach/202103/849527_7V3Y3R7PXPB6PQT.png)

 

这个时候你 binwalk 暴力提取并不会有什么好的发现，没有办法分离干净。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_JTNXP842DW2F4N8.png)

 

得到的两个文件，第一个并不是 kernel 包含的地方，第二个文件中使用 binwalk 查看是有内核信息，我们继续使用 binwalk 暴力提取，会发现生成如下文件

 

![](https://bbs.pediy.com/upload/attach/202103/849527_M3JZU6KV8G9K9VD.png)

 

然后你再去 binwalk，一样的结果。

 

这样我们提取内核文件就没办法使用暴力破解的方式。

 

我们来看官方一点的提取方法，除了上面使用的 imgtool 外，还可以依然使用实例 1 中的方法。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_GSTT9TJBKETZ3Q8.png)

 

可以看出有 LZ4 压缩算法管理者我们的内核文件，我们只能先解 LZ4 算法再进行提取。

 

那我们就使用 lz4 工具解压

 

![](https://bbs.pediy.com/upload/attach/202103/849527_2H9EHSRFTP4K89G.png)

 

解压之后的问价那我们查看就成功提取到我们的 kernel 文件啦。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_UKUZEJRREGUGAUV.png)

#### 实例 3

固件：某嵌入式设备内核

 

![](https://bbs.pediy.com/upload/attach/202103/849527_EWX74B88CPR34VP.png)

 

查看基础文件类型，我们可以看到另外一个新的压缩算法 LZO

 

同样我们可以使用 binwalk 暴力破解。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_XGJCD7U4EJT6YDB.png)

 

这个 76C5 文件就是我们想要的文件。

 

当时使用比较正式的方法也是可以进行提取的，这里就不过多讲解。

第三节 逆向分析
--------

在逆向分析过程中，我们加载上面分离出来的内核文件。

 

工具使用：

 

IDA pro 7.5

 

droiding（[https://github.com/nforest/droidimg](https://github.com/nforest/droidimg)）

 

通常的做法是使用 IDA 加载二进制文件，根据我们系统架构来选择相应的分析

 

![](https://bbs.pediy.com/upload/attach/202103/849527_854WM4KN45ZUCBX.png)

 

选择处理器类型，然后在设置内核起始地址为 0xC0008000（0xFFFFFFC0008000）

 

![](https://bbs.pediy.com/upload/attach/202103/849527_435NMC88VH5NYQA.png)

 

这样就将二进制文件加载到 IDA 中，但是我们的 IDA 并不会自动给我们进行反编译分析，需要我们手动进行转换成代码文件，或者我们也可以使用脚本进行。

 

同时，这样加载的二进制内核文件并没有符号表，我们还需要去提取符号表。

 

我们现在已经 root 过的手机中查询我们的符号表。

```
cat /proc/kallsyms >> kernel_synbols.txt

```

然后使用 python 脚本导入 IDA 中

```
import idaapi
import idautils
import idc
 
def do_rename(l):
    splitted = l.split()
    straddr = splitted[0]
    strname = splitted[2].replace("\r", "").replace("\n", "")
 
    eaaddr = int(straddr, 16)
    idc.MakeCode(eaaddr)
    idc.MakeFunction(eaaddr)
    idc.MakeNameEx(int(straddr, 16), strname, idc.SN_NOWARN)
 
if __name__ == "__main__":
    Message("Hello IDC")
    f = open( "D:\\TOP1_201601\\android\\exploits\\kernel_symbols.txt", "r")
    for l in f:
        do_rename(l)
    f.close()

```

就可以完成我们的加载，但是这样的分析方式比较的复杂，我们在载入二进制内核文件是会出现 IDA 不分析的提示，需要我们手动去选择分析点。

 

使用 IDA 插件 droiding

 

该插件的功能可以自动的解析我们的内核文件，并对符号进行重命名，使用的效果非常不错，根据官方文档我们将 vmlinux.py 文件放在 IDAxxx\loaders \ 文件下就可以在加载二进制内核是多一个选项。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_ZGHQ6D45KR4QVMF.png)

 

这样就只需要等待加载完毕就可以直接进行内核分析。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_6JDXDPW628GTYDC.png)

 

我们的内核文件就已经被完全反编译出来了，可以进行漏洞分析的逆向分析。

第四节 实战内核提权
----------

实战环节我们就使用相关的未打补丁的内核文件进行实战分析并简单的验证我们的漏洞。

#### 案例 1

固件：某嵌入式设备内核

 

Linux-kernel 版本：3.0.35-g4113e5c-dirty

 

在我们使用上面的技术手段拿到我们的 kernel 文件，直接载入含有 vmlinux 查看的 IDA 中进行逆向分析。

 

我是在线去搜寻一下该版本的内核存在哪些历史 CVE

 

[https://www.cvedetails.com/vulnerability-list/vendor_id-33/product_id-47/version_id-135854/opov-1/Linux-Linux-Kernel-3.0.35.html](https://www.cvedetails.com/vulnerability-list/vendor_id-33/product_id-47/version_id-135854/opov-1/Linux-Linux-Kernel-3.0.35.html)

 

![](https://bbs.pediy.com/upload/attach/202103/849527_A3BKWJR4RW4A7YP.png)  
每一个已经修复的 CVE 都存在补丁，我们检测漏洞的存在与否首先是去确认 kernel 文件是否打补丁，然后构造 poc 去验证漏洞的存在性，如果我们是做漏洞安全研究的，还会去梳理漏洞触发的原理，检查补丁是否准确有效，是否可以绕过补丁达到发掘新漏洞的目的。

 

我们以 CVE-2014-3153 为例来进行分析

#### 补丁分析

首先该漏洞年代比较久远，算是比较经典的漏洞类型，原理我们在后面进行剖析，现在我们先确认我们的版本中是否存在该漏洞。

 

从打补丁的位置进行分析

 

![](https://bbs.pediy.com/upload/attach/202103/849527_AX7C5EZRVPN8ADV.png)

 

总共有三个地方进行了修改，我们在 IDA 中去分析这三个函数。

 

我们在我们待分析的固件中查看到并没有对该地方进行修复。

 

![](https://bbs.pediy.com/upload/attach/202103/849527_654W9D4Y2WH4QXN.png)

 

在我们修复的 kernel 文件中是这样存在的

 

![](https://bbs.pediy.com/upload/attach/202103/849527_AJ6HCSPPBXTAUFM.png)

 

除此之外，我们查看一下第二个补丁点，进一步确认漏洞的修复性

 

而我们的第二个补丁位置如下：

 

未打补丁：

 

![](https://bbs.pediy.com/upload/attach/202103/849527_UT6DJ9844YP4FFD.png)

 

而我们来看看打过补丁的位置

 

![](https://bbs.pediy.com/upload/attach/202103/849527_K26AJTZUTFRKJMJ.png)

 

经过一步一步的去逆向补丁位置，我们可以初步认定该 kernel 是未打补丁。

 

接下来就是构造该漏洞的 POC 进行漏洞验证。

 

原则上，想要写出 POC 是要在理清楚漏洞触发的原理才能够 poc 的编写。当然，我们同样可以使用公开的 poc 来进行漏洞的验证。

#### POC 构造

[http://blog.topsec.com.cn/cve2014-3153/](http://blog.topsec.com.cn/cve2014-3153/)

 

参考：  
[https://bbs.pediy.com/thread-209387.htm](https://bbs.pediy.com/thread-209387.htm)

 

[https://www.cnblogs.com/kevingrace/p/10271581.html](https://www.cnblogs.com/kevingrace/p/10271581.html)

 

[https://juejin.cn/post/6844903993924141069](https://juejin.cn/post/6844903993924141069)

 

[https://www.cnblogs.com/0xJDchen/p/6007938.html](https://www.cnblogs.com/0xJDchen/p/6007938.html)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#逆向分析](forum-161-1-118.htm) [#漏洞相关](forum-161-1-123.htm)