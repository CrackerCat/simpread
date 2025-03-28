> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286228.htm)

> [原创] 某加密的 a26 生成分析全过程

本帖本着分享逆向思路的初心而发，实际内容是自己随笔录，有些部分已脱敏，本身也是一个思路分析贴，所以不会含有源码

a26 有 8 套算法，这是其中一套，当然我不会说第几套![](https://bbs.kanxue.com/view/img/face/030.gif)

需要准备：
=====

**qbdi-python**：[https://bbs.kanxue.com/thread-276523.htm](https://bbs.kanxue.com/thread-276523.htm)

**traceFunction**: [https://github.com/CookingDuck/TraceFunction](elink@592K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6o6L8$3!0C8K9h3&6Y4c8s2g2U0K9#2)9J5c8W2c8J5j5h3y4W2c8Y4g2F1j5%4c8A6L8$3^5`.)

**ida**：我很少看 ida，只是辅助吧，因为现在的 so 混淆太严重，手动去花费时费力，而且好多地方也反编译不出来，我的选择，铁锭硬刚汇编

**unidbg**：unidbg 的作用是为了看真机的算法，unidbg 是否执行，如果有，在还原算法的时候，可以用 unidbg 代替

开始：
===

已知这个加密中会有 26 个参数，其他参数的获取相对简单，所以目标也就是第 26 个参数

通过 unidbg 的插件 traceFunction，找到 protobuf 第一次生成的位置

![](https://bbs.kanxue.com/upload/attach/202503/994069_JUW429ACA483D78.jpg)

在 ida 中查看，发现是一个跳转

![](https://bbs.kanxue.com/upload/attach/202503/994069_UTVDAZTH6W6TR2A.jpg)

通过对比 frida 的日志传参和结果，发现与 ida 的伪 c 结果一致，

所以直接 hook `0xc4c70`即可

![](https://bbs.kanxue.com/upload/attach/202503/994069_PBQ75BNSQ8VTGC9.jpg)

计数器发现，这个函数被调用了 4 次（unidbg 是 3 次），通过搜索最后一节 12 字节的数据，有两个结果，找到第一个，

![](https://bbs.kanxue.com/upload/attach/202503/994069_F5M4G464S627P4A.jpg)

日志的获取：
======

so 的逆向肯定离不开日志，我使用的是 yang 神开源的 qbdi-python，

具体看 yang 神的文章，感谢开源

目前 qbdi 的工具也有很多，也有速度很快的，但是我感觉都是一样，除非是快了点，打把 lol 就完事了

我这里 trace 了三份日志，实际分析只用了两个，

我的逆向习惯，第一份分析，第二份验证

因为换了设备，第一分日志是分析了前两个字节，后面都是以第二份日志分析为主

**如果有需要，请留言，**

**如果有需要，请留言**

**如果有需要，请留言**

**因为我分析的日志做全寄存器输出，导致日志很大，上传至云盘再分享出来**

a26 结果记录
--------

1、d2 9e 4b 80 d8 62 cd e8 87 ea 10 bc（第一次 trace）

2、e2 55 48 8a e8 a9 ce e2 b7 21 13 b6 （第二次 trace）

第一部分：
=====

真机 trace 的 a26 `d2 9e 4b 80 d8 62 cd e8 87 ea 10 bc`

一般加密会在组装的附近，所以，先搜`d2`

![](https://bbs.kanxue.com/upload/attach/202503/994069_FYHZNG9YUR69YZV.jpg)

因为同时监控了读写，所以，直接搜 = ab 就可以

这个距离拼接比较近，记录一下，

= d2    8[w] 7d85d0a600   1[w] 7d85f94d20

如法炮制

= 9e    8[w] 7d85d0a600   1[w] 7d85f94d21  
= 4b    8[w] 7d85d0a600   1[w] 7d85f94d22  
= 80    8[w] 7d85d0a600   1[w] 7d85f94d23

这个时候就已经可以看到规律了，直接搜这个地址吧

![](https://bbs.kanxue.com/upload/attach/202503/994069_PTGFFCGHBA24N6S.jpg)

可以直接得到 a26 的写入，可惜我这次 trace，硬扣汇编，板凳流

日志就可以看到，从 `0x7d85d0a600` 读取的 0xd2 ，看一下其他的是否也是这种情况，

`0x7d85d0a600`是不是中间存储

![](https://bbs.kanxue.com/upload/attach/202503/994069_V7GS6BGF9X6N5A9.jpg)

所以，`0x7d85d0a600`是存储 a26 每次计算的结果的，那就好办了，把 read 改成 write，看看写入就可以把 a26 逐个击破了

1、d2（trace1）
------------

![](https://bbs.kanxue.com/upload/attach/202503/994069_C425HPV43BJHHYG.jpg)

0xd2 是由 0x71 和 0xa3 异或而来，追一下 x8 和 x10 怎么来的，记录吧

### 0x71

![](https://bbs.kanxue.com/upload/attach/202503/994069_8AM7KMD4TDJBKNQ.jpg)

可以看到从 `7d85d0a670`读取的值，搜索这个地址什么时候写入的

![](https://bbs.kanxue.com/upload/attach/202503/994069_UW9VY9PVBZRBTFP.jpg)

从这里可以看到 `0x71`从 `7d85d095b8`读取一个字节，写入 `7d85d0a670`

看看什么时候写入

从`7d85d0b128`读取，继续跟踪

![](https://bbs.kanxue.com/upload/attach/202503/994069_BSE55U8SWVRNFAN.jpg)

从 `7d85d0b1c28`读取，继续跟踪

![](https://bbs.kanxue.com/upload/attach/202503/994069_XR2ZSN5UDN4YM2P.jpg)好像有很多，直接一步到位

![](https://bbs.kanxue.com/upload/attach/202503/994069_5GWXC9UDZSA4ST7.jpg)

从`7d85d0b128`读取，继续跟踪

![](https://bbs.kanxue.com/upload/attach/202503/994069_B6GEWR3FZGQJ36Z.jpg)

从 `7d85d0b1c28`读取，继续跟踪

![](https://bbs.kanxue.com/upload/attach/202503/994069_UVATFK232Z7HKHE.jpg)好像有很多，直接一步到位

![](https://bbs.kanxue.com/upload/attach/202503/994069_N7QB8MJPU6YFUNC.jpg)

可以看到，最终结果是函数返回的，那我们看看这个函数是什么

![](https://bbs.kanxue.com/upload/attach/202503/994069_2MTVU5HEWSSMQ6V.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_PHFG8XEGVNHAXSU.jpg)

减 4 位，然后往上找一位，这个是函数入口

![](https://bbs.kanxue.com/upload/attach/202503/994069_P2V4PN7AS8XQGWG.jpg)

ida 看一眼

![](https://bbs.kanxue.com/upload/attach/202503/994069_ENXUDE4UQZ4F6FJ.jpg)

#### 结论

是个随机数，所以 0x71 是由随机数（`0x71314d5e`），反转 (`0xe54d3171`)， 取最低位`0x71`

`0x71314d5e`[0]

### 0xa3

像 0x71 一样，如法炮制

![](https://bbs.kanxue.com/upload/attach/202503/994069_Q3QXMDYC2M8MK7V.jpg)

从 `7d85d0a688`读取，跟踪看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_JFUHJXZ46UU6E6E.jpg)从 `7d85f94bdb`读取，跟踪看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_FQB4WVNAGSZZUBE.jpg)

从 `7d89a8e2e3`读取，跟踪看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_UG6X2VT8P8XRXV6.jpg)

又是加载一个字节，跟踪看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_68CDN66EZ7K3GH3.jpg)

再次搜索，发现搜索 write 失效了，只能硬着头破跟了

![](https://bbs.kanxue.com/upload/attach/202503/994069_42S292SKTK9RN3B.jpg)

还是有一定规律在里面的，分析分析

![](https://bbs.kanxue.com/upload/attach/202503/994069_9J9RUKRV5VQRCYR.jpg)

搜索大法失效 - -！跟踪看看 x21 寄存器怎么赋值的吧

![](https://bbs.kanxue.com/upload/attach/202503/994069_XR9MUBCGNHKDXUG.jpg)

通过搜索，唉，跟上面一样，指令执行这么多，x21 都没动过 - - ！

![](https://bbs.kanxue.com/upload/attach/202503/994069_9WCKHZX33SR3HGU.jpg)

可以看到，x21 是由地址偏移来的

现在捋一下

`0x7dc912258c` ldr `0x0` -> `0xde`

`0x7dc912258c` ldr `0x1` -> `0x06`

`0x7dc912258c` ldr `0x2` -> `0xaf`

`0x7dc912258c` ldr `0x3` -> `0xa3` (目标值)

`0x7dc912258c` - `0x7dc9122590` 存储着读取的值 `0xde06afa3`

![](https://bbs.kanxue.com/upload/attach/202503/994069_K6T8TVJQ8EWN39Z.jpg)

搜索这个值看看，当前行号 795471

![](https://bbs.kanxue.com/upload/attach/202503/994069_5H63W5UW49U8SPJ.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_GYWG5GNJZAP4CQG.jpg)

跟踪这个地址看看 memory write at 7d85d08f5b

![](https://bbs.kanxue.com/upload/attach/202503/994069_NZ9YFFQ479KE9K7.jpg)

最终在这里生成

通过不停的滑动  
![](https://bbs.kanxue.com/upload/attach/202503/994069_ZNTAXHX4U5MTZHG.jpg)

这个看起来像是加密的地方 `0x93cf0`

![](https://bbs.kanxue.com/upload/attach/202503/994069_3U3HUTB6TGYYHS6.jpg)

ida 中，

![](https://bbs.kanxue.com/upload/attach/202503/994069_AFJQQMXSXMHT8H4.jpg)

可以在伪 c 中，看到两个常量，百度搜一下

![](https://bbs.kanxue.com/upload/attach/202503/994069_YGNKPWKNEQPNFB3.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_UJ3AVPWDS8WU4DX.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_VAVGFB3MBPV79FU.jpg)

都指向了 sm3

#### 验证 sm3

既然他可能是 sm3，在 xa 的别的地方也用到了 sm3，看看用的是否是同一个函数, 又到了小工具的高光时刻

![](https://bbs.kanxue.com/upload/attach/202503/994069_HN8RG54YT4SMJHJ.jpg)

<table width="500"><tbody><tr><td width="250"><p>14</p></td><td width="250"><p>48 e3 40 a3 ad 76</p></td></tr></tbody></table>

通过搜索 a14 的值，第一次出现在`0x8ff88`中，ida 中看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_6A7FS948MX2TBTF.jpg)

看不到什么，结合汇编分析吧，看看 lr，返回值给了谁

![](https://bbs.kanxue.com/upload/attach/202503/994069_Y9D5KMCYJB2DK8F.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_Y92FF826Q5Z5PTX.jpg)

通过分析 traceWrite 在写入`0xbfffc510`这个地址的时候，断住，然后看汇编，往上找调用（也就是不连续的地址处），同时用 unidbg 将可疑的函数断住看情况，最终分析出，这个函数是 sm3

![](https://bbs.kanxue.com/upload/attach/202503/994069_27N3NP3BN8WRXSJ.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_W53MYE24REZVDBH.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_BQ4PYR3UPXYY39E.jpg)

x0 是待加密值

x1 是长度

x2 是空的，结合写入的日志，这个肯定是 buffer 了，

![](https://bbs.kanxue.com/upload/attach/202503/994069_9MKY2AFW5TKMNW8.jpg)

在这个地址断住

![](https://bbs.kanxue.com/upload/attach/202503/994069_BB8VSP79VKW5XE3.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_SXKVGS4RPAKRHYS.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_56SDWUQH5NHEPE2.jpg)

是标准的 sm3 了

回到 `0xde06afa3`

![](https://bbs.kanxue.com/upload/attach/202503/994069_Y3E8N2NVF9D3W4T.jpg)

通过分析，这个地址，确实在`0x942d4`函数内（因为日志在入参和返回中，且日志行数变化不大）

想了想，不给 frida 代码，解释的好像很苍白。。。dddd(懂得都懂)，so 打码了

```
function hook_so_x() {
  var soAddr = Module.findBaseAddress("****.so")
  console.log("模块基址：", soAddr)
  var four_add = soAddr.add(0x942d4)
  let result;
  let num = 0;
  Interceptor.attach(four_add, {
    onEnter: function (args) {
      result = args[2]
      console.log(result, num, "入参2：", hexdump(args[2]))
      console.log(result, num, "入参0：", hexdump(args[0]))
      console.log(result, num, "入参1：", args[1])
    }
  })
 
  Interceptor.attach(soAddr.add(0x95b70), {
    onEnter: function (args) {
      console.log(result, num, "入参的出参：", hexdump(result))
      num += 1
    }
  })
}
 
 
function call_x_func() {
  let baselibEncryptor = Module.findBaseAddress("libmetasec_ov.so");
  let addr_8bb80 = baselibEncryptor.add(0x8c4a0);
  let str0 = "https://api16-core-useast5.us.tiktokv.com/aweme/v1/aweme/post/?source=0&user_avatar_shrink=96_96&video_cover_shrink=248_330&max_cursor=0&sec_user_id=MS4wLjABAAAAMTGT9uzD56ixJVlbJ4A9JUd_OW_qQVYce5511ce45mISCfOPDBn5ZZAsLODCkaAk&count=9&locate_item_id&sort_type=0&iid=7294518724337190698&device_id=6661258642405328389&ac=wifi&channel=googleplay&aid=1233&app_name=musical_ly&version_code=310901&version_name=31.9.1&device_platform=android&os=android&ab_version=31.9.1&ssmix=a&device_type=SM-G977N&device_brand=samsung&language=en&os_api=28&os_version=9&openudid=a6729e370acc5af5&manifest_version_code=2023109010&resolution=1600%2A900&dpi=320&update_version_code=2023109010&_rticket=1701144971610&is_pad=0¤t_region=US&app_type=normal&sys_region=US&mcc_mnc=31016&timezone_name=America%252FPhoenix&carrier_region_v2=310&residence=US&app_language=en&carrier_region=US&ac2=wifi5g&uoo=0&op_region=US&timezone_offset=-25200&build_number=31.9.1&host_abi=arm64-v8a&locale=en®ion=US&ts=1701144975&cdid=88487472-748a-4f9c-b945-8ebe4052bc18"
  let arg0 = Memory.allocUtf8String(str0);
 
  let str1 = decodeURIComponent("x-tt-bypass-dp%0D%0A1%0D%0Asdk-version%0D%0A2%0D%0Apassport-sdk-version%0D%0A19%0D%0Ax-ss-req-ticket%0D%0A1717052656600%0D%0Ax-vc-bdturing-sdk-version%0D%0A2.3.0.i18n%0D%0Ax-tt-dm-status%0D%0Alogin%3D0%3Bct%3D0%3Brt%3D7%0D%0Acontent-type%0D%0Aapplication%2Fx-www-form-urlencoded%3B%20charset%3DUTF-8%0D%0Ax-ss-stub%0D%0AE10ADC3949BA59ABBE56E057F20F883E%0D%0Acontent-length%0D%0A81%0D%0Ax-tt-trace-id%0D%0A00-c8504d6c010de75c6a96840a3f9a04d1-c8504d6c010de75c-01%0D%0Auser-agent%0D%0Acom.zhiliaoapp.musically%2F2022900010%20(Linux%3B%20U%3B%20Android%208.1.0%3B%20en_US%3B%20AOSP%20on%20taimen%3B%20Build%2FOPM1.171019.011%3B%20Cronet%2FTTNetVersion%3A55e3b3c8%202023-03-20%20QuicVersion%3Ad298137e%202023-02-13)%0D%0Aaccept-encoding%0D%0Agzip%2C%20deflate\n")
  let arg1 = Memory.allocUtf8String(str1);
 
  var sub9ad50 = new NativeFunction(addr_8bb80, "pointer", ["pointer", "pointer"]);
  var result = sub9ad50(arg0, arg1)
  console.log(result.readCString())
}
```

因为我多次不改参数调用和发现 a3 似乎固定，想着可能与入参有关，于是在统一参数之后，

直接搜索`0xde06afa3`，可以看到是第五次（号有问题，入参是调用次数，出参是 + 1 次数），看看入参（4）

入参是 0x00000001

```
0x72ff34bd20 4 入参0：              0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
73020fab00  00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00  ................
73020fab10  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

python 复现一下看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_J3E3EADPJSUCTM2.jpg)

确实对上了，

a3 就是 0x 00 00 00 01 的 sm3 了

#### 结论：

`0xa3`是 0x00000001 的 sm3 最后一组结果的最后一位

`0xde06afa3`[3]

2、9e（trace1）
------------

![](https://bbs.kanxue.com/upload/attach/202503/994069_926D59NT8BMCQ87.jpg)

0x9e 是由 0x31 与 0xaf 异或 而来，分开追

### 0x31

![](https://bbs.kanxue.com/upload/attach/202503/994069_MN3BVHH3ZMSFD48.jpg)

搜索写入，![](https://bbs.kanxue.com/upload/attach/202503/994069_2K5DYX478HBMRZB.jpg)

结果有点多，搜索最近一次的写入，可以看到是从`0x7d85d095b9`地址取第一个字节，0x31

![](https://bbs.kanxue.com/upload/tmp/994069_B658XHYQEQK4UEM.jpg)

通过搜索发现，从 `0x7d85d095b8， 4` 读取四个字节，读了好几次，然后地址连续之后，对应的值是 `0x71314d5e`，看看在哪里生成

![](https://bbs.kanxue.com/upload/attach/202503/994069_AAU4DMMNPFHYSVT.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_6WXWBQ7KPM4BP9Y.jpg)

地址减一下，看看调用位置，懵逼了，这 Timi 的不是随机数吗，后知后觉，这不就是 d2 的随机数，亏我跟这么久，都快放弃了，还好没放弃，睡前在挣扎一下

#### 结论

取跟 d2 用的随机数一样的数（`0x71314de5`），取第 2 位 `0x31`

`0x71314de5`[1]

### 0xaf（这里因为换了设备，日志也换成 2，不影响前四个字节）

![](https://bbs.kanxue.com/upload/attach/202503/994069_EKGWWU7G7VYKKA5.jpg)

从 `0x77e110a688`读取值，为 0xaf，跟上去

![](https://bbs.kanxue.com/upload/attach/202503/994069_6GMJ4GF7AYWM4ZG.jpg)

又是加载了一个字节，看看能不能跟这个地址`0x77e0e4431a`

![](https://bbs.kanxue.com/upload/attach/202503/994069_KYJFPR5JXHUT77M.jpg)

继续跟

![](https://bbs.kanxue.com/upload/attach/202503/994069_JJYTT9NN2WSND2M.jpg)

继续

![](https://bbs.kanxue.com/upload/attach/202503/994069_WE5PKAHJ7RDBNMZ.jpg)

发现，跟到这个地址，线索就中断了，看看这个`0xaf`从哪来的

![](https://bbs.kanxue.com/upload/attach/202503/994069_7FNPA9E2V4CXE4G.jpg)

通过不看最后一位，发现，这个也是连续的读取， `0x782278306c， 4`，又仔细一看，这不是 00 00 00 01，sm3 的结果吗  
![](https://bbs.kanxue.com/upload/attach/202503/994069_H26FYD9ZR4JCRUV.jpg)

#### 结论：

`0xaf`是从`0xde06afa3`取的第 3 位

`0xde06afa3`[2]

3、48（trace2）
------------

继续回到搜索关键词

`memory read at 77e110a600, data size = 4,`

![](https://bbs.kanxue.com/upload/attach/202503/994069_Y8TKMQBKA67ENZG.jpg)

可以看到`0x48`是直接读出来的，看看哪里读的

![](https://bbs.kanxue.com/upload/attach/202503/994069_6T7WYBZMGJBA5Y9.jpg)

搜索写入，又回到异或分支

### 0x4e（待验证）

`memory write at 77e110a670`

![](https://bbs.kanxue.com/upload/attach/202503/994069_CMVK8JETCSXNXCC.jpg)

跟踪，又是加载一个字节

搜索 `0x77e11095ba`依旧没有目标值，所以，还是不看最后一位搜索

![](https://bbs.kanxue.com/upload/attach/202503/994069_2C3HXTDVQ5WMEP9.jpg)

#### 结论：

其实第一次搜索，就已经看出来端倪，这个还是随机数（`0x41fa4e54`）的第 3 位 `0x4e`

`0x41fa4e54`[2]

### 0x06（待验证）

看看 0x06 从哪来的，结合上面规律，大概率还是`0x00 00 00 01`的 sm3 的最后 4 字节的第二位

![](https://bbs.kanxue.com/upload/attach/202503/994069_Y9V9CV8KNU7BNCQ.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_HD75QJZ37PEWX3Q.jpg)

可以看到，读取的地址都是一样的

#### 结论：

所以 `0x06`应该就是  `0xde06afa3`的第二位

`0xde06afa3`[1]

4、8a（trace2）
------------

![](https://bbs.kanxue.com/upload/attach/202503/994069_XHPUGPBGYEAF6F9.jpg)

搜索关键词 `memory read at 77e110a600, data size = 4,`按照规律，这次应该也是随机数与 sm3 的结果异或，验证看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_489VGAS3FABCFXZ.jpg)

#### 结论：

按照规律，随机数该取 [3]，sm3 取 [0]

随机数：41 fa 4e 54

sm3:     de 06 af a3

对上了，确实如此

前四个字节总结
-------

随机数：41 fa 4e 54

00000001_sm3:     de 06 af a3

第一个：0x41 xor 0xa3 = e2

第二个：0xfa xor 0xaf = 55

第三个：0x4e xor 0x06 = 48

第四个：0x54 xor 0xde = 8a

手动计算后，对比前四个字节，确实可以对上

第二部分
====

5、e8（trace2）
------------

![](https://bbs.kanxue.com/upload/attach/202503/994069_DMASGU9CZMNC6AA.jpg)

也是异或出来的，分开跟踪看看

### 0x41

这个 0x41 的地址有点眼熟，好像是随机数。。。

![](https://bbs.kanxue.com/upload/attach/202503/994069_99B9PGMK37H2PN2.jpg)

可以看到从 `0x77e11095b8`读取一个字节

![](https://bbs.kanxue.com/upload/attach/202503/994069_HG5C2M4RPNUKNDX.jpg)

随机数的反转

![](https://bbs.kanxue.com/upload/attach/202503/994069_VGAACW4YJANTS9T.jpg)

读取随机数

![](https://bbs.kanxue.com/upload/attach/202503/994069_ECNXW85B628U4EX.jpg)

继续加载

![](https://bbs.kanxue.com/upload/attach/202503/994069_CC64KKKNUS7GYFV.jpg)还有读取，继续跟

![](https://bbs.kanxue.com/upload/attach/202503/994069_VWFH4K7VTMBSWS3.jpg)

继续

![](https://bbs.kanxue.com/upload/attach/202503/994069_DRF34Z875FQE9YG.jpg)

最终跟到 `0x77e110c008`接收了随机数的返回值，

![](https://bbs.kanxue.com/upload/attach/202503/994069_HZ58F244DK3FBKY.jpg)

地址 -4， 来到了 调用随机数的方法

#### 结论：

读取了随机数 `0x41fa4e54`的第 1 个字节

`0x41fa4e54`[0]

已经知道了，为什么还要跟呢，因为日志不一样，d2 跟随机数的时候，用的是 trace1 的日志

### 0xa9

![](https://bbs.kanxue.com/upload/attach/202503/994069_N6NKNF88SWV69KD.jpg)

跟踪看看，哪里写入

![](https://bbs.kanxue.com/upload/attach/202503/994069_8NE7HCARTJDR4CT.jpg)

加载了一个字节，搜索看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_WVCBJ5A9XDBUQ76.jpg)

通过搜索 `memory read at 77e0e4431`，可以看到规律

从`0x77e0e44310`开始， `0x77e0e44318`开始是 sm3 结果的最后一组

![](https://bbs.kanxue.com/upload/attach/202503/994069_ZBQD2QHK4JZ4BRV.jpg)

先不考虑这么多，继续往上跟，跟到这里，没有写入了，看看这个 `0xa9`怎么来的

![](https://bbs.kanxue.com/upload/attach/202503/994069_ZX87S76NER3VA29.jpg)

通过搜索，也是一组数据，直接搜搜看

![](https://bbs.kanxue.com/upload/attach/202503/994069_2NC4M4NU6DQHDQZ.jpg)

最终异或出来，并且组成 8 字节的，有点像分组运算，这里应该是加密算法了，

看执行流程，感觉像是 sm3 加密，往上搜了一下 sm3 的入口，和 sm3 的结束，发现，这个值生成在 sm3 中，

![](https://bbs.kanxue.com/upload/attach/202503/994069_F2D8PKHDDGM5BQG.jpg)

通过查看寄存器，长度是 0x10 ，大概率是 body 了，

![](https://bbs.kanxue.com/upload/attach/202503/994069_9SQRHM7MU84WR5S.jpg)

python 实现一下，确实如此，body 的 sm3 的最后一个分组

#### 结论：

body 经过 sm3 加密后（`b68053a9`），取的最后一个字节

`b68053a9`[3]

6、a9（trace2）
------------

![](https://bbs.kanxue.com/upload/attach/202503/994069_T7XSKCNGJVJEMCM.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_3RUMUMK6MHKQXDZ.jpg)

老规矩，先定位到位置，然后 read 改成 write

### 0xfa

看到 0xfa，想到了随机数 `41 fa 4e 54`

还是跟一下看看吧

![](https://bbs.kanxue.com/upload/attach/202503/994069_KGK8Z4NFHNKT35Q.jpg)

跟到最后，也是这样，所以还是同一个随机数了

#### 结论：

随机数的第 2 个字节

`41 fa 4e 54`[1]

### 0x53

![](https://bbs.kanxue.com/upload/attach/202503/994069_VMJZ3SJEFDTJGFA.jpg)

跟踪生成吧

![](https://bbs.kanxue.com/upload/attach/202503/994069_7QNFDEU82Z52JZU.jpg)

读取一个字节，继续跟踪看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_U2V4J6WZ3CD2HBQ.jpg)

继续跟踪

![](https://bbs.kanxue.com/upload/attach/202503/994069_YYVJ28XUR8W89SK.jpg)

跟到这里，就到顶了，看看`0x53`怎么来的

ps：结合上面的规律，这个`0x53`应该也是 body 的 sm3 结果，也确实在里面，为了严谨，还是自己跟吧

![](https://bbs.kanxue.com/upload/attach/202503/994069_A7JCFAEKGPHU7TF.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_7DF6PTNSFSSDCUF.jpg)

确实是 body_ sm3 的结果，因为地址连续，且顺序一样

#### 结论：

body 经过 sm3 加密后（`b68053a9`），取的第 3 个字节

`b68053a9`[2]

7、ce（trace2）
------------

按照规律，这个应该是

随机数：`41 fa 4e 54`[2]

body_sm3: `b6 80 53 a9`[1]

![](https://bbs.kanxue.com/upload/attach/202503/994069_JN76BMPFKGCFMTK.jpg)

可以自己尝试逆向看看

#### 结论：

是的

8、e2（trace2）
------------

按照规律，这个应该是

随机数：`41 fa 4e 54`[3]

body_sm3: `b6 80 53 a9`[0]

![](https://bbs.kanxue.com/upload/attach/202503/994069_AGX64GMZWWQ6EF3.jpg)

#### 结论：

是的

第二组字节总结
-------

随机数：41 fa 4e 54

00000001_sm3:     b6 80 53 a9

第一个：0x41 xor 0xa9 = e8

第二个：0xfa xor 0x53 = a9

第三个：0x4e xor 0x80 = ce

第四个：0x54 xor 0xb6 = e2

手动计算后，对比前四个字节，确实可以对上

第三部分
====

9、0xb7（trace2）
--------------

![](https://bbs.kanxue.com/upload/tmp/994069_FQPCRFA7CVUUWDD.jpg)通过搜索，看到同样是异或，看到`0x41`还有规律，最后一部分，不会是 query 的 sm3 的异或吧。。

### 0x41

ps：这个不会还是随机数吧，根据前面的经验。。。

跟上看看

![](https://bbs.kanxue.com/upload/attach/202503/994069_T5JNUE398A2H486.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_QW852MQBUQX28AC.jpg)

还是随机数，只不过这里是被反转了，上面还有 rev 反转，其实通过内存地址，也看出来了，跟随机数用一个地址

#### 结论：

随机数：`41 fa 4e 54`[0]

### 0xf6:

![](https://bbs.kanxue.com/upload/attach/202503/994069_EHRX6KVZK9XWNJG.jpg)

`0xf6`的开始

![](https://bbs.kanxue.com/upload/attach/202503/994069_83ASFVQK9X9NFA9.jpg)

继续跟

![](https://bbs.kanxue.com/upload/attach/202503/994069_PHPNJ7JH6XBKB8V.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_69TZ6644H9NWS25.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_34HTJQHD7HG7N83.jpg)

跟到这里，就没有结果了，看看规律

![](https://bbs.kanxue.com/upload/attach/202503/994069_7GDYRZHKBUBHCUK.jpg)

![](https://bbs.kanxue.com/upload/attach/202503/994069_3ZCPGBXW6QZD9B4.jpg)

雀食，就是 quert 的 sm3 的最后

#### 结论：

query_sm3 的最后一组：`e2 5d db f6`[3]

剩下部分：
-----

结合上面规律，最后一组可能是这样

随机数：41 fa 4e 54

query_sm3:    e2 5d db f6

第一个：0x41 xor 0xf6 = b7

第二个：0xfa xor 0xdb = 21

第三个：0x4e xor 0x5d = 13

第四个：0x54 xor 0xe2 = b6

1.  最后一部分：`b7 21 13 b6`，正好对上，所以就不需要找了
    

结尾
==

吐槽：现在的圈子有个现象，分享思路的越来越少了，取而代之的就是发帖、引流、拉群、开星球、卖课，再也没见到相对于龙哥之外质量更好的文章了，强烈推荐一下龙哥星球，当然我的质量也不行哈哈哈，我这是随笔录和分享思路

[[招生] 科锐逆向工程师培训 (2025 年 3 月 11 日实地，远程教学同时开班, 第 52 期)！](https://bbs.kanxue.com/thread-51839.htm)

[#逆向分析](forum-161-1-118.htm)