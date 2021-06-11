> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/215820)

> 书接前文，本文详细介绍 Go 二进制文件中的数据类型信息，如何定位并解析所有数据类型定义。

[![](https://p5.ssl.qhimg.com/t01a10c24a01079c7d3.jpg)](https://p5.ssl.qhimg.com/t01a10c24a01079c7d3.jpg)

> 书接前文，本文详细介绍 Go 二进制文件中的数据类型信息，如何定位并解析所有数据类型定义。

**传送门**：

1.  [Go 二进制文件逆向分析从基础到进阶——综述](https://www.anquanke.com/post/id/214940)
2.  [Go 二进制文件逆向分析从基础到进阶——MetaInfo、函数符号和源码文件路径列表](https://www.anquanke.com/post/id/215419)

8. types
--------

### 8.1 简介

重温一下本系列第一篇《[Go 二进制文件逆向分析从基础到进阶——综述](https://www.anquanke.com/post/id/214940)》文末介绍的 Go 语言中的数据类型。Go 在构建二进制文时把项目中所有数据类型信息也打包到二进制文件中，这些数据类型信息主要为 Go 语言中的 Stack Trace、Type Reflection 和类型转换服务。Go 语言内置的标准数据类型如下：

[![](https://p3.ssl.qhimg.com/t010f01af0e76e97c60.png)](https://p3.ssl.qhimg.com/t010f01af0e76e97c60.png)

而这些类型的底层实现，其实都基于一个底层的结构定义扩展组合而来：

[![](https://p3.ssl.qhimg.com/t01ec2c57eccba40a41.png)](https://p3.ssl.qhimg.com/t01ec2c57eccba40a41.png)

如果只是一个没有绑定任何 Method 的 Basic Type ，那么用 **rtype** 的结构就可以简单表示。如果一个数据类型绑定了 Methods(这种数据类型也叫 **Uncommon Type**)，或者属于复杂的组合类型 (Composite Type)，那么就需要用扩展组合的方式来表示了。复杂类型的扩展组合方式可以简单描述为（虚线表示可选）：

[![](https://p0.ssl.qhimg.com/t018161cef684a29527.png)](https://p0.ssl.qhimg.com/t018161cef684a29527.png)

这里以一个典型的 Struct 类型的数据结构为例，源码级表示为：

```
// structType represents a struct type.
type structType struct {
    rtype
    pkgPath name
    fields []structField // fields address, sorted by offset
}

type uncommonType struct {
    pkgPath nameOff // import path; empty for built-in types like int, string
    mcount uint16   // number of methods
    xcount uint16   // number of exported methods
    moff uint32     // offset from this uncommontype to [mcount]method
    _ uint32        // unused
}

type structField struct {
    name name           // name is always non-empty
    typ *rtype          // type of field
    offsetEmbed uintptr // byte offset of field<<1 | isEmbedded
}
// Method on non-interface type
type method struct {
    name nameOff // name of method
    mtyp typeOff // method type (without receiver)
    ifn textOff  // fn used in interface call (one-word receiver)
    tfn textOff  // fn used for normal method call
}


```

在 Go 二进制文件中，用 IDAPro 查看实际的数据排列顺序，也如上面源码顺序一样由 “上” 到“下”（地址由高到低）。以 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析的某样本中一个实际的 Struct 数据为例，可以仔细对比一下源码的定义：

[![](https://p2.ssl.qhimg.com/t01b3b1d2e416e821b0.png)](https://p2.ssl.qhimg.com/t01b3b1d2e416e821b0.png)

### 8.2 types 遍历思路

#### 8.2.1 runtime.newobject 的交叉引用

一个 Go 二进制文件中，被静态链接打包进去几千个函数，连带着打包进去的变量、常量加起来也数以千计，对应于数以千计的 Type 定义。上面介绍了其中一个典型的 Struct 类型数据的定义，一个问题很自然地就抛出来了：如何在 Go 二进制文件中定位到这所有的数据类型定义，并解析他们 Type 信息？

如果对 Go 语言稍有了解的话，会知道 Go 底层在定义一个变量 (为某类型的变量分配内存空间) 时，都会调用一个函数 `runtime.newobject()` 。该函数的[源码实现](https://golang.org/src/runtime/malloc.go)如下 (其中的 `_type` 其实就是上面所说的 **rtype**， 参考： [src/runtime/type.go](https://golang.org/src/runtime/type.go)：

```
// implementation of new builtin
// compiler (both frontend and SSA backend) knows the signature
// of this function
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)
}

// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer


```

在 IDAPro 中看到的调用 `runtime.newobject()` 函数的形式通常如下：

[![](https://p3.ssl.qhimg.com/t015e910b3631e00b7c.png)](https://p3.ssl.qhimg.com/t015e910b3631e00b7c.png)

如此一来，就可以在 IDAPro 中遍历 `runtime.newobject()` 函数的交叉引用，然后在汇编指令中提取参数，就获取到了目标数据类型定义的地址。按照这个思路，就能一一定位到这些数据类型：

[![](https://p2.ssl.qhimg.com/t014fc2dfdf3a0c5c6b.png)](https://p2.ssl.qhimg.com/t014fc2dfdf3a0c5c6b.png)

然而，这种方式存在一些问题。可以看到上面 `runtime.newobject()` 函数的交叉引用个数为 **2568** ，而在前文 《**[Go 二进制文件逆向分析从基础到进阶——MetaInfo、函数符号和源码文件路径列表](https://www.anquanke.com/post/id/215419)**》 介绍的 **firstmoduledata** 结构中的 **typelinks** 的个数为 **0xA91(2705)** 。说明 `runtime.newobject()` 函数的交叉引用覆盖不全。漏了哪些呢？下图所示的用法是情况之一，即把某个类型指针当作参数传入一个函数：

[![](https://p5.ssl.qhimg.com/t0123778485c451e459.png)](https://p5.ssl.qhimg.com/t0123778485c451e459.png)

#### 8.2.2 遍历 firstmoduledata.typelinks

所以，通过遍历 `runtime.newobject()` 函数交叉引用的方式来遍历所有数据类型定义，不够完美。最好的方式，上面已经暗示了，是遍历 **firstmoduledata** 结构中的 **typelinks**。 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的 **typelinks** 如下：

[![](https://p2.ssl.qhimg.com/t01c5dfee34c9711f91.png)](https://p2.ssl.qhimg.com/t01c5dfee34c9711f91.png)

**typelinks** 中的数据，以 4-Bytes(uint32) 为单位，每个值代表一个相对于 `firstmoduledata.types` 起始地址的 **Offset**，即 `firstmoduledata.types` 加上这个 **Offset** 值，就是相应的数据类型定义信息的**地址**。 **[go_parser](https://github.com/0xjiayu/go_parser)** 会把每个计算好的地址值，以 Comment 的形式打到每个 Offset 后面，双击 Comment 中的地址值，即可跳转到对应的数据类型定义的位置。

### 8.3 rtype 解析

上文已经阐述了 Go 二进制文件中所有 Type 信息如何组织、存放的，以及通过什么样的方式可以找到这些数据类型定义信息。接下来的问题就是，如何解析每一个找到的数据类型定义，从中提取有助于逆向分析的信息，并以友好的方式在 IDAPro 中展示出来。

#### 8.3.1 rtype

前面提过多次 **rtype**，可以表示最简单的数据类型 (Common Basic Types)。**rtype** 在源码中的定义如下：

```
// Refer: https://golang.org/src/reflect/type.go

type rtype struct {
    size       uintptr
    ptrdata    uintptr  // number of bytes in the type that can contain pointers
    hash       uint32   // hash of type; avoids computation in hash tables
    tflag      tflag    // extra type information flags
    align      uint8    // alignment of variable with this type
    fieldAlign uint8    // alignment of struct field with this type
    kind       uint8    // enumeration for C
    alg        *typeAlg // algorithm table
    gcdata     *byte    // garbage collection data
    str        nameOff  // string form
    ptrToThis  typeOff  // type for pointer to this type, may be zero
}


```

还是以前面贴过的 `*x509.SystemRootsError` 这个类型为例：

[![](https://p5.ssl.qhimg.com/t0174d565caa7f23d93.png)](https://p5.ssl.qhimg.com/t0174d565caa7f23d93.png)

**rtype** 中对逆向分析最有用的字段有以下几个：

*   **tflag**：1 Byte(uint8)，当前类型的 flag；
*   **kind**：1 Byte(uint8)，当前类型的底层类型代码；
*   **str**：4 Bytes(uint32)，当前类型 name string 的偏移 (相对于 **firstmoduledata.types**)
*   **ptrtothis**：4 Bytes(uint32)，如果有另外的**指针类型**指向本类型，那么 **ptrtothis** 就是那个指针类型相对于 **firstmoduledata.types** 的偏移量；

**tflag** 可能的值有 3 个：

*   **star prefix**：即 **nams string** 以星号 `*` 开头，如果 **tflag** 值与 **1** 按位与的结果不为 0，则该类型的 star prefix flag 为 True；
*   **named**：即该类型是被显示命名的，或者是为标准类型拟了一个别名，如果 **tflag** 值与 **2** 按位与的结果不为零，则 named flag 为 True；
*   **Uncommon**：该类型有绑定的 Methods，如果 **tflag** 的值与 **4** 按位与的结果不为零，则该类型为 Uncommon Type。

**kind** 是个 uint 类型的枚举变量，在 [src/reflect/type.go](https://golang.org/src/reflect/type.go) 中的定义的如下：

```
// A Kind represents the specific kind of type that a Type represents.
// The zero Kind is not a valid kind.
type Kind uint

const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    Int16
    Int32
    Int64
    Uint
    Uint8
    Uint16
    Uint32
    Uint64
    Uintptr
    Float32
    Float64
    Complex64
    Complex128
    Array
    Chan
    Func
    Interface
    Map
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)


```

但是 Go 二进制文件中看到的 **rtype** 里 **kind** 字段的值，并不是跟上面的枚举值对应的，而是与一个 Kind 掩码进行按位与计算之后，才能与上面的枚举值对应。Go 语言中的 Kind 掩码定义如下：

```
KIND_MASK           = (1 << 5) - 1


```

**name** 是一个 uint32 类型的值，代表一个相对于 **firstmoduledata.types** 的**偏移量**，这个偏移量加上 **firstmoduledata.types** 得出一个**地址值**，这个地址就是当前 **rtype** 的 **name** 信息所在的位置。不过这个 **name** 既不是以 `0x00` 结尾的 C-String，也不是以 Length 指定长度的一串字符，而是另外一个专门的 **type name** 结构体。

#### 8.3.2 type name

先看一下 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个基础的 type name：

[![](https://p5.ssl.qhimg.com/t01b4866145a51d1778.png)](https://p5.ssl.qhimg.com/t01b4866145a51d1778.png)

一个 type name 基础数据，包含以下字段：

*   **flag**: 1 Byte(uint8)，可以标记本 type name 是否**可导出** (首字母大写)，以及是否附带 **tag data** 或 **package path** ；
*   **length**: 2 Bytes(uint16)，2 个字节通过计算得出本 **type name** string 的长度
*   **name string**：**length** 个 Bytes，就是最终的 type name string。

先介绍一下 2 Bytes 的 **type name** 的 Length 如何计算。**type name** 的源码定义中的一段注释如下：

```
The first byte of type name is a bit field containing:

    1<<0 the name is exported
    1<<1 tag data follows the name
    1<<2 pkgPath nameOff follows the name and tag

The next two bytes are the type name string length:

    len := uint16(type_name_data[1])<<8 | uint16(type_name_data[2])

Bytes [3:3+l] are the string data.


```

Length 计算方式也一目了然，如果用 IDAPython 来表示，就是：

```
len = ((idc.Byte(type_name_addr + 1) & 0xFF << 8) | (idc.Byte(type_name_addr + 2) & 0xFF)) & 0xFFFF


```

**name flag** 的 3 种情况，与 **rtype tflag** 的 3 种情况类似，计算方式也相同，都是分别与 (1, 2, 4) 进行**按位与**运算，依据结果来看本 **type name** 是否可导出，以及是否附带 **tag data** 或 **package path**。在 Go 语言的规范中，**可导出** 就意味着 **首字母大写**。

**tag** 在 Go 中也很常见，相当于给相应的 **type name** 又起了一个别名。最常见的用法是用 Struct 定义 Json 结构时，给 Struct 中的字段打上一个 Json 的 Tag。如果 **type name** 附带了 **tag data**，那么紧接着 **type name** 的 String 数据后 2 Bytes，就是 **tag string** 的长度，该长度值计算方式同于 **type name** length。2 Bytes 之后，就是指定长度的 **tag string** 数据。如下所示：

[![](https://p2.ssl.qhimg.com/t011ac09f71120e4222.png)](https://p2.ssl.qhimg.com/t011ac09f71120e4222.png)

**package path** 则是在 **type name** 的基础数据和 **tag data** 之后，是一个 4 Bytes(uint32) 类型的 Offset 值，这个 Offset 是相对于 **firstmoduledata.types** 来说的，相加之后得出一个 **type name** 的 **地址**。这个地址，指向的是另外一个 **type name** 结构，这个结构就存放了 **pacakge path** 的信息。这个 **package path** 结构的解析，也就跟 **type name** 一样了。下图是一个标准库 **sync** 中的 type name，一目了然：

[![](https://p3.ssl.qhimg.com/t01eb85620189117c71.png)](https://p3.ssl.qhimg.com/t01eb85620189117c71.png)

### 8.4 composite type 解析

Go 中的 Common Basic Types 都可以用上面的 **rtype** 来表示，如果 **rtype.kind** 对应于 Composite Types 其中一个，那么完整的类型定义就需要在 **rtype** 的基础上加上各自独有的字段或者属性才能表示了。本小节就盘点一下这些 Composite Types 的结构，以及解析思路。

#### 8.4.1 Ptr Type

Ptr Type 即指针类型，它指向某一个具体的数据类型。源码定义如下：

```
type ptrType struct {
    rtype
    elem *rtype // pointer element (pointed at) type
}


```

即在 **rtype** 后面又附带了一个指向 **rtype** 的指针 (是地址，不是偏移)，对这个被指向的 **rtype** 的解析，参考上文即可。 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个 Ptr Type 结构效果如图所示：

[![](https://p3.ssl.qhimg.com/t01e595f14ff94b850b.png)](https://p3.ssl.qhimg.com/t01e595f14ff94b850b.png)

#### 8.4.2 Struct Type

Struct Type 即 Go 语言中的结构体。不同于 C 中的 Struct，Go 中的 Struct 中的字段有的可以导出，有的只能私用，还可以匿名存在，最重要的时可以绑定方法，其实更像是面向对象概念中的 **类 (Class)**。Struct Type 源码定义如下：

```
type structType struct {
    rtype
    pkgPath name          // !! pointer
    fields  []structField // sorted by offset
}


```

可以看到 Struct Type 是在 **rtype** 数据后面加了一个 **package path** 和一组 **structField**。**pkgPath** 其实是相对于 **firstmoduledata.types** 的一个偏移，指向一个 **type name** 结构，解析方式参考上文。**fields** ，顾名思义，就是 Struct 中的字段定义信息。

**structField** 在源码中的定义如下：

```
type structField struct {
    name        name    // name is always non-empty
    typ         *rtype  // type of field
    offsetEmbed uintptr // byte offset of field<<1 | isEmbedded
}


```

**structField** 前两个成员对逆向分析最有帮助：

1.  指向一个 **type name** 结构的地址，表示本 **structField** 的 **field name**；
2.  指向一个 **type** 的地址，表示本 **structField** 的数据类型。

综合起来，一个完整的 Struct 结构，经过 **[go_parser](https://github.com/0xjiayu/go_parser)** 的解析，在 IDAPro 中展示如下：

[![](https://p3.ssl.qhimg.com/t016007bb2f15ebe7e0.png)](https://p3.ssl.qhimg.com/t016007bb2f15ebe7e0.png)

#### 8.4.3 Slice Type

Slice 即切片。Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，所以 Go 就以 “动态数组” 的概念提供了一个类似数组，但可灵活伸缩的数据结构——切片。Slice 的源码定义如下：

```
type sliceType struct {
    rtype
    elem *rtype // slice element type
}


```

结构类似 Ptr Type，在 **rtype** 数据后面加上一个指向 **element type** 的地址。**[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个典型的 Slice 类型如下：

[![](https://p0.ssl.qhimg.com/t01835ff3102a17c324.png)](https://p0.ssl.qhimg.com/t01835ff3102a17c324.png)

#### 8.4.4 Array Type

数组类型源码定义如下：

```
type arrayType struct {
    rtype
    elem  *rtype // array element type
    slice *rtype // slice type
    len   uintptr
}


```

Array Type 是在 **rtype** 数据后面附上 Array element type 的地址、对应的 Slice 类型的地址和本 Array Type 的长度。**[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个 Array Type 在 IDAPro 中展示如下：

[![](https://p5.ssl.qhimg.com/t01d5b7b2f51a2d18c8.png)](https://p5.ssl.qhimg.com/t01d5b7b2f51a2d18c8.png)

#### 8.4.5 Interface Type

Go 中的 Interface 类型，指的是定义一组行为 / 方法的数据类型。任何其他实现了这一组方法的类型，都可以说**实现了这个接口**。 Interface Type 的源码定义如下：

```
type interfaceType struct {
    rtype
    pkgPath name      // import path
    methods []imethod // sorted by hash
}


```

即在 **rtype** 的数据后面加上了一个 **pkgPath** 和一组 **imethod**。**pkgPath** 是一个指向 **type name** 结构的地址。**imethod** 就是 Interface 中定义的、必须实现的方法，其源码定义如下：

```
type imethod struct {
    name nameOff // name of method
    typ  typeOff // .(*FuncType) underneath
}


```

两个成员都是相对于 **firstmoduledata.types** 的 **偏移量**，第一个成员 **name** 即当前 Method 的名字，计算得出的地址，指向一个 **type name** 结构；第二个 **typ** 即当前 Method 的类型，其实就是方法的声明信息，计算得出的地址，指向一个 **func type** 的结构。 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个完整的 Interface Type 如下：

[![](https://p1.ssl.qhimg.com/t013277175f8ec3d642.png)](https://p1.ssl.qhimg.com/t013277175f8ec3d642.png)

#### 8.4.6 Func Type

Func Type，顾名思义，就是函数或者方法的类型。源码定义如下：

```
type funcType struct {
    rtype
    inCount  uint16
    outCount uint16 // top bit is set if last input parameter is ...

    padding  uint32 // ! only on some architectures (e.g. x64)
}


```

即在 **rtype** 数据后面放置了 3 个成员，对逆向分析最有用的是 **inCount** 和 **outCount**。**inCount** 其实就是参数的个数；**outCount** 是返回值个数。紧随其后的就是每个参数类型定义的地址、每个返回值类型定义的地址。 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个 Func Type 如下：

[![](https://p5.ssl.qhimg.com/t015f280b5239298f58.png)](https://p5.ssl.qhimg.com/t015f280b5239298f58.png)

#### 8.4.7 Map Type

Map Type 就是映射或者字典类型，由 Key 和 Value 构成。Map Type 的源码定义如下：

```
type mapType struct {
    rtype
    key    *rtype // map key type
    elem   *rtype // map element (value) type
    bucket *rtype // internal bucket structure
    // function for hashing keys (ptr to key, seed) -> hash
    hasher     func(unsafe.Pointer, uintptr) uintptr
    keysize    uint8  // size of key slot
    valuesize  uint8  // size of value slot
    bucketsize uint16 // size of bucket
    flags      uint32
}


```

可见 Map Struct 比较复杂，在 **rtype** 数据后附加了比较多的字段，而其中对逆向分析比较有帮助的只有 2 个：**key** 和 **elem**，顾名思义，就是 **key** 指向的类型定义数据和 **element(value)** 的数据类型定义数据。**[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个 Map Type 如下：

[![](https://p4.ssl.qhimg.com/t01a877f86bf136c894.png)](https://p4.ssl.qhimg.com/t01a877f86bf136c894.png)

#### 8.4.8 Chan Type

Chan Type，即 **Channel(通道)** 类型，是 Go 中一个比较特殊的数据类型。这个类型主要是用来在 Goroutine 之间传递消息、同步数据，是 Go 原生高并发特性的支撑要素之一。Chan Type 的源码定义如下：

```
type chanType struct {
    rtype
    elem *rtype  // channel element type
    dir  uintptr // channel direction (ChanDir)
}


```

一个 Channel 在使用时，只能传输一种类型的数据，在声明或者创建时，要指定一个可传输的数据类型，比如创建一个可传输 int 类型值的 channel：

```
ch := make(chan int)


```

另外，Go 中的 Channel 是有方向的。虽然 Channel 默认既可以发送数据，也可以接收数据，但也可以通过指定方向让它做到只能发送或只能接收数据。

所以，上面可以看到 Chan Type 的源码定义中，在 **rtype** 数据后附加了两个字段：指向一个可发送的数据类型的定义的地址 **elem**，和一个代表 Channel 方向（单向接收为 1；单向发送为 2，双向收发为 3）的值。**[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的一个 Chan Type 如下：

[![](https://p0.ssl.qhimg.com/t01dd36df81edfa96c5.png)](https://p0.ssl.qhimg.com/t01dd36df81edfa96c5.png)

#### 8.4.9 Ucommon Type

前面提了多次可以绑定 Methods 的 **Uncommon Type**，具体是什么样的呢？源码定义如下：

```
// uncommonType is present only for defined types or types with methods
// (if T is a defined type, the uncommonTypes for T and *T have methods).
// Using a pointer to this struct reduces the overall size required
// to describe a non-defined type with no methods
type uncommonType struct {
    pkgPath nameOff // import path; empty for built-in types like int, string
    mcount  uint16  // number of methods
    xcount  uint16  // number of exported methods
    moff    uint32  // offset from this uncommontype to [mcount]method
    _       uint32  // unused
}


```

任何一个 Type，无论是 Basic Type 还是 Composite Type，都可以是 Uncommon Type。如果一个 Type 的 **tflag** 字段标记该 Type 时 **Uncommon Type**，那么在该 Type 前面所有的字段之后，就是 **Uncommon Type** 的信息了。

第一个字段是 **pkgPath**，这个字段的用法与 **Interface Type** 中的 **pkgPath** 相同。

第二个字段是 **mcount**，即所有绑定的 Methods 的数量；第三个字段 **xcount** 则是可导出的 Methods 的数量，即 Method name 首字母大写。第 4 个字段，是 Methods 列表到 **Uncommon Type** 信息起始地址的 **偏移**。

Uncommon Type 这里绑定的 Method，与 Interface 那里声明的 **Interface Method** 定义还不一样：

```
type method struct {
    name nameOff // name of method
    mtyp typeOff // method type (without receiver) // offset to an *rtype
    ifn  textOff // fn used in interface call (one-word receiver) // offset from top of text section
    tfn  textOff // fn used for normal method call // offset from top of text section
}


```

*   **name** 是相对于 **firstmoduledata.types** 的一个偏移，两者相加之后得出的地址指向一个 **type name** 结构，解析出来就是 Method name；
*   **mtyp** 也是对于 **firstmoduledata.types** 的一个便宜，两者相加之后得出的地址，指向一个 Type 定义信息，其实就是 Method 的声明信息；
*   后面的 **ifn/tfn** 通常指向这个 Method 实际的函数实现，不过一个是面向 Interface 的，一个就是普通的实现。

综合起来，一个简单的 Uncommon Type 由 **[go_parser](https://github.com/0xjiayu/go_parser)** 解析好的效果如下：

[![](https://p5.ssl.qhimg.com/t017270dcf121c5d49a.png)](https://p5.ssl.qhimg.com/t017270dcf121c5d49a.png)

### 8.5 总结

至此，本文阐述了如何遍历找到 Go 二进制文件中每一个 Type 的定义信息，以及每一个 Type 源码级的定义和解析方式。之后就可以基于这些知识完成对 Go 二进制文件中 Types 信息的解析了。

参考资料：
-----

1.  [https://github.com/0xjiayu/go_parser](https://github.com/0xjiayu/go_parser)
2.  [https://www.anquanke.com/post/id/214940](https://www.anquanke.com/post/id/214940)
3.  [https://www.anquanke.com/post/id/215419](https://www.anquanke.com/post/id/215419)
4.  [https://golang.org/src/runtime/type.go](https://golang.org/src/runtime/type.go)
5.  [https://golang.org/src/runtime/malloc.go](https://golang.org/src/runtime/malloc.go)
6.  [https://www.pnfsoftware.com/blog/analyzing-golang-executables/](https://www.pnfsoftware.com/blog/analyzing-golang-executables/)
7.  [http://home.in.tum.de/~engelke/pubs/1709-ma.pdf](http://home.in.tum.de/~engelke/pubs/1709-ma.pdf)