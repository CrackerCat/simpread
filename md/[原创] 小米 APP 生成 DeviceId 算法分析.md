> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266342.htm)

0x00 前言
-------

前面发过 2 篇分析小米 APP 的文章，分别是 [env、envKey、hash 算法分析](https://bbs.pediy.com/thread-266334.htm) 以及 [hashedEnvFactors 属性溯源](https://bbs.pediy.com/thread-266338.htm) 这次又来拿它开刀啦。

0x01 介绍
-------

这次是我们发现数据包里都会传一个 cookie，里面带有一个属性是 deviceId，按通常理解，cookie 都是由服务器所产生作为临时保存的缓存机制。 但是经过一番搜索并未发现 deviceId 是从服务器返回的。但也可能是它返回的是一串其它字符串，然后在 app 内经过加密处理在发的呢？ 这就需要我们去分析了。  
![](https://bbs.pediy.com/upload/attach/202103/885524_X6RV9ADVRW2AKC2.png)

0x02 定位技巧
---------

这里没有选择通过直接搜索 deviceId 字符串进行搜索（其实我尝试过了~），因为搜索出来的结果过多，非常难精准定位到该地方，所以我们直接按照之前发布的文章，先重新回到登录触发的接口。

```
public static AccountInfo loginByPassword(PasswordLoginParams passwordLoginParams) throws InvalidResponseException, InvalidCredentialException, InvalidUserNameException, NeedVerificationException, NeedCaptchaException, IOException, AccessDeniedException, AuthenticationFailureException, NeedNotificationException {
     if (passwordLoginParams == null || passwordLoginParams.password == null) {
         throw new IllegalArgumentException("password params should not be null");
     }
     String str = passwordLoginParams.userId;
     String str2 = passwordLoginParams.password;
     String str3 = passwordLoginParams.deviceId;
     String str4 = TextUtils.isEmpty(passwordLoginParams.serviceId) ? PASSPORT_SID : passwordLoginParams.serviceId;
     String str5 = passwordLoginParams.captIck;
     String str6 = passwordLoginParams.captCode;
     String[] strArr = passwordLoginParams.hashedEnvFactors;
     boolean z = passwordLoginParams.returnStsUrl;
     boolean z2 = passwordLoginParams.needProcessNotification;
     MetaLoginData metaLoginData = passwordLoginParams.metaLoginData;
     ActivatorPhoneInfo activatorPhoneInfo = passwordLoginParams.activatorPhoneInfo;
     EasyMap easyPut = new EasyMap().easyPutOpt(ProfileRecordUtils.Area.f6514a, str).easyPut(a.e, CloudCoder.getMd5DigestUpperCase(str2)).easyPutOpt("sid", str4).easyPutOpt("captCode", str6).easyPutOpt(d.p, passwordLoginParams.countryCode).easyPut("_json", Constants.SdkSettings.VALUE_TRUE);
     addEnvToParams(easyPut, strArr);
     EasyMap easyPutOpt = new EasyMap().easyPutOpt("ick", str5).easyPutOpt("ticketToken", passwordLoginParams.ticketToken);
     addDeviceIdInCookies(easyPutOpt, str3);
     addAntiSpamIpAddressInCookies(easyPutOpt);
     if (activatorPhoneInfo != null) {
         easyPut.easyPutOpt("userHash", activatorPhoneInfo.phoneHash);
         easyPutOpt.easyPutOpt("activatorToken", activatorPhoneInfo.activatorToken);
     }
     String str7 = URLs.URL_LOGIN_AUTH2_HTTPS;
     PassportRequestArguments passportRequestArguments = new PassportRequestArguments();
     passportRequestArguments.putAllParams(easyPut);
     passportRequestArguments.putAllCookies(easyPutOpt);
     passportRequestArguments.setUrl(str7);
     passportRequestArguments.setReadBody(true);
     PassportLoginRequest.ByPassword byPassword = new PassportLoginRequest.ByPassword(passportRequestArguments, str, str4, metaLoginData);
     try {
         ProtocolLogHelper.newRequestLog(str7, HttpMethod.POST, new String[]{a.e, "ticketToken", "userHash", "activatorToken"}).paramWithMaskOrNull(easyPut).cookieWithMaskOrNull(easyPutOpt).log();
         SimpleRequest.StringContent executeEx = byPassword.executeEx();
         logLoginResponseAllowNull(str7, executeEx);
         if (executeEx == null) {
             throw new IOException("failed to get response from server");
         }
         try {
             return processLoginContent(executeEx, str4, z2, z);
         } catch (PackageNameDeniedException unused) {
             throw new InvalidResponseException("It's not loginByPassToken(), PackageNameDeniedException is unexpected");
         }
     } catch (PassportCAException unused2) {
         throw new IllegalStateException("this should never happen in product environment.Have you set sDisableLoginFallbackForTest to be true? ");
     }
 }

```

其中，我们发现有 2 行代码是插入 cookie 的，因为这个接口也正好是一个封装 HTTP 数据包的过程。

```
addDeviceIdInCookies(easyPutOpt, str3);
addAntiSpamIpAddressInCookies(easyPutOpt);

```

从字面意思我们可以看到，add（插入）DeviceId（设备 ID）In（进）Cookies（小饼干）。。。  
所以我们就跟进去看看，

```
private static void addDeviceIdInCookies(EasyMap easyMap, String str) {
      if (easyMap == null) {
          throw new IllegalArgumentException("cookie params should not be null");
      }
      Application applicationContext = XMPassportSettings.getApplicationContext();
      String oaid = OAIDUtil.getOAID(applicationContext);
      if (TextUtils.isEmpty(str)) {
          str = getHashedDeviceId();
      }
      if (applicationContext != null) {
          AssertionUtils.checkCondition(applicationContext, !TextUtils.isEmpty(str), "deviceId cannot be empty", true);
      }
      easyMap.easyPutOpt("deviceId", str).easyPutOpt("pass_o", oaid).easyPutOpt(SimpleRequestForAccount.COOKIE_NAME_USER_SPACE_ID, UserSpaceIdUtil.getNullableUserSpaceIdCookie());
  } 
```

由于它传进来的第二个参数就是 deviceId，我们本来进来只是核实它是不是给 cookies 插入的 deviceId，没想到这个方法里面竟然做了如果 deviceId 为空则重新获取的判断，这样就省了我们去往上溯源的操作了。

0x03 算法分析
---------

```
if (TextUtils.isEmpty(str)) {
        str = getHashedDeviceId();
    }

```

所以直接就跟进 getHashedDeviceId 里面

```
private static String getHashedDeviceId() {
    return new HashedDeviceIdUtil(XMPassportSettings.getApplicationContext()).getHashedDeviceIdNoThrow();
}

```

没啥好说的，继续跟进 getHashedDeviceIdNoThrow 里面。

```
public synchronized String getHashedDeviceIdNoThrow() {
      return getHashedDeviceId(true);
  }

```

继续跟进 getHashedDeviceId 里面。

```
public synchronized String getHashedDeviceId(boolean z) {
     IUnifiedDeviceIdFetcher unifiedDeviceIdFetcher;
     DeviceIdPolicy policy = policy();
     if (policy == DeviceIdPolicy.RUNTIME_DEVICE_ID_ONLY) {
         return getRuntimeDeviceIdHashed();
     } else if (policy != DeviceIdPolicy.CACHED_THEN_RUNTIME_THEN_PSEUDO) {
         throw new IllegalStateException("unknown policy " + policy);
     } else {
         String loadHistoricalHashedDeviceId = loadHistoricalHashedDeviceId();
         if (!TextUtils.isEmpty(loadHistoricalHashedDeviceId)) {
             return loadHistoricalHashedDeviceId;
         }
         String runtimeDeviceIdHashed = getRuntimeDeviceIdHashed();
         if (runtimeDeviceIdHashed != null) {
             saveHistoricalHashedDeviceId(runtimeDeviceIdHashed);
             return runtimeDeviceIdHashed;
         }
         if (z) {
             if (!isMainThread() && (unifiedDeviceIdFetcher = GlobalConfig.getInstance().getUnifiedDeviceIdFetcher()) != null) {
                 String hashedDeviceId = unifiedDeviceIdFetcher.getHashedDeviceId(this.context);
                 if (!TextUtils.isEmpty(hashedDeviceId)) {
                     saveHistoricalHashedDeviceId(hashedDeviceId);
                     return hashedDeviceId;
                 }
             }
         }
         String oaid = OAIDUtil.getOAID(this.context);
         if (!TextUtils.isEmpty(oaid)) {
             String str = OAID_PREFIX + DeviceIDCloudCoder.getDataMd5Digest(oaid.getBytes());
             saveHistoricalHashedDeviceId(str);
             return str;
         }
         String androidID = getAndroidID(this.context);
         if (!TextUtils.isEmpty(androidID)) {
             String str2 = ANDROID_ID_PREFIX + DeviceIDCloudCoder.getDataMd5Digest(androidID.getBytes());
             saveHistoricalHashedDeviceId(str2);
             return str2;
         }
         String createPseudoDeviceId = createPseudoDeviceId();
         saveHistoricalHashedDeviceId(createPseudoDeviceId);
         return createPseudoDeviceId;
     }
 }

```

这里我们看到有好几层判断，我们粗略看了一下，它就是用不同的方式获取 deviceId，我们可以看到有获取 androidId 的，获取 OAID 的方式。如果其中一个可以获取则直接返回。那岂不是美哉？　　随便跟一个就可以了。  
我们就跟 String androidID = getAndroidID(this.context); 的方式.

```
public static String getAndroidID(Context context2) {
    return Settings.Secure.getString(context2.getContentResolver(), "android_id");
}

```

我们跟进去看到, 就是一个很基础的获取手机 android_id 操作. 获取后我们看到它刚才调用了  
String str2 = ANDROID_ID_PREFIX + DeviceIDCloudCoder.getDataMd5Digest(androidID.getBytes());  
字符串拼接, 加上 MD5 的方式, 我们看下这个字符串里面是啥常量.

```
private static final String ANDROID_ID_PREFIX = "an_";

```

我们看到它会在 Android_id 前面加上 an_ 我估计是方便服务器判断这是一个什么样的 deviceId. 但是我们抓包看到并没有 an _也就是我这个 deviceId 并不是这个方式获取的. 但是理论上讲这样也没错, 只是有可能服务器会判定为某个等级的风险级别把..  
我们用同样的方式也可以看到 OAID 的方式获取, 前面也会带 oa_ 字符串

```
private static final String OAID_PREFIX = "oa_";
 
String str = OAID_PREFIX + DeviceIDCloudCoder.getDataMd5Digest(oaid.getBytes());

```

但是我们发现一处地方, 它没有拼接常量的, 估计就是货真价实的 deviceId 的获取方式.

```
String runtimeDeviceIdHashed = getRuntimeDeviceIdHashed();
        if (runtimeDeviceIdHashed != null) {
            saveHistoricalHashedDeviceId(runtimeDeviceIdHashed);
            return runtimeDeviceIdHashed;
        }

```

所以我们就跟进 getRuntimeDeviceIdHashed 里面看看是如何获取的.

```
public String getRuntimeDeviceIdHashed() {
    try {
        String userEnvironmentPlainDeviceId = getUserEnvironmentPlainDeviceId();
        if (legal(userEnvironmentPlainDeviceId)) {
            return DeviceIdHasher.hashDeviceInfo(userEnvironmentPlainDeviceId);
        }
        return null;
    } catch (SecurityException e) {
        AccountLog.w(TAG, "can't get deviceid.", e);
        return null;
    }
}

```

我们看到他获取到之后会传给一个 hashDeviceInfo 方法进行处理, 看字面应该是 hash 某种算法, 但我们先继续跟 getUserEnvironmentPlainDeviceId 方法内看看是如何获取的.

```
public String getUserEnvironmentPlainDeviceId() {
    return this.plainDeviceIdFetcher.getPlainDeviceId(this.context);
}

```

继续跟进

```
public final class PlainDeviceIdUtil {
 
    public interface IPlainDeviceIdFetcher {
        String getPlainDeviceId(Context context);
    }
 
    private static class FetcherHolder {
        /* access modifiers changed from: private */
        public static volatile IPlainDeviceIdFetcher sInstance = new PlainDeviceIdUtilImplDefault();
 
        private FetcherHolder() {
        }
    }
 
    public static IPlainDeviceIdFetcher getFetcherInstance() {
        return FetcherHolder.sInstance;
    }
 
    public static void setFetcherInstance(IPlainDeviceIdFetcher iPlainDeviceIdFetcher) {
        IPlainDeviceIdFetcher unused = FetcherHolder.sInstance = iPlainDeviceIdFetcher;
    }
 
    public static final class PlainDeviceIdUtilImplDefault implements IPlainDeviceIdFetcher {
        public String getPlainDeviceId(Context context) {
            if (context == null) {
                return null;
            }
            String deviceId = ((TelephonyManager) context.getSystemService("phone")).getDeviceId();
            return TextUtils.isEmpty(deviceId) ? MacAddressUtil.getMacAddress(context) : deviceId;
        }
    }
}

```

这里我直接给出了跟进去的这个类的所有代码, 因为我们看到它的第一个方法就是刚刚跟进他, 他的定义是 interface(接口), 但是正巧他就在底下调用了, 所以它的实现就是最下面的那个 PlainDeviceIdUtilImplDefault 方法. 最终得出它获取的是手机的 deviceId 在进行刚刚我们看到的 hash 某种算法

```
String deviceId = ((TelephonyManager) context.getSystemService("phone")).getDeviceId();

```

得知是这种方式后, 我们返回去跟进刚刚看到的算法方法.

```
public static String hashDeviceInfo(String str) {
       return hashDeviceInfo(str, 8);
   }
 
   public static String hashDeviceInfo(String str, int i) {
       if (str == null) {
           return null;
       }
       try {
           return Base64.encodeToString(MessageDigest.getInstance("SHA1").digest(str.getBytes()), i).substring(0, 16);
       } catch (NoSuchAlgorithmException unused) {
           throw new IllegalStateException("failed to init SHA1 digest");
       }
   }

```

我们可以清楚的看到, 算法就是进行了 SHA1 然后在进行 BASE64 编码, 最后得出的 deviceId

0x04 总结
-------

该算法也是非常简单的, 重要的是能够掌握快速的代码定位技巧, 也能够节省出不少时间. 有时候找出一个参数的使用比找出参数的生成算法还要麻烦..

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)