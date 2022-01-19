> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271211.htm)

> [原创]Windows Kernel Programming 笔记 1~5 内核开发入门

Windows Kernel Programming 笔记 1~5 内核开发入门
========================================

1 windows 内部概况
--------------

描述一些 Windows 内部工作中最重要、最基本的概念，部分概念将在后面的章节做更详细的研究

### 1.1 进程

进程不运行（Processes dont't run - processes manage），线程才执行代码

 

进程拥有以下内容：

*   一个可执行程序（PE 文件），包括代码和数据
*   私有的虚拟内存空间
*   主令牌（primary token），是一个对象，存储进程默认安全上下文
*   对象（事件、信号、文件）句柄表
*   一个或多个线程（没有线程的用户态进程一般情况下会被内核销毁）

### 1.2 虚拟内存

每个进程拥有自己的**虚拟、私有、线性地址**空间  
（该地址空间初始时几乎为空，然后 pe、ntdll.dll 开始被影射，接着是其他子系统 dll）

 

32 位进程默认地址空间 **2GB**，设置 pe 中的`LARGEADDRESSAWARE`标志可以增加到 **3GB**（32 位系统）或 **4GB**（64 位系统）

 

64 位进程默认地址空间 **128TB**（win8 之前是 8TB）

 

虚拟内存被映射到物理内存（RAM）或临时驻留在文件中（如 page file）  
如果不在物理内存，则触发 page fault 异常，并或取数据到物理内存中

 

**页（page）**是内存管理的单位，默认大小为 **4KB**

#### 页状态

虚拟内存中的页处于三种状态之一

*   Free：未分配
*   Committed：已分配，通常映射到 RAM 或文件（例如 page file）
*   Reserved：未分配，对 cpu 而言与 Free 相似，自动分配将不会使用该页  
    一个例子是线程栈（thread stack）

#### 系统内存

系统空间与进程无关

 

系统空间就是内核

### 1.3 线程

实际执行代码的是线程

 

线程拥有的最重要的内容：

*   当前访问模式（用户或内核）
*   执行上下文
*   一个或两个栈（stack）
*   Thread Local Storage（TLS）
*   基本优先级和当前（动态）优先级
*   处理器关联信息

线程最常处于的状态：

*   Running：在逻辑处理器运行中
*   Ready：等待运行（所有处理器在忙或不可用）
*   Waiting：等待某个事件，事件触发就变成 Ready

括号中的数字是状态号：

 

Running(2) => Waiting(5) => Deferred Ready(7), Ready(1) => Running(2)

#### 1.3.1 线程栈

线程至少有一个位于内核空间的栈（32 位系统 12KB，64 位系统 24KB）

 

用户态的线程还有一个位于所属进程空间的栈（默认上限 1MB）

 

线程`Running`或`Ready`时，内核栈驻留在 RAM

 

栈初始时会尽可能少提交页（最少一页），剩下的页设置为`Reserved`，而最后一个`Committed`的页的下一页设置为`PAGE_GUARD`

### 1.4 系统调用（又名系统服务）

原标题：System Services (a.k.a. System Calls)

 

R3 代码通过系统调用完成一些只能在 R0 下完成的功能，如分配内存、打开文件、创建线程等

 

大致流程是：  
调用 subsystem dll（如 kernel32.dll）中的文档化 api（如 CreateFile）  
进入 NTDLL 中的 Native Api（如 NtCreateFile）  
进入内核中的系统服务分发函数  
进入 Native Api 对应的内核中的函数

 

Native Api 将调用号存入 eax 然后进入 r0 的系统服务分发函数，eax 实际是 SSDT（System Service Dispatch Table）的下标

### 1.5 通用系统架构

