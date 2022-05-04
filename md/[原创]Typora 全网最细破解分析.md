> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272538.htm)

> [原创]Typora 全网最细破解分析

前言
==

我是逆向练习生，羽墨。

 

目前最为流行的 md 文件编辑器，当属 Typora，免费，简洁，让人爱不释手

 

但是去年突然开始收费，让人不知所措。。。

 

所以今天用简洁的手法，带大家体验破解第一视角

 

（其实刚收费的时候笔者就尝试过破解，那个时候刚学了几天汇编，打开调试器愣是看了一整天，毛都看不出来。。所以经过一段时间学习，又来了。。这就是不怕贼偷，就怕贼惦记？。。。）

 

**注：新手上路 如有错误请大佬指正**

磨刀霍霍 ：准备工作
==========

开发环境识别
------

用 IDA 打开 Typora.exe （不建议，可能是我电脑问题，IDA 分析了三四个小时。。。），打开后发现程序中有 electron ，V8 字样，由于前几天刚好分析了一个 IE 漏洞，由 V8 联想到 JavaScript 引擎，此时猜测与 JS 语言有关

 

![](https://bbs.pediy.com/upload/attach/202204/930159_8CPFNZX888ZQMNP.png)

 

![](https://bbs.pediy.com/upload/attach/202204/930159_AQAG2GGWJCTWND5.png)

 

打开程序目录，查看一下有没有 JS 代码，一番查找，看到了 asar 文件以及 node 字样，感觉有些眼熟，哪里见过。好的，问一下度娘，百度搜索 asar，electron，nodemodules

 

不搜不知道，一搜吓一跳，原来这是使用 NodeJs 的 electron 框架开发的桌面应用，JS 也能写桌面程序了。。。原谅我孤陋寡闻

 

继续搜索相关资料，得到以下信息：

 

1.electron 使用了谷歌的 V8 引擎以及渲染引擎（意思是这种程序跟个浏览器差不多呗）

 

2.electron 有主进程与渲染进程 ，通过 IPC 交换信息，渲染进程只负责渲染（难怪我附加调试有好几个进程。。）

 

3.Typora.exe 是 electron 框架，基本与开发者的代码无关（修改框架复杂度太高，一般人不会去动，所以别去逆向它了）

 

4. 查看目录发现，在 app.asar.unpacked 中发现了一个 main.node ，看名字很奇怪，.node 文件是啥东西

 

5.app.asar 是打包的 JS 代码，并且只是简单的打包，没有任何加密措施，且 electron 也没有代码保护措施（圈起来要考）

 

**根据第五条信息，笔者尝试使用工具对 asar 进行解包，解包失败不知道什么原因（或许必须使用 NodeJS 自带的解包工具？但是解包也是加密文件，我又懒得下载 nodejs，需要的时候再解吧。。），放进 010editor 查看，得到文件名与一些密文，此时陷入了僵局**

 

![](https://bbs.pediy.com/upload/attach/202204/930159_2M2VS93B3SVHC79.png)

尝试调试
----

打开 typora 进程，x64dbg 附加，查看有没有有用的信息（刚开始我不知道这个是 NodeJS 开发，想着通过注册窗口跟踪程序流程，好家伙差点就逆到引擎去了），发现了有好几个进程，后面命令行参数可以看到 渲染 gpu 等关键字，并且目录参数指向了 app.asar

 

![](https://bbs.pediy.com/upload/attach/202204/930159_JSWZZJ838PQNYKK.png)

 

根据前面得到的信息，这些进程都是由主进程创建的，程序逻辑是由主进程处理，那么附加主进程看一下（没参数的那个）

 

![](https://bbs.pediy.com/upload/attach/202204/930159_TVE9AUZA3HZDUTH.png)

 

看到了主进程加载了 main.node 模块，什么. node 也能当 dll 加载 ？？， 或许它本来就是一个 dll 呢（也不排除加壳后，手动加载的情况）

 

那么使用 PE 工具查一下，VS2017 编译的 64 位 DLL。好家伙，以为换个名字我就不认识你了

 

![](https://bbs.pediy.com/upload/attach/202204/930159_FMC56VNNPSJPJXZ.png)

梅开二度 ：逻辑分析
==========

好，到这一步，已知信息：

 

1.JS 文件被加密

 

2. 框架为 electron

 

3. 框架会加载 main.node 模块

 

4. 解析 JS 脚本的是 V8 引擎

 

5.c++ 支持 node api 进行开发

 

**根据我们的已知信息，来对程序的整体逻辑进行一个简单的分析**

 

1. 框架对代码无保护且修改框架难度过高，V8 引擎不支持解析加密的 JS 代码，那么 JS 代码如何运行？

 

​ 站在一个开发者的角度，我可能会想到由框架加载我的解密代码，把 js 代码解密后送到 js 引擎去执行就可以了

 

2. 那么解密代码放在哪里合适呢？

 

​ 既然框架被编译为二进制了，且根据之前的分析，妥妥的 c++ 开发，想要最简单的方法实现解密，加载同样为 c/c++ 编译的二进制代码即可。在 Windows 平台上，想要加载代码执行，也就剩动态链接库了。由此可以推测，之前找到的 main.node，可能就是解密模块。

逻辑总结
----

*   框架加载解密模块
*   解密模块对 app.asar 进行解密（解密后会不会把文件写出来呢，如果写出来可以直接拷走。。）
*   解密后的代码送入 JS 引擎执行
*   另外的逻辑 ：由解密模块解密 app.asar 的 xxx.js 代码，xxx.js 代码执行后，由它来负责解密剩下的 js 代码并执行，可以提高破解难度

精益求精 ：main.node 分析
==================

1. 根据之前的推测，main.node 负责解密，按照程序员开发习惯，Ctrl CV 实现，也就是使用公开的一些算法

 

2.IDA 加载 main.node，按照逆向惯例，先搜索一波字符串

 

此时看到了 buffer，base64，app.asar 等关键字。猜测一下，app.asar 加载到 buffer 然后进行 base64 解密（哈哈哈，我都笑了）

 

![](https://bbs.pediy.com/upload/attach/202204/930159_9E9U5RYNFZ96JDH.png)

 

3. 继续开展搜索工作，base64 未免太过简单了吧，使用 FindCrypt3 插件 ，搜索一下算法常量吧

 

此时找到了 AES 的算法常量，前两个是重复的，可能是插件问题。

 

![](https://bbs.pediy.com/upload/attach/202204/930159_7TNX263Y6DTXN7F.png)

 

4. 好的，现在面临一个问题，我不懂算法，怎么解密。。。只能去问度娘了，搜索一下 AES 加密解密原理与 C 实现代码

 

5. 根据搜索得知：

*   AES 使用最后那个常量数组进行解密
*   AES 有五种加密模式，常用的为 ECB CBC 模式
*   根据 AES 密钥长度，有不同的加密轮数（不是很懂哈。。）

6. 对解密常量进行交叉引用跟踪，找到这个函数以后，继续对函数进行交叉引用跟踪

 

![](https://bbs.pediy.com/upload/attach/202204/930159_KRHJYMSF6YASVYF.png)

 

大概经过三四次的跟踪，发现了这个函数，在这个函数里 F5 查看反编译代码，发现了 app.asar 字符串的引用

 

![](https://bbs.pediy.com/upload/attach/202204/930159_CH5JMPGYMW3TDJC.png)

 

![](https://bbs.pediy.com/upload/attach/202204/930159_PTZYXV4DPPXHNJX.png)

 

7. 此时进行推测，这个函数加载了 app.asar 的内容，并且调用 SUB_180003E40 进行解密

 

8. 跟进 SUB_180003E40 进行查看 ，此时发现了 base64 字符串的引用，推测对 buffer 进行了 base64 解密

 

![](https://bbs.pediy.com/upload/attach/202204/930159_WBVDP783S4BWSH3.png)

 

9. 看到很多不认识的 API，百度搜索得知，这是 Node API，简单去看一下函数功能 http://nodejs.cn/api/n-api.html#napi_call_function

```
NAPI_EXTERN napi_status napi_call_function(napi_env env,            //环境
                                           napi_value recv,         //名为global的值
                                           napi_value func,            //要调用的javascript函数
                                           size_t argc,                //JavaScript函数的参数个数  类似argc
                                           const napi_value* argv,  //JavaScript函数的参数数组  类似argv
                                           napi_value* result);     //返回的JavaScript对象

```

10. 根据文档得到的信息，参照这一部分 Node API，得到了如下信息（猜测调用了此函数）笔者对这语法难以理解，程序的目的是把对象进行 base64 编码？但是看这代码，base64 也没有被当作参数传递进去（不纠结了）

```
Buffer.from( object, encoding )
object:此参数可以包含字符串，缓冲区，数组或arrayBuffer。
encoding:如果对象是字符串，则用于指定其编码。它是可选参数。其默认值为utf8。
 
Buffer.from(string[, encoding])： 返回一个被 string 的值初始化的新的 Buffer 实例

```

11. 暂时先不管 node api 的语法与功能，继续往下看

 

![](https://bbs.pediy.com/upload/attach/202204/930159_PQQ485PNC6QQZSY.png)

 

看到了这部分的函数调用，进入查看，发现与 C 实现 AES 算法结构相似，推测这部分为 AES 解密

分析总结
----

根据目前的分析，得到如下信息

*   main.node 模块使用 node api 进行 js 函数调用
*   main.node 模块使用了 AES 解密算法（模式未知）

到了这一步，想要继续破解，首先要得到 js 代码，有两个办法以及面临的问题

 

1. 分析算法，找到密钥，如果是 CBC 模式，还需要找到 IV ，之后使用解密算法，解密 app.asar 的 js 代码

 

2. 分析程序执行流程，找到解密后的缓冲区，直接拷走，得到彻底解密后的 js 代码

 

面临的问题：

*   nodejs 语法不懂，这些 api 调用的具体 js 函数不清晰
*   AES 解密流程不熟悉，找到密钥或者 iv 的难度较高（通过加密轮数判断密钥长度，通过算法部分判断加密模式，并找到相关数据）
*   就算你找到解密 AES 的办法，它会不会还有别的加密措施与防护措施，例如密钥需经过 hash 摘要，文件完整性校验等（如果还有加密保护措施，那还得继续分析别的算法，一步一步还原，对于算法不熟悉的人时间成本太大了）

大海捞针 ： 寻找 JS 代码
===============

1. 根据之前的分析，我选择第二种获得 js 代码的办法 ， 分析程序执行流程，得到解密后的 JS 代码

 

2. 分析解密前的 js 函数调用， 由之前的分析得知， 参数有两个 ， 在 V27 的位置， V27 是由参数 a3 +8 得来的 。好，动态调式一波

 

![](https://bbs.pediy.com/upload/attach/202204/930159_H2E9RJX63J9RQXN.png)

 

3.x64dbg 打开 typora.exe，下一个 dll 断点，根据之前分析，他是动态加载的（多打了个 a 懒得换图了）

 

![](https://bbs.pediy.com/upload/attach/202204/930159_5J79SRDXP2ZV7BY.png)

 

4. 断点设置完成后，断在 loadlibrary ，进入模块入口后 ，去 IDA 计算偏移，定位到前面分析的函数位置

 

对了，记得使用 x64dbg 自带的 PEB 隐藏功能，并忽略所有异常，这个模块有简单的反调试手段（查看导入表可知，太过简单不分析反调试手段了）

 

![](https://bbs.pediy.com/upload/attach/202204/930159_F2AHE7MR47DEH9A.png)

 

定位函数位置，直接看后 4 位即可，同样为模块偏移 674A 的位置

 

![](https://bbs.pediy.com/upload/attach/202204/930159_3XKSA4R5Q73DKRV.png)

 

![](https://bbs.pediy.com/upload/attach/202204/930159_NQKJ6SRWGGBTMYG.png)

 

5. 下断后查看 decrypt（命名一下方便）函数的参数，x64 架构下，函数参数为 rcx rdx r8 r9 rsp+0x20

 

前四个从左到右，超过四个则入栈，rsp+0x20 为起始地址，详情可参考微软 x64 调用约定

 

![](https://bbs.pediy.com/upload/attach/202204/930159_K76SRFYK5R266D8.png)

 

6.r8 = a3 查看 *（a3 + 8）的值 ，这个值为之前分析的 v27 地址， 也就是 argv 继续查看指针指向内容

 

![](https://bbs.pediy.com/upload/attach/202204/930159_KPHU4THTZHEDUQ2.png)

 

![](https://bbs.pediy.com/upload/attach/202204/930159_PN2N7NRPCE7Q4K5.png)

 

现在得到了 buffer.from js 函数的两个参数 ，第一个像是密文，第二个没看出来，不是预料中的 base64，所以之前的推断貌似是错的？

 

7. 关注一下这两个地址 0000079908482119 00000799083CFEA5 在调用完 js 函数会有什么改变，直接来到调用的位置，调试器同步来到这个位置

 

![](https://bbs.pediy.com/upload/attach/202204/930159_AQXW59VR8ZFNBK3.png)

 

![](https://bbs.pediy.com/upload/attach/202204/930159_WPP9J66BN69KDCZ.png)

 

好，来到这个位置再次确认参数， 第五个参数为 rsp+20 ， 也就是 rax 的值， rax 为 argv ，进入内存查看，得到前面同样的地址 ，也就是 （a3 + 8），步过这个函数，查看对参数的改变

 

8. 执行完后查看刚才记录的位置，好的好的，耍我呢，啥都没变，并且后续也没调用相关数据（或许是最后一个参数返回了一个对象，忘记看了，如果返回的话应该是密文相关的东西，并且放到了某个数据结构中，所以在 IDA 中没看到直接使用的行为）

 

继续分析，到了 AES 解密代码部分，既然是解密，那肯定得把密文的缓冲区拿过来吧

 

![](https://bbs.pediy.com/upload/attach/202204/930159_MUGVGPD6PJG5B8C.png)

 

首先看到一串 16 进制的赋值，v46 开头的数组 刚好 32 个字节，也就是 256bit 。 有点像是 AES-256 的样子了

 

然后看到申请了 32 字节的内存，v32

 

之后调用了 sub_18000B060 函数，对 v46 与 v32 进行操作， 目测参数为 （目标地址，源地址，大小）

 

进入 sub_18000B060 函数查看 ，一大堆运算 ，根本不想看 ，根据之前的推测，可能是作者感觉直接把密钥放在程序中有些不妥，所以对密钥进行一个类似于解密或 hash 运算的工作（不展开分析了）

 

9. 继续分析

 

![](https://bbs.pediy.com/upload/attach/202204/930159_2JV7VWM58MYRRZ7.png)

 

可以看到 sub_180007000 函数 ，参数 v45 IDA 提示我是一个 char[256] 的数组，v32 为 32 字节的地址，v10 为一串神秘数据

 

可以得到一个结论，在经过 v46 的一系列运算，得到了一个同样大小 32 字节的数据，再把数据 与 v10 神秘数据进行操作，放入 v45 的 256 字节的数组中（好家伙 这是密钥吗 搞这么复杂）

 

跟进 sub_180007000 查看， 把 v10 放到了 v45 数组的 0xF0 的位置

 

之后调用了 sub_180007800 函数对自己的 PE 文件有些操作，简单看了下前面的汇编，主要内容为，把 v32 放到 v45 中，大小为 32 字节

 

![](https://bbs.pediy.com/upload/attach/202204/930159_QGMGWFRM9YVJNUE.png)

 

10. 继续分析

 

![](https://bbs.pediy.com/upload/attach/202204/930159_TXGGD7CGPCD6E4R.png)

 

接下来看一下 sub_180005c00 函数， 使用了 v27 ，之前分析出来的密文地址就在 v27 中， v30 没看出来，应该是传出参数，后面用到了

 

猜测这个函数对密文进行一波操作， 看一下返回值用来做什么，这个伪代码看的头疼，汇编看一下

```
.text:0000000180004021                 call    sub_180005C00
.text:0000000180004026                 mov     rbx, rax
.text:0000000180004029                 mov     r14, [rax+8]    ;返回值+8的内容给 r14
.text:000000018000402D                 sub     r14, [rax]       ; r14 - 返回值的内容
.text:0000000180004030                 mov     rcx, r14        ; 得到一个大小 Size

```

人工反编译一下 首先确定 rax 为一个指针 `*(rax+8) - *rax` 就是这个地址里面存储了两个值，拿第二个值减第一个值得到一个 size

 

此时猜测，这两个值或许是 密文的开始地址与结束地址？

 

然后用这个 size 申请了一块内存 ， IDA 命名为 Block ， sub_18000B060 之前分析过， 对 * v12 进行操作， 结果给到 Block

 

把 V12 代入 rax 中 ， `*(v12+8) - *v12` ， 结束地址减去开始地址得到 size

 

由此可以验证猜测， *v12 为密文开始地址 ，V13 为密文大小

 

11. 继续分析第三个方框的内容

 

以 v13 + 1 的大小 申请了一块内存 v14 ， sub_18000B060 对 Block 再次进行操作， 结果给到 v14

 

v14 的最后一个字节置为 0 ，推测已经把密文转换为字符串了 ， 需要一个 NULL 结尾

```
;v15 = r8d   rcx = v13    rbx = v14 
.text:0000000180004094                 movsxd  rcx, r14d
.text:0000000180004097                 movzx   r8d, byte ptr [rcx+rbx-1]

```

由上可得 v15 = v14[v13-1] , 也就是从 v14 中取了一个字节的值 ，位置在 null 字符的前一 byte

 

12. 继续分析 sub_180006AC0

 

![](https://bbs.pediy.com/upload/attach/202204/930159_4KRUV4ECK4A6NJ8.png)

 

可以看到，sub_180006AC0 的参数 ， v45（256 字节数组），block ，v13

 

结合之前对 sub_180007000 的分析，可以得知， 目前 v45 的状态 ，v45[0-31] 为 32 字节的类似密钥的东西 v45[0xF0] 为 v10 的神秘数据 ，v13 为 Block 大小

 

跟进简单查看 ，查看后感觉可读性不好，笔者对照汇编代码，重新修改了一下反编译代码

 

![](https://bbs.pediy.com/upload/attach/202204/930159_U6Y89BV3492K77K.png)

```
__int64 __fastcall sub_180006AC0(v45,block,block_size)
{
 
  if ( block_size )
  {
    v3 = block;
    v5 = v45 + 0xF0 - (_QWORD)block;           //v45+0xF0的地址  减去  block的地址得到v5
    v6 = ((block_size - 1) >> 4) + 1;          //做为外圈循环的次数
    do
    {
      v7 = *v3;             
     //v7为 xmmword 16字节浮点寄存器 ，把block的内容取16字节给v7  16字节符合AES块大小 
     //由此推测block是真正的密文，将在这个函数中进行解密操作
 
      sub_180007320(v3, v45);    //用到了AES解密常量  应该是解密相关  并且对推测的key  也就是前32字节有一些操作
      v8 = 16i64;                //内圈循环16次
      do
      {
        result = *((char*)(v3 + v5));       //block地址 + v5偏移  取一个字节内容
        *(char*)v3 ^= result;               //取block的1字节数据，与block地址 + v5偏移  进行异或
        v3 = (__int128 *)((char *)v3 + 1);  //block += 1
        --v8;                                //总共16次 也就是16个字节异或
      }
      while ( v8 );
      v5 -= 16i64;                          //外圈循环  v5 每次-16  也就是每次异或 异或的值都会变化 范围为-16字节
      v45 + 0xF0 = v7;                        //block的16字节内容  给到v45+0xF0
      --v6;                                    //外圈循环次数
    }
    while ( v6 );
  }
  return result;  
}

```

根据目前的分析，可以推测 ，sub_180006AC0 函数为 主要的解密算法函数，看着像是 AES CBC 模式，因为对算法不熟悉，大胆猜测一下

 

key 存放在 v45 中， 前 32 字节 ，也就是 256 位 ， iv 存放在 block+v5 中 （不清楚对不对）

 

13. 继续分析剩下的内容

 

![](https://bbs.pediy.com/upload/attach/202204/930159_YYYCMAMYBRTUJBN.png)

 

好的好的，看着有点头疼，后面的代码大概意思就是，又对解密后的数据进行了一系列操作，最后返回了一个缓冲区

 

**读者感兴趣可以自行分析，实在是写不动了**

釜底抽薪 ： 得到 JS 代码
===============

1. 根据前面的分析，我们已经大致了解了程序流程，来到调用解密函数的函数，只需要在彻底解密后，送到 JS 引擎执行的时候，拿到解密的 JS 代码即可

 

![](https://bbs.pediy.com/upload/attach/202204/930159_FTQ2NJNVEU3N485.png)

 

2. 根据上层调用代码，可以得到，解密后返回了一个值，作为调用 JS 函数的参数 ，定位到 678F 偏移处，x64dbg 同步定位

 

![](https://bbs.pediy.com/upload/attach/202204/930159_6ZKPWGRY96HW2HF.png)

 

3. 断下后查看 v28 的内容 ， RSP+20 的位置，然后继续查看这个指针的指针的内容，最后得到了解密后 unicode 形式的 JS 代码

 

![](https://bbs.pediy.com/upload/attach/202204/930159_UAFKWS6BKCKJVAZ.png)

 

4. 把内容拷走，拿到 010editor，把 00 去掉，变成 ascii 形式，检查一下得到的数据

 

![](https://bbs.pediy.com/upload/attach/202204/930159_M8WTKPRRDYTD6WW.png)

 

看起来跟密钥有关的一串字符编码数据

 

![](https://bbs.pediy.com/upload/attach/202204/930159_FQXBHH4DZFCJBUD.png)

 

搜索一下 license 相关的数据， 找到不少，看起来也像是代码，应该没问题

指鹿为马 ： 破解可行性分析
==============

修改文件破解
------

如果懂算法与 NodeJS，可以通过分析，找到关键的 key 等数据，对 app.asar 进行解包解密操作得到 JS 代码进行修改后，打包回去即可

 

可能遇到的问题：对 app.asar 进行完整性校验

内存破解
----

简单说几种思路，由于 main.node 是后加载的模块，所以内存破解有些难度

1.  调试器加载 ： 参照上述手段，在模块加载通知中断下，定位到解密函数下断，修改内存中的 JS 代码
2.  导出表 HOOK： 参考病毒木马使用的进程替换（傀儡进程）技术，创建进程后挂起，由于 main.node 中的 node api 是使用框架中的导出 api，所以可以替换导出函数为自己的函数，在调用时进行参数判断，如果为 JS 代码，则修改
3.  DLL 劫持：替换 main.node，由自己加载真正的 main.node 并调用，调用时，定位到解密函数并 hook，等待 JS 代码并修改
4.  PE 代码注入 ： 修改框架的 PE 文件，并加载自己的 DLL，加载后进行导出表 hook

可能遇到的问题：对 main.node 或者框架进行完整性校验，更加强大的反调试手段

 

**方法还有很多，不再一一列举，这里只能提出思路**

点到为止 ： 总结
=========

1.  通过这次逆向分析，踩了不少坑，学到了不少东西，并且加深了逆向技术的基础
2.  作为一个逆向练习生，遇到不懂的，不会的，应该迎难而上，扬长避短，不可轻言放弃
3.  遇到一个纠结的地方，不要过度停留，逆向分析应该是分析大方向，站在开发者角度，根据分析出来的功能猜测作者的意图，以找到关键突破点

结语
==

1.  由于笔者对算法与 Node Js 开发并不熟悉，所以没办法得到密钥与其它解密数据（文章中关于算法的一些操作皆为推测，相信熟悉算法的大佬可以看出来密钥所在）
2.  对于最后得到的 JS 代码也没办法判断到底完不完整，是否还有未解密的部分，所以只能到此为止了（看起来是完整了）
3.  由于笔者个人能力有限，没办法对提取的代码做分析修改，所以无法继续，如果有大佬帮忙，我会完成剩下的内存破解实现（心愿）

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 4 天前 被 yumoqaq 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#系统底层](forum-4-1-2.htm) [#软件保护](forum-4-1-3.htm) [#加密算法](forum-4-1-5.htm)