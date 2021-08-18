> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [valsamaras.medium.com](https://valsamaras.medium.com/the-application-sandbox-9abd09a5c6da)

> Many people who are starting to work with the Android OS are having difficulties to understand the ap......

The Application Sandbox
=======================

[![](https://miro.medium.com/fit/c/35/35/0*4uechiEB5YonWvdZ)](/?source=post_page-----9abd09a5c6da--------------------------------)[

+Ch0pin

](/?source=post_page-----9abd09a5c6da--------------------------------)[

3 days ago·6 min read

](/the-application-sandbox-9abd09a5c6da?source=post_page-----9abd09a5c6da--------------------------------)

Many people who are starting to work with the Android OS are having difficulties to understand the application sandbox concept. This usually leads to misconceptions in respect to data and resource sharing between the apps which by its turn leads to unsubstantial findings and false security alarms.

The main objective behind this article is to demystify concepts regarding one of the most important security features of Android as well as to give answers to questions like:

*   Why a cleartext username/password that you found in the shared_prefs folder is not a critical finding :) ?
*   Is it possible for an application that I just installed to “steal” my personal data ?
*   Is it possible for an app to read or write data from another app ?
*   Why it is a risk to root my device ?

The good old UNIX-Style permission model
========================================

Android is an operating system which is vastly based in Linux. This being said it is not a surprise to realise that many of Linux’s security features have been imported and modified in order to fit the needs of a mobile device. No Androidisms here… when it comes to file permissions the good old UNIX-style permission model is here to stay.

As this is an Android related post I won’t insist much in to how this permission model works. The need-to-know though can be easily summarised as follows:

*   The Unix security model is based on the **discretionary access control** (DAC) model, which enables users to decide who can access the resources that they own.
*   The list of entities that may access a file consists of an **owner** a **group** and the rest of the world, or else the **others.** What an entity can do with a file is defined by a **read, write** and **execute** permission:

![](https://miro.medium.com/max/38/1*6FpqEaMFBRX9gmhv-bNbiQ.png?q=20)![](https://miro.medium.com/max/875/1*6FpqEaMFBRX9gmhv-bNbiQ.png)Photo by [http://z.cliffe.schreuders.org/edu/ADS/Access%20Controls.pdf](http://z.cliffe.schreuders.org/edu/ADS/Access%20Controls.pdf)

**Consider the above as well as the multi-user support of the Linux Kernel and you have the Android Sandbox.**

The Application-As-User approach
================================

When I am asked to simplify the concept of the application sandbox, the first thing that comes to my mind is a **high security prison**. Inmates kept in their dedicated cells while there is no way for them to communicate with each other besides a well defined inter-cell communication system. Add the fact that in order to use the prison’s facilities the inmates need to get the director’s permission and there you go…

By the time that an application is installed, it is assigned with a **User Identity** (**UID**) or since we are talking about an application an **Application Identity** (**AID**). The Android OS uses a UID above 10000 for normal apps:

![](https://miro.medium.com/max/38/1*x7UFtrGrvVTZvCvpqsiS5A.png?q=20)![](https://miro.medium.com/max/875/1*x7UFtrGrvVTZvCvpqsiS5A.png)

The list of the UIDs is statically defined → [**here**](https://android.googlesource.com/platform/system/core/+/08c370c/include/private/android_filesystem_config.h)**.** Although the traditional passwd file is missing each application has a user name which is defined in the format u[**uid**]_a[**offset**]. The uid in that case is the id of the user account, while the offset is simply the value AID minus 10000. In the following screenshots the application com.google.android.gms has the username u0_a157 and AID = 10157:

![](https://miro.medium.com/max/38/1*Wi9X33OAH_n0L84A9rK26A.png?q=20)![](https://miro.medium.com/max/875/1*Wi9X33OAH_n0L84A9rK26A.png)

The /data/data and memory isolation
===================================

Each application is executed in a dedicated process and is assigned with a dedicated data directory. The **/data/data/com.google.android.gms/** (or more general the /data/data/[package_name]) folder in the screenshot above, will contain all the operational data of the app, while only the user u0_a157 will be able to access it. To get more details regarding the application’s ID and groups you may use the /proc/[pid]/status file:

![](https://miro.medium.com/max/38/1*YoQ5GR5G87T-g7BM6kuSHA.png?q=20)![](https://miro.medium.com/max/875/1*YoQ5GR5G87T-g7BM6kuSHA.png)

The **Groups** **label** refers to the groups that the application belongs and it is related with the permission set which has been approved for the particular app. By referring to the [android_filesystem_config.h](https://android.googlesource.com/platform/system/core/+/08c370c/include/private/android_filesystem_config.h) you may get an idea of what type of permission the application is using:

![](https://miro.medium.com/max/38/1*DpC7CrGEHY2c2xMpTKdblA.png?q=20)![](https://miro.medium.com/max/875/1*DpC7CrGEHY2c2xMpTKdblA.png)

It should be clear by now that by the time that a permission is granted for an app, the corresponding GID will be added to the process of the app. The mapping between the permissions looks like bellow:

![](https://miro.medium.com/max/38/1*fxDEnoc_V2HsBZrAFhb5ZQ.png?q=20)![](https://miro.medium.com/max/765/1*fxDEnoc_V2HsBZrAFhb5ZQ.png)

while the full list can be found → [here](https://android.googlesource.com/platform/frameworks/base/+/master/data/etc/platform.xml)

Sandbox Sharing
===============

Imagine that someone is developing two applications A and B and wants a more loose communication between them. There are two main aspects that should be taken under consideration, the application’s signing certificate and the shared user id. The first is pretty clear, **application A must be signed with the same certificate as application B**. The second regards the android manifest and more specifically the ‘**sharedUserId**’ tag, which should be explicitly defined as bellow:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.foo.A"
          android:sharedUserId="com.foo.B">
    . . .
</manifest>

```

It has to be mentioned that this constant was deprecated in API level 29, while is strongly encouraged by the Android Developer’s page to use proper communication mechanisms, such as services and content providers, to facilitate interoperability between shared components. [More](https://source.android.com/security/app-sandbox) specifically:

*   If your app needs to share files with another app, use a [content provider](https://developer.android.com/guide/topics/providers/content-provider-basics.html). Content providers share data with the proper granularity and without the many downsides of world-accessible UNIX permissions (for details, refer to [Content provider basics](https://developer.android.com/guide/topics/providers/content-provider-basics.html)).
*   If your app has files that genuinely should be accessible to the world (such as photos), they must be media-specific (photos, videos, and audio files only) and stored using the [MediaStore](https://developer.android.com/reference/android/provider/MediaStore) class. (For more details on how to add a media item, see [Access media files from shared storage](https://developer.android.com/training/data-storage/shared/media#add-item).)

Privileged Applications require special permissions
===================================================

System apps are pre-installed apps in the system partition with your ROM. In other words, a system app is simply an app placed under ‘/system/app’ folder on an Android device. ‘/system/app’ is a read-only folder. Android device users do not have access to this partition.

Another interesting characteristic of the system apps which is relative to this article relies on the **special UID**. An application may be assigned with a special UID in order to gain access to resources that otherwise won’t be able to access. As mentioned in the previous paragraphs, these special IDs belong to the range 1000 to 1999 and may be retrieved by the time that the application is signed with the platform key (by adding a LOCAL_CERTIFICATE := platform to the Android.mk) as well as adding the android:sharedUserId tag in the android manifest:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.foo.myApp"
          android:sharedUserId="android.uid.nfc">
    . . .
</manifest>

```

Takeaways
=========

*   Each Android application operates in its own isolated environment which is called sandbox.
*   It is not possible for an unprivileged application to access the sandbox (thus the data) of another app.
*   Saving cleartext sensitive data in the application’s data directory is generally considered safe assuming that the corresponding programming best practices are followed.
*   A ‘rooted’ device imposes a great risk, since by the time an application will gain root permission can invade another application’s sandbox.