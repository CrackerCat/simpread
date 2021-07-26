> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268527.htm)

> [原创] 使用 bindiff 恢复 lua 符号 (调用 lua 的程序逆向)

使用 bindiff 恢复 lua 符号 (调用 lua 的程序逆向)
===================================

Lua 是一种轻量小巧的脚本语言，用标准 C 语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能

 

对于逆向一个会调用 lua 的程序，程序中会存在很多调用 lua 的标准函数，例如：luaL_newstate，lua_getglobal，luaL_openlibs，lua_tonumber 等

[](#一：c和lua相互调用的分析)一：C 和 lua 相互调用的分析
====================================

lua 官方文档：[http://www.lua.org/manual/5.4/](http://www.lua.org/manual/5.4/)

一：add.c 调用 add.lua
------------------

```
//add.c
//你需要include这几个lua头文件
#include  #include  "lua.h"
#include  "lualib.h"
#include  "lauxlib.h"
lua_State* L;
int
luaadd(int x, int y)
{
 int sum;
 /*函数名*/
 lua_getglobal(L,"add");
 /*参数入栈*/
 lua_pushnumber(L, x);
 /*参数入栈*/
 lua_pushnumber(L, y);
 /*开始调用函数，有2个参数，1个返回值*/
 lua_call(L, 2, 1);
 /*取出返回值*/
 sum = (int)lua_tonumber(L, -1);
 /*清除返回值的栈*/
 lua_pop(L,1);
 return sum;
}
int
main(int argc, char *argv[])
{
 int sum;
 L = luaL_newstate(); /* 创建lua状态机 */
 luaL_openlibs(L); /* 打开Lua状态机中所有Lua标准库 */
 /*加载lua脚本*/
 luaL_dofile(L, "add.lua");
 /*调用C函数，这个里面会调用lua函数*/
 sum = luaadd(99, 10);
 printf("The sum is %d \n",sum);
 /*清除Lua*/
 lua_close(L);
 return 0;
} 
```

```
//add.lua
function add(x,y)
  return x + y
end

```

add.lua 放在 add.c 的同路径下

[](#二：lua调c)二：lua 调 C
---------------------

```
1.通过在C中注册函数给lua调用(常)
2.封装成c动态链接库，在lua中require

```

### 1. 通过在 C 中注册函数给 lua 调用 (常)

lua 提供了 lua_register 函数注册 C 函数给 lua 端调用

```
//hello.c
#include  #include  #include  "lua.h"
#include  "lualib.h"
#include  "lauxlib.h"
static int l_SayHello(lua_State *L)
{
 const char *d = luaL_checkstring(L, 1);//获取参数，字符串类型
 int len = strlen(d);
 char str[100] = "hello ";
 strcat(str, d);
 lua_pushstring(L, str); /* 返回给lua的值压栈 */
 return 1;
}
int
main(int argc, char *argv[])
{
 lua_State *L = luaL_newstate(); /* 创建lua状态机 */
 luaL_openlibs(L); /* 打开Lua状态机中所有Lua标准库 */
 lua_register(L, "SayHello", l_SayHello);//注册C函数到lua
 const char* testfunc = "print(SayHello('lijia'))";//lua中调用c函数
 if(luaL_dostring(L, testfunc)) // 执行Lua命令。
  printf("Failed to invoke.\n");
 
 /*清除Lua*/
 lua_close(L);
 return 0;
} 
```

gcc -o hello hello.c -llua 编译执行

 

./hello

### **2. 调用 C 动态链接库**

创建一个 mylib.c 的文件，然后我们把它编译成动态链接库

```
#include #include #include #include #include #include #include /* 所有注册给Lua的C函数具有
 * "typedef int (*lua_CFunction) (lua_State *L);"的原型。
 */
static int l_sin(lua_State *L)
{
 // 如果给定虚拟栈中索引处的元素可以转换为数字，则返回转换后的数字，否则报错。
 double d = luaL_checknumber(L, 1);
 lua_pushnumber(L, sin(d)); /* push result */
 
 /* 这里可以看出，C可以返回给Lua多个结果，
  * 通过多次调用lua_push*()，之后return返回结果的数量。
  */
 return 1; /* number of results */
}
/* 需要一个"luaL_Reg"类型的结构体，其中每一个元素对应一个提供给Lua的函数。
 * 每一个元素中包含此函数在Lua中的名字，以及该函数在C库中的函数指针。
 * 最后一个元素为“哨兵元素”（两个"NULL"），用于告诉Lua没有其他的函数需要注册。
 */
static const struct luaL_Reg mylib[] = {
 {"mysin", l_sin},
 {NULL, NULL}
};
/* 此函数为C库中的“特殊函数”。
 * 通过调用它注册所有C库中的函数，并将它们存储在适当的位置。
 * 此函数的命名规则应遵循：
 * 1、使用"luaopen_"作为前缀。
 * 2、前缀之后的名字将作为"require"的参数。
 */
extern int luaopen_mylib(lua_State* L)
{
 /* void luaL_newlib (lua_State *L, const luaL_Reg l[]);
  * 创建一个新的"table"，并将"l"中所列出的函数注册为"table"的域。
  */
 luaL_newlib(L, mylib);
 
 return 1;
} 
```

使用 gcc -o [mylib.so](http://mylib.so/) -fPIC -shared mylib.c -llua -ldl 编译成 so

 

然后创建一个 lua 文件，把我们编译出来的 c 库引入进来

```
--[[ 这里"require"的参数对应C库中"luaopen_mylib()"中的"mylib"。
  C库就放在"a.lua"的同级目录，"require"可以找到。]]
local mylib = require "mylib"
-- 结果与上面的例子中相同，但是这里是通过调用C库中的函数实现。
print(mylib.mysin(3.14 / 2)) --> 0.99999968293183

```

执行 a.lua 文件，后报错，说 Lua 存在多个虚拟机

 

因为 lua 默认编译的是静态链接库，这样会导致链接多个 VM 冲突。

 

那么我们自己再编译个 lua 解释器动态链接一下。

```
//mylua.c
#include #include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
int main() {
 lua_State *L = luaL_newstate();
 luaL_openlibs(L);
if (luaL_loadfile(L, "a.lua") || lua_pcall(L, 0, 0, 0)) {
  printf("%s", lua_tostring(L, -1));
 }
} 
```

gcc -o mylua mylua.c -llua -ldl -lm -Wall

 

这样就能编译出 mylua 可执行文件

 

在命令行./mylua 执行，成功打印出 0.99999968293183

[](#二：编译lua)二：编译 lua
====================

首先我们获得 lua 的源码包

```
curl -R -O [http://www.lua.org/ftp/lua-5.3.4.tar.gz](http://www.lua.org/ftp/lua-5.3.0.tar.gz)

```

然后解压

```
tar zxf lua-5.3.4.tar.gz

```

cd lua-5.3.4

 

sudo make linux test

 

（执行这条命令时若报错 fatal error: readline/readline.h: No such file or directory，是缺少 readline 相关的库，执行 sudo apt-get install libreadline-dev 装好即可）

 

sudo make install

 

全部成功后

```
ub20@ubuntu:~$ cd /usr/local/bin
ub20@ubuntu:/usr/local/bin$ ls
arm_now  freeze_graph  pip2    saved_model_cli
distro   lua           pip2.7  tensorboard
f2py     luac          pip3    toco
f2py2    markdown_py   pip3.5  toco_from_protos
f2py2.7  pip           render  wheel
ub20@ubuntu:/usr/local/bin$ file lua
lua: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
BuildID[sha1]=a401cd154b2c1efae00a3390e0f287bacf1187b8, for
GNU/Linux 3.2.0, not stripped

```

将这个 lua 文件复制到主机即可

[](#三：bindiff恢复lua符号)三：bindiff 恢复 lua 符号
========================================

先使用 IDA 载入我们上面编译好的 lua

 

可以看到函数名很完整

 

![](https://bbs.pediy.com/upload/attach/202107/921830_A2U3AQCH64YUQP9.png)

 

退出保存得到 lua.i64 文件

 

然后将我们要分析的程序重新拖入 IDA，ctrl+6 使用 bindiff

 

![](https://bbs.pediy.com/upload/attach/202107/921830_KVM639RRMSB4MRH.png)  
Diff Database（记住两个 i64 文件的路径都不能有中文）

 

载入成功后再次 ctrl+6  
![](https://bbs.pediy.com/upload/attach/202107/921830_DXZRNV2KS2VGF7M.png)

 

结果

 

![](https://bbs.pediy.com/upload/attach/202107/921830_J735ZZ9HYYMSDNT.png)

四：(实践)2021 津门杯 easyRe
=====================

调用 lua 的程序逆向

 

找到 main 函数

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int i; // [rsp+4h] [rbp-1Ch]
  char *input; // [rsp+10h] [rbp-10h]
  _DWORD *v6; // [rsp+18h] [rbp-8h]
 
  input = (char *)malloc(0x21uLL);
  printf("input the a str:");
  __isoc99_scanf("%s", input);
  if ( (unsigned int)strlen(input) != 32 )
  {
    printf("oh,no!");
    exit(0);
  }
  v6 = encrypt(~(53 * (2 * input[6] + input[15] + 3 * input[29])) & 0xFFF, (__int64)input);
  for ( i = 0; i <= 31; ++i )
  {
    if ( dword_63A380[i] != v6[i] )
    {
      printf("oh,no!");
      exit(0);
    }
  }
  puts("Success!");
  return 0;
}

```

输入长度为 32 的字符串，然后从中截取三个字符来计算得到 encrypt 函数的参数 1，并将字符串作为参数 2 传入函数

 

使用 bindiff 修复符号

 

效果如下：

 

![](https://bbs.pediy.com/upload/attach/202107/921830_QKJSVHVVZ74DAA4.png)

 

可以很准确的识别出 lua 的函数

 

对照 lua 文档链接：[http://www.lua.org/manual/5.4/](http://www.lua.org/manual/5.4/)

 

分析 encrypt 函数：（请查看注释）

```
void *__fastcall encrypt(unsigned int a1, __int64 a2)
{
  _BYTE *functionname; // rax
  double lua_func_parameter; // xmm0_8
  __int64 v4; // rcx
  __int64 v5; // r8
  __int64 v6; // r9
  int i; // [rsp+20h] [rbp-70h]
  int j; // [rsp+24h] [rbp-6Ch]
  int k; // [rsp+28h] [rbp-68h]
  FILE *fp; // [rsp+38h] [rbp-58h]
  __int64 size; // [rsp+40h] [rbp-50h]
  void *ptr; // [rsp+48h] [rbp-48h]
  const char *decode_file; // [rsp+50h] [rbp-40h]
  __int64 L; // [rsp+58h] [rbp-38h]
  void *s; // [rsp+70h] [rbp-20h]
  void *final; // [rsp+78h] [rbp-18h]
 
  setvbuf(stdout, 0LL, 2, 0LL);
  fp = fopen("my.lua", "rb");
  if ( !fp )
    exit(0);
  fseek(fp, 0LL, 2);
  size = ftell(fp);
  ptr = malloc(size);
  rewind(fp);
  fread(ptr, 1uLL, size, fp);                   // 读取文件
  decode_file = xorfunc((__int64)ptr, size);    // 对文件进行异或解密
  L = luaL_newstate();                          // luaL_newstate创建lua状态机
  luaopen_base(L);                              // luaopen_base调用基本库
  luaL_openlibs(L);                             // /* 打开之前luaopen_base调用的lua状态机中所有Lua标准库 */
  if ( !L )
    exit(0);
  if ( (unsigned int)luaL_loadstring(L, decode_file) )// Loads a string as a Lua chunk.
    exit(0);
  s = malloc(132uLL);
  memset(s, 0, 132uLL);
  for ( i = 0; i <= 32; ++i )                   // 根据传入的a1值,循环33次生成一个序列
  {
    *((_DWORD *)s + i) = (0x1ED0675 * (unsigned __int64)a1 + 0x6C1) % 254;
    a1 = *((_DWORD *)s + i);
  }
  final = malloc(0x100uLL);
  memset(final, 0, 0x100uLL);
  for ( j = 0; j <= 31; ++j )
  {
    for ( k = 0; k <= 32; ++k )
    {
      *((_DWORD *)final + j + k) += *(char *)(j + a2) ^ *((_DWORD *)s + k);
      lua_pcallk(L, 0, 0, 0, 0LL, 0LL);         // Calls a function (or a callable object) in protected mode
      functionname = xorfunc((__int64)off_63A360, 7);// 对这个字符串进行异或解密
      lua_getglobal(L, (__int64)functionname);  // lua_getglobal 要调用的lua的函数名
      lua_func_parameter = (double)*((int *)final + j + k);
      lua_pushnumber(L, lua_func_parameter);    // 参数入栈
      lua_pcallk(L, 1, 1, 0, 0LL, 0LL);
      lua_tonumberx(L, 0xFFFFFFFFLL, 0LL, v4, v5, v6);// lua_tonumberx  取出返回值
      *((_DWORD *)final + j + k) = (int)lua_func_parameter;
    }
  }
  return final;
}

```

```
_BYTE *__fastcall xorfunc(__int64 a1, int a2)
{
  _BYTE *result; // rax
  int i; // [rsp+14h] [rbp-2Ch]
  int v4[6]; // [rsp+20h] [rbp-20h]
  unsigned __int64 v5; // [rsp+38h] [rbp-8h]
 
  v5 = __readfsqword(0x28u);
  result = malloc(a2);
  v4[0] = 2;
  v4[1] = 3;
  v4[2] = 5;
  if ( a2 )
  {
    for ( i = 0; i < a2; ++i )
      result[i] = *(_BYTE *)(i + a1) ^ LOBYTE(v4[i % 3]);
  }
  return result;
}

```

本来想动态调试，让程序自己载入 my.lua 进行解密，但是拖入 winhex 查看，发现 OEP（e_entry）被修改过，直接./re1，会提示非法指令

 

只能静态实现它对文件的异或解密了

```
# _*_ coding: utf-8 _*_
# editor: SYJ
# function: Reversed By SYJ
# describe:
fp = open("D:\\CTF\\COMPETITIONS\\2021jingmen\\REVERSE\\easyRe\\my.lua", "rb")
data = fp.read()
xor_arr = [2, 3, 5]
def xor_func(data):
    for i in range(len(data)):
        print(chr((data[i] ^ xor_arr[i % 3]) & 0xff), end='')
 
print("下面是调用xorfuncd对文件解密的结果")
xor_func(data)
print("\n下面是循环中调用xorfunc对off_63A360位置的字符串cgfffce解密的结果")
xor2 = [0x63, 0x67, 0x66, 0x66, 0x66, 0x63, 0x65]
xor_func(xor2)

```

运行可以得到：

```
下面是调用xorfuncd对文件解密的结果
function BitXOR(a,b)
    local p,c=1,0
    while a>0 and b>0 do
        local ra,rb=a%2,b%2
        if ra~=rb then c=c+p end
        a,b,p=(a-ra)/2,(b-rb)/2,p*2
    end
    if a**0 do
        local ra=a%2
        if ra>0 then c=c+p end
        a,p=(a-ra)/2,p*2
    end
    return c
end
 
function adcdefg(j) 
    return BitXOR(5977654,j)
end
下面是循环中调用xorfunc对字符串cgfffce解密的结果
adcdefg** 
```

分析这段 lua：

```
function BitXOR(a,b)
    local p,c=1,0
    while a>0 and b>0 do
        local ra,rb=a%2,b%2         # 取最后一bit
        if ra~=rb then c=c+p end    # 如果最后一bit不相等，c = c+p
        a,b,p=(a-ra)/2,(b-rb)/2,p*2  # a等于除开最后一bit后还有多少个2，b等于除开最后一bit后还有多少个2，p=p*2，二进制进位
    end
    if a**0 do
        local ra=a%2          
        if ra>0 then c=c+p end    # a的最后一bit如果为1，就和b对应的bit(因为b小于a,这个bit必为0)异或，等于1，要二进制进位
        a,p=(a-ra)/2,p*2               # a等于除开最后一bit后还有多少个2，p二进制进位
    end
    return c
end
 
function adcdefg(j) 
    return BitXOR(5977654,j)      # 所以这个函数就是一个bit一个bit的进行判断，得到最后异或的值
end** 
```

其实就是实现的按 bit 异或，跟函数名 BitXOR 照应，所以 function abcdefg(j) 就是 5977654 **^ j**

 

然后 encrypt 加密后返回，在 main 函数中与内存中的一段数据 dword_63A380 进行比较

```
seg023:000000000063A380 ; int dword_63A380[32]
seg023:000000000063A380 dword_63A380    dd 5B368Bh, 136h, 5B37CBh, 0E0Ch
seg023:000000000063A380                                         ; DATA XREF: main+D3↑r
seg023:000000000063A380                 dd 5B396Ah, 39h, 5B32A5h, 0CBAh
seg023:000000000063A380                 dd 5B2683h, 0CB9h, 5B3777h, 0E47h
seg023:000000000063A380                 dd 5B33BDh, 1F49h, 5B34A8h, 4D1h
seg023:000000000063A380                 dd 5B27E7h, 0D76h, 5B3BDFh, 2EFh
seg023:000000000063A380                 dd 5B3CD3h, 1F6Eh, 5B3D90h, 3DAh
seg023:000000000063A380                 dd 5B3184h, 524h, 5B330Eh, 1A10h
seg023:000000000063A380                 dd 5B2E84h, 0D3h, 5B3DA1h, 2F7Bh

```

使用写出最后的解密脚本

```
# decrypt部分
from z3 import *
 
dest_enc = [0x005B368B, 0x00000136, 0x005B37CB, 0x00000E0C, 0x005B396A, 0x00000039, 0x005B32A5, 0x00000CBA, 0x005B2683, 0x00000CB9, 0x005B3777, 0x00000E47, 0x005B33BD, 0x00001F49, 0x005B34A8, 0x000004D1, 0x005B27E7, 0x00000D76, 0x005B3BDF, 0x000002EF, 0x005B3CD3, 0x00001F6E, 0x005B3D90, 0x000003DA, 0x005B3184, 0x00000524, 0x005B330E, 0x00001A10, 0x005B2E84, 0x000000D3, 0x005B3DA1, 0x00002F7B]
for seed in range(0xfff):
    xor_data = []
 
    for i in range(33):        # 循环33次生成序列
        r = (0x1ED0675 * seed + 0x6c1) % 0xfe
        xor_data.append(r)
        seed = r
 
    s=Solver()
 
    flag = [BitVec(('x%d' % i), 8) for i in range(32)]
    xor_result = [0 for i in range(64)]
    for i in range(32):
        for j in range(33):
            a = flag[i] ^ xor_data[j]
            xor_result[i + j] += a
            xor_result[i+j] = (xor_result[i+j] ^ 5977654)
 
    for i in range(0, 32):
        s.add(flag[i] <= 127)
        s.add(flag[i] >= 32)
        s.add(xor_result[i] == dest_enc[i])
 
    if s.check() == sat:
        model = s.model()
        str = [chr(model[flag[i]].as_long().real) for i in range(32)]
        print("".join(str))
        exit()
 
# flag{cfcd208495d565ef66e7dff9f98764da}

```

参考：

 

[https://www.cnblogs.com/lijiajia/p/8284328.html](https://www.cnblogs.com/lijiajia/p/8284328.html)

 

[https://www.cnblogs.com/huangjianxin/p/10581174.html](https://www.cnblogs.com/huangjianxin/p/10581174.html)

 

[https://www.runoob.com/lua/lua-environment.html](https://www.runoob.com/lua/lua-environment.html)

 

[https://blog.csdn.net/sln_1550/article/details/116614228](https://blog.csdn.net/sln_1550/article/details/116614228)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#Reverse](forum-37-1-111.htm)

上传的附件：

*   [easyRe.zip](javascript:void(0)) （1.49MB，6 次下载）