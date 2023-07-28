> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [fuzhe1989.github.io](https://fuzhe1989.github.io/2020/01/22/optimizations-in-cpp-compilers/)

> 原文：Optimizations in C++ Compilers 作者：Matt Godbolt 在将上层容易写的代码转换为高效的由计算机去执行的机器码的过程中，编译器必不可少。

> *   原文：[Optimizations in C++ Compilers](https://queue.acm.org/detail.cfm?id=3372264)
> *   作者：[Matt Godbolt](https://www.linkedin.com/in/godbolt/)

在将上层容易写的代码转换为高效的由计算机去执行的机器码的过程中，编译器必不可少。但它们在其中完成的复杂工作却常常被人忽视。你也许会花许多时间来慎重考虑算法和解决错误，但可能没有足够的时间关注编译器能做什么。

本文介绍了一些编译器和代码生产方面的概念，之后着重介绍一些你的编译器为你所做的令人印象深刻的转换工作，以及我最喜欢的优化方式的一些实际例子。希望你能了解编译器可以做哪些优化，以及如何进一步探索该主题。最重要的是，你可能也会爱上看汇编输出，并开始对编译器的工程质量肃然起敬。

本文举的都是 C/C++ 的例子，这是我最有经验的语言。但其中的许多优化方法也适用于其它编译语言。事实上，像 LLVM 这样的前端不可见的编译器工具包的出现意味着多数优化方法都会以相同方式作用在 Rust/Swift/D 语言等语言上。

[](#关于我 "关于我")关于我
-----------------

我一直着迷于编译器能做什么。我曾经花了 10 年去制作一款视频游戏，并力争在相同 CPU 周期数下得到比竞争对手更多的精灵（sprite）、爆炸效果，或更复杂的场景。因此手写汇编和直接看汇编输出就成了我的基本技能。

5 年后，我当时在一家交易公司工作，精灵和多边形换成了快速处理金融数据。和以前一样，知道编译器对代码的处理有助于了解我们如何去写代码。

显然，写得好的，容易测的代码非常重要——尤其是如果这些代码可能一秒完成几千笔金融交易。跑得快很重要，但不出 bug 更重要。

2012 年时，我们在讨论可以把 C++11 的哪个新特性加入可接受的编码实践标准。当每一纳秒都很重要时，最好能给出不伤害性能的编码建议。在试验如何使用如`auto`、lambda、`range-for`时，我写了一个 shell 来持续编译并显示过滤后的输出：

```
$ g++ /tmp/test.cc -O2 -c -S -o - -masm=intel \
    | c++filt \
    | grep -vE '\s+\.'
```

事实证明，这个脚本对于回答所有的 “what if” 问题都很有用，我当天晚上回家就写了 Compiler Explorer。

这些年我一直惊讶于编译器为了将我们的代码转换为如艺术品般的汇编指令所做的工作。为了了解编译器做的事情，我建议所有用编译语言的程序员都学一点汇编语言。即使你自己不写，能读懂汇编也很有用。

本文中的所有汇编代码都是针对 X86-64 处理器的，这是我最熟悉的 CPU，也是最常见的架构之一。一些例子只用于 X86, 但事实上许多优化方法很容易应用到其它架构上。另外，我只用到了 GCC 和 Clang 两种编译器，但 Microsoft Visual Studio 和 Intel 的编译器也有同样聪明的优化方法。

[](#优化入门 "优化入门")优化入门
--------------------

不需要深入编译器的优化，只了解一些编译器会用到的概念就很有用。

许多优化方法属于**强度降低（strenth reduction）**的范畴：将昂贵的操作转换为代价更低的操作。一个非常简单的例子是在循环中对循环变量使用乘法：

```
for (int i = 0; i < 100; ++i)
{
    func(i * 1234);
}
```

即使是今天的 CPU，乘法仍然要比更简单的算术运算要慢一些。因此编译器会将循环重写为：

```
for (int iTimes1234 = 0; iTimes1234 < 100 * 1234; i += 1234)
{
    func(iTimes1234);
}
```

在这里强度降低法将使用了乘法的循环转换为了只用加法的循环。

后面的实际示例将会显示更多的强度降低方法。

另一个关键优化是**内联（inlining）**，即编译器将函数调用替换为函数体本身。它可以去掉调用的开销，因为编译器能将组合在一起的代码优化为一个编译单元，经常还能解锁进一步的优化。稍后你会看到大量这方面的例子。

其它优化类别包括：

*   常量折叠（constant folding）。编译器将编译期能计算为常量的表达式直接替换为计算结果。
*   常量传播（constant propagation）。编译器追踪到一个值的源头，发现它是常量后，会将所有地方出现的这个值替换为常量。
*   公共子表达式消除（common subexpression elimination）。将重复的计算过程重写掉，只算一次，其它地方复制结果。
*   移除死代码（dead code removal）。用许多其它方法优化后，可能有些代码对输出不产生影响，就可以移除这些代码。这里包含了对没用到的值的读写操作，以及完全没用到的整个函数或表达式。
*   指令选择（instruction selection）。这个不算是通常意义的优化，但既然编译器会将程序转换为它的内部表示形式，并生成 CPU 指令，编译器通常有一个庞大的等效指令序列的集合可供选择。编译需要知道目标处理器架构的细节以作出正确选择。
*   移动循环中的不变代码（loop invariant code movement）。编译器能识别一块代码在循环过程中值不变，并将这块代码移出循环。其于此，编译器还能将循环中不变的条件检查移出循环外，再将循环体复制两次：一次针对条件为真，一次针对条件为假。之后还能做进一步优化。
*   窥孔优化（peephole optimization）。编译器取一小段指令序列并做局部优化。
*   尾调用移除（tail call removal）。一个在结尾处调用自身的递归函数通常可被重写为循环，从而降低函数调用开销，并减小栈溢出的可能。

帮助编译器进行优化的要点就是保证它能获得尽可能多地信息，从而做出正确的优化决定。其中一个信息源就是你的代码：编译器能看到的代码越多，能做的决定越优。另一个信息源是你用的编译器配置：告诉编译器准确的目标 CPU 架构就能带来大不同。当然，编译器拥有的信息越多，编译时间越长，因此这里还要保持平衡。

我们看个例子，统计一个`vector`中通过测试的元素个数（GCC -O3 编译，[godbolt](https://godbolt.org/z/acm19_count1)）：

```
int count(const vector<int> &vec)
{
    int numPassed = 0;
    for (size_t i = 0; i < vec.size(); ++i)
    {
        if (testFunc(vec[i]))
            numPassed++;
    }
    return numPassed;
}
```

如果编译器对`testFunc`一无所知，它会产生这样的内循环：

```
.L4:
  mov edi, DWORD PTR [rdx+rbx*4] ; read rbx'th element of vec
                                 ; (inlined vector::operator [])
  call testFunc(int)             ; call test function
  mov rdx, QWORD PTR [rbp+0]     ; reread vector base pointer
  cmp al, 1                      ; was the result of test true?
  mov rax, QWORD PTR [rbp+8]     ; reread the vector end pointer
  sbb r12d, -1                   ; add 1 if true, 0 if false
  inc rbx                        ; increment loop counter
  sub rax, rdx                   ; subtract end from begin...
  sar rax, 2                     ; and divide by 4 to get size()
                                 ; (inlined vector::size())
  cmp rbx, rax                   ; does loop counter equal size()?
  jb .L4                         ; loop if not
```

为了理解这段代码，知道`std::vector`包含一些指针会很有用：一个指向数据的开始，一个指向数据的结尾，一个指定当前分配的存储空间的结尾。

```
template<typename T> struct _Vector_impl {
  T *_M_start;
  T *_M_finish;
  T *_M_end_of_storage;
};
```

vector 不直接存储它的大小，而是依赖`begin()`和`end()`的差值计算得到。注意`vector<>::size()`和`vector<>::operator[]`已经被彻底内联掉了。

在上面的汇编代码中，`ebp`指向 vector 对象，`begin()`和`end()`指针因此分别是`QWORD PTR [rbp+0]`和`QWORD PTR [rpb+8]`。

另一个编译器用到的技巧是移除分支：你也许有理由地期望`if (testFunc(...))`会变成比较和分支。这里编译器会用`cmp al, 1`进行比较，如果`testFunc()`返回`false`，`cmp`指令会设置 CPU 的进位标志，否则清除进位标志。之后`sbb r12d, -1`指令会带借位地减 - 1。减法等效于进位，也会用到进位标志。这会产生编译器想要的副作用：如果进位标志被清除了（`testFunc()`返回了`true`），它就会减 - 1, 相当于加 1；如果进位标志被设置了，它会减 - 1 再加 1，不改变原值。在一些 CPU 不好预测分支的情况下，避免分支会有帮助。

编译器每次循环都会重新载入`begin()`和`end()`指针，这可能令人惊讶，事实上它每次还会重新去拿`size()`。但编译器必须这么做：它不知道`testFunc()`会做什么，必须假设最坏情况。也就是，它必须假设调用`testFunc()`可能导致`vec`被修改。因为以下原因，这里`const`引用不会开启进一步的优化：`testFunc()`可能持有`vec`的非`const`引用，或者`testFunc()`会使用`const_cast`。

但如果编译器能看到`testFunc()`的函数体，因此得知它不会修改`vec`，故事就很不一样了（[godbolt](https://godbolt.org/z/acm19_count2)）：

```
.L6:
  mov edi, DWORD PTR [rdx]  ; read next value
  call testFunc(int)        ; call testFunc with it
  cmp al, 1                 ; check return code
  sbb r8d, -1               ; add 1 if true, 0 otherwise
  add rdx, 4                ; move to next element
  cmp rcx, rdx              ; have we hit the end?
  jne .L6                   ; loop if not
```

此时编译器已经知道了`vector`的`begin()`和`end()`在循环过程中是不变的。它也因此知道了`size()`的值是不变的。因此编译器可以将这些常量的计算移出循环，再将索引操作（`vec[i]`）重写为从`begin()`开始，每次移动一个`int`，直到`end()`的指针遍历。这极大简化了生成的汇编代码。

这个例子中我提供了一个`testFunc`函数，但将其标记为不可内联（GNU 扩展）来单独展示这一优化。在更实际的代码库中，如果编译器觉得有好处，它是可以内联掉`testFunc`的。

另一个不暴露函数体就能启用这一优化的方法是标记`testFunc`为`[[gnu:pure]]`（另一个语言扩展）。它是向编译器保证这是个纯函数——功能只与它的参数有关，不带任何副作用。

有趣的是，第一个例子中如果我们使用`range-for`，编译器就算不知道`testFunc`会不会修改`vec`，也会生成优化版本的汇编代码（[godbolt](https://godbolt.org/z/acm19_count3)）。这是因为`range-for`被定义为了将`begin()`和`end()`保存到局部变量的代码变换：

```
for (auto val : vec)
{
    if (testFunc(val))
        numPassed++;
}
```

会被解释为：

```
{
    auto __begin = begin(vec);
    auto __end == end(vec);
    for (auto __it = __begin; __it != __end; ++__it)
    {
        if (testFunc(*__it))
            numPassed++;
    }
}
```

考虑各种因素，如果你需要裸写循环，推荐使用现代的`range-for`：它在编译器看不到函数体时也能生成优化代码，且更清晰。但更好的方法是用 STL 的`count_if`完成所有工作：编译器也会生成优化代码（[godbolt](https://godbolt.org/z/acm19_count4)）。

在传统的一次一个编译单元的编译模型下，函数调用处通常看不到函数体，只能看到函数声明。LTO（链接时优化，也称作 LTCG，链接时代码生成）允许编译器看到跨编译单元的代码。在 LTO 中，单个编译单元会被编译为中间代码，而不是机器码。在链接时——整个程序（或动态链接库）都可见时——再去生成机器码。编译器可以利用这点跨编译单元内联，或至少能知道被调用的函数有没有副作用，从而进行优化。

通常在需要优化的构建中启用 LTO 是不错的选择，这样编译器就能看到整个程序了。我现在依赖于 LTO 将更多的函数体移出头文件，从而减少耦合程序、编译时间、debug 构建和测试中的依赖，且还能在最终构建产物中达到想要的性能。

尽管 LTO 已经是很成熟的技术了（我在 21 世纪初的 XBOX 上就用过了 LTCG），我仍然惊讶于只有很少的项目使用了 LTO。部分原因可能是程序员无意间依赖了编译器的未定义行为，这种行为（优化）只有在编译器有更高的可见性（看到更多代码）时才会变得更明显：我知道我犯了这样的错误。

[](#我最喜欢的优化示例 "我最喜欢的优化示例")我最喜欢的优化示例
-----------------------------------

过去这些年我收集了不少有趣的真实世界中的优化示例，既有来自我自己代码的第一手经验，也有来自在 Compiler Explorer 上帮助其他人理解代码的经验。下面是我最喜欢的，展示了编译器有多聪明的几个例子。

### [](#除数为常量的整数除法 "除数为常量的整数除法")除数为常量的整数除法

（直到最近）我们可能惊讶于整数除法是现代 CPU 能执行的最昂贵的操作。除法比加法慢 50 倍以上，比乘法慢 10 倍以上。（这一说法直到 Intel 的 Cannon Lake 之前都成立，Cannon Lake 将 64 位整数除法的最大延时从 96 个周期降为了 18 个周期。这样除法就只比加法慢 20 倍，比乘法慢 5 倍。）

庆幸的是，当除数为常量时，编译器作者有一些降低强度的技巧。我相信大家都知道当除数为 2 的整数次幂时，除法可以替换为逻辑右移——请放心，编译器会帮你做的。我建议不要在代码里写`<<`做除法；让编译器帮你做。这样会更清晰，编译器也知道怎么正确处理有符号数：整数除法朝 0 的方向截断，而负数自身移位会朝着负无穷的方向截断。

但是，如果你除的不是 2 的整数幂呢？你会失去运气吗？

```
unsigned divideByThree(unsigned x)
{
    return x / 3;
}
```

幸运的是编译器又一次站在了你身后。这段代码被编译为（[godbolt](https://godbolt.org/z/acm19_div3)）：

```
divideByThree(unsigned int):
  mov eax, edi          ; eax = edi
  mov edi, 2863311531   ; edi = 0xaaaaaaab
  imul rax, rdi         ; rax = rax * 0xaaaaaaab
  shr rax, 33           ; rax >>= 33
  ret
```

其中看不到除法指令。只是一次移位，以及乘一个奇怪的巨大的常数：输入的 32 位无符号整数乘上`0xaaaaaaab`，结果是一个 64 位整数，再右移 33 位。编译器将除法替换为了更廉价的定点乘法逆运算。这里的定点是 33 位，常数是这种形式下的 1/3（实际是 0.33333333337213844）。编译器有种算法来决定合适的定点和常数值，同时在输入范围内以相同的精度保留与真正的除法运算相同的四舍五入。有时这需要一些额外的运算——例如除以 1023（[godbolt](https://godbolt.org/z/acm19_div1023)）：

```
divideBy1023(unsigned int):
  mov eax, edi
  imul rax, rax, 4198405
  shr rax, 32
  sub edi, eax
  shr edi
  add eax, edi
  shr eax, 9
  ret
```

该算法广为人知，在《[Hacker’s Delight](https://book.douban.com/subject/1784887/)》中有大量记录。

简而言之，你可以依靠编译器通过编译期已知的常量来很好地优化除法。

你可能在想：这为什么是如此重要的优化方法？我们执行除法的频率是多少？它不光与除法本身有关，还与相关的取余操作有关，后者常被用于 hash-map 实现中将 hash 值映射到 hash 桶数范围的操作中。

知道这里编译器能做什么可以通往有趣的 hash-map 实现。一种方法是使用固定数量的桶以允许编译器产生完美的不使用昂贵的除法指令的取余。

大多数 hash-map 支持 rehash 到不同数量的桶。朴素的实现会用运行期才知道的数字去取余，导致编译器只能用慢的除法指令。事实上 gcc 的 libstdc++ 实现中的`std::unordered_map`就是这么做的。

Clang 的 libc++ 往前走了一步：它会检查桶的数量是否是 2 的幂，如果是的话就跳过除法指令，转而使用逻辑与。桶数量是 2 的幂的想法很诱人，因为它使模运算变快了，但它要依靠好的 hash 函数实现来避免频繁冲突。而质数个桶可以在非常简单的 hash 函数时也能很好地避免冲突。

诸如`boost::multi_index`这样的库又往前走了一步：与其保存实际的桶数，不如使用固定的质数作为桶数。

```
size_t reduce(size_t hash, int bucketCountIndex) {
    switch (tableSizeIndex)
    {
        case 0: return hash % 7;
        case 1: return hash % 17;
        case 2: return hash % 37;
        
    }
}
```

这样编译器对于所有可能的 hash-map 大小都能产生完美的取余代码，仅有的额外开销就是`switch`中的分派代码。

gcc9 有一个技巧来检查是否可被非 2 的幂整除（[godbolt](https://godbolt.org/z/acm19_multof3)）：

```
bool divisibleBy3(unsigned x)
{          
    return x % 3 == 0;
}
```

会被编译为：

```
divisibleBy3(unsigned int):
  imul edi, edi, -1431655765    ; edi = edi * 0xaaaaaaab
  cmp edi, 1431655765 ; compare with 0x55555555
  setbe al                      ; return 1 if edi <= 0x55555555
  ret
```

Daniel Lemire 的[博客](https://lemire.me/blog/2019/02/08/faster-remainders-when-the-divisor-is-a-constant-beating-compilers-and-libdivide/)中很好地解释了这种表面上的巫术。另外，运行时也有可能用到这些整数除法的技巧。如果你需要用相同的除数去除很多数字，你可以用像 [`libdivide`](https://libdivide.com/)这样的库。

### [](#统计为1的位数 "统计为1的位数")统计为 1 的位数

有多少次你想知道，一个整数中有多少位是 1？也许没那么频繁。但事实证明，这种简单的操作在许多情况下非常有用。例如，计算两个位集合的 hamming 距离，处理稀疏矩阵的紧凑表示，或处理向量运算的结果。

你可能会写这样的函数来统计 1：

```
int countSetBits(unsigned a)
{
    int count = 0;
    while (a != 0)
    {
        count++;
        a &= (a - 1); 
    }
    return count;
}
```

值得注意的是其中的位运算技巧`a &= (a - 1)`，它会清除最低位的 1。在纸上证明这一点很有意思，试一下吧。

目标架构是 Haswell 时，gcc8.2 会产生这样的汇编（[godbolt](https://godbolt.org/z/acm19_bits)）：

```
countSetBits(unsigned int):
  xor eax, eax      ; count = 0
  test edi, edi     ; is a == 0?
  je .L4            ; if so, return
.L3:
  inc eax           ; count ++
  blsr edi, edi     ; a &= (a - 1);
  jne .L3           ; jump back to L3 if a != 0
  ret  
.L4:
  Ret
```

请注意 gcc 如何巧妙地找到了`BLSR`指令来去掉最低位的 1。很干净，但还是不如 Clang7.0 聪明：

```
countSetBits(unsigned int):
  popcnt eax, edi     ; count = number of set bits in a
  ret
```

这个操作足够通用，大多数 CPU 都有一条指令可以一次完成：`POPCNT`（population count）。Clang 聪明到将 C++ 中的整个循环简化为一条指令。这是良好的指令选择的非常棒的例子：Clang 的代码生成器认出了这个模式，并能选出最好的指令。

前面我对 gcc 有点不公平，gcc9 也实现了这种方法，但还有点区别：

```
countSetBits(unsigned int):
  xor eax, eax          ; count = 0
  popcnt eax, edi       ; count = number of set bits in a
  ret
```

第一眼看上去不够优化：为什么要写一个马上被`POPCNT`指令的返回值覆盖的 0 呢？

简单研究之后，我们找到了 Intel CPU 的勘误 SKL029：“`POPCNT`指令的执行时间可能比预期要长”——这是 CPU 的 bug！尽管`POPCNT`指令的输出会完全覆盖`eax`寄存器，它被错误地标记为依赖于`eax`之前的值。这会限制 CPU 将`POPCNT`指令调度到它前面的对`eax`写操作完成后执行——尽管它们完全没关系。

gcc 的解法是破除对`eax`的依赖：CPU 将`xor eax, eax`视作打破依赖的惯用法。不会有`POPCNT`之前指令可以在`xor eax, eax`之后还影响到`eax`的值了，因此`POPCNT`可以在它的输入`edi`准备好后立即执行。

这只会影响 Intel 的 CPU，而且看起来在 Cannon Lake 中已经修复了，但 gcc 在目标为 Cannon Lake 时仍然会产生`xor`指令。

### [](#链式条件 "链式条件")链式条件

也许你从未需要统计一个整数中 1 的数量，但你也许写过这样的代码：

```
bool isWhitespace(char c)
{
    return c == ' '
      || c == '\r'
      || c == '\n'
      || c == '\t';
}
```

我本能地以为生成的代码会充满比较和分支，但 Clang 和 gcc 都用了一个技巧令这段代码非常高效。下面是 gcc9.1 的输出（[godbolt](https://godbolt.org/z/acm19_conds)）：

```
isWhitespace(char):
  xor eax, eax              ; result = false
  cmp dil, 32               ; is c > 32
  ja .L4                    ; if so, exit with false
  movabs rax, 4294977024    ; rax = 0x100002600
  shrx rax, rax, rdi        ; rax >>= c
  and eax, 1                ; result = rax & 1
.L4:
  ret
```

编译器将一系列比较转换为了查表。加载到`rax`中的魔数是一个 33 位的查找表，表中为 1 的位置是你需要返回`true`的情况（下标为 32、13、10、9，分别对应 、`\r`、`\n`、`\t`）。之后移位和`&`就可以取到第`c`位并返回。Clang 生成的代码与之有细微差别，但大体上等价。这是另一个强度降低的例子。

我被这种优化惊到了。在使用 Compiler Explorer 调查问题之前，我会假设我比编译器更懂，因此会手写这样的代码。

但在试验时我发现一件不幸的事（至少对于 gcc）：比较的顺序可以影响编译器能不能做这种优化。如果你交换了`\r`和`\n`的顺序，gcc 会生成如下代码：

```
isWhitespace(char):
  cmp dil, 32   ; is c == 32?
  sete al       ; al = 1 if so, else 0
  cmp dil, 10   ; is c == 10?
  sete dl       ; dl = 1 if so, else 0
  or al, dl     ; al |= dl
  jne .L3       ; if al is non-zero return it (c was ` ` or `\n`)
  and edi, -5   ; clear bit 2 (the only bit that differs between
                ;              `\r` and `\t`)
  cmp dil, 9    ; compare with `\t`
  sete al       ; dl = 1 if so, else 0
.L3:
  ret
```

用`and`将对`\r`和`\n`的比较合并到一起绝对是非常巧妙的，但看起来它会导致生成比之前的例子更差的代码。[Quick Bench 上的一个简化测试](http://quick-bench.com/0TbNkJr6KkEXyy6ixHn3ObBEi4w)表明，在可预测的紧凑循环中，基于比较的版本可能会稍快一点点。是谁说这东西简单的？

### [](#求和 "求和")求和

有时你需要将一堆东西加起来。编译器非常擅长利用大多数现代 CPU 都支持的向量指令来加速求和，因此下面这段非常直接的代码：

```
int sumSquared(const vector<int> &v)
{
    int res = 0;
    for (auto i : v)
    {
        res += i * i;
    }
    return res;
}
```

转化后的核心循环长这样（[godbolt](https://godbolt.org/z/acm19_sum)）：

```
.loop:
  vmovdqu ymm2, YMMWORD PTR [rax]   ; read 32 bytes into ymm2
  add rax, 32                       ; advance to the next element
  vpmulld ymm0, ymm2, ymm2          ; square ymm2, treating as
                                    ;   8 32-bit values
  vpaddd ymm1, ymm1, ymm0           ; add to sub-totals
  cmp rax, rdx                      ; have we reached the end?
  jne .loop                         ; if not, keep looping
```

通过将总和分成 8 个部分和，编译器每条指令能处理 8 个值。最后它再将所有部分和汇总为最终的总和。这相当于把代码重写成这样：

```
int res_[] = {0,0,0,0,0,0,0,0};
for (; index < v.size(); index += 8)
{
    
    
    
    for (size_t j = 0; j < 8; ++j)
    {
        auto val = v[index + j];
        res_[j] += val * val;
    }
}
res = res_[0] + res_[1]
    + res_[2] + res_[3]
    + res_[4] + res_[5]
    + res_[6] + res_[7];
```

只要简单地将编译器的优化级别设置得足够高，并设置合适的目标 CPU 架构，向量化就自己完成了。太棒了！

这要依赖于一个事实，将总和分成若干个部分和，最终再加起来，等效于按顺序累加。显然对于整数这是对的，但对于浮点数就不一定了。浮点数是不可结合的：`(a+b)+c`不等价于`a+(b+c)`，因为浮点加法的结果精度依赖于两个输入的相对量级。

这就意味着，很不幸，将`vector<int>`改为`vector<float>`得不到你想要的代码。编译器可以用一些向量指令（它可以一次算 8 个值的平方），但必须按顺序累加这些值（[godbolt](https://godbolt.org/z/acm19_sumf)）：

```
.loop:
  vmovups ymm4, YMMWORD PTR [rax]   ; read 32 bytes into ymm4
  add rax, 32                       ; advance
  vmulps ymm1, ymm4, ymm4           ; square 8 floats
                                    ; (the one parallel operation)
  vaddss xmm0, xmm0, xmm1           ; accumulate the first value
  vshufps xmm3, xmm1, xmm1, 85      ; shuffle things around
                                    ; (permutes the 8 floats
                                    ;  within the register)
  vshufps xmm2, xmm1, xmm1, 255     ; ...
  vaddss xmm0, xmm0, xmm3           ; accumulate the second value
  vunpckhps xmm3, xmm1, xmm1        ; more shuffling
  vextractf128 xmm1, ymm1, 0x1      ; ...
  vaddss xmm0, xmm0, xmm3           ; accumulate third...
  vaddss xmm0, xmm0, xmm2           ; and fourth value
  vshufps xmm2, xmm1, xmm1, 85      ; shuffling
  vaddss xmm0, xmm0, xmm1           ; accumulate fifth
  vaddss xmm0, xmm0, xmm2           ; and sixth
  vunpckhps xmm2, xmm1, xmm1        ; shuffle some more...
  vshufps xmm1, xmm1, xmm1, 255     ; ...
  vaddss xmm0, xmm0, xmm2           ; accumulate the seventh
  vaddss xmm0, xmm0, xmm1           ; and final value
  cmp rax, rcx                      ; are we done?
  jne .loop                         ; if not, keep going
```

不幸的是还没有简单的方法绕过这个限制。如果你保证这种情况下加法的顺序不重要，你可以启用 gcc 的一个危险的（但名字很有趣）标志：`-funsafe-math-optimizations`。这样 gcc 就能生成漂亮的内循环了（[godbolt](https://godbolt.org/z/acm19_sumf_unsafe)）：

```
.loop:
  vmovups ymm2, YMMWORD PTR [rax]   ; read 8 floats
  add rax, 32                       ; advance
  vfmadd231ps ymm0, ymm2, ymm2      ; for the 8 floats:
                                    ;   ymm0 += ymm2 * ymm2
  cmp rax, rcx                      ; are we done?
  jne .loop                         ; if not, keep going
```

令人吃惊：一次处理 8 个浮点数，用一条指令完成累加和平方。缺点是可能有无上限的精度损失。另外 gcc 不允许你只对你需要的函数打开这个功能——它是编译单元粒度的标志。Clang 至少允许你在代码中用`#pragma Clang fp contract`来控制开关。

在尝试这些优化时，我发现编译器还有更多的花招：

```
int sumToX(int x)
{
    int result = 0;
    for (int i = 0; i < x; ++i)
    {
        result += i;
    }
    return result;
}
```

gcc 会很直接地翻译这些代码，配上合适的设置后它就会像上面一样用上向量指令。而 Clang 会生成下面这样的代码（[godbolt](https://godbolt.org/z/acm19_sum_up)）：

```
sumToX(int): # @sumToX(int)
  test edi, edi             ; test x
  jle .zeroOrBelow          ; skip if x <= 0
  lea eax, [rdi - 1]        ; eax = x - 1
  lea ecx, [rdi - 2]        ; ecx = x - 2
  imul rcx, rax             ; rcx = ecx * eax
  shr rcx                   ; rcx >>= 1
  lea eax, [rcx + rdi]      ; eax = rcx + x
  add eax, -1               ; return eax - 1
  ret                      
.zeroOrBelow:
  xor eax, eax              ; answer is zero
  ret
```

首先，请注意这里完全没有循环。通过生成的代码，你发现 Clang 返回了：

```
(x-1) * (x-2) / 2 + x - 1
```

它将循环换成了封闭形式的通用求和解法。这种解法与我自己会写出来的朴素代码不同：

```
x * (x - 1) / 2
```

这大概是 Clang 使用的通用算法的结果。

进一步的试验显示 Clang 聪明到能优化很多种类似的循环。Clang 和 gcc 追踪循环变量的方式都能做这类优化，但只有 Clang 选择生成这种封闭形式的代码。但它不保证总是降低工作量：对于很小的`x`，封闭形式的开销也许比直接循环要大。Krister Walfridsson 在[他的博客](https://kristerw.blogspot.com/2019/04/how-llvm-optimizes-geometric-sums.html)中详细介绍了如何实现这种优化。

同样值得注意的是，为了做这种优化，编译器可能要依赖于 “有符号整数溢出是未定义行为”。这样它就能假设你的代码不会传入可能会使结果溢出（这个例子中是 65536）的`x`。如果 Clang 不能做这个假设，有时候它没办法找到封闭形式的解（[godbolt](https://godbolt.org/z/acm19_sum_fail)）。

### [](#去虚拟化 "去虚拟化")去虚拟化

尽管传统的基于虚函数的多态看起来有点过气了，但它仍然有一定的市场。无论是需要真正的多态行为，还是要为可测性增加 “接缝”，或是允许未来的扩展，基于虚函数的多态都是不错的选择。

但如我们所知，虚函数很慢。是不是呢？我们看它们是怎么影响前面的平方和例子吧——有这样的代码：

```
struct Transform
{
    int operator()(int x) const { return x * x; }
};
 
int sumTransformed(const vector<int> &v,
                   const Transform &transform)
{
    int res = 0;
    for (auto i : v)
    {
        res += transform(i);
    }
    return res;
}
```

显然现在它还没有多态。快速用编译器跑一下可以看到它生成了相同的高度向量化的汇编（[godbolt](https://godbolt.org/z/acm19_poly1)）。

现在我们为`int operator()`加上`virtual`，就会得到一个慢得多的实现，被填进了间接调用，对吧？当然，有点（[godbolt](https://godbolt.org/z/acm19_poly2)）。生成的代码要比之前更多，但核心循环可能会让你想不到：

```
; rdx points to the vtable
.L8:
  mov rax, QWORD PTR [rdx]  ; read the virtual function pointer
  mov esi, DWORD PTR [rbx]  ; read the next int element
  ; compare the function pointer with the address of the only
  ; known implementation...
  cmp rax, Transform::operator()(int) const
  jne .L5                   ; if it's not the only known impl,
                            ; then jump off to a more complex case
  imul esi, esi             ; square the number
  add rbx, 4                ; move to next
  add r12d, esi             ; accumulate the square
  cmp rbp, rbx              ; finished?
  jne .L8                   ; if not, loop
```

这里 gcc 赌了一把。已知它只看到了`Transform`的一个实现，这里用到的很可能就是这个实现。相比无脑通过虚表间接跳转，将虚表指针与已知的唯一实现做比较只需要一点点时间。如果相同，编译器就知道该做什么了：它会内联掉`Transform::operator()`的函数体，并原地平方。

是的：编译器内联掉了一个虚函数调用。棒极了，我第一次发现这个的时候非常吃惊。这种优化叫做推测性去虚拟化（speculative devirtualization），是编译器作者不断研究和改进的源泉。编译器也能在 LTO 时做去虚拟化，能在整个程序范围内确定可能的函数实现。

但编译器漏掉了一个技巧。注意到每次循环入口它都重新从虚表中载入虚函数指针。如果编译器能发现这个值在被调函数不会修改`Transform`的动态类型时保持不变，这次检查就可以移出循环，这样在循环内就完全没有动态检查了。编译器可以用移动循环不变量的方法将虚表检查移出循环。此时其它优化方法就可以介入了，在虚表检查通过时，整段代码可以替换为之前的向量化循环。

你可能以为对象的动态类型不可能变化，但这是标准允许的：对象可以对自身调用 placement new，析构时再变回原来的类型。但建议你别这么做。Clang 有选项承诺你不会这么做：`-fstrict-vtable-pointers`。

在我用的编译器中，gcc 是仅有的这么做的一个，但 Clang 正在重构它的类型系统，从而更多利用上这类优化。

C++11 增加了`final`限定符以允许标记类和虚函数不可重写。这就给了编译器更多的关于哪些方法能受益于这类优化的信息了，在某些情况下甚至允许编译器完全避免虚函数调用（[godbolt](https://godbolt.org/z/acm19_poly3)）。即使没有`final`，有时分析阶段也能证明代码中用到的是特定的具体类（[godbolt](https://godbolt.org/z/acm19_poly4)）。这类静态去虚拟化操作能带来明显的性能提升。

[](#结论 "结论")结论
--------------

希望在读完本文以后，你能欣赏编译器为确保生成高效代码所付出的努力。我希望其中一些优化能让你感到惊喜，帮助你决定写出清晰的、意图明显的代码，将优化工作留给编译器去做。我再次强调，编译器知道的越多，它能做得越好。这包括允许编译器一次看到更多代码，以及将你的目标平台信息交给编译器。在给编译器更多信息时你要做一些权衡：这会让编译更慢。LTO 之类的优化能让你兼顾两者。

编译器中的优化一直在提高，即将到来的间接调用和虚函数分派上的提高也许很快带来更快的多态。我为编译器优化技术的未来而感到兴奋。快去看看你的编译器的输出吧。

### [](#致谢 "致谢")致谢

The author would like to extend his thanks to Matt Hellige, Robert Douglas, and Samy Al Bahra, who gave feedback on drafts of this article.

### [](#引用 "引用")引用

1.  Godbolt, M. 2012. Compiler explorer; [https://godbolt.org/](https://godbolt.org/).
2.  Lemire, D. 2019. Faster remainders when the divisor is a constant: beating compilers and libdivide. [https://lemire.me/blog/2019/02/08/faster-remainders-when-the-divisor-is-a-constant-beating-compilers-and-libdivide/](https://lemire.me/blog/2019/02/08/faster-remainders-when-the-divisor-is-a-constant-beating-compilers-and-libdivide/).
3.  LLVM. 2003. The LLVM compiler infrastructure.; [https://llvm.org](https://llvm.org/).
4.  Padlewski, P. 2018. RFC: Devirtualization v2. LLVM; [http://lists.llvm.org/pipermail/llvm-dev/2018-March/121931.html](http://lists.llvm.org/pipermail/llvm-dev/2018-March/121931.html).
5.  ridiculous_fish. 2010. Libdivide; [https://libdivide.com/](https://libdivide.com/).
6.  Uops. Uops.info Instruction Latency Tables; [https://uops.info/table.html](https://uops.info/table.html).
7.  Walfridsson, K. 2019. How LLVM optimizes power sums; [https://kristerw.blogspot.com/2019/04/how-llvm-optimizes-geometric-sums.html](https://kristerw.blogspot.com/2019/04/how-llvm-optimizes-geometric-sums.html).
8.  Warren, H. S. 2012. Hacker’s Delight. 2nd edition. Addison-Wesley Professional.

### [](#相关文章 "相关文章")相关文章

*   [C Is Not a Low-level Language](https://queue.acm.org/detail.cfm?id=3212479) Your computer is not a fast PDP-11. - David Chisnall
*   [Uninitialized Reads](https://queue.acm.org/detail.cfm?id=3041020) Understanding the proposed revisions to the C language - Robert C. Seacord
*   [You Don’t Know Jack about Shared Variables or Memory Models](https://queue.acm.org/detail.cfm?id=2088916) Data races are evil. - Hans-J. Boehm, Sarita V. Adve