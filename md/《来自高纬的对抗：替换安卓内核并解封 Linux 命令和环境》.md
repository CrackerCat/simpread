> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/PIiGZKW6oQnOAwlCqvcU0g)

  

  

以上文章由作者【r0ysue】的连载有赏投稿，共有五篇，本文为第五篇；也欢迎广大朋友继续投稿，详情可点击 [OSRC 重金征集文稿！！！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247484531&idx=1&sn=6925d63e60984c8dccd4b0162dd32970&chksm=fa7b053fcd0c8c2938d1c5e0493a20ec55c2090ae43419c7aef933bcc077692b1997f4710afa&scene=21#wechat_redirect)了解~~  

温馨提示：建议投稿的朋友尽量用 markdown 格式，特别是包含大量代码的文章

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8IlxLg3UwKZ6tZkEctaVIEicMXh73ttiaqWuL0zEdO5VLQHBbfUdMuAA3f8Vu52Zn8AOAABr27xBic9w/640?wx_fmt=jpeg)

*   `Kali Nethunter`
    
*   安卓内核替换
    
*   完整`Linux`环境
    
*   监控内核调用
    
*   制作路由器抓包
    

在前文《定制安卓内核过反调试》中，已经体验过修改内核源码并编译，从根本上绕过`TracerPid`检测的反调试的威力。

也就是说，只要对安卓内核足够熟悉，加上对`AOSP`源码的深刻理解，是可以开启安卓系统的上帝模式，即人与系统的完美融合，实现对加解密（一代壳：整体型、二代壳：函数抽取型）、框架检测、反调试等技术的`bypass`的。

作为本系列的最后一篇，再介绍一种对读者来说非常实用的技术，也是通过 “降维打击” 的思路来实现，那就是替换安卓内核，解封完整`Linux`系统命令，使用内核命令过滤收发包，实现彻底解决抓不到包的问题。

本文所涉及的环境、实验数据结果及代码资料等都在作者的`Github`：https://github.com/r0ysue/AndroidSecurityStudy 上，欢迎大家取用和`star~`

大家知道在`Linux`上，我们想要对数据封包进行统计、定位、可视化、分析和转储等，有一系列的工具和软件。比如：

*   `jnettop`：得到通讯`IP`、端口、URL、速率信息；
    
*   `nethogs`：按进程、端口、PID 分类整理收发包速率
    
*   `htop`：当前系统负载、前台活跃进程、线程和占用
    
*   `netstat -tunlp`：端口对应进程号、监听、收发包端口
    
*   `tcpdump`：网口收发包转储工具，后续供`Wireshark`分析
    
*   `Wireshark`：网络封包分析软件，网络数据瑞士军刀
    
*   `mitmproxy/Charles/BurpSuite`：应用层协议分析工具
    
*   `iwlist/aircrack-ng/HID`：网卡、无线网络、无线键盘等驱动
    

等很多优秀的工具，这些工具为我们进行网络数据封包的分析和处理提供了无与伦比的支持，可以说有这些工具的存在，没有抓不到的包，如果再结合`hook`技术，可以说没有解不开的协议。

工具虽好，但是它们的运行都需要一个完整的`Linux`环境，比如像`jnettop/htop/nethogs`等需要`root`权限，像`iwlist/aircrack/HID`等需要内核驱动支持、监听模式的打开等，`Wireshark/Charles/BP`会需要桌面环境的支持，在`GUI`中进行可视化分析。

有没有一种可能性，在手机上可以跑上述所有的软件，让手机成为网络数据封包和协议分析的 “瑞士军刀”，虽然`GUI`部分有些鸡肋聊胜于无（因为手机屏幕肯定太小了），但是纯`Linux`的环境、还跑在`arm`芯片上，听着就灰常吸引人，最关键的是直接从内核读取`App`的进程、网络、文件、内存等信息，实现从内核层到应用层的 “降维打击”，这才是“瑞士军刀” 的核心竞争力，也是本系列文章的主旨。

这个可能性是有的，那就是`Kali Nethunter`，接下来让我们一起来探索下，`Kali Nethunter`能为`App`的逆向分析、抓包解包分析工作，带来哪些帮助。

### `Kali Nethunter`

`2020`年`4`月初，`Kali`在其官方博客上释出了最新的`Kali Nethunter 2020.1`，带来了船新的`Kali NetHunter Rootless`和`Kali NetHunter Lite`，同时对完整版`Kali Nethunter`进行了更加深入的优化，使用了全新的内核编译工具，从内核全面支持`USB`键盘、光驱和网卡模拟，功能更加强大，系统更加稳定。

