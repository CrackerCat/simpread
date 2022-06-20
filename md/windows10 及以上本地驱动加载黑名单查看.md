> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273316.htm)

> windows10 及以上本地驱动加载黑名单查看

之前测试程序一直用上海域联的签名进行测试，后来有一天发现用不了，突然意识到 windows 开始使用证书黑名单策略了。它的这个策略不仅会在系统启动以后拦截驱动加载，还可以拦截 boot 启动的驱动。它的这个策略是通过系统更新下发的。如果你的系统已经加载了黑签名签过的驱动，策略更新将会失败，这样有效的避免了因拦截驱动加载可能引发的一系列问题。  
昨天看到论坛上有人讨论这个问题，[原帖在这里](https://bbs.pediy.com/thread-272998.htm)，评论区有人提到了这个问题且给出了[微软的官方说明](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/microsoft-recommended-driver-block-rules)  
既然这个文件更新到本地的话，它到底保存在哪里呢？可不可以查看？可不可以修改呢？带着这些问题我出发了。最后在 **C:\Windows\System32\CodeIntegrity\driversipolicy.p7b** 下找到了。  
结果发现并不是 XML 格式。后来在 github 上找到了把二进制转化为 XML 的脚本。  
1、启用脚本执行 **set-ExecutionPolicy RemoteSigned**  
![](https://bbs.pediy.com/upload/attach/202206/444209_VC8SHZZR4AUHHEH.jpg)  
**注：必须使用管理员权限**

 

2、导入脚本，我的脚本文件夹在桌面。  
**cd C:\Users\test\Desktop  
Import-Module .\WDACTools**

 

3、执行脚本函数转换格式 **ConvertTo-WDACCodeIntegrityPolicy**  
然后依次输入要转换的源文件路径和转换后的文件路径  
![](https://bbs.pediy.com/upload/attach/202206/444209_J22AXQ7HNRQNV4T.jpg)  
![](https://bbs.pediy.com/upload/attach/202206/444209_BZWAJ83ENS3AKC5.jpg)  
到这里已经转换成功了。

 

XML 格式转 p7b 的话，直接使用原生命令就可以  
**ConvertFrom-CIPolicy -XmlFilePath XML 路径 -BinaryFilePath p7b 文件路径**  
修改完以后重新转换成 p7b 格式替换掉原来的文件就可以了。

 

参考来源  
[启用脚本执行](https://zhuanlan.zhihu.com/p/493496089)  
[xml 格式转 p7b 格式](https://docs.microsoft.com/en-us/powershell/module/configci/convertfrom-cipolicy?view=windowsserver2022-ps)  
[脚本 github 地址](https://github.com/mattifestation/WDACTools)

[【看雪培训】《Adroid 高级研修班》2022 年春季班招生中！](https://bbs.pediy.com/thread-271992.htm)

最后于 4 天前 被简单未遂编辑 ，原因：

[#开源分享](forum-41-1-134.htm)

上传的附件：

*   [WDACTools.7z](javascript:void(0)) （26.88kb，39 次下载）