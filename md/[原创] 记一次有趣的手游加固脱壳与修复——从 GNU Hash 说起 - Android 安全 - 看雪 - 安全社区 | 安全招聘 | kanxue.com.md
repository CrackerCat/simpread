> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277443.htm)

> [原创] 记一次有趣的手游加固脱壳与修复——从 GNU Hash 说起

[原创] 记一次有趣的手游加固脱壳与修复——从 GNU Hash 说起

8 小时前 401

### [原创] 记一次有趣的手游加固脱壳与修复——从 GNU Hash 说起

 [![](http://passport.kanxue.com/upload/avatar/365/872365.png?1670222690)](user-home-872365.htm) [乐子人](user-home-872365.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 8 小时前  401

记一次有趣的手游加固脱壳与修复——从 GNU Hash 说起
==============================

0. 前言
-----

起因是这样的，前两天一位老哥问我对某 Guard 的保护有啥方法，

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_MFRGJDK4RWTCPJ4.jpg)

 

是一个捕鱼游戏。刚好小弟我很早就对这家保护感兴趣了，因为其介绍是 “业界独创的无导入函数 so 加固”，一直想研究但是又没有碰见使用的游戏，所以就赶快要了样本拿来研究一番。看了两天基本把加固原理搞得差不多了，也脱壳修复完成，比之前的某盾在自定义 linker 上确实更有新意，于是想写篇文章总结一下。  
![](https://bbs.kanxue.com/upload/attach/202306/872365_U2RRJJ4QDFU43K8.png)

1. 概览
-----

拿到样本，先用 jadx 看了一下 java 层，没什么加固。然后去看了一下 lib 目录，发现了

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_7BJZHVM8B28YDQB.jpg)

 

libF*Guard.so 特征模块。

 

ida 打开后，发现主体部分全被加密了，只保留了壳的一小部分代码：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_3M7G6DW43P4WVHQ.png)

 

壳的代码

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_VJ9MF86MY8PRDB9.png)

 

看一下 got 表，发现导入的是一堆三角函数 tan, 拿三角函数做什么？？？

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_HA8ENKYR55H8XZJ.png)

 

字符串表大部分被抽空了：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_Q9MXXM3WU9B2JUC.png)

 

符号表里也是一堆 nullsub，不知所云：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_NXJWJN7R87Q3Z52.png)

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_EM5VU3DYAWP8JCD.png)

 

好吧，看起来挺复杂，不过无所谓，初始化函数会出手。

 

看看 init array：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_N7UJC4W7KKRK9VN.png)

 

有三个函数，并且没有混淆，那我们就从可以这三个 init 函数入手分析。

2.GNU Hash
----------

在正式开始分析之前，我们先说说 GNU Hash，顾名思义，GNU Hash 就是一种 Hash 算法，具体代码如下：

```
uint32_t gnu_hash(const char* str)
{
    uint_32 h = 5381;// 0x1505
    while(*str != 0)
    {
        h += (h<<5) +*str++;// 33 * h + *str
    }
    return h;
}

```

算法很简单，就是对一个字符串（也可以是 byte 数组），先设定值 0x1505, 然后每次将该值乘 33，再加上下一个字符的数值，直到字符串结束。用位运算表示就是每次左移 5 位自加，再加上下一个字符数值。

 

简单来说，GNU Hash 将给定的字符串映射成为一个 32 位无符号整数。

 

在 linux 平台中，与 GNU Hash 相对应，有另外一种 Hash 叫 ELF Hash，算法如下：

```
uint_32 elf_hash(const char* str)
{
    uint32_t h=0,g;
    while(*str)
    {
        h = (h<<4) + *str++;
        g = h & 0xf0000000;
        h^= g;
        h^ = g > >24;
    }
    return h;
}

```

也是一种计算给定字符串的 hash 值的算法。

 

这两种算法有啥用？

 

GNU Hash 和 Elf Hash 主要是用于加速符号的查找。例如要使用 dlsym 函数获取一个符号的地址，背后就依赖这两个函数。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_4U3AB7M47WY2QDK.png)

 