新的`Nethunter`针对不同的版本，有不同的要求，并且可以实用的功能也不同。  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEbmD1oajeAicK1XASVsuNkYk2wh4xaYehLIpsBVvViax2n1qfibKmQpMUw/640?wx_fmt=png)

官方甚至还提供了对照表，表明哪些功能在哪些版本上是否存在：

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEKYpxy8Td1NkDNKoqOd1OqByia8GHU47Xia1IpQ11YaSJsDRaTZvv9yaQ/640?wx_fmt=jpeg)

虽然`Nethunter Rootless`可以在任意手机上安装和使用`Kali Linux`的功能，但是官方博客却说，`Nethunter Rootless`可以最多只有`85%`的威力，带`root`、三方`recovery`和`Kali`定制内核的完整版`Nethunter`才能发挥`100%`的威力，作为专业的逆向工程分析师，我们肯定选择`100%`威力的。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMENLD7mpP9sjB8eFDzvBje3Atdibup2qLMh06mKriccbnuEn8s4FiaR8xaQ/640?wx_fmt=jpeg)

`Kali Nethunter`的完整镜像不是所有手机都能装的，只有官网支持的设备才能装。这里使用`Nexus 6p(angler)`进行举例，主要流程分四步：

*   刷入官方原版镜像：
    
*   三方`recovery`：肯定是`twrp`；
    
*   `root`：选择经典的`SuperSU`；
    
*   刷入`Kali Nethunter`；
    

刷入原版镜像的过程不再赘述，相信看完前文内容掌握这个技能还是绰绰有余的，这里采用的是官网的安卓`8.1.0`版本号的第一个下载链接。

刷机结束后，从附件里找到`SR5-SuperSU-v2.82-SR5-20171001224502.zip`，上传到手机上。

```
# adb push SR5-SuperSU-v2.82-SR5-20171001224502.zip /sdcard/
SR5-SuperSU-v2.82-SR5-20171001224502.zip: 1 file pushed, 0 skipped. 14.4 MB/s (6882992 bytes in 0.4

```

用附件的种子`nethunter-2020.2-pre3-angler-oreo-kalifs-full.zip.torrent`，下载`n6p`的`Nethunter`，下载完算下校验码，再上传到手机上去。

```
# adb push nethunter-2020.2-pre3-angler-oreo-kalifs-full.zip /sdcard/
nethunter-2020.2-pre3-angler-oreo-kalifs-full.zip: 1 fi...hed, 0 skipped. 16.7 MB/s (1317485081 bytes in 75.189s)

```

手机重启进入`bootloader`：

```
# adb reboot bootloader

```

在附件里找到`twrp-3.3.1-0-angler.img`，用`fastboot`刷入进去；

```
# fastboot flash recovery twrp-3.3.1-0-angler.img 
target reported max download size of 494927872 bytes
sending 'recovery' (16844 KB)...
OKAY [  2.019s]
writing 'recovery'...
OKAY [  0.246s]
finished. total time: 2.265s

```

刷完之后按音量向下键，选择`Recovery mode`，按电源键进入。

进入`Recovery`之后，选择`Install`→`SR5-SuperSU-v2.82-SR5-20171001224502.zip`开始刷机。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEIstI6wa7d26icN8WjqxhlZ8iabzokFo6DWqTNLVnNeyxpDujbwlj2CxQ/640?wx_fmt=png)

刷完之后选择`Reboot`→`Do Not Install`重启进入系统（可能会重启数次）。

进入系统之后发现已经有了`root`：

```
# adb shell
angler:/ $ su  
angler:/ # whoami
root

```

再次进入`recovery`，把`nethunter-2020.2-pre3-angler-oreo-kalifs-full.zip.torrent`刷进去，中间解压`Kali rootfs`的过程，会至多`25`分钟。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEyTuJAZVqOMsdg0riaTvoiaVIttAN9tbIIpAnrxY17FvmJyUOLNG1EuhQ/640?wx_fmt=png)

刷机结束后进入系统首次也要先点击`Nethunter`的应用，申请的所有权限都给，左侧导航进入`Kali Chroot Manager`，点击`START KALI CHROOT`，只要初始化这一次，后续无论如何重启都会出现如图所示的`Everything is fine and Chroot has been started!`。

至此详细刷机流程结束。

### 安卓内核替换

开机后发现首先内核已经被替换掉了。

> （这里又换了台`nexus 5x`，刷机方法与上述一模一样）

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEyn1icQmQwKeSnaDFyPXhCicWm9z5iaGKZTVoVjDfekc1YbKSvjFibfjKNA/640?wx_fmt=png)

