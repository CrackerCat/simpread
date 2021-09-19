> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.oversecured.com](https://blog.oversecured.com/Common-mistakes-when-using-permissions-in-Android/)

> When an Android app needs access to sensitive resources on the device, the app developers make use of......

When an Android app needs access to sensitive resources on the device, the app developers make use of the permissions model. While the model can be quite simple to use, developers often make mistakes when using permissions and this leads to security problems.

In this blog, we will first explore a few different types of permissions and then go on to explain the common mistakes that developers make while implementing the permissions model and how to avoid them.

If you’re a developer or an app owner, you can integrate Oversecured into your CI/CD to proactively secure your apps against these vulnerabilities. The CI/CD process can also be fully automated using plugins. Our solutions will continuously monitor your apps and alert you if any new vulnerabilities are detected.

Start securing your apps by beginning a trial from [Quick Start](https://oversecured.com/docs/quick-start/), or you can [contact us](https://oversecured.com/contact-us) to learn more.

If you’re a security researcher, you can automate the process of bug detection by using Oversecured’s mobile app scanner to scan for these bugs. All you have to do is [sign up](https://oversecured.com/sign-up) and upload your app’s files. Our scanner will take care of the rest.

[](#declaration-of-permissions-and-their-types)[Declaration of permissions and their types](#declaration-of-permissions-and-their-types)

Permissions help in restricting access to certain Android components like activities, broadcast receivers, services, and content providers. A developer can declare various permission types in the app’s `AndroidManifest.xml` file.

Permissions are also used at runtime to verify if the app has the rights to access sensitive information or perform dangerous actions. For example, for an app to have access to contacts, a developer must include a `uses-permission` declaration in their `AndroidManifest.xml` file:

```
<uses-permission android: />


```

This is a built-in permission that has the `protectionLevel` set to `dangerous`. On implementing this permission level, the Android system asks the user during installation if they are willing to grant such rights to this app. If the user refuses, then the app will not be installed.

If a developer wants to create their own permission, then they must declare it in the manifest like this:

```
<permission android: />


```

```
<activity android:>
    <intent-filter>
        <action android: />
        <category android: />
    </intent-filter>
</activity>


```

Then, if a third-party app wants to use the functionality of this example, it will need to make a declaration like this:

```
<uses-permission android: />


```

and upon installation (and use, if it’s a runtime permission) ask the user to grant permissions.

Apart from these, there are also a few other permission levels that you should know about, like:

*   `normal`: this permission is “not visible” to the user. A user doesn’t know that the app is requesting access to this resource because the rights are automatically granted during installation.
*   `dangerous`: As mentioned earlier, this level requires the user to confirm if an app can access a particular resource.
*   `signature`: The app that uses this permission level must be signed with the same certificate as the app that declared it. This prevents an attacker from creating a `uses-permission` declaration for this level because Android will not allow the app to be installed.
*   There are several other levels, like `system`, `installer`, `privileged`, etc. which are used by system apps. These should be considered as a `signature` level from an attacker’s point of view, i.e. permissions using this `protectionLevel` cannot be declared in `uses-permission`

Now let’s look at some common mistakes that developers make when working with permissions.

[](#forgotten-protectionlevel)[Forgotten `protectionLevel`](#forgotten-protectionlevel)

What happens if you don’t mention the `protectionLevel` in the `permission` declaration?

```
<permission android: />


```

You guessed it! It will be marked as `normal` by default and this allows any app to be able to use it. A similar mistake [was made](https://blog.oversecured.com/Two-weeks-of-securing-Samsung-devices-Part-2/#file-theft-from-uid-1001-in-callBGProvider) by the developers of a built-in Samsung app, which led to the possibility of other apps on the phone reading a victim’s call history, SMS messages, etc.

[](#ecosystem-mistakes)[Ecosystem mistakes](#ecosystem-mistakes)

Let’s assume that the developer has an ecosystem of two apps: `My Cool Cam` and `My Cool Reader`. The reader app uses the functionality of the camera app.

The manifest of the cam app:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.mycoolcam">
    <permission android: />
    <uses-permission android: />

    <application android:label="My Cool Cam">
        <activity android:>
            <intent-filter>
                <action android: />
                <category android: />
            </intent-filter>
        </activity>
        <!-- ... -->
    </application>
</manifest>


```

The manifest of the reader app:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.mycoolreader">
    <uses-permission android: />

    <application android:label="My Cool Reader">
        <provider android: />
        <!-- ... -->
    </application>
</manifest>


```

At one glance, it may appear that everything is good and safe since only apps from the ecosystem can access sensitive information stored in the `AllUserNotesContentProvider`.

But, what happens if the victim’s device only has the reader app installed? In this case, the Android system will not know anything about the `com.mycoolcam.USE_COOL_CAMERA` permission declaration, and therefore, the permission level will be marked as `normal` by default.

**To prevent this, make sure that each permission from an ecosystem of several apps is also declared individually in every app.**

[](#typos-in-permission-names)[Typos in permission names](#typos-in-permission-names)

It’s extremely common for developers to make typos in permission names:

```
<permission android: />


```

```
<activity android:>
    <intent-filter>
        <action android: />
        <category android: />
    </intent-filter>
</activity>


```

The permission for `com.mycoolcam.USE_COOL_CAMERA` will have `protectionLevel` = `signature` and `com.mycoolcam.USE_COOL_CAM` will be set to `normal`. This allows any app to make a `uses-permission` declaration with `com.mycoolcam.USE_COOL_CAM` which will in turn, give access to the `CoolCamActivity` activity.

[](#typos-in-component-declarations)[Typos in component declarations](#typos-in-component-declarations)

```
<permission android: />


```

```
<activity android:>
    <intent-filter>
        <action android: />
        <category android: />
    </intent-filter>
</activity>


```

In the above example, instead of using the `android:permission` attribute (which sets the access level to the component), it says `android:uses-permission`. This will allow any third-party app to access it, as it means that the component has no protection level.

[](#insufficient-protection-of-permissions)[Insufficient protection of permissions](#insufficient-protection-of-permissions)

Some apps do not completely protect the permissions they use, which leaves room for a third-party app to exploit the vulnerable app and gain rights.

Let’s understand this better with a simple example.

File `AndroidManifest.xml`:

```
<uses-permission android: />


```

```
<provider android: />


```

File `ContactsProvider.java`:

```
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
    return getContext().getContentResolver().query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
            projection,
            selection,
            selectionArgs,
            sortOrder);
}


```

In this case, you can clearly see that accessing the URI `ContactsContract.CommonDataKinds.Phone` (alias to `content: //com.android.contacts/data/phones`) requires the permission `android.permission.READ_CONTACTS`. However, no rights are required to access `content://com.exampleapp.contacts`.

**To prevent this, apps should ensure the protection of any permissions received by them.**

[](#faqs-on-working-with-permissions)[FAQs on working with permissions](#faqs-on-working-with-permissions)

**Q: What happens if an app tries to access a component without having permission to do so?**

A: In case of an explicit intent, Android automatically throws a `SecurityException: Permission Denial`. However, in the case of an implicit activity, the activity will have an `ActivityNotFoundException` error. For receivers, the broadcast will be ignored and services cannot be started with implicit intents.

**Q: Is it possible to override the permission declaration?**

A: If the analyzed app is already installed and you try to install your own app with the same name, then it’s not possible to override the permission declaration as the system will throw an `INSTALL_FAILED_DUPLICATE_PERMISSION` error.

However, this does not apply to apps that are signed with the same certificate and in that case, to prevent the vulnerabilities described earlier, permissions must be re-declared.

**Q: Can the permissions of other apps be intercepted?**

A: This is not possible directly. However, there are several bypasses that may allow another app to perform privileged actions, such as in the example above. There are also possibilities where [intercepting access to content providers](https://blog.oversecured.com/Gaining-access-to-arbitrary-Content-Providers/) might be possible if the vulnerable app has access to them or if they are protected with permissions owned by the vulnerable app.

[](#preventing-these-vulnerabilities)[Preventing these vulnerabilities](#preventing-these-vulnerabilities)

It can be difficult to keep a track of security, especially in large projects. You can make use of the Oversecured code scanner, since it tracks all known security issues on Android including errors when working with permissions. We [notify](https://oversecured.com/vulnerabilities#Permission_with_%22normal%22_protection_level) developers about the use of interceptable permissions, like `normal` and `dangerous`, unprotected components, cases of downgraded permissions, and so on. To begin testing your apps, use [Quick Start](https://oversecured.com/docs/quick-start/) or [contact us](https://oversecured.com/contact-us).