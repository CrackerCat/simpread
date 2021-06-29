> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268267.htm)

> [分享]Windows 初始化过程

> 最近在专研 UEFI，发现有段时间没发帖子了... 所以讲一下 Windows 初始化的过程。  
> 环境：Windows 2004 x64  
> 个人见解，有不对的可以私信我，QQ：1045551070  
> 个人博客：http://blog.leanote.com/archives/only_the_brave

1. 前言
-----

```
随着计算机的发展, 传统的BIOS引导已经过时, 关于UEFI引导的安全对抗已经展开。
从如下流程图我们可以看出UEFI中MBR和VBR不再存在, 而是UEFI自己负责加载bootmgr, 这也意味着更加安全和迅速。 

```

![](https://bbs.pediy.com/upload/attach/202106/749013_GPG6GRSRPDWS62S.png)

2. UEFI 规范
----------

```
与传统的BIOS引导相反, UEFI规范覆盖了硬件初始化开始到操作系统启动前的每一个步骤, 该规范主要分七个步骤，如下：
 
  1.     安全性阶段(SEC)：初始化临时缓存区, 即将Cpu的Cache设置为no-eviction模式, 并将整理的参数传递给BFV中找出的PEI入口函数。(在SEC阶段执行的代码是从SPI闪存运行)
 
  2.     Pre-EFI 初始化阶段(PEI)：配置内存控制器，初始化芯片组，并处理S3恢复过程。在此阶段执行的代码在临时内存中运行，直到初始化内存控制器为止。随后从DXE IPL PPI的Entry服务中找到DXE Image的入口函数并调用。
 
  3.     驱动执行环境阶段(DXE)：初始化系统管理模式(SMM) 和 DXE服务(Protocol)以及BS和RT服务。初始化完毕后DXE通过 EFI_BDS_ARCH_PROTOCOL找到BDS并调用入口函数。
 
  4.     引导设备选择阶段(BDS)：通过枚举可能包含UEFI兼容引导程序的PCI总线上的外围设备，来发现可以从中引导OS的硬件设备(Os loader)。
 
  5.     临时系统加载阶段(TSL)：操作系统加载器(Os loader)执行的第一阶段, 这个阶段是为Os loader准备执行环境，直至启动服务调用ExitBootServices()，系统将进入RT阶段。
 
  6.     运行时阶段(RT)：此时系统的控制权已由UEFI内核转交给Os loader手中, 随着Os loader的运行，最终会进入内核入口KiSystemStartup函数，将控制权完全交给OS。
 
  7.     AL阶段：在RT阶段系统遇到灾难性错误会来到这(不做详细描述)。

```

如上所述，我们主要将视线放到 BDS、TSL、RT 阶段。

3. BootMgr
----------

### 3.1 EfiEntry

```
在BDS后，SPI储存的UEFI固件代码已完成工作, 随后UEFI固件启动管理器先查询NVRAM UEFI变量以找到ESP，并找到OS特定的启动管理器bootmgfw.efi调用它入口函数(DXE 驱动)。

```

BootMgr 的入口函数：

```
EFI_STATUS __fastcall EfiEntry(
    EFI_HANDLE ImageHandle,         // 程序内存映像的句柄
    EFI_SYSTEM_TABLE *SystemTable   // 系统表指针
    )
{
  int unKnow;
  __int64 *BootParameters;
  unsigned int Status;
 
  BootParameters = EfiInitCreateInputParametersEx(
      ImageHandle,
      SystemTable,
      unKnow);// 将EfiEntry参数转换为bootmgfw所期望的应用程序参数格式
  if ( BootParameters )
    Status = BmMain(BootParameters);   // 调用Windows引导管理器入口点
  else
    Status = 0xC000000D;                        // STATUS_INVALID_PARAMETER
  return EfiGetEfiStatusCode(Status);           // 将NT状态代码转换为EFI代码
}

```

```
该函数首先会调用 EfiInitCreateInputParametersEx 函数, 该函数主要用于将EfiEntry参数转换为bootmgfw.efi所期望的参数格式。
 
随后调用Windows引导管理器入口点 BmMain 函数。

```

### 3.2 BmMain

```
在该函数中调用了 BmFwInitializeBootDirectoryPath 用于初始化启动应用程序(BootDirectory)路径(\EFI\Microsoft\Boot)。
 
随后BootMgr会读取系统引导配置信(BCD), 如果有多个启动选项，其会调用 BmDisplayGetBootMenuStatus 显示启动菜单，如下图：

```

![](https://bbs.pediy.com/upload/attach/202106/749013_HEGEX5TE7CUWDRH.png)

```
再然后其会调用 BmpLaunchBootEntry 函数, 启动应用程序（winload.efi）。
 
当然bootmgfw.efi做的不止这些还有启动策略验证代码完整性以及安全启动组件的初始化，这些就不细说了。

```

### 3.3 BmpLaunchBootEntry

```
在Windows引导管理器(BootMgr)最后阶段, BmpLaunchBootEntry 函数会根据之前BCD的值选择正确的启动项， 如果启用了全卷加密(BitLocker), 则会先解密系统分区，然后才能将控制权转移到winload.efi。
 
其次会调用 BmTransferExecution 函数，检查启动选项并将执行流传递给BlImgStartBootApplication函数。
 
再然后 BlImgStartBootApplication 函数中会调用 ImgFwStartBootApplication 函数, 而最终调用 ImgArchStartBootApplication 函数。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_52882P4VWWDJ3FX.png)

### 3.4 ImgArchStartBootApplication

```
ImgArchStartBootApplication 函数原型如下：

```

```
EFI_STATUS EFIAPI ImgArchStartBootApplication(
    PBL_APPLICATION_ENTRY AppEntry,
    VOID* ImageBase,            // winload.efi镜像基址
    UINT32 ImageSize,           // winload.efi镜像大小
    UINT8 BootOption,
    PBL_RETURN_ARGUMENTS ReturnArguments
    );

```

```
在其中会初始化winload.efi的内存保护模式,
随后调用BlpArchTransferTo64BitApplication 函数,                 
BlpArchTransferTo64BitApplication 会调用
Archpx64TransferTo64BitApplicationAsm 函数，最终将控制权交给winload.efi。

```

### 3.5 Archpx64TransferTo64BitApplicationAsm

```
该函数会启用新的GDT和IDT， 随后完全把控制权交给winload.efi, 到此BootMgr完成使命，Winload开始工作。  

```

![](https://bbs.pediy.com/upload/attach/202106/749013_JFGW2N6G9X26MBP.png)

4. Winload
----------

### 4.1 OslMain

```
该函数会初始化所需的支持库随后调用 OslpMain。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_9Z8F7XC96A2XZV4.png)

### 4.2 OslpMain

```
winload.efi加载后 会调用 OslpMain 函数。 

```

![](https://bbs.pediy.com/upload/attach/202106/749013_4XZ5ZGFYDANNVMS.png)

### 4.3 OslPrepareTarget

```
该函数首先调用 BcdUtilGetBootOption 函数确定BCD中激活的选项是否处于活动状态, 随后调用OslpLoadDriverStoreNodes函数读取"system32\\config\\system", 其可以提供哪些驱动需要被加载起来, 接下来调用 OslpLoadSystemHive 函数读取和加载注册表的System Hive, 因为其中包含更多的系统运行参数, 随后调用 OslpLoadAllModules 函数加载所有系统所需要的模块, 并为内核准备新的GDT和IDT，以及建立内存映射。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_4K57647ZERQE4PG.png)

### 4.4 OslpLoadAllModules

```
该函数负责执行核心任务, 那就是加载操作系统的内核文件和boot类型的设备驱动。
其首先调用 OslpGetBootDriverFlags 函数获取引导标志并检查一些事件, 随后会判断 5 级分页是否处于活动状态，是的话加载内核ntkrla57.exe。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_QBAB38U47BDTYDC.png)

```
之后会为 Kernel 和 Hal 分配空间。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_ZJMMUNYEJFW4WKY.png)

```
随后首先加载ntoskrnl.exe, 这个模块包含了操作系统的内核和执行体。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_TYUTNNUNTMUGMDU.png)

