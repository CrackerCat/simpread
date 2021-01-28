> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265661.htm)

该笔记仅自己用于记录学习内容，网络渗透知识基本为 0。望大佬轻喷。  
该练习参考 [https://unit630.com/2020/08/23/malware-traffic-analysis-exercise-1/](https://unit630.com/2020/08/23/malware-traffic-analysis-exercise-1/)  
首先打开提供的 PACP 包，可以看到以下内容。我们可以看到有大量不同协议请求的流量包。  
![](https://bbs.pediy.com/upload/attach/202101/784955_UP722E9ENWZV87Q.png)

 

我们如果想要直接通过肉眼观察流量包找到被感染的 VM 主机是很困难的。但是我们可以根据病毒的不同行为设置不同的流量策略过滤数据包获取我们想要的信息。往往恶意软件在感染了目标主机后会有反向链接的请求行为，我们可以利用这一点设置请求过滤查看被感染主机的 IP。可以看到感染主机的 ip 为：172.16.165.165。

```
http.request

```

![](https://bbs.pediy.com/upload/attach/202101/784955_N3M99HW7GA9T4R8.png)  
接下来，我们想要获取被感染主机的主机名。在作者提供的这种环境下，黑客使用的是 DHCP 自动解析协议。而该协议会将指定服务器发送数据包，其中申请动态 ip 的主机名作为数据包含在 DHCP 请求的 HostName 字段中。  
首先过滤下 DHCP 协议。  
![](https://bbs.pediy.com/upload/attach/202101/784955_ZZ87N28EF9FMGDM.png)  
双击查看该数据包包解析，找到 HostName 字段。可以看到主机名是：K34EN6W3N-PC  
![](https://bbs.pediy.com/upload/attach/202101/784955_G39FEGFMYD8X8AC.png)  
获取被感染主机的 MAC 地址，只需要过滤 ARP 协议。查看发起 ARP 广播的流量包即可。可以看到感染主机的 MAC 地址为 f0:19:af:02:9b:f1  
![](https://bbs.pediy.com/upload/attach/202101/784955_PWE7XMC3BVBFA2C.png)  
在获取了被感染主机信息后，我们要确定黑客 C2 服务器 ip / 被入侵的服务器 ip 地址。在当前的情况下，我们只需要设置好过滤条件，主动发起请求为感染主机的 ip，向下翻阅可以看到大量的 DNS 请求，这些 DNS 请求可以帮助我们有效缩小范围，我们可以看到以下解析的网址域名，通过域名进行筛选。

```
ip.src=172.16.165.165&&dns

```

![](https://bbs.pediy.com/upload/attach/202101/784955_ZJNPZABRN94QEX2.png)  
接下来，我们需要查看 Http 协议相关的数据包进行第二次筛选。因为感染主机很可能已经与入侵的服务器建立通讯已经浏览过具体内容。所以我们在查看 Http 相关请求的时候要格外注意 "referer" 字段。该字段为引荐网址，用于通过某一域名连接到某一其他域名。由此下图基本可以确定，感染主机通过搜索引擎访问到被感染服务器。  
![](https://bbs.pediy.com/upload/attach/202101/784955_P6VW4NWBYXDRX3G.png)  
接下来我们可以通过引用数据包获取搜索引擎真实访问的服务器域名：www.ciniholland.nl。  
![](https://bbs.pediy.com/upload/attach/202101/784955_9NTCAYYM6FTRX4B.png)  
当我们想要找到真正提供下载黑客工具的网址时，我们需要仔细浏览从访问向后的所有数据包。获取到最终的黑客工具提供网址 ip 为：37.200.69.143。  
![](https://bbs.pediy.com/upload/attach/202101/784955_ZM9DHKR8CZ3ZNAR.png)  
首先根据请求的 bing 搜索访问域名 www.ciniholland.nl。  
![](https://bbs.pediy.com/upload/attach/202101/784955_GA6J3AVKJGGWP4S.png)  
访问的 ip 为 adultbiz.in 域名下的 jquery.php 文件。一般被感染的网站会有外连其他网站行为。  
![](https://bbs.pediy.com/upload/attach/202101/784955_8UCEGE6CFGYU8VB.png)  
接下来会发现访问域名为 24corp-shop.com。  
![](https://bbs.pediy.com/upload/attach/202101/784955_C8GJMSXQ3HRU3SR.png)  
该域名最终访问黑客工具提供站点为 stand.trustandprobaterealty.com。  
![](https://bbs.pediy.com/upload/attach/202101/784955_3QV8ZBN5YS4V3D7.png)  
我们可以看到 24corp-shop.com 是一个重定向链接，因为该链接请求一共不超过 3 个。  
![](https://bbs.pediy.com/upload/attach/202101/784955_AYEM2RKFRCPP4Z5.png)  
接下来需要查看对应网站向感染机器发送的请求。选择文件 -> 导出对象 ->HTTP 协议。  
![](https://bbs.pediy.com/upload/attach/202101/784955_3QQ3VNBFMJHUER9.png)  
由于我们只需要 "stand"，在文本过滤器中过滤 stand 字符串即可看到漏洞利用相关的请求。  
![](https://bbs.pediy.com/upload/attach/202101/784955_QJZY95KJE6MYJAT.png)  
设置过滤条件找到对应包数据。  
![](https://bbs.pediy.com/upload/attach/202101/784955_BNK3JM6AWRVZMER.png)  
查看内容有可以字符串，可以看到疑似漏洞利用 application/x-shockwave-flash、application/x-msdownload、application/java-archive。  
![](https://bbs.pediy.com/upload/attach/202101/784955_GNQJFMCJ28FKUWV.png)  
![](https://bbs.pediy.com/upload/attach/202101/784955_Y6DYSJ7ZWSYGYHU.png)  
application/x-msdownload 只是 MINI 应用程序一部分，所以只用了两种漏洞。  
![](https://bbs.pediy.com/upload/attach/202101/784955_M2BQT9V9SKFJBXC.png)  
我们还要确定有效载荷被传递的次数。我们通过 google 可以看到改 MIME 类型通常用于 DLL 文件的编码，所以判断该流量包是用于传递恶意载荷的 PE 程序的。  
![](https://bbs.pediy.com/upload/attach/202101/784955_UV62GMWUKUZRDS4.png)  
通过文件对象过滤，可以看到一共传播了 3 次。  
![](https://bbs.pediy.com/upload/attach/202101/784955_XQTU9ZP658NSQYV.png)  
由于确定是恶意域名向感染机器发送恶意载荷，将 ip 源地址设置成黑客恶意 ip 进行过滤即可。  
![](https://bbs.pediy.com/upload/attach/202101/784955_JF35A7TMRZ34CVE.png)  
将病毒上传 VT 可以看到 VT 检测出两个漏洞。VT 只是提供一个方向，具体漏洞需要自己对流量以及程序调试分析。  
![](https://bbs.pediy.com/upload/attach/202101/784955_HTB25RD35KJ7AB2.png)  
我们可以根据 VT 提供的 Snort 告警查看漏洞利用工具包。可以看到 EK（EXPLOIT-KIT 漏洞利用工具包）签名为 Rig exploit kit。  
![](https://bbs.pediy.com/upload/attach/202101/784955_VECPJP9F557VKU7.png)  
查看 Suricata 告警二次确认，漏洞利用工具包是 RIG EK。  
![](https://bbs.pediy.com/upload/attach/202101/784955_62378YSUH4BS69X.png)  
接下来我们还需要知道被感染的哪个网页中存在重定向链接。首先我们需要对网页源代码进行代码审计。  
![](https://bbs.pediy.com/upload/attach/202101/784955_HT62SU8NE5Q74NQ.png)  
根据流追踪可以看到在 JS 函数中嵌入了重定向链接 http://24corp-shop.com。  
![](https://bbs.pediy.com/upload/attach/202101/784955_UYZFKTMJMP7N854.png)  
这与我们 google 找到的感染方式一致 [https://labs.sucuri.net/signatures/malwares/js-malware-hidden-iframe-006/](https://labs.sucuri.net/signatures/malwares/js-malware-hidden-iframe-006/)  
这种方式被称为 Hidden IFRAME（隐藏 IFRAME 表单）  
![](https://bbs.pediy.com/upload/attach/202101/784955_4BACQXBCMBRB4JC.png)  
![](https://bbs.pediy.com/upload/attach/202101/784955_Z532QPCVHD68VUB.png)  
接下来可以根据 VT 提供的图形功能结合 pacp 看到完整的网络关联。  
![](https://bbs.pediy.com/upload/attach/202101/784955_U5NZ9B6FPRC3P37.png)  
通过 VT 获取其他用户提供的更多信息。  
![](https://bbs.pediy.com/upload/attach/202101/784955_6CUFE2MUG4TWBPG.png)  
![](https://bbs.pediy.com/upload/attach/202101/784955_R9RA7UDTEEUN54Y.png)  
关联到 flash 样本。  
![](https://bbs.pediy.com/upload/attach/202101/784955_ZY42CP6FX5SV9ZR.png)  
相关 java 漏洞的样本。  
![](https://bbs.pediy.com/upload/attach/202101/784955_TCKY4CCFZGPNFFY.png)

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)

最后于 1 天前 被独钓者 OW 编辑 ，原因：

上传的附件：

*   [2014-11-16-traffic-analysis-exercise.pcap.zip](attach-download-233512-4105531e4c59c611b5c4b975508ac7aa@7ecJZwFV5V_2Ba8lCvfSUU4A_3D_3D.htm) （2.03MB，4 次下载）
*   [2014-11-16-traffic-analysis-exercise-answers.p.zip](attach-download-233513-4105531e4c59c611b5c4b975508ac7aa@7ecJZwFV5V_2Ba8lCvfSUU4A_3D_3D.htm) （824.67kb，2 次下载）