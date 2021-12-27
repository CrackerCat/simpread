> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [sensepost.com](https://sensepost.com/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/)

> Leaders in Information Security

With the release of windows 11, Microsoft announced the **[Windows Subsystem for Android](https://docs.microsoft.com/en-us/windows/android/wsa/)** or WSA. This following their previous release, **[Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install)** or WSL. These enable you to run a virtual Linux or Android environment directly on your Windows Operating System, without the prerequisite compatibility layer provided by third-party software. In this post I’ll show you how to use WSA for Android mobile application pentesting, such that you can perform all of the usual steps using only Windows, with no physical Android device or emulator needed.

The WSA experience is great; when installing applications within WSA, they are automatically available to the Windows host. Likewise, all files and resources from the Windows host can interact with the Android Subsystem. In the past, Android emulation has been notoriously tricky. Third party applications always required you to place your trust somewhere else, and have always been slow and resource heavy.

This got me thinking about how WSA could potentially be used for Android security testing, without the need for an Android device. In order for this to be successful, I would need to be able to install, patch, and run Android applications, as well as intercept traffic… in same manner as on a physical Android device. This would serve two purposes – the first being removing the need for a potentially expensive device, and secondly creating a uniform and reproducible testing platform.

The following applications are often needed to conduct Android application security testing. Not all of them are not necessary for WSA itself, but are useful for Android testing and connectivity in general.

*   [Android Studio](https://developer.android.com/studio) is a must for any Android testing. It provides a debugger, and a way to view the components of an APK in an easily digestible format.

*   [Android SDK Tools](https://developer.android.com/studio/releases/platform-tools) are needed for the [Android Debug Bridge](https://developer.android.com/studio/command-line/adb) (ADB), which will be used to connect to WSA from the command line, transfer files, and install applications.

*   We will be using [Objection](https://github.com/sensepost/objection) to patch APKs prior to installation as well as to perform some instrumentation of the target application. Objection has various prerequisites which are mentioned in the [Wiki](https://github.com/sensepost/objection/wiki/Patching-Android-Applications) and should be installed as well.
*   Windows 11 – with the virtual machine platform enabled in the Windows Features settings.

*   Lastly, we’ll be needing [Python](https://www.python.org/downloads/windows/) to run objection.

On my system, I added the paths to all of the above-mentioned tools to my Windows environment variables so that they can be run from any directory. I recommend this for a smoother experience. It can be done by clicking Start, and then typing `environmental variables`, and editing the system-wide variables.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Env_variables-1-1024x445.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Env_variables-1.png)Adding all the previously installed tools to your PATH environmental variable

Microsoft has detailed instructions on how to install “base” WSA [here](https://docs.microsoft.com/en-us/windows/android/wsa/).

At the time of writing this post, in order to install WSA directly from the Microsoft Store, you need to be part of the [Windows Insider Program](https://insider.windows.com/), and running on the Beta channel. This will change in future releases. It is also currently only possible to install a small subset of applications from the Amazon Appstore, which is installed from the Microsoft Store [here](https://www.microsoft.com/store/productId/9P3395VX91NR) – it’s just stores all the way down!

These limitations don’t help us when a client sends an APK or AAB bundle, so we need a way to side load applications, the same as we would on a regular physical device. Thankfully this is made possible following my second installation method below.

Installation method 1 – Limited WSA
-----------------------------------

For demonstration purposes, let’s see how to install Microsoft’s limited WSA distributed through the insider program. We won’t modify it to allow Google Play Services to work, or to side load applications, that’s done in the next section, and you’ll have to uninstall this. So this is really just to show the difference. The [MSIXBundle](https://docs.microsoft.com/en-us/windows/msix/packaging-tool/bundle-msix-packages) that you will get when signing up for the Insiders Program file can be discovered using [this](https://store.rg-adguard.net/) service. This site is a link aggregator for downloading directly from the Microsoft store. Credits to [RG_Adguard](https://twitter.com/rgadguard?lang=en) for this.

Within the URL field, enter the following (which is the URL for the Amazon Appstore):

```
https://www.microsoft.com/store/productId/9P3395VX91NR

```

Then select the SLOW channel and hit search. This should reveal the URL to download the WSA application directly from Microsoft.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Download_msix-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Download_msix-1.png)Make sure to select the SLOW channel

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Download_msix_2-1-1024x133.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Download_msix_2-1.png)Select the 1.2GB MSIXbundle file

Once downloaded, open Windows Terminal as Administrator and run the following:

```
Add-AppxPackage -Path "C:\path\to\wsa.msixbundle"

```

This should then give you the Windows Subsystem for Android in your Windows start menu.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_start.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_start.png)WSA Settings should now be available in your start menu.

Once the application is open, you’ll see a couple of options. The first is choosing to have the subsystem resources available as needed, or continuously available. This is a personal preference. I chose to have them running continuously, as I have resources to spare.

The second option is not personal preference. You MUST enable developer mode within WSA, or you will not be able to connect via the Android Debug Bridge or [ADB](https://developer.android.com/studio/command-line/adb).

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_enable-1-1024x459.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_enable-1.png)Ensure Developer Mode is enabled.

You will see that you currently have no IP assigned, and this is expected. Even though the WSA settings are open, WSA itself is not necessarily running.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_No_IP-1-1024x313.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_No_IP-1.png)IP Address is unavailable currently.

In order to actually start WSA, you need to click the open icon next to the FILES option, which will present you with a very basic Android file browser GUI.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_files-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_files-1.png)

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_Starting.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_Starting.png)

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_Files_open.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_Files_open.png)This is the screen you’ll see once the subsystem is running, and the “device” has started.

At this point you’ll have a basic WSA installation up and running. However, it is limited to the Amazon app store, and that is not very useful for what we want to do!

Installation method 2 – Google Play enabled WSA with root
---------------------------------------------------------

Let’s redo the installation step-by-step using a method which provides more functionality, such as the ability to install applications directly from Google Play within Windows, the ability to sideload applications, and the ability to Root your WSA “device”.

Firstly, if you were previously playing around with WSA, or you followed my initial method, you’re going to need to uninstall all traces of WSA from your system.

Next, Open your Windows settings and enable Developer Mode within Windows itself, as opposed to within WSA (although both are needed). This will allow you to install third party application bundles from outside of the Microsoft Store.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Developer_Settings-2.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Developer_Settings-2.png)

