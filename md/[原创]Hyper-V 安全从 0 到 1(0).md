> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-222624.htm)

### Hyper-V 安全从 0 到 1(1)

### [https://bbs.pediy.com/thread-222626.htm](https://bbs.pediy.com/thread-222626.htm)

### Hyper-V 安全从 0 到 1(2)

### [https://bbs.pediy.com/thread-222641.htm](https://bbs.pediy.com/thread-222641.htm)

### Hyper-V 安全从 0 到 1(3)

### [https://bbs.pediy.com/thread-222656.htm](https://bbs.pediy.com/thread-222656.htm)

### Hyper-V 安全从 0 到 1(4)

### [https://bbs.pediy.com/thread-222657.htm](https://bbs.pediy.com/thread-222657.htm)

### Hyper-V 安全从 0 到 1(5)

### [https://bbs.pediy.com/thread-222692.htm](https://bbs.pediy.com/thread-222692.htm)

引言：研究微软虚拟化产品 Hyper-V 历时 6 个月，从去年的 7 月份到今年年初挖到了第一枚 0day。之后因为种种原因没有再继续研究下去，所以决定和大家分享当时研究过程的一些经验。内容不算太多，但是一次也整理不完，干脆分批次发到看雪上了。当时研究 Hyper-V 版本和现在也有差别，微软已经做了不少的升级改进，所以文中的描述可能会和最新版本有所出入。本人能力有限，并不是业界内的大牛，所以可能文中有很多处描述不是很正确，希望大牛们一起讨论并指正。

  
    本文包含了一些基本知识（一些废话），望能力强的大牛轻喷 2333.  

1. Hyper-V 介绍及安装

Hyper-V 是一款微软编写的虚拟化软件，其功能类似于其他虚拟化产品，例如：VMware，QEMU，Xen 等产品。现阶段 Hyper-V 被广泛应用于 Microsoft Azure 云计算平台中，普通用户可以很轻松的购买 Azure 的云计算服务。并且，在最新版本的 Windows 操作系统中，Hyper-V 也可以非常方便的安装与使用。无论是云计算服务还是桌面虚拟化，Hyper-V 都以强大的生命力竞争着当今的虚拟化市场。

1.1. Hyper-V 介绍

支持 Hyper-V 的操作系统版本需要 Windows server 2008 或更高版本 Windows 操作系统，并且宿主机 CPU 要求支持 Intel VT 技术或者 AMD-V 技术。早期版本的 Hyper-V 还不能支持 Linux 虚拟机，并且功能单一。通过这几年 Hyper-V 版本的不断更新，它已经支持 Linux 虚拟机，并且完善了许多功能，在虚拟机性能和安全性上也有了极大的提升，充分具备了与市面上其他虚拟化软件竞争的条件。

    Hyper-V 采用了微内核的架构，充分保证了宿主机的安全性和稳定性，同时也提高了虚拟机性能。正是因为 Hyper-V 采用了微内核架构，所以最底层为虚拟机的 Hypervisor 层，微软称之为 Ring -1 层，而作为 Host 的 Windows 操作系统运行在 Ring 0 层。在系统启动过程中，Hypervisor 层要先于 Windows 系统启动，并且运行权限要比 Windows 操作系统还要高一级，说白了，就是在系统启动过程中，Hypervisor 层接管了 Windows 操作系统。而这种架构十分像 XenProject 开发的 Xen。

图 1-1 概括了 Hyper-V 的运行架构，便于读者更好的理解 Hyper-V 的大致架构。

