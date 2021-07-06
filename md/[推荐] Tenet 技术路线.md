> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267179.htm)

> [推荐] Tenet 技术路线

【摘要】

    大前天（2021-4-20)林版在群里分享了 Tenet 的链接，链接页面的动态演示有种 “大杀器” 的感觉，于是做了跟进。这是基于 IDA7.5 和 python3 的对 Trace 人性化浏览工具，但需要使用者根据格式自己制作 Trace 记录，虽有样例，还是需要一定的折腾。整体感觉比较新奇，就技术的试验性版本而言，还是比较惊艳。如作者总结的【在此文中，我们展示了一款称为 Tenet 的 IDA Pro 插件。这是一个试验性的插件，设计来查览软件执行跟踪记录。它能用来当作逆向工程过程的一个辅助，也能用来当作在程序分析、虚拟化、下一代调试实践的有效性（等）的探索中使用的研究技术。我们对这些技术的开发实践是首屈一指的。...（省略涉及技术合作部分，参考翻译详情）】。

【关键词】

    态势感知

    时间无关调试 timeless debugging

【技术提要】

    可以解决什么？  

    （1）基于时间线的跟踪记录，理论上可以定位任意时刻的执行状态，所以也可以随意推进和回溯程序执行过程。  

    （2）某个指令，在整个运行周期（时间线）上那些时刻执行了该指令，并可以在不同时刻间来回切换。  

    （3）某个寄存器的当前值是在什么时候设置的，在整个时间线上对某个寄存器赋值操作指令的定位来回切换。  

    （4）某个内存字节（或区域），在整个时间线上，那些时刻进行了读、写、访问，并可以直接在不同时间点切换。  

    这是一个什么技术？  

        如果没有理解错，按作者的意思，调试技术很多都是建立在某一个时刻的程序执行状态下的。而这个技术是建立在所有状态之间的关系上进行辅助研究分析。  

【难点】

    插件安装比较简单，github 下载拖放进 IDA7.5 即可。

    但使用时，需要加载 “执行跟踪记录”（简称记录）；这些记录是需要通过其他工具进行捕获收集的。记录的格式作者在 README 中做了说明，也提及一些工具，这里复现作者使用 Intel 的 Pin 开发的 Pintool 插件来展示记录的提取。这个过程我也像作者一样是试验性的，但试验后难免会有些骨感。对简单的程序还好（如简单的 HelloWord console 应用，线程少），对于复杂点的程序（如 HellowWord MFC，线程多），几乎绝望，诸君亲测下，若发现更好的获取收集” 执行跟踪记录 “的方式，还请不吝分享一二。

Tenet 链接

```
#python3
import base64
base64.b64decode(b'aHR0cHM6Ly9ibG9nLnJldDIuaW8vMjAyMS8wNC8yMC90ZW5ldC10cmFjZS1leHBsb3Jlci8=')

```

本文分两部分，第一部分是对官方的摘译，第二部分主要是复现 Intel 的 Pin 采集” 执行跟踪记录 “，以展示插件的使用。

一、官方摘译（意译，若缪请斧正）

    【前言】  

    Tenet ：一款跟踪记录查览器（供逆向工程师使用）  

    传统的调试器变得越来越复杂，该何去何从？  

    传统意义上，调试是一种枯燥乏味尝试。虽然一些大侠热衷于这种类似考古的过程（使用调试器来发掘软件缺陷或在逆向工程中执行任务），  

但几乎所有人都会觉得相对于现代软件，这些工具开始显得力不从心。

    我们运行时探查的方法仍然高度聚焦在个别的执行状态。这种思路存在明显的缺陷，源于变得越来越复杂的软件要求更多的情景和 “态势感知” 才能正确解读。时间无关调试（timeless dubgging）的出现虽减轻了些许日渐增加的痛楚，但这些方案依然是建立在探查个别状态的传统方法上，而不是建立在各个状态之间是如何关联的基础上。

    我（Tenet 作者，gaasedelen）开源了 Tenet，一个对执行跟踪记录的执行探查浏览的 IDA Pro 插件。此插件的目的是，（对某个给定的二进制代码的执行跟踪记录）提供一种更自然、人性化操作的导览方案。此开源工作的基础源于一种渴望，一种探索新的或革命性的（在软件中检测和提取复杂执行模型的）方法的渴望。

