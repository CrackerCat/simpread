> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1971554-1-1.html)

> 看论坛内帖子 “从 Sandboxie 源码分析软件注册机制及逆向思路”（https://www.52pojie.cn/thread-1793118-1-1.html）发现最后加载驱动时要关闭驱动程序强制签名......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)whdfog _ 本帖最后由 whdfog 于 2024-10-11 19:35 编辑_  
看论坛内帖子 “从 Sandboxie 源码分析软件注册机制及逆向思路”（https://www.52pojie.cn/thread-1793118-1-1.html）发现最后加载驱动时要关闭驱动程序强制签名，而关闭强制签名会被部分游戏或程序检测到导致程序拒绝启动。如果能够不关闭强制签名去加载内核驱动，对系统的影响是最小的。  
（以下方法在 Windows 10 企业版 21H2 19044.5011 测试通过）  
在网络上查资料找到了这个项目 “HyperSine/Windows10-CustomKernelSigners”（https://github.com/HyperSine/Windows10-CustomKernelSigners），  
查看项目介绍（https://github.com/HyperSine/Windows10-CustomKernelSigners/blob/master/README.zh-CN.md#1-%E4%BB%80%E4%B9%88%E6%98%AFcustom-kernel-signers）  

> Custom Kernel Signers(CKS) 是 Windows10（可能从 1703 开始）支持的一种产品策略。这个产品策略的全名是 CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners，它允许用户自定义内核代码证书，从而使得用户可以摆脱 “驱动必须由微软签名” 的强制性要求。  
> 如果一个 Windows10 PC 满足下列条件：  
>     1. 产品策略 CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners 是开启的。 （也许 CodeIntegrity-AllowConfigurablePolicy 也要开启。）  
>     2. SecureBoot 也是开启的。  
> 那么任何拥有该 PC 的 UEFI Platform Key 的人都可以自定义内核代码证书。这意味着，在不开启调试模式、不开启 TestSigning、不关闭 DSE 的情况下，他可以使系统允许自签名驱动的加载。

于是跟随项目内教程（https://github.com/HyperSine/Windows10-CustomKernelSigners/blob/master/asset/build-your-own-pki.zh-CN.md）创建了 3 个证书（localhost-root-ca，localhost-km，localhost-pk），localhost-root-ca 作为根证书用于签发 localhost-km 和 localhost-pk，localhost-km 用于给要加载的内核驱动签名，localhost-pk 作为 PK 证书用于导入 UEFI 固件。  
为了导入自己 PK 证书需要先将安全启动重置为 Setup Mode，重置后现有 PK 证书会被清除。如果电脑的 BIOS 设置可以直接导入 crt 格式 PK 证书直接导入就行，不必重置进 Setup Mode。部分 BIOS 只支持通过 AUTH 文件导入 PK 证书，这种就需要重置为 Setup Mode。  
生成 AUTH 文件需要 efitools，efitools 只能在 Linux 系统上使用。Windows 电脑可以通过 WSL 1（不需要 WSL 2）安装一个 Ubuntu，在 Ubuntu 内安装 efitools 的命令是  
[Shell] _纯文本查看_ _复制代码_

```
sudo apt install efitools

```

efitools 的软件包下载网页：  
https://packages.ubuntu.com/zh-cn/jammy/amd64/efitools/download  
efitools 二进制文件下载网页：  
https://archlinux.org/packages/extra/x86_64/efitools/（下载地址：https://archlinux.org/packages/extra/x86_64/efitools/download/）  
以下是在 WSL 内用 efitools 生成 AUTH 文件的代码的示例  
[Shell] _纯文本查看_ _复制代码_

