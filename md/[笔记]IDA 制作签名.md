> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265330.htm)

1. 目的
-----

识别静态链接且被去除符号的库函数。

2. 原理
-----

IDA 会用签名匹配 IDA 数据库中的函数，如果匹配上会自动重命名函数

 

**注意：只有使用 IDA 哑名的函数才能被自动重命名。换言之，如果你已经对一个函数进行了重命名，而后这个函数与一个签名相匹配，那么，这时 IDA 不会再对这个函数进行重命名。因此，在分析过程中，你应该尽可能早地应用签名**

3. 制作方法
-------

### 3.1 静态链接且被去除符号的可执行文件

为了展示 IDA 签名的效果，我用以下例子：

```
#include #include #include #include int main(int argc,char* argv[]){
    std::map maps;
    std::vector vecs;
    maps[1] = "android";
    maps[2] = "ios";
    maps[3] = "macos";
 
    vecs.push_back("andorid");
    vecs.push_back("ios");
    vecs.push_back("macos");
 
    std::map::const_iterator cit;
    for(cit = maps.begin();cit != maps.end();cit++){
        std::cout << cit->second << " re book published." << std::endl;
    }
 
    std::vector::const_iterator cit2;
    for(cit2 = vecs.begin();cit2 != vecs.end();cit2++){
        std::cout << *cit2 << " 2nd re book published." << std::endl;
    }
 
    return 0;
} 
```

**静态编译**  
`armv7a-linux-androideabi21-clang++ -static app7.cpp -o app7_static`

 

**去除符号**  
`arm-linux-androideabi-strip.exe -x --strip-unneeded app7_static`

 

![](https://bbs.pediy.com/upload/attach/202101/825187_VADXAZMTZ8HYWWC.jpg)

 

**没有签名的 IDA 的识别效果**

 

![](https://bbs.pediy.com/upload/attach/202101/825187_N7G2YBPQGV634BR.jpg)

 

**我们以`libc.a`这个静态库为例制作并应用 IDA 签名**

### 3.2 用到的工具

**工具包下载链接见附件**

 

**工具包中包含的文档资料**

*   readme.txt。这个文件总体概述签名创建过程。
*   plb.txt。这个文件描述静态库解析器`plb.exe`的用法
*   pat.txt。这个文件详细说明了模式文件的格式，它是签名创建过程的第一步。
*   sigmake.txt。这个文件描述`sigmake.exe`文件的用法，该文件用于从模式文件生成. sig 文件。

**工具在工具包中的`bin/平台(linux/mac/win)/`目录下**

 

![](https://bbs.pediy.com/upload/attach/202101/825187_4UF8HX46WCSEAAC.jpg)

### 3.3 制作流程

*   (1) 获得一个你希望为其创建签名文件的静态库。
*   (2) 利用其中一个 FLAIR 解析器为该库创建一个模式文件。
*   (3) 运行 `sigmake.exe` 来处理生成的模式文件，并生成一个签名文件。
*   (4) 将新的签名文件复制到`<IDADIR>/sig`目录中，安装这个文件

#### 3.3.1 获取静态库

*   线索：已经识别的函数名、搜索字符串（string -a 或者 IDA 的字符串窗口）、网络搜索
*   提高准确性：具体的操作系统、操作系统版本、发行版本（`file`命令）

这里我已经知道例子中的`libc.a`是`D:\soft\Android\Sdk\ndk-bundle\toolchains\llvm\prebuilt\windows-x86_64\sysroot\usr\lib\arm-linux-androideabi\21\libc.a`

#### 3.3.2 创建模式文件

*   plb.exe/plb。OMF 库的解析器（ Borland 编译器常用)。
*   pcf.exe/pcf。COFF 库的解析器（微软编译器常用)。
*   pelf.exe/pelf。ELF 库的解析器（许多 Unix 系统常用)。
*   ppsx.exe/ppsx。Sony PlayStation PSX 库的解析器。
*   ptmobj.exe/ptmobj。TriMedia 库的解析器。
*   pomf166.exe/pomf166。Kiel OMF 166 对象文件的解析器。

`libc.a`文件是 elf 格式的，所以用`pelf.exe`制作模式文件  
`pelf.exe libc.a libc.pat`

 

