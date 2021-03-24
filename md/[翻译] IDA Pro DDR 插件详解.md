> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265227.htm)

1 DDR 概述
========

IDA 中的静态逆向工程通常会成问题。某些值是在运行时计算的，这使得很难理解某个基本块在做什么。如果您尝试通过调试一个恶意软件来执行动态分析，则该恶意软件通常会检测到并开始以不同的方式运行。思科 Talos 推出了动态数据解析器（DDR），该插件是 IDA 的插件，可简化反向工程恶意软件。DDR 正在使用检测技术在运行时从样本中解析动态值。

2 DDR 架构
========

DDR 具有如图所示客户端 / 服务器体系结构。DDR IDA 插件和 DDR 服务器是 Python 脚本。DynamoRio 客户端是用 C 编写的 DLL，它由 DynamoRIO 工具 drrun.exe 执行。该 DLL 使用检测技术在运行时分析和监视恶意软件样本。IDA 插件运行在 IDA 中，是前端；DDR 服务器运行在恶意代码机器上，是服务端。通常，所有分析过程都是通过插件控制的。后端 DynamoRIO 客户端分析完成后，将结果发送回插件。我们选择 JSON 作为此数据的格式，以使其更易于故障排除并易于由第三方 Python 脚本解析。  
![](https://bbs.pediy.com/upload/attach/202101/800470_FSMWKE67GNADKR5.png)  
从理论上讲，可以在同一台 PC 上运行插件和服务器，但是就执行恶意软件示例而言，强烈建议在单独的计算机上执行此操作。在大多数情况下，可以从 IDA 中的 DDR 菜单开始分析以收集动态值。下图显示了通用的工作流程。但是，如果要在无 Python 的系统上执行恶意软件，则也可以手动执行分析并单独运行 DynamoRio 客户端。  
![](https://bbs.pediy.com/upload/attach/202101/800470_NTTR5238EH7QNJ2.png)  
![](https://bbs.pediy.com/upload/attach/202101/800470_XD3X9QUKT6SH7WE.png)  
一旦在 IDA 日志输出窗口中看到插件已成功接收到 JSON 文件，就可以选择 “获取值...” 或“获取内存...”菜单之一来解析动态值或操作数。

3 程序安装
======

安装要求 Python 模块和 DynamoRIO 框架。  
DynamoRIO 框架地址：  
https://github.com/DynamoRIO/dynamorio/releases/tag/cronbuild-8.0.18547  
首先，在 “恶意软件 PC” 上安装 Python 3，并确保 “分析 PC” 上的 IDA 也使用 Python 3 设置。DDR 不支持 Python2。由于 IDA 早期版本中的错误，如果您想在 IDA <7.5 中运行 DDR 插件，则应在 IDA 计算机上安装 Python 3.7。DDR 服务器端仍可以是 Python 3.8 或 3.7。

3.1 原版安装
--------

将归档文件提取到 “恶意软件 PC” 上。最后，从归档文件根目录执行 DDR_INSTALLER.py 脚本。该脚本不会影响您现有的 Python 环境，它将为 DDR 创建专用的虚拟 Python 环境。它还没有任何非标准的 Python 模块依赖项。脚本在运行时会下载所有依赖项。它还会将 DynamoRio 框架安装到您的 DDR 安装目录中。该脚本将指导您完成 DDR 服务器和 DDR IDA 插件的整个安装过程。仔细阅读脚本的输出，其中包含许多其他提示，这些提示对于安装非常重要。  
运行 DDR_INSTALLER.py 脚本后按照提示进行安装。  
![](https://bbs.pediy.com/upload/attach/202101/800470_YB3AVWQ7XF47NTD.png)  
安装过程中服务端 IP 使用 “恶意软件 PC” 的 IP。  
安装完成后会要求启动 ddr_server.py 即服务端。  
在 DDR 目录将 ddr_plugin/ddr 和 ddr_plugin/ddr_plugin.py 放到 IDA 插件目录。  
开启 IDA 后即可使用 DDR 进行跟踪。

3.2 汉化版安装
---------

将 plugins 文件夹内文件放到 IDA 插件目录内。  
修改 ddr_config.json，修改 WEBSERVER 为服务端 IP(“恶意软件 PC”)；修改 CA_CERT 为 DDR 证书绝对路径；修改 DDR_WEBAPI_KEY 和服务端 key 一致；将服务端生成的 ddr_server.crt 证书放到 IDA 插件目录的 ddr 目录内。

4 功能介绍
======

4.1 查询 模块
---------

### 4.1.1 命令行参数

#### 4.1.1.1 设置命令行参数

eg：-a 1 -b “test string”  
![](https://bbs.pediy.com/upload/attach/202101/800470_Y37HFEHSY8P6QTY.png)

#### 4.1.1.2 显示命令行参数

![](https://bbs.pediy.com/upload/attach/202101/800470_X3QGDSCBJMJASUQ.png)

#### 4.1.1.3 清除命令行参数

![](https://bbs.pediy.com/upload/attach/202101/800470_9YE2H8V8HFXBR34.png)

### 4.1.2 中断地址

#### 4.1.2.1 设置中断地址

光标所在处设置为断点，只运行断点之前命令。  
![](https://bbs.pediy.com/upload/attach/202101/800470_2MJT8F6YHT59Q29.png)

#### 4.1.2.2 显示中断地址

#### 4.1.2.3 清除中断地址

### 4.1.3 起始偏移量

#### 4.1.3.1 设置起始偏移量

光标选定位置设置为本次追踪起始执行地址，类似于设定执行 esp。  
![](https://bbs.pediy.com/upload/attach/202101/800470_BZA3VFPU95QQ5Q2.png)

#### 4.1.3.2 显示起始偏移量

#### 4.1.3.3 清除起始偏移量

### 4.1.4 基本块

#### 4.1.4.1 添加基本块到基本块列表

光标选定基本块，将其添加到列表，在后续跟踪时可以选定跟踪基本块列表。  
![](https://bbs.pediy.com/upload/attach/202101/800470_KFJUFF5WJG6XNBZ.png)

#### 4.1.4.2 将该基本块从基本快列表删除

#### 4.1.4.3 显示基本块列表

#### 4.1.4.4 清除基本块列表

4.2 跟踪 模块
---------

### 4.2.1 对标记的地址范围进行完整跟踪

选定地址范围进行追踪。  
![](https://bbs.pediy.com/upload/attach/202101/800470_5PPCVPQVRYPFQPA.png)

### 4.2.2 对标记的基本块进行完整跟踪

对标记所在的整个基本块进行追踪。  
![](https://bbs.pediy.com/upload/attach/202101/800470_CBKRMSN2VEBCSHM.png)

### 4.2.3 对基本块列表进行完整跟踪

对标记的基本块列表进行跟踪。可通过显示基本块列表进行查看。

### 4.2.4 对段进行完整跟踪

对当前代码段进行完整跟踪。

### 4.2.5 对段进行轻跟踪

相对于完整跟踪。

4.3 转储 模块
---------

### 4.3.1 使用带标记的操作数来获取缓冲区大小

选定缓冲区大小。  
![](https://bbs.pediy.com/upload/attach/202101/800470_ZVCZF9UDNCDF8PP.png)

### 4.3.2 使用有标记的操作数来获取缓冲区地址

![](https://bbs.pediy.com/upload/attach/202101/800470_GBJVB2HG68GGKH7.png)

### 4.3.3 使用标记的地址转储缓冲区到文件

将设置的值转储到缓冲区文件。

### 4.3.4 执行样本和转储缓冲区

将缓冲区文件保存到文件。

4.4 补丁 模块
---------

### 4.4.1 以 NOP 填充指令运行

光标选定指令在运行是以 nop 指令填充执行。

### 4.4.2 在运行时切换 EFLAG

切换 EFlag 标志，再运行补丁可执行程序，则程序会加载另一条路。如在跳转时修改 SF 标志，则程序会执行另一条路。  
eg：程序不会执行标记块，修改 SF 标志位后则执行该块。  
![](https://bbs.pediy.com/upload/attach/202101/800470_EXDHXJDWSJBSZS2.png)  
![](https://bbs.pediy.com/upload/attach/202101/800470_9BJWGEZDRCKB5SX.png)

### 4.4.3 在运行时跳过标记地址的函数

选定后标记的函数不执行，执行补丁程序后可见。  
eg：跳过了 func1 函数。  
![](https://bbs.pediy.com/upload/attach/202101/800470_FF6G5T8CG2B92HY.png)  
![](https://bbs.pediy.com/upload/attach/202101/800470_U94HJBQMFQK9B5X.png)  
![](https://bbs.pediy.com/upload/attach/202101/800470_VKBKK7XG95433ZG.png)  
![](https://bbs.pediy.com/upload/attach/202101/800470_BHTHN638Y8PPGAE.png)

### 4.4.4 运行补丁可执行程序

执行创建的补丁可执行程序后。

4.5 调试 模块
---------

### 4.5.1 创建带有 bp 标记地址的 x64dbg 脚本

创建带有断点的 x64dbg 脚本。

### 4.5.2 在标记地址处创建循环及生成可执行文件

创建一个指令循环的补丁程序。  
修复参考 DDR 服务端提示。  
![](https://bbs.pediy.com/upload/attach/202101/800470_CE3YGSFPG6XV46W.png)

4.6 配置 模块
---------

### 4.6.1 设置 IDA DISASM 视图的跟踪次数

### 4.6.2 设置 IDA 日志窗口的跟踪次数

### 4.6.3 设置运行时 log 的指令数量

### 4.6.4 设置 API 超时的秒数

设置等待 DDR server 回复时间。  
![](https://bbs.pediy.com/upload/attach/202101/800470_Z8G2A83YBXPFU2V.png)

### 4.6.5 编辑 DDR API 请求 (仅限专家)

编辑 DDR API 请求格式。  
![](https://bbs.pediy.com/upload/attach/202101/800470_SJD6Z73EJ6HZ6RA.png)

4.7 寄存器 模块
----------

### 4.7.1 在 x?? 中获取指针的内存

光标选定地址，获取该地址条件下寄存器指针值。

4.8 获取源操作数 / 目的操作数的值
--------------------

X86 体系，源操作数通常是右边，目的操作数通常是左边。如果此时源操作数 / 目标操作数是指，则显示值；如果是地址，则通过 “获取源操作数 / 目标操作数中的指针值” 来对整个地址值进行获取，二级指针依此类推。  
![](https://bbs.pediy.com/upload/attach/202101/800470_8VXA68692BPW2KT.png)

4.9 获取源操作数 / 目标操作数中的指针值
-----------------------

光标选定地址，获取该指针值。  
![](https://bbs.pediy.com/upload/attach/202101/800470_XQRA33JUMXQNMTA.png)

4.10 获取源操作数 / 目标操作数中的二级指针值
--------------------------

光标选定地址，获取二级指针值。

4.11 获取源操作数 / 目标操作数中的指针内存
-------------------------

查看操作数内存值或指针值。  
![](https://bbs.pediy.com/upload/attach/202101/800470_R85UJ8RUXVRHB93.png)  
![](https://bbs.pediy.com/upload/attach/202101/800470_SJ652J3VRN6KUUB.png)

4.12 获取源操作数 / 目标操作数中的二级指针内存
---------------------------

获取操作数二级指针内存。

4.13 加载跟踪指令
-----------

加载跟踪模块执行后的结果。

4.14 关闭跟踪指令
-----------

关闭跟踪模块执行后的结果。

4.15 显示字符串跟踪
------------

![](https://bbs.pediy.com/upload/attach/202101/800470_S6UD2FM9MX9Y7X9.png)

4.16 显示 API 调用跟踪
----------------

![](https://bbs.pediy.com/upload/attach/202101/800470_7A3H9ACKZHV63HU.png)

4.17 删除 DDR 注释
--------------

光标选中地址，删除 DDR 注释。

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 2021-2-4 15:32 被 kanxue 编辑 ，原因：

上传的附件：

*   [DDR 汉化版. zip](javascript:void(0)) （28.65kb，20 次下载）