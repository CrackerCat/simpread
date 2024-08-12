> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282859.htm)

> 初探某加固方案

初探某加固方案

发表于: 6 小时前 232

### 初探某加固方案

 [![](http://passport.kanxue.com/upload/avatar/679/979679.png?1693204967)](user-home-979679.htm) [Shangwendada](user-home-979679.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 6 小时前  232

初探某加固方案
=======

APK 界面：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723360034175-1.png)

特征分析
----

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723360286852.png)

观察 attachBaseContext 类：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723360508386.png)

值得注意的是 attachBaseContext 与 onCreate 类都属于 Application 的回调方法，会进行一些初始化操作。

这里我们观察他在加载字符串的时候都使用了 a.a 方法，那么显然 a.a 方法就是就是一个解混淆的方法咯，我们还可以 hook 住看一下都做了些什么

```
let a = Java.use("com.tianyu.util.a");
a["a"].overload('java.lang.String').implementation = function (str) {
    console.log(`a.a is called: str=${str}`);
    let result = this["a"](str);
    console.log(`a.a result=${result}`);
    return result;
};
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723360976232.png)

在 Stub 最开始我们可以找到这个字符串：

private static String b = "libjiagu";  
对他交叉引用看一看

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723361041790.png)

可以发现加固保会判断手机的架构，针对不同的架构加载不同的 Native 文件。

Native 层分析
----------

显然我们可以发现该加固是通过 Native 层去释放 Dex 文件的，因此，我们主要分析的还是 Native 层的内容。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723361548930.png)

这里我们主要分析 arm64:

### 壳 ELF 导入导出表修复：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723361603747.png)

可以发现该加固对于最外面的 ELF 做了处理，抹除了 SO 的导出表，如果没有导入导出表的话，这个 ELF 文件是如何运行的呢，那么不难发现其实加固保应该是使用了自己定义的链接器，在装载内存的时候才做相应的链接操作。

首先我们先 hook dlopen 来查看 APK 加载了哪些 so 文件：

```
function hook_dlopne() {
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            console.log("Load -> ", args[0].readCString());
        }, onLeave: function () {
 
        }
    })
}
 
setImmediate(hook_dlopne);
```

正常来说，我们如果 Hook dlopen 的话，在安卓 7.0 之后我们需要 hook 的值则为 "android_dlopen_ext"：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723362609260.png)

我们可以发现加载了 libjiagu_64.so，接下来就是想办法把它 dump 下来了，**首先我们启动 APP**，使用 frida -U -F -l Hook.js  
参数注入如下脚本

```
function dump_so() {
    var soName = "libjiagu_64.so";
    var libSo = Process.getModuleByName(soName);
    var save_path = "/data/data/com.swdd.txjgtest/" + libSo.name + "_Dump";
    console.log("[Base]->", libSo.base);
    console.log("[Size]->", ptr(libSo.size));
    var handle = new File(save_path, "wb");
    Memory.protect(ptr(libSo.base), libSo.size, 'rwx');
    var Buffer = libSo.base.readByteArray(libSo.size);
    handle.write(Buffer);
    handle.flush();
    handle.close();
    console.log("[DumpPath->]", save_path);
 
}
setImmediate(dump_so);
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723363653409.png)

```
[Base]-> 0x78c1443000
[Size]-> 0x27d000
[DumpPath->] /data/data/com.swdd.txjgtest/libjiagu_64.so_Dump
```

这三个参数很重要，等下修复 SO 的时候需要使用。

SoFixer:[https://github.com/F8LEFT/SoFixer](https://github.com/F8LEFT/SoFixer)

接下来我们需要使用 SoFixer 修复我们 dump 下来的 So 文件：

```
.\SoFixer-Windows-64.exe -s .\libjiagu_64.so_Dump -o .\libjiagu_64.so_Fix -m 0x78c1443000 -d
```

-m 是刚刚我们脚本输出的偏移地址

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723364239423.png)

### 壳 ELF 分析：

接下来我们要做的就是分析 ELF 的逻辑了：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723364415785.png)

刚开始拿到这个 ELF 我们还无从下手，但是根据加固思路，我们可以先 hook open 函数，查看读取了哪些文件。

```
function hookOpen() {
    var openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    Interceptor.replace(openPtr, new NativeCallback(function (fileNamePtr, flag) {
        var fileName = fileNamePtr.readCString();
        if (fileName.indexOf('dex') != -1) {
            console.log("[Open]-> ", fileName);
        }
        return open(fileNamePtr, flag);
 
    }, 'int', ['pointer', 'int']))
}
function hook_dlopne() {
 
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            var loadFileName = args[0].readCString();
          //  console.log("Load -> ", loadFileName);
            if (loadFileName.indexOf('libjiagu') != -1) {
                this.is_can_hook = true;
            }
        }, onLeave: function () {
            if (this.is_can_hook) {
                hookOpen();
            }
        }
    })
}
 
 
setImmediate(hook_dlopne);
```

