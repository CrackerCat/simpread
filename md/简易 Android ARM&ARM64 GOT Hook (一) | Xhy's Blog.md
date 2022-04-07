> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.xhyeax.com](https://blog.xhyeax.com/2021/08/23/android-arm-got-hook/)

> 更新简易 Android ARM&ARM64 GOT Hook (二) 基本思路为：基于执行视图，解析内存中的 ELF，查找导入符号并替换函数地址 概述本文以 Hook 公共库 libc.so 的 getpid 函数为例，基于 ELF 的链接视图 (Linking View)，讨论 Android ARM&ARM64 架构的 GOT/PLT Hook。

### [](#更新 "更新")更新

[简易 Android ARM&ARM64 GOT Hook (二)](https://blog.xhyeax.com/2021/08/30/android-arm-plt-hook/)  
基本思路为：基于执行视图，解析内存中的 ELF，查找导入符号并替换函数地址

[](#概述 "概述")概述
--------------

本文以 Hook 公共库`libc.so`的`getpid`函数为例，基于 ELF 的链接视图 (`Linking View`)，讨论`Android` `ARM&ARM64`架构的`GOT/PLT Hook`。

[](#原理 "原理")原理
--------------

程序`加载`后，在执行之前，需要先进行`动态链接`，并进行`重定位`。

调用外部函数时，需要先跳转到 PLT(`Procedure Link Table` 程序链接表，位于代码段)，再跳转到 GOT(`Global Offset Table` 全局偏移表，位于数据段)，执行目标函数。

延迟绑定 (`Lazy Binding`)：当外部函数被调用时，才进行地址解析和重定位

**由于 Android ARM 架构不支持延迟绑定，在 linker 重定位后，GOT 已被填充为内存地址** (可使用 IDA 动态调试验证)

因此，可以通过比对函数地址，修改指定模块的对应 GOT 表项，实现对外部导入函数的 Hook

[](#具体思路 "具体思路")具体思路
--------------------

编写 so 库，在 so 加载的构造函数（`linker`会主动调用）中完成以下操作：定位目标模块基址、基于链接视图解析 ELF 文件，得到 GOT 表地址及大小、遍历 GOT 表替换目标函数地址。

[](#注入方式 "注入方式")注入方式
--------------------

使用`LIEF`修改 ELF 文件，导入该动态库。

[](#编译环境 "编译环境")编译环境
--------------------

`Android Studio 2020.3.1`、`Gradle 7.0.1`、`CMake 3.18.1`、`NDK 23.0.7599858`  
`ABI`: `armeabi-v7a,arm64-v8a`

PS：其实也可以脱离`Android Studio`手动编译，见[使用 CMake 交叉编译 Android ARM 程序](https://blog.xhyeax.com/2021/08/22/arm-compile-without-as/)

[](#编码 "编码")编码
--------------

使用`Android Studio`创建`Native C++`项目，然后增加两个`Android Native Library`模块，分别命名为`victim`和`inject`

### [](#待注入程序 "待注入程序")待注入程序

#### [](#victim-cpp "victim.cpp")victim.cpp

```
#include <stdio.h>
#include <unistd.h>

int main()
{
    printf("my pid: %d\n", getpid());
    
    return 0;
}
```

调用`getpid`获取进程 id（IDA 动态调试时使用`getchar`，方便查看内存）

#### [](#CMakeLists-txt "CMakeLists.txt")CMakeLists.txt

```
cmake_minimum_required(VERSION 3.18.1)
project(victim)

add_executable(victim victim.cpp)
```

编译为可执行文件

### [](#注入动态库 "注入动态库")注入动态库

```
uintptr_t hackBySection(const char *moudle_path, const char *target_lib, const char *target_func,
                        uintptr_t replace) {
    LOGI("hack start.\n");
    
    void *handle = dlopen(target_lib, RTLD_LAZY);
    auto ori = (uintptr_t) dlsym(handle, target_func);
    LOGI("hack ori addr: %lX\n", ori);
    int GOTSize = 0;
    
    uintptr_t GOTBase = getGOTBase(GOTSize, moudle_path);
    
    uintptr_t replaceAddr = getSymAddrInGOT(GOTBase, GOTSize, ori);
    
    replaceFunction(replaceAddr, replace, ori);
    return ori;
}


int (*getpidOri)();


int getpidReplace() {
    LOGI("before hook getpid\n");
    
    int pid = (int) getpidOri();
    LOGI("after hook getpid: %d\n", pid);
    return 233333;
}


void __attribute__((constructor)) init() {
    uintptr_t ori = hackBySection(MODULE_PATH, "libc.so", "getpid",
                                  (uintptr_t) getpidReplace);
    getpidOri = (int (*)()) (ori);
}
```

注意：`Android 7.0`以上，`dlopen`只能加载公共库，加载非公共库需要绕过命名空间限制。

### [](#子函数 "子函数")子函数

#### [](#获取模块基址 "获取模块基址")获取模块基址

```
uintptr_t getModuleBase(const char *modulePath) {
    uintptr_t addr = 0;
    char buff[256] = "\n";

    FILE *fp = fopen("/proc/self/maps", "r");
    while (fgets(buff, sizeof(buff), fp)) {
        if (strstr(buff, "r-xp") && strstr(buff, modulePath) &&
            sscanf(buff, "%lx", &addr) == 1)
            return addr;
    }

    LOGI("[%s] moduleBase not found!\n", modulePath);
    fclose(fp);
    return 0;
}
```

遍历`/proc/self/maps`文件内容，根据权限及模块路径找到基址。

#### [](#获取GOT表地址及其大小 "获取GOT表地址及其大小")获取 GOT 表地址及其大小

篇幅所限，此处仅给出思路（以`ARM`为例），完整代码见 [AndroidGotHook](https://github.com/XhyEax/AndroidGotHook)

根据模块路径打开 ELF 文件（本例为`/data/local/tmp/victim-patch-arm`），解析 ELF 文件结构（图片为`010Editor` ELF 模板解析结果）:

首先从`elf header`中得到`section header table`的起始偏移（`e_shoff`）、字符串表索引（`e_shstrndx`）、`section header`大小 (`e_shentsize`) 和总`section header`个数（`e_shnum`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-got-hook-elf-header.png)

然后计算出字符串表`section header`的偏移地址（`e_shoff` + `e_shstrndx` * `e_shentsize`），从而得到字符串表的偏移值（`s_offset`）及大小（`s_size`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-got-hook-elf-str.png)

再遍历`section header table`，查找`sh_type`为`SHT_PROGBITS` 且 section 名（通过`sh_name`查询字符串表）为`.got`的`section header`，得到 GOT 表偏移值（`s_offset`）及大小（`s_size`）  
![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-got-hook-elf-got.png)

最终将 GOT 表偏移值与模块基址相加，得到 GOT 表地址。

#### [](#查找目标函数地址 "查找目标函数地址")查找目标函数地址

```
uintptr_t getSymAddrInGOT(uintptr_t GOTBase, int GOTSize, uintptr_t ori) {
 if (GOTBase == 0) {
        LOGI("getSymAddrInGOT failed! addr [%lX] is wrong\n", GOTBase);
        return 0;
    }

    for (int i = 0; i < GOTSize; ++i) {
        uintptr_t addr = GOTBase + i * 4;
        uintptr_t item = *(uintptr_t *) (addr);

        if (item == ori) {
            return addr;
        }
    }

    LOGI("getSymAddrInGOT %lX not found!\n", ori);
    return 0;
}
```

遍历 GOT 表，查找目标函数地址。

#### [](#替换函数地址 "替换函数地址")替换函数地址

```
void replaceFunction(uintptr_t addr, uintptr_t replace, uintptr_t ori) {
    if (addr == 0) {
        LOGI("replace failed! addr is wrong\n");
        return;
    }
    
    uintptr_t item = *(uintptr_t *) (addr);
    if (item == replace) {
        LOGI("function has been replaced!\n");
        return;
    }
    if (item != ori) {
        LOGI("replace failed! unexpected function address %X\n", item);
        return;
    }
    
    LOGI("replace %X to %X\n", ori, replace);
    mprotect((void *) PAGE_START(addr), PAGE_SIZE, PROT_READ | PROT_WRITE);
    *(uintptr_t *) addr = replace;
    __builtin___clear_cache((char *) PAGE_START(addr), (char *) PAGE_END(addr));
}
```

首先判断函数是否已被替换，然后与期望值相比较，如果一致，则进行以下操作：  
将该地址权限设置为可读可写，然后替换函数地址，并清空指令缓存。

#### [](#适配ARM64 "适配ARM64")适配 ARM64

添加宏，将`Elf32_w`替换为`ELFW(w)`：

```
#if defined(__LP64__)
#define ELFW(what) Elf64_ ## what
#else
#define ELFW(what) Elf32_ ## what
#endif
```

根据架构使用不同的路径：

```
#if defined(__LP64__)
#define MODULE_PATH "/data/local/tmp/victim-patch-arm64"
#else
#define MODULE_PATH "/data/local/tmp/victim-patch-arm"
#endif
```

[](#使用LIEF添加依赖 "使用LIEF添加依赖")使用 LIEF 添加依赖
----------------------------------------

编译完成后，打开`victim/build/intermediates/cmake/debug/obj/CPU架构/`，使用`LIEF`注入`victim`

```
import lief,sys

abi = "armeabi-v7a"
tail = "-arm"
if(len(sys.argv) > 1 and sys.argv[1] == "arm64-v8a"):
    abi = sys.argv[1]
    tail = "-arm64"

path = "../victim/build/intermediates/cmake/debug/obj/"+abi+"/victim"
elf = lief.parse(path)
elf.add_library("/data/local/tmp/libinject"+tail+".so")
elf.write("victim-patch"+tail)
print("patch success")
```

[](#测试 "测试")测试
--------------

打开`inject/build/intermediates/cmake/debug/obj/CPU架构/`，  
使用`adb`将生成的`libinject.so`和`victim-patch-arm`发送到手机的`/data/local/tmp/`目录（so 需要重命名）

```
adb push libinject.so /data/local/tmp/libinject-arm.so
adb push victim-patch-arm /data/local/tmp
```

设置可执行权限后，运行`victim-patch-arm`

```
adb shell chmod +x /data/local/tmp/victim-patch-arm
adb shell /data/local/tmp/victim-patch-arm
```

（`ARM64`同理）

[](#运行结果 "运行结果")运行结果
--------------------

![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-got-hook-output.png)

成功实现对`getpid`函数的`GOT Hook`。

[](#日志 "日志")日志
--------------

### [](#ARM "ARM")ARM

![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm-got-hook-log.png)

### [](#ARM64 "ARM64")ARM64

![](http://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/arm64-got-hook-log.png)

[](#存在的问题 "存在的问题")存在的问题
-----------------------

1.  未绕过`dlopen`命名空间限制，在`Android 7`以上无法打开非公共库
2.  未 hook`dlopen`，无法实时修改加载模块的 GOT 表
3.  解析 maps 获取模块基址，兼容性可能存在一定问题
4.  基于链接视图静态解析 ELF，无法处理加壳的 so
5.  静态注入可执行文件，无法绕过完整性检测
6.  未提供卸载函数，无法恢复 GOT 表
7.  模块路径硬编码，通用性不足
8.  …

[](#总结 "总结")总结
--------------

通过本项目，学习了`GOT Hook`原理和 ELF 文件结构，并适配了`ARM64`架构，目的基本达到。虽然功能还不够完善，但~短期内应该不会再改动了~（俗话说得好：不要重复造轮子）。

实际应用可以考虑使用字节的 [bhook](https://github.com/bytedance/bhook)

[](#参考 "参考")参考
--------------

[android 中基于 plt/got 的 hook 实现原理](https://blog.csdn.net/byhook/article/details/103500524)  
[聊聊 Linux 动态链接中的 PLT 和 GOT(2)——延迟重定位](https://blog.xhyeax.com/2021/08/23/android-arm-got-hook/%5Bhttps://linyt.blog.csdn.net/article/details/51636753%5D)  
[constructor 属性函数在动态库加载中的执行顺序](https://zhuanlan.zhihu.com/p/108274829)  
[Android7.0 以上命名空间详解 (dlopen 限制)](https://www.52pojie.cn/thread-948942-1-1.html)  
[Android 中 GOT 表 HOOK 手动实现](https://blog.csdn.net/u011247544/article/details/78564564)  
[Android GOT Hook](https://www.cnblogs.com/mmmmar/p/8228391.html)