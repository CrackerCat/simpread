> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [archive.ph](https://archive.ph/OL9ZZ)

> archived 18 Jul 2023 13:59:19 UTC

针对大麦网部分演唱会门票仅能在 app 渠道抢票的问题，本文研究了 APK 的抢票接口并编写了抢票工具。本文介绍的顺序为环境搭建、抓包、trace 分析、接口参数获取、rpc 调用实现，以及最终的功能实现。通过阅读本文，你将学到反抓包技术破解、Frida hook、jadx apk 逆向技术，并能对淘系 APP 的运行逻辑有所了解。本文仅用于学习交流，严禁用于非法用途。

**关键词**： frida, 大麦网, Android 逆向  
先放成功截图：

![](https://archive.ph/OL9ZZ/d41daeb2986451904d094ea67be92beda9255f05.png)

疫情结束的 2023 年 5 月，大家对出去玩都有点疯狂，歌手们也扎堆开演唱会。演唱会虽多，票却一点也不好抢，抢五月天的门票难度不亚于买五一的高铁票。所以想尝试找一些脚本来辅助抢票，之前经常用 selenium 和 request 做一些小爬虫来搞定自动化的工作，所以在 [MakiNaruto/Automatic_ticket_purchase](https://archive.ph/o/OL9ZZ/https://github.com/MakiNaruto/Automatic_ticket_purchase) 的基础上改了改，实现抢票功能。但是大麦网实在太**狡猾**了，改完爬虫才发现几乎所有的热门演唱会只允许在 app 购买，所以就需要利用 APP 实现接口自动化。

*   windows 10
*   大麦网 apk 版本 8.5.4 (2023-04-26)
*   bluestacks 5.11.56.1003 p64
*   adb 31.0.2
*   Root Checker 6.5.3
*   wireshark 4.0.5
*   frida 16.0.19
*   jadx-gui 1.4.7

目前 Android 模拟器竞品很多，选择 Bluestacks **5** 是因为它能和 windows 的 hyper-v 完美兼容，root 过程也相对简单。

1.  下载安装 Bluestacks。
2.  运行 Bluestacks Multi-instance Manager，发现默认安装的版本为 Android Pie 64bit 版本，即 Android 9.0。此时退出 bluestack 所有程序。  
    ![](https://archive.ph/OL9ZZ/c3be73ed866c8ddcad317ec30a83aa67150cb876.png)
3.  关闭 bluestack 后编辑 bluestacks 配置文件， `%programdata%\BlueStacks_nxt\bluestacks.conf`
    
    > 由于作者安装时 C 盘空间不足，真实的`bluestacks.conf`在`D:\BlueStacks_nxt\bluestacks.conf`，大家也根据实际情况调整  
    > ![](https://archive.ph/OL9ZZ/f32a9997261d3947a6157ad135554109b8718f3a.png)
    
4.  在配置文件中查找 root 关键词，对应值修改为 1，共两处。
    
    <table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>bst.feature.rooting</code><code>=</code><code>"1"</code></p><p><code>bst.instance.Pie64.enable_root_access</code><code>=</code><code>"1"</code></p></td></tr></tbody></table>
    
    ![](https://archive.ph/OL9ZZ/c2b98855e87b4087f6d843a32c29a7b3679f1f9f.png)
    
5.  启动 bluestack 模拟器，安装 `Root Checker` APP，点击验证 root，即可发现 root 已成功。  
    ![](https://archive.ph/OL9ZZ/98a96110bba5fbadd51e56e082fba71c28e18cbb.png)
    

> 上述 root 过程主要参考了 https://appuals.com/root-bluestacks/ ，部分地方做了改正，在此感谢原文作者。

1.  bluestack 设置 - 高级中打开 Adb 调试，并记录下端口
    
    ![](https://archive.ph/OL9ZZ/3bb548fbdd9f8fac63af80cd6a72e55fc779f7c5.png)
    
2.  打开主机命令行，运行 `adb connect localhost:6652`，端口号修改为上一步的端口号，即可连接。再运行`adb devices`，如有对应设备则连接成功。
    
3.  进入 adb shell，执行 su 进入 root 权限，命令行标识由`$`变为`#`，即表示 adb 进入 root 权限成功。

![](https://archive.ph/OL9ZZ/17b19b7dd98a60c0988e44ec66687ca51c6f5e73.png)

frida 是大名鼎鼎的动态分析的 hook 神器，用它可以直接访问修改二进制的内存、函数和对象，非常方便。它对于 Android 的支持也是很完美。

frida 的运行采用 C/S 架构，客户端为电脑端的开发环境，服务器端为 Android，均需对应部署搭建。

firda 客户端基于 python3 开发，因此首先需要配置好 python3 的运行环境，然后执行 `pip install frida-tools`即可完成安装。运行 `frida --version`可验证 frida 版本。

<table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>(py3) PS E:\TEMP\damai&gt; frida </code><code>-</code><code>-</code><code>version</code></p><p><code>16.0</code><code>.</code><code>19</code></p></td></tr></tbody></table>

环境搭建第二步是在 Android 模拟器中运行 frida-server。这样可以让 Frida 通过 ADB/USB 调试与我们的 Android 模拟器连接。

1.  下载 frida-server  
    最新的 frida-server 可以从 https://github.com/frida/frida/releases 下载。请注意下载与设备匹配的架构。如果您的模拟器是 x86_64，请下载相应版本的 frida-server。本文使用的版本为 [frida-server-16.0.18-android-x86_64.xz](https://archive.ph/o/OL9ZZ/https://github.com/frida/frida/releases/download/16.0.18/frida-server-16.0.18-android-x86_64.xz)  
    ![](https://archive.ph/OL9ZZ/470de2f2f5778461ec789b30294271ab2aafcf0f.png)
    
2.  传入 Android 模拟器。
    
    将下载后的. xz 文件解压，将`frida-server`传入 Android 模拟器
    
    <table><tbody><tr><td><p>1</p></td><td><p><code>adb push frida</code><code>-</code><code>server </code><code>/</code><code>data</code><code>/</code><code>local</code><code>/</code><code>tmp</code><code>/</code></p></td></tr></tbody></table>
3.  运行 frida-server
    
    使用 adb root 以 root 模式重新启动 ADB，并通过 adb shell 重新进入 shell 的访问。进入 shell 后，进入我们放置 frida-server 的目录并为其授予执行权限：
    
    <table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>cd </code><code>/</code><code>data</code><code>/</code><code>local</code><code>/</code><code>tmp</code><code>/</code></p><p><code>chmod </code><code>+</code><code>x frida</code><code>-</code><code>server</code></p></td></tr></tbody></table>
    
    执行：`./frida-server`，运行 frida-server，并保持本 shell 窗口开启。
    
    成功截图：
    
    ![](https://archive.ph/OL9ZZ/fa3a931e9ec6f45b773e6e9b84b13839f3482853.png)
    
    > 有些情况下，应用程序会检测在是否在模拟器中运行，但对于大麦网 app 的分析暂无影响。
    
4.  测试是否连接成功
    
    在 window 端运行 frida-ps 命令：
    
    ![](https://archive.ph/OL9ZZ/e0ba453390399ac00d43a11d22dfabf2d518068a.png)
    
    看到一堆熟悉的 Android 进程，我们就连接成功啦
    
5.  转发 frida-server 端口 (可选)
    
    frida-server 跑在 Android 端，frida 需要通过连接 frida-server。上一步使用 adb 的方式连接，frida 认为是 USB 模式，需要`-U`命令。frida 也支持依赖端口的远程连接模式，在某些场景下更加灵活。可以通过端口转发的方式实现此功能。
    
    <table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>adb forward tcp:</code><code>27042</code> <code>tcp:</code><code>27042</code></p><p><code>adb forward tcp:</code><code>27043</code> <code>tcp:</code><code>27043</code></p></td></tr></tbody></table>
    
    27042 是用于与 frida-server 通信的默认端口号, 之后的每个端口对应每个注入的进程，检查 27042 端口可检测 Frida 是否存在。
    

> 本部分主要参考了 https://learnfrida.info/java/ ， 在此感谢原文作者。

本章节将介绍用 tcpdump+frida+wireshark 实现 Android 的全流量抓包，能实现 https 解密。

惯用的 Android 抓包手段是用 fiddler/burpsuite/mitmproxy 搭建代理服务器，设置 Android 代理服务器并用中间人劫持的方式获取 http 协议流量的内容。如需对 https 流量解密，还需要在安卓上安装 https 根证书。Android9.0 以后的版本对用户自定义根证书有了一些限制，抓包不再那么简单，但这难不倒技术大神们，大家可以可以参考以下几篇文章：

*   [从原理到实战，全面总结 Android HTTPS 抓包](https://archive.ph/o/OL9ZZ/https://segmentfault.com/a/1190000041674464)
*   [Android 高版本 HTTPS 抓包解决方案](https://archive.ph/o/OL9ZZ/https://jishuin.proginn.com/p/763bfbd5f92e)

上述的抓包方式只能抓到 http 协议层以上的流量，这次我们来点不一样的，用 tcpdump+frida+wireshark 实现 Android 的全流量抓包，能实现 https 解密。

本文基于 termux 安装使用 tcpdump。

首先安装 termux apk。

![](https://archive.ph/OL9ZZ/1188ed707666fb7878fa97b43dfdce0f4ec2a308.png)

打开 termux 运行：

*   挂载存储
    
    <table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p></td><td><p><code>termux</code><code>-</code><code>setup</code><code>-</code><code>storage</code></p><p><code>## 会弹出授权框，点允许</code></p><p><code>ls ~</code><code>/</code><code>storage</code><code>/</code></p><p><code>## 如果出现dcim, downloads等目录，即表示成功</code></p></td></tr></tbody></table>
    *   安装 tcpdump<table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>pkg install root</code><code>-</code><code>repo</code></p><p><code>pkg install sudo tcpdump</code></p></td></tr></tbody></table>
*   运行抓包
    

<table><tbody><tr><td><p>1</p></td><td><p><code>sudo tcpdump </code><code>-</code><code>i </code><code>any</code> <code>-</code><code>s </code><code>0</code> <code>-</code><code>w ~</code><code>/</code><code>storage</code><code>/</code><code>downloads</code><code>/</code><code>capture.pcap</code></p></td></tr></tbody></table>

*   tcpdump 成功截图:  
    ![](https://archive.ph/OL9ZZ/d4044616b682c94288fc1f31c94fea3d4d9abfe1.png)

之后就可以把 downloads 目录下的抓包文件拷贝到电脑上，用 wireshark 打开做进一步分析。

Wireshark 解密 https 流量的方法和原理介绍有很多，可参考以下文章，本文不再赘述。

> *   https://unit42.paloaltonetworks.com/wireshark-tutorial-decrypting-https-traffic/
> *   https://zhuanlan.zhihu.com/p/36669377

wireshark 解密技术的重点在于拿到客户端通信的密钥日志文件 (ssl key log)，像下面这种：

![](https://archive.ph/OL9ZZ/1ffbfc31e1ddcccd0c63ae8cd9355d945224700b.png)

在 Android 中实现抓取 ssl key log 需要 hook 系统的 SSL 相关函数，可以用 frida 实现。

*   首先将下面的 hook 代码保存为`sslkeyfilelog.js`

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p></td><td><p><code>/</code><code>/</code> <code>sslkeyfilelog.js</code></p><p><code>function startTLSKeyLogger(SSL_CTX_new, SSL_CTX_set_keylog_callback) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(</code><code>"start----"</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>function keyLogger(ssl, line) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(new NativePointer(line).readCString());</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>const keyLogCallback </code><code>=</code> <code>new NativeCallback(keyLogger, </code><code>'void'</code><code>, [</code><code>'pointer'</code><code>, </code><code>'pointer'</code><code>]);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Interceptor.attach(SSL_CTX_new, {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>onLeave: function(retval) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>const ssl </code><code>=</code> <code>new NativePointer(retval);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>const SSL_CTX_set_keylog_callbackFn </code><code>=</code> <code>new NativeFunction(SSL_CTX_set_keylog_callback, </code><code>'void'</code><code>, [</code><code>'pointer'</code><code>, </code><code>'pointer'</code><code>]);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>SSL_CTX_set_keylog_callbackFn(ssl, keyLogCallback);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>});</code></p><p><code>}</code></p><p><code>startTLSKeyLogger(</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Module.findExportByName(</code><code>'libssl.so'</code><code>, </code><code>'SSL_CTX_new'</code><code>),</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Module.findExportByName(</code><code>'libssl.so'</code><code>, </code><code>'SSL_CTX_set_keylog_callback'</code><code>)</code></p><p><code>)</code></p></td></tr></tbody></table>

*   然后用 frida 加载运行 hook
    
    <table><tbody><tr><td><p>1</p></td><td><p><code>frida </code><code>-</code><code>U </code><code>-</code><code>l .\sslkeyfilelog.js&nbsp; </code><code>-</code><code>f cn.damai</code></p></td></tr></tbody></table>
    
    ![](https://archive.ph/OL9ZZ/3ee60d573c95df7040c505eb2eca9dcd5dd7237b.png)
    
*   最后，抓包结束后将得到的 key 保存到 sslkey.txt，格式是下面这样的，不要掺杂别的。
    

<table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>CLIENT_RANDOM </code><code>557e6dc49faec93dddd41d8c55d3a0084c44031f14d66f68e3b7fb53d3f9586d</code> <code>886de4677511305bfeaee5ffb072652cbfba626af1465d09dc1f29103fd947c997f6f28962189ee809944887413d8a20</code></p><p><code>CLIENT_RANDOM e66fb5d6735f0b803426fa88c3692e8b9a1f4dca37956187b22de11f1797e875 </code><code>65a07797c144ecc86026a44bbc85b5c57873218ce5684dc22d4d4ee9b754eb1961a0789e2086601f5b0441c35d76c448</code></p></td></tr></tbody></table>

在运行 Frida Hook 获取 sslkey 的同时，运行 tcpdump 抓包。抓包中依次测试获取详情页、选择价位、提交订单等操作，并对应记录下执行操作的时间，方便后续分析。

抓包完成后，用 wireshark 打开 tcpdump 抓包获得的 pcap 文件，在 wireshark 首选项 - protocols-TLS 中，设置 (Pre)-Master-Secret log filename 为上述 sslkey.txt。

![](https://archive.ph/OL9ZZ/feb0d7727e4d0663aba312b1b2c64993f131a4e9.png)

即可实现 https 流量的解密。

> 本部分主要参考了 https://www.52pojie.cn/thread-1405917-1-1.html ，向原作者表示感谢

在上述步骤中拿到了解密后的流量，我们就能对大麦网的流量做进一步分析了。

在此铺垫一下，通过前期对大麦网 PC 端和移动端 H5 的分析，大麦网购票的工作流程大概为：

1.  获得详情：接口为`mtop.alibaba.damai.detail.getdetail`。基于某演出的 id(itemId)获得演出的详细信息，包括详情、场次、票档 (SkuId) 价位及状态信息，
2.  构建订单：接口为`mtop.trade.order.build.h5`。发送 演出 id + 数量 + 票档 id(`itemId_count_skuId`)，得到提交订单所需的表单信息，包括观众、收货地址等。
3.  提交订单：接口为`mtop.trade.order.create.h5`。对上一步构建订单得到的表单参数作出修改后，发送给服务器，得到最后的订单提交结果和支付信息。

首先用过滤器`http && tcp.dstport==443`，得到向服务器发送的 https 包，如下图：

![](https://archive.ph/OL9ZZ/940ab623180e79de399b3d1b6af488354e07a222.png)

可以看到大量向服务器请求的数据包，但其中有很多干扰的图片请求，因为修改过滤器把图片过滤一下。过滤器：`http && tcp.dstport==443 and !(http.request.uri contains ".webp" or http.request.uri contains ".jpg" or http.request.uri contains ".png")`

结果清爽了很多。

#### [](#%E8%AE%A2%E5%8D%95%E6%9E%84%E5%BB%BA(order.build))订单构建 (order.build)

根据之前记录的操作的时间，以及对网页版的分析结果，笔者注意到了下图的这条流量：

![](https://archive.ph/OL9ZZ/153b9a3c715b6df92a43f2695635c5f4024892d8.png)

然后我们右键选择这条流量包，点击追踪 http 流，可以看到对应的响应包。

![](https://archive.ph/OL9ZZ/5f3145f77eb838566ea9b6dd4ede84bc17c709fa.png)

![](https://archive.ph/OL9ZZ/e8254bc26b3bfac942b8b79ad17f04522d3e0122.png)

响应包里有些中文使用了 UTF-8 编码，可以点击右下角的`Show data as`，选择 UTF-8，便可以正常显示。此时可以点击另存为，保存为 txt 文件，方便后续分析。

![](https://archive.ph/OL9ZZ/cd6f3cb60c4421ad6b29166136514097beee36a6.png)

订单构建的请求包中核心的数据部分为图中青色圈出来的部分，使用 URL 解码后为：

<table><tbody><tr><td><p>1</p></td><td><p><code>{</code><code>"buyNow"</code><code>:</code><code>"true"</code><code>,</code><code>"buyParam"</code><code>:</code><code>"716435462268_2_5005943905715"</code><code>,</code><code>"exParams"</code><code>:</code><code>"{\"atomSplit\":\"1\",\"channel\":\"damai_app\",\"coVersion\":\"2.0\",\"coupon\":\"true\",\"seatInfo\":\"\",\"umpChannel\":\"10001\",\"websiteLanguage\":\"zh_CN_#Hans\"}"</code><code>}</code></p></td></tr></tbody></table>

buyParam 为最核心的部分，拼接方式为演出 id + 数量 + 票档 id。其他部分只需照抄。

请求包中还包含大量的各种加密参数、ID，而破解实现自动购票脚本的关键就在于如何通过代码的方式拿到这些加密参数。

订单构建的响应包为订单提交表单的各项参数，用于生成 “确认订单” 的表单。

![](https://archive.ph/OL9ZZ/025e1c15e60cc12cb72c4a572e55f955156c9c38.png)

![](https://archive.ph/OL9ZZ/bea033129af289f5d92dd1ff79ef7fd25baefaf7.png)

#### [](#%E8%AE%A2%E5%8D%95%E6%8F%90%E4%BA%A4(order.create))订单提交 (order.create)

按照同样的方式可以找到订单提交包，订单提交包的 API 路径为`/gw/mtop.trade.order.create`，

![](https://archive.ph/OL9ZZ/0e239cb12c5649bfc74bec18b8e9d1a5cf3e4b23.png)

其中青色圈出来的部分为 data 发送的核心数据，对数据用 URL 解码后为：

<table><tbody><tr><td><p>1</p></td><td><p><code>{</code><code>"feature"</code><code>:</code><code>"{\"gzip\":\"true\"}"</code><code>,</code><code>"params"</code><code>:</code><code>"H4sIAAAAAAA.................AAWk3NKAAA\n"</code><code>}</code></p></td></tr></tbody></table>

看起来像是把原始数据用 gzip 压缩后又使用了 base64 编码，尝试解码：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p></td><td><p><code>import</code> <code>base64</code></p><p><code>import</code> <code>gzip</code></p><p><code>import</code> <code>json</code></p><p><code># 解码后变为python dict</code></p><p><code>decode_data</code><code>=</code><code>base64.b64decode(params_str.replace(</code><code>"\\n"</code><code>,""))</code></p><p><code>decompressed_data</code><code>=</code><code>gzip.decompress(decode_data).decode(</code><code>"utf-8"</code><code>)</code></p><p><code>params</code><code>=</code><code>json.loads(decompressed_data)</code></p><p><code>with </code><code>open</code><code>(</code><code>"reverse\order.create-params.json"</code><code>,</code><code>"w"</code><code>) as f:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>json.dump(params,f,indent</code><code>=</code><code>2</code><code>)</code></p></td></tr></tbody></table>

解码成功，存到`order.create-params.json`,

![](https://archive.ph/OL9ZZ/fafb2a4f84e7727dd3d0ae0e87966d2e2dc18076.png)

解码后发现 order.create 发送的 data 参数和 order.build 请求返回的结果很相似，增加了一些用户对表单操作的记录。

![](https://archive.ph/OL9ZZ/21cf8b7e6737d17a089faee8137102fd6b426925.png)

order.create 请求的 header 中的各种加密参数和 order.build 一致。

order.create 请求的返回结果中包含了订单创建是否成功的结果以及支付链接。

![](https://archive.ph/OL9ZZ/820b34de94621abe33436f01d0df48e7e02813d0.png)

通过前面对流量的分析，我们已经知道客户端向服务器发送的核心数据和加密参数，核心数据的拼接相对简单，但加密参数怎么获得还比较困难。因此，下面要开始分析加密参数的生成方法。本章节主要采用 frida trace 动态分析和 jadx 静态分析相结合的方式，旨在找到加密参数生成的核心函数和输入输出数据的格式。

运行`frida-trace -U -j "*InnerSignImpl*!*" 大麦`，执行选座提交订单的操作，发现确实有结果输出：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p></td><td><p><code>(py3) PS E:\TEMP\damai&gt; frida</code><code>-</code><code>trace </code><code>-</code><code>U </code><code>-</code><code>j </code><code>"*InnerSignImpl*!*"</code> <code>大麦</code></p><p><code>Instrumenting...</code></p><p><code>InnerSignImpl$</code><code>1.</code><code>$init: Loaded handler at </code><code>"E:\\TEMP\\damai\\__handlers__\\mtopsdk.security.InnerSignImpl_1\\_init.js"</code></p><p><code>....此处省略...</code></p><p><code>InnerSignImpl.init: Loaded handler at </code><code>"E:\\TEMP\\damai\\__handlers__\\mtopsdk.security.InnerSignImpl\\init.js"</code></p><p><code>Started tracing </code><code>27</code> <code>functions. Press Ctrl</code><code>+</code><code>C to stop.</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>*</code> <code>TID </code><code>0x144f</code> <code>*</code><code>/</code></p><p><code>&nbsp;&nbsp;</code><code>6725</code> <code>ms&nbsp; InnerSignImpl.getUnifiedSign(</code><code>"&lt;instance: java.util.HashMap&gt;"</code><code>, </code><code>"&lt;instance: java.util.HashMap&gt;"</code><code>, </code><code>"23781390"</code><code>, null, true)</code></p><p><code>&nbsp;&nbsp;</code><code>6726</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | InnerSignImpl.convertInnerBaseStrMap(</code><code>"&lt;instance: java.util.Map, $className: java.util.HashMap&gt;"</code><code>, </code><code>"23781390"</code><code>, true)</code></p><p><code>&nbsp;&nbsp;</code><code>6726</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | &lt;</code><code>=</code> <code>"&lt;instance: java.util.Map, $className: java.util.HashMap&gt;"</code></p><p><code>&nbsp;&nbsp;</code><code>6727</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | InnerSignImpl.getMiddleTierEnv()</code></p><p><code>&nbsp;&nbsp;</code><code>6727</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | &lt;</code><code>=</code> <code>0</code></p><p><code>&nbsp;&nbsp;</code><code>6737</code> <code>ms&nbsp; &lt;</code><code>=</code> <code>"&lt;instance: java.util.HashMap&gt;"</code></p></td></tr></tbody></table>

点击发送请求时，调用了 InnerSignImpl.getUnifiedSign 函数。但是输入参数和数据参数均为 HashMap 类型，结果中未显示具体内容。从结果输出中猜测 frida-trace 是通过对需要 hook 的函数在 **handlers** 下生成 js 文件，并调用 js 文件进行 hook 操作的，因此笔者修改了 “**handlers**\mtopsdk.security.InnerSignImpl\getUnifiedSign.js”，使其能正确输出 HashMap 类型。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p></td><td><p><code>/</code><code>/</code> <code>__handlers__\mtopsdk.security.InnerSignImpl\getUnifiedSign.js</code></p><p><code>{onEnter(log, args, state) {</code></p><p><code>&nbsp;</code><code>/</code><code>/</code> <code>增加了HashMap2Str函数，将HashMap类型转换为字符串</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>function HashMap2Str(params_hm) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var HashMap</code><code>=</code><code>Java.use(</code><code>'java.util.HashMap'</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var args_map</code><code>=</code><code>Java.cast(params_hm,HashMap);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>args_map.toString();</code></p><p><code>&nbsp;&nbsp;</code><code>};</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>/</code> <code>当调用函数时，输出函数参数</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>log(`InnerSignImpl.getUnifiedSign(${HashMap2Str(args[</code><code>0</code><code>])},${HashMap2Str(args[</code><code>1</code><code>])},${args[</code><code>2</code><code>]},${args[</code><code>3</code><code>]})`);</code></p><p><code>&nbsp;&nbsp;</code><code>}, onLeave(log, retval, state) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>function HashMap2Str(params_hm) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var HashMap</code><code>=</code><code>Java.use(</code><code>'java.util.HashMap'</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var args_map</code><code>=</code><code>Java.cast(params_hm,HashMap);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>args_map.toString();&nbsp;&nbsp;&nbsp; };</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>(retval !</code><code>=</code><code>=</code> <code>undefined) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>/</code> <code>当函数运行结束时，输出函数结果</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>log(`&lt;</code><code>=</code> <code>${HashMap2Str(retval)}`);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>} }}</code></p></td></tr></tbody></table>

再次运行 frida-trace，输出的结果已经可以看到具体内容了：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p></td><td><p><code>(py3) PS E:\TEMP\damai&gt; frida</code><code>-</code><code>trace </code><code>-</code><code>U </code><code>-</code><code>j </code><code>"*InnerSignImpl*!*"</code> <code>大麦</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>......</code></p><p><code>Started tracing </code><code>27</code> <code>functions. Press Ctrl</code><code>+</code><code>C to stop.</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>*</code> <code>TID </code><code>0x15ab</code> <code>*</code><code>/</code></p><p><code>&nbsp;&nbsp;</code><code>2653</code> <code>ms&nbsp; InnerSignImpl.getUnifiedSign({data</code><code>=</code><code>{</code><code>"itemId"</code><code>:</code><code>"719193771661"</code><code>,</code><code>"performId"</code><code>:</code><code>"211232892"</code><code>,</code><code>"skuParamListJson"</code><code>:</code><code>"[{\"count\":1,\"price\":36000,\"priceId\":\"251592963\"}]"</code><code>,</code><code>"dmChannel"</code><code>:</code><code>"*@damai_android_*"</code><code>,</code><code>"channel_from"</code><code>:</code><code>"damai_market"</code><code>,</code><code>"appType"</code><code>:</code><code>"1"</code><code>,</code><code>"osType"</code><code>:</code><code>"2"</code><code>,</code><code>"calculateTag"</code><code>:</code><code>"0_0_0_0"</code><code>,</code><code>"source"</code><code>:</code><code>"10101"</code><code>,</code><code>"version"</code><code>:</code><code>"6000168"</code><code>}, deviceId</code><code>=</code><code>null, sid</code><code>=</code><code>13abe677c5076a4fa3382afc38a96a04</code><code>, uid</code><code>=</code><code>2215803849550</code><code>, x</code><code>-</code><code>features</code><code>=</code><code>27</code><code>, appKey</code><code>=</code><code>23781390</code><code>, api</code><code>=</code><code>mtop.damai.item.calcticketprice, utdid</code><code>=</code><code>ZF3KUN8khtQDAIlImefp4RYz, ttid</code><code>=</code><code>10005890</code><code>@damai_android_8.</code><code>5.4</code><code>, t</code><code>=</code><code>1684828096</code><code>, v</code><code>=</code><code>2.0</code><code>},{pageId</code><code>=</code><code>, pageName</code><code>=</code><code>},</code><code>23781390</code><code>,null)</code></p><p><code>&nbsp;&nbsp;</code><code>2654</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | InnerSignImpl.convertInnerBaseStrMap(</code><code>"&lt;instance: java.util.Map, $className: java.util.HashMap&gt;"</code><code>, </code><code>"23781390"</code><code>, true)</code></p><p><code>&nbsp;&nbsp;</code><code>2655</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | &lt;</code><code>=</code> <code>"&lt;instance: java.util.Map, $className: java.util.HashMap&gt;"</code></p><p><code>&nbsp;&nbsp;</code><code>2655</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | InnerSignImpl.getMiddleTierEnv()</code></p><p><code>&nbsp;&nbsp;</code><code>2655</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; | &lt;</code><code>=</code> <code>0</code></p><p><code>&nbsp;&nbsp;</code><code>2662</code> <code>ms&nbsp; &lt;</code><code>=</code> <code>{x</code><code>-</code><code>sgext</code><code>=</code><code>JA2qmBOxRVDxFRzca3r9BZibqJqvn7uerZOriayYu4mpnKCeoJiunKGZu5qqyfmaqJqhmvqYr5n8zPyJqImpmbvLrImomqidu5m7m7uYu5u7mLuYu5u7m7ubqYmtiaiJqImoiaiJqImoiaiJu8</code><code>+</code><code>7iaCf</code><code>/</code><code>cypnruaqJqomruau5j8y7uau4mgiaiJqInf6fDIu5o</code><code>=</code><code>, x</code><code>-</code><code>umt</code><code>=</code><code>+</code><code>D0B</code><code>/</code><code>05LPEvOgwKIQ1x</code><code>+</code><code>SeV5wNE6NzOo, x</code><code>-</code><code>mini</code><code>-</code><code>wua</code><code>=</code><code>atASnVJw3vGX1Tw3Y</code><code>/</code><code>zDaVZkDUbLxOxtlUmgDOnIjMTBcMPMqQJLpnxoOWEL53Fq</code><code>/</code><code>OPcQZiMpDXWNvDz8UQkI5mtkZvIcDN1oxZnuH0M22LHKar4rnO</code><code>/</code><code>xm4LtAiniKgYtfgMGK3stXuCmvtE4raIhROimslSk7hCkxaL</code><code>/</code><code>DYuLzBLYwXmNyr9UZi1g, x</code><code>-</code><code>sign</code><code>=</code><code>azG34N002xAAK0H9KwNr3txWFMxzW0H7ROfkLQK</code><code>+</code><code>Db7ueJHktR</code><code>/</code><code>yP</code><code>/</code><code>0TcdPFzoYf36zd9lJYMsHCmYX3EcoFnJPMk2pxu0H7QbtB</code><code>+</code><code>0</code><code>}</code></p></td></tr></tbody></table>

可以看到返回结果中包含了 `x-sgext`,`x-umt`,`x-mini-wua`,`x-sign` 等加密参数。至此，前面的一大堆分析也算有了小的收获。但对比流量分析结果中的发送参数，还是缺失了很多参数。下面我们继续跟踪，找出剩下的参数。

调研发现淘系的 apk 都包含 mtopsdk，猜想会不会有公开的官方文档描述 mtopsdk 的使用方法，因此我们就找到了 [【阿里云 mtopsdk Android 接入文档】](https://archive.ph/o/OL9ZZ/https://help.aliyun.com/apsara/agile/v_3_5_0_20210228/emas/development-guide/android.html) 。其中介绍了请求构建的流程为，笔者重点关注了请求构建和发送的部分：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p></td><td><p><code>/</code><code>/</code> <code>3.</code> <code>请求构建</code></p><p><code>/</code><code>/</code> <code>3.1</code><code>生成MtopRequest实例</code></p><p><code>MtopRequest request </code><code>=</code> <code>new MtopRequest();</code></p><p><code>/</code><code>/</code> <code>3.2</code> <code>生成MtopBuilder实例</code></p><p><code>MtopBuilder builder </code><code>=</code> <code>instance.build(MtopRequest request, String ttid);</code></p><p><code>/</code><code>/</code> <code>4.</code> <code>请求发送</code></p><p><code>/</code><code>/</code> <code>4.2</code> <code>异步调用</code></p><p><code>ApiID apiId </code><code>=</code> <code>builder.addListener(new MyListener).asyncRequest();</code></p></td></tr></tbody></table>

因此我们不妨大胆一些，直接跟踪所有对 mtopsdk 中函数的调用。

<table><tbody><tr><td><p>1</p></td><td><p><code>(py3) PS E:\TEMP\damai&gt; frida</code><code>-</code><code>trace </code><code>-</code><code>U </code><code>-</code><code>j </code><code>"*mtopsdk*!*"</code> <code>大麦</code></p></td></tr></tbody></table>

![](https://archive.ph/OL9ZZ/450e08a4f50f2c071d7f6cf89ae0910d7a399fb5.png)

输出的结果大概有 2000 行，直接看太费劲，我们复制到文本编辑器里做进一步分析。

我们按照阿里的官方文档介绍的流程，对应可以找到在输出的 trace 中找到一些关键的日志。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p></td><td><p><code># MtopRequest初始化</code></p><p><code>&nbsp;&nbsp;</code><code>3249</code> <code>ms&nbsp; MtopRequest.$init()</code></p><p><code>&nbsp;&nbsp;</code><code>3249</code> <code>ms&nbsp; MtopRequest.setApiName(</code><code>"mtop.trade.order.build"</code><code>)</code></p><p><code>&nbsp;&nbsp;</code><code>3249</code> <code>ms&nbsp; MtopRequest.setVersion(</code><code>"4.0"</code><code>)</code></p><p><code>&nbsp;&nbsp;</code><code>3249</code> <code>ms&nbsp; MtopRequest.setNeedSession(true)</code></p><p><code>&nbsp;&nbsp;</code><code>3249</code> <code>ms&nbsp; MtopRequest.setNeedEcode(true)</code></p><p><code>&nbsp;&nbsp;</code><code>3249</code> <code>ms&nbsp; MtopRequest.setData(</code><code>"{\"buyNow\":\"true\",\"buyParam\":\"7191937661_1_51826442779\",\"exParams\":\"{\\\"atomSplit\\\":\\\"1\\\",\\\"channel\\\":\\\"damai_app\\\",\\\"coVersion\\\":\\\"2.0\\\",\\\"coupon\\\":\\\"true\\\",\\\"seatInfo\\\":\\\"\\\",\\\"umpChannel\\\":\\\"10001\\\",\\\"websiteLanguage\\\":\\\"zh_CN_#Hans\\\"}\"}"</code><code>)</code></p><p><code># MtopBuilder初始化</code></p><p><code>&nbsp;&nbsp;</code><code>3251</code> <code>ms&nbsp; MtopBuilder.$init(</code><code>"&lt;instance: mtopsdk.mtop.intf.Mtop&gt;"</code><code>, </code><code>"&lt;instance: mtopsdk.mtop.domain.MtopRequest&gt;"</code><code>, null)</code></p><p><code># MtopBuilder发送异步请求</code></p><p><code>3268</code> <code>ms&nbsp; MtopBuilder.asyncRequest()</code></p><p><code># 参数构建</code></p><p><code>3301</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; |&nbsp;&nbsp;&nbsp; |&nbsp;&nbsp;&nbsp; | InnerProtocolParamBuilderImpl.buildParams(</code><code>"&lt;instance: mtopsdk.framework.domain.MtopContext&gt;"</code><code>)</code></p><p><code>3391</code> <code>ms&nbsp;&nbsp;&nbsp;&nbsp; |&nbsp;&nbsp;&nbsp; |&nbsp;&nbsp;&nbsp; | &lt;</code><code>=</code> <code>"&lt;instance: java.util.Map, $className: java.util.HashMap&gt;"</code><code>,{wua</code><code>=</code><code>CofS_</code><code>+</code><code>7HCuvRCdz1EN8ICI6A4ZBCJwgY1hi</code><code>+</code><code>Bsivjcijs8GggmUxLQQUVTEQ5mYYtPuV7R2QNG5JEONIJRfmzjxFXMrs9AHdepIuqoJJJAyewWALprRnjIAu75t47Tm</code><code>/</code><code>RU9xRi7IEo9w0P2aCquLzf7uhiO8JEDSRK</code><code>/</code><code>ZdVhURBbof7reFtzEBoYYeIPgnwz7CL3kRlbyqyJcYKxO7ZmmVq1PtMXF2HGJqRSDjdv9l4mySJljIQzBmpX393L6eO1ZQVG1fpp6RaCRcFF</code><code>+</code><code>UgfjJXaeMFziHzfQF7KfUQZIeAJV</code><code>/</code><code>4GyVEE2f55RwPluOTuQubXQnq</code><code>+</code><code>qu41a0V5oyEOFXMoQRYFZzLOv3CjwkiIXsqJFeIHc</code><code>=</code><code>, x</code><code>-</code><code>sgext</code><code>=</code><code>JA0VLKcO8e9Fqqhj38VJuiwkHCUbIA8jGCwUNh0mDzYdIxQhFCcVJxskDyUedk0lHCUVJU4nGyZIc0g2HDYdJg90GDYcJRwiDyYPJA8kDyQPJA8kDyQPJA8nDyQPJQ8lDyUPJQ8lDyUPJQ82STYPLRlwSiUcNhwlHCUcNhw2HnFNNhw2Dy0PJQ8lD1JvfU42HA</code><code>=</code><code>=</code><code>, nq</code><code>=</code><code>WIFI, data</code><code>=</code><code>{</code><code>"buyNow"</code><code>:</code><code>"true"</code><code>,</code><code>"buyParam"</code><code>:</code><code>"719193771661_1_5182956442779"</code><code>,</code><code>"exParams"</code><code>:</code><code>"{\"atomSplit\":\"1\",\"channel\":\"damai_app\",\"coVersion\":\"2.0\",\"coupon\":\"true\",\"seatInfo\":\"\",\"umpChannel\":\"10001\",\"websiteLanguage\":\"zh_CN_#Hans\"}"</code><code>}, pv</code><code>=</code><code>6.3</code><code>, sign</code><code>=</code><code>azG34N002xAAKiYA2sv237H04abW2iYKIxaD3GVPak</code><code>+</code><code>JifYV0u6VzpriFiKiP</code><code>+</code><code>HuuF26BzWpVTClaOIGdjtibfQ99JomGiYKJhomCi, deviceId</code><code>=</code><code>null, sid</code><code>=</code><code>13abe677c5076a4fa3382afc38a96a04</code><code>, uid</code><code>=</code><code>2215803849550</code><code>, x</code><code>-</code><code>features</code><code>=</code><code>27</code><code>, x</code><code>-</code><code>app</code><code>-</code><code>conf</code><code>-</code><code>v</code><code>=</code><code>0</code><code>, x</code><code>-</code><code>mini</code><code>-</code><code>wua</code><code>=</code><code>a3gSvx5K5</code><code>/</code><code>NRy</code><code>/</code><code>W8</code><code>+</code><code>fDouCSQ6VSmMK3awHwo5X</code><code>+</code><code>IayY7JL5SwHtiL0soynSAvCobk01qRQ2fQcTvZWakhmhA9xlNOKdwvxdA5nZ4Tno2asO5e7EvSMj6yqVYAXZZUBjZPUOBw3vpH8L2GUq9Gi6MTszU57a58</code><code>+</code><code>hJE2BCGTVsxhRonDw1Nnxp74Ffm, appKey</code><code>=</code><code>23781390</code><code>, api</code><code>=</code><code>mtop.trade.order.build, umt</code><code>=</code><code>+</code><code>D0B</code><code>/</code><code>05LPEvOgwKIQ1x</code><code>+</code><code>SeV5wNE6NzOo, f</code><code>-</code><code>refer</code><code>=</code><code>mtop, utdid</code><code>=</code><code>ZF3KUN8khtQDAIlImefp4RYz, netType</code><code>=</code><code>WIFI, x</code><code>-</code><code>app</code><code>-</code><code>ver</code><code>=</code><code>8.5</code><code>.</code><code>4</code><code>, x</code><code>-</code><code>c</code><code>-</code><code>traceid</code><code>=</code><code>ZF3KUN8khtQDAIlImefp4RYz1684829318230001316498, ttid</code><code>=</code><code>10005890</code><code>@damai_android_8.</code><code>5.4</code><code>, t</code><code>=</code><code>1684829318</code><code>, v</code><code>=</code><code>4.0</code><code>, user</code><code>-</code><code>agent</code><code>=</code><code>MTOPSDK</code><code>/</code><code>3.1</code><code>.</code><code>1.7</code> <code>(Android;</code><code>9</code><code>;samsung;SM</code><code>-</code><code>S908E)}</code></p></td></tr></tbody></table>

笔者注意到了 InnerProtocolParamBuilderImpl.buildParams 函数的输出结果完全覆盖了需要的各类加密参数，其输入类型是 MtopContext。从 jadx 逆向的 apk 代码中可以找到 MtopContext 类，即包含 Mtop 生命周期的各个类的一个容器。

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p></td><td><p><code>public </code><code>class</code> <code>MtopContext {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public ApiID apiId;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public String baseUrl;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public MtopBuilder mtopBuilder;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public Mtop mtopInstance;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public MtopListener mtopListener;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public MtopRequest mtopRequest;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public MtopResponse mtopResponse;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public Request networkRequest;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public Response networkResponse;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public MtopNetworkProp </code><code>property</code> <code>=</code> <code>new MtopNetworkProp();</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public </code><code>Map</code><code>&lt;String, String&gt; protocolParams;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public </code><code>Map</code><code>&lt;String, String&gt; queryParams;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public ResponseSource responseSource;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public String seqNo;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>@NonNull</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>public MtopStatistics stats;</code></p><p><code>}</code></p></td></tr></tbody></table>

所以现在的问题变为如何能够构建出来 MtopContext，然后调用 buildParams 函数生成各类加密参数。

在写本文复盘分析过程的时候，笔者发现仅依赖 mtopsdk 的调用过程其实已经可以得到 MtopContext 的全部生成逻辑了。但所谓当局者迷，笔者在当时分析的时候还是一头雾水。因此在此也介绍一下笔者的思考逻辑。

当时看着 mtopsdk 的调用过程，感觉很复杂。但是猜想从用户点击操作 -> 业务代码 ->mtopsdk 的数据流，以及模块间高内聚低耦合的原则，所以猜想模块间的调用不会很复杂，所以笔者就想分析业务代码与 mtopsdk 的调用逻辑。所以就想跟踪主要业务代码的 trace。所以笔者继续跟踪 trace，运行`frida-trace -U -j "*cn.damai*!*" 大麦`，以分析`cn.damai`包的调用过程，在其中发现了 `NcovSkuFragment.buyNow()` 函数，看起来是和购买紧密相关的函数。又找到 DMBaseMtopRequest 类。

![](https://archive.ph/OL9ZZ/5343f28cd1b6d5999eaffc8201a6bffa1767f393.png)

但是在这里有点卡住了，因为只找到了构建 MtopRequest，并未在 cn.damai 的 trace 日志中并未发现其他对 mtop 的调用。

然后笔者又尝试搜索和 api(order.build) 相关的代码，找到了：

![](https://archive.ph/OL9ZZ/be27359d13cd7a23bac61a17138d0a0f99671cab.png)

然而并没有多大用处。

然后，作者又读了大量的源代码，终于定位到了 `com.taobao.tao.remotebusiness.MtopBussiness`这个关键类。

![](https://archive.ph/OL9ZZ/182f4cb27243b1ed4613876f2724dd06e3630ecc.png)

笔者本以为 com.taobao 开头的代码不是那么重要，所以最开始把这个类完全忽略了。但通过对源码的阅读，发现这个类是 motpsdk 中 MtopBuilder 类的子类，主要负责管理业务代码和 Mtopsdk 的交互。

因此我们继续通过 trace 跟踪 MtopBussiness 类。运行`frida-trace -U -j "*!*buyNow*" -j "com.taobao.tao.remotebusiness.MtopBusiness!*" -j "*MtopContext!*" -j "*mtopsdk.mtop.intf.MtopBuilder!*" 大麦`

![](https://archive.ph/OL9ZZ/5261963ef7b7e87c59e96cef248108491007db7d.png)

现在业务代码和 mtopsdk 的交互就很清晰了，红色的部分是业务代码的函数，绿色的部分是 mtopsdk 的函数。

通过以上对 trace 的分析，已经知道了具体执行的操作，因此我们可以使用 frida 编写 js 代码，直接调用 APK 中的类，实现功能调用。

先展示一个简单的示例，用于构建一个自定义的 MtopRequest 类:

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p></td><td><p><code>/</code><code>/</code> <code>new_request.js</code></p><p><code>Java.perform(function () {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>const MtopRequest </code><code>=</code> <code>Java.use(</code><code>"mtopsdk.mtop.domain.MtopRequest"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>let myMtopRequest </code><code>=</code> <code>MtopRequest.$new();</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>myMtopRequest.setApiName(</code><code>"mtop.trade.order.build"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>/</code><code>item_id </code><code>+</code> <code>count </code><code>+</code> <code>ski_id&nbsp; </code><code>716435462268_1_5005943905715</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>myMtopRequest.setData(</code><code>"{\"buyNow\":\"true\",\"buyParam\":\"716435462268_1_5005943905715\",\"exParams\":\"{\\\"atomSplit\\\":\\\"1\\\",\\\"channel\\\":\\\"damai_app\\\",\\\"coVersion\\\":\\\"2.0\\\",\\\"coupon\\\":\\\"true\\\",\\\"seatInfo\\\":\\\"\\\",\\\"umpChannel\\\":\\\"10001\\\",\\\"websiteLanguage\\\":\\\"zh_CN_#Hans\\\"}\"}"</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>myMtopRequest.setNeedEcode(true);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>myMtopRequest.setNeedSession(true);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>myMtopRequest.setVersion(</code><code>"4.0"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>console.log(`${myMtopRequest}`)</code></p><p><code>});</code></p></td></tr></tbody></table>

再使用运行命令 `frida -U -l .\reverse\new_request.js 大麦`，以在大麦 Apk 中执行 js hook 代码。运行之后即可输出笔者自己构建的 MtopRequest 实例。（frida 真的很奇妙！）

![](https://archive.ph/OL9ZZ/7809548d081526adb326b4ff0ba30cb9d72e232f.png)

有了上面的结果，下面继续完善这个示例，添加 MtopBussiness 的构建过程和输出过程

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p><p>29</p><p>30</p><p>31</p><p>32</p><p>33</p><p>34</p><p>35</p><p>36</p><p>37</p><p>38</p><p>39</p></td><td><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>/</code><code>引入Java中的类</code></p><p><code>const MtopBusiness </code><code>=</code> <code>Java.use(</code><code>"com.taobao.tao.remotebusiness.MtopBusiness"</code><code>);</code></p><p><code>const MtopBuilder </code><code>=</code> <code>Java.use(</code><code>"mtopsdk.mtop.intf.MtopBuilder"</code><code>);</code></p><p><code>/</code><code>/</code> <code>let RemoteBusiness </code><code>=</code> <code>Java.use(</code><code>"com.taobao.tao.remotebusiness.RemoteBusiness"</code><code>);</code></p><p><code>const MethodEnum </code><code>=</code> <code>Java.use(</code><code>"mtopsdk.mtop.domain.MethodEnum"</code><code>);</code></p><p><code>const MtopListenerProxyFactory </code><code>=</code> <code>Java.use(</code><code>"com.taobao.tao.remotebusiness.listener.MtopListenerProxyFactory"</code><code>);</code></p><p><code>const System </code><code>=</code> <code>Java.use(</code><code>'java.lang.System'</code><code>);</code></p><p><code>const ApiID </code><code>=</code> <code>Java.use(</code><code>"mtopsdk.mtop.common.ApiID"</code><code>);</code></p><p><code>const MtopStatistics </code><code>=</code> <code>Java.use(</code><code>"mtopsdk.mtop.util.MtopStatistics"</code><code>);</code></p><p><code>const InnerProtocolParamBuilderImpl </code><code>=</code> <code>Java.use(</code><code>'mtopsdk.mtop.protocol.builder.impl.InnerProtocolParamBuilderImpl'</code><code>);</code></p><p><code>/</code><code>/</code> <code>create MtopBusiness</code></p><p><code>let myMtopBusiness </code><code>=</code> <code>MtopBusiness.build(myMtopRequest);</code></p><p><code>myMtopBusiness.useWua();</code></p><p><code>myMtopBusiness.reqMethod(MethodEnum.POST.value);</code></p><p><code>myMtopBusiness.setCustomDomain(</code><code>"mtop.damai.cn"</code><code>);</code></p><p><code>myMtopBusiness.setBizId(</code><code>24</code><code>);</code></p><p><code>myMtopBusiness.setErrorNotifyAfterCache(true);</code></p><p><code>myMtopBusiness.reqStartTime </code><code>=</code> <code>System.currentTimeMillis();</code></p><p><code>myMtopBusiness.isCancelled </code><code>=</code> <code>false;</code></p><p><code>myMtopBusiness.isCached </code><code>=</code> <code>false;</code></p><p><code>myMtopBusiness.clazz </code><code>=</code> <code>null;</code></p><p><code>myMtopBusiness.requestType </code><code>=</code> <code>0</code><code>;</code></p><p><code>myMtopBusiness.requestContext </code><code>=</code> <code>null;</code></p><p><code>myMtopBusiness.mtopCommitStatData(false);</code></p><p><code>myMtopBusiness.sendStartTime </code><code>=</code> <code>System.currentTimeMillis();</code></p><p><code>let createListenerProxy </code><code>=</code> <code>myMtopBusiness.$</code><code>super</code><code>.createListenerProxy(myMtopBusiness.$</code><code>super</code><code>.listener.value);</code></p><p><code>let createMtopContext </code><code>=</code> <code>myMtopBusiness.createMtopContext(createListenerProxy);</code></p><p><code>let myMtopStatistics </code><code>=</code> <code>MtopStatistics.$new(null, null); </code><code>/</code><code>/</code><code>创建一个空的统计类</code></p><p><code>createMtopContext.stats.value </code><code>=</code> <code>myMtopStatistics;</code></p><p><code>myMtopBusiness.$</code><code>super</code><code>.mtopContext.value </code><code>=</code> <code>createMtopContext;</code></p><p><code>createMtopContext.apiId.value </code><code>=</code> <code>ApiID.$new(null, createMtopContext);</code></p><p><code>let myMtopContext </code><code>=</code> <code>createMtopContext;</code></p><p><code>myMtopContext.mtopRequest.value </code><code>=</code> <code>myMtopRequest;</code></p><p><code>let myInnerProtocolParamBuilderImpl </code><code>=</code> <code>InnerProtocolParamBuilderImpl.$new();</code></p><p><code>let res </code><code>=</code> <code>myInnerProtocolParamBuilderImpl.buildParams(myMtopContext);</code></p><p><code>console.log(`myInnerProtocolParamBuilderImpl.buildParams </code><code>=</code><code>&gt; ${HashMap2Str(res)}`)</code></p></td></tr></tbody></table>

再次执行`frida -U -l .\reverse\new_request.js 大麦`，输出结果如下图，此时已能根据笔者任意构建的请求 data 输出其他加密参数：

![](https://archive.ph/OL9ZZ/c9d9a059039ce582afacd6bfe9eaa32b26b3cc21.png)

对于 order.create 的原理类似，此处不再赘述。

通过 frida 调用 Apk 中的 Java 类有时候会出现找不到类的情况，原因可能是 classloader 没有正确加载。可以在 js 代码前的最前面加上下面的代码，指定正确的 classloader，即可解决该问题

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p></td><td><p><code>Java.perform(function () {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>/</code><code>get real classloader</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>/</code><code>/</code><code>from</code> <code>http:</code><code>/</code><code>/</code><code>www.lixiaopeng.top</code><code>/</code><code>article</code><code>/</code><code>63.html</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var application </code><code>=</code> <code>Java.use(</code><code>"android.app.Application"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var classloader;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>application.attach.overload(</code><code>'android.content.Context'</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>.implementation </code><code>=</code> <code>function (context) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>var result </code><code>=</code> <code>this.attach(context); </code><code>/</code><code>/</code> <code>run attach as it </code><code>is</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>classloader </code><code>=</code> <code>context.getClassLoader(); </code><code>/</code><code>/</code> <code>get real classloader</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Java.classFactory.loader </code><code>=</code> <code>classloader;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>result;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>});</code></p></td></tr></tbody></table>

通过 frida 操纵 Java 类的功能实在过于强大，安全人员可以执行以下操作：

1.  _打印函数输入输出_。通过 hook 函数，以实现打印函数的输入输出结果。  
    操作代码可以在 jadx 右键菜单可以很方便的生成。
    
    ![](https://archive.ph/OL9ZZ/2ae14441a19f2ef9e7e3ecde9fb93bf8280f5979.png)
    
    <table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p></td><td><p><code>let LocalInnerSignImpl </code><code>=</code> <code>Java.use(</code><code>"mtopsdk.security.LocalInnerSignImpl"</code><code>);</code></p><p><code>LocalInnerSignImpl[</code><code>"$init"</code><code>].implementation </code><code>=</code> <code>function (</code><code>str</code><code>, str2) {</code></p><p><code>console.log(`LocalInnerSignImpl.$init </code><code>is</code> <code>called: </code><code>str</code><code>=</code><code>${</code><code>str</code><code>}, str2</code><code>=</code><code>${str2}`);</code></p><p><code>this[</code><code>"$init"</code><code>](</code><code>str</code><code>, str2);</code></p><p><code>};</code></p></td></tr></tbody></table>
2.  _修改已有的类和函数_。
3.  _定义新类和新函数_。
4.  _主动生成类的实例或调用函数_。
5.  _RPC 调用_。通过 RPC 调用提供 python 编程接口。

前文提到 frida 的一个特性是可以通过 rpc 调用提供 python 编程接口。

一个简单的示例：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p></td><td><p><code>import</code> <code>frida</code></p><p><code>def</code> <code>on_message(message, data):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>message[</code><code>"type"</code><code>] </code><code>=</code><code>=</code> <code>"send"</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(</code><code>"[*] {0}"</code><code>.</code><code>format</code><code>(message[</code><code>"payload"</code><code>]))</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>else</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(message)</code></p><p><code># hook代码</code></p><p><code>jscode </code><code>=</code> <code>"""</code></p><p><code>rpc.exports = {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>testrpc: function (a, b) { return a + b; },</code></p><p><code>};&nbsp; """</code></p><p><code>def</code> <code>start_hook():</code></p><p><code># 开始hook</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>process </code><code>=</code> <code>frida.get_usb_device().attach(</code><code>"大麦"</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>script </code><code>=</code> <code>process.create_script(jscode)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>script.on(</code><code>"message"</code><code>, on_message)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>script.load()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>script</code></p><p><code>script </code><code>=</code> <code>start_hook()</code></p><p><code># 调用hook代码</code></p><p><code>print</code><code>(script.exports.testrpc(</code><code>1</code><code>, </code><code>2</code><code>))</code></p><p><code># &gt;&gt;&gt; 输出</code></p><p><code># 3</code></p></td></tr></tbody></table>

frida 使用 rpc 的方法也很简单，仅需使用 rpc.exports，将对应的函数暴露出来，就能被 python 调用。

完整的代码就是将上一章的代码封装为函数，并通过 rpc 对外提供接口，就可以了。为避免侵权，本文不贴出完整利用代码。

代码封装完成后测了一下，平均一次调用的时间为 0.024 秒，完全可以达到抢票的要求。

txthinking 放出了一个抓包辅助工具 [wiresharkhelper](https://archive.ph/o/OL9ZZ/https://github.com/txthinking/wiresharkhelper)，看视频介绍很诱人很方便，但是实测是要收费的。本人穷，所以就没用他的方法。然而也是因为这个才开始尝试用 frida 工具得到 https 的密钥，发现了 frida 这个神器。

细心的朋友可能发现发送的请求头里是包含 cookie 的，但是本文没有介绍。其实笔者本来是再继续找 cookie 的，但是发现把`InnerProtocolParamBuilderImpl.buildParams`函数的参数填进去之后，就已经能正常获取服务器的返回值了，所以就没继续搞 cookie

MtopStatistics 是 mtopsdk 里比较重要的一个类，用来跟踪用户的操作记录状态，可能有助于判断用户是否是机器人。但笔者尝试自己构建 MtopStatistics 失败，所以直接生成了一个空的 MtopStatistics 类，好在也没对服务器的正常返回造成影响。

这里笔者是直接用的大麦网 Web 端 PC 版，网页中有一段 json，包含静态的描述信息和动态的场次、余票信息。

目前是需要模拟器一直运行着的，而且仅能用一个人的账户。这对于个人使用是完全够用的。如何能脱离模拟器，而且增加并发用户数量还需要继续研究。目前时间不允许，暂时不再继续此问题的研究。

虽然流程全都搞定，而且对于非热门场次抢票完全没有问题。但对于热门场次，官方可能还是增加了或明或暗的检测机制。比如有些是淘票票限定渠道，在对特权用户开放抢票一段时间后才会对其他人，但开放状态仅从网页端无法判断，导致脚本会提前开抢，被系统提前拦截。或者有的场次明明第一时间开抢，却还是一直提示请求失败。这个还需要进一步踩坑理解大麦网的机制。

这篇公众号文章 (https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA4MDEzMTg0OQ==&action=getalbum&album_id=2885498232984993792#wechat_redirect) 介绍了大麦网的 bp 链接及使用方式，可以跳过票档选择直接进入订单确认页面。后续可以尝试用于自动抢票。

如：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p></td><td><p><code>-</code> <code>毛不易 </code><code>-</code>&nbsp; <code>5</code><code>月</code><code>27</code><code>日 上海站</code></p><p><code>1</code><code>、五层</code><code>480</code> <code>x </code><code>1</code><code>张</code></p><p><code>https:</code><code>/</code><code>/</code><code>m.damai.cn</code><code>/</code><code>app</code><code>/</code><code>dmfe</code><code>/</code><code>h5</code><code>-</code><code>ultron</code><code>-</code><code>buy</code><code>/</code><code>index.html?buyParam</code><code>=</code><code>718707599799_1_5008768308765</code><code>&amp;buyNow</code><code>=</code><code>true&amp;exParams</code><code>=</code><code>%</code><code>257B</code><code>%</code><code>2522channel</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>2522damai_app</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522damai</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>25221</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522umpChannel</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>2522100031004</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522subChannel</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>2522damai</code><code>%</code><code>2540damaih5_h5</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522atomSplit</code><code>%</code><code>2522</code><code>%</code><code>253A1</code><code>%</code><code>257D</code><code>&amp;spm</code><code>=</code><code>a2o71.project.sku.dbuy&amp;sqm</code><code>=</code><code>dianying.h5.unknown.value</code></p><p><code>2</code><code>、五层</code><code>480</code> <code>x </code><code>2</code><code>张</code></p><p><code>https:</code><code>/</code><code>/</code><code>m.damai.cn</code><code>/</code><code>app</code><code>/</code><code>dmfe</code><code>/</code><code>h5</code><code>-</code><code>ultron</code><code>-</code><code>buy</code><code>/</code><code>index.html?buyParam</code><code>=</code><code>718707599799_2_5008768308765</code><code>&amp;buyNow</code><code>=</code><code>true&amp;exParams</code><code>=</code><code>%</code><code>257B</code><code>%</code><code>2522channel</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>2522damai_app</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522damai</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>25221</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522umpChannel</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>2522100031004</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522subChannel</code><code>%</code><code>2522</code><code>%</code><code>253A</code><code>%</code><code>2522damai</code><code>%</code><code>2540damaih5_h5</code><code>%</code><code>2522</code><code>%</code><code>252C</code><code>%</code><code>2522atomSplit</code><code>%</code><code>2522</code><code>%</code><code>253A1</code><code>%</code><code>257D</code><code>&amp;spm</code><code>=</code><code>a2o71.project.sku.dbuy&amp;sqm</code><code>=</code><code>dianying.h5.unknown.value</code></p></td></tr></tbody></table>

本文完整的记录了笔者对于 Apk 与服务器交互 API 的解析过程，包括环境搭建、抓包、trace 分析、hook、rpc 调用。本文对于淘系 Apk 的分析可以提供较多参考。本文算是笔者第一次深入且成功的用动态 + 静态分析结合的方式，借助神器 frida+jadx，成功破解 Apk，因此本文的记录也较为细致的记录了作者的思考过程，可以给新手提供参考。

本文也有一些不足之处，如无法脱离模拟器运行、仅能单用户、抢票成功率仍不高等。对于这些问题，如果未来作者有时间，会再回来填坑。

本文作者为 m2kar，原文发表在 https://github.com/m2kar/m2kar.github.io/issues/21，转载请注明出处。

最后，欢迎大家多多提出问题相互交流。

最后于  2023-5-24 15:18 被 m2kar 编辑 ，原因： 更正章节编号错误