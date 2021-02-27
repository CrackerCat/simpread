> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-222641.htm)

3. Hyper-V 的调试

        下面我们介绍 Hyper-V 的调试方法。在前面，我们简单介绍了 Hyper-V 的结构，它的虚拟机和宿主机运行在同一个权限下，并且，由 Windows 系统中的驱动虚拟出对应的虚拟设备。所以，在调试方面，Hyper-V 便显得略繁琐。如果要调试 Hyper-V 在宿主机中的驱动的话，不得不要进行 Windows 内核的调试。

         而且，由于 Hypervisor 层的权限要高于 Windows 内核权限，所以当需要调试 Hypervisor 层代码时，仅仅调试 Windows 内核并不能满足这一要求。

         下面，我们介绍两种不同的方法来调试 Hyper-V。一种是实体机双机调试，一种是 VMware 调试 Hyper-V，并且我们会分别介绍两种方法的优缺点。

3.1. 需要准备的工具

         在介绍两种调试方法之前，我们需要做一些准备。首先，我们要确定以哪种方式进行调试。如果你使用的机器是笔记本或者电脑主机没有串口接口，并且确定主机的 CPU 支持 Intel VT-X 和 VT-d 技术 (如果您的 CPU 是 AMD，请确认是否支持 AMD-V 和 AMD-Vi)。拿笔者主机的 CPU 为例，如图 1-20，支持 Intel VT-x & VT-d。那么，您可以优先选择使用 VMware 双机调试 Hyper-V。