![](https://bbs.pediy.com/upload/attach/202201/907036_A35UQENYZESHCBA.jpg)

### 1.6 句柄和对象

对象被引用计数，当计数为 0 时才会被释放

 

句柄是进程的对象表的索引

> 注意：返回值为句柄的函数，大多数失败时返回`0`。有些返回`INVALID_HANDLE_VALUE (-1)`，比如`CreateFile`

 

句柄值是 4 的倍数，0 不是有效句柄值

#### 1.6.1 对象名

某些类型的对象可以有名称，可用于通过合适的 Open 函数按名称打开对象。

 

用户模式调用 Create 函数按名称创建对象，如果存在，则仅打开现有对象。

 

winObj 中显示的名称有时不是对象的真实名称：

*   进程和线程显示 ID
*   文件对象显示文件名（或设备名），因为共享的原因，无法通过文件名获得文件对象句柄
*   （注册表）键对象与注册表的路径一起显示，原因同文件对象
*   目录对象显示路径，目录不是文件系统对象，而是对象管理器目录，可通过 Sysinternals WinObj 查看
*   令牌对象名称与存储在令牌中的用户名一起显示

#### 1.6.2 访问现有对象

Process Explorer 的句柄视图中的访问列显示用于打开或创建句柄的访问掩码

 

Process Explorer 中显示的引用数（References）不是实际引用数（outstanding references）

 

[windbg] 中用`!trueref`获取实际引用数（actual reference）

2 内核开发入门
--------

本章主要是关于准备内核开发所需的环境，包括开发和调试的工具以及环境配置

 

以及启动和运行内核驱动的知识

 

然后写一个可以加载和卸载的驱动

### 驱动开发准备工作

首先按 2.1 安装工具 完成安装，然后为驱动开发配置虚拟机（未包括内核调试的配置）

 

**安装无签名驱动**

 

如果驱动没有签名，安装驱动需要以该模式启动系统

```
bcdedit /set testsigning on

```

**显示内核调试信息**

 

在`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager`添加一个名为`Debug Print Filter`的键  
在键中添加一个`DWORD`，名为`DEFAULT`，值为`8`

 

**虚拟机文件共享**

 

实际操作时，安装在虚拟机中（避免本机崩溃），需要共享项目的文件给虚拟机

 

共享本机的 vs 解决方案文件夹 给虚拟机，名称为`MyDriver`

 

虚拟机中的 debug 输出路径为：`\\vmware-host\Shared Folders\MyDriver\x64\Debug`

 

**驱动调试工具**

 

安装完 WDK 后，把`C:\Program Files (x86)\Windows Kits\10\Tools\x64`这个目录复制到虚拟机中，这个是 x64 下的驱动开发调试工具，比如用于查看内存池的 poolmon

### 2.1 安装工具

需要 vs2019、windows 10 sdk（vs2019 中安装）、windows 10 driver kit（WDK）

 

以及 Sysinternals，该工具包含 debug view、process monitor 等一系列有用的工具

> 在实际编译中发现，新版本的 vs 驱动项目默认开启缓解 Spectre 漏洞
> 
> 可以在 c/c++、代码生成中关闭该项，或在 vs installer 中安装对应工具

### 2.2 创建一个驱动工程

vs2019 中选择创建一个`Empty WDM Driver`，创建完成后有个`inf`后缀的文件，暂时不需要，删除掉

### 2.3 DriverEntry 和 Unload Routines

DriverEntry 是驱动的默认入口点

 

系统线程以`IRQL_PASSIVE_LEVEL`(0) 调用 DriverEntry

 

DriverEntry 函数原型：

```
extern "C"
NTSTATUS
DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath);

```

一个简单的驱动（sample.cpp）：

```
#include void SampleUnload(_In_ PDRIVER_OBJECT DriverObject) {
    UNREFERENCED_PARAMETER(DriverObject);
}
 
extern "C"
NTSTATUS
DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);
 
    DriverObject->DriverUnload = SampleUnload;
 
    return STATUS_SUCCESS;
} 
```

### 2.4 安装驱动

安装驱动和安装用户态服务相似，需要调用 Create Service API 或使用工具

 

sc.exe（系统自带）是著名工具之一

 

安装驱动需要管理员权限

 

创建服务项：

```
sc create sample type= kernel binPath= "\\vmware-host\Shared Folders\MyDriver\x64\Debug\sample.sys"

```

随后就能在注册表（regedit.exe）的`HKLM\System\CurrentControlSet\Services\Sample`中看到该项

> 注册表项位置：HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Sample
> 
> 假设`binPath= c:\`，注册表项`ImagePath= \??\c:\`
> 
> 假设`binPaht= "\\vmware-hots\"`，注册表项`ImagePath= \??\UNC\vmware-hots\`

 

加载驱动（启动服务）：

```
sc start sample

```

在 process explorer 中，选择 System 进程，查看 dll 窗口，拉到最下面就能看到 sample.sys

 

卸载驱动（停止服务）：

```
sc stop sample

```

### 2.5 简单跟踪（S1）

`KdPrint 宏`是`DbgPrint API`的包装

 

通过在每个函数开头加入`KdPrint(("Debug messgae"));`可以观察函数调用的发生

 

使用 DebugView，选择 capture Kernel 可以看到内核调试信息

### 2.6 练习：显示系统信息（E1）

创建一个驱动用于显示系统版本信息，使用`RtlGetVersion`

 

code:

```
// Get Version
RTL_OSVERSIONINFOW versionInfo = { 0, };
versionInfo.dwOSVersionInfoSize = sizeof(RTL_OSVERSIONINFOW);
RtlGetVersion(&versionInfo);
 
// Print
DbgPrint("[E1] Major:%d\n[E1] Minor:%d\n[E1] Build:%d",
    versionInfo.dwMajorVersion,
    versionInfo.dwMinorVersion,
    versionInfo.dwBuildNumber);

```

shell:

```
sc create E1_OSVersion type= kernel binPath= "\\vmware-host\Shared Folders\MyDriver\x64\Debug\E1_OSVersion.sys"
sc start E1_OSVersion
sc stop E1_OSVersion

```

3 内核编程基础
--------

研究一些内核的 API、结构和定义，以及一些驱动程序中的机制

### 3.1 通用内核编程指南

用户模式和内核模式调试的重要区别

<table><thead><tr><th></th><th>用户模式</th><th>内核模式</th></tr></thead><tbody><tr><td>未处理异常</td><td>进程崩溃</td><td>系统崩溃</td></tr><tr><td>终止</td><td>当进程终止，所有内存和资源都会被自动释放</td><td>当驱动卸载，如果没有手动释放，会造成泄露直到重启</td></tr><tr><td>返回值</td><td>API 错误有时候会忽略</td><td>应该不忽略任何错误</td></tr><tr><td>IRQL</td><td>总是 PASSIVE_LEVEL (0)</td><td>可能为更高</td></tr><tr><td>错误代码</td><td>通常只会影响本进程</td><td>影响整个系统</td></tr><tr><td>测试和调试</td><td>通常在开发机器上调试</td><td>需要双机调试</td></tr><tr><td>库（Lib）</td><td>可以使用 C/C++ 库（如 STL、boost）</td><td>大多数标准库无法使用</td></tr><tr><td>异常处理</td><td>可以使用 C++ 异常或 SEH</td><td>只能使用 SEH</td></tr><tr><td>C++ 支持</td><td>完全的 C++ 支持</td><td>不支持 C++ runtime</td></tr></tbody></table>

#### 3.1.1 未处理异常

未处理异常会导致蓝屏，原因是防止继续执行代码、对系统造成不可逆转的伤害

 

内核代码不应该跳过任何细节或错误检查

#### 3.1.2 终止

如果驱动程序卸载时仍保留分配的内存或打开的内核句柄，这些资源不会自动释放，只会在下次系统启动时释放

 

原因是驱动程序可以分配一些缓冲区，然后将其传递给另一个与之合作的驱动程序

#### 3.1.3 函数返回值

忽略内核 API 的返回值很危险，应该总是检查返回值

#### 3.1.4 IRQL

中断请求级（Interrupt Request Level, IRQL）通常为 0

 

用户模式下始终为 0，内核模式下大部分时间为 0

#### 3.1.5 C++ 使用

没有 C++ runtime

 

一些不支持的 C++ 特性：

*   不支持`new`和`delete`，这正常是在用户模式堆分配的
*   不会调用具有非默认构造函数的全局变量
    *   避免在构造函数中使用代码，创建一些要显式调用的 Init 函数
    *   仅将指针分配为全局变量，动态创建实例
*   不支持 C++ 异常处理（`try`、`catch`、`throw`）
*   不可使用标准 C++ 库，如`std::vector<>`、`std::wstring`等

一些支持的 C++ 特性：

*   `nullptr`关键字
*   `auto`关键字
*   模板将在有意义时使用
*   重载 new 和 delete 运算符
*   构造函数和析构函数，尤其是用于构建 RAII 类型

#### 3.1.6 测试和调试

内核调试需要双机调试，一台作为调试者、另一台作为被调试者运行驱动程序

### 3.2 Debug vs. Release 生成

内核术语是 Checked（Debug）和 Free（Release）

 

Debug 意味着可以使用 DBG 符号

### 3.3 内核 API

内核 API 常用前缀的意义：

*   Ex：一般执行函数
*   Ke：一般内核函数
*   Mm：内存管理
*   Rtl：一般运行时库
*   FsRtl：文件系统运行时库
*   Flt：文件系统迷你过滤库
*   Ob：对象管理
*   Io：I/O 管理
*   Se：安全
*   Ps：进程结构
*   Po：电源管理
*   Wmi：Windows 管理工具
*   Zw：native API 包装
*   Hal：硬件抽象层
*   Cm：配置管理器（注册表）

Nt 前缀的内核函数对应 NtDll.Dll 的函数，会根据 KTHREAD 结构的标记（调用者是否来自内核）对参数进行检查

 

Zw 前缀的内核函数先将调用者模式设为`KernelMode(0)`，然后调用 Nt 前缀的内核函数

### 3.4 函数和错误代码

可以在`ntstatus.h`中找到`NTSTATUS`值的定义

 

大多数代码并不关心错误具体是什么，仅测试最高位即可，可以使用`NT_SUCCESS`宏

 

当返回到用户层时，会由`STATUS_xxx`转成`ERROR_yyy`，用户模式通过 GetLastError 可以得到这些错误

 

通常遇到错误时，会返回相同的 NTSTATUS 到调用函数

### 3.5 字符串

内核使用`UNICODE_STRING`

```
typedef struct _UNICODE_STRING {
USHORT Length;
USHORT MaximumLength;
PWCH Buffer;
} UNICODE_STRING;
typedef UNICODE_STRING *PUNICODE_STRING;
typedef const UNICODE_STRING *PCUNICODE_STRING;

```

`Length`是字符串的字节数（不包括 \ x00\x00 结束符）

 

`MaximumLength`是不需要重新分配内存的情况下、字符串字节数上限

 

需要注意的是，UNICODE_STRING 并**不总是有 \ x00\x00 结尾**

 

一些常用的字符串操作函数：

*   RtlInitUnicodeString
*   RtlCopyUnicodeString
*   RtlCompareUnicodeString
*   RtlEqualUnicodeString
*   RtlAppendUnicodeStringToString
*   RtlAppendUnicodeToString

### 3.6 动态内存分配（S2）

内核提供两种通用内存池（general memory pools）给驱动使用：

*   页池（Paged pool）：可能会被换出（paged out）的内存池
*   非页池（Non Paged Pool）：一直在 RAM 中的内存池

枚举类型`POOL_TYPE`表示池类型，只有三种是可以用于驱动的：  
`PagedPool`、`NonPagedPool`、`NonPagedPoolNx`  
（non-page pool 没有可执行权限）

 

常用内存池函数：

*   ExAllocatePool（已过时，将被下面的函数取代）
*   ExAllocatePoolWithTag
*   ExAllocatePoolWithQuotaTag
*   ExFreePool

tag 是 4 字节的值

 

可以在 PoolMon（WDK 的 Windows Kits 中）中观察到有 tag 的内存池（tag 以大端序字符串显示）

 

给 ustring 分配页池内存：

 

code:

```
UNICODE_STRING strA;
int length;
// allocate
strA.Buffer = (WCHAR*)ExAllocatePoolWithTag(PagedPool,
    length, 'dcba');
if (strA.Buffer == nullptr) {
    KdPrint(("Failed to allocate memory\n"));
    return STATUS_INSUFFICIENT_RESOURCES;
}
strA.MaximumLength = length;

```

shell:

```
sc create S2_DynMemAlloc type= kernel binPath= "\\vmware-host\Shared Folders\MyDriver\x64\Debug\S2_DynMemAlloc.sys"
sc start S2_DynMemAlloc
sc stop S2_DynMemAlloc

```

### 3.7 链表

内核使用循环双向链表：

```
typedef struct _LIST_ENTRY {
    struct _LIST_ENTRY *Flink;
    struct _LIST_ENTRY *Blink;
} LIST_ENTRY, *PLIST_ENTRY;

```

`CONTAINING_RECORD`宏执行适当的偏移计算并转换为实际数据类型  
`CONTAINING_RECORD(pvoid, type, entry_member_name)`

```
struct MyDataItem {
    // some data members
    LIST_ENTRY Link;
    // more data members
};
 
MyDataItem* GetItem(LIST_ENTRY* pEntry) {
    return CONTAINING_RECORD(pEntry, MyDataItem, Link);
}

```

常用链表函数（时间复杂度都是常数）：

*   InitializeListHead
*   InsertHeadList
*   InsertTailList
*   IsListEmpty
*   RemoveHeadList
*   RemoveTailList
*   RemoveEntryList
*   ExInterlockedInsertHeadList
*   ExInterlockedInsertTailList
*   ExInterlockedRemoveHeadList

后三个关于自旋锁，在第 6 章详细讨论

### 3.8 驱动对象（The Driver Object）

常用 major function 代码：

*   IRP_MJ_CREATE (0)
*   IRP_MJ_CLOSE (2)
*   IRP_MJ_READ (3)
*   IRP_MJ_WRITE (4)
*   IRP_MJ_DEVICE_CONTROL (14)
*   IRP_MJ_INTERNAL_DEVICE_CONTROL (15)
*   IRP_MJ_PNP (31)
*   IRP_MJ_POWER (22)

`MajorFunction`数组由内核初始化指向内核内部例程`IopInvalidDeviceRequest`，该例程直接返回失败，表示不支持该操作

### 3.9 设备对象（Device Objects）

驱动通过设备与 r3 代码通信，驱动应该至少创建一个设备对象并为其命名

 

CreateFile 可以打开设备，第一个参数为设备对象名称

 

打开文件或设备的句柄会创建内核结构 FILE_OBJECT 的实例，这是个半文档化的结构。

 

更准确的说，CreteFile 接受一个`symbolic link`（符号链接）

 

对象管理器中名为`??`的目录下的符号链接都可被用户模式代码通过 CreateFile 或 Createfile2 调用

 

可以通过 WinObj 查看（WinObj 中目录名为`Global??`）

 

使用符号链接的 CreateFile 的文件名（第一个参数），必须加上前缀`\\.\`（c++ 中是`"\\\\.\\"`）

 

如果创建了多个设备对象，将形成一个单向链表，添加设备时是头插法，所以第一个创建的设备在链表的最后

4 驱动从头到尾（Driver from Start to Finish）（S3）
-----------------------------------------

将完成一个完整的驱动及客户端程序，利用驱动完成只能在内核模式下完成的功能（设置任意级别的线程优先级）

### 4.1 绪论

线程优先级 = 进程优先级 + 相对线程优先级

 

用户模式下，设置进程优先级可以用`SetPriorityClass`，共有 6 个级别  
设置相对线程优先级可以用`SetThreadPriority`，共有 7 个级别

 

下面是线程优先级合法值的表（通过 windows api 设置），据别的书说是个未文档化的东西，windows 不建议开发时考虑线程优先级，该表的值随 windows 版本变化可能发生改变

<table><thead><tr><th>进程优先级</th><th>-Sat</th><th>-2</th><th>-1</th><th>0</th><th>+1</th><th>+2</th><th>+sat</th></tr></thead><tbody><tr><td>Idle(low)</td><td>1</td><td></td><td></td><td>4</td><td></td><td></td><td>15</td></tr><tr><td>Below Normal</td><td>1</td><td></td><td></td><td>6</td><td></td><td></td><td>15</td></tr><tr><td>Normal</td><td>1</td><td></td><td></td><td>8</td><td></td><td></td><td>15</td></tr><tr><td>Above Normal</td><td>1</td><td></td><td></td><td>10</td><td></td><td></td><td>15</td></tr><tr><td>High</td><td>1</td><td></td><td></td><td>13</td><td></td><td></td><td>15</td></tr><tr><td>Real-time</td><td>16</td><td></td><td></td><td>24</td><td></td><td></td><td>31</td></tr></tbody></table>

 

进程优先级枚举：级别 +`_PRIORITY_CLASS`

 

线程优先级枚举：`THREAD_PRIORITY_`+ 级别

### 4.2 驱动初始化

大多数驱动需要在 DriverEntry 中做如下操作：

*   设置 Unload 例程
*   设置驱动支持的调度例程
*   创建一个设备对象
*   创建一个指向设备对象的符号链接

所有驱动必须支持`IRP_MJ_CREATE`和`IRP_MJ_CLOSE`，不然无法打开一个驱动的设备的句柄，通常这两个调度例程是相同的

 

调度例程的函数原型：`NTSTATUS Function(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)`

#### 4.2.1 将信息传给驱动

用户模式客户端可用的三个基础函数：`WriteFile`、`ReadFile`、`DeviceIoControl`

#### 4.2.2 客户端 / 驱动程序通信协议

必须使用`CTL_CODE`宏来构建控制代码

```
#define CTL_CODE( DeviceType, Function, Method, Access ) (                 \
    ((DeviceType) << 16) | ((Access) << 14) | ((Function) << 2) | (Method) \
)

