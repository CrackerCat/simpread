> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/10701419.html)

一、编译 LineageOS 源码[#](#一、编译lineageos源码)
--------------------------------------

### 准备[#](#准备)

*   设备：小米 MIX2
*   设备代号：chiron
*   Android 版本：9.0
*   PC 配置：
    *   系统：Ubuntu18.04
    *   至少 12G RAM
    *   至少 200GB 剩余硬盘空间
    *   良好的网络环境

### 1. 下载并解压 SDK[#](#1下载并解压sdk)

sdk 中包含 fastboot 和 adb

#### 下载[#](#下载)

`wget https://dl.google.com/android/repository/platform-tools-latest-linux.zip`

#### 解压[#](#解压)

`unzip platform-tools-latest-linux.zip -d ~`

#### 添加到环境变量[#](#添加到环境变量)

`gedit ~/.profile`

输入：

```
# add Android SDK platform tools to path
if [ -d "$HOME/platform-tools" ] ; then
    PATH="$HOME/platform-tools:$PATH"
fi


```

保存。

使改动生效:

`source ~/.profile`

### 2. 安装依赖[#](#2安装依赖)

#### 安装必要库和工具[#](#安装必要库和工具)

`sudo apt-get install bc bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev libwxgtk3.0-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev`

#### 安装 openjdk-8-jdk[#](#安装openjdk-8-jdk)

`sudo apt install -y openjdk-8-jdk`

### 3. 配置源[#](#3配置源)

#### 创建 repo 存放目录[#](#创建repo存放目录)

`mkdir -p ~/bin`

#### 创建源码存放目录[#](#创建源码存放目录)

`mkdir -p ~/android/lineage`

× 注：请确保该目录所在的磁盘有足够的空间（至少 200G）

#### 安装 repo[#](#安装repo)

`curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo`

`chmod a+x ~/bin/repo`

#### 将`~/bin`放入环境变量[#](#将bin放入环境变量)

`gedit ~/.profile`

加入：

```
# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi


```

使改动生效:  
`source ~/.profile`

##### 初始化 repo

`cd ~/android/lineage`

`repo init -u https://github.com/LineageOS/android.git -b lineage-16.0`

× 注：这里的 lineage-16.0 是分支名, 对应 Android 9.0。更多分支请浏览：[https://github.com/LineageOS/android](https://github.com/LineageOS/android)

### 4. 拉取源码[#](#4拉取源码)

`repo sync`

### 5. 配置源码[#](#5配置源码)

`cd device/xiaomi`

`git clone https://github.com/LineageOS/android_device_xiaomi_chiron -b lineage-16.0 chiron`

#### 提取 vendor 信息[#](#提取vendor信息)

提取手机厂商的库。这一步将手机连接电脑，并保证手机能被 adb  
命令操控，还需要手机授予 adb root 权限。

`adb shell su -c setenforce 0`

`cd chiron`

`./extract-files.sh`

> 提取需要点时间，需耐心等待

#### 拉取 kernel[#](#拉取kernel)

`cd kernel/qcom`

`git clone https://github.com/LineageOS/android_kernel_qcom_msm8998 -b lineage-16.0 msm8998`

#### 配置构建环境[#](#配置构建环境)

`source build/envsetup.sh`

#### 配置设备信息[#](#配置设备信息)

`breakfast chiron`

### 6. 配置构建工具[#](#6配置构建工具)

#### 配置 ccache[#](#配置ccache)

`gedit ~/.bashrc`

加入：

`export USE_CCACHE=1`

使改动生效:

`source ~/.bashrc`

执行：

`ccache -M 50G`

来设置缓存大小

× 注：ccache 默认在 home 目录，请确保 home 目录有足够的空间。如果想自定义 ccache 的目录，可以在`~/.bashrc`文件里加入`export CCACHE_DIR=/path/to/.ccache`。缓存大小根据自己硬盘大小设置，设置 25G 以上可以显著提高源码的构建速度。

#### 配置 jack[#](#配置jack)

`gedit ~/.bashrc`

加入：

`export ANDROID_JACK_VM_ARGS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx8G"`

使改动生效:

`source ~/.bashrc`

### 7. 构建[#](#7-构建)

#### 构建整个源码[#](#构建整个源码)

`croot`

`brunch chiron`

× 注：构建成功后，在源码目录执行`cd $OUT`进入编译好的 ROM 的存放目录。`lineage-16.0-xxxxxxxx-UNOFFICIAL-chiron.zip`为编译好的刷机包。

#### 只构建某个模块[#](#只构建某个模块)

`mmm <模块名>`

例如构建 frameworks 中的 base

`mmm frameworks/base`

#### 打包成 system.img[#](#打包成systemimg)

有时候我们只修改 system 里的模块，没必要编译整个源码，就只打包 system.img

`make snod`

#### 编译 system[#](#编译system)

`make systemimage`

### 8. 刷入手机（可选）[#](#8刷入手机（可选）)

#### 下载 twrp[#](#下载twrp)

[https://dl.twrp.me/chiron/](https://dl.twrp.me/chiron/)

#### 刷入 trwp[#](#刷入trwp)

刷入 recovery 前，要先解锁手机的 bootloader，如何解锁，各个厂商不太一样，这里就不阐述了。

`adb reboot bootloader`

`fastboot flash recovery twrp.img`

`fastboot reboot`

#### 刷入 system.img[#](#刷入systemimg)

我们调试源码时，如果不想刷入整个 ROM，可以只刷入 system.img。以下命令将 system.img 刷入 system 分区

`fastboot flash system system.img`

#### 刷入 ROM[#](#刷入rom)

将编译好的刷机包，通过`adb push`命令将刷机包传输到手机存储。进入 twrp 界面，擦除 system 分区，data 分区。选择手机存储中的刷机包，刷入!!!

### 9. 常见问题[#](#9常见问题)

#### (1) 编译目标设备为 emulatorx86 时在编译时出错，提示 yasm 找不到。[#](#1-编译目标设备为emulatorx86时在编译时出错，提示yasm找不到。)

安装 yasm

`sudo apt-get install yasm`

#### (2) adb 命令提示没有权限[#](#2-adb命令提示没有权限)

`lsusb`

找到类似一行：

`Bus 001 Device 005: ID 18d1:4ee7 Google Inc.`

编辑 51-android.rules：

`sudo gedit /etc/udev/rules.d/51-android.rules`

输入类似的内容：

```
# MIX2 normal
SUBSYSTEM=="usb", ATTRS{idVendor}=="18d1", ATTRS{idProduct}=="4ee7",MODE="0666",GROUP="plugdev"
# MIX2 fastboot
SUBSYSTEM=="usb", ATTRS{idVendor}=="18d1", ATTRS{idProduct}=="d00d",MODE="0666",GROUP="plugdev"


```

× 注：idVendor,idProduct 分别为 lsusb 命令显示的 ID。开机状态和 fastboot 模式都需要添加权限，所以，需要增加两行。

更改文件权限：

`sudo chmod a+r /etc/udev/rules.d/51-android.rules`

### 二、模拟器使用 LineageOS[#](#二、模拟器使用lineageos)

将 emulator 所在的目录加入环境变量

在源码目录，`cd $OUT`

#### 设置模拟器路径[#](#设置模拟器路径)

```
# 设置模拟器目录
if [ -d "/android_source_path/prebuilts/android-emulator/linux-x86_64" ] ; then
    PATH="/android_source_path/prebuilts/android-emulator/linux-x86_64:$PATH"
fi


```

#### 设置 OUT 目录[#](#设置out目录)

`export ANDROID_PRODUCT_OUT=/android_source_path/out/target/product/emulatorx86`

#### 使用编译好的镜像启动（arm）：[#](#使用编译好的镜像启动（arm）：)

`emulator64-arm -kernel ./prebuilts/qemu-kernel/arm64/kernel-qemu -sysdir $ANDROID_PRODUCT_OUT -system $ANDROID_PRODUCT_OUT/system.img -data $ANDROID_PRODUCT_OUT/userdata.img -ramdisk $ANDROID_PRODUCT_OUT/ramdisk.img`

#### 使用编译好的镜像启动（x86）：[#](#使用编译好的镜像启动（x86）：)

`emulator64-x86 -kernel ./prebuilts/qemu-kernel/x86/kernel-qemu -sysdir $ANDROID_PRODUCT_OUT -system $ANDROID_PRODUCT_OUT/system.img -data $ANDROID_PRODUCT_OUT/userdata.img -ramdisk $ANDROID_PRODUCT_OUT/ramdisk.img`

### 参考[#](#参考)

*   [https://wiki.lineageos.org/devices/chiron/build](https://wiki.lineageos.org/devices/chiron/build)
*   [https://www.htcp.net/741.html](https://www.htcp.net/741.html)
*   [https://www.isthnew.com/archives/build-lineageos.html](https://www.isthnew.com/archives/build-lineageos.html)
*   [http://blog.csdn.net/luoshengyang/article/details/6566662/](http://blog.csdn.net/luoshengyang/article/details/6566662/)

[![](http://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**