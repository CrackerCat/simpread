> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269889.htm)

> [原创] 手动编译 Hluda Frida Server

> 本文基于 ubuntu 21.04 操作

1. 搭建编译环境
---------

### 1.1 Install dependencies

```
sudo apt update
sudo apt-get install build-essential tree ninja-build gcc-multilib g++-multilib lib32stdc++-9-dev flex bison xz-utils ruby ruby-dev python3-requests python3-setuptools python3-dev python3-pip libc6-dev libc6-dev-i386 -y
 
sudo gem install fpm -v 1.11.0 --no-document
python3 -m pip install lief

```

### 1.2 Setup ndk

ndk 版本与你想要编译的版本相关，在其`/releng/setup-env.sh`注明了需要的 NDK 版本

 

![](https://bbs.pediy.com/upload/attach/202110/757977_RUXY7ZWMW9HK763.png)

 

这里以最新版的 frida ndk 依赖 22 进行

 

ndk 下载网址：https://developer.android.com/ndk/downloads?hl=zh-cn

```
wget https://dl.google.com/android/repository/android-ndk-r22b-linux-x86_64.zip
unzip android-ndk-r22b-linux-x86_64.zip

```

```
sudo mv android-ndk-r22b /opt/
 
#add env variables
export ANDROID_NDK_ROOT='/opt/android-ndk-r22b'

```

### 1.3 Setup nodejs

https://github.com/nvm-sh/nvm

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
 
# install node 10
nvm install 10

```

2. 编译 frida
-----------

### 2.1 最新版

当前最新版本是：15.1.5

```
git clone --recurse-submodules https://github.com/frida/frida

```

Apply hluda patch

```
git clone https://github.com/AAAA-Project/Patchs.git
 
cd frida/frida-core/
 
git am ../../Patchs/strongR-frida/frida-core/*.patch
 
# 回到frida 根目录
cd ..

```

编译：

 

编译时会自动下载 对应的 toolchain 和 sdk。

```
make core-android-arm
make core-android-arm64
make core-android-x86
make core-android-x86_64

```

when compile completed, into `build/frida-android-arm/bin` ,you will see:

 

![](https://bbs.pediy.com/upload/attach/202110/757977_8M8JR34GXEPURYW.png)

### 2.2 老版本

看了看 [Patchs](https://github.com/AAAA-Project/Patchs) 的 commit message、时间, 基本就能知道 commit 对应的 patch，对应哪些版本：

 

![](https://bbs.pediy.com/upload/attach/202110/757977_9RRVSPHBJ7DM8KG.png)

 

看看编译 14.2.12 怎么弄

```
git clone --recurse-submodules https://github.com/frida/frida.git
cd frida
git checkout 14.2.12

```

这里有个坑，当 checkout 的时候，仅 frida 这个仓库回滚到 14.2.12，其中的 submodule 依然是最新的，要让所有 submodule 也是 14.2.12 时的版本才行：

```
git submodule update --recursive

```

检查一下需要的 ndk 版本，依然是 22：  
![](https://bbs.pediy.com/upload/attach/202110/757977_H8M5KHUXTMBB94Y.png)

 

checkout Patchs 到 14.2.12：  
![](https://bbs.pediy.com/upload/attach/202110/757977_CCDBJCJYJHZUA5J.png)

```
git checkout 8e1308b

```

Apply hluda patch:

```
cd frida/frida-core
git am ../../Patchs/strongR-frida/frida-core/*.patch

```

check 一下，没报错就行。

 

接下来和之前的编译步骤一样

```
cd frida
 
make core-android-arm
make core-android-arm64
make core-android-x86
make core-android-x86_64

```

![](https://bbs.pediy.com/upload/attach/202110/757977_MXASVRFATR34XY5.png)

 

Git History - https://githistory.xyz/ 在某些情况下确实有用：

 

![](https://bbs.pediy.com/upload/attach/202110/757977_ER4U44RJQ9PYBZ5.png)

 

参考：

1.  hluwa - [actions build.xml](https://github.com/lushann/strongR-frida-android/blob/main/.github/workflows/build.yml)

[2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

最后于 7 小时前 被 lushanu 编辑 ，原因：

[#HOOK 注入](forum-161-1-125.htm)