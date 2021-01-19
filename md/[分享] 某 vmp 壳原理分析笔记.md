> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-225798.htm)

某 vmp 壳原理分析笔记
=============

分析的样本为某数字公司最新免费壳子。之前的壳子已经被很多大佬分析了，这篇笔记的主要目的是比较详细的分析下该 vmp 壳子的原理，数字壳子主要分为反调试，linker，虚拟机三部分。笔记结构如下：

*   反调试
    
    *   时间反调试
    *   rtld_db_dlactivity 反调试
    *   traceid 反调试
    *   端口反调试
*   linker 部分：用来加载第二个 so
    
    *   装载
    *   创建 soinfo
    *   链接
    *   dump second so
*   虚拟机部分：解释执行保护的代码
    
    *   dump dex
    *   onCreate 分析
    *   虚拟机入口

准备部分
----

因为懒得 hook 系统函数，每次过反调试就要手工更改寄存器的值，所以用 IDAPython 来模拟手工调试，下面是一些辅助函数，其中 addBrk 根据模块名和偏移地址下断点，fn_f7, fn_f8, fn_f9 分别模拟 f7, f8, f9.

```
def getModuleBase(moduleName):
    base = GetFirstModule()
    while (base != None) and (GetModuleName(base).find(moduleName) == -1):
        base = GetNextModule(base)
    if base == None:
        print "failed to find module: " + moduleName
        return None
    else:
        return base
 
def addBrk(moduleName, functin_offset):
    base = getModuleBase(moduleName)
    AddBpt(base + functin_offset)
 
 
def fn_f7():
    idaapi.step_into()
    GetDebuggerEvent(WFNE_SUSP | WFNE_SUSP, -1)
 
def fn_f8():
    idaapi.step_over()
    GetDebuggerEvent(WFNE_SUSP | WFNE_CONT, -1)
 
def fn_f9():
    idaapi.continue_process()
    GetDebuggerEvent(WFNE_SUSP | WFNE_CONT, -1)
 
def delBrk(moduleName, function_offset):
    base = getModuleBase(moduleName)
    DelBpt(base + function_offset)

```

反调试
---

### 时间反调试

反调试的第一处就是时间反调试，在反调试的开始调用 time 获取时间，在结束再调用一次 time，当差值大于 3 就会触发反调试。因为使用脚本过反调试，所以两次时间差肯定是小于 3 的，所以只是用脚本简单的输出时间就可以了。0x19FC 是 time 在 libjiagu.so 的 plt 段地址。

```
#add breakpoint at time
addBrk("libjiagu.so", 0x19FC)
fn_f9()
delBrk("libjiagu.so", 0x19FC)
print("first call time")
lr = GetRegValue("LR")
AddBpt(lr)
fn_f9()
r0 = GetRegValue("R0")
DelBpt(lr)
print("first get time: %x" % (r0))

```

### rtld_db_dlactivity 反调试

rtld_db_dlactivity 函数：这个函数默认情况下为空函数，这里的值应该为 0，而当有调试器时，这里会被改为断点指令，所以可以被用来做反调试。libjiagu.so 的 sub_1E9C 就是用来查找 rtld_db_dlactivity 函数的地址的。sub_1E9C 的代码就不贴了。过这个反调试我们就可以在 sub_1E9C 返回后，获取到 rtld_db_dlactivity 的地址，然后将其修改为 nop 指令 0x46C0。

```
#add breakpoint at sub_1E9C
addBrk("libjiagu.so", 0x1E9c)
fn_f9()
delBrk("libjiagu.so", 0x1E9c)
print("call sub_1E9C")
#add breakpoint at return from sub_1E9C
lr = GetRegValue("LR")
AddBpt(lr)
fn_f9()
DelBpt(lr)
#get rtld_db_dlactivity address
r0 = GetRegValue("R0")
print("rtld_db_dlactivity address is: %x" % (r0 - 1))
#set rtld_db_dlactivity nop
PatchDword(r0 - 1, 0x46c0)

```

### tracepid 反调试

