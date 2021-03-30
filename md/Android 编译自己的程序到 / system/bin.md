> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/10052234.html)

背景[#](#背景)
----------

有时候我们想创建一个程序，放在系统中，供其他 APP 执行。我们知道，在生成 system.img 的时候，编译系统会将 out/target/product/[product]/system/bin 目录打包进去。所以，我们想办法让编译系统在编译的过程中，把我们的程序编译了，并把编译生成的二进制文件自动放到 out/target/product/[product]/system/bin。

做法[#](#做法)
----------

假如我们要创建一个 mytest 的程序  
1. 在 external 目录下创建名为 mytest 的文件夹，这个文件夹用于存放我们的程序代码和 makefile 文件以及这个程序的依赖库。  
2. 创建我们的程序代码和 makefile 文件，这里主要看看 makefile 文件怎么写，例子如下：

```
LOCAL_PATH := $(call my-dir)

common_cflags := \
    -std=c99 \
    -Os \
    -Wall \
    -Wextra \
    -Wno-char-subscripts \
    -Wno-sign-compare \
    -Wno-string-plus-int \
    -Wno-uninitialized \
    -Wno-unused-parameter \
    -funsigned-char \
    -ffunction-sections -fdata-sections \
    -fno-asynchronous-unwind-tables \

# static executable for use in limited environments
include $(CLEAR_VARS)
LOCAL_SRC_FILES := mytest.c
LOCAL_CFLAGS := $(common_cflags)
LOCAL_CXX_STL := none
LOCAL_CLANG := true
# LOCAL_MODULE_PATH and LOCAL_UNSTRIPPED_PATH do not equal
LOCAL_UNSTRIPPED_PATH := $(PRODUCT_OUT)/symbols/utilities
LOCAL_MODULE := mytest_static
LOCAL_MODULE_CLASS := UTILITY_EXECUTABLES
LOCAL_MODULE_PATH := $(PRODUCT_OUT)/system/bin
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_STEM := mytest
LOCAL_PACK_MODULE_RELOCATIONS := false
LOCAL_STATIC_LIBRARIES := libc libcurl libz libcrypto_static
LOCAL_SHARED_LIBRARIES := libssl 
LOCAL_FORCE_STATIC_EXECUTABLE := true
include $(BUILD_EXECUTABLE)


```

上面的代码中，值得关注的几个内容：

```
LOCAL_SRC_FILES:要编译的源文件
LOCAL_CFLAGS:编译时所要使用的编译选项
LOCAL_UNSTRIPPED_PATH:生成的二进制文件临时存放目录。$(PRODUCT_OUT)是产品生成目录，等于上文提到的out/target/product/[product]
LOCAL_MODULE:模块的名称。这里我们生成的是静态可执行文件，所以后面加上'_static'
LOCAL_MODULE_PATH:模块生成后的存放的目录。$(PRODUCT_OUT)是产品生成目录，等于上文提到的out/target/product/[product]
LOCAL_MODULE_TAGS:设置模块标签，这影响在什么版本下编译该模块。
LOCAL_STATIC_LIBRARIES:依赖的静态库
LOCAL_SHARED_LIBRARIES:依赖的动态库
LOCAL_FORCE_STATIC_EXECUTABLE:是否强制生成静态可执行文件
include $(BUILD_EXECUTABLE):生成可执行文件


```

注意：创建可执行文件，源代码要有 main 函数

3. 让编译系统运行后编译我们的模块  
(1) 在`/build/target/product`目录下，创建一个 makefile 文件，这里就取为`mytest.mk`吧  
(2) 在里面输入：

```
PRODUCT_PACKAGES := mytest_static


```

PRODUCT_PACKAGES 是编译系统定义的变量，在后面写上我们模块名，这样我们的模块就会被编译。  
(3) 让编译系统找到我们的 mytest.mk, 在`build/target/product/core.mk`文件末尾追加一行：  
`$(call inherit-product, $(SRC_TARGET_DIR)/product/mytest.mk)`

测试模块[#](#测试模块)
--------------

生成我们自己的模块后，有两种方式测试。  
**1. 只编译模块，然后把它 push 到我们的设备。这种方式是最方便的。在源码目录下执行下面两个命令:**  
(1) 编译模块

```
mmm ./external/mytest


```

(2) 将模块 push 到手机

```
adb push $OUT/system/bin/mytest /system/bin


```

注：如果提示`couldn't create file: Read-only file system`，先执行`adb remount`  
(3) 执行我们的编译好的模块

```
adb shell mytest


```

**2. 编译整个 system.img, 模块会打包进镜像中**  
(1) 编译 system.img

```
make systemimage


```

(2) 刷入 system.img

```
fastboot flash system $OUT/system.img


```

(3) 执行我们的编译好的模块

```
adb shell mytest


```

[![](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**