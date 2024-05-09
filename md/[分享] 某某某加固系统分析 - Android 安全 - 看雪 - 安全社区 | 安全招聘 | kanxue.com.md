> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281656.htm)

> [分享] 某某某加固系统分析

某某某加固系统分析

         依然是四年前的分析总结，时过境迁，应该没啥价值了，留作纪念！                                      

某某某加固系统内核 so dump 和修复：
======================

某某某加固系统采取了内外两层 native 代码模式，外层主要为了保护内层核心代码，从分析来看外层模块主要用来反调试，释放内层模块，维护内存模块的某些运行环境达到防止分离内外模块，另外由于内层模块不是通过系统加载的，所以实现了自主的 ELF 加载，这样就实现内层模块的加密处理。这些实现后就可以依赖内层模块保护 dex。从而达到系统保护目的。

对于某某某加固系统的反调试网上已经有很多资料了，就不再作为重点介绍了。下面主要介绍对内层模块的加载，加密，保护。这些外面资料不多的内容。

外层解密内层核心模块的解密算法是使用 zlib 库的 uncompress 函数实现的，不过解密函数解密出来的并不是整个的模块，而是被加密了或者说被移除了四个部分的模块，包含：

program_header_table、.rel.dyn、.rel.plt、Dynamic Segment 。由于是自己的加载系统加载，所以这些被 move 的部分依赖父模块组装，防止内存直接 dump 出解密的内层模块。

```
下面给出frida dump这些数据的脚本，并加以说明：
Java.perform(function () {
  
     var i = 0;
     var phadd = 0;
     var jmprel = 0;
     var rel = 0;
     var dynadd = 0;
     var buff = 0;
     console.log("begin")
     var fileclass = Java.use("java.io.File");
     var mysavePath = "/data/data/" + pkg_name + "/myso";
     var pathDir = fileclass.$new(mysavePath);
     if (!pathDir.exists()) {
         pathDir.mkdirs();
     }
     console.log("mysavepath:"+pathDir)
     Interceptor.attach(Module.getExportByName('libz.so', 'uncompress'), {
         onEnter: function (args) {
             if (i == 0) {
                 if (args[2] != null) {
                     var memcpy_add = Module.findExportByName("libc.so", "memcpy");
                     console.log("memcpy:" + memcpy_add);
                     Interceptor.attach(memcpy_add, {
                         onEnter: function (args) {
                             console.log("begin:memcpy,len:" + args[2]);
                             console.log(hexdump(args[1]));
                             if (args[2] == 0x100) {
                                 // program_header_table 的大小一般是固定的
                                 phadd = args[0];
                                 console.log(hexdump(args[1]))
                             }
  
                             if (args[2] == 0x948) {
                                 //.rel.plt 数据;
                                 jmprel = args[0];
                                 console.log(hexdump(args[1]))
                             }
  
                             if (args[2] == 0x4a58) {
                                 //.rel.dyn 数据
                                 rel = args[0];
                                 console.log(hexdump(args[1]))
                             }
  
                             if (args[2] == 0xd8) {
                                 //Dynamic Segment  数据也是一般固定长度
                                 dynadd = args[0];
                                 console.log(hexdump(args[1]))
                             }
  
                             if (args[2] == 0xbbff4) {
                                 //这个就是某某某加固的加载，加载基地址就是copy到的空间地址，这个数据是经过解密的移除上面部分的模块数据;
  
                                 console.log(hexdump(args[1]));                                //这个hexdump中可以看到ELF头，所以是个标记，上面顺序就可以确定了。
                                 var new_so_base = args[0];
                                 console.log("newso_base:" + args[0])
                                 
                                          
                                 }
                              
                             if (args[2] == 0x38fc) {
                                 //到这里上面得到被move的四个部分数据也已经解密出来了，在这里就可以开始dump了
                                 console.log(args[0])
                                 var file_path = pathDir + "/mydump_" +phadd+ ".dat";
                                 console.log("----save begin1-----");
                                 var file_handle = new File(file_path, "wb");
                                 console.log("----save begin2-----");
                                 if (file_handle) {
                                     console.log("----save begin3-----");
                                     Memory.protect(ptr(phadd), 0x100, 'rwx');
                                     var libso_buffer = ptr(phadd).readByteArray(0x100);
                                     file_handle.write(libso_buffer);
                                     file_handle.flush();
                                     file_handle.close();
                                     console.log("[dump]:"+ file_path);
                                 }
  
                                 var file_path = pathDir + "/mydump_" +jmprel+ ".dat";
                                 console.log("----save begin1-----");
                                 var file_handle = new File(file_path, "wb");
                                 console.log("----save begin2-----");
                                 if (file_handle) {
                                     console.log("----save begin3-----");
                                     Memory.protect(ptr(jmprel), 0x948, 'rwx');
                                     var libso_buffer = ptr(jmprel).readByteArray(0x948);
                                     file_handle.write(libso_buffer);
                                     file_handle.flush();
                                     file_handle.close();
                                     console.log("[dump]:"+ file_path);
                                 }
  
                                 var file_path = pathDir + "/mydump_" +rel+ ".dat";
                                 console.log("----save begin1-----");
                                 var file_handle = new File(file_path, "wb");
                                 console.log("----save begin2-----");
                                 if (file_handle) {
                                     console.log("----save begin3-----");
                                     Memory.protect(ptr(rel), 0x4a58, 'rwx');
                                     var libso_buffer = ptr(rel).readByteArray(0x4a58);
                                     file_handle.write(libso_buffer);
                                     file_handle.flush();
                                     file_handle.close();
                                     console.log("[dump]:"+ file_path);
                                 }
  
                                 var file_path = pathDir + "/mydump_" +dynadd+ ".dat";
                                 console.log("----save begin1-----");
                                 var file_handle = new File(file_path, "wb");
                                 console.log("----save begin2-----");
                                 if (file_handle) {
                                     console.log("----save begin3-----");
                                     Memory.protect(ptr(dynadd), 0xd8, 'rwx');
                                     var libso_buffer = ptr(dynadd).readByteArray(0xd8);
                                     file_handle.write(libso_buffer);
                                     file_handle.flush();
                                     file_handle.close();
                                     console.log("[dump]:"+ file_path);
                                 }
  
  
                             }
                             if (args[2] == 0xb4) {
                                 //console.log(JSON.stringify(this.context))
                                 console.log(args[0])
                                 var file_path = pathDir + "/mydump_" + buff+".so";
                                 console.log("----save begin1-----");
                                 var file_handle = new File(file_path, "wb");
                                 console.log("----save begin2-----");
                                 if (file_handle) {
                                     console.log("----save begin3-----");
                                     Memory.protect(ptr(buff), 0xc0000, 'rwx');
                                     var libso_buffer = ptr(buff).readByteArray(0xc0000);
                                     file_handle.write(libso_buffer);
                                     file_handle.flush();
                                     file_handle.close();
                                     console.log("[dump]:"+ file_path);
                                 }
                             }
                 }
  
             },
              
  
         onLeave: function (retval) {
             // simply replace the value to be returned with 0
  
         }
     })
  
 })

```

上面代码只是说明功能，扣出来的，需要自己整理下可能才能执行。

脚本的原理就是根据某某某加固流程，父模块使用 uncompress 解压后会把解压出来的被偷走的数据重新解密到新的内存地址，在 memcpy 时得到内存地址和长度，然后等解密出来后 dump 数据。

另外是根据数据的大小取相关数据的，每个 APP 可能会不同，需要先跑下看看。

需要说明下，首先跑下 hook uncompress 后的 memcpy hexdump，memcpy 加载的新地址数据出现 ELF 头数据的，表明加载了。然后向上逆推其他数据，这样就能确定每个的数据大小，然后更改脚本，获取数据，并 dump 下来。

比如本例：

