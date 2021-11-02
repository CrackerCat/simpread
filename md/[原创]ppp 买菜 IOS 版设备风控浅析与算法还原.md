> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270097.htm#msg_header_h3_6)

> [原创]ppp 买菜 IOS 版设备风控浅析与算法还原

发点存货，本文仅限学习交流，请勿用于非法以及商业用途，由于时间和水平有限，文中错漏之处在所难免，敬请各位大佬多多批评指正。

```
目录:
一、线上买菜场景简述
二、风控在业务中的应用
三、产品整体框架
四、初始化分析
五、反爬签名流程
六、设备指纹分析
七、算法还原
八、总结

```

### [](#一、线上买菜场景简述)一、线上买菜场景简述

#### [](#1、分析说明)1、分析说明

```
1. 产品基本信息
产品名称：ppp买菜(匿称)；
产品版本：5.25.0；
Slogan：30分钟送达，新鲜送得快；
所处行业：生鲜电商;
 
2. 设备环境
机型：iPhone 7；
系统：IOS 13.4；
工具: IDA7.6 Frida;

```

#### [](#2、简单流程梳理)2、简单流程梳理

一次完整的线上买菜过程都经过了哪些环节呢？大致流程是从供应商送货到仓或到店，再由零售商售卖，最终到用户手里，这样便完成了一次买菜，如图 1-1 所示：  
![](https://bbs.pediy.com/upload/attach/202111/536985_7HPTYGZBYMBBM3Q.png)  
         图 1-1  
上图的业务流程从供应商送货到仓或到店，再由零售商售卖，最终可以多种方式到用户手里，完成了一次买菜的过程。

### [](#二、反作弊风控在业务中的应用)二、反作弊风控在业务中的应用

#### [](#1、app推广拉新)1、APP 推广拉新

还记得在 2020 年的下半年时候，当时生鲜电商的社区团购大战非常火爆，各种买菜 APP 蜂拥而入，砸钱、抢流量，你争我抢玩得不亦乐乎。  
不夸张地说，我记得当时最常见的情形是，你随便在小区溜达一圈，就能碰见穿着各种颜色制服的地推工作人员，追赶着小哥哥小姐姐下载 APP 给送福利，下载完 APP 后注册登录 APP 买菜。

#### [](#2、存在的风险)2、存在的风险

烧大把的钱把流量吸引过来，这个过程中会有黑灰产人员通过非法的技术手段，伪造新增用户并从中获利的行为，如果只是把流量吸引过来不考虑质量的话，会增加大量的企业无效成本。怎么识别出有效的流量与虚假流量，需要一个完善的风控体系与制定有效的策略找出高质量流量，然后把这些流量留下来。  
接下来为了提高用户的购买频率，实现反复转化，就出现了各种红包、优惠券活动吸引用户提高打开 APP 频率与购买频率。这个环节中就会有各种薅羊毛的人群出现，同样需要完善的风控体系与制定有效的策略来最大程度地甄别风险。  
活动流程大致如图 2-1 所示:  
![](https://bbs.pediy.com/upload/attach/202111/536985_PB542CGMZ4EA9ZW.png)  
         图 2-1

### [](#三、产品整体框架)三、产品整体框架

#### 3.1、从初化到获取设备指纹整个框架如图 3-1 与 3-2 所示：

![](https://bbs.pediy.com/upload/attach/202111/536985_QRSQFQJ4PRK5HTB.png)  
         图 3-1  
![](https://bbs.pediy.com/upload/attach/202111/536985_36T5G68ED8J66RY.png)  
         图 3-2  
下面将围绕框架进行详细分析与算法还原。

### [](#四、初始化分析)四、初始化分析

#### 4.1、代码混淆

在正式进行代码分析之前还是很必要交代一下我分析过程中发现的代码混淆，方便后继分析代码做准备。  
反 F5：大致模板如下：

```
__text:0000000103743850 FF C3 00 D1 SUB             SP, SP, #0x30 ; '0'
__text:0000000103743854 E0 7B 01 A9 STP             X0, X30, [SP,#0x10]
__text:0000000103743858 05 00 00 94 BL              sub_10374386C ; 跳到方法返回值
__text:000000010374385C 86 A5 23 4F SSHLL2          V6.2D, V12.4S, #3
__text:0000000103743860 B1 FB A7 9B UMSUBL          X17, W29, W7, X30
__text:0000000103743864 D2 76 2F F9 STR             X18, [X22,#0x5EE8]
__text:0000000103743868 7F 16 04 FA DCB 0x7F, 0x16, 4, 0xFA ; 混淆数据
__text:000000010374386C
__text:000000010374386C         
__text:000000010374386C
__text:000000010374386C           
__text:000000010374386C
__text:000000010374386C             sub_10374386C   ;
__text:000000010374386C 80 01 00 10 ADR              X0, dword_10374389C ;方法返回值
__text:0000000103743870 FE 03 00 AA MOV             X30, X0 ; 方法返回值给返回寄存器
__text:0000000103743874 FF C3 00 91 ADD             SP, SP, #0x30 ; '0'
__text:0000000103743878 C0 03 5F D6 RET

```

原理是通过动态赋值给 x30 实现跳转，X30 链接寄存器 (LR)，用于保存子程序的返回地址。通篇都是这样的代码混淆方式，基本不怎么影响分析或脚本直接清除。  
动态调试时如图 4-1 所示：  
![](https://bbs.pediy.com/upload/attach/202111/536985_9CVCPZEV99WHCYJ.jpg)  
         图 4-1  
字符串加解密:

```
__text:000000010180C0C0             DecStrng_sub_1036F80C0
__text:000000010180C0C0
__text:000000010180C0C0 7F 04 00 71                 CMP             W3, #1
__text:000000010180C0C4 0B 06 00 54                 B.LT            locret_10180C184
__text:000000010180C0C8 E9 03 03 2A                 MOV             W9, W3
__text:000000010180C0CC 7F 40 00 71                 CMP             W3, #0x10
__text:000000010180C0D0 43 04 00 54                 B.CC            loc_10180C158
__text:000000010180C0D4 08 00 09 8B                 ADD             X8, X0, X9
__text:000000010180C0D8 2A 00 09 8B                 ADD             X10, X1, X9
__text:000000010180C0DC 5F 01 00 EB                 CMP             X10, X0
__text:000000010180C0E0 00 81 41 FA                 CCMP            X8, X1, #0, HI
__text:000000010180C0E4 A8 03 00 54                 B.HI            loc_10180C158
__text:000000010180C0E8 2A 6D 7C 92                 AND             X10, X9, #0xFFFFFFF0
__text:000000010180C0EC 68 04 03 0B                 ADD             W8, W3, W3,LSL#1
__text:000000010180C0F0 08 01 02 0B                 ADD             W8, W8, W2
__text:000000010180C0F4 6B 0C 00 12                 AND             W11, W3, #0xF
__text:000000010180C0F8 6B 09 0B 4B                 SUB             W11, W11, W11,LSL#2
__text:000000010180C0FC 08 01 0B 0B                 ADD             W8, W8, W11
__text:000000010180C100 40 0C 01 0E                 DUP             V0.8B, W2
__text:000000010180C104 8B 8C 00 D0                 ADRP            X11, #qword_10299E470@PAGE
__text:000000010180C108 61 39 42 FD                 LDR             D1, [X11,#qword_10299E470@PAGEOFF]
__text:000000010180C10C 00 84 21 0E                 ADD             V0.8B, V0.8B, V1.8B
__text:000000010180C110 2B 20 00 91                 ADD             X11, X1, #8
__text:000000010180C114 0C 20 00 91                 ADD             X12, X0, #8
__text:000000010180C118 01 E7 00 0F                 MOVI            V1.8B, #0x18
__text:000000010180C11C 02 E6 01 0F                 MOVI            V2.8B, #0x30 ; '0'
__text:000000010180C120 ED 03 0A AA                 MOV             X13, X10
__text:000000010180C124
__text:000000010180C124             loc_10180C124
__text:000000010180C124 03 84 21 0E                 ADD             V3.8B, V0.8B, V1.8B
__text:000000010180C128 64 95 7F 6D                 LDP             D4, D5, [X11,#-8]
__text:000000010180C12C 84 1C 20 2E                 EOR             V4.8B, V4.8B, V0.8B
__text:000000010180C130 A3 1C 23 2E                 EOR             V3.8B, V5.8B, V3.8B
__text:000000010180C134 84 8D 3F 6D                 STP             D4, D3, [X12,#-8]
__text:000000010180C138 00 84 22 0E                 ADD             V0.8B, V0.8B, V2.8B
__text:000000010180C13C 6B 41 00 91                 ADD             X11, X11, #0x10
__text:000000010180C140 8C 41 00 91                 ADD             X12, X12, #0x10
__text:000000010180C144 AD 41 00 F1                 SUBS            X13, X13, #0x10
__text:000000010180C148 E1 FE FF 54                 B.NE            loc_10180C124
__text:000000010180C14C 5F 01 09 EB                 CMP             X10, X9
__text:000000010180C150 81 00 00 54                 B.NE            loc_10180C160
__text:000000010180C154 0C 00 00 14                 B               locret_10180C184
__text:000000010180C158
__text:000000010180C158
__text:000000010180C158             loc_10180C158
__text:000000010180C158
__text:000000010180C158 0A 00 80 D2                 MOV             X10, #0
__text:000000010180C15C E8 03 02 AA                 MOV             X8, X2
__text:000000010180C160
__text:000000010180C160             loc_10180C160
__text:000000010180C160 2B 00 0A 8B                 ADD             X11, X1, X10
__text:000000010180C164 0C 00 0A 8B                 ADD             X12, X0, X10
__text:000000010180C168 29 01 0A CB                 SUB             X9, X9, X10
__text:000000010180C16C
__text:000000010180C16C             loc_10180C16C
__text:000000010180C16C 6A 15 40 38                 LDRB            W10, [X11],#1
__text:000000010180C170 4A 01 08 4A                 EOR             W10, W10, W8
__text:000000010180C174 8A 15 00 38                 STRB            W10, [X12],#1
__text:000000010180C178 08 0D 00 11                 ADD             W8, W8, #3
__text:000000010180C17C 29 05 00 F1                 SUBS            X9, X9, #1
__text:000000010180C180 61 FF FF 54                 B.NE            loc_10180C16C
__text:000000010180C184
__text:000000010180C184             locret_10180C184
__text:000000010180C184
__text:000000010180C184 C0 03 5F D6                 RET

```

通用的解密字符串，同样可以用脚本跑一遍就可以解密出来。  
采集设备信息跳转：

```
__text:0000000101864F1C
__text:0000000101864F1C E0 7B 7B A9                 LDP             X0, X30, [SP,#-0x50] ; 获取m设备信息跳转
__text:0000000101864F20 09 2D 00 10                 ADR             X9, unk_1018654C0
__text:0000000101864F24 1F 20 03 D5                 NOP
__text:0000000101864F28 28 79 A8 B8                 LDRSW           X8, [X9,X8,LSL#2]
__text:0000000101864F2C 08 01 09 8B                 ADD             X8, X8, X9
__text:0000000101864F30 E0 7B 3B A9                 STP             X0, X30, [SP,#-0x50]
__text:0000000101864F34 00 01 1F D6                 BR              X8      ; 获取m设备信息跳转对应的方法

```

只要在上面地方下好断点就能分析对应的采集设备信息的方法。  
4.2、解密资源文件获取 PIC

##### 生成密钥:

获取 APP Bundle ID:

```
com.baobaoaichi.imaicai

```

解析 Info.plist 读取 <key>ss</key > 中的值:

```
885B25AAFD830249B81AF699187E5752

```

解密常量字符串:

```
WU@TEN

```

组合字符串，BundleID + 常量字符串 (WU@TEN)+Info.plist 中的 < key>ss</key > 值:

```
com.baobaoaichi.imaicaiWU@TEN885B25AAFD830249B81AF699187E5752

```

计算组合后字符的 hmac 值:

```
__text:0000000101B834F8             hamc_256_loc_1052974F8
__text:0000000101B834F8 FF 43 01 D1 SUB             SP, SP, #0x50 ; 'P'
__text:0000000101B834FC E0 7B 02 A9 STP             X0, X30, [SP,#0x20]
__text:0000000101B83500 08 00 00 94 BL              sub_101B83520
__text:0000000101B83504 D9 92 B2 98 LDRSW           X25, loc_101AE875C
__text:0000000101B83508 B2 19 08 BD STR             S18, [X13,#0x818]
__text:0000000101B8350C 1B 50 F0 B7 TBNZ            X27, #0x3E, loc_101B83F0C ; '>'
__text:0000000101B8350C 
__text:0000000101B83510 81 05 6D 1B+DCQ 0x440C4BA91B6D0581, 0x75CE544974E8B95A
__text:0000000101B83520
__text:0000000101B83520
__text:0000000101B83520
__text:0000000101B83520
__text:0000000101B83520             sub_101B83520
__text:0000000101B83520 40 02 00 10 ADR             X0, loc_101B83568
__text:0000000101B83524 FE 03 00 AA MOV             X30, X0
__text:0000000101B83528 FF 43 01 91 ADD             SP, SP, #0x50 ; 'P'
__text:0000000101B8352C C0 03 5F D6 RET
__text:0000000101B8352C             ; End of function sub_101B83520
__text:0000000101B8352C
__text:0000000101B8352C
__text:0000000101B83530 BA C7 78 F8+DCQ 0x5AC568B8F878C7BA, 0x9A5A15BB2AA24258, 0x748421186E938E6D, 0x53C1BE1404BC0FB9
__text:0000000101B83530 B8 68 C5 5A+DCQ 0x3C6162795B16F2AB, 0x3F6EEF1BA5E72F0, 0xBF4042773B6E82C5
__text:0000000101B83568  
__text:0000000101B83568
__text:0000000101B83568             loc_101B83568                           ; DATA XREF: sub_101B83520↑o
__text:0000000101B83568 E0 7B 7D A9 LDP             X0, X30, [SP,#-0x30]
__text:0000000101B8356C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B83570 20 00 80 D2 MOV             X0, #1
__text:0000000101B83574 04 00 00 14 B               loc_101B83584
 
__text:0000000101B835B0 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:0000000101B835B4 FF 43 01 D1 SUB             SP, SP, #0x50 ; 'P'
__text:0000000101B835B8 F4 4F 03 A9 STP             X20, X19, [SP,#0x30]
__text:0000000101B835BC FD 7B 04 A9 STP             X29, X30, [SP,#0x40]
__text:0000000101B835C0 FD 03 01 91 ADD             X29, SP, #0x40 ; '@'
__text:0000000101B835C4 F3 03 08 AA MOV             X19, X8
__text:0000000101B835C8 88 B9 00 90 ADRP            X8, #___stack_chk_guard_ptr@PAGE
__text:0000000101B835CC 08 FD 43 F9 LDR             X8, [X8,#___stack_chk_guard_ptr@PAGEOFF]
__text:0000000101B835D0 08 01 40 F9 LDR             X8, [X8]
__text:0000000101B835D4 A8 83 1E F8 STUR            X8, [X29,#-0x18]
__text:0000000101B835D8 00 E4 00 4F MOVI            V0.16B, #0
__text:0000000101B835DC E0 83 81 3C STUR            Q0, [SP,#0x18]
__text:0000000101B835E0 E0 83 80 3C STUR            Q0, [SP,#8]
__text:0000000101B835E4 E2 23 00 91 ADD             X2, SP, #8
__text:0000000101B835E8 B6 EB FF 97 BL              hmac256_loc_1052924C0   ; x0:组合的字符串com.baobaoaichi.imaicaiWU@TEN885B25AAFD830249B81AF699187E5752
__text:0000000101B835E8                                                     ; x1:长度,x2:返回值
__text:0000000101B835EC E0 23 00 91 ADD             X0, SP, #8
__text:0000000101B835F0 E1 03 1B 32 MOV             W1, #0x20 ; ' '
__text:0000000101B835F4 E8 03 13 AA MOV             X8, X19
__text:0000000101B835F8 E3 FD FF 97 BL              Hex2String_loc_105296D84
__text:0000000101B835FC A8 83 5E F8 LDUR            X8, [X29,#-0x18]
__text:0000000101B83600 89 B9 00 90 ADRP            X9, #___stack_chk_guard_ptr@PAGE
__text:0000000101B83604 29 FD 43 F9 LDR             X9, [X9,#___stack_chk_guard_ptr@PAGEOFF]
__text:0000000101B83608 29 01 40 F9 LDR             X9, [X9]
__text:0000000101B8360C 3F 01 08 EB CMP             X9, X8
__text:0000000101B83610 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B83614 A1 FC FF 54 B.NE            loc_101B835A8
__text:0000000101B83618 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B8361C 60 00 00 18 LDR             W0, =0
__text:0000000101B83620 D9 FF FF 17 B               loc_101B83584

```

计算后的值:

```
39 D3 B7 71 76 74 09 F5 E7 4F 4B 57 9B 86 8A 5C  01 92 13 18 61 C1 79 1C
83 3B 5C 95 E9 9C 41 2B

```

转换成字符串:

```
39d3b771767409f5e74f4b579b868a5c0192131861c1791c833b5c95e99c412b

```

读取 mt_security

```
__text:0000000101B202CC             Read_mt_security_loc_1052342CC
__text:0000000101B202CC E0 FB 3E A9 STP             X0, X30, [SP,#-0x18]
__text:0000000101B202D0 09 00 00 94 BL              sub_101B202F4
__text:0000000101B202D0            
__text:0000000101B202D4 C8 94 EB B2 DCD 0xB2EB94C8
__text:0000000101B202D8 93 66 85 65+DCQ 0xACCCADAF65856693, 0xA99940F0FB589042, 0x4DC4974493C6E96
__text:0000000101B202F0 DF D2 CB 41 DCB 0xDF, 0xD2, 0xCB, 0x41
__text:0000000101B202F4
__text:0000000101B202F4          
__text:0000000101B202F4
__text:0000000101B202F4
__text:0000000101B202F4             sub_101B202F4   const&,std::function)+4405F4↑p
__text:0000000101B202F4 E0 00 00 10 ADR             X0, loc_101B20310
__text:0000000101B202F8 FE 03 00 AA MOV             X30, X0
__text:0000000101B202FC C0 03 5F D6 RET
__text:0000000101B202FC             ; End of function sub_101B202F4
__text:0000000101B202FC
__text:0000000101B202FC            
__text:0000000101B20300 0F 49 C1 AC+DCQ 0x6E3E564CACC1490F, 0xB1B6C5A72F9066B1
__text:0000000101B20310           
__text:0000000101B20310
__text:0000000101B20310             loc_101B20310                           ; DATA XREF: sub_101B202F4↑o
__text:0000000101B20310 E0 FB 7E A9 LDP             X0, X30, [SP,#-0x18]
__text:0000000101B20314 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B20318 60 00 00 18 LDR             W0, =1
__text:0000000101B2031C 04 00 00 14 B               loc_101B2032C 
```

读取后数据 (部分):

```
0000000104899800  89 50 4E 47 0D 0A 1A 0A  00 00 00 0D 49 48 44 52  .PNG........IHDR
0000000104899810  00 00 00 3C 00 00 00 3C  08 06 00 00 00 3A FC D9  ...<...<.....:..
0000000104899820  72 00 00 13 43 49 44 41  54 78 DA ED 5B 79 70 55  r...CIDATx..[ypU
0000000104899830  D7 79 BF 4F C2 2C 06 03  36 C6 80 89 D9 C4 EA 64  ...O....6ƀ .....
0000000104899840  92 4E 92 76 62 37 9E 8E  27 4D 9D A4 71 DA 64 5A  .N.vb7..'M..q..Z
0000000104899850  77 D2 F1 34 99 F1 1F 1D  DA C9 64 EC C1 1D D7 F5  w..4............
0000000104899860  C4 71 9A E0 C4 98 C5 08  6D 68 45 2B 02 1B 0B 03  ........mhE+....
0000000104899870  12 BC 27 BD 27 BD A7 A7  7D 47 80 04 08 81 10 12  ..'.'...}G......
0000000104899880  08 10 A0 05 84 96 F7 EB  EF 9C BB 9D FB 24 81 BC  .............$..

```

解析图片:

```
__int64 __fastcall sub_1036B61F0(
        _QWORD *a1,
        unsigned int *a2,
        unsigned int *a3,
        _DWORD *a4,
        unsigned __int64 a5,
        unsigned __int64 a6)
{
 
  v8 = a4;
  v144 = 0x7BB1713F51A90031LL;
  *a1 = 0LL;
  *a3 = 0;
  *a2 = 0;
  result = sub_1036B52D8(a2, a3, a4, a5, a6);
  v8[126] = result;
  if ( (_DWORD)result )
    return result;
  v13 = *a2;
  v14 = *a3;
  v15 = (int)v8[52];
  v127 = v8 + 52;
  v16 = v8[53];
  if ( (unsigned int)v15 > 6 )
    v17 = 0;
  else
    v17 = dword_104824040[v15];
  v18 = v17 * v16;
  v19 = v8[39];
  v20 = (int)v8[38];
  if ( (unsigned int)v20 > 6 )
    v21 = 0;
  else
    v21 = dword_104824040[v20];
  if ( v18 <= v21 * v19 )
  {
    if ( (unsigned int)v20 > 6 )
      v23 = 0;
    else
      v23 = dword_104824040[v20];
    v24 = v23 * v19;
  }
  else
  {
    if ( (unsigned int)v15 > 6 )
      v22 = 0;
    else
      v22 = dword_104824040[v15];
    v24 = v22 * v16;
  }
  v25 = v8 + 126;
  if ( (_DWORD)v13 && v14 * v13 && 8 * v14 * v13 / (unsigned __int64)(v14 * v13) != 8
    || (v26 = v24 * (unsigned __int64)((unsigned int)v13 >> 3) + ((v24 * (unsigned __int64)(v13 & 7) + 7) >> 3) + 5,
        v26 * v14 / v26 != v14) )
  {
    result = 92LL;
    goto LABEL_97;
  }
  v124 = a3;
  v125 = a2;
  v27 = 0LL;
  v28 = 0LL;
  v128 = 0LL;
  v130 = v8 + 126;
  v120 = v8 + 48;
  v29 = a5 + 33;
  v121 = v8 + 2;
  v122 = a1;
  v126 = 1;
  v123 = v8;
  while ( 1 )
  {
    if ( v29 < a5 || (v30 = v29 - a5 + 12, v30 > a6) )
    {
      if ( v8[12] )
        goto LABEL_107;
      v46 = 30;
      goto LABEL_104;
    }
    v31 = bswap32(*(_DWORD *)v29);
    if ( (v31 & 0x80000000) != 0 )
    {
      if ( v8[12] )
        goto LABEL_107;
      v46 = 63;
      goto LABEL_104;
    }
    v32 = v31;
    if ( v30 + v31 > a6 || v29 + v31 + 12 < a5 )
    {
      v46 = 64;
LABEL_104:
      *v25 = v46;
      goto LABEL_107;
    }
    v34 = (char *)(v29 + 8);
    if ( (unsigned int)sub_1036B3A50(v29, "IDAT") )
    {
      v35 = v27 + v31;
      if ( !__CFADD__(v27, v31) )
      {
        v36 = v128;
        if ( v128 < v35 )
        {
          if ( 2 * v128 >= v35 )
            v37 = (3 * v35) >> 1;
          else
            v37 = v27 + v31;
          v38 = v37;
          v39 = realloc(v28, v37);
          if ( !v39 )
          {
            v98 = 83;
            goto LABEL_171;
          }
          v28 = v39;
          v36 = v38;
          v8 = v123;
        }
        v128 = v36;
        if ( v31 )
        {
          do
          {
            v40 = *v34++;
            *((_BYTE *)v28 + v27++) = v40;
            --v32;
          }
          while ( v32 );
        }
        else
        {
          LODWORD(v32) = 0;
        }
        v126 = 3;
        goto LABEL_46;
      }
      v46 = 95;
      goto LABEL_103;
    }
    if ( (unsigned int)sub_1036B3A50(v29, "IEND") )
    {
      LODWORD(v32) = 1;
      goto LABEL_43;
    }
    if ( !(unsigned int)sub_1036B3A50(v29, "PLTE") )
      break;
    v42 = sub_1036B55E4(v127, v29 + 8, v31);
    v25 = v130;
    *v130 = v42;
    if ( v42 )
      goto LABEL_107;
    LODWORD(v32) = 0;
    v126 = 2;
LABEL_62:
    v35 = v27;
LABEL_47:
    if ( !v8[10] && (unsigned int)crc32_sub_1036B3AC8(v29) )
    {
      *v25 = 57;
LABEL_106:
      v27 = v35;
      goto LABEL_107;
    }
    if ( (_DWORD)v32 )
      goto LABEL_106;
    v41 = *v25;
    v27 = v35;
LABEL_51:
    v29 = sub_1036B3B20(v29);
    if ( v41 )
      goto LABEL_107;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "tRNS") )
  {
    v43 = sub_1036B56DC(v127, v29 + 8, v31);
LABEL_60:
    v25 = v130;
    *v130 = v43;
    if ( v43 )
      goto LABEL_107;
    LODWORD(v32) = 0;
    goto LABEL_62;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "bKGD") )
  {
    v43 = sub_1036B57D0(v120, v29 + 8, v31);
    goto LABEL_60;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "tEXt") )
  {
    if ( !v8[14] )
      goto LABEL_72;
    v43 = sub_1036B58DC(v120, v29 + 8, v31);
    goto LABEL_60;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "zTXt") )
  {
    if ( !v8[14] )
      goto LABEL_72;
    v43 = sub_1036B5A00(v120, v121, v29 + 8, v31);
    goto LABEL_60;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "iTXt") )
  {
    if ( !v8[14] )
    {
LABEL_72:
      LODWORD(v32) = 0;
      goto LABEL_43;
    }
    v43 = sub_1036B5BB8(v120, v121, v29 + 8, v31);
    goto LABEL_60;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "tIME") )
  {
    if ( v31 == 7 )
    {
      LODWORD(v32) = 0;
      v8[82] = 1;
      v8[83] = *(unsigned __int8 *)(v29 + 9) | (*(unsigned __int8 *)(v29 + 8) << 8);
      v8[84] = *(unsigned __int8 *)(v29 + 10);
      v8[85] = *(unsigned __int8 *)(v29 + 11);
      v8[86] = *(unsigned __int8 *)(v29 + 12);
      v8[87] = *(unsigned __int8 *)(v29 + 13);
      v8[88] = *(unsigned __int8 *)(v29 + 14);
LABEL_76:
      v8[126] = 0;
LABEL_43:
      v35 = v27;
LABEL_46:
      v25 = v130;
      goto LABEL_47;
    }
    v46 = 73;
LABEL_103:
    v25 = v130;
    goto LABEL_104;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "pHYs") )
  {
    v44 = sub_1036B5F60(v120, v29 + 8, v31);
LABEL_79:
    *v130 = v44;
    if ( v44 )
      goto LABEL_205;
    LODWORD(v32) = 0;
    v35 = v27;
    v25 = v130;
    v8 = v123;
    goto LABEL_47;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "gAMA") )
  {
    if ( v31 != 4 )
    {
      v98 = 96;
      goto LABEL_171;
    }
    LODWORD(v32) = 0;
    v8 = v123;
    v123[93] = 1;
    v123[94] = bswap32(*(_DWORD *)(v29 + 8));
    goto LABEL_76;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "cHRM") )
  {
    v44 = sub_1036B5FA4(v120, v29 + 8, v31);
    goto LABEL_79;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "sRGB") )
  {
    if ( v31 != 1 )
    {
      v98 = 98;
      goto LABEL_171;
    }
    LODWORD(v32) = 0;
    v8 = v123;
    v123[104] = 1;
    v123[105] = (unsigned __int8)*v34;
    goto LABEL_76;
  }
  if ( (unsigned int)sub_1036B3A50(v29, "iCCP") )
  {
    v44 = sub_1036B6028(v120, v121, v29 + 8, v31);
    goto LABEL_79;
  }
  if ( v123[11] || (*(_BYTE *)(v29 + 4) & 0x20) != 0 )
  {
    if ( v123[15] )
    {
      v45 = sub_1036B3BB0(&v123[2 * (v126 - 1) + 114], &v123[2 * (v126 - 1) + 120], v29);
      v123[126] = v45;
      if ( v45 )
      {
LABEL_205:
        v25 = v130;
        goto LABEL_172;
      }
    }
    v41 = 0;
    v25 = v130;
    v8 = v123;
    goto LABEL_51;
  }
  v98 = 69;
LABEL_171:
  v25 = v130;
  *v130 = v98;
LABEL_172:
  v8 = v123;
LABEL_107:
  v132 = 0LL;
  v133 = 0LL;
  v131 = 0LL;
  v47 = *v125;
  if ( v8[50] )
  {
    v48 = *v124;
    v49 = (*v124 + 7) >> 3;
    v50 = v127;
    v51 = sub_1036B79FC((unsigned int)(v47 + 7) >> 3, v49, v127);
    v52 = v47 + 3;
    if ( (unsigned int)v47 >= 5 )
      v51 += sub_1036B79FC(v52 >> 3, v49, v127);
    v53 = sub_1036B79FC(v52 >> 2, (v48 + 3) >> 3, v127) + v51;
    v54 = v47 + 1;
    if ( (unsigned int)v47 >= 3 )
      v53 += sub_1036B79FC(v54 >> 2, (v48 + 3) >> 2, v127);
    v55 = sub_1036B79FC(v54 >> 1, (v48 + 1) >> 2, v127) + v53;
    if ( (unsigned int)v47 >= 2 )
      v55 += sub_1036B79FC((unsigned int)v47 >> 1, (v48 + 1) >> 1, v127);
    v56 = sub_1036B79FC(v47, v48 >> 1, v127) + v55;
    v25 = v130;
  }
  else
  {
    v50 = v127;
    v56 = sub_1036B79FC(*v125, *v124, v127);
  }
  if ( !*v25 )
  {
    if ( !v56 )
      goto LABEL_120;
    v57 = realloc(0LL, v56);
    if ( !v57 )
    {
      v60 = 83;
      goto LABEL_130;
    }
    v133 = v56;
    v131 = v57;
    if ( !*v25 )
    {
LABEL_120:
      v58 = (__int64 (__fastcall *)(void **, __int64 *, void *, unsigned __int64, _DWORD *))*((_QWORD *)v8 + 2);
      if ( v58 )
        v59 = v58(&v131, &v132, v28, v27, v121);
      else
        v59 = sub_1036B389C(&v131, &v132, v28, v27, v121);
      if ( v132 != v56 && v59 == 0 )
        v60 = 91;
      else
        v60 = v59;
LABEL_130:
      *v25 = v60;
    }
  }
  free(v28);
  v62 = v125;
  if ( !*v25 )
  {
    v63 = *v125;
    v64 = *v124;
    v66 = v8[52];
    v65 = v8[53];
    v67 = sub_1036B3D08(*v125, *v124, v66, v65);
    v68 = malloc(v67);
    *v122 = v68;
    if ( v68 )
    {
      if ( v67 )
      {
        for ( i = 0LL; i != v67; ++i )
        {
          v68[i] = 0;
          v68 = (_BYTE *)*v122;
        }
        v63 = *v125;
        v64 = *v124;
        v66 = v8[52];
        v65 = v8[53];
      }
      v70 = (char *)v131;
      if ( v66 > 6 )
        v71 = 0;
      else
        v71 = dword_104824040[v66];
      v73 = (unsigned int)(v71 * v65);
      if ( (_DWORD)v73 )
      {
        if ( v8[50] )
        {
          v129 = v64;
          sub_1036B7E10(v138, v137, v136, v135, v134, v63, v64, v73);
          for ( j = 0LL; j != 7; ++j )
          {
            v75 = &v70[v135[j]];
            v76 = v138[j];
            v77 = (unsigned int)v137[j];
            v72 = sub_1036B7A4C(v75, &v70[v136[j]], v76, v77, v73);
            if ( v72 )
            {
              v97 = 0;
              goto LABEL_186;
            }
            if ( (unsigned int)v73 <= 7 )
              sub_1036B7D74(&v70[v134[j]], v75, v76 * (unsigned int)v73, (v76 * (_DWORD)v73 + 7) & 0xFFFFFFF8, v77);
          }
          sub_1036B7E10(v143, v142, v141, v140, v139, v63, v129, v73);
          if ( (unsigned int)v73 <= 7 )
          {
            for ( k = 0LL; k != 7; ++k )
            {
              v100 = v142[k];
              if ( v100 )
              {
                v101 = 0;
                v102 = (unsigned int)v143[k];
                do
                {
                  if ( (_DWORD)v102 )
                  {
                    v103 = 0LL;
                    v104 = 8 * v139[k];
                    v105 = dword_104823FA4[k];
                    v106 = dword_104823FC0[k] + (dword_104823FF8[k] + dword_104823FDC[k] * v101) * v63;
                    do
                    {
                      v107 = (unsigned int)((v106 + v105 * v103) * v73);
                      v108 = v104 + (unsigned int)((v101 * v102 + v103) * v73);
                      v109 = v73;
                      do
                      {
                        v110 = (unsigned __int8)v70[v108 >> 3] >> (~(_BYTE)v108 & 7);
                        ++v108;
                        if ( (v110 & 1) != 0 )
                          v68[v107 >> 3] |= (v110 & 1) << (~(_BYTE)v107 & 7);
                        ++v107;
                        --v109;
                      }
                      while ( v109 );
                      ++v103;
                    }
                    while ( v103 != v102 );
                  }
                  ++v101;
                }
                while ( v101 != v100 );
              }
            }
          }
          else
          {
            v78 = 0LL;
            v79 = (unsigned int)v73 >> 3;
            do
            {
              v80 = v142[v78];
              if ( v80 )
              {
                v81 = 0;
                v82 = 0;
                v83 = (unsigned int)v143[v78];
                do
                {
                  if ( (_DWORD)v83 )
                  {
                    v84 = 0LL;
                    v85 = dword_104823FA4[v78];
                    v86 = dword_104823FC0[v78] + v63 * (dword_104823FF8[v78] + dword_104823FDC[v78] * v82);
                    v87 = &v70[v139[v78]];
                    v88 = v81;
                    do
                    {
                      if ( (_DWORD)v79 )
                      {
                        v89 = &v68[v79 * v86];
                        v90 = &v87[v79 * v88];
                        v91 = (unsigned int)v73 >> 3;
                        do
                        {
                          v92 = *v90++;
                          *v89++ = v92;
                          --v91;
                        }
                        while ( v91 );
                      }
                      ++v84;
                      v86 += v85;
                      ++v88;
                    }
                    while ( v84 != v83 );
                  }
                  ++v82;
                  v81 += v83;
                }
                while ( v82 != v80 );
              }
              ++v78;
            }
            while ( v78 != 7 );
          }
          v72 = 0;
          v97 = 1;
LABEL_186:
          v50 = v127;
          v62 = v125;
          if ( v97 )
LABEL_187:
            v72 = 0;
        }
        else
        {
          v93 = v64;
          v94 = v73 * v63;
          v95 = (v73 * v63 + 7) & 0xFFFFFFF8;
          if ( (unsigned int)v73 > 7 || v94 == v95 )
          {
            v72 = sub_1036B7A4C(v68, v131, v63, v93, v73);
            if ( !v72 )
              goto LABEL_187;
          }
          else
          {
            v118 = v63;
            v119 = v93;
            v72 = sub_1036B7A4C(v131, v131, v118, v93, v73);
            if ( !v72 )
            {
              sub_1036B7D74(v68, v70, v94, v95, v119);
              goto LABEL_187;
            }
          }
        }
      }
      else
      {
        v72 = 31;
      }
    }
    else
    {
      v72 = 83;
    }
    v25 = v130;
    *v130 = v72;
  }
  v132 = 0LL;
  v133 = 0LL;
  free(v131);
  result = (unsigned int)*v25;
  if ( !(_DWORD)result )
  {
    if ( v8[13] )
    {
      if ( (unsigned int)sub_1036B4698(v8 + 38, v50) )
        return 0LL;
      v111 = (_BYTE *)*v122;
      v112 = v8[38];
      if ( (v112 | 4) != 6 && v8[39] != 8 )
        return 56LL;
      v113 = *v62;
      v114 = *v124;
      v115 = sub_1036B3D08(*v62, *v124, v112, v8[39]);
      v116 = malloc(v115);
      *v122 = v116;
      if ( v116 )
        v117 = sub_1036B4148(v116, v111, v8 + 38, v50, v113, v114);
      else
        v117 = 83;
      *v25 = v117;
      free(v111);
      return (unsigned int)*v25;
    }
    result = sub_1036B3C4C(v8 + 38, v50);
LABEL_97:
    *v25 = result;
  }
  return result;
}

```

解析返回 PIC 数据 (部分):

```
00000001050CDA00  2E 50 49 43 90 01 00 00  10 02 00 00 AC 02 00 00  .PIC............
00000001050CDA10  01 A3 A0 68 E4 0A 6B 23  00 00 00 00 E2 D2 82 82  ...h...#........
00000001050CDA20  4F 62 7C 40 82 53 99 C0  4F 26 5B 8E E0 2C 66 D5  Ob|@.S...&[.....
00000001050CDA30  8A A6 9D 5D 6D 32 E7 CD  1A AC C0 10 93 40 49 3E  ...]m2.......@I>
00000001050CDA40  C0 94 99 F5 F6 8C 6D DB  76 67 81 E7 4E BC 73 92  .........g....s.
00000001050CDA50  4C 31 D4 CD E5 5D AC 48  D0 64 1E D7 5B 2B BA 2E  L1.....H.....+..
00000001050CDA60  8C EC 98 E9 9B 4A 0D 3D  81 45 FD 58 49 25 94 47  .....J.=.E.XI%.G
00000001050CDA70  62 61 48 45 4A B9 43 87  59 E9 8C 6D D5 EE C2 B7  baHEJ.C.Y.....·
00000001050CDA80  0C D0 29 4C 10 D6 98 CE  7B E1 90 A9 40 4D 9E 3F  ...L.....ᐩ  @M.?
00000001050CDA90  28 D1 8E ED 8E 14 8B AD  CB 21 CB 0F FF 6D 29 1D  (ю ..........m).
00000001050CDAA0  A6 88 9B 84 E9 38 DC 5F  E8 B1 06 14 2A 39 90 5B  ......._....*9.[

```

#### 4.3、解密 PIC 数据

生成 AES KEY  
将上面计算得到的 hmac 值转换成 16 进制:

```
39d3b771767409f5e74f4b579b868a5c0192131861c1791c833b5c95e99c412b
 
__text:0000000101B83748             loc_101B83748
__text:0000000101B83748 F7 B3 00 78 STURH           W23, [SP,#0xB]
__text:0000000101B8374C C8 02 40 39 LDRB            W8, [X22]
__text:0000000101B83750 E8 37 00 39 STRB            W8, [SP,#0xD]
__text:0000000101B83754 C8 06 40 39 LDRB            W8, [X22,#1]
__text:0000000101B83758 E8 3B 00 39 STRB            W8, [SP,#0xE]
__text:0000000101B8375C FF 3F 00 39 STRB            WZR, [SP,#0xF]
__text:0000000101B83760 E0 2F 00 91 ADD             X0, SP, #0xB
__text:0000000101B83764 01 00 80 D2 MOV             X1, #0
__text:0000000101B83768 02 00 80 52 MOV             W2, #0
__text:0000000101B8376C D3 51 42 94 BL              _strtol
__text:0000000101B83770 20 17 00 38 STRB            W0, [X25],#1
__text:0000000101B83774 D6 0A 00 91 ADD             X22, X22, #2
__text:0000000101B83778 18 07 00 F1 SUBS            X24, X24, #1
__text:0000000101B8377C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B83780 41 FE FF 54 B.NE            loc_101B83748
__text:0000000101B83784 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B83788 A0 00 00 18 LDR             W0, =1
__text:0000000101B8378C C4 FF FF 17 B               loc_101B8369C

```

转换后:

```
0000000281E86790  39 D3 B7 71 76 74 09 F5  E7 4F 4B 57 9B 86 8A 5C
0000000281E867A0  01 92 13 18 61 C1 79 1C  83 3B 5C 95 E9 9C 41 2B

```

取转换后的的前 0x10 字节生成最终的 AES KEY

```
__text:0000000101B2170C             GenAesKey_loc_102FD570C
__text:0000000101B2170C 8D 69 68 38 LDRB            W13, [X12,X8]           ; 生成解密PIC的AES KEY
__text:0000000101B21710 AE 2D 1C 11 ADD             W14, W13, #0x70B
__text:0000000101B21714 CF 09 C9 1A UDIV            W15, W14, W9
__text:0000000101B21718 EE B9 09 1B MSUB            W14, W15, W9, W14
__text:0000000101B2171C AF 0D 2F 11 ADD             W15, W13, #0xBC3
__text:0000000101B21720 F0 09 CA 1A UDIV            W16, W15, W10
__text:0000000101B21724 0F BE 0A 1B MSUB            W15, W16, W10, W15
__text:0000000101B21728 EE 39 09 1B MADD            W14, W15, W9, W14
__text:0000000101B2172C CE 05 0E 0B ADD             W14, W14, W14,LSL#1
__text:0000000101B21730 CE 09 00 11 ADD             W14, W14, #2
__text:0000000101B21734 6E 49 6E 38 LDRB            W14, [X11,W14,UXTW]     ; 查表
__text:0000000101B21738 AD 01 0E 0A AND             W13, W13, W14
__text:0000000101B2173C AD 19 1F 12 AND             W13, W13, #0xFE
__text:0000000101B21740 8D 69 28 38 STRB            W13, [X12,X8]           ; 存AES KEY
__text:0000000101B21744 08 05 00 91 ADD             X8, X8, #1
__text:0000000101B21748 1F 41 00 F1 CMP             X8, #0x10
__text:0000000101B2174C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B21750 E1 FD FF 54 B.NE            GenAesKey_loc_102FD570C ; 生成解密PIC的AES KEY

```

生成后的 AESKEY:

```
38 90 B6 70 76 74 00 C0  E6 4E 4A 02 98 80 8A 1C

```

AES 解密 PIC 数据:

```
__text:0000000101B1FA10 82 07 00 94 BL              DecPic_loc_105235818    ; x0:第1个指针pic数据,X1:key
__text:0000000101B1FA14 E8 03 00 AA MOV             X8, X0                  ; X0:返回解密后的pic明文值
 
//获取IV  0102030405060708
//AES解密
__text:0000000101B1ABCC E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:0000000101B1ABD0 E0 03 00 91 MOV             X0, SP
__text:0000000101B1ABD4 82 1E 80 52 MOV             W2, #0xF4
__text:0000000101B1ABD8 01 00 80 52 MOV             W1, #0
__text:0000000101B1ABDC BA F1 43 94 BL              _memset
__text:0000000101B1ABE0 E1 03 19 32 MOV             W1, #0x80
__text:0000000101B1ABE4 E2 03 00 91 MOV             X2, SP
__text:0000000101B1ABE8 E0 03 18 AA MOV             X0, X24
__text:0000000101B1ABEC 69 51 FF 97 BL              InitKey_sub_102CBF190   ; 初始化KEY
__text:0000000101B1ABF0 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B1ABF4 E3 03 00 91 MOV             X3, SP
__text:0000000101B1ABF8 E0 03 16 AA MOV             X0, X22
__text:0000000101B1ABFC E1 03 13 AA MOV             X1, X19
__text:0000000101B1AC00 E2 03 15 AA MOV             X2, X21
__text:0000000101B1AC04 E4 03 17 AA MOV             X4, X23
__text:0000000101B1AC08 05 00 80 52 MOV             W5, #0
__text:0000000101B1AC0C C9 53 FF 97 BL              Aes_Enc_Dec_sub_102CBFB30 ; 加密时:X1:原始数据,X2:大小,X3:初始化后的key,X4:IV,X5:模式:0:解密,1:加密
__text:0000000101B1AC0C                                                     ; 解密时:X0:原始数据,x1:返回,X2:大小,X3:初始化后的key,X4:IV,X5:模式:0:解密,1:加密
__text:0000000101B1AC10 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000101B1AC14 E0 03 13 AA MOV             X0, X19
__text:0000000101B1AC18 E1 03 15 AA MOV             X1, X21

```

解密后数据 (部分)

```
0000000104486020  78 9C 45 92 6D 6F DA 30  14 85 FF 4B B4 7D DA 44  x.E.mo.....K.}..
0000000104486030  FD 9E 18 69 5A CB DA 91  C2 02 2D B4 94 F0 ED 3A  ...iZ.....-.....
0000000104486040  76 C0 4D 42 50 20 6D 42  C7 7F 9F 3D 4D 9A 64 E9  v..BP mB...=M.d.
0000000104486050  DA 47 D7 CF 3D 3E F2 47  00 38 18 E2 AF AE A0 60  ....=>.....⯮  .`
0000000104486060  C8 10 72 3B 12 0C 83 AC  AE 06 0A 6A B7 C0 66 3B  ..r;.......j...;
0000000104486070  3B B0 95 AB 60 03 DF E8  2E 04 08 42 A1 29 A3 3C  ;...`......B.).<

```

解压缩 AES 解密后数据得到明文，后面的加解密数据都会使用里面数据做为 KEY:

```
{"a1":1,"a10":400,"a2":"com.baobaoaichi.imaicai","a11":"0a76d34357f7c7859c1a3fd25516b4e4021ec931fd56b6a36ebf73e5aa34c406","a3":"b9eb65dbc4c7109259edc07826390faf3bd09e3920d66580b04a0853d3ee172b","a4":5230,"k0":{"k1":"meituan1sankuai0","k2":"meituan0sankuai1","k3":"$MXMYBS@HelloPay","k4":"Maoyan010iauknaS","k5":"34281a9dw2i701d4","k6":"X%rj@KiuU+|xY}?f"},"a5":"5.23.0","a0":"sdk9xWZTg5V9nKAxVFB5mB1ipZIJGmYSysreJ1f/rlvXJ7Ydxd3hJRdWb4QdZKr/","a6":"HdPfNPzY9GK6wzp0lEgaMaX06uEMke8y0H3eD0l4RapMpRmVaOWzyQkHMmOavR47","a7":"1yuZHjO43la6rhDXzMkjGiseg9yoRxxDtzwourYASiiAp4Yl0TUGvOiN4UcoJ6pQ","c0":{"c1":true,"c2":false},"a9":"gC4xYEhYfboH/8kOYsdIcbyYRTKfrVgmHLb3x8uNBag=","a8":1627281940842}

```

解析上面的 json 数据获取对应的 key:

```
__text:0000000101AFF460 04 8E 00 94 BL              getPICkey_sub_102CF2C70 ; x0:返回pic json中的key
__text:0000000101B243E4 BF EA FF 97 BL              ParsingJsonGetPicKEY_loc_105232EE0 ; x0:返回key
__text:0000000101B1F050 3D 00 00 94 BL              PasingPicKey_loc_105187144 ; 解析pic获取key
 
__text:0000000101B1F144             PasingPicKey_loc_105187144
__text:0000000101B1F144 E0 7B 3F A9 STP             X0, X30, [SP,#-0x10]
__text:0000000101B1F148 0F 00 00 94 BL              sub_101B1F184
__text:0000000101B1F14C 74 C8 65 4A EON             W20, W3, W5,LSR#50
__text:0000000101B1F14C 
__text:0000000101B1F150 92 64 9A 0F+DCQ 0x16FC3DEB0F9A6492, 0x3C2CBC796F6AD130, 0xD907DA6BC5A30DF2, 0xD2EA795466662B7B
__text:0000000101B1F150 EB 3D FC 16+DCQ 0xBAC13E778CE19C3, 0x227363C9AEC0FB96
__text:0000000101B1F180           
__text:0000000101B1F180 F7 8B 97 A9 STP             X23, X2, [SP,#0x178]!
__text:0000000101B1F184
__text:0000000101B1F184 
__text:0000000101B1F184
__text:0000000101B1F184
__text:0000000101B1F184             sub_101B1F184
__text:0000000101B1F184 40 01 00 10 ADR             X0, loc_101B1F1AC
__text:0000000101B1F188 FE 03 00 AA MOV             X30, X0
__text:0000000101B1F18C C0 03 5F D6 RET

```

### [](#五、反爬签名流程)五、反爬签名流程

#### 5.1、APP 防爬背景

App 防爬主要通过对 App 客户端发起的请求进行签名。然后将签名与业务数据请求发送到服务器端，服务端 WAF 应用服务器收到的请求后，通过解析签名串进行风险识别、拦截恶意请求, 通过校验 App 请求签名，识别 App 业务中的风险、拦截恶意请求，实现 App 防护的目的。  
如图 5-1 所示，请求头中携带的签名信息:  
![](https://bbs.pediy.com/upload/attach/202111/536985_MFQS8YMUSHJE3BN.png)  
         图 5-1  
识别异常爬虫:  
App 签名异常：对使用未携带签名或签名非法的 App 访问防爬防护目标的请求进行检测和拦截。  
设备特征异常：检测设备的异常特征，是否使用模拟器、使用代理、Root 设备、HOOK 框架等。  
基于以上逻辑，所以 app 会检测客户端环境。

#### 5.2、扫描设备风险

检测越狱、hook 框架等, 风险特征:

```
{
    "0": 5,
    "1": ["/var/mobile/iGrimace", "/var/mobile/Library/Preferences/orgioshackigrimaceadvplist", "/var/mobile/Library/Preferences/com007gaijiselappplist", "/Library/MobileSubstrate/DynamicLibraries/ALSplist", "/Library/MobileSubstrate/DynamicLibraries/rstweakplist", "/Library/MobileSubstrate/DynamicLibraries/AXJplist", "/Library/MobileSubstrate/DynamicLibraries/fakephonelibplist", "/Library/MobileSubstrate/DynamicLibraries/IGGplist", "/Library/MobileSubstrate/DynamicLibraries/AWZplist", "/Library/MobileSubstrate/DynamicLibraries/iGrimaceX9Tweakplist", "/Library/MobileSubstrate/DynamicLibraries/igvxplist", "/Library/MobileSubstrate/DynamicLibraries/R8plist", "/Library/MobileSubstrate/DynamicLibraries/iGrimaceplist", "/Library/MobileSubstrate/DynamicLibraries/V8Eplist", "/Library/MobileSubstrate/DynamicLibraries/zorroplist", "/Applications/NZTapp", "/Applications/AWZapp", "/var/mobile/awzdata", "/var/mobile/hdFaker", "/var/mobile/NZTResultplist", "/usr/bin/XGenDaemondylib", "/var/mobile/GFaker", "/var/mobile/nztdata", "/usr/bin/iGevo", "/var/root/Forge9_fix", "/var/root/igvx_fix", "/var/root/igvx_flag", "/var/mobile/Library/XXAssistant/Lua/LocalLuas/", "/Library/ApplicationSupport/XXAssistant/Lua/LocalLuas/", "/var/root/igfix", "/var/root/igflag", "/var/root/R8_fix", "/Library/ApplicationSupport/XXAssistant/Lua/Luas/Temp/public", "/var/mobile/Library/XXIDEHelper/xsp/", "/Library/ApplicationSupport/XXIDEHelper/xsp/", "/var/mobile/Library/XXAssistant/Lua/Luas/Temp/public", "/Applications/HiddenApiapp", "/Applications/Xgenapp", "/Applications/BirdFaker9app", "/Applications/VPNMasterProapp", "/Applications/GuizmOVPNapp", "/Applications/AXJapp", "/var/touchelf/scripts/", "/var/mobile/Media/TouchSprite/lua/", "/Applications/iGapp", "/Applications/Forge9app", "/Applications/Forgeapp", "/Applications/GFakerapp", "/Applications/hdfakersetapp", "/Applications/R8app", "/Applications/Pranavaapp", "/Applications/RSTapp", "/Applications/WujiVPNapp", "/Applications/TouchSpriteapp", "/Applications/TouchElfapp", "/Applications/igvxapp", "/usr/sbin/frida-server"],
    "2": ["hdfakerdylib", "quickdobdylib", "Unfloddylib", "SogouInputIPhonedylib", "MTTweakdylib", "iAcessdylib", "NZTdylib", "OTRLocationdylib", "txytweakdylib", "GPSTravellerTweakdylib", "GPSTravellerTweak360dylib", "SkyWalkerBaseTweakdylib", "Lithiumdylib", "akLocationXdylib", "daniutweakdylib", "gpsmanagerplugindylib", "pbyydylib", "jbreakdylib", "GPSCheatdylib", "GPSTravellerTweakProXdylib", "zlocaspritiExdylib", "locationexpertplugindylib", "GPSTravellerTweakVIPdylib", "meituanadylib", "MATweakExdylib", "fakegpsplugindylib", "TEGPSdylib", "MFTweakExdylib", "Aerialdylib", "rstweakdylib", "Relocatedylib", "AWZdylib", "gpsmasterplugindylib", "ALSdylib", "zAdaptiveKeyboarddylib", "CatSysHelperdylib", "heiying518TweakExdylib", "0Shadowdylib", "UnSubdylib", "zzzzLibertydylib", "Libertydylib", "xCondylib", "Libertasdylib"],
    "3": ["comsengledprotocolbtspeaker", "comgpsmockgps", "comcompanyaccessory", "comqzbzsproqpro1810", "commmpmmp", "comlocaspritihw", "comlogitechm100"],
    "4": [
        ["UIDevice", "systemVersion", "/System/Library/PrivateFrameworks/UIKitCoreframework/UIKitCore"],
        ["UIDevice", "identifierForVendor", "/System/Library/PrivateFrameworks/UIKitCoreframework/UIKitCore"],
        ["UIDevice", "model", "/System/Library/PrivateFrameworks/UIKitCoreframework/UIKitCore"],
        ["UIDevice", "hwmodel", "/usr/lib/libobjcAdylib"],
        ["NSBundle", "executablePath", "/System/Library/Frameworks/Foundationframework/Foundation"],
        ["NSBundle", "bundleIdentifier", "/System/Library/Frameworks/Foundationframework/Foundation"]
    ],
    "5": ["/usr/sbin/frida-server", "/etc/apt/sourceslistd/electralist", "/etc/apt/sourceslistd/sileosources", "/bootstrapped_electra", "/usr/lib/libjailbreakdylib", "/jb/lzma", "/cydia_no_stash", "/installed_unc0ver", "/jb/offsetsplist", "/usr/share/jailbreak/injectmeplist", "/etc/apt/undecimus/undecimuslist", "/var/lib/dpkg/info/mobilesubstratemd5sums", "/jb/jailbreakdplist", "/jb/amfid_payloaddylib", "/jb/libjailbreakdylib", "/usr/libexec/cydia/firmwaresh", "/var/lib/cydia", "/private/var/Users/", "/var/log/apt", "/private/var/lib/apt/", "/private/var/cache/apt/", "/private/var/log/syslog", "/Applications/blackra1napp", "/Applications/FakeCarrierapp", "/private/var/mobile/Library/SBSettings/Themes", "/Library/MobileSubstrate/CydiaSubstratedylib", "/Applications/Sileoapp", "/var/binpack", "/Library/PreferenceBundles/LibertyPrefbundle", "/Library/PreferenceBundles/ShadowPreferencesbundle", "/Library/PreferenceBundles/ABypassPrefsbundle", "/Library/PreferenceBundles/FlyJBPrefsbundle", "/usr/lib/libhookerdylib", "/usr/lib/libsubstitutedylib", "/usr/lib/substrate", "/usr/lib/TweakInject"],
    "6": [],
    "7": [],
    "8": [],
    "9": [],
    "10": [],
}

```

检测后结果:

```
{
    "b1": "{\"7\":\"-\",\"3\":\"-\",\"4\":\"-\",\"5\":\"-\",\"1\":\"-\",\"33\":\"{\\\"7\\\":\\\"-\\\",\\\"3\\\":\\\"-\\\",\\\"8\\\":\\\"-\\\",\\\"4\\\":\\\"-\\\",\\\"0\\\":\\\"-\\\",\\\"9\\\":\\\"-\\\",\\\"5\\\":\\\"-\\\",\\\"1\\\":\\\"-\\\",\\\"6\\\":\\\"-\\\",\\\"2\\\":\\\"-\\\",\\\"10\\\":\\\"-\\\"}\",\"6\":\"-\",\"2\":\"-\"}",
    "b2": 1,
    "b3": 1,
    "b4": "com.baobaoaichi.imaicai",
    "b5": "5.25.0",
    "b6": 5250,
    "b7": 1635652007,
    "b8": 1635650608,
    "b9": 1635650608,
    "b10": "5.2.11",
    "b11": "5.2.11",
    "b12": 2
}

```

压缩后 json

```
0000000107C97080  78 9C 6D 90 4D 0E 84 30  08 85 EF C2 DA 69 A0 4A  x.m.M..0.....i.J
0000000107C97090  D5 9E 85 4D 75 33 5D 18  0F 60 BC FB C0 18 93 8E  ՞ .Mu3]..`......
0000000107C970A0  43 D2 84 F7 3E 7E 5A 7A  C0 42 90 E1 10 18 05 B2  C......z........
0000000107C970B0  C0 4B A0 13 E8 1B 3D 34  9A 1B 4D 6D FD D5 70 88  .......4..Mm....
0000000107C970C0  D8 1C 31 23 96 13 CB 8A  CD FB 67 93 C3 06 87 A1  ..1#..ˊ ..g.....
0000000107C970D0  C3 66 87 B1 C3 C8 61 C9  61 D1 EB 7D 5C 7C 7E 37  ......a....}\|~7
0000000107C970E0  4D CD D6 F1 D6 27 74 B0  44 C8 A4 A1 BF C2 A0 9F  M.......DȤ ..  .
0000000107C970F0  BA EE 5B 58 CA AE A7 D4  F5 5D 43 DD 34 96 6A C5  ....ʮ ...]C...j.
0000000107C97100  AC 69 0E 91 03 9A 4B 90  39 32 AA 1A B5 3B F5 9C  .i....K.92...;..
0000000107C97110  38 22 8E EA A7 DB 63 C2  49 FD FC F0 84 D7 A0 40  8"....c........@

```

RC4 加密压缩后数据:  
组合加密密钥：

```
1635653901 6d1efb41-1bb2-4db1-88ee-b89d21d06e5f  //当前时间加获取appkey(ak:info.plist)

```

加密:

```
__text:00000001052F9848             Rc4Enc_sub_10310D848
__text:00000001052F9848
__text:00000001052F9848             var_118= -0x118
__text:00000001052F9848             var_18= -0x18
__text:00000001052F9848             var_10= -0x10
__text:00000001052F9848             var_s0=  0
__text:00000001052F9848
__text:00000001052F9848 FF C3 04 D1 SUB             SP, SP, #0x130
__text:00000001052F984C FC 6F 11 A9 STP             X28, X27, [SP,#0x120+var_10]
__text:00000001052F9850 FD 7B 12 A9 STP             X29, X30, [SP,#0x120+var_s0]
__text:00000001052F9854 FD 83 04 91 ADD             X29, SP, #0x120
__text:00000001052F9858 68 BC 00 D0 ADRP            X8, #___stack_chk_guard_ptr@PAGE
__text:00000001052F985C 08 FD 43 F9 LDR             X8, [X8,#___stack_chk_guard_ptr@PAGEOFF]
__text:00000001052F9860 08 01 40 F9 LDR             X8, [X8]
__text:00000001052F9864 A8 83 1E F8 STUR            X8, [X29,#var_18]
__text:00000001052F9868 41 06 00 34 CBZ             W1, loc_1052F9930
__text:00000001052F986C 08 00 80 D2 MOV             X8, #0
__text:00000001052F9870 29 8B 00 B0 ADRP            X9, #qword_10645E4D8@PAGE
__text:00000001052F9874 20 6D 42 FD LDR             D0, [X9,#qword_10645E4D8@PAGEOFF]
__text:00000001052F9878 E9 23 00 91 ADD             X9, SP, #0x120+var_118
__text:00000001052F987C 01 E5 00 0F MOVI            V1.8B, #8
__text:00000001052F9880
__text:00000001052F9880             loc_1052F9880
__text:00000001052F9880 20 69 28 FC STR             D0, [X9,X8]
__text:00000001052F9884 08 21 00 91 ADD             X8, X8, #8
__text:00000001052F9888 00 84 21 0E ADD             V0.8B, V0.8B, V1.8B
__text:00000001052F988C 1F 01 04 F1 CMP             X8, #0x100
__text:00000001052F9890 81 FF FF 54 B.NE            loc_1052F9880
__text:00000001052F9894 08 00 80 D2 MOV             X8, #0
__text:00000001052F9898 0A 00 80 52 MOV             W10, #0
__text:00000001052F989C E9 23 00 91 ADD             X9, SP, #0x120+var_118
__text:00000001052F98A0
__text:00000001052F98A0             loc_1052F98A0
__text:00000001052F98A0 2B 69 68 38 LDRB            W11, [X9,X8]
__text:00000001052F98A4 4A 01 0B 0B ADD             W10, W10, W11
__text:00000001052F98A8 0C 0D C1 1A SDIV            W12, W8, W1
__text:00000001052F98AC 8C A1 01 1B MSUB            W12, W12, W1, W8
__text:00000001052F98B0 0C 48 6C 38 LDRB            W12, [X0,W12,UXTW]
__text:00000001052F98B4 4A 01 0C 0B ADD             W10, W10, W12
__text:00000001052F98B8 4A 1D 00 12 AND             W10, W10, #0xFF
__text:00000001052F98BC 2C 49 6A 38 LDRB            W12, [X9,W10,UXTW]
__text:00000001052F98C0 2C 69 28 38 STRB            W12, [X9,X8]
__text:00000001052F98C4 2B 49 2A 38 STRB            W11, [X9,W10,UXTW]
__text:00000001052F98C8 08 05 00 91 ADD             X8, X8, #1
__text:00000001052F98CC 1F 01 04 F1 CMP             X8, #0x100
__text:00000001052F98D0 81 FE FF 54 B.NE            loc_1052F98A0
__text:00000001052F98D4 7F 04 00 71 CMP             W3, #1
__text:00000001052F98D8 CB 02 00 54 B.LT            loc_1052F9930
__text:00000001052F98DC 08 00 80 52 MOV             W8, #0
__text:00000001052F98E0 09 00 80 52 MOV             W9, #0
__text:00000001052F98E4 EA 23 00 91 ADD             X10, SP, #0x120+var_118
__text:00000001052F98E8 EB 03 03 2A MOV             W11, W3
__text:00000001052F98EC
__text:00000001052F98EC             loc_1052F98EC
__text:00000001052F98EC 29 05 00 11 ADD             W9, W9, #1
__text:00000001052F98F0 29 1D 00 12 AND             W9, W9, #0xFF
__text:00000001052F98F4 4C 49 69 38 LDRB            W12, [X10,W9,UXTW]
__text:00000001052F98F8 08 01 0C 0B ADD             W8, W8, W12
__text:00000001052F98FC 08 1D 00 12 AND             W8, W8, #0xFF
__text:00000001052F9900 4D 49 68 38 LDRB            W13, [X10,W8,UXTW]
__text:00000001052F9904 4D 49 29 38 STRB            W13, [X10,W9,UXTW]
__text:00000001052F9908 4C 49 28 38 STRB            W12, [X10,W8,UXTW]
__text:00000001052F990C 4D 00 40 39 LDRB            W13, [X2]
__text:00000001052F9910 4E 49 69 38 LDRB            W14, [X10,W9,UXTW]
__text:00000001052F9914 CC 01 0C 0B ADD             W12, W14, W12
__text:00000001052F9918 8C 1D 00 12 AND             W12, W12, #0xFF
__text:00000001052F991C 4C 49 6C 38 LDRB            W12, [X10,W12,UXTW]
__text:00000001052F9920 8C 01 0D 4A EOR             W12, W12, W13
__text:00000001052F9924 4C 14 00 38 STRB            W12, [X2],#1
__text:00000001052F9928 6B 05 00 F1 SUBS            X11, X11, #1
__text:00000001052F992C 01 FE FF 54 B.NE            loc_1052F98EC
__text:00000001052F9930
__text:00000001052F9930             loc_1052F9930
__text:00000001052F9930      
__text:00000001052F9930 A8 83 5E F8 LDUR            X8, [X29,#var_18]
__text:00000001052F9934 69 BC 00 D0 ADRP            X9, #___stack_chk_guard_ptr@PAGE
__text:00000001052F9938 29 FD 43 F9 LDR             X9, [X9,#___stack_chk_guard_ptr@PAGEOFF]
__text:00000001052F993C 29 01 40 F9 LDR             X9, [X9]
__text:00000001052F9940 3F 01 08 EB CMP             X9, X8
__text:00000001052F9944 A1 00 00 54 B.NE            loc_1052F9958
__text:00000001052F9948 FD 7B 52 A9 LDP             X29, X30, [SP,#0x120+var_s0]
__text:00000001052F994C FC 6F 51 A9 LDP             X28, X27, [SP,#0x120+var_10]
__text:00000001052F9950 FF C3 04 91 ADD             SP, SP, #0x130
__text:00000001052F9954 C0 03 5F D6 RET
__text:00000001052F9958  
__text:00000001052F9958
__text:00000001052F9958             loc_1052F9958 
__text:00000001052F9958 8F C2 43 94 BL              ___stack_chk_fail
__text:00000001052F9958             ; End of function Rc4Enc_sub_10310D848
__text:00000001052F9958
__text:00000001052F9958        
__text:00000001052F995C             ; id __cdecl +[SAKDFPIDTimeStamp sharedManager](id, SEL)
__text:00000001052F995C FF 03 01 D1 __SAKDFPIDTimeStamp_sharedManager_ DCD 0xD10103FF
__text:00000001052F995C                                                   
__text:00000001052F9960 E0 FB 01 A9+DCQ 0x94000005A901FBE0, 0x7B1CA1F15A7EA9B6, 0xD5B672DB5B30C29E
__text:00000001052F9978
__text:00000001052F9978        
__text:00000001052F9978
__text:00000001052F9978
__text:00000001052F9978             sub_1052F9978
__text:00000001052F9978 80 01 00 10 ADR             X0, qword_1052F99A8
__text:00000001052F997C FE 03 00 AA MOV             X30, X0
__text:00000001052F9980 FF 03 01 91 ADD             SP, SP, #0x40 ; '@'
__text:00000001052F9984 C0 03 5F D6 RET
__text:00000001052F9984             ; End of function sub_1052F9978
__text:00000001052F9984
__text:00000001052F9984           
__text:00000001052F9988 3A E6 C9 EB+DCQ 0x9154F1ADEBC9E63A, 0x8279F50529021640, 0xEF7B3AB1903E210A, 0xC58C700A3623B980
__text:00000001052F99A8 E0 FB 7D A9+qword_1052F99A8 DCQ 0xA93B7BE0A97DFBE0, 0xD2800020180000C0, 0x9400000314000006, 0x72C5000014BC
__text:00000001052F99A8 E0 7B 3B A9+                                        ; DATA XREF: sub_1052F9978↑o
__text:00000001052F99C8 03 00 00 00 DCD 3
__text:00000001052F99CC CB 00 00 00 DCD 0xCB
__text:00000001052F99D0 EE 49 FF 97+DCQ 0x4C97FF49EE, 0x4400000010, 0xA97B7BE091000000, 0xF944D508B000F1C8
__text:00000001052F99D0 4C 00 00 00+DCQ 0xA93B7BE0B100051F, 0xA93B7BE0540001C1, 0xD280004018000080, 0x9400000117FFFFF2
__text:00000001052F99D0 10 00 00 00+DCQ 0x8A00000005, 0xA93B7BE0A97B7BE0, 0xB000F1C8A97B7BE0, 0x1443C6BFF944D100
__text:00000001052F99D0 44 00 00 00+DCQ 0x910003FDA9BF7BFD, 0x9126A000B000F1C0, 0x9138A0219000BE21, 0xA8C17BFD9443C367
__text:00000001052F99D0 00 00 00 91+DCQ 0x18000100A93B7BE0, 0x17FFFFDDD2800000, 0x65DA94000005, 0x8CBA00003503
__text:00000001052F99D0 E0 7B 7B A9+DCQ 0xB7AE
__text:00000001052F9A78 0E 00 00 00 DCB 0xE, 0, 0, 0
__text:00000001052F9A7C           
__text:00000001052F9A7C
__text:00000001052F9A7C             ; void __cdecl loc_1052F9A7C(id)
__text:00000001052F9A7C             loc_1052F9A7C                           ; DATA XREF: __const:0000000106ABDE28↓o
__text:00000001052F9A7C E0 7B 3F A9 STP             X0, X30, [SP,#-0x10]
__text:00000001052F9A80 0F 00 00 94 BL              sub_1052F9ABC
__text:00000001052F9A84 77 CE 22 D9 STG             X23, [X19,#0x2C0]!
__text:00000001052F9A84          
__text:00000001052F9A88 DD AA 5D 7E+DCQ 0x67FAB5857E5DAADD, 0x636F90947C5C6294, 0x64C3CAEA9D743E2D, 0xF71751CA5B2EBCED
__text:00000001052F9A88 85 B5 FA 67+DCQ 0x294A37644F0A024C, 0xB85C5F3824B80A03
__text:00000001052F9AB8 6F 8E 76 44 DCB 0x6F, 0x8E, 0x76, 0x44
__text:00000001052F9ABC
__text:00000001052F9ABC       
__text:00000001052F9ABC
__text:00000001052F9ABC
__text:00000001052F9ABC             sub_1052F9ABC
__text:00000001052F9ABC 40 01 00 10 ADR             X0, loc_1052F9AE4
__text:00000001052F9AC0 FE 03 00 AA MOV             X30, X0
__text:00000001052F9AC4 C0 03 5F D6 RET
__text:00000001052F9AC4             ; End of function sub_1052F9ABC
__text:00000001052F9AC4
__text:00000001052F9AC4            
__text:00000001052F9AC8 DA 60 A5 84+DCQ 0x87C42B9D84A560DA, 0xFBEA799E68262D89, 0x71110D6F56979F07
__text:00000001052F9AE0            
__text:00000001052F9AE0 24 E1 09 B3 BFXIL           X4, X9, #9, #0x30 ; '0'
__text:00000001052F9AE4
__text:00000001052F9AE4             loc_1052F9AE4                           ; DATA XREF: sub_1052F9ABC↑o
__text:00000001052F9AE4 E0 7B 7F A9 LDP             X0, X30, [SP,#-0x10]
__text:00000001052F9AE8 FD 7B BF A9 STP             X29, X30, [SP,#-0x10]!
__text:00000001052F9AEC FD 03 00 91 MOV             X29, SP
__text:00000001052F9AF0 A8 E9 00 D0 ADRP            X8, #classRef_SAKDFPIDTimeStamp@PAGE
__text:00000001052F9AF4 00 29 43 F9 LDR             X0, [X8,#classRef_SAKDFPIDTimeStamp@PAGEOFF]
__text:00000001052F9AF8 28 E7 00 90 ADRP            X8, #selRef_new@PAGE
__text:00000001052F9AFC 01 91 41 F9 LDR             X1, [X8,#selRef_new@PAGEOFF]
__text:00000001052F9B00 75 C6 43 94 BL              _objc_msgSend
__text:00000001052F9B04 C9 F1 00 B0 ADRP            X9, #qword_1071329A0@PAGE
__text:00000001052F9B08 28 D1 44 F9 LDR             X8, [X9,#qword_1071329A0@PAGEOFF]
__text:00000001052F9B0C 20 D1 04 F9 STR             X0, [X9,#qword_1071329A0@PAGEOFF]
__text:00000001052F9B10 E0 03 08 AA MOV             X0, X8
__text:00000001052F9B14 FD 7B C1 A8 LDP             X29, X30, [SP],#0x10
__text:00000001052F9B18 7B C6 43 14 B               _objc_release
__text:00000001052F9B1C     
__text:00000001052F9B1C
__text:00000001052F9B1C     
__text:00000001052F9B1C FF 03 01 D1 SUB             SP, SP, #0x40 ; '@'
__text:00000001052F9B20 E0 FB 01 A9 STP             X0, X30, [SP,#0x18]
__text:00000001052F9B24 04 00 00 94 BL              sub_1052F9B34
__text:00000001052F9B24            
__text:00000001052F9B28 DC BD 4D CC+DCQ 0xB1164D88CC4DBDDC
__text:00000001052F9B30 BF 34 C1 37 DCB 0xBF, 0x34, 0xC1, 0x37
__text:00000001052F9B34
__text:00000001052F9B34
__text:00000001052F9B34
__text:00000001052F9B34
__text:00000001052F9B34             sub_1052F9B34
__text:00000001052F9B34 40 01 00 10 ADR             X0, dword_1052F9B5C
__text:00000001052F9B38 FE 03 00 AA MOV             X30, X0
__text:00000001052F9B3C FF 03 01 91 ADD             SP, SP, #0x40 ; '@'
__text:00000001052F9B40 C0 03 5F D6 RET

```

RC4 加密后:

```
0000000107C97080  25 F2 BB 8E 4A F9 CA 7C  F7 9A 5F 7D CD 38 67 69  %.............gi
0000000107C97090  2E 4B EF 8D CE E5 F8 58  80 55 D6 0E 5E B3 CB 6A  .K.......U..^...
0000000107C970A0  2A DB 14 AF D7 35 DC A5  ED 02 A8 58 E9 D8 AB 28  *.....ܥ ...X...(
0000000107C970B0  E4 3F F5 08 E9 6A 8D 7F  51 C7 B3 26 4F 8D 0E 42  ........Qǳ &O..B
0000000107C970C0  B2 1C D9 CA 6F 73 C8 06  68 0F 64 30 D1 2B 7E 00  ....os..h.d0..~.
0000000107C970D0  76 A1 25 AB 6A EC D1 FE  67 9A 29 82 A4 44 31 2E  v.%.j...g.)..D1.
0000000107C970E0  14 0B 96 E6 31 81 1F 34  F2 71 AA 86 60 A8 C0 CE  .......4....`...
0000000107C970F0  3D 16 23 83 61 A7 C0 6A  E6 A0 2A A0 7A 6B 1F 42  =.#.a.......zk.B
0000000107C97100  90 30 CC 59 5F 03 7F ED  44 B6 BC 36 B5 0C 97 9D  .0.._......6....
0000000107C97110  82 A0 E4 E6 AB C2 C8 4E  29 F7 55 CA 87 D0 9A 1F  .......N)....К .
0000000107C97120  8E 6D 57 52 00 68 BF 1F  62 D8 DD 67 E4 00 00 00  .mWR.h..b..g....

```

Base64 加密:

```
JfK7jkr5ynz3ml99zThnaS5L743O5fhYgFXWDl6zy2oq2xSv1zXcpe0CqFjp2Kso5D/1COlqjX9Rx7MmT40OQrIc2cpvc8gGaA9kMNErfgB2oSWrauzR/meaKYKkRDEuFAuW5jGBHzTycaqGYKjAzj0WI4Nhp8Bq5qAqoHprH0KQMMxZXwN/7US2vDa1DJedgqDk5qvCyE4p91XKh9CaH45tV1IAaL8fYtjdZ+Q=

```

#### 5.3、获取本地 XID

判断本地是否有存储，如果有优先读取本地，如果本地没有存储就生成一个, 详细逻辑在设备指纹一节中再细说。

```
__text:00000001052E2E58 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001052E2E5C F4 4F BE A9 STP             X20, X19, [SP,#-0x20]!
__text:00000001052E2E60 FD 7B 01 A9 STP             X29, X30, [SP,#0x10]
__text:00000001052E2E64 FD 43 00 91 ADD             X29, SP, #0x10
__text:00000001052E2E68 88 F2 00 90+ADRL            X8, unk_1071328EC
__text:00000001052E2E68 08 B1 23 91
__text:00000001052E2E70 1F FD DF 88 LDAR            WZR, [X8]
__text:00000001052E2E74 E9 03 00 32 MOV             W9, #1
__text:00000001052E2E78 09 FD 9F 88 STLR            W9, [X8]
__text:00000001052E2E7C 48 EA 00 90 ADRP            X8, #classRef_SAKGuardDeviceFingerprint@PAGE
__text:00000001052E2E80 00 7D 45 F9 LDR             X0, [X8,#classRef_SAKGuardDeviceFingerprint@PAGEOFF]
__text:00000001052E2E84 68 E9 00 90 ADRP            X8, #selRef_getFingerprintXID@PAGE
__text:00000001052E2E88 01 25 45 F9 LDR             X1, [X8,#selRef_getFingerprintXID@PAGEOFF]
__text:00000001052E2E8C 92 21 44 94 BL              _objc_msgSend           ; 计取本地
__text:00000001052E2E90 F3 03 00 AA MOV             X19, X0
__text:00000001052E2E94 FD 03 1D AA MOV             X29, X29
__text:00000001052E2E98 A7 21 44 94 BL              _objc_retainAutoreleasedReturnValue
__text:00000001052E2E9C 48 EA 00 90 ADRP            X8, #classRef_NSString@PAGE
__text:00000001052E2EA0 00 A9 41 F9 LDR             X0, [X8,#classRef_NSString@PAGEOFF]
__text:00000001052E2EA4 68 E9 00 90 ADRP            X8, #selRef_isNil_@PAGE
__text:00000001052E2EA8 01 21 45 F9 LDR             X1, [X8,#selRef_isNil_@PAGEOFF]
__text:00000001052E2EAC E2 03 13 AA MOV             X2, X19
__text:00000001052E2EB0 89 21 44 94 BL              _objc_msgSend           ; 判断是否为空
__text:00000001052E2EB4 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052E2EB8 1F 00 00 72 TST             W0, #1
__text:00000001052E2EBC E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052E2EC0 A0 FA FF 54 B.EQ            loc_1052E2E14
__text:00000001052E2EC4 54 C1 00 D0+ADRL            X20, stru_106B0C488
__text:00000001052E2EC4 94 22 12 91
__text:00000001052E2ECC E0 03 14 AA MOV             X0, X20
__text:00000001052E2ED0 90 21 44 94 BL              _objc_retain
__text:00000001052E2ED4 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052E2ED8 00 00 80 D2 MOV             X0, #0
__text:00000001052E2EDC CA FF FF 17 B               loc_1052E2E04

```

如果是第一次运行 APP 或本地没有存储时就本地生成 XID：

```
-[SAKGuardDeviceFingerprint generateLocalXID]

```

本地存储获取到的 xid:

```
mw0bruZSgWId6ew08pp0a3d2Vpfq1fcZfyJrTVmk89oqGNr5754r2zbh6YfpvQ4CijQe+0LfaB+WbyR9njkTQ8iCiFQzqg8rh18j7EntWdk=

```

#### 5.4、获取 dfpid

同样也是如果也是判断本地是否有存储，如果有优先读取本地，如果本地没有存储就生成一个

```
+[SAKGuardDeviceFingerprint getFingerprintID]
 
__text:000000010530E17C E8 03 00 32 MOV             W8, #1
__text:000000010530E180 68 FE 9F 88 STLR            W8, [X19]
__text:000000010530E184 FB E8 00 90 ADRP            X27, #classRef_SAKGuardDeviceFingerprint@PAGE
__text:000000010530E188 60 7F 45 F9 LDR             X0, [X27,#classRef_SAKGuardDeviceFingerprint@PAGEOFF]
__text:000000010530E18C 88 E6 00 90 ADRP            X8, #selRef_sharedInstance@PAGE
__text:000000010530E190 13 D9 45 F9 LDR             X19, [X8,#selRef_sharedInstance@PAGEOFF]
__text:000000010530E194 E1 03 13 AA MOV             X1, X19
__text:000000010530E198 CF 74 43 94 BL              _objc_msgSend
__text:000000010530E19C F4 03 00 AA MOV             X20, X0
__text:000000010530E1A0 FD 03 1D AA MOV             X29, X29
__text:000000010530E1A4 E4 74 43 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010530E1A8 08 E8 00 90 ADRP            X8, #selRef_static_dfpID@PAGE
__text:000000010530E1AC 01 FD 45 F9 LDR             X1, [X8,#selRef_static_dfpID@PAGEOFF]
__text:000000010530E1B0 C9 74 43 94 BL              _objc_msgSend           ; 读取dfpid
__text:000000010530E1B4 F5 03 00 AA MOV             X21, X0
__text:000000010530E1B8 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010530E1BC FD 03 1D AA MOV             X29, X29
__text:000000010530E1C0 E0 03 15 AA MOV             X0, X21
__text:000000010530E1C4 DC 74 43 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010530E1C8 E0 03 14 AA MOV             X0, X20
__text:000000010530E1CC CE 74 43 94 BL              _objc_release
__text:000000010530E1D0 68 E6 00 D0 ADRP            X8, #selRef_isEqual_@PAGE
__text:000000010530E1D4 01 15 43 F9 LDR             X1, [X8,#selRef_isEqual_@PAGEOFF]
__text:000000010530E1D8 F6 03 15 AA MOV             X22, X21
__text:000000010530E1DC 82 ED 00 F0+ADRL            X2, cfstr_R_5           ; "r!\x83Rn2\x8C"
__text:000000010530E1DC 42 80 14 91
__text:000000010530E1E4 E0 03 15 AA MOV             X0, X21
__text:000000010530E1E8 BB 74 43 94 BL              _objc_msgSend           ; 判断是否为空
__text:000000010530E1EC E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010530E1F0 1F 00 00 72 TST             W0, #1
__text:000000010530E1F4 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010530E1F8 41 19 00 54 B.NE            loc_10530E520
__text:000000010530E1FC 68 E6 00 B0 ADRP            X8, #selRef_length@PAGE
__text:000000010530E200 01 C1 47 F9 LDR             X1, [X8,#selRef_length@PAGEOFF]
__text:000000010530E204 F6 03 15 AA MOV             X22, X21
__text:000000010530E208 E0 03 15 AA MOV             X0, X21
__text:000000010530E20C B2 74 43 94 BL              _objc_msgSend
__text:000000010530E210 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010530E214 F6 03 15 AA MOV             X22, X21
__text:000000010530E218 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010530E21C 40 0A 00 B5 CBNZ            X0, loc_10530E364
__text:000000010530E220 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010530E224 A0 00 00 18 LDR             W0, =9

```

如果是第一次运行 APP 或本地没有存储时就本地生成 dfpid：

```
+[SAKGuardLocalIDKeychainStorage generateLocalID]

```

本地存储获取到的 dfpid:

```
dad72f7de813ef8dfd0bbd58f3a775dacf5121ec1a2552173a0e314b

```

#### 5.4、获取系统风险

```
{
    "0": 2,
    "1": ["-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-"],
    "2": ["-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-", "-"],
    "3": "{}"
}

```

压缩 json

```
0000000283F81500  78 9C AB 56 32 50 B2 32  D2 51 32 54 B2 8A 56 D2  x..V2P.2..2T..V.
0000000283F81510  55 D2 A1 00 C7 EA 28 19  51 C5 14 63 25 2B A5 EA  Uҡ ...(.Q..c%+..
0000000283F81520  5A A5 5A 00 EE 45 1A 61  00 00 00 00 00 00 00 00  Z.Z....a........

```

解密解析 pic 获取加密 key(k6)

```
X%rj@KiuU+|xY}?f

```

计算压缩后数据的 crc 值:

```
__text:00000001052CC27C E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001052CC280 E8 03 01 2A MOV             W8, W1
__text:00000001052CC284 09 00 80 12 MOV             W9, #0xFFFFFFFF
__text:00000001052CC288 4A 8C 00 D0+ADRL            X10, unk_1064565CC
__text:00000001052CC288 4A 31 17 91
__text:00000001052CC290 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052CC294
__text:00000001052CC294             loc_1052CC294 
__text:00000001052CC294 0B 14 40 38 LDRB            W11, [X0],#1            ; 取压缩后的数据
__text:00000001052CC298 2C 1D 00 12 AND             W12, W9, #0xFF
__text:00000001052CC29C 8B 01 0B 4A EOR             W11, W12, W11
__text:00000001052CC2A0 4B 59 6B B8 LDR             W11, [X10,W11,UXTW#2]
__text:00000001052CC2A4 69 21 49 4A EOR             W9, W11, W9,LSR#8       ; 计算
__text:00000001052CC2A8 08 05 00 F1 SUBS            X8, X8, #1
__text:00000001052CC2AC E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052CC2B0 21 FF FF 54 B.NE            loc_1052CC294           ; 取压缩后的数据
__text:00000001052CC2B4 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052CC2B8 60 00 00 18 LDR             W0, =0
__text:00000001052CC2BC DC FF FF 17 B               loc_1052CC22C

```

计算后得到:

```
73bbf8c5

```

取 PIC 中获到的值 (k6) 后 8 字节与 crc 值组合做为 AES KEY:

```
000000016B707800  55 2B 7C 78 59 7D 3F 66  00 00 00 00 00 00 00 00  U+|xY}?f
73bbf8c5U+|xY}?f

```

AES 加密压缩后的数据:

```
IV 0102030405060708
KEY 73bbf8c5U+|xY}?f
__text:00000001052EEA08 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001052EEA0C E0 43 00 91 ADD             X0, SP, #0x10
__text:00000001052EEA10 82 1E 80 52 MOV             W2, #0xF4
__text:00000001052EEA14 01 00 80 52 MOV             W1, #0
__text:00000001052EEA18 2B F2 43 94 BL              _memset
__text:00000001052EEA1C E1 03 19 32 MOV             W1, #0x80
__text:00000001052EEA20 E2 43 00 91 ADD             X2, SP, #0x10
__text:00000001052EEA24 E0 03 17 AA MOV             X0, X23
__text:00000001052EEA28 1A 51 FF 97 BL              InitKey_sub_102CBEE90   ; x0:key,x1:长度,x2:初始化后的key
__text:00000001052EEA2C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052EEA30 E2 07 40 F9 LDR             X2, [SP,#8]
__text:00000001052EEA34 E3 43 00 91 ADD             X3, SP, #0x10
__text:00000001052EEA38 E5 03 00 32 MOV             W5, #1
__text:00000001052EEA3C E0 03 15 AA MOV             X0, X21
__text:00000001052EEA40 E1 03 13 AA MOV             X1, X19
__text:00000001052EEA44 E4 03 16 AA MOV             X4, X22
__text:00000001052EEA48 3A 54 FF 97 BL              Aes_Enc_Dec_sub_102CBFB30 ; X0:原始数据,X1:初始化后的key,x2:大小,x3:key,X4:IV,X5:模式:0:解密,1:加密
__text:00000001052EEA4C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052EEA50 E8 07 40 F9 LDR             X8, [SP,#8]
__text:00000001052EEA54 88 02 00 F9 STR             X8, [X20]
__text:00000001052EEA58 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]

```

加密后:

```
0000000280D20750  F7 7A 2E 6A 76 E2 C9 B4  70 F0 62 3B 07 62 91 D7  ....v...p....b..
0000000280D20760  BF 58 DF A6 69 1A D1 0E  0F DE 6A 87 34 00 B8 62  .Xߦ i.......4..b
0000000280D20770  AE DA CA 15 9F 12 62 7F  D2 7B B0 D1 BC FA E9 45  ......b....Ѽ ...

```

base64 加密与 crc 值组合:

```
73bbf8c593ouanbiybRw8GI7B2KR179Y36ZpGtEOD95qhzQAuGKu2soVnxJif9J7sNG8+ulF //前8字节为上面计算的crc值

```

第一次组合签名 json，还差计算 a2 值:

```
{
    "a0": "2.0",
    "a1": "6d1efb41-1bb2-4db1-88ee-b89d21d06e5f",
    "a3": 0,
    "a4": 1635653901,
    "a5": "JfK7jkr5ynz3ml99zThnaS5L743O5fhYgFXWDl6zy2oq2xSv1zXcpe0CqFjp2Kso5D/1COlqjX9Rx7MmT40OQrIc2cpvc8gGaA9kMNErfgB2oSWrauzR/meaKYKkRDEuFAuW5jGBHzTycaqGYKjAzj0WI4Nhp8Bq5qAqoHprH0KQMMxZXwN/7US2vDa1DJedgqDk5qvCyE4p91XKh9CaH45tV1IAaL8fYtjdZ+Q=",
    "a6": 0,
    "a7": "mw0bruZSgWId6ew08pp0a3d2Vpfq1fcZfyJrTVmk89oqGNr5754r2zbh6YfpvQ4CijQe+0LfaB+WbyR9njkTQ8iCiFQzqg8rh18j7EntWdk=",
    "a8": "dad72f7de813ef8dfd0bbd58f3a775dacf5121ec1a2552173a0e314b",
    "a9": "73bbf8c593ouanbiybRw8GI7B2KR179Y36ZpGtEOD95qhzQAuGKu2soVnxJif9J7sNG8+ulF",
    "a10": "",
    "x0": 2
}

```

#### 5.5、计算请求体签名

获取请求体，与上面组合的 json 签名拼接在一起计算签名

```
__text:00000001052DC014 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001052DC018 E0 03 19 AA MOV             X0, X25
__text:00000001052DC01C A1 83 59 F8 LDUR            X1, [X29,#-0x68]
__text:00000001052DC020 E2 03 1C AA MOV             X2, X28
__text:00000001052DC024 64 F1 FF 97 BL              copyData_sub_102CD45B4  ; 拷贝请求体
__text:00000001052DC028 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052DC02C 20 03 1C 8B ADD             X0, X25, X28
__text:00000001052DC030 A1 0B 7A A9 LDP             X1, X2, [X29,#-0x60]
__text:00000001052DC034 60 F1 FF 97 BL              copyData_sub_102CD45B4  ; 拷贝签名值与请求体组合
__text:00000001052DC038 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001052DC03C E8 03 14 AA MOV             X8, X20
__text:00000001052DC040 E0 03 19 AA MOV             X0, X25
__text:00000001052DC044 A1 83 58 F8 LDUR            X1, [X29,#-0x78]
__text:00000001052DC048 E2 03 16 AA MOV             X2, X22
__text:00000001052DC04C AC DC 01 94 BL              GenBodyMtsig_loc_1030332FC ; x0:原始数据,x1:大小
__text:00000001052DC050 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]

```

组合后的请求体

```
00000001080B5600  50 4F 53 54 20 2F 61 70  70 75 70 64 61 74 65 2F  POST /appupdate/
00000001080B5610  61 6C 69 74 61 2F 63 68  65 63 6B 55 70 64 61 74  alita/checkUpdat
00000001080B5620  65 20 5F 5F 72 65 71 54  72 61 63 65 49 44 3D 46  e __reqTraceID=F
00000001080B5630  38 33 31 30 34 37 32 2D  31 38 37 45 2D 34 34 44  8310472-187E-44D
00000001080B5640  34 2D 39 46 37 33 2D 33  31 36 32 34 30 46 30 39  4-9F73-316240F09
00000001080B5650  38 41 30 26 63 69 3D 32  26 6C 61 6E 67 75 61 67  8A0&ci=2&languag
00000001080B5660  65 3D 7A 68 5F 43 4E 26  75 74 6D 5F 63 61 6D 70  e=zh_CN&utm_camp
00000001080B5670  61 69 67 6E 3D 41 69 6D  61 69 63 61 69 5F 63 42  aign=Aimaicai_cB
00000001080B5680  69 6D 61 69 63 61 69 5F  63 48 30 26 75 74 6D 5F  imaicai_cH0&utm_
00000001080B5690  63 6F 6E 74 65 6E 74 3D  30 30 30 30 30 30 30 30  content=00000000
00000001080B56A0  30 30 30 30 30 32 31 38  38 37 45 34 41 39 46 34  0000021887E4A9F4
00000001080B56B0  39 34 41 39 41 41 38 32  45 39 43 38 37 38 32 39  94A9AA82E9C87829
00000001080B56C0  45 43 46 46 37 41 31 36  33 33 37 39 32 30 30 38  ECFF7A1633792008
00000001080B56D0  32 37 37 36 38 32 32 38  26 75 74 6D 5F 6D 65 64  27768228&utm_med
00000001080B56E0  69 75 6D 3D 69 70 68 6F  6E 65 26 75 74 6D 5F 73  ium=iphone&utm_s
00000001080B56F0  6F 75 72 63 65 3D 41 70  70 53 74 6F 72 65 26 75  ource=AppStore&u
00000001080B5700  74 6D 5F 74 65 72 6D 3D  35 2E 32 35 2E 30 26 75  tm_term=5.25.0&u
00000001080B5710  75 69 64 3D 30 30 30 30  30 30 30 30 30 30 30 30  uid=000000000000
00000001080B5720  30 32 31 38 38 37 45 34  41 39 46 34 39 34 41 39  021887E4A9F494A9
00000001080B5730  41 41 38 32 45 39 43 38  37 38 32 39 45 43 46 46  AA82E9C87829ECFF
00000001080B5740  37 41 31 36 33 33 37 39  32 30 30 38 32 37 37 36  7A16337920082776
00000001080B5750  38 32 32 38 26 76 65 72  73 69 6F 6E 5F 6E 61 6D  8228&version_nam
00000001080B5760  65 3D 35 2E 32 35 2E 30  7B 22 63 68 61 6E 6E 65  e=5.25.0{"channe
00000001080B5770  6C 22 3A 22 41 70 70 53  74 6F 72 65 22 2C 22 61  l":"AppStore","a
00000001080B5780  70 70 56 65 72 73 69 6F  6E 22 3A 22 35 30 30 32  ppVersion":"5002
00000001080B5790  35 30 30 30 30 22 2C 22  61 70 70 22 3A 22 69 6D  50000","app":"im
00000001080B57A0  61 69 63 61 69 22 2C 22  62 75 6E 64 6C 65 73 22  aicai","bundles"
00000001080B57B0  3A 5B 5D 2C 22 66 69 6E  67 65 72 70 72 69 6E 74  :[],"fingerprint
00000001080B57C0  22 3A 22 4D 65 70 68 69  73 74 6F 22 2C 22 73 64  ":"Mephisto","sd
00000001080B57D0  6B 56 65 72 73 69 6F 6E  22 3A 22 31 2E 30 2E 30  kVersion":"1.0.0
00000001080B57E0  22 2C 22 70 6C 61 74 66  6F 72 6D 22 3A 22 69 4F  ","platform":"iO
00000001080B57F0  53 22 7D 7B 22 61 30 22  3A 22 32 2E 30 22 2C 22  S"}{"a0":"2.0","
00000001080B5800  61 31 22 3A 22 36 64 31  65 66 62 34 31 2D 31 62  a1":"6d1efb41-1b
00000001080B5810  62 32 2D 34 64 62 31 2D  38 38 65 65 2D 62 38 39  b2-4db1-88ee-b89
00000001080B5820  64 32 31 64 30 36 65 35  66 22 2C 22 61 33 22 3A  d21d06e5f","a3":
00000001080B5830  30 2C 22 61 34 22 3A 31  36 33 35 36 35 33 39 30  0,"a4":163565390
00000001080B5840  31 2C 22 61 35 22 3A 22  4A 66 4B 37 6A 6B 72 35  1,"a5":"JfK7jkr5
00000001080B5850  79 6E 7A 33 6D 6C 39 39  7A 54 68 6E 61 53 35 4C  ynz3ml99zThnaS5L
00000001080B5860  37 34 33 4F 35 66 68 59  67 46 58 57 44 6C 36 7A  743O5fhYgFXWDl6z
00000001080B5870  79 32 6F 71 32 78 53 76  31 7A 58 63 70 65 30 43  y2oq2xSv1zXcpe0C
00000001080B5880  71 46 6A 70 32 4B 73 6F  35 44 2F 31 43 4F 6C 71  qFjp2Kso5D/1COlq
00000001080B5890  6A 58 39 52 78 37 4D 6D  54 34 30 4F 51 72 49 63  jX9Rx7MmT40OQrIc
00000001080B58A0  32 63 70 76 63 38 67 47  61 41 39 6B 4D 4E 45 72  2cpvc8gGaA9kMNEr
00000001080B58B0  66 67 42 32 6F 53 57 72  61 75 7A 52 2F 6D 65 61  fgB2oSWrauzR/mea
00000001080B58C0  4B 59 4B 6B 52 44 45 75  46 41 75 57 35 6A 47 42  KYKkRDEuFAuW5jGB
00000001080B58D0  48 7A 54 79 63 61 71 47  59 4B 6A 41 7A 6A 30 57  HzTycaqGYKjAzj0W
00000001080B58E0  49 34 4E 68 70 38 42 71  35 71 41 71 6F 48 70 72  I4Nhp8Bq5qAqoHpr
00000001080B58F0  48 30 4B 51 4D 4D 78 5A  58 77 4E 2F 37 55 53 32  H0KQMMxZXwN/7US2
00000001080B5900  76 44 61 31 44 4A 65 64  67 71 44 6B 35 71 76 43  vDa1DJedgqDk5qvC
00000001080B5910  79 45 34 70 39 31 58 4B  68 39 43 61 48 34 35 74  yE4p91XKh9CaH45t
00000001080B5920  56 31 49 41 61 4C 38 66  59 74 6A 64 5A 2B 51 3D  V1IAaL8fYtjdZ+Q=
00000001080B5930  22 2C 22 61 36 22 3A 30  2C 22 61 37 22 3A 22 6D  ","a6":0,"a7":"m
00000001080B5940  77 30 62 72 75 5A 53 67  57 49 64 36 65 77 30 38  w0bruZSgWId6ew08
00000001080B5950  70 70 30 61 33 64 32 56  70 66 71 31 66 63 5A 66  pp0a3d2Vpfq1fcZf
00000001080B5960  79 4A 72 54 56 6D 6B 38  39 6F 71 47 4E 72 35 37  yJrTVmk89oqGNr57
00000001080B5970  35 34 72 32 7A 62 68 36  59 66 70 76 51 34 43 69  54r2zbh6YfpvQ4Ci
00000001080B5980  6A 51 65 2B 30 4C 66 61  42 2B 57 62 79 52 39 6E  jQe+0LfaB+WbyR9n
00000001080B5990  6A 6B 54 51 38 69 43 69  46 51 7A 71 67 38 72 68  jkTQ8iCiFQzqg8rh
00000001080B59A0  31 38 6A 37 45 6E 74 57  64 6B 3D 22 2C 22 61 38  18j7EntWdk=","a8
00000001080B59B0  22 3A 22 64 61 64 37 32  66 37 64 65 38 31 33 65  ":"dad72f7de813e
00000001080B59C0  66 38 64 66 64 30 62 62  64 35 38 66 33 61 37 37  f8dfd0bbd58f3a77
00000001080B59D0  35 64 61 63 66 35 31 32  31 65 63 31 61 32 35 35  5dacf5121ec1a255
00000001080B59E0  32 31 37 33 61 30 65 33  31 34 62 22 2C 22 61 39  2173a0e314b","a9
00000001080B59F0  22 3A 22 37 33 62 62 66  38 63 35 39 33 6F 75 61  ":"73bbf8c593oua
00000001080B5A00  6E 62 69 79 62 52 77 38  47 49 37 42 32 4B 52 31  nbiybRw8GI7B2KR1
00000001080B5A10  37 39 59 33 36 5A 70 47  74 45 4F 44 39 35 71 68  79Y36ZpGtEOD95qh
00000001080B5A20  7A 51 41 75 47 4B 75 32  73 6F 56 6E 78 4A 69 66  zQAuGKu2soVnxJif
00000001080B5A30  39 4A 37 73 4E 47 38 2B  75 6C 46 22 2C 22 61 31  9J7sNG8+ulF","a1
00000001080B5A40  30 22 3A 22 22 2C 22 78  30 22 3A 32 7D 20 31 37  0":"","x0":2}

```

解密 PIC 获取 a0 值

```
sdk9xWZTg5V9nKAxVFB5mB1ipZIJGmYSysreJ1f/rlvXJ7Ydxd3hJRdWb4QdZKr/

```

解密 a0

```
key appkey:6d1efb41-1bb2-4db1-88ee-b89d21d06e5f
 
__text:000000010535363C             DecPic_a0_loc_1051E763C
__text:000000010535363C 09 09 DC 9A UDIV            X9, X8, X28             ; 解密pic a0
__text:0000000105353640 29 A1 1C 9B MSUB            X9, X9, X28, X8
__text:0000000105353644 6A 6A 69 38 LDRB            W10, [X19,X9]           ; appkey 6d1efb41-1bb2-4db1-88ee-b89d21d06e5f
__text:0000000105353648 AB 6A 68 38 LDRB            W11, [X21,X8]           ; PIC a0
__text:000000010535364C 6A 01 0A 4A EOR             W10, W11, W10
__text:0000000105353650 0A 6B 29 38 STRB            W10, [X24,X9]
__text:0000000105353654 08 05 00 91 ADD             X8, X8, #1
__text:0000000105353658 5F 03 08 EB CMP             X26, X8                 ; 判断是否结束
__text:000000010535365C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000105353660 E1 FE FF 54 B.NE            DecPic_a0_loc_1051E763C ; 解密pic a0
__text:0000000105353664 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000105353668 A0 00 00 18 LDR             W0, =0x17

```

解密后 a0

```
0000000280A08750  7C 55 57 4A 14 0E 42 69  67 06 3B 06 4A 49 07 0C  |UWJ..Big.;.JI..
0000000280A08760  28 63 49 6F 5A 51 34 49  38 73 4B 4B 75 5C 3D 63  (cIoZQ4I8sKKu\=c
0000000280A08770  4F 16 47 03

```

再次解密 a0 分为两组

```
000000016D10C530                      20 09  0B 16 48 52 1E 35 3B 5A  66666. ...HR.5;Z
000000016D10C540  67 5A 16 15 5B 50 74 3F  15 33 06 0D 68 15 64 2F  gZ..[Pt?.3..h.d/
000000016D10C550  17 17 29 00 61 3F 13 4A  1B 5F 5C 5C 5C 5C 5C 5C  ..).a?.J._\\\\\\
000000016D10C560  5C 5C 5C 5C 5C 5C 5C 5C  5C 5C 5C 5C 5C 5C 5C 5C  \\\\\\\\\\\\\\\\
000000016D10C570  5C 5C 5C 5C 5C 5C
 
0000000106B78A00  4A 63 61 7C 22 38 74 5F  51 30 0D 30 7C 7F 31 3A  Jca|"8t_Q0.0|.1:
0000000106B78A10  1E 55 7F 59 6C 67 02 7F  0E 45 7D 7D 43 6A 0B 55  .U.Ylg...E}}Cj.U
0000000106B78A20  79 20 71 35 36 36 36 36  36 36 36 36 36 36 36 36  y q5666666666666
0000000106B78A30  36 36 36 36 36 36 36 36  36 36 36 36 36 36 36 36  6666666666666666

```

将第二次解密后的 a0 值其中一组与请求体组合

```
00000001089E5A00  4A 63 61 7C 22 38 74 5F  51 30 0D 30 7C 7F 31 3A  Jca|"8t_Q0.0|.1:
00000001089E5A10  1E 55 7F 59 6C 67 02 7F  0E 45 7D 7D 43 6A 0B 55  .U.Ylg...E}}Cj.U
00000001089E5A20  79 20 71 35 36 36 36 36  36 36 36 36 36 36 36 36  y q5666666666666
00000001089E5A30  36 36 36 36 36 36 36 36  36 36 36 36 36 36 36 36  6666666666666666
00000001089E5A40  50 4F 53 54 20 2F 61 70  70 75 70 64 61 74 65 2F  POST /appupdate/
00000001089E5A50  61 6C 69 74 61 2F 63 68  65 63 6B 55 70 64 61 74  alita/checkUpdat
00000001089E5A60  65 20 5F 5F 72 65 71 54  72 61 63 65 49 44 3D 46  e __reqTraceID=F
00000001089E5A70  38 33 31 30 34 37 32 2D  31 38 37 45 2D 34 34 44  8310472-187E-44D
00000001089E5A80  34 2D 39 46 37 33 2D 33  31 36 32 34 30 46 30 39  4-9F73-316240F09
00000001089E5A90  38 41 30 26 63 69 3D 32  26 6C 61 6E 67 75 61 67  8A0&ci=2&languag
00000001089E5AA0  65 3D 7A 68 5F 43 4E 26  75 74 6D 5F 63 61 6D 70  e=zh_CN&utm_camp
00000001089E5AB0  61 69 67 6E 3D 41 69 6D  61 69 63 61 69 5F 63 42  aign=Aimaicai_cB
00000001089E5AC0  69 6D 61 69 63 61 69 5F  63 48 30 26 75 74 6D 5F  imaicai_cH0&utm_
00000001089E5AD0  63 6F 6E 74 65 6E 74 3D  30 30 30 30 30 30 30 30  content=00000000
00000001089E5AE0  30 30 30 30 30 32 31 38  38 37 45 34 41 39 46 34  0000021887E4A9F4

```

计算 hmac 值

```
__text:00000001052EEEDC             hmac_256_sub_105182EDC  
__text:00000001052EEEDC                              
__text:00000001052EEEDC
__text:00000001052EEEDC             __src= -0x81
__text:00000001052EEEDC             var_80= -0x80
__text:00000001052EEEDC             var_78= -0x78
__text:00000001052EEEDC             var_70= -0x70
__text:00000001052EEEDC             var_68= -0x68
__text:00000001052EEEDC             var_60= -0x60
__text:00000001052EEEDC             var_18= -0x18
__text:00000001052EEEDC             var_10= -0x10
__text:00000001052EEEDC             var_s0=  0
__text:00000001052EEEDC
__text:00000001052EEEDC FF 83 02 D1 SUB             SP, SP, #0xA0
__text:00000001052EEEE0 F4 4F 08 A9 STP             X20, X19, [SP,#0x90+var_10]
__text:00000001052EEEE4 FD 7B 09 A9 STP             X29, X30, [SP,#0x90+var_s0]
__text:00000001052EEEE8 FD 43 02 91 ADD             X29, SP, #0x90
__text:00000001052EEEEC F3 03 02 AA MOV             X19, X2
__text:00000001052EEEF0 E8 03 01 AA MOV             X8, X1
__text:00000001052EEEF4 E9 03 00 AA MOV             X9, X0
__text:00000001052EEEF8 CA BC 00 B0 ADRP            X10, #___stack_chk_guard_ptr@PAGE
__text:00000001052EEEFC 4A FD 43 F9 LDR             X10, [X10,#___stack_chk_guard_ptr@PAGEOFF]
__text:00000001052EEF00 4A 01 40 F9 LDR             X10, [X10]
__text:00000001052EEF04 AA 83 1E F8 STUR            X10, [X29,#var_18]
__text:00000001052EEF08 8A 8B 00 90 ADRP            X10, #qword_10645E478@PAGE
__text:00000001052EEF0C 40 3D 42 FD LDR             D0, [X10,#qword_10645E478@PAGEOFF]
__text:00000001052EEF10 E0 0F 00 FD STR             D0, [SP,#0x90+var_78]
__text:00000001052EEF14 1F 20 03 D5 NOP
__text:00000001052EEF18 40 41 42 FD LDR             D0, [X10,#qword_10645E480@PAGEOFF]
__text:00000001052EEF1C E0 13 00 FD STR             D0, [SP,#0x90+var_70]
__text:00000001052EEF20 FF 33 00 B9 STR             WZR, [SP,#0x90+var_60]
__text:00000001052EEF24 1F 20 03 D5 NOP
__text:00000001052EEF28 40 5D 42 FD LDR             D0, [X10,#qword_10645E4B8@PAGEOFF]
__text:00000001052EEF2C E0 17 00 FD STR             D0, [SP,#0x90+var_68]
__text:00000001052EEF30 E0 63 00 91 ADD             X0, SP, #0x90+var_78    ; int
__text:00000001052EEF34 E1 03 09 AA MOV             X1, X9                  ; __src
__text:00000001052EEF38 E2 03 08 AA MOV             X2, X8
__text:00000001052EEF3C 39 00 00 94 BL              sub_1052EF020
__text:00000001052EEF40 08 00 80 52 MOV             W8, #0
__text:00000001052EEF44 09 00 80 D2 MOV             X9, #0
__text:00000001052EEF48 EA 43 00 91 ADD             X10, SP, #0x90+var_80
__text:00000001052EEF4C
__text:00000001052EEF4C             loc_1052EEF4C    
__text:00000001052EEF4C 3F 11 00 F1 CMP             X9, #4
__text:00000001052EEF50 EB 27 9F 1A CSET            W11, CC
__text:00000001052EEF54 EC 63 00 91 ADD             X12, SP, #0x90+var_78
__text:00000001052EEF58 6C 01 7E B3 BFI             X12, X11, #2, #1
__text:00000001052EEF5C 8B 15 40 B9 LDR             W11, [X12,#0x14]
__text:00000001052EEF60 EC 03 28 2A MVN             W12, W8
__text:00000001052EEF64 8C 05 1D 12 AND             W12, W12, #0x18
__text:00000001052EEF68 6B 25 CC 1A LSR             W11, W11, W12
__text:00000001052EEF6C 4B 69 29 38 STRB            W11, [X10,X9]
__text:00000001052EEF70 29 05 00 91 ADD             X9, X9, #1
__text:00000001052EEF74 08 21 00 11 ADD             W8, W8, #8
__text:00000001052EEF78 3F 21 00 F1 CMP             X9, #8
__text:00000001052EEF7C 81 FE FF 54 B.NE            loc_1052EEF4C
__text:00000001052EEF80 E8 03 19 32 MOV             W8, #0x80
__text:00000001052EEF84 E8 3F 00 39 STRB            W8, [SP,#0x90+__src]
__text:00000001052EEF88 F4 63 00 91 ADD             X20, SP, #0x90+var_78
__text:00000001052EEF8C 02 00 00 14 B               loc_1052EEF94
__text:00000001052EEF90        
__text:00000001052EEF90
__text:00000001052EEF90             loc_1052EEF90  
__text:00000001052EEF90 FF 3F 00 39 STRB            WZR, [SP,#0x90+__src]
__text:00000001052EEF94
__text:00000001052EEF94             loc_1052EEF94  
__text:00000001052EEF94 E0 63 00 91 ADD             X0, SP, #0x90+var_78    ; int
__text:00000001052EEF98 E1 3F 00 91 ADD             X1, SP, #0x90+__src     ; __src
__text:00000001052EEF9C E2 03 00 32 MOV             W2, #1
__text:00000001052EEFA0 20 00 00 94 BL              sub_1052EF020
__text:00000001052EEFA4 E8 2F 40 B9 LDR             W8, [SP,#0x90+var_68+4]
__text:00000001052EEFA8 08 15 1D 12 AND             W8, W8, #0x1F8
__text:00000001052EEFAC 1F 01 07 71 CMP             W8, #0x1C0
__text:00000001052EEFB0 01 FF FF 54 B.NE            loc_1052EEF90
__text:00000001052EEFB4 E0 63 00 91 ADD             X0, SP, #0x90+var_78    ; int
__text:00000001052EEFB8 E1 43 00 91 ADD             X1, SP, #0x90+var_80    ; __src
__text:00000001052EEFBC E2 03 1D 32 MOV             W2, #8
__text:00000001052EEFC0 18 00 00 94 BL              sub_1052EF020
__text:00000001052EEFC4 08 00 80 52 MOV             W8, #0
__text:00000001052EEFC8 09 00 80 D2 MOV             X9, #0
__text:00000001052EEFCC
__text:00000001052EEFCC             loc_1052EEFCC   
__text:00000001052EEFCC 2A 7D 42 D3 UBFX            X10, X9, #2, #0x1E
__text:00000001052EEFD0 8A 7A 6A B8 LDR             W10, [X20,X10,LSL#2]
__text:00000001052EEFD4 EB 03 28 2A MVN             W11, W8
__text:00000001052EEFD8 6B 05 1D 12 AND             W11, W11, #0x18
__text:00000001052EEFDC 4A 25 CB 1A LSR             W10, W10, W11
__text:00000001052EEFE0 6A 6A 29 38 STRB            W10, [X19,X9]
__text:00000001052EEFE4 29 05 00 91 ADD             X9, X9, #1
__text:00000001052EEFE8 08 21 00 11 ADD             W8, W8, #8
__text:00000001052EEFEC 3F 51 00 F1 CMP             X9, #0x14
__text:00000001052EEFF0 E1 FE FF 54 B.NE            loc_1052EEFCC
__text:00000001052EEFF4 A8 83 5E F8 LDUR            X8, [X29,#var_18]
__text:00000001052EEFF8 C9 BC 00 B0 ADRP            X9, #___stack_chk_guard_ptr@PAGE
__text:00000001052EEFFC 29 FD 43 F9 LDR             X9, [X9,#___stack_chk_guard_ptr@PAGEOFF]
__text:00000001052EF000 29 01 40 F9 LDR             X9, [X9]
__text:00000001052EF004 3F 01 08 EB CMP             X9, X8
__text:00000001052EF008 A1 00 00 54 B.NE            loc_1052EF01C
__text:00000001052EF00C FD 7B 49 A9 LDP             X29, X30, [SP,#0x90+var_s0]
__text:00000001052EF010 F4 4F 48 A9 LDP             X20, X19, [SP,#0x90+var_10]
__text:00000001052EF014 FF 83 02 91 ADD             SP, SP, #0xA0
__text:00000001052EF018 C0 03 5F D6 RET

```

计算后的值

```
000000016B7078B0  D8 49 F8 46 FA 3A 4C 93  AC 68 76 4E 15 11 6C E2
000000016B7078C0  A5 81 4B 1F

```

计算后 hmac 值与解密后的 a0 其中一组组合

```
000000016B707850                                    20 09 0B 16 48
000000016B707860  52 1E 35 3B 5A 67 5A 16  15 5B 50 74 3F 15 33 06
000000016B707870  0D 68 15 64 2F 17 17 29  00 61 3F 13 4A 1B 5F 5C
000000016B707880  5C 5C 5C 5C 5C 5C 5C 5C  5C 5C 5C 5C 5C 5C 5C 5C
000000016B707890  5C 5C 5C 5C 5C 5C 5C 5C  5C 5C 5C D8 49 F8 46 FA
000000016B7078A0  3A 4C 93 AC 68 76 4E 15  11 6C E2 A5 81 4B 1F

```

再次计算 hmac 值

```
000000016B707A50  58 26 1C D2 C2 35 BC D4  CE 83 F3 AF E0 BA 76 8C
000000016B707A60  C5 90 AF 5C

```

加密计算的 hmac 值得到最终的签名值

```
__text:0000000105353BC4             loc_105353BC4
__text:0000000105353BC4 81 02 80 52 MOV             W1, #0x14
__text:0000000105353BC8 E0 03 19 AA MOV             X0, X25
__text:0000000105353BCC E2 03 1A AA MOV             X2, X26
__text:0000000105353BD0 2D 1E 00 94 BL              EncHmac_sha_loc_10303B484 ; x0:计算后的hmac,x1:大小，x0：返回
__text:0000000105353BD4 F9 03 00 AA MOV             X25, X0
__text:0000000105353BD8 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000105353BDC
__text:0000000105353BDC             loc_105353BDC
__text:0000000105353BDC E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:0000000105353BE0 41 03 40 F9 LDR             X1, [X26]
__text:0000000105353BE4 E8 03 16 AA MOV             X8, X22
__text:0000000105353BE8 E0 03 19 AA MOV             X0, X25
__text:0000000105353BEC 66 0C 00 94 BL              Hex2String_loc_105296D84 ; hmac转换成字符串
__text:0000000105353BF0 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000105353BF4 C8 0A 40 F9 LDR             X8, [X22,#0x10]
__text:0000000105353BF8 88 0A 00 F9 STR             X8, [X20,#0x10]
__text:0000000105353BFC C0 02 C0 3D LDR             Q0, [X22]
__text:0000000105353C00 80 02 80 3D STR             Q0, [X20]
__text:0000000105353C04 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000105353C08 B9 DA FF B4 CBZ             X25, loc_10535375C
__text:0000000105353C0C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:0000000105353C10 A0 00 00 18 LDR             W0, =1
__text:0000000105353C14 51 FE FF 17 B               loc_105353558

```

加密后

```
0000000280190580  7F 26 8F D8 7F 5D 01 F3  2D D5 C7 E0 86 84 87 8E

```

转换成字符串

```
7f268fd87f5d01f32dd5c7e08684878e

```

组合成最终的签名

```
{
    "a0": "2.0",
    "a1": "6d1efb41-1bb2-4db1-88ee-b89d21d06e5f",
    "a3": 0,
    "a4": 1635653901,
    "a5": "JfK7jkr5ynz3ml99zThnaS5L743O5fhYgFXWDl6zy2oq2xSv1zXcpe0CqFjp2Kso5D/1COlqjX9Rx7MmT40OQrIc2cpvc8gGaA9kMNErfgB2oSWrauzR/meaKYKkRDEuFAuW5jGBHzTycaqGYKjAzj0WI4Nhp8Bq5qAqoHprH0KQMMxZXwN/7US2vDa1DJedgqDk5qvCyE4p91XKh9CaH45tV1IAaL8fYtjdZ+Q=",
    "a6": 0,
    "a7": "mw0bruZSgWId6ew08pp0a3d2Vpfq1fcZfyJrTVmk89oqGNr5754r2zbh6YfpvQ4CijQe+0LfaB+WbyR9njkTQ8iCiFQzqg8rh18j7EntWdk=",
    "a8": "dad72f7de813ef8dfd0bbd58f3a775dacf5121ec1a2552173a0e314b",
    "a9": "73bbf8c593ouanbiybRw8GI7B2KR179Y36ZpGtEOD95qhzQAuGKu2soVnxJif9J7sNG8+ulF",
    "a10": "",
    "x0": 2,
    "a2": "7f268fd87f5d01f32dd5c7e08684878e"
}

```

整个请求体的签名结束，然后发起网络请求。

### [](#六、设备指纹分析)六、设备指纹分析

#### 6.1、请求服务器设备指纹

应用启动后会生成两个 ID，一个是 XID，一个是 DFPID，如图 6-1、6-2 所示:  
![](https://bbs.pediy.com/upload/attach/202111/536985_93ZNYXTJK7KQQFK.png)  
图 6-1  
![](https://bbs.pediy.com/upload/attach/202111/536985_67HGB89PCQGXAT6.png)  
图 6-2

#### 6.2、XID 生成

获取设备信息:

```
__text:00000001010F3234             getmDeviceInfo_loc_104F0F234
__text:00000001010F3234
__text:00000001010F3234 FF 43 01 D1 SUB             SP, SP, #0x50 ; 'P'     ; 获取设备信息,x0:编号，根据编号走到对应的获取方法中
__text:00000001010F3238 E0 7B 02 A9 STP             X0, X30, [SP,#0x20]
__text:00000001010F323C 03 00 00 94 BL              sub_1010F3248
__text:00000001010F323C 
__text:00000001010F3240 2A FC 35 E0+DCQ 0x27CDE47EE035FC2A
__text:00000001010F3248
__text:00000001010F3248 
__text:00000001010F3248
__text:00000001010F3248
__text:00000001010F3248             sub_1010F3248
__text:00000001010F3248 00 01 00 10 ADR             X0, loc_1010F3268
__text:00000001010F324C FE 03 00 AA MOV             X30, X0
__text:00000001010F3250 FF 43 01 91 ADD             SP, SP, #0x50 ; 'P'
__text:00000001010F3254 C0 03 5F D6 RET
__text:00000001010F3254             ; End of function sub_1010F3248
__text:00000001010F3254
__text:00000001010F3254 
__text:00000001010F3258 47 86 7E 27+DCQ 0x183E35D4277E8647, 0x78B503779BA7D3FF
__text:00000001010F3268 
__text:00000001010F3268
__text:00000001010F3268             loc_1010F3268                           ; DATA XREF: sub_1010F3248↑o
__text:00000001010F3268 E0 7B 7D A9 LDP             X0, X30, [SP,#-0x30]
__text:00000001010F326C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010F3270 80 01 80 D2 MOV             X0, #0xC
__text:00000001010F3274 08 00 00 14 B               loc_1010F3294
 
 
//循环获取设备信息
__text:00000001010F4EC8 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]    ; 获取m设备信息
__text:00000001010F4ECC E9 2B 00 10 ADR             X9, unk_1010F5448
__text:00000001010F4ED0 1F 20 03 D5 NOP
__text:00000001010F4ED4 28 79 A8 B8 LDRSW           X8, [X9,X8,LSL#2]
__text:00000001010F4ED8 08 01 09 8B ADD             X8, X8, X9
__text:00000001010F4EDC E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010F4EE0 00 01 1F D6 BR              X8                      ; 走到对应的获取信息方法

```

将每一个获取到的信息组合单个 json 值, 格式如下

```
{"value":"E68684F0-7573-4EBC-99BD-A03D58888888","code":1} //获取的IDFA

```

获取本地 XID，如果是第一次或本地没有存储就本地生成一个：

```
-[SAKGuardDeviceFingerprint generateLocalXID]
+[SAKGuardLocalIDKeychainStorage localID]
+[SAKGuardLocalIDKeychainStorage generateLocalID]
 
__text:000000010110C9D8 E8 03 00 32 MOV             W8, #1
__text:000000010110C9DC 68 FE 9F 88 STLR            W8, [X19]
__text:000000010110C9E0 F4 83 00 D1 SUB             X20, SP, #0x20 ; ' '
__text:000000010110C9E4 9F 02 00 91 MOV             SP, X20
__text:000000010110C9E8 F7 83 00 D1 SUB             X23, SP, #0x20 ; ' '
__text:000000010110C9EC FF 02 00 91 MOV             SP, X23
__text:000000010110C9F0 56 EC 00 D0+ADRL            X22, cfstr_M_7          ; "m"
__text:000000010110C9F0 D6 82 30 91
__text:000000010110C9F8 E0 03 16 AA MOV             X0, X22
__text:000000010110C9FC C5 BA 42 94 BL              _objc_retain
__text:000000010110CA00 68 E7 00 D0 ADRP            X8, #classRef_NSUUID@PAGE
__text:000000010110CA04 00 ED 43 F9 LDR             X0, [X8,#classRef_NSUUID@PAGEOFF]
__text:000000010110CA08 08 E5 00 F0 ADRP            X8, #selRef_UUID@PAGE
__text:000000010110CA0C 01 11 43 F9 LDR             X1, [X8,#selRef_UUID@PAGEOFF]
__text:000000010110CA10 B1 BA 42 94 BL              _objc_msgSend
__text:000000010110CA14 F5 03 00 AA MOV             X21, X0
__text:000000010110CA18 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CA1C FD 03 1D AA MOV             X29, X29
__text:000000010110CA20 E0 03 15 AA MOV             X0, X21
__text:000000010110CA24 C4 BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CA28 08 E5 00 F0 ADRP            X8, #selRef_UUIDString@PAGE
__text:000000010110CA2C 01 C9 43 F9 LDR             X1, [X8,#selRef_UUIDString@PAGEOFF]
__text:000000010110CA30 A9 BA 42 94 BL              _objc_msgSend           ; 生成UUID字符串
__text:000000010110CA34 F9 03 00 AA MOV             X25, X0
__text:000000010110CA38 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CA3C FD 03 1D AA MOV             X29, X29
__text:000000010110CA40 E0 03 19 AA MOV             X0, X25
__text:000000010110CA44 BC BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CA48 08 E5 00 B0 ADRP            X8, #selRef_stringByReplacingOccurrencesOfString_withString_@PAGE
__text:000000010110CA4C 01 D1 41 F9 LDR             X1, [X8,#selRef_stringByReplacingOccurrencesOfString_withString_@PAGEOFF]
__text:000000010110CA50 42 EC 00 D0+ADRL            X2, cfstr_K_7           ; "k"
__text:000000010110CA50 42 00 31 91
__text:000000010110CA58 98 BE 00 90+ADRL            X24, stru_1028DC488
__text:000000010110CA58 18 23 12 91
__text:000000010110CA60 E3 03 18 AA MOV             X3, X24
__text:000000010110CA64 9C BA 42 94 BL              _objc_msgSend           ; 替换掉UUID"-"
__text:000000010110CA68 F3 03 00 AA MOV             X19, X0
__text:000000010110CA6C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CA70 FD 03 1D AA MOV             X29, X29
__text:000000010110CA74 E0 03 13 AA MOV             X0, X19
__text:000000010110CA78 AF BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CA7C E0 03 19 AA MOV             X0, X25
__text:000000010110CA80 A1 BA 42 94 BL              _objc_release
__text:000000010110CA84 E0 03 15 AA MOV             X0, X21
__text:000000010110CA88 9F BA 42 94 BL              _objc_release
__text:000000010110CA8C 68 E7 00 D0 ADRP            X8, #classRef_NSDate@PAGE
__text:000000010110CA90 00 B9 42 F9 LDR             X0, [X8,#classRef_NSDate@PAGEOFF]
__text:000000010110CA94 08 E5 00 B0 ADRP            X8, #selRef_date@PAGE
__text:000000010110CA98 01 AD 41 F9 LDR             X1, [X8,#selRef_date@PAGEOFF]
__text:000000010110CA9C 8E BA 42 94 BL              _objc_msgSend
__text:000000010110CAA0 F5 03 00 AA MOV             X21, X0
__text:000000010110CAA4 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CAA8 FD 03 1D AA MOV             X29, X29
__text:000000010110CAAC E0 03 15 AA MOV             X0, X21
__text:000000010110CAB0 A1 BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CAB4 08 E5 00 B0 ADRP            X8, #selRef_timeIntervalSince1970@PAGE
__text:000000010110CAB8 01 B1 41 F9 LDR             X1, [X8,#selRef_timeIntervalSince1970@PAGEOFF]
__text:000000010110CABC 86 BA 42 94 BL              _objc_msgSend           ; 获取时间
__text:000000010110CAC0 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CAC4 A8 85 00 90 ADRP            X8, #off_1021C05D8@PAGE
__text:000000010110CAC8 01 ED 42 FD LDR             D1, [X8,#off_1021C05D8@PAGEOFF]
__text:000000010110CACC 00 08 61 1E FMUL            D0, D0, D1
__text:000000010110CAD0 19 00 78 9E FCVTZS          X25, D0
__text:000000010110CAD4 E0 03 15 AA MOV             X0, X21
__text:000000010110CAD8 8B BA 42 94 BL              _objc_release
__text:000000010110CADC 7B E7 00 D0 ADRP            X27, #classRef_NSString@PAGE
__text:000000010110CAE0 60 AB 41 F9 LDR             X0, [X27,#classRef_NSString@PAGEOFF]
__text:000000010110CAE4 08 E5 00 90 ADRP            X8, #selRef_stringWithFormat_@PAGE
__text:000000010110CAE8 1A 4D 40 F9 LDR             X26, [X8,#selRef_stringWithFormat_@PAGEOFF]
__text:000000010110CAEC F9 0F 1F F8 STR             X25, [SP,#-0x10]!
__text:000000010110CAF0 42 EC 00 D0+ADRL            X2, stru_102E96C60      ; " ?\x0F"
__text:000000010110CAF0 42 80 31 91
__text:000000010110CAF8 E1 03 1A AA MOV             X1, X26
__text:000000010110CAFC 76 BA 42 94 BL              _objc_msgSend           ; 格式化时间
__text:000000010110CB00 FF 43 00 91 ADD             SP, SP, #0x10
__text:000000010110CB04 F5 03 00 AA MOV             X21, X0
__text:000000010110CB08 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CB0C FD 03 1D AA MOV             X29, X29
__text:000000010110CB10 E0 03 15 AA MOV             X0, X21
__text:000000010110CB14 88 BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CB18 40 EC 00 D0+ADRL            X0, cfstr_6_7           ; "6"
__text:000000010110CB18 00 00 32 91
__text:000000010110CB20 7C BA 42 94 BL              _objc_retain
__text:000000010110CB24 68 AB 41 F9 LDR             X8, [X27,#classRef_NSString@PAGEOFF]
__text:000000010110CB28 7F 02 00 F1 CMP             X19, #0
__text:000000010110CB2C 09 03 93 9A CSEL            X9, X24, X19, EQ
__text:000000010110CB30 BF 02 00 F1 CMP             X21, #0
__text:000000010110CB34 0A 03 95 9A CSEL            X10, X24, X21, EQ
__text:000000010110CB38 FF 83 00 D1 SUB             SP, SP, #0x20 ; ' '
__text:000000010110CB3C EA 03 01 A9 STP             X10, X0, [SP,#0x10]
__text:000000010110CB40 F6 27 00 A9 STP             X22, X9, [SP]
__text:000000010110CB44 42 EC 00 D0+ADRL            X2, stru_102E96CA0      ; "\x8A|\x96\x12\x04\xAEu\x8E"
__text:000000010110CB44 42 80 32 91
__text:000000010110CB4C E0 03 08 AA MOV             X0, X8
__text:000000010110CB50 E1 03 1A AA MOV             X1, X26
__text:000000010110CB54 60 BA 42 94 BL              _objc_msgSend           ; UUID+时间
__text:000000010110CB58 FF 83 00 91 ADD             SP, SP, #0x20 ; ' '
__text:000000010110CB5C F9 03 00 AA MOV             X25, X0
__text:000000010110CB60 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CB64 FD 03 1D AA MOV             X29, X29
__text:000000010110CB68 E0 03 19 AA MOV             X0, X25
__text:000000010110CB6C 72 BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CB70 08 E5 00 D0 ADRP            X8, #selRef_lowercaseString@PAGE
__text:000000010110CB74 01 D5 47 F9 LDR             X1, [X8,#selRef_lowercaseString@PAGEOFF]
__text:000000010110CB78 57 BA 42 94 BL              _objc_msgSend           ; 转换成小写
__text:000000010110CB7C F6 03 00 AA MOV             X22, X0
__text:000000010110CB80 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CB84 FD 03 1D AA MOV             X29, X29
__text:000000010110CB88 E0 03 16 AA MOV             X0, X22
__text:000000010110CB8C 6A BA 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110CB90 E0 03 19 AA MOV             X0, X25
__text:000000010110CB94 5C BA 42 94 BL              _objc_release
__text:000000010110CB98 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110CB9C 36 DA FF B4 CBZ             X22, loc_10110C6E0
__text:000000010110CBA0 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
 
//加密
__text:000000010110C6E0 9F 7E 00 A9 STP             XZR, XZR, [X20]
__text:000000010110C6E4 9F 0A 00 F9 STR             XZR, [X20,#0x10]
__text:000000010110C6E8 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C6EC E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:000000010110C6F0 60 AB 41 F9 LDR             X0, [X27,#classRef_NSString@PAGEOFF]
__text:000000010110C6F4 E8 E4 00 F0 ADRP            X8, #selRef_alloc@PAGE
__text:000000010110C6F8 01 B1 47 F9 LDR             X1, [X8,#selRef_alloc@PAGEOFF]
__text:000000010110C6FC 76 BB 42 94 BL              _objc_msgSend
__text:000000010110C700 F9 03 00 AA MOV             X25, X0
__text:000000010110C704 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C708 E8 03 17 AA MOV             X8, X23
__text:000000010110C70C E0 03 14 AA MOV             X0, X20
__text:000000010110C710 39 69 FE 97 BL              EncCRC32_LodalID_loc_1056A2BF4 ; x0:指针uuid+时间
__text:000000010110C714 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C718 88 E6 00 D0 ADRP            X8, #selRef_initFromCppString_@PAGE
__text:000000010110C71C 01 11 46 F9 LDR             X1, [X8,#selRef_initFromCppString_@PAGEOFF]
__text:000000010110C720 E0 03 19 AA MOV             X0, X25
__text:000000010110C724 E2 03 17 AA MOV             X2, X23
__text:000000010110C728 6B BB 42 94 BL              _objc_msgSend
__text:000000010110C72C F9 03 00 AA MOV             X25, X0
__text:000000010110C730 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C734 E8 5E C0 39 LDRSB           W8, [X23,#0x17]
__text:000000010110C738 1F 01 01 72 TST             W8, #0x80000000
__text:000000010110C73C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C740 80 00 00 54 B.EQ            loc_10110C750
__text:000000010110C744 E0 02 40 F9 LDR             X0, [X23]
__text:000000010110C748 B3 B6 42 94 BL              __ZdlPv                 ; operator delete(void *)
__text:000000010110C74C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C750
__text:000000010110C750             loc_10110C750                           ; CODE XREF: __text:facebook::react::JSIExecutor::defaultTimeoutInvoker(std::function const&,std::function)+488A64↑j
__text:000000010110C750 77 AB 41 F9 LDR             X23, [X27,#classRef_NSString@PAGEOFF]
__text:000000010110C754 08 E5 00 90 ADRP            X8, #selRef_longLongValue@PAGE
__text:000000010110C758 01 F9 42 F9 LDR             X1, [X8,#selRef_longLongValue@PAGEOFF]
__text:000000010110C75C E0 03 19 AA MOV             X0, X25
__text:000000010110C760 5D BB 42 94 BL              _objc_msgSend
__text:000000010110C764 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C768 E0 0F 1F F8 STR             X0, [SP,#-0x10]!
__text:000000010110C76C 42 EC 00 D0+ADRL            X2, cfstr_H_8           ; "h\xEA\xCF\x2D\x01D"
__text:000000010110C76C 42 00 33 91
__text:000000010110C774 E0 03 17 AA MOV             X0, X23
__text:000000010110C778 E1 03 1A AA MOV             X1, X26
__text:000000010110C77C 56 BB 42 94 BL              _objc_msgSend
__text:000000010110C780 FF 43 00 91 ADD             SP, SP, #0x10
__text:000000010110C784 F7 03 00 AA MOV             X23, X0
__text:000000010110C788 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C78C FD 03 1D AA MOV             X29, X29
__text:000000010110C790 E0 03 17 AA MOV             X0, X23
__text:000000010110C794 68 BB 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110C798 60 AB 41 F9 LDR             X0, [X27,#classRef_NSString@PAGEOFF]
__text:000000010110C79C DF 02 00 F1 CMP             X22, #0
__text:000000010110C7A0 08 03 96 9A CSEL            X8, X24, X22, EQ
__text:000000010110C7A4 FF 02 00 F1 CMP             X23, #0
__text:000000010110C7A8 09 03 97 9A CSEL            X9, X24, X23, EQ
__text:000000010110C7AC E8 27 BF A9 STP             X8, X9, [SP,#-0x10]!
__text:000000010110C7B0 42 EC 00 D0+ADRL            X2, cfstr_A_9           ; "A"
__text:000000010110C7B0 42 00 30 91
__text:000000010110C7B8 E1 03 1A AA MOV             X1, X26
__text:000000010110C7BC 46 BB 42 94 BL              _objc_msgSend
__text:000000010110C7C0 FF 43 00 91 ADD             SP, SP, #0x10
__text:000000010110C7C4 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C7C8 FD 03 1D AA MOV             X29, X29
__text:000000010110C7CC 5A BB 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110C7D0 BA EF 00 F0 ADRP            X26, #qword_102F03060@PAGE
__text:000000010110C7D4 48 33 40 F9 LDR             X8, [X26,#qword_102F03060@PAGEOFF]
__text:000000010110C7D8 40 33 00 F9 STR             X0, [X26,#qword_102F03060@PAGEOFF]
__text:000000010110C7DC E0 03 08 AA MOV             X0, X8
__text:000000010110C7E0 49 BB 42 94 BL              _objc_release
__text:000000010110C7E4 98 E7 00 F0 ADRP            X24, #classRef_SAKGuardLocalIDKeychainStorage@PAGE
__text:000000010110C7E8 00 1F 43 F9 LDR             X0, [X24,#classRef_SAKGuardLocalIDKeychainStorage@PAGEOFF]
__text:000000010110C7EC 42 33 40 F9 LDR             X2, [X26,#qword_102F03060@PAGEOFF]
__text:000000010110C7F0 88 E6 00 D0 ADRP            X8, #selRef_hexString2Byte_@PAGE
__text:000000010110C7F4 01 D9 46 F9 LDR             X1, [X8,#selRef_hexString2Byte_@PAGEOFF]
__text:000000010110C7F8 37 BB 42 94 BL              _objc_msgSend
__text:000000010110C7FC FA 03 00 AA MOV             X26, X0
__text:000000010110C800 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C804 FD 03 1D AA MOV             X29, X29
__text:000000010110C808 E0 03 1A AA MOV             X0, X26
__text:000000010110C80C 4A BB 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110C810 00 1F 43 F9 LDR             X0, [X24,#classRef_SAKGuardLocalIDKeychainStorage@PAGEOFF]
__text:000000010110C814 88 E6 00 D0 ADRP            X8, #selRef_xorLocalEncrypt_@PAGE
__text:000000010110C818 01 DD 46 F9 LDR             X1, [X8,#selRef_xorLocalEncrypt_@PAGEOFF]
__text:000000010110C81C E2 03 1A AA MOV             X2, X26
__text:000000010110C820 2D BB 42 94 BL              _objc_msgSend           ; xorLocalEncrypt
__text:000000010110C824 FB 03 00 AA MOV             X27, X0
__text:000000010110C828 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:000000010110C82C FD 03 1D AA MOV             X29, X29
__text:000000010110C830 E0 03 1B AA MOV             X0, X27
__text:000000010110C834 40 BB 42 94 BL              _objc_retainAutoreleasedReturnValue
__text:000000010110C838 88 E6 00 D0 ADRP            X8, #selRef_byte2HexString@PAGE
__text:000000010110C83C 01 39 45 F9 LDR             X1, [X8,#selRef_byte2HexString@PAGEOFF]
__text:000000010110C840 25 BB 42 94 BL              _objc_msgSend
__text:000000010110C844 F8 03 00 AA MOV             X24, X0 
```

检测风险工具:

```
/Library/MobileSubstrate/DynamicLibraries/AXJ.plist
/Library/MobileSubstrate/DynamicLibraries/ALS.plist
/Library/MobileSubstrate/DynamicLibraries/fakephonelib.plist
/Library/MobileSubstrate/DynamicLibraries/AWZ.plist
iGrimace-X9
iGrimace3
/Library/MobileSubstrate/DynamicLibraries/igvx.plist
iGrimace-R8
/Library/MobileSubstrate/DynamicLibraries/R8.plist
iGrimace144
/Library/MobileSubstrate/DynamicLibraries/iGrimace.plist
iGrimaceV8E
zorro
/Applications/NZT.app
/Applications/AWZ.app
/var/mobile/awzdata
/var/mobile/hdFaker
/usr/bin/XGenDaemon.dylib
/var/mobile/GFaker
/usr/bin/iGevo
/var/root/Forge9_fix
/var/mobile/Library/XXAssistant/Lua/LocalLuas/
/Library/ApplicationSupport/XXAssistant/Lua/LocalLuas/
/Library/ApplicationSupport/XXIDEHelper/xsp/
/var/mobile/Library/XXAssistant/Lua/Luas/Temp/public
/Applications/HiddenApi.app
/Applications/Xgen.app
/Applications/BirdFaker9.app
/Applications/VPNMasterPro.app
/Applications/GuizmOVPN.app
/Applications/AXJ.app
/var/touchelf/scripts/
/var/mobile/Media/TouchSprite/lua/
/Applications/iG.app
/Applications/Forge9.app
/Applications/Forge.app
/Applications/GFaker.app
/Applications/hdfakerset.app
/Applications/R8.app
/Applications/Pranava.app
/Applications/RST.app
/Applications/WujiVPN.app
/Applications/TouchSprite.app
/Applications/TouchElf.app
/Applications/igvx.app
/var/mobile/iGrimace
/var/mobile/Library/Preferences/org.ioshack.igrimace.adv.plist
/Library/MobileSubstrate/DynamicLibraries/zorro.plist
/var/mobile/Library/Preferences/com.007gaiji.selapp.plist
/Library/MobileSubstrate/DynamicLibraries/rstweak.plist

```

检测代码

```
__text:00000001010FAACC E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001010FAAD0 F4 4F BE A9 STP             X20, X19, [SP,#-0x20]!
__text:00000001010FAAD4 FD 7B 01 A9 STP             X29, X30, [SP,#0x10]
__text:00000001010FAAD8 FD 43 00 91 ADD             X29, SP, #0x10
__text:00000001010FAADC F3 03 00 AA MOV             X19, X0
__text:00000001010FAAE0 8C 02 43 94 BL              _objc_retain
__text:00000001010FAAE4 59 38 00 94 BL              chrck_MobileSubstrate.dylib_loc_101108C48 ; 检测越狱风险
__text:00000001010FAAE8 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAAEC 1F 00 00 72 TST             W0, #1
__text:00000001010FAAF0 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAAF4 61 02 00 54 B.NE            loc_1010FAB40
__text:00000001010FAAF8 E0 03 13 AA MOV             X0, X19
__text:00000001010FAAFC 72 37 00 94 BL              fileExistsAtPath_loc_1011088C4
__text:00000001010FAB00 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB04 1F 00 00 72 TST             W0, #1
__text:00000001010FAB08 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB0C A1 01 00 54 B.NE            loc_1010FAB40
__text:00000001010FAB10 E0 03 13 AA MOV             X0, X19
__text:00000001010FAB14 E0 37 00 94 BL              fopen_loc_101108A94
__text:00000001010FAB18 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB1C 1F 00 00 72 TST             W0, #1
__text:00000001010FAB20 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB24 E1 00 00 54 B.NE            loc_1010FAB40
__text:00000001010FAB28 E0 03 13 AA MOV             X0, X19
__text:00000001010FAB2C 82 38 00 94 BL              access_loc_101108D34
__text:00000001010FAB30 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB34 1F 00 00 72 TST             W0, #1
__text:00000001010FAB38 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB3C 40 01 00 54 B.EQ            loc_1010FAB64
__text:00000001010FAB40
__text:00000001010FAB40             loc_1010FAB40
__text:00000001010FAB40                                                     ;
__text:00000001010FAB40 F4 03 00 32 MOV             W20, #1
__text:00000001010FAB44 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010FAB48 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001010FAB4C E0 03 13 AA MOV             X0, X19
__text:00000001010FAB50 6D 02 43 94 BL              _objc_release
__text:00000001010FAB54 E0 03 14 AA MOV             X0, X20
__text:00000001010FAB58 FD 7B 41 A9 LDP             X29, X30, [SP,#0x10]
__text:00000001010FAB5C F4 4F C2 A8 LDP             X20, X19, [SP],#0x20
//调用SVC 0X80
__text:000000010110DA0C E0 FB 7E A9 LDP             X0, X30, [SP,#var_18]
__text:000000010110DA10 E8 03 01 AA MOV             X8, X1
__text:000000010110DA14 E9 03 00 AA MOV             X9, X0
__text:000000010110DA18 0A 00 80 D2 MOV             X10, #0
__text:000000010110DA1C 8B 17 80 52 MOV             W11, #0xBC
__text:000000010110DA20 F0 03 0B AA MOV             X16, X11
__text:000000010110DA24 E0 03 09 AA MOV             X0, X9
__text:000000010110DA28 E1 03 08 AA MOV             X1, X8
__text:000000010110DA2C E2 03 0A AA MOV             X2, X10
__text:000000010110DA30 E3 03 0A AA MOV             X3, X10
__text:000000010110DA34 E4 03 0A AA MOV             X4, X10
__text:000000010110DA38 E5 03 0A AA MOV             X5, X10
__text:000000010110DA3C 01 10 00 D4 SVC             0x80
__text:000000010110DA40 E8 03 00 AA MOV             X8, X0
__text:000000010110DA44 00 7D 40 93 SXTW            X0, W8
__text:000000010110DA48 C0 03 5F D6 RET

```

转换成最终的 json 格式

```
{
    "m1": "0",
    "m2": "1505899243198",
    "m3": "imaicai",
    "m4": "AppSyncUnified-FrontBoard.dylib\nSSLKillSwitch2.dylib\n",
    "m5": "unknown",
    "m6": "144048128",
    "m7": "1.0",
    "m8": "[{}]",
    "m9": "中国电信",
    "m10": "1",
    "m11": "15F59763-196D-4B79-A514-AAE602B2DE888",
    "m12": "E68684F0-7573-4EBC-99BD-A03D54B88888",
    "m13": "1585614288859",
    "m14": "19.4.0",
    "m15": "0.000000",
    "m16": "0",
    "m17": "1",
    "m18": "0",
    "m19": "1",
    "m27": "7707a7cc28b649dc8d898888886bc6636114f9b83bad286d6388c7b4",
    "m122": "AA==",
    "m125": "AppStore",
    "m126": "unknown",
    "m127": "116.4853251045|39.9816589355",
    "m128": "[{\"bssid\":\"80:c5:48:4c:d4:6d\",\"ssid\":\"360wifi\"}]",
    "m129": "428015897",
    "m130": "unknown",
    "m131": "116180860928",
    "m132": "127989493760",
    "m133": "0e9eb7333944b26d58e302d324bb64d978888826f7361c5020dc945cb157427e",
    "m134": "unknown",
    "m135": "0.000000",
    "m136": "unknown",
    "m137": "DarwinKernelVersion19.4.0:MonFeb2422:04:12PST2020;root:xnu-6153.102.3~1/RELEASE_ARM64_T8010",
    "m138": "[5,100]",
    "m139": "2",
    "m140": "arm64",
    "m141": "1",
    "m142": "0.352187",
    "m143": "0",
    "m144": "5.25.0",
    "m145": "appstore",
    "m146": "1",
    "m147": "Darwin",
    "m148": "1634003530921.712",
    "m149": "1633915717678.990",
    "m150": "1635236908030",
    "m151": "iPhone",
    "m152": "5.2.11",
    "m153": "000000000000021887E4A9F494A9AA82E9C87829ECFF7A163379200827768888",
    "m154": "com.baobaoaichi.imaicai",
    "m155": "1634003189760",
    "m156": "172.17.225.15",
    "m157": "2099249152",
    "m158": "D10AP",
    "m159": "iPhone",
    "m160": "iOS13.4",
    "m161": "0.000000",
    "m162": "WiFi",
    "m163": "0",
    "m164": "zh-Hans-CN",
    "m165": "Asia/Shanghai(GMT+8)offset28800",
    "m166": "iPhone9,1",
    "m167": "750*1334",
    "m175": "5EAB68481C6CCC0A19DC5826B3D8C4261399DBA26D74AC5A051D0A5B0627CF80C73EDC1FA82951A5AFA81F41CF43CE30D6718CB9595C473E307ECB99B5A2B6E8268B52F6FC180CC39E9262610D21A9FA9583E51B5BE8907268E0E6283FCC94161E438F64BC5ED6AEBB8CB5ABE1E7836815A08C58A5F34F84471EF88AC4E562063B93280F9BFE648D5AEBCC770DA34CE50E9EAB0F535EC721113AB1012AC754A270A3EED79AD9D0F74CB9285A6D98837315F458CA8CB5F649F1C6D88812A2A28C1F92F6B714D2E38F37EE2E26564BDCAE77C94871B69EE8E0D1278D6788B68FB1A184CD9890B9D4EDECC016F153C7BB2D33E4F8F94446538EADCC47609F7ABA720D6A785C86C6E0F36263BAA556D367A1BD5E9E9771784ADAB668F38F612C78B70BAD7362718659020182CE84C4DD73021D5AAE948B28FC3B326DB8812EFEE47103342AABDA171E4E9A52288493FB6CCD07A13CD152161216BEB68D6DC19B2E88",
    "m200": "1583489309",
    "m249": "iphone",
    "m250": "Apple",
    "m253": "[]",
    "m254": "0",
    "m255": "0",
    "m256": "0",
    "m274": "1",
    "m293": "dad7bca2ce0c45fffb568e4e6436a921d88888ea3e5fcd07babfb97b",
    "m294": "{\"7\":\"\",\"3\":\"1\",\"4\":\"\",\"5\":\"\",\"1\":\"1\",\"33\":\"{\\\"7\\\":\\\"-\\\",\\\"3\\\":\\\"-\\\",\\\"8\\\":\\\"-\\\",\\\"4\\\":\\\"-\\\",\\\"0\\\":\\\"-\\\",\\\"9\\\":\\\"-\\\",\\\"5\\\":\\\"-\\\",\\\"1\\\":\\\"-\\\",\\\"6\\\":\\\"-\\\",\\\"2\\\":\\\"-\\\",\\\"10\\\":\\\"-\\\"}\",\"6\":\"1\",\"2\":\"\"}",
    "m304": "unknown",
    "m305": "750*1334",
    "m313": "2",
    "m306": "{}",
    "m307": "{\"m5\":5,\"m18\":7,\"m126\":7,\"m134\":5,\"m136\":5,\"m161\":7,\"m253\":5,\"m256\":7,\"m304\":5}"
}

```

压缩 json 文数据, 压缩后 (部分)

```
0000000104AF3800  78 9C 6D 56 CB 6E 2B C7  11 FD 15 82 AB 3C 44 BA  x.mV..+.......
0000000104AF3850  6F EC 7C 35 37 F3 8B F9  0D A8 60 9D 71 12 02 10  o...7.....`.q...
0000000104AF3860  DA 20 15 44 05 87 9B 6E  D8 76 43 1D 92 0E E3 ED  ...D...n..C.....
0000000104AF3870  ED E5 77 87 ED 6F 0E C3  7E D8 F5 8B F6 EE 78 78  ................ 
```

解密 PIC 获取 key(k1)

```
meituan1sankuai0

```

AES 加密压缩后数据

```
_text:00000001010BEA08 E0 7B 7B A9 LDP             X0, X30, [SP,#-0x50]
__text:00000001010BEA0C E0 43 00 91 ADD             X0, SP, #0x10
__text:00000001010BEA10 82 1E 80 52 MOV             W2, #0xF4
__text:00000001010BEA14 01 00 80 52 MOV             W1, #0
__text:00000001010BEA18 2B F2 43 94 BL              _memset
__text:00000001010BEA1C E1 03 19 32 MOV             W1, #0x80
__text:00000001010BEA20 E2 43 00 91 ADD             X2, SP, #0x10
__text:00000001010BEA24 E0 03 17 AA MOV             X0, X23
__text:00000001010BEA28 1A 51 FF 97 BL              InitKey_sub_102CBEE90   ; x0:key,x1:长度,x2:初始化后的key
__text:00000001010BEA2C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010BEA30 E2 07 40 F9 LDR             X2, [SP,#8]
__text:00000001010BEA34 E3 43 00 91 ADD             X3, SP, #0x10
__text:00000001010BEA38 E5 03 00 32 MOV             W5, #1
__text:00000001010BEA3C E0 03 15 AA MOV             X0, X21
__text:00000001010BEA40 E1 03 13 AA MOV             X1, X19
__text:00000001010BEA44 E4 03 16 AA MOV             X4, X22
__text:00000001010BEA48 3A 54 FF 97 BL              Aes_Enc_Dec_sub_102CBFB30 ; X0:原始数据,X1:初始化后的key,x2:大小,x3:key,X4:IV,X5:模式:0:解密,1:加密
__text:00000001010BEA4C E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]
__text:00000001010BEA50 E8 07 40 F9 LDR             X8, [SP,#8]
__text:00000001010BEA54 88 02 00 F9 STR             X8, [X20]
__text:00000001010BEA58 E0 7B 3B A9 STP             X0, X30, [SP,#-0x50]

```

AES 加密后 (部分)

```
0000000104AC3200  2C 36 30 1D 89 2C 90 E9  7E F7 1C DB 62 73 D7 5E  ,60..,.......s..
0000000104AC3210  D2 FB 8C AC 1F 31 43 F2  AF B2 DE F1 29 78 33 4B  .....1C........K
0000000104AC3220  8F AA 19 B8 40 E1 CF 85  CF 78 DB C1 19 B6 9A F5  ....@...........
0000000104AC3230  02 1A 23 C1 AC 88 31 C0  98 2A 3E 1A 30 2F 75 D5  ..#...1..*>.0/u.
0000000104AC3240  73 23 B8 15 E2 2A 32 A3  13 4B 38 86 6A 4F 87 2B  s#.......K8.jO.+
0000000104AC3250  D8 D4 84 69 35 B4 B3 2B  BE D8 37 D3 50 C0 FC 9E  ...i5..+........
0000000104AC3260  95 E6 40 38 3D FA 91 4B  4B EA BF 4D BE C3 46 DE  ....=..KK.......
0000000104AC3270  A7 EB EE BC CF 06 1B 42  E9 52 3F 7A 7F AF B8 55  .......B...z...U

```

base64 加密

```
LDYwHYkskOl+9xzbYnPXXtL7jKwfMUPyr7Le8Sl4M0uPqhm4QOHPhc9428EZtpr1AhojwayIMcCYKj4aMC911XMjuBXiKjKjE0s4hmpPhyvY1IRpNbSzK77YN9NQwPyeleZAOD36kUtL6r9NvsNG3qfr7rzPBhtC6VI/en+vuFXQEnZJ/Tv6/C03xQCAfJS2Uh7lKMgZe0MZGoANUpLs1+J6rxG9X+LkynUQKPKBxZNSt/q6FywBCbHA5uDKuoxKVa4rSlZCYfBaZbLaIC6iiwmKg4PjUoMzeyIHUkHh+nzEgJWfwE14H/O9ZwUQG68yGBYmtHEY05Bn6V5BROAVpXtyqJTuKg/PIUueX9QMouF0OzcdZwIJt7kAOfdDKfZfYVkovWBwYRXEnHiQ5Q54WyBqU79b8G0PlMFVvYkyw5xMXzmS/5wHuHSDU2xMDePTDDoxNHae4vmqrx+WMgT5M81CWQl/Jyv+Qo/bTj2UGgWwyQpn3TrMbxeB+sNKvBHAiBAG1nS2Q5JmPkpNcYZXQSu1JjDa+eDXuUlpeBl+qRN4wwjIrCZVAYdyGEVsMN461ajPIjrx8uwaBNDlT348tA9vKvADG4na33OK/wK0fM8d+IzktB8OxzLNTxAl6jN6u9CdWTLW1rixFGA8dHqv4RorX2DYIbZSclw4vS2vJrdDBLOuzE28sNZTAhA5I9MSqZkusMXfrua9KS+yiNwXeSWoaj5/DZ452nhfKtJWujRj5rjxI+y3s9e5+/E7+pFvI8WsDOPyURSkb/aDZXshpr1IWyHlDCVrF7OdPZ7dYpfnEYsvRrDIPhkl3vjwtCV8LlaGR0nRZWINBBCQvGcefbIAgAdRYOsxbpJxiKNjKL/ckPIa4c2QdzlaVHyvrtmwOOr2KLADIxNqzVG+b5n0Fw4ERXFd3F/+HEi8bfeXHBDfmTny9Az989dE1CffiyFR/BtiZr85BpGgooTI/C4arlmDtZMqArN0m5lb5IBbrxiQ4F5jRfSai+hwMziJtCRKj1LzXyVP4is+GXRN9MkSvu7qnwkCmmXLHArQD6qs3FP/yO6mplr494Q0YfAm0EGcIeph1lIKT6c+zVlzTxgZnWGzgoVJ9SwtDSOmG1Njm8ZXD1HeoqRO3b8LcWuKYScJqmplk3HlfwNvzlPhDaiSlvESsi4CpbDWeBhaU0vDoYva8MwAd6q3Bo8ePnp941fidcfIJV176wzQdixhLuYje4RzoYMl9fAKd4ns6LLDLEyMv65T3xAykoXWyjI+RS6MZtwob9JWt+gUkXQMoI41re0/w0MPGrJX09M+K7eCVZNGZmJ7I0DpvclVVCRT17BGUjoLd+9TvOi/bWlnTd29b0TqP0hVI9hgCY/0F9B5kYQNKvGNjUtXqIEiIEZEHE9tmBaWwnSmiZ4qgiTTCoiiZm4IJ7yPIFNkgdGkQkgLGbn7tVNyjDasLWcFZ9jmj8PYE6a4fQ3BtT3RGXCRm5v6JNEvo42pi6yOmBzZy2hdIkwKGtQT/JNH3HccgyiPjz4JIuU/LRhgxyPu5TdD29vAaCS5XhfkMDxIiMOWZcNkyeDUez9DHx7XFnJn6b89541ZJewkZjZHf5TxZAPEG7oM6eEX4e9tA/SpQ8uuNyyrZ1EbCrWe/AtjNd/seXw8Nrfy34SpgvbsfdPR1gIk6fjD2TfVR/JBPlEyJxoxFaU1jQ3550pahTtMDtDEgManoqNL4+ux+cvVJoTwOrf4rt4xzfdy+4H8b508dhdBZAHyUa3A2ipmeYc8bIWKNt6AdS6DjrqYRvfoifod1lsDqU7xhIo+LnIE6F5p+jF+QYe+VwTJ/yIp+o8JUyxqJv/N3j4j3cH/EVQDon7Mxtsh1nYW598eMzeQVmrfydwUo9E73C6yp4xtLR0sKEbkJfczx53lPNpFxm7wZxBLao35p0Pawa30YJwTmCLG9iVbC3YKaWEvki6YI4ZgUQvemSkZDEtfJmo5b0FP9ih/1soqWKbXgW1lEr86oDmRHiFwxONgyvzgXmN0mITkYN4Wtxu6jMWZ

```

组合请求体

```
{
    "encryptVersion": "1",
    "src": "1",
    "fingerPrintData": "LDYwHYkskOl+9xzbYnPXXtL7jKwfMUPyr7Le8Sl4M0uPqhm4QOHPhc9428EZtpr1AhojwayIMcCYKj4aMC911XMjuBXiKjKjE0s4hmpPhyvY1IRpNbSzK77YN9NQwPyeleZAOD36kUtL6r9NvsNG3qfr7rzPBhtC6VI\/en+vuFXQEnZJ\/Tv6\/C03xQCAfJS2Uh7lKMgZe0MZGoANUpLs1+J6rxG9X+LkynUQKPKBxZNSt\/q6FywBCbHA5uDKuoxKVa4rSlZCYfBaZbLaIC6iiwmKg4PjUoMzeyIHUkHh+nzEgJWfwE14H\/O9ZwUQG68yGBYmtHEY05Bn6V5BROAVpXtyqJTuKg\/PIUueX9QMouF0OzcdZwIJt7kAOfdDKfZfYVkovWBwYRXEnHiQ5Q54WyBqU79b8G0PlMFVvYkyw5xMXzmS\/5wHuHSDU2xMDePTDDoxNHae4vmqrx+WMgT5M81CWQl\/Jyv+Qo\/bTj2UGgWwyQpn3TrMbxeB+sNKvBHAiBAG1nS2Q5JmPkpNcYZXQSu1JjDa+eDXuUlpeBl+qRN4wwjIrCZVAYdyGEVsMN461ajPIjrx8uwaBNDlT348tA9vKvADG4na33OK\/wK0fM8d+IzktB8OxzLNTxAl6jN6u9CdWTLW1rixFGA8dHqv4RorX2DYIbZSclw4vS2vJrdDBLOuzE28sNZTAhA5I9MSqZkusMXfrua9KS+yiNwXeSWoaj5\/DZ452nhfKtJWujRj5rjxI+y3s9e5+\/E7+pFvI8WsDOPyURSkb\/aDZXshpr1IWyHlDCVrF7OdPZ7dYpfnEYsvRrDIPhkl3vjwtCV8LlaGR0nRZWINBBCQvGcefbIAgAdRYOsxbpJxiKNjKL\/ckPIa4c2QdzlaVHyvrtmwOOr2KLADIxNqzVG+b5n0Fw4ERXFd3F\/+HEi8bfeXHBDfmTny9Az989dE1CffiyFR\/BtiZr85BpGgooTI\/C4arlmDtZMqArN0m5lb5IBbrxiQ4F5jRfSai+hwMziJtCRKj1LzXyVP4is+GXRN9MkSvu7qnwkCmmXLHArQD6qs3FP\/yO6mplr494Q0YfAm0EGcIeph1lIKT6c+zVlzTxgZnWGzgoVJ9SwtDSOmG1Njm8ZXD1HeoqRO3b8LcWuKYScJqmplk3HlfwNvzlPhDaiSlvESsi4CpbDWeBhaU0vDoYva8MwAd6q3Bo8ePnp941fidcfIJV176wzQdixhLuYje4RzoYMl9fAKd4ns6LLDLEyMv65T3xAykoXWyjI+RS6MZtwob9JWt+gUkXQMoI41re0\/w0MPGrJX09M+K7eCVZNGZmJ7I0DpvclVVCRT17BGUjoLd+9TvOi\/bWlnTd29b0TqP0hVI9hgCY\/0F9B5kYQNKvGNjUtXqIEiIEZEHE9tmBaWwnSmiZ4qgiTTCoiiZm4IJ7yPIFNkgdGkQkgLGbn7tVNyjDasLWcFZ9jmj8PYE6a4fQ3BtT3RGXCRm5v6JNEvo42pi6yOmBzZy2hdIkwKGtQT\/JNH3HccgyiPjz4JIuU\/LRhgxyPu5TdD29vAaCS5XhfkMDxIiMOWZcNkyeDUez9DHx7XFnJn6b89541ZJewkZjZHf5TxZAPEG7oM6eEX4e9tA\/SpQ8uuNyyrZ1EbCrWe\/AtjNd\/seXw8Nrfy34SpgvbsfdPR1gIk6fjD2TfVR\/JBPlEyJxoxFaU1jQ3550pahTtMDtDEgManoqNL4+ux+cvVJoTwOrf4rt4xzfdy+4H8b508dhdBZAHyUa3A2ipmeYc8bIWKNt6AdS6DjrqYRvfoifod1lsDqU7xhIo+LnIE6F5p+jF+QYe+VwTJ\/yIp+o8JUyxqJv\/N3j4j3cH\/EVQDon7Mxtsh1nYW598eMzeQVmrfydwUo9E73C6yp4xtLR0sKEbkJfczx53lPNpFxm7wZxBLao35p0Pawa30YJwTmCLG9iVbC3YKaWEvki6YI4ZgUQvemSkZDEtfJmo5b0FP9ih\/1soqWKbXgW1lEr86oDmRHiFwxONgyvzgXmN0mITkYN4Wtxu6jMWZ"
}

```

计算签名，签名过程与上面分析的流程是一样的。

```
+[SAKGuardCommon sign:attachSiua:]
//获取info.plist中的appkeey 6d1efb41-1bb2-4db1-88ee-b89d21d06e5f

```

dfpid 逻辑差不多，这里就不分析了。

### [](#七、算法还原)七、算法还原

#### 7.1、加密设备指纹请求体算法 (不全部展开了吧, 大多都是标准算法)

设备指纹相关用到的算法有 AES、压缩、RC4、hmac、base64。  
RC4:

```
#include #include //#include "base64.h"
 
/*********************************************************************
* Filename:   rc4.c
* Copyright:
*********************************************************************/
 
typedef unsigned long ULONG;
 
void rc4_init(unsigned char* s, unsigned char* key, unsigned long Len) //初始化函数
{
    int i = 0, j = 0;
    char k[256] = { 0 };
    unsigned char tmp = 0;
    for (i = 0; i < 256; i++) {
        s[i] = i;
        k[i] = key[i % Len];
    }
    for (i = 0; i < 256; i++) {
        j = (j + s[i] + k[i]) % 256;
        tmp = s[i];
        s[i] = s[j]; //交换s[i]和s[j]
        s[j] = tmp;
    }
}
 
void rc4_crypt(unsigned char* s, unsigned char* Data, unsigned long Len) //加解密
{
    int i = 0, j = 0, t = 0;
    unsigned long k = 0;
    unsigned char tmp;
    for (k = 0; k < Len; k++) {
        i = (i + 1) % 256;
        j = (j + s[i]) % 256;
        tmp = s[i];
        s[i] = s[j]; //交换s[x]和s[y]
        s[j] = tmp;
        t = (s[i] + s[j]) % 256;
        Data[k] ^= s[t];
    }
} 
```

AES

```
/*******************
* AES - CBC
*******************/
int aes_encrypt_cbc(const BYTE in[], size_t in_len, BYTE out[], const WORD key[], int keysize, const BYTE iv[])
{
    BYTE buf_in[AES_BLOCK_SIZE], buf_out[AES_BLOCK_SIZE], iv_buf[AES_BLOCK_SIZE];
    int blocks, idx;
 
    if (in_len % AES_BLOCK_SIZE != 0)
        return(FALSE);
 
    blocks = in_len / AES_BLOCK_SIZE;
 
    memcpy(iv_buf, iv, AES_BLOCK_SIZE);
 
    for (idx = 0; idx < blocks; idx++) {
        memcpy(buf_in, &in[idx * AES_BLOCK_SIZE], AES_BLOCK_SIZE);
        xor_buf(iv_buf, buf_in, AES_BLOCK_SIZE);
        aes_encrypt(buf_in, buf_out, key, keysize);
        memcpy(&out[idx * AES_BLOCK_SIZE], buf_out, AES_BLOCK_SIZE);
        memcpy(iv_buf, buf_out, AES_BLOCK_SIZE);
    }
 
    return(TRUE);
}

```

测试解密设备指纹请求体

```
//解密fingerPrintData
    BYTE base64_fingerPrintData[1][10434] = {
    {"LDYwHYkskOl+9xzbYnPXXtL7jKwfMUPyr7Le8Sl4M0uPqhm4QOHPhc9428EZtpr1AhojwayIMcCYKj4aMC911XMjuBXiKjKjE0s4hmpPhyvY1IRpNbSzK77YN9NQwPyeleZAOD36kUtL6r9NvsNG3qfr7rzPBhtC6VI\/en+vuFXQEnZJ\/Tv6\/C03xQCAfJS2Uh7lKMgZe0MZGoANUpLs1+J6rxG9X+LkynUQKPKBxZNSt\/q6FywBCbHA5uDKuoxKVa4rSlZCYfBaZbLaIC6iiwmKg4PjUoMzeyIHUkHh+nzEgJWfwE14H\/O9ZwUQG68yGBYmtHEY05Bn6V5BROAVpXtyqJTuKg\/PIUueX9QMouF0OzcdZwIJt7kAOfdDKfZfYVkovWBwYRXEnHiQ5Q54WyBqU79b8G0PlMFVvYkyw5xMXzmS\/5wHuHSDU2xMDePTDDoxNHae4vmqrx+WMgT5M81CWQl\/Jyv+Qo\/bTj2UGgWwyQpn3TrMbxeB+sNKvBHAiBAG1nS2Q5JmPkpNcYZXQSu1JjDa+eDXuUlpeBl+qRN4wwjIrCZVAYdyGEVsMN461ajPIjrx8uwaBNDlT348tA9vKvADG4na33OK\/wK0fM8d+IzktB8OxzLNTxAl6jN6u9CdWTLW1rixFGA8dHqv4RorX2DYIbZSclw4vS2vJrdDBLOuzE28sNZTAhA5I9MSqZkusMXfrua9KS+yiNwXeSWoaj5\/DZ452nhfKtJWujRj5rjxI+y3s9e5+\/E7+pFvI8WsDOPyURSkb\/aDZXshpr1IWyHlDCVrF7OdPZ7dYpfnEYsvRrDIPhkl3vjwtCV8LlaGR0nRZWINBBCQvGcefbIAgAdRYOsxbpJxiKNjKL\/ckPIa4c2QdzlaVHyvrtmwOOr2KLADIxNqzVG+b5n0Fw4ERXFd3F\/+HEi8bfeXHBDfmTny9Az989dE1CffiyFR\/BtiZr85BpGgooTI\/C4arlmDtZMqArN0m5lb5IBbrxiQ4F5jRfSai+hwMziJtCRKj1LzXyVP4is+GXRN9MkSvu7qnwkCmmXLHArQD6qs3FP\/yO6mplr494Q0YfAm0EGcIeph1lIKT6c+zVlzTxgZnWGzgoVJ9SwtDSOmG1Njm8ZXD1HeoqRO3b8LcWuKYScJqmplk3HlfwNvzlPhDaiSlvESsi4CpbDWeBhaU0vDoYva8MwAd6q3Bo8ePnp941fidcfIJV176wzQdixhLuYje4RzoYMl9fAKd4ns6LLDLEyMv65T3xAykoXWyjI+RS6MZtwob9JWt+gUkXQMoI41re0\/w0MPGrJX09M+K7eCVZNGZmJ7I0DpvclVVCRT17BGUjoLd+9TvOi\/bWlnTd29b0TqP0hVI9hgCY\/0F9B5kYQNKvGNjUtXqIEiIEZEHE9tmBaWwnSmiZ4qgiTTCoiiZm4IJ7yPIFNkgdGkQkgLGbn7tVNyjDasLWcFZ9jmj8PYE6a4fQ3BtT3RGXCRm5v6JNEvo42pi6yOmBzZy2hdIkwKGtQT\/JNH3HccgyiPjz4JIuU\/LRhgxyPu5TdD29vAaCS5XhfkMDxIiMOWZcNkyeDUez9DHx7XFnJn6b89541ZJewkZjZHf5TxZAPEG7oM6eEX4e9tA\/SpQ8uuNyyrZ1EbCrWe\/AtjNd\/seXw8Nrfy34SpgvbsfdPR1gIk6fjD2TfVR\/JBPlEyJxoxFaU1jQ3550pahTtMDtDEgManoqNL4+ux+cvVJoTwOrf4rt4xzfdy+4H8b508dhdBZAHyUa3A2ipmeYc8bIWKNt6AdS6DjrqYRvfoifod1lsDqU7xhIo+LnIE6F5p+jF+QYe+VwTJ\/yIp+o8JUyxqJv\/N3j4j3cH\/EVQDon7Mxtsh1nYW598eMzeQVmrfydwUo9E73C6yp4xtLR0sKEbkJfczx53lPNpFxm7wZxBLao35p0Pawa30YJwTmCLG9iVbC3YKaWEvki6YI4ZgUQvemSkZDEtfJmo5b0FP9ih\/1soqWKbXgW1lEr86oDmRHiFwxONgyvzgXmN0mITkYN4Wtxu6jMWZ"}
    };
    base64_len = mc_base64(base64_fingerPrintData, strlen(base64_fingerPrintData[0]), outdata, 0);
    if (0 == base64_len) {
        printf("mc_base64 error!\n");
        return -1;
    }
    aesret = aes_decrypt_cbc(outdata, base64_len, out_ciphertext[0], key_schedule, 128, iv[0]);
    if (1 != aesret) {
        printf("aes_decrypt_cbc error!\n");
        return -1;
    }
 
    /* 解压缩 */
    uLong blen;
    uLong dslen;
    BYTE un_outdata[10434] = { 0 };
 
    blen = compressBound(base64_len);
 
    if (uncompress(un_outdata, &dslen, out_ciphertext[0], blen) != Z_OK)
    {
        printf("uncompress failed!\n");
        return -1;
    }
    printf("fingerPrintData: %s\n", un_outdata);

```

解密出来的数据与上面分析的组合设备指纹 json 是一样的，解密成功，如图 7-1 所示:  
![](https://bbs.pediy.com/upload/attach/202111/536985_9HHXNFNE3D4JCTM.png)  
图 7-1

### [](#八、总结)八、总结

我从分析的角度说下自己的看法，不对的地方还请指正，抗分析能力一般，代码混淆规律性很强，字符串加密方法用的一个容易被一次性还原。获取设备信息过于频繁，影响性能，回到开始说的风控在业务中的作用，大部分用户使用生鲜类 APP 时的目的性比较强，业务在拉新促活增加用户粘性的同时高质量留存与业务安全更是重中之重，所以产品流畅的用户体验是促进高留存的重要条件之一。  
还有一些隐藏的彩蛋比较有意思，感兴趣的可以去自行分析。

 

样本太大，获取方式，关注公众号，公众号输入框回复 “mc” 获取下载链接。  
![](https://bbs.pediy.com/upload/attach/202111/536985_ARD8XV3PCYTRWGF.png)

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 1 小时前 被我是小三编辑 ，原因：

[#逆向分析](forum-166-1-189.htm)