```
cert-to-efi-sig-list -g "$(cat GUID.txt)" "localhost-pk.crt" "localhost-pk.esl"
sign-efi-sig-list -k localhost-pk.key -c localhost-pk.crt PK localhost-pk.esl PK.auth
sign-efi-sig-list -g "$(cat GUID.txt)" -k localhost-pk.key -c localhost-pk.crt PK /dev/null WipePK.auth
 
openssl x509 -inform der -outform pem -in "microsoft corporation kek 2k ca 2023.crt" -out "microsoft corporation kek 2k ca 2023.pem"
openssl x509 -inform der -outform pem -in "MicCorKEKCA2011_2011-06-24.crt" -out "MicCorKEKCA2011_2011-06-24.pem"
cert-to-efi-sig-list -g 77fa9abd-0359-4d32-bd60-28f4e78f784b "microsoft corporation kek 2k ca 2023.pem" "microsoft corporation kek 2k ca 2023.esl"
cert-to-efi-sig-list -g 77fa9abd-0359-4d32-bd60-28f4e78f784b "MicCorKEKCA2011_2011-06-24.pem" "MicCorKEKCA2011_2011-06-24.esl"
cat "kek 2k ca 2023.esl" "MicCorKEKCA2011_2011-06-24.esl" > "kek.esl"
sign-efi-sig-list -k localhost-pk.key -c localhost-pk.crt KEK kek.esl KEK.auth
 
openssl x509 -inform der -outform pem -in "MicCorUEFCA2011_2011-06-27.crt" -out "MicCorUEFCA2011_2011-06-27.pem"
openssl x509 -inform der -outform pem -in "microsoft uefi ca 2023.crt" -out "microsoft uefi ca 2023.pem"
openssl x509 -inform der -outform pem -in "MicWinProPCA2011_2011-10-19.crt" -out "MicWinProPCA2011_2011-10-19.pem"
openssl x509 -inform der -outform pem -in "windows uefi ca 2023.crt" -out "windows uefi ca 2023.pem"
cert-to-efi-sig-list -g 77fa9abd-0359-4d32-bd60-28f4e78f784b "MicCorUEFCA2011_2011-06-27.pem" "MicCorUEFCA2011_2011-06-27.esl"
cert-to-efi-sig-list -g 77fa9abd-0359-4d32-bd60-28f4e78f784b "microsoft uefi ca 2023.pem" "microsoft uefi ca 2023.esl"
cert-to-efi-sig-list -g 77fa9abd-0359-4d32-bd60-28f4e78f784b "MicWinProPCA2011_2011-10-19.pem" "MicWinProPCA2011_2011-10-19.esl"
cert-to-efi-sig-list -g 77fa9abd-0359-4d32-bd60-28f4e78f784b "windows uefi ca 2023.pem" "windows uefi ca 2023.esl"
cert-to-efi-sig-list -g "$(cat GUID.txt)" "cert_uefidb.crt" "cert_uefidb.esl"
cat "MicCorUEFCA2011_2011-06-27.esl" "microsoft uefi ca 2023.esl" "MicWinProPCA2011_2011-10-19.esl" "windows uefi ca 2023.esl" "cert_uefidb.esl" > "db.esl"
sign-efi-sig-list -k localhost-pk.key -c localhost-pk.crt DB db.esl DB.auth
 
sign-efi-sig-list -k localhost-pk.key -c localhost-pk.crt DBX x64_DBXUpdate.bin DBX.auth

```

  
需要在运行 Shell 的目录下创建一个 GUID.txt，文件内容是自己生成的 8-4-4-4-12 格式的 GUID（如微软的所有者 GUID：77fa9abd-0359-4d32-bd60-28f4e78f784b）。  
Shell 的目录下还需要有 base-64 编码的证书文件 localhost-pk.crt 及其私钥 localhost-pk.key。  
涉及到的证书文件在文章最后有下载地址。  
设置好 PK 证书后就要构建内核代码证书规则。一个现成的内核代码证书规则文件（https://www.geoffchappell.com/notes/windows/license/selfsign.xml.htm）。  
这个规则文件需要包含自己生成的 localhost-km 证书。  
转换 localhost-pk.crt 为签名可用的 localhost-pk.pfx 的代码  
[Shell] _纯文本查看_ _复制代码_

```
openssl pkcs12 -export -in localhost-pk.crt -inkey localhost-pk.key -out localhost-pk.pfx

```

  
导出 localhost-pk.pfx 时会提示输入密码，全部留空直接回车即可。  
规则生成并签名后需要移动到当前系统的 EFI 目录下（以下提供的命令可自动完成）。  
以下是构建内核代码证书规则的 PowerShell 代码  
[PowerShell] _纯文本查看_ _复制代码_

