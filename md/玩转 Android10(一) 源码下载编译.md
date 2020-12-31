> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484198&idx=1&sn=15841e76329922aff977455671d56f6f&chksm=cebb4668f9cccf7e2643e21892bc0172c134e9438ffefe57bd022490f78a7ce93d2c7b3aa209&scene=21#wechat_redirect)

一、前期准备
------

**测试手机准备**  
 测试设备：  
   oneplus 3/3T  
 设备代号：  
   oneplus3  
 Android 系统版本：10.0  
**PC 环境配置**：  
 开发环境:  
  Windows10 64bit+VMware+ubuntu  
 虚拟机版本:  
  VMware Workstation 15 Player  
 Ubuntu 系统分配情况:  
  版本 Ubuntu18.04  
  内存至少 12G RAM  
  硬盘空间至少 200GB  

二、配置 adb 和 fastboot
-------------------

**1. 下载 platform-tools 压缩包**  

 下载地址: https://dl.google.com/android/repository/platform-tools_r30.0.5-linux.zip  

**2. 解压压缩包到指定目录**  
 执行如下命令:  

> mkdir -p  /home/qiang/Android  
> unzip  platform-tools_r30.0.5-linux.zip  -d  /home/qiang/Android

 如下图所示:![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRricgic9AHLRFMJgXqTpKmZoHVibYTZgh9fIPcUChFJEibsdwMZzESGFRHA/640?wx_fmt=jpeg)  
**3. 配置 adb 和 fastboot 命令**  

  用 vim 编辑器打开~/.bashrc 文件，添加如下文本并保存

> #add Android Sdk  platform tools to path  
> #add START  
> export ADB_PATH=/home/qiang/Android/platform-tools  
> export PATH=$PATH:$ADB_PATH  
> #add END

 执行 source  ~/.bashrc 命令更新环境变量, 打开终端查看 adb 和 fastboot 是否生效。如下图所示:![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRVnZicLVp5LzKt1BgSw65cgGKKpvVWE7DacwoXTibynm65mxEsq5iaMZbg/640?wx_fmt=jpeg)

三、下载编译 LineageOs
----------------

_**1. 安装依赖**_  
 执行以下命令安装必要库和工具  

>     sudo  apt-get  install  bc bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev libwxgtk3.0-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev

 执行以下命令安装 openjdk-8-jdk  

> sudo apt-get install  openjdk-8-jdk

_**2. 创建必要目录**_  
 执行如下命令创建源码保存目录:  

> mkdir -p /home/qiang/lineageAndroid10

 执行如下命令创建 git-repo 工具保存目录  

> mkdir -p /home/qiang/bin

_**3. 安装 repo 命令**_  
   由于使用 google 的 repo 源需要挂代理，所以我们用国内的清华 repo 源, 无需挂代理就可以很快的下载 Android 的源码了。执行如下命令下载 repo:  

> curl   https://mirrors.tuna.tsinghua.edu.cn/git/git-repo  -o /home/qiang/bin/repo  
> chmod  +x  /home/qiang/bin/repo

 将 repo 命令加入环境变量  
 使用 vim 工具打开~/.bashrc 文件, 命令如下:  

> vim  ~/.bashrc

 将如下内容加入文件中:  

> export  REPO_PATH=/home/qiang/bin/repo  
> export  PATH=$PATH:$REPO_PATH  
> export  REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'

 使用如下命令更新环境变量:  

> source  ~/.bashrc

_**4. 下载源码**_  
 lineageOs 中 17.1 版本对应 Android 10。执行如下命令初始化代码仓库:  

> cd  /home/qiang/lineageAndroid10  
> repo  init  -u  https://github.com/LineageOS/android.git  -b  lineage-17.1

 初始化完成之后，执行如下命令下载源码:  

> repo  sync  -j4

_**5. 使用不死脚本下载源码**_  
 (1). 由于在终端执行 repo sync 命令下载源码容易意外终止退出，可以使用如下的不死脚本进行下载。将以下脚本内容保存为 download.sh:  

> #!/bin/bash  
> echo  "==========start repo sync==="  
> repo  init  -u  https://github.com/LineageOS/android.git  -b  lineage-17.1  
> repo   sync  -j4  -d  --force-sync  --no-clone-bundle  
> while  [$?!=0];  
> do  
>   echo  "===resync==="  
>   repo  sync  -j4   -d  --force-sync  --no-clone-bundle  
> done  

 (2) 将 download.sh 文件复制到 / home/qiang/lineageAndroid10, 并执行 download.sh 脚本  
命令参考:

> cp  /home/qiang/download.sh     /home/qiang/lineageAndroid10/download.sh  
> cd  /home/qiang/lineageAndroid10  
> chmod  777  /home/qiang/lineageAndroid10/download.sh  
> ./download.sh

完成以上工作之后，就可以去喝喝茶、晒晒太阳等待源码同步完成。

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF1zbMLiadYicdLiaWicYYKaqo02fXibupoMQW7VIChYCoichk4WQT2EhlLTVPYk1Liah2gibrnMJIqicDETtgg/640?wx_fmt=jpeg)