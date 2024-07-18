> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282546.htm)

> [原创] 某易新闻逆向全过程

[原创] 某易新闻逆向全过程

2 小时前 229

### [原创] 某易新闻逆向全过程

 [![](http://passport.kanxue.com/upload/avatar/390/945390.png?1673331326)](user-home-945390.htm) [行简](user-home-945390.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2 小时前  229

0x01、前言
-------

#### 本文中所有内容仅供研究与学习使用，请勿用于任何商业用途和非法用途，否则后果自负！

0x02、环境
-------

app 版本：110.1(1827) 酷安下载  
设备：Pixel 2XL Android 8.1  
抓包工具：Charles + Postern VPX 抓包  
反汇编工具：JADX 1.4.5、IDA Pro 8.3.0  
hook：frida 12.8.0、frida-tools 5.3.0

0x03、分析手法
---------

静态分析  
动态分析  
网络流量分析  
猜，你猜猜我猜猜

0x04、抓包
-------

![](https://bbs.kanxue.com/upload/attach/202407/945390_6QQ5V4ZJDB7END3.webp)

### 1、Header

> :method: GET  
> :path: /nc/api/v1/search/flow/comp?start=Kg%3D%3D&limit=20&q=NuS6uuWFseS6qzIwMjTmrKfmtLLmna%2Fph5HpnbQ%3D&deviceId=bvYT4UzsuBXNhfobtucjoWbhxxvqginXtZw7lCuhHuFXCnf2uGrW417TW%2FmVXqy1IIGNeE0nI41SFrBIaL1THA%3D%3D&version=newsclient.110.1.android&channel=c2VhcmNo&canal=UVFfbmV3c195dW55aW5nNA%3D%3D&dtype=0&tabname=zonghe&position=5pCc57Si5qGG6aKE572u6K%2BN&ts=1721110863&sign=W7Xphf%2BpqyOGN9JoXUHnuo0ikgGVNQynXl0Z9SFrPG148ErR02zJ6%2FKXOnxX046I&spever=FALSE  
> :authority: gw.m.163.com  
> :scheme: https  
> add-to-queue-millis: 1721110863718  
> data4-sent-millis: 1721110863719  
> cache-control: no-cache  
> user-agent: NewsApp/110.1 Android/8.1.0 (google/Pixel 2 XL)  
> x-nr-trace-id: 1721110863720_199869332_ODUwNjE0ZjAyOGJhZTNiYV9fZ29vZ2xlX1BpeGVsIDIgWEw%3D  
> x-nr-ise: 0  
> x-xr-original-host: gw.m.163.com  
> user-c: 5pCc57Si  
> user-rc: UjgzLZ+E4Lemnj+sMro9qwqQ3xlDp4PUECu18073DbE2Sp1cMm3KWoG4EVq0Iff0  
> user-d: bvYT4UzsuBXNhfobtucjoWbhxxvqginXtZw7lCuhHuFXCnf2uGrW417TW/mVXqy1IIGNeE0nI41SFrBIaL1THA==  
> user-vd: WfXzE4qBsQrzqoxcYqkdAh77TaqeHGwDc3c5MKUbPwL1PDEQtOAxYiJJj6XJo9TGePBK0dNsyevylzp8V9OOiA==  
> user-appid: TItcOwjV9bndQ91C5VadYg==  
> user-sid: jeYbWWG30X4+b4psq4KnvtJQ7bvpgJC2TvUpgWA0pfw=  
> user-lc: 67NqtW9W02z/qXjaEOOHag==  
> user-n: yEWDFuJGE3Gmj2a0IPdYcA==  
> user-id: rTboMPOe7X3a3PlAcfTomAyKsptKyhPdg7sH0emGPiqAQ4ozbxeRq4WEUnIhA/QejIuMU9rtRKIrUSa49DmGut4ZRL0MIJQQ8KOb5fiSUJuMVMtqqnzmaVeVeFXjZ10SePBK0dNsyevylzp8V9OOiA==  
> x-nr-ts: 1721110863728  
> x-nr-sign: 6db19e20ad7a9a890ca02e7a509ba00d  
> x-nr-net-lib: okhttp  
> accept-encoding: br,gzip

### 2、params

> 'start': 'Kg==',  
> 'limit': '20',  
> 'q': 'NuS6uuWFseS6qzIwMjTmrKfmtLLmna/ph5HpnbQ=',  
> 'deviceId': 'bvYT4UzsuBXNhfobtucjoWbhxxvqginXtZw7lCuhHuFXCnf2uGrW417TW/mVXqy1IIGNeE0nI41SFrBIaL1THA==',  
> 'version': 'newsclient.110.1.android',  
> 'channel': 'c2VhcmNo',  
> 'canal': 'UVFfbmV3c195dW55aW5nNA==',  
> 'dtype': '0',  
> 'tabname': 'zonghe',  
> 'position': '5pCc57Si5qGG6aKE572u6K+N',  
> 'ts': '1721110863',  
> 'sign': 'W7Xphf+pqyOGN9JoXUHnuo0ikgGVNQynXl0Z9SFrPG148ErR02zJ6/KXOnxX046I',  
> 'spever': 'FALSE',

0x05、逆向分析
---------

多次抓包对比确定需要逆向的参数，这边参数实在太多了，将 app 拖入 jadx 等待反编译完成后逐个进行分析。

### 1、Host

固定：'gw.m.163.com',

### 2、add-to-queue-millis

整数型 13 位时间戳：int(time.time() * 1000),

### 3、data4-sent-millis

整数型 13 位时间戳：int(time.time() * 1000),

### 4、cache-control

固定：'no-cache',

### 5、user-agent

给了就好：'NewsApp/110.1 Android/8.1.0 (google/Pixel 2 XL)',

### 6、x-nr-trace-id

x-nr-trace-id: 1721110863720_199869332_ODUwNjE0ZjAyOGJhZTNiYV9fZ29vZ2xlX1BpeGVsIDIgWEw%3D  
初步猜测：  
整数型 13 位时间戳 + "_"+"???"+"_" + ODUwNjE0ZjAyOGJhZTNiYV9fZ29vZ2xlX1BpeGVsIDIgWEw%3D  
在 Jadx 中查找，看看具体是如何生成的：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_AP4HRQKTPDCSGEN.webp)  
总计找到 37 处，着重查看 request 发包相关的：  
定位到：GalaxyResponse a() 函数，代码有做删减，看到 X-NR-Trace-Id 是通过 this.f12927b 来的：

```
@Override // com.netease.galaxy.net.IRequest
public GalaxyResponse a() throws Throwable {
    OkHttpClient a2;
    method.header("X-NR-Trace-Id", this.f12927b);
} 

```

跟进 c(String str) 函数中:

```
private String c(String str) {
    return System.currentTimeMillis() + "_" + str + "_" + Galaxy.O(Galaxy.N());
}

```

从 c(String str) 函数中可以看出，和我们的初步猜测是一样的，str 为 String.valueOf(hashCode()) 也就是当前对象的哈希码（HashCode）的字符串形式，使用 frida 进行 hook：

```
Java.perform(function () {
    var GalaxyRequest = Java.use("com.netease.galaxy.net.GalaxyRequest");
    GalaxyRequest["c"].implementation = function (str) {
        console.log('c is called' + ', ' + 'str: ' + str);
        var ret = this.c(str);
        console.log('c ret value is ' + ret);
        return ret;
    };
})

```

![](https://bbs.kanxue.com/upload/attach/202407/945390_B2ACTHTNNEZ2BCK.webp)  
据返回值可以发现：ODUwNjE0ZjAyOGJhZTNiYV9fZ29vZ2xlX1BpeGVsIDIgWEw%3D，字符一直是固定的，那么总结可得：  
整数型 13 位时间戳 + "_"+"hashCode()"+"_" + ODUwNjE0ZjAyOGJhZTNiYV9fZ29vZ2xlX1BpeGVsIDIgWEw%3D

### 7、x-nr-ise

固定值：'0',

### 8、x-xr-original-host

固定值：'gw.m.163.com',

### 9、user 系列参数

在 Jadx 中查找 user-c，查看其具体是如何生成的，仅定位到两处，一眼出：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_895RGMDFCMTSD25.webp)  
定位到：Request F1() 函数，代码有做删减，看到 User-C 是通过 URLEncoder.encode(StringUtil.e(o2, "UTF-8") 来的，User-U、User-D、User-N 三个参数都是通过 Encrypt.getEncryptedParams(i2) 来的。

#### 1、user-c

```
public static Request F1(String str, String str2) {
    ArrayList arrayList = new ArrayList();
    String d2 = Common.g().a().getData().d();
    if (!TextUtils.isEmpty(d2)) {
        arrayList.add(new Header("User-U", Encrypt.getEncryptedParams(d2)));
    }
    String s2 = SystemUtilsWithCache.s();
    if (!TextUtils.isEmpty(s2)) {
        arrayList.add(new Header("User-D", Encrypt.getEncryptedParams(s2)));
    }
    String i2 = NetUtil.i();
    if (!TextUtils.isEmpty(i2)) {
        try {
            arrayList.add(new Header("User-N", Encrypt.getEncryptedParams(i2)));
        } catch (Exception unused) {
        }
    }
    String o2 = CommonGalaxy.o();
    if (!TextUtils.isEmpty(o2)) {
        try {
            arrayList.add(new Header("User-C", URLEncoder.encode(StringUtil.e(o2, "UTF-8"), "UTF-8")));
        } catch (UnsupportedEncodingException unused2) {
        }
    }
    return BaseRequestGenerator.a(String.format(NGRequestUrls.PicSet.f23298a, str, str2), arrayList);
}

```

先分析 User-C 对 StringUtil.e(o2, "UTF-8") 返回的字符串进行 URL 编码，使用 UTF-8 字符集，写个 hook 代码，查看其编码的对象：

```
Java.perform(function () {
  var StringUtil = Java.use("com.netease.newsreader.support.utils.string.StringUtil");
    StringUtil["e"].implementation = function (str, str2) {
        console.log('e is called' + ', ' + 'str: ' + str + ', ' + 'str2: ' + str2);
        var ret = this.e(str, str2);
        console.log('e ret value is ' + ret);
        return ret;
    };
})

```

![](https://bbs.kanxue.com/upload/attach/202407/945390_62PQTT7PX8NCMH2.webp)  
反复进行抓包，其有两个值反复横跳：

> user-c: 5pCc57S  
> user-c: 5aS05p2h

对其进行 base64 解码：

> 5pCc57S -> 搜索  
> 5aS05p2h -> 头条

#### 2、user-d、user-n、user-rc 等等

继续分析 User-U、User-D、User-N ...... 等参数，跟进 getEncryptedParams() 方法：

```
public static String getEncryptedParams(String str, int i2) {
    if (TextUtils.isEmpty(str)) {
        return str;
    }
    synchronized (sEncryptCache) {
        Map map = sEncryptCache.get(i2);
        if (map != null && !TextUtils.isEmpty(map.get(str))) {
            return map.get(str);
        }
        String encryptedParamsInner = getEncryptedParamsInner(str, i2);
        if (map == null) {
            map = new HashMap<>(2);
            sEncryptCache.put(i2, map);
        }
        map.put(str, encryptedParamsInner);
        return encryptedParamsInner;
    }
} 
```

调用 getEncryptedParamsInner(str, i2) 方法，跟进查看：

```
private static String getEncryptedParamsInner(String str, int i2) {
    return getBase64Str(callEncrypt(Core.context(), str, i2));
}

```

调用 callEncrypt 方法，跟进查看：

```
private static byte[] callEncrypt(Context context, String str, int i2) {
    try {
        return encrypt(context, str, i2);
    } catch (Error e2) {
        e2.printStackTrace();
        return null;
    } catch (Exception e3) {
        e3.printStackTrace();
        return null;
    }
}

```

调用 encrypt 方法，跟进查看：

```
private static native synchronized byte[] encrypt(Context context, String str, int i2);

```

到 so 层了，定位其对应的 so 文件：

```
static {
    try {
        System.loadLibrary("random");
    } catch (Error unused) {
    }
}

```

需到 librandom.so 文件中，查看 Java_com_netease_nr_biz_pc_sync_Encrypt_encrypt 方法的实现，在分析之前先 hook 确认下查找的点没错：

```
Java.perform(function () {
  var ByteString = Java.use("com.android.okhttp.okio.ByteString");
  var Encrypt = Java.use("com.netease.nr.biz.pc.sync.Encrypt");
  Encrypt["encrypt"].implementation = function (context, str, i2) {
      console.log('encrypt is called' + ', ' + 'context: ' + context + ', ' + 'str: ' + str + ', ' + 'i2: ' + i2);
      var ret = this.encrypt(context, str, i2);
      console.log('encrypt ret value is ' + ret);
      console.log("\n\ncallEncrypt ret str_hex: "+ ByteString.of(ret).hex());
      return ret;
  };
})

```

![](https://bbs.kanxue.com/upload/attach/202407/945390_FFRPJ24ZF52ZRJJ.webp)  
确认无误，跟进 so 中进行分析：

```
__int64 __fastcall Java_com_netease_nr_biz_pc_sync_Encrypt_encrypt(
__int64 a1,
__int64 a2,
__int64 a3,
__int64 a4,
unsigned int a5)
{
    __int64 v9; // x1
    __int64 RandomKey; // x4
 
    RandomKey = getRandomKey(a1, a2, a3, a5);
    if ( a5 == 3 )
        return enUnderpants(a1, v9, a3, a4, RandomKey);
    else
        return doEn();
}

```

分析其走，doEn() 方法，跟进查看：

```
__int64 __fastcall doEn(__int64 a1, __int64 a2, __int64 a3, __int64 a4, __int64 a5)
{
  v8 = (*(*a1 + 48LL))(a1, "java/lang/String", a3);
  v9 = (*(*a1 + 1336LL))(a1, "utf-8");
  v10 = (*(*a1 + 264LL))(a1, v8, "getBytes", "(Ljava/lang/String;)[B");
  v11 = (*(*a1 + 272LL))(a1, a5, v10, v9);
  v12 = (*(*a1 + 272LL))(a1, a4, v10, v9);
  v13 = malloc(0x15uLL);
  strcpy(v13, "AES/ECB/PKCS7Padding");
  v14 = v13;
  v15 = (*(*a1 + 1336LL))(a1, "D@V");
  free(v14);
  v16 = (*(*a1 + 48LL))(a1, "javax/crypto/spec/SecretKeySpec");
  v17 = (*(*a1 + 264LL))(a1, v16, "", "([BLjava/lang/String;)V");
  v18 = (*(*a1 + 224LL))(a1, v16, v17, v11, v15);
  v19 = malloc(0x15uLL);
  strcpy(v19, "AES/ECB/PKCS7Padding");
  v20 = (*(*a1 + 1336LL))(a1, v19);
  free(v19);
  v21 = malloc(3uLL);
  strcpy(v21, "BC");
  v22 = v21;
  v23 = (*(*a1 + 1336LL))(a1, v21);
  free(v22);
  v24 = (*(*a1 + 48LL))(a1, "javax/crypto/Cipher");
  v25 = (*(*a1 + 904LL))(a1, v24, "getInstance", "(Ljava/lang/String;Ljava/lang/String;)Ljavax/crypto/Cipher;");
  v26 = (*(*a1 + 912LL))(a1, v24, v25, v20, v23);
  v27 = (*(*a1 + 264LL))(a1, v24, "init", "(ILjava/security/Key;)V");
  (*(*a1 + 488LL))(a1, v26, v27, 1LL, v18);
  v28 = (*(*a1 + 264LL))(a1, v24, "doFinal", "([B)[B");
  return (*(*a1 + 272LL))(a1, v26, v28, v12);
} 
```

可确定使用：AES/ECB/PKCS7Padding 进行加密，使用自吐脚本进行 hook：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_5EV8XH94GDMS7WU.webp)  
则可得，整个加密流程为：str -> AES/ECB/PKCS7Padding -> base64，如：

> user-d: bvYT4UzsuBXNhfobtucjoWbhxxvqginXtZw7lCuhHuFXCnf2uGrW417TW/mVXqy1IIGNeE0nI41SFrBIaL1THA==

![](https://bbs.kanxue.com/upload/attach/202407/945390_Q54GMDTPQZS9A4W.webp)  
对照抓包及 hook 到的参数，可以确定 user-d、user-n、user-rc、user-vd、user-appid、user-lc、user-id 参数的生成都是如此，其原始值大多为某一个值经过 base64 编码后在经过 url 编码得到的：

> user-n：unknown  
> user-lc：110000  
> user-appid：2x1kfBk63z  
> user-rc：{"ad":true,"adCrossPlatform":true}  
> user-d：ODUwNjE0ZjAyOGJhZTNiYV9fZ29vZ2xlX1BpeGVsIDIgWEw%3D  
> user-vd：MTcyMTA5NzUxNjc2Ml83OTk1NzcwOV9NQ1hDUk1jTA%3D%3D  
> user-id：3E9B7EF20462938EBAEBED8BF0E5338B12A5A5F17B58B98104C823E4F8F2452656EF1BC4BCA66104804E0889C39A2A4B

#### 3、user-sid

在 Jadx 中查找，仅有一处进行了定义  
![](https://bbs.kanxue.com/upload/attach/202407/945390_3WEBZ67FPG4FXNM.webp)  
进入查看其引用也仅有一处：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_BGJ5YYRJZ56JGWM.webp)  
进入查看 G() 方法：

```
public String G() {
    return ((IGalaxyApi) SDK.a(IGalaxyApi.class)).getSessionId();
}

```

获取一个实现了 IGalaxyApi 接口的对象，并调用其 getSessionId() 方法来获取当前会话的 ID，然后将该 ID 作为字符串返回。对其进行 hook：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_483YRWYSXYR67JN.webp)  
查看其抓包得到的值：

> user-sid：OnXVDmIU6Mqla688+2Zr+psF7OETLwuUZSrrZY5mdmM=

类似于之前分析的 user 系列参数得到的值，复制 eddhaa1721202144359 字符串进行加密验证，无误：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_DRJ5C9EPZJCFU7S.webp)

分析其值的构成应为：字符 + 时间戳。  
反复抓包得出结论其时间戳为本次 app 启动时的时间戳，在启动后时间戳不变。  
而在字符串前的六位字符，并未发现其生成点，猜测为随机生成的值，实际在爬取过程中随意给定加上对应时间戳遍可。

### 10、x-nr-ts

整数型 13 位时间戳：int(time.time() * 1000),

### 11、x-nr-net-lib

固定：'gw.m.163.com',

### 12、x-nr-sign

在 Jadx 中查找，仅有一处进行了定义：

```
public static final String f29790p = "X-NR-SIGN";

```

![](https://bbs.kanxue.com/upload/attach/202407/945390_VVCM4ZARRSX4EFN.webp)

进入查看其引用，定位到 com.netease.newsreader.common.net.interceptor：

```
public Response intercept(@NotNull Interceptor.Chain chain) {
    Intrinsics.p(chain, "chain");
    Request request = chain.request();
    if (b(request)) {
        String query = request.url().query();
        if (!TextUtils.isEmpty(query)) {
            Request.Builder newBuilder = request.newBuilder();
            long currentTimeMillis = System.currentTimeMillis();
            request = newBuilder.header(HttpUtils.f29791q, String.valueOf(currentTimeMillis)).header(HttpUtils.f29790p, a(query, currentTimeMillis)).build();
        }
    }
    return chain.proceed(request);
}

```

分析其生成在 a() 方法中：

```
private final String a(String str, long j2) {
    if (TextUtils.isEmpty(str)) {
        return "";
    }
    String n2 = StringUtils.n(((Object) str) + HttpUtils.f29793s + j2);
    Intrinsics.o(n2, "md5(\"$queryString${HttpUtils.SING_SALT}$ts\")");
    return n2;
}

```

重点就是这句代码

> String n2 = StringUtils.n(((Object) str) + HttpUtils.f29793s + j2);

将 str、HttpUtils.f29793s 和 j2 进行字符串拼接。其中，HttpUtils.f29793s 为静态变量其值为：f29793s = "gNlVGcSKf5"。然后使用 StringUtils.n() 方法对拼接后的字符串进行处理，StringUtils.n() 方法，用于计算字符串的 MD5 哈希值。

```
public static String n(String str) {
    if (TextUtils.isEmpty(str)) {
        return str;
    }
    try {
        return a(MessageDigest.getInstance("MD5").digest(g(str, Charset.forName("UTF-8"))), false);
    } catch (NoSuchAlgorithmException e2) {
        throw new AssertionError(e2);
    }
}

```

对 StringUtils.n() 方法其进行 hook：

```
Java.perform(function () {
    var StringUtils = Java.use("com.netease.newsreader.framework.util.string.StringUtils");
    StringUtils["n"].implementation = function (str) {
        console.log('n is called' + ', ' + 'str: ' + str);
        var ret = this.n(str);
        console.log('n ret value is ' + ret);
        return ret;
    };
})

```

![](https://bbs.kanxue.com/upload/attach/202407/945390_ZPNW838JACJQME6.webp)  
![](https://bbs.kanxue.com/upload/attach/202407/945390_49XTKNZYVQHUR2R.webp)  
对比发现无误，继续分析字段内容，通过在 Jadx 中搜索关键词，定位到 Request M1 函数：

```
public static Request M1(String str, String str2, String str3, String str4, String str5, String str6) {
    String s2 = SystemUtilsWithCache.s();
    String g2 = SearchModel.g(BaseApplication.h());
    String b2 = CurrentColumnInfo.b();
    long currentTimeMillis = System.currentTimeMillis() / 1000;
    String str7 = s2 + String.valueOf(currentTimeMillis);
    String c2 = OpenInfo.c();
    String b3 = OpenInfo.b();
    if (!TextUtils.isEmpty(str7)) {
        str7 = StringUtil.c(Encrypt.getEncryptedParams(StringUtils.n(str7)));
    }
    ArrayList arrayList = new ArrayList();
    arrayList.add(new FormPair("start", StringUtil.h(str)));
    arrayList.add(new FormPair("limit", String.valueOf(20)));
    arrayList.add(new FormPair("q", StringUtil.h(str2)));
    arrayList.add(new FormPair("deviceId", StringUtil.c(Encrypt.getEncryptedParams(s2))));
    arrayList.add(new FormPair("version", g2));
    arrayList.add(new FormPair("channel", StringUtil.h(b2)));
    arrayList.add(new FormPair("canal", StringUtil.h(SystemUtilsWithCache.n())));
    arrayList.add(new FormPair("dtype", str5));
    arrayList.add(new FormPair("tabname", str6));
    if (!TextUtils.isEmpty(str4)) {
        arrayList.add(new FormPair("qId", StringUtil.h(str4)));
    }
    if (!TextUtils.isEmpty(str3)) {
        arrayList.add(new FormPair("position", StringUtil.h(str3)));
    }
    arrayList.add(new FormPair("ts", String.valueOf(currentTimeMillis)));
    arrayList.add(new FormPair("sign", str7));
    arrayList.add(new FormPair("spever", "FALSE"));
    if (!TextUtils.isEmpty(c2)) {
        arrayList.add(new FormPair("open", c2));
    }
    if (!TextUtils.isEmpty(b3)) {
        arrayList.add(new FormPair("openpath", b3));
    }
    return BaseRequestGenerator.b(NGRequestUrls.Search.f23357c, arrayList);
}

```

可以看到相关参数几乎都是在这生成的，具体针对抓到报的内容进行分析：

> start=Kg==& 固定  
> limit=20& 固定  
> q=5a6d6ams5Lit5Zu95YWo57O75rao5Lu3& 搜索词的 base64 编码  
> deviceId=bvYT4UzsuBXNhfobtucjoWbhxxvqginXtZw7lCuhHuFXCnf2uGrW417TW/mVXqy1IIGNeE0nI41SFrBIaL1THA==& 固定  
> version=newsclient.110.1.android& 固定  
> channel=c2VhcmNo& 固定 base64 编码 原值：search  
> canal=UVFfbmV3c195dW55aW5nNA==& 固定 base64 编码 原值：QQ_news_yunying4  
> dtype=0& 固定  
> tabname=zonghe& 固定  
> qId=Nzg3NjM1NDU1NTE2MDU5NA==& 未知  
> position=6L6T5YWl& 固定 base64 编码 原值：输入  
> ts=1721207040& 时间戳  
> sign=+gfuFTKMwWg3lozy1k7VVRaRMDojncFD1rm9fQqD+cJ48ErR02zJ6/KXOnxX046I& 未知  
> spever=FALSE 固定  
> gNlVGcSKf5 固定  
> 1721207040973 时间戳

#### 1、qid

可得在此处需要进一步分析的内容有两处：qId、sign，又经反复不同姿势的方式抓包发现 qid 参数可要可不要，在首次搜索并无该参数：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_5RWYGQK22VK8KKD.webp)  
重复在搜索栏搜索时 qid 参数出现：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_Y2NJQ8DV6BQ9HVM.webp)

#### 2、sign

根据代码分析，sign 值的来源为 str7，str7 的赋值代码就在 Request M1 函数中，如下所示，代码有删减：

```
if (!TextUtils.isEmpty(str7)) {
    str7 = StringUtil.c(Encrypt.getEncryptedParams(StringUtils.n(str7)));
}
arrayList.add(new FormPair("sign", str7));

```

StringUtils.n(str7)： StringUtils.n 方法对传入的字符串 str7 进行 MD5 操作。

```
public static String n(String str) {
    if (TextUtils.isEmpty(str)) {
        return str;
    }
    try {
        return a(MessageDigest.getInstance("MD5").digest(g(str, Charset.forName("UTF-8"))), false);
    } catch (NoSuchAlgorithmException e2) {
        throw new AssertionError(e2);
    }
}

```

Encrypt.getEncryptedParams(...)： Encrypt.getEncryptedParams 方法，在上面有进行分析，对传入的参数使用 AES/ECB/PKCS7Padding 进行加密。  
StringUtil.c(...)： StringUtil.c 方法，输入进行 URLEncoder 形式的处理，然后返回结果。

```
public static String c(String str) {
    if (TextUtils.isEmpty(str)) {
        return "";
    }
    try {
        return URLEncoder.encode(str, "UTF-8");
    } catch (Exception unused) {
        return "";
    }
}

```

分别对 StringUtils.n、StringUtil.c 方法进行 hook：

```
Java.perform(function (){
  var StringUtils = Java.use("com.netease.newsreader.framework.util.string.StringUtils");
    StringUtils["n"].implementation = function (str) {
        console.log('n is called' + ', ' + 'str: ' + str);
        var ret = this.n(str);
        console.log('n ret value is ' + ret);
        return ret;
    };
     
    var StringUtil = Java.use("com.netease.newsreader.support.utils.string.StringUtil");
    StringUtil["c"].implementation = function (str) {
        console.log('c is called' + ', ' + 'str: ' + str);
        var ret = this.c(str);
        console.log('c ret value is ' + ret);
        return ret;
    };
})

```

![](https://bbs.kanxue.com/upload/attach/202407/945390_JAHKXBSSFTKA8DH.webp)  
所有参数分析完毕。

0x06、模拟请求
---------

上述将所有相关的参数都分析完毕，接下来就是针对会变动的参数进行还原生成后组包进行请求，怎么写代码这种事情相信各位佬随手拈来，我就不在讲解了，直接上图：  
![](https://bbs.kanxue.com/upload/attach/202407/945390_WZ7XZ6KJG9YMC4Y.webp)  
至此，已成艺术。

0x07、总结
-------

#### 本文中所有内容仅供研究与学习使用，请勿用于任何商业用途和非法用途，否则后果自负！

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

[#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm) [#HOOK 注入](forum-161-1-125.htm)