Next you’ll need to create and download a patched WSA installer and platform tools. This can be done by following the description on the Github repo [here](https://github.com/LSPosed/MagiskOnWSA). In essence you will fork the repo, and specify the version of the open source Google Play Store you want by name, based on [OpenGApps](https://opengapps.org/). A small selection of the different variants descriptions are below:

*   [Pico](https://github.com/opengapps/opengapps/wiki/Pico-Package): This package is designed for users who want the absolute minimum GApps installation available.
*   [Nano](https://github.com/opengapps/opengapps/wiki/Nano-Package): This package is designed for users who want the smallest Google footprint possible while still enjoying native “Okay Google” and Google Search support.
*   [Stock](https://github.com/opengapps/opengapps/wiki/Stock-Package): This package includes all the Google Apps that come standard on Pixel smartphones.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/fork.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/fork.png)The forked repo on my gitHub

Once forked, click Actions, and click on the `Build WSA` workflow, specifying the OpenGApps version you selected from above. In summary, [the workflow](https://github.com/LSPosed/MagiskOnWSA/blob/main/.github/workflows/magisk.yml) will:

*   Download WSA from the Microsoft Store (via [RD-Adguard](https://store.rg-adguard.net/))
*   Download Magisk from the [Canary branch on the Magisk repo](https://raw.githubusercontent.com/topjohnwu/magisk-files/canary/app-debug.apk)
*   Download your specified version of [OpenGApps](https://opengapps.org/)
*   Unzip the OpenGApps
*   Mount all images
*   Integrate OpenGApps
*   Unmount all images
*   Shrink the resultant image
*   Integrate ADB
*   Package everything into the final zip file.

You can see this workflow in action in [this](https://github.com/LSPosed/MagiskOnWSA/blob/main/.github/workflows/magisk.yml) file.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Build-1024x514.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Build.png)Select the GApps version in the build options on the right hand side

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Build-complete.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Build-complete.png)The WSA Build is complete

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_gapps.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/WSA_gapps.png)The resultant file size will vary greatly based on the selection of GApps

You will see a download link included for a specific version of Magisk in the build options, which currently works 100% with WSA. [Magisk](https://magiskmanager.com/) is an application used to [Root](https://en.wikipedia.org/wiki/Rooting_(Android)) android devices. Rooting is NOT necessary for patching or intercepting to work, but does provide extra functionality which may be useful to you.

Extract the resultant WSA-with-magisk-GApps.zip contents and open a Terminal window as Admin within the unzipped directory. Then run:

```
Add-AppxPackage -Register .\AppManifest.xml

```

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_new-1-1024x179.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_new-1.png)

You can now re-open windows Subsystem for Android from the Windows Start menu. You’ll need to enable Developer Mode from within the WSA settings, and click on the open files button, just as before.

Back in the terminal, we can now run ADB (either directly from the platform tools directory, or from any directory if you modified your environment variables). Typing ADB devices will give a list of all the current available devices which will initially be empty. We’ll fix that in a moment.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_devices.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_devices.png)

Looking at the WSA settings, an IP address should now be configured.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/IP_address-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/IP_address-1.png)

We can connect to this “device” using _ADB connect `IP`_. We can then re-list the devices and confirm we have connectivity.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_de-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_de-1.png)

Installing applications is as simple as _ADB install ‘APK Name’_. We will be installing Magisk in this manner with _ADB install magisk.apk_. Once installed, click on Start and search for Magisk. As WSA shares resources with Windows, Magisk can be opened from the Start menu as if it is installed in windows directly, however it is running seamlessly under WSA.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Magisk_windows.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Magisk_windows.png)

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Magisk_settings.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Magisk_settings.png)

In the terminal, type _ADB shell_ to gain access to the WSA “device”. We can confirm the current user with _whoami_.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_shell.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_shell.png)

Typing `su` to become root will result in a pop-up within Magisk, asking to allow `root` access. Once allowed, the SuperUser tab within Magisk will show that we have granted `root` access within the ADB shell.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_SuperUser-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_SuperUser-1.png)

We can confirm we have rooted the device with `whoami`.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_SU-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/ADB_SU-1.png)

