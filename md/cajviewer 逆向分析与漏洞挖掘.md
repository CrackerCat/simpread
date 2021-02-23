> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/7STPL-2nCUKC3LHozN6-zg)

  

  

以下文章由作者【hackedbylh】的连载有赏投稿，详情可点击 [OSRC 重金征集文稿！！！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247484531&idx=1&sn=6925d63e60984c8dccd4b0162dd32970&chksm=fa7b053fcd0c8c2938d1c5e0493a20ec55c2090ae43419c7aef933bcc077692b1997f4710afa&scene=21#wechat_redirect)了解~~  

温馨提示：建议投稿的朋友尽量用 markdown 格式，特别是包含大量代码的文章

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSSHYtqYw5uTTco31RQX79OwsINNMDvZnbm5t9iaOgWTq6lWyxXaTQJxg/640?wx_fmt=jpeg)

**前言**
======

`CAJViewer`是一个论文查看工具，主要用于查看`caj`文件格式的论文。本文介绍对该软件进行逆向分析和漏洞挖掘的过程。
===============================================================

**Fuzz 测试**  

**Windows 版本**

首先分析的是`CAJViewer`的 Windows 版本，由于我们的目的是挖掘软件的漏洞，通过介绍我们知道 CAJViewer 本质上是一个文件解析程序，因此该软件的高危模块应该是软件中**解析文件数据的部分**，因此首先应该大概定义软件数据处理部分所在位置，Windows 平台下可以使用 process monitor 来进行初步的分析。

首先打开 process monitor 并开始捕获事件，然后使用`CAJViewer`打开一个`caj`文件，等文件解析完成后停止捕获事件。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSEia0AfxPHMSh2f45UaEC7OumtTEPibVcukRsJZz0xPuQFEyvnf7JEnoQ/640?wx_fmt=png)

然后我们可以过滤一下需要查看的事件，比如上图设置了**只查看文件操作并且只查看对** **`input.caj` 文件的操作**，该文件就是之前让`CAJViewer`打开的文件。

然后我们可以找一下读文件的操作（`ReadFile`），因为大部分文件解析逻辑应该读一部分文件内容解析一部分，因此通过查看读文件时的调用栈就可以大概定位解析数据的模块，然后双击就可以查看调用相应函数的**调用栈**。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSeib0otj2YgrGOp3ec5jkfmBDFQS8hUt1icyLmzetj0xTQHhP7znrYicEw/640?wx_fmt=png)

通过查看多个数据读取的调用栈，可以发现 ReaderEx.dll 在调用栈中出现多次，因此大概可以猜测 ReaderEx.dll 应该主要负责处理文件数据。

**Linux 版本**

逆向了一会 ReaderEx.dll 后，发现 CAJViewer 今年还发布了 Linux 版本，于是下载下来分析了一下。下载下来后是一个可执行文件`CAJViewer-x86_64-libc-2.24.AppImage`，执行起来查看进程的 maps 发现其实软件会在 tmp 目录把打包好的二进制解压，然后去执行 tmp 目录下的二进制。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS3PiaKs3qpjEV47d2sl1L21eYEyTaUA4OYaA3vjjnMhDFbtIDDlmTib9g/640?wx_fmt=png)

这里可以直接把`/tmp/.mount_CAJVierjayBH/`拷贝到一个目录，然后就可以直接执行 `cajviewer`了。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSYcgCsN59bpLWUz2vuZia6tWGRA97keK0HgxpGM3tQnBZE5qPsNTX4mg/640?wx_fmt=png)

查看解压处理的二进制发现一个 libreaderex_x64.so，看名字应该是 ReaderEx.dll 的 Linux 版本，然后使用 IDA 打开，发现比 Windows 版本的要好分析一点，信息也比 ReaderEx.dll 的多。于是接下来决定对 Linux 版本的二进制进行分析。

首先看看主程序 cajviewer，查看 main 函数可以发现软件是用 qt 写的

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS64Mn9R9PmukDuh81s83tG5aicoES5BLtCpeRtI8pwN1dxkp9PDqiaUPw/640?wx_fmt=png)

之后翻了一下函数列表，发现了`MainWindow::OpenFile`，看名称应该是打开一个文件。

