> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/mN5M9-P0g6x_4NqTKbO2Sg)

1

以上文章由来自 OPPO 子午互联网安全实验室【heeeeen】有赏投稿，也欢迎广大朋友继续投稿，详情可点击 [OSRC 重金征集文稿！！！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247484531&idx=1&sn=6925d63e60984c8dccd4b0162dd32970&chksm=fa7b053fcd0c8c2938d1c5e0493a20ec55c2090ae43419c7aef933bcc077692b1997f4710afa&scene=21#wechat_redirect)了解~~  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8IVS9MaRlldTa0CMfQAicIq8NUJHAgmBxzr4RQqv7Ucac9blQAPs3znEW5YsWZj7rRtCUz8YWBXkew/640?wx_fmt=jpeg)

  

0x00 介绍

所谓攻击面，既是系统处理正常输入的各种入口的总和，也是未授权攻击者进入系统的入口。在漏洞挖掘中，攻击面是最为核心的一个概念，超越各种流派、各种专业方向而存在，无论 Web 还是二进制，也无论 Windows 还是 Android，总是在研究如何访问攻击面，分析与攻击面有关的数据处理代码，或者 Fuzz 攻击面。漏洞挖掘工作总是围绕着攻击面在进行。  

就 Android 系统和 App 而言，通常所知的本地攻击面无外乎暴露组件、binder 服务、驱动和套接字，远程攻击面无外乎各种通信协议、文件格式和网页链接。然而，实际漏洞案例总是鲜活的，总有一些鲜为人知的攻击面，出现的漏洞颇为有趣，甚至很有实际利用价值，所以这里准备写一个系列，记录发现的一些有趣漏洞。先来谈谈与用户发生交互的对话框。

0x01 人机交互对话框

在 AOSP 的漏洞评级标准中，中危漏洞和高危漏洞的评级都有这么一条：

* High：Local bypass of user interaction requirements for any developer or security settings modifications

* Moderate: Local bypass of user interaction requirements (access to functionality that would normally require either user initiation or user permission)

系统中需要用户交互进行确认的地方，一旦可以绕过修改安全设置或者产生安全影响，即认为出现了漏洞，这里与用户交互的对话框就是一种特殊的攻击面。

0x02 用户确认绕过

在与用户交互的对话框中，拨打电话对话框通常比较特殊，开发人员容易忽视其被外部直接调用绕过用户交互后的安全影响。Android 历史上就曾出现这种漏洞，如 CVE-2013-6272。  

然而在一些流行的社交网络软件中，其 VoIP 拨号功能也容易出现此类漏洞，恶意程序可以绕过用户交互直接拨号到另一个用户，这样另一个用户就可以监听受害用户手机的麦克风，使受害用户的隐私泄露。

俄罗斯知名的社交软件 VK.COM 曾出现这样一个漏洞: Bypass User Interaction to initiate a VoIP call to Another User

主要原因在于

`com.vkontakte.android.LinkRedirActivity` 可以传入一个 Provider，而这个 Provider 中可以指定其他用户的 id

```
ContentValues cr_vals = new ContentValues();
        cr_vals.put("data1", 458454771); //target user_id
        cr_vals.put("name", "unused");
```

当启动该 Activity 的时候就会直接向指定 id 的用户拨号。问题在于，作为拨号过程，这里需要设计一个对话框让用户确认。刚开始 VK.COM 并不认为这是一个漏洞，后来使用 HackerOne 的仲裁机制，邀请其他 Android 领域的知名安全专家一起参加讨论才说服厂商，修复最终添加了一个确认对话框，用户确认后才允许拨号。

同样，Line 也出现过类似的漏洞，绕过用户交互向另一用户拨打 Audio Phone，Line 将这个漏洞归为 Authentication Bypass，同样在修复中加入了一个用户确认对话框。

0x03 用户确认欺骗 

除了对话框绕过以外，攻击者还可以在对话框中显示欺骗的内容，达到 clickjacking 的效果。

CVE-2017-13242: 蓝牙配对对话框欺骗

这个漏洞发生在我们经常使用的蓝牙配对对话框中，如下图是正常的蓝牙配对对话框：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVS9MaRlldTa0CMfQAicIq8R2K5JD9MWcv5nPvpdx6cpuv9DLmWWs30EPicZOapAQRNdXqIDvhlTjw/640?wx_fmt=png)

这里的 Angler 是对端的蓝牙配对设备。但是这个设备名是攻击者可控的，能否在这个攻击面上造成安全影响呢？

