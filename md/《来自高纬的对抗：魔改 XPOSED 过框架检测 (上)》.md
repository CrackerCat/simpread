> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/c97zoTxRrEeYLvD8YwIUVQ)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bTzEZUL1XMbcIm64P2hjKh0nnr3iaKI6GQyXf5mry8bNXoMWmlcYrSicQ/640?wx_fmt=png)

市面上大多数加固厂商、或者大型`App`都会或多或少对`Xposed`框架执行基于特征的检测，而突破这些检测的基本思路就是找到检测的地方，不管是在`Java`层还是`Native`层，然后通过`hook`的方式修改返回结果，或者硬编码、直接置零返回通过等方式来通过校验。

但是，且不论直接修改二进制能不能通过完整性校验，大多数工程师或许连在哪里进行的校验都很难找到，如果加固厂商再使用`Ollvm`这样的混淆工具来一轮混淆，那几乎是欲哭无泪，无力回天。

其实，跟加固厂商在特征检测层面斗智斗勇，是很不明智的选择，敌在暗、我在明，寻找检测宛如大海捞针；不如从源头消灭特征，任你万般检测，我自笑傲江湖。

所以我们回到正题，本篇首先会介绍官方原版`XPOSED`的编译流程，刷入手机、安装插件、开发插件都能正常使用；然后在此基础之上，进行源码的一些魔改，修改之后再编译刷入手机，最后讲根据新源码编译出来的`API`进行自己的插件开发流程，实现从源码的**高维**来`bypass`**低维**框架检测的目的。

在前文中，我们学习了如何编译`aosp6.0.0r1`，从安卓源码编译系统是编译`XPOSED`框架的基础。本篇编译的对象是`aosp7.1.2_r8`，编译的流程与`aosp6.0.0r1`完全相同，唯一的区别就是在安装`openjdk`的时候，安装的是版本`8`，仅此而已；`编译aosp7.1.2r8`的过程不再赘述。`aosp7.1.2_r8`的源码是与`aosp6.0.0_r1`在一起的，都在`Github`主页：下方的百度云盘中的`aosp_pure_source_code`目录中。

在阅读本文之前，还有一篇《XPOSED 魔改一：获取特征》需要先看下，介绍了很多前置知识，比如为什么要选择`aosp7.1.2r8`这个版本，因为这个版本是`XPOSED`正式版支持到的最后一个版本，并且也是其开源的最后的一个版本。还有诸多`XPOSED`的检测点和检测原理和代码，也进行了详细的介绍，理解这些内容，对理解后续操作有很大的帮助。

本文所涉及的环境、实验数据结果及代码资料等都在我的`Github`：https://github.com/r0ysue/AndroidSecurityStudy 上，欢迎大家取用和`star`，蟹蟹。

接下来撸起袖子开干。

### 编译官方原版`XPOSED`

首先来看下源码的基本结构和框架。为了防止`Xposed`框架万一更新，笔者也将官网上的五大项目的源码直接各`Download Zip`了一份，见本项目附件文件夹。

#### `Xposed`源码框架结构

官方目录共有五个项目：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bWxkIiafNe1r7IsaQiayl6yePVcQrm7WIjZ17shKm0xuicOh8MLoaicJIUQ/640?wx_fmt=png)

按照前文的内容整理，可以理清楚其分别对应的功能：

<table><tbody><tr><td width="137" valign="top">模块</td><td width="398" valign="top">功能</td></tr><tr><td width="137" valign="top">XposedInstaller</td><td width="398" valign="top">下载安装<code>Xposed.zip</code>刷机包、下载安装和管理模块</td></tr><tr><td width="137" valign="top">XposedBridge</td><td width="398" valign="top">位于<code>Java</code>层的<code>API</code>提供者，模块调用功能时首先就是到这里，然后再 “转发” 到<code>Native</code>方法</td></tr><tr><td width="137" valign="top">Xposed</td><td width="398" valign="top">位于<code>Native</code>层的<code>Xposed</code>实际实现，实现方法替换的实际逻辑等功能，主要是在<code>app_process</code>上进行的二次开发</td></tr><tr><td width="137" valign="top">android_art</td><td width="398" valign="top">在原版<code>art</code>上进行的二次开发，目录及文件基本上与原版<code>art</code>相同，稍加修改提供对<code>Xposed</code>的支持</td></tr><tr><td width="137" valign="top">XposedTools</td><td width="398" valign="top"><code>XposedInstaller</code>下载的那个刷机<code>zip</code>包，就是用<code>XposedTools</code>编译打包出来的。</td></tr></tbody></table>