```
if(!([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] 'Administrator')) {
Start-Process -FilePath PowerShell.exe -Verb RunAs -ArgumentList ("-NoExit",("cd {0} ;" -f $PSScriptRoot),("`"$($MyInvocation.MyCommand.Path)`" $($MyInvocation.UnboundArguments)"))
Exit
}
 
Write-Host "本脚本同目录下需要有signtool.exe/localhost-km.crt/localhost-pk.pfx 3个文件"
Set-Location -path "$(Get-Location)"
New-CIPolicy -FilePath SiPolicy.xml -Level RootCertificate -ScanPath C:\windows\System32\
Add-SignerRule -FilePath .\SiPolicy.xml -CertificatePath localhost-km.crt -Kernel -Update -Supplemental
Set-RuleOption -FilePath .\SiPolicy.xml -Option 2 -Delete   #2 - Required:WHQL
Set-RuleOption -FilePath .\SiPolicy.xml -Option 3 -Delete   #3 - Enabled:Audit Mode (Default)
Set-RuleOption -FilePath .\SiPolicy.xml -Option 12 -Delete  #12 - Required:Enforce Store Applications
Set-RuleOption -FilePath .\SiPolicy.xml -Option 6   #6  - Enabled：Unsigned System Integrity Policy
Set-RuleOption -FilePath .\SiPolicy.xml -Option 9   #9  - Enabled：Advanced Boot Options Menu
Set-RuleOption -FilePath .\SiPolicy.xml -Option 10  #10 - Enabled：Boot Audit on Failure
Set-RuleOption -FilePath .\SiPolicy.xml -Option 17  #17 - Enabled: Allow Supplemental Policies
Set-CIPolicyVersion -FilePath .\SiPolicy.xml -Version 10.0.0.0
Write-Host ""
Write-Host "    "
Write-Host "    "
Write-Host "      "
Write-Host " "
Write-Host "    "
Write-Host "    "
Write-Host "      "
Write-Host " "
Write-Host ""
Write-Host "          "
Write-Host "          "
Write-Host ""
Read-Host -Prompt "修改SiPolicy.xml完成后按任意键继续"
ConvertFrom-CIPolicy -XmlFilePath .\SiPolicy.xml -BinaryFilePath .\SiPolicy.bin
signtool.exe sign /v /p7 . /p7co 1.3.6.1.4.1.311.79.1 /fd sha256 /ac localhost-root-ca.crt /f localhost-pk.pfx SiPolicy.bin
Move-Item -Force -Path .\SiPolicy.bin.p7 -Destination .\SiPolicy.p7b
mountvol X: /s
Copy-Item -Force -Path .\SiPolicy.p7b -Destination X:\EFI\Microsoft\Boot\
Write-Host "(EFI)SiPolicy.p7b签名状态："
certutil.exe -asn X:\EFI\Microsoft\Boot\SiPolicy.p7b
mountvol X: /d
Read-Host -Prompt "已完成，请按任意键继续"
$host.SetShouldExit(0)