得到输出如下：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723385541334.png)

我们发现，非常奇怪，居然没有打开与我们程序相关的 dex 文件，我们取消对 dex 的过滤，查看一下都打开了一些什么内容。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723385802548.png)

多次的打开了 maps 文件，那么我们知道该文件包含了进程的内存映射信息，程序频繁读取是为了什么呢，其实猜测就是为了隐藏打开 dex 的操作，那么我们只需要重定向一下 maps 就可以了，hook open 将打开 open 时如果存在扫描 maps，就定向到自己的 fakemaps。

```
function hookOpen() {
    var FakeMaps = "/data/data/com.swdd.txjgtest/maps";
    var openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var readPtr = Module.findExportByName("libc.so", "read");
     
    Interceptor.replace(openPtr, new NativeCallback(function (fileNamePtr, flag) {
        var FD = open(fileNamePtr, flag);
        var fileName = fileNamePtr.readCString();
        
        if (fileName.indexOf("maps") >= 0) {
            console.warn("[Warning]->mapsRedirect Success");
            var filename = Memory.allocUtf8String(FakeMaps);
            return open(filename, flag);
        }
        if (fileName.indexOf('dex') != -1) {
            console.log("[OpenDex]-> ", fileName);
        }
        return FD;
 
    }, 'int', ['pointer', 'int']))
}
function hook_dlopne() {
 
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            var loadFileName = args[0].readCString();
          //  console.log("Load -> ", loadFileName);
            if (loadFileName.indexOf('libjiagu') != -1) {
                this.is_can_hook = true;
            }
        }, onLeave: function () {
            if (this.is_can_hook) {
                hookOpen();
            }
        }
    })
}
 
 
setImmediate(hook_dlopne);
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723387649607.png)

那么我们能够发现确实使用了 open 去打开 classes，并且实锤了是通过处理 maps 隐藏了内存映射，接下来我们就可以通过

```
console.log('RegisterNatives called from:\n' + Thread.backtrace(this.context, Backtracer.FUZZY).map(DebugSymbol.fromAddress).join('\n') + '\n');
```

来打印读取 dex 文件的 Native 地址。

```
function hookOpen() {
    var FakeMaps = "/data/data/com.swdd.txjgtest/maps";
    var openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var readPtr = Module.findExportByName("libc.so", "read");
     
    Interceptor.replace(openPtr, new NativeCallback(function (fileNamePtr, flag) {
        var FD = open(fileNamePtr, flag);
        var fileName = fileNamePtr.readCString();
        
        if (fileName.indexOf("maps") >= 0) {
            console.warn("[Warning]->mapsRedirect Success");
            var filename = Memory.allocUtf8String(FakeMaps);
            return open(filename, flag);
        }
        if (fileName.indexOf('dex') != -1) {
            console.info("[OpenDex]-> ", fileName);
            console.log('RegisterNatives called from:\n' + Thread.backtrace(this.context, Backtracer.FUZZY).map(DebugSymbol.fromAddress).join('\n') + '\n');
        }
        return FD;
 
    }, 'int', ['pointer', 'int']))
}
function hook_dlopne() {
 
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            var loadFileName = args[0].readCString();
          //  console.log("Load -> ", loadFileName);
            if (loadFileName.indexOf('libjiagu') != -1) {
                this.is_can_hook = true;
            }
        }, onLeave: function () {
            if (this.is_can_hook) {
                hookOpen();
            }
        }
    })
}
 
 
setImmediate(hook_dlopne);
```

获取到的输出如下：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723387891885-1.png)

可以发现每次打开 dex 的调用栈基本一致，我们在 IDA 中查看这个地址。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723388155090.png)

居然都未识别。

翻了一下这段数据，翻到头的时候可以发现

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723388670457.png)

在这一段有被引用，我们跟过去看一看

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723388700096.png)

后缀被加了. so，然后分段加载了一些东西。这个时候基本可以猜测是在 linker 另一个 so 了。

### 主 ELF 解密

在自实现 linker 的时候，完成 linker 之后肯定是需要用 dlopen 去加载这个 so 的，那么我们 hook 一下 dlopen 验证一下我们的猜想。

```
function hook_dlopne() {
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            console.warn("[android_dlopen_ext] -> ", args[0].readCString());
        }, onLeave: function () {
 
        }
    })
}
 
function hook_dlopne2() {
    Interceptor.attach(Module.findExportByName("libdl.so", "dlopen"), {
        onEnter: function (args) {
            console.log("[dlopen] -> ", args[0].readCString());
        }, onLeave: function () {
 
        }
    })
}
setImmediate(hook_dlopne2);
setImmediate(hook_dlopne);
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723389034476.png)

