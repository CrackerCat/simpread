> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/mobis3c/introduction-to-android-security-64609edeb18c)

> Android is an open source, Linux-based software stack created for a wide array of devices and form fa......

Introduction to Android Security
================================

[![](https://miro.medium.com/fit/c/96/96/1*uTzz4c17GKaaSp47mMSkgw.jpeg)](https://3kal.medium.com/?source=post_page-----64609edeb18c--------------------------------)[Kal](https://3kal.medium.com/?source=post_page-----64609edeb18c--------------------------------)Follow[Feb 1](/mobis3c/introduction-to-android-security-64609edeb18c?source=post_page-----64609edeb18c--------------------------------) · 6 min read[](/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F64609edeb18c&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fmobis3c%2Fintroduction-to-android-security-64609edeb18c&source=post_actions_header--------------------------bookmark_preview-----------)![](https://miro.medium.com/max/1400/0*NVdFlOeUhKB8EfHP.jpg)Android Security

**Overview of Android Architecture**
====================================

Android is an open source, Linux-based software stack created for a wide array of devices and form factors. The following diagram shows the major components of the Android platform.

![](https://miro.medium.com/max/40/0*tRnHxuNNHER6LoZC.png?q=20)![](https://miro.medium.com/max/875/0*tRnHxuNNHER6LoZC.png)![](https://miro.medium.com/max/1400/0*tRnHxuNNHER6LoZC.png)Android Software Stack

**The Linux Kernel**

The foundation of the Android platform is the Linux kernel. For example, the Android Runtime (ART) relies on the Linux kernel for underlying functionalities such as threading and low-level memory management.

Using a Linux kernel allows Android to take advantage of key security features and allows device manufacturers to develop hardware drivers for a well-known kernel.

**Hardware Abstraction Layer (HAL)**

The hardware abstraction layer (HAL) provides standard interfaces that expose device hardware capabilities to the higher-level Java API framework. The HAL consists of multiple library modules, each of which implements an interface for a specific type of hardware component, such as the camera or Bluetooth module. When a framework API makes a call to access device hardware, the Android system loads the library module for that hardware component.

**Android Runtime**

For devices running Android version 5.0 (API level 21) or higher, each app runs in its own process and with its own instance of the Android Runtime (ART). ART is written to run multiple virtual machines on low-memory devices by executing DEX files, a bytecode format designed specially for Android that’s optimized for minimal memory footprint. Build toolchains, such as Jack, compile Java sources into DEX bytecode, which can run on the Android platform.

Some of the major features of ART include the following:

*   Ahead-of-time (AOT) and just-in-time (JIT) compilation
*   Optimized garbage collection (GC)
*   On Android 9 (API level 28) and higher, conversion of an app package’s Dalvik Executable format (DEX) files to more compact machine code.
*   Better debugging support, including a dedicated sampling profiler, detailed diagnostic exceptions and crash reporting, and the ability to set watchpoints to monitor specific fields

Prior to Android version 5.0 (API level 21), Dalvik was the Android runtime. If your app runs well on ART, then it should work on Dalvik as well, but the reverse may not be true.

Android also includes a set of core runtime libraries that provide most of the functionality of the Java programming language, including some Java 8 language features, that the Java API framework uses.

**Native C/C++ Libraries**

Many core Android system components and services, such as ART and HAL, are built from native code that require native libraries written in C and C++. The Android platform provides Java framework APIs to expose the functionality of some of these native libraries to apps. For example, you can access OpenGL ES through the Android framework’s Java OpenGL API to add support for drawing and manipulating 2D and 3D graphics in your app.

If you are developing an app that requires C or C++ code, you can use the Android NDK to access some of these native platform libraries directly from your native code.

**Java API Framework**

The entire feature-set of the Android OS is available to you through APIs written in the Java language. These APIs form the building blocks you need to create Android apps by simplifying the reuse of core, modular system components and services, which include the following:

*   A rich and extensible View System you can use to build an app’s UI, including lists, grids, text boxes, buttons, and even an embeddable web browser
*   A Resource Manager, providing access to non-code resources such as localized strings, graphics, and layout files
*   A Notification Manager that enables all apps to display custom alerts in the status bar
*   An Activity Manager that manages the lifecycle of apps and provides a common navigation back stack
*   Content Providers that enable apps to access data from other apps, such as the Contacts app, or to share their own data

Developers have full access to the same framework APIs that Android system apps use.

**System Apps**

Android comes with a set of core apps for email, SMS messaging, calendars, internet browsing, contacts, and more. Apps included with the platform have no special status among the apps the user chooses to install. So a third-party app can become the user’s default web browser, SMS messenger, or even the default keyboard (some exceptions apply, such as the system’s Settings app).

The system apps function both as apps for users and to provide key capabilities that developers can access from their own app. For example, if your app would like to deliver an SMS message, you don’t need to build that functionality yourself — you can instead invoke whichever SMS app is already installed to deliver a message to the recipient you specify.

**Android Security Features**
=============================

Below are the security features provided by Android to make the Android devices you develop as secure as possible.

**App sandbox**

The Android platform takes advantage of the Linux user-based protection to identify and isolate app resources. To do this, Android assigns a unique user ID (UID) to each Android app and runs it in its own process. Android uses this UID to set up a kernel-level App Sandbox.

**App signing**

App signing allows developers to identify the author of the app and to update their app without creating complicated interfaces and permissions. Every app that runs on the Android platform must be signed by the developer.

**Authentication**

Android uses the concept of user-authentication-gated cryptographic keys that requires cryptographic key storage and service provider and user authenticators.

On devices with a fingerprint sensor, users can enroll one or more fingerprints and use those fingerprints to unlock the device and perform other tasks. The Gatekeeper subsystem performs device pattern/password authentication in a Trusted Execution Environment (TEE).

Android 9 and higher includes Protected Confirmation, which gives users a way to formally confirm critical transactions, such as payments.

**Biometrics**

Android 9 and higher includes a BiometricPrompt API that app developers can use to integrate biometric authentication into their apps in a device- and modality-agnostic fashion. Only strong biometrics can integrate with `BiometricPrompt`.

**Encryption**

Once a device is encrypted, all user-created data is automatically encrypted before committing it to disk and all reads automatically decrypt data before returning it to the calling process. Encryption ensures that even if an unauthorized party tries to access the data, they won’t be able to read it.

**Keystore**

Android offers a hardware-backed Keystore that provides key generation, import and export of asymmetric keys, import of raw symmetric keys, asymmetric encryption and decryption with appropriate padding modes, and more.

**Security-Enhanced Linux**

As part of the Android security model, Android uses Security-Enhanced Linux (SELinux) to enforce mandatory access control (MAC) over all processes, even processes running with root/superuser privileges (Linux capabilities).

**Trusty Trusted Execution Environment (TEE)**

Trusty is a secure Operating System (OS) that provides a Trusted Execution Environment (TEE) for Android. The Trusty OS runs on the same processor as the Android OS, but Trusty is isolated from the rest of the system by both hardware and software.

**Verified Boot**

Verified Boot strives to ensure all executed code comes from a trusted source (usually device OEMs), rather than from an attacker or corruption. It establishes a full chain of trust, starting from a hardware-protected root of trust to the bootloader, to the boot partition and other verified partitions.

APK File Structure
==================

An APK file is an archive that usually contains the following files and directories:

![](https://miro.medium.com/max/60/0*Nol1NP4-wjTLc8nN.png?q=20)![](https://miro.medium.com/max/563/0*Nol1NP4-wjTLc8nN.png)![](https://miro.medium.com/max/900/0*Nol1NP4-wjTLc8nN.png)APK File Structure

**Manifest file:** An additional Android manifest file, describing the name, version, access rights, referenced library files for the application. This file may be in Android binary XML that can be converted into human-readable plaintext XML with tools such as AXMLPrinter2, apktool, or Androguard.

**Signatures (META-INF):** It contains the Certificates and the signatures.

**Assets:** A directory containing applications assets, which can be retrieved by AssetManager.

**Compiled resources(resources.arsc):** A file containing precompiled resources, such as binary XML for example.

**Native Libraries(lib):** The directory containing the compiled code that is platform dependent; the directory is split into more directories within it:

*   `armeabi-v7a`: compiled code for all ARMv7 and above based processors only
*   `arm64-v8a`: compiled code for all ARMv8 arm64 and above based processors only
*   `x86`: compiled code for x86 processors only
*   `x86_64`: compiled code for x86 64 processors only

**Dalvik bytecode(class.dex):** The classes compiled in the dex file format understandable by the Dalvik virtual machine and by the Android Runtime.

**Resources(res):** the directory containing resources not compiled into resources.arsc

Check out [Setting environment for android](/mobis3c/setting-up-an-android-pentesting-environment-29991aa0c3f1).