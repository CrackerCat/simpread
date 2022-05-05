> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.com](https://github.com/comwrg/FUCK-GFW)

> 记录各个包管理器使用代理的方法, 因为 GFW 已经浪费了已经数不清的时间, FUCK GFW. Contribute to comwrg/FUCK-GFW development by creating ......

[](#fuck-gfw)FUCK-GFW
=====================

I do not want waste time in GFW again.

[](#pip)pip
-----------

~/.config/pip/pip.conf

```
[global]
proxy=http://localhost:1087


```

注意不支持 socks5

### [](#reference)Reference

*   [https://my.oschina.net/tangshi/blog/699190](https://my.oschina.net/tangshi/blog/699190)

[](#git)git
-----------

### [](#clone-with-ssh)clone with ssh

在 文件 `~/.ssh/config` 后添加下面两行

```
Host github.com
ProxyCommand nc -X 5 -x 127.0.0.1:1080 %h %p

```

### [](#clone-with-http)clone with http

```
git config --global http.proxy http://127.0.0.1:1087


```

建议使用 http, 因为 socks5 在使用 git-lfs 时会报错`proxyconnect tcp: dial tcp: lookup socks5: no such host`

### [](#reference-1)Reference

[https://gist.github.com/laispace/666dd7b27e9116faece6](https://gist.github.com/laispace/666dd7b27e9116faece6)

[](#cargo)cargo
---------------

Cargo 会依次检查以下位置

1.  环境变量 `CARGO_HTTP_PROXY`

```
export CARGO_HTTP_PROXY=http://127.0.0.1:1080

```

2.  [任意 `config.toml`](https://doc.rust-lang.org/cargo/reference/config.html#hierarchical-structure) 中的 `http.proxy`

```
[http]
proxy = "127.0.0.1:1080"

```

3.  环境变量 `HTTPS_PROXY` & `https_proxy` & `http_proxy`

```
export https_proxy=http://127.0.0.1:1080
export http_proxy=http://127.0.0.1:1080

```

`http_proxy` 一般来讲没必要，除非使用基于 HTTP 的 Crate Repository

Cargo 使用 libcurl，故可接受任何符合 [libcurl format](https://everything.curl.dev/usingcurl/proxies) 的地址与协议

( `127.0.0.1:1080` , `http://127.0.0.1:1080`, `socks5://127.0.0.1:1080` ）均可

### [](#reference-2)Reference

[https://doc.rust-lang.org/cargo/reference/config.html#httpproxy](https://doc.rust-lang.org/cargo/reference/config.html#httpproxy)

[](#apt-apt-get)apt (apt-get)
-----------------------------

在 `/etc/apt/apt.conf.d/` 目录下新增 `proxy.conf` 文件，加入：

```
Acquire::http::Proxy "http://127.0.0.1:8080/";
Acquire::https::Proxy "http://127.0.0.1:8080/";


```

注：无法使用 Socks5 代理。

### [](#reference-3)Reference

[https://askubuntu.com/a/349765/883355](https://askubuntu.com/a/349765/883355)

[](#curl)curl
-------------

```
socks5 = "127.0.0.1:1080"

```

add to `~/.curlrc`

### [](#reference-4)Reference

[https://www.zhihu.com/question/31360766](https://www.zhihu.com/question/31360766)

[](#gradle)Gradle
-----------------

这个浪费了好长时间额 ~/.gradle/gradle.properties

```
systemProp.http.proxyHost=127.0.0.1
systemProp.http.proxyPort=1087
systemProp.https.proxyHost=127.0.0.1
systemProp.https.proxyPort=1087


```

### [](#reference-5)Reference

[https://stackoverflow.com/questions/5991194/gradle-proxy-configuration](https://stackoverflow.com/questions/5991194/gradle-proxy-configuration)

[](#maven)Maven
---------------

%Maven 安装目录 %/conf/settings.xml

```
  <!-- proxies
   | This is a list of proxies which can be used on this machine to connect to the network.
   | Unless otherwise specified (by system property or command-line switch), the first proxy
   | specification in this list marked as active will be used.
   |-->
  <proxies>
    <!-- proxy
     | Specification for one proxy, to be used in connecting to the network.
     |
    <proxy>
      <id>optional</id>
      <active>true</active>
      <protocol>http</protocol>
      <username>proxyuser</username>
      <password>proxypass</password>
      <host>proxy.host.net</host>
      <port>80</port>
      <nonProxyHosts>local.net|some.host.com</nonProxyHosts>
    </proxy>
    -->
     <proxy>
      <id>proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>127.0.0.1</host>
      <port>1087</port>
    </proxy>
  </proxies>

```

### [](#reference-6)Reference

[https://maven.apache.org/guides/mini/guide-proxies.html](https://maven.apache.org/guides/mini/guide-proxies.html)

[](#go-get)go get
-----------------

```
HTTP_PROXY=socks5://localhost:1080 go get


```

测试了下 HTTPS_PROXY 和 ALL_PROXY 都不起作用

OR 使用 [goproxy.io](https://goproxy.io/)

[](#npm)npm
-----------

```
npm config set proxy http://127.0.0.1:1087
npm config set https-proxy http://127.0.0.1:1087


```

用 socks5 就报错 - -

推荐使用 yarn，npm 是真的慢

### [](#reference-7)reference

*   [https://stackoverflow.com/questions/7559648/is-there-a-way-to-make-npm-install-the-command-to-work-behind-proxy](https://stackoverflow.com/questions/7559648/is-there-a-way-to-make-npm-install-the-command-to-work-behind-proxy)

[](#rustup)rustup
-----------------

```
export https_proxy=http://127.0.0.1:1080

```

废物中国电信，不挂代理慢如龟

[](#yarn)yarn
-------------

```
yarn config set proxy http://XX
yarn config set https-proxy http://XX


```

不支持 socks5

### [](#reference-8)Reference

[yarnpkg/yarn#3418](https://github.com/yarnpkg/yarn/issues/3418)

[](#gem)gem
-----------

~/.gemrc

```
---
# See 'gem help env' for additional options.
http_proxy: http://localhost:1087


```

[](#reference-9)reference
-------------------------

*   google

[](#brew)brew
-------------

```
ALL_PROXY=socks5://localhost:1080 brew ...


```

[](#wget)wget
-------------

```
use_proxy=yes
http_proxy=127.0.0.1:1087
https_proxy=127.0.0.1:1087


```

~/.wgetrc

### [](#reference-10)Reference

*   [https://stackoverflow.com/questions/11211705/how-to-set-proxy-for-wget](https://stackoverflow.com/questions/11211705/how-to-set-proxy-for-wget)

[](#snap)snap
-------------

```
sudo snap set system proxy.http="http://127.0.0.1:1087"
sudo snap set system proxy.https="http://127.0.0.1:1087"

```

### [](#reference-11)Reference

[https://askubuntu.com/questions/764610/how-to-install-snap-packages-behind-web-proxy-on-ubuntu-16-04#answer-1146047](https://askubuntu.com/questions/764610/how-to-install-snap-packages-behind-web-proxy-on-ubuntu-16-04#answer-1146047)

[](#docker)docker
-----------------

```
$ sudo mkdir -p /etc/systemd/system/docker.service.d
$ sudo vim /etc/systemd/system/docker.service.d/proxy.conf

[Service]
Environment="ALL_PROXY=socks5://localhost:1080"

$ sudo systemctl daemon-reload
$ sudo systemctl restart docker


```

必须是 socks5，http 不生效

[](#electron-dev-dependency)Electron Dev Dependency
---------------------------------------------------

设置环境变量

```
ELECTRON_GET_USE_PROXY=true
GLOBAL_AGENT_HTTPS_PROXY=http://localhost:1080


```

### [](#references)References

*   [https://www.electronjs.org/docs/latest/tutorial/installation#proxies](https://www.electronjs.org/docs/latest/tutorial/installation#proxies)
*   [https://github.com/gajus/global-agent/blob/v2.1.5/README.md#environment-variables](https://github.com/gajus/global-agent/blob/v2.1.5/README.md#environment-variables)