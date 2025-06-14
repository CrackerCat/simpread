> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287241.htm)

> [原创]Windbg TTD 还原 .NET JIT 保护壳探索

之前分析一款 .net 软件，使用 dnspy 打开一些可执行文件后，发现方法都无法被正常反编译，经对壳代码一些字样进行分析和搜索后，判断是加了德国一款保护软件的 .NET JIT 壳，把函数的方法体指令即 ILCode 都给抽取走了，会在运行时进行解密提供使用。

接着我开始琢磨怎么还原，搜索到 wwh1004 前辈的分析文章《[.NET JIT 脱壳指南与工具源码](elink@d1dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6$3R3I4x3o6l9@1i4K6u0W2j5$3!0E0i4K6u0r3L8X3g2@1i4K6u0V1K9X3W2@1i4K6u0V1N6h3&6H3j5h3y4C8i4K6u0V1k6%4g2A6k6r3g2Q4x3X3c8S2L8X3c8Q4x3X3c8K6L8%4g2J5j5$3g2Q4x3V1j5`.)》 和相应的开源项目 [JitUnpacker-Framework](elink@001K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6%4N6$3R3I4x3o6l9@1i4K6u0r3d9X3W2@1g2h3&6H3j5h3y4C8k6i4u0Q4x3X3c8r3M7X3q4E0k6i4N6G2M7X3D9`.)，但经过简单尝试后发现并不能解决我碰到的保护壳类型。

主要因为该保护程序有着较为复杂的保护模块和运行逻辑，无法简单地通过项目工具对单个 .net 可执行文件进行加载解析所有方法，再通过 `MethodDesc::DoPrestub` 主动调用获取还原需要的信息，因为保护程序模块还没启动起来，其次哪怕能模拟启动起来了，还有一堆检查反调试在等着呢。比如经测试，如果直接调试或者 frida 注入会被检测到，保护程序的授权会立马被注销掉，软件也无法正常运行启动了。

其次的话 JitUnpacker-Framework 项目工具是六年前发布的，支持范围是 .NET 2.0~4.72，.NET 4.8+ 的版本是没有进行适配的。而现在新版 Windows 系统几乎默认安装高版本 4.8+ 的 Framework，并跟随 Windows 系统进行更新升级，可以兼容执行 .NET 4.0+ 的程序，无疑给工具带来不少局限性。

随后我在 wwh1004 前辈对 .net 的 JIT 编译流程详细分析的基础，在解决工具方案上另辟蹊径，选择使用 Windbg TTD 对该软件执行过程进行录制，对录制文件进行调试分析，结合 Windbg 对 .net 的 SOS 调试器拓展，完成了在 .NET 4.8+ 平台上对被保护程序关键函数方法方法体指令的恢复提取，最后再使用 Javascript 脚本实现自动化。

事后整理时我又仔细阅读了 wwh1004 前辈的文章和项目代码，发现分析的德国保护软件所加的 .NET JIT 保护，只是简单的抽取了方法体指令，并没有动方法的局部变量签名和异常子句信息，更不提进阶保护对 token 进行加密了。于是我又下载了 .NET Reactor 6.9.0，开启拥有对方法体指令、局部变量签名和异常子句信息三者进行保护的 NecroBit 选项，使用 Windbg TTD 又进行了一番探索尝试，成功也完成了对后两者的还原恢复。

虽然整个过程不算难，但目前互联网似乎并没有多少对 .NET Framework 4.8+ 平台上 .NET JIT 壳分析与恢复的资料和工具，更是搜索不到使用 Windbg TTD 来进行恢复还原的内容，此文便是使用 Windbg TTD 还原一般性的 .NET JIT 保护壳的说明与记录，提供一种更为简单灵活的新方案。

.NET JIT
========

文档资源
----

wwh1004 前辈文章中提到的 CLR 与 JIT 学习文档 [The Book of the Runtime](elink@832K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6U0L8%4u0W2j5$3I4J5i4K6u0r3j5X3I4G2j5W2)9J5c8X3#2S2M7%4c8W2M7W2)9J5c8V1c8G2j5%4g2E0k6h3&6@1j5i4c8A6L8$3&6Q4x3V1k6T1L8%4c8J5i4K6u0r3f1V1g2m8c8p5#2q4i4K6u0W2L8h3b7`.) 所属的项目 [dotnet/coreclr](elink@cceK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6U0L8%4u0W2j5$3I4J5) 已经被归档为只读状态，项目代码现在转移到了 [donet/runtime](elink@df2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6J5N6h3&6@1K9h3#2W2) 中，文档也有一个单独的 Github Pages 构建仓库，可以在相应的[构建网站](elink@f30K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6B7N6i4u0S2K9$3!0$3K9h3y4Q4x3X3g2Y4K9i4c8Z5N6h3u0Q4x3X3g2A6L8#2)9J5c8Y4u0#2L8Y4c8A6L8h3g2Q4x3V1j5`.)上直接阅读 [RyuJIT](elink@d20K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6B7N6i4u0S2K9$3!0$3K9h3y4Q4x3X3g2Y4K9i4c8Z5N6h3u0Q4x3X3g2A6L8#2)9J5c8Y4u0#2L8Y4c8A6L8h3g2Q4x3V1k6J5P5i4g2B7K9i4c8Q4x3X3c8G2N6X3g2J5N6X3W2W2N6#2)9J5c8R3`.`.) 部分。

