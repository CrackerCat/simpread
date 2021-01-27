> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [security.tencent.com](https://security.tencent.com/index.php/blog/msg/179)

SSRF(Server-Side Request Forgery: 服务器端请求伪造) 是一种由攻击者构造形成，由服务端发起请求的一个安全漏洞。SSRF 是笔者比较喜欢的一个漏洞，因为它见证了攻防两端的对抗过程。本篇文章详细介绍了 SSRF 的原理，在不同语言中的危害及利用方式，常见的绕过手段，新的攻击手法以及修复方案。

SSRF 是 Server-side Request Forge 的缩写，中文翻译为服务端请求伪造。产生的原因是由于服务端提供了从其他服务器应用获取数据的功能且没有对地址和协议等做过滤和限制。常见的一个场景就是，通过用户输入的 URL 来获取图片。这个功能如果被恶意使用，可以利用存在缺陷的 web 应用作为代理攻击远程和本地的服务器。这种形式的攻击称为服务端请求伪造攻击。

以 PHP 为例，常见的缺陷代码如下：

```
function curl($url){  
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    curl_exec($ch);
    curl_close($ch);
}
$url = $_GET['url'];
curl($url);

```

从上面的示例代码可以看出，请求是从服务器发出的，那么攻击者可以通过构造恶意的 url 来访问原本访问不到的内网信息，攻击内网或者本地其他服务。这里根据后续处理逻辑不同，还会分为回显型 ssrf 和非回显型 ssrf，所谓的回显型的 ssrf 就是会将访问到的信息返回给攻击者，而非回显的 ssrf 则不会，但是可以通过 dns log 或者访问开放 / 未开放的端口导致的延时来判断。

SSRF 的最大的危害在于穿透了网络边界，但具体能做到哪种程度还需要根据业务环境来判断。例如我们在 SSRF 的利用中，如果需要更深一步扩展，第一反应通常是去攻击可利用的 redis 或者 memcache 等内网服务拿 shell，但需要注意的是操作 redis，memcache 的数据包中是需要换行的，而 http/https 协议一般无法满足我们要求，所以即使内网存在可利用的 redis，也并非所有的 ssrf 都能利用成功的。但是，对于 memcache 来说，即使只能使用 https 协议，利用 memcache 来 getshell 却并非不可能，本文会详细介绍一种新型的攻击方式。

2.1 SSRF 在 PHP 中的利用
-------------------

在 PHP 中，经常出现 SSRF 的函数有 cURL、file_get_contents 等。

cURL 支持 http、https、ftp、gopher、telnet、dict、file 和 ldap 等协议，其中 gopher 协议和 dict 协议就是我们需要的。利用 gopher,dict 协议，我们可以构造出相应 payload 直接攻击内网的 redis 服务。

![](https://security.tencent.com/uploadimg_dir/202101/7a4c01244bf33f09b6b6c58aef2017d7.png)

需要注意的是：

1. file_get_contents 的 gopher 协议不能 UrlEncode

2. file_get_contents 关于 Gopher 的 302 跳转有 bug，导致利用失败

3. curl/libcurl 7.43 上 gopher 协议存在 bug（截断），7.45 以上无此 bug

4. curl_exec() 默认不跟踪跳转

5. file_get_contents() 支持 php://input 协议

2.2 SSRF 在 Python 中的利用
----------------------

在 Python 中，常用的函数有 urllib(urllib2) 和 requests 库。以 urllib(urllib2) 为例， urllib 并不支持 gopher,dict 协议，所以按照常理来讲 ssrf 在 python 中的危害也应该不大，但是当 SSRF 遇到 CRLF，奇妙的事情就发生了。

urllib 曾爆出 CVE-2019-9740、CVE-2019-9947 两个漏洞，这两个漏洞都是 urllib(urllib2) 的 CRLF 漏洞，只是触发点不一样，其影响范围都在 urllib2 in Python 2.x through 2.7.16 and urllib in Python 3.x through 3.7.3 之间，目前大部分服务器的 python2 版本都在 2.7.10 以下，python3 都在 3.6.x，这两个 CRLF 漏洞的影响力就非常可观了。其实之前还有一个 CVE-2016-5699，同样的 urllib（urllib2）的 CRLF 问题，但是由于时间比较早，影响范围没有这两个大，这里也不再赘叙

python2 代码如下：

```
import sys
import urllib2
host = "127.0.0.1:7777?a=1 HTTP/1.1\r\nCRLF-injection: test\r\nTEST: 123"
url = "http://"+ host + ":8080/test/?test=a"
try:
  info = urllib2.urlopen(url).info()
  print(info)
except Exception as e:
print(e)

```

![](https://security.tencent.com/uploadimg_dir/202101/827e6d22caf1a909f8e32023883b6efb.png)

可以看到我们成功注入了一个 header 头，利用 CLRF 漏洞，我们可以实现换行对 redis 的攻击

![](https://security.tencent.com/uploadimg_dir/202101/435ca645fc6f7803b0c8daa013d4ec0b.png)

除开 CRLF 之外，urllib 还有一个鲜有人知的漏洞 CVE-2019-9948，该漏洞只影响 urllib，范围在 Python 2.x 到 2.7.16，这个版本间的 urllib 支持 local_file/local-file 协议，可以读取任意文件，如果 file 协议被禁止后，不妨试试这个协议来读取文件。

![](https://security.tencent.com/uploadimg_dir/202101/03faccbec3d3d559d5cdaa42c1f3c439.png)

2.3 SSRF 在 JAVA 中的利用
--------------------

相对于 php，在 java 中 SSRF 的利用局限较大，一般利用 http 协议来探测端口，利用 file 协议读取任意文件。常见的类中如 HttpURLConnection，URLConnection，HttpClients 中只支持 sun.net.www.protocol (java 1.8) 里的所有协议: http，https，file，ftp，mailto，jar，netdoc。

但这里需要注意一个漏洞，那就是 weblogic 的 ssrf，这个 ssrf 是可以攻击可利用的 redis 拿 shell 的。在开始看到这个漏洞的时候，笔者感到很奇怪，因为一般 java 中的 ssrf 是无法攻击 redis 的，但是网上并没有找到太多的分析文章，所以特地看了下 weblogic 的实现代码。

详细的分析细节就不说了，只挑重点说下过程，调用栈如下

![](https://security.tencent.com/uploadimg_dir/202101/65524259e181c1d0bec85b852d099329.png)

我们跟进 sendMessage 函数（UDDISoapMessage.java）

![](https://security.tencent.com/uploadimg_dir/202101/4a3d3334b1be3c112698796719cb133e.png)

sendMessage 将传入的 url 赋值给 BindingInfo 的实例，然后通过 BindingFactory 工厂类，来创建一个 Binding 实例，该实例会通过传入的 url 决定使用哪个接口。

![](https://security.tencent.com/uploadimg_dir/202101/186468480a01912d82c927440ec88e65.png)

这里使用 HttpClientBinding 来调用 send 方法，

![](https://security.tencent.com/uploadimg_dir/202101/d566b8aabed356a63e56dd4178f3d500.png)

send 方法使用 createSocket 来发送请求，这里可以看到直接将传入的 url 代入到了 socket 接口中

![](https://security.tencent.com/uploadimg_dir/202101/56c79a037f1e43a19b64f5325c1bfff6.png)

这里的逻辑就很清晰了, weblogic 并没有采用常见的网络库，而是自己实现了一套 socket 方法，将用户传入的 url 直接带入到 socket 接口, 而且并没有校验 url 中的 CRLF。

SSRF 的攻防过程也是人们对 SSRF 漏洞认知不断提升的一个过程，从开始各大厂商不认可 SSRF 漏洞 -> 攻击者通过 SSRF 拿到服务器的权限 -> 厂商开始重视这个问题，开始使用各种方法防御 -> 被攻击者绕过 -> 更新防御手段，在这个过程中，攻击者和防御者的手段呈螺旋式上升的趋势，也涌现了大量绕过方案。

常见的修复方案如下：

![](https://security.tencent.com/uploadimg_dir/202101/81138e5f4403b0c5b7b1dc96bcd650ff.png)

用伪代码来表示的话就是

```
if check_ssrf(url):
    do_curl(url)
else:
    print(“error”)

```

图中的获取 IP 地址和判断 IP 地址即是 check_ssrf 的检验，所有的攻防都是针对 check_ssrf 这个函数的绕过与更新，限于篇幅原因，这里取几个经典绕过方案讲解一下

3.1 30x 跳转
----------

30x 跳转也是 SSRF 漏洞利用中的一个经典绕过方式，当防御方限制只允许 http(s) 访问或者对请求的 host 做了正确的校验后，可以通过 30x 方式跳转进行绕过。

针对只允许 http(s) 协议的情况，我们可以通过  
Location: dict://127.0.0.1:6379 跳转到 dict 协议，从而扩大我们攻击面，来进行更深入的利用

针对没有禁止 url 跳转，但是对请求 host 做了正确判断的情况，我们则可以通过 Location: [http://127.0.0.1:6379 的方式来绕过限制](http://127.0.0.1:6379的方式来绕过限制)

3.2 URL 解析绕过
------------

其实在 ssrf 利用的过程中也零星有利用 url 解析导致绕过 check_ssrf 的 payload，但大部分利用 payload 之所以能成功是因为防御者在编写代码时使用的正则匹配不当。第一个正式的深入利用是 orange 在 blackhat 大会上提出的 [A-New-Era-Of-SSRF-Exploiting](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf "A-New-Era-Of-SSRF-Exploiting")，利用语言本身自带的解析函数差异来绕过检测，在该 ppt 中举例了大量不同编程语言的 url 解析函数对 url 解析的差异，从而导致 check_ssrf 和 do_curl 解析不同导致的绕过，有兴趣的同学可以参看附录一，这里以笔者发现的一个例子作为讲解。

在 python3 中，笔者发现对于同一个 url:http://baidu.com\@qq.com，urllib 和 urllib3 的解析就不一致。   

![](https://security.tencent.com/uploadimg_dir/202101/7f377f14d06f2748e490c043b639814e.png)

可以看到，对于 [http://baidu.com\](http://baidu.com/)[@qq](https://github.com/qq "@qq").com 来说，urllib3 取到的 host 是 baidu.com，而 urllib 取到的 host 是 qq.com。笔者曾就此问题与 python 官方联系过，但是官方并不认为这是一个安全问题，表示 urllib3 与 chrome 浏览器的解析保持一致（将 \ 视为 /），所以这个问题会一直存在。如果在 check_ssrf 中解析 url 函数用的是 urllib3，而业务代码发送请时采用的是 urllib，两者之间的解析差异就会导致绕过的情况。

这里多说一句，其实 url 解析问题涉及到了 web 安全的底层逻辑，不仅仅是在 ssrf 中有利用，其危害范围很广，包括不限于 url 跳转，oauth 认证，同源策略（如 postMessage 中 origin 的判断）等一切会涉及到 host 判断的场景。 

3.3 DNS rebinding
-----------------

从 SSRF 修复方案来看，这里流程中进行了两次 DNS 解析，第一次在 check_ssrf 的时候会对 URL 的 host 进行 DNS 解析，第二次在 do_curl 请求时进行解析。这两次 DNS 解析是有时间差的，我们可以使用这个时间差进行绕过。  

 

时间差对应 DNS 中的机制是 TTL。TTL 表示 DNS 里面域名和 IP 绑定关系的 Cache 在 DNS 上存活的最长时间。即请求了域名与 iP 的关系后，请求方会缓存这个关系，缓存保持的时间就是 TTL。而缓存失效后就会删除，这时候如果重新访问域名指定的 IP 的话会重新建立匹配关系及 cache。

当我们设置 TTL 为 0 时，当第一次解析域名后，第二次会重新请求 DNS 服务器获取新的 ip。DNS 重绑定攻击的原理是：利用服务器两次解析同一域名的短暂间隙，更换域名背后的 ip 达到突破同源策略或过 waf 进行 ssrf 的目的。

1. 在网站 SSRF 漏洞处访问精心构造的域名。网站第一次解析域名，获取到的 IP 地址为 A；  
2. 经过网站后端服务器的检查，判定此 IP 为合法 IP。  
3. 网站获取 URL 对应的资源（在一次网络请求中，先根据域名服务器获取 IP 地址，再向 IP 地址请求资源），第二次解析域名。此时已经过了 ttl 的时间，解析记录缓存 IP 被删除。第二次解析到的域名为被修改后的 IP 即为内网 IP B；  
4. 攻击者访问到了内网 IP。   

当然，上述情况是最理想的情况，在不同的语言，不同服务器中也存在差异

 

1. java 中 DNS 请求成功的话默认缓存 30s(字段为 networkaddress.cache.ttl，默认情况下没有设置)，失败的默认缓存 10s。（缓存时间在 /Library/Java/JavaVirtualMachines/jdk /Contents/Home/jre/lib/security/java.security 中配置）  
2. 在 php 中则默认没有缓存。  
3. Linux 默认不会进行 DNS 缓存，mac 和 windows 会缓存 (所以复现的时候不要在 mac、windows 上尝试)  
4. 有些公共 DNS 服务器，比如 114.114.114.114 还是会把记录进行缓存，但是 8.8.8.8 是严格按照 DNS 协议去管理缓存的，如果设置 TTL 为 0，则不会进行缓存。   

在传统的 ssrf 修复方案中，由于 java 会存在默认的 dns 缓存，所以一般认为 java 不存在 DNS rebinding 问题。但是试想这么一个场景，如果刚刚好到了 DNS 缓存时间，此时更新 DNS 缓存，那些已经过了 SSRF Check 而又没有正式发起业务请求的 request，是否使用的是新的 DNS 解析结果。其实理论上只要在发起第一次请求后等到 30 秒之前的时候再请求即可，但为了保证效果，可以在 28s 左右，开始以一个较短的时间间隔去发送请求，以达到时间竞争的效果。相关示例代码可参考附录三。

Dns rebinding 常见方案除了自建 dns 服务器之外，还可以通过绑定两个 A 记录，一个绑定外网 ip，一个绑定内网 ip。当然这种情况访问顺序是随机的，无法保证成功率。

自建 dns 服务器需要将域名的 dns 服务指向自己的 vps，然后在 vps 上运行 dns_server 脚本，dns_server.py 内容如下   

```
from twisted.internet import reactor, defer
from twisted.names import client, dns, error, server


record={}
class DynamicResolver(object):
    def _doDynamicResponse(self, query):
        name = query.name.name
        if name not in record or record[name]<1:
            # 随意一个 IP，绕过检查即可
            ip="104.160.43.154"
        else:
            ip="127.0.0.1"
        if name not in record:
            record[name]=0
        record[name]+=1
        print name+" ===> "+ip
        answer = dns.RRHeader(
            name=name,
            type=dns.A,
            cls=dns.IN,
            ttl=0,
            payload=dns.Record_A(address=b'%s'%ip,ttl=0)
        )
        answers = [answer]
        authority = []
        additional = []
        return answers, authority, additional
    def query(self, query, timeout=None):
        return defer.succeed(self._doDynamicResponse(query))

def main():
    factory = server.DNSServerFactory(
        clients=[DynamicResolver(), client.Resolver(resolv='/etc/resolv.conf')]
    )
    protocol = dns.DNSDatagramProtocol(controller=factory)
    reactor.listenUDP(53, protocol)
    reactor.run()

if __name__ == '__main__':
    raise SystemExit(main())

```

效果如下：

![](https://security.tencent.com/uploadimg_dir/202101/ea24775d47a542c758fcf5fabded4e1a.png)  

当第一次访问时，解析为外网 ip 通过 ssrf 检测，  
第二次访问时，也即业务访问时，ip 会指向 127.0.0.1，从而达到了绕过目的。

除此之外，还有一些绕过技巧，例如利用 IPv6,  
有些服务没有考虑 IPv6 的情况，但是内网又支持 IPv6，则可以使用 IPv6 的本地 IP 如::1 或 IPv6 的内网域名 --x.1.ip6.name 来绕过过滤

 

如果只支持 https 协议是否存在深入利用的可能？答案是存在的，2020 年的 blackhat 大会上有一个议题 When TLS hacks you，这篇文章利用 tls 对 ssrf 进行深入利用。该手法是将 ssrf+dns rebinding+tls session 完美结合在一起的利用链，能够在不依赖其他漏洞（如 CRLF）的情况下，攻击内网中的 memcache、SMTP 服务。

**这里介绍一下相关原理：**

当客户端和服务器端初次建立 TLS 握手时（例如浏览器访问 HTTPS 网站），需要双方建立一个完整的 TLS 连接，该过程为了保证数据的传输具有完整性和机密性，需要做很多事情，如密钥协商出会话密钥，数字签名身份验证，消息验证码 MAC 等。这个过程是非常消耗资源的，而且当下一次客户端访问同一个 HTTPS 网站时，这个过程需要再重复一次，这无疑会造成大量的资源消耗。

 

为了提高性能，TLS/SSL 提供了会话恢复的方式，允许客户端和服务端在某次关闭连接后，下一次客户端访问时恢复上一次的会话连接。会话恢复有两种，一种是基于 session ID 恢复，一种是使用 Session Ticket 的 TLS 扩展。这里主要介绍一下 session ID。

每一个会话都由一个 Session ID 标识符标识，当建立一个 TLS 连接时，服务器会生成一个 session ID 给客户端，服务端保留会话记录，重新连接的客户端可以在 clientHello 消息期间提供此会话 ID，并重新使用此前建立的会话密钥，而不需要再次经历秘钥协商等过程。TLS session id 在 RFC-5077 中有着详细描述，基本所有数据都可以用作会话标志符，包括换行符。

 

Session ticket 和 session id 作用类似，在客户端和服务端建立了一次完整的握手后，服务端会将本次的会话数据加密，但是 session ticket 将会话记录保存在客户端，而且与 session id 32 字节的大小不同，session ticket 可提供 65k 的空间，这就能为我们的 payload 提供足够的空间。

在讲完这些细节之后，攻击的思路就会很清晰了，session id 是服务器提供给客户端的，如果我们构建一个恶意的 tls 服务器，然后将我们的恶意 session id 发送给客户端，然后通过 dns rebinding，将服务器域名的地址指向内网 ip 应用，例如 memcache，客户端在恢复会话时就会带上恶意的 session id 去请求内网的 memcache，从而攻击了内网应用。

1. 利用服务器发起一个 HTTPS 请求。  
2. 请求时会发起一个 DNS 解析请求，DNS 服务器回应一个 TTL 为 0 的结果，指向攻击者的服务器。  
3. 攻击者服务器响应请求，并返回一个精心构造过的 SessionID，并延迟几秒后回应一个跳转。  
4. 客户端接收到这个回应之后会进行跳转，这次跳转时由于前面那一次 DNS 解析的结果为 TTL 0，则会再次发起一次解析请求，这次返回的结果则会指向 SSRF 攻击的目标（例如本地的 memcache 数据库）。  
5. 因为请求和跳转时的域名都没有变更，本次跳转会带着之前服务端返回的精心构造过的 SessionID 进行，发送到目标的那个端口上。  
6. 则达到目的，成功对目标端口发送构造过的数据，成功 SSRF。

可以看到当请求同一个 https 站点时，curl 会 re-use 之前的 session Id，

![](https://security.tencent.com/uploadimg_dir/202101/b63622c546554ff2a0ea073a44e4eb7b.png)  

我们使用恶意的 https 服务器设置 session id，可以看到当跳转到本地时，客户端会带着恶意的 session id 请求本地的 11211 端口  

![](https://security.tencent.com/uploadimg_dir/202101/539cb710393a857ecbf814c6254b40e3.png)  

可以看到 memcache 已经被写入了数据  

![](https://security.tencent.com/uploadimg_dir/202101/8e225097b988ff4ba7efcd476c68fa4e.png)  

通过 wireshark 抓包来查看整个流程  

![](https://security.tencent.com/uploadimg_dir/202101/c7ebaa16053a6cdf6dd1472bdc9ed36d.png)  

在红框部分是客户端与恶意 HTTPS 服务器建立了一个完整的 TLS 连接，Server Hello 中恶意服务器返回了一个恶意的 session ID，然后进行了一个跳转，利用 DNS rebinding 将域名指向了本地 127.0.0.1，  

![](https://security.tencent.com/uploadimg_dir/202101/e61fd7c82bfe20b4fd060e03febf87ca.png)  

可以看到客户端向本地发送了一个 client hello 数据包，数据包中的 session id 就是恶意服务器设置的 session id，从而攻击了客户端本地的 memcache。

 

当然，这种攻击存在一定的局限性，除了依赖于发起请求的客户端外（客户端是否实现 TLS 缓存），由于 TLS 协议带有各种字符，例如 \ 0x00，可能会导致一些应用解析失败，例如就无法通过该方式来攻击 redis。以下是议题作者给出的受影响的 HTTPS client 列表以及可以攻击的应用   

![](https://security.tencent.com/uploadimg_dir/202101/7581626d5b7e25222d0c6acf68bcbb8b.png)  

![](https://security.tencent.com/uploadimg_dir/202101/a08201808a8d46114ffa064911c67f09.png)  

除此之外在复现过程中，笔者还发现了一些其他的限制条件，例如在某些 curl 版本中会无法复用 session id，而且在 tls1.3 版本中，也会出现复现失败的问题。

 

整体来讲，虽然复现起来存在一些限制，但是并不妨碍这个议题的思路的独到之处，笔者认为这是 2020 年 web 端最厉害的议题。拆开来讲，发起请求的客户端如 curl，tls 协议都不存在任何问题，议题作者曾表示 curl 在判断同源时，只判断了协议，域名，端口，但是没有判断 ip，这里在笔者看来并非是 curl 问题，由于负载均衡等问题，域名可能对应多个 ip，这种修复方式应该是不会被采纳的; 而 session id/ticket 更是 tls 为了提高性能而提出的机制，这个机制本身不存在问题，但是这两个组合起来加上 ssrf 就达到了意想不到的效果。

SSRF 的修复比较复杂，需要根据业务实际场景来采取不同的方案，例如前面说到的 python 中不同 url 库对 url 的解析就不一致，所以对于有条件的公司，建立一个代理集群是比较可靠的方案，将类似请求外部 url 的需求整理出来，分为纯外网集群和内网集群进行代理请求。

1. 去除 url 中的特殊字符  
2. 判断是否属于内网 ip  
3. 如果是域名的话，将 url 中的域名改为 ip  
4. 请求的 url 为 3 中返回的 url  
5. 请求时设置 host header 为 ip  
6. 不跟随 30x 跳转（跟随跳转需要从 1 开始重新检测）

 

其中第一步是为了防止利用 url parse 的特性造成 url 解析差异，第三步是为了防止 dns rebinding，第 5 步是为了防止以 ip 请求时，某些网站无法访问的问题，第 6 步是为了防止 30x 跳转进行绕过。   

在笔者看来，安全问题从某种程度来讲，其实是攻击者与防御者对同一事物的认知问题，当攻击者的认知超出防御者时，就会找到新的绕过方式，而当防御者的认知跟上之后，又会促使攻击者寻找新的方式，这一点在 ssrf 漏洞上体现得淋漓尽致。安全上的攻防，其实就是人与人之间的博弈，这也是安全的魅力所在。  
附录

 

1. SSRF 利用新纪元：https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf  
2. When TLS Hacks You:https://i.blackhat.com/USA-20/Wednesday/us-20-Maddux-When-TLS-Hacks-You.pdf  
3. Java 环境下通过时间竞争实现 DNS rebinding 绕过 ssrf 防御限制: https://mp.weixin.qq.com/s/dA40CUinwaitZDx6X89TKw   

腾讯蓝军（Tencent Force）由腾讯 TEG 安全平台部于 2006 年组建，十余年专注前沿安全攻防技术研究、实战演练、渗透测试、安全评估、培训赋能等，采用 APT 攻击者视角在真实网络环境开展实战演习，全方位检验安全防护策略、响应机制的充分性与有效性，最大程度发现业务系统的潜在安全风险，并推动优化提升，助力企业领先于攻击者，防患于未然。