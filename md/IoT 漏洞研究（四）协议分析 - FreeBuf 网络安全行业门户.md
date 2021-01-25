> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.freebuf.com](https://www.freebuf.com/articles/terminal/257628.html)

前情提要
----

> [IoT 漏洞研究（一）固件基础](https://www.freebuf.com/articles/terminal/254257.html)
> 
> [IoT 漏洞研究（二）Web 服务](https://www.freebuf.com/articles/terminal/254258.html)
> 
> [IoT 漏洞研究（三）硬件剖析](https://www.freebuf.com/articles/terminal/255742.html)

协议分析
----

IoT 模型一般分为感知层、传输层、应用层，本篇将探讨 IOT 漏洞研究中传输层的协议分析。当然由于研究角度不同也会出现四层、五层的分类模型，但从整体脆弱性作漏洞风险分析，无非是设备终端、管理软件 (APP/Browser)、服务(云) 平台三个端点，以及这三个点相互的数据传输。

![](https://image.3001.net/images/20201212/1607765063_5fd48c479a04e5832795f.png!small)三个端点通信的模块是接口，接口的连线就是通信协议，接口和连线都会有漏洞隐患，也是本文讨论的主题，web 协议之前已经单做讨论，这里不再赘述，下面主要从 “通用” 协议、“专用”协议和 “专有” 协议三个方面作漏洞分析。由于笔者词穷，并没有想到好的标题表达自己的分类方式，而且也仅限个人想法并非权威，所以都打上引号。

### 4.1 “通用” 协议

这里的 “通用” 协议指的是不仅在 IOT 上使用的一般网络协议，可以是有线也可以是无线。由于这些协议的研究手法比较通用，这里只作简单介绍。

*   SSH

SSH(Secure Shell) 是大家最熟悉的远程管理协议之一，许多设备都提供了该接口。SSH 最大的隐患当然是弱口令爆破攻击，下面几种协议亦然，当然，除了爆破认证绕过也是此类协议漏洞挖掘的重要方向。由于实现代码已经比较成熟，设备存在 SSH 漏洞案例不多，但也并非全然没有安全隐患，研究时可以作一些协议上的 fuzz。

> [https://github.com/tintinweb/paramiko-sshfuzz](https://github.com/tintinweb/paramiko-sshfuzz)

*   Telnet

Telnet 被认为是 SSH 低配版，安全性也较低。研究者一般在漏洞利用中会打开 telnetd 服务获取 shell，Telnet 协议比较简单，可以利用或自己开发一些 fuzz 工具对设备进行测试。

> [https://github.com/naliferopoulos/telnet-fuzzer](https://github.com/naliferopoulos/telnet-fuzzer)

*   FTP

FTP/SMB 在设备中也比较常见，也有些溢出漏洞的案例，通过二进制危险函数审计或者借助测试工具一般很快可以定位。

> [https://github.com/exploitsecurity/Python-FTP-Fuzzer](https://github.com/exploitsecurity/Python-FTP-Fuzzer)

*   SNMP

SNMP 如果实现不当，有敏感信息泄露的隐患，甚至可以直接控制设备。比如前几年的某网关设备，由于代码中没有正确处理 community 认证，导致任意 community 均可以通过认证，直接使用 snmpget 命令发送 SNMP GET 请求，并指定任意字符串作为 community 均可通过认证。

```
snmpget -v 1 -c public $IP iso.3.6.1.2.1.1.1.0
snmpget -v 1 -c '#Stringbleed' $IP iso.3.6.1.4.1.4491.2.4.1.1.6.1.1.0
snmpget -v 1 -c '#Stringbleed' $IP iso.3.6.1.4.1.4491.2.4.1.1.6.1.2.0


```

*   (SSL/Ipsec) VPN

边界和防护设备一般会提供 VPN 功能，尤其疫情期间，远程办公需求不断增加，在零信任还没普及落地前 VPN 仍是不二之选。有些厂商会将其集成到 web 中，方便使用的同时也带来不少安全隐患。比如某边界设备由于 sslvpn 功能实现不当造成登录等敏感信息泄漏问题。

```
import requests 
r = requests.get('https://sslvpn/dana-
na/../dana/html5acc/guacamole/../../../../../../etc/
passwd?/dana/html5acc/guacamole/') print r.content

```

![](https://image.3001.net/images/20201212/1607750247_5fd45267c05b9c13976f3.png!small)

> 对于这些协议除了上述专用测试工具以外，研究中还会使用到一些集成工具或 fuzz 框架，以 sulley(不再更新) 为代表，大致思路都差不多，比如 boofuzz，kitty，或是比较新的 Fuzzowski 等，这些工具文末会作介绍。

### 4.2 “专用” 协议

所谓 “专用” 协议就是一般只在设备上采用，即之前章节提到的 IOT 无线电研究范畴。由于篇幅限制，下文重点介绍这些协议常见的攻击方式和漏洞点，对协议细节不再赘述。

**4.2.1 WiFi**

WIFI 是一种标准，且 PC 上也广泛使用，但作为无线路由的重要功能点，设备 wifi 模块中也可能存在的一些协议漏洞。爆破和中间人攻击最为常见，此外还有其他协议漏洞点。

*   Krack

> 密钥重装攻击 (Key Reinstallation Atacks， 即 Krack)，该攻击对加密安全构成理论性的威胁，某些条件下，可以恢复用户明文数据、实施重放攻击、或者会话劫持。

![](https://image.3001.net/images/20201212/1607750307_5fd452a318c6740f59d1a.png!small)攻击者与 station 完成四次握手，但不转发四次握手的第四帧 Msg4 给 AP。此时 station 认为四次握手完成，开始加密并发送数据，AP 会重传 Msg3，station 收到重传的 Msg3 后重装会话密钥 PTK，重置报文序号，重置密钥流，重新开始加密数据。

*   Kr00k

> Kr00k 是 2020 年 2 月份 RSA 大会上披露的一个漏洞 CVE-2019-15126，由芯片驱动实现问题造成，主要影响 Broadcom 和 Cypress 网卡。

![](https://image.3001.net/images/20201212/1607750378_5fd452eaec4ac4652e474.png!small)其核心漏洞点是在解除客户端关联后，其 PTK 会被置零，但是 WiFi 芯片会继续用置零的 PTK 发送缓冲中剩余的无线数据，攻击者收到这些数据后使用全零的 PTK 即可解密。

**4.2.2 RFID**

RFID(Radio Frequency Identification)，即射频识别，是自动识别技术的一种，生活中的各种智能卡和 RFID 技术关系密切。说到 RFID，一般都会提起现在应用广泛的 NFC(Near Field Communication) 近场通信，NFC 可以看作是 RFID 的子集，物理层、协议层遵循 RFID 标准，只是应用层协议不同。  
RFID 涉及的通信基础较多，以下是 RFID 常用的几种攻击方式。

**嗅探攻击**

![](https://image.3001.net/images/20201212/1607765502_5fd48dfeeb93c7e8a2032.png!small)

**伪造数据越权读写**

![](https://image.3001.net/images/20201212/1607765526_5fd48e162cca4ebf27f35.png!small)

**存储数据篡改**

![](https://image.3001.net/images/20201212/1607765546_5fd48e2accafb9a451167.png!small)

**攻击中间件和后端系统**

![](https://image.3001.net/images/20201212/1607765559_5fd48e37df323ff567e74.png!small)除此之外还有其他的一些攻击手段，比如频率干扰等，这里不再一一列举。

**4.2.3 Bluetooth**

蓝牙是 IOT 设备中常用的传输方式，现在的智能家庭网除了使用 WIFI，蓝牙传输也是重要手段。蓝牙这个名称十分有趣，来自十世纪一位丹麦国王 Harald Blatand，Blatand 在英文里可以被解释为 Bluetooth，因为国王喜欢吃蓝梅，牙龈每天都是蓝色的而得名。

![](https://image.3001.net/images/20201212/1607750565_5fd453a568490fa30ea12.png!small)由于蓝牙 (BLE) 实现功耗逐步降低，包括手机在内的许多设备都默认打开，进一步增加了安全隐患。

**BleedingBit**

BleedingBit 由 Armis 的安全研究人员发现，漏洞存在于德州仪器生产的 BLE 芯片中，影响 Cisco、Meraki、Aruba 等多家公司设备。BleedingBit 包括 BleedingBit RCE(CVE-2018-16986) 和 BleedingBit OAD RCE(CVE-2018-7080) 两个漏洞。

*   BleedingBit RCE

首先发送正常广播信息，这些信息被接收并存储到目标设备中；继续发送恶意数据包，数据包特定头信息置位改变，从而触发漏洞。

*   BleedingBit OAD RCE

利用自己修改过的固件覆盖原先的系统，从而控制目标设备。

**BIAS**

BIAS(Bluetooth Impersonation AttackS)，即蓝牙冒充攻击。Bluetooth BR/EDR 中被发现了一些严重的安全漏洞，包括缺乏强制相互身份认证、角色转化过度轻松、认证过程降级等。攻击者利用该漏洞可以打破标准适配设备的蓝牙安全机制，最后可以在安全连接建立后仿冒已配对设备发送数据。从本质上说，BIAS 攻击是利用了蓝牙设备如何处理长期连接的漏洞。  
![](https://image.3001.net/images/20201212/1607750587_5fd453bb940231bc9e109.jpg!small)**4.2.4 zigbee**

ZigBee 是比较新的无线通信技术，适用于短距离设备之间数据传输。Zigbee 与蓝牙相比能建立更大的网络，功耗比起 wifi 相对要低不少，所以经常在家庭、工厂等应用场景使用。  
![](https://image.3001.net/images/20201212/1607750603_5fd453cb988350bc0553e.png!small)ZigBee 协议分为物理层、MAC 层、网络称和应用层 4 层:  
![](https://image.3001.net/images/20201212/1607750617_5fd453d983f3df04512fc.png!small)提供 3 个等级的安全模式：

> 1、非安全模式：不采取任何安全服务，容易被窃听；  
> 2、访问控制模式：通过 ACL 限制非法节点；  
> 3、安全模式：采用 AES 128 位加密通信。

Zigbee 常见攻击方式：

数据窃听

> 当 ZigBee 使用非安全模式，也就是其默认安全策略时，对传输数据将不作加密，所以可以通过嗅探窃听到传输数据。

密钥攻击

> 在密钥传输过程中，可能会以明文形式传输密钥，因此有被窃取密钥的风险。

![](https://image.3001.net/images/20201212/1607750642_5fd453f2dabfc4ed456f9.png!small)目前针对 ZigBee 的攻击，主要还是窃听和密钥安全方向，比较流行的研究工具有 [KillerBee](https://github.com/riverloopsec/killerbee)，可实现抓包、分析和发包等功能。虽然 ZigBee 没有 WiFi、蓝牙那样流行，但其安全问题仍不容忽视。

**4.2.5 MQTT**