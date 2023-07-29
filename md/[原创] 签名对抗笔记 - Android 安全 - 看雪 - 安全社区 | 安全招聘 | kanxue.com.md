> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278195.htm)

> [原创] 签名对抗笔记

[原创] 签名对抗笔记

10 小时前 506

### [原创] 签名对抗笔记

 [![](http://passport.kanxue.com/upload/avatar/902/950902.png?1686620741)](user-home-950902.htm) [简单的简单](user-home-950902.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 10 小时前  506

写在前面
====

本篇笔记记录了笔者签名对抗的学习路线，共分为 PMS 和 IO 重定向两个主要部分

前置知识
====

解包打包工具
------

### apktool

dex 反编译 smali、xml 解析、生成资源序号文件与资源名称对应表

```
apktool.jar -r d HelloWorld.apk


```

smali 文件编译为应用，应用生成目录为反编译应用的 dist 目录中

```
apktool.jar b HelloWorld


```

签名工具
----

### apksigner

查看目标应用是否签名，及使用的签名版本

```
java -jar apksigner.jar verify -v app-debug.apk
-------------------------
Verifies
Verified using v1 scheme (JAR signing): false
Verified using v2 scheme (APK Signature Scheme v2): true
Verified using v3 scheme (APK Signature Scheme v3): false
Verified using v4 scheme (APK Signature Scheme v4): false
Verified for SourceStamp: false
Number of signers: 1

```

### zipalign

查看目标应用是否对齐

```
zipalign -c -v 4 app-debug.apk
----------------
Verification succesful

```

制作对齐的应用

```
zipalign -v 4 app-debug.apk zipalign.apk

```

### keytool

制作签名，会生成一个 demo.keystore 签名文件

```
keytool -genkey -alias demo.keystore -keyalg RSA -validity 40000 -keystore demo.keystore

```

使用生成的签名文件对目标应用进行签名

```
java -jar apksigner.jar sign --ks demo.keystore --ks-key-alias demo.keystore --ks-pass pass:123456 --key-pass pass:123456 --out sign.apk app-debug.apk

```

### AndroidStudio 中配置签名

在 build.gradle(app) 中添加 signingConfigs 字段，另外注意 buildTypes 中 signingConfig 应分别为 signingConfigs.debug 和 signingConfigs.release

```
signingConfigs {
    release {
        storeFile file('C:\\Users\\lxz\\Documents\\mytools\\sign\\demo.keystore.jks')
        storePassword '123456'
        keyAlias 'demo.keystore'
        keyPassword '123456'
    }
}

```

GetSignature
============

首先我们需要制作一个可以获取其他 app 签名的工具，这里直接写一个可以获取其他应用 signature 的 app，需要注意的就是权限配置

```
package com.example.myapplication;
 
import androidx.appcompat.app.AppCompatActivity;
import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.content.pm.SigningInfo;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.TextView;
import com.example.myapplication.databinding.ActivityMainBinding;
 
import static android.content.pm.PackageManager.PERMISSION_GRANTED;
 
 
public class MainActivity extends AppCompatActivity {
    private ActivityMainBinding binding;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
 
        TextView textView = findViewById(R.id.mytext);
         
        String str = getSignatuer(this,"com.example.myapplication4");
 
        Log.e("lxz" , "获取到目标签名：" + str);
 
    }
 
    private String getSignatuer(Context context, String packagename){
        try {
 
            PackageInfo info = context.getPackageManager().getPackageInfo(packagename, PackageManager.GET_SIGNATURES);
            if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.P) {
                SigningInfo signingInfo = info.signingInfo;
            }else{
                Log.e("lxz","未获取到签名信息");
            }
            byte[] bytes = info.signatures[0].toByteArray();
            String str = bytesToHex(bytes);
            return str;
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(e);
        }
    }
 
    // 将 byte[] 转换为 16 进制字符串
    public static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            String hex = Integer.toHexString(0xFF & b);
            if (hex.length() == 1) {
                hexString.append('0');
            }
            hexString.append(hex);
        }
        return hexString.toString();
    }
 
    // 将 16 进制字符串转换为 byte[]
    public static byte[] hexToBytes(String hexString) {
        int length = hexString.length();
        if (length % 2 != 0) {
            throw new IllegalArgumentException("Invalid hex string length");
        }
        byte[] bytes = new byte[length / 2];
        for (int i = 0; i < length; i += 2) {
            String hex = hexString.substring(i, i + 2);
            bytes[i / 2] = (byte) Integer.parseInt(hex, 16);
        }
        return bytes;
    }
 
}

```

注意权限配置

CheckSignature
==============

然后我们还需要写一个目标 app 作为实验对象

```
package com.example.checksignatuer;
 
import androidx.appcompat.app.AppCompatActivity;
 
import android.content.Context;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.content.pm.SigningInfo;
import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;
 
public class MainActivity extends AppCompatActivity {
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        TextView textView = findViewById(R.id.mytext);
 
        String str = getSignatuer(this);
        if(checkSignatuer(str))
        {
            Log.d("lxz","签名正确");
            textView.setText("签名正确");
        }else{
            Log.d("lxz","签名错误");
            textView.setText("签名错误");
        }
    }
 
    private boolean checkSignatuer(String str)
    {
        // 这里的 signature_str 可以使用 GetSignature 获取
        String signature_str = "308202e4308201cc020101300d06092a864886f70d01010b050030373116301406035504030c0d416e64726f69642044656275673110300e060355040a0c07416e64726f6964310b30090603550406130255533020170d3233303430363038313332325a180f32303533303332393038313332325a30373116301406035504030c0d416e64726f69642044656275673110300e060355040a0c07416e64726f6964310b300906035504061302555330820122300d06092a864886f70d01010105000382010f003082010a02820101009e80d9990b9622e00c04c4608d404c89ab3b5f28ea1ababfe2c9e4d172d3004c1a0c4494fb3359dc0071024738d942ae7b2db78a5f5898261ef1e49a353651facc6c7aca2a7d2ace9bfb158f63a2177a7c7d7dd18e0f57ca59dd56d325d14b6a697257f20e9f923363ac7724ac4e6c2600648fd81f8dc85db648a8d8e06947d15c8c89bbb469e86c685bb6165ec936264c8af54f04f86794887371c564d6340c838e4821dd3e72824dfcc0246efab42241324887de2fbe043113d4544e6e6c3c6b42a77ad8f50f91f8c43fc2e2cf4c3de689995326fedbfd8f86606ec9ffae1e5e30bc4f5296fd8d30ac0117cd66efba7292ea1377a56f9890fa9948ec9b57010203010001300d06092a864886f70d01010b05000382010100754cf3804666464192d206c9f76354b91e88d664fd553f0dceababf007d26eb208424fb92829dc029b0b291ccab55b642bdc0c8b9346089552970460918306262218bf6c776eb803019928568a53ed97a970003c570e20fd472170e0057203ed7ed8118fb944f984e78536b020f44558e518f46f70132216d4162d9363a2ccc62a53083b7e3a3df4b8da03bc7cd8b06b8d193b2b33aa20acc66c9f53cb0c2bd2eeb691285e8cc3eeb897c160ddf916f556a4606527e9e1b9d3856000193af4641dbc951660cf018621267486ac7b4b0d57c7e64e9880ce5ae4e70cc37cb72bdc5f2fef1b50004872844a23f8706b226349583316e3ed7770eb662bd3567ffb96";
 
        if(str.equals(signature_str))
        {
            return true;
        }else{
            return false;
        }
    }
 
    private String getSignatuer(Context context){
        try {
            PackageInfo info = context.getPackageManager().getPackageInfo(context.getPackageName(), PackageManager.GET_SIGNATURES);
            if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.P) {
                SigningInfo signingInfo = info.signingInfo;
            }else{
                Log.e("lxz","未获取到签名信息");
            }
            byte[] bytes = info.signatures[0].toByteArray();
            String str = bytesToHex(bytes);
            return str;
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(e);
        }
    }
 
    // 将 byte[] 转换为 16 进制字符串
    public static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            String hex = Integer.toHexString(0xFF & b);
            if (hex.length() == 1) {
                hexString.append('0');
            }
            hexString.append(hex);
        }
        return hexString.toString();
    }
 
    // 将 16 进制字符串转换为 byte[]
    public static byte[] hexToBytes(String hexString) {
        int length = hexString.length();
        if (length % 2 != 0) {
            throw new IllegalArgumentException("Invalid hex string length");
        }
        byte[] bytes = new byte[length / 2];
        for (int i = 0; i < length; i += 2) {
            String hex = hexString.substring(i, i + 2);
            bytes[i / 2] = (byte) Integer.parseInt(hex, 16);
        }
        return bytes;
    }
}

```

PMS 签名
======

PMS 是 PackageManagerSignature 的缩写，其原理是将目标解包后添加一个父类，在父类中重写 PackageManager 中的 getPackageManager

JAVA 层实现
--------

Java 层实现的核心代码如下

**java/com/lxz/hookactivity.java**

```
package com.lxz;
 
import android.content.ComponentName;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.ActivityInfo;
import android.content.pm.ApplicationInfo;
import android.content.pm.ChangedPackages;
import android.content.pm.FeatureInfo;
import android.content.pm.InstrumentationInfo;
import android.content.pm.PackageInfo;
import android.content.pm.PackageInstaller;
import android.content.pm.PackageManager;
import android.content.pm.PermissionGroupInfo;
import android.content.pm.PermissionInfo;
import android.content.pm.ProviderInfo;
import android.content.pm.ResolveInfo;
import android.content.pm.ServiceInfo;
import android.content.pm.SharedLibraryInfo;
import android.content.pm.Signature;
import android.content.pm.VersionedPackage;
import android.content.res.Resources;
import android.content.res.XmlResourceParser;
import android.graphics.Rect;
import android.graphics.drawable.Drawable;
import android.os.Build;
import android.os.UserHandle;
import android.util.Log;
 
import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
 
import java.util.List;
 
public class hook extends PackageManager {
    private PackageManager mBase;
    private String mPackageName;
 
    // 此处的签名使用 GetSignature 获取原包的签名信息
    String signature_str = "3082037130820259a003020102020419f0a927300d06092a864886f70d01010b05003068310b300906035504061302434e310f300d0603550408130646754a69616e310f300d0603550407130646755a686f753111300f060355040a13084c69616e596f6e673111300f060355040b13084c69616e596f6e673111300f060355040313084c6f756973204c753020170d3138303632393037323931365a180f32303733303430313037323931365a3068310b300906035504061302434e310f300d0603550408130646754a69616e310f300d0603550407130646755a686f753111300f060355040a13084c69616e596f6e673111300f060355040b13084c69616e596f6e673111300f060355040313084c6f756973204c7530820122300d06092a864886f70d01010105000382010f003082010a0282010100a55431f01fb453e65d7e070bc82e606c11e9bf77831367701e4d7c79a44c014afa2be320caade2e75f8d9160ecaa6e5a39ca63d8ee5ddaffe54f6da0d1f7ea24efa9591681fa39561780c2bef75ec72096e121524da2f9c84d9455593639e63ca41cdbf7a349e0e26cf5f27564825fa524eb3efdbac5ec62f851053cc833537182e6d24dffaaf50274e6062650d527d76856e188d144116731689881a05db10d8bb159bbef9cfd314b205c785e51d4a34e0db54fa89b7ddb837559338f1f58e38df78f7e5acceeaf94546c78d1b8eea3a25e095fc1c959e77860962b3e980e31b63c3089e3541e27cea1631c3b2c59bcfd4c7384123c778c599473a3a319b2270203010001a321301f301d0603551d0e04160414fc7d539fc8aea2e08ade9cd07c47f43621f3b209300d06092a864886f70d01010b05000382010100088fe8de887969eb896e5d9c31aead82bc348faff1917fb224018a38f6d0126e0a9af191bf84ca84cf530cbe2ba0a4993059ae89ce2a05266a8192b044b4a8e18e510a8c6b7e022bebe5482b09a7b80f47661adf9f53fd65b69cf3acb2efc69b89bac3e90eabf1e7ed719b0efd38159cb5f3fea51ee62307ca5f09cbb85660323a41597438aba3999d3626fcbfa628d5510a435c78a82482d1447c3cff3ea19a7ae87d347b7e5e7b0237029cd2e5e57baa907bde58cb5483ef54dcee05fabaadb1a46c88113d85333c6979d846490d3e8def5aa4c3d94d2fc6497bea36901e6a18e1000db8da1f4bb31421138fbf2cb3a22d0458ef28e7167458278a79bf67ef";
 
    public hook(PackageManager base, String packageName) {
        this.mPackageName = packageName;
        this.mBase = base;
    }
 
    @Override // android.content.pm.PackageManager
    public PackageInfo getPackageInfo(String packageName, int flags) throws PackageManager.NameNotFoundException {
 
        if (!mPackageName.equals(packageName)) {
            // 不是自己在检查 PackageInfo 信息，返回真实信息
            return this.mBase.getPackageInfo(packageName, flags);
        }
        if (flags != PackageManager.GET_SIGNATURES) {
            // 检查的不是 signature, 返回真实信息
            return this.mBase.getPackageInfo(packageName, flags);
        }
 
        PackageInfo pkgInfo = new PackageInfo();
        byte bytes[] = hexToBytes(signature_str);
        pkgInfo.signatures = new Signature[]{new Signature(bytes)};
        Log.e("lxz","强制返回正确签名");
        return pkgInfo;
    }
 
    // 将 byte[] 转换为 16 进制字符串
    public static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            String hex = Integer.toHexString(0xFF & b);
            if (hex.length() == 1) {
                hexString.append('0');
            }
            hexString.append(hex);
        }
        return hexString.toString();
    }
 
    // 将 16 进制字符串转换为 byte[]
    public static byte[] hexToBytes(String hexString) {
        int length = hexString.length();
        if (length % 2 != 0) {
            throw new IllegalArgumentException("Invalid hex string length");
        }
        byte[] bytes = new byte[length / 2];
        for (int i = 0; i < length; i += 2) {
            String hex = hexString.substring(i, i + 2);
            bytes[i / 2] = (byte) Integer.parseInt(hex, 16);
        }
        return bytes;
    }
 
    @Override
    public PackageInfo getPackageInfo(@NonNull VersionedPackage versionedPackage, int flags) throws NameNotFoundException {
        // 这里好像也可以返回 PackageInfo，但我没做处理
        return this.mBase.getPackageInfo(versionedPackage, flags);
    }
    @Override
    public List getInstalledPackages(int flags) {
        return this.mBase.getInstalledPackages(flags);
    }
    @Override
    public String[] currentToCanonicalPackageNames(String[] names) {
        return this.mBase.currentToCanonicalPackageNames(names);
    }
    @Override
    public String[] canonicalToCurrentPackageNames(String[] names) {
        return this.mBase.canonicalToCurrentPackageNames(names);
    }
    @Override
    public Intent getLaunchIntentForPackage(String packageName) {
        return this.mBase.getLaunchIntentForPackage(packageName);
    }
    @Override
    public Intent getLeanbackLaunchIntentForPackage(String packageName) {
        return this.mBase.getLeanbackLaunchIntentForPackage(packageName);
    }
    @Override
    public int[] getPackageGids(String packageName) throws PackageManager.NameNotFoundException {
        return this.mBase.getPackageGids(packageName);
    }
    @Override
    public int[] getPackageGids(@NonNull String packageName, int flags) throws NameNotFoundException {
        return new int[0];
    }
    @Override
    public int getPackageUid(@NonNull String packageName, int flags) throws NameNotFoundException {
        return 0;
    }
    @Override
    public PermissionInfo getPermissionInfo(String name, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getPermissionInfo(name, flags);
    }
    @Override
    public List queryPermissionsByGroup(String group, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.queryPermissionsByGroup(group, flags);
    }
    @Override
    public PermissionGroupInfo getPermissionGroupInfo(String name, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getPermissionGroupInfo(name, flags);
    }
    @Override
    public List getAllPermissionGroups(int flags) {
        return this.mBase.getAllPermissionGroups(flags);
    }
    @Override
    public ApplicationInfo getApplicationInfo(String packageName, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getApplicationInfo(packageName, flags);
    }
    @Override
    public ActivityInfo getActivityInfo(ComponentName component, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityInfo(component, flags);
    }
    @Override
    public ActivityInfo getReceiverInfo(ComponentName component, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getReceiverInfo(component, flags);
    }
    @Override
    public ServiceInfo getServiceInfo(ComponentName component, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getServiceInfo(component, flags);
    }
    @Override
    public ProviderInfo getProviderInfo(ComponentName component, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getProviderInfo(component, flags);
    }
    @Override
    public List getPackagesHoldingPermissions(String[] permissions, int flags) {
        return this.mBase.getPackagesHoldingPermissions(permissions, flags);
    }
    @Override
    public int checkPermission(String permName, String pkgName) {
        return this.mBase.checkPermission(permName, pkgName);
    }
    @Override
    public boolean isPermissionRevokedByPolicy(String permName, String pkgName) {
        return this.mBase.isPermissionRevokedByPolicy(permName, pkgName);
    }
    @Override
    public boolean addPermission(PermissionInfo info) {
        return this.mBase.addPermission(info);
    }
    @Override
    public boolean addPermissionAsync(PermissionInfo info) {
        return this.mBase.addPermissionAsync(info);
    }
    @Override
    public void removePermission(String name) {
        this.mBase.removePermission(name);
    }
    @Override
    public int checkSignatures(String pkg1, String pkg2) {
        return this.mBase.checkSignatures(pkg1, pkg2);
    }
    @Override
    public int checkSignatures(int uid1, int uid2) {
        return this.mBase.checkSignatures(uid1, uid2);
    }
    @Override
    public String[] getPackagesForUid(int uid) {
        return this.mBase.getPackagesForUid(uid);
    }
    @Override
    public String getNameForUid(int uid) {
        return this.mBase.getNameForUid(uid);
    }
    @Override
    public List getInstalledApplications(int flags) {
        return this.mBase.getInstalledApplications(flags);
    }
    @Override
    public boolean isInstantApp() {
        return false;
    }
    @Override
    public boolean isInstantApp(@NonNull String packageName) {
        return false;
    }
    @Override
    public int getInstantAppCookieMaxBytes() {
        return 0;
    }
    @NonNull
    @Override
    public byte[] getInstantAppCookie() {
        return new byte[0];
    }
    @Override
    public void clearInstantAppCookie() {
 
    }
    @Override
    public void updateInstantAppCookie(@Nullable byte[] cookie) {
 
    }
    @Override
    public String[] getSystemSharedLibraryNames() {
        return this.mBase.getSystemSharedLibraryNames();
    }
    @NonNull
    @Override
    public List getSharedLibraries(int flags) {
        return null;
    }
 
    @Nullable
    @Override
    public ChangedPackages getChangedPackages(int sequenceNumber) {
        return null;
    }
    @Override
    public FeatureInfo[] getSystemAvailableFeatures() {
        return this.mBase.getSystemAvailableFeatures();
    }
    @Override
    public boolean hasSystemFeature(String name) {
        return this.mBase.hasSystemFeature(name);
    }
 
    @Override
    public boolean hasSystemFeature(@NonNull String featureName, int version) {
        return false;
    }
    @Override
    public ResolveInfo resolveActivity(Intent intent, int flags) {
        return this.mBase.resolveActivity(intent, flags);
    }
    @Override
    public List queryIntentActivities(Intent intent, int flags) {
        return this.mBase.queryIntentActivities(intent, flags);
    }
    @Override
    public List queryIntentActivityOptions(ComponentName caller, Intent[] specifics, Intent intent, int flags) {
        return this.mBase.queryIntentActivityOptions(caller, specifics, intent, flags);
    }
    @Override
    public List queryBroadcastReceivers(Intent intent, int flags) {
        return this.mBase.queryBroadcastReceivers(intent, flags);
    }
    @Override
    public ResolveInfo resolveService(Intent intent, int flags) {
        return this.mBase.resolveService(intent, flags);
    }
    @Override
    public List queryIntentServices(Intent intent, int flags) {
        return this.mBase.queryIntentServices(intent, flags);
    }
    @Override
    public List queryIntentContentProviders(Intent intent, int flags) {
        return this.mBase.queryIntentContentProviders(intent, flags);
    }
    @Override
    public ProviderInfo resolveContentProvider(String name, int flags) {
        return this.mBase.resolveContentProvider(name, flags);
    }
    @Override
    public List queryContentProviders(String processName, int uid, int flags) {
        return this.mBase.queryContentProviders(processName, uid, flags);
    }
    @Override
    public InstrumentationInfo getInstrumentationInfo(ComponentName className, int flags) throws PackageManager.NameNotFoundException {
        return this.mBase.getInstrumentationInfo(className, flags);
    }
    @Override
    public List queryInstrumentation(String targetPackage, int flags) {
        return this.mBase.queryInstrumentation(targetPackage, flags);
    }
    @Override
    public Drawable getDrawable(String packageName, int resid, ApplicationInfo appInfo) {
        return this.mBase.getDrawable(packageName, resid, appInfo);
    }
    @Override
    public Drawable getActivityIcon(ComponentName activityName) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityIcon(activityName);
    }
    @Override
    public Drawable getActivityIcon(Intent intent) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityIcon(intent);
    }
    @Override
    public Drawable getActivityBanner(ComponentName activityName) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityBanner(activityName);
    }
    @Override
    public Drawable getActivityBanner(Intent intent) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityBanner(intent);
    }
    @Override
    public Drawable getDefaultActivityIcon() {
        return this.mBase.getDefaultActivityIcon();
    }
    @Override
    public Drawable getApplicationIcon(ApplicationInfo info) {
        return this.mBase.getApplicationIcon(info);
    }
    @Override
    public Drawable getApplicationIcon(String packageName) throws PackageManager.NameNotFoundException {
        return this.mBase.getApplicationIcon(packageName);
    }
    @Override
    public Drawable getApplicationBanner(ApplicationInfo info) {
        return this.mBase.getApplicationBanner(info);
    }
    @Override
    public Drawable getApplicationBanner(String packageName) throws PackageManager.NameNotFoundException {
        return this.mBase.getApplicationBanner(packageName);
    }
    @Override
    public Drawable getActivityLogo(ComponentName activityName) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityLogo(activityName);
    }
    @Override
    public Drawable getActivityLogo(Intent intent) throws PackageManager.NameNotFoundException {
        return this.mBase.getActivityLogo(intent);
    }
    @Override
    public Drawable getApplicationLogo(ApplicationInfo info) {
        return this.mBase.getApplicationLogo(info);
    }
    @Override
    public Drawable getApplicationLogo(String packageName) throws PackageManager.NameNotFoundException {
        return this.mBase.getApplicationLogo(packageName);
    }
    @Override
    public Drawable getUserBadgedIcon(Drawable icon, UserHandle user) {
        return this.mBase.getUserBadgedIcon(icon, user);
    }
    @Override
    public Drawable getUserBadgedDrawableForDensity(Drawable drawable, UserHandle user, Rect badgeLocation, int badgeDensity) {
        return this.mBase.getUserBadgedDrawableForDensity(drawable, user, badgeLocation, badgeDensity);
    }
    @Override
    public CharSequence getUserBadgedLabel(CharSequence label, UserHandle user) {
        return this.mBase.getUserBadgedLabel(label, user);
    }
    @Override
    public CharSequence getText(String packageName, int resid, ApplicationInfo appInfo) {
        return this.mBase.getText(packageName, resid, appInfo);
    }
    @Override
    public XmlResourceParser getXml(String packageName, int resid, ApplicationInfo appInfo) {
        return this.mBase.getXml(packageName, resid, appInfo);
    }
    @Override
    public CharSequence getApplicationLabel(ApplicationInfo info) {
        return this.mBase.getApplicationLabel(info);
    }
    @Override
    public Resources getResourcesForActivity(ComponentName activityName) throws PackageManager.NameNotFoundException {
        return this.mBase.getResourcesForActivity(activityName);
    }
    @Override
    public Resources getResourcesForApplication(ApplicationInfo app) throws PackageManager.NameNotFoundException {
        return this.mBase.getResourcesForApplication(app);
    }
    @Override
    public Resources getResourcesForApplication(String appPackageName) throws PackageManager.NameNotFoundException {
        return this.mBase.getResourcesForApplication(appPackageName);
    }
    @Override
    public void verifyPendingInstall(int id, int verificationCode) {
        this.mBase.verifyPendingInstall(id, verificationCode);
    }
    @Override
    public void extendVerificationTimeout(int id, int verificationCodeAtTimeout, long millisecondsToDelay) {
        this.mBase.extendVerificationTimeout(id, verificationCodeAtTimeout, millisecondsToDelay);
    }
    @Override
    public void setInstallerPackageName(String targetPackage, String installerPackageName) {
        this.mBase.setInstallerPackageName(targetPackage, installerPackageName);
    }
    @Override
    public String getInstallerPackageName(String packageName) {
        return this.mBase.getInstallerPackageName(packageName);
    }
    @Override
    public void addPackageToPreferred(String packageName) {
        this.mBase.addPackageToPreferred(packageName);
    }
    @Override
    public void removePackageFromPreferred(String packageName) {
        this.mBase.removePackageFromPreferred(packageName);
    }
    @Override
    public List getPreferredPackages(int flags) {
        return this.mBase.getPreferredPackages(flags);
    }
    @Override
    public void addPreferredActivity(IntentFilter filter, int match, ComponentName[] set, ComponentName activity) {
        this.mBase.addPreferredActivity(filter, match, set, activity);
    }
    @Override
    public void clearPackagePreferredActivities(String packageName) {
        this.mBase.clearPackagePreferredActivities(packageName);
    }
    @Override
    public int getPreferredActivities(List outFilters, List outActivities, String packageName) {
        return this.mBase.getPreferredActivities(outFilters, outActivities, packageName);
    }
    @Override
    public void setComponentEnabledSetting(ComponentName componentName, int newState, int flags) {
        this.mBase.setComponentEnabledSetting(componentName, newState, flags);
    }
    @Override
    public int getComponentEnabledSetting(ComponentName componentName) {
        return this.mBase.getComponentEnabledSetting(componentName);
    }
    @Override
    public void setApplicationEnabledSetting(String packageName, int newState, int flags) {
        this.mBase.setApplicationEnabledSetting(packageName, newState, flags);
    }
    @Override
    public int getApplicationEnabledSetting(String packageName) {
        return this.mBase.getApplicationEnabledSetting(packageName);
    }
    @Override
    public boolean isSafeMode() {
        return this.mBase.isSafeMode();
    }
    @Override
    public void setApplicationCategoryHint(@NonNull String packageName, int categoryHint) {
 
    }
    @Override
    public PackageInstaller getPackageInstaller() {
        return this.mBase.getPackageInstaller();
    }
    @Override
    public boolean canRequestPackageInstalls() {
        return false;
    }
} 
```

**java/com/lxz/hookactivity.java**

```
package com.lxz;
 
import android.app.Activity;
import android.content.pm.PackageManager;
import androidx.appcompat.app.AppCompatActivity;
 
// 注意这里继承的 AppCompatActivity 类是目标应用 activity 继承的类
public class hookactivity extends AppCompatActivity {
    @Override // android.content.ContextWrapper, android.content.Context
    public PackageManager getPackageManager() {
        PackageManager pm = new hook(super.getPackageManager());
        return pm;
    }
}

```

修改目标主启动类为 hookactivity，其实就是多了一层继承，中间修改了 PMS

```
public class MainActivity extends hookactivity {
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        TextView textView = findViewById(R.id.mytext);
 
        String str = getSignatuer(this);
        if(checkSignatuer(str))
        {
            Log.d("lxz","签名正确");
            textView.setText("签名正确");
        }else{
            Log.d("lxz","签名错误");
            textView.setText("签名错误");
        }
    }
    ......
}

```

smail 层实现
---------

使用反编译工具将刚刚用 java 实现的 PMS 反编译，并取出其中的 lxz 文件夹

```
apktool.jar -r d .\app-debug.apk

```

再将 CheckSignature 中 刚刚添加的 psm 部分删掉，让其还原成一个可以检查 Signature 的应用，重新打包成 apk，再使用 apktool 将其解包

```
apktool.jar -r d .\app-debug.apk

```

将 lxz 文件夹拷贝到 smali_classes3/com 目录下，并修改 smali_classes3\com\example\checksignatuer\MainActivity.smali 的父类

androidx.multidex.MultiDexApplication

com\babybus\app\App.smali

```
.class public Lcom/example/checksignatuer/MainActivity;
#.super Landroidx/activity/ComponentActivity;
.super Lcom/lxz/hookactivity;
.source "MainActivity.java"

```

重新打包回 apk 文件，生成 signature.apk

```
apktool.jar b app-debug -o signature.apk

```

使用 zipalign 对生成的文件对齐，需要制作 v2 的签名，不然 android 11 及以后安装会失败

```
zipalign -v 4 signature.apk zipalign.apk

```

接下来我们需要进行签名，但我们现在还没有签名文件，所以还得做个签名文件，密码 123456，问题答案随便填，这样我们就获得了签名文件 demo.keystore

```
keytool -genkey -alias demo.keystore -keyalg RSA -validity 40000 -keystore demo.keystore

```

使用生成的签名文件对目标应用进行签名，此时安装并启动 app 会发现强制返回正确签名的 log

```
apksigner.jar sign --ks demo.keystore --ks-key-alias demo.keystore --ks-pass pass:123456 --key-pass pass:123456 --out signature_final.apk zipalign.apk

```

IO 重定向
======

简单介绍下 io 重定向，编写一个自定义的文件打开函数来替代、修改标准的文件打开函数。在这个自定义函数中，可以实现 IO 重定向的逻辑，比如将文件的读写重定向到其他文件，说白了就是把 open 函数中的路径参数换成我们想要的路径

CheckSignature
--------------

老套路还是先制作一个通过 base.apk 检查 signature 的 app，相较于之前我们只需要改下 getSignatuer 这个函数，另外最好新建一个 带 native 的工程会方便些

```
private String getSignatuer(Context context) {
    try (ZipFile zipFile = new ZipFile(getPackageResourcePath())) {
        Enumeration entries = zipFile.entries();
        // 遍历ZIP文件中的每个条目
        while (entries.hasMoreElements()) {
            ZipEntry entry = entries.nextElement();
            // 检查条目名称是否匹配模式"META-INF/*.RSA"、"META-INF/*.DSA"或"META-INF/*.EC"
            if (entry.getName().matches("(META-INF/.*)\\.(RSA|DSA|EC)")) {
                // 从ZIP文件中获取当前条目的输入流
                InputStream is = zipFile.getInputStream(entry);
                // 创建一个CertificateFactory实例，使用"X509"证书类型
                CertificateFactory certFactory = CertificateFactory.getInstance("X509");
                // 通过输入流生成一个X509Certificate对象，表示证书
                X509Certificate x509Cert = (X509Certificate) certFactory.generateCertificate(is);
                // 将证书的编码字节转换为十六进制表示
                return bytesToHex(x509Cert.getEncoded());
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    // 如果找不到匹配的签名，返回空字符串
    return "";
}

```

Hook Open
---------

此时竟有些不知如何提笔接着写，因为笔者在这里踩了太多的坑，以至于距离上次写这篇笔记已过去一个月有余，虽然可以像上边一样直接放出答案，但还是决定把我遇到的几个坑复现给大家，心急的也可以直接跳过看最后的答案

IO 重定向这里我参考了网上的几个帖子和 github 上的项目，无一例外，所有的矛头都指向了 libc 中的这四个函数 open、open64、openat、openat64，大致的 hook 伪代码大概是下面这样样子，只要找个 hook 框架就可以直接起飞

```
int (*old_open)(const char *, int, mode_t);
static int new_open(const char *pathname, int flags, mode_t mode) {
    if (strcmp(pathname, apkPath__) == 0){
        return old_open(repPath__, flags, mode);
    }
    return old_open(pathname, flags, mode);
}
 
hook("libc.so", "open", new_open, (void **) &old_open);

```

但当我实际进行 hook 时却发现了一个非常致命的问题，我在 libc.so 的 open 可以 hook 到几乎所有被打开的文件，但唯独缺少最重要的目标文件 base.apk，一度我以为是我 hook 姿势的问题，甚至我还换了好几个 hook 框架，但大同小异，都无法 hook 到 base.apk，这里就卡了我好久，下面直接放分析思路

首先是找到读取 Zip 文件使用类为 ZipFile

![](https://jiandanyun.myds.me:4567/2023/07/image-20230727234951039.png)

跟进去发现是调用 两个参数的 ZipFile，继续跟

![](https://jiandanyun.myds.me:4567/2023/07/image-20230727235235734.png)

继续跟会发现最后调用了三个参数的 ZipFile 函数，在其中可以发现调用了一个 jni 的 open 函数，ZipFile 所在的类是 package java.util.zip，所以我们需要找到其对应的 Native 代码

![](https://jiandanyun.myds.me:4567/2023/07/image-20230727235347536.png)

在在线源码网站中搜索一番找到了下面这段代码，从文件名可以确定这就是我们要找的地方了，这里面有两个函数比较重要，一个是 ZIP_Get_From_Catch 另一个是 JVM_Open

![](https://jiandanyun.myds.me:4567/2023/07/image-20230727235757098.png)

先看 JVM_Open，跟进去后发现调用了一个名为 open 的函数，说实话因为是在线源码，没有办法直接看这个 open 到底是啥，但在和群里大佬探讨的时候，大佬斩钉截铁的告诉我，这个 open 就是 libc 的 open ！事已至此，hook 不到 bask.apk 的原因只有一个！那就是 ZIP_Get_From_Catch ！

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728000430084.png)

如何验证我们的猜想呢，因为 ZIP_Get_From_Catch 并不是一个导出函数，所以验证起来手段就少了许多，我选择了一个对我来讲比较简单，并且非常粗暴的方法，那就是在 art 源码里加 log，并且还发现了之前分析的一个问题，那就是 native 的文件找错了，它的真正路径是 **libcore/ojluni/src/main/native/ZipFile.c**

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728215308391.png)

刷机后重新启动 app 点击获取签名的按钮后，可以看到如下日志，很明显在还未获取签名时系统加载过一次 base.apk，而在我们点击获取签名时走的是 ZIP_Get_From_Catch 分支，这个地方我有尝试过将 hook 的时机提到 so init 的时机，但还是 hook 不到这个系统加载的这次打开，只能暂且认为系统加载这个 zip 的时候 app 还没有启动

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728215335148.png)

shadowhook
----------

于是乎这里我们可以做的选择就比较有限了，hook ZipFile_open 这个导出的 jni 函数貌似是可行的，但事情真的会如此顺利么，这里我有尝试使用 xhook 和 shadowhook 去 hook 这个 ZipFile_open 函数，但不好意思，全部翻车，xhook 完全 hook 不到（不排除是我姿势不对，毕竟没怎么用过），shadowhook 则比较诡异，只能 hook 到一次，这里我还问了我们公司的大佬，只能说这个框架确实是多少有问题的

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728010250190.png)

Find Symbol
-----------

一个 jni 函数又是走缓存又是不给 hook 的，我反手就是一个 diy inlinehook，倒要看看它还能不能继续这么傲娇，话不多说，继续参考网上各路大神的 inlinehook 帖子和项目，写代码嘛，主打的就是一个抄，肉丝大佬的代码更是得大抄特抄

首先是 so 解析部分，想要 hook 一个函数首先要拿到它在内存中的地址，本来这里准备直接照抄肉丝老师的代码的，但没想到肉丝老师的代码无法正确解析 libopenjdk.so 这个库（痛苦面具直接），最后经过几天的学习与分析，最终定位问题所在是符号表的大小计算有问题，肉丝大佬的代码中，直接以字符串表头的地址减符号表头的地址得到的大小作为整个符号表的大小，但这样其实是有问题的，有啥问题我也不知道，但我知道的是像他那样计算在访问字符串的时候会越界，GPT 说正确的判断符号表的大小的方式是从符号表头遍历到最后 st_name 为 0 时作为符号表的结束

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728012233796.png)

但不幸的是我使用 010 editor 中查看 libopenjdk.so 符号表的结尾并不是 0，而是 00 00 02 00 这种诡异的数据，不过通过观察发现一个正常的 sym.st_info 的值只为 0x12 0x11 0x22，如果不是这三个就结束了，这里就先这样判断吧（我太难了...）

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728012758427.png)

