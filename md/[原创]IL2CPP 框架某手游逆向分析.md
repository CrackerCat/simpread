> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272248.htm)

> [原创]IL2CPP 框架某手游逆向分析

本文内容仅供学习。  
可以百度下载阿拉德之怒手游 app。  
![](https://bbs.pediy.com/upload/attach/202204/856105_PVEH35959Y2QU6K.png)

1.  解开 APK 包，可以看出该手游是 IL2CPP 框架的游戏，那么在 apk 中肯定还有一个 global-metadata.dat 文件（除非加密）。  
    ![](https://bbs.pediy.com/upload/attach/202204/856105_H3GCUPTV5P9V6HK.png)
2.  将 global-metadata.dat 和 libil2cpp.so 文件拷贝到同一个文件夹下。
3.  使用 Il2CppDumper 将 global-metadata.dat 和 libil2cpp.so 进行解析，下载网站：https://github.com/Perfare/Il2CppDumper/
4.  解析完成。  
    ![](https://bbs.pediy.com/upload/attach/202204/856105_JB38Y69Y78QWNUU.png)
5.  生成几个文件，现在分析 DummyDll 文件夹下的文件  
    ![](https://bbs.pediy.com/upload/attach/202204/856105_WJC73TW79HDMMR7.png)  
    ![](https://bbs.pediy.com/upload/attach/202204/856105_4WDQMWHKDQ9SFZD.png)
6.  将 Assembly-CSharp.dll 脱到 dnSpy（可以百度下载）中后就可以正常分析了，修改 GetHP 函数猜测可能可以实现无敌。  
    ![](https://bbs.pediy.com/upload/attach/202204/856105_TACJ6R7A37QVF6G.png)
7.  一些常用游戏关键词。

> 游戏搜索方发名：  
> onResult，onchinabilling，resulton，Paycenter，Callback  
> 联通游戏搜索方发名：  
> OnPayResult，PyaResulton，Activity，result，callback  
> 电信爱游戏搜索方发名：  
> paySuccess 成功，payCancel 取消，payFailed 失败  
> 移动 mm 搜索方发名：  
> onBillingFinish，Billing，CallBack  
> 支付宝和银行卡方法名：handlemessage  
> 支付宝搜索字符串 9000 360  
> 支付  
> onfinishedon，Activityresult  
> 发送短信限: android.permission.SEND_SMS  
> 发送短信锁定支付式关建字符串: CHINA_TELECOM  
> 中国电信 46003，46005，46011  
> CHINA_MOBILE  
> 中国移动  
> 46020，46000，46002，46007  
> CHINA_UNICOM  
> 中国联通 46001，46006，NOT_DEFINE 未定义  
> 英雄、玩家 (Hero、player)  
> 怪物、敌人 (monster、Enemies、enemy)  
> 初始化 (init)  
> 力量 (power、str、strength)  
> 智力 (int、Intelligence)  
> 运气 (luk、luck)  
> 敏捷 (AGI、agile)  
> 魔法 (magic、mag)  
> 金币 (gold、coin)  
> 钱，钞票 (Money)  
> 现金 (cash)  
> 钻石、宝石 (Gem、diamond)  
> 血量 (health、life、HP、Max hp、blood)  
> 蓝 (mp、sp、Power)  
> 攻击 (attack(atk)、fight、hit、damage)  
> 防御 (defence、def、defense)  
> 护甲 (Armor)  
> 物理 (physic、phy)  
> 暴击 (Crit、cri、crt)  
> 闪避 (Dodge)  
> 范围 (range)  
> 速度、频率 (speed)  
> 改路 (Rate)  
> 恢复 (Recover)  
> 取 (get)  
> 置 (set)  
> 支付 (bill、billing、pay、purchase)  
> 成功 (success)  
> 失败 (fail)  
> 取消 (cancel)  
> 分数 (Score)  
> 死亡 (Dead)  
> 伤害 (hurt)  
> 体质 (vital、vit、stamina)  
> 技能伤害 (skilldamage)  
> 换弹速度 (initalize)

 

8. 用 IDA 打开 libil2cpp.so，要花费不少时间，挺大的程序，IDA 解析完成后，里面的函数名都是不太友好的，伪 C 代码也很难看明白，所以下一步就是把符号表（函数名）恢复。  
![](https://bbs.pediy.com/upload/attach/202204/856105_RFW55QRZA42CXTG.png)  
9. 使用 Il2CppDumper 下生成了一个 dump.cs 文件，该文件里面有很多符号表和地址，大概思路是将符号表和对应的地址，通过 IDA 自己编写的插件进行恢复（做 IOT 漏洞挖掘时的思路）。由于同名很多，所以最好将类名加上，主要防止重名。

```
1. 用python将GetHashCode和RVA 一一对应导出。
2. IDA恢复符号表（本人使用的是IDA7.0，IDA7.5貌似不能正常使用以下恢复函数）。

```

```
# 主要恢复函数
idc.MakeName(func_addr, func_name)
idc.MakeCode(func_addr)
idc.MakeFunction(func_addr)

```

![](https://bbs.pediy.com/upload/attach/202204/856105_ARVC6F97JTEPXCY.png)  
10. 恢复完成后，看起来就舒服多了（有的函数没有正常恢复，应该是 dump.cs 文件中没有对应的符号表，不过问题不大，主要的函数已经恢复正常）。  
![](https://bbs.pediy.com/upload/attach/202204/856105_F42HK4MWTKTM753.png)  
11. 开始使用 frida 进行 hook 对应的函数地址。

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#逆向分析](forum-161-1-118.htm)