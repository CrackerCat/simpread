> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268415.htm)

> [原创][讨论] 安卓 ro.debuggable 属性的设置（安卓 6.0.1 - 10[11]）

目前看到的一些设置只读属性的工具基本都不能完全正常工作，magisk 或许有，但是安装 magisk 并不在我的选项内。

 

相关主题：[https://bbs.pediy.com/thread-267980.htm](https://bbs.pediy.com/thread-267980.htm)

 

兼容各个版本起来挺烦人的，但是我又没有什么太好的方法，在看雪看到过用 ptrace，反正那个工具我无法在我的机器上正常运行，而且，他还不开源，逆了一下，没什么可说的。magisk resetprop，安装这个会成为一个影响因素不会考虑。

 

github 链接包含**源码**以及**使用教程**，这是个 all-in-one 工具，但是这个功能值得单独拿出来开一个主题。

 

[https://github.com/rev1si0n/bxxt](https://github.com/rev1si0n/bxxt)

 

如果不想自己编译可以下载编译好的  
全平台 (android arm/64, x86/64) 下载链接：  
[https://github.com/rev1si0n/bxxt-binaries/raw/master/bxxt.zip](https://github.com/rev1si0n/bxxt-binaries/raw/master/bxxt.zip)

 

已经经过手边一加 5 10.0 系统，以及 nubia 6.0 系统的测试，以及安卓官方模拟器的测试，work as what i expected.

 

如果你有更好的实现方法欢迎一起讨论。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#逆向分析](forum-161-1-118.htm) [#系统相关](forum-161-1-126.htm) [#工具脚本](forum-161-1-128.htm)