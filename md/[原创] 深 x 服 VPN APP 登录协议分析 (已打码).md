> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269733.htm)

> [原创] 深 x 服 VPN APP 登录协议分析 (已打码)

一、概述  
最近逆深信服的 Easyconnect.apk，无壳但有混淆，so 层使用自编译的 openssl 再走 libc 为后续与服务器通信（图片已厚码脱敏）。有什么错误还请师傅指出。感谢看雪高研。

 

二、连接服务器请求分析  
连接服务器时，会发送先后两个请求，下面先看第一个：  
先使用脚本 hook onclik 点击事件，得到所实现的 ConnectActivity：  
[WatchEvent] onClick: com.sangfor.vpn.client.phone.ConnectActivity  
![](https://bbs.pediy.com/upload/attach/202110/718877_7PEX8QDG6H5TEZD.png)  
跟入 o():  
![](https://bbs.pediy.com/upload/attach/202110/718877_YSM7XV8AD95TWKU.png)  
o() 中调用 showDialog(10) 显示对话框后，再新建 cb 对象，并新建异步任务：  
![](https://bbs.pediy.com/upload/attach/202110/718877_TAV88BUB8GFHU6B.png)  
任务中调用了 cb 对象的 a():  
![](https://bbs.pediy.com/upload/attach/202110/718877_AEPPTF3Y5P668DY.png)  
继续跟进 b() 方法，其中 arg2 为 url：  
![](https://bbs.pediy.com/upload/attach/202110/718877_6B6NGAET573Q9RU.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_PAQGWN9PXQAN7U4.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_FMVDB5BDQ5275Y5.png)  
url 经过判断后，进入 ConnectActivity 的 a() 方法，在该方法中设置好相应参数后调用 l() 方法：  
![](https://bbs.pediy.com/upload/attach/202110/718877_MQUV4R7RWMPEMAC.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_F6U8SUVM29F39KW.png)  
L() 方法中拼接 url 新建异步任务：  
![](https://bbs.pediy.com/upload/attach/202110/718877_A83VGTATJRDZ6CR.png)  
其中在任务中 doInBackground() 调用了 ConnectActivity.a() 方法：  
![](https://bbs.pediy.com/upload/attach/202110/718877_87TD89B6AHHN5PA.png)  
在 A() 中主要调用了 HttpConnect.requestStringWithURL():  
![](https://bbs.pediy.com/upload/attach/202110/718877_M8QXG7BAR286AZF.png)  
继续跟入最终调用了 httprequest() 的 native 方法:  
![](https://bbs.pediy.com/upload/attach/202110/718877_THFGN2794WK2JEM.png)  
根据后续的分析 arg1 为 url，arg2 为 TWFID 等 cookie 信息，agr3 为传输的内容，arg4 为是否为 post 提交，arg5 后续加密验证方式，arg6 为 http 版本  
![](https://bbs.pediy.com/upload/attach/202110/718877_2AQQ663D9X4A4Y7.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_9YQA84N8T4YDJSC.png)

 

接着在 onPostExecute 发送第二个请求给服务器：  
![](https://bbs.pediy.com/upload/attach/202110/718877_N2MXU873QTK9VJA.png)  
跟入其 a() 方法，同样调用了 ConnectActivity.a() 方法：  
![](https://bbs.pediy.com/upload/attach/202110/718877_U8SDAZ6739BAP2K.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_WD9ER8VKA5KU4P8.png)  
进行跟入 b()，可见在函数新建了线程，跟入其线程函数：  
![](https://bbs.pediy.com/upload/attach/202110/718877_FDD3SGWCE5MK4NN.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_WEKDVWFF2ZCGZG2.png)  
注意这里 v2.put(“mobileid,u.f().b()”) 把 mobileid 一起发给服务器 https://ip:port/por/login_auth.csp?dev=android-phone&language=zh_CN  
![](https://bbs.pediy.com/upload/attach/202110/718877_D9Q4NHGFB7V43WT.png)  
最后调用 httpRequest() 发送给服务器的数据包，：  
![](https://bbs.pediy.com/upload/attach/202110/718877_W9PJHYUQBQA5C66.png)

 

在发第二个请求包时，会先获取到内存中的 mobileid，V2 为 hashmap，在调用 httpRequest 前压入了值，跟入 b():：  
![](https://bbs.pediy.com/upload/attach/202110/718877_PUAWWUMCVD3YE4T.png)  
B() 返回 f 的值，查找引用：  
![](https://bbs.pediy.com/upload/attach/202110/718877_7NNDEKV4QBXD4DU.png)  
可见 f 为 nDe ![](https://bbs.pediy.com/upload/attach/202110/718877_NH6YSYB7JCUPHNM.png)precatedEncryptMobileId() 调用后的结果返回给 a() 执行后的值  
![](https://bbs.pediy.com/upload/attach/202110/718877_CVPCZMG7UFCFFAV.png)  
跟入 A()，可见函数主要对传入参数进行 md5：  
![](https://bbs.pediy.com/upload/attach/202110/718877_8MPTGFUJNCA3Q8P.png)  
故我们 hook 住 nDeprecatedEncryptMobileId 函数得到其 mobileid 和返回值：  
("27b3d55fdc6e0c60")  
"P\u0004\u0006\u0006QSQ\u0005R\u0006\u0006\u0006\u0006S\u0004\u0007"  
![](https://bbs.pediy.com/upload/attach/202110/718877_9ER54MG45GYDUJN.png)

 

我们回头看看 mobileid 是如何产生的，可见是先从 map 中取出：  
![](https://bbs.pediy.com/upload/attach/202110/718877_ZK43QP7Y4RE624N.png)  
查找引用找到相关赋值的地方，跟入 c()：  
![](https://bbs.pediy.com/upload/attach/202110/718877_33K5HBFSQP9M87W.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_UW5MUBCQVAB2GZW.png)  
找到其 this.g 赋值，可见先调用了 e() 进行判断是否有 telephony 模块：  
![](https://bbs.pediy.com/upload/attach/202110/718877_5KZPG6ANG4YSQXQ.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_QD9E4TZGET48AJ9.png)  
如果有则调用 getDeviceId():  
![](https://bbs.pediy.com/upload/attach/202110/718877_P37SAVYZQ3M7FSJ.png)

 

三、登录请求分析  
接着就是登录请求的数据包，我们通过上面找到发包的函数，先进行 hook：  
注意第二个数据包的响应包含 <TwfID>xxxxxxxxxxx</TwfID>，该值会先保存在 cookie 中在登录时一同发送：  
![](https://bbs.pediy.com/upload/attach/202110/718877_P846NVQZWB3WTS7.png)

 

Ver 为当前软件版本  
mobileid 是 md5 后的值，固定的  
TWFID 是第二个请求服务器后返回的值，后续保存在 cooike 中。  
MOBILETWFID 与 TWFID 一样  
svpn_password 用户名  
svpn_name 密码

 

我们跟入函数所在 so，经观察 so 中自编译 openssl 库，但符号还在：  
![](https://bbs.pediy.com/upload/attach/202110/718877_59MCH6A4ZUV77XX.png)  
直接上脚本 hook libc，因为 so 中自编译 openssl 库，直接调用 libc 的 write 和 read，不走 libssl 的 ssl_write/bio_wriet 和 ssl_read/bio_read，所以 hook libc 只能捉到对应的密文：  
![](https://bbs.pediy.com/upload/attach/202110/718877_ADJC9GNGBRH3989.png)

 

我们可以 hook libhttps.so 的 SSL_write() 和 SSL_read() 即可得到 SSL 处理前的明文：  
这是连接服务是所发送的第一个请求及其响应的数据：  
![](https://bbs.pediy.com/upload/attach/202110/718877_PS3RFB9YNFP6S69.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_GYCZS2Y398DHXR4.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_PEM2Z6ZHGHD8QY4.png)  
这是连接服务是所发送的第二个请求及其响应的数据：  
![](https://bbs.pediy.com/upload/attach/202110/718877_E566A25NCPHQNKJ.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_WRNDZZHBFZ3NP85.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_KSK88WDB4X6VW4S.png)  
这是用户登录时所发送的用户名及数据给服务器的请求及其响应：  
![](https://bbs.pediy.com/upload/attach/202110/718877_QEVWGE3CMPSBRK9.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_W9WKAT6MMUSNN8X.png)  
![](https://bbs.pediy.com/upload/attach/202110/718877_VT94Y3J5F37NRGJ.png)

 

最后我们可以构造 mobileid 后请求服务器 https://ip:port/por/login_auth.csp 得到后续 TWFID，使用 frida 主动调用 httpRequest 并传入 TWFID 访问 https://ip:port/por/login_psw.csp 进行爆破。

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 6 小时前 被快乐鸡哥编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)