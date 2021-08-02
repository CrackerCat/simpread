> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/mobis3c/insecure-data-storage-clear-text-storage-of-sensitive-information-hard-coded-strings-fb7b056c0d0)

> Before we get started, we need to have the apk which can be extracted from the device by installing t......

Insecure Data Storage: Clear Text Storage of Sensitive Information (Hard-coded strings, credentials, tokens & keys)
===================================================================================================================

[![](https://miro.medium.com/fit/c/96/96/1*uTzz4c17GKaaSp47mMSkgw.jpeg)](https://3kal.medium.com/?source=post_page-----fb7b056c0d0--------------------------------)[Kal](https://3kal.medium.com/?source=post_page-----fb7b056c0d0--------------------------------)Follow[Feb 5](/mobis3c/insecure-data-storage-clear-text-storage-of-sensitive-information-hard-coded-strings-fb7b056c0d0?source=post_page-----fb7b056c0d0--------------------------------) · 3 min read

Before we get started, we need to have the apk which can be extracted from the device by installing the application through the play store or by downloading the apk from online sources.

For practical, we will be looking for hardcoded google api key.

Now, let’s start analyzing the application by opening it in [Jadx](/mobis3c/setting-up-an-android-pentesting-environment-29991aa0c3f1#e4f6) (check this post to setup this tool)

> **Note:** Most of the cases, the hardcoded secrets will be found in **AndroidManifest.xml** and **Strings.xml** and make sure you check raw folder as well for the secrets.

so if you go through the resources.arsc/res/values/strings.xml file we will be able to find the google api key as shown.

![](https://miro.medium.com/max/60/1*3cDLNfzWrV5aUr-4UlamFQ.png?q=20)![](https://miro.medium.com/max/875/1*3cDLNfzWrV5aUr-4UlamFQ.png)![](https://miro.medium.com/max/1400/1*3cDLNfzWrV5aUr-4UlamFQ.png)jadx

But, we have no idea about the key and whether it is valid or not.

so let us use [**KeyHacks**](https://github.com/streaak/keyhacks)**,** it shows the way to check if the keys we found are valid keys which leaks sensitive information and valid ways in which particular API keys can be used.

Goto [KeyHacks](https://github.com/streaak/keyhacks) and search for google and found multiple services available for api key, so i replaced **key_here** keyword in the service urls and check for the response.

![](https://miro.medium.com/max/60/1*3hdpmbAx9qrAg_wR6Kq-5A.png?q=20)![](https://miro.medium.com/max/875/1*3hdpmbAx9qrAg_wR6Kq-5A.png)![](https://miro.medium.com/max/1400/1*3hdpmbAx9qrAg_wR6Kq-5A.png)google api response

I got this error, for few services by which i assume i’m authorized to use this service but not from this ip address i,e referer restriction.

while browsing the code, i came across this one ongoing request that had two unique headers which caught my attention. They were:  
`X-Android-Cert` and `X-Android-Package`

![](https://miro.medium.com/max/60/1*MQjsT2HHe00KxuHvgekxkg.png?q=20)![](https://miro.medium.com/max/875/1*MQjsT2HHe00KxuHvgekxkg.png)![](https://miro.medium.com/max/1400/1*MQjsT2HHe00KxuHvgekxkg.png)unique headers

`X-Android-Cert` contains app’s certificate hash in SHA1. To check it out:-

Extract the apk using an archive manager and select `/META-INF` which contains the certificate file. To view information related to certificate use **keytool**.

```
keytool -printcert -file CERT.RSA

```

![](https://miro.medium.com/max/60/1*zOaud58cWG6WaFR2q4LPwg.png?q=20)![](https://miro.medium.com/max/875/1*zOaud58cWG6WaFR2q4LPwg.png)![](https://miro.medium.com/max/1400/1*zOaud58cWG6WaFR2q4LPwg.png)keytool

The second header is pretty descriptive with the name `X-Android-Package`. It is the package name of the apk - `com.redacted.app`

add both the headers in the request of google APIs to see if the authorization can be bypassed.

![](https://miro.medium.com/max/60/1*tWeB4_Jop0PGMztX1gS1tA.png?q=20)![](https://miro.medium.com/max/875/1*tWeB4_Jop0PGMztX1gS1tA.png)![](https://miro.medium.com/max/1400/1*tWeB4_Jop0PGMztX1gS1tA.png)After bypassing the restriction