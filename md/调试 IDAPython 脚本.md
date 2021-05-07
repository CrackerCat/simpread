> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267362.htm)

当我们在静态分析程序的时候往往会根据 IDA 提供的各种线索对关键数据进行定位解密，这个时候编写 IDAPython 脚本往往显得格外重要。但是在 IDA 中没有嵌入对应的 Python 脚本调试器，由此我们只能借助第三方开发人员的技术支持实现便捷的调试 IDAPython 脚本。

VS 附加调试 IDAPython 脚本
====================

参考链接：[https://www.52pojie.cn/thread-1356861-1-1.html](https://www.52pojie.cn/thread-1356861-1-1.html)

该方式是利用 VS 的附加形式进行调试。首先下载 VS，安装 Python 支持模块。

![](https://bbs.pediy.com/upload/attach/202105/784955_W8C5D7M7SUKTY5Z.jpg)
==========================================================================

编写一段 Python 脚本让 ida 暂时停止，给 VS 提供时间附加

```
import idc
 
idc.ask_yn(1,"Hello World")
#调试代码写在此处
disasm = idc.GetDisasm(0x10999999)
print("end")

```

使用 ALT+F7 选择调试脚本，让 IDA 弹框暂停。

![](https://bbs.pediy.com/upload/attach/202105/784955_ZHNFF64KCE4CSU2.jpg)

使用 VS 打开 python 脚本设置断点，并点击附加选项

![](https://bbs.pediy.com/upload/attach/202105/784955_4GV58WZVQ7YCHAR.jpg)

选择 Python 代码，附加 ida.exe 进程

![](https://bbs.pediy.com/upload/attach/202105/784955_SS475PEF5B73U58.jpg)

但是这种方式非常考验人品，有的时候可以直接进行附加调试，有的时候就多次抛出异常非常玄学。

![](https://bbs.pediy.com/upload/attach/202105/784955_CMTMDG9CXM3HKA2.jpg)

如果提示就绪，就进入 IDA 界面点击关闭窗口，一定要点击关闭（虽然我也不知道为什么）

![](https://bbs.pediy.com/upload/attach/202105/784955_H75WFYEAGT72US7.jpg)

成功后就会出现以下中断样式，可以正常的调试。（之前成功了，写文的时候失败了就拿论坛图代替了）

![](https://bbs.pediy.com/upload/attach/202105/784955_8X87GGBQ38JBP3Z.jpg)

wing python IDE 调试 IDAPython 脚本
===============================

参考链接：[https://wingware.com/doc/howtos/idapython](https://wingware.com/doc/howtos/idapython)

下载 WING PYTHON IDE 安装：Wing pro 版本，其他版本无法查看变量值。（我使用的是正版 + 吾爱注册机）

在要调试的 IDAPython 代码前嵌入以下调试代码

```
import wingdbstub
wingdbstub.Ensure()

```

我的调试脚本如下

```
import wingdbstub
import idc
import idaapi
 
wingdbstub.Ensure()
#要调试的的Python代码信息
print("Hello IDAPython Debugger")
eax=3+5
print("Let's start Debuger")
print(eax)

```

找到 Wing Pro 安装目录，拷贝 wingdbstub.py 模块，将该文件放到测试用 python 脚本的同级目录下。按照官方文档的要求修改配置（安装版本的 WING 本地调试默认不需要需改）

![](https://bbs.pediy.com/upload/attach/202105/784955_4UD973CT7EGSYHX.jpg)

放到同级目录下的脚本

![](https://bbs.pediy.com/upload/attach/202105/784955_PWT9QF67T57E8CB.jpg)

调试步骤参考：[https://wingware.com/doc/debug/debugging-externally-launched-code](https://wingware.com/doc/debug/debugging-externally-launched-code)

接下来用 Wing pro 打开测试用 IDAPython 脚本设置断点

![](https://bbs.pediy.com/upload/attach/202105/784955_GRRYSVEVKTPE9UA.jpg)

点击左下角小虫子按钮，选择接受调试链接

![](https://bbs.pediy.com/upload/attach/202105/784955_ZZMUPK6S3ZF2TJF.jpg)

可以看到控制台输出以下信息

![](https://bbs.pediy.com/upload/attach/202105/784955_NTBHVMV7BPHZ2N7.jpg)

IDA 启动 IDAPython 脚本，因为脚本中嵌入了调试信息，执行的时候会反向链接 Wing pro 调试器。使用 Alt+F7 在 IDA 中启动脚本

![](https://bbs.pediy.com/upload/attach/202105/784955_TRFRTWK6Z2J239F.jpg)

可以看到 Wing Pro 在 IDA 脚本运行后中断住。

![](https://bbs.pediy.com/upload/attach/202105/784955_WB9BWGQX4YZ5X27.jpg)

按 F6 单步步过，F7 步入。可以使用 watch 窗口监视变量值。但是无法看到 print 的控制台信息。只能通过 WING 右下角的 Debug Console 交互式控制台输出变量，这点有些鸡肋。不过 Wing 支持断点调试 Python 而且支持查看 Python 的栈回溯，查看函数返回值等功能就可以取代传统的 print，没有 print 也无伤大雅（想要远程调试输出 print 可以根据官方文档修改对应环境配置）

![](https://bbs.pediy.com/upload/attach/202105/784955_V7S847FBG2PMCGR.jpg)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 14 小时前 被独钓者 OW 编辑 ，原因：