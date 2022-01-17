> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271191.htm)

> [原创]Unity Il2cpp 应用逆向分析

前言
==

关于 Il2cpp 逆向的文章网上已经有很多了，但是最近自己实践的时候发现很多文章对于逆向的技术细节讲的不是很清楚，提供的工具也不能支持最新版本的 Unity，所以决定基于 Ghidra 从头写一个 Il2cpp 应用的分析脚本，这篇文章就是对自己最近学习的一个总结，适合从零开始搞 Unity 逆向的同学。我们就从一个最简单的 CrackMe 应用开始分析，目标就是分析出正确的 key。  
![](https://bbs.pediy.com/upload/tmp/748271_HZFTUDN9SBZPY3R.png)

工具
==

Unity 2021.2.0f1c1  
Ghidra 10.0.4  
Apktool  
Vscode

Il2cpp 简介
=========

![](https://bbs.pediy.com/upload/tmp/748271_USK9MSWBEF9F8NK.png)  
关于 Il2cpp 技术网上已经有很多文章详细讲解了，这里就不做过多讲述，简单来说就是通过 Il2cpp 程序把 IL 转换成成 C++ 代码，然后再用 native 的编译器把 C++ 代码编译成目标平台的可执行代码，对于 Android 应用来说就是 so 文件。这个过程中最关键的就是 C++ 代码，这部分代码分为两个部分一部分是 Il2cpp 框架代码，这部分代码就在我们安装的 Unity 目录下，不同版本和平台的代码位置可能会不一样，我现在用的 mac 版的 Unity 就在 / Applications/Unity/Hub/Editor/2021.2.0f1c1/Unity.app/Contents/il2cpp 这个目录下。还有一部分是应用源码编译时生成的，这个目录视配置而定。实际我们在分析其它应用时是不能拿到应用源码的，这里用到是为了方便学习 Il2cpp 应用的执行过程，下面会用 Il2cpp 源码和应用源码来简称这两部分代码。

目录结构
====

用 Apktool 解包 apk 文件后可以看到，Il2cpp 应用的目录结构如下。

```
.
├── AndroidManifest.xml
├── apktool.yml
├── assets
│   └── bin
│       └── Data
│           ├── Managed
│           │   ├── Metadata
│           │   │   └── global-metadata.dat
│           │   └── Resources
│           │       └── mscorlib.dll-resources.dat
│           ├── RuntimeInitializeOnLoads.json
│           ├── ScriptingAssemblies.json
│           ├── boot.config
│           ├── data.unity3d
│           ├── unity default resources
│           └── unity_app_guid
├── lib
│   ├── arm64-v8a
│   │   ├── lib_burst_generated.so
│   │   ├── libil2cpp.so
│   │   ├── libmain.so
│   │   └── libunity.so
│   └── armeabi-v7a
│       ├── lib_burst_generated.so
│       ├── libil2cpp.so
│       ├── libmain.so
│       └── libunity.so
├── original
│   ├── AndroidManifest.xml
│   └── META-INF
│       ├── CERT.RSA
│       ├── CERT.SF
│       └── MANIFEST.MF
└── res
    ├── mipmap-anydpi-v26
    │   ├── app_icon.xml
    │   └── app_icon_round.xml
    ├── mipmap-mdpi
    │   ├── app_icon.png
    │   ├── ic_launcher_background.png
    │   └── ic_launcher_foreground.png
    ├── values
    │   ├── ids.xml
    │   ├── public.xml
    │   ├── strings.xml
    │   └── styles.xml
    └── values-v28
        └── styles.xml

```

这里我们主要关心 global-metadata.dat 和 libil2cpp.so 连个文件，其中 global-metadata.dat 是 Il2cpp 翻译 C++ 代码之后存放类型和符号信息的文件，libil2cpp.so 文件就是应用业务逻辑所在的文件。将 global-metadata.dat 中的类型和符号信息解析出来定位到 libil2cpp.so 中才能更方便的去做逆向分析。

程序入口分析
======

首先我们要找到在 Il2cpp 源码中 global-metadata 是如何解析的，在源码中全局搜索 global-metadata 后我们可以找到这段代码。

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
 
    s_MetadataImagesCount = *imagesCount = s_GlobalMetadataHeader->imagesCount / sizeof(Il2CppImageDefinition);
    *assembliesCount = s_GlobalMetadataHeader->assembliesCount / sizeof(Il2CppAssemblyDefinition);
 
    // Pre-allocate these arrays so we don't need to lock when reading later.
    // These arrays hold the runtime metadata representation for metadata explicitly
    // referenced during conversion. There is a corresponding table of same size
    // in the converted metadata, giving a description of runtime metadata to construct.
    s_MetadataImagesTable = (Il2CppImageGlobalMetadata*)IL2CPP_CALLOC(s_MetadataImagesCount, sizeof(Il2CppImageGlobalMetadata));
    s_TypeInfoTable = (Il2CppClass**)IL2CPP_CALLOC(s_Il2CppMetadataRegistration->typesCount, sizeof(Il2CppClass*));
    s_TypeInfoDefinitionTable = (Il2CppClass**)IL2CPP_CALLOC(s_GlobalMetadataHeader->typeDefinitionsCount / sizeof(Il2CppTypeDefinition), sizeof(Il2CppClass*));
    s_MethodInfoDefinitionTable = (const MethodInfo**)IL2CPP_CALLOC(s_GlobalMetadataHeader->methodsCount / sizeof(Il2CppMethodDefinition), sizeof(MethodInfo*));
    s_GenericMethodTable = (const Il2CppGenericMethod**)IL2CPP_CALLOC(s_Il2CppMetadataRegistration->methodSpecsCount, sizeof(Il2CppGenericMethod*));
 
    ProcessIl2CppTypeDefinitions(InitializeTypeHandle, InitializeGenericParameterHandle);
 
    return true;
}

```

分析代码可以知道 global-metadata.dat 这个文件是直接映射到内存中的，可以通过 Il2CppGlobalMetadataHeader 这个数据结构去解析，另外还有一个关键的数据结构 Il2CppMetadataRegistration 也参与到了初始化的过程中，我们再分析一下这个数据结构是什么时候初始化的。在同一个文件里可以找到这个函 il2cpp::vm::GlobalMetadata::Register 然后通过调用关系分析，我们可以找到在 Il2cpp 代码中只有 il2cpp::vm::MetadataCache::Register 这个函数调用了它，并且初始化了三个数据结构 Il2CppCodeRegistration，Il2CppMetadataRegistration 和 Il2CppCodeGenOptions。这三个数据结构是和源码相关的，我们去应用源码里搜索一下。

```
void s_Il2CppCodegenRegistration()
{
    il2cpp_codegen_register (&g_CodeRegistration, &g_MetadataRegistration, &s_Il2CppCodeGenOptions);
}
#if RUNTIME_IL2CPP
typedef void (*CodegenRegistrationFunction)();
CodegenRegistrationFunction g_CodegenRegistration = s_Il2CppCodegenRegistration;
#endif

```

在应用源码中可以找到这三个数据结构是怎么初始化的，再回到 Il2cpp 代码中去分析，这个函数是在哪里调用的。

```
    bool Runtime::Init(const char* domainName)
    {
        os::FastAutoLock lock(&s_InitLock);
 
        IL2CPP_ASSERT(s_RuntimeInitCount >= 0);
        if (s_RuntimeInitCount++ > 0)
            return true;
 
        SanityChecks();
 
        os::Initialize();
        os::Locale::Initialize();
        MetadataAllocInitialize();
 
        // NOTE(gab): the runtime_version needs to change once we
        // will support multiple runtimes.
        // For now we default to the one used by unity and don't
        // allow the callers to change it.
        s_FrameworkVersion = framework_version_for("v4.0.30319");
 
        os::Image::Initialize();
        os::Thread::Init();
 
#if !IL2CPP_TINY && !IL2CPP_MONO_DEBUGGER
        il2cpp::utils::DebugSymbolReader::LoadDebugSymbols();
#endif
 
        // This should be filled in by generated code.
        IL2CPP_ASSERT(g_CodegenRegistration != NULL);
        g_CodegenRegistration();
 
        if (!MetadataCache::Initialize())
        {
            s_RuntimeInitCount--;
            return false;
        }
 
        Assembly::Initialize();
        gc::GarbageCollector::Initialize();
 
        // Thread needs GC initialized
        Thread::Initialize();
 
        register_allocator(il2cpp::utils::Memory::Malloc);
 
        memset(&il2cpp_defaults, 0, sizeof(Il2CppDefaults));
 
        const Il2CppAssembly* assembly = Assembly::Load("mscorlib.dll");
        const Il2CppAssembly* assembly2 = Assembly::Load("__Generated");
 
        // 省略部分代码
 
        return true;
    }

```

通过分析我们找到了上面说的三个关键数据结构初始化的地方，这个函数也是整个应用的入口。这里我们只关心 global-metadata 初始化的过程，其它无关代码已经省略。

数据结构还原
======

通过上面的分析我们可以看到解析出 Il2CppCodeRegistration，Il2CppMetadataRegistration 和 Il2CppCodeGenOptions 这三个数据结构是我们解析 global-metadata 的前提，通过分析应用入口源码，我们可以根据特征定位出这段代码在 libil2cpp.so 中的位置，然后就需要写脚本去解析还原数据结构。  
![](https://bbs.pediy.com/upload/tmp/748271_ZDW3EAEVGJRWB6U.png)  
![](https://bbs.pediy.com/upload/tmp/748271_G4RXA9G2MEE7A6N.png)  
上图是还原后在 Ghidra 中的效果，脚本可以在附录中下载。

目标代码定位
======

还原数据结构后我们可以大概定位代码的位置，在 Il2CppCodeRegistration->codeGenModules 这个字段中我们可以看到源码中的不同 module，对应 C# 中的 dll。其中 moduleName 等于 Assembly-CSharp.dll 的 module 就是应用业务逻辑所在的 dll，其它是引擎代码所在的 dll 可以不用关心。  
![](https://bbs.pediy.com/upload/tmp/748271_BJ6UBV6YDA6BWUB.png)  
上面图片就是 Assembly-CSharp.dll 对应的数据结构还原后的效果。其中 methodPointers 就是所有业务逻辑函数的数组，但是这个数组只是一个函数指针的数据，看不到函数名和类型信息。这个时候就需要解析 global-metadata 来还原函数所属的 Class，方法名和签名等类型信息。通过解析 Il2CppGlobalMetadataHeader 我们可以得到所有类型信息的偏移量和数量。

```
typedef struct Il2CppGlobalMetadataHeader
{
    int32_t sanity;
    int32_t version;
    int32_t stringLiteralOffset; // string data for managed code
    int32_t stringLiteralCount;
    int32_t stringLiteralDataOffset;
    int32_t stringLiteralDataCount;
    int32_t stringOffset; // string data for metadata
    int32_t stringCount;
    int32_t eventsOffset; // Il2CppEventDefinition
    int32_t eventsCount;
    int32_t propertiesOffset; // Il2CppPropertyDefinition
    int32_t propertiesCount;
    int32_t methodsOffset; // Il2CppMethodDefinition
    int32_t methodsCount;
    int32_t parameterDefaultValuesOffset; // Il2CppParameterDefaultValue
    int32_t parameterDefaultValuesCount;
    int32_t fieldDefaultValuesOffset; // Il2CppFieldDefaultValue
    int32_t fieldDefaultValuesCount;
    int32_t fieldAndParameterDefaultValueDataOffset; // uint8_t
    int32_t fieldAndParameterDefaultValueDataCount;
    int32_t fieldMarshaledSizesOffset; // Il2CppFieldMarshaledSize
    int32_t fieldMarshaledSizesCount;
    int32_t parametersOffset; // Il2CppParameterDefinition
    int32_t parametersCount;
    int32_t fieldsOffset; // Il2CppFieldDefinition
    int32_t fieldsCount;
    int32_t genericParametersOffset; // Il2CppGenericParameter
    int32_t genericParametersCount;
    int32_t genericParameterConstraintsOffset; // TypeIndex
    int32_t genericParameterConstraintsCount;
    int32_t genericContainersOffset; // Il2CppGenericContainer
    int32_t genericContainersCount;
    int32_t nestedTypesOffset; // TypeDefinitionIndex
    int32_t nestedTypesCount;
    int32_t interfacesOffset; // TypeIndex
    int32_t interfacesCount;
    int32_t vtableMethodsOffset; // EncodedMethodIndex
    int32_t vtableMethodsCount;
    int32_t interfaceOffsetsOffset; // Il2CppInterfaceOffsetPair
    int32_t interfaceOffsetsCount;
    int32_t typeDefinitionsOffset; // Il2CppTypeDefinition
    int32_t typeDefinitionsCount;
    int32_t imagesOffset; // Il2CppImageDefinition
    int32_t imagesCount;
    int32_t assembliesOffset; // Il2CppAssemblyDefinition
    int32_t assembliesCount;
    int32_t fieldRefsOffset; // Il2CppFieldRef
    int32_t fieldRefsCount;
    int32_t referencedAssembliesOffset; // int32_t
    int32_t referencedAssembliesCount;
    int32_t attributeDataOffset;
    int32_t attributeDataCount;
    int32_t attributeDataRangeOffset;
    int32_t attributeDataRangeCount;
    int32_t unresolvedVirtualCallParameterTypesOffset; // TypeIndex
    int32_t unresolvedVirtualCallParameterTypesCount;
    int32_t unresolvedVirtualCallParameterRangesOffset; // Il2CppMetadataRange
    int32_t unresolvedVirtualCallParameterRangesCount;
    int32_t windowsRuntimeTypeNamesOffset; // Il2CppWindowsRuntimeTypeNamePair
    int32_t windowsRuntimeTypeNamesSize;
    int32_t windowsRuntimeStringsOffset; // const char*
    int32_t windowsRuntimeStringsSize;
    int32_t exportedTypeDefinitionsOffset; // TypeDefinitionIndex
    int32_t exportedTypeDefinitionsCount;
} Il2CppGlobalMetadataHeader;

```

根据 imagesOffset 和 imagesCount 可以遍历出所有 dll，再把 dll 信息解析为 Il2CppImageDefinition

```
typedef struct Il2CppImageDefinition
{
    StringIndex nameIndex;
    AssemblyIndex assemblyIndex;
 
    TypeDefinitionIndex typeStart;
    uint32_t typeCount;
 
    TypeDefinitionIndex exportedTypeStart;
    uint32_t exportedTypeCount;
 
    MethodIndex entryPointIndex;
    uint32_t token;
 
    CustomAttributeIndex customAttributeStart;
    uint32_t customAttributeCount;
} Il2CppImageDefinition;

```

根据 nameIndex 字段和 Il2CppGlobalMetadataHeader->stringOffset 可以解析出 dll 名，typeStart 和 typeCount 字段可以遍历出这个 dll 下面所有的 Class 信息。

```
typedef struct Il2CppTypeDefinition
{
    StringIndex nameIndex;
    StringIndex namespaceIndex;
    TypeIndex byvalTypeIndex;
 
    TypeIndex declaringTypeIndex;
    TypeIndex parentIndex;
    TypeIndex elementTypeIndex; // we can probably remove this one. Only used for enums
 
    GenericContainerIndex genericContainerIndex;
 
    uint32_t flags;
 
    FieldIndex fieldStart;
    MethodIndex methodStart;
    EventIndex eventStart;
    PropertyIndex propertyStart;
    NestedTypeIndex nestedTypesStart;
    InterfacesIndex interfacesStart;
    VTableIndex vtableStart;
    InterfacesIndex interfaceOffsetsStart;
 
    uint16_t method_count;
    uint16_t property_count;
    uint16_t field_count;
    uint16_t event_count;
    uint16_t nested_type_count;
    uint16_t vtable_count;
    uint16_t interfaces_count;
    uint16_t interface_offsets_count;
 
    // bitfield to portably encode boolean values as single bits
    // 01 - valuetype;
    // 02 - enumtype;
    // 03 - has_finalize;
    // 04 - has_cctor;
    // 05 - is_blittable;
    // 06 - is_import_or_windows_runtime;
    // 07-10 - One of nine possible PackingSize values (0, 1, 2, 4, 8, 16, 32, 64, or 128)
    // 11 - PackingSize is default
    // 12 - ClassSize is default
    // 13-16 - One of nine possible PackingSize values (0, 1, 2, 4, 8, 16, 32, 64, or 128) - the specified packing size (even for explicit layouts)
    uint32_t bitfield;
    uint32_t token;
} Il2CppTypeDefinition;

```

根据 nameIndex 字段可以解析出 Class 名，fieldStart 和 field_count 可以解析出字段信息，methodStart 和 method_count 可以解析出方法信息，这里我们更关心方法信息。

```
typedef struct Il2CppMethodDefinition
{
    StringIndex nameIndex;
    TypeDefinitionIndex declaringType;
    TypeIndex returnType;
    ParameterIndex parameterStart;
    GenericContainerIndex genericContainerIndex;
    uint32_t token;
    uint16_t flags;
    uint16_t iflags;
    uint16_t slot;
    uint16_t parameterCount;
} Il2CppMethodDefinition;

```

根据这个数据结构可以解析出方法名，签名等信息，但是怎么关联到 libil2cpp.so 代码中的函数呢，通过分析 Il2cpp 源码可以知道是通过解析 token 得出 index 从而关联到 so 中的代码，具体代码如下。

```
static inline uint32_t GetDecodedMethodIndex(EncodedMethodIndex index)
{
    return (index & 0x1FFFFFFEU) >> 1;
}

```

这个函数的参数 EncodedMethodIndex 其实就是 global-metadata 中的 token。解析出的 index 就对应 Il2CppCodeRegistration->codeGenModules[0]->methodPointers 这个函数数组的 index，这样我们就还原出了函数的方法名和类型信息。字段信息的解析也是同理，就不再赘述。把我们的分析结果打印到 Ghidra 中，如下图所示。  
![](https://bbs.pediy.com/upload/tmp/748271_8VAXANMT4AY5A8V.png)  
从图中可以看到 dll 中所有的 Class，Class 的字段和方法还有它们的 token。其中有一个方法名是. ctor 这个是 C# 中 Class 的构造方法，也是我们逆向分析中关键的方法。

目标代码还原
======

还原出函数的类型信息后就可以进一步分析函数的业务逻辑，这时我们发现函数中还有一部分信息是经过编码的，例如 Enter 的. ctor 构造方法。  
![](https://bbs.pediy.com/upload/tmp/748271_Z9J5M28S69JKG3X.png)  
上图是经过 Ghidra 反编译后的代码，我们可以看到在第 9 行，表达式的含义是把 DAT_01273d78 赋值给 param_1 加上一个偏移量，param_1 其实就是 this 指针，偏移量其实就是这个类的一个字段，字段的偏移量也可以从 global-metadata 中解析出来这里就不讲了。关时间是我们查看 DAT_01273d78 这个地址的数据，发现是一个没有意义的 64 位数字，这是因为 Il2cpp 编译成 C++ 代码后会把字面量等类型信息也进行编码存放在 global-metadata 中，通过分析 Il2cpp 源码我们可以知道解析方法为下图。

```
enum Il2CppMetadataUsage
{
    kIl2CppMetadataUsageInvalid,
    kIl2CppMetadataUsageTypeInfo,
    kIl2CppMetadataUsageIl2CppType,
    kIl2CppMetadataUsageMethodDef,
    kIl2CppMetadataUsageFieldInfo,
    kIl2CppMetadataUsageStringLiteral,
    kIl2CppMetadataUsageMethodRef,
};
 
#ifdef __cplusplus
static inline Il2CppMetadataUsage GetEncodedIndexType(EncodedMethodIndex index)
{
    return (Il2CppMetadataUsage)((index & 0xE0000000) >> 29);
}
 
static inline uint32_t GetDecodedMethodIndex(EncodedMethodIndex index)
{
    return (index & 0x1FFFFFFEU) >> 1;
}

```

其中 GetEncodedIndexType 这个函数的作用是解析出编码数据的类型，对应 Il2CppMetadataUsage 这个 enum，GetDecodedMethodIndex 这个函数和之前我们解析方法的 token 是同一个函数，但是这里的作用是解析出编码数据在 global-metadata 中对应数据类型的偏移量。通过这两个函数可以知道 DAT_01273d78 这个编码的类型是 kIl2CppMetadataUsageStringLiteral 也就是字面量，通过 Il2CppGlobalMetadataHeader 可以解析出实际值。其它类型也是同理。到这里我们就可以还原出目标函数的业务逻辑，从而进行分析。通过分析可以得出 DAT_01273d78 就是我们想要找的 key 的值。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#逆向分析](forum-161-1-118.htm) [#源码分析](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)