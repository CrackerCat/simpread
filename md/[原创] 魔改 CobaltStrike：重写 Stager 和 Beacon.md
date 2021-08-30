> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269115.htm)

> [原创] 魔改 CobaltStrike：重写 Stager 和 Beacon

[](#一、概述)一、概述
=============

这次我们一起用 C# 来重写 stager 及其 Beacon 中的大部分常用功能，帖子主要介绍该项目的运行原理（LolBins-->Stager-->Beacon）及相应的功能介绍及展示。LolBins 部分是由 GadgetToJs 使 Stager 转换为 js、vba、hta 文件后，再结合相应的 csript、mshta 等程序来运行；Stager 功能包括从网络中拉取 Beacon 的程序集并在内存中加载及 AMSIBypass；Beacon 部分主要有包括正常上线、文件管理、进程管理、令牌管理、结合 SysCall 进行注入、原生端口转发、关 ETW 等一系列功能。

 

项目地址：  
https://github.com/mai1zhi2/SharpBeacon/tree/master

 

项目基于. net4.0，暂支持 cs4.1（更高版本待测试），感谢 M 大、WBG 师傅、SharpSploit、Geason 的分享。另因最近出去广州找工作没时间弄，就暂时写到这里，开发进度比较赶致使封装不是很好、设计模式也没有用，但每个实现功能点都有较详细注释，等后续工作安定后会进行重构及完善更多功能。若有错误之处还请师傅指出，谢谢大家。

[](#二、lolbins)二、LolBins
=======================

LOLBins，全称 “Living-Off-the-Land Binaries”，直白翻译为 “生活在陆地上的二进 “，我大概将其分为两大类：  
1、带有 Microsoft 签名的二进制文件，可以是 Microsoft 系统目录中二进制文件。  
2、第三方认证签名程序。  
LolBins 的程序除了正常的功能外，还可以做其他意想不到的行为。在 APT 或红队渗透常用，常见于海莲花等 APT 组织所使用。下图是较常见的 LolBins，还有很多就不一一列出了：  
![](https://bbs.pediy.com/upload/attach/202108/718877_QZGSZA7D2NHXTMU.png)  
而 GadgetToJS 项目则可以把源码 cs 文件动态编译再 base64 编码后，保存在 js、vba、vbs、hta 文件，而在其相关文件中文件利用了当 BinaryFormatter 属性在进行反序列化时，可以触发对 Activator.CreateInstance() 的调用，，从而实现 .NET 程序集加载 / 执行。  
![](https://bbs.pediy.com/upload/attach/202108/718877_3D2AGNG9MZCVEHS.png)  
但这需要在. net 程序集中把相应的功能写在默认 / 公共构造函数，这样才能触发 .NET 程序集执行。下面以实例程序为例：  
![](https://bbs.pediy.com/upload/attach/202108/718877_Z75SEXCNFVNSTFV.png)  
在相应文件夹下执行如下命令：  
.\GadgetToJScript.exe -w js -c Program.cs -d System.Windows.Forms.dll -b -o gg  
其中命令参数解析如下：  
-w js 表示所生成的是 js 文件，可以生成其他形式的文件  
-c Program.cs 是所选择的 cs 文件  
-d System.Windows.Forms.dll cs 文件所用到的 dll  
-b 会在 js 文件中的引入第一个 stager，因为当在. NET4.8 + 的版本中引入了旁路类型检查控件，默认值为 false，如果所生成的脚本要在. NET4.8 + 的环境中运行，则设置为 true（--Bypass/-b）。生成的 stager1 就是 bypass 这个检查的。  
-o gg 生成文件名  
生成 js、hta、vbs 等文件后默认是会被杀的：  
![](https://bbs.pediy.com/upload/attach/202108/718877_8WP8F4UB5RFJVKR.png)  
而我们只需要简单修改下单引号为 / 就行了:  
![](https://bbs.pediy.com/upload/attach/202108/718877_XVE333EJHBGH9MY.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_YCAAV9QRVHP9MEC.png)  
最后执行所生成的 js 或 hta：  
![](https://bbs.pediy.com/upload/attach/202108/718877_8W9GNSQ9BJSVBU6.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_PC76SH9SJGBH3CT.png)

[](#三、stager)三、Stager
=====================

Stager 部分的功能可以包括下图几项：  
![](https://bbs.pediy.com/upload/attach/202108/718877_ENN2R27AWYQJAM7.png)  
我主要实现了从网络中拉取 Beacon 的程序集并在内存中加载及 AMSIBypass，沙箱及虚拟机检测的方式有挺多方式的，师傅可以自行添加。  
拉取程序集及内存加载这个较为简单，就不细说了：  
![](https://bbs.pediy.com/upload/attach/202108/718877_8TKPNQ2RCCV7X8B.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_KCU7NQBPR2NWT26.png)  
下面说说 bypassAMSI，这里一开始找的不是 AmsiScanBuffer，而是找 DllCanUnloadNow 的地址：  
![](https://bbs.pediy.com/upload/attach/202108/718877_TQCSM323R9A32TM.png)  
然后再通过相关的硬编码找到 AmsiScanBuffer 后，再进行相应的 patch：  
![](https://bbs.pediy.com/upload/attach/202108/718877_XZ4XBYPXBCM8BCP.png)

[](#四、beacon)四、Beacon
=====================

Beacon 部分主要有包括正常上线、文件管理、进程管理、令牌管理、结合 SysCall 进行注入、原生端口转发、关 ETW 等一系列功能。  
![](https://bbs.pediy.com/upload/attach/202108/718877_9MQGM8PC94JZSWS.png)

4.1 文件管理
--------

先从文件管理部分说，包含了 cp、mv、upload、download、filebrowse、rm、mkdir 上述这七个功能点：  
Cp:  
![](https://bbs.pediy.com/upload/attach/202108/718877_JC76WGGKSPSVZTF.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_EQVSR2DDCW3737X.png)  
Mv:  
![](https://bbs.pediy.com/upload/attach/202108/718877_FKJNUYRG9V4EZXY.png)  
Upload:  
![](https://bbs.pediy.com/upload/attach/202108/718877_T5YEVGZJCF77NJQ.png)  
Download:  
![](https://bbs.pediy.com/upload/attach/202108/718877_FD7PFVP6WPRPXRA.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_TXH2Q7BF73WTNFF.png)  
Filebrowse:  
![](https://bbs.pediy.com/upload/attach/202108/718877_23E78UFDGD8HRPZ.png)  
rm:  
![](https://bbs.pediy.com/upload/attach/202108/718877_C8FWQEFD3FVQGCX.png)  
mkdir  
![](https://bbs.pediy.com/upload/attach/202108/718877_3PMKW3YT3C6X68R.png)

4.2 进程部分
--------

进程部分，已完成的有 run、shell、execute、runas、kill，未完成的有 runu:  
Run:  
![](https://bbs.pediy.com/upload/attach/202108/718877_U44GKQEB8WZXMPY.png)  
shell:  
![](https://bbs.pediy.com/upload/attach/202108/718877_S4CRC8SSAA66QV6.png)  
execute:  
![](https://bbs.pediy.com/upload/attach/202108/718877_C58HXYRHW5G2RW7.png)  
runas:  
![](https://bbs.pediy.com/upload/attach/202108/718877_EHZESSPMJXJ6RNX.png)  
ps:  
![](https://bbs.pediy.com/upload/attach/202108/718877_UHSBXHFKAP8DV57.png)  
kill:  
![](https://bbs.pediy.com/upload/attach/202108/718877_4RBJC8X7A2KWQDF.png)

4.3 令牌权限
--------

令牌权限部分，已完成的有 getprivs、make_token、steal_token、rev2self：  
Getprivs:  
![](https://bbs.pediy.com/upload/attach/202108/718877_DDK2ZNNUZPRKQYK.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_B3XHM5P5MY5MQMJ.png)  
make_token：测试时在 make_token 后执行了 cmd.exe /C dir \10.10.10.165\C$  
![](https://bbs.pediy.com/upload/attach/202108/718877_JTTD6QDAQ9ARS7Y.png)  
steal_token：测试时在 steal_token 后执行了 whoami  
![](https://bbs.pediy.com/upload/attach/202108/718877_TBR8UAGTDYEZG93.png)  
rev2self：  
![](https://bbs.pediy.com/upload/attach/202108/718877_KVY6NM3R5444RQY.png)

4.4 端口转发
--------

端口转发部分，已完成的有 rportfwd、rportfwd stop：  
Rportfwd，注意这里端口转发 teamserver 只返回了本地需要绑定的端口，没有返回需转发的 ip 和 port。  
在 192.168.202.180:22222 上新建 msf 监听：  
![](https://bbs.pediy.com/upload/attach/202108/718877_ZD6WNK7AEGGHDRQ.png)  
在本机地址 192.168.202.1 的 23456 端口转发到上述 msf 的监听  
![](https://bbs.pediy.com/upload/attach/202108/718877_TD4GXFPQXF6Z36F.png)  
![](https://bbs.pediy.com/upload/attach/202108/718877_PR4Y3EJHW4NTG9Z.png)  
本地访问 23456 端口：  
![](https://bbs.pediy.com/upload/attach/202108/718877_AJBEFXP7FKH35CC.png)  
另一个网段访问 23456 端口：  
![](https://bbs.pediy.com/upload/attach/202108/718877_TZBZKE29AUZ2TRN.png)  
rportfwd stop：  
![](https://bbs.pediy.com/upload/attach/202108/718877_B7R89X8Z9WF3WUU.png)

4.5 注入部分
--------

注入部分，cs 的 shinject、dllinject、inject 都用来远程线程注入，我个人机器是 win10 x64 1909,shellcode 是用 cs 的 64 位 c# shellcode，被注入的程序是 64 位的 calc.exe，程序返回的 NTSTSTUS 均为 SUCCESS，且 shellcode 均已注入在相应的程序中，并新建出线程进行执行，但最后 calc.exe 都崩了，有点奇怪呀：  
![](https://bbs.pediy.com/upload/attach/202108/718877_DXA8YMT7UJRES7T.png)  
申请 rwx 内存空间存放 shellcode 后并在所执行 shellcode 下断:  
![](https://bbs.pediy.com/upload/attach/202108/718877_DR9MBMAWCBD64ZN.png)

 

执行 NtCreateThreadEx(), 被注入的 calc.exe 新建线程执行此 shellcode：  
![](https://bbs.pediy.com/upload/attach/202108/718877_P9Q9X6JCJYWRTPG.png)  
最后跑起来报的 c05，但分配的内存属性是 rwx 的：  
![](https://bbs.pediy.com/upload/attach/202108/718877_SC6Y75F54U39E6G.png)

4.6 杂项部分
--------

杂项部分，已完成有 sleep、pwd、exit、setenv、drives、cd：  
Sleep：  
![](https://bbs.pediy.com/upload/attach/202108/718877_NAPGEEQRJGA8VHF.png)  
exit：  
![](https://bbs.pediy.com/upload/attach/202108/718877_5WT33FN9BRTJVXQ.png)  
setenv：  
![](https://bbs.pediy.com/upload/attach/202108/718877_RQQVN7EFFRCEU98.png)

 

drives：  
![](https://bbs.pediy.com/upload/attach/202108/718877_C5Z77N9TK635EDT.png)  
Pwd：  
![](https://bbs.pediy.com/upload/attach/202108/718877_HU3RFE6VTB79JVW.png)  
cd：  
![](https://bbs.pediy.com/upload/attach/202108/718877_6ZPAXRMY99PAF4A.png)

[](#五、完善及改进)五、完善及改进
===================

后续需要改进的地方还有很多，有如下几点：  
1、该封装好就封装好，该用设计模式就用  
2、目前 rsa 密钥是 pem 方式就用了 BouncyCastle 库，要用回 Exponent 和 Modulus  
3、更多的注入方式，APC、傀儡进程等  
4、更多的通信协议，如 DNS、ICMP  
5、支持 spawn**，因为当执行 spawn 和 job 后，teamserver 端会回传相应的 dll，要改 ts 端  
6、更多的功能，如 mimi、keylogger、portscan、加载 pe 等  
最后谢谢大家观看。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 2 小时前 被快乐鸡哥编辑 ，原因：

[#开发技巧](forum-41-1-135.htm)