这里给出可以识别到 ZipFile_open 的代码（符号表的大小判断估计还是有问题的，所以找到就赶紧 break，如果想通用一点还是得多观察几个 lib，看看结尾的共同之处）

```
void* findSoLibraryAddress(const char* libraryName) {
    char line[1024];
    uintptr_t start = 0;
    uintptr_t end = 0;
 
    FILE* fp = fopen("/proc/self/maps", "r");
    if (fp == nullptr) {
        perror("Error opening /proc/self/maps");
        return nullptr;
    }
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, libraryName)) {
            __android_log_print(ANDROID_LOG_INFO, "lxz", "Found %s", libraryName);
 
            start = strtoull(strtok(line, "-"), NULL, 16);
            end = strtoull(strtok(NULL, " "), NULL, 16);
 
            __android_log_print(ANDROID_LOG_INFO, "lxz", "Start address: %p, End address: %p",
                                reinterpret_cast(start), reinterpret_cast(end));
            fclose(fp);
            return reinterpret_cast(start);
        }
    }
    fclose(fp);
    return nullptr;
}
 
// 获取函数名称的偏移地址
unsigned long getFunctionOffset(char* startr, const char* functionName) {
    Elf64_Ehdr header;
    Elf64_Phdr cc;
    int phof = 0;
    char *strtab_ = nullptr;
    Elf64_Sym *symtab_ = nullptr;
    memcpy(&header, startr, sizeof(Elf64_Ehdr));
 
    for (int y = 0; y < header.e_phnum; y++) {
        memcpy(&cc, startr + header.e_phoff + sizeof(Elf64_Phdr) * y, sizeof(Elf64_Phdr));
        if (cc.p_type == 1) {
            __android_log_print(6, "lxz", "寻找到首段偏移");
            phof = cc.p_paddr;
            break;
        }
    }
 
    for (int y1 = 0; y1 < header.e_phnum; y1++) {
        memcpy(&cc, startr + header.e_phoff + sizeof(Elf64_Phdr) * y1, sizeof(Elf64_Phdr));
        if (cc.p_type == 2) {
            Elf64_Dyn dd = {0};
            for (int yy = 0; yy == 0 || dd.d_tag != 0; yy++) {
                memcpy(&dd, startr + cc.p_offset + yy * sizeof(Elf64_Dyn), sizeof(Elf64_Dyn));
                if (dd.d_tag == 5) {
                    __android_log_print(6, "lxz", "获取到字符串表");
                    strtab_ = (char*)(startr + dd.d_un.d_ptr - phof);
                }else if (dd.d_tag == 6) {
                    __android_log_print(6, "lxz", "获取到符号表");
                    symtab_ = (Elf64_Sym*)(startr + dd.d_un.d_ptr - phof);
                }
            }
        }
    }
 
    Elf64_Sym mytmpsym = {0};
    for(int t = 1; ; t++){
        memcpy(&mytmpsym,symtab_ + t,sizeof(Elf64_Sym));
        if((mytmpsym.st_info & 0xf0) == 0){
            // 据说正常符号表以空表做结尾，其中的 st_name 应该是 0，
            // 但这个不知道为什么不是 它的结尾是这样的
            // 00 00 02 00 02 00 01 00 02 00 02 00 02 00 02 00
            // 01 00 02 00 02 00 02 00
            // 观察发现 mytmpsym.st_info 的值只为 0x12 0x11 0x22，这里就先这样判断吧（我太难了...）
            break;
        }
 
        if(strcmp(strtab_ + mytmpsym.st_name, functionName) == 0){
            __android_log_print(6, "lxz", "找到目标函数名称 %s 偏移 %p", strtab_ + mytmpsym.st_name, mytmpsym.st_value);
            return mytmpsym.st_value;
        }
    }
    return 0;
} 
```