刷之前是谷歌团队编译的内核，刷之后是`kamarush@krsh`编译的，具体二者有哪些不同呢？其实可以在官方文档：nethunter-kernel-1-patching 找到答案。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEOlfHAmP5IQLn4EKf9L7oGoOsfxQEh0pL1Lq6ygwbqWZVy7mbmIAQHw/640?wx_fmt=png)

可以知道这个内核是在标准安卓内核的基础上，给它打的`patch`，主要是对网络功能、`WIFI`驱动、`SDR`无线电、`HID`模拟键盘等功能在内核层面添加支持和驱动，打开模块和驱动加载支持等，文档内容很新，都是四月份最新更新的。

利用这个定制内核，普通的安卓手机就可以进行诸如外接无线网卡使用`Aircrack-ng`工具箱进行无线渗透，模拟鼠标键盘进行`HID Badusb`攻击，模拟`CDROM`直接利用手机绕过电脑开机密码，一键部署`Mana`钓鱼热点等功能。

当然这些对我们进行安卓`App`的逆向好像关系不是很大，关系比较大的在于`Kali Nethunter`在安卓手机里，装上了一个完整的`Linux`环境，也就是第二套系统：`Kali Nethunter`。

### 完整`Linux`环境

这个环境有多完整呢，完整到可以连上显示器和键鼠，直接成为一台电脑，直接把桌面环境带着走。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEk1oqPpJGsSECW9np34ZibnYicficKOKmjhowcUwj6PgXZxq34oiauJMmJA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME54hoLgibXW7s5XR2bb80GmSX51n9tNQL1WqabFspibzvMibJqG2U3pVeA/640?wx_fmt=png)

当然，首先从简单的看起，

1. 点开`Nethunter`这个`app`，给它所有的权限，左上角选择`Kali Chroot Manager`页面，看到`chroot`系统初始化完成。

2. 点开`Nethunter`终端这款`App`，选择`KALI`，进入`Kali`系统。

3.`apt update`升级系统中的软件库信息。

4.`apt install neofetch htop jnettop`，分别康康安卓系统明显不带的这些只有完整`linux`环境才能跑的`命令`能不能执行，可以看到支援的非常完美。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEWCqbsFVMibzI4tQU44gVWrotar6ZPMSxrKKON9qmVT7bK1y1fTYNFJw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME5quESBj0qA9DVVxxjSEfGoE1OOPCNB8dNDW1fNd1dwdsQicKA7odbhg/640?wx_fmt=png)

5. 点开`Nethunter`这个`app`，左上角切换到`KeX Manager`标签页，点击 “SETUP LOCAL SERVER”，会要求输入一个连接密码和显示密码，输入和确认即可，然后点击“START SERVER” 开启服务器。点开 “KeX Client” 这个`App`，在密码那一栏输入密码之后，点击 “Connect” 进行连接，即可直接进入`Kali Nethunter`操作系统的桌面。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEXicW9dacmVTRrMFJtz0aIxSZyk2T7kGCAVcTsD6iaeh523dN1he3qCVg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEIRic9iabX0vt0NyQTDswXMUiaCpvjGuU57GQgcqJD80ItfIdE3hnXJ0dg/640?wx_fmt=png)

6. 搭配 QtScrcpy 就可以在电脑上观看手机屏幕上的内容，还可以输入命令，不用怼着手机屏幕了。

7. 搭配 wifiadb，可以连手机数据线都省掉了。

8. 这个桌面的环境非常的完整，比如它有`Java`的环境，可以直接运行`Java`的大型应用，比如这个用`loader`加载的盗版`BurpSuite2020.06`，（请支持正版）。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME5NxFnTtDIwhd44Y634BSDWFvvnYPxhzw97MwZEkdEnicyF1Ribg8ApMA/640?wx_fmt=png)

9. 可以直接使用`Wireshark`抓所有网卡上的包，可以看到连手机`SIM`卡的网口都可以直接抓，功能十分强大。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME11AOK40sEBSO68p5gNsppGC80UdnaicibYZd3Sicm6tQ7g5j887wjxLiaA/640?wx_fmt=png)

10.`charles`竟然不支持`arm64`，看来无缘明年的`macOS`了。  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEcWfBicol92NjDU61e9bJ6F2RGaH1mKrVuSIDOBSggcForweg6ONcSGg/640?wx_fmt=png)

11.`python`和`python3`都有，`python`的环境也不用担心。只是`frida`由于没有`arm`的`egg`，源码编译笔者也试过好几天，都失败了，如果有大佬可以在`arm`上编译`frida`，希望不吝赐教。

