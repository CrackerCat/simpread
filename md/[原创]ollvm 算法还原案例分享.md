> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268163.htm)

> [原创]ollvm 算法还原案例分享

看雪 3W 班 12 月 ollvm 题，重点考察去混淆

[](#解题思路：)解题思路：
===============

得益于 Unicorn 的强大的指令 trace 能力，可以很容易实现对 cpu 执行的每一条汇编指令的跟踪，进而对 ollvm 保护的函数进行剪枝，去掉虚假块，大大提高逆向分析效率

[](#通过对题目的考察，学习到如下知识点：)通过对题目的考察，学习到如下知识点：
=========================================

1.  使用 unidbg 获取 trace 日志
2.  ida python 常用函数的复习，遍历函数指令，并做修改
3.  利用常量表识别 base64 编码

详细步骤
====

1 apk 拖入 jadx，获取基础信息
====================

com.kanxue.crackme  
com.kanxue.crackme.MainActivity  
public boolean check(String content) {  
if (jnicheck(content)) {  
return true;  
}  
return false;  
}

 

System.loadLibrary("native-lib");  
public native boolean jnicheck(String str);  
public static native boolean crypt2(String str);

2. 利用 frida 去 hook 关键函数，同时输入字符，点按钮验证，观察日志后发现：
=============================================

jnicheck 是我们的输入、crypt2 是输入字符串 + 字符串 666  
![](https://bbs.pediy.com/upload/attach/202012/717171_E4FS8J489S447DB.png)

3. ida 分析 so，直接跟踪到关键函数 sub_22390、sub_19a60；发现都是 ollvm 混淆的，没法看
=============================================================

![](https://bbs.pediy.com/upload/attach/202012/717171_YRN7U3SCHBM86XJ.png)  
![](https://bbs.pediy.com/upload/attach/202012/717171_A3JMU7GHVNH4ME8.png)

4. 用 unidbg 跑关键的 crypt2 函数，并拿到 trace 日志，其中重要的是地址
================================================

![](https://bbs.pediy.com/upload/attach/202012/717171_TCFWGT2DAFNCKGF.png)

5. 根据题目提示信息——利用 trace 日志进行剪枝，查了下 IDAPython API，编写剪枝代码，对 2 个函数剪枝。发现关键函数 sub_19a60 就是 base64、sub_22390 就是很简单的调用。
==============================================================================================================

![](https://bbs.pediy.com/upload/attach/202012/717171_U8K6R3QVEH4GWGS.png)  
![](https://bbs.pediy.com/upload/attach/202012/717171_6VPH2TW4F68PZF9.png)

6. frida 直接看 strcmp 函数参数：
=========================

sub_strcmp: WFVlNjY2 WFVlNjY2  
对参数 0 解 base64=XUe666，再去掉 666。明文就是 XUe。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

上传的附件：

*   [3.apk](javascript:void(0)) （1.59MB，2 次下载）
*   [3、readme.txt](javascript:void(0)) （0.24kb，2 次下载）