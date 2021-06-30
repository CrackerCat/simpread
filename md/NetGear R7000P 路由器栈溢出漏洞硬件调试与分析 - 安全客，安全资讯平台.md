> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/241100)

> NetGear 系列中的路由器存在栈缓冲区溢出漏洞，漏洞的产生是由于在对处理输入的字符串时，将字符串存入栈中，没有对字符串的长度做限制。

[![](https://p4.ssl.qhimg.com/t01a4e481e6c868ff0c.jpg)](https://p4.ssl.qhimg.com/t01a4e481e6c868ff0c.jpg)

漏洞信息
----

NetGear 系列中的路由器存在栈缓冲区溢出漏洞，漏洞的产生是由于在对处理输入的字符串时，将字符串存入栈中，没有对字符串的长度做限制。  
调试漏洞设备： NetGear R7000P  
漏洞固件版本：R7000P-V1.3.1.64_10.1.36  
固件下载链接：【[http://www.downloads.netgear.com/files/GDC/R7000P/R7000P-V1.3.1.64_10.1.36.zip】](http://www.downloads.netgear.com/files/GDC/R7000P/R7000P-V1.3.1.64_10.1.36.zip%E3%80%91)  
poc 链接： 【[https://github.com/grimm-co/NotQuite0DayFriday/tree/master/2020.06.15-netgear】](https://github.com/grimm-co/NotQuite0DayFriday/tree/master/2020.06.15-netgear%E3%80%91)

漏洞静态分析
------

把下载好的固件包使用 binwali -Me 解开。在后查看 httpd 文件所在的位置

[![](https://p5.ssl.qhimg.com/t01b42c1acc462adfd6.png)](https://p5.ssl.qhimg.com/t01b42c1acc462adfd6.png)

首先根据公开的 poc 开分析，漏洞产生的点有 “ upgrade_check.cgi “ 的字符串，可以定位到如下图的代码段。可以看到有两处，很明显是下面的判断才能使程序继续往下走，因为第一个判断里面讲初始化的 stop_while 原本为 1，设置为 0，会导致程序去校验其他的文件，无法达到漏洞点。

[![](https://p2.ssl.qhimg.com/t01210843d61e098e3a.png)](https://p2.ssl.qhimg.com/t01210843d61e098e3a.png)

最终会跳转到下面代码片段，最终跳出循环。

[![](https://p4.ssl.qhimg.com/t0160b1c4ba61b24471.png)](https://p4.ssl.qhimg.com/t0160b1c4ba61b24471.png)

因为在循环之前，就设置 stop_while 的值为 1

[![](https://p4.ssl.qhimg.com/t01e223930475545b92.png)](https://p4.ssl.qhimg.com/t01e223930475545b92.png)

跳出循环后，往下继续看，发现会对数据进行判断，这里分别对 “name=”mtenFWUpload”” 和 “\r\n\r\n” 进行判断，如果存在，则跳出当前循环

[![](https://p1.ssl.qhimg.com/t011c411038e4a1832e.png)](https://p1.ssl.qhimg.com/t011c411038e4a1832e.png)

跳出循环后，就会执行到将产生溢出点的函数

[![](https://p0.ssl.qhimg.com/t01ed83159776277bec.png)](https://p0.ssl.qhimg.com/t01ed83159776277bec.png)

进入 vulfunc 函数，可以看到会对数据进行判断是否以”*#$^” 开头。然后会将数据通过 memcpy 函数由 src 指向地址为起始地址的连续 v9 个字节的数据复制到以 s 指向地址为起始地址的空间内。并且可以看到 v9 的值是可以通过 src 中的数据进行获取，因此可以进行通过控制 v9 的大小，将 src 的数据覆盖到 s 中，但 s 的大小只有 140 个字节，从而造成大缓冲区向小缓冲区复制的数据覆盖相邻内存内的数据。造成栈缓冲区溢出。

[![](https://p5.ssl.qhimg.com/t01851d47b82a2a4b56.png)](https://p5.ssl.qhimg.com/t01851d47b82a2a4b56.png)

接入路由器 UART 串口
-------------

首先需要识别串口  
识别工具：万用表、 DSLogic 、 FT232 转换 USB

拆开路由器的外壳，可以很明显的看到路由器 PCB 中有 UART 的四根引脚，如下图所示。

[![](https://p1.ssl.qhimg.com/t01368cc6020904c71a.png)](https://p1.ssl.qhimg.com/t01368cc6020904c71a.png)

### 识别 UART 引脚

接下来我们来识别 UART 的引脚。  
这里使用万用表来识别 URAT 四个引脚的定义，  
URAT 主要的引脚分别是：

*   TX （接收）
*   RX （发送）
*   GND （接地）
*   Vcc （供电）

**识别 GND 引脚**  
路由器 PCB（通电）: 我是通电状态下测的  
万用表：将万用表指针指向一个类似 wifi 信号的符号。  
万用表的黑色探针放在地面或者任意金属面，红色探针分别在 4 根引脚出逐一测试，直到听到滴滴或者哔哔的声音。下面这块 PCB 设备，第三根是 GND 引脚。  
这种测试被称为通导性测试。

[![](https://p4.ssl.qhimg.com/t0104ea1a58b11a7fb7.png)](https://p4.ssl.qhimg.com/t0104ea1a58b11a7fb7.png)

**识别 Vcc 引脚**  
PCB： 需要给 PCB 板子通电  
万用表：指针指向 20V 的直流档位

1、黑色探针接地，红色探针在除了 GND 引脚的其他引脚测量，电压值为最大值，例如 3.6V，5V 等。就可以确定是 VCC 引脚。但是我在实际测试的时候，在第二根引脚和第四根引脚都是出现了最大电压值，不好判断。  
第四根引脚电压值如下图所示

[![](https://p0.ssl.qhimg.com/t01e8f675abf5cba383.png)](https://p0.ssl.qhimg.com/t01e8f675abf5cba383.png)

第二根引脚电压值如下图所示

[![](https://p4.ssl.qhimg.com/t01a066107c7355dbd8.png)](https://p4.ssl.qhimg.com/t01a066107c7355dbd8.png)

2、不好判断的时候，将这两根引脚分别和 GND 引脚短接，设备重启，就可以判断是 Vcc 引脚。  
这里我将 GND 引脚和第四根短接的时候，设备会重启。  
因此第四根是 Vcc 引脚

[![](https://p2.ssl.qhimg.com/t01c8d5cf708649c552.png)](https://p2.ssl.qhimg.com/t01c8d5cf708649c552.png)

**识别 TX 引脚和 RX 引脚**  
PCB ： 通电  
万用表：指针指向 20V 的直流档位

1、 黑色探针放在 GND 引脚，红色探针测试剩余的两根引脚。 测试 GND 引脚和其他引脚之间的电压，由于设备在引导式会进行大量的数据传输，因此在电压有变化并且增大后固定一个值，那么这个引脚就是 TX 引脚。

2、 如果电压波动很小，或者一直为 0，那该引脚就是 Rx 引脚。

### 确定波特率

确定波特率有不同的方法，基本上常见的波特率都试一遍，一般都可以确定串口的波特率，但是这里介绍使用 DSLogic 逻辑分析仪 来确定 UART 串口波特率。  
**学习使用 DSLogic**  
首先要先学会使用逻辑分析仪，这里简单的测试让逻辑分析仪接收 “ helloworld “  
接线： 将逻辑分析仪通道 1 的线接到 FT232 上的 TXD（Tx）引脚。 然后将分析仪的黑色引线（接地线）接到 GND 引脚。

[![](https://p5.ssl.qhimg.com/t0163428d15f9a2de07.png)](https://p5.ssl.qhimg.com/t0163428d15f9a2de07.png)

这里的 DSView 工具中参数的设置需要去看说明文档，点击 “开始” 按钮，采集数据，然后使用 XCOM 串口工具设置好波特率和串口，打开串口后，设置发送的信息，点击发送即可。然后在 DSView 就可以看到采集的信息了。采集完之后设置好解析，就可以看到对应的信息了

[![](https://p0.ssl.qhimg.com/t0189cdbe602f49403b.png)](https://p0.ssl.qhimg.com/t0189cdbe602f49403b.png)

**使用 DSLogic 采集串口信息**  
接线： 之前用万用表将引脚都识别出来了，因此直接把逻辑分析仪的通道 1 的线接到 PCB 串口的 TX 引脚上，然后将分析仪的黑色引线（接地线）接到 GND 引脚。如下图所示

[![](https://p1.ssl.qhimg.com/t01578f45a5d7d54b60.png)](https://p1.ssl.qhimg.com/t01578f45a5d7d54b60.png)

1、 线接好了之后，打开 DSView 软件  
* 设置采样率为 1MHZ  
* 设置采样时间为 1s ，当然你也可以设置大一些，10s 或者更大，这样采集的信息可能会更多。  
* 设置阈值电压，用万用表测是 TX 和 GND 之间的电压为 3.34V，我这里设置为 4V。  
* 设置通道为 6 个，这个其实是用几个就设置几个。

[![](https://p2.ssl.qhimg.com/t01e72f4970266117b4.png)](https://p2.ssl.qhimg.com/t01e72f4970266117b4.png)

点击开始按钮，进行采集，然后重启路由器 PCB。  
采集结果如下图所示

[![](https://p4.ssl.qhimg.com/t016fb4a6bf642e9d01.png)](https://p4.ssl.qhimg.com/t016fb4a6bf642e9d01.png)

注意这里显示了正常的 ASCII 码，是因为我设置好了波特率。一般来说，常用的波特率有常见波特率 1200、2400、4800、19200、38400、57600、9600、115200。  
如下图所示，如果不对，则为乱码，因此可以确定波特率为 115200

[![](https://p3.ssl.qhimg.com/t01a2adf7e3999066b9.png)](https://p3.ssl.qhimg.com/t01a2adf7e3999066b9.png)

**PCB 串口接入**  
接线方式：  
TTL ——- 路由器 UART  
GND ——- GND  
TX ——- RX  
RX ——- TX

[![](https://p1.ssl.qhimg.com/t010c1e846a35ccb932.jpg)](https://p1.ssl.qhimg.com/t010c1e846a35ccb932.jpg)

使用 SecureCRT 接入连接

[![](https://p1.ssl.qhimg.com/t01a0b69fc72768225f.png)](https://p1.ssl.qhimg.com/t01a0b69fc72768225f.png)

远程动态调试
------

动态调试工具：编译好的 gdbserver (arm 架构)、gdb(arm 架构)  
poc: 【[https://github.com/grimm-co/NotQuite0DayFriday/blob/trunk/2020.06.15-netgear/exploit.py】](https://github.com/grimm-co/NotQuite0DayFriday/blob/trunk/2020.06.15-netgear/exploit.py%E3%80%91)

**gdbserver 启动**  
在使用串口接入设备之后，把 gdbserver 上传到设备的 / tmp 目录中

```
cd /tmp/;wget http://xx.xx.xx.xx:xxxx/gdbserver;chmod 777 gdbserver


```

[![](https://p5.ssl.qhimg.com/t01e13816d4fbaa7004.png)](https://p5.ssl.qhimg.com/t01e13816d4fbaa7004.png)

在设备中查找 “httpd” 的进程号

[![](https://p0.ssl.qhimg.com/t0130acbb18e151ef78.png)](https://p0.ssl.qhimg.com/t0130acbb18e151ef78.png)

将 httpd 进行附加到 gdbserver

```
./gdbserver *:12345 --attach 3256


```

[![](https://p4.ssl.qhimg.com/t0199b75eaf011f5d96.png)](https://p4.ssl.qhimg.com/t0199b75eaf011f5d96.png)

**gdb 远程连接 gdbserver**

```
target remote 192.168.1.1:12345


```

[![](https://p3.ssl.qhimg.com/t01e43de40202de1ea8.png)](https://p3.ssl.qhimg.com/t01e43de40202de1ea8.png)

**开始调试**  
在关键的地点打好断点  
打断点

```
(gdb) break *0x00017398
Breakpoint 1 at 0x17398
(gdb) break *0x0001CFF0
Breakpoint 2 at 0x1cff0
(gdb) break *0x0001D0C4
Breakpoint 3 at 0x1d0c4


```

然后输入”continue”， 接着执行 exploit

[![](https://p2.ssl.qhimg.com/t01653cd903579620b6.png)](https://p2.ssl.qhimg.com/t01653cd903579620b6.png)

[![](https://p0.ssl.qhimg.com/t01f1bb5a355c74ea15.png)](https://p0.ssl.qhimg.com/t01f1bb5a355c74ea15.png)

到 $R0 寄存器中作为 vulfunc 的参数

[![](https://p0.ssl.qhimg.com/t014618a67b6dbb3260.png)](https://p0.ssl.qhimg.com/t014618a67b6dbb3260.png)  
接下来我们看寄存器的值，可以看到 $r0 寄存器的值为 0xbe940f89

[![](https://p5.ssl.qhimg.com/t0120f15add17b87e28.png)](https://p5.ssl.qhimg.com/t0120f15add17b87e28.png)

查看 0xbe940f89 内存地址中的值，可以看到内存中成功的被覆盖了构造好的数据。

[![](https://p1.ssl.qhimg.com/t01902533fb4c4d7a0d.png)](https://p1.ssl.qhimg.com/t01902533fb4c4d7a0d.png)

然后接着 “continue” 往下执行，执行到 0x0001cff0 断点位置会执行 memcpy 函数。

[![](https://p3.ssl.qhimg.com/t01914a401bc022e2d5.png)](https://p3.ssl.qhimg.com/t01914a401bc022e2d5.png)

查看执行到 memcpy 函数时，寄存器中的值，由于 memcpy 函数会调用 $r0 , $r1 , $r2 寄存器的值作为参数

[![](https://p5.ssl.qhimg.com/t012228fcd32262f751.png)](https://p5.ssl.qhimg.com/t012228fcd32262f751.png)

查看 $r0 , 和 r1 寄存器内的值， 在执行前 r0 还没被 $r1 中的值覆盖。

[![](https://p3.ssl.qhimg.com/t018fe6fa96f25a0ab2.png)](https://p3.ssl.qhimg.com/t018fe6fa96f25a0ab2.png)

执行完 memcpy 函数之后 r0 被 r1 中的值所覆盖

[![](https://p2.ssl.qhimg.com/t01829f940578588281.png)](https://p2.ssl.qhimg.com/t01829f940578588281.png)

接着往下执行，vulfunc 函数 会把栈上的地址复制到 r4~r11,pc 寄存器中

[![](https://p1.ssl.qhimg.com/t01b764477d49869c55.png)](https://p1.ssl.qhimg.com/t01b764477d49869c55.png)

接着往下执行到 system 函数，可以看到 $r0 寄存器地址上的值是我们要执行的 command。接着往下执行就会达到执行命令的效果

[![](https://p4.ssl.qhimg.com/t0118e019dc981a7dc6.png)](https://p4.ssl.qhimg.com/t0118e019dc981a7dc6.png)

构造 ROP
------

虽然 poc 中已经计算好了偏移，但是我们需要自己去计算。  
如下图所示，我们可以知道 memcpy 中 第一个参数 s 的离栈底的偏移是 136

[![](https://p3.ssl.qhimg.com/t011f955163f9f169ab.png)](https://p3.ssl.qhimg.com/t011f955163f9f169ab.png)

最终构造的 rop 链如下图所示

[![](https://p0.ssl.qhimg.com/t017d964142e1dd7ada.png)](https://p0.ssl.qhimg.com/t017d964142e1dd7ada.png)

总结
--

通过这个漏洞的调试，可以学些到通过 UART 在路由器中进行远程调试。