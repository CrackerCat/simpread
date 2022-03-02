> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271696.htm)

> [原创] 某 RCE 漏洞复现

漏洞复现
====

基础环境
----

测试服务器：Win7 虚拟机  
测试服务器 IP：192.168.220.134  
软件版本：SunloginClient_11.0.0.33162_X64  
EXP 下载地址： https://github.com/Mr-xn/sunlogin_rce （感谢开源作者）

复现流程
----

在 Win7 虚拟机里执行 SunloginClient_11.0.0.33162_X64.exe  
![](https://bbs.pediy.com/upload/attach/202203/598931_YZUPU5FVB7DJCMP.png)  
查看对外开放端口，这里为 49218，这个端口不是固定的，重启程序会变。  
![](https://bbs.pediy.com/upload/attach/202203/598931_KMQGYGNBRTYSCKA.png)  
配合查找的命令

```
netstat -ano | findstr LISTEN
tasklist | findstr SunloginClient

```

测试 exp，命令执行成功，是 system 权限  
![](https://bbs.pediy.com/upload/attach/202203/598931_EB6W63TR5D88CYY.png)

漏洞原理分析
======

EXP 源码分析
--------

其实直接看 exp 源码，也能猜个差不多，莫过于  
1. 授权认证出了问题，任意用户可以获得访问令牌；  
2. 存在命令注入问题  
![](https://bbs.pediy.com/upload/attach/202203/598931_9YE4W6788DVZFC6.png)  
对导致命令执行的 url 进行 url 解码可以看的更清楚些  
![](https://bbs.pediy.com/upload/attach/202203/598931_AMXHC97XSY58PKH.png)

流量分析
----

执行 exp，并使用 wireshark 抓包  
抓包时可以使用 bpf 语句过滤掉无关的报文，如

```
host 192.168.220.134 and tcp port 49168

```

![](https://bbs.pediy.com/upload/attach/202203/598931_26CNACM5WZYWJJT.png)  
对抓包结果进行分析  
请求令牌，存在未授权访问的问题  
![](https://bbs.pediy.com/upload/attach/202203/598931_4QPEWHWV26CXE4V.png)  
命令执行，存在命令注入的问题  
![](https://bbs.pediy.com/upload/attach/202203/598931_QT7RZ3QG9YTZ34X.png)

为了定位命令执行的关键代码位置
---------------

### 行为分析

使用 Procmon 对程序进行行为分析，找到命令执行的关键函数，CreateProcessA。其实不用找，大概也能猜出来，可以把常见造成命令执行的函数都下个断点，断下来之后再进行判断。  
![](https://bbs.pediy.com/upload/attach/202203/598931_THMZ8ZY7FXNPHJR.png)  
![](https://bbs.pediy.com/upload/attach/202203/598931_H6PPYUBFZJNJQGU.png)

### 动态调试

小技巧，使用 PsExec 得到 system 权限

```
PsExec64.exe -i -s cmd

```

以 system 权限启动 x64dbg，以方便附加调试目标进程  
![](https://bbs.pediy.com/upload/attach/202203/598931_NUHJMN68D9W24Y2.png)  
在调试时需要隐藏调试器，否则在调试过程中会报异常

```
调试->高级->隐藏调试器（PEB）

```

对 CreateProcessA 函数下断点，然后执行 exp 触发断点  
![](https://bbs.pediy.com/upload/attach/202203/598931_GGZ2FTYM2BEXATU.png)

### 静态分析

有个 upx 壳，使用 “upx -d” 直接脱掉。脱掉后的程序直接运行的话，还是会报错，没有探究原因，不过 ida 可以正常分析了。  
![](https://bbs.pediy.com/upload/attach/202203/598931_HXQ75PHBV8KAXBA.png)

 

根据动态调试的结果，可以很快定位到关键代码位置。找到 URL 路由，这可以用来分析其他 api 功能。另外，发现除了 exp 里提到的 ping 可以导致命令执行，nslookup 也是可以的，可以自行编写脚本测试。  
![](https://bbs.pediy.com/upload/attach/202203/598931_GR958H7CN3YZ53X.png)  
![](https://bbs.pediy.com/upload/attach/202203/598931_EHCUCB3QPEE925D.png)

渗透测试
====

编写 Goby 脚本
----------

### [](#配置“漏洞信息”)配置 “漏洞信息”

![](https://bbs.pediy.com/upload/attach/202203/598931_3EZGKHTFJWCREK3.png)  
这里的指纹信息十分关键，Goby 在扫描的时候，会先扫描资产，这个指纹就是用来判断资产种类的，匹配上指纹之后，才会打对应的 poc。指纹的好坏，直接决定了扫描速度。  
Goby 语法和 fofa 是一致的，这里匹配的是 "GET /" 的应答，因为 Goby 在做资产探测的时候不会探测太多 URL。

```
body="Verification failure" && body="false" && header="Cache-Control: no-cache" && header="Content-Length: 46" && header="Content-Type: application/json"

```

### [](#配置“扫描测试”)配置 “扫描测试”

扫描测试有两个步骤，1. 获得访问令牌 CID；2. 带令牌执行命令。

#### Test1

访问获取令牌的 URL  
![](https://bbs.pediy.com/upload/attach/202203/598931_S8PM5PFZYMB8CHH.png)  
指纹判断，如果访问成功，则根据正则提取 CID  
![](https://bbs.pediy.com/upload/attach/202203/598931_9TYZE6QHF47N7K9.png)  
可以用 python 快速测试正则

```
import re
 
a=r'''{"__code":0,"enabled":"1","verify_string":"ysDRmcQu37usMmdA60fniHTv3cJzlWHz","code":0}'''
filter=re.compile(r'''"verify_string":"(\S+?)",''')
res = filter.findall(a)
print(res)

```

#### Test2

带 Cookie 访问命令执行的 URL，上一步设置的变量 CID 可以套三个大括号来使用，即 {{{CID}}}  
![](https://bbs.pediy.com/upload/attach/202203/598931_SGDGMCPNNX78KBZ.png)

 

![](https://bbs.pediy.com/upload/attach/202203/598931_P5KVN3YWEMVGV6T.png)

### 测试效果

测试效果，发现漏洞。  
![](https://bbs.pediy.com/upload/attach/202203/598931_KVNHD6NVXBYVGNP.png)  
测试过程可以使用 wireshark 抓包来辅助 poc 编写，也可以参考老的 poc 脚本，其目录在  
goby-win-x64-1.8.293\golib\exploits\user  
或者通过 poc 管理导出来也是可以的。  
![](https://bbs.pediy.com/upload/attach/202203/598931_N6TC74J7G5R3GYN.png)

态势感知
====

流量监测
----

### 尽可能多的生成多种形式的攻击流量

考虑合理变形，尽可能多的生成多种形式的攻击流量，以便用来测试检测规则。  
![](https://bbs.pediy.com/upload/attach/202203/598931_D6UBDXJRW4DFVAH.png)

 

打 poc 的同时，用 wireshark 抓包，这里得到攻击流量包 sunlogin_rce_multi_payload.pcap  
![](https://bbs.pediy.com/upload/attach/202203/598931_2MQMQAX5269GWK7.png)

### 编写规则并测试

测试环境为 Kali-Linux-2021.2-vmware-amd64，suricata 版本为 6.0.4 。

 

根据 payload 编写 pcre 正则

```
/\/check?.*cmd[\s]*?=(?:ping|nslookup).*?(?:\.\.\/|\.\.\\)/

```

正则解释

```
[\s]*?=             匹配任意个空白符，非贪婪匹配，匹配最近的“=”
(?:ping|nslookup)    匹配 ping 或 nslookup，?:表示不获取匹配结果
(?:\.\.\/|\.\.\\)       匹配 ../ 或 ..\

```

测试 pcre 正则

```
apt install pcre2-utils
pcre2test

```

![](https://bbs.pediy.com/upload/attach/202203/598931_BW2E6AE6RGRXDDW.png)

 

编写 suricata 规则  
（此规则可以应对正常攻击和部分变形，但依然存在被绕过的可能。规则写严了容易漏报，写松了容易误报，另外还应该要考虑报文分片传输、丢包的问题。）  
vim /etc/suricata/rules/test.rules

```
alert http any any -> any any (msg:"CNVD-2022-10270 SunloginClient RCE"; pcre:"/\/check?.*cmd[\s]*?=(?:ping|nslookup).*?(?:\.\.\/|\.\.\\)/Ui"; classtype:attempted-admin; sid:22022801; rev:2;)

```

/Ui 里的 U 表示在标准 uri 上进行 pcre 匹配（类似于 http_uri，相当于 URL 解码后再匹配），i 表示大小写敏感。

 

vim /etc/suricata/suricata.yaml

```
rule-files:
  #- suricata.rules
  - test.rules

```

测试 suricata 规则，5 条攻击报文触发了 5 次告警，测试成功。

```
suricata -r sunlogin_rce_multi_payload.pcap

```

![](https://bbs.pediy.com/upload/attach/202203/598931_8XZVGPW4TWNM8KK.png)

 

流量监测相关资料  
https://suricata.readthedocs.io/en/suricata-6.0.0/rules/  
https://rocknsm.io/  
https://github.com/arkime/arkime

终端监测
----

windows 下的事件监控可以用 sysmon，Linux 下则可以用 auditd，然后借助 wazuh 来管理日志。

```
Sysmon64.exe -i

```

![](https://bbs.pediy.com/upload/attach/202203/598931_GXJ58E4JPSHJ4V9.png)

 

![](https://bbs.pediy.com/upload/attach/202203/598931_FZUDWUW4T49EMTK.png)

 

终端监测相关资料  
https://www.sysgeek.cn/sysmon/  
https://www.maliciouskr.cc/2018/11/15 / 使用 OSSEC 构建主机层入侵检测 /  
https://blog.csdn.net/single7_/article/details/110038117

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#漏洞分析](forum-150-1-153.htm) [#漏洞利用](forum-150-1-154.htm) [#Windows](forum-150-1-160.htm)