流程说明了一切，直接实锤了自定义 linker 加固 so，那接下来我们应该如何做呢，首先需要把另一个 ELF 给分离出来。

自定义 linker SO 加固的大部分思路其实就是分离出 program header 等内容进行单独加密，然后在 link 的时候补充 soinfo。

我们使用 010 在之前 Fix 后的 so 里面查找 ELF

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723389441925.png)

在 e8000 处我们发现了 ELF 头，我们使用 python 脚本将其分离出来：

```
with open('libjiagu_64.so_Dump','rb') as f:
    s=f.read()
with open('libjiagu_64.so','wb') as f:
    f.write(s[0xe8000::])
```

但是 program header 已经被加密了，那么接下来我们需要做的就是找到在哪儿解密的。

这里推荐 oacia 大佬的项目：[https://github.com/oacia/stalker_trace_so](https://github.com/oacia/stalker_trace_so)

但是在使用大佬的项目的时候，我输出的 js 文件显示的内容都是乱码，所以对源码做了一些修改

改动点如下：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723390039525.png)

接下来利用 stalker_trace_so 来分析程序执行流：  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723390119224.png)

注意此处需要改为 libjiagu_64.so

接下来我们拿到如下执行流：

```
call1:JNI_OnLoad
call2:j_interpreter_wrap_int64_t
call3:interpreter_wrap_int64_t
call4:_Znwm
call5:sub_13364
call6:_Znam
call7:sub_10C8C
call8:memset
call9:sub_9988
call10:sub_DE4C
call11:calloc
call12:malloc
call13:free
call14:sub_E0B4
call15:_ZdaPv
call16:sub_C3B8
call17:sub_C870
call18:sub_9538
call19:sub_9514
call20:sub_C9E0
call21:sub_C5A4
call22:sub_9674
call23:sub_15654
call24:sub_15DCC
call25:sub_15E98
call26:sub_159CC
call27:sub_1668C
call28:sub_15A4C
call29:sub_15728
call30:sub_15694
call31:sub_94B0
call32:sub_C8C8
call33:sub_CAC4
call34:sub_C810
call35:sub_906C
call36:dladdr
call37:strstr
call38:setenv
call39:_Z9__arm_a_1P7_JavaVMP7_JNIEnvPvRi
call40:sub_9A08
call41:sub_954C
call42:sub_103D0
call43:j__ZdlPv_1
call44:_ZdlPv
call45:sub_9290
call46:sub_7BAC
call47:strncpy
call48:sub_5994
call49:sub_5DF8
call50:sub_4570
call51:sub_59DC
call52:_ZN9__arm_c_19__arm_c_0Ev
call53:sub_9F60
call54:sub_957C
call55:sub_94F4
call56:sub_CC5C
call57:sub_5D38
call58:sub_5E44
call59:memcpy
call60:sub_5F4C
call61:sub_583C
call62:j__ZdlPv_3
call63:j__ZdlPv_2
call64:j__ZdlPv_0
call65:sub_9F14
call66:sub_9640
call67:sub_5894
call68:sub_58EC
call69:sub_9B90
call70:sub_2F54
call71:uncompress
call72:sub_C92C
call73:sub_440C
call74:sub_4BFC
call75:sub_4C74
call76:sub_5304
call77:sub_4E4C
call78:sub_5008
call79:mprotect
call80:strlen
call81:sub_3674
call82:dlopen
call83:sub_4340
call84:sub_3A28
call85:sub_3BDC
call86:sub_2F8C
call87:dlsym
call88:strcmp
call89:sub_5668
call90:sub_4C40
call91:sub_5BF0
call92:sub_7CDC
call93:sub_468C
call94:sub_7E08
call95:sub_86FC
call96:sub_8A84
call97:sub_7FDC
call98:interpreter_wrap_int64_t_bridge
call99:sub_9910
call100:sub_15944
call101:puts
```

现在我们知道了控制流，然而还不够，因为自定义 linker 加固 so，最后还是需要 dlopen 去手动加载的，那么我们对导入表中的 dlopen 进行交叉。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723390870918.png)

发现只有一次掉用，我们跟踪过去看一看。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723390896453.png)

全是 Switch case：  
[http://androidxref.com/9.0.0_r3/xref/bionic/linker/linker.cpp](http://androidxref.com/9.0.0_r3/xref/bionic/linker/linker.cpp)

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723391018779.png)

可以看看 linker 源码中的预链接部分，代码如出一折，那么此时我们就可以导入 soinfo 结构体了

在 ida 中依次点击 View->Open subviews->Local Types , 导入结构体（点击 insert）

