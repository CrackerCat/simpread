> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267980.htm)

> [推荐] 一个自用安卓 BOOT.IMG 修改工具 (已开源)

https://github.com/rev1si0n/bxxt

 

如果你会接触到 boot.img 的话，那么这个工具可能会对你有帮助。因为当初需要研究 supersu 之类的相关原理，  
而网上关于 boot.img 解包打包等操作的文章及工具实在太杂乱了，挺费时间，所以我抱着顺便理解的心态编写了此工具。

 

当然，第一版主要是 python 来实现的，功能也仅限于 ramdisk 的解包打包，但是随着后来遇到了更多的问题及需要，  
改用 c 编写了目前仅可在安卓设备上运行的二进制程序。功能也增加到了基本可以自动处理 boot.img 中的任何对象。

 

原本，如果你需要对内核进行 patch，  
你需要，解包 boot.img，解压内核，修改，压缩内核，打包 boot.img 来完成这一系列动作  
但是这个工具，可以让你缩减到只用三个动作  
解包 boot.img，修改，打包 boot.img

 

作为一个过来人，我想说，光解压和压缩内核这两个动作，就有可能让新手因为某些原因或玄学问题让你刷入后无法开机。

 

有了这个，你可以很方便的修改内核 / 命令行参数，ramdisk，甚至镜像中的设备树。当然作为附加功能，也支持 selinux 规则的注入（感谢 sepolicy-inject）

 

不过它也有缺点，它只能在设备上运行（android-arm/64，这也意味着你只能在安卓 adb shell 用 sed 之类的工具编辑)，只支持 android 6.0 - 10 的启动镜像 (因为自从 11 改变了很多东西，可能会支持一小部分 android 11 的镜像，当然也不支持 6.0 以下的，因为它们太老了）。  
(注意请不要尝试将解包的内容 adb pull 到 windows 主机，你将会丢失 ramdisk 的文件权限)

 

Github: https://github.com/rev1si0n/bxxt

 

当然，它目前也没有发布源码仅有二进制文件，一是我要把它从依赖三方和私有模块甚至 Makefile 的项目单独提取出来挺麻烦，二是我写的挺烂而且 BUG 应该多的不行。不过完善以后肯定会发布。  
使用方法都在 README 当中。

 

基础使用演示（asciinema ）  
https://asciinema.org/a/HcZC215U9CIViz7w5oXThOCgL

 

新人第一次发帖，多多关照。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 24 分钟前 被 RiDiN 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#工具脚本](forum-161-1-128.htm)