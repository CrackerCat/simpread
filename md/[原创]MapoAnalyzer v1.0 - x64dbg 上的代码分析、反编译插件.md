> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268502.htm)

> [原创]MapoAnalyzer v1.0 - x64dbg 上的代码分析、反编译插件

让 x64dbg 拥有和 IDA 一样的函数识别，反编译功能。

 

**函数识别**  
在 x64dbg 载入要调试的程序后会自动分析出相关的函数信息，带有 delphi7、delphi10、lazrus、mingw、rust、vc6- vs2019，易语言的函数特征库，可以快速识别出相应的库函数。  
![](https://bbs.pediy.com/upload/tmp/301937_2KPRH59RVNFGR67.jpg)

 

**字符串识别**  
字符串识别，支持中文，可以快速根据字符串定位到关键函数。  
![](https://bbs.pediy.com/upload/tmp/301937_WSWDZ8FB3APMVWH.jpg)

 

**反编译**  
双击函数、字符串可以对函数代码进行反编译，以伪 C 代码显示，核心引擎基于 Ghidra，下面是和 IDA 的 F5 对比图。  
![](https://bbs.pediy.com/upload/tmp/301937_SWF65BHRDQHX8VA.jpg)

 

**中文支持**  
IDA 和原版 Ghidra 对中文的支持都不好，无法正常显示中文，这里对中文做了特别支持。  
![](https://bbs.pediy.com/upload/tmp/301937_R9AGJWUDV7J9V3A.jpg)

 

**将分析结果应用到 x64dbg 的反汇编页面**  
默认不会将函数识别结果应用到 x64dbg 的反汇编中，需要手动点击 右键 - MapoAnanlyzer-Analyze，应用后会将识别的函数以标签、注释的方式添加到反汇编窗口中，方便动态调试时识别函数。

 

分析前  
![](https://bbs.pediy.com/upload/tmp/301937_TCKH8R8VA3BHBK5.jpg)  
分析后  
![](https://bbs.pediy.com/upload/tmp/301937_HFC7M2VV8BWGWX4.jpg)

 

**使用方法：**  
将压缩包里的 x32、x64 文件夹覆盖掉 x64dbg 里对应的 x32、x64 文件夹即可。

 

此插件为从我们的加密产品 [MapoEngine](http://maposafe.com/index/index/mapoengine.html) 里分离出来的功能，是我们做加密产品过程中产生的一个副产品。

 

下载附件大，分了 5 个包  
网盘  
https://pan.baidu.com/s/1LKgYowUCez2I6VeCDXjc5w  
3qut

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

[#调试逆向](forum-4-1-1.htm) [#系统底层](forum-4-1-2.htm) [#软件保护](forum-4-1-3.htm) [#VM 保护](forum-4-1-4.htm)