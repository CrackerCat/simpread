> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267916.htm)

> [原创] 手环 BLE 蓝牙认证绕过，可实现远程控制

01 低功耗蓝牙（BLE）
-------------

BLE 是常见的手环所用蓝牙，低功耗蓝牙（Bluetooth low energy，简称 BLE）指支持蓝牙协议 4.0 或更高的模块。相较于传统蓝牙，BLE 的特点是最大化的待机时间、快速连接和低峰值的发送 / 接收功耗。BLE 只在需要时传输少量数据，而除此之外则会保持关闭状态，这大大降低了其功耗，也使其成为了在低数据速率下需要长久连接使用的理想选择。

由此来看，BLE 非常适合运用于手环这种数据量比较少的传输场景。下面分析手环认证机制后，进行认证绕过。

![](https://bbs.pediy.com/upload/tmp/908247_2UBHQBUJFSN72NE.jpg)

02 重放攻击

首先，手机开启开发者模式，在开发者选项中启用蓝牙 HCI 信息收集日志。然后，通过手机 APP 点击寻找手环功能，向手环发送蓝牙数据。

![](https://bbs.pediy.com/upload/attach/202106/908247_3V6U5B77H86W2RU.jpg)

分析蓝牙日志，确认蓝牙重放句柄的 handle 和 value，使用 BLE 调试助手或 nRF Connect 等蓝牙调试工具即可重放攻击。

![](https://bbs.pediy.com/upload/attach/202106/908247_YEUNJ3VVPYVFNS6.jpg)

03 认证机制分析

手环认证分两种情况，一种是未绑定的情况，一种是已绑定的情况。已绑定情况只是少了一步发送 key 到手环的步骤。由后面分析可知，已绑定的手环同样可以发送 key 值覆盖之前的 key，所以这里只介绍未绑定的情况。

通过手机 APP，绑定手环，抓取蓝牙日志。

首先，获取认证 characteristic 的 descriptor，向 desc 句柄写入 0x0100。

![](https://bbs.pediy.com/upload/tmp/908247_V9M9FQ43V9HKXGA.jpg)

如果该手环处于未绑定的状态，需要向手环中写入 0x0100+key（16 bytes）。

![](https://bbs.pediy.com/upload/attach/202106/908247_59N3GFTHFBUST3K.jpg)

之后，手机 APP 端再向手环发送 0x020002，来获取随机数。手环会返回 0x100201 + 随机数（16 bytes）到手机 APP 端。

![](https://bbs.pediy.com/upload/attach/202106/908247_NZ66XF6NU7ETGXT.jpg)

然后，手机 APP 端再将 key 与随机数加密，并以 0x0300 + 加密数据（16 bytes）的形式发送给手环。

![](https://bbs.pediy.com/upload/attach/202106/908247_J5HMB3B7Q4P398U.jpg)

最后，手环内部将 key 与随机数按同样方式加密，对比加密数据，如果相同则返回 0x100301，表示认证通过。失败则返回 0x100304。

![](https://bbs.pediy.com/upload/attach/202106/908247_37VGMGU6CDWM3GC.jpg)

由于已知 key，随机数及加密后的数据，我们可以推出，该加密方式为 AES_ECB。

![](https://bbs.pediy.com/upload/attach/202106/908247_CSZF9YBX8N7FX7Y.jpg)

未绑定状态认证流程

04 认证机制绕过

根据上面分析，我们其实可以看出来这里面有一个很严重的漏洞。在手环未进行绑定的情况下，如果我写入任意 key，根据返回的随机数，按 AES_ECB 的方式加密，再向手环发送正确的加密后的数据，即可绕过认证。

经多次测试，在已绑定的情况下，如果也向手环写入 key，则可以覆盖掉之前的 key（之前绑定的设备将无法连接，需重新绑定）。同样也能绕过认证机制。

最后通过蓝牙抓包，确定手环短信提示，来电提示，寻找手环等蓝牙包的 handle 及 value，即可实现远程控制手环。

![](https://bbs.pediy.com/upload/attach/202106/908247_FTYQVG699SQ5DMN.jpg)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)