> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268435.htm)

> [原创]VMProtect 分析（二）

[VmProtect 分析（一）](https://bbs.pediy.com/thread-268377.htm)

Tls 回调函数（上）
===========

继续看下程序的 Handler 是如何计算的，查看 VmJMP 代码：

![](https://bbs.pediy.com/upload/attach/202107/906381_PW9E4XFBFSC9WGY.jpg)

算法教简单：**Handler 表中根据 BYTE:[RSI-1] 取偏移，循环右移 5 位 ，再加上 Handler 基址**。

寄存器状态如下：

![](https://bbs.pediy.com/upload/attach/202107/906381_8GU6EZX73D8FBPM.jpg)

程序的 Handler 表 (部分截图)（00007FF7C63B065C , L800）：

![](https://bbs.pediy.com/upload/attach/202107/906381_FBEEKY8T4AA6VR6.jpg)

Handler 数量有 0n256 个之多，我们此次**将 Tls 回调函数作为分析目标**，先走一小步，只看那些会用到的，没用到的先不管它。

首先需要确定 Tls 回调函数的结束地址，在启动中断在 Tls 时，查看调用栈（下图），Tls 回调执行完毕后，会返回到 00007FFDBB969A1D 这个地址，可以在这个地址下断，用于标识 Tls 回调函数已经执行完毕。

![](https://bbs.pediy.com/upload/attach/202107/906381_6NAGHKK5AVCWCBC.jpg)  

然后我们写个插件，用于辅助分析 Handler，插件注册了 4 个命令 (插件源码见附件 vm_plug)：  

![](https://bbs.pediy.com/upload/attach/202107/906381_QFGZDQ5FYZ4Q6Q9.jpg)

写脚本如下：

```
    vardel $handlerTable
    vardel $handlerCount
    vardel $handlerBaseAddress
    vardel $vmJmp
    vardel $tlsEnd
    var $handlerTable, VMP_UserDebugger.exe:0 + 19065C 
    var $handlerCount, 0x100
    var $handlerBaseAddress, VMP_UserDebugger.exe:0 - 54F80000
    var $vmJmp, VMP_UserDebugger.exe:0 + 18C8F2
    var $tlsEnd, ntdll.dll:0 + 19A1D
    clearlog
    vmtraceclear
    vminit $handlerTable
    bd
    bp $tlsEnd
    bp $vmJmp  
    SetBreakpointSilent $tlsEnd
    SetBreakpointSilent $vmJmp 
.begin:
    be $tlsEnd
    be $vmJmp
    g
    bd $vmJmp
    bd $tlsEnd
.loop:
    cmp cip, $tlsEnd
    jz .leave
    cmp cip, $vmJmp
    jz .trace
    log "Unexpected breakpoit: {p:rip}"
    jmp .leave   
.trace:
    vmtracestart "vm_{p:rcx}", 1
    cmp $result, 1
    jnz .begin
    ticnd "cip == $vmJmp || cip == $tlsEnd"
    vmtracestop
    jmp .loop
.leave:
    bd $tlsEnd
    bd $vmJmp
    ret

```

调试启动程序，中断在 Tls 回调函数起始处，执行上文脚本，各个 handler 的 trace 文件会以名字 vm_[handler 地址].trace64 保存至 X64DBG 所在文件夹下 (可调用 vmclear 删除)，跟踪文件见附件 trace.zip。

分析各个 Handler（需要一点耐心），根据实现定义伪操作码如下：

![](https://bbs.pediy.com/upload/attach/202107/906381_XGGDY26Q3TS9PY8.jpg)

看几个有代表性的 Handler:

![](https://bbs.pediy.com/upload/attach/202107/906381_Z2RFRFSGV9ATSDX.jpg)

![](https://bbs.pediy.com/upload/attach/202107/906381_UG6996FJFCNGZ4J.jpg)

从上面两个 Handler 可以判断出栈应是 2 字节对齐的。

![](https://bbs.pediy.com/upload/attach/202107/906381_7TEA9X5K4KEZU54.jpg)

可以发现，上一节分析过的 VmInitialize 是 VmCALL 的一部分，将其改名为 VmCallInitialize。

![](https://bbs.pediy.com/upload/attach/202107/906381_YDT9TGQRRKE3BTM.jpg)

![](https://bbs.pediy.com/upload/attach/202107/906381_UP5WRXB3UX6QDCG.jpg)

我们注意到会有多个 Handler 实现同一个功能。

重新调试执行程序，修改脚本，使用已分析的 Handler 翻译程序（vmdump）：

```
    vardel $handlerTable
    vardel $handlerCount
    vardel $handlerBaseAddress
    vardel $vmJmp
    vardel $tlsEnd
    var $handlerTable, VMP_UserDebugger.exe:0 + 19065C 
    var $handlerCount, 0x100
    var $handlerBaseAddress, VMP_UserDebugger.exe:0 - 54F80000
    var $vmJmp, VMP_UserDebugger.exe:0 + 18C8F2
    var $tlsEnd, ntdll.dll:0 + 19A1D
    clearlog
    vmtraceclear
    vminit $handlerTable
    bd
    bp $tlsEnd
    bp $vmJmp  
    SetBreakpointSilent $tlsEnd
    SetBreakpointSilent $vmJmp 
.begin:
    g
.loop:
    cmp cip, $tlsEnd
    jz .leave
    cmp cip, $vmJmp
    jz .dump
    log "Unexpected breakpoit: {p:rip}"
    jmp .leave   
.dump:
    vmdump rcx, rsi, rbx
    jmp .begin
.leave:
    bd $tlsEnd
    bd $vmJmp
    ret

```

得到伪代码如下（见附件 vm_tls.txt）：

```
[Anakin] VmPOP V_98
[Anakin] VmPUSH FFFFFFFF9F5A5C32
[Anakin] VmADD
[Anakin] VmPOP V_40
[Anakin] VmPOP V_B8
[Anakin] VmPOP V_28
[Anakin] VmPOP V_18
[Anakin] VmPOP V_00
[Anakin] VmPOP V_78
[Anakin] VmPOP V_A0
[Anakin] VmPOP V_90
[Anakin] VmPOP V_40
[Anakin] VmPOP V_20
[Anakin] VmPOP V_68
[Anakin] VmPOP V_50
[Anakin] VmPOP V_58
[Anakin] VmPOP V_30
[Anakin] VmPOP V_B0
[Anakin] VmPOP V_38
[Anakin] VmPOP V_48
[Anakin] VmPOP V_70
[Anakin] VmPOP V_88
[Anakin] VmPOP V_10
[Anakin] VmPOP V_A8
[Anakin] VmPUSH 0000000064765E24
[Anakin] VmPUSHB8 00
[Anakin] VmPUSH 000000014018B3E7
[Anakin] VmPUSH V_98
[Anakin] VmADD
[Anakin] VmPOP V_08
[Anakin] VmREADB
[Anakin] VmSBP
[Anakin] VmREADB
[Anakin] VmNOTANDB
[Anakin] VmPOP V_60
[Anakin] VmADDB
[Anakin] VmPOP V_10
...
...
...

```

3W 多行的汇编代码已然被翻译为 300 多行的伪代码，是一个较大的进步，后面我们需要进一步分析这些伪代码，进而把 Tls 回调的执行搞清楚。

另一方面，一个一个地进行 Handler 的手工分析，终归不能令人满意，这也是需要改善的一点。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 22 小时前 被 Anakin Stone 编辑 ，原因：

上传的附件：

*   [vm_plug.zip](javascript:void(0)) （466.79kb，5 次下载）
*   [vm_trace.zip](javascript:void(0)) （36.38kb，4 次下载）
*   [vm_tls.txt](javascript:void(0)) （7.67kb，4 次下载）