> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-260547.htm)

> [原创] 记一次 so 文件动态解密

写在前面
----

整个程序基本上就是一个 动态注册 + so 函数加密 的逻辑，中间加了一些`parser`的东西

 

主要考察了`elf`文件结构的一些知识以及在攻防对抗中防止`IDA`静态分析的姿势

题目描述
----

找到 flag

WriteUp
-------

360 加固, 先脱壳，看入口函数`MainActivity`

 

![](https://bbs.pediy.com/upload/attach/202005/715334_VEFDNVQWEDJC5AF.png)

 

具体的逻辑写到`so`里了，使用`IDA`打开`so`文件, 先看有没有`.init`和`.init_array`, 发现只有`.init_array`节，

 

![](https://bbs.pediy.com/upload/attach/202005/715334_4VMKEDGQ5H53MHA.jpg)

 

跟进去一看又是字符串解密函数，解密之后，代码如下（这里我根据解密后的数据进行了重命名）

```
unsigned int datadiv_decode4192348989750430380()
{
 
  v29 = 0;
  do
  {
    v0 = v29;
    Find_ooxx_failed[v29++] ^= 0x14u;
  }
  while ( v0 < 0x10 );
  v28 = 0;
  do
  {
    v1 = v28;
    mem_privilege_change_failed[v28++] ^= 0xD3u;
  }
  while ( v1 < 0x1B );
  v27 = 0;
  do
  {
    v2 = v27;
    kanxuetest[v27++] ^= 0x63u;
  }
  while ( v2 < 0xA );
  v26 = 0;
  do
  {
    v3 = v26;
    Hello_from_Cjiajia[v26++] ^= 0x3Fu;
  }
  while ( v3 < 0xE );
  v25 = 0;
  do
  {
    v4 = v25;
    test[v25++] ^= 0xF3u;
  }
  while ( v4 < 4 );
  v24 = 0;
  do
  {
    v5 = v24;
    sig_Ljava_lang_Object_Z[v24++] ^= 0xFAu;
  }
  while ( v5 < 0x15 );
  v23 = 0;
  do
  {
    v6 = v23;
    com_kanxue_test_MainActivity[v23++] ^= 0x2Du;
  }
  while ( v6 < 0x1C );
  v22 = 0;
  do
  {
    v7 = v22;
    maps[v22++] ^= 0xF5u;
  }
  while ( v7 < 0xD );
  v21 = 0;
  do
  {
    v8 = v21;
    r[v21++] ^= 0xF8u;
  }
  while ( !v8 );
  v20 = 0;
  do
  {
    v9 = v20;
    open_failed[v20++] ^= 0xE6u;
  }
  while ( v9 < 0xB );
  v19 = 0;
  do
  {
    v10 = v19;
    heng[v19++] ^= 0x66u;
  }
  while ( !v10 );
  v18 = 0;
  do
  {
    v11 = v18;
    Find__dynamic_segment[v18++] ^= 0x2Du;
  }
  while ( v11 < 0x15 );
  v17 = 0;
  do
  {
    v12 = v17;
    Find_needed__section_failed[v17++] ^= 9u;
  }
  while ( v12 < 0x1C );
  v16 = 0;
  do
  {
    v13 = v16;
    basic_string[v16++] ^= 0x9Eu;
  }
  while ( v13 < 0xC );
  v15 = 0;
  do
  {
    result = v15;
    allocate_exceeds_maximum_supported_size[v15++] ^= 0xDBu;
  }
  while ( result < 0x43 );
  return result;
}

```

回过来看`JNI_Onload`函数,

 

![](https://bbs.pediy.com/upload/attach/202005/715334_R28BX4HQ7WT4985.png)

 

其实就是将`native`函数`test`函数动态注册到`ooxx`函数，直接看`ooxx`函数

 

![](https://bbs.pediy.com/upload/attach/202005/715334_Z8R9XQ9RTCG6743.jpg)

 

可以发现除了调用了`sub_8930`之外，就是一堆垃圾代码，先跟进`sub_8930`函数

 

![](https://bbs.pediy.com/upload/attach/202005/715334_CEXSSHQ95XYAV3K.jpg)

 

这里我把函数分为三块，先看第一块  
![](https://bbs.pediy.com/upload/attach/202005/715334_UGCCF5Z7635XJDD.jpg)

 

经过分析，实际上就是读`/proc/self/maps`的标准输出，从而获取到对应于`libnaitve-lib.so`的那一行，然后以`-`分割字符串，并将分割后的第一段解析为 16 进制的数，实际上就是获取`libnaitve-lib.so`的加载基地址。

 

![](https://bbs.pediy.com/upload/attach/202005/715334_JHFKB9XAZQJ97HX.png)

 

再看第二块，也就是`sub_8B90`函数的实现

```
int __fastcall find_symbol_value_and_size(int base_addr, char *a2, _DWORD *a3)
{
  int v3; // ST38_4
  _DWORD *ELF_Hash_Table; // ST28_4
  unsigned int v5; // ST20_4
  int elf_hash_chain; // [sp+14h] [bp-5Ch]
  int ELF_Symbol_Table; // [sp+24h] [bp-4Ch]
  int elf_hash_table; // [sp+28h] [bp-48h]
  int string_table; // [sp+2Ch] [bp-44h]
  int elf_symbol_table; // [sp+30h] [bp-40h]
  _DWORD *v12; // [sp+34h] [bp-3Ch]
  int dynamic_segment_base_addr; // [sp+40h] [bp-30h]
  _DWORD *header_table; // [sp+44h] [bp-2Ch]
  signed int i; // [sp+4Ch] [bp-24h]
  unsigned int j; // [sp+4Ch] [bp-24h]
  int elf_hash_bucket; // [sp+4Ch] [bp-24h]
  char v18; // [sp+57h] [bp-19h]
  char v19; // [sp+57h] [bp-19h]
  char v20; // [sp+57h] [bp-19h]
  _DWORD *value; // [sp+58h] [bp-18h]
  char *s2; // [sp+5Ch] [bp-14h]
  int so_base_addr; // [sp+60h] [bp-10h]
 
  so_base_addr = base_addr;
  s2 = a2;
  value = a3;
  v18 = -1;
  header_table = (base_addr + *(base_addr + 0x1C));// header_table_offset
  for ( i = 0; i < *(base_addr + 0x2C); ++i )   // *(base_addr + 0x2C) = 8
  {
    if ( *header_table == 2 )
    {
      v18 = 0;
      puts_0();                                 // find_dynamic_segment
      break;
    }
    header_table += 8;
  }
  if ( v18 )
    goto LABEL_27;
  dynamic_segment_base_addr = header_table[2] + so_base_addr;// 找到dynamic_segment的虚拟地址
  v19 = 0;
  for ( j = 0; j < header_table[4] >> 3; ++j )
  {
    v12 = (dynamic_segment_base_addr + 8 * j);
    if ( *(dynamic_segment_base_addr + 8 * j) == 6 )
    {
      elf_symbol_table = v12[1];                // 0x1f0
      ++v19;
    }
    if ( *v12 == 4 )
    {
      elf_hash_table = v12[1];                  // 0x46e0
      v19 += 2;
    }
    if ( *v12 == 5 )
    {
      string_table = v12[1];                    // 0x1d00
      v19 += 4;
    }
    if ( *v12 == 10 )
    {
      v3 = v12[1];                              // 0x1eb6
      v19 += 8;
    }
  }
  if ( (v19 & 0xF) != 0xF )
  {
    puts_0();
LABEL_27:
    return -1;
  }
  ELF_Hash_Table = (so_base_addr + elf_hash_table);// v4 =elf_hash_table
  v5 = turn_ooxx(s2);                            // v5 = 0x766f8
  ELF_Symbol_Table = so_base_addr + elf_symbol_table;// ELF Symbol Table
  elf_hash_chain = &ELF_Hash_Table[*ELF_Hash_Table + 2];
  v20 = -1;
  for ( elf_hash_bucket = ELF_Hash_Table[v5 % *ELF_Hash_Table + 2];// ELF_Hash_Table[v5 % *ELF_Hash_Table + 2] = 0x4918
        elf_hash_bucket;
        elf_hash_bucket = *(elf_hash_chain + 4 * elf_hash_bucket) )
  {
    if ( !strcmp((so_base_addr + string_table + *(ELF_Symbol_Table + 16 * elf_hash_bucket)), s2) )// string_table[] = "ooxx"
    {
      v20 = 0;
      break;
    }
  }
  if ( v20 )
    goto LABEL_27;
  *value = *(ELF_Symbol_Table + 16 * elf_hash_bucket + 4);
  value[1] = *(ELF_Symbol_Table + 16 * elf_hash_bucket + 8);
  return 0;
}

```

这个地方你仔细地去分析对比，会发现其实就是一个读 so 文件的对应于`symbol name`为`ooxx`的`symbol table`表项中的`value`和`size`, 其实就是读`ooxx`的函数起始地址以及函数大小。其实也就是一个`parser`的过程之一

 

对了，这个函数中的一行，也就是`v5 = turn_ooxx(s2);`这里调用的`turn_ooxx`函数中的伪代码直接`copy`出来跑一跑，就可以得到`v5`的值。我也没有分析这个过程，直接跑的。。

 

接着看`sub_8930`函数的第三块。

 

![](https://bbs.pediy.com/upload/attach/202005/715334_DFRAQJQUUT5R67A.jpg)

 

经过分析会发现，围绕`mprotect`函数将这个部分再次分成三块，分别实现功能为

1.  第一块，设置`ooxx`函数所在内存页为`rwx`
2.  第二块，还原`ooxx`函数中`code`
3.  第三块，恢复内存页为`r-x`

这里第二块中的`*i ^= byte_1C180[&i[-v5]];`这个部分，再加上`byte_1C180`实际上在`bss`段，不想再去分析了，直接动态吧。  
这里使用`objection`在动态运行时`dump`出对应内存中的数据，

 

![](https://bbs.pediy.com/upload/attach/202005/715334_8SWFGTXVZUNJ6QR.png)

 

使用`010 editor`查看对应文件

 

![](https://bbs.pediy.com/upload/attach/202005/715334_QTS29BQCYEAWV5P.jpg)

 

很明显那就是 0-255 的字节咯，继续看伪码，会发现实际上这里的`&i[-v5]`实际上就相当于`i-v5`, 而`v5`为`i`的初值，那么`patch`脚本就有了

```
def patchBytes(addr,length):
    for i in range(0,length):
        byte=get_bytes(addr + i,1)
        byte = ord(byte) ^ (i%0xff)
        patch_byte(addr+i,byte)
patchBytes(0x8e00,0x8fd0-0x8e00)

```

执行这个脚本之后, 查看`ooxx`函数内容

```
int __fastcall ooxx(JNIEnv *a1, int a2, int a3)
{
  JNIEnv *v3; // ST20_4
  int input; // r0
  int v5; // r0
  unsigned __int8 v7; // [sp+17h] [bp-19h]
 
  v3 = a1;
  sub_8930(); //
  v7 = 0;
  input = getStringUtf(v3);
  if ( input )
  {
    input = strcmp(aKanxuetest, input);
    if ( !input )
    {
      input = 1;
      v7 = 1;
    }
  }
  v5 = *(input + 8);
  sub_8930();
  return v7;
}

```

最终会发现，实际上`ooxx`就是拿我的输入和`kanxuetest`进行对比。。验证下

 

![](https://bbs.pediy.com/upload/attach/202005/715334_DR9SWV2NBYZGF4H.png)

 

拿到`flag`

后记
--

整个程序实际上真正难的地方在于看出`parser`的过程，不过我猜如果写过`parser`相信会很容易的看出来，还有  
另外，这个程序有点类似于之前寒冰师傅说的在函数执行开始之前对函数内容进行恢复，函数执行结束时再还原回加密状态，再加上插入了一堆`MOV R0, R0`这种无效代码，让我感觉真像`so`层的 "函数抽取壳" 的实现。。神奇的题目，最后，附上附件

[[培训] 优秀毕业生寄语：恭喜 id: 一颗金柚子获得阿里 offer《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-15958.htm)

最后于 2020-7-8 11:32 被 Simp1er 编辑 ，原因：

上传的附件：

*   [附件. zip](javascript:void(0)) （2.14MB，92 次下载）