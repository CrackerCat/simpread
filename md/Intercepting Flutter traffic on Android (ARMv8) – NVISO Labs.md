> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.nviso.eu](https://blog.nviso.eu/2020/05/20/intercepting-flutter-traffic-on-android-x64/)

> In a previous blogpost, I explained my steps for reversing the flutter.so binary to identify the corr......

[In a previous blogpost](https://blog.nviso.eu/2019/08/13/intercepting-traffic-from-android-flutter-applications/), I explained my steps for reversing the flutter.so binary to identify the correct offset/pattern to bypass certificate validation. As a very quick summary: **Flutter doesn’t use the system’s proxy settings, and it doesn’t use the system’s certificate store**, so normal approaches don’t work. My previous guide only explained how to intercept Flutter on ARMv7 Android devices, but the steps don’t fully transfer to ARMv8 so this blogpost quickly explains the steps for ARMv8

This blogpost is written as a guide / thought process, so you can find a **TL;DR at the bottom**.

### Testing apps

First, we’ll need a testing app. I’ve slightly updated the previous one to have two buttons: one for HTTP and one for HTTPS calls. This way, I can validate whether the proxy works, and then whether the Frida script works.

The [app can be downloaded](https://github.com/NVISO-BE/blogposts/tree/master/flutter-testapps) from our GitHub.

There are two functions in the app that call an HTTP and HTTPS endpoint:

```
void callHTTP(){
  client = HttpClient();
  _status = "Calling...";
  client
      .getUrl(Uri.parse('http://neverssl.com'))
      .then((request) => request.close())
      .then((response) => setState((){_status = "HTTP: SUCCESS (" + response.headers.value("date") + ")" ;}))
      .catchError((e) =>
          setState(() {
            _status = "HTTP: ERROR";
            print(e.toString());
          })
      );
}
void callHTTPS(){
  client = HttpClient();
  _status = "Calling...";
  client
      .getUrl(Uri.parse('https://www.nviso.eu')) // produces a request object
      .then((request) => request.close()) // sends the request
      .then((response) => setState((){
                                        _status = "HTTPS: SUCCESS (" + response.headers.value("date") + ")" ;
                                    }))
      .catchError((e) =>
                      setState(() {
                        _status = "HTTPS: ERROR";
                        print(e.toString());
                      })
                );
}


```

### Proxying the application

Flutter applications still don’t automatically use the system’s proxy, unless the developer adds this functionality by creating custom Android & iOS plugins that provide this information. Obviously, many developers won’t do this, so we still need to intercept the traffic using ProxyDroid’s root-based method rather than configuring the WIFI’s proxy through the Android OS. After configuring ProxyDroid with the correct settings, Burp can see the requests from the app.

![](https://i0.wp.com/blog.nviso.eu/wp-content/uploads/2020/04/httpvshttps.jpg?resize=288%2C296&is-pending-load=1#038;ssl=1)

The HTTP requests work without any special requirement, while the HTTPS call prints an error to logcat:

```
04-26 16:59:02.758 11773 11802 E flutter : [ERROR:flutter/lib/ui/ui_dart_state.cc(157)] Unhandled Exception: HandshakeException: Handshake error in client (OS Error: 
04-26 16:59:02.758 11773 11802 E flutter :      NO_START_LINE(pem_lib.c:622)
04-26 16:59:02.758 11773 11802 E flutter :      PEM routines(by_file.c:148)
04-26 16:59:02.758 11773 11802 E flutter :      NO_START_LINE(pem_lib.c:622)
04-26 16:59:02.758 11773 11802 E flutter :      PEM routines(by_file.c:148)
04-26 16:59:02.758 11773 11802 E flutter :      CERTIFICATE_VERIFY_FAILED: self signed certificate in certificate chain(handshake.cc:354))



```

### Disabling SSL verification

I initially thought the x64 version would be identical to the x86 version. It’s the same source code, so why would the steps be any different… Unfortunately, when searching for ‘x509.cc’ in flutter.so, I found the same number of hits, but none of them were the correct function:

![](https://i2.wp.com/blog.nviso.eu/wp-content/uploads/2020/04/x509stringrefs.png?resize=783%2C138&is-pending-load=1#038;ssl=1)The previous approach seems inefective

It’s pretty obvious that the [ssl_x509.cc](https://github.com/google/boringssl/blob/master/ssl/ssl_x509.cc#L362) class has been compiled somewhere in the 0x650000 region, but that’s still a lot of functions to try to find the correct one. If searching for the filename doesn’t work, maybe searching for the line number would work. If we take a look at the [ssl_crypto_x509_session_verify_cert_chain](https://github.com/google/boringssl/blob/master/ssl/ssl_x509.cc#L362) function again, we can see that the OPENSSL_PUT_ERROR macro is called at line 390. Searching for the number 390 (or 0x186) gives us some results (Search > For Scalars…):

![](https://i1.wp.com/blog.nviso.eu/wp-content/uploads/2020/04/0x186.png?resize=794%2C408&is-pending-load=1#038;ssl=1)Searching for magic numbers

A few of the results are around the 0x650000 region. The highlighted function (FUN_0065a4ec) looks like a good candidate, as the constant is loaded in w3 (the lower 32bit part of the x3 register), which is one of the argument registers on ARMv8. The function FUN_0065a4ec also has the correct signature, and it generally looks the same as the ARMv7 version:

![](https://i2.wp.com/blog.nviso.eu/wp-content/uploads/2020/04/32bitvs64bit.png?resize=1100%2C630&is-pending-load=1#038;ssl=1)Decompiled method in x86 vs x64

My normal approach would be to copy the first bytes of FUN_0065a4ec and search for them in-memory while the application is running, [as I did in the previous blogpost](https://blog.nviso.eu/2019/08/13/intercepting-traffic-from-android-flutter-applications/), so I don’t need to find the offset each time. Unfortunately, Frida’s Memory.scan seems to [crash](https://github.com/frida/frida/issues/1273) on my test app, so for now we’ll have to use the offset. (_Edit: An alternative approach was posted in the comments of this post, using Process.enumerateRangesSync_)

Ghidra uses 0x100000 as the base address of the module, so we have to subtract that from the Ghidra offset, resulting in an offset of **0x55a4ec**.

Opening Ghidra every time works, but it’s not that convenient. We can also use binwalk to find the correct offset based on those first bytes of the function:

```
# The first bytes of the FUN_0065a4ec function
ff 03 05 d1 fc 6b 0f a9 f9 63 10 a9 f7 5b 11 a9 f5 53 12 a9 f3 7b 13 a9 08 0a 80 52
# Find it using binwalk
binwalk -R "\xff\x03\x05\xd1\xfc\x6b\x0f\xa9\xf9\x63\x10\xa9\xf7\x5b\x11\xa9\xf5\x53\x12\xa9\xf3\x7b\x13\xa9\x08\x0a\x80\x52" libflutter.so
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
5612780       0x55A4EC        Raw signature (\xff\x03\x05\xd1\xfc\x6b\x0f\xa9\xf9\x63\x10\xa9\xf7\x5b\x11\xa9\xf5\x53\x12\xa9\xf3\x7b\x13\xa9\x08\x0a\x80\x52)



```

Let’s throw this in a Frida script and test it!

```
function hook_ssl_verify_result(address)
{
  Interceptor.attach(address, {
    onEnter: function(args) {
      console.log("Disabling SSL validation")
    },
    onLeave: function(retval)
    {
      console.log("Retval: " + retval)
      retval.replace(0x1);
 
    }
  });
}
function disablePinning(){
	// Change the offset on the line below with the binwalk result
	// If you are on 32 bit, add 1 to the offset to indicate it is a THUMB function.
	// Otherwise, you will get  'Error: unable to intercept function at ......; please file a bug'
	var address = Module.findBaseAddress('libflutter.so').add(0x55a4ec)
	hook_ssl_verify_result(address);
}
setTimeout(disablePinning, 1000)


```

Running this file using Frida gives the expected outcome:

```
(secenv) ➜  flutter frida -Uf be.nviso.flutter_app -l hook.js --no-pause
     ____
    / _  |   Frida 12.8.20 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ | |       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Spawned `be.nviso.flutter_app`. Resuming main thread!                   
[SM-G950F::be.nviso.flutter_app]-> disablePinning()                                                                                                                                                                                                                                     
[SM-G950F::be.nviso.flutter_app]-> Disabling SSL validation
Retval: 0x0
[SM-G950F::be.nviso.flutter_app]-> 


```

![](https://i0.wp.com/blog.nviso.eu/wp-content/uploads/2020/04/httpswin.jpg?resize=249%2C512&is-pending-load=1#038;ssl=1)SSL Verification successfully disabled

### Tangent: Why can this app perform cleartext HTTP calls?

My flutter app is making HTTP connections. This is [forbidden by default since Android P](https://developer.android.com/training/articles/security-config#CleartextTrafficPermitted), and you have to add a Network Security Policy that explicitly allows cleartext request if you still want to do so on Android 9+. My test app does not have a Network Security Policy, so what’s going on?

The reason for this is the same reason why these blog posts are necessary: Flutter doesn’t use default Android libraries. Because Flutter creates low level sockets and implements the HTTP stack on top of that, the requests never pass by the Android security controls that should prevent cleartext traffic from being used. This is an important thing to keep in mind when auditing the security of Flutter apps, as you might miss things if you’re not careful.

### TL;DR (ARMv7 and ARMv8)

1.  Redirect with ProxyDroid on rooted device since Flutter apps are still proxy-unaware
2.  Find the offset using binwalk
3.  Use the Frida script to hook the method at that offset

Since the last blogpost, the signature for 32bit also changed, so I’ve included both signatures.

```
# Method signatures for ARMv7 (32bit)
2d e9 f0 4f a3 b0 81 46 50 20 10 70
2d e9 f0 4f a3 b0 82 46 50 20 10 70
# Get the offset
binwalk -R "\x2d\xe9\xf0\x4f\xa3\xb0\x81\x46\x50\x20\x10\x70" -R "\x2d\xe9\xf0\x4f\xa3\xb0\x82\x46\x50\x20\x10\x70" ./libflutter.so
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
3831160       0x3A7578        Raw signature (\x2d\xe9\xf0\x4f\xa3\xb0\x81\x46\x50\x20\x10\x70)
# Method signature for ARMv8 (64bit)
ff 03 05 d1 fc 6b 0f a9 f9 63 10 a9 f7 5b 11 a9 f5 53 12 a9 f3 7b 13 a9 08 0a 80 52
# Get the offset
binwalk -R "\xff\x03\x05\xd1\xfc\x6b\x0f\xa9\xf9\x63\x10\xa9\xf7\x5b\x11\xa9\xf5\x53\x12\xa9\xf3\x7b\x13\xa9\x08\x0a\x80\x52" libflutter.so
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
5612780       0x55A4EC        Raw signature (\xff\x03\x05\xd1\xfc\x6b\x0f\xa9\xf9\x63\x10\xa9\xf7\x5b\x11\xa9\xf5\x53\x12\xa9\xf3\x7b\x13\xa9\x08\x0a\x80\x52)



```

Frida script to use the offset:

```
function hook_ssl_verify_result(address)
{
  Interceptor.attach(address, {
    onEnter: function(args) {
      console.log("Disabling SSL validation")
    },
    onLeave: function(retval)
    {
      console.log("Retval: " + retval)
      retval.replace(0x1);
 
    }
  });
}
function disablePinning(){
	// Change the offset on the line below with the binwalk result
	// If you are on 32 bit, add 1 to the offset to indicate it is a THUMB function: .add(0x1)
	// Otherwise, you will get  'Error: unable to intercept function at ......; please file a bug'
	var address = Module.findBaseAddress('libflutter.so').add(0x55a4ec)
	hook_ssl_verify_result(address);
}
setTimeout(disablePinning, 1000)


```

And launch it using Frida:

```
frida -Uf hook.js -f be.nviso.flutter_app --no-pause


```

If it still doesn’t work, you’ll have to figure out the correct method to hook yourself. You can try following [the steps for ARMv7 as described on this blog.](https://blog.nviso.eu/2019/08/13/intercepting-traffic-from-android-flutter-applications/)

About the author
----------------

Jeroen Beckers is a mobile security expert working in the NVISO Cyber Resilience team and co-author of the OWASP Mobile Security Testing Guide (MSTG). He also loves to program, both on high and low level stuff, and deep diving into the Android internals doesn’t scare him. You can find Jeroen on [LinkedIn](https://www.linkedin.com/in/beckersjeroen/).

Published by Jeroen Beckers
---------------------------

Jeroen Beckers is a mobile security expert working in the NVISO Software and Security assessment team. He is a SANS instructor and SANS lead author of the SEC575 course. Jeroen is also a co-author of OWASP Mobile Security Testing Guide (MSTG) and the OWASP Mobile Application Security Verification Standard (MASVS). He loves to both program and reverse engineer stuff. [View all posts by Jeroen Beckers](https://blog.nviso.eu/author/dauntless/)

**Published** May 20, 2020November 21, 2020