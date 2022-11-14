> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.oversecured.com](https://blog.oversecured.com/Discovering-vendor-specific-vulnerabilities-in-Android/)

> For several years, Oversecured has been the best way to discover vulnerabilities in Android and iOS m......

For several years, Oversecured has been the best way to discover vulnerabilities in Android and iOS mobile apps. We are always carrying out research and improving the quality of our detection. In the middle of the year, we realized that Oversecured was ideal for regular apps, such as banking or messaging. Still, our rules and checks covered only a small percentage of the Android system APIs that privileged apps can use. This meant we might miss a large number of vulnerabilities when scanning apps of this kind.

And so it was time for us to fix the problem, and ensure our checks covered absolutely all system APIs! As a result, the Oversecured Android vulnerability scanner now supports two environments: Google and Samsung. In the process, we have discovered tens of Samsung vulnerabilities and a few on Google.

In this article, we shall describe our research, provide examples of Android’s fragmentation across different vendors, explain the reasons, and give some examples of vulnerabilities. The vulnerabilities discovered by our research will be published subsequently: some of them have not yet been fixed.

Do you want to check your mobile apps for these types of vulnerabilities? Oversecured’s mobile app scanner provides an automatic solution that helps to detect vulnerabilities in Android and iOS mobile apps. You can integrate Oversecured into your development process and check every new line of your code to ensure your users are always protected.

Start securing your apps by starting a free 2-week trial from [Quick Start](https://oversecured.com/docs/quick-start/), or you can [book a call](https://calendly.com/oversecured/30min) with our team or [contact us](https://oversecured.com/contact-us) to explore further.

We also give all new users two free scans, so they can check any apps for vulnerabilities! You can do this on the [New Scan](https://oversecured.com/scan/create) page.

[](#understanding-the-android-architecture)[Understanding the Android architecture](#understanding-the-android-architecture)

You can see that if you decompile an obfuscated app, some code is nonetheless not obfuscated. For example, standard Android operations such as working with an `Activity`. This is because that class is part of the standard Android APIs. When any app is run, the `/system/framework/framework.jar` file is loaded into its class loader, where the class `android.app.Activity` is declared. This is the reason why no obfuscator can rename standard calls to this and many other classes.

Thus, all the standard Android APIs are in `.jar` files in the following directories:

*   `/system/`
*   `/system_ext/`
*   `/apex/`

We began by gathering all the `.jar` files from these directories on Google Emulator API 33 and decompiling them, merging all the resulting source code. That gave us all the code to begin our investigation. The next step was to find out which among tens of thousands of classes were APIs that could present a security risk if they were handled improperly. How could we do that? It was very simple!

Android has lots of standard managers: [`ActivityManager`](https://developer.android.com/reference/android/app/ActivityManager), [`TelephonyManager`](https://developer.android.com/reference/android/telephony/TelephonyManager), and others. These, in their turn, use system services that are not usually visible to developers. We can say that all these managers are just pretty, convenient wrappers around lower-level system services, which actually provide all inter-process communications. And these system services are AIDL interfaces working in the privileged `system_server`. Only some services, like NFC or Bluetooth, are processed by separate and less privileged system apps.

A list of all the services that an app can access can be obtained by running the following code:

```
    try {
        for (String serviceName : listServices()) {
            IBinder binder = getService(serviceName);
            String descriptor = binder != null ? binder.getInterfaceDescriptor() : null;
            Log.d("dump", serviceName + " : " + descriptor);
        }
    } catch (Throwable th) {
        throw new RuntimeException(th);
    }


```

```
    private String[] listServices() throws Throwable {
        return (String[]) Class.forName("android.os.ServiceManager")
                .getDeclaredMethod("listServices")
                .invoke(null);
    }

    private IBinder getService(String name) throws Throwable {
        return (IBinder) Class.forName("android.os.ServiceManager")
                .getDeclaredMethod("getService", String.class)
                .invoke(null, name);
    }


```

The system reports more than 250 services. The result for the emulator is as follows: ![](https://blog.oversecured.com/assets/images/list_services.png)

In nearly every case, the interface descriptor is an AIDL interface whose name is the same as that of a Java class. For example, the `android.content.IClipboard` interface corresponds to `android/content/IClipboard.java` when decompiled, or `android/content/IClipboard.aidl` in its original form.

As an example of AIDL interaction, we can consider reading data from the clipboard. This is done using the interface `android.content.IClipboard.getPrimaryClip()`:

```
    android.content.ClipData getPrimaryClip(java.lang.String pkg, int userId) throws android.os.RemoteException;


```

```
        static final int TRANSACTION_getPrimaryClip = 4;


```

A call to it looks like this:

```
    ClipData data = getPrimaryClip();
    Log.d("dump", "getPrimaryClip(): " + data.getItemAt(0).getText());


```

```
    static final int TRANSACTION_getPrimaryClip = 4;

    private ClipData getPrimaryClip() throws Throwable {
        IBinder binder = getService("clipboard");
        Parcel parcel = Parcel.obtain();
        parcel.writeInterfaceToken(binder.getInterfaceDescriptor());
        parcel.writeString(getPackageName()); // first argument `pkg`
        parcel.writeInt(0); // second argument `userId`

        Parcel reply = Parcel.obtain();
        binder.transact(TRANSACTION_getPrimaryClip, parcel, reply, 0);
        reply.readException();

        if (reply.readBoolean()) {
            return ClipData.CREATOR.createFromParcel(reply);
        } else {
            return null;
        }
    }

    private IBinder getService(String name) throws Throwable {
        return (IBinder) Class.forName("android.os.ServiceManager")
                .getDeclaredMethod("getService", String.class)
                .invoke(null, name);
    }


```

Static fields beginning with `TRANSACTION_` are generated automatically for each method when the Java class is generated from the `.aidl` file. To see the method code, you only need to open the decompiled source and examine it.

[](#security-of-aidl-interfaces)[Security of AIDL interfaces](#security-of-aidl-interfaces)

The security of all AIDL operations is based on permission checks at the implementation level for each interface method. The AIDL server knows the UID (and therefore the package name) of the calling app and checks that the caller has the required permission.

As another short and simple example, we can look at the method for rebooting the device, `android.os.IPowerManager.reboot()`. Its implementation is in the file `com/android/server/power/PowerManagerService.java`:

```
    public void reboot(boolean confirm, String reason, boolean wait) {
        mContext.enforceCallingOrSelfPermission("android.permission.REBOOT", null); // permission check
        if ("recovery".equals(reason) || "recovery-update".equals(reason)) {
            mContext.enforceCallingOrSelfPermission("android.permission.RECOVERY", null);
        }
        final long ident = Binder.clearCallingIdentity();
        try {
            shutdownOrRebootInternal(0, confirm, reason, wait);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }


```

If an app that calls this method does not have the `android.permission.REBOOT` permission, the method `Context.enforceCallingOrSelfPermission()` will throw a `SecurityException` and the device will not reboot.

[](#studying-system-apis)[Studying system APIs](#studying-system-apis)

As we said, our research began with Google Android installed on Google Emulator. We studied all interfaces and all their methods, to find which were privileged and create rules for the Oversecured Android scanner regarding improper interaction with them. We also created rules for all managers and other classes that work with these interfaces internally. It should be added that our manual audit of all interfaces also found absent or vulnerable checks for various actions (such as reading or altering various data or the device configuration). We have sent reports of these bugs to Google.

[](#inventing-a-new-method-for-code-analysis)[Inventing a new method for code analysis](#inventing-a-new-method-for-code-analysis)

Oversecured was previously based on a large number of well-known approaches to static and dynamic code analysis, but the main feature was dataflow analysis. The scanner examined all an app’s possible execution paths and the data propagation in it. This is an excellent way of finding most, if not all, vulnerabilities in Android and iOS apps.

But when we were studying the system API functionality we realized that the mere fact some piece of code is executed could be a danger. As we saw, rebooting the device requires the `android.permission.REBOOT` permission. But what if an attacker can force a privileged app to execute this code? It will lead to a vulnerability!

Oversecured now carries out two additional analyses:

*   control flow: if an attacker can force the app we are analyzing to perform an unsafe action, e.g. via an exported component
*   dataflow to control flow: we now discover dynamically created handlers (such as unprotected broadcast receivers or local web servers) and report if their data processing involves executing unsafe actions

[](#understanding-samsung-android)[Understanding Samsung Android](#understanding-samsung-android)

Next, we chose to extend our rules to cover Samsung’s Android, because we already worked with it last year and [discovered 18 vulnerabilities](https://blog.oversecured.com/Two-weeks-of-securing-Samsung-devices-Part-1/) in pre-installed apps, of various levels of criticality. Our experience working with Samsung’s VDP was very positive.

We performed the same actions:

1.  Pulled all `.jar` files from the `/system/`, `/system_ext/` и `/apex/` directories
2.  Decompiled the code and merged the resulting files
3.  Obtained a list of all accessible system services
4.  Compared the services received and the most important classes

The scale of the changes amazed us! Almost every key class was changed in one way or another. For example, there are around 400 system services, and many service AOSPs are extended. The reasons are good ones:

*   support for Samsung’s enormous ecosystem
*   additional options for device configuration
*   extensive MDM functionality
*   integration with Samsung hardware, including the S Pen and S Watch

The main differences between the Google and Samsung versions of Android, from the security point of view, are:

1.  An extended set of permissions
2.  An extended set of protected broadcasts (the list of actions that non-privileged apps can receive but cannot send)
3.  Additional APIs, protected by various checks of the app signature, UID, package name, permissions, etc.

In the process of studying the changes, we also found some funny vulnerabilities. One example is in the `backup` service (`android.app.backup.IBackupManager`), file `com/android/server/backup/BackupManagerService.java`: ![](https://blog.oversecured.com/assets/images/backup_manager_example.png)

As you see from the screenshot, `isBackupEnabled()` is an AOSP method. The permission `android.permission.BACKUP` is required to find out whether backups are switched on or not. But Samsung added a custom method `semIsBackupEnabled()`, which returns the value with no checks at all.

We created an additional scanning environment to support this version of Android. Then we added rules to search for vulnerabilities in Samsung Android, the list of built-in permissions, and protected broadcasts. We ran the scan in the new environment:

1.  Samsung Framework
2.  System apps
3.  All other Samsung apps

As an example of the scan results and of Android’s fragmentation, here is a vulnerability we discovered using a dataflow to control flow analysis of the Samsung Android Framework, in the patched file `com/android/server/StorageManagerService.java`: ![](https://blog.oversecured.com/assets/images/samsung_reboot.png)

When the user sends a broadcast with the action `com.samsung.intent.action.RESTART_OF_SDCARDBADREMOVED_HASAPK`, the device automatically reboots — sidestepping permission restrictions.

An example of system app fragmentation is a vulnerability in the Settings (`com.android.settings`) app. Samsung extended the `com.android.settings.homepage.SettingsHomepageActivity` class and added custom code that allowed a third-party app to access arbitrary activities: ![](https://blog.oversecured.com/assets/images/launchanywhere.png)

In a regular app, this would lead to the ability to access only within it, but since Settings has UID 1000 (`system`), the vulnerability led to access to any non-exported activities of any apps. This type of attack was previously called LaunchAnyWhere.

We revealed a large number of security problems in the Samsung ecosystem, and we have informed Samsung’s VDP. So far Samsung has fixed more than 60 vulnerabilities, but there is still a large number outstanding.

[](#preventing-these-vulnerabilities)[Preventing these vulnerabilities](#preventing-these-vulnerabilities)

It can be challenging to keep track of security, especially in large projects. You can use the Oversecured vulnerability scanner since it tracks all known security issues on Android and iOS including errors when working with files. To begin testing your apps, use [Quick Start](https://oversecured.com/docs/quick-start/), [book a call](https://calendly.com/oversecured/30min) or [contact us](https://oversecured.com/contact-us).