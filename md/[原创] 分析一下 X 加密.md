> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267412.htm)

> [原创] 分析一下 X 加密

### 前言

本地环境依然是 6.0.1 的系统，这应该是最后一个分析的抽取类型的壳，后面会正式进入 VMP 的分析，文章没有分析的太透彻，主要还是以脱壳为主。文中 ida 中出现的字段和函数名可能根据自己的理解被修改过，也可能出现错误，还请各位大佬多多担待，并且指正，由于某些原因最终的脱壳脚本没办法给大家提供，但是会有思路，还请大家多多包涵.

### Java 层入口

首先，这个壳有点不太一样  
![](https://bbs.pediy.com/upload/attach/202105/762912_K4ZX2S59JRTQBAF.png)

 

大家通过看上面图片可以发现，这个应用的 dex 抽取过后和壳打到一起了。所以后面就不会有再去加载 dex 的操作。OK，下面正式开始分析

 

定位 s.h.e.l.l.S

 

![](https://bbs.pediy.com/upload/attach/202105/762912_M7U3SVFDX8PBTXX.png)

 

s.h.e.l.l.N-> 静态块

 

![](https://bbs.pediy.com/upload/attach/202105/762912_YFTQ6VU2695YT5F.png)

 

之后进入 so 层的分析

### So 层入口

![](https://bbs.pediy.com/upload/attach/202105/762912_QGE978CDWJWUAQ5.png)

#### so 脱壳

调试的话，直接定位 linker，调试一下. init_proc

 

![](https://bbs.pediy.com/upload/attach/202105/762912_FBT5DFT8WR3HHJG.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_62V6EZFRK4AH5GM.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_K78MYN3E5CNNB3B.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_5MJCJWY9M8Y43UV.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_7XADYTZ5AUKCNER.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_X327UGNGZ95W56S.png)

 

过掉 UND 之后，可以开始 dump 了。

 

![](https://bbs.pediy.com/upload/attach/202105/762912_GDAEY842QSQ56DW.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_BFC6FZZ3STVJHZX.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_UMGV59JXMNB3XR4.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_ZNHYD36EHFBQXJR.png)

 

发现并不是从 ELF 头开始的，而是 0x10000 起始的。所以直接修复了

 

我采用新建一个 2 进制文件，先将 0x10000 之前的数据 copy 进去，后面再拼接此 dump 出的数据，然后实际大小，是 dump 出的大小加上前面的数据得出总大小

 

![](https://bbs.pediy.com/upload/attach/202105/762912_EUDT7ZG3PPHS6NS.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_MPVR3K7CDJMJRK8.png)

 

然后把数据追加到新创建的 so 中，注意对齐

 

![](https://bbs.pediy.com/upload/attach/202105/762912_68DS3CBPMDDDEWP.png)

 

由于我并不需要把 so 完美修复好，我本地修复主要是动态调试有个对照，所以这 2 个 segment 修复后，就可以看到大致的代码了。

 

![](https://bbs.pediy.com/upload/attach/202105/762912_NZ5RAPMEFGNSD8R.png)

#### 分析 INIT_ARRAY

init_array 做了保护，被编译器拆分成很多函数，我分析的时候几乎是一个一个看的，总结出来了，init_array 主要做 2 个事情，第一，字符串解密，并保存；第二，反调试，下面简单介绍一下主要流程.

 

![](https://bbs.pediy.com/upload/attach/202105/762912_AD6KG9SPPWD3X5X.png)

 

前面若干个函数主要做了一些字符串的解密，类似于下图

 

![](https://bbs.pediy.com/upload/attach/202105/762912_GNESQCEDEC8A7M5.png)

 

定位 **sub_12BDC**:

 

兼容性处理与函数地址初始化

 

![](https://bbs.pediy.com/upload/attach/202105/762912_WU93W4MPY878RK6.png)

 

定位 **sub_2CB80**

 

这函数主要做反调试

 

![](https://bbs.pediy.com/upload/attach/202105/762912_E2MW87ZZR9HDCX3.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_8AXJKKHDFQQE528.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_X39J369WTSXQE7E.png)

#### JNI_ONLOAD

这个函数并不是很长，通过分析，核心做了 2 个事情。(这里说一下)

 

**初始化一些数据**: 例如: 机型相关的数据：HARDWARE,MODEL,RELEASE,sdk_version 等等（这个是为了兼容性考虑，后面逻辑会有一些判断），applicationInfo，processName，sourceDir, 待使用的文件路径，和一些 Java 层的 class 名字，最终都会保存在一个全局结构体中，和乐固类似。

 

**Java 层的函数动态注册**: 其中主要涉及到如下函数

 

**N:l->sub_3C02C

 

N:r->sub_3EF5C

 

N:ra->sub_3F46C**

 

下面做简单分析，定位 **sub_3AB48(JNI_ONLOAD 核心实现在这里)**

 

![](https://bbs.pediy.com/upload/attach/202105/762912_VV7RTTJWKS98HPK.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_M9XY84UTEBREZH7.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_VNE8KWFTTA8H4Y6.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_D8CADJ27RR9Q4UV.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_EU3QVYW5FK3QWNK.png)

 

JNI_ONLOAD 走完了，下面继续回到 Java 层，看看调用了哪个 native 函数

 

![](https://bbs.pediy.com/upload/attach/202105/762912_QBPUJKK8D93QY2J.png)

#### sub_3C02C:N->l

这个函数有很多的兼容性的操作.

 

![](https://bbs.pediy.com/upload/attach/202105/762912_4EU9HGTTVBVZ8TA.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_C96Q4UXZNFY6S5K.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_ZD3HEA3EKQSKVCY.png)

 

这里 hook 了非常多的 art 的函数，我们比较关注的, 定位 sub_26BAC, 壳的还原时机就是 hook 了这个 loadMethod 函数

 

![](https://bbs.pediy.com/upload/attach/202105/762912_NNYJF3Z8QQ5RCHM.png)

 

后面还有一些逻辑，不过到这里，这个壳已经可以脱了。

#### opcode 填充时机

定位 sub_27034

 

![](https://bbs.pediy.com/upload/attach/202105/762912_PJTAWPVQN9QA3E8.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_7KU7JDAHWRJJRX5.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_TATDCXVJTY8BZYC.png)

 

然后进入 sub_51CD8 开始真正的填充

 

![](https://bbs.pediy.com/upload/attach/202105/762912_YWQVSXTWG3T72C6.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_WVPSF4QJ3XBKE3Y.png)

### 反调试与环境检测

#### 反调试

init_array 已经有 3 中反调试了

 

![](https://bbs.pediy.com/upload/attach/202105/762912_E2MW87ZZR9HDCX3.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_8AXJKKHDFQQE528.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_X39J369WTSXQE7E.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_UCM5XJFTHQK6A8V.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_XYUEKZGBJMXEAT5.png)

 

后面还有，但是我 idb 丢失了一次，这个忘记了，大家到时候自己调试一下在 sub_2D6CC

#### 环境检测

检测是否有 / data/dexname  
![](https://bbs.pediy.com/upload/attach/202105/762912_U86AARZRUZSMYDX.png)

 

尝试 env->loadClass("cn/youlor/Unpacker")

 

![](https://bbs.pediy.com/upload/attach/202105/762912_BF6YEHFUKK9NXEA.png)

 

检测是否存在 “/data/local/tmp/unpacker.config”  
![](https://bbs.pediy.com/upload/attach/202105/762912_RGFNA45WFWYZCN8.png)

 

检测是否存在 fart

 

![](https://bbs.pediy.com/upload/attach/202105/762912_CB25PKWHMNFQH66.png)

 

![](https://bbs.pediy.com/upload/attach/202105/762912_WDEJXAFZ67AX8BP.png)

 

检测 / data/local/tmp/re.frida.server

 

![](https://bbs.pediy.com/upload/attach/202105/762912_WSEA4GYMH7VTDCF.png)

 

然后有个专门的线程，检测 maps 中的内容，检测了如下字符串

 

com.android.reverse-

 

/data/local/tmp/libFupk3.so

 

xposed.Fdex2

 

/system/bin/app_process32_xposed

 

xposed.installer

 

app_process64_xposed

 

libxposed_art.so

 

io.va.exposed

 

io.virtualapp.sandvxposed

 

libriru_

 

com.saurik.substrate

 

re.frida.server

 

mapp.rm-

 

_frida-agent.so

 

com.example.FunDex-

### nop 反调试与环境检测

经过下面一段脚本的执行，可以直接 F9 让应用完美运行起来

```
base =0x13FB4
 
 addr_patch1 = 0x2CB90
 addr_patch2 = 0x2CC5E
 addr_patch3 = 0x2CD48
 addr_patch4 = 0x4035C
 addr_patch5 = 0x3C336
 addr_pathc6 = 0x3C340
 addr_patch7 = 0x3C3B2
 addr_patch8 = 0x3C3FE
 
 addr_patch1_add = addr_patch1 - base
 addr_patch2_add = addr_patch2 - base
 addr_patch3_add = addr_patch3 - base
 addr_patch4_add = addr_patch4 - base
 addr_patch5_add = addr_patch5 - base
 addr_patch6_add = addr_pathc6 - base
 #去环境检测
 addr_patch7_add = addr_patch7 - base
 addr_patch8_add = addr_patch8 - base
 
 print(hex(addr_patch1_add))
 print(hex(addr_patch2_add))
 print(hex(addr_patch3_add))
 print(hex(addr_patch4_add))
 print(hex(addr_patch5_add))
 print(hex(addr_patch6_add))
 print(hex(addr_patch7_add))
 
 # only bypass
 if addr_patch1_add == 0x18bdc and addr_patch2_add == 0x18caa:
     addr_patch1_real = base_ea+addr_patch1_add
     addr_patch2_real = base_ea+addr_patch2_add
     addr_patch3_real = base_ea+addr_patch3_add
     addr_patch4_real = base_ea+addr_patch4_add
     addr_patch5_real = base_ea+addr_patch5_add
     addr_patch6_real = base_ea+addr_patch6_add
     addr_patch7_real = base_ea+addr_patch7_add
     addr_patch8_real = base_ea+addr_patch8_add
 
     idaapi.patch_dword(addr_patch1_real, 0x0000F04F)
     idaapi.patch_word(addr_patch2_real, 0xBF00)
     idaapi.patch_dword(addr_patch3_real, 0x0000F04F)
     idaapi.patch_dword(addr_patch4_real, 0xBF00BF00)
     idaapi.patch_word(addr_patch5_real, 0xBF00)
     idaapi.patch_dword(addr_patch6_real, 0xBF00BF00)
     idaapi.patch_dword(addr_patch7_real, 0x0000F04F)
     idaapi.patch_dword(addr_patch8_real, 0xBF00BF00)

```

### 脱壳

比较抱歉，这边由于某些原因，最终的脚本不能给到大家，下面说一下思路。

#### 1.Dump 壳已经准备好的数据

大家要先仔细调试一下 sub_27034 这个函数，就知道 dump 哪里了，很明显。

 

根据上面壳本身的还原逻辑，我们可以直接从内存中 dump 出 3 段数据，

 

第一段是真正的 opcode 相关信息存放的数据段，例如 opcode 的长度，以及真正的 opcode。

 

后面 2 段是存放有关 debuginfo 相关的数据，例如 debuginfo 的值，以及当前 debuginfo 在第一段数据中对应的起始地址。

 

然后我们还需要保存当前 3 段数据的起始地址，因为这些数据 dump 出来的时候，存放的都是实际地址，所以我们需要减去起始地址，纯计算偏移去做。

```
int getReallyAddr1(int addr) {
    int malloc_base = 0xAA640000;
    int r_addr = addr - malloc_base;
    return r_addr;
}
 
int getReallyAddr2(int addr) {
    int malloc_base = 0xAB140000;
    int r_addr = addr - malloc_base;
    return r_addr;
}
 
//0xAA640000：第二段数据的起始地址
//0xAB140000：第三段数据的起始地址
//这个函数计算真正的地址
int getReallyAddr(int addr) {
    if (addr > 0xAA640000 and addr < 0xAB140000) {
        return getReallyAddr1(addr);
    } else {
        return getReallyAddr2(addr);
    }
}
//这个函数计算用哪个mmap的内存去取数据
char *getReallyMp(int addr, char *mp1, char *mp2) {
 
    if (addr > 0xAA640000 and addr < 0xAB140000) {
        return mp1;
    } else {
        return mp2;
    }
}

```

#### 2. 根据原 dex 提取 debuginfo

这里用 py 脚本解析 DexFile 就好。把每个 code_item 的 debuginfo 保存到文件，这里是否保存到文件取决于大家最终的修复脚本用 py 还是 C，我是用 C 的，但是我的解析脚本是 py，所以保存文件中给 C 解析.

```
ff = 'classes.dex'
p = DexParser(ff)
p.parse()
for classDef in p.class_def:
    dataOff = classDef['class_data_off']
    if dataOff != 0:
        dataItem = classDef['class_data_item']
        direct_method = dataItem['direct_methods']
        vir_methods = dataItem['virtual_methods']
        for dirMethod in direct_method:
            info_off = dirMethod['code_item']['debug_info_off']
        for vir_method in vir_methods:
            info_off = vir_method['code_item']['debug_info_off']

```

#### 3. 还原壳的修复逻辑

这里还原大家一定要直接看指令，不要 F5，不要 F5，不要 F5！

 

1. 还原 sub_152D2

 

2. 还原 sub_51CD8

#### 4. 最终效果

脱壳前:

 

![](https://bbs.pediy.com/upload/attach/202105/762912_BY8EHT9HEU8AHH5.png)

 

脱壳后:  
![](https://bbs.pediy.com/upload/attach/202105/762912_3Y3SQ9ADRSSA7DK.png)

[[培训] 优秀毕业生寄语：恭喜 id: 一颗金柚子获得阿里 offer《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-15958.htm)

最后于 2021-5-9 14:59 被 GitRoy 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [res.zip](javascript:void(0)) （7.84MB，82 次下载）