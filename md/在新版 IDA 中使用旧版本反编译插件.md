> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/aqmwaCIOKzKfrGFfhRXBPQ)

众所周知，上周末 IDA Pro 最新的 8.3 版本泄漏了，自带 x86 和 x86_64 的反编译插件。

看雪和吾爱上不少网友都提到没有 hexarm 和 hexarm64，那么我们能不能直接在 8.3 版本上直接使用 7.7 的反编译插件呢？

搜索类似 “ida old version hexrays decompiler github” 的关键词，很快能找到别人提供的一个开源解决方案 ida_dll_shim[1]

粗略浏览代码可知，这个 dll 会提供一些 ida.dll 和 ida64.dll 不再导出的 API 给旧版本插件使用。

下载代码然后自行编译，装上之后发现确实可以加载 hexarm.dll 以及 hexarm64.dll 了。

但是有的函数点进去 IDA 就会卡死，估计是啥地方死循环了。

windbg 附加上去，切换到 0 号线程上，观察栈回溯：

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 0000001c`8dbf9c50 00000000`6eae826e     hexarm64+0xbcc61
01 0000001c`8dbf9cb0 00000000`6ec1347e     hexarm64+0xc826e
02 0000001c`8dbf9ce0 00000000`6ec136f1     hexarm64+0x1f347e
03 0000001c`8dbf9d10 00000000`6ec136f1     hexarm64+0x1f36f1
04 0000001c`8dbf9d40 00000000`6ec135a3     hexarm64+0x1f36f1
05 0000001c`8dbf9d70 00000000`6ec135a3     hexarm64+0x1f35a3
06 0000001c`8dbf9da0 00000000`6ec190e1     hexarm64+0x1f35a3
07 0000001c`8dbf9dd0 00000000`6ec1347e     hexarm64+0x1f90e1
08 0000001c`8dbf9e00 00000000`6ec136f1     hexarm64+0x1f347e
09 0000001c`8dbf9e30 00000000`6ec135df     hexarm64+0x1f36f1
0a 0000001c`8dbf9e60 00000000`6ec1380b     hexarm64+0x1f35df
0b 0000001c`8dbf9e90 00000000`6ead8142     hexarm64+0x1f380b
0c 0000001c`8dbf9ef0 00000000`6ec141e9     hexarm64+0xb8142
0d 0000001c`8dbf9fa0 00000000`6ec2534f     hexarm64+0x1f41e9
0e 0000001c`8dbfa050 00000000`6eb65977     hexarm64+0x20534f
0f 0000001c`8dbfa260 00000000`6eb64bf4     hexarm64+0x145977
10 0000001c`8dbfa450 00000000`6ebd6d46     hexarm64+0x144bf4
11 0000001c`8dbfa4a0 00000000`6ebd48b7     hexarm64+0x1b6d46
12 0000001c`8dbfa630 00000000`6ebd4787     hexarm64+0x1b48b7
13 0000001c`8dbfa6c0 00007ff7`807da0d3     hexarm64+0x1b4787
14 0000001c`8dbfa790 00007ffd`b1df7848     ida64_exe+0x6a0d3
15 0000001c`8dbfa830 00007ffd`b1e986f0     Qt5Widgets!QT::QWidget::event+0x148
16 0000001c`8dbfa8c0 00007ffd`b130501c     Qt5Widgets!QT::QFrame::event+0x30
17 0000001c`8dbfa8f0 00007ffd`b1dd483d     Qt5Core!QT::QCoreApplicationPrivate::sendThroughObjectEventFilters+0xdc
18 0000001c`8dbfa950 00007ffd`b1dd28a0     Qt5Widgets!QT::QApplicationPrivate::notify_helper+0xfd
19 0000001c`8dbfa980 00007ff7`8088aead     Qt5Widgets!QT::QApplication::notify+0x750
1a 0000001c`8dbfaec0 00007ff7`8092d5a5     ida64_exe+0x11aead
1b 0000001c`8dbfaf20 00007ffd`b1302f5b     ida64_exe+0x1bd5a5
1c 0000001c`8dbfb0b0 00007ffd`b1dd5ba5     Qt5Core!QT::QCoreApplication::notifyInternal2+0xbb
1d 0000001c`8dbfb120 00007ffd`b1e20252     Qt5Widgets!QT::QApplicationPrivate::sendMouseEvent+0x3c5
1e 0000001c`8dbfb1e0 00007ffd`b1e1e1ee     Qt5Widgets!QT::QSizePolicy::QSizePolicy+0x2d22
1f 0000001c`8dbfb570 00007ffd`b1dd4851     Qt5Widgets!QT::QSizePolicy::QSizePolicy+0xcbe
20 0000001c`8dbfb640 00007ffd`b1dd3a03     Qt5Widgets!QT::QApplicationPrivate::notify_helper+0x111
21 0000001c`8dbfb670 00007ff7`8088aead     Qt5Widgets!QT::QApplication::notify+0x18b3
22 0000001c`8dbfbbb0 00007ff7`8092d5a5     ida64_exe+0x11aead
23 0000001c`8dbfbc10 00007ffd`b1302f5b     ida64_exe+0x1bd5a5
24 0000001c`8dbfbda0 00007ffd`b1764cc5     Qt5Core!QT::QCoreApplication::notifyInternal2+0xbb
25 0000001c`8dbfbe10 00007ffd`b174ecd0     Qt5Gui!QT::QGuiApplicationPrivate::processMouseEvent+0xef5
26 0000001c`8dbfc2c0 00007ffd`b134bb2a     Qt5Gui!QT::QWindowSystemInterface::sendWindowSystemEvents+0x90
27 0000001c`8dbfc2f0 00007ffd`b544d9f9     Qt5Core!QT::QEventDispatcherWin32::processEvents+0x6a
28 0000001c`8dbff410 00007ffd`b12ff3af     qwindows!qt_plugin_query_metadata+0x1f99
29 0000001c`8dbff440 00007ffd`b1301f25     Qt5Core!QT::QEventLoop::exec+0x1bf
2a 0000001c`8dbff4a0 00007ff7`8088e88c     Qt5Core!QT::QCoreApplication::exec+0x155
2b 0000001c`8dbff500 00007ff7`8088e8bc     ida64_exe+0x11e88c
2c 0000001c`8dbff540 00007ff7`8088ff0a     ida64_exe+0x11e8bc
2d 0000001c`8dbff580 00007ff7`8088ffc6     ida64_exe+0x11ff0a
2e 0000001c`8dbffac0 00007ff7`809adbea     ida64_exe+0x11ffc6
2f 0000001c`8dbffb20 00007ffd`f4dc7604     ida64_exe+0x23dbea
30 0000001c`8dbffb60 00007ffd`f4f026a1     KERNEL32!BaseThreadInitThunk+0x14
31 0000001c`8dbffb90 00000000`00000000     ntdll!RtlUserThreadStart+0x21


```

看到确实卡在 hexarm64.dll 里了。

跑几步看看具体是哪个循环卡住了：

```
0:000> pc
hexarm64+0xbcc6c:
00000000`6eadcc6c e83f1f0000      call    hexarm64+0xbebb0 (00000000`6eadebb0)
0:000> pc
hexarm64+0xbcc6c:
00000000`6eadcc6c e83f1f0000      call    hexarm64+0xbebb0 (00000000`6eadebb0)
0:000> pc
hexarm64+0xbcc6c:
00000000`6eadcc6c e83f1f0000      call    hexarm64+0xbebb0 (00000000`6eadebb0)
0:000> pc
hexarm64+0xbcc6c:
00000000`6eadcc6c e83f1f0000      call    hexarm64+0xbebb0 (00000000`6eadebb0)


```

看看这个调用是在做什么：

```
char __fastcall sub_170BEBB0(__int64 a1, __int64 a2)
{
  __int64 v2; // rbp
  char v5; // r14
  unsigned __int64 v6; // rsi
  __int64 v7; // rcx

  v2 = *(_QWORD *)(a2 + 24);
  if ( *(_BYTE *)(v2 + 13) )
    return 0;
  v5 = 0;
  v6 = *(int *)(*(_QWORD *)(*(_QWORD *)(a1 + 16) + 8i64 * *(_QWORD *)(a1 + 24) - 8) + 8i64);
  if ( ((unsigned int)(v6 - 19) <= 2 || (_DWORD)v6 == 50)
    && (!(unsigned __int8)is_numop(*(unsigned int *)(v2 + 8), (unsigned int)*(char *)(v2 + 12))
     || (unsigned int)get_radix(*(unsigned int *)(v2 + 8), (unsigned int)*(char *)(v2 + 12)) != 16) )
  {
    v5 = 1;
    *(_DWORD *)(*(_QWORD *)(a2 + 24) + 8i64) = 0x1100000;
  }
  if ( (unsigned int)v6 > 0x3A || (v7 = 0x4003FFFFFFEFFFEi64, !_bittest64(&v7, v6)) )
  {
    if ( !**(_QWORD **)(a2 + 24) && (unsigned int)off_172AF1F0(*(unsigned int *)(a2 + 48), 3i64) == 2 )
    {
      sub_170B91F0(a2, *(_QWORD *)(a1 + 48), 1i64, 0i64);
      if ( !(unsigned __int8)sub_170BDE30(a1, *(_QWORD *)(*(_QWORD *)(a1 + 48) + 48i64)) )
        return 1;
      *(_DWORD *)(a1 + 8) |= 8u;
      return 1;
    }
  }
  if ( v5 )
    return 1;
  return 0;
}


