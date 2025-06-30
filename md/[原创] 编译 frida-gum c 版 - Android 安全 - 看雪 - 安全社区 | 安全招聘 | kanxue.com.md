> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287401.htm)

> [原创] 编译 frida-gum c 版

搞逆向一直没有一个特别顺手的工具，官方的 stalker 经常崩溃，qbdi 需要主动调用，所以就想要自己编译 gum，将报错的地方手动处理掉

刚开始研究编译总会遇到各种各样的问题，被折腾半个月，睡觉做梦都在编译 gum![](https://bbs.kanxue.com/view/img/face/017.gif)

用官方的编译命令`./configure --host=android-arm64 -- -Dfrida-gum:devkits=gum`

我环境编译出来确实跟官方编译好的结构相同，但是某些 api 会报错，折腾半个月，准备打算放弃，这种突然又觉得不甘心，最终也是解决了，

目前可以稳定完整 trace，使用的内存映射，速度也是蛮快，1800w 行 20s 就可以完成，后面看情况开源吧

我使用的是 Android studio，因为 cmake 交叉编译并不熟悉，而且自我感觉 Android studio 方便一点，可以动态调试，实时处理 bug，在使用 stalker 处理日志时，动态调试倒是帮了大忙

准备：
===

Ubuntu（我的是 20.04）

git

ndk（我的是 ndk-r25c）

node npm

Android Studio

开始：
===

观看顺序是先文字，下面是配图

将 frida 拉到本地

```
git clone https://github.com/frida/frida.git
cd frida
git checkout 16.2.1 // frida-server比较习惯用这个版本，所以也是编译的这个版本
```

![](https://bbs.kanxue.com/upload/attach/202506/994069_DYAHQQ8GWPWRXVN.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_TTXW5NJ96BQAREP.jpg)

之后直接执行 `make`

会罗列出所有可以编译的版本，找到想要编译的版本

![](https://bbs.kanxue.com/upload/attach/202506/994069_6UG7XZF93MN4VHB.jpg)

再次执行 `make gum-android-arm64`

![](https://bbs.kanxue.com/upload/attach/202506/994069_DR95CJ5SNQA7GUM.jpg)

然后报错停止了

![](https://bbs.kanxue.com/upload/attach/202506/994069_268YS3JPPTBY3C4.jpg)

是因为没有找到 AndroidNDK，手动配置一下

下载官方 ndk [https://developer.android.com/ndk/downloads](ndk版本)

添加全局变量编译~/bashrc 的文件

添加 ndk 路径

```
export ANDROID_NDK_ROOT=/home/zheng/lib/android-ndk-r25c
export PATH=$ANDROID_NDK_ROOT:$PATH
```

刷新配置 `source ~/.bashrc`

重新执行编译

![](https://bbs.kanxue.com/upload/attach/202506/994069_XMJZWAREGDMA5M2.jpg)

再次遇到问题，缺少 npm 环境

![](https://bbs.kanxue.com/upload/attach/202506/994069_MFSBT9UYNHAVCK4.jpg)

安装一下 sudo apt install -y nodejs npm

![](https://bbs.kanxue.com/upload/attach/202506/994069_CMXQGRH8NGTMGN2.jpg)

再次编译 `make gum-android-arm64`

![](https://bbs.kanxue.com/upload/attach/202506/994069_MRGRQ9FVEMRYMU7.jpg)

这个编译会将 gum 相关的都编译出来，编译之后的静态链在 `frida/build/frida-android-arm64/lib`中

![](https://bbs.kanxue.com/upload/attach/202506/994069_SZ6MNDDGDCBCYKG.jpg)

引入 Android Studio
=================

将这个`libfrida-gum.a`还有`inlude`文件，复制到 Android studio 中，(这里我方便管理，塞到了自定义的 frida 文件中，如图三

![](https://bbs.kanxue.com/upload/attach/202506/994069_KX9UVB36KNBW3A5.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_3JMBZJQ7N5GB94K.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_M399EMCJ5RNFBNA.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_8C263B6B5X7XEHV.jpg)

在`CMakeLIsts.txt`中配置，在实际引用的时候，会有很多报错，我是一个一个补`.h`和`.a`最终编译成功

CMakeLists 参考
=============

最终我的完整 `CMakeLists.txt`文件

```
cmake_minimum_required(VERSION 3.22.1)

project("native-lib")

set(GUM_LIB_PATH ${CMAKE_SOURCE_DIR}/../lib/${ANDROID_ABI})

include_directories( # 添加frida依赖
        ${CMAKE_SOURCE_DIR}/frida
        ${CMAKE_SOURCE_DIR}/frida/capstone
        ${CMAKE_SOURCE_DIR}/frida/glib-2.0
        ${CMAKE_SOURCE_DIR}/frida/glib-2.0/include
        ${CMAKE_SOURCE_DIR}/frida/include
        ${CMAKE_SOURCE_DIR}/frida/lib/glib-2.0/include
)


include_directories( # 添加自定义依赖
        ${CMAKE_SOURCE_DIR}/include
)


include_directories( # 添加本体app依赖
        ${CMAKE_SOURCE_DIR}/libt
)

add_library(${CMAKE_PROJECT_NAME} SHARED
        native-lib.cpp
        libt/md5.cpp
)


target_link_libraries(${CMAKE_PROJECT_NAME}
        android
        log)

add_library(
        trace SHARED
        gum.cpp
        lib/frida_stalker.cpp
        lib/frida_attach.cpp
        lib/raw_logger.cpp
)

target_link_libraries(trace
        ${GUM_LIB_PATH}/libfrida-gum.a
        ${GUM_LIB_PATH}/libcapstone.a
        ${GUM_LIB_PATH}/libglib.a
        ${GUM_LIB_PATH}/libpcre.a
        ${GUM_LIB_PATH}/libgobject.a
        ${GUM_LIB_PATH}/libffi.a
        ${GUM_LIB_PATH}/libiconv.a
        ${GUM_LIB_PATH}/libminizip.a
        ${GUM_LIB_PATH}/libz.a
        android
        log
)
```

文件树

```
 main
├──  cpp
│   ├──  frida
│   │   ├──  capstone
│   │   ├──  glib-2.0
│   │   │   ├──  gio
│   │   │   ├──  glib
│   │   │   │   ├──  deprecated
│   │   │   ├──  gmodule
│   │   │   ├──  gobject
│   │   ├──  gum
│   │   │   ├──  arch-arm
│   │   │   ├──  arch-arm64
│   │   │   ├──  arch-mips
│   │   │   ├──  arch-x86
│   │   │   ├──  heap
│   │   │   ├──  prof
│   │   ├──  gumjs
│   │   ├──  gumpp
│   │   ├──  lib
│   │   │   ├──  glib-2.0
│   │   │   │   ├──  include
│   ├──  include
│   ├──  lib
│   ├──  libt
├──  java
│   ├──  com
├──  lib
│   ├──  arm64-v8a
├──  res
│   ├──  drawable
│   ├──  layout
│   ├──  mipmap-anydpi-v26
│   ├──  mipmap-hdpi
│   ├──  mipmap-mdpi
│   ├──  mipmap-xhdpi
│   ├──  mipmap-xxhdpi
│   ├──  mipmap-xxxhdpi
│   ├──  values
│   ├──  values-night
│   ├──  xml
```

![](https://bbs.kanxue.com/upload/attach/202506/994069_CJZJCPQ44AEAAKK.jpg)

[[培训] 科锐逆向工程师培训第 53 期 2025 年 7 月 8 日开班！](https://bbs.kanxue.com/thread-51839.htm)

最后于 14 小时前 被郑 z 编辑 ，原因：

[#HOOK 注入](forum-161-1-125.htm) [#源码框架](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)