begin:memcpy,len:0xbbff4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9085589 7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 .ELF............  
c9085599 03 00 28 00 01 00 00 00 00 00 00 00 34 00 00 00 ..(.........4...  
c90855a9 bc fa 0b 00 00 00 00 05 34 00 20 00 08 00 28 00 ........4. ...(.  
c90855b9 19 00 18 00 11 a6 dd 35 da cf 22 1a 71 b7 8b 08 .......5..".q...

由于父模块需要内存偏移修正，所以完整的模块需要在后面的一次才能 dump。

拿到了需要的 so 模块数据，我们需要修复这个 so 模块，否则 ida 无法分析，用 010Editor 也会打开失败。下面进入 so 修复：

使用的工具有 010Editor 和 ELFfix

ELFfix 修复原理具体请参考：  
[https://bbs.pediy.com/thread-192874.htm](https://bbs.pediy.com/thread-192874.htm)

用 010Editor 打开 dump 的 so 模块，会提示错误。

我们已我们已经知道被移除了填充杂乱数据的几个关键的部分。所以肯定不能正常加载。当然修复是需要理解下 ELF 的文件格式和内存加载原理。这样更便于理解为啥需要这样步骤修复。这个各位自己学习学习了。

首先我们需要还原 program_header_table ，这个是系统解析加载的关键数据，用 010Editor 打开我们刚才 dump 出来的 0x100 字节的那个数据

![](https://bbs.kanxue.com/upload/attach/202405/981_JGJZKJB9REZJCBJ.webp)

复制并覆盖到 dumpso 的 program_header_table 里面，

![](https://bbs.kanxue.com/upload/attach/202405/981_PN24MV6SF7ST72P.webp)  

这样我们就可以正确的打开了，找到这个表 (RW_) Dynamic Segment

![](https://bbs.kanxue.com/upload/attach/202405/981_4WYQ6AXTSMUZCP6.webp)

我们把 dump 的 Dynamic Segment 数据也用 010Editor 打开，就是大小 0xd8 的那个

![](https://bbs.kanxue.com/upload/attach/202405/981_2RJAZWWGF5SE2VW.webp)  

找到这个表的位置：

![](https://bbs.kanxue.com/upload/attach/202405/981_5658QYDWQGM4TQY.webp)

跳转到这个位置：

![](https://bbs.kanxue.com/upload/attach/202405/981_22X96UU337BY3MJ.webp)

复制数据并覆盖：

![](https://bbs.kanxue.com/upload/attach/202405/981_AYT97BED9QAA9NU.webp)

好了，到这里我们完成 ELFfix 需要的关键数据，保存这个文件，然后可以使用 ELFfix 工具进行修复了

复制到 ELFfix 目录下，并执行相关的修复命令会得到修复后的文件：

  
![](https://bbs.kanxue.com/upload/attach/202405/981_32QUE2WUE45GAJQ.webp)

dump_new_full.so ß 修复后的 so

再次用 010Editor 打开这个修复后的 so，还是会提示错误。不过打开后 section_header_table 已经正确了，原来的 section_header_table 是错误的：

![](https://bbs.kanxue.com/upload/attach/202405/981_JC8S6782B8RS5FZ.webp)

修复后已经正确解析出来了：

![](https://bbs.kanxue.com/upload/attach/202405/981_6ZCVQCDCP6JMNFE.webp)

同时 dynamic_symbol_table 表也出现了，打开看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_AYXAXJNGSBRZFE2.webp)

来说下为啥先要修复 program_header_table 节和这个段里面的 Dynamic Segment，根据 ELFfix 作者文章里面说明，修复是依赖这两个数据进行解析得到 ELF section，所以必须先还原这两个数据块。

下面开始还原 rel 数据，一个是 jmprel，一个是 rel

我们打开这两个节：

![](https://bbs.kanxue.com/upload/attach/202405/981_MXBY4FKVWDED43F.webp)

看到这个长度了吧，我们 dump 了，用 010Editor 打开 dump 的数据，然后 010Editor 中转到偏移地址中，用 dump 的解密数据还原。

![](https://bbs.kanxue.com/upload/attach/202405/981_9Y9E5XCTDWR94J7.webp)

同样还原下面数据：

![](https://bbs.kanxue.com/upload/attach/202405/981_WRQH8FE9BBB8M2F.webp)

到这里内层的 so 模块被修复还原出来了，IDA 加载完全没问题了，得到这个模块我们就可以用 IDA 分析调试了。当然也可以脱壳使用了。

![](https://bbs.kanxue.com/upload/attach/202405/981_KJMPQSDX6RZB69V.webp)

跟内存调试的比较看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_WB7X5338PMFH8JB.webp)

注意：目前这个 ELFfix 不支持 64 位程序修复。

好了，有了这个就可以详细的分析某某某内层模块的功能，当然也可以 patch 代码等操作了。

下面来看看某某某加固系统如何来保护内层 so 功能模块的。比较有特色

Java 层偷 Native 层代码
------------------

         某某某加固为了保护内层 so 不能脱离外层 so 环境，做了一些防范措施，这种也是，通过移除关键部位的二进制代码，如果脱离了整个环境，那么就会执行错误，被移除的二进制代码丢失。

```
def getBytes(self,mu,**kwargs):
     #采坑：这个是某某某偷代码的一种方式，把代码先偷走，然后再放回来，模拟这个的时候参数中没有出现这个值。    str = "B54FF67474E8BF10"
     dat = bytearray.fromhex(str)
     return dat

```

内层在 Java 层使用 getByte 这个函数获取被偷走的二进制代码数据，填充回 so 的执行中。

被偷的数据：

**>dump 0x10004b1**

**010004B1: 00 00 00 00 00 00 00 00  BD 10 B5 4F F6 75 74 E8  ...........O.ut.**

**010004C1: BF 10 BD 10 B5 4F F6 76  74 E8 BF 10 BD 10 B5 4F  .....O.vt......O**

![](https://bbs.kanxue.com/upload/attach/202405/981_73SMFZS2WS9D8HV.webp)

来看看这个是啥函数，原来是：JNIEnv->CallStaticObjectMethodV

这种 Java 层偷函数二进制执行代码的方式还是比较新颖的。

关键参数放父模块，通过导出函数调用
-----------------

一般模块的导出函数都是给其他模式引用的接口，某某某这个导出函数却是调用父模块的接口，因为子模块的加载完全是父模块负责的，所以这个接口的填充是父模块做的，所以如果脱离了父模块，这个函数就变成了空的。而且这个函数还是个特别的核心函数，来看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_3NM6ZD57SQ3M75D.webp)

获取 key，这个函数是获取解密算法 rc4 的 key 的，如果没有这个解密 key，后面的 dex 解密都会失败。所以必须到父模块中执行。

通过查询有两个地方调用：

![](https://bbs.kanxue.com/upload/attach/202405/981_N5T2SQ52887GFTJ.webp)

通过分析发现这个函数被填充为父模块的‘_Z9__arm_a_2PcjS_Rii’ 这个导出函数：

![](https://bbs.kanxue.com/upload/attach/202405/981_HMCST6Y2YW5DV8H.webp)

这个函数是个多功能函数，根据不同的传入参数，执行不同的功能。很复杂的一个校验，计算，antidebug 等集成函数。

![](https://bbs.kanxue.com/upload/attach/202405/981_YCRZGVZZMMB6VSH.webp)

这个就是某某某自己实现的类似乱序处理过的多功能集成函数。

来看看 key 计算的时候参数是啥：

======================= Registers =======================

  R0=0x20917ec  R1=0x350  R2=0x2024006  R3=0x2024018

R4=0x2024000  R5=0x2024006  R6=0x0  R7=0xcbcca6e8  R8=0x350

R9=0x350  R10=0x2024018  R11=0x202c350  R12=0x80000000  SP=0x7ffad8

LR=0xcbda5859  PC=0xcbdb08fc

======================= Disassembly =====================

0xcbdb08fc:    blx   r7

020917EC: 22 39 52 52 54 52 52 52  42 52 52 52 13 02 02 19  "9RRTRRRBRRR....

020917FC: 17 0B 60 30 60 31 6A 61  66 67 33 6A 65 61 64 6A  ..`0`1jafg3jeadj

0209180C: 62 34 22 39 52 52 5A 52  52 52 53 52 52 52 01 36  b4"9RRZRRRSRRR.6

0209181C: 39 17 3C 26 20 2B 63 22  39 52 52 5E 52 52 52 4E  9.<& +c"9RR^RRRN

0209182C: 52 52 52 33 31 26 3B 24  3B 26 2B 1C 33 3F 37 31  RRR31&;$;&+.3?71

0209183C: 3D 3F 7C 33 3B 35 28 7C  27 3B 7C 30 33 21 37 7C  =?|3;5(|';|03!7|

执行下这个函数，得到的 key：

>dump 0x2024006

02024006: 67 5E 7F 35 70 37 78 2E  7D 22 75 27 08 56 4A A1  g^.5p7x.}"u'.VJ.

上面的参数哪里来的呢？

分析发现原来是从原始包里面用 libz 解压出来的：

Executing syscall openat(ffffff9c, 02029000, 00020000, 00000000) at 0xcbc28be4

path:/data/app/xxxxxxxx-1/base.apk

这个解出来就是原始包里面的 classes.dex 部分：

![](https://bbs.kanxue.com/upload/attach/202405/981_MEM46G2QG2JRV6B.webp)

另外一个地方的调用参数如下：

======================= Registers =======================

  R0=0x7ffbf9  R1=0x0  R2=0x2024040  R3=0x7ffb10

R4=0xcbc761c8  R5=0x7ffbf8  R6=0x202b054  R7=0xcbde56df  R8=0xcbc761c8

R9=0x202c34c  R10=0x0  R11=0x2024000  R12=0x2024040  SP=0x7ffb08

LR=0xcbcca6e8  PC=0xcbdb0982

======================= Disassembly =====================

0xcbdb0982:   blx  r2

>dump 0x7ffbf9

007FFBF9: 00 DA CB 54 B0 02 02 DF  56 DE CB C8 61 C7 CB 00  ...T....V...a...

007FFC09: 00 00 00 2D 00 00 00 00  60 02 02 00 70 02 02 F0  ...-....`...p...

这是个校验调用，如果脱离父模块就会死在这个调用里面，堆栈会被破坏掉。

当然还有其他类似的这样调用父模块校验：

![](https://bbs.kanxue.com/upload/attach/202405/981_U4SNUCGAW5YMCSM.webp)

DEX 文件保护
--------

某某某加固系统最重要的功能应该就是为了对 dex 文件的保护了，因为一般得到原始的 dex 后，去掉加固模块就可以直接运行。相当于完整脱壳了。为了达到这个目的，某某某加固系统对 dex 的保护特别重要。前面的这些对模块的保护最终的目的也是为了保护 dex 文件。下面来看看他的保护是怎么样的。

某某某加固系统第一次会把原始包中的 classes.dex 用 libz 函数解压出来，然后系统会把他重新编译成 oat 文件，这个文件中的 dex 头被加密处理了。里面的数据部分也被加密处理。在运行的时候，加固系统直接解内存中加载的 classes.oat 文件，而不用再次解压原始文件了。这样带来了效率的提升。也保证了没有明文的 dex 存在磁盘上。

数据的解密用到了 rc4 算法，算法的 key 刚才已经说过了是从父模块的函数中获取的，保证了解密的安全性。

rc4 算法部分大家可以网上找资料看看，用 key 生成 0x100 的解密盒子。然后用这个盒子去解密数据。

nt __fastcall rc4(int result, _BYTE *a2, int a3)

{

  int v3; // lr

  int v4; // r12

  int v5; // r5

  if (a3)

  {

    v3 = 0;

    v4 = 0;

    do

    {

      --a3;

      v3 = (v3 + 1) % 256;

      v5 = *(unsigned __int8 *)(result + v3);

      v4 = (v4 + v5) % 256;

      *(_BYTE *)(result + v3) = *(_BYTE *)(result + v4);

      *(_BYTE *)(result + v4) = v5;

      *a2++ ^= *(_BYTE *)(result + (unsigned __int8)(*(_BYTE *)(result + v3) + v5));

    }

    while (a3);

  }

  return result;

}

算法核心部分。

下面是跟踪的截图和注释：这个循环保证解密 oat 中的所有 dex 文件：

![](https://bbs.kanxue.com/upload/attach/202405/981_DVYG5W3A5PTBK83.webp)

使用内存中的 oat 文件，避免出现明文 dex 在磁盘上。也提高了效率。

![](https://bbs.kanxue.com/upload/attach/202405/981_S2W46F9HC6WMWGP.webp)

下面是查询每个 dex 的开始地址：

![](https://bbs.kanxue.com/upload/attach/202405/981_UJ9KFMYRPH5NUX4.webp)

解密出 dex 头：

![](https://bbs.kanxue.com/upload/attach/202405/981_M2BN8HZZTCPYAD4.webp)

进入 rc4 解密函数，rc4 解密其中被加密的数据，而不是整个的 dex 文件：

![](https://bbs.kanxue.com/upload/attach/202405/981_764KDYHS7RKGUQA.webp)

下面就已经解密出来加密的数据了：

![](https://bbs.kanxue.com/upload/attach/202405/981_NKM5J9PDZGPWQCX.webp)

所有的 oat 文件中的 dex 都被解密出来后，需要立即保存下来，否则某某某加固系统会把 dex 的头再次删除，我们在这个地方先 dump 下来 oat 文件。

![](https://bbs.kanxue.com/upload/attach/202405/981_VRCA5H2JWGSKK6E.webp)

Dump 这个已经解密的 oat 文件后，我们就可以把其中的 dex 文件都找出来：

根据 oat 文件结构知道，dex 数据是从 0x1000 开始的，前面是头，后面接着就是 dex 文件，从我们刚才跟踪的时候知道，这个 oat 中有三个 dex 文件。向下找下看看，也可以根据刚才 R5 中的偏移找到开始的位置，第一个位置偏移是 0x1808，这个里面的 dex 头保存完整：

![](https://bbs.kanxue.com/upload/attach/202405/981_BASBYC6VRXMJEG6.webp)

从 dex 结构我们知道，开始的偏移 + 0x20 后面的 4 个字节就是长度，第一个长度为：0x60E694，把这个数据保存出来，然后继续这样找其他的 dex：

![](https://bbs.kanxue.com/upload/attach/202405/981_BWTBZJRNANJ3MPC.webp)

全部找到后我们用解析工具打开看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_JV4GJWXRVJXFWCG.webp)

这样原始的 dex 就被我们抓出来了。

Dump dex 原理：

       某某某加固通过 hook 系统的 LoadDexFile 在系统加载 dex 文件之前把修改部分的字节还原后交给系统使用。所以在这个时候获取的 dex 文件是正确的。

某某某的 VMP 保护：
============

       某某某加固系统的核心保护就是对 dex 代码中的 onCreate 函数进行 VMP 保护。把原指令用自己实现的指令集替换，并在 Native 层实现这些指令，达到替换指令集，移除 dex 代码，加密指令数据等。并且在 vmp 的代码中还可以插入自己的检测代码。非常有特色，保护能力也非常强，下面就来仔细的研究下某某某 vmp 的实现原理和具体实现方法。

来看这个 app 的例子：

通过 frida hook RegisterNatives 脚本, 获取加固系统注册 Natives 方法：

```
if (addrRegisterNatives != null) {
        Interceptor.attach(addrRegisterNatives, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                console.log("Register_address:", call_add, "offset:", ptr(call_add).sub(new_so_base));
                console.log("[RegisterNatives] method_count:", args[3]);
  
                var methods_ptr = ptr(args[2]);
                var call_addr = this.returnAddress;
                var method_count = parseInt(args[3]);
                for (var i = 0; i < method_count; i++) {
                    var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));
                    var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));
                    var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * 12 + 8));
                    var fuPtr = fnPtr_ptr.sub(1);
                    var name = Memory.readCString(name_ptr);
                    var sig = Memory.readCString(sig_ptr);
                    var module_base = new_so_base;
                    JNI_RegisterNatives_array.push(fuPtr);
                    JNI_RegisterNatives_array.push(name);
                    console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "fnPtr:", fuPtr, "offset:", ptr(fnPtr_ptr).sub(module_base));
                    if (name == "onCreate") {
                        console.log(hexdump(fnPtr_ptr));
                        var code = Instruction.parse(fnPtr_ptr);
                        var next_code = code.next;
                        console.log(code.address, ":", code);
                        for (var i = 0; i < 11; i++) {
                            var next_c = Instruction.parse(next_code);
                            console.log(next_c.address, ":", next_c);
                            next_code = next_c.next;
                        }
                        var onCreate_fun_ptr = next_c.next.add(0xf);
                        var onCreate_fun_addr = Memory.readPointer(onCreate_fun_ptr);
                     console.log("onCreate_function_address:",onCreate_fun_addr,"offset:",ptr(onCreate_fun_addr).sub(module_base));
                    }

```

并获取参数打印如下：

这个 APP 有两个地方被某某某加固使用 VMP 保护：

[RegisterNatives] java_class: com/aigz/ui/base/BBSAppStart name: onCreate sig: (Landroid/os/Bundle;)V fnPtr: 0xd32fb39c  

[RegisterNatives] java_class: com/aigz/ui/base/BBSAppStart name: onCreate sig: (Landroid/os/Bundle;)V fnPtr: 0xe857c39c

来到这两个函数的地方看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_EA6PP95VW923FXU.webp)

确实是个 Native 函数

用 IDA 打开是加密的数据块了：

![](https://bbs.kanxue.com/upload/attach/202405/981_8F99T8Z37SSNZN9.webp)

Method 表头中被定义为 Native 函数：

![](https://bbs.kanxue.com/upload/attach/202405/981_7Z8Y9RTS9YZWTUE.webp)

通过 frida 的跟踪分析，某某某加固系统还隐藏了真实的 vmp 处理函数地址，真正的函数是在入口函数中通过地址跳转过去的，可以通过内嵌的 capstone 引擎反汇编出来，得到真实的函数入口地址：  
0xe857c39c : mov r8, r8

0xe857c39e : sub sp, #0xc

0xe857c3a0 : push {r7}

0xe857c3a2 : push {lr}

0xe857c3a4 : sub sp, #4

0xe857c3a6 : mov ip, r0

0xe857c3a8 : add r0, sp, #0xc

0xe857c3aa : stm r0!, {r1, r2, r3}

0xe857c3ac : add r2, sp, #0xc

0xe857c3ae : mov r0, pc

0xe857c3b0 : mov r1, ip

0xe857c3b2 : str r2, [sp]

onCreate_function_address: 0xd2858985 offset: 0x1c985

这样就定位到真实的函数偏移是 0x1c985

通过比较长时间的分析，大概的了解了这个函数的参数如下：

R0：加密函数的 key，这个 key 决定了后面的 jump table 的跳转，也决定了后面使用的参数的计算和资源的获取。也就是说某某某的 VMP 引擎是同一个，就是通过这个 key 识别处理不同的函数。

R1：jni_env

R2: 堆栈参数

Vmp 函数的大概结构是，通过 R0 key 值获取需要处理的函数资源，通过函数的参数值生成后面解密的 xor_key，这个 key 还被一个参数值保护，失去这个参数值就会出现错误。另外这个 key 还被 inker:__dl_rtld_db_dlactivity 值处理，如果被调试，inker:__dl_rtld_db_dlactivity 值不为 0，这个参数也是错误的。

![](https://bbs.kanxue.com/upload/attach/202405/981_PSQS9H8V2GPC7BJ.webp)

所以调试的时候需要修改这个值为 0

Xor_key 的算法：

![](https://bbs.kanxue.com/upload/attach/202405/981_WVW763Z8F9EGC7T.webp)

R7 是加密函数的 dex 参数地址。

一堆校验后进入 opcode vmp 处理函数：

![](https://bbs.kanxue.com/upload/attach/202405/981_RNU4C9DXS9DG36G.webp)

```
for ( *(_DWORD *)v107 += 4; ; *(_DWORD *)v107 = v92 + 2 * v76 )
  {
    while ( 1 )
    {
      while ( 1 )
      {
        while ( 1 )
        {
          while ( 1 )
          {
            v106 = (v107[4] | (v107[4] << 8)) ^ **(unsigned __int16 **)v107;
            v13 = ((int (__fastcall *)(unsigned __int8 *, _DWORD))sub_CC0BDEFE)(v107, (unsigned __int16)v106);
            v14 = v13;
            if ( v13 < 0x7E )
              break;
            if ( v13 >= 0xBE )
            {
              if ( v13 >= 0xDD )
              {
                if ( v13 < 0xEE )
                {
                  if ( v13 >= 0xE6 )
                  {
                    if ( v13 < 0xEA )
                    {
                      if ( v13 < 0xE8 )
                      {
                        if ( v13 != 0xE7 )
                          JUMPOUT

```

下面详细的解释下这个处理函数的过程：

![](https://bbs.kanxue.com/upload/attach/202405/981_MTRWRMNBZABUEGA.webp)![](https://bbs.kanxue.com/upload/attach/202405/981_M2XVMC5MUHTE7ZA.webp)

.text&.ARM.extab:CC0B042A                 LDR.W           R0, [R9]

.text&.ARM.extab:CC0B042E                 LDRB.W          R1, [R9,#4]

.text&.ARM.extab:CC0B0432                 SUBS            R2, R0, R6

.text&.ARM.extab:CC0B0434                 LDRH            R0, [R0]

.text&.ARM.extab:CC0B0436                 ORR.W           R1, R1, R1,LSL#8

.text&.ARM.extab:CC0B043A                 ASRS            R4, R2, #1

.text&.ARM.extab:CC0B043C                 EOR.W           R8, R1, R0

.text&.ARM.extab:CC0B0440                 MOV             R0, R9

.text&.ARM.extab:CC0B0442                 UXTH.W          R11, R8

.text&.ARM.extab:CC0B0446                 MOV             R1, R11

上面这段就是整个处理的核心部分，其中 R9 中是参数：

本次 VMP 运行参数如下:

02029000: EC 9B E5 D2 95 00 00 00  00 52 C6 E5 50 3B 00 00  .........R..P;..

这个参数的的第一个 dword 是被加密函数的加密数据指针，第五个字节是 xor_key，被加密数据需要通过 xor 这个 xor_key 来解密。第三个 dword 是第二参数指针，第二参数包含解密查询指针的低位值和查询表地址。

.text&.ARM.extab:CC0B0448   BL     sub_CC0BDEFE  // 模拟 opcode 查询算法

.text&.ARM.extab:CC0B044C   MOV    R10, R0       // 返回值就是 opcode_v

.text&.ARM.extab:CC0B044E   CMP     R0, #0x7E ; '~'  // 后面就是根据这个返回值处理

实现代码如下：

```
def Get_Opcode_v(vm_index,table):
    """
    args_list: 0xd1c0f790
              0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
    00000000  0c 1e 55 cf 91 00 00 00 00 52 de e4 60 b3 00 00  ..U......R..`...
               0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
    e4de5200  08 78 fa ce 20 ef e4 e4 b9 0d 00 00 00 00 00 00  .x.. ...........
    e4de5210  d0 01 4b e2 00 00 00 00 00 00 00 00 00 00 00 00  ..K.............
    e4de5220  00 00 00 00 48 27 0e e6 48 27 0e e6 01 00 00 00  ....H'..H'......
    e4de5230  58 67 f9 cb 58 26 0e e6 04 00 00 00 b8 dd a1 e7  Xg..X&..........
    e4de5240  88 26 0e e6 04 00 00 00 4c 52 de e4 00 00 00 00  .&......LR......
    e4de5250  00 00 00 00 30 77 c7 d1 20 78 c7 d1 08 00 00 00  ....0w.. x......
    e4de5260  40 8c 4f e2 64 00 00 00 80 54 de e4 d8 54 de e4  @.O.d....T...T..
    var opcode_table = Memory.readPointer(buf.add(8));
    // var table_args = Memory.readByteArray(opcode_table.add(60), 0x10);
    // console.log(table_args);
     console.log(hexdump(opcode_table));
     opcode_table = Memory.readPointer(opcode_table.add(0x60));
     console.log("Opcede_table_addr:", opcode_table);
    """
    #key1 =  0xb某某某 % 0x64 #查询的高位为这个值
    #key1 = 0x24C4 % 0x64
    key1 = 0x3b50 % 0x64
    key1 =  key1 <<8
    key = vm_index % 0x100
    index = key1 ^ key
    opcode_v = int.from_bytes(ReadFile(table, index, 1), byteorder="little", signed=False)
return opcode_v

```

也就是说先把加密函数的两个字节取出来，然后 xor xor_key 然后用解密后的前两个字节进入查询函数进行查询计算。得到查询的 index，根据 index 获取模拟的 opcode 值。

查询表格每个加密函数都有一个属于自己的表：

![](https://bbs.kanxue.com/upload/attach/202405/981_U9ZPM8UW5PEKKQ4.webp)

在上面的查询过程中会使用 inker:__dl_rtld_db_dlactivity 值来破坏查询值，所以必须清零。

![](https://bbs.kanxue.com/upload/attach/202405/981_CKC3G8XKCCZFPR6.webp)

```
看一段运行的日志：
2020-10-13 14:05:20,625    INFO                samples.debug_utils | 本次VMP运行参数如下:
2020-10-13 14:05:20,625    INFO                samples.debug_utils | 
02029000: EC 9B E5 D2 95 00 00 00  00 52 C6 E5 50 3B 00 00  .........R..P;..
  
2020-10-13 14:05:20,628    INFO                samples.debug_utils | vmp_opcode_v:0xb5c1
2020-10-13 14:05:20,628    INFO                samples.debug_utils | vmp_opcode_v_dat:0x2054
2020-10-13 14:05:20,628    INFO                samples.debug_utils | vmp_opcode:0x7a
2020-10-13 14:05:20,629    INFO                samples.debug_utils | ['0x7a']
2020-10-13 14:05:20,629    INFO                samples.debug_utils | opcode run-->
2020-10-13 14:05:20,641   DEBUG            androidemu.java.jni_env | JNIEnv->FindClass(androidx/appcompat/app/AppCompatActivity) was called
2020-10-13 14:05:20,643   DEBUG            androidemu.java.jni_env | JNIEnv->NewGlobalRef(2) was called
2020-10-13 14:05:20,668   DEBUG            androidemu.java.jni_env | JNIEnv->GetMethodId(, onCreate, (Landroid/os/Bundle;)V) was called
2020-10-13 14:05:20,669    INFO            androidemu.java.jni_env | find method_id ->0xd2000188
2020-10-13 14:05:20,684   DEBUG            androidemu.java.jni_env | JNIEnv->DeleteLocalRef(2) was called
call remove  2020-10-13 14:05:20,687    INFO                samples.debug_utils | vmp_opcode_v_ptr:0x3
2020-10-13 14:05:20,687    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 14:05:20,687    INFO                samples.debug_utils | ['0x3']
  
2020-10-13 14:05:20,687    INFO                samples.debug_utils | vmp_opcode_v:0x97ea
2020-10-13 14:05:20,687    INFO                samples.debug_utils | vmp_opcode_v_dat:0x27f
2020-10-13 14:05:20,687    INFO                samples.debug_utils | vmp_opcode:0x12
2020-10-13 14:05:20,688    INFO                samples.debug_utils | ['0x7a', '0x12']
2020-10-13 14:05:20,688    INFO                samples.debug_utils | opcode run-->
2020-10-13 14:05:20,688   DEBUG                samples.debug_utils | 0xcbd7b648: 2ada   bge    #0xcbd7b6a0
2020-10-13 14:05:20,688   DEBUG                samples.debug_utils | 0xcbd7b64a: 3f28   cmp    r0, #0x3f
2020-10-13 14:05:20,688   DEBUG                samples.debug_utils | 0xcbd7b64c: 4cda   bge    #0xcbd7b6e8
2020-10-13 14:05:20,688   DEBUG                samples.debug_utils | 0xcbd7b64e: 1f28   cmp    r0, #0x1f
2020-10-13 14:05:20,689   DEBUG                samples.debug_utils | 0xcbd7b650: 80f2e080      bge.w  #0xcbd7b814
2020-10-13 14:05:20,689   DEBUG                samples.debug_utils | 0xcbd7b654: 0f28   cmp    r0, #0xf
2020-10-13 14:05:20,689   DEBUG                samples.debug_utils | 0xcbd7b656: 80f25f81       bge.w  #0xcbd7b918
2020-10-13 14:05:20,689   DEBUG                samples.debug_utils | 0xcbd7b918: 1728   cmp    r0, #0x17
2020-10-13 14:05:20,689   DEBUG                samples.debug_utils | 0xcbd7b91a: 80f24382      bge.w  #0xcbd7bda4
2020-10-13 14:05:20,690   DEBUG                samples.debug_utils | 0xcbd7b91e: 1328   cmp    r0, #0x13
2020-10-13 14:05:20,690   DEBUG                samples.debug_utils | 0xcbd7b920: 80f23984      bge.w  #0xcbd7c196
2020-10-13 14:05:20,690   DEBUG                samples.debug_utils | 0xcbd7b924: 1128   cmp    r0, #0x11
2020-10-13 14:05:20,690   DEBUG                samples.debug_utils | 0xcbd7b926: 81f2c180      bge.w  #0xcbd7caac
2020-10-13 14:05:20,690   DEBUG                samples.debug_utils | 0xcbd7caac: 74f4c5a8      bne.w  #0xcbd70c3a
2020-10-13 14:05:20,691   DEBUG                samples.debug_utils | 0xcbd70c3a: d9f80010      ldr.w   r1, [sb]
2020-10-13 14:05:20,691   DEBUG                samples.debug_utils | 0xcbd70c3e: 99f80400      ldrb.w  r0, [sb, #4]
2020-10-13 14:05:20,691   DEBUG                samples.debug_utils | 0xcbd70c42: 2a68   ldr      r2, [r5]
2020-10-13 14:05:20,691   DEBUG                samples.debug_utils | 0xcbd70c44: 5b4b   ldr      r3, [pc, #0x16c]
2020-10-13 14:05:20,691   DEBUG                samples.debug_utils | 0xcbd70c46: 8f88   ldrh    r7, [r1, #4]
2020-10-13 14:05:20,691   DEBUG                samples.debug_utils | 0xcbd70c48: 40ea0020      orr.w   r0, r0, r0, lsl #8
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c4c: 4988   ldrh    r1, [r1, #2]
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c4e: 7b44   add    r3, pc
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c50: 4740   eors    r7, r0
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c52: 4840   eors    r0, r1
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c54: c8f30721      ubfx    r1, r8, #8, #8
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c58: c0ea0740      pkhbt  r0, r0, r7, lsl #0x10
2020-10-13 14:05:20,692   DEBUG                samples.debug_utils | 0xcbd70c5c: 42f82100      str.w   r0, [r2, r1, lsl #2]
2020-10-13 14:05:20,693   DEBUG                samples.debug_utils | 0xcbd70c60: b0fa80f0       clz      r0, r0
2020-10-13 14:05:20,693   DEBUG                samples.debug_utils | 0xcbd70c64: 4009   lsrs     r0, r0, #5
2020-10-13 14:05:20,693   DEBUG                samples.debug_utils | 0xcbd70c66: 53f82000      ldr.w   r0, [r3, r0, lsl #2]
2020-10-13 14:05:20,693   DEBUG                samples.debug_utils | 0xcbd70c6a: 8746   mov    pc, r0
2020-10-13 14:05:20,693   DEBUG                samples.debug_utils | 0xcbd70c74: d9f80000      ldr.w   r0, [sb]
2020-10-13 14:05:20,693   DEBUG                samples.debug_utils | 0xcbd70c78: 0630   adds   r0, #6  //opcode len
2020-10-13 14:05:20,694   DEBUG                samples.debug_utils | 0xcbd70c7a: 0af0d0bc      b.w     #0xcbd7b61e
2020-10-13 14:05:20,694   DEBUG                samples.debug_utils | 0xcbd7b61e: c9f80000      str.w   r0, [sb]
2020-10-13 14:05:20,694   DEBUG                samples.debug_utils | 0xcbd7b622: 0f9d   ldr      r5, [sp, #0x3c]
2020-10-13 14:05:20,694   DEBUG                samples.debug_utils | 0xcbd7b624: d9f80000      ldr.w   r0, [sb]
2020-10-13 14:05:20,696    INFO                samples.debug_utils | vmp_opcode_v_ptr:0x6
2020-10-13 14:05:20,696    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 14:05:20,697    INFO                samples.debug_utils | ['0x3', '0x3'] 
```

从这个运行日志中可以看到，某某某加固有两种处理，一种就是对 Java class 的处理，还有一种就是数值处理。

对 Java class 的处理日志能明确的打印出结果，根据结果基本上能知晓函数的作用。而对数值的处理比较麻烦，某某某加固的 vmp 是自己模拟实现的，需要跟踪调试反分析出来函数的作用。

另外 opcode 的长度是某某某加固 vmp 自己维护的。也就是根据 opcode 硬编码的，所以需要记录下来。

现在来总结下某某某加固 vmp 的处理方法：

1.     用需要加密的 onCreate 函数的参数 + 常量参数 + anti 参数生成的 xor_key 加密函数数据。

2.     每个 opcode 的第一个字节被某某某加固 vmp 的模拟 opcode 所替换，并被上面的 xor_key 加密保护。

3.     每个 opcode 类型自己实现功能。

4.     每个 opcode 长度在实现代码中自己维护。硬编码实现。

5.     每个函数生成高地位的参数加密，并和密文的 opcode 计算获得查询 index。

6.     每个函数都拥有一张模拟 opcode_v 查询表。模拟的 opcode 是通过这个表查询得到的。

也就是说模拟的 opcode 是两层加密解密出来的。并且都是自己实现具体功能的。

某某某加固 VMP 代码还原：
===============

首先要想还原得有一个官方的 opcode 文档，推荐：

[http://pallergabor.uw.hu/androidblog/dalvik_opcodes.html](http://pallergabor.uw.hu/androidblog/dalvik_opcodes.html)

![](https://bbs.kanxue.com/upload/attach/202405/981_TWGAVDE6F5ZNA3G.webp)

从官方的文档中大致能知道 opcode 类型和 opcode 长度定义。

从文档中我们也知道 dex 的代码分为 opcode 操作码和后面的操作数组成的。

![](https://bbs.kanxue.com/upload/attach/202405/981_5T4EGGQBZ6PT2MD.webp)

Opcode 占一个字节。后面都是操作数。

而某某某加固的 VMP 简单的加密了整个代码，就是用 xor_key 加密，而重点的保护的就是 opcode 操作码，还原代码其实也就是找回 opcode 操作码，替换回去就行。后面的操作数通过 xor_key 还原出来就可以了。

下面就来重点说说怎么还原操作码的：

从上面的分析来看就是两种类型，处理 Java class 的通过打印的日志基本就清楚是啥功能的。然后通过正常的代码结合官方文档大致就能还原出来。

而对数据的处理比较麻烦，因为都是某某某加固系统自己处理的，所以需要到每个处理单元里面跟踪分析。

当然最好是在 AndroidNativeEm 模拟器中运行，这样能方便跟踪，如果你能用 ida 动态跟踪也可以。

下面是一个 demo 加固后的跟踪结果和分析结果：

```
2020-10-13 11:21:46,942    INFO                samples.debug_utils | 本次VMP运行参数如下:
2020-10-13 11:21:46,942    INFO                samples.debug_utils | 
02029000: EC 9B E5 D2 95 00 00 00  00 52 C6 E5 50 3B 00 00  .........R..P;..
  
2020-10-13 11:21:46,946    INFO                samples.debug_utils | vmp_opcode:0x7a  ==>6F invoke-super
2020-10-13 11:21:46,946    INFO                samples.debug_utils | ['0x7a']
2020-10-13 11:21:46,946    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:46,987   DEBUG            androidemu.java.jni_env | JNIEnv->GetMethodId(, onCreate, (Landroid/os/Bundle;)V) was called
2020-10-13 11:21:46,988    INFO            androidemu.java.jni_env | find method_id ->0xd2000188
2020-10-13 11:21:47,007    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:47,007    INFO                samples.debug_utils | ['0x3']
  
2020-10-13 11:21:47,008    INFO                samples.debug_utils | vmp_opcode:0x12  ==>14 const vx, lit32
2020-10-13 11:21:47,008    INFO                samples.debug_utils | ['0x7a', '0x12']  
2020-10-13 11:21:47,008    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:47,012   DEBUG                samples.debug_utils | 0xcbd70c3a: d9f80010      ldr.w   r1, [sb] data Buff
2020-10-13 11:21:47,012   DEBUG                samples.debug_utils | 0xcbd70c3e: 99f80400      ldrb.w  r0, [sb, #4]   xor key
2020-10-13 11:21:47,012   DEBUG                samples.debug_utils | 0xcbd70c42: 2a68   ldr      r2, [r5]
2020-10-13 11:21:47,012   DEBUG                samples.debug_utils | 0xcbd70c46: 8f88   ldrh    r7, [r1, #4]
2020-10-13 11:21:47,013   DEBUG                samples.debug_utils | 0xcbd70c48: 40ea0020      orr.w   r0, r0, r0, lsl #8
2020-10-13 11:21:47,013   DEBUG                samples.debug_utils | 0xcbd70c4c: 4988   ldrh    r1, [r1, #2]
2020-10-13 11:21:47,013   DEBUG                samples.debug_utils | 0xcbd70c50: 4740   eors    r7, r0   解密数据
2020-10-13 11:21:47,013   DEBUG                samples.debug_utils | 0xcbd70c52: 4840   eors    r0, r1   解密数据
2020-10-13 11:21:47,014   DEBUG                samples.debug_utils | 0xcbd70c54: c8f30721      ubfx    r1, r8, #8, #8
2020-10-13 11:21:47,014   DEBUG                samples.debug_utils | 0xcbd70c58: c0ea0740      pkhbt  r0, r0, r7, lsl #0x10  合并寄存器值 取常量给寄存器
2020-10-13 11:21:47,014   DEBUG                samples.debug_utils | 0xcbd70c5c: 42f82100      str.w   r0, [r2, r1, lsl #2]   R0=0x7f0a001c  
2020-10-13 11:21:47,018    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:47,018    INFO                samples.debug_utils | ['0x3', '0x3']
  
2020-10-13 11:21:47,018    INFO                samples.debug_utils | vmp_opcode:0x1  ==>6e   invoke-virtual 
2020-10-13 11:21:47,018    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1']
2020-10-13 11:21:47,019    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:47,032   DEBUG            androidemu.java.jni_env | JNIEnv->FindClass(com/example/test/MainActivity) was called
2020-10-13 11:21:47,033   DEBUG            androidemu.java.jni_env | JNIEnv->NewGlobalRef(3) was called
2020-10-13 11:21:47,053   DEBUG            androidemu.java.jni_env | JNIEnv->GetMethodId(, setContentView, (I)V) was called
2020-10-13 11:21:47,053    INFO            androidemu.java.jni_env | find method_id ->0xd2000190
2020-10-13 11:21:53,918    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:53,918    INFO                samples.debug_utils | ['0x3', '0x3', '0x3']
  
2020-10-13 11:21:53,918    INFO                samples.debug_utils | vmp_opcode:0x12
2020-10-13 11:21:53,918    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12']
2020-10-13 11:21:53,918    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:53,927    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:53,927    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3']
  
2020-10-13 11:21:53,927    INFO                samples.debug_utils | vmp_opcode:0x1
2020-10-13 11:21:53,927    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1']
2020-10-13 11:21:53,927    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:53,955   DEBUG            androidemu.java.jni_env | JNIEnv->GetMethodId(, findViewById, (I)Landroid/view/View;) was called
2020-10-13 11:21:53,955    INFO            androidemu.java.jni_env | find method_id ->0xd200018c
2020-10-13 11:21:53,968   DEBUG            androidemu.java.jni_env | JNIEnv->CallObjectMethodV(com/example/test/MainActivity, findViewById <(I)Landroid/view/View;>, 0x2033000) was called
2020-10-13 11:21:53,975    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:53,975    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3', '0x3']
  
2020-10-13 11:21:53,975    INFO                samples.debug_utils | vmp_opcode:0x21  ==>0x0c  move-result-object vx
2020-10-13 11:21:53,975    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21']
2020-10-13 11:21:53,975    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:53,978   DEBUG                samples.debug_utils | 0xcbd7c99c: 2746   mov    r7, r4
2020-10-13 11:21:53,978   DEBUG                samples.debug_utils | 0xcbd7c99e: 0a9c   ldr      r4, [sp, #0x28]
2020-10-13 11:21:53,978   DEBUG                samples.debug_utils | 0xcbd7c9a0: 3846   mov    r0, r7
2020-10-13 11:21:53,978   DEBUG                samples.debug_utils | 0xcbd7c9a2: ab46   mov    fp, r5
2020-10-13 11:21:53,978   DEBUG                samples.debug_utils | 0xcbd7c9a4: 4ff00008       mov.w r8, #0
2020-10-13 11:21:53,979   DEBUG                samples.debug_utils | 0xcbd7c9a8: a268   ldr      r2, [r4, #8]
2020-10-13 11:21:53,979   DEBUG                samples.debug_utils | 0xcbd7c9aa: e4f731fe       bl       #0xcbd61610
2020-10-13 11:21:53,979   DEBUG                samples.debug_utils | 0xcbd61610: 70b5   push   {r4, r5, r6, lr}
2020-10-13 11:21:53,979   DEBUG                samples.debug_utils | 0xcbd61612: 0546   mov    r5, r0
2020-10-13 11:21:53,979   DEBUG                samples.debug_utils | 0xcbd61614: c069   ldr      r0, [r0, #0x1c]
2020-10-13 11:21:53,979   DEBUG                samples.debug_utils | 0xcbd61616: 1446   mov    r4, r2
2020-10-13 11:21:53,980   DEBUG                samples.debug_utils | 0xcbd61618: 0e46   mov    r6, r1
2020-10-13 11:21:53,980   DEBUG                samples.debug_utils | 0xcbd6161a: 50f82100      ldr.w   r0, [r0, r1, lsl #2]
2020-10-13 11:21:53,980   DEBUG                samples.debug_utils | 0xcbd6161e: 10b1   cbz     r0, #0xcbd61626
2020-10-13 11:21:53,980   DEBUG                samples.debug_utils | 0xcbd61620: 0068   ldr      r0, [r0]
2020-10-13 11:21:53,980   DEBUG                samples.debug_utils | 0xcbd61622: a042   cmp    r0, r4
2020-10-13 11:21:53,980   DEBUG                samples.debug_utils | 0xcbd61624: 13d0   beq    #0xcbd6164e
2020-10-13 11:21:53,981   DEBUG                samples.debug_utils | 0xcbd6164e: a869   ldr      r0, [r5, #0x18]
2020-10-13 11:21:53,981   DEBUG                samples.debug_utils | 0xcbd61650: 002c   cmp    r4, #0
2020-10-13 11:21:53,981   DEBUG                samples.debug_utils | 0xcbd61652: 40f82640      str.w   r4, [r0, r6, lsl #2]
2020-10-13 11:21:53,981   DEBUG                samples.debug_utils | 0xcbd61656: 14d0   beq    #0xcbd61682
2020-10-13 11:21:53,981   DEBUG                samples.debug_utils | 0xcbd61682: 70bd   pop    {r4, r5, r6, pc}
2020-10-13 11:21:53,981   DEBUG                samples.debug_utils | 0xcbd7c9ae: 84f81080      strb.w  r8, [r4, #0x10]
2020-10-13 11:21:53,982   DEBUG                samples.debug_utils | 0xcbd7c9b2: 3c46   mov    r4, r7
2020-10-13 11:21:53,985    INFO                samples.debug_utils | vmp_opcode_length:0x1
2020-10-13 11:21:53,985    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3', '0x3', '0x1']
  
2020-10-13 11:21:53,985    INFO                samples.debug_utils | vmp_opcode:0xa6 ==0x1f  check-cast vx, type_id
2020-10-13 11:21:53,985    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21', '0xa6']
2020-10-13 11:21:53,985    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:53,993   DEBUG            androidemu.java.jni_env | JNIEnv->FindClass(android/widget/TextView) was called
2020-10-13 11:21:54,004    INFO                samples.debug_utils | vmp_opcode_length:0x2
2020-10-13 11:21:54,004    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3', '0x3', '0x1', '0x2']
  
2020-10-13 11:21:54,005    INFO                samples.debug_utils | vmp_opcode:0x1
2020-10-13 11:21:54,005    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21', '0xa6', '0x1']
2020-10-13 11:21:54,005    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:54,030   DEBUG            androidemu.java.jni_env | JNIEnv->GetMethodId(, stringFromJNI, ()Ljava/lang/String;) was called
2020-10-13 11:21:54,030    INFO            androidemu.java.jni_env | find method_id ->0xd2000194
2020-10-13 11:21:54,041   DEBUG            androidemu.java.jni_env | JNIEnv->CallObjectMethodV(com/example/test/MainActivity, stringFromJNI <()Ljava/lang/String;>, 0x2035000) was called
2020-10-13 11:21:54,048    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:54,048    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3', '0x3', '0x1', '0x2', '0x3']
  
2020-10-13 11:21:54,048    INFO                samples.debug_utils | vmp_opcode:0x21
2020-10-13 11:21:54,048    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21', '0xa6', '0x1', '0x21']
2020-10-13 11:21:54,048    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:54,065    INFO                samples.debug_utils | vmp_opcode_length:0x1
2020-10-13 11:21:54,065    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3', '0x3', '0x1', '0x2', '0x3', '0x1']
  
2020-10-13 11:21:54,065    INFO                samples.debug_utils | vmp_opcode:0x1
2020-10-13 11:21:54,065    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21', '0xa6', '0x1', '0x21', '0x1']
2020-10-13 11:21:54,065    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:54,094   DEBUG            androidemu.java.jni_env | JNIEnv->GetMethodId(, setText, (Ljava/lang/CharSequence;)V) was called
2020-10-13 11:21:54,094    INFO            androidemu.java.jni_env | find method_id ->0xd2000170
2020-10-13 11:21:54,114    INFO                samples.debug_utils | vmp_opcode_length:0x3
2020-10-13 11:21:54,114    INFO                samples.debug_utils | ['0x3', '0x3', '0x3', '0x3', '0x3', '0x1', '0x2', '0x3', '0x1', '0x3']
  
2020-10-13 11:21:54,114    INFO                samples.debug_utils | vmp_opcode:0xc5 ==>0x0e  return-void
2020-10-13 11:21:54,114    INFO                samples.debug_utils | ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21', '0xa6', '0x1', '0x21', '0x1', '0xc5']
2020-10-13 11:21:54,114    INFO                samples.debug_utils | opcode run-->
2020-10-13 11:21:54,117   DEBUG                samples.debug_utils | 0xcbd7d2a6: 0598   ldr      r0, [sp, #0x14]
2020-10-13 11:21:54,117   DEBUG                samples.debug_utils | 0xcbd7d2a8: 099a   ldr      r2, [sp, #0x24]
2020-10-13 11:21:54,117   DEBUG                samples.debug_utils | 0xcbd7d2aa: d0e90001     ldrd    r0, r1, [r0]
2020-10-13 11:21:54,117   DEBUG                samples.debug_utils | 0xcbd7d2ae: c2e90001      strd    r0, r1, [r2]
2020-10-13 11:21:54,117   DEBUG                samples.debug_utils | 0xcbd7d2b2: 64a8   add    r0, sp, #0x190
2020-10-13 11:21:54,117   DEBUG                samples.debug_utils | 0xcbd7d2b4: 0021   movs  r1, #0
2020-10-13 11:21:54,118   DEBUG                samples.debug_utils | 0xcbd7d2b6: 04f026fe       bl       #0xcbd81f06
2020-10-13 11:21:54,118   DEBUG                samples.debug_utils | 0xcbd81f06: 0a46   mov    r2, r1
2020-10-13 11:21:54,118   DEBUG                samples.debug_utils | 0xcbd81f08: 0168   ldr      r1, [r0]
2020-10-13 11:21:54,118   DEBUG                samples.debug_utils | 0xcbd81f0a: 0260   str      r2, [r0]
2020-10-13 11:21:54,118   DEBUG                samples.debug_utils | 0xcbd81f0c: 09b1   cbz     r1, #0xcbd81f12
2020-10-13 11:21:54,118   DEBUG                samples.debug_utils | 0xcbd81f0e: 00f001b8      b.w     #0xcbd81f14
2020-10-13 11:21:54,119   DEBUG                samples.debug_utils | 0xcbd81f14: 49b1   cbz     r1, #0xcbd81f2a
2020-10-13 11:21:54,119   DEBUG                samples.debug_utils | 0xcbd81f16: 10b5   push   {r4, lr}
2020-10-13 11:21:54,119   DEBUG                samples.debug_utils | 0xcbd81f18: 0846   mov    r0, r1
2020-10-13 11:21:54,119   DEBUG                samples.debug_utils | 0xcbd81f1a: 0c46   mov    r4, r1
2020-10-13 11:21:54,119   DEBUG                samples.debug_utils | 0xcbd81f1c: fef752f8        bl       #0xcbd7ffc4
2020-10-13 11:21:54,119   DEBUG                samples.debug_utils | 0xcbd7ffc4:  10b5   push   {r4, lr}
2020-10-13 11:21:54,120   DEBUG                samples.debug_utils | 0xcbd7ffc6:  0446   mov    r4, r0
2020-10-13 11:21:54,120   DEBUG                samples.debug_utils | 0xcbd7ffc8:  007c   ldrb    r0, [r0, #0x10]
2020-10-13 11:21:54,120   DEBUG                samples.debug_utils | 0xcbd7ffca:  38b1   cbz     r0, #0xcbd7ffdc
2020-10-13 11:21:54,120   DEBUG                samples.debug_utils | 0xcbd7ffdc: 0020   movs  r0, #0
2020-10-13 11:21:54,120   DEBUG                samples.debug_utils | 0xcbd7ffde: 2074   strb    r0, [r4, #0x10]
2020-10-13 11:21:54,120   DEBUG                samples.debug_utils | 0xcbd7ffe0: 10bd   pop    {r4, pc}
2020-10-13 11:21:54,121   DEBUG                samples.debug_utils | 0xcbd81f20: 2046   mov    r0, r4
2020-10-13 11:21:54,121   DEBUG                samples.debug_utils | 0xcbd81f22: bde81040     pop.w  {r4, lr} 
```

通过上面的日志分析基本上还原出来代码。

opcode_len_dat = [3,3,3,3,3,1,2,3,1,3,1]

xor_key = 0x9595

opcode_操作码： ['0x7a', '0x12', '0x1', '0x12', '0x1', '0x21', '0xa6', '0x1', '0x21', '0x1', '0xc5']

使用的 opcode 种类： ['0x7a', '0x12', '0x1', '0x21', '0xa6', '0xc5'] 合计个数： 6

恢复的代码： 6f20e70d210014021c000a7f6e20513b210014027e00077f6e204f3b21000c021c0200026e10523b01000c006e20fe0b02000e00

用这个还原的代码修复 dex 文件就可以实现剥离某某某加固，还原原程序了。

Dex 二进制修改方法：
============

用 IDA 打开 dex 文件，搜索 class 名称，找到要修改的方法的地方：

![](https://bbs.kanxue.com/upload/attach/202405/981_BNAUN2TH7VKXU4C.webp)

IDA 没有显示 Native 函数，所以找 onCreate 函数附近的 init 函数：  
  

![](https://bbs.kanxue.com/upload/attach/202405/981_4CCD6SNP6WDYYVE.webp)

看到上面那个没被解析的数据就是被加密的 vmp 后的 onCreate 函数，点击这个找到 Method 定义的地方：

![](https://bbs.kanxue.com/upload/attach/202405/981_P74YHMCBV7Y7GF8.webp)

看到 Native 函数定义的数据了吧。比较下正常的 public 函数定义，知道类型应该是 0004，而不是 0284，后面跟着的是 code 地址，修改这个数据：

直接到 IDA 的 Hex View 页中按 F2 修改：

![](https://bbs.kanxue.com/upload/attach/202405/981_4QBXMWCYN8F9RVM.webp)

把字节 84 修改成 04：

![](https://bbs.kanxue.com/upload/attach/202405/981_2U7AGKDN8KE8Z9C.webp)

下面修改 code offset，这个地址是 LEB128 编码，编码介绍：

[https://berryjam.github.io/2019/09/LEB128(Little-Endian-Base-128) 格式介绍 /](https://berryjam.github.io/2019/09/LEB128(Little-Endian-Base-128)%E6%A0%BC%E5%BC%8F%E4%BB%8B%E7%BB%8D/)

Andorid 系统在 Dex 文件采用 LEB128 变长编码格式，相对固定长度的编码格式，leb128 编码存储利用率比较高，能让 Dex 文件尽可能的小。对于存储空间比较紧缺的移动设备，这非常有用。就是在字节的第七位插入 1，计算地址时，去掉这个第七位的 1 再组合。

![](https://bbs.kanxue.com/upload/attach/202405/981_PNGC848EYHV7UYS.webp)

注意 Hex View 中字节是反的，需要修改的是前面而不是后面的，保存后重新 IDA 打开：

![](https://bbs.kanxue.com/upload/attach/202405/981_BMBZZT2ZEAGH79A.webp)

函数中出现了 onCreate，到 Method 表中看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_EYGAU7W9QFF557V.webp)  

被还原成正常的方法函数了。

下面还原方法函数代码，这个用 IDA 就不方便了，我们使用 winhex 来实现：

![](https://bbs.kanxue.com/upload/attach/202405/981_3NHK6UPMH2MRFE5.webp)

来的偏移地址：0x1b7478 处，从 IDA 中也可以看到前面 0x10 是函数头参数，后面就是代码了：

![](https://bbs.kanxue.com/upload/attach/202405/981_QAWJKE3AQY7MDGR.webp)

我们把还原的代码覆盖到 0x01B7488 处：

![](https://bbs.kanxue.com/upload/attach/202405/981_HWNGK484KWG6CFK.webp)

保存，用 IDA 重新打开：

![](https://bbs.kanxue.com/upload/attach/202405/981_ZTRUD87ERCUVTJ5.webp)

代码还原出来了，然后我们跟原始的 dex 比较下看看：

![](https://bbs.kanxue.com/upload/attach/202405/981_6YMJHGK8BF35RJC.webp)

除了偏移地址发生改变，基本上一样。说明还原成功。

某某某加固修正重新打包技术要点如下：

1. 首先需要使用自己编写的某某某_jiagu_info 脚本跑出 oat 文件的 dex，dump 出来 oat 文件后用 winhex 把 dex 头前面的数据去掉就是 dex 文件。

2. 一般 dump 多个 dex 文件，需要处理所有的 oat 文件得到所有的 dex

3. 一般被加固的 apk 的原始入口在 Manifest xml 中已经改变，需要找回原来的入口，这个尤其重要，这个坑踩了好久。这个入口中脚本中会打印出来，App_Application_Entry is:

4. 由于 Java vm 并不能运行 nop 指令，一般也不可能出现这个指令，所以清除某某某加固的 SDK 时，如果不能进行 code 对齐，中间有 nop 时 Java vm 会执行错误。所以建议保留 SDK，但是修改 SDK 代码为返回原值就行。

某某某加固一般有三种 SDK 插入：

 主 onCreate 函数的 stub->Mark()  这个少，一般只有一个，反编译后直接在 smile 中可以去掉，搜索 dex 中的 mark 就能找到地方。

 onCreate 的 interface11（）     这个由于是在函数的开头插入，所以可以写代码清除并进行 code 对齐。 需要写代码修复。先用 winhex 通过特征码搜索所有的调用地址，然后导出这个搜索结果给下面修复脚本用就能修复好。

脚本见附件：某某某加固 dex 嵌入 SDK 修复脚本。

 invoke-static/range StubApp->getOrigApplicationContext(Context)Context, v0 .. v0   这种数量巨大，一般都是在调用 getcontext 函数的后面，所以建议保留，请修改 StubApp 里面的函数代码，直接返回原值即可。

```
修改成：
.class public final Lcom/stub/StubApp;
.super Landroid/app/Application;
.source "SourceFile"
# static fields
.field private static loadFromLib:Z
.field private static needX86Bridge:Z
.field public static strEntryApplication:Ljava/lang/String;
.end field
# direct methods
.method public static getOrigApplicationContext(Landroid/content/Context;)Landroid/content/Context;
    .locals 1
    .prologue
    .line 68
    return-object p0
.end method

```

5. 用 dextools 工具 dump 的 dex 存在问题，需要用自己写的某某某_info 脚本跑出来的 oat 修复的 dex 才可以。

把上面修改后的 classes 文件和 dump 的所有其他 dex 文件一起放到 apk 包中，用 apktool 工具反编译后修改 mark 的地方，就是去掉就行。然后修改 Manifest xml 中 application 选项中的 android:name 为 Python 跑出来的 App_Application_Entry

6. 如果存在 onCreate VMP 那就复杂，需要修复 vmp 后才能进行上面的反编译修改再打包。vmp 的修复最好是用 AndroidNativeEmu 模拟器修复。修复过程在某某某加固分析文档中有。参数和数据需要自己到内存中去抓。

附录 1  获取某某某加固信息脚本：
==================

本脚本用来获取某某某加固系统关键数据，包含 dump so，查询 apk 的原始入口，根据特征码搜索 onCreate vmp 处理函数等。

```
/*
Get  jiagu info Script
                    by fxyang
                        2021.01.10
 */
  
var ishook_libart = false;
var new_so_base = 0;
var new_so_size = 0;
var JNI_RegisterNatives_array = [];
var jiagu_sdk = [];
var hook_call_addr = [];
var jiagu_vmp = [];
var addrmemcpy = Module.findExportByName("libc.so", "memcpy");
var addrmalloc = Module.findExportByName("libc.so", "malloc");
var addrmmap = Module.findExportByName("libc.so", "mmap");
var mprotect = Module.findExportByName("libc.so", "mprotect");
var addrstrstr = Module.findExportByName("libc.so", "strstr");
var addrstrlen = Module.findExportByName("libc.so", "strlen");
var addrdlopen = Module.findExportByName("libc.so","dlopen");
var addruncompress = Module.findExportByName('libz.so', 'uncompress');
var addrpthread_create = Module.findExportByName("libc.so", "pthread_create");
var addrstrdup = Module.findExportByName("libc.so","strdup");
var k = 1;
var new_so_dump = 0;
var n = 0;
var str_flag = 0;
var up_flag = 0;
var app_activity = 0;
var App_Activity_str = "";
var App_Activity = "";
var dex_up_pthread_addr = 0;
var dex_uncompress_fun_flag = 0;
var dex_up_flag = 0;
var jiagu_end = 0;
var packname_flag = 1;
var pkg_name ="";
var pathDir = "";
var so_dump = 0;
var oat_dump = 0;
var so_data_k = 0;
var new_so_pht = 0;
var new_so_dyn = 0;
var new_so_pltrel = 0;
var new_so_dynrel = 0;
var new_so_pltrel_size = 0;
var get_so_size = 0;
var dex_base = 0;
var dex_size = 0;
var vmp_fun_addr = 0;
 
function get_pkg_dir() {
    var fileclass = Java.use("java.io.File");
    var mysavePath = "/data/data/" + pkg_name + "/mydump";
    pathDir = fileclass.$new(mysavePath);
    if (!pathDir.exists()) {
        pathDir.mkdirs();
    }
    // console.log("mysavepath:", pathDir);
}
 
function find_vmp_fun() {
    Memory.scan(ptr(new_so_base), parseInt(new_so_size), "D9 F8 00 00 99 F8 04 10 82 1B 00 88 41 EA 01 21", {
        onMatch: function (address, size) {
            vmp_fun_addr = address.add(16);
            var offset  = address.sub(new_so_base);
            console.log("find vmp fun addr :",address,"offset :",offset);
            var code = Instruction.parse(vmp_fun_addr.add(1));
            var next_code = code.next;
            for (var i = 0; i < 8; i++) {
                var next_c = Instruction.parse(next_code);
                console.log(next_c.address, ":", next_c);
                if(JSON.stringify(next_c).indexOf(JSON.stringify("bl")) != -1){
                    // var vmp_fun_addr_str = parseInt(next_c.toString().split("#")[1]);
                    var vmp_fun_addr_str = next_c.toString().split("#")[1];
                    // vmp_fun_addr_str = JSON.stringify(vmp_fun_addr_str);
                    // vmp_fun_addr = vmp_fun_addr_str.replace(/\"/g, "");
                    // vmp_fun_addr = next_c.toString().split("#")[1];
                    // vmp_fun_addr = vmp_fun_addr.add(1);
                    vmp_fun_addr = parseInt(vmp_fun_addr_str) +1;
                    console.log("find call_addr",vmp_fun_addr_str);
                }
                next_code = next_c.next;
            }
        },
        onError: function (reason) {
            console.log('scan error');
        },
        onComplete: function () {
            // console.log("scan over")
        }
    });
}
 
if (mprotect != 0) {
    Interceptor.attach((mprotect), {
        onEnter: function (args) {
            // console.log("mprotect base:",args[0],"mprotect size:",args[1],"mprotect prot:",args[2]);
            // console.log("call addr is ",this.returnAddress);
            if (up_flag == 1) {
                // get new so base and size
                if (args[1] > 0x10000) {
                    new_so_base = args[0];
                    new_so_size = args[1];
                    console.log("new_so_base:", new_so_base, "base_size:", new_so_size);
                    up_flag = 0;
                    new_so_dump = 1;
                    // Memory.scanSync(parseInt(new_so_base), parseInt(new_so_size), "D9 F8 00 00 99 F8 04 10 82 1B 00 88 41 EA 01 21", {
                    //     onMatch: function (address, size) {
                    //         vmp_fun_addr = address;
                    //         console.log("find vmp fun addr :", address);
                    //     },
                    //     onError: function (reason) {
                    //         console.log('scan error');
                    //     },
                    //     onComplete: function () {
                    //         // console.log("scan over")
                    //     }
                    // });
                    // find_vmp_fun();
                }
            }
 
            // if(get_so_size == 1){
            //     var buff_size = args[0].sub(new_so_base);
            //     new_so_size = buff_size.add(args[1]);
            //     console.log("new so size is:",new_so_size);
            //     get_so_size = 0;
            // }
 
            if (dex_up_flag == 1) {
                // console.log("mprotect base:", args[0], "mprotect size:", args[1], "mprotect prot:", args[2]);
                if (args[2] == 7) {
                    var oat_base = args[0];
                    var oat_size = parseInt(args[1]);
                    // Memory.scan(oat_base, oat_size, "64 65 78 0a 30 ?? ?? 00", {
                    //     onMatch: function (address, size) {
                    //         dex_base = address;
                    //         // console.log("find dex dat in",address);
                    //     },
                    //     onError: function (reason) {
                    //         console.log('scan error');
                    //     },
                    //     onComplete: function () {
                    //         // console.log("scan over")
                    //     }
                    // });
                    // dex_size = dex_base.add(0x20).readUInt();
                    console.log('find oat addr is', oat_base, "oat size is ", oat_size);
                    var file_path = pathDir + "/dump_" + oat_base.toString() + ".oat";
                    var file_handle = new File(file_path, "wb");
                    if (file_handle) {
                        Memory.protect(ptr(oat_base), oat_size, 'rwx');
                        var oat_buffer = ptr(oat_base).readByteArray(oat_size);
                        file_handle.write(oat_buffer);
                        file_handle.flush();
                        file_handle.close();
                        console.log("[dump oat in]:" + file_path);
                    }
                    oat_dump += 1;
                }
            }
        }
    })
}
 
if (addruncompress != 0) {
    Interceptor.attach((addruncompress), {
        onEnter: function (args) {
            if (new_so_base == 0) {
                up_flag = 1;
                n = 1;
            }
        }
    })
}
 
if (addrstrlen != 0) {
    Interceptor.attach((addrstrlen), {
        onEnter: function (args) {
            if (str_flag == 11) {
                var str_len = Memory.readCString(args[0]);
                if (str_len.length > 10){
                    console.log("string len is",str_len);
                }
                // console.log("string len is",str_len);
            }
            if (app_activity == 1){
                // Get APP Activity Entry
                if (Memory.readCString(args[0]) == "activityName"){
                    console.log("App_Application_Entry is:",App_Activity_str);
                    app_activity = 0;
                    App_Activity = App_Activity_str;
                }
                 App_Activity_str = Memory.readCString(args[0]);
                 dex_up_flag = 0;
                 str_flag = 0;
            }
            if(jiagu_end == 1){
                console.log("================== The APP  jiagu info ==================");
                console.log("");
                console.log("App_Application_Entry is:",App_Activity);
                console.log("dump save path is ", pathDir);
                if(so_dump == 1){
                    console.log("360_jiagu_so dump over");
                }
                if(oat_dump > 0){
                    console.log("The App oat_dex dump:",oat_dump);
                }
                if (JSON.stringify(jiagu_sdk).indexOf(JSON.stringify("onCreate")) != -1){
                    console.log("The App find ocCreate VMP");
                }
                if (JSON.stringify(jiagu_sdk).indexOf(JSON.stringify("interface11"))!= -1){
                    console.log("The App find interface11 sdk!");
                }
                if(JSON.stringify(jiagu_sdk).indexOf(JSON.stringify("mark"))!= -1){
                    console.log("The App find mark sdk!");
                    // console.log("360_jiagu_native_fun",jiagu_sdk);
                }
                console.log("");
                console.log("=======================jiagu end =======================");
                jiagu_end = 0;
            }
        }
    })
}
 
// var data = new ObjC.Object(args[2]);
// console.log(data)
 
if (addrstrdup != 0) {
    Interceptor.attach((addrstrdup), {
        onEnter: function (args) {
            if (k == 1) {
                var str = Memory.readCString(args[0]);
                console.log("copy string is :", str);
                if (packname_flag == 1){
                    if(str.startsWith("/data/app/com")){
                        var pkg_name_str = str.split("/")[3];
                        pkg_name = pkg_name_str.split("-")[0];
                        console.log("pkg_name is:", pkg_name);
                        packname_flag = 0;
                        get_pkg_dir();
                    }
                }
            }
        }
    })
}
 
if (addrstrstr != 0) {
    Interceptor.attach((addrstrstr), {
        onEnter: function (args) {
            if (str_flag == 1) {
                var str = Memory.readCString(args[0]);
                var str1 = Memory.readCString(args[1]);
                console.log("string in str :", str,str1);
                // var calladdr = this.returnAddress;
                // console.log("call addr:",calladdr)
            }
        }
    })
}
 
if (addrmemcpy != 0) {
    Interceptor.attach((addrmemcpy), {
        onEnter: function (args) {
            if (new_so_dump == 1) {
                var file_path = pathDir + "/dump_knso_" + new_so_base + ".so";
                var file_handle = new File(file_path, "wb");
                if (file_handle) {
                    Memory.protect(ptr(new_so_base), parseInt(new_so_size), 'rwx');
                    var oat_buffer = ptr(new_so_base).readByteArray(parseInt(new_so_size));
                    file_handle.write(oat_buffer);
                    file_handle.flush();
                    file_handle.close();
                    console.log("[dump kn so in]:" + file_path);
                }
                find_vmp_fun();
                new_so_dump = 0
            }
            if (n == 1) {
                console.log("memcpy-->src,dest,len:",args[1],args[0],args[2]);
                if(args[2]== 0x100){
                    so_data_k = 5;
                    new_so_pht = args[0]
                }
                else if(args[2] == 0xb4){
                    n = 0;
 
                    var file_path = pathDir + "/dump_pht_" + new_so_pht + ".dat";
                    // console.log("begin save so dat!",file_path);
                    var file_handle = new File(file_path, "wb");
                    if (file_handle) {
                        Memory.protect(ptr(new_so_pht), 0x100, 'rwx');
                        var oat_buffer = ptr(new_so_pht).readByteArray(0x100);
                        file_handle.write(oat_buffer);
                        file_handle.flush();
                        file_handle.close();
                        console.log("[dump pht dat in]:" + file_path);
                    }
 
                    var file_path = pathDir + "/dump_dyn_" + new_so_dyn + ".dat";
                    var file_handle = new File(file_path, "wb");
                    if (file_handle) {
                        Memory.protect(ptr(new_so_dyn), 0xd8, 'rwx');
                        var oat_buffer = ptr(new_so_dyn).readByteArray(0xd8);
                        file_handle.write(oat_buffer);
                        file_handle.flush();
                        file_handle.close();
                        console.log("[dump dyn dat in]:" + file_path);
                    }
 
                    var file_path = pathDir + "/dump_pltrel_" + new_so_pltrel + ".dat";
                    var file_handle = new File(file_path, "wb");
                    if (file_handle) {
                        Memory.protect(ptr(new_so_pltrel), new_so_pltrel_size, 'rwx');
                        var oat_buffer = ptr(new_so_pltrel).readByteArray(new_so_pltrel_size);
                        file_handle.write(oat_buffer);
                        file_handle.flush();
                        file_handle.close();
                        console.log("[dump pltrel dat in]:" + file_path);
                    }
 
                    var file_path = pathDir + "/dump_dynrel_" + new_so_dynrel + ".dat";
                    var file_handle = new File(file_path, "wb");
                    if (file_handle) {
                        Memory.protect(ptr(new_so_dynrel), 0x4a58, 'rwx');
                        var oat_buffer = ptr(new_so_dynrel).readByteArray(0x4a58);
                        file_handle.write(oat_buffer);
                        file_handle.flush();
                        file_handle.close();
                        console.log("[dump dynrel dat in]:" + file_path);
                    }
 
                    var file_path = pathDir + "/dump_newso_" + new_so_base + ".so";
                    var file_handle = new File(file_path, "wb");
                    if (file_handle) {
                        Memory.protect(ptr(new_so_base), parseInt(new_so_size), 'rwx');
                        var oat_buffer = ptr(new_so_base).readByteArray(parseInt(new_so_size));
                        file_handle.write(oat_buffer);
                        file_handle.flush();
                        file_handle.close();
                        console.log("[dump new so in]:" + file_path);
                        so_dump = 1;
                    }
                }
                else if (so_data_k > 0){
                    if (so_data_k == 1){
                        var size_buff = args[0].sub(new_so_base);
                        new_so_size = size_buff.add(args[2]);
                        console.log("new_so_size is :",new_so_size);
                        // get_so_size = 1;
                    }
                    else if (args[2] > 0xd0){
                        if (args[2] == 0xd8){
                            new_so_dyn = args[0];
                            so_data_k -= 1;
                        }
                        else if(args[2] > 0x800 && args[2] < 0x1000){
                            new_so_pltrel = args[0];
                            new_so_pltrel_size = parseInt(args[2]);
                            so_data_k -= 1;
                        }
                        else if (args[2] > 0x4000 && args[2] < 0x5000){
                            new_so_dynrel = args[0];
                            so_data_k -= 1;
                        }
                        else {
                            so_data_k -= 1;
                        }
                    }
                }
            }
        },
 
        onLeave: function (letval) {
 
        }
    })
}
 
function hook_libart() {
    if (ishook_libart === true) {
        return;
    }
    var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrCheckcakkArgs = null;
    var addrGetStringUTFChars = null;
    var addrNewStringUTF = null;
    var addrFindClass = null;
    var addrGetMethodID = null;
    var addrGetStaticMethodID = null;
    var addrGetFieldID = null;
    var addrGetStaticFieldID = null;
    var addrRegisterNatives = null;
    var addrAllocObject = null;
    var addrCallObjectMethod = null;
    var addrGetObjectClass = null;
    var addrReleaseStringUTFChars = null;
    var addCallStaticObjectMethodV = null;
    var addCallObjectMethodV = null;
    var addCallStaticbooleanMethodV = null;
    var onCreate_hook = null;
    var class_name = null;
    var patch_flag = null;
    var vmp_hook = null;
    var opcode_hook = null;
    var hook_flag = null;
    var GetOpcode_flag = null;
    var vmp_dex_data = [];
    var vmp_opcode_index_v = [];
    var vmp_opcode_v = [];
    var vmp_code_len = [];
 
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name == "_ZN3art3JNI17GetStringUTFCharsEP7_JNIEnvP8_jstringPh") {
            addrGetStringUTFChars = symbol.address;
            console.log("GetStringUTFChars is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc") {
            addrNewStringUTF = symbol.address;
            console.log("NewStringUTF is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI9FindClassEP7_JNIEnvPKc") {
            addrFindClass = symbol.address;
            console.log("FindClass is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetMethodID = symbol.address;
            console.log("GetMethodID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI17GetStaticMethodIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetStaticMethodID = symbol.address;
            console.log("GetStaticMethodID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI10GetFieldIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetFieldID = symbol.address;
            console.log("GetFieldID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI16GetStaticFieldIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetStaticFieldID = symbol.address;
            console.log("GetStaticFieldID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi") {
            addrRegisterNatives = symbol.address;
            console.log("RegisterNatives is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI11AllocObjectEP7_JNIEnvP7_jclass") >= 0) {
            addrAllocObject = symbol.address;
            console.log("AllocObject is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI16CallObjectMethodEP7_JNIEnvP8_jobjectP10_jmethodIDz") >= 0) {
            addrCallObjectMethod = symbol.address;
            console.log("CallObjectMethod is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI14GetObjectClassEP7_JNIEnvP8_jobject") >= 0) {
            addrGetObjectClass = symbol.address;
            console.log("GetObjectClass is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI21ReleaseStringUTFCharsEP7_JNIEnvP8_jstringPKc") >= 0) {
            addrReleaseStringUTFChars = symbol.address;
            console.log("ReleaseStringUTFChars is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI23CallStaticObjectMethodVEP7_JNIEnvP7_jclassP10_jmethodIDSt9__va_list") >= 0) {
            addCallStaticObjectMethodV = symbol.address;
            console.log("CallStaticObjectMethodV is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI17CallObjectMethodVEP7_JNIEnvP8_jobjectP10_jmethodIDSt9__va_list") >= 0) {
            addCallObjectMethodV = symbol.address;
            console.log("CallObjectMethodV is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI24CallStaticBooleanMethodVEP7_JNIEnvP7_jclassP10_jmethodIDSt9__va_list") >= 0) {
            addCallStaticbooleanMethodV = symbol.address;
            console.log("CallStaticbooleanMethodV is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art8CheckJNI13CheckCallArgsERNS_18ScopedObjectAccessERNS_11ScopedCheckEP7_JNIEnvP8_jobjectP7_jclassP10_jmethodIDNS_10InvokeTypeEPKNS_7VarArgsE") >= 0) {
            addrCheckcakkArgs = symbol.address;
            console.log("CheckCallArgs is at ", symbol.address, symbol.name);
        }
    }
 
    if (addrGetObjectClass != null) {
        Interceptor.attach(addrFindClass, {
            onEnter: function (args) {
                console.log('');
                class_name = Memory.readCString(args[1]);
                console.log("Call GetObjectClass; obj_name:", class_name);
                // var call_add = this.returnAddress;
                // console.log(call_add);
                // console.log(hexdump(call_add.sub(0x10)))
 
            },
            onLeave: function (letval) {
            }
        })
    }
 
    if (addrFindClass != null) {
        Interceptor.attach(addrFindClass, {
            onEnter: function (args) {
                //console.log("");
                class_name = Memory.readCString(args[1]);
                console.log("Call findclass; classname:", class_name);
                // console.log("call addr:",this.returnAddress);
            }
        })
    }
 
    if (addrCheckcakkArgs != null) {
        Interceptor.attach(addrCheckcakkArgs, {
            onEnter: function (args) {
                var args_list = Memory.readPointer(args[6]);
                console.log("Call CheckCallArgs; args:", args_list);
            }
        })
    }
 
    if (addrGetStaticMethodID != null) {
        Interceptor.attach(addrGetStaticMethodID, {
            onEnter: function (args) {
                //console.log("call GetStaticMethodID");
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetStaticMethodID,class_name:", class_name, "Method_name:", Method_name, "sig:", Method_sig);
                if(Method_name == "currentPackageName"){
                    // packname_flag  = 1;
                    // str_flag = 1;
                }
                if(Method_name == "getSoPath2"){
                    n = 0;
                }
                if(Method_name == "hashCode"){
                    jiagu_end = 1;
                }
            }
        })
    }
 
    if (addrGetMethodID != null) {
        Interceptor.attach(addrGetMethodID, {
            onEnter: function (args) {
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetMethodID,class_name:", class_name, "Method_name:", Method_name, "sig:", Method_sig);
            }
        })
    }
 
    if (addrGetFieldID != null) {
        Interceptor.attach(addrGetFieldID, {
            onEnter: function (args) {
                //console.log("call GetFieldID");
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetFieldID,className:", class_name, "Method_name:", Method_name, 'sig:', Method_sig);
                if (Method_name == "dexElements"){
                    dex_up_flag = 1;
                    str_flag = 1;
                }
            }
        })
    }
 
    if (addrGetStaticFieldID != null) {
        Interceptor.attach(addrGetStaticFieldID, {
            onEnter: function (args) {
                //console.log("call GetStaticFieldID");
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetStaticFieldID,className:", class_name, "Method_name:", Method_name, "sig:", Method_sig);
                if(Method_name == "strEntryApplication"){
                    app_activity = 1;
                    // find_app_activity();
                }
            }
        })
    }
 
    if (addCallStaticbooleanMethodV != null) {
        Interceptor.attach(addCallStaticbooleanMethodV, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                //console.log("CallStaticbooleanMethodV:", call_add, "offset:", ptr(call_add).sub(new_so_base));
            }
        })
    }
 
    if (addCallObjectMethodV != null) {
        Interceptor.attach(addCallObjectMethodV, {
            onEnter: function (args) {
                var call_address = this.returnAddress;
                //console.log("CallObjectMethodV:", call_address, "offset:", ptr(call_address).sub(new_so_base));
 
            }
        })
    }
 
    if (addrCallObjectMethod != null) {
        Interceptor.attach(addrCallObjectMethod, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                //console.log("CallObjectMethod:", call_add, "offset:", ptr(call_add).sub(new_so_base));
            }
        })
    }
 
 
    if (addCallStaticObjectMethodV != null) {
        Interceptor.attach(addCallStaticObjectMethodV, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                //console.log("CallStaticObjectMethodV:", call_add, "offset:", ptr(call_add).sub(new_so_base));
            }
        })
    }
 
    if (addrRegisterNatives != null) {
        Interceptor.attach(addrRegisterNatives, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                if (call_add < new_so_base.add(new_so_size) && call_add > new_so_base) {
                    call_so_name = "libjiagu_kn.so";
                    console.log("Register_address:", call_add, "offset:", ptr(call_add).sub(new_so_base), "at:", call_so_name);
                    console.log("[RegisterNatives] method_count:", args[3]);
                } else {
                    var call_so_info = Process.getModuleByAddress(call_add);
                    var call_so_base = call_so_info.base;
                    var call_so_name = call_so_info.name;
                    console.log("Register_address:", call_add, "offset:", ptr(call_add).sub(call_so_base), "at:", call_so_name);
                    console.log("[RegisterNatives] method_count:", args[3]);
                }
 
                var methods_ptr = ptr(args[2]);
                var call_addr = this.returnAddress;
                var method_count = parseInt(args[3]);
                for (var i = 0; i < method_count; i++) {
                    var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));
                    var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));
                    var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * 12 + 8));
                    var fuPtr = fnPtr_ptr.sub(1);
                    var name = Memory.readCString(name_ptr);
                    var sig = Memory.readCString(sig_ptr);
                    var module_base = new_so_base;
                    var offset = 0;
                    if (Process.findModuleByAddress(fnPtr_ptr)) {
                        var so_info = Process.getModuleByAddress(fnPtr_ptr);
                        module_base = so_info.base;
                        var so_name = so_info.name;
                        fuPtr = fnPtr_ptr.sub(1);
                        offset = fnPtr_ptr.sub(module_base);
                        console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "call_fun_Ptr:", fnPtr_ptr, "fnPtr:", fuPtr, "so_offset:", offset, "at :", so_name);
                    } else {
                        var so_name = "libjiagu_kn.so";
                        if (parseInt(fnPtr_ptr) > parseInt(module_base)) {
                            offset = fnPtr_ptr.sub(module_base);
                        } else {
                            offset = module_base.sub(fnPtr_ptr);
                        }
                        if (parseInt(offset) > parseInt(new_so_size)) {
                            offset = fnPtr_ptr.sub(5);
                            fnPtr_ptr = Memory.readPointer(offset);
                            offset = fnPtr_ptr.sub(module_base);
                            if (parseInt(offset) > parseInt(new_so_size)) {
                                fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * 12 + 8));
                            } else {
                                console.log("Find shellcode,fix fuPtr!");
                            }
                            fuPtr = fnPtr_ptr.sub(1);
                            offset = fnPtr_ptr.sub(module_base);
                            fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * 12 + 8));
                            console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "shell_Jump_Ptr:", fnPtr_ptr, "fnPtr:", fuPtr, "so_offset:", offset, "at :", so_name);
                        } else {
                            offset = fnPtr_ptr.sub(module_base);
                            console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "fnPtr:", fuPtr, "so_offset:", offset, "at :", so_name);
                        }
                    }
 
                    if(name == "onCreate"){
                        var jiagu_vmp_info = class_name +"/"+ name;
                        jiagu_vmp.push(jiagu_vmp_info);
                        console.log("find dex Methon onCreate code vmp in ",jiagu_vmp_info);
                        console.log("ocCreate code vmp info:\n",JSON.stringify(jiagu_vmp));
                    }
                    fuPtr = fnPtr_ptr.sub(1);
                    if (JSON.stringify(JNI_RegisterNatives_array).indexOf(JSON.stringify(fuPtr)) == -1) {
                        JNI_RegisterNatives_array.push(fuPtr);
                        JNI_RegisterNatives_array.push(name);
                        var func_addr = fnPtr_ptr;
                        var dex_code_len = 0;
                        var opcode_base = null;
                        var get_vmp_data_base = 0;
                        var temp = 0;
                        Interceptor.attach(func_addr, {
                            onEnter: function (args) {
                                var context_r = this.returnAddress;
                                var context_pc = JSON.stringify(this.context["pc"]);
                                console.log("Call_address:", context_r, "fun_address:", context_pc);
                                var pc_add = context_pc.replace(/\"/g, "");
                                for (var j = 0; j < JNI_RegisterNatives_array.length; j++) {
                                    if (JNI_RegisterNatives_array[j] == pc_add) {
                                        this.funname = undefined;
                                        if (JSON.stringify(hook_call_addr).indexOf(JSON.stringify(context_pc)) == -1) {
                                            hook_call_addr.push(context_r);
                                            console.log("Call_fun_begin-->", JNI_RegisterNatives_array[j + 1]);
                                            if(jiagu_sdk.indexOf(JNI_RegisterNatives_array[j + 1])== -1){
                                                jiagu_sdk.push(JNI_RegisterNatives_array[j + 1]);
                                            }
                                            console.log("fun_key0:", this.context.r0);
                                            console.log("fun_key1:", this.context.r1);
                                            console.log("fun_key2:", this.context.r2);
                                            k = 1;
                                            this.funname = JNI_RegisterNatives_array[j + 1];
                                            if(this.funname == "onCreate"){
                                                console.log("====================================================================");
                                                console.log("onCreate VMP begin!");
                                                // str_flag = 11;
                                                GetOpcode_flag = 1;
                                                if (vmp_fun_addr != 0 && vmp_hook == null) {
                                                    name = "vmp_fun";
                                                    JNI_RegisterNatives_array.push(vmp_fun_addr);
                                                    JNI_RegisterNatives_array.push(name);
                                                    Interceptor.attach(ptr(vmp_fun_addr), {
                                                        onEnter: function (args) {
                                                            if (GetOpcode_flag == 1) {
                                                                if (get_vmp_data_base == 0) {
                                                                    var data_base = Memory.readPointer(ptr(this.context.r9));
                                                                    // var dex_info = Process.getModuleByAddress(data_base);
                                                                    // var data_offset = data_base.sub(dex_info.base);
                                                                    var dex_vmp_data_len =  Memory.readPointer(data_base.sub(4));
                                                                    // console.log("dex name is :", dex_info.name);
                                                                    // console.log("call in :",this.depth);
                                                                    console.log("vmp data base is:", data_base);
                                                                    console.log("vmp data len is :",ptr(parseInt(dex_vmp_data_len)*2));
                                                                    console.log(hexdump(data_base,{length:parseInt(dex_vmp_data_len)*2}));
                                                                    console.log("");
                                                                    get_vmp_data_base = 1;
                                                                    var xor_key = this.context.r1;
                                                                    var data_buf = Memory.readPointer(ptr(this.context.r9));
                                                                    var vmp_opcode_index = Memory.readPointer(data_buf);
                                                                    vmp_opcode_index = vmp_opcode_index & 0xFFFF;
                                                                    xor_key = vmp_opcode_index ^ xor_key ;
                                                                    console.log("vmp dex data xor key is :",ptr(xor_key));
                                                                }
                                                                var data_buff = Memory.readPointer(ptr(this.context.r9));
                                                                var vmp_opcode_data = Memory.readPointer(data_buff);
                                                                vmp_opcode_data = vmp_opcode_data & 0xFF;
                                                                var vmp_opcode_index = this.context.r1;
                                                                vmp_opcode_index = vmp_opcode_index & 0xFF;
                                                                console.log("");
                                                                console.log("vmp fun data is :\n", hexdump(data_buff,{length:0x10}));
                                                                console.log("vmp opcode data is :",ptr(vmp_opcode_data));
                                                                vmp_dex_data.push(ptr(vmp_opcode_data));
                                                                console.log("vmp opcode index is :",ptr(vmp_opcode_index));
                                                                vmp_opcode_index_v.push(ptr(vmp_opcode_index));
                                                                // console.log("vmp opcode context is :",JSON.stringify(this.context));
                                                                dex_code_len = parseInt(this.context.r2)- temp;
                                                                temp = parseInt(this.context.r2);
                                                            }
                                                        },
                                                        onLeave:function (retval) {
                                                            if(GetOpcode_flag == 1){
                                                                console.log("vmp opcode is:",ptr(this.context.r0));
                                                                vmp_opcode_v.push(ptr(this.context.r0));
                                                                console.log("dex code len :",dex_code_len);
                                                                vmp_code_len.push(dex_code_len);
                                                                console.log("");
                                                            }
                                                        }
                                                    })
                                                }
                                                vmp_hook = 1;
                                                // str_flag = 11;
                                            }
                                        }
 
                                    }
                                }
                            },
 
                            onLeave: function (retval) {
                                if (this.funname != undefined) {
                                    console.log("Call_fun_end-->", this.funname);
                                    if(this.funname == "onCreate"){
                                        console.log("The onCreate VMP end!");
                                        console.log("");
                                        console.log("The onCreate VMP run info :");
                                        console.log("vmp_opcode_data :\n",vmp_dex_data);
                                        console.log("vmp_opcode_index_v:\n",vmp_opcode_index_v);
                                        console.log("vmp_opcode_v:\n",vmp_opcode_v);
                                        console.log("vmp_code_len:\n",vmp_code_len);
                                        console.log("====================================================================");
                                        str_flag = 0;
                                        GetOpcode_flag = 0;
                                        get_vmp_data_base = 0;
                                        temp = 0;
                                        vmp_opcode_index_v = [];
                                        vmp_opcode_v = [];
                                        vmp_code_len = [];
                                        vmp_dex_data = [];
                                    }
                                    k = 0;
                                }
                            }
                        })
                    }
 
                }
            },
            onLeave: function (retval) {
 
            }
        });
    }
 
    ishook_libart = true;
}
 
hook_libart();
// hook_getByte();

```

附录 2  某某某加固的 frida 跟踪调试脚本
=========================

本脚本主要用来通过跟踪调试获取加固 VMP 中 JAVA 层的函数调用和被 VMP 的 dex 指令长度，通过函数调用比较容易知道后面的 dex 指令类型。再通过长度更容易定位 dex 指令。

当然最好是在 AndroidNativeEm 模拟器中跟踪每个指令过程，或者根据跟踪到的返回值和长度来确定 dex 指令。

```
var ishook_libart = false;
var new_so_base = 0;
var JNI_RegisterNatives_array = [];
var memcpy = Module.findExportByName("libc.so", "memcpy");
var addrmalloc = Module.findExportByName("libc.so", "malloc");
var addrmmap = Module.findExportByName("libc.so", "mmap");
var mprotect = Module.findExportByName("libc.so", "mprotect");
var addrcalloc = Module.findExportByName("libc.so", "calloc");
var addrstrlen = Module.findExportByName("libc.so","strlen");
var addrpthread_create = Module.findExportByName("libc.so","pthread_create");
var VMP_addr = null;
var opcode_addr = null;
var GetOpcode_fun = null;
var k = 0;
var opcode_table_addr = null;
var header_table = 0
  
Interceptor.attach(Module.getExportByName('libz.so', 'uncompress'), {
    onEnter: function (args) {
        if (args[2] != null) {
            Interceptor.attach(addrmalloc, {
                onEnter: function (args) {
                    this.malloc_len = args[0]
                },
  
                onLeave: function (retval) {
                    if (k == 3) {
                        console.log("Call_fun_add,malloc_size;addr:",this.returnAddress, this.malloc_len, this.context.r0)
                        var call_address = this.returnAddress;
                        if (call_address < new_so_base.add(0xC4000) && call_address > new_so_base) {
                            console.log("Call new_so,address:", call_address, "fix_address:", ptr(call_address).sub(new_so_base).add(0xcbd8e000));
                            var call_add = call_address.sub(0x4);
                            var code = Instruction.parse(call_add);
                            console.log(code.address, ":", code);
                            var next_code = code.next;
                            for (var i = 0; i < 3; i++) {
                                var next_c = Instruction.parse(next_code);
                                console.log(next_c.address, ":", next_c);
                                next_code = next_c.next;
                            }
                        }else {
                            console.log("Call malloc,address:", call_address);
                            var call_add = call_address.sub(0x4);
                            var code = Instruction.parse(call_add);
                            console.log(code.address, ":", code);
                            var next_code = code.next;
                            for (var i = 0; i < 3; i++) {
                                var next_c = Instruction.parse(next_code);
                                console.log(next_c.address, ":", next_c);
                                next_code = next_c.next;
                            }
                        }
                    }
                }
            });
  
            Interceptor.attach(addrcalloc, {
                onEnter: function (args) {
                    this.malloc_len = args[0];
                    this.n = args[1];
                },
  
                onLeave: function (retval) {
                    if (k == 3) {
                        console.log("Call_fun_add,callocc_size x n;addr:", this.returnAddress,this.malloc_len, "x", this.n, this.context.r0)
                        var call_address = this.returnAddress;
                        if (call_address < new_so_base.add(0xC4000) && call_address > new_so_base) {
                            console.log("Call new_so,address:", call_address, "fix_address:", ptr(call_address).sub(new_so_base).add(0xcbd8e000));
                            var call_add = call_address.sub(0x4);
                            var code = Instruction.parse(call_add);
                            console.log(code.address, ":", code);
                            var next_code = code.next;
                            for (var i = 0; i < 3; i++) {
                                var next_c = Instruction.parse(next_code);
                                console.log(next_c.address, ":", next_c);
                                next_code = next_c.next;
                            }
                        }else {
                            console.log("Call malloc,address:", call_address);
                            var call_add = call_address.sub(0x4);
                            var code = Instruction.parse(call_add);
                            console.log(code.address, ":", code);
                            var next_code = code.next;
                            for (var i = 0; i < 3; i++) {
                                var next_c = Instruction.parse(next_code);
                                console.log(next_c.address, ":", next_c);
                                next_code = next_c.next;
                            }
                        }
                    }
                }
            });
  
            Interceptor.attach(addrstrlen,{
               onEnter:function (args) {
                   if(k == 1){
                       var str = Memory.readCString(args[0]);
                       console.log("string is :",str)
                   }
               },
               onLeave:function (letval) {
  
               }
            });
            Interceptor.attach(addrmmap, {
                onEnter: function (args) {
                    this.malloc_len = args[1];
                    this.flag = args[4];
                    //console.log(args[4])
                },
  
                onLeave: function (retval) {
                    if (k == 3) {
                        console.log("callfun_add;mmap_size;addr:",this.returnAddress, this.malloc_len, this.context.r0);
                        if (this.flag != 0xFFFFFFFF) {
                            console.log(hexdump(this.context.r0))
                        }
                        // var call_address = this.returnAddress;
                        // var patch_add = call_address.sub(1);
                        // console.log("hook_code_begin:", patch_add);
                        // console.log(hexdump(patch_add));
                        // var patch = [0xFE, 0xE7];
                        // Memory.protect(patch_add, 2, 'rwx');
                        // Memory.writeByteArray(patch_add, patch);
                        // console.log(hexdump(patch_add));
                        // Interceptor.detachAll();
                        if (call_address < new_so_base.add(0xC4000) && call_address > new_so_base) {
                            console.log("Call new_so,address:", call_address, "fix_address:", ptr(call_address).sub(new_so_base).add(0xcbd8e000));
                            var call_add = call_address.sub(0x4);
                            var code = Instruction.parse(call_add);
                            console.log(code.address, ":", code);
                            var next_code = code.next;
                            for (var i = 0; i < 3; i++) {
                                var next_c = Instruction.parse(next_code);
                                console.log(next_c.address, ":", next_c);
                                next_code = next_c.next;
                            }
                        } else {
                            console.log("Call malloc,address:", call_address);
                            var call_add = call_address.sub(0x4);
                            var code = Instruction.parse(call_add);
                            console.log(code.address, ":", code);
                            var next_code = code.next;
                            for (var i = 0; i < 3; i++) {
                                var next_c = Instruction.parse(next_code);
                                console.log(next_c.address, ":", next_c);
                                next_code = next_c.next;
                            }
                        }
                    }
                }
            });
            //var memcpy = Module.findExportByName("libc.so", "memcpy");
            //console.log("memcpy:" + memcpy)
            Interceptor.attach(memcpy, {
                onEnter: function (args) {
                    //console.log("memcpy-->len:" + args[2]);
                    //console.log(JSON.stringify(this.context));
                    if (k == 8){
                        if(args[2] > 0xd0){
                            //console.log("memcpy-->src,dest,len:",args[1],args[0],args[2]);
                            if (args[2] > 0xd0){
                                var data_ptr = args[1];
                                //var len = args[2].add(1) - 1;
                                console.log("memcpy-->len:" + args[2]);
                                var len = 0x40;
                                var data = Memory.readByteArray(data_ptr,len);
                                // if (data == 0x464c457f){
                                //     console.log("memcpy-->src,dest,len:",args[1],args[0],args[2]);
                                // }
                                console.log(hexdump(data));
                            }
                        }
                    }
  
                    if (args[2] == 0x38fc){
                        k = 0
                    }
  
                    if (args[2] == 0x150){
                        console.log("header:",args[0])
                        header_table = args[0]
                        console.log(hexdump(args[1],args[2]))
                    }
  
                    if (args[2] == 0xbccd4) {
                        //console.log(JSON.stringify(this.context));
                        //console.log(hexdump(args[1]));
                        new_so_base = args[0];
                        console.log("new_so_base:", new_so_base);
                        //console.log(hexdump(header_table,0x150));
                        k = 8;
                        //this.hookadd = new_so_base.add(0x37625); //code_op
                        //this.hookadd = new_so_base.add(0x1d458);  //'interface11'
                        //this.hookadd = new_so_base.add(0x1cc4c); //onCreate()
                        // Process.enumerateThreads({
                        //         onMatch: function (thread) {
                        //             // console.log("threadID:", thread.id);
                        //             // console.log("thread_addr:", thread.context.pc);
                        //         },
                        //
                        //         onComplete: function () {
                        //
                        //         }
                        //     });
                    }
  
                },
  
                onLeave: function (retval) {
                    if (this.hookadd != null) {
                        //Interceptor.detachAll();
                        console.log("hook_code_begin:",this.hookadd);
                        console.log(hexdump(this.hookadd));
                        var patch = [0xFE, 0xE7];   //人造一个死循环指令，以便在关键位置调试，防止调试过程中的anti
                        Memory.protect(this.hookadd, 2, 'rwx');
                        Memory.writeByteArray(this.hookadd, patch);
                        var dat = Memory.readByteArray(this.hookadd, 4);
                        console.log(hexdump(dat));
                        this.hookadd = null;
  
                    }
                    if (new_so_base != null){
                        if (k == 3){
                            //console.log("find call new_so!");
                            var call_address = this.returnAddress;
                            //console.log("my_threadID:", this.threadId);
                            if (call_address < new_so_base.add(0xC4000) && call_address > new_so_base) {
                                console.log("Call new_so,address:", call_address,"fix_address:", ptr(call_address).sub(new_so_base).add(0xcbd8e000));
                                var call_add = call_address.sub(0x4);
                                var code = Instruction.parse(call_add);
                                console.log(code.address, ":", code);
                                var next_code = code.next;
                                for (var i = 0; i < 3; i++) {
                                    var next_c = Instruction.parse(next_code);
                                    console.log(next_c.address, ":", next_c);
                                    next_code = next_c.next;
                                }
                                //Interceptor.detachAll();
                                // var patch_add = call_address.sub(1);
                                // console.log("hook_code_begin:",patch_add);
                                // console.log(hexdump(patch_add));
                                // var patch = [0xFE, 0xE7];
                                // Memory.protect(patch_add, 2, 'rwx');
                                // Memory.writeByteArray(patch_add, patch);
                                // console.log(hexdump(patch_add));
  
                                //Thread.sleep(50);
                            }
                        }
                    }
                }
            })
        }
    }
});
  
function hook_libart() {
    if (ishook_libart === true) {
        return;
    }
    var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrCheckcakkArgs = null;
    var addrGetStringUTFChars = null;
    var addrNewStringUTF = null;
    var addrFindClass = null;
    var addrGetMethodID = null;
    var addrGetStaticMethodID = null;
    var addrGetFieldID = null;
    var addrGetStaticFieldID = null;
    var addrRegisterNatives = null;
    var addrAllocObject = null;
    var addrCallObjectMethod = null;
    var addrGetObjectClass = null;
    var addrReleaseStringUTFChars = null;
    var addCallStaticObjectMethodV = null;
    var addCallObjectMethodV = null;
    var addCallStaticbooleanMethodV = null;
    var onCreate_hook = null;
    var class_name = null;
    var patch_flag = null;
    var vmp_hook = null;
    var opcode_hook = null;
    var hook_flag = null;
    var GetOpcode_flag = null;
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name == "_ZN3art3JNI17GetStringUTFCharsEP7_JNIEnvP8_jstringPh") {
            addrGetStringUTFChars = symbol.address;
            console.log("GetStringUTFChars is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc") {
            addrNewStringUTF = symbol.address;
            console.log("NewStringUTF is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI9FindClassEP7_JNIEnvPKc") {
            addrFindClass = symbol.address;
            console.log("FindClass is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetMethodID = symbol.address;
            console.log("GetMethodID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI17GetStaticMethodIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetStaticMethodID = symbol.address;
            console.log("GetStaticMethodID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI10GetFieldIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetFieldID = symbol.address;
            console.log("GetFieldID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI16GetStaticFieldIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetStaticFieldID = symbol.address;
            console.log("GetStaticFieldID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi") {
            addrRegisterNatives = symbol.address;
            console.log("RegisterNatives is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI11AllocObjectEP7_JNIEnvP7_jclass") >= 0) {
            addrAllocObject = symbol.address;
            console.log("AllocObject is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI16CallObjectMethodEP7_JNIEnvP8_jobjectP10_jmethodIDz") >= 0) {
            addrCallObjectMethod = symbol.address;
            console.log("CallObjectMethod is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI14GetObjectClassEP7_JNIEnvP8_jobject") >= 0) {
            addrGetObjectClass = symbol.address;
            console.log("GetObjectClass is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI21ReleaseStringUTFCharsEP7_JNIEnvP8_jstringPKc") >= 0) {
            addrReleaseStringUTFChars = symbol.address;
            console.log("ReleaseStringUTFChars is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI23CallStaticObjectMethodVEP7_JNIEnvP7_jclassP10_jmethodIDSt9__va_list") >= 0) {
            addCallStaticObjectMethodV = symbol.address;
            console.log("CallStaticObjectMethodV is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI17CallObjectMethodVEP7_JNIEnvP8_jobjectP10_jmethodIDSt9__va_list") >= 0) {
            addCallObjectMethodV = symbol.address;
            console.log("CallObjectMethodV is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI24CallStaticBooleanMethodVEP7_JNIEnvP7_jclassP10_jmethodIDSt9__va_list") >= 0) {
            addCallStaticbooleanMethodV = symbol.address;
            console.log("CallStaticbooleanMethodV is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art8CheckJNI13CheckCallArgsERNS_18ScopedObjectAccessERNS_11ScopedCheckEP7_JNIEnvP8_jobjectP7_jclassP10_jmethodIDNS_10InvokeTypeEPKNS_7VarArgsE") >= 0) {
            addrCheckcakkArgs = symbol.address;
            console.log("CheckCallArgs is at ", symbol.address, symbol.name);
        }
    }
  
    if (addrGetObjectClass != null) {
        Interceptor.attach(addrFindClass, {
            onEnter: function (args) {
                console.log('');
                class_name = Memory.readCString(args[1]);
                console.log("Call GetObjectClass; obj_name:", class_name);
                // var call_add = this.returnAddress;
                // console.log(call_add);
                // console.log(hexdump(call_add.sub(0x10)))
  
            },
            onLeave:function (letval) {
            }
        })
    }
  
    if (addrFindClass != null) {
        Interceptor.attach(addrFindClass, {
            onEnter: function (args) {
                //console.log("");
                class_name = Memory.readCString(args[1]);
                console.log("Call findclass; classname:", class_name);
            }
        })
    }
  
    if (addrCheckcakkArgs != null) {
        Interceptor.attach(addrCheckcakkArgs, {
            onEnter: function (args) {
                var args_list = Memory.readPointer(args[6]);
                console.log("Call CheckCallArgs; args:", args_list);
            }
        })
    }
  
    if (addrGetStaticMethodID != null) {
        Interceptor.attach(addrGetStaticMethodID, {
            onEnter: function (args) {
                //console.log("call GetStaticMethodID");
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetStaticMethodID,class_name:", class_name, "Method_name:", Method_name, "sig:", Method_sig);
            }
        })
    }
  
    if (addrGetMethodID != null) {
        Interceptor.attach(addrGetMethodID, {
            onEnter: function (args) {
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetMethodID,class_name:", class_name, "Method_name:", Method_name, "sig:", Method_sig);
                // var call_add = this.returnAddress;
                // console.log(hexdump(call_add.sub(0x10)));
                // var code = Instruction.parse(call_add);
                // var next_code = Memory.readPointer(fun_address.add(4));
                // console.log(code.address, ":", code);
                // for (var i = 0; i < 3; i++) {
                //     var next_c = Instruction.parse(next_code);
                //     console.log(next_c.address, ":", next_c);
                //     next_code = next_c.next;
                // }
  
            }
        })
    }
  
    if (addrGetFieldID != null) {
        Interceptor.attach(addrGetFieldID, {
            onEnter: function (args) {
                //console.log("call GetFieldID");
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetFieldID,className:", class_name, "Method_name:", Method_name, 'sig:', Method_sig);
            }
        })
    }
  
    if (addrGetStaticFieldID != null) {
        Interceptor.attach(addrGetStaticFieldID, {
            onEnter: function (args) {
                //console.log("call GetStaticFieldID");
                var Method_name = Memory.readCString(args[2]);
                var Method_sig = Memory.readCString(args[3]);
                console.log("call GetStaticFieldID,className:", class_name, "Method_name:", Method_name, "sig:", Method_sig);
            }
        })
    }
  
    if (addCallStaticbooleanMethodV != null) {
        Interceptor.attach(addCallStaticbooleanMethodV, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                //console.log("CallStaticbooleanMethodV:", call_add, "offset:", ptr(call_add).sub(new_so_base));
            }
        })
    }
  
    if (addCallObjectMethodV != null) {
        Interceptor.attach(addCallObjectMethodV, {
            onEnter: function (args) {
                var call_address = this.returnAddress;
                //console.log("CallObjectMethodV:", call_address, "offset:", ptr(call_address).sub(new_so_base));
  
            }
        })
    }
  
    if (addrCallObjectMethod != null) {
        Interceptor.attach(addrCallObjectMethod, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                //console.log("CallObjectMethod:", call_add, "offset:", ptr(call_add).sub(new_so_base));
            }
        })
    }
  
  
    if (addCallStaticObjectMethodV != null) {
        Interceptor.attach(addCallStaticObjectMethodV, {
            onEnter: function (args) {
                var call_add = this.returnAddress;
                //console.log("CallStaticObjectMethodV:", call_add, "offset:", ptr(call_add).sub(new_so_base));
            }
        })
    }
  
    if (addrRegisterNatives != null) {
        Interceptor.attach(addrRegisterNatives, {
            onEnter: function (args) {
                //console.log("")
                //var context_p = JSON.stringify(this.context["lr"]);
                var call_add = this.returnAddress;
                console.log("Register_address:", call_add, "offset:", ptr(call_add).sub(new_so_base));
                console.log("[RegisterNatives] method_count:", args[3]);
  
                var methods_ptr = ptr(args[2]);
                var call_addr = this.returnAddress;
                var method_count = parseInt(args[3]);
                for (var i = 0; i < method_count; i++) {
                    var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));
                    var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));
                    var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * 12 + 8));
                    var fuPtr = fnPtr_ptr.sub(1);
                    var name = Memory.readCString(name_ptr);
                    var sig = Memory.readCString(sig_ptr);
                    var module_base = new_so_base;
                    JNI_RegisterNatives_array.push(fuPtr);
                    JNI_RegisterNatives_array.push(name);
                    console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "fnPtr:", fuPtr, "offset:", ptr(fnPtr_ptr).sub(module_base));
                    if (name == "onCreate") {
                        console.log(hexdump(fnPtr_ptr));
                        var code = Instruction.parse(fnPtr_ptr);
                        var next_code = code.next;
                        console.log(code.address, ":", code);
                        for (var i = 0; i < 11; i++) {
                            var next_c = Instruction.parse(next_code);
                            console.log(next_c.address, ":", next_c);
                            next_code = next_c.next;
                        }
                        var onCreate_fun_ptr = next_c.next.add(0xf);
                        var onCreate_fun_addr = Memory.readPointer(onCreate_fun_ptr);
                        console.log("onCreate_function_address:",onCreate_fun_addr,"offset:",ptr(onCreate_fun_addr).sub(module_base));
                        //VMP_addr = onCreate_fun_addr.add(0xE870);
                        VMP_addr = onCreate_fun_addr.add(0xE900);  //new
                        opcode_addr = VMP_addr.add(0xC0D8);
                        GetOpcode_fun = opcode_addr.add(0x1D56);
                        console.log("vmp_addr,opcode_addr,GetOpcode_fun:",VMP_addr,opcode_addr,GetOpcode_fun);
                        console.log("begin hook onCreate_fun");
                        // var patch_add = onCreate_fun_addr.sub(1);
                        // console.log(hexdump(patch_add));
                        // var patch = [0xFE, 0xE7];
                        // Memory.protect(patch_add, 2, 'rwx');
                        // Memory.writeByteArray(patch_add, patch);
                        // console.log(hexdump(patch_add));
                        // Interceptor.detachAll();
                    }
  
                    var func_addr = fnPtr_ptr;
                    var opcode_len = 0;
                    var opcode_base = null;
                    Interceptor.attach(func_addr, {
                        onEnter: function (args) {
                            //console.log("");
                            //k = 1;
                            var context_r = this.returnAddress;
                            var context_pc = JSON.stringify(this.context["pc"]);
                            console.log("Call_address:", context_r, "fun_address:", context_pc);
                            var pc_add = context_pc.replace(/\"/g, "");
                            for (var j = 0; j < JNI_RegisterNatives_array.length; j++) {
                                if (JNI_RegisterNatives_array[j] == pc_add) {
                                    console.log("Call_fun_begin-->", JNI_RegisterNatives_array[j + 1]);
                                    this.funname = JNI_RegisterNatives_array[j + 1];
                                    if (JNI_RegisterNatives_array[j + 1] == "interface11") {
                                        console.log("handle_key:", args[2]);
                                        // var patch_add = this.context.pc.add(0x58A);
                                        // console.log(hexdump(patch_add));
                                        // var patch = [0xFE, 0xE7];
                                        // Memory.protect(patch_add, 2, 'rwx');
                                        // Memory.writeByteArray(patch_add, patch);
                                        // console.log(hexdump(patch_add));
                                        // Interceptor.detachAll();
                                        if (args[2] == 0x30a6){
                                            //patch_flag = 1
                                        }
  
                                        if (args[2] == 0x1a03){
                                            //patch_flag = 1;
                                            var patch_add = this.context.pc.add(4);
                                            console.log(patch_add);
                                            Interceptor.attach(addrmalloc,{
                                                onEnter:function(args){
                                                    this.malloc_len = args[0]
                                                },
  
                                                onLeave:function(retval){
                                                    //console.log("malloc_size;addr:",this.malloc_len,this.context.r0)
                                                }
                                            });
                                            Interceptor.attach(addrcalloc,{
                                                onEnter:function(args){
                                                    this.malloc_len = args[0];
                                                    this.n = args[1];
                                                },
  
                                                onLeave:function(retval){
                                                    console.log("callocc_size x n;addr:",this.malloc_len,"x",this.n,this.context.r0)
                                                }
                                            });
                                            Interceptor.attach(addrmmap, {
                                                onEnter: function (args) {
                                                    this.malloc_len = args[1];
                                                    this.flag = args[4];
                                                    //console.log(args[4])
                                                },
  
                                                onLeave: function (retval) {
                                                    console.log("mmap_size;addr:",this.malloc_len,this.context.r0);
                                                    if (this.flag != 0xFFFFFFFF){
                                                        console.log(hexdump(this.context.r0))
                                                    }
                                                }
                                            });
                                            // console.log(hexdump(patch_add));
                                            // var patch = [0xFE, 0xE7];
                                            // Memory.protect(patch_add, 2, 'rwx');
                                            // Memory.writeByteArray(patch_add, patch);
                                            // //var patch_code = Memory.readByteArray(patch_add, 4);
                                            // //console.log(patch_code)
                                            // console.log(hexdump(patch_add));
                                            // Interceptor.detachAll();
  
                                        }
                                    }
                                    if (JNI_RegisterNatives_array[j + 1] == "onCreate") {
                                        //k = 2;
                                        var opcode_use = [];
                                        var opcode_len_data = [];
                                        var l = 0;
                                        hook_flag = 1;
                                        k = 0;
                                        // var args_addr = this.context.r0;
                                        // console.log("onCreate_fun_args:",args_addr);
                                        //k = 1;
                                        Interceptor.attach(onCreate_fun_addr,{
                                           onEnter:function (args) {
                                               console.log("beging onCreate_fun_addr!");
                                               console.log("onCreate_fun_addr_args:",args[0]);
                                           }
  
                                        });
  
                                        vmp_hook = Interceptor.attach(VMP_addr, {
                                            onEnter: function (args) {
                                                console.log("Call vmp begin!");
                                                var args_l = args[1];
                                                console.log("onCreate args:");
                                                console.log(hexdump(args_l));
                                                var args_2 = args[2];
                                                console.log("onCreate args:");
                                                console.log(hexdump(args_2));
                                                var args_sp = this.context.sp;
                                                args_sp = args_sp.sub(0x68);
                                                console.log(hexdump(args_sp));
                                                k = 0;
                                                //Interceptor.detachAll();
                                                if (hook_flag == 1) {
                                                    if (opcode_use.length > 0) {
                                                        if (l == 1) {
                                                            console.log("opcode_use_list-->len:", opcode_use.length, "opcode_data:", opcode_use,"l=",l);
                                                            opcode_use = [];
                                                            console.log("opcode_len_usedata-->len:",opcode_len_data.length,"opcode_len_usedata-->data:",opcode_len_data,"l=",l);
                                                            opcode_len_data = []
                                                        }
                                                    }
                                                }
                                                opcode_use = [];
                                                l += 1;
                                                opcode_len = 0;
  
                                            },
                                            onLeave: function (letval) {
                                                console.log("Call vmp end!");
                                                if (hook_flag == 1) {
                                                    if (l == 1) {
                                                        console.log("opcode_use_list-->len:", opcode_use.length, "opcode_data:", opcode_use,"l=",l);
                                                        opcode_use = [];
                                                        console.log("opcode_len_usedata-->len:",opcode_len_data.length,"opcode_len_usedata-->data:",opcode_len_data,"l=",l);
                                                        opcode_len_data = []
                                                    }
                                                }
                                                opcode_use = [];
                                                l -= 1;
                                                opcode_len = 0;
                                                k = 0;
                                                //Interceptor.detachAll();
                                            }
                                        });
  
                                        /*                                            opcode_hook = Interceptor.attach(opcode_addr, {
                                                                                        onEnter: function (args) {
                                                                                            console.log("Call opcode_addr begin:");
                                                                                            var base = this.context.r6;
                                                                                            var buf = this.context.r9;
                                                                                            var args_buf = Memory.readByteArray(buf,0x10);
                                                                                            console.log("args_list:",buf);
                                                                                            console.log(hexdump(args_buf));
                                                                                            var opcode_buff = Memory.readPointer(buf);
                                                                                            var opcode_v = Memory.readByteArray(opcode_buff, 4);
                                                                                            var key = Memory.readByteArray(buf.add(4), 1);
                                                                                            var opcode_table = Memory.readPointer(buf.add(8));
                                                                                            //console.log(hexdump(opcode_table));
                                                                                            opcode_table = Memory.readPointer(opcode_table.add(0x60));
                                                                                            console.log("Opcede_table_addr:", opcode_table);
                                                                                            //console.log(hexdump(opcode_table));
                                                                                            console.log("opcode_base:", base);
                                                                                            //console.log(hexdump(base));
                                                                                            console.log("opcode_v:");
                                                                                            console.log(opcode_v);
                                                                                            console.log("key:");
                                                                                            console.log(key)
                                                                                        },
                                                                                        onLeave: function (letval) {
                                                                                            //console.log("Call opcode_addr end!")
                                                                                            opcode_hook.detach()
                                                                                        }
                                                                                    });*/
  
                                        Interceptor.attach(GetOpcode_fun, {
                                            onEnter: function (args) {
                                                console.log("get_opcode begin:")
                                                if (l == 1){
                                                    if (hook_flag == 1) {
                                                        var base = this.context.r6;
                                                        console.log("opcode_base-->",base, opcode_base);
                                                        if (opcode_base !== base) {
                                                            console.log("opcode_v_data:");
                                                            console.log(hexdump(base));
                                                        }
                                                        opcode_base = base;
                                                        var buf = this.context.r9;
                                                        var opcode_ptr = this.context.r7;
                                                        opcode_len = opcode_ptr - opcode_len;
                                                        if (opcode_len > 0) {
                                                            console.log("opcode_len:", opcode_len);
                                                            this.opcode_len_use = opcode_len
                                                        }
                                                        opcode_len = opcode_ptr;
                                                        var args_buf = Memory.readByteArray(buf, 0x10);
                                                        console.log("args_list:", buf);
                                                        console.log(hexdump(args_buf));
                                                        var opcode_buff = Memory.readPointer(buf);
                                                        console.log("opcode_ptr:",opcode_buff);
                                                        var opcode_v = Memory.readByteArray(opcode_buff, 4);
                                                        console.log("opcode_v:");
                                                        console.log(opcode_v);
                                                        var key = Memory.readByteArray(buf.add(4), 1);
                                                        var opcode_table = Memory.readPointer(buf.add(8));
                                                        var opcode_l = Memory.readPointer(new_so_base.add(0xC09D4));
                                                        opcode_l = Memory.readPointer(opcode_l.add(0x24));
                                                        console.log("opcode_lenthg:",opcode_l);
                                                        // var table_args = Memory.readByteArray(opcode_table.add(60), 0x10);
                                                        // console.log("args_2",table_args);
                                                        console.log(hexdump(opcode_table));
                                                        opcode_table = Memory.readPointer(opcode_table.add(0x68));
                                                        console.log("Opcede_table_addr:", opcode_table);
                                                        // if (opcode_table_addr != opcode_table){
                                                        //     opcode_table_addr =opcode_table;
                                                        //     var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
                                                        //     var dir = currentApplication.getApplicationContext().getFilesDir().getPath();
                                                        //     var table_dat = Memory.readByteArray(opcode_table, 0x85C0);
                                                        //     var file_path = dir + "/" + opcode_table + ".dat";
                                                        //     var file_handle = new File(file_path, "wb");
                                                        //     if (file_handle) {
                                                        //         // Memory.protect(ptr(libso.base), libso.size, 'rwx');
                                                        //         // var libso_buffer = ptr(libso.base).readByteArray(libso.size);
                                                        //         file_handle.write(table_dat);
                                                        //         file_handle.flush();
                                                        //         file_handle.close();
                                                        //         console.log("[dump]:", file_path);
                                                        //     }
                                                        // }
  
                                                        //console.log("opcode_base:", base);
                                                        //console.log(hexdump(base));
                                                        // console.log("opcode_v:");
                                                        // console.log(opcode_v);
                                                        console.log("key:");
                                                        console.log(key);
                                                        console.log("opcode_v_buff,key_v", args[0], args[1]);
                                                        //console.log(hexdump(args[0]));
                                                        // patch_add = opcode_addr.sub(1);
                                                        // console.log("hook_begin:", patch_add);
                                                        // var patch = [0xFE, 0xE7];
                                                        // Memory.protect(patch_add, 2, 'rwx');
                                                        // Memory.writeByteArray(patch_add, patch);
                                                        // console.log(hexdump(patch_add));
                                                        // Thread.sleep(500);
                                                        //Interceptor.detachAll();
                                                    }
                                                }
                                            },
  
                                            onLeave: function (letval) {
                                                if (hook_flag == 1){
                                                    if (l == 1) {
                                                        console.log("opcode is :", letval);
                                                        opcode_use.push(letval.toString());
                                                        opcode_len_data.push(this.opcode_len_use)
                                                    }
                                                }
                                            }
                                        });
  
  
                                        //onCreate_hook = fuPtr;
                                        var call_address = this.returnAddress;
                                        console.log("onCreate_call_address:", call_address, "offset:", ptr(call_address).sub(module_base));
                                        var fun_address = this.context.pc;
                                        //console.log(hexdump(fun_address));
                                        //var onCreate_addr = new_so_base.add(0x1C984);
/*                                        Interceptor.attach(onCreate_addr, {
                                            onEnter: function (args) {
                                                console.log("find onCreate fun!")
                                            },
  
                                            onLeave: function (retval) {
  
                                            }
                                        });*/
                                        //var patch_add =  Memory.readPointer(fun_address.add(4));
                                        // var patch_add = call_address.sub(1);
                                        // console.log(hexdump(patch_add));
                                        // console.log("hook_begin:",patch_add);
                                        // var patch = [0xFE, 0xE7];
                                        // Memory.protect(patch_add, 2, 'rwx');
                                        // Memory.writeByteArray(patch_add, patch);
                                        // console.log(hexdump(patch_add));
                                        //Interceptor.detachAll();
                                        // console.log("my_threadID:", this.threadId);
                                        // var mythreadid =  this.threadId
/*                                        Process.enumerateThreads({
                                                onMatch: function (thread) {
                                                    console.log("threadID:",thread.id);
                                                    console.log("thread_addr:", thread.context.pc);
                                                    // console.log("sleep thread!",thread.id);
                                                    // thread.sleep(500)
/!*                                                    if (thread.id != mythreadid){
                                                        console.debug("sleep thread!",thread.id);
                                                        thread.sleep(500)
                                                    }*!/
                                                },
  
                                                onComplete: function () {
  
                                                }
                                            }
                                        );*/
                                        var code = Instruction.parse(fnPtr_ptr);
                                        var next_code = Memory.readPointer(fun_address.add(4));
                                        console.log(code.address, ":", code);
                                        for (var i = 0; i < 3; i++) {
                                            var next_c = Instruction.parse(next_code);
                                            console.log(next_c.address, ":", next_c);
                                            next_code = next_c.next;
                                        }
                                        var next_code = Memory.readPointer(next_c.address.add(0x10));
                                        //console.log(hexdump(next_code.sub(1)));
                                        for (var i = 0; i < 5; i++) {
                                            var next_c = Instruction.parse(next_code);
                                            console.log(next_c.address, ":", next_c);
                                            next_code = next_c.next;
                                        }
                                        // var next_code = Memory.readPointer(next_c.address.add(0x42));
                                        // console.log(hexdump(next_code.sub(1)));
                                        // for (var i = 0; i < 10; i++) {
                                        //     var next_c = Instruction.parse(next_code);
                                        //     console.log(next_c.address, ":", next_c);
                                        //     next_code = next_c.next;
                                        // }
                                        //console.log(hexdump(next_c.next));
                                        //var dat = Memory.readPointer(fuPtr.add(4));
  
                                        //patch_add =  Memory.readPointer(dat);
                                        //Interceptor.detachAll();
                                        // var patch_add = next_code;
                                        // patch_add = patch_add.sub(1);
                                        // //console.log(hexdump(patch_add));
                                        // console.log("hook_begin:",patch_add);
                                        // var patch = [0xFE, 0xE7];
                                        // Memory.protect(patch_add, 2, 'rwx');
                                        // Memory.writeByteArray(patch_add, patch);
                                        // console.log(hexdump(patch_add));
                                    }
                                }
                            }
                        },
  
                        onLeave: function (retval) {
                            k = 0;
                            if (this.funname == "onCreate"){
                                console.log("onCreate_fun_end!");
                                //hook_flag = null;
                            }
                            console.log("Call_fun_end-->", this.funname);
                            //Interceptor.detach(vmp_hook);
                            // if(vmp_hook != null){
                            //     vmp_hook.detach();
                            // }
                            // if (opcode_hook != null){
                            //     opcode_hook.detach();
                            // }
                            if (patch_flag != null) {
                                //Interceptor.detachAll();
                                //var patch_addr = new_so_base.add(0x1d458);  //'interface11'
                                // var patch_addr = new_so_base.add(0x1C984);  // 'onCreate'
                                // console.log("hook_code_begin:",patch_addr);
                                // console.log(hexdump(patch_addr));
                                // var patch = [0xFE, 0xE7];
                                // Memory.protect(patch_addr, 2, 'rwx');
                                // Memory.writeByteArray(patch_addr, patch);
                                // var dat = Memory.readByteArray(patch_addr, 4);
                                // console.log(hexdump(dat));
                                // patch_flag = null;
                                //Interceptor.detach(memcpy_hook)
                            }
                        }
                    })
                }
            },
            onLeave: function (retval) {
                if (onCreate_hook != null){
                    // console.log(hexdump(onCreate_hook));
                    // //var patch_addr = new_so_base.add(0x1d458);  //'interface11'
                    // var patch_addr = onCreate_hook;
                    // Interceptor.detachAll();
                    // console.log("hook_code_begin:", patch_addr);
                    // console.log(hexdump(patch_addr));
                    // var patch = [0xFE, 0xE7];
                    // Memory.protect(patch_addr, 2, 'rwx');
                    // Memory.writeByteArray(patch_addr, patch);
                    // var dat = Memory.readByteArray(patch_addr, 4);
                    // console.log(hexdump(dat));
                    //Interceptor.detachAll();
                }
            }
        });
    }
  
    ishook_libart = true;
}
  
hook_libart();

```

附录 3  某某某加固 dex 嵌入 SDK 修复脚本
===========================

```
import codecs
import leb128
  
def Get_uleb128(vmp_addr,mode):
    if mode == 1:
        offset = leb128.u.encode(vmp_addr)
    if mode == 2:
        offset = leb128.u.decode(vmp_addr)
    return offset
  
def ReadFile(file, address, len,flag=0):
    add = int(address)
    file.seek(add)
    if flag == 0:
        data = int.from_bytes(file.read(len), byteorder="little", signed=False)
    else:
        data =file.read(len)
    return data
  
def main():
    vmp_dex1_filename = r"\dex_new\newdump\classes.dex"
    vm_data = codecs.open(vmp_dex1_filename, "rb")
    vmp_data_buff = vm_data.read()
    dex_data = list(vmp_data_buff)
    sdk_addr_index = r"\dex_new\app_class_interface11.pos"
    sdk_index = codecs.open(sdk_addr_index, "rb")
    ptr = 0x1c
    index_len = 0x2b
    # for i in range(0x21):
    #     sdk_add = ReadFile(sdk_index,ptr,4)
    #     sdk_offset = sdk_add
    #     for k in range(0x8):
    #         dex_data[sdk_offset + k] = 0
    for i in range(0x21):
        sdk_add = ReadFile(sdk_index,ptr,4)
        sdk_offset = sdk_add - 0xa
        code_len = ReadFile(vm_data,sdk_offset,4)
        code_offset = sdk_offset + 4
        if code_len == 0x7:
            dex_data[code_offset] = 0xE
            dex_data[code_offset + 1] = 0
            code_offset += 2
            for k in range(0xC):
                dex_data[code_offset + k] = 0
        else:
            code_addr = code_offset + 0xC
            code =  ReadFile(vm_data,code_addr,((code_len * 2) - 0xC),flag=1)
            for j in range(len(code)):
                dex_data[code_offset + j] = code[j]
            code_offset += len(code)
            for k in range(0xC):
                dex_data[code_offset + k] = 0
  
        ptr += index_len
    data = bytearray(dex_data)
    bak_filename = r"\dex_new\newdump\app_classes_fix_interface11.dex"
    f = open(bak_filename,"wb")
    if f:
        f.write(data)
    f.close()
  
if __name__ == '__main__':
    main()

```

附录 4  某某某加固 vmp 还原 Python 代码：
=============================

利用 IDA 反编译出来的代码结合分析出来的 opcode 指令和真实内存中抓到的数据，用加固自己的算法来还原。

```
import codecs
import os
import sys
  
def Get_opcode_fun(opcode_v):
    opcode_v = opcode_v
    if opcode_v == 0x7000:
        print("出错了")
    if opcode_v < 0x7E:
        if opcode_v >= 0x3E:
            if opcode_v >= 0x5E:
                if opcode_v < 0x6E:
                    if opcode_v >= 0x66:
                        if opcode_v < 0x6A:
                            if opcode_v < 0x68:
                                if opcode_v == 0x67:
                                    return 0x6F
                            if opcode_v == 0x68:
                                return 0
                        if opcode_v >= 0x6C:
                            if opcode_v == 0x6C:  #or 语句 9603 0001 - or-int v3, v0, v1 Calculates v0 OR v1 and puts the result into v3.
                                return 0x96
                        if opcode_v == 0x6B: #move-object vx,vy
                            return 0x07
            else:
                if opcode_v >= 0x62:
                    if opcode_v < 0x64:
                        if opcode_v == 0x63:
                            return 0
                    if opcode_v == 0x64:
                        return 0
                if opcode_v >= 0x60:
                    if opcode_v == 0x60:
                        return 0
                if opcode_v == 0x5F:
                    return 0
                if opcode_v < 0x4D:
                    if opcode_v < 0x45:
                        if opcode_v < 0x41:
                            if opcode_v < 0x3F:
                                return 0
                            if opcode_v == 0x3F:
                                return 0
                        if opcode_v < 0x43:
                            if opcode_v == 0x42:
                                return 0
                if opcode_v == 0x43:
                    return 0
                if opcode_v < 0x49:
                    if opcode_v < 0x47:
                        if opcode_v == 0x46:
                            return 0
                    if opcode_v == 0x47:
                        return 0
                if opcode_v >= 0x4B:
                    if opcode_v == 0x4B:
                        return 0
                    if opcode_v == 0x4A:
                        return 0
                    if opcode_v == 0x52:
                        return 0
                    if opcode_v == 0x50:
                        return 0
                    if opcode_v == 0x53:
                        return 0
                    if opcode_v == 0x54:
                        return 0
                    if opcode_v == 0x57:
                        return 0
                    if opcode_v == 0x58:
                        return 0
                    if opcode_v == 0x5C:
                        return 0
                    if opcode_v == 0x5B:#goto target
                        return 0x28
        else:
            if opcode_v == 0x1F: #iget p0, p0, BBSAppStart->mSplashMaxTimeInSecond:I  5210 0300 - iget v0, v1, Test2.i6:I // field@0003 Reads field@0003 into v0 (entry #3 in the field id table). The instance is referenced by v1.
                return 0x52
            if opcode_v == 0x2E:
                return 0
            if opcode_v == 0x36:
                return 0
            if opcode_v == 0x3A:
                return 0
            if opcode_v == 0x3B:
                return 0
            if opcode_v == 0x3C:
                return 0
            if opcode_v >= 0x38:
                if opcode_v == 0x38:
                    return 0
            if opcode_v == 0x37:
                return 0
            if opcode_v >= 0x32:
                if opcode_v >= 0x34:
                    if opcode_v == 0x34:  #5210 0300 - iget v0, v1, Test2.i6:I // field@0003 JNIEnv->GetFieldId(9
                        return 0x52
            if opcode_v == 0x33:
                return 0
            if opcode_v >= 0x30:
                if opcode_v == 0x30:
                    return 0
            if opcode_v == 0x2F:
                return 0
            else:
                if opcode_v < 0x26:
                    if opcode_v >= 0x22:
                        if opcode_v < 0x24:
                            if opcode_v == 0x23:
                                if opcode_v == 0x24:
                                    return 0
                if opcode_v == 0x20:
                    return 0
                if opcode_v == 0x21:
                    return 0x0c
                if opcode_v < 0x2A:
                    if opcode_v == 0x27:
                        return 0
                    if opcode_v == 0x28:
                        return 0
                    if opcode_v == 0x29: #const/16 vx,lit16
                        return 0x13
                if opcode_v < 0x2C:
                    if opcode_v == 0x2B:
                        return 0
                    if opcode_v == 0x2C:
                        return 0
                    else:
                        if opcode_v >= 0xF:
                            if opcode_v >= 0x17:
                                if opcode_v >= 0x1B:
                                    if opcode_v < 0x1D:
                                        if opcode_v == 0x1C:
                                            return 0
                                if opcode_v == 0x1D:
                                    return 0
                            if opcode_v == 0xF:
                                return 0x1A
                            if opcode_v == 0x19:
                                return 0
                            if opcode_v == 0x18:
                                return 0
                            if opcode_v == 0x17:
                                return 0
                            if opcode_v == 0x16: #div-int/lit16 vx,vy,lit16
                                return 0xd3
                            if opcode_v == 0x13:
                                return 0x54
                            if opcode_v == 0x15: #move 12 03     const/4   p2, 0 Puts the 4 bit constant into vx
                                return 0x12
                            if opcode_v == 0x14:
                                return 0x52
                        if opcode_v == 0x10:
                            return 0
                        if opcode_v == 0x11: # if 跳转
                            return 0x39
                        if opcode_v == 0x12: #const vx, lit32
                            return 0x14
                        if opcode_v >= 7:
                            if opcode_v < 0xB:
                                if opcode_v >= 9:
                                    if opcode_v == 9:
                                        return 0
                                if opcode_v == 8:
                                    return 0
                        if opcode_v < 0xD:
                            if opcode_v == 0xC:
                                return 0
                        if opcode_v == 0xD: #return-void
                            return 0x0F
                        else:
                            if opcode_v >= 3:
                                return 0
                            if opcode_v < 5:
                                if opcode_v == 4:
                                    return 0
                        if opcode_v == 5:
                            return 0
                        else:
                            if opcode_v == 1: #invoke-virtual
                                return 0x6e
  
    if opcode_v >= 0xBE:
        if opcode_v >= 0xDD:
            if opcode_v < 0xEE:
                if opcode_v >= 0xE6:
                    if opcode_v < 0xEA:
                        if opcode_v >= 0xE8:
                            if opcode_v == 0xE8:
                                return 0
                    if opcode_v == 0xE7:
                        return 0
                if opcode_v >= 0xEC:
                    if opcode_v == 0xEC:
                        return 0
                if opcode_v == 0xEB:
                    return 0x46
            if opcode_v < 0xE1:
                if opcode_v >= 0xDF:
                    if opcode_v == 0xDF:
                        return 0
                if opcode_v == 0xDE:
                    return 0
                if opcode_v == 0xDD:
                    return 0x71
            if opcode_v < 0xE3:
                if opcode_v == 0xE2:
                    return 0
        if opcode_v == 0xE3:
            return 0
        if opcode_v < 0xF6:
            if opcode_v < 0xF2:
                if opcode_v < 0xF0:
                    if opcode_v == 0xEF:
                        return 0
            if opcode_v == 0xF0:
                return 0
        if opcode_v < 0xF4:
            if opcode_v == 0xF3:
                return 0
        if opcode_v == 0xF4:
            return 0
        if opcode_v < 0xFB:
            if opcode_v >= 0xF9:
                if opcode_v == 0xF9:
                    return 0
                if opcode_v == 0xF8:
                    return 0
        if opcode_v == 0xFF:  #move 0C 01  move-result-object  v1  0c XX  str.w   r1, [r0, r2, lsl #2] R2=0x2
            return 0x0C
        if opcode_v < 0xFE:
            if opcode_v == 0xFD: #Jumps to target if vx==02. vx is an integer value.
                return 0x38
            if opcode_v == 0xFE:
                return 0
            else:
                if opcode_v >= 0xCD:
                    if opcode_v >= 0xD5:
                        if opcode_v >= 0xD9:
                            if opcode_v == 0xDC: #ushr-int/lit8 vx, vy, lit8
                                return 0xe2
                            if opcode_v == 0xDB:
                                return 0
                if opcode_v == 0xDA:
                    return 0
            if opcode_v == 0xD7: #iget-object vx,vy,field_id
                return 0x54
            if opcode_v == 0xD6:
                return 0
            else:
                if opcode_v >= 0xD1:
                    if opcode_v >= 0xD3:
                        if opcode_v == 0xD3:
                            return 0
            if opcode_v == 0xD1:
                return 0x22
            if opcode_v == 0xD2: #move-result-object vx
                return 0x0c
            if opcode_v == 0xD4: #const vx, lit32
                return 0x14
            if opcode_v < 0xCF:
                if opcode_v == 0xCE:
                    return 0
            if opcode_v == 0xCF:
                return 0
            else:
                if opcode_v == 0xC5:
                    return 0x0e
                if opcode_v == 0xC1:
                    return 0
                if opcode_v == 0xC3:
                    return 0
                if opcode_v == 0xC3:
                    return 0
                if opcode_v == 0xC2:
                    return 0
                if opcode_v >= 0xBF:
                    return 0
                if opcode_v == 0xC9:
                    return 0
                if opcode_v == 0xC7:
                    return 0
                if opcode_v == 0xC7:
                    return 0
                if opcode_v == 0xC6:
                    return 0
                if opcode_v == 0xCB:
                    return 0
                if opcode_v == 0xCA:
                    return 0
                if opcode_v == 0xCB:
                    return 0
    if opcode_v >= 0x9E:
        if opcode_v < 0xAE:
            if opcode_v == 0xA6: #check-cast vx, type_id
                return 0x1c
            if opcode_v < 0xAA:
                return 0
            if opcode_v < 0xA8:
                return 0
            if opcode_v == 0xA7:
                return 0
            if opcode_v == 0xA8:
                return 0
            if opcode_v == 0xAD:  #调用虚方法 invoke-virtual Invokes a virtual method with parameters.
                return 0x6E
            if opcode_v == 0xAC:
                return 0
            if opcode_v == 0xA4: #sget-wide vx, field_id
                return 0x61
            if opcode_v == 0xA3:
                return 0
        if opcode_v < 0xA0:
            if opcode_v == 0x9F:
                return 0
        if opcode_v == 0xA0:
            return 0
        if opcode_v < 0xB6:
            if opcode_v < 0xB2:
                if opcode_v >= 0xB0:
                    if opcode_v == 0xB0:
                        return 0
            if opcode_v == 0xAF:
                return 0
        if opcode_v == 0xB3: #iget vx, vy, field_id
             return 0x52
        if opcode_v == 0xB4: #move-result vx
            return 0x0a
        if opcode_v >= 0xBA:
            if opcode_v >= 0xBC:
                if opcode_v == 0xBC:
                    return 0
            if opcode_v == 0xBB:
                return 0
        else:
            if opcode_v >= 0xB8:
                if opcode_v == 0xB8:
                    return 0
        if opcode_v == 0xB7:
            return 0
    if opcode_v == 0x8D:
        return 0
    if opcode_v == 0x85:
        return 0
    if opcode_v == 0x81:
        return 0
    if opcode_v == 0x7F:
        return 0
    if opcode_v >= 0x83:
        if opcode_v == 0x83:
            return 0
    if opcode_v == 0x82:
        return 0
    if opcode_v < 0x89:
        if opcode_v < 0x87:
            if opcode_v == 0x86:
                return 0
    if opcode_v == 0x87:
        return 0
    if opcode_v == 0x8E:
        return 0
    if opcode_v == 0x8B:  # if-ez
        return 0x38
    if opcode_v == 0x8A:
        return 0
    if opcode_v == 0x92: #if-lt vx,vy,target
        return 0x34
    if opcode_v == 0x8F:
        return 0
    if opcode_v == 0x90:
        return 0
    if opcode_v == 0x93:
        return 0
    if opcode_v == 0x94:
        return 0
    if opcode_v == 0x9A:
        return 0
    if opcode_v == 0x9b:
        return 0
    if opcode_v == 0x9C:
        return 0
    if opcode_v == 0x9d:
        return 0
    if opcode_v == 0x97:
        return 0
    if opcode_v == 0x98:
        return 0
    if opcode_v == 0x76:
        return 0
    if opcode_v == 0x7A:  #invoke-super {parameter},methodtocall
        return 0x6f
    if opcode_v == 0x7C:
        return 0
    if opcode_v == 0x7B:
        return 0
    if opcode_v == 0x7C:  #1C 00 74 13   const-class  0, HttpEvent  const-class vx,type_id
        return 0x1C
    if opcode_v >= 0x78:
        if opcode_v == 0x78:
            return 0
        if opcode_v == 0x77:
            return 0
        if opcode_v >= 0x72:
            if opcode_v >= 0x74:
                if opcode_v == 0x74:
                    return 0
            else:
                if opcode_v == 0x73:
                    return 0
    if opcode_v == 0x6F:
        return 0
    if opcode_v == 0x70:#invoke-super {parameter},methodtocall
        return 0x6f
    return 0
  
def Get_Opcode_v(vm_index,table):
    """
    args_list: 0xd1c0f790
              0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
    00000000  0c 1e 55 cf 91 00 00 00 00 52 de e4 60 b3 00 00  ..U......R..`...
               0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
    e4de5200  08 78 fa ce 20 ef e4 e4 b9 0d 00 00 00 00 00 00  .x.. ...........
    e4de5210  d0 01 4b e2 00 00 00 00 00 00 00 00 00 00 00 00  ..K.............
    e4de5220  00 00 00 00 48 27 0e e6 48 27 0e e6 01 00 00 00  ....H'..H'......
    e4de5230  58 67 f9 cb 58 26 0e e6 04 00 00 00 b8 dd a1 e7  Xg..X&..........
    e4de5240  88 26 0e e6 04 00 00 00 4c 52 de e4 00 00 00 00  .&......LR......
    e4de5250  00 00 00 00 30 77 c7 d1 20 78 c7 d1 08 00 00 00  ....0w.. x......
    e4de5260  40 8c 4f e2 64 00 00 00 80 54 de e4 d8 54 de e4  @.O.d....T...T..
    var opcode_table = Memory.readPointer(buf.add(8));
    // var table_args = Memory.readByteArray(opcode_table.add(60), 0x10);
    // console.log(table_args);
     console.log(hexdump(opcode_table));
     opcode_table = Memory.readPointer(opcode_table.add(0x60));
     console.log("Opcede_table_addr:", opcode_table);
    """
    key1 = 0x3b50 % 0x64
    key1 =  key1 <<8
    key = vm_index % 0x100
    index = key1 ^ key
    opcode_v = int.from_bytes(ReadFile(table, index, 1), byteorder="little", signed=False)
    return opcode_v
  
def Get_args(vm_index):
    dat = vm_index >>8
    return dat
  
def VMP_某某某_fun(vm_item,code,m_l):
    dat = vm_item % 0x100
    dat = format(dat,"x")
    if len(dat) < 2:
        dat = "0" + dat
    code = code + dat
    dat2 = vm_item >> 8
    dat2 = format(dat2,"x")
    if len(dat2) < 2:
        dat2 = "0" + dat2
    code = code + dat2
    return code
  
def ReadFile(file, address, len):
    add = int(address)
    file.seek(add)
    data = file.read(len)
    return data
  
def main():
    k = 0
    i = 0
opcode_len_dat = [3,3,3,3,3,1,2,3,1,3,1]   #test
xor_key = 0x9595
    vm_data = codecs.open(r"MainActivity_opcode_v", "rb") #test
    opcode_v_table = codecs.open(r"comexampletestMainActivity", "rb") #test
    code = ""
    opcode_v_data = []
    opcode_v_list = []
    opcode_v_not = []
    code_up = ""
    for p in range(len(opcode_len_dat)):
        op_len = opcode_len_dat[p]
        m_l = op_len
        print("")
        print("opcode_le:",op_len)
        while op_len:
            data = int.from_bytes(ReadFile(vm_data, k, 2), byteorder="little", signed=False)
            # print(hex(k))
            print("data:",hex(data))
            vm_index = data ^ xor_key
            op_len -= 1
            if i == 0:
                opcode_v = Get_Opcode_v(vm_index, opcode_v_table)
                opcode = Get_opcode_fun(opcode_v)
                print("opcode:",hex(opcode_v))
                opcode_v_data.append(hex(opcode_v))
                if hex(opcode_v) not in opcode_v_list:
                    opcode_v_list.append(hex(opcode_v))
                    if opcode == 0:
                        opcode_v_not.append(hex(opcode_v))
                opcode = format(opcode, 'x')
                if len(opcode) < 2:
                    opcode = "0" + opcode
                code = code + opcode
                opcode_args = Get_args(vm_index)
                opcode_args = format(opcode_args, "x")
                if len(opcode_args) <2:
                    opcode_args = "0" + opcode_args
                code = code + opcode_args
                i = 1
                k += 2
                if op_len == 0:
                    print("code:", code)
                    code_up = code_up + code
                    i = 0
                    code = ""
                continue
            code = VMP_某某某_fun(vm_index,code,m_l)
            if op_len == 0:
                i = 2
            if i == 2:
                print("code:",code)
                code_up = code_up + code
                code = ""
                i = 0
            k += 2
    print("opcode_操作码：",opcode_v_data)
    print("使用的opcode种类：",opcode_v_list,"合计个数：",len(opcode_v_list))
    print("未查询到的opcode：",opcode_v_not,"合计个数：",len(opcode_v_not))
    print("恢复的代码：",code_up)
  
if __name__ == '__main__':
main()

```

[[培训] 二进制漏洞攻防（第 3 期）；满 10 人开班；模糊测试与工具使用二次开发；网络协议漏洞挖掘；Linux 内核漏洞挖掘与利用；AOSP 漏洞挖掘与利用；代码审计。](https://www.kanxue.com/book-section_list-174.htm)

[#混淆加固](forum-161-1-121.htm)