```

注意到其中的`is_numop`和`get_radix`。

这个函数的返回值 al 一直是 1，尝试直接修改为 0，发现还是在死循环，看来是一些内部状态的修改不符合新版本要求？仔细跟一下这个函数，发现 is_numop 总是返回 0。

对比新旧版本 hexrays.dll，发现也有类似的代码，注意到新版本多了一次函数调用：

```
char __fastcall sub_17123890(__int64 a1, __int64 a2)
{
  __int64 v2; // rbp
  char v5; // r14
  unsigned __int64 v6; // rsi
  __int64 v7; // rcx

  v2 = *(_QWORD *)(a2 + 16);
  if ( *(_BYTE *)(v2 + 13) )
    return 0;
  v5 = 0;
  v6 = *(int *)(*(_QWORD *)(*(_QWORD *)(a1 + 16) + 8i64 * *(_QWORD *)(a1 + 24) - 8) + 4i64);
  if ( ((unsigned int)(v6 - 19) <= 2 || (_DWORD)v6 == 50)
    && (!(unsigned __int8)is_numop(*(_QWORD *)(v2 + 40), (unsigned int)*(char *)(v2 + 12))
     || (unsigned int)get_radix(*(_QWORD *)(v2 + 40), (unsigned int)*(char *)(v2 + 12)) != 16) )
  {
    sub_171C6B60(*(_QWORD *)(a2 + 16) + 8i64, 0x11111101100000i64);
    v5 = 1;
  }
  if ( (unsigned int)v6 > 0x3A || (v7 = 0x4003FFFFFFEFFFEi64, !_bittest64(&v7, v6)) )
  {
    if ( !**(_QWORD **)(a2 + 16) && (unsigned int)off_172A42C0(*(unsigned int *)(a2 + 40), 3i64) == 2 )
    {
      sub_1711DE70(a2, *(_QWORD *)(a1 + 48), 1i64, 0i64);
      if ( !(unsigned __int8)sub_17122C40(a1, *(_QWORD *)(*(_QWORD *)(a1 + 48) + 48i64)) )
        return 1;
      *(_DWORD *)(a1 + 8) |= 8u;
      return 1;
    }
  }
  if ( v5 )
    return 1;
  return 0;
}


```

尝试用断点命令直接修改`is_numop`的返回值：

```
0:000> p
hexarm64+0xbec0b:
00000000`6eadec0b 84c0            test    al,al
0:000> r rax
rax=0000000000000000

0:011> bp 00000000`6eadec0b "r rax=1;.echo 1;g"
breakpoint 0 redefined
0:011> g
1
1
1
1
1
1
1
1
1


```

断点触发多次之后，ida 终于正常了。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Juurxo786NA14QXH0pOWNjibHvqmKL6tLp20yb7koA8XqFp9NIASpOkR13nrBkhqqVyaXxtzratm6orQVdDgFQw/640?wx_fmt=png&from=appmsg)

通过对比新旧 hexrays 或者 hexx64 插件，可以大致猜出应该是新版本要求的某些对象的保存状态相关的成员变量没有在旧版本的这个函数中设置？也不是非常确定，如果之前逆向过 IDA 应该能有一个比较准确的答案。

命令断点终究是不方便，还是要考虑静态 patch 的方案。我随便找了个`is_numop0`的 API 来替换这个`is_numop`的导入表项，似乎也能达到类似效果。

但`is_numop`不是只在这一处使用，而且这么修改也不知道是否会对其他反编译过程中的状态有影响？只能说目前这么改下来还没观察到明显的副作用。

另外，微博网友沈沉舟昨天发了微博 [2] 来提醒大家使用这份 IDA 要谨慎点。

总而言之，还是尽量购买正版软件吧。

[1]

ida_dll_shim: _https://github.com/x0rloser/ida_dll_shim_

[2]

沈沉舟微博: _https://weibo.com/1273725432/Nuv0d7nbD_