![](https://bbs.pediy.com/upload/attach/201711/624619_bp163d7f0mjucly.png)  

图 1-1 Hyper-V 架构

(图片来源：[https://i-msdn.sec.s-msft.com/dynimg/IC194739.gif](https://i-msdn.sec.s-msft.com/dynimg/IC194739.gif)）

    如图 1-1 所示，在启动 Hyper-V 管理器中的虚拟机时，会为每一个虚拟机创建一个名为 vmwp.exe 的进程，在图中就是 Root Partition 中的 VMWP(虚拟机工作进程) 部分，而这个 Root Partition 就是宿主机系统，即当前负责管理虚拟机的 Windows 操作系统。在 Root Partition 中，宿主机通过 VMMS(虚拟机管理服务) 来管理虚拟机的状态。

    启动后的虚拟机在图中划分为 Child Partition，如图 1-1 中所示，Root Partition 和 Child Partition 都是运行在相同级别的 Hypervisor 层之上，换句话说，虚拟机和宿主机在硬件中的运行级别是相同的，也就是说，如果虚拟机发生了故障，不会影响到宿主机；宿主机发生了非致命故障，例如并非蓝屏等 Ring 0 下的故障，也不会影响虚拟机的正常运行。笔者曾经对一个虚拟机工作进程 vmwp.exe 进行下断点调试，会发现表面上虚拟机像是停止运行了，但是实际上 SSH 连上虚拟机的终端还是能完美的运行我所输入的指令。因此 Hyper-V 的这种微内核架构极大程度上提高了系统稳定性，并非像 QEMU 和 VMware 那样，断下虚拟机工作进程后，虚拟机内部就确确实实的停止工作了。

    宿主机和虚拟机之间的数据传输通过 VMBus 实现，VMBus 是微软开发的虚拟数据传输总线，用来宿主机和虚拟机之间交换数据和信息。使用 VMBus 的优点就是能进行快速、高效、大数据量的数据传输，极大的提升了虚拟机的性能。VMBus 的原理和 QEMU 中的 VirtIO 设备类似，通过共享环形内存，每当要传输的数据写满整个环形内存时，就把数据发送到宿主机中。可以说，微软成功的借鉴了 VirtIO 设备的优点，取其优点用之，才会使得 Hyper-V 虚拟机性能大幅提升。

    图 1-1 中，WinHv/LinuxHv 是用来进行宿主机和虚拟机通信的模块。例如，当有数据要从虚拟机中发送至宿主机中，会通过虚拟机中的 WinHv/LinuxHv 通知宿主机进行接收并处理；从宿主机传递数据到虚拟机也是同理。WinHv/LinuxHv 模块通信原理是 Intel VT 技术或者 AMD-V 技术，实现的是硬件层面上的虚拟化，所以在安全性上看来，Hyper-V 这种利用硬件虚拟化的方式与传统的模拟设备端口读写的虚拟化方式相比，硬件虚拟化更加具有安全性上的优势。而且，硬件虚拟化也有在虚拟机性能方面的优势。

    Hyper-V 在虚拟机性能和安全性上可以说是有不俗的表现，但是在功能上来讲，VMware 要比它做的好的多，例如，Hyper-V 不支持虚拟串口，不支持虚拟 USB 设备等。这对依赖这些功能的用户来讲，确实是一个麻烦的问题，只能期待微软在以后的 Hyper-V 版本中加入此类功能吧。

1.2. Hyper-V 安装及配置

1.Hyper-V 的安装

Hyper-V 的安装十分简单，不似 VMware 那般需要自行下载安装，更不需 QEMU，Xen 那般需要编译，解决一大堆依赖问题。

如果您的 Windows 版本是桌面版, 那么您可以通过如下方式安装 Hyper-V：控制面板 -> 程序和功能。如图 1-2，点击启用或关闭 Windows 功能，出现图 1-3 对话框。在图 1-3 中，勾选上所有 Hyper-V 相关的功能，点击确定，然后重新启动计算机。之后的安装 Windows 会自动运行，重启后，Hyper-V 便安装完成，如图 1-4。

![](https://bbs.pediy.com/upload/attach/201711/624619_9hca1171nioq076.png)

图 1-2

![](https://bbs.pediy.com/upload/attach/201711/624619_sro4kmqixwebeop.png)

图 1-3

![](https://bbs.pediy.com/upload/attach/201711/624619_bzwqp81bviktyus.png)

图 1-4

如果您的 Windows 版本是 Windows Server 版本，则可以到服务器管理器中，点击右上角的管理菜单，点击添加角色功能，如图 1-5 所示。

![](https://bbs.pediy.com/upload/attach/201711/624619_rj4q3f1mypfmzki.png)

图 1-5

在添加角色和功能向导窗口，选择安装类型为基于角色或基于功能的安装，在之后的服务器角色栏中选中 Hyper-V，如图 1-6 所示。  
![](https://bbs.pediy.com/upload/attach/201711/624619_xuyl2iycva815ga.png)  

图 1-6

点击添加功能，下一步，在图 1-7 中勾选方框内的选项，使 Hyper-V 自动配置虚拟网卡。

![](https://bbs.pediy.com/upload/attach/201711/624619_lf114uvle5m3tav.png)  

图 1-7

然后下一步，直到出现安装按钮，点击。Windows 便会自动安装 Hyper-V，稍等片刻后，Windows server 下的 Hyper-V 便安装好了。

2.Hyper-V 新建虚拟机及配置

完成了 Hyper-V 的安装后，下面我们就要开始创建并配置虚拟机。值得一说的是，Hyper-V 在网络配置上并没有大家所熟知的 NAT 网络地址转换这类连接方式，Hyper-V 默认网络配置是桥接，也就是宿主机与虚拟机工作在相同的网段内。但是并不意味着我们无法使用 NAT 网络地址转换这种连接方式进行配置。下面会介绍 Hyper-V 创建和配置虚拟机，并在网络连接上按照桥接方法和 NAT 网络地址转换两种不同的方法来配置。

创建虚拟机步骤如图 1-8 所示。

![](https://bbs.pediy.com/upload/attach/201711/624619_33359v7b1oylp64.png)  

图 1-8

在这之后，会弹出新建虚拟机向导，在图 1-9 中，我们可以指定虚拟机的名称和存放虚拟机的位置。点击下一步后，在指定代数选项上选择第二代虚拟机，并且将之后的分配内存选项中，如图 1-10，分配这个虚拟机需要的运行内存大小，或者可以选上使用动态内存，可以更加灵活的分配虚拟机资源。

在配置网络选项中，我们暂时选择未连接，在之后的网络配置中专门介绍这个配置。

在连接虚拟硬盘选项中，如图 1-11，框内可以指定新建的虚拟机磁盘镜像的名称，存储的位置和磁盘镜像的最大大小。

在选项安装选项中，这里可以直接选择要安装的操作系统镜像，也可以选择之后再配置安装镜像。点击下一步，完成，便新建并配置完成虚拟机。

![](https://bbs.pediy.com/upload/attach/201711/624619_j8xnk7fzagv0gdk.png)  

图 1-9

  
![](https://bbs.pediy.com/upload/attach/201711/624619_iqlw52tfy96idu8.png)  
图 1-10  
![](https://bbs.pediy.com/upload/attach/201711/624619_0ncmrhrqiv9519y.png)  
图 1-11  

3.Hyper-V 虚拟机网络配置

桥接模式的配置：

点击 Hyper-V 管理器界面中的虚拟机交换机管理器选项，如图 1-12，选择外部类型的虚拟交换机，再点击创建虚拟交换机。然后来到图 1-13 的对话框，在虚拟交换机属性页面，可以任意指定虚拟交换机名称，这里我们把虚拟交换机连接类型选择为笔者的无线网卡上，这样就可以在使用虚拟机时使宿主机和虚拟机在同一个虚拟的局域网内，并且由宿主机模拟出的交换机访问外网，这大大方便了虚拟机与宿主机之间的网络访问。

![](https://bbs.pediy.com/upload/attach/201711/624619_xnjc90ntcn5z3h8.png)  
图 1-12  
![](https://bbs.pediy.com/upload/attach/201711/624619_kbcv9p72voat32m.png)  

图 1-13

    NAT 网络地址转换的配置：

在现实使用中往往使用桥接模式来配置虚拟网络是不能满足需求的，在微软官方文档中有介绍 Hyper-V 配置 NAT 网络的详细步骤 ([https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network))。但是，微软的官方文档中要求宿主机为 Windows10 以上，故官方这种方法并不是非常通用。所以，以下我们介绍一种比较通用的实现 Hyper-V NAT 虚拟网络的办法。

这种办法需要我们对系统进行一些配置。首先，打开设备管理器，选择添加过时硬件，如图 1-14。然后在弹出的对话框中选择 “安装我手动从列表选择的硬件”，在“常见硬件类型” 中选择“网络适配器”。

![](https://bbs.pediy.com/upload/attach/201711/624619_yrosd9raxhk9aq7.png)  

图 1-14

如图 1-15 所示，厂商选择 “Microsoft”，型号选择 “Microsoft KM-TEST 回环适配器”，单击下一步，按照所给出的提示 Windows 将自动安装这个设备。

下面我们打开 Hyper-V 管理器中的 “虚拟机交换机管理器”，进行下一步的配置。

![](https://bbs.pediy.com/upload/attach/201711/624619_qmr3uin9bd57dao.png)  

图 1-15

打开 “虚拟交换机管理器” 之后，和前面介绍的新建虚拟网络交换机步骤一样，新建一个外部网络，然后按照图 1-16 所示配置连接类型。将连接类型中的设备填上我们刚刚添加过的 “Microsoft KM-TEST 回环适配器” 设备，然后点击确定完成虚拟交换机的配置。

![](https://bbs.pediy.com/upload/attach/201711/624619_ba2l8p37o8phkr6.png)  

图 1-16

最后，打开 “网络和共享中心”，点击左边栏中的“更改适配器选项”。选择你现在能连接网络的网卡，右键“属性”，切换到“共享” 一栏，勾选上“允许其他网络用户通过此计算机的 Internet 连接来连接”。在笔者的机器上，如图 1-17 所示。如果您有多块网卡设备的话，需要选择刚才新建的那个虚拟交换机设备。

![](https://bbs.pediy.com/upload/attach/201711/624619_81wsffv3lqd99h5.png)  

图 1-17

这样，便可以实现虚拟网络的 NAT 功能。

  
第 0 章，废话篇结束。  

______________________________________________________________________________________________

本文如需引用转载请联系本文作者，看雪 ID：ifyou

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

上传的附件：

*   [1.pdf](javascript:void(0)) （2.94MB，88 次下载）