> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280562-1.htm)

> [原创]Directory Opus 13.2 逆向分析

[原创]Directory Opus 13.2 逆向分析

2024-2-18 20:10 16323

### [原创]Directory Opus 13.2 逆向分析

 [![](http://passport.kanxue.com/upload/avatar/361/947361.png?1678883702)](user-home-947361.htm) [Sw1ndler](user-home-947361.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif) 2024-2-18 20:10  16323

目录

*   [Directory Opus 13.2 逆向分析](#directory-opus-132-逆向分析)
*            [Ⅰ. 写在前面](#ⅰ写在前面)
*            [Ⅱ. 整体思路](#ⅱ整体思路)
*            [Ⅲ. 分析](#ⅲ分析)
*                    [1. 完整性校验](#1-完整性校验)
*                    [2. 程序证书替换](#2-程序证书替换)
*                    [3. 注册以及解决证书存根验证问题](#3-注册以及解决证书存根验证问题)
*                    [4. 公钥完整性检测](#4-公钥完整性检测)
*            [Ⅳ. 总结](#ⅳ总结)

Directory Opus 13.2 逆向分析
========================

> 文章仅供学习使用，切勿用于非法用途。如有造成侵权，请及时联系作者处理。

Ⅰ. 写在前面
-------

DIrectory Opus 是一款非常优秀的 Windows 文件多功能资源管理器，下面是来自官网的一个简单介绍：

> “Windows 系统自带的资源管理器固然好用，但是对于需求很多的用户来说它就远远不够，因此专业且强大的文件管理器成为很多用户心目中的选择，而 Directory Opus 就可以完美的替代系统默认的资源管理器，成为你管理资源中更好的帮助。与系统自带的相比它有更高度自由的界面定制，1000 位使用者中有 1000 种喜爱，而它 100% 自定义性可以完美的满足每一位使用者，让大家都可以使用的满意。与系统自带的相比它使用起来更简单，不需要学习任何复杂的脚本或者是技术就可以上手使用，运作方式与 Explorer 或许有些相似，人人都可以快速的使用它。与系统自带的相比它还有更多出色的功能，如计算文件大小、切换格式、模式选择、FTP 链接、预览功能、备份与恢复等等，这些功能可以使操作文件变得更加轻而易举，不管是需要功能强大还是简单易用，它都可以成为大家心目中的首选。”

但是有朋友指出再其早期版本存在资源占用高的问题，提到 “资源占用” 的问题也是瞬间激起了笔者对其最新版本进行逆向分析兴趣。所以有了这篇文章。

网络上并没有对 DIrectory Opus 进行逆向分析的一些较为详细的文章，比较早的是 [cntrump](https://bbs.kanxue.com/user-home-48065.htm) 前辈对 DIrectory opus 9 版本的一些证书提取的思路。

> [[原创] 偷出 Directory Opus 9 的授权证书 -- 另类提取被 Thinapp 打包的文件 - 软件逆向 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-120978.htm)

此文章并没有生成注册机，而是使用其 12 版本注册码进行注册。文章不提供下载，读者如果有需求可与笔者进行交流学习。笔者水平有限，如果文章内容有错误或者读者有更好的思路，还请不吝赐教。

Ⅱ. 整体思路
-------

笔者在之前使用的 12.3 版本的 DO 使用的是 **文件替换 + 证书验证** 的方式，推测可能使用了类似 RSA 公钥替换攻击，我们可以沿用这一方法。这里笔者从网上下载了之前公布的一些破解版。

![](https://bbs.kanxue.com/upload/attach/202402/947361_7KU94DMTDJF9SGM.jpg)

可以看到对原始的几个文件进行了替换。其中`Language`和`Viewers`文件夹从名字上就容易猜到不是我们主要进行逆向的目标，而`dopuslib.dll`可以推测其提供一些库函数供其他逻辑调用，根据文件大小可以猜测程序主要逻辑在`dopus.exe`

这里一个思路是：使用 12.33 版本的源文件与被 patch 后的文件进行 bindiff 比对，但因为大版本的更新，需要将 12.33 两个版本进行比对，然后移植到 13.2 版本上，不敢说这样一定会减少逆向的工作量，所以 bindiff 比对这里只作为一种参考辅助。

Ⅲ. 分析
-----

### 1. 完整性校验

首先这里笔者想关闭`dopuslib.dll`和`dopus.exe`的 ASLR 时很快就遇到了完整性校验问题。

![](https://bbs.kanxue.com/upload/attach/202402/947361_PRPEWC32E7JVKBS.jpg)

可以猜测进行完整性校验时可能会使用`CreateFilew`，使用 xdbg 对其下日志断点，记录打开的文件

![](https://bbs.kanxue.com/upload/attach/202402/947361_3XREVWFPDEB3TF5.jpg)

将 <a >CreateFileW 调用日志 </a > 导出，可以发现其并没有打开`dopus.exe`或者`dopuslib.dll`，如下，推测其可能不是使用 CRC 的方式计算完整性。但是留意到程序多次打开了`dopus.cert`文件，推测是证书的存根。读者可以记录试用时的证书，与下文注册成功的`dopus.cert`进行对比。

![](https://bbs.kanxue.com/upload/attach/202402/947361_7JND2QGSRE4TZ2N.jpg)

这里联想到另一种使用 WIndows 受信组件验证方式，其核心 API:`WinVerifyTrust`，一个简单介绍：

> [【windows + 证书验证】 验证数字签名的方法 - mooooonlight - 博客园 (cnblogs.com)](https://www.cnblogs.com/mooooonlight/p/14034592.html)

同样，在其 PE 结构中也有相应字段暗示：

![](https://bbs.kanxue.com/upload/attach/202402/947361_SFUMWKGB38Y3M7Z.jpg)

对`WinVerifyTrust`下日志断点，记录感兴趣的返回地址以及验证的文件。

![](https://bbs.kanxue.com/upload/attach/202402/947361_88M8ZDBGFJ9HQ86.jpg)

断点被触发，简单整理

![](https://bbs.kanxue.com/upload/attach/202402/947361_9FTPGTYHZBUYCA5.jpg)

两个位于`dopuslib.dll`一个位于`dopus.exe`，转到相应位置

**在`dopus.exe`中 VA—>0x140967A93**

![](https://bbs.kanxue.com/upload/attach/202402/947361_2JYJYTWYXPKHD23.jpg)

可以看到调用方式为间接调用，这里记录

```
WinVerifyTrust在dopus中的引用：
Direction   Type    Address Text
Down    r   140FD0490   mov     rax, cs:qword_1418DBA50
Down    r   140FD03D1   cmp     cs:qword_1418DBA50, rsi
Down    r   140A49941   cmp     cs:qword_1418DBA50, 0
Up  r       140967A7F   mov     rax, cs:qword_1418DBA50
Up  r       140967817   cmp     cs:qword_1418DBA50, 0
Up  r       14083784B   cmp     cs:qword_1418DBA50, 0
Up  r       14082353B   cmp     r9, cs:qword_1418DBA50
Up  w       14004401F   mov     cs:qword_1418DBA50, rax

```

**`dopuslib.dll`中 VA—>0x180035AE7 & 0x180035BDB** 同样可以看到对其调用方式为`GetProcAddress`间接调用

![](https://bbs.kanxue.com/upload/attach/202402/947361_JQ34NSDS59BCGTT.jpg)

![](https://bbs.kanxue.com/upload/attach/202402/947361_H4ADWNGURZN5XXB.jpg)

可以看到字符串进行了简单的拼接加密，简单解密脚本如下：

```
def decStr(input, index):
    str_list = list(input)[index % 5:]
    output = ''
    for i in range(0,len(str_list),5):
        if(ord(str_list[i]) != 33):
            output+=str_list[i]
    print(output)

```

可以看到解密出很多敏感字符串

![](https://bbs.kanxue.com/upload/attach/202402/947361_F98AYM5HES9J7BB.jpg)

**在`dopus.exe`中 VA—>0x140CC19C0 & 0x140CC1A5E**

![](https://bbs.kanxue.com/upload/attach/202402/947361_RYB9UR7KW59WEAR.jpg)

可以看到同样有一些字符串加密，其实是一个简单的 Base64 换表以及一个异或，这里不多赘述。

接下来将 patch 后的文件替换会原文件，并对五个检查位置下断点，获取 WinVerifyTrust 的返回值。读者可能觉得多次一举，因为`WinVerifyTrust`验证成功返回 0，其余值都是不授信的。

![](https://bbs.kanxue.com/upload/attach/202402/947361_483Y8K9B3VGA44G.jpg)

收集到的信息以及简单的 patch 如下：

![](https://bbs.kanxue.com/upload/attach/202402/947361_STEBMR7CNNR5FQ6.jpg)

![](https://bbs.kanxue.com/upload/attach/202402/947361_93KJP484MTUBJ3Z.jpg)

可以看到对于`0x140967A93`处的 patch 需要多加小心在第一调用时，返回`0x800B0100`但在第二次时返回成功。这里笔者对这一处的处理是在 patch 掉`0000000140967A8A`使其`eax`值为`0x800B0100`并禁用掉第二次的调用，简单对这个位置下断点，可以找到第二次调用位置`0x00140966DFE`

将如下所指区域的调用全部`nop`掉即可。

![](https://bbs.kanxue.com/upload/attach/202402/947361_8YK7BH33QBWFNXX.jpg)

patche 之后运行一段时间会发下，程序虽然没有退出，但是出现了异常：

![](https://bbs.kanxue.com/upload/attach/202402/947361_WTHTRQPYAUW5WWR.jpg)

会发现日志里多了一条`WinVerifyTrust`的调用，同样 patch 返回值为 0 即可。

![](https://bbs.kanxue.com/upload/attach/202402/947361_DWFGGHJ5PAKDPSV.jpg)

程序不在出现异常。

### 2. 程序证书替换

对于证书存放的位置，这里读者使用的方法是对使用对`GetWindowsTextW`下断

![](https://bbs.kanxue.com/upload/attach/202402/947361_D86SDUGGA6YYPE2.jpg)

进行栈回溯，比较简单就可以定位到`0000000140CEEB76`，函数`sub_140CEAF50`为证书验证逻辑。

![](https://bbs.kanxue.com/upload/attach/202402/947361_ZC2DY6UCRETJBMA.jpg)

跟进`0000000140CEEB76`，进行一些逆向分析可以发现函数`sub_140CE6EC0`的第三个参数为 key

![](https://bbs.kanxue.com/upload/attach/202402/947361_ETWEHE9WXV24UJR.jpg)

猜测其可能是使用一个全局变量来存储证书内容，对`qword_1418BF450`下写入断点，可以定位到位置`0000000140CCE331`，整个函数在完成初始化操作，在这里对`a1 + 0xDC0`下写入断点

![](https://bbs.kanxue.com/upload/attach/202402/947361_6V6ZSHGFYPJACPP.jpg)

定位到写入位置`0000000140CCE7A1`：

![](https://bbs.kanxue.com/upload/attach/202402/947361_95V5JG6Q2ZJFP86.jpg)

可见`0xDC0`位置数据来自于`dopuslib.dll`，跟进`dopuslib_23`

![](https://bbs.kanxue.com/upload/attach/202402/947361_VZK8BBCATWUV9VP.jpg)

这里出现了比较熟悉的字符串，跟进`dopuslib_10`可以发现其加载名为`RSA`的资源，秘钥存放在资源里，我们可以用 12.33 版本中的 RSA 公钥进行替换。看起来 DO 有两种的验证方式，而证书验证默认使用的是不是 RSA，我们可以 patch 为使用 RSA

![](https://bbs.kanxue.com/upload/attach/202402/947361_RTEMM2QJ3DUJAJE.jpg)

替换的公钥：

```
00 04 00 00 C1 22 5C 36 FD 3B 9C 3F D2 A5 B6 89 8B 0E C5 77 6A 78 23 F8 A1 E3 DB 15 94 D7 83 C4
00 A8 55 0B 83 14 43 16 E3 E1 6B 68 50 FA 36 B2 0D 06 F8 86 4C AC 6A 24 81 8F 49 CD 89 F4 35 8C
7F 26 0C F2 36 27 20 94 24 A3 ED 27 59 A0 AF 28 36 07 65 7B ED 31 75 EA 88 8F CF 35 E2 95 96 0D
73 59 63 76 1D DA 89 48 C7 19 6C 30 F6 DE 2D 97 F4 1F 04 D6 73 73 FA 22 ED 57 E1 8C CC C6 83 FA
08 7E 73 E1 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 01 00 01

```

![](https://bbs.kanxue.com/upload/attach/202402/947361_WVKGMR6WMMZ46WF.jpg)

再次运行后发生了栈溢出

![](https://bbs.kanxue.com/upload/attach/202402/947361_M6D8R6XQTJC8ZTT.jpg)

定位到发生溢出的位置`0018003CAD4`，可以推测是加载语言的 dll 时出现了问题，如下：

![](https://bbs.kanxue.com/upload/attach/202402/947361_SCB3AF7EW9KCFJN.jpg)

跟进`sub_18003CD00`

![](https://bbs.kanxue.com/upload/attach/202402/947361_Q5Q7A3SHCHGNDMP.jpg)

为了让程序正常运行，需要 patch `dopuslib_14`返回 1，`dopuslib_14`似乎整个程序只是用于验证 dll 的合法性，所以直接 patch`dopuslib_14`，patch 内容以及位置如下

```
000000018003D05A               | 0F84 CF000000         | je dopuslib.18003D12F                            |
000000018003D060               | 33D2                  | xor edx,edx                                      |
000000018003D08F               | 0F84 9A000000         | je dopuslib.18003D12F                            |
000000018003D095               | 817C24 30 90000000    | cmp dword ptr ss:[rsp+30],90                     |
000000018003D12F               | B8 01000000           | mov eax,1                                        |
000000018003D134               | 90                    | nop                                              |


```

### 3. 注册以及解决证书存根验证问题

重新运行，可见程序正常运行，并输入替换公钥对应的注册码

```
-----BEGIN GPSOFT PROGRAM CERTIFICATE-----
DGuOAAKkWbKadvhL5aDx/289NPqpFkZ5t0nWrEKIbYUvL7UgAVPDBy44gb0Yu
DVV6w4ICytAywy0jYpN3GRuGg13hv/xLjn8uXiFDT1szlVhEdyEpxVsd1lFyE
D1y6hjPuu2hs0jHiRpiS8w1RMF7/F7GqoIhj3AqhhHYO/8Mh2kA4kF5ofGuOA
DAEetKGx30HIBAdFK/z/mq41/uqM3c1b3JqIfgMstgIwj/4dQquLL5cC0yK0n
DOOrYxlQ5eUrLgq4fPEM6DP5kvKaI8GXiDBg4oj464qZM/XMNCFnXOmFpvWII
D+oRxNHQJe+2S+7TxJb247mBMzCoqPOFV8ED0PIJqqcL46rPzllGuGuOAAAbP
D2j4rzUb4KnEA6vdKTDYtreCpWBxzougWvWcQRFtyuCjPTurx49VBbCw6ePfa
D9+gFUgOXBBTHOejk+9/eQxP1KMZOmsz0Od34vYEOf00l8APgxv1Ws3sDIdGm
Dx1bP8xk3Sk8nRfRqXcDVh1odQUhtS2l9JJKfFA0KRq4PGXrlGuOAAFZqs7+j
DHV6hs9LnH/DfplhfqcTWOovrUInnzXeRW5sWIr60yHLLBJBy4F21ZgTelbiT
DNnFF8xdtOaB9D101zpsIsHM3AoaKq0S7W9uHHElxI0R6zPlbIDwB3KXtLBSR
DRxe7AJntH8oP2SOMBLm2xcMVeAKzDIGCim8xlczlDpveGuOAAAEYK64KAEG5
D9mijNQ9wfEFhKgHH02/Gid0QY5ROhSJvkc4pBtpMEZ/Xp6t66R398I6JfSDH
DMymwKgWcXhd+fcuQ9KPkCdGScfeVc8WsyvYKc/YPqUEQHbxdLTitqMFdZjtI
D5HFRveoaxyVlXjerup5chjI4Uvniqazq9EiunEBH
ce1qDfukmiuD+JLiGziFBAw==
-----END GPSOFT PROGRAM CERTIFICATE-----

```

![](https://bbs.kanxue.com/upload/attach/202402/947361_GYVDD43DVCY5HNT.jpg)

但是重启程序后依然会弹出未授权的提示。这里联想到在上方的 [CreateFileW 日志](#CreateFileW%E8%B0%83%E7%94%A8%E6%97%A5%E5%BF%97)中，程序大量使用`CreateFileW`打开`opus.cert`。通过对比可以发现`opus.cert`在我们注册完成之后已经发生了变化。所以我们的注册逻辑是成功的，证书已经成功被写入了存根，而在此打开提示未授权的原因大概率出在对存根证书的验证逻辑上。

同样在`CreateFileW`下条件断点

![](https://bbs.kanxue.com/upload/attach/202402/947361_UFS5SMHX4BRGUS5.jpg)

整理得到如下调用：

```
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197
CreateFileW file--->C:\\Program Files\\GPSoftware\\Directory Opus:stockcert13 retAddr--->0x1401E3197
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert:naughtypirates retAddr--->0x140CE5C0F
CreateFileW file--->C:\\ProgramData\\GPSoftware\\Directory Opus\\dopus.cert retAddr--->0x1401E3197

```

![](https://bbs.kanxue.com/upload/attach/202402/947361_MZGFTQYASC7ZJXE.jpg)

`0x1401E3197`位置所在的函数是一个公共函数，其有很多的引用。这里的话只能硬着头皮去分析了，8 个调用工作量也不算太大。

对函数`CreateFileW`所在的函数`sub_1401E30F0`下断点。

![](https://bbs.kanxue.com/upload/attach/202402/947361_PEWGC6E9WQMAW7G.jpg)

最终定位 VA`0000000140CE721A`中的如下位置：

![](https://bbs.kanxue.com/upload/attach/202402/947361_V53WADNH9J82WDB.jpg)

经过上面的分析我们知道`qword_1418BF450 + 0xDC0`所在的内存区域，这里对公钥进行了哈希，与程序内置的一串哈希进行比对，这里肯定是会失败的。所以我们强跳在这个检测就可以。

```
0000000140CE8533               | 48:3B45 A0            | cmp rax,qword ptr ss:[rbp-60]                    |
0000000140CE8537               | EB 27                 | jmp dopus.140CE8560                              |
0000000140CE8539               | 48:8B05 208CB800      | mov rax,qword ptr ds:[141871160]                 |
0000000140CE8540               | 48:3B45 A8            | cmp rax,qword ptr ss:[rbp-58]                    |
0000000140CE8544               | EB 1A                 | jmp dopus.140CE8560                              |
0000000140CE8546               | 45:33FF               | xor r15d,r15d                                    |
0000000140CE8549               | 41:8BDF               | mov ebx,r15d                                     |
0000000140CE854C               | 48:895C24 78          | mov qword ptr ss:[rsp+78],rbx                    |
0000000140CE8551               | BA 00020000           | mov edx,200                                      |
0000000140CE8556               | 48:8BCF               | mov rcx,rdi                                      |
0000000140CE8559               | E8 62800400           | call dopus.140D305C0                             |
0000000140CE855E               | EB 03                 | jmp dopus.140CE8563                              |
0000000140CE8560               | 45:33FF               | xor r15d,r15d                                    |

```

### 4. 公钥完整性检测

重新运行后进入主程序，但是马上就出现了完整性校验失败，我们可以排除`WinVerifyTrust`校验问题

![](https://bbs.kanxue.com/upload/attach/202402/947361_QA8QF9TDG4EQA99.jpg)

前面我们分析到程序大概率没有使用 CRC 对自身完整性进行校验，但是可能会对证书进行校验呢，事实上证实如此，我们可以在`dopus.exe`的函数窗口搜索 CRC，如下，可以看到虽然没有对`CalcCRC32`的直接引用，但是对其中的函数`sub_1412A4960`有些许引用。这里我们可以对此函数下断，根据传入的数据跟我们已知的内容 (比如证书存根或者我们替换的 publickey) 进行比对，来找到可疑位置。

![](https://bbs.kanxue.com/upload/attach/202402/947361_9B43EKN9JXKCEPW.jpg)

因为考虑到对 CRC 函数进行下断分析工作可能会非常繁重，所以这里笔者通过 bindiff 比对的方式，将原 12.33 版本的`opus.exe`与网上公布的同版本破解版进行比对，发现其大体的修复思路相同。我们注意到如下位置：

![](https://bbs.kanxue.com/upload/attach/202402/947361_SXXZ6SJCN9MGDGH.jpg)

一个简单的分析结果如下

![](https://bbs.kanxue.com/upload/attach/202402/947361_2PK9S4SD4Z73RBG.jpg)

所以我们使用正常的程序拿到一个正常的哈希值来替换这个结果即可。patch 如下

```
0000000140CD6D6E               | B8 F90F2B14           | mov eax,142B0FF9       |0x142B0FF9为正常公钥的哈希值

```

最终结果：

![](https://bbs.kanxue.com/upload/attach/202402/947361_D9Y8XNDVZVVMEXB.jpg)

Ⅳ. 总结
-----

总的来说，整个程序的反逆向工程结构如下

1.  通过 Windows 受信组件 (WinVerifyTurst) 对文件完整性进行校验
2.  使用程序中自己实现的 Hash 对公钥进行校验
3.  将程序运行的关键组件 (窗口类名) 与公钥信息 Hash 进行绑定

比较有意思的是，程序维护一个区域`0x00000001801C0230`并将检测结果维护其中，其中的某些字段会引起程序的异常行为而导致逆向成本的增加，对产品安全性有一定的提升。  
**2024 年 3 月 5 日更新**  
在 docs.dll RVA:EC72 中新增一处过完整性校验 Patch,Patch 为 jmp 即可，在调试过程中出现出现一些域名信息，可能系网络验证，找到这个点的思路同样是对`WinverifuTrust`下断记录即可，出于时间原因不再进一步逆向，有兴趣的朋友可以深入研究。  
![](https://bbs.kanxue.com/upload/attach/202403/947361_WR45TEVB7ZW43DB.webp)  
**2024 年 3 月 29 日更新**  
dopus.exe RVA:`0xcc7a0f`出触发了一些文件完整性校验  
![](https://bbs.kanxue.com/upload/attach/202403/947361_ZG8UG43DBPSCFGD.webp)  
定位方法使用比较简单的对检测弹出的窗口进行附加查看窗口启动的命令行信息：/BADSHOW1，然后对`CreateProcessW`下断，以命令行特征为断点，很快可以定位到线程函数，向上回溯就可。

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 2024-3-29 10:16 被 Sw1ndler 编辑 ，原因： 遗漏一处完整性校验。 [#软件保护](forum-4-1-3.htm) [#其他内容](forum-4-1-10.htm) [#调试逆向](forum-4-1-1.htm)