> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/k75HLvMnh22hY9WMIykrCw)

    前段时间杂事太多, 也没碰到什么好分享的东西就没写文章了, 最近工作中又逆向了个航司, 拿出来分享记录下.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficpVk3npCUJajeRT8icTiaj0D1KAdjwIxOvUqvj4sxBqaaFGW2mlOpF6Pg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficbIFp1Q0qM85lK3XA11GF3gWOPvEjRqibwIfibTH8ThfMia6tbXJAdMC5g/640?wx_fmt=png)

    首先抓包发现关心的内容全是加密了的, 看着像 base64, 去试探性的解码了下, 果然不对. 然后就要找是怎么生成的数据. 这种全是加密的 不能 grep 咋办呀? 

看了下请求头 这个 okhttp3 也太明显了...  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficykg1gQiaib4J0CVYZQq4sD61qO5ug6DtnCxfP5M78Y8c6oLSyvUkQibDw/640?wx_fmt=png)

    那岂不是用 hook 抓包脚本就能抓到了? 我这里用了肉丝姐课上的 hook socket 脚本 

```
hook
java.net.SocketOutputStream 的 write 和 
java.net.SocketInputStream 的 read 
抓http的请求
hook
com.android.org.conscrypt.ConscryptFileDescriptorSocket$SSLOutputStream 的 write 和
com.android.org.conscrypt.ConscryptFileDescriptorSocket$SSLInputStream 的 read
抓https的请求

```

    在其中加上打印调用栈的功能 就变成了追踪请求的脚本了. 去掉耗时会蹦的打印内容的代码. attach 上 app 后发送请求.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficlUIr2k8oqUZhUV9GZIOqcvF6gqJJXqic3oOoNugX1icicM1I5O6ialbtNw/640?wx_fmt=png)

    啪! 很快嗷 调用栈一下子就吐出来了 直接在调用栈里搜索包名相关信息,

业务代码一下子就出现了, 光速定位, easy~ 

    请求代码大概和这两个有关 

    com.csair.mbp.net.b.a.a 和 

    com.csair.mbp.netrequest.net.okhttp.interceptor 下的类 

    估计都是实现 okhttp3 的拦截器的方法来对数据进行加密  

    那就去脱壳呗 看了下这个壳和之前写的文章一样 都是爱加密的, 用 frida_fart dumpclass 拖指定的类就完了. 具体操作在下面文章提过.

> 寄予蓝 y，公众号：搞点逆向 778[App 逆向｜实战 protobuf 加二代壳的 app](https://mp.weixin.qq.com/s?__biz=MzUxMjE3Nzc3Mw==&mid=2247483922&idx=1&sn=1882d193fa7d457a88e54e6e16c5ac88&chksm=f9692110ce1ea8068b09adbfeb27be821a01be3879681b81bfa0b6893d4a2ae3e1b044c070af&token=478623711&lang=zh_CN#rd)

    在脱下来的 dex 中根据调用栈 找了找 很快就找到 URL 中的参数来源  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficyjJZgbvbeayibSejcu6J5onJDBMoLVF1yNysibaMzpvUC5MH8VxibEV7w/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficID953ZgPGZe9B53OfNtg6SGcibgbd8VWl8NAydsEiaEL3cmpwv6G9aZw/640?wx_fmt=png)

    根据调用栈 找到一个很可疑加密 body 的地方

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficcu44ueuO2Y5dRvCLHWibySIjJTicfgTodIPLfjvDRr7nG4kSV9LbSsSw/640?wx_fmt=png)

    进这个方法一看是一个接口类

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4fic3qZhA7WbNg6fgjtTAtDWz85AejwydqOYSwnVsMCeQBWpIvXKw0icuyg/640?wx_fmt=png)

    直接上脚本, 暴力枚举接口! -o 保存到文件 来搜索  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4fic8rA1WOiaJFesib5sNibGKQq3yT33sWG7ibDIwx3Fv8YeJ66aC30FHAksQQ/640?wx_fmt=png)

    出来了好几个实现类 去代码中找下 然后 hook 验证一下 发现是这个实现类

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficgvLhc7aiakaXt9wOZJKjjzbsLEWWiaj9T0HnVZZvazSr5RlhWCuBQRzA/640?wx_fmt=png)

    如果 objection 上不了 先用 frida spawn 上 再用 objection 上  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficgKtxuJCCUyfQ0469lP0FAKrCL6BrbD0ldhjlKy25DGPP63icN6pCOAw/640?wx_fmt=png)

    然后 hook 一下入参出参 发现就是走的这里加密 然后发现响应解密也是走的这里.... 事半功倍  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficYzSzhCWZvzKoibJ3tXmorrDpRwtrfeGbJhmTJ8FMgCy15jJaIicJ8C1A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficBsR1ErcpkV5V0rudnibaaN6aiaNnL7nDdfj9KQx9wCWZAve3HMecuVTw/640?wx_fmt=png)

    往下一追 就到了 natvie 了 看着名字 估计就是爱加密的加密, 先放一放, 再看看其他的.  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4fic9zdh24g4GFJ6PUdJKqe9fZ5ONiaGgHmczFaes0NopnBsw4YvhmAgic3Q/640?wx_fmt=png)

    还有个请求头的关键参数 wtoken 和 body   

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficpicaA9jaNfomhYyT8swLmYUiaVOFgwaKdicUticCUzo8JibCpT2EvlGw40A/640?wx_fmt=png)

    body 则是对 请求的 body 进行了 md5 的计算, 因为请求头中带有