12. 点开`Nethunter`这个`app`，切换到`Kali Services`，将`SSH`启动并且勾选`Start at Boot`，这样就拥有了`sshd`，可以在电脑上操作手机上的命令行了。比如直接查看`App`的以下信息：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEHib37ichB1rp7HlLK99658ecKF5Yc08dibicTx4emzRNLDwGd7E9VCQ1TA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEHib37ichB1rp7HlLK99658ecKF5Yc08dibicTx4emzRNLDwGd7E9VCQ1TA/640?wx_fmt=png)

为什么我们在`Kali`的系统中，可以获取到`App`如此诸多的信息？因为`Kali`系统是与安卓共用同一个内核的，`Kali`只是把内核信息打印出来而已，只是这些内核信息中，携带了`App`的网络、进程、文件系统等信息。

那想从`Kali nethunter`中获取`App`的更多信息，有办法做到么？当然是可以的。

### 监控内核调用

大佬的文章《来自高维的对抗 - 逆向 TinyTool 自制》中实现的系统调用监控模块，从内核打印一份`syscall`的调用记录，其实在完整的`linux`环境中，是直接支持的，那就是`strace`命令。

`strace`是一个可用于诊断和调试`Linux`用户空间`syscall`跟踪器，可以用它来监控用户空间进程和内核的交互，比如系统调用、信号传递、进程状态变更等，`strace`的底层使用也是使用的内核的`ptrace`特性来实现其功能。

当然要分析系统调用，首先得介绍下什么是系统调用，系统调用是指运行在用户空间的程序向操作系统内核请求需要更高权限运行的服务，系统调用提供用户程序与操作系统内核之间的接口。

大家都知道操作系统的进程空间分为用户空间和内核空间，操作系统内核直接运行在硬件上，提供设备管理、内存管理、任务调度等功能；而用户空间通过`API`请求内核空间的服务来完成其功能——内核提供给用户空间的这些`API`, 就是系统调用。

在安卓（`Linux`）系统上，应用代码通过`bionic-C`/`glibc`库封装的函数，**间接**使用系统调用。而某些加固会通过静态编译`bin/so`文件或通过`svc`汇编指令实现绕过`libc`而直接进行系统调用，这样我们`hook/trace libc`也得不到其运行流程，自然无法改变其运行结果。

安卓有三百多个系统调用，具体可以通过`man syscall`命令来查看。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMELzqBEFvfaqHTIm0fKyQ4VDDfITMvWmc5XiapIPcibAiaHylAQuARM3Edw/640?wx_fmt=png)

这三百多个系统调用大部分可以分为以下几个种类：

*   文件和设备访问类：比如`open/close/read/write/chmod`等
    
*   进程管理类：`fork/clone/execve/exit/getpid`等
    
*   信号类：`signal/sigaction/kill` 等
    
*   内存管理：`brk/mmap/mlock`等
    
*   进程间通信 IPC：`shmget/semget` 信号量，共享内存，消息队列等
    
*   网络通信 `socket/connect/sendto/sendmsg` 等
    

接下来实战一下，首先安装这个命令：

```
# apt install strace


```

*   跟踪一下`cat 1.txt`这个命令使用了哪些`syscall`
    

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEeiaYcibh19h8NcQCVApEerXoTc4ptbeS3ynNntarKVg04ozL4OLsHDHg/640?wx_fmt=png)

*   找到某恶意应用的`pid`，并观察它在干什么：
    

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEZFVZ12SoLYoxRJ4ibnHGXrxtQXHZz9icTOwFfHd2qy3neA7aYNr85evg/640?wx_fmt=png)

*   将输出保存到文件中
    

```
# strace -p 27531 -o 27531.txt
strace: Process 27531 attached
strace: [ Process PID=27531 runs in 32 bit mode. ]
strace: WARNING: Proper structure decoding for this personality is not supported, please consider building strace with mpers support enabled.
^Cstrace: Process 27531 detached

```

*   稍等一会儿文件会膨胀到数兆
    

```
# du -h *
1.2M    27531.txt

```

*   寻找打开过哪些文件，可能如果是文件型的一代壳在这里就可以脱壳了。
    

```
# cat 27531.txt |grep -i open
openat(AT_FDCWD, "/data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/base.apk", O_RDONLY|O_LARGEFILE) = 66
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 66
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 44
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 44
openat(AT_FDCWD, "/data/user/0/com.ilulutv.fulao2/files/Fulao2", O_RDONLY|O_LARGEFILE|O_CLOEXEC|O_DIRECTORY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 78
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 44
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 44
openat(AT_FDCWD, "/data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/base.apk", O_RDONLY|O_LARGEFILE) = 44
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 44
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 94
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 113
openat(AT_FDCWD, "/data/user/0/com.ilulutv.fulao2/files/Fulao2", O_RDONLY|O_LARGEFILE|O_CLOEXEC|O_DIRECTORY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 124
openat(AT_FDCWD, "/data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/base.apk", O_RDONLY|O_LARGEFILE) = 124
openat(AT_FDCWD, "/dev/ashmem", O_RDWR|O_LARGEFILE) = 124

```

