> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268465.htm)

> [原创]dex 起步探索

参考
==

> 部分内容摘自：android 软件安全权威指南：丰生强
> 
> 从根源上搞懂基础的原理是很有必要的，这样有助于我们更方便的利用它的特性，达到我们的目的。
> 
> 我把内容主要分为二个部分。原理探索、案例分析

原理探索
====

dex 文件
------

我们在正向开发 app 编译时，编写的 java 代码，会编译成 java 字节码保存在. class 后缀的文件中。然后再用 dx 工具将 java 字节码转换成 dex 文件（Dalvik 字节码）。在转换的过程中，会将所有 java 字节码中的所有冗余信息组成一个常量池。例如多个 class 文件中都存在的字符串 "hello world"。转换后将单独存放在一个地方，并且所有类共享。包括方法的签名也会组成常量池。我们将编译好的 apk 文件解压后就能拿到 classes.dex 文件。

dex 文件格式
--------

### [](#1、dexfile结构)1、DexFile 结构

上面拿到的 classes.dex 文件包含了 apk 的可执行代码。Dalvik 虚拟机会解析加载文件并执行代码。只要我们了解这个文件格式的组成，那么就可以自己解析这个文件获取到想要的数据。

 

首先是安卓源码中的`dalvik/libdex/DexFile.h`这里可以找到 dex 文件的数据结构。下面贴上源码部分

```
struct DexFile {
    /* odex的头 */
    const DexOptHeader* pOptHeader;
 
    /* dex文件头，指定了dex文件的一些数据，记录了其他数据结构在dex文件中的物理偏移 */
    const DexHeader*    pHeader;
      /* 索引结构区 */
    const DexStringId*  pStringIds;
    const DexTypeId*    pTypeIds;
    const DexFieldId*   pFieldIds;
    const DexMethodId*  pMethodIds;
    const DexProtoId*   pProtoIds;
      /* 真实的数据存放 */
    const DexClassDef*  pClassDefs;
      /* 静态链接数据区 */
    const DexLink*      pLinkData;
    /*
     * These are mapped out of the "auxillary" section, and may not be
     * included in the file.
     */
    const DexClassLookup* pClassLookup;
    const void*         pRegisterMapPool;       // RegisterMapClassPool
 
    /* points to start of DEX file data */
    const u1*           baseAddr;
 
    /* track memory overhead for auxillary structures */
    int                 overhead;
 
    /* additional app-specific data structures associated with the DEX */
    //void*               auxData;
};

```

为了更好的理解。dex 文件格式，我们可以用 010 编辑器打开一个 dex 文件对照这个结构体来观察一下。

 

