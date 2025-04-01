> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286278.htm)

> [原创]VMP 入门: VMP1.81 Demo 分析

一、前言
====

本文主要是通过分析 VMProtect 1.81 Demo 版加密后的程序，讲述 VMP 的基本结构，并尝试分析虚拟代码的功能以及虚拟代码还原。

二、虚拟机结构
=======

虚拟机保护技术是将程序的可执行代码转化为自定义的中间操作码（一字节或多字节），用以保护源程序不被逆向和篡改，这些中间操作码通过虚拟机解释执行，实现程序原来的功能。在这种情况下，如果要逆向程序，就需要对整个虚拟机结构进行逆向，理解程序功能，还需要结合操作码进行分析，整个程序逆向工程将会十分繁琐。以下是一个一般虚拟机结构：

![](https://bbs.kanxue.com/upload/attach/202503/985561_4H9SAKRQG2VP2UV.webp)

虚拟机解释执行流程如下：

1.  将真实环境的上下文入栈，初始化虚拟机运行所需要的环境，例如初始化 VIP、VSP 等。
2.  根据 VIP 获取下一条虚拟指令的操作码，根据虚拟指令的操作码，跳转到对应 VM Handler。
3.  执行 VM Handler，更新虚拟指令指针。
4.  虚拟堆栈检查，检查虚拟栈空间是否充足。
5.  重复执行 2~4 直至执行完所有虚拟指令字节码。
6.  还原真实环境的上下文，退出虚拟机。

与虚拟机相关的数据结构如下：

*   **VIP**：虚拟指令指针，相当于 x86-64 中的 RIP 寄存器，指向下一条将要被执行的虚拟指令的地址。一般使用 RSI 寄存器来保存 VIP。
*   **VSP**：虚拟堆栈指针，相当于 x86-64 中的 RSP 寄存器，用来保存虚拟堆栈栈顶指针。一般使用 RBP 寄存器来保存 VSP。
*   **VM Stack**：虚拟堆栈，基于栈的虚拟机的一切指令的执行都借助虚拟栈来完成，例如加法指令，需要事先将两个操作数入栈，然后在栈上进行运算。
*   **VM Registers/VM Context**：虚拟寄存器组，或者称之为虚拟机的上下文。一般使用 RDI 寄存器来指向虚拟寄存器组所在地址。
*   **VM Instruction**：虚拟指令，也称为虚拟字节码，是虚拟机解释并随后执行的字节。每条虚拟指令至少由一个或多个字节。
*   **VM Opcode**：虚拟指令操作码，每条虚拟指令中的第一个字节。
*   **IMM**：虚拟指令中的立即数，这些立即数可能充当常量值、虚拟寄存器的索引值等。
*   **VM Handler**：用于执行真实指令的虚拟代码片段，例如一个 ADD 指令对应一个 VM Handler。根据 VM Opcode 选取对应的 VM Handler 来解释执行该操作码的具体功能。
*   **VM Handler Table**：存放所有 VM Handler 的偏移地址的表。
*   **VM Opcode Table**：存放虚拟指令字节码的表。

三、案例分析
======

本案例的样本来自于[如何分析虚拟机系列 (1)：新手篇 VMProtect 1.81 Demo - 吾爱破解 - 52pojie.cn](elink@b2bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3f1#2x3Y4m8G2K9X3W2W2i4K6u0W2j5$3&6Q4x3V1k6@1K9s2u0W2j5h3c8Q4x3X3b7%4x3e0x3J5x3e0W2Q4x3X3b7I4i4K6u0V1x3g2)9J5k6h3S2@1L8h3H3`.)。样本的汇编代码如下，就是一个简单的加减同一个常量。

```
.text:00401000 sub_401000      proc near               ; CODE XREF: start↓p
.text:00401000                 mov     eax, dword_403000; 值为 0deadbeefh
.text:00401005                 add     eax, 12345678h
.text:0040100A                 sub     eax, 12345678h
.text:0040100F                 mov     dword_403000, eax
.text:00401014                 retn
.text:00401014 sub_401000      endp
```

经过 VMP 保护后，变成如下所示

```
.text:00401000 ; void sub_401000()
.text:00401000 sub_401000      proc near               ; CODE XREF: start↓p
.text:00401000                 push    offset byte_404781
.text:00401005                 call    sub_40472C
.text:0040100A
.text:0040100A loc_40100A:                             ; CODE XREF: sub_40472C+2B↓j
.text:0040100A                                         ; DATA XREF: .vmp0:jpt_404757↓o
.text:0040100A                 mov     esi, [ebp+0]    ; jumptable 00404757 cases 126,128,146,149,246,255
.text:0040100D                 add     ebp, 4
.text:00401010                 jmp     loc_40474E
.text:00401010 sub_401000      end
```

这里有个典型的 VMP 入口特征：push call。push 压入栈的数据是虚拟指令操作码表（VM Opcode Table）所在地址，然后通过 call sub_40472C 进入 VM Entry。byte_404781 如下

```
.vmp0:00404781 byte_404781     db 0Eh, 0E8h, 81h       ; DATA XREF: sub_401000↑o
.vmp0:00404784                 dd 5C765DA9h, 3E0A1A1Eh, 3A36262Eh, 602122Ah, 3000E822h
.vmp0:00404798                 dd 16790040h, 34567869h, 1E7A1412h, 5678E836h, 97341234h
.vmp0:004047AC                 dd 5C066B4Dh, 1121973Eh, 4C3C0622h, 0EB161121h, 1E11F7EAh
.vmp0:004047C0                 dd 326B2020h, 78081577h, 36325C32h, 30006904h, 0ED0040h
.vmp0:004047D4                 dd 4382010h, 8342C24h, 0E1001Ch, 8 dup(0)
.vmp0:00404800                 dd 200h dup(?)
```

VM Entry 分析
-----------

```
.vmp0:0040472C                 push    esi
.vmp0:0040472D                 push    edi
.vmp0:0040472E                 push    esp
.vmp0:0040472F                 push    ebx
.vmp0:00404730                 push    eax
.vmp0:00404731                 push    edx
.vmp0:00404732                 push    ebp
.vmp0:00404733                 pushf
.vmp0:00404734                 push    ecx
```

首先一个很典型的特征就是 VMP 在创建虚拟机环境时，会将真实环境中寄存器的值入栈保存。

```
.vmp0:00404735                 push    ds:dword_404649
.vmp0:0040473B                 push    0
```

以上两条指令把要被操作的数据压入栈，这是基于栈的虚拟机的一个特点：数据的操作需要在栈上进行。那么可以确定这里应该是虚拟栈了。

```
.vmp0:00404740                 mov     esi, [esp+2Ch+arg_0] ;arg_0 = dword ptr  4
```

初始化 esi，通过画堆栈图分析，可知 esi 的值为 byte_404781，也就是虚拟指令操作码表（VM Opcode Table）的基址。

```
.vmp0:00404744                 mov     ebp, esp
.vmp0:00404746                 sub     esp, 0C0h
.vmp0:0040474C                 mov     edi, esp
```

初始化 ebp 的值为 esp，指向虚拟栈的栈顶，然后创建一个新的栈帧，用于当作虚拟寄存器使用，edi 指向虚拟寄存器的基址。

```
.vmp0:0040474E loc_40474E:                            
.vmp0:0040474E                 add     esi, [ebp+0]     ;index of vm opcode
```

之后是获取虚拟栈栈顶数据，即虚拟指令操作码的偏移地址，然后加上虚拟操作码表的基址，此时 esi 指向第一个将要被执行的虚拟指令操作码。

VM Dispatcher 分析
----------------

```
mov     al, [esi]       ;get vm opcode
movzx   eax, al
inc     esi             ;index + 1
jmp     ds:jpt_404757[eax*4] ; jmp to vm handler
```

根据虚拟指令操作码跳转到对应的 VM Handler 来执行相应功能。jpt_404757 是一个函数地址表，存放的是 VM Handlers 的偏移。

![](https://bbs.kanxue.com/upload/attach/202503/985561_4FE58HV8ME5WRGP.webp)

VM Handler 分析
-------------

### ADD——加法

#### ADDB——两个单字节数相加

```
loc_40476F:
mov     al, [ebp+0]
sub     ebp, 2
add     [ebp+4], al
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

虚拟栈栈顶的两个单字节数相加，结果存入后者所在位置，标志寄存器入真实栈，然后将真实栈顶元素（标志寄存器的值）存入到虚拟栈顶处。

> 参与加法运算的是 [源 ebp+2] 、[源 ebp] 的两个字节相加，而不是 [源 ebp+1] 、[源 ebp] 的两个字节相加，可能的原因是虚拟栈按两字节对齐。

#### ADDW——两个双字节数相加

```
loc_4045FC:
mov     ax, [ebp+0]
sub     ebp, 2
add     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

虚拟栈栈顶两个双字节数相加。此时虚拟栈提升两字节。虚拟栈顶的前四字节为运算产生的标志位，后两字节为运算产生的值。

#### ADDDW——两个四字节数相加

```
loc_404000:
mov     eax, [ebp+0]
add     [ebp+4], eax
pushf
pop     dword ptr [ebp+0]
jmp     loc_404751
```

虚拟栈栈顶两个四字节数相加。此时虚拟栈顶的位置不变。

### SHR——逻辑右移

#### SHRB——单字节数逻辑右移

```
loc_404680:
mov     al, [ebp+0]
mov     cl, [ebp+2]
sub     ebp, 2
shr     al, cl
mov     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈栈顶取出两个单字节数据，进行逻辑右移运算。将结果零扩展至双字节（右移结果存储在 al 中，但入栈的却是 ax）。此时虚拟栈提升两字节。

#### SHRW——双字节数逻辑右移

```
loc_40454E:
mov     ax, [ebp+0]
mov     cl, [ebp+2]
sub     ebp, 2
shr     ax, cl
mov     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈中依次取出双字节数据、单字节数据（作为位移值），进行逻辑右移运算。此时虚拟栈提升两字节。

#### SHRDW——四字节数逻辑右移

```
loc_404077:
mov     eax, [ebp+0]
mov     cl, [ebp+4]
sub     ebp, 2
shr     eax, cl
mov     [ebp+4], eax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈栈顶依次取四字节数据、一字节数据（作为位移值），进行逻辑右移运算。此时虚拟栈提升两字节。

### SHRD——双精度逻辑右移

#### SHRDDW——四字节双精度逻辑右移

```
loc_404610:
mov     eax, [ebp+0]
mov     edx, [ebp+4]
mov     cl, [ebp+8]
add     ebp, 2
shrd    eax, edx, cl
mov     [ebp+4], eax
pushf
pop     dword ptr [ebp+0]
jmp     loc_404751
```

从虚拟栈栈顶依次取两个四字节数据、一个一字节数据，进行双精度右移，这里仅将高位 eax 放入栈中。此时虚拟栈缩小两字节。

### SHL——逻辑左移

#### SHLB——单字节数逻辑左移

```
loc_4045C0:
mov     al, [ebp+0]
mov     cl, [ebp+2]
sub     ebp, 2
shl     al, cl
mov     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈中依次取出两个单字节数据，进行逻辑左移运算。将结果零扩展为双字节（结果存储在 al，但入栈的却是 ax）。此时虚拟栈提升两字节。

#### SHLW——双字节数逻辑左移

```
loc_404698:
mov     ax, [ebp+0]
mov     cl, [ebp+2]
sub     ebp, 2
shl     ax, cl
mov     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈的栈顶中依次取出双字节数据、单字节数据（位移值），进行逻辑左移。此时虚拟栈提升两字节。

#### SHLDW——四字节数逻辑左移

```
loc_4046EF:
mov     eax, [ebp+0]
mov     cl, [ebp+4]
sub     ebp, 2
shl     eax, cl
mov     [ebp+4], eax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈的栈顶中依次取出四字节数据、单字节数据（位移值），进行逻辑左移。此时虚拟栈提升两字节。

### SHLD——双精度逻辑左移

#### SHLDDW——四字节双精度逻辑左移

```
loc_4046BF:
mov     eax, [ebp+0]
mov     edx, [ebp+4]
mov     cl, [ebp+8]
add     ebp, 2
shld    eax, edx, cl
mov     [ebp+4], eax
pushf
pop     dword ptr [ebp+0]
jmp     loc_404751
```

从虚拟栈栈顶依次取两个四字节数据、一个一字节数据，进行双精度左移。此时虚拟栈缩小两字节。

### NAnd——先非后与

#### NAndB——两个单字节数据先非后与

```
loc_404591:
mov     ax, [ebp+0]
mov     dx, [ebp+2]
not     al
not     dl
sub     ebp, 2
and     al, dl
mov     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

从虚拟栈中依次取出两个双字节数据，实际上是低位一字节先进行非运算，然后再进行与运算。此时虚拟栈提升两字节。

#### NAndW——两个双字节数据先非后与

```
loc_404041:
not     dword ptr [ebp+0]
mov     ax, [ebp+0]
sub     ebp, 2
and     [ebp+4], ax
pushf
pop     dword ptr [ebp+0]
jmp     loc_40400F
```

对虚拟栈栈顶的四字节取反，然后分成两个双字节数进行与运算。此时虚拟栈提升两字节。

#### NAndDW——两个四字节数据先非后与

```
loc_40451A:
mov     eax, [ebp+0]
mov     edx, [ebp+4]
not     eax
not     edx
and     eax, edx
mov     [ebp+4], eax
pushf
pop     dword ptr [ebp+0]
jmp     loc_404751
```

从虚拟栈栈顶依次取两个四字节数据，先进行非运算，然后与运算。此时虚拟栈顶位置不变。

### PushConst——虚拟指令中的立即数入栈

#### PushConstB——单字节立即数入栈

```
loc_4044E4:
movzx   eax, byte ptr [esi]
sub     ebp, 2
mov     [ebp+0], ax
add     esi, 1
jmp     loc_40400F
```

取虚拟指令中的操作数（单字节），零扩展至单字，然后入栈。此时虚拟栈提升两字节。

#### PushConstW——双字节立即数入栈

```
loc_40464D:
mov     ax, [esi]
lea     esi, [esi+2]
sub     ebp, 2
mov     [ebp+0], ax
jmp     loc_40400F
```

取虚拟指令中操作数（两字节），将其入栈。此时虚拟栈提升两字节。

#### PushConstDW——四字节数入栈

```
loc_40462B:
mov     eax, [esi]
sub     ebp, 4
lea     esi, [esi+4]
mov     [ebp+0], eax
jmp     loc_40400F
```

取虚拟指令中操作数（四字节），将其入栈。此时虚拟栈提升四字节。

#### PushConstCBDE——单字节立即数符号扩展至四字节并入栈

```
loc_40453B:
mov     al, [esi]
cbw
cwde
sub     ebp, 4
lea     esi, [esi+1]
mov     [ebp+0], eax
jmp     loc_40400F
```

取虚拟指令中操作数（单字节），符号扩展成双字，然后入栈。此时虚拟栈提升四字节。

#### PushConstCWDE——双字节立即数符号扩展至四字节并入栈

```
loc_4044BE:
mov     ax, [esi]
cwde
add     esi, 2
sub     ebp, 4
mov     [ebp+0], eax
jmp     loc_40400F
```

取虚拟指令中的操作数（两字节），符号扩展至双字，然后入栈。此时虚拟栈提升四字节。

### PushReg——将指定虚拟寄存器的值入栈

#### PushRegB——取指定虚拟寄存器的单字节数据入栈

```
loc_4044D0:
mov     al, [esi]
mov     al, [edi+eax]
sub     ebp, 2
sub     esi, 0FFFFFFFFh
mov     [ebp+0], ax
jmp     loc_40400F
```

从虚拟指令中取出操作数（单字节），作为虚拟寄存器的索引值，然后取出对应的虚拟寄存器中的值（单字节），将其入栈。此时虚拟栈提升两字节。

> sub esi, 0FFFFFFFFh 等价于 add esi, 1

#### PushRegW——取指定虚拟寄存器的双字节数据入栈

```
loc_4045E9:
mov     al, [esi]
inc     esi
mov     ax, [edi+eax]
sub     ebp, 2
mov     [ebp+0], ax
jmp     loc_40400F
```

从虚拟指令中取出操作数（单字节），作为虚拟寄存器的索引值，取出对应虚拟寄存器的数据（双字节），将其入栈。此时虚拟栈提升两字节。

#### PushRegDW——取指定虚拟寄存器的四字节数据入栈

```
loc_4045AF:
and     al, 3Ch
mov     edx, [edi+eax]
sub     ebp, 4
mov     [ebp+0], edx
jmp     loc_40400F
```

虚拟指令的操作码（单字节）与常量 0x3C 进行与运算，作为虚拟寄存器的索引值，取出对应虚拟寄存器中的值（四字节），将其入栈。此时虚拟栈提升四字节。

### PopReg——将栈顶数据加载到指定虚拟寄存器中

#### PopRegB——加载栈顶一字节数据到指定虚拟寄存器中

```
loc_404568:
mov     al, [esi]
mov     dx, [ebp+0]
sub     esi, 0FFFFFFFFh
add     ebp, 2
mov     [edi+eax], dl
jmp     loc_404751
```

从虚拟指令中取出一字节操作数作为虚拟寄存器索引，取虚拟栈栈顶的两字节数据，将其存储到对应虚拟寄存器中，实际存储低位（一字节）。此时虚拟栈缩小两字节。

#### PopRegW——加载栈顶两字节数据到指定虚拟寄存器中

```
loc_4046DA:
mov     al, [esi]
add     esi, 1
mov     dx, [ebp+0]
add     ebp, 2
mov     [edi+eax], dx
jmp     loc_404751
```

从虚拟指令中取出一字节操作数作为虚拟寄存器索引，然后取虚拟栈栈顶的两字节数据，将其存入对应虚拟寄存器中。此时虚拟栈缩小两字节。

#### PopRegDW——加载栈顶四字节数据到指定虚拟寄存器中

```
loc_404058:
and     al, 3Ch
mov     edx, [ebp+0]
add     ebp, 4
mov     [edi+eax], edx
jmp     loc_404751
```

虚拟指令的操作码（一字节）与 0x3C 进行与运算，取出虚拟栈栈顶的四字节数据，将其存放到指定的虚拟寄存器中。此时虚拟栈缩小四字节。

### PushVSP——虚拟栈指针入栈

#### PushVSPW——虚拟栈指针低位两字节入栈

```
loc_40463B:
mov     eax, ebp
sub     ebp, 2
mov     [ebp+0], ax
jmp     loc_40400F
```

虚拟栈的栈顶指针入栈（低位两字节）。此时虚拟栈提升两字节。

#### PushVSPDW——虚拟栈指针四字节入栈

```
loc_4046B2:
mov     eax, ebp
sub     ebp, 4
mov     [ebp+0], eax
jmp     loc_40400F
```

虚拟栈的栈顶指针入栈（四字节）。此时虚拟栈提升四字节。

### PopVSP——将栈顶数据加载到虚拟栈指针寄存器中

#### PopVSPW——加载栈顶两字节数据到虚拟栈指针寄存器中

```
loc_404532:
mov     bp, [ebp+0]
jmp     loc_40400F
```

虚拟栈栈顶的两字节数据加载到虚拟栈指针寄存器中。此时虚拟栈顶变化未知。

#### PopVSPDW——加载栈顶四字节数据到虚拟栈指针寄存器中

```
loc_40457C:
mov     ebp, [ebp+0]
jmp     loc_40400F
```

虚拟栈栈顶的四字节数据加载到虚拟栈指针寄存器中。此时虚拟栈顶变化未知。

### Store——写内存

#### StoreB——单字节数据写入指定内存中

```
loc_4044AE:
mov     eax, [ebp+0]
mov     dl, [ebp+4]
add     ebp, 6
mov     [eax], dl
jmp     loc_404751
```

从虚拟栈中依次取四字节数据（内存地址）、一字节数据（待存储的值），将一字节数据存储到四字节数据所指向的内存中。此时虚拟栈缩小六字节。

另一种使用堆栈段寄存器 ss 的情况：

```
loc_40475E:
mov     eax, [ebp+0]
mov     dl, [ebp+4]
add     ebp, 6
mov     ss:[eax], dl
jmp     loc_404751
```

#### StoreW——双字节数据写入指定内存中

```
loc_4044F6:
mov     eax, [ebp+0]
mov     dx, [ebp+4]
add     ebp, 6
mov     [eax], dx
jmp     loc_404751
```

从虚拟栈中依次取四字节数据（内存地址）、两字节数据（待存储的值），将两字节数据存储到四字节数据所指向的内存中。此时虚拟栈缩小六字节。

另一种使用堆栈段寄存器 ss 的情况：

```
loc_404706:
mov     eax, [ebp+0]
mov     dx, [ebp+4]
add     ebp, 6
mov     ss:[eax], dx
jmp     loc_404751
```

#### StoreDW——四字节数据写入指定内存中

```
loc_404670:
mov     eax, [ebp+0]
mov     edx, [ebp+4]
add     ebp, 8
mov     [eax], edx
jmp     loc_404751
```

从虚拟栈栈顶依次取两个四字节数据，前者作为内存地址，后者作为待存储的值，然后将值存储到对应内存中。此时虚拟栈缩小八字节。

另一种使用堆栈段寄存器 ss 的情况：

```
loc_4045D8:
mov     eax, [ebp+0]
mov     edx, [ebp+4]
add     ebp, 8
mov     ss:[eax], edx
jmp     loc_404751
```

### Load——读内存

#### LoadB——加载指定内存的单字节数据到虚拟堆栈中

```
loc_40465F:
mov     edx, [ebp+0]
add     ebp, 2
mov     al, [edx]
mov     [ebp+0], ax
jmp     loc_404751
```

从虚拟栈顶中取出四字节数据作为内存地址，从对应地址中取出单字节数据，零扩展至单字后将其入栈。此时虚拟栈缩小两字节。

另一种使用堆栈段寄存器 ss 的情况：

```
loc_40449C:
mov     edx, [ebp+0]
add     ebp, 2
mov     al, ss:[edx]
mov     [ebp+0], ax
jmp     loc_404751
```

#### LoadW——加载指定内存的双字节数据到虚拟堆栈中

```
loc_404508:
mov     eax, [ebp+0]
add     ebp, 2
mov     ax, [eax]
mov     [ebp+0], ax
jmp     loc_404751
```

从虚拟栈中依次取四字节数据作为内存地址，取出对应内存的两字节数据，将其压入栈。此时虚拟栈缩小两字节。

另一种使用堆栈段寄存器 ss 的情况：

```
loc_404719:
mov     eax, [ebp+0]
add     ebp, 2
mov     ax, ss:[eax]
mov     [ebp+0], ax
jmp     loc_404751
```

#### LoadDW——加载指定内存的四字节数据到虚拟堆栈中

```
loc_404584:
mov     eax, [ebp+0]
mov     eax, [eax]
mov     [ebp+0], eax
jmp     loc_404751
```

取虚拟栈栈顶的四字节数据作为内存地址，然后取对应内存地址的四字节数据，将其压入栈。此时虚拟栈顶位置不变。

另一种使用堆栈段寄存器 ss 的情况：

```
loc_404069:
mov     eax, [ebp+0]
mov     eax, ss:[eax]
mov     [ebp+0], eax
jmp     loc_404751
```

VM Exit 分析
----------

```
loc_40408E:
mov     esp, ebp
pop     edx     ;被覆盖
pop     ebx     ;被覆盖
pop     ecx
popf
pop     ebp
pop     edx
pop     eax
pop     ebx
pop     esi
pop     edi
pop     esi
retn
```

按照 vm_entry 入栈顺序进行逆序出栈，还原本机环境。

VM Check 分析
-----------

```
//栈空间是否充足
loc_40400F:
lea     eax, [edi+50h]
cmp     ebp, eax
ja      loc_404751
// 栈空间调整
mov     edx, esp
lea     ecx, [edi+40h]
sub     ecx, edx
lea     eax, [ebp-80h]
and     al, 0FCh
sub     eax, ecx
mov     esp, eax
pushf
push    esi
mov     esi, edx
lea     edi, [eax+ecx-40h]
push    edi
mov     edi, eax
cld
rep movsb
pop     edi
pop     esi
popf
jmp     loc_404751
```

先检查栈空间是否充足，如果充足的话，跳转到 vm_dispatcher，否则进行栈空间调整然后再跳转到 vm_dispatcher。

虚拟代码还原
------

### 虚拟代码还原成一级虚拟指令

这里借助 OllyDBG 的 Trace 功能，跟踪程序执行的代码流，使用方法如下：

1.  在菜单栏中点击 E 按钮（或者快捷键 Alt+E）打开可执行模块窗口，右键主模块，选择” 添加模块到运行跟踪协议 “（ Limit run trace protocol to selected module），表示 Trace 时记录该模块的所有地址，其他模块（如系统模块）不进行记录。
2.  在菜单栏中点击选项 -> 选项按钮（或者快捷键 Alt+o），选择运行跟踪页面（Run Trace），勾选记住内存（Remember memory）。
3.  在菜单栏中点击跟踪 -> 跟进 (Trace -> Trace into ，或者快捷键 Ctrl + F11)，开始记录程序运行时的代码流。
4.  在菜单栏中点击 ... 按钮，打开运行跟踪窗口，右键日子记录文件（Log to file）， 在保存时勾选添加可用目录（Add available contents） 和独立列标签（Separate columns with tabs）。

通过 VM Handler 的分析，已经知晓了这些指令类型，通过地址与一级虚拟指令的映射关系，可以初步根据代码流将虚拟代码还原成一级虚拟指令代码流，但是这些一级虚拟指令在不同环境下，操作数是不同的，因此还需要根据 OD Trace 到的寄存器和内存再做进一步优化。我这里偷点懒，使用的方法仅针对该样本，并不是一个通用的方法：找到一级虚拟指令地址后，选取该 Handler 中合适的指令，从记录到的上下文环境中提取操作数，手动执行运算得到结果。

具体代码如下：

```
import re
from typing import Dict
 
class insnContext:
    insn: str
    addr: int
    memory: Dict[str, str]
    regs: Dict[str, str]
 
    def __init__(self, insn: str, addr: int, memory: Dict[str, str], regs: Dict[str, str]):
        self.insn = insn
        self.addr = addr
        self.memory = memory
        self.regs = regs
 
    def getRegValue(self, reg: str) -> str:
        return self.regs.get(reg)
 
    def getMemValue(self) -> str:
        key, value = next(iter(self.memory.items()))
        return value
 
def parseInsnStream(insn_stream: bytes) -> insnContext:
    insn_line = insn_stream.split(b'\t')
    addr = int(insn_line[1].decode(), 16)
    insn = insn_line[2].decode()
    # 使用正则表达式提取内存地址和对应的值
    memory = dict(re.findall(r'\[([a-fA-F0-9]{8})]=([a-fA-F0-9]{8})', insn_line[3].decode()))
    # 使用正则表达式提取寄存器和对应的值
    regs = dict(re.findall(r'([a-zA-Z]{3})=([a-fA-F0-9]{8})', insn_line[4].decode()))
    context = insnContext(insn, addr, memory, regs)
    return context
 
vm_handlers = {
    0x40476F: "ADDB",
    0x4045FC: "ADDW",
    0x404000: "ADDDW",
    0x404680: "SHRB",
    0x40454E: "SHRW",
    0x404077: "SHRDW",
    0x404610: "SHRDDW",
    0X4045C0: "SHLB",
    0x404698: "SHLW",
    0x4046EF: "SHLDW",
    0x4046BF: "SHLDDW",
    0x404591: "NAndB",
    0x404041: "NAndW",
    0x40451A: "NAndDW",
    0x4044E4: "PushConstB",
    0x40464D: "PushConstW",
    0x40462B: "PushConstDW",
    0x40453B: "PushConstCBDE",
    0x4044BE: "PushConstCWDE",
    0x4044D0: "PushRegB",
    0x4045E9: "PushRegW",
    0x4045AF: "PushRegDW",
    0x404568: "PopRegB",
    0x4046DA: "PopRegW",
    0x404058: "PopRegDW",
    0x40463B: "PushVSPW",
    0x4046B2: "PushVSPDW",
    0x404532: "PopVSPW",
    0x40457C: "PopVSPDW",
    0x4044AE: "StoreB",
    0x40475E: "StoreB",
    0x4044F6: "StoreW",
    0x404706: "StoreW",
    0x404670: "StoreDW",
    0x4045D8: "StoreDW",
    0x40465F: "LoadB",
    0x40449C: "LoadB",
    0x404508: "LoadW",
    0x404719: "LoadW",
    0x404584: "LoadDW",
    0x404069: "LoadDW"
}
handler_keys = vm_handlers.keys()
 
with open(r".\trace.txt", "rb") as f:
    insn_stream = f.readlines()[1:-2]
 
step = 0
end = len(insn_stream)
while step < end:
    ctx = parseInsnStream(insn_stream[step])
    step += 1
    if ctx.addr in handler_keys:
        vm_insn_info = ''
        mnemonic = vm_handlers.get(ctx.addr)
        if mnemonic == "PushRegDW":
            ctx = parseInsnStream(insn_stream[step + 2])
            reg_value = ctx.getRegValue('EDX')
            reg_index = int(ctx.getRegValue('EAX'), 16) / 4 # 32位系统
            vm_insn_info = "PushRegDW R%d;      R%d = 0x%s" % (reg_index, reg_index, reg_value)
        elif mnemonic == "PopRegDW":
            ctx = parseInsnStream(insn_stream[step + 2])
            reg_value = ctx.getRegValue('EDX')
            reg_index = int(ctx.getRegValue('EAX'), 16) / 4
            vm_insn_info = "PopRegDW R%d;       [ebp] = 0x%s" % (reg_index, reg_value)
        elif mnemonic == "PushConstDW":
            ctx = parseInsnStream(insn_stream[step + 2])
            imm = ctx.getRegValue("EAX")
            vm_insn_info = "PushConstDW 0x%s;" % imm
        elif mnemonic == "PushConstCWDE":
            ctx = parseInsnStream(insn_stream[step + 3])
            imm = ctx.getRegValue("EAX")
            vm_insn_info = "PushConstCWDE 0x%s;" % imm
        elif mnemonic == "ADDDW":
            ctx = parseInsnStream(insn_stream[step])
            ebp_0 = ctx.getRegValue("EAX")
            ebp_4 = ctx.getMemValue()
            result = hex(int(ebp_4, 16) + int(ebp_0, 16))[2:].zfill(8).upper()
            vm_insn_info = "ADDDW [ebp+4], [ebp+0];     [ebp+4] = 0x%s, [ebp] = 0x%s, [ebp+4]+[ebp] = 0x%s" % (ebp_4, ebp_0, result)
        elif mnemonic == "LoadDW":
            ctx = parseInsnStream(insn_stream[step])
            mem_addr = ctx.getRegValue("EAX")
            mem_value = ctx.getMemValue()
            vm_insn_info = "LoadDW [[ebp]];     [ebp] = 0x%s, [[ebp]] = 0x%s" % (mem_addr, mem_value)
        elif mnemonic == "StoreDW":
            ctx = parseInsnStream(insn_stream[step + 2])
            value = ctx.getRegValue("EDX")
            mem_addr = ctx.getRegValue("EAX")
            vm_insn_info = "StoreDW [[ebp+0]], [ebp+4];     [ebp] = 0x%s, [ebp+4] = 0x%s" % (mem_addr, value)
        elif mnemonic == "NAndDW":
            ctx = parseInsnStream(insn_stream[step])
            ebp_0 = ctx.getRegValue("EAX")
            ebp_4 = ctx.getMemValue()
            result = hex(((~int(ebp_4, 16)) & (~int(ebp_0, 16)) & 0xffffffff))[2:].zfill(8).upper()
            vm_insn_info = "NAndDW [ebp+4], [ebp+0];     [ebp] = 0x%s, [ebp+4] = 0x%s, ~[ebp+4] & ~[ebp]=0x%s" % (ebp_0, ebp_4, result)
        elif mnemonic == "PushVSPDW":
            ebp = ctx.getRegValue('EBP')
            vm_insn_info = 'PushVSPDW;      EBP = %s' % ebp
        else:
            print("unknow vm insntruction at address %s, vm_insn is %s" % (hex(ctx.addr).upper(), mnemonic))
            continue
        print(vm_insn_info)
```

最终得到一级虚拟指令如下：

```
PopRegDW R3;       [ebp] = 0x00000000
PushConstDW 0x765DA981;
ADDDW [ebp+4], [ebp+0];     [ebp+4] = 0x00000000, [ebp] = 0x765DA981, [ebp+4]+[ebp] = 0x765da981
PopRegDW R7;       [ebp] = 0x00000206
PopRegDW R6;       [ebp] = 0x765DA981
PopRegDW R2;       [ebp] = 0x00401015
PopRegDW R15;       [ebp] = 0x00000246
PopRegDW R11;       [ebp] = 0x0019FF84
PopRegDW R9;       [ebp] = 0x00401015
PopRegDW R13;       [ebp] = 0x0019FFCC
PopRegDW R14;       [ebp] = 0x003F9000
PopRegDW R10;       [ebp] = 0x0019FF64
PopRegDW R4;       [ebp] = 0x00401015
PopRegDW R0;       [ebp] = 0x00401015
PopRegDW R1;       [ebp] = 0x0040100A
PopRegDW R8;       [ebp] = 0x00404781
PushConstDW 0x00403000;
LoadDW [[ebp]];     [ebp] = 0x00403000, [[ebp]] = 0xDEADBEEF
PopRegDW R5;       [ebp] = 0xDEADBEEF
PushConstDW 0x12345678;
PushRegDW R5;      R5 = 0xDEADBEEF
ADDDW [ebp+4], [ebp+0];     [ebp+4] = 0x12345678, [ebp] = 0xDEADBEEF, [ebp+4]+[ebp] = 0xF0E21567
PopRegDW R7;       [ebp] = 0x00000292
PopRegDW R13;       [ebp] = 0xF0E21567
PushConstDW 0x12345678;
PushRegDW R13;      R13 = 0xF0E21567
PushVSPDW;      EBP = 0019FF6C
LoadDW [[ebp]];     [ebp] = 0x0019FF6C, [[ebp]] = 0xF0E21567
NAndDW [ebp+4], [ebp+0];     [ebp] = 0xF0E21567, [ebp+4] = 0xF0E21567, ~[ebp+4] & ~[ebp]=0x0F1DEA98
PopRegDW R1;       [ebp] = 0x00000202
ADDDW [ebp+4], [ebp+0];     [ebp+4] = 0x12345678, [ebp] = 0x0F1DEA98, [ebp+4]+[ebp] = 0x21524110
PopRegDW R15;       [ebp] = 0x00000212
PushVSPDW;      EBP = 0019FF70
LoadDW [[ebp]];     [ebp] = 0x0019FF70, [[ebp]] = 0x21524110
NAndDW [ebp+4], [ebp+0];     [ebp] = 0x21524110, [ebp+4] = 0x21524110, ~[ebp+4] & ~[ebp]=0xDEADBEEF
PopRegDW R8;       [ebp] = 0x00000282
PopRegDW R1;       [ebp] = 0xDEADBEEF
PushRegDW R15;      R15 = 0x00000212
PushVSPDW;      EBP = 0019FF70
LoadDW [[ebp]];     [ebp] = 0x0019FF70, [[ebp]] = 0x00000212
NAndDW [ebp+4], [ebp+0];     [ebp] = 0x00000212, [ebp+4] = 0x00000212, ~[ebp+4] & ~[ebp]=0xFFFFFDED;
PopRegDW R5;       [ebp] = 0x00000286
PushConstCWDE 0xFFFFF7EA;
NAndDW [ebp+4], [ebp+0];     [ebp] = 0xFFFFF7EA, [ebp+4] = 0xFFFFFDED, ~[ebp+4] & ~[ebp]=0x00000010;
PopRegDW R7;       [ebp] = 0x00000202
PushRegDW R8;      R8 = 0x00000282
PushRegDW R8;      R8 = 0x00000282
NAndDW [ebp+4], [ebp+0];     [ebp] = 0x00000282, [ebp+4] = 0x00000282, ~[ebp+4] & ~[ebp]=0xFFFFFD7D
PopRegDW R12;       [ebp] = 0x00000286
PushConstCWDE 0x00000815;
NAndDW [ebp+4], [ebp+0];     [ebp] = 0x00000815, [ebp+4] = 0xFFFFFD7D, ~[ebp+4] & ~[ebp]=0x00000282
PopRegDW R12;       [ebp] = 0x00000206
ADDDW [ebp+4], [ebp+0];     [ebp+4] = 0x00000010, [ebp] = 0x00000282, [ebp+4]+[ebp] = 0x00000292
PopRegDW R12;       [ebp] = 0x00000202
PopRegDW R13;       [ebp] = 0x00000292
PushRegDW R1;      R1 = 0xDEADBEEF
PushConstDW 0x00403000;
StoreDW [[ebp+0]], [ebp+4];     [ebp] = 0x00403000, [ebp+4] = 0xDEADBEEF
PushRegDW R0;      R0 = 0x00401015          R0  = esi
PushRegDW R4;      R4 = 0x00401015          R4  = edi
PushRegDW R8;      R8 = 0x00000282          R8  = vm eflag
PushRegDW R14;      R14 = 0x003F9000        R14 = ebx
PushRegDW R1;      R1 = 0xDEADBEEF         
PushRegDW R9;      R9 = 0x00401015          R9  = edx
PushRegDW R11;      R11 = 0x0019FF84        R11 = ebp
PushRegDW R13;      R13 = 0x00000292
PushRegDW R2;      R2 = 0x00401015          R2  = ecx
PushRegDW R7;      R7 = 0x00000202
PushRegDW R0;      R0 = 0x00401015          R0  = esi
```

### 一级虚拟指令还原成二级虚拟指令

一级虚拟指令还原成二级虚拟指令的方法是利用**虚拟堆栈平衡**，即**二级虚拟指令执行后，虚拟栈保持平衡**。具体步骤如下：

1.  从第一条一级虚拟指令执行前开始，记下虚拟栈栈顶指针相对偏移为 0（或某一值）
2.  执行一级虚拟指令，根据对应一级虚拟指令的栈指针变化情况，更新虚拟栈栈顶指针相对偏移。
3.  直至虚拟栈栈顶指针的相对偏移重新变为 0（或重新变为原值），执行过的一级虚拟指令片段就对应一个二级虚拟指令。

需注意：

*   连续的虚拟出栈指令和连续的虚拟入栈指令，认为这部分指令是虚拟寄存器环境还原和保存。
*   虚拟栈平衡不能以一级虚拟入栈指令结束。
*   虚拟栈平衡不能以一级虚拟运算指令结束。

对于刚才还原出的一级虚拟指令样本，先从最后面分析，往上连续的 PushReg 指令一般都是虚拟寄存器入栈，用于后续恢复真实环境。那么，切割出一级虚拟指令片段如下：

```
PushRegDW R0;       R0  = esi
PushRegDW R4;       R4  = edi
PushRegDW R8;      
PushRegDW R14;      R14 = ebx
PushRegDW R1;      
PushRegDW R9;       R9  = edx
PushRegDW R11;      R11 = ebp
PushRegDW R13;     
PushRegDW R2;       R2  = ecx
PushRegDW R7;      
PushRegDW R0;       R0  = esi
```

那么根据这个，就需要往前寻找相同数量且连续的 PopReg 指令，对应一级虚拟指令片段如下：

```
PopRegDW R2;        R2  = ecx
PopRegDW R15;       R15 = eflag
PopRegDW R11;       R11 = ebp
PopRegDW R9;        R9  = edx
PopRegDW R13;       R13 = eax
PopRegDW R14;       R14 = ebx
PopRegDW R10;       R10 = esp
PopRegDW R4;        R4  = edi
PopRegDW R0;        R0  = esi
PopRegDW R1;        R1  = call sub_40472C的返回地址
PopRegDW R8;        R8  = byte_404781, 即VM Opcode Table
```

这里根据 VM Entry 处的代码进行了虚拟寄存器与真实寄存器之间的映射。

找到了虚拟栈到虚拟寄存器的映射后，就可以从这里开始往下利用虚拟栈平衡划分一级虚拟指令片段了，这里挑出几个片段进行分析。

1.  ```
    //此处记虚拟栈指针偏移为0
    PushConstDW 0x00403000;         -4         
    LoadDW [[ebp]];                 -4          [ebp] = 0x00403000, [[ebp]] = 0xDEADBEEF
    PopRegDW R5;                    0           [ebp] = 0xDEADBEEF
    // 虚拟栈指针偏移重新变为0，以上一级虚拟指令片段对应一个二级虚拟指令
    ```
    
    以上片段可还原成二级虚拟指令
    
    ```
    MOV R5, [0x00403000]; 值为0xDEADBEEF
    ```
    
2.  ```
    //此处记虚拟栈指针偏移为0
    PushConstDW 0x12345678;     -4
    PushRegDW R5;               -8          R5 = 0xDEADBEEF
    ADDDW [ebp+4], [ebp+0];     -8          [ebp+4] = 0x12345678, [ebp] = 0xDEADBEEF, [ebp+4]+[ebp] = 0xF0E21567
    PopRegDW R7;                -4          [ebp] = 0x00000292
    PopRegDW R13;               0           [ebp] = 0xF0E21567
    // 虚拟栈指针偏移重新变为0，以上一级虚拟指令片段对应一个二级虚拟指令
    ```
    
    以上片段可还原成二级虚拟指令
    
    ```
    ADDDW R13, R5, 0x12345678;  R13 = R5 + 0x12345678
    ```
    
3.  ```
    //此处记虚拟栈指针偏移为0
    PushConstDW 0x12345678;     -4
     
    PushRegDW R13;              -8      R13 = 0xF0E21567
    PushVSPDW;                  -12     EBP = 0019FF6C
    LoadDW [[ebp]];             -12     [ebp] = 0x0019FF6C, [[ebp]] = 0xF0E21567
    NAndDW [ebp+4], [ebp+0];    -12     [ebp] = 0xF0E21567, [ebp+4] = 0xF0E21567, ~[ebp+4] & ~[ebp]=0x0F1DEA98
    PopRegDW R1;                -8      [ebp] = 0x00000202
     
    ADDDW [ebp+4], [ebp+0];     -8      [ebp+4] = 0x12345678, [ebp] = 0x0F1DEA98, [ebp+4]+[ebp] = 0x21524110
    PopRegDW R15;               -4      [ebp] = 0x00000212
     
    PushVSPDW;                  -8      EBP = 0019FF70
    LoadDW [[ebp]];             -8      [ebp] = 0x0019FF70, [[ebp]] = 0x21524110
    NAndDW [ebp+4], [ebp+0];    -8      [ebp] = 0x21524110, [ebp+4] = 0x21524110, ~[ebp+4] & ~[ebp]=0xDEADBEEF
    PopRegDW R8;                -4      [ebp] = 0x00000282
    PopRegDW R1;                0       [ebp] = 0xDEADBEEF
    // 虚拟栈指针偏移重新变为0，以上一级虚拟指令片段对应一个二级虚拟指令
    ```
    
    以上片段可还原成二级虚拟指令
    
    ```
    mov R1, ~(~R13 + 0x12345678);
    ```
    
    对于有符号整数`x`，根据补码规则可得`−x=∼x+1`，进一步有`∼x=−(x+1)`，通过这个化简上述二级虚拟指令，可得
    
    ```
    sub R1, R13, 0x12345678;        R13 - 0x12345678
    ```
    
4.  ```
    //此处记虚拟栈指针偏移为0
        // ~R15     ADDDW产生的eflag
    PushRegDW R15;              -4      R15 = 0x00000212;
    PushVSPDW;                  -8      EBP = 0019FF70
    LoadDW [[ebp]];             -8      [ebp] = 0x0019FF70, [[ebp]] = 0x00000212
    NAndDW [ebp+4], [ebp+0];    -8      [ebp] = 0x00000212, [ebp+4] = 0x00000212, ~[ebp+4] & ~[ebp]=0xFFFFFDED;
    PopRegDW R5;                -4      [ebp] = 0x00000286
        // ~0xFFFFF7EA & R15, 结果为0x00000010, AF标志位为1
    PushConstCWDE 0xFFFFF7EA;   -8
    NAndDW [ebp+4], [ebp+0];    -8      [ebp] = 0xFFFFF7EA, [ebp+4] = 0xFFFFFDED, ~[ebp+4] & ~[ebp]=0x00000010;
    PopRegDW R7;                -4      [ebp] = 0x00000202
        // ~R8  NAndDW产生的eflag
    PushRegDW R8;               -8      R8 = 0x00000282
    PushRegDW R8;               -12     R8 = 0x00000282
    NAndDW [ebp+4], [ebp+0];    -12     [ebp] = 0x00000282, [ebp+4] = 0x00000282, ~[ebp+4] & ~[ebp]=0xFFFFFD7D
    PopRegDW R12;               -8      [ebp] = 0x00000286
        // ~0x00000815 & R8
    PushConstCWDE 0x00000815;   -12
    NAndDW [ebp+4], [ebp+0];    -12     [ebp] = 0x00000815, [ebp+4] = 0xFFFFFD7D, ~[ebp+4] & ~[ebp]=0x00000282
    PopRegDW R12;               -8      [ebp] = 0x00000206
        // (~0xFFFFF7EA & R15) + (~0x00000815 & R8)     ~0xFFFFF7EA = 0x00000815    两个eflag合并
    ADDDW [ebp+4], [ebp+0];     -8      [ebp+4] = 0x00000010, [ebp] = 0x00000282, [ebp+4]+[ebp] = 0x00000292
    PopRegDW R12;               -4      [ebp] = 0x00000202
    PopRegDW R13;               0       [ebp] = 0x00000292
    // 虚拟栈指针偏移重新变为0，以上一级虚拟指令片段对应一个二级虚拟指令
    ```
    
    这部分代码进行了复杂的标志位计算，然而其产生的结果在后续代码中并没有用到，因此是一段混淆代码。对于标志位的计算，一般与条件跳转指令有关，即先通过逻辑运算提取某一标志位是否为 1（例如 ZF），然后根据这个结果计算目标跳转指令，接着就是执行条件跳转指令。
    

至于最上面那个一级虚拟指令片段，通过分析发现其虚拟栈并没有平衡，原因在于，VM Entry 处在保存真实环境时，额外入栈了两个数据，加上这两个 push 指令就平衡了。

```
//0
push    ds:dword_404649;                -4
push    0;                              -8
 
PopRegDW R3;                            -4
PushConstDW 0x765DA981;                 -8
ADDDW [ebp+4], [ebp+0];                 -8
PopRegDW R7;                            -4
PopRegDW R6;                            0
```

不过后续并没有用到该 ADDDW 虚拟指令产生的结果和标志位，因此认定为这个片段为混淆指令片段。

最终去除混淆代码以及虚拟寄存器与真实寄存器映射（Pop 和 Push 指令，这个主要是在多个虚拟机中跟踪真实寄存器），还原出的二级虚拟指令如下：

```
mov R5, [0x00403000];           值为0xDEADBEEF
ADDDW R13, R5, 0x12345678;      R13 = R5 + 0x12345678
sub R1, R13, 0x12345678;        R1 = R13 - 0x12345678
mov [0x00403000], R1;           将值存回原位
```

此时的二级虚拟指令可以说是与源指令一模一样了，所实现的功能并无差异。

四、总结
====

对于虚拟代码还原，首要是识别处虚拟机的相关数据结构，例如虚拟栈指针、虚拟字节码指针、虚拟字节码表等等，然后人工分析处 VM Handler 的功能，当然，对于 VM Handler 的分析，可以借助 AI 来完成，需要告知虚拟机的相关数据结构，使 AI 的回答能够更精确。分析完虚拟机的结构以及相关数据结构后，可以尝试提取虚拟机结构的特征，比如 VM Dispatcher 存在 VM Handler Table 的使用，然后可以尝试编写代码自动化虚拟机结构的识别。之后借助模拟执行来获取整个代码流的执行情况，根据这个还原出虚拟代码流的一级虚拟指令。

对于一级虚拟指令还原成二级虚拟指令，需要利用到虚拟栈平衡这一特征来划分一级虚拟指令片段，这些片段对应一条或多条二级虚拟指令（复杂的逻辑运算可以尝试简化）。这里我总结了一点规律，如有不对的地方，恳请斧正：

**因为基于栈的虚拟机，对于一个完整的二级虚拟指令，首先是将操作数据入栈，然后在栈中进行运算，然后将运算产生的标志位或运算结果出栈，因此对于 push 操作，可以认定是二级虚拟指令开始的标志，对于 pop 操作，可以认定是二级虚拟指令结束标志。但仍需要注意的是，运算指令后跟随的第一个出栈指令，是将运算产生的标志位出栈，如果后续没有第二个出栈指令，即运算结果仍然保留在虚拟栈中，则有可能该运算结果仍参与后续的指令的执行，因此还需往后进行虚拟栈平衡分析（test、cmp 除外，它们只在意运算产生的标志位）**

对于模拟执行得到的本机指令以及还原出的一级虚拟指令中，可能存在无用代码，可以借助 ** 变量存活分析（Live-variable analysis）** 识别出无用代码并删除，达到简化代码目的。

参考：

[如何分析虚拟机系列 (1)：新手篇 VMProtect 1.81 Demo - 吾爱破解 - 52pojie.cn](elink@696K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3f1#2x3Y4m8G2K9X3W2W2i4K6u0W2j5$3&6Q4x3V1k6@1K9s2u0W2j5h3c8Q4x3X3b7%4x3e0x3J5x3e0W2Q4x3X3b7I4i4K6u0V1x3g2)9J5k6h3S2@1L8h3H3`.)

https://blog.back.engineering/17/05/2021

[[招生] 科锐逆向工程师培训 (2025 年 3 月 11 日实地，远程教学同时开班, 第 52 期)！](https://bbs.kanxue.com/thread-51839.htm)

[#软件保护](forum-4-1-3.htm) [#VM 保护](forum-4-1-4.htm)