所以我们在安装`Xposed`框架时的主要逻辑就是，`XposedInstaller`会下载由`XposedTools`打包的含有`XposedBridge`、`Xposed`和`android_art`文件的`zip`包，并将其刷入，所谓 “刷入” 实际执行的操作就是安放和替换相应的系统文件。

接下来分别对这些模块进行编译。

#### 编译模块管理器 (XposedInstaller)

先下载官网源代码：

```
# git clone https://github.com/rovo89/XposedInstaller.git


```

直接用`Android Studio`打开，报`ERROR: Failed to find target with hash string 'android-27' in: /root/Android/Sdk`错误。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bibHpWmIicbNpBx8eMGz93Xwzq4wRNhLgO83UwvhxpfxBRZpAa4hQ1Qibg/640?wx_fmt=jpeg)

点击`Install missing platform(s) and sync project`，来配置下载一下`android-27`这个`sdk`。选择`Accept`→`Next`，就会开始自动开始下载安装。

然后就会出现新的错误：

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bZvuLDF4Bnma5TlkNIwUaenBqN6CI3bQgXqhzvKd6Sxm1HPwMNsPHyw/640?wx_fmt=jpeg)

不断点击、安装即可。

等它所有的包下载完成，就可以编译成功，并且安装到手机上了。

> 如果还有网络相关错误发生，只要关闭项目，再打开项目即可。

同样在安装了`n2g47o`同版本的镜像的手机中，刷入`SuperSU`来`root`，然后连接`Android Studio`来安装新编译出来的`XposedInstaller`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bx8M3LHjibfq9jJuux7a66Ra8ffVNreMjBqlfdLBOcA8EUKJG7Jj0R8g/640?wx_fmt=png)

#### 编译运行时支持库 (XposedBridge)

依旧先下载项目源码：

```
# git clone https://github.com/rovo89/XposedBridge.git


```

然后用`Android Studio`打开，会发现准备环境就要花费很长时间，可以看个流量监测，有很多以`kb`级速度下载的项目。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bCKGNZ3uYmyGoPaLuQKzxeWicKrKLvCoSXvNMZrlblah1tw95Wnf5oiaw/640?wx_fmt=jpeg)

此处应该有耐心，感谢党和国家，毕竟还给留了几`kb`，耐心等待下载完毕。

如果遇到这种红字错误，那就关闭项目，重开，

```
A problem occurred configuring root project 'XposedBridge'.
> Could not resolve all dependencies for configuration ':classpath'.
   > Could not download guava.jar (com.google.guava:guava:18.0)
      > Could not get resource 'https://jcenter.bintray.com/com/google/guava/guava/18.0/guava-18.0.jar'.
         > Connection reset


```

如果出现如下所示的缺少`sdk`源码的错误：

```
ERROR: assert sdkSources.exists()
       |          |
       |          false
       /root/Android/Sdk/sources/android-23


```

`File`→`Settings`→`Android SDK`→勾选`Android 6.0(Marshmallow)`→`Apply`→`OK`，就会开始下载`Sources for Android 23`。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bAO0oNc5NWNUCia0Ba8YuKe0VeHpVTmRCR0n5M8icumpfoY4vhpKwdTFA/640?wx_fmt=jpeg)

全部完成之后，再关闭项目，重新打开项目。又出现缺少`Build Tools`：

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bbnuU0l6atbOFs9vItokQ2Vu6icMOPaibibfCJgT1Ek6IDV5aXjPtg83DA/640?wx_fmt=jpeg)

再点击`Install Build Tools 23.0.3 and sync project`，安排！

然后貌似就同步成功了。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9br0dlNcwl10EQ1mx2tWTicqJvj8BtznGIepEtCTIVB56Q3p8nyzLNSkQ/640?wx_fmt=jpeg)

直接编译，`Build`→`Make Project`会出现如下错误：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bahEGOZFjOzc3IQbMEb0QqnlF8qGtBFj6hPeBfGrlFv1JYDJ6SVKibAA/640?wx_fmt=png)

```
/root/Android/Sdk/build-tools/23.0.3/aapt: error while loading shared libraries: libz.so.1: cannot open shared object file: No such file or directory


```

这时候需要安装`lib32z1`库：

```
# apt install lib32z1


```

重新`Build`→`Make Project`之后，就编译成功了

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9b8k9MyTHrLo9E2MuNOVkgEgXJMibJ5mp9J6497qLaNMJic1iaggAwzM8jw/640?wx_fmt=jpeg)

将这个`apk`直接重命名为`XposedBridge.jar`即可。在安卓源码目录中的`out/`文件夹中新建`java`文件夹，并且将`XposedBridge.jar`放置其中，路径为：`/root/Desktop/COMPILE/aosp712r8/out/java/XposedBridge.jar`。