```
__int64 __fastcall MainWindow::OpenFile(MainWindow *this, const QString *a2)
{


  v2 = this;
  QString::toUtf8_helper(&v16, a2);
  memset(v19, 0, sizeof(v19));
  *v19 = 0x2D8;
  *&v19[4] = 256;
  *&v19[8] = CAJFILE_CreateErrorObject(&v20);
  v3 = *&v19[8];
  if ( *v16 > 1 || (v5 = *(v16 + 2), v4 = v16, v5 != 24) )
  {
    QByteArray::reallocData(&v16, v16[1] + 1, *(v16 + 11) >> 31);
    v4 = v16;
    v5 = *(v16 + 2);
  }
  v6 = CAJFILE_OpenEx1(v4 + v5, v19);           // 打开文件


```

这里对输入的`QString`进行一些处理后，调用了`CAJFILE_OpenEx1`函数，该函数位于`libreaderex_x64.so`。

**Fuzz CAJFILE_OpenEx1 函数**

函数代码如下

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSib9JORw8RW0iavBuB6UnAOibBehicRmTFnvMNbQicAaOm5u5CCp2iaz50baw/640?wx_fmt=png)

函数的第一个参数是要解析的文件路径，第二个参数是一块内存，这个参数的结构可以查看`MainWindow::OpenFile`调用点。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSnHyA2b8iaAa17W6L2tfSgpg1TVnUHoibwwBYicIibLoGUmPzn1ND6bVCXQ/640?wx_fmt=png)

可以看到 in_buf 的结构如下

```
+0: 4个字节 in_buf的长度
+4: 4个字节 一个整形值
+8: 一个指针， 存放构造好的 ErrorObject


```

使用调试器在这个函数下个断点，然后打开一个文件就可以看到入参如下

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSRgn66N3NWib4OWRJD1muqunicWQxFkabUFkc1UyoYNlAG9QianFIMQ9KQ/640?wx_fmt=png)

之后有简单的翻了一些该函数的实现，以及使用该函数的位置可以大概确定`CAJFILE_OpenEx1`用于打开一个文件，并会对文件的内容进行解析，因此下面打算使用 AFL Qemu 模式 Fuzz 一下这个函数。

Fuzz 之前需要写一点代码把 so 加载到内存，然后构造参数对目标函数进行测试。

首先需要把 SO 加载到内存中并获取目标函数的地址

```
void my_init(void) __attribute__((constructor)); //告诉gcc把这个函数扔到init section
void my_init(void)
{
    void *handle;
    handle = dlopen("/home/hac425/cajviewer/cajviewer-bin/usr/lib/libreaderex_x64.so", RTLD_LAZY);

    struct link_map *lm = (struct link_map *)handle;
    printf("%lx\n", lm->l_addr);

    p_CAJFILE_OpenEx1 = dlsym(handle, "CAJFILE_OpenEx1");
    p_CAJFILE_CreateErrorObject = dlsym(handle, "CAJFILE_CreateErrorObject");
}


```

my_init 会在 main 函数之前执行，代码流程如下

*   首先`dlopen`把 so 加载到内存，并把 so 在内存中的基地址打印到屏幕，便于后续测试。
    
*   然后使用`dlsym`获取`CAJFILE_OpenEx1`和`CAJFILE_CreateErrorObject`函数的地址。
    

然后在 main 函数中就会构造参数调用目标函数

```
int main(int argc, char **argv)
{
    char buf[0x2D8];
    printf("main:%p\n", main);

    memset(buf, 0, 0x2D8);
    *(unsigned int *)buf = 0x2D8;
    // *(unsigned int *)(buf + 4) = 256;

    // *(char* *)(buf + 8) = p_CAJFILE_CreateErrorObject();

    char *ret = p_CAJFILE_OpenEx1(argv[1], buf);
    return 0;
}


```

代码逻辑很简单，首先构造`CAJFILE_OpenEx1`函数的第二个参数，然后把`argv[1]`作为文件路径传入函数。

然后编译一下

```
gcc CAJFILE_OpenEx1.c -o test_CAJFILE_OpenEx1_dbg -ldl -lheapasan -L libheapasan/  -g


```

编译后执行一下，可以看到正常执行完了，并打印出 so 的基地址和 main 函数的地址。

```
harness$ ./test_CAJFILE_OpenEx1_dbg ~/input.caj
string to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstringto intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to int
image base:0x7f6d87bff000
p_CAJFILE_OpenEx1:0x7f6d881e486c
main:0x555ed124cb71


```

