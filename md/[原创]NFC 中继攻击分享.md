> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268371.htm)

> [原创]NFC 中继攻击分享

前段时间不是写过一篇重放攻击绕过手环蓝牙认证的文章嘛，在这里[《手环 BLE 蓝牙认证绕过，可实现远程控制》](https://bbs.pediy.com/thread-267916-1.htm#1693385)。有读者问了这样一个问题：蓝牙能遭受到攻击，那同样是无限通信的 NFC 呢？

我是这样回答的：NFC 可以通过中继攻击来获取数据信息，而如何在不直接接触的情况下，实现 NFC 卡与读卡器的数据交互，需要重点研究一下。因为与重放攻击不同，中继攻击在攻击者获取数据信息的同时，必须保证与终端设备的通信。

#01 NFC 的原理

NFC（近场通信，Near Field Communication），又称近距离无线通信，是一种短距离的高频无线通信技术，允许电子设备之间进行非接触式点对点数据传输。

![](https://bbs.pediy.com/upload/attach/202107/908247_UG3SPBFJXYSCGQG.jpg)

与前文利用的蓝牙技术相比，NFC 使用更加方便，成本更低，能耗更低，建立连接的速度也更快，只需 0.1 秒。但是 NFC 的使用距离比蓝牙要短得多，有的只有 10CM，传输速率也比蓝牙低许多。

#02 中继攻击原理

假设准备 1 个 NFC 卡， 1 个服务器， 1 个读卡终端，2 个具备 NFC 功能的设备①、②。将这 2 个设备与服务端连接在同一 wifi，保持通信。设备①作为读卡器靠近 NFC 卡，设备②作为仿真卡靠近读卡终端。

![](https://bbs.pediy.com/upload/attach/202107/908247_G2WM6NB9X4X44ZM.jpg)

NFC 中继攻击流程图

建立 NFC 卡与读卡终端的通信隧道。通过数据的传输，完成 NFC 卡与读卡终端的数据交互。

理论上，任何使用 RFID 的设备都可能遭受类似的中继攻击。目前，我们的公交卡、银行卡、门禁卡、身份证等卡片都应用了相关技术。而且，越来越多的智能汽车也在利用 RFID 射频识别技术，来实现近距离开锁及一键启动。

接下来模拟一下银行卡的 NFC 攻击过程。

![](https://bbs.pediy.com/upload/attach/202107/908247_NEUQBWPFMKYWZ4Z.jpg)

准备设备

1. 两部带有 NFC 芯片的 Android 手机作为设备①和设备②，运行 Android 4.4+ (API-Level 19+)，并安装 NFCGate APP。

2. 设备②需要支持 HCE（主机卡仿真），并且 HCE 设备需要 root 并安装并启用 Xposed。

3.NFCGate 服务端。代理两部手机之间的通信。

4. 待读取信息的银行卡若干。

5. 读卡终端。这里用一台带有 NFC 功能的手机代替。

#03 搭建近距离攻击模型

将设备①和②及 NFCGate 的服务端连接在同一 wifi 下，开启 NFCGate 服务。连接服务端（将两个设备上 NFCGate 的 Hostname 为服务端 IP，Port 为服务端端口，默认为 5566，Session 相同即可）。

![](https://bbs.pediy.com/upload/attach/202107/908247_3GNMF8CQFE3J57F.jpg)

选择 Relay Mode，设备①选择 Reader，作为读卡器，设备②选择 Tag 作为仿真卡。

![](https://bbs.pediy.com/upload/attach/202107/908247_F2HQ768EMW4KMDM.jpg)

#04 读取银行卡信息

准备若干银行卡和读卡设备（一台带有 NFC 功能的 Android 手机代替)，安装信用卡读卡器，可以读取银行卡具体信息。

利用刚刚搭建好的近距离攻击模型，将设备①贴在银行卡上，读卡设备贴在设备②上，成功读取到银行卡的数据信息，包括银行卡号、近 10 笔交易记录、电子钱包余额、身份证号等重要信息。

下图为读取到的银行卡的部分信息：

![](https://bbs.pediy.com/upload/attach/202107/908247_32KJKWMYHCADNVW.jpg)![](https://bbs.pediy.com/upload/attach/202107/908247_KHSCAQUNSHYDB2X.jpg)

![](https://bbs.pediy.com/upload/attach/202107/908247_XMRXJRBNEQJDPS4.jpg)![](https://bbs.pediy.com/upload/attach/202107/908247_5AW27M5BF532CJ3.jpg)

值得注意的是，不同银行卡读取到的信息也略有不同。  

#05 总结

IC 芯片银行卡均可被 NFC 手机读取信息。卡面有 IC 卡芯片的银行卡，一般也带有闪付标志。在咖啡厅、餐厅使用这种芯片银行卡结账时只需要将银行卡贴近读卡器，勿须密码即可完成 200 元以下的小额支付。

虽然只有小额支付可以免密支付，但仍有部分人拿着设备去人员密集处盗刷。为了保护个人财产，请大家务必不要把银行卡等贵重卡片置于衣物的外层口袋，保护好个人财产安全！

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#安全研究](forum-128-1-167.htm) [#家用设备](forum-128-1-173.htm)