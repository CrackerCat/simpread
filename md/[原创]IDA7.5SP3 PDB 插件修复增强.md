> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266189.htm)

[](#修复了以下问题：)修复了以下问题：
=====================

**1.**pdb 所在路径包含了中文或者 Unicode 特殊字符，IDA 会无法自动打开该 pdb。  
如下图所示：  
![](https://bbs.pediy.com/upload/attach/202102/346736_ZKPY76QYSQXVP65.png)  
**2.**VS2015 及以上生成的某些带 PDB 的发布版的程序，使用 IDA 打开会报错并崩溃，即使切换到 PDB_PROVIDER_MSDIA 也依然报错崩溃。  
如下图 (切换到 PDB_PROVIDER_MSDIA 前)：  
![](https://bbs.pediy.com/upload/attach/202102/346736_YB2YBX9RQMFBCWV.png)  
如下图 (切换到 PDB_PROVIDER_MSDIA 后)：  
![](https://bbs.pediy.com/upload/attach/202102/346736_6SWWYVPCMV3895Z.png)  
切换后依然崩溃是因为 IDA 目前最高只能使用 VS2008 的 MSDIA 动态链接库，即图中红色圈出来的 "C:\Program Files\Common Files\Microsoft Shared\VC\msdia90.dll", 而这个版本的库太老了，并不能保证完美兼容后面版本的 VC 生成的各种 PDB 文件。切换前报错是因为 IDA 的 BUG，对 PDB 文件的解析并不完美。  
**3.**VS2015 及之后版本中使用 / DEBUG:FASTLINK(也即 Partial PDB) 链接的程序无法识别显示用户定义的类型变量。  
![](https://bbs.pediy.com/upload/attach/202102/346736_S574CP7QZ9UDPYS.png)  
**_注意：从 VS2017 开始调试版默认的 / DEBUG 就是 / DEBUG:FASTLINK 了。_**  
安装本修复增强插件后的效果图：  
![](https://bbs.pediy.com/upload/attach/202102/346736_B69SCQBKP37VEX9.png)  
**4.** 修复了无法下载压缩格式的 PDB 的文件的问题  
如 Mozilla Firefox 的 PDB 文件都是压缩格式的，IDA 当前最新版本安装本修复插件前下载会报错。

[](#加入了以下增强特性：)加入了以下增强特性：
=========================

**1.** 在安装了 Internet Download Manager(以下简称 IDM) 的机器上支持自动调用 IDM 进行多线程高速下载 PDB。  
**2.** 使用 MSDIA 内建下载时 (通过 symsrv.dll) 也支持了显示下载进度百分比。  
**3.** 使用 MSDIA 内建下载时 (通过 symsrv.dll) 也支持了立刻取消下载。

[](#目前还存在的问题：)目前还存在的问题：
=======================

**1.** 加载某些网上的 PDB 文件会报一次下面这个警告，不过不是非常严重。  
![](https://bbs.pediy.com/upload/attach/202102/346736_FCKSBXZ2XJAMEDQ.png)  
目前发现的是某些 Mozilla 和 Google 服务器上的文件会，微软服务器上的倒还没碰到。这个问题在 SP2 中是存在的，但在 SP3 中我看被官方修复了，可是我手中的 idasdk75.zip 也是网上下载的 2020 年 5 月 27 日的版本，并非 SP3 发布后的版本，所以是带这个问题的。如果哪个坛友有 idasdk 的最新版本可以共享给我一份，我重新编译一下应该这个小问题就修复了。

 

**源代码开源在：https://github.com/sonyps5201314/pdb**  
如果坛里有正版用户，可以将代码发送给官方，希望官方能吸纳进去，那样就免得用户手动去修复了，省去了麻烦。  
编译好的程序在附件，用户也可以自己编译源码。  
使用方法是：解压压缩包中的三个文件到你的 IDA7.5 SP3 的根目录。注意解压会替换掉插件 plugins 目录下的 pdb.dll 和 pdb64.dll。建议先将这两个原始文件压缩备份好。

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 3 小时前 被 sonyps 编辑 ，原因：

上传的附件：

*   [pdb_enhance_and_bugfix_20210227.7z](javascript:void(0)) （192.67kb，13 次下载）