```

*   DeviceType：设备类型标识，`FILE_DEVICE_xxx`，第三方应以 0x8000 开头
*   Function：指示特定操作的升序数字，第三方应该以 0x800 开头
*   Method：指示客户端提供的输入和输出缓冲区如何传递给驱动程序（将在第 6 章详细讨论）
*   Access：指示对驱动来说这个操作是什么？

示例：

```
#define MY_DEVICE 0x800
#define IOCTL_MY_OP CTL_CODE(\
MY_DEVICE, 0x800, METHOD_NEITHER, FILE_ANY_ACCESS)

```

#### 4.2.3 创建一个设备对象

**创建设备名：**

 

在创建一个设备对象前，需要先创建一个`UNICODE_STRING`存储内部设备名称

 

下面是两种初始化方式：

```
// plan A
UNICODE_STRING devName = RTL_CONSTANT_STRING(L"\\Device\\YourName");
 
// plan B
UNICODE_STRING devName;
RtlInitUnicodeString(&devName, L"\\Device\\YourName");

```

设备名称需要在设备对象管理器目录下

 

（RtlInitUnicodeString 函数内部字符串的长度，RTL_CONSTANT_STRING 宏在编译时计算长度）

 

**创建设备对象：**

 

创建设备对象需要调用`IoCreateDevice`

```
NTSTATUS IoCreateDevice(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ ULONG DeviceExtensionSize,
    _In_opt_ PUNICODE_STRING DeviceName,
    _In_ DEVICE_TYPE DeviceType,
    _In_ ULONG DeviceCharacteristics,
    _In_ BOOLEAN Exclusive,
    _Outptr_ PDEVICE_OBJECT *DeviceObject);

