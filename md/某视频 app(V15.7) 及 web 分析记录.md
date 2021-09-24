> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269480.htm)

> 某视频 app(V15.7) 及 web 分析记录

某视频 app(V15.7) 及 web 分析记录

   初衷是刷抖音太多，发现不能在点赞过的视频列表中直接搜索，就想自己实现下，把这个过程做了下记录，当学习笔记了，纯技术交流用。

**一、****软硬件环境：**

**抖音 android (V15.7****，应用宝渠道)**

**抖音 web (V16.1)**

IDA 7.5

Frida 14.2.2

Gda3.86

JEB

jadx-gui

unidbg

LineageOs 17.1 (android 10)

小米 8

**二、****流水账 (****直接按时间顺序，有坑写坑)**

**1.** **App X-Gorgon** **算法定位**

**开始也是直接网上搜了下，有下面这个：**

[**https://blog.csdn.net/weixin_48271161/article/details/108544446**](https://blog.csdn.net/weixin_48271161/article/details/108544446)

![](https://bbs.pediy.com/upload/attach/202109/44250_M5ZJPHX8WVFHQTU.jpg)

在这个 V15.7 版本里面没找到相关的 so（后来发现我看的这个版本对应的是 libmetasec_ml.so）。

还有其它一些，比如搜索 X-Gorgon，hook HashMap，发现对这个版本都不好使了，那就从头开始了，自己找这个加密关键点。

先上 frida，输出 jni 函数:

![](https://bbs.pediy.com/upload/attach/202109/44250_K8PXFWK5GQNT86Q.jpg)

现在回过头看，发现当时就输出了 libmetasec_ml.so jni 记录（虽然不是直接的加密函数），但是当时可能是这个记录太多了，竟然没注意到，估计当时看到了也没留意，因为开始也不知道这个 so 是做什么的。

看到这个 Cronet，搜了下：

Cronet 网络库系列 (一)：用例与原理实现详解

[https://segmentfault.com/a/1190000021095757](https://segmentfault.com/a/1190000021095757)

https://blog.csdn.net/u010983881/article/details/97770544/

【Android】移动端接入 Cronet 实践

根据登录 url 找到下面代码：

![](https://bbs.pediy.com/upload/attach/202109/44250_NVZXBEWZSTP2HM8.jpg)

然后根据调用关系翻代码，找 Cronet 相关调用, 浏览代码，翻到 com.ttnet.org.chromium.net.impl.CronetUrlRequest，也没

发现有设置 X-Gorgon 头的地方（其它 header 倒是有设置），中间也尝试通过 send 反找，调用线太长，还是决定换个方式来做。

直接下了个 cronet 代码：

[https://github.com/hanpfei/chromium-net](https://github.com/hanpfei/chromium-net)

然后就是对着代码，确定认为会相关的几个函数, 上 IDA 调试：

nativeCreateRequestAdapter

nativeAddRequestHeader

最后找到这个点

![](https://bbs.pediy.com/upload/attach/202109/44250_Y8TYTEM4FBVJSM5.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_RXZNR8MJXQFSVXP.jpg)

到这里就跟 libmetasec_ml.so 关联起来了。

知道了调用点，就想直接调用 libmetasec_ml.so 方便调试，写了个程序来加载这个 so，发现会异常，考虑到可能是上下文环境不全，就打算按正常流程来加载 so, 也调用 JNI_OnLoad:

void* hMetasecSo = dlopen("/data/local/tmp/libmetasec_ml.so", RTLD_LAZY);

    typedef jint(*FUN)(JavaVM* vm, void* res);

    FUN func_onload = (FUN)dlsym(hMetasecSo, "JNI_OnLoad");

这个调用是需要 JavaVM 参数的，就准备加载 libart 来调用 JNI_CreateJavaVM 创建 JavaVM，参考网上资料设置好参数：

    JavaVMInitArgs vm_args;

    JavaVMOption options[2];

    options[0].optionString = "-Djava.class.path=.";

    vm_args.version = JNI_VERSION_1_6;

    vm_args.options = options;

    vm_args.nOptions = 2;

    vm_args.ignoreUnrecognized = JNI_TRUE;

编译运行后，直接崩了，查看日志，提示没有设置 NoSigChain,

然后查看 android 源码，找到对应地方看了下，是在检查创建 VM 的选项参数，应该是没有 no-sig-chain 这个参数，网上搜了下，没有找到怎么设置这个参数的，然后根据代码，结合其它属性改了下代码，增加了一条：

options[1].optionString = "-Xno-sig-chain";

android 源码里面也跳过了这个相关的，编译运行，之前的错误就没有了，但是后面还是异常退出了，后来查了下，找到下面信息：

从 Android N 开始（SDK >= 24），通过 dlopen 打开系统私有库，或者 lib 库中依赖系统私有库，都会产生异常，甚至可能导致 app 崩溃。

应用可以调用 / vendor/etc/public.libraries.txt 和 / system/etc/public.libraries.txt 里面的所有 so 库，

所以往这个文件写入自己希望被调用的 so，这个库就变成共用的了，任意应用就可以找到这个 so 库了

试了下上面的方法，包括把 libart.so 及相关的库放到其它用户目录下，还是不行，考虑到本来就是想还原运行环境的，就想直接上 APK 吧，还能省去自己创建 VM，就写了个测试 APK 调用这个 so:

![](https://bbs.pediy.com/upload/attach/202109/44250_HVSNPGCSY9HYS2J.jpg)

编译好 apk 后导入到手机运行：

![](https://bbs.pediy.com/upload/attach/202109/44250_RURRY6VNJ7YJZVZ.jpg)

Didn't find class"com.bytedance.mobsec.metasec.ml.MS"

直接参考抖音补上对应的包路径：

![](https://bbs.pediy.com/upload/attach/202109/44250_8YW4H27EGW6THK3.jpg)

现在能运行了，但是调用 JNI_OnLoad 会异常，准备上 IDA 调试，看了下流程图：

![](https://bbs.pediy.com/upload/attach/202109/44250_CKZS9YG7XBG499C.jpg)

带了 llvm，看 so 文件尾部记录的是 Apple LLVM version 10.0.1 (clang-1001.0.46.3)。

跟了下，一些跳转都做了处理，不是很好分析，准备上 unidbg ([https://github.com/dqzg12300/unidbg_tools](https://github.com/dqzg12300/unidbg_tools)，可以用这个大佬整理的)，trace 代码下来分析。

![](https://bbs.pediy.com/upload/attach/202109/44250_M2XUJJW2RKZF2SW.jpg)

根据 trace 得到的指令流，发现有这种访问 "/proc/self/exe" 路径返回 -1 的情况，我是直接修改这个函数，直接返回 1 了，这个函数只是用来取 e_machine 字段的：

![](https://bbs.pediy.com/upload/attach/202109/44250_ZYJJ7R7Z6KXNR3N.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_67XXYEYRKD85VSU.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_UNQ79R762UCZ35E.jpg)

后面继续 trace，然后结合动态调试，发现有代码校验的地方：

![](https://bbs.pediy.com/upload/attach/202109/44250_939YES8HXU4XKTY.jpg)

上面都处理后，测试 apk 就可以正常跑完 JNI_OnLoad 了，后面就是主动调用加密函数测试了，直接在 jni 接口中调用 libmetasec_ml.so：

![](https://bbs.pediy.com/upload/attach/202109/44250_PW96Z8838SSPGYD.jpg)

下面插个分支，到这里的时候，在一个群里看到信息说抖音开了 web，直接去看了下.

**2.** **抖音 web 请求参数_signature 算法分析**

直接访问 [www.douyin.com](http://www.douyin.com/)，看访问参数多了个_signature，这种格式：

&_signature=_02B4Z6wo00901qf0GiQAAIDAwkLkeQfbXMKn9B6AAMkm74。

多拿几个比较，发现前面一段（_02B4Z6wo00901）是前缀，后面初步判断是 base64 格式.

直接网上搜了下，找到下面这篇：

[网络爬虫 - 今日头条_signature 参数逆向 (第一弹)_井蛙不可语于海的博客 -CSDN 博客_byted_acrawler](https://blog.csdn.net/qq_39802740/article/details/104911315)

[https://blog.csdn.net/qq_39802740/article/details/104911315](https://blog.csdn.net/qq_39802740/article/details/104911315)

主要加密算法在 acrawler.js，参考这个用 node 跑起来了，里面的实现是一个 js 虚拟机，关于虚拟机还找到这篇：

[StriveMario/jsvm: 给 " 某音 " 的 js 虚拟机写一个编译器 (github.com)](https://github.com/StriveMario/jsvm)

不过目前版本都看起来跟这个不同了，并且本地调试算法其实也用不上，但是还是可以学习下的。

![](https://bbs.pediy.com/upload/attach/202109/44250_J5MA9QM5KCC2VZU.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_M8C5PKZK4UAYVVE.jpg)

除了文章提到的，还有下面的几个参数要改下：

![](https://bbs.pediy.com/upload/attach/202109/44250_R4PHAGDGSS653KK.jpg)

然后就可以调用 global.byted_acrawler.init 及 global.byted_acrawler.sign 参数了（这个后面发现这样调用的算法流程跟浏览器跑的其实是不同的）.

用浏览器调试也是确定有 init ,sign 函数的:

![](https://bbs.pediy.com/upload/attach/202109/44250_ATG4W4FWF8G4QTU.jpg)

后面就是直接用 node 调试，因为原代码的格式都是这样的：

![](https://bbs.pediy.com/upload/attach/202109/44250_PRRJV4WM2XFUSD2.jpg)

都是一句话写完逻辑，不方便调试，先整理了下代码，比如这种

opCode = 3 & initCode;         //  initCode % 4

那 > 2 的分支就是 ==3 了，长的代码就分割下.

整理后也方便加日志，输出中间数据.

调试中把 vm 的基础操作整理出来 (vm_xor,vm_and 等等)，特别字符串连接的指令，可以加上日志，方便定位。

Vm 中会检查运行环境，包括下面这些：

domDetect

debuggerDetect

nodeDetect

phantomDetect

webdriverDetect

incognitoDetect

hookDetect

locationDetect

检查结果会参与加密，作为其中一段的因子。

当时看了几个签名数据，就有个疑问的，相同的数据，_signature 总是不同的，当时想到应该有个变量因子的，但是看 https 提交的参数没发现这个变量，那这样就是直接记录在_signature 中了。

直接 base64 解密排除前缀后的数据，类似这种：

<Buffer 7c 7e 62 a4 00 00 20 30 83 81 9d 5b 28 d0 87 e9 7c 76 e3 80 00 07 26 f8>

比较多个后也没发现明显的变量因子，猜到可能是时间戳，但是没找到哪个数据段是标识的时间，那就直接开始看流程了。

调试过程发现有 getTime 的调用，直接改为固定的，最后得到的_signature 就不会变了：

![](https://bbs.pediy.com/upload/attach/202109/44250_JU479AC7J7CU793.jpg)

那就是确定跟时间戳相关了，也就是服务器可以通过这个得到时间戳或者转换过的值，来验证客户端上报的_signature 是否正确了。

后面算法中，主要就是参数字符串（location,user_agent,param）的处理（xor,or,and 等）得到各个 hash 值，然后类似 base64 加密得到加密字符串（这里不是直接得到完整明文，再最后一次 base64 的方式）：

![](https://bbs.pediy.com/upload/attach/202109/44250_KJFW5YMGUB8JKAA.jpg)

处理完后，最后 2 个字符是附加的校验字符，是对前面数据得到一个 DWORD 值的低字节：

_02B4Z6wo00f014U9W.wAAIDB4IuloLYVYYOFPV9AAIGw

2260354722 '86ba46a2'

_02B4Z6wo00f014U9W.wAAIDB4IuloLYVYYOFPV9AAIGwa2

本以为搞完了，直接跟浏览器访问的一比较，悲剧了，竟然加密结果不同，直接上浏览器调试。

前面已经整理了代码的，浏览器访问的时候 js 是直接下载的，直接修改 DNS，把这个 js 定向到本地服务器:

127.0.0.1 sf1-ttcdn-tos.pstatp.com

这样浏览器访问的时候也是用的整理后的 js 了，有了前面的调试经历，这个也很快搞清楚了流程，

![](https://bbs.pediy.com/upload/attach/202109/44250_WJZ7WDQMW4KX8FS.jpg)

data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAQCAYAAABQrvyxAAACfElEQVRIS9WVO2hUURCGv20Eg1bapVC0sFNJJahFRPDRWFgoRokgQbAQLFJJdNFKFAVBNMROI1oYsBITsPDRpAgkNkmjEoggqIWNIILyywwMs3MxhevjwLLs3XPnzP+a0+L/XZuAE63U/wBwNz0bBWYAfcd1DrjYJfxrgHFgT9HLGeALUAIYAS6kl54A+lxNz3cAL7sEoCq7HdgVSCsBVMiPWrWszJ8EsBK4BjwEDgInrafRbKEIQJa5BawDViVrzQKHgAUrJOsdM3ZkrdP2n1sy203sPQC2eCOAW6NiX3U2JMuWCuTCKlYpIADDwBQgaV+ERpSXPgN/BLgHXAqgnCTlRxZ0dp+b7zMANb/TAPakbHQoMARMAB8BZ6/JQsqF9uw1dp7ayfJpL/AZeAR8AK4HABGwKxObVEC1HKhAVsOiQwFnQv66b0UO2/RRM2I8B1o5mA/M6aAbwHlgH3AWWJsAqLQOj6AyAA0TAdVzkVmtDgDR/xnAehtpss5kAKPfYqxtJ+hQHX4T2GzM5Wa11fd5g5UC3nRla7f2eA6xj1GpIB/rWzK/thDHO0FqSBkBup0AvAXumMcjgEWbJqrrFnS7us/dQhVQB+VuWaoAyL+PzQIRgCykMfbebKUGtgZl/EAFOjYXAWjPfmDalPqVAiJU2aruG9VtRwAxA1EBsf4MGATehFt5DvgE9AeDekDj2GxSQMNBt61Wk4VUT5mKI1v7SwWUAdlmm7HsFrpsDOgu0Cx2O30DTgFjDSH7XY+bMvDzIs0WWs6hq80i8r6HdznvdWVPBWAF8LXhNEl9xTLw15tXjxWAA8Bxu4DeAd+BjcBu4JVNEYX8n1g/AAC2uh6gEsDjAAAAAElFTkSuQmCC

![](https://bbs.pediy.com/upload/attach/202109/44250_2V3X6Y65TZ7K9DS.jpg)

多了对这个图片数据的参与，最后整理测试：

![](https://bbs.pediy.com/upload/attach/202109/44250_G4HXZQDJKMVQDH2.jpg)

整体来看，web 上虽然用了 JS 虚拟机，跟二进制的 VM 比起来还是弱些了，调试环境弄好后，执行流程就都比较清晰了。

![](https://bbs.pediy.com/upload/attach/202109/44250_YFS8UF4W8WF9MHQ.jpg)

这里 web 签名就基本完成了，继续 app 分析

**3.** **继续 APP 算法分析**

被上个分支中断了下，思路都断了，又重新熟悉了下，继续开始跟这个加密算法。

在测试 apk 中调用 libmetasec_ml.so 加密函数后，返回的是 NULL，调试后发现会从 native 调用 java 的情况:

![](https://bbs.pediy.com/upload/attach/202109/44250_SUF329MNSEWMYHU.jpg)

然后参考抖音补全需要的包，里面有热更新保护相关的代码，屏蔽掉，让测试工程能跑起来就行：

_//import com.bytedance.JProtect;  
//import com.bytedance.covode.number.Covode;  
//import com.meituan.robust.ChangeQuickRedirect;  
//import com.meituan.robust.PatchProxy;  
//import com.meituan.robust.PatchProxyResult;_

![](https://bbs.pediy.com/upload/attach/202109/44250_FK3YABSGYD4Q6AQ.jpg)

反正就是各种补，模拟全之前的调用环境。

看到一些检测 root 相关的字符串：

![](https://bbs.pediy.com/upload/attach/202109/44250_7C6PJMV89834YNR.jpg)

通过分析 trace 日志，过滤掉一些跳转计算流程后，发现可能的关键调用，用 unidbg 跑的时候返回是 null:

![](https://bbs.pediy.com/upload/attach/202109/44250_2Z3QY6D3CK7XSXN.jpg)

下面就是调试抖音进行验证了，修改对应指令为循环点，附加后循环处下断点：

![](https://bbs.pediy.com/upload/attach/202109/44250_K4RZN3Z9J24PFUN.jpg)

那就确定是下面函数返回的签名字符串了：

![](https://bbs.pediy.com/upload/attach/202109/44250_MVMBA9RSW5TS7GF.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_9BHASKH7JQQD8KT.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_PMX267A4YRWBFBS.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_5ZT8CBE9GJWERZJ.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_CTV84DM5KSNCKY3.jpg)

对于自己 APK 调用时候，可以直接设置首尾地址：

```
//修改so的起始地址
 *(unsigned  int *)(dwContextAddr+0x10)=0x100000;
 //修改结束地址
 *(unsigned  int *)(dwContextAddr+0x10)=0xFFFFFFFF;

```

跳过这个检查。

检查 trace 代码，发现有对代码指令的检查

![](https://bbs.pediy.com/upload/attach/202109/44250_W2GVHX6QSMRP9SA.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_PVYHZ37N8CYG46U.jpg)

根据调试流程，整理调用链，补全初始化调用：

![](https://bbs.pediy.com/upload/attach/202109/44250_PP5MZ5WYTZTV46C.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_GHC8UWTV88XQTGX.jpg)

测试 app 可以直接跑出结果了：

![](https://bbs.pediy.com/upload/attach/202109/44250_9Z76HGED7SXXTX6.jpg)

![](https://bbs.pediy.com/upload/attach/202109/44250_YYMCBRFFN92WE9A.jpg)

现在是自己写的程序可以跑了，后面是用 app 中提供签名服务，还是撸算法出来，都方便很多了。

搞完测试：

![](https://bbs.pediy.com/upload/attach/202109/44250_9D6WPXETCTW25HD.jpg)

  整体感觉流程的分析难度不如 wegame.

  这次分析过程中学习的技能点总结：

1.   开发测试 APK 模拟目标的调用环境.

2.   Unidbg 模拟调用 so 中任意地址

3.   RSA 验签过程，其它工作涉及的，之前只是使用 API（

//1. 明文计算 sha256

//2.RSA 解密密文（一般是 base64 解密后的字节流）

//3. 比较尾部的串是否跟 1 的一致（因为解密后的结果是包含这个 hash 串，不是等于）

）

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

[#逆向分析](forum-161-1-118.htm)