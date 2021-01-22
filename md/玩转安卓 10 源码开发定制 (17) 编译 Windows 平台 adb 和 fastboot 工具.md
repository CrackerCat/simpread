> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484205&idx=1&sn=3ab53e36478bdcd5ad7ed2b2bd88106c&scene=21#wechat_redirect)

  

     在安卓系统源码中，我们可以通过在 Ubuntu 中安装 mingw 64(32 位不讨论了，现在都流行 64 位的) 提供交叉编译 Windows 平台运行的程序。下图中展示的 windows 中安卓 sdk tools 一些常用的工具命令都可以 ubuntu 安卓源码中编译:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430UfruuMX2pibtk8VSjibG7ibqyDvib4xQfRLCTNKIrAy1UEsRAniaGOFDbTzhWxHdP2nN1eBr4v3O01ibw/640?wx_fmt=png)

一、mingw 简要说明
------------

MinGW 的全称是：Minimalist GNU on Windows 。它将开源 C 语言编译器 GCC 移植到了 Windows 平台下，并且包含了 Win32API, 因此可以将源代码在 Linux 系统中编译为可在 Windows 中运行的可执行程序。而且还可以使用一些 Windows 不具备的, Linux 平台下的开发工具。

二、编译 Windows 安卓 sdk platform-tools 工具
-------------------------------------

### 1、配置安装 mingw-w64

执行如下命令安装 mingw-w64

```
sudo apt-get install mingw-w64



```

### 2. 源码环境初始化

源码根目录执行如下命令初始化编译命令:

```
qiang@ubuntu:~/lineageOs$ source build/envsetup.sh


```

### 3. 命令编译

编译 adb.exe 执行如下:

```
qiang@ubuntu:~/lineageOs$ make Use_MINGW=y adb


```

编译 fastboot.exe 执行如下:

```
qiang@ubuntu:~/lineageOs$ make Use_MINGW=y fastboot


```

### 4. 编译输出展示

编译完成后 adb 和 fastboot 保存如下路径:

```
/home/qiang/lineageOs/out/host/windows-x86


```

如下所示我个人编译的出来的:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430UfruuMX2pibtk8VSjibG7ibqd5xibtxKRHfC6waeLE04spK5SFj0QdkbdGO5ATnsefZV7C6cqEzuWOw/640?wx_fmt=png)

Windows 平台中经常遇到 adb 5037 的端口被占用的情况, 可以通过改掉 adb 源码中的默认端口 5037, 打造属于自己专用机的 adb 工具。

上一篇[玩转 Android10 源码开发定制 (17) 开发并内置具有系统权限 (system) 的 App](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484144&idx=1&sn=5b45a35ec9f164ffea23b5869a4a888f&scene=21#wechat_redirect)

如果对玩转安卓 10 系列文章感兴趣，可以进入公众号菜单 "文章列表" 中点击查看相关文章。  

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430UfruuMX2pibtk8VSjibG7ibqL69LPpFCu4FLWhDI8MOPhDicCt0K1OjVksD1ADuso3kbPf4YibBAaPVw/640?wx_fmt=jpeg)扫一扫关注公众号