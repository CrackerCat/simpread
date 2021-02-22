> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1369484-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/001/32/18/04_avatar_middle.jpg)TLHorse  

[你没看错：动手开发 GUI 简单操作系统（一）](https://www.52pojie.cn/thread-1369484-1-1.html)  
[你没看错：动手开发 GUI 简单操作系统（二）](https://www.52pojie.cn/thread-1369814-1-1.html)  
正在更新……

前言
==

今天我终于想好发布这篇文章，以前自己一直在摸索开发，保证 100% 原创。这个操作系统异常简单，没有 Windows 的高级，没有 OS X 的华丽，更没有 Linux 的强大——也别指望了，**对于个人来说根本没多少生产力，只能用来学习知识，自己整着玩。**但是，OS 开发的资料太少了，“你没看错” 系列中的每一行代码，确实是作者我本人摸滚打爬才得来的。

或许我的文字在各位大佬眼中会很简单。所以说，我尽力吧，简明易懂，不加废话。如果有不专业的地方，直接留言改正，谢谢。

我准备出一系列 “你没看错” 文章，一定会有后续的。OS 一篇文章讲不完，我的写法是理论和实践相辅相成，一点点讲。

学习目标
====

第一天我们的目标很简单，主要是写启动扇区：

1.  实现在启动扇区打印字符串
2.  在启动扇区打印地址
3.  添加读取磁盘的功能

这些实现主要是为以后加载内核、出现错误调试做准备。

要求知识
====

1.  汇编语言不要求精通，但一定要熟悉，有基本了解；
2.  C 语言要会，写内核要用；
3.  shell 必须会敲命令，没得说；
4.  可以先修一些附加技能，比如 gdb、Makefile 等，也可以先了解相关概念。

环境配置
====

在开发之前，我们需要配置开发环境。我使用的是 Mac，终端用的是 zsh。如果有能力，可以用 Linux，因为 Linux 包含开发过程中大部分的工具。如果是 Windows…… 那就去论坛下载个虚拟机，使用 Linux 吧。为了不让诸位一上来就被各种安装震慑住，我们开发一点安装一点。首先（假设你有`Homebrew`，一定要**换源**）：

```
brew install qemu nasm # 怎么样？很简单吧

```

简单认识一下：`qemu`是个开源的模拟器，`nasm`是 Netwide 汇编编译器。

加载启动扇区
======

我们的操作系统，从`bootsector`写起。这个`bootsector`是个启动扇区。**当这个分区被识别有效后，系统就会启动。**我们的首要目标是创建能被识别的 bs。

为了检测磁盘是可启动的，BIOS 会检测第 511 和 512 字节是否为十六进制 AA55。记住 0xAA55，这个数字是硬件开发者所设置的。

新建你的项目文件夹，给你自己的系统起个名字，比如我的叫`Venus`。创建`bootsect.asm`：

```
loop:
    jmp loop ; 开始递归，在这里做无限循环。其实也可以用hlt或者jmp $实现。

times 510 - ($-$$) db 0 ; 在bs前放上510个0
dw 0xAA55 ; 在第511字节处，定义0xAA55，覆盖两个字节

```

我们编译、模拟两步走：

```
$ nasm -fbin boot_sect.asm -o boot_sect.bin 
$ qemu-system-x86_64 boot_sect.bin # 如果错误，改成qemu boot_sect.bin

```

 ![](https://attach.52pojie.cn/forum/202102/10/180025kali76772liezqvi.png)   

**至此，你迈开了第一步！**系统在引导之后，进入了无限循环。

输出至屏幕
=====

先来了解一下中断：

> 在点击鼠标或键盘时（正如我现在在做的事情），计算机会立即给我反馈处理结果，计算机与我们之间是在进行实时交互的。**而实时性的实现便是依赖了中断**，中断是为了顺应人们对实时性交互的需求而产生的技术。中断之所以有用，是因为它会立刻停下当前的程序（软件）去做另外一件事。

我们希望启动后，让系统在屏幕上输出几个字符：'Venus'。**我们需要用到`int 0x10`。这个中断用于控制屏幕输出，它好比一个约定俗成的函数，有两个参数，ax 寄存器的低位 al 就是要输出的字符，高位 ah 就是控制输出模式的指示符。**代码如下：

```
mov ah, 0x0E ; 指示符为0x0E代表tty模式（你应该知道tty是什么，TeleTYpe）
mov al, 'V'  ; 把al赋值'V'
int 0x10     ; 终端输出
mov al, 'e'  ; 重复以上流程
int 0x10
mov al, 'n'
int 0x10
mov al, 'u'
int 0x10
mov al, 's'
int 0x10

jmp $

; BIOS识别的数字
times 510 - ($-$$) db 0
dw 0xAA55 

```

我们编译、模拟两步走：

 ![](https://attach.52pojie.cn/forum/202102/10/180127rtegckt9eift1gsi.png)   

完善打印功能
======

为了方便我们今后的调试，我们需要完善打印功能，这样出了什么差错直接 print 就 OK 了。我们的打印分为两种：打印字符串和打印地址。

打印字符串
-----

都知道，C 语言中的字符串结构长这样：

```
"Venus" -> 'V' 'e' 'n' 'u' 's' '0x0'

```

都是几个字符再加上一个空字节 0x0。如果要打印字符串，而不是单个字符，在汇编里面，可以对应成一个栈来处理。同目录新建一个`print.asm`：

```
print:
    pusha ; 将所有东西压入栈

; 记住：一直循环打印栈的字符，直到碰到字符串末0x0
; while (string[i] != 0) { print string[i]; i++ }

start:
    mov al, [bx] ; bx相当于字符串参数，是字符串的首位
    cmp al, 0    ; al和0比较
    je done      ; 如果相等，就到了字符串末尾，跳转到结束done

    mov ah, 0x0E ; 如果不相等，开始打印，先进入tty模式
    int 0x10     ; 直接中断。因为al参数已经有字符了

    add bx, 1    ; 如果你把这个栈+1，相当于地址后移一位，这样再打印就是下一个字符串
    jmp start    ; 递归

done:
    popa         ; 弹出栈  
    ret          ; 返回主程序

```

我们再空几行，实现一个附加功能——换行：

```
print_nl:        ; print NewLine
    pusha

    mov ah, 0x0E ; tty模式
    mov al, 0x0A ; 把0x0A和0x0D合起来相当于\n
    int 0x10
    mov al, 0x0D ; 把0x0A和0x0D合起来相当于\n
    int 0x10

    popa
    ret

```

打印地址（4 位）
---------

打印地址也很有用的。但它涉及到一个把指定字符转换为 ASCII 的问题。因为传入的参数不是带引号的字符串，而是譬如 0x1234 这样的地址，那到底应该打印什么呢？转换方法如下：

### 字符与 ASCII 对应关系

数字转换：0~9 是 0x30~0x39，所以**把数字加上 0x30** 即是 ASCII；  
字母转换：A~F（当成 1~6）是 0x41~0x46，所以**把字母加上 0x40**。

### 代码

```
print_hex:
    pusha
    mov cx, 0 ; cx在循环指令和重复前缀中，作循环次数计数器

; 参数dx：要打印的地址
hex_loop:
    cmp cx, 4 ; cx是不是已经循环了四次？
    je end    ; 如果是，跳转到end结束

    ; 如果不是：开始处理

    mov ax, dx     ; 在ax上对字符处理，（dx是我们的地址参数）
    and ax, 0x000F ; 先把这个地址只保留最后一位。比如0x1234就变成0x0004
    add al, 0x30   ; 加上30，这样4就会变成ASCII：34（别忘了这个al是ax的一部分，是一个寄存器——
    cmp al, 0x39   ; 如果发现这个数字>9，不是0～9，那么这个数字就是字母，加上7，就会是A~F中的一个
    jle step2      ; Jump if Lower or Equal：al小于等于0x39跳转至step2
    add al, 7

step2:
    ; 第二步：我们的ASCII字符应该放在哪个地址呢？
    ; 地址BX：基地址+字符串长度（5位，别忘了还有最后的0x0）-字符索引
    mov bx, HEX_OUT + 5 ; 基+长
    sub bx, cx          ; -索引
    mov [bx], al        ; 把al中的字符移到[bx]，中括号表示地址的内容
    ror dx, 4           ; ROll Right：0x1234 -> 0x4123 -> 0x3412 -> 0x2341 -> 0x1234. ror帮我们实现类似遍历字符串的效果。你可以去掉这行指令，看看会发生什么

    add cx, 1           ; 循环计数器+1
    jmp hex_loop        ; 回到循环

end:
    mov bx, HEX_OUT     ; 把HEX_OUT设置到bx里，作为下一个call的参数
    call print          ; 调用写好的print.asm

    popa
    ret

HEX_OUT:
    db '0x0000', 0 ; 这是我们输出的地址，先定义下来

```

尝试使用打印功能
--------

编写`bootsect.asm`：

```
[org 0x7C00]

mov bx, GREETINGS ; 设置参数
call print        ; 打印
call print_nl     ; 换行

mov dx, 0x4567    ; 设置参数（地址）
call print_hex    ; 打印十六进制

mov bx, SHUTDOWN  ; 同上
call print
call print_nl

jmp $             ; 挂起程序，无限循环（hlt也行）

%include "boot_sect_print.asm"
%include "boot_sect_print_hex.asm"

; 定义两个数据，注意末尾一定要带0字节
GREETINGS:
    db 'Welcome to Venus', 0

SHUTDOWN:
    db 'Shutdown', 0

times 510-($-$$) db 0
dw 0xAA55

```

说明两个地方：

1.  `[org 0x7C00]`：org 是用来设置程序基址的。因为 BIOS 将 bs 加载到 0x7C00 的位置，所以我们设置基址为 0x7C00。这行指令的中括号去掉也行。
2.  `%include`：用来引用文件，后面跟上空格和双引号，双引号里写文件名称。值得注意的是，**`%include`命令相当于把引用的文件直接替换到程序中**，不做任何操作。

还是按老办法编译、模拟：

 ![](https://attach.52pojie.cn/forum/202102/10/180148uidxx21c9crzq9tt.png)   

读取磁盘
====

好了，最枯燥却最有用的功能来了，读取磁盘。我们总不能神经质地把整个系统都放在启动扇区。我们先了解一下磁盘（这个部分必须看）：

磁盘基础
----

### 盘片、片面和磁头

 ![](https://attach.52pojie.cn/forum/202102/10/180223bllgevv21pe7n27t.png)   

硬盘中一般会有多个盘片组成，每个盘片包含两个面，每个盘面都对应地有一个读写磁头。受到硬盘整体体积和生产成本的限制，盘片数量都受到限制，一般都在 5 片以内。盘片的编号自下向上从 0 开始，如最下边的盘片有 0 面和 1 面，再上一个盘片就编号为 2 面和 3 面。

### 扇区（sector）和磁道（track）

 ![](https://attach.52pojie.cn/forum/202102/10/180239rd8ovwmhz4w0c2dz.png)   

上图显示的是一个盘面，盘面中一圈圈灰色同心圆为一条条磁道，从圆心向外画直线，可以将磁道划分为若干个弧段，每个磁道上一个弧段被称之为一个扇区（图践绿色部分）。扇区是磁盘的最小组成单元，通常是 512 字节。（由于不断提高磁盘的大小，部分厂商设定每个扇区的大小是 4096 字节）。

### 磁头（head）和柱面（cylinder）

 ![](https://attach.52pojie.cn/forum/202102/10/180247ixxseexib0fxbzbz.png)   

硬盘通常由重叠的一组盘片构成，每个盘面都被划分为数目相等的磁道，并从外缘的 “0” 开始编号，具有相同编号的磁道形成一个圆柱，称之为磁盘的柱面。磁盘的柱面数与一个盘面上的磁道数是相等的。由于每个盘面都有自己的磁头，因此，盘面数等于总的磁头数。

开始读取吧！
------

我就直接放代码了，没什么技术含量，只不过有一些关键的寄存器数值与中断号码需要明白：

```
; 参数：
;   - dh：扇区个数
;   - dl：磁盘
; 读取的数据存入es:bx

disk_load:
    pusha              ; 压入栈
                       ; 将dx也压入栈
    push dx            ; dx一会会被读取磁盘的操作覆盖，所以先压入栈保存

    mov ah, 0x02       ; BIOS 读取扇区的功能编号
    mov al, dh         ; AL - 扇区读取个数，也就是我们的dh
    mov cl, 0x02       ; CL - 从哪里开始读取，因为第一个扇区是启动扇区，所以这里是0x02
    mov ch, 0x00       ; CH - 柱面编号(0x0-0x3FF)
    mov dh, 0x00       ; DH - 磁头编号(0x0-0xF)

    int 0x13           ; 读取磁盘的中断标号
    jc disk_error      ; Jump if Carry：如果CF被设置，就是出现了错误，跳转

    ; 如果没有错误
    pop dx             ; dx我们用完了，弹出栈
    cmp al, dh         ; 此时bios会把al设置为扇区个数，对比一下
    jne sectors_error  ; 如果两者不一样，读取扇区出现了错误，跳转
    popa               ; 如果一样，停止程序
    ret

; 剩下的是错误处理部分，大家都明白
disk_error:
    mov bx, DISK_ERROR
    call print
    call print_nl
    mov dh, ah
    call print_hex
    jmp disk_loop

sectors_error:
    mov bx, SECTORS_ERROR
    call print

disk_loop:
    jmp $

DISK_ERROR: db "Disk read error", 0
SECTORS_ERROR: db "Incorrect number of sectors read", 0

```

我们将启动扇区代码`bootsect.asm`做出如下更改：

```
[org 0x7C00]
mov bp, 0x8000 ; 把栈顶设成0x8000，这样不与BIOS相干
mov sp, bp     ; 同上
mov bx, 0x9000 ; es:bx == 0x0000:0x9000 == 0x09000

; 现在我们要设置disk_load参数
mov dh, 2 ; 读取两个扇区
; 此处不用设置dl，BIOS已经帮我们设置过了
call disk_load ; 调用

mov dx, [0x9000] ; 获取第一扇区
call print_hex
call print_nl

mov dx, [0x9000 + 512] ; 获取第二扇区（注意偏移地址，跟下面数据对应）
call print_hex

jmp $

%include "print.asm"
%include "print_hex.asm"
%include "disk.asm"

times 510 - ($-$$) db 0
dw 0xAA55

; 上面是bs（第一个扇区）
times 256 dw 0x1234 ; 第2
times 256 dw 0x5678 ; 第3
; 上面的第二第三也不一定，因为有的磁盘一个扇区512，现在有的4096
; …………

```

 ![](https://attach.52pojie.cn/forum/202102/10/180304i0oa00201iz6mpm6.png)   

后记
==

别着急，这只是第一天呢，离加载内核还远着呢，项目里只有四个文件。我给大家指指路，我们已经可以读取磁盘，接下来我们需要：

1.  加载启动扇区
2.  读取磁盘，加载内核
3.  从命令行转成 GUI 图形界面
4.  设置 GDT（代码最简单，但是最困难的部分，也消耗了我的大部分研究时间）
5.  切换到 32bit 保护模式
6.  执行内核：kernel_main
7.  正式切换到 C 语言！

剩下的几个步骤我会划分成几天的内容，发布文章讲解。

其实我写着写着突然想到这不就跟革命斗争一样吗，在执行内核前是多么煎熬，执行内核切换 C 语言后跟解放了一样。

一点点来吧。

相信我！一定有后续！！！很快就出
================

THE END
=======

![](https://avatar.52pojie.cn/data/avatar/000/87/90/80_avatar_middle.jpg)涛之雨

> [TLHorse 发表于 2021-2-10 18:14](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=36857476&ptid=1369484)  
> 啊？似乎我这篇文章写得很失败  
> 我失望极了 查看 82 回复 4

好的作品需要时间来检测，而不是多少人查看和多少人去回复。  
与其满眼都是回复 “感谢楼主”，“学到了，感谢” 之类的灌水（注：论坛禁止恶意灌水！）还不如看到几个回复都是认真阅读后的感悟或是疑问等。  
（小声逼逼估计很多人也像我一样看不懂，不敢说话![](https://static.52pojie.cn/static/image/smiley/laohu/laohu7.gif)，自己写操作系统这种东西太过高深，过于遥远。。。） ![](https://avatar.52pojie.cn/data/avatar/001/44/43/90_avatar_middle.jpg) 《30 天自制操作系统》![](https://avatar.52pojie.cn/data/avatar/001/54/11/07_avatar_middle.jpg)networkdwl

> [TLHorse 发表于 2021-2-10 19:18](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=36858114&ptid=1369484)  
> 技术就是拿来折腾的  
> 不管怎么样，现在许多台式机启动还都是传统 BlOS，mbr 这类的啊

应该叫兼容传统 MBR 启动, 自从 15 年以后的 PC 几乎都是 uefi 模式启动的多 ![](https://avatar.52pojie.cn/data/avatar/000/55/21/86_avatar_middle.jpg) MSLOS 反正我也看不懂。。。。。![](https://avatar.52pojie.cn/data/avatar/001/03/45/21_avatar_middle.jpg)testunpack 火鉗流氓![](https://avatar.52pojie.cn/data/avatar/000/65/24/43_avatar_middle.jpg)高山小溪美人

> [TLHorse 发表于 2021-2-10 22:49](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=36860377&ptid=1369484)  
> 好的，你们推荐的东西，我一定学习学习，毕竟在下也不是什么高级人才，也是个小白  
> 没有 GUI 咱不怕，毕竟 g ...

https://github.com/chyyuu/os_course_info 这是总览，东西还是挺多的，期待你的后续![](https://avatar.52pojie.cn/data/avatar/001/32/18/04_avatar_middle.jpg)从 0 开始的小小怪

> [涛之雨 发表于 2021-2-10 19:19](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=36858123&ptid=1369484)  
> 好的作品需要时间来检测，而不是多少人查看和多少人去回复。  
> 与其满眼都是回复 “感谢楼主”，“学到了， ...

好，你这回复给了我不少信心啊，一定坚持 ![](https://avatar.52pojie.cn/data/avatar/000/74/07/98_avatar_middle.jpg) TLHorse 你以为我看的懂？![](https://avatar.52pojie.cn/data/avatar/001/32/18/04_avatar_middle.jpg)RemMai  

> [RemMai 发表于 2021-2-10 18:12](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=36857450&ptid=1369484)  
> 你以为我看的懂？

啊？似乎我这篇文章写得很失败  
我失望极了 查看 82 回复 4  
<table cellspacing="0"><tbody><tr><td><br></td></tr></tbody></table>![](https://avatar.52pojie.cn/data/avatar/001/34/37/12_avatar_middle.jpg)TLHorse 行吧，目前还看不出什么。静待后篇 ![](https://avatar.52pojie.cn/data/avatar/001/32/18/04_avatar_middle.jpg) ashi876

> [ashi876 发表于 2021-2-10 18:18](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=36857521&ptid=1369484)  
> 行吧，目前还看不出什么。静待后篇

好，我一定努力，  
其实后面那几部分源代码我都写出来了，就差写文章的事了 ![](https://avatar.52pojie.cn/data/avatar/000/37/91/37_avatar_middle.jpg) TLHorse 等着看后续，如果可以的话就该动手了 ![](https://avatar.52pojie.cn/data/avatar/001/34/19/99_avatar_middle.jpg) E 式丶男孩 期待后续~![](https://avatar.52pojie.cn/data/avatar/001/46/06/52_avatar_middle.jpg)_paopao 大哥加油![](https://static.52pojie.cn/static/image/smiley/laohu/laohu3.gif)，虽然我啥都看不懂，但还是想看看大哥的后续![](https://static.52pojie.cn/static/image/smiley/laohu/laohu33.gif)