*   过滤`recvfrom`的接受数据，可以打开个视频流媒体，观察到疯狂接受数据；
    

```
# cat 27531.txt |grep -i recv
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\236\222\16\346\32)\0\0\257'z>\366\220\n?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0p\374\27\371\32)\0\0\205\373\201>\37\35\10?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0U3!\f\33)\0\0g&\223>\37\35\10?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\366\347\245\25\33)\0\0]o\204>\37\35\10?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0V$\257(\33)\0\0004\343\206>\366\220\n?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\264\35032\33)\0\0\257'z>\342\312\v?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0b\233=E\33)\0\0]o\204>\vW\t?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0p{\302N\33)\0\0H\241a>H\251\5?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\207\1\314a\33)\0\0]\17\177>\vW\t?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0(\303Pk\33)\0\0H\241a>\vW\t?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\2605Z~\33)\0\0H\241a>\37\35\10?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0K\31\337\207\33)\0\00043D>\342\312\v?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\261\346\350\232\33)\0\0\1@u>\220\262\20?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\17\344m\244\33)\0\0004\343\206>\342\312\v?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 1664
recvfrom(122, "h\0\0\0\1\0\0\0\1\0\0\0\0\0\0\0\267\326w\267\33)\0\0SXp>\366\220\n?"..., 26624, MSG_DONTWAIT, NULL, NULL) = 832

```

*   可以加上`-e trace=file`参数，只跟踪和文件访问相关的调用 (参数中有文件名)。直接定位出`m3u8`视频文件的路径。
    

```
# strace -p 1431 -e trace=file -o 1431file.txt
strace: Process 1431 attached
strace: [ Process PID=1431 runs in 32 bit mode. ]
strace: WARNING: Proper structure decoding for this personality is not supported, please consider building strace with mpers support enabled.
^Cstrace: Process 1431 detached
# cat 1431file.txt |grep -i ilulu
faccessat(AT_FDCWD, "/data/user/0/com.ilulutv.fulao2", F_OK) = 0
faccessat(AT_FDCWD, "/data/user/0/com.ilulutv.fulao2/files", F_OK) = 0
openat(AT_FDCWD, "/data/user/0/com.ilulutv.fulao2/files/fulao2.m3u8", O_WRONLY|O_CREAT|O_TRUNC|O_LARGEFILE, 0666) = 73

```

还有一些过滤参数如下：  
-e trace=process 和进程管理相关的调用，比如 fork/exec/exit_group  
-e trace=network 和网络通信相关的调用，比如 socket/sendto/connect  
-e trace=signal 信号发送和处理相关，比如 kill/sigaction  
-e trace=desc 和文件描述符相关，比如 write/read/select/epoll 等  
-e trace=ipc 进程见同学相关，比如 shmget 等  

另外一个非常有用的命令是`lsof`，`(list open files)`是一个列出当前系统打开文件的工具。在`Linux`环境下，任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。

所以如传输控制协议 (TCP) 和用户数据报协议 (UDP) 套接字等，系统在后台都为该应用程序分配了一个文件描述符，无论这个文件的本质如何，该文件描述符为应用程序与基础操作系统之间的交互提供了通用接口。

因为应用程序打开文件的描述符列表提供了大量关于这个应用程序本身的信息，因此通过`lsof`工具能够查看这个列表对系统监测以及排错将是很有帮助的。

比如将`lsof`的信息保存到文件，再新进过滤的话：

```
# lsof -p 10694 > 10694.txt

```

*   可以直接看到`App`与远程服务器的通行信息；
    

