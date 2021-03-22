> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266546.htm)

1. 前言
-----

一道 so 文件动态加解密的 CrackMe，运行时解密要执行的函数，且在执行后立马加密

*   CrackMe：dex 文件加的数字壳，so 文件无壳，因为反调试，所以 so 文件采用全静态分析
*   分析环境：
    *   脱壳工具：[FART](https://bbs.pediy.com/thread-252630.htm)
    *   GDA
    *   IDA
    *   Frida
    *   PyCharm
    *   VSCode

2. 分析过程
-------

![](https://bbs.pediy.com/upload/attach/202103/825187_6WHA6Q5N5JCAGK2.jpg)

### 2.1 脱壳

![](https://bbs.pediy.com/upload/attach/202103/825187_A3CEHU6EK9BMGUK.jpg)

 

拿到 FART 定制 ROM 下跑，得到想要的 dex 文件，数字壳抹去了前八个字节的 dex 文件魔数，需要填充一下，才能用 GDA 进行解析

### 2.2 定位校验函数

![](https://bbs.pediy.com/upload/attach/202103/825187_JVTU9KYWUTAG2H3.jpg)

 

![](https://bbs.pediy.com/upload/attach/202103/825187_BJE5MEZUY5RG6N5.jpg)

 

从上图可知，校验函数为`libnative-lib.so`文件中的`test`函数

### 2.3 分析 so 文件

**首先分析 so 文件提前加载的三处函数（`init、init_array、JNI_OnLoad`）**

 

用`readelf -d`查看是否有`init、init_array`

 

![](https://bbs.pediy.com/upload/attach/202103/825187_WHFJQ9WFMMSYY7R.jpg)

 

发现只有`init_array`，用 IDA 查看`init_array`数组中的函数

 

![](https://bbs.pediy.com/upload/attach/202103/825187_Y9UKDS7FEP7C4XU.jpg)

 

`datadiv_decode4192348989750430380`函数的作用是解密字符串

 

![](https://bbs.pediy.com/upload/attach/202103/825187_EPR6A9QABPWMD74.jpg)

 

**接着分析`JNI_OnLoad`函数，导入`jni.h`头文件，用于解析 JNI 函数**

 

![](https://bbs.pediy.com/upload/attach/202103/825187_YXY5R8SFTZZVZ76.jpg)

 

**接着分析`ooxx`函数**

 

![](https://bbs.pediy.com/upload/attach/202103/825187_SUWCH2QNZFUYRB8.jpg)

 

`sub_8930`函数的内容如下：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_MNDM8UCR9NGNDS7.jpg)

 

其中`sub_8A88`函数的作用是获取 so 文件的加载基址，如下：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_TE2VSM83EZMVEH8.jpg)

 

获取 so 文件的加载基址的方法是，通过读取 CrackMe 进程的内存映射文件`maps`，然后通过搜索切割字符串得到的，`maps`文件的内容如下：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_ZSTSSAURRDWJAK7.jpg)

 

`sub_8930`函数接着调用了`sub_8B90`函数用于获取`xxoo`函数的相对虚拟地址和大小，如下：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_9QM6PM85R6X6TPV.jpg)

 

![](https://bbs.pediy.com/upload/attach/202103/825187_2CRV8G6KWBYM4TW.jpg)

 

其中步骤 5——**通过计算，得到 xxoo 函数在符号表中的索引`k`**中使用的算法和[文章：简单粗暴的 so 加解密实现](https://bbs.pediy.com/thread-191649.htm)中第四部分——**基于特定函数的加解密实现**介绍的查找函数的算法完全一致，可以导入`elf.h`头文件解析 ELF 文件的结构体

 

在`sub_8930`函数中，根据上面得到的 so 文件的加载基址、`xxoo`函数的相对虚拟地址和大小等信息，接着就是修改内存属性，解密`xxoo`函数，还原内存属性，最后刷新指令缓存，分析完成后的`sub_8930`函数如下：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_2F6C4RR4NPUJDWA.jpg)

 

其中解密用到的密钥存储在`byte_1C180`中，是在 bss 段，在文件中是未初始化的，所以我们需要在运行时，从内存中 dump 下来

3. 解密函数
-------

### 3.1 解密需要的数据

使用打开文件的方式进行解密，而不是运行时解密，所以需要以下数据

*   `xxoo`函数的文件偏移 (`xxoo_offset`)：
*   `xxoo`函数的大小 (`xxoo_size`)
*   密钥 (`xor_array`)

**获取`xxoo`函数的文件偏移 (`xxoo_offset`)**

 

`xxoo`函数的文件偏移 = `.txt`段的文件偏移 + `xxoo`函数相对于`.txt`段的文件偏移  
`xxoo`函数相对于`.txt`段的文件偏移 = `xxoo`函数的相对虚拟地址 - `.txt`段的相对虚拟地址

 

通过上面两个公式可得  
`xxoo`函数的文件偏移 = `.txt`段的文件偏移 + `xxoo`函数的相对虚拟地址 - `.txt`段的相对虚拟地址

 

`.txt`段的文件偏移和`.txt`段的相对虚拟地址在`.txt`的区段头中，`xxoo`函数的相对虚拟地址在符号表中，如下：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_B8QY2DZ3JZH95NZ.jpg)

 

![](https://bbs.pediy.com/upload/attach/202103/825187_PX3K7TC7RM6B6AW.jpg)

 

所以`xxoo_offset` = 0x8dc5

 

**获取`xxoo`函数的大小 (`xxoo_size`)**

 

如上图，`xxoo_size` = 584

 

**获取密钥 (`xor_array`)**

*   密钥在内存中的起始地址：so 文件的加载基址 + 0x1C180
*   密钥的大小：xxoo_size - 61 - 59 = 464

根据上述信息，通过 frida 脚本 dump 内存即可得到密钥，脚本如下：

```
//获取用于异或解密的密钥
Java.perform(function () {
    var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
    //1.获取app的文件目录files，/data/user/0/$package_name/files/
    var dir = currentApplication.getApplicationContext().getFilesDir().getPath();
    //2.获取so文件信息
    var libso = Process.getModuleByName("libnative-lib.so");
    var byte_1C180_addr = 0x1C180
    //3.获取用于异或解密的密钥的起始地址
    var xor_array_base  = ptr(Number(libso.base) + byte_1C180_addr)
    //4.用于异或解密的密钥的大小
    var xor_array_size = 464;
    console.log("[xor_array_base]: ", xor_array_base);
    console.log("[xor_array_size]: ", ptr(xor_array_size));
    //5.拼接密钥文件路径
    var xor_array_path = dir + "/" + "xor_array" + "_" + xor_array_base + "_" + ptr(xor_array_size);
    //6.打开密钥文件
    var file_handle = new File(xor_array_path, "wb");
    if (file_handle && file_handle != null) {
        //7.修改密钥所在内存的属性为可读
        Memory.protect(xor_array_base, xor_array_size, 'r');
        //8.读取密钥到缓冲区
        var xor_array_buffer = xor_array_base.readByteArray(xor_array_size);
        //9.把缓冲区内容写入文件
        file_handle.write(xor_array_buffer);
        //10.刷新缓存
        file_handle.flush();
        //11.关闭文件
        file_handle.close();
        console.log("[dump]: ", xor_array_path);
    }
});

```

### 3.2 解密脚本

```
import mmap
 
# 密钥数组
xor_array = [0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F,
             0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18, 0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x1E, 0x1F,
             0x20, 0x21, 0x22, 0x23, 0x24, 0x25, 0x26, 0x27, 0x28, 0x29, 0x2A, 0x2B, 0x2C, 0x2D, 0x2E, 0x2F,
             0x30, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x3A, 0x3B, 0x3C, 0x3D, 0x3E, 0x3F,
             0x40, 0x41, 0x42, 0x43, 0x44, 0x45, 0x46, 0x47, 0x48, 0x49, 0x4A, 0x4B, 0x4C, 0x4D, 0x4E, 0x4F,
             0x50, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58, 0x59, 0x5A, 0x5B, 0x5C, 0x5D, 0x5E, 0x5F,
             0x60, 0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68, 0x69, 0x6A, 0x6B, 0x6C, 0x6D, 0x6E, 0x6F,
             0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78, 0x79, 0x7A, 0x7B, 0x7C, 0x7D, 0x7E, 0x7F,
             0x80, 0x81, 0x82, 0x83, 0x84, 0x85, 0x86, 0x87, 0x88, 0x89, 0x8A, 0x8B, 0x8C, 0x8D, 0x8E, 0x8F,
             0x90, 0x91, 0x92, 0x93, 0x94, 0x95, 0x96, 0x97, 0x98, 0x99, 0x9A, 0x9B, 0x9C, 0x9D, 0x9E, 0x9F,
             0xA0, 0xA1, 0xA2, 0xA3, 0xA4, 0xA5, 0xA6, 0xA7, 0xA8, 0xA9, 0xAA, 0xAB, 0xAC, 0xAD, 0xAE, 0xAF,
             0xB0, 0xB1, 0xB2, 0xB3, 0xB4, 0xB5, 0xB6, 0xB7, 0xB8, 0xB9, 0xBA, 0xBB, 0xBC, 0xBD, 0xBE, 0xBF,
             0xC0, 0xC1, 0xC2, 0xC3, 0xC4, 0xC5, 0xC6, 0xC7, 0xC8, 0xC9, 0xCA, 0xCB, 0xCC, 0xCD, 0xCE, 0xCF,
             0xD0, 0xD1, 0xD2, 0xD3, 0xD4, 0xD5, 0xD6, 0xD7, 0xD8, 0xD9, 0xDA, 0xDB, 0xDC, 0xDD, 0xDE, 0xDF,
             0xE0, 0xE1, 0xE2, 0xE3, 0xE4, 0xE5, 0xE6, 0xE7, 0xE8, 0xE9, 0xEA, 0xEB, 0xEC, 0xED, 0xEE, 0xEF,
             0xF0, 0xF1, 0xF2, 0xF3, 0xF4, 0xF5, 0xF6, 0xF7, 0xF8, 0xF9, 0xFA, 0xFB, 0xFC, 0xFD, 0xFE, 0xFF,
             0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F,
             0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18, 0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x1E, 0x1F,
             0x20, 0x21, 0x22, 0x23, 0x24, 0x25, 0x26, 0x27, 0x28, 0x29, 0x2A, 0x2B, 0x2C, 0x2D, 0x2E, 0x2F,
             0x30, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x3A, 0x3B, 0x3C, 0x3D, 0x3E, 0x3F,
             0x40, 0x41, 0x42, 0x43, 0x44, 0x45, 0x46, 0x47, 0x48, 0x49, 0x4A, 0x4B, 0x4C, 0x4D, 0x4E, 0x4F,
             0x50, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58, 0x59, 0x5A, 0x5B, 0x5C, 0x5D, 0x5E, 0x5F,
             0x60, 0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68, 0x69, 0x6A, 0x6B, 0x6C, 0x6D, 0x6E, 0x6F,
             0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78, 0x79, 0x7A, 0x7B, 0x7C, 0x7D, 0x7E, 0x7F,
             0x80, 0x81, 0x82, 0x83, 0x84, 0x85, 0x86, 0x87, 0x88, 0x89, 0x8A, 0x8B, 0x8C, 0x8D, 0x8E, 0x8F,
             0x90, 0x91, 0x92, 0x93, 0x94, 0x95, 0x96, 0x97, 0x98, 0x99, 0x9A, 0x9B, 0x9C, 0x9D, 0x9E, 0x9F,
             0xA0, 0xA1, 0xA2, 0xA3, 0xA4, 0xA5, 0xA6, 0xA7, 0xA8, 0xA9, 0xAA, 0xAB, 0xAC, 0xAD, 0xAE, 0xAF,
             0xB0, 0xB1, 0xB2, 0xB3, 0xB4, 0xB5, 0xB6, 0xB7, 0xB8, 0xB9, 0xBA, 0xBB, 0xBC, 0xBD, 0xBE, 0xBF,
             0xC0, 0xC1, 0xC2, 0xC3, 0xC4, 0xC5, 0xC6, 0xC7, 0xC8, 0xC9, 0xCA, 0xCB, 0xCC, 0xCD, 0xCE, 0xCF,
             ]
 
file_name = "libnative-lib.so"
with open(file_name, "r+b") as file_descriptor:
    memory_map = mmap.mmap(file_descriptor.fileno(), 0)
    # xxoo函数的文件偏移
    xxoo_offset = 0x8dc5
    # xxoo函数的大小
    xxoo_size = 584
    # 被解密的代码的相对起始地址
    begin_relative_addr = xxoo_offset + 59
    # 被解密的代码的相对结束地址+1
    end_relative_addr = xxoo_offset + xxoo_size - 61
    # size和xxoo_size相等
    size = end_relative_addr - begin_relative_addr
    print("begin_relative_addr: 0x%x, end_relative_addr: 0x%x, size: %d"
          % (begin_relative_addr, end_relative_addr, size))
    for i in range(size):
        original_byte = int.from_bytes(memory_map[begin_relative_addr + i:begin_relative_addr + i + 1], "little")
        modified_byte = original_byte ^ xor_array[i]
        memory_map[begin_relative_addr + i:begin_relative_addr + i + 1] = bytes([modified_byte])
        read_modified_byte = int.from_bytes(memory_map[begin_relative_addr + i:begin_relative_addr + i + 1], "little")
        print("i: %d, original_byte: 0x%02x, modified_byte: 0x%02x, read_modified_byte: 0x%02x"
              % (i, original_byte, modified_byte, read_modified_byte))
    memory_map.flush()
    memory_map.close()

```

![](https://bbs.pediy.com/upload/attach/202103/825187_6MZWS9H6353W288.jpg)

 

解密过后的`ooxx`函数：

 

![](https://bbs.pediy.com/upload/attach/202103/825187_RHBWF6PG66B6RC8.jpg)

 

**flag 即是`kanxuetest`**

4. 附件
-----

见附件

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 3 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)

最后于 3 天前 被 genliese 编辑 ，原因：

上传的附件：

*   [1.apk](javascript:void(0)) （2.44MB，12 次下载）