在 android linker 的源码，链接时的符号决议过程也有 gnu hash 和 elf hash 的参与。gnu hash 晚于 elf hash 诞生，据说是能将符号查找的过程提速 50% 左右。在具体执行时，会根据标志位选择一种 hash 算法查找符号。在使用 gcc 编译时，通过 --hash-style = gnu 来指定生成的 so 使用 gnu hash 来管理符号, 否则，默认使用 elf hash。

 

具体如何查找的，可以参考 android linker 部分的源码，大致如下：

```
unint32_t h = gnu_hash(name);
----省略-----
n = gnu_bucket_[h % gnu_nbucket_];
do {
    Elf64_Sym * s = symtab_ + n;
    char * sb=strtab_+ s->st_name;
    if ((gnu_chain_[n] ^ hash) >> 1) == 0 && strcmp(sb ,name)) == 0 ) {
        break;
    }
} while ((gnu_chain_[n++] & 1) == 0);
Elf64_Sym * mysymf=symtab_+n;
long* finaladdr= reinterpret_cast(sb->st_value + (char *) start);
return finaladdr; 
```

每个 elf 文件中会有一个特殊的区域，用来专门存放 GNU Hash 相关的信息，主要有 bucket 和 chain。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_3VRBECXE845QMS5.png)

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_F626HT9R3TQC5VX.png)

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_EHXRUWABMFJTJZK.png)

 

首先根据符号名计算出 gnu hash，然后根据 hash 找到对应的 hash 桶，hash 桶中先拿到初始索引 n。

 

接着根据索引 n 去符号表中获取第 n 个符号。符号表的结构如下：

```
typedef struct {
        Elf64_Word      st_name;
        unsigned char   st_info;
        unsigned char   st_other;
        Elf64_Half      st_shndx;
        Elf64_Addr      st_value;
        Elf64_Xword     st_size;
} Elf64_Sym;

```

获取到符号后根据 st_name 字段（符号名在字符串表中的偏移），拿到符号名，使用 strcmp 比较符号名，如果同时满足 (gnu_chain_[n] ^ hash) >> 1) == 0 条件，则认为找到了符号。

 

找到符号后获取 st_value 的值（符号在模块内部的地址偏移），加上模块的基地址，就是符号的真实地址了，然后返回。

 

如果比对失败，则 n++，继续执行符号查找。

 

使用 elfhash 查找原理大致相似，这里就不在赘述了。

3. 三个 init 函数
-------------

简单介绍了一下通过 GNU Hash 获取符号地址的原理，接下来我们分析三个 init 函数

### 3.1 第一个 init 函数

该函数先执行了一个 get_module_addr 的函数，通过 syscall 打开 / proc/self/maps 文件，然后分别读出 libc,liblog,libdl,libstdc++ 的基地址

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_F78A4GPF4PMF696.png)

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_M2THP3GBT3BVSVN.png)

 

当然，这些字符串都被内置到函数里进行加密了，使用的时候临时解密：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_KA96MDYF9SJFADP.png)

 

拿到四个模块的基地之后，分别遍历他们的 dynamic 节区，拿到各个模块的字符串表，符号表，hash 表，重定位表等信息（一个弱化版的 prelink 操作）  
![](https://bbs.kanxue.com/upload/attach/202306/872365_MQJ3JX497FKPUKD.png)

 

拿到一些基本信息后，通过遍历 libc，libdl 等模块的符号表，获取到 dlsym，dlclose，等函数的地址。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_8XU6V5DX8Q7THNJ.png)

 

然后通过 RC4 解密了一大坨数据，主要是函数名和偏移：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_EUEGUKST52W7VMS.png)

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_H7MV4U5A86GZWA9.png)

 

可以看到解密出了 munmap,mmap,calloc 等字符串。解密这些字符串干啥？往下看

 