tracepid 反调试: 当我们使用 Ptrace 方式跟踪一个进程时，目标进程会记录自己被谁跟踪。过 tracepid 反调试可以在调用 strtol 后，将其返回值修改为 0.

```
#add breakpoint at strtol
addBrk("libjiagu.so", 0x1A80)
fn_f9()
delBrk("libjiagu.so", 0x1A80)
print("call strtol")
lr = GetRegValue("LR")
#add breakpoint at return from strtol
AddBpt(lr)
fn_f9()
DelBpt(lr)
r0 = GetRegValue("R0")
print("get trace id: %x" % (r0))
SetRegValue(0, "R0")

```

### 端口反调试

检测 IDA 的默认端口 23946 是否被占用, 通过跟踪，可以发现 sub_6DD0() 函数判断 23946 端口是否被占用。原理就是查看 / proc/net/tcp，过端口反调试，就可以通过更改 sub_6DD0 的返回值。

```
#add break point at sub_6DD0
addBrk("libjiagu.so", 0x6DD0)
fn_f9()
lr = GetRegValue("LR")
print("call sub_6DD0")
delBrk("libjiagu.so", 0x6DD0)
AddBpt(lr)
fn_f9()
SetRegValue(0, "R0")
DelBpt(lr)

```

### 时间反调试

在反调试的最后还会调用一次 time，计算差值，因为使用脚本过得，所以时间差值肯定小于 3，这里只是简单输出一下时间

```
#add break point at time in  sub_6EF8
addBrk("libjiagu.so", 0x19FC)
fn_f9()
delBrk("libjiagu.so", 0x19FC)
print("second call time")
lr = GetRegValue("LR")
AddBpt(lr)
fn_f9()
DelBpt(lr)
r0 = GetRegValue("R0")
print("second get time: %x" % (r0))
#SetRegValue(r0 + 2, "R0")

```

Linker 部分
---------

这部分的主要作用是手工实现加载链接第二个 so，并且调用第二个的 so 的 JNI_Onload 方法。因为篇幅问题，只列出一些核心的步骤，具体的调试的方法是在 case 31 的 BLX LR 处下断点，跟踪 sub_7BBC 函数，就可以分析 linker 部分。既然是手工实现 linker，那 linker 的主要功能就必不可少：装载和链接。下面就对应 libjiagu.so 中的函数来分析具体的装载与链接过程。分析的起点为 sub_3DBC, sub_3DBC 中调用了 sub_4780,sub_3D60,sub_33F8, 分别对应装载，创建 soinfo，链接三个过程。首先定义两个后面会用到的数据结构，这两个数据结构的来源是通过动态调试分析出的，是后续很多函数的参数类型，和 libjiagu.so 的真实实现可能会有出入。

```
struct EncryptSoInfo {
void *encryptSo;             //指向压缩加密的so
int sizeOfEncryptSo;         //压缩加密so的大小
void *uncompressedSo;        //解压后的so位置
int sizeOfUncompressSo;      //解压后的so大小
int num;                     //表示某种类型，在后面会用到一次，具体含义不知道
char name[4];                //第二个so的名字
char data[124];
 
};
 
//类似于标准linker中的ELFReader
struct MElfReader {
void * soinfo
Byte[52] header;             // offset 4
int num_program_header;      //段表的个数 offset 56
ProgreamHeaderTable *pht;    //段表 offset 60
int unkonw;                  // offset 64
void *load_base;               //segment 起始地址 offset 68
int size; //offset 72
void *load_bias;         //offset 76
};

```

### 装载

这一步骤 libjiagu.so 的实现和标准 linker 的实现十分相似。先说下标准 linker 的做法，创建 ElfReader 对象，通过 ElfReader 对象的 Load 方法将 SO 文件装载到内存。

```
//标准linker的装载
bool ElfReader::Load() {
  return ReadElfHeader() &&      //读取elf header
         VerifyElfHeader() &&    //验证elf header
         ReadProgramHeader() &&  //读取program header
         ReserveAddressSpace() &&  //分配空间
         LoadSegments() &&         //按照program header的指示装载segments
         FindPhdr();               //找到装载后的phdr
}

```

