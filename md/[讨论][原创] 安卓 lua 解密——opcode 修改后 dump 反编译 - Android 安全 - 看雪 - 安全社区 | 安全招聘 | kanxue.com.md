> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282259.htm)

> [讨论][原创] 安卓 lua 解密——opcode 修改后 dump 反编译

[讨论][原创] 安卓 lua 解密——opcode 修改后 dump 反编译

12 小时前 263

### [讨论][原创] 安卓 lua 解密——opcode 修改后 dump 反编译

 [![](https://bbs.kanxue.com/view/img/avatar.png)](user-home-995191.htm) [packets_64](user-home-995191.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif) 12 小时前  263

随着近些年 lua 语言在安卓游戏大火，lua 逆向已经越来越重要。早期游戏的 lua 源码放在安装包的 assets 目录，但随着对抗的升级，越来越多的游戏使用动态更新和扩展应用功能的方式，特别是在需要频繁更新脚本或配置的情况下。这种方法在游戏开发、插件系统、远程配置等场景中有较多应用。而且这种方法是网络上加载 Lua 代码，你在安装包, APK 是得不到 lua 的代码的。遇到这种我们首先要确认是否是 lua 语言的应用：

![](https://bbs.kanxue.com/upload/attach/202406/995191_DWG5E5WRVM2CY66.webp)

打开 apk\lib\arm64-v8a \ 目录，如果发现 libxlua.so,libslua.so 等等类似目录，那么有很大可能是 lua。（lua 又分 luajit 与普通 lua，判断是不是 luajit，将 so 文件放进 IDA，字符串搜索 luajit 即可）

再使用 IDA 或者 frida hook lual_loadbuffer 获取字节码，使用 unluac 网址（[unluac download | SourceForge.net](https://sourceforge.net/projects/unluac/)）进行反编译。

java -jar unluac.jar 你的字节码文件 > 存放文件

![](https://bbs.kanxue.com/upload/attach/202406/995191_VEVMP9MH279TW67.webp)

反编译成功一般不会有输出信息，但看到类似上面的情况大概有 3 种可能：

    1. 文件头被改  

    2.opcode 修改  

    3. 文件头和 opcode 都被改

关于文件头：

![](https://bbs.kanxue.com/upload/attach/202406/995191_884ATYV32NDAPYB.webp)

**0 到 3 字节：1B 4C 75 61** 表示这是一个 Lua 字节码文件。

**第 4 字节：****51**：版本号，表示 Lua 5.1。

**第 5 字节：00**：格式号，表示标准格式。01 为非官方，魔改 lua。

**第 6 字节：01**：endianness 标识，表示大端格式。字节码文件需要指示其数据的字节顺序，以确保在不同平台和硬件架构上的正确解析。不同平台可能采用不同的字节序，比如大多数 x86 和 x86-64 架构采用小端序，而一些嵌入式系统和网络协议采用大端序。

**第 8 字节到 11 字节：**Lua 整数大小，指令大小等。

关于 opcode 修改：liblua.so 文件是 lua 源码文件编译的，lua 源码的 "lua-5.1.5\src\lopcodes.h" 文件定义了 opcode：![](https://bbs.kanxue.com/upload/attach/202406/995191_RXW2KTBJ6YBTN3B.webp)

如果在编译的时候将 opcode 的顺序修改，比如 OP_MOVE 在 OP_GETUPVAL 前面，改为 OP_GETUPVAL 在 OP_MOVE 前面，再编译为 so，这时再使用 unluac 反编译会报错。

解决办法有两个：

一：将正常的 so 文件与修改后的 so 文件，放进 IDA 反编译，搜索函数：luaV_execute 找到 case，进行对比，还原 opcode 的顺序。

![](https://bbs.kanxue.com/upload/attach/202406/995191_K6HERUR4JHKKVNU.webp)

还原 opcode 之后，修改 unluac 的源码的 src/unluac/decompile/OpcodeMap.java 文件：

![](https://bbs.kanxue.com/upload/attach/202406/995191_UEWMVZNA49CPQP6.webp)

再使用 unluac 反编译。  

二：[[原创] 用 Lua 简单还原 OpCode 顺序 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-250618.htm)

大佬的办法，从应用内部运行 lua 脚本，获取字节码。再将同一个 lua 脚本运行在未被修改的环境。

![](https://bbs.kanxue.com/upload/attach/202406/995191_37ZMATSZ5AJGDEM.webp)

使用 python 将两个字节码文件对比，将红色的部分使用 python 替换，然后使用 unluac 反编译.

![](https://bbs.kanxue.com/upload/attach/202406/995191_MJXSGKZKVMQQDDX.webp)

反编译成功！

  

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm)