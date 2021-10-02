> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269601.htm)

> [原创] 静态 InlineHook 的脚本实现

##### 前言

前文说到使用[基于 LIEF 的 InlineHook 实现](https://www.jianshu.com/p/f53d6ad0e728) ，在这里我们再借助 [keystone](https://github.com/keystone-engine/keystone) 和 [capstone](https://github.com/aquynh/capstone) 来完善一下这个想法，解决一些比较枯燥且容易出错的事，比如 地址偏移的计算，指令备份还原 ...

##### 简介

1.  合并代码的操作和前文基本一致，不过这里我们记录四个地址  
    这四个地址的代码都是全部使用 asm(nop) 填充的空白函数，并放置于. inject 节区，合并的时候只合并该节（之前在 libil2cpp.so 添加导出函数会导致崩溃，后来索性就直接拿一个字典来存放）

记录下用到的四个表，以及要 hook 的函数  
![](https://bbs.pediy.com/upload/attach/202109/868525_8CG4YK6Q2EZ35E5.png)

*   GLOBAL_TABLE 用于存放我们需要初始化的一些值，可以理解我们自己写代码的. bss/.data ... 等等的集合（由于这里的这个. inject 段属于代码段，没有写权限，建议需要写直接修改 elf 将整个 loadable Segment 改为 rwx）
*   STR_TABLE 由于用到字符串比较多，然后我就单独列出来了一个字符串表
*   trampolines 跳板代码存放位置（1. 环境的保存 2. 跳转到 textCodes 3.textCodes 返回时跳转回原代码）
*   textCodes 真实执行的代码位置 （hook 代码存放位置）

hook 操作  
![](https://bbs.pediy.com/upload/attach/202109/868525_TFWYSV6C7DYAPW8.png)

1.  ins.addPtr(100) 是在往 GLOBAL_TABLE 添加一项
2.  ins.getStr("this is a test string!") 是再往 STR_TABLE 添加一项，同时也会添加到 GLOBAL_TABLE
3.  ins.addHook hook 指定的地址并将上下文的状态保存好（trampolines），跳转到 textCodes 同时 setPC 到这个位置，这个位置我们就可以写自己的汇编代码逻辑
4.  ins.endHook() 结束一个 hook，主要是从 textCodes 跳回到 trampolines
5.  ins.android_log_print(msg="called this function") 编写的汇编代码调用 log 的 demo ，这里后续可以去拓展到 strcmp strcat 再或者是一些我们常用到的一些其他函数

我这么描述可能还是不直观，直接上 IDA 看看我们效果

 

原来的样子  
![](https://bbs.pediy.com/upload/attach/202109/868525_25BSMBGNFTAHX34.png)

 

hook 之后使用三条指令跳转到 trampolines  
![](https://bbs.pediy.com/upload/attach/202109/868525_UEVCXXCUXDRFVBB.png)

 

进入到 trampolines  
![](https://bbs.pediy.com/upload/attach/202109/868525_CBBNT84S5MG3W9W.png)

 

进入到 textCodes  
![](https://bbs.pediy.com/upload/attach/202109/868525_46XYWCKRB39PJAF.png)

 

GLOBAL_TABLE  
![](https://bbs.pediy.com/upload/attach/202109/868525_Y58MQ6D56W63J86.png)

 

STR_TABLE  
![](https://bbs.pediy.com/upload/attach/202109/868525_JT3DM5UBHU8PAWZ.png)

##### 总结

1.  继[（基于 LIEF 的 InlineHook 实现）](https://www.jianshu.com/p/f53d6ad0e728)的想法落实。当时是还觉得没啥用，毕竟手动去计算偏移修改容易出错，没有效率就没有生产力
2.  虽然目前这个版本只是个初版，一个简单的想法的实现，但是在满足工作需求的情况下一定程度的脱离 [Dobby](https://github.com/jmpews/Dobby) 的使用
3.  还有很多需要完善的，比如 dobby 最核心的指令修复（现在就只能选点不需要修复的指令来 hook emmmmm....）
4.  如果要写文件记得手动去 010 修改一下 loadable 段属性
5.  设计初衷是尽量将一些可能用到的功能模块化以提升汇编的可读性，以及修改的便利性
6.  [欢迎大家一起完善](https://github.com/axhlzy/PyAsmPatch)，争取搞一个类似动态 hook 框架 的 静态 hook 框架
    
    [手动狗头].png
    

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 2 天前 被唱过阡陌编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)