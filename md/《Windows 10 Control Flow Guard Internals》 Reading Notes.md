> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267302.htm)

0x00 前言
-------

Wikipedia：

> Control Flow Guard (CFG) was first released for Windows 8.1 Update 3 (KB3000850) in November 2014. Developers can add CFG to their programs by adding the /guard:cf linker flag before program linking in Visual Studio 2015 or newer.
> 
> As of Windows 10 Creators Update (Windows 10 version 1703), the Windows kernel is compiled with CFG.The Windows kernel uses Hyper-V to prevent malicious kernel code from overwriting the CFG bitmap.
> 
> CFG operates by creating a per-process bitmap, where a set bit indicates that the address is a valid destination. Before performing each indirect function call, the application checks if the destination address is in the bitmap. If the destination address is not in the bitmap, the program terminates.This makes it more difficult for an attacker to exploit a use-after-free by replacing an object's contents and then using an indirect function call to execute a payload.

 

Windows CFG 是 Control-Flow Integrity(CFI) 的具体实现，该机制由 Windows 8.1 Update 3 (KB3000850) 开始引入，需编译器和操作系统相结合，目的在于防止不可靠间接调用。漏洞利用常常通过修改间接调用地址以劫持执行流，而 CFG 会于编译链接期间将程序所有间接调用地址记录在 PE 文件中，并在执行所有间接调用前增加校验，若间接调用地址被修改，则抛出异常。

 

环境：

*   物理机 OS：Windows 10 20H2 x64
*   物理机 WinDbg：10.0.17134.1
*   虚拟机 OS：Windows 10 1511(10586.164) x86
*   虚拟机 WinDbg：10.0.19041.685
*   VMware：VMware Workstation 15 Pro
*   Visual Studio 2019

0x01 How CFG Works:User Mode Part
---------------------------------

使用如下代码进行编译及调试：

```
typedef int(*fun_t)(int);
 
int foo(int a)
{
    printf("hellow world %d\n",a);
    return a;
}
class CTargetObject
{
public:
    fun_t fun;
};
int main()
{
    int i = 0;
    CTargetObject *o_array = new CTargetObject[5];
    for (i = 0; i < 5 ; i++)
        o_array[i].fun = foo;
    o_array[0].fun(1); 
    return 0;
}

```

启用 CFG：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_CPTC4UV3GTEMCYU.jpg)

 

修改 "调试信息格式"：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_UEUTRPX8JQK7JGR.jpg)

 

编译完成，`dumpbin.exe /headers /loadconfig Project1.exe`：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_SHD37C62K3Q4D4P.jpg)

 

可以看到 CFG 已启用。

 

![](https://bbs.pediy.com/upload/attach/202105/817966_V96R2YYDCFZ7Z6R.jpg)

*   `Guard CF address of check-function pointer`：指向`___guard_check_icall_fptr`，载入 PE 后，其指向`ntdll!LdrpValidateUserCallTarget`

![](https://bbs.pediy.com/upload/attach/202105/817966_XMT8BQGZ266UKMW.jpg)

*   `Guard CF address of dispatch-function pointer`：Reserved
*   `Guard CF function table`：指向 RVA 列表，供 NT 内核使用

![](https://bbs.pediy.com/upload/attach/202105/817966_PSS4C3WMH43NGY4.jpg)

*   `Guard CF function count`：RVA 数量
*   `Guard Flags`

 

系统若不支持 CFG，则不对`___guard_check_icall_fptr`进行更新 (如下为 Windows 7 SP1 x86)：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_76S3G8S645EFY4J.jpg)

 

若系统支持 CFG，首先是`nt!PspPrepareSystemDllInitBlock`(由`nt!NtCreateUserProcess`—>`nt!PspAllocateProcess`—>`nt!PspSetupUserProcessAddressSpace`调用) 初始化`ntdll!LdrSystemDllInitBlock`：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_W7MXHR6ZRWN4UYW.jpg)

 

其偏移`0x60`为 Bitmap Address，`0x68`为 Bitmap Size。

 

![](https://bbs.pediy.com/upload/attach/202105/817966_PRCXSH8H2BNX8J5.jpg)

 

载入 PE 文件时，`ntdll!LdrpCfgProcessLoadConfig`会校验`OptionalHeader.DllCharacteristics`：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_2MK6YWCBNQCHW2Z.jpg)

 

修改`_guard_check_icall_nop`指向`ntdll!LdrpValidateUserCallTarget`：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_V5T5Y8UPDFAECFN.jpg)

 

间接调用地址传递给 ECX 寄存器：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_JCE7XWHN9GGERUW.jpg)

 