```
其次加载硬件抽象层模块HAL.DLL、支持双机调试的KDCOM.DLL, 以及它们所依赖的模块

```

![](https://bbs.pediy.com/upload/attach/202106/749013_9S5XY76Y2ACMCNQ.png)  
![](https://bbs.pediy.com/upload/attach/202106/749013_94FCM97FHU4HN6W.png)

```
之后加载完系统模块后，Winload还需要调用 OslLoadDrivers 函数加载Boot Type类型的设备驱动, 也就是Start为0的驱动文件

```

![](https://bbs.pediy.com/upload/attach/202106/749013_QG8G6S2FSX4RV5B.png)

```
至此 OslpLoadAllModules 函数执行完毕，所有的模块信息存储在了OslLoaderBlock全局变量中。

```

### 4.5 OslExecuteTransition

![](https://bbs.pediy.com/upload/attach/202106/749013_EK6Q4UK4BKJB8UU.png)

```
当 OslPrepareTarget 函数执行完毕, 代表着前期的准备工作已经完毕, 其随后调用 OslExecuteTransition 函数进入内核。
 
OslExecuteTransition 函数第一个调用的函数是 OslFwpKernelSetupPhase1 它负责调用调用 ExitBootServices 结束BS启动服务, 该函数还负责调用 SetVirtualAddressMap 设置物理地址到虚拟地址的映射, 之后其会调用 OslArchTransferToKernel 函数跳转至内核。

```

### 4.6 OslArchTransferToKernel

```
该函数的函数原型如下：

```

```
void __stdcall OslArchTransferToKernel(
    ULONG64 OslLoaderBlock, // 存储当前所有系统加载模块的信息
    PVOID OslEntryPoint     // 内核入口点
    );

```

```
该函数的操作大体如下， 设置一些信息，随后跳至内核入口点(KiSystemStartup), 并控制权交给内核。

```

![](https://bbs.pediy.com/upload/attach/202106/749013_5NDE9GKRXCQ45GJ.png)

5. 参考
-----

软件调试第二版  
深入解析 windows 操作系统第六版  
UEFI 原理与编程  
[https://www.n4r1b.com/posts/](https://www.n4r1b.com/posts/)

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年秋季班火热招生！！](https://bbs.pediy.com/thread-267018.htm)