![](https://bbs.kanxue.com/upload/attach/202506/802108_4CHQUUJYXJF4VVK.png)

如果翻一翻 [donet/runtime](elink@697K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6J5N6h3&6@1K9h3#2W2) 项目仓库的话，会发现是从 .NET 5.0 开始进行的持续开发，到现在刚不久已经[发布](elink@558K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6D9k6h3q4J5L8W2)9J5k6h3#2A6j5%4u0G2M7$3!0X3N6q4)9J5k6h3y4G2L8g2)9J5c8X3g2F1i4K6u0V1N6i4y4Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6U0L8%4u0W2i4K6u0r3N6$3S2S2N6s2y4Q4x3X3c8F1k6i4N6Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3X3b7I4x3q4)9J5c8X3!0$3k6i4u0$3K9h3g2%4)到 [.NET 10.0](elink@6b6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6J5N6h3&6@1K9h3#2W2i4K6u0r3M7X3g2D9k6h3q4K6k6i4y4Q4x3V1k6@1j5h3N6Q4x3V1k6$3x3e0m8Q4x3X3f1H3i4K6u0W2x3q4)9J5k6s2m8J5k6i4k6A6k6i4N6Q4x3X3f1#2i4K6u0W2x3U0f1J5y4K6N6Q4x3X3f1I4x3e0b7`.) 了，并没有我们在 Windows 上运行的 .NET Framework 4.8+ 版本。经搜索了解到 .NET Framework 未开源是一个历史遗留问题，其在代码设计上与 Windows 系统深度耦合，还有一些闭源组件依赖和许可限制问题，导致剥离开源成本高。所以即使微软曾发布过各个版本 .NET Framework 的 [Reference Source](elink@bcdK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6J5k6h3k6W2M7X3g2F1j5$3g2K6L8%4g2J5j5$3g2Q4x3X3g2E0K9h3y4J5L8%4y4G2k6Y4c8Q4x3X3g2U0L8$3#2Q4x3V1j5`.) 源码，但也并非真正开源，是没有核心组件如 CLR、JIT 代码的。微软对 .NET 的开发重心放在跨平台的 [donet/runtime](elink@e27K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6J5N6h3&6@1K9h3#2W2) 后，目前 4.8 就作为 .NET Framework 的最后一个主要版本，只持续进行安全和稳定性修复。

我们分析的 JIT 编译流程在 .NET Framework 和 .NET Runtime 中基本是保持一致变动不大，所以我们分析自己本机 .NET Framework 4.8 的 clr.dll 和 clrjit.dll 执行的 JIT 编译时候，可以下载 [donet/runtime](elink@da7K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6J5N6h3&6@1K9h3#2W2) 的源码，以及阅读 [RyuJIT 模块文档](elink@96dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6B7N6i4u0S2K9$3!0$3K9h3y4Q4x3X3g2Y4K9i4c8Z5N6h3u0Q4x3X3g2A6L8#2)9J5c8Y4u0#2L8Y4c8A6L8h3g2Q4x3V1k6J5P5i4g2B7K9i4c8Q4x3X3c8G2N6X3g2J5N6X3W2W2N6#2)9J5c8R3`.`.)，来进行参考和辅助分析。

![](https://bbs.kanxue.com/upload/attach/202506/802108_E7ZYXNWZS7D2C3Q.png)

此外在使用 Winbdg 调试的过程中，会使用到 .NET 的 SOS 调试拓展来解析一些 C# 代码结构体的不透明句柄，相关资料可以阅读官方的[文档](elink@96cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6D9k6h3q4J5L8W2)9J5k6h3#2A6j5%4u0G2M7$3!0X3N6q4)9J5k6h3y4G2L8g2)9J5c8X3g2F1i4K6u0V1N6i4y4Q4x3V1k6V1L8%4c8F1k6i4c8Q4x3V1k6U0L8%4u0W2i4K6u0r3k6r3W2S2k6$3&6G2M7%4c8A6j5%4y4Q4x3V1k6K6L8%4y4Q4x3X3c8V1k6h3u0#2k6$3N6A6L8X3N6Q4x3X3c8W2P5s2c8W2L8Y4y4A6L8$3^5`.)。

保护分析与还原
-------

### 模块方法信息与 ILCode

阅读文章资料，我们知道 .NET JIT 不同版本都会变动，加壳出于实现的稳定和兼容性，都会通过 Hook ICorJitCompiler 虚表来实现，如果选择 Hook 更其他的函数，可能会面临麻烦的棘手版本偏移变化问题，而 ICorJitCompiler 可以直接通过 getJit 函数获取。

```
// runtime/src/coreclr/jit/ee_il_dll.cpp
static CILJit g_CILJit;
DLLEXPORT ICorJitCompiler* getJit()
{
    if (!g_jitInitialized)
    {
        return nullptr;
    }
 
    return &g_CILJit;
}
```

我们可以在 Runtime 文档的 [RyuJIT Overview](elink@239K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6B7N6i4u0S2K9$3!0$3K9h3y4Q4x3X3g2Y4K9i4c8Z5N6h3u0Q4x3X3g2A6L8#2)9J5c8Y4u0#2L8Y4c8A6L8h3g2Q4x3V1k6J5P5i4g2B7K9i4c8Q4x3X3c8G2N6X3g2J5N6X3W2W2N6#2)9J5c8R3`.`.) 的 Execution Environment and External Interface 部分看到对 ICorJitCompiler 接口和其方法的介绍，里面写到 compileMethod 是 JIT 的主要入口点，EE（执行引擎）会向其传递一个 ICorJitInfo 对象，以及包含 IL 代码、方法头和各种其他有用信息的参数。

![](https://bbs.kanxue.com/upload/attach/202506/802108_Z5JY9A6R4VD2B2X.png)

先来看下 .NET Rntime 项目中 ICorJitCompiler 的定义代码及注释，可以看到很详细，32 位和 64 位系统有着不同的 compileMethod 函数实现。

![](https://bbs.kanxue.com/upload/attach/202506/802108_JRYR37D6MBZFS3X.png)

此文以 32 位的 .NET Reactor 保护程序举例，所以就找到 CILInterp::compileMethod 方法实现。

![](https://bbs.kanxue.com/upload/attach/202506/802108_HSNS6DK575NYXJB.png)

注意到其中的 CORINFO_METHOD_INFO 结构体，找到其实现可以看到有 ILCode 信息、maxStack 和异常子句信息数量 EHcount，我们使用 Windbg 可以断在此函数获取这些信息。

```
// runtime/src/coreclr/inc/corinfo.h
 
struct CORINFO_METHOD_INFO
{
    CORINFO_METHOD_HANDLE       ftn;
    CORINFO_MODULE_HANDLE       scope;
    uint8_t *                   ILCode;
    unsigned                    ILCodeSize;
    unsigned                    maxStack;
    unsigned                    EHcount;
    CorInfoOptions              options;
    CorInfoRegionKind           regionKind;
    CORINFO_SIG_INFO            args;
    CORINFO_SIG_INFO            locals;
};
```

额外的还有 CORINFO_METHOD_HANDLE 和 CORINFO_MODULE_HANDLE，看起来是和方法与模块信息有关，继续查看定义。

```
// runtime/src/coreclr/inc/corinfo.h
 
// Cookie types consumed by the code generator (these are opaque values
// not inspected by the code generator):
 
typedef struct CORINFO_MODULE_STRUCT_*      CORINFO_MODULE_HANDLE;
typedef struct CORINFO_METHOD_STRUCT_*      CORINFO_METHOD_HANDLE;
```

经过注释和搜索分析知道，这两个值在 JIT 和 CLR/CoreCLR 运行时中是作为不透明句柄，或者说是标识符，标识着 CLR/CoreCLR 运行时中对应的方法元数据和内部表示。因为其结构可能非常复杂，还可能随着 CLR 版本的不同而变化，以及 JIT 并不需要知道这个结构体内部的具体字段和布局，只需要一个标识就行，所以就是不透明句柄的形式了。反过来，如果 JIT 直接依赖 CLR/CoreCLR 运行时内部数据结构的具体布局，那么 CLR/CoreCLR 运行时的任何内部重构都可能破坏 JIT，这会使得维护和发展变得非常困难。

简而言之，这导致在使用 Windbg 调试分析 compileMethod 函数时候，看起来无法通过简单的解析内存结构体值来知道当前传入的是哪个模块的方法。而 JitUnpacker-Framework 项目是通过解析 .net 可执行文件的所有方法逐一主动调用，获取恢复方法信息时候自然知道当前主动调用的是哪个。

后来我经过一番摸索实践，在翻阅 Windbg .net 的 SOS 拓展调试文档时候，注意到 DumpMD 和 DumpModule 两个命令，经尝试结合 CORINFO_METHOD_INFO 结构体 中的 CORINFO_METHOD_HANDLE 和 CORINFO_MODULE_HANDLE 值是可以获取到模块方法信息的。

![](https://bbs.kanxue.com/upload/attach/202506/802108_QV4AJ7BGMXRE68P.png)

![](https://bbs.kanxue.com/upload/attach/202506/802108_39JVCQY9FHW2UVB.png)

### 局部变量签名

现在通过 Windbg 断点在 compileMethod 方法，可以获取到 ILCode 信息，借助 SOS 拓展调试命令也能知道当前 Jit 要编译的是哪个模块的方法，下面就是继续分析局部变量签名。

使用 dnspy 加载分析程序，选中一个方法右击选择 `编辑方法体/IL`，在 `指令栏` 看到有一个项 LocalVarSigTok，通过该值可以在 dnspy 的 `程序->PE->存储流 #0: #~->表流->StandAloneSig` 找到位于 `#Blob` 堆中的文件偏移位置，进行解析之后就是 `局部变量` 栏中展示的信息了。

![](https://bbs.kanxue.com/upload/attach/202506/802108_JFDRTS7G8QG5JW3.png)

调试分析的话定位到 `Compiler::lvaInitTypeRef()` 函数，里面会有一个循环来处理方法局部变量。

![](https://bbs.kanxue.com/upload/attach/202506/802108_JEXC9UXMFQR33TC.png)

其中 `info.compMethodInfo->locals` 便是前面 `CORINFO_METHOD_INFO` 结构体中的 `CORINFO_SIG_INFO locals`，我们继续看下 `CORINFO_SIG_INFO` 的定义。其中 `PCCOR_SIGNATUREp Sig` 项是指针，指向方法在 `#Blob` 堆中的局部变量签名数据，经分析这些数据是 C# 结构体变量序列化而来的，包含变量个数和不同变量的类型信息。也许就是说解析方法的局部变量签名数据流，同样是通过 C# 进行反序列化的，内部类型结构大小是一致的。

![](https://bbs.kanxue.com/upload/attach/202506/802108_9R6WNAHZXHVWGUR.png)

那么问题就来了，如何确定一个方法体的局部变量签名数据流结束位置呢，我翻了下 JitUnpacker-Framework 的项目源码，发现是模拟进行了反序列化的操作来通过 `Sig` 指针确定的。  
![](https://bbs.kanxue.com/upload/attach/202506/802108_DAG36E9NSDXJPE8.png)

可以看到 `MethodDumperBase.cs` 中的 `WalkType` 方法对十几种 .NET 类型进行了解析操作。

![](https://bbs.kanxue.com/upload/attach/202506/802108_3AP8G9HCVTCTJ3C.png)

所以现在又来了一个新问题，使用 Windbg 进行调试分析是不能进行如此复杂的反序列化解析的，那该怎么确定方法局部变量签名数据流的结束位置呢？

经过一番调试分析发现，我找到了一个巧妙的办法。注意到 for 循环一次执行结束后，进行 `i++` 的同时还会进行 `localsSig = info.compCompHnd->getArgNext(localsSig)` 操作，`localsSig` 值是具体的变量签名指针。当最后一次 `i` 增加到 `locals.numArgs` 值时不再小于将退出循环时候，还执行了一次 `getArgNext` 操作，执行完后 `localsSig` 将指向整个方法局部变量签名数据流的末尾。

这样，使用 Windbg 获取方法局部变量签名信息数据流的问题也解决了。

### 异常子句信息

最后方法的异常子句信息了，打开 dnspy 的 `编辑方法体/IL` 窗口的 `异常处理程序`，可以看到有着异常类型的信息，其中的捕获开始结束和处理开始结束，是指在方法指令中的偏移位置。

![](https://bbs.kanxue.com/upload/attach/202506/802108_3B84PQUEWUTFXQD.png)

定位到 `Compiler::fgFindBasicBlocks` 方法内的循环，

![](https://bbs.kanxue.com/upload/attach/202506/802108_G3UD8CRUT6NPRVJ.png)

可以看到循环内执行了 `info.compCompHnd->getEHinfo(info.compMethodHnd, XTnum, &clause)`，执行完后 clause 便获取到了值，这里就是使用 Windbg 提取异常子句信息的地方，可以进一步观察其类型 `CORINFO_EH_CLAUSE` 定义。异常子句不止一个，数量的话之前我们已经在 `CORINFO_METHOD_INFO` 结构体中获取到了。

```
// runtime/src/coreclr/inc/corinfo.h
 
//----------------------------------------------------------------------------
// Exception handling
 
struct CORINFO_EH_CLAUSE
{
    CORINFO_EH_CLAUSE_FLAGS     Flags;
    uint32_t                    TryOffset;
    uint32_t                    TryLength;
    uint32_t                    HandlerOffset;
    uint32_t                    HandlerLength;
    union
    {
        uint32_t                ClassToken;       // use for type-based exception handlers
        uint32_t                FilterOffset;     // use for filter-based exception handlers (COR_ILEXCEPTION_FILTER is set)
    };
};
```

Windbg TTD 还原
=============

Windbg TTD 的使用可以看我之前的文章[《TTD 调试与 ttd-bindings 逆向工程实践》](elink@f1bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6T1K9h3^5@1M7X3g2Q4x3X3g2Y4K9i4c8Z5N6h3u0Q4x3X3g2A6L8#2)9J5c8X3u0D9L8$3N6Q4x3V1j5J5x3o6t1K6i4K6u0r3x3o6N6Q4x3V1j5I4z5q4)9J5c8Y4c8@1k6q4)9J5k6r3c8W2j5Y4g2Y4k6$3W2F1k6#2)9J5k6s2c8@1k6q4)9J5k6r3u0A6L8X3c8A6L8X3N6K6i4K6u0V1M7X3g2$3k6i4u0K6k6g2)9J5k6r3g2F1k6$3W2F1k6h3g2J5K9h3&6Y4i4K6u0V1M7s2u0S2j5%4c8A6j5$3g2Q4x3V1j5`.)，使用 Windbg TTD 还原方案的流程是，先用 TTD 执行对 .NET JIT 加壳保护的可执行程序的录制，在录制文件基础上进行分析，通过 [TTD Calls Objects](elink@1a6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6D9k6h3q4J5L8W2)9J5k6h3#2A6j5%4u0G2M7$3!0X3N6q4)9J5k6h3y4G2L8g2)9J5c8X3g2F1i4K6u0V1N6i4y4Q4x3V1k6%4K9h3&6V1L8%4N6K6i4K6u0V1K9r3q4J5k6s2N6S2M7X3g2Q4x3V1k6V1M7X3W2$3k6i4u0K6i4K6u0r3k6r3g2T1N6h3N6Y4k6i4u0U0L8h3c8K6i4K6u0r3N6r3W2E0k6g2)9J5k6s2c8J5j5i4k6W2L8q4)9J5k6r3c8W2j5Y4g2Y4k6$3W2F1k6#2)9J5k6r3y4S2L8r3I4K6i4K6u0V1L8$3u0B7k6h3y4@1M7H3`.`.) 命令列出所有对 `clrjit!CILJit::compileMethod` 方法（x32 位）的调用，随后可使用 `!tt` 跳转到相应的时间点，然后设断点执行，针对此次 Jit 编译方法进行需要信息的提取，最后可以通过 [Windbg JavaScript Debugger Scripting](elink@201K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6D9k6h3q4J5L8W2)9J5k6h3#2A6j5%4u0G2M7$3!0X3N6q4)9J5k6h3y4G2L8g2)9J5c8X3g2F1i4K6u0V1N6i4y4Q4x3V1k6%4K9h3&6V1L8%4N6K6i4K6u0V1K9r3q4J5k6s2N6S2M7X3g2Q4x3V1k6V1M7X3W2$3k6i4u0K6i4K6u0r3k6r3g2T1N6h3N6Y4k6i4u0Q4x3V1k6B7j5i4k6S2M7$3y4J5K9i4m8@1i4K6u0V1k6r3g2T1N6h3N6Y4k6i4u0Q4x3X3c8K6j5%4u0A6M7s2c8A6L8X3M7`.) 来自动化这个单次过程，即可实现对所有执行 Jit 编译方法信息的提取。

![](https://bbs.kanxue.com/upload/attach/202506/802108_Q2ZB9PG4UPJR78R.png)

最后所有恢复方法需要的信息输出到一个 Json 文件中，再使用 .NET C# 写一个项目加载目标被加壳保护文件和此 Json 文件，参考 JitUnpacker-Framework 借助 dnlib 进行修复还原。

调试过程
----

现在我们根据上面的分析，来具体看下怎么进行断点调试和分析，执行一遍单次调试定位提取恢复方法需要的信息。

在 Windbg 中执行命令 `lm`，可以看到 clrjit.dll 文件符号所在的位置 `C:\ProgramData\Dbg\sym\clrjit.pdb\97077D9E2E3C48B29B28B6E5E35FEC932\clrjit.pdb`，这是我自己系统 .NET Framework 4.8+ 的 JIT 引擎文件，想确定系统 Framework 具体版本的话，可以进 `C:\Windows\Microsoft.NET\Framework\v4.0.xxxxx` 目录执行命令 `MSBuild -version` 查看。

在使用 JitUnpacker-Framework 工具进行对壳修复时候，就需要检查下自己系统的 .NET Framework 4.8 以下，否则在执行 `RuntimeFunctionConfigGenerator.bat` 会报错获取符号偏移失败。经过我测试，虚拟机里面启用 Windows 1803 版本，自带的是 .NET Framework 4.7.3+ 的版本，也是可以正常获取到符号偏移进行脱壳修复的。

```
PS C:\Windows\Microsoft.NET\Framework\v4.0.30319> .\MSBuild -version
Microsoft(R) 生成引擎版本 4.8.9037.0
[Microsoft .NET Framework 版本 4.0.30319.42000]
版权所有 (C) Microsoft Corporation。保留所有权利。
```

接着可以借助 `clrjit.pdb` 使用 IDA 来分析 `clrjit.dll`，找到 `CILJit::compileMethod` 函数，看到自动分析出了是 `stdcall` 传参，也可以通过调试来确定。我们要获取的是第三个参数 `struct CORINFO_METHOD_INFO* info`，断在 `CILJit::compileMethod` 函数后直接通过 `esp+C` 取到。

![](https://bbs.kanxue.com/upload/attach/202506/802108_2J2UNTGJZBQA8BP.png)

随后找到 `Compiler::lvaInitTypeRef` 方法，结合调试定位到 `localsSig = info.compCompHnd->getArgNext(localsSig)` 位置，下面不远就是循环结束的地方，可在那里设一个断点，最后一次执行 `getArgNext` 函数的结果还存放在 `eax` 寄存器中，断后直接获取即可。

![](https://bbs.kanxue.com/upload/attach/202506/802108_VHG5AFFYJVFBQAP.png)

最后是异常子句信息，前面已经获取到了数量，找到 `Compiler::fgFindBasicBlocks`，结合调试定位到 `info.compCompHnd->getEHinfo(info.compMethodHnd, XTnum, &clause)`，断在其执行完可通过 `clause` 指针所处栈偏移解析获取异常子句信息。

![](https://bbs.kanxue.com/upload/attach/202506/802108_8T2GEKPH9HEPE82.png)

JS 自动化提取信息
----------

写 `Windbg Javascript` 脚本来进行自动化上面的过程，除了阅读[官方文档](elink@ef2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6D9k6h3q4J5L8W2)9J5k6h3#2A6j5%4u0G2M7$3!0X3N6q4)9J5k6h3y4G2L8g2)9J5c8X3g2F1i4K6u0V1N6i4y4Q4x3V1k6%4K9h3&6V1L8%4N6K6i4K6u0V1K9r3q4J5k6s2N6S2M7X3g2Q4x3V1k6V1M7X3W2$3k6i4u0K6i4K6u0r3k6r3g2T1N6h3N6Y4k6i4u0Q4x3V1k6B7j5i4k6S2M7$3y4J5K9i4m8@1i4K6u0V1k6r3g2T1N6h3N6Y4k6i4u0Q4x3X3c8K6j5%4u0A6M7s2c8A6L8X3M7`.)，还可以参考 [WinDbg JavaScript Scripts](elink@e3fK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6Z5N6h3N6K6P5g2)9J5c8Y4N6A6L8X3c8T1k6#2)9#2k6X3A6K6i4K6g2X3M7$3y4J5K9i4m8@1M7H3`.`.) 项目仓库中丰富的实现。这里简单介绍下我实现的一些辅助工具代码，完整的实现可以下载附件查看，使用 Windbg 调试也可以先跟着里面的命令走一遍。

先定义一个 Windbg 类里面包含常用交互方法，包括日志输出、命令执行和内存与寄存器值获取。

```
class Windbg{
    static log = x => host.diagnostics.debugLog(`${x}\n`)
 
    static show = x => {
        for (var line of x) {
            Windbg.log(line);
        }
    }
 
    static system = x => host.namespace.Debugger.Utility.Control.ExecuteCommand(x)
 
    static getUint32(addr){
        return host.memory.readMemoryValues(addr, 1, 4)[0]
    }
 
    static getBytesArr(addr,size){
        return host.memory.readMemoryValues(addr, size, 1)
    }
 
    static getRegVal(reg){
        return host.currentThread.Registers.User[reg]
    }
}
```

方法信息和异常子句信息内容项比较多，我们简单定义两个类，这样读其结构体内存数据时候，可以通过构造方法实现一个反序列化的效果，这样解析就简洁方便很多。

```
function ArrToDatView(arr){
    var buffer = new ArrayBuffer(arr.length);
    var uint8Array = new Uint8Array(buffer);
    arr.forEach((value, index) => {
        uint8Array[index] = value;
    });
 
    return new DataView(buffer);
}
 
class MethodInfo{
    constructor(dataView, offset = 0){
        this.method_handle = dataView.getUint32(offset, true);
        this.module_handle = dataView.getUint32(offset + 4, true);
        this.ilcode_addr =  dataView.getUint32(offset + 8, true);
        this.ilcode_size =  dataView.getUint32(offset + 12, true);
        this.max_stack =  dataView.getUint32(offset + 16, true);
        this.eh_count =  dataView.getUint32(offset + 20, true);
    }
}
 
class EHClauseInfo{
        constructor(dataView, offset = 0) {
        this.HandlerType = dataView.getUint32(offset, true);
        this.TryOffset = dataView.getUint32(offset + 4, true);
        this.TryLength = dataView.getUint32(offset + 8, true);
        this.HandlerOffset = dataView.getUint32(offset + 12, true);
        this.HandlerLength = dataView.getUint32(offset + 16, true);
        this.CatchTypeTokenOrFilterOffset = dataView.getUint32(offset + 20, true);
    }
}
 
var arr = Windbg.getBytesArr(method_info_addr,24);
var method_info = new MethodInfo(ArrToDatView(arr));
 
 
var arr = Windbg.getBytesArr(ptr,0x1C);
var clauseInfo = new EHClauseInfo(ArrToDatView(arr));
```

最后要输出数据到 Json 文件中，还要写一个写数据到文件中的函数：

```
function WriteToFile(path,data) {
    var logFile;
    if (host.namespace.Debugger.Utility.FileSystem.FileExists(path)) {
        logFile = host.namespace.Debugger.Utility.FileSystem.CreateFile(path, "CreateNew");
    } else {
        logFile = host.namespace.Debugger.Utility.FileSystem.CreateFile(path);
    }
    var textWriter = host.namespace.Debugger.Utility.FileSystem.CreateTextWriter(logFile, "Utf8");
    try {
            textWriter.WriteLine(data)
    } finally {
        logFile.Close();
    }
}
```

额外的，我还使用 Gemini 2.5 Pro 生成了一个规则过滤模块代码，可以自己定义要提取的 Jit 方法信息，就不用每次执行都记录所有被 Jit 编译的函数了。

```
规则说明:
target: "module" 或 "method"
pattern: 要匹配的字符串
type: "include" (包含此模式的才处理) 或 "exclude" (包含此模式的不处理)
matchType: (可选) "contains" (默认), "exact", "startsWith", "endsWith"
 
规则处理逻辑:
1. 排除规则 (exclude) 具有最高优先级。如果任何排除规则匹配，则目标（模块或方法）将被跳过。
2. 如果存在包含规则 (include) 针对某个目标（模块或方法），则该目标必须至少匹配一个包含规则才能被处理。
   如果没有包含规则针对某个目标，则该目标默认被视为通过包含检查（除非被排除规则排除）。
3. 模块规则先应用，然后是方法规则。如果模块被过滤掉，其下的所有方法都不会被处理。
```

一套下来，主函数是这样的

```
const filterRules = [
    {target:FilterTargets.MODULE, type:FilterTypes.INCLUDE, matchType:FilterMatchTypes.CONTAINS, pattern:"FileCrypto.exe"},
    {target:FilterTargets.METHOD, type:FilterTypes.INCLUDE, matchType:FilterMatchTypes.CONTAINS, pattern:"Process()"},
];
 
Windbg.log('Start...');
 
Windbg.system("!tt 100");
const startTime = Date.now();
var jitMethodCount = parseInt(Windbg.system(`dx -r2 @$cursession.TTD.Calls("clrjit!CILJit::compileMethod").Count()`)[0].split(": ")[1].trim(),16);
Windbg.log(`Total JIT compilations to process: ${jitMethodCount}`);
for (var index = 0; index < jitMethodCount; index++) {
    const elapsedTime = ((Date.now() - startTime) / 1000).toFixed(2);
    Windbg.log(`Progress: ${index}/${jitMethodCount} ${((index / jitMethodCount) * 100).toFixed(2)}%, Elapsed Time: ${elapsedTime}s`);
    Windbg.system(`dx -r2 @$cursession.TTD.Calls("clrjit!CILJit::compileMethod")[0x${index.toString(16)}].@"TimeStart".SeekTo()`)
    try {
        extractFuncInfo(filterRules);
    } catch (e) {
        Windbg.log(`Error processing index ${index}: ${e.message}`);
        Windbg.log(`Stack: ${e.stack}`);
    }
}
Windbg.log("Using filter rules: " + JSON.stringify(filterRules, null, 2));
 
WriteToFile("D:\\FileCrypto.json", JSON.stringify({"ModulesInfo":modulesInfo}, null, 2));
Windbg.log('Finished extracting JIT information.');
```

最终的 Json 输出效果是这样的

![](https://bbs.kanxue.com/upload/attach/202506/802108_EJ3G9M7Q4JVGBTH.png)

C# JitPatcher 恢复方法
------------------

获取到方法信息后就简单多了，先写一些用于 Json 文件内容反序列化的结构类型，再读取 Json 文件后进行反序列化。

<table border="0" cellpadding="0" cellspacing="0"><tbody><tr><td>1234567891011121314151617181920212223242526272829303132333435</td><td><code>public</code> <code>class</code> <code>JitDumpInfo</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>List&lt;ModuleDumpInfo&gt; ModulesInfo { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>}</code><code>public</code> <code>class</code> <code>ModuleDumpInfo</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>string</code> <code>ModuleName { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>List&lt;MethodDumpInfo&gt; MethodsInfo { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>}</code><code>public</code> <code>class</code> <code>MethodDumpInfo</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>string</code> <code>MethodName { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>uint</code> <code>MethodToken { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>string</code> <code>ILBytes { </code><code>get</code><code>; </code><code>set</code><code>; } </code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>ushort</code> <code>MaxStack { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>string</code> <code>LocalsSignatureBytes { </code><code>get</code><code>; </code><code>set</code><code>; } </code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>List&lt;ExceptionHandlerInfo&gt; ExceptionHandlers { </code><code>get</code><code>; </code><code>set</code><code>; } = </code><code>new</code> <code>List&lt;ExceptionHandlerInfo&gt;();</code><code>}</code>&nbsp;<code>public</code> <code>class</code> <code>ExceptionHandlerInfo</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>int</code>&nbsp; <code>HandlerType { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>uint</code> <code>TryStartOffset { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>uint</code> <code>TryEndOffset { </code><code>get</code><code>; </code><code>set</code><code>; } </code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>uint</code> <code>HandlerStartOffset { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>uint</code> <code>HandlerEndOffset { </code><code>get</code><code>; </code><code>set</code><code>; } </code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>uint</code> <code>CatchTypeTokenOrFilterOffset { </code><code>get</code><code>; </code><code>set</code><code>; }</code><code>}</code>&nbsp;<code>JitDumpInfo jitDumpInfo = </code><code>null</code><code>;</code><code>try</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>string</code> <code>jsonData = File.ReadAllText(jsonDataPath);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>jitDumpInfo = JsonSerializer.Deserialize&lt;JitDumpInfo&gt;(jsonData, </code><code>new</code> <code>JsonSerializerOptions { PropertyNameCaseInsensitive = </code><code>true</code> <code>});</code><code>}</code></td></tr></tbody></table>

之后就是参考 [JitUnpacker-Framework](elink@1dbK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6%4N6$3R3I4x3o6l9@1i4K6u0r3d9X3W2@1g2h3&6H3j5h3y4C8k6i4u0Q4x3X3c8r3M7X3q4E0k6i4N6G2M7X3D9`.) 项目借助强大的 `dnlib` 库进行修复了，主要部分的实现逻辑可见附件代码。

总结
==

本文记录了笔者使用 Windbg TTD 还原 .NET JIT 保护壳的探索过程，提出了使用 Windbg 针对 JITUnpacker-Framework 不支持的 .NET Framework 4.8+ 版本 JIT 保护进行恢复还原的方案，相比之下会更为灵活一些，需要使用者手动定位分析一些内容。

以及 Windbg TTD 录制程序应该是向进程注入了记录器 dll 随后像 Pin 一样进行动态二进制插桩记录，如果没有针对性检测的话，常见的反调试手段都会失效，所以有时候如果目标程序的反调试检测比较严格的话，试一下 Windbg TTD 方案可能会有意想不到的效果。

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 20 小时前 被 0x 指纹编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm) [#.NET 平台](forum-4-1-7.htm)

上传的附件：

*   [JitExtractor.js](javascript:void(0);) （10.98kb，4 次下载）
*   [JitPatcher.cs](javascript:void(0);) （9.98kb，4 次下载）