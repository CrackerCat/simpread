> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267189.htm)

文章同时发表在我[个人博客](https://lovesykun.cn/archives/android-hidden-api-bypass.html)上。转载请务必注明出处。

 

目录

*   [背景](#背景)
*   [旧方法](#旧方法)
*   [柳暗](#柳暗)
*   [花明](#花明)
*            提取 `ArtMethod`
*            计算 `ArtMethod` 大小
*            `Method` 与 `ArtMethod` 互转
*   [稳定性分析](#稳定性分析)
*   [源码](#源码)

背景
==

> 2018 年发布的 Android P 中引入了对[隐藏 API 的限制](https://developer.android.google.cn/distribute/best-practices/develop/restrictions-non-sdk-interfaces)，这对整个 Android 生态来说当然是一件好事，但也严重限制了以往我们通过反射等手段实现的 “黑科技”（如插件化等），所以开发者们纷纷寻找手段绕过这个限制。比如 Canyie 就曾经提出了两个绕过方法，其中一个便是几乎完美的双重反射（即“元反射”，现在来看叫“套娃反射” 比较好）；而在即将发布的 Android R 中把这个方法封杀了。

旧方法
===

这是 Canyie 在其博客上 [^1](https://blog.canyie.top/2020/06/10/hiddenapi-restriction-policy-on-android-r/) 提到的安卓隐藏 API 的问题。博客上还提出了一个看似可行的解决方案：设置类的 `classloader` 为 `null` 成为 `BootClassPath` 中的类以解除限制。此法看似可行，然并非万全：

1.  首先在有隐藏 API 限制的情况下修改自己的 `classloader` 非常困难（但是仍可行）
2.  类是否有隐藏 API 限制是由其在加载时候就设置好的 Domain 决定，所以在加载类之后再修改 `classloader` 就没有用了。
3.  利用 `DexFile` 加载一个没有 `classloader` 的类可以（甚至可以通过 base64 加载一个预制在 java 字符串中的 dex 文件），但是 `DexFile` 已经 deprecated 掉，并且在不日加入豪华隐藏 API 列表 [^2](https://android-review.googlesource.com/c/platform/libcore/+/1666599)。
4.  谷歌方面已经开始着手不信任 `classloader` 为 `null` 的类了 [^6](https://android-review.googlesource.com/c/platform/art/+/1664304)。

柳暗
==

这样看来，纯 Java 绕过隐藏 API 似乎没戏了。只能通过 native 代码，用魔法绕过 `dlopen` 限制来 `dlsym` 在 `libart.so` 中的 `_ZN3artL32VMRuntime_setHiddenApiExemptionsEP7_JNIEnvP7_jclassP13_jobjectArray` 来设置允许隐藏 API 了。

花明
==

等等。

 

我们好像还有一个非常适合玩魔法的 Java 自带的 API：`Unsafe`!  
这个魔法类顾名思义，是非常不安全的：它可以纯 Java 读写内存！也就是说，有了这个东西，我们就可以读写类中的任意数据成员了！甚至如果拿到 native 指针，还能直接读指针指向的内存内容。那么我们是不是可以藉此来读取类的隐藏函数？答案是肯定的。

 

但是，ART 中的函数不是存在一个 Java 数组中乖乖等着给你拿的。ART 的模型是 Java 代码中的重要类和 native 代码中的一个类相互 mirror：即共享同一块内存。所以在 Java 中读写这些 Java 类对象的成员相当于同时修改对应的 native 对象成员。一些常见的类就是 `Class`、`Method`、`Field` 等。于是为了方便 native 代码访问这些对象，这些类的成员很多都不是以 Java 对象形式存在，而是以 native 指针形式存在。比如一个 `Class` 对应的 Java 结构如下 ^3：

```
public final class Class {
    private transient ClassLoader classLoader;
    private transient Class componentType;
    private transient Object dexCache;
    private transient ClassExt extData;
    private transient Object[] ifTable;
    private transient Class superClass;
    private transient Object vtable;
    private transient long iFields;
    private transient long methods;
    private transient long sFields;
    private transient int accessFlags;
    private transient int classFlags;
    //还有其他成员，这里就写了
} 
```

可以看到有很多 `long` 成员，这些就是 native 指针了。很可惜，我们想要的 `methods` 就是以指针形式存在的，而且指针对应的是 `ArtMethod` 的 native 对象，并非一个对应到 Java 的 mirror 的 `Method`。感兴趣的小伙伴可以看看 `Executable` 的实现，其中有一个 `long` 成员是 `artMethod` 对应的就是这个东西。

 

那么我们需要把这个 `methods` 指针对应的 `artMethod` 给一个一个枚举出来，并且想办法把他们转换为 Java 可以用的 `Executable` （用 `Executable` 是这个包含 `Constructor` 和 `Method`）。

提取 `ArtMethod`
--------------

要理解这个指针到底存了啥，我们就要回到 native 代码 ^4。

```
class Class {
  // Pointer to an ArtMethod length-prefixed array. All the methods where this class is the place
  // where they are logically defined. This includes all private, static, final and virtual methods
  // as well as inherited default methods and miranda methods.
  //
  // The slice methods_ [0, virtual_methods_offset_) are the direct (static, private, init) methods
  // declared by this class.
  //
  // The slice methods_ [virtual_methods_offset_, copied_methods_offset_) are the virtual methods
  // declared by this class.
  //
  // The slice methods_ [copied_methods_offset_, |methods_|) are the methods that are copied from
  // interfaces such as miranda or default methods. These are copied for resolution purposes as this
  // class is where they are (logically) declared as far as the virtual dispatch is concerned.
  //
  // Note that this field is used by the native debugger as the unique identifier for the type.
  uint64_t methods_;
};

```

这里解释它是一个 `length-prefixed array` 对象。实际上我们看使用它的地方：

```
inline LengthPrefixedArray* Class::GetMethodsPtr() {
  return reinterpret_cast*>(
      static_cast(GetField64(OFFSET_OF_OBJECT_MEMBER(Class, methods_))));
} 
```

可以看到，它强转成一个模板类 `LengthPrefixedArray` 的指针，其部分定义如下：

```
template class LengthPrefixedArray {
  static size_t OffsetOfElement(size_t index,
                                size_t element_size = sizeof(T),
                                size_t alignment = alignof(T)) {
    DCHECK_ALIGNED_PARAM(element_size, alignment);
    return RoundUp(offsetof(LengthPrefixedArray, data_), alignment) + index * element_size;
  }
  T& AtUnchecked(size_t index, size_t element_size, size_t alignment) {
    return *reinterpret_cast(
        reinterpret_cast(this) + OffsetOfElement(index, element_size, alignment));
  }
  uint32_t size_;
  uint8_t data_[0];
}; 
```

很明显，“类” 如其名，就是一个数组，但是前面塞了一个 `uint32_t` 的长度的数组而已。并且看其取成员的函数 `AtUnchecked`，可以看到数组首元素就是 `this + sizeof(size_)` 对齐到 `alignof(T)`。`sizeof(size_)` 就是 `4`，而 `alignof` 的话，看 `ArtMethod` 定义是没有 `alignas` 或者 `pack` 的属性定义，那么在 32 位下就是 `4` 在 64 位下就是 `8`，对应 `Unsafe` 就是 `addressSize()` 了。

 

总结来说，首个元素地址就是 `methods + unsafe.addressSize()`。而读取方法数量就是 `unsafe.readInt(methods)`。

计算 `ArtMethod` 大小
-----------------

拿到数组首个元素的地址 `base`，接下来第 `i` 个元素就是 `base + i * size` 啦。但是问题来了，怎么计算 `ArtMethod` 的大小呢？这个直接参考 `SandHook` 的实现就可以了：直接在 Java 定义两个连续的方法，然后指针相减就是了 [^5](https://github.com/ganyao114/SandHook/blob/master/doc/doc.md#artmethod-%E7%9A%84%E5%A4%A7%E5%B0%8F)。

 

这时候就涉及第二个问题：怎么从 Java 的 `Method` 转换成 `ArtMethod` 指针？反过来又如何？

`Method` 与 `ArtMethod` 互转
-------------------------

我们之前提到过，`Executable` 里面放了一个 `ArtMethod` 的指针。既然映射到 Java 就证明它有被使用的地方，我们看看它的源码：

```
public abstract class Executable {
    /**
     * @hide - exposed for use by {@code java.lang.invoke.*}.
     */
    public final long getArtMethod() {
        return artMethod;
    }
}

```

非常明确地说明给 `java.lang.invoke.*` 接口使用的。这很好，因为这些接口都是 Java 原生地公开接口，谷歌可不能隐藏他们。那么怎么被使用的呢？或者说，怎么被转换的呢？

```
public abstract class MethodHandle {
  //其他成员...
  /** @hide */ protected final long artFieldOrMethod;
}

```

原来是放在了 `MethodHadle` 上。而且这个东西也是一个 mirror 类，native 和 Java 一对一的。也就是说，只要把一个 `Method` 转换成 `MethodHandle` 然后读出这个 `artFieldOrMethod` 就可以了。

 

然后就是转回来。我们把某个 `MethodHandle` 的这个 `artFieldOrMethod` 设置成 `ArtMethod` 指针之后，再把这个 `MethodHandle` 转成 `Executable` 就行了。要注意，转换成 `Member` 时候，这个接口会检查解析出来的 `Member` 是否能被当前类访问。 不过好在，他得先转换出来才能进行检查。那么，只要忽略这个检查（抛出的异常），然后直接拿出这个成员就行了。

稳定性分析
=====

这个方法在 P-S DP2 上均进行了测试，都能完美运行。并且，谷歌方面已经承诺 `Unsafe` 不会被彻底隐藏，而剩下使用的都是 Java 公开接口，更不可能被隐藏。依赖的 mirror 类的 offset 都被谷歌小心翼翼地维护，改动可能性也不大。因而可以说方法稳定切通用，并且大概率不会被谷歌后续掐掉。

源码
==

源码已经[公开](https://github.com/LSPosed/AndroidHiddenApiBypass)，并且已经上架 Maven，欢迎推广和使用。

[[公告]5 月 14 日腾讯安全零信任发展趋势论坛重磅开幕！邀您一起从 “零” 开始，共建信任！！](https://zta.insecworld.com/?utm_campaign=MJTG&utm_source=KX&utm_medium=WZLJ)

最后于 12 小时前 被 yujincheng08 编辑 ，原因：