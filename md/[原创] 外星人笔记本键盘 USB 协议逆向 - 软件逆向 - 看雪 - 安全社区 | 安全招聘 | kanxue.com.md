> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-279341.htm)

> [原创] 外星人笔记本键盘 USB 协议逆向

### 前言

我朋友一台 dell g16 购买时直接安装了 linux 系统，但是 linux 上没有官方的键盘控制中心，所以无法控制键盘灯光，于是我就想着能不能逆向一下键盘的协议，然后自己写一个控制键盘灯光的程序。我自己的外星人笔记本是 m16，所以我就先从 m16 开始逆向。

![](https://bbs.kanxue.com/upload/attach/202310/852613_NR2R4Q49R2UBM9D.webp)

### USB 协议分析

通过 chatgpt 得知，AlienFX 设备通常通过 USB 接口连接到计算机。键盘的灯光控制是通过 HID (人机接口设备) 协议进行的。当你使用 AlienFX 软件时，这些程序会发送特定的命令到键盘，告诉它如何设置灯光效果。

现在 wireshark 已经支持 HID 协议的解析，所以我们可以直接使用 wireshark 来分析 USB 协议。在安装 wireshark 是需要勾选安装`USBPcap`

![](https://bbs.kanxue.com/upload/attach/202310/852613_JRRS4A6JYBP2TCE.webp)

打开 wireshark，选择 USBPcap1

![](https://bbs.kanxue.com/upload/attach/202310/852613_NKNYRNPJEXT5F46.webp)

设置要捕获的 usb 设备，然后点击 start

![](https://bbs.kanxue.com/upload/attach/202310/852613_R3FWWA3SHAEGAR7.webp)

打开 `alienware command center`，设置键盘灯光，在灯光效果除设置颜色为红色，然后点击应用。

![](https://bbs.kanxue.com/upload/attach/202310/852613_M9DYU4T3WGHTAYV.webp)

然后我们就可以在 wireshark 中看到 usb 协议的数据包了， 我们可以看到有两个数据包，一个是发送数据包，一个是接收数据包。可以通过设置过滤器来过滤掉接收数据包，只看发送数据包，过滤设置为 `usb.src == "host"`

![](https://bbs.kanxue.com/upload/attach/202310/852613_PN7EPPHMSMTRJVJ.webp)

发现仅仅是改了一个按键的颜色，就发送了很多数据包，而且每个数据包的长度都不一样，这是因为每个数据包都是一个命令，而且每个命令的长度都不一样，所以我们需要找到每个命令的格式，然后才能解析出每个命令的含义。

### 验证数据包

这个时候我们需要写一个测试程序，来分析哪一个包让键盘改变了颜色，然后再分析这个包的格式。这里我们使用 python 将数据重发到 usb 设备，然后观察键盘的变化。

```
import logging
import time
 
import usb
from usb import USBError
 
 
class AlienwareUSBDriver:
    VENDOR_ID = 0xd62
    PRODUCT_ID = 0xc2b0
 
    SEND_BM_REQUEST_TYPE = 0x21
    SEND_B_REQUEST = 0x09
    SEND_W_VALUE = 0x3cc
    SEND_W_INDEX = 0x0
 
    PACKET_LENGTH = 63
 
    def __init__(self):
        self._control_taken = False
        self._device = None
 
    def acquire(self):
        """ Acquire control of the USB controller."""
        if self._control_taken:
            return
 
        self._device = usb.core.find(idVendor=AlienwareUSBDriver.VENDOR_ID, idProduct=AlienwareUSBDriver.PRODUCT_ID)
 
        if self._device is None:
            logging.error("ERROR: No AlienFX USB controller found; tried VID {}, PID {}"
                          .format(AlienwareUSBDriver.VENDOR_ID, AlienwareUSBDriver.PRODUCT_ID))
 
        try:
            self._device.set_configuration()
        except USBError as exc:
            logging.error("Cant set configuration. Error : {}".format(exc.strerror))
 
        try:
            usb.util.claim_interface(self._device, 0)
        except USBError as exc:
            logging.error("Cant claim interface. Error : {}".format(exc.strerror))
 
        self._control_taken = True
        logging.debug("USB device acquired, VID={}, PID={}".format(hex(AlienwareUSBDriver.VENDOR_ID),
                                                                   hex(AlienwareUSBDriver.PRODUCT_ID)))
 
    def release(self):
        if not self._control_taken:
            return
 
        try:
            usb.util.release_interface(self._device, 0)
        except USBError as exc:
            logging.error("Cant release interface. Error : {}".format(exc.strerror))
 
        try:
            self._device.attach_kernel_driver(0)
        except USBError as exc:
            logging.error("Cant re-attach. Error : {}".format(exc.strerror))
 
        self._control_taken = False
        logging.debug("USB device released, VID={}, PID={}".format(hex(AlienwareUSBDriver.VENDOR_ID),
                                                                   hex(AlienwareUSBDriver.PRODUCT_ID)))
 
    def write_packet(self, pkt):
        if not self._control_taken:
            return
 
        try:
            num_bytes_sent = self._device.ctrl_transfer(
                self.SEND_BM_REQUEST_TYPE, self.SEND_B_REQUEST,
                self.SEND_W_VALUE, self.SEND_W_INDEX,
                pkt, 0)
 
            logging.debug("wrote: {}, {} bytes".format(pkt, len(pkt)))
            if len(pkt) != num_bytes_sent:
                logging.error("writePacket: intended to write {} of {} bytes but wrote {} bytes"
                              .format(pkt, len(pkt), num_bytes_sent))
 
            return num_bytes_sent
        except USBError as exc:
            logging.error("writePacket: {}".format(exc))
```

其中设备信息可以在 wireshark 中查看

![](https://bbs.kanxue.com/upload/attach/202310/852613_79HT966BN9W3A63.webp)

```
VENDOR_ID = 0xd62
PRODUCT_ID = 0xc2b0
```

在使用 `device.ctrl_transfer` 发送数据时需要指定 `bmRequestType`, `bRequest`, `wValue`, `wIndex`，这些信息也可以在 wireshark 中查看

![](https://bbs.kanxue.com/upload/attach/202310/852613_KY8PX6D8W3VMMN5.webp)

```
OUT_BM_REQUEST_TYPE = 0x21
OUT_B_REQUEST = 0x09
OUT_W_VALUE = 0x3cc
OUT_W_INDEX = 0x0
```

尝试将 wireshark 中的数据包发送到键盘，通过测试发现，其中一条数据包发送后，Q 键的灯光才会改变

```
if __name__ == '__main__':
    device = AlienwareUSBDriver()
    device.acquire()
 
    data = bytes.fromhex('cc8c020073072f46121278b56519a6f9661799e568127ab7691aaaff6a2aaaff6c2aaaff6e137fbf7019a6f9711aaaff8608334b8708334b8808334b2bfc0000')
    device.write_packet(data)
```

### 数据包格式分析

通过分析得知，每次改变颜色，awcc 会通过`CC 8C 02 00` 命令把所有按键的颜色发送一遍 ，经过多次测试，发现每个按键的颜色都是由三个字节表示，分别是 `R` `G` `B`，所以我们可以通过改变这三个字节来改变按键的颜色。包格式如下:

![](https://bbs.kanxue.com/upload/attach/202310/852613_TA42XQA4T8G5UZS.webp)

经过多次尝试后，将整个键盘的对应序号，得到如下表格

<table><thead><tr><th>Key</th><th>Code</th><th>Key</th><th>Code</th><th>Key</th><th>Code</th><th>Key</th><th>Code</th></tr></thead><tbody><tr><td>esc</td><td>1</td><td>f4</td><td>5</td><td>u</td><td>0x31</td><td>lshift</td><td>0x52</td></tr><tr><td>f1</td><td>2</td><td>f5</td><td>6</td><td>i</td><td>0x32</td><td>z</td><td>0x54</td></tr><tr><td>f2</td><td>3</td><td>f6</td><td>7</td><td>o</td><td>0x33</td><td>x</td><td>0x55</td></tr><tr><td>f3</td><td>4</td><td>f7</td><td>8</td><td>p</td><td>0x34</td><td>c</td><td>0x56</td></tr><tr><td>f8</td><td>9</td><td>3</td><td>0x18</td><td>[</td><td>0x35</td><td>v</td><td>0x57</td></tr><tr><td>f9</td><td>0xa</td><td>4</td><td>0x19</td><td>]</td><td>0x36</td><td>b</td><td>0x58</td></tr><tr><td>f10</td><td>0xb</td><td>5</td><td>0x1a</td><td>\</td><td>0x38</td><td>n</td><td>0x59</td></tr><tr><td>f11</td><td>0xc</td><td>6</td><td>0x1b</td><td>a</td><td>0x3f</td><td>m</td><td>0x5a</td></tr><tr><td>f12</td><td>0xd</td><td>7</td><td>0x1c</td><td>s</td><td>0x40</td><td>,</td><td>0x5b</td></tr><tr><td>home</td><td>0xe</td><td>8</td><td>0x1d</td><td>d</td><td>0x41</td><td>.</td><td>0x5c</td></tr><tr><td>end</td><td>0xf</td><td>9</td><td>0x1e</td><td>f</td><td>0x42</td><td>/</td><td>0x5d</td></tr><tr><td>del</td><td>0x10</td><td>0</td><td>0x1f</td><td>g</td><td>0x43</td><td>rshift</td><td>0x5f</td></tr><tr><td>`</td><td>0x15</td><td>-</td><td>0x20</td><td>h</td><td>0x44</td><td>up</td><td>0x73</td></tr><tr><td>1</td><td>0x16</td><td>=</td><td>0x21</td><td>j</td><td>0x45</td><td>lctrl</td><td>0x65</td></tr><tr><td>2</td><td>0x17</td><td>back</td><td>0x24</td><td>k</td><td>0x46</td><td>fn</td><td>0x66</td></tr><tr><td>tab</td><td>0x29</td><td>caps</td><td>0x3e</td><td>l</td><td>0x47</td><td>lwin</td><td>0x68</td></tr><tr><td>q</td><td>0x2b</td><td>enter</td><td>0x4b</td><td>;</td><td>0x48</td><td>lalt</td><td>0x69</td></tr><tr><td>w</td><td>0x2c</td><td>space</td><td>0x6a</td><td>'</td><td>0x49</td><td>ralt</td><td>0x70</td></tr><tr><td>e</td><td>0x2d</td><td>rwin</td><td>0x6e</td><td>right</td><td>0x88</td><td>rctrl</td><td>0x71</td></tr><tr><td>r</td><td>0x2e</td><td>ralt</td><td>0x70</td><td>down</td><td>0x87</td><td>left</td><td>0x86</td></tr><tr><td>t</td><td>0x2f</td><td>microphone</td><td>0x14</td><td>voice0</td><td>0x11</td><td>voice+</td><td>0x13</td></tr><tr><td>y</td><td>0x30</td><td>voice-</td><td>0x12</td><td>voice+</td><td>0x13</td><td>voice-</td><td>0x12</td></tr></tbody></table>

可以用一个例子来验证上面的 keymap 是否正确

```
if __name__ == '__main__':
    device = AlienwareUSBDriver()
    device.acquire()
 
 
    keymap = {
        'esc': 1, 'f1': 2, 'f2': 3, 'f3': 4, 'f4': 5, 'f5': 6, 'f6': 7, 'f7': 8, 'f8': 9, 'f9': 0xa, 'f10': 0xb, 'f11': 0xc, 'f12': 0xd, 'home': 0xe, 'end': 0xf,
        'del': 0x10, '`': 0x15, '1': 0x16, '2': 0x17, '3': 0x18, '4': 0x19, '5': 0x1a, '6': 0x1b, '7': 0x1c, '8': 0x1d, '9': 0x1e, '0': 0x1f, '-': 0x20, '=': 0x21,
        'back': 0x24, 'microphone': 0x14, 'tab': 0x29, 'q': 0x2b, 'w': 0x2c, 'e': 0x2d, 'r': 0x2e, 't': 0x2f, 'y': 0x30, 'u': 0x31, 'i': 0x32, 'o': 0x33, 'p': 0x34,
        '[': 0x35, ']': 0x36, '\\': 0x38, 'voice0': 0x11, 'caps': 0x3e, 'a': 0x3f, 's': 0x40, 'd': 0x41, 'f': 0x42, 'g': 0x43, 'h': 0x44, 'j': 0x45, 'k': 0x46,
        'l': 0x47, ';': 0x48, '\'': 0x49, 'enter': 0x4b, 'voice+': 0x13, 'lshift': 0x52, 'z': 0x54, 'x': 0x55, 'c': 0x56, 'v': 0x57, 'b': 0x58, 'n': 0x59, 'm': 0x5a,
        ',': 0x5b, '.': 0x5c, '/': 0x5d, 'rshift': 0x5f, 'up': 0x73, 'voice-': 0x12, 'lctrl': 0x65, 'fn': 0x66, 'lwin': 0x68, 'lalt': 0x69, 'space': 0x6a, 'ralt': 0x70,
        'rwin': 0x6e, 'rctrl': 0x71, 'left': 0x86, 'down': 0x87, 'right': 0x88
    }
 
    def get_key_bytes(a, b, a_color, b_color):
        header = bytes.fromhex('cc8c0200')
        a_bytes = (a << 24 | a_color).to_bytes(4, byteorder='big')
        b_bytes = (b << 24 | b_color).to_bytes(4, byteorder='big')
 
        data = header + a_bytes + b_bytes
        out = data + bytes(64 - len(data))
        return out
 
 
    chars = list(keymap.keys())
    a = 0
    a_color = 0
    b = 0
    b_color = 0
    for i, k in enumerate(chars):
        key = keymap[k]
        a = key
        a_color = 0xff0000
        b = 0
        b_color = 0
        if i > 0:
            b = keymap[chars[i - 1]]
            b_color = 0x00ff00
        device.write_packet(get_key_bytes(a, b, a_color, b_color))
        time.sleep(0.3)
 
    a_color = 0x00ff00
    b_color = 0x00ff00
    device.write_packet(get_key_bytes(a, b, a_color, b_color))
```

效果如下：  
![](https://bbs.kanxue.com/upload/attach/202310/852613_69QZXSTXYPJMJ77.webp)

这样我们就可以根据需要，来动态设置每个按键的颜色了。

### 其他命令

其中还有清除灯光的命令，格式如下：

```
cc8c1000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

关闭波动灯光的命令，格式如下：

```
cc8c0500010101010101010101010101010101010101010101010101010101010101010101010101000000000100010101010101010101010101010100000000
cc8c0600000101010101010101010101010101000000000000010001010101010101010101000100000000000101000101010001000100010100010000000000
```

开启波动灯光的命令，格式如下：

```
cc800305000001010101000000000000050000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

### 最后

在分析过程中，也是花费了不少时间， 主要是 `alineware command center` 在全键盘设置成一个颜色时，会发送很多数据包，而且每个键对应的颜色值还不一样，我以为有特殊的算法

比如我设置成绿色时，发送的数据包如下：

![](https://bbs.kanxue.com/upload/attach/202310/852613_K3NT93Y47XHWQ3V.webp)

他会为每个键随机生成颜色相近的值， 这些值同样是以十六进制表示的 RGB 颜色代码。我们可以解析这些代码来看看这些颜色是否相似。

以下是解析的颜色及其 RGB 值：

1.  00 3F 11 - RGB(0, 63, 17)
2.  00 FF 4A - RGB(0, 255, 74)
3.  00 8E 29 - RGB(0, 142, 41)
4.  00 91 2A - RGB(0, 145, 42)
5.  00 A3 2F - RGB(0, 163, 47)
6.  00 9E 2D - RGB(0, 158, 45)
7.  00 9B 2D - RGB(0, 155, 45)
8.  00 99 2C - RGB(0, 153, 44)
9.  00 A0 2E - RGB(0, 160, 46)
10.  00 91 2A - RGB(0, 145, 42)
11.  00 8E 29 - RGB(0, 142, 41)
12.  00 9B 2D - RGB(0, 155, 45)
13.  00 CC 3B - RGB(0, 204, 59)
14.  00 8E 29 - RGB(0, 142, 41)
15.  00 FF 4A - RGB(0, 255, 74)

从这些解析的值可以看出，这些颜色大多是绿色调，但是其中有不同的亮度和饱和度。例如，00 FF 4A 是一个明亮的绿色，而 00 8E 29 是一个相对较深的绿色。

总的来说，这些颜色都是绿色调，并且大部分的颜色是相似的。不过，其中的一些颜色（如 00 FF 4A）会显得明显更亮和饱和。所以，这些颜色大部分是相似的。

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 2023-10-24 15:22 被 evilbeast 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm)