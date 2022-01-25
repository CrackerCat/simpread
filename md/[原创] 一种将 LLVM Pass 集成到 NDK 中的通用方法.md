> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271271.htm)

> [原创] 一种将 LLVM Pass 集成到 NDK 中的通用方法

关于 “如何将自己写的 LLVM Pass 集成到 NDK 中” 这个问题，目前网上并没有很完美的方法，并且大多数已经过时。[史上最优雅的 NDK 加载 pass 方案](https://xz.aliyun.com/t/6643#toc-6)此贴中提出的方法也很麻烦，在我看来并不算太 “优雅”。经过我的一番折腾，最终摸索出了一种比较简单实用的方法，可供参考。

0x00. 概览
========

简单来说，本文要介绍的方法是下载 NDK 中`llvm-android`部分的源码，在`llvm-android`中的`llvm-project`子项目中加入自己的 Pass 后与整个项目一起编译，用编译好的文件替换掉 NDK 中的工具链。  
该方法有以下优点：

*   Windows、macOS、Linux 通用，各 NDK 版本也通用
*   理论上加入自己的 Pass 后不会出现不兼容的问题
*   操作过程简单易懂

当然也有缺点：

*   无法直接照搬 OLLVM、Hikari、Armariris 等现成项目的源码，需要手动做一些迁移
*   第一次编译比较耗时

0x01. 操作流程
==========

本文并不是直接将最终的操作方法摆在大家面前，还讲解了为什么要这么操作，因此过程会显得较冗长，需要有一点耐心看下去。

1. 环境准备
-------

如果是 Windows，首先需要准备 Linux 虚拟机进行交叉编译，因为 NDK 不支持在 Windows 上直接编译，我这里使用的是 **Ubuntu 20.04.3**；macOS 则不需要。NDK 版本我选择的是 **23.1.7779620**。  
本文以 Windows+Linux 虚拟机为例讲解，macOS 下的操作大同小异。  
以下使用的指令全部以 root 权限执行。

2. 下载 llvm-android 源代码
----------------------

NDK 使用的是`git-repo`这一工具管理，而不是`git`，所以首先需要安装`git-repo`：

```
curl https://storage.googleapis.com/git-repo-downloads/repo > /usr/bin/repo
chmod a+x /usr/bin/repo

```

上面这个地址需要科学上网才能访问，我们可以换成国内的清华源：

```
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > /usr/bin/repo
chmod a+x /usr/bin/repo

```

在`$NDK_PATH\toolchains\llvm\prebuilt\windows-x86_64\AndroidVersion.txt`中查看 NDK 使用的 LLVM 版本，在我使用的 NDK 中，AndroidVersion.txt 的内容是：

```
12.0.8
based on r416183c1

```

在`https://android.googlesource.com/toolchain/llvm_android/`中找到对应版本的 llvm-android 源码：  
![](https://bbs.pediy.com/upload/attach/202201/910514_AY6XK8JRE7TF4TV.png)  
Google 的文档中给出了下载 llvm-android 源代码的方法，但这里默认下载的是最新版本：

```
mkdir llvm-toolchain && cd llvm-toolchain
repo init -u https://android.googlesource.com/platform/manifest -b llvm-toolchain
repo sync -c

```

我们需要做一些操作来换成我们想要的版本，在`$NDK_PATH\toolchains\llvm\prebuilt\windows-x86_64`目录下找到一个 **manifest_xxxx.xml** 文件，我这里是 manifest_7714059.xml。执行完下列指令后：

```
mkdir llvm-toolchain && cd llvm-toolchain
repo init -u

```

将 manifest_7714059.xml 复制到`.repo/mainifests`文件夹中（注意. repo 是隐藏文件夹，并且从 Windows 复制到 Linux 会有格式转换的问题，这里建议使用 VSCode 的 SSH-Remote 插件避免上述问题）：  
![](https://bbs.pediy.com/upload/attach/202201/910514_7JRCVD7UTBZAHGG.png)  
继续执行：

```
repo -m manifest_7714059.xml
repo sync -c

```

这里也要把 Google 的地址换成清华源，替换规则见 [Android 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)，替换后完整的指令如下：

```
mkdir llvm-toolchain && cd llvm-toolchain
repo init -u
https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b llvm-toolchain
repo -m manifest_7714059.xml
repo sync -c

```

并且 manifest_7714059.xml 中的包含地址也要替换，否则即使科学上网，在下载的时候也非常慢：  
![](https://bbs.pediy.com/upload/attach/202201/910514_SBFKTMWCMUKQKA2.png)  
`repo sync -c`指令会下载一大堆的源代码和预编译文件，需要耗费十来分钟的样子，喝杯茶慢慢等吧。

3. 编译 llvm-android 源代码
----------------------

在编译之前需要提前安装一些环境，否则编译会出错：

```
apt install cmake bison

```

如果是在 Ubuntu 20.04.3 下还需要做一个软链接，否则会报错`ImportError: libffi.so.6: cannot open shared object file: No such file or directory`。具体操作如下：  
![](https://bbs.pediy.com/upload/attach/202201/910514_A867QTEZ993UG2B.png)  
然后就可以开始愉快且漫长的编译了（大概需要一两个小时，取决于机器性能）：

```
python toolchain/llvm_android/build.py --no-build linux

```

因为我是在 Windows 环境使用 NDK，所以无需编译 Linux 下的 toolchain，这里加上`--no-build linux`参数。  
另外编译的时候最好把虚拟机内存开到 8G 以上，我开的是 8G 内存，编译的时候还会因为内存不足时不时中断，如果中断了重新运行编译指令就好。  
编译结束后可以在`out`文件夹中找到编译好的内容：  
![](https://bbs.pediy.com/upload/attach/202201/910514_R7RWFYGVBQMW7A3.png)

4. 加入自己的 Pass 并重新编译
-------------------

在这里我要推销一下我的项目 [Pluto-Obfuscator](https://github.com/bluesadi/Pluto-Obfuscator) ，如果使用的是 OLLVM, Armariris 等，需要注意一下版本适配的问题。  
此时我们需要向`toolchain/llvm-project/llvm/lib/Transforms/Obfuscation/`中加入自己的代码：  
![](https://bbs.pediy.com/upload/attach/202201/910514_CSAMC5FEB58ZMER.png)  
向`toolchain/llvm-project/llvm/lib/Transforms/IPO/PassManagerBuilder.cpp`中加入几段代码：  
![](https://bbs.pediy.com/upload/attach/202201/910514_J6QKXBPJFSDRU5J.png)  
![](https://bbs.pediy.com/upload/attach/202201/910514_3SVJCCZ72JJZFXN.png)  
修改`toolchain/llvm-project/llvm/lib/Transforms/IPO/CMakeLists.txt`：  
![](https://bbs.pediy.com/upload/attach/202201/910514_GW3D8ZDYAH5AVB4.png)  
改好之后重新编译，重新编译的速度会快很多，一分钟左右就能搞定：

```
python toolchain/llvm_android/build.py --no-build linux

```

编译好后`out/install/windows-x86/clang-dev`中对应的就是 NDK 中的`toolchains\llvm\prebuilt\windows-x86_64`部分：  
![](https://bbs.pediy.com/upload/attach/202201/910514_RRUVB62M3JTY2P7.png)  
![](https://bbs.pediy.com/upload/attach/202201/910514_SY5BZFEGQJFZU8B.png)  
少了一些东西，但是无关紧要，我们直接替换就好。

0x03. 效果测试
==========

我复制了一份 NDK，命名为`pluto-r23`，并将其中`toolchains\llvm\prebuilt\windows-x86_64`文件夹里的内容替换成我们刚刚编译的内容：  
![](https://bbs.pediy.com/upload/attach/202201/910514_DMP3VGWK95PQBQ3.png)  
随便写一个 Native 项目测试：  
![](https://bbs.pediy.com/upload/attach/202201/910514_QBUPNU4SWPSVUM8.png)  
设置 NDK 地址：  
![](https://bbs.pediy.com/upload/attach/202201/910514_C58B46UCRAT7AAJ.png)  
加上混淆参数：  
![](https://bbs.pediy.com/upload/attach/202201/910514_US9TWRXSRVVPHD2.png)  
编译然后查看混淆效果：  
![](https://bbs.pediy.com/upload/attach/202201/910514_RVXUDY54976MTJW.png)  
X86 架构和 ARM 架构均混淆成功：  
![](https://bbs.pediy.com/upload/attach/202201/910514_R8M3GX94WHWSM88.png)  
![](https://bbs.pediy.com/upload/attach/202201/910514_EPZ2SXZQ33JHMAT.png)

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 32 分钟前 被 34r7hm4n 编辑 ，原因：

[#混淆加固](forum-161-1-121.htm)