```

**创建设备完整示例：**

```
UNICODE_STRING devName = RTL_CONSTANT_STRING(L"\\DEVICE\\devName");
PDEVICE_OBJECT devObj;
status = IoCreateDevice(
    DriverObject,        // our driver object
    0,                   // no need for extra bytes
    &devName,            // the device name
    FILE_DEVICE_UNKNOWN, // device type
    0,                   // characteristics flags
    FALSE,               // not exclusive
    &devObj              // the resulting pointer
);
if (status < 0) {
    KdPrint(("[] Failed to create device object (0x%08X)\n", status));
    return status;
}

```

**创建符号链接：**

 

需要创建一个指向设备的符号链接，供 r3 调用

 

同样需要先创建一个字符串作为符号链接对象名称

```
UNICODE_STRING symLink = RTL_CONSTANT_STRING(L"\\??\\symLinkName");
status = IoCreateSymbolicLink(&symLink, &devName);
if (status < 0) {
    KdPrint(("[] Failed to create symbolic link (0x%08X)\n", status));
    return status;
}

```

**注意：资源释放**

 

上面创建的字符串会自动释放（好像在函数的栈中）？但对象不会，需要（在 unload 例程中）手动删除

```
void Unload(_In_ PDRIVER_OBJECT DriverObject) {
    // delete symbolic link
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(L"\\??\\symLinkName");
    IoDeleteSymbolicLink(&symLink);
    // delete device object
    IoDeleteDevice(DriverObject->DeviceObject);
}