#### 编译模块开发 API(XposedBridge)

编写插件时，我们需要的是可以导入的`jar`包，也就是`API`，先点击右侧的`Gradle`展开`Gradle`面板，展开`XposedBridge`→`App`→`Tasks`→`Other`，双击`jarStubs`就会自动开始编译`API`。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9boVx8x9jjyopQGFxHKZF3mUibwPrib8yZiahMpdQvZT86PaGibiayiaXrmQpA/640?wx_fmt=jpeg)

`jarStubs`下面还有一个`jarStubsSource`，也双击一下进行编译。在`app/build/api/`目录下即会生成`api.jar`和`api-sources.jar`。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bNkwQibdj1y5pde1EeMPRbOIKrcShfRksz5rP5IWPlCiaV9gusm6IPjug/640?wx_fmt=jpeg)

解压个`api-source.jar`康康，跟我们前文下载的`api-82-sources.jar`包的结构和内容是完全一致的。

```
# tree
.
├── android
│   ├── app
│   │   ├── AndroidAppHelper.java
│   │   └── package-info.java
│   └── content
│       └── res
│           ├── package-info.java
│           ├── XModuleResources.java
│           ├── XResForwarder.java
│           └── XResources.java
├── de
│   └── robv
│       └── android
│           └── xposed
│               ├── callbacks
│               │   ├── IXUnhook.java
│               │   ├── package-info.java
│               │   ├── XCallback.java
│               │   ├── XC_InitPackageResources.java
│               │   ├── XC_LayoutInflated.java
│               │   └── XC_LoadPackage.java
│               ├── DexCreator.java
│               ├── IXposedHookCmdInit.java
│               ├── IXposedHookInitPackageResources.java
│               ├── IXposedHookLoadPackage.java
│               ├── IXposedHookZygoteInit.java
│               ├── IXposedMod.java
│               ├── package-info.java
│               ├── SELinuxHelper.java
│               ├── services
│               │   ├── BaseService.java
│               │   ├── BinderService.java
│               │   ├── DirectAccessService.java
│               │   ├── FileResult.java
│               │   ├── package-info.java
│               │   └── ZygoteService.java
│               ├── XC_MethodHook.java
│               ├── XC_MethodReplacement.java
│               ├── XposedBridge.java
│               ├── XposedHelpers.java
│               ├── XposedInit.java
│               └── XSharedPreferences.java
└── META-INF
    └── MANIFEST.MF

11 directories, 33 files



```

到了这里我们写工程时导入的项目包和源码包就都有了。

#### 编译定制版`art`解释器 (android_art)

接下来的几个模块的编译，需要继续在源码目录中完成，首先进入源码目录中，然后要确保`aosp`编译成功，并且刷入手机毫无问题，正常开机。这一步就非常长，且前文已经有过介绍，正常编译`aosp`的过程此处不表。任何时候重新编译也是非常快的，一般只要几分钟。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9b6YBZmJksdp0WBlmeXwsBYOmSLKXPjtVK9yufekNK1ErKzawS2DAkow/640?wx_fmt=jpeg)

然后将现有的`art/`目录修改文件夹名，保留到桌面上。

```
# mv art/ ~/Desktop/art_backup/


```

```
# git clone https://github.com/rovo89/android_art.git
# mv android_art/ art/


```

然后再来编译一次：

```
# make -j8


```

只要环境与刚刚相同，这次也是一遍过，只是时间会长很多，一般为几十分钟。

