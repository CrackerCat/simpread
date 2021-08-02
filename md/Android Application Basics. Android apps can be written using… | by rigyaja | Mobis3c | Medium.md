> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/mobis3c/android-application-basics-b4da5aaa3e68)

> Android apps can be written using Kotlin, Java, and C++ languages. Android application components are......

**Android Application Basics**
==============================

[![](https://miro.medium.com/fit/c/96/96/0*skW74tBKNAPUakbV.jpg)](https://rigyaja.medium.com/?source=post_page-----b4da5aaa3e68--------------------------------)[rigyaja](https://rigyaja.medium.com/?source=post_page-----b4da5aaa3e68--------------------------------)Follow[Feb 7](/mobis3c/android-application-basics-b4da5aaa3e68?source=post_page-----b4da5aaa3e68--------------------------------) · 7 min read![](https://miro.medium.com/max/1400/0*vwqww0ZBqK3mtp18.jpg)

Android apps can be written using Kotlin, Java, and C++ languages. The Android SDK tools is used to compile the code written by the developer along with any data and resource files associated with the code into an APK, an _Android package_, which is an archive file with an `.apk` suffix.

One APK file contains all the contents of an Android app and is the file, Android-powered devices use to install the app.

Each Android app lives in its own security sandbox, protected by the following Android security features:

*   The Android operating system is a multi-user Linux system in which each app is a different user. Ex: if 40 apps are installed in your mobile, then there are 40 users for your mobile.
*   By default, the system assigns each app a unique Linux user ID (the ID is used only by the system and is unknown to the app). Ex: for twitter the system may assign ID 10, Facebook → 12, Instagram → 13 so on. Only the system knows these IDs.
*   The system sets permissions for all the files in an app so that only the user ID assigned to that app can access them. Ex: ID no 12 can only access files related to Facebook app.
*   Each process has its own virtual machine (VM), so an app’s code runs in isolation from other apps. Ex: run ps -A in adb shell.

![](https://miro.medium.com/max/60/1*eJ5kUrdAu9-RQM7q1kTIdw.png?q=20)![](https://miro.medium.com/max/870/1*eJ5kUrdAu9-RQM7q1kTIdw.png)![](https://miro.medium.com/max/1392/1*eJ5kUrdAu9-RQM7q1kTIdw.png)adb shell used to list all the processes running in the device.

*   By default, every app runs in its own Linux process. The Android system starts the process when any of the app’s components need to be executed, and then shuts down the process when it’s no longer needed or when the system must recover memory for other apps.

![](https://miro.medium.com/max/60/1*MI8Ap5b3YMJE-VV07FC7Sg.png?q=20)![](https://miro.medium.com/max/875/1*MI8Ap5b3YMJE-VV07FC7Sg.png)![](https://miro.medium.com/max/1400/1*MI8Ap5b3YMJE-VV07FC7Sg.png)process of twitter apk (com.twitter.android) is running as USER: u0_a48, having PID 5137.![](https://miro.medium.com/max/60/1*WsTxeI_YjHM4dYW3CDoeBQ.png?q=20)![](https://miro.medium.com/max/875/1*WsTxeI_YjHM4dYW3CDoeBQ.png)![](https://miro.medium.com/max/1400/1*WsTxeI_YjHM4dYW3CDoeBQ.png)After it is closed, the process is stopped.

The Android system uses the **_principle of least privilege_**. That is, each app, by default, has access only to the components that it requires to do its work and no more. This creates a very secure environment in which an app cannot access parts of the system for which it is not given permission.

But, there are ways through which an app, can share its data with other apps and also access system services:

*   **Apps sharing same Linux user ID can access each other’s files**:

It’s possible to arrange for two apps to share the same Linux user ID, in which case they are able to access each other’s files. To conserve system resources, apps with the same user ID can also arrange to run in the same Linux process and share the same VM. The apps must also be signed with the same certificate.

*   **Apps use system services such as location, camera and Bluetooth:**

An app can request permission to access device data such as the device’s location, camera, and Bluetooth connection. The user has to explicitly grant these permissions.

**Components of Application**
=============================

App components are the essential building blocks of an Android app. Each component is an entry point through which the system or a user can enter your app. Some components depend on others.

There are four different types of app components:

*   Activities
*   Services
*   Broadcast receivers
*   Content providers

**Android Intent** is the _message_ that is passed between components such as activities, content providers, broadcast receivers and services

**Activities**

An _activity_ represents a single screen with a user interface. It is the entry point for interacting with the user.

For example, an social media app might have one activity to post tweets, or images, another activity that shows a live feed, and another activity for chat.

These activities work together to form a cohesive user experience in the social media app, But each one is independent of the others. As such, a different app can start any one of these activities if the social media app allows it.

For example, a camera app can start the activity in the social media app to post images and allow the user to share a picture. An activity allows many key interactions between system and app including the following:

*   **Tracking activities**: To ensure that the system keeps running the process that is hosting the activity, it keeps track of what the user currently cares about (what is on screen).
*   **Prioritise Processes:** Knowing that previously used processes contain things the user may return to (stopped activities), and thus more highly prioritise keeping those processes around.
*   **Kills inactive process:** Helping the app handle having its process killed so the user can return to activities with their previous state restored.
*   Providing a way for apps to implement user flows between each other, and for the system to coordinate these flows. (The most classic example here being share.)

**Services**

A _service_ is a general-purpose entry point for keeping an app running in the background for all kinds of reasons. It is a component that runs in the background to perform long-running operations or to perform work for remote processes. A service does not provide a user interface.

For example, a service might play music in the background while the user is in a different app, or it might fetch data over the network without blocking user interaction with an activity. Another component, such as an activity, can start the service and let it run or bind to it in order to interact with it.

There are actually two very distinct semantics services tell the system about how to manage an app: Started services tell the system to keep them running until their work is completed.

This could be to sync some data in the background or play music even after the user leaves the app. Syncing data in the background or playing music also represent two different types of started services that modify how the system handles them:

*   **User Aware Service**

Ex: Music playback service, Here the app tells the system by saying it wants to be foreground with a notification to tell us about it.

*   **User Unaware Service**

Ex: A regular background service is not something the user is directly aware as running, so the system may allow the service to be killed (and then restarting the service sometime later) if it needs RAM for things that are of more immediate concern to the user.

**Broadcast receivers**

A _broadcast receiver_ is a component that enables the system to deliver events to the app outside of a regular user flow, allowing the app to respond to system-wide broadcast announcements.

The system delivers broadcasts even to apps that aren’t currently running. for ex, I will set an alarm at 6 AM in the Clock app and close the app, It wants to schedule an alarm to post a notification to tell me to wake up at 6 AM. Now, the system delivers that alarm to a BroadcastReceiver of the clock app and then the app rings accordingly.

![](https://miro.medium.com/max/60/1*7uc9fzQ5Iy1lKAjttByMLQ.png?q=20)![](https://miro.medium.com/max/875/1*7uc9fzQ5Iy1lKAjttByMLQ.png)![](https://miro.medium.com/max/1400/1*7uc9fzQ5Iy1lKAjttByMLQ.png)

An application has to register for intent to receive broadcasts from the system.

Broadcasts can be originated from both (system and apps):

*   System broadcasts — for example, a broadcast announcing that the screen has turned off, the battery is low, or a picture was captured.
*   Apps broadcasts — for example, to let other apps know that some data has been downloaded to the device and is available for them to use.

**Content providers**

A _content provider_ manages a shared set of app data that you can store in the file system, in a SQLite database, on the web, or on any other persistent storage location that your app can access.

![](https://miro.medium.com/max/60/1*U9H4CRv-jXYT-culNt1ToA.png?q=20)![](https://miro.medium.com/max/875/1*U9H4CRv-jXYT-culNt1ToA.png)![](https://miro.medium.com/max/1400/1*U9H4CRv-jXYT-culNt1ToA.png)

Through the content provider, other apps can query or modify the data if the content provider allows it. For Ex: A camera app will store all the photos in file storage, now a social media app which already has file storage read access permission queries the content provider in order to access the photos taken by the camera app.

A content provider is an entry point into an app for publishing named data items, identified by a URI scheme. Apps have control over the definition of URIs. Other apps have to query the content provider using URI.

To query a content provider, an app can specify the query string in the form of a URI which has following format to access the data.

```
<prefix>://<authority>/<data_type>/<id>

```