下面来看`ntdll!LdrpValidateUserCallTarget`校验过程：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_HUMYKAVBKMY2CNZ.jpg)

1.  获取`ntdll!LdrSystemDllInitBlock+0x60`处 Bitmap Address，当前进程空间每 8 Bytes 对应 Bitmap 中 1 bit[`mov edx, ds:dword_6A30C1A0`]
2.  获取指向 Destination Address 低 8 位 (即 1 Byte) 范围内的 Bitmap 值(低 8 位范围：0x00—0xFF，256 Bytes；Bitmap 值：32 bit—32_8=256 Bytes)[`mov eax, ecx`;`shr eax, 8`;`mov edx, [edx+eax_4]`]
3.  Destination Address 是否以 0x10 对齐，是则取最低字节前 5 位作为索引获取 bit；否则取最低字节前 5 位再与 0x01 作或运算之后作为索引获取 bit

![](https://bbs.pediy.com/upload/attach/202105/817966_X7YDSHQNXXTGF5S.jpg)

1.  该 bit 值为 1，则 Destination Address 合法

注：

 

`shr eax, 3` 即 Destination Address/8，Bitmap 中每 1bit 对应进程空间 8 Bytes。`bt`指令功能如下：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_YVYADE3AHBRUYZ9.jpg)

 

由于`SRC`寄存器是 32 位，故`POSITION`值需模 32(0x20—0010 0000)，如此一来，便做到取 Destination Address 最低字节前 5 位为索引。若 Destination Address 合法，则直接`retn`，非法会交由`ntdll!RtlpHandleInvalidUserCallTarget`，最终触发`int 29`中断。

 

在执行`mov edx,dword ptr [edx+eax*4]`时，若 EAX 寄存器已被篡改，则可能会触发内存访问异常，而`ntdll!LdrpValidateUserCallTarget`中并未加入异常处理，其异常处理位于`ntdll!RtlDispatchException`中：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_HS88UMEVAHWGGW9.jpg)

 

![](https://bbs.pediy.com/upload/attach/202105/817966_X5UZSMM7UWXU83A.jpg)

 

![](https://bbs.pediy.com/upload/attach/202105/817966_2UJ4DYHTXJTW678.jpg)

 

最后交由`ntdll!RtlpHandleInvalidUserCallTarget`来处理：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_CWCCNZDPZ85JUTF.jpg)

0x02 How CFG Works:Kernel Mode Part
-----------------------------------

`nt!MiInitializeCfg`首先调用`nt!PsIsSystemWideMitigationOptionSet`校验注册表`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Kernel\MitigationOptions`键值：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_C727WJPC3283339.jpg)

 

`nt!PsIsSystemWideMitigationOptionSet`函数返回零，则调用`MmCreateSection`：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_ZBEXRNSN8MTJXHM.jpg)

 

`Section Object Address`保存于`nt!MiState+0x604`处，InputMaximumSize 为 0x3000000。

 

`nt!MiCfgInitializeProcess`会调用`nt!MiReferenceCfgVad`向 0xC0802174 处写入`_MI_CFG_BITMAP_INFO`：

 

![](https://bbs.pediy.com/upload/attach/202105/817966_Q2EDGCRZVBB72G8.jpg)

 

![](https://bbs.pediy.com/upload/attach/202105/817966_SW3XUG44FRHSNTP.jpg)

0x03 参阅链接
---------

*   [Windows 10 Control Flow Guard Internals—MJ0011](https://www.powerofcommunity.net/poc2014/mj0011.pdf)
*   [Exploring Control Flow Guard in Windows 10—TrendMicro](http://sjc1-te-ftp.trendmicro.com/assets/wp/exploring-control-flow-guard-in-windows10.pdf)
*   [Control-flow integrity—Wikipedia](https://en.wikipedia.org/wiki/Control-flow_integrity#Microsoft_Control_Flow_Guard)
*   [Bit Test—Wikipedia](https://en.wikipedia.org/wiki/Bit_Test)
*   [_MI_CFG_BITMAP_INFO](https://kernelstruct.gitee.io/kernels/x64/Windows%2010%20%7C%202016/1511%20Threshold%202/_MI_CFG_BITMAP_INFO)
*   [深入探索 Win32 结构化异常处理](https://blog.csdn.net/diamont/article/details/4259590)
*   [Windows 10 Technical Preview adds a feature that blocks untrusted fonts—Microsoft Docs](https://docs.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/feature-to-block-untrusted-fonts#use-registry-editor)
*   [MmCreateSection](http://www.codewarrior.cn/ntdoc/wrk/mm/MmCreateSection.htm)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)