From this point, we have full read/write access to the WSA filesystem.

Objection works in much the same way from this point. Simply running `objection patchapk -s 'sourceapk'` will allow ADB to determine the device architecture the same way as it would from a mobile device via USB. I will be using the “Purposefully Insecure and Vulnerable Android Application” ([PIVAA](https://github.com/HTBridge/pivaa)) for demonstration purposes.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Objection_patch-1-1024x189.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Objection_patch-1.png)Objection successfully determined the x86_64 architecture, and patched the application accordingly.

Once patched, install the application with

`adb install .\pivaa.objection.apk`

As before, it is now available directly within Windows. However, starting the application will cause it to “pause” until we use Objection explore to hook into the [Frida](https://frida.re/) server.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/PIVAA.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/PIVAA.png)

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Objection_explore-1-1024x282.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Objection_explore-1.png)Objection successfully hooks into the Frida server

You will see this device show up as a “Pixel 5”, which seems to be the archetype for the virtual device, and may change in future updates. It’s important to note that no USB, nor a physical Pixel 5 are involved.

I now had a working WSA installation, a rooted device, patching with Objection, and hooking into running patched applications with Objection. The main question that I was faced with at this stage was how to do the final step – intercepting traffic with [Burp Suite](https://portswigger.net/burp).

As I had no way of seeing the virtual network settings or installing a certificate authority on the device, I decided to try installing an Android Launcher. A launcher is similar to a desktop environment on a Linux distribution, and is what you see when you unlock your physical android device. Some popular ones include Niagara, Nova, Lawnchair, and Stock launchers from the likes of Google and Samsung.

After trying multiple launchers, third party settings applications, and Windows specific proxy applications, I had the idea that the launcher with the highest probability of being compatible was the Microsoft Launcher. You can install it directly onto the device by using OpenGapps and using the official play store [link](https://play.google.com/store/apps/details?id=com.microsoft.launcher&hl=en_ZA&gl=US). I opted to side load the APK from [here](https://m.apkpure.com/microsoft-launcher/com.microsoft.launcher/download?from=details) (however multiple other APK mirroring sites exist, so use your preferred one), and installed it with `adb install`. This gave me exactly what I was looking for, a settings application.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Settings_microsoft_launcher-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Settings_microsoft_launcher-1.png)Other applications can be opened from the launcher directly, as opposed to the Windows Start menu

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Settings_android-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Settings_android-1.png)

Within the Network and Internet settings, I finally had a Virtual WiFi adapter, which I was able to set proxies for.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Virt_Wifi.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Virt_Wifi.png)

When in the VirtWiFi settings, I was able to select the _Advanced options_ to install certificates. Knowing this, I generated a certificate from Burp Suite and moved it to the “device” using:

```
adb push .\burpwsa.cer /storage/emulated/0/Download

```

I knew I had access to all folders after rooting with Magisk, but the Downloads folder was visible within the files application of WSA itself, so I selected this as the destination directory.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/BurpWSA.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/BurpWSA.png)

Installing the certificate needed to be done through the settings however.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_Certs-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_Certs-1.png)Click on advanced – Install certificates.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_certs_2-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_certs_2-1.png)

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_burpWSA.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Install_burpWSA.png)

