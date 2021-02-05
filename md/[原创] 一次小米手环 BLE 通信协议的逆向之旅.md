> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-259330.htm)

0x0 前言
------

​ 纯逆向新手，接触逆向两个月左右。

 

​ 手头有一个小米手环 4NFC 版本，内置小爱同学，能够语音对话，进而控制智能家居，做一些操作如设置闹钟等。因对其实现原理好奇，并且想拓展其功能，遂逆向。百度只有 Miband3 之前的研究，到了 Miband4 的新功能，我应该还是第一个。

0x1 逆向前准备
---------

​ 首先，要想使用小米手环中的语音助手，手环必须和手机保持连接，并且小米运动 APP 要保持后台运行。很显然，小米手环并没有内置语音识别的功能，猜测其应该是将声音传感器的数据发送给手机，然后手机进行语音识别，自然语义处理之后再发送给手环。

 

​ 而在安卓中，想要实现 BLE（低功耗蓝牙）通讯，必须要调用

```
boolean android.bluetooth.BluetoothGatt.writeCharacteristic(BluetoothGattCharacteristic characteristic)

```

​ 具体 API 可以参考 Google 的

 

​ 我们就从这里开始入手。

0x2 开始逆向
--------

​ 我们使用 Jadx 打开小米运动 APP，查看其源代码，发现大部分代码混淆得有点厉害，不过这不是问题。

 

​ 我们直接使用代码搜索功能搜索 “writeCharacteristic”，发现了我们想要的结果。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_MHBUU8NRGEN5JKZ.png)

 

​ 此处省略代码查找过程，最终我们找到了实际发送字节流的函数。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_FYPBTV733KB6N6S.png)

 

​ 我们继续向上查找引用，或许可以直接通过静态分析就知道它的数据包结构呢？结果...

 

![](https://bbs.pediy.com/upload/attach/202005/873064_6M4UR9N38DX7AZQ.png)

 

​ Jadx 反编译失败了...Emm.... 不管了，直接 Hook 吧。

 

​ 我们可以通过 Hook 发送数据包的函数，获取它到底发送了啥。当然也可以直接 Hook 安卓 API，防止遗漏。这里我们这里选择了后者。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_TW82S3YZQYMHR5D.png)

 

​ 成功了！这些包的结构都很明显，大部分都是字符串和一些整数，且没有加密，通过多次对比和猜测就能知道，就不深入讲了。接下来我们重点分析小爱同学的通讯协议和小米手环认证算法。

0x3 小爱同学的通讯协议
-------------

​ 为了抓到小爱同学的交互包，我们还需要 Hook 蓝牙的读取字节流的函数。在安卓里面，这个过程是通过 Callback 实现的，那我们需要 Hook 那个 Callback 函数。稍微一查就找到了：

```
com.xiaomi.hm.health.bt.O00000oO.O00000Oo.onCharacteristicChanged(BluetoothGatt bluetoothGatt, BluetoothGattCharacteristic bluetoothGattCharacteristic)

```

![](https://bbs.pediy.com/upload/attach/202005/873064_GA4HPTUNGXNBRTQ.png)

 

​ 这样我们也能抓到从手环发到 APP 的包了：

 

![](https://bbs.pediy.com/upload/attach/202005/873064_TTV3YJ599DH4K3N.png)

 

​ 因为我正向开发的经验还比较多，这里直接猜测就知道，前几个包仅涉及到一些状态的交换，如开始录音，切换 UI 等。后面的声音数据的解密才是重点。

 

​ 我观察这些声音数据包，发现第一个字节代表数据包类型，第二个字节代表数据包序号，应该是防止乱序之类的（不知道 BLE 有没有乱序这一说）。接下来应该是声音数据，初步猜测是 PCM，事实证明不是，还有一层编码。打印调用堆栈似乎是不可行的，我这里继续静态分析 APP。

 

​ 通过搜索一些关键词，我找到一些有意思的 ENUM。

 

​ 注意，这里向下追溯 onCharacteristicChanged 函数是没用的，因为大多数 APP 都是用 Intent 间接传递参数的，并不是直接调用解密函数。

 

​ ![](https://bbs.pediy.com/upload/attach/202005/873064_J2F8YHSGKQKMAUR.png)

 

![](https://bbs.pediy.com/upload/attach/202005/873064_E9GWBAEWCWPT393.png)

 

​ 我们只要向上查找这个枚举的引用，就找到了解析语音数据包的代码。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_BPH4TQAZPM8J8B2.png)

 

​ 嗯，看来是找对了。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_38CK9ZZ54YQPJC7.png)

 

​ 这里是解密函数，可以看到他在遍历一个数组，先将元素写到一个文件中，接着又调用了另一个函数，我们跟进去看看。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_ZCF9Y6647YKF3N8.png)

 

​ 这里果然调用了一个 decode 函数，并且通过字符串我们知道最后的格式应该是 pcm，16k。解密之前应该是 opus。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_3GDNBJ3ZCJXG3F6.png)

 

​ 发现他调用了一个 so，不过 opus 的解密代码是开源的，我们直接用开源库写一个 demo 试试。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_7H9GBAA8F28W27H.png)

 

![](https://bbs.pediy.com/upload/attach/202005/873064_FYE5YN7F2SACWRV.png)

 

![](https://bbs.pediy.com/upload/attach/202005/873064_Y52NRTU6VXNGRJA.png)

 

​ 成功了！这样我们就完成了从声音数据的捕获到声音数据的解密。

0x4 小米手环认证算法
------------

​ 小爱同学等相关功能在未认证的情况下是不能使用的，所以有一个认证过程。

 

​ 这个网上也有相关的文章，静态动态结合分析就行了，不过提醒一点，那个验证包的一个参数貌似变了，跟我网上搜索的文章讲的不一样。主要就是 AES 算法的 data 和 key。

 

![](https://bbs.pediy.com/upload/attach/202005/873064_K7ANSFG5H638MMT.png)

0x5 心得与后记
---------

​ 分析这个 APP 很简单，整个过程花费了一个下午，也没啥复杂的加密流程，软件本身也没保护，但还是学到很多，比如：

1.  从软件调用的 API 考虑，向上回溯查找
    
2.  关键词查找
    
3.  动态静态结合分析
    
4.  掌握一些软件设计模式（软件工程这门学问），APP 正向开发思路，能更快更好的猜想。
    
    其实，还有一个更简单的方法，那就是 Hook 那个打印日志的函数，等我一波骚操作后才发现，小米运动的日志是真详细（逃）。
    
    再贴一个和 github 老哥亲切交谈的图，貌似小米手环要走向国际化了呢。
    

![](https://bbs.pediy.com/upload/attach/202005/873064_SQ6QAFGFSK9ZSA2.png)

 

![](https://bbs.pediy.com/upload/attach/202005/873064_FUESU68PF3XPTXH.png)

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)