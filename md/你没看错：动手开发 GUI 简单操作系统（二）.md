> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1369814-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/001/32/18/04_avatar_middle.jpg)TLHorse  

> www.52pojie.cn @TLHorse  
> 原创作品

[《你没看错：动手开发 GUI 简单操作系统（一）》](https://www.52pojie.cn/thread-1369484-1-1.html)

学习目标
====

1.  编写 GDT
2.  切换到 32 位保护模式（32-bit protected mode，又叫 PM）

> 我突然发现写到第二篇文章就难以启齿了，因为切 PM、加载内核这些东西几乎是环环相扣的，有一点差错都不行。
> 
> 另外劝大家一定要学好英文，怎么着也得七八千词汇量吧。

编写 GDT
======

GDT 是最难理解的部分，却不可避免，不好意思，我这次又整了一大段理论写在这里。我觉得下面说的还算比较全吧，如果我一下引入那么多名词，我都难以下笔。但是，必须看，如果实在读不进去，看**加粗字体**：

认识全局描述符表
--------

> 为了行文方便，下文使用缩写。
> 
> <table><thead><tr><th>中文名称</th><th>英文名称</th><th>缩写</th></tr></thead><tbody><tr><td>全局描述符表</td><td>Global Descriptor Table</td><td>GDT</td></tr><tr><td>保护模式</td><td>Protected Mode</td><td>PM</td></tr><tr><td>真实模式</td><td>Real Mode</td><td>RM</td></tr></tbody></table>

GDT 在 PM 下，是一个重要的必不可少的数据结构。

为什么要有 GDT？我们首先考虑一下在 RM（就是切 PM 之前）下的编程模型：在 RM 下，我们对一个内存地址的访问是通过`Segment:Offset`的方式来进行的，其中 Segment 是一个段的基地址，一个 Segment 的最大长度是 64KB，这是 16 位系统所能表示的最大长度。而 Offset 则是相对于此段基地址的偏移量。**基地址 + 偏移就是一个内存绝对地址。**

由此，我们可以看出，一个段具备两个因素：基地址和段的最大长度。而对一个内存地址的访问，则是需要指出两点：

1.  使用的是哪个段；
2.  相对于这个段基地址的偏移：这个偏移应该小于此段的最大长度。

当然对于 16 位系统，最大长度不用指定，默认为最大长度 64KB，16 位的便宜也永远不可能大于最大长度。而我们在实际编程的时候，使用 16 位段寄存器 CS，DS，SS 来指定段，CPU 将段寄存器中的数值向左偏移 4 位，放到 20 位的地址线上就成为 20 位的基地址。

到了 PM，内存的管理模式分为两种，段模式和页模式，其中页模式也是基于段模式的。也就是说，PM 的内存管理模式事实上是：纯段模式和段页式。进一步说，段模式是必不可少的，而页模式则是可选的——如果使用页模式，则是段页式；否则这是纯段模式。

既然是这样，我们就先不去考虑页模式。对于段模式来讲，访问一个内存地址仍然使用 Segment:Offset 的方式，这是很自然的。由于 PM 运行在 32 位系统上，那么 Segment 的两个因素：基地址和最大长度也都是 32 位的。IA-32 允许将一个段的基地址设为 32 位所能表示的任何值（最大长度则可以被设为 32 位所能表示的 2<sup>12</sup > 的整数倍的任何值），而不像 RM 下，一个段的基地址只能是 16 的倍数（因为其低 4 位是通过左移运算得来的，只能为 0，从而达到使用 16 位段寄存器表示 20 位基地址的目的），而一个段的最大长度只能为固定值 64KB。

**另外 PM 顾名思义，就是为段访问提供了保护机制**，也就说一个段的描述符需要规定对自身的访问权限（Access）。所以在 PM 下，对一个段的描述则包括 3 方面因素：Base Address（基地址）、Limit（最大长度）、Access（访问权限），它们加在一起被放在一个 64 位长的数据结构中，被称为段描述符。这种情况下，如果我们直接通过一个 64 位段描述符来引用一个段的时候，就必须使用一个 64 位长的段寄存器装入这个段描述符。Intel 为了保持向后兼容，但将段寄存器仍然规定为 16 位（尽管每个段寄存器事实上有一个 64 位长的不可见部分，但对于编程人员来说段寄存器就是 16 位的），那么很明显，我们无法通过 16 位长度的段寄存器来直接引用 64 位的段描述符。怎么办？**解决的方法就是把这些长度为 64 位的段描述符放入一个数组中，而将段寄存器中的值作为下标索引来间接引用（事实上，是将段寄存器中的高 13 位的内容作为索引）。**

**——这个全局的数组就是 GDT。**事实上，在 GDT 中存放的不仅仅是段描述符，还有其它描述符，它们都是 64-bit 长，我们随后再讨论。GDT 可以被放在内存的任何位置，那么当程序员通过段寄存器来引用一个段描述符时，C**PU 必须知道 GDT 的入口，也就是基地址放在哪里，所以 Intel 的设计者门提供了一个寄存器`gdtr`用来存放 GDT 的入口地址**。程序员将 GDT 设定在内存中某个位置之后，可以通过 LGDT 指令将 GDT 的入口地址装入此寄存器，从此以后，CPU 就根据此寄存器中的内容作为 GDT 的入口来访问 GDT 了。GDT 是 PM 所必须的数据结构，也是唯一的——不应该，也不可能有多个 GDT。

另外，正像它的名字 Global Descriptor Table 所揭示的，它是全局可见的，对任何一个任务而言都是这样。除了 GDT 之外，IA-32 还允许程序员构建与 GDT 类似的数据结构，它们被称作 LDT，但与 GDT 不同的是，**LDT 在系统中可以存在多个，并且从 LDT 的名字可以得知，LDT 不是全局可见的，它们只对引用它们的任务可见，每个任务最多可以拥有一个 LDT。**

**另外，每一个 LDT 自身作为一个段存在，它们的段描述符被放在 GDT 中。**IA-32 为 LDT 的入口地址也提供了一个寄存器 LDTR，因为在任何时刻只能有一个任务在运行，所以 **LDT 寄存器全局也只需要有一个**。如果一个任务拥有自身的 LDT，那么当它需要引用自身的 LDT 时，它需要通过 LLDT 将其 LDT 的段描述符装入此寄存器。

LLDT 指令与 LGDT 指令不同的是，LGDT 指令的操作数是一个 32 位的内存地址，这个内存地址处存放的是一个 32 位 GDT 的入口地址，以及 16 位的 GDT 最大长度。而 LLDT 指令的操作数是一个 16 位的选择子，这个选择子主要内容是：**被装入的 LDT 的段描述符在 GDT 中的索引值**——这一点和刚才所讨论的通过段寄存器访问段的情况是一样的。

现在你知道为什么要有 GDT 了吧……

GDT 结构
------

 ![](https://attach.52pojie.cn/forum/202102/11/172427na21tajed5uyqq3u.png)   

这是 GDT 的结构。其中 Flags 和 Access Byte 部分又分为如下表格：

 ![](https://attach.52pojie.cn/forum/202102/11/172436wc5xzpxws5hct1wb.png)   

编写 GDT
------

我们先讲讲上面两个图每一项都代表什么，并且应该设置什么值。

<table><thead><tr><th>图中标签</th><th>中文</th><th>英文</th><th>值</th><th>说明</th></tr></thead><tbody><tr><td>Pr</td><td>展示</td><td>present</td><td>1</td><td>段在内存中被展示</td></tr><tr><td>Privl</td><td>访问权限</td><td>privilege</td><td>0</td><td>俗话说的 ring 级别。ring0 = 最高权限（内核）；ring3 = 最低权限（用户 App）</td></tr><tr><td>S</td><td>描述符类型</td><td>descriptor type</td><td>1</td><td>设为 1，表示是 CS/DS</td></tr><tr><td>Ex</td><td>是否为可执行</td><td>Executable</td><td>1</td><td>1 表示可执行，说明是代码；0 表示不可执行，说明是数据</td></tr><tr><td>DC</td><td>指示 / 遵循</td><td>Direction/Conforming</td><td>0</td><td>代码段中，0：代码只能被 Privl 权限执行，1：代码可以被≤Privil 权限执行。数据段中，0：段从下到上，1：段从上到下。</td></tr><tr><td>RW</td><td>读写性</td><td>Readable/Writable</td><td>1</td><td>对于代码段：1 是可读（可以获取常量），0 是可执行，不可写；对于数据段：1 是可写，2 是可读，不可执行</td></tr><tr><td>Ac</td><td>已访问段</td><td>Accessed</td><td>0</td><td>设为 0。当 CPU 访问段，access 会设成 1，由 CPU 控制</td></tr><tr><td>Gr</td><td>粒度</td><td>Granularity</td><td>1</td><td>如果是 1，我们的 limit 会扩大四倍</td></tr><tr><td>Sz</td><td>大小</td><td>Size</td><td>1</td><td>1 代表使用 32 位 PM，0 是使用 16 位 PM</td></tr><tr><td>0（第一个）</td><td>64 位代码段</td><td>64-bit CS</td><td>0</td><td>32 位处理器我们不用，0</td></tr><tr><td>0（第二个）</td><td>系统可使用</td><td>AVailabLe for use by system software （AVL）</td><td>0</td><td>调试用，0</td></tr></tbody></table>

我们根据需要的值写一个 GDT。GDT 有不止一种写法（也有许多比这高级的），我这种写法是定义了一个数据体，简单明了，但是可能不易于后期维护，项目里新建`gdt.asm`：

```
gdt_start: ; 在这里写一个标签，待会用来计算大小

gdt_null:  ; 这个叫做空段，是Intel预留的
    dd 0   ; DD = double word  （此处也可以把两个dd合并为一个dq）
    dd 0   ;    = 4 byte

gdt_code: 
    dw 0xFFFF    ; Limit            0-15 bits
    dw 0         ; Base address     0-15
    db 0         ; 同上              16-23
    db 10011010B ; 按照图二Access Byte从右至左（0-7）的顺序填写。注意Privl因为值可以是0-3，所以说占两字节，填两个0
    db 11001111B ; 按图二flags从右至左填写。我们再看图一，flags右边还有limits最后4位，不满8位编译不通过，所以把limits合并在flags右面，0xF=0b1111
    db 0         ; Base             24-31

gdt_data:        ; 同上
    dw 0xFFFF
    dw 0
    db 0  
    db 10010010B ; 不同的是这里把Ex改成0，因为这里是数据段
    db 11001111B
    db 0

gdt_end:         ; gdt结束标签，

; GDT
gdt_descriptor:
    dw gdt_end - gdt_start - 1 ; 大小=结束-开始-1（真实大小永远-1）
    dd gdt_start               ; 开始地址

; 常量
CODE_SEG equ gdt_code - gdt_start ; CS
DATA_SEG equ gdt_data - gdt_start ; DS

```

GDT 大功告成！**如果你没有编译成功，出了问题，请留言跟帖或者私信，我一定会回复！**

在 32 位保护模式下打印
=============

我们现在写一个文件`32bit-print`，用来在 32 位保护模式下打印，不过只用来测试，到时候加载了内核切了 GUI 就可以删了。

```
[bits 32]

VIDEO_MEMORY equ 0xB8000 ; 这是VGA开始的地方
WHITE_ON_BLACK equ 0x0F  ; 是一个颜色代码，黑背景，白色字符

print_string_pm:
    pusha
    mov edx, VIDEO_MEMORY

print_string_pm_loop:
    mov al, [ebx]          ; [ebx]是字符串参数
    mov ah, WHITE_ON_BLACK ; ah颜色参数

    cmp al, 0 ; 是不是已经到末尾？
    je print_string_pm_done ; 结束

    ; 如果不是
    mov [edx], ax ; 在Vram中保存字符
    add ebx, 1 ; 下一个字符
    add edx, 2 ; 下一个Vram字符位置，+2

    jmp print_string_pm_loop ; 递归

print_string_pm_done:
    popa
    ret

```

切换到 32 位保护模式
============

简单说一下为什么会有所谓保护模式：仔细想一想就能知道，如果这个 OS 给用户用就是开玩笑，就好像用户和操作系统一块在计算机里玩，而不是用户在操作系统里玩，它没有安全性可言，可以随便访问内存，更别提什么 ring0、ring3 的了，所以说切 PM 是为了让 OS 得到保护。

新建代码`switch_pm.asm`，很简单。主要目标是认识并使用这个控制寄存器 cr0：

```
[bits 16] ; 代表在16位模式下。中括号可以去掉
switch_to_pm:
    cli                   ; 1. 一定要关掉CPU中断！让CPU专心干一件事
    lgdt [gdt_descriptor] ; 2. 还记得lgdt命令吗？加载我们的gdt_descriptor标签
    mov eax, cr0          ; 把cr0暂存到eax
    or eax, 0x1           ; 3. cr0设置为1，切到32位PM
    mov cr0, eax
    jmp CODE_SEG:init_pm  ; 4. 远跳转，跳到代码段（下面）

[bits 32] ; 32位！
init_pm:
    mov ax, DATA_SEG ; 5. 更新段寄存器
    mov ds, ax
    mov ss, ax
    mov es, ax  ; 把每个都刷一遍
    mov fs, ax
    mov gs, ax

    mov ebp, 0x90000 ; 6. 基址指针寄存器，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的底部，把它放到0x90000，待会加载内核
    mov esp, ebp

    call BEGIN_PM ; 跳到bootsect

```

`cr0`全称叫做 Control Register 0，这个控制寄存器专门用来在 RM 和 PM 之间切换。

我们编辑一下`bootsect.asm`，使用上面的 “函数”：

```
[org 0x7C00]
    mov bp, 0x9000
    mov sp, bp

    mov bx, MSG_REAL_MODE
    call print ; 在实模式下打印

    call switch_to_pm ; 切PM
    ; 这里不管加什么代码都不会被执行

%include "print.asm"
%include "gdt.asm"
%include "32bit-print.asm"
%include "switch_pm.asm"

[bits 32] ; 32位
BEGIN_PM: ; 切换后跳转到这里
    mov ebx, MSG_PROT_MODE ; 打印
    call print_string_pm ; 注意！！！这个信息会被打印到整个屏幕的最左上角，覆盖BIOS的打印
    jmp $ ; 挂起

MSG_REAL_MODE db "Started in 16-bit real mode", 0
MSG_PROT_MODE db "Loaded 32-bit protected mode", 0

times 510-($-$$) db 0
dw 0xAA55

```

编译运行两步走：

 ![](https://attach.52pojie.cn/forum/202102/11/172456y74jynqno0q6q6pe.png)   

加载并执行空内核
========

切 32 位 PM 是为加载内核准备的，我就一块整了吧，不然切 PM 毫无用处。

首先我们得认识到，汇编作为底层语言的代价是功能太少，不方便我们编程。内核编程需要用到 C。但是一个操作系统项目里，怎么能容得下两种语言编译呢？你别着急，我们的做法是将启动扇区和内核部分分开，启动扇区编译成一个 bin，内核编译成一个 bin，编译的过程中我们把 ld 命令做一些调整，使数据便宜到 0x1000，之后把两个 bin 用 cat 合并。

内核编写
----

所以说咱们先开始写内核吧，新建`kernel.c`：

```
void main() {
    char *video_memory = (char*) 0xb8000;
    *video_memory = 'V';
    char *video_memory1 = (char*) 0xb8002;
    *video_memory1 = 'e';
    char *video_memory2 = (char*) 0xb8004;
    *video_memory2 = 'n';
    char *video_memory3 = (char*) 0xb8006;
    *video_memory3 = 'u';
    char *video_memory4 = (char*) 0xb8008;
    *video_memory4 = 's';
}


```

它的作用就是到 VGA 的地址打印`Venus`。

汇编编写
----

同级新建`kernel_entry.asm`，仅有四行代码：

```
[bits 32]
[extern main] ; 像C一样，对外部函数得先声明。这个main就是kernel.c里面那个
call main     ; 执行
jmp $

```

是时候到`bootsect.asm`加载了：

```
[org 0x7c00]
KERNEL_OFFSET equ 0x1000 ; 定义常量，这个常量是内核的位置

    mov [BOOT_DRIVE], dl ; BIOS会自动把磁盘编号设置到dl。我们在下面间一个常量，先存起来，因为dl可能会被覆盖
    mov bp, 0x9000
    mov sp, bp

    mov bx, MSG_REAL_MODE ; 实模式打印
    call print
    call print_nl

    call load_kernel  ; 加载内核
    call switch_to_pm ; 切PM

%include "print.asm"
%include "print_hex.asm"
%include "disk.asm"
%include "gdt.asm"
%include "print.asm"
%include "switch_pm.asm"

[bits 16]
load_kernel:
    mov bx, MSG_LOAD_KERNEL
    call print
    call print_nl

    mov bx, KERNEL_OFFSET ; 读取到内核偏移地址
    mov dh, 2
    mov dl, [BOOT_DRIVE]
    call disk_load
    ret

[bits 32]
BEGIN_PM:
    mov ebx, MSG_PROT_MODE
    call print_string_pm
    call KERNEL_OFFSET ; 执行内核代码
    jmp $

BOOT_DRIVE db 0
MSG_REAL_MODE db "Started in 16-bit Real Mode", 0
MSG_PROT_MODE db "Landed in 32-bit Protected Mode", 0
MSG_LOAD_KERNEL db "Loading kernel into memory", 0

times 510 - ($-$$) db 0
dw 0xAA55

```

 ![](https://attach.52pojie.cn/forum/202102/11/172510okzi66iy4aqyrt73.png)   

今天就到这里，**以后更新可能会慢点**，太累了，我还要干别的呢。项目要么就不做，要么就做成。我的个人理念是不做就不做，做就做大。

![](https://avatar.52pojie.cn/data/avatar/000/37/91/37_avatar_middle.jpg)E 式丶男孩 功不功利吧，总是写一堆 demo，最后啥用都没，稀里糊涂的让人很不舒服的。  
支持你的精神 ![](https://avatar.52pojie.cn/data/avatar/000/74/07/98_avatar_middle.jpg) RemMai 特意前来跟帖  
1. 上次回复 [你以为我看的懂？] 并不是说你文章写得不好，而是本人笨拙，需要花费很多时间来学习。  
2. 既然坛里说要好好回复，那我就发表一份我认真看完的观点：  
首先需要通过一种语言来实现最底层的搭建，也就是需要把电脑上面的设备驱动起来，有了这个基础后，后面的工作才能陆陆续续的进行。![](https://avatar.52pojie.cn/data/avatar/000/83/54/29_avatar_middle.jpg)侃遍天下无二人 楼主我能把文章下到本地吗，准备用树莓派练习练习![](https://avatar.52pojie.cn/data/avatar/001/50/12/16_avatar_middle.jpg)雷欧库珀 似懂不懂，不明觉厉 ![](https://avatar.52pojie.cn/data/avatar/001/56/79/68_avatar_middle.jpg) ly260248556 似懂不懂，不明觉厉![](https://avatar.52pojie.cn/data/avatar/001/32/79/00_avatar_middle.jpg)Eaglecad 不明觉厉，但是加油 ![](https://avatar.52pojie.cn/data/avatar/000/09/61/60_avatar_middle.jpg) xiaosuobjsd 来看看中奖了没 ![](https://avatar.52pojie.cn/data/avatar/001/48/03/47_avatar_middle.jpg) x761532475 可以看看 ![](https://avatar.52pojie.cn/data/avatar/001/55/33/65_avatar_middle.jpg) dongge666 一脸懵，不过感觉好厉害的样子 ![](https://avatar.52pojie.cn/data/avatar/000/41/64/33_avatar_middle.jpg) youngnku 看不懂，不明觉厉