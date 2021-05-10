> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/FSRIEr9pgyXSImjfUD3t_Q)

前言
--

    由于壳都需要通过 Classloader 来加载 App 本身的 Dex 文件, 一般都是通过 DexClassLoader 来加载, 安卓 8.0 以后引入了 InMemoryDexClassLoader 从内存中加载 dex 文件.

整体壳加壳方案
-------

    由于 DexClassloader 加载的 Dex 不能运行 Activity, 所以要通过 ClassLoader 的双亲委派机制来修正 ClassLoader, 从而使 App 的 Activity 可以运行起来

```
 1. 替换当前android.app.LoadedApk的mClassLoader为需要加载的DexClassLoader,
 并把父节点指向PathClassLoader.(常用方案)
 2. 把加载的DexClassLoader的父节点指向BootClassLoader
 然后将PathClassLoader的父节点替换成DexClassLoader.(有兼容性问题)
 3. 找到当前的PathClassLoader的Elements和DexClassLoader的Elements进行合并(也适用于热修复)
等等...

```

整体壳脱壳方案
-------

    从加壳角度出发, 研究 DexClassLoader 和 InMemoryDexClassLoader 的源码所经之处, 找到最后相同的点即为一个不错的脱壳点

    存在 DexFile 或者可以间接拿到 DexFile 的地方都可以是脱壳点.

    本文分析 DexClassLoader 和 InMemoryDexClassLoader 构造函数源码所经之处, 找到两个函数相同经过的函数即为一个不错的脱壳点.

### DexClassLoader

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECibSzD27d1KMJdAC1C32HqTyYJKLNbK2bc4TibYrGFfiaV4XrT6ia1XlH5Q/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECrFh4DTtHNokasY53cLERqBuHoMuKLJJ3vLtnJKEAh6fruOibsT9SpPQ/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECqNcmyOCOhjwnRDBOlEb3MK6OttLzGZfEdDrMUbKeFFYteasz6mJiaUw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECsMzC8Yo2adG30OALCFvaiciaCErYztNNO8WwmWbMX2yBtT9QmeFRMDyg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECuibwwdl1zuWNgoZT205kGbRcGmp24SMxo2RANP3mBn1Nlw5jJhBT4tA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECVDf5SDrGRIdYxHJ8AcUibnRwWZibLUhQbxibaiatkra2EtZUw0gmtib25JA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECVDf5SDrGRIdYxHJ8AcUibnRwWZibLUhQbxibaiatkra2EtZUw0gmtib25JA/640?wx_fmt=png)

分别进入看一下, 以下进入 Native 层 API

/art/runtime/native/dalvik_system_DexFile.cc

1.createCookieWithDirectBuffer

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECaHbfuqgc46aXPsOczzHugolLOQ92sTJoBLgJXtQtIEpE8C8iaIMElvw/640?wx_fmt=png)

2.createCookieWithArray

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECHE7zrxwn46uUm61SUPJt2PwD7ktqUAaDsyxVP319qeawXFZugA2xGQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECPZ5ictdz8QvcvT0VRpo9hjR5vPnmWmB8hxFRjvC13zDptpJogrMf5hA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECm8KBLVKrYCnJamiadFSKUGGFOOqVOWquTWL3XgfDqLdBm6T3zZcF4gg/640?wx_fmt=png)

/art/runtime/dex_file.cc

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFEClqnENuWqvkFWxguzDahtn4pGQJ3icFuf3oC6wcicEJxwg6Lp4dVTDYYg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECettJGGmjtTicyHeicfXLGQ0yyiaSJOPZxAgYEtPWyozLp2mNHX6ENtFcg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECyhWqHmQuyoQmObAJfia00UhxjIKsoFFLmvJs195TxlKqYibfKl690Tfw/640?wx_fmt=png)

运行完构造函数后 Dex 就加载进内存了

脱壳点:

DexFile::Open -> DexFile::OpenCommon -> DexFile::DexFile

