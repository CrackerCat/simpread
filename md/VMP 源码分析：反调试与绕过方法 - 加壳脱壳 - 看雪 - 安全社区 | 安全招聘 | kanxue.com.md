> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282244.htm)

> VMP 源码分析：反调试与绕过方法

1.vmp 反调试相关源码部分
===============

1.1 如何检索反调试源码
-------------

我们都知道，当 vmp 检测到被调试，会有如下弹框  
![](https://bbs.kanxue.com/upload/attach/202406/995602_QXBWZD9XJPYVBA6.webp)  
通过这条报错信息，不难在源码中找到  
![](https://bbs.kanxue.com/upload/attach/202406/995602_GCG2ENYJRTVXJTH.webp)  
然后通过它的消息传递机制，不难找到

```
void LoaderMessage(MessageType type, const void *param1 = NULL, const void *param2 = NULL)
{
    const VMP_CHAR *message;
    bool need_format = false;
    switch (type) {
    case mtDebuggerFound:
        message = reinterpret_cast(FACE_DEBUGGER_FOUND);
        break;
    case mtVirtualMachineFound:
        message = reinterpret_cast(FACE_VIRTUAL_MACHINE_FOUND);
        break;
    case mtFileCorrupted:
        message = reinterpret_cast(FACE_FILE_CORRUPTED);
        break;
    case mtUnregisteredVersion:
        message = reinterpret_cast(FACE_UNREGISTERED_VERSION);
        break;
    case mtInitializationError:
        message = reinterpret_cast(FACE_INITIALIZATION_ERROR);
        need_format = true;
        break;
    case mtProcNotFound:
        message = reinterpret_cast(FACE_PROC_NOT_FOUND);
        need_format = true;
        break;
    case mtOrdinalNotFound:
        message = reinterpret_cast(FACE_ORDINAL_NOT_FOUND);
        need_format = true;
        break;
    default:
        return;
    } 
```

然后查找 **mtDebuggerFound** 的引用即可检索到各处反调试相关源码，也就是此文将要详细说的，至于其他部分的检测，感兴趣的童鞋可以自行研究。

1.2 源码阅读：反调试手段
--------------

### 1.2.1 系统版本号的判断

```
if (!os_build_number) {
    if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
    tmp_loader_data->set_is_debugger_detected(true);
}

```

那么这个 **os_build_number** 怎么获取的呢  
![](https://bbs.kanxue.com/upload/attach/202406/995602_RJN5RTY8SB3D8XK.webp)  
简单说，一共两种获取方式，1. 从 **peb** 里直接去取得；2. 从 **ntdll.dll** 的头部获取文件版本号从而确定系统版本。  
可能有的童鞋会问了，系统版本号拿来判断反调试是不是有点什么大病，其实不是，私以为，这边判断系统版本号纯纯的只是为了方便取 syscall 所使用的系统调用号。  
如下，**vmp** 应该是把全量的发行版系统都是硬编码了：  
![](https://bbs.kanxue.com/upload/attach/202406/995602_NTDCJTWFB7AXHSZ.webp)  
当系统版本号不在 vmp 适配过的范围（比如测试版 windows），他则会去 map 一份新的 ntdll ，然后从中找他要的 NT 函数的系统调用号，至于系统调用号是什么，这里就不赘述了。

### 1.2.2 peb->BeingDebugged 标记

```
if (peb->BeingDebugged) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
}

```

会心一笑，peb 里的这个位就不用过多解释了

### 1.2.3 ProcessDebugPort

```
if (NT_SUCCESS(reinterpret_cast(syscall | sc_query_information_process)(process, ProcessDebugPort, &debug_object, sizeof(debug_object), NULL)) && debug_object != 0) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
} 
```

查询 ProcessDebugPort，如果查到了，自然是被调试了，也是很常见的反调

### 1.2.4 ProcessDebugObjectHandle

```
if (NT_SUCCESS(reinterpret_cast(syscall | sc_query_information_process)(process, ProcessDebugObjectHandle, &debug_object, sizeof(debug_object), reinterpret_cast(&debug_object)))
    || debug_object == 0) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
} 
```

查询 ProcessDebugObjectHandle, 如果 存在调试对象句柄，那也是被调试了，也属于常见反调

### 1.2.5 SystemKernelDebuggerInformation

```
SYSTEM_KERNEL_DEBUGGER_INFORMATION info;
NTSTATUS status = nt_query_system_information(SystemKernelDebuggerInformation, &info, sizeof(info), NULL);
if (NT_SUCCESS(status) && info.DebuggerEnabled && !info.DebuggerNotPresent) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
}

```

针对内核调试器的监测，也属常见

### 1.2.6 针对驱动模块名的匹配

```
SYSTEM_MODULE_INFORMATION *buffer = NULL;
ULONG buffer_size = 0;
status = nt_query_system_information(SystemModuleInformation, &buffer, 0, &buffer_size);
if (buffer_size) {
    buffer = reinterpret_cast(LoaderAlloc(buffer_size * 2));
    if (buffer) {
        status = nt_query_system_information(SystemModuleInformation, buffer, buffer_size * 2, NULL);
        if (NT_SUCCESS(status)) {
            for (size_t i = 0; i < buffer->Count && !is_found; i++) {
                SYSTEM_MODULE_ENTRY *module_entry = &buffer->Module[i];
                for (size_t j = 0; j < 5 ; j++) {
                    const char *module_name;
                    switch (j) {
                    case 0:
                        module_name = reinterpret_cast(FACE_SICE_NAME);
                        break;
                    case 1:
                        module_name = reinterpret_cast(FACE_SIWVID_NAME);
                        break;
                    case 2:
                        module_name = reinterpret_cast(FACE_NTICE_NAME);
                        break;
                    case 3:
                        module_name = reinterpret_cast(FACE_ICEEXT_NAME);
                        break;
                    case 4:
                        module_name = reinterpret_cast(FACE_SYSER_NAME);
                        break;
                    }
                    if (Loader_stricmp(module_name, module_entry->Name + module_entry->PathLength, true) == 0) {
                        is_found = true;
                        break;
                    }
                }
            }
        }
        LoaderFree(buffer);
    }
} 
```

这也是针对了一些常见的内核级调试器的检测，他们的驱动名

### 1.2.7 线程隐藏

```
if (sc_set_information_thread)
    reinterpret_cast(syscall | sc_set_information_thread)(thread, ThreadHideFromDebugger, NULL, 0); 
```

对调试器隐藏了当前线程

### 1.2.8 函数头 0xCC 断点检测

```
tNtOpenFile *open_file = reinterpret_cast(LoaderGetProcAddress(ntdll, reinterpret_cast(FACE_NT_OPEN_FILE_NAME), true));
tNtCreateSection *create_section = reinterpret_cast(LoaderGetProcAddress(ntdll, reinterpret_cast(FACE_NT_CREATE_SECTION_NAME), true));
tNtMapViewOfSection *map_view_of_section = reinterpret_cast(LoaderGetProcAddress(ntdll, reinterpret_cast(FACE_NT_MAP_VIEW_OF_SECTION), true));
tNtUnmapViewOfSection *unmap_view_of_section = reinterpret_cast(LoaderGetProcAddress(ntdll, reinterpret_cast(FACE_NT_UNMAP_VIEW_OF_SECTION), true));
tNtClose *close = reinterpret_cast(LoaderGetProcAddress(ntdll, reinterpret_cast(FACE_NT_CLOSE), true));
 
if (!create_section || !open_file || !map_view_of_section || !unmap_view_of_section || !close) {
    LoaderMessage(mtInitializationError, INTERNAL_GPA_ERROR);
    return LOADER_ERROR;
}
 
// check breakpoint
uint8_t *ckeck_list[] = { reinterpret_cast(create_section),
                                    reinterpret_cast(open_file),
                                    reinterpret_cast(map_view_of_section),
                                    reinterpret_cast(unmap_view_of_section),
                                    reinterpret_cast(close) };
for (i = 0; i < _countof(ckeck_list); i++) {
                if (*ckeck_list[i] == 0xcc) {
                    if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
                        LoaderMessage(mtDebuggerFound);
                        return LOADER_ERROR;
                    }
                    tmp_loader_data->set_is_debugger_detected(true);
                }
} 
```

```
if (*reinterpret_cast(virtual_protect) == 0xcc) {
    if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
    tmp_loader_data->set_is_debugger_detected(true);
} 
```

检测自己要调用的函数有没有被下 0xCC 断点

### 1.2.9 内存断点检测

```
if (old_protect & PAGE_GUARD) {
    if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
    tmp_loader_data->set_is_debugger_detected(true);
}

```

### 1.2.10 假句柄

```
tCloseHandle *close_handle = reinterpret_cast(LoaderGetProcAddress(kernel32, reinterpret_cast(FACE_CLOSE_HANDLE_NAME), true));
if (close_handle) {
    __try {
        if (close_handle(HANDLE(INT_PTR(0xDEADC0DE)))) {
            LoaderMessage(mtDebuggerFound);
            return LOADER_ERROR;
        }
    } __except(EXCEPTION_EXECUTE_HANDLER) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
} 
```

通过关闭无效句柄来判断是否成功，如果成功则中了陷阱

### 1.2.11 TrapFlag 与 硬件断点检测

```
__try {
    __writeeflags(__readeflags() | 0x100);
     val = __rdtsc();
     __nop();
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
} __except(ctx = (GetExceptionInformation())->ContextRecord,
    drx = (ctx->ContextFlags & CONTEXT_DEBUG_REGISTERS) ? ctx->Dr0 | ctx->Dr1 | ctx->Dr2 | ctx->Dr3 : 0,
    EXCEPTION_EXECUTE_HANDLER) {
    if (drx) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
}

```

可还行，两个检测写在一起了，通过设置 flags 的 TrapFlag 触发异常，然后在异常处理里检查硬件断点寄存器是否设置  
**至此，反调弹框部分基本看完了**

1.3 源码部分总结
----------

*   用了 10 + 种反调手段，基本都属于常见范畴
*   有检测普通调试器的，有检测内核调试器的
*   一些查询 api 使用的 syscall，难以从 r3 直接突破

2. vmp 反调试的 bypass (纯 r3)
=========================

2.1 几句废话
--------

vmp 的反调试基本是一些常见的反调试手段，  
其中比较棘手的是一些 NT 函数的调用，他使用了 SYSCALL，通过自实现的系统调用规避了我们从 r3 hook 然后绕过的可能。  
通过网上一顿检索，确实看到了不少从 r0 来过 vmp 反调的插件 / 工具 / 源码。  
但是！难道！我们就只能上驱动了么？ 它是 r3 却把我们逼到了 r0，有没有纯纯的三环方法还能绕过他的呢？  
答案当然是，当然存在（狗头），不然我也就不写这个分享了。

2.2 bypass 关键点
--------------

通过不死心的源码阅读，终于让我看到了这块代码  
![](https://bbs.kanxue.com/upload/attach/202406/995602_EJEGK7TRXP6TJAE.webp)  
也就是关键的这一句

```
LoaderGetProcAddress(ntdll, reinterpret_cast(FACE_WINE_GET_VERSION_NAME), true) 
```

此时，小伙伴就会问了，VMP 在搞啥？  
这其实是 vmp 在给 wine 环境做兼容，如果发现 ntdll.dll 的导出表存在 wine_get_version 函数，则会关闭使用系统调用的特性！！！  
关闭系统调用以后，那还不是随便我们 hook？  
所以理论上只要给 ntdll.dll 的导出表做点手脚即可  
理论可行，开始动手

2.3 bypasss 代码
--------------

首先，重复造轮子的事不要做，针对 vmp 的那些常见的反调，已经有很多大佬写好插件并开源了，  
我在这里拿这个 x64dbg 官方的插件做例子 [https://github.com/x64dbg/ScyllaHide](https://github.com/x64dbg/ScyllaHide)  
找到 x64dbg 插件的代码 ScyllaHide\InjectorCLI\ApplyHooking.cpp  
在其中插入一段新的代码

```
void AddWineFunctionName(HANDLE hProcess)
{
    BYTE* remote_ntdll = (BYTE*)GetModuleBaseRemote(hProcess, L"ntdll.dll");
    // check input
    if (!remote_ntdll)
        return;
 
    SIZE_T readed = 0;
    // check module's header
    IMAGE_DOS_HEADER dos_header;
    ReadProcessMemory(hProcess, remote_ntdll, &dos_header, sizeof(IMAGE_DOS_HEADER), &readed);
    if (dos_header.e_magic != IMAGE_DOS_SIGNATURE)
        return;
 
    // check NT header
    IMAGE_NT_HEADERS pe_header;
    ReadProcessMemory(hProcess, (BYTE*)remote_ntdll + dos_header.e_lfanew, &pe_header, sizeof(IMAGE_NT_HEADERS), &readed);
    if (pe_header.Signature != IMAGE_NT_SIGNATURE)
        return;
 
    // get the export directory
    DWORD export_adress = pe_header.OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress;
    if (!export_adress)
        return;
 
    DWORD export_size = pe_header.OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].Size;
 
    BYTE* new_export_table = (BYTE*)VirtualAllocEx(hProcess, remote_ntdll + 0x1000000, export_size + 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
 
    IMAGE_EXPORT_DIRECTORY export_directory;
    ReadProcessMemory(hProcess, remote_ntdll + export_adress, &export_directory, sizeof(IMAGE_EXPORT_DIRECTORY), &readed);
 
    BYTE* tmp_table = (BYTE*)malloc(export_size + 0x1000);
    if (tmp_table == nullptr)return;
 
    //copy functions table
    BYTE* new_functions_table = new_export_table;
    ReadProcessMemory(hProcess, remote_ntdll + export_directory.AddressOfFunctions, tmp_table, export_directory.NumberOfFunctions * sizeof(DWORD), &readed);
    WriteProcessMemory(hProcess, new_functions_table, tmp_table, export_directory.NumberOfFunctions * sizeof(DWORD), &readed);
    g_log.LogInfo(L"[VMPBypass] new_functions_table: %p", new_functions_table);
 
    //copy ordinal table
    BYTE* new_ordinal_table = new_functions_table + export_directory.NumberOfFunctions * sizeof(DWORD) + 0x100;
    ReadProcessMemory(hProcess, remote_ntdll + export_directory.AddressOfNameOrdinals, tmp_table, export_directory.NumberOfNames * sizeof(WORD), &readed);
    WriteProcessMemory(hProcess, new_ordinal_table, tmp_table, export_directory.NumberOfNames * sizeof(WORD), &readed);
    g_log.LogInfo(L"[VMPBypass] new_ordinal_table: %p", new_ordinal_table);
     
    //copy name table
    BYTE* new_name_table = new_ordinal_table + export_directory.NumberOfNames * sizeof(WORD) + 0x100;
    ReadProcessMemory(hProcess, remote_ntdll + export_directory.AddressOfNames, tmp_table, export_directory.NumberOfNames * sizeof(DWORD), &readed);
    WriteProcessMemory(hProcess, new_name_table, tmp_table, export_directory.NumberOfNames * sizeof(DWORD), &readed);
    g_log.LogInfo(L"[VMPBypass] new_name_table: %p", new_name_table);
 
    free(tmp_table);
    tmp_table = nullptr;
 
    //setup new name  & name offset
    BYTE* wine_func_addr = new_name_table + export_directory.NumberOfNames * sizeof(DWORD) + 0x100;
    WriteProcessMemory(hProcess, wine_func_addr, "wine_get_version\x00", 17, &readed);
    DWORD wine_func_offset = (DWORD)(wine_func_addr - remote_ntdll);
    WriteProcessMemory(hProcess, new_name_table + export_directory.NumberOfNames * sizeof(DWORD), &wine_func_offset, 4, &readed);
 
    //set fake ordinal
    WORD last_ordinal = export_directory.NumberOfNames;
    WriteProcessMemory(hProcess, new_ordinal_table + export_directory.NumberOfNames * sizeof(WORD), &last_ordinal, 2, &readed);
 
    //set fake function offset
    BYTE* query_information_process = reinterpret_cast(GetProcAddress(hNtdll, "NtCurrentProcess"));
    DWORD function_offset = (DWORD)(query_information_process - remote_ntdll);
    WriteProcessMemory(hProcess, new_functions_table + export_directory.NumberOfFunctions * sizeof(DWORD), &function_offset, 4, &readed);
 
    //setup new directory
    export_directory.NumberOfNames++;
    export_directory.NumberOfFunctions++;
 
    DWORD name_table_offset = (DWORD)(new_name_table - remote_ntdll);
    export_directory.AddressOfNames = name_table_offset;
 
    DWORD function_talble_offset = (DWORD)(new_functions_table - remote_ntdll);
    export_directory.AddressOfFunctions = function_talble_offset;
 
    DWORD ordinal_table_offset = (DWORD)(new_ordinal_table - remote_ntdll);
    export_directory.AddressOfNameOrdinals = ordinal_table_offset;
 
    //// change the offset of header data
    DWORD old_prot;
    VirtualProtectEx(hProcess, remote_ntdll + export_adress, sizeof(IMAGE_EXPORT_DIRECTORY), PAGE_EXECUTE_READWRITE, &old_prot);
    WriteProcessMemory(hProcess, remote_ntdll + export_adress, &export_directory, sizeof(IMAGE_EXPORT_DIRECTORY), &readed);
    VirtualProtectEx(hProcess, remote_ntdll + export_adress, sizeof(IMAGE_EXPORT_DIRECTORY), old_prot, &old_prot);
} 
```

通过这段代码，复制了一份 ntdll.dll 的导出表，并在里边添加了 wine_get_version 的导出项，实际调用其实是调用的 NtCurrentProcess。  
然后，调用这个函数  
![](https://bbs.kanxue.com/upload/attach/202406/995602_79H3RYQMAUV7MPF.webp)  
然后编译产物，放到 x64dbg 插件目录  
![](https://bbs.kanxue.com/upload/attach/202406/995602_8RU83DNSR9JESKV.webp)  
至此，vmp 的反调保护已经被我们从纯纯的 r3 bypass 了（狗头）  
![](https://bbs.kanxue.com/upload/attach/202406/995602_E67GBBEVBRN6CAF.webp)

3. 结语
=====

加个导出表还是个比较简单的操作，vmp 想要兼容 wine 环境，但是又只进行了很简单的校验，而且源码又泄露了，这才给了我们可乘之机。  
此外 vmp 源码种还有很多别的地方值得学习，这点下次有机会再说。  
ps： 我本次实验用的 vmp 版本是 最我能找到的最高版本的 3.8.4

[[培训] 科锐软件逆向 50 期预科班报名即将截止，速来！！！ 50 期正式班报名火爆招生中！！！](https://mp.weixin.qq.com/s/HFghXQRTiTlk6oRKGotpHA)

最后于 1 天前 被 JoJoRun 编辑 ，原因： 勘误