> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283616.htm)

> [原创]React Native Hermes 逆向

最近没什么事，找了个看视频 App，发现又是广告又是视频植入的，烦不胜烦，想着花点时间去个广告捣鼓捣鼓。本文只为了技术分享讨论，切勿使用技术进行非法用途。  
apk 拿过来先拖动到 jadx 查看  
![](https://bbs.kanxue.com/upload/tmp/973070_JCZW28CJGR2RAWD.webp)  
loadAd 大概率就是加载广告的地方，点到这个类中往下滑，发现有 show 方法，通过修改改对应 dex 的 smail 将 show 改为空，改好后将 smail 文件夹重新编译为 dex，在放到 jadx 中验证是否错误。  
![](https://bbs.kanxue.com/upload/attach/202409/973070_XW83KT6D67GQHS6.webp)  
上面这个是 splash，然后依次将插屏，banner 都像这样修改掉。  
![](https://bbs.kanxue.com/upload/attach/202409/973070_CKJS2DQNSG6ETDR.webp)  
这都是基础的修改 smail，不在本文中详细描述。  
React native 的代码打包都都放在 assets 下的 index.android.bundle 中，正常情况会将代码压缩，混淆，但是我这个 apk 中的 index.android.bundle 打开并未显示 js 代码，拖到 notepad++ 中发现  
![](https://bbs.kanxue.com/upload/attach/202409/973070_6EQP4EASCFACVDE.webp)  
乱码显示，摸索了一下 java 代码发现  
![](https://bbs.kanxue.com/upload/attach/202409/973070_9DPRAQ4JTNTGB86.webp)  
这里比较可疑，应该是乱码的源头，然后搜索关键字 Hermes，发现这是一个对于 React native 的 JavaScript 引擎，将 js 代码都编译为 bytecode，Hermes 引擎会优化 React native 应用的启动速度和文件大小。  
![](https://bbs.kanxue.com/upload/attach/202409/973070_UNR9EUCR3ZVKPEP.webp)  
本来想使用 frida 大法，将 assets 读取文件内容，执行 Js 引擎时将内容直接打印出来，无奈，找了半天源码没发现好的切入点，发现一个开源的反编译工具，hbctool，官方也有 hbcdump 提供反编译工具，使用 hbcdump 执行反编译提示

```
Error: fail to deserializing bytecode: Wrong bytecode version. Expected 96 but got 90
```

看上去是我的 index.android.bundle 样本的 hbc 版本是 90，我下载的最新版本 hbcdump 为 96，找到 hbcdump 90 的工具后执行

```
hbcdump.exe index.android.bundle -mode=hbc -out=myout.js
disassemble
```

在当前目录将会看到  
![](https://bbs.kanxue.com/upload/attach/202409/973070_GNFHYSURKN5KSUF.webp)  
反编译的结果，但是但是，还没这么快结束。由于我的目的是需要去广告，解锁会员功能，所以修改 js 代码后还需要回编译，但是官方的这个工具找了半天也没发现有什么文档说明怎么编译回去。无奈切换为另外一个作者的 hbctool 工具  
**[https://github.com/bongtrop/hbctool](https://github.com/bongtrop/hbctool)**  
这个工具在 pull request 中有 90 和 94 的反编译源码提供，感谢每一个开源的作者。在文章末尾我也会将我使用到的代码上传提供下载使用。  
![](https://bbs.kanxue.com/upload/attach/202409/973070_PX79P2WCJDGAYW8.webp)  
将源码下载，在 pycharm 中打开，添加 hbc90 目录，  
![](https://bbs.kanxue.com/upload/attach/202409/973070_8K2DG45YWPP2RET.webp)  
在__init__.py 中添加 90 版本的处理，  
![](https://bbs.kanxue.com/upload/attach/202409/973070_AJ6VR3TZ38HR5QP.webp)  
执行 disasm 将会反编译文件，执行 asm 将会把反编译修改的的文件夹编译回 index.android.bundle  
反编译后的文件夹内容  
![](https://bbs.kanxue.com/upload/attach/202409/973070_9FXFJFJ9S9CB6GE.webp)  
instruction.hasm 便是源码，string.json 是字符表，metadata.json 看上去是文件的描述符和完整性验证使用的文件，没有仔细研究这个文件。  
按照我的目的我应该去修改 instruction.hasm，打开 instruction.hasm 查看。  
![](https://bbs.kanxue.com/upload/attach/202409/973070_REAFZHWFCV6KQWN.webp)  
不认识这种代码，后来搜索到相关文章有描述操作符之类的说明，现在已经忘记了。  
仔细看代码，和汇编有些相似，但是看上去比汇编简单很多，都是左边一个指令右边操作数，例如

```
LoadConstTrue           Reg8:22 将一个8位的寄存器22设置为true
Mov                     Reg8:22, Reg8:25  将25寄存器的值赋值給寄存器22
LoadConstString         Reg8:39, UInt16:16781  加载一个字符到39寄存器，字符id为16781，查看string.json可以找到这个字符的值。
 {
        "id": 16781,
        "isUTF16": false,
        "value": "contain"
    }
LoadConstTrue           Reg8:4 将寄存器4设为true
LoadConstFalse          Reg8:4 将寄存器4设为false
```

接下来只要找到 Vip 字样相关的地方，修改为 true, 将反编译后的文件夹重新编译为 index.android.bundle，替换 assets 下面的 index.android.bundle，使用 apktool 重新打包就完成了。

[[课程]FART 脱壳王！加量不加价！FART 作者讲授！](https://bbs.kanxue.com/thread-281194.htm)

最后于 15 小时前 被一颗小草编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)