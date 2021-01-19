> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265418.htm)

#前言  
frida 的官方文档写的并不是很好，有些例子好像还有些问题。这就不得不去研究它的源码了。frida 的源码有许多个模块，我们这只关注 **[frida-java-bridge](https://github.com/frida/frida-java-bridge)** 这个模块。为什么呢？这个模块实现了 **js 世界到 java 世界的单向通道**。所以我们主要的代码在这。可以看看这篇文章对 frida-java 的介绍 **[frida 源码阅读之 frida-java](https://bbs.pediy.com/thread-229215.htm)**。  
这里我就记录一下 frida-java 的编译环境搭建

 

#环境  
VMware 12  
Ubuntu16  
Android8.0  
Google pixel (已 root)

 

#步骤  
1. 下载安装配置 Ubuntu16 需要使用到的软件  
1. 安装配置 JDK  
2. 安装配置 SDK  
3. 安装配置 NDK  
4. 编译安装 Nodejs  
5. 编译运行 frida-java

 

这里就跳过安装 Ubuntu16 虚拟机的步骤了

 

#下载安装配置 Ubuntu16 需要使用到的软件  
**配置 apt 国内软件源镜像**  
使用[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)

 

将软件源的配置拷贝到

```
sudo gedit /etc/apt/sources.list

```

更新 apt

```
sudo apt-get update

```

**安装 curl**

```
sudo apt-get install curl

```

**安装 git**

```
sudo apt-get install git

```

 

#安装配置 JDK  
下载 jdk  
http://jdk.android-studio.org/  
这里为了方便我在家目录创建一个 work 目录，有关环境我都安装到这个目录下。读者可以自行选择目录、

```
mkdir work

```

将下载的 jdk 解压

```
tar -zxvf  jdk-8u77-linux-x64.tar.gz

```

配置环境变量

```
sudo gedit /etc/profile

```

加入配置如下，请修改自己的 jdk 位置

```
#set java env
export JAVA_HOME=/home/fj/work/jdk1.8.0_231
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH

```

加载配置

```
source /etc/profile

```

查看是否配置成功

```
fj@ubuntu:~/work/jdk1.8.0_231$ java -version
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)

```

 

#安装配置 SDK  
这里我借助 [android studio](http://www.android-studio.org/) 进行下载 sdk。  
android studio 下载地址：http://www.android-studio.org/

 

下载后移动到 work 目录，解压运行

```
tar -zxvf android-studio-ide-191.5977832-linux.tar.gz
cd android-studio/bin
./studio.sh

```

其他操作和 Windows 上的 as 一样

 

安装 sdk 版本, 我们下载 29 版本。

 

这里可以看到 as 下载的 sdk 在 / home/fj/Android/Sdk，所以我们需要配置这个目录的环境变量

```
sudo gedit /etc/profile

```

配置如下, 请注意修改自己 sdk 的位置:  
**注意配置 build-tools/30.0.3 因为编译时会用到 dx**

```
#set sdk
export ANDROID_SDK_HOME=/home/fj/Android/Sdk
export PATH=$PATH:${ANDROID_SDK_HOME}/tools
export PATH=$PATH:${ANDROID_SDK_HOME}/build-tools/30.0.3
export PATH=$PATH:${ANDROID_SDK_HOME}/platform-tools

```

加载配置文件

```
source /etc/profile

```

运行 dx 命令, 如果出现以下信息说明成功

```
fj@ubuntu:~/work/android-ndk-r21d$ dx
error: no command specified
usage:
  dx --dex [--debug] [--verbose] [--positions=] [--no-locals]
  [--no-optimize] [--statistics] [--[no-]optimize-list=] [--no-strict]
  [--keep-classes] [--output=] [--dump-to=] [--dump-width=]
  [--dump-method=[*]] [--verbose-dump] [--no-files] [--core-library]
  [--num-threads=] [--incremental] [--force-jumbo] [--no-warning]
  [--multi-dex [--main-dex-list= [--minimal-main-dex]]
  [--input-list=] [--min-sdk-version=]
  [--allow-all-interface-method-invokes] 
```

unzip android-ndk-r21d-linux-x86_64.zip cd android-ndk-r21d

添加 / etc/profile 配置

```
#set NDK env
export NDK_HOME=/home/fj/work/android-ndk-r21d
export PATH=$NDK_HOME:$PATH

```

加载配置文件

```
source /etc/profile

```

查看 ndk 版本, 如果出现以下信息说明成功

```
fj@ubuntu:~/work/android-ndk-r21d$ ndk-build --v
GNU Make 4.2.1
Built for x86_64-pc-linux-gnu
Copyright (C) 1988-2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. 
```

 

#编译安装 Nodejs  
这里我使用源码安装，使用 apt 安装的话版本过老。不支持 frida 的编译，并且升级麻烦。

```
$ sudo git clone https://github.com/nodejs/node.git
Cloning into 'node'...

```

修改目录权限：

```
$ sudo chmod -R 755 node

```

使用 ./configure 创建编译文件，并按照：

```
$ cd node
$ sudo ./configure
$ sudo make
$ sudo make install

```

查看 node 和 npm 版本：

```
fj@ubuntu:~/work/node$ npm -v
7.4.2
fj@ubuntu:~/work/node$ node -v
v16.0.0-pre

```

 

#编译运行 frida-java  
使用 Git 下载 frida-java-bridge 源码，这里我选择 3.9.4，因为我的 frida 用的就是这个版本

```
fj@ubuntu:~/work$ git clone https://github.com/frida/frida-java-bridge.git --tag 3.9.4

```

frida-java 对应的版本在 test/Makefile 下可以看到

```
frida_version := 12.11.14

```

**修改配置文件**

```
cd frida-java-bridge/
 
sudo gedit test/config.mk

```

修改为如下内容

```
#修改为自己的sdk目录
ANDROID_SDK_ROOT ?= $(shell echo /home/fj/Android/Sdk)
#修改为自己的ndk目录
ANDROID_NDK_ROOT ?= /home/fj/work/android-ndk-r21d
ANDROID_ARCH ?= arm64
ANDROID_ABI ?= arm64-v8a
#因为我们前面在As中下载了29版本，所以可以不用更换
ANDROID_API_LEVEL ?= 29
ANDROID_BINDIR ?= /system/bin
ANDROID_LIBDIR ?= /system/lib64
APEX_LIBDIRS ?= /apex/com.android.runtime/$(shell basename $(ANDROID_LIBDIR)):/apex/com.android.art/$(shell basename $(ANDROID_LIBDIR))
DEBUG_PORT ?= 5042

```

一般情况修改上面我注释的部分就行了，如果你的机型的架构不一样注意修改一下。

 

**进行编译**

```
fj@ubuntu:~/work/frida-java-bridge$ make check
npm install
 
added 234 packages, and audited 235 packages in 25s
 
found 0 vulnerabilities
make -C test deploy
make[1]: Entering directory '/home/fj/work/frida-java-bridge/test'
curl -Ls https://github.com/frida/frida/releases/download/12.11.14/frida-gumjs-devkit-12.11.14-android-arm64.tar.xz | tar -xJf - -C build/obj/local/arm64-v8a
/home/fj/work/android-ndk-r21d/ndk-build \
    NDK_PROJECT_PATH=$(pwd) \
    NDK_APPLICATION_MK=$(pwd)/Application.mk \
    NDK_OUT=$(pwd)/build/obj \
    NDK_LIBS_OUT=$(pwd)/build \
    FRIDA_JAVA_TESTS_DATA_DIR=/data/local/tmp/frida-java-bridge-tests \
    FRIDA_JAVA_TESTS_CACHE_DIR=/data/local/tmp/frida-java-bridge-tests/dalvik-cache
make[2]: Entering directory '/home/fj/work/frida-java-bridge/test'
[arm64-v8a] Compile        : artpalette <= artpalette.c
[arm64-v8a] SharedLibrary  : libartpalette.so
[arm64-v8a] Install        : libartpalette.so => build/arm64-v8a/libartpalette.so
[arm64-v8a] Compile        : runner <= runner.c
[arm64-v8a] Compile++      : runner <= dummy.cpp
[arm64-v8a] Executable     : runner
[arm64-v8a] Install        : runner => build/arm64-v8a/runner
make[2]: Leaving directory '/home/fj/work/frida-java-bridge/test'
curl -Ls https://github.com/junit-team/junit4/releases/download/r4.12/junit-4.12.jar > build/junit.jar
curl -Ls https://search.maven.org/remotecontent?filepath=org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar > build/hamcrest.jar
cd build/java/ \
    && jar xf ../junit.jar \
    && jar xf ../hamcrest.jar
javac \
    -cp .:build/java/:/home/fj/Android/Sdk/platforms/android-29/android.jar \
    -bootclasspath /home/fj/Android/Sdk/platforms/android-29/android.jar \
    -source 1.8 \
    -target 1.8 \
    -Xlint:deprecation \
    -Xlint:unchecked \
    re/frida/Script.java re/frida/Eatable.java re/frida/Formatter.java re/frida/MethodTest.java re/frida/EatableWithField.java re/frida/ClassCreationTest.java re/frida/Fruit.java re/frida/PrimitiveArray.java re/frida/TestRunner.java re/frida/ClassRegistryTest.java \
    -d build/java/
jar cfe build/tests.jar re.frida.tests.Runner -C build/java .
dx --dex --output=build/tests.dex build/tests.jar
npm install
 
> frida-java-bridge-bundle@1.0.0 prepare
> npm run build
 
 
> frida-java-bridge-bundle@1.0.0 build
> frida-compile bundle -o build/frida-java-bridge.js -x -c
 
 
added 427 packages, and audited 428 packages in 2m
 
found 0 vulnerabilities
npm run build
 
> frida-java-bridge-bundle@1.0.0 build
> frida-compile bundle -o build/frida-java-bridge.js -x -c
 
adb shell "rm -rf /data/local/tmp/frida-java-bridge-tests && mkdir -p /data/local/tmp/frida-java-bridge-tests"
* daemon not running; starting now at tcp:5037
* daemon started successfully
adb: no devices/emulators found
Makefile:26: recipe for target 'deploy' failed
make[1]: *** [deploy] Error 1
make[1]: Leaving directory '/home/fj/work/frida-java-bridge/test'
Makefile:5: recipe for target 'check' failed
make: *** [check] Error 2

```

如果你看到  
daemon started successfully  
adb: no devices/emulators found  
这些信息说明编译成功了，后面的错误是因为我还有没有连接我的手机。

 

**测试运行**  
现在连接上我的手机，这里我使用无线 adb 的方式连接

```
fj@ubuntu:~/work/frida-java-bridge$ adb connect 192.168.124.2
connected to 192.168.124.2:5555
fj@ubuntu:~/work/frida-java-bridge$ adb devices
List of devices attached
192.168.124.2:5555    device

```

再次编译运行

```
fj@ubuntu:~/work/frida-java-bridge$ make check
make -C test deploy
make[1]: Entering directory '/home/fj/work/frida-java-bridge/test'
adb shell "rm -rf /data/local/tmp/frida-java-bridge-tests && mkdir -p /data/local/tmp/frida-java-bridge-tests"
adb push build/arm64-v8a/runner build/tests.dex build/frida-java-bridge.js build/arm64-v8a/libartpalette.so /data/local/tmp/frida-java-bridge-tests
build/arm64-v8a/runner: 1 file pushed, 0 skipped. 2.0 MB/s (16624056 bytes in 7.970s)
build/tests.dex: 1 file pushed, 0 skipped. 635.5 MB/s (314016 bytes in 0.000s)
build/frida-java-bridge.js: 1 file pushed, 0 skipped. 0.4 MB/s (293597 bytes in 0.764s)
build/arm64-v8a/libartpalette.so: 1 file pushed, 0 skipped. 0.1 MB/s (5760 bytes in 0.040s)
4 files pushed, 0 skipped. 1.6 MB/s (17237429 bytes in 10.091s)
make[1]: Leaving directory '/home/fj/work/frida-java-bridge/test'
make -C test run
make[1]: Entering directory '/home/fj/work/frida-java-bridge/test'
adb shell "LD_LIBRARY_PATH='/apex/com.android.runtime/lib64:/apex/com.android.art/lib64:/data/local/tmp/frida-java-bridge-tests' /data/local/tmp/frida-java-bridge-tests/runner"
JUnit version 4.12
..........................................................
Time: 18.474
 
OK (58 tests)
 
make[1]: Leaving directory '/home/fj/work/frida-java-bridge/test'

```

看到最后面的 OK 就知道成功了，其中有 58 个方法被测试。

[看雪社区年底排行榜，查查你的排名？](https://www.kanxue.com/rank.htm)