```
# cat  10694.txt  |grep TCP
.ilulutv. 10694    10106   33u     IPv6             607468       0t0    TCP *:1130 (LISTEN)
.ilulutv. 10694    10106   37u     IPv6             606867       0t0    TCP promote.cache-dns.local:43887->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106   45u     IPv6             602914       0t0    TCP promote.cache-dns.local:45712->223.111.108.40:https (CLOSE_WAIT)
.ilulutv. 10694    10106   46u     IPv6             959562       0t0    TCP promote.cache-dns.local:49787->tsa03s06-in-f10.1e100.net:https (ESTABLISHED)
.ilulutv. 10694    10106   58u     IPv6             606720       0t0    TCP promote.cache-dns.local:45407->server-54-192-16-68.hkg62.r.cloudfront.net:https (ESTABLISHED)
.ilulutv. 10694    10106   70u     IPv6             603094       0t0    TCP promote.cache-dns.local:44281->112.25.43.107:https (CLOSE_WAIT)
.ilulutv. 10694    10106   75u     IPv6             606606       0t0    TCP promote.cache-dns.local:44282->112.25.43.107:https (CLOSE_WAIT)
.ilulutv. 10694    10106   80u     IPv6             606435       0t0    TCP promote.cache-dns.local:46874->server-54-192-16-55.hkg62.r.cloudfront.net:https (ESTABLISHED)
.ilulutv. 10694    10106   83u     IPv6             604772       0t0    TCP promote.cache-dns.local:43858->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106   84u     IPv6             604766       0t0    TCP promote.cache-dns.local:49129->server-54-192-16-46.hkg62.r.cloudfront.net:https (ESTABLISHED)
.ilulutv. 10694    10106   94u     IPv6             606903       0t0    TCP promote.cache-dns.local:47059->server-143-204-129-49.sfo5.r.cloudfront.net:https (ESTABLISHED)
.ilulutv. 10694    10106  104u     IPv6             604779       0t0    TCP promote.cache-dns.local:43859->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  106u     IPv6             605724       0t0    TCP promote.cache-dns.local:43860->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  116u     IPv6             605731       0t0    TCP promote.cache-dns.local:43861->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  117u     IPv6             606523       0t0    TCP promote.cache-dns.local:43862->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  120u     IPv6             606529       0t0    TCP promote.cache-dns.local:43863->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  124u     IPv6             603017       0t0    TCP promote.cache-dns.local:43864->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  127u     IPv6             603019       0t0    TCP promote.cache-dns.local:43865->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  135u     IPv6             606556       0t0    TCP promote.cache-dns.local:43866->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  138u     IPv6             606561       0t0    TCP promote.cache-dns.local:43867->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  140u     IPv6             605786       0t0    TCP promote.cache-dns.local:43869->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  143u     IPv6             603052       0t0    TCP promote.cache-dns.local:43868->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  151u     IPv6             603059       0t0    TCP promote.cache-dns.local:43870->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  185u     IPv6             606873       0t0    TCP promote.cache-dns.local:43888->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  186u     IPv6             612044       0t0    TCP promote.cache-dns.local:43889->223.111.108.189:https (CLOSE_WAIT)
.ilulutv. 10694    10106  203u     IPv6             612078       0t0    TCP promote.cache-dns.local:44301->112.25.43.107:https (CLOSE_WAIT)
.ilulutv. 10694    10106  219u     IPv6             610446       0t0    TCP localhost:49965->localhost:1130 (ESTABLISHED)
.ilulutv. 10694    10106  221u     IPv6             610449       0t0    TCP localhost:1130->localhost:49965 (ESTABLISHED)

```

*   可以直接看到打开的`so`文件：
    

```
# cat  10694.txt  |grep ".so"
.ilulutv. 10694    10106  mem       REG              259,9    826816   1649 /system/lib/libblas.so
.ilulutv. 10694    10106  mem       REG              259,9   1246476   1617 /system/lib/libRSCpuRef.so
.ilulutv. 10694    10106  mem       REG              259,9    828740   1647 /system/lib/libbcinfo.so
.ilulutv. 10694    10106  mem       REG              259,9    162432   1618 /system/lib/libRSDriver.so
.ilulutv. 10694    10106  mem       REG              259,9    252484   1619 /system/lib/libRS_internal.so
.ilulutv. 10694    10106  mem       REG              259,9     45756   1777 /system/lib/libqdutils.so
.ilulutv. 10694    10106  mem       REG              259,9     29088   1737 /system/lib/libmemalloc.so
.ilulutv. 10694    10106  mem       REG              259,9     16132   1776 /system/lib/libqdMetaData.so
.ilulutv. 10694    10106  mem       REG              259,9     37676   1779 /system/lib/libqservice.so
.ilulutv. 10694    10106  mem       REG              259,7              343 /vendor/lib/hw/gralloc.msm8992.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           352102 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/lib/arm/libpl_droidsonroids_gif.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           352094 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/lib/arm/libcipher-lib.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              455 /vendor/lib/libllvm-glnext.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              309 /vendor/lib/egl/libGLESv2_adreno.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,9     20304   1857 /system/lib/libwebviewchromium_loader.so
.ilulutv. 10694    10106  mem       REG              259,9     32184   1666 /system/lib/libcompiler_rt.so
.ilulutv. 10694    10106  mem       REG              259,7              306 /vendor/lib/egl/eglSubDriverAndroid.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              308 /vendor/lib/egl/libGLESv1_CM_adreno.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              444 /vendor/lib/libgsl.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              392 /vendor/lib/libadreno_utils.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              307 /vendor/lib/egl/libEGL_adreno.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,7              325 /vendor/lib/hw/android.hardware.graphics.mapper@2.0-impl.so (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              259,9    133776   1709 /system/lib/libjavacrypto.so
.ilulutv. 10694    10106  mem       REG              259,9     46116   1797 /system/lib/libsoundpool.so
.ilulutv. 10694    10106  mem       REG              259,9    781912   1853 /system/lib/libvixl-arm64.so
.ilulutv. 10694    10106  mem       REG              259,9    159056   1629 /system/lib/libart-dexlayout.so
.ilulutv. 10694    10106  mem       REG              259,9    872188   1852 /system/lib/libvixl-arm.so
.ilulutv. 10694    10106  mem       REG              259,9   2135452   1628 /system/lib/libart-compiler.so

```