接下来再使用 afl-qemu-trace 执行一下，获取一些地址用于 Fuzz，使用 afl-qemu-trace 执行一个可执行程序时，其进程的 so 的地址都是固定的。

```
harness$ ~/AFLplusplus-2.66c/afl-qemu-trace ./test_CAJFILE_OpenEx1_dbg ~/input.caj
string to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstringto intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to intstring to int
image base:0x400133e000
p_CAJFILE_OpenEx1:0x400192386c
main:0x4000000b71


```

可以看到`libreaderex_x64.so`的基地址为`0x400133e000`， `test_CAJFILE_OpenEx1_dbg` 的`main`函数的地址为`0x4000000b71`。

然后去 IDA 中查看`libreaderex_x64.so`中代码段的范围

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS3dac1KfAswgV1H8fWNyORSJ40H4Mxy41YI2I8czspicONhKibGM758Jw/640?wx_fmt=png)

所以可以得到`afl-qemu-trace`执行时`libreaderex_x64.so`中代码段的范围为

```
开始地址： 0x400133e000+0x3D4880 = 0x4001712880
结束地址： 0x400133e000+0x90984F = 0x4001c4784f


```

然后可以使用 AFL 进行测试了

```
export AFL_CODE_START=0x4001712880
export AFL_CODE_END=0x4001c4784f
export AFL_ENTRYPOINT=0x4000000b71
/home/hac425/AFLplusplus-2.66c/afl-fuzz -m none -Q -t 20000 -i in -o out -- ./test_CAJFILE_OpenEx1_dbg @@


```

其中设置的环境变量的作用如下

```
AFL_CODE_START 和 AFL_CODE_END 表示需要统计覆盖率的范围
AFL_ENTRYPOINT 表示开启forkserver的位置


```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSNe5l4D7hZaQ0OUQGBINE5Qx6JsKVjZCcWCHqic0UIHDiaUOTS9IPLuoA/640?wx_fmt=png)

**Fuzz UnCompressImage 函数**

### 在测试`CAJFILE_OpenEx1`时，去翻了一下`libreaderex_x64.so`里面的其他函数，在查看字符串时发现了一些源码路径。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSyCTJIONRM1ewPybjcrdEwUjLXAgZud3jUEFN7ULkXUUF9U9ZEYiamGg/640?wx_fmt=png)

拿路径去网上搜了一下，发现是用到了`Kakadu_V2.2.3`这个开源库，这个库很古老了（2008 年的），用于解析`jpeg2000`格式，版本老往往表示存在漏洞几率较大，而且`jpeg2000`格式很复杂，在其他软件中也发现了很多漏洞，于是下面仔细的看了下。  

下载到这个库的代码，然后一路回溯发现`libreaderex_x64.so`应该是在 `jpeg2000.cpp`里面实现了部分代码，最后一路跟到了`DecodeJpeg2000`函数，并基于开源代码把`DecodeJpeg2000`的参数基本弄清楚了。

继续往上跟`DecodeJpeg2000`，找到了`UnCompressImage`函数，这个函数应该是解析图片数据的统一接口了。

`CAJViewer`在解析 CAJ 等文件时，如果文件中嵌入了图片数据时，就会会使用 **libreaderex_x64.so** 中的`UnCompressImage`函数来对图片数据进行解析。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSIzVHo6VGU1FBwquKdwHHZx3NEXy3tswB0pbxmID8rib7n7M4iaawCwUA/640?wx_fmt=png)

函数的参数信息如下：

```
buffer: 保存从文件中提取出的图片数据
type: 图片的类型
buffer_length: 图片数据的长度
剩下两个参数a4,a5: 个人猜测可能是需要将图片缩放的大小


```

然后编写代码，`my_init`的主要逻辑和 `CAJFILE_OpenEx1`函数的一致，只是需要 hook 一些函数，避免比 Fuzz 识别为 crash，比如在代码里面有很多 assert，如果直接执行到这个函数的话，会被 afl 识别为 crash.

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS5vkKycYiaVltia2XM7vCne8KulTZRhkUf0s8GEzkXvZ6tTIFThI98NWg/640?wx_fmt=png)

  
因此这里使用 plt hook，把 libreaderex_x64.so 模块中的一些函数给 hook 了。

