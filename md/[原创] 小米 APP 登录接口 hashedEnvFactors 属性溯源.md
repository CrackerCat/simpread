> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266338.htm)

> [原创] 小米 APP 登录接口 hashedEnvFactors 属性溯源

0x00 前言
-------

继上一次小米 APP 登录接口遇到的几个参数加密算法分析后， 发现其中 env 中是通过读取一串叫 hashedEnvFactors 的数组字符串进去拼接加密的，这次我们主要讲讲是如何找出 hashedEnvFactors 的赋值过程。 [原帖](https://bbs.pediy.com/thread-266334.htm)

#01 溯源
------

我们回到之前的 loginByPassword 方法，也就是登录时触发的方法，我们知道，它只有一个传参 PasswordLoginParams ，直接跟进查看这个类。  
![](https://bbs.pediy.com/upload/attach/202103/885524_BV5KB6CBFBBYAZN.png)  
我们可以直接右键这个属性来看看是在哪里赋值的，因为我当初学习 JAVA 的时候书中曾就讲过，最好不要直接给类里面的属性直接用 = 号赋值，都是直接在新建一个方法来赋值，所以这也跟了我们逆向带来的便捷。。。

```
public Builder setHashedEnvFactors(String[] strArr) {
          this.hashedEnvFactors = strArr;
          return this;
      }

```

我们看到这个类下面有一个 set 方法了，然后在进行调用用例看看是哪个地方执行了该方法。  
![](https://bbs.pediy.com/upload/attach/202103/885524_35N5EUWKDRD7VA9.png)  
我们看到有好几个，其中第一个的方法名称跟我们的登录接口是一样的，就先点进去看看。

```
static AccountInfo loginByPassword(String str, String str2, String str3, String str4, String str5, String str6, MetaLoginData metaLoginData, boolean z, String[] strArr, PassportCATokenManager passportCATokenManager, boolean z2) throws InvalidResponseException, InvalidCredentialException, InvalidUserNameException, NeedVerificationException, NeedCaptchaException, IOException, AccessDeniedException, AuthenticationFailureException, NeedNotificationException, PassportCAException {
       return loginByPassword(new PasswordLoginParams.Builder().setUserId(str).setPassword(str4).setDeviceId(str3).setCaptCode(str5).setCaptIck(str6).setServiceId(str2).setMetaLoginData(metaLoginData).setNeedProcessNotification(z).setIsReturnStsUrl(z2).setHashedEnvFactors(strArr).build());
   }

```

它这里就是一行语句的方法。而且参数贼长，从单个字符串传参变成了封装成一个 PasswordLoginParams 类进行传参。。那么我们看到其中 strArr 就是我们要找的，继续看看是谁调用了这个方法。  
![](https://bbs.pediy.com/upload/attach/202103/885524_NAUHQGEAGPQHQBE.png)  
依旧点击第一个进去  
![](https://bbs.pediy.com/upload/attach/202103/885524_X29KHA9253VN95F.png)  
还是熟悉的配方熟悉的味道，继续往上跟。。  
![](https://bbs.pediy.com/upload/attach/202103/885524_7THAW28AWYVG6CC.png)  
这里注意，我们不要选第一行了，因为我们要的参数是最后一个，而它传的是 NULL，肯定不是我们要的，我们可以点第 3 个看看  
![](https://bbs.pediy.com/upload/attach/202103/885524_K5SB74VHNV6AC7C.png)  
可以看到是调用了 getEnvInfoArray() 获取到的，继续跟进。  
![](https://bbs.pediy.com/upload/attach/202103/885524_CC6NDC6FFPEADF5.png)  
我看可以看是通过 getEnvInfoArray 返回的，继续跟进。  
![](https://bbs.pediy.com/upload/attach/202103/885524_X8RMEWW4T6AD8UJ.png)  
我们可以看到通过调用 getAllLinkedEnvInfos 方法获得 List 然后枚举出来插入到 arrayList 里面的。  
并且我们看到一个小细节，之前抓出来有些数组是空字符串，然而这里正好看到它有一个如果是 null 则赋值空字符串的判断语句，更加证实了该方法是我们需要的。  
我们继续跟入 getAllLinkedEnvInfos  
![](https://bbs.pediy.com/upload/attach/202103/885524_JK6M5NKHTQM5M8K.png)  
跟到这里，基本可以宣告结束了, 这里他就是获取了手机的一些基本设备信息，并且我们数了一下他 add 的数量跟 hashedEnvFactors 正好一样。。（我也是够闲的）

0x02 总结
-------

总的来说这个字符串还是挺好跟进去的，只要思路比较清晰就行了，写的也不是非常野，最后我们只需要知道把他的 hash 和 base64 方法拿出来自己就可以定义这个字符串了。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

[#逆向分析](forum-161-1-118.htm)