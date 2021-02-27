> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-222626.htm)

2. Hyper-V 的结构

随着 Hyper-V 的发展，它内部的体系结构也渐渐成熟稳定。我们下面将介绍 Hyper-V 的体系结构，主要是了解 Hyper-V 内部虚拟化的大致原理。并且介绍 Hyper-V 主要组件的功能和这些组件之间的关系。

2.1. Hyper-V 虚拟化原理和实现方式

Hyper-V 和其他虚拟化产品例如 KVM，VMware 等都使用了相同的硬件虚拟化技术—Intel VT 技术或者 AMD-V 技术。使用这种技术的虚拟化产品可以很好的提升虚拟机的性能，大幅提高虚拟化的性能，同时保证了虚拟化的完整性和安全性。

可以说 Intel VT 或 AMD-V 技术是 Hyper-V 虚拟化的核心。Hyper-V 将这类硬件虚拟化技术用在宿主机和虚拟机的管理和数据交换上，也就是说，这类硬件虚拟化技术成为了宿主机和虚拟机之间的 “信使”，掌管着虚拟机的状态信息以及宿主机和虚拟机之间的数据信息交换。如果将 Hyper-V 比喻为一棵大树的话，那么 Intel VT 或 AMD-V 技术就是大树的根，一切的虚拟化操作都离不开这棵 “大树” 的“根”。

在 Hyper-V 中，Intel VT 和 AMD-V 实现了 Hypervisor 层的逻辑，并且将 Hypervisor 层的功能封装在一个可执行性文件中。如果宿主机为 Intel CPU，那么 Hyper-V 将使用 C:\Windows\System32\hvix64.exe 文件作为 Hypervisor 层的实现；如果宿主机为 AMD CPU，那么 Hyper-V 会使用 C:\Windows\System32\hvax64.exe 文件作为 Hypervisor 层的实现。

在安装了 Hyper-V 的 Windows 系统启动过程中，会先运行 hvix64.exe 或 hvax64.exe 来初始化 Hypervisor 层，然后再运行 Windows 内核 ntoskrnl.exe 进行 Windows 系统的初始化过程。这个过程里，Hypervisor 层的代码级别要比 Windows 内核代码级别高，实际上，微软把 Hypervisor 层称之为 Ring -1 级别，而 Windows 内核代码级别为 Ring 0。Windows 宿主机也称为 Root Partition，其他的虚拟机称之为 Child Partition，事实上无论是宿主机还是虚拟机都是运行在同一个级别下的，也就是说宿主机系统内核代码和虚拟机系统内核代码运行级别是相同的。不同的是宿主机有管理虚拟机的状态的权限，可以对虚拟机进行开机关机增添设备等操作，但是底层代码运行级别却是一样的。宿主机和虚拟机之间的关系如图 1-18。

