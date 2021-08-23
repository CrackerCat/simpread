> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269008.htm)

> VMP 导入表修复

[](#一、vmp的iat处理的几种情况：)一、VMP 的 IAT 处理的几种情况：
==========================================

1、call ds:[xxx]  
2、jmp ds:[xxx]  
3、mov reg,ds:[xxx] + call reg  
4、壳开始运行时填充到原来的导入表地址，不改变指令。

 

在 1 和 2 的情况下，原来都是 FF 15 或 FF 25 的 6 字节，vmp iat 保护后，代码中也会变为 1 字节 push/pop reg+5 字节 E8 call vmp0 或 5 字节 E8 call vmp0+1 字节的的两种类型：  
第一种类型：  
原来：  
![](https://bbs.pediy.com/upload/attach/202108/753304_6QYFMZAS67QRUSP.png)  
保护后变为 call vmp0 + 1 字节，这里 call 后面不全是填充的 retn，有可能是其他字节：  
![](https://bbs.pediy.com/upload/attach/202108/753304_52KFVPCTN4G2J3K.png)

 

第二种类型：  
原来：  
![](https://bbs.pediy.com/upload/attach/202108/753304_QARDJTSBUGHSUUY.png)  
保护后变为 push/pop reg + call vmp0：  
![](https://bbs.pediy.com/upload/attach/202108/753304_Q65HPW9HAPFPKPZ.png)

 

在 3 的情况下，根据原来对寄存器是否为 eax 的赋值，也会有两种类型：  
第一种：  
5 字节 A1 开头的 mov eax,ds:[xxx] 直接变为 5 字节的 E8 call vmp0  
第二种:  
6 字节 8B 开头的 mov reg,ds:[xxx]，则变为 5+1 或 1+5 类型的 E8 call vmp0

 

在 4 的情况下，原始的指令代码 call ds:[xxx]，jmp ds:[xxx]，push ds:[xxx]，mov reg,ds:[xxx] 不改变，壳在启动时候直接填充函数地址到 xxx。这类情况直接搜代码段 FF??，8B??，A1????，获取到对应内存的函数地址进行修复。

二、获取 call vmp0 指令地址：
====================

这里我直接对 text 段进行爆搜 E8 字节，然后判断是否是 call 到的 vmp0 区段，这样筛选出一堆地址，分析时候发现 call vmp0 区段后是以 nop 0x90 开头，之后再通过 unicorn 模拟获取在 retn 指令时候的返回地址，如果返回地址是当前其他模块的导出函数，则认为是没加壳程序原来调用导入函数的地址

[](#三、判断哪种iat处理类型：)三、判断哪种 IAT 处理类型：
===================================

1、对于 jmp ds:[xxx] 类型，由于本身不会对堆栈进行操作，而保护后改为了 E8 Call 方式，所以 vmp 为了恢复堆栈在最后 retn 时候是采用 retn 0x4 方式，对应 64 位为 retn 0x8。  
![](https://bbs.pediy.com/upload/attach/202108/753304_22S4QP7H9C2TMJ5.png)  
2、对于 mov reg,ds:[xxx] 方式，通过 unicorn 模拟可以发现返回地址是在 E8 call vmp0 导致的 + 5 或者 + 6 位置，对应上面 1+5 和 5+1 情况，同时只会写入除了 ESP 外的一个寄存器。  
![](https://bbs.pediy.com/upload/attach/202108/753304_WV7ADME3HY8CQSB.png)  
3、对于 call ds:[xxx] 情况，返回时候是以 retn 返回，并且返回的地址不在调用 call 的地址附近。

四、判断 5+1 或者 1+5 或者只有 5 字节模式：
============================

判断这个的目的是确定还原代码开始的地址，最开始我这里是通过模拟得到 retn 时候的返回地址和 E8 call 的地址进行对比是 + 6 还是 + 5 从而判断是 5+1 还是 1+5 情况，但是这种方式对于 jmp ds:[xxx] 类型不能取到返回地址，对于 mov reg,ds:[xxx] 方式也会因为有 5 字节的 mov eax,ds:[xxx] 而判断错误。

 

后面分析的时候发现对于 1+5 情况使用 push/pop reg + E8 call 方式，模拟时候把除了 esp 外的寄存器值都初始化为 0：  
1、对于 push reg +E8 call 方式在 call 内会通过 pop reg 来恢复 call 前因为 push reg 减少的堆栈，比如如下调用，模拟开始给的 esp 是 0x1000，恢复堆栈的指令为：

```
pop ebx            //恢复堆栈，同时ebx = 返回地址
xchg ebx,[esp]        //将返回地址和ebx值还原

```

所以此时会出现 esp = 0x1004，并且堆栈为：  
[0x1000+4] = 返回地址  
[0x1000+8] = 0  
![](https://bbs.pediy.com/upload/attach/202108/753304_JBA5BKJWE2XG3R4.png)  
![](https://bbs.pediy.com/upload/attach/202108/753304_PCWGU7ZV6CDUGNR.jpg)

 

2、pop reg+E8 call 方式会在 call 中进行 xchg reg,[esp]，push reg 操作，将原先保存在 reg 中的参数和返回地址互换，并且重新压入返回地址：

```
xchg edi,[esp]        //执行后edi = 返回地址，[esp] = 之前的参数
push edi            //push 返回地址

```

所以当 xchg 指令执行后会出现 esp = 0x1000，并且堆栈为：  
[0x1000] = 0  
[0x1000+4] = 0  
![](https://bbs.pediy.com/upload/attach/202108/753304_R5TDUXX4AXQCNM9.png)  
![](https://bbs.pediy.com/upload/attach/202108/753304_9H63SD9BFNRNHMH.png)

 

3、其他除了 5 字节的 mov eax,ds:[xxx] 外都是 E8 call + 1 字节的情况

[](#五、代码：)五、代码：
===============

代码直接在这个基础上改动 https://github.com/mike1k/VMPImportFixer，原代码把所有的指令都判断为 FF 15 类型，并且没有判断 5+1、1+5 和 5 的模式，以及 mov reg，xxx 类型和直接填充的类型。

 

参考：  
1、[https://github.com/mike1k/VMPImportFixer](https://github.com/mike1k/VMPImportFixer)  
2、[使用模拟器进行 x64 驱动的 Safengine 脱壳 + 导入表修复](https://bbs.pediy.com/thread-249322.htm)  
3、[手动分析 VMP 加密的 x64 驱动导入表](https://bbs.pediy.com/thread-248812.htm)

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

上传的附件：

*   [code+exe.zip](javascript:void(0)) （8.55MB，24 次下载）