> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/K5v4JvjK8H_mFB5eQXPQDg)

**一、前期准备**  

      **手机设备需要满足如下条件:**  

```
1.手机adb 拥有root权限，可以通过执行adb root命令
2.手机系统版本>=8.0
3.本文讨论的riru不是最新版本的，是基于替换libmemtrack.so完成zygote注入的版本

```

 **riru 和 Edxposed 相关文件准备:**

```
可以微信搜索"QDROID88888"或者扫描二维码加入公众号，发送消息"adbEdxposed"获取下载地址

```

  riru 和 Edxposed 相关文件下载好之后，目录如下所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431EPCNVr0wnIiaopvCzPNL2fKU9NRemibFt9RXNbmawb4kM4UtGupjQGyOA2njO3Ptia5T3kmER8kBlw/640?wx_fmt=png)

**二、相关文件说明**

 riru 和 Edxposed 下载好之后，有两个目录和一个 apk 文件。

 **1.EdxposedManager**

  该 apk 文件安装到手机用来检测框架是否安装成功

 **2.data 目录**

 data 目录下面的文件是用 Magisk 刷 riru-core 的时候生成的文件。"/data/misc/riru/modules" 存放了 riru 模块信息。

 **3.system 目录**

 该目录存放了修改过的 libmemtrack.so 文件，riru 库文件以及 Edxposed 安装释放的 jar 和 so 文件。

**三、adb 命令安装 Edxposed**

**1.adb 安装 riru  
**

 执行如下命令:

```
C:\Users\Qiang>adb root
adbd is already running as root
C:\Users\Qiang>adb remount
[libfs_mgr]dt_fstab: Skip disabled entry for partition vendor
[libfs_mgr]dt_fstab: Skip disabled entry for partition vendor
[libfs_mgr]dt_fstab: Skip disabled entry for partition vendor
remount succeeded
C:\Users\Qiang>adb push E:\studyspace\Android10\magisk\test\data   /
E:\studyspace\Android10\magisk\test\data\: 5 files pushed, 0 skipped. 0.0 MB/s (329 bytes in 0.030s)

```

**2.adb 安装 Edxposed**

```
C:\Users\Qiang>adb root
adbd is already running as root
C:\Users\Qiang>adb remount
[libfs_mgr]dt_fstab: Skip disabled entry for partition vendor
[libfs_mgr]dt_fstab: Skip disabled entry for partition vendor
[libfs_mgr]dt_fstab: Skip disabled entry for partition vendor
remount succeeded
C:\Users\Qiang>adb push E:\studyspace\Android10\magisk\test\system   /
E:\studyspace\Android10\magisk\test\system\: 12 files pushed, 0 skipped. 19.4 MB/s (4418316 bytes in 0.218s)

```

**3. 安装 EdXposedManager**

```
C:\Users\Qiang>adb install E:\studyspace\Android10\magisk\mogai\test\EdXposedManager-4.5.7-45700-org.meowcat.edxposed.manager-release.apk
Performing Streamed Install
Success

```

**4. 重启手机  
**

 以上安装完成之后，重启手机，打开 Edxposed Manager 查看是否安装成功。

 成功之后如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431EPCNVr0wnIiaopvCzPNL2fqaPiaGCszoV3v8G0gIF4Xey4qJIB8mjrDkcDJGKq6e6VbsKwcZV0VGg/640?wx_fmt=png)

Edxposed 学习研究相关文章:  

[Edxposed 学习研究 (一) 手把手教你安装 Edxposed](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484016&idx=1&sn=a2c2bc275c1c7a17a1c1fd2988923001&chksm=ce075335f970da23ff69f81cae607ade411afbeb11d5aaaf703a037b982a55f162e55c028ae4&scene=21#wechat_redirect)

[Edxposed 学习研究 (二) 手把手编译 Riru 和 Edxposed 工程源码](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484033&idx=1&sn=20bd2ce390d418a90ce3f87d2ccf0368&chksm=ce0753c4f970dad27462339b6fddc9d818cd04d4921c8e0d6accb7042b24d839537af5f8027d&scene=21#wechat_redirect)  

**专注安卓系统开发定制、安卓 ndk 开发、安卓应用安全开发和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳、安卓行为分析沙箱、安卓 app 检测等等。关注公众号第一时间接收更新文章。**

**微信搜索公众号 "QDOID88888" 关注或者扫以下二维码关注:**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)