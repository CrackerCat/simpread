> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483871&idx=1&sn=ef4971b75f64b56891d372524b02f36d&chksm=ce07509af970d98c35940ff547c4afdf783bc04bde007012bfb80dc8a6e6f0d5a64f1a6d3bf4&scene=126&sessionid=1609675989&key=d15f9fba2045db2074ee58a01f112d6a7d9eef09f46ca2f907747234eb2e55f82f048508a0c697b2afa46d25e4957b8f247b9b8732eb7a3909d601890e1c3f97e6528a615db50c8c54907eb8b8dffbe2184ac532e602fb7a3c54db4d9329080fb2fe0b14870c2d7b9bb1037711dfab7ef4c5fb4b9e259bfa26036281d89ce480&ascene=1&uin=MTEzMjI1NDQxNg%3D%3D&devicetype=Windows+10+x64&version=62090538&lang=zh_CN&exportkey=A5MwOe%2FRi1DwPzMP3oqmCyw%3D&pass_ticket=VdmGoHi12%2F5L2IgPrGbEsnQ4As%2F5lDdJyRfnrfUW5gUrPt%2FXKT9yA9wDV9sLT%2BSp&wx_header=0)

**一、内置方法**

    在 Android 系统中 , 预编译 so 或者可执行程序可以使用 Android.mk 配置模块方式或者使用源码中提供的 RODUCT_COPY_FILES 复制拷贝。

  Android.mk 中预编译脚本可以参考上一章中的说明。

  PRODUCT_COPY_FILES:

  在 Android 源码中可以使用使用 PRODUCT_COPY_FILES 预拷贝文件和目录。以下列举了一个使用的方式:

```
RODUCT_COPY_FILES += \
    vendor/lineage/prebuilt/common/bin/backuptool.sh:install/bin/backuptool.sh \
    vendor/lineage/prebuilt/common/bin/backuptool.functions:install/bin/backuptool.functions \
    vendor/lineage/prebuilt/common/bin/50-lineage.sh:$(TARGET_COPY_OUT_SYSTEM)/addon.d/50-lineage.sh

```

    本文中准备使用 Android.mk 文件 "include $(BUILD_PREBUILT)" 方式内置 frida-server, 使用 PRODUCT_COPY_FILES 内置 frida-gadget  arm 和 arm64 平台动态库到系统中。 

**二、开始内置**  

       2.1 准备素材以及源码存放目录

       分别官网下载 frida-server 可执行程序 (由于我的是 64 位系统，只考虑 arm64) 和 frida-gadget 动态库(arm arm64)。在源码中创建保存文件的路径 framework/base/cmds/mycmds, 并将文件拷贝到该目录。如下所示:

```
qiang@ubuntu:~/lineageOs$ mkdir -p frameworks/base/cmds/mycmds
qiang@ubuntu:~/lineageOs$ cd frameworks/base/cmds/mycmds/
qiang@ubuntu:~/lineageOs/frameworks/base/cmds/mycmds$ ls -la
total 74412
drwxrwxr-x  2 qiang qiang     4096 1月   3 03:03 .
drwxrwxr-x 36 qiang qiang     4096 1月   2 05:59 ..
-rwxrw-rw-  1 qiang qiang 20162208 1月   2 05:56 libmyfridagadgetarm64.so
-rwxrw-rw-  1 qiang qiang 14677128 1月   2 05:56 libmyfridagadgetarm.so
-rwxrw-rw-  1 qiang qiang 41338528 1月   2 05:38 myfridaserverarm64

```

    2.2 内置 frida-gadget 动态库

            在源码中搜索 PRODUCT_COPY_FILES 使用的地方，找一个最好和具体设备无关的使用的地方。此处我选择 build/make/target/product/handheld_system.mk 文件中添加。在该文件中添加如下内容完成 frida-gadget 动态库的复制工作。

```
# ///ADD START
PRODUCT_COPY_FILES += \
    frameworks/base/cmds/mycmds/libmyfridagadgetarm.so:$(TARGET_COPY_OUT_SYSTEM)/lib/libmygadget.so \
    frameworks/base/cmds/mycmds/libmyfridagadgetarm64.so:$(TARGET_COPY_OUT_SYSTEM)/lib64/libmygadget.so
# ///ADD END

```

2.3 内置 frida-server 可执行文件

 在上面的 framework/base/cmds/mycmds 文件夹中，添加 Android.mk 实现 frida-server 的内置工作。Android.mk 内容如下:

```
#///ADD START
#///ADD END
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := myfridaserverarm64
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_SRC_FILES := myfridaserverarm64
include $(BUILD_PREBUILT)

```

            在 frida-server 编译模块 Android.mk 文件创建好之后，将 myfridaserverarm64 模块加入 build/make/target/product/base_system.mk 中的 PRODUCT_PACKAGES 

编译文件链中。加入之后 PRODUCT_PACKAGES 如下:

```
#///ADD START
# add frida server to system
#///ADD END
# Base modules and settings for the system partition.
PRODUCT_PACKAGES += \
    myfridaserverarm64 \
    abb \
    adbd \
    am \
    ...(此处省略)

```

**三、编译刷机测试**   

```
source build/envsetup.sh
breakfast oneplus3
brunch oneplus3

```

关注公众号, 及时获取更新内容![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431OFuEQDfZ2gLpl60bK0KvRNyVWZLOgVueo2AXqTKPKCjXicIWP37spOUsjicf9MqD0gmlQj1Fcv6rw/640?wx_fmt=png):

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)