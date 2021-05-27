> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267842.htm)

> ELF 文件格式与 got 表 hook 简单实现

ELF 文件格式与 got 表 hook 简单实现 (安卓)
==============================

本文目的:
-----

通过了解 ELF 文件格式，了解 got 表 hook 原理。

 

本文用到的简单 demo 以及编译后的 so 库名称为 libtestLib.so，[源码](https://github.com/whulzz1993/SuperHook)

```
static void testOpen(JNIEnv* env, jclass clazz, jstring jPath) {
 
    const char* cPath = env->GetStringUTFChars(jPath, JNI_FALSE);
    int fd = open(cPath, O_RDONLY);
    LOGD("open %s fd: %d", cPath, fd);
    if (fd > 0) close(fd);
    env->ReleaseStringUTFChars(jPath, cPath);
 
}

```

ELF 简述
------

Executable and Linkable Format 可执行可链接文件，主要用在 linux 相关操作系统上，文件中存储了程序运行的信息

*   重定向文件。比如. a 与. o
*   可执行文件。比如 adb, init, zygote
*   共享动态库。比如平常 apk 中打包的 so 库，系统 so 库 (libc.so libdl.so libnativeloader.so libart.so)

ELF 文件魔数
--------

很多文件格式都是有固定的魔数信息，代表了文件属于哪一种格式。比如 zip 格式, tar.gz，exe 等

 

elf 文件也有固定魔数，我们通过 hexdump 看一下头部的信息

```
admin@C02D7132MD6R ~ % hexdump -C -n 20 ~/Downloads/libart.so                  
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  03 00 b7 00                                       |....|
00000014
admin@C02D7132MD6R ~ %

```

前四个字节就代表了 elf 文件的魔数，为什么是前四个字节？因为源码中就定义了魔数信息。

```
admin@C02D7132MD6R bionic % vim libc/kernel/uapi/linux/elf.h     
admin@C02D7132MD6R bionic %
admin@C02D7132MD6R bionic %

```

![](https://bbs.pediy.com/upload/attach/202105/855618_D5QWSYQWJXJE2KK.png)

 

SELFMAG 就是长度的意思，4 个字节。"\177" 是 8 进制，16 进制就是 7f。与 hexdump 呈现的 16 进制视图完全一致。

ELF 文件头信息
---------

文件头包含魔数，还包含其他关键信息。

 

文件头的 size 是多少呢？其实 armv8 文件头是固定的长度，armv7 文件头也是固定的长度。长度只与当前 so 库对应的平台相关。

 

比如 apk 中 libs/armeabi-v7a / 下面的所有 so 库都拥有一样长度的文件头。

 

使用 ida 工具打开一个 arm64 的 so 库，我们看看工具所解析的 elf 文件头包含的信息:  
![](https://bbs.pediy.com/upload/attach/202105/855618_M4568B5H2NBJ5PA.png)

 

结合该图，再来看看 linker 的源码，增加大家对 ELF 文件头的印象

 

源码位置 aosp7.1 bionic/linker/linker_phdr.cpp

### 信息校验

```
//通过pread64将文件头的内容读到内存header_中
bool ElfReader::ReadElfHeader() {
  ssize_t rc = TEMP_FAILURE_RETRY(pread64(fd_, &header_, sizeof(header_), file_offset_));
  if (rc < 0) {
    DL_ERR("can't read file \"%s\": %s", name_.c_str(), strerror(errno));
    return false;
  }
 
  if (rc != sizeof(header_)) {
    DL_ERR("\"%s\" is too small to be an ELF executable: only found %zd bytes", name_.c_str(),
           static_cast(rc));
    return false;
  }
  return true;
}
//校验elf文件头
bool ElfReader::VerifyElfHeader() {
  //1.校验魔数信息
  if (memcmp(header_.e_ident, ELFMAG, SELFMAG) != 0) {
    DL_ERR("\"%s\" has bad ELF magic", name_.c_str());
    return false;
  }
 
  //2.校验文件类别，64位运行时只能加载64位so库，32位运行时只能加载32位so库
  // Try to give a clear diagnostic for ELF class mismatches, since they're
  // an easy mistake to make during the 32-bit/64-bit transition period.
  int elf_class = header_.e_ident[EI_CLASS];
#if defined(__LP64__)
  if (elf_class != ELFCLASS64) {
    if (elf_class == ELFCLASS32) {
      DL_ERR("\"%s\" is 32-bit instead of 64-bit", name_.c_str());
    } else {
      DL_ERR("\"%s\" has unknown ELF class: %d", name_.c_str(), elf_class);
    }
    return false;
  }
#else
  if (elf_class != ELFCLASS32) {
    if (elf_class == ELFCLASS64) {
      DL_ERR("\"%s\" is 64-bit instead of 32-bit", name_.c_str());
    } else {
      DL_ERR("\"%s\" has unknown ELF class: %d", name_.c_str(), elf_class);
    }
    return false;
  }
#endif
 
  //3.检验大小端对齐方式，目前安卓上为小端对齐
  if (header_.e_ident[EI_DATA] != ELFDATA2LSB) {
    DL_ERR("\"%s\" not little-endian: %d", name_.c_str(), header_.e_ident[EI_DATA]);
    return false;
  }
 
  //4.校验文件类型是否为动态库
  if (header_.e_type != ET_DYN) {
    DL_ERR("\"%s\" has unexpected e_type: %d", name_.c_str(), header_.e_type);
    return false;
  }
 
  //5.校验版本号，安卓上目前只能是EV_CURRENT
  if (header_.e_version != EV_CURRENT) {
    DL_ERR("\"%s\" has unexpected e_version: %d", name_.c_str(), header_.e_version);
    return false;
  }
 
  //6.校验machine
  if (header_.e_machine != GetTargetElfMachine()) {
    DL_ERR("\"%s\" has unexpected e_machine: %d", name_.c_str(), header_.e_machine);
    return false;
  }
 
  return true;
} 
```

### 获取程序头表信息

```
bool ElfReader::ReadProgramHeaders() {
  phdr_num_ = header_.e_phnum;
 
  // Like the kernel, we only accept program header tables that
  // are smaller than 64KiB.
  if (phdr_num_ < 1 || phdr_num_ > 65536/sizeof(ElfW(Phdr))) {
    DL_ERR("\"%s\" has invalid e_phnum: %zd", name_.c_str(), phdr_num_);
    return false;
  }
 
  // Boundary checks
  size_t size = phdr_num_ * sizeof(ElfW(Phdr));
  if (!CheckFileRange(header_.e_phoff, size, alignof(ElfW(Phdr)))) {
    DL_ERR_AND_LOG("\"%s\" has invalid phdr offset/size: %zu/%zu",
                   name_.c_str(),
                   static_cast(header_.e_phoff),
                   size);
    return false;
  }
 
  //将elf文件的程序头表内容映射到内存中,程序头表的文件偏移为header_.e_phoff
  if (!phdr_fragment_.Map(fd_, file_offset_, header_.e_phoff, size)) {
    DL_ERR("\"%s\" phdr mmap failed: %s", name_.c_str(), strerror(errno));
    return false;
  }
 
  phdr_table_ = static_cast(phdr_fragment_.data());
  return true;
} 
```

### 获取节头表信息

```
bool ElfReader::ReadSectionHeaders() {
  shdr_num_ = header_.e_shnum;
 
  if (shdr_num_ == 0) {
    DL_ERR_AND_LOG("\"%s\" has no section headers", name_.c_str());
    return false;
  }
 
  size_t size = shdr_num_ * sizeof(ElfW(Shdr));
  if (!CheckFileRange(header_.e_shoff, size, alignof(const ElfW(Shdr)))) {
    DL_ERR_AND_LOG("\"%s\" has invalid shdr offset/size: %zu/%zu",
                   name_.c_str(),
                   static_cast(header_.e_shoff),
                   size);
    return false;
  }
 
  //将elf文件的节头表内容映射到内存中，节头表的文件偏移为header_.e_shoff
  if (!shdr_fragment_.Map(fd_, file_offset_, header_.e_shoff, size)) {
    DL_ERR("\"%s\" shdr mmap failed: %s", name_.c_str(), strerror(errno));
    return false;
  }
 
  shdr_table_ = static_cast(shdr_fragment_.data());
  return true;
} 
```

这两个表大家要牢记，elf 文件头包含了这两个表的信息。

### [](#重要的事情说三遍：)重要的事情说三遍：

#### ELF 文件头包含了 PHT,SHT 这两个表的信息

#### ELF 文件头包含了 PHT,SHT 这两个表的信息

#### ELF 文件头包含了 PHT,SHT 这两个表的信息

![](https://bbs.pediy.com/upload/attach/202105/855618_PUBGXDZT5C8YUNE.png)

通过程序头表拿到动态节区
------------

先啰嗦一下_表_这个关键字：elf 文件中，一个区域存储了很多个一摸一样的结构体，这个区域就可以视作表。比如 got 表，符号表，字符串表等等。这些表后边也会提到。

 

程序头表上面已经从 ELF 文件头信息中拿到了。通过循环遍历这个表，可以找到一个 PT_DYNAMIC 类型的程序头，这个程序头中存储了动态表的偏移大小等信息。我把 demo 中的 so 库用 010editor 这个工具解析出来贴了个图  
![](https://bbs.pediy.com/upload/attach/202105/855618_MAYN8NJM4QQ3SZJ.png)

 

**大家可能有疑问，程序头表中有很多个程序头，为什么仅仅要拿到 PT_DYNAMIC 这个程序头，拿到动态表呢？**

 

其实程序头表中的信息都很重要，比如图中的前两个程序头 (loadable segment)，这两个程序头包含了一些装载信息。

 

linker 会用到装载信息。比如 so 库装载到内存中需要多大空间，内存中的属性 (可读可写可执行等)。

 

既然提到了装载，那就简单说一下装载和链接吧。

### [](#装载：)装载：

so 库包含了程序运行需要用的数据以及指令，这些数据以及指令需要加载到内存中，加载到内存的过程就可以简单理解为装载。

### [](#链接：)链接：

so 库一般会使用到外部的一些符号，比如 open,close,mmap,strlen 等导入符号，这些符号的真实地址不在本 so 库，可能在系统的 libc 库等等。那么本 so 库要调用 open 这些函数的话，肯定需要一些特殊的方式拿到其真实地址。这些真实地址其实保存在 so 库的 got 表中。linker 将 so 库装载后, 会将这些符号进行重新定位，重新定位简称为重定位。其实重定位不仅仅包含外部的一些符号，还包含本 so 库的一些符号。重定位修复的整个过程可以理解为链接。

 

上边提到了前两个程序头 (loadable_segment) 与装载有关系。那么链接过程需要用的重定位信息在哪里呢？**寻找这个答案，就需要 PT_DYNAMIC 这个程序头**, 程序头有一个字端为 p_vaddr，值为 0x2D28，这个地址就是**动态节区，因为这个区域又是数组，后边就把这个节区称为动态表**。

dynamic 表的内容 (用 IDA 工具看看 0x2D28 这个地方)
-------------------------------------

![](https://bbs.pediy.com/upload/attach/202105/855618_2Q72YRG2T5FPGAF.png)

 

动态表中的 DT_JMPREL 这个结构体，包含了重定位表的信息，继续用 ida 工具查看 0x9f8-> 重定位表

重定位表 (地址 0x9f8)
---------------

![](https://bbs.pediy.com/upload/attach/202105/855618_RH5ZD7XE7J7EYDT.png)

 

重定位表包含了需要重定位的函数信息，包括__open_2 函数。

 

前边提到了 linker 对 so 库链接的过程会进行符号重定位，重定位的信息就在重定位表中。

 

继续以__open_2 这个重定位信息为例

 

Elf64_Rela<0x2F90, 0xc00000402, 0>, IDA 已经解析出来了，这个结构体中 0x2f90 是个偏移，指向的地址就是 got 表, 0xc00000402 这个数据其实是一个 info，会解析出符号索引，通过这个索引拿到符号名 (__open_2)，也可以拿到这个重定位信息类型。

```
ElfW(Word) type = ELFW(R_TYPE)(rel->r_info);//符号的类型
ElfW(Word) sym = ELFW(R_SYM)(rel->r_info);//符号的索引

```

ELFE(R_TYPE) 与 ELFW(R_SYM) 宏定义在 libc/kernel/uapi/linux/elf.h 中

```
#if defined(__LP64__)
#define ELFW(what) ELF64_ ## what
#else
#define ELFW(what) ELF32_ ## what
#endif
 
#define ELF32_R_SYM(x) ((x) >> 8)
#define ELF32_R_TYPE(x) ((x) & 0xff)
#define ELF64_R_SYM(i) ((i) >> 32)
#define ELF64_R_TYPE(i) ((i) & 0xffffffff)

```

刚才提到的 0x2f90 这个偏移也贴个图吧  
![](https://bbs.pediy.com/upload/attach/202105/855618_BNAG89AZH7FEKG5.png)

 

linker 重定位修复的数据就包括 got 表，比如 0x2f90 这个地方，使用 8 个字节存放了__open_2 函数的真实地址 (64 位系统需要 8 个字节存放一个地址，这个地址的数据类型简单理解为 void*)。

 

关键的地方就在这里，既然 linker 通过重定位，将 0x2f90 这个地方的数据改了。那我们也仿照 linker 搞一个重定位的修复，将这个地方的数据改为我们的函数地址，是不是就可以实现 hook 了。答案是 yes。

### 重要的事情说三遍

### [](#仿照linker源码搞一个重定位修复，实现hook)仿照 linker 源码搞一个重定位修复，实现 hook

### [](#仿照linker源码搞一个重定位修复，实现hook)仿照 linker 源码搞一个重定位修复，实现 hook

### [](#仿照linker源码搞一个重定位修复，实现hook)仿照 linker 源码搞一个重定位修复，实现 hook

[](#再用流程图总结一下got表hook的关键，如何找到got表对应的地址)再用流程图总结一下 got 表 hook 的关键，如何找到 got 表对应的地址
-------------------------------------------------------------------------------

![](https://bbs.pediy.com/upload/attach/202105/855618_D7FJCZY6CFYZW93.png)

如何实现运行时 got 表 hook
------------------

上述文件格式的分析，都是基于 so 库的静态分析。所有的偏移地址，都是一个相对地址，我们只要拿到 so 库在内存中加载的基址加上这个偏移，就可以了。

### 基址如何获取:

#### 1. 通过读取 / proc/self/maps 文件，遍历后拿到 base

具体的代码百度一大堆，大家自行实现。这里简单的贴个图，看看 maps 文件

```
admin@C02D7132MD6R bionic % adb shell
angler:/ $ ps |grep superhook
u0_a69    7680  540   1620436 71348 SyS_epoll_ 0000000000 S com.example.superhook
angler:/ $ su
angler:/ # cat /proc/7680/maps
12c00000-12e08000 rw-p 00000000 00:04 16462                              /dev/ashmem/dalvik-main space (deleted)
12e08000-13008000 rw-p 00208000 00:04 16462                              /dev/ashmem/dalvik-main space (deleted)
13008000-1ec00000 ---p 00408000 00:04 16462                              /dev/ashmem/dalvik-main space (deleted)
32c00000-32c01000 rw-p 00000000 00:04 16463                              /dev/ashmem/dalvik-main space 1 (deleted)
32c01000-3ec00000 ---p 00001000 00:04 16463                              /dev/ashmem/dalvik-main space 1 (deleted)
6f7b1000-6fa50000 rw-p 00000000 fd:00 278001                             /data/dalvik-cache/arm64/system@framework@boot.art
6fa50000-6fbcc000 rw-p 00000000 fd:00 278002                             /data/dalvik-cache/arm64/system@framework@boot-core-libart.art
6fbcc000-6fc01000 rw-p 00000000 fd:00 278003                             /data/dalvik-cache/arm64/system@framework@boot-conscrypt.art
6fc01000-6fc3a000 rw-p 00000000 fd:00 278004                             /data/dalvik-cache/arm64/system@framework@boot-okhttp.art
6fc3a000-6fc3d000 rw-p 00000000 fd:00 278005                             /data/dalvik-cache/arm64/system@framework@boot-core-junit.art
6fc3d000-6fc83000 rw-p 00000000 fd:00 278006                             /data/dalvik-cache/arm64/system@framework@boot-bouncycastle.art
6fc83000-6fcc5000 rw-p 00000000 fd:00 278007                             /data/dalvik-cache/arm64/system@framework@boot-ext.art
6fcc5000-70484000 rw-p 00000000 fd:00 278008                             /data/dalvik-cache/arm64/system@framework@boot-framework.art
70484000-70516000 rw-p 00000000 fd:00 278009                             /data/dalvik-cache/arm64/system@framework@boot-telephony-common.art
70516000-7051e000 rw-p 00000000 fd:00 278010                             /data/dalvik-cache/arm64/system@framework@boot-voip-common.art
7051e000-7052a000 rw-p 00000000 fd:00 278011                             /data/dalvik-cache/arm64/system@framework@boot-ims-common.art
7052a000-7054d000 rw-p 00000000 fd:00 278012                             /data/dalvik-cache/arm64/system@framework@boot-apache-xml.art
7054d000-70574000 rw-p 00000000 fd:00 278013                             /data/dalvik-cache/arm64/system@framework@boot-org.apache.http.legacy.boot.art
 
77b4a3c000-77b4a59000 r--s 00000000 103:0b 837                           /system/fonts/NotoNaskhArabicUI-Regular.ttf
77b4a59000-77b4aaa000 r--s 00000000 103:0b 949                           /system/fonts/Roboto-BoldItalic.ttf
77b4aaa000-77b4af5000 r--s 00000000 103:0b 948                           /system/fonts/Roboto-Bold.ttf
77b4af5000-77b4b46000 r--s 00000000 103:0b 947                           /system/fonts/Roboto-BlackItalic.ttf
77b4b46000-77b4b91000 r--s 00000000 103:0b 946                           /system/fonts/Roboto-Black.ttf
77b4b91000-77b4be2000 r--s 00000000 103:0b 954                           /system/fonts/Roboto-MediumItalic.ttf
77b4be2000-77b4c2d000 r--s 00000000 103:0b 953                           /system/fonts/Roboto-Medium.ttf
77b4c2d000-77b4c7e000 r--s 00000000 103:0b 950                           /system/fonts/Roboto-Italic.ttf
77b4c7e000-77b4cc9000 r--s 00000000 103:0b 955                           /system/fonts/Roboto-Regular.ttf
77b4cc9000-77b61f6000 r--s 00000000 103:0b 2205                          /system/usr/icu/icudt56l.dat
77b61f6000-77b61f8000 r-xp 00000000 fd:00 531594                         /data/app/com.example.superhook-1/lib/arm64/libtestLib.so
77b61f8000-77b61f9000 r--p 00001000 fd:00 531594                         /data/app/com.example.superhook-1/lib/arm64/libtestLib.so
77b61f9000-77b61fa000 rw-p 00002000 fd:00 531594                         /data/app/com.example.superhook-1/lib/arm64/libtestLib.so
77b6200000-77b6400000 rw-p 00000000 00:00 0                              [anon:libc_malloc]
77b6400000-77b6600000 rw-p 00000000 00:00 0                              [anon:libc_malloc]
77b6600000-77b6602000 r--s 00000000 103:0b 910                           /system/fonts/NotoSansRejang-Regular.ttf
77b6602000-77b660b000 r--s 00000000 103:0b 883                           /system/fonts/NotoSansKhmerUI-Bold.ttf
77b660b000-77b6619000 r--s 00000000 103:0b 894                           /system/fonts/NotoSansMalayalamUI-Bold.ttf
77b6619000-77b666b000 r--s 00000000 103:0b 952                           /system/fonts/Roboto-LightItalic.ttf
77b666b000-77b66b7000 r--s 00000000 103:0b 951                           /system/fonts/Roboto-Light.ttf
77b66b7000-77b66b8000 r--p 00000000 00:00 0                              [anon:linker_alloc]
77b66b8000-77b66d8000 rw-p 00000000 00:04 16053                          /dev/ashmem/dalvik-CompilerMetadata (deleted)
77b66d8000-77b66f8000 rw-p 00000000 00:04 16052                          /dev/ashmem/dalvik-CompilerMetadata (deleted)

```

比如要 hook 的 libtestLib.so 库

 

base 地址为 77b61f6000，**open_2 在 got 表的偏移地址为 0x2F90，我们通过将 base + offset 这个内存块的数据改为我们的函数地址就可以实现__**open_2 的 hook 了。

```
77b61f6000-77b61f8000 r-xp 00000000 fd:00 531594                         /data/app/com.example.superhook-1/lib/arm64/libtestLib.so
77b61f8000-77b61f9000 r--p 00001000 fd:00 531594                         /data/app/com.example.superhook-1/lib/arm64/libtestLib.so
77b61f9000-77b61fa000 rw-p 00002000 fd:00 531594                         /data/app/com.example.superhook-1/lib/arm64/libtestLib.so

```

#### 2.targetsdk <= 23，通过 dlopen 调用返回 soinfo, 然后通过 soinfo 拿到 base

为什么是 targetsdk<= 23，继续查看 linker 的源码

 

bionic/linker/linker.cpp

```
void* soinfo::to_handle() {
  if (get_application_target_sdk_version() <= 23 || !has_min_version(3)) {
    return this;
  }
 
  return reinterpret_cast(get_handle());
} 
```

bionic/linker/linker.cpp

```
void* do_dlopen(const char* name, int flags, const android_dlextinfo* extinfo,
                  void* caller_addr) {
  soinfo* const caller = find_containing_library(caller_addr);
 
  if ((flags & ~(RTLD_NOW|RTLD_LAZY|RTLD_LOCAL|RTLD_GLOBAL|RTLD_NODELETE|RTLD_NOLOAD)) != 0) {
    DL_ERR("invalid flags to dlopen: %x", flags);
    return nullptr;
  }
 
  android_namespace_t* ns = get_caller_namespace(caller);
  ...
  ...
  ...
  ProtectedDataGuard guard;
  soinfo* si = find_library(ns, translated_name, flags, extinfo, caller);
  if (si != nullptr) {
    si->call_constructors();
    return si->to_handle();
  }
 
  return nullptr;
}

```

dlopen 函数最终会调用到 linker 中的 do_dlopen，do_dlopen 的返回值为 soinfo->to_handle()

 

void* soinfo::to_handle() 这个函数中明确了 targetsdk <= 23 返回 this(即 so info)，否则会通过 get_handle 返回一个随机值。

 

来看看 soinfo 结构体:

 

省略了很多，可以查看源码 (bionic/linker/linker.h)

```
struct soinfo {
...
...
...
 public:
  const ElfW(Phdr)* phdr;
  size_t phnum;
  ElfW(Addr) entry;
  ElfW(Addr) base;
  size_t size;
 
#if defined(__work_around_b_24465209__)
  uint32_t unused1;  // DO NOT USE, maintained for compatibility.
#endif
 
  ElfW(Dyn)* dynamic;
 
#if defined(__work_around_b_24465209__)
  uint32_t unused2; // DO NOT USE, maintained for compatibility
  uint32_t unused3; // DO NOT USE, maintained for compatibility
#endif
 
  soinfo* next;
 private:
  uint32_t flags_;
 
  const char* strtab_;
  ElfW(Sym)* symtab_;
  ...
  ...
#if defined(USE_RELA)
  ElfW(Rela)* plt_rela_;//重定位表（函数）
  size_t plt_rela_count_;
 
  ElfW(Rela)* rela_;//重定位表（数据）
  size_t rela_count_;
#else
  ElfW(Rel)* plt_rel_;
  size_t plt_rel_count_;
 
  ElfW(Rel)* rel_;
  size_t rel_count_;
#endif

```

soinfo 中可以拿到 phdr*(程序头表), phnum(程序头个数), base(装载基址), size(装载所需空间) 等等。

### 移植 linker 源码，仿照重定位过程修复 got 表数据:

linker 源码文件结构图，便于大家理解关键文件  
![](https://bbs.pediy.com/upload/attach/202105/855618_YMG3MYM6DETUSM3.png)

 

目前只简单实现了 arch64 的 got 表重定位 hook。下面是一部分 demo 源码

```
bool startRelocateHook(const char* libpath, hook_entry* entries, bool isExport) {
    if (isExport) {
        LOGE("startHook failed! not support export-symbols Hook");
        return 0;
    } else {
        return relocateHook(libpath, entries);
    }
}
 
static int __open_2_hook(const char* path, int flags) {
    LOGD("__open_2_hook %s", path);
    return open(path, flags);
}
 
static void testHookOpen(JNIEnv* env, jclass clazz, jstring jPath) {
 
    const char* cPath = env->GetStringUTFChars(jPath, JNI_FALSE);
    hook_entry entry =  {
            "__open_2",
            (void*)__open_2_hook,
            nullptr
    };
    startRelocateHook(cPath, &entry, false);
    env->ReleaseStringUTFChars(jPath, cPath);
 
}

```

```
bool relocateHook(const char* libpath, hook_entry* entries) {
    ElfSoinfo* elfSoinfo = ElfSoinfo::create(libpath);
    elfSoinfo->relocate(entries, false);
    return false;
}

```

```
bool ElfSoinfo::relocate(hook_entry* entries, bool exported) {
#if defined(USE_RELA)
    if (soinfoCompat_->rela_ != nullptr) {
        LOGD("[ relocating %s ]", get_realpath());
        relocateInner(plain_reloc_iterator(soinfoCompat_->rela_,
                soinfoCompat_->rela_count_), entries, exported);
 
    }
    if (soinfoCompat_->plt_rela_ != nullptr) {
        LOGD("[ relocating %s plt ]", get_realpath());
        relocateInner(plain_reloc_iterator(soinfoCompat_->plt_rela_,
                soinfoCompat_->plt_rela_count_), entries, exported);
    }
#else
    if (soinfoCompat_->rel_ != nullptr) {
        LOGD("[ relocating %s ]", get_realpath());
        relocateInner(plain_reloc_iterator(soinfoCompat_->rel_,
                soinfoCompat_->rel_count_), entries, exported);
    }
    if (soinfoCompat_->plt_rel_ != nullptr) {
        LOGD("[ relocating %s plt ]", get_realpath());
        relocateInner(plain_reloc_iterator(soinfoCompat_->plt_rel_,
                soinfoCompat_->plt_rel_count_), entries, exported);
    }
#endif
  return false;
}
 
 
bool ElfSoinfo::relocateInner(plain_reloc_iterator&& rel_iterator, hook_entry* entries, bool exported) {
    for (size_t idx = 0; rel_iterator.has_next(); ++idx) {
        const auto rel = rel_iterator.next();
        if (rel == nullptr) {
            return false;
        }
 
        ElfW(Word) type = ELFW(R_TYPE)(rel->r_info);
        ElfW(Word) sym = ELFW(R_SYM)(rel->r_info);
 
        ElfW(Addr) reloc = static_cast(rel->r_offset + soinfoCompat_->load_bias);
        ElfW(Addr) sym_addr = 0;
        const char *sym_name = nullptr;
        ElfW(Addr) addend = get_addend(rel, reloc);
 
        LOGD("Processing \"%s\" relocation at index %zd addend: %16p", get_realpath(), idx, addend);
        if (type == R_GENERIC_NONE) {
            continue;
        }
 
        const ElfW(Sym) *s = nullptr;
 
        if (sym != 0) {
            sym_name = get_string(soinfoCompat_->symtab_[sym].st_name);
            s = &soinfoCompat_->symtab_[sym];
            LOGD("func(%s@%p)", sym_name, (void *) reloc);
            if (s->st_shndx == SHN_UNDEF && sym_name && strcmp(sym_name, entries->symbol) == 0) {
                switch (type) {
                    case R_GENERIC_JUMP_SLOT: {
                        ElfW(Addr) page_start = PAGE_START(reloc);
                        mprotect((ElfW(Addr) *) page_start, PAGE_SIZE, PROT_WRITE | PROT_READ);
                        LOGD("RELO JMP_SLOT %16p <- %16p %s", reinterpret_cast(reloc),
                             entries->hook_sym, sym_name);
                        if (entries->orig_sym) {
                            *entries->orig_sym = (ElfW(Addr)*) reloc;
                        }
                        *((ElfW(Addr) *) reloc) = (ElfW(Addr)) entries->hook_sym + addend;
                        mprotect((ElfW(Addr) *) page_start, PAGE_SIZE, PROT_READ);
                        __clear_cache((void*) page_start, (void*)PAGE_END(reloc));
                        break;
                    }
                    default:
                        LOGE("currently only support arch64 got hook!");
                        break;
                }
            }
        }
 
#if defined(DEBUG) && 0
 
        switch (type) {
            case R_GENERIC_JUMP_SLOT:
                LOGD("RELO JMP_SLOT %16p <- %16p %s\n",
                           reinterpret_cast(reloc),
                           reinterpret_cast(sym_addr + addend), sym_name);
 
                break;
            case R_GENERIC_GLOB_DAT:
                LOGD("RELO GLOB_DAT %16p <- %16p %s\n",
                           reinterpret_cast(reloc),
                           reinterpret_cast(sym_addr + addend), sym_name);
                break;
            case R_GENERIC_RELATIVE:
                LOGD("RELO RELATIVE %16p <- %16p\n",
                           reinterpret_cast(reloc),
                           reinterpret_cast(soinfoCompat_->load_bias + addend));
                break;
            case R_GENERIC_IRELATIVE:
                LOGD("RELO IRELATIVE %16p <- %16p\n",
                           reinterpret_cast(reloc),
                           reinterpret_cast(soinfoCompat_->load_bias + addend));
                break;
 
#if defined(__aarch64__)
            case R_AARCH64_ABS64:
                LOGD("RELO ABS64 %16llx <- %16llx %s\n",
                           reloc, sym_addr + addend, sym_name);
                break;
            case R_AARCH64_ABS32:
                LOGD("RELO ABS32 %16llx <- %16llx %s\n",
                           reloc, sym_addr + addend, sym_name);
                break;
            case R_AARCH64_ABS16:
                LOGD("RELO ABS16 %16llx <- %16llx %s\n",
                           reloc, sym_addr + addend, sym_name);
                break;
            case R_AARCH64_PREL64:
                LOGD("RELO REL64 %16llx <- %16llx - %16llx %s\n",
                           reloc, sym_addr + addend, rel->r_offset, sym_name);
                break;
            case R_AARCH64_PREL32:
                LOGD("RELO REL32 %16llx <- %16llx - %16llx %s\n",
                           reloc, sym_addr + addend, rel->r_offset, sym_name);
                break;
            case R_AARCH64_PREL16:
                LOGD("RELO REL16 %16llx <- %16llx - %16llx %s\n",
                           reloc, sym_addr + addend, rel->r_offset, sym_name);
                break;
 
            case R_AARCH64_COPY:
                LOGE("%s R_AARCH64_COPY relocations are not supported", get_realpath());
            case R_AARCH64_TLS_TPREL64:
                LOGD("RELO TLS_TPREL64 *** %16llx <- %16llx - %16llx\n",
                           reloc, (sym_addr + addend), rel->r_offset);
                break;
            case R_AARCH64_TLS_DTPREL32:
                LOGD("RELO TLS_DTPREL32 *** %16llx <- %16llx - %16llx\n",
                           reloc, (sym_addr + addend), rel->r_offset);
                break;
#endif
            default:
                LOGE("unknown reloc type %d @ %p (%zu)", type, rel, idx);
                return false;
        }
 
#endif
    }
    return true;
} 
```

通过 android studio 断点调试验证是否实现 hook  
![](https://bbs.pediy.com/upload/attach/202105/855618_9SGKEWRK2HYRTHP.png)

 

__open_2_hook 确实走到了，图中也可以明确的看到调用栈

 

另外__open_2_hook 函数中打印了文件路径

```
static int __open_2_hook(const char* path, int flags) {
    LOGD("__open_2_hook %s", path);
    return open(path, flags);
}

```

__open_2_hook 函数中打印的文件路径，其实是 libtestLib.so 访问的。运行程序后发现确实打印了相关日志：

```
05-23 14:31:07.413 10659 10659 D superhook: __open_2_hook /data/app/com.example.superhook-2/base.apk
05-23 14:31:07.413 10659 10659 D superhook-test: open /data/app/com.example.superhook-2/base.apk fd: 52

```

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 1 小时前 被 whulzz 编辑 ，原因： 添加丢失图片以及附加源码地址