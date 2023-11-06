> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [y9nhjy.github.io](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen)

[](#Declaration "Declaration")Declaration
=========================================

本项目仅用作学习教育目的, 不用于任何其他用途, 如有侵权请第一时间联系作者删除

This project is only for learning and educational purposes and is not intended for any other purpose. If there is any infringement, please contact the author immediately to delete it

[](#安装 "安装")安装
==============

软件官网 [Proxifier - The Most Advanced Proxy Client](https://www.proxifier.com/)

这里下载的是安装版 (setup)

![](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen/image-20230816131222485.png)

一路 next 下去安装完毕即可

[](#正向 "正向")正向
==============

[](#开始 "开始")开始
--------------

启动 Proxifier 点击 Registration Key

![](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen/image-20230816131258954.png)

随便输入会提示 Key 格式不对, 为`xxxxx-xxxxx-xxxxx-xxxxx-xxxxx`, 为方便描述, 本文将每 5 个字符称为 1 组, 下文也会提到。然后照着格式再次随便输入查看 Key 错误时的反馈

![](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen/image-20230808113527603.png)

接下来, IDA, 启动！搜索字符串`The registration Key is incorrect.`

![](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen/image-20230816162850059.png)

定位至此处

![](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen/image-20230808113322050.png)

通过交叉引用定位关键 Check 函数

![](https://y9nhjy.github.io/post/20230816-Proxifier-Keygen/image-20230816163111716.png)

[](#Check算法 "Check算法")Check 算法
------------------------------

```
char __fastcall Check(unsigned int *a1, _QWORD *a2, __int64 a3)
{
  _QWORD *v4; // rsi
  const wchar_t *v6; // rdx
  _QWORD *v8; // r8
  _QWORD *v9; // r8
  _QWORD *v10; // r8
  _QWORD *v11; // r8
  _QWORD *v12; // rax
  __int16 v13; // dx
  _QWORD *v14; // rax
  __int64 v15; // rax
  __int64 v16; // rcx
  int v17; // edi
  int v18; // ebx
  __int64 v19; // rax
  __int64 v20; // rcx
  __int64 v21; // rax
  __int64 v22; // rcx
  int v23; // er15
  __int64 v24; // rax
  unsigned int v25; // er15
  __int64 v26; // rcx
  int v27; // eax
  unsigned int v28; // er10
  unsigned int v29; // ecx
  char v30[32]; // [rsp+20h] [rbp-78h] BYREF
  __int64 v31; // [rsp+40h] [rbp-58h]
  _QWORD *v32; // [rsp+48h] [rbp-50h]
  int v33[4]; // [rsp+50h] [rbp-48h] BYREF

  v31 = -2i64;
  v4 = a2;
  v32 = a2;
  if ( a2[2] != 29i64 )
  {
    v6 = L"Incorrect Key length.";
LABEL_3:
    sub_140003550(a3, v6);
    unknown_libname_3(v4);
    return 0;
  }
  v8 = a2;
  if ( a2[3] >= 8ui64 )
    v8 = (_QWORD *)*a2;
  sub_1400034B0(a2, v33, (char *)v8 + 46);   // 函数作用:去掉'-'
  v9 = v4;
  if ( v4[3] >= 8ui64 )
    v9 = (_QWORD *)*v4;
  sub_1400034B0(v4, v33, (char *)v9 + 34);
  v10 = v4;
  if ( v4[3] >= 8ui64 )
    v10 = (_QWORD *)*v4;
  sub_1400034B0(v4, v33, (char *)v10 + 22);
  v11 = v4;
  if ( v4[3] >= 8ui64 )
    v11 = (_QWORD *)*v4;
  sub_1400034B0(v4, v33, (char *)v11 + 10);   // 最终得到'XXXXXXXXXXXXXXXXXXXXXXXXX'
  v12 = v4;
  if ( v4[3] >= 8ui64 )
    v12 = (_QWORD *)*v4;
  v13 = *((_WORD *)v12 + 14);                 // 取第3组第5个字符 '4'
  v14 = v4;
  if ( v4[3] >= 8ui64 )
    v14 = (_QWORD *)*v4;
  *((_WORD *)v14 + 2) = v13;                  // 第1组第3个字符赋值为第3组第5个字符, 即此字符不影响Key
  v15 = sub_1400037E0(v4, v30, 20i64);        // 取第5组字符
  v17 = sub_1400031A0(v16, v15);              // 第一个参数:v15一个字符的ascii值
                                              // 此函数的作用:将字符串转换为数字, 本函数可逆
  v18 = v17 ^ (v17 << 7);
  v19 = sub_1400037E0(v4, v30, 15i64);        // 取第4组字符
  a1[7] = sub_1400031A0(v20, v19);
  v21 = sub_1400037E0(v4, v30, 0i64);         // 取整个Key的前7个字符
  v23 = sub_1400031A0(v22, v21);
  v24 = sub_1400037E0(v4, v30, 7i64);         // 取整个Key的第8-14个字符
  v25 = v18 ^ v23 ^ 0x12345678;               // v33的低4字节
  v27 = sub_1400031A0(v26, v24);
  v33[0] = v25;
  v33[1] = v18 ^ v27 ^ 0x87654321;            // v33的中4字节
  v33[2] = a1[7];                             // v33的高4字节
  if ( v17 != (sub_140003260(v33) & 0x1FFFFFF) )  // 此函数类似CRC32, 目测不可逆
  {
    v6 = L"Incorrect Key";
    goto LABEL_3;
  }
  *a1 = v25 >> 21;
  a1[1] = HIWORD(v25) & 0x1F;
  a1[2] = (unsigned __int16)v25 >> 5;
  a1[3] = v25 & 0x1F;
  v29 = HIWORD(v28);
  a1[6] = (unsigned __int16)v28;
  if ( HIWORD(v28) )
  {
    a1[4] = v29 / 0xC + 2000;
    a1[5] = v29 % 0xC;
  }
  else
  {
    *((_QWORD *)a1 + 2) = 0i64;
  }
  unknown_libname_3(v4);
  return 1;
}

```

Copy

前面会检查 Key 的长度是否为 29, 即`XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`的长度

然后主要需要分析`sub_1400034B0, sub_1400037E0, sub_1400031A0, sub_140003260`这 4 个函数的功能

*   本地调试直接步过看 Key 的变化, 很容易就能看出`sub_1400034B0`的功能, 即去掉 ‘-‘
    
*   `sub_1400037E0`, 同样的办法可以看出此函数是取出 Key 的部分字符
    
*   然后就是`sub_1400031A0`, 进去看看
    
    ```
    __int64 __fastcall sub_1400031A0(__int64 a1, __int64 a2)
    {
      unsigned int v2; // ebx
      int v3; // eax
      __int64 i; // r8
      __int64 v5; // rax
      unsigned __int16 v6; // ax
    
      v2 = 0;
      v3 = *(_DWORD *)(a2 + 16) - 1;
      for ( i = v3; i >= 0; --i )
      {
        v2 *= 32;
        v5 = a2;
        if ( *(_QWORD *)(a2 + 24) >= 8ui64 )
          v5 = *(_QWORD *)a2;
        v6 = *(_WORD *)(v5 + 2 * i);
        switch ( v6 )
        {
          case 'W':
            continue;
          case 'X':
            v6 = 79;
            break;
          case 'Y':
            v2 = v2 - 48 + 49;
            continue;
          case 'Z':
            v6 = 73;
            break;
          default:
            if ( (unsigned __int16)(v6 - 48) <= 9u )
            {
              v2 = v6 + v2 - 48;
              continue;
            }
            break;
        }
        v2 = v6 + v2 - 55;
      }
      unknown_libname_3(a2);
      return v2;
    }
    
    ```
    
    Copy
    
    可以自己跟着写一遍此算法, 主要作用：将字符串根据此算法转换为数字并返回, 此函数的 python 实现可以参考本人 [y9nhjy/Proxifier_Keygen](https://github.com/y9nhjy/Proxifier_Keygen) 项目中 Proxifier_Checker.py 里的 handle 函数
    
*   `sub_140003260`：算法类似 CRC32, 目测不可逆
    

最后总结 Check 完整逻辑：

*   去除 ‘-‘
*   将第 1 组第 3 个字符赋值为第 3 组第 5 个字符
*   v17：第 5 组字符经过 handle 函数所生成的数字
*   v18 = v17 ^ (v17 << 7)
*   v23：取整个 Key 的前 7 个字符经过 handle 函数所生成的数字
*   v27：取整个 Key 的第 8-14 个字符经过 handle 函数所生成的数字
*   v33 的低 4 字节：v18 ^ v23 ^ 0x12345678
*   v33 的中 4 字节：v18 ^ v27 ^ 0x87654321
*   v33 的高 4 字节：第 4 组字符经过 handle 函数所生成的数字
*   最后比较 v17 != (sub_140003260(v33) & 0x1FFFFFF)

[](#逆向 "逆向")逆向
==============

项目源代码：[y9nhjy/Proxifier_Keygen](https://github.com/y9nhjy/Proxifier_Keygen) , 欢迎 **star**

既然`sub_140003260`不可逆, 就考虑构造 v33

目前已知 v33, 逆推 Key 的流程：

1.  根据高 4 字节逆推第 4 组字符
2.  根据 v17 = (sub_140003260(v33) & 0x1FFFFFF) 得到 v17 后可逆推第 5 组字符和得到 v18
3.  v23 = v33 的低 4 字节 ^ v18 ^ 0x12345678, 然后逆推出 Key 的前 7 个字符
4.  v27 = v33 的中 4 字节 ^ v18 ^ 0x87654321, 然后逆推出 Key 的第 8-14 个字符
5.  Key 的第 15 个字符即 Key 的第 3 个字符
6.  随机生成 Key 的第 3 个字符
7.  得到最终的 Key

注意事项：

1.  v33 的 1~2 字节小于 0x2580 是过期的 Key, 高两字节是产品版本, 为 0 则是安装版, 1 是便携版, 2 是 Mac 版
2.  v33 的 7~8 字节是证书有效期, 为零直接无限期
3.  Key 的第 3 位不影响 Key, 但是不能为 ‘Y’, 否则版本对不上

[](#碎碎念 "碎碎念")碎碎念
=================

断断续续调了几天, 将 Check 算法还原成功后, 参考了 [Danz17/Proxifier-Keygen](https://github.com/Danz17/Proxifier-Keygen) 注册机的源代码, 学到了不少, 对其中的流程也更加清晰, 最终总算是实现了注册机的编写, 欢迎各位师傅给我的项目 **star**, 传送门：[y9nhjy/Proxifier_Keygen](https://github.com/y9nhjy/Proxifier_Keygen)

时隔满满一年终于更新了, 之前电脑出问题系统重装数据全没了, 包括博客, 血压拉满, 就一直没再折腾博客, 这次重新搭建博客, 更换了一个更酷炫的主题, 整体还是比较满意, 就这样吧。