> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-271801.htm)

> [原创] 对 PCHunterV 1.57 授权的逆向分析

前言
==

众所周知，PCHunter 是一款前沿的 AntiRootkit 软件，是我们查看系统信息的好伙伴。然而在不久前，我发现之前申请试用的授权已经到期了，我发现这一点也不 cool。这才有了这篇文章.  
![](https://bbs.kanxue.com/upload/attach/202203/768770_KGV5WTXUBNTMZG6.png)  
![](https://bbs.kanxue.com/upload/attach/202203/768770_YT6MKQ8FRPGXMHU.png)

初探
==

###### 程序名: pchunter64.exe

###### 版本: V1.57 64 位程序 固定基址: 0x000140000000

###### MD5:5E8E043EE6AD5D62DCDA7445367672C7

###### SHA1:EB0F8979FBFB0EB803ABC5BECD3B0B3286FE131D

###### Exeinfo:Microsoft Visual C++ v12 - 2013 - (x64) www.microsoft.com (exe 4883ec28/48) - no sec. CAB/7z [ *Win Vista ] , Overlay : 384400... Nothing discovered

 

软件 64 为程序，能够正常被 x64dbg 加载或附加，无反调试功能

分析
==

软件采用的是 license 文件注册的方式，license 的文件名是 pchunter.ek, 尝试用 notepad++ 打开，发现一串长度为 256 的注册码

```
B63E065F31A65170C7DD99538A7AB937CC9742B392A81E49327C26E74FB6F10FCCFF22E0278F11685E7B04010B3BAB00CCFF22E0278F11685E7B04010B3BAB00400AAB14BCF37048C028C5F498E990F44DF556BC4F347975F048AFCBBB87942A1722F48B7784D578693ED0C055B3EC878A096DEADC93133D26CE32DBAC9B9487

```

可以合理猜测注册方式是读 license 文件里的注册码，然后进行一系列的解密运算得到授权时间等相关信息，以供程序进行授权检测。那我们的方向就很简单了，对程序下 CreateFile 断点，使程序断在参数是 license 路径的地方，通过栈回溯，发现在 0x000000014007282C 处调用，之后接着调用 GetFileSize、ReadFile 等 API。很明显，我们找到关键位置了。  
![](https://bbs.kanxue.com/upload/attach/202203/768770_8F36U7ZQE3M8CGN.png)  
那我不装了，直接上 IDA 吧！  
通过动静结合的分析，我锁定了验证 license 的函数在 **sub_14007C570** 里。  
![](https://bbs.kanxue.com/upload/attach/202203/768770_EK2DXSTWNCXE3Q7.png)  
其中 **sub_140072720** 从授权文件读取 0x200 个字节并做了相关的运算, v21 从 0x200 变成 0x100，但是 **sub_1400435F0** 进行解密操作的时候 buffer 还是原来的，并没有截断的操作。因此我没有具体分析 **sub_140072720** 函数，有兴趣的同学可以试着分析一下，看看里面做了什么运算。我重点分析的是 **sub_1400435F0** 函数。分析情况如下：

```
__int64 __fastcall sub_1400435F0(__int64 pInputbuffer, int a2, FILETIME *a3, _DWORD *a4, FILETIME *a5, _DWORD *a6)
{
  __int64 nKeyLen; // rbx
  char *v10; // rax
  __int64 v11; // rcx
  int nIndex; // er10
  char *pOut; // r8
  unsigned int i; // er9
  BYTE v15; // dl
  unsigned __int8 v16; // cl
  __int64 result; // rax
  int pOutBuffer[124]; // [rsp+20h] [rbp-358h] BYREF
  DWORD aKey[8]; // [rsp+210h] [rbp-168h] BYREF
  char v20[80]; // [rsp+230h] [rbp-148h] BYREF
  DWORD v21; // [rsp+280h] [rbp-F8h]
  int v22; // [rsp+284h] [rbp-F4h]
  DWORD v23; // [rsp+288h] [rbp-F0h]
  int v24; // [rsp+28Ch] [rbp-ECh]
 
  nKeyLen = -1i64;
  if ( a2 != 0x100 || !pInputbuffer ) //对输入的参数进行验证
    return 0xFFFFFFFFi64;
 
  //接下来这一段 Memset 0 一块0x80的空间
  v10 = v20;
  v11 = 2i64;
  do
  {
    *(_QWORD *)v10 = 0i64;
    *((_QWORD *)v10 + 1) = 0i64;
    *((_QWORD *)v10 + 2) = 0i64;
    v10 += 0x40;
    *((_QWORD *)v10 - 5) = 0i64;
    *((_QWORD *)v10 - 4) = 0i64;
    *((_QWORD *)v10 - 3) = 0i64;
    *((_QWORD *)v10 - 2) = 0i64;
    *((_QWORD *)v10 - 1) = 0i64;
    --v11;
  }
  while ( v11 );
 
// 1)第一次解密
  //对读进来的0x200大小的buffer进行第一次解密运算 得到0x80大小的buffer1，并且放到上面的初始化的空间中
 
  *(_DWORD *)v10 = 0;
  nIndex = 0;
  pOut = v20;
  for ( i = 1; i < 0x101; i += 2 )
  {
    v15 = *(_BYTE *)((unsigned int)(2 * nIndex) + pInputbuffer);
    if ( (unsigned __int8)(v15 - '0') > 9u )
    {
      if ( (unsigned __int8)(v15 - 'A') <= 5u )
        v15 -= 0x37;
    }
    else
    {
      v15 -= '0';
    }
    if ( v15 > 0xFu )
      break;
    v16 = *(_BYTE *)(i + pInputbuffer);
    *pOut = 16 * v15;
    if ( (unsigned __int8)(v16 - '0') > 9u )
    {
      if ( (unsigned __int8)(v16 - 'A') <= 5u )
        v16 -= 55;
    }
    else
    {
      v16 -= '0';
    }
    if ( v16 > 0xFu )
      break;
    *pOut |= v16;
    ++nIndex;
    ++pOut;
  }
  if ( nIndex != 128 )
    return 0xFFFFFFFFi64;
 
// 2)AES解密
  //初始化AES解密的KEY
  strcpy((char *)aKey, "ShouJiErShiSiShi");
  BYTE1(aKey[4]) = 0;
  HIWORD(aKey[4]) = 0;
  sub_140043040();
 
  //计算Key的长度
  do
    ++nKeyLen;  
  while ( *((_BYTE *)aKey + nKeyLen) );
 
  //AES ECB 模式解密
  sub_140042B70((DWORD *)pOutBuffer, aKey, nKeyLen);
  sub_1400411E0((unsigned int *)pOutBuffer, (__int64)v20, (__int64)v20, 0x80u);
  result = 0i64;
 
  //获取授权时间
  a3->dwLowDateTime = v21;
  *a4 = v22;      
  a5->dwLowDateTime = v23;
  *a6 = v24;
  return result;
}

```

上面就是解密的相关流程，至于你问我为什么知道第二部分是 AES 解密，我只能说我在 **sub_140042B70** 里面发现了下面这一段特征  
据说是一种 AES 128 非标准实现的特征。

```
if ( (unsigned int)(nKeyLen + 24) > 4 )
  {
    v31 = pOutBuffer + 5;
    v32 = (unsigned int)(nKeyLen + 20);
    do
    {
      v33 = (2 * (*v31 & 0xFF7F7F7F)) ^ (27 * ((*v31 >> 7) & 0x1010101));
      v34 = (2 * (v33 & 0xFF7F7F7F)) ^ (27 * ((v33 >> 7) & 0x1010101));
      v35 = (2 * (v34 & 0xFF7F7F7F)) ^ (27 * ((v34 >> 7) & 0x1010101));
      v36 = v35 ^ *v31;
      v37 = sub_140043340(v36 ^ v34, 16);
      v38 = sub_140043340(v36 ^ v33, 8) ^ v37;
      ++v31;
      v31[59] = v33 ^ v34 ^ v35 ^ sub_140043340(v36, 24) ^ v38;
      --v32;
    }
    while ( v32 );
  }

```

并且分析的过程中也没有发现 iv 的相关参数。那抱着试一试的心态，我就尝试用 AES_ECB 模式进行解密，还真让我解出来了。  
程序 buffer 解密前后对比如下:  
解密前：  
![](https://bbs.kanxue.com/upload/attach/202203/768770_GX5T26PK6E2NC8W.png)  
解密后：  
![](https://bbs.kanxue.com/upload/attach/202203/768770_J2XKSVYY4SYRRP4.png)  
通过分析，buffer[0x50] 开始的 16 个字节的数据就是授权时间的信息。这个是 UTC 格式的时间，可以从验证授权时间的逻辑里面调用的 API 得到这个结论。

KeyGen
======

讲讲思路吧。那就是对原授权解密的 buffer 中的 buffer[50] 做修改大小 0x10 的数据，然后用原密钥 "ShouJiErShiSiShi" 以 AES ECB 模式进行加密。最后逆向第一轮解密的算法，生成符合要求的序列号就可以了。  
这里就不放代码了，有兴趣的就自己分析写注册机，没有兴趣的就用我生成的授权文件吧。 ![](https://bbs.kanxue.com/upload/attach/202203/768770_Q9USKGFG236XZ9B.png)  
PS:AES 加密可以用 openssl，之前面试的时候面试官问我有了解或者用过这个库吗，我说没有，然而现在我可以说我用过了 Zzz。

参考
==

没有参考，全靠自己分析。

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

[#调试逆向](forum-4-1-1.htm)

上传的附件：

*   [pchunter.ek](javascript:void(0)) （0.25kb，611 次下载）