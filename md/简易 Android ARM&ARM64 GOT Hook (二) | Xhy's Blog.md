> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.xhyeax.com](https://blog.xhyeax.com/2021/08/30/android-arm-plt-hook/)

> 概述前文简易 Android ARM&ARM64 GOT Hook (一)，基于链接视图，解析 section 查找 GOT 表偏移值，无法应对 section 被处理过的情况（如加固）。

[](#概述 "概述")概述
--------------

前文[简易 Android ARM&ARM64 GOT Hook (一)](https://blog.xhyeax.com/2021/08/23/android-arm-got-hook/)，基于链接视图，解析 section 查找 GOT 表偏移值，无法应对 section 被处理过的情况（如加固）。

本文基于 ELF 的执行视图 (`Execution View`)，讨论`Android` `ARM&ARM64`架构的`GOT/PLT Hook`（仍以 Hook 公共库`libc.so`的`getpid`函数为例）。

[](#思路 "思路")思路
--------------

1.  通过 maps 文件获取模块基址，解析内存中的 ELF。
2.  查找并解析`.dynamic`段，得到`.rel.plt`、`.rel.dyn`、`.dynsym`、`.dynstr`和`.hash`表。
3.  通过`.hash -> .dynsym -> .dynstr`查找导入符号，再遍历`.rel.plt`和`.rel.dyn`，得到函数偏移。
4.  基址加上偏移得到内存地址，修改函数地址即可。

注入方式及编译环境同[前文](https://blog.xhyeax.com/2021/08/23/android-arm-got-hook/)，不再赘述。

[](#具体实现 "具体实现")具体实现
--------------------

### [](#核心代码 "核心代码")核心代码

```
uintptr_t hackBySegment(const char *moudle_path, const char *target_lib, const char *target_func,
                        uintptr_t replace) {
    LOGI("hackDynamic start.\n");
    
    void *handle = dlopen(target_lib, RTLD_LAZY);
    auto ori = (uintptr_t) dlsym(handle, target_func);
    LOGI("hackDynamic getpid addr: %lX\n", ori);
    
    uintptr_t replaceAddr = getSymAddrDynamic(moudle_path, target_func);
    
    replaceFunction(replaceAddr, replace, ori);
    return ori;
}
```

完整代码见 [AndroidGotHook](https://github.com/XhyEax/AndroidGotHook)

### [](#获取-dynamic段 "获取.dynamic段")获取. dynamic 段

解析 maps 文件获取到模块地址后，基于执行视图解析 ELF。

从`elf header`中得到`program header table`的起始偏移（`e_phoff`）、`program header`大小 (`e_phentsize`) 和总`program header`个数（`e_phnum`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-elf-header.png)

遍历`program header table`，查找`p_type`为`PT_DYNAMIC`的`program header`，得到`.dynamic`段在内存中的偏移值（`p_vaddr`）和大小（`p_memsz`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-ph-dynamic.png)

将模块基址与`.dynamic`偏移值相加，得到`.dynamic`段的起始地址。

### [](#解析-dynamic段 "解析.dynamic段")解析. dynamic 段

遍历`.dynamic`段，根据`d_tag`解析不同的表。  
`.rel.plt`：`DT_JMPREL`（大小储存在`DT_PLTRELSZ`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-relplt.png)  
`.rel.dyn`：`DT_REL`（大小储存在`DT_RELSZ`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-reldyn.png)  
`.dynsym`：`DT_SYMTAB`  
`.dynstr`：`DT_STRTAB`  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-dynsym-str.png)  
`.hash`：`DT_HASH`  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-hash.png)

