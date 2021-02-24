> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266144.htm)

最近学了点 ollvm 相关的分析方法, 正好之前朋友发我一个小 demo 拿来练练手.

 

![](https://bbs.pediy.com/upload/attach/202102/917350_JKXK2ECGQ5PZ42E.png)  
看上去很简单 就是找 flag 用 jadx 打开发现加壳了  
![](https://bbs.pediy.com/upload/attach/202102/917350_F9F34SX9M2GMSBJ.png)  
然后想试试直接用 fridadexdump 脱壳的时候发现 frida 上就崩了

 

![](https://bbs.pediy.com/upload/attach/202102/917350_3PJTKGTTDXMXXV7.png)  
上葫芦娃的 strongfrida 直接重启了!....

 

这只能去过反调试了, 打开 so 找了下. init 和. initarray(反调试常见位置, so 比较早的加载时机)  
![](https://bbs.pediy.com/upload/attach/202102/917350_86NF4KDBMVM6N62.png)  
ctrl+s 打开 initarray 里面有两个奇怪的函数 decode, 这就是 ollvm 默认的字符串加密  
![](https://bbs.pediy.com/upload/attach/202102/917350_64VF34UW32X5BEU.png)  
导出函数中搜索 init 找到里面也有函数定义, 创建了一个线程执行反调试函数, 进入这个函数看到字符串都被加密了  
![](https://bbs.pediy.com/upload/attach/202102/917350_EYN3U5RJQE4P2QS.png)  
看了下代码 有个 kill 函数, 一开始就行 hook kill 函数不让他自杀, 发现没用...

 

![](https://bbs.pediy.com/upload/attach/202102/917350_E4PB2XWZX33H6ZQ.png)  
然后我继续找了下发现有个 strstr 函数 这有点可疑, 拿来比较字符串的, 直接 hook 一波  
![](https://bbs.pediy.com/upload/attach/202102/917350_2PAZW372W3HHZRW.png)

 

从代码中得知 str2 是我们要关心的对象, 是个 char * 类型直接 readCstring 打印

 

然后发现出现很多 frida 然后进程就崩了  
![](https://bbs.pediy.com/upload/attach/202102/917350_B4MWHASQUEQ5JFV.png)

 

所以接下来就有两种方法 直接 hook create_pthread 不让这个线程程起来 和 hook strstr, 我这就直接 hook str 了.  
![](https://bbs.pediy.com/upload/attach/202102/917350_9V6B5FJXMSVFJS2.png)

 

然后就不崩了, 可以愉快的 frida 了~

 

先拖个壳, Fridadexdump 一波 (可以直接 fart 脱的全, 但是懒的刷机, 直接用这个了..),hook events 定位  
![](https://bbs.pediy.com/upload/attach/202102/917350_XBJ38BJMC9WDHDR.png)

 

![](https://bbs.pediy.com/upload/attach/202102/917350_VE4Q5X3VJQM3AUW.png)

 

oncreate 直接是 native 化了 应该是 360 的 vmp, 这里肯定不是让你逆 360 的 vmp, 盲猜一波就是 check 方法, 先 hook 一下.

 

![](https://bbs.pediy.com/upload/attach/202102/917350_N2SAYRUFCY58D6E.png)

 

直接返回 false, 应该是 true 就会拿到 flag, 为了不每次都手动输入 直接写个主动调用.  
![](https://bbs.pediy.com/upload/attach/202102/917350_7SN7VA3Q8TB6H32.png)  
接下来去看 so 了, 导出函数就这几个.. 肯定是动态注册, 先直接 hook  
![](https://bbs.pediy.com/upload/attach/202102/917350_TCMNVQ7PUGK23UT.png)  
![](https://bbs.pediy.com/upload/attach/202102/917350_NJ42ETYQ9A9AQHA.png)

 

hook 到了偏移, 去 so 看一下  
![](https://bbs.pediy.com/upload/attach/202102/917350_AHVK9Q9BWKAC574.png)  
改了些名字后 操作流程就是将输入的 jstring 转为 char* 然后判断长度是否为 20 看到很多都是 unk_开头的, 这些就是被 ollvm 加密后的字符串, 要怎么看他解密后的内容呢? 因为他加载到内存的时候肯定是解密状态, 直接读这个地址打印 cstring 就可以得到解密后的字符串了.  
![](https://bbs.pediy.com/upload/attach/202102/917350_PEW28HP5YYK6JPE.png)  
然后继续改名, 改完名字就是这样  
![](https://bbs.pediy.com/upload/attach/202102/917350_KWB846HS6XA7KT5.png)  
发现他重新对 check 的方法进行了动态注册, 然后 sub_1234 里面也进行了动态注册里面嵌套了好几次动态注册, 然后用 strcmp 比较 输入的值和 &unk_5100(解密后为 kanxue) 如果相等那就返回 true. 但是输入要 20 个字符 kanxue 只有 6 个字符, 我们 hook 一下这个比较的地方看看,

 

直接 hook strcmp 蹦的比较厉害, 我选择 inlinehook, 按 tab 切换到汇编, 打开 opcode,s1 的值给到了 R0 我选择在 0x1564 进行 hook, 因为是 2 个 opcode 所以为 thumb 指令, 需要地址 + 1  
![](https://bbs.pediy.com/upload/attach/202102/917350_AKBQU7RTAU2YE9J.png)  
![](https://bbs.pediy.com/upload/attach/202102/917350_X3SWPZBQJ3NUPMQ.png)  
![](https://bbs.pediy.com/upload/attach/202102/917350_YF7ZF8STZNJ53FY.png)  
我们输入了 kanxue00000000000000,hexdump 打印 s1, 发现 s1 就是 kanxue, 但是程序没提示通过, 然后 hook registernative hook 到他又进行了 2 次动态注册, 根据这个地址 我们往前找  
![](https://bbs.pediy.com/upload/attach/202102/917350_KJMEQP6C8H2RX6R.png)  
![](https://bbs.pediy.com/upload/attach/202102/917350_BUT4UUN46AQYFXA.png)  
发现第二次动态注册的和最开始的代码很像, 第三次就短了很多, 然后仔细一看第二次动态注册中又注册了 sub_1148, 原来就是嵌套了 3 次动态注册, 注册了 3 个函数, 根据一开始的分析 对 strcmp 这里的字符串解密

 

![](https://bbs.pediy.com/upload/attach/202102/917350_ZXY4ZXBPJSSSKSK.png)  
然后 inlinehook 剩下几个分别 strcmp 验证下  
![](https://bbs.pediy.com/upload/attach/202102/917350_Y7XRK5YZM863GR3.png)  
发现第二次比较是输入的 8 个 0 这样区分不了是第几个输入, 我们用 abcd 来实验, 输入 kanxueabcdefghieklfn 20 个字符  
![](https://bbs.pediy.com/upload/attach/202102/917350_DRZD5HQ8YRPRX5J.png)  
是 abcdefgh 这 8 个, 所以是加到 kanxue 后面的, 然后真正的字符串之前我们已经解密了, 是 unk_50F7(即为 training), 然后后面猜也猜得到剩下 6 个字符是之前解密的 unk_5096(即为 course) 为了保证严谨性, 我就在 inlinehook 一次

 

![](https://bbs.pediy.com/upload/attach/202102/917350_YANAZZ95DNMKEFE.png)

 

果然就是我们最后六位进行比较 所以答案前面的解密的字符合起来, 就是 kanxuetrainingcourse  
![](https://bbs.pediy.com/upload/attach/202102/917350_Q3ADZCFE6XYUNAT.png)  
![](https://bbs.pediy.com/upload/attach/202102/917350_RFY4TP6QAZRCHAD.png)  
这里有个 bug 每次输错都要重启一次, 不然到最后答案就变成 course 或者 trainingcourse 了~

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

上传的附件：

*   [2、找出 flag.apk](javascript:void(0)) （2.36MB，9 次下载）