> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278264.htm)

> [原创] UnrealEngine4 恢复符号过程分析

[原创] UnrealEngine4 恢复符号过程分析

2 小时前 226

### [原创] UnrealEngine4 恢复符号过程分析

* * *

之前看到过很多游戏都是用虚幻引擎写的，就萌生了研究它的兴趣，但是发现网上关于如何 Dump 出 SDK 没有很好的资料，正好研究具体原理并整理一下。本文基于 UnrealEngine4.27 版本，对于 4.25 之前的版本需要针对 UProperty 和 FProperty 的变动部分进行修改。

*   虚幻引擎是目前主流两大引擎之一，该引擎采用 C++ 实现，相比于 Unity3d，适用于开发大型 3A 游戏。在逆向方面，该引擎由于使用了 C++ 来实现，因此本身是不会存在虚拟机的概念的，这点与 Unity 不同，因此理论上在逆向方面比 Unity 更加困难。但是虚幻引擎为了实现对象的管理，垃圾收集系统以及一些动态特性，采用了一种编译时期的反射系统。  
    ![](https://bbs.kanxue.com/upload/tmp/948449_M9JXY6GAEEJE4SB.png)
*   该反射机制在编译过程中发挥作用，预处理源码中需要参与反射机制的类，并为其生成反射数据，其中包括了类名，类的方法，类中的属性等相关信息，并将这些信息包装在数据结构中保存下来，这给了我们可乘之机，能够借助这些信息恢复出原有的结构体和类以及函数名等符号信息。

*   在本节中，将给出虚幻引擎反射机制中比较重要的集中数据类型 / 类。虚幻引擎中所有对象的类都是 UObject 的派生类，其中为了描述反射获得的对象，虚幻引擎采用 UField 来实现 (类似于 Java，同样继承自 UObject)。UField 派生出了一系列的类用于描述数据结构。具体如下所示：
    
    > **U/FProperty**: 表示 C++ 中的属性，即类或结构体的成员变量。  
    > **UEnum**: 表示 C++ 中的枚举，内部保存了一个 TMap，维护了 Name、Value、Index 三大信息的对应关系。  
    > **UStruct**: 表示 C++ 中的复杂类型，包含函数、类、结构体三种。内部维护了所表示类型的所有 UProperty。  
    > **UFunction**: 表示 C++ 中的函数，内部维护了函数指针、栈帧、参数返回值信息，还提供了反射执行所表示函数的方法。  
    > **UClass**：表示 C++ 中的类，在 UStruct 的基础上扩展了 UFunction 的保存与查找方法。  
    > **UScriptStruct**: 表示 C++ 中的结构体，只是在 UStruct 的基础上增加了一些工具方法而已。
    
*   其中 UStruct 并不代表 C++ 中的结构体，而是用于描述能够承载 UProperty 的数据结构，例如 UClass，UScriptStruct 中的属性以及 UFunction 中的函数参数。具体的继承图如下图所示：  
    ![](https://bbs.kanxue.com/upload/tmp/948449_RX49DBQM5CNJTS2.png)
*   值得注意的是，上图描述的是虚幻引擎 4.25 之后版本的继承关系，4.25 之前则是使用 UProperty 来描述属性，而 UProperty 继承自 UField，这样带来了大量的花销，4.25 之后的版本进行了效率优化，使用了 UField 的子集 FField 来描述 Property，且不用继承自 UObject，这样大幅度减少了属性对象的占用，在下文中将使用 FProperty 来指代 UProperty，两者意思实际上是一样的。

*   可以发现即使能够直接通过 UObject::FName Name 来获取类 / 函数 / 属性的名称，解析 FName 同样不是一件简单的事情。因为 FName 实际并未存储实际的字符串，而是存储了字符串的引用。在虚幻引擎中，FName 为静态字符串类型，类似于存储与. rodata 段的 const char * 字符串。在 FName 中实际上存储的是一个 Index，即 FNameEntryId。  
    ![](https://bbs.kanxue.com/upload/tmp/948449_UVP26ZY2Y5JWCTF.png)
    
*   通过 FNameEntryId 即可从 FNamePool GName 中查找到实际的字符串值。GName 的具体逆向方法这里不做详细介绍，这里介绍如何通过 GName 和 FNameEntryId 来访问具体的字符串。查看 FNamePool 的定义，可以发现其中即 FNameEntryAllocator，而其中存在这一个指针数组，数组中存储着指向 FNameEntry 的指针，名为 Blocks。  
    ![](https://bbs.kanxue.com/upload/tmp/948449_RG2YSH3PXMMGVXM.png)
    
*   查看 FNameEntry 可以发现其中存储了字符串本体，因此我们根据 FNamePool 和 FName 中的 FNameEntryId 查表即可获取到字符串本体。  
    ![](https://bbs.kanxue.com/upload/tmp/948449_SWFG7B824ZVHPA5.png)
    
*   查表方式借鉴虚幻引擎 FNameEntryAllocator 下的 Resolve 方法，FNameEntryHandle 其实是通过移位分裂 FNameEntryId 的方式获得的，代码如下。  
    ![](https://bbs.kanxue.com/upload/tmp/948449_JDFKP6YT5UUA75A.png)  
    ![](https://bbs.kanxue.com/upload/tmp/948449_F2D9YAUBS92GAH6.png)
    
*   至此便可以通过 FNamePool GName 和对应的 FName 获得对象的名称了，具体的实现代码请查看源码 Dump.cpp 下的 GetNameByIndex 函数。  
    ![](https://bbs.kanxue.com/upload/tmp/948449_Q4AXNKN2R7D79Y6.png)
    

在了解了解析 UClass 等反射数据的方法和通过 FName 获取具体字符串的操作后，现在只需要获取到相应的类对象即可进行 dump 了。

虽然上文将我们感兴趣的 UObject 分为了三类，但实际上需要处理的只有两类：UEnum 和 UStruct，因为 UClass 和 UScriptStruct 皆继承自 UStruct，且仅通过 UStruct 结构就可提取出类的属性和方法等所有生成 Sdk 的信息。

*   到这里已经能够获取所有我们关心的对象并进行解析了，具体的 SDK 逻辑也就是一些字符串拼接的操作，可以直接查看源码，这里给出效果图。  
    ![](https://bbs.kanxue.com/upload/attach/202308/948449_49GBE5S5K4W5T88.png)

[议题征集启动！看雪 · 第七届安全开发者峰会 10 月 23 日上海](https://bbs.kanxue.com/thread-276136.htm)

返回