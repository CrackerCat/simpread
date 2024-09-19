> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283505.htm)

> [原创]Android 主流 Root 方案 Magisk 和 KernelSU 的对比和详解

刚好最近在研究内核级 Root，感觉这玩意还挺复杂的，研究了挺久。为了让大家少踩坑，特意写一遍文章解答一下这些 Root 方式的原理和细节。

一、Magisk 框架  

==============

![](https://bbs.kanxue.com/upload/attach/202409/3WVV7DZZR96Y6CN.jpg)

Magisk 是一款用于 Android 设备的开源 ROOT 框架，它通过独特的原理实现了对系统的深度定制和优化。

### 1. Systemless Root 技术

Magisk 的核心原理在于采用了一种名为 “Systemless Root” 的技术。与传统的 root 方法不同，Systemless Root 技术允许用户在不修改系统分区（/system）的情况下获得 root 权限。这意味着，Magisk 在修改系统时不会破坏系统分区的完整性和稳定性，从而避免了因 root 而导致的问题，如 OTA 升级困难和系统无法启动等。

### 2. Boot 分区修改

Magisk 通过在 boot 分区中创建一个 Magisk 镜像（通常是 / data/magisk.img），并将需要修改的文件和目录挂载到这个镜像中，实现对系统的修改。这样，所有的修改都发生在 boot 分区内，而不会影响到系统分区。在 Android 设备的启动过程中，Magisk 通过修改 boot 分区中的 init 文件，实现在系统启动时挂载 Magisk 镜像，从而实现对系统的隐藏和修改。

![](https://bbs.kanxue.com/upload/attach/202409/862224_BCR7V49JWHEVFQW.webp)

**Magisk 框架通过向 boot.img 的 ramdisk 进行解析，并通过 cpio 工具将 magiskinit 替换掉 / init 文件。这样系统在启动时就会首先执行 magiskinit，magiskinit 会首先以 init 进程的身份完成 systemless 加载，然后再调用原来的 init 文件完成系统的启动。**

### 3. Hook 和 Bind Mount 机制

Magisk 使用了一种 Hook 和 Bind Mount 的机制来构建出一个在 system 基础上能够自定义替换、增加以及删除的文件系统 (/sbin)。具体来说，Magisk 在启动时会在 boot 中创建钩子，然后将 / data/magisk.img 挂载到 / magisk，从而构建出一个可以自定义的文件系统。这个文件系统与系统的原始文件系统并行存在，但 Magisk 能够控制哪些文件或目录被替换、增加或删除。

**具体流程如下：**

*   magiskinit 会在 Pre-Init（预启动）阶段首先挂载 magisk.img 文件到 / sbin 当中去，然后通过对 init.rc 的挂载实现修改并注入 / sbin/magiskd 服务。这样 / sbin 中就会有 su 文件，su 文件通过与 magiskd 通信实现获取 Root。
    
*   在 post-fs-data 触发后， /data 被解密并挂载时。守护进程 Magiskd 将启动，执行 post-fs-data 脚本，并调用已经安装模块中对应的流程文件 (shell)。例如 posf-fs-data.sh 之类的。
    
*   **在启动过程中 magiskd 会修改 sepolicy 规则，并增加 `magisk` 命名空间规则，为后面 ROOT 和模块提供更高的权限。**
    
*   通过执行 resetprop 程序实现将各个模块的 system.prop 信息强制覆盖到系统的 property 里面。
    

### 4. 模块化和功能丰富

除了上述的核心原理外，Magisk 还提供了丰富的模块和功能。用户可以通过 Magisk Manager（Magisk 的管理程序）来安装、升级和管理这些模块。这些模块可以实现各种功能和特性的增加和定制，如 Xposed 模块用于进一步修改和优化系统、Magisk Hide 用于隐藏 Magisk 和其他应用程序的存在以绕过一些应用的检测、MagiskSU 作为 root 权限的管理工具等。

![](https://bbs.kanxue.com/upload/attach/202409/862224_4GWRMNQ3KR7SVCC.webp)

通过 module_install.sh 和 util_function.sh 实现对 Magisk 模块的安装，将插件文件解压到 / data/adb/module 下。

### 5. 方案优势和劣势

由于 Magisk 采用的 Systemless Root 技术，并且所有的修改都发生在 boot 分区内，因此它能够保持非常良好的兼容性，无视原来系统和内核被定制过的情况，不会影响系统的稳定性和安全性。这使得用户可以在不牺牲系统稳定性和安全性的前提下，享受 root 带来的各种便利和功能。

劣势是 magisk 虽然成功获取了 root 权限和模块加载，但是在加载过程中留下很多特征（例如，环境变量，/sbin 目录，特征文件）。所以就需要 shamiko 这类插件来实现隐藏。

二、KernelSU  

=============

KernelSU 是 weishu 编写的一个内核级 ROOT 框架。与 Magisk 不同的是，KernelSU 不通过替换 init 文件，而是通过修改或者注入内核来实现 root。  

### 1、注入方式

KernelSU 通过了两种方式，一种是通过直接刷入已经带 KernelSU 的 GKI 内核来获取权限，另一种则是 KernelSU 最近发布的通过对 boot.img 进行 patch 来实现安装的方式。  

**值得注意的是，KernelSU 虽然也提供 patch 模式，但是与 Magisk 不同的是，KernelSU 仅采用 AnyKernel3 来实现将内核模块注入系统内核。不会对用户空间造成太大的污染。**

### **2.ROOT 获取原理**

*   首先 KernelSU 通过修改内核的 faccessat、stat 调用入口，让需要 root 的应用进程能够感知到 su 文件。![](https://bbs.kanxue.com/upload/attach/202409/862224_KB4RDDZHQ7RJNSG.webp)
    
*   然后在程序通过 execve 执行 / system/bin/su 时，KernelSU 通过在 execve 插桩，判断是否是被运行获取 root 的应用。如果是，则通过**修改 uid 为 0，SELinux context 为 u:r:su:s0**，然后重定向到 / system/bin/sh 的方式为需要的应用提供 Root 权限。          ![](https://bbs.kanxue.com/upload/attach/202409/862224_754U7HN4X2AADDS.webp)
    
*   ksud.c: 通过修改 open 和 read 调用，实现将 ksud 服务注入到 uevent.rc 中，然后通过 ksud 调用模块内的脚本文件，来实现对 Magisk 模块的兼容。![](https://bbs.kanxue.com/upload/attach/202409/862224_S5Q4W7H7Y7B954B.webp)
    
*   其他的，KernelSU 也是一样通过 resetprop 实现对 property 和修改，和修改 sepolicy 实现更高的权限获取。但是 KernelSU 更为高明的是，KernelSU 通过在内核中对 security 模块的 struct policydb 修改来实现在内核空间对 sepolicy 的修改，无需通过用户空间来修改。
    
*   KernelSU 使用 OverlayFS 这一项 Linux 内核新特性来实现和 Magisk 一样的 systemless，并且更为高效，无效在系统启动时进行 bindmount。
    

参考：

Root 注入：[https://github.com/tiann/KernelSU/blob/main/kernel/sucompat.c](https://github.com/tiann/KernelSU/blob/main/kernel/sucompat.c)

服务注入：[https://github.com/tiann/KernelSU/blob/main/kernel/ksud.c](https://github.com/tiann/KernelSU/blob/main/kernel/ksud.c)

sepolicy 修改： [https://github.com/tiann/KernelSU/blob/main/kernel/selinux/sepolicy.c](https://github.com/tiann/KernelSU/blob/main/kernel/selinux/sepolicy.c)

三、附言  

=======

本篇文章版权属于恋空（https://bbs.kanxue.com/homepage-862224.htm），QQ：2928455383。转载请声明出处，之前看到很多人将我的文章转载到 cs 某 n，某乎，并且不声明来源。请大家尊重原创！  

[[课程]Linux pwn 探索篇！](https://www.kanxue.com/book-section_list-172.htm)

[#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm)