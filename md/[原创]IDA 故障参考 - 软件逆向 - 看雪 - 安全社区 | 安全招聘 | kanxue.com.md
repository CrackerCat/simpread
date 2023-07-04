> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277872.htm)

> [原创]IDA 故障参考

[原创]IDA 故障参考

10 小时前 548

### [原创]IDA 故障参考

 [![](http://passport.kanxue.com/upload/avatar/183/923183.png?1618528084)](user-home-923183.htm) [Hedione](user-home-923183.htm) ![](https://bbs.kanxue.com/view/img/rank/6.png) 1  ![](http://passport.kanxue.com/pc/view/img/star.gif) 10 小时前  548

IDA 故障排除过程记录
============

由于 IDA 闭源，又加上其十分无效的官方文档（指和代码无异，代码也没有注释，文档也没有注释）。因此如果出现任何错误，都需要进行分析和查错，这一过程很麻烦。于是我在这里给出逆向 IDA 错误代码的方法。本文可以作为部分错误的索引，也可以作为逆向一个大型软件的初学者教程。

也算是久病成医了，众所周知越是容易发生的故障在要紧的时候越会发生。上次 ciscn 边逆边修 ida。这次打完了，我就要看看 ida 到底抽什么疯！

情景再现
----

你遇到了下述故障

![](https://bbs.kanxue.com/upload/attach/202307/923183_ACPP6XDVWY98P4J.png)

是的，这个故障没有给出其他的信息。像这样的 “一刀砍死” 的故障还有很多。

![](https://bbs.kanxue.com/upload/attach/202307/923183_MWGZ295BK26JT77.png)

![](https://bbs.kanxue.com/upload/attach/202307/923183_PFBXTFME8GE83AK.png)

IDA 的霸王条款是告诉你要么生成一个 dump，然后关闭 IDA，要么直接退出。于是你生成了 dump 文件。

![](https://bbs.kanxue.com/upload/attach/202307/923183_STJ6D5MDU7PPVJT.png)

按下确定之后，IDA 就随风而逝了。你气急败坏的不断导入文件，但是 IDA 就是屹然不动。欸，错误修复了吗？如修！

观察故障的表和 log，也没有任何反馈。于是你开始去搜索。搜索的结果就是，没有任何帮助。于是你来到了一片没有知识的荒原。于是只好干回老本行，用逆向手段观察这个错误是什么。简称，用 IDA 调试 IDA。

![](https://bbs.kanxue.com/upload/attach/202307/923183_JZDG52BQ2CQV5X8.png)

### 安全模式

借鉴 Windows 的灾难修复过程，程序的 “安全模式” 指的是关闭其他一切插件、附件，以最初的配置启动 IDA。因为你很明确在你第一次启动 IDA 的时候这个程序运行是正常的，只是有一天你心血来潮看到了 IDA 的某个插件，于是你装了第一个插件。于是你接着装了第二个，第三个，直到今天这个可怕的错误爆发。

首先你要禁用所有 plugin。将你的 plugin 文件夹按照日期排序，从新到旧移除你以前安装的插件。

![](https://bbs.kanxue.com/upload/attach/202307/923183_X76C9R3XTPG95NH.png)

最终你发现，如果装上红框里的插件，这个错误就会出现。于是你锁定了这个插件，修复这个插件就可以了。

修复的参考步骤将放在后文技术节

### 未知 bug

可是你没有安装任何插件，或者说这个 bug 如影随形，根本不是插件引起的。于是我们进入 Exception Analysis 阶段。

使用 WinDBG 打开导出的 dump 文件

![](https://bbs.kanxue.com/upload/attach/202307/923183_C35WXU2PBXRCKUS.png)

windbg 大名鼎鼎，其威名必不用说。载入后，你会看到红框部分的提示

```
For analysis of this file, run !analyze -v

```

windbg 的语法在此不表，虽然诸位不开发驱动或者硬件，但是 windbg 是推荐学一学的。根据上面的提示，我们按下这个超链接（或者键入指令）

```
!analyze -v

```

经过 windbg 的一顿操作，下面的输出多了不少

![](https://bbs.kanxue.com/upload/attach/202307/923183_W98JQKFSTCMESZP.png)

我们继续向下翻

![](https://bbs.kanxue.com/upload/attach/202307/923183_F3AVXRWMV8GV996.png)

在这里，记录了发生错误的进程名称和错误代码。一般而言，根据规范引发的错误，尤其是经由 RaiseException 的错误，大多都应该返回一个有效的错误码。这一点在 Windows 的蓝屏分析上尤其重要。使用 WinDBG 分析 BSOD 错误也是这个思路，根据 PROCESS NAME 就可以分析错误应用，并由此大概可以猜测到底是什么故障了。

我们本寄希望于错误代码能告诉我们就行发生了什么错误，但是奈何这个错误代码`0xe0424242`根本狗屁不通。于是我们只好接着向下看。注意到接下来出现`STACK TEXT`。这里代表栈追踪，用于指示程序在执行过程中发生异常或错误时的调用堆栈信息。系统的代码调用路径就显示在堆栈跟踪上。

![](https://bbs.kanxue.com/upload/attach/202307/923183_8VSQJHJV5NMA7QN.png)

堆栈跟踪中显示，调用链如下：

```
ida64_exe+0x17b002 -> ida64!init_database+0xe2d -> ida64!user2bin+0x4200 -> ida64!user2bin+0x681f -> ida64!interr+0x37 -> ida64!print_fpval+0x5bc2 -> ida64!verror+0x25 -> ida64_exe+0x53bd4 -> ida64_exe+0x57bf7 -> ida64_exe+0x58282 -> ida64_exe+0x56f16 -> KERNELBASE!RaiseException+0x69

```

那么回溯上去，我们可以从内存地址`ida64!init_database+0xe2d`开始看。打开 IDA，注意到此时的被调用方为`ida64`，代表 dll。载入 dll，根据导出表找到`init_database`的基址

![](https://bbs.kanxue.com/upload/attach/202307/923183_F4MEP5ZDAW7X9QD.png)

计算`init_database+0xe2d`，追踪到对应的内存位置

![](https://bbs.kanxue.com/upload/attach/202307/923183_RVZ85DYH3H8DNUQ.png)

由于栈底保留的是返回指针，即函数 ret 之后应该执行的下一条指令的位置，于是我们知道异常的发生应该在函数`sub_101CA560`内。

![](https://bbs.kanxue.com/upload/attach/202307/923183_MGTAYM9TSY58PZ7.png)

注意`user2bin+0x4200`位置的代码，在往下追踪之前，我们先看看这是干什么的。

![](https://bbs.kanxue.com/upload/attach/202307/923183_RG7FPCW6MQQHQ3U.png)

根据提示字符串，这是在加载插件时出现故障了。

追踪到`sub_101CBC60`函数内查看

![](https://bbs.kanxue.com/upload/attach/202307/923183_SAM4S8B56H4PHJX.png)

注意到这里发生错误 (`interr`的条件为`!v44 && *((char *)v40 + 144) < 0`)，很明显`v40`是一个结构体。动态调试一下看

![](https://bbs.kanxue.com/upload/attach/202307/923183_PWEA37QWJ3UFJRF.png)

分析过程不表，因为已经可以用动态调试的方法获得参数，在这里给出函数的原型

```
__int64 **__fastcall sub_5755BC60(__int64 a1, __int64 a2, char *file_path, int a4, unsigned __int16 a5, int a6, unsigned int a7)

```

其第三个参数`file_path`为加载的`dll`等一众插件的地址。这里会对插件进行初始化。

![](https://bbs.kanxue.com/upload/attach/202307/923183_SGF8TBD3VMUGBKT.png)

其中参数`v44`为当前插件的代码入口。

![](https://bbs.kanxue.com/upload/attach/202307/923183_228TRE9AD7ZBVSM.png)

而`v40+144`我猜测是链表的`next`项。如果插件出现未卸载或者什么别的后卸载故障（例如内存未回收等），那么这里的代码入口和下一表项可能出现为空、为负数的故障，因此引发错误，错误编号`1827`。

部分故障码分析结果
---------

有了上面的分析基础，又加上`ida`本身并不进行反调试，因此可以搜索`interr`的函数调用，分析上下文，给出错误原因。

### 1491

IDA 7.0 故障，已经在后续版本中被修复了。

这个故障由 IDA Pro 中的 `winbase_debmod.cpp` （ Windows 调试器模块）引发。这段代码的目的是向 `ntdll_vec_t` 类型的容器中添加一个新的 `ntdll_range_t` 元素。

```
// winbase_debmod.cpp
// Line 388
// ......
bool ntdll_vec_t::add(eanat_t addr, size_t sz, HANDLE h)
{
  if ( has(addr) )
    return false;
 
  // max number of ntdlls: ntdll32.dll and ntdll.dll
  //QASSERT(1491, size() < 2);
  ntdll_range_t &r = push_back();
  r.start = addr;
  r.end = addr + sz;
  r.handle = h;
  return true;
}

```

> [reference](https://forum.exetools.com/showthread.php?t=19452)

该函数 `ntdll_vec_t::add()` 在调试器模块中用于添加对应的 ntdll（NT DLL）库的信息。这段代码包含了一个断言（assertion）`QASSERT(1491, size() < 2)`，它的作用是确保容器中的元素数量不超过 2。如果当前容器中的元素数量已经达到或超过 2，那么这个断言将会触发一个错误（error）编号为 1491。

根据提供的信息，为了解决在 Windows 10 的版本大于 16xxx 下使用 IDA Pro 7.0 时的问题，建议注释掉这个断言或者通过修复二进制文件来禁用这个检查。

### 40343

此故障属于 Qt 界面方面的故障，根据调试信息

![](https://bbs.kanxue.com/upload/attach/202307/923183_AS7BRKFFH4NT56S.png)

这里会进入选择架构框

![](https://bbs.kanxue.com/upload/attach/202307/923183_TXGPWHJUB5DEABA.png)

根据网上的教程，引发此错误的原因应该是由于路径中出现宽字符（例如中文），导致`qword_7ff755247b70`对应的参数（即支持的架构数量）归零，然后选中的架构又高于 0，于是引发了 40341 错误。

### 1203

来源：[origin](https://www.techbliss.org/threads/oops-internal-error-1203-in-ida.942/)

由于版本是 6.8，已经老旧到近 10 年了，这个报错很可能已经被消除了，因此在此不表。如果确实有大哥 2024 年还在用 2014 年的工具.... 升级一下 ida 吧

### 40178

> BUGFIX: debugger: using instant debugger for debugging a 32bit MacOSX application could cause internal error 40178

[IDA: What's new in 6.4 (hex-rays.com)](https://hex-rays.com/products/ida/news/6_4/)，早在 6.4 版本就修复了 MACOSX 的故障，

### 520

> BUGFIX: Leaving a mark, and then right-clicking on the address of an instruction could cause IDA to INTERR with the code 520

[IDA: What's new in 6.9 (hex-rays.com)](https://hex-rays.com/products/ida/news/6_9/)，虽然听上去 520，但是这是一个 bug，已经在 6.9 版本修复了。

### 1827

分析过程不表，插件出现故障。修复的方法是修改插件的加载 flag。（目前不确定是否 100% 有效，Use at your own risk）

![](https://bbs.kanxue.com/upload/attach/202307/923183_S53P4MEKJHY255B.png)

这里插播一个小知识

> *   `PLUGIN_MOD (0x0001)`: 表示插件可以修改 IDA 的状态或行为。
> *   `PLUGIN_DRAW (0x0002)`: 表示插件可以在 IDA 的图形界面上进行绘图或显示自定义图形。
> *   `PLUGIN_SEG (0x0004)`: 该标志在 IDA 7.0 之前使用，现在已被弃用，不再使用。
> *   `PLUGIN_UNL (0x0008)`: 该标志在 IDA 7.0 之前使用，现在已被弃用，不再使用。
> *   `PLUGIN_HIDE (0x0010)`: 表示插件应该隐藏其界面，以便在加载时不显示插件的任何用户界面元素。
> *   `PLUGIN_DBG (0x0020)`: 表示插件是一个调试器插件，用于与调试器交互。
> *   `PLUGIN_PROC (0x0040)`: 表示插件是一个处理器模块（Processor Module），用于支持特定的处理器架构。
> *   `PLUGIN_FIX (0x0080)`: 表示插件是一个修复插件，用于修复 IDA 中的一些问题或提供额外的修复功能。
> *   `PLUGIN_MULTI (0x0100)`: 表示插件是一个多实例插件，允许同时存在多个实例。
> *   `PLUGIN_SKIP`: 表示插件在初始化期间被跳过。
> *   `PLUGIN_OK`: 表示插件初始化成功。
> *   `PLUGIN_KEEP`: 表示插件在 IDA 关闭时保持加载状态。
> 
> 这些常量用于设置插件的标志属性，并根据插件的需求来定义其行为和特性。插件开发者可以根据需要选择适当的标志来定义插件的行为。

> `PLUGIN_KEEP` 是一个 IDA 插件的常量，用于定义插件的行为和生命周期。具体来说，它指定了插件在 IDA 关闭时是否保持加载状态。
> 
> 当插件被设置为 `PLUGIN_KEEP` 时，它将保持加载状态，即使用户关闭了 IDA。这意味着，在下次启动 IDA 时，插件将继续保持加载状态，并自动运行。
> 
> 插件保持加载状态的好处是，它可以在 IDA 重新启动时无缝地恢复之前的状态和设置。例如，如果插件负责维护一些用户自定义的配置或数据，那么通过设置 `PLUGIN_KEEP`，它可以确保这些数据在 IDA 关闭和重新打开后仍然可用。
> 
> 值得注意的是，插件的保持加载状态并不意味着它会一直运行。插件的运行仍然取决于其初始化函数的调用时机和其他条件。

_来源：ChatGPT_

### 2028

遇到过一次，原因是在文件加载之前就调用了 Strings 窗口。

![](https://bbs.kanxue.com/upload/attach/202307/923183_KPYPK4AN75ABV6G.png)

可能是出于安全性设计，因此修复方案也很简单，别调用就行。

这个也是插件故障。

  

[VMProtect 分析与还原](https://www.kanxue.com/book-section_list-87.htm)

最后于 10 小时前 被 Hedione 编辑 ，原因： 修复图片链接 [#其他内容](forum-4-1-10.htm)