下面看下 libjiagu.so 的实现，只是少了验证 elf header 的环节。

```
int __fastcall sub_4780(MELFReader *a1, EncryptSoInfo *a2)
{
  Byte *v2; // r5@1
  Byte *v3; // r4@1
  int result; // r0@3
 
  v2 = a2;
  v3 = a1;
  if ( a2 && sub_40B0(a1, a2)   //读取elf header
          && sub_41A8(v3, v2)   //读取program header
          && sub_4424((int)v3, (int)v2) && //分配空间
           sub_4448(v3, v2) )    //按照program header的指示装载segments
    result = sub_46CC(v3);      //找到装载后的phdr
  else
    result = 0;
  return result;
}

```

#### load ELF Header

装载的第一步就是加载 ELF Header，这一过程比较简单，直接看代码

```
int __fastcall sub_40B0(MELFReader *a1, EncryptSoInfo *a2)
{
  Byte *v2; // r4@1
  const void *base; // r1@1
  int v4; // r4@3
  Byte *v6; // r5@5
  _DWORD *v7; // r0@5
  _DWORD *v8; // r6@5
 
  v2 = a2;
  base = (const void *)*((_DWORD *)a2 + 2); //a2->uncompressedSo
  if ( base
    && *((_DWORD *)v2 + 3) > 0x34u   //a2->int sizeOfUncompressSo
    && (v6 = a1 + 4,
     memcpy(a1 + 4, base, 0x34u),  //拷贝加密的后的ELF Header
     v7 = sub_6534(*((_DWORD *)v2 + 40)), //新建解密对象v7
     (v8 = v7) != 0)
    && sub_58F0((int)v7, (int)v6, 52)     //初始化解密对象
    && (v4 = (*(int (__fastcall **)(_DWORD *))(*v8 + 12))(v8)) != 0 //解密加密后的so
    )
  {
    (*(void (__fastcall **)(_DWORD *))(*v8 + 4))(v8); //析构解密对象
  }
  else
  {
    v4 = 0;
  }
  return v4;
}

```

#### load program header

这一过程也比较简单，利用上一步读取的 program header，得到 program header 数量，然后分配空间，拷贝加密后到的 program header table 到分配的空间，然后解密。

```
int __fastcall sub_41A8(MELFReader *a1, EncryptSoInfo *a2)
{
  Byte *v2; // r5@1
  Byte *v3; // r4@1
  size_t v4; // r0@2
  size_t v5; // r6@3
  void *v6; // r0@3
  _DWORD *v7; // r0@4
  _DWORD *v8; // r5@4
  int v9; // r4@6
 
  v2 = a2;
  v3 = a1;
  if ( a2
    && (v4 = *((_WORD *)a1 + 24), //program header 数量
    *((_DWORD *)v3 + 14) = v4, ((v4 - 1) & 0xFFFF) <= 0x7FF)
    && (v5 = 32 * v4, *((_DWORD *)v3 + 16) = 32 * v4, v6 = calloc(v4, 0x20u), // 分配空间
     (*((_DWORD *)v3 + 15) = v6) != 0)
    && (memcpy(v6, (const void *)(*((_DWORD *)v2 + 2) + *((_DWORD *)v3 + 8)), v5), //拷贝段表
        v7 = sub_6534(*((_DWORD *)v2 + 41)), 
        (v8 = v7) != 0)
    && sub_58F0((int)v7, *((_DWORD *)v3 + 15), *((_DWORD *)v3 + 16))
    && (v9 = (*(int (__fastcall **)(_DWORD *))(*v8 + 12))(v8)) != 0 ) //解密段表
  {
    (*(void (__fastcall **)(_DWORD *))(*v8 + 4))(v8);
  }
  else
  {
    v9 = 0;
  }
  return v9;
}

```

#### 计算加载所需的空间并分配空间

ELF 文件中所有类型为 LOAD 的 segment 都需加载到内存，这一步骤主要是计算加载所需要的空间并分配空间。这里需要说明一下 loadbias，因为 so 可以指定加载基址，但指定的基址可能不是页对齐，所以实际加载的基址可能会和指定基址有个差值，loadbias 用来保存这个差值。但一般 so 都是可以加载到任意地址的，所以 loadbias 的值一般就是 so 实际加载的基址。

```
signed int __fastcall sub_43A8(MELFReader *a1)
{
  Byte *v1; // r4@1
  int size; // r0@1
  signed int result; // r0@2
  _BYTE *v4; // r5@3
  _BYTE *v5; // r0@3
  int v6; // r5@4
  void *addr; // [sp+Ch] [bp-14h]@1
 
  v1 = a1;
  //计算加载所需的空间并且将加载的基址保存在addr中
  size = sub_4264(*((_DWORD *)a1 + 15), *((_DWORD *)a1 + 14), (unsigned int *)&addr, 0);
  *((_DWORD *)v1 + 18) = size;
  //mmap分配空间
  if ( size && (v4 = addr, v5 = mmap(addr, size, 0, 34, -1, 0), v5 != (_BYTE *)-1) )
  {
    v6 = v5 - v4;   //计算 loadbias
    *((_DWORD *)v1 + 17) = v5;
    result = 1;
    *((_DWORD *)v1 + 19) = v6;
  }
  else
  {
    result = 0;
  }
  return result;
}
sub_4264的原理，因为篇幅原因就不贴代码了，就是循环段表，读取每个需要load的段，然后找到加载的最小虚拟地址，和最大的虚拟地址，页对齐后，取差值就是load size。

```

#### load segment

遍历 program header table，找到类型为 PT_LOAD 的 segment:

1.  计算 segment 在内存空间中的起始地址 segstart 和结束地址 seg_end，seg_start 等于虚拟偏移加上基址 load_bias，同时由于 mmap 的要求，都要对齐到页边界得到 seg_page_start 和 seg_page_end。
2.  计算 segment 在文件中的页对齐后的起始地址 file_page_start 和长度 file_length。
3.  使用 mrpotect 和 memcpy 将段映射到内存，指定映射地址为 seg_page_start，长度为 file_length，文件偏移为 file_page_start。
    
    ```
    signed int __fastcall sub_4448(MELFReader *a1, EncryptSoInfo *a2)
    {
    Byte *v2; // r6@1
    unsigned int v3; // r2@3
    int v4; // r4@4
    int v5; // r5@7
    int v6; // r3@7
    int v7; // r5@7
    int v8; // r12@8
    int v9; // r2@8
    int v10; // r10@8
    unsigned int v11; // r11@8
    unsigned int v12; // r12@8
    bool v13; // cf@8
    bool v14; // zf@8
    unsigned int v15; // r8@8
    void *v16; // r7@8
    int v17; // r12@8
    int v18; // r10@8
    void *v19; // r0@16
    signed int v20; // r2@19
    size_t n; // [sp+4h] [bp-34h]@10
    unsigned int v23; // [sp+8h] [bp-30h]@2
    Byte *v24; // [sp+Ch] [bp-2Ch]@1
     
    v24 = a2;
    v2 = a1;
    if ( a2 )
    {
     v23 = *((_DWORD *)a2 + 3);                  // 解压后的so的起始地址
     if ( *((_DWORD *)a2 + 3) )
     {
       v3 = *((_DWORD *)a1 + 14);                // segment 数量
       if ( !v3 )
         return 1;
       v4 = 0;
       while ( 1 )
       {
         while ( 1 )                             // 找到类型为LOAD的段
         {
           v5 = *((_DWORD *)v2 + 15);            // 段表起始地址
           v6 = *(_DWORD *)(v5 + 32 * v4);       // 迭代读取段表
           v7 = v5 + 32 * v4;
           if ( v6 == 1 )
             break;
           if ( v3 <= ++v4 )
             return 1;
         }
         v8 = *(_DWORD *)(v7 + 4);               // offset segment在文件的偏移
         v9 = *(_DWORD *)(v7 + 16);              // filesz segment在so中大小
         v10 = *((_DWORD *)v2 + 19) + *(_DWORD *)(v7 + 8);// segment 在虚拟内存的结束地址
         v11 = v8 & 0xFFFFF000;
         v12 = v8 + v9;
         v13 = v23 >= v12;
         v14 = v23 == v12;
         v15 = (*(_DWORD *)(v7 + 20) + 4095 + v10) & 0xFFFFF000;
         v16 = (void *)(v10 & 0xFFFFF000);
         v17 = v12 - v11;
         v18 = v10 + v9;
         if ( v14 || !v13 )
           break;
         n = v17;
         if ( mprotect(v16, v15 - (_DWORD)v16, 3) == -1 )
           break;
         if ( n )
           memcpy(v16, (const void *)(*((_DWORD *)v24 + 2) + v11), n);
         if ( *(_DWORD *)(v7 + 24) & 2 && v18 & 0xFFF )
           memset((void *)v18, 0, 4096 - (v18 & 0xFFF));
         v19 = (void *)((v18 + 4095) & 0xFFFFF000);
         if ( v15 > (unsigned int)v19 )
           memset(v19, 0, v15 - (_DWORD)v19);
         v20 = *(_DWORD *)(v7 + 24) & 1 ? 4 : 0;
         if ( mprotect(v16, v15 - (_DWORD)v16, *(_DWORD *)(v7 + 24) & 2 | (*(_DWORD *)(v7 + 24) << 29 >> 31) | v20) == -1 )
           break;
         v3 = *((_DWORD *)v2 + 14);
         if ( v3 <= ++v4 )
           return 1;
       }
     }
    }
    return 0;
    }
    
    ```
    
    至此第二个 so 已经完成了装载。
    
    ### 创建 soinfo
    
    完成装载后，就需要创建 soinfo 结构，并利用 MELFReader 设置一些 soinfo 参数。
    

```
char *__fastcall sub_3D60(EncryptSoInfo *a1)
{
  const char *v1; // r4@1
  char *result; // r0@2
  char *v3; // r5@2
 
  v1 = (const char *)(a1 + 20);
  if ( strlen((const char *)a1 + 20) > 0x7F )
  {
    result = 0;
  }
  else
  {
    result = (char *)operator new(0x128u); //创建soinfo
    v3 = result;
    if ( result )
    {
      memset(result, 0, 0x128u);
      strncpy(v3, v1, 0x7Fu);
      result = v3;
    }
  }
  return result;
}
 
sub_3DBC(EncryptSoInfo *a1)
{
  //节选
  soinfo = sub_3D60(*(Byte **)v1);
  v3 = (int)soinfo;
  if ( !soinfo )
    goto LABEL_16;
  v4 = nmemb_56;
  v5 = nmemb_68;
  v6 = nmemb_72;
  v7 = nmemb_76;
  *((_DWORD *)soinfo + 33) = nmemb_56;          // 设置 soinfo->phnum;
  *((_DWORD *)soinfo + 35) = v5;                // 设置 soinfo->loadstart
  *((_DWORD *)soinfo + 36) = v6;                // 设置 soinfo->size; 
  *((_DWORD *)soinfo + 62) = v7;                // 设置 soinfo->loadbias
  //
}

```

### 链接

自定义 linker 最重要的一步就是链接，链接主要步骤是：

1.  定位 dynamic segment
2.  解析 dynamic section
3.  加载该 so 所依赖的 so
4.  重定位  
    这一部分是由 sub_33F8 函数完成的，因为这个函数很长，所以节选主要步骤贴出来

```
signed int __fastcall sub_33F8(soinfo *a1)
{
    //遍历program header找到类型为Dynamic的program header，从而定位到Dynamic segment
    sub_45F4(*((_DWORD *)a1 + 32), *((_DWORD *)v2 + 33), *((_DWORD *)v2 + 62), (_DWORD *)v2 + 37, &v38);
    v3 = *((_DWORD *)v2 + 37); //dynamic segment
 
    //解析dynamic segment
    v4 = *(_DWORD *)v3; //符号类型
 
    if ( *(_DWORD *)v3 ) {
        v5 = (int *)(v3 + 8);
        while ( 1 ) {
            switch ( v4 ) {
 
                //根据符号类型做重定位
            }
        }
    }
 
    LABEL_7:
    if ( *(_DWORD *)v3 )
    {
      v11 = 0;
      do
      {
        if ( v10 == 1 )
        {
          v13 = (const char *)(*((_DWORD *)v2 + 39) + *(_DWORD *)(v3 + 4));
          if ( strlen(v13) > 0x80 )
            goto LABEL_21;
          if ( *((_DWORD *)v2 + 72) <= v11 )
            goto LABEL_21;
          v14 = 136 * v11++;
          strncpy((char *)(*((_DWORD *)v2 + 73) + v14 + 4), v13, 0x7Fu);
          v15 = dlopen(v13, 0);  //加载所需要的so
          if ( !v15 )
            goto LABEL_21;
          *(_DWORD *)(*((_DWORD *)v2 + 73) + v14) = v15;
          *(_DWORD *)(*((_DWORD *)v2 + 73) + v14 + 132) = 0;
        }
        v12 = *(_DWORD *)(v3 + 8);
        v3 += 8;
        v10 = v12;
      }
      while ( v12 );
    }
 
}

```

### dump second so

清楚第二个 so 的加载流程后，就可以 dump 出第二个 so，具体做法是在 sub_3DBC 处下断点，dump 内存，并且解密 ELF header 和 program header table。解密算法是 libjiagu.so 中的 sub_6868.

```
#sub_6868
def decryptso(data, size):
    result = bytearray(data)
    for i in range(size):
        result[i] ^= 0x50
    return str(result)
 
 
def dump_so(start_address, size):
    fp = open("E:\\dump.so", "wb")
 
    #read elf header and decrypt header
    data = idaapi.dbg_read_memory(start_address, 52)
    header = decryptso(data, 52)
    fp.write(header)
 
    #read elf program header tables
    phnum = 9
    data = idaapi.dbg_read_memory(start_address + 52,  phnum * 32)
    pht = decryptso(data, phnum * 32)
    fp.write(pht)
 
    #read other part
    data = data = idaapi.dbg_read_memory(start_address + 52 + phnum * 32,  size - phnum * 32 - 52)
    fp.write(data)
 
    fp.close()

```

### 进入 JNI_Onload

在加载完第二个 so 后，libjiagu.so 通过 sub_3F7C(soinfo _info, char_ fucname), 来找到第二个 so 的 JNI_Onload 入口，并在 case 35 中跳转到第二个 so 的 JNI_Onload.

 

