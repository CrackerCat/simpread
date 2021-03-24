> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-192746-1.htm)

对于在 Windows 上写代码写习惯的人, 调试是必不可少的手段, 但是转到 Android 以后, 发现调试手段异常简陋, 跟 Windows 简直不是一个级别, 特别是 Android 的内核调试, 网上资料也相对较少, 不过通过一段时间的倒腾, 我终于找到了还算靠谱的调试方法. 就是利用 Emulator + Eclipse 进行 Android Linux 内核调试.

1. 系统预装环境

在目前为止, 都是使用的最新版本的 Android 开发环境

Ubuntu 14.04

Android SDK(adt-bundle-linux-x86_64-20140702)

Android NDK(android-ndk32-r10b-linux-x86_64)

安装好这几个环境以后, 设置一下环境变量

export PATH=$PATH:ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.6/prebuilt/linux-x86_64/bin

ANDROID_NDK_HOME 键值为 Android NDK 安装目录, 设置这个环境变量的目的主要是为了使用 gcc 4.6 版本编译 linux 内核.

export PATH=$PATH:ANDROID_SDK_HOME/sdk/tools

ANDROID_SDK_HOME 是 Android SDK 的安装目录, 设置这个环境变量的目的是方便使用 emulator 命令!

万事具备. 使用前面安装的 Android SDK 创建一个虚拟的设备. 并且确保 emulator -avd Device_Test 这条命令可以启动 Android 模拟器. 先热身下.

