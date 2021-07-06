> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268356.htm)

> [原创] GhidraDec 编译测试

**GhidraDec 编译测试**
==================

**摘要：**本文对 GhidraDec 进行编译和测试，以了解 GhidraDec，基本手段是解决编译过程可能遇到的问题和测试对比该插件与原生 Ghidra 反编译效果，无甚特色。

**关键词：**Ghidra，IDA，RetDec，反编译

1、介绍  

-------

    GhidraDec 是一款用于 IDA Pro 的插件，主要目的是在 IDA 中使用 Ghidra 的反编译器（decompiler.exe），即类似 IDA 的 F5 伪码反编译功能，并非反汇编功能。由于 Ghidra 9.x 和 10.x 版本间的协议差异，GhidraDec 只支持 Ghidra  10.x 版本的反编译器，兼容 IDA 6/7x 版本。支持 Ghidra 和 IDA 所支持的处理器。更多信息参考官方说明：

    (1) 官仓： [https://github.com/GregoryMorse/GhidraDec](https://github.com/GregoryMorse/GhidraDec)

    (2) Ghidra 支持的处理器列表： [https://github.com/NationalSecurityAgency/ghidra/tree/master/Ghidra/Processors](https://github.com/NationalSecurityAgency/ghidra/tree/master/Ghidra/Processors)

    (3) IDA 支持的处理器列表： https://hex-rays.com/products/ida/processors/

**2、编译**
--------

### **2.1 编译要求  
**

    这里主要为 Windows 10 x64 的 IDA Pro 7.5 编译 GhidraDec 插件

    （1）由于是 IDA Pro 7.5 的插件，别无二选是 IDASDK75；

    （2）参考 IDASDK75\readme.txt，这里我们选择 Visual Studio 2017；   

    （3）这里我们使用 win_flex_bison，版本尽量 2.x 以上（参考附件）；  

    （4）这里我们使用 VC 编译环境，搭配 cmake 进行编译；  

    （5）git 是搬运 github 代码必备。  

**![](https://bbs.pediy.com/upload/attach/202107/760675_UB4JE5S2SUHD27M.jpg)**

图 2-1 官方说明的编译要求

### **2.2 编译**

  我们主要通过 cmd.exe 命令行完成编译。

#### **2.2.1 编译环境设置**

  （1）启动 cmd.exe，初始化 Visual Studio 2017 x64 编译环境；

```
%comspec% /k "C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\VC\Auxiliary\Build\vcvars64.bat"

```

    （2）添加 cmake、win_flex_bison、git 路径到 path 环境变量；  

```
set path=D:\ProgramData\CMake\bin;D:\win_flex_bison;D:\ProgramData\Git\cmd;%path%

```

    （3）搬运 GhidraDec 和 Ghidra 代码；  

        说明，搬运 Ghidra 主要是使用 Ghidra\Features\Decompiler\src\decompile 目录的源码，本次编译测试中需要修正 decompiler.cpp 源码，附件提供了修正版，如果使用附件的 decompile 目录代码，也可以忽略下载 ghidra 代码；虽然 GhidraDec 有 decompile9 源码，考虑到官方提到的只支持 Ghidra10.x，这里直接使用目前最新的 10.x 版本源码，至于直接修改 decompile9 为 decompile 行不行，这里未作测试。

```
git clone https://github.com/GregoryMorse/GhidraDec.git
git clone https://github.com/NationalSecurityAgency/ghidra.git
cd ghidra
git checkout
#rem 然后将Ghidra\Features\Decompiler\src\decompile目录复制到GhidraDec目录下，或直接使用附件decompile目录源码

```

    注意：如果环境除了 win_flex_bison 外，也有其他 bison，为避免其他版本可能较低或是其他环境专属带来的干扰，  

需要修改 GhidraDec\CMakeLists.txt 的下述行内容，挑战 win_bison 的为优先查找顺序。

![](https://bbs.pediy.com/upload/attach/202107/760675_75NC6MNCCX4JTND.jpg)

图 2-2 设置 win_bison 优先于其他版本 bison

#### **2.2.2 编译安装  
**

    （1）cmake 配置和编译  

        应当注意到，下述 - DIDA_SDK_DIR 应设置为实际的 IDASDK75 路径。

```
cd GhidraDec
mkdir build
cd build
cmake .. -DIDA_SDK_DIR="D:\Program Files\IDA 7.5 SP3\plugins\idasdk75" -G "Visual Studio 15 2017 Win64"
 
cmake --build . --config Release

```

    （2）可能出现的错误与解决方案

        A、"unrecognized: %destructor" 错误。这一般是由于 bison 版本过低造成，换较高版本 bison 解决。

        B、error C2398: 元素 “2”: 从“const color_t” 转换到“_Elem”。这是编译器太高端所致，这里需要修改 decompiler.cpp 的源码进行排错。

```
#问题在于 D:\Program Files\IDA 7.5 SP3\plugins\idasdk75\include\lines.hpp 定义的 color_t 为uchar,而 COLOR_ON 默认为char。
std::string({ COLOR_ON, COLOR_AUTOCMT })
 
#这里通过static_cast 将所有引起C2398错误的地方修正
std::string({ COLOR_ON, static_cast(COLOR_AUTOCMT) }) 
```

![](https://bbs.pediy.com/upload/attach/202107/760675_2HCKZ6G5BXWWXPU.jpg)

图 2-3 混合 char 与 uchar 引起的 C2398 错误

    （3）将 GhidraDec\build\Release 目录下的 GhidraDec 插件 ghidradec.dll、ghidradec64.dll 复制到 IDA 的 plugins 目录即可完成安装。

        A、设置 decompiler.exe 的目录。选择 “Options|GhidraDec plugin opitons" 进行配置，然后选定 ghidra 的安装目录，  

    ![](https://bbs.pediy.com/upload/attach/202107/760675_A5R9YDM2TJNFGRZ.jpg)

图 2-4 GhidraDec 配置选项菜单

![](https://bbs.pediy.com/upload/attach/202107/760675_YFSWB4GVS79Q5YA.jpg)

图 2-5 GhidraDec 配置中选定 ghidra 安装目录

![](https://bbs.pediy.com/upload/attach/202107/760675_8AWTCDJ2XFGXNPT.jpg)

图 2-6 通过 "Edit|Plugins|Ghidra Decompiler Plugin" 或快捷键 "Ctrl+G" 进行反编译调用

**3、测试**
--------

 这里我们对 ghidradec.dll 插件本身进行反编译测试。

### **3.1 Ghidra 与 GhidraDec 对比**

 如图 3-1 所示，其左侧是 Ghidra 的反编译展示，右侧是 IDA Pro 的 GhidraDec 插件对同一函数反编译展示。总体而言，函数体反编译的结构基本一致，但 GhidraDec 很多时候会无法反编译且导致 IDA 奔溃，且反编译结果没法直接重命名修改，这就非常鸡肋。

 **![](https://bbs.pediy.com/upload/attach/202107/760675_W6GEG4UJG56EDG8.jpg)** 

图 3-1 Ghidra 与 IDA 的 GhidraDec 反编译比对

### 3.2 GhidraDec 原理

    如下是上述行数在 GhidraDec 反编译时输出的日志信息，可见:

    A、GhidraDec 先对 IDA 的 idb 数据库 ghidradec.dll.i64 存副本为 ghidradec.dll.dec-backup.i64，

    B、然后根据 IDA 反汇编结果转换为 IR（llvmIr）存为 ghidradec.dll.i64.json，

    C、最后通过 ghidra 的反编译器 decompile.exe 转换为高级伪码。

```
[GhidraDec info]   :    Found decompile.exe at D:\ghidra_10.0_PUBLIC_20210621/Ghidra/Features/Decompiler/os/win64/ -> plugin is properly configured.
[GhidraDec info]   :    Working on input file "D:\GhidraDec\build\Release\ghidradec.dll".
[GhidraDec info]   :    Running Ghidra decompiler plugin:
[GhidraDec info]   :    Saving IDA database ...
[GhidraDec info]   :    IDA database saved into :  D:\GhidraDec\build\Release\ghidradec.dll.dec-backup.i64
[GhidraDec info]   :    Generating retargetable decompilation DB ...
[GhidraDec info]   :    Retargetable decompilation DB saved into: D:\GhidraDec\build\Release\ghidradec.dll.i64.json
[GhidraDec info]   :    Decompile input ...
[GhidraDec info]   :    Decompilation command: D:\ghidra_10.0_PUBLIC_20210621/Ghidra/Features/Decompiler/os/win64/decompile.exe
[GhidraDec info]   :    Running the decompilation command ...
[GhidraDec info]   :    Detected Processor spec: D:\ghidra_10.0_PUBLIC_20210621/Ghidra/Processors\x86/data/languages/x86-64.pspec Compiler spec: D:\ghidra_10.0_PUBLIC_20210621/Ghidra/Processors\x86/data/languages/x86-64-win.cspec Sleigh file: D:\FTP\CTF2021\newtools\ghidra_10.0_PUBLIC_20210621/Ghidra/Processors\x86/data/languages/x86-64.sla
[GhidraDec info]   :    Local decompilation ...
[GhidraDec info]   :    Decompiling function: sub_18009B070 @ 0x18009b070
[GhidraDec info]   :    Decompilation completed: sub_18009B070 in 1 seconds

```

### **3.3 Ghidra、GhidraDec、IDA's F5 差异**

    A、Ghidra 采用自身的反汇编器，生成自己的 IR(llvmIr)，然后再用 decompile.exe 反编译器反编译为高级伪码。

```
bin --> Ghidra-CPU-disassembler --> Ghidra-disasm --> llvmIr --> decompiler.exe --> pesudo-code

```

    B、IDA's F5 采用自身的反汇编器，生成自己的 IR(微码），然后再用自己的反编译器对 IR 反编译为高级伪码。

```
bin --> IDA-CPU-disassembler -->  IDA-disasm --> IDA-mIR --> IDA-decompiler --> pesudo-code

```

    C、GhidraDec 是利用 IDA 反汇编器的输出结果，生成 ghidra 的 IR(llmvlr), 再用 decompile.exe 反编译器反编译为高级伪码。

```
bin --> IDA-CPU-disassembler --> IDA-disasm --> llvmIr --> decompiler.exe --> pesudo-code

```

**4、总结**
--------

 目前看来，GhidraDec 只是作为打通 IDA 和 ghidra 之间通道的一种尝试，可以作为一种思路参考。在 IDA-diasm 到 llvmIR 转换的环节缺陷，导致偶发性的无法反编译和奔溃，GhidraDec 返回给 IDA 的反编译结果无法如 IDA 或 ghidra 原生环境中都可以进一步修改。纵然有许多缺陷，实用性上也比较鸡肋，但不妨作为融合 IDA 和 ghidra 两者优势的试验典型。

**附件说明：**

[1] win_flex_bions-latest.zip 为找到 Windows 的 2.7 版；  

[2] decompile.zip 为 ghidra 对应目录的源码，这里主要修正了 decompiler.cpp 可能导致的 C2398 编译错误；

[3] GhidraDec_x64Release.zip 为编译 GhidraDec 插件，要正常运行，还需要安装 ghidra 10.x。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 16 小时前 被 tritium 编辑 ，原因：

[#其他内容](forum-4-1-10.htm)

上传的附件：

*   [win_flex_bison-latest.zip](javascript:void(0)) （691.36kb，4 次下载）
*   [decompile.zip](javascript:void(0)) （1.46MB，3 次下载）
*   [GhidraDec_x64Release.zip](javascript:void(0)) （1.04MB，3 次下载）