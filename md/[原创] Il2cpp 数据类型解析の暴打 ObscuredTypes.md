> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267375.htm)

前言
==

摸鱼了好几个月, 期间是有写过文章的不过咕掉了, 搓文章太累了  
[![](https://mihoyo.top/wp-content/uploads/20210419092040.jpg)](https://mihoyo.top/wp-content/uploads/20210419092040.jpg "呜呜呜")

 

4 月初我 hxd 介绍了款手游给我 <坎 ****>, 说来让我碰碰  
我一看! 嗷! **ObscuredFloat** 啊, 游戏概述, 没意思, 骡的岛用的就是这个, 太弱鸡了不搞  
然后我就试着玩了一下, 意外的有点上头, 然后就研究了一下伤害的解析  
刚一碰, 我大 E 了啊, 没有闪, 一套不定长内存把我打麻瓜了  
我说你这小伙子不讲武德, 来骗, 来偷袭, 我这单身的几十年的老同志

环境 & 工具
=======

Android 10 aarch64  
GDTS 国际服: `2.13.1`  
IDA Pro: `7.2`  
Dobby: [gayhub](https://github.com/jmpews/Dobby)

康康 DamageInfo 类型
================

[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_10-05-01.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_10-05-01.png "DamageInfo")

 

映入眼帘的就是 `_damage` 这个 **成员变量**  
不过类型有点点奇怪, 是 `Nullable<T>` 类型  
再看看偏移, 就更奇怪了 `convertAttackTo` 这个成员变量的大小跟 `_damage` 似乎不一样

```
成员变量自身大小 = 下一个成员变量的偏移 - 自身偏移
Nullablesize: 0x34 - 0x18 = 0x1C
Nullablesize: 0xE4 - 0xDC = 0x8 
```

ん, 确实不一样, 开始头大了

细品 Nullable<> 类型
================

[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_11-55-21.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_11-55-21.png "Nullable")  
好家伙, 就一个 `has_value` 成员变量是个 `bool` 类型以外, 其他信息全部木大  
至于为何特化出来的类型大小不一, 因为成员变量 `value` 的类型为特化类型, 主要还是要看具体特化类型的内存占用大小

 

这时就引出了一个问题, `T value` 为何不应该以指针存储呢?  
或者说, 所有对象都继承于 `Il2cppObject` 却不以指针方式存储值而导致长度不定?

Struct & Class 区别
=================

老司机可能会发现这几个类型的定义并不是传统的 `class` 而是 `struct`, 并且首个成员变量的偏移量并不是以 `0x10` 开始 (32 位为 `0x8`, 其实就是`Il2cppObject` 内部的两个指针), so, `struct` 的成员变量, 多半没法直接去调用 api 获取对应的东西  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_16-42-13.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_16-42-13.png "Il2cppObject")

 

还有一个区别就是, `class` 的成员变量都是以二级指针形式存储, 指向的是另一块内存, 获取时必须 `*` 取值一下才能得到目标 `Object`  
而 `struct` 是直接占用结构体内部, 偏移即可获得目标 `Object`

 

这种特性也解释了为何 `struct` 的成员变量的长度是不同的  

康康 ObscuredFloat 类型
===================

[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_17-32-15.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_17-32-15.png "ObscuredFloat")  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_17-36-00.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-19_17-36-00.png "ACTkByte4")  
也是 `struct` 的, 成员变量并没有很有用的信息  
需要注意的是 C# 里 `byte` 类型对应 C++ 内是 `char` (8 字节长度), 并且 `ACTkByte4` 也是 `struct` 结构体, so `hiddenValueOldByte4` 这个成员变量可以看做 `int32_t` 类型

内存分析 & 猜想验证
===========

怎么 Hook 我就不贴出来了, 只贴关键代码

```
#include //内存读取
template inline R MemoryRead(void* addr, ulong off)
{
    return *reinterpret_cast((static_cast(addr) + off));
}
 
//xx伤害 函数定义
void (* _xxxDamages)(Il2CppObject*, void*) = nullptr;
void xxxDamages(Il2CppObject* behaviour, void* damageInfo)
{
    for (auto i = 0; i < 0x50; i += sizeof(void*))
        LOGE("0x%x:\t0x%08" PRIX32 " - 0x%08" PRIX32 " - 0x%016" PRIX64, i,
        MemoryRead(damageInfo, i),
        MemoryRead(damageInfo, i + 0x4),
        MemoryRead(damageInfo, i));
    return _xxxDamages(behaviour, damageInfo);
} 
```

随便找个陷阱撞一下  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-26_09-34-34.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-26_09-34-34.png "未处理的内存打印")

 

有点丑, 先按照 `类型占用大小`去划分一下内存对照吧  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-27_09-29-10.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-27_09-29-10.png "内存对照图")  

内存对照说明
------

### 0x0 - 0x8

可以看得出并不是指向某处的指针, 所以验证了 `struct` 类型并没有继承 `Il2cppObject` 的成员变量

 

其值是 `0x80` , 瞄了一下下面的 `0x8-0x10`, 确认该数据类型是 `8字节` 长度  
对照一下结构体的定义, 第一个成员变量 `DamageType type`, 而 `0x80(128)` 跟 `DamageType::Trap` 对应的上  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-27_09-46-11.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-04-27_09-46-11.png "DamageType")

### 0x8 - 0x10

很显然是一枚指针, 在 **64 位**下指针的长度是 `8字节`  
对应于结构体中第二个成员变量 `IFieldObject sender`

### 0x10 - 0x18

一枚指针  
对应于结构体中第三个成员变量 `IFieldObject target`

### 0x18 - 0x34

对应于结构体中第四个成员变量 `Nullable<ObscuredFloat> _modifer`  
由于值为空, 没啥好说明的, 直接跳过

### 0x34 - 0x50

对应于结构体中第五个成员变量 `Nullable<ObscuredFloat> _damage`

 

由于 `Nullable<T>` 的第一个成员变量是 `T value`, 所以特化后**展开**成员变量样子是这样的

```
struct Nullable {
    int currentCryptoKey;
    int hiddenValue;
    ACTkByte4 hiddenValueOldByte4; // 可以当做int32_t
    bool inited;
    float fakeValue;
    bool fakeValueActive;
    bool has_value;
}; 
```

C# 中 `int` 类型占用 4 字节  
0x34 - 0x38 对应 `int currentCryptoKey`  
0x38 - 0x3C 对应 `int hiddenValue`

 

`ACTkByte4` 类型可以当做 `int32_t` , 也是 4 字节  
0x3C - 0x40 对应 `ACTkByte4 hiddenValueOldByte4`

 

`bool` 类型应该占用 1 字节, 但是由于结构体需要 **字节对齐**, 被拓展成了 4 字节  
0x40 - 0x44 对应 `bool inited`

 

`float` 类型占用 4 字节  
0x44 - 0x48 对应 `float fakeValue`

 

0x48 - 0x4C 对应 `bool fakeValueActive`

 

最后的四字节则是 `bool has_value`

使用 C++ 实现 Nullable<>
--------------------

```
// 可空类型
template struct Nullable
{
    T value;
    bool has_value;
 
    bool HasValue() const { return has_value; }
    const T& GetValue() const { return value; }
    void SetValue(T& newValue) { value = newValue; }
}; 
```

再自行把特化类型的定义抄下来, 就能对 Il2cpp 中的 `Nullable<T>` 类型进行操作了  
不嫌麻烦的也可以手动展开特化类型的成员变量, 能用就是不够优雅

解密 ObscuredFloat 类型
===================

定位解密函数
------

通过关键词 `Decrypt` 就能得到几个关键函数, 再过滤一下 `形参` 跟 `返回值` , 传入 key 的那些就可以无视了  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_15-57-57.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_15-57-57.png "ObscuredFloat")

 

看了一圈最可疑的就这三个函数了

```
public float GetDecrypted();                             // 0x17868F4
private float InternalDecrypt();                         // 0x1786904
public static float op_Implicit(ObscuredFloat value);    // 0x21605BC

```

用 **IDA** 查看一下  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-09-26.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-09-26.png "GetDecrypted")  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-12-09.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-12-09.png "InternalDecrypt")  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-13-29.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-13-29.png "op_Implicit")

 

均直接调用另一个内部的解密函数  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-24-47.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_16-24-47.png "sub_2161A10")

测试解密
----

直接 Hook `sub_2161A10` 函数并且调用试试看

 

先进行定义

```
// struct 定义直接照搬就行了, 类型占用大小需要注意一下
struct ACTkByte4
{
    char b1; // 0x0
    char b2; // 0x1
    char b3; // 0x2
    char b4; // 0x3
};
struct ObscuredFloat
{
    int32_t currentCryptoKey; // 0x0
    int32_t hiddenValue; // 0x4
    ACTkByte4 hiddenValueOldByte4; // 0x8
    bool inited; // 0xC
    float fakeValue; // 0x10
    bool fakeValueActive; // 0x14
};
// 原型定义
float (*_ObscuredFloat_Decrypted)(ObscuredFloat*) = nullptr;
float ObscuredFloat_Decrypted(ObscuredFloat* self)
{
    return _ObscuredFloat_Decrypted(self);
}

```

hook 解密函数, map 获取动态库地址的源码太多就不贴上来了

```
DobbyHook((void*)(Il2cppBaseAddr + 0x2161A10),
          (void*)ObscuredFloat_Decrypted, (void**)&_ObscuredFloat_Decrypted);

```

调用解密函数

```
//内存偏移
template inline R MemoryOff(void* addr, ulong off)
{
    return reinterpret_cast((static_cast(addr) + off));
}
 
void (* _xxxDamages)(Il2CppObject*, void*) = nullptr;
void xxxDamages(Il2CppObject* behaviour, void* damageInfo)
{
    auto _damage = MemoryOff*>(damageInfo, 0x34);
    if(_damage->HasValue())
    {
        auto value = ObscuredFloat_Decrypted(_damage->GetValue());
        LOGE("_damage: %0.3f", value);
    }
    return _xxxDamages(behaviour, damageInfo);
} 
```

再去找个陷阱撞一下, 正确识别, 收工  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_17-33-41.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_17-33-41.png "游戏截图")  
[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_17-33-11.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-05_17-33-11.png "logcat")  
[![](https://mihoyo.top/wp-content/uploads/派蒙好耶.gif)](https://mihoyo.top/wp-content/uploads/派蒙好耶.gif "好耶")

用优雅一点的方式 Hook
-------------

解析 B 指令获取目标地址然后进行 hook  
至于怎么解析 B 指令 emmm, 看 arm 手册就行了, 简单概述的话就是去掉 **指令标志位**  
以后有机会再讲一下吧, 这玩意也是个雷, 去年那篇 [Android10 aarch64 dlopen Hook](https://mihoyo.top/android10-aarch64-dlopen-hook) 的解析写法就是有点问题的, 虽然还没触发到那颗雷

```
// 提取B指令的偏移
inline ulong BxxExtract(void* symbol)
{
#if defined(__arm__)
    return static_cast((*static_cast(symbol) << 0x8 >> 0x6));
#elif defined(__arm64__) || defined(__aarch64__)
    return static_cast((*static_cast(symbol) << 0x6 >> 0x4));
#else
#error ABI Error
#endif
}
 
// 修正B指令转跳
template inline R Amend_Bxx(R symbol, ulong off = 0ul)
{
    symbol = MemoryOff(reinterpret_cast(symbol), off);
#if defined(__arm__)
    return MemoryOff(reinterpret_cast(symbol),
                BxxExtract(reinterpret_cast(symbol)) + 0x8);
#elif defined(__arm64__) || defined(__aarch64__)
    return MemoryOff(reinterpret_cast(symbol),
                BxxExtract(reinterpret_cast(symbol)));
#else
#error ABI Error
#endif
}
 
// 地址可通过 主动式Hook 或 被动式Hook 动态获取
DobbyHook((void*)Amend_Bxx(Il2cppBaseAddr + 0x17868F4, 0x4),
          (void*)ObscuredFloat_Decrypted, (void**)&_ObscuredFloat_Decrypted); 
```

加密 & 修改 & 其他类型
==============

[![](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-06_17-00-33.png)](https://mihoyo.top/wp-content/uploads/Snipaste_2021-05-06_17-00-33.png "NTM犯法了你知道吗")

 

点到为止! 点到为止!  
已经讲了很多了, 再多说会被 **♂** 的.

 

搞懂怎么解密后, 加密  
其他类型的也是同样的套路  
先定义类型, 然后阿吧阿吧阿吧

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班开始招生！！](https://bbs.pediy.com/thread-267018.htm)