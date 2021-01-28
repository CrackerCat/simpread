> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265662.htm)

原文链接：https://unit630.com/2020/08/24/malware-traffic-analysis-exercise-2/  
第二个流量数据分析开始之前，我们要清楚私有 IP 地址范围。wiki 定义的私有 ip 地址范围如下图。  
![](https://bbs.pediy.com/upload/attach/202101/784955_WCNEURU7YDVQRUG.png)  
![](https://bbs.pediy.com/upload/attach/202101/784955_CFARQHQB89BXMVV.png)  
我们再观察 pacp 数据包，可以看到有存在于 172.16.0.0/12 范围内的 ip。可以通过该范围缩小感染机器 ip 范围。在 ip 筛选器中设置网段作为筛选条件，可以看到过滤出的 ip 为 172.16.165.132 和 172.16.165.2，然后继续通过观察通讯相关协议可以排除 DNS 相关的 ip。确定感染 ip 为 172.16.165.132。

```
ip.src==172.16.0.0/12 or ip.src==192.168.0.0/16

```

![](https://bbs.pediy.com/upload/attach/202101/784955_S2QJQKXYA4C9XH6.png)  
还有另一种更简单的方法确定，打开统计菜单中的协议分级窗口。  
![](https://bbs.pediy.com/upload/attach/202101/784955_5W8HPCDQ7G8FUFJ.png)  
打开的窗口显示如下，我们只需查看用户数据报协议（UDP，User Datagram Protocol）下使用了哪些协议。作者提供的 PACP 包中存在 NTP 协议。  
![](https://bbs.pediy.com/upload/attach/202101/784955_CVDUCJP4PMJKA77.png)  
过滤 NTP 协议即可确定感染主机 ip。  
![](https://bbs.pediy.com/upload/attach/202101/784955_F5VGS8YZ993DAUQ.png)  
我们查看维基百科对 [UDP 协议](https://en.wikipedia.org/wiki/User_Datagram_Protocol)的属性，发现其中包括我们本例子中所需的 NTP 协议  
![](https://bbs.pediy.com/upload/attach/202101/784955_M75AHPBJ2FED858.png)  
继续详细查看 [NTP 协议](https://en.wikipedia.org/wiki/Network_Time_Protocol)，可以发现该协议用于 UDP 建立的客户端 - 服务器模型的被动时间校准。所以如果存在 UDP 协议默认都会使用 NTP，所以 NTP 协议成为了判断反向通讯的一个过滤条件。  
![](https://bbs.pediy.com/upload/attach/202101/784955_BEGH2K7BAJVAXWC.png)  
接下来需要解析感染机器 MAC 地址，可以随便获取一个数据包查看 MAC。发现 MAC 地址为 00:0c:29:c5:b7:a1。  
![](https://bbs.pediy.com/upload/attach/202101/784955_ZZRAK5HATMZ3GBH.png)  
根据感染 ip 和 dns 协议过滤出所有解析的域名，发现解析出了很多域名，接下来需要根据 google 对域名筛选。筛选出正常域名和可疑域名。然后同第一节方法相同，查看 hijinksensue.com。  
![](https://bbs.pediy.com/upload/attach/202101/784955_SRP5XUN2Q52HQF5.png)  
依次对 DNS 解析出的域名进行过滤，在筛选器中输入以下内容进行过滤。

```
http.host==hijinksensue.com

```

可以看到该网址是通过 google.co.uk 浏览器搜索引擎打开。该域名被识为可疑域名，如果对每个域名进行筛查后实际上也是如此。  
![](https://bbs.pediy.com/upload/attach/202101/784955_F262UUH2RWWRUDA.png)  
定位提供恶意 EK 工具包的地址，需要文件过滤器筛选 PE 提供者的域名。首先点击内容类型，将同类型归类到一起。然后查看内容类型是否与第三方组件（漏洞利用）或 PE 相关的类型。这里我们注意到 application/octet-stream 类型，该类型一般表示 Wireshark 无法识别的文件类型，所以默认以数据流标记。  
![](https://bbs.pediy.com/upload/attach/202101/784955_4DUARF3EQVKSZH9.png)  
找到该数据包可以看到熟悉的 MZ，该数据包中数据部分存放的是 PE 文件格式的数据。所以确定为黑客工具提供站点。ip 为 37.157.6.226，域名为 h.trinketking.com 和 g.trinketking.com（该域名被 dns 解析没有立即使用，后续分析会展示该域名被使用的方式）。  
![](https://bbs.pediy.com/upload/attach/202101/784955_SDEEFAK5MK4ZXFZ.png)  
既然确定了恶意流量包，那么接下来我们可以将流量包中的 PE 样本导出到本地分析。右键 Data 选项，选择导出分组字节流。  
![](https://bbs.pediy.com/upload/attach/202101/784955_AHU4UR7J6PBTH9Q.png)  
随便给导出的数据起个名字，我这里命名为 1.exe。  
![](https://bbs.pediy.com/upload/attach/202101/784955_YSJ8B3XJHHZJ9GV.png)  
查看桌面可以看到导出的恶意 PE 程序。  
![](https://bbs.pediy.com/upload/attach/202101/784955_ETN7XGGRKDCHSNN.png)  
使用 die 查壳为无壳程序。  
![](https://bbs.pediy.com/upload/attach/202101/784955_SKK2GBX7N5U2QKH.png)  
查看 PE 熵值，发现区段完整并存在附加数据。  
![](https://bbs.pediy.com/upload/attach/202101/784955_GG8BDM4MUMH752A.png)  
通过火绒剑观察，行为完整。证明从流量中提取出的恶意数据是完整的。  
![](https://bbs.pediy.com/upload/attach/202101/784955_9SQ8CEBCMCRCT47.png)  
![](https://bbs.pediy.com/upload/attach/202101/784955_DVMQH26WUTURV2U.png)  
接下来我们继续分析感染网站是如何重定向到黑客工具下载站的。首先选择文件对象导出 ->HTTP 文件对象过滤，根据域名简单判断出网站内容。  
![](https://bbs.pediy.com/upload/attach/202101/784955_AY6NK8Z8HBV3P3B.png)

 

最终筛选出可疑的域名为：static.charlotteretirementcommunities.com，一般情况下网站域名都是简洁概括内容的单词缩写或首字母拼写，而该域名介绍过于详细反倒引起了注意。  
![](https://bbs.pediy.com/upload/attach/202101/784955_DPTM2EX9QFB6ZF3.png)  
根据文件过滤器我们可以得知被感染的内网机器向该网站请求了一个 “text/javascript” 类型文件，我们根据数据包的流跟踪内容可以看到以下内容。可以看到数据部分定义了一个 "main_request_data_content" 变量，该变量中的数据明显是被编码过或以某种加密形式处理过的。  
![](https://bbs.pediy.com/upload/attach/202101/784955_HM9QFQFWDE7HSQG.png)  
traffic-analysis.net 网站上有一篇文章可以提取出该字符串解码后的真实内容。我们仿照写下 Python 脚本提取字符串，我这里直接抄作者的了。

```
EncryptStr = '(6i8h(74$X7o4w(70(z3a)2fY_2f)6H7U@K2es.X74k_O72x$P69Y;R6e=R6b;6v5j!74m;H6b=69)L6QeP_M6S7_2he@63R=6vfJ;6d;i3a,L3P5@y31g.L34J)33Z(39w$t2fw!T63(6fr(r6peV.P7X3,7P5t,6dx_z65,7V2J@Z2f)6V5(w6dJ$7U0!74W;p79q$s2f=K6k2x_69n=7o2=G64_73;Z2pe;Z70.68_7N0@3f(R7O7q,6Q9;S6Oej(K74(t65,7O2k$t3d,3i3'
DecryptStr = []
for i in EncryptStr:
    if i in '0123456789abcdef':
        DecryptStr.append(i)
 
print(''.join(DecryptStr))

```

得到的数据如下。  
![](https://bbs.pediy.com/upload/attach/202101/784955_X42YN44V9W2HVXP.png)

```
687474703a2f2f672e7472696e6b65746b696e672e636f6d3a35313433392f636f6e73756d65722f656d7074792f62697264732e7068703f77696e7465723d33

```

可以使用 010Editor 显示 16 进制对应的字符串。  
![](https://bbs.pediy.com/upload/attach/202101/784955_W7H49HYMGDWH2A2.png)  
也可以使用 notepad++ 将数据转换成 Ascii。  
![](https://bbs.pediy.com/upload/attach/202101/784955_F797UT27RG5YKS9.png)  
转换后可以获得域名如下，正是文件导出对象过滤没有显示的域名 g.trinketking.com。由此可以确定 static.charlotteretirementcommunities.com 域名实际上是做了重定向的操作。重定向的 ip 为 50.87.149.90 为提供 EK 工具包的黑客服务器。

```
http://g.trinketking.com:51439/consumer/empty/birds.php?winter=3

```

确定了一切相关信息后，需要我们上传 pacp 包到 VT 等沙箱鉴别是否存在漏洞利用特征。发现 Sweet Orange（甜橙）漏洞利用工具包特征。  
![](https://bbs.pediy.com/upload/attach/202101/784955_ZD9T7JDSSDTSRBB.png)  
将恶意样本从文件导出后，提取文件 Hash。

```
MD5: 1408275C2E2C8FE5E83227BA371AC6B3
SHA1: DAC3D479CE4AF6D2FFD5314191E768543ACFE32D
CRC32: 1E47AACF

```

![](https://bbs.pediy.com/upload/attach/202101/784955_CTP7SP87U5HN22E.png)  
我们还可以通过 VT 的嗅探中找到漏洞的流量特征，利用的漏洞为 cve-2014-6332。  
![](https://bbs.pediy.com/upload/attach/202101/784955_8Z5V3HPVGYSJC2G.png)

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)

最后于 1 天前 被独钓者 OW 编辑 ，原因：

上传的附件：

*   [2014-11-23-traffic-analysis-exercise.pcap.zip](attach-download-233510-4105531e4c59c611b5c4b975508ac7aa@wI_2FoQfHf2v6xSzqHD3315Q_3D_3D.htm) （1.96MB，2 次下载）
*   [2014-11-23-traffic-analysis-exercise-answers.p.zip](attach-download-233511-4105531e4c59c611b5c4b975508ac7aa@wI_2FoQfHf2v6xSzqHD3315Q_3D_3D.htm) （98.44kb，2 次下载）