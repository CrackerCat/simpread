> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288890.htm)

> [原创] Android 加固技术回忆录 (0x1) 一代 TX 加固完整分析

小白逆向记录, 有错误的地方还请各位大佬指正

内容涉及到结构体还原, 自实现 linker 脱壳修复, dex 解密和 dump 分析, 以及加固技术原理

逆向环境
====

*   lg g5 android 4.4.4
    
*   ida 9.1
    
*   010editor
    
*   记事本, 记录一些重要信息, 哈哈
    
*   一双小手, 大手也没事, 按的准键盘就可以  
    

Java 层简单分析  

=============

拿到 Apk 后先简单分析一下 Application, 没有发现骚操作, 确定直接加载 lib 中 so 后, 直接进入 Native 分析  

![](https://bbs.kanxue.com/upload/attach/202510/954363_RH2A7VSD4A2XYZH.webp)

调试 / 分析壳子 ELF
=============

ida 打开 so 后发现导入是空的, 看来了壳子也做了保护, 分析一下

![](https://bbs.kanxue.com/upload/attach/202510/954363_S855TUX9ZSDMZFP.webp)

这里 shdr 有一些问题, 查看段表找不到 .init.array, 那就直接动态调试抓把

![](https://bbs.kanxue.com/upload/attach/202510/954363_MGGF2PFUG65RG5D.webp)

断点. init / .init.array  

-------------------------

挂上调试, 直接断在 linker 的 callFunction, br .init / .init.array 的位置

会先加载 Bugly.so, 可以直接跳过, 直到 libshella.so 加载后再 f7 跟进去即可

![](https://bbs.kanxue.com/upload/attach/202510/954363_MXPNZGZV46V396P.webp)

AUV, 您猜怎么着, 断住了!

(其实这里已经分析下去了, 忘了截图, 后补的图)

![](https://bbs.kanxue.com/upload/attach/202510/954363_MQK9RMKHZFFKR5U.webp)

调试发现只调用了一次 callFunction, 说明只有一个 .init.array

ida 静态跳到 0x944, 开始分析, 乍一看没有看出什么, 卒  

![](https://bbs.kanxue.com/upload/attach/202510/954363_B3A7ZZ55XDMEJFF.webp)

动态分析一下, 一行一行跟着汇编看

补了一下, 发现这里先取了 so 在内存中的起始地址给 elfStart

initArray01 & 0xFFFFF000 取函数页起始地址, *elfStart != 0x464C457F 判断 elf magic, 每次向下减一页大小, 直到找到 elf magic

(还发现遍历了一下 phdr, 但是没有引用, 暂不知道用途)

![](https://bbs.kanxue.com/upload/attach/202510/954363_QU6XBX9SP2RJQJN.webp)

修复壳子 ELF
--------

调试时, 发现这一块在读 elfStart+idx 的字节, 再各种异或后存回原处, 直接猜测是解密

![](https://bbs.kanxue.com/upload/attach/202510/954363_AU7TXYAM4UZ8R6W.webp)

结合动态调试和静态修补, 以及接下来的函数调用中其中一个函数代码乱码状态, 最终可以确定是动态解密

![](https://bbs.kanxue.com/upload/attach/202510/954363_6STWF9BA7VPBDC7.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_M3DSTG39J2AJ547.webp)

elfStart_backup 这个是一个拼接的 int, 前一个字 (short) 代表需要解密的起始 idx, 后一个字 (short) 代表需要解密的结束 idx

起名 elfStart_backup 是因为, 取完解密的 idx 以后就赋值为 elfStart 地址被后面引用了

![](https://bbs.kanxue.com/upload/attach/202510/954363_3BSWUMGTAXZEY86.webp)

那就 run to 解密完成的地方, 直接 dump 这一小块, 填进 idb 中 (偷个懒, 不用重新打开 so 分析)

mprotect 是恢复内存属性为 read exec, 断在这里 dump 即可

![](https://bbs.kanxue.com/upload/attach/202510/954363_J2EZQTVTSGNSE8W.webp)

```
auto i, fp, start, end;
fp = fopen("d:\\dump_split.so","wb");
start = 0x766AF000; // elfStart + startIdx
end = 0x766B0AB4; // elfStart + endIdx
 
for (i = start; i <= end; i++) {
fputc(Byte(i),fp);
}
 
fclose(fp);

```

dump 后再使用 idapy 填回 idb 即可开始分析  

```
import idc
 
def set_arm_code_range(start_ea, end_ea):
"""
将指定范围设置为 ARM 代码
"""
current = start_ea
while current < end_ea:
idc.split_sreg_range(current, "T", 0)
current += 4
 
print(f"Converted range 0x{start_ea:X} - 0x{end_ea:X} to ARM code")
 
# 使用示例
set_arm_code_range(0x16c3, 0x24b4)import ida_bytes
import os
 
def patch_elf_from_file(offset, file_path, size=None):
"""
从文件读取内容并修补到ELF的指定偏移
:param offset: ELF文件中的偏移地址
:param file_path: 包含修补数据的文件路径
:param size: 要修补的字节数（如果为None，则使用文件大小）
"""
if not os.path.exists(file_path):
print("Error: File not found - %s" % file_path)
return False
 
with open(file_path, "rb") as f:
file_data = f.read()
 
if size is not None:
file_data = file_data[:size]
 
# 修补数据
for i, byte in enumerate(file_data):
ida_bytes.patch_byte(offset + i, byte)
 
print("Successfully patched %d bytes from %s at offset 0x%X" %
(len(file_data), file_path, offset))
return True
 
# 使用示例
if __name__ == "__main__":
# 将patch.bin文件的内容写入到ELF的0x1000偏移处
patch_elf_from_file(0x1000, "dump_elf_dec_0x1000_0x2ab4")

```

ai 大法好, 哈哈, 效率 +++

接下来在恢复完内存权限后, 做了 cache 刷新, 避免 cup 指令缓存没有更新

![](https://bbs.kanxue.com/upload/attach/202510/954363_WUEYUBBGRGH5B67.webp)

等一下 syscall 983042 ??

![](https://bbs.kanxue.com/upload/attach/202510/954363_WPGKNRBXFUEK4HJ.webp)

孤陋寡闻了, arm 中 syscall 调用号 983042 是 cacheflush

![](https://bbs.kanxue.com/upload/attach/202510/954363_SUBS37UDDYYWE84.webp)

接下来就走到了较为核心的逻辑, sub_766B074C 是 JNI_OnLoad 函数, 先下断点

但是这里没有调用, 因为 getnenv("DEX_PATH") 没有取到东西

![](https://bbs.kanxue.com/upload/attach/202510/954363_9QE3FWSC3V5UA7K.webp)

然后继续往下走, .init.array 就返回来了, 回到了 linker

直接 f9, 我们就到了 JNI_OnLoad 函数, 之前断过, 这里是走到了 libdvm call JNI_OnLoad

![](https://bbs.kanxue.com/upload/attach/202510/954363_5GZYN3E6GKUEJ32.webp)

readOffset 是 0x6dc0, 命名为 buildSdkVersion 是因为这个值用完以后就被赋值 buildSdkVersion 了

记住这里的 0x6dc0, 下面会考

![](https://bbs.kanxue.com/upload/attach/202510/954363_EENZENB55N792BE.webp)

跟进 sub_766B05B4 看一下

根据 elf 开始地址, 在内存中寻找 so 的完整路径, 然后写入 a1, 结束返回上一层

![](https://bbs.kanxue.com/upload/attach/202510/954363_FP6V9XMRBGFNCBP.webp)

现在 jniOnLoad 长这个样子, 跟进 sub_766AF63C

![](https://bbs.kanxue.com/upload/attach/202510/954363_W2ECFCQMVTNK9BJ.webp)

修补自实现 linker 加载 so 的 info 结构体  

--------------------------------

这里的 info 不是 linker 的 soinfo, 而是 elf 的结构信息 !  

这里的 info 不是 linker 的 soinfo, 而是 elf 的结构信息 !

这里的 info 不是 linker 的 soinfo, 而是 elf 的结构信息 !

重要的事情说三遍

sub_766AF63C 也是有一点难以阅读, 不过大概总览了一下, 猜测是自实现 linker 加载 elf

有了这个方向, 下面的分析就事半功倍了

![](https://bbs.kanxue.com/upload/attach/202510/954363_SN4VXF5J3ZTQJD9.webp)

这里就是用完 0x6dc0 后写成 buildSdkVerison 的地方

![](https://bbs.kanxue.com/upload/attach/202510/954363_56QKK2JJGPWFMG7.webp)

先打开 soFd, 在读取壳子 so+0x6dc0 的位置

那么为什么是 0x6dc0 呢, 这有什么特殊呢, 为什么不读取内存中这个位置呢 ???

小小的脑袋有大大的问号 ???

![](https://bbs.kanxue.com/upload/attach/202510/954363_HDRYSKTF7XYUYGE.webp)

看了一下内存中 libshella.so 的内存占用, 计算了一下大小只有 33kb

但是 so 静态大小是 98kb, 那很明显了, 后半部分就是藏着的真实 elf

![](https://bbs.kanxue.com/upload/attach/202510/954363_MQS7SP759JAK6NW.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_ZEJUD9AEQ6E7K9U.webp)

继续跟着分析, 发现一个无根之木, size 从哪里来 ???

这里就要开始补全结构体了, 先设置 minVAddr 为 char[88]

![](https://bbs.kanxue.com/upload/attach/202510/954363_DAG7TCHP985BADG.webp)

然后就很明了了, 从 0x6dc0 读取的 0x58 字节, 必然是一个结构体

开始逆向分析结构体中每一个字段代表什么

![](https://bbs.kanxue.com/upload/attach/202510/954363_2UP9HKV8ZGS8M64.webp)

根据 log 直接得知第一个 int 和第二个 int 分别是 minvaddr 和 size

![](https://bbs.kanxue.com/upload/attach/202510/954363_6BM9SURKCN4BQ6P.webp)

**直接开补 !!!**

**这里注意, 没有定义的地方也需要保留, 不然结构体大小对不上, 就会飘红, 干扰正常分析**

![](https://bbs.kanxue.com/upload/attach/202510/954363_8JKSTTYA2MHHDUT.webp)

这里对着变量 y 一下, 直接设置刚刚声明的结构体类型

![](https://bbs.kanxue.com/upload/attach/202510/954363_C498CE5JAMGCBY5.webp)

接着就柳暗花明了, 继续猛猛补全即可

看到这里 mmap 已经可以完全确定了, 这是一个自实现 linker 加载真实 elf, 那么下面就可以更大胆的猜测了

![](https://bbs.kanxue.com/upload/attach/202510/954363_D526JHYTKNQEWF6.webp)

补全过程比较枯燥, 就不一个一个字段展示流程了, 展示一下大的点的分析基本就很明了了

通过这里可以推断这是另一个结构体, 占据 24 字节, 再结合 [再 mmap] 推测应该是 segment 信息

那么 another 中第一个 short 就是 segmentInfo 结构体存储对应的偏移, 第二个 short 是 segment 的数量

(short 类型根据汇编 LDRH 确定)

![](https://bbs.kanxue.com/upload/attach/202510/954363_X9NZWZTNNA683G7.webp)

再往下根据 log 再结合猜测就可以再补齐很多字段

![](https://bbs.kanxue.com/upload/attach/202510/954363_UP8PUZ49QU5YVSP.webp)

再记录一下 segmentInfo 的补全

v48 是从 v50 做页对齐来的, 那么 v50 就是 loadBias+minVAddr 了, 那也就是 segmentInfo 的第一个 int 就是起始虚拟地址, 后面用来 re-map

v46 是 page end 对齐减去 (v50 + segmentInfos[1]), 那么 segmentInfo 的第二个 int 就是 **segmentMemSize**

![](https://bbs.kanxue.com/upload/attach/202510/954363_BDE9BQEJ8XYAS7J.webp)

猜测是 segmentInfo 的第二个 int 是 memSize 而不是 fileSize 的原因在下面

这里的从文件读取才是 fileSize, 又确定了 fileSize 和 segOffset 字段

![](https://bbs.kanxue.com/upload/attach/202510/954363_UY7ERKJA68W693X.webp)

这里还像是做了一个解密的样子

![](https://bbs.kanxue.com/upload/attach/202510/954363_T9F5RFJP426QXM6.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_7FKUWAYDJ5GN3HX.webp)

一路进来, 是不是很眼熟

![](https://bbs.kanxue.com/upload/attach/202510/954363_XNJG9U2GFM7WVS6.webp)

让我们 google 一下, 原来是 tea 算法, 如果想静态解密可以写脚本试试了

![](https://bbs.kanxue.com/upload/attach/202510/954363_2PFMP78FXBAM72S.webp)

那么 segmentInfo[5] 自然就是解密的数据大小, 单位应该是 bit

![](https://bbs.kanxue.com/upload/attach/202510/954363_U8YFMAK9BDKMM75.webp)

加密以后又做了一个 xz 解压, 这里 zStream 的结构体直接导入就可以

![](https://bbs.kanxue.com/upload/attach/202510/954363_THCH66D9H7GGC85.webp)

修补结构体完成  

----------

最后, 经过了七七四十九天的修补, realElfInfo, segmentInfo, zStream 全部补全的差不多以后就非常明了了

![](https://bbs.kanxue.com/upload/attach/202510/954363_CGU2AXZ94PWD5YA.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_F95UVMZ4Q4X47NV.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_9SBWYT6BNNBR387.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_KJNFCXH7PW6ZBGP.webp)

```
struct realElfInfo
{
    int minVAddr;
    int totalSize;
    __int16 segmentInfoOffset;
    __int16 segmentCount;
    char notUsed[4];
    int strtabOffset;
    int symtabOffset;
    int initFuncOffset;
    int initArrayFuncsOffset;
    char notUsed_1[8];
    bool unusedBool;
    __int16 initArrayFuncCount;
    __int16 neededLibCount;
    __int16 short6;
    int neededLibNameIdxArrOffset;
    int bucketNum1;
    int bucketNum2;
    int bucketOffset;
    int relaOffset;
    int realSymCount;
    int pltSymCount;
    int pltOffset;
    __int16 short2;
    __int16 short3;
    __int16 short4;
    __int16 short5;
};
 
struct segmentInfo
{
    int minVAddr;
    int memSize;
    int segmentOffset;
    int segFileSize;
    int flags;
    int segDecryptBits;
};
 
struct z_stream
{
    char *next_in;
    unsigned int avail_in;
    unsigned int total_in;
    char *next_out;
    unsigned int avail_out;
    unsigned int total_out;
    char *msg;
    void *state;
    void *zalloc;
    void *zfree;
    void *opaque;
    int data_type;
    unsigned int adler;
    unsigned int reserved;
};

```

手搓字节码修复真实 So  

===============

补完结构体仅仅是可以看清楚逻辑, 真实 elf 到现在还没有修复出来, 目测是只能手搓字节码回填了

找真实 So 字段信息  

--------------

这里可以静态解析所有字段, 不过我选择动态一把梭, 直接拿数据, 哈哈哈

![](https://bbs.kanxue.com/upload/attach/202510/954363_URMWWSZ8AGRHR69.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_G6RZJ46JX9Y38XK.webp)

都解密完了以后, 就要 dlopen neededLib 再重定位函数了

![](https://bbs.kanxue.com/upload/attach/202510/954363_KYSNKSAXNCNF8VX.webp)

这是几个 neededLibName 的下标数组, copy 一下

![](https://bbs.kanxue.com/upload/attach/202510/954363_FZ7E4TQBHUSJNVM.webp)

再跟进去就是重定位函数, 一眼顶针

这几个地址在前面预先计算好了, 修复的时候就要手动填进 dynamic 节

![](https://bbs.kanxue.com/upload/attach/202510/954363_GBBWGAWW7NQTDX8.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_7Y6RSV4K53J7UE6.webp)

巧妙的壳子 So 符号覆盖操作
---------------

再往下做了一个很好玩的操作, 把真实 so 的符号表, 字符串表 覆盖 到壳子 so 的符号表和字符串表, **这里先记一下**

![](https://bbs.kanxue.com/upload/attach/202510/954363_SDM8P54WD997BFV.webp)

再恢复 segment 的 flag

![](https://bbs.kanxue.com/upload/attach/202510/954363_6WNZM8EGEKC4SRG.webp)

再 call .init / .init.array 就结束了

![](https://bbs.kanxue.com/upload/attach/202510/954363_F846DKNU8UAJDMA.webp)

修复真实 ELF
--------

这里根据上面的消息, 尝试修复真实 elf

把 dump 出的整块内存丢进 010ediotr, 再把 ehdr, phdr, dynamic 都手打字节码拼一下, 再使用 SoFixer32 修复一下即可 

![](https://bbs.kanxue.com/upload/attach/202510/954363_43X5YMVV3U4H53C.webp)

修复完成
----

丢进 ida 直接分析, 完美不报错,, 就是导入函数没有, 但是影响不大, 一边调试, 一边命名 -- 静态分析青春版

还请各位大佬指点一下, 这种导入函数看不到是不是 plt, rela 没修好的原因 ?

![](https://bbs.kanxue.com/upload/attach/202510/954363_BCH7KXFRMJJBRRU.webp)

正常 f5, 但是因为没有 import 函数, 很多需要函数调用需要手动重命名

![](https://bbs.kanxue.com/upload/attach/202510/954363_2ETVD5A7DUF4AWC.webp)

截至这里, 真实 so 就修复好了

调试 / 分析真实 so 逻辑
===============

分析. init.array 函数  

--------------------

linker 加载完 so 就到了. init.array 函数, 这里分析一下. init.array 先  

到了真实 elf 以后就全都是 thumb 指令集了, getEnv 还是没取到东西, 直接返回

![](https://bbs.kanxue.com/upload/attach/202510/954363_DR78P6UCBYJKGRM.webp)

剩下几个. init.array 都长这样, 暂且跳过分析

![](https://bbs.kanxue.com/upload/attach/202510/954363_NT9FUJYPX9VGNGH.webp)

分析 JNI_OnLoad 函数
----------------

前面 sub_765B163C 的函数 call 完. init.arry, 出来后就到了调用了真实 elf 的 JNI_OnLoad

![](https://bbs.kanxue.com/upload/attach/202510/3NA48MGGA3FKUK9.jpg)

壳子 So 符号覆盖答案揭晓
--------------

但是为什么还是 dlsym 找 JNI_OnLoad 呢 ? 明明是自实现 linker 加载的 elf, 怎么可能通过 dlsym 找到非 linker 加载的 elf 的函数呢 ?

![](https://bbs.kanxue.com/upload/attach/202510/954363_PCK93CCPHZ4Y357.webp)

答案揭晓, 这里的 memcpy 符号表, 让 linker 在壳子 so 找 JNI_OnLoad 符号的时候, 返回已经修改后的 JNI_OnLoad 函数

此时, 壳子 so 的符号表中的 JNI_OnLoad 已经指向了真实 elf 的 JNI_OnLoad

![](https://bbs.kanxue.com/upload/attach/202510/954363_J2QMZVBYMQYZD5R.webp)

跟入分析, 进去后发现就是 thumb 指令集了, 大概还原了一下

![](https://bbs.kanxue.com/upload/attach/202510/954363_MU7JG23HQYEH6EB.webp)

初始化了很多字符串

![](https://bbs.kanxue.com/upload/attach/202510/954363_8XWKHBJNSXZFPEV.webp)

再注册了 native 函数, 然后就等待 java 调用了

![](https://bbs.kanxue.com/upload/attach/202510/954363_9NA82UFNYCG4B6D.webp)

全部打上断点, 等待调用

![](https://bbs.kanxue.com/upload/attach/202510/954363_PWAUXBS5JQ6PS73.webp)

分析 load 函数  

-------------

第一个跑到了 load 函数, 分析一下

![](https://bbs.kanxue.com/upload/attach/202510/954363_G6YSUXKEGZ8HRU3.webp)

跟入 sub_7660B16C, 非常明显了, 进入下一个

![](https://bbs.kanxue.com/upload/attach/202510/954363_QRRRS8ZAZUU7HZR.webp)

跟入 sub_7660C91C, 呜呜呜, 调试一半手滑点错了, 程序死了, 还好一边调试, 一遍改 idb

判断 jvm 版本是不是 2. 开头

![](https://bbs.kanxue.com/upload/attach/202510/954363_JC8XQJDQ3JKNN2S.webp)

跟入 sub_76717470, 注册 receiver

![](https://bbs.kanxue.com/upload/attach/202510/954363_EJUDQ6SZEUJ6G2H.webp)

跟入 sub_76725AF4, 注册了 txShell 的 loadClass 函数

![](https://bbs.kanxue.com/upload/attach/202510/954363_YUZUR9YSP4YSUN4.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_ZJ33XT9UKH2XMV8.webp)

这里先不着急看, 先跟着调试看看对于 java.vm.version 的版本的不同分支做了什么

![](https://bbs.kanxue.com/upload/attach/202510/954363_SA5RJU7KED7XZAX.webp)

现在的 load 函数长这样, 大概流程: 

1.  取 buildSdkInt
    
2.  注册 recevier
    
3.  判断 jvm 版本, 进入不同分支加载 dex
    
4.  注册 **MeShell 的 loadClass 函数
    
5.  定位了 artInterpreterToInterpreterBridge 和 art_quick_to_interpreter_bridge 函数
    

![](https://bbs.kanxue.com/upload/attach/202510/954363_8GHQKMKUC35EUGE.webp)

跟入看看这个 sub_CFC4(env, context, 0) 分支, 先从 java 层取几个值

![](https://bbs.kanxue.com/upload/attach/202510/954363_XH8HS7PU6RREG4S.webp)

### 内存解密 Dex

然后寻找 apk 对应的 DexFile, 找到以后在 maps 中取 / data/dalvik-cache/data@app@com.xxxxxx.apk@classes.dex 的起始地址

![](https://bbs.kanxue.com/upload/attach/202510/954363_YMWN2EX8PSSUN6Z.webp)

找到 classes.dex 在内存中的位置以后:

1.  解析 dex, 取到 dex 中 data_size+data_off 的偏移, 也就是 data 之后的位置, 这里应该是原始 dex 的位置
    
2.  修改 clasess.dex 内存区域权限为读写
    
3.  找到原始 dex 的内存地址以后使用 tea 算法解密 dexHeader
    
4.  如果直接修改 classes.dex 内存区域的权限为读写失败就使用保底方案, 读取原始 dex 的大小, 并 mmap 出一片内存, 将原始 dex move 到 mmap 出来的内存区域
    

![](https://bbs.kanxue.com/upload/attach/202510/954363_GRTCA2BSADBRHZA.webp)

这里取原始 dex 地址的时候做了一个页对齐的操作, 向后对齐

![](https://bbs.kanxue.com/upload/attach/202510/954363_3D77ENZ4HNHVZEY.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_97PM4AK3DPXFZHW.webp)

解密 dexHeader 以后就可以尝试 dump 出来并且还原原始 dex 了

![](https://bbs.kanxue.com/upload/attach/202510/954363_EG47KEUCMTVSTW7.webp)

010editor 去掉没用的头部, 粘贴原始 dexheader 以后丢进 jdax 发现就可以正常解析了

![](https://bbs.kanxue.com/upload/attach/202510/954363_MA6EUZJDMPH2T5H.webp)

发现还是有注入 data 到 dex 中, 说明不止一个 dex, 后面应该还有别的 dex, 暂且先不管, 继续跟着跑

![](https://bbs.kanxue.com/upload/attach/202510/954363_XYGJK865Y5Y5AH5.webp)

### 加载 Dex  

拼接了 mixSoPath 和 mixDexPath 以后就开始解密并加载所有 dex 了

![](https://bbs.kanxue.com/upload/attach/202510/954363_JEA39YD7MBSZ3H7.webp)

先加载 mix.dex (空壳 dex, 只有类), 然后取了 mix.dex 的 DexFile 的 mCookie 去内存解析, 一路找到 DvmDex 结构体指针

![](https://bbs.kanxue.com/upload/attach/202510/954363_7Q39EAUH5PFYDBV.webp)

然后解析了原始 dex header, 重新 pack 了结构体

![](https://bbs.kanxue.com/upload/attach/202510/954363_KN5DDVUGBXBC6GR.webp)

```
struct dexHeader
{
int unused;
int memAddr;
int strIdxTab;
int typeIdxTab;
int fieldIdxTab;
int methodIdxTab;
int protoIdxTab;
int clsDefIdxTab;
int linkData;
char unused3[8];
int memAddr3_Normal;
char unused2[44];
int memAddr2_Android2;
};

```

dexCreateClassLookup, 创建类查找表

![](https://bbs.kanxue.com/upload/attach/202510/954363_ZTNMY627G2MHPSP.webp)

填充 DvmDex 结构体,  

![](https://bbs.kanxue.com/upload/attach/202510/954363_K85A7GXY2WYHRGY.webp)

将所有 DexFile 放进 elements 数组以后, 替换原 dexElements 数组

![](https://bbs.kanxue.com/upload/attach/202510/954363_BNAK9AM76TF5JYC.webp)

然后函数就 return 退出了, 回到了 load 函数最后一个部分

![](https://bbs.kanxue.com/upload/attach/202510/954363_ZEQF2JMVURNAB9F.webp)

没什么太麻烦的点, 直接上图, **实际上这里是没有跑到的, 我的样本 reg native 返回 0, 然后就退出了**

![](https://bbs.kanxue.com/upload/attach/202510/954363_QRNBRRPYM5E949S.webp)

这里还 check 了是不是 yunos, android 4.4 时代的 yunos, 我搜了搜貌似是阿里做的定制版系统, 可惜没有见过

![](https://bbs.kanxue.com/upload/attach/202510/954363_UBZSFFFDYKUMJMV.webp)

定位了 artInterpreterToInterpreterBridge 和 art_quick_to_interpreter_bridge 函数

![](https://bbs.kanxue.com/upload/attach/202510/954363_6MK86VZ3PUKWSY6.webp)

分析 changeEnv 函数
---------------

load 函数结束以后, 直接 f9 就来到了 changeEnv 函数

函数逻辑比较简单, remove 了 mAllApplications 中的 mInitialApplication, 也就是加固自身的 application

![](https://bbs.kanxue.com/upload/attach/202510/954363_NC5K58EDV4T334A.webp)

再替换原始 application

1.  修改 mApplicationInfo 的 clsName 为原始 application name
    
2.  通过 loadedApk.makeApplication 构造原始 application
    
3.  替换 activityThread 的 mInitialApplication 为原始 application
    

![](https://bbs.kanxue.com/upload/attach/202510/954363_E84MCXTJMAJ9E2A.webp)

分析 runCreate 函数  

------------------

非常简单, 调用了原始 application 的 onCreate 就结束了  

![](https://bbs.kanxue.com/upload/attach/202510/954363_FDWPCGFR5DVGWKX.webp)

还有两个函数, 分别是 receiver 和 txEntries, 没有调用到, 函数体也比较简单, 贴个反汇编把

![](https://bbs.kanxue.com/upload/attach/202510/954363_A6SWVU8SS4P3Y7D.webp)

![](https://bbs.kanxue.com/upload/attach/202510/954363_5KTW32VNC3768TC.webp)

接着 f9 几下, 程序就正常进入了

小结  

=====

一路调下来感觉最惊艳我的还是自实现 linker 加载真实 elf, 真是麻雀虽小, 五脏俱全, 只用了大概一两百行就做出了完整加载实现, 受益匪浅.

还有解密壳子 so 的操作, 通过地址页边界对齐, 再逐次减去页大小向下寻找 elf 头, 真是非常骚的操作

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 2 天前 被 SharkFall 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm)