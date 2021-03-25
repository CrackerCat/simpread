> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [luoyesiqiu.github.io](https://luoyesiqiu.github.io/2021/03/22/%E4%BF%AE%E6%94%B9Android%E6%BA%90%E7%A0%81%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%94%9F%E5%BA%94%E7%94%A8%E5%8F%8C%E5%BC%80%EF%BC%8C%E5%BA%94%E7%94%A8%E5%A4%9A%E5%BC%80/)

![](https://luoyesiqiu.github.io/images/multi-app/sample.png)

把某系统双开的两个 app 的信息进行对比

[](#1-1-目录的对比 "1.1 目录的对比")1.1 目录的对比
-----------------------------------

### [](#1-1-1-data目录对比 "1.1.1 data目录对比")1.1.1 data 目录对比

原应用：

```
/data/user/0/com.luoyesiqiu.crackme/files
```

被复制的应用：

```
/data/user/999/com.luoyesiqiu.crackme/files
```

### [](#1-1-2-apk所在目录对比 "1.1.2 apk所在目录对比")1.1.2 apk 所在目录对比

原应用：

`/data/app/com.luoyesiqiu.crackme-H1Dvbka0t42rzlCAqSpgHQ==/base.apk`

被复制的应用：

`/data/app/com.luoyesiqiu.crackme-H1Dvbka0t42rzlCAqSpgHQ==/base.apk`

通过对比 apk 安装目录和数据目录，我们可以知道，该系统的双开是**共用同一个 apk**，但是却拥有**独立的数据目录**。

[](#1-2-进程信息对比 "1.2 进程信息对比")1.2 进程信息对比
--------------------------------------

```
USER           PID  PPID     VSZ    RSS WCHAN            ADDR S NAME
u0_a161      30284   918 2276572  48420 SyS_epoll_wait      0 S com.luoyesiqiu.crackme
u999_a161    30311   918 2276572  48004 SyS_epoll_wait      0 S com.luoyesiqiu.crackme
```

通过查看进程信息，可以知道，这两个应用运行于**不同的用户**中。

为了实现和它相似的功能，我们做下文的配置。

从 Android5.0 开始，Android 支持创建 Profile. 默认情况下，系统只允许创建一个新的多开用户，也就是只能双开，但是修改源码可以达到创建多个用户。

修改`frameworks/base/services/core/java/com/android/server/pm/UserManagerService.java`  
的 MAX_MANAGED_PROFILES 字段，改成自己想要创建的最大用户数，它的默认值是 1.

创建一个用户即创建一个多开容器，调用 createProfile 方法，flag 传入 0x00000020，以创建一个用户并将它开启

```
private  static int getUserIdFromUserInfo(Object userInfo) {
    int userId = -1;
    try {
        Field field_id = userInfo.getClass().getDeclaredField("id");
        field_id.setAccessible(true);
        userId = (Integer)field_id.get(userInfo);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return userId;
}

public boolean startUser(int userId){
    Object iActivityManager = null;
    try {
        iActivityManager = Class.forName("android.app.ActivityManagerNative").getMethod("getDefault").invoke(null);

        boolean isOk=(boolean)iActivityManager.getClass().getMethod("startUserInBackground",int.class)
                .invoke(iActivityManager,userId);
        return isOk;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return false;
}


public  String createProfile(Context context, String userName, int flag) {
    UserManager mUserManager = (UserManager) context.getSystemService(Context.USER_SERVICE);

    UserHandle userHandle = UserHandle.getUserHandleForUid(0);

    Log.d(TAG,"userHandle = "+userHandle.toString());
    try {
        int getIdentifier=(int)userHandle.getClass().getMethod("getIdentifier").invoke(userHandle);
        Log.d(TAG,"Identifier = "+getIdentifier);
        mUserInfo=mUserManager.getClass().getMethod("createProfileForUser",String.class, int.class, int.class)
                .invoke(mUserManager
                        ,userName
                        , flag
                        ,getIdentifier);
        if(mUserInfo==null){
            Log.d(TAG, "mUserInfo is null!");
            return null;
        }
        int userId = getUserIdFromUserInfo(mUserInfo);
        boolean isOk=startUser(userId);
        Log.d(TAG, "startUserInBackground() userId = " + userId + " | isOk = " + isOk);
        return isOk ? ""+userId : null;
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

> 注：创建用户的 App 要在 AndroidManifest.xml 的 manifest 节点下加入`android:sharedUserId="android.uid.system"`字段，加入`<uses-permission android:/>`权限，还要使用系统的 platform 签名对 apk 进行签名。

上面的方法似乎有点麻烦，其实创建多开用户还有更简单的办法，只需要一条命令：

```
pm create-user --profileOf 0 --managed myspace
```

*   profileOf 指定父用户 id,0 即系统用户
*   managed 指定创建的用户为多开用户
*   myspace 为用户名

默认情况下，在创建一个新用户的时候，系统会给新用户复制一份系统应用，但是在子用户中我们并不需要系统应用，所以我们要在子用户中取消安装这些系统应用。

> 注：系统应用可以不安装到子用户，但是系统服务一定要安装到子用户，否则，运行在子用户的 app 可能无法正常运行。

修改 frameworks/base/services/core/java/com/android/server/pm/Settings.java

createNewUserLI 方法，对系统应用和系统服务是否安装到子用户进行配置。

```
private final String[] excludeLiStrings={
    "android",
    "android.ext.services",
    "android.ext.shared",
    "com.android.bluetooth",
    "com.android.htmlviewer",
    "com.android.inputdevices",
    "com.android.shell",
    "com.android.certinstaller",
    "com.android.externalstorage",
    "com.android.providers.contacts",
    "com.android.providers.downloads",
    "com.android.providers.media",
    "com.android.providers.settings",
    "com.android.providers.userdictionary",
    "com.android.server.telecom",
    "com.android.packageinstaller",
    "com.android.settings",
    "com.android.providers.telephony",
    "com.android.mms.service",
    "com.android.webview",
    "com.android.location.fused",
    "com.android.cts.priv.ctsshim",
    "com.android.statementservice",
    "com.android.defcontainer",
    "com.android.keychain",
    "com.android.proxyhandler",
    "com.android.dreams.basic",
    "com.android.printspooler",
    "com.android.pacprocessor",
    "com.android.providers.downloads.ui"
};
private boolean isInExcludeList(String pkg){
    for(String excludePkg:excludeLiStrings){
        if(excludePkg.equals(pkg)){
            return true;
        }
    }
    return false;
}
void createNewUserLI(@NonNull PackageManagerService service, @NonNull Installer installer,
        int userHandle) {
    String[] volumeUuids;
    String[] names;
    int[] appIds;
    String[] seinfos;
    int[] targetSdkVersions;
    int packagesCount;
    synchronized (mPackages) {
        Collection<PackageSetting> packages = mPackages.values();
        packagesCount = packages.size();
        volumeUuids = new String[packagesCount];
        names = new String[packagesCount];
        appIds = new int[packagesCount];
        seinfos = new String[packagesCount];
        targetSdkVersions = new int[packagesCount];
        Iterator<PackageSetting> packagesIterator = packages.iterator();
        for (int i = 0; i < packagesCount; i++) {
            PackageSetting ps = packagesIterator.next();
            if (ps.pkg == null || ps.pkg.applicationInfo == null) {
                continue;
            }
            
            
            
            if(userHandle > 0 && !isInExcludeList(ps.name)){
                ps.setInstalled(false, userHandle);
            }
            else{
                 ps.setInstalled(ps.isSystem(), userHandle);
            }
            
            
            volumeUuids[i] = ps.volumeUuid;
            names[i] = ps.name;
            appIds[i] = ps.appId;
            seinfos[i] = ps.pkg.applicationInfo.seinfo;
            targetSdkVersions[i] = ps.pkg.applicationInfo.targetSdkVersion;
        }
    }
    for (int i = 0; i < packagesCount; i++) {
        if (names[i] == null) {
            continue;
        }
        
        final int flags = StorageManager.FLAG_STORAGE_CE | StorageManager.FLAG_STORAGE_DE;
        try {
            installer.createAppData(volumeUuids[i], names[i], userHandle, flags, appIds[i],
                    seinfos[i], targetSdkVersions[i]);
        } catch (InstallerException e) {
            Slog.w(TAG, "Failed to prepare app data", e);
        }
    }
    synchronized (mPackages) {
        applyDefaultPreferredAppsLPw(service, userHandle);
    }
}
```

1.  安装 app 到指定用户

`pm install -t -r --user <userId> <apkPath>`

*   -r 替换存在的
    
*   –user 指定安装到的用户
    

> 注：安装 app 后要重启默认启动器（Launcher），不然可能会出现奇怪的问题

2.  从指定用户卸载 app

`pm uninstall --user <userId> <pkgName>`

开启子用户后，如果调用`adb install`或者`pm install`来安装 apk, 会把 apk 安装所有用户。这不是我们想要的，所以，我们修改成执行这些命令时，只把 app 安装到主用户。

第一步：针对用 pm install 命令安装 apk 的方式

frameworks/base/cmds/pm/src/com/android/commands/pm/Pm.java

```
private static class InstallParams {
    SessionParams sessionParams;
    String installerPackageName;
    
    int userId = UserHandle.USER_SYSTEM;
}
```

第二步：针对用 adb install 命令安装 apk 的方式

在旧版本系统上，`adb install`会调用`pm install`来安装 apk, 但在新版的系统上会调用`cmd package`命令来安装 apk。

`package`命令的具体实现在：

frameworks/base/services/core/java/com/android/server/pm/PackageManagerShellCommand.java

所以，修改以下代码：

```
private static class InstallParams {
    SessionParams sessionParams;
    String installerPackageName;
    
    int userId = UserHandle.USER_SYSTEM;
}
```

`adb shell pm remove-user <userId>`

或者调用以下代码删除

```
public  void deleteUser(Context context,int userId){
    UserManager userManager=(UserManager) context.getSystemService(Context.USER_SERVICE);
    try {
        userManager.getClass().getMethod("removeUser",int.class).invoke(userManager,userId);

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

开启多用户后，如果给多个子用户安装相同的 App，它们会显示相同的右下小图标 (在源码中被叫做 Badge)，让我们难以辨别，我们可以让不同的用户显示不同的右下小图标，数字图标就是不错的选择，我们来看看如何修改

`frameworks/base/core/java/android/app/ApplicationPackageManager.java`的 getBadgeResIdForUser 方法，返回一个 Drawable 资源 id，作为子用户 App 的右下小图标

```
private int getBadgeResIdForUser(int userId) {
    
    if (isManagedProfile(userId)) {
        return com.android.internal.R.drawable.ic_corp_icon_badge;
    }
    return 0;
}
```

这个图标是要在右下角的，所以在做图标的时候，要做一张大的图标，它的右下角放着要显示的小图标，其余部分以透明填充。做好图标后要生成 svg 格式，用 Android Studio 以 Vector Assets 导入，大小 64x64，导入成功会在项目的 res/drawable 生成 drawable 资源文件，把资源文件替换到`frameworks/base/core/res/res/drawable/ic_corp_icon_badge.xml`，并在`frameworks/base/core/res/res/values/symbols.xml`添加对 drawable 文件的声明

默认情况下，最近任务列表是不会出现多开应用的。

在`frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`的 getRecentTasks 方法中，有一段校验：

```
for (int i = 0; i < recentsCount && maxNum > 0; i++) {
    TaskRecord tr = mRecentTasks.get(i);
    
    if (!tr.mUserSetupComplete) {
         
        if (DEBUG_RECENTS) Slog.d(TAG_RECENTS,
                     "Skipping, user setup not complete: " + tr);
                continue;
        }
    
    res.add(rti);
    
}
```

可以将这段校验注释掉，tr 是`frameworks/base/services/core/java/com/android/server/am/TaskRecord.java`类实例，mUserSetupComplete 赋值如下：

```
mUserSetupComplete = Settings.Secure.getIntForUser(mService.mContext.getContentResolver(),
        USER_SETUP_COMPLETE, 0, userId) != 0;
```

子用户默认会在最近任务的应用名称前加上” 工作” 这两个字，语言是英文会显示”Work”，如果想隐藏它们

中文：

`frameworks/base/core/res/res/values-zh-rCN/strings.xml`

英文：

`frameworks/base/core/res/res/values/strings.xml`

修改`managed_profile_label_badge`字段，去掉” 工作” 或者”Work” 即可。

如果是多开应用，卸载时提示的文本和主用户是不一样的，如果需要修改，则修改下面的文件

中文：

`packages/apps/PackageInstaller/res/values-zh-rCN/strings.xml`

英文：

`packages/apps/PackageInstaller/res/values/strings.xml`

修改`uninstall_application_text_user`字段的值

[Android 多用户 —— 从入门到应用分身 (上)](https://mp.weixin.qq.com/s/CeoijS42tkNLejkHw0waOw)