```

  
需要根据脚本的提示修改生成的 SiPolicy.xml 内 Signers 和 AllowedSigners 项的内容。  
signtool.exe 是 Windows Software Development Kit (SDK) 的一部分，可以在网络上单独下载到。  
据项目介绍（https://github.com/HyperSine/Windows10-CustomKernelSigners/blob/master/README.zh-CN.md#25-%E5%BC%80%E5%90%AFcustomkernelsigners）  

> CKS 的开关保存在 HKLM\SYSTEM\CurrentControlSet\Control\ProductOptions 键的 ProductPolicy 值里。  
> 尽管管理员可以修改这个值，但是这个值在被修改后会立即恢复原状。这是因为在内核初始化完后，这个值只是内核里一个变量的映射，只有通过 ExUpdateLicenseData 这个内核 API 才能修改。而这个 API 只能在内核里被调用，或者通过 NtQuerySystemInformation 的 SystemPolicyInformation 功能号间接调用。很遗憾的是后者只有 Protected Process 才能 成功 调用。  
> 所以我们只能在内核还尚未初始化完的时候修改 CKS 开关。有这个机会吗？有，Windows 的 Setup Mode 可以给我们提供这个机会。  
> 我已经写了一个程序来帮助我们打开 CKS，二进制程序是 EnableCKS.exe。EnableCKS.exe 会自动启动并开启  
> CodeIntegrity-AllowConfigurablePolicy  
> CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners

但微软在新系统中删除了 CodeIntegrity-AllowConfigurablePolicy 这个产品策略，运行 EnableCKS.exe 后程序无法开启 CodeIntegrity-AllowConfigurablePolicy 会导致无限重启。（如果已经无限重启，可进 PE 的注册表编辑器，点击 HKEY_LOCAL_MACHINE，再点击文件 - 加载配置单元，选择 C:\Windows\System32\config 文件夹下的 SYSTEM 文件，修改加载的注册表 HKEY_LOCAL_MACHINE\SYSTEM\Setup 下 SetupType 的值为 0 即可。修改完成后记得卸载配置单元保存修改。）  
我用了另一个项目 valinet/ssde（https://github.com/valinet/ssde），这个项目作者编译的 ssde_enable.exe 解决了这个错误，可以正常开启 CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners 并重启。  
据项目介绍（https://github.com/HyperSine/Windows10-CustomKernelSigners/blob/master/README.zh-CN.md#26-customkernelsigners%E6%8C%81%E4%B9%85%E5%8C%96）  

> CustomKernelSigners 持久化  
> 重新进入正常模式后，你应该就可以加载由 localhost-km.pfx 签署的驱动了。但是别高兴得太早，大约在 10 分钟之内，CKS 会被 sppsvc 服务重置为关闭，除非你的 Windows10 是中国政府特供版。但不用担心，关闭还得等重启后才会实际生效。  
> 所以我们得趁这个机会，加载自己编写的驱动，通过不断调用 ExUpdateLicenseData 来持久化 CKS。

ssde 项目的作者同样编译了一个驱动 ssde.sys 来持久化 CKS，但是这个驱动没有签名。需要通过以下命令用 localhost-km 证书签名 ssde.sys  
[Shell] _纯文本查看_ _复制代码_

```
signtool sign /v /fd sha1 /ac localhost-root-ca.crt /f localhost-km.pfx /tr "http://timestamp.digicert.com" ssde.sys
signtool sign /v /fd sha256 /as /ac localhost-root-ca.crt /f localhost-km.pfx /tr "http://timestamp.digicert.com" ssde.sys

```

  
将签名完成的 ssde.sys 复制到 %SystemRoot%\System32\drivers \ 目录下（一般都是 C:\Windows\System32\drivers\）。  
然后用管理员权限运行以下命令  
[Shell] _纯文本查看_ _复制代码_

```
sc create ssde binpath=%SystemRoot%\System32\drivers\ssde.sys type=kernel start=auto error=normal

