> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2110319-1-1.html)

> [md]Qt QML 资源在 PE 文件中存储结构分析最近分析了一个 Qt QML 程序，第一次接触 Qt 应用，记录分享一下。开始以为它是个纯 Qt 程序，用 IDA 跟踪没有头绪，查了一下引入的动态 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)hfade0915 _ 本帖最后由 hfade0915 于 2026-5-31 07:08 编辑_  

Qt QML 资源在 PE 文件中存储结构分析  
最近分析了一个 Qt QML 程序，第一次接触 Qt 应用，记录分享一下。  
开始以为它是个纯 Qt 程序，用 IDA 跟踪没有头绪，查了一下引入的动态库发现是 Qt QML 应用，感觉从 QML 的存储机制入手更容易。  
  
![](https://attach.52pojie.cn/forum/202605/30/150535uw1nsm1onfsfoo1f.png)

**image_01.png** _(28.3 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcxNHw1MWQxZGRiNXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

下载了 Qt5 源码和 qt creator 开源版，写编了一个 QtQuick 的测试程序 qtquicktest.exe，就用它来分析 QML 在 PE 文件中的存储结构。  
首先通过 qtquicktest 工程输出目录中的 Makefile 和 Makefile.Debug 文件，大概了解了基本的构建机制。qt creator 用 rcc 将所有 qml 资源放到 qrc_qml.cpp 中，QML 资源就随 qrc_qml.cpp 一起被编译到 PE 中。  
qtquicktest 工程输出目录：  
  
![](https://attach.52pojie.cn/forum/202605/30/150537ezsdfusa0rbta6rb.png)

**image_02.png** _(84.25 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcxNXw0NzU0NTczMHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

  
  
  
下图是 Makefile.Debug 中 rcc 的命令行：  
  
![](https://attach.52pojie.cn/forum/202605/30/180109yf6gvcr17rg7gruf.png)

**bbb.png** _(17.08 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc2OHwxOWEzOGVmZHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 18:01 上传

  
  
  
qrc_qml.cpp 有 4 万多行，但实际代码非常少，可以看出所有 QML 资源被放到 qt_resource_data、qt_resource_name、 qt_resource_struct 三个数组中，排除这三个数组实际代码不到 100 行。  
以下是 qrc_qml.cpp 删除三个数组初始化数据的代码：  
  
![](https://attach.52pojie.cn/forum/202605/30/173916djyoeszz4ti877re.png)

**aaa.png** _(314.13 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc2NHwyNTNhOGY3YXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 17:39 上传

查看代码，QML 资源是由 qInitResources_qml 调用 qRegisterResourceData 加载的，代码如下：

![](https://attach.52pojie.cn/forum/202605/30/180107kdyiwddt5429i3q5.png)

**ccc.png** _(48.24 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc2N3w2YzE0MzA3ZHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 18:01 上传

  
  
从上述代码可以知道，找到 qInitResources_qml 函数就能定位 qt_resource_data、qt_resource_name、 qt_resource_struct 三个数组。  
用 IDA 反汇编 qtquicktest.exe，因为 qtquicktest.exe 带调试信息，可以直接在 Names 窗口搜索  
qt_resource_data，如下图：

![](https://attach.52pojie.cn/forum/202605/30/150549tlu8ilm8jebn0ink.png)

**image_06.png** _(20.21 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcxOXw5NDNjN2Y2ZnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

  
  
查看内容，三个数组中的内容和 qrc_qml.cpp 中初始化数组的数据一致。

![](https://attach.52pojie.cn/forum/202605/30/150552uxj8xl4l73mdnxlr.png)

**image_07.png** _(57.03 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyMHwyZjgyNTIyYXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

  
  
再看是谁调用了 qt_resource_data？发现两处：

![](https://attach.52pojie.cn/forum/202605/30/150554knlcznntrntn0nnn.png)

**image_08.png** _(26.61 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyMXw2MzE5MjQ0YXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

  
  
上述结果和 qrc_qml.cpp 源码是一致的。  
查看 qInitResources_qml(void) 函数，汇编代码如下：

![](https://attach.52pojie.cn/forum/202605/30/150556zfk94464yfsk6qeh.png)

**image_09.png** _(102.64 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyMnw2MWUxNjljOXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

用 IDA 跟踪 qInitResources_qml，也能定位到 qrc_qml.cpp  
![](https://attach.52pojie.cn/forum/202605/30/150559oloegl5g89ulzwl4.png)

**image_10.png** _(100.03 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyM3w0OTc5MzhlY3wxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:05 上传

  
  
  
单步进 qRegisterResourceData 函数，可以看到它的源码在..\qtbase\src\corelib\io\qresource.cpp1080 行。

![](https://attach.52pojie.cn/forum/202605/30/150602br4q7n7irmdm7x8q.png)

**image_11.png** _(94.16 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyNHxjMTM5MzJmNHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

  
  
qRegisterResourceData 函数的参数 tree、name、data 分别对应 qt_resource_struct、 qt_resource_name、 qt_resource_data，搞清楚这三个指针指向的内容，就可以继续分析 QML 在 PE 中的存储结构了。

![](https://attach.52pojie.cn/forum/202605/30/150605pdfqouo75doh1afz.png)

**image_12.png** _(111.06 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyNXxjYTMxYmE4MnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

![](https://attach.52pojie.cn/forum/202605/30/150607cw1vybkez6fvvb3e.png)

**image_13.png** _(94.26 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyNnwxNDQ1NzJjNHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

![](https://attach.52pojie.cn/forum/202605/30/150610qss494y3uy7svv0b.png)

**image_14.png** _(111.67 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyN3xkN2NjMzIwNHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

  
  
上面三图显示了这三个指针指向的内容以及内容加载后的虚拟地址和 PE 文件的偏移。  
根据上面获取的信息结合 qrc_qml.cpp 和 qresource.cpp 源代码，可以分析出 QML 这三块数据的存储结构，总结如下（如果不做说明字节序都是大端的）：  
  
一、qRegisterResourceData 的参数解释

<table><thead><tr><th></th><th></th><th></th><th></th><th></th></tr></thead><tbody><tr><td>参数</td><td>传参寄存器</td><td>PE 文件偏移</td><td>含义</td></tr><tr><td>version</td><td>rcx</td><td></td><td>资源格式版本号，支持 1-3，当前为 3</td></tr><tr><td>qt_resource_struct</td><td>rdx</td><td>0xBF190</td><td>资源树结构数组，描述目录 / 文件层级</td></tr><tr><td>qt_resource_name</td><td>r8</td><td>0xBE560</td><td>资源名称字符串表</td></tr><tr><td>qt_resource_data</td><td>r9</td><td>0x203F0</td><td>资源文件实际数据</td></tr></tbody></table>

  
二、三块数据的内存布局  
1、qt_resource_name（name）  
qt_resource_name 是资源节点名称表，在 qrc_qml.cpp 中定义如下：  

![](https://attach.52pojie.cn/forum/202605/30/191441m0jwaq0rx3zkaf30.png)

**ddd.png** _(75 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc3NXw1NzBlOWFlNnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 19:14 上传

  
  
在 PE 文件的内容如下：

![](https://attach.52pojie.cn/forum/202605/30/150615vikhak7jy1wbiawm.png)

**image_16.png** _(28.88 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyOXwzODk2YTgyZnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

  
  
从 qrc_qml.cpp 的定义中可以知道 qt_resource_name 中定义了一些字符串，每个字符串 4 行，比如 “Manager.qml” 就是：  
0x0,0xb,  
0xd,0x62,0x60,0xdc,  
0x0,0x4d,  
0x0,0x61,0x0,0x6e,0x0,0x61,0x0,0x67,0x0,0x65,0x0,0x72,0x0,0x2e,0x0,0x71,0x0,0x6d,0x0,0x6c,  
每个字符串分三部分：2 字节长度、4 字节字符串 hash、UTF-16LE 格式的字符串。上图用不同颜色显示字符串的三部分，蓝色是 2 字节的长度 11，绿色是 hash：0xD6260DC ，黑色就是 “Manager.qml”11 个字符。  
  
2、qt_resource_data（data）  
qt_resource_data 包含了 QML 资源树中节点文件的内容，在 qrc_qml.cpp 中定义如下：

![](https://attach.52pojie.cn/forum/202605/30/192733bku5hju2zppu7ceu.png)

**eee.png** _(84.7 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc3NHw1MzVkODM2MHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 19:27 上传

PE 文件的内容如下：

![](https://attach.52pojie.cn/forum/202605/30/150620oqi1ixnpzg1s61kr.png)

**image_18.png** _(62.77 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczMXxhYzFiODhmY3wxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

Manager.qml 源文件的开始几行如下：

![](https://attach.52pojie.cn/forum/202605/30/191437hyr1fwz3washwy04.png)

**fff.png** _(35.5 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc3M3w0ZDYyMjVlM3wxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 19:14 上传

上面三个图展示了 qt_resource_data 中的第一组数据就是 Manager.qml 文件的内容，每组数据分两部分，前 4 个字节是内容长度（data_length），后面就是文件内容。从 PE 文件的内容中我们可以看到 0x000007A5（十进制 1957）是 Manager.qml 的大小，后面就是 Manager.qml 文件的内容（EF BB BF 是 BOM 头表示文件是 UTF-8 编码）。这里需要注意，如果文件是压缩的结构还是一样，data_length（前 4 个字节）是后续内容的长度，但后续内容的前 4 个字节是压缩文件解压后的长度，再后面是被压缩的数据。  
![](https://attach.52pojie.cn/forum/202605/31/065649fk4vfzzlmf24l32l.png)

**hhh.png** _(138.32 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDgzOXxjMGU5M2QyZHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-31 06:56 上传

  
上述代码说明 data_length - sizeof(quint32) 是实际的压缩数据长度。所以当是压缩文件时，data_length = 解压后大小（4 字节）+ 被压缩数据的大小。

Manager.qml 的大小 1957，同文件管理器中显示的一致：

![](https://attach.52pojie.cn/forum/202605/30/150624ej67yz3k6ik1z7y1.png)

**image_20.png** _(24.66 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczM3xhODM1M2QyNnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

qt_resource_data 内存布局如下：

<table><thead><tr><th></th><th></th><th></th></tr></thead><tbody><tr><td>内容长度（data_length）</td><td>内容</td><td></td></tr><tr><td>4B</td><td>文件内容</td><td></td></tr><tr><td>…</td><td></td><td></td></tr><tr><td>4B</td><td>4B 解压后大小</td><td>压缩数据</td></tr><tr><td>…</td><td></td><td></td></tr></tbody></table>

3、qt_resource_struct（tree）  
qt_resource_struct 是描述 QML 资源树节点的‘记录‘数组，一条记录对应一个节点，每条记录 22 个字节。要得到每个节点的信息，一定是先获取 qt_resource_struct 中节点的描述记录，再根据节点记录在 qt_resource_name 和 qt_resource_data 中得到节点名和内容。  
qrc_qml.cpp 中 qt_resource_struct 定义如下：

![](https://attach.52pojie.cn/forum/202605/30/193434m0gmfi50pmm54ifh.png)

**image_15.png** _(101.5 KB, 下载次数: 2)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDcyOHw5ZWU1YjhiMnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 19:34 上传

下图是 PE 中的 qt_resource_struct 内容，展示了开始的 8 条记录：

![](https://attach.52pojie.cn/forum/202605/30/150629oal0u0936n0qho0p.png)

**image_22.png** _(39.79 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczNXwzZmY0Y2I1Y3wxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

qt_resource_struct 在内存布局如下：

<table><thead><tr><th></th></tr></thead><tbody><tr><td>记录 1（22 字节）</td></tr><tr><td>记录 2（22 字节）</td></tr><tr><td>记录 3（22 字节）</td></tr><tr><td>…</td></tr><tr><td>记录 n（22 字节）</td></tr><tr><td>…</td></tr></tbody></table>

记录的字段含义：

<table><thead><tr><th></th><th></th><th></th><th></th><th></th></tr></thead><tbody><tr><td>偏移</td><td>字节数</td><td>名称</td><td>第一条记录</td><td>含义</td></tr><tr><td>0x00</td><td>4</td><td>nameOffset</td><td>00 00 00 00</td><td>指向节点名在 qt_resource_name 中的偏移，0 表示第一个字符串：Manager.qml</td></tr><tr><td>0x04</td><td>2</td><td>flags</td><td>00 02</td><td>节点的属性：&nbsp;&nbsp;bit0：zlib 压缩、bit1：是否目录、bit2：zstd 压缩&nbsp;&nbsp;0x0002 表示该节点是目录。</td></tr><tr><td>0x06</td><td>4</td><td>locale</td><td>00 00 00 07</td><td>地区和语言</td></tr><tr><td>0x0A</td><td>4</td><td>dataOffset</td><td>00 00 00 01</td><td>如果是文件：指向该节点内容在 qt_resource_data 表的偏移&nbsp;&nbsp;如果是目录：表示第一个子节点的索引（从 1 开始）</td></tr><tr><td>0x0E</td><td>8</td><td>lastModified</td><td>00 00 00 00 00 00 00 00</td><td>最后修改时间，目录一般是 0。</td></tr></tbody></table>

以上就是 QML 资源在 PE 文件存储的细节，根据这些信息就可以编写工具任意提取或替换 PE 文件中的 QML 资源了。我在 github 找到一个提取 QML 的工具 qtextract，如果你只想提取 QML 资源用这个工具应该够用。qtextract 通过静态扫描给 qRegisterResourceData 函数传递 qt_resource_struct、qt_resource_data、qt_resource_name 三个参数的指令，通过分析指令提取这三块数据在 PE 中的位置。由于不同版本、不同环境编译的 Qt 应用传参指令有差异，qtextract 扫描指令不够智能，有些指令组合不能自动识别，像我编译的 qtquicktest.exe 就无法用 qtextract 自动提取 QML 资源。但 qtextract 有高级模式，手工分析得到这三个偏移后用 --data 或 --datarva 命令行参数告诉 qtextract，qtextract 就能提取 QML 资源。所以要处理 QML 资源，无论是用现成的工具还是自己开发工具，都需要定位 qt_resource_struct、qt_resource_data、qt_resource_name 这三块数据的位置。要定位这三块数据，本质上就是要找 qInitResources_qml 函数。下面来看看在不带调试信息的 Qt QML 应用中如何定位 qInitResources_qml 函数？

三、定位 qInitResources_qml 函数

因为 qInitResources_qml 调用 qRegisterResourceData 函数，qRegisterResourceData 是 Qt 动态库引出的函数，所以只要在引入表搜索 qRegisterResourceData 找到 qRegisterResourceData 引入项，就可以找到 qInitResources_qml 函数。  
在 IDA 引入窗口中找到 qRegisterResourceData

![](https://attach.52pojie.cn/forum/202605/30/150632xnqtvkbq4i34qv6a.png)

**image_23.png** _(32.27 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczNnw2ZGJjNGFiZXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

再双击它跳转到引入表

![](https://attach.52pojie.cn/forum/202605/30/150634vujlbtbydrxsgcrd.png)

**image_24.png** _(29.04 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczN3w3OWVlMzdjZXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

再逐级追溯调用者就能定位到 qInitResources_qml 函数

![](https://attach.52pojie.cn/forum/202605/31/051531ebpybr9igzehioiy.png)

**image_25.png** _(98.28 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczOHwzMDM2N2ZiNnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-31 05:15 上传

上图中的 sub_140001130 就是 qInitResources_qml 函数，这样我们就得到 qt_resource_struct、qt_resource_data、qt_resource_name 三个数据块在 PE 中的偏移和虚拟地址。  
你可以比较一下 sub_140001130 和前面分析的 qtquicktest.exe 中的 qInitResources_qml 在给 qRegisterResourceData 函数传参指令上的细微差异（mov ecx,xxx 的位置和形式不同），更好理解 qtextract 为什么不能自动扫描 qtquicktest.exe 的 QML 资源。  
  
![](https://attach.52pojie.cn/forum/202605/30/150639pf7s7q7bq12m99sz.png)

**image_26.png** _(91.43 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDczOXw0ZmU3MzZmMXwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

![](https://attach.52pojie.cn/forum/202605/30/150642qzeelo9zeopl6lzq.png)

**image_27.png** _(123.21 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc0MHxiNDA4YjEwZnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

![](https://attach.52pojie.cn/forum/202605/30/150644zrkvbss6fs8nr6f4.png)

**image_28.png** _(103.28 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDc0MXxjNjZiMWIwZnwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-30 15:06 上传

根据上述三个图的内容，很容易确定  
byte_1401C5A60 是 qt_resource_data，文件偏移是 0x1c4260  
byte_14026A4F0 是 qt_resource_name，文件偏移是 0x268cf0  
byte_14026B4C0 是 qt_resource_struct，文件偏移是 0x269cc0  
这样就完成 Qt QML 资源的三块数据的定位。  
  
  
下面以 qt_resource_struct 的第四条记录为例，看看三个数组的对应关系：  
  
  
![](https://attach.52pojie.cn/forum/202605/31/061819h7la3zmd7d9ixdm8.png)

**ggg.png** _(180.1 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjg1NDgzNnw2MWIyZjZiMHwxNzgwNTM2NjQ5fDIxMzQzMXwyMTEwMzE5&nothumb=yes)

2026-5-31 06:18 上传

  
从上图的数据中可以知道，第四条记录是一个未压缩的文件，名称为：main.qml，最后修改时间是：2026-03-02 05:35:46（00 00 01 9C AD 0B A0 0F）文件大小是 0x1851（6225）  
前几行代码是：  
[C] _纯文本查看_ _复制代码_

```
import QtQuick 2.15
import QtQuick.Window 2.15
import Qt.labs.platform 1.1
import QtQuick.Controls 2.15
import Clipboard 1.0

```

  
到此基本搞清楚了 QML 在 PE 中的存储结构。对于开启了 QML 缓存的应用，要让导入的 QML 资源起作用还需要搞清楚怎样关闭缓存。开始我不清楚这些，导入了 QML 资源后怎么运行都不起作用，浪费了不少时间。关于缓存的内容下次再找时间贴出来，主要讲一下怎样查找 QML 节点对应的缓存。缓存的信息在官网文档提到一些：[https://doc.qt.io/archives/qt-5.15/qtquick-deployment.html](https://doc.qt.io/archives/qt-5.15/qtquick-deployment.html)。![](https://avatar.52pojie.cn/data/avatar/000/08/22/58_avatar_middle.jpg)冥界 3 大法王 我代表组织给你加分！