```
//IMPORTANT
//ELF64 启用该宏
#define __LP64__  1
//ELF32 启用该宏
//#define __work_around_b_24465209__  1
 
/*
//https://android.googlesource.com/platform/bionic/+/master/linker/Android.bp
架构为 32 位 定义__work_around_b_24465209__宏
arch: {
        arm: {cflags: ["-D__work_around_b_24465209__"],},
        x86: {cflags: ["-D__work_around_b_24465209__"],},
    }
*/
 
//android-platform\bionic\libc\include\link.h
#if defined(__LP64__)
#define ElfW(type) Elf64_ ## type
#else
#define ElfW(type) Elf32_ ## type
#endif
 
//android-platform\bionic\linker\linker_common_types.h
// Android uses RELA for LP64.
#if defined(__LP64__)
#define USE_RELA 1
#endif
 
//android-platform\bionic\libc\kernel\uapi\asm-generic\int-ll64.h
//__signed__-->signed
typedef signed char __s8;
typedef unsigned char __u8;
typedef signed short __s16;
typedef unsigned short __u16;
typedef signed int __s32;
typedef unsigned int __u32;
typedef signed long long __s64;
typedef unsigned long long __u64;
 
//A12-src\msm-google\include\uapi\linux\elf.h
/* 32-bit ELF base types. */
typedef __u32   Elf32_Addr;
typedef __u16   Elf32_Half;
typedef __u32   Elf32_Off;
typedef __s32   Elf32_Sword;
typedef __u32   Elf32_Word;
 
/* 64-bit ELF base types. */
typedef __u64   Elf64_Addr;
typedef __u16   Elf64_Half;
typedef __s16   Elf64_SHalf;
typedef __u64   Elf64_Off;
typedef __s32   Elf64_Sword;
typedef __u32   Elf64_Word;
typedef __u64   Elf64_Xword;
typedef __s64   Elf64_Sxword;
 
typedef struct dynamic{
  Elf32_Sword d_tag;
  union{
    Elf32_Sword d_val;
    Elf32_Addr  d_ptr;
  } d_un;
} Elf32_Dyn;
 
typedef struct {
  Elf64_Sxword d_tag;       /* entry tag value */
  union {
    Elf64_Xword d_val;
    Elf64_Addr d_ptr;
  } d_un;
} Elf64_Dyn;
 
typedef struct elf32_rel {
  Elf32_Addr    r_offset;
  Elf32_Word    r_info;
} Elf32_Rel;
 
typedef struct elf64_rel {
  Elf64_Addr r_offset;  /* Location at which to apply the action */
  Elf64_Xword r_info;   /* index and type of relocation */
} Elf64_Rel;
 
typedef struct elf32_rela{
  Elf32_Addr    r_offset;
  Elf32_Word    r_info;
  Elf32_Sword   r_addend;
} Elf32_Rela;
 
typedef struct elf64_rela {
  Elf64_Addr r_offset;  /* Location at which to apply the action */
  Elf64_Xword r_info;   /* index and type of relocation */
  Elf64_Sxword r_addend;    /* Constant addend used to compute value */
} Elf64_Rela;
 
typedef struct elf32_sym{
  Elf32_Word    st_name;
  Elf32_Addr    st_value;
  Elf32_Word    st_size;
  unsigned char st_info;
  unsigned char st_other;
  Elf32_Half    st_shndx;
} Elf32_Sym;
 
typedef struct elf64_sym {
  Elf64_Word st_name;       /* Symbol name, index in string tbl */
  unsigned char st_info;    /* Type and binding attributes */
  unsigned char st_other;   /* No defined meaning, 0 */
  Elf64_Half st_shndx;      /* Associated section index */
  Elf64_Addr st_value;      /* Value of the symbol */
  Elf64_Xword st_size;      /* Associated symbol size */
} Elf64_Sym;
 
 
#define EI_NIDENT   16
 
typedef struct elf32_hdr{
  unsigned char e_ident[EI_NIDENT];
  Elf32_Half    e_type;
  Elf32_Half    e_machine;
  Elf32_Word    e_version;
  Elf32_Addr    e_entry;  /* Entry point */
  Elf32_Off e_phoff;
  Elf32_Off e_shoff;
  Elf32_Word    e_flags;
  Elf32_Half    e_ehsize;
  Elf32_Half    e_phentsize;
  Elf32_Half    e_phnum;
  Elf32_Half    e_shentsize;
  Elf32_Half    e_shnum;
  Elf32_Half    e_shstrndx;
} Elf32_Ehdr;
 
typedef struct elf64_hdr {
  unsigned char e_ident[EI_NIDENT]; /* ELF "magic number" */
  Elf64_Half e_type;
  Elf64_Half e_machine;
  Elf64_Word e_version;
  Elf64_Addr e_entry;       /* Entry point virtual address */
  Elf64_Off e_phoff;        /* Program header table file offset */
  Elf64_Off e_shoff;        /* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;
  Elf64_Half e_phentsize;
  Elf64_Half e_phnum;
  Elf64_Half e_shentsize;
  Elf64_Half e_shnum;
  Elf64_Half e_shstrndx;
} Elf64_Ehdr;
 
/* These constants define the permissions on sections in the program
   header, p_flags. */
#define PF_R        0x4
#define PF_W        0x2
#define PF_X        0x1
 
typedef struct elf32_phdr{
  Elf32_Word    p_type;
  Elf32_Off p_offset;
  Elf32_Addr    p_vaddr;
  Elf32_Addr    p_paddr;
  Elf32_Word    p_filesz;
  Elf32_Word    p_memsz;
  Elf32_Word    p_flags;
  Elf32_Word    p_align;
} Elf32_Phdr;
 
typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;       /* Segment file offset */
  Elf64_Addr p_vaddr;       /* Segment virtual address */
  Elf64_Addr p_paddr;       /* Segment physical address */
  Elf64_Xword p_filesz;     /* Segment size in file */
  Elf64_Xword p_memsz;      /* Segment size in memory */
  Elf64_Xword p_align;      /* Segment alignment, file & memory */
} Elf64_Phdr;
 
typedef struct elf32_shdr {
  Elf32_Word    sh_name;
  Elf32_Word    sh_type;
  Elf32_Word    sh_flags;
  Elf32_Addr    sh_addr;
  Elf32_Off sh_offset;
  Elf32_Word    sh_size;
  Elf32_Word    sh_link;
  Elf32_Word    sh_info;
  Elf32_Word    sh_addralign;
  Elf32_Word    sh_entsize;
} Elf32_Shdr;
 
typedef struct elf64_shdr {
  Elf64_Word sh_name;       /* Section name, index in string tbl */
  Elf64_Word sh_type;       /* Type of section */
  Elf64_Xword sh_flags;     /* Miscellaneous section attributes */
  Elf64_Addr sh_addr;       /* Section virtual addr at execution */
  Elf64_Off sh_offset;      /* Section file offset */
  Elf64_Xword sh_size;      /* Size of section in bytes */
  Elf64_Word sh_link;       /* Index of another section */
  Elf64_Word sh_info;       /* Additional section information */
  Elf64_Xword sh_addralign; /* Section alignment */
  Elf64_Xword sh_entsize;   /* Entry size if section holds table */
} Elf64_Shdr;
 
 
//android-platform\bionic\linker\linker_soinfo.h
typedef void (*linker_dtor_function_t)();
typedef void (*linker_ctor_function_t)(int, char**, char**);
 
#if defined(__work_around_b_24465209__)
#define SOINFO_NAME_LEN 128
#endif
 
struct soinfo {
#if defined(__work_around_b_24465209__)
  char old_name_[SOINFO_NAME_LEN];
#endif
  const ElfW(Phdr)* phdr;
  size_t phnum;
#if defined(__work_around_b_24465209__)
  ElfW(Addr) unused0; // DO NOT USE, maintained for compatibility.
#endif
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
  uint32_t flags_;
 
  const char* strtab_;
  ElfW(Sym)* symtab_;
 
  size_t nbucket_;
  size_t nchain_;
  uint32_t* bucket_;
  uint32_t* chain_;
 
#if !defined(__LP64__)
  ElfW(Addr)** unused4; // DO NOT USE, maintained for compatibility
#endif
 
#if defined(USE_RELA)
  ElfW(Rela)* plt_rela_;
  size_t plt_rela_count_;
 
  ElfW(Rela)* rela_;
  size_t rela_count_;
#else
  ElfW(Rel)* plt_rel_;
  size_t plt_rel_count_;
 
  ElfW(Rel)* rel_;
  size_t rel_count_;
#endif
 
  linker_ctor_function_t* preinit_array_;
  size_t preinit_array_count_;
 
  linker_ctor_function_t* init_array_;
  size_t init_array_count_;
  linker_dtor_function_t* fini_array_;
  size_t fini_array_count_;
 
  linker_ctor_function_t init_func_;
  linker_dtor_function_t fini_func_;
 
/*
#if defined (__arm__)
  // ARM EABI section used for stack unwinding.
  uint32_t* ARM_exidx;
  size_t ARM_exidx_count;
#endif
  size_t ref_count_;
// 怎么找不 link_map 这个类型的声明...
  link_map link_map_head;
 
  bool constructors_called;
 
  // When you read a virtual address from the ELF file, add this
  //value to get the corresponding address in the process' address space.
  ElfW (Addr) load_bias;
 
#if !defined (__LP64__)
  bool has_text_relocations;
#endif
  bool has_DT_SYMBOLIC;
*/
};
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723391273696.png)

可以发现还是不完全，证明这个 soinfo 是被魔改过的

对其交叉，向上查看：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723391649358.png)

下方还调用了一个函数，我们进去看看。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723391703744.png)

发现这里的步长是 0x38

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723391789170.png)

好巧不巧，程序头就是 0x38 大小，那么这个方法肯定是在加载程序头了。

既然需要加载程序头，那么他肯定是需要解密之前被加密的程序头段的  
加载在 sub_5668 调用于 sub_4340 又调用于 sub_440C 最后形成闭环  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723392411052.png)

那我们只需要找 sub_7BAC 到 sub_440C 之间调用的函数就可以了。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723392631511.png)

这个 uncompress 就及其可疑了，这是个解压缩的方法，我们稍微向上找找：

就能发现我们的老熟人了

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723392982581.png)

妥妥一 RC4，但是找不到他的初始化算法，虽然就算没有初始化算法，我们也可以通过 dump 他的 S 盒，来解密，但是总感觉怪怪的，一筹莫展之际，我们向上交叉发现了 loc_571c  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723393162313.png)  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723393168774.png)

未识别居然，我们识别看看

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723393197091.png)

哟，这不是初始化算法么。

我们 hook 他看看密钥

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723394148528.png)

```
function hook_Rc4() {
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x571C), {
        onEnter(args) {
            console.log(hexdump(args[0], {
                offset: 0, length: 0x10, header: true, ansi: true
            }))
        }, onLeave(reval) {
 
        }
 
    });
}
 