```

  
注意命令中的 %SystemRoot% 最好不要替换为 %Windir%，尽管 %SystemRoot% 和 %Windir% 的值应该都是 C:\Windows\，但个人实测使用 %Windir% 会导致驱动无法启动，而用 %SystemRoot% 驱动就可以启动。  
根据微软官方文档（https://learn.microsoft.com/zh-cn/dotnet/api/system.serviceprocess.servicestartmode#fields），start 参数最好是 auto，设置为 boot 可能会导致 ssde.sys 驱动没有对应的设备从而无法启动。  
现在再重启电脑，产品策略 CodeIntegrity-AllowConfigurablePolicy-CustomKernelSigners 应该会保持开启（注册表 HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\CI\Protected 的值为 1 即为开启），也就可以加载自己签名的内核驱动了。以后启动 Windows 系统 CustomKernelSigners 策略应该都会保持开启。  
微软 KEK 和 DB 证书（https://learn.microsoft.com/zh-cn/windows-hardware/manufacture/desktop/windows-secure-boot-key-creation-and-management-gu[IDA](https://www.52pojie.cn/thread-1874203-1-1.html)nce）：  
Microsoft Corporation KEK CA 2011  
    SHA-1 证书哈希：31 59 0b fd 89 c9 d7 4e d0 87 df ac 66 33 4b 39 31 25 4b 30。  
    SignatureOwner GUID：{77fa9abd-0359-4d32-bd60-28f4e78f784b}。  
    Microsoft 会向合作伙伴提供证书，可将该证书添加为 EFI_CERT_X509_GUID 或 EFI_CERT_RSA2048_GUID 类型的签名。  
    可从以下位置下载 Microsoft KEK 证书：https://go.microsoft.com/fwlink/?LinkId=321185  
Microsoft Corporation KEK 2K CA 2023  
    SHA-1 证书哈希：45 9a b6 fb 5e 28 4d 27 2d 5e 3e 6a bc 8e d6 63 82 9d 63 2b。  
    SignatureOwner GUID：{77fa9abd-0359-4d32-bd60-28f4e78f784b}。  
    Microsoft 会向合作伙伴提供证书，可将该证书添加为 EFI_CERT_X509_GUID 或 EFI_CERT_RSA2048_GUID 类型的签名。  
    可从以下位置下载 Microsoft KEK 证书：https://go.microsoft.com/fwlink/?linkid=2239775  
Microsoft Windows Production PCA 2011  
    SHA-1 证书哈希：58 0a 6f 4c c4 e4 b6 69 b9 eb dc 1b 2b 3e 08 7b 80 d0 67 8d。  
    SignatureOwner GUID：{77fa9abd-0359-4d32-bd60-28f4e78f784b}。  
    Microsoft 会向合作伙伴提供证书，可将该证书添加为 EFI_CERT_X509_GUID 或 EFI_CERT_RSA2048_GUID 类型的签名。  
    Windows Production PCA 2011 可以从以下位置下载：https://go.microsoft.com/fwlink/p/?linkid=321192  
Windows UEFI CA 2023  
    SHA-1 证书哈希：45 a0 fa 32 60 47 73 c8 24 33 c3 b7 d5 9e 74 66 b3 ac 0c 67。  
    SignatureOwner GUID：{77fa9abd-0359-4d32-bd60-28f4e78f784b}。  
    Microsoft 会向合作伙伴提供证书，可将该证书添加为 EFI_CERT_X509_GUID 或 EFI_CERT_RSA2048_GUID 类型的签名。  
    可从以下位置下载 Windows UEFI CA 2023：https://go.microsoft.com/fwlink/?linkid=2239776  
Microsoft Corporation UEFI CA 2011  
    SHA-1 证书哈希：46 de f6 3b 5c e6 1c f8 ba 0d e2 e6 63 9c 10 19 d0 ed 14 f3。  
    SignatureOwner GUID：{77fa9abd-0359-4d32-bd60-28f4e78f784b}。  
    Microsoft 会向合作伙伴提供证书，可将该证书添加为 EFI_CERT_X509_GUID 或 EFI_CERT_RSA2048_GUID 类型的签名。  
    可从以下位置下载 Microsoft Corporation UEFI CA 2011：https://go.microsoft.com/fwlink/p/?linkid=321194  
Microsoft UEFI CA 2023  
    SHA-1 证书哈希：b5 ee b4 a6 70 60 48 07 3f 0e d2 96 e7 f5 80 a7 90 b5 9e aa。  
    SignatureOwner GUID：{77fa9abd-0359-4d32-bd60-28f4e78f784b}。  
    Microsoft 会向合作伙伴提供证书，可将该证书添加为 EFI_CERT_X509_GUID 或 EFI_CERT_RSA2048_GUID 类型的签名。  
    可从以下位置下载 Microsoft UEFI CA 2023：https://go.microsoft.com/fwlink/?linkid=2239872  
从 Microsoft 下载最新的 UEFI 吊销列表【禁止的签名数据库（DBX）】：https://www.uefi.org/revocationlistfile  
其他参考文章：  
Licensed Driver Signing in Windows 10（https://www.geoffchappell.com/notes/windows/license/customkernelsigners.htm）  
实施 Secure Boot（https://cascade.moe/posts/secure-boot/）  
Properly install into a new system #4 （https://github.com/valinet/ssde/issues/4#issuecomment-1926593978）