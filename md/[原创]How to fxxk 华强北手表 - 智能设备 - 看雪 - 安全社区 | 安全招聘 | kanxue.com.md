> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284600.htm)

> [原创]How to fxxk 华强北手表

[原创]How to fxxk 华强北手表

发表于: 6 天前 1498

举报

### [原创]How to fxxk 华强北手表

![](https://passport.kanxue.com/pc/view/img/moon.gif)

* * *

某个周末，找发小去玩，一眼看到他手腕上的苹果表，看起来比我的牛逼，遂抢过来玩玩。

![](https://bbs.kanxue.com/upload/attach/202411/934060_AKWSAKADBQMN29V.webp)

不得不说，还挺流畅，表盘也比我的好看（苹果你该死啊！！！）。

突然想起来，手里还有传奇的 nRF52832 Dongle。

link start。

> 关于低功耗蓝牙嗅探和渗透，参考这篇文章：[https://p1yang.github.io/article/94a01627.html#%E5%97%85%E6%8E%A2](https://p1yang.github.io/article/94a01627.html#%E5%97%85%E6%8E%A2)
> 
> 感谢 yichen115 师傅的启发：[https://bbs.kanxue.com/thread-283915.htm](https://bbs.kanxue.com/thread-283915.htm)

mac 地址：4d:23:a0:c1:00:ec  
![](https://bbs.kanxue.com/upload/attach/202411/934060_K8A3AWF53JDRD7U.webp)

通过广播包，可以看到设备名称叫 H14 Ultar+。

app 叫 lewear，长得还挺像样子。

![](https://bbs.kanxue.com/upload/attach/202411/934060_8VTKUBTJ58BA55D.webp)

通过 app 绑定和使用三方调试助手对比，发现并没有做加密或认证绑定操作，所以可以直接绕过绑定分析和加密分析。（世界安全能力降低 1w 倍之全世界没有加密）

不过华强北嘛，哪有安全，能用就行。

![](https://bbs.kanxue.com/upload/attach/202411/934060_HTV5V4BSP6EUQRK.webp)

![](https://bbs.kanxue.com/upload/attach/202411/934060_P2AXHHZBCZ9SHTT.webp)

这里先尝试微信消息显示。

![](https://bbs.kanxue.com/upload/attach/202411/934060_P5Y28DYPFE7RRRX.webp)

![](https://bbs.kanxue.com/upload/attach/202411/934060_ME3KJ2Y7YD9YAZY.webp)

包是这样的，数据为 72030101703179616e673a54657374a1cf71a9，大概可以看出来 07030101 为消息标识，70 到 a1 为消息体，最后一段还不确定，但是不影响重放。

这里没截到，是对 UUID 为 6е400002b5a3f393e0a9e50e24dcca9e 的特征（Characteristics）进行的数据交换。

并且发现几乎所有的数据交换都是通过该特征进行的，所以大胆推测下 07 的代表的功能就是消息推送。

后续再分析。

![](https://bbs.kanxue.com/upload/attach/202411/934060_7WDT96GAV387P6N.webp)

后边测试重放是可以的，甚至可以通过重放完成拒绝服务攻击。

具体攻击：

[https://www.bilibili.com/video/BV11nzbY8ESx/](https://www.bilibili.com/video/BV11nzbY8ESx/)

在通过查找 uuid 关键字，能发现 CommandHanle，即所有写入都调用这个文件下的方法

所以功能基本都调用 UUID_READ 和 UUID_WRITE，通过不同的标识区分不同的功能。

![](https://bbs.kanxue.com/upload/attach/202411/934060_JRYQQMN6THWVZTX.webp)

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p></td><td><p><code>6e40fff0</code><code>-</code><code>b5a3</code><code>-</code><code>f393</code><code>-</code><code>e0a9</code><code>-</code><code>e50e24dcca9e UUID_SERVICE</code></p><p><code>6e400003</code><code>-</code><code>b5a3</code><code>-</code><code>f393</code><code>-</code><code>e0a9</code><code>-</code><code>e50e24dcca9e UUID_READ</code></p><p><code>6e400002</code><code>-</code><code>b5a3</code><code>-</code><code>f393</code><code>-</code><code>e0a9</code><code>-</code><code>e50e24dcca9e UUID_WRITE</code></p><p><code>00002902</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805f9b34fb</code> <code>GATT_NOTIFY_CONFIG</code></p><p><code>0000180A</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805F9B34FB</code> <code>SERVICE_DEVICE_INFO 服务设备信息</code></p><p><code>00002A26</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805F9B34FB</code> <code>CHAR_FIRMWARE_REVISION 固件版本</code></p><p><code>00002A27</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805F9B34FB</code> <code>CHAR_HW_REVISION 硬件版本</code></p><p><code>00002A28</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805F9B34FB</code> <code>CHAR_SOFTWARE_REVISION 软件版本</code></p><p><code>de5bf728</code><code>-</code><code>d711</code><code>-</code><code>4e47</code><code>-</code><code>af26</code><code>-</code><code>65e3012a5dc7</code> <code>SERIAL_PORT_SERVICE 串口服务</code></p><p><code>de5bf729</code><code>-</code><code>d711</code><code>-</code><code>4e47</code><code>-</code><code>af26</code><code>-</code><code>65e3012a5dc7</code> <code>SERIAL_PORT_CHARACTER_NOTIFY 串口字符通知</code></p><p><code>de5bf72a</code><code>-</code><code>d711</code><code>-</code><code>4e47</code><code>-</code><code>af26</code><code>-</code><code>65e3012a5dc7</code> <code>SERIAL_PORT_CHARACTER_WRITE 串口字符写入</code></p><p><code>00003802</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805f9b34fb</code> <code>PAY_MAIN_SERVICE_UUID 支付主服务UUID</code></p><p><code>00004a02</code><code>-</code><code>0000</code><code>-</code><code>1000</code><code>-</code><code>8000</code><code>-</code><code>00805f9b34fb</code> <code>PAY_BASIC_WRITE_NOTIFY_UUID 支付基本写入通知UUID</code></p></td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202411/934060_FD8NXQNF9QKZT3V.webp)

通过代码调用可知，CommandHandle 为蓝牙交互的基本方法，所都的调用类型都在这里。

其参数 baseReqCmd.getData() 就是整个功能编码成蓝牙交互数据的关键。

![](https://bbs.kanxue.com/upload/attach/202411/934060_KUABRHZ3PNMWJ26.webp)

这里通过查找手表功能来理解下。

数据包：

![](https://bbs.kanxue.com/upload/attach/202411/934060_TUGSBNE5GQ9NYNR.webp)

findWatch

![](https://bbs.kanxue.com/upload/attach/202411/934060_2J3AEN823A4Z7VB.webp)

![](https://bbs.kanxue.com/upload/attach/202411/934060_44QBCUNY5XKY7M7.webp)

所有发包功能继承实现了 baseReqCmd。

所以理清 baseReqCmd 的逻辑，然后找到功能标识参数，即可伪造发送功能。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p><p>29</p><p>30</p></td><td><p><code>public</code> <code>abstract</code> <code>class</code> <code>BaseReqCmd {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>protected</code> <code>static</code> <code>final</code> <code>String TAG = </code><code>"Jxr35"</code><code>;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>protected</code> <code>byte</code> <code>key;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>protected</code> <code>int</code> <code>type;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>protected</code> <code>abstract</code> <code>byte</code><code>[] getSubData();</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>BaseReqCmd(</code><code>byte</code> <code>b) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>this</code><code>.key = b;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public</code> <code>byte</code><code>[] getData() {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>byte</code><code>[] bArr = </code><code>new</code> <code>byte</code><code>[Constants.CMD_DATA_LENGTH];&nbsp;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>bArr[</code><code>0</code><code>] = </code><code>this</code><code>.key;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>byte</code><code>[] subData = getSubData();</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(subData != </code><code>null</code><code>) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>System.arraycopy(subData, </code><code>0</code><code>, bArr, </code><code>1</code><code>, subData.length);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>addCRC(bArr);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>bArr;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>private</code> <code>void</code> <code>addCRC(</code><code>byte</code><code>[] bArr) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>int</code> <code>i = </code><code>0</code><code>;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code> <code>(</code><code>int</code> <code>i2 = </code><code>0</code><code>; i2 &lt; bArr.length - </code><code>1</code><code>; i2++) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>i += bArr[i2];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>bArr[bArr.length - </code><code>1</code><code>] = (</code><code>byte</code><code>) (i &amp; </code><code>255</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>}</code></p></td></tr></tbody></table>

所以大致思路就是传入 key + 内容 + CRC 校验码共 16 位。

通过该思路，可以反推一个数据包中所代表的功能。

这里我用 python 推了一个编码脚本。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p><p>29</p><p>30</p><p>31</p><p>32</p><p>33</p><p>34</p><p>35</p><p>36</p><p>37</p><p>38</p><p>39</p><p>40</p><p>41</p></td><td><p><code>class</code> <code>Constants:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>CMD_DATA_LENGTH </code><code>=</code> <code>16</code></p><p><code>class</code> <code>BaseReqCmd:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>TAG </code><code>=</code> <code>"Jxr35"</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>__init__(</code><code>self</code><code>, key: </code><code>int</code><code>):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>.key </code><code>=</code> <code>key</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>.</code><code>type</code> <code>=</code> <code>0</code>&nbsp;</p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>getSubData(</code><code>self</code><code>) </code><code>-</code><code>&gt; bytes:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>raise</code> <code>NotImplementedError(</code><code>"This method should be overridden by subclasses"</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>getData(</code><code>self</code><code>) </code><code>-</code><code>&gt; bytes:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>bArr </code><code>=</code> <code>bytearray(Constants.CMD_DATA_LENGTH)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>bArr[</code><code>0</code><code>] </code><code>=</code> <code>self</code><code>.key</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>subData </code><code>=</code> <code>self</code><code>.getSubData()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>subData:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>bArr[</code><code>1</code><code>:</code><code>1</code> <code>+</code> <code>len</code><code>(subData)] </code><code>=</code> <code>subData</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>.addCRC(bArr)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>bytes(bArr)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>addCRC(</code><code>self</code><code>, bArr: bytearray):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>crc </code><code>=</code> <code>sum</code><code>(bArr[:</code><code>-</code><code>1</code><code>]) &amp; </code><code>0xFF</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>bArr[</code><code>-</code><code>1</code><code>] </code><code>=</code> <code>crc</code></p><p><code>class</code> <code>FindDeviceReq(BaseReqCmd):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>__init__(</code><code>self</code><code>):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>super</code><code>().__init__(</code><code>80</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>getSubData(</code><code>self</code><code>) </code><code>-</code><code>&gt; bytes:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>bytes([</code><code>85</code><code>, </code><code>-</code><code>86</code><code>&amp;</code><code>0xff</code><code>])</code></p><p><code>if</code> <code>__name__ </code><code>=</code><code>=</code> <code>"__main__"</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>req </code><code>=</code> <code>FindDeviceReq()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>data </code><code>=</code> <code>req.getData()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(f</code><code>"Generated data: {data}"</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(f</code><code>"Hex representation: {[hex(b) for b in data]}"</code><code>)</code></p></td></tr></tbody></table>

这里重点关注了消息推送服务，尝试能否解析伪造。

通过对多次发消息的对比和代码逆向。

![](https://bbs.kanxue.com/upload/attach/202411/934060_ZS4BP5MXESAQ2XJ.webp)

大致思路就是，将一段长消息分成每段 11 个字节的短消息。

整个数据包为：key：75, app: 01-xx, 消息段数, 消息段, 消息, 校验码。

根据这个思路写的脚本：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p><p>29</p><p>30</p><p>31</p><p>32</p><p>33</p><p>34</p><p>35</p><p>36</p><p>37</p><p>38</p></td><td><p><code>import</code> <code>sys</code></p><p><code>def</code> <code>addCRC(</code><code>list</code><code>):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>checkNum </code><code>=</code> <code>sum</code><code>(</code><code>list</code><code>[:</code><code>-</code><code>1</code><code>])&amp;</code><code>0xff</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>k </code><code>=</code> <code>15</code> <code>-</code> <code>len</code><code>(</code><code>list</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code> <code>i </code><code>in</code> <code>range</code><code>(</code><code>0</code><code>,k):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>list</code><code>.append(</code><code>0</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>list</code><code>.append(checkNum)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>list</code></p><p><code>def</code> <code>string_to_unicode_and_ascii(input_str):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result </code><code>=</code> <code>[]</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code> <code>char </code><code>in</code> <code>input_str:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>char.isascii():</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result.append(</code><code>ord</code><code>(char))</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>else</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>s </code><code>=</code> <code>char.encode(</code><code>'utf-8'</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code> <code>i </code><code>in</code> <code>s:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result.append(i)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>result</code></p><p><code>def</code> <code>format_list_to_hex_string(lst):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>''.join(f</code><code>"{item:02x}"</code> <code>for</code> <code>item </code><code>in</code> <code>lst)</code></p><p><code>if</code> <code>__name__ </code><code>=</code><code>=</code> <code>'__main__'</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>input_str </code><code>=</code> <code>sys.argv[</code><code>1</code><code>]</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>output_str </code><code>=</code> <code>string_to_unicode_and_ascii(input_str)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>num_0 </code><code>=</code> <code>len</code><code>(output_str)</code><code>/</code><code>/</code><code>11</code><code>+</code><code>1</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>num_1 </code><code>=</code> <code>1</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code> <code>i </code><code>in</code> <code>range</code><code>(</code><code>0</code><code>,</code><code>len</code><code>(output_str),</code><code>11</code><code>):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>list</code> <code>=</code> <code>[</code><code>0x72</code><code>,</code><code>3</code><code>,num_0,num_1]</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code> <code>s </code><code>in</code> <code>output_str[i:i</code><code>+</code><code>11</code><code>] :</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>list</code><code>.append(s)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>addCRC(</code><code>list</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>num_1 </code><code>+</code><code>=</code> <code>1</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(format_list_to_hex_string(</code><code>list</code><code>))</code></p></td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202411/934060_2H6FZDV6KYQ5KTC.webp)

![](https://bbs.kanxue.com/upload/attach/202411/934060_3R8W8KH4G8AHK4T.webp)

这里应该不能算结束，我本来是想怎么想办法刷机 getadb，提取手表 app，拿来看看能不能 rce 玩的，但是没有 wifi 功能，只能算个蓝牙设备。无奈狗日的不让我拆，等我放假回老家给 byd 揍一顿再拆。

[[招生] 科锐逆向工程师培训 (2024 年 11 月 15 日实地，远程教学同时开班, 第 51 期)](https://bbs.kanxue.com/thread-51839.htm)

最后于 6 天前 被 p1yang 编辑 ，原因：