Once installed, I set my proxy listener in Burp to that of my host Ethernet IP, and a port of my choosing as per usual.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Burp_proxy-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Burp_proxy-1.png)

Within the VirtWifi I edited the proxy to have the same settings.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Virt_proxy-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Virt_proxy-1.png)Click the pencil on the top right to edit the config.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Virt_proxy3-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Virt_proxy3-1.png)Match the proxy settings to Burp Suite

Now when running the PIVAA application, hook into it with Objection as before, however this time specifying `Android SSLPinning Disable` as the startup command, I was able to intercept the traffic from WSA.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/android_ssl_disable-1-1024x436.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/android_ssl_disable-1.png)An exception was thrown for TrustManager, indicating the SSLpinning was being bypassed

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Intercept4-1.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/Intercept4-1.png)Specifying Sensepost.com within the application.

[![](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/intercept2-1024x677.png)](https://sensepost.com/img/pages/blog/2021/android-application-testing-using-windows-11-and-windows-subsystem-for-android/intercept2.png)

Summary
-------

We can see that using the initial method, we’re severely limited to what we can do, but there’s always workarounds and more we can do!

Digging a bit deeper, we’re able to install WSA without using the Microsoft store, and without being on the Windows 11 Beta program, but that wasn’t quite where we needed to be.

Taking it one step further allows us to install any application we want, and obtain root privileges on WSA itself, which is a critical step in terms of security testing Android.

The final step from here, which undoubtedly took the longest, was figuring out how to intercept traffic from this virtual device in Burpsuite, in order to perform dynamic testing, and modify the data in transit, but with this done, there is now no longer a need for a physical android device to perform security testing on Android applications.

Please feel free to reach out to me with any questions you may have. This is still a new and evolving technology, and I hope to add to this as new methods emerge.