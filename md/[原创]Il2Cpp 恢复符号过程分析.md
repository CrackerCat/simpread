> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273064.htm)

> [原创]Il2Cpp 恢复符号过程分析

0x1 Il2Cpp 介绍
=============

*   unity 作为两大游戏引擎之一，其安全性也值得被关注。在早期 unity 的脚本都是采用 C# 编写的，直接编译成 C# 模块，所以直接使用 C# 反编译器即可非常完整的获得游戏的源码，此时的 Unity 脚本后处理引擎为 Mono。
*   而由于 C# 的效率问题和安全性问题，Unity 推出了新的脚本后处理引擎 Il2Cpp，该引擎分为两个部分，一个是 AOT 静态编译引擎，一个是 libil2cpp 运行时库。
*   前者通过将 C# IL 编译成 C++ 代码，从而交由不同平台编译器编译，后者则实现了诸如垃圾回收、线程 / 文件获取、内部调用直接修改托管数据结构的原生代码的服务与抽象。  
    ![](https://bbs.pediy.com/upload/tmp/948449_KYS78NS7E4ZFZSB.png)
*   Il2Cpp 编译的游戏，往往有两个重要的文件，一个是 GameAssembly.dll，该文件是由 C++ 代码编译而成的程序，游戏重要的逻辑都在该文件内，其中包含了 il2cpp 的运行时库。另一个文件是 global-metadata.dat，该文件保存了一些重要的字符串信息和一些元数据，用于 il2cpp 的动态特性，例如反射等。
*   这里着重分析如果通过解析 GameAssembly.dll 和 global-metadata.dat 来恢复其中包含的类名，方法名，field 偏移。

0x2 解析 global-metadata.dat
==========================

*   首先得弄明白该文件保存了哪些内容，可以找到 libil2cpp 的源码。直接搜索 global-metadata.dat 即可找到相关代码。该函数位于 vm/GlobalMetadata.cpp 下。  
    ![](https://bbs.pediy.com/upload/tmp/948449_D9AS9WS9U6JPA34.png)
*   可以看到代码直接将文件内容读入后，转化成了一个结构体，Il2CppGlobalMetadataHeader。该结构体定义在了 GlobalMetadataFileInternals.h 里，其中包含了一些重要的信息。  
    ![](https://bbs.pediy.com/upload/tmp/948449_66XV9M2DQFA5WPY.png)

[](#1）获取所有image)1）获取所有 Image
----------------------------

*   首先得知道其中包含的各个 Image 信息。直接定位到结构体中的这两个域，对应着 Il2CppImageDefinition 数组的偏移，该结构保存了 Image 的信息。imagesSize 记录了 Il2CppImageDefinition 数组的大小。  
    ![](https://bbs.pediy.com/upload/tmp/948449_VMHXYB9VJNPDQHE.png)
*   可以通过如下方法获得 Il2CppImageDefinition 数组，并且进行遍历。
    
    ```
    Il2CppGlobalMetadataHeader *header=(Il2CppGlobalMetadataHeader*)ptr;
      if(header->sanity!=0xFAB11BAF || header->stringLiteralOffset!=sizeof(Il2CppGlobalMetadataHeader))
      {
          printf("invalid file..\n");
          return 0;
      }
      int image_count=header->imagesSize/sizeof(Il2CppImageDefinition);
     
      for(int i=0;i
    ```
    
*   接着来查看 Il2CppImageDefinition 结构体，可以发现里面包含了 Image 的名字相关信息 StringIndex nameIndex。  
    ![](https://bbs.pediy.com/upload/tmp/948449_MN9DVRRYB68XCK9.png)
*   StringIndex 是一个整型，是一个索引，global-metadata.dat 中存在一个字符串表，所有 metadata 相关的字符串都放在了一起，通过这个索引进行引用，这个字符串表通过 Il2CppGlobalMetadataHeader 下的偏移 stringOffset 计算得到。
*   可以通过这样的函数获取 StringIndex 对应的字符串，ptr 是 Il2CppGlobalMetadataHeader 的地址。
    
    ```
    static const char* GetStringFromIndex(StringIndex index)
    {
      return (const char*)(((Il2CppGlobalMetadataHeader*)ptr)->stringOffset+ptr+index);
    }
    ```
    

[](#2）获取image下的类)2）获取 Image 下的类
-------------------------------

*   如何获取该 image 所有的类呢，这就需要获取对应的 Il2CppTypeDefinition，在 Il2CppGlobalMetadataHeader 结构体下存在 typeDefinitionsOffset，而由于 Il2CppImageDefinition 存在一个 TypeDefinitionIndex typeStart 的域，所以可以类比 StringIndex。  
    ![](https://bbs.pediy.com/upload/tmp/948449_4HKT7S5NBDK3MPT.png)
*   类比 StringIndex 写出如下代码。
    
    ```
    static const Il2CppTypeDefinition* GetTypeDefinitionFromIndex(TypeDefinitionIndex index)
    {
      return (const Il2CppTypeDefinition*)(ptr+((Il2CppGlobalMetadataHeader*)ptr)->typeDefinitionsOffset)+index;
    }
    ```
    
*   然后便可以获得 Image 下的所有 Il2CppTypeDefinition 了。也就是获取所有类的元数据。
    
    ```
    const Il2CppImageDefinition *image=&image_arr[i];
      printf("image: %s\n",GetStringFromIndex(image->nameIndex));
      for(int j=0;jtypeCount;j++)
      {
          const Il2CppTypeDefinition *type=GetTypeDefinitionFromIndex(image->typeStart+j);
          printf("class: %s:%s\n",GetStringFromIndex(type->namespaceIndex),GetStringFromIndex(type->nameIndex));
      } 
    ```
    
*   这样就能解析出了各种 Image 下的类。  
    ![](https://bbs.pediy.com/upload/tmp/948449_ZNCH43TMAMVGYSQ.png)

[](#3）获取类下的方法名和field)3）获取类下的方法名和 Field
--------------------------------------

*   此时需要我们进一步的分析类下面的方法，此时查看 Il2CppTypeDefinition 结构体可以发现其下有两个域值得注意。  
    ![](https://bbs.pediy.com/upload/tmp/948449_8WY8RBT2PDG685M.png)  
    ![](https://bbs.pediy.com/upload/tmp/948449_3STB5RGV6AJM9U4.png)
*   同样的在 Il2CppGlobalMetadataHeader 由两个域正好对应的上。  
    ![](https://bbs.pediy.com/upload/tmp/948449_FQST9VDW8TBWY4S.png)  
    ![](https://bbs.pediy.com/upload/tmp/948449_ME43QBEJQK47R9A.png)
*   类比上文的方法，可以写出如下代码，来获取类对应的方法和 field 的信息。
    
    ```
    static const Il2CppMethodDefinition* GetMethodDefinitionFromIndex(MethodIndex index)
    {
      return (const Il2CppMethodDefinition*)(((Il2CppGlobalMetadataHeader*)ptr)->methodsOffset+ptr)+index;
    }
    static const Il2CppFieldDefinition* GetFieldDefinitionFromIndex(FieldIndex index)
    {
      return (const Il2CppFieldDefinition*)(ptr+((Il2CppGlobalMetadataHeader*)ptr)->fieldsOffset)+index;
    }
    ```
    
*   同样的，获得的结构体中都保存了名字。  
    ![](https://bbs.pediy.com/upload/tmp/948449_WAH57RGDUF55A8W.png)  
    ![](https://bbs.pediy.com/upload/tmp/948449_W54Z7J8A6D6SHDD.png)
*   效果如下：  
    ![](https://bbs.pediy.com/upload/tmp/948449_UCDB67KTSM99VJM.png)

0x3 恢复方法符号
==========

*   由于此时仅仅获得了方法的名称，还无法和 Gameassembly.dll 中的函数对应起来，得继续查看 il2cpp 的源码。
*   从 il2cpp 的 api 入手，其中有个函数名称为 il2cpp_runtime_invoke.  
    ![](https://bbs.pediy.com/upload/tmp/948449_2DT9BD7F6Y8U9B6.png)
*   点进其中所调用的函数，可以发现有一个 InvokeWithThrow，不难猜到这应该就是调用 method 的函数。  
    ![](https://bbs.pediy.com/upload/tmp/948449_Z5KPCDGBTTYTXRM.png)
*   继续分析 InvokeWithThrow，可以发现里面调用了 method->invoker_method，并且其第一个参数 method->methodPointer 就是方法的指针。  
    ![](https://bbs.pediy.com/upload/tmp/948449_PUMF87254WWCND2.png)
*   继续搜索对 method->methodPointer 的修改，在 Class.cpp 文件中的 Class::SetupMethodsLocked(Il2CppClass *klass, const il2cpp::os::FastAutoLock& lock) 方法下成功找到了赋值语句。该函数的作用即通过 metadata 构造类的所有 MethodInfo，而 MethodInfo 对象则包含了方法函数指针。  
    ![](https://bbs.pediy.com/upload/tmp/948449_7XW5JZQ22KFS9B3.png)
*   可以看到 MetadataCache::GetMethodPointer 通过 image 对象找到了 codeGenModule，再定位到了其下的 methodPointers 数组，再根据 token 取出对应的函数指针。  
    ![](https://bbs.pediy.com/upload/tmp/948449_TSAF62Q4JFEFRK5.png)
*   刚好一个方法的 token 在 Il2CppMethodDefinition 中存在，就差怎么定位 codeGenModule->methodPointers 了。  
    ![](https://bbs.pediy.com/upload/tmp/948449_R3TJA2XMJHCAACX.png)
*   在源代码中搜索 codeGenModule 的类型 Il2CppCodeGenModule。  
    ![](https://bbs.pediy.com/upload/tmp/948449_7EGRNUTYJNZ892J.png)
*   得到了一个结构体 Il2CppCodeRegistration。  
    ![](https://bbs.pediy.com/upload/tmp/948449_Y7DYB2ZA629Z8JU.png)
*   然后再 libil2cpp 中就没法找到相关的定义了，可能是由 il2cpp aot 编译器生成的，然后注入到 il2cpp runtime 里去，所以去看看 GameAssembly.dll 看看能否定位到 Il2CppCodeRegistration 这个结构体。  
    ![](https://bbs.pediy.com/upload/tmp/948449_KFPRQD957UH5CES.png)
*   由于 Il2CppCodeRegistration 中的 codeGenModules 的 Il2CppCodeGenModule 最大的特征就是 moduleName，试试字符串搜索。  
    ![](https://bbs.pediy.com/upload/tmp/948449_XPHD4XHYZDCD7S6.png)
*   合理猜测这个字符串就是 Il2CppCodeGenModule 的 moduleName，交叉引用，猜测这个就是 Il2CppCodeGenModule。  
    ![](https://bbs.pediy.com/upload/tmp/948449_AAJPD3Z94DSBZCT.png)
*   继续交叉引用，不出所料应该就得到了一个数组，这个数组应该就是 Il2CppCodeRegistration->codeGenModules 指向的数组。  
    ![](https://bbs.pediy.com/upload/tmp/948449_QZXYVY7U373MRJ6.png)
*   再次交叉引用找到 Il2CppCodeRegistration 变量。  
    ![](https://bbs.pediy.com/upload/tmp/948449_VJX55TMFCWHUNVD.png)
*   所以只需要定位到 Il2CppCodeRegistration codeRegistration 这个全局变量即可，然后遍历 codeGenModules 找到方法对应的 Image，然后通过方法的 token(Il2CppMethodDefinition 下有) 去 codeGenModules->methodPointers 取出函数指针，即可将方法和函数指针对应上。
*   具体代码如下：
    
    ```
    uint32_t GetMethodPointer(const Il2CppImageDefinition *image,uint32_t token)
    {
      for(int i=0;icodeGenModulesCount;i++)
      {
          const Il2CppCodeGenModule *module=CodeRegistration->codeGenModules[i];
          if(!strcmp(module->moduleName,GetStringFromIndex(image->nameIndex)))
          {
              return module->methodPointers[GetTokenRowId(token)-1];
          }
      }
      printf("invalid!\n");
      return 0;
    } 
    ```
    

0x4 获得 field 的偏移地址
==================

*   同样的从 il2cpp 的 api 出发，可以发现该函数直接调用了 Class::GetFieldFromName，直接去查看该函数。  
    ![](https://bbs.pediy.com/upload/tmp/948449_YYUSA9RTKHW92JV.png)
*   找到 Class::GetFieldFromName，它通过 GetFields 获得所有的 FieldInfo，然后返回对应的 FieldInfo。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_78CWZ5WAP2JBGHY.png)
*   看看 GetFields 函数，其中调用了 SetupFieldsLocked。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_4TC9NWBBYQFXGER.png)
*   继续找到 SetupFieldsLocked，该函数递归初始化了类和父类的所有 FieldInfo，使用 SetupFieldsFromDefinitionLocked 进行初始化。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_T7TWNAZYE2UX3J8.png)
*   找到 SetupFieldsFromDefinitionLocked，在这个函数中就能找到具体的初始化过程了，可以看到其 offset 是通过 MetadataCache::GetFieldOffsetFromIndexLocked 计算而出的。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_95EGET8CPJ3BDYW.png)
*   继续找到 MetadataCache::GetFieldOffsetFromIndexLocked，该函数进一步调用了 GlobalMetadata::GetFieldOffset。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_AY7PV9EBTJA73FY.png)
*   最后一步，GlobalMetadata::GetFieldOffset。该函数通过 typeIndex 和 fieldIndexInType 进行查表。typeIndex 可以通过查当前 type 的 Il2CppTypeDefinition 在 global-metadata.dat 的序号获得，而 fieldIndexInType 在遍历 type 的 field 可以直接获得。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_YNKDWUCAKWCWBJS.png)
*   获取 typeIndex 代码如下
    
    ```
    static TypeDefinitionIndex GetIndexForTypeDefinitionInternal(const Il2CppTypeDefinition* typeDefinition)
    {
      const Il2CppTypeDefinition* typeDefinitions=(const Il2CppTypeDefinition*)(ptr+((Il2CppGlobalMetadataHeader*)ptr)->typeDefinitionsOffset);
      ptrdiff_t index=typeDefinition-typeDefinitions;
      return (TypeDefinitionIndex)index;
    }
    ```
    
*   索引信息都能获取了，现在需要找到的是 s_Il2CppMetadataRegistration->fieldOffsets，也就是要定位到 s_Il2CppMetadataRegistration。该变量的类型为 Il2CppMetadataRegistration  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_6CJTBG7NAZNMXMH.png)
*   该变量存在于 GameAssemly.dll，也是由 il2cpp aot 编译生成注册到 il2cpp runtime 的，而刚好之前发现 codeRegistration 注册时一并注册了 metadataRegistration.  
    ![](https://bbs.pediy.com/upload/tmp/948449_KFPRQD957UH5CES.png)
*   于是我们根据上文找到的 codeRegistration 进行交叉引用，成功定位到 metadataRegistration。  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_7ZCY6B7XGH8DDFQ.png)  
    ![](https://bbs.pediy.com/upload/attach/202205/948449_3U6APXX4QDNWZWC.png)
*   所以只需要根据这个变量直接用 typeIndex 和 fieldIndexInType 查表即可。
    
    ```
    uint32_t GetFieldOffset(TypeDefinitionIndex typeIndex,uint32_t index)
    {
      return MetadataRegistration->fieldOffsets[typeIndex][index];   
    }
    ```
    

0x5 完整代码
========

*   由于使用了大量结构体，不方便直接贴上来，请下载附件，同时还有对应的测试文件和输入。
*   代码不够完善，无法用于生产，仅用于学习。

[【游戏】KCTF2022 春季赛结果竞猜，猜中有奖！](https://ctf.pediy.com/guessgame-pawn.htm)

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm)

上传的附件：

*   [src.zip](javascript:void(0)) （6.80MB，4 次下载）