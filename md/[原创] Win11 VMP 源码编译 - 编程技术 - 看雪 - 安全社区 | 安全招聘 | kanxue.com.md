> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280613.htm)

> [原创] Win11 VMP 源码编译

[原创] Win11 VMP 源码编译

21 小时前 740

[举报](javascript:void(0);)

### [原创] Win11 VMP 源码编译

 [![](http://passport.kanxue.com/upload/avatar/675/760675.png?1503406192)](user-home-760675.htm) [tritium](user-home-760675.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 2  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 21 小时前  740

 VMP 源码编译  

============

【摘要】
----

        本文展示 Leak 的 VMP 源码编译 console 与 GUI 的调试版，以及注册测试使用配置的图文过程。  

【前言】
----

        完整的泄露已经两月有余，这里只简测。

        测试环境：Win11 64 位系统，Visual Studio Community 2015（免费社区版），另安装 VC2008 的【en_visual_studio_2008_professional_x86_dvd_x14-26326.iso】里的 64 位编译工具 x86_amd（如果不改变配置，解决方案中的 VMProtectSDK 工程会用到），【qt-opensource-windows-x86-msvc2015-5.6.0.exe】中的 msvc2015_64（编译 GUI 版本需要，用于使用 QT 共享库模式，若要编译静态模式，需要另寻或自行编译 msvc2015_static64），最终得到可用及可调试 console 与 GUI 的 64 位版。

第 1 节 部分三方比较  

---------------

    对部分三方 leak 源代码做了部分比较，完整的是 git 上的 [GitHub - jmpoep/vmprotect-3.5.1](https://github.com/jmpoep/vmprotect-3.5.1/tree/master) ，其他一个【vmp】及【vmpsrc】是早前其他 leak 缺省版。

主要还是 git 上的 vmprotect351 与 vmpsrc 的差别。三方比较通过 UltraCompare 进行。

### （1）mac 平台文件

    vmpsrc 独有的 Info.plist 为 mac 平台的相关信息  

![](https://bbs.kanxue.com/upload/attach/202402/760675_KHJGPJ75D8WC2U9.webp)

### （2）dll 与 inc

    vmpsrc 多了许多 NET 相关的 dll，初步估计对应于 inc，本文测试未考究

![](https://bbs.kanxue.com/upload/attach/202402/760675_4WYNX2RBEBJQPXF.webp)![](https://bbs.kanxue.com/upload/attach/202402/760675_DPHXNSM7D79CD5V.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_PM5MQYUEDFSPX32.webp)  

### （3）核心 intel.cc 及 processors.cc

        相对早前 leak 版本，vmprotect351 具有缺失的核心文件 intel.cc、processors.cc，部分代码声明与定义有出入，需要修正，参见后述。

        下图中，vmpprotect351 缺失的 lang*.inc\.tmp 等由 lang.bat 生成，缺失的 version.h 由 version.bat 生成。  

![](https://bbs.kanxue.com/upload/attach/202402/760675_S4KMHY5HQ88QRVQ.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_3SJYUHJGMWAXP3Z.webp)

### （4）脚本测试样例

    缺失的 *.exe 应该是用于对应脚本功能的 vmp 防护测试实例，相关 vmp 防护配置 *.vmp 还保留

![](https://bbs.kanxue.com/upload/attach/202402/760675_PY3TRAYYVVJTVQ7.webp)  

### （5）Qt 的 moc 文件

    缺失的 MOC 文件会由 Qt 编译工具的 moc.exe 对相应头文件编译产生

![](https://bbs.kanxue.com/upload/attach/202402/760675_JGT4UZ4PX9ATMXP.webp)

### (6) 其他

![](https://bbs.kanxue.com/upload/attach/202402/760675_Y4QXAYT3A3T8PRN.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_3FVCT8UD2MHRHJK.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_FFGDF4VD9UV9ZAP.webp)

第 2 节 VMPProtectCon 项目工程编译
--------------------------

    即 console 控制台版本  

### （1）类的非重载成员函数错误  

    这是由于【IntelFunction 类中 Mutate 成员函数】及【IntelObfuscation 类中 Compile 成员函数】的声明与定义参数不一致产生，  

    在定义中添加完整的参数定义即可，如下述图示  

![](https://bbs.kanxue.com/upload/attach/202402/760675_YCC35D7J287VUZ7.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_PAWTEWVPDKJZF5G.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_FX3YHADFQT3FZD3.webp)

### （2）缺失 lib 错误

    修正编译后，会出现上图中的 VMProtectSDK64.lib 链接缺失错误，这时候需要我们完成 VMProtectSDK（visual studio 2008) 的 64 位编译。

    这里没有改动原来编译工具配置，所以需要安装 vc2008 的 64 位编译工具。  

可以下载 VS2008 的相关 ISO 安装镜像，要注意，如果默认安装，其可能会忽略我们需要的 64 位编译工具，如图，选中 64 位工具安装，

![](https://bbs.kanxue.com/upload/attach/202402/760675_CV7ZFBMSHANG8Q7.webp)

### （3）con 版本

    直接编译 VMProtectSDK 即可得到 VMProtectSDK64.lib，再编译 VMPProtectCon 工程即可完成。

    如果不使用 GUI 版本，到此即可，当然，还需要提供测试用的注册配置，否则无法正常使用和对其他程序进行编译保护。  

相关注册配置参考后续内容。

第 3 节 VMProtect 项目工程编译
----------------------

    即 Qt 的 GUI 版本  

### （1）Qt 缺失错误

    直接编译，若没有 Qt 配置环境，直接编译或有下列错误。

![](https://bbs.kanxue.com/upload/attach/202402/760675_22KMNA7T53P4V4C.webp)

### （2）Qt 共享库版本

        需要 [https://download.qt.io/new_archive/qt/5.6/5.6.0/](https://download.qt.io/new_archive/qt/5.6/5.6.0/) 中下载其中的 msvc2015_64 版本。

        特别注意，如果安装默认路径 C:\Qt，后续不需要改项目工程配置，否则需要做相应工程配置修改，  

        这里安装非默认路径，相应工程配置修改参考后续说明。  

![](https://bbs.kanxue.com/upload/attach/202402/760675_QR4ZY2993ZUY9NU.webp)

### （3）关键环境变量配置

    从 VC 项目工程以及相关用户宏属性配置可知，原配置用的是 Qt5.6.0 版本，QIDIR 配置路径是默认的 C:\Qt\Qt5.6.0\5.6

![](https://bbs.kanxue.com/upload/attach/202402/760675_D3UGBUX6D7EAM3H.webp)

![](https://bbs.kanxue.com/upload/attach/202402/760675_D7PN8BCT97UPCHY.webp)  

### (4) 命令行启动变量 PATH 及 QTDIR

    若 Qt 不是 5.6.0 版本或不是默认路径，则需要在属性管理中作出相应修改，如下图所示。  

    若要调试，最好在启动新命令行，在命令行中设置 PATH，并在命令行中启动 VS2015，这样，调试启动时，会自动找到 Qt 相关动态库，否则启动调试会出错。

    运行 GUI 程序也同理，需要在 PATH 中找到 Qt 相关动态库  

```
set path=D:\Qt\Qt5.6.0_x64\5.6\msvc2015_64\bin;%path%               PATH中添加Qt动态库路径
"C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Visual Studio 2015.lnk"  启动VS2015
 
 
D:\vmprotect-3.5.1\bin\64\Debug\VMProtect.exe    执行编译得到的GUI版vmp

```

![](https://bbs.kanxue.com/upload/attach/202402/760675_8PJE88S4ZUCKMYT.webp)

第 4 节 版本管理与注册配置
---------------

### （1）版本管理

    其版本号是由 D:\vmprotect-3.5.1\core\version.bat 批文件管理的，

    如下图，我们将原来的 1.0.0.0 build 0 修改位 【3.5.1 build 1024】，重新编译（非增量编译）生效  

![](https://bbs.kanxue.com/upload/attach/202402/760675_FHUNFGB9JK39BDE.webp)

### （2）注册配置

    根据 VMP 源码，要让源码版 VMP 正常工具，需要在编译结果 exe 同目录下提供注册配置信息  

    以下是根据 VMP 源码拟出的一种可用测试配置，保存为【VMProtectLicense.ini】文件，与 VMProtect.exe（或 VMProtectCon.exe）同目录。

```
;VMProtectLicense.ini
[TestLicense]
AcceptedSerialNumber=SerialNumber
BlackListedSerialNumber=BlkSerialNumber
TimeLimit=-1
ExpDate=20481024
MaxBuildDate=20481024
KeyHWID=ABC
MyHWID=ABC
UserName=0day
EMail=0day@xday.com
;UserData[0] as productId, should in 
;  00 VPI_NOT_SPECIFIED
;  05 VPI_ULTM_PERSONAL = VPI_ULTM_WIN_PERSONAL
;  06 VPI_ULTM_COMPANY = VPI_ULTM_WIN_COMPANY
UserData=05

```

![](https://bbs.kanxue.com/upload/attach/202402/760675_8WFRE9NRKK4J22C.webp)

【附件】
----

    [vmprotect-3.5.1-master.part01.rar](https://bbs.kanxue.com/upload/attach/202402/760675_26PS4T5X25YYCR9.rar) 和 [vmprotect-3.5.1-master.part02.rar](https://bbs.kanxue.com/upload/attach/202402/760675_NAQ9AKX5N8Z98VG.rar) 为 git 上的代码，7M 分包。

    [VMProtectLicense.ini](https://bbs.kanxue.com/upload/attach/202402/760675_ANYNC72WQ2425XB._ini) 为注册测试使用配置。

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#开源分享](forum-41-1-134.htm) [#基础知识](forum-41-1-130.htm)

上传的附件：

*   [vmprotect-3.5.1-master.part01.rar](javascript:void(0);) （7.00MB，23 次下载）
*   [vmprotect-3.5.1-master.part02.rar](javascript:void(0);) （5.78MB，23 次下载）
*   [VMProtectLicense.ini](javascript:void(0);) （0.38kb，23 次下载）