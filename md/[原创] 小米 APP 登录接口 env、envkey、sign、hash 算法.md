> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266334.htm)

0x00 前言
-------

最近大家都在抢茅台酒，我也来凑凑热闹，我也好想抢到一瓶~，上次也发过一个酒仙 APP 的分析帖子了 [上次分析贴](https://bbs.pediy.com/thread-266240.htm)；这次正好又碰到了一个小米有品的登录接口，碰到有好几个加密的数据，这次我们继续本着技术学习的方向去研究一下。

0x01 整理
-------

![](https://bbs.pediy.com/upload/attach/202103/885524_8Z542XM2V6TRV93.png)  
这就是登录时发送的登录数据接口传递的参数，其中我们看到加密数据包括：HASH、envKey、env、_sign 这四项，我们逐一进行查找

0x02 定位
-------

首先第一步就是接口的定位，我们可以通过搜索接口名称（serviceLoginAuth2）或者搜索它传递的参数名称进行查找，这得看搜索的结果来决定用哪种好了； 这里我直接搜索名称 serviceLoginAuth2  
![](https://bbs.pediy.com/upload/attach/202103/885524_K8YUVSZTVZJ69HJ.png)  
这里一共 3 条结果，明显最后 1 条并不是，因为多了几个字符，然而前面 2 条是一样的，我们随便点进去，然后进行引用跟踪  
![](https://bbs.pediy.com/upload/attach/202103/885524_582GK4ECU3S84H2.png)  
我们直接选择最后一条，进去查看  
![](https://bbs.pediy.com/upload/attach/202103/885524_SY6SWPA4RXZT9RB.png)  
这条函数简直是非常的清晰，我们可以看到几乎这里做的操作跟接口传的参数都差不多都有，这里我把代码复制到 vscode 里面进行重命名修改，以便方便阅读。

0x02 证实
-------

接下来就要证实它是不是确实调用这个接口，我们有请 frida 登场（frida：“低调，低调”）。接下来编写 hook 代码~  
![](https://bbs.pediy.com/upload/attach/202103/885524_NRXACX678CN4QRF.png)  
因为这个 APP 正好使用了阿里的 json，所以我就直接调用它的方法来格式化输出整个类的属性，就非常方便！后面我们开启 HOOK 输入账号密码登录一下~  
![](https://bbs.pediy.com/upload/attach/202103/885524_MYBDVCZDG36RMRC.png)

0x03 分析
-------

成功拿到数据了，证明该方法确实被触发，我们整理一下代码：

```
{
  "returnStsUrl": false,
  "password": "11112222",
  "deviceId": "g7bqCchccIn5oue0",
  "captCode": "",
  "userId": "1376764646",
  "serviceId": "miotstore",
  "hashedEnvFactors": [
    "UzqtqM",
    "FtRqzw",
    "EsfzyR#UzqtqM",
    "MQ==",
    "TmV4dXMgNlA=",
    "RU5VN04xNjMyMzU0Njk3OA==",
    "XqFbOh",
    "g7bqCc",
    "OdHsAk",
    "1abbBq",
    "null",
    "null",
    "",
    "",
    "",
    "TmV4dXMgNlA="
  ],
  "captIck": "",
  "needProcessNotification": true
}

```

这里我们直接把刚刚复制的代码全部重命名，然后设置固定的值，这样方便我们进行阅读。  
![](https://bbs.pediy.com/upload/attach/202103/885524_JM83FDXBUANK4ZZ.png)

 

这里我们慢慢的进行按需分析，我们目前就想知道 HASH、envKey、env、_sign 这四项参数，首先第一项 HASH，我们看到 VSCODE 上的第 17 行代码

```
.easyPut("hash", CloudCoder.getMd5DigestUpperCase(password))

```

经过手动 MD5 签名对比了一下确实如此，由此得知 HASH = 大写 MD5(密码)。  
接下来我们发现代码中并未出现 envKey，env 以及_sign。

 

至于_sign 我也不想卖关子了。。其实我也是经过各种查找无果，发现居然这是服务器返回的包里面包含的，天呐~  
![](https://bbs.pediy.com/upload/attach/202103/885524_EZ5BBMX4BFJ4FXS.png)  
![](https://bbs.pediy.com/upload/attach/202103/885524_X9HPSFNZZFP5PVA.png)  
所以我们得知经过了几小时的不懈努力的无脑搜关键字，最终知道_sign 是在服务器的接口内返回，由此得知

 

HASH = MD5（密码）  
_SIGN = 服务器接口返回

0x04 深入虎穴
---------

最终我们还需要查找 env 以及 envKey，我们从当前的方法看不出有直接赋值，但是我们发现。它是通过新建 easyPut 一个类，把参数都传进去；所以我们发现它还把 easyPut 传到了一个 addEnvToParams 方法里面（第 23 行代码）

```
String[] hashedEnvFactors = ["UzqtqM","FtRqzw","EsfzyR#UzqtqM","MQ==","TmV4dXMgNlA=","RU5VN04xNjMyMzU0Njk3OA==","XqFbOh","g7bqCc","OdHsAk","1abbBq","null","null","","","","TmV4dXMgNlA="];
addEnvToParams(easyPut, hashedEnvFactors);

```

所以我们只能切换到 jadx 界面，跟进 addEnvToParams 看看在作何操作~  
![](https://bbs.pediy.com/upload/attach/202103/885524_BNHZWNVX7W357BD.png)  
我们发现了一个醒目的字眼，envKey，它也同样是调用 easyPutOpt 方法进行插入，我们看到在它上面也进行了一次插入，名字是个变量，我们跟进去看看  
![](https://bbs.pediy.com/upload/attach/202103/885524_C9MB7B7R6QCGXKP.png)  
终于，我们看到了 env 和 envkey 是这样赋值的，但这并没有结束，因为我们知道，它传入了一个 string 数组进去处理。 所以即使我们知道了 env 和 envKey 的加密方法，但还需要知道传进去的字符串数组，也就是 hashedEnvFactors 是从何而赋值的。

0x05 env 算法分析
-------------

我们暂且不管 hashedEnvFactors 从何而来，先看看 env 的算法

```
PassportEnvEncryptUtils.EncryptResult encrypt = PassportEnvEncryptUtils.encrypt(strArr);
easyMap.easyPutOpt(com.xiaomi.verificationsdk.internal.Constants.e, encrypt.content);

```

我们看到，它是调用 PassportEnvEncryptUtils.encrypt(strArr); 直接返回的 encrypt，并且直接取出 encrypt 里的 content（对应 env）和 encryptedKey（对应 envKey）得到的结果。所以直接跟进 encrypt 方法里面查看

 

第一层跟进  
![](https://bbs.pediy.com/upload/attach/202103/885524_GB2HERXNMSYQCR5.png)  
第二层跟进  
![](https://bbs.pediy.com/upload/attach/202103/885524_ZPH4N2MZENAS6W6.png)  
发现一个常量，去看下是什么值  
![](https://bbs.pediy.com/upload/attach/202103/885524_JUZ4X2AEHDXA8GM.png)  
发现值："0102030405060708"，继续返回跟进  
![](https://bbs.pediy.com/upload/attach/202103/885524_FV8GYW8MURP4MNP.png)  
最终我们看到它是进行的一个 ASE 算法，整个结果其实也是相当简单的了，我们把代码抠出来 VSCODE 分析下。  
![](https://bbs.pediy.com/upload/attach/202103/885524_VCYXBAJ88DAWDRH.png)  
首先参数 1 则是我们第一层看到它，他是把 hashedEnvFactors 数组用：分割连接起来了，所以我们写死他。  
而参数 2 就是第三层看到的 DEFAULT_IV，我们也写死  
![](https://bbs.pediy.com/upload/attach/202103/885524_PU6D5QYGV6EGB67.png)  
所以它的传参应该是这样的；接着我们可以看到，它 new 了自己封装的一个 ASE 的 EncryptResultWithIv 类，然后赋值参数 2 给 iv，估计就是加密的 KEY。

```
SecretKey generateSymmetricKey = generateSymmetricKey();

```

关键在于它又调用了这行代码，generateSymmetricKey()，我们进去看一下~  
![](https://bbs.pediy.com/upload/attach/202103/885524_KWH7VWQXUVQEGJE.png)  
好像也并不是很关键 0.0，就是随机初始化了一个 ASE 然后返回它的 key。那么我们继续阅读刚刚的代码，因为它连接的有一点长，我们把他格式处理一下。

 

![](https://bbs.pediy.com/upload/attach/202103/885524_45QCBFEM3VP5EGD.png)  
因为这里的计算感觉有点绕，前 3 行代码图片已经打出注释了，我们看下第接下来一行就是赋值跟 env 结果的关键代码，它又跑去执行了一个 aesEncrypt 方法，并且我们看到传入的正是 hashedEnvFactors 以及随机证书的 KEY 以及刚刚看到的 0102030405060708 的 IV 进行调用的，所以我们还需要跟进 aesEncrypt 查看一下都是怎么处理的。  
![](https://bbs.pediy.com/upload/attach/202103/885524_MR9YWQTR6AERF3X.png)  
这里我们看到它以随机生成的证书作为构造类定义，然后在传入要加密的  
hashedEnvFactors 以及 IV，我们在进去看看。  
![](https://bbs.pediy.com/upload/attach/202103/885524_P2SJJKZUSEYMAWR.png)

 

这里跟进去后发现就是调用 encryptWithIv 进行 ASE 签名的，然后在转成 BASE64 完成 env 的加密！！

0x06 总结
-------

至此我们已经完成了 4 项参数的算法~  
HASH = 大写 MD5(密码)  
_SIGN = 来自服务器返回  
ENVKEY = BASE64 编码 (RSA(一个随机证书, 公共密钥))  
ENV = BASE64 编码 (ASE(hashedEnvFactors，"0102030405060708"))

 

这些完全可以直接扣掉它自己定义的 ASE 类来放到 JAVA 独立调用的，就不再作演示，由于文章挺长了，下次在讲一讲 hashedEnvFactors 这 10 几个字符串的定义来源

[[公告] 名企招聘！](https://job.kanxue.com/position-list-1-99-99-99-99-99.htm)