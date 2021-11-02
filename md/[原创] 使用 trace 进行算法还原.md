> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270078.htm)

> [原创] 使用 trace 进行算法还原

[](#一、题目)一、题目
-------------

### 1、android 逆向题

小伙伴发来的题，说是某比赛线下题目

### [](#2、涉及算法)2、涉及算法

#### AES

#### 魔改 RC5

[](#二、解题过程)二、解题过程
-----------------

### [](#1、java层)1、java 层

![](https://bbs.pediy.com/upload/attach/202111/590753_8HG84DMFFRBZGBE.png)

### [](#2、native层)2、native 层

![](https://bbs.pediy.com/upload/attach/202111/590753_CQT3GUJ5V4V5R7H.png)  
sub_F08

```
int __fastcall sub_F08(JNIEnv *a1, int a2, int a3)
{
  const char *v3; // r9
  size_t v4; // r6
  char *v5; // r4
  void *v6; // r8
  _DWORD *v7; // r5
  int v8; // r9
  unsigned __int8 *v9; // r6
  int v10; // r0
  char *v11; // r1
  int v12; // r2
  int v13; // t1
  int v14; // t1
 
  v3 = (*a1)->GetStringUTFChars(a1, a3, 0);
  v4 = strlen(v3);
  v5 = malloc(v4);
  v6 = malloc(4 * v4 + 64);
  v7 = malloc(0x20u);
  *v7 = 0x1010101;
  v7[1] = 0x1010101;
  v7[2] = 0x1010101;
  v7[3] = 0x1010101;
  qmemcpy(v5, v3, v4);
  v8 = sub_D68(v6, v5, v4, v7);
  free(v5);
  memset(v7, 2, 0x20u);
  v9 = malloc(4 * v8 + 32);
  v10 = sub_E80(v9, v6, v8, v7);
  v11 = &c_cipher;
  while ( v10 )
  {
    v13 = *v11++;
    v12 = v13;
    --v10;
    v14 = *v9++;
    if ( v14 != v12 )
      return 0;
  }
  return 1;
}

```

#### [](#（1）对伪码进行分析，流程如下)（1）对伪码进行分析，流程如下

```
第一加密 v8 = sub_D68(v6, v5, v4, v7);
第二次加密 v10 = sub_E80(v9, v6, v8, v7);
解密结果与 &c_cipher进行比较

```

#### [](#（2）动态调试)（2）动态调试

##### c_cipher

```
unsigned char c_cipher[56] = {
0xCA, 0x60, 0x55, 0x30, 0xB5, 0xDB, 0xD4, 0xA6, 0x01, 0x15, 0x3F, 0xB8, 0xBC, 0x4C, 0x9C, 0x88,
0xEA, 0xF4, 0x76, 0xDD, 0x8D, 0x7B, 0x1A, 0x26, 0xDA, 0x74, 0x2C, 0x1D, 0x28, 0x63, 0x4B, 0x88,
0x44, 0x22, 0x7E, 0x21, 0x0E, 0x6C, 0xF4, 0xAE, 0xE4, 0x21, 0xC7, 0x67, 0x21, 0x40, 0xC5, 0x3B,
0xB2, 0x55, 0x92, 0x21, 0x9B, 0x29, 0xFA, 0x33
 };

```

##### 第一加密 v8 = sub_D68(v6, v5, v4, v7);

![](https://bbs.pediy.com/upload/attach/202111/590753_EJ37KJCJS8GKZ2U.png)  
AES 的 S 盒

```
输入 flag{0123456789}
key 01010101010101010101010101010101
加密结果
7E D8 C7 01 11 B4 88 27  0E 42 0B 31 59 CB 42 63
68 43 4D 37 D7 F6 9F 0E  03 B7 5B B1 5B C9 4B 6C

```

验证一下  
![](https://bbs.pediy.com/upload/attach/202111/590753_W5SVNF9JBH945KV.png)

##### 第二次加密 v10 = sub_E80(v9, v6, v8, v7);

```
输入
7E D8 C7 01 11 B4 88 27  0E 42 0B 31 59 CB 42 63
68 43 4D 37 D7 F6 9F 0E  03 B7 5B B1 5B C9 4B 6C
key
02020202020202020202020202020202
02020202020202020202020202020202
加密结果
8B A8 1D 1F DF E9 09 98  78 B0 0A CA F7 8F DE 1D
56 6F B4 65 E0 32 DC BA  BE 7B 8A 81 A1 41 59 BA
B2 55 92 21 9B 29 FA 33

```

###### sub_E80

```
unsigned int __fastcall sub_E80(int a1, int a2, int a3, int a4)
{
  unsigned int result; // r0
  int i; // r10
  int v9; // r9
  int v10; // r1
  int v11; // r4
  int j; // r5
  __int64 v13; // kr00_8
 
  sub_DA8(a4);
  result = sub_994(a1, a2, a3, 8);
  for ( i = 0; i != result >> 3; ++i )
  {
    v9 = 2 * i + 1;
    v10 = *(a1 + 8 * i) + dword_5E50[0];
    v11 = *(a1 + 4 * v9) + unk_5E54;
    for ( j = 0; j != 12; ++j )
    {
      v13 = *&dword_5E50[2 * j + 2];
      v10 = __ROR4__(v10 ^ v11, 32 - v11) + v13;
      v11 = __ROR4__(v10 ^ v11, 32 - v10) + HIDWORD(v13);
    }
    *(a1 + 8 * i) = v10;
    *(a1 + 4 * v9) = v11;
  }
  return result;
}

```

我使用 trace 进行的算法还原  
![](https://bbs.pediy.com/upload/attach/202111/590753_G3G9E5P4C5NZ9B7.png)

###### [](#对trace进行分析，算法如下)对 trace 进行分析，算法如下

```
输入
7E D8 C7 01 11 B4 88 27  0E 42 0B 31 59 CB 42 63
68 43 4D 37 D7 F6 9F 0E  03 B7 5B B1 5B C9 4B 6C
 
加密结果
8B A8 1D 1F DF E9 09 98  78 B0 0A CA F7 8F DE 1D
56 6F B4 65 E0 32 DC BA  BE 7B 8A 81 A1 41 59 BA
B2 55 92 21 9B 29 FA 33

```

输入由 32 补全 40 位，08 补全  
![](https://bbs.pediy.com/upload/attach/202111/590753_3DBP64YCNU9HWK5.png)

```
输入
7E D8 C7 01 11 B4 88 27  0E 42 0B 31 59 CB 42 63
68 43 4D 37 D7 F6 9F 0E  03 B7 5B B1 5B C9 4B 6C
08 08 08 08 08 08 08 08

```

输入长度 40 >> 3 = 5  
![](https://bbs.pediy.com/upload/attach/202111/590753_SWNB67YW8YWY326.png)

 

进行了 5 次大循环，每次循环进行了 12+1 次  
![](https://bbs.pediy.com/upload/attach/202111/590753_R3ACN7F9R4HHND5.png)  
每次小循环使用的数组：

```
uint32_t dword_C30DDE50[] = {
        0x751EF375, 0x9A777FE8,
        0xCF327F57, 0x378B39DC,
        0xB0CF04C8, 0xA0AA7A08,
        0xCE01CEA5, 0xF400183,
        0x58FBD0BB, 0x8D44D42A,
        0x59F0BB54, 0xB632AA3A,
        0x40030399, 0xA9C58CAC,
        0xB1F1F560, 0x29796A99,
        0x513FE4A6, 0x8204DA32,
        0xC435BFCA, 0x465706D,
        0xECB351E9, 0x9A83C02C,
        0x4615414F, 0x8728979D,
        0x8CED0CFA, 0x15647C9C};

```

逆推  
![](https://bbs.pediy.com/upload/attach/202111/590753_WNYV3YDCPJ4NZEQ.png)  
还原算法

```
uint32_t dword_C30DDE50[] = {
        0x751EF375, 0x9A777FE8,
        0xCF327F57, 0x378B39DC,
        0xB0CF04C8, 0xA0AA7A08,
        0xCE01CEA5, 0xF400183,
        0x58FBD0BB, 0x8D44D42A,
        0x59F0BB54, 0xB632AA3A,
        0x40030399, 0xA9C58CAC,
        0xB1F1F560, 0x29796A99,
        0x513FE4A6, 0x8204DA32,
        0xC435BFCA, 0x465706D,
        0xECB351E9, 0x9A83C02C,
        0x4615414F, 0x8728979D,
        0x8CED0CFA, 0x15647C9C};
 
uint32_t plaintext[] = {0x01C7D87E, 0x2788B411, 0x310B420E, 0x6342CB59, 0x374D4368, 0x0E9FF6D7, 0xB15BB703, 0x6C4BC95B, 0x08080808, 0x08080808};
 
int len = sizeof(plaintext) / sizeof(uint32_t);
 
for(int i = 0; i < len/2; i++){
    int R1 = plaintext[i*2];
    int R4 = plaintext[i*2+1];
    int R2,R5,R6,R9;
    int R3=0xC30DDE50;
    for(int j = 0; j < 13; j++){
 
        if(j == 0){
            R1 = R1 + dword_C30DDE50[j*2];
 
            R4 = R4 + dword_C30DDE50[j*2+1];
        }else{
            R1 = R1 ^ R4;
 
            R6 = 0x20 - R4;
 
            R1 = ror(R1, R6);
 
            R2 = dword_C30DDE50[j*2];
 
            R6 = dword_C30DDE50[j*2+1];
 
            R1 = R1 + R2;
 
            R2 = R1 ^ R4;
 
            R4 = 0x20 - R1;
 
            R2 = ror(R2, R4);
 
            R4 = R2 + R6;
 
        }
    }
 
    printf("0x%x,0x%x, \n", R1, R4);
 
}

```

![](https://bbs.pediy.com/upload/attach/202111/590753_XEFKGYJG2FBG58U.png)  
对加密算法进行逆运算

```
uint32_t dword_C30DDE50[] = {
        0x751EF375, 0x9A777FE8,
        0xCF327F57, 0x378B39DC,
        0xB0CF04C8, 0xA0AA7A08,
        0xCE01CEA5, 0xF400183,
        0x58FBD0BB, 0x8D44D42A,
        0x59F0BB54, 0xB632AA3A,
        0x40030399, 0xA9C58CAC,
        0xB1F1F560, 0x29796A99,
        0x513FE4A6, 0x8204DA32,
        0xC435BFCA, 0x465706D,
        0xECB351E9, 0x9A83C02C,
        0x4615414F, 0x8728979D,
        0x8CED0CFA, 0x15647C9C};
 
uint32_t cipher[] = {0x305560CA, 0xA6D4DBB5, 0xB83F1501, 0x889C4CBC, 0xDD76F4EA, 0x261A7B8D,
                     0x1D2C74DA, 0x884B6328, 0x217E2244, 0xAEF46C0E, 0x67C721E4, 0x3BC54021,
                     0x219255B2, 0x33FA299B};
 
// 解密
 
int len2 = sizeof(cipher) / sizeof(uint32_t);
for(int i = 0; i< len2/2; i++){
    int R1_cipher = cipher[i*2];
    int R4_cipher = cipher[i*2+1];
    int R1, R4;
    for(int j = 12; j >=0; j--){
        if(j == 0){
            R1_cipher = R1_cipher - dword_C30DDE50[j*2];
 
            R4_cipher = R4_cipher - dword_C30DDE50[j*2+1];
        }else{
 
            R4_cipher = rol(R4_cipher - dword_C30DDE50[j*2+1], 0x20 - R1_cipher) ^ R1_cipher;
 
            R1_cipher = rol(R1_cipher - dword_C30DDE50[j*2], 0x20 - R4_cipher) ^ R4_cipher;
 
        }
    }
 
    printf("%x %x \n", R1_cipher, R4_cipher);
}

```

![](https://bbs.pediy.com/upload/attach/202111/590753_G96M8C2J7ESWWUX.png)

 

使用 AES Decrypt  
![](https://bbs.pediy.com/upload/attach/202111/590753_BZB3A5WU54MQ4CQ.png)

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm)

上传的附件：

*   [cry.apk](javascript:void(0)) （1.56MB，4 次下载）
*   [trace.txt](javascript:void(0)) （66.73kb，3 次下载）