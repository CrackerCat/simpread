> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [thief.one](https://thief.one/2017/05/19/1/)

发表于 2017-05-19   |   分类于 [安全工具](https://thief.one/categories/%E5%AE%89%E5%85%A8%E5%B7%A5%E5%85%B7/)   |   热度 ℃

> 动动指尖，弹指一挥间

　　搜索引擎是我日常工作中用得最多的一款工具，国内常用的搜索引擎包括 Baidu，sougou，bing 等。但我本篇要纪录的并不是这些常用的搜索引擎，而是信息安全从业人员必备的几款网络搜索引擎。本篇要介绍的搜索引擎包括：Shodan，censys，钟馗之眼，Google，FoFa，Dnsdb 等。介绍的内容主要是这几款搜索引擎的一些高级语法，掌握高级语法会让搜索结果更准确。  
  
_对搜索引擎语法有所遗忘的，本文可当参考，仅此而已_

### [](#Google搜索引擎 "Google搜索引擎")Google 搜索引擎

　　这里之所以要介绍 google 搜索引擎，是因为它有别于百度、搜狗等内容搜索引擎，其在安全界有着非同一般的地位，甚至专门有一名词为 google hacking 用来形容 google 与安全非同寻常的关系。

#### [](#google基本语法 "google基本语法")google 基本语法

Index of/　　使用它可以直接进入网站首页下的所有文件和文件夹中。  
intext:　　将返回所有在网页正文部分包含关键词的网页。  
intitle:　　将返回所有网页标题中包含关键词的网页。  
cache:　　搜索 google 里关于某些内容的缓存。  
define:　　搜索某个词语的定义。  
filetype:　　搜索指定的文件类型，如：.bak，.mdb，.inc 等。  
info:　　查找指定站点的一些基本信息。  
inurl:　　搜索我们指定的字符是否存在于 URL 中。  
Link:　　link:thief.one 可以返回所有和 thief.one 做了链接的 URL。  
site:　　site:thief.one 将返回所有和这个站有关的 URL。

+　　把 google 可能忽略的字列如查询范围。  
-　　把某个字忽略，例子：新加 - 坡。  
~　　同意词。  
.　　单一的通配符。  
*　　通配符，可代表多个字母。  
“”　　精确查询。

#### [](#搜索不同国家网站 "搜索不同国家网站")搜索不同国家网站

```
inurl:tw　　台湾

inurl:jp　　日本
```

#### [](#利用google暴库 "利用google暴库")利用 google 暴库

利用 goole 可以搜索到互联网上可以直接下载到的数据库文件，语法如下：  

```
inurl:editor/db/ 

inurl:eWebEditor/db/ 

inurl:bbs/data/ 

inurl:databackup/ 

inurl:blog/data/ 

inurl:\boke\data 

inurl:bbs/database/ 

inurl:conn.asp 

inc/conn.asp

Server.mapPath(“.mdb”)

allinurl:bbs data

filetype:mdb inurl:database

filetype:inc conn

inurl:data filetype:mdb

intitle:"index of" data
```

#### [](#利用goole搜索敏感信息 "利用goole搜索敏感信息")利用 goole 搜索敏感信息

利用 google 可以搜索一些网站的敏感信息，语法如下:  

```
intitle:"index of" etc

intitle:"Index of" .sh_history

intitle:"Index of" .bash_history

intitle:"index of" passwd

intitle:"index of" people.lst

intitle:"index of" pwd.db

intitle:"index of" etc/shadow

intitle:"index of" spwd

intitle:"index of" master.passwd

intitle:"index of" htpasswd

inurl:service.pwd
```

#### [](#利用google搜索C段服务器信息 "利用google搜索C段服务器信息")利用 google 搜索 C 段服务器信息

此技巧来自 [lostwolf](http://wolvez.club/)  

```
site:218.87.21.*
```

可通过 google 可获取 218.87.21.0/24 网络的服务信息。

### [](#shodan搜索引擎 "shodan搜索引擎")shodan 搜索引擎

shodan 网络搜索引擎偏向网络设备以及服务器的搜索，具体内容可上网查阅，这里给出它的高级搜索语法。  
地址：[https://www.shodan.io/](https://www.shodan.io/)

#### [](#搜索语法 "搜索语法")搜索语法

*   hostname：　　搜索指定的主机或域名，例如 hostname:”google”
*   port：　　搜索指定的端口或服务，例如 port:”21”
*   country：　　搜索指定的国家，例如 country:”CN”
*   city：　　搜索指定的城市，例如 city:”Hefei”
*   org：　　搜索指定的组织或公司，例如 org:”google”
*   isp：　　搜索指定的 ISP 供应商，例如 isp:”China Telecom”
*   product：　　搜索指定的操作系统 / 软件 / 平台，例如 product:”Apache httpd”
*   version：　　搜索指定的软件版本，例如 version:”1.6.2”
*   geo：　　搜索指定的地理位置，例如 geo:”31.8639, 117.2808”
*   before/after：　　搜索指定收录时间前后的数据，格式为 dd-mm-yy，例如 before:”11-11-15”
*   net：　　搜索指定的 IP 地址或子网，例如 net:”210.45.240.0/24”

以上内容参考：[http://xiaix.me/shodan-xin-shou-ru-keng-zhi-nan/](http://xiaix.me/shodan-xin-shou-ru-keng-zhi-nan/)

### [](#censys搜索引擎 "censys搜索引擎")censys 搜索引擎

censys 搜索引擎功能与 shodan 类似，以下几个文档信息。  
地址：[https://www.censys.io/](https://www.censys.io/)  

```
https://www.censys.io/certificates/help 帮助文档

https://www.censys.io/ipv4?q=  ip查询

https://www.censys.io/domain?q=  域名查询

https://www.censys.io/certificates?q= 证书查询
```

#### [](#搜索语法-1 "搜索语法")搜索语法

默认情况下 censys 支持全文检索。

*   23.0.0.0/8 or 8.8.8.0/24　　可以使用 and or not
*   80.http.get.status_code: 200　　指定状态
*   80.http.get.status_code:[200 TO 300]　　200-300 之间的状态码
*   location.country_code: DE　　国家
*   protocols: (“23/telnet” or “21/ftp”)　　协议
*   tags: scada　　标签
*   80.http.get.headers.server：nginx　　服务器类型版本
*   autonomous_system.description: University　　系统描述
*   正则

### [](#钟馗之眼 "钟馗之眼")钟馗之眼

钟馗之眼搜索引擎偏向 web 应用层面的搜索。  
地址：[https://www.zoomeye.org/](https://www.zoomeye.org/)

#### [](#搜索语法-2 "搜索语法")搜索语法

*   app:nginx　　组件名
*   ver:1.0　　版本
*   os:windows　　操作系统
*   country:”China”　　国家
*   city:”hangzhou”　　城市
*   port:80　　端口
*   hostname:google　　主机名
*   site:thief.one　　网站域名
*   desc:nmask　　描述
*   keywords:nmask’blog　　关键词
*   service:ftp　　服务类型
*   ip:8.8.8.8　　ip 地址
*   cidr:8.8.8.8/24　　ip 地址段

### [](#FoFa搜索引擎 "FoFa搜索引擎")FoFa 搜索引擎

FoFa 搜索引擎偏向资产搜索。  
地址：[https://fofa.so](https://fofa.so/)

#### [](#搜索语法-3 "搜索语法")搜索语法

*   title=”abc” 从标题中搜索 abc。例：标题中有北京的网站。
*   header=”abc” 从 http 头中搜索 abc。例：jboss 服务器。
*   body=”abc” 从 html 正文中搜索 abc。例：正文包含 Hacked by。
*   domain=”qq.com” 搜索根域名带有 qq.com 的网站。例： 根域名是 qq.com 的网站。
*   host=”.gov.cn” 从 url 中搜索. gov.cn, 注意搜索要用 host 作为名称。
*   port=”443” 查找对应 443 端口的资产。例： 查找对应 443 端口的资产。
*   ip=”1.1.1.1” 从 ip 中搜索包含 1.1.1.1 的网站, 注意搜索要用 ip 作为名称。
*   protocol=”https” 搜索制定协议类型 (在开启端口扫描的情况下有效)。例： 查询 https 协议资产。
*   city=”Beijing” 搜索指定城市的资产。例： 搜索指定城市的资产。
*   region=”Zhejiang” 搜索指定行政区的资产。例： 搜索指定行政区的资产。
*   country=”CN” 搜索指定国家 (编码) 的资产。例： 搜索指定国家 (编码) 的资产。
*   cert=”google.com” 搜索证书 (https 或者 imaps 等) 中带有 google.com 的资产。

高级搜索：

*   title=”powered by” && title!=discuz
*   title!=”powered by” && body=discuz
*   (body=”content=\”WordPress” || (header=”X-Pingback” && header=”/xmlrpc.php” && body=”/wp-includes/“) ) && host=”gov.cn”

### [](#Dnsdb搜索引擎 "Dnsdb搜索引擎")Dnsdb 搜索引擎

dnsdb 搜索引擎是一款针对 dbs 解析的查询平台。  
地址：[https://www.dnsdb.io/](https://www.dnsdb.io/)

#### [](#搜索语法-4 "搜索语法")搜索语法

DnsDB 查询语法结构为条件 1 条件 2 条件 3 …., 每个条件以空格间隔, DnsDB 会把满足所有查询条件的结果返回给用户.

##### [](#域名查询条件 "域名查询条件")域名查询条件

域名查询是指查询顶级私有域名所有的 DNS 记录, 查询语法为 domain:.  
例如查询 google.com 的所有 DNS 记录: domain:google.com.  
域名查询可以省略 domain:.

##### [](#主机查询条件 "主机查询条件")主机查询条件

查询语法: host:  
例如查询主机地址为 mp3.example.com 的 DNS 记录: host:map3.example.com  
主机查询条件与域名查询查询条件的区别在于, 主机查询匹配的是 DNS 记录的 Host 值

##### [](#按DNS记录类型查询 "按DNS记录类型查询")按 DNS 记录类型查询

查询语法: type:.  
例如只查询 A 记录: type:a  
使用条件: 必须存在 domain: 或者 host: 条件, 才可以使用 type: 查询语法

##### [](#按IP限制 "按IP限制")按 IP 限制

查询语法: ip:  
查询指定 IP: ip:8.8.8.8, 该查询与直接输入 8.8.8.8 进行查询等效  
查询指定 IP 范围: ip:8.8.8.8-8.8.255.255  
CIDR: ip:8.8.0.0/24  
IP 最大范围限制 65536 个

##### [](#条件组合查询的例子 "条件组合查询的例子")条件组合查询的例子

查询 google.com 的所有 A 记录: google.com type:a

本文将会持续补充一些内容……

### [](#传送门 "传送门")传送门

[【渗透神器系列】nc](http://thief.one/2017/04/10/1/)  
[【渗透神器系列】nmap](http://thief.one/2017/05/02/1/)  
[【渗透神器系列】Fiddler](http://thief.one/2017/04/27/1)  
[【渗透神器系列】WireShark](http://thief.one/2017/02/09/WireShark%E8%BF%87%E6%BB%A4%E8%A7%84%E5%88%99/)

[![](https://thief.one/uploads/wechat-qcode.jpg)](https://thief.one/uploads/wechat-qcode.jpg)

欢迎您扫一扫上面的微信公众号，订阅我的博客！