### InMemoryDexClassLoader

不会经过 dex2oat

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECL9rWBTuQzIPnNIzZPliaCqIoDTEYaXuAwY4YAkSGueFaF8Wws8xf72g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECpMibS8Ljh4GguhFIpjF0V6jMibl5FMyNE8ibkuyEo10CWiaESSEUq4YOzA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECaCumniadNuicDRlukPKeqF0rqP8gmG0BnyhE35M0Fok2Snr9ic2IROQnw/640?wx_fmt=png)

后续流程同 DexClassLoader

流程图

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFEC1CbndXr3BKhKdfpDokAQP1eCVIRuxuhNXBH2GTMEroT33HN3VziaJ3w/640?wx_fmt=png)

接下来进入 Native 层

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECRe6vNx7UNkRH7n4GHzl5LuMiaHibZAfzylw5zJeQZiczIiaahF64tPcibfw/640?wx_fmt=png)

总结:

DexFile::DexFile 是一个不错的脱壳点, 其中入参包括 Dex 的起始地址和大小, 直接可以 Dump 下来保存为 dex 文件.(在 Android7.1 和 Android8 测试过 其他 Android 版本可以自行实验修改) 需要给 App 存储卡权限 或者 自己修改存储路径到 app 自身的路径.

Frida 脚本

```
var savepath = "/sdcard"
function FindArtAddr() {
    var symbols = Process.getModuleByName("libart.so").enumerateSymbols();
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
      //_ZN3art7DexFileC2EPKhmRKNSt3__112basic_stringIcNS3_11char_traitsIcEENS3_9allocatorIcEEEEjPKNS_10OatDexFileE
        if (symbol.name.indexOf("DexFileC2") >= 0
            && symbol.name.indexOf("OatDexFileE") >= 0
        ) {
            console.log(JSON.stringify(symbol))
            dexfile_dexfile_addr = symbol.address
        }
    }
    hook_DexFile_DexFile()
}
function hook_DexFile_DexFile() {
    if (dexfile_dexfile_addr) {
        console.log("dexfile_dexfile_addr",dexfile_dexfile_addr)
        Interceptor.attach(dexfile_dexfile_addr, {
            onEnter: function (args) {
                this.base = ptr(args[1])
                this.size = parseInt(args[2], 16)
                //var size = ptr(parseInt(base,16) + 0x20).readInt() // 通过dex格式来计算出size
            }, onLeave: function (retval) {
                var name = "dexfile_dexfile_" + this.size + ".dex"
                var path = savepath + "/" + name
                var dex_file = new File(path, "wb")
                //Memory.protect(base,size,"rwx")
                dex_file.write(Memory.readByteArray(this.base, this.size))
                dex_file.flush();
                dex_file.close();
                console.log("dexfile::dexfile dump over path -> ", path)
            }
        })
    }
}
}
FindArtAddr()

```

优点: 只要通过 DexClassLoader 或者 InMemoryClassLoader 都可以 dump 下 dex, 可以对抗整体壳.

缺点: dump 时机太早, 没有任何类进行加载, 无法对抗抽取壳

    拿了一个抽取壳的 app 做测试, 虽然 dump 出了很多 dex 但是其中的指令部分都是 nop, 还没达到真正的脱壳.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECnB4G3Vx6uiagYiaFMZcufdMBaN0SxYr5vE8jKtqjZWXlaGtAic7IgVozg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpS1mOllQ3KSBURTdsvGQFECnyekzjVFdEaKlrRTmw35CtzWkIyYibUfXc6MkyVmUdnaRMoQfjN4kkg/640?wx_fmt=png)

    感谢寒冰老师带来精彩的课程! 从课程和源码分析中学习到了很多

    下一篇文章介绍 FridaFart 中 hook 脚本的细节原理和 fdex2 脚本如何移植到安卓 7 以上的版本并使用 Frida 实现, 以上都可以一定程度上去对抗抽取壳.