注：解析`.dynamic`的 ELF 模板来自 [Android7.0 以上命名空间详解 (dlopen 限制) 附上 010editor 模块](https://www.52pojie.cn/thread-948942-1-1.html)

### [](#查找符号 "查找符号")查找符号

通过`.hash -> .dynsym -> .dynstr`查找符号。

```
ELFW(Sym) *target = nullptr;
uint32_t hash = elf_sysv_hash((const uint8_t *) symName);
for (uint32_t i = buckets[hash % buckets_cnt];
        0 != i; i = chains[i]) {
    ELFW(Sym) *sym = dynsym + i;
    unsigned char type = ELF_ST_TYPE(sym->st_info);
    if (STT_FUNC != type && STT_GNU_IFUNC != type && STT_NOTYPE != type)
        continue; 
    if (0 == strcmp(dynstr + sym->st_name, symName)) {
        target = sym;
        break;
    }
}
```

具体操作为：

1.  计算符号名称的哈希值，然后从`.hash`获取符号在`.dynsym`中的索引。
    
    > 接受符号名称的散列函数会返回一个值，用于计算 bucket 索引。因此，如果散列函数为某个名称返回值 x，则 bucket [x% nbucket] 将会计算出索引 y。此索引为符号表和链表的索引。如果符号表项不是需要的名称，则 chain[y] 将使用相同的散列值计算出符号表的下一项。
    
2.  通过索引从`.dynsym`获取符号，根据符号类型和符号名（通过`st_name`从`.dynstr`获取字符串），判断与目标函数是否匹配。
    

PS：也可直接遍历`.dynsym`并比对字符串，时间复杂度更高

> .hash -> .dynsym -> .dynstr，时间复杂度：O(x) + O(1) + O(1)  
> .dynsym -> .dynstr，时间复杂度：O(n) + O(1)

### [](#计算内存地址 "计算内存地址")计算内存地址

遍历`.rel.plt`和`.rel.dyn`，比对符号和重定位类型，获取函数偏移值（`r_offset`）。与模块基址相加得到内存地址。

```
for (int i = 0; i < rel_plt_cnt; i++) {
        ELFW(Rel) &rel = rel_plt[i];
        if (&(dynsym[ELF_R_SYM(rel.r_info)]) == target &&
            ELF_R_TYPE(rel.r_info) == ELF_R_JUMP_SLOT) {

            return moduleBase + rel.r_offset;
        }
    }
    for (int i = 0; i < rel_dyn_cnt; i++) {
        ELFW(Rel) &rel = rel_dyn[i];
        if (&(dynsym[ELF_R_SYM(rel.r_info)]) == target &&
            (ELF_R_TYPE(rel.r_info) == ELF_R_ABS
             || ELF_R_TYPE(rel.r_info) == ELF_R_GLOB_DAT)) {

            return moduleBase + rel.r_offset;
        }
    }
```

之后替换该内存地址对应的函数即可，同[前文](https://blog.xhyeax.com/2021/08/23/android-arm-got-hook/)。

### [](#适配ARM64 "适配ARM64")适配 ARM64

```
#if defined(__LP64__)
#define ELFW(what) Elf64_ ## what
#define ELF_R_TYPE(what) ELF64_R_TYPE(what)
#define ELF_R_SYM(what) ELF64_R_SYM(what)
#else
#define ELFW(what) Elf32_ ## what
#define ELF_R_TYPE(what) ELF32_R_TYPE(what)
#define ELF_R_SYM(what) ELF32_R_SYM(what)
#endif

#if defined(__arm__)
#define ELF_R_JUMP_SLOT R_ARM_JUMP_SLOT     
#define ELF_R_GLOB_DAT  R_ARM_GLOB_DAT      
#define ELF_R_ABS       R_ARM_ABS32         
#elif defined(__aarch64__)
#define ELF_R_JUMP_SLOT R_AARCH64_JUMP_SLOT
#define ELF_R_GLOB_DAT  R_AARCH64_GLOB_DAT
#define ELF_R_ABS       R_AARCH64_ABS64
#endif
```

[](#测试 "测试")测试
--------------

同[前文](https://blog.xhyeax.com/2021/08/23/android-arm-got-hook/)，替换动态库即可测试。

[](#日志 "日志")日志
--------------

### [](#ARM "ARM")ARM

![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-plt-hook-log.png)

### [](#ARM64 "ARM64")ARM64

![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm64-plt-hook-log.png)

[](#作为动态链接库 "作为动态链接库")作为动态链接库
-----------------------------

详见[项目](https://github.com/XhyEax/AndroidGOTHook)的`vitcim_app`模块。

通过配置`CMakeLists.txt`将`libinject.so`作为动态库链接，在`JNI_OnLoad`调用`hackBySegment`替换`getpid`函数。

日志如下：  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm64-so-plt-hook-log.png)

[](#总结 "总结")总结
--------------

主要学习了`.dynamic`段的解析和导入符号的查找方式。虽然实现了基于执行视图解析 ELF，但仍存在许多不足。不过这次是真的短期内不会再改动了。（**不要重复造轮子**）

[](#参考 "参考")参考
--------------

[基于 Android 的 ELF PLT/GOT 符号和重定向过程 ELF Hook 实现](https://blog.csdn.net/L173864930/article/details/40507359)  
[ELF 文件格式与 got 表 hook 简单实现](https://bbs.pediy.com/thread-267842.htm)  
[重定位节 - 链接程序和库指南](https://docs.oracle.com/cd/E26926_01/html/E25910/chapter6-54839.html)  
[符号表节 - 链接程序和库指南](https://docs.oracle.com/cd/E26926_01/html/E25910/chapter6-79797.html)  
[散列表节 - 链接程序和库指南](https://docs.oracle.com/cd/E26926_01/html/E25910/chapter6-48031.html)  
[动态节 - 链接程序和库指南](https://docs.oracle.com/cd/E26926_01/html/E25910/chapter6-42444.html)  
[ELF 文件结构详解](https://bbs.pediy.com/thread-255670.htm)  
[PLT HOOK](https://zhuanlan.zhihu.com/p/269441842)  
[bhook](https://github.com/bytedance/bhook)  
[xhook](https://github.com/iqiyi/xhook)