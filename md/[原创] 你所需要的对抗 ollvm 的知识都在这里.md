> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272414.htm)

> [原创] 你所需要的对抗 ollvm 的知识都在这里

整理了一些 ollvm 对抗的文章，大家可以参考。  
ollvm 还原主要有静态分析的方法和动态分析的方法。无论哪种方法基本流程都是：找到所有基本块 (特征匹配)- 确定真实块之间的关联 (静态的：符号执行 / 反编译器提供的 IL 的 API；动态的：模拟执行 / IDA trace)-patch 原程序

基于符号执行
======

angr
----

利用符号执行去除控制流平坦化: [https://security.tencent.com/index.php/blog/msg/112](https://security.tencent.com/index.php/blog/msg/112)  
利用 angr 符号执行去除虚假控制流: [https://bbs.pediy.com/thread-266005.htm](https://bbs.pediy.com/thread-266005.htm)  
TetCTF2022 一道代码混淆题分析——crackme_pls: [https://bbs.pediy.com/thread-271164.htm](https://bbs.pediy.com/thread-271164.htm)  
Angr Control Flow Deobfuscation: [https://research.openanalysis.net/angr/symbolic%20execution/deobfuscation/research/2022/03/26/angr_notes.html](https://research.openanalysis.net/angr/symbolic%20execution/deobfuscation/research/2022/03/26/angr_notes.html)

miasm
-----

Deobfuscation: recovering an OLLVM-protected program:  
[https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html)  
我印象中 quarkslab 这篇文章是最早的关于去 ollvm 混淆的文章，有点老了，不过还是值得学习。  
MODeflattener - Miasm's OLLVM Deflattener: [https://mrt4ntr4.github.io/MODeflattener/](https://mrt4ntr4.github.io/MODeflattener/)

基于反编译器提供的 IL 的 API
==================

IDA 的 microcode
---------------

[https://github.com/RolfRolles/HexRaysDeob](https://github.com/RolfRolles/HexRaysDeob)  
[https://github.com/idapython/pyhexraysdeob](https://github.com/idapython/pyhexraysdeob)  
相关文章：  
[https://hex-rays.com/blog/hex-rays-microcode-api-vs-obfuscating-compiler/](https://hex-rays.com/blog/hex-rays-microcode-api-vs-obfuscating-compiler/)  
[https://www.virusbulletin.com/uploads/pdf/conference_slides/2019/VB2019-Haruyama.pdf](https://www.virusbulletin.com/uploads/pdf/conference_slides/2019/VB2019-Haruyama.pdf)

binary ninja
------------

Dissecting LLVM Obfuscator Part 1: [https://rpis.ec/blog/dissection-llvm-obfuscator-p1/](https://rpis.ec/blog/dissection-llvm-obfuscator-p1/)  
使用 Binary Ninja 去除 ollvm 流程平坦混淆: [https://bbs.pediy.com/thread-256299.htm](https://bbs.pediy.com/thread-256299.htm)

ghidra 的 p-code
---------------

使用 Ghidra P-Code 对 OLLVM 控制流平坦化进行反混淆: [http://galaxylab.com.cn/%e4%bd%bf%e7%94%a8ghidra-p-code%e5%af%b9ollvm%e6%8e%a7%e5%88%b6%e6%b5%81%e5%b9%b3%e5%9d%a6%e5%8c%96%e8%bf%9b%e8%a1%8c%e5%8f%8d%e6%b7%b7%e6%b7%86/](http://galaxylab.com.cn/%e4%bd%bf%e7%94%a8ghidra-p-code%e5%af%b9ollvm%e6%8e%a7%e5%88%b6%e6%b5%81%e5%b9%b3%e5%9d%a6%e5%8c%96%e8%bf%9b%e8%a1%8c%e5%8f%8d%e6%b7%b7%e6%b7%86/)

基于模拟执行
======

基于 Unicorn 的 ARM64 OLLVM 反混淆: [https://bbs.pediy.com/thread-252321-1.htm](https://bbs.pediy.com/thread-252321-1.htm)  
ARM64 OLLVM 反混淆（续: [https://bbs.pediy.com/thread-253533.htm](https://bbs.pediy.com/thread-253533.htm)  
细说 arm 反类 ollvm 混淆 - 基本思想: [https://bbs.pediy.com/thread-257878.htm](https://bbs.pediy.com/thread-257878.htm)  
这篇文章介绍了模拟执行和 IDA trace 两种方法。

其他方法
====

这位是通过编译后端优化干掉混淆，这已经超过我的理解能力了...  
一种通过后端编译优化脱混淆壳的方法: [https://bbs.pediy.com/thread-260626-1.htm](https://bbs.pediy.com/thread-260626-1.htm)  
一种通过后端编译优化脱虚拟机壳的方法: [https://bbs.pediy.com/thread-266014.htm](https://bbs.pediy.com/thread-266014.htm)

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

[#混淆加固](forum-161-1-121.htm)