> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282821.htm)

> [原创] 什么？IL2CPP APP 分析这一篇就够啦！

[原创] 什么？IL2CPP APP 分析这一篇就够啦！

发表于: 27 分钟前 42

### [原创] 什么？IL2CPP APP 分析这一篇就够啦！

 [![](http://passport.kanxue.com/upload/avatar/425/753425.png?1677809830)](user-home-753425.htm) [三六零 SRC](user-home-753425.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 27 分钟前  42

> 本文作者：SWDD@360SRC

前言
--

近年来，由 U3D 开发的游戏越来越多，诸如最近很火的手游版 “永劫无间” 等等，因此针对于 U3D 游戏安全的保护也越来越高级，目前大多数厂商都会选择 IL2CPP 来编译游戏。即便如此，只使用简单的 IL2CPP 虽然在反编译上极大的增加了难度，但是由于 C#+.Net 的特性，无法像传统的 ELF，EXE 文件等完全抹除符号，所以还是给破解者留下了很大的操作空间，破解者可以通过对内存的读写来绕过游戏内一些机制的判定，使得游戏运营防遭受损失，因此本篇文章将针对于 IL2CPP 这一技术探究一下逆向与破解的过程。

最近刷抖音又发现了一款小爆款游戏，叫做我们是战士，后续调查发现抖音上那一款估计是套壳游戏，而原本这个游戏的名字叫做《We Are Warriors!》，但是由于这个游戏某些恶心的设定，于是决定逆向这个游戏看看到底是怎么回事，解压 APK 发现了该程序妥妥的 U3D 结构，并且使用了 IL2CPP, 于是便诞生了这篇总结文章。注意本文中不会涉及到对该游戏的逆向，而是利用 demo 程序等做同等的操作替换。

Unity3D 项目结构
------------

在开始分析 il2cpp 之前，首先我们了解一下一个 unity app 的项目结构：

```
AppName/
├── Assets/         # 包含所有游戏资源和脚本文件
├── Library/        # Unity的库文件和缓存，自动生成
├── ProjectSettings/ # Unity项目设置文件夹
├── Packages/       # Unity Package Manager (UPM) 的依赖包
├── obj/            # 中间对象文件夹，用于编译时
├── Temp/           # 临时文件夹，包括临时生成的资源
├── Build/          # 打包输出目录，包括生成的App文件和数据
├── Logs/           # 日志文件夹
├── Packages/       # Unity Package Manager (UPM) 的依赖包
└── ProjectSettings/ # Unity项目设置文件夹

```

在生成的时候 \ Temp\StagingArea 目录下则会产生安卓编译时所需要的内容。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_XVZGK82HTMRFBVC.webp)  
生成 APK 后的结构如下：

```
AppName/
├── AndroidManifest.xml    # Android应用清单文件
├── assets/                # 包含资源文件夹，如图像和声音
├── res/                   # 包含资源文件夹，如布局和字符串
├── lib/                   # 包含原生库文件夹，如armeabi-v7a和arm64-v8a
├── META-INF/              # 包含APK签名的META-INF文件夹
├── classes.dex            # Android应用的主要DEX文件
├── resources.arsc         # 包含资源表文件
├── AndroidManifest.xml    # Android清单文件
├── res/                   # 包含资源文件夹，如布局和字符串
├── assets/                # 包含资源文件夹，如图像和声音
├── lib/                   # 包含原生库文件夹，如armeabi-v7a和arm64-v8a

```

打包成 APK 的 unity 项目实际上和正常的 unity 项目是一致的，其中的 Java 代码主要用于实现 unity 和 Android 平台的交互，是由 unity 自己生成的代码，因此我们在对 Unity 项目分析时主要关注的还是在 assets 目录下储存的项目信息。

Assets 目录结构：

```
│   bin\
│   │   Data\
│   │   │   ├── boot.config
│   │   │   ├── data.unity3d
│   │   │   ├── platform_native_link.xml
│   │   │   ├── resources.resource
│   │   │   ├── RuntimeInitializeOnLoads.json
│   │   │   ├── ScriptingAssemblies.json
│   │   │   └── unity default resources
│   │   │   Managed\
│   │   │   │   dll文件\
│   │   Managed\
│   │   │   dll文件\
│   Data\
│   │   ├── boot.config
│   │   ├── data.unity3d
│   │   ├── platform_native_link.xml
│   │   ├── resources.resource
│   │   ├── RuntimeInitializeOnLoads.json
│   │   ├── ScriptingAssemblies.json
│   │   └── unity default resources
│   │   Managed\
│   │   │   dll文件\
│   Managed\
│   │   dll文件\
bin\
│   Data\
│   │   ├── boot.config
│   │   ├── data.unity3d
│   │   ├── platform_native_link.xml
│   │   ├── resources.resource
│   │   ├── RuntimeInitializeOnLoads.json
│   │   ├── ScriptingAssemblies.json
│   │   └── unity default resources
│   │   Managed\
│   │   │   dll文件\
│   Managed\
│   │   dll文件\
Data\
│   ├── boot.config
│   ├── data.unity3d
│   ├── platform_native_link.xml
│   ├── resources.resource
│   ├── RuntimeInitializeOnLoads.json
│   ├── ScriptingAssemblies.json
│   └── unity default resources
│   Managed\
│   │   dll文件\
Managed\
│   dll文件\

```

游戏的主要脚本逻辑就在 assets\bin\Data\Managed\Assembly-CSharp.dll 中，因此针对于使用 Mono 打包的 Unity 3D 项目直接使用 ILSPY 或者使用 DNSPY 就可以实现反编译，并且阅读性与源码基本无异。因为如此，导致了 Mono 打包的 Unity App 存在着极高的被破解风险，由此现在大多数的 Unity 游戏都不使用 Mono 打包了，都开始使用 iL2cpp，那么 iL2cpp 是如何提升逆向破解难度的呢？请继续往下看。

IL2CPP 分析
---------

IL2CPP 是 Unity 引入的一种新的脚本后处理方式，用于加强对编译后代码的保护。在 Unity 开发中，人物操作、攻击、伤害和死亡判断通常通过 C# 脚本实现。IL2CPP 通过将 C# 脚本编译为 C++ 代码，然后再编译为本地代码，提供了额外的安全层。这种方式使得反编译和逆向工程变得更加困难，有助于保护知识产权和游戏内容的安全性。

### IL2CPP 生成分析

IL2CPP 的代码位于 \ Editor\Data\il2cpp\libil2cpp\codegen，通过分析可以发现，IL2CPP 采用了一种类似于虚拟机的机制。它通过将 C# 代码编译成中间语言（IL），然后再将 IL 代码转换成 C++ 代码，最终编译为本地机器码。在这个过程中，IL2CPP 对代码进行了多层次的优化和处理。

IL2CPP 在将 C# 代码编译成 IL 代码时，会对代码进行一定程度的优化，例如移除不必要的代码和进行常量折叠。接着，IL 代码会被转化为 C++ 代码。在这个阶段，IL2CPP 生成的 C++ 代码不仅包含了原始 C# 代码的逻辑，还加入了一些辅助的代码，用于实现运行时环境和垃圾回收等功能。生成的 C++ 代码会被编译为本地机器码。这一步通常会使用平台相关的编译器，以确保生成的代码在目标平台上具有最佳的性能和兼容性。由于 C++ 代码在编译后变成了本地机器码，反编译的难度大大增加，从而增强了代码的安全性。IL2CPP 还通过各种手段来优化代码执行的效率。它会对频繁执行的代码路径进行优化，以减少运行时的开销；它还会使用高效的数据结构和算法，以提高整体性能。

iL2cpp 编译过程首先是将 C# 的脚本，还有 Unity 引擎的代码，Boo 代码通过各自的编译器编译为 IL 指令代码，然后还有一些其他的 IL 代码一起通过 iL2cpp 转化成 C++ 代码，然后通过 C++ 编译成 libIL2cpp.so，再由 Il2cpp 提供的虚拟机对代码进行解释和运行。在其源码的结构中，也可以发现其两个部分的代码。

![](https://bbs.kanxue.com/upload/attach/202408/753425_JBEWH4AP34BVF5Z.webp)

将流程转换成线性图则如下图所示：  
![](https://bbs.kanxue.com/upload/attach/202408/753425_CK6G75EBTFYQTYM.webp)

### IL2CPP 加载分析

根据 IL2CPP 的生成过程，我们会发现游戏的逻辑都到了 Native 运行，那么 C# 的语言特性需要如何继续实现呢，我们可以看 vm 中的代码。

在 \ Unity Edit\2021.3.22f1c1\Editor\Data\il2cpp\libil2cpp\vm\GlobalMetadata.cpp 中我们可以发现如下代码

```
bool il2cpp::vm::GlobalMetadata::Initialize(int32_t* imagesCount, int32_t* assembliesCount)
{
    s_GlobalMetadata = vm::MetadataLoader::LoadMetadataFile("global-metadata.dat");
    if (!s_GlobalMetadata)
        return false;
 
    s_GlobalMetadataHeader = (const Il2CppGlobalMetadataHeader*)s_GlobalMetadata;
    IL2CPP_ASSERT(s_GlobalMetadataHeader->sanity == 0xFAB11BAF);
    IL2CPP_ASSERT(s_GlobalMetadataHeader->version == 29);
    IL2CPP_ASSERT(s_GlobalMetadataHeader->stringLiteralOffset == sizeof(Il2CppGlobalMetadataHeader));
 
    s_MetadataImagesCount = *imagesCount = s_GlobalMetadataHeader->imagesSize / sizeof(Il2CppImageDefinition);
    *assembliesCount = s_GlobalMetadataHeader->assembliesSize / sizeof(Il2CppAssemblyDefinition);
 
    // Pre-allocate these arrays so we don't need to lock when reading later.
    // These arrays hold the runtime metadata representation for metadata explicitly
    // referenced during conversion. There is a corresponding table of same size
    // in the converted metadata, giving a description of runtime metadata to construct.
    s_MetadataImagesTable = (Il2CppImageGlobalMetadata*)IL2CPP_CALLOC(s_MetadataImagesCount, sizeof(Il2CppImageGlobalMetadata));
    s_TypeInfoTable = (Il2CppClass**)IL2CPP_CALLOC(s_Il2CppMetadataRegistration->typesCount, sizeof(Il2CppClass*));
    s_TypeInfoDefinitionTable = (Il2CppClass**)IL2CPP_CALLOC(s_GlobalMetadataHeader->typeDefinitionsSize / sizeof(Il2CppTypeDefinition), sizeof(Il2CppClass*));
    s_MethodInfoDefinitionTable = (const MethodInfo**)IL2CPP_CALLOC(s_GlobalMetadataHeader->methodsSize / sizeof(Il2CppMethodDefinition), sizeof(MethodInfo*));
    s_GenericMethodTable = (const Il2CppGenericMethod**)IL2CPP_CALLOC(s_Il2CppMetadataRegistration->methodSpecsCount, sizeof(Il2CppGenericMethod*));
 
    ProcessIl2CppTypeDefinitions(InitializeTypeHandle, InitializeGenericParameterHandle);
 
    return true;
}

```

这个代码在加载 global-metadata.dat，并且对其做了合法性判断。继续阅读后我们还会发现其使用了 GetStringLiteralFromIndex(StringLiteralIndex index) 等函数加载了字符信息，函数指针信息等一系列内容。

为了更好的分析，我们可以通过 010 导入 UnityMetadata.bt 的模板文件，使得文件的结构更加清晰。

```
//------------------------------------------------
//--- 010 Editor v13.0.1 Binary Template
//
//      File: UnityMetadata.bt
//   Authors: xia0
//   Version: 0.2
//   Purpose: Parse unity3d metadata file
//  Category: Game
// File Mask: *.dat
//  ID Bytes: FA B1 1B AF
//   History:
//   0.2    2023-03-24 avan: Automatically generate the string content of all StringLiterals based on the offset value of the StringLiteral in GlobalMetadataHeader.
//   0.1    2019-10-31 xia0: init basic unity3d metadata info version
//------------------------------------------------
// Blog: https://4ch12dy.site
// Github: https://github.com/4ch12dy
// https://www.sweetscape.com/010editor/manual/DataTypes.htm
// http://www.sweetscape.com/010editor/repository/templates/
 
typedef int32 TypeIndex;
typedef int32 TypeDefinitionIndex;
typedef int32 FieldIndex;
typedef int32 DefaultValueIndex;
typedef int32 DefaultValueDataIndex;
typedef int32 CustomAttributeIndex;
typedef int32 ParameterIndex;
typedef int32 MethodIndex;
typedef int32 GenericMethodIndex;
typedef int32 PropertyIndex;
typedef int32 EventIndex;
typedef int32 GenericContainerIndex;
typedef int32 GenericParameterIndex;
typedef int16 GenericParameterConstraintIndex;
typedef int32 NestedTypeIndex;
typedef int32 InterfacesIndex;
typedef int32 VTableIndex;
typedef int32 InterfaceOffsetIndex;
typedef int32 RGCTXIndex;
typedef int32 StringIndex;
typedef int32 StringLiteralIndex;
typedef int32 GenericInstIndex;
typedef int32 ImageIndex;
typedef int32 AssemblyIndex;
typedef int32 InteropDataIndex;
 
 
typedef struct Il2CppGlobalMetadataHeader
{
    int32 sanity ;
    int32 version;
    int32 stringLiteralOffset ;
    int32 stringLiteralCount;
    int32 stringLiteralDataOffset;
    int32 stringLiteralDataCount;
    int32 stringOffset ;
    int32 stringCount;
    int32 eventsOffset ;
    int32 eventsCount;
    int32 propertiesOffset ;
    int32 propertiesCount;
    int32 methodsOffset ;
    int32 methodsCount;
    int32 parameterDefaultValuesOffset ;
    int32 parameterDefaultValuesCount;
    int32 fieldDefaultValuesOffset ;
    int32 fieldDefaultValuesCount;
    int32 fieldAndParameterDefaultValueDataOffset; //uint8_t
    int32 fieldAndParameterDefaultValueDataCount;
    int32 fieldMarshaledSizesOffset ;
    int32 fieldMarshaledSizesCount;
    int32 parametersOffset ;
    int32 parametersCount;
    int32 fieldsOffset ;
    int32 fieldsCount;
    int32 genericParametersOffset ;
    int32 genericParametersCount;
    int32 genericParameterConstraintsOffset ;
    int32 genericParameterConstraintsCount;
    int32 genericContainersOffset ;
    int32 genericContainersCount;
    int32 nestedTypesOffset ;
    int32 nestedTypesCount;
    int32 interfacesOffset ;
    int32 interfacesCount;
    int32 vtableMethodsOffset ;
    int32 vtableMethodsCount;
    int32 interfaceOffsetsOffset ;
    int32 interfaceOffsetsCount;
    int32 typeDefinitionsOffset ;
    int32 typeDefinitionsCount;
    int32 rgctxEntriesOffset ;
    int32 rgctxEntriesCount;
    int32 imagesOffset ;
    int32 imagesCount;
    int32 assembliesOffset ;
    int32 assembliesCount;
    int32 metadataUsageListsOffset ;
    int32 metadataUsageListsCount;
    int32 metadataUsagePairsOffset ;
    int32 metadataUsagePairsCount;
    int32 fieldRefsOffset ;
    int32 fieldRefsCount;
    int32 referencedAssembliesOffset; // int32
    int32 referencedAssembliesCount;
    int32 attributesInfoOffset ;
    int32 attributesInfoCount;
    int32 attributeTypesOffset ;
    int32 attributeTypesCount;
    int32 unresolvedVirtualCallParameterTypesOffset ;
    int32 unresolvedVirtualCallParameterTypesCount;
    int32 unresolvedVirtualCallParameterRangesOffset ;
    int32 unresolvedVirtualCallParameterRangesCount;
    int32 windowsRuntimeTypeNamesOffset ;
    int32 windowsRuntimeTypeNamesSize;
    int32 exportedTypeDefinitionsOffset ;
    int32 exportedTypeDefinitionsCount;
} Il2CppGlobalMetadataHeader;
 
 
typedef struct Il2CppStringLiteralInfoDefinition
{
    uint32 Length;
    uint32 Offset;
} Il2CppStringLiteralInfoDefinition;
 
 
typedef struct (uint infoSize, uint stringLiteralDataOffset, Il2CppStringLiteralInfoDefinition StringLiteralInfos[])
{
    typedef struct (uint stringLiteralDataOffset, uint index, Il2CppStringLiteralInfoDefinition StringLiteralInfos[])
    {
        local uint infoOffset = StringLiteralInfos[index].Offset;
        local uint infoLength = StringLiteralInfos[index].Length;
        FSeek(stringLiteralDataOffset + infoOffset);
        if (infoLength > 0) char data[infoLength] ;
    } StringLiteralDefinition 0 ? data : "null")>;
    local uint index = 0;
    while (index + 1 < infoSize) StringLiteralDefinition StringLiteralDefinitions(stringLiteralDataOffset, index++, StringLiteralInfos);
} Il2CppStringLiteralDefinition;
 
 
typedef struct Il2CppImageDefinition
{
    StringIndex nameIndex;
    AssemblyIndex assemblyIndex;
 
    TypeDefinitionIndex typeStart;
    uint32 typeCount;
 
    TypeDefinitionIndex exportedTypeStart;
    uint32 exportedTypeCount;
 
    MethodIndex entryPointIndex;
    uint32 token;
 
    CustomAttributeIndex customAttributeStart;
    uint32 customAttributeCount;
} Il2CppImageDefinition;
 
 
#define PUBLIC_KEY_BYTE_LENGTH 8
typedef struct Il2CppAssemblyNameDefinition
{
    StringIndex nameIndex;
    StringIndex cultureIndex;
    StringIndex hashValueIndex;
    StringIndex publicKeyIndex;
    uint32 hash_alg;
    int32 hash_len;
    uint32 flags;
    int32 major;
    int32 minor;
    int32 build;
    int32 revision;
    ubyte public_key_token[PUBLIC_KEY_BYTE_LENGTH];
} Il2CppAssemblyNameDefinition;
 
typedef struct Il2CppAssemblyDefinition
{
    ImageIndex imageIndex;
    uint32 token;
    int32 referencedAssemblyStart;
    int32 referencedAssemblyCount;
    Il2CppAssemblyNameDefinition aname;
} Il2CppAssemblyDefinition;
 
typedef struct Il2CppTypeDefinition
{
    StringIndex nameIndex;
    StringIndex namespaceIndex;
    TypeIndex byvalTypeIndex;
    TypeIndex byrefTypeIndex;
 
    TypeIndex declaringTypeIndex;
    TypeIndex parentIndex;
    TypeIndex elementTypeIndex; // we can probably remove this one. Only used for enums
 
    RGCTXIndex rgctxStartIndex;
    int32 rgctxCount;
 
    GenericContainerIndex genericContainerIndex;
 
    uint32 flags;
 
    FieldIndex fieldStart;
    MethodIndex methodStart;
    EventIndex eventStart;
    PropertyIndex propertyStart;
    NestedTypeIndex nestedTypesStart;
    InterfacesIndex interfacesStart;
    VTableIndex vtableStart;
    InterfacesIndex interfaceOffsetsStart;
 
    uint16 method_count;
    uint16 property_count;
    uint16 field_count;
    uint16 event_count;
    uint16 nested_type_count;
    uint16 vtable_count;
    uint16 interfaces_count;
    uint16 interface_offsets_count;
 
    // bitfield to portably encode boolean values as single bits
    // 01 - valuetype;
    // 02 - enumtype;
    // 03 - has_finalize;
    // 04 - has_cctor;
    // 05 - is_blittable;
    // 06 - is_import_or_windows_runtime;
    // 07-10 - One of nine possible PackingSize values (0, 1, 2, 4, 8, 16, 32, 64, or 128)
    uint32 bitfield;
    uint32 token;
} Il2CppTypeDefinition;
 
typedef struct Il2CppMetadataUsageList
{
    uint32 start;
    uint32 count;
} Il2CppMetadataUsageList;
 
 
typedef struct Il2CppMetadataUsagePair
{
    uint32 destinationIndex;
    uint32 encodedSourceIndex;
} Il2CppMetadataUsagePair;
 
Il2CppGlobalMetadataHeader metadataHeader ;
 
local uint infoSize = metadataHeader.stringLiteralCount / sizeof(Il2CppStringLiteralInfoDefinition);
 
FSeek(metadataHeader.stringLiteralOffset);
Il2CppStringLiteralInfoDefinition StringLiteralInfoDefinitions[infoSize] ;
Il2CppStringLiteralDefinition StringLiteralDefinitions(infoSize, metadataHeader.stringLiteralDataOffset, StringLiteralInfoDefinitions) ;
 
FSeek(metadataHeader.imagesOffset);
Il2CppImageDefinition imagesDefinitions[metadataHeader.imagesCount / sizeof(Il2CppImageDefinition)] ;
 
FSeek(metadataHeader.assembliesOffset);
Il2CppAssemblyDefinition assemblyDefinitions[metadataHeader.imagesCount / sizeof(Il2CppImageDefinition)] ;
 
FSeek(metadataHeader.typeDefinitionsOffset);
Il2CppTypeDefinition typeDefinitions[metadataHeader.assembliesCount / sizeof(Il2CppAssemblyDefinition)] ;
 
FSeek(metadataHeader.metadataUsagePairsOffset);
Il2CppMetadataUsagePair metadataUsagePair[metadataHeader.metadataUsagePairsCount / sizeof(Il2CppMetadataUsagePair)] ;
 
FSeek(metadataHeader.metadataUsageListsOffset);
Il2CppMetadataUsageList metadataUsageList[metadataHeader.metadataUsageListsCount / sizeof(Il2CppMetadataUsageList)] ; 
```

在游戏被打包成 APK 后 metadata 被储存的路径位于 \ assets\bin\Data\Managed\Metadata 中，我们使用 010 对其分析，当我们导入并且运行模板的时候就可以在检查器中查找到各个结构体了

![](https://bbs.kanxue.com/upload/attach/202408/753425_9KPFAKMW8QAB5Z8.webp))

既然如此，我们则可以通过解析 global-metadata.dat 的信息来获取函数指针，并且通过偏移去查找 libil2cpp 中的游戏逻辑，当然仅凭这些，我们无法得知具体的字符串的，还需要利用到 ilbil2cpp.so 进行寻址的操作。

IL2CPP 逆向分析
-----------

由于在 libil2cpp 中我们是无法直接看到符号信息的，因此我们如果直接逆向 il2cpp 的话十分困难。但是我们可以通过 global-metadata.dat 以及 il2cpp 来恢复 libil2cpp 的符号信息。

### Il2cppDumper 使用以及分析

#### 源码分析

这里介绍一下 Il2cppDumper 的原理，并对其源码做一定的解释。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_7G7VN9QYG3EUW33.webp))

在执行 dump 之前，Il2cppDumper 会对 il2cpp 进行解析。

在 Init 方法中加载了 il2cpp。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_HM26H4HMM2CUMKG.webp)

之后，根据不同的 magic number 对二进制文件进行解析。

![](https://bbs.kanxue.com/upload/attach/202408/753425_DQBUA5JKHTG7BQW.webp))

后在 dump 中继续解析，完成 dll 的构建，其中用到了 Il2CppDecompiler 等反编译 il2cpp 的方法，用于提取其符号信息等等，这里就不过多赘述了。

<table border="0" cellpadding="0" cellspacing="0"><tbody><tr><td>123456789101112131415161718192021</td><td><code>private</code> <code>static</code> <code>void</code> <code>Dump(Metadata metadata, Il2Cpp il2Cpp, </code><code>string</code> <code>outputDir)</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Console.WriteLine(</code><code>"Dumping..."</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var</code> <code>executor = </code><code>new</code> <code>Il2CppExecutor(metadata, il2Cpp);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var</code> <code>decompiler = </code><code>new</code> <code>Il2CppDecompiler(executor);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>decompiler.Decompile(config, outputDir);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Console.WriteLine(</code><code>"Done!"</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(config.GenerateStruct)</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Console.WriteLine(</code><code>"Generate struct..."</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var</code> <code>scriptGenerator = </code><code>new</code> <code>StructGenerator(executor);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>scriptGenerator.WriteScript(outputDir);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Console.WriteLine(</code><code>"Done!"</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(config.GenerateDummyDll)</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>{</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Console.WriteLine(</code><code>"Generate dummy dll..."</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>DummyAssemblyExporter.Export(executor, outputDir, config.DummyDllAddToken);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Console.WriteLine(</code><code>"Done!"</code><code>);</code><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code><code>}</code></td></tr></tbody></table>

#### 使用 il2cppDumper 获取 Assembly-Csharp.dll

下载地址:[Perfare/Il2CppDumper: Unity il2cpp reverse engineer (github.com)](https://github.com/Perfare/Il2CppDumper)

根据刚刚的源码分析，我们也能发现 iL2cppDumper 的操作，运行 Il2CppDumper.exe 依次选择 libil2cpp.so 以及 global-metadata.dat，运行完成后显示如下界面（其实可以发现其输出界面和我们刚刚的流程是一致的）  
![](https://bbs.kanxue.com/upload/attach/202408/753425_CNM5QB6NTP5EDJH.webp)

则代表解密成功，如果报错，则可能是 global-metadata.dat 被加密了，这个时候我们首先需要解密 global-metadata.dat。解密该文件的方法有很多，可以通过分析起加密过程或者通过 Hook 动态 dump 等方式解密原本的 dll，可以使用 [Perfare/Zygisk-Il2CppDumper: Using Zygisk to dump il2cpp data at runtime (github.com)](https://github.com/Perfare/Zygisk-Il2CppDumper)

现在介绍正常 dump 下来的 dll，dump 下来之后我们需要关注几个关键文件：  
![](https://bbs.kanxue.com/upload/attach/202408/753425_F5HEEN9YU7PK285.webp)

这几个文件中我们首先需要看到的是 DummyDll 文件  
![](https://bbs.kanxue.com/upload/attach/202408/753425_2E3UP5FVNV774PW.webp)

其中可以找到我们在做 U3D 逆向时熟悉的 dll，但是需要注意的是这个 dll 并不会储存源码，我们能在这个 dll 文件中看到原本程序的方法名和变量名，还有程序的结构。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_YVB8Z6XBP7PWHXK.webp)

### U3D iL2cpp 游戏逆向实战

#### 初步分析

这里用到的是一个坦克大战小游戏的 demo，游戏主界面如下：  
![](https://bbs.kanxue.com/upload/attach/202408/753425_9VZUPM7DA6T2ECY.webp)

游戏可以通过购买坦克来增加自己的实力，那么我们就从白嫖他的坦克开始分析这个程序，首先还是用 Dnspy 或者使用 ILspy 去解析这个 Assembly-Csharp.dll 获取方法的参数和偏移。

既然要分析购买功能，那我们可以通过方法的关键词去搜索，类似于 Buy，Price，Coins 之类的，这里我们搜索 Buy。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_28QJAR2HVTZRG8A.webp)  
我们使用搜索功能（CTRL+F），这里还需要注意的是我们需要选择非泛型类型，这个其实就是找类了，熟悉开发的同学应该可以明白，很多功能都是在类中实现的。那么对于这个 demo 我们很快就搜索到了 BuyTank 类，我们可以点进去看看类里面有一些什么方法。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_8W97BDND5XENNVW.webp)  
哎，局势瞬间明了，购买逻辑是通过 Buy() 方法实现的，然后其中涉及到了一些成员变量，比如 price 和 textPrice 什么的，点开 Buy 方法，查看方法的一些基本信息。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_BAE8T33U6CP4N44.webp)  
这里可以发现，Buy 方法没有返回值，没有参数，那么购买逻辑肯定是在内部判断的，也就是意味着 Buy 方法内部存在获取用户金钱数量的逻辑，那么我们通过修改这个逻辑就可以白嫖他的坦克了。

#### 逻辑修改

不难发现的是 dll 程序中是没有实现逻辑的，逻辑存在于 il2cpp.dll 中，初步打开里面啥也没有，符号也没有什么都没有，因此我们需要恢复符号表。其实在我们 dump 下 dll 的时候恢复符号表的脚本就已经在 il2cppdumper 的目录下产生了，我们只需要使用其准备好的脚本即可。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_2CUMBM9KZYTR944.webp)  
在 IDA 中按 alt+f7 进入加载脚本界面，然后选择 ida_py3.py，选择 scripts.json 就可以恢复符号表了，值得注意的是产生的 ida database 文件大小可能会是以 G 为单位，需要预留足够的空间。

恢复符号表后我们可以直接在 IDA 中搜索 Buy，或者使用 Dnspy 提供的便宜寻找，这里我们已经知道偏移是 0x19a2928，使用 IDA 的 G 键跳转过去即可。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_SVBN6DH3JAMXEUZ.webp)  
逻辑比较杂乱，那么我们依旧需要借助 Dnspy 中的信息来分析

对于 Buy 逻辑，price 肯定是一个重要的变量，那么我们就可以在 Dnspy 中查找到他的偏移地址。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_5W7JHT3DTCPHC2A.webp)  
这里可以发现偏移地址是 0x48  
![](https://bbs.kanxue.com/upload/attach/202408/753425_U7MGRH4DZ4GQTCZ.webp)  
那么这个 v1+0x48 则是我们的坦克的价格了。

于此同时我们还可以在 gameDataBase 中找到 money 的偏移。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_RFAE5X7SYE27AC7.webp)  
正好是 0x18，那么就可以确定 这是在判断是否买得起了。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_6XY62PECTQUB77N.webp)  
那么我们只需要 trace 到 FCMP S0, S1 这一行并且保证 S0>S1 就可以了，Frida 代码如下:

```
function main() {
    Java.perform(function () {
 
        var module = Process.getModuleByName("libil2cpp.so");
        var addr = module.base.add("0x19A299C");
        var func = new NativePointer(addr.toString());
        Interceptor.attach(func, {
            onEnter: function (args) {
                this.context.s0 = 50000;
            },
            onLeave: function (retval) {
               console.log('[+] Method onLeave : ', retval);
            }
        });
 
    });
}
 
setTimeout(main, 250);

```

当我们购买时可以发现钱数量减少成复数，并且成功购买。  
![](https://bbs.kanxue.com/upload/attach/202408/753425_35PCSYT49CRM7TH.webp)

扩展
--

当然上文中有提到 [Perfare/Zygisk-Il2CppDumper: Using Zygisk to dump il2cpp data at runtime (github.com)](https://github.com/Perfare/Zygisk-Il2CppDumper) 项目，这是一个动态的 dump 的 Magisk 模块，通常在上面的 Il2cppDumper 无法静态的获取符号信息的时候我们用这个来处理。

首先下载源码之后使用 Android Studio 打开：  
![](https://bbs.kanxue.com/upload/attach/202408/753425_EUWJ5D4CYVB5W4X.webp)  
修改 game.h 中的包名之后，选择 Gradle 中的编译模式。后再 Build 中 Make Project 即可：

![](https://bbs.kanxue.com/upload/attach/202408/753425_XGZ8GAJ4X2VS46M.webp)  
导入模块运行之后 app 私有目录下 file 文件夹中就会存在 dump.cs  
![](https://bbs.kanxue.com/upload/attach/202408/753425_QVBVK23YJ7DVTJA.webp)

总结
--

在逆向工程中，IL2CPP 为 Unity 应用程序引入了额外的复杂性，通过将 C# 脚本转换为 C++ 代码再编译成本地机器码，从而提高了反编译和逆向工程的难度。然而，借助工具和方法，如 IL2CPPDumper 和 Frida，可以有效地分析和修改 IL2CPP 应用程序。

通过本文的分析和实践，我们了解了 Unity3D 项目的基本结构，以及 IL2CPP 的生成和加载过程。我们通过使用 IL2CPPDumper 成功地提取了关键的元数据和符号信息，并通过反编译工具和动态分析工具对其进行了深入分析和修改。具体的步骤包括解析 global-metadata.dat 文件，恢复 libil2cpp.so 的符号表，以及使用 Frida 进行运行时修改。

值得注意的是，随着技术的发展，各大厂商对 global-metadata.dat 文件的加密手段层出不穷，但无论如何加密，这些数据最终都会在运行时被解密并使用。因此，利用 Hook 等动态分析手段是获取解密后数据的有效方法。

在实际操作中，Frida 提供了强大的动态修改和调试功能，可以避免重打包和签名绕过的问题，极大地简化了逆向工程的过程。通过本文的实例演示，我们成功地修改了游戏的购买逻辑，实现了零成本购买道具的效果。

总的来说，IL2CPP 逆向工程虽然复杂，但通过合理利用现有工具和技术手段，可以有效地进行分析和修改，满足逆向工程的需求。希望本文的总结能够对读者在 IL2CPP 逆向工程中提供有价值的参考和帮助。

  

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 24 分钟前 被三六零 SRC 编辑 ，原因：