```
int my_assert_fail()
{
    printf("my_assert_fail\n");
    exit(1);
    return 0;
}

int my_cxa_throw()
{
    printf("my_cxa_throw\n");
    exit(1);
    return 0;
}

void my_init(void)
{
 ........................................
 ........................................
    plt_hook_function("libreaderex_x64.so", "__assert_fail", my_assert_fail);
    plt_hook_function("libreaderex_x64.so", "__cxa_throw", my_cxa_throw);
}


```

然后再 main 函数中调用目标函数

```
int main(int argc, char **argv)
{
    printf("main:%p\n", main);
    int f_sz = 0;
    char* buffer = read_to_buf(argv[1], &f_sz);
    char *ret = p_UnCompressImage(buffer, 4, f_sz, 100, 100);
    return 0;
}


```

然后其他的操作和`Fuzz` `CAJFILE_OpenEx1`函数时一致，只是环境变量需要重新设置

```
/home/hac425/AFLplusplus-2.66c/afl-fuzz -m none -Q -t 20000 -i image_fuzz/ -o UnCompressImageOutput -- ./test_UnCompressImage @@


```

**部分漏洞分析**
----------

**CImage::LoadBMP 内存为初始化漏洞**
----------------------------

Cajviewer For Linux 在解析 BMP 图片时会进入 CImage::LoadBMP 函数，该函数中存在内存未初始化漏洞。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS73xjKWpDSibLtuUSdOyR8Rga91bqKxk57J5KyWbkAPKMh5cJlVwibTdw/640?wx_fmt=png)

函数的流程如下

1.  第 8 行，调用 BaseStream::streamLength 获取文件的大小。
    
2.  第 9 行，调用 FileStream::read 从文件中读出 14 字节的文件头。
    
3.  第 10 行，调用 gmalloc 分配内存用于存放文件的其他数据，这里实际上是直接调用 malloc 分配内存。
    
4.  第 12 行，这里将分配的内存**没有初始化**直接传入 FindDIBBits，该函数计算一个地址保存到 this->DIBBits 域。
    
5.  第 14 行，这里会从 this->DIBBits 中读取数据，导致 crash。
    

下面看看 FindDIBBits 的实现

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS3HhtKHTPg09pS6TLcAPPDau0dXA3CoZfhDr7wDSqJ6R0xo067oyUFQ/640?wx_fmt=png)

这里取出 a1 的**开始 4 个字节**作为一个**偏移值 v1**，然后调用 PaletteSize，这个函数的返回值的可以为 0，128 等数字值。  

由于 **a1 这个内存没有初始化，故 v1 有可能会很大**，进而导致 FindDIBBits 会返回一个**越界的地址**。

然后在 **CImage::CalibrateColor** 中就会去访问这个内存。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSKb01bjNJPGW5OUWmVbtVLF2nibibGX7KO8TkpXe3kzlwtHTNL4JAbg9g/640?wx_fmt=png)

### **CImage::DecodeJbig 越界读写漏洞**

CajViewer 在解析 CAJ 等文件时，如果需要解析文件中嵌入的图片数据时，会使用 **libreaderex_x64.so** 中的函数来对图片进行解析，其中如果带解析的文件类型为 Jbig 文件时，会进入 **CImage::DecodeJbig** 函数进行解析：

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSEQXEruriaM0L99a4wSAibAgqxJYsC3HWZ9uRPW7KtJ4cuZP4Nic0SXXMQ/640?wx_fmt=jpeg)

其中重要函数的参数和作用如下：

1.  buf: 保存从文件中提取出的图片数据
    
2.  len: buf 的长度
    

其中 buf 一开始是一个 **JbigInfo** 的结构，结构体的定义如下：

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSS4zogicFHT7oB6r1BndYTcQtL5fXI09pGh6av7bnrO6ekjJo2TTQJicA/640?wx_fmt=jpeg)

然后然后会进入 **CImage::CImage** 进行简单的文件解析。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSn63Dh8Cq0SMcZRjC47hgATnJ6gyYUkiaesSYQCfcQBfGeBWamNgGY9g/640?wx_fmt=jpeg)

首先使用 **JbigInfo** 中的字段计算一个 sz, 然后使用 gmalloc 分配内存，之后会使用 memcpy 拷贝数据。

漏洞位于在计算 sz 时会导致**整数溢出**，进而导致会分配一个小于 4LL * (1 << jbig_info->width2) 的内存，然后在下面 memcpy 时会导致**越界写。**