![](https://bbs.pediy.com/upload/attach/202104/WTN87ZM9WU82AEG.jpg)

Tenet 是一个实验性的用于查览软执行跟踪记录的 IDA Pro 插件     

    【背景】  

    Tenet 的灵感来自 QIRA （译注：QIRA 是一款时间无关调试器）。我对它最初的原型的开发时间可以追溯倒 2015 年，那时我作为微软的一名软件安全工程师。这些原型建立在为 TTD 设计的不公开的 “跟踪记录阅读器” 应用接口（API）之上（译注：TTD，微软官方 Time Travel Debugging 缩写，是一个可以记录执行进程的执行，再后续可以向前或向后回放的工具。TTD 允许回撤调试会话。）；TTD 以前称为 TTT 或 iDNA。

    我辞掉了微软的工作，因为我不觉得，在（Window 被称为 “开源软件” 的世界）这样的一个公司里，一个汇编层的跟踪记录查览器会有意义。（译注：作者可能是反讽，众所周知，虽然有部分开源项目，但 Windows 并不算 “开源软件”）

    我重拾 “跟踪记录查览器” 的想法，是因为现在已经是 2021 年，而该领域依然杂乱而没有发展。在其他一些想法中，我坚信：（1）有一张地图对应着程序执行；（2）传统的制图学基本原理能应用在这些图的汇总、展示、导向上。  

    我希望大伙能发现此插件的炫酷点，Tenet 只是推进研究这些想法的一小步。  

    【用法】

    Tenet 可在 github 上搜索下载。允许条件是 IDA 7.5 和 Python 3.  

    正确安装后（译注：参考后述安装方法），在反汇编器中会有一个新的菜单项。此菜单项可以用来加载外部收集的执行跟踪记录倒 Tenet 中。（译注：这里应该注意，实际 Tenet 只是一个契合 IDA7.5 的 “跟踪记录” 的查览器，“跟踪记录”需要根据格式通过其他途径采集，且采集的 “跟踪记录” 的模块基址如果随机，则需要对 IDA 反编译器中目标模块的基址进行 rebase 重定位，参考后续描述）  

![](https://bbs.pediy.com/upload/attach/202104/760675_FUBJUK6CFM6WRMU.jpg)

Tenet 给 IDA 增加一个加载文件的子菜单来实现执行跟踪记录的加载

    由于是最初的分发开源版本，Tenet 只接受简单的可供人阅读的文本跟踪记录。更多有关该跟踪记录格式、局限和相关的追踪器的其他信息请参考 github 开源项目中 tenet/tracers/README.md 文件。（译注：跟踪记录格式是我们应该关注的，相关的追踪器也是我们应该关注的重点）

    【双向查览】  

    当使用 Tenet 时，该插件会给运行踪迹上色暗示执行方向：蓝色标记向前执行流，红色标记向后执行流，都是基于当前执行位置相对位置由深到浅渐变色表示。  

![](https://bbs.pediy.com/upload/attach/202104/HWAHXRRUKCEXZZD.jpg)

Tenet 在浏览跟踪记录时提供过去和当前的执行区域流  

    要通过执行时间向前或向后 “步进”，我们只需简单地把鼠标放在反编译器右侧的时间线上，然后上下滚动鼠标滑轮即可。  

    要 “步过” 函数调用，只需要在滚动鼠标滑轮的同时按下 “SHIFT” 键即可。（译注：应该注意到差别，如果不按 SHIFT，相当于单步跟进）  

    【跟踪记录时间线】  

    跟踪记录的时间线在反编译器的右边。该窗口用来对不同事件类型沿着时间线可视化，且用于上面所述的基本浏览功能。  

![](https://bbs.pediy.com/upload/attach/202104/760675_C73KTRU733YUNJU.jpg)

跟踪记录时间线是一个关键的功能部件，用于浏览 Tenet 跟踪记录和聚焦感兴趣的区域

    通过在时间线上 “单击按下鼠标并拖动划过”，可以对执行跟踪记录的指定区间进行放大。此操作可以重复任意次以达到期望的粒度。  

    【执行断点】  

        单击寄存器窗口的指令指针寄存器（EIP 或 RIP 等），会红色高亮显示，同时会在整个时间线上定位显示出该指令被执行的时间点。  

（译注：这点还是很炫酷，这能在整个时间线上找出所有执行过某个指令或指令流的所有时间节点，可通过鼠标放置在 EIP 上进行滚轮滑动切换到该指令不同的执行时间点）。

![](https://bbs.pediy.com/upload/attach/202104/760675_Y73QPTDGJKBXP9G.jpg)

在当前指令上放置断点并通过滚动鼠标滑轮在不同执行时间点上切换

    IDA 的 F2 热键也可以用来再任意指令处设置断点。（译注：目前未发现 Tenet 使用 F2 设置所谓断点的用处）  

    （译注：所谓的放置断点，一般在反汇编窗口定位到感兴趣的指令位置（可以通过 g 快捷键之间输入地址跳转），然后右键选择 Tenet 的 Go to... 到 Go to first exectuion 即可，然后再单击 EIP 和滚动鼠标滑轮进行执行时间点的切换，如下为译者配图）

![](https://bbs.pediy.com/upload/attach/202104/760675_MZ876N2GBRB89K8.jpg)

    【内存断点】

       通过再栈或内存视图中单击一个字节，将即刻在可视化的整个跟踪记录时间线上看到对该地址所有读写操作。（译注：GoodJob!）

![](https://bbs.pediy.com/upload/attach/202104/760675_6VCPNKPAESGD7NJ.jpg)

基于对选中的特定内存的访问，在全部执行状态中定位对应踪迹  

    内存断点可使用执行断点相同的技术来导览。单击某个字节，然后在鼠标悬停在选中字节上的同时滚动鼠标滑轮即可定位访问它的每一个踪迹。  

    在感兴趣的字节上右击将提供内存的读 / 写 / 访问等定位选项，当然，前提是该处字节存在你想要的指定导览动作。  

![](https://bbs.pediy.com/upload/attach/202104/760675_C4A5SM7PFQHAU4K.jpg)

Tenet 为针对内存访问提供许多快捷导览

    要导览（或定位）到内存视图中的某个地址，点击进内存视图然后通过 g 快捷功能输入地址或数据库符号来定位转到目标位置。  

    【区域断点】  

        一个十分试验性的功能是对一个内存区域设置访问断点。可通过高亮选定一块内，然后在右键菜单中找到相应访问动作来达到。  

![](https://bbs.pediy.com/upload/attach/202104/760675_CWBB4299D3HGAUD.jpg)

 Tenet 有允许在一个内存区域上设置访问断点，并在访问动作之间导览  

        与普通内存断点一样，在区域上悬停并滚动鼠标滑轮能用来在对选择的内存区域的不同访问操作之间切换。  

    【寄存器定位】  

    在逆向工程中，我们经常会遇到这样的情形，“哪个指令设置了这个寄存器的当前的值？”。  

    通过 Tenet，我们能通过单击一下就能回溯到该指令（设置寄存器值的指令）。（译注：Nice！通过单击寄存器值左侧箭头即可）

![](https://bbs.pediy.com/upload/attach/202104/760675_F3Y34DHDXYPNCTZ.jpg)

定位到对感兴趣的寄存器进行设置的时刻

    回溯定位（向后定位）在寄存器值变化方向之间要常用得多，但出于方便我们也能向前定位到下一处对寄存器赋值的地方，通过单击寄存器右侧的蓝色箭头就可以（译注：鼠标悬停在寄存器左 \ 右侧的箭头上才会显示红 \ 蓝色）。  

    【时刻终端】  

    提供了一个简单的 “终端”（shell）来在跟踪记录中导览到指定的时刻。在“终端” 中粘贴（或输入）一个时刻（带或不到叹号！）就足够。  

    （译注：就在反汇编窗口右侧时间线旁的寄存器窗口底部，一个类似命令行窗口，“时刻”或者可以称为 “位置”（position），该处会实时同步，所以可以记录特定时刻位置，后续可以之间精准输入定位的记录的时刻，叹号表示百分数的意思，如定位到末端时刻 100%(!100) 或中间时刻 50%(!50）)  

![](https://bbs.pediy.com/upload/attach/202104/760675_8MZ4SJDH3TZBMN8.jpg)

“时刻终端” 能用来将跟踪记录阅读器导览到指定时刻

    我们也能通过感叹号定位到跟踪记录的百分几位置。如输入 “!100"将定位到跟踪记录的最后一个指令，而"!50" 到位跟踪记录一半的位置。（译注：应当注意的，最后一个跟踪记录的指令并不一定在反汇编的模块当中，即 EIP 可能是当前反汇编模块之外的指令）  

    【主题皮肤】  

        Tenet 推出两种默认主题：高亮、暗黑。根据当前反编译器使用的颜色，Tenet 会尝试选择合适的主题。  

![](https://bbs.pediy.com/upload/attach/202104/760675_HWNK5M8KPUMQ9BZ.jpg)

Tenet 会基于用户反编译主题选择高亮和暗黑主题中的一个

    主题文件以 json 的方式简单存放在磁盘且是容易定制。若不喜欢默认的主题或亚瑟，可以创建自己的主题，简单将其托放进用户的主题目录（IDA 的 plugins\tenet\ui\resources\themes）即可。  

    Tenet 会在后续加载和使用中记住偏好的主题。  

    【开源】  

    Tenet 目前是一个没有资金支持的研究项目。可在 github 上基于 MIT 许可自由获取。在 README 中，可以获取更多其他信息，如，如何开始使用 Tenet，未来的一些想法，乃至于在此文中未涉及的一些常见问题。  

    没有资金支持，我能对该项目的贡献的时间是有限的。若你所在的组织对这里提出的想法感兴趣且能对 Tenet 提供资金以赞助专用的开发时间，请联系我们。

    【相关作品】  

        在这个领域并没有太多强悍的项目，对少数存在的提高认知很重要。若你发现此文有趣，请考虑进一步浏览和支持下面这些技术：

（1） QIRA：QEMU 交互式运行时分析器

（2）Microsoft TTD：针对 Windows 的时域追踪调试（Time Travel Debugging）

（3）RR//Pernosce：针对 Linux 的时间无关调试（Timeless Debugging）

（4）PANDA：基于 QEMU 的全系统追踪（Full Systen Tracing)

（5）REVEN：基于 PANDA 或 VirtualBox 的全系统追踪

（6）UndoDB：针对 Linux 和 Android 的可逆调试（Reversible Debugging）

（7）DejaVM     :^)

    【结语】  

    在此文中，我们展示了一款称为 Tenet 的 IDA Pro 插件。这是一个试验性的插件，设计来查览软件执行跟踪记录。它能用来当作逆向工程过程的一个辅助，也能用来当作在程序分析、虚拟化、下一代调试实践的有效性（等）的探索中使用的研究技术。  

    我们对这些技术的开发实践是首屈一指的。RET2 很乐意在这些领域提供咨询，提供插件开发服务，对现有工作追加定制功能，或通过相关的安全工具提供独一无二的合作。若你所在的组织对这些专用技能有所需，尽管联系我们。  

二、执行跟踪记录  的 采集

    （一）pin 的 pintool 插件

    Tenet 的 github 开源代码【tenet/tracers/pin/】目录提供了 pintool 源码。

    在 Window 下，源码的编译需要下载 Intel 的 Pin 工具和 cygwin（目的是使用 GNU 的 make），然后配置相应的缓解进行编译执行，如下图  

![](https://bbs.pediy.com/upload/attach/202104/760675_KMMR8NGUGU8EEZD.jpg)

即使按上面的配置好，也不一定会一帆风顺，比如 pin 的 include\msvc_compat.h 头文件找不到呀等等的问题。

这时候可以修改 Intel 的 Pin 安装路径下的【Pin_318\source\tools\Config\makefile.win.config】文件相关内容排除，注意是文件路径分隔符的问题。

将 "/" 分隔符全部改为 “\"分隔符，但对【/FI"include\msvc_compat.h" 】、【pin.h】等还是会有问题。附件提供了对 Intel 的 Pin 目录下的【source\tools\Config\makefile.win.config】的修改版，请再覆盖该文件时，先备份原文件

```
@rem 启动命令行cmd
@rem 进入【tenet/tracers/pin/】目录
pushd d:\src\tenet\tracers\pin
@rem 设置Intel的Pin安装路径
set PIN_ROOT=c:\PinDir
@rem 设置cygwin的make路径
set path=C:\cygwin\bin;%path%
@rem 启动Visual Studio 32
"C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Auxiliary\Build\vcvars32.bat"
@rem 编译32位pintool插件
make TARGET=ia32
 
 
 
@rem 遇到如下类似错误时
cl /EHs- /EHa- /wd4530 /DTARGET_WINDOWS /nologo /Gy /Oi- /GR- /GS- /D__PIN__=1 /DPIN_CRT=1 /D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um" /D__i386__ /Zc:threadSafeInit- /Zc:sizedDealloc- /wd5208 /DTARGET_IA32 /DHOST_IA32  /Ic:\PinDir\source\include\pin /Ic:\PinDir\source\include\pin\gen -Ic:\PinDir\extras\stlport\include -Ic:\PinDir\extras -Ic:\PinDir\extras\libstdc++\include -Ic:\PinDir\extras\crt\include -Ic:\PinDir\extras\crt -Ic:\PinDir\extras\crt\include\arch-x86 -Ic:\PinDir\extras\crt\include\kernel\uapi -Ic:\PinDir\extras\crt\include\kernel\uapi\asm-x86 /FI"c:\PinDir\extras\crt\include\msvc_compat.h" /Ic:\PinDir\extras\components\include /Ic:\PinDir\extras\xed-ia32\include\xed /Ic:\PinDir\source\tools\Utils /Ic:\PinDir\source\tools\InstLib /MD /O2  /c /Foobj-ia32/pintenet.obj pintenet.cpp
pintenet.cpp
pintenet.cpp(77): error C3861: “va_start”: 找不到标识符
pintenet.cpp(79): error C3861: “va_end”: 找不到标识符
pintenet.cpp(88): fatal error C1083: 无法打开包括文件: “pin.H”: No such file or directory
make[1]: *** [c:\PinDir\source\tools/Config/makefile.default.rules:205: obj-ia32/pintenet.obj] Error 2
make[1]: Leaving directory '/d/src/tenet/tracers/pin'
make: *** [c:\PinDir/source/tools/Config/makefile.config:339: all] Error 2
 
 
@rem 直接复制上述cl编译器的编译命令如下进行编译
cl /EHs- /EHa- /wd4530 /DTARGET_WINDOWS /nologo /Gy /Oi- /GR- /GS- /D__PIN__=1 /DPIN_CRT=1 /D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um" /D__i386__ /Zc:threadSafeInit- /Zc:sizedDealloc- /wd5208 /DTARGET_IA32 /DHOST_IA32  /Ic:\PinDir\source\include\pin /Ic:\PinDir\source\include\pin\gen -Ic:\PinDir\extras\stlport\include -Ic:\PinDir\extras -Ic:\PinDir\extras\libstdc++\include -Ic:\PinDir\extras\crt\include -Ic:\PinDir\extras\crt -Ic:\PinDir\extras\crt\include\arch-x86 -Ic:\PinDir\extras\crt\include\kernel\uapi -Ic:\PinDir\extras\crt\include\kernel\uapi\asm-x86 /FI"c:\PinDir\extras\crt\include\msvc_compat.h" /Ic:\PinDir\extras\components\include /Ic:\PinDir\extras\xed-ia32\include\xed /Ic:\PinDir\source\tools\Utils /Ic:\PinDir\source\tools\InstLib /MD /O2  /c /Foobj-ia32/pintenet.obj pintenet.cpp
 
@rem 然后重复执行make
make TARGET=ia32
 
@rem 同理，遇到错误时，直接复制编译命令在cmd中执行

```

如果【/D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um"】和【c:\PinDir】相同，也可以不用 cywin，而直接执行下述编译命令对 pintool 的【32 位】插件进行编译和链接

```
mkdir obj-ia32
 
cl /EHs- /EHa- /wd4530 /DTARGET_WINDOWS /nologo /Gy /Oi- /GR- /GS- /D__PIN__=1 /DPIN_CRT=1 /D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um" /D__i386__ /Zc:threadSafeInit- /Zc:sizedDealloc- /wd5208 /DTARGET_IA32 /DHOST_IA32  /Ic:\PinDir\source\include\pin /Ic:\PinDir\source\include\pin\gen -Ic:\PinDir\extras\stlport\include -Ic:\PinDir\extras -Ic:\PinDir\extras\libstdc++\include -Ic:\PinDir\extras\crt\include -Ic:\PinDir\extras\crt -Ic:\PinDir\extras\crt\include\arch-x86 -Ic:\PinDir\extras\crt\include\kernel\uapi -Ic:\PinDir\extras\crt\include\kernel\uapi\asm-x86 /FI"c:\PinDir\extras\crt\include\msvc_compat.h" /Ic:\PinDir\extras\components\include /Ic:\PinDir\extras\xed-ia32\include\xed /Ic:\PinDir\source\tools\Utils /Ic:\PinDir\source\tools\InstLib /MD /O2  /c /Foobj-ia32/pintenet.obj pintenet.cpp
 
cl /EHs- /EHa- /wd4530 /DTARGET_WINDOWS /nologo /Gy /Oi- /GR- /GS- /D__PIN__=1 /DPIN_CRT=1 /D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um" /D__i386__ /Zc:threadSafeInit- /Zc:sizedDealloc- /wd5208 /DTARGET_IA32 /DHOST_IA32  /Ic:\PinDir\source\include\pin /Ic:\PinDir\source\include\pin\gen -Ic:\PinDir\extras\stlport\include -Ic:\PinDir\extras -Ic:\PinDir\extras\libstdc++\include -Ic:\PinDir\extras\crt\include -Ic:\PinDir\extras\crt -Ic:\PinDir\extras\crt\include\arch-x86 -Ic:\PinDir\extras\crt\include\kernel\uapi -Ic:\PinDir\extras\crt\include\kernel\uapi\asm-x86 /FI"c:\PinDir\extras\crt\include\msvc_compat.h" /Ic:\PinDir\extras\components\include /Ic:\PinDir\extras\xed-ia32\include\xed /Ic:\PinDir\source\tools\Utils /Ic:\PinDir\source\tools\InstLib /MD /O2  /c /Foobj-ia32/ImageManager.obj ImageManager.cpp
 
link /DLL /EXPORT:main /NODEFAULTLIB /NOLOGO /INCREMENTAL:NO /IGNORE:4210 /IGNORE:4049 /FORCE:MULTIPLE  /DYNAMICBASE /NXCOMPAT c:\PinDir\ia32\runtime\pincrt\crtbeginS.obj /MACHINE:x86 /ENTRY:Ptrace_DllMainCRTStartup@12 /BASE:0x55000000 /OPT:REF  /out:obj-ia32/pintenet.dll obj-ia32/pintenet.obj obj-ia32/ImageManager.obj  /LIBPATH:c:\PinDir\ia32\lib /LIBPATH:c:\PinDir\ia32\lib-ext /LIBPATH:c:\PinDir\ia32\runtime\pincrt /LIBPATH:c:\PinDir\extras\xed-ia32\lib pin.lib xed.lib pinvm.lib pincrt.lib ntdll-32.lib kernel32.lib

```

编译 pintool 的【64 位】插件同理，只是修改为【vcvars64.bat】和【make TARGET=intel64】，若错误，则直接复制编译命令在 cmd 中编译  

```
pushd d:\src\tenet\tracers\pin
@rem 设置Intel的Pin安装路径
set PIN_ROOT=c:\PinDir
@rem 设置cygwin的make路径
set path=C:\cygwin\bin;%path%
@rem 启动Visual Studio 64
"C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Auxiliary\Build\vcvars64.bat"
@rem 编译64位pintool插件
make TARGET=intel64

```

同理，如果【/D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um"】和【c:\PinDir】相同，也可以不用 cywin，而直接执行下述编译命令对 pintool 的【64 位】插件进行编译和链接

```
mkdir obj-intel64
 
cl /EHs- /EHa- /wd4530 /DTARGET_WINDOWS /nologo /Gy /Oi- /GR- /GS- /D__PIN__=1 /DPIN_CRT=1 /D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um" /D__LP64__ /Zc:threadSafeInit- /Zc:sizedDealloc- /wd5208 /DTARGET_IA32E /DHOST_IA32E  /Ic:\PinDir\source\include\pin /Ic:\PinDir\source\include\pin\gen -Ic:\PinDir\extras\stlport\include -Ic:\PinDir\extras -Ic:\PinDir\extras\libstdc++\include -Ic:\PinDir\extras\crt\include -Ic:\PinDir\extras\crt -Ic:\PinDir\extras\crt\include\arch-x86_64 -Ic:\PinDir\extras\crt\include\kernel\uapi -Ic:\PinDir\extras\crt\include\kernel\uapi\asm-x86 /FI"c:\PinDir\extras\crt\include\msvc_compat.h" /Ic:\PinDir\extras\components\include /Ic:\PinDir\extras\xed-intel64\include\xed /Ic:\PinDir\source\tools\Utils /Ic:\PinDir\source\tools\InstLib /MD /O2  /c /Foobj-intel64/pintenet.obj pintenet.cpp
 
cl /EHs- /EHa- /wd4530 /DTARGET_WINDOWS /nologo /Gy /Oi- /GR- /GS- /D__PIN__=1 /DPIN_CRT=1 /D_WINDOWS_H_PATH_="C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um" /D__LP64__ /Zc:threadSafeInit- /Zc:sizedDealloc- /wd5208 /DTARGET_IA32E /DHOST_IA32E  /Ic:\PinDir\source\include\pin /Ic:\PinDir\source\include\pin\gen -Ic:\PinDir\extras\stlport\include -Ic:\PinDir\extras -Ic:\PinDir\extras\libstdc++\include -Ic:\PinDir\extras\crt\include -Ic:\PinDir\extras\crt -Ic:\PinDir\extras\crt\include\arch-x86_64 -Ic:\PinDir\extras\crt\include\kernel\uapi -Ic:\PinDir\extras\crt\include\kernel\uapi\asm-x86 /FI"c:\PinDir\extras\crt\include\msvc_compat.h" /Ic:\PinDir\extras\components\include /Ic:\PinDir\extras\xed-intel64\include\xed /Ic:\PinDir\source\tools\Utils /Ic:\PinDir\source\tools\InstLib /MD /O2  /c /Foobj-intel64/ImageManager.obj ImageManager.cpp
 
link /DLL /EXPORT:main /NODEFAULTLIB /NOLOGO /INCREMENTAL:NO /IGNORE:4210 /IGNORE:4049 /FORCE:MULTIPLE  /DYNAMICBASE /NXCOMPAT c:\PinDir\intel64\runtime\pincrt\crtbeginS.obj /MACHINE:x64 /ENTRY:Ptrace_DllMainCRTStartup /BASE:0xC5000000 /OPT:REF  /out:obj-intel64/pintenet.dll obj-intel64/pintenet.obj obj-intel64/ImageManager.obj  /LIBPATH:c:\PinDir\intel64\lib /LIBPATH:c:\PinDir\intel64\lib-ext /LIBPATH:c:\PinDir\intel64\runtime\pincrt /LIBPATH:c:\PinDir\extras\xed-intel64\lib pin.lib xed.lib pinvm.lib pincrt.lib ntdll-64.lib kernel32.lib

```

（二）跟踪记录采集

    有了 Intel 提供的 Pin，和上述编译的 pintool 插件（用于捕获收集适用于 Tenet 的跟踪记录），就可以对目标进行跟踪记录的采集。  

这里我们以下述简单样例【consoleTest.cpp】为目标，命令行启动相应的编译缓解【vcvars32.bat】或【vcvars64.bat】然后，直接一个命令【cl consoleTest.cpp】编译链接即可。

```
//File consoleTest.cpp
 
// to compile
// 32-bit: use "C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Auxiliary\Build\vcvars32.bat"
// 64-bit: use "C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Auxiliary\Build\vcvars64.bat"
//cl consoleTest.cpp
 
 
#include  #define XBUF_LEN 0x100
int main(int argc, char* argv[]) {
    static char gbuf[XBUF_LEN] = { 0 };
    char lbuf[XBUF_LEN] = { 0 };
    int i = 0;
    char msg[3][XBUF_LEN] = { "abc message","c++ pro","ida frida" };
    for (i = 0; i < 3; i++) {
        memcpy(lbuf, msg[i], strlen(msg[i]));
        memcpy(gbuf, lbuf, strlen(lbuf));
        std::cout << gbuf << std::endl;
    }
    return 0;
} 
```

通过 pin 对目标执行 pintool 进行采集，这里适用 32 位的 pin 加载 32 位的 pintool 插件对 32 位对象进行跟踪采集，由于 consoleTest.exe 目标体量比较小，这里我们没有适用 pintool 插件的 - w 白名单参数进行过来跟踪，而是对所有模块执行跟踪记录。（应该注意的是，对于稍微复杂些的目标，如简单的多线程 MFC 目标，如果不适用白名单参数进行过滤，几乎不能正常执行；另外作者发布的 pintool 插件的白名单 - w 参数并不像 Tenet 说明上说的，多个模块用逗号分隔即可；因为种原因，我对作者的 pintool 工具进行的部分改动，改动后的源码和编译好的 pintool 参考附件）

```
c:\PinDir\ia32\bin\pin.exe -t d:\src\tenet\tracers\pin\obj-ia32\pintenet.dll  -- d:\xtest\consoleTest.exe

```

会在命令行中得到如下输出

```
[tritium] main :PIN_InitSymbols()
OK!
[tritium] main :PIN_Init(argc, argv)
        argv[0] : c:\PinDir\ia32\bin\pin.exe
        argv[1] : -t
        argv[2] : D:\src\tenet\tracers\pin\obj-ia32\pintenet.dll
        argv[3] : --
        argv[4] : D:\xtest\consoleTest.exe
[tritium] main : white-listing-count is 0
[tritium] main: PIN_StartProgram()
Loaded image: 0xed0000:0xefdfff -> consoleTest.exe
Loaded image: 0x75440000:0x7552ffff -> kernel32.dll
Loaded image: 0x75530000:0x75742fff -> KernelBase.dll
Loaded image: 0x774f0000:0x77691fff -> ntdll.dll
abc message
c++ prosage
ida fridage
Loaded image: 0x74630000:0x7463efff -> kernel.appcore.dll
Loaded image: 0x76360000:0x7641efff -> msvcrt.dll
Loaded image: 0x76420000:0x764d9fff -> rpcrt4.dll

```

这个是针对 console 样例目标才有的输出，因为原 pintool 对桌面型如 MFC 目标不会有上述信息输出，所以我做了些许改动，无论对 console 还是桌面样例，上述信息都会追加到 cmd 当前目录的 lg.txt 文件中，通过 lg.txt 文件可以查阅上述信息。如下

一般线程 0 是我们在 IDA 的 Tenet 中要适用的跟踪记录数据。

```
2021/04/23  19:06               747 lg.txt
2021/04/23  19:06        84,210,271 trace.TID0.log
2021/04/23  19:06         9,046,216 trace.TID1.log

```

如果要适用 pintool 的白名单参数，可以通过 - w 多次指定多个模块（而不是 Tenet 说明的用逗号分隔）  

```
c:\PinDir\ia32\bin\pin.exe -t d:\src\tenet\tracers\pin\obj-ia32\pintenet.dll  -w consoleTest.exe -w kernel32.dll -- d:\xtest\consoleTest.exe

```

有了采集的跟踪记录，即可在 IDA 对【consoleTest.exe】的反编译中通过 Tenet 加载跟踪记录。

应该注意到，上述 consoleTest.exe 的加载基址是随机的 0xed0000，采集的跟踪记录时基于这些基址的，

所以在需要多相应的分析模块的基址重定位位采集数据的基址，如 IDA Pro 中 consoleTest.exe 的基址是 0x400000,

如图，这里【Edit》Segments》Rebase progam】进行重定位

![](https://bbs.pediy.com/upload/attach/202104/760675_UXH7ZZXUUX9FC68.jpg)

修改位采集记录的模块实际基址

![](https://bbs.pediy.com/upload/attach/202104/760675_4EZS6F24KAA6FEM.jpg) 

然后通过【File》Load File 》Tenet trace file】加载【trace.TID0.log】即可，一般加载主线程执行跟踪记录即可。这时刻就可以愉快的对 Tenet 插件进行上述第一部分的提及的适用方式进行测试了。

三、后话

   （1）Tenet 的技术路线的确炫酷，但在执行记录的采集上可能比较痛苦，虽然严格上已经不算是 Tenet 的范畴。  

   （2）在测试时，一个简单的多线程 MFC 如果没有过滤参数的参与，基本无法让目标正常运行，如下述对【确定】按钮响应函数只添加三行代码，几乎跑不起来。  

![](https://bbs.pediy.com/upload/attach/202104/760675_5C4K8NGSCMHX25A.jpg)

还没完全跑起来，数据已达到惊人的 30G，从结束进程的时间上看，这个可能也是错误测试导致的结果，比较已经执行了四个小时，的确有些痛苦，跟踪记录数据的采集可能让 Tenet 的应用有些骨感，还需要更好的执行根据记录采集方案，如果道友有所发现，不吝分享一二。

![](https://bbs.pediy.com/upload/attach/202104/760675_F8Z9425HGJHG632.jpg)

    （3）关于 Intel 的 Pin  

    上次见到它不晓得是几年前了，如下图所示介绍。Pin 是一种对程序进行监测的工具。主要手段是在执行目标任意位置插入任意监测代码。主要的监测代码是通过为 Pin 提供插件的形式提供的，如以 pin 的 - t 参数提供 pintool（监测代码）。而维护程序或指令执行状态不变的过程对开发 pintool 插件的开发人员是透明的。  

instrumentation 这个词

![](https://bbs.pediy.com/upload/attach/202104/760675_82JY5GD34FJNNQH.jpg)

无独有偶，我们可以看看 frida 的跟踪命令，【Instrumenting...】。

监测思路上相似，Pin 最基础的可以基于指令粒度进行监测，当然也可以对分块的指令流，某些函数、线程或进程状态进行监测。

frida 的监测代码是通过 js 等方式添加的，而 Pin 的监测代码（称为 pintool）则通过 C\C++ 代码的以插件的方式添加（通过为 Pin 指定 - t 参数）

如 MessageBoxExA.js 在 frida 中实现对 MessageBoxExA 函数的监测；而 pintenet.dll（提供对执行指令的跟踪记录）在 pin 中实现对目标程序执行指令的监测（pin 说明文档虽然不建议在指令的粒度上进行监测）。

```
python frida_tools\tracer.py -i "MessageBox*" -p 33068
Instrumenting...
MessageBoxExA: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxExA.js"
MessageBoxExW: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxExW.js"
MessageBoxTimeoutA: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxTimeoutA.js"
MessageBoxIndirectA: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxIndirectA.js"
MessageBoxTimeoutW: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxTimeoutW.js"
MessageBoxIndirectW: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxIndirectW.js"
MessageBoxA: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxA.js"
MessageBoxW: Loaded handler at "D:\\\\src\\frida\\frida-tools\\build\\lib\\__handlers__\\USER32.dll\\MessageBoxW.js"
Started tracing 8 functions. Press Ctrl+C to stop.

```

附件清单，如下，使用 32 或 64 位的 pin.exe 和 pintenet.dll 即可对目标如 consoleTest.exe 进行跟踪记录采集。如需重新编译 pintenet.dll，需要到 Intel 的 Pin 官方下载 Pin 的完整开发包；makefile.win.config 替换 Pin 开发包对于的文件或直接参考上述编译命令执行。

Tenet.zip 只官方的 IDA 插件部分，直接解压释放到 IDA 的 plugins 目录即可

```
SRCBIN
│  makefile.win.config
│
├─pin                         
│  ├─ia32
│  │  └─bin
│  │          dbghelp.dll
│  │          dia_client.dll
│  │          msdia140.dll
│  │          pin.exe
│  │          pincrt.dll
│  │          pindb.exe
│  │          pinjitprofiling.dll
│  │          pinvm.dll
│  │
│  └─intel64
│      └─bin
│              dbghelp.dll
│              dia_client.dll
│              msdia140.dll
│              pin.exe
│              pincrt.dll
│              pindb.exe
│              pinjitprofiling.dll
│              pinvm.dll
│
├─tenet
│  └─traces
│      └─pin
│          │  ImageManager.cpp
│          │  ImageManager.h
│          │  Makefile
│          │  pintenet.cpp
│          │  README.md
│          │
│          ├─obj-ia32
│          │      pintenet.dll
│          │
│          └─obj-intel64
│                  pintenet.dll
│
└─xtest
        consoleTest.cpp
        consoleTest.exe
        lg.txt
        trace.TID0.log
        trace.TID1.log

```

（4）Tenet 插件快速测试  

    A、附件 Tenet.zip 中 plugins 里的全部文件和文件夹解压释放到 IDA 7.5 的 plugins 目录。  

    B、IDA 加载 srcbin 中的 xtest\consoleTest.exe 反编译，并通过【Edit》Segments》Rebase progam】进行重定位为 0xed0000。

    C、通过【File》Load File 》Tenet trace file】加载【trace.TID0.log】

    愉快地和 Tenet 玩耍。  

【参考文献】

    1、参见开头 py 代码解密出来的 [https://blog.ret2.io/2021/04/20/tenet-trace-explorer/](https://blog.ret2.io/2021/04/20/tenet-trace-explorer/) 

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 2021-5-6 09:09 被 tritium 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#系统底层](forum-4-1-2.htm)

上传的附件：

*   [srcbin.zip](javascript:void(0)) （5.19MB，17 次下载）
*   [srcbin.z01](javascript:void(0)) （7.00MB，13 次下载）
*   [srcbin.z02](javascript:void(0)) （7.00MB，9 次下载）
*   [Tenet.zip](javascript:void(0)) （91.32kb，10 次下载）