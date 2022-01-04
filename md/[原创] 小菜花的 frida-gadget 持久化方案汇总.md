> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268256.htm)

> [原创] 小菜花的 frida-gadget 持久化方案汇总

0x1：为什么要写这篇文章
-------------

    看到 frida-gadget 持久化的话题依旧在讨论，遂想记录下自己实践过的方案，第一次在看雪发文，不当之处大手子请轻喷 

                         ![](https://bbs.pediy.com/upload/attach/202106/844301_7R5UB4ZTB2CGHCF.jpg)

0x2：引言
------

    今天咱们讨论的是 frida-gadget 的持久化，通俗理解也就是注入 frida-gadget，让目标 app 加载该 so 文件，进而实现 frida 的 hook 功能，并且和 app 共生，一定程度上也免去了反调试，反 frida（修改 so 名字，从 maps 看检测风险减小，或许从 hook 原理继续检测？先不说 find_mem_string 检测）的情况，比较适用于大厂强风控下的群控 hook 的生产场景（这里不谈一键新机 - 设备指纹什么的），所以这个方案还是适合和需要掌握并落地的。  

    一般来说，需要考虑三种设备环境的情况，那就是 root 和非 root 和源码定制，接下来我根据自己的了解和实践分别介绍下拙见。  

    先来看这篇文章[在未 root 的设备上使用 frida](https://bbs.pediy.com/thread-229970.htm "在未root的设备上使用frida")，作者主要介绍的是利用 lief 工具把 frida-gadget 链接到目标 app 的 so 文件上进而实现加载和 hook 功能，优缺点作者也都介绍了，作为引言部分，就先从这篇文章获取到 lief 工具的用途和使用吧。  

0x3：root 环境
-----------

    1，（实践通过）目标 app 有使用 so，利用 lief 工具把 frida-gadget 和目标 app 的 so 链接到一起，实现加载和 hook  

        如：从 / data/app/packageName-xxx/lib 下，找到 app 的 so 然后和 frida-gadget 进行链接  

        风险点：需要过 root 检测，so 文件完整性检测（如：目标 app 可扫描 /data/app/packageName-xxx/lib 目录下所有文件，和文件 md5 上传服务器做校验）  

    2，（未实践）利用 lief 工具把 frida-gadget 和系统库（如 libart，libc）链接到一起，实现加载和 hook

        风险点：需要过 root 检测  

    3，（未实践）magisk 模块方案注入 frida-gadget，实现加载和 hook  

        见文章 [Frida 持久化方案 (Xcube) 之方案一——基于 Magisk 和 Riru](https://bbs.pediy.com/thread-266787.htm)，然后 hanbing 老师也把 fridamanger 做成了 magisk 模块，那天见在群里已经通知了  

        风险点：需要过 root 检测，magisk 检测

    4，（未实践）xposed 模块方案注入 frida-gadget，实现加载和 hook

        见文章 [Frida 持久化方案 (Xcube) 之方案二——基于 xposed](https://bbs.pediy.com/thread-266784.htm)  

        风险点：需要过 root 检测，xposed 检测

0x4：非 root 环境
-------------

    1，（未实践）目标 app 没有使用 so，修改 smali 文件加入代码 load frida-gadget 然后再重打包，实现加载和 hook

            风险点：重打包检测（签名检测，文件完整性检测）

    2，（实践通过）目标 app 有使用 so，解包利用 lief 工具把 frida-gadget 和目标 app 的 so 链接到一起然后再重打包，实现加载和 hook

            风险点：重打包检测（签名检测，文件完整性检测）

    3，（实践通过）类 xpatch 方式加载 frida-gadget，众所周知 xpatch 是修改的 manifest 文件替换了 application 入口，然后进行了 sandhook 初始化，xp 模块的查找与加载，application 的修正和过签名检测，三大主要功能，然后再回头看 sandhook 都注入了，再注入个 frida-gadget 不就改改代码的事嘛

            风险点：重打包检测（签名检测，文件完整性检测，xpatch-sandhook 检测）

0x5：源码定制
--------

    1，（实践通过）见文章 [ubuntu 20.04 系统 AOSP(Android 11) 集成 Frida](https://www.mobibrw.com/2021/28588)，作者是直接在 system/lib 下放入了 frida-gadget，我是参考了强哥文章[玩转 Android10 源码开发定制 (九) 内置 frida-gadget so 文件和 frida-server 可执行文件到系统](https://bbs.pediy.com/thread-264916.htm)，利用了 PRODUCT_COPY_FILES 的方式把文件 copy 到了 system/lib 下，然后我又改了`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp的``com_android_internal_os_Zygote_nativeForkAndSpecialize`函数，实现了 fork 进程的时候 dlopen frida-gadget 给每个进程都加上 frida，进而实现了 hanbing 老师的 fridamanger 低配版（[FridaManager:Frida 脚本持久化解决方案](https://bbs.pediy.com/thread-266767.htm)），然后强哥说他是直接改的 java 层代码 system.load 的，我猜测是在 makeApplication 之类的地方做的，由于已经实现功能，所以未做后续验证。

0x6：总结
------

    其实本文的真正的出发点还是想从【反反制 / 较小风险点的情况下去实现 frida-gadget 的持久化】来聊方案，故还是推荐【非 root 环境和源码定制】的方式（ps：国产厂商 root 越来越难，非 root 更加通用）

     非 root 环境推荐：魔改 xpatch + 魔改 sandhook+VirtualApp 的 iohook = 渣总 ratel 低配版（[平头哥开放文档](https://git.virjar.com/ratel/ratel-doc)），这种方案下，魔改的 xposed 和 frida 都有了，想用嘛用嘛，基本只能从【java 层 hook 原理 + inline hook 原理 + syscall（[Android svc 获取设备信息](https://bbs.pediy.com/thread-264641.htm)）】几个点来检查，纬度也算来到了高维对抗，并且开源轮子的显著特征都没了，至于能不能过还要看检测的具体强度，当然这个方案适配是个问题，32 位下基本没问题，64 位还要根据厂商再看看。大手子看到这里已经猜到了选用该方案已经不单单是为了做 hook 持久化了，后续还要做新机，多开等功能，尽可能完善地落地一个闭环的通用型方案。  

    源码定制推荐：自己动手去实现 fridamanger 低配版就好，因为后续还要做设备指纹，systemapp，内核模块开发，多开等一系列功能。  

0x7：感谢
------

    感谢上面内容提到每一个名字和项目

    哥哥们的先行和分享推进了社区的进步  

    写代码和做技术因你们变得更加美好  

    哥哥们牛批！  

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 2021-11-16 10:58 被 huaerxiela 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码分析](forum-161-1-127.htm)