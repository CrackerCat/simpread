> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271849.htm)

> [原创] 一个 BLE 智能手环的分析

首先确定手环的 MAC 地址，使用 APP 进行扫描连接，连接成功就会显示设备的 MAC 地址：A4:C1:38:6A:1C:BF

![](https://bbs.pediy.com/upload/attach/202203/837755_JV5B5EQD8B86G7E.jpg)

接下来解除绑定，使用 TI 的 packet sniffer 进行蓝牙数据包的捕获，插入 CC2540 后选择好类型，开始捕获后再查找手环进行绑定，因为 BLE 会随机在 37、38、39 三个信道中选择一个，而  packet sniffer 只能监听一个信道，所以可能需要多试几次才能捕获到，其中 M->S 表示是手机发送给手环，S->M 是手环发送给手机

![](https://bbs.pediy.com/upload/attach/202203/837755_XYPDUJTDTHRAXSZ.jpg)

有些可疑的包标记一下，这两个都包含了当前的步数 100，即 0x64，且都是从手机端发送一个请求之后从手环发回来的

![](https://bbs.pediy.com/upload/attach/202203/837755_M2T6X3BQWMTFU6P.jpg)

![](https://bbs.pediy.com/upload/attach/202203/837755_NRUVP3AYG3X8TCK.jpg)

在捕获的最后阶段，我摁下了几次查找手环的命令，这应该是查找手环的震动效果

![](https://bbs.pediy.com/upload/attach/202203/837755_82JGRGYUHHF4H3J.jpg)

但是此时通过 gatttool 或者 nRFconnect 进行发送是不成功的，因为手环需要进行绑定才能通信，接下来使用 AndroidKiller 抓取 APP 的日志进行分析，查看 APP 是如何与手环进行通信的

打开 AndroidKiller 后连接手机，在 Android 中已找到设备中就可以看到手机了，然后点击日志

![](https://bbs.pediy.com/upload/attach/202203/837755_44N399VXRJPJ72V.jpg)

点击开始进行日志的捕获

![](https://bbs.pediy.com/upload/attach/202203/837755_CN5Q82XKQ6786RH.jpg)

这里进行 BLE 的连接

![](https://bbs.pediy.com/upload/attach/202203/837755_3U6Y8FD9XPSS4UB.jpg)

BLE 连接成功，此时虽然连接成功但是仍然没有绑定设备

![](https://bbs.pediy.com/upload/attach/202203/837755_PEEPJEUF7W8MM6K.jpg)

向手环发送了一条指令：430000dc，猜测是用来进行绑定，可以先记录下来，后边通过 gatttool 进行验证

![](https://bbs.pediy.com/upload/attach/202203/837755_YYDJWGKQP4PRBTC.jpg)

获取手环电量指令：27000074

![](https://bbs.pediy.com/upload/attach/202203/837755_CFZVXPRP7CDADWN.jpg)

获取步数、卡路里、距离等指令：2001000070

获取心率等指令：21010000c6

(这里只是根据值大致推测了一下，并没有逆向 APP 查看每个字段的范围)

![](https://bbs.pediy.com/upload/attach/202203/837755_AQQN27C8GWZBH4P.jpg)

获取体温数据指令：2c01000078（我还没测过体温，所以没啥数据）

![](https://bbs.pediy.com/upload/attach/202203/837755_2NG95TUC2SPZ39U.jpg)

查找手环指令：1008000000000001000000c00000000000000000，效果是三次长时间的震动

![](https://bbs.pediy.com/upload/attach/202203/837755_KHUABC9DFK3E9DN.jpg)

手环还支持将 APP 收到的消息推送到手环上显示，打开该功能后给手环支持的 APP 发送消息，手环就会显示收到的消息

![](https://bbs.pediy.com/upload/attach/202203/837755_QQJ67ZJEYVVQ7XH.jpeg)

查看一下这个过程的日志，可以发现如下信息：

首先是 0a020000020e，告诉手环要推送消息

![](https://bbs.pediy.com/upload/attach/202203/837755_9AQ7TA75B4PUCS3.jpg)

然后是发送人的昵称：test，这里的指令为 0a050001746573743a

![](https://bbs.pediy.com/upload/attach/202203/837755_UBRYUAATZQRQT2U.jpg)

这里是发送的消息，1234，指令为 0a05000231323334

![](https://bbs.pediy.com/upload/attach/202203/837755_6GNR95REAEVP7SZ.jpg)

最后是指令 0a0100030e 表示消息都发送完了，手环可以显示了

![](https://bbs.pediy.com/upload/attach/202203/837755_SDRR8S7MECKGZCV.jpg)

除了前后两条指令，消息部分指令中间一部分可以很明显知道是 ASCII 的十六进制，但是整条指令的构成需要对 APP 进行逆向分析，通过日志前面的 tag 可以知道这是在 CmdHelper.java 中实现的，使用 jadx 打开 APP，找到 CmdHelper，位置在 com->runmifit.android->util->ble->CmdHelper，找到 setMessage2，可以看到他接收的参数是一个整形一个字符串，根据日志的上下文可以推测是 IntelligentNotificationService 传给他的参数，定位到 com->runmifit.android->sevice->IntelligentNotificationService，发现只有两个地方调用了 setMessage2，一个传 1 一个传 2

![](https://bbs.pediy.com/upload/attach/202203/837755_GRX8UACE9DHGRY6.jpg)

此时再回来看一下 setMessage2 中参数 i 的用途，猜测这个是用来区分是发送人昵称还是消息的字段

![](https://bbs.pediy.com/upload/attach/202203/837755_USPKSS8SV3RDX53.jpg)

setMessage2 函数主要作用是构造一个 bArr2 数组给 spliteData 函数，这里主要看以下 bArr2 的构成

![](https://bbs.pediy.com/upload/attach/202203/837755_SEDVKHHNWS2KHNK.jpg)

这里的 i2 是之前计算的长度，它加了 5 个字节作为构造出的指令的长度，bArr2[0] = 10; 开头是固定的 0a，然后两个是用来存放长度 + 1 的，然后是区分昵称还是消息的字段，接着把消息拼接上，最后是一个 completeCheckCode，往上面翻一下定位到该函数

![](https://bbs.pediy.com/upload/attach/202203/837755_H2KF36WP4DSPTRE.jpg)

他只是把传进去的数组挨个遍历加起来然后乘上 86 再加上 90，得到的结果取末尾一字节，作为一个校验。因为昵称和消息内容都是用 setMessage2 函数生成的，所以构成方法一致

至此，消息指令的构成就分析完了，试着来构造一个消息：sec 发送的 hacked

sec 的 ascii 码分别是 73 65 63，长度是 3+1=4，计算最后的校验位 0a+04+01+73+65+63=14A 即十进制的 330，330*86+90=28470，也就是 6F36，取末尾一字节 36

![](https://bbs.pediy.com/upload/attach/202203/837755_M4392JFSUZUPTGC.jpg)

所以昵称的指令为：0a04000173656336，同理消息的指令为：0a0700026861636b6564fc

接下来使用 gatttool 进行验证，需要蓝牙适配器支持 ble 通信，使用命令 hciconfig hci0 up 将蓝牙适配器激活

![](https://bbs.pediy.com/upload/attach/202203/837755_7Q9XJTCE4E4R4YA.jpg)

gatttool -I -b A4:C1:38:6A:1C:BF 进入交互模式，其中 -I 表示进入交互模式，-b 指定 MAC 地址，进入后使用 connect 进行连接，输入 help 可以查看帮助

![](https://bbs.pediy.com/upload/attach/202203/837755_J28F3H727V34E76.jpg)

输入 characteristics 查看所有的特征，我们要使用的句柄是 0x11

![](https://bbs.pediy.com/upload/attach/202203/837755_5SY7FEE8DM3E5AM.jpg)

首先发送 430000dc 进行绑定，然后就可以发送其他的指令。例如获取手环步数、查找手环让手环震动、获取手环心率等，我们先给手环推送一下之前分析的消息，测试一下判断的是否正确

![](https://bbs.pediy.com/upload/attach/202203/837755_S232XYZ3QQHVE3S.jpeg)

可以看到我们自定义的消息成功被推送到了手环上并显示出来

另外既然是通过 BLE 通信的，那么打开消息通知和关闭消息通知肯定也是有对应的指令的，通过 APP 日志也可以找出来

附：

021000320500010001010001ff000000000000e2     打开消息推送

021000320500010001010000ff0000000000008c     关闭消息推送

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

[#家用设备](forum-128-1-173.htm) [#技术分享](forum-128-1-168.htm)