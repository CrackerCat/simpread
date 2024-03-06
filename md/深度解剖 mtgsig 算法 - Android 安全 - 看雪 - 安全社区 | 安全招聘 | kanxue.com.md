> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280779.htm)

> 深度解剖 mtgsig 算法

深度解剖 mtgsig 算法

17 小时前 249

### 深度解剖 mtgsig 算法

 [![](https://bbs.kanxue.com/view/img/avatar.png)](user-home-994949.htm) [我是小趴菜](user-home-994949.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star_0.gif) 17 小时前  249

1. 前奏说明分析
=========

1.1 分析 app：
-----------

Y29tLnNhbmt1YWkubWVpdHVhbg==

1.2 参考文章：
---------

[1] 我是小三大佬对 2.0 版本的参数分析  
[https://www.cnblogs.com/2014asm/p/15391247.html](https://www.cnblogs.com/2014asm/p/15391247.html)  
[2]qxpy 大佬对 1.5 版本的 a2 分析  
[https://mp.weixin.qq.com/s/S_3tM_TbOfbBRprmQGDwrA](https://mp.weixin.qq.com/s/S_3tM_TbOfbBRprmQGDwrA)  
[3]qxpy 大佬对 2.1 版本的 a5 分析  
[https://mp.weixin.qq.com/s/bfGvUyIN8aS4Y1KkOWwRpQ](https://mp.weixin.qq.com/s/bfGvUyIN8aS4Y1KkOWwRpQ)  
[4]TS 大佬对风控参数的分析  
[https://blog.vivcms.com/2021/01/07/429.html](https://blog.vivcms.com/2021/01/07/429.html)  
[5]houjingyi 大佬对 libmtguard.so 混淆的还原分析  
[https://bbs.kanxue.com/thread-271853.htm](https://bbs.kanxue.com/thread-271853.htm)

1.3mtgsig 参数分析：
---------------

> a0：mtgsig 版本号（这里分析 2.4）  
> a1：appkey（相同版本 app 此值固定）  
> a3：Android 设备版本  
> a4：时间戳  
> a5：加密的设备信息 1  
> a6：固定值  
> a7：xid  
> a8：dfp  
> a9：加密的设备信息 2  
> a10：a2 参数加密时需 xor 的随机数  
> x0：固定值  
> a2：sign 值

2.dfp 参数构造
==========

基于对 libmtguard.so 的 JNI 方法 main 函数 hook，如下图：  
![](https://bbs.kanxue.com/upload/tmp/994949_ADJSXVWB7R57XZU.webp)

> 以上图片分析：  
> 1. 调用 main(47) 前就已经生成过 mtgsig 参数，证明 dfp 已经生成并通过对比 main(47) 返回值和 mtgsig['a8'] 相等，验证猜想  
> 2.unidbg 模拟执行时，如果 main(47) 前调用了 main(1), 则会在全局变量中存储 dfp 并在 main(47) 调用用时存储到 / data/data/xxx/file/.mtg_dfpid 文件中  
> 3.unidbg 模拟执行时，如果只调用 main(47)，则会自行生成 uuid 和时间戳等参数构造 dfp 并存储到上述文件中  
> 4. 后续会发现 mtgsig['a8'] 值会发生改变（通过后续分析则是请求返回）

这里分析来源 unidbg 只调用 main(47) 的逻辑，猜测 main(1) 中生成 dfp 算法一致

2.1 算法流程
--------

> dfp 算法步骤  
> 1."0000"+uuid(去除'-')+ 时间戳 (十六进制)+"0"+CRC(前面部分)  
> 2.0000 8a8921de5fc14ed1b428d1d7485f99c1 18bc6958715 0 eb79a8ab  
> 3. 以上内容转成 hex  
> 4.00 00 8a 89 21 de 5f c1 4e d1 b4 28 d1 d7 48 5f 99 c1 18 bc 69 58 71 50 eb 79 a8 ab  
> 5. 固定 hex（多次测试目前版本 app 该值固定）  
> 6. 两组 hex 进行 xor 得到结果取大写

2.2 算法分析
--------

### 2.2.1unidbg 模拟执行并寻找切入点

![](https://bbs.kanxue.com/upload/tmp/994949_5UZQCRTAFMMNJFN.webp)

> 1. 从生成时间戳入手，分析地址 0x17bf5（GetStringUtfChars），全局一共两处调用，第一次是 uuid，第二次是时间戳  
> 2. 结合 ida 代码和 unidbg 调试代码分析方法的执行逻辑

### 2.2.2 结合 trace 日志分析 GetStringUtfChars 前后调用逻辑

![](https://bbs.kanxue.com/upload/tmp/994949_RZHB75BJPERPF2U.webp)

> 1.0x6F68E -> bl sub_17bc4  
> 2.0x17bc4 -> GetStringUtfChars  
> 3.0x6f2bb -> bl #0x40006e22（解密字符串得到'0000'）  
> 4.0x6f2e1 -> bl #0x40012aa4（拼接字符串：'0000'+uuid）  
> 5.0x6f333 -> bl #0x4002aa18（CRC 算法，查看入参："0000"+uuid(去除'-')+ 时间戳 (十六进制)+"0"）  
> 6.0x6f375 -> bl #0x4000c8f4(拼接 "0000"+uuid(去除'-')+ 时间戳 (十六进制)+"0"+CRC(前面部分))  
> 7.0x6f389 -> bl #0x4006ee60(入参：上诉拼接好的字符串，返回：dfp)  
> 8.0x6efe7 -> bl #0x4006e968（String to Hex）  
> 9.0x6ef61 -> eors r1, r0（r1 则是 so 中固定 hex，r0 则是 StringToHex 内容）

3.Xid 参数构造
==========

这里描述的 xid 是指在请求 fingerprint/v1/info/report 获取真正的 xid 之前使用的 a7  
构造 main(1) 和 main(2) 的时候，都会生成 a7，如果指定了. mtg_dfpid_com.sankuai.meituan 文件，可以发现生成的 a7 并非此文件的 xid（文件中的 xid 是请求返回的，抓包时的 a7 就是该值）  
![](https://bbs.kanxue.com/upload/tmp/994949_2QQ7QDAB7ENX8WF.webp)  
猜测我构造的 unidbg 可能并不完善，并没有读取到已经返回的 a7，那么此时的 a7 有可能是自己加密生成的，分析代码，看 a7 如何构造，然后抓包分析获取真正 xid 之前的 mtgsig 参数中的 a7，对比加密是否一致  
【以下分析是在分析 a9 参数时，发现的一个 aes 算法中打印发现，具体如何寻找 aes 算法后续 a9 参数分析】

3.1 算法流程
--------

> xid 算法步骤  
> 1.AES_data = '0'+dfp+'1'+ 时间戳（10 进制 hex）  
> 2.AES_KEY='meituan1sankuai0'  
> 3.AES_IV='0102030405060708'  
> 4.AES_CBC(AES_data) -> base64

3.2 算法分析
--------

1.unidbg 对标准 AES 算法 0x9dfc5 下断点调试  
![](https://bbs.kanxue.com/upload/tmp/994949_ZR2EHYCHWGF6PDA.webp)  
2. 通过入参很容易分析出'0'+dfp+'1'+ 时间戳（10 进制 hex）, 至于后面的 1+6551f39f，根据经验分析 8 位 6xxxxx 这样的 hex，可能就是时间戳 10 位的 hex  
![](https://bbs.kanxue.com/upload/tmp/994949_98UC6TC9GXVMBWY.webp)  
还有时间戳 13 位的 hex，就是 11 位的 18cxxxx 这样的开头（后续也会用到）  
3. 通过 unidbg 的 blr 指令查看调用 AES 算法 0x9dfc5 的位置，并分析 so 层，找到 key 初始化的地方和 iv 的地方  
![](https://bbs.kanxue.com/upload/tmp/994949_RP2FECTM3F9EMK6.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_DUJYDG8DZYRGVZK.webp)  
4.0x2ABF6 -> bl sub_9D32C 就是 key 初始化的地方，可以获取到 key，至于 iv，则 mt 大部分 iv 都是'0102030405060708'

4.a5 参数构造
=========

4.1 算法流程
--------

> 1. 设备信息进行 zlib 压缩  
> 2. 压缩内容进行魔改 RC4 算法  
> 3. 最后结果进行 base64  
> 4. 魔改 RC4 的 key: 固定 hex xor (a1+a3+a4)

4.2 算法分析
--------

1.a5='EGQJ3E...VR6X='，明显的 base64 编码，在 so 文件中定位 base64 编码进行分析  
2. 搜索'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'  
![](https://bbs.kanxue.com/upload/tmp/994949_BGKAQFNM9RU4A84.webp)  
3. 交叉引用定位引用位置  
![](https://bbs.kanxue.com/upload/tmp/994949_YRDWMNARXD5YJCZ.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_JJWS7TP7SJT54SA.webp)  
4.unidbg 对 0xfe6c 下断点进行调试，并且 blr 找到调用 base64 的位置  
![](https://bbs.kanxue.com/upload/tmp/994949_9H2H2XTT52VQ3WE.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_RC42QEJAGF6A6UC.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_J54WJ8E9HQ7PGPS.webp)  
5. 定位 base64 的入参哪里来，如上图发现是 0xbdc2 调用一个方法来的（分析后是魔改 RC4），如果不严谨还可以结合 trace 日志向上分析  
![](https://bbs.kanxue.com/upload/tmp/994949_49B62Z8FJ2473WV.webp)  
6. 分析 0xbdc2 调用方法内部逻辑，结合 RC4 源码，发现类似：v9 = *(result + v7); *(result + v8) = v9 这样的代码，猜测就是 RC4 算法，而 RC4 算法在加密前会进行密钥初始化，上面的图片中可以知道在调用 RC4 算法前还有一个 0xbdb8 调用了 0x2af9c 方法，明显这个就是 rc4_init 方法，而参数 2 就是 key，参数 3 就是 key_len  
![](https://bbs.kanxue.com/upload/tmp/994949_4DEKUJMNAG6DKRB.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_F9VDDV7SU5VSG25.webp)  
7.unidbg 验证猜想  
![](https://bbs.kanxue.com/upload/tmp/994949_X6PA7VHGB5W5HQ6.webp)  
8. 使用 RC4 算法工具去计算，发现结果还是不对，这是因为魔改了 RC4，在 RC4 方法代码对比没有问题，那么可能问题出现在 rc4_init 上  
![](https://bbs.kanxue.com/upload/tmp/994949_RPDDQFQEKV6U27B.webp)  
9. 下面分析 RC4 的 KEY 怎么来的，对 key 的地址 0x402ed090 进行 tracewrite，搜索哪里对这个地址进行了写入  
![](https://bbs.kanxue.com/upload/tmp/994949_MJUTN3EQTT9R925.webp)  
10. 分析 0x2b8a8 地址前后的代码，看该值如何获得  
![](https://bbs.kanxue.com/upload/tmp/994949_EMVM8ZZTM7NX22X.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_HW5USPECEHKX4RM.webp)  
11.Key 是 xor 而来，分析 xor 的数据内容，很容易分析出是 a1+a3+a4 的值 xor 固定值，至于 rc4 的入参，直接 unidbg 下断点看内容就知道是 0x78、0x9c 开头的 zlib 压缩数据，解压缩得设备信息  
![](https://bbs.kanxue.com/upload/tmp/994949_N5NTT6GN8HJKP8H.webp)

5.a9 参数构造
=========

5.1 算法流程
--------

> 1. 设备信息 zlib 压缩  
> 2.crc32 获取校验值  
> 3.AES_KEY=crc32+'MXMYBS@H'  
> 4.AES_CBC（zlib 压缩信息，iv='0102030405060708'）  
> 5.a9=crc32+aes 加密结果

5.2 算法分析
--------

1.a9:"c0ee827xGyF...pJrn4", 对 base64 方法 hook，获取和 a9 一样的返回值  
![](https://bbs.kanxue.com/upload/tmp/994949_936GEJ6NW9S2VRE.webp)  
2. 发现和 a9 很像的部分，猜测 a9 是两部分组成; 找到 base64 的入参地址，tracewrite 看哪里对这个地址进行了赋值 0x402ff100  
![](https://bbs.kanxue.com/upload/tmp/994949_H726W6RCHYJYSHP.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_BAHCMKKEQDG52EQ.webp)  
3. 通过 ida 发现这个地址 0x9d800 方法很像 aes 算法，但是是否魔改还不知道，继续 hook 分析  
![](https://bbs.kanxue.com/upload/tmp/994949_YCZ6MRBKQEJ847Z.webp)  
4. 末尾看，aes 的赋值肯定是在最后，所有搜索 * a2 = HIBYTE(v7); 的地址，并且存在 0x1b 的，a2[1] = BYTE2(v7); 地址，并且存在 0x21 的，则代表是我们要找的执行方法（过滤一下是因为其他地方可能也会调用到这个地址，并且值刚好一样）  
![](https://bbs.kanxue.com/upload/tmp/994949_T5WQCUPYYUTN9SZ.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_WJJJ93MV6VB7K32.webp)  
5. 交叉引用 sub_9d800 发现两个方法调用，通过上面的定位，确定是哪个方法调用的，也可以 unidbg 进行 blr，但是如果是 aes 算法，可能很多地方都调到这个函数，太麻烦了，所以使用静态分析  
![](https://bbs.kanxue.com/upload/tmp/994949_6CBBP3455B924ZZ.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_MRQUDGYGTRU8FMK.webp)  
6. 最终发现是：0x9DFC4 方法中调用了 0x9e05f: "bl #0x4009d800" 这个方法，直接通过 unidbg 对 0x9DFC4 进行调试，可以得出这个就是标准的 aes 算法（最终定位到调用逻辑 0x2ac11 -> 0x9DFC4 -> 0x9d800）（aes 算法前后对 key 进行了初始化）  
![](https://bbs.kanxue.com/upload/tmp/994949_NPRU8WUYPXYDR3H.webp)  
7. 对 aes 加密入参进行分析，发现也是 0x78 0x9c 开头，zlib 压缩了，解开就是设备信息明文  
8. 对 key 进行分析，发现 key 是 0x402d950c 这个地址（分析这个地址的赋值情况，和上面一样 tracewrite 分析）  
![](https://bbs.kanxue.com/upload/tmp/994949_36VUPTUR98DQFCX.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_2U2X3HT8K9C9V8T.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_TV96BZ8CKV4VG7D.webp)  
9.memcpy (目标地址，拷贝地址，len)（000AD960 MOVS R0, R6 分析这个地址的 r0 是 0x402d950c 的时候，并且看拷贝地址，再次找拷贝地址的值的来源）  
![](https://bbs.kanxue.com/upload/tmp/994949_ST8MS4PVKJBB3NC.webp)  
10. 发现两处来源，最终根据 r1 定位到第二处。因为 r1 0x4037fcc0 存在 0xbc 的赋值（最终发现 key 也是 xor 来的，固定 hex xor 未知 hex）  
![](https://bbs.kanxue.com/upload/tmp/994949_8FDDFKY8AN97CKW.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_9HD4EMPXNSF8RZE.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_B6CCSAMPTPRVPW2.webp)  
11. 接下来找 xor 另外部分的来源 0x402d952c（可以发现 0x402d952c 的内容，也就是 xor 部分的前半部分和 a9 的前缀一样 c0ee827）（0xad3dd 地址也是定位到 memcpy，所以找 r1）  
![](https://bbs.kanxue.com/upload/tmp/994949_D2MRTPFXKCTHGUD.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_RW6JKNJGF9Q3E56.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_UXF2NB6GKS4DAFF.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_Y9XMSPZYNDPGCVW.webp)  
12. 以上办法定位 r10xbffff4a4 时，定位到 libc 去了，找不到哪里赋值（搜索地址 0xbffff4a4，如下，对一些 push 存在该地址的时候进行断点打印该地址是否赋值）  
![](https://bbs.kanxue.com/upload/tmp/994949_KWWW6FJUH4BZGTM.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_M6R9FTBUYMZN4TH.webp)  
13. 发现了 r2=0xc0ee827（搜索 0xc0ee827 查找来源）  
![](https://bbs.kanxue.com/upload/tmp/994949_AK5624DFPURU7T3.webp)  
![](https://bbs.kanxue.com/upload/tmp/994949_EJAC4XKJG82W7ZT.webp)  
13. 结合 ida 分析，发现该位置就是 CRC 算法 0x2AA18，而入参则是设备信息  
![](https://bbs.kanxue.com/upload/tmp/994949_KQ68C338NTGR3HD.webp)  
14.key 后半部分'MXMYBS@H'则是固定值，如下：此数据来源于 app 中的文件解密得来，a2 还原时详细说明  
![](https://bbs.kanxue.com/upload/tmp/994949_FKYU58X9AJB37PT.webp)

6.a2 算法构造
=========

6.1 算法流程
--------

> 1. 获取 xor 随机数（获取时间戳 - 设置随机数种子 - 生成随机数 - 计算随机值并结果 + 1）（该值还体现在 mtgsig['a10']）  
> 2. 计算 hmac_sha1_key（pic 解密获取 a0 并重新排序 - xor appkey - xor 上面随机数）  
> 3.hmac_sha1 加密（body 是请求 body+mtgsig(除 a2)）  
> 4.hmac_sha1 的结果做 AES 加密（白盒 AES）(起初以为魔改 AES, 后续通过 DFA 差分法攻击拿到了 key)  
> 5.AES 结果后 16 位重新计算得最终结果

6.2 算法分析
--------

1.unidbg 模拟执行得到 a2= ... 2a e1 a6 5f cc ac 28 30; 直接 010 中查找哪里生成和赋值得位置；结果非常多，关键赋值位置附近是否还有对其他得几个值赋值，（搜索 str._=0x30）最终定位到 0x0e033 这个位置  
![](https://bbs.kanxue.com/upload/attach/202403/994949_SWB698SXKTJ8R6D.webp)  
2. 向上找到该地址所在得方法，0xdf04 方法入参是 0x10 字节大小内容，返回值则是构造好得 a2，经过代码分析发现是将后 16 位进行计算得到 a2  
3. 通过对 tracewrite 定位到赋值位置（入参 0x4037fe10 赋值得位置）  
![](https://bbs.kanxue.com/upload/attach/202403/994949_AR4HD6CS4PE6B4H.webp)  
4. 返回地址前是 blx r2，证明该方法内进行了赋值；定位发现 r2 的地址都一样，查看内部哪里有 str 之类的赋值指令，并且详细搜索 0xdf 对指定地址的赋值  
![](https://bbs.kanxue.com/upload/attach/202403/994949_UHYQWF6J535FQCB.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_YDMKVF98M66TUZX.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_HSE5QVCMQRHTXU7.webp)  
5. 接下来要找这个地址所在的方法，其中传入参数，最后能够生成结果的大方法；找方法的思路：从该地址向上找 bl 调用方法的地方，unidbg 进行调试分析返回值、找到生成的末尾，根据 lr 返回地址判断哪里调用的（在最末尾赋值处搜索赋值地址）  
![](https://bbs.kanxue.com/upload/attach/202403/994949_SSZ2GG4MKGQAJVT.webp)  
6. 分析以下包括赋值地址的位置，是否赋值完成，以及，该地址附近的方法是否方法的末尾，最终定位到 0x0db56 是方法末尾，而 0xDB52 则是方法调用位置 (hook 该地址看入参和出参)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_XTU5WZM77KDJUAC.webp)  
7. 参看入参哪里返回，根据 tracewrite 搜索对该地址写入位置，定位到 0x10044 地址，而 0x10044 地址所在的方法很像 sha1，hook 验证一下  
![](https://bbs.kanxue.com/upload/attach/202403/994949_F5WBEC8ZN9KR76X.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_W5WYB9P4VBFGBG4.webp)  
8. 通过 hook 0xFFA8 地址，发现就是 sha1，并且调用了两次，第一次入参是一段内容 + body，第二次入参则是一段内容 + 前一次 sha1 结果  
![](https://bbs.kanxue.com/upload/attach/202403/994949_E9F24KHNWM3PYQ8.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_HVBT4AKW3ABCKAZ.webp)  
9. 这里想看 ida 伪代码但是没有恢复混淆，所以看不了，但是到这里可以发现这个特征和 xxx 其中用到的一个算法很像，hmac 算法  
![](https://bbs.kanxue.com/upload/attach/202403/994949_2H9RKUPE7AXJTDK.webp)  
10. 直接 hook 调用 sha1 的方法，看入参和出参（通过在线算法工具测试）  
![](https://bbs.kanxue.com/upload/attach/202403/994949_VJVKZZPPFAGX54J.webp)  
11. 分析 key 得来源：除了 tracewrite 来定位对某个地址写入，还可以在 trea 日志中搜索（str._=0x6f.*0x402da280）  
![](https://bbs.kanxue.com/upload/attach/202403/994949_ZRCZHHCZ5NHSFVU.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_ZZUE29UKVX7G3RA.webp)  
12. 仔细分析会发现，0x0e3ad 地址异或后的结果在 0x0e3b1 地址进行异或 0x14

> 00000000 42 49 37 34 46 49 66 64 39 30 4b 6c 54 69 6c 31 |BI74FIfd90KlTil1|  
> 00000010 6d 30 69 37 46 46 2f 6e 30 62 68 59 69 68 4e 6b |m0i7FF/n0bhYihNk|  
> 00000020 72 4a 2b 75 |rJ+u|  
> xor  
> 00000000 39 62 36 39 66 38 36 31 2d 65 30 35 34 2d 34 62 |9b69f861-e054-4b|  
> 00000010 63 34 2d 39 64 61 66 2d 64 33 36 61 65 32 30 35 |c4-9daf-d36ae205|  
> 00000020 65 64 33 65 |ed3e|  
> xor  
> 0x14

13.0x0e3ad 地址其中一个内容是 appkey，另外一处并不指定，还有就是 0x14 怎么来的？(如下搜索 0x14 赋值指定地址位置，定位到地址，在 Ida 分析，发现如下特征: 时间戳作为随机数种子，生成随机数传入 0x11a58 调用的方法进行计算，得到的值在加 1)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_5MSMH9CR95DTNUM.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_9QFR2RX6YEZJ6X9.webp)  
14. 而另外一组 xor 的内容，通过 ldrb r0, [r0, r5]" r0=0x405b82dc 取值可以知道是在 0x405b82dc 这个地址进行获取内容，并且分析这几次的取值，发现是在这个地址内取几段数据拼接起来  
![](https://bbs.kanxue.com/upload/attach/202403/994949_8AUWH8GU3HBWCN8.webp)  
15. 通过 tracewrite 定位到 0xad96b 这个地址返回，而在这个地址上面调用的是 memcpy，明显是拷贝赋值，那么查找 r0=0x405b82dc 时的 memcpy，并且地址则是在 0xad96b 上面的 memcpy 的 r0 赋值处  
![](https://bbs.kanxue.com/upload/attach/202403/994949_ZK3CY7RNP84P2F7.webp)  
16. 定位到地址 0x405b81e0 (发现是 0x4037647b 这个地址赋值)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_AJWT2CUM9VBQUUP.webp)  
17. 而在 unidbg 中查看该地址的内容发现并不是内容开始位置，应该还有很多内容; 最终定位到 0x40376300 这个地址开头  
![](https://bbs.kanxue.com/upload/attach/202403/994949_423KZ3XAR5HC4RV.webp)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_VP6HJ8HXD9FS4MP.webp)  
18. 到这里基本确定该值也是固定得，具体解法并未深入分析  
![](https://bbs.kanxue.com/upload/attach/202403/994949_8FRSBN644C2PEC6.webp)

7. 获取 dfp 请求和获取 xid 请求
======================

[dfp]：[https://appsec-mobile.meituan.com/v5/sign](https://appsec-mobile.meituan.com/v5/sign)  
[xid]：[https://appsec-mobile.meituan.com/fingerprint/v1/info/report](https://appsec-mobile.meituan.com/fingerprint/v1/info/report)  
1. 重点是 data 的加密，分两部分，两个 base64 相加，而 main(41) 返回 data 数据  
2. 通过 unidbg 分析到 0x92653 这个地址进行了 iv 和 data 的异或（向下分析发现类似 aes 的查表法，白盒 aes）  
3. 多次 hook 发现有概率走了 aes 正常算法，但是 key 会改变，经测试发现，base64 前部分应该是 rsa 加密的 key，如果这部分不变则 key 不变  
4. 而正常的 aes 算法走的是 0x2ac11 地址，和上面正常的 aes_cbc 算法地址一致  
5.dfp 和 xid 得请求 body 算法一致，区别在于设备信息不同

8.a2 中 DFA 差分法获取 AES_KEY
========================

1. 根据 trace 日志，定位到 10 轮的循环  
2. 其中 0x2bdfd 地址找到对 input 内容的读取 (这里注释 Input 内容就是上面 hamc_sha1 的结果前 16 字节)  
3. 通过对 input 内容左移后找到表中数据，最终通过表的数据进行异或得到 AES 查表法中 TE 表的内容，之后进行异或得到，每一轮的结果（4 字节 input 数据获取到 4 个 TE 表的内容进行异或）  
4.0x2bea3 地址是开始 xor 得到 TE 表的数据，后续通过其他算法来模拟了异或运算，得到最终每一轮的结果  
5. 其中在 input 内容获取 TE 表内容时，和轮数还有当前 Input 内容的下标都关联了运算  
6. 差分法攻击是在第 9 轮时，随机修改 1 个字节得 Input，然后得到结果，通过系列公式计算（可参考资料差分故障攻击的原理. pdf）  
7. 使用工具：[https://github.com/SideChannelMarvels/JeanGrey/tree/master/phoenixAES；参考文章：https://bbs.kanxue.com/thread-254042.htm](https://github.com/SideChannelMarvels/JeanGrey/tree/master/phoenixAES%EF%BC%9B%E5%8F%82%E8%80%83%E6%96%87%E7%AB%A0%EF%BC%9Ahttps://bbs.kanxue.com/thread-254042.htm)  
8. 其中重点定位到第九轮，和 AES 算法得结果，也就是第 10 轮后得结果；使用方式：先获取正确得结果，在随机修改第九轮入参中 Input 得一个字节，并获取错误得故障结果；使用 python 脚本进行 key10 得获取；如果正确会获取到 key，否则得不到 key，如果得不到 key 则分析 aes 算法前后，是否选择得结果不对  
![](https://bbs.kanxue.com/upload/attach/202403/994949_4MDCMBSDG7ZGJ67.webp)  
9. 通过 key10 获取 key 得工具链接：[https://github.com/SideChannelMarvels/Stark](https://github.com/SideChannelMarvels/Stark)  
![](https://bbs.kanxue.com/upload/attach/202403/994949_W5A6MS9KSK6DMKD.webp)

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#协议分析](forum-161-1-120.htm) [#逆向分析](forum-161-1-118.htm) [#基础理论](forum-161-1-117.htm) [#其他](forum-161-1-129.htm) [#混淆加固](forum-161-1-121.htm)