```
root@roysue:~/Desktop/COMPILE/aosp712r8/aosp712r8# make -j8
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=7.1.2
TARGET_PRODUCT=aosp_sailfish
TARGET_BUILD_VARIANT=userdebug
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm64
TARGET_ARCH_VARIANT=armv8-a
TARGET_CPU_VARIANT=generic
TARGET_2ND_ARCH=arm
TARGET_2ND_ARCH_VARIANT=armv7-a-neon
TARGET_2ND_CPU_VARIANT=krait
HOST_ARCH=x86_64
HOST_2ND_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-5.3.0-kali2-amd64-x86_64-with-debian-kali-rolling
HOST_CROSS_OS=windows
HOST_CROSS_ARCH=x86
HOST_CROSS_2ND_ARCH=x86_64
HOST_BUILD_TYPE=release
BUILD_ID=N2G47O
OUT_DIR=out
============================================
fatal: not a git repository (or any parent up to mount point /root/Desktop)
Stopping at filesystem boundary (GIT_DISCOVERY_ACROSS_FILESYSTEM not set).
Running kati to generate build-aosp_sailfish.ninja...
art/libart_fake/Android.mk was modified, regenerating...
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=7.1.2
TARGET_PRODUCT=aosp_sailfish
TARGET_BUILD_VARIANT=userdebug
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm64
TARGET_ARCH_VARIANT=armv8-a
TARGET_CPU_VARIANT=generic
TARGET_2ND_ARCH=arm
TARGET_2ND_ARCH_VARIANT=armv7-a-neon
TARGET_2ND_CPU_VARIANT=krait
HOST_ARCH=x86_64
HOST_2ND_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-5.3.0-kali2-amd64-x86_64-with-debian-kali-rolling
HOST_CROSS_OS=windows
HOST_CROSS_ARCH=x86
HOST_CROSS_2ND_ARCH=x86_64
HOST_BUILD_TYPE=release
BUILD_ID=N2G47O
OUT_DIR=out
============================================
including ./abi/cpp/Android.mk ...
including ./art/Android.mk ...
including ./bionic/Android.mk ...
including ./bootable/recovery/Android.mk ...
including ./build/libs/host/Android.mk ...
including ./build/target/board/Android.mk ...
including ./build/target/product/security/Android.mk ...
including ./build/tools/Android.mk ...
including ./cts/Android.mk ...
FindEmulator: find: `cts/apps/CtsVerifier/src/android': No such file or directory
FindEmulator: find: `cts/hostsidetests/os/test-apps/HostLinkVerificationApp/src': No such file or directory
FindEmulator: find: `cts/libs/commonutil/src': No such file or directory
FindEmulator: cd: cts/tests/libcore/ojluni/resources: No such file or directory
FindEmulator: find: `cts/libs/commonutil/src': No such file or directory
including ./dalvik/Android.mk ...
including ./developers/samples/android/security/FingerprintDialog/Application/src/main/Android.mk ...
including ./development/apps/BluetoothDebug/Android.mk ...
including ./development/apps/BuildWidget/Android.mk ...
including ./development/apps/CustomLocale/Android.mk ...
including ./development/apps/Development/Android.mk ...
...
...
including ./system/media/audio_utils/Android.mk ...
including ./system/media/brillo/audio/audioservice/Android.mk ...
including ./system/media/camera/src/Android.mk ...
including ./system/media/camera/tests/Android.mk ...
including ./system/media/radio/src/Android.mk ...
including ./system/nativepower/Android.mk ...
including ./system/netd/Android.mk ...
including ./system/security/keystore-engine/Android.mk ...
including ./system/security/keystore/Android.mk ...
including ./system/security/softkeymaster/Android.mk ...
including ./system/sepolicy/Android.mk ...
including ./system/tools/aidl/Android.mk ...
including ./system/tpm/trunks/Android.mk ...
including ./system/update_engine/Android.mk ...
including ./system/vold/Android.mk ...
including ./system/weaved/Android.mk ...
including ./system/webservd/Android.mk ...
including ./test/vts/Android.mk ...
including ./tools/external/fat32lib/Android.mk ...
including ./tools/test/connectivity/Android.mk ...
PRODUCT_COPY_FILES device/generic/goldfish/data/etc/apns-conf.xml:system/etc/apns-conf.xml ignored.
PRODUCT_COPY_FILES device/google/marlin/audio_effects.conf:system/etc/audio_effects.conf ignored.
PRODUCT_COPY_FILES device/google/marlin/fstab.common:root/fstab.sailfish ignored.
No private recovery resources for TARGET_DEVICE sailfish
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/bin/nanotool'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/bin/nanotool'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/etc/permissions/com.android.ims.rcsmanager.xml'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/etc/permissions/com.android.ims.rcsmanager.xml'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/framework/com.android.ims.rcsmanager.jar'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/framework/com.android.ims.rcsmanager.jar'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib64/libminui.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib64/libminui.so'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib64/libtinyxml.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib64/libtinyxml.so'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib64/libwifi-hal-qcom.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib64/libwifi-hal-qcom.so'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib/libion.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib/libion.so'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib/libminui.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib/libminui.so'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib/libmm-qcamera.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib/libmm-qcamera.so'
build/core/Makefile:34: warning: overriding commands for target `out/target/product/sailfish/system/lib/libtinyxml.so'
build/core/base_rules.mk:319: warning: ignoring old commands for target `out/target/product/sailfish/system/lib/libtinyxml.so'
Starting build with ninja
ninja: Entering directory `.'
[  1% 43/2298] Ensure Jack server is installed and started
Jack server already installed in "/root/.jack-server"
Server is already running
Bad request, see Jack server log
[  6% 160/2298] host Java: ahat (out/host/common/obj/JAVA_LIBRARIES/ahat_intermediates/classes)
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[ 94% 1059/1123] target R.java/Manifest.java: LiveTv (out/target/common/obj/APPS/LiveTv_intermediates/src/R.stamp)
warning: string 'title_br_tv_10' has no default translation.
warning: string 'title_br_tv_12' has no default translation.
warning: string 'title_br_tv_14' has no default translation.
warning: string 'title_br_tv_16' has no default translation.
warning: string 'title_br_tv_18' has no default translation.
warning: string 'title_br_tv_l' has no default translation.
warning: string 'title_kr_tv_12' has no default translation.
warning: string 'title_kr_tv_15' has no default translation.
warning: string 'title_kr_tv_19' has no default translation.
warning: string 'title_kr_tv_7' has no default translation.
warning: string 'title_kr_tv_all' has no default translation.
Warning: AndroidManifest.xml already defines minSdkVersion (in http://schemas.android.com/apk/res/android); using existing value in manifest.
Warning: AndroidManifest.xml already defines targetSdkVersion (in http://schemas.android.com/apk/res/android); using existing value in manifest.
[ 96% 1079/1123] host Java: ahat-tests (out/host/common/obj/JAVA_LIBRARIES/ahat-tests_intermediates/classes)
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Note: art/tools/ahat/test/SortTest.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[ 99% 1104/1113] build out/target/product/sailfish/obj/NOTICE.html
Combining NOTICE files into HTML
Combining NOTICE files into text
[ 99% 1112/1113] Target system fs image: out/target/product/sailfish/obj/PACKAGING/systemimage_intermediates/system.img
Running:  mkuserimg.sh -s /tmp/tmpsdYYzs out/target/product/sailfish/obj/PACKAGING/systemimage_intermediates/system.img ext4 / 2113941504 -D out/target/product/sailfish/system -L / out/target/product/sailfish/root/file_contexts.bin
make_ext4fs -s -T -1 -S out/target/product/sailfish/root/file_contexts.bin -L / -l 2113941504 -a / out/target/product/sailfish/obj/PACKAGING/systemimage_intermediates/system.img /tmp/tmpsdYYzs out/target/product/sailfish/system
Creating filesystem with parameters:
    Size: 2113941504
    Block size: 4096
    Blocks per group: 32768
    Inodes per group: 8064
    Inode size: 256
    Journal blocks: 8064
    Label: /
    Blocks: 516099
    Block groups: 16
    Reserved block group size: 127
Created filesystem with 2359/129024 inodes and 213371/516099 blocks
build_verity_tree -A aee087a5be3b982978c923f566a94613496b417f2af592639bc80d141e34dfe7 out/target/product/sailfish/obj/PACKAGING/systemimage_intermediates/system.img /tmp/tmpb1HG7F_verity_images/verity.img
system/extras/verity/build_verity_metadata.py build 2113941504 /tmp/tmpb1HG7F_verity_images/verity_metadata.img 64dd075c54653cc41da5db970d0313aaa41238456d3dea260f6cd83549175e3a aee087a5be3b982978c923f566a94613496b417f2af592639bc80d141e34dfe7 /dev/block/bootdevice/by-name/system verity_signer build/target/product/security/verity.pk8
cat /tmp/tmpb1HG7F_verity_images/verity_metadata.img >> /tmp/tmpb1HG7F_verity_images/verity.img
fec -e -p 0 out/target/product/sailfish/obj/PACKAGING/systemimage_intermediates/system.img /tmp/tmpb1HG7F_verity_images/verity.img /tmp/tmpb1HG7F_verity_images/verity_fec.img
cat /tmp/tmpb1HG7F_verity_images/verity_fec.img >> /tmp/tmpb1HG7F_verity_images/verity.img
append2simg out/target/product/sailfish/obj/PACKAGING/systemimage_intermediates/system.img /tmp/tmpb1HG7F_verity_images/verity.img
[100% 1113/1113] Install system fs image: out/target/product/sailfish/system.img
out/target/product/sailfish/system.img+ maxsize=2192424960 blocksize=135168 total=873995832 reserve=22167552

#### make completed successfully (16:38 (mm:ss)) ####

root@roysue:~/Desktop/COMPILE/aosp712r8/aosp712r8# ls


```

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bCbalmWI2S1Ja8uFYc7vfIISeQSAwRYwrUk8AUNGRdMNxP1aYp7aicoQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bl7kmQvVIMRs1hVbibAWfX5BpQLLxiaLZ2C6ASI4AibBtoWU5twB72vYNg/640?wx_fmt=jpeg)

#### 编译本体 (Xposed)

编译`Xposed`倒是非常轻松，只需要将其放在源码的指定目录中即可，`XposedTools`会自动找到它进行编译：

```
# cd /root/Desktop/COMPILE/aosp712r8/
# cd frameworks/base/cmds
# git clone https://github.com/rovo89/Xposed xposed

```

#### 编译刷机包 (XposedTools)

最后一步也是最难、最复杂的步骤，那就是使用`XposedTools`来编译出可以刷入到手机的`xposed-v89-sdk25-arm64.zip`刷机包，由`XposedInstaller`下载并且刷入至手机。

首先下载源码：

```
git clone https://github.com/rovo89/XposedTools.git


```

将文件夹内的编译配置模板拷贝一份：

```
# cp build.conf.sample build.conf


```

并对`build.conf`进行修改：

```
[General]
outdir = /root/Desktop/COMPILE/aosp712r8/out
javadir = /root/Desktop/COMPILE/aosp712r8/out/java

[Build]
# Please keep the base version number and add your custom suffix
version = 89 (custom build by r0ysue / %s)
# makeflags = -j4

[GPG]
sign = release
user = 852109AA!

# Root directories of the AOSP source tree per SDK version
[AospDir]
25 = /root/Desktop/COMPILE/aosp712r8

# SDKs to be used for compiling BusyBox
# Needs https://github.com/rovo89/android_external_busybox
[BusyBox]
arm = 25
x86 = 25
armv5 = 25




```

要将上面编译`XposedBridge`的成品重命名为`XposedBridge.jar`，放到`/root/Desktop/COMPILE/aosp712r8/out/java`目录中去。

然后来尝试编译一下试试，不出意外出错了。

```
# ./build.pl -t arm64:25
Can't locate File/ReadBackwards.pm in @INC (you may need to install the File::ReadBackwards module) (@INC contains: /root/Desktop/XPOSED/XposedTools /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.30.0 /usr/local/share/perl/5.30.0 /usr/lib/x86_64-linux-gnu/perl5/5.30 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl/5.30 /usr/share/perl/5.30 /usr/local/lib/site_perl /usr/lib/x86_64-linux-gnu/perl-base) at /root/Desktop/XPOSED/XposedTools/Xposed.pm line 12.
BEGIN failed--compilation aborted at /root/Desktop/XPOSED/XposedTools/Xposed.pm line 12.
Compilation failed in require at ./build.pl line 9.
BEGIN failed--compilation aborted at ./build.pl line 9.


```

`XposedTools`使用的是`perl`开发环境，需要安装一系列的`perl`环境及三方包，首先安装环境：

```
# apt install libconfig-inifiles-perl libauthen-ntlm-perl libclass-load-perl libcrypt-ssleay-perl libdata-uniqid-perl libdigest-hmac-perl libdist-checkconflicts-perl libfile-copy-recursive-perl libfile-tail-perl


```

全部安装完成之后，再使用`perl`的包管理器安装第三方工具包，安装这些第三方包的时候，由于包管理服务器位于国外，速度必定非常慢，要有耐心。

```
# perl -MCPAN -e 'install Config::IniFiles'

CPAN.pm requires configuration, but most of it can be done automatically.
If you answer 'no' below, you will enter an interactive dialog for each
configuration option instead.

Would you like to configure as much as possible automatically? [yes] yes


Autoconfiguration complete.

commit: wrote '/root/.cpan/CPAN/MyConfig.pm'

You can re-run configuration any time with 'o conf init' in the CPAN shell
Fetching with LWP:
http://www.cpan.org/authors/01mailrc.txt.gz
Reading '/root/.cpan/sources/authors/01mailrc.txt.gz'
............................................................................DONE
Fetching with LWP:
http://www.cpan.org/modules/02packages.details.txt.gz
read timeout at /usr/share/perl5/Net/HTTP/Methods.pm line 243. at /usr/share/perl5/LWP/UserAgent.pm line 984.



```

```
# perl -MCPAN -e 'install File::Tail'
Reading '/root/.cpan/sources/authors/01mailrc.txt.gz'
............................................................................DONE
Fetching with LWP:
http://www.cpan.org/modules/02packages.details.txt.gz
Reading '/root/.cpan/sources/modules/02packages.details.txt.gz'
  Database was generated on Thu, 23 Apr 2020 04:17:02 GMT
.............
  New CPAN.pm version (v2.27) available.
  [Currently running version is v2.22]
  You might want to try
    install CPAN
    reload cpan
  to both upgrade CPAN.pm and run the new version without leaving
  the current session.


...............................................................DONE
Fetching with LWP:
http://www.cpan.org/modules/03modlist.data.gz
Reading '/root/.cpan/sources/modules/03modlist.data.gz'
DONE
Writing /root/.cpan/Metadata
File::Tail is up to date (1.3).



```

```
# perl -MCPAN -e 'install File::ReadBackwards'
Reading '/root/.cpan/Metadata'
  Database was generated on Thu, 23 Apr 2020 04:17:02 GMT
Running install for module 'File::ReadBackwards'
Fetching with LWP:
http://www.cpan.org/authors/id/U/UR/URI/CHECKSUMS
Checksum for /root/.cpan/sources/authors/id/U/UR/URI/File-ReadBackwards-1.05.tar.gz ok
'YAML' not installed, will not store persistent state
Configuring U/UR/URI/File-ReadBackwards-1.05.tar.gz with Makefile.PL
Checking if your kit is complete...
Looks good
Generating a Unix-style Makefile
Writing Makefile for File::ReadBackwards
Writing MYMETA.yml and MYMETA.json
  URI/File-ReadBackwards-1.05.tar.gz
  /usr/bin/perl Makefile.PL INSTALLDIRS=site -- OK
Running make for U/UR/URI/File-ReadBackwards-1.05.tar.gz
cp ReadBackwards.pm blib/lib/File/ReadBackwards.pm
Manifying 1 pod document
  URI/File-ReadBackwards-1.05.tar.gz
  /usr/bin/make -- OK
Running make test for URI/File-ReadBackwards-1.05.tar.gz
PERL_DL_NONLAZY=1 "/usr/bin/perl" "-MExtUtils::Command::MM" "-MTest::Harness" "-e" "undef *Test::Harness::Switches; test_harness(0, 'blib/lib', 'blib/arch')" t/*.t
t/bw.t .......... ok
t/large_file.t .. ok
All tests successful.
Files=2, Tests=173,  0 wallclock secs ( 0.01 usr  0.02 sys +  0.15 cusr  0.07 csys =  0.25 CPU)
Result: PASS
  URI/File-ReadBackwards-1.05.tar.gz
  /usr/bin/make test -- OK
Running make install for URI/File-ReadBackwards-1.05.tar.gz
Manifying 1 pod document
Installing /usr/local/share/perl/5.30.0/File/ReadBackwards.pm
Installing /usr/local/man/man3/File::ReadBackwards.3pm
Appending installation info to /usr/local/lib/x86_64-linux-gnu/perl/5.30.0/perllocal.pod
  URI/File-ReadBackwards-1.05.tar.gz
  /usr/bin/make install  -- OK



```

最后还会有一个`perl -MCPAN -e 'install Archive::Zip'`在安装时会一直报错：

```
# ./build.pl -t arm64:25
Can't locate Archive/Zip.pm in @INC (you may need to install the Archive::Zip module) (@INC contains: /root/Desktop/XPOSED/XposedTools /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.30.0 /usr/local/share/perl/5.30.0 /usr/lib/x86_64-linux-gnu/perl5/5.30 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl/5.30 /usr/share/perl/5.30 /usr/local/lib/site_perl /usr/lib/x86_64-linux-gnu/perl-base) at ./build.pl line 11.
BEGIN failed--compilation aborted at ./build.pl line 11.


```

这时可以使用`cpan`命令进入命令行模式，输入`install Archive::Zip`命令来手动安装`Archive::Zip`包。

全部安装完成之后，直接编译即可。

```
# ./build.pl -t arm64:25

```

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bS6yLXe8ohiatTmaVoABQAg4wkfhek1beATV2Aj2LicFeqzJsicpz7Aumw/640?wx_fmt=jpeg)

由上图可见，`XposedTools`在编译生成刷机包时，使用的编译链工具还是安卓源码包里的工具。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bGicibdCRVRicHbnDrDqIZWnOibobhYS3iazZkHH1DrQ2gTDN3R421qG32Iw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9b7aq8J5LRmYRSmtcYu7IamHU3Gpb8L6lUXNZJejpREABMsiawGYe0ictw/640?wx_fmt=png)

编译成品的位置如下图：

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bKFS6AickIGmRyHn2ylbDX8ggwop0MpeuDicvJcgicBanQDVyzC3xmRB2g/640?wx_fmt=jpeg)

### 刷机、运行模块并开发插件

#### 编译成品进行刷机

编译出来的刷机包为`xposed-v89-sdk25-arm64-custom-build-by-r0ysue-20200425.zip`，重命名为与官网相同的`xposed-v89-sdk25-arm64.zip`即可。

> 另外可以跟官网下载的进行比对下，大小、解压出来的文件大小、位置等是否相符或接近，按照此篇教程来效果肯定是一样的，毕竟很多人已经跟笔者一样编译成功了，解压虚拟机也有从零开始的效果，同样的环境得到的会是同样的结果。

最终刷入手机我采用的是替换`XposedInstaller`的下载文件的方式，也就是在本地搭一个服务器（我使用的是`lighttpd`，一键傻瓜搭建`http`服务器），修改下`XposedInstaller`的源码：

```
package de.robv.android.xposed.installer.util;

import android.app.DownloadManager;
...
import de.robv.android.xposed.installer.repo.ReleaseType;

public class DownloadsUtil {
    public static final String MIME_TYPE_APK = "application/vnd.android.package-archive";
    public static final String MIME_TYPE_ZIP = "application/zip";
    private static final Map<String, DownloadFinishedCallback> mCallbacks = new HashMap<>();
    private static final XposedApp mApp = XposedApp.getInstance();
    private static final SharedPreferences mPref = mApp
            .getSharedPreferences("download_cache", Context.MODE_PRIVATE);

    public static class Builder {
        private final Context mContext;
        private String mTitle = null;
        private String mUrl = null;
        private DownloadFinishedCallback mCallback = null;
        private MIME_TYPES mMimeType = MIME_TYPES.APK;
        private File mDestination = null;
        private boolean mDialog = false;

        public Builder(Context context) {
            mContext = context;
        }

        public Builder setTitle(String title) {
            mTitle = title;
            return this;
        }

        public Builder setUrl(String url) {
            //mUrl = url;
            //将这里改成指向本地服务器的刷机包文件
            mUrl = "http://192.168.0.9/xposed-v89-sdk25-arm64.zip";
            return this;
        }


```

> 注意要测试下，手机浏览器要可以直接访问并下载这个刷机包文件才行。可以先打开`Chrome`测试一下。

重新编译`XposedInstaller`安装到手机上，点击 “安装 / 更新”→`Install`，即即可自动下载和刷入成功。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9biaV2FYnXZOkb0emY75zvhBVWRlAo4hgqTtwqswvDFnXZ5Lu3GMiaIggg/640?wx_fmt=png)

#### 运行`GravityBox`模块

重启之后，自己编译的`Xposed`安装成功。再安装个`GravityBox`来看下，是可以直接激活和使用的，大家看到我的状态栏变成了深蓝色。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9b118XmYI8JvmPmxrH6PpdAibUnl4Cias0t06e6yKsYo7YGic8YtJltze2w/640?wx_fmt=png)

试试看`Xposed Checker`，这也是我们最终要`bypass`的主要对象：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9buXfyIAxIjycjtMhMweTkdtwSprCFQFSyWticXwlNrH39KUibbQ0rF18A/640?wx_fmt=png)

#### 使用编译成品`API`进行模块开发

这部分其实网上的资料浩如烟海，跟正常开发一个`Xposed`模块唯一不同的地方在于，不需要使用在线的`api`，而是使用我们刚刚编译出来的那个`api.jar`，把`api.jar`放到项目的`libs/`目录中，在`app`的`build.gradle`里，使用`compileOnly files('libs/api.jar')`来指定使用刚刚编译出来的库：

```
dependencies {
    implementation 'androidx.appcompat:appcompat:1.1.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test.ext:junit:1.1.1'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'
    compileOnly files('libs/api.jar')
}


```

最终`hook`的代码如下，当然，这也是官方`Tutorial`的案例：

```
public class hook implements IXposedHookLoadPackage {
    public void handleLoadPackage(final LoadPackageParam lpparam) throws Throwable {
        if (!lpparam.packageName.equals("com.android.systemui"))
            return;

        findAndHookMethod("com.android.systemui.statusbar.policy.Clock", lpparam.classLoader, "updateClock", new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                TextView tv = (TextView) param.thisObject;
                String text = tv.getText().toString();
                tv.setText(text + "r0ysue :)");
                tv.setTextColor(Color.RED);
            }
        });
    }
}


```

核心逻辑就是时间栏后面加个字符串以及变个颜色，可以看到生效之后的效果如下图（注意要关掉`GravityBox`噢）。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8J1VzlUx4yPj4ewZQ5bQm9bOzCW1IsCuibicHkGkbQGd8YHnXvRPB6nE78wcbDNZibz0wJQ3Or4Briccw/640?wx_fmt=jpeg)

完整项目打包地址在附件中这里：XposedApp2.zip

### 小总结

本篇中我们完成了魔改代码的前置条件——编译官方原版、运行模块并能开发插件，下一篇中介绍如何修改`XPOSED`源码，实现真正的修改特征，从高维过框架检测，敬请期待。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)