> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.shi1011.cn](https://blog.shi1011.cn/re/android/1216)

> 好久没整 LUA 啦，最近翻到了一篇文章，比较详细的介绍了 LUA 文件的保护方式 我也粗略的了解了一点 Lua 的保…

好久没整 LUA 啦，最近翻到了一篇文章，比较详细的介绍了 LUA 文件的保护方式

我也粗略的了解了一点 Lua 的保护形式，才有了一篇《解密 Cocos2d-x 默认保护形式下的 LUAC》，并且写了一份脚本调用

然而事实上，XXTEA 的保护形式只是 LUA 保护方式之一，这也就是这篇文章的起始点咯

Lua 文件的保护形式
-----------

目前主流的 Lua 文件的保护形式有以下三种：

*   对称加密  
    打包在应用安装包中的 Lua 脚本经过加密，应用加载 Lua 脚本前先进行解密而后运行脚本，通过解密就可以获取 Lua 脚本源代码。  
    解密后也可能获得的是 Luac 字节码，文件头为 `0x1B 0x4C 0x75 0x61 0x51`，这是将 Lua 脚本预编译成二进制文件，从而提升加载速度，需要再进行一次反编译才可以获得 Lua 脚本源代码。
*   编译 LuaJIT 字节码并加密  
    JIT 指的是 Just-In-Time（即时解析运行），文件头为 `0x1B 0x4C 0x4A`。LuaJIT 使用了一种全新的方式来编译和执行 Lua 脚本。经过处理后的 LuaJIT 程序，字节码的编码实现更加简单，执行效率相比 Lua 和 Luac 更加高效。  
    开发者将 Lua 脚本编译成 LuaJIT 字节码，而后再加密 LuaJIT 字节码，应用先进行解密后加载 LuaJIT 字节码。这种方式能够较好的保护 Lua 脚本源代码，反编译存在一定门槛。
*   打乱 Lua 虚拟机中 OpCode 顺序或修改引擎逻辑  
    通过修改 Lua 源代码，将 Lua 脚本预编译成 Luac 字节码，可以理解为重新映射 Lua 虚拟机执行指令，实现门槛较高，无法通过通用工具进行反编译。  
    需使用 IDA 定位到虚拟机的 OpMode 和 `luaV_execute` 函数，通过对比原先 Lua 虚拟机的执行过程，分析出修改后的 OpCode 顺序才可以编译生成反编译工具。

先介绍一下 cocos2dlua 默认的加密方式叭~（咕咕咕，字节码和 Opcode 的坑以后填）

XXTEA 加密算法
----------

如果开发者使用了 Cocos2d-x 框架自带的 XXTEA 加密算法

那摸，可以通过以下几种方法来获得 Lua 脚本源代码：

*   静态分析 so 库  
    使用 IDA 定位到 `luaL_loadbuffer` 函数后向上回溯，分析脚本解密过程后制作解密工具。
*   动态调试 so 库  
    使用 IDA 动态调试，定位到 `luaL_loadbuffer` 地址并下断点，应用会在启动时调用 `luaL_loadbuffer` 函数加载必要的 Lua 脚本，断下后即可将 Lua 脚本源代码导出。
*   Hook so 库  
    Hook 方式与动态调试原理类似，通过 Hook 函数 `luaL_loadbuffer` 地址实现 Lua 脚本源代码导出。  
    动态调试每次只能获取单个 Lua 脚本源代码，如果使用 Hook 只需运行一次即可全部导出。

**更快的方法？**

XXTEA 加密算法需要 Key 和 Sign 才可以进行解密，但是由于 Cocos2d-x 框架的设计原因，导致这个加密形同虚设，所以有一个取巧的办法可以快速获取 Key 和 Sign  
获取 Sign 只需随意打开一个 `.luac` 后缀的加密文件，在文件头看到的一串字符串便是 Sign

Key 也是个字符串，藏在 `libcocos2dlua.so` 文件中，打开 IDA ，能在刚刚得到的 Sign 值附近得到 Key

> Q：为什么可以取巧？  
> A：通过 Cocos2d-x 的[源代码](https://github.com/cocos2d/cocos2d-x/blob/v4/cocos/scripting/lua-bindings/manual/CCLuaStack.cpp#L708)可以了解到开发者需要调用 `setXXTEAKeyAndSign` 函数进行文件解密，Key 和 Sign 都是在同一函数进行调用，所以生成 so 库时，这两个字符串也在一起。  
> 另外也可以使用 IDA 载入 `libcocos2dlua.so` 文件，在 String 中搜索 Sign 值，一般 Key 就在附近，如果附近字符串均无法解密的话，亦可尝试暴力枚举。

当然啦，Hook `setXXTEAKeyAndSign` 函数也是拿到 Key 和 Sign 的一种方法，我在引文中也展示了 Frida 下 Hook 的脚本和解密工具，这里就不再赘述了

字节码的反编译
-------

~咕咕咕~

总结一下
----

由于 Cocos2d-x 框架自身设计原因，自带的加密解决方案只防君子，不防小人，可轻易实现对游戏客户端进行调试、修改等操作，建议通过上文介绍的源代码保护方式自定义加密。

LUAJIT 2.0.3/2.1.0beta3 Opcode 的比对和校验
-------------------------------------

> 最近浅析了一下 luajit，想要实现下反编译，通过万能的度娘，我发现网上的文章大多都是 17 年甚至更前，不过干货不会过期，在非虫大佬的 [lua_re](https://github.com/feicong/lua_re) 系列文章中，可以了解到关于 luajit 编译后的文件的字节码的剖析，然而对于我这样，喜欢用轮子但又有着煤渣技术的人来说，寻找轮子必不可少
> 
> 2021-04-09

通过非虫大佬的分析文章，可以了解到 luajit 文件的结构，但是使用 010Editor 模板并不易阅读，有没有能够反编译出代码形式的轮子呢？

搜索引擎用起来~ 前辈们主要使用的工具：

多数文章介绍并使用了以上几种工具，然而这些工具早已停止维护，ljd 更是在 2014 年停止维护，不过查看这些文章也并不是一无所获：通过修改 luajit 对应版本的 Opcode 就能进行不同版本的反编译

随着 luajit 版本的迭代，其 Opcode 会改变，具体可以查看 LuaJit 仓库的`src/lj_bc.h`找到相应的 Opcode

我对比了版本 2.0.3 以及截止撰写文章前正式发布的最新版本 2.1.0beta3 的 Opcode

![](https://cdn.shi1011.cn/2021/04/44e097c8a939b36ee4bc73c5c003acf7.png?imageMogr2/format/webp/interlace/0/quality/90|watermark/2/text/wqlNYXMwbg/font/bXN5aGJkLnR0Zg/fontsize/14/fill/IzMzMzMzMw/dissolve/80/gravity/southeast/dx/5/dy/5)

看到原有的 Opcode 并没有删除或修改，只是增加了几条新 Opcode，在 ljd 项目中，只需更改`rawdump/code.py`和`bytecode/instructions.py`中的 opcode

**关于 luajit 版本的查找**：一般的，使用 IDA 加载`Shift+F12`或直接使用 010 搜寻 _luajit_ 就能查询到版本信息

![](https://cdn.shi1011.cn/2021/04/35e98ff88192561d1e404cdd1c989053.png?imageMogr2/format/webp/interlace/0/quality/90|watermark/2/text/wqlNYXMwbg/font/bXN5aGJkLnR0Zg/fontsize/14/fill/IzMzMzMzMw/dissolve/80/gravity/southeast/dx/5/dy/5)

基于 ijd 项目，我找到了衍生至最近时间的一个项目 [luajit-decompiler](https://github.com/Dr-MTN/luajit-decompiler)，支持目前 luajit 正式发布的所有版本 2.0.x，2.1.x，我 fork 了作者的仓库，有部分的更改，贴上地址

后续可能会分析下轮子的原理… 下次下次！（逃了）

…….

参考
--

*   [【漏洞分析】浅析 android 手游 lua 脚本的加密与解密](https://gslab.qq.com/article-294-1.html)
*   [Lua 程序逆向分析](https://github.com/feicong/lua_re)
*   [Lua LuaJit 指令表 (整理)](https://blog.csdn.net/zzz3265/article/details/41146569)
*   [luaJIT 指令集介绍](https://blog.csdn.net/Hello_YJQ/article/details/78345069?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242)
*   [[原创] 浅析 android 手游 lua 脚本的加密与解密](https://bbs.pediy.com/thread-216969.htm)
*   [LuaJIT 反编译总结](https://www.freebuf.com/column/177810.html)
*   [[原创]cocos2dx lua 反编译 (20170417 增加补充说明)](https://bbs.pediy.com/thread-216800.htm)
*   [luajit 反编译、解密](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1378796&highlight=luajit)
*   ……