解密完后开始计算这些字符串的 GNU Hash！

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_EJKZYA3E69NSKJA.png)  
![](https://bbs.kanxue.com/upload/attach/202306/872365_BD48XC59HT86VN4.png)

 

（还记得 GNU hash 的 magic 5381（0x1505）吗?)

 

然后遍历各个之前获取到的模块的 hash 表，执行 “通过 GNU Hash 获取符号地址” 的操作：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_R8KSBGN4SSCZG2M.png)

 

自定义的字符串比较函数，没有调用 strcmp

 

比较成功后，获取到符号的真实地址：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_9J8BBAF3D5PJP3M.png)

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_JH2AKBA7EQ7EBTR.png)

 

其中，X1 是符号的 st_value,x13 是 libc 的基地址。加完后 X13 就是 munmap 的真实地址。

 

然后将 x13 存储到了一个地址：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_4Q778DVJREXPHXU.png)

 

其中，X20 是 libF*Guard.so 的基址，而 x10 是：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_89AFZZY7GGJ2VHP.png)

 

可以看到，写入的地址，正好是之前 tan 函数的地址：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_YCEQB738P8GGPAB.png)

 

所以就知道了，该加固通过函数名，计算 GNU Hash，然后直接去目标模块中拿到函数地址，写回 got 表，通过这种方式实现了导入函数的重定位。同时由于自己计算，所以符号表中该符号没用了（否则要进行符号重定位（0x402 类型重定位）来更正地址，对应 jmprela 表），于是实现了导入符号的抹去！

 

通过 GNU Hash 的方式，将之前解密出来的符号名全部获取地址，然后写入 got 表：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_PX88WJCVUQ9BRTS.png)

 

第一个 init 函数到这里就结束了。

### 3.2 第二个 init 函数

第二个 init 函数主要是初始化了两个全局变量：  
![](https://bbs.kanxue.com/upload/attach/202306/872365_SPACEG79Z74U7TX.png)

### 3.3 第 3 个 init 函数

第三个 init 函数里面有三个函数：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_XQQAJ6JSJBKFUZJ.png)

 

第一个函数仍然是做一些初始化工作：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_CJ8N2YCX2ZPMNBA.png)

 

第三个函数通过间接调用 free 做了一些收尾工作。主要逻辑在第二个函数：

#### 3.3.1 解密子 so

第二个函数首先间接调用了一个函数，真实地址是 0x3E8F20

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_WCFN79X2GZWHFKM.png)

 

该函数的作用是解密子 so 的代码和数据。具体算法是先通过 RC4 解密了一段初始信息，然后根据初始信息，分别使用四种解密算法对原始数据进行分段解密，每 256 字节换一种算法：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_ERCBMQAKWAWG4TC.png)

 

解密完成后，又对末尾的一大段数据进行了二次解密：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_ZAFEA8GSDPUMSYF.png)

 

至此，之前加密的数据可以全部 dump 下来了，因为已经完成解密。

#### 3.3.2 对子 so 执行 prelink

解密出的子 so 是没有重定位的，所以还有需要执行正常的重定位操作。该加固首先根据解密的数据，使用子 so 的 dynamic 段做一次 prelink：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_5V8S83XME65X8KE.png)

#### 3.3.3 根据 GNU Hash 对子 so 重定位

子 so 解密完后有一大串字符串：  
![](https://bbs.kanxue.com/upload/attach/202306/872365_8U8N38PJYJ5MUAS.png)

 

可以看到，有模块名和函数名。该加固接下来先检查模块信息是否已经加载。然后根据后面的函数名，计算 GNU Hash，去模块中获取函数地址，然后重定位。! ![](https://bbs.kanxue.com/upload/attach/202306/872365_K4QSWX4RQK7XKB3.png)

 

等等，地址获取到了写入到哪里呢？

 

重定位主要有基址重定位（0x403）和符号重定 (0x402) 位两种，基址重定位通常就叫 rela，而符号重定位通常叫 jumprela。

```
//重定位项的数据结构
typedef struct {
        Elf64_Addr      r_offset;
        Elf64_Xword     r_info;
        Elf64_Sxword    r_addend;
} Elf64_Rela;

```

其中，r_info 为重定位类型。

 

我们看到子 so 的基址重定位是正常的：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_9BUU6T7RS8F9UJN.png)

 

但是符号重定位有问题：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_7X4PWXEZQ7PFXEA.png)

 

