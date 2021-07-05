> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2021/03/10/Tesla%E8%BD%A6%E6%9C%BA%E5%9B%BA%E4%BB%B6%E5%88%86%E6%9E%90%E7%AC%AC%E4%B8%80%E6%AD%A5%EF%BC%9A%E4%BB%8E%E7%A3%81%E7%9B%98%E6%98%A0%E5%83%8F%E4%B8%AD%E6%8F%90%E5%8F%96%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/)

> Tesla 车机固件分析第一步：从磁盘映像中提取文件系统提取磁盘映像文件近期拿到了一个车机的磁盘映像文件，文件是从 Flash 上提取出来的。

[](#Tesla车机固件分析第一步：从磁盘映像中提取文件系统 "Tesla车机固件分析第一步：从磁盘映像中提取文件系统")Tesla 车机固件分析第一步：从磁盘映像中提取文件系统
------------------------------------------------------------------------------------------

### [](#提取磁盘映像文件 "提取磁盘映像文件")提取磁盘映像文件

近期拿到了一个车机的磁盘映像文件，文件是从 Flash 上提取出来的。文件大小为 59.2G，估计是个 64G 的 Flash。

平常分析的设备 Flash 小的就几百 KB，大的也不会超过百兆。一朋友想 binwalk 一把梭，这得提取到猴年马月，拿回来的东西还能看吗（结构可能混乱）。

拿到手 64G，然后是个车机的，根据车型可以知道车机运行 Linux 系统。既然是 Linux 系统，文件系统想必是 EXT4。

![](https://delikely.github.io/2021/03/10/Tesla%E8%BD%A6%E6%9C%BA%E5%9B%BA%E4%BB%B6%E5%88%86%E6%9E%90%E7%AC%AC%E4%B8%80%E6%AD%A5%EF%BC%9A%E4%BB%8E%E7%A3%81%E7%9B%98%E6%98%A0%E5%83%8F%E4%B8%AD%E6%8F%90%E5%8F%96%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/image-20210221171753886.png)

用 binwalk 看了看，确实是 EXT 的文件系统。但是分析的不对，怎么还有 squashfs，这不一般小型的设备才会用吗（PS. 后来 mount 发现确实是 Squashfs）。还是用 fdisk 看吧，这样更靠谱。使用 fdisk 查看磁盘映像的各个分区，`-u` 是以一个扇区为单位显示分区地址。

```
root@kali:/mnt/hgfs
Disk tesla.bin: 59.28 GiB, 63652757504 bytes, 124321792 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: C4F2313A-541D-4377-8B12-17ABDF1EF5CD

Device       Start       End   Sectors   Size Type
tesla.bin1     512    262143    261632 127.8M Linux filesystem
tesla.bin2  262144   4194303   3932160   1.9G Linux filesystem
tesla.bin3 4194304   8126463   3932160   1.9G Linux filesystem
tesla.bin4 8126464 124321757 116195294  55.4G Linux LVM
```

可以看到磁盘中有 4 个分区, 前面三个是 Linux 文件系统，后面是一个 Linux LVM。是 Linux 文件系统这样就好办了，可以用 `mount -o loop` 命令挂载回环设备。以下是对每个分区进行临时挂载。

```
root@kali:/mnt/hgfs
root@kali:/mnt/hgfs
bank_a.dmssize  bank_a.iasImage  bank_b.dmssize  bank_b.iasImage  bootlog.0  elk-product-release  iasImage  lost+found  offline-iasImage

root@kali:/mnt/hgfs
root@kali:/mnt/hgfs
bin  deploy  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  scratch  service  sys  tmp  usr  var
mount -o loop,offset=$((262144*512)) tesla.bin /media/part1/

losetup -f -o $[512*512] tesla.bin
root@kali:/mnt/hgfs
root@kali:/mnt/hgfs
bin  deploy  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  service  sys  tmp  usr  var

root@kali:/mnt/hgfs
mount: /media/root: unknown filesystem type 'LVM2_member'.
```

第一个分区，里面是写一些镜像文件。

第二个分区和第三个分区，里面都是 Linux 根文件系统。两个分区的内容一模一样，其中一个应该是作为备份使用。

第四个分区是 LVM，之前还真没接触过。直接 mount 无法挂载。查资料 了解到 LVM(Logical Volume Manager) 逻辑卷管理是一种将一个或多个硬盘的分区在逻辑上集合，相当于一个大硬盘来使用，当硬盘的空间不够使用的时候能够灵活调整。LVM 磁盘挂载稍微麻烦一点。

1.  使用回环设备管理命令 losetup 加载磁盘映像，`-o` 指定偏移地址。
    
    ```
    root@kali:/mnt/hgfs
    ```
    
    使用 `losetup -a` 可以看到加载的磁盘映像。
    
    ```
    root@kali:/mnt/hgfs
    /dev/loop0: [0040]:18 (/mnt/hgfs/tesla.bin), offset 4160749568
    ```
    
    查看物理磁盘卷。lvm 命令是专门用于管理 lvm 磁盘的。
    
    ```
    root@kali:/mnt/hgfs
      PV /dev/loop0   VG ivg             lvm2 [55.40 GiB / 20.04 GiB free]
      Total: 1 [55.40 GiB] / in use: 1 [55.40 GiB] / in no VG: 0 [0   ]
    ```
    
    可以看出磁盘磁盘的大小为 55.40GB，有 20.04GB 的剩余空间。
    
2.  激活逻辑卷
    
    使用 `lvm vgchange -ay` 命令激活了 14 个逻辑卷。
    
    ```
    root@kali:/mnt/hgfs
      14 logical volume(s) in volume group "ivg" now active
    ```
    
    虚拟出来的设备在`/dev/mapper/`目录下。
    
    ![](https://delikely.github.io/2021/03/10/Tesla%E8%BD%A6%E6%9C%BA%E5%9B%BA%E4%BB%B6%E5%88%86%E6%9E%90%E7%AC%AC%E4%B8%80%E6%AD%A5%EF%BC%9A%E4%BB%8E%E7%A3%81%E7%9B%98%E6%98%A0%E5%83%8F%E4%B8%AD%E6%8F%90%E5%8F%96%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/image-20210221224245419.png)
    
    也可以使用 `lvm lvs` 命令查看逻辑卷信息。
    
    ![](https://delikely.github.io/2021/03/10/Tesla%E8%BD%A6%E6%9C%BA%E5%9B%BA%E4%BB%B6%E5%88%86%E6%9E%90%E7%AC%AC%E4%B8%80%E6%AD%A5%EF%BC%9A%E4%BB%8E%E7%A3%81%E7%9B%98%E6%98%A0%E5%83%8F%E4%B8%AD%E6%8F%90%E5%8F%96%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/image-20210221223841984.png)
    
3.  挂载设备
    
    将 `/dev/mapper/` 目录下的设备挂载后就可以查看磁盘文件内容了。
    
    ```
    root@kali:/mnt/hgfs
    ```
    
    ![](https://delikely.github.io/2021/03/10/Tesla%E8%BD%A6%E6%9C%BA%E5%9B%BA%E4%BB%B6%E5%88%86%E6%9E%90%E7%AC%AC%E4%B8%80%E6%AD%A5%EF%BC%9A%E4%BB%8E%E7%A3%81%E7%9B%98%E6%98%A0%E5%83%8F%E4%B8%AD%E6%8F%90%E5%8F%96%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/image-20210221224006363.png)  
    后面测试发现，可以使用`losetup -f -o`加载磁盘映像文件，然后使用文件管理器点击自动完成激活与挂载过程。
    
    ![](https://delikely.github.io/2021/03/10/Tesla%E8%BD%A6%E6%9C%BA%E5%9B%BA%E4%BB%B6%E5%88%86%E6%9E%90%E7%AC%AC%E4%B8%80%E6%AD%A5%EF%BC%9A%E4%BB%8E%E7%A3%81%E7%9B%98%E6%98%A0%E5%83%8F%E4%B8%AD%E6%8F%90%E5%8F%96%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/image-20210221222246375.png)
    
    此外，losetup 也可以用来挂载其他格式的硬磁盘映像。
    
    ```
    mount -o loop,offset=$((512*512)) tesla.bin /media/part1/
    losetup -f -o $[512*512] tesla.bin
    ```
    

### [](#参考 "参考")参考

*   [从磁盘映像中挂载或提取指定分区](https://blog.csdn.net/xkf58014/article/details/8174232)
    
*   [mount 一个 lvm 格式的磁盘映像文件](https://blog.csdn.net/qq_43668159/article/details/87702139)
    

### [](#致谢 "致谢")致谢

感谢一个不愿透露姓名的白帽汇大佬提供了车机固件。

首发于：[火线 Zone](https://mp.weixin.qq.com/s/gN7jwFWPZh3K_sPQJfVYuA)