此外整个过程没有**校验 jbig_info 的长度**，所以会导致**越界读。**

**HN 文件格式逆向**
=============

本节介绍对`cajviewer`中对 HN 文件格式的逆向分析并介绍如何编写相应的 010editor 模板，最后介绍通过分析如何构造 POC，触发 cajviewer 在解析 HN 文件中的图片时存在的漏洞。HN 文件是 cajviewer 支持的其中一种文件格式，这个文件类似于 PDF，可以包含文字、图片等，下图是一个 HN 文件应用模板后的截图，具体的分析过程请看正文部分。
===============================================================================================================================================================================================

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSNXBGDulBaiaiaUibvEHk0yGlNM83AJ3Ltq69nNFRfu4sz0Q8ib0uLWYfsw/640?wx_fmt=png)

样例文件和 010 模板

```
https://github.com/hac425xxx/cajviewer-fuzz-data
https://github.com/hac425xxx/cajviewer-fuzz-data/releases/download/2020-8-2/sample.7z


```

**解析文件头**
---------

基于上文的分析，我们知道`cajviewer`使用`CAJFILE_OpenEx1`函数来打开和解析一个文件，因此这个函数就是我们的分析入口.
-----------------------------------------------------------------------

```
CCAJReaderStruct *__fastcall CAJFILE_OpenEx1(char *fpath, char *a2)
{

  file_type = CAJFILE_GetDocTypeEx1(fpath, a2, 0LL);// 获取文档类型
  switch ( file_type )
  {
    case 1u:
    case 2u:
    case 8u:
    case 0xAu:
    case 0x1Bu:
      ccaj_reader = operator new(0x210uLL);
      a2 = v12;
      CCAJReader::CCAJReader(ccaj_reader, v12); // 根据文件类型，构造Reader对象


```

函数首先调用`CAJFILE_GetDocTypeEx1`根据文件头和文件名返回一个表示文档类型的 int 值，对于样本文件来说会进入`CCAJReader::CCAJReader` 构造文档对象用于后续的解析。

通过分析类的构造函数可以大概了解对象的内存布局，比如通过 new 函数的参数可以知道 `CCAJReader::CCAJReader` 对象的大小为 `0x210`字节，下面看看类的构造函数

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSNwzbNdOt9iaI8ia1TEZR2HWgyVnnKiasRIKFS19CLh0ngThOB8MGtj0ng/640?wx_fmt=png)

首先赋值虚表为 `vtable for CCAJReader + 2`，其实就是`0xB19B0`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSt645mOc1alczXdX0NbkHO35k6w2xibw2lwfhklob4jHbLdBFxcSkqbA/640?wx_fmt=png)

我们可以把这个抠出来，作为一个结构体以便后续分析

```
struct CCAJReaderVtableStruct
{
  void *_ZN10CCAJReaderD2Ev;
  void *_ZN10CCAJReaderD0Ev;
  ....................................
  ....................................
  void *_ZN7CReader16InternalFileOpenEPKc;
  void *_ZN7CReader18InternalFileLengthEPv;
  void *_ZN7CReader16InternalFileSeekEPvll;
  void *_ZN7CReader16InternalFileReadEPvS0_l;
  void *_ZN7CReader17InternalFileCloseEPv;
  void *_ZN7CReader19InternalFileIsReadyEPKcijj;
};


```

然后设置`CCAJReaderStruct`的`vtbl`的类型为`CCAJReaderVtableStruct*`，这样再看虚函数调用时就可以很方便的定位到目标函数，其他用到的类也用这种方式逆向即可，继续往下看

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSTxh0lFXR0dRHMVYiccGY96QKCggvSE6TMZ4d6N3jLHKXbcabPkem2zA/640?wx_fmt=png)

这里调用`CCAJReader::Open`对文件进行初步解析，该函数实际会进入`CAJDoc::Open`读取文件内容并解析  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSdqZneox7vd5E06JS4yHIBicrLMIlS2iaGBHM6G2Oic5UeiccaxZLGjTCJQ/640?wx_fmt=png)

首先这里调用 BaseStream::getStream 来创建一个 stream 对象，在 cajviewer 里面通过 stream 对象来从各种来源读取数据，比如网络、文件、内存等。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSRic7221xMydh8mbxhtR2muWH6Gv0L5nE3MrhMdh4PsF8IQB4bVd4VDQ/640?wx_fmt=png)

