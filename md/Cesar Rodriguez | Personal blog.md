> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cmrodriguez.me](https://cmrodriguez.me/blog/nsc-bypass/)

> frida android network security config bypass

**TL;DR In this blogpost I show the results of an analysis done over some frida scripts that allows the bypass of the network security configuration in Android. I show a new way to bypass the configuration. Also I will show the procedure used to test the scripts in multiple scenarios, and an analysis of the reason some scripts do not work in all the test-cases.**

A couple of months ago I was in the middle of an Android application security assessment. One of the first steps while preparing the environment to start the pentest is configuring the application to bypass the network security config (check this link for references regarding network security config: [https://developer.android.com/training/articles/security-config)](https://developer.android.com/training/articles/security-config)). As I am a frida fan, I tend to do everything with it (there are other alternatives to do this: [https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2017/november/bypassing-androids-network-security-configuration/)](https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2017/november/bypassing-androids-network-security-configuration/)).

I downloaded two or three scripts at that moment, but when I run it in my Android 7.1.0, none of those worked. That was the reason I started to analyze how the network security config worked, and how to bypass it with frida. The first thing I did was to generate different test cases. I tried to choose the most popular ones:

*   OKHttp (it is also used in retrofit and volley also)
*   HttpsURLConnection
*   WebView (Chromium in API 25 and 26, and Default WebView in API 24).

Then I generated three applications with different network security config:

1.  An application that used the default NSC configuration (without network-security-config xml file) - called BypassNSC
2.  An application with a NSC file that used only system certificates - called BypassNSC2.
3.  An application with a NSC file that enforced the certificate pinning - called BypassNSC3.

At the time of evaluating the scripts with my test suite, only one worked in the 100% of the cases. I also developed an alternative that also works in all of the cases.

The code that parses and validates the network security config is in the Android SDK, so it might change between versions. I tested the scripts with the versions 24,25 and 26.

The following github repository has all the applications I developed and the scripts I used.

[https://github.com/CesarMRodriguez/network-security-config-frida](https://github.com/CesarMRodriguez/network-security-config-frida)

The scripts are called:

*   network-security-config-bypass-1.js
*   network-security-config-bypass-2.js
*   network-security-config-bypass-3.js
*   network-security-config-bypass-cr.js <– this is the one I created

The following is a screenshot of the table I created with the results of the analysis:

![](https://cmrodriguez.me/images/featured-post/nsc-table.png)

I analysed all the scripts I downloaded to know why it didn’t work in some of the scenarios.

### network-security-config-bypass-1.js

Original reference: [Link](https://codeshare.frida.re/@tiiime/android-network-security-config-bypass/)

In this case the script basically modifies the getEffectiveCertificatesEntryRefs method from the NetworkSecurityConfig.Builder class. This method returns the list of valid certificates. In a standard Android configuration, the list of certificates it returns is the ones installed in the system. In this script, the certificates installed by the user are also returned. So it is logical that the bypass for the first two applications worked, but I was surprised that the ones with the certificate pinning configuration also worked. The method that validates the pinning is the following one:

```
android.security.net.config.NetworkSecurityTrustManager.checkPins


```

The following stacktrace shows the path executed till it gets to the checkPins function:

```
    at android.security.net.config.NetworkSecurityTrustManager.checkPins(Native Method)
    at android.security.net.config.NetworkSecurityTrustManager.checkServerTrusted(NetworkSecurityTrustManager.java:95)
    at android.security.net.config.RootTrustManager.checkServerTrusted(RootTrustManager.java:88)
    at com.android.org.conscrypt.Platform.checkServerTrusted(Platform.java:178)
    at com.android.org.conscrypt.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:596)
    at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
    at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:357)
    ...


```

If the patch is not executed, the following exception is thrown when it gets to that method:

```
   Caused by: java.security.cert.CertificateException: Pin verification failed
        at android.security.net.config.NetworkSecurityTrustManager.checkPins(NetworkSecurityTrustManager.java:148)
        at android.security.net.config.NetworkSecurityTrustManager.checkServerTrusted(NetworkSecurityTrustManager.java:95)
        at android.security.net.config.RootTrustManager.checkServerTrusted(RootTrustManager.java:88)
        at com.android.org.conscrypt.Platform.checkServerTrusted(Platform.java:178)
        at com.android.org.conscrypt.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:596)
        at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
        at com.android.org.


```

Let’s check the implementation of this method (API 25):

```
    private void checkPins(List<X509Certificate> chain) throws CertificateException {
        PinSet pinSet = mNetworkSecurityConfig.getPins();
        if (pinSet.pins.isEmpty()
                || System.currentTimeMillis() > pinSet.expirationTime
                || !isPinningEnforced(chain)) {
            return;
        }
        Set<String> pinAlgorithms = pinSet.getPinAlgorithms();
        Map<String, MessageDigest> digestMap = new ArrayMap<String, MessageDigest>(
                pinAlgorithms.size());
        for (int i = chain.size() - 1; i >= 0 ; i--) {
            X509Certificate cert = chain.get(i);
            byte[] encodedSPKI = cert.getPublicKey().getEncoded();
            for (String algorithm : pinAlgorithms) {
                MessageDigest md = digestMap.get(algorithm);
                if (md == null) {
                    try {
                        md = MessageDigest.getInstance(algorithm);
                    } catch (GeneralSecurityException e) {
                        throw new RuntimeException(e);
                    }
                    digestMap.put(algorithm, md);
                }
                if (pinSet.pins.contains(new Pin(algorithm, md.digest(encodedSPKI)))) {
                    return;
                }
            }
        }

        // TODO: Throw a subclass of CertificateException which indicates a pinning failure.
        throw new CertificateException("Pin verification failed");
    }


```

This method receives a list of certificates returned by the site when the communication starts. The first thing it does is checking some conditions:

1.  the pinset (list of pins loaded during the configuration instantiation) is empty.
2.  the pinset has expired at the moment of validation
3.  pinning is NOT enforced by configuration

If any of those conditions are true, pinning validation is ignored. In case the validation must be achieved, the application checks if any of the certificates provided by the site matches with one of the pins defined in the network security config file. In that case the validation is successful. If that does not happen, the method throws the exception shown in the previous stacktrace.

Initially I thought the issue was in the for loop that analyzed each of the certificates, so I added the following log in the frida script:

```
    var Pin = Java.use("android.security.net.config.Pin");
    Pin.$init.implementation = function (digestAlg, digest) {
        var bt = Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new());
        console.log("\nBacktrace:\n" + bt);
        console.log(digestAlg);
        return this.$init(digestAlg,digest);
    }


```

It should print each of the Pins created during the evaluation. When I run the application with the changes I found that the application wasn’t getting to that point.

I thought it could be because of the for loop, so I added a log in the call to pinSet.getPinAlgorithms(), that was executed before the for loop:

```
    var PinSet = Java.use("android.security.net.config.PinSet");
    PinSet.getPinAlgorithms.implementation = function () {
        var bt = Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new());
        console.log("\nBacktrace:\n" + bt);
        return this.getPinAlgorithms();
    }


```

And nothing was printed. My next idea was that the first if was evaluated to true, and that was the reason it didn’t get to the rest of the method. To check which conditions were the ones that made the method exit, I added the following lines to the script:

```
     NetworkSecurityTrustManager.checkPins.implementation = function (pins) {
        var bt = Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new());
        console.log("\nBacktrace:\n" + bt);
        pinSet = this.mNetworkSecurityConfig.value.getPins();
        console.log("pinSet.pins.value.isEmpty: " +pinSet.pins.value.isEmpty());
        console.log("isPinningEnforced: " +this.isPinningEnforced(pins));
        console.log("pins.isEmpty: " +pins.isEmpty());
        console.log(System.currentTimeMillis())
        console.log(pinSet.expirationTime.value);
        console.log(System.currentTimeMillis() > pinSet.expirationTime.value);
        this.checkPins(pins); 
    }


```

So after running the application I got the following output:

```
pinSet.pins.value.isEmpty: false
isPinningEnforced: false <-- this condition is the problematic one
pins.isEmpty: false
1562031248274
9223372036854775807
false


```

As it can be seen, the isPinningEnforced was evaluated to false and then as it was negated, it transformed all the expression to true.

The method has the following implementation:

```
    private boolean isPinningEnforced(List<X509Certificate> chain) throws CertificateException {
        if (chain.isEmpty()) {
            return false;
        }
        X509Certificate anchorCert = chain.get(chain.size() - 1);
        TrustAnchor chainAnchor =
                mNetworkSecurityConfig.findTrustAnchorBySubjectAndPublicKey(anchorCert);
        if (chainAnchor == null) {
            throw new CertificateException("Trusted chain does not end in a TrustAnchor");
        }
        return !chainAnchor.overridesPins;
    }


```

I knew the chain wasn’t empty, as I already executed the evaluation of the List, so the problem should ne in the findTrustAnchorBySubjectAndPublicKey, which returned a chainAnchor with the attribute overridesPins in true.

The findTrustAnchorBySubjectAndPublicKey method is implemented in the NetworkSecurityConfig class:

```
    public TrustAnchor findTrustAnchorBySubjectAndPublicKey(X509Certificate cert) {
        for (CertificatesEntryRef ref : mCertificatesEntryRefs) {
            TrustAnchor anchor = ref.findBySubjectAndPublicKey(cert);
            if (anchor != null) {
                return anchor;
            }
        }
        return null;
    }


```

It iterates over all the CertificatesEntryRef created during the seting up, and returns the first one that matches the SubjectAndPublicKey. In this scenario it will always return the ones from the proxy. After reading the source code I went to the CertificatesEntryRef class to check where the classes were instantiated, and found that the only constructor was the following one:

```
    public CertificatesEntryRef(CertificateSource source, boolean overridesPins) {
        mSource = source;
        mOverridesPins = overridesPins;
    }


```

If you read the frida script again, you will see that the CertificatesEntryRef were created in the following way:

```
    NetworkSecurityConfig_Builder.getEffectiveCertificatesEntryRefs.implementation = function(){

        origin = this.getEffectiveCertificatesEntryRefs()

        source = UserCertificateSource.getInstance()
        userCert = CertificatesEntryRef.$new(source,true) <-- sets overridesPins in true
        origin.add(userCert)

    return origin
    }


```

And that is the reason why this script works for all the scenarios.

### network-security-config-bypass-2.js

Original Reference: [Link](https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2017/november/bypassing-androids-network-security-configuration/)

In this case, the only case that worked was the one where the network security config file was not present. I analyzed why this patch didn’t work. The issue is in the XmlConfigSource in the method “parseNetworkSecurityConfig”:

```
        XmlUtils.beginDocument(parser, "network-security-config");
        int outerDepth = parser.getDepth();
        while (XmlUtils.nextElementWithin(parser, outerDepth)) {
            //here it creates a NetworkSecurityconfig.Builder based on the xml structure.
            ...
        }
        ...
        NetworkSecurityConfig.Builder platformDefaultBuilder =
                NetworkSecurityConfig.getDefaultBuilder(mTargetSdkVersion); <-- this is the method changed with the script
        addDebugAnchorsIfNeeded(debugConfigBuilder, platformDefaultBuilder);

        //baseConfigBuilder is null only if the xml network-security-config is not defined in the AndroidManifest.xml 
        if (baseConfigBuilder != null) {
            baseConfigBuilder.setParent(platformDefaultBuilder);
            addDebugAnchorsIfNeeded(debugConfigBuilder, baseConfigBuilder);
        } else {
            baseConfigBuilder = platformDefaultBuilder;
        }
        ...
        mDefaultConfig = baseConfigBuilder.build();
        mDomainMap = configs;
    }


```

The build method generates the NetworkSecurityConfig entity:

```
        public NetworkSecurityConfig build() {
            boolean cleartextPermitted = getEffectiveCleartextTrafficPermitted();
            boolean hstsEnforced = getEffectiveHstsEnforced();
            PinSet pinSet = getEffectivePinSet();
            List<CertificatesEntryRef> entryRefs = getEffectiveCertificatesEntryRefs(); 
            return new NetworkSecurityConfig(cleartextPermitted, hstsEnforced, pinSet, entryRefs);
        }


```

The valid certificate sources are defined in the entryRefs variable, which is constructed as:

```
private List<CertificatesEntryRef> getEffectiveCertificatesEntryRefs() {
            if (mCertificatesEntryRefs != null) {
                return mCertificatesEntryRefs;
            }
            if (mParentBuilder != null) {
                return mParentBuilder.getEffectiveCertificatesEntryRefs();
            }
            return Collections.<CertificatesEntryRef>emptyList();
        }


```

In this case mCertificatesEntryRefs is not null, as it returns the standard SystemCertificateSource (looks for all the certs in the system ca folder). So the mParentBuilder (the one modified in the script) is never called.

Later on, when the server certificate is validated, the application calls the method NetworkSecurityConfig.findTrustAnchorBySubjectAndPublicKey, which filters the valid certificates from the system folder:

```
    public TrustAnchor findTrustAnchorBySubjectAndPublicKey(X509Certificate cert) {
        for (CertificatesEntryRef ref : mCertificatesEntryRefs) {
            TrustAnchor anchor = ref.findBySubjectAndPublicKey(cert);
            if (anchor != null) {
                return anchor;
            }
        }
        return null;
    }


```

leading to the exception thrown in the stacktrace, because there aren’t coincidences as the certificate return by the server is self-signed:

```
 com.android.org.conscrypt.TrustManagerImpl.checkTrusted(TrustManagerImpl.java:375)
        at com.android.org.conscrypt.TrustManagerImpl.getTrustedChainForServer(TrustManagerImpl.java:304)
        at android.security.net.config.NetworkSecurityTrustManager.checkServerTrusted(NetworkSecurityTrustManager.java:94)
        at android.security.net.config.RootTrustManager.checkServerTrusted(RootTrustManager.java:88)
        ...


```

### network-security-config-bypass-3.js

Original Reference: [Link](https://sensepost.com/blog/2018/tip-toeing-past-android-7s-network-security-configuration/)

This works in two of the three scenarios, because the patch is executed in the method that validates certificates. But it does not work in the third scenario because the pin validation is executed in other method, as shown in the error stacktrace:

```
at android.security.net.config.NetworkSecurityTrustManager.checkPins(NetworkSecurityTrustManager.java:148)
        at android.security.net.config.NetworkSecurityTrustManager.checkServerTrusted(NetworkSecurityTrustManager.java:95)
        at android.security.net.config.RootTrustManager.checkServerTrusted(RootTrustManager.java:88)
        at com.android.org.conscrypt.Platform.checkServerTrusted(Platform.java:203)
        at com.android.org.conscrypt.OpenSSLSocketImpl.verifyCertificateChain(OpenSSLSocketImpl.java:592)
        at com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(Native Method)
        at com.android.org.conscrypt.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:351)
        ... 25 more


```

### network-security-config-bypass-cr.js

In this case the method patched is getConfigSource, which is called when the network-security-config is parsed. As it can be seen, the reimplementation proposed creates a DefaultConfigSource, setting as a parameter the android version 23. So regardless of the file uploaded in the application, it will always retrieve the configuration of Android 23 (adds user certificates, does not validate certificate pinning, and allows http communication at http level).

### Conclusion:

I could achieve my goal, that was finding a script to bypass the network security configuration implemented by the Android SDK. Creating a new alternative to the one I found was a bonus track. During the process I learnt many aspects of the SDK, frida and the Android ecosystem. It is not easy to find bypasses. The hardest part during my analysis was the definition and setting up of all the environment to execution different test scenarios. Adding a new script, a new way to execute requests or another SDK would lead to at least 18 new cases. I guess this is the main reason many of the scripts I found worked for some cases (but not for all of them).