> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [darvincitech.wordpress.com](https://darvincitech.wordpress.com/2020/10/11/virtual-dynamic-analysis-part-2/)

This is the 2nd and last post on the research findings related to the security issues found in Cloning Apps([Parallel Space](https://play.google.com/store/apps/details?id=com.lbe.parallel.intl), [Dual Space](https://play.google.com/store/apps/details?id=com.ludashi.dualspace) etc). To recap, the [previous post](https://darvincitech.wordpress.com/2020/07/18/all-your-crypto-keys-belongs-to-me-in-android-virtual-containers/) is about bringing out the security issues for cloned apps(guest apps) from other cloned apps or from the Cloning Apps(host app). It is also demonstrated, how applications can lose the cryptographic secrets even if it is protected within Android KeyStore and with a fake security provider injected by the cloning app. In this post, I will be discussing more ways of attacking a cloned app

![](https://darvincitech.files.wordpress.com/2020/10/security-vulnerability-shutterstock-andy-dean-photography-601x513-568x485-1.jpg?w=568)

Access to App Sandbox Data to Other Apps
----------------------------------------

As mentioned in previous post, all guest apps inside the host app share the same user id, which makes all the guest apps to share the sandbox data with each other and also to the host app. [Vikas](https://github.com/su-vikas) and I demonstrated this issue with [Conware](https://github.com/su-vikas/conbeerlib) app which can fetch sandbox data of all cloned apps. Do try the app inside cloning apps to fetch other app’s sandbox data so easily. Here is a snapshot of the list of apps whose sandbox data is accessible for each other apps inside a Cloning App.

![](https://darvincitech.files.wordpress.com/2020/10/screenshot-1942-07-19-at-12.54.54-am.png?w=501)List of cloned Apps. Pressing the zip icon exports the app sandbox data to external storage directory

Many of you might know that using Chrome developer tools on desktop can intercept tabs opened in Chrome browser on device. Likewise facebook developed an open source tool called [Stetho](http://facebook.github.io/stetho/) which help developers to access application sandbox data, intercept network etc via Chrome Desktop browser. This is a useful tool for developers. But will it be useful for reverse engineers? Let’s explore it in the next section!

### Stetho for Reverse Engineers

Stetho is an android library project. When enabled, developers have access to the application specific data on a non-rooted device via Chrome desktop browser. Stetho requires few lines of code to initialize it and rest of the magic is taken care by the stetho library. Because of its easiness to use, it is a boon for developers and also for reverse engineers. For a reverse engineer, an application need not have stetho embedded, but he/she can inject the Stetho library(read as smali code) into the application. The downside of this approach is, it requires tampering an application. But in the context of Cloning App, it is found to be possible to inject such libraries from host app to the guest apps without tampering the application. With little effort Stetho library can be injected in each and every guest application. Once the application is opened, on the Desktop Chrome browser, you can inspect the databases and shared preferences of each of the guest applications which is active. Here are the steps to use Chrome developer tool provided by the Chrome browser when Stetho is used with Cloning Apps

*   Connect your device to desktop with USB.
*   Open the Desktop Chrome browser and type [chrome://inspect/#devices](https://inspect/#devices).
*   You will see the mobile device model
*   Open any guest app which you wanted to inspect
*   You will start seeing the application(s) that can be inspected

![](https://darvincitech.files.wordpress.com/2020/10/stetho.png?w=673)List of Apps that can be inspected via Chrome Dev tool in Desktop

*   Press the inspect button corresponding to the app opened on device.
*   Boom! You can find the databases and shared preferences of the opened app(provided it created database and shared preferences)

![](https://darvincitech.files.wordpress.com/2020/10/stetho-google.png?w=820)Inspecting Google Authenticator database shows OTP Seed in clear

You can also modify the data inside databases or shared preferences. Stetho also provides dumpapp tool, which allows you to list the files in app’s sandbox or dump specific folder/file or the entire sandbox data similar to what Conware app can do. I slightly modified the PoC App to inject Stetho on the cloned apps and you can easily view the application sandbox data in Chrome Desktop Browser and also can access the files via dumpapp utility.

![](https://darvincitech.files.wordpress.com/2020/10/duo-filetree.png?w=731)Using dumapp utility of Stetho to view the list of files in an application sandbox

For another demo, I used Duo Authenticator app. This app also stores the OTP Seed in clear in the application sandbox which makes it easier to show how the secrets can be fetched through Stetho library.

![](https://darvincitech.files.wordpress.com/2020/10/duo_stetho.png?w=820)Dumping accounts.json file shows the OTP Seed in clear

Debugging a release APK
-----------------------

In Android, an app can be made debuggable by just marking the Manifest file with app:debuggable=true. However this requires tampering the application. Though apktool comes in handy, it has its own bugs when processing certain APKs. When guest apps are cloned inside the host app, it inherit the properties of the host App. So, if a host app is debuggable, then all cloned apps become debuggable too. Now with the help of [smalidea](https://github.com/JesusFreke/smali/wiki/smalidea), you can debug the smali code of cloned applications without tampering the application. For a demo, I used the Duo Authenticator App. Decompiled the dex to smali files and then imported onto Android Studio. Using the smalidea plugin it is possible to debug an app built for release.

![](https://darvincitech.files.wordpress.com/2020/10/debug.png?w=820)Shows debug breakpoint hit on an app downloaded from playstore and cloned inside the PoC app

Network interception
--------------------

From Android 7.0, [Network Security Configuration](https://developer.android.com/training/articles/security-config)(NSC) feature lets app customize their network security settings in a safe, declarative configuration file without modifying app code. Though it is useful for developers, it is a boon for reverse engineers too, as they just need to tamper with the configuration file and then to inject self-signed CA instead of requiring to root the device and tamper with the system trust store. But the main downside is, application needs to be tampered. In the context of cloning apps, just having a network security configuration set for the host app, all the guest apps inherit this configuration even if the apps had a specific NSC. You are all set for performing a Man-In-The-Middle attack without repackaging the application. Now the network can be intercepted with your choice of tools (such as Burp Suite, Charles Proxy). If some apps perform additional certificate validation with system trust store or perform Certificate Pinning other than by using NSC, then one has to resort to other mechanisms of bypassing ssl pinning with dynamic instrumentation tools such as Frida, xposed etc. With the [PoC app](https://github.com/darvincisec/VirtualDynamicAnalysis) and Burp Suite, some network packets can be intercepted. Because of some hitches, the demo does not work seamlessly. However the point to be noted is, it is possible to intercept traffic of guest applications even if the host application has NSC set. In the PoC app, following changes are made

1.  Created network-security-config.xml. It is intentionally made weak so that guest apps inherit this configuration

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config xmlns:android="http://schemas.android.com/apk/res/android">
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="@raw/cacert"/>
            <certificates src="system"/>
            <certificates src="user"/>
        </trust-anchors>
    </base-config>
</network-security-config>


```

2. Added the below configuration in AndroidManifest.xml

```
android:networkSecurityConfig="@xml/network_security_config"

```

3. Pushed the Burp suite certificate got via [http://burp/cert](http://burp/cert) on my PC and stored it as cacert in raw resource folder

Now you are all set for performing MITM attack on the cloned apps.

Dynamic Instrumentation with Frida
----------------------------------

In the previous blog post, I have covered how fake crytographic libraries can be executed in the context of a cloned application. Just above, we see how an injected Stetho java library can be used to snoop into application sandbox. On the same lines, the next thing I tried is to launch a native library in the context of the application. This also worked well in the [PoC app](https://github.com/darvincisec/VirtualDynamicAnalysis/). Here I used [frida-gadge](https://frida.re/docs/gadget/)t as its a default choice for dynamically instrumenting an application on a non-rooted device. I also used an accompanying configuration file like below.

```
{
  "interaction": {
    "type": "script-directory",
    "path": "/data/local/tmp/frida-scripts",
    "on_change": "rescan"
  }
}


```

This configuration file is stored as libgadget.config.so(the name has to correspond to how you named the frida gadget .so file), so that frida loads the scripts directly from the given path. With these changes done on the Cloning App, the frida gadget library is loaded in the guest app process memory as and when the application is loaded. Push all the frida scripts to the configured path and make them as executable and voila, you can now instrument the cloned applications.

For a demo, modified publicly available [frida scripts](https://github.com/FSecureLABS/android-keystore-audit/tree/master/frida-scripts) such that the output can be seen in logcat. With AndOTP App, you can find the frida intercepted data in logcat

![](https://darvincitech.files.wordpress.com/2020/10/frida.png?w=820)

### Virtual Dynamic Analysis – PoC App

[Virtual Dynamic Analysis](https://github.com/darvincisec/VirtualDynamicAnalysis) is a modified Virtual App to enable dynamic analysis without requiring to tamper applications on the same lines as VirtualXposed. This is just a PoC app and it works well only on certain Android versions (v7.x and v8.x)

With this app following are the **list of analysis** that can be done **_without tampering_** app.

1.  Inject Security Provider and fetch cryptographic assets of application depending on default security provider
2.  Fetch app sandbox data via Chrome desktop browser or via Conware
3.  Enable debugging on release applications
4.  Enable MITM attacks by providing a network security configuration (Works on Android 7.x)
5.  Dynamically instrument application using Frida-gadget (Works on Android 7.x)

Mitigations
-----------

The mitigations defined in the previous blog post holds good for the new attacks introduced in this post. There can be other mitigations such as having certificate pinning, anti-debug, anti-frida, but the foremost thing to address is to detect and prevent sensitive applications from running in such environment.

Executing on untrusted environment
----------------------------------

Largely, applications just hinge on the fact that the device environment is always safe and hence not much care is taken for protecting the most sensitive user specific assets leave aside incorporating Runtime Application Self Protection(RASP). It is surprising to see, even the popular 2FA applications store sensitive assets(OTP Seed) in clear. Though there is less to no chance for users to install such 2FA apps inside Cloning Apps, but there is an inherent risk in such apps when users use them on unsafe devices. Leaving this decision to users does not sound convincing as not all users are security conscious and may not be aware of the security gaps in each of the popular apps. Though users definitely have a role, I strongly feel, application developers developing security sensitive applications have a larger responsibility to protect their application specific sensitive assets.

  
_**Thanks for your time!**_