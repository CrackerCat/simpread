> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283237.htm)

> [原创] 拳打脚踢系列之棒子的游戏——记一次探索做大哥的原理从 0 到 1

前言
==

那天酒足饭饱，夜不能寐，在日韩区看的津津有味，索性动身打开电脑，来一场酣畅淋漓的文章。废话不多说，欢迎大家来到日韩专区，这次的主角是 **NHN AppGuard** ，主角的特征是 **libdiresu.so libloader.so**  
 

游戏启动
====

主角圆润的弹窗，不得不说很标特否，粗略扫一眼，完蛋居然被检测到我那藏起来的根（Rooting）  
![](https://bbs.kanxue.com/upload/tmp/807696_SXYR94Z44ZYBMKU.webp)

![](https://bbs.kanxue.com/upload/tmp/807696_8F5GAMU4WGS6ZQN.webp)  
直接给我干闪退，网上说日韩很温柔，都是骗人的！！！幸好我打印了 so 加载，不得不让我怀疑 **libloader.so**，现在目标明确，先干它  
 

libloader
=========

现在就让我们用 IDA 好好蹂躏它吧，可以看到. init_array 有很多个函数，只有第一个函数 **sub_1718D4** 被 IDA 识别到，嗦嘎！！！明显是第一个函数解密了后面的函数，这里有个执行顺序的问题，如果 JNI_Onload 是加密的，那么就往 init_array，init_proc 去分析，假如都是加密的，那没得说了  
![](https://bbs.kanxue.com/upload/tmp/807696_KH6YZKCRXJYNW8C.webp)

**这个壳的解密过程就不细说了，感兴趣可以去看下乐佬写的文章，看的我意犹未尽，从壳的加载解密到闪退等等，说的很详细，点赞**  
[乐佬的文章](https://bbs.kanxue.com/thread-278113.htm)

当看完乐佬的文章，你已经可以进入这次的主角，距离成功只差临门一脚。  
 

弹窗反推
====

分析 Security Warning 弹窗，从弹窗的样式以 so 加载，可以看出是 Java 层触发的  
![](https://bbs.kanxue.com/upload/tmp/807696_NC6A3EM49KBBU2W.webp)  
通过 Frida 对 Android Dialog.show 打堆栈，可以定位到 com.siem.ms7.DetectionPopup  
![](https://bbs.kanxue.com/upload/tmp/807696_P4TEXG5ETZ2THP4.webp)  
代码中存在很多隐晦难理解的字符串，先不管它，直接找 create 的地方，什么鬼，居然没调用  
![](https://bbs.kanxue.com/upload/tmp/807696_UDVTHQ7PW9N6S2F.webp)  
别慌，从堆栈打印中看到了 native，得知这是从 native 反射调用滴，嗦嘎！！！  
![](https://bbs.kanxue.com/upload/tmp/807696_UZWHA3ANXM65Z9S.webp)  
当我们打印 art 的所有 JNI 函数，在 NewStringUtf 发现令人激动的一幕  
![](https://bbs.kanxue.com/upload/tmp/807696_R5F5JA5VWCS5ZBB.webp)  
图上所示，这堆栈怎么没有 so 名称和偏移啊！！！！不会是自定义 linker 和开辟了一块匿名内存做检测吧？没事，我们先根据堆栈地址，从 maps 能不能找到一些有用的信息（正常逻辑是要分析 libloader.so 对 Engine 的加载）  
![](https://bbs.kanxue.com/upload/tmp/807696_Z239A6SKZYFRXEF.webp)  
通过对 maps 分析，找到堆栈位置在 / data/data/com.mobirix.mbpdh/.dtam145zau/fkr5gbebm3 (deleted) 里面，其中 (deleted) 符合堆栈没有 so 名字与符号的情况，不用看，直接 Hook remove  
![](https://bbs.kanxue.com/upload/attach/202409/807696_P86TES7QYJEHFFQ.webp)  
可以看到 remove 了加载的 so 地址，我们阻止 remove 将 so 拷贝出来 (暂且叫它 “Engine”)，此时打印堆栈就能看到对应的 so 名字  
![](https://bbs.kanxue.com/upload/attach/202409/807696_E4EYR5XTKW2QC4M.webp)  
 

Engine so 分析
============

除了导出函数混淆以外，代码段没加密（奇怪，我分析那会，是对代码段有加密的，现在没加密了）  
![](https://bbs.kanxue.com/upload/attach/202409/807696_A2EFA7KU5J7T2PX.webp)  
跳到 0x706391c34c iuyz5u972r!0xa834c 处，调用了 JNI NewStringUTF  
![](https://bbs.kanxue.com/upload/attach/202409/807696_C275JGFYN8W4WT8.webp)  
打印堆栈，最终找出触发点，看出有 libc 的 kill 和 syscall exit，把 a2 的值改成 0 绕过检测  
![](https://bbs.kanxue.com/upload/attach/202409/807696_CPS7MYNCHAJD8QC.webp)

检测点，就不一一说了
----------

![](https://bbs.kanxue.com/upload/attach/202409/807696_UDTXFMU6H3R4UKZ.webp)  
![](https://bbs.kanxue.com/upload/attach/202409/807696_A7YA29TFCPCQ944.webp)  
现在 frida 可以无忧无虑调式了  
 

il2cpp
======

好家伙，init_array 和导出函数加密了，也没见 start 与 .init_proc 函数，基本确定是通过其他 so 去解密的（elf 结构没啥问题）  
![](https://bbs.kanxue.com/upload/attach/202409/807696_7KJH43XAEWX9XEF.webp)  
直接揭晓，解密由 libloader.so 操作，过程有点复杂就不贴出来了，大概流程就是加载 elf，解析 elf，找到代码段，解压缩代码段，复制解压的数据回填到 il2cpp 里，这里推荐一个简单的办法，对 il2cpp 的代码段监听读写，就得用上 MemoryAccessMonitor.enable  
![](https://bbs.kanxue.com/upload/attach/202409/807696_D9YV5ECAEGGMJGR.webp)  
打印出一个大概的地址，偏差不会很大，decode_iL2cpp 的 v10 代码段起始地址，v11 是长度，hook dump 出指定起始地址与长度，回填到 il2cpp  
![](https://bbs.kanxue.com/upload/attach/202409/807696_5C77UXQR588N7YM.webp)  
 

global-metadata.dat
===================

舒服，终于看到了熟悉的操作，dump 出 global-metadata.dat  
![](https://bbs.kanxue.com/upload/attach/202409/807696_P8X4FDQUHGDMKQG.webp)  
用 Il2CppDumper dump 出游戏的 sdk  
![](https://bbs.kanxue.com/upload/attach/202409/807696_TMGW77PGSV9TKKN.webp)  
查看 dump.cs，舒服啊，还有韩文注释了函数的作用，hook 这些函数达到让人意想不到的结果  
![](https://bbs.kanxue.com/upload/attach/202409/807696_8QMHRKQTEGXSUFZ.webp)  
 

总结
==

难度中下，骚操作不多，集中在 libloader，有机会我得好好学习日文跟韩文，大佬们有没有资料发下  
样本：com.mobirix.mbpdh

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#逆向分析](forum-161-1-118.htm)