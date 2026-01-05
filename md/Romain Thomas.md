> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.romainthomas.fr](https://www.romainthomas.fr/post/26-01-dexprotector/)

> This blog post provides a high-level overview of DexProtector's security features and their limitatio......

Introduction
------------

[DexProtector](https://licelus.com/products/dexprotector) is a comprehensive security solution providing a complete set of features to protect mobile apps (Android/iOS) against different threats including reverse engineering and malware.

Its core capabilities include:

*   Obfuscation, Encryption, and Virtualization
*   RASP (Runtime Application Self-Protection)
*   Anti-Tampering and Integrity Control

This protector renewed my interest when I noticed that [Revolut](https://play.google.com/store/apps/details?id=com.revolut.revolut) is using this solution to protect their apps. Interestingly, I also found that the solution was used by [Live Net TV](https://www.romainthomas.fr/post/26-01-dexprotector/img/livenet.png), a dubious IPTV application.

This post synthesizes my findings from a deep dive into DexProtector.

You can download the original LiveNet APK for reference here: [com.playnet.androidtv.ads.5.0.1.apk](https://www.romainthomas.fr/post/26-01-dexprotector/assets/com.playnet.androidtv.ads.5.0.1.apk)[1](#fn:1)

Bootstrap
---------

DexProtector uses a complex loading chain designed to hinder static/dynamic analysis and memory dumping.

It all starts with a custom class named `Protected<suffix>` which is injected in the main package of the application and referenced in the `AndroidManifest.xml`:

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="56"
    android:version
    package="com.playnet.androidtv.ads">
  <application
    android:>
  </application>
</manifest>


```

This class is involved in various stages of DexProtector but first, it is used to load a native library: `libdpboot.so`:

```
package com.playnet.androidtv;

public class ProtectedLiveNetTV extends Application {
  @Override
  protected void attachBaseContext(Context context) {
      super.attachBaseContext(context);
      try {
          DeFcpynjg(); // Basic integrity check
          System.loadLibrary("dpboot");
          oagfhBoAe(); // Load libdexprotector.so (or libdexprotector_h.so)
      } catch (Throwable th) {
          ProtectedLiveNetTV$R$id.EfxsfkH(this, th);
      }
  }
}


```

`libdpboot.so` serves multiple purposes, one of which is loading `libdexprotector.so`. `libdexprotector.so` is loaded by a Java native function (named `oagfhBoAe` in the previous example) that uses the JNI to call `System.loadLibrary("dexprotector")`.

`libdexprotector.so` is a custom ELF loader[2](#fn:2) that is responsible for decrypting and mapping the final protected payload into memory.

This protected payload is embedded within the library itself:

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector.webp)

In some versions of DexProtector, the beginning of the packed library can be identified by looking for the magic bytes: `DPLF`:

```
0000fac0  44 50 4c 46 c0 b1 f2 ea e1 c6 0d 5b 45 6e fd e5  DPLF.......[En..
0000fad0  86 f2 2e c5 46 82 66 44 e7 68 b4 e1 5b 87 36 9e  ....F.fD.h..[.6.
0000fae0  09 54 ef b4 17 94 94 71 46 88 8d 47 c4 ee ba a7  .T.....qF..G....
0000faf0  e7 aa da c0 55 32 4b b3 8c 1f 09 db fc a6 04 fd  ....U2K.........
0000fb00  0e 22 04 8c d6 11 05 18 fb 93 3b 27 32 ca 97 e6  ."........;'2...
0000fb10  b2 9b 7b 87 ed 35 64 32 aa 8b 0e ee ca 1c 02 7b  ..{..5d2.......{
0000fb20  56 e9 8f c7 1e dd e1 58 4d 9b d9 ca cd 5f 38 f1  V......XM...._8.


```

In other versions, the payload is located in the last `PT_LOAD` segment:

```
-> revolut-10-109 git:(main) ✗ readelf -lW ./libdexprotector.so

Elf file type is DYN (Shared object file)
Entry point 0x0
There are 8 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x0001c0 0x0001c0 R   0x8
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x0026bc 0x0026bc R E 0x4000
  LOAD           0x0026c0 0x00000000000066c0 0x00000000000066c0 0x0000f8 0x0000f8 RW  0x4000
  LOAD           0x0027b8 0x000000000000a7b8 0x000000000000a7b8 0x000a70 0x000a80 RW  0x4000
  DYNAMIC        0x0026c8 0x00000000000066c8 0x00000000000066c8 0x0000f0 0x0000f0 RW  0x8
  GNU_RELRO      0x0026c0 0x00000000000066c0 0x00000000000066c0 0x0000f8 0x001940 R   0x1
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x0
  LOAD           0x003630 0x000000000000f630 0x000000000000f630 0x057535 0x057535 RW  0x4000
   ^
   |
   +------------ Packed library


```

The most clever aspect of `libdexprotector.so` is how it derives the 32-byte key that is used to decrypt the payload.

It uses a static salt located in its library but it also uses the **runtime state** of the system linker.

The key is partially derived from the assembly code of the linker function `rtld_db_dlactivity()`.

By default, `rtld_db_dlactivity()` is an empty function (i.e. a `ret`). However, when `frida-server` is used, it hooks this function by injecting a “trampoline” It is worth mentioning that this trampoline is persistent even if `frida-server` is no longer running. This means that if `frida-server` runs at least **once**, the key will be corrupted by the **persistent** trampoline.

Consequently, the second stage won’t be executed

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-2.webp)

Given the correct computed key, `libdexprotector.so` decrypts the beginning of the payload, which starts with a header followed by ELF-like segments describing the content to be mapped into memory.

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-3.webp)

The unpacked library was originally named `libdp.so`. It is worth mentioning that neither the packed nor the unpacked library contains the original ELF header.

Instead, `libdexprotector.so` acts as a custom ELF loader that relies on its own custom header rather than using the official `Elf64_Ehdr` structure. Similarly, the segments table uses a custom structure to represent the segments that need to be mapped in memory.

When `libdexprotector.so` has finished mapping the protected-packed library, it jumps to the function referenced in the `DT_FINI_ARRAY` entry of the protected library.

During the loading phase, `libdexprotector.so` clears the different regions referenced in the dynamic table. For instance, the relocations table referenced in the `DT_ANDROID_RELA` entry is cleared with zeros once `libdexprotector.so` has processed the relocations.

This means that if attackers try to dump the protected library after it has fully loaded, they will miss critical information from the dynamic table.

`libdp.so`
----------

The protected library loaded through `libdexprotector.so` is a key component to understand most of the DexProtector’s security features.

It contains the RASP detections, the engine to load encrypted classes, the logic to load protected `assets/` etc. It’s a masterpiece of engineering and different detections are very juicy.

From a cryptography perspective, it uses various algorithms and everything is implemented following standards and good practices. In addition, DexProtector uses a highly context-sensitive approach to generate and derive key material.

Key Derivation
--------------

One of the purposes of `libdp.so` is to generate a 32-byte master key. This key is critical, as it is used to derive the subkeys necessary for various security features, such as asset decryption.

To ensure integrity, the master key is generated using specific elements that create a strong cryptographic binding to the host application.

These elements typically include:

*   The APK signature
*   Unprotected DEX files
*   The DexProtector configuration (embedded within `libdp.so`)

Because of this binding, even minimal static or dynamic modifications to the APK will result in a corrupted master key, preventing the application from executing correctly.

The key derivation process also uses the content of `libdp.so` to derive or corrupt the key. This acts as an anti-tampering measure: if an attacker attempts to hook or instrument functions within `libdp.so`, the resulting key will be invalid.

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-4.webp)

In theory, this design is robust. However, while it was challenging, I managed to develop a workaround to instrument and hook `libdp.so` without triggering these corruption mechanisms.

Ultimately, I was able to generate the valid master key without executing the protected applications (e.g., Revolut, Kaspersky). With this master key, it is straightforward to derive the subkeys required to decrypt assets and access DexProtector’s proprietary files, such as:

*   `se.dat`
*   `resources.dat`
*   `mm.dat`
*   `dp.mp3`
*   `classes.dex.dat`
*   `ic.dat`
*   `ct.dat`
*   `rcdb.dat`

Class Encryption
----------------

One of the major features provided by DexProtector is the ability to encrypt classes.

As detailed in the official documentation[3](#fn:3), this is configured by defining the target classes or packages within the `<classEncryption>` tag:

```
<classEncryption>
    <filters>
        <filter>glob:com/mypackage/**</filter>
    </filters>
</classEncryption>


```

Internally, DexProtector protects all `classes<N>.dex` files that match the classes or packages defined in the configuration. For instance, protecting the packages `com/mypackage` and `com/iptv` may require DexProtector to protect the entire `classes.dex` and `classes2.dex`.

The protected DEX files are bundled into a single file located in `assets/classes.dex.dat`. This file contains the encrypted and compressed DEX data, along with a header located at the end of the file. At runtime, the protection works by decrypting and decompressing the given DEX files and then using internal Android APIs to dynamically load the clear DEX files from memory.

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-5.webp)

Note that DexProtector implements an anti-dump mechanism to prevent an attacker from extracting the clear DEX file from memory. This mechanism works by unmapping[4](#fn:4) unused regions of the in-memory DEX files.

For instance, consider that the plain `classes.dex` is mapped in the memory region `[0x60000, 0x70000]` and that DexProtector unmaps the unused region `[0x64000, 0x68000]`. If an attacker tries to dump the whole range `[0x60000, 0x70000]`, it will trigger a `SEGV_MAPERR` because the region `[0x64000, 0x68000]` is unmapped.

Nevertheless, this protection can be defeated to access the “unprotected” DEX files:

**`com.playnet.androidtv.ads - assets/classes.dex.dat`**

*   [classes0.decrypted.dex](https://www.romainthomas.fr/post/26-01-dexprotector/assets/classes0.decrypted.dex)
*   [classes1.decrypted.dex](https://www.romainthomas.fr/post/26-01-dexprotector/assets/classes1.decrypted.dex)
*   [classes2.decrypted.dex](https://www.romainthomas.fr/post/26-01-dexprotector/assets/classes2.decrypted.dex)
*   [classes3.decrypted.dex](https://www.romainthomas.fr/post/26-01-dexprotector/assets/classes3.decrypted.dex)

When we open these unprotected DEX files, we notice that some classes exhibit obfuscated code:

```
package com.playnet.androidtv;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;


public class BootReceiver extends BroadcastReceiver {
    @Override // android.content.BroadcastReceiver
    public void onReceive(Context context, Intent intent) {
        Object objI;
        try {
            Object objI2 = LibLiveNetTV.i(1263, intent);
            if (objI2 == null ||
                !LibLiveNetTV.i(0, objI2, ProtectedLiveNetTV.s("\u5a7d")) ||
                !LibLiveNetTV.i(440, LibLiveNetTV.i(666, context),
                    LibLiveNetTV.i(3238, context, 2131951803), false) ||
                (objI = LibLiveNetTV.i(567, LibLiveNetTV.i(2489, context),
                                       LibLiveNetTV.i(1465, context))) == null)
            {
                return;
            }
            LibLiveNetTV.i(904, objI, 268435456);
            LibLiveNetTV.i(1054, context, objI);
        } catch (Exception e) {
            LibLiveNetTV.i(69, e);
        }
    }
}


```

This output demonstrates the presence of two additional security layers: string encryption and indirect method/field access (invocation hiding).

String Encryption
-----------------

As described in the official documentation[3](#fn:3), developers can protect sensitive strings by applying the `<stringEncryption>` tag in their configuration:

```
<stringEncryption>
    <filters>
        <filter>glob:!**/**</filter>
        <filter>glob:com/test/**</filter>
    </filters>
</stringEncryption>


```

From an implementation perspective, this protection works by replacing sensitive strings with calls to a native function. This function accepts an encoded index (passed as a string) to retrieve the original string.

Consider the following example:

```
public class BootReceiver extends BroadcastReceiver {
    @Override // android.content.BroadcastReceiver
    public void onReceive(Context context, Intent intent) {
        // ...
        String clear = ProtectedLiveNetTV.s("\u5a7d");
        // ...
    }
}


```

In this example, the native function is `ProtectedLiveNetTV.s`, and the index is `0x5a7d` (represented by the character `\u5a7d`).

The native function `ProtectedLiveNetTV.s(String enc)` is implemented within the library `libdp.so` and the decryption process operates as follows:

*   **Lookup**: The function uses the external `assets/se.dat` file to convert the input index (`0x5a7d`) into a file offset.
    
*   **Retrieval**: This offset points to the specific location of `se.dat`.
    
*   **Decryption**: `ProtectedLiveNetTV.s` decrypts the data found at that offset and returns the plain-text string using a standard cryptography algorithm and a custom one.
    

The algorithm used to decrypt the strings relies on a specific key and a nonce constructed using a combination of:

1.  The string index (e.g., `0x5a7d`).
2.  The hash code of the calling class (e.g., `com.playnet.androidtv.BootReceiver`).

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-6.webp)

After that, we get the clear string `android.intent.action.BOOT_COMPLETED`.

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-8.webp)

DexProtector adds an additional layer of security by binding the decryption logic to the memory address of the native function itself.

The internal crypto context used for decryption is masked using the address of `ProtectedLiveNetTV.s`. This acts as an integrity check: if an attacker attempts replace the function during `env->RegisterNative`, the memory address will not match. Consequently, the unmasking process will fail, the crypto context will be corrupted, and the string will not decrypt correctly.

Method & Field Access Protection
--------------------------------

The second layer of protection focuses on obfuscating method calls and field access. This process involves transforming these operations into native invocations.

```
- context.getPackageName()
+ LibLiveNetTV.i(1465, context)


```

Similar to string encryption, developers can use the `<hideAccess>` tag to apply this protection to specific packages and classes defined in the filters:

```
<hideAccess>
    <filters>
        <filter>glob:!**/**</filter>
        <filter>glob:com/test/**</filter>
    </filters>
</hideAccess>


```

When an instruction requires protection, DexProtector replaces it with a call to a native bridge function (e.g., `LibLiveNetTV.i(...)`). This function accepts an index as the first parameter, followed by any arguments required by the original method or field.

This index is used to resolve the targeted method or field thanks to the asset file `assets/dp.mp3`. This file is decrypted and decompressed during the DexProtector’s initialization routine and it contains the information to make the relationship between indexes and the hidden methods or fields.

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-7.webp)

The layout of the data file is divided into four distinct sections:

**Header**

Contains integrity hashes and the number of elements in the subsequent sections.

**Elements Array**

An array of structures describing the hidden methods and fields. Each element contains references to:

*   The name (e.g., `getPackageName`)
*   The signature (e.g., `()Ljava/lang/String;`)
*   The defining class (e.g., `android/content/Context`)

**Classes Array**

An array listing the class names that own the elements in the previous section. Note that this is not an array of strings, but an array of integers serving as references into the Strings Pool.

**Strings Pool**

A collection of all string literals referenced by the previous sections.

Using the previous example, `LibLiveNetTV.i(1465, context)`:

1.  The native function `LibLiveNetTV.i` whose implementation is located in `libdp.so` takes the index `1465`.
2.  This number is used as an index into the _Elements Array_ of `dp.mp3`
3.  It resolves the mapping to: `android/content/Context.getPackageName() - ()Ljava/lang/String;`

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-9.webp)

Then, it executes the function via the JNI:

```
jclass clazz = env->FindClass("android/content/Context");
jmethodID mid = env->GetMethodID(clazz, "getPackageName", "()Ljava/lang/String;");
return env->CallObjectMethod(context, mid);


```

Recovery
--------

Based on our understanding of the string encryption and hidden access mechanisms, we can now strip the protections from the different DEX files using [Redex](https://github.com/facebook/redex).

Redex is a DEX bytecode optimizer that provides a reliable framework for reading, writing, and analyzing `.dex` files. It also offers facilities to orchestrate and configure passes and performing both type inference and abstract interpretation. These features make it the ideal tool to strip these protections.

To achieve this, we create two custom passes, one targeting each protection mechanism:

```
{
  "redex" : {
    "passes" : [
      "StringEncryption",
      "RecoverHiddenAccess",

      "PeepholePass",
      "ConstantPropagationPass",
      "ResultPropagationPass",

      "RegAllocPass",
      "CopyPropagationPass",
      "LocalDcePass",

      "ReduceGotosPass"
     ]
  },
  "RecoverHiddenAccess": {
    "info": "/home/romain/research/dexprotector/livenet/dp.mp3"
  },
  "StringEncryption": {
    "se_dat_file": "/home/romain/research/dexprotector/livenet/se.dat.clear"
  },
}


```

These passes work by identifying calls to the obfuscation wrappers, specifically `ProtectedLiveNetTV.s()` or `LibLiveNetTV.i()`. The system then replaces these calls with the recovered data:

1.  Strings are restored using the `se.dat` file.
2.  Methods/Fields are restored using the `dp.mp3` file.

The output is an unprotected DEX file.

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-10.webp)

To verify the effectiveness of the Redex approach, you can compare the files below:

*   **Before Redex:** [classes2.decrypted.dex](https://www.romainthomas.fr/post/26-01-dexprotector/assets/classes2.decrypted.dex)
*   **After Redex:** [classes2.unprotected.dex](https://www.romainthomas.fr/post/26-01-dexprotector/assets/classes2.unprotected.dex)

This Redex-based deobfuscation approach has been successfully tested on other applications secured by DexProtector (examples below).

package com.revolut;public final class BuildConfig { // [...] public static final int VERSION_CODE = 1010907515; public static final String CHECKOUT_REVOLUT_HOST = ProtectedRevolutApplication.s("\u0000"); public static final String APPS_FLYER_KEY = ProtectedRevolutApplication.s("\u0001"); public static final String MY_INFO_LOGIN_URL = ProtectedRevolutApplication.s("\u0002"); public static final String APPLICATION_ID = ProtectedRevolutApplication.s("\u0003"); public static final String CHAT_HOST = ProtectedRevolutApplication.s("\u0004"); public static final String SSL_HOST_PATTERN_KEY = ProtectedRevolutApplication.s("\u0005"); public static final String REVOLUT_VISION_HOST_URL = ProtectedRevolutApplication.s("\u0006"); public static final String REAUTH_CLIENT_ID = ProtectedRevolutApplication.s("\u0007"); public static final String PAY_WITH_REVOLUT_REDIRECT_HOST = ProtectedRevolutApplication.s("\b"); public static final String REVOLUT_SSO_HOST_URL = ProtectedRevolutApplication.s("\t"); public static final String OPEN_BANKING_HOST = ProtectedRevolutApplication.s("\n"); public static final String SSO_HCAPTCHA_SITE_KEY = ProtectedRevolutApplication.s("\u000b"); public static final String CHAT_HOST_WEB_SOCKET = ProtectedRevolutApplication.s("\f"); public static final String GOOGLE_CLOUD_MESSAGING_SENDER_ID = ProtectedRevolutApplication.s("\r"); public static final String REVOLUT_WEB_3_HOST = ProtectedRevolutApplication.s("\u000e"); public static final String REVOLUT_SERVER_API = ProtectedRevolutApplication.s("\u000f"); public static final String REVOLUT_AQUEDUCT_HOST_PROD = ProtectedRevolutApplication.s("\u0010"); public static final String PAY_WITH_REVOLUT_HOST = ProtectedRevolutApplication.s("\u0011"); public static final String REVOLUT_AQUEDUCT_HOST_DEV = ProtectedRevolutApplication.s("\u0012"); public static final String REVOLUT_AQUEDUCT_HOST = ProtectedRevolutApplication.s("\u0013"); public static final String REVOLUT_ASSETS_URL = ProtectedRevolutApplication.s("\u0014"); public static final String BUILD_TYPE = ProtectedRevolutApplication.s("\u0015"); public static final String SSO_GOOGLE_SIGN_IN_SERVER_CLIENT_ID = ProtectedRevolutApplication.s("\u0016"); public static final String MANUAL_VERSION_SUFFIX = ProtectedRevolutApplication.s("\u0017"); public static final String CHAT_BOT_ID = ProtectedRevolutApplication.s("\u0018"); public static final String PAYMENT_PROFILES_HOST_V2 = ProtectedRevolutApplication.s("\u0019"); public static final String PAYMENT_PROFILES_HOST_V1 = ProtectedRevolutApplication.s("\u001a"); public static final String VERSION_NAME = ProtectedRevolutApplication.s("\u001b"); public static final String RETAIL_PACKAGE_NAME = ProtectedRevolutApplication.s("\u001c"); public static final String REVOLUT_SSO_CLIENT_ID = ProtectedRevolutApplication.s("\u001d"); public static final String OPEN_BANKING_HOST_BRAZIL = ProtectedRevolutApplication.s("\u001e"); public static final String RETAIL_API_URL = ProtectedRevolutApplication.s("\u001f");}

package com.revolut;public final class BuildConfig { // [...] public static final int VERSION_CODE = 1010907515; public static final String CHECKOUT_REVOLUT_HOST = "checkout.revolut.com"; public static final String APPS_FLYER_KEY = "pzfbMY**********CDtdy"; public static final String MY_INFO_LOGIN_URL = "https://api.myinfo.gov.sg/com/v3/authorise"; public static final String APPLICATION_ID = "com.revolut.revolut"; public static final String CHAT_HOST = "https://chat.revolut.com"; public static final String SSL_HOST_PATTERN_KEY = "*.revolut.com"; public static final String REVOLUT_VISION_HOST_URL = "https://vision-api.revolut.com"; public static final String REAUTH_CLIENT_ID = "o3r08********f2y5fdc"; public static final String PAY_WITH_REVOLUT_REDIRECT_HOST = "merchant-redirect.revolut.com"; public static final String REVOLUT_SSO_HOST_URL = "https://sso.revolut.com"; public static final String OPEN_BANKING_HOST = "oba.revolut.com"; public static final String SSO_HCAPTCHA_SITE_KEY = "e1dd321d-****-4505-****-605b005e705c"; public static final String CHAT_HOST_WEB_SOCKET = "https://chat.revolut.com/api/client/ws"; public static final String GOOGLE_CLOUD_MESSAGING_SENDER_ID = "58*******838"; public static final String REVOLUT_WEB_3_HOST = "https://web3.revolut.com"; public static final String REVOLUT_SERVER_API = "https://api.revolut.com"; public static final String REVOLUT_AQUEDUCT_HOST_PROD = "aqueduct.revolut.com"; public static final String PAY_WITH_REVOLUT_HOST = "merchant.revolut.com"; public static final String REVOLUT_AQUEDUCT_HOST_DEV = "aqueduct.revolut.codes"; public static final String REVOLUT_AQUEDUCT_HOST = "aqueduct.revolut.com"; public static final String REVOLUT_ASSETS_URL = "https://assets.revolut.com"; public static final String BUILD_TYPE = "release"; public static final String SSO_GOOGLE_SIGN_IN_SERVER_CLIENT_ID = "650277366363-****************.apps.googleusercontent.com"; public static final String MANUAL_VERSION_SUFFIX = "NONE"; public static final String CHAT_BOT_ID = "20ab2b05-4c74-4328-9a0e-************"; public static final String PAYMENT_PROFILES_HOST_V2 = "revolut.me"; public static final String PAYMENT_PROFILES_HOST_V1 = "pay.revolut.com"; public static final String VERSION_NAME = "10.109.1"; public static final String RETAIL_PACKAGE_NAME = "com.revolut.revolut"; public static final String REVOLUT_SSO_CLIENT_ID = "o3r************y5fdc"; public static final String OPEN_BANKING_HOST_BRAZIL = "oba-br.revolut.com"; public static final String RETAIL_API_URL = "https://api.revolut.com";}

package com.dexprotector.detector.envchecks;import java.security.cert.X509Certificate;public class KeystoreUtils { private static String keyCertificateAttestationExt; private static String keyName = ProtectedApplication.s("\u0000"); public static String getKeyAttestationExt() { Object[] objArrI; String strS = ProtectedApplication.s("\u0001"); String strS2 = ProtectedApplication.s("\u0002"); Object objI = LibApplication.i(203); if (objI != null) { return (String) objI; } try { Object objI2 = LibApplication.i(1480, ProtectedApplication.s("\u0003"), strS); Object objI3 = LibApplication.i(1479, strS); LibApplication.i(710, objI3, (Object) null, (Object) null); if (LibApplication.i(2392, objI3, LibApplication.i(131))) { objArrI = LibApplication.i(507, objI3, LibApplication.i(131)); } else { Object objI4 = LibApplication.i(1466); LibApplication.i(682, objI4, LibApplication.i(131), 4); Object objI5 = LibApplication.i(2366); LibApplication.i(968, objI5, ProtectedApplication.s("\u0004")); LibApplication.i(1729, objI2, LibApplication.i(1927, LibApplication.i(894, LibApplication.i(2240, LibApplication.i(2171, LibApplication.i(708, LibApplication.i(1045, objI4, objI5), new String[]{ ProtectedApplication.s("\u0005"), ProtectedApplication.s("\u0006"), ProtectedApplication.s("\u0007") }), true), 300), LibApplication.i(307, ProtectedApplication.s("\b"), LibApplication.i(858))))); LibApplication.i(1693, objI2); objArrI = LibApplication.i(507, objI3, LibApplication.i(131)); } byte[] bArrI = LibApplication.i(1221, (X509Certificate) objArrI[0], ProtectedApplication.s("\t")); if (bArrI == null) { LibApplication.i(199, ProtectedApplication.s("\n")); } else { LibApplication.i(199, LibApplication.i(292, bArrI, 0)); } LibApplication.i(1538, objI3, ProtectedApplication.s("\u000b")); } catch (Exceptione ) { LibApplication.i(1516, strS2, ProtectedApplication.s("\f"), e); LibApplication.i(199, ProtectedApplication.s("\r")); } LibApplication.i(373, strS2, ProtectedApplication.s("\u000e")); LibApplication.i(373, strS2, LibApplication.i(203)); return (String) LibApplication.i(203); }}

package com.dexprotector.detector.envchecks;import android.security.keystore.KeyGenParameterSpec;import android.util.Base64;import android.util.Log;import java.nio.charset.StandardCharsets;import java.security.KeyPairGenerator;import java.security.KeyStore;import java.security.cert.Certificate;import java.security.cert.X509Certificate;import java.security.spec.ECGenParameterSpec;public class KeystoreUtils { private static String keyCertificateAttestationExt; private static String keyName = "dexprotector_ec_keys"; public static String getKeyAttestationExt() { Certificate[] certificateChain; String strEncodeToString; String str = keyCertificateAttestationExt; if (str == null) { try { KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("EC", "AndroidKeyStore"); KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore"); keyStore.load(null, null); if (keyStore.containsAlias(keyName)) { certificateChain = keyStore.getCertificateChain(keyName); } else { keyPairGenerator.initialize( new KeyGenParameterSpec.Builder(keyName, 4) .setAlgorithmParameterSpec(new ECGenParameterSpec("secp256r1")) .setDigests({"SHA-256", "SHA-384", "SHA-512"}) .setUserAuthenticationRequired(true) .setUserAuthenticationValidityDurationSeconds(300) .setAttestationChallenge("getting sec info".getBytes(StandardCharsets.UTF_8)) .build()); keyPairGenerator.generateKeyPair(); certificateChain = keyStore.getCertificateChain(keyName); } byte[] extensionValue = ((X509Certificate) certificateChain[0]) .getExtensionValue("1.3.6.1.4.1.11129.2.1.17"); if (extensionValue == null) { strEncodeToString = "Not Present"; } else { strEncodeToString = Base64.encodeToString(extensionValue, 0); } keyCertificateAttestationExt = strEncodeToString; keyStore.deleteEntry("dexprotector_ec_keys"); } catch (Exception e) { Log.e("DEXPROTECTOR", "Couldn't get attestation ext", e); keyCertificateAttestationExt = "Error"; } Log.i("DEXPROTECTOR", "Ext:"); Log.i("DEXPROTECTOR", keyCertificateAttestationExt); str = keyCertificateAttestationExt; } return str; }}

package com.applisto.appcloner;import android.content.BroadcastReceiver;import android.content.Context;import androidx.annotation.NonNull;import androidx.annotation.Nullable;import java.util.List;import java.util.Map;public class Config { private static final String ACTION_CONFIG_UPDATED = ProtectedMainApplication.s("\u08a5"); public static class Version { public int versionCode; public String versionName; public boolean equals(Object obj) { return ha.i(-25080, this, obj, new String[0]); } public int hashCode() { return ha.i(-5295, this, new String[0]); } @NonNull public String toString() { Object objI = ha.i(-15428); ha.i(97946, objI, this); return (String) ha.i(-14405, objI); } } public static void onConfigLoaded(Context context) { ha.i(82382, true); Object objI = ha.i(-31151, context); Object objI2 = ha.i(2194); ha.i(4131, objI2, ProtectedMainApplication.s("\u08a6")); ha.i(88235, objI, objI2); } public static void registerOnConfigUpdatedReceiver(Context context, BroadcastReceiver receiver) { Object objI = ha.i(-31151, context); Object objI2 = ha.i(14156); String strS = ProtectedMainApplication.s("\u08a7"); ha.i(12653, objI2, strS); ha.i(11770, objI, receiver, objI2); if (ha.i(-21314)) { Object objI3 = ha.i(2194); ha.i(4131, objI3, strS); ha.i(-16580, receiver, context, objI3); } }}

package com.applisto.appcloner;import android.content.BroadcastReceiver;import android.content.Context;import android.content.Intent;import android.content.IntentFilter;import android.preference.PreferenceManager;import androidx.annotation.NonNull;import androidx.annotation.Nullable;import androidx.localbroadcastmanager.content.LocalBroadcastManager;import java.util.HashMap;import java.util.List;import java.util.Map;import org.apache.commons.lang3.builder.EqualsBuilder;import org.apache.commons.lang3.builder.HashCodeBuilder;import org.apache.commons.lang3.builder.ReflectionToStringBuilder;public class Config { private static final String ACTION_CONFIG_UPDATED = "com.applisto.appcloner.action.CONFIG_UPDATED"; public static class Version { public int versionCode; public String versionName; public boolean equals(Object obj) { return EqualsBuilder.reflectionEquals(this, obj, new String[0]); } public int hashCode() { return HashCodeBuilder.reflectionHashCode(this, new String[0]); } @NonNull public String toString() { return new ReflectionToStringBuilder(this).toString(); } } public static void onConfigLoaded(Context context) { sConfigLoaded = true; LocalBroadcastManager.getInstance(context).sendBroadcast( new Intent("com.applisto.appcloner.action.CONFIG_UPDATED")); } public static void registerOnConfigUpdatedReceiver(Context context, BroadcastReceiver receiver) { LocalBroadcastManager.getInstance(context) .registerReceiver( receiver, new IntentFilter("com.applisto.appcloner.action.CONFIG_UPDATED")); if (sConfigLoaded) { receiver.onReceive(context, new Intent("com.applisto.appcloner.action.CONFIG_UPDATED")); } }}

package com.flashget.kid.common.base;import java.security.MessageDigest;import javax.crypto.Cipher;import javax.crypto.SecretKey;import javax.crypto.SecretKeyFactory;import javax.crypto.spec.DESKeySpec;public class MyCryptoDESHelper { /* renamed from: a */ static byte[] f1132a; /* renamed from: b */ private static final String f1133b = "DES"; /* renamed from: c */ private static MyCryptoDESHelper f1134c; /* renamed from: d */ public static String m1612d(byte[] bArr, boolean z) throws Exception { int length; if (f1132a == null) { f1132a = C5145o.m29896f("FUd******Fo="); } byte[] bArr2 = f1132a; if (bArr == null || (length = bArr.length) < 8) { return null; } byte[] bArr3 = new byte[16]; System.arraycopy(bArr, 0, bArr3, 0, 8); byte[] bArr4 = new byte[8]; Cipher cipher = Cipher.getInstance("DES/ECB/NoPadding"); cipher.init(2, m1618k(bArr2)); cipher.doFinal(bArr3, 0, 8, bArr4, 0); byte[] bArr5 = new byte[length - 8]; int i = 0; int i2 = 0; int i6 = 8; while (i6 < length) { bArr5[i] = (byte) (bArr[i6] ^ bArr4[i2]); i2++; if (i2 == 8) { i2 = 0; } i6++; i++; } return new String(bArr5); } /* renamed from: o */ private String m1622o(byte[] bArr) { try { MessageDigest messageDigest = MessageDigest.getInstance("MD5"); messageDigest.reset(); messageDigest.update(bArr); return m1621n(messageDigest.digest(), ""); } catch (NoSuchAlgorithmException e) { e.toString(); throw new RuntimeException(e); } }}

package com.flashget.kid.common.base;import java.security.MessageDigest;import javax.crypto.Cipher;import javax.crypto.SecretKey;import javax.crypto.SecretKeyFactory;import javax.crypto.spec.DESKeySpec;public class MyCryptoDESHelper { /* renamed from: a */ static byte[] f1132a; /* renamed from: b */ private static final String f1133b = ProtectedSandApp.s("\u2516"); /* renamed from: c */ private static MyCryptoDESHelper f1134c; /* renamed from: d */ public static String m1612d(byte[] bArr, boolean z) throws Exception { int length; if (f1132a == null) { f1132a = C5145o.m29896f(ProtectedSandApp.s("\u2518")); } byte[] bArr2 = f1132a; if (bArr == null || (length = bArr.length) < 8) { return null; } byte[] bArr3 = new byte[16]; System.arraycopy(bArr, 0, bArr3, 0, 8); byte[] bArr4 = new byte[8]; Cipher cipher = Cipher.getInstance(ProtectedSandApp.s("\u2519")); cipher.init(2, m1618k(bArr2)); cipher.doFinal(bArr3, 0, 8, bArr4, 0); byte[] bArr5 = new byte[length - 8]; int i = 0; int i2 = 0; int i6 = 8; while (i6 < length) { bArr5[i] = (byte) (bArr[i6] ^ bArr4[i2]); i2++; if (i2 == 8) { i2 = 0; } i6++; i++; } return new String(bArr5); } /* renamed from: o */ private String m1622o(byte[] bArr) { try { MessageDigest messageDigest = MessageDigest.getInstance(ProtectedSandApp.s("\u251f")); messageDigest.reset(); messageDigest.update(bArr); return m1621n(messageDigest.digest(), ""); } catch (NoSuchAlgorithmException e) { e.toString(); throw new RuntimeException(e); } }}

  It is worth mentioning that this app, which has been downloaded over 10 million times, uses weak `DES/ECB-MD5` cipher suite along with clear and **explicit** `http://` communications. (c.f., `network-security-config.xml`)

Assets Protections
------------------

Sensitive application data is often stored within files attached to the APK/XAPK. These assets can include certificates, images, Machine Learning models, or serialized keystores. DexProtector provides a means to protect these embedded resources.

According to the documentation[3](#fn:3), asset protection can be configured using the following structure:

```
<resourceEncryption>
    <assets>
        <filters>
            <filter>glob:cert/**</filter>
        </filters>
    </assets>
    <res>
        <filters>
            <filter>glob:raw/**</filter>
        </filters>
    </res>
    <root>
        <filters>
            <filter>glob:fonts/**</filter>
        </filters>
    <strings>
        <filters>
            <filter>my_api_key</filter>
            <filter>glob:mobile_token*</filter>
            <filter>glob:payments_**</filter>
            <filter>glob:sensitive_strings_arrays_etc*</filter>
        </filters>
    </strings>
</resourceEncryption>


```

To demonstrate this protection, I will analyze the application [com.dexprotector.detector.envchecks](https://play.google.com/store/apps/details?id=com.dexprotector.detector.envchecks). The `.xapk` can be downloaded here: [`com.dexprotector.detector.envchecks.2.1.xapk`](https://www.romainthomas.fr/post/26-01-dexprotector/assets/com.dexprotector.detector.envchecks.2.1.xapk).

  LiveNet’s protected assets (`zpoasosdi.dat, regtbeonuev.dat, and btylusqrepu.dat`) are serialized BouncyCastle keystores used to authenticate the application on the IPTV backend. Due to the sensitive nature of this identification, I took a different application to illustrate how this protection mechanism works.

This application contains a file named `assets/chinook.db`. While the extension suggests it is a database, the file is protected and the hexdump reveals high entropy data rather than a standard file header.

```
00000000  7c 96 af 76 c2 8b 88 b5  18 e6 d7 12 d1 8d f1 a5  |...v............|
00000010  00 80 0d 00 cc 6f ce 95  30 3d 50 61 05 cd 8e 5f  |.....o..0=Pa..._|
00000020  2a 55 ae 81 85 32 24 53  cb 11 c6 a1 f1 f7 bd 56  |*U...2$S.......V|
00000030  bc 1a 67 0e 1e b5 fc 60  3c 20 6a 08 dc f1 d2 7f  |..g....`< j.....|
00000040  8e f8 7a 5b 89 14 2e 37  fc 4b 5e f9 db d9 e2 f5  |..z[...7.K^.....|
00000050  6c e4 be 83 2b 18 2e 22  00 b4 1a f1 6b d4 3c 86  |l...+.."....k.<.|
00000060  78 0a f6 0e 5c 39 fd 2b  5a b1 33 e4 6f 19 23 49  |x...\9.+Z.3.o.#I|


```

When DexProtector runs its initialization routine via `libdp.so`, it modifies the vtable of the internal class related to assets processing which is located in `libandroidfw.so`.

The modifications of the vtable are not trivial but the main idea is to intercept all the virtual calls from `android::_FileAsset::*`.

This interception occurs whenever the application attempts to access asset files using:

*   The Java API: `AssetManager.open()`
*   The Native API: `AAssetManager_open()`

When DexProtector intercepts these calls, it decrypts and potentially uncompress the underlying file on-the-fly, providing the clear content to the application.

The key and nonce required to decrypt the file are distributed across different elements, including the file header and a subkey derived from a master key. By recovering these elements, it is possible to decrypt the asset manually and reveal the original content.

```
00000000  53 51 4c 69 74 65 20 66  6f 72 6d 61 74 20 33 00  |SQLite format 3.|
00000010  04 00 01 01 00 40 20 20  00 00 00 19 00 00 03 60  |.....@  .......`|
00000020  00 00 00 00 00 00 00 00  00 00 00 22 00 00 00 01  |..........."....|
00000030  00 00 00 00 00 00 00 00  00 00 00 01 00 00 00 00  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 19  |................|
00000060  00 2d e2 1e 05 00 00 00  07 03 dd 00 00 00 00 19  |.-..............|


```

You can find the encrypted and decrypted files here:

*   [`chinook.db`](https://www.romainthomas.fr/post/26-01-dexprotector/assets/chinook.db)
*   [`chinook.decrypted.db`](https://www.romainthomas.fr/post/26-01-dexprotector/assets/chinook.decrypted.db)

The other mechanisms used by DexProtector to protect resources under the tags `<res>, <strings>` are similar but less sophisticated. They consist of hooking internal Android API like `android.content.res.StringBlock.{nativeGetString, nativeGetResourceStringArray}` and `android/content/res/AssetManager.nativeGetResourceIdentifier` to decrypt the protected content on-the-fly.

RASP
----

DexProtector uses state-of-the-art RASP mechanisms that secure both its core and the application against tampering.

For instance, it bypasses the standard `PackageManager` API in favor of raw Binder communication to detect installed root-related packages (such as `com.zachspong.temprootremovejb`).

Developers can enable these protections using the following configuration:

```
<antiDebug>true</antiDebug>
<antiEmulator>true</antiEmulator>
<antiManualInstall>true</antiManualInstall>
<antiMalware>true</antiMalware>
<runtimeChecks/>


```

When DexProtector flags a threat (such as hooking), it typically records the detection and defers its reaction to a later point in the execution flow.

However, if a threat occurs very early during startup, it may trigger immediate countermeasures, such as corrupting the master key or terminating the application.

Despite these measures, these detections are susceptible to bypass and reverse engineering in a quasi-systematic way:

![](https://www.romainthomas.fr/post/26-01-dexprotector/img/libdexprotector-11.webp)

Conclusion
----------

DexProtector provides a post-build, no-code solution requiring minimal configuration by developers to protect their mobile applications. While this approach is appealing, it introduces a generic design that weakens the solution: successfully reverse engineering one instance of DexProtector enables a scalable attack on all applications protected by this tool (see [Annexes](#annexes)).

Although DexProtector uses a highly context-sensitive approach to derive cryptographic material, this is insufficient to prevent key recovery and access protected assets.

DexProtector remains a good solution for protecting assets and IP but its limitations must be weighed against the sensitivity of the content being secured.

You can find additional material in this repo: [romainthomas/dexprotector](https://github.com/romainthomas/dexprotector)

_These different weaknesses were shared with Licel ahead of time._

### Annexes

List of applications successfully unprotected:

<table><thead><tr><th>App</th><th>Version</th></tr></thead><tbody><tr><td><a href="https://play.google.com/store/apps/details?id=com.revolut.revolut"><code>com.revolut.revolut</code></a></td><td><code>10.109.1</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=istark.vpn.starkreloaded"><code>istark.vpn.starkreloaded</code></a></td><td><code>7.1-rc</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=com.dexprotector.detector.envchecks"><code>com.dexprotector.detector.envchecks</code></a></td><td><code>2.1</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=ar.tvplayer.tv"><code>ar.tvplayer.tv</code></a></td><td><code>5.2.0</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=org.unhcr.zakat"><code>org.unhcr.zakat</code></a></td><td><code>2.1.54</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=com.Hyatt.hyt"><code>com.Hyatt.hyt</code></a></td><td><code>6.16.0</code></td></tr><tr><td><a href="https://support.kaspersky.com/common/beforeinstall/16085"><code>com.kms.free</code></a></td><td><code>11.129.4.14969</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=com.flashget.parentalcontrol"><code>com.flashget.parentalcontrol</code></a></td><td><code>1.3.6.0</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=com.belongtail.ai"><code>com.belongtail.ai</code></a></td><td><code>2.8.4</code></td></tr><tr><td><a href="https://play.google.com/store/apps/details?id=com.kidoprotect.app"><code>com.kidoprotect.app</code></a></td><td><code>11.1</code></td></tr></tbody></table>