*   可以直接看到`/data/data`目录下的相关信息，包括音视频缓存、数据库等。
    

```
# cat  10694.txt  |grep "/data"
.ilulutv. 10694    10106  mem       REG              253,0           352139 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/oat/arm/base.odex (stat: No such file or directory).ilulutv. 10694    10106  mem       REG              253,0           352239 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/oat/arm/base.vdex (stat: No such file or directory).ilulutv. 10694    10106  mem       REG              253,0           130879 /data/misc/shared_relro/libwebviewchromium32.relro (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553088 /data/data/com.ilulutv.fulao2/cache/image_manager_disk_cache/5ac0849c922825f83614103fe2c8af81504425804c1ffef7caab3ea863f0d49b.0 (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           261832 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/base.apk (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553119 /data/data/com.ilulutv.fulao2/app_webview/Cookies (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553089 /data/data/com.ilulutv.fulao2/cache/image_manager_disk_cache/e1bdfc488d9dcb35b8f87c1296491e23a7e3030385e3b642e7887295158e4f36.0 (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553070 /data/data/com.ilulutv.fulao2/cache/image_manager_disk_cache/50a07d4dbc6bc0988fc09b0111e5c63a7a7b485ee0183c30a2eb6f06fc17b431.0 (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553059 /data/data/com.ilulutv.fulao2/cache/image_manager_disk_cache/83588f8ff8f3d6c0b34b70b7e361186535b3f100a090eaaae4ec8a33a814ff59.0 (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553158 /data/data/com.ilulutv.fulao2/cache/image_manager_disk_cache/6228f356158d2b8349cb4e4054a38ca052bd3616b606bdf2ffa4a7d2bfaa8395.0 (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           553091 /data/data/com.ilulutv.fulao2/cache/image_manager_disk_cache/1c3f0d13cf068d44f8a1992a4fb730614a96a3499902e42c3a672f0d8c70567e.0 (stat: No such file or directory)
.ilulutv. 10694    10106  mem-r     REG              253,0           553084 /data/data/com.ilulutv.fulao2/app_webview/Web Data (stat: No such file or directory)
.ilulutv. 10694    10106  mem-r     REG              253,0           553068 /data/data/com.ilulutv.fulao2/databases/Fulao2.db-shm (stat: No such file or directory)
.ilulutv. 10694    10106  mem       REG              253,0           352271 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/oat/arm/base.art (stat: No such file or directory)
.ilulutv. 10694    10106   27r      REG              253,0   7164208 261832 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/base.apk
.ilulutv. 10694    10106   35ur     REG              253,0      4096 553066 /data/data/com.ilulutv.fulao2/databases/Fulao2.db
.ilulutv. 10694    10106   36r      REG              253,0   7164208 261832 /data/app/com.ilulutv.fulao2-OY-Rxd8TtjlQqOrdtNmoHA==/base.apk
.ilulutv. 10694    10106   39u      REG              253,0    234872 553067 /data/data/com.ilulutv.fulao2/databases/Fulao2.db-wal

```

还有一个比较厉害的命令：`sar（System Activity Reporter系统活动情况报告）`是目前`Linux`上最为全面的系统性能分析工具之一，可以从多方面对系统的活动进行报告，包括：文件的读写情况、系统调用的使用情况、磁盘 I/O、CPU 效率、内存使用状况、进程活动及 IPC 有关的活动等方面。其使用方法和特性留给读者去探索。  

### 制作路由器抓包

得益于`Kali Nethunter`为诸多`USB`设备和无线网卡打上了驱动补丁，可以在手机上直接制作路由器，然后在网卡上进行抓包。比如通过手机的`type-c`接口，连接：

*   一个有线网口接有线网络
    
*   一个`USB`
    
*   一个无线网卡
    

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME9qrgbdqykkC2zBO4uZ8YvIzFCf8nVcck10ONBfHfa6GBV6Qg1SCosA/640?wx_fmt=jpeg)

