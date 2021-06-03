> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1451677-1-1.html)

> [md]# hook 目的 ***360hookport 驱动所实现的功能就是在不修改系统的 SSDT 表 / ShadowSSDT 表的情况下拦截一些重要的系统调用，并进行根据一定的过滤规则进行参数和返回值的 .........

 ![](https://avatar.52pojie.cn/data/avatar/001/28/98/46_avatar_middle.jpg) 镇北看雪 _ 本帖最后由 镇北看雪 于 2021-6-2 14:38 编辑_  

hook 目的
=======

* * *

360hookport 驱动所实现的功能就是在不修改系统的 SSDT 表 / ShadowSSDT 表的情况下拦截一些重要的系统调用，并进行根据一定的过滤规则进行参数和返回值的检查，阻止一些不安全或危险的系统调用。

hook 思路
=======

* * *

360hookport 的 hook 思路就是 ：因为 32 位系统下所有的 API 调用一般都是通过 sysenter 快速系统调用指令进入到内核中，然后在内核中调用的第一个函数就是 ntoskrnl.exe 的 KiFastCallEntry 函数，然后根据从 3 环传进的系统服务号查询系统描述符表（SSDT/ShadowSSDT 表）得到其对应的地址后然后调用它。  
360hookport 做的就是将就是利用 inline hook 将这个 KiFastCallEntry 拦截，然后 jmp 到 hookport 模块中，调用 ssdt_fiter()函数根据 SERVICE_FILTER_INFO_TABLE 和 FILTERFUN_RULE_TABLE 表中的 hook 开关和代 {过}{滤} 理函数开关是否打开来判断是否需要对此次调用进行 hook，如果需要 hook 就返回 SERVICE_FILTER_INFO_TABLE 表中对应的服务例程的代 {过}{滤} 理函数，如果不需要 hook 就返回原始例程的地址，然后通过 push KiFastCallEntryhook 位置的下一条指令的地址，然后执行 ret 指令返回到 KiFastCallEntryhook 位置的下一条指令处继续执行。  
这样如果返回的是我们代 {过}{滤} 理函数的地址，接下来 KiFastCallEntry 就会调用我们的代 {过}{滤} 理函数，如果返回的是原始例程的地址 KiFastCallEntry 就会调用原始例程。

![](https://attach.52pojie.cn/forum/202106/01/235017funnum699956o922.png)

**M~7F7AP@~4TYR@4@7)F(C.png** _(34.39 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY2OHw5NDY1NjU3ZHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:50 上传

hook 细节
=======

* * *

两个重要的数据结构
---------

```
00000000 _SERVICE_FILTER_INFO_TABLE struc ; (sizeof=0x5DDC, align=0x4, copyof_10)
00000000 SSDTCnt dd ?                                            ; Shadowssdt的最大服务数目（shadowssdt中服务要包含ssdt中的）
00000004 SavedSSDTServiceAddress dd 1001 dup(?)                  ; 原始SSDT例程地址
00000FA8 ProxySSDTServiceAddress dd 1001 dup(?)                  ; SSDT调用的代{过}{滤}理函数地址
00001F4C SavedShadowSSDTServiceAddress dd 1001 dup(?)            ; 原始shadowSSDT例程地址
00002EF0 ProxyShadowSSDTServiceAddress dd 1001 dup(?)            ; ShadowSSDT调用的代{过}{滤}理函数地址
00003E94 SwitchTableForSSDT dd 1001 dup(?)                       ; SSDT代{过}{滤}理开关（HOOK开关）
00004E38 SwitchTableForShadowSSDT dd 1001 dup(?)                 ; ShadowSSDT代{过}{滤}理函数开关
00005DDC _SERVICE_FILTER_INFO_TABLE ends

00000000 _FILTERFUN_RULE_TABLE struc ; (sizeof=0x1A8, mappedto_12)
00000000 bSize dd ?                                    ; FILTERFUN_RULE_TABLE结构的的大小
00000004 pNextRuleTable dd ?                           ; 指向下一个张FILTERFUN_RULE_TABLE表包含新的过滤规则
00000008 IsFilterFunFilledReady dd ?                   ; 过滤函数开关（表示过滤函数是否准备好）
0000000C CheckServiceRoutine dd 101 dup(?)             ; 过滤函数表，一共有101（0x65）个
000001A0 ShadowSSDTRuleTableBase dd ?                  ; ShadowSSDT过滤规则表的基地址
000001A4 SSDTRuleTableBase dd ?                        ; SSDT过滤规则表的基地址
000001A8 _FILTERFUN_RULE_TABLE ends


```

ssdt_fiter（）
------------

看一下 ssdt_fiter()函数，其内部会先判断是 ShadowSSDT 调用还是 SSDT 调用，接着判断代 {过}{滤} 理函数 hook 开关是否打开，过滤函数表是否准备好。只有都打开后才会返回代 {过}{滤} 理函数地址，并将原函数的地址保存在 SERVICE_FILTER_INFO_TABLE 的原程序例程数组中

```
ULONG __stdcall ssdt_filter(unsigned int call_index, int ori_pfn, int ServiceBase)
{
  if ( ServiceBase == g__ssdt_bast && call_index <= g_max_shadowIndex )
    goto LABEL_15;
  if ( ServiceBase == g_shadow_ssdt_base && call_index <= g_max_SSDT_index )        // 如果是ShadowSSDT调用
  {
    if ( g_myServiceBase->SwitchTableForShadowSSDT[call_index] && check_needto_filter(call_index, 1) )  //判断代{过}{滤}理函数hook开关是否打开，过滤表是否准备好
    {
      g_myServiceBase->SavedShadowSSDTServiceAddress[call_index] = ori_pfn;                             //将原始例程的地址保存到我们的SERVICE_FILTER_INFO_TABLE表中
      return g_myServiceBase->ProxyShadowSSDTServiceAddress[call_index];                                //返回代{过}{滤}理函数地址
    }                                          
    return ori_pfn;                                                                                     // 返回原例程地址
  }
  if ( ServiceBase == *(_DWORD *)addr_g_KeServiceDescriptorTable )                  //如果是SSDT调用
  {
LABEL_15:
    if ( g_myServiceBase->SwitchTableForSSDT[call_index] && check_needto_filter(call_index, 0) )         //判断理函数hook开关是否打开，过滤表是否准备好
    {
      g_myServiceBase->SavedSSDTServiceAddress[call_index] = ori_pfn;                                    //将原始例程的地址保存到我们的SERVICE_FILTER_INFO_TABLE表中                                       
      return g_myServiceBase->ProxySSDTServiceAddress[call_index];                                       // 返回代{过}{滤}理函数地址
    }
  }                                             
  return ori_pfn;                                                                                        //返回原例程地址
}

```

一般的代 {过}{滤} 理函数
---------------

我们来看一下一般的代 {过}{滤} 理函数是怎么处理的，有一些代 {过}{滤} 理函数还会进行额外的处理。我们要检查系统调用是否安全肯定要检查其对应的参数，代 {过}{滤} 理函数会先调用 pre_check 函数，第一个参数是此服务在 FILTERFUN_RULE_TABLE 的 CheckServiceRoutine 过滤函数数组中的对应的索引（一共有 0x65 个过滤函数对应 0-0x64 过滤索引），第二个参数为对应服务的参数数组（参数以数组的形式传递）。

![](https://attach.52pojie.cn/forum/202106/01/235055akicdxgpdwxazgdr.png)

**J2_IX$__[D%%K7B[IM{CR(D.png** _(46.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY2OXxjZmI3OWJkMHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:50 上传

在 pre_check（）函数内部其会先判断 FILTERFUN_RULE_TABLE 的 IsFilterFunFilledReady 过滤规则表是否准备就绪，然后会根据传入的过滤函数表索引获取所有 FILTERFUN_RULE_TABLE 表中对应的过滤函数并调用，（过滤函数的实现不在 hookport 模块中，其实现在其他模块中，然后通过 hookport 提供的接口将过滤函数的地址填写到 FILTERFUN_RULE_TABLE 的过滤函数数组中）。过滤函数中会对参数进行检查，如果有问题就返回错误，没问题就返回一个函数地址 CheckReturn()。  
光检查参数不行，有时候还需要检查返回值，CheckReturn() 这个函数就是从来检测返回值的。接着会继续遍历 FILTERFUN_RULE_TABLE 的 pNextRuleTable 寻找下一张 FILTERFUN_RULE_TABLE 表继续调用过滤函数。然后返回返回值。最多有 16 张 FILTERFUN_RULE_TABLE 表。  
当返回值都没问题后，将调用各个 FILTERFUN_RULE_TABLE 表中过滤函数返回的 CheckReturn()函数的地址保存到数组中并返回到代 {过}{滤} 理函数中。

![](https://attach.52pojie.cn/forum/202106/01/235123dv4g5mwvo0gf0qw5.png)

**S}}0W2ZLT%AH}}5]1H$HO0A.png** _(85.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3MHwwOTUxODg3MXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:51 上传

然后代 {过}{滤} 理函数会判断此次调用时 SSDT 调用还是 ShadowSSDT 调用，然后从 SERVICE_FILTER_INFO_TABLE 中获得原始的服务例程并调用，然后检查返回值，如果调用出错肯定就不用管了，如果调用成功就调用我们刚刚调用 pre_check 函数返回的 CheckReturn 函数数组，依次调用各个 CheckReturn 函数，此函数会在内部检查返回值然后返回，如果所有的返回值都没问题代 {过}{滤} 理函数就可以将调用原始例程的返回值返回了。

![](https://attach.52pojie.cn/forum/202106/01/235135cul8kdek8z83k3dp.png)

**H~C608`]A4K[5_GU8WTLCCO.png** _(65.85 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3MXxmOTE5Y2Y0OXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:51 上传

具体细节
----

DriverEntry 入口处先判断操作系统的版本信息，然后查看是否处于安全模式下，如果是就返回错误代码。

![](https://attach.52pojie.cn/forum/202106/01/235147mghwh7vwtwzwgiph.png)

**ZQSM~4_L)N0J]]E(O[$COJS.png** _(33.36 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3Mnw5ZWQ0OTE0OHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:51 上传

接着调用 init_and_hook_KiFastCallEntry_KeUserModeCallback（）函数初始化，并 hook_KiFastCallEntry 和 KeUserModeCallback。  
在函数内部其会先调用 findmodule（）查看是否加载了 win32k.sys 模块，findmodule 是通过 ZwQuerySystemInformation 传入 SystemModuleInformation 枚举内核模块。注意得到的第一个模块就是 ntoskrnl.sys 的模块。获得 SYSTEM_MODULE_INFORMATION_ENTRY 结构数组，然后根据模块的名称进行判断得到模块的基地址和大小

```
NTSTATUS 
ZwQuerySystemInformation (
    SYSTEM_INFORMATION_CLASS SystemInformationClass, 
    PVOID SystemInformation, 
    ULONG SystemInformationLength, 
    ULONG *ReturnLength);

```

如果加载了此模块就调用 search_ssdtTable_byHardCode（）得到 shadowSSDT 表的基地址。search_ssdtTable_byHardCode（）函数时在函数 KeAddSystemServiceTable 根据特征码 0x888d 搜寻 KeServiceDescriptorTableShadow 地址，或者在函数 KeRemoteSystemServiceTable 中根据特征码 0x8889 得到 KeServiceDescriptorTableShadow 地址

![](https://attach.52pojie.cn/forum/202106/01/235200azphpww1jymjwmie.png)

**$OXC]Z()OK2QDBPB56D6V2U.png** _(15.17 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3M3wxZmUzZjBhZHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:52 上传

接着调用 GetProcAddress 得到 KeServiceDescriptorTable 地址进一步得到 SSDT 表的基地址。  
GetProcAddress 内部通过 findmodule 得到第一个内核模块的基地址，第一个内核模块也就是 ntoskrnl.exe，然后解析 ntoskrnl.exePE 头文件根据导出表得到 KeServiceDescriptorTable 的地址。  
因为 KeServiceDescriptorTable 是由 ntoskrnl.exe 导出的，而 win32k.sys 并没有导出 KeServiceDescriptorTableShadow

![](https://attach.52pojie.cn/forum/202106/01/235219aavmmugmzvi8xlu0.png)

**J2KEX]AC9T)@4`(QLLX[U71.png** _(30.69 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3NHwzODI0Njk3MHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:52 上传

接着调用 CollectAllHookFun_Index（）得到所有需要 hook 的服务例程的服务索引，对于有对应 ZW 系列导出函数的索引我们直接通过获得 ZW 系列地址然后，在函数开头位置获得索引号，而如果 hook 的函数没有对应的 ZW 系列导出函数则判断系统的版本信息，根据硬编码得到对应的服务索引  
如果索引号获取不到，或者得到了大于 1000 的索引号，统一将服务索引号设置为 1000，（后面所有服务号为 1000 的 hook 开关都会被关闭，防止错误的调用）。

![](https://attach.52pojie.cn/forum/202106/01/235235hqxq5nxzyp8i9qf9.png)

**UGX@E(M~4AXPK2{{MP][ZUU.png** _(35.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3NXxlMDYyZTJjZXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:52 上传

因为 ShadowSSDT 表中包含了 SSDT 表，所以得到 ShadowSSDT 表中最多有多少项就得到了所有服务的最大数目。获得服务的最大数目后申请内存存放 SERVICE_FILTER_INFO_TABLE 表并将最大的服务数存到第一个字段中 SSDTCnt

![](https://attach.52pojie.cn/forum/202106/01/235247h818tnttnps88cmp.png)

**CF6TF{ANKSIO)W_UQ~NLCNG.png** _(22.68 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3NnwxOTRkYzdhOXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:52 上传

接着根据刚刚获得的各个需要 hook 的服务例程的索引将对应的代 {过}{滤} 理函数的地址写到 SERVICE_FILTER_INFO_TABLE 中代 {过}{滤} 理函数数组 ProxySSDTServiceAddress 对应的位置中。

![](https://attach.52pojie.cn/forum/202106/01/235302gy005caourr5u59m.png)

**3HH4E0}69]6jZ}1[ML8HY.png** _(49.01 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3N3xiMzYyOGMzZnwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:53 上传

然后就是调用 hook_KiFastCallEntry() 了，其是通过 ssdt hook NtSetEvent 函数，然后主动调用 zwSetEvent 函数并传入特殊的句柄值 0x288C58F1，此函数就会调用我们在 ssdt hook 中安装的 MyNtSetEvent。然后在 MySetEvent 中进行 KiFastCallEntry inline hook 的安装。

![](https://attach.52pojie.cn/forum/202106/01/235314innzik0i0adyeliy.png)

**IXQ1{IM9SI983I]1L6HTU`D.png** _(51.87 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3OHw0MDE5MmVlZXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:53 上传

看一下 MySetEvent，其会先判断传入的句柄值是否为 0x288c58f1，如果不是那就不是我们的调用直接调用原始例程 NtSetEvent，如果是证明这是我们自己的调用就进行 KiFastCallEntry inlinehook 的安装，  
360 hookport 的 KiFastCallEntry inlinehook 位置很好，向 8053e621 地址处写入一级 jmp 跳转指令跳到 360hookport 的二级跳转指令处。 然后二级跳转指令又跳转到 KiFastCallEntryFiter 入口处。一级地址跳转前 eax 为服务号，而 ebx 为从系统服务表中取出的服务例程地址。  
那么我们如何在 MySetEvent 中找到这个 inline hook 的位置呢，因为我们的 MySetEvent 是通过 call ebx 调用的，那么在栈中就一定存在 call ebx 下一条指令的地址 [ebp +4]。我们通过栈回溯找到 call ebx 下一条指令的地址然后往上进行硬编码匹配，当找到 0x02E9C1E12B 时表示我们找到了 hook 的位置  
然后将一级跳转指令写入此处。inline hook 安装完成后恢复 NtSetEvent 的 ssdt hook。返回到 hook_KiFastCallEntry() 函数中检查刚刚调用 MyNtSetEvent（）是否完成了 KiFastCallEntry 的 inline hook，如果没有下面做的和 MyNtSetEvent 做的差不多，就是进行 inline hook KiFastCallEntry。

![](https://attach.52pojie.cn/forum/202106/01/235327ozsrxhstos9o9nt7.png)

**3Y`0TEEIFYEK$[[Q)JWM][9.png** _(112.47 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY3OXw4MTgwZmM5ZXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:53 上传

```
nt!KiFastCallEntry+0xcc:
8053e60c ff0538f6dfff    inc     dword ptr ds:[0FFDFF638h]
8053e612 8bf2            mov     esi,edx
8053e614 8b5f0c          mov     ebx,dword ptr [edi+0Ch]
8053e617 33c9            xor     ecx,ecx
8053e619 8a0c18          mov     cl,byte ptr [eax+ebx]
8053e61c 8b3f            mov     edi,dword ptr [edi]                                         ；edi为系统服务表基地址
8053e61e 8b1c87          mov     ebx,dword ptr [edi+eax*4]                                   ；eax =系统服务号，ebx为对应的系统服务地址
8053e621 2be1            sub     esp,ecx
8053e623 c1e902          shr     ecx,2
8053e626 8bfc            mov     edi,esp
8053e628 3b35d4995580    cmp     esi,dword ptr [nt!MmUserProbeAddress (805599d4)]
8053e62e 0f83a8010000    jae     nt!KiSystemCallExit2+0x9f (8053e7dc)
8053e634 f3a5            rep movs dword ptr es:[edi],dword ptr [esi]
8053e636 ffd3            call    ebx                                                         ；调用相应的系统服务
8053e638 8be5            mov     esp,ebp

```

接着调用 PsSetCreateProcessNotifyRoutine 创建了一个进程通知回调，其内不会调用 pre_check() 并传入 0x45 过滤函数索引。对应的过滤函数会检查进程创建是否存在问题。

![](https://attach.52pojie.cn/forum/202106/01/235341on035ad33ffbhhk2.png)

**UZJCV$PBW~FDAN%YMD$%SA2.png** _(25.39 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY4MHwwYzlmOWUwM3wxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:53 上传

然后会判断 win32k.sys 模块是否加载，如果没加载就将 NtSetSystemInformation 的 hook 开关先打开，如果 win32k.sys 已经加载了就直接获取 csrss.exe 进程的 PID，然后将进程空间切到 csrss.exe 所在的进程空间中，然后 IAThook win32k.sys 模块中的 KeUserModeCallback 函数。

![](https://attach.52pojie.cn/forum/202106/01/235354ptnllnfmlq7utbfq.png)

**V7QC4IW%5O%3YU8LD{P3XU0.png** _(39.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY4MXw2OGE0ZGYwMnwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:53 上传

这里注意为什么将进程切换到 csrss.exe 所在的进程空间中再 hook_KeUserModeCallback 呢，实际没必要非得切换都 csrss.exe 进程空间中，主要不是在 System 和 smss.exe 进程空间中都可以，因为在在 System 和 smss.exe 进程空间中 win32k.sys 其虚拟地址并没有被映射物理内存，访问无效

```
NTSTATUS 
ZwSetSystemInformation (
    SYSTEM_INFORMATION_CLASS SystemInformationClass, 
    PVOID SystemInformation, 
    ULONG SystemInformationLength);

```

那么如果 win32k.sys 没有加载为什么要先打开 NtSetSystemInformation 的 hook 开关呢，我们分析一下 NtSetSystemInformation 对应的代 {过}{滤} 理函数，发现其会在调用完过滤函数后判断参数 SystemInformationClass 是否为 SystemExtendedServiceTableInformation（0x26），  
如果是说明 win32k.sys 正在加载然后对 KeAddSystemServiceTable 进行所在的 ntoskrnl.exe 的 EAT_hook。   因为 windows 系统再加载时运行的第一个用户进程就是 smss.exe 会话管理器，其会调用 NtSetSystemInformation 并传入参数 SystemExtendedServiceTableInformation（0x26）来加载 win32k.sys  
随后 win32k.sys 就会在 DriverEntry 驱动入口调用 KeAddSystemServiceTable 函数添加 ShadowSSDT 表。 HOOK_KeAddSystemServiceTable 的原因是判断添加的是否为 win32k.sys 的 ShadowSSDT 表，防止攻击者加载自己的 ShadowSSDT 替换正确的 ShadowSSDT 表  
然后 NtSetSystemInformation 的代 {过}{滤} 理函数会继续判断 win32k.sys 是否已经加载，如果加载了就 hook_KeUserModeCallback 函数。

![](https://attach.52pojie.cn/forum/202106/01/235456p3rvvvcxx64qj3kc.png)

**A3A]6J[8F[E60CZ2OKLCMNM.png** _(96.86 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY4M3w2OTRkYjc4OHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:54 上传

那么为什么要 hook_KeUserModeCallback 函数呢，调用 KeUserModeCallback 函数时在内核中调用用户层代码的一种手段，例如我们利用全局钩子注入 dll，或者是利用输入法入 dll，或者是设置鼠标键盘消息记录钩子 WH_JOURNALRECORD，都会调用 win32k.sys 模块的 KeUserModeCallback 函数  
我们将此函数 hook 了就可以监控这些这些操作。我们看一看 hook 后的 MyKeUserModeCallback，其通过判断 apiNumber 是否为 ClientLoadLibrary，ClientImmLoadLayout 或 fnHkOPTINLPEVENTMSG 来监控这些操作。如果没问题就正常返回。

![](https://attach.52pojie.cn/forum/202106/01/235420mmi370s8zhrui8zr.png)

**[GZM]~X`[{X%G$V~)BI$C]9.png** _(61.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY4Mnw0YWJlNzY4NnwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:54 上传

做完这些之后在设置过滤函数索引与此服务的实际服务索引之间的对应关系数组 FilterToServiceIndex[0x65]。例如 NtCreateKey 的过滤函数索引为 0，那么 FilterToServiceIndex[0] 就等于 NtCreateKey 在服务表中的索引。  
最后最后在调用 CmRegisterCallback 函数注册注册表回调来检测 KiFastCallEntry 的 inline hook 是否安装成功。在注册表通知回调函数中如果检测到 KiFastCallEntry 的 inline hook 还没有安装就再次进行 KiFastCallEntry 的 inline hook

![](https://attach.52pojie.cn/forum/202106/01/235542fz57ummumr86g85i.png)

**~D%{@YHU2HU_I%$BUHAO~3U.png** _(33.3 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY4NHw3MTdiZmFkOXwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:55 上传

驱动向外提供了 3 个扩展接口：  
1.g_port_extension_v1_for_AddRule   ：向 FILTERFUN_RULE_TABLE 规则表中添加新的规则（即添加一张新的 FILTERFUN_RULE_TABLE 表，FILTERFUN_RULE_TABLE 的 pNextRulTable 就指向下一张 FILTERFUN_RULE_TABLE 表）

2.g_port_extension_v2_register_ssdt_check_handle_callback：设置 FILTERFUN_RULE_TABLE 中过滤函数以及对应的 SERVICE_FILTER_INFO_TABLE 表中的 SSDT/ShadowSSDThook 开关（其会将所有服务索引为 1000 的服务的 hook 开关关闭）

3.g_port_extension_v3_register_ruletable_base：设置过滤规则表对应的过滤规则

![](https://attach.52pojie.cn/forum/202106/01/235554dnymnicucm8pctci.png)

**BL{R})}C5V(6N{PT~WYPIQ2.png** _(30.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODY4NXwwN2EyZGRhZHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3&nothumb=yes)

2021-6-1 23:55 上传

参考：[https://bbs.pediy.com/thread-99460.htm](https://bbs.pediy.com/thread-99460.htm)![](https://avatar.52pojie.cn/data/avatar/000/25/89/14_avatar_middle.jpg)鱼无论次 _ 本帖最后由 鱼无论次 于 2021-6-2 16:45 编辑_  

> [镇北看雪 发表于 2021-6-2 15:42](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38782776&ptid=1451677)  
> 前辈厉害

  
谦虚了，我记得 360 的很多人逆过，加油 360SelfProtection 也不太难，也挺有意思的  
我记得还有笔记，代码太垃圾就不分享了。  
one 转换成 pdf 好糊

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[数字驱动分析笔记之 360SelfProtection.zip](forum.php?mod=attachment&aid=MjI5ODg1OXwyZDI2NDRhNHwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3)

2021-6-2 16:37 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

2.69 MB, 下载次数: 12, 下载积分: 吾爱币 -1 CB

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[数字驱动分析笔记之 HookPort.zip](forum.php?mod=attachment&aid=MjI5ODg2MHxlYzZiMDNiNnwxNjIyNzEwMzIzfDIxMzQzMXwxNDUxNjc3)

2021-6-2 16:38 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

1.17 MB, 下载次数: 12, 下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/data/avatar/001/63/08/72_avatar_middle.jpg)djxding 感谢分享。  
学习一下，不过对我来说感觉太难了。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)tzlqjyx 标记下，收藏了好多教程，慢慢学习 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) aonima 学习了，感谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) x800600 高手的杰作，值的学习![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)打瓜瓜 码住，留着学习![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)老衲不怕和尚 谢谢分享  学到了很多 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) JakerPower 看不懂，大神功力了得，膜拜收藏学习。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)auko48 太厉害了，学习下 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) zzy07112 大佬牛逼