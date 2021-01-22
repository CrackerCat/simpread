> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/228981)

[![](https://p0.ssl.qhimg.com/t0185e45ada1356475f.jpg)](https://p0.ssl.qhimg.com/t0185e45ada1356475f.jpg)

0 目录
----

> 1 简易开发环境  
> 2 Hello world(GUI)  
> 3 Hello world(CMD)  
> 4 访问 DEX 内容  
> 5 访问 DEX 反编译内容  
> 6 访问 JAVA 语法树内容  
> 7 访问 Native 反汇编内容  
> 8 访问 Native 语法树内容
> 
> 9 其他常用操作  
> (1)DEX 交叉引用查询  
> (2)Native 调用查询  
> (3)Native 交叉引用查询  
> (4)Native 地址所属查询  
> (5) 内容搜索  
> (6) 访问 xml  
> (7) 异步任务  
> (8) 操作 ADB  
> (9) 接收用户输入  
> (10) 列表框  
> (11) 为某一脚本绑定快捷键  
> (12)NativeIR 中间码  
> (13) 反混淆 / 字节串解密用例  
> (14) 脚本中使用三方 JAR 包

1 简易开发环境
--------

(1) 安装 IDEA

(2) 在 IDEA 中插件市场搜索 Python 插件并安装

[https://plugins.jetbrains.com/plugin/631-python](https://plugins.jetbrains.com/plugin/631-python)

(3) 在 Jython 官网下载 Jython 并安装

[https://www.jython.org/download](https://www.jython.org/download)

(4) 打开 IDEA 创建简单一个 Java 工程, 并勾选 python 模块

将安装的 Jython 设置为该工程解释器路径

将’Jeb 根目录 \ bin\app\’下的 jeb.jar 文件拷贝到工程中, 并右键设置将该 jar 设置为库

最后在工程 src 中创建一个 hello.py 文件, 开始 编写代码

2 Hello world(GUI)
------------------

参考如图所示格式, run 函数为执行入口, 这里注意 py 的文件名和类名要一致, 像图中都为”hello”,

[![](https://p5.ssl.qhimg.com/t01a8b96f6d0e970c0d.png)](https://p5.ssl.qhimg.com/t01a8b96f6d0e970c0d.png)

然后在 JEB 中通过” 脚本 - 执行脚本”, 执行 hello.py 文件

[![](https://p4.ssl.qhimg.com/t01646919d8b28fc873.png)](https://p4.ssl.qhimg.com/t01646919d8b28fc873.png)

输出会出现在 JEB 的日志窗口中

[![](https://p2.ssl.qhimg.com/t01abf4734b767f1043.png)](https://p2.ssl.qhimg.com/t01abf4734b767f1043.png)

3 Hello world(CMD)
------------------

也可以脱离图形界面, 使用终端执行 py 脚本

java -jar “jeb 根目录 \ bin\app\jeb.jar” —srv2 —script=hello.py

(可在脚本中通过 ctx.open(path) 载入一个 apk 文件)

[![](https://p0.ssl.qhimg.com/t01208c167e54d0270e.png)](https://p0.ssl.qhimg.com/t01208c167e54d0270e.png)

[![](https://p2.ssl.qhimg.com/t014950703dbaf4ec86.png)](https://p2.ssl.qhimg.com/t014950703dbaf4ec86.png)

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/SampleScript.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/SampleScript.py)

4 访问 DEX 内容
-----------

### 常见操作

DEX 字节码层面操作, 关注 **IDexUnit** 部件, 常见操作如:

*   (1) 访问 DEX 与 Class  
    [https://github.com/acbocai/jeb_script/blob/main/samples/11%20IDexUnit-DEX%20Class.py](https://github.com/acbocai/jeb_script/blob/main/samples/11%20IDexUnit-DEX%20Class.py)
*   (2) 遍历 Field / Method  
    [https://github.com/acbocai/jeb_script/blob/main/samples/12%20IDexUnit-Field%20Method%20.py](https://github.com/acbocai/jeb_script/blob/main/samples/12%20IDexUnit-Field%20Method%20.py)
*   (3) 访问某个 Method  
    [https://github.com/acbocai/jeb_script/blob/main/samples/13%20IDexUnit-SingleMethod.py](https://github.com/acbocai/jeb_script/blob/main/samples/13%20IDexUnit-SingleMethod.py)
*   (4) 访问指令  
    [https://github.com/acbocai/jeb_script/blob/main/samples/14%20IDexUnit-Insn.py](https://github.com/acbocai/jeb_script/blob/main/samples/14%20IDexUnit-Insn.py)
*   (5) 访问基本块  
    [https://github.com/acbocai/jeb_script/blob/main/samples/15%20IDexUnit-BasicBlock.py](https://github.com/acbocai/jeb_script/blob/main/samples/15%20IDexUnit-BasicBlock.py)
*   (6) 访问控制流数据流  
    [https://github.com/acbocai/jeb_script/blob/main/samples/16%20IDexUnit-CFG.py](https://github.com/acbocai/jeb_script/blob/main/samples/16%20IDexUnit-CFG.py)

### 总结 JEB 脚本中的 DEX 访问

如果了解 DEX 结构, 应该不会对 class_def_item / code_item 等感到陌生, JEB 对这些结构进行了抽象, 每个结构都可以在定义中找到对应的类.

具体看一下 **com.pnfsoftware.jeb.core.units.code.android.dex** 包的文档, 它们都在这里.

其中有一部分实现了 IDexItem ICodeItem 接口, 如 IDexClass 和 IDexMethod, 这个接口的含义后面再说.

需要特别关注的是 **IDexCodeItem**, 指令相关内容 (CFG/BLOCK 等), 都在 IDexCodeItem 这个类中.

[![](https://p2.ssl.qhimg.com/t01428f73fea4a71b3d.png)](https://p2.ssl.qhimg.com/t01428f73fea4a71b3d.png)

### JEB 对编程概念的抽象 (ICodeItem)

常见编程语言中必不可少的概念, 如 class method field package call 等概念.

JEB 中通过 **ICodeItem** 这个顶层接口来概括他们 (DEX 层面为 **IDexItem**), 像类对应的 IDexClass, 方法对应的 IDexMethod, 都直接或间接 implements 了 ICodeItem 和 IDexItem 接口

[![](https://p3.ssl.qhimg.com/t01f4b1e6e64a3bf4c2.png)](https://p3.ssl.qhimg.com/t01f4b1e6e64a3bf4c2.png)

### 总结几个常用的类

DEX 类———————**IDexClass**  
DEX 方法——————**IDexMethod**  
方法指令序列————-**IDexCodeItem**  
指令————————**IDalvikInstruction**  
基本块———————**BasicBlock**  
控制流———————**CFG**

5 访问 DEX 反编译内容
--------------

DEX 反编译层面操作, 关注 **IDexDecompilerUnit** 部件

### 反编译一个 Class

参考:

[https://github.com/acbocai/jeb_script/blob/main/samples/21%20IDexDecompilerUnit-decompile.py](https://github.com/acbocai/jeb_script/blob/main/samples/21%20IDexDecompilerUnit-decompile.py)

### 如何获取 IDexDecompilerUnit?

若要获取 Class 反编译后内容 (像 IDA 的 F5), 仅使用 IDexUnit 不够啦, 还需要获取 **IDexDecompilerUnit** 部件.

如何获取 IDexDecompilerUnit?

一般通过 **DecompilerHelper** 类, 给它传入一个 IDexUnit 作为输入, 得到对应的 IDexDecompilerUnit 实例.

### 对类 / 方法 / 属性执行反编译

IDecompilerUnit 的 API 很容易理解, 执行反编译的目标可以是整个文件, 也可以是某个 class method 等, 如:

decompile()  
decompileAllClasses()  
decompileAllMethods()  
decompileClass()  
decompileMethod()  
decompileField()

参考:

[https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/code/IDecompilerUnit.html](https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/code/IDecompilerUnit.html)

之后通过 getDecompiled_*_Text(), 获取到反编译后的源代码 (String 文本).

### 个性化配置

在反编译前, 还可以通过 **DecompilationOptions** 和 **DecompilationContext** 等, 进行一些配置.

6 访问 JAVA 语法树内容
---------------

### 常见操作

*   (1) 遍历某 Method 所有 AST 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/31%20IJavaSourceUnit-DisplayAstTree.py](https://github.com/acbocai/jeb_script/blob/main/samples/31%20IJavaSourceUnit-DisplayAstTree.py)
*   (2) 解析 if…else 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/32%20IJavaSourceUnit-IJavaIf.py](https://github.com/acbocai/jeb_script/blob/main/samples/32%20IJavaSourceUnit-IJavaIf.py)
*   (3) 解析 call 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/33%20IJavaSourceUnit-IJavaCall.py](https://github.com/acbocai/jeb_script/blob/main/samples/33%20IJavaSourceUnit-IJavaCall.py)
*   (4) 解析 try 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/34%20IJavaSourceUnit-IJavaTry.py](https://github.com/acbocai/jeb_script/blob/main/samples/34%20IJavaSourceUnit-IJavaTry.py)
*   (5) 解析 for 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/35%20IJavaSourceUnit-IJavaFor.py](https://github.com/acbocai/jeb_script/blob/main/samples/35%20IJavaSourceUnit-IJavaFor.py)

### 访问 JAVA 语法树

如果打算访问 DEX 中方法对应的 AST 内容, 需要关注 **IJavaSourceUnit** 部件.

通过它可以拿到某个 CLASS 的 AST(**IJavaClass**), 接着访问 Class 内的其他内容.

### 访问 JAVA 语法树元素

如果访问某个具体的 AST 元素, 要进一步关注 **IJavaElement**,IJavaElement 是对 AST 元素的顶层抽象, 每个 AST 元素都是一个 IJavaElement.

比如 IJavaClass/IJavaIf/IJavaFor/IJavaNew, 这些元素都实现了 IJavaElement 接口.

### 获得 IJavaSourceUnit 或者 IJavaElement

需要先执行反编译的步骤, 参考这几个 API

ISourceUnit IDecompilerUnit.decompile(…)  
IJavaClass IDexDecompilerUnit.getClass(sign)  
IJavaClass IJavaSourceUnit.getClassElement()

### IJavaElement 的层次关系

元素细分成表达式 / 语句等, 语句又分复合语句 / 终结语句, 比如 for 和 if 是复合语句, return 和 throw 是终结语句, 具体关注 **com.pnfsoftware.jeb.core.units.code.java** 包的文档.

参考:

[https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/code/java/package-summary.html](https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/code/java/package-summary.html)

用一张图表达它们的层次关系：

[![](https://p5.ssl.qhimg.com/t01ac7648c2d4fcc6c7.png)](https://p5.ssl.qhimg.com/t01ac7648c2d4fcc6c7.png)

### 总结几个 Unit 的层次关系

DEX 方面

字节码层面 IDexUnit  
反编译层面 IDexDecompilerUnit  
语法树层面 IJavaSourceUnit

Native 方面类似, 可参考 DEX, 层次和过程是类似的.

字节码层面 INativeCodeUnit  
反编译层面 INativeDecompilerUnit  
语法树层面 INativeSourceUnit

参考:

[https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/package-summary.html](https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/package-summary.html)

[![](https://p4.ssl.qhimg.com/t01942e384dddb1d161.png)](https://p4.ssl.qhimg.com/t01942e384dddb1d161.png)

7 访问 Native 反汇编内容
-----------------

### 常见操作

原生库反汇编层面操作, 关注 **INativeCodeUnit** 部件, 常见操作如:

*   (1) 遍历函数  
    [https://github.com/acbocai/jeb_script/blob/main/samples/41%20INativeCodeUnit-Function.py](https://github.com/acbocai/jeb_script/blob/main/samples/41%20INativeCodeUnit-Function.py)
*   (2) 访问指令  
    [https://github.com/acbocai/jeb_script/blob/main/samples/42%20INativeCodeUnit-Insn.py](https://github.com/acbocai/jeb_script/blob/main/samples/42%20INativeCodeUnit-Insn.py)
*   (3) 访问基本块  
    [https://github.com/acbocai/jeb_script/blob/main/samples/43%20INativeCodeUnit-BasicBlock.py](https://github.com/acbocai/jeb_script/blob/main/samples/43%20INativeCodeUnit-BasicBlock.py)
*   (4) 访问控制流  
    [https://github.com/acbocai/jeb_script/blob/main/samples/44%20INativeCodeUnit-CFG.py](https://github.com/acbocai/jeb_script/blob/main/samples/44%20INativeCodeUnit-CFG.py)

8 访问 Native 语法树内容
-----------------

### 常见操作

过程和 DEX 类似, IJavaSourceUnit 换成了 **INativeSourceUnit**.

IJavaElement 换成了 **ICElement**.

*   (1) 遍历某函数所有 AST 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/51%20INativeSourceUnit-DisplayAstTree.py](https://github.com/acbocai/jeb_script/blob/main/samples/51%20INativeSourceUnit-DisplayAstTree.py)
*   (2) 解析 if…else 元素  
    [https://github.com/acbocai/jeb_script/blob/main/samples/52%20INativeSourceUnit-ICIfStm.py](https://github.com/acbocai/jeb_script/blob/main/samples/52%20INativeSourceUnit-ICIfStm.py)

### ICElement 的层次关系

具体关注 **com.pnfsoftware.jeb.core.units.code.asm.decompiler.ast** 包的文档.

参考:

[https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/code/asm/decompiler/ast/package-summary.htmll](https://www.pnfsoftware.com/jeb/apidoc/reference/com/pnfsoftware/jeb/core/units/code/asm/decompiler/ast/package-summary.html)

9 其他常用操作
--------

(1)DEX 交叉引用查询

查询 DEX 中方法的交叉引用信息, 使用 **ActionXrefsData** 和 **ActionContext**

[https://github.com/acbocai/jeb_script/blob/main/samples/61%20Dex-QUERY_XREFS.py](https://github.com/acbocai/jeb_script/blob/main/samples/61%20Dex-QUERY_XREFS.py)

(2)Native 调用查询

查询一个函数被谁调用了, 或查询它内部调用了谁，关注：

**INativeCodeAnalyzer**  
**INativeCodeModel**  
**ICallGraphManager**  
**ICallGraph**

[https://github.com/acbocai/jeb_script/blob/main/samples/62%20Native-Callees%20Callers.py](https://github.com/acbocai/jeb_script/blob/main/samples/62%20Native-Callees%20Callers.py)

(3)Native 交叉引用查询

查询一个 Function 的交叉引用情况 或查询一个 Block 入口指令的引用交叉引用情况

**INativeCodeAnalyzer  
INativeCodeModel  
IReferenceManager  
IReference**

[https://github.com/acbocai/jeb_script/blob/main/samples/63%20Native-CrossReference.py](https://github.com/acbocai/jeb_script/blob/main/samples/63%20Native-CrossReference.py)

(4)Native 地址所属查询

获取一条指令的地址，所属 Function 或 Block.

[https://github.com/acbocai/jeb_script/blob/main/samples/64%20Native-Contain.py](https://github.com/acbocai/jeb_script/blob/main/samples/64%20Native-Contain.py)

(5) 内容搜索

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/SearchAll.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/SearchAll.py)

(6) 访问 XML

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/ApkManifestView.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/ApkManifestView.py)

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/AndroidXrefResId.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/AndroidXrefResId.py)

(7) 异步任务

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/AsyncTask.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/AsyncTask.py)

(8) 操作 ADB

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/AdbDemo.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/AdbDemo.py)

(9) 接收用户输入

[![](https://p3.ssl.qhimg.com/t01cbb047c294dd3566.png)](https://p3.ssl.qhimg.com/t01cbb047c294dd3566.png)

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/RequestUserInput.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/RequestUserInput.py)

(10) 列表框

参考:

[![](https://p1.ssl.qhimg.com/t016f6fda916658feda.png)](https://p1.ssl.qhimg.com/t016f6fda916658feda.png)

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/WidgetList.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/WidgetList.py)

通过输入对话框和列表框, 实现书签列表

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/BookmarkSet.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/BookmarkSet.py)

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/BookmarkList.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/BookmarkList.py)

(11) 为某一脚本绑定快捷键

在脚本文件首行添加 #?shortcut=Mod1+Mod3+?

可在 GUI 中通过该组合快捷键触发脚本的执行, 其中 Mod1 代表 CTRL,Mod3 代表 ALT

[![](https://p5.ssl.qhimg.com/t0194e53dee6cc82d3c.png)](https://p5.ssl.qhimg.com/t0194e53dee6cc82d3c.png)

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/a313e4962fd535d0f70ad284bb0062ce84e5809e/scripts/BookmarkList.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/a313e4962fd535d0f70ad284bb0062ce84e5809e/scripts/BookmarkList.py)

(12)NativeIR 中间码

除了代码的语法树形态以外, Jeb 还支持一种中间码的形态,**IEStatement**

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/PrintNativeRoutineIR.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/PrintNativeRoutineIR.py)

(13) 反混淆 / 字节串解密用例

参考官方用例:

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/analysis/TriadaStringDecryptor.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/analysis/TriadaStringDecryptor.py)

[https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/analysis/WhatsAppStringDecryptor.py](https://github.com/pnfsoftware/jeb2-samplecode/blob/master/scripts/analysis/WhatsAppStringDecryptor.py)

(14) 脚本中使用三方 JAR 包

Jython 可以理解为通过 Python 语法来调用 Java 类和方法, 所以在某些场景如果不满足于 JEB 官方文档中提供的 API, 可使用反射等手段加载其他 jar 包, 增强解析能力.

(另外也可通过 Jython pip 扩展 python 类)

例如在 Jeb 中使用 soot flowDroid 类库

[![](https://p5.ssl.qhimg.com/t016f66a5c4e0839cce.png)](https://p5.ssl.qhimg.com/t016f66a5c4e0839cce.png)

参考文档:
-----

Scripting for Android Reversing

[https://www.pnfsoftware.com/jeb/manual/dev/android-scripting/](https://www.pnfsoftware.com/jeb/manual/dev/android-scripting/)

pnfsoftware/jeb2-samplecode/scripts/

[https://github.com/pnfsoftware/jeb2-samplecode/tree/master/scripts](https://github.com/pnfsoftware/jeb2-samplecode/tree/master/scripts)

JEB API Documentation

[https://www.pnfsoftware.com/jeb/apidoc/reference/packages.html](https://www.pnfsoftware.com/jeb/apidoc/reference/packages.html)

Author: acbc[@Vulpecker](https://github.com/Vulpecker "@Vulpecker")