Inline Hook
-----------

android 的 inline hook 写起来要比 x86 麻烦许多，主要有以下几点：

*   函数头部没有预留 hook 空间，需要恢复（搞过 x86 的都知道， mov edi, edi）
*   没有 popad 这种方便至极的指令可以使用，甚至 ebp 都没有
*   头部如果是特殊指令在恢复时需要每一个都单独处理（这是暂时不做考虑）

下面就是参考肉丝大佬代码写（~直接开抄~）的一个简陋版 inline hook 框架

```
#include "lxz_inline_hook.h"
#define PAGE_START(x)  ((x) & PAGE_MASK)
 
void hook_template(){
 
    asm("sub SP, SP, #0x200");              // 开辟 200 个字节大小的栈用于保存寄存器
    asm("mrs x16, NZCV");                   // 将条件状态寄存器的值加载到X16寄存器中
    asm("stp X29, X30, [SP,#0x10]");
    asm("stp X0, X1, [SP,#0x20]");          // 保存所有寄存器到栈中
    asm("stp X2, X3, [SP,#0x30]");
    asm("stp X4, X5, [SP,#0x40]");
    asm("stp X6, X7, [SP,#0x50]");
    asm("stp X8, X9, [SP,#0x60]");
    asm("stp X10, X11, [SP,#0x70]");
    asm("stp X12, X13, [SP,#0x80]");
    asm("stp X14, X15, [SP,#0x90]");
    asm("stp X16, X17, [SP,#0x100]");
    asm("stp X28, X19, [SP,#0x110]");
    asm("stp X20, X21, [SP,#0x120]");
    asm("stp X22, X23, [SP,#0x130]");
    asm("stp X24, X25, [SP,#0x140]");
    asm("stp X26, X27, [SP,#0x150]");
 
    // blr用于实现函数调用和返回机制，保存返回地址并跳转到指定地址，用于函数间的跳转和调用。
    // br用于无条件跳转到指定地址，通常用于简单的无条件分支需求，不涉及函数调用和返回。
    asm("mov x0,x0");      // *(int *) ((char *) (a) + n) = 0x58000070;    // ldr x16, #0xc
    asm("mov x0,x0");      // *(int *) ((char *) (a) + n+4) = 0xd63f0200;  // blr x16
    asm("mov x0,x0");      // *(int *) ((char *) (a) + n+8) = 0x14000003;  // b #0xc
    asm("mov x0,x0");      // *(long *) ((char *) (a) + n+12) = (long)(address2); // 填充hook函数地址
    asm("mov x0,x0");     
 
    asm("ldp X26, X27, [SP,#0x150]");
    asm("ldp X24, X25, [SP,#0x140]");
    asm("ldp X22, X23, [SP,#0x130]");
    asm("ldp X20, X21, [SP,#0x120]");
    asm("ldp X28, X19, [SP,#0x110]");
    asm("ldp X16, X17, [SP,#0x100]");
    asm("ldp X14, X15, [SP,#0x90]");
    asm("ldp X12, X13, [SP,#0x80]");
    asm("ldp X10, X11, [SP,#0x70]");
    asm("ldp X8, X9, [SP,#0x60]");
    asm("ldp X6, X7, [SP,#0x50]");
    asm("ldp X4, X5, [SP,#0x40]");
    asm("ldp X2, X3, [SP,#0x30]");
    asm("ldp X0, X1, [SP,#0x20]");
    asm("ldp X29, X30, [SP,#0x10]");
    asm("msr NZCV, x16");
    asm("add SP, SP, #0x200");
 
    asm("mov x0,x0");   // 指令膨胀，为修复特殊指令预留空间
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
    asm("mov x0,x0");   asm("mov x0,x0");
}
 
void register_hook(void* address1, void* address2){
 
    // 设置内存读写执行权限
    mprotect((void *) PAGE_START((long) address1), PAGE_SIZE, PROT_WRITE | PROT_READ | PROT_EXEC);
    mprotect((void *) PAGE_START((long) address2), PAGE_SIZE, PROT_WRITE | PROT_READ | PROT_EXEC);
 
    // 保存原函数指令
    int old_code1 = *(int *) ((char *) address1);
    int old_code2 = *(int *) ((char *) address1 + 4);
    int old_code3 = *(int *) ((char *) address1 + 8);
    int old_code4 = *(int *) ((char *) address1 + 12);
    long ret_addr = (long)address1 + 0x10;
 
    // 申请跳板空间，将模板拷贝到申请的空间中
    void* a = malloc(0x10c);
    memcpy(a, (void*)hook_template, 0x10c);
    mprotect((void *) PAGE_START((long) a), PAGE_SIZE, PROT_WRITE | PROT_READ | PROT_EXEC);
 
    // 将原函数头部改为跳板指令
    *(int *) ((char *) address1) = 0x58000050;              // ldr x16,8
    *(int *) ((char *) address1 + 4) = 0xd61f0200;          // br x16
    *(long *) ((char *) address1 + 8) = (long)a;            // 跳板空间地址
 
    // 将模板预留的位置 1 填充为跳转 address2 的跳板指令
    int n=0x44;
    *(int *) ((char *) (a) + n) = 0x58000070;               // ldr x16, #0xC
    *(int *) ((char *) (a) + n+4) = 0xd63f0200;             // blr x16
    *(int *) ((char *) (a) + n+8) = 0x14000003;             // b #0xC
    *(long *) ((char *) (a) + n+12) = (long)address2;       // 填充hook函数地址
 
    int m=0xA4;
    *(int *) ((char *) (a) + m) = old_code1;
    *(int *) ((char *) (a) + m + 4) = old_code2;
    *(int *) ((char *) (a) + m + 8) = old_code3;
    *(int *) ((char *) (a) + m + 12) = old_code4;
 
    *(int *) ((char *) (a) + m + 24) = 0x58000050;;         // ldr x16,8
    *(int *) ((char *) (a) + m + 28) = 0xd61f0200;          // br x16
    *(long *) ((char *) (a) + m + 32) = ret_addr;           // 跳回原函数
}

```

