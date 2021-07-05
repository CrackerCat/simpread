> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/)

> LWPANLPWAN（low-power Wide-Area Network，低功耗广域网）专为低带宽、低功耗、远距离、大量连接的物联网应用而设计。

[](#LWPAN "LWPAN")LWPAN
-----------------------

LPWAN（low-power Wide-Area Network，低功耗广域网）专为低带宽、低功耗、远距离、大量连接的物联网应用而设计。主流的技术有 NB-IoT、LoRa、eMTC、SigFox、RPMA、Weightless 等。

### [](#NB-IOT-与-LoRa-对比 "NB-IOT 与  LoRa 对比")NB-IOT 与 LoRa 对比

LPWAN 中 NB-IOT 与 LoRa 使用应用较为广泛。

*   NB-IOT 与 LoRa 对比 1

<table><thead><tr><th></th><th>NB-IOT</th><th>LoRa</th></tr></thead><tbody><tr><td>标准制定方</td><td>3GPP</td><td>LoRa 联盟（Semtech 所有）</td></tr><tr><td>技术特点</td><td>蜂窝</td><td>线性扩频</td></tr><tr><td>部署方式</td><td>与现有蜂窝网络复用</td><td>独立建网</td></tr><tr><td>自由度</td><td>低，依赖运营商的基础设施</td><td>高，自有部署</td></tr><tr><td>最大传输距离</td><td>城市 1-2km，郊区 20km</td><td>20km</td></tr><tr><td>频段</td><td>运营商频段</td><td>470MHz、920MHz、867MHz 等</td></tr></tbody></table>

*   应用场景对比 2
    
    ![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/compare.jpg)
    

#### [](#参考 "参考")参考

*   1 [看懂物联网（4）LoRa VS NB-IoT 的江湖](https://baijiahao.baidu.com/s?id=1613528481616705401&wfr=spider&for=pc)
*   2 [物联网接入技术：NB-IOT 与 LoRa 之争](https://baijiahao.baidu.com/s?id=1620998158592000869&wfr=spider&for=pc)

[](#LoRaWAN1-0-3-安全综述 "LoRaWAN1.0.3 安全综述")LoRaWAN1.0.3 安全综述
-----------------------------------------------------------

Long Range (LoRa®) 是低功耗广域网络（LPWAN）技术之一，LoRa 由美国公司 Semtech 制定，属于私有技术。LoRa 采用非授权频段，不同的地区采用的频段也不相同。 LoRaWAN 是为 LoRa 远距离通信网络设计的一套通讯协议和系统架构。它是一种媒体访问控制（MAC）层协议。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200206121856850.png)

<table><thead><tr><th>类别</th><th>应用实体</th><th>冲突解决方案</th><th>优点</th><th>缺点</th></tr></thead><tbody><tr><td>Class A（主流）</td><td>电池供电传感器</td><td>异步 ALOHA 协议</td><td>最佳节能</td><td>Server 无法唤醒 End Node</td></tr><tr><td>Class B</td><td>电池供电执行器</td><td></td><td>节能并唤醒时延可控</td><td>复杂，实现代价大</td></tr><tr><td>Class C</td><td>市电供电执行器</td><td></td><td>随时唤醒通信</td><td>耗能大</td></tr></tbody></table>

### [](#网络架构 "网络架构")网络架构

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200206122207783.png)

LoRaWAN 采用星星的网络拓扑结果， 网络中存在 4 类网络实体, 上图中从左到右分别是：

*   End Nodes（终端节点）
    
    终端节点通常搭配传感器使用，从环境中采集各种信息，如烟雾、天气等。终端设备在每次发送数据包都需要随机切换信道，以便降低同频干扰和无线信号衰减。
    
*   Gateway（网关）
    
    网关用于转发 “终端节点” 与“网络服务器”的之间的数据。**网关**与**终端节点**之间没有进行绑定，同一个节点的数据能被多个接收到，他们之间采用 LoRa RF 传输，国内采用 470MHz 频段。
    
*   Network Server（网络服务器）
    
    网络服务器用于把终端节点产生的数据转发给对应的应用服务器吗，并提供对终端节点认证和授权。网关与网络服务器之间使用 TCP/IP 协议栈，采用透明传输。常见的协议有 Packet Forwarder（现在被归类为 Legacy）、MQTT(主流)、CoAP、Protobuf。
    
*   Applicaton Server（应用服务器）
    
    应用服务器根据用户需要而设计，通常包括终端节点数据的展示（数据统计、异常数据告警）以及对节点的远程控制等。
    
    当前使用较为广泛的开源服务器是 [ChirpStack](https://www.chirpstack.io/)。
    

数据传输的过程是：终端节点采集到数据通过 LoRa RF 直接传送给网关，再由网关将数据转发给服务器进行处理。流量上行这个过程也被成为 “uplink”，相反流量从服务器到终端节点的这个过程被成为 “downlink”。

### [](#安全机制 "安全机制")安全机制

LoRaWAN 在设计之初就考虑到了安全问题，定义了两个密钥（NwkSKey 和 APPSKey）。NwkSKey(Network Session Key) 用于保障终端节点传输到网络服务器之间的数据的完整性；APPSKey(Application Session Key) 用于加密传输的数据，保障终端节点到应用服务器之间的数据的机密性。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200206163258251.png)

下图是 LoRa 的[消息帧格式](https://lora-alliance.org/sites/default/files/2018-07/lorawan1.0.3.pdf)。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200206165323731.png)

*   **MIC(message integrity code）**: 消息完整性代码为 4 个字节，用于确保数据的**完整性**（数据没有被篡改），计算公式如下。
    
    ```
    msg = MHDR | FHDR | FPort | FRMPayload
    B0= 0x49 | 4 * 0x00 | Dir | DevAddr| FCntUp or FCntDown | 0x00 |len(MHDR | MACPayload)
    cmac = aes128_cmac(NwkSKey, B0 | MHDR | MACPayload)
    MIC = cmac[0..3]
    ```
    
    `Dir` 在 uplink 帧中为 0，在 downlink 帧中为 1。
    
*   **Frame counter (FCnt)** ：一个 16 位计数器，数据上下行中分别称为 uplink 计数器 和 downlink 计数器，FCnt 设计的目的是，防止**重放攻击**，即当接受方收到的 FCnt 比之前收到的 FCnt 小，接收方会丢弃这个数据包。
    
*   **MAC Frame Payload Encryption (FRMPayload)**: 如果数据帧携带有 payload，在计算 MIC 之前，需要使用 AES 进行加密，以保障数据的**机密性**。
    

### [](#设备激活 "设备激活")设备激活

终端节点在加入 LoRaWAN 之前需要进行激活，也成为入网。入网有两种方式: ABP(Activation by Personalization，个性化激活) 和 OTAA(Over-the-Air Activation，空中激活)。

#### [](#ABP "ABP")ABP

ABP(Activation by Personalization，个性化激活) 是一种简单的入网机制，DevAddr、NwkSKey 和 AppSKey 硬编码保存在终端节点中，服务端也保存有这三个删除。这三个参数在整个生命周期中保持不变。这种入网方式不太安全，适合搭建私有网络。

每一个终端节点都有一个 DevEUI（Device Extended Unique Identifier，设备扩展唯一标识），最常见的做法是，取 MCU 的 SN（Serial Number，序列号），经过某种算法得到 64 位的 DevEUI。然后根据 DevEUI 采用某种算法得到 DevAddr、NwkSKey 和 AppSKey。如果采用的算法过于简单，能够被攻击者猜解出来，攻击者便可以利用这些值**伪造**出虚无的终端节点。

#### [](#OTAA "OTAA")OTAA

OTAA(Over-the-Air Activation，空中激活) 需要与网络服务器协商产生所需的密钥 NwkSKey 和 AppSKey。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200206181809995.png)

```
AppSKey = AES(AppKey, 0x1 + AppNonce + NetID + DevNonce)
NwkSKey = AES(AppKey, 0x2 + AppNonce + NetID + DevNonce)
```

*   AppKey：AppKey 是一个 128 位的 AES-128 key 。
*   DevNonce (JoinRequest) 和 AppNonce (JoinAccept) ：是入网中引入的两个随机数，用于抵御重放攻击。
*   Network Identifier (NetID) ：在同一个 LoRaWAN 中 所有的终端节点共享一个 NetID。
*   End Device Address (DevAddr) ：1 个 32 位的标识，在当前网络中的终端节点唯一标识，相当于会话 ID。

### [](#LoRaWAN-v1-1-中的安全改进 "LoRaWAN v1.1  中的安全改进")LoRaWAN v1.1 中的安全改进

1.  从 Network server 中独立出了 Join Server ，用于生成和管理密钥。Network server 不在处理 AppSKey 。
2.  新加入了一个根密钥 NwkKey ，现有两个根密钥 AppKey 以及 NwkKey。
3.  在网络层和应用层使用独立的随机数，位数从 16 位提高到了 32 位。
    
4.  会话密钥从 2 个增加到 5 个。
    

### [](#安全风险与威胁 "安全风险与威胁")安全风险与威胁

LoRa 声称是一个安全的物联网协议，也得到了广泛的应用。LoRa 安全更多的是密钥的安全。密钥可以通过以下几种方式获取。

*   通过逆向从固件中获取
    
    [使用 UART 或者 SPI 接口通过监听或者伪造 MCU 与 LoRa 模块的通信](https://core.ac.uk/download/pdf/84932416.pdf)；充设备中提取出固件，或从互联网上获取到固件，然后逆向分析出密钥；
    
*   设备标签
    
    不少设备上的标签以文本或二维码记录着 DevEUI、AppKey 等敏感信息，如果部署后没有移除，攻击者通过物联接触能够轻松获取到。
    
*   硬编码在开源代码中
    
    在开源代码中的密钥未经修改直接应用到产品中。
    
*   易猜解的密钥
    
    厂商在设计时，使用过于简单的算法来实现，容易被攻击者猜解出来。例如，AppKey = DevEUI + AppEUI or AppKey = AppEUI + DevEUI、 AppKey = DevEUI 、所有的设备采用相同的 AppEUI 等。
    
*   网络服务器中使用默认密码或弱口令
    
    使用 shodan 或 zoomeye 等检索出暴露在互联网上的网络服务器，不少的服务器使用了默认密码，如 admin:admin。攻击者可以在登录后获取到密钥
    
*   服务器存在安全漏洞
    
    服务器操作系统或者其他组件存在安全漏洞被入侵也可能导致密钥的泄露。
    
*   设备商被攻击
    
    设备商的网络被攻击导致密钥泄露。
    
*   设备 / 设施部署机制
    
    部署时常用计算机、手机 APP 或其他专用设备进行配置，密钥可能在部署后残留在部署的计算机、手机或其他专用设备中。
    
*   文件泄露
    
    设备制造商通常把密钥存储在文件中，并通过邮件等方式分享给客户。这些文件被多人经手或因管理不当导致密钥文件泄露。
    
*   服务提供商信息泄露
    
    网络服务器与应用服务器中存储有 APPKeys，密钥可能以文件形式被备份或保存在数据库中等。服务提供商数据泄露可导致用户的密钥被泄露。
    

### [](#离线密钥攻击 "离线密钥攻击")离线密钥攻击

Appkey 可以使用字典进行暴力破解，实现攻击有以下几种方式：

#### [](#使用一个-JoinRequest-以及-一个JoinAccept-或者-两个JoinRequest-JoinAccept-消息 "使用一个 JoinRequest 以及 一个JoinAccept  或者 两个JoinRequest/JoinAccept 消息")使用一个 JoinRequest 以及 一个 JoinAccept 或者 两个 JoinRequest/JoinAccept 消息

通过计算并对比入网消息中的 MIC 来寻找 appkey。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200207101925911.png)

首先使用字典中的 Appkey 计算出 MIC， 然后和 JoinRequest 消息中的 MIC 进行对比。如果 MIC 相同，这个 Appkey 可能真正的 Appkey，还需要进一步确认，因为 MIC 是取得前 4 个字节可能存在碰撞问题（不同的 Appkey 计算出相同的 MIC）。还需要进一步验证，使用这个 Appkey 按照下面的算法解密 JoinAccept 得到 MIC。

```
aes128_decrypt(AppKey, AppNonce | NetID | DevAddr | DLSettings | RxDelay | CFList | MIC)
```

最后，把之前计算出 MIC 与解密出的 MIC 进行对比。如果相同，Appkey 就被找到了。

使用两个 JoinRequest/JoinAccept 消息的方法也相似，首先使用字典中的 Appkey 计算出 MIC， 然后和 JoinRequest 中的 MIC 进行对比。不过这时需要执行两次，如果有同一个 Appkey 计算出的 MIC 与 两个 JoinRequest 消息中的都相同，Appkey 就被找到了。使用两个 JoinAccept 消息唯一不同的需要解密出 MIC 然后对比。

#### [](#使用一个-JoinAccept-消息和-一个数据消息 "使用一个 JoinAccept 消息和 一个数据消息")使用一个 JoinAccept 消息和 一个数据消息

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200207111442783.png)

使用 Appkey 解密 JoinAccept 消息得到 MIC 与直接计算的 MIC 相同，这个 Appkey 可能是真实的 Appkey。还需要进一步确认，对比 **JoinAccept 消息中解密得到的 DevAdr** 与 **数据消息中明文的 DevAddr** ，如果相同这个 Appkey 就是正确的。

#### [](#使用两个数据消息 "使用两个数据消息")使用两个数据消息

下图是，通过遍历 DevNonce and AppNonce 来验证 Appkey 和 NetID 的过程。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200207115019104.png)

使用这种方法效率很低，需要暴力破解出 DevNonce 、AppNonce、Appkey 以及 NetID。但是可以用来验证是否使用了脆弱的 Appkey（开源产品应用的或泄露）。

#### [](#密钥的其他问题 "密钥的其他问题")密钥的其他问题

*   在许多场景中，同一组设备往往使用相同的密钥。
*   在不部署时，没有替换默认的密钥，IOACTIVE 整理了一份[字典](https://github.com/IOActive/laf/blob/master/auditing/analyzers/bruteforcer/keys.txt)。
*   密钥不可更改，一旦密钥被攻陷，将无法进行修补。

### [](#在攻陷密钥后可执行的攻击方法 "在攻陷密钥后可执行的攻击方法")在攻陷密钥后可执行的攻击方法

#### [](#拒绝服务攻击 "拒绝服务攻击")拒绝服务攻击

*   发送大于真实 Fcnt 的 uplink 消息
    
    根据协议规范，服务器会拒绝接受 **Fcnt** 小于上次收到的 **FCnt** 。如果拥有会话密钥的攻击者发送一个大于真实设备的 FCnt 给服务器，那么真实的消息将会被拒绝接受。
    
*   重新生成会话密钥
    
    攻击者伪造 JoinRequest 请求发网络服务重新发起入网请求，产生了新的会话密钥，旧的会话密钥将失效。真实的设备节点使用旧的会话密钥生产的数据就会被服务器拒绝。
    
*   发送有效的 MAC 命令
    
    协议中定义了，MAC 命令用于网络管理，包含射频同步、信道管理、定时设置等。通过 FOpts 字段可以对通信的参数设置。攻击者可以伪装终端节点发送消息修改通信参数，当两端的通信参数不同时，通信将会受到影响。
    

#### [](#发送虚假消息 "发送虚假消息")发送虚假消息

这是最为严重的情况，攻击者在获取到密钥后可以伪装成终端节点给服务器发送伪造的数据。在特殊敏感的场景中将会带来巨大的危害，如误报天然气管道气压，对天然气管道进行物理破坏，主营方没能及时发现修复，给企业带来经济损失，给环境带来破坏甚至可能发展成灾难等。

### [](#防御：审计与入侵检测 "防御：审计与入侵检测")防御：审计与入侵检测

*   密钥生成规则要健壮，不易被攻击者猜解出来；
    
*   需要加强的密钥管理，防止密钥泄露；
    
*   在网络中添加入侵检测模块
    
    可以通过持续检测 **FCnt** 的值来检测攻击，这是因为攻击者发送伪造的消息或发起拒绝服务攻击， **FCnt** 的值会出现异样，在收到攻击者的消息后，真实中单节点发出的 **FCnt** 会小于等于攻击者发出的。
    
    通过分析流量，识别是否有同一设备出现平行会话的情况（devAddr），此时可能是攻击者通过重新入网发起了拒绝服务
    
    此外，还可以进一步分析数据中**被丢弃的数据包**被**丢弃的原因**来实现入侵检测。
    
*   尽量采用 OTTA 入网，因为使用 ABP 入网的终端节点中固化了密钥，密钥很容易被窃取。
    

### [](#相关信息 "相关信息")相关信息

*   运营商
    
    Loriot、TheThingsNetwork、Sertone、Archos PicoWan
    
*   常见芯片
    
    SX1301/SX1272/SX1276
    
*   开源服务器
    
    ChirpStack Application Server
    

### [](#参考-1 "参考")参考

*   [LoRaWAN 介绍](http://www.loraapp.com/lora-university-case/201701051107/)
*   [What is LoRaWAN](https://lora-alliance.org/sites/default/files/2018-04/what-is-lorawan.pdf)
*   [LoRaWAN® 1.0.3 Specification](https://lora-alliance.org/sites/default/files/2018-07/lorawan1.0.3.pdf)
*   [LoRaWAN Networks Susceptible to Hacking](https://act-on.ioactive.com/acton/attachment/34793/f-87b45f5f-f181-44fc-82a8-8e53c501dc4e/1/-/-/-/-/LoRaWAN%20Networks%20Susceptible%20to%20Hacking.pdf)
*   [https://github.com/IOActive/laf](https://github.com/IOActive/laf)

[](#LoRaWAN-安全实践 "LoRaWAN 安全实践")LoRaWAN 安全实践
--------------------------------------------

### [](#抓包分析 "抓包分析")抓包分析

待实现

### [](#入侵检测 "入侵检测")入侵检测

待实现

### [](#服务器使用弱密码 "服务器使用弱密码")服务器使用弱密码

使用 zoomeye 搜索暴露在公网的开源服务器 —— [ChirpStack Application Server](https://www.zoomeye.org/searchResult?q=%22ChirpStack%20Application%20Server%22%20%2Bport:%228080%22&t=all)，查询到 218 个，不少服务器使用了默认的账号 admin/admin。

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200207163234651.png)

![](https://delikely.github.io/2020/06/11/LoraWAN1-0-3-security/image-20200207163922450.png)