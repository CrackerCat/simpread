> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265404.htm)

APP 抓包问题已经是老生常谈的一个问题了，今天正好碰到这个问题，经过一番折腾最终解决了这个问题。
=================================================

### 先解决安卓 7 手机 / 模拟器正常抓包问题：

1、先把 burp 的 der 证书导出

![](https://bbs.pediy.com/upload/attach/202101/777761_3MS9CDA8ZR6PS8N.jpg)

![](https://bbs.pediy.com/upload/attach/202101/777761_JMRJNWD8GBKMWVU.jpg)

![](https://bbs.pediy.com/upload/attach/202101/777761_U43BU86QK9A4NTB.jpg)

2、使用 opensl 对证书进行相应的配置

将 der 转换成 pem，命令如下：

```
openssl x509 -inform DER -in cacert.der -out cacert.pem

```

![](https://bbs.pediy.com/upload/attach/202101/777761_JSTXVMYCNHWXYFC.jpg)

3、查看 pem 证书的 hash 值并记录，命令如下：

```
openssl x509 -inform PEM -subject_hash_old -in cacert.pem |head -1

```

![](https://bbs.pediy.com/upload/attach/202101/777761_3YQERB2RGJSXJPY.jpg)

4、将 pem 证书改名为 “hash 值. 0”：

```
mv cacert.pem 9a5ba575.0

```

![](https://bbs.pediy.com/upload/attach/202101/777761_82TJBCR4WR3MY4P.jpg)

5、将证书上传到手机 / 模拟器：

使用 root 权限启动

```
adb root

```

![](https://bbs.pediy.com/upload/attach/202101/777761_UX524SY93KQ5FB7.jpg)

6、重新安装分区读写：

```
adb remount

```

![](https://bbs.pediy.com/upload/attach/202101/777761_UTZJXHGA9HE4QGT.jpg)

7、把 9a5ba575.0 证书文件复制到手机 / 模拟器的系统证书文件夹，并设置 644 权限，设置完成后重启手机 / 模拟器： 以下路径与手机 / 模拟器路径一致（如：/sdcard/Download/9a5ba575.0）

```
adb shell
mv /sdcard/Download/9a5ba575.0 /system/etc/security/cacerts/
chmod 644 /system/etc/security/cacerts/9a5ba575.0

```

![](https://bbs.pediy.com/upload/attach/202101/777761_3BNSZTG6GKWA2C5.jpg)

8、重启完成后查看证书是否为系统级信任证书

![](https://bbs.pediy.com/upload/attach/202101/777761_A2NY2DV3CG9ZY4V.jpg)

至此，安卓 7 的 burp 证书已被系统信任（上述演示的模拟器为逍遥模拟器，这个模拟器我也只是偶尔用一下，其他手机 / 模拟器操作步骤差不多）, 但是经过以上一顿乱搞后，你会发现抓手机 / 模拟器浏览器的包时还是会弹出证书的问题，被测 APP 如果不是证书绑定（SSL Pinning）和双向认证的问题，还是可以正常进行抓包测试的。我安卓 7 用的少，因为证书问题有点麻烦，基本上都是用安卓 5 进行测试，除非一些 APP 只能安卓 7 以上版本才能运行，才会用到安卓 7 进行测试。

开始进入正题（突破双向认证）
==============

#### 之前有看到过双向认证证书的突破思路，今天正好在项目中碰到了，而且 app 又正好没有加固，免除了脱壳烦恼，因此有了以下突破过程：

#### 刚开始不知道 APP 做了双向证书校验，结果杯具就发生了。

![](https://bbs.pediy.com/upload/attach/202101/777761_Y54J797BUM2YJDW.jpg)

#### 然后开始寻找之前某大佬提供的突破思路

#### 因为 app 没有进行加固，可以直接用 jadx 反编译出来，然后全局搜索 ".pfx"、".pfx"、"PKCS12"、"keyStore" 等等关键字，我这里搜索的是 client.pfx。

![](https://bbs.pediy.com/upload/attach/202101/777761_NVQBZHBKMK498XT.jpg)

转到代码位置查看详情（证书安装密码、其他密码等等信息），可以看到，pfx 证书没有设置密码（本地测试安装了下，看到有需要输入密码，第一次尝试过输入密码，结果提示密码错误，第二次尝试不输入密码却安装成功了，于是开启了全局搜索之旅，看到没有设置证书安装密码）：

![](https://bbs.pediy.com/upload/attach/202101/777761_4GQ4TGRYU2DEQ22.jpg)

1、将 APP 以压缩包形式解压出来

![](https://bbs.pediy.com/upload/attach/202101/777761_HRF9AT7KV2BYWCD.jpg)

2、进入解压出来的目录，可以搜索一些证书的后缀文件，例如 cer/p12/pfx 等，一般安卓下的为 bks，也可以先去 assets 或者 res 目录下去找找。我碰到的 apk 就在 assets 目录下存放：

![](https://bbs.pediy.com/upload/attach/202101/777761_Q6AUA7NB5F2W3CE.jpg)

3、本来以为还得在文件夹里面继续找 key，找了一圈无果，然后咨询了某大佬，大佬让我直接把 key 导出来，因为之前没遇到过，觉得有点新鲜，就刚了一波

![](https://bbs.pediy.com/upload/attach/202101/777761_CMS7A2PGQ96VZRG.jpg)

```
openssl pkcs12 –in client.pfx –nocerts –nodes –out client.key

```

![](https://bbs.pediy.com/upload/attach/202101/777761_VUNGWSSAN43XBM3.jpg)

### **这里需要注意的是，导出 key 的时候需要输入密码，也就是这个地方（如下图）**

![](https://bbs.pediy.com/upload/attach/202101/777761_879NRQHYBNCRMN4.jpg)

因为我这里是没有设置密码的，所以不用输入，直接回车导出；如果设置了证书密码，这里需要输入证书密码才能把 key 导出来。

4、有了 key 剩下的就好办了，将 crt 证书和 key 文件合并成 “.p12” 证书文件，合并的时候记得对证书进行加密（也就是加个证书密码），不加密码 burpsuite 是无法导入的。

合并证书命令如下：

```
openssl pkcs12 -export -inkey client.key -in client.crt -out client.p12

```

![](https://bbs.pediy.com/upload/attach/202101/777761_JHY7VSM5R43W24H.jpg)

5、将证书导入到 burpsuite：

![](https://bbs.pediy.com/upload/attach/202101/777761_ATG4GQKPQ7H6GDR.jpg)

![](https://bbs.pediy.com/upload/attach/202101/777761_78SPUAXPCEXX8KB.jpg)

![](https://bbs.pediy.com/upload/attach/202101/777761_4U99NXWTEQG7SFR.jpg)

6、证书导入成功，并且已启用

![](https://bbs.pediy.com/upload/attach/202101/777761_79SWN98GSN43TRA.jpg)

7、接下来就是见证奇迹的时刻了，为了保险起见先重启一下手机 / 模拟器（最好是彻底关闭然后再打开，我是这样操作的），最后成功抓到该 APP 的数据包

![](https://bbs.pediy.com/upload/attach/202101/777761_RP82QUPHST7YFYE.jpg)

![](https://bbs.pediy.com/upload/attach/202101/777761_9D6YCF48A8JBH2Z.jpg)

以上就是本次突破双向认证成功抓包的全过程，有问题可共同探讨、学习，本人技术也不是很好，可能有很多地方描述的不是很好，有问题的地方还请各位指正。
-----------------------------------------------------------------------

###### 参考链接：

##### [https://www.secpulse.com/archives/117194.html](https://www.secpulse.com/archives/117194.html)

##### [https://www.secpulse.com/archives/54027.html](https://www.secpulse.com/archives/54027.html)

[看雪社区年底排行榜，查查你的排名？](https://www.kanxue.com/rank.htm)

最后于 13 小时前 被丶空心编辑 ，原因：