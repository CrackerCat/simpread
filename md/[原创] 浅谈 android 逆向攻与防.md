> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271311.htm)

> [原创] 浅谈 android 逆向攻与防

android 逆向攻与防
=============

常规 App 逆向反编与二次打包过程
------------------

### 反编译工具：

> *   apkTool
> *   dex2Jar，jd-gui
> *   jadx、Jeb、AndroidKiller
> *   其他三方反编译工具

 

### 逆向分析修改工具：

> *   Xposed
> *   Frida
> *   FakeAndroid
> *   反射大师、FDex
> *   VMOS、IDA Pro
> *   Fiddler、Http Canary、Charels

 

逆向的最终目的是拿到 dex 修改并进行二次打包  
举例（单机游戏或工具类应用 对比支付类应用）

```
apktool d [apk路径] #反编译apk
apktool b [apk文件夹] [新的apk存储位置] #编译apk

```

### 对抗反编译

> 通过三方厂商加固（腾讯御安全、腾讯乐固、奇虎 360、爱加密、梆梆加固）  
> 给 App 包一层壳，将业务代码打包进 so 文件  
> [腾讯御安全](https://console.cloud.tencent.com/ms/index#)

> 手动或借助三方工具砸壳，Dump 出原本 dex，在 App 启动进程加载 hook.so，装载修改 dex 覆盖原 dex，从而达到修改目的

### 对抗修改

> 增加代码签名校验，判断二次打包签名是否与原签名 md5 一致，不一致禁止使用
> 
> 开启混淆，添加自定义混淆字典，添加合并包名

> 直接删除签名校验代码
> 
> ### 对抗抓包
> 
> 使用 https 进行 SSL 证书校验  
> 代码开启代理检测，检测使用代理禁止请求  
> 自定义 DNS 解析  
> ![](https://i.niupic.com/images/2022/01/12/9TcY.png)

### [](#接口安全处理（某电商）)接口安全处理（某电商）

> 最根本还是得从根源来提升逆向门槛，也就是接口安全  
> 即核心业务一定要进行服务器校验甚至二次校验，同时接口设计要尽可能防止用户篡改参数，以及限制单用户短时请求频率  
> sign  
> 旧版本——>so  
> ![](https://gitee.com/yue7/ZxlPub/raw/master/img/%E4%BA%AC%E4%B8%9C%E8%80%81%E7%89%88%E6%9C%AC%E7%AD%BE%E5%90%8D11.png)  
> st=1641978118469&sign=9a67f8649bdd22cb8ec7828cf9fb73b5&sv=111  
> 根据 funcationId 和 body 计算出字符串 st=1641978118469&sign=9a67f8649bdd22cb8ec7828cf9fb73b5&sv=111  
> 然后拼接到参数中  
> 绑定时间戳防篡改  
> sandHook\virturlHook  
> 新版本  
> ![](https://gitee.com/yue7/ZxlPub/raw/master/img/%E4%BA%AC%E4%B8%9C%E6%96%B0%E7%89%88%E6%9C%AC%E7%AD%BE%E5%90%8D.png)  
> 大部分核心参数会先进行一次加密，然后所有参数排序遍历拼接 value 通过 Sha256 加密为 64 位字符串  
> ![](https://i.niupic.com/images/2022/01/12/9TcW.png)

### 总结

> 没有最安全的程序，也没有最强大的逆向  
> 尽量少写本地全局重要变量，如 IsVip 等  
> 尽量数据从服务器获取  
> 尽量客户端、服务器双向验证数据可靠性

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#逆向分析](forum-161-1-118.htm)