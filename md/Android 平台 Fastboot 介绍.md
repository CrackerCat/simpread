> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/DLGdMunDHI0Wd39f4D8ZDw)

**一、   Fastboot 定义及功能**
-----------------------

fastboot 协议是一种通过 USB 连接与 bootloader 通讯的机制。它被设计的非常容易实现，适用于 Linux、Windows 或者 macOS 等多种平台。fastboot 是 Android 系统提供的一种较 recovery 更底层的通过 USB 更新文件系统的方式。

Android 开发包提供了 fastboot.exe 工具用于与 Android 系统通信，主要完成分区镜像烧录、分区擦除、设备重启、获取设备状态信息等操作。当需要通过 fastboot 协议与 Android 系统交互时，Android 系统需要启动到 bootloader 模式，此时只有最基础的硬件初始化，包括按键、USB、存储系统、显示等模块，以支持分区镜像的写入和系统启动模式的切换。

**二、    如何进入 Fastboot 模式**
--------------------------

分析 fastboot 启动模式，要从手机启动过程开始，如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmxlTDfU3cZXjZ7dPiajHWYB4ynrclLQxiamSmASTLu1GHztclFJANFMZw/640?wx_fmt=png)

图 1  Linux Kernel 启动流程

A1：上电后执行 BootROM 代码，探测启动媒介，查找第一阶段的引导加载程序 bootloader；

A2：一旦 boot 媒介顺序确定，Boot ROM 会试着装载 bootloader 的第一阶段到内部 RAM 中，一旦 bootloader 就位，BootROM 代码会跳到并执行 bootloader；

B1：bootloader 第一阶段会检测和设置外部 RAM；

B2：一旦外部 RAM 可用，系统会准备装载主 bootloader，把它放到外部 RAM 中；

B3：bootloader 第二阶段是运行的第一个重要程序，它包含了设置文件系统，内存，网络支持和其他的代码；

B4：一旦 bootloader 完成这些特殊任务，开始寻找 Linux 内核，它会从 boot 媒介上装载 Linux 内核，把它放到 RAM 中，同时它也会在内存中为内核替换一些在内核启动时读取的启动参数；

B5：跳到 Linux 内核。

fastboot 就是在 bootloader 阶段中运行的。进入 fastboot 模式一般有两种方式，一种是在关机状态通过按键进入，另外一种是在 Android 系统启动之后通过 adb 指令进入到 bootloader 模式。

通过按键方式进入 fastboot 模式的过程：bootloader 完成硬件初始化之后，启动 Linux 内核时，启动流程会检测按键，如果检测到对应的按键组合则将启动模式设置为 fastboot 模式。

按键检测逻辑为：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmujlFGiaYwroInicRjDeiavfQ03DYZLODPuTzl7XKPnjIfxDJ6OzYSJruA/640?wx_fmt=png)

检测到按键之后，根据按键对启动模式设定：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmwmF7GQatNib0icyD374OjqzL1EmPc3icM683NJ4feQ38YORJbvu3Ym6lg/640?wx_fmt=png)

设置启动模式时各个 ODM 厂商可以根据自身业务需求进行客制化调整。

通过 adb 指令进入 fastboot 模式的指令为：

Adb reboot bootloader

这条指令会将启动模式作为参数传到 Linux 的启动过程中，按键检测完成之后，还会检测是否有启动模式传入，如果存在参数传递，则重新设置启动模式，忽略之前的按键检测。

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmjaohZb1zdbaFGOKZchbK77cCTNt7ibjQXXa8mlne2YQLJic84nibyZ4Zg/640?wx_fmt=png)

在最终获取的启动方式为 FASTBOOT_MODE 时，系统将进入 fastboot 模式启动流程，主要完成 USB 设备的设备的初始化，并启动 fastboot 指令处理线程；注册 fastboot 指令处理函数；显示 fastboot 菜单，并初始化菜单按钮检测程序，处理菜单切换和选择事件，对于一些厂商会客制化一些鉴权过程，鉴权失败则会切换到正常启动流程。

初始化 fastboot 的入口函数为：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmUtEj8g2yz676zp9X5Aa8UCmicrJhEaHLQPQMHkEllL2YKRkuAznnowQ/640?wx_fmt=png)

注册 fastboot 处理函数的过程，简化之后的主要流程为：  

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmsnibrNBkicgCvgibichJ0StEMlJxlJTSFEPBicueib9ibicl5hgXUIaUAP87IQ/640?wx_fmt=png)

**三、    Fastboot 常用指令**
-----------------------

fastboot 指令形式为：fastboot[<option>] <command>

