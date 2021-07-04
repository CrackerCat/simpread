> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268340.htm)

> [原创] 关于多 classloader 的 hook 和原理探究

**问题提出**

群里有朋友问 “我枚举出那个类了，但是用 java.use 的时候就报错找不到，这是这个 app 做保护了吗”。很早的时候我用 frida 也遇到过同样的问题。

**问题现象**

当我使用 enumerateClassLoaders 时，在 onMatch 里可以找到我想找的类，但是我在外部直接 Java.use 时则报错找不到指定类。

**原因及解决**

该 APP 本身有多个 classloader，我想 Java.use 的类并不在 frida 的默认的 classFactory.loader 的 classloader 里面，这个时候怎么办呢，一个办法就是我们把默认的 classloader 改为我们能找到指定类的 classloader，改法如下。

![](https://bbs.pediy.com/upload/attach/202107/867442_2ZMTUFF23K8FPPY.jpg)

这样通过枚举找到所有 classloader，再挨个找到这个类，找到的时候将这个 classloader 改为默认的 classloader 就好了。那为什么这么改呢？我们就得看看源码啦。

**深入研究**

在 firda-java 的源码文件 lib/class-factory.js#L143 可以找到 Java.use 的方法原型。（这里参考了看雪的一篇关于 firda 源码分析的帖子，找不到链接了。瑞斯拜）

![](https://bbs.pediy.com/upload/attach/202107/867442_96RPY7HY4W2JFTB.jpg)

其中 loader 的初始化来自 firda-java 的源码文件 /index.js#L549

![](https://bbs.pediy.com/upload/attach/202107/867442_JN8URG86ZJQ85CH.jpg)

那我们现在知道了，如果默认的 getClassloader() 不包含我们想要的类，我们就需要修改 factory.loader 切换 classloader 就可以指定 classloader 进行 Java.use 了。如果你想理清楚这部分逻辑，可以从 index.js 的 perform 函数开始看就很容易梳理清楚了。

frida 能这么枚举出所有 classloader，那在 java 层具体体现在哪里呢？

**举例说明**

我们以阿里系的 APP 为例，全局搜索 classloader 之后，定位到这两行代码。

![](https://bbs.pediy.com/upload/attach/202107/867442_DP3WRXU5JM9UHTS.jpg)

这里就是多个 classloader 生成的地方了 (类: com.alibaba.wireless.security.framework.d)，他通过加载特定 dex 到新的 classloader，自己实现了相关的管理。我们如果想调用该 loader 下相关类的方法(如 x-signature 等) 就需要 hook 了上述第一个函数将返回结果作为 loadClass 的对象，就可以找到具体的类和方法了。

感兴趣的可以自己实现一下～

可以参考下述代码:

![](https://bbs.pediy.com/upload/attach/202107/867442_BBQC7U2CDT2J5EK.jpg)

以上。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm)