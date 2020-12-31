> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484148&idx=1&sn=269ddfa70f832a9ef01df7402bb0e9d4&chksm=cebb47baf9ccceac5147bf99428ff0ebf1d37696268f0e5fc3be0417f7c0895ded50230e343c&scene=21#wechat_redirect)

**使用 IDA 进行动态调试 Android so，有两种方式进行调试****，****如下所示:**

  (1). 一种是调试启动方式，调试启动可以调试 jni_onload ，init_array 处的代码，可以在较早的时机得到调试权限，一般反调试会在较早的时候进行启动。

  (2). 另一种则是附加调试，附加调试是在 App 已经运行起来的时候进行附加调试，这两种方式与 Windows 上的调试启动和附加调试概念很相似。

**实践环境: Windows 10 + IDA Pro 7.0 + Nexus5(Android 7.1.2)**

**第一种****:** **调试启动**

**1. 以调试的方式启动程序（可以在 jni_onload、init_array 处下断点）**

adb shell am start -D -n com.***.***/.MainActivity

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2Pu1EGm3EJicrl5iaDwKmOFG8PnADoxywqqb1SbegLVwAZfBZmCnv9OSkfg/640?wx_fmt=png)

**2. 在 IDA 的主目录 dbgsrv 目录下，找到 android_server，拷贝到手机的** data/local/tmp / 目录下

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuB8rk7Fpic4l7Fp7KgGibfZnpMIvxBOItLv2ib6ILNp9glRl9zYtHEjic5A/640?wx_fmt=png)

adb push （android_server 所在目录） /data/local/tmp

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PubYeoySmmPsBiaicicQ697csMnn29Pia1SNAdXYKZxU2nmAfzrgmicvrxyjQ/640?wx_fmt=png)

**3. 使用 chmod 修改 android_server 程序的执行权限并执行**

adb shell chmod 755 /data/local/tmp/android_server 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuGJQWUqopic09HHb6DDZPxnEKmGvQqbCicPprAibXtfEgibUACbmGcrgrlw/640?wx_fmt=png)

adb shell

su

cd data/local/tmp

./android_server

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PujI511pmzuYlo8fScFat3xiaS3Q7rIqIJ5icBbt2Bdtz03HhuSGvnHcxA/640?wx_fmt=png) 

**4. 新开一个 cmd 窗口使用 forward 程序进行端口转发**

adb forward tcp:23946 tcp:23946

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuoexUCqM8rOVZbQKvzQtVQDcEfeW6A8svLuDMiaETVibD2eysH9AoM8oQ/640?wx_fmt=png)

**5. 打开方法调试**

打开 IDA，选择菜单 Debugger -> Attach -> Remote ARM Linux/Android debugger

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuaWazS0gScrERSqpT8PtnPFvfhucEGqTZPibkh1dx6Sr6r4ic2nyImyqA/640?wx_fmt=png)

**6.Hostname 填写 127.0.0.1 或者 locahost**

Port 端口自动填 23946 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuicbDdnbTIicicZCb1IW0PnNmgq4hENFrhE2hF55nSJVibkwmQWQzycbBFQ/640?wx_fmt=png)

**7. 选择要进行调试的 App**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuPkNNRYsbJOJ28Z93XXdTeRQsl4S1ozPO5mVU7nVtARuxnojLCgMytQ/640?wx_fmt=png)

点击 OK 按钮则进入调试界面，注意，这时候还不能进行动态调试

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuYppIGRlSVu740sy9icrPeaWJ69axoygSlxq4nwZNUSeBLFicLnsqYic1Q/640?wx_fmt=png)

8. 点击菜单 Debugger->Debugger Opitions 在弹出的 Debugger setup 窗口的 Events 中选择 Suspend on Process entry 和 Suspend on thread start/exit 以及 Suspend on library load/unload，再点击 OK 退出。通过此操作可以设置程序在创建新线程和加载 so 时自动中断

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PukYuFAYc5iblqHAOSibic3B3BU2axKKSyJO8MpAyObbHKtCz1yngSXDDtQ/640?wx_fmt=png)

通过 DDMS 获取相应进程的端口号，然后使用 jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=XXXX（DDMS 查询到的端口号，一般 ddms 进行转发后的端口都是 8700） 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuG14SZZC8ib5NaM3Yqgvm42qVUxj03VcG2o5A9ziahRiaFymo6FiaGqZCSA/640?wx_fmt=png)

如果成功执行 jdb -connect DDMS 上虫子图标会变成绿色 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuLzVXfI4icJxa3fE8keJSVcGRzlfLuiahf39RAvxq9M9FukVGjSiajAUNg/640?wx_fmt=png)