hook ZipFile_open
-----------------

到了这里就是 hook 的实现部分了，但这里也有一点问题，那就是在 hook 函数中如何取出栈帧，虽然可以用汇编取出最上层栈帧，但因为 arm 中没有 ebp 这种寄存器，所以在 inline_fun 这个函数给自己开辟了多少的栈帧空间就是个不固定未知数，所以这里我采取解析函数头部汇编指令的方式获取 inline_fun 给自己开辟的栈帧大小，再向前倒推一下 hook 模板中存放栈的偏移，最终获取被 hook 函数的参数

```
#include "lxz_find_symbol.h"
#include "lxz_inline_hook.h"
 
JNIEnv* g_env = nullptr;
jobject g_thiz = nullptr;
 
void inline_fun(){
    __android_log_print(6,"lxz","i am inline_fun");
 
    long* sp = nullptr;
    __asm__ __volatile__ ("mov %0, sp\n": "=r" (sp));
 
    // 这里有点难顶， inline_fun 这个函数刚进来时会给自己开辟一些栈帧
    // 简单尝试了一下，开辟栈帧数量 0x1000 以下这样计算开辟栈帧的数量应该是没啥问题
    int first_code = *(int*)inline_fun;
    first_code = first_code >> 8;
    first_code = first_code & 0x0000FFFF;
    int sp_offset = (first_code)/4;
 
    // 定位到 hook 模板的栈顶
    sp = (long*)((char*)sp + sp_offset);
 
    // 定位到 hook 模板备份被 hook 函数参数的栈地址
    sp = (long*)((char*)sp + 0x20);
 
    // arg_id 从 0 开始
    int arg_id = 2;
    if(g_env != nullptr){
        const char* arg_str = g_env->GetStringUTFChars(*(jstring*)(sp+arg_id),0);
 
        jclass jclass1 = g_env->GetObjectClass(g_thiz);
        jmethodID jmethodId1 = g_env->GetMethodID(jclass1,"getPackageResourcePath","()Ljava/lang/String;");
        jstring jpackage_path = (jstring)g_env->CallObjectMethod(g_thiz, jmethodId1);
        const char* package_path = g_env->GetStringUTFChars(jpackage_path,0);
 
        if(strcmp(arg_str,package_path) == 0){
            __android_log_print(6,"lxz","识别成功 %s", arg_str);
 
            // 获取包名
            jclass jclass2 = g_env->GetObjectClass(g_thiz);
            jmethodID jmethodId2 = g_env->GetMethodID(jclass2,"getPackageName","()Ljava/lang/String;");
            jstring jpackage_name = (jstring)g_env->CallObjectMethod(g_thiz, jmethodId2);
            const char* package_name = g_env->GetStringUTFChars(jpackage_name,0);
 
            // 拼接傀儡app的绝对路径
            std::string replace_path = "/data/data/";
            replace_path.append(package_name);
            replace_path.append("/org.apk");
            jstring jreplace_path = g_env->NewStringUTF(replace_path.c_str());
 
            // 修改栈帧达到替换参数的目的
            *(jstring*)(sp+arg_id) = jreplace_path;
        }
    }
}
 
extern "C"
JNIEXPORT void JNICALL
Java_com_example_myinlinehook_MainActivity_hook_1zip_1fileopen(JNIEnv *env, jobject thiz) {
 
    g_env = env;
    g_thiz = env->NewGlobalRef(thiz);
 
    void* so_base = findSoLibraryAddress("libopenjdk.so");
 
    unsigned long ZipFile_open_offset = getFunctionOffset((char*)so_base, "ZipFile_open");
 
    register_hook(ZipFile_open_offset + (char*)so_base,(void*)inline_fun);
 
    sleep(1);   // 这一秒十分关键，如果不添加这个延时，被hook函数的内存还未修改时
                        // cpu 就有可能已经执行过被 hook 的函数了，完善一些的话可以选择加锁
}

```