function hook_dlopne() {
 
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            var loadFileName = args[0].readCString();
            //  console.log("Load -> ", loadFileName);
            if (loadFileName.indexOf('libjiagu') != -1) {
                this.is_can_hook = true;
            }
        }, onLeave: function () {
            if (this.is_can_hook) {
                hook_Rc4();
 
            }
        }
    })
}
 
 
setImmediate(hook_dlopne);
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723394432407.png)

在最开始找到的这个函数中，我们可以看到加载的地址和文件大小，那么我们直接使用 python 读取这个大小的段进行解密

最开始写一个 python 脚本解密的时候我们出现了如下问题

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723394863937.png)

显然 RC4 解密出问题了，我们再回去看看 rc4 加密部分

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723395064913.png)

对着其一顿还原，发现居然有魔改

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723395103050.png)

但是解密出来依旧不对，我们看看 S 盒是否一致

```
function hook_Rc4_init() {
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x58EC), {
        onEnter(args) {
            console.log(hexdump(args[2], {
                offset: 0, length: 0x100, header: true, ansi: true
            }))
        }, onLeave(reval) {
 
        }
 
    });
}
 
function hook_Rc4_cry() {
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x571C), {
        onEnter(args) {
            console.log(hexdump(args[0], {
                offset: 0, length: 0x10, header: true, ansi: true
            }))
        }, onLeave(reval) {
 
        }
 
    });
}
 
function hook_dlopne() {
 
    Interceptor.attach(Module.findExportByName("libdl.so", "android_dlopen_ext"), {
        onEnter: function (args) {
            var loadFileName = args[0].readCString();
            //  console.log("Load -> ", loadFileName);
            if (loadFileName.indexOf('libjiagu') != -1) {
                this.is_can_hook = true;
            }
        }, onLeave: function () {
            if (this.is_can_hook) {
                hook_Rc4_init();
                hook_Rc4_cry();
            }
        }
    })
}
 
 
setImmediate(hook_dlopne);
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723396172044.png)

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723395236086.png)

显然 Sbox 是不一致的，那么反正有了 Sbox，我们就不需要再去管 init 方法了，直接使用现成的 Sbox 就可以了。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723395352141.png)

还是没能解出来，有点难受了，还有一个细节就是 i , j 来自于 Sbox 的第 257 和 258 位，他们会不会不是 0 呢？我们多 dump 两个看看

果然不是 0！

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723396192798.png)

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723396244985.png)

解密成功

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723396349931.png)

拿到的东西，似乎依旧不是正确的，继续查看调用链，最后在 sub_5304 找到了这样的一个函数

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723396821178.png)

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723396892222.png)

这里其实是一个向量运算，第一个字节为异或的值，后面的四个字节表示异或的大小。

接下来重新用脚本把他们都解密出来：

```
import zlib
import struct
def RC4(data, key):
    for i in range(0,8):
        print(hex(data[i]))
    S = list(range(256))
    j = 0
    out = []
 
    S = [0x76,0xac,0x57,0x5d,0x84,0x1a,0x43,0x9d,0xfb,0x5f,0xf8,0x59,0x35,0x9c,0x05,0x36,0xcd,0xd1,0x01,0xcc,0x39,0x49,0xb6,0x10,0x0e,0x5e,0x2e,0x2a,0x29,0x7f,0x72,0x88,0x9f,0x13,0x2c,0x6f,0x44,0x9b,0x67,0x4a,0xe0,0xee,0x77,0x34,0x97,0x0b,0x68,0x0c,0x4f,0xcf,0x8f,0x95,0x83,0x52,0xef,0x78,0x6a,0xde,0x09,0x1d,0xb5,0x48,0xa8,0xa1,0x46,0x85,0x02,0xe7,0xcb,0x41,0xb3,0x3e,0x71,0xb9,0x3b,0xe4,0x53,0xc9,0x73,0x42,0xe5,0x30,0x25,0x75,0xf9,0xdf,0x14,0x38,0xae,0xd2,0x0d,0x82,0x6c,0x93,0x6e,0xbe,0x5b,0x20,0xf3,0x47,0xd8,0xf1,0x8b,0x64,0xb1,0xab,0xad,0xf6,0xb8,0x7a,0x80,0x4d,0xb7,0x56,0xec,0xb0,0x66,0x18,0xc4,0x92,0x33,0xc8,0x60,0x4e,0x31,0xd9,0x5a,0x03,0xe6,0x15,0xd3,0xa3,0x21,0xa7,0x1c,0xc1,0x26,0x3c,0x1e,0x70,0xbf,0xa2,0xc5,0xc3,0xa0,0xc2,0xc0,0x98,0x28,0x89,0x50,0x4b,0x90,0x6b,0xe1,0x55,0x79,0x7c,0xfd,0xff,0xe3,0xaa,0x2b,0xa4,0xbd,0x62,0x2f,0x16,0xb4,0x7e,0xc6,0xfe,0x63,0xda,0x51,0xd6,0x32,0x3a,0x11,0xc7,0x3f,0x8e,0xd5,0xea,0xa5,0xba,0xca,0xed,0x08,0x22,0x74,0x5c,0x24,0x4c,0x7b,0xbb,0xa9,0x8d,0x96,0x91,0x1b,0xf2,0x17,0x94,0x45,0x19,0xce,0x06,0x8a,0x65,0x37,0x86,0xf5,0x12,0x9a,0x69,0x8c,0x87,0xd4,0xe8,0x6d,0xeb,0x58,0x23,0x00,0x40,0x1f,0xaf,0x99,0xdd,0x04,0x9e,0x7d,0x0a,0xa6,0x81,0xf0,0xf7,0x3d,0xe9,0xdb,0x0f,0xbc,0x27,0xfa,0xe2,0xfc,0xf4,0xb2,0xd0,0xdc,0xd7,0x54,0x07,0x2d,0x61]
    i = 0x3
    j = 0x5
    for ch in data:
        i = (i + 2) % 256
        j = (j + S[i] + 1) % 256
        S[i], S[j] = S[j], S[i]
        out.append(ch ^ S[(S[i] + S[j]) % 256])
 
    return out
 
