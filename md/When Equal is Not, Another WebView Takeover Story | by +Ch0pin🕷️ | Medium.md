> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [valsamaras.medium.com](https://valsamaras.medium.com/when-equal-is-not-another-webview-takeover-story-730be8d6e202)

> I have been assessing Android applications for some time and I must admit that despite the countless ......

I have been assessing Android applications for some time and I must admit that despite the countless write-ups about unprotected WebViews, the particular issue is still on top of my list. Almost 80% of the reviewed apps were vulnerable to forced browsing with the actual impact varying from JavaScript execution to full account takeover.

The scenario is the same in most of the cases… A malicious URL, makes its way to the application’s WebView via an **unvalidated query parameter**, an **intent extra** or even an **open redirect**. The impact is usually low to medium since the dangerous features of this component are disabled by default, but it can get really rough when special conditions apply. For example, an “attached” [JavascriptInterface](https://developer.android.com/reference/android/webkit/JavascriptInterface) with some fruitful exported functionality can be a [recipe for disaster](https://dphoeniixx.medium.com/tiktok-for-android-1-click-rce-240266e78105).

![](https://miro.medium.com/max/1188/1*ynlnuheMQDcAl45EjlNgRA.png)

You might wonder that, since this issue is so common, haven’t the developer community heard about it ?… After all it can’t be that difficult to sanitise the user input.

Well, let me put it this way… think of multiple layers of java inheritance, where some superclass is loading “any” given url from an un-sanitised parameter. When you are writing code on the child of a child of a child … of a naughty parent, these issues are not as obvious as you might think.

To tackle this problem I wrote a [Medusa](https://github.com/Ch0pin/medusa) module which, between else, can monitor the application’s WebView for “risky” features, including Javascript Interfaces, file access, url loading etc. Indeed this module pinpointed some nice findings, but as this post is not to do oneself proud I’ll get straight to the point.

During my assessments I encountered cases where the mistake is pretty much obvious and this is what this post is all about. I won’t be using real application names for obvious reasons, so I wrote a simple Android app to prove my concept. Let’s start with a [deeplink](https://developer.android.com/training/app-links/deep-linking) declaration in my AndroidManifest file:

And here is the application’s code:

TL;DR Our application registers the deeplink `example://webview` which means that the `**MainActivity**` will be triggered through the intent filter for intents with action set to `**android.intent.action.VIEW**` and data `**example://webview**` . The `**onNewIntent**` callback inspects the data string and if this is not empty it calls the `**handleDeeplink**` to handle the intent. Finally the handleDeeplink will call the `**isAuthorisedURL**` function in order to check the validity of the incoming URL. If the return value is set to true, the WebView loads the URL. In the code snippet above, the isAuthorisedURL returns always true so any given URL will be loaded.

There are cases where the applications need to load various URLs in their WebViews including subdomains which are not always given during the application development. These subdomains are get added or removed from time to time to facilitate or discard various features and services.

To handle this problem, the developers come up with various solutions which sometimes are not the best from a security aspect. The **startsWith** and **endsWith** or even the **contains** functions of the java.lang.String class are used to filter out invalid domains, in the most unsafe way. Let’s see a real example which I encountered in a 100,000,000+ downloads application.

The `**isAuthorisedURL**` function looked like bellow:

The objective was probably to include the foo subdomains of the foobar domain, but the implementation is obviously wrong since all the `**foo.foobar***` URLs will be considered as valid and will be loaded to the WebView. Believe it or not, a similar issue with an endsWith function was found in a 10,000,000+ application. This time the request to the loaded URL was authenticated while the isAuthorisedURL looked like below:

One more time all the `***foobar.com**` domains will pass the `if` condition thus they will be loaded to the WebView.

Sometimes the development team during the staging phase uses hostnames which are valid inside a company’s local network but not in the world wide web … but they can be valid in the world wide web…. Here is an example:

In this case, the developers wanted to include the `example.com`, `google.com` and `test.com` but during the staging phase they also added the `staging.site` to test out this feature. While this is OK during the tests, when they publish the app they forgot to remove the extra URL. As you understand, assuming that the `staging.site` is available an attacker can register the domain and takeover the WebView. The same condition of course stands for expired domains.

This post just scratches the tip of the iceberg when it comes to WebView security vulnerabilities. The intention though was to pinpoint some cases where small mistakes (even in very popular applications) can literally have very serious impact.

See you on the next post !