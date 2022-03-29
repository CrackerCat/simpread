> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272091.htm)

> MemoryModule x86x64 内存加载 DLL

MemoryModule x86x64 内存加载 DLL
----------------------------

by devseed，此篇教程同时发在论坛和[我的博客](https://blog.schnee.moe/posts/MemoryModule/)上，完整源码见我的 [github](https://github.com/YuriSizuku/MemoryModule)。

0x0 前言
------

距离上次我写的 [SimpleDpack](https://github.com/YuriSizuku/SimpleDpack) 教程（为了理解加壳原理自己写的轻量级压缩壳），已经有一年多了。前几天修复`x64`的`IAT`载入序号问题，于是想着可以用这个原理自己重写一个`解析PE`和`内存加载DLL`的功能。虽然已有类似实现功能，如 [MemoryModule](https://github.com/fancycode/MemoryModule)，但是这个项目还是有点复杂，模块间耦合度有些高，**没法作为`shellcode`附加到 EXE 来引导 DLL**。

 

因此我决定重写一个低耦合，同时支持`x86`和`x64`，可以**作为`shellcode`引导” 附加到 EXE 区段上的 DLL” 的框架** 。

0x1 MemoryModule 设计思路
---------------------

### (1) usage

本框架采取与 windows 加载 DLL 相似的名称来设计 api，可以实现：

*   将 DLL 按照内存对齐模式载入到内存中
*   动态重定向 DLL 中的`reloc`地址，并载入`IAT`

```
const char *dllpath = "test.dll";
size_t mempesize = 0;
void *memdll = NULL;
 
// load the pe file in memory and align it to memory align
void *mempe = winpe_memload_file(dllpath, &mempesize, TRUE);
 
// memory loadlibrary
memdll = winpe_memLoadLibrary(mempe);
winpe_memFreeLibrary(memdll);
 
// memory loadlibrary at specific address
size_t targetaddr = sizeof(size_t) > 4 ? 0x140030000: 0x90000;
memdll = winpe_memLoadLibraryEx(mempe, targetaddr,
    WINPE_LDFLAG_MEMALLOC, (PFN_LoadLibraryA)winpe_findloadlibrarya(),
    (PFN_GetProcAddress)winpe_memGetProcAddress);
winpe_memFreeLibrary(memdll);
free(mempe);

```

*   将 DLL 注入到 EXE 的附加区段，并且用`shellcode`引导该 DLL

```
win_injectmemdll.exe exepath dllpath [outpath]

```

### (2) libwinpe.c

参考 [stb](https://github.com/nothings/stb) 用`XXX_IMPLEMENTATION`宏控制代码实现，使得 include 单个头文件就能把模块功能全部引入的做法，有着单头文件结成非常容易实用的特点。于是我就用了一周多时间，自己写了个类似的低耦合单头文件 [winpe.h](https://github.com/YuriSizuku/ReverseUtil/blob/master/include/winpe.h)，用于**解析与加载 PE**。为了便于`shellcode`引导，系统函数大多用函数指针传入参数，关键函数全部声明为`inline`。另外，我还用内联汇编写了从`ldr`中获取`kernel32`的基地址，从而减少对 IAT 硬编码的依赖。

 

`libwinpe.c`, 与`libwinpe.def`（引入`def`是因为`stdcall`编译后会被修改名称），把`winpe.h`的声明的导出函数编译为 DLL，如下：

```
#define WINPE_SHARED
#define WINPE_IMPLEMENTATION
#include "winpe.h"

```

```
EXPORTS
    winpe_appendsecth
    winpe_findgetprocaddress
    winpe_findkernel32
    winpe_findloadlibrarya
    winpe_findspace
    winpe_findmodulea
    winpe_imagebaseval
    winpe_imagesizeval
    winpe_memFreeLibrary
    winpe_memFreeLibraryEx
    winpe_memGetProcAddress
    winpe_memLoadLibrary
    winpe_memLoadLibraryEx
    winpe_membindiat
    winpe_membindtls
    winpe_memfindexp
    winpe_memfindiat
    winpe_memforwardexp
    winpe_memload
    winpe_memload_file
    winpe_memreloc
    winpe_noaslr
    winpe_oepval
    winpe_overlayload_file
    winpe_overlayoffset

```

### (3) win_injectmemdll.c

对于编写`shellcode`， 这里为了方便调试，采取用`pyhton`的`keystone`进行`jit`生成的机器码写入`win_injectmemdll.c`的全局变量里的做法， 如下：

```
def inject_shellcodestubs(srcpath, libwinpepath, targetpath):
    pedll = lief.parse(libwinpepath)
    pedll_oph = pedll.optional_header
 
    # generate oepint shellcode
    if pedll_oph.magic == lief.PE.PE_TYPE.PE32_PLUS:
        oepinit_code = gen_oepinit_code64()
        pass
    elif pedll_oph.magic == lief.PE.PE_TYPE.PE32:
        oepinit_code = gen_oepinit_code32()
        pass
    else:
        print("error invalid pe magic!", pedll_oph.magic)
        return
 
    # find necessary functions
    FUNC_SIZE =0x400
    codes = {"winpe_memreloc": 0,
        "winpe_membindiat": 0,
        "winpe_membindtls": 0,
        "winpe_findloadlibrarya": 0,
        "winpe_findgetprocaddress": 0}
    for k in codes.keys():
        func = next(filter(lambda e : e.name == k,
            pedll.exported_functions))
        codes[k] = pedll.get_content_from_virtual_address(
            func.address, FUNC_SIZE)
    codes['winpe_oepinit'] = oepinit_code
 
    # write shellcode to c source file
    with open(srcpath, "rb") as fp:
        srctext = fp.read().decode('utf8')
    for k, v in codes.items():
        k = k.replace("winpe_", "")
        _codetext = ",".join([hex(x) for x in v])
        srctext = re.sub("g_" + k + r"_code(.+?)(\{0x90\})",
            "g_" + k  +  r"_code\1{" + _codetext +"}", srctext)
    with open(targetpath, "wb") as fp:
        fp.write(srctext.encode('utf8'))

```

详见 [win_injectmemdll_shellcodestub.py](https://github.com/YuriSizuku/MemoryModule/blob/master/src/memdll/win_injectmemdll_shellcodestub.py)

0x2 内存加载 DLL
------------

### (1) windows dll 加载机制

参考 https://bbs.pediy.com/thread-252260.htm， windows 对于加载 DLL 调用如下：

> **a) 当调用 LoadLibrary 时首先打开文件并将按照 PE 文件格式解析数据，并以 IMAGE 方式将 PE 各段内容**
> 
> **逐块映射到内存中, 如果需要并修复重定位：**
> 
> kernel32!LoadlibraryA
> 
> ->kernel32!LoadLibraryExA
> 
> ​ ->kernel32!LoadLibraryExW
> 
> ​ ->ntdll!LdrLoadDll
> 
> ​ ->ntdll!LdrpLoadDll
> 
> ​ ->ntdll!LdrpMapDll
> 
> ​ ->ntdll!LdrpCreateDllSection
> 
> ​ ->ntdll!NtOpenFile // 打开目标文件 *
> 
> ​ ->ntdll!NtCreateSection // 创建映射 *
> 
> ​ ->ntdll!NtMapViewOfSection // 映射 *
> 
> ​ ->LdrRelocateImageWithBias // 修复重定位
> 
> **b) 接着处理调入表等，然后调用 TLS 回调函数和 DLLMAN 入口函数；**
> 
> kernel32!LoadlibraryA
> 
> ->kernel32!LoadLibraryExA
> 
> ->kernel32!LoadLibraryExW
> 
> ​ ->ntdll!LdrLoadDll
> 
> ​ ->ntdll!LdrpLoadDll
> 
> ​ ->ntdll!LdrpMapDll
> 
> ​ ->ntdll!LdrpWalkImportDescriptor // 处理导入表等
> 
> ​ ->LdrpRunInitializeRoutines // 调用 DLLMAN
> 
> ​ ->LdrpCallTlsInitializers
> 
> ​ ->LdrpCallInitRoutine

 

总结为如下流程：

1.  载入 DLL 到内存上，将 DLL 进行`memory align`对齐
2.  通过`reloc`段重定向硬编码地址
3.  逐一扫码`IAT`表`OFT`的模块和函数名，加载对于模块，将地址填入`FT`中
4.  调用`DllMain函数`（即 DLL 的`OEP`），`fdwReason`参数为`DLL_PROCESS_ATTACH`
5.  初始化`TLS`, `reason` 为`DLL_PROCESS_ATTACH`

### (2) 模拟 Windows 加载 DLL

#### .1 载入 DLL 到内存上，将 DLL 进行`memory align`对齐

这里为了降低耦合，没有用系统的`memset`和`memcpy`，自己用 inline 实现的。

```
inline size_t STDCALL winpe_memload(
    const void *rawpe, size_t rawsize,
    void *mempe, size_t memsize,
    bool_t same_align)
{
    // load rawpe to memalign
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)rawpe;
    PIMAGE_NT_HEADERS  pNtHeader = (PIMAGE_NT_HEADERS)
        ((void*)rawpe + pDosHeader->e_lfanew);
    PIMAGE_FILE_HEADER pFileHeader = &pNtHeader->FileHeader;
    PIMAGE_OPTIONAL_HEADER pOptHeader = &pNtHeader->OptionalHeader;
    PIMAGE_SECTION_HEADER pSectHeader = (PIMAGE_SECTION_HEADER)
        ((void *)pOptHeader + pFileHeader->SizeOfOptionalHeader);
    WORD sectNum = pFileHeader->NumberOfSections;
    size_t imagesize = pOptHeader->SizeOfImage;
    if(!mempe) return imagesize;
    else if(memsize!=0 && memsizeSizeOfHeaders);
 
    for(WORD i=0;ie_lfanew);
        pFileHeader = &pNtHeader->FileHeader;
        pOptHeader = &pNtHeader->OptionalHeader;
        pSectHeader = (PIMAGE_SECTION_HEADER)
            ((void *)pOptHeader + pFileHeader->SizeOfOptionalHeader);
        sectNum = pFileHeader->NumberOfSections;
 
        pOptHeader->FileAlignment = pOptHeader->SectionAlignment;
 
        for(WORD i=0;i
```

#### .2 通过`reloc`段重定向硬编码地址

手动将以`imagebase`为基准的`va`重定向到目标`imagebase`的`va`。

```
inline size_t STDCALL winpe_memreloc(
    void *mempe, size_t newimagebase)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)mempe;
    PIMAGE_NT_HEADERS  pNtHeader = (PIMAGE_NT_HEADERS)
        ((void*)mempe + pDosHeader->e_lfanew);
    PIMAGE_FILE_HEADER pFileHeader = &pNtHeader->FileHeader;
    PIMAGE_OPTIONAL_HEADER pOptHeader = &pNtHeader->OptionalHeader;
    PIMAGE_DATA_DIRECTORY pDataDirectory = pOptHeader->DataDirectory;
    PIMAGE_DATA_DIRECTORY pRelocEntry = &pDataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];
 
    DWORD reloc_count = 0;
    DWORD reloc_offset = 0;
    int64_t shift = (int64_t)newimagebase -
        (int64_t)pOptHeader->ImageBase;
    while (reloc_offset < pRelocEntry->Size)
    {
        PIMAGE_BASE_RELOCATION pBaseReloc = (PIMAGE_BASE_RELOCATION)
            ((void*)mempe + pRelocEntry->VirtualAddress + reloc_offset);
        PRELOCOFFSET pRelocOffset = (PRELOCOFFSET)((void*)pBaseReloc
            + sizeof(IMAGE_BASE_RELOCATION));
        // RELOCOFFSET block num
        DWORD item_num = (pBaseReloc->SizeOfBlock -
            sizeof(IMAGE_BASE_RELOCATION)) / sizeof(RELOCOFFSET);
        for (size_t i = 0; i < item_num; i++)
        {
            if (!pRelocOffset[i].type &&
                !pRelocOffset[i].offset) continue;
            DWORD targetoffset = pBaseReloc->VirtualAddress +
                    pRelocOffset[i].offset;
            size_t *paddr = (size_t *)((void*)mempe + targetoffset);
            size_t relocaddr = (size_t)((int64_t)*paddr + shift);
            *paddr = relocaddr;
        }
        reloc_offset += sizeof(IMAGE_BASE_RELOCATION) +
            sizeof(RELOCOFFSET) * item_num;
        reloc_count += item_num;
    }
    pOptHeader->ImageBase = newimagebase;
    return reloc_count;
}

```

#### .3 逐一扫码`IAT`表`OFT`的模块和函数名，加载对于模块，将地址填入`FT`中

用`LoadLibraryA`和`GetProcAddress`实现，考虑到 shellcode，不能直接调用，应该通过传入函数指针调用。

 

另外还要考虑`OFT`中会出现用序号的情况，`x86`和`x64`判断表中不同，用`sizeof(size_t)`区分。

```
WINPEDEF WINPE_EXPORT
inline size_t STDCALL winpe_membindiat(void *mempe,
    PFN_LoadLibraryA pfnLoadLibraryA,
    PFN_GetProcAddress pfnGetProcAddress)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)mempe;
    PIMAGE_NT_HEADERS  pNtHeader = (PIMAGE_NT_HEADERS)
        ((void*)mempe + pDosHeader->e_lfanew);
    PIMAGE_FILE_HEADER pFileHeader = &pNtHeader->FileHeader;
    PIMAGE_OPTIONAL_HEADER pOptHeader = &pNtHeader->OptionalHeader;
    PIMAGE_DATA_DIRECTORY pDataDirectory = pOptHeader->DataDirectory;
    PIMAGE_DATA_DIRECTORY pImpEntry = 
        &pDataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
    PIMAGE_IMPORT_DESCRIPTOR pImpDescriptor = 
        (PIMAGE_IMPORT_DESCRIPTOR)(mempe + pImpEntry->VirtualAddress);
 
    PIMAGE_THUNK_DATA pFtThunk = NULL;
    PIMAGE_THUNK_DATA pOftThunk = NULL;
    LPCSTR pDllName = NULL;
    PIMAGE_IMPORT_BY_NAME pImpByName = NULL;
    size_t funcva = 0;
    char *funcname = NULL;
 
    // origin GetProcAddress will crash at InitializeSListHead
    if(!pfnLoadLibraryA) pfnLoadLibraryA =
        (PFN_LoadLibraryA)winpe_findloadlibrarya();
    if(!pfnGetProcAddress) pfnGetProcAddress =
        (PFN_GetProcAddress)winpe_findgetprocaddress();
 
    DWORD iat_count = 0;
    for (; pImpDescriptor->Name; pImpDescriptor++)
    {
        pDllName = (LPCSTR)(mempe + pImpDescriptor->Name);
        pFtThunk = (PIMAGE_THUNK_DATA)
            (mempe + pImpDescriptor->FirstThunk);
        pOftThunk = (PIMAGE_THUNK_DATA)
            (mempe + pImpDescriptor->OriginalFirstThunk);
        size_t dllbase = (size_t)pfnLoadLibraryA(pDllName);
        if(!dllbase) return 0;
 
        for (int j=0; pFtThunk[j].u1.Function
            &&  pOftThunk[j].u1.Function; j++)
        {
            size_t _addr = (size_t)(mempe + pOftThunk[j].u1.AddressOfData);
            if(sizeof(size_t)>4) // x64
            {
                if((_addr>>63) == 1)
                {
                    funcname = (char *)(_addr & 0xffff);
                }
                else
                {
                    pImpByName=(PIMAGE_IMPORT_BY_NAME)_addr;
                    funcname = pImpByName->Name;
                }
            }
            else
            {
                if(((size_t)pImpByName>>31) == 1)
                {
                    funcname = (char *)(_addr & 0xffff);
                }
                else
                {
                    pImpByName=(PIMAGE_IMPORT_BY_NAME)_addr;
                    funcname = pImpByName->Name;
                }
            }
 
            funcva = (size_t)pfnGetProcAddress(
                (HMODULE)dllbase, funcname);
            if(!funcva) continue;
            pFtThunk[j].u1.Function = funcva;
            assert(funcva == (size_t)GetProcAddress(
                (HMODULE)dllbase, funcname));
            iat_count++;
        }
    }
    return iat_count;
}

```

#### .4 调用`DllMain函数`（即 DLL 的`OEP`），`fdwReason`参数为`DLL_PROCESS_ATTACH`

前三步都完成后可以来通过 DllMain 来引导了。

```
// initial memory module
if(!winpe_memreloc((void*)imagebase, imagebase))
    return NULL;
if(!winpe_membindiat((void*)imagebase,
    pfnLoadLibraryA, pfnGetProcAddress)) return NULL;
winpe_membindtls(mempe, DLL_PROCESS_ATTACH);
PFN_DllMain pfnDllMain = (PFN_DllMain)
    (imagebase + winpe_oepval((void*)imagebase, 0));
pfnDllMain((HINSTANCE)imagebase, DLL_PROCESS_ATTACH, NULL);
return (void*)imagebase;

```

#### .5 初始化`TLS`, `reason` 为`DLL_PROCESS_ATTACH`

这部分我没有找到有 TLS 的 DLL，所以没法进一步测试了。这部分代码直接借鉴的 [MemoryModule](https://github.com/fancycode/MemoryModule) 的相关函数。

```
inline size_t STDCALL winpe_membindtls(void *mempe, DWORD resaon)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)mempe;
    PIMAGE_NT_HEADERS  pNtHeader = (PIMAGE_NT_HEADERS)
        ((void*)mempe + pDosHeader->e_lfanew);
    PIMAGE_FILE_HEADER pFileHeader = &pNtHeader->FileHeader;
    PIMAGE_OPTIONAL_HEADER pOptHeader = &pNtHeader->OptionalHeader;
    PIMAGE_DATA_DIRECTORY pDataDirectory = pOptHeader->DataDirectory;
    PIMAGE_DATA_DIRECTORY pTlsDirectory =
        &pDataDirectory[IMAGE_DIRECTORY_ENTRY_TLS];
    if(!pTlsDirectory->VirtualAddress) return 0;
 
    size_t tls_count = 0;
    PIMAGE_TLS_DIRECTORY pTlsEntry = (PIMAGE_TLS_DIRECTORY)
        (mempe + pTlsDirectory->VirtualAddress);
    PIMAGE_TLS_CALLBACK *tlscb= (PIMAGE_TLS_CALLBACK*)
        pTlsEntry->AddressOfCallBacks;
    if(tlscb)
    {
        while(*tlscb)
        {
            (*tlscb)(mempe, reason, NULL);
            tlscb++;
            tls_count++;
        }
    }
    return tls_count;
}

```

0x3 DLL 附加到 EXE 区段与 shellcode 编写
--------------------------------

### (1) EXE 区段扩容

首先需要修改 exe 的区段头，在后面增加一个区段。这里要特别注意区段的对齐，之后还要修正`SizeOfImage`。

```
inline size_t STDCALL winpe_appendsecth(void *pe,
    PIMAGE_SECTION_HEADER psecth)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)pe;
    PIMAGE_NT_HEADERS  pNtHeader = (PIMAGE_NT_HEADERS)
        ((void*)pe + pDosHeader->e_lfanew);
    PIMAGE_FILE_HEADER pFileHeader = &pNtHeader->FileHeader;
    PIMAGE_OPTIONAL_HEADER pOptHeader = &pNtHeader->OptionalHeader;
    PIMAGE_SECTION_HEADER pSectHeader = (PIMAGE_SECTION_HEADER)
        ((void*)pOptHeader + pFileHeader->SizeOfOptionalHeader);
    WORD sectNum = pFileHeader->NumberOfSections;
    PIMAGE_SECTION_HEADER pLastSectHeader = &pSectHeader[sectNum-1];
    DWORD addr, align;
 
    // check the space to append section
    if(pFileHeader->SizeOfOptionalHeader
        + sizeof(IMAGE_SECTION_HEADER)
     > pSectHeader[0].PointerToRawData) return 0;
 
    // fill rva addr
    align = pOptHeader->SectionAlignment;
    addr = pLastSectHeader->VirtualAddress + pLastSectHeader->Misc.VirtualSize;
    if(addr % align) addr += align - addr%align;
    psecth->VirtualAddress = addr;
 
    // fill file offset
    align = pOptHeader->FileAlignment;
    addr =  pLastSectHeader->PointerToRawData+ pLastSectHeader->SizeOfRawData;
    if(addr % align) addr += align - addr%align;
    psecth->PointerToRawData = addr;
 
    // adjust the section and imagesize
    pFileHeader->NumberOfSections++;
    _inl_memcpy(&pSectHeader[sectNum], psecth, sizeof(IMAGE_SECTION_HEADER));
    align = pOptHeader->SectionAlignment;
    addr = psecth->VirtualAddress + psecth->Misc.VirtualSize;
    if(addr % align) addr += align - addr%align;
    pOptHeader->SizeOfImage = addr;
    return pOptHeader->SizeOfImage;
}

```

### (2) 引导 DLL 的 shellcode 编写

这部分内容和`winpe_memLoadLibrary`函数实现的方式类似，只不过要用汇编来写`shellcode`，引导完成后跳转到原来 exe 的`oep`。 关键地址可以先留空，在注入时候填入对位置。对于相应的函数的 shellcode，由`win_injectmemdll_shellcodestub.py`读取对应的`libpe32.dll`函数，填入对应的全局变量。下面代码以 x86 为例：

```
def gen_oepinit_code32():
    ks = Ks(KS_ARCH_X86, KS_MODE_32)
    code_str = f"""
    // for relative address, get the base of addr
    call getip;
    lea ebx, [eax-5];
 
    // get the imagebase
    mov eax, 0x30; // to avoid relative addressing
    mov edi, dword ptr fs:[eax]; //peb
    mov edi, [edi + 0ch]; //ldr
    mov edi, [edi + 14h]; //InMemoryOrderLoadList, this
    mov edi, [edi -8h + 18h]; //this.DllBase
 
    // get loadlibrarya, getprocaddress
    mov eax, [ebx + findloadlibrarya];
    add eax, edi;
    call eax;
    mov [ebx + findloadlibrarya], eax;
    mov eax, [ebx + findgetprocaddress];
    add eax, edi;
    call eax;
    mov [ebx + findgetprocaddress], eax;
 
    // reloc
    mov eax, [ebx + dllrva];
    add eax, edi;
    push eax;
    push eax;
    mov eax, [ebx + memrelocrva];
    add eax, edi;
    call eax;
 
    // bind iat
    mov eax, [ebx + findgetprocaddress];
    push eax; // arg3, getprocaddress
    mov eax, [ebx + findloadlibrarya];
    push eax; // arg2, loadlibraryas
    mov eax, [ebx + dllrva];
    add eax, edi;
    push eax; // arg1, dllbase value
    mov eax, [ebx + membindiatrva];
    add eax, edi
    call eax;
 
    // bind tls
    xor eax, eax;
    inc eax;
    push eax; // arg2, reason for tls
    mov eax, [ebx + dllrva]
    add eax, edi;
    push eax; // arg1, dllbase
    mov eax, [ebx + membindtlsrva];
    add eax, edi;
    call eax;
 
    // call dll oep, for dll entry
    xor eax, eax;
    push eax; // lpvReserved
    inc eax;
    push eax; // fdwReason, DLL_PROCESS_ATTACH
    mov eax, [ebx + dllrva];
    add eax, edi;
    push eax; // hinstDLL
    mov eax, [ebx + dlloeprva];
    add eax, edi;
    call eax;
 
    // jmp to origin oep
    mov eax, [ebx+exeoeprva];
    add eax, edi;
    jmp eax;
    getip:
    mov eax, [esp]
    ret
    exeoeprva: nop;nop;nop;nop;
    dllrva: nop;nop;nop;nop;
    dlloeprva: nop;nop;nop;nop;
    memrelocrva: nop;nop;nop;nop;
    membindiatrva: nop;nop;nop;nop;
    membindtlsrva: nop;nop;nop;nop;
    findloadlibrarya: nop;nop;nop;nop;
    findgetprocaddress: nop;nop;nop;nop;

```

```
// these functions are stub function, will be filled by python
#define FUNC_SIZE 0x400
#define SHELLCODE_SIZE 0X2000
unsigned char g_oepinit_code[] = {0x90};
unsigned char g_memreloc_code[] = {0x90};
unsigned char g_membindiat_code[] = {0x90};
unsigned char g_membindtls_code[] = {0x90};
unsigned char g_findloadlibrarya_code[] = {0x90};
unsigned char g_findgetprocaddress_code[] = {0x90};
 
void _makeoepcode(void *shellcode,
    size_t shellcoderva, size_t dllrva,
    DWORD orgexeoeprva, DWORD orgdlloeprva)
{
    // bind the pointer to buffer
    size_t oepinit_end = sizeof(g_oepinit_code);
    size_t memreloc_start = FUNC_SIZE;
    size_t membindiat_start = memreloc_start + FUNC_SIZE;
    size_t membindtls_start = membindiat_start + FUNC_SIZE;
    size_t findloadlibrarya_start = membindtls_start + FUNC_SIZE;
    size_t findgetprocaddress_start = findloadlibrarya_start + FUNC_SIZE;
 
     // fill the address table
    size_t *pexeoeprva = (size_t*)(g_oepinit_code + oepinit_end - 8*sizeof(size_t));
    size_t *pdllbrva = (size_t*)(g_oepinit_code + oepinit_end - 7*sizeof(size_t));
    size_t *pdlloeprva = (size_t*)(g_oepinit_code + oepinit_end - 6*sizeof(size_t));
    size_t *pmemrelocrva = (size_t*)(g_oepinit_code + oepinit_end - 5*sizeof(size_t));
    size_t *pmembindiatrva = (size_t*)(g_oepinit_code + oepinit_end - 4*sizeof(size_t));
    size_t *pmembindtlsrva = (size_t*)(g_oepinit_code + oepinit_end - 3*sizeof(size_t));
    size_t *pfindloadlibrarya = (size_t*)(g_oepinit_code + oepinit_end - 2*sizeof(size_t));
    size_t *pfindgetprocaddress = (size_t*)(g_oepinit_code + oepinit_end - 1*sizeof(size_t));
    *pexeoeprva = orgexeoeprva;
    *pdllbrva =  dllrva;
    *pdlloeprva = dllrva + orgdlloeprva;
    *pmemrelocrva = shellcoderva + memreloc_start;
    *pmembindiatrva = shellcoderva + membindiat_start;
    *pmembindtlsrva = shellcoderva + membindtls_start;
    *pfindloadlibrarya = shellcoderva + findloadlibrarya_start;
    *pfindgetprocaddress = shellcoderva + findgetprocaddress_start;
 
    // copy to the target
    memcpy(shellcode ,
        g_oepinit_code, sizeof(g_oepinit_code));
    memcpy(shellcode + memreloc_start,
        g_memreloc_code, sizeof(g_memreloc_code));
    memcpy(shellcode + membindiat_start,
        g_membindiat_code, sizeof(g_membindiat_code));
    memcpy(shellcode + membindtls_start,
        g_membindtls_code, sizeof(g_membindtls_code));
    memcpy(shellcode + findloadlibrarya_start,
        g_findloadlibrarya_code, sizeof(g_findloadlibrarya_code));
    memcpy(shellcode + findgetprocaddress_start,
        g_findgetprocaddress_code, sizeof(g_findgetprocaddress_code));
}

```

0x4 技巧 tips
-----------

### (1) x64 适配

由于大部分都用的`size_t`或`void*`等与 cpu 字长相关的变量，大部分代码不用修改就能编译。

 

其余的差异部分，可以用`sizeof(size_t)>4`区分，或者`_WIN64`宏。如下所示

```
// append section header to exe
size_t align = sizeof(size_t) > 4 ? 0x10000: 0x1000;
size_t padding = _sectpaddingsize(mempe_exe, mempe_dll, align);
secth.Characteristics = IMAGE_SCN_MEM_READ |
    IMAGE_SCN_MEM_WRITE | IMAGE_SCN_MEM_EXECUTE;
secth.Misc.VirtualSize = SHELLCODE_SIZE + padding + mempe_dllsize;
secth.SizeOfRawData = SHELLCODE_SIZE + padding + mempe_dllsize;
strcpy((char*)secth.Name, ".module");
winpe_noaslr(mempe_exe);
winpe_appendsecth(mempe_exe, §h);

```

```
inline void* winpe_findkernel32()
{
    // return (void*)LoadLibrary("kernel32.dll");
    // TEB->PEB->Ldr->InMemoryOrderLoadList->curProgram->ntdll->kernel32
    void *kerenl32 = NULL;
#ifdef _WIN64
    __asm{
        mov rax, gs:[60h]; peb
        mov rax, [rax+18h]; ldr
        mov rax, [rax+20h]; InMemoryOrderLoadList, currentProgramEntry
        mov rax, [rax]; ntdllEntry, currentProgramEntry->->Flink
        mov rax, [rax]; kernel32Entry,  ntdllEntry->Flink
        mov rax, [rax-10h+30h]; kernel32.DllBase
        mov kerenl32, rax;
    }
#else
    __asm{
        mov eax, fs:[30h]; peb
        mov eax, [eax+0ch]; ldr
        mov eax, [eax+14h]; InMemoryOrderLoadList, currentProgramEntry
        mov eax, [eax]; ntdllEntry, currentProgramEntry->->Flink
        mov eax, [eax]; kernel32Entry,  ntdllEntry->Flink
        mov eax, [eax - 8h +18h]; kernel32.DllBase
        mov kerenl32, eax;
    }
#endif
    return kerenl32;
}

```

### (2) 扫描 export 找到 LoadLibrary 和 GetProcAddress

上面的代码我们可以获取`kernel32`的基址了，之后可以自己通过扫描`export表`来查找对应的导出函数了。

 

这里注意有个坑点，有的导出表项会 forward 到其他 dll 上，如`kernel32!InitializeSListHead -> NTDLL!RtlInitializeSListHead`，详见 [crashing-kernel32-initializeslisthead](https://www.unknowncheats.me/forum/general-programming-and-reversing/289451-manualy-mapped-dll-crashing-kernel32-initializeslisthead.html)。通过`DirRVA < FuncRVA < DirRVA + ExportSize`来判断是否 forward。

 

此处为了找到`LoadLibraryA`和`GetProcAddress`函数，不考虑 forward 情况，否则会引入递归问题。

```
inline void* STDCALL winpe_memfindexp(
    void *mempe, LPCSTR funcname)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)mempe;
    PIMAGE_NT_HEADERS  pNtHeader = (PIMAGE_NT_HEADERS)
        ((void*)mempe + pDosHeader->e_lfanew);
    PIMAGE_FILE_HEADER pFileHeader = &pNtHeader->FileHeader;
    PIMAGE_OPTIONAL_HEADER pOptHeader = &pNtHeader->OptionalHeader;
    PIMAGE_DATA_DIRECTORY pDataDirectory = pOptHeader->DataDirectory;
    PIMAGE_DATA_DIRECTORY pExpEntry = 
        &pDataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT];
    PIMAGE_EXPORT_DIRECTORY  pExpDescriptor = 
        (PIMAGE_EXPORT_DIRECTORY)(mempe + pExpEntry->VirtualAddress);
 
    WORD *ordrva = mempe + pExpDescriptor->AddressOfNameOrdinals;
    DWORD *namerva = mempe + pExpDescriptor->AddressOfNames;
    DWORD *funcrva = mempe + pExpDescriptor->AddressOfFunctions;
    if((size_t)funcname <= MAXWORD) // find by ordnial
    {
        WORD ordbase = LOWORD(pExpDescriptor->Base) - 1;
        WORD funcord = LOWORD(funcname);
        return (void*)(mempe + funcrva[ordrva[funcord-ordbase]]);
    }
    else
    {
        for(int i=0;iNumberOfNames;i++)
        {
            LPCSTR curname = (LPCSTR)(mempe+namerva[i]);
            if(_inl_stricmp(curname, funcname)==0)
            {
                return (void*)(mempe + funcrva[ordrva[i]]);
            }      
        }
    }
    return NULL;
}
 
inline PROC winpe_findloadlibrarya()
{
    // return (PROC)LoadLibraryA;
    HMODULE hmod_kernel32 = (HMODULE)winpe_findkernel32();
    char name_LoadLibraryA[] = {'L', 'o', 'a', 'd', 'L', 'i', 'b', 'r', 'a', 'r', 'y', 'A', '\0'};
    return (PROC)winpe_memfindexp( // suppose exp no forward, to avoid recursive
        (void*)hmod_kernel32, name_LoadLibraryA);
}
 
inline PROC winpe_findgetprocaddress()
{
    // return (PROC)GetProcAddress;
    HMODULE hmod_kernel32 = (HMODULE)winpe_findkernel32();
    char name_GetProcAddress[] = {'G', 'e', 't', 'P', 'r', 'o', 'c', 'A', 'd', 'd', 'r', 'e', 's', 's', '\0'};
    return (PROC)winpe_memfindexp(hmod_kernel32, name_GetProcAddress);
} 
```

0x5 后记
------

本项目是我实现不加壳`vfs`（即通过 hook nt 函数将文件和文件夹等读取重定向到 zip 文件中）的衍生产物，采取的是尽量单头文件法和内联函数减小耦合，方便集成到代码中的做法。与其他的`MemoryModule`最大的区别是支持从`shellcode`引导 DLL。由于以前我有了自己写压缩壳的经验，整体上处理起来还算是顺利。强迫症，感觉好多时间都花在了怎么统一化命名 api 和代码风格规范上了。至于实现内存加载`Resource`方面，暂时不考虑实现，汉化游戏一般也用不到这方面内容。

 

另外，这里还有个坑卡了我好长时间，就是`x64`的堆栈必须要`0x10`对齐（即调用函数时要`sub rsp, 0x28`预留空间与对齐），才能在`movaps`指令上不报错。之前一直不知道这条指令需要对齐，一直以为是少了哪些初始化，走偏了好久。特别感谢 [@YeLikERs](https://github.com/YeLikERs) 指出了这一点，同时在`PEB`，`LDR`相关内容上帮助了很多。

 

经测试兼容性还算不错，`windows xp`, `windows 7`, `windows 10`, `windows 11`测试 example 文件都通过了，甚至我连`linux`上的`wine`也测试了（即用我汉化的游戏测试 shellcode 引导附加到 exe 区段上的 DLL）。最后，附上几张测试图：

 

![](https://p.sda1.dev/5/b0e1d131252736cc71a14e294bf1bf1f)  
![](https://p.sda1.dev/5/94773e992a24d39452d7dae0d524c110)

[【公告】看雪招聘大学实习生！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#HOOK / 注入](forum-41-1-133.htm)