提供与 PC 端的交互指令，通过命令行的形式实现单镜像更新、分区擦除、系统启动等操作。常用指令有：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmHXIh69djsocmfGh6sLtc8ZTicAMYiaNgccZj7amHwL4icx2MjpDYGSbLw/640?wx_fmt=png)

参数 (options)：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmoF1XoFfsoVTwFtAybiaTOVprGCaTnLscFibsmHgbLmiaeroEDSvJFBqvA/640?wx_fmt=png)

例如：

擦除分区：fastbooterase boot

烧录分区：fastbootflash boot path/boot.img

打包烧录：fastbootupdate update.zip(将需要烧录的分区打包到 update.zip 中)

重启手机：fastbootreboot

获取手机端版本信息：fastboot getvar version:version-bootloader(获取 bootloader 版本号)

**四、    Fastboot 镜像烧录权限控制**  

------------------------------

通过命令行进行镜像烧录时，首先在 PC 端进行简单的过滤：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmVfT05pNqbtfCMGtO8CHsnwIp7p6VYD4bPOj0xYukazjvFWicze34Puw/640?wx_fmt=png)

判断设备是否锁定，分区是否存在，是否为保护分区，校验通过之后给手机端发送 flash 指令，手机端收到之后进行更加详细的判断。验证识别锁定，分区是否保护代码如下：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmf74vCGPiaPdNAu9MXDFas9rfHj8xdpmCUD22k2JjeslJG6o7ib6gTLvA/640?wx_fmt=png)

校验通过之后根据镜像的不同格式调用不同的下载处理过程

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmoV1wBIW5XKVubJ6hE98icK4HiaUgJcO9wUNCy1Rb89ElGOmGEaro8eCA/640?wx_fmt=png)

**五、    传输要求及传输示例**
-------------------

如果使用 USB 通道实现 fastboot 传输，对于 USB 的基本要求如下：

*     两个端点，一个输入端，一个输出端；
    
*     对于全速（full-speed）USB，最大包尺寸必须是 64 个字节；
    
*     对于高速（high-speed）USB，最大包尺寸必须是 512 个字节；
    
*     对于超高速（super-speed）USB，最大包尺寸必须是 1024 个字节；
    
*     协议完全是主机驱动（注：相对于设备客户端而言），并且是同步的。这与多通道、双向、异步的 ADB 协议不同。
    

fastboot 传输和组帧：

步骤 1、主机发送命令。一个命令是一个 ASCII 字符串，并且只能包含在不大于 64 个字节的单个包内；

步骤 2、客户端用一个单个的不大于 64 个字节的包响应。响应包开头四个字节是 “OKAY”、“FAIL”、“DATA” 或者“INFO”。响应包剩余的字节可以包含 ASCII 格式的说明性信息；

步骤 3、数据阶段。根据命令的不同，主机或者客户端将发送指定大小的数据。比指定长度短的包总是可接受的，零长度的包将被忽略。这个阶段会一直持续，直到客户端已经发送或接收了上面数据响应包中指定大小的字节数为止；

步骤 4、客户端用一个单个的不大于 64 个字节的包响应；

步骤 5、命令执行成功, 结束交互。

以烧录 boot 分区镜像为例，fastboot 数据传输过程如下：

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddm4M7y7DdgHUpFl4lVOcicQC9EH7um4WKyaa3Z3qjsicaqYrKichzTRPib4w/640?wx_fmt=png)

图 2  fastboot 数据传输过程

**六、    与其他刷机方式对比**
-------------------

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmx5awCVFtMlWDW7UEh45GAWv78qRfGXfNjPXRE7u9UZtwnHTbl7Wiang/640?wx_fmt=png)

**七、  总结**
----------

fastboot 提供对镜像烧录、擦除、手机信息查询等操作，系统权限较高，使用时需要格外留意。相比于其他刷机方式，具有操作方便使用灵活的优点，但是同时存在解锁复杂的缺点。同时对于 fastboot 需要运行在 bootloader 中，对于空板，无法使用，需要首先通过底层线刷模式烧录一个小的系统，之后才能启动 fastboot 的烧录流程。

参考资料：

android/system/core/fastboot/README.md

![](https://mmbiz.qpic.cn/mmbiz_png/d4hoYJlxOjOzteEZic0uPxFBmKnhLAddmDKbaKZZNRjgyW3j6kHdibDib0W5wUFscoJjwiaUWPHJS9ETd4JLgvEFPw/640?wx_fmt=png)

**长按关注**

**内核工匠微信**

Linux 内核黑科技 | 技术文章 | 精选教程