连接成功后，在 IDA 按 F9 后手机上的 “waiting for debugger" 提示会自动消失，这个时候应该已经断在新线程，或者加载 so 处了。

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2Pu9QBDNtB1PNOjC5Oadniaulu7iabZ8QrzqRONaezTGMPasOibSXZ0QPFsw/640?wx_fmt=png)

9. 现在就可以在 IDA 中按下快捷键 CTRL + S 来查看要调试的 so 是否已经加载了，如果没有就按 F9，直到加载了为止；如果已经有了，就记下该 so 的 start 位置，然后另开一个 ida 分析. so 库，找到 JNI_ONLOAD 的偏移地址，那么该 JNI_OnLoad 函数在进程中的真实地址就是 so.start + JNI_OnLoad_Offset

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuibNibQ4wMwgAohicMlSIr88OMF3u1xBavyv4icrtgw91iaJhOqVTkgbplJQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2Pu4FyjlVsZt1OowTtQCrJA8kc2FMAzcich4gsb6NIeuJiaaohARqRtYaqA/640?wx_fmt=png)

0x753030000 + 0x1B9C = 0x75304B9C（JNI_OnLoad 在 App 进程中的地址）

这里需要说明的是: 有可能在快捷键 CTRL + S 跳出的窗口中有两个同名的 so，我们应当选择权限为 RX 的这个，RX 一般是代码段，RW 一般是数据段。

10. 得到真实地址后，在 IDA 中按下快捷键 G 跳转到这个地址，然后按下快捷键 F2 就完成在 JNI_OnLoad 函数入口处下断点了。如果想直接在 native 函数下断点，那么另开一个 IDA 进行静态分析查看函数地址，然后转换成为动态加载起来的地址，接着按快捷键 G 跳转到该地址下断点即可

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2Putzt5ic65laWm9ickiaMS2nvUYTaiakswLpqCwia3frOiancnuXYRhLibC2zTA/640?wx_fmt=png)

**11. 再次按下 F9，则在 JNI_OnLoad 断了下来，接着就可以进行动态调试逆向分析功能**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2Puq54iaaK1wgicjvxBgicJwiagINEA6AF0bMibryPDsmjuKNVVMHiaReT55oBA/640?wx_fmt=png)

**第二种****:** **附加方式调试 so**

**1. 拷贝 android_server**

adb push （android_server 所在目录） /data/local/tmp

**2. 修改执行权限并执行**

adb shell chmod 755 /data/local/tmp/android_server

adb shell

su

cd data/local/tmp

./android_server

**3. 转发端口**

adb forward tcp:23946 tcp:23946

**4. 启动 android_server**

发现一个小问题，就是退出调试以后再次调试，android_server 没有 close，按道理是可以再一次连接的，但是这里则是无法进行再次连接调试，所以需要重新启动一下

android_server

如下:(14061 是 pid)

root@android:/ # ps | grep android_server

root 14061 13574 11180 9504 ffffffff 40183da0 S /data/local/tmp/android_server 

root@android:/ # kill -s 9 14061 

**5. 打开 IDA，选择菜单 Debugger -> Attach -> Remote ARM Linux/Android debugger**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuaWazS0gScrERSqpT8PtnPFvfhucEGqTZPibkh1dx6Sr6r4ic2nyImyqA/640?wx_fmt=png)

Hostname 填写 127.0.0.1 或者 locahost

Port 端口自动填 23946

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuicbDdnbTIicicZCb1IW0PnNmgq4hENFrhE2hF55nSJVibkwmQWQzycbBFQ/640?wx_fmt=png)

选择要进行调试的 App

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2Puew0W1FPAPucoTyQnxWSCgKJGV8yzwIWUDHuKjbFePo7jL7pDYl0JNQ/640?wx_fmt=png)

得到进程中的地址后，在 IDA 中按下快捷键 G 跳转到这个地址，然后按下快捷键 F2 就完成在 JNI_OnLoad 函数入口处下断点了。如果想直接在 native 函数下断点，那么另开一个 IDA 进行静态分析查看函数地址，然后转换成为动态加载起来的地址，接着按下快捷键 G 跳转到该地址下断点即可 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuAu1tgyOFgFXMSF2t9HRjcNiaibmD7qaWW8XKf8Y10Oex3RdnNtWk0bKA/640?wx_fmt=png)

运行程序指定功能，触发断点，接着就可以到达该函数进行动态调试逆向分析功能

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuMNY96TxJOj8MheppYiaH5jyo26DmrHEvgiaXtNeJ84icZ4wPRH5F5egfw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF01OoKCbZEMW0ZPL0WrX2PuO2EicuYzyTq65NtwRn7z1boaYqCsq3EKxmhTdaJZ2KkolzibuNiciaEvkA/640?wx_fmt=jpeg)

欢迎转发关注公众号