accept-encoding:gzip

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficl8lF6LO0EwCqNSKrvmXTfO6srFGynCLs55eeRI1lMQ3LtsjklibQeyg/640?wx_fmt=png)

    所以这里的 body 是经过 gzip 压缩后再 md5 的. 

    然后 wtoken 跟进去一看也是个接口类 同上面方法找到实现类后找到方法....  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4fic4T7nZVPslyLGuJfcRaygFHETtWkV96yeU5wfnupLf8O91PaVOg56Tw/640?wx_fmt=png)

    avmpSign 还是 com.alibaba.wireless.security 的 这名字就足够吓人了....

    对不起, 我走主动调用! 因为需要应用到生产环境 然后刚学了 xposed 就他来开刀! 谁叫你不检测 xposed.

    根据课上所学, 因为是带壳的 app 不能直接获取到真实 app 的 classloader, 如何获取到真正的 classloader? 所以参考 fart 源码在 android.app.ActivityThread 的 performLaunchActivity 中开启 fart 线程进行脱壳的代码, 通过 getClassloader 拿到真正的 classloader

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficoYwoZfpVWz2zibG7SVFdn2BLy4pUU0FUdZ8X24tU5Z7vsiccFSy1icMicA/640?wx_fmt=png)

    跟进去看

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficvficn7oYicvnQ9wqB7dOynCl2rd6dVZaXUXF9jxqlUWIk0zZuMIsdCKg/640?wx_fmt=png)

    只要拿到 Application 这个类的对象, 然后调用 getClassLoader() 就能获取到了.

    使用 wallbreaker 观察 android.app.ActivityThread 这个对象

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficuib6a0O9icQiaicjmDXAcUSSuemQk61Qx3bhOH5HMTs1AD5skVJ3piaib1RA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficPSvcX4f3XVfKicqX2qKd1qsia5oWQKLa77Y1hciba5CpvTP5AiaXcTvT6g/640?wx_fmt=png)

    发现正好存在这个 Application 这个类的对象 那我们直接 hook performLaunchActivity 然后获取 mInitialApplication 的动态域, 然后调用 getClassLoader 方法即可拿到真正 app 的 classloader. 然后就按正常流程获取类调用就好了.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficyEWlXblfULa0AOoL6UGibiaiaj7JyuytiaicLh3lTTJqUZkTfgT5foNxiajw/640?wx_fmt=png)

    下面再起一个 nanohttpd 开服务 稳的一批!  

    写成 python 的时候 body 那个地方有坑, 需要自行对他进行 gzip 编码发送才能成功. 其中 mtime 和时间戳有关, 若不指定为 0, 每次压缩的 byte 都是不一样 (一开始和 Java gzip 的结果对比坑死我了! 每次都不一样, 然后压缩等级 = 7 的时候和 Java gzip 后的结果最接近 好像就第十个 byte 不一样)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficWFdOq1wm04tssDqquvteDbc87Y0fI3CHnicKiaks5VsmWx0aAVgH90Mw/640?wx_fmt=png)

    而且 data 需要发送 byte 类型 (gzip 编码后就是 byte 类型). 一开始用 request 测试 需要把 data 转成 bytearray(data) 才能 post 不报错, 然后果断换成 httpx 正好这请求也是 http2 的 然后 httpx 更方便 直接支持 post byte 类型 和 http2.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSrsfQutLECDxJkmfibzR4ficp6cBjExdCh8GS3Xnq9wicD500a3KnE6tqva9Tib3xEGicYia2zyibxCrmVA/640?wx_fmt=png)

    很快很稳很香~