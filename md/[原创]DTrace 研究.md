> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274563.htm)

> [原创]DTrace 研究

DTrace 研究
=========

Windbg 调试器单步异常，直接 gn 不处理。

```
32.0: kd:x86> gn
The context is partially valid. Only x86 user-mode context is available.
WOW64 single step exception - code 4000001e (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
vmp3_6+0x4fa074:
00000000`008fa074 90              nop
32.0: kd:x86> ub
vmp3_6+0x4fa063:
00000000`008fa063 33d3            xor     edx,ebx
00000000`008fa065 ffca            dec     edx
00000000`008fa067 41              inc     ecx
00000000`008fa068 84d9            test    cl,bl
00000000`008fa06a f7da            neg     edx
00000000`008fa06c e91832f0ff      jmp     vmp3_6+0x3fd289 (007fd289)
00000000`008fa071 9d              popfd
00000000`008fa072 0f31            rdtsc
32.0: kd:x86> gn

```

DTrace

 

[DTrace on Windows - Windows drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/dtrace)

 

使用 D 语言编写监控

```
#pragma D option quiet
#pragma D option destructive
 
syscall::Nt*:entry
/ execname == $1 /
{
    /*printf("%s [Caller %s]\n",probefunc, execname);*/
    if(probefunc == "NtQuerySystemInformation") {
        if(arg0 == 35){
            printf("Detect Kernel Debugger\n");
        }
    }
 
    if(probefunc == "NtQueryInformationProcess") {
        if(arg1 == 0x7){
            printf("Detect ProcessDebutPort\n");
        }
 
        if(arg1 == 0x1E){
            printf("Detect ProcessDebugObjectHandle\n");
        }
 
        if(arg1 == 0x1F){
            printf("Detect DebugFlags");
        }
    }
 
    if(probefunc == "NtSetInformationThread"){
        if(arg1 == 0x11){
            printf("HideFromDebugger\n");
        }
    }
 
    if(probefunc == "NtQueryInformationProcess"){
        if(arg1 == 0){
            printf("Query Process Basic Information \n");
        }
    }
 
    if(probefunc == "NtQueryObject"){
        if(arg1 == 2) {
            printf("Query Object Type Information\n");
        }
 
        if(arg1 == 3) {
            printf("Query Object Types Information\n");
        }
    }
 
    if(probefunc == "NtClose") {
        printf("Close Handle : 0x%x\n",arg0);
    }
 
    if(probefunc == "NtSetInformationObject") {
        if(arg1 == 4){
            printf("Set Handle Flag\n");
        }
    }
 
    if(probefunc == "NtGetContextThread"){
        printf("Get thread by thread handle : 0x%x",arg0);
    }
 
    if(probefunc == "NtYieldExecution") {
        printf("NtYieldExecution\n");
    }
 
    if(probefunc == "DbgSetDebugFilterState") {
        printf("DbgSetDebugFilterState\n");
    }
}

```

效果发现其检测内核调试器的存在。

 

使用 Dtrace 进行追踪

```
dtrace.exe -s test.d vmp3.6.exe

```

![](https://bbs.pediy.com/upload/attach/202209/770117_3HDP5Y4C2PCWAFR.png)

 

使用 WinArk 做 inline hook。

 

![](https://bbs.pediy.com/upload/attach/202209/770117_3UYCNHMMF2BFKCF.png)

 

成功绕过 vmp3.6 反内核调试器。

 

![](https://bbs.pediy.com/upload/attach/202209/770117_KUC866EAY69QTDS.png)

其他研究
----

```
0033:00000000`00e06169 48c7c101000000     mov     rcx, 1
0033:00000000`00e06170 48c7c21b52e000     mov     rdx, 0E0521Bh
0033:00000000`00e06177 ff1425d1c09a00     call    qword ptr [9AC0D1h]
0033:00000000`00e0617e cc                 int     3
0033:00000000`00e0617f 488bc8             mov     rcx, rax
0033:00000000`00e06182 ff142577c19a00     call    qword ptr [9AC177h]
0033:00000000`00e06189 4883c428           add     rsp, 28h
0033:00000000`00e0618d cb                 retf

```

```
0: kd> gn
The context is partially valid. Only x86 user-mode context is available.
WOW64 single step exception - code 4000001e (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
00000000`00528bd5 e948073800      jmp     008a9322
32.0: kd:x86> gn
The context is partially valid. Only x86 user-mode context is available.
WOW64 single step exception - code 4000001e (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
00000000`0052a6b6 e98dee3700      jmp     008a9548
32.0: kd:x86> gn
The context is partially valid. Only x86 user-mode context is available.
WOW64 single step exception - code 4000001e (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
00000000`00529a6a e9f7f93700      jmp     008a9466
32.0: kd:x86> gn
The context is partially valid. Only x86 user-mode context is available.
WOW64 single step exception - code 4000001e (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
00000000`00527c85 e9c6153800      jmp     008a9250
32.0: kd:x86> gn

```

[[2022 夏季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

[#系统底层](forum-4-1-2.htm)