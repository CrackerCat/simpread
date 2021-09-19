> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.nviso.eu](https://blog.nviso.eu/2019/08/13/intercepting-traffic-from-android-flutter-applications/)

> Update: The explanation below explains the step for ARMv7. For ARMv8 (64bit), see this blogpost. Flut......

**Update:** The explanation below explains the step for ARMv7. [For ARMv8 (64bit), see this blogpost](https://blog.nviso.eu/2020/05/20/intercepting-flutter-traffic-on-android-x64/).

[Flutter](https://flutter.dev/) is Google’s new open source mobile development framework that allows developers to write a single code base and build for Android, iOS, web and desktop. Flutter applications are written in [Dart](https://dart.dev/), a language created by Google more than 7 years ago.

![](https://i1.wp.com/blog.nviso.eu/wp-content/uploads/2019/08/flutter_og.png?resize=427%2C240&is-pending-load=1#038;ssl=1)

It’s often necessary to intercept traffic between a mobile application and the backend (either for a security assessment or a bounty hunt), which is typically done by adding Burp as an intercepting proxy. Flutter applications are a little bit more difficult to proxy, but it’s definitely possible.

TL;DR
-----

*   Flutter uses Dart, which doesn’t use the system CA store
*   Dart uses a list of CA’s that’s compiled into the application
*   Dart is not proxy aware on Android, so use ProxyDroid with iptables
*   Hook the [session_verify_cert_chain](https://github.com/google/boringssl/blob/master/ssl/ssl_x509.cc#L362) function in x509.cc to disable chain validation
*   You might be able to use the script at the bottom of this article directly, or you can follow the steps below to get the right bytes or offset.

Test setup
----------

In order to perform my tests, I [installed the flutter plugin](https://flutter.dev/docs/get-started/editor) and created a Flutter application that comes with a default interactive button that increments a counter. I modified it to fetch a URL through the HttpClient class:

```
class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;
  HttpClient client;

  _MyHomePageState()
  {
      _start();
  }
  void _start() async
  {
    client = HttpClient();
  }
  void _incrementCounter() {
    setState(() {
      if(client != null)
      {
          client
              .getUrl(Uri.parse('http://www.nviso.eu')) // produces a request object
              .then((request) => request.close()) // sends the request
              .then((response) => print("SUCCESS - " + response.headers.value("date")));
          _counter++;
       }
    });
  }


```

The app can be compiled using `flutter build aot` and pushed to the device through `adb install`.

![](https://i2.wp.com/blog.nviso.eu/wp-content/uploads/2019/08/android_screenshot.png?resize=270%2C480&is-pending-load=1#038;ssl=1)

Every time we press the button, a call is sent to [http://www.nviso.eu](http://www.nviso.eu/) and if it’s successful it is printed to the device logs.

On my device I have Frida installed through [Magisk-Frida-Server](https://github.com/TheDauntless/Magisk-Frida-Server) and my Burp certificate is added to the system CA store with the [MagiskTrustUserCerts](https://github.com/NVISO-BE/MagiskTrustUserCerts) module. Unfortunately, Burp does not see any traffic passing through, even though the app logs indicate that the request was successful.

Sending traffic to the proxy through ProxyDroid/iptables
--------------------------------------------------------

The HttpClient has a [findProxy](https://api.dartlang.org/stable/2.4.0/dart-io/HttpClient/findProxy.html) method and its documentation is pretty clear on this: By default all traffic is sent directly to the target server, without taking any proxy settings into account:

> Sets the function used to resolve the proxy server to be used for opening a HTTP connection to the specified `url`. If this function is not set, direct connections will always be used.
> 
> [findProxy documentation](https://api.dartlang.org/stable/2.4.0/dart-io/HttpClient/findProxy.html)

The application can set this property to `HttpClient.findProxyFromEnvironment` which searches for specific environment variables such as `http_proxy` and `https_proxy`. Even if the application would be compiled with this implementation, it would be pretty useless on Android since all applications are children of the initial zygote process which does not have these environment variables.

It’s also possible to define a custom findProxy implementation that returns the preferred proxy. A quick modification on my test application indeed shows that this configuration sends all HTTP data to my proxy:

```
client.findProxy = (uri) {        
    return "PROXY 10.153.103.222:8888";     
};


```

![](https://i2.wp.com/blog.nviso.eu/wp-content/uploads/2019/08/burp_http.png?resize=1100%2C810&is-pending-load=1#038;ssl=1)

Of course, we can’t modify the application during a black-box assessment, so another approach is needed. Luckily, we always have the iptables fallback to route all traffic from the device to our proxy. On a rooted device, ProxyDroid handles this pretty well and we can see all HTTP traffic flowing through Burp.

![](https://i2.wp.com/blog.nviso.eu/wp-content/uploads/2019/08/proxydroid.png?resize=270%2C480&ssl=1)ProxyDroid with root access using iptables

Intercepting HTTPS traffic
--------------------------

This is where it gets more tricky. If I change the URL to HTTPS, Burp complains that the SSL handshake fails. This is weird since my device is set up to include my Burp certificate as a trusted root CA.

After some research, I ended up on a [GitHub issue](https://github.com/dart-lang/sdk/issues/32131#issuecomment-365137754) that explains the issue for Windows, but the same is applicable to Android: Dart generates and [compiles its own Keystore](https://github.com/dart-lang/root_certificates) using [Mozilla’s NSS library.](https://hg.mozilla.org/mozilla-central/raw-file/tip/security/nss/lib/ckfw/builtins/certdata.txt)

This means that we can’t bypass SSL validation by adding our proxy CA to the system CA store. To solve this we have to dig into libflutter.so and figure out what we need to patch or hook in order to validate our certificate. Dart uses Google’s BoringSSL to handle everything SSL related, and luckily both Dart and BoringSSL are open source.

When sending HTTPS traffic to Burp, the Flutter application actually throws an error, which we can take as a starting point:

```
E/flutter (10371): [ERROR:flutter/runtime/dart_isolate.cc(805)] Unhandled exception:
 E/flutter (10371): HandshakeException: Handshake error in client (OS Error: 
 E/flutter (10371):  NO_START_LINE(pem_lib.c:631)
 E/flutter (10371):  PEM routines(by_file.c:146)
 E/flutter (10371):  NO_START_LINE(pem_lib.c:631)
 E/flutter (10371):  PEM routines(by_file.c:146)
 E/flutter (10371):  CERTIFICATE_VERIFY_FAILED: self signed certificate in certificate chain(handshake.cc:352))
 E/flutter (10371): #0      _rootHandleUncaughtError. (dart:async/zone.dart:1112:29)
 E/flutter (10371): #1      _microtaskLoop (dart:async/schedule_microtask.dart:41:21)
 E/flutter (10371): #2      _startMicrotaskLoop (dart:async/schedule_microtask.dart:50:5)
 E/flutter (10371): #3      _runPendingImmediateCallback (dart:isolate-patch/isolate_patch.dart:116:13)
 E/flutter (10371): #4      _RawReceivePortImpl._handleMessage (dart:isolate-patch/isolate_patch.dart:173:5)

```

The first thing we need to do is find this error in the [BoringSSL library](https://github.com/google/boringssl). The error actually shows us where the error is triggered: `handshake.cc:352`. [Handshake.cc](https://github.com/google/boringssl/blob/master/ssl/handshake.cc) is indeed part of the BoringSSL library and does contain logic to perform certificate validation. The code at line 352 is shown below, and this is most likely the error we are seeing. The line numbers don’t match exactly, but this is most likely the result of a version difference.

```
if (ret == ssl_verify_invalid) {
    OPENSSL_PUT_ERROR(SSL, SSL_R_CERTIFICATE_VERIFY_FAILED);
    ssl_send_alert(ssl, SSL3_AL_FATAL, alert);
  }


```

This is part of the ssl_verify_peer_cert function which returns the ssl_verify_result_t enum which is defined in [ssl.h](https://github.com/google/boringssl/blob/master/include/openssl/ssl.h#L2290) at line 2290:

```
enum ssl_verify_result_t BORINGSSL_ENUM_INT {
  ssl_verify_ok,
  ssl_verify_invalid,
  ssl_verify_retry,
};


```

If we can change the return value of ssl_verify_peer_cert to ssl_verify_ok (=0), we should be good to go. However, a lot of stuff is going on in this method, and Frida can only (easily) change the return value of a function. If we change this value, it would still fail due to the ssl_send_alert() function call above (trust me, I tried ![](https://s.w.org/images/core/emoji/13.1.0/svg/1f642.svg) ).

Let’s find a better method to hook. Right above the snippet from handshake.cc is the following code, which is the actual part of the method that is validating the chain:

```
ret = ssl->ctx->x509_method->session_verify_cert_chain(
              hs->new_session.get(), hs, &alert)
              ? ssl_verify_ok
              : ssl_verify_invalid;


```

The session_verify_cert_chain function is defined in [ssl_x509.cc](https://github.com/google/boringssl/blob/master/ssl/ssl_x509.cc#L362) at line 362. This function also returns a primitive datatype (boolean) and is a better candidate to hook. If a check fails in this function, it only reports the issue via OPENSSL_PUT_ERROR, but it doesn’t have side effects like the ssl_verify_peer_cert function. The OPENSSL_PUT_ERROR is a macro defined in [err.h at line 418](https://github.com/google/boringssl/blob/master/include/openssl/err.h#L418) that includes the source filename. This is the same macro that was used for the error that made it to the Flutter app.

```
#define OPENSSL_PUT_ERROR(library, reason) \
  ERR_put_error(ERR_LIB_##library, 0, reason, __FILE__, __LINE__)


```

Now that we know which function we want to hook, we need to find it in libflutter.so. The OPENSSL_PUT_ERROR macro is called a few times in the session_verify_cert_chain function, which makes it easy to find the correct method using Ghidra. So import the library into Ghidra, use Search -> Find Strings and search for `x509.cc`.

![](https://i0.wp.com/blog.nviso.eu/wp-content/uploads/2019/07/ghidra_search.png?resize=1100%2C552&is-pending-load=1#038;ssl=1)Searching for the x509.cc string

There are only 4 XREFs so it’s easy to go over them and find one that looks like the session_verify_cert_chain function:

![](https://i1.wp.com/blog.nviso.eu/wp-content/uploads/2019/07/xrefs.png?resize=1100%2C155&is-pending-load=1#038;ssl=1)Only 4 xrefs

One of the functions takes 2 ints, 1 ‘undefined’ and contains a single call to OPENSSL_PUT_ERROR (`FUN_00316500`). In my version of libflutter.so, this is `FUN_0034b330`. What you typically do now is calculate the offset of this function from one of the exported functions and hook it. I usually take a lazy approach where I copy the first 10 or so bytes of the function and check how often that pattern occurs. If it only occurs once, I know I found the function and I can hook it. This is useful because I can often use the same script for different versions of the library. With an offset based approach, this is more difficult.

![](https://i1.wp.com/blog.nviso.eu/wp-content/uploads/2019/08/function_header.png?resize=1100%2C545&is-pending-load=1#038;ssl=1)

So now we let Frida search the `libflutter.so` library for this pattern:

```
var m = Process.findModuleByName("libflutter.so"); 
var pattern = "2d e9 f0 4f a3 b0 82 46 50 20 10 70"
var res = Memory.scan(m.base, m.size, pattern, {
  onMatch: function(address, size){
      console.log('[+] ssl_verify_result found at: ' + address.toString());  
    }, 
  onError: function(reason){
      console.log('[!] There was an error scanning memory');
    },
    onComplete: function()
    {
      console.log("All done")
    }
  });


```

Running this script on my Flutter application gives just a single result:

```
 (env) ~/D/Temp » frida -U -f be.nviso.flutter_app -l frida.js --no-pause                 
 [LGE Nexus 5::be.nviso.flutter_app]-> [+] ssl_verify_result found at: 0x9a7f7040
 All done 

```

Now we just need to use the Interceptor to change the return value to 1 (true):

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
function disablePinning()
{
 var m = Process.findModuleByName("libflutter.so"); 
 var pattern = "2d e9 f0 4f a3 b0 82 46 50 20 10 70"
 var res = Memory.scan(m.base, m.size, pattern, {
  onMatch: function(address, size){
      console.log('[+] ssl_verify_result found at: ' + address.toString());

      // Add 0x01 because it's a THUMB function
      // Otherwise, we would get 'Error: unable to intercept function at 0x9906f8ac; please file a bug'
      hook_ssl_verify_result(address.add(0x01));
      
    }, 
  onError: function(reason){
      console.log('[!] There was an error scanning memory');
    },
    onComplete: function()
    {
      console.log("All done")
    }
  });
}
setTimeout(disablePinning, 1000)


```

After setting up ProxyDroid and launching the application with this script, we can now finally see HTTPs traffic:

![](https://i0.wp.com/blog.nviso.eu/wp-content/uploads/2019/08/burp_https.png?resize=1100%2C747&is-pending-load=1#038;ssl=1)

I’ve tested this on a few Flutter apps and this approach worked on all of them. As the BoringSSL library will most likely stay rather stable, this approach might work for some time to come.

Disable SSL Pinning (SecurityContext)
-------------------------------------

Finally, let’s see how we can get around SSL Pinning. One way of doing this is by defining a new SecurityContext that contains specific certificates. While this is not technically SSL pinning (you don’t protect against a compromised private key), it’s often implemented to prevent against easy eavesdropping of the communication channel.

For my app, I added the following code to have it accept only my burp certificate. The [SecurityContext constructor](https://api.flutter.dev/flutter/dart-io/SecurityContext/SecurityContext.html) takes one argument, `withTrustedRoots`, which defaults to false.

```
ByteData data = await rootBundle.load('certs/burp.crt');
    SecurityContext context = new SecurityContext();
    context.setTrustedCertificatesBytes(data.buffer.asUint8List());
    client = HttpClient(context: context);


```

The application will now automatically accept our Burp proxy as the certificate for any website, which shows that this method can be used to specify a specific certificate that the application must comply to. If we now switch this to the nviso.eu certificate, we can no longer intercept the connection.

Fortunately, the Frida script listed above already bypasses this kind of root-ca-pinning implementation, as the underlying logic still depends on the same methods of the BoringSSL library.

Disable SSL Pinning (ssl_pinning_plugin)
----------------------------------------

One of the ways Flutter developers might want to perform ssl pinning is through the [ssl_pinning_plugin](https://github.com/macif-dev/ssl_pinning_plugin) flutter plugin. This plugin is actually designed to send one HTTPS connection and verify the certificate, after which the developer will trust the channel and perform non-pinned HTTPS requests:

With correct timing of ProxyDroid, this can already be circumvented, but let’s just disable it anyway.

```
void testPin() async
  {
    List<String> hashes = new List<String>();
    hashes.add("randomhash");
    try
    {
      await SslPinningPlugin.check(serverURL: "https://www.nviso.eu", headerHttp : new Map(), sha: SHA.SHA1, allowedSHAFingerprints: hashes, timeout : 50);

      doImportanStuff()
    }catch(e)
    {
      abortWithError(e);
    }
  }


```

The plugin is a bridge to a [Java implementation](https://github.com/macif-dev/ssl_pinning_plugin/blob/master/android/src/main/kotlin/com/macif/plugin/sslpinningplugin/SslPinningPlugin.kt) which we can easily hook with Frida:

```
function disablePinning()
{
    var SslPinningPlugin = Java.use("com.macif.plugin.sslpinningplugin.SslPinningPlugin");
    SslPinningPlugin.checkConnexion.implementation = function()
    {
        console.log("Disabled SslPinningPlugin");
        return true;
    }
}

Java.perform(disablePinning)


```

Conclusion
----------

This was a pretty fun ride, and it went quite smoothly since both Dart and BoringSSL are open source. Due to just a few interesting strings, it’s pretty easy to find the correct place to disable the ssl verification logic, even without any symbols. My approach with scanning for the function prologue might not always work, but since BoringSSL is pretty stable, it should work for some time to come.

About the author
----------------

Jeroen Beckers is a mobile security expert working in the NVISO Cyber Resilience team and co-author of the OWASP Mobile Security Testing Guide (MSTG). He also loves to program, both on high and low level stuff, and deep diving into the Android internals doesn’t scare him. You can find Jeroen on [LinkedIn](https://www.linkedin.com/in/beckersjeroen/).

![](https://i1.wp.com/blog.nviso.eu/wp-content/uploads/2018/01/aaeaaqaaaaaaaayhaaaajguzzmuxmmvmlwy3m2mtndrmny05yzzllwmxztk1zte5mwyzmq.jpg?resize=300%2C300&is-pending-load=1#038;ssl=1)