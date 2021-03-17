> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266488.htm#msg_header_h2_19)

本文是针对刚开始接触 X64dbg 的新人写的实用技巧和插件合集

前言
--

萌新一个，接触逆向时间不长，但因为很喜欢 x64dbg 这款调试器，所以花了一些时间去了解，x64dbg 的这类帖子相对较少，本贴的初衷是希望其他新人在学习的时候可以多一些参考，少一些时间精力的浪费。希望大家为我的帖子指正错误和做补充。

 

以下均基于 Mar 12 2021 版本 x64dbg 官方版本，可到官方网站直接下载

 

官方网站：https://x64dbg.com/

第一次运行
-----

双击 x96dbg，出现三个弹窗，会生成 x96dbg.ini 文件。当你需要把整个程序文件夹移动到其他路径时，最好把这个 ini 删除，重新双击让它再生成。当你需要把它加到发送到菜单中的时候，也推荐添加 x96dbg，它会自动选择用 x32 还是 x64dbg 打开。

 

![](https://bbs.pediy.com/upload/attach/202103/912377_EXBTDEM24JHZSC9.png)

你需要做些什么设置
---------

### 外观

*   字体大小调整 选项——外观——字体——最右边那个数字（图右）（我个人用 12）
*   主题选择 选项——主题——选择你喜欢的（图左），也可以在 Github 搜索 x64dbg theme，或者百度搜索 x64dbg 配色，使用别人的配色方案文件
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_4539HCESX2BF6HT.png)
    

### 选项设置

*   事件
    
    推荐萌新只勾选**入口断点**（这样相当于 OD 设置为 第一次暂停于主模块入口点）
    

*   引擎
    
    保持默认即可，需要特别说明的是**禁用数据压缩**和**将数据保存于应用程序目录**，所指的数据是指调试相关数据，默认保存在调试器 db 文件夹下。调试引擎选项里的 **GleeBug**，据作者以前博客内容说是跟踪速度会比默认的 Titan 快很多，但我试了没什么感觉。
    

*   异常
    
    保持默认
    

*   反汇编
    
    **仅在 CIP 处显示自动注释**——勾选之后界面会更整洁，该有的自动注释也还是有的，自行体验取舍
    
    **模块名最长长度**——默认 - 1 不限制，如果遇到比较长的文件名，调试起来界面观感会比较差（图左），OD 的限制长度为 8，自行取舍。
    
    **值前加 0x 前缀**——x64dbg 里默认以 16 进制显示数值（不显示 0x），如果你需要表示十进制，可以在前面加点，汇编里输入加点 10 进制需要设置 XEDparse 引擎（图右）。
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_4T3MV9EVHZ95JH7.png)
    

*   图形界面
    
    其他选项根据个人需要调整，一般保持默认，特别说明一项
    
    **图形缩放模式**——这里指的是 x64dbg 流程图功能 是否支持缩放，勾选后流程图操作体验会稍微接近 IDA。流程图功能会在后面提到
    

*   杂项
    
    其他选项根据个人需要调整，一般保持默认
    
    可以将搜索引擎改为百度：https://www.baidu.com/s?wd=@topic 遇到不认识的函数选中，右键，符号名称帮助，直接进行查询。
    

**使用技巧篇**
=========

*   **收藏夹参数**
    ---------
    
    这里直接贴上官方文档对于收藏夹的使用说明：
    
    如果你在工具的命令行添加 `%PID%`，它将被调试对象 (如果不进行调试或 0) 的(十进制) PID 替换。
    
    如果你添加`%DEBUGGEE%`，则它将添加调试对象 (不带引号) 的完整的路径。
    
    如果你添加 `%MODULE%`，则将添加在当前反汇编中模块 (不带引号) 的完整路径。
    
    如果你添加 `%-????-%`，则它将执行无论你放在何处的 `????`字符串格式。示例：`%-{cip}-%`将会用 `cip` 的十六进制值替换。
    

**所以你可以这样 "C:\Tools\DIE\die.exe" %DEBUGGEE% 添加你常用的工具（记得路径加双引号），然后把收藏夹——收藏工具箱选上。点击 Die 图标，将会启动并且分析当前在调试的模块。这样的方法好过安装什么 PE 查看器插件，让专业的工具做专业的事**

 

![](https://bbs.pediy.com/upload/attach/202103/912377_G2NGN2V4JZAF4HY.png)

*   **流程图**
    -------
    
    在反汇编窗口按下 G 键切换到流程图，再按 G 回来。建议配合图形缩放模式设置，可以用 Ctrl + 滚轮进行缩放。
    
    右键点击脱离后，可以与反汇编窗口同时使用，让我想起 ret-sync 那个插件，后面插件篇再说（我一般还是用 IDA，但万一有人喜欢这种呢）
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_Q2K4MS86UEM9EED.png)
    

*   Ctrl+G **跳转到**
    --------------
    
    可以直接输入 API，会自动补全。默认是输入 VA，如果要输入 RVA 需要这样： **:$RVA**
    
    例： :$0 跳到基址处
    
    如果需要输入 FOA 可以这样： **:#FOA** 或者按下 Ctrl+Shift+G 再输入 FOA
    

*   按 H 键进入**高亮模式**
    ---------------
    
    此时单击需要高亮显示的内容即可高亮显示，再次按下 H，点击空白处可以取消。（也可以设置一直启用高亮，但有误点的问题）  
    ![](https://bbs.pediy.com/upload/attach/202103/912377_4ERUW48RGGFGDSM.png)
    

*   **数据库**
    -------
    
    (或者是 Database)
    
    遇到过几次 x64dbg 数据丢失，写了很多注释就没了，这种情况千万不要慌张，因为你每次关闭 x64dbg 自动保存的时候，都会把上次的数据文件做备份 (.bak)，这时候可以试着点击**还原备份数据库**。
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_TPTKN25BMVGYZP7.png)
    

*   **x64dbg 计算器**
    --------------
    
    个人觉得比系统自带计算器好用，可以一边调试一边计算（非模态 + 置顶），不用切来切去，支持 x64dbg 的各种表达式
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_VQ6A2XYBQV45FMC.png)
    

*   右键——**显示指令提示**
    --------------
    
    会显示汇编指令对应的解释，对于初学者很有帮助，有一个王苏汉化的版本，是将这些帮助文本也完全汉化了的。（自行搜索）
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_CSPH5VJEXA2A3WK.png)
    

*   文件——**改变命令行**
    -------------
    
    可以支持带参数调试
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_6P866CWFG4NJW49.png)
    

*   **复制各种格式的数据**
    -------------
    
    选中你要的部分——右键二进制——编辑——复制数据
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_EGN7G58EEXVUUSV.png)
    

*   **少数情况下 64 位系统使用 x64dbg 调试 32 位程序界面卡顿问题**
    -----------------------------------------
    
    这个问题很玄学，我遇到了，搜了网上也有一个朋友遇到过，我身边的同学也有极少数遇到的，这里给出解决方法。
    
    x32dbg.exe——右键属性——兼容性——调成 Windows 7 （那个遇到同样问题的博主是这么解决的）
    
    但是我这么设置后还是卡顿，后来装了个叫 xAnalyzer 的插件，又恢复了。（我的情况是这个插件和兼容性设置缺一不可，不然就会卡顿，非常奇葩，重装了系统还是一样，在此做个记录，防止有人再遇到）
    

*   ESP 硬件断点
    --------
    
    寄存器窗口空白处右键
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_HAUMZP5PTYRWX4J.png)
    

*   自动跟踪
    ----
    
    简单说一下使用方法，因为我的了解也不是很深
    
    右键——追踪记录——启动运行跟踪——点击菜单栏 跟踪 (N)——步进 \ 步过直到条件满足
    
    在暂停条件或者命令里类似这样设置：eip== 地址
    
    接着就可以在跟踪窗口查看跟踪结果了。
    

*   切换 XX
    -----
    
    切换断点、切换书签，这类命名在最初给我带来了一些困扰和误解，后来发现切换的意思就是 设置 / 取消设置。点一下是设置，再点一下就取消设置，所以命名为切换。
    
    这里顺带一说，x64dbg 的搜索结果，内存窗口，跟踪结果，几乎所有带地址的地方，都是支持直接 F2 设置断点的，各位可以多加利用提高效率。
    

![](https://bbs.pediy.com/upload/attach/202103/912377_9BG4T9ZY9C7FDBM.png)

**插件推荐篇**
=========

以下主要详细介绍我平时常用的插件，建议大家按照需求来安装（插件放在 release\x32 或者 x64\plugins 目录下，以. dp32 或. dp64 结尾）

**Scylla**（脱壳与导入表修复）
--------------------

*   自带的插件，个人觉得比 OD+ImpREC 更方便快捷
    
    常见的脱壳流程：x64dbg 停在需要脱壳的地址上，点击 Scylla 的图标启动，会自动为你设定 IAT info 里的 OEP 为当前地址，点击 dump。
    
    修复导入表：点击 IAT Autosearch，有可能提示：高级搜索结果和普通搜索结果不同，是否使用高级搜索结果。一般都选是，接着点 Get Imports，自动获取需要修复的函数。我遇到过高级搜索反而搜不到的情况，这时候可以试试点否之后再点 Get Imports。确认结果无误后点 Fix Dump，选中刚才 Dump 出的文件，会生成 SCY 结尾的文件。
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_GFHZUKDM58T8Q4X.png)
    

**xAnalyzer**（代码分析辅助）
---------------------

https://github.com/ThunderCls/xAnalyzer

 

该插件对调试程序的 API 函数调用进行检测，自动添加函数定义，参数和数据类型以及其他补充信息，安装以后可以让 x64dbg 与 OllyDbg 的使用体验更接近。

 

效果如图，可以在插件菜单里把自动分析（Automatic Analysis）打开。我个人不喜欢插件自动添加很多注释，会让页面看起来很乱，所以一般在需要分析的时候通过右键菜单来让它分析当前函数。

 

