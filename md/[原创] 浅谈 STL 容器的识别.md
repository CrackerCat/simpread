> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270547.htm)

> [原创] 浅谈 STL 容器的识别

浅谈 STL 容器的识别
============

[](#一、前言)一、前言
-------------

在逆向由 C++ 编写的产品时，最头疼的莫过于两个问题：虚函数和 STL，前者会生成一些间接 CALL，导致 IDA 在分析时无法进一步交叉引用，后者采用了大量模板技术，再加上编译器优化，导致比较难提取到固定的签名来识别这些函数。因此，本文打算结合源码、静态分析和动态调试，谈一谈常用的 STL 容器的基本原理和识别方法。

[](#二、stl容器分析)二、STL 容器分析
------------------------

### 编译环境

vs2019（v142）+ x86 + Release + MT

### **string**

首先简单浏览一下 string 在源码中的内存布局，有一个大致印象。

```
template , class _Alloc = allocator<_Elem>>
class basic_string {
private:
    using _Alty        = _Rebind_alloc_t<_Alloc, _Elem>;
    using _Alty_traits = allocator_traits<_Alty>;
    using _Scary_val   = _String_val, _Simple_types<_Elem>,
        _String_iter_types<_Elem, typename _Alty_traits::size_type, typename _Alty_traits::difference_type,
            typename _Alty_traits::pointer, typename _Alty_traits::const_pointer, _Elem&, const _Elem&>>>;
    // 省略若干代码
private:
    _Compressed_pair<_Alty, _Scary_val> _Mypair;
}; 
```

估计对 C++ 不熟悉的朋友看到上面这堆代码已经懵逼了，我在这里只根据源码简单的解释一下 string 中的内存布局，因此看不懂的也没有关系，并不影响。

 

string 中只有一个类成员变量_Mypair 并且类型是 pair（对_Compressed_pair 感兴趣可以自行深入看一下，关键技术：Empty base optimization），绝大多数情况下，_Mypair 在 Release 下的内存布局与_Scary_val（_Mypair 的第二个类型）一致，_Scary_val 是一个根据 Alloc 类型来推导出来的，具体细节不在赘述，最终找到了如下代码：

```
template class _String_val : public _Container_base {
public:
    using value_type = typename _Val_types::value_type;
    using pointer    = typename _Val_types::pointer;
    using size_type  = typename _Val_types::size_type;
    // 对于string的情况下_BUF_SIZE == 16
    static constexpr size_type _BUF_SIZE = 16 / sizeof(value_type) < 1 ? 1 : 16 / sizeof(value_type);
    // 省略若干代码
    union _Bxty { // storage for small buffer or pointer to larger one
        value_type _Buf[_BUF_SIZE];
        pointer _Ptr;
        char _Alias[_BUF_SIZE]; // TRANSITION, ABI: _Alias is preserved for binary compatibility (especially /clr)
    } _Bx;
    // 在x86下size_type = uint32_t
    size_type _Mysize; // current length of string
    size_type _Myres; // current storage reserved for string
}; 
```

现在代码已经非常清晰了，可以简单的理解为 string 由三个类成员变量组成：Union 类型的_Bx、size_type 类型的_Mysize 和_Myres。现在必须要说一下 string 的短字符串优化（SSO）技术，string 为了减少对内存的申请次数，如果字符串过短（小于 16 字节）则直接将字符串拷贝到自身的 buffer 中，如果字符串长度超过自身 buffer 预设的大小，则会申请一块内存来存储该字符串，自身则只记录申请的内存地址。基于源码的讲解就到这里，接下来，编译成二进制进行分析。

 

**原始 C++ 代码：**

```
int main(int argc, char* argv[]) {
    std::string v1 = argv[0];
    printf("%s\n", v1.c_str());
    return 0;
}

```

**IDA 反编译代码：**

```
// 代码从IDA中提取，为了方便解释，结构和变量名有所调整
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char *v3; // edx
  const char *v4; // ecx
  void **v5; // eax
  void *v6; // ecx
  // string的内存大小为sizeof(void*) * 6
  void *str[6]; // [esp+4h] [ebp-1Ch] BYREF
 
  // 1. 初始化string
  // 将_Mysize设置成0
  str[4] = 0;
  // 将_Myres设置成15
  str[5] = (void *)15;
  // 由于C风格字符串由0结尾，因此初始化阶段将_Buf的首个字节设置成0
  LOBYTE(str[0]) = 0;
 
  // 2. 将argv[0]赋值给string
  v3 = (char *)*argv;
  v4 = *argv;
  // 具体的赋值函数
  sub_401260(str, v3, strlen(v4));
 
  // 3. 获取string的字符串首地址
  v5 = str; // 通过_Buf获取字符串首地址
  // 判断_Myres是否大于等于16
  if ( str[5] >= (void *)0x10 )
    v5 = (void **)str[0]; // 通过_Ptr获取字符串首地址
  // 打印结果
  printf("%s\n", (const char *)v5);
 
  // 4. 释放string的内存
  // 如果_Myres大于16则需要释放字符串的内存
  if ( str[5] >= (void *)0x10 )
  {
    v6 = str[0];
    // 与BigAllocation有关，本文不过多介绍
    if ( (unsigned int)str[5] + 1 >= 0x1000 )
    {
      v6 = (void *)*((_DWORD *)str[0] - 1);
      if ( (unsigned int)(str[0] - v6 - 4) > 0x1F )
        _invalid_parameter_noinfo_noreturn();
    }
    sub_4014BC(v6); // j_free的间接调用
  }
  return 0;
}

```

**关于 string 的识别：**

 

在 IDA 中，可以根据字符串 "string too long" 来进行识别，例如上面这个例子，在函数 sub_401260 中，有一个关于 Size 的判断：

```
void *__thiscall sub_401260(void *this, void *Src, size_t Size)
{
    // ...
    if ( Size > 0x7FFFFFFF )
        sub_4011A0();
    // ...
}
void __noreturn sub_4011A0()
{
  sub_40145B("string too long");
}

```

在 OD/X64DBG 中，可以根据内存布局来快速判断，基本有两种形式：

```
// 第一种，小于16字节的字符串
// _Buf[16]
012FF9AC 6C6C6548 Hell
012FF9B0 726F576F oWor
012FF9B4 0000646C ld..
012FF9B8 00855054 TP..
012FF9BC 0000000A .... // _Mysize=0000000A
012FF9C0 0000000F .... // _Myres=0000000F
 
// 第二种，大于等于16字节的字符串
// _Ptr
0133F844 013F7E88 .~?. &"C:\\Users\\Administrator\\Desktop\\stl_reverse\\test\\Release\\test.exe"
// 12个字节的未使用空间
0133F848 008515B0 °...
0133F84C 00000000 ....
0133F850 00855054 TP..
0133F854 00000040 @... // _Mysize=00000040
0133F858 0000004F O... // _Myres=0000004F

```

### **vector**

```
template >
class vector { // varying size array of values
    using _Alty        = _Rebind_alloc_t<_Alloc, _Ty>;
    using _Alty_traits = allocator_traits<_Alty>;
    using _Scary_val   = _Vector_val, _Simple_types<_Ty>,
        _Vec_iter_types<_Ty, size_type, difference_type, pointer, const_pointer, _Ty&, const _Ty&>>>;
    // 省略若干代码
    _Compressed_pair<_Alty, _Scary_val> _Mypair;
}; 
```

根据分析 string 的经验，可以很容易看出来_Vector_val 的内存布局即是 vector 的内存布局

```
template class _Vector_val : public _Container_base {
public:
    using pointer = typename _Val_types::pointer;
    pointer _Myfirst; // pointer to beginning of array
    pointer _Mylast; // pointer to current end of sequence
    pointer _Myend; // pointer to end of array
}; 
```

vector 的实现还是非常简单的，只有三个类成员变量：_Myfirst、_Mylast 和_Myend，分别指向数组的开始、有效元素的结尾和数组的结尾。vector 中没有记录 size，而是通过_Mylast-_Myfirst 计算而来的，同理，capacity 是通过_Myend-_Myfirst 计算而来的。下面，我们直接看代码。

 

**原始 C++ 代码：**

```
int main(int argc, char* argv[]) {
    struct Value {
        int a;
        int b;
    };
    std::vector v;
    for (int i = 0; i < argc; ++i) {
        v.push_back(Value{ i, i + 1 });
    }
    for (auto& i : v) {
        printf("%d, %d\n", i.a, i.b);
    }
    return 0;
} 
```

**IDA 反编译代码：**

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  _DWORD *v3; // ecx
  DWORD v4; // esi
  DWORD v5; // ebx
  int v6; // edi
  int v7; // eax
  _DWORD *v8; // edi
  char *v9; // eax
  int v11; // [esp+10h] [ebp-24h] BYREF
  DWORD v12; // [esp+14h] [ebp-20h]
  DWORD v13[3]; // [esp+18h] [ebp-1Ch] BYREF
  int v14; // [esp+30h] [ebp-4h]
 
  v3 = 0;
  v4 = 0;
  v12 = 0;
  v5 = 0;
  // 1. 初始化vector
  v13[0] = 0; // _Myfirst
  v13[1] = 0; // _Mylast
  v13[2] = 0; // _Myend
  v6 = 0;
  v14 = 0;
 
  // 2. for (int i = 0; i < argc; ++i)
  if ( argc > 0 )
  {
    do
    {
      v7 = v6++;
      v11 = v7;
      v12 = v6;
      // v4代表_Mylast
      // v5代表_Myend
      if ( v4 == v5 ) // 判断一下是否有足够的空间
      {
        // 没有足够的空间容纳新的Value，调用sub_401300分配新的内存并将Value push到最后
        // sub_401300对应vector源码中的_Emplace_reallocate
        sub_401300((const void **)v13, (_BYTE *)v4, &v11);
        v5 = v13[2];
        v4 = v13[1];
      }
      else
      {
        // 有足够的空间容纳新的Value，不需要重新分配内存
        *(_DWORD *)v4 = v7; // 设置Value::a
        *(_DWORD *)(v4 + 4) = v6; // 设置Value::b
        v4 += 8;
        v13[1] = v4; // _Mylast++
      }
    }
    while ( v6 < argc );
    v3 = (_DWORD *)v13[0];
    v12 = v13[0];
  }
  v8 = v3;
 
  // 3. for (auto& i : v)
  // 具体遍历过程比较直观，就是对线性数组的遍历，不再给出详细解释
  if ( v3 != (_DWORD *)v4 )
  {
    do
    {
      // v8代表_Myfirst
      sub_401010("%d, %d\n", *v8, v8[1]);
      v8 += 2;
    }
    while ( v8 != (_DWORD *)v4 );
    v3 = (_DWORD *)v12;
  }
 
  // 4. 释放vector的内存
  if ( v3 )
  {
    v9 = (char *)v3;
    if ( ((v5 - (_DWORD)v3) & 0xFFFFFFF8) >= 0x1000 )
    {
      v3 = (_DWORD *)*(v3 - 1);
      if ( (unsigned int)(v9 - (char *)v3 - 4) > 0x1F )
        _invalid_parameter_noinfo_noreturn();
    }
    sub_40159B(v3);
  }
  return 0;
}

```

**关于 vector 的识别：**

 

在 IDA 中，可以采用和 string 相同的方式来识别 vector，使用 vector 后编译会产生 "vector too long" 的字符串

```
_DWORD *__thiscall sub_401300(const void **this, _BYTE *a2, _DWORD *a3)
{
  // ...
  // 通过2^32 / 0x1FFFFFFF计算得出8即是该vector的元素大小
  if ( v4 == 0x1FFFFFFF )
    sub_401460();
  // ...
}
void __noreturn sub_401460()
{
  sub_40153A("vector too long");
}

```

在 OD/X64DBG 中，可以根据内存布局来快速判断，vector 只有一种形式：

```
// vector内存布局
005EFB2C 00878470 p... // _Myfirst
005EFB30 00878478 x... // _Mylast
005EFB34 00878478 x... // _Myend
 
// vector指向的数组内存
00878470 00000000 ....
00878474 00000001 ....

```

在最后，顺便说一下 vector 的内存分配策略，直接上源码：

```
size_type _Calculate_growth(const size_type _Newsize) const {
    // given _Oldcapacity and _Newsize, calculate geometric growth
    const size_type _Oldcapacity = capacity();
    const auto _Max              = max_size();
    if (_Oldcapacity > _Max - _Oldcapacity / 2) {
        return _Max; // geometric growth would overflow
    }
    const size_type _Geometric = _Oldcapacity + _Oldcapacity / 2;
    if (_Geometric < _Newsize) {
        return _Newsize; // geometric growth would be insufficient
    }
    return _Geometric; // geometric growth is sufficient
}

```

代码也非常清晰，通过这句代码可以看出`const size_type _Geometric = _Oldcapacity + _Oldcapacity / 2;`，一般情况下，增长后的内存大小是原内存大小的 1.5 倍。

### **list**

list 是一个双向循环链表，整体的设计与前两个容器非常相似，直接看代码。

```
template >
class list { // bidirectional linked list
private:
    using _Alty          = _Rebind_alloc_t<_Alloc, _Ty>;
    using _Alty_traits   = allocator_traits<_Alty>;
    using _Node          = _List_node<_Ty, typename allocator_traits<_Alloc>::void_pointer>;
    using _Alnode        = _Rebind_alloc_t<_Alloc, _Node>;
    using _Alnode_traits = allocator_traits<_Alnode>;
    using _Nodeptr       = typename _Alnode_traits::pointer;
    using _Val_types     = conditional_t<_Is_simple_alloc_v<_Alnode>, _List_simple_types<_Ty>,
        _List_iter_types<_Ty, typename _Alty_traits::size_type, typename _Alty_traits::difference_type,
            typename _Alty_traits::pointer, typename _Alty_traits::const_pointer, _Ty&, const _Ty&, _Nodeptr>>;
    using _Scary_val     = _List_val<_Val_types>;
    // 省略若干代码
private:
    _Compressed_pair<_Alnode, _Scary_val> _Mypair;
};
 
template struct _List_node { // list node
    using value_type = _Value_type;
    using _Nodeptr   = _Rebind_pointer_t<_Voidptr, _List_node>;
    _Nodeptr _Next; // successor node, or first element if head
    _Nodeptr _Prev; // predecessor node, or last element if head
    _Value_type _Myval; // the stored value, unused if head
    // 省略若干代码
};
 
template class _List_val : public _Container_base {
public:
    using _Nodeptr  = typename _Val_types::_Nodeptr;
    using size_type = typename _Val_types::size_type;
    // 省略若干代码
private:
    _Nodeptr _Myhead; // pointer to head node
    size_type _Mysize; // number of elements
}; 
```

list 的内存布局在 Release 下可以基本看成和_List_val 一致，_List_val 是由两个类成员函数_Myhead 和_Mysize 组成的，前者指向 list 的头节点，后者用于统计 list 的当前 size（在 c++11 以前，list::size() 的算法时间复杂度可能是 O(n)，c++11 后保证了时间复杂度是 O(1)，因此可以放心用 list::size() 获取链表的元素个数了）。根据刚才的介绍，应该很容易能算出 list 本身所占的内存大小了（在 32 位下是 8，64 位下是 16）。

 

然后我们再看一下_List_node 的声明，刚才已经说过 stl 的 list 是双向循环链表，因此肯定包含 Next 和 Prev，在这里可以记一下 Next 和 Prev 的顺序，以后在用动态调试器分析时可以快速遍历整个 list，紧接着是 list 的 Value 类型，也就是 list<T> 当中 T 的类型。至此，list 的全部类成员变量都已经分析完毕，接下来，编译成二进制进行分析。

 

**原始 C++ 代码：**

```
int main(int argc, char* argv[]) {
    struct Value {
        int a;
        int b;
    };
    std::list v;
    for (int i = 0; i < argc; ++i) {
        v.push_back(Value{ i, i + 1 });
    }
    for (auto& i : v) {
        printf("%d, %d\n", i.a, i.b);
    }
    return 0;
} 
```

**IDA 反编译代码：**

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  _DWORD *v3; // edi
  int i; // esi
  int v5; // ebx
  _DWORD *v6; // eax
  _DWORD *v7; // ecx
  _DWORD *v8; // esi
  _DWORD *v9; // eax
  _DWORD *v10; // esi
  DWORD Block[2]; // [esp+18h] [ebp-18h]
 
  // 1. 初始化list
  // 设置list的_Mysize为0
  Block[1] = 0;
  // 申请一个_List_node作为头节点
  v3 = operator new(0x10u);
  // 双向链表初始化
  *v3 = v3;   // v3->_Next = head;
  v3[1] = v3; // v3->_Prev = head;
  // 将初始化后的node赋值给list的_Myhead
  Block[0] = (DWORD)v3;
 
  // 2. for (int i = 0; i < argc; ++i)
  for ( i = 0; i < argc; v3 = (_DWORD *)Block[0] )
  {
    // 这里list::push_back的代码全部被inline进来了
    v5 = i++;
    if ( Block[1] == 0xFFFFFFF )
      sub_4013D2("list too long");
    v6 = operator new(0x10u); // 申请一个新的_List_node
    // 设置Value
    v6[2] = v5;
    v6[3] = i;
    ++Block[1]; // 递增list的_Mysize
    // 串链表操作
    v7 = (_DWORD *)v3[1]; // v7 = v3->Prev;  // v7是临时变量
    *v6 = v3;             // v6->_Next = v3; // v6是新节点
    v6[1] = v7;           // v6->_Prev = v7; // v3是head节点
    v3[1] = v6;           // v3->_Prev = v6;
    *v7 = v6;             // v7->_Next = v6;
  }
 
  // 3. for (auto& i : v)
  v8 = (_DWORD *)*v3;
  if ( (_DWORD *)*v3 != v3 ) // 先判断一下链表是否是空的
  {
    // 循环打印链表的每一个节点
    do
    {
      sub_401010("%d, %d\n", v8[2], v8[3]);
      v8 = (_DWORD *)*v8; // v8 = v8->_Next;
    }
    while ( v8 != v3 );
    v3 = (_DWORD *)Block[0];
  }
 
  // 4. 释放list的内存
  // 对应list::_Tidy的代码
  *(_DWORD *)v3[1] = 0; // 将最后一个节点的_Next置为NULL，循环时判断到NULL了就结束循环
  v9 = (_DWORD *)*v3;
  if ( *v3 )
  {
    // 循环释放链表中的每一个节点
    do
    {
      v10 = (_DWORD *)*v9;
      sub_401433(v9); // jmp to free
      v9 = v10;
    }
    while ( v10 );
  }
  // 释放链表的头节点
  sub_401433((void *)Block[0]); // jmp to free
  return 0;
}

```

**关于 list 的识别：**

 

在 IDA 中的识别方式与 string 和 vector 相同，不再具体说明，在 OD/X64DBG 中，可以根据内存布局来快速判断，list 只有一种形式：

```
// _List_val
00DAF8EC 011A8470 p... // *_List_Node，指向头节点
00DAF8F0 00000001 .... // _Mysize
 
// 头节点
011A8470 011A8488 .... // _Next
011A8474 011A8488 .... // _Prev
011A8478 73656C69 iles // 无效数据
011A847C 4D4F4300 .COM // 无效数据
 
// Value节点
011A8488 011A8470 p... // _Next
011A848C 011A8470 p... // _Prev
011A8490 00000000 .... // i
011A8494 00000001 .... // i + 1

```

简单容器类型到这里就结束了，下面将介绍较复杂的容器类型 map 和 unordered_map(hash_map)。

### **map**

stl 中的 map 是基于红黑树实现的，本文不涉及红黑树的具体原理，仅站在内存布局的角度说明 msvc 版本红黑树的实现。

 

**map 和 set 的实现区别：**

 

map 和 set 底层都是红黑树实现的，在 msvc 版本的 stl 中，它们均继承自_Tree class，唯一的区别就是 map 是由 key 和 value 组成的，而 set 只有 key 而没有 value。multimap 和 multiset 除了允许重复 key 以外其它和非 multi 版本一致。

```
template class _Tree { // ordered red-black tree for map/multimap/set/multiset
public:
    using value_type     = typename _Traits::value_type;
    using allocator_type = typename _Traits::allocator_type;
 
protected:
    using _Alty          = _Rebind_alloc_t;
    using _Alty_traits   = allocator_traits<_Alty>;
    using _Node          = _Tree_node;
    using _Alnode        = _Rebind_alloc_t;
    using _Alnode_traits = allocator_traits<_Alnode>;
    using _Nodeptr       = typename _Alnode_traits::pointer;
 
    using _Scary_val = _Tree_val, _Tree_simple_types,
        _Tree_iter_types>>;
 
    enum _Redbl { // colors for link to parent
        _Red,
        _Black
    };
    // 省略若干代码
 
private:
    _Compressed_pair> _Mypair;
}; 
```

根据前几个容器的分析经验，直接找到_Scary_val 对应的类型即可，最终代码如下：

```
template class _Tree_val : public _Container_base {
public:
    using _Nodeptr  = typename _Val_types::_Nodeptr;
    using size_type = typename _Val_types::size_type;
    // 省略若干代码
    _Nodeptr _Myhead; // pointer to head node
    size_type _Mysize; // number of elements
};
 
template struct _Tree_node {
    using _Nodeptr   = _Rebind_pointer_t<_Voidptr, _Tree_node>;
    using value_type = _Value_type;
    _Nodeptr _Left; // left subtree, or smallest element if head
    _Nodeptr _Parent; // parent, or root of tree if head
    _Nodeptr _Right; // right subtree, or largest element if head
    char _Color; // _Red or _Black, _Black if head
    char _Isnil; // true only if head (also nil) node; TRANSITION, should be bool
    value_type _Myval; // the stored value, unused if head
 
    enum _Redbl { // colors for link to parent
        _Red,
        _Black
    };
}; 
```

相比普通的红黑树，stl 中的红黑树多了一个 head 节点，并且有以下几个特点：

1.  head 节点中的_Isnil=true
2.  head 节点的_Parent 指向红黑树的根节点
3.  head 节点的_Left 指向红黑树的最左节点，以便于常数时间实现 begin()
4.  head 节点的_Right 指向红黑树的最右节点

![](https://bbs.pediy.com/upload/attach/202112/791694_HAACU3G9G5856KM.png)

 

节点的 left 或者 right 为 null 的情况没有在图中表现出来，实际在 stl 中，它们被指向 head，例如：most left 节点的 left 节点本来应该是 null，但在 stl 的实现中，它被指向 head，可通过_Isnil 判断。

 

介绍完了和普通红黑树的基本区别，接下来我们看一下_Tree 的内存布局，_Tree 本身是由_Myhead 和_Mysize 组成的，在 32 位下_Tree 总共占用 8 字节空间，而_Tree_node 的内存大小要根据实际 Value 的类型来计算，在下面的例子中，_Tree_node 总共占用 24 字节空间（_Left(4)+_Parent(4)+_Right(4)+_Color(1)+_Isnil(1)+2 字节填充 +_Myval(8)）。

 

**原始 C++ 代码：**

```
int main(int argc, char* argv[]) {
    std::map v;
    for (int i = 0; i < argc; ++i) {
        v[i] = i + 1;
    }
    for (auto& i : v) {
        printf("%d, %d\n", i.first, i.second);
    }
    return 0;
} 
```

**IDA 反编译代码：**

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  _DWORD *v3; // eax
  int v4; // ecx
  int *v5; // esi
  int **v6; // eax
  int *i; // eax
  int *j; // ecx
  _DWORD *v9; // esi
  void *v10; // eax
  int v12; // [esp+Ch] [ebp-1Ch] BYREF
  int v13[2]; // [esp+10h] [ebp-18h] BYREF
  int v14; // [esp+24h] [ebp-4h]
 
  // 1. 初始化Tree
  v13[1] = 0;
  // 申请_Tree_node作为head节点
  v3 = operator new(24u);
  *v3 = v3;   // v3->_Left = v3;
  v3[1] = v3; // v3->_Parent = v3;
  v3[2] = v3; // v3->_Right = v3;
  // 同时给_Color和_Isnil，这里是固定的
  // _Color = _Redbl::_Black;
  // _Isnil = true;
  *((_WORD *)v3 + 6) = 0x101;
  v13[0] = (int)v3; // 将Tree的head设置为刚才申请的_Tree_node
 
  // 2. for (int i = 0; i < argc; ++i)
  v4 = 0;
  v14 = 0;
  v12 = 0;
  if ( argc > 0 )
  {
    do
    {
      // 这里生成了一个emplace对应的函数
      // 参数是key，返回值是value的指针
      *sub_401310(v13, &v12) = v4 + 1;
      v4 = v12 + 1;
      v12 = v4;
    }
    while ( v4 < argc );
    v3 = (_DWORD *)v13[0];
  }
 
  // 3. for (auto& i : v)
  // 重点分析一下红黑树的遍历过程
  v5 = (int *)*v3;
  if ( !*(_BYTE *)(*v3 + 13) ) // !v3->_Isnil
  {
    do
    {
      printf("%d, %d\n", v5[4], v5[5]);
      // 以下是"_Tree_unchecked_const_iterator::_Tree_unchecked_const_iterator& operator++() noexcept"展开后的代码
      v6 = (int **)v5[2];
      // 判断_Right节点是否是nil
      if ( *((_BYTE *)v6 + 13) )
      {
        // 是nil，向上回溯，直到v5是i（父节点）的_Left节点时停止，此时i（父节点）即为下一个节点
        for ( i = (int *)v5[1]; !*((_BYTE *)i + 13); i = (int *)i[1] )
        {
          if ( v5 != (int *)i[2] ) // 判断i（父节点）节点的_Right节点与v5不相等时，即与_Left相等时终止
            break;
          v5 = i;
        }
        v5 = i;
      }
      else
      {
        // 不是nil，获取_Right节点的most left节点，即为下一个节点
        v5 = (int *)v5[2];
        for ( j = *v6; !*((_BYTE *)j + 13); j = (int *)*j )
          v5 = j;
      }
    }
    while ( !*((_BYTE *)v5 + 13) );
    v3 = (_DWORD *)v13[0];
  }
 
  // 4. 释放Tree的内存
  v9 = (_DWORD *)v3[1];
  if ( !*((_BYTE *)v9 + 13) )
  {
    do
    {
      // 递归释放子树
      sub_401620((int)v13, (void *)v9[2]);
      v10 = v9;
      v9 = (_DWORD *)*v9;
      sub_40178B(v10); // jmp to free
    }
    while ( !*((_BYTE *)v9 + 13) );
    v3 = (_DWORD *)v13[0];
  }
  sub_40178B(v3); // jmp to free
  return 0;
}

```

对遍历过程有疑问的可以对照下面的 C++ 源码：

```
_Tree_unchecked_const_iterator& operator++() noexcept {
    if (_Ptr->_Right->_Isnil) { // climb looking for right subtree
        _Nodeptr _Pnode;
        while (!(_Pnode = _Ptr->_Parent)->_Isnil && _Ptr == _Pnode->_Right) {
            _Ptr = _Pnode; // ==> parent while right subtree
        }
 
        _Ptr = _Pnode; // ==> parent (head if end())
    } else {
        _Ptr = _Mytree::_Min(_Ptr->_Right); // ==> smallest of right subtree
    }
 
    return *this;
}

```

最后看一下在内存中的布局：

```
// _Tree_val
004FF7C0 00818470 p... // _Myhead
004FF7C4 00000001 .... // _Mysize
 
// 头节点的数据
00818470 00818490 .... // _Left
00818474 00818490 .... // _Parent
00818478 00818490 .... // _Right
0081847C 4D4F0101 ..OM // _Isnil=true, _Color=1
// 头节点后面是无效数据
00818480 45545550 PUTE
00818484 4D414E52 RNAM
 
// 节点的数据
00818490 00818470 p... // _Left
00818494 00818470 p... // _Parent
00818498 00818470 p... // _Right
0081849C 5C3A0001 ..:\ // _Isnil=false, _Color=1
// 非头节点，有效数据
008184A0 00000000 .... // key=0
008184A4 00000001 .... // value=1

```

### **unordered_map**

最后一个使用率比较高的容器莫过于 unordered_map 了，它内部是由 Hash Table 实现的，它与 hash_map 基本一致，但是 unordered_map 已经被纳入到 C++ 标准当中的，因此在日常使用上，也更推荐使用 unordered_map 而不是 hash_map。

 

与之前一样，先看一下源码：

```
template class _Hash { // hash table -- list with vector of iterators for quick access
protected:
    using _Mylist = list;
 
private:
    _Traits _Traitsobj; // traits to customize behavior
    _Mylist _List; // list of elements, must initialize before _Vec
    _Hash_vec<_Aliter> _Vec; // "vector" of list iterators for buckets:
                             // each bucket is 2 iterators denoting the closed range of elements in the bucket,
                             // or both iterators set to _Unchecked_end() if the bucket is empty.
    size_type _Mask; // the key mask
    size_type _Maxidx; // current maximum key value, must be a power of 2
}; 
```

Hash Table 有两种常见的实现方式：开放定址法和链地址法。stl 中的 unordered_map 中使用的便是链地址法，正如其名，unordered_map 中所有 Value 都是存储在_List 当中的，方便顺序遍历所有节点，_Vec 记录 Hash(Key) 对应的 List Node 位置，用于快速索引到对应的 List Node。

 

我们来先分析下_Hash 的内存布局，_Traitsobj 是一个模板类型，不是很直观，经过我的搜索，内部最主要记录了 load factor，是一个 float 类型，占用 4 个字节，默认情况是 1，_List 和_Vec 和标准 stl 的 List 和 Vector 类型的内存布局一致，_Mask 和_Maxidx 在 32 位下个占用 4 个字节，一般情况下_Mask=_Maxidx-1，这点在后面的反编译代码中也能体现出来。最终计算得出_Hash 总共占用 32 字节（_Traitsobj(4)+_List(8)+_Vec(12)+_Mask(4)+_Maxidx(4)）。

 

下面看一段测试代码：

 

**原始 C++ 代码：**

```
int main(int argc, char* argv[]) {
    std::unordered_map v;
    for (int i = 0; i < argc; ++i) {
        v[i] = i + 1;
    }
    for (auto& i : v) {
        printf("%d, %d\n", i.first, i.second);
    }
    return 0;
} 
```

**IDA 反编译代码：**

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  _DWORD *v3; // eax
  int v4; // eax
  _DWORD *j; // esi
  _DWORD v7[8]; // [esp+Ch] [ebp-34h] BYREF
  int i; // [esp+2Ch] [ebp-14h] BYREF
  int v9; // [esp+3Ch] [ebp-4h]
 
  // 1. 初始化unordered_map
  v7[2] = 0; // 初始化链表的size
  v3 = operator new(0x10u); // 分配链表头的内存
  // 初始化链表头
  *v3 = v3;
  v3[1] = v3;
  v7[1] = v3; // 将链表头赋值给_List成员变量
  v9 = 1;
  v7[3] = 0; // 初始化_Vec._Myfirst
  v7[4] = 0; // 初始化_Vec._Mylast
  v7[5] = 0; // 初始化_Vec._Myend
  v7[6] = 7; // _Mask，一般情况下都是_Maxidx-1
  v7[7] = 8; // _Maxidx，默认是8
  v7[0] = 0x3F800000; // _Traitsobj记录的load factor，float类型，值是1.0
  sub_401400((void **)&v7[3], 0x10u, (int)v3); // unordered_map在初始化阶段会进行rehash操作
 
  // 2. for (int i = 0; i < argc; ++i)
  v4 = 0;
  v9 = 2;
  for ( i = 0; v4 < argc; i = v4 )
  {
    // insert操作比较复杂，没有被inline代码，不是本文关注的重点，不再展开分析内部实现
    *(_DWORD *)sub_401350(&i) = v4 + 1;
    v4 = i + 1;
  }
 
  // 3. for (auto& i : v)
  // 实际上，unordered_map的遍历就是对内部的_List进行顺序遍历
  for ( j = *(_DWORD **)v7[1]; j != (_DWORD *)v7[1]; j = (_DWORD *)*j )
    printf("%d, %d\n", j[2], j[3]);
 
  // 4. 释放unordered_map的内存
  sub_4012C0((int)v7);
  return 0;
}

```

最后看一下在内存中的布局：

```
// _Hash
00A4F934 3F800000 ...? // load factor=float(1)
00A4F938 00EB8470 p.ë. // _List._List_Node
00A4F93C 00000001 .... // _List._Mysize
00A4F940 00EB8488 ..ë. // _Vec._Myfirst
00A4F944 00EB84C8 È.ë. // _Vec._Mylast
00A4F948 00EB84C8 È.ë. // _Vec._Myend
00A4F94C 00000007 .... // _Mask
00A4F950 00000008 .... // _Maskidx

```

[](#三、总结)三、总结
-------------

在实际逆向的过程中，通过 IDA 生成的反编译代码可能要比本文展示的更加复杂，通常都是结构体加上 stl 容器嵌套造成的，因此并没有一种可以识别出任意 stl 容器的方案。对于这类代码的还原，我更倾向于动态静态相结合，根据内存布局初步判断容器类型，然后结合初始化代码进一步验证我们的猜想。

[[注意] 欢迎加入看雪团队！base 上海，招聘 CTF 安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#调试逆向](forum-4-1-1.htm)