![](https://bbs.pediy.com/upload/attach/201711/624619_4xalc2heqp9sike.png)  

图 1-18

宿主机和虚拟机之间的通信是通过 Intel VT 特权指令实现的，无论是在宿主机还是虚拟机，当需要通知宿主机或虚拟机有数据需要传输或接收时，都会在内核态执行 vmcall 汇编指令。这时，宿主机 / 虚拟机引发了一个 vm-exit 事件，于是 Hypervisor 层的代码开始处理相关的请求。下面我们会介绍 Hyper-V 虚拟化的实现方式，但是在这之前，我们需要了解一些概念。

**Intel VT** **指令**

VM-EXIT：从 VMX non-root operation 切换到 VMX root operation 的行为。可以理解为：虚拟机 / 宿主机中的代码切换到了 Hypervisor 层的代码，并在 Hypervisor 层完成一些操作。

    VM-ENTRY: 从 VMX root operation 切换到 VMX non-root operation 的行为。可以理解为：Hypervisor 层的代码切换到了虚拟机 / 宿主机中的代码，即 Hypervisor 层完成了某些操作返回到虚拟机 / 宿主机中继续运行。

    VMCALL 指令：在虚拟机 / 宿主机中执行 VMCALL 指令会引发一个 VM-EXIT 事件，并且陷入 Hypervisor 层处理。VMCALL 指令即调用服务例程指令。

**Hypercall**

    Hyper-V 中的 Hypercall 和 XEN 中的 Hypercall 是非常相似的，都是用于和 Hypervisor 层进行交互的接口。他们都调用了 VMCALL 指令，用来进入 Hypervisor 层的代码。VMCALL 指令非常类似于系统调用 SYSCALL 指令，主要用于虚拟机 / 宿主机调用 Hypervisor 中的例程。

**VMBus** **总线**

    VMBus 总线是 Hyper-V 抽象出来用于数据传输的通信信道。VMBus 起着虚拟机和宿主机之间的信息交流作用，并且所有虚拟设备的数据全部由 VMBus 总线进行处理和分发。每个虚拟设备都会分配自己的 Channel，每个 Channel 都对应着一个环形的内存结构，每次从 VMBus 总线接收和发送数据，都要向这个环形内存中读写，而且，只有当这个环形内存读满一圈或者写满才会通知宿主机继续发送数据或者接收数据。

**虚拟设备驱动**

和 QEMU 模拟设备端口读写的方法不同的是，Hyper-V 全部使用了虚拟设备，而不是模拟硬件端口。Hyper-V 不再使用传统的读写模拟硬件端口的方式进行硬件设备的虚拟化，而是使用了虚拟设备作为设备模拟。例如，在虚拟机中网卡，硬盘等设备的驱动都是由微软提供，如果没有这些驱动程序，虚拟机则无法使用虚拟设备，就像是 QEMU 中的 VIRTIO 设备一样，需要加载特殊驱动，而不是使用和真实硬件一样的驱动。

这些虚拟设备都是由 VMBus 总线和宿主机进行通信。Hyper-V 在虚拟机中的驱动收到虚拟机内核发送来的数据或请求时，会先在 Hyper-V 虚拟机驱动中做处理，然后将数据或请求通过 VMBus 总线传到宿主机。这样做的好处是相比于传统的模拟硬件端口有更高的效率并且更好的利用硬件资源。

图 1-19 是以 Linux 虚拟机举例，展示 Hyper-V 内部各部件之间的关系图。其中，虚拟机中的虚拟设备由 Devices 代替；虚拟设备在宿主机中对应的设备驱动由 Host_vdevices 代替。

![](https://bbs.pediy.com/upload/attach/201711/624619_d4a9d6xwtt3f3hz.png)

图 1-19

上图展示了 Hyper-V 内部虚拟化的结构。假设现在 Linux Guest 要发送一个网络数据包，首先 Linux kernel 将数据传入对应的虚拟网卡驱动中 (图中 Devices)，之后网卡驱动处理从内核传来的数据便将处理后的数据准备经过 VMBus 总线发出；VMBus 将数据处理后，便写入到虚拟网卡 Channel 对应的环形内存 (图中 Buffer Ring) 中；数据写入完成后，通过 Hypercall 通知宿主机我有数据需要其接收，虚拟机产生 VM-Exit 事件，陷入 Hypervisor 层执行代码。

数据经过 Hypervisor 层处理完成后，便以触发宿主机中断方式来进行宿主机部分的处理。宿主机的中断处理例程会将包含环形内存地址的数据结构发送到 VMBus 总线，VMBus 读取环形内存的数据，并解析数据，最后将解析后的数据发送到虚拟设备在宿主机 (图中 Host_vdevices) 中对应的设备驱动中处理。

反之，宿主机有数据要发送至虚拟机也是同样的流程。

通过上面介绍可以发现 Hyper-V 中的虚拟机和宿主机确实是同一权限下运行的，并且，虚拟机可以发送数据到宿主机的驱动中。如果宿主机驱动在解析数据时发生了问题，哪怕只是简单的越界读到一个非法地址，都给宿主机带来的是例如蓝屏之类的严重后果。如果这时 Hyper-V 虚拟机中运行着各种业务，那带来的损失是不可估量的。但是，也正是因为 Hyper-V 这种把主要模块放在宿主机 Ring0 权限下运行的特性，也增大了安全人员对其调试和审计的难度。

以上是对 Hyper-V 虚拟化的原理和实现方法做一个大致的介绍，之后的内容还会做详细的介绍。

2.2. Hyper-V 组件及组件功能

下面我们介绍 Hyper-V 的组件及功能。我们将 Hyper-V 使用的文件和系统文件区别开，并加以整理和描述其用途。下列收录的文件是 Hyper-V 常用功能使用的文件，不能代表 Hyper-V 使用的所有文件。这些描述主要用于学习，掌握 Hyper-V 结构，为下面的研究做准备。

同样的，下面组件共能的介绍还是以 Linux 虚拟机为例。

**Hypervisor** **层**

| 

文件名称

 | 

文件位置

 | 

功能

 | 

虚拟机对应设备

 |
| 

hvax64.exe

hvix64.exe

 | 

C:\Windows\System32

 | 

Hypervisor 层逻辑，这一层也被微软称之 Ring -1 层，主要负责 Hyper-V 虚拟化。

 | 

无

 |
| 

hvloader.exe

hvloader.efi

 | 

C:\Windows\System32

 | 

用于系统启动时，初始化 Hypervisor。

 | 

无

 |

表 1-1 Hypervisor 层组件介绍

**内核空间层**

| 

文件名称

 | 

文件位置

 | 

功能

 | 

虚拟机对应设备

 |
| 

vmbusr.sys

vmbkmclr.sys

 | 

C:\Windows\System32\drivers

 | 

VMBus 总线

 | 

hv_vmbus.ko

 |
| 

winhvr.sys

 | 

C:\Windows\System32\drivers

 | 

Hypervisor 层与宿主机之间交互，并且初始化 Hypercall。详细功能类似于 Linux kernel 中./drivers/hv/hv.c

 | 

hv_vmbus.ko

 |
| 

storvsp.sys

 | 

C:\Windows\System32\drivers

 | 

虚拟硬盘设备

 | 

hv_storvsc.ko

 |
| 

vmswitch.sys

vmsproxy.sys

 | 

C:\Windows\System32\drivers

 | 

虚拟网络交换机设备 (网卡)

 | 

hv_netvsc.ko

 |
| 

vid.sys

 | 

C:\Windows\System32\drivers

 | 

虚拟化基础设施 (Hyper-V Virtualization Infrastructure)

 | 

无

 |
| 

synth3dvsp.sys

 | 

C:\Windows\System32\drivers

 | 

RemoteFX 显示加速

 | 

无

 |
| 

hvsocket.sys

 | 

C:\Windows\System32\drivers

 | 

Hyper-V Socket Provider

 | 

无

 |
| 

hvservice.sys

 | 

C:\Windows\System32\drivers

 | 

Hypervisor Boot Driver

 | 

无

 |

表 1-2 内核空间层组件介绍

**应用空间层**

| 

文件名称

 | 

文件位置

 | 

功能

 | 

虚拟机对应设备

 |
| 

vmwp.exe

 | 

C:\Windows\System32

 | 

虚拟机工作进程

 | 

无

 |
| 

vmuidevices.dll

 | 

C:\Windows\System32

 | 

虚拟显示器，键盘，鼠标

 | 

hyperv_fb.ko(显示)

hid_hyperv.ko(鼠标)

hyperv_keyboard.ko

 |
| 

vmicvdev.dll

 | 

C:\Windows\System32

 | 

虚拟机集成服务

 | 

hv_utils.ko(集成服务)

 |
| 

vmdynmem.dll

 | 

C:\Windows\System32

 | 

虚拟机动态内存设备

 | 

hv_balloon.ko

 |
| 

vmbuspiper.dll

 | 

C:\Windows\System32

 | 

用户态的 VMBus 总线设备

 | 

无

 |
| 

vid.dll

 | 

C:\Windows\System32

 | 

用户态的 Hyper-V 虚拟化基础设施设备

 | 

无

 |
| 

virtdisk.dll

 | 

C:\Windows\System32

 | 

虚拟硬盘

 | 

无

 |
| 

vmsynthfcvdev.dll

 | 

C:\Windows\System32

 | 

虚拟光纤通道适配器

 | 

无

 |
| 

vmsynthstor.dll

 | 

C:\Windows\System32

 | 

虚拟存储器适配器

 | 

无

 |
| 

vmsif.dll

 | 

C:\Windows\System32

 | 

虚拟交换机接口

 | 

无

 |
| 

vmchipset.dll

 | 

C:\Windows\System32

 | 

虚拟 BIOS

 | 

无

 |
| 

vmsynth3dvideo.dll

 | 

C:\Windows\System32

 | 

虚拟 3D 显示设备

 | 

无

 |
| 

VmSynthNic.dll

 | 

C:\Windows\System32

 | 

虚拟网卡

 | 

无

 |
| 

vmicrdv.dll

 | 

C:\Windows\System32

 | 

远程虚拟桌面

 | 

无

 |
| 

vmwpctrl.dll

 | 

C:\Windows\System32

 | 

虚拟机控制模块

 | 

无

 |
| 

vsconfig.dll

 | 

C:\Windows\System32

 | 

虚拟机配置模块

 | 

无

 |
| 

vmprox.dll

 | 

C:\Windows\System32

 | 

Hyper-V Component Proxy

 | 

无

 |
| 

rdp4vs.dll

 | 

C:\Windows\System32

 | 

Virtual Machine Remoting Services API

 | 

无

 |
| 

vmrdvcore.dll

 | 

C:\Windows\System32

 | 

VmRdvCore Endpoint

 | 

无

 |
| 

vmsmb.dll

 | 

C:\Windows\System32

 | 

虚拟 SMB 设备

 | 

无

 |

表 1-3 用户态空间层组件介绍

______________________________________________________________________________________________

本文如需引用转载请联系本文作者，看雪 ID：ifyou

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

上传的附件：

*   [2.pdf](javascript:void(0)) （780.04kb，98 次下载）