![](https://bbs.pediy.com/upload/attach/202101/825187_NWGEBFST3JB87ZW.jpg)

 

忽略了 93 个函数

 

模式文件是一个文本文件，其中包含提取出的、表示被解析库中的函数的模式 (每行显示一种模式)，如下：

 

![](https://bbs.pediy.com/upload/attach/202101/825187_BYD8DFNEWQZUVQ6.jpg)

 

FLAIR 的`pat.txt`文件说明各个模式的格式。简单地说，模式的第一部分列举了它所代表的函数的初始字节序列，最长为 32 个字节。一些字节因为重定位的入口而有所不同，这些字节将得到 “补偿”, 每个字节以两点显示。如果一个函数短于 32 个字节 (例如前面代码中的`je_malloc_initialized`函数), 用点将模式填充到 64 个字符。除 32 个初始字节外，模式中记录的其他信息专用于提高签名匹配过程的准确性。每个模式行中的其他信息包括由函数的某个部分计算得出的 CRC162 值、函数的字节长度以及函数引用的符号名称列表。一般来说, 引用许多其他符号的函数越长，它生成的模式行就越复杂。在前面生成的`libc.pat`文件中，一些模式行的长度超过了 7000 个字符。

 

IDB_2_PAT：几名第三方程序员开发出了一些实用工具，可用于从现有的 IDA 数据库生成模式。其中一个实用工具为 [IDB_2_PATR](http://www.openrce.org/downloads/details/26/IDB_2_PAT)，这个 IDA 插件由 J.C.Roberts 编写，它能够为现有数据库中的一个或多个函数生成模式。如果你想在其他数据库中遇到与现有数据库中的函数的代码类似的代码, 但却无法访问用于创建被分析的二进制文件的原始库文件，就可以用到这些实用工具。

#### 3.3.3 创建签名文件

创建模式文件后, 创建签名过程的下一个步骤是生成一个适合 IDA 使用的`.sig`文件。IDA 签名文件的格式与模式文件的格式截然不同。签名文件采用一种专用二进制格式，最大限度地减少呈现模式文件中的全部信息所需的空间数量, 并且努力根据具体的数据库内容实现高效的签名匹配。[Hex-Rays 的网站](https://www.hex-rays.com/products/ida/tech/flirt/)宏观介绍了签名文件的结构。

 

制作签名的命令  
`sigmake.exe -n"libc for android arm" libc.pat libc_android_arm.sig`

 

参数`-n`是添加签名注释，用于描述该签名的相关信息

*   执行后的结果
    *   结果一：生成`.sig`文件
    *   结果二：生成`.err和.exc`文件

结果一就不用说了，我们说结果二，出现结果二的截图如下：

 

![](https://bbs.pediy.com/upload/attach/202101/825187_2V6R5Y65QWMQCPQ.jpg)

 

截图说明有 78 个冲突的模式了，冲突的模式意思是有两个及以上的函数生成的模式相同，我们必须解决冲突。如果不能解决冲突，在应用签名的过程中，我们就无法确定函数到底与哪一个签名相匹配。因此，`sigmake`必须能够将每一个生成的签名解析成一个函数名称。否则，如果一个或几个函数的模式完全相同，`sigmake`生成的不是. sig 文件，而是一个排斥文件`.exc`文件。排斥文件是文本文件，它详细说明了`sigmake`在处理模式文件时遇到的冲突。你必须编辑排斥文件，以指导`sigmake`应如何解决任何相互冲突的模式。下面我们将讨论编辑排斥文件的一般过程。

 

`sigmake`生成的所有排斥文件均以下面的代码开头：

```
;--------- (delete these lines to allow sigmake to read this file)
; add '+' at the start of a line to select a module
; add '-' if you are not sure about the selection
; do nothing if you want to exclude all modules

```

这些代码的目的是告诉你如何解决冲突，以成功生成签名。你需要做的头件大事是删除 4 行以分号开头的代码，否则，`sigmake`将无法在随后的执行过程中解析排斥文件。下一步是告诉`sigmake`你希望如何解决冲突。从上面生成的`libc_android_arm.exc`提取的几行代码如下：

```
pthread_barrierattr_destroy                           00 0000 0021016000207047................................................
pthread_condattr_init                                 00 0000 0021016000207047................................................
pthread_mutexattr_init                                00 0000 0021016000207047................................................
pthread_rwlockattr_init                               00 0000 0021016000207047................................................
pthread_barrierattr_init                              00 0000 0021016000207047................................................
 
je_extent_heap_new                                    00 0000 002101607047....................................................
je_extent_avail_new                                   00 0000 002101607047....................................................
 
atol                                                  00 0000 00210A22........................................................
atoll                                                 00 0000 00210A22........................................................
atoi                                                  00 0000 00210A22........................................................

```

这些代码详细说明了 3 个冲突: `pthread_barrierattr_destroy`函数很难与`pthread_condattr_init、pthread_mutexattr_init、pthread_rwlockattr_init、pthread_barrierattr_init`函数区分开, `je_extent_heap_new`的签名与`je_extent_avail_new`相同，`atol`与`atoll、atoi`互相冲突。如果你熟悉其中一些函数，对于上面的结果，你就不会觉得奇怪, 因为相互冲突的函数基本上完全相同。  
为了让你 “掌握自己的命运”，`sigmake`让你仅指定一个函数作为相关签名的匹配函数。任何时候，如果在数据库中发现一个对应的签名，并且你想应用一个函数的名称，那么，你可以在该函数名称前附加一个**加号**; 如果你只想在数据库中添加某个函数的注释，则在该函数名称前附加一个**减号**; 如果在数据库中发现对应的签名时，你不想应用任何名称，那么，你**不需要添加任何符号**。下面的代码为上面提到的 3 个冲突提供了一种可行的解决方案:

```
+pthread_barrierattr_destroy                           00 0000 0021016000207047................................................
pthread_condattr_init                                 00 0000 0021016000207047................................................
pthread_mutexattr_init                                00 0000 0021016000207047................................................
pthread_rwlockattr_init                               00 0000 0021016000207047................................................
pthread_barrierattr_init                              00 0000 0021016000207047................................................
 
je_extent_heap_new                                    00 0000 002101607047....................................................
je_extent_avail_new                                   00 0000 002101607047....................................................
 
-atol                                                  00 0000 00210A22........................................................
atoll                                                 00 0000 00210A22........................................................
atoi                                                  00 0000 00210A22........................................................

```

在这个代码段中，我们决定在数据库中发现第一个签名时，使用函数名`pthread_barrierattr_destroy`; 发现第二个签名时，不做任何处理; 发现第三个签名时，在数据库中添加一段有关`atol`的注释。在解决冲突时，请记住以下要点。

*   为最大限度地减少冲突，请删除排斥文件开头的 4 个注释行。
*   最多只能给冲突函数组中的一个函数附加`+/-`。
*   如果一个冲突函数组仅包含一个函数，不要在该函数前附加`+/-`，让它保持原状即可。
*   `sigmake`连续运行失败会将数据 (包括注释行）附加到现有的任何排斥文件后。在再次运行`sigmake`之前, 你必须删除这些额外的数据, 并更正原始数据 (如果这些数据是正确的, `sigmake`将不会再次运行失败)。

更改排斥文件后，你必须保存这个文件, 并使用你最初使用的命令行参数（如`sigmake.exe -n"libc for android arm" libc.pat libc_android_arm.sig`）重新运行`sigmake`。这一次，`sigmake`应该能够定位和遵照你的排斥文件，并成功生成一个. sig 文件。如果没有显示错误消息，且生成一个. sig 文件，如下所示，即表示`sigmake`操作成功:

 

![](https://bbs.pediy.com/upload/attach/202101/825187_4A69P2RKGA876AT.jpg)

 

![](https://bbs.pediy.com/upload/attach/202101/825187_V8SSBFVMP3JYG4U.jpg)

 

成功生成签名文件后，你需要将它复制到你的`<IDADIR>/sig`目录中，以便 IDA 使用这个文件。随后，你可以通过  
`File->Load File->FLIRT Signature File`访问这个新签名。

 

**需要注意的是，我们有意隐藏了所有可应用于模式生成工具和`sigmake`的选项**。有关可选项的完整列表，请参阅`plb.txt`和 `sigmake.txt`文件。这里我们仅介绍`sigmake`的`-n`选项，它用于在一个生成的签名文件中植入一个描述性的名称。这个名称将在选择签名的过程中显示, 并可在对签名排序时提供极大的帮助。下面的命令行将名称字符串 “libc for android arm” 植入到生成的签名文件中:

 

`sigmake.exe -n"libc for android arm" libc.pat libc_android_arm.sig`

 

![](https://bbs.pediy.com/upload/attach/202101/825187_W93YWM9M63YE8YN.jpg)

 

另外，你还可以使用排斥文件中的指令指定库名称。但是，并不是所有生成签名的过程都会需要排斥文件，因此使用命令行的方法更加有用。欲了解更多详情，请参阅`sigmake.txt`文件。

#### 3.3.4 应用签名文件

**放入签名**  
![](https://bbs.pediy.com/upload/attach/202101/825187_JPZB92VAD3DKY8M.jpg)

 

有`sig\架构`目录就把签名放入`sig\架构`目录，没有就直接放入`sig`目录

 

**应用签名**

*   方法一：你可以通过`File->Load File->FLIRT Signature File`访问这个新签名。
*   方法二：`SHIFT+F5`快捷键打开签名子窗口，然后`Ins`添加签名

![](https://bbs.pediy.com/upload/attach/202101/825187_253P4QCDNKW55PH.jpg)

 

**应用签名后 IDA 的识别效果**

 

![](https://bbs.pediy.com/upload/attach/202101/825187_8PATW2N75E7VUWZ.jpg)

4. 参考
-----

《IDA Pro 权威指南 (第二版)》

5. 附件
-----

工具包下载链接：https://pan.baidu.com/s/1h3x1Bb9uqHdCshXyg9gZ6Q  
提取码：s36f

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 3 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)