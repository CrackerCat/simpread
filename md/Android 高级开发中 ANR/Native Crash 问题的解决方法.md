> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484032&idx=1&sn=f6983ee2d8f0a9108785bcc8ae5d81e6&chksm=cebb47cef9ccced8ce805157107f9601d73ec8ba951816e776fa3e1feef7977f5a70bea441b8&scene=21#wechat_redirect)

Android ANR 的定位和分析方法

Android Native Crash 的定位和分析方法

Android ANR 的避免和检测方法

Android Native Crash 捕获方法

Android App ANR 原理（基于 O）

Android-ANR 总结原理分析

NativeCrash 分析 (一)-NativeCrash 原理

**Android ANR**

Application Not Responding：即应用无响应

如果应用程序主线程在超时时间内对输入事件没有处理完毕, 或者对特定操作没有执行完毕, 就会出现 ANR

(主线程在特定的时间内没有做完特定的事情)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF0fpwVNMI1IrqibibcttYBtteyVKr6kYf8AwgoicVtTQOTkKd6IeeE7cPCbibmwnznXDVs7xT0lDzwnHQ/640?wx_fmt=png)

**Android ANR 或 Crash 的定位、检测和避免**

**Android Native Crash**

Native 程序是指可以直接运行在操作系统上，并且处理器直接执行机器码的程序，比如 “/system/bin” “/system/lib” 目录下的文件，这些应用程序都是由 GCC(c/c++) 编译生成，这些程序的崩溃统称为 Native Exception，比如空指针，非法指针，程序跑飞，内存踩坏等。

Native Crash 都是进程收到信号引起的.

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF0fpwVNMI1IrqibibcttYBtteO0wznkV2DGCicC1j4Llicme6wibHeic7FLcTBg3EQ2XVRAxAP2KLBfYrZw/640?wx_fmt=png)

**Android Native Crash 捕获**

Android App ANR 原理（基于 O）

https://blog.csdn.net/TaylorPotter/article/details/81432522

Android-ANR 总结原理分析

https://blog.csdn.net/fanxudonggreat/article/details/81840791

NativeCrash 分析 (一)-NativeCrash 原理

https://blog.csdn.net/TaylorPotter/article/details/103779294

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF0zQWAYibuUc6ZPBfGbZbxlyZLTRCazQibcMzOtOwWxuVyWkEUN7YdiclwWc7UaFVTwS3JfMC2qWXCMg/640?wx_fmt=jpeg)