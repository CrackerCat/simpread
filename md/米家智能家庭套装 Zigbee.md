> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2019/12/12/%E7%B1%B3%E5%AE%B6%E6%99%BA%E8%83%BD%E5%AE%B6%E5%BA%AD%E5%A5%97%E8%A3%85-Zigbee/)

> 米家智能家庭套装礼品装包含：米家多功能网关、米家智能插座（Zigbee 版)、米家无线开关、米家人体传感器、米家门窗传感器 拆解 米家门窗传感器 网关 网关拆卸比较简单，卸掉三颗螺丝即可，但是使用的螺丝型号为 U2.6，不怎么常见，好在之前买的米家螺丝刀里面有。

```
dac_freq_set_ = 44100 , 44100 
03-30  13:17:49.526 
{"id":29,"sid":"lumi.158d000232b856","model":"lumi.sensor_motion.v2","method":"props","params":{"device_log":"[1585545469,[\"event.motion\",[]]]"}}
03-30  13:17:49.548 
{"id":30,"method":"props","params":{"music_op":"p_door_10","from.music_op":"5,2719655867,1585545469,lumi.158d000232b856.motion"}}
03-30  13:17:49.561 
{"id":31,"method":"_sync.upLocalSceneRuningLog","params":[{"us_id":2719655867,"time":1585545469,"msg":[{"n":1,"e":0}]}]}
=eSL_WriteMessage= t=9f18,l=0,data=
=recv zigbee message:Free=174816 ,t=9f98;l=54;data= 01 87 20 a5 26 73 1c 71 82 1f d8 fd 9e 8b 54 49 7a 01 01 0f b0 f7 00 03 00 03 a0 21 00 15 8d 00 02 4a eb 9d 00 15 8d 00 02 4a eb 9d 65 46 1c f9 95 61 e4 76 23 11
03-30  13:17:57.003 

###{"id":32,"method":"_async.store","params":{"bak_data":{"did":"80969777","model":"lumi.gateway.v3","data_type":"syscfg","total":1,"cur":0,"flag":"1585545477_1","data":"{\"spk_v\":60,\"gtw_v\":40,\"arm_v\":80,\"dbl_v\":60,\"clt_en\":1,\"nlt_rgb\":1686077311,\"dms_id_0\":0,\"dms_id_1\":10,\"dms_id_2\":20,\"dms_id_3\":30,\"dms_id_4\":40,\"cdl_time\":60,\"dbps_en\":0,\"awt_time\":5,\"alen_time\":30,\"alt_en\":1}","ver":15}}},T=137003
=recv zigbee message:Free=174192 ,t=8000;l=5;data= 00 00 9f 18 00
```

Flash 大小为 16M。dump Flash 的时候可能会中断几次，此时需要根据实际情况，分段进行下载并拼接起来。然后将 raw 等区段 dump 出来，以供静态分析。

strings 查看字符串，libfaac 1.26.1 多次出现时，libfaac 用于音频编码。多次出现的 LAME3.99.5。[LAME](http://www.linuxfromscratch.org/blfs/view/7.7/multimedia/lame.html) 是一种的 MP3 编码器，网关中有提示音和广播，这些部分是音频数据。mcu 关键的特征值（可读字符串）在 Flash 中没有找到，其块中也没找到，也应该在这块，猜测被加密了无法直接读取。

将小米多功能网关上电，通过米家 APP 连接后，选择” 网关”，然后在右上角点击三个点的图标，选择” 关于”, 进入新界面后多次点击底部的版本后，进入开发者模式。从多出来的 “网关信息” 中得到信道为 15。

将 CC2531 USB Donglg 插入电脑，并安装好[驱动](https://download.csdn.net/download/u013075443/16490601)。 然后下载并安装 [TIMAC](http://www.ti.com/tool/TIMAC) 套件或 [PACKET-SNIFFER](https://www.ti.com.cn/tool/cn/PACKET-SNIFFER)，打开 TiWsPc 对设备进行配置，选择 “Device Configuration” 将信道设置为 15，并点击 start。以上的目的是配置 CC2531 USB Donglg 抓取 15 信道的数据，并创建管道，将数据传递传递出来。最后在 Wireshark 快捷方式属性中的目标中添加`-i\\.\pipe\tiwspc_data -k`，使用 TiWsPc 创建的管道抓取 15 信道上的 Zigbee 数据。

双击 Wireshark 就可以使用了，但现在数据还是加密的。由于小米多功能网关的密钥在串口打印出来了，我们可以用来解密流量，使用快捷键 CTRL+SHIFT+P 弹出首选项，在协议中选择 Zigbee，并配置密钥。在下图的 Key 中添加密钥。