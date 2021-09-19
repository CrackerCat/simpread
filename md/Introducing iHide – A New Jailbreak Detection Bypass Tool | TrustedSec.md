> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.trustedsec.com](https://www.trustedsec.com/blog/introducing-ihide-a-new-jailbreak-detection-bypass-tool/)

> TrustedSec's blog is an expert source of information on information security trends and best practice......

September 2, 2021

Today, we are releasing iHide, a new tool for bypassing jailbreak detection in iOS applications. You can install iHide by adding the repo [https://repo.kc57.com](https://repo.kc57.com/) in Cydia or clicking [here](https://cydia.saurik.com/api/share#?source=https://repo.kc57.com) on an iOS device with Cydia installed. Additionally, you can check out the [code](https://github.com/Kc57/iHide) and build/install it yourself if you prefer.

Once installed, iHide will add a new entry in the iOS settings pane that can be used to enable/disable bypassing common jailbreak detection methods. Simply enable iHide, select any applications to enable it for, and iHide will attempt to bypass common jailbreak detection techniques. This should be all you need to get up and running.

![](https://d26v3d89gbih6n.cloudfront.net/wp-content/uploads/2021/09/Picture1.gif)Bypassing Jailbreak Detection

For details about the creation of iHide, read on!

Background
----------

During mobile engagements, we often test on rooted or jailbroken devices because this gives us the ability to do things like inspect Keychain, hook interesting code using [Frida](https://www.trustedsec.com/blog/mobile-hacking-using-frida-to-monitor-encryption/), or bypass certificate pinning with tools like [SSL Kill Switch 2](https://github.com/nabla-c0d3/ssl-kill-switch2). For the most part, this works great! Occasionally, though, we run into applications that implement some form of jailbreak detection.

What is Jailbreak Detection?
----------------------------

Jailbreak detection is any mechanism that a developer implements to identify whether the application is running on a jailbroken device. App developers may implement jailbreak detection for a variety of reasons, such as:

*   Protecting account information in banking apps
*   Securing sensitive data in Mobile Device Management (MDM) solutions
*   Preventing cheating in games, especially ones that are multiplayer or have in-game purchases

Once an application with jailbreak detection determines that it is running on a jailbroken device, it usually takes one of the following actions:

*   Alerts the user but still functions normally
*   Alerts the user and limits access to features or refuses to run altogether
*   Silently exits with no warning

![](https://d26v3d89gbih6n.cloudfront.net/wp-content/uploads/2021/09/Picture1.png)Jailbreak Detection Message

How Jailbreak Detection Works
-----------------------------

Jailbreak detection works by looking for differences between a stock and a jailbroken system. Some common things applications search for to determine where a device is jailbroken include:

*   Jailbreak-related apps like Cydia
*   Dylib files related to popular jailbreak tools and tweaks
*   Log files/plist files related to popular jailbreak tools and tweaks
*   Read/write access to directories not normally accessible
*   System function calls like fork() to check for unexpected behavior
*   A listening SSH service on the device

![](https://d26v3d89gbih6n.cloudfront.net/wp-content/uploads/2021/09/Picture2.png)Jailbreak Detection Code

How to Combat Jailbreak Detection
---------------------------------

There are several tools available to avoid jailbreak detection. Most of them work by hooking or ‘method swizzling’ to alter the logic or results from common jailbreak detections. Liberty Lite is a popular option and one that we use often. Occasionally, we do run across an application using a jailbreak detection technique that existing tools do not successfully bypass.

In this situation, we would typically use some combination of call tracing via Frida, live debugging, and static analysis to determine the detection technique. Sometimes it may be a technique that isn’t covered by the existing tools. Often it may be a known technique like checking for jailbreak related files, but the specific file identified by the jailbreak detection logic isn’t hidden by the existing tools.

Once we identify the technique, we would create a custom Frida script or use a debugger to hide a specific file or modify the jailbreak detection logic to always return successfully. While this works, it isn’t as convenient or as portable as using an iOS tweak.

Creating iHide
--------------

With a jailbreak detection technique identified, the goal was to implement the bypass into a tool that could be easily installed/updated on a device and easily enabled/disabled when needed on a per-app basis. I also wanted the tool to be usable without being tethered to a MacBook running a Frida script or debugger. Finally, the goal was to have something open source where new bypasses could be added as needed.

I figured this would be the perfect opportunity to learn some tweak development using [Theos](https://github.com/theos/theos). If you haven’t heard of Theos yet, it is a cross-platform build environment that includes several templates that can be used for iOS tweak development. I started working on building out a tweak and integrating some of the bypasses from my Frida scripts…and this is how iHide was born!

For the initial release, iHide likely will not work on everything. It bypasses several common detection techniques in applications we needed it for, such as Microsoft Intune and some others, where the available tools we tested did not work at the time. It can be installed alongside other jailbreak hiding tools (although I would recommend only enabling one at a time). If your tool of choice does not currently work for something, you can try using iHide.

If you run into applications that are not supported, feel free to open a request on the GitHub project—I may look at adding support as I have time. If you want to updates or have any questions, feel free to reach out on Twitter [@_Kc57](https://twitter.com/_Kc57) or the [TrustedSec Discord](https://discord.gg/trustedsec).