def RC4decrypt(ciphertext, key):
    return RC4(ciphertext, key)
 
 
wrap_elf_start = 0x2D260
wrap_elf_size = 0xB9956
key = b"vUV4#\x91#SVt"
with open('libjiagu_64.so_Fix','rb') as f:
    wrap_elf = f.read()
 
 
 
# 对密文进行解密
dec_compress_elf = RC4decrypt(wrap_elf[wrap_elf_start:wrap_elf_start+wrap_elf_size], key)
dec_elf = zlib.decompress(bytes(dec_compress_elf[4::]))
with open('wrap_elf','wb') as f:
    f.write(dec_elf)
 
 
class part:
    def __init__(self):
        self.name = ""
        self.value = b''
        self.offset = 0
        self.size = 0
 
 
index = 1
extra_part = [part() for _ in range(7)]
seg = ["a", "b", "c", "d"]
v_xor = dec_elf[0]
for i in range(4):
    size = int.from_bytes(dec_elf[index:index + 4], 'little')
    index += 4
    extra_part[i + 1].name = seg[i]
    extra_part[i + 1].value = bytes(map(lambda x: x ^ v_xor, dec_elf[index:index + size]))
    extra_part[i + 1].size = size
    index += size
for p in extra_part:
    if p.value != b'':
        filename = f"libjiagu.so_{hex(p.size)}_{p.name}"
        print(f"[{p.name}] get {filename}, size: {hex(p.size)}")
        with open(filename, 'wb') as f:
            f.write(p.value)
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723397480329.png)

