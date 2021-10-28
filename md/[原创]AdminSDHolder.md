> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269995.htm)

> [原创]AdminSDHolder

本次安全科普为大家介绍 AD 域中的 AdminSDHolder，与 AdminSDHolder 相关的特性频繁被攻击者利用来进行留后门等操作，在检查 AD 域安全时 AdminSDHolder 相关属性也是排查的重点。

0x00 域内受保护的用户和组
---------------

在 Active Directory 中，一些高权限的用户和组被视为受保护的对象

 

通常对于受保护的用户和组，权限的设置和修改是由一个自动过程来完成的，这样才能保证在对象移动到其他目录时，对象的权限也始终保持一致

 

不同系统版本的域控制器上受保护的用户和组也不同，具体可以参考微软文档：[APPENDIX-C--PROTECTED-ACCOUNTS-AND-GROUPS-IN-ACTIVE-DIRECTORY](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory)

 

也可以使用 adfind 来查询

```
Adfind.exe -f "&(objectcategory=group)(admincount=1)" -dn
 
Adfind.exe -f "&(objectcategory=user)(admincount=1)" -dn

```

![](https://bbs.pediy.com/upload/attach/202110/931257_X5M8XYE6F5W5N2U.png)

### 1. AdminSDHolder

AdminSDHolder 对象的目的是为域内受保护的用户和组提供权限的 “模板”，其在 LDAP 上的路径为：`CN=AdminSDHolder,CN=System,DC=<domain_component>,DC=<domain_component>`

 

![](https://bbs.pediy.com/upload/attach/202110/931257_JPGCRY3RGNSQUP6.png)

 

AdminSDHolder 由 Domain Admins 组拥有，默认情况下，EA 可以对任何域的 AdminSDHolder 对象进行更改，域的 Domain Admins 和 Administrators 组也可以进行更改

 

尽管 AdminSDHolder 的默认所有者是域的 Domain Admins 组，但是 Administrators 或 Enterprise Admins 的成员可以获取该对象的所有权

 

![](https://bbs.pediy.com/upload/attach/202110/931257_UZK4YVPU7WWSTXG.png)

### 2. SDProp

SDProp 是一个进程，该进程每 60 分钟（默认情况下）在包含域的 PDC 模拟器（PDCE）的域控制器上运行

 

SDProp 将域的 AdminSDHolder 对象的权限与域中受保护的帐户和组的权限进行比较。如果任何受保护帐户和组的权限与 AdminSDHolder 对象的权限不匹配，则将受保护帐户和组的权限重置为与域的 AdminSDHolder 对象的权限匹配

0x01 利用
-------

既然默认每 60 分钟 SDProp 会将受保护帐户和组的权限重置为与域的 AdminSDHolder 对象的权限匹配，那么我们完全可以对 AdminSDHolder 添加 ACL 来留后门

 

利用权限：

1.  对 AdminSDHolder 有`WriteDACL`权限的账户

### 1. 添加 ACL

#### (1) Admod

```
.\Admod.exe -b "CN=AdminSDHolder,CN=System,DC=testad,DC=local" "SD##ntsecuritydescriptor::{GETSD}{+D=(A;;GA;;;testad\test1)}"

```

![](https://bbs.pediy.com/upload/attach/202110/931257_MBN4MDX2KZ6RD6Z.png)

#### (2) PowerView

这里有一个坑点，PowerView 在 github 的主分支中很多功能是没有的，所以推荐使用 Dev 分支

 

[PowerView_dev](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1)

```
Import-Module .\PowerView.ps1
 
Add-ObjectAcl -TargetADSprefix 'CN=AdminSDHolder,CN=System' -PrincipalSamAccountName test1 -Verbose -Rights All

```

### 2. 执行 SDProp

除了等待默认的 60 分钟后 SDProp 自动执行，我们还可以用以下两种方法来更快速的执行 SDProp

#### (1) 修改默认时间

如果需要修改 60min 的执行时间间隔，只需要在`HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters`中添加或修改`AdminSDProtectFrequency`的值

 

该值的范围是从 60 到 7200，单位为秒，键类型为 DWORD

 

可以直接使用命令行更改：

```
reg add hklm\SYSTEM\CurrentControlSet\Services\NTDS\Parameters /v AdminSDProtectFrequency /t REG_DWORD /d 600

```

如果需要恢复为默认的 60min，则可以在注册表中删除`AdminSDProtectFrequency`这一项

#### (2) 手动执行

首先启动 **Ldp.exe**，然后选择菜单栏中 **“连接”** ---> **“连接”**

 

![](https://bbs.pediy.com/upload/attach/202110/931257_HA2KXMMFSPD62G3.png)

 

输入拥有 PDC 模拟器（PDCE）角色的 DC 的 FQDN 或 IP：

 

![](https://bbs.pediy.com/upload/attach/202110/931257_W8XYAAC94FUUAVH.png)

 

选择菜单栏菜单栏中 **“连接”** ---> **“绑定”**

 

![](https://bbs.pediy.com/upload/attach/202110/931257_TK9FN3PGCKFWUCH.png)

 

在**绑定**窗口中输入有权修改 rootDSE 对象的用户帐户的凭据，或者直接已当前已登录的用户身份绑定

 

![](https://bbs.pediy.com/upload/attach/202110/931257_WDXEPSNURJGV5PM.png)

 

选择菜单栏菜单栏中 **“浏览”** ---> **“修改”**

 

![](https://bbs.pediy.com/upload/attach/202110/931257_ZHGY4M5QAE722CY.png)

 

在修改窗口这里针对不同版本的域控制器有不同的情况：

*   域控为 Windows Server 2008: 将 **“DN”** 字段留空。在 **“编辑条目属性”** 字段中，输入 **FixUpInheritance**，在 **“值”** 字段中，输入 **Yes**。单击**输入**填充条目列表
*   域控为 Windows Server 2008 R2 或 Windows Server 2012: 将 **“DN”** 字段留空。在 **“编辑条目属性”** 字段中，输入 **RunProtectAdminGroupsTask**，在 **“值”** 字段中，输入 **1**。单击**输入**填充条目列表

![](https://bbs.pediy.com/upload/attach/202110/931257_W5UBMCXRPM3MC3W.png)

 

最后在 **“修改”** 对话框中点击 **“运行”** 即可

### 3. 添加特权

SDProp 执行后，这些受保护的用户和组就被同步与 AdminSDHolder 一样的 ACL

 

![](https://bbs.pediy.com/upload/attach/202110/931257_N3M6CVSNFADP4C2.png)

 

现在我们已经对这些特权组 / 用户拥有 FC 权限了，以添加域管组成员为例：

 

![](https://bbs.pediy.com/upload/attach/202110/931257_GF7JWTHBTG7MBJJ.png)

0x02 防御与检测
----------

该攻击手法的核心点在于需要修改 AdminSDHolder 的 ACL，因此我们只需要检测对 AdminSDHolder 的 ACL 的修改行为即可，可以通过 5136 日志来监控

 

![](https://bbs.pediy.com/upload/attach/202110/931257_XR2D2RKURTABTGU.png)

[2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

[#资讯](forum-45-1-67.htm)