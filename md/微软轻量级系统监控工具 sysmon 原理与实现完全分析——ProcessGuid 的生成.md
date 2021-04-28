> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267252.htm)

Sysmon 的众多事件看起来都是独立存在的，但是它们确实都是由每个进程的产生的，而关联这些信息的东西正是 ProcessGuid，这个对进程是唯一的。如下图

Event 23

![](https://bbs.pediy.com/upload/attach/202104/36653_FEDN34FMPDHZFNA.jpg)

![](https://bbs.pediy.com/upload/attach/202104/36653_S8TZS39GK9VA6UV.jpg)

都会有个 ProcessGuid 字段，今天的这篇文章主要就是讲解 Sysmo 是如何生成事件的 ProcessGuid 的。  
Sysmon 有两个地方会调用 ProcessGuid 创建的函数，第一个是初始化的时候，在服务启动的时候会初始化一次当前机器里 已经打开进程的 ProcessGuid

![](https://bbs.pediy.com/upload/tmp/36653_U8Y9XFJV8NM3K3J.jpg)

查看这个红色框的函数 Sysmon_InitProcessCache

![](https://bbs.pediy.com/upload/attach/202104/36653_DN2R9XPWKG836CE.jpg)

首先会枚举当前机器所有进程，然后下面就会调用 Guid 的创建的函数

![](https://bbs.pediy.com/upload/tmp/36653_9GGDHYR9DC27BC3.jpg)

传入 ProcessPid，进程的 pid

![](https://bbs.pediy.com/upload/tmp/36653_AJABQ6UBNPXWM8Z.jpg)

然后像 Sysmon 的驱动发送 IoControl 码去获取该进程的相关信息，获取哪些信息呢？接下来我们看内核驱动。  
Sysmom 内核驱动处理 Io 的函数 SysmonDispatchIrpDeviceIO

![](https://bbs.pediy.com/upload/tmp/36653_SR4KT2Z2R7JDTXP.jpg)

找到对应的 IoControl

![](https://bbs.pediy.com/upload/attach/202104/36653_XKU47J3FSM54T7Q.jpg)

这里就是处理的地方，可以发现会先检验传入参数的大小是不是正确的，然后会调用  
SysmonInsertCreateThreadProcess(pUserBuffer->ProcessId, 1u, pUserBuffer->NewAdd, pIrp); 去获取该进程 pid 的进程相关的信息。

首选会获取进程的 Token 相关的信息

![](https://bbs.pediy.com/upload/attach/202104/36653_SU6Y8EK6D6Z2844.jpg)

在这里函数里使用了 ZwOpenProcessToken(ProcessHandle, 0x20008u, &TokenHandle); 去获取进程 token 句柄

![](https://bbs.pediy.com/upload/attach/202104/36653_NW76G7X4E6DYX6Q.jpg)

然后 Query 句柄，使用 ZwQueryInformationToken 函数去获取，TokenInfomationClass 分别是 TokenUser、TokenGroups、TokenStatistics、TokenIntegrityLevel。最后会把这些信息输出到传输的缓冲区结构体里

![](https://bbs.pediy.com/upload/attach/202104/36653_AXS7UEQV9TFH549.jpg)

![](https://bbs.pediy.com/upload/attach/202104/36653_4WFYYFFYM9WVHYA.jpg)

获取了以上信息后，还会继续获取进程的全路径和参数

![](https://bbs.pediy.com/upload/attach/202104/36653_4494BEGT6YZXMGE.jpg)

AttackProcess 进程后去获取进程的参数的获取，是通过 PEB—>ProcessParameter 获取

![](https://bbs.pediy.com/upload/attach/202104/36653_EWHNZDQQ7PCRC6R.jpg)

![](https://bbs.pediy.com/upload/tmp/36653_DM9ZD244XXV659Q.jpg)

接下来就是获取进程创建时间

```
ZwQueryInformationProcess(ProcessHandlev9, ProcessTimes, &KernelUserTime, 0x20u, 0);if (!KernelUserTime.CreateTime.QuadPart) {
    ResultLength = 48;    if (ZwQuerySystemInformation(SystemTimeOfDayInformation, &SystemTimeDelayInformation, 0x30u, &ResultLength) >= 0) KernelUserTime.CreateTime.QuadPart = SystemTimeDelayInformation.BootTime.QuadPart

    - SystemTimeDelayInformation.BootTimeBias;
}

```

![](https://bbs.pediy.com/upload/attach/202104/36653_DXJDKX3A2P855KY.jpg)

继续还会计算改进程的 hash

![](https://bbs.pediy.com/upload/attach/202104/36653_6CH7VWSZPW27BRV.jpg)

最后把这些信息输出到 ring3 输入的缓冲区里

![](https://bbs.pediy.com/upload/attach/202104/36653_Q2REW9E52NBHHY5.jpg)

![](https://bbs.pediy.com/upload/attach/202104/36653_28U7ADW4AVG8BC2.jpg)

结构体是：

```
struct _Report_Process {
    Report_Common_Header Header;
    CHAR data[16];
    ULONG ProcessPid;
    ULONG ParentPid;
    ULONG SessionId;
    ULONG UserSid;
    LARGE_INTEGER CreateTime;
    LUID AuthenticationId;
    ULONG TokenIsAppContainer;
    LUID TokenId;
    ULONG HashingalgorithmRule;
    DWORD DataChunkLength[6];
    CHAR Data[1];
};

```

后面的数据 data 有

SID  
TokenIntegrityLevel  
ImageFileName  
HASH  
CommandLine  
DirectoryPath

驱动过程结束后继续回到应用层调用的地方，应用层获取了以上数据后就是对数据进程组装来生成 ProcessGuid。

![](https://bbs.pediy.com/upload/attach/202104/36653_HPMWF55DVTS7G38.jpg)

可以看到函数 v7 = (const __m128i *)SysmonCreateGuid(&OutBuffer.TokenId, (int)&OutBuffer.CreateTime, &Guid, 0x10000000); 传入的参数是 TokenId、CreateTime  
继续分析该函数内部实现

![](https://bbs.pediy.com/upload/attach/202104/36653_CYWHQEVD9VT4RAU.jpg)

OutGuid 的参数是 v6

V6 的 data1 赋值为 g_Guid_Data1，这个从哪里来的呢，答案很简单。

在另外一个函数里有个给 g_Guid_Data1 赋值的地方

![](https://bbs.pediy.com/upload/attach/202104/36653_MWD6X2JNSUHCETY.jpg)

可以看到读取注册表键值为 MachineGuid 的键值获取这个键值 Guid 的第一个 Data1 给 g_Guid_Data1，网上翻阅，会看到

![](https://bbs.pediy.com/upload/attach/202104/36653_SWWZ9ZQ479XDA42.jpg)

哦，答案一目了然了，是  
v2 = RegOpenKeyW(HKEY_LOCAL_MACHINE, L”SOFTWARE\Microsoft\Cryptography”, &phkResult);

我们看下自己机器的注册表里

![](https://bbs.pediy.com/upload/attach/202104/36653_Y99U3F756B3Y87M.jpg)

确实存在一个 Guid，sysmon 只去了第一个 data1.

接着看 _(_DWORD_ )&v6->Data2 = v8;

V8 就是 进程的 CreateTime，只是经过了

RtlTimeToSecondsSince1970(Time, &v8); 的时间格式化

最后就是 8 个字节的数据

_(_DWORD_ )v6->Data4 = GudingValue | TokenIdv5->HighPart;  
_(_DWORD_ )&v6->Data4[4] = TokenIdv5->LowPart;

GudingValue 是哥固定值 0x10000000

填充 Tokenid 的高位与低位

现在我们大概清楚 ProcessGuid 的算法就是

MachineIdData1 - 进程创建时间 - TokenId 组合而成的

以上是初始化的时候 ProcessGuid 过程，第二个部分是实时从内核获取进程线程事件后，Sleep(500u); 每 500 毫秒从驱动内获得事件缓冲数据

![](https://bbs.pediy.com/upload/attach/202104/36653_Q4RDB5689F9KS37.jpg)

然后调用 SysmonDisplayEvent(lpOutBuffer); 去解析事件

![](https://bbs.pediy.com/upload/attach/202104/36653_W39P9ZSPX4FB763.jpg)

![](https://bbs.pediy.com/upload/attach/202104/36653_C7WANJZ5K5XWVYE.jpg)

事件类型为 7 的时候，同样会调用以上函数 Sysmon_DeviceIo_Process_Pid_Guid(_(_DWORD_ )v22, (GUID *)&v67, 4, SizeInWords); 去生成 ProcessGuid，事件类型 7 就是我们上一篇文章讲的进程线程事件。

最后就是实验我们的结果，写代码

```
if (SUCCEEDED(StringCchPrintf(szDriverKey, MAX_PATH, _T(“\\.\ % s”), _T(“SysmonDrv”)))) {
    HANDLE hObjectDrv = CreateFile(szDriverKey, GENERIC_READ | GENERIC_WRITE, 0, 0, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0);    if (hObjectDrv != INVALID_HANDLE_VALUE) {
        LARGE_INTEGER Request;
        Request.LowPart = GetCurrentProcessId();
        Request.HighPart = FALSE;
        BYTE OutBuffer[4002] = {            0
        };
        ULONG BytesReturned = 0;        if (SUCCEEDED(DeviceIoControl(hObjectDrv, SYSMON_REQUEST_PROCESS_INFO, &Request, 8, OutBuffer, sizeof(OutBuffer), &BytesReturned, 0))) {            if (BytesReturned) {
                Sysmon_Report_Process * pSysmon_Report_Process = (Sysmon_Report_Process * ) & OutBuffer[0];                if (pSysmon_Report_Process - >Header.ReportSize) {                    typedef void(__stdcall * pRtlTimeToSecondsSince1970)(LARGE_INTEGER * , LUID * );
                    LUID CreateTime;
                    GUID ProcessGuid = {                        0
                    };
                    ProcessGuid.Data1 = 0x118a010c;
                    pRtlTimeToSecondsSince1970 RtlTimeToSecondsSince1970 = (pRtlTimeToSecondsSince1970) GetProcAddress(GetModuleHandle(_T("ntdll.dll")), "RtlTimeToSecondsSince1970");                    if (RtlTimeToSecondsSince1970) {

                        RtlTimeToSecondsSince1970( & pSysmon_Report_Process - >CreateTime, &CreateTime); * (DWORD * ) & ProcessGuid.Data2 = CreateTime.LowPart; * (DWORD * ) ProcessGuid.Data4 = pSysmon_Report_Process - >TokenId.HighPart; * (DWORD * ) & ProcessGuid.Data4[4] = pSysmon_Report_Process - >TokenId.LowPart;
                    }
                    CheckServiceOk = TRUE;
                }
            }
        }
        CloseHandle(hObjectDrv);
        hObjectDrv = NULL;
    }
}

```

![](https://bbs.pediy.com/upload/attach/202104/36653_EREBF99XE59JAW4.jpg)

我们计算自己进程的结果是

![](https://bbs.pediy.com/upload/attach/202104/36653_BD6JYYWEGRJQETZ.jpg)

ProcessGuid = {118A010C-7A53-5FAB-0000-0010B1AED716}

再看看 sysmon 里记录的记过是

![](https://bbs.pediy.com/upload/attach/202104/36653_6B9EAJBAC6MGJJP.jpg)

是一样的，算法结果没问题

这篇文章就到此结束，希望能对读者有些帮助，当然我这个是 v8 的 sysmon 的版本对应的算法，最新版 v11 有点小的变动，但变化是在内核里的数据，其他变化不大，读者可以自己去研究。

[[公告]5 月 14 日腾讯安全零信任发展趋势论坛重磅开幕！邀您一起从 “零” 开始，共建信任！！](https://zta.insecworld.com/?utm_campaign=MJTG&utm_source=KX&utm_medium=WZLJ)