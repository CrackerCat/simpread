> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281789.htm)

> [分享] 不动内存不使用 VirtualProtect 的 AMSI 绕过原理（暂未修复）

[分享] 不动内存不使用 VirtualProtect 的 AMSI 绕过原理（暂未修复）

15 小时前 482

### [分享] 不动内存不使用 VirtualProtect 的 AMSI 绕过原理（暂未修复）

 [![](http://passport.kanxue.com/upload/avatar/851/856851.png?1588830148)](user-home-856851.htm) [菜鸟 m 号](user-home-856851.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 15 小时前  482

前言 & 背景
-------

Microsoft 的反恶意软件扫描接口 (AMSI) 在 Windows 10 和更高版本的 Windows 中提供，用于帮助检测和预防恶意软件。 AMSI 是一个接口，它将各种安全应用程序（例如防病毒或反恶意软件软件）集成到应用程序和软件中，并在执行之前检查它们的行为。 OffSec 工作人员 Victor “Vixx” Khoury 在 `System.Management.Automation.dll` 中发现了一个可写条目，其中包含 `AmsiScanBuffer` 的地址，`AmsiScanBuffer` 是 AMSI 的关键组件，应该被标记为只读，类似于导入地址表 (IAT) 条目。在这篇文章中，我们将介绍并利用此漏洞进行 0day AMSI 绕过。

大多数 AMSI 绕过都会破坏 AMSI 库 Amsi.dll 中的函数或字段，从而导致 AMSI 崩溃来绕过它。除了崩溃或 patch Amsi.dll 之外，攻击者还可以使用 CLR Hooking 绕过 AMSI，通过调用 VirtualProtect 并使用返回 TRUE 的 HOOK 覆盖它来更改 ScanContent 函数的保护功能。虽然 VirtualProtect 本身并不是恶意的，但恶意软件可能会滥用它来修改内存，从而逃避 EDR 和防病毒 (AV) 软件的检测。

分析过程
----

首先检查 Amsi.dll 的 AmsiScanBuffer 函数，该函数扫描内存缓冲区中是否存在恶意软件，许多应用程序和服务都利用此功能。在 .NET 框架内，_Common Language_ _Runtime_ (CLR) 利用 `System.Management.Automation.dll` 内 AmsiUtils 类中的 ScanContent 函数，该函数是 PowerShell 核心库的一部分，来调用 `AmsiScanBuffer` 。可以通过`[PSObject].Assembly.Location`查看该 DLL 位置，

![](https://bbs.kanxue.com/upload/attach/202405/856851_JUTQAYV9ZSPHD6V.png)

然后用 dnspy 找到 ScanContent 位置，找到调用 AmsiScanBuffer 调用位置：

![](https://bbs.kanxue.com/upload/attach/202405/856851_5W85DP527VAWTR9.png)

![](https://bbs.kanxue.com/upload/attach/202405/856851_B2SQP2JB7M7CA6A.png)

使用 windbg 附加 powershell.exe，查看 amis 模块，以 Amsi 关键词的符号（symbol):

![](https://bbs.kanxue.com/upload/attach/202405/856851_35632VF53ZBFSAB.png)

在 AmsiScanBuffer 上设置一个断点：

![](https://bbs.kanxue.com/upload/attach/202405/856851_5P3X83KFH4UXXYX.png)

随机读取一个文件或者键入字符串，触发断点，查看调用堆栈如下：

![](https://bbs.kanxue.com/upload/attach/202405/856851_Y85P9GXZ33M6ZQ6.png)

![](https://bbs.kanxue.com/upload/attach/202405/856851_8UJYBESVVMGMK4X.png)

本次攻击的目标是 _System Management Automation ni_ 模块，它调用的函数，而不是直接去操作 AmsiScanBuffer 函数。

从 System_Management_Automation_ni+0x1099e37 处往后反汇编查看代码，每个人地址可能不一样：

```
0:011> ub System_Management_Automation_ni+0x1099e37 L10
System_Management_Automation_ni+0x1099df1:
00007ffa`109c9df1 488975c0        mov     qword ptr [rbp-40h],rsi
00007ffa`109c9df5 4c8d4e0c        lea     r9,[rsi+0Ch]
00007ffa`109c9df9 4c897db8        mov     qword ptr [rbp-48h],r15
00007ffa`109c9dfd 4c8b5db0        mov     r11,qword ptr [rbp-50h]
00007ffa`109c9e01 4d8b5b20        mov     r11,qword ptr [r11+20h]
00007ffa`109c9e05 498b03          mov     rax,qword ptr [r11]
00007ffa`109c9e08 488b7530        mov     rsi,qword ptr [rbp+30h]
00007ffa`109c9e0c 4889742420      mov     qword ptr [rsp+20h],rsi
00007ffa`109c9e11 4c897c2428      mov     qword ptr [rsp+28h],r15
00007ffa`109c9e16 41bb10000000    mov     r11d,10h
00007ffa`109c9e1c 4c8b55b0        mov     r10,qword ptr [rbp-50h]
00007ffa`109c9e20 4c895588        mov     qword ptr [rbp-78h],r10
00007ffa`109c9e24 4c8d150c000000  lea     r10,[System_Management_Automation_ni+0x1099e37 (00007ffa`109c9e37)]
00007ffa`109c9e2b 4c8955a0        mov     qword ptr [rbp-60h],r10
00007ffa`109c9e2f 41c644240c00    mov     byte ptr [r12+0Ch],0
00007ffa`109c9e35 ffd0            call    rax

```

`00007ffa 109c9e35 ffd0 call rax`就是对 AmsiScanBuffer 的调用位置，绕过 AMSI 的一种方法是 patch call rax，这需要 VirtualProtect. 但是在调用函数之前，研究人员发现 AmsiScanbuffer 函数已经是可写状态了，注意看：

![](https://bbs.kanxue.com/upload/attach/202405/856851_DBYQ9KKTDDPWAGQ.png)

![](https://bbs.kanxue.com/upload/attach/202405/856851_YPZWPEXFYR85KPK.png)

`dqs @rbp-0x50 L1`查看基址指针寄存器 rbp（当前堆栈帧的基址）之前 80 字节 (0x50) 地址。一步步找到 AmsiScanBuffer 地址 0x7ffa10b5d9b0，相对 System_Management_Automation_ni 模块偏移 0x78d9b0。

### 分析调用

现在我们深入研究一下这块是怎么填充和保护的，现在用 windbg launch powershell.exe。在 System.Management.Automation.ni.dll 设置触发断点，执行 g，然后再在 _System Management Automation ni +_ _0x78d9b0_ 处中断读 / 写`ba r1 System_Management_Automation_ni + 0x78d9b0`，执行 g

![](https://bbs.kanxue.com/upload/attach/202405/856851_74P9YW4KZ48YKA9.png)

clr!NDirectMethodDesc::SetNDirectTarget+0x3b 处被断下，反汇编查看附件汇编代码：

![](https://bbs.kanxue.com/upload/attach/202405/856851_2K4ZUH95DP4NTJP.png)

我们发现 rdi 已经是 AmsiScanBuffer 函数地址，rsi 正在被写入目标地址。

继续 g 执行，就会发现前文提到的调用位置：

![](https://bbs.kanxue.com/upload/attach/202405/856851_SY9EFQUKGMSSUPT.png)

所以 PowerShell 刚开始执行就会初始化 AmsiScanBuffer 函数地址，然后后面会直接调用一次。在初始化时候的断点，查看调用堆栈，会发现初始调用块也在 clr!ThePreStub 中：

![](https://bbs.kanxue.com/upload/attach/202405/856851_VQEPMP2KFXHM8KH.png)

它是 .NET Framework 中的一个辅助函数，用于为初始执行准备代码，其中包括即时 (JIT) 编译。它会创建一个位于被调用方和原始调用方函数之间的存根。

所以，作为 JIT 的一部分，辅助函数将 AmsiScanBuffer 地址写入 DLL 入口地址中偏移量 0x78d9b0 处，但不会将权限更改回只读。我们可以通过覆盖该函数地址来绕过 AMSI 而不调用 VirtualProtect 来滥用此漏洞。

### PoC-ps1

计算机和安装的 CLR 版本不一样会导致上述的 0x78d9b0 偏移量不一样，poc 参考 [https://github.com/V-i-x-x/AMSI-BYPASS/。](https://github.com/V-i-x-x/AMSI-BYPASS/%E3%80%82)

  

[[培训] 二进制漏洞攻防（第 3 期）；满 10 人开班；模糊测试与工具使用二次开发；网络协议漏洞挖掘；Linux 内核漏洞挖掘与利用；AOSP 漏洞挖掘与利用；代码审计。](https://www.kanxue.com/book-section_list-174.htm)

[#.NET 平台](forum-4-1-7.htm) [#调试逆向](forum-4-1-1.htm)