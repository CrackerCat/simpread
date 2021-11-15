> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270286.htm)

> [原创] 从零开始制作一个 linux iso 镜像

[](#一、前言)一、前言
=============

    **对于一个极简化的 linux 系统而言，只需要三个部分就能组成，它们分别是一个 linux 内核、一个根文件系统和引导。以下是本文制作 linux iso 镜像所用到的系统和软件：**

 

    **OS: ubuntu 20**  
    **软件: xorriso**

[](#二、制作linux内核)二、制作 linux 内核
=============================

    **1、首先需要去官网选择一个需要的版本下载下来，官网下载地址：[https://mirrors.edge.kernel.org/pub/linux/kernel/](https://mirrors.edge.kernel.org/pub/linux/kernel/)**

 

    **2、利用 tar 将其解压，然后进入其目录中，然后配置内核，常见的配置有以下几种：**  
      **a、make defconfig - 默认配置**  
      **b、make allyesconfig - 创建能选 yes 就选 yes 的配置**  
      **c、make allnoconfig - 创建能选 no 就选 no 的配置**  
      **d、make menuconfig - 基于 ncurser 的图形化界面配置**  
      **这里采用命令 make defconfig 使用默认的即可，如下图所示：**

 

![](https://pic.liesio.com/2021/09/17/e122c59b81597.png)

 

    **3、然后使用`make bzImage`命令编译出内核即可，如下图所示：**

 

![](https://pic.liesio.com/2021/09/17/966d9f7dceb44.png)

 

    **编译好的内核文件在`arch`文件夹相应的架构文件夹下面，如下图所示：**

 

![](https://pic.liesio.com/2021/09/17/e90b4f942b59b.png)

[](#三、制作根文件系统)三、制作根文件系统
=======================

    **1、我们这里利用 busybox 来制作一个根文件系统，busybox 可以简单理解为一个 linux 工具的集合。首先还是下载 busybox，官网下载地址：[https://busybox.net/downloads/](https://busybox.net/downloads/)**

 

    **2、编译 busybox 与编译内核步骤基本一致，将下载好的压缩包进行解压，然后进入文件夹中，使用 make defconfig 配置默认编译选项，这里需要注意的是，在生成的`.config`配置文件中，需要设置`CONFIG_STATIC=y`，如果没有，添加即可，如下图所示：**

 

![](https://pic.liesio.com/2021/09/17/a6cdf68cc7f7b.png)

 

![](https://pic.liesio.com/2021/11/14/8085eef972d4b.png)

 

    **3、然后使用`make busybox install`命令编译 busybox，编译好后会在当前目录下面生产一个`_install`文件夹，如下图所示：**

 

![](https://pic.liesio.com/2021/09/17/2ab200c3cb012.png)  
![](https://pic.liesio.com/2021/09/17/edf22dfe42990.png)

 

    **4、然后创建一个`rootfs`文件夹，并将`_install`文件夹下面除`linuxxrc`以外的所有文件及文件夹都拷贝到`rootfs`文件夹下面，最后创建`dev`等文件夹，最后在根目录下面创建`init`文件即可，文件内容如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/42300f18cd695.png)

 

![](https://pic.liesio.com/2021/11/14/086b36888b052.png)

 

    **5、最后利用命令`find . | cpio -R root:root -H newc -o | gzip > ../rootfs.gz`将文件系统打包，至此，一个文件系统就创建完成了，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/4507a605826e2.png)

[](#四、bios)四、BIOS
=================

    **1、这里我们使用`syslinux`来创建`bios`引导的一个 linux iso 镜像，`syslinux`官方下载地址如下：[https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/](https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/)**

 

    **2、将下载好的`syslinux`解压，然后创建文件夹`isobios`，将解压后的`syslinux`文件夹下面的`bios/core/isolinux.bin`、`bios/com32/elflink/ldlinux/ldlinux.c32`复制到`isobios`文件夹下面，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/e76e958333301.png)

 

    **3、在`isobios`文件夹下面创建配置文件`isolinux.cfg`，文件内容如下所示：**

 

![](https://pic.liesio.com/2021/11/14/5fb18392f0744.png)

 

    **4、最后，在`isobios`文件夹下面使用命令`xorriso -as mkisofs -o ../testbios.iso -b isolinux.bin -c boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table ./`生成 iso 镜像文件，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/a75d337678ef4.png)

 

    **5、使用虚拟机`vmware`创建一个虚拟机，如下图所示，便是我们创建的一个 linux iso 镜像跑起来的样子。**

 

![](https://pic.liesio.com/2021/11/14/2651c2f647a3d.png)

[](#五、uefi)五、UEFI
=================

    **1、uefi 这里采用`system-boot`和`syslinux`配合来制作，首先，创建两个文件夹`isouefi`和`tmp`，其中，`isouefi`用来挂载设备，`tmp`文件夹用来临时存放文件以计算大小，然后在`tmp`文件夹下面创建`EFI/BOOT`和`loader/entries`目录，接着，将解压后的`systemboot`下面的`uefi_boot/EFI/BOOT/BOOTx64.EFI`文件拷贝到`tmp/EFI/BOOT`目录下面，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/0505685083595.png)

 

    **2、接着，在`tmp/loader`目录下面，创建文件`loader.conf`配置文件，第一行表示默认配置是`entries`目录下那个文件，第二行设置默认超时时间；然后在`entries`文件夹下面创建相应的配置文件，这里是`mll-x86_64.conf`，文件内容和`bios`的差不多，不在单独细说，最后再将前面准备好的内核和文件系统拷贝到`tmp`目录下面，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/c2dece72087e0.png)

 

![](https://pic.liesio.com/2021/11/14/d19030d72f305.png)

 

![](https://pic.liesio.com/2021/11/14/513c14a55ba21.png)

 

    **3、此时就可以根据`tmp`文件夹的总大小创建一个相同大小的`img`文件了，这里的`tmp`是`11M`，为了稳妥起见，这里创建一个`12M`的`img`文件，命令为`truncate -s 12M uefi.img`，然后使用`losetup -f`命令寻找一个当前未使用的逻辑设备，然后使用`losetup`命令将我们前面创建的`img`文件虚拟成改逻辑设备，接着利用`mkfs.vfat`将该设备格式化成`vfat`系统，接着使用`mount`命令将其挂载到`isouefi`文件夹下面，最后将`tmp`文件夹下面所有文件及其文件夹拷贝到`isouefi`目录下面，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/7dc920e50d6bf.png)

 

![](https://pic.liesio.com/2021/11/14/cd9207ca0c31e.png)

 

    **4、接着利用`umount`命令取消挂载，这样我们就得到一个包含`内核`、`文件系统`等的`img`文件，接着创建一个`iso`文件夹，并且在该文件夹下面将创建一个`boot`文件夹，然后将`img`复制到`iso/boot`下面，最后利用`xorriso`工具生成`iso`文件即可，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/6afa32b95786a.png)

 

    **5、最后，新建一个虚拟机，引导选择 uefi，启动即可，如下图所示：**

 

![](https://pic.liesio.com/2021/11/14/237f42a95bcff.png)

 

![](https://pic.liesio.com/2021/11/14/f0ad6b532d49d.png)

[](#六、相关链接)六、相关链接
=================

    **github 链接：[https://github.com/windy-purple/make_linux_iso](https://github.com/windy-purple/make_linux_iso)**

[[公告] 欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#其他内容](forum-4-1-10.htm)