> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [nibarius.github.io](https://nibarius.github.io/learning-frida/2021/08/28/hpandro-part2)

> Last time we started with the hpAndro Vulnerable Application CTF and solved several different challen......

[Last time](https://nibarius.github.io/learning-frida/2021/08/26/hpandro-part1) we started with the [hpAndro Vulnerable Application](http://ctf.hpandro.raviramesh.info/) CTF and solved several different challenges. Now it’s time to take on another batch of challenges. Like last time I’ve been working on these challenges using several different versions of the app, so my code might not work with the latest version.

Binary Protection: Native Function Call
---------------------------------------

The first challenge we take on this time is the Native Function Call challenge from the binary protection category. This challenge has a native function called `hello()` which is called to provide instructions for the challenge. There is another native function called `flag()` that returns the flag if called, the only problem is that it’s never called by the app.

![](https://nibarius.github.io/learning-frida/assets/hpandro/native.png) The instructions given by hello()

You could of course reverse the native code and find the flag there, but I usually try to avoid that whenever possible. Instead, let’s just do what the instructions say and call the `flag()` function to get the flag. One easy way to do that is to change the implementation of the `hello()` function so that it calls and returns the value of `flag()`. Then when you click the “Call hello() function” button you will be presented with the flag.

```
Java.perform(function(){
  var bin = Java.use("com.hpandro.androidsecurity.ui.activity.task.binary.NativeFunTaskActivity");
  bin.hello.implementation = function() { 
    var ret = this.flag();
    console.log("flag: " + ret);
    return ret;
  }
});

```

Miscellaneous: Backdoor 6
-------------------------

Next up is the 6th backdoor challenge. Unlike the other backdoor challenges this one is very well suited for Frida so let’s tackle this now. In this challenge you have a dialog where you provide a pin code that’s validated in native code. Like in the previous challenge you could reverse the native code and figure out the correct pin that way, but that’s not what I’m doing.

![](https://nibarius.github.io/learning-frida/assets/hpandro/pin.png) Please enter the correct PIN

When you submit your pin code, the native function `hello()` is called. If the pin is wrong it returns “NO”, if it’s correct it returns the flag. Instead of manually testing all the different possibilities let’s automate it with Frida.

To brute force the pin code we can change the implementation of the `hello()` function to a for loop that repeatedly calls `hello()` with all values between 0000 and 9999 until it returns something else than “NO”.

```
Java.perform(function(){
  var bd6 = Java.use("com.hpandro.androidsecurity.ui.activity.task.misc.Backdoor6Activity");
  bd6.hello.implementation = function(a) { 
    var ret = "NO";
    for (let i = 0; i <= 9999; i++) {
      ret = this.hello(("000" + i).substr(-4,4));
      if (ret != "NO") {
        console.log("correct pin found at: " + i);
        break;
      }
    }
    return ret;
  }
});

```

With this script running you can enter anything in the pin field, press the check button and you’ll have the flag after a while.

Symmetric Encryption
--------------------

Let’s put the native code behind us and move on to network related things instead. Most of the symmetric encryption challenges worked the same way: an encrypted version of the flag is sent to us from the server. Since the flag is encrypted, intercepting the network traffic won’t help. The good news is that the app decrypts the flag for us, the bad news is that the decrypted flag is not used anywhere and definitely not shown to us.

![](https://nibarius.github.io/learning-frida/assets/hpandro/aes_code.png) The decryption method from the AES task

When you have Frida at your hands that doesn’t have to be a problem. All you do is hook the decryption method and check what value it returns. Here’s one example from the AES challenge:

```
Java.perform(function(){
  var aes = Java.use('com.hpandro.androidsecurity.ui.activity.task.encryption.AESActivity');
  aes.decrypt.overload('java.lang.String', 'java.lang.String', '[B').implementation = function(a, b, c) {
    var decrypted = this.decrypt(a, b, c);
    console.log("Decrypted: " + Java.use('java.lang.String').$new(decrypted));
    return decrypted;
  }
});

```

One thing worth noting here is that some of the decryption methods return a byte array instead of a string. So to get a readable version of the flag I pass the return value through the `String` constructor which has overloads for both strings and byte arrays.

### Honorable mention: Predictable Initialization Vector

The “Predictable Initialization Vector” challenge is a bit special though. Even if there is a decode function here, it is never called at all. To solve this you have to call it yourself with the correct IV. I didn’t actually use Frida for this, so I won’t go into much details on how I solved it, but I basically copied the decompiled decrypt methods to my own small java program and brute forced the 4 digit pin.

![](https://nibarius.github.io/learning-frida/assets/hpandro/predict.png) Predictable Initialization Vector task

One thing I found really hilarious about this challenge was that you are supposed to “Predict 4 digit IV” and that the challenge is called “Predictable Initialization Vector”, so before running my brute force code I just tried one 4 digit code that felt probable and it turned out to be correct. It was indeed very predictable.

Even though I didn’t end up using Frida for this one it was probably one of my favorite challenges.

WebSocket Traffic - Web Socket Secure (WSS)
-------------------------------------------

Let’s move on to the last challenge for this time, the Web Socket Secure challenge. Like in the previous challenge the flag is sent to us from the server, but this time web sockets are used instead of http(s). This challenge was interesting to me since I’ve never tried to intercept web socket traffic before. I wasn’t able to get the web socket traffic tunneled through my Burp Suite proxy so I used [Wireshark](https://www.wireshark.org/) to capture the plain text traffic for the Web Socket task.

In the WSS challenge the traffic is encrypted and I have no way of intercepting encrypted web socket traffic, so this is where Frida comes in. If you can’t intercept the traffic on it’s way to the app you can look at it after the app has received it instead. Having multiple options at hand is always great.

The main challenge is to find where the requested data ends up after the request finishes. With some digging it’s possible to track down `WebSocketSecureActivity$createWebSocketClient$1$onTextReceived$1` where the response is handled.

![](https://nibarius.github.io/learning-frida/assets/hpandro/wss_code.png) WebSocketSecureActivity$createWebSocketClient$1$onTextReceived$1

Now that we know where the data ends up, we can hook the constructor of this class and log the data:

```
Java.perform(function(){
  // Hook the method that is called when data is submitted over the websocket and take a look at it there.
  var ws = Java.use('com.hpandro.androidsecurity.ui.activity.task.websocket.WebSocketSecureActivity$createWebSocketClient$1$onTextReceived$1');
  ws.$init.implementation = function(a, b) {
    console.log("data retrieved: " + b);
    this.$init(a,b);
  }
})

```

That’s all for this time, but when you’re ready please [continue to the third and final part](https://nibarius.github.io/learning-frida/2021/08/29/hpandro-hidden-levels) where I cover the last two challenges I solved using Frida.

Also if you have some good guides on how to intercept and analyze secure non-http(s) traffic such as wss and other protocols using TLS I would be really happy if you could share it.

[All the Frida scripts written for these challenges are available on GitHub](https://github.com/nibarius/learning-frida/blob/master/src/hpandro/)