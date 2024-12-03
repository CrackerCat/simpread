> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1988024-1-1.html)

> [md]# 前言 - App ：Fake Location 1.3.5 BETA（http://fakeloc.cc/app） - 工具：jadx、burp、frida、pycharm、ida、雷电模拟......

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 方块君 _ 本帖最后由 方块君 于 2024-12-3 22:18 编辑_  

前言
--

*   App ：Fake Location 1.3.5 BETA（[http://fakeloc.cc/app](http://fakeloc.cc/app)）
*   工具：jadx、burp、frida、pycharm、ida、雷电模拟器、算法助手
*   难点：加壳、代码混淆、双向证书校验、so 层加解密

### app 界面

![](https://attach.52pojie.cn/forum/202412/03/205748wxuoaupx5pf8zb6f.png)

**image.png** _(141.49 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE2NXwzNmUzMGQ0N3wxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:57 上传

#### 开始分析

通过前人的分析已经知道有 SSL 双向认证

将软件脱壳后使用 jadx 分析

![](https://attach.52pojie.cn/forum/202412/03/205752snvclenqgnygyznv.png)

**image-1.png** _(163.39 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE2NnxkMTEyOWFhZnwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:57 上传

搜索关键词找到 SSL Pinning 实现

`translucent.png` 是 客户端证书。

`transparent.png` 是 服务器证书。

![](https://attach.52pojie.cn/forum/202412/03/205757bgth3tcwkjnqq3t1.png)

**image-2.png** _(17.75 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE2N3w4OGQxYTlhM3wxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:57 上传

证书密码是 `lerist.key.2021`

![](https://attach.52pojie.cn/forum/202412/03/205801fpx0ug4jauu4jpu4.png)

**image-3.png** _(107.17 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE2OHxhNmIxOWZhYXwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

打开 Burp 添加客户端证书 (BKS 转 PKCS#12 过程请自行搜索)

![](https://attach.52pojie.cn/forum/202412/03/205806bakz7a1kbn8hkz79.png)

**image-4.png** _(136 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE2OXwwZDA3ZTMzN3wxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

打开算法助手，打开 JustTrustME + 升级版

设置好代 {过}{滤} 理，启动软件，登录账号

#### API 分析

这里只分析主要 API，其他 API 就不分析了

所有信息已脱敏处理

##### /FakeLocation/app/getAppConfigs

获取程序配置

```
{
    "body": {
        "createTime": 1731500000000,
        "disabledApps": [ // 禁用包名
            "com.xxx.xxx.xxx",
        ],
        "disabledFuncs": [ // 禁用功能
            "mock_loc_strong",
            "antidetect_strong",
            "hideroot_strong",
            "antidetect",
            "diypackage",
            "hideroot"
        ],
        "disabledInfos": [
            "bootads",
            "donate",
            "moreapps",
            "recommend",
            "discuss",
            "about"
        ],
        "id": 0,
        "isAllowRun": 1,
        "isAvailable": 1,
        "notice": "",
        "updateTime": 0
    },
    "code": 200,
    "returnTime": 1731500000000, // 服务器时间
    "success": true
}
```

##### /FakeLocation/user/login

登录账号

```
{
    "body": {
        "regtime": 1730000000000, // 注册时间
        "proindate": 1732600000000, // 会员过期时间
        "createTime": 1730000000000, // 注册时间
        "loginType": "email",
        "loginName": "xxxx@xx.com",
        "updateTime": 0,
        "type": 1,
        "userId": "xxxxxxxxxxxxxxxxxxxxxxxxxx", // 用户ID
        "key": "xxxxxxxxxxxxxxxx", // 加密用户信息
        "token": "xxxxxxxxxxxxxxxxxx" // 用户Token
    },
    "code": 200,
    "returnTime": 1731500000000, // 服务器时间
    "success": true
}
```

##### /FakeLocation/user/get

获取用户信息

```
{
    "body": {
        "regtime": 1730000000000, // 注册时间
        "proindate": 1732600000000, // 会员过期时间
        "createTime": 1730000000000, // 注册时间
        "loginType": "email",
        "loginName": "xxxx@xx.com",
        "updateTime": 0,
        "type": 1,
        "userId": "xxxxxxxxxxxxxxxxxxxxxxxxxx", // 用户ID
        "key": "xxxxxxxxxxxxxxxx", // 加密用户信息
        "token": "xxxxxxxxxxxxxxxxxx" // 用户Token
    },
    "code": 200,
    "extras": "", // 用户配置，与 /FakeLocation/app/getAppConfigs 相同
    "returnTime": 1731500000000,
    "success": true
}
```

##### /FakeLocation/cell/queryNearby

获取基站信息

```
{
    "body": [
        {
            "averageSignal": 0,
            "cellid": 0,
            "distance": 0,
            "lac": 0,
            "lat": 0,
            "lon": 0,
            "mcc": 0,
            "mnc": 0,
            "radio_type": "LTE",
            "range": 0,
            "unit": 0
        }
    ],
    "code": 200,
    "returnTime": 1731500000000,
    "success": true
}
```

#### 接下来我们分析用户信息解密以及校验

![](https://attach.52pojie.cn/forum/202412/03/205810dg1g4s5uznq2qvu2.png)

**image-5.png** _(104.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3MHxhMjliYTgwM3wxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

![](https://attach.52pojie.cn/forum/202412/03/205814gssjyy6mwbksvej5.png)

**image-6.png** _(26.01 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3MXxkYWEzMDA1ZHwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

![](https://attach.52pojie.cn/forum/202412/03/205818x4dyyjdm151edybb.png)

**image-7.png** _(48.07 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3MnxkMGZlNzE5YnwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

可以发现，解密走的 Native 层

![](https://attach.52pojie.cn/forum/202412/03/205823d7hj7rhf3r2amqu3.png)

**image-8.png** _(51.62 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3M3w2OTBjZjBjZHwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

函数注册

![](https://attach.52pojie.cn/forum/202412/03/205827i5m50gw5h4cu4o66.png)

**image-9.png** _(77.53 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3NHw1YjdhMTc5OHwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

过签检测

![](https://attach.52pojie.cn/forum/202412/03/205831c6i0v1i6it54osi0.png)

**image-10.png** _(109.14 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3NXxmZDg3NDhmNnwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

签名校验

##### 解密函数分析

传入参数

1.  Key
2.  Token

```
if ( (unsigned int)sq(a1) ) // 签名校验
  {
    v7 = getpid();
    kill(v7, 9);
    kill(0, 9);
    return 0LL;
  }
  v8 = (*a1)->FindClass(a1, "android/util/Base64");
  v9 = (*a1)->GetStaticMethodID(a1, v8, "decode", "([BI)[B");
  v10 = _JNIEnv::CallStaticObjectMethod(a1, v8, v9, a3, 0LL);
  if ( (*a1)->ExceptionCheck(a1) )
  {
    v11 = *a1;
LABEL_6:
    v11->ExceptionClear(a1);
    return 0LL;
  }
  v12 = (*a1)->NewStringUTF(a1, off_4EF8); // 载入公钥
  v13 = (*a1)->FindClass(a1, "java/lang/String");
  v14 = (*a1)->GetMethodID(a1, v13, "getBytes", "()[B");
  v15 = _JNIEnv::CallObjectMethod(a1, v12, v14); // 获取 Bytes
  v16 = _JNIEnv::CallStaticObjectMethod(a1, v8, v9, v15, 0LL); // 调用 Java 层解密 Base64
  v17 = (*a1)->FindClass(a1, "java/security/spec/X509EncodedKeySpec");
  v18 = (*a1)->GetMethodID(a1, v17, "<init>", "([B)V");
  v60 = v17;
  v57 = (void *)v16;
  v19 = _JNIEnv::NewObject(a1, v17, v18, v16);
  v20 = (*a1)->FindClass(a1, "java/security/KeyFactory");
  v21 = (*a1)->GetStaticMethodID(a1, v20, "getInstance", "(Ljava/lang/String;)Ljava/security/KeyFactory;");
  v22 = (*a1)->NewStringUTF(a1, "RSA");
  v23 = (*a1)->NewStringUTF(a1, "RSA/ECB/PKCS1Padding");
  v61 = v22;
  v25 = _JNIEnv::CallStaticObjectMethod(a1, v20, v21, v22, v24);
  v26 = (*a1)->GetMethodID(a1, v20, "generatePublic", "(Ljava/security/spec/KeySpec;)Ljava/security/PublicKey;");
  v56 = (void *)v25;
  v62 = (void *)v19;
  v27 = _JNIEnv::CallObjectMethod(a1, v25, v26); // 加载公钥
  v28 = (*a1)->FindClass(a1, "javax/crypto/Cipher");
  v29 = (*a1)->GetStaticMethodID(a1, v28, "getInstance", "(Ljava/lang/String;)Ljavax/crypto/Cipher;");
  v58 = v23;
  v31 = (void *)_JNIEnv::CallStaticObjectMethod(a1, v28, v29, v23, v30);
  v32 = (*a1)->GetMethodID(a1, v28, "init", "(ILjava/security/Key;)V");
  v59 = (void *)v27;
  _JNIEnv::CallVoidMethod(a1, v31, v32, 2LL, v27);
  v33 = (*a1)->GetMethodID(a1, v28, "doFinal", "([B)[B");
  v34 = _JNIEnv::CallObjectMethod(a1, v31, v33); // 使用公钥解密
  v35 = (*a1)->ExceptionCheck(a1);
  v11 = *a1;
  if ( v35 )
    goto LABEL_6;
  v38 = v11->FindClass(a1, "java/lang/String");
  v39 = (*a1)->GetMethodID(a1, v38, "<init>", "([B)V");
  v54 = (void *)_JNIEnv::NewObject(a1, v38, v39, v34);
  v55 = (void *)v34;
  (*a1)->NewStringUTF(a1, "#");
  v53 = (void *)v10;
  v40 = (*a1)->FindClass(a1, "java/lang/String");
  v41 = (*a1)->GetMethodID(a1, v40, "split", "(Ljava/lang/String;)[Ljava/lang/String;"); // 解密数据使用 # 分割
  v36 = (void *)_JNIEnv::CallObjectMethod(a1, v54, v41);
  v42 = (*a1)->GetArrayLength(a1, v36);
  v43 = (*a1)->GetObjectArrayElement(a1, v36, (unsigned int)(v42 - 1)); // 获取分割后的数据，用户Token
  v44 = (*a1)->GetStringUTFChars(a1, v43, 0LL);
  v45 = (*a1)->GetStringUTFChars(a1, a4, 0LL);
  v46 = strcmp(v44, v45); // 判断传入Token是否与解密Token一致
  if ( v45 && !v46 && *v45 )
  {
    v47 = strstr(v45, "-T");
    v48 = (int)v47 - (int)v45 + 1;
    if ( !v47 )
      v48 = 0LL;
    v49 = v44;
    v50 = v43;
    v51 = 1000 * strtoll(&v45[v48 + 37], 0LL, 16); // 获取用户Token后6为转成时间戳
    v52 = 1000 * time(0LL) < v51; // 判断用户VIP是否过期
    v43 = v50;
    v44 = v49;
    if ( v52 )
    {
      return v36;
    }
  }
  return 0LL;
```

通过上面分析，用户信息是使用 RSA 公钥解密

然后判断传入 Token 和解密 Token 是否一致

以及判断 Token 内隐藏的 VIP 时间是否过期

### 虚拟定位内核分析

#### 注入相关

释放资源目录下的 so 和 `3DFly.lis` 到 `/data/fl`

通过调用释放到 `/data/fl/inject` 将 `/data/fl/libfl_app.so` 和 `/data/fl/libfl_init.so` 注入到 `system_service` 和 `com.android.phone` 中

校验 `libfl.so`(`3DFly.lis`) 的 md5 是否为 `f2fe0b7e56eb6f32d277224eb37a20e5`

`libfl_init.so` 将 `libfl.so`(`3DFly.lis`) 注入并调用 `o.O.o.O#init(Ljava/lang/Object;)[Ljava/lang/Object;`

`libfl_app.so` 将 `libfl.so`(`3DFly.lis`) 注入并调用 `o.O.o.O#hookApp(Ljava/lang/Object;)[Ljava/lang/Object;`

具体如何注入的未分析

#### 用户信息校验

![](https://attach.52pojie.cn/forum/202412/03/205835j5rb6orir8e5gg5p.png)

**image-11.png** _(98.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3NnxhMzY3Mjc3MHwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

软件过期时间 `2025-09-26 08:07:28`

过期时间检查

##### 用户信息构成

<table><thead><tr><th>Index</th><th>信息</th></tr></thead><tbody><tr><td>0</td><td>VIP 类型</td></tr><tr><td>1</td><td>VIP 过期时间</td></tr><tr><td>2</td><td>Token 过期时间</td></tr><tr><td>3</td><td>Token</td></tr></tbody></table>

##### VIP 类型

<table><thead><tr><th>Type</th><th>信息</th></tr></thead><tbody><tr><td>0</td><td>非 VIP</td></tr><tr><td>1</td><td>VIP</td></tr><tr><td>100</td><td>长时间 VIP (大于 4 年)</td></tr><tr><td>200</td><td>长时间 Token (大于 60 天)</td></tr></tbody></table>

#### Hook 方法

```
// com.android.phone
com.android.phone.PhoneInterfaceManager#getAllCellInfo
com.android.phone.PhoneInterfaceManager#getCellLocation
com.android.phone.PhoneInterfaceManager#getAllCellInfo
com.android.phone.PhoneInterfaceManager#getCellLocation
com.android.phone.PhoneInterfaceManager#getNeighboringCellInfo
com.android.phone.PhoneInterfaceManager#getAllCellInfo
com.android.phone.PhoneInterfaceManager#getCellLocation // sdk >= 28
com.android.phone.PhoneInterfaceManager#getNeighboringCellInfo
com.android.phone.PhoneInterfaceManager#getActivePhoneTypeForSlot
com.android.phone.PhoneInterfaceManager#getActivePhoneType
com.android.phone.PhoneInterfaceManager#getNetworkType
com.android.phone.PhoneInterfaceManager#getDataNetworkType
com.android.phone.PhoneInterfaceManager#getNetworkTypeForSubscriber
com.android.phone.PhoneInterfaceManager#getDataNetworkTypeForSubscriber
com.android.phone.PhoneInterfaceManager#getNetworkTypeForSubscriber
com.android.phone.PhoneInterfaceManager#getDataNetworkTypeForSubscriber

// android
com.android.server.TelephonyRegistry#listen
com.android.server.TelephonyRegistry#listenForSubscriber
com.android.server.TelephonyRegistry#listenForSubscriber
com.android.server.TelephonyRegistry#listenWithEventList

// wifi
com.android.server.wifi.WifiServiceImplget#ScanResults

// gps
com.android.server.location.LocationManagerService // sdk >= 30
com.android.server.LocationManagerService // sdk < 30

#unregisterGnssStatusCallback

// sdk >= 30
#isProviderEnabled
#isProviderEnabledForUser
#getBestProvider
#getProviders

// sdk < 31
#getLastLocation
#requestLocationUpdatesLocked
#removeUpdatesLocked
#callStatusChangedLocked
#callLocationChangedLocked
#registerGnssStatusCallback
#addGpsStatusListener
#removeGpsStatusListener

// sdk >= 31
#getLastLocation
#registerLocationListener
#unregisterLocationListener
#registerLocationPendingIntent
#unregisterLocationPendingIntent
#registerGnssStatusCallback
android.location.ILocationListener$Stub$Proxy#onLocationChanged
android.location.LocationManager$LocationListenerTransport#onLocationChanged
```

#### 内核内置黑名单

![](https://attach.52pojie.cn/forum/202412/03/205839cgm4a99o19985c4d.png)

**image-14.png** _(58.23 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3N3xmNzdlNjZhOHwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

### 软件后门

![](https://attach.52pojie.cn/forum/202412/03/205843x87nzt7tprs97mm7.png)

**image-12.png** _(39.27 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3OHw2MTllYTQzOXwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

连接到服务器校验用户 Token

##### AES 密钥

*   Key: hd7x809H$l1OI863
*   IV : IUdH0kG1kDTgLkPl

##### 验证回调

![](https://attach.52pojie.cn/forum/202412/03/205848ci453yph32sh11sa.png)

**image-13.png** _(44 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE3OXwwZWExYmNjNnwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

验证通过服务器返回 `pass.`

验证失败返回 `NOPASS.` 并停止模拟

通过分析能看到，允许服务器发送 Shell 命令以 `system_service` 权限运行命令

这是一个非常危险的操作，使用 socket 连接到服务器，并且没有任何校验，还能以 `system_service` 权限运行命令

可以在用户不知情的情况下执行任意命令

#### 关于破解

*   `libfl_app.so`
*   `libfl_init.so`
*   `liblh.so`
*   `libStepSensor.so`

以上的 so 都需要去除签名校验和文件校验

如有破解需求可以使用国际版，国际版没有加壳

![](https://attach.52pojie.cn/forum/202412/03/205852ip624v4kz5v46j2p.png)

**image-15.png** _(140.03 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE4MHw3MDBhNGUzYXwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

自己破了一个，但是开始模拟会导致系统重启，没有崩溃日志，应该是有地方改坏了或者还有校验未去除

![](https://attach.52pojie.cn/forum/202412/03/205858i1o221362dd3ncv0.png)

**image-16.png** _(703.43 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc0MTE4MXxlMDA3NzJjMXwxNzMzMjM2NTA3fDIxMzQzMXwxOTg4MDI0&nothumb=yes)

2024-12-3 20:58 上传

这东西特征太过于明显，已经没什么作用了，打算自己写一个来用了，就没继续研究了 ![](https://avatar.52pojie.cn/data/avatar/000/39/40/03_avatar_middle.jpg) 大佬牛逼，打算自写自用，自己动手丰衣足食呀！！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)文西思密达 有了这个东西，就可以虚拟定位，DD 刷脸打卡的问题就解决了