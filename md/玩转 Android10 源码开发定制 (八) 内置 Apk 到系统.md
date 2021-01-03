> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483855&idx=1&sn=06138a9db04ba2a761f8dd9495ecd56a&chksm=ce07508af970d99c1ab1718ea55e5e668029ab68ff438cef2076349faccdb5fbffe14c97cfa3&key=5bb81a5ac4a09a80d9ec56cacafffa23d887c99d11f027e4e387597414021c5f61410734ed9ee97f15c66c08d29bb823cfa6a4bed412d3ca4502965dd8e604b76a7997980c4fa12d151635c4b20861775f25cb35adedb900b963c1b3320e85abdfa4eb85dcd4c95f0c2347ce019957aededcceded6615e7920077a3f84b67e95&ascene=1&uin=MTEzMjI1NDQxNg%3D%3D&devicetype=Windows+10+x64&version=62090538&lang=zh_CN&exportkey=A%2BOn43lKnqv3MQuznO1OqwM%3D&pass_ticket=VdmGoHi12%2F5L2IgPrGbEsnQ4As%2F5lDdJyRfnrfUW5gUrPt%2FXKT9yA9wDV9sLT%2BSp&wx_header=0)

**1.Android.mk 文件说明**

    Android.mk 是 Android 提供的一个 makefile 文件，可以将源文件分组为模块。用来引用的头文件目录、需要编译的 *.c/*.cpp 文件、jni 源文件、指定编译生成 *.so 共享库文件或者 *.a 静态库文件，可以定义一个或多个模块，也可以多个模块中使用同一个源文件。

      为方便模块编译，编译系统设置了很多模块描述环境变量和宏定义，如下列举一些常用的。

   _模块描述环境变量:_

*   LOCAL_SRC_FILES：
    
               当前模块包含的源文件；
    
*   LOCAL_MODULE：
    
                 当前模块的名称；
    
*   LOCAL_PACKAGE_NAME
    
                 当前 APK 应用的名称；
    
*   LOCAL_C_INCLUDES：
    
                   C/C++ 所需的头文件路径;
    
*   LOCAL_STATIC_LIBRARIES：
    
                   当前模块在静态链接时需要的静态链接库名;
    
*   LOCAL_SHARED_LIBRARIES：
    
                   当前模块在运行时依赖的动态链接库名;
    
*   LOCAL_STATIC_JAVA_LIBRARIES：
    
                    当前模块依赖的 Java 静态库;
    
*   LOCAL_JAVA_LIBRARIES：
    
                   当前模块依赖的 Java 共享库;
    
*   LOCAL_CERTIFICATE：
    
                   当前应用的签名方式，比如使用系统签名 platform。
    
*   LOCAL_MODULE_TAGS：
    
           表示该模块编译的标识，可能值为 eng,user,tests 或 optional（默认值）。值具体表示:
    
               user: 指该模块只在 user 版本下才编译 
    
              eng: 指该模块只在 eng 版本下才编译
    
              tests: 指该模块只在 tests 版本下才编译
    
              optional: 指该模块在所有版本下都编译
    

  定义的 include 变量:
-----------------

*      include $(BUILD_SHARED_LIBRARY) 
    

           表示需要编译为 so 动态链接库。  

*      include $(BUILD_JAVA_LIBRARY) 
    

           表示需要编译为 java 动态库。

*      include $(BUILD_JAVA_STATIC_LIBRARY) 
    

           表示需要编译为 java 静态库。

*      include $(BUILD_STATIC_LIBRARY)
    
         表示需要编译为 so 静态库。 
    
*   include $(BUILD_EXECUTABLE)
    
         表示编译为 Native C 可执行程序  
    
*   include $(BUILD_PREBUILT)  
    

             表示源文件已经是编译好的，只需要进行复制操作

     定义的函数宏，原型为 `$(call <function>)`，以下列举几个:  

*   $(call my-dir)
    
              获取当前文件夹路径；
    
*   $(call all-java-files-under,)
    
           获取指定目录下的所有 Java 文件；
    
*   $(call all-c-files-under,)
    
            获取指定目录下的所有 C 文件；
    
*   $(call all-Iaidl-files-under,)
    
            获取指定目录下的所有 AIDL 文件；
    

 **2. 提供几个 Android.mk 模板**  

```
#编译so静态库    
LOCAL_PATH := $(call my-dir)    
include $(CLEAR_VARS)    
LOCAL_MODULE = libtest 
LOCAL_SRC_FILES = testxxx.c    
LOCAL_C_INCLUDES = $(INCLUDES)    
LOCAL_SHARED_LIBRARIES := libcutils      
include $(BUILD_STATIC_LIBRARY)    
#编译为so动态库链接库    
LOCAL_PATH := $(call my-dir)    
include $(CLEAR_VARS)    
LOCAL_MODULE = libtest  
LOCAL_SRC_FILES = testxx.c    
LOCAL_C_INCLUDES = $(INCLUDES)    
LOCAL_SHARED_LIBRARIES := libcutils       
include $(BUILD_SHARED_LIBRARY)    
 #编译java动态库
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := testjar
include $(BUILD_JAVA_LIBRARY)
#编译java静态库
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := testjar
include $(BUILD_STATIC_JAVA_LIBRARY)   
#编译为可执行文件  
LOCAL_PATH := $(call my-dir)    
include $(CLEAR_VARS)    
LOCAL_MODULE := hellos    
LOCAL_STATIC_LIBRARIES := libhellos    
LOCAL_SHARED_LIBRARIES :=    
LOCAL_LDLIBS += -ldl    
LOCAL_CFLAGS := $(L_CFLAGS)    
LOCAL_SRC_FILES := mains.c    
LOCAL_C_INCLUDES := $(INCLUDES)    
include $(BUILD_EXECUTABLE)    
#使用动态库    
LOCAL_PATH := $(call my-dir)    
include $(CLEAR_VARS)    
LOCAL_MODULE := hellod    
LOCAL_MODULE_TAGS := debug    
LOCAL_SHARED_LIBRARIES := libc libcutils libhellod    
LOCAL_LDLIBS += -ldl    
LOCAL_CFLAGS := $(L_CFLAGS)    
LOCAL_SRC_FILES := maind.c    
LOCAL_C_INCLUDES := $(INCLUDES)    
include $(BUILD_EXECUTABLE)    
#预编so动态链接库到指定目录
LOCAL_PATH := $(call my-dir)  
include $(CLEAR_VARS)  
LOCAL_MODULE := libtest.so  
LOCAL_MODULE_CLASS := SHARED_LIBRARIES  
LOCAL_MODULE_PATH := $(ANDROID_OUT_SHARED_LIBRARIES)  
LOCAL_SRC_FILES := lib/$(LOCAL_MODULE )  
OVERRIDE_BUILD_MODULE_PATH := $(TARGET_OUT_INTERMEDIATE_LIBRARIES)  
include $(BUILD_PREBUILT)  
#预编二进制文件到系统
include $(CLEAR_VARS)
LOCAL_MODULE := testbin
LOCAL_SRC_FILES := testbin
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_MODULE_TAGS := optional
include $(BUILD_PREBUILT)
# 内置Apk文件到系统
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := adbinputmethod
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_MODULE_TAGS := optional
LOCAL_PRIVILEGED_MODULE := true
LOCAL_BUILT_MODULE_STEM := package.apk
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
# LOCAL_CERTIFICATE := platform
LOCAL_CERTIFICATE := PRESIGNED
include $(BUILD_PREBUILT)

```

**3. 源码中内置 apk 到系统**

      此处以内置搜狗输入法到系统为例说明。  

    (1). 创建存放内置 App 的目录，此处我创建为 packages/myapps/sougou, 如下所示:

```
qiang@ubuntu:~/lineageOs$ pwd
/home/qiang/lineageOs
qiang@ubuntu:~/lineageOs$ mkdir -p packages/myapps/sougou
qiang@ubuntu:~/lineageOs$ 

```

       (2). 将搜狗输入法 apk 复制到 packages/myapps/sougou, 并创建内置 apk 到系统的 Android.mk。如下所示：

```
qiang@ubuntu:~/lineageOs/packages/myapps/sougou$ pwd
/home/qiang/lineageOs/packages/myapps/sougou
qiang@ubuntu:~/lineageOs/packages/myapps/sougou$ ls -la
total 58480
drwxrwxrwx 2 qiang qiang     4096 1月   1 18:22 .
drwxrwxr-x 3 qiang qiang     4096 1月   2 02:25 ..
-rwxrw-rw- 1 qiang qiang      755 1月   1 18:29 Android.mk
-rwxrw-rw- 1 qiang qiang 59870272 11月  3 07:35 sougou.apk

```

     对应的 Android.mk 内容为:  

```
# ///ADD START
# ///ADD END
# 设置当前工作路径
LOCAL_PATH:= $(call my-dir)
# 清除变量值
include $(CLEAR_VARS)
# 生成的模块名称
LOCAL_MODULE := sougou
# 生成的模块类型
LOCAL_MODULE_CLASS := APPS
# 生成的模块后缀名,此处为apk
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
# 设置模块tag，tags取值可以为:user debug eng tests optional
# optional表示全平台编译
LOCAL_MODULE_TAGS := optional
# LOCAL_PRIVILEGED_MODULE := true
LOCAL_BUILT_MODULE_STEM := package.apk
# 设置源文件
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
# LOCAL_CERTIFICATE := platform
# 设置签名，此处表示保持apk原有签名
LOCAL_CERTIFICATE := PRESIGNED
# 此处表示预编译方式
include $(BUILD_PREBUILT)

```

    (3) 将创建的搜狗输入法编译模块配置到编译系统，这个非常重要, 如果不加入编译镜像默认是不会编译刚添加的模块。lineageOs 中，可以在如下路径中添加 Settings 模块的地方追加 sougou。  

文件位置:

```
build/make/target/product/handheld_product.mk

```

        添加 sougou 内容之后的部分内容如下:  

```
PRODUCT_PACKAGES += \
    Browser2 \
    Calendar \
    Camera2 \
    Contacts \
    DeskClock \
    Gallery2 \
    Launcher3QuickStep \
    Music \
    OneTimeInitializer \
    Provision \
    QuickSearchBox \
    Settings \
    SettingsIntelligence \
    StorageManager \
    SystemUI \
    WallpaperCropper \
    frameworks-base-overlays \
    sougou

```

   （4）编译刷机  

```
qiang@ubuntu:~/lineageOs/build/make/target/product$ croot
qiang@ubuntu:~/lineageOs$ breakfast oneplus3
Looking for dependencies in device/oneplus/oneplus3
Looking for dependencies in device/oppo/common
device/oppo/common has no additional dependencies.
Looking for dependencies in kernel/oneplus/msm8996
kernel/oneplus/msm8996 has no additional dependencies.
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=10
LINEAGE_VERSION=17.1-20210102-UNOFFICIAL-oneplus3
TARGET_PRODUCT=lineage_oneplus3
TARGET_BUILD_VARIANT=userdebug
TARGET_BUILD_TYPE=release
TARGET_ARCH=arm64
TARGET_ARCH_VARIANT=armv8-a
TARGET_CPU_VARIANT=kryo
TARGET_2ND_ARCH=arm
TARGET_2ND_ARCH_VARIANT=armv8-a
TARGET_2ND_CPU_VARIANT=kryo
HOST_ARCH=x86_64
HOST_2ND_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-5.4.0-58-generic-x86_64-Ubuntu-20.04.1-LTS
HOST_CROSS_OS=windows
HOST_CROSS_ARCH=x86
HOST_CROSS_2ND_ARCH=x86_64
HOST_BUILD_TYPE=release
BUILD_ID=QQ3A.200805.001
OUT_DIR=out
PRODUCT_SOONG_NAMESPACES=vendor/oneplus/oneplus3 device/oneplus/oneplus3 vendor/nxp/opensource/pn5xx hardware/qcom-caf/msm8996
============================================
qiang@ubuntu:~/lineageOs$ brunch oneplus3
Looking for dependencies in device/oneplus/oneplus3
Looking for dependencies in device/oppo/common
device/oppo/common has no additional dependencies.
Looking for dependencies in kernel/oneplus/msm8996
kernel/oneplus/msm8996 has no additional dependencies.

```

**4. 效果展示**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432yTn30yvB9N4wcnG2FPSz0fh4sh4KKT36RFqgfR4Exed1p1bicuy4JRkum0WTYbGrZrWMZXTVvfdQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432yTn30yvB9N4wcnG2FPSz0JN7gaVSs67OibDdZEhfWUSWBibQY3ZCDYkstCoe6ZUcj5KMw3C9MpRmg/640?wx_fmt=png)

关注公众号, 及时获取更新内容:  

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)