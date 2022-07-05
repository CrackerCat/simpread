> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273545.htm)

> [原创]Flutter APP 逆向实践 - 初级篇

Flutter APP 逆向实践 - 初级篇
======================

0. 前言
-----

很长一段时间对于 Flutter 的 app 逆向都比较头疼，它不像纯 Java app 那样可以使用 jadx-gui 看源码，也不能像原生 native 那样可以用 ida 看字符串交叉引用。

 

分析 flutter 的时候，没有交叉引用，那真是想它他拖进回收站。  
![](https://bbs.pediy.com/upload/attach/202207/548459_WTVNBJ24KRKMF79.png)

 

最近又遇到了几个 flutter 开发的 app，对它进行了一些研究，总结了以下的分析流程。

 

本文以 iOS app 为例子作为讲解，Android 的 flutter app 和 iOS 的 flutter app 分析方法类似。

1. 简述流程
-------

抓包，使用 frida-ios-dump 进行砸壳后，得到目标 app 的 ipa 文件，reFlutter 对 ipa 重打包后得到 release.RE.ipa，使用 ios-app-signer 对 release.RE.ipa 进行自签名，安装到手机上并运行，得到 dump.dart 文件，根据 dump.dart 里的类名、函数名、函数相对_kDartIsolateSnapshotInstructions 的偏移，配合 ida 静态分析 + frida 动态分析，分析出加密算法。

2. 前期准备
-------

#### 2.1 抓包，确定需要分析的 signsafe 算法

![](https://bbs.pediy.com/upload/attach/202207/548459_XJ4XR8MRRNSG8HU.png)

#### 2.2 frida-ios-dump 砸壳

python dump.py app 名

 

![](https://bbs.pediy.com/upload/attach/202207/548459_NN96G9PHQSC7X8S.png)

 

打开 ipa 能看到 Frameworks 目录下有 **App.framework, Flutter.framework** 两个框架，表示这个 app 是 flutter 开发的。  
![](https://bbs.pediy.com/upload/attach/202207/548459_WF327KK7MKPYWGS.png)

#### 2.3 reFlutter 重打包

reflutter XXXX.ipa

 

![](https://bbs.pediy.com/upload/attach/202207/548459_S9YD6UJDWKHX7DS.png)

#### 2.4 ios-app-signer 重签名

如果没有 iOS 付费开发者账号，使用免费开发者账号的话，需要先使用 xcode 新建一个 demo 工程，并运行在 iOS 上，然后才能使用 ios-app-signer 重签名

 

![](https://bbs.pediy.com/upload/attach/202207/548459_G6MHTH8ZXYAS9A6.png)

#### 2.5 xcode 安装重打包的 ipa

xcode -> Window -> Devices and Simulators -> INSTALLED APPS，有个 + 号，可以把上一步重签名的 ipa 安装到手机上

#### 2.6 查看 dump.dart 日志

Devices and Simulators 界面有个 Open Console 按钮，可以打开控制台，查看 dump.dart 的路径

 

![](https://bbs.pediy.com/upload/attach/202207/548459_GEJ26C6E4FA8GQ8.png)

 

reFlutter dump file: /private/var/mobile/Containers/Data/Application/F4F2810A-C863-4732-B871-480BFD1C101B/Documents/dump.dart

 

使用 scp 命令把 dump.dart 拷贝到本地，可以看到里面包含了**类名，函数名，相对_kDartIsolateSnapshotInstructions 的偏移**

 

![](https://bbs.pediy.com/upload/attach/202207/548459_W8JWH4TCV2Y7DZE.png)

#### 2.7 ida 加载 App.framework 下的 App 文件

使用 ida 加载 App 文件，看 Exports 窗口，有如下几个导出符号，而且**运气比较好**，很多函数都有符号，Precompiled_xxx

 

![](https://bbs.pediy.com/upload/attach/202207/548459_XAGDZPB8USUMMZF.png)

3. signsafe 详细分析流程
------------------

去 dump.dart 搜索 **signsafe 字符串**，没有搜索到

 

根据前面抓包 signsafe 的内容 10a7d81a264e79640ce3433f1b03d992，这是一个 32 位的字符串，猜的可能是 md5 相关的算法

 

于是去 dump.dart 里搜索 **md5 字符串**, 搜索到 3 处 md5 相关的类

```
Library:'package:pointycastle/digests/md5.dart' Class: MD5Digest extends MD4FamilyDigest implements Type: Digest {
  FactoryConfig factoryConfig = sentinel ;
  Function 'get:algorithmName': getter const. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000cc8a1c
       }
  Function 'get:digestSize': getter const. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000ccd5f4
       }
  Function 'MD5Digest.': constructor. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000002828
       }
  Function 'resetState':. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000cc4d98
       }
  Function 'processBlock':. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000ce4ea0
       }
      }
```

```
Library:'package:crypto/src/md5.dart' Class: _MD5Sink@404143612 extends HashSink {
  Function '_MD5Sink@404143612.': constructor. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x000000000026309c
       }
  Function 'updateHash':. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000c649f8
       }
      }
```

```
Library:'package:crypto/src/md5.dart' Class: _MD5@404143612 extends Hash {
  Function 'startChunkedConversion':. String: null {
               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000c4f2d4
       }
      }
```

对三个 md5 相关的函数，进行 hook,

```
function print_native_stack(addr, context) {
    return ('\r\n[' + addr + '] called from:\n' +
        Thread.backtrace(context.context, Backtracer.ACCURATE)
            .map(DebugSymbol.fromAddress).join('\n') + '\n');
}
function hook_md5() {
    let kDartIsolateSnapshotInstructions = Module.findExportByName("App", "kDartIsolateSnapshotInstructions")
    console.log(kDartIsolateSnapshotInstructions);
    let processBlock = kDartIsolateSnapshotInstructions.add(0x0000000000ce4ea0);
    let updateHash = kDartIsolateSnapshotInstructions.add(0x0000000000c649f8);
    let startChunkedConversion = kDartIsolateSnapshotInstructions.add(0x0000000000c4f2d4);
    Interceptor.attach(processBlock, {
        onEnter(args) {
            console.log("processBlock:", print_native_stack("processBlock", this));
        }
    })
    Interceptor.attach(updateHash, {
        onEnter(args) {
            console.log("updateHash:", print_native_stack("updateHash", this));
        }
    })
    Interceptor.attach(startChunkedConversion, {
        onEnter(args) {
            console.log("startChunkedConversion:", print_native_stack("startChunkedConversion", this));
        }
    })
}
```

手机上操作一下，可以看到 **updateHash 被调用**

 

![](https://bbs.pediy.com/upload/attach/202207/548459_F97C8UNKGK5G6YF.png)

 

去 ida 中看看 **PrecompiledapiSign_7813 函数**

 

![](https://bbs.pediy.com/upload/attach/202207/548459_PBMU6NHAZ2SGBYY.png)

 

对上图画红框的 5 个函数进行 hook

```
function hook_addr(addr, name) {
    Interceptor.attach(addr, {
        onEnter(args) {
            this.log = []
            this.log.push(name + " onEnter:\r\n")
            for(let i = 0; i < 8; i++) {
                try {
                    this.log.push(hexdump(args[i]), "\r\n");  
                } catch (error) {
                    this.log.push((args[i]), "\r\n");
                }
            }
        }, onLeave(retval) {
            this.log.push(name + " onLeave:\r\n")
            try {
                this.log.push(hexdump(retval), "\r\n");  
            } catch (error) {
                this.log.push((retval), "\r\n");
            }
            this.log.push("=======================")
            console.log(this.log);
        }
    })
}
 
function hook_apisign() {
    let Precompiled_Hmac_Hmac__7814 = DebugSymbol.fromName("Precompiled_Hmac_Hmac__7814")
    let Precompiled_Hmac_convert_37042 = DebugSymbol.fromName("Precompiled_Hmac_convert_37042")
    let Precompiled____base64Encode_5267 = DebugSymbol.fromName("Precompiled____base64Encode_5267")
    let Precompiled_Hash_convert_37041 = DebugSymbol.fromName("Precompiled_Hash_convert_37041")
    let Precompiled_Digest_toString_34431 = DebugSymbol.fromName("Precompiled_Digest_toString_34431")
    hook_addr(Precompiled_Hmac_Hmac__7814.address, Precompiled_Hmac_Hmac__7814.name);
    hook_addr(Precompiled_Hmac_convert_37042.address, Precompiled_Hmac_convert_37042.name);
    hook_addr(Precompiled____base64Encode_5267.address, Precompiled____base64Encode_5267.name);
    hook_addr(Precompiled_Hash_convert_37041.address, Precompiled_Hash_convert_37041.name);
    hook_addr(Precompiled_Digest_toString_34431.address, Precompiled_Digest_toString_34431.name);
}
```

得到以下日志，省略了部分无关的日志，以...... 代替

```
Precompiled_Hmac_Hmac__7814 onEnter:
,            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
10c0afba9  03 6d 00 00 00 00 00 c0 fb 0a 0c 01 00 00 00 00  .m..............
10c0afbb9  00 00 00 14 00 00 00 44 32 33 41 42 43 40 23 35  .......D23ABC@#5
10c0afbc9  36 00 00 00 00 00 00 00 00 00 00 00 00 00 00 04  6...............
 
。。。。。。
 
Precompiled_Hmac_convert_37042 onEnter:
,            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
10c0afd79  07 6d 00 00 00 00 00 90 fd 0a 0c 01 00 00 00 00  .m..............
10c0afd89  00 00 00 9a 00 00 00 61 70 69 2e 78 78 78 2e 63  .......api.xxx.c
10c0afd99  6e 2f 78 78 78 78 2f 61 70 69 3f 70 68 6f 6e 65  n/xxxx/api?phone
10c0afda9  3d 31 33 38 30 30 31 33 38 30 30 30 26 74 78 79  =13800138000&txy
10c0afdb9  7a 6d 3d 26 75 72 69 3d 61 70 69 78 78 78 78 2f  zm=&uri=apixxxx/
10c0afdc9  61 70 69 2f 75 73 65 72 2f 73 65 6e 64 73 6d 73  api/user/sendsms
10c0afdd9  63 6f 64 65 00 00 00 00 00 00 00 00 00 00 00 04  code............
 
。。。。。。
 
Precompiled____base64Encode_5267 onEnter:
,            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
10c0b0789  03 6d 00 00 00 00 00 a0 07 0b 0c 01 00 00 00 00  .m..............
10c0b0799  00 00 00 28 00 00 00 f6 41 84 fc 8b 76 4a f3 03  ...(....A...vJ..
10c0b07a9  a0 fe d9 2f e8 5d 85 0f ee 6d b2 00 00 00 00 04  .../.]...m......
。。。。。。
,Precompiled____base64Encode_5267 onLeave:
,            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
10c0b0859  03 55 00 00 00 00 00 38 00 00 00 00 00 00 00 39  .U.....8.......9
10c0b0869  6b 47 45 2f 49 74 32 53 76 4d 44 6f 50 37 5a 4c  kGE/It2SvMDoP7ZL
10c0b0879  2b 68 64 68 51 2f 75 62 62 49 3d 00 00 00 00 00  +hdhQ/ubbI=.....
 
。。。。。。
,Precompiled_Digest_toString_34431 onLeave:
,            0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
10c0b0d19  03 55 00 00 00 00 00 40 00 00 00 00 00 00 00 31  .U.....@.......1
10c0b0d29  30 61 37 64 38 31 61 32 36 34 65 37 39 36 34 30  0a7d81a264e79640
10c0b0d39  63 65 33 34 33 33 66 31 62 30 33 64 39 39 32 00  ce3433f1b03d992.
```

由于结果有比较明显特征，从日志可以猜出大概算法，有 md5(32 位字符串或 16 个字节), base64(ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/= 结果在这个字符串里), hmac_sha1（40 位字符串或 20 字节）

```
md5("9kGE/It2SvMDoP7ZL+hdhQ/ubbI=") -> 10a7d81a264e79640ce3433f1b03d992
```

```
00000000  f6 41 84 fc 8b 76 4a f3 03 a0 fe d9 2f e8 5d 85  |öA.ü.vJó. þÙ/è].|
00000010  0f ee 6d b2                                      |.îm²|
 
进行 base64 可以得到
9kGE/It2SvMDoP7ZL+hdhQ/ubbI=
```

```
hmac_sha1("D23ABC@#56", "api.xxx.cn/xxxx/api?phone=13800138000&txyzm=&uri=apixxxx/api/user/sendsmscode")    //已打码xxxx
```

![](https://bbs.pediy.com/upload/attach/202207/548459_DKZ52KG5Q2WY9F6.png)

4. 总结
-----

分析以上的例子，因为运气好，dump.dart 里面有很多业务相关的函数符号，所以降低了很大的难度。

 

对抗以上的逆向方法也很简单，flutter 官方就有解决方案。

 

加上 **--obfuscate --split-debug-info** 两个参数就能抹去这些类名和函数名  
![](https://bbs.pediy.com/upload/attach/202207/548459_B2PB5S8YNP94EEA.png)

参考工具
----

https://github.com/AloneMonkey/frida-ios-dump

 

https://github.com/Impact-I/reFlutter

 

https://github.com/DanTheMan827/ios-app-signer

 

https://frida.re

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

[#逆向分析](forum-166-1-189.htm)