![](https://bbs.pediy.com/upload/attach/201804/657395_54H6YW432MJSE9D.png)

虚拟机部分
-----

这一部分就正式开始执行源 apk 中的代码了。

### dump dex

源 dex 要被执行，肯定需要加载，所以在 libart.so 的 OpenMemory 函数下断点就可以 dump 出 dex，使用 IDAPython dump dex

```
#add break at OpenMemory
addBrk("libart.so", OpenMemory_offset)
fn_f9()
r0 = GetRegValue("R0")
r1 = GetRegValue("R1")
print("orgin dex at: %x" % (r0))
delBrk("libart.so", OpenMemory_offset)
#dump dex
data = idaapi.dbg_read_memory(r0, r1)
fp = open('d:\\dump.dex', 'wb')
fp.write(data)
fp.close()

```

用 jeb 打开 dump 出来的 dex 可以发现 oncreate 函数 native 化了。

 

![](https://bbs.pediy.com/upload/attach/201804/657395_E52HM78D5K8KKZA.png)

### 找到 onCreate 函数

native 化通过静态修改 dex 中 onCreate 函数的 DexMethod 结构就可以，但要想正确执行，必须在执行时动态注册，通过拦截 JNI 注册函数 RegisterNative 可以找到 onCreate 函数的地址

```
def hook_RegisterNative():
    #add break at RegisterNatives
    addBrk("libart.so", RegisterNative_offset)
    fn_f9()
    delBrk("libart.so", RegisterNative_offset)
    #JNINativeMethod method[]
    r2 = GetRegValue("R2")
    #nMethods
    r3 = GetRegValue("R3")
    for index in range(r3):
        name = get_string(Dword(r2))
        address = Dword(r2 + 8)
        print("native function %s address is %x" % (name, address))
        r2 = r2 + 12

```

![](https://bbs.pediy.com/upload/attach/201804/657395_QHERWV4Z7Y65M7D.png)

 

然后在 oncreate 函数下断点

### onCreate 函数分析

通过上一步可以找到 onCreate 在第二个 so 中的地址，f5 之后代码如下：

```
int __fastcall onCreate(JNIEnv *a1, int a2, int a3, int a4)
{
  int v5; // [sp+1Ch] [bp-Ch]@1
  int v6; // [sp+20h] [bp-8h]@1
  int v7; // [sp+24h] [bp-4h]@1
 
  v7 = a4;
  v6 = a3;
  v5 = a2;
  return sub_D930(0, a1, &v5);
}

```

直接调用了 sub_D930，进入 sub_D930 分析，这个函数比较长，大致分析了一下流程：

1.  jni 的一些初始化工作，FindClass，GetMethodID 之类的工作
2.  利用 java.lang.Thread.getStackTrace 获取到调用当前方法的类的类名以及函数名
3.  通过上一步获取的类名以及方法名获取被保护方法的 DexCode 结构
4.  调用自己的虚拟机执行代码

其中第二步的核心代码如下：

```
sub_D930()
{
 
  //节选
  v54 = sub_66BD4(v122, (int)&v127);
  if ( v127 & 1 )
    j_j_j__ZdlPv(*((void **)v50 + 5));
  if ( v54 && (v55 = *(_DWORD *)(v54 + 4), (*(_DWORD *)(v54 + 8) - v55) >> 2 > v4) )
  {
    v56 = v55 + 4 * v4;                         // **v56为Dexprotoid， *(*v56+4)为DexMethodID， *(*v56+12)为codeoff
    v114 = (int)jni_env;
    v123 = *(_DWORD *)v56;                      // DexProtoId
    v107 = *(_DWORD **)v54;                     // **v54为dex在内存的地址
    DexCode = (_WORD *)(**(_DWORD **)v54 + *(_DWORD *)(*(_DWORD *)v56 + 12));// 获取DexCode地址
    ((void (*)(void))(*jni_env)->PushLocalFrame)();
    v58 = j_j_j__Znwj(0x20u);
    v59 = *DexCode;
    *(_DWORD *)v58 = v114;                      // jni_env
    *(_DWORD *)(v58 + 4) = v59;                 // 使用的寄存器个数
    *(_DWORD *)(v58 + 8) = DexCode;
    *(_DWORD *)(v58 + 12) = 0;
 
    //进入虚拟机
    sub_3FE5C()
 
  }

```

核心为 sub_66BD4 函数，具体原理没分析。获取到了 onCreate 的 DexCode 后, 进入虚拟机执行加密后的 DexCode。

### 虚拟机入口

虚拟机的入口在 sub_3FE5C 中调用的 sub_3FF5C，进入虚拟机入口后，就如下图

 

![](https://bbs.pediy.com/upload/attach/201804/657395_CK3ZHVN2BFA5UMK.png)

 

具体原理就是：

1.  取出加密后的指令
2.  根据加密后指令，还原操作数，并算出一个分支数
3.  跳转到具体分支解释执行。

具体分支数的计算算法，参考文章中说的很详细了，就不重复了。至此壳子的基本原理分析完了，接下来还要学习下 davik 虚拟机是如何解释执行指令的。逆向新手，有错误欢迎指正。  
参考文章：  
https://bbs.pediy.com/thread-223796.htm

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)

最后于 2018-4-8 21:02 被 glider 菜鸟编辑 ，原因： 添加附件

上传的附件：

*   [debug.py](javascript:void(0)) （5.52kb，319 次下载）
*   [dump.so](javascript:void(0)) （649.65kb，237 次下载）
*   [样本. apk](javascript:void(0)) （2.04MB，288 次下载）