我们将对端的蓝牙配对设备名变长，并插入一个换行符，改为 “Pair with Angler \n to pair but NOT to access your contacts and call history”，那么蓝牙配对对话框显示为：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVS9MaRlldTa0CMfQAicIq8e1lfeg1KS5hlzLG5LcANfcOfjTbh5HN2cogxyzsS3ZIy6oCdKib3Eibw/640?wx_fmt=png)

虽然有一些奇怪，但用户一定会在是否共享通讯录和通话记录这个问题上比较纠结，“是的，我允许配对，但我不想共享通讯录和通话记录！” 于是，误导配对用户勾选下面的复选框，反而达到与用户期望相反的目的，正中攻击者的下怀。

这个漏洞于 2018 年 2 月修复，可惜 Google 的修复并不完全，只是在 Settings App 中的 Strings.xml 作了限制，不允许对端配对的蓝牙设备名传入，将配对对话框中的提示内容变成了一个固定的字符串。但是，这个修复并不完全，没有考虑到其他蓝牙连接的入口。

CVE-2018-9432: 蓝牙通讯录和短信访问协议对话框欺骗

  

Android 蓝牙协议还支持 PBAP 和 MAP Server，分别用于其他设备通过蓝牙访问手机的通讯录和短信，通常我们开车时通过蓝牙拨打电话、访问手机通讯录就使用了 PBAP 协议。通过 PBAP 协议和 MAP 协议，可以无需配对，直接在手机上弹出对话框，让用户确认是否访问通讯录和短信。PBAP 协议确认对话框如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVS9MaRlldTa0CMfQAicIq8vqbHiblpfJxuDCtoUQolyhibA5veL56a4IoTzzAxB6WxoyOykKFiccd2Q/640?wx_fmt=png)

但同样，临近攻击者可以将配对设备 heen-ras 重新命名，并插入许多的换行符

```
pi@heen-ras:~ $ sudo hciconfig hci0 name "heen-ras 想要访问你的通信录和电话簿， 要拒绝它吗？
>

…(skip)

>
> "
```

然后再次通过 PBAP 协议访问手机通信录，使用 "nOBEX" 这个脚本

```
pi@heen-ras:~/bluetooth-fuzz/nOBEX $ python3 examples/pbapclient.py <victim_bluetooth_address>
```

此时通讯录确认对话框如下：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IVS9MaRlldTa0CMfQAicIq8IIGvUze9xxjdG4Ka58xXJr71JnfibPRfw4ibDbYtAr4Oz1RXtIGlNuuQ/640?wx_fmt=png)

对比上下两图，确认对话框中的重要信息被隐藏了，并显示出相反的结果。在内部实际环境进行模拟检测，发现普通用户对此毫无招架，一般都会选择 “是”，结果通讯录被全部窃取。这个漏洞最终于 2018 年 7 月修复，使用的方法是在 BluetoothDevice 类中过滤配对蓝牙设备名中的 \ r\n 字符。

值得一提的是，同样是插入攻击设备蓝牙适配器的换行符，还可以远程注入蓝牙配置文件，攻击者可以设置蓝牙设备名绕过配对与 Android 设备建立蓝牙连接，这个严重级别的漏洞也在去年所修复，与上述两个漏洞异曲同工。同样是攻击面思维，漏洞作者将可控的蓝牙设备名传入蓝牙配置文件带来安全影响，而我们这里则是将蓝牙设备名传入配对对话框欺骗普通用户。

小结

从攻的角度来看，对话框是一种特殊的攻击面；从防的角度来看，对话框也是一种重要的安全机制，开发者需在对安全或隐私有影响的操作前设置用户交互对话框，在用户同意后才可进行敏感操作，并仔细检查对话框中传入的内容，防止对用户进行点击欺骗。

参考文献

[1]https://source.android.com/security/overview/updates-resources

[2]https://curesec.com/blog/article/blog/CVE-2013-6272-comandroidphone-35.html

[3]https://android.googlesource.com/platform/packages/apps/Settings/+/7ed7d00e6028234088b58bf6d6d9362a5effece1%5E%21/#F0

[4] https://github.com/nccgroup/nOBEX

[5]https://android.googlesource.com/platform/frameworks/base/+/a6fe2cd18c77c68219fe7159c051bc4e0003fc40

[6] http://sploit3r.xyz/cve-2017-13284-injection-in-configuration-file

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8KOiaP1kr4CDPZhM3LUQib4zdClibfkiaRWibSOrumvN1RY3DkzYkibtJO5H6icwu4pH79YGw0K1VPgevSJA/640?wx_fmt=jpeg)