其中的 r_info 字段不是正常的 0x402,ida 表示 unknown

 

正常的符号重定位表是这样的：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_JD49K2H9C6KGBF5.png)

 

可以看到 r_info 低位是 0x402，高位的数字代表该重定位项目对应的符号在符号表中的索引。

 

符号重定位的过程是这样的：

 

1. 根据重定位项目 r_info 信息获取重定位项目的符号表索引

 

2. 根据符号表索引拿到对应的符号

 

3. 根据对应的符号的 s_name 信息，在字符串表中获取到符号名

 

4. 通过 gnuhash 或 elfhash 获取到符号对应的真实地址（在其他模块中）

 

5. 将符号的地址填写到重定位项目的 r_offset 中。

 

该加固的符号重定位项目有问题，其实是将 r_info 字段直接替换成了需要重定位的符号名在字符串表中的索引。然后直接根据 r_info 字段获取符号名，再通过 GNU Hash 拿到符号的地址，然后直接回填到 r_offset 中，少了根据符号索引获取符号这一步。

 

这样一来，也就是和第一个 init 函数中做重定位的方法一致了，只是这里是对子 so 也是直接用 GNU Hash 做重定位。

#### 3.3.4 对子 so 进行基址重定位

对子 so 做完符号重定位后，根据基址重定位表做了基址重定位，方法和 linker 中的相同。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_37672E7N65W9GTH.png)

 

执行到这里，子 so 的重定位工作就完成了。

#### 3.3.5. 抹除子 so 的链接信息

子 so 完成重定位后，该加固抹去了解密出来的子 so 链接字符串信息，dynamic 表，和重定位表等信息：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_8D3PPKC9DMRKZ7U.png)

#### 3.3.6 解密导出符号

但是事情并没有结束，如果子 so 有导出符号呢？

 

导出符号必须严格符合 linker 的查找方式，即 GNU Hash 或者 Elf Hash 否则 linker 在链接其他 so 的时候就找不到符号。

 

同时，由于 linker 拿到的符号表信息和字符串表信息是壳 so 的，所以导出符号的解密是在壳 so 中完成的。

 

具体方法是这样的：

 

1.（壳的）符号表中符号如果 s_other 字段为 0x10 或者 0x30，则为加密符号，执行解密操作。

 

2. 先解密符号对应的符号名（在壳的字符串表中）。

 

3. 根据是 0x10 与 0x30 的不同，执行不同的解密逻辑解密符号的 s_value

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_MDZ8CF5DEGJ58JG.png)

 

可以看到，有一些符号的 s_other 字段为 0x10

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_GK7PW3ZGQV5EGV2.png)

#### 3.3.7 hook 关键函数

由于该加固是保护游戏的，所以会对 unity，unreal 一些关键函数进行 hook。解密完导出符号后，会在 sub_3f6ad0 中 hook 一些函数：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_4AG7QXKD2TMPN4U.png)

 

对于 u3d 游戏，该壳 hook 了加载 global-metadata.dat 的函数，在 il2cpp 加载 global-metadata 时，通过 inline hook 的方式，跳转到 libF*Guard.so 中对加载的数据进行解密，然后返回。

 

正常的加载 global-metadata 的函数：  
![](https://bbs.kanxue.com/upload/attach/202306/872365_UTGYCHHVYZKVGX5.png)

 

hook 之后的：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_2E5NHYTXP5TZUC3.png)

#### 3.3.8 执行子 so 的初始化函数：

子 so 很有可能也有一些初始化函数，所以解密出来后要先执行掉：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_7JUUQ7TKB2KEURN.png)

 

