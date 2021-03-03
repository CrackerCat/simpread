> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266247.htm)

### 梆 x 加固分析

#### 前言

这个样本是年初自己拿到官网去加固，回来没有直接 dump 就直接分析了。可能是我的 demo 代码太少了就一句 log. 结果，分析了几天，发现这个样本没抽取。。。。但是还是简单记录一下，感兴趣的小伙伴可以看看。文章中若有出错的地方，请大佬们指正，然后由于是没抽取的版本，可能有些校验之类的没有调试到，还请多多包涵.

#### 0x0 对样本进行简单分析

![](https://bbs.pediy.com/upload/attach/202103/762912_TSNAUGKKFGAD53V.png)  
Java 层 先定位 Application

 

![](https://bbs.pediy.com/upload/attach/202103/762912_W2AFHE8MZD5T9Q4.png)

 

so 层看一下

 

![](https://bbs.pediy.com/upload/attach/202103/762912_CJ5ZFXXKXCKKZXF.png)

 

![](https://bbs.pediy.com/upload/attach/202103/762912_RBX25XX4GZ87G73.png)

 

调试了一段时间，发现和 upx 特征很相似，所以也按 upx 的通用方法进行脱壳，下面是修复 so 的方法。

#### 0x1 so 脱壳与修复

##### 获取解压缩后的 segment 数据

断点: 0xA38F0（mprotect）

 

![](https://bbs.pediy.com/upload/attach/202103/762912_WZGV8NKDEEMBYMG.png)  
进行 dump 被压缩的 segment。第一个参数为被压缩的 segment 原大小，第二个就是起始地址了，根据原大小和起始地址，可以 dump 出解压后的内容，然后本地进行修复。

##### 本地修复

修复流程基本是固定的，主要就是把解压缩后的内容，填充回去，这边篇幅原因，修复之前已经写过了，就直接已资源的形式放到文章末尾了。。关于最后函数的真实地址，可以直接断点 linker 执行 init 的时机（脚本里面有），F7 后面就是原来 so init 函数的真实地址，可以选择性的修复回去。  
![](https://bbs.pediy.com/upload/attach/202103/762912_4ZH5KBDY5KDVEUR.png)

##### 修复后

![](https://bbs.pediy.com/upload/attach/202103/762912_K44TR56CZ4U2C79.png)

 

![](https://bbs.pediy.com/upload/attach/202103/762912_JUVQ66XNPGSQX4Z.png)

#### 0x2 搭建一个调试环境

1.  我这里大概开 2 个 IDA，一个用来打开修复后的 so 记录日志，另一个用来动态调试.
2.  由于修复后的 so 没办法直接运行，所以只能直接动态调试原包，每次定位 init_array
3.  快速定位上一次调试的地址
4.  对 ITE 指令做一些特殊的处理

idapython 也会在文章末尾给出。注: 目前脚本中的偏移地址都是根据 Android6.0.1 来的

#### 0x3 init_array

init_array 在修复后 so 的偏移 sub_E4E8

 

经过一段时间分析，没发现特别重要的，主要是做了一些初始化，比如 libc 中相关函数的初始化.....

 

![](https://bbs.pediy.com/upload/attach/202103/762912_GD2UA3QRHQV2NG2.png)

#### 0x4 JNI_ONLOAD

##### 简介

UPX 修复好之后，JNI_ONLOAD 已经暴露出来了，看一下代码执行流程图

 

![](https://bbs.pediy.com/upload/attach/202103/762912_Y64XM7R4N33WSNU.png)

 

比较明显，加了不是很复杂的 ollvm，IDA 7.5 F5 后可以看到大致代码的逻辑，就是很多控制流平坦化，调试的时候比较耗时间。

 

字符串基本都是加密的，不过到时候用我的 idb，已经都标注好了

 

![](https://bbs.pediy.com/upload/attach/202103/762912_DZ3MSK25SPFEWQ3.png)

##### 第一阶段

这个阶段主要是做初始化，例如初始化一些后面常用 libc 函数的地址，初始化一些后面会用到的私有目录、sdcard 目录等等......，和一些兼容性的处理 (根据不同的机型，不同的 Android 版本号).  
![](https://bbs.pediy.com/upload/attach/202103/762912_B62WJ93UG4PX3E9.png)  
![](https://bbs.pediy.com/upload/attach/202103/762912_8MKRE6W3GMRYBBN.png)

##### 第二阶段

第二阶段是比较重要的一个阶段，主要做了一些动态注册 Java 层函数，与 dex 的初始化 (包括拷贝 assets，以及 load 到内存)，hook 一些系统函数的操作，下面详细介绍一下这个流程. 主要偏移从 0x1EA1C 开始。

##### 注册 Java 层的 native 函数

注册基本都在 **sub_126BC** 函数中:  
![](https://bbs.pediy.com/upload/attach/202103/762912_H88PYCBPRFD44VU.png)

 

注册函数详情:  
![](https://bbs.pediy.com/upload/attach/202103/762912_4WRPA9HFMM5CWK3.png)  
有个单独注册的函数

 

![](https://bbs.pediy.com/upload/attach/202103/762912_DJ64FHNAVE4KV37.png)

##### 操作 assets 目录下的资源

定位偏移 0x1F1F0  
![](https://bbs.pediy.com/upload/attach/202103/762912_P6K4TVCY58HRWYP.png)  
![](https://bbs.pediy.com/upload/attach/202103/762912_HTUV8RWD9AZ8G94.png)

 

上面进行一系列校验后，下面就开始走 copy 流程了

 

![](https://bbs.pediy.com/upload/attach/202103/762912_TXQT7SHR5TXWVGR.png)

 

文件本地落地  
![](https://bbs.pediy.com/upload/attach/202103/762912_HBVYZJV7BS98UBA.png)  
执行好后，可以看到私有目录多了文件  
![](https://bbs.pediy.com/upload/attach/202103/762912_N94GETB3JSWU24Y.png)

##### Hook 一些函数

Hook API 实现在 pECFD6C6E0567ABA6034040D903F38908 这个函数中

 

主要对 libc、libart 进行了 hook

 

例如:**sub_33B08** 函数中 libart 下的 hook:3art15DexFileVerifier6Verify、3art7DexFile10OpenMemory 等  
![](https://bbs.pediy.com/upload/attach/202103/762912_N9YKY226XWKJC8N.png)

 

**sub_392E8** 函数中主要是 libc 相关的 hook: write、read、mmap、munmap 等

 

![](https://bbs.pediy.com/upload/attach/202103/762912_P7K3B49XM4Z7PZ2.png)

##### 加载 dex

定位函数 **sub_15928**  
先构造 path  
![](https://bbs.pediy.com/upload/attach/202103/762912_NUBWN7FMXZSQ4XV.png)

 

JNI 调用 java 层加载 dex

 

![](https://bbs.pediy.com/upload/attach/202103/762912_AAAS6NQZAZ2GXSW.png)

 

java 层的实现  
![](https://bbs.pediy.com/upload/attach/202103/762912_9WMBWAM5MQRGABN.png)

 

加载后可以看到，dex 已经到内存中了  
![](https://bbs.pediy.com/upload/attach/202103/762912_MBCB9SCHCHNRNXC.png)  
其实后面还有一些流程，但是一看没有抽取，发现自己这个版本不太对劲，瞬间没有动力分析下去了。

#### 0x5 脱壳

由于没抽取，所以脱壳还是比较简单，这里说一句，本身用来加载的 dex 的落地文件是不规则的，个人猜测是通过 fread 和 fwrite 进行解密的，但是没去验证.

 

执行完 sub_15928 之后，对 maps 下 classes.jar 的地址进行 dump 就可以了。

 

![](https://bbs.pediy.com/upload/attach/202103/762912_ZHNBBF3ATFQTFXG.png)

#### 0x6 一些检测

sub_17218: 签名校验

 

![](https://bbs.pediy.com/upload/attach/202103/762912_5QPRQSVH7P9PE7M.png)

 

sub_18DF4:Hook Android 日志

 

![](https://bbs.pediy.com/upload/attach/202103/762912_D3E4AK6FPXR7UFT.png)

 

sub_49794: 检测 FART

 

![](https://bbs.pediy.com/upload/attach/202103/762912_PQAQDVWJ9UG2N2B.png)

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

上传的附件：

*   [upx 修复. pdf](javascript:void(0)) （988.83kb，10 次下载）
*   [IDA 脚本. zip](javascript:void(0)) （41.86kb，6 次下载）
*   [test.apk](javascript:void(0)) （3.29MB，6 次下载）
*   [libSecShell_fix.so.zip](javascript:void(0)) （3.54MB，6 次下载）