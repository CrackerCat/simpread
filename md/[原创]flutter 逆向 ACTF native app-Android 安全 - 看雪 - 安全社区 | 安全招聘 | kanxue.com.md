> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-279403.htm)

> [原创]flutter 逆向 ACTF native app

[原创]flutter 逆向 ACTF native app

13 小时前 313

### [原创]flutter 逆向 ACTF native app

 [![](http://passport.kanxue.com/upload/avatar/320/963320.png?1667284712)](user-home-963320.htm) [oacia](user-home-963320.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 13 小时前  313

前言
==

算了一下好长时间没打过 CTF 了, 前两天看到 ACTF 逆向有道 flutter 逆向题就过来玩玩啦, 花了一个下午做完了. 说来也巧, 我给 DASCTF 十月赛出的逆向题其中一道也是 flutter, 不过那题我难度降的相当之低啦, 不知道有多少人做出来了呢~

还原函数名
=====

flutter 逆向的一大难点就是不知道`libapp.so`的函数名, 虽然有工具`reflutter`可以帮助我们得到其中的符号, 但是我个人认为基于对`libflutter.so`源码插桩后重编译再重打包 apk 的方式具有极大的不可预料性, 极有可能导致 apk 闪退, 这一题便出现了这种情况, 所以接下来我将介绍的工具`blutter`是纯静态分析来还原函数名, 更令人惊喜的是它提供了`IDApython`脚本来让我们可以在 IDA 中对函数进行重命名, 而这个项目中提供的其他文件也相当好用

blutter 的编译及使用
--------------

blutter 项目地址

```
https://github.com/worawit/blutter

```

在各个平台如何编译在这个项目的 README.md 中写的已经相当详细了, 这里我就简单介绍一下 Windows 上的编译过程吧, 注意一下这些命令需要全程运行在代理环境否则会导致无法下载

首先 clone 项目

```
git clone https://github.com/worawit/blutter --depth=1

```

随后运行初始化脚本

```
cd .\blutter\
python .\scripts\init_env_win.py

```

请注意, 接下来我们需要打开`x64 Native Tools Command Prompt`, 它可以在`Visual Studio`文件夹中找到

![](https://bbs.kanxue.com/upload/attach/202310/963320_MFVX569Z367GMZN.png)

然后运行`blutter.py`并提供`libapp.so`和`libflutter.so`的文件夹路径以及输出文件夹路径

```
python .\blutter.py ..\chall\lib\arm64-v8a\ .\output

```

![](https://bbs.kanxue.com/upload/attach/202310/963320_8SE9Q6XPXPQNKRG.png)

输出文件夹目录如下

![](https://bbs.kanxue.com/upload/attach/202310/963320_EXQ7X5P6KS763RX.png)

随后我们用 ida 反编译`libapp.so`, 并运行输出文件夹中的 IDApython 脚本`ida_script/addNames.py`, 符号就被全部恢复出来啦

![](https://bbs.kanxue.com/upload/attach/202310/963320_2YZJDCGNUYWEFE9.png)

hook 关键函数 获取函数参数
================

这里我们需要关注的函数是`flutter_application_1_main__LongPressDemoState::_onTap`, 因为在 flutter 的开发中, onTap 函数是按钮点击之后的响应函数

![](https://bbs.kanxue.com/upload/attach/202310/963320_7FHYGYEEZYMWY6Y.png)

随后我们进入`sub_1DE500`, 在该函数中双击`sub_1DE59C`进入

![](https://bbs.kanxue.com/upload/attach/202310/963320_2NGTJ6XQQ6V2PWP.png)

在这个函数中我们发现了`256`,`%`,`^`这些特征, 合理猜测一下算法可能是 RC4![](https://bbs.kanxue.com/upload/attach/202310/963320_BQ4VDSEVN49PGKA.png)

![](https://bbs.kanxue.com/upload/attach/202310/963320_GM8G96YGJ2H8SBG.png)

![](https://bbs.kanxue.com/upload/attach/202310/963320_MC624UWY8EDGAN6.png)

接下来我们使用输出文件夹中的`blutter_frida.js`hook 一下`sub_1DE59C`看看情况

![](https://bbs.kanxue.com/upload/attach/202310/963320_F32MESU23H6G8ZB.png)

```
PS D:\hgame\ACTF\native app\work\blutter> frida -U -f "com.example.flutter_application_1" -l .\output\blutter_frida.js
[Pixel 3::com.example.flutter_application_1 ]->
Unhandle class id: 46, TypeArguments
GrowableList@6d00488c29 = [
  188676,
  0,
  {
    "key": "Unhandle class id: 46, TypeArguments"
  },
  34,
  {
    "key": [
      184,
      132,
      137,
      215,
      146,
      65,
      86,
      157,
      123,
      100,
      179,
      131,
      112,
      170,
      97,
      210,
      163,
      179,
      17,
      171,
      245,
      30,
      194,
      144,
      37,
      41,
      235,
      121,
      146,
      210,
      174,
      92,
      204,
      22
    ]
  },
  0,
  0,
  0
]

```

这里我们只 hook 到一个数组的值, 另一个数组的类型是`TypeArguments`, 研究了一下`blutter_frida.js`后发现作者还没有对这种数据类型格式提供 hook 支持

![](https://bbs.kanxue.com/upload/attach/202310/963320_NCJGBA8PY5RKHD3.png)

IDA 动态调试 libapp.so
==================

现在我们得到了一个数组, 我们就暂时认为它就是 flag 经过加密之后得到的结果, 接下来我们在 IDA 中对`sub_1DE59C`下断点动态调试来更加深入的研究一下

首先我们需要将 IDA 文件夹中的`dbgsrv/android_server64` push 到手机上面, 然后运行一下并且指定端口

```
blueline:/data/local/tmp # ./as64 -p 11112
IDA Android 64-bit remote debug server(ST) v7.7.27. Hex-Rays (c) 2004-2022
Listening on 0.0.0.0:11112...

```

随后端口转发一下

```
PS C:\Users\oacia> adb forward tcp:11112 tcp:11112
11112

```

在 IDA 中选择调试器为`Android debugger`

![](https://bbs.kanxue.com/upload/attach/202310/963320_CWU2U7SEGDTUXEV.png)

随后点击`Debugger->Debugger options...`选择如下配置

![](https://bbs.kanxue.com/upload/attach/202310/963320_3WDWY3VANC7XDVA.png)

点击`Debugger->Process options...`,`Hostname`修改为`127.0.0.1`,`Port`修改为`11112`

![](https://bbs.kanxue.com/upload/attach/202310/963320_F5ZXSQ9S9F8PRHS.png)

然后点击`Debugger->Attach to process...`, 附加到我们目标包名的进程上面

![](https://bbs.kanxue.com/upload/attach/202310/963320_T5PNZXHPZ2V2K83.png)

弹出该弹窗选择 Same 即可

![](https://bbs.kanxue.com/upload/attach/202310/963320_6DSHQUM3QAC9W9H.png)

在手机上点击按钮, 然后在 IDA 中点击这个绿色的剪头, 就可以动态调试啦

![](https://bbs.kanxue.com/upload/attach/202310/963320_RYU2BUUNECY5R5J.png)

![](https://bbs.kanxue.com/upload/attach/202310/963320_EJZA7BBN43NHQRT.png)

在动态调试之后, 未知的变量也逐渐浮现了出来, 这里我们发现了`v28>=256`, 那么很有可能就是 RC4 了哦

![](https://bbs.kanxue.com/upload/attach/202310/963320_EDRUMJUKRPN897U.png)

既然这样, 那么直接在这里唯一的异或的地方用 IDA 去 trace 一下, 把异或的数组 dump 下来不就行了:)

![](https://bbs.kanxue.com/upload/attach/202310/963320_6F573XUGZBVAAEV.png)

于是我们得到了被异或的数组了

![](https://bbs.kanxue.com/upload/attach/202310/963320_B6TPMVQNHB8SVQ2.png)

但是在异或运算的地方下断点之后, 我输入的数全都是`1`, 这里被异或的数也全是`0xce`

![](https://bbs.kanxue.com/upload/attach/202310/963320_VAG2DFT5HDT9V4E.png)

所以莫非不是 RC4? 让 0xce 和`0x31`异或一下看看, 竟然是`0xff`这么有意义的数字

![](https://bbs.kanxue.com/upload/attach/202310/963320_J7YU5M4WFBEMB2P.png)

所以 exp 也就能写出来啦~

```
final = [184, 132, 137, 215, 146, 65, 86, 157, 123, 100, 179, 131, 112, 170, 97, 210, 163, 179, 17, 171, 245, 30, 194,
         144, 37, 41, 235, 121, 146, 210, 174, 92, 204, 22]
xor = [14, 14, 68, 80, 29, 201, 241, 46, 197, 208, 123, 79, 187, 55, 234, 104, 40, 117, 133, 12, 67, 137, 91, 31, 136,
       177, 64, 234, 24, 27, 26, 214, 122, 217]
 
flag = [chr(xor[i]^final[i]^0xff) for i in range(len(final))]
print(''.join(flag))
# Iu2xpwXLAK734btEt9kXIhfpRgTlu6KuI0

```

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#逆向分析](forum-161-1-118.htm)

上传的附件：

*   [native app.zip.001](javascript:void(0)) （8.00MB，1 次下载）
*   [native app.zip.002](javascript:void(0)) （8.00MB，1 次下载）
*   [native app.zip.003](javascript:void(0)) （1.09MB，1 次下载）