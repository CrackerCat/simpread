> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267840.htm)

> mona 之 Win10_Windbg_X64 安装

安装方法如下 (x64)：

[](#系统环境：)系统环境：
===============

操作系统：Windows 10 Version 19042 MP (4 procs) Free x64  
Windbg: ![](https://bbs.pediy.com/upload/attach/202105/919672_2V5M79364JJ75AN.png)  
Python:2.7.18(x64)  
下载链接：https://www.python.org/ftp/python/2.7.18/python-2.7.18.amd64.msi  
**注意**：自行设置环境变量，太细节的操作就不赘述了，保持言简意赅。

[](#mona环境：)mona 环境：
====================

**mona**:[https://github.com/corelan/mona](https://github.com/corelan/mona)  
**windbglib:**[https://github.com/corelan/windbglib](https://github.com/corelan/windbglib)  
**pykd.dll:**[https://githomelab.ru/pykd/pykd/-/wikis/0.3.4.15](https://githomelab.ru/pykd/pykd/-/wikis/0.3.4.15)  
![](https://bbs.pediy.com/upload/attach/202105/919672_6KNTBMZJRK85PMY.png)

[](#操作步骤：)操作步骤：
===============

1. 注册：(Admin 权限)  
命令：regsvr32 msdia140.dll  
2. 拷贝：  
mona.py===>C:\Program Files\Windows Kits\10\Debuggers\x64  
windbglib===> 同上  
pykd.dll===>pykd.pyd 修改为 “pykd.dll”,C:\Program Files\Windows Kits\10\Debuggers\x64\winext  
![](https://bbs.pediy.com/upload/attach/202105/919672_XJ6HZH9A3DM6AFQ.png)  
![](https://bbs.pediy.com/upload/attach/202105/919672_FTXE72F22WYU4YY.png)  
3. 命令：  
.load pykd  
!py mona.py  
![](https://bbs.pediy.com/upload/attach/202105/919672_RBBXYTCFC7QVJF5.png)  
4. 实验：  
!py mona modules  
![](https://bbs.pediy.com/upload/attach/202105/919672_P6FW3K2XNJQWBXF.png)  
总结：  
总体来说还可以，就是最新版本 pykd 没被 mona 官方测试，稳定情况不明。可见：[https://github.com/corelan/windbglib/issues/23](https://github.com/corelan/windbglib/issues/23)  
![](https://bbs.pediy.com/upload/attach/202105/919672_QM5RZSBNCZ73FTE.png)

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 1 小时前 被 mJqalJqN 编辑 ，原因：