就我们这个例子实际构建的对象为`FileStream`，创建完后就会调用`FileStream::open`和`FileStream::seek`打开文件并把文件指针重定向到文件开头。  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSAK3DKOISicNwAibCVicRhicOFyMwO0Rf8ngQmd4E5ia7G1kfGe2uGwfY9gg/640?wx_fmt=png)

然后会进入 `CAJDoc::OpenNHCAJFile` 进行具体的解析，第二个参数为 0，在该函数里面首先会调用`FileStream::read`读取文件开头的`0x88`字节，并进行简单的判断  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSXWPdCJIaQZYoVH1XZDib5RjpVkvWC3oWicZGFsvImaia5OIpK6MM7P3aQ/640?wx_fmt=png)

校验了前 0x88 字节的部分数据后，会再次读取 0x50 字节的数据 (0x10+0x40)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS9heOvLCiaI3kTtYjsq3uXxks3es0Glkk7Vic2TQ3qGAYJk6KhER6SF2g/640?wx_fmt=png)

其中`buffer_0x10.page_count`表示文件中包含的页面数，这个通过观察下面的引用来推测，继续往下

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSxObeyvamyLOmMvNmAVITgyQHtCl5lCbbB7BDMickk3UddcTrngveoSA/640?wx_fmt=png)

这里首先校验`buffer_0x10.field_0`是否大于 `0x18f` ，如果大于`0x18f`就会再次读取一些内容作为元数据，然后会根据这个值设置`item_size`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSVO4EftS5vplA6s4qicJaFPWBCfduWL1RFNiaqo3paat8sKXBbocLsDCQ/640?wx_fmt=png)

首先`cajdoc->current_offset`在前面读取内容时会进行调整，从`cajdoc->current_offset`开始就是表示`CAJPage`的信息数组，数组中每一项的大小为 `cajdoc->item_size`，类的构造函数的最重要的参数是第三个参数，表示该`CAJPage`在文件中的偏移，后面解析时会用到这些。

至此我们可以得到文件开头的格式为

```
0x88字节的hn_header
0x10字节的buffer_0x10;
0x40字节的buffer_0x40;
如果buffer_0x10.field_0 > 0x18F，后面还会跟一个 0x84字节的buffer_0x84 和 308 * buffer_0x84.count 字节的内存
然后是buffer_0x10.page_count个page_info结构，每个结构的大小item_size为12或者20，item_size 根据buffer_0x10.field_0来判断


```

此时我们可以写一个简单的 010editor 模板，来解析文件头的数据

```
typedef struct{
    ubyte data[0x88];
}HN_FILE_HEADER;

typedef struct{
    uint32 field_0;
    uint32 field_4;
    uint32 page_count;
    uint32 field_0xc;
}BUFFER_0X10;

typedef struct{
    ubyte gap[12];
    uint16 w1;
    uint16 w2;
    uint32 unknown_dword;
    uint32 dword_20;
    ubyte data[40];
}BUFFER_0X40;

typedef struct{
    ubyte data[0x80];
    uint32 count;
}BUFFER_0X84;

local uint32 item_size = 12;

HN_FILE_HEADER hn_header;
BUFFER_0X10 buffer_0x10;
BUFFER_0X40 buffer_0x40;

local uint64 page_info_offset = FTell();

if(buffer_0x10.field_0 > 0x18F)
{
    BUFFER_0X84 buffer_0x84;
    local uint64 cur_pos = FTell();    
    page_info_offset = 308 * buffer_0x84.count + cur_pos;
}

if(buffer_0x10.field_0 <= 0xC7)
{
    item_size = 12;
}
else
{
    item_size = 20;
}

FSeek(page_info_offset);


```

这里有几个关键的点，在 010editor 的模板中类型定义和 local 开头的局部变量不会导致文件指针的移动，当直接定义结构体变量时就会导致 010editor 读取文件内容并进行解析。

```
HN_FILE_HEADER hn_header;


```

比如这个代表`010editor`会读取`0x88`字节到`hn_header` 并会移动文件指针，最后会使用`FSeek(page_info_offset)`把文件指针移动到`page_info`开始的位置，详细的教程和语法可以看下面的链接

```
https://bbs.pediy.com/thread-257797.htm


```

