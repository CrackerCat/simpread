> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280373.htm)

> [原创]autojs 简介与对抗

[原创]autojs 简介与对抗

发表于: 2024-1-28 19:56 21336

### [原创]autojs 简介与对抗

 [![](http://passport.kanxue.com/upload/avatar/851/963851.png?1663908286)](user-home-963851.htm) [xiaoc996](user-home-963851.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2024-1-28 19:56  21336

前言
==

最近发现现在大多安卓工具很多都是 autojs 和按键精灵等这些非常易于开发的工具来做黑灰产自动化工具，此帖也是由此做为出发点来展开说说这个 autojs，我们如何去做对抗。由于笔者对这块也是刚了解学习不久而且这篇帖子也会说的很简单，有很多不足的地方或者解释不充分还请大佬们指出。

autojs 介绍与版本
============

autojs 介绍
---------

我们知道使用 js 来编写一个 apk 是多么的爽，ui 部分不使用 xml，代码部分不使用 Java，全程使用 js 来编写对刚入门想要开发一款安卓应用来说是一件非常方便易学的事，所以就有了 autojs 这么一款工具。  
使用官方的一句话就是

![](https://bbs.kanxue.com/upload/attach/202401/963851_2JS97XZBHBMTX4J.webp)

是的，我也觉得 autojs 这款工具对于老人难以进行复杂的操作和子女进行微信视频提供了很大的帮助，具体有没有人这样做我也不知道。

autojs 版本
---------

autojs 的版本还是分得比较多，现在常用版本分为：autojs4.1、 autojs pro7、autojs pro 8、autoxjs（此帖子也是围绕这个版本来的）但是本身 autojs 的作者已经不更新了，并且这个开源项目也已经删除，原因是这款工具被太多的黑灰产利用了。还有一些其他的版本，但是主流还是上面那 4 个版本用的比较多，其中 pro7 和 pro8 都是收费版但是现在都被破解了。现在还在更新和维护的是 autoxjs。

autojs 原理
=========

1. 这个软件的 ui 界面并不是由 js 写的。是这个软件提供了一个可以编写界面的 js 环境。这个软件本身的界面是由 Java 和 Android XML 编写的。  
2. 这是利用了 AccessibilityService 的 API。参见 AccessibilityService 的 getRootInActivieWindow() 函数。  
`common`模块提供了其他各个模块的公用类、工具等，例如一些数据结构、View 工具类等。是其他各个模块的依赖。  
`automator`模块实现了自动操作的大部分内容。包括选择器的实现、简单操作的实现、控件节点的封装等。是 autojs 模块的依赖。  
autojs 模块是 Auto.js 的 JavaScript 运行环境，包括脚本引擎的封装，核心运行库的实现，对 JavaScript 层暴露的 API，JavaScript 和 Java 的交互。同时提供了管理运行的 JavaScript 脚本的服务。  
app 模块是界面、业务逻辑。依赖 autojs 模块。  
项目主要需要`Android`基础，和`uiautomator`基础没有太大关系  
这里是转载来自：[https://www.jianshu.com/p/1ff3f2c8e540](https://www.jianshu.com/p/1ff3f2c8e540)

以下为个人理解的 autojs 较为核心的模块，因本人也没有深入到源码去了解，可能会有出入欢迎指出。

![](https://bbs.kanxue.com/upload/attach/202401/963851_R337YR8A2Q55KQB.webp)

执行流程
----

由于现在的 autojs 版本没有做太多混淆处理，而因为他本身就是一款模拟点击的 app，我们去做分析的话可能不会有太多价值，比较源码里面也都是模拟点击的 api，后续我们对抗肯定也不会从模拟点击的 api 入手，所以这个执行流程我简化为：读 js 代码 --> 有加密则执行解密 --> 先转换 ui 部分代码为 xml 代码再转换 js 代码为 art 字节码 --> 对界面做展示。

文件结构
----

怎么去判断一个 apk 是 autojs 开发的只需要找 assets/project 有 js 文件和一个 json 文件就可以判断，其次就是看 libc 库，里面有  
"libjackpal-androidterm5.so",  
"libjackpal-termexec2.so"  
这两个也可以确定是 autojs 开发的 app

![](https://bbs.kanxue.com/upload/attach/202401/963851_W3BXSCE8RT3KEJR.webp)

解密 js
=====

如果加密过的 js 肯定会执行解密，这里我们可以从两个地方进行 hook 拿到解密后的 js

![](https://bbs.kanxue.com/upload/attach/202401/963851_N27VDQ5TD7M3BQT.webp)

其实这两个点都可以做一个 hook，我自己做的一个小工具也是把这两个点做了 hook，首先就是第一个函数 StringScriptSource 这个点，他会传入两个 String 参数，一个是解密的文件名一个是解密后的 js，可以把解密后的文件名和 js 做保存。

![](https://bbs.kanxue.com/upload/attach/202401/963851_VMF2NRKDRFZU3NW.webp)

第二个就是解密的函数，如果 hook 解密的函数需要对返回值做转换然后保存即可，直接在这个函数做 hook 后面的逻辑我们也不用管。  
![](https://bbs.kanxue.com/upload/attach/202401/963851_GN8VTJVVHUEYX48.webp)

这两个点就已经可以拿到源码了，如果做了加固，再换以下 classloader 即可，这边贴出使用 xposed 做的 hook

```
public void de_autojs(ClassLoader classLoader) {
    XposedHelpers.findAndHookConstructor("com.stardust.autojs.script.StringScriptSource", classLoader, java.lang.String.class, java.lang.String.class, new XC_MethodHook() {
        @Override
        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
            super.beforeHookedMethod(param);
            XposedBridge.log("xiaoc----StringScriptSource" + param.args[0].toString());
            //XposedBridge.log("xiaoc----StringScriptSource"+param.args[1].toString());
 
            mainhook.main(param.args[1].toString(), param.args[0].toString());
        }
 
        @Override
        protected void afterHookedMethod(MethodHookParam param) throws Throwable {
            super.afterHookedMethod(param);
        }
    });
    XposedHelpers.findAndHookMethod("com.stardust.autojs.engine.encryption.ScriptEncryption", classLoader, "decrypt", byte[].class,
            int.class,
            int.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    super.beforeHookedMethod(param);
                }
 
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    String str = mainhook.bytesToString((byte[]) param.getResult());
                    XposedBridge.log("xiaoc_ScriptEncryption" + str);
                    mainhook.main(str, "de");
                }
            });
}

```

对抗思路
====

虽然我们上面拿到了源码，但我一开始也说了我们即使拿到源码也看不出东西全是 autojs 模拟点击的方法，如果要做对抗我们应把重点放到无障碍服务与风控上。

这也是我的主要对抗思路，下面介绍几种思路。

1. 无障碍列表可疑应用
------------

获取已开启无障碍的应用，通过入口函数来查看，包含 autojs 字样或者 stardust 字样，直接闪退即可，但是使用这种方法也得考虑自己的用户场景，有误杀的可能，最好结合多种来进行检测。也可进行动态检测，如果发现此款 app 后端发现是黑灰产工具，也可以直接闪退 (笔者有点不成熟，最好不不要使用哈哈，会有更好的检测手段)

```
ResolveInfo{ea35d2a org.autojs.autoxjs.v6/com.stardust.autojs.core.accessibility.AccessibilityService m=0x108000}

```

2. 主动抛出异常
---------

上面的介绍图中说到，AccessibilityService 这个是无障碍服务，如果没有这个服务我是实现不了模拟点击的，我们可以通过主动抛出异常，来检测控件是否被 Accessibility 控制按下或者滑动。

![](https://bbs.kanxue.com/upload/attach/202401/963851_FUZWZAGJ7ZE5XEG.webp)

3. 轨迹变化特征
---------

![](https://bbs.kanxue.com/upload/attach/202401/963851_3PPJYJUFXUJUMQY.webp)  
![](https://bbs.kanxue.com/upload/attach/202401/963851_K4WB47EUR8C446J.webp)

通过以上两张图，我们已经可以很形象的看出来，通过轨迹特征来判断一个 app 被自动化脚本操作检测起来是多么简单事

4. 群控、按部就班
----------

如果被群控了，那么肯定是多个用户在一个比较集中的时间内开始作业，而他们的作业路径非常统一都是按部就班的执行一切操作，此时我们可以先把这批账号拉入监控名单，一旦触发可直接封

总结
--

其实现在大厂的检测很多都是做的风控检测，也就是后面所说的两种，但肯定是远远不止这两种，主要还是围绕着来做，说起来简单做起来难，现针对模拟点击也没有一个很好的针对客户端来做检测，包括按键精灵、触动精灵等脚本等工具。总结下来还是做好风控。

笔者废话
====

整篇看下来很简单，内容也不多，都被简化了而且也挺多没提到，就好比无障碍这个服务是如何去调用并实现点击功能的整条链路，或许如果能去学习下无障碍这个服务怎么实现的一个点击或者说阅读 autojs 的源码会有更好的对抗思路，文章很简单，笔者也是做一个记录，后面如果有更好的客户端检测手法也会给读者们贡献出来，让这个圈子更卷

  

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#逆向分析](forum-161-1-118.htm) [#基础理论](forum-161-1-117.htm) [#工具脚本](forum-161-1-128.htm)