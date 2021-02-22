> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [darvincitech.wordpress.com](https://darvincitech.wordpress.com/2020/07/18/all-your-crypto-keys-belongs-to-me-in-android-virtual-containers/)

Recently I and my friend [Vikas](https://github.com/su-vikas) presented [our research](https://github.com/su-vikas/conbeerlib/blob/master/android_virtual_containers_slides.pdf) about risks for apps executing inside virtual containers aka cloning apps([VirtualApp](https://github.com/asLody/VirtualApp), [ParallelSpace](https://play.google.com/store/apps/details?id=com.lbe.parallel.intl), [DualSpace](https://play.google.com/store/apps/details?id=com.ludashi.dualspace), [Dr.Clone](https://play.google.com/store/apps/details?id=com.trendmicro.tmas), [Clone](https://play.google.com/store/apps/details?id=com.cloneapp.parallelspace.dualspace), [Multi-Parallel](https://play.google.com/store/apps/details?id=multi.parallel.dualspace.cloner) and many [more cloning apps](https://play.google.com/store/search?q=clone%20apps&c=apps)) in [Android Security Symposium](https://www.youtube.com/watch?v=J4qI_4pLdg4&t=9s). The basis for most of the risks is, apps inside the virtual container get the same UID(Unix User ID). In Android, the fundamental security of applications is based on the fact that upon installation each app is assigned a unique UID. We set out to find the different security mechanisms that can be broken when applications receive the same UID. One such risk, we found to be interesting is about fetching the cryptographic keys using Java Security Providers and Android Keystore. This blog post is all about fetching the cryptographic keys on popular apps inside virtual containers.

### A quick intro to Java Security Providers

The Java Cryptography Architecture(JCA) contains a provider architecture and a set of APIs for digital signatures, encryption (symmetric/asymmetric block/stream ciphers), key generation and management, secure random number generation, create java keystore, store/fetch/remove keys to/from keystore etc.These APIs allow developers to easily integrate cryptographic operations in their application code. Specifically, an application is not bound to a specific provider, and a provider is not bound to a specific application. Android platform, like any Java platform, includes a number of built-in providers that implement a basic set of security services.

Here are typical usages of crypto and keystore APIs by applications

```
//AES Encryption
Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
cipher.init(ENCRYPT_MODE, new SecretKeySpec(key, "AES"), new  IvParameterSpec(ivData));
byte[] output = cipher.doFinal(data);

//Keystore Operation
KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
//Loading a Keystore
char[] keyStorePassword = "123abc".toCharArray();
InputStream keyStoreData = new FileInputStream("keystore.jks");
keyStore.load(keyStoreData, keyStorePassword);     
//Setting Keys
SecretKey secretKey = generateKey(); 
KeyStore.SecretKeyEntry secretKeyEntry = new KeyStore.SecretKeyEntry(secretKey);
keyStore.setEntry("keyAlias2", secretKeyEntry, entryPassword);

```

### Android and JCA APIs

In Android Platform, most of the applications just rely on the JCA APIs for the cryptographic and keystore operations. On analysing different android applications, I found there are 5 different usages of cryptographic and keystore APIs.

1.  Default Platform Providers

```
Cipher.getInstance("AES");        
KeyStore.getInstance("BKS");

```

2. Google provides [dynamic Security provider](https://developer.android.com/training/articles/security-gms-provider#java) through Play Services. By calling Google Play services methods, your app can ensure that it’s running on a device that has the latest updates to protect against known exploits. This inserts the provider at No. 1. The downside of this usage is this provider does not have support for java keystore management. Also, there are some [devices without Google Play services](https://www.noypigeeks.com/android/huawei-honor-without-google-mobile-services/).

3. Insert an application provided Provider at No.1

```
Security.insertProviderAt(new BouncyCastleProvider(), 1);
Cipher.getInstance("AES");
KeyStore.getInstance("BKS");

```

4. Add a Provider and use JCA APIs specifying the provider

```
Security.addProvider(new BouncyCastleProvider());
Cipher.getInstance("AES","BC");
KeyStore.getInstance("BKS","BC");

```

5. Use the direct APIs from such Provider. Providers internally depend on core crypto APIs which can be called by Apps also like below

```
// AES Encryption in CBC mode with PKCS7Padding 
BlockCipherPadding padding = new PKCS7Padding();
BufferedBlockCipher cipher = new PaddedBufferedBlockCipher(new CBCBlockCipher(new AESEngine()), padding);

CipherParameters cipherParameters = new ParametersWithIV(new KeyParameter(key), iv);

cipher.init(true, cipherParameters);
int outLen = cipher.getOutputSize(inputText.length);
byte[] output = new byte[outLen];
cipher.processBytes(inputText, 0, inputText.length,  output, 0);
cipher.doFinal(output, 0);

```

### Attacking the Cryptographic Assets in Android

Following are the different ways an app’s cryptographic dependency on default provider can be exploited  
1) Execute on rooted or compromised device and patch the default provider to point to a malicious provider 2) Tamper an App to insert a malicious provider at No. 1 3) Hook the JCA APIs using popular instrumentation tools such as Frida, Xposed etc. This again requires a rooted device or a tampered application([VirtualXposed](https://github.com/android-hacker/VirtualXposed) is an exception).  
_Will it not be more ideal and interesting, if the attack can be done on a clean device and without tampering the app?  
Yes indeed. Such an ideal environment is provided by the Virtual containers._

### Attacking the Cryptographic Assets in Virtual Containers

As a proof of concept, a Cloning App was modified to demonstrate how apps depending on default provider are at risk. To perform the attack on cryptographic assets of applications, a [fake security provider](https://github.com/darvincisec/SpongyCastle-binary) is created by patching the [SpongyCastle library](https://github.com/rtyley/spongycastle) to print the key, key size, algorithm parameters. For java keystore, it prints the keystore password and the keys. In the modified Cloning App, this patched java provider(FAKE_SC) is inserted at No.1 when the guest app(apps installed inside the Virtual App) is invoked. Invariably, apps depending on default security provider will use this fake provider. Now, execute such apps and you can find the crypto keys and output in the logcat whenever crypto, keystore operation is done by the app.

### Risks for popular apps

To show the attack on some of the popular apps, I tried with [Google Authenticator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2) and [Microsoft Authenticator](https://play.google.com/store/apps/details?id=com.azure.authenticator) apps. These apps generate One Time Password (OTP) using the HMAC crypto operation where HMAC Key is the OTP Seed and OTP can be constructed from the HMAC output based on [specification](https://en.wikipedia.org/wiki/Initiative_for_Open_Authentication). The primary business asset for such 2-Factor Authentication apps is OTP Seed. If one can get the OTP seed, any number of OTPs can be generated on behalf of the user, assuming other parameters used during the computation can also be fetched. With Google authenticator, patching the crypto provider and recompiling never works because of an issue in [apktool](https://ibotpeaches.github.io/Apktool/). With Microsoft Authenticator, apktool works perfectly but the App exits due to signing certificate mismatch checks. Though this can be circumvented, the purpose here is to demonstrate the hassles of tampering an apk and to show there is another easy way to attack the apps without tampering the apk.

When executed inside the virtual container, these apps work seamlessly. I used [FreeOTP](https://freeotp.github.io/qrcode.html) to provision the OTP seed to these apps. Since, these apps depend on the default security provider, it is easy to attack with fake security provider inserted by the virtual vontainer. When these apps generate OTP, the HMAC Key and output gets printed in the logcat. A malicious host container app can export these sensitive keys and algorithm parameters to a remote server and start using them to generate OTP on behalf of the user.

Important point to note is, these apps are not inherently vulnerable. It just shows how a compromised execution environment can make such apps vulnerable leading to leak of sensitive cryptographic assets.

There are other easy attacks on these apps when executed in the virtual container. These apps store the OTP seed in clear in a database. Since the app sandbox is accessible by all apps, it is possible for any app installed in the container to fetch the OTP Seed.

### Attacks based on Android Keystore

During our presentation, we demonstrated that all apps inside the container can access each other’s keys generated/imported inside Android Keystore. To show a different example, I took a biometric authentication [sample app](https://github.com/android/security-samples/tree/master/BiometricAuthentication) and [modified](https://github.com/darvincisec/VirtualDynamicAnalysis/tree/master/Attacks/BiometricAuthentication) it to store the encrypted data using the key generated in Android Keystore after user authentication. This app when executed on a normal device has no security vulnerability. However to showcase the vulnerability created by virtual containers, a fake biometric authentication [app](https://github.com/darvincisec/VirtualDynamicAnalysis/tree/master/Attacks/BiometricAuthentication_Fake) is created. This app does the exact reverse of the original app

*   accesses the encrypted data from biometric authentication app’s sandbox
*   decrypt the encrypted data to fetch the “very secret message” after user authentication.

In a practical scenario, a malicious app can phish the user to authenticate for a different reason.

During this experiment, there is an interesting observation about a parameter [setUserAuthenticationValidityDurationSeconds](https://developer.android.com/reference/android/security/keystore/KeyGenParameterSpec.Builder#setUserAuthenticationValidityDurationSeconds(int)) used when importing/generating key inside Android Keystore. Setting this value indicates the key is authorized to be used within a duration of time after the user is successfully authenticated.If an app inside the virtual container sets this parameter, then a malicious app executing in the same container can access the key within the authenticated duration window without requiring any user authentication.

### Mitigations from such Attacks

*   Prevent your apps from executing in such containers. For this, you can detect Virtual Containers using [conbeerlib](https://github.com/su-vikas/conbeerlib/tree/master/conbeerlib)(open source library developed by us) and prevent executing sensitive apps on such virtual containers.
*   Depend on google play services to install a dynamic security provider, provided there is no limitation of supporting only devices with google play services.
*   If you think your apps should run on all devices irrespective of the availability of google play services, use popular security providers such as [Bouncycastle](https://www.bouncycastle.org/), [conscrypt](https://github.com/google/conscrypt). Use them as given in _pts 3, 4 and 5 of Android and JCA APIs section_. The downside of this approach is, whenever there are security vulnerabilities found in such providers, app needs to be upgraded with the vulnerability fixes.
*   Apps which can detect hooking on libc and system APIs can detect presence of such virtual execution environment. Infact, one of the frida detection mechanisms (disk to memory checks) explained in my earlier [post](https://darvincitech.wordpress.com/2019/12/23/detect-frida-for-android/) can detect such compromised environment.

### Virtual Dynamic Analysis – PoC App

[Virtual Dynamic Analysis](https://github.com/darvincisec/VirtualDynamicAnalysis) is a modified Virtual App to enable dynamic analysis without requiring to tamper applications on the same lines as VirtualXposed. This is just a PoC app and it works well only on certain Android versions (v7.x and v8.x)

With this app following are the **list of analysis** that can be done **_without tampering_** apps.

1.  Inject Security Provider and fetch cryptographic assets of application depending on default security provider
2.  Enable debugging on release application
3.  Enable MITM attacks by providing a [network security configuration](https://developer.android.com/training/articles/security-config)
4.  Dynamically instrument application using [Frida-gadget](https://frida.re/docs/gadget/)

2, 3, 4 will be covered in subsequent post