得到每个段的大小和偏移，还记不记得我们之前讲过程序头是 6 个大小 0x38 的字节组成的，那么计算一波这个 a 正好满足程序头的大小，那么我们使用 010 修补原本程序

然后我们继续观察，.rela.plt，.rela.dyn 储存的内容是要远远大于 dynamic 的，所以我们可以锁定 dynamic 是 d，那么我们根据 program header table 找到 dynamic 的偏移：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723397825264.png)  
CTRL+G 跳转过去

填入 d 的内容

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398022498.png)

接下来：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398046876.png)

根据这个我们就能知道  
0x7 是 dyn 偏移，0x8 是 dyn 大小  
0x17 是 plt 偏移，0x2 是 plt 大小

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398137093.png)

对应的那么 b 就是 plt 了，c 就是 dyn 了，依旧是填入修复：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398306801.png)

至此，主 so 修复完成。

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398397177.png)

### Dex 释放分析

为了方便 hook 和分析，我们需要设置基地址为这个 so 的地址：这里是 0xe8000

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398478986.png)

设置完了之后大家是否还记得最开始我们跟踪 open 的时候那几个函数呢，我们现在继续过去看看。  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723398637773.png)

我们可以观察到在打开 dex 之后会调用 0x136788，调用完他之后调用了 artso 里面的一些擀卷函数 FindClass？？？显然在 0x136788 之后就解密完成了，那么我们跟过去看看  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723399267616.png)