![](https://bbs.pediy.com/upload/attach/202107/659397_G65Y5GU8TT5VNSA.png)

 

可以看到，这个 dex 文件由这 8 个部分组成。

 

dex_header：dex 文件头，指定了 dex 文件的一些数据，记录了其他数据结构在 dex 文件中的物理偏移

 

string_ids：字符串列表（前面说的去掉冗余信息组成的常量池，全局共享使用的）

 

type_ids：类型签名列表（去掉冗余信息组成的常量池）

 

proto_ids：方法声明列表（去掉冗余信息组成的常量池）

 

field_ids：字段列表（去掉冗余信息组成的常量池）

 

method_ids：方法列表（去掉冗余信息组成的常量池）

 

class_def：类型结构体列表（去掉冗余信息组成的常量池）

 

map_list：这里记录了前面 7 个部分的偏移和大小。

 

然后我们开始逐个的看各个部分的结构。

### [](#2、dex_header)2、dex_header

先是贴上源码看看这个部分的结构体

```
struct DexHeader {
    u1  magic[8];           /* 表示是一个有效的dex文件。值一般固定为64 65 78 0A 30 33 35 00（dex.035） */
    u4  checksum;           /* adler32 checksum dex文件的校验和，用来判断文件是否已经损坏或者篡改 */
    u1  signature[kSHA1DigestLen]; /* SHA-1 hash 用来识别未经dexopt优化的dex文件*/
    u4  fileSize;           /* length of entire file 记录了包括dexHeader在内的整个dex文件的大小*/
    u4  headerSize;         /* offset to start of next section  dexHeader占用的字节数，一般都是0x70*/
    u4  endianTag;                    /* 指定dex运行环境的cpu字节序。预设是ENDIAN_CONSTANT等于0x12345678，也就是默认小端字节序 */
    u4  linkSize;                        /* 链接段的大小 */
    u4  linkOff;                        /* 链接段的偏移 */
    u4  mapOff;                            /* DexMapList结构的文件偏移 */
    u4  stringIdsSize;            /* 下面都是数据段的大小和文件偏移 */
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize;
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};

```

这里可以看到，如果在 DexHeader 中，可以找到其他部分的偏移和大小，以及整个文件的大小，解析出这块数据，其他部分的任意数据，我们都可以获取到。然后再使用对应的结构体来解析。另外留意这里 DexHeader 的结构体的大小是固定 0x70 字节的。所以有的脱壳工具中会将`70 00 00 00`来作为特征在内存中查找 dex 进行脱壳（比如 FRIDA-DEXDump 的深度检索）

 

然后我们贴一下真实 classes.dex 文件的 DexHeader 数据是什么样的。

 

![](https://bbs.pediy.com/upload/attach/202107/659397_YC4P7G9YUMVTSFT.png)

### [](#3、string_ids)3、string_ids

先看看字符串列表的结构体，非常简单，就是字符串的偏移，但是并不是普通的 ascii 字符串，而是 MUTF-8 编码的。这个是一个经过修改的 UTF-8 编码。和传统的 UTF-8 相似

```
struct DexStringId {
    u4 stringDataOff;      /* 字符串数据偏移 */
};

```

![](https://bbs.pediy.com/upload/attach/202107/659397_2E8Q4X5FNQW6GFB.png)

### [](#4、type_ids)4、type_ids

类型签名列表的结构体也是非常简单，和上面字符串列表差不多

```
struct DexTypeId {
    u4  descriptorIdx;      /* index into stringIds list for type descriptor */
};

```

真实数据图如下，可以看到值类型签名都在前面，后面都是引用类型签名。

 

![](https://bbs.pediy.com/upload/attach/202107/659397_J2B45HPECJKZ7JF.png)

### [](#5、proto_ids)5、proto_ids

方法声明的列表的结构体较为复杂，因为方法签名必然是有几点信息构成：返回值类型、参数类型列表（就是每个参数是什么类型）。方法声明的结构体如下

```
struct DexTypeList {
    u4  size;               /* dexTypeItem的个数 */
    DexTypeItem list[1];    /* entries */
};
struct DexTypeItem {
    u2  typeIdx;            /* DexTypeId的索引 */
};
struct DexProtoId {
    u4  shortyIdx;          /* DexStringId列表的索引，方法签名字符串，由返回值和参数类型列表组合*/
    u4  returnTypeIdx;      /* DexTypeId的索引，返回值的类型 */
    u4  parametersOff;      /* 指向DexTypeList的偏移，参数类型列表 */
};

```

同样看看这个结构的真实数据

 

![](https://bbs.pediy.com/upload/attach/202107/659397_WE77Z3Z9KC2NZMZ.png)

### [](#6、field_ids)6、field_ids

字段描述的结构体，我们可以先想象一下，要找一个字段，我们需要些什么：字段所属的类，字段的类型，字段名称。有这些信息，就可以找到各自对应的字段了。接下来看看定义的结构体

```
struct DexFieldId {
    u2  classIdx;           /* 类的类型，指向DexTypeId的索引，字段所属的类 */
    u2  typeIdx;            /* 字段类型，指向DexTypeId的索引，字段的类型 */
    u4  nameIdx;            /* 字段名，指向DexStringId的索引，字段的名称 */
};

```

然后看一段真实数据

 

![](https://bbs.pediy.com/upload/attach/202107/659397_UWCDGUGDSA4BGJE.png)

### [](#7、method_ids)7、method_ids

方法描述的结构体，同样先了解找一个方法的几个必须项：方法所属的类，方法的签名（签名中有方法的返回值和方法的参数，也就是上面的 proto_ids 中记录的），方法的名称。然后下面看结构体

```
struct DexMethodId {
    u2  classIdx;           /* 类的类型，指向DexTypeId的索引，方法所属的类 */
    u2  protoIdx;           /* 声明类型，指向DexProtoId的索引，方法的签名 */
    u4  nameIdx;            /* 方法名，指向DexStringId索引，方法的名称 */
};

```

看看一组方法的真实数据

 

![](https://bbs.pediy.com/upload/attach/202107/659397_G8F75VSAAD57D37.png)

### [](#8、class_def)8、class_def

类定义的结构体，这个比较复杂。直接贴上结构体和原文的说明。这里大致可以看出来，和上面的原理差不多，通过这个结构体来描述类的内容。

```
struct DexClassDef {
    u4  classIdx;           /* 类的类型，指向DexTypeId的索引 */
    u4  accessFlags;                /* 访问标志 */
    u4  superclassIdx;      /* 父类的类型，指向DexTypeId的索引 */
    u4  interfacesOff;      /* 接口，指向DexTypeList的偏移，如果没有接口的声明和实现，值为0 */
    u4  sourceFileIdx;      /* 类所在的源文件名，指向DexStringId的索引 */
    u4  annotationsOff;     /* 注释，根据类型不同会有注解类，注解字段，注解方法，注解参数，没有注解值就是0，指向DexAnnotationsDirectoryItem的结构体 */
    u4  classDataOff;       /* 类的数据部分，指向DexClassData结构的偏移 */
    u4  staticValuesOff;    /* 类中的静态数据，指向DexEncodeArray结构的偏移 */
};

```

下面同样展示一组真实数据

 

![](https://bbs.pediy.com/upload/attach/202107/659397_4WCGMCDP36MJVUJ.png)

 

上面数据看到里面的 class_data 也是一个结构体，然后继续看这个类数据的结构体

```
/* expanded form of a class_data_item header */
struct DexClassDataHeader {
    u4 staticFieldsSize;                /* 静态字段的个数 */
    u4 instanceFieldsSize;            /* 实例字段的个数 */
    u4 directMethodsSize;                /* 直接方法的个数 */
    u4 virtualMethodsSize;            /* 虚方法的个数 */
};
 
/* expanded form of encoded_field */
struct DexField {
    u4 fieldIdx;    /* 指向DexFieldId的索引 */
    u4 accessFlags;    /* 访问标志 */
};
 
/* expanded form of encoded_method */
struct DexMethod {
    u4 methodIdx;    /* 指向DexMethodId的索引 */
    u4 accessFlags;     /* 访问标志 */
    u4 codeOff;      /* 指向DexCode结构的偏移 */
};
 
struct DexClassData {
    DexClassDataHeader header;                            /* 指定字段和方法的个数 */
    DexField*          staticFields;                /* 静态字段 */
    DexField*          instanceFields;            /* 实例字段 */
    DexMethod*         directMethods;                /* 直接方法 */
    DexMethod*         virtualMethods;            /* 虚方法 */
};

```

到这里我们基本看到了在开发中，一个类的所有特征。完整的描述出了一个类的所有信息。

 

9、DexCode 上面最后看到方法的代码是通过上面的 DexCode 结构体来找到的。最后看下这个结构体

```
struct DexCode {
    u2  registersSize;            /* 使用寄存器的个数 */
    u2  insSize;                        /* 参数的个数 */
    u2  outsSize;                        /* 调用其他方法时使用的寄存器个数 */
    u2  triesSize;                    /* try/catch语句的个数 */
    u4  debugInfoOff;       /* 指向调试信息的偏移 */
    u4  insnsSize;          /* 指令集的个数，以2字节为单位 */
    u2  insns[1];                        /* 指令集 */
    /* 2字节空间用于对齐 */
    /* followed by try_item[triesSize] DexTry结构体 */
    /* followed by uleb128 handlersSize */
    /* followed by catch_handler_item[handlersSize] DexCatchHandler结构体 */
};

```

到了这里，存储的就是执行的指令集了。通过执行指令来跑这个方法。下面看一组真实数据

 

![](https://bbs.pediy.com/upload/attach/202107/659397_G3YWFCVZENQYRR3.png)

 

这里看到这个类的具体描述字段和函数，还有访问标志等等信息。然后我们继续看里面函数的执行代码部分。看下面一组数据

 

观察的函数是 MainActivity 类的 DoTcp 函数。DexCode（也就是 code_item，也叫 codeOff）的偏移地址是 0x127ec。

 

下面观察到 DexCode 结构体的偏移到指令集 insns 字段的偏移是 codeOff+2+2+2+2+4+4=0x127fc（这里 + 的 2 和 4 是看下面结构体 insns 前面的字段占了多少个字节计算的，可以当做固定 + 16 个字节）。指令集的长度是 0x93。

 

![](https://bbs.pediy.com/upload/attach/202107/659397_XWE73KTS9SYJ9DZ.png)

 

最后看看指令集的开始数据是 0x62、0xe26、0x11a、0x232。但是我们要注意前面有说明，这里是两字节空间对齐。所以，这里的值我们应该前面填充一下。

 

前面四个字节我们要看做 0x0062、0x0e26、0x011a、0x0232。但是我们还要注意，还有个端序问题会影响字节的顺序，这里是小端序，所以我们再调整下

 

前面四个字节我们要看做 0x6200、0x260e、0x1a01、0x3202。把这段指令集的数据看明白后，我们用 gda 打开这个 dex 文件。然后找到对应的方法，查看一下。

 

然后发现数据对上了。这里存储的果然就是我们 dex 分析方法的字节码了。

 

![](https://bbs.pediy.com/upload/attach/202107/659397_U9DWUGH4RFJTS99.png)

### [](#9、map_list)9、map_list

在书中的意思是，Dalvik 虚拟机解析 Dex 后，将其映射成 DexMapList 的数据结构，然后在里面可以找到前面 8 个部分的偏移和大小。先看看结构体

```
struct DexMapItem {
    u2 type;              /* kDexType开头的类型 */
    u2 unused;                        /* 未使用，用于字节对齐 */
    u4 size;              /* 数据的大小 */
    u4 offset;            /* 指定类型数据的文件偏移 */
};
/*
 * Direct-mapped "map_list".
 */
struct DexMapList {
    u4  size;               /* 有多少个DexMapItem */
    DexMapItem list[1];     /* entries */
};
 
enum {
    kDexTypeHeaderItem               = 0x0000,
    kDexTypeStringIdItem             = 0x0001,
    kDexTypeTypeIdItem               = 0x0002,
    kDexTypeProtoIdItem              = 0x0003,
    kDexTypeFieldIdItem              = 0x0004,
    kDexTypeMethodIdItem             = 0x0005,
    kDexTypeClassDefItem             = 0x0006,
    kDexTypeCallSiteIdItem           = 0x0007,
    kDexTypeMethodHandleItem         = 0x0008,
    kDexTypeMapList                  = 0x1000,
    kDexTypeTypeList                 = 0x1001,
    kDexTypeAnnotationSetRefList     = 0x1002,
    kDexTypeAnnotationSetItem        = 0x1003,
    kDexTypeClassDataItem            = 0x2000,
    kDexTypeCodeItem                 = 0x2001,
    kDexTypeStringDataItem           = 0x2002,
    kDexTypeDebugInfoItem            = 0x2003,
    kDexTypeAnnotationItem           = 0x2004,
    kDexTypeEncodedArrayItem         = 0x2005,
    kDexTypeAnnotationsDirectoryItem = 0x2006,
};

```

每个 DexMapItem 对应了一块数据，例如 type=kDexTypeHeaderItem 则对应 DexHeader 的偏移地址和大小。下面看真实数据

 

![](https://bbs.pediy.com/upload/attach/202107/659397_JDC8RNJMFE7XDJK.png)

 

这里就能看到 string_ids 的偏移和大小。如此，根据这个 map_list 就能找到所有块的数据了。

 

那么在源码中是如何使用这个数据的呢，我好奇的翻了一下。然后再 dex 文件优化的流程中 dexSwapAndVerify 函数找到了使用的地方。

```
//字节排序优化
int dexSwapAndVerify(u1* addr, size_t len)
{
    ...
    if (okay) {
        /*
         * Look for the map. Swap it and then use it to find and swap
         * everything else.
         */
        if (pHeader->mapOff != 0) {
            DexFile dexFile;
            DexMapList* pDexMap = (DexMapList*) (addr + pHeader->mapOff);
 
            okay = okay && swapMap(&state, pDexMap);
            okay = okay && swapEverythingButHeaderAndMap(&state, pDexMap);
 
            dexFileSetupBasicPointers(&dexFile, addr);
            state.pDexFile = &dexFile;
 
            okay = okay && crossVerifyEverything(&state, pDexMap);
        } else {
            ALOGE("ERROR: No map found; impossible to byte-swap and verify");
            okay = false;
        }
    }
        ...
    return !okay;       // 0 == success
}

```

这个函数中获取了 DexMapList。然后交给 swapMap 函数来处理。

```
static bool swapMap(CheckState* state, DexMapList* pMap)
{
    DexMapItem* item = pMap->list;
    u4 count;
    u4 dataItemCount = 0; // Total count of items in the data section.
    u4 dataItemsLeft = state->pHeader->dataSize; // See use below.
    u4 usedBits = 0;      // Bit set: one bit per section
    bool first = true;
    u4 lastOffset = 0;
 
    SWAP_FIELD4(pMap->size);
    count = pMap->size;
    const u4 sizeOfItem = (u4) sizeof(DexMapItem);
    CHECK_LIST_SIZE(item, count, sizeOfItem);
 
    while (count--) {
        SWAP_FIELD2(item->type);
        SWAP_FIELD2(item->unused);
        SWAP_FIELD4(item->size);
        SWAP_OFFSET4(item->offset);
 
        if (first) {
            first = false;
        } else if (lastOffset >= item->offset) {
            ALOGE("Out-of-order map item: %#x then %#x",
                    lastOffset, item->offset);
            return false;
        }
 
        if (item->offset >= state->pHeader->fileSize) {
            ALOGE("Map item after end of file: %x, size %#x",
                    item->offset, state->pHeader->fileSize);
            return false;
        }
 
        if (isDataSectionType(item->type)) {
            u4 icount = item->size;
 
            /*
             * This sanity check on the data section items ensures that
             * there are no more items than the number of bytes in
             * the data section.
             */
            if (icount > dataItemsLeft) {
                ALOGE("Unrealistically many items in the data section: "
                        "at least %d", dataItemCount + icount);
                return false;
            }
 
            dataItemsLeft -= icount;
            dataItemCount += icount;
        }
 
        u4 bit = mapTypeToBitMask(item->type);
 
        if (bit == 0) {
            return false;
        }
 
        if ((usedBits & bit) != 0) {
            ALOGE("Duplicate map section of type %#x", item->type);
            return false;
        }
 
        if (item->type == kDexTypeCallSiteIdItem) {
            state->pCallSiteIds = item;
        } else if (item->type == kDexTypeMethodHandleItem) {
            state->pMethodHandleItems = item;
        }
 
        usedBits |= bit;
        lastOffset = item->offset;
        item++;
    }
 
    if ((usedBits & mapTypeToBitMask(kDexTypeHeaderItem)) == 0) {
        ALOGE("Map is missing header entry");
        return false;
    }
 
    if ((usedBits & mapTypeToBitMask(kDexTypeMapList)) == 0) {
        ALOGE("Map is missing map_list entry");
        return false;
    }
 
    if (((usedBits & mapTypeToBitMask(kDexTypeStringIdItem)) == 0)
            && ((state->pHeader->stringIdsOff != 0)
                    || (state->pHeader->stringIdsSize != 0))) {
        ALOGE("Map is missing string_ids entry");
        return false;
    }
 
    if (((usedBits & mapTypeToBitMask(kDexTypeTypeIdItem)) == 0)
            && ((state->pHeader->typeIdsOff != 0)
                    || (state->pHeader->typeIdsSize != 0))) {
        ALOGE("Map is missing type_ids entry");
        return false;
    }
 
    if (((usedBits & mapTypeToBitMask(kDexTypeProtoIdItem)) == 0)
            && ((state->pHeader->protoIdsOff != 0)
                    || (state->pHeader->protoIdsSize != 0))) {
        ALOGE("Map is missing proto_ids entry");
        return false;
    }
 
    if (((usedBits & mapTypeToBitMask(kDexTypeFieldIdItem)) == 0)
            && ((state->pHeader->fieldIdsOff != 0)
                    || (state->pHeader->fieldIdsSize != 0))) {
        ALOGE("Map is missing field_ids entry");
        return false;
    }
 
    if (((usedBits & mapTypeToBitMask(kDexTypeMethodIdItem)) == 0)
            && ((state->pHeader->methodIdsOff != 0)
                    || (state->pHeader->methodIdsSize != 0))) {
        ALOGE("Map is missing method_ids entry");
        return false;
    }
 
    if (((usedBits & mapTypeToBitMask(kDexTypeClassDefItem)) == 0)
            && ((state->pHeader->classDefsOff != 0)
                    || (state->pHeader->classDefsSize != 0))) {
        ALOGE("Map is missing class_defs entry");
        return false;
    }
 
    state->pDataMap = dexDataMapAlloc(dataItemCount);
    if (state->pDataMap == NULL) {
        ALOGE("Unable to allocate data map (size %#x)", dataItemCount);
        return false;
    }
 
    return true;
}

```

通过这里的例子。我们可以看到他是如何使用这个 map_list 来访问所有的部分的。到这里 dex 文件的格式基本差不多了。

 

然后我们看看一个实战的项目。大佬写的 fart 中的 py 部分就的运用了 dex 文件格式相关的知识。

案例分析
====

[](#案例一：fart)案例一：fart
---------------------

### [](#1、github)1、github

> [FART](https://github.com/hanbinglengyue/FART.git)

### [](#2、功能说明)2、功能说明

这是一个脱壳工具，使用主动调用的方式来解决二代抽取壳。脱出来的数据不止是 dex。还有一种. bin 的数据，这种数据可以用来辅助我们修复 dex 的一些没有脱出来的函数。我们先看下. bin 数据是什么，下面是. bin 中的一组数据

```
{name:ooxx,method_idx:1830,offset:180516,code_item_len:24,ins:AQABAAEAAAB+oAMABAAAAHAQwAsAAA4A};

```

name：是随意填的，因为并不是使用 name 来找对应函数，而是通过 method_idx

 

method_idx：函数的索引。

 

offset：函数的偏移

 

code_item_len：code_item 的大小

 

ins：code_item 结构体这段数据的 base64 编码

 

另外贴一下这几个数据的 dump 来源的代码，以防有人把 ins 当成指令集了。

```
var base64ptr = funcBase64_encode(ptr(codeitemstartaddr), codeitemlength, ptr(base64lengthptr));
var b64content = ptr(base64ptr).readCString(base64lengthptr.readInt());
funcFreeptr(ptr(base64ptr));
var content = "{name:ooxx,method_idx:" + dex_method_index_ + ",offset:" + dex_code_item_offset_ + ",code_item_len:" + codeitemlength + ",ins:" + b64content + "};";

```

这几个数据是一个函数最关键的片段。拿到就可以还原出最关键的 code_item 了。

### [](#3、使用)3、使用

然后看看 fart.py 的使用`./fart.py -d 431528_29868.dex -i 431528_29868.bin`。下面是执行结果，基本对这个 dex 进行完整的解析，打印出来大多数的数据。

 

![](https://bbs.pediy.com/upload/attach/202107/659397_7QVQEGXUWGQ785N.png)

### [](#4、源码分析)4、源码分析

我们可以通过对这个项目的阅读，直观的了解到是如何进行 dex 文件格式进行解析的。下面开始看看具体的实现流程

```
def main():
    #加载dex文件
    dex = dex_parser(filename)
if __name__ == "__main__":
    #获取到参数filename和insfilename
    init()
    methodTable.clear()
    #加载.bin文件的内容，也就是insfilename设置的文件，给methodTable填充上值。每项的数据是：方法名称，方法id，方法偏移，方法大小，指令集
    parseinsfile()
    print "methodTable length:" + str(len(methodTable))
    #开始处理
    main()

```

然后看看是如何加载. bin 文件的

```
def parseinsfile():
    global insfilename
    insfile=open(insfilename)
    content=insfile.read()
    insfile.close()
    #;{name:artMethod::dumpmethod DexFile_dumpDexFile'
    # dexfile name:classes.dex--
    # insfilepath:/data/data/com.wlqq/10668484_ins.bin--
    # code_item_len:40,
    # code_item_len:40,
    # ins:AgABAAIAAABLnY4ADAAAACIAFwNwEPoOAABuIP4OEAAMAR8BFwMRAQ==};
    insarray=re.findall(r"{name:(.*?),method_idx:(.*?),offset:(.*?),code_item_len:(.*?),ins:(.*?)}",content) #(.*?)最短匹配
    #按照我们前面看到的格式进行匹配数据并遍历
    for eachins in insarray:
      #这里其实是固定的ooxx
      methodname=eachins[0].replace(" ","")
      number=(int)(eachins[1])
      offset=(int)(eachins[2])
      inssize=int(eachins[3])
      ins=eachins[4]
      tempmethod=CodeItem(number,methodname,inssize,ins)
      methodTable[number]=tempmethod #添加method

```

可以看到就是遍历，组装好那些数据，最后保存到一个 methodTable 里面，索引就是 method_id。

 

再看看最关键的 dex_parse

```
class dex_parser:
 
    def __init__(self,filename):
        #dex的标志
        global DEX_MAGIC
        #odex的标志
        global DEX_OPT_MAGIC
        self.m_javaobject_id = 0
        #dex的文件路径
        self.m_filename = filename
        self.m_fd = open(filename,"rb")
        self.m_content = self.m_fd.read()
        self.m_fd.close()
        self.m_dex_optheader = None
        self.m_class_name_id = {}
        self.string_table = []
        #如果发现是odex文件，则填充opt_header，否则只填充dex_header
        if self.m_content[0:4] == DEX_OPT_MAGIC:
            self.init_optheader(self.m_content)
            self.init_header(self.m_content,0x40)
        elif self.m_content[0:4] == DEX_MAGIC:
            self.init_header(self.m_content,0)
        #上面填充dex_header的时候取到的string_ids的偏移位置和大小
        bOffset = self.m_stringIdsOff
        if self.m_stringIdsSize > 0:
            #遍历字符串列表
            for i in xrange(0,self.m_stringIdsSize):
                #这里是取出每个字符串的偏移地址
                offset, = struct.unpack_from("I",self.m_content,bOffset + i * 4)
                #如果是第一个则直接存放到start，然后处理下一次
                if i == 0:
                    start = offset
                else:
                    #取出上一个偏移的字符串的偏移地址，这里由于存储格式是uleb128的。要转换成真正的偏移地址。
                    skip, length = get_uleb128(self.m_content[start:start+5])
                    #上面还原出了真实的偏移，这里取字符串的地址，保存起来
                    self.string_table.append(self.m_content[start+skip:offset-1])
                    start = offset
            #处理最后的一条
            for i in xrange(start,len(self.m_content)):
                if self.m_content[i]==chr(0):
                    self.string_table.append(self.m_content[start+1:i])
                    break
        #遍历classDef。填充m_class_name_id
        for i in xrange(0,self.m_classDefSize):
            str1 = self.getclassname(i)
            self.m_class_name_id[str1] = i
        #遍历classDef，填充classdef相关的属性，并且打印，合并.bin的内容是在print中进行的。
        for i in xrange(0,self.m_classDefSize):
            str1 = self.getclassname(i)
            dex_class(self,i).printf(self)
            pass

```

这里看到是通过 init_header 来解析 dex 文件中的 dexHeader 的。我就不贴代码了，整体就是偏移然后取数据。

 

这里有一点要说的是 get_uleb128 函数，uleb128 在 android 里面是一个特殊的类型。是一个可变长度的类型。大致意思就是例如第一个字节的最高位，如果是 1，则第二个字节也是有效数据，如果第二个字节的最高位也是 1，下一个字节也是有效数据，如果最高位不是 1，就结束了。最后左移拼接就 ok 了。

 

这个类型有没有很眼熟？没错，特别像是 protobuf 里面的 varint 编码。所以我觉得完全可以用 protobuf 包自带的 varint 解码来获取这个数据。也可以用这种自己写的函数来处理。

```
def varint_encode(number):
    buf = b''
    while True:
        towrite = number & 0x7f
        number >>= 7
        if number:
            buf += struct.pack("B",(towrite | 0x80))
        else:
            buf +=  struct.pack("B",towrite)
            break
    return buf
def varint_decode(buff):
    shift = 0
    result = 0
    idx=0
    while True:
        if idx>len(buff):
            return ""
        i = buff[idx]
        idx+=1
        result |= (i & 0x7f) << shift
        shift += 7
        if not (i & 0x80):
            break
    return result

```

继续看后面的关键函数 dex_class，这里来处理每一个类的打印

 

先看看初始化函数，基本和之前的 initHeader 的差不多，就是偏移取数据，然后保存下来。把 classDef 相关的数据都读取出来了。

```
class dex_class:
    def __init__(self,dex_object,classid):
        if classid >= dex_object.m_classDefSize:
            return ""
        offset = dex_object.m_classDefOffset + classid * struct.calcsize("8I")
        self.offset = offset
        format = "I"
        self.thisClass,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.modifiers,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.superClass,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.interfacesOff,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.sourceFileIdx,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.annotationsOff,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.classDataOff,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.staticValuesOff,=struct.unpack_from(format,dex_object.m_content,offset)
        offset += struct.calcsize(format)
        self.index = classid
        self.interfacesSize = 0
        if self.interfacesOff != 0:
            self.interfacesSize, = struct.unpack_from("I",dex_object.m_content,self.interfacesOff)
        if self.classDataOff != 0:
            offset = self.classDataOff
            count,self.numStaticFields = get_uleb128(dex_object.m_content[offset:])
            offset += count
            count,self.numInstanceFields = get_uleb128(dex_object.m_content[offset:])
            offset += count
            count,self.numDirectMethods = get_uleb128(dex_object.m_content[offset:])
            offset += count
            count,self.numVirtualMethods = get_uleb128(dex_object.m_content[offset:])   
        else:
            self.numStaticFields = 0
            self.numInstanceFields = 0
            self.numDirectMethods = 0
            self.numVirtualMethods = 0

```

最后就是 print 函数，这里比较大，就只挑最关键的部分出来说下

```
def printf(self,dex_object):
    ...
    print "=========numDirectMethods[%d]=numVirtualMethods[%d]=numStaticMethods[0]========="%(self.numDirectMethods,self.numVirtualMethods)
    method_idx = 0
# 遍历实例函数
    for i in xrange(0,self.numDirectMethods):   
  #获取到method_id
        n,method_idx_diff = get_uleb128(dex_object.m_content[offset:offset+5])
        offset += n
        n,access_flags = get_uleb128(dex_object.m_content[offset:offset+5])
        offset += n
        n,code_off = get_uleb128(dex_object.m_content[offset:offset+5])
        offset += n
  #这里看到method_idx实际上是累加的。我们可以找例子去看下。例如第一个函数的idx是0xd2。第二个的idx是1，实际上是0xd3。也就是上一个的值+1
        method_idx += method_idx_diff
        if code_off != 0:
            method)
            method=None
            try:
      #这里获取了之前的.bin文件中对应的数据
                method = methodTable[method_idx]
            except Exception as e:
                pass
            if method != None:
      #如果bin文件中有这个函数，则先打印下dex中的对应函数
                print "\nDirectMethod:" + dex_object.getmethodfullname(method_idx, True) + "\n"
                try:
                    print "before repire method+++++++++++++++++++++++++++++++++++\n"
                    method_code(dex_object, code_off).printf(dex_object, "\t\t")
                except Exception as e:
                    print e
      #然后再把bin文件中的给转换成method_code，然后打印
                try:
                    bytearray_str = base64.b64decode(method.insarray)
                    print "after repire method++++++++++++++++++++++++++++++++++++\n"
                    repired_method_code(dex_object, bytearray_str).printf(dex_object, "\t\t")
                except Exception as e:
                    print e
    ...

```

最后我们看下怎么获取的 bin 文件数据的函数

```
class repired_method_code:
    dex_obj=None
    content = ""
    trylist = []
    def __init__(self, dex_obj,content):
        offset=0
        format = "H"
        self.dex_obj=dex_obj
        self.content=content
    # 这段数据我们之前看到就是存的DexCode的结构体，所以直接按格式进行读取数据出来
        self.registers_size, = struct.unpack_from(format, content, offset)
        offset += struct.calcsize(format)
        self.ins_size, = struct.unpack_from(format, content, offset)
        offset += struct.calcsize(format)
        self.outs_size, = struct.unpack_from(format, content, offset)
        offset += struct.calcsize(format)
        self.tries_size, = struct.unpack_from(format, content, offset)
        offset += struct.calcsize(format)
        format = "I"
        self.debug_info_off, = struct.unpack_from(format, content, offset)
        offset += struct.calcsize(format)
        self.insns_size, = struct.unpack_from(format, content, offset)
        offset += struct.calcsize(format)
        self.insns = offset
        offset += 2 * self.insns_size
        if self.insns_size % 2 == 1:
            offset += 2
        if self.tries_size == 0:
            self.tries = 0
            self.handlers = 0
        else:
            self.handlerlist_offset = offset + 8 * self.tries_size
            self.tries = offset
            for i in range(0, self.tries_size):
                temptryitem = tryitem(self.dex_obj,content, self.handlerlist_offset, offset + 8 * i)
                self.trylist.append(temptryitem)
            self.handlers = offset + self.tries_size * struct.calcsize("I2H")  #

```

最后还有个打印指令集的部分就不贴了。感兴趣的可以自己看一看

[](#案例二：dex2jar)案例二：dex2jar
---------------------------

### [](#1、github：)1、github：

> [dex2jar](https://github.com/pxb1988/dex2jar.git)

### [](#2、功能说明)2、功能说明

这个工具基本大家都用过。功能非常强大，不过这里只分析下 d2j_dex2jar 功能。也就是把 dex 给转换成 jar 文件。

### [](#3、使用)3、使用

下载 release 版本直接`./d2j-dex2jar.sh classes.dex`，就生成出了对应的 classes-dex2jar.jar 文件。然后我们直接用其他分析工具就能愉快的看 java 代码了。那么神奇的事情是如何做到的呢。下面跟踪分析一下大佬的作品。

### [](#4、源码分析)4、源码分析

第一步是找到入口，根据我们上面的使用例子，先搜索下 d2j-dex2jar。然后找到了下面的文件

 

`./dex2jar/dex-tools/src/main/java/com/googlecode/dex2jar/tools/Dex2jarCmd.java`

 

先简单的看一下这里的代码，看来这里就是入口函数了。

```
@BaseCmd.Syntax(cmd = "d2j-dex2jar", syntax = "[options] [file1 ... fileN]", desc = "convert dex to jar")
public class Dex2jarCmd extends BaseCmd {
 
    public static void main(String... args) {
        new Dex2jarCmd().doMain(args);
    }
    ...
} 
```

然后看看 doMain 的实现

```
public void doMain(String... args) {
        ...
        doCommandLine();
        ...
    }

```

这里调用了抽象方法，doCommandLine，所以继续看看实现

```
protected void doCommandLine() throws Exception {
        ...
        for (String fileName : remainingArgs) {
            // long baseTS = System.currentTimeMillis();
            String baseName = getBaseName(new File(fileName).toPath());
              //这里看到默认使用的jar文件名
            Path file = output == null ? currentDir.resolve(baseName + "-dex2jar.jar") : output;
            System.err.println("dex2jar " + fileName + " -> " + file);
                        //BaseDexFileReader这个类型就是dex的所有解析处理
            BaseDexFileReader reader = MultiDexFileReader.open(Files.readAllBytes(new File(fileName).toPath()));
              //用来处理异常信息的
            BaksmaliBaseDexExceptionHandler handler = notHandleException ? null : new BaksmaliBaseDexExceptionHandler();
              //这句就是最关键的转换部分，前面都是设置转换的一些参数，最后的to用来执行dex转换jar
            Dex2jar.from(reader).withExceptionHandler(handler).reUseReg(reuseReg).topoLogicalSort()
                    .skipDebug(!debugInfo).optimizeSynchronized(this.optmizeSynchronized).printIR(printIR)
                    .noCode(noCode).skipExceptions(skipExceptions).to(file);
                        //有异常的话，就通过异常处理handler保存信息
            if (!notHandleException) {
                if (handler.hasException()) {
                    Path errorFile = exceptionFile == null ? currentDir.resolve(baseName + "-error.zip")
                            : exceptionFile;
                    System.err.println("Detail Error Information in File " + errorFile);
                    System.err.println(BaksmaliBaseDexExceptionHandler.REPORT_MESSAGE);
                    handler.dump(errorFile, orginalArgs);
                }
            }
            // long endTS = System.currentTimeMillis();
            // System.err.println(String.format("%.2f", (float) (endTS - baseTS) / 1000));
        }
    }

```

到这里就可以看出来两个最重要的部分了

 

1、解析 dex 的 MultiDexFileReader.open 方法

 

2、执行转换的 Dex2jar 的 to 方法

 

下面先看看解析的处理

```
public static BaseDexFileReader open(byte[] data) throws IOException {
              //dex文件太小就直接抛出异常
        if (data.length < 3) {
            throw new IOException("File too small to be a dex/zip");
        }
        if ("dex".equals(new String(data, 0, 3, StandardCharsets.ISO_8859_1))) {// dex
            return new DexFileReader(data);
        } else if ("PK".equals(new String(data, 0, 2, StandardCharsets.ISO_8859_1))) {// ZIP
              //可以看到如果是zip文件，它会帮我们在里面找classes开头并且.dex结尾的文件来处理，也就是说直接参数使用apk也是可以的。
            TreeMap dexFileReaders = new TreeMap<>();
            try (ZipFile zipFile = new ZipFile(data)) {
                for (ZipEntry e : zipFile.entries()) {
                    String entryName = e.getName();
                    if (entryName.startsWith("classes") && entryName.endsWith(".dex")) {
                        if (!dexFileReaders.containsKey(entryName)) { // only the first one
                            dexFileReaders.put(entryName, new DexFileReader(toByteArray(zipFile.getInputStream(e))));
                        }
                    }
                }
            }
              //下面是单dex文件和多dex文件的处理
            if (dexFileReaders.size() == 0) {
                throw new IOException("Can not find classes.dex in zip file");
            } else if (dexFileReaders.size() == 1) {
                return dexFileReaders.firstEntry().getValue();
            } else {
                return new MultiDexFileReader(dexFileReaders.values());
            }
        }
        throw new IOException("the src file not a .dex or zip file");
    } 
```

我们直接看参数为 dex 文件的情况就好了，所以继续看 DexFileReader 的构造函数

```
//直接调用了另一个重载
public DexFileReader(byte[] data) {
        this(ByteBuffer.wrap(data));
    }
//解析dex的关键位置
public DexFileReader(ByteBuffer in) {
        in.position(0);
        in = in.asReadOnlyBuffer().order(ByteOrder.BIG_ENDIAN);
        int magic = in.getInt() & 0xFFFFFF00;
 
        final int MAGIC_DEX = 0x6465780A & 0xFFFFFF00;// hex for 'dex ', ignore the 0A
        final int MAGIC_ODEX = 0x6465790A & 0xFFFFFF00;// hex for 'dey ', ignore the 0A
                //这里区分一下是dex还是odex。odex情况就直接抛出异常了
        if (magic == MAGIC_DEX) {
            ;
        } else if (magic == MAGIC_ODEX) {
            throw new DexException("Not support odex");
        } else {
            throw new DexException("not support magic.");
        }
              //下面是取出dexHeader相关的一系列数据了
        int version = in.getInt() >> 8;
        if (version < 0 || version < DEX_035) {
            throw new DexException("not support version.");
        }
        this.dex_version = version;
        in.order(ByteOrder.LITTLE_ENDIAN);
 
        // skip uint checksum
        // and 20 bytes signature
        // and uint file_size
        // and uint header_size 0x70
              // 意思是偏移跳过上面的那些数据，跳过是根据上面的字段占的空间来直接偏移即可
        skip(in, 4 + 20 + 4 + 4);
                //这个是cpu字节序，非小端序就直接抛出异常了。
        int endian_tag = in.getInt();
        if (endian_tag != ENDIAN_CONSTANT) {
            throw new DexException("not support endian_tag");
        }
 
        // skip uint link_size
        // and uint link_off
              //再跳过上面的两个字段
        skip(in, 4 + 4);
                //把下面那些重要部分的偏移全部取出来保存。
        int map_off = in.getInt();
        string_ids_size = in.getInt();
        int string_ids_off = in.getInt();
        type_ids_size = in.getInt();
        int type_ids_off = in.getInt();
        proto_ids_size = in.getInt();
        int proto_ids_off = in.getInt();
        field_ids_size = in.getInt();
        int field_ids_off = in.getInt();
        method_ids_size = in.getInt();
        int method_ids_off = in.getInt();
        class_defs_size = in.getInt();
        int class_defs_off = in.getInt();
        // skip uint data_size data_off
 
        int call_site_ids_off = 0;
        int call_site_ids_size = 0;
        int method_handle_ids_off = 0;
        int method_handle_ids_size = 0;
              //如果是高版本的相关处理（大于DEX_037的版本我好像没见过。）
        if (dex_version > DEX_037) {
            in.position(map_off);
            int size = in.getInt();
            for (int i = 0; i < size; i++) {
                int type = in.getShort() & 0xFFFF;
                in.getShort(); // unused;
                int item_size = in.getInt();
                int item_offset = in.getInt();
                switch (type) {
                case TYPE_CALL_SITE_ID_ITEM:
                    call_site_ids_off = item_offset;
                    call_site_ids_size = item_size;
                    break;
                case TYPE_METHOD_HANDLE_ITEM:
                    method_handle_ids_off = item_offset;
                    method_handle_ids_size = item_size;
                    break;
                default:
                    break;
                }
            }
        }
              //看这个意思是只有DEX_037以上的dex才有这个值，低版本默认0即可。
        this.call_site_ids_size = call_site_ids_size;
        this.method_handle_ids_size = method_handle_ids_size;
                //直接从内存中把这些重要数据的块给切片出来单独存放，后面使用就不需要全部通过偏移来找了。
              //这里的长度计算，是根据各个结构体的大小*列表长度计算。
        stringIdIn = slice(in, string_ids_off, string_ids_size * 4);
        typeIdIn = slice(in, type_ids_off, type_ids_size * 4);
        protoIdIn = slice(in, proto_ids_off, proto_ids_size * 12);
        fieldIdIn = slice(in, field_ids_off, field_ids_size * 8);
        methoIdIn = slice(in, method_ids_off, method_ids_size * 8);
        classDefIn = slice(in, class_defs_off, class_defs_size * 32);
              //下面这两个不用在意把，DEX_037以上的版本才有
        callSiteIdIn = slice(in, call_site_ids_off, call_site_ids_size * 4);
        methodHandleIdIn = slice(in, method_handle_ids_off, method_handle_ids_size * 8);
                //初始化下面这些数据，并且设置好字节序，这里还没设置偏移来着。感觉应该用一个变量就行了。
        in.position(0);
        annotationsDirectoryItemIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        annotationSetItemIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        annotationItemIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        annotationSetRefListIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        classDataIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        codeItemIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        stringDataIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        encodedArrayItemIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        typeListIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
        debugInfoIn = in.duplicate().order(ByteOrder.LITTLE_ENDIAN);
    }

```

上面看了 dexHeader 的解析的核心部分，接下来我们看看转换实现 to 方法

```
public void to(Path file) throws IOException {
        if (Files.exists(file) && Files.isDirectory(file)) {
            doTranslate(file);
        } else {
            try (FileSystem fs = createZip(file)) {
                doTranslate(fs.getPath("/"));
            }
        }
    }

```

继续看看 doTranslate 方法

```
private void doTranslate(final Path dist) throws IOException {
                ...
        DexFileNode fileNode = new DexFileNode();
        try {
              //这里的reader就是我们前面看到的那个解析dexHeader的处理，这里就是完整解析dex转换成fileNode
            reader.accept(fileNode, readerConfig | DexFileReader.IGNORE_READ_EXCEPTION);
        } catch (Exception ex) {
            exceptionHandler.handleFileException(ex);
        }
        ...
                //.convertDex(fileNode, cvf);        最后调用的这个函数是把前面转换好的fileNode给转换成.class文件。最后打包成jar
    }

```

我们主要观察的是 dex 的结构和解析，所以就不详细看转换部分了，继续看解析部分的后续 accept 方法

```
//这里遍历所有class_def，然后调用另一个重载
public void accept(DexFileVisitor dv, int config) {
        dv.visitDexFileVersion(this.dex_version);
        for (int cid = 0; cid < class_defs_size; cid++) {
            accept(dv, cid, config);
        }
        dv.visitEnd();
    }
 
public void accept(DexFileVisitor dv, int classIdx, int config) {
              //根据classdef索引找到偏移位置
        classDefIn.position(classIdx * 32);
              //解析出classdef的各项字段
        int class_idx = classDefIn.getInt();
        int access_flags = classDefIn.getInt();
        int superclass_idx = classDefIn.getInt();
        int interfaces_off = classDefIn.getInt();
        int source_file_idx = classDefIn.getInt();
        int annotations_off = classDefIn.getInt();
        int class_data_off = classDefIn.getInt();
        int static_values_off = classDefIn.getInt();
 
        String className = getType(class_idx);
              //这里看到可以设置忽略的classname，他这里是空的，固定返回false
        if(ignoreClass(className)) return;
        String superClassName = getType(superclass_idx);
        String[] interfaceNames = getTypeList(interfaces_off);
        try {
              //visit这个是一个保存的功能，把值保存在了那个fileNode的里面。
            DexClassVisitor dcv = dv.visit(access_flags, className, superClassName, interfaceNames);
            if (dcv != null)// 不处理
            {
                  //拿到了classdef的相关数据的偏移位置，再根据这些数据对类进行详细解析
                acceptClass(dcv, source_file_idx, annotations_off, class_data_off, static_values_off, config);
                dcv.visitEnd();
            }
        } catch (Exception ex) {
            DexException dexException = new DexException(ex, "Error process class: [%d]%s", class_idx, className);
            if (0 != (config & IGNORE_READ_EXCEPTION)) {
                niceExceptionMessage(dexException, 0);
            } else {
                throw dexException;
            }
        }
    }

```

这里最关键的就是 acceptClass 来解析具体的类的内容

```
private void acceptClass(DexClassVisitor dcv, int source_file_idx, int annotations_off, int class_data_off,
            int static_values_off, int config) {
        if ((config & SKIP_DEBUG) == 0) {
            // 获取源文件
            if (source_file_idx != -1) {
                dcv.visitSource(this.getString(source_file_idx));
            }
        }
                //字段的注释
        Map fieldAnnotationPositions;
              //方法的注释
        Map methodAnnotationPositions;
              //参数的注释
        Map paramAnnotationPositions;
        if ((config & SKIP_ANNOTATION) == 0) {
            // 获取注解
            fieldAnnotationPositions = new HashMap();
            methodAnnotationPositions = new HashMap();
            paramAnnotationPositions = new HashMap();
              // 如果有注释的偏移，下面则解析出注释的数据，保存到上面的对应map中
            if (annotations_off != 0) { // annotations_directory_item
 
                annotationsDirectoryItemIn.position(annotations_off);
 
                int class_annotations_off = annotationsDirectoryItemIn.getInt();
                int field_annotation_size = annotationsDirectoryItemIn.getInt();
                int method_annotation_size = annotationsDirectoryItemIn.getInt();
                int parameter_annotation_size = annotationsDirectoryItemIn.getInt();
 
                for (int i = 0; i < field_annotation_size; i++) {
                    int field_idx = annotationsDirectoryItemIn.getInt();
                    int field_annotations_offset = annotationsDirectoryItemIn.getInt();
                    fieldAnnotationPositions.put(field_idx, field_annotations_offset);
                }
                for (int i = 0; i < method_annotation_size; i++) {
                    int method_idx = annotationsDirectoryItemIn.getInt();
                    int method_annotation_offset = annotationsDirectoryItemIn.getInt();
                    methodAnnotationPositions.put(method_idx, method_annotation_offset);
                }
                for (int i = 0; i < parameter_annotation_size; i++) {
                    int method_idx = annotationsDirectoryItemIn.getInt();
                    int parameter_annotation_offset = annotationsDirectoryItemIn.getInt();
                    paramAnnotationPositions.put(method_idx, parameter_annotation_offset);
                }
                                // 如果有对类的注释偏移
                if (class_annotations_off != 0) {
                    try {
                        read_annotation_set_item(class_annotations_off, dcv);
                    } catch (Exception e) {
                        throw new DexException("error on reading Annotation of class ", e);
                    }
                }
            }
        } else {
            fieldAnnotationPositions = null;
            methodAnnotationPositions = null;
            paramAnnotationPositions = null;
        }
                //类详细数据的解析
        if (class_data_off != 0) {
            ByteBuffer in = classDataIn;
            in.position(class_data_off);
                        //静态字段
            int static_fields = (int) readULeb128i(in);
              //实例字段
            int instance_fields = (int) readULeb128i(in);
              //实例函数
            int direct_methods = (int) readULeb128i(in);
              //虚函数
            int virtual_methods = (int) readULeb128i(in);
            {
                int lastIndex = 0;
                {
                    Object[] constant = null;
                    if ((config & SKIP_FIELD_CONSTANT) == 0) {
                        if (static_values_off != 0) {
                            constant = read_encoded_array_item(static_values_off);
                        }
                    }
                    for (int i = 0; i < static_fields; i++) {
                        Object value = null;
                        if (constant != null && i < constant.length) {
                            value = constant[i];
                        }
                          // 解析并填充字段
                        lastIndex = acceptField(in, lastIndex, dcv, fieldAnnotationPositions, value, config);
                    }
                }
                lastIndex = 0;
                for (int i = 0; i < instance_fields; i++) {
                      // 解析并填充字段
                    lastIndex = acceptField(in, lastIndex, dcv, fieldAnnotationPositions, null, config);
                }
                lastIndex = 0;
                boolean firstMethod = true;
                for (int i = 0; i < direct_methods; i++) {
                      // 解析并填充方法
                    lastIndex = acceptMethod(in, lastIndex, dcv, methodAnnotationPositions, paramAnnotationPositions,
                            config, firstMethod);
                    firstMethod = false;
                }
                lastIndex = 0;
                firstMethod = true;
                for (int i = 0; i < virtual_methods; i++) {
                      // 解析并填充方法
                    lastIndex = acceptMethod(in, lastIndex, dcv, methodAnnotationPositions, paramAnnotationPositions,
                            config, firstMethod);
                    firstMethod = false;
                }
            }
 
        }
    } 
```

对类数据进行详细解析后，接着是对字段和方法的解析填充。先看看字段的解析处理

```
//字段数据的解析获取
private Field getField(int id) {
        fieldIdIn.position(id * 8);
        int owner_idx = 0xFFFF & fieldIdIn.getShort();
        int type_idx = 0xFFFF & fieldIdIn.getShort();
        int name_idx = fieldIdIn.getInt();
        return new Field(getType(owner_idx), getString(name_idx), getType(type_idx));
    }
 
private int acceptField(ByteBuffer in, int lastIndex, DexClassVisitor dcv,
            Map fieldAnnotationPositions, Object value, int config) {
        int diff = (int) readULeb128i(in);
        int field_access_flags = (int) readULeb128i(in);
        int field_id = lastIndex + diff;
              // 取出字段的数据
        Field field = getField(field_id);
              // 下面是直接填充字段内容
        // //////////////////////////////////////////////////////////////
        DexFieldVisitor dfv = dcv.visitField(field_access_flags, field, value);
        if (dfv != null) {
            if ((config & SKIP_ANNOTATION) == 0) {
                  //字段注释的相关处理
                Integer annotation_offset = fieldAnnotationPositions.get(field_id);
                if (annotation_offset != null) {
                    try {
                        read_annotation_set_item(annotation_offset, dfv);
                    } catch (Exception e) {
                        throw new DexException(e, "while accept annotation in field:%s.", field.toString());
                    }
                }
            }
            dfv.visitEnd();
        }
        // //////////////////////////////////////////////////////////////
        return field_id;
    } 
```

字段填充完毕，然后看看方法是怎么解析填充的。

```
//方法数据的解析获取
private Method getMethod(int id) {
        methoIdIn.position(id * 8);
        int owner_idx = 0xFFFF & methoIdIn.getShort();
        int proto_idx = 0xFFFF & methoIdIn.getShort();
        int name_idx = methoIdIn.getInt();
        return new Method(getType(owner_idx), getString(name_idx), getProto(proto_idx));
    }
 
private int acceptMethod(ByteBuffer in, int lastIndex, DexClassVisitor cv, Map methodAnnos,
            Map parameterAnnos, int config, boolean firstMethod) {
        int offset = in.position();
        int diff = (int) readULeb128i(in);
        int method_access_flags = (int) readULeb128i(in);
        int code_off = (int) readULeb128i(in);
        int method_id = lastIndex + diff;
        Method method = getMethod(method_id);
                ...
        try {
              //填充方法数据
            DexMethodVisitor dmv = cv.visitMethod(method_access_flags, method);
            if (dmv != null) {
                if ((config & SKIP_ANNOTATION) == 0) {
                      //处理方法的注释
                    Integer annotation_offset = methodAnnos.get(method_id);
                    if (annotation_offset != null) {
                        try {
                            read_annotation_set_item(annotation_offset, dmv);
                        } catch (Exception e) {
                            throw new DexException(e, "while accept annotation in method:%s.", method.toString());
                        }
                    }
                      //处理参数的注释
                    Integer parameter_annotation_offset = parameterAnnos.get(method_id);
                    if (parameter_annotation_offset != null) {
                        try {
                            read_annotation_set_ref_list(parameter_annotation_offset, dmv);
                        } catch (Exception e) {
                            throw new DexException(e, "while accept parameter annotation in method:%s.",
                                    method.toString());
                        }
                    }
                }
                  //如果有code_item。还继续进行指令的解析
                if (code_off != 0) {
                    boolean keep = true;
                    if (0 != (SKIP_CODE & config)) {
                        keep = 0 != (KEEP_CLINIT & config) && method.getName().equals("");
                    }
                    if(keep) {
                        DexCodeVisitor dcv = dmv.visitCode();
                        if (dcv != null) {
                            try {
                                  //解析并填充code_item的数据
                                acceptCode(code_off, dcv, config, (method_access_flags & DexConstants.ACC_STATIC) != 0,
                                        method);
                            } catch (Exception e) {
                                throw new DexException(e, "while accept code in method:[%s] @%08x", method.toString(),
                                        code_off);
                            }
                        }
                    }
                }
                dmv.visitEnd();
            }
        } catch (Exception e) {
            throw new DexException(e, "while accept method:[%s]", method.toString());
        }
 
        return method_id;
    } 
```

继续看 code_item 的解析

```
/* package */void acceptCode(int code_off, DexCodeVisitor dcv, int config, boolean isStatic, Method method) {
        ByteBuffer in = codeItemIn;
        in.position(code_off);
              //取出code_item的字段数据
        int registers_size = 0xFFFF & in.getShort();
        in.getShort();// ins_size ushort
        in.getShort();// outs_size ushort
        int tries_size = 0xFFFF & in.getShort();
        int debug_info_off = in.getInt();
              //指令长度
        int insns = in.getInt();
                //这里是指令集，好像是因为指令是2个字节为一个单位的。所以要*2
        byte[] insnsArray = new byte[insns * 2];
        in.get(insnsArray);
        dcv.visitRegister(registers_size);
        BitSet nextInsn = new BitSet();
        Map labelsMap = new TreeMap();
        Set handlers = new HashSet();
        // 处理异常处理
        if (tries_size > 0) {
            if ((insns & 0x01) != 0) {// skip padding
                in.getShort();
            }
            if (0 == (config & SKIP_EXCEPTION)) {
                findTryCatch(in, dcv, tries_size, insns, labelsMap, handlers);
            }
        }
        // 处理debug信息
        if (debug_info_off != 0 && (0 == (config & SKIP_DEBUG))) {
            DexDebugVisitor ddv = dcv.visitDebug();
            if (ddv != null) {
                read_debug_info(debug_info_off, registers_size, isStatic, method, labelsMap, ddv);
                ddv.visitEnd();
            }
        }
 
        BitSet badOps = new BitSet();
        findLabels(insnsArray, nextInsn, badOps, labelsMap, handlers, method);
              //解析并填充指令集
        acceptInsn(insnsArray, dcv, nextInsn, badOps, labelsMap);
        dcv.visitEnd();
    } 
```

最后的 acceptInsn 指令集的解析方法太大了，我就不放上来了。整个解析填充的流程就完成了。后面就使用解析好的数据进行转换的操作。

 

可以看到两个案例的解析的方式差不多，总体都是根据结构体的大小偏移来取得想要的数据。最后一层一层的处理。

 

为了方便使用和测试，我将 fart 的解析整理了一下，改到了 python3 运行的。然后整合到了我的整合怪里面。感兴趣的可以看看。刚跑通，不知道有没啥问题，后面我再慢慢修复把。

 

github：[fridaUiTools](https://github.com/dqzg12300/fridaUiTools)

 

贴上效果图

 

![](https://bbs.pediy.com/upload/attach/202107/659397_AW33NVJU8XKCSTN.png)

疑问
--

fart 的修复方案仅仅是打印出了保存的指令数据。如果我们是想要直接转成. class 文件或者是 jar 文件。是否可行呢。

 

我设想了两种方案来做，但是最后都被自己给否定了。

 

1、bin 文件中的指令数据直接插入到 dex 文件的对应数据中，最后保存为一个新的文件。但是这样中间插一段数据，会有大量的偏移数据要修改。

 

2、和 dex2jar 的模式一样。构造好一个 fileNode 对象，里面就不存在偏移的问题了。最后把里面的类数据导出成. class 文件。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#基础理论](forum-161-1-117.htm)