至此，该加固完成了他的工作，子 so 开始正常运行。

4. 修复
-----

该加固的工作原理我们已经分析清楚了，我们直接在 3.3.1 之后 dump 解密的子 so，在 3.3.6 后 dump 壳 so 的符号表和字符串表信息，覆盖回去就可以。

### 4.1 修复符号重定位表

最关键的是子 so 的符号重定位如何修复，即该加固做了无导入符号加固，他自己 link，可以。但是我们修复的目的是让系统 linker 可以正确的导入这些符号。

 

查看壳 so 的符号表，我们发现有大量的空符号：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_SRSP5HQV4SFZAND.png)

 

我们可以利用这些空符号，将导入符号填进其中，然后修正符号重定位表，让原来的直接通过字符串偏移获取符号名，改为正常的通过符号表索引获取符号，这样 linker 就可以正常重定位了。同时，我们需要将符号名回填至壳的字符串表中，与回填的符号对应。算法如下：

```
oldstrbase = 0x444
oldstrsize = 0xa18
rawjumprelabase = 0xB5339C
rawjumprelacount = 0x8c9
relaentrysize = 0x18
strbase = 0x3a73a48
symbase = 0x3A6D9D0
symentrysize = 0x18
newbase = 0x3a75444
 
def getInt8(data,offset):
    return data[offset]
 
def getInt32(data,offset):
    return data[offset]|data[offset+1]<<8|data[offset+2]<<16|data[offset+3]<<24
 
def putInt32(data,offset,value):
    for i in range(4):
        data[offset+i]=value&0xff
        value = value >> 8
 
def getInt64(data, offset):
    return getInt32(data, offset) | getInt32(data, offset + 4) << 32
 
def putInt64(data, offset, value):
    for i in range(8):
        data[offset + i] = value & 0xff
        value = value >> 8
 
usedindex = []
 
def getNextEmptySym():
    global usedindex
    index = 1
    while True:
        name = getInt32(data,symbase + index * 24)
        ch = getInt8(data,strbase+name)
        if ch == 0 and index not in usedindex:
            usedindex.append(index)
            return index
        else:
            index = index + 1
 
with open("f:\\libil2cpp.so",'rb') as f:
    data = list(f.read())
 
for i in range(rawjumprelacount):
    r_addr = getInt64(data,rawjumprelabase+24*i)
    r_info = getInt64(data,rawjumprelabase+24*i+8)
    r_addend = getInt64(data,rawjumprelabase+24*i+16)
    if r_info == 0:
        newaddend = getInt64(data,r_addr) + r_addend
        putInt64(data,rawjumprelabase+24*i,r_addr)
        putInt64(data,rawjumprelabase+24*i+8,0x403)
        putInt64(data,rawjumprelabase+24*i+16,newaddend)
    else:#在符号表中创建一个新的符号
        index = getNextEmptySym()
        putInt32(data,symbase + 24*index,(r_info + newbase - strbase) & 0xffffffff)
        putInt32(data,symbase + 24*index + 4,0x12)
        putInt64(data,symbase + 24*index + 8,0)
        putInt64(data,symbase + 24*index + 16,0)
        print("new symbol create:"+hex((r_info + newbase - strbase) & 0xffffffff)+","+hex(index))
        #修正重定位表
        putInt64(data,rawjumprelabase+24*i,r_addr)
        putInt32(data,rawjumprelabase+24*i+8,0x402)
        putInt32(data,rawjumprelabase+24*i+0xc,index)
        putInt64(data,rawjumprelabase+24*i+16,0)
 
#迁移字符串
for item in range(oldstrsize):
    data[newbase + item] = data[oldstrbase + item]
 
with open("f:\\libil2cpp1.so","wb") as f:
    f.write(bytes(data))

```