![](https://bbs.pediy.com/upload/attach/201409/383488_235e460978153ab6.png)

2.Android Linux 内核编译

2.1 下载 GoldFish 源码

mkdir kernel

cd kernel

git clone http://android.googlesource.com/kernel/goldfish.git

GoldFish 是适配模拟器的内核源码, 如果是要具体适配其他机型, 请选择其他源码, 这边不展开了, 详情参考链接有说明. 如果失败了, 换 https. 我换 https 是因为使用了代理, 现在 google 被墙, 不使用代理搞不动!

git clone https://android.googlesource.com/kernel/goldfish.git

![](https://bbs.pediy.com/upload/attach/201409/383488_05228e8c65daca00.png)

下载过程看你的代理速度了, 而且不能中断. 中断了就要重新来, 特别的麻烦和恶心! 所以我上传了一份到百度云. 和上面 goldfish 出来的一样. 可以考虑去下载

[http://pan.baidu.com/s/1i3yzhbv](http://pan.baidu.com/s/1i3yzhbv)

下载或者解压完成以后会在 kernel 目录下会生成一个 goldfish 文件夹, 进入此目录. 查看所有分支

![](https://bbs.pediy.com/upload/attach/201409/383488_3635acf3a6770815.png)

可以看到, 有很多的版本, 2.6.29 和 3.4 我都测试过. 编译和运行没有任何问题. 所以这边我们拉 2.6.29 的源码

git checkout remotes/origin/android-goldfish-2.6.29

![](https://bbs.pediy.com/upload/attach/201409/383488_d96fd0a2f513e594.png)

然后目录下就有很多文件了, 说明 Android Linux 的源码下载成功!

![](https://bbs.pediy.com/upload/attach/201409/383488_249eb3a357a4af67.png)

3.2 编译 GoldFish 源码

编译源码之前, 请确认已经将 NDK 的编译工具设置到环境变量中. 我们将使用上述这个目录下的交叉编译器 arm-linux-androideabi-gcc

export PATH=$PATH:ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.6/prebuilt/linux-x86_64/bin

然后在 glodfish 目录下用 gedit 打开 Makefile 文件, 找到这两行文字:

#ARCH ?= $(SUBARCH)

#CROSS_COMPILE ?=

修改成

ARCH ?= arm

CROSS_COMPILE ?= arm-linux-androideabi-

![](https://bbs.pediy.com/upload/attach/201409/383488_9f136ae514cef65e.png)

保存文件, 然后

make goldfish_armv7_defconfig

![](https://bbs.pediy.com/upload/attach/201409/383488_517ee2972acdb3ca.png)

注: 用 $make goldfish_defconfig 这样配置也可以编译通过, 模拟器也可以启动, 但是 Android 的开机画机就显示不了,$adb shell 也死活连不上, 原因就是这个 goldfish_defconfig 这个配置文件问题.

Android Linux 的基本编译就设置完成了. 我们先 make 一下

make

![](https://bbs.pediy.com/upload/attach/201409/383488_7ebaa81535915c55.png)

这就表示编译成功了, Linux 的源码是 Linux 上少有的一键 make 过去的软件, 比编译其他 Linux 应用简单不少. 当然到这里编译出来的这个 zImage 已经可以运行了, 但是离我们用来做调试的还是有差距. 我们还要开启内核调试和关闭优化.

3.3 开启调试选项

开启 Linux 内核的调试选项, 先安装依赖性

sudo apt-get install ncurses-dev

然后

make menuconfig

![](https://bbs.pediy.com/upload/attach/201409/383488_5880e7bd096f2ed3.png)

进入内核配置界面, 勾选下列选项, 同时关闭优化

General setup —>

[ ] Optimize for size，进行开启 / 关闭

*    Kernel hacking   
    

*    Compile the kernel with debug info   
    

*    KGDB: kernel debugging with remote gdb —>        
    

*    Enable dynamic printk() call support   
    

关闭 Linux 内核优化比较麻烦. 我通过和朋友讨论, 以及网络搜索还没有找到很好的解决办法, 原因是默认的 Linux 内核编译是开启 - O2 优化的, 这种模式之下会造成 gdb 和实际的源码对不上, 相信使用过 windbg 调试 - O2 的朋友都有这个经历, 所以我们需要关闭 Linux 的 - O2, 不过目前还没有很好的解决办法下面这篇文章讨论的解决办法是. 针对文件进行关闭优化. 下面这两篇文章的讨论都非常有意义:

http://www.lenky.info/archives/2013/03/2238

http://www.ibm.com/developerworks/cn/linux/l-kdb/

这边我们将 - Os 和 - O2 都调成 - O. 针对具体文件关闭优化, 这边就不搞了. 具体到自己的调试任务的时候再看.

![](https://bbs.pediy.com/upload/attach/201409/383488_d6d5f419546cff3d.png)

再进行编译,

make -B

选项 - B 以强制所有内核源文件全部重新编译 (因为我前面编译过一次了, 为了保险起见, 就让目标文件全部重新生成吧) 当出现这个画面, 就表示编译成功了

![](https://bbs.pediy.com/upload/attach/201409/383488_ed360f49a31f20d9.png)

4.Android Linux 内核调试

使用 emulator 启动我们编译的内核试试

emulator -verbose -show-kernel -kernel ~/kernel/goldfish/arch/arm/boot/zImage -avd Device_Test

![](https://bbs.pediy.com/upload/attach/201409/383488_d1a4755feed8705d.png)

没错, 启动的就是我们的内核 2.6.29 时间也对的上. 说明我们编译的内核是可以运行的. 下一步使用这条命令

emulator -verbose -show-kernel -kernel -netfast ~/kernel/goldfish/arch/arm/boot/zImage -avd Device_Test -qemu -gdb tcp::1234,ipv4, -S

这条命令会在 tcp 端口的 1234 监听. 加了 - S 还会暂停下来, 等待着 gdb 链接上来. 这时候我们开启 NDK 目录下面的 gdb. 链上去然后

target remote localhost:1234

![](https://bbs.pediy.com/upload/attach/201409/383488_3f71d2b65aa74081.png)

还可以测试几条命令, 看看源码是否跟上了.

![](https://bbs.pediy.com/upload/attach/201409/383488_ac79638ececfec45.png)

到这里为止, 基本上是用 gdb 连上 emulator 进行内核调试应该没问题了. 但是仅仅到这里那离 windbg 的调试还是差好几条街. 所以我们还是需要一个更好的调试方法. 是用 eclipse 来作为调试的前端!

5.Eclipse 前端

是用 Eclipse 作为前端的好处是, 无论是在 windows, 在 linux 下面都没有问题. 可以在一台 windows 的机器上, 远程调试 android 内核. 所以为了截图方便, 我下面的操作都是在 windows 上弄的, 在 Linux 上也是一样! 当然要在 windows 上进行调试, 首先要将上面的 gold 目录复制到 windows 的机器上, 或者是共享给 windows. 这里就不展开了!

运行 Eclipse, 点击菜单 Help->Install New Software… 在弹出的对话框里点击 Work with: 后面的下拉按钮, 选择 Kepler – http://download.eclipse.org/releases/kepler

不同的 Eclipse 版本选择不一样, 与自己下载的版本一致一即可. 然后在下面的选择框中将以下选项安装上

Programming Languages

C/C++ Autotools support

C/C++ Visual C++ Support

C/C++ Development Tools

C/C++ Development Tools SDK

Linux Tools

GDB Tracepoint Analysis

Mobile and Device Development

C/C++ GDB Hardware Debugging

安装好后自动重启 Eclipse 即可. 再配置点击菜单 Window -> Preferences 在弹出的对话框中, 点击左边的 General->Workspace 将右边的 Build automatically 复选框不选中.

再点击对话框左边的 C/C++->Indexer, 将右边的 Enable indexer 和 Automatically update the index 两复选框不选中.

接下来就简单了. 创建一个工程, 点击菜单 File->New->Project… 在弹出的对话框中选择 C/C++->C Project 再点击 Next > 按钮

![](https://bbs.pediy.com/upload/attach/201409/383488_912b14d3efb5834d.png)![](https://bbs.pediy.com/upload/attach/201409/383488_d54ad762a54c2e83.png)

其中 Project name: 为工程名, 可自定义. 而 Location: 则为工程文件所在路径, 此处设置为我们下载的源码路径而 Project type: 则设置为 Makefile project/Empty Project, Toolchains: 则设置为 Linux GCC, 如果是 windows 设置成 Android GCC 最后点击 Finish 即可.

接下来 Windows 和 Linux 都一样, 进行 DEBUG 配置, 在 Project Explorer 里右击刚创建的 Linux_Kernel 项目, 在右键菜单中点击 Debug As->Debug Configurations… 在弹出的对话框中双击 GDB Hardware Debugging 然后配置调试选项如下图

![](https://bbs.pediy.com/upload/attach/201409/383488_5be760d198848838.png)

将 C/C++ Application: 栏设置为 Linux Kernel 源码编译出来的 vmlinux 文件所在路径 (包含文件名), 然后将 Disable auto build 选上, 切换到 Debugger 页, 修改配置如下截图.

![](https://bbs.pediy.com/upload/attach/201409/383488_7b486a33fe30e558.png)

这里是设置 gdb 的路径还有远程地址和端口. Gdb 的路径在 ndk 安装目录下的如下路径

\toolchains\arm-linux-androideabi-4.6\prebuilt\windows-x86_64\bin

远程地址, 我的 ubuntu 机器是 192.168.1.2 这个随机应变即可. 端口是我们运行 emulator 命令定义的端口. 搞定这个切换到 Startup 页面

![](https://bbs.pediy.com/upload/attach/201409/383488_e79163d9d0f07c74.png)

将 Reset and Delay(seconds) 和 Halt 还有 Load image 复选框的勾都去掉. 然后点击 Debug, 这时候就停在第一条指令了

![](https://bbs.pediy.com/upload/attach/201409/383488_0974fbb44c513770.png)

这时候还不能按 F5, F6 单步. 我们在 Execut 窗口指定到 main.c 然后在 start_kernel 上下个断点也可以再 Console 窗口敲命令 break start_kernel. 然后敲入命令 C.

![](https://bbs.pediy.com/upload/attach/201409/383488_260e5bbe7245cc1d.png)

这时候就停在了 Linux 内核的入口函数 start_kernel. 也可以使用 F5,F6 了. 寄存器显示各方面都可以了. 如果在 Windows 上, 有一个毛病, 源文件都要自己重新指定路径. 不然认不到. 默认都是编译路径 / home/xxx 什么的. 要重新指定成 Windows 的盘符形式. 不过在 Linux 上调试就没有这个问题了!

![](https://bbs.pediy.com/upload/attach/201409/383488_2c4f0b8c2247c663.png)

这个调试差不多是搞起走了. 如果是分析 Android 的源码, 看一看跟一下还是很不错的. 不过还是有一个问题没有解决, 关于汇编和符号不对应的问题.

大家可以群策群力搞一下 QQ 群: 127285697

这边文档格式排起来真麻烦. 我在我的博客也发了这篇文章.

[http://www.joenchen.com/archives/1093](http://www.joenchen.com/archives/1093)

参考链接:

http://wenku.baidu.com/view/95c69448e518964bcf847c2f.html

http://blog.csdn.net/flydream0/article/details/7070392

http://www.lenky.info/archives/2013/03/2238

http://blog.csdn.net/liushuaikobe/article/details/8646555

http://x-slam.com/da_jian_eclipse_qemu_gdb_diao_shi_linux_kernel_huan_jing

[[公告] 名企招聘！](https://job.kanxue.com/position-list-1-99-99-99-99-99.htm)