```
# lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 005: ID 0951:1643 Kingston Technology DataTraveler G3
Bus 001 Device 004: ID 0bda:8153 Realtek Semiconductor Corp. RTL8153 Gigabit Ethernet Adapter
Bus 001 Device 003: ID 0bda:8176 Realtek Semiconductor Corp. RTL8188CUS 802.11n WLAN Adapter
Bus 001 Device 002: ID 0bda:5411 Realtek Semiconductor Corp. 4-Port USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

```

手机可以全部识别出来，并且在有线网络上获得了`192.168.0.7`的`IP`地址。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEhIbzC2Gibh0NCicrcGWnE6y0orhJiavUF76Cb4bTB1Z4n2RL89p3iaULiaQ/640?wx_fmt=jpeg)

接下来就是利用`Nethunter`→`Kali Services`，勾选`Hostapd`和`Dnsmasq`，将手机直接变成一个路由器，并且在`Kali Nethunter`桌面上配置`Burp Suite`和`Wireshark`进行抓包。

这条路虽然肯定是没问题的，不过真实场景下估计不会有人这样做，因为性能实在是孱弱，而且屏幕看着也太小了。

在真实的场景之下，最有效的搭建路由器进行抓包的场景，其实肯定是在`Kali Linux`的虚拟机上，用的虚拟机也是来自《2020 年安卓源码编译指南及 FART 脱壳机谷歌全设备镜像发布》文章中所使用的虚拟机，然后随便搭配几块无线网卡即可。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME0VjoHjB2L6METckuTjAwBNiaVzIGPQiaDVZ9DAKBhYKDKzAPxaicCdALA/640?wx_fmt=png)

> 当然树莓派上也可以装`Kali Linux`，配置方法同虚拟机桌面版本。

网卡连接到虚拟机之后，使用`ifconfig`命令查看该网口确定已经出现，默认名称是`wlan0`，然后使用`nm-connection-editor`命令来编辑热点的配置：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEqkfR4yI9ZzFd1xJdonjEYb2lhVMG0N7WJfxMdXmAQmlfAC7ES5UPRw/640?wx_fmt=png)

编辑之后保存，该热点就立即生效了，可以使用`jnettop`来查看该手机的所有的通讯 IP、地址、URL、和速率，以及`Wireshark`来保存`pcap`进行深度分析。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEKOntoAXibhPKzpMODJXDoeV552VG0WzcSFZtteRytU3mKvw3QS7Hqkg/640?wx_fmt=png)

`Kali Linux`虚拟机是开箱即用的，免安装，怎么折腾都无所谓，坏了就解压一个新的，又是船新的系统，干净卫生、可循环使用。

当然，根据`OSI`七层模型，我们在网络层做的路由器，可以抓所有传输层（`Socket`）、应用层（`Http`等）的明文协议包；

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEe76iad54fv6uecWHTlzxYPdrU92oxkgMqrarvjquY07wfq4t1MYjtibw/640?wx_fmt=png)

制作路由器来抓包的优势在于，可以彻底解决抓不到包的问题，因为在路由器上抓，其实跟一个`App`的日常使用场景是没有任何区别的，可以做到对`App`的完全 “无感知” 抓包，实现对`App`的降维打击。

但是如果对`Socket`进行了`encode`比如`gzip`、`tar`等，或者给`Http`加了`SSL`，那么则必须在更高的层次进行解密了，方法已经在《实用 FRIDA 进阶：内存漫游、hook anywhere、抓包》这篇文章中进行了详细的分析，具体不再赘述。

这里再补充和强调一个概念，作为安卓应用安全研究员，凡事都更应该系统的角度出发，比如定位自定义加解密算法的过程中，可以采用 “所见即所得” 的思路：只要是肉眼可见的内容，一定是可以倒追到解密函数的。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEQwu35vn5tflwtE6WywfrgZXBia5TNs6PFxMwp7rFJBH4VZiall5HdWaw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXME3HOY2sRTMA6o1ianKdlqwxlSLWhNkmiaA8OCUQguU9pGeVBIjZmtKbQw/640?wx_fmt=png)

或者采用`hook`系统框架库方法，从`SSL`的`Socket`收发包的函数去`hook`，打调用栈得到解密的方法。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVjByFDuppDAFjszicmibXMEh9OhL3WwEw7GtaWa0nCpn54xSAxiaUX49hA9DuXGtncPLcW7yHd68cQ/640?wx_fmt=png)

这才是作为开源系统安卓上的应用安全研究员，应有的 “我就是系统、系统就是我” 的`App`逆向观念。其实也是 “降维攻击” 思路的一种体现。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)