**解析页面数据**
----------

解析完文件头的数据后会调用`CAJPage::LoadPageInfo`解析具体的页面信息
---------------------------------------------

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSXSfYp6m4ia8eyNnp28qC2ic54Ehex4MHmr5YLAcMFSWHOVCk8Rnp5Qgg/640?wx_fmt=png)

函数逻辑比较简单，就是`FileStream::seek`到指定的文件偏移，然后读取`item_size`数据用于`page_info`，然后会把`page_info`的数据保存到当前 page 对应的结构体里面, `page_info`的结构如下

```
struct page_info
{
  int file_offset;  // page数据在文件中的偏移
  int size; // page数据的大小
  __int16 pic_count;  // page中的图片个数
  __int16 field_A;
  __int64 field_C;
};


```

然后会跳到`page_info.file_offset`，读取`page`数据的前 0x20 个字节，然后从里面解析了一些数据，用途不明。

加载完`page_info`后会调用`CAJPage::LoadPage`加载页面的文本数据

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSLodG8H6r8NxQrrbbEMneqDVC6Qia2vZn1ZGe5icsTolicbzPr4EfevNrw/640?wx_fmt=png)

这里首先跳转到`page`数据所在的文件偏移，然后把页面的数据读出来

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSibVphtIdEc9Toaw7l1nrGZ9zEbz1eJl3erNXwuAXFkiaYw1U8gjsctHg/640?wx_fmt=png)

这里对文件内容解析，首先从头 8 个字节里面解析出当前`page`的`heigh`和`width`，然后后面是具体的文本数据，然后判断文本数据开头是否有`COMPRESSTEXT`，如果是表示文本数据是压缩过的会使用`UnCompress`对文本数据进行解压。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSBxcswvW44paKH5Leicc0dmiaCpxbkwJCVp9ibRxMt1f7MZgb0FJxbgd9w/640?wx_fmt=png)

解析完`page`的文本数据后会把`page`的图片数据在文件的起始偏移记录在`page->pic_info_foffset`里面，解析完之后会进入`CAJPage::LoadPicInfo`加载图片的元数据

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS3WIibk4xod4VGlhINPCOJEgxibzCgOhiazqTZxypicx9VYicJSesTTDny0w/640?wx_fmt=png)

这里会根据`page->page_info.pic_count`创建`CAJ_FILE_PICINFO`数组，数组中的每个元素为`pic_info`结构，结构体定义如下

```
struct pic_info_struct
{
  int type;  // 图像类型
  int offset; // 图像数据在文件中的偏移
  int size; // 图像数据的大小
};


```

通过这个函数每个`page`的图片信息会保存到`page->caj_picinfo_list`里面，然后会在`CAJPage::LoadImage`里面对页面的某个图片数据进行解析

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSNZHRTfS3tibKeD5gibiaNevfdItpCN2UmA6TfxvJ3rTXCXh5MOCvCQ9pg/640?wx_fmt=png)

函数的流程也简单，首先根据图片的索引在`cajpage->caj_picinfo_list`里面找到图片的`picinfo`结构，然后根据该结构读取图片的数据并使用`UnCompressImage`对图片数据进行解析。

至此我们可以得到 page 数据的组织方式如下

首先在文件头后面是`buffer_0x10.page_count`个`page_info`结构，`page_info`结构里面记录了页面的数据所在的文件偏移、内容的大小以及页面包含图片的个数，然后根据这些信息可以得到页面的文本数据和图片数据（图片数据紧跟在文本数据的后面）。

这部分的 010 模板如下

```
typedef struct{
    uint32 type;
    uint32 file_offset;
    uint32 size;
    local uint64 backup_offset = FTell(); 

    FSeek(file_offset);  // move to data offset
    ubyte pic_data[size];   // page_data
    FSeek(backup_offset);  // move back
}PICINFO;


typedef struct (uint32 size){
    PAGE_CONENT_HEADER page_hdr;


    local char tmp[12];
    ReadBytes(tmp, FTell(), 12);

    if(Memcmp(tmp, "COMPRESSTEXT", 12) == 0)
    {
        char compress_sig[12];
        uint32 decompressed_size;
        char compressed_data[size - 12 - 4 - sizeof(PAGE_CONENT_HEADER)];
    }
    else
    {
        ubyte page_text_content[size - sizeof(PAGE_CONENT_HEADER)];   // page_data
    }
    
}PAGE_CONTENT;

typedef struct _PAGE_INFO_ITEM{
    uint32 file_offset;
    uint32 size;
    uint16 pic_count;
    uint16 field_A;
  
    if(item_size==20)
    {
        uint64 field_C;
    }

    local uint64 backup_offset = FTell(); 
    
    FSeek(file_offset);  // move to data offset
    
    PAGE_CONTENT page_content(size);
    
    local uint32 i = 0;

    while(i < pic_count)
    {
        PICINFO pic_info;
        i++;
    }

    FSeek(backup_offset);  // move back
}PAGE_INFO_ITEM;


```