下面是 java 层的代码，重点就是在 hook 之前要把傀儡 app 释放到指定位置最为 io 重定向的目标

```
private ActivityMainBinding binding;
TextView tv;
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    binding = ActivityMainBinding.inflate(getLayoutInflater());
    setContentView(binding.getRoot());
    tv = binding.sampleText;
 
    // 将傀儡app存放到指定位置 /data/data/"+getPackageName()+"/org.apk
    try {
        copyFileFromAssets(this,"org.apk","/data/data/"+getPackageName()+"/org.apk");
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
 
 
}
public void get_signature(View view) {
    Log.e("lxz", "点击按钮获取签名");
    tv.setText(tv.getText() + "\n\n" + "获取到signature:\n" + md5(signatureFromAPK()));
}
 
public void hook_zipopen(View view) {
    // 执行 hook 代码，进行 io 重定向
    tv.setText(tv.getText() + "\n\n" + "执行 hook 代码，进行 io 重定向");
    hook_zip_fileopen();
}
 
private byte[] signatureFromAPK() {
    try (ZipFile zipFile = new ZipFile(getPackageResourcePath())) {
        Enumeration entries = zipFile.entries();
        while (entries.hasMoreElements()) {
            ZipEntry entry = entries.nextElement();
            if (entry.getName().matches("(META-INF/.*)\\.(RSA|DSA|EC)")) {
                InputStream is = zipFile.getInputStream(entry);
                CertificateFactory certFactory = CertificateFactory.getInstance("X509");
                X509Certificate x509Cert = (X509Certificate) certFactory.generateCertificate(is);
                return x509Cert.getEncoded();
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
 
private String md5(byte[] bytes) {
    if (bytes == null) {
        return "null";
    }
    try {
        byte[] digest = MessageDigest.getInstance("MD5").digest(bytes);
        String hexDigits = "0123456789abcdef";
        char[] str = new char[digest.length * 2];
        int k = 0;
        for (byte b : digest) {
            str[k++] = hexDigits.charAt(b >>> 4 & 0xf);
            str[k++] = hexDigits.charAt(b & 0xf);
        }
        return new String(str);
    } catch (NoSuchAlgorithmException e) {
        throw new RuntimeException(e);
    }
}
 
private static void copyFileFromAssets(Context context, String assetFileName, String destinationFilePath) throws IOException {
    AssetManager assetManager = context.getAssets();
 
    InputStream inputStream = null;
    OutputStream outputStream = null;
 
    try {
        inputStream = assetManager.open(assetFileName);
        outputStream = new FileOutputStream(destinationFilePath);
 
        byte[] buffer = new byte[1024];
        int length;
        while ((length = inputStream.read(buffer)) > 0) {
            outputStream.write(buffer, 0, length);
        }
    } finally {
        if (inputStream != null) {
            inputStream.close();
        }
        if (outputStream != null) {
            outputStream.close();
        }
    }
}

```

最后放一张效果图，虽然我们的 inline hook 十分简陋，但 IO 重定向没有问题，成功改签

![](https://jiandanyun.myds.me:4567/2023/07/image-20230728225506629.png)

总结
==

本次研习历时一个月有余，本以为是简单复现，却不曾想在其中经历了各种意外的波折，尽管事情并没有像最初预期的那样顺利进行，但都已成为我宝贵的学习经验。前人的方法不总是全面的，还需要我们后来者继续补充，逆向之路，道阻且长，笔者借以此文抛砖引玉，希望今后可以看到更多的技术贴，大家一起交流学习

  

[看雪 ·2023 KCTF 年度赛即将来袭！ [防守方] 规则发布，征题截止 08 月 25 日](https://bbs.kanxue.com/thread-276642.htm)

最后于 10 小时前 被简单的简单编辑 ，原因： [#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm)