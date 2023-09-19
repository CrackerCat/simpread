> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278934.htm)

> [原创] xhs 最新版本 web 加密参数分析 （如有侵权联系删除）

[原创] xhs 最新版本 web 加密参数分析 （如有侵权联系删除）

1 小时前 43

### [原创] xhs 最新版本 web 加密参数分析 （如有侵权联系删除）

* * *

其实最早改版的时候我就还原出来了，至于为啥没发，毕竟人家刚改版。  
以下只是思路，没有算法也不要问我要算法，我不做那个，只是研究学习。

三个参数 X-S，X-S-Common，X-T, 先说下我的思路，我是找到了一个 / webprofile 的接口，这个有的人应该比我懂，data 里面长这样  
![](https://bbs.kanxue.com/upload/attach/202309/921346_N5FHXCW62WN3QY7.png)

这个 profiledata 就是浏览器一堆环境，我最开始破解的版本是 3.3.3，看来这已经升级了，但估计大体没变化。咱们直接搜索 profiledata。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_J9RSB6RNZ3N9DCF.png)  
我们先下断 o.getv18, reportBroswerInfo_awaiter 和 reportBroswerInfo_generator  
vmp 文件就在 window.xhsFingerprintV3.getV18 里面，点进去看看大体框架，我只能说跟某程的加固框架是同一个，特征很明显。等过一会再写一个 testab 的文章哈哈哈

咱们随机先下一个断点 log，随便输出什么都可以，然后步过执行发现 reportBroswerInfo_awaiter 和 reportBroswerInfo_generator 都没有日志输出，而且等 reportBroswerInfo_generator 之后 n 就已经出来了，说明 getv18 里面就是一个整体流程。

然后就开始下日志断点，搜索 apply 一些关键点来下，观察一下作用域里的值，那些是用来存储数据的哪些是用来存储指令集的，第一次多下一点一点一点缩小范围减少日志输出。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_29GPZ374KXE2PAJ.png)  
比如这都调用 charcodeat 了，那说明就开始算法的加密了，直接设置条件断点设置字符串等于 charcodeat 中断

断下之后打开闭包数据找一找，发现了这么一堆，看到 ey 开头赶紧拿去 base64 解码，这个搞多了都懂。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_D7Y2V8Y4EZF54QC.png)  
出来是这么个东西  
![](https://bbs.kanxue.com/upload/attach/202309/921346_FUWCWJ5C4VPBY2S.png)  
这就是初始化的浏览器环境，这省事了，直接下这个变量的日志断点就可以了，然后观察值的变化，发现 v18 结束，这个值确实出来了。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_43UX2HF6YB42AJQ.png)  
这一看就是 16 进制的数据，直接复制出来看最后两位 F3 十进制是 243，直接控制台往上翻，确实对应上了，82 的十进制也是 130  
![](https://bbs.kanxue.com/upload/attach/202309/921346_Q48KXT9CB8XKPDW.png)  
看下这个数组长度是 8，那说明这个是个分组加密，上面的日志还有很多，这里就不说了，说到这基本算法都能猜出来了，就差还原了，也是继续下 log，这次 log 就下在微运算上，把微算法分析在拼接。  
幸亏是对称加密，结果不变，不然都不好断。  
所以初始位置暂定为 192, 88 来断点这是加密结果开头的两个十六进制。  
断下之后我们一步一步往下走，就开始发现每 8 个字节进行加密了，后续把这个 charcodeat 的日志打全或者单步就发现这些字节就是上面 base64 的指纹。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_9K7387A55U7QFQ9.png)  
后面的流程上面也说过了，就开始拼接微运算的算法，结果大致如下。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_62JZMBYH76UGBU7.png)  
发现运算的时候还会去取一些固定的数组的值，就是最开始截图的数据存储的地方  
![](https://bbs.kanxue.com/upload/attach/202309/921346_EHJ5N6QSFJDWS96.png)  
等还原结束我才发现了这个算法其实调用库会非常简单，key 在我还原的时候已经发现了，但是不能说哈哈哈，最后就是 8 个一组的数据，转 16 进制。至此这个指纹就到这了，具体收集什么就不讲了，得挨个去分析。

然后我们继续往下走的时候发现开始开始获取 x-s,x-t 了，就是一个三元表达式，这个对象没有就走老的 sign，有的话就走新的。  
进去也是 vmp，但是不是同一个 vmp 文件  
![](https://bbs.kanxue.com/upload/attach/202309/921346_363FZ9WPKKSRZ86.png)  
老办法还是上断点，先断 apply 一些重要的，最后的 base64 也是标准的，直接把这个 x-s 先解码  
![](https://bbs.kanxue.com/upload/attach/202309/921346_TDFZG5WKMQ5ZNCE.png)  
这个 playload 跟那个 data 不能说完全一样那也大差不大了，我当时感觉就是用的同一个算法，后面看了一眼还真是，就是 key 不一样 ，算法怎么追 data 已经讲过了，直接找原文就行了。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_NHYTBQPSYP5MHQC.png)  
同上像上面一样断 65，ce 的 10 进制是最简单的  
![](https://bbs.kanxue.com/upload/attach/202309/921346_7BJVKDNAPDM5S4Y.png)  
然后发现一段字符串，x1 是 url 的 md5，我校验过没有魔改，x2 是浏览器环境，x3 是 local_id，这个 local_id 就是随机的字符串和时间戳还有 platform 和校验码组成的，x4 就是 x-t。  
'x1=6667296f17398985a2a087300d1474e4;x2=0|0|0|1|0|0|1|0|0|0|1|0|0|0|0;x3=1899576b2c4e9kacvosh7lks0930l0gtkf07f0qjv30000393857;x4=1695126119863;'  
然后发现也是 8 个一组在运算，只是 key 不同，到这 x-s 就完事了.

重点说一下这个 common，听别人说获取游客 id 必须要，之前还不是那么重要。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_NBGNA9JKS64XN9R.png)  
这里面的算法就有说法了，重点是这个玩意，他是在 localStorage 存储直接 hook 一下 set 看看堆栈，发现他是初始化的时候在 fingerprint 之前就存储了，跟 fingerprint 同一个 vmp  
![](https://bbs.kanxue.com/upload/attach/202309/921346_3KRYC2RSBAEY7MV.png)  
然后就是 trace 微运算还原之后，其实就是 fingerprint 里面个别的字段，看着就是一堆环境，进行了 rc4 和魔改 base64，里面还有字符串的运算。  
![](https://bbs.kanxue.com/upload/attach/202309/921346_Q2TVGBERDQG3N7J.png)  
像这些混淆直接 ast 还原在还原成 python 就行了，很简单  
![](https://bbs.kanxue.com/upload/attach/202309/921346_NWG5J7ED8NYYR3X.png)

到这 common 就结束了，后面写着写着想水文章了，发现我的字数一段比一段少

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

最后于 59 分钟前 被 wbwnnx 编辑 ，原因： 没有大标题