![](https://bbs.pediy.com/upload/attach/202103/912377_UDMEDCV5A3CRTCT.png)

**SwissArmyKnife**（导入 Map 文件）
-----------------------------

https://github.com/Nukem9/SwissArmyKnife

 

没什么好解释的，分析 C 程序的时候用 IDA 生成 MAP 文件，然后用这个插件加载。分析 delphin 程序推荐用 Interactive Delphi Reconstructor 先分析，再生成 MAP 文件用，还有其他程序也同样。MAP 文件可以让你的分析进度大大加快，减少分析一些已知的库函数。

**x64dbg_tol**（中文搜索支持）
----------------------

https://bbs.pediy.com/thread-261942-1.htm

 

必装插件，虽然官方在某个版本开始改善了对中文的支持，但他们在博客上也说了，可能会不全，建议自行安装插件。我自己测试了，嗯，他们说的没错。装了这个插件就能搜到之前搜不到的中文字符串了。

[](#scyllahide（反反调试）)ScyllaHide（反反调试）
-------------------------------------

https://github.com/x64dbg/ScyllaHide

 

x64dbg 官方开发的开源反反调试插件，同类插件也有好几个，但官方的用得人比较多且更新快，所以这边推荐官方的插件。使用方法也很简单，插件菜单——Options——Loaded 里可以选择自带的绕过方案（过一般的反调试可以用 Basic 甚至直接用自带的 调试——高级——隐藏调试器）

 

试了下 VM3.x 的反调试可以用自带的 VM 方案直接过掉（反而 OD 的 StrongOD 插件不行）

 

![](https://bbs.pediy.com/upload/attach/202103/912377_XVRDHMZPU35AY6C.png)

**Ret-Sync** （IDA x64dbg\OD 同步调试插件）
-----------------------------------

https://github.com/bootleg/ret-sync

 

https://bbs.pediy.com/thread-252634.htm

 

可以让 IDA 和 x64dbg\OD\windbg 进行同步调试，支持在 IDA 中直接进行单步、下断点、运行等操作，x64dbg 将会与 IDA 保持同步，提高你调试效率的神器，关于安装、使用说明可以参考官方 github 说明（更详细）和看雪的帖子。

 

这里要特别提一点，如果你需要在虚拟机里用 x64dbg，实体机里用 IDA，需要创建两个. sync 文件，内容都为

```
[INTERFACE]
host=192.168.128.1（实体机IP）
port=9234（端口）

```

一个在装了 IDA 的机器上，放在你需要调试的程序目录下（跟 IDA 生成的 IDB 文件放一起）

 

一个放在安装了动态调试器的机器上，用户文件夹根目录（C:\Users \ 你的用户名）

 

这样就可以实现虚拟机和实体机同步调试了

 

![](https://bbs.pediy.com/upload/attach/202103/912377_2YE8G26VNV3XVAG.png)

**E-ApiBreak**（常用断点设置）
----------------------

https://www.52pojie.cn/forum.php?mod=viewthread&tid=1384349

 

OD 上常用的一个工具，现在 x64dbg 也有了。简单易用，不多说明。

 

![](https://bbs.pediy.com/upload/attach/202103/912377_UD3TDNQQUUF77JW.png)

**E-Debug**（分析易语言程序必备）
----------------------

https://www.52pojie.cn/forum.php?mod=viewthread&tid=1374290

 

原作者 githubhttps://github.com/fjqisba/E-debug-plus

 

最开始我在 OD 上用这个插件，可以让分析易语言的效率倍增。最近发现有人移植到 x64dbg 了，在此推荐一波，打开后会自动识别易语言库函数并且注释。

 

![](https://bbs.pediy.com/upload/attach/202103/912377_3JXH279CKG5EB4G.png)

**HotSpots**（用来寻找事件断点）
----------------------

https://github.com/ThunderCls/xHotSpots

 

下拉框选择对应语言，按下按钮可以在按钮事件处理函数 下断点。

 

![](https://bbs.pediy.com/upload/attach/202103/912377_TXZX2FRYGZNTQ5K.png)

*   **BaymaxTools**（特征码提取和搜索）
    -------------------------
    
    https://github.com/sicaril/BaymaxTools
    
    快速提取选区特征码和特征码搜索功能（当然你也可以用 Ctrl+B 搜特征码）
    
    ![](https://bbs.pediy.com/upload/attach/202103/912377_3ZB7N5ZV4PXH4BG.png)
    

*   漏洞利用
    ----
    
    最后对于研究漏洞利用的同学（因为我们最近也刚开始学），我发现两个好用的插件：
    
    ERC.Xdbg（二进制漏洞挖掘辅助，帮你找溢出点之类的）
    
    https://github.com/Andy53/ERC.Xdbg （官方提供五篇教学文档，关于插件的实战，非常详细）
    
    checksec （检测各个模块的各种安全措施状态）
    
    https://github.com/klks/checksec
    

以上就是一些适合萌新的实用插件推荐，其他还有一些没提到的好用的插件，各位可以在底下补充。当然随着你的学习，后面也会需要用到其他类型的插件，甚至自己写插件。可以到官方的插件页面逛逛（http://plugins.x64dbg.com/），翻翻官方的手册。

 

我把个人觉得必备的几个插件打包了，下载覆盖相应文件夹即可。（xAnalyzer，SwissArmyKnife，x64dbg_tol，ScyllaHide，BaymaxTools，xHotSpots）其他插件如果需要自行下载即可

[[公告] 名企招聘！](https://job.kanxue.com/position-list-1-99-99-99-99-99.htm)

最后于 1 天前 被 Endali 编辑 ，原因： 添加附件

上传的附件：

*   [x64dbg 插件. 7z](javascript:void(0)) （2.07MB，40 次下载）