```

### 4.3 客户端代码

将用`CTL_CODE`构造的控制代码放到一个头文件中，供驱动代码和用户模式客户端代码同时使用

 

通过符号链接获驱动的设备的句柄

```
{
    HANDLE hDevice = CreateFile(L"\\\\.\\symLinkName", GENERIC_WRITE,
        FILE_SHARE_WRITE, nullptr, OPEN_EXISTING, 0, nullptr);
    if (hDevice == INVALID_HANDLE_VALUE)
        return Error("Failed to open device");
}
 
int Error(const char* msg) {
    printf("%s (error=%d)\n", msg, GetLastError());
    return 1;
}

```

### 4.4 Create 和 Close 调度例程

该例程什么都不用做，直接返回成功即可

```
NTSTATUS PriorityBoosterCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp) {
    UNREFERENCED_PARAMETER(DeviceObject);
 
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

```

IRP 是半文档化结构，通常来自运行中的管理器：I/O Manager, Plug & Play Manager or Power Manager

 

对驱动程序的每个请求总是包装在 IRP 中

 

IRP 中有一个或多个`IO_STACK_LOCATION`结构

 

为了完成 IRP，需要调用`IoCompleteRequest`，这个函数做很多东西，基本上理解为将 IRP 传播回创建者（通常是 I/O 管理器），然后由管理器通知客户端操作完成

### 4.5 DeviceIoControl 调度例程

调用`IoGetCurrentIrpStackLocation`获取当前设备对应的`IO_STACK_LOCATION`

 

`IO_STACK_LOCATION`中有控制代码、输入输出 buffer 指针等

> 调度例程运行在调用该例程的用户模式进程的上下文中

```
DWORD threadId;
PETHREAD Thread;
status = PsLookupThreadByThreadId(ULongToHandle(threadId), &Thread);

```

使用`ULongToHandle`（这实际上只是个 casting）将 pid 转换成`HANDLE`

 

线程和进程存在一个全局私有内核句柄表，句柄的 “值” 实际上就是 ID

 

（HANDLE 在 64 位系统是 64 位，线程 ID 始终是 32 位）

### 4.6 安装和测试

```
sc create S3_PriorityBooster type= kernel binPath= "\\vmware-host\Shared Folders\MyDriver\x64\Debug\S3_PriorityBooster.sys"
sc start S3_PriorityBooster
sc stop S3_PriorityBooster
sc delete S3_PriorityBooster

```

start 后可以在 WinObj 中的`Driver`目录下看到驱动、`GLOBAL??`目录下看到符号链接

 

可以在 Process Explorer 中查看进程的 pid 以及其线程的动态优先级

5 调试
----

关于使用 WinDbg 进行调试

### 5.1 windows 的调试工具

四个调试器：

*   Cdb 和 Ntsd 是用户模式调试器，可以附加到进程上，是命令行界面，没有什么大的区别
*   Kd 是内核调试器，提供命令行界面，可以附加到本地内核或其他机器
*   WinDbg 是有图形化界面的调试器，可以调试用户和内核模式

> WinDbg Preview 是 WinDbg 的 “最新版”，解决了一些 WinDbg 上的 bug

 

这些调试器都是基于`DbgEng.Dll`

### 5.2 WinDbg 简介

虽然有 GUI，实际上还是命令行，所有 UI 操作都会转成命令，显示在命令行窗口上

 

WinDbg 支持三种类型的命令：

*   标准命令 (Intrinsic)：内置在调试器中，在被调试的目标上运行
*   元命令 (Meta)：以`.`开头，作用于调试器 (debugging process) 本身，而不是直接作用于被调试目标
*   拓展命令：以`!`开头，提供调试器大部分功能，都在拓展 DLL 中实现

#### [](#教程：用户模式调试基础)教程：用户模式调试基础

**符号信息：**

 

设置符号的方法 1：`.symfix`

 

设置符号的方法 2：设置环境变量  
`_NT_SYMBOL_PATH`=`SRV*c:\Symbols*http://msdl.microsoft.com/download/symbols`

 

`lm`：显示进程加载的模块，以及各模块是否加载了符号

 

`.reload /f modulename.dll`：强制加载模块的符号

 

`!sym noisy`：记录符号加载尝试的详细信息

 

**线程：**

 

`~`：显示调试进程中所有线程的信息  
线程信息前的`.`表示当前线程，`#`表示触发中断的线程  
输入提示冒号右边的数字是当前线程的索引

```
0  Id: 874c.18068 Suspend: 1 Teb: 00000001`2229d000 Unfrozen
[下标] Id: [PID].[TID] Suspend: [挂起计数] Teb: [TEB地址] [是否冻结]

```

`~ns`：切换到索引为 n 的线程  
可以组合命令`~nk`，这样可以在不切换线程的情况下，在别的线程执行操作（这里是显示别的线程的调用堆栈）

 

`k`：当前线程的调用堆栈（stack trace）

 

`!teb`：查看 TEB 的部分信息，默认当前线程的

 

**进制转换：**

 

16 转 10：

```
0:000> ? 874c
Evaluate expression: 34636 = 0000874c

```

10 转 16：

```
0:000> ? 0n34636
Evaluate expression: 34636 = 0000874c

```

**数据或结构的显示：**

 

`dt [type]`：显示数据结构的定义（如显示_TEB：`dt ntdll!_teb`）

 

`dt [type] [addr]`：显示数据结构的数据（如显示某个_TEB：``dt ntdll!_teb 000000`012229d000``）

 

`r [reg]`：读取寄存器（如读取 rcx：`r rcx`）

 

`d{a|b|c|d|D|f|p|q|u|w|W}`：以指定类型显示指定地址的数据  
a：ascii 字符  
b,w,d,q：字节  
u：unicode  
f：单精浮点  
D：双精浮点

 

`u`：显示反汇编，默认 8 句汇编指令

 

`!error [error_code]`：显示错误信息

 

**断点和运行：**

 

`bp [symbol]`：设置断点（如 CreateFile：`bp kernel32!createfilew`）

 

`bl`：显示当前设置的断点

 

`bd`：禁用断点，禁用所有断点：`bd *`

 

`bc`：删除断点

 

`g`(F5)：运行直到断点

 

`p`(F10)：步过

 

`t`(F11)：步进

### 5.3 内核调试（本地）

#### 本地内核调试

修改启动项：`bcdedit /debug on`

#### 本地内核调试教程

`!process 0 0`：显示所有进程的基本信息

```
lkd> !process 0 0
**** NT ACTIVE PROCESS DUMP ****
PROCESS ffff8d0e682a73c0
    SessionId: none Cid: 0004 Peb: 00000000 ParentCid: 0000
    DirBase: 001ad002 ObjectTable: ffffe20712204b80 HandleCount: 9542.
    Image: System
 
(truncated)

```

*   PROCESS 旁边的地址：EPROCESS 的地址
*   SessionId：进程所处的对话
*   Cid：pid
*   Peb：PEB 地址（在用户模式地址空间）
*   ParentCid：父进程 pid
*   DirBase：进程主页目录的物理地址（x32 是 PDPT 基址、x64 是 PML4 基址）
*   ObjectTable：指向进程的私有句柄表的指针
*   HandleCount：进程中的句柄数
*   Image：可执行文件名称，或与可执行文件无关的特殊进程名称

`!process`指令后第一个数字是筛选特定进程，0 表示所有进程；第二个数字是细节掩码，0 表示最少细节；第三个参数是筛选可执行文件

 

`.process /p [EPROCESS]`：切换到指定进程

 

peb 在用户模式地址空间中，查看 peb 需要先设置正确的用户模式进程环境

 

不切换的做法：`.process /p ffff8d0e849df080; !peb e8a8c9c000`

 

调用堆栈中，nt 前缀表示内核

 

`.reload /user`：加载用户模式符号

 

其余常用 / 有趣的内核模式调试指令：

*   `!pcr`：显示指定为附加索引的处理器的进程控制区域 (PCR)（如果未指定索引，则默认显示处理器 0）
*   `!vm`：显示系统和进程的内存统计信息
*   `!running`：显示有关在系统上所有处理器上运行的线程的信息

### 5.4 完全内核调试（双机）

完全内核调试需要” 双机 “

 

最好的连接方式是通过网络，这需要主机和被调试目标系统版最少为 Win8

 

另外一种方法是 COM 串口，大多数虚拟机支持虚拟串口而不需要真实（物理的）串口线

 

详细配置方式略过

#### 配置目标机器

```
bcdedit /debug on
bcdedit /dbgsettings serial debugport:1 baudrate:115200

```

#### 配置主机

调试器需要设置调试端口映射和命名管道，与虚拟机上的相同

 

输入提示 kd 左边的数字是引起中断的处理器的索引

### 5.5 内核驱动调试教程

可以设置未来断点（在运行程序前设置断点）

 

如设置驱动 prioritybooster 的入口点：`bu prioritybooster!driverentry`

 

可以设置只在指定进程上中断：`bp /p [EPROCESS] [symbol]`  
如：`bp /p ffffdd06042e4080 prioritybooster!priorityboosterdevicecontrol`

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#驱动开发](forum-41-1-132.htm)