这附近肯定就存在解密解密点了，向下看：

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723400487374.png)  
我们可以找到该方法，通过之前 hook 的方式打印他的调用栈。  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723400556195.png)  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723400579545.png)  
这

之后立马就是加载 findclass 这些，我们直接看看他的参数是什么样的，这里需要注意，之前我们 hook 的是 Android_dlopen_ext，但是由于这个是主 elf 不是使用上面的加载的了，所以我们得改用对 dlopen 做 hook。

hook 一下 0x193868，然后看一下内存

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723401028269.png)

全体起立。

接下来就是把这个内存给 dump 下来了，那么要 dump 多大还不知道，我们看看别的参数。  
![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723401145996.png)

发现 aargs[2] 就是 dex 的大小，那么我们根据这个写脚本就可以了。

```
function travedec() {
    var base = Process.findModuleByName("libjiagu_64.so").base.add(0x193868);
    var fileIndex = 0
    //console.log(base);
    Interceptor.attach(base, {
        onEnter: function (args) {
            console.log(hexdump(args[1], {
                offset: 0, length: 0x30, header: true, ansi: true
            }))
            console.log(args[2]);
 
            try {
                var length = args[2].toInt32();
                var data = Memory.readByteArray(args[1], length);
 
                var filePath = "/data/data/com.swdd.txjgtest/files/" + fileIndex + ".dex";
                var file_handle = new File(filePath, "wb");
 
                if (file_handle && file_handle != null) {
                    file_handle.write(data);
                    file_handle.flush();
                    file_handle.close();
                    console.log("Data written to " + filePath);
                    fileIndex++;
                } else {
                    console.log("Failed to create file: " + filePath);
                }
            } catch (e) {
                console.log("Error: " + e.message);
            }
 
        }, onLeave: function (args) { }
    })
}
 
function hook_dlopne() {
    var once = true;
    Interceptor.attach(Module.findExportByName(null, "dlopen"), {
        onEnter: function (args) {
            var loadFileName = args[0].readCString();
            if (loadFileName.indexOf('libjiagu') != -1) {
                console.log("Load -> ", loadFileName);
                this.is_can_hook = true;
            }
        }, onLeave: function () {
            if (this.is_can_hook && once) {
                travedec();
                once = false;
 
            }
        }
    })
}
 
 
setImmediate(hook_dlopne);
```

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723401316623.png)

![](https://shangwendada.top/wp-content/uploads/2024/08/image-1723401492288.png)

至此，结束。

参考
--

整个内容复现于 [https://oacia.dev/360-jiagu/](https://oacia.dev/360-jiagu/)

  

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 23 分钟前 被 Shangwendada 编辑 ，原因： 上传附件 [#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm)

上传的附件：

*   [dumpDex.js](javascript:void(0)) （1.78kb，0 次下载）
*   [dumpSo.js](javascript:void(0)) （0.56kb，0 次下载）
*   [finddlopenLoad.js](javascript:void(0)) （0.59kb，0 次下载）
*   [findString.js](javascript:void(0)) （0.31kb，0 次下载）
*   [hookOpen.js](javascript:void(0)) （1.53kb，0 次下载）
*   [hookRc4.js](javascript:void(0)) （1.26kb，0 次下载）
*   [Hookuncompress.js](javascript:void(0)) （1.57kb，1 次下载）
*   [trace_libjiagu_64_ozqyp.js](javascript:void(0)) （11.64kb，0 次下载）
*   [testjg_e18a5a41_enc.apk](javascript:void(0)) （7.63MB，0 次下载）