> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282541.htm)

> [原创] 利用 Framework Patch 过掉 BL 锁状态检测（小白向）

介绍
==

该项目源自 Github 外国大佬  
![](https://bbs.kanxue.com/upload/attach/202407/988654_B5RKGB5M3USUBHB.jpg)  
**项目链接：**  
[GitHub - chiteroman/FrameworkPatch: Modify framework.jar to build on system level a valid certificate chain](https://github.com/chiteroman/FrameworkPatch)  
在此声明，本教程也依托于 Github 项目上的教程和其他人发的教程，但后者这篇教程好像被原作者删除了。我也找不到了，无法提供相关信息，若原作者看到，请联系我补充。  
PS：  
原项目自带教程，但是只有英文版教程，并且教程详尽程度对我这样的小白不是很友好  
所以写了这篇小白向的教程  
大佬 or 有安卓开发经验的师傅可以直接看 Github 原项目

作用
--

对于已解锁 BL 的手机，这个项目通过对手机根目录下`/system/framework/framework.jar`这个文件进行修改来过掉 BL 锁状态检测  
如果仅利用项目自带的 keybox，可以过掉非硬件检测 or 密钥检测的 BL 锁状态验证，对付大部分软件，是没有问题的，而且目前有 BL 锁检测的软件其实很少。  
如果你有谷歌下发的 keybox，那么利用该项目，你可以近乎完美的过掉 BL 锁状态检测。  
注意，对`TEE损坏`的手机没有效果！！！例如 OPPO / 一加等品牌手机，解锁 BL 就会使 TEE 假死。  
本教程只有前者，就是教如何使用该项目。  
我也不知道如何向谷歌申请下发 keybox（我还是一个无安卓开发经验的小白）

存在的问题
-----

*   目前检测 BL 的软件还是较少
*   完美隐藏 BL 状态需要谷歌下发的密钥
*   与`Kitsune Magisk`的`Su List`有冲突
*   和`Shamiko`模块有冲突
*   影响性能？
*   未知的问题

研究价值
----

目前，有些安卓游戏是存在检测 BL 状态的  
以此为依据在设备上使用不同严格程度的检测方案  
之后，或许会有更多的软件对 BL 锁状态进行检测  
故，还是有一些研究价值

教程
==

**所需设备和工具：**

*   1 台已 Root 的安卓手机（装有 Magisk/Apatch 等）
*   Termux 或者 ZeroTermux
*   MT 管理器和密钥认证 APP
*   `Framework Patch` [Github 链接](https://github.com/chiteroman/FrameworkPatch)及 `Framework Patcher Go`[GitHub 链接](https://github.com/changhuapeng/FrameworkPatcherGO)
*   Magisk 模块模板`framework-modify`
*   科学上网

部分所需工具：  
[百度网盘](https://pan.baidu.com/s/1LtwGXQn5NOYGoPuGIyq3fA?pwd=ew6v)  
有 FrameworkPatcherGO，模块模板，密钥认证 APP  
其他工具请自行准备

该教程内容不需要电脑就可以实现  
Magisk 模块模板我会提供附件（非本人制作）  
**该项目有风险，请做好救砖的准备！！！**  
**该项目有风险，请做好救砖的准备！！！**  
**该项目有风险，请做好救砖的准备！！！**

内容预览
----

1.  配置 Termux 编译环境
2.  使用 Termux 编译所需`dex`
3.  利用`Framework Patcher Go`模块自动修改`framework.jar`中的`dex`（部分手机到此就已结束）
4.  手动修改`framework.jar`中的`dex`（部分手机的 framework.jar 无法使用`FrameworkPatcherGo`模块自动修改并安装）并制作模块

效果预览
----

![](https://bbs.kanxue.com/upload/attach/202407/988654_RVD2VZWBRRFV3RJ.jpg)![](https://bbs.kanxue.com/upload/attach/202407/988654_UKQFGTQJ3VEWTT6.jpg)  
第一张图为安装前，第二张图为安装后（使用项目自带证书，故会显示来自 AOSP 的根证书，非完美隐藏）

配置 Termux 编译环境
--------------

### 换国内源

使用 Termux 执行

```
sed -i 's@^\(deb.*stable main\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/termux-packages-24 stable main@' $PREFIX/etc/apt/sources.list
sed -i 's@^\(deb.*games stable\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/game-packages-24 games stable@' $PREFIX/etc/apt/sources.list.d/game.list
sed -i 's@^\(deb.*science stable\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/science-packages-24 science stable@' $PREFIX/etc/apt/sources.list.d/science.list
pkg update

```

### 安装 Java 和配置 Android 编译环境

大量参考该文章 [CSDN 链接](https://blog.csdn.net/Mingyueyixi/article/details/136014207)（嘿，就是 Copy）  
如果之后想在手机上利用 Termux 编译 APK 项目的，推荐观看下

```
#安装git
pkg install git -y
#安装openssh
pkg install openssh -y
#安装Java—sdk—17
pkg install openjdk-17 -y

```

请科学上网

```
curl -O https://googledownloads.cn/android/repository/commandlinetools-linux-11076708_latest.zip
 
ANDROID_HOME=~/android/sdk
mkdir -p $ANDROID_HOME/latest
unzip `ls |grep "commandlinetools-linux.*_latest.zip"` -d $ANDROID_HOME
# cmdline-tools 的产物需要移动到cmdline-tools/latest目录中，这是android sdk固定的路径组织形式
# 压缩包没有包含在latest文件夹中，自己移动一下
mv $ANDROID_HOME/cmdline-tools/* $ANDROID_HOME/latest
mv $ANDROID_HOME/latest $ANDROID_HOME/cmdline-tools

```

MT 管理器打开`/data/user/0/com.termux/files/home/`  
创建文件，名字是`.bashrc`  
填入以下内容

```
echo "用户："$(whoami)
  
if pgrep -x "sshd" >/dev/null
  then
   echo
   #echo "sshd运行中..."
  else
    sshd
    echo "自动启动sshd"
fi
export ANDROID_HOME=~/android/sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$PATH

```

Termux 执行如下命令

```
cd ~
source .bashrc

```

然后彻底关闭 Termux，重新打开。  
继续  
由于我们只需要编译，故执行第三条命令即可

```
#查看sdk列表
#sdkmanager --list
#安装安卓14平台开发工具
#sdkmanager --install "platforms;android-34"
#安装支持安卓14的构建工具
sdkmanager --install "build-tools;34.0.0"

```

接下来，我们下载 arm 版本的 sdk 工具（google 编译的安卓 sdk 没有 arm 版本 ）

```
cd ~
curl -LJO https://github.com/lzhiyong/android-sdk-tools/releases/download/34.0.3/android-sdk-tools-static-aarch64.zip
 
#根据构架选择，一般用上面那个就行了，如果更改了，需要把解压命令也更改下
#curl -LJO https://github.com/lzhiyong/android-sdk-tools/releases/download/34.0.3/android-sdk-tools-static-arm.zip
 
unzip android-sdk-tools-static-aarch64.zip -d ./armtools
# 下载的是34版本的，所以，覆盖到34版本的目录
mkdir -p ~/android/sdk/platform-tools
cp -p ./armtools/build-tools/*  ~/android/sdk/build-tools/34.0.0
cp -p ./armtools/platform-tools/*  ~/android/sdk/platform-tools

```

git 项目  
注意科学上网

```
cd ~
git clone https://github.com/chiteroman/FrameworkPatch.git

```

编译 dex
------

这里需要科学上网  
并且，执行所需时间较长，耐心等待  
最后，这里执行完了，还不算完  
会有报错，请勿担心

```
cd ./FrameworkPatch
echo "sdk.dir=$ANDROID_HOME" > local.properties
chmod +x ./gradlew
./gradlew build

```

执行后，你会看到如下报错  
![](https://bbs.kanxue.com/upload/attach/202407/988654_T2Z5T4223H32QUR.jpg)  
不急，执行以下命令替换 aapt2

```
TARGET="/data/user/0/com.termux/files/home/.gradle/caches/transforms-4"
find "$TARGET" -type f -name "aapt2" | while read -r aapt2_file; do
    cp -f ~/android/sdk/build-tools/34.0.0/aapt2 "$aapt2_file"
done

```

接下里继续编译

```
./gradlew assembleRelease
cp -f app/build/intermediates/dex/release/minifyReleaseWithR8/classes.dex ~

```

我们打开`/data/user/0/com.termux/files/home/`  
就可以看到有一个 dex 文件，留着备用

`Framework Patcher Go`模块自动修改
----------------------------

从`Framework Patcher Go`[GitHub 链接](https://github.com/changhuapeng/FrameworkPatcherGO)上下载模块  
MT 管理器打开 zip  
将`classes.dex`添加到 zip 中的  
`/META-INF/com/google/android/magisk/dex/`  
文件夹下  
然后 Magisk 刷入模块  
它会自动修改系统自带的`framework.jar`中的`dex`  
然后以面具模块的形式替换系统原来的`framework.jar`  
PS：过程中需要按音量上下键的  
![](https://bbs.kanxue.com/upload/attach/202407/988654_ZX68NVC7379PNUF.jpg)  
**注意，注意，注意**  
前面都是按音量上键  
但到了最后  
你看到

```
This step is not required unless your device crashes after installing this module.
Do you want to apply this step?

```

这两行英语后，请按音量下键  
等待刷完  
请提前做好救砖准备  
如 TWRP，音量键救砖模块等等

如果最后按音量下键后  
是会卡开机页面的  
则救砖

然后继续刷入模块  
但在最后选择按音量上键

如果正常开机  
那么到这里就结束了

但如果还是无法开机  
那就只能手动修改`framework.jar`中的`dex`  
请看接下来的教程

手动修改`framework.jar`
-------------------

文件路径：`/system/framework/framework.jar`  
复制文件到某个路径下  
不要直接修改系统路径下的 jar

MT 管理器打开 jar  
`查看`——`Dex编辑器++`——`全选`  
接下来

### **搜索方法名**：`engineGetCertificateChain`

在方法的末尾附近应该有如下几行代码：

```
const/4 v4, 0x0
aput-object v2, v3, v4
return-object v3

```

类似结构，但寄存器的值可能是不一样的  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_8DDKM5BA67PB93M.jpg)  
我们在`return-object XX`前加入

```
invoke-static {XX}, Lcom/android/internal/util/framework/Android;->engineGetCertificateChain([Ljava/security/cert/Certificate;)[Ljava/security/cert/Certificate;
move-result-object XX

```

将 `XX` 替换为对应的值  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_RD35AWVC7JVGXQU.jpg)  
保存返回

### **搜索方法名：**`newApplication`

可以看到有两个结果  
我们先点开第一个  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_4JDNE7XZBCPFB22.jpg)  
存在类似代码

```
.param XX，"context" #Landroid/content/Context;

```

在方法末尾`return` 之前添加以下代码：

```
invoke-static {XX}, Lcom/android/internal/util/framework/Android;->newApplication(Landroid/content/Context;)V

```

将 `XX` 替换为寄存器。  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_VKVU7F533FQDCQP.jpg)  
保存，看第二个搜索结果  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_PHP4G3KS8D6H32Y.jpg)  
看到和刚刚不同  
有 p1，p2，p3  
我们还是和第一个一样  
选择绿色高亮文本为`context`的那一行对应的寄存器  
在方法末尾`return` 之前添加以下代码：

```
invoke-static {XX}, Lcom/android/internal/util/framework/Android;->newApplication(Landroid/content/Context;)V

```

将 `XX` 替换为寄存器  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_8RQ7F9D9AVVZQAM.jpg)  
保存返回

### **搜索方法名：**`hasSystemFeature`

结果有很多  
我们只看`ApplicationPackageManager`类下的第一个  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_ZTEZNP44Z7YEPYE.jpg)  
在方法末尾`return` 之前添加以下代码：

```
invoke-static {v0, p1}, Lcom/android/internal/util/framework/Android;->hasSystemFeature(ZLjava/lang/String;)Z
move-result v0

```

如寄存器有不同，请自行更改，和之前一样就行  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_3AUQCAWCFF2WQUG.jpg)  
保存返回  
保存并退出  
在压缩文件中更新

### 利用模板制作模块

找到我们之前编译的 dex  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_2TRBR9HMKB9SM7X.jpg)  
根据`framework.jar`中 dex 的数量`n`个  
重命名编译好的 dex 为 classes[n+1].dex  
然后添加到`jar`内  
如图  
![](https://bbs.kanxue.com/upload/attach/202407/988654_7DR598CXCXAHTDW.jpg)  
保存返回

找到 Magisk 模块模板`Frist-framework-modify`  
注意最开始，选择带`Frist`的  
将修改后的`framework.jar`添加到压缩包`/system/framework/`下  
然后利用面具刷入，重启

能开机，结束  
不能开机  
选择不带`Frist`的`framework-modify`模板  
将修改后的`framework.jar`添加到压缩包`/system/framework/`下  
然后利用面具刷入，重启

> 带 Frist 和不带的区别：  
> 其实就和 Framework Patch Go 模块一样  
> 带 Frist 的模块和 Go 模块最后按音量下键的不会执行下列代码  
> 而不带 Frist 和 Go 模块最后按音量上键的会执行

```
if [ "$BOOTMODE" ] && { [ "$KSU" ] || [ "$APATCH" ]; }; then
    find "/system/framework" -type f -name 'boot-framework.*' -print0 |
        while IFS= read -r -d '' line; do
            mkdir -p "$(dirname "$MODPATH$line")" && mknod "$MODPATH$line" c 0 0
        done
elif [ "$BOOTMODE" ] && [ "$MAGISK_VER_CODE" ]; then
    find "/system/framework" -type f -name 'boot-framework.*' -print0 |
        while IFS= read -r -d '' line; do
            mkdir -p "$(dirname "$MODPATH$line")" && touch "$MODPATH$line"
        done
fi

```

如果刷入后还是无法正常开机  
只能先救砖  
再向项目作者提交 issue 了

结语
==

本篇文章基本上是对原项目作者的教程进行翻译，简单化，和补充  
希望未来可以写出更高质量的文章

在看雪的第一篇文章 Over  
未来待续

[[培训] 科锐软件逆向 50 期预科班报名即将截止，速来！！！ 50 期正式班报名火爆招生中！！！](https://mp.weixin.qq.com/s/HFghXQRTiTlk6oRKGotpHA)

[#系统相关](forum-161-1-126.htm)