解析完后的效果图如下：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xS4MplQArBsC4SURjMTk0VKhp2PxaX6CNNI3iaYTsvY6Q3xQFf3npKTxA/640?wx_fmt=png)

**构造 POC 的技巧**

通过前面的分析可知`UnCompress`会对页面文本数据进行解压，简单的看下`UnCompress`的实现我们可以知道该函数调用了`zlib 1.1.3`版本解压文本数据，这个版本有很多漏洞，如果我们想触发`UnCompress`的漏洞就可以把文件中压缩文本数据替换成 zlib 的 poc 数据即可

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSlEGXvibjia2nvfQ2kNiabelflzVs07YqTvouppOoKiaeVfOnvO2jnFApMQ/640?wx_fmt=png)

如果是要触发解析图片的的漏洞时也是一样的思路，替换掉正常文件中的某个图片数据即可

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8JhZHfib5mVLEt3kFhQic96xSJ7T9J0pdz4ISm6gEX8W7flKOVliajiavuiaticCicYIxd7PNTJ9yjmrbKHQ/640?wx_fmt=png)

**总结**
======

本文介绍了如何分析一个软件并使用 afl qemu 模式来测试闭源二进制，并分析了其中的一些漏洞，最后介绍了逆向`HN`文件格式的过程，并介绍如何编写`010editor`模板来辅助`poc`构造。
===================================================================================================

**附录**
======

```
https://github.com/hac425xxx/cajviewer-fuzz-data


```

**最新动态**

[2020 年 OPPO 安全大事记](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486790&idx=1&sn=2ed3360ec9ac2f856cdb9ee392a14c81&chksm=fa7b0c0acd0c851c248a0a72a732d39ef7a1b179dde6e931fbb7dfcd80a44ea74972f81268d7&scene=21#wechat_redirect)

[年度奖励 | 期待已久的年终奖出炉啦~](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486779&idx=1&sn=fa9faf269510fa521d136b5d6139abd7&chksm=fa7b0c77cd0c856156b61b1465a412336d484581062598a5351cc20d013b8a90bb1afa576683&scene=21#wechat_redirect)  

[CVE-2020-16040: Chromium V8 引擎整数溢出漏洞分析](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486771&idx=1&sn=d79a5348255121d247a47009321e01ef&chksm=fa7b0c7fcd0c85699d6ab93c95935a3af98ec88023c8d95713814221ceaab082736df954e8a2&scene=21#wechat_redirect)

[揭秘 QUIC 的性能与安全](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486735&idx=1&sn=233bf264097c4742b0d76e1f81933cd5&chksm=fa7b0c43cd0c8555fec4c5e0617520237bcea45f97706c1d6b3b4d27cb736509d1b4272b50db&scene=21#wechat_redirect)

[OSRC 2 周年第二弹——第五次奖励升级](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486080&idx=1&sn=d863c75ac9ffece2dd6078d89f23eae7&chksm=fa7b0bcccd0c82daf627c867929239a9567b99eb0c7c0c2288b11da50b191d0f432f5a7b6500&scene=21#wechat_redirect)

[OPPO 互联网 DevSecOps 实践](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486168&idx=1&sn=9c2b467885e449d630008a97bd18daee&chksm=fa7b0b94cd0c82820314716c179f7cd7246309dd4c0ee19042caac251c2fa13b8b405d3c0c7f&scene=21#wechat_redirect)

[OPPO 安全最新招聘信息](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486504&idx=2&sn=a04134339307eed5812430da859c39df&chksm=fa7b0d64cd0c8472e17dc20706ff81e0d4bbb2415e5cf72fe2a7e65d1f50e5c5be851853b305&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)