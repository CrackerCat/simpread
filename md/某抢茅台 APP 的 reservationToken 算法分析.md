> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266240.htm)

0x00 前言：
--------

在调用下单接口（addorder.html），里面包含一个参数名称为 reservationToken，它的值是一串类似 md5 的十六进制。这里分享下逆向追踪过程，目前已实现。

0x01 算法定位：
----------

拖入 APK 到 jadx，直接搜索关键字 reservationToken，我们看到图中的代码

 

![](https://bbs.pediy.com/upload/attach/202103/885524_6S6YDJJ64TXARSD.png)

 

其中 orderPrepareParameter2.mReservationToken = MD5Util.encode("ANDROID" + W.e() + X.b()); 是根据 3 个字符串进行拼接在执行的操作，那么我们接下来就来查找这 2 个方法的字符串。

 

目前已知：md5("ANDROID" + ？ + ？)

0x02 W.e() 算法扣出：
----------------

我们追入：W.e() 内进行查看  
![](https://bbs.pediy.com/upload/attach/202103/885524_9ZFX4GD8U9YVHGQ.png)  
我们可以看到，它这里是直接把结果存入的 f26606b 进行返回的，由于该 APP 它都做了缓存判断，所以如果直接下 frida 的话，也没办法直接 HOOK 到它生成的过程，但是我们发现他是调用的 static 方法 UTDevice.getUtdid 获得的，我们可以编写 frida 代码主动调用，然后看看它输出的结果，以下是 frida 代码：

 

![](https://bbs.pediy.com/upload/attach/202103/885524_TTJRU4EC3TRVRV4.png)

 

编写好代码直接运行，因为是主动调用的，无需等待 hook， ![](https://bbs.pediy.com/upload/attach/202103/885524_TZTDWFRTR8Q65V5.png) 那么我们看到是一串字符串，感觉有点像 base64 编码。

 

这部操作的意义并不是很大，只是知道了 W.e() 的返回值，但我们需要知道它的生成过程，所以继续 UTDevice.getUtdid 跟进。

 

我们继续跟进去后，发现它是调用的 b.b(context) 获取的一个 class 类，然后在调用 f() 返回的结果：  
![](https://bbs.pediy.com/upload/attach/202103/885524_QZPP8U7SB2Z8PP7.png)

 

那么我们继续跟如 b.b() 内的实现部分  
![](https://bbs.pediy.com/upload/attach/202103/885524_M77SQTACYQVQ4XR.png)

 

这里我们看到，它先判断 f13187a 是不是 null，如果是的话，继续调用 a(context) 创建，这些都是它的缓存机制判断，我们继续跟进 a 内

 

![](https://bbs.pediy.com/upload/attach/202103/885524_KSRC2MJ8VAHRQQZ.png)

 

这里就是它创建 class 类的参数赋值代码了，我们知道，刚刚它是调用 b.b(context) 然后调用 f() 来得到的，我们现在去看看 f() 它返回的是对应哪个属性，这样就知道这里对应的是哪个值了。

 

![](https://bbs.pediy.com/upload/attach/202103/885524_CT8BF6QZHNH6XVA.png)

 

我们发现，f 是返回的 f13186g，而 f13186g 是由 e 方法赋值的，那么上一张截图，我们看到，e 方法就是传入的 value 这个值。aVar.e(value) 也就是上一张截图的这行代码了，那么我们就继续跟进 value 的方法，也就是 String value = c.a(context).getValue(); 里面的过程

 

![](https://bbs.pediy.com/upload/attach/202103/885524_NZPCTSMKMQ2JWHK.png)

 

我们这里可以看到，它调用的 h() 方法，继续跟进，事情逐渐变得好玩了。

 

![](https://bbs.pediy.com/upload/attach/202103/885524_MKTM6JYRWEM4WWH.png)

 

这里代码比较多，我们先来一下，它是调用的 this.h = i(); ，如果不是空的，就直接返回，其实这里也是获取缓存的值，我们跟进 i() 里面，由于这个方法代码反编译失败，都是一片绿色，阅读比较吃力，但是经过详细的阅读得出就是读取的缓存，所以这行忽略，我们往下看：  
byte[] c2 = m5c();

 

this.h = b.encodeToString(c2, 2);

 

得出 2 行关键代码，这里它通过调用 m5c() 获得一个 byteArr 数组，接着进行 base64 编码后返回，也就是 m5c 里面估计就是关键生成了。

 

![](https://bbs.pediy.com/upload/attach/202103/885524_8FQ7XPGW9SQB9AK.png)

 

这里基本大部分都是 Android 的 SDK 代码，我们可以直接新建一个 java 工程，直接复制他的代码，将报错地方纠正。

 

![](https://bbs.pediy.com/upload/attach/202103/885524_55XY88D3YCP7YRH.png)

 

我们扣出代码，发现报错的地方就是 d.getBytes 、 e.a(this.mContext) 、 g.a 、b 四处地方，我们逐一进去查看，首先查看 d.getBytes（）

 

![](https://bbs.pediy.com/upload/attach/202103/885524_SHNZDPWAPGA4RUP.png)

 

它就是进行的一个位移操作，我还以为是什么复杂的类，我们直接复制出来就好了，重新命名下。

 

![](https://bbs.pediy.com/upload/attach/202103/885524_4FDG3BUY6FDFB2X.png)

 

直接复制出来，我重新命名成了 DgetBytes，然后把 d.getBytes 改成这个方法名，就解决了，那么我们继续查看下一个方法 e.a(this.mContext) ，跟进

 

![](https://bbs.pediy.com/upload/attach/202103/885524_ADXE6K324SXBGR4.png)

 

这个方法也很简单，就是获取手机的 deviceId，下面的就是判断如果获取不到执行别的获取方案，就不跟进了。 我们直接写死 deviceId，也就是 IMEI 码就行了，到时候在封装成一个传参的方式即可。  
最后跟进 g.a() 里面查看

 

![](https://bbs.pediy.com/upload/attach/202103/885524_5CCH4JUTESXPEBD.png)

 

这也是一个相加操作，很简单，也直接跟刚刚的方式一样扣出来，我重新命名成了 ga，然后修改下调用的代码即可。

 

还剩最后一个 b() 方法，这个也是可以直接扣的

 

![](https://bbs.pediy.com/upload/attach/202103/885524_N7GBTFKU6FK4QJN.png)

 

我们重新命名成了 be，那么看下完全扣完的代码，一点报错没有了  
![](https://bbs.pediy.com/upload/attach/202103/885524_ZEU9TUAUV6YB6QQ.png)

 

后面我们在进行 BASE64 编码就完成了第二个参数的生成过程了！！ 如果要转成其它语言代码的话，就看你的代码阅读能力了，因为都是直接扣出来的~

 

目前已知：md5("ANDROID" + 算法 A(IMEI) + ？)

 

这里我就定义成它叫做算法 A，他的调用过程就是 Base64.getEncoder().encodeToString(m5c()); 的返回，接下来我们解析第三个参数

0x03 X.b() 算法扣出：
----------------

![](https://bbs.pediy.com/upload/attach/202103/885524_3VGE78CJJADMGVX.png)

 

我们继续进行扣代码，继续跟进 X.b() 内查看  
![](https://bbs.pediy.com/upload/attach/202103/885524_4TNYGBXKQUN8ZEM.png)

 

这个方法很简单，看着代码挺多，我们慢慢阅读解析。  
首先它调用 String packageName = AppContext.getContext().getPackageName() 获取当前 APP 的包名，然后获取缓存，看看有没有记录 device_id 这个值，我们要知道他的生成过程。  
如果没有，那么它继续调用 Settings.Secure.getString(AppContext.getContext().getContentResolver(), "android_id")，这个就是获取系统参数的 android_id  
如果获取的值是一个它固定的值 9774d56d682e549c 它就拿来返回了。  
最终我们肯定确定，它就是调用  
String deviceId = ((TelephonyManager) AppContext.getContext().getSystemService("phone")).getDeviceId();  
f8203a = (deviceId != null ? UUID.nameUUIDFromBytes(deviceId.getBytes("utf8")) : UUID.randomUUID()).toString();

 

获取手机的 IMEI，然后在通过 nameUUIDFromBytes 进行格式化，自此就结束了~

 

其实我们用 frida HOOK 发现，它返回的值跟我们抓包看到的 APPKEY 是一样的值。

 

目前已知：md5("ANDROID" + 算法 A(IMEI) + UUID 格式化 (IMEI))

0x04 算法复原：
----------

![](https://bbs.pediy.com/upload/attach/202103/885524_T9QVDN94B78GTSU.png)

 

最后我们根据上面已知的生成过程，成功扣出算法。

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 3 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)