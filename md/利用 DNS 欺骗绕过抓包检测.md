> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1842061-1-1.html)

> [md]# 利用 DNS 欺骗绕过抓包检测# 昨天帖子被管理删了，现在重发# 情况说明 IOS 某软件的代 - 理检测，无论使用 WiFi 的代 - 理还是威屁~ 恩都会显示如下图，并且点击后 ... 利用 DNS 欺骗绕过抓包......

![](https://avatar.52pojie.cn/data/avatar/001/26/73/53_avatar_middle.jpg)k452b

利用 DNS 欺骗绕过抓包检测
===============

昨天帖子被管理删了，现在重发
==============

情况说明
====

IOS 某软件的代 - 理检测，无论使用 WiFi 的代 - 理还是威屁~ 恩都会显示如下图，并且点击后自动退出软件。

我们通过某种方式知道了软件的 API 服务器为`v3-api.china-****.cn`，我们需要抓包的就是这个网址。

![](https://attach.52pojie.cn//forum/202310/08/023410nm1qks3mibir4vig.png?l)

#### DNS 欺骗示例图

![](https://attach.52pojie.cn//forum/202310/08/023617q3b4zk4prrrrlr4s.png?l)

客户端使用自建的 DNS 服务器，DNS 服务器返回我们反向代 - 理后的虚假的网站。

目录
==

`需要python环境`

1. 自建 DNS 服务器

2. 自建反向代 - 理网站

3. 反向代 - 理网站自签名 SSL 证书

4. 手机上的设置

1. 自建 DNS 服务器
=============

获取本机 IP
-------

我们先用 cmd 的`ipconfig`命令获取本机的内网地址

![](https://attach.52pojie.cn//forum/202310/08/023645aozzr629o9re9pye.png?l)

搭建 DNS 服务器
----------

简单写了个 python 的代码，复制以下代码然后保存成`.py`文件，并修改需要代 - 理的网站和内网 IP。

```
import logging
import socket
from dns.resolver import Resolver
from dnslib import DNSRecord, QTYPE, RD, SOA, DNSHeader, RR, A

class DNSServe(object):
    mydns = {}
    def __init__(self,dns1="119.29.29.29",dns2="114.114.114.114"):
        self.dns_resolver = Resolver()
        self.dns_resolver.nameservers = [dns1, dns2]
        self.udp_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.udp_sock.bind(('', 53))
        LOG_FORMAT = "%(asctime)s - %(levelname)s - %(message)s"
        logging.basicConfig(level=logging.DEBUG, format=LOG_FORMAT)
        logging.info('dns server is started')
    def start(self):
        while True:
            message, address = self.udp_sock.recvfrom(8192)
            self.dns_handler(self.udp_sock, message, address)
    def add_dns(self,domain, ip):
        self.mydns.update({domain: ip})
        logging.info(f'添加记录成功\t"{domain}":"{ip}"')
    def get_ip_from_domain(self,domain):
        domain = domain.lower().strip()
        try:
            if domain in self.mydns:
                dns = self.mydns.get(domain)
            else:
                dns = self.dns_resolver.resolve(domain, 'A')[0].to_text()
            return dns
        except:
            return None
    def reply_for_not_found(self,income_record):
        header = DNSHeader(id=income_record.header.id, bitmap=income_record.header.bitmap, qr=1)
        header.set_rcode(0)  # 3 DNS_R_NXDOMAIN, 2 DNS_R_SERVFAIL, 0 DNS_R_NOERROR
        record = DNSRecord(header, q=income_record.q)
        return record
    def reply_for_A(self,income_record, ip, ttl=None):
        r_data = A(ip)
        header = DNSHeader(id=income_record.header.id, bitmap=income_record.header.bitmap, qr=1)
        domain = income_record.q.qname
        query_type_int = QTYPE.reverse.get('A') or income_record.q.qtype
        record = DNSRecord(header, q=income_record.q, a=RR(domain, query_type_int, rdata=r_data, ttl=ttl))
        return record
    def dns_handler(self,s, message, address):
        try:
            income_record = DNSRecord.parse(message)
        except:
            logging.error('from %s, parse error' % address)
            return
        try:
            qtype = QTYPE.get(income_record.q.qtype)
        except:
            qtype = 'unknown'
        domain = str(income_record.q.qname).strip('.')
        info = '%s -- %s, from %s' % (qtype, domain, address)
        if qtype == 'A':
            ip = self.get_ip_from_domain(domain)
            if ip:
                response = self.reply_for_A(income_record, ip=ip, ttl=60)
                s.sendto(response.pack(), address)
                return logging.info(info)
        # at last
        response = self.reply_for_not_found(income_record)
        s.sendto(response.pack(), address)
        logging.info(info)

if __name__ == '__main__':
    server = DNSServe()
    # 添加DNS记录
    server.add_dns('baidu.com', '2.2.2.2')
    server.add_dns('v3-api.china-****.cn', '192.168.8.102')
    # 开始启动
    server.start()

```

只要在`server.add_dns('baidu.com', '2.2.2.2')`下面增加代码就行。

`v3-api.china-****.cn`就是需要代 - 理软件的 API 服务器，`192.168.8.102`是上面获取的内网地址。

启动 DNS 服务器后，本机电脑使用 cmd 的 nslookup 命令测试一下 DNS 服务器是否正常运行。

##### 启动 DNS 服务器

![](https://attach.52pojie.cn//forum/202310/08/023725k5m6iucli8qqlnnp.png?l)

##### 使用 cmd 的 nslookup 命令指定自建 dns 服务器获取的记录

![](https://attach.52pojie.cn//forum/202310/08/023821nz4h7uz6piuz25xz.png?l)

2. 自建反向代 - 理网站
==============

安装依赖
----

直接下载我提供的包，解压后在目录下打开 cmd 使用`pip install -r requirements.txt`安装依赖包

直接打开`app.py`运行成功后打开 [http://127.0.0.1](http://127.0.0.1)，如果成功打开，那就代表反向代 - 理成功了。

[下载地址](https://ldz2020.lanzoue.com/i1F3N1b6erej)

##### 运行 app.py

![](https://attach.52pojie.cn//forum/202310/08/023904qdm58pddekd5ad15.png?l)

修改配置
----

我们需要修改项目中的`app.py`文件，让项目代 - 理到我们想要抓包的网站。

修改反向代 - 理的网站后，因为我们需要抓包，所以服务器还得经过我们的 Fiddler 代 - 理，修改后的代码如下。

![](https://attach.52pojie.cn//forum/202310/08/023937cee91davca1cv8yv.png?l)

打开 Fiddler 需要把捕获通信关闭，否则无法正常使用。

![](https://attach.52pojie.cn//forum/202310/08/024055dvgh5ti2d6tht44g.png?l)

重新运行`app.py`，再次访问 [http://127.0.0.1](http://127.0.0.1)，Fiddler 出现抓包的数据

![](https://attach.52pojie.cn//forum/202310/08/024141t6wj4b96p93wxfnl.png?l)

![](https://attach.52pojie.cn//forum/202310/08/024105koiwkmxekhjr0jk0.png?l)

3. 反向代 - 理网站自签名 SSL 证书
======================

虽然成功反向代 - 理了 api 服务器的网站，但是在软件客户端使用了 HTTPS 协议加密，我们需要自签名证书然后设备信任后才可以访问。

这里我们先自签名反向代 - 理网站的证书。

我们需要使用另外一个工具`mkcert` [下载地址](https://ldz2020.lanzoue.com/iPg171b38suh)

生成自签证书
------

解压`mkcert.exe`文件到文件夹后，用 cmd 打开当前目录，输入命令生成证书。

在控制台中命令行输入： `mkcert.exe v3-api.china-****.cn`

![](https://attach.52pojie.cn//forum/202310/08/024109nkjkjmx0xxlxjx8k.png?l)

服务器配置证书
-------

生成了`./v3-api.china-****.cn.pem`证书文件和`./v3-api.china-****.cn-key.pem`私钥文件

把这两个文件更名成`server.pem`和`server-key.pem`并移动到反向代 - 理服务器的目录下。

![](https://attach.52pojie.cn//forum/202310/08/024121sln2794ln4ffx91n.png?l)

修改文件`app.py`，把代码替换成`app.run(host='0.0.0.0', port=443, ssl_context=('server.pem', 'server-key.pem'))`

![](https://attach.52pojie.cn//forum/202310/08/024144nnulihufutun4hq2.png?l)

发放证书
----

现在局域网访问测试地址会出现警告页面，我们把 CA 证书给手机客户端安装就可以了。

命令行查看 mkcert 的 CA 证书所在位置

命令行输入：`mkcert -CAROOT`

![](https://attach.52pojie.cn//forum/202310/08/024119ozfcrjcrcedentlr.png?l)

打开这个目录我们就可以看到 CA 证书了。

![](https://attach.52pojie.cn//forum/202310/08/024112mi3wwxj0hn133x1m.png?l)

4. 手机客户端上的设置
============

信任证书
----

我们上面的`rootCA.pem`发送到手机，安装并信任，安卓的话另外百度搜索。

![](https://attach.52pojie.cn//forum/202310/08/024114w7h9hhvdttw3uja3.png?l)

![](https://attach.52pojie.cn//forum/202310/08/024049kld1nh02xl9vy2q1.png?l)

设置 DNS 服务器
----------

保持 iPhone 和电脑主机在同一局域网里，并且 DNS 服务器和反向代 - 理服务器和 Fiddler 都打开，打开无线局域网 - WiFi 名感叹号 - 配置 DNS - 手动，多余的 DNS 删掉，把我们内网的 IP 填入进去，点击存储。

![](https://attach.52pojie.cn//forum/202310/08/024139n2rtehpw8thhr598.png?l)

完成
--

最后我们打开之前不能使用抓包的网站，可以看到成功打开了，同时 DNS 服务器，反向代 - 理服务器，还有 Fiddler 都有数据产生，接下来大家就可以自由发挥了

##### DNS 服务器

![](https://attach.52pojie.cn//forum/202310/08/024100yksl2dpeep255fks.png?l)

##### 反向代 - 理服务器

![](https://attach.52pojie.cn//forum/202310/08/024047jdf65z9yhzjuarar.png?l)

##### Fiddler

![](https://attach.52pojie.cn//forum/202310/08/024107y3hf3iitb1i93v6v.png?l)

##### 数据修改

![](https://attach.52pojie.cn//forum/202310/08/024103m0bzo6allxpj0naf.png?l)

此方法只是提供一种思路，实际上这种方式繁琐且无法过双向证书认证的 app，如果有更好的方法，麻烦与我分享
====================================================

 ![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg) 首先你这不是软件，不应该发原创区，我给你移动了，其次你图片粘贴有问题，MD 可以用论坛附件上传插入，不能这么引用临时地址，很快会失效，看这个编辑一下：[https://www.52pojie.cn/misc.php? ... 29&messageid=36](https://www.52pojie.cn/misc.php?mod=faq&action=faq&id=29&messageid=36)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Hmily 其实核心部分可以使用 charles 实现，但对于 https 类网站都需要安装根证书才行，像安卓装根证书就必须 ROOT，装不上根证书，基本就是抓不到数据的（非安全网络），服务端可以使用 passwall 类软路由，不需要做任何 DNS 劫持，只要添加一个 charles 的透明代 {过}{滤} 理节点就行了，至于拦到数据包了，想怎么使用就是 charles 的事情了，这个还是很强大的，我经常使用其中的 Map Remote 功能，可以将感兴趣的请求 URL，直接转发到自己的 http 服务器，好处就是这里面就不需要再用 https 了，自由度更大，这样做的好处就是对于手机来说，只要装一个根证书，其它的全部服务端搞定，根本不需要 DNS 劫持 ![](https://avatar.52pojie.cn/data/avatar/000/50/68/41_avatar_middle.jpg) 大佬啊，之前只看到过有越狱后的插件，这个方法虽然看着繁琐，但是对小白还是很实用 ![](https://avatar.52pojie.cn/data/avatar/001/26/73/53_avatar_middle.jpg) bdzwater

> [Hmily 发表于 2023-10-9 12:22](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48193599&ptid=1842061)  
> 首先你这不是软件，不应该发原创区，我给你移动了，其次你图片粘贴有问题，MD 可以用论坛附件上传插入，不能 ...

好的谢谢![](https://avatar.52pojie.cn/images/noavatar_middle.gif)蜀黍说不能太长 选择干掉检测![](https://static.52pojie.cn/static/image/smiley/default/4.gif) ![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg) k452b

> [k452b 发表于 2023-10-9 12:56](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=48193827&ptid=1842061)  
> 好的谢谢

现在就把图片改一下，不然很快会失效。 ![](https://avatar.52pojie.cn/data/avatar/001/67/09/02_avatar_middle.jpg) 这个只能一个个域名添加，有些繁琐，期待能有支持全域名的方法。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)二师兄。 小白跟着一起学习一下 争取早日跟大佬一样厉害![](https://static.52pojie.cn/static/image/smiley/default/17.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Hmily 适用范围不大  不过思路不错