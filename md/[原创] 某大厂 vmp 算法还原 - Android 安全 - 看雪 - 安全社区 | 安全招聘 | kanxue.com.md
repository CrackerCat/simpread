> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287180.htm)

> [原创] 某大厂 vmp 算法还原

**本着分享思路而发，部分敏感位置已打码，如有侵权，请联系删除**

最近因工作原因，需要分析样本的 3.0 算法，其他几个加密都相对于简单与朋友一起完成，已知情报也与公开的信息差别不是很大，vmp 是自己还原，所以本文的侧重点是，最后验签的 vmp 算法还原部分，本人底子较薄，如有专业名词叫错的部分，欢迎大佬纠错更正![](https://bbs.kanxue.com/upload/attach/202506/994069_GKVK3V7DUN428TT.jpg)  

准备：
===

工具准备：

*   两份信息量足够的日志，一个为主，另一个来核对，某些是否是固定值（yang 神开源的 qbdi 足以使用，如果加上读写时寄存器内存状态就比较完美了，我这里使用的是朋友用 stalker 的 trace 日志）
*   010editor
*   记录笔记的软件，我这里习惯使用语雀 （个人感觉，也可以不用追求大佬们还原虚拟机，通过日志也是可以还原算法，本质也是为了算法嘛，当然能还虚拟机最好， 目前我也在朝这个方向努力）

我的两份日志文件在后续会上传发布，没在当前发布文章的电脑上，到时候就会在这个位置吧

// TODO 上传日志  

**文章都是图片在上，文字在下！！！**

目标值：        
============

`fe 3d fe e3 af 56 13 8c 25 f9 88 a1 2b 24 2f 62`

(`254,61,254,227,175,86,19,140,37,249,136,161,43,36,47,98`)

我的日志中，十六进制均被转为数组存储

![](https://bbs.kanxue.com/upload/attach/202506/994069_6WME5PASVCW3WY4.jpg)

匹配到一个结果，这里 trace 打印的是 0x32 的长度，这里是结果，就要跟踪生成了

![](https://bbs.kanxue.com/upload/attach/202506/994069_R6F895Y5TJUTXJ8.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_TWCC58U6GTYUDMF.jpg)

ida 中看到，他在`0x36e80`函数中，又通过日志发现，`0x7b732755b0`是 x0，也就是第一个参数，frida hook 一下看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_X8U76JEEQWQ8A3P.jpg)

与预期相符

![](https://bbs.kanxue.com/upload/attach/202506/994069_AATRGJFEC745D9M.jpg)

直接搜索字节形式的结果前 8 个字节 `254,61,254,227,175,86,19,140`，可以看到结果还是挺多的，然后来分析一下这些结果吧

![](https://bbs.kanxue.com/upload/attach/202506/994069_SJY22MMEFAD5A5P.jpg)

因为直接搜索结果，只有一条搜索的结果，所以我们直接看上面最近一条的区别

### 发现一：

可以看到，当前结果的地址，在`0x7b732755a0`写入，与最终地址 `0x7b732755b0` 正好相差 16，而且数据的开始也是从`0x7b732755a0 + 16`开始

### 发现二：

这个字节与最终结果只有部分不同，

254,61,254,227,175,86,19,140, 237,249,136,161,43,36,45,66   （当前

254,61,254,227,175,86,19,140, 37,  249,136,161,43,36,47,98   （结果  

这时候可以猜想，他是否是分成两部分，然后再组成最后的 a2 部分，根据经验来看，这种单字节变化的，大概率是单个字节运算，运算指令就那么多，首先试试 eor

前 8 个字节：
========

![](https://bbs.kanxue.com/upload/attach/202506/994069_VMVM52KYA8ZBXKB.jpg)

通过搜索，可以定位到这里，可以看到，异或两次出的结果

第一次异或的结果再异或一次，得到最终结果

再仔细一看，第二次异或的是固定数组

![](https://bbs.kanxue.com/upload/attach/202506/994069_STP2W25XUUFDUWQ.jpg)

从另一组 trace 来看，第一次异或的值也是固定的，需要看第一次异或的 w22 从哪里来

d6 来源: (0x1d6 & 0xff)
---------------------

![](https://bbs.kanxue.com/upload/attach/202506/994069_ZQJZA3GJQA3TC4R.jpg)

重新回到第一份 trace 中，0xd6 从`0x7b8c9303c0 + 8`加载，搜索看看，能不能搜到写入

![](https://bbs.kanxue.com/upload/attach/202506/994069_N8NB78NTZVD7N8G.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_KUQ4TQEVYUUUP3P.jpg)

可以搜到，是 and 运算得到`0x1d6 and 0xff = 0xd6`

![](https://bbs.kanxue.com/upload/attach/202506/994069_EJ3DNYNAVKXMYW9.jpg)

继续往上看，0x1d6 从`0x7b8c930d28`加载，跟踪看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_7FHQZZZBYT7KPFR.jpg)

又是一个 and

![](https://bbs.kanxue.com/upload/attach/202506/994069_R9GSRXMMXP5GYY6.jpg)

过搜索，发现 and 的结果就是前 8 个字节需要异或的值，老老实实分开跟踪一下吧

### 0xf7 来源：

![](https://bbs.kanxue.com/upload/attach/202506/994069_CPW8SFNXR3NMT8Q.jpg)

从 x1 读取 0xf7，x1 是 从`0x7b8c9d5340`读取一个字节，继续跟踪

![](https://bbs.kanxue.com/upload/attach/202506/994069_3SB9H6PRGRE3BP6.jpg)

看着像什么值的 hash，转一下看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_ZU5Q6SPQVEHRXCB.jpg)

感觉有点东西，前 16 字节，回到 a2 前 8 个字节的异或看看，

![](https://bbs.kanxue.com/upload/attach/202506/994069_NMM2R84C8GY5WQ8.jpg)

正好对上

![](https://bbs.kanxue.com/upload/attach/202506/994069_H9Y9KTZFGG7Q7PP.jpg)

直接搜索对应字节，最后一次出现是在这里，当前地址是 `0x7c15c256e0`，搜索什么时候写入的

![](https://bbs.kanxue.com/upload/attach/202506/994069_JXQCXNR8YK8BBC9.jpg)

往上翻找，可以找到写入的情况，20 字节，地址在`0x52034`，ida 查看

![](https://bbs.kanxue.com/upload/attach/202506/994069_MVEQ77PSJFCGZUN.jpg)

跳转过来，看到是 hmac-sha1

### 0xdf 来源:

![](https://bbs.kanxue.com/upload/attach/202506/994069_SARPS8HP945EQCF.jpg)

一样的操作，注意要核对寄存器，因为有概率，搜索的是一个正好是目标值，但不是目标值的情况，跟踪地址看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_TA58Q5DE9PBMRPG.jpg)

继续跟踪看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_9MQ88GUBREFTHBT.jpg)

跟到这个位置，看起来又像一个 hash 的值

![](https://bbs.kanxue.com/upload/attach/202506/994069_P3CKMH2QZNPTFDX.jpg)

转换完之后，确实像哈希的结果

![](https://bbs.kanxue.com/upload/attach/202506/994069_62MK4E3CC5YYFVJ.jpg)

搜索只有两个结果，跟踪第一次出现的地址看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_85RJVXHZ3NXFR84.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_CFZMCPEWR5NG2A5.jpg)

最终正好 16 个字节

![](https://bbs.kanxue.com/upload/attach/202506/994069_YRJK4PRPXS7U96D.jpg)

异或出来，最终位置了，看一下 ida 中的位置

![](https://bbs.kanxue.com/upload/attach/202506/994069_UDVPRHRBQFQR9DK.jpg)

是一个另一个 aes 加密了

目前已知：
-----

a2 前 8 字节：`fe 3d fe e3 af 56 13 8c`

1、0xfe = (0xd6 xor 固定数组) xor 另一个固定数组

2、0xd6 = (0xf7 & 0xdf) & 0xff

3、0xd6 与 固定数组 异或 两次 得到最终结果

3、0xf7 是 hmac-sha1 的 16 字节的数组

4、0xdf 是 aes 的 16 字节的数组

那么与猜想相同，确实是分开最终拼接的

后 8 个字节：
========

0x24 的来源: (0x7e & 0xa4)
-----------------------

![](https://bbs.kanxue.com/upload/attach/202506/994069_RAJUUVAS8MS3KZ2.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_RVNUNBUH5RV5JSM.jpg)

可以对比得上，其中最后两位被写入两次，跟一跟看看吧

  
![](https://bbs.kanxue.com/upload/attach/202506/994069_3W6M2MYEWHB9GJP.jpg)

向上跟踪

![](https://bbs.kanxue.com/upload/attach/202506/994069_SRJY9G9MYUJS6EZ.jpg)

继续跟踪

![](https://bbs.kanxue.com/upload/attach/202506/994069_AQN8A32XBQ6ZEPJ.jpg)

运算出来的，分支运算，挨个跟一下吧

### 0x7e:

![](https://bbs.kanxue.com/upload/attach/202506/994069_RXFBUHMDJEMPR3A.jpg)

搜索跟踪看看

![](https://bbs.kanxue.com/upload/attach/202506/994069_6FCV2WQQ8NVB3YQ.jpg)

竟然只有这一个结果，

![](https://bbs.kanxue.com/upload/attach/202506/994069_VGSE4XZ58C7X2MD.jpg)

![](https://bbs.kanxue.com/upload/attach/202506/994069_BU96T83N3YDGMHW.jpg)

从 ida 看出，走到了跟第一个分组一样的地方，但是这里是在 vmp 中，纯静态分析困难，继续跟日志

简单跟了一下，跳转有点多，直接画流程图了

流程图：
----

//  因为 vmp 跟踪比较繁琐，这里省去了截图，其实原来就是对着值一直搜索 写入和 读取，直到跟到 运算指令生成，或者顶部是读取指令，耐心就可以

[https://www.yuque.com/zheng-cqyir/araztx/bzy6iwyfu6rgf5z3?singleDoc# 《vmp 流程图》](elink@326K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2&6N6i4q4#2k6g2)9J5k6h3y4G2L8g2)9J5c8Y4A6Z5k6h3&6Y4i4K6u0V1j5%4q4&6K9i4u0Q4x3V1k6S2M7X3q4*7N6s2S2Q4x3V1k6T1P5Y4V1$3K9i4N6&6k6Y4f1$3M7X3N6X3y4i4Z5K6i4K6y4r3M7$3W2F1k6$3I4W2c8r3!0U0i4K6t1K6i4K6t1#2x3U0m8Q4x3U0g2q4x3#2)9J5y4e0R3H3i4K6t1#2z5p5q4$3L8i4m8Q4x3U0g2q4y4W2)9J5y4f1t1#2i4K6t1#2z5o6q4Q4x3U0g2q4y4#2)9J5y4f1p5^5i4K6t1#2z5p5u0Q4x3U0g2q4y4g2)9J5y4e0W2n7i4K6t1#2b7V1g2Q4x3U0g2q4x3#2)9J5y4e0R3H3i4K6t1#2z5p5t1`.)

值得一提的是，a2[8] 是最后生成，算法有时候会有额外多一个 and 的操作，如果发现自己的结果跟分析后的每次都差一，那就直接加上 and 1 就可以了，因为我的第三份日志确实比我前两份日志多了一次 and

结语：
===

vmp 算法还原并不可怕，可怕的是望而却步的念头，一步一步来就可以拨云见日

题外话：
====

最近在研究 stalker，进而了解到 frida-gum，但是编译中各种磕磕绊绊，期望可以寻找志同道合的朋友一起讨论和研究技术，私信我吧（接单勿扰哈）

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#NDK 分析](forum-161-1-119.htm)