修复前，直接通过符号名，计算 GNU Hash 做重定位：  
![](https://bbs.kanxue.com/upload/attach/202306/872365_F7A5WHAC3YK7JK8.png)

 

修复后，变为了正常的 0x402 类型的重定位：  
![](https://bbs.kanxue.com/upload/attach/202306/872365_2STDCNYRX2CQTR8.png)

### 4.2 修复 dynamic 段，section 信息

我们需要将壳 so 的 dynamic 中重定位表，init array，符号表等信息修正成子 so 的对应信息。保持壳 so 的 Hash 表，字符串表和符号表不变。

 

同时，添加正确的 section 信息，这里就不赘述了。

### 4.3 段错误？？

修复完后我们替换手机里的 libil2cpp.so。结果出现了段错误，程序直接闪退。

 

观察日志发现是在 linker 重定位时出现的段错误。

 

原来，子 so 的符号表被放在了程序的代码段？！代码段的权限是 rx，在重定位写入的时候就是会段错误。

 

壳 so 可以正常重定位是因为壳 so 在重定位前通过 mprotect 把整个代码段赋予了 w 权限。但是在 andorid 8 以后，系统禁止出现具有 W+E 权限的段了，我们不能直接改段属性：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_FW7HFKFBDSEM959.png)

 

所以我们只好新添加一个 data 段。

 

刚好符号表相关的信息在代码段的末尾，我们直接切割出一个数据段，赋予 RW 权限即可。

 

原来的段信息，代码段从 0-0x3a9c980

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_FQTHZYEM6HSJ8KJ.png)

 

我们替换 GNU Read-only After Relocation 段：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_UWV4PMQ6ZX7J68R.png)

 

代码段大小变成了 0x32fda00, 新增了一个从 0x32fe000 开始的段，这个段刚好包含了符号表等数据信息。

### 4.4 替换 global-metadata

修复完成后，由于跳过了壳的 init 函数，导致 libF*Guard.so 没有 hook 到 global-matadata 的解密函数，如果直接加载的话，就是一个加密过的 global-metadata。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_WP9C8HR298Y7YKQ.png)

 

所以我们需要 dump 出正常的 global-metadata，替换掉手机里的。  
![](https://bbs.kanxue.com/upload/attach/202306/872365_G5ZMKPE39RTHSNB.png)

 

这样之后，游戏就可以正常启动运行了:)

5.u3d 的一些逆向
-----------

修复完成后，我们试试 il2cppdumper。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_2JAVT2273Z7PPJZ.png)

 

直接成功。

 

打开脚本看了半天没发现什么有用的逻辑，这时候，突然看到项目目录下的这个：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_76J55MWY3NK4ZXF.png)

 

这游戏的核心逻辑在 lua 不在 c# ？？！

 

随便打开一个 lua 文件，发现是 base64 编码的，但是解码后依然是乱码，应该是加密的。

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_3TTRZEFHYXRF36U.png)

 

不过解密逻辑应该在 il2cpp 里，搜了一个函数：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_6FHS4V9VYMASFN5.png)

 

看到叫 xxtea_decryptBase64String，解密完直接丢给 lua 了。hook 了这个函数，打印出了 xxtea 的秘钥：

 

![](https://bbs.kanxue.com/upload/attach/202306/872365_TTTEUYBYW59NYH9.jpg)

 

写了个脚本解密了一下，把所有 lua 文件都解密了：  
![](https://bbs.kanxue.com/upload/attach/202306/872365_27JZPGJUMYPFEG6.png)

 

嗯，这下算善始善终了，收工~

6. 总结
-----

这次主要对某 Guard 的手游加固进行了分析，其中最为出色的地方在于通过 GNU Hash 和函数名来对导入函数进行重定位，然后在符号表中删除了导入符号相关的信息，从而实现了导入符号隐藏。同时也有很多不同的解密算法在对各种数据进行解密，总体来说，相较于之前的某盾加固，有一些新意，各有千秋。

 

不过，加固做的再复杂，也总是有办法脱掉的，因为，逆向工程师永远胜利！:)

  

[Windows 开发不完全指南](https://www.kanxue.com/book-section_list-85.htm)

最后于 8 小时前 被乐子人编辑 ，原因： [#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)