![](https://bbs.pediy.com/upload/attach/201711/624619_mfrxzvo3zqk2pdn.png)

图 1-20 CPU 型号为 Intel i7- 4790

         如果您的机器不支持 Intel VT-d(AMD-Vi) 的话，只能使用双实体机双机调试的办法调试 Hyper-V。

如果调试方法采用 VMware 双机调试，那么在下面准备过程中，只需要准备软件工具即可。如果采用双实体机双机调试方法，则在软件工具基础上还需要准备硬件上的工具。

         在调试方法上，一般情况下，无论是实体机双机调试还是 VMware 双机调试，笔者还是优先选择使用网络调试的办法。因为网络调试的速度快，配置容易，调试效率极高。但是，在网络调试的调试底层代码过程中，可能因为网络调试引入新的问题，影响调试的结果。但串口调试不会产生这种 “干扰”，但随着而来的是调试效率的极度下降。所以一般来说，笔者会使用网络调试作为一般情况下调试的方法。如果发现问题，会再在串口调试的环境下重新调试一遍，以排除干扰。而且，一般来说比较旧的机型保留串口的几率要大些，可以充分利用身边淘汰下来的旧机器作为串口调试机器。除此之外，也有 1394 线的调试办法，介于 1394 接口已经在现在的主机上消失了，现在使用 1394 线调试还需要自行安装 1394 PCI 板载卡。过程比较麻烦，所以在此不做考虑。

那么下面介绍一下我们所需要的软件和硬件上的工具。

硬件工具

         首先，需要确定您的主机后面板是否有串口接口，如果没有，那只能通过网络调试的办法进行双机调试。这需要您将两台主机连接在一个局域网内，并且能保证双方的正常通信。

         如果主机后面板有串口接口，那么您可以使用串口进行双机调试。那么您需要一根 RS232 Null- modem 线。Null-modem 线的接线见表 1-4。

![](https://bbs.pediy.com/upload/attach/201711/624619_9v8hqw76vhgyefw.png)

表 1-4 Null-modem 接线方式

         Null-modem 线一般在商场中是买不到的，所以可能需要你自己做或者找能定做线缆的商家定做。事实上，笔者在调试时，只需要将 9 针串口线两端的 2，3 脚对调，其他线路直连便可正常进行双机的通信和调试。

![](https://bbs.pediy.com/upload/attach/201711/624619_33k9a0z7m22fut0.png)

图 1-21 9 针串口管脚定义 (左公右母)

图 1-21 为 9 针串口管脚定义，左边为公头，右边为母头。

软件工具

         软件方面需要安装好 WinDbg，最新版的 SDK 安装包可以到微软官方下载 (https://developer.microsoft.com/en-us/windows/hardware/windows-driver-kit)。除此之外，我们还需要微软的符号文件，请根据你被调试的系统到微软官方下载符合您被调试系统的符号文件包 (https://developer.microsoft.com/en-us/windows/hardware/download-symbols)。

         如果使用双实体机串口调试，那么还需要安装好 HyperTerminal(http://www.hilgraeve.com/hyperterminal-trial/)，或者其他串口通信的软件，用于检测串口是否正常通信。

         下面我们配置 WinDbg 符号文件地址。我们需要新建一个环境变量，如图 1-22 所示。

![](https://bbs.pediy.com/upload/attach/201711/624619_2awtnpsfgddclhq.png)

图 1-22 添加系统环境变量

         在 “环境变量” 对话框中，点击 “新建” 按钮来新建一个系统环境变量。在 “新建用户变量” 对话框中，在 “变量名” 中填入 “_NT_SYMBOL_PATH”，在“变量值” 中填入 “SRV*d:\symbols*http://msdl.microsoft.com/download/symbols”，然后确定保存。其中，“变量值” 一栏中 “d:\symbols” 代表你安装符号文件包时选择的安装位置；“http://msdl.microsoft.com/download/symbols”代表微软的符号文件服务器，如果本地符号文件中没有匹配的符号文件时，便会从微软符号文件服务器上下载所需的符号文件。这个新建的环境变量用于配置全局符号文件地址，由于我们添加在了系统环境变量中，所以不用每次启动 WinDbg 再去重新设置符号文件地址。。由于笔者之前配置过环境变量，所以图中显示的是 “编译用户变量” 对话框。

         如果您用 WinDbg 附加到任意一个进程时，输入命令 “.sympath” 如图 1-23 所示，那么说明符号已经设置成功了。

![](https://bbs.pediy.com/upload/attach/201711/624619_i5h0jdw4xrnd9ho.png)

图 1-23

3.2. 实体机双机调试环境搭建

         下面我们介绍实体机的双机调试，这部分只介绍使用串口进行双机调试的方法。通过网络双机调试方法和下面 “VMware 双机调试环境搭建” 大部分是一样的，只不过实体机双机网络调试用的是真实的网络设备而已。

         首先，用 Null-modem 线或者 2，3 口交叉的 RS232 线将两台实体机连接起来。然后开机，分别在 “设备管理器” 中查看“端口”。如图 1-24 所示，左边是调试机，已经把另一端的被调试机主机串口识别为 COM2；右边是被调试机，把调试机主机的串口识别为 COM1。

![](https://bbs.pediy.com/upload/attach/201711/624619_hux6ktgk5g8jgam.png)

图 1-24

         下面，设备双方串口的波特率，我们以调试机为例。在 “设备管理器” 的“端口”项中右键单击 “通信端口”，“属性”，然后选择“端口设置” 选项卡，如图 1-25。

![](https://bbs.pediy.com/upload/attach/201711/624619_5xpmzod3pnh5yea.png)

图 1-25

         然后将选项卡中的内容选择和图 1-25 一样，单击确定后保存退出。被调试机也是这样设置，并且一定要和调试机设置成一样的配置。然后两台实体机分别打开 HyperTerminal，选择对方的端口进行通信，如果双方信息的收发都没有问题，那么恭喜您，串口已经能正常通信了。

         如果两台实体机之间能稳定通过串口通信，那么下面还需要配置被调试机。笔者被调试机系统版本是 Windows 10 14393 rs1，我们需要修改系统的 BCD 来完成被调试机一端的配置。下面的配置过程都是在命令提示符中操作的。

         首先，先设置被调试机的 Root Partition(Windows 内核) 部分的配置。输入以下命令。

```
$bcdedit /set dbgtransport kdhvcom.dll   
$bcdedit /dbgsettings serial DEBUGPORT:1   BAUDRATE:115200   
$bcdedit /debug on

```

        上面三句分别为：设置调试中用于通信的文件；设置调试方法为串口调试，并且使用 com1 口 (这里的 com1 端口，便是上文中被调试机识别出的调试机端口号) 作为调试端口，波特率为 115200；设置调试开关为开启。

       下面是设置 Hypervisor 层代码调试的配置。命令如下。

```
$bcdedit /hypervisorsettings serial DEBUGPORT:1 BAUDRATE:115200
$bcdedit /set hypervisordebug on
$bcdedit /set hypervisorlaunchtype auto

```

        上面语句用来设置 Hypervisor 层代码由 com1 口作为调试端口，设备波特率为 115200，并开启 Hypervisor 层代码的调试开关。

如果上面的命令都成功的执行，便可以开始设置调试机配置了。

        首先，在调试机上新建一个 connect.bat 文件，打开编辑器，输入下面代码。

 

```
vmdemux.exe -src com:port=com2 

```

然后保存退出编辑器。

         这个命令的意思是，打开串口号为 com2 的串口，并且自动设置波特率，然后创建两个管道文件，分别供 Hypervisor 代码和 Windows 内核代码调试用。

         下面新建一个快捷方式，在 “请键入对象的位置” 中填入 “"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -k com:port=\\.\pipe\vm0,baud=115200,pipe,reconnect”。“"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe"” 部分是 WinDbg 的地址，然后跟的是参数。这些参数意思是，开始内核调试，并通过端口 “\\.\pipe\vm0”，波特率为 115200。这里面“\\.\pipe\vm0” 便是 vmdemux.exe 创建的管道文件，vm0 用于调试 Hypervisor 层代码用。最后将快捷方式重命名为“hypervisor”。

         再新建一个快捷方式，和上面一样，但是这次在 “请键入对象的位置” 中填入“"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -k com:port=\\.\pipe\vm1,baud=115200,pipe,reconnect”。这个用来调试 Windows 内核代码，并将快捷方法重命名为“kernelroot”。

       到此，双机调试的准备已经完成了。下面要先运行 connect.bat 然后在运行两个快捷方式 “hypervisor” 和“kernelroot”。开启被调试机电源，观察 WinDbg 的回显。

         如图 1-26 所示，双机环境便搭建完成了，调试机可以随时中断被调试机的运行。

![](https://bbs.pediy.com/upload/attach/201711/624619_7x25kw3qyd9bktb.png)

图 1-26

3.3. VMware 双机调试环境搭建

         下面我们介绍 VMware 双机调试环境的搭建。这一部分也适用于实体机双机网络调试，只不过我们使用的网络环境是 VMware 虚拟出来的。在开始搭建环境之前，先确保您的主机 CPU 支持 Intel VT-d 和 VT-x(AMD-V AMD-Vi) 特性，并且拥有足够大的内存，至少要大于 8G，否则之后嵌套运行虚拟机时性能会非常低下。

         第一步，修改 VMware 配置文件和虚拟机配置。

        由于我们要在 VMware 虚拟出来的 Windows 系统中安装并运行 Hyper-V 虚拟机，所以需要一些特殊的配置来 “欺骗”Windows 系统，让它认为自己是一台真实的机器，并得以安装和运行 Hyper-V 虚拟机。

![](https://bbs.pediy.com/upload/attach/201711/624619_27yqkhv8rrmqjlc.png)

图 1-27

        下面先在 VMware 中新建一个虚拟机，并分配不少于 4G 的内存，使用 “NAT 网络地址转换” 作为网络方面的配置。然后安装 Windows 系统，笔者这里安装的虚拟机系统版本是 Windows 10 14393 RS1。安装好系统后，关机，下面需要更改一些配置。

         打开 VMware 的 “虚拟机设置” 面板，在 “硬件” 选项卡中，选择 “处理器” 选项，如图 1-27。图 1-27 中，在 “虚拟化引擎” 中的 “首选模式” 选择 “Intel VT-x/EPT 或 AMD-V/RVI”，并勾选上“虚拟化 Intel VT-x/EPT 或 AMD-V/RVI(V)” 和“虚拟化 CPU 性能计数器”两项。点击确定保存配置并退出。

         然后修改创建的虚拟机的配置文件，找到创建的虚拟机文件位置，在虚拟机目录下，找到 Windows 10 x64.vmx 文件 (这里以笔者机器为例，你的机器上应该是: 你的虚拟机名称. vmx)。然后用编辑器打开这个文件，在文件的末尾加上:

```
hypervisor.cpuid.v0 = “FALSE” 
mce.enable = “TRUE” 
vhv.enable = “TRUE”  

```

         在 vmx 文件中可能会存在上面中的配置语句，那么先要检查下原有的配置是否和上面语句相同，若不相同需要修改，然后把 vmx 文件中未出现的配置语句加到文件末尾，保存文件退出。

         “hypervisor.cpuid.v0” 设置成 FALSE 的意义为欺骗虚拟 Windows 系统没有运行在虚拟机中；“mce.enable” 设置成 TRUE 意义为启用 Machine Check Exception(MCE)；“vhv.enable” 设置成 TRUE 意义为启动嵌套虚拟化。

         下面启动修改好配置的虚拟机，然后就可以按照 6.1.2 章节的方法安装和配置 Hyper-V 了。

         第二步，配置虚拟 Windows 主机的调试环境。

         现在我们已经成功在 VMware 虚拟机中安装好 Windows 系统并且可以运行 Hyper-V 虚拟机了。下面还要对 Windows 虚拟机进行配置，实现网络双机调试。

         由于我们在创建虚拟机时网络配置选择的是 “NAT 网络地址转换”，所以虚拟机的 IP 是 192.168.230.175。那么宿主机便是 192.168.230.1。如图 1-28。如果宿主机和 VMware 虚拟的 Windows 系统之间能互相 ping 通对方，说明网络配置正常，可以继续下面的配置。

![](https://bbs.pediy.com/upload/attach/201711/624619_ibkq1dxrwwc5zgs.png)

图 1-28

         同样的，在打开的命令提示符中，我们输入如下的命令：

```
$bcdedit /set dbgtransport kdnet.dll
$bcdedit /dbgsettings NET HOSTIP:192.168.230.1 PORT:50000
$bcdedit /debug on
$bcdedit /hypervisorsettings NET HOSTIP:192.168.230.1 PORT:50001
$bcdedit /set hypervisordebug on
$bcdedit /set hypervisorlaunchtype auto

```

         上面的命令已经在 3.2 章节中做过介绍了，不同的是第 2 个命令和第 4 个命令。这两个命令作用分别是：在被调试机系统启动过程中连接到 IP 为 192.168.230.1，端口为 50000 的调试机，用于调试被调试机的 Windows 内核代码；在被调试机系统启动过程中连接到 IP 为 192.168.230.1，端口为 50001 的调试机，用于调试被调试机的 Hypervisor 层代码。

         以上命令执行完成后，查看配置信息。输入如下命令：

```
$bcdedit /dbgsettings
$bcdedit /hypervisorsettings

```

         结果如图 1-29 所示，这两个命令分别显示出了刚才设置的配置。其中，“key” 字段十分重要的，它相当于调试的凭证，如果凭证不对是无法调试目标主机的，所以它需要我们记录下来，并且妥善保存。注意，其中的 key 值因不同的机器而异。

![](https://bbs.pediy.com/upload/attach/201711/624619_6oswxgknoqslqqe.png)

图 1-29

         上面的配置见表 1-5。

![](https://bbs.pediy.com/upload/attach/201711/624619_1liti2zfs04cxj0.png)

         这张表中的数据需要保存下来，因为在下面的配置中会用到。

         第三步，配置调试机的调试环境。

         现在我们开始配置调试机端。还是和 3.2 章节中介绍的一样，新建一个快捷方式，在 “请键入对象的位置” 中填入“"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windb

g.exe"-k net:port=50000,key=1alb1q6593i2s.1k6o4ts8tzv39.1w72krxwkrv11.3bqw9vhvl5uth”，并命名为 “kernelroot”，用于调试 Windows 内核代码。同样的，再新建一个快捷方式，在“请键入对象的位置” 中填入“"C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\windbg.exe" -k net:port=50001,key=n05t1fb0wyf0.137txzjv7e9ze.1srubjv5hkoo9.1iw56k4pkn6a3”，并命名为“hypervisor”，用于调试 Hypervisor 层代码。

         和 3.2 章节不同的是，网络双机调试不需要运行 vmdemux 命令，不同的端口对应不同的调试目标。下面我们双击运行两个快捷方式，然后开启虚拟机电源，等待被调试机主动连上调试机的 WinDbg。

         如图 1-30，调试机中的 WinDbg 分别捕获到了 Hypervisor 层和 Windows 内核的调试信息，并且可以随时中断被调试机的运行。通过 WinDbg 对被调试机简单的调试操作，会明显的发现在调试速度上有了明显的提升，基本和调试用户态下的程序速度相同。这对我们对 Hyper-V 进行安全研究提供了非常有利的条件。

         所以，综上所述，在硬件条件允许的情况下，还是优先选择使用 VMware 进行双机调试的办法去调试 Hyper-V，这样做确实很省时省力。笔者曾经使用串口进行双机调试，一条汇编指令运行 2-3 秒都是很常见的情况，在调试中浪费了大量的时间。

![](https://bbs.pediy.com/upload/attach/201711/624619_ml2qb6qpsx6wujt.png)

图 1-30

______________________________________________________________________________________________

本文如需引用转载请联系本文作者，看雪 ID：ifyou

[[公告] 推荐好文功能上线，分享知识还可以得雪币！推荐一篇文章获得 20 雪币！](https://zhuanlan.kanxue.com/article-external_link.htm)

上传的附件：

*   [3.pdf](javascript:void(0)) （1.89MB，87 次下载）