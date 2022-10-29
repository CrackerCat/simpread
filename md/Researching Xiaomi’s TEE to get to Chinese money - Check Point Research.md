> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [research.checkpoint.com](https://research.checkpoint.com/2022/researching-xiaomis-tee/)

> Research By: Slava Makkaveev Introduction Have you ever wondered if it is safe to make payments from ......

August 12, 2022

**Research By:** Slava Makkaveev

### Introduction

Have you ever wondered if it is safe to make payments from a mobile device? Can a malicious app steal money from your digital wallet?

According to the latest [statistics](https://www.statista.com/statistics/1227576/mobile-wallet-transactions-worldwide), the Far East and China accounted for two-thirds of the world’s mobile payments in 2021. This is about $4 billion in mobile wallet transactions. Such a huge amount of money certainly attracts the attention of hackers.

We researched the payment system built into Xiaomi smartphones powered by MediaTek chips, which are very popular in China. As a result, we discovered vulnerabilities that can allow forging of payment packages or disabling the payment system directly from an unprivileged Android application.

Trusted execution environment (TEE) has been an integral part of mobile devices for many years. Its main purpose is to process and store sensitive security information such as cryptographic keys and fingerprints. TEE protection is based on hardware extensions (such as ARM TrustZone) that keep the TEE world safe even on rooted devices or those compromised by malware.

Getting back to our questions about mobile payments, the mobile payment signatures are carried out in the TEE. So as long as the TEE is safe, so are your payments.

Trusted code written by device vendors, such as Xiaomi, and not by chip manufacturers, were not thoroughly researched, even though security management and the core of mobile payments are implemented there. Our study marks the first time Xiaomi’s trusted applications are being reviewed for security issues.

### Xiaomi’s TEE

On the Internet, you can easily find many articles about TEE architecture. Therefore, we will not go into it in detail here. Instead, we note only the key points:

*   TEE creates a virtual secure world managed by a trusted OS that runs trusted apps.
*   A trusted app implements a specific security feature.
*   The normal world OS (Android) can send commands to a trusted app and receive a response.

Xiaomi devices that are based on Qualcomm chips use QSEE trusted OS. MediaTek-based devices use Beanpod TEE or Mitee. In both cases, Xiaomi can embed and sign their own trusted applications. In our research, we focus on trusted apps of MediaTek-powered devices. The test device is the Xiaomi Redmi Note 9T 5G with MIUI Global [12.5.6.0](https://12.5.6.0/ "https://12.5.6.0") OS. This device uses Beanpod TEE.

#### File format of a trusted app

On Xiaomi devices, trusted apps are stored in the `/vendor/thh/ta` directory. Each app is represented by an unencrypted binary file. For example, the `thhadmin` app responsible for security management is presented by the `0102030405060708090a0b0c0d0e0f10.ta` file.

In Figure 1, you can see the Xiaomi trusted apps file format. Actually, we do not even need to understand all the format fields to research the app’s code, because the initial part of the Xiaomi’s app binary is the ELF file.

![](https://research.checkpoint.com/wp-content/uploads/2022/08/1.png)

**Figure 1:** Trusted app format.

A trusted app can have multiple signatures following the magic fields. The magic fields are the same across all trusted apps on the device. In addition, they are the same with the app fields of all other devices that we have checked, such as Xiaomi T11 and Xiaomi Note 8 Pro.

#### A trusted app can be downgraded

As you can see, the version control field is omitted in the trusted app’s file format. This means that an attacker can transfer an old version of a trusted app to the device and use it to overwrite the new app file. As the signature of the old app is correct, this app will be successfully loaded by the TEE.

Therefore, an attacker can bypass security fixes made by Xiaomi or MediaTek in trusted apps by downgrading them to unpatched versions.

To prove the issue, we successfully overwrote the `thhadmin` trusted app on our test device running MIUI Global 12.5.6.0 OS with an old one extracted from another device running MIUI Global 10.4.1.0 OS. The old `thhadmin` app was successfully launched, even though its code is significantly different from the original.

#### Calling a trusted app from Android

Xiaomi follows the GlobalPlatform TEE [Client API](https://globalplatform.org/wp-content/uploads/2010/07/TEE_Client_API_Specification-V1.0.pdf) Specification that defines the communication API between a client and the TEE.

The API is implemented in the `/vendor/lib/libTEECommon.so` library. The major functions are:

*   `TEEC_OpenSession`, which opens a new session between the client application and the specified trusted app. The target trusted app is identified by a UUID that matches the app’s file name.
*   `TEEC_InvokeCommand`, which invokes a command within the specified session. The command ID and up to four command arguments can be passed to the trusted app for processing.

For example, we used the following function to invoke the `thhadmin` trusted app from Android:

![](https://research.checkpoint.com/wp-content/uploads/2022/08/2.png)

The `thhadmin` app expects to receive one input buffer and one output buffer as arguments. The command ID is ignored.

An unprivileged Android application has no permissions to communicate with Xiaomi’s trusted apps. SELinux only allows access to the `teei_client_device` object for a limited number of privileged users such as `mediacodec`, `drm`, etc.

We [recently](https://research.checkpoint.com/2022/bad-alac-one-codec-to-hack-the-whole-world) discovered several vulnerabilities in the ALAC media decoder that allow an unprivileged Android application to execute arbitrary code on smartphones powered by MediaTek or Qualcomm chips from under the `mediacodec` user. Such vulnerabilities could be used by an unprivileged user to invoke Xiaomi’s trusted apps.

#### Looking for vulnerabilities in trusted apps

Xiaomi follows the GlobalPlatform TEE [Internal Core API](https://globalplatform.org/wp-content/uploads/2020/09/GPD_TEE_Internal_Core_API_Specification_v1.2.1.31_PublicRvw.pdf) Specification in implementing trusted apps. Each app exports the “TA Interface” functions that are the entry points to create the app instance, notify the instance that a new client is connecting, notify the instance when the client invokes a command, and so on.

The `TA_InvokeCommandEntryPoint` function is the perfect target for fuzzing-based vulnerability research. It handles the command ID and the buffers sent from the Android side.

It is relatively easy to build a harness to call the function. The import table needs to be patched to redirect memory management calls to their `libc` equivalent. The verification functions like `TEE_CheckMemoryAccessRights` should be nopped.

We used a classic combination of AFL and QEMU to fuzz trusted apps implemented by Xiaomi (or more accurately, implemented by MicroTrust and integrated by Xiaomi).

We discovered several vulnerabilities in the `thhadmin` trusted app that could be exploited to leak stored keys or to execute code in the context of the app. For example, the input buffer in Figure 2 results in a heap overflow, allowing the attacker to overwrite 0x41414141 bytes of heap memory with data from 0x42424242 address.

![](https://research.checkpoint.com/wp-content/uploads/2022/08/3.png)

**Figure 2:** Malformed input buffer.

### Tencent soter

Xiaomi devices have an embedded mobile payment framework named Tencent Soter that provides an API for third-party Android applications to integrate the payment capabilities. Tencent soter is a software platform held by Tencent Holdings Ltd. whose main function is to provide the ability to verify payment packages transferred between a mobile application and a remote backend server. According to Tencent, hundreds of millions Android devices support Tencent soter.

WeChat Pay and Alipay are the two largest players in the Chinese digital payment industry. Together, they account for about 95% of the Chinese mobile payments market. Each of these platforms has over [1 billion](https://www.statista.com/statistics/1271130/mobile-wallet-user-forecast-in-china) users. WeChat Pay is based on the Tencent soter.

WeChat provides an open SDK which can be integrated by a third-party Android app to make payments using the WeChat app (`com.tencent.mm`) as a proxy. Any app can request WeChat to make a payment transaction. The WeChat app then takes care of all the necessary verifications and package signatures, and notifies the applicant of the status and results of the payment.

If an app vendor wants to implement his own payment system, including the backend that stores users’ credit cards, bank accounts, etc., without being tied to the WeChat app, he can directly use the Tencent soter to verify the authenticity of transactions on its backend server. Specifically, make sure that a payment packet was sent from his app installed on a specific device and approved by the user.

#### Architecture

In Figure 3, you can see the architecture of the Tencent soter.

![](https://research.checkpoint.com/wp-content/uploads/2022/08/4.png)

**Figure 3:** Tencent soter architecture.

In Tencent soter, there are three levels of keys: device key (ATTK), application key (ASK) and business key (AuthKey). These keys are all asymmetric keys of RSA-2048. All keys are protected and stored safely by TEE.

The ATTK private key is generated in the TEE before the device leaves the factory. The public key is safely transmitted to Tencent’s TAM server by the manufacturer, and the private key is stored in the TEE.

A third-party app can ask to generate the ASK key in the TEE. After the key pair is generated, the private ASK key is stored in the TEE (or more accurately, encrypted and stored in the secure file system using the security key in the TEE) and the public key and its ATTK signature are returned to the app. The app sends the key and the signature to its backend server, which forwards it further to the TAM via the WeChat open platform interface. The TAM verifies the legitimacy of the packet using the ATTK public key. If it is legitimate, the third party saves the ASK public key on the server for future use.

For each business scenario like a payment business or login business, the app should generate a pair of the AuthKey. The generation process is similar to the ASK generation. The private key is stored in the TEE, and the public key is uploaded to the app server. The difference is that the third-party app should check the legitimacy of AuthKey by themselves using the ASK public key.

To make a payment transaction, the user must pass fingerprint authentication. Before authentication, the app needs to request a challenge factor (usually a random string) from its server as the object for signing. After user fingerprint authorization, the app sends the challenge factor, finger ID, device information and the payment data to the app server. All this data is signed by the AuthKey private key in the TEE. On the server side, the AuthKey public key is used to validate the transaction.

As you can see, all critical data storage and operations are fundamentally dependent on TEE.

Tencent soter does not provide TEE-related code, leaving the implementation to chip or device manufacturers, which is why Xiaomi implemented the `soter` trusted app (`d78d338b1ac349e09f65f4efe179739d.ta`) to store and manage key operations. In the app logs, Xiaomi named it the “wechat” app.

#### Vulnerability in the soter trusted app

To start the signing session, the `soter` app provides the `initSigh` (command ID 0x100C) function, which expects to receive the AuthKey name (alias) and the challenge strings as arguments. The Tencent soter platform defines the AuthKey alias as a concatenation of multiple persistent words and identifiers such as process ID and payment scene ID. For example, it might look like “Wechatuid777777_demo_salt_account_1_scene1”. This is a short string as well as the challenge.

The `initSigh` handler creates a session ID by concatenating the key alias and the challenge into a fixed-size buffer without checking for overflow. The code looks like this:

![](https://research.checkpoint.com/wp-content/uploads/2022/08/5.png)

The attacker can provide the challenge in a size larger than 0x198 bytes or the alias larger than 0x8C bytes to overwrite the heap after the session buffer with arbitrary values.

When the `soter` app crashes, we can find a partially encrypted crash dump in the Android kernel log:

![](https://research.checkpoint.com/wp-content/uploads/2022/08/6.png)

Xiaomi assigned CVE-2020–14125 to the issue.

This vulnerability can be exploited to execute a custom code. Xiaomi trusted apps do not have ASLR. There are examples on the Internet of exploiting such a classic heap overflow vulnerability.

In practice, our goal is to steal one of the soter private keys, not execute the code. The key leak completely compromises the Tencent soter platform, allowing an unauthorized user to sign fake payment packages.

To steal a key, we used another arbitrary read vulnerability that exists in the old version of the `soter` app (extracted from the MIUI Global 10.4.1.0). As noted, we can downgrade the app on Xiaomi devices.

#### Extracting the ATTK private key

Let’s take a look at the `TEEC_Operation` argument of the `TEEC_InvokeCommand` function, which is used to invoke trusted apps from Android. The caller can pass up to four parameters, such as a number or a buffer, to the app via the `TEEC_Operation` object. The parameter types are encoded in the `paramTypes` field. For example, if the parameter type is `TEEC_VALUE_INPUT` (1), the trusted app expects to see a number in the corresponding `params` entry. If the type is `TEEC_MEMREF_TEMP_INPUT` (5), then a pointer to a buffer is passed. TEE uses the `paramTypes` to ensure that the data sent from the insecure side matches the expectations of the trusted app. All apps must use the `TEE_CheckMemoryAccessRights` API function to check the command parameters before processing.

What happens if a trusted app wants to receive a pointer to a buffer, but we pass in a number? If the app does not check the type of the parameter, then our number is recognized as a virtual address and the app will read or write to the memory we specified. This is a vulnerability.

The old `soter` app lacks type checking of incoming parameters. We can use this issue to gain an arbitrary memory read.

If we hold the target device in our hands, it is enough to send the `HAS_AUTH_KEY` (0x100b) command to the app and specify the address we are interested in as the second parameter to read the memory content in the Android kernel log.![](https://research.checkpoint.com/wp-content/uploads/2022/08/9.png)

The `soter` app expects to receive an AuthKey alias as the parameter. It logs the string (four symbols), the address of which we specified. The log entry looks like “uid=0, name=\x92″M\xc8\x0d”. The memory value is 0xc84d2292.

If we want to read data without interacting with the log, we first need to send the `GENERATE_AUTH_KEY` (0x1008) command and specify the target address in the second parameter. The app generates the AuthKey based on the contents of the specified memory. Next, we can use the `HAS_AUTH_KEY` command to find the value that matches the generated key.

We now know how to read the memory of the `soter` app. In addition, there is no ASLR. All we need to know to steal a private key is the virtual address of the key.

To force the ATTK private key to be loaded into the app’s memory, we can call the `EXPORT_ASK_KEY` (0x1005) command. In this case the app creates an ASK packet and signs it with the ATTK private key. The heap memory allocated for the ATTK key is freed after signing, but not reset to zero. In our PoC, the ATTK key is located at 0x38c140.

The ASK and the AuthKey private keys can be obtained in a similar way.

It is interesting to note what the soter private key is. The soter key is the RSA key, but it is not stored in the PEM format as is usually done. Instead, the app only stores the private exponent (d) and the modulus (N) of the key. OpenSSL does not have the functionality to sign with `d` and `N`, but the MatrixSSL does. The `soter` app uses the MatrixSSL to sign the SHA256 hash of data packets.

#### Reaching the soter trusted app from an unprivileged Android app

In Figure 4, you can see the implementation of the Tencent soter on Xiaomi devices.

![](https://research.checkpoint.com/wp-content/uploads/2022/08/7.png)

**Figure 4:** Tencent soter implementation.

The `com.tencent.soter.soterserver` system app exports (shares for the public access) the `SoterService` service, which provides the API to manage the soter keys. The service binds the `vendor.microtrust.hardware.soter@1.0-service` system service to communicate with the `soter` trusted app.

An unprivileged Android application has no permissions to communicate with the TEE directly, but it can use the `SoterService` as a proxy. The following Java code invokes the `initSigh` function of the `soter` app and causes a crash in the trusted app:

![](https://research.checkpoint.com/wp-content/uploads/2022/08/8.png)

Therefore, a third-party Android application can easily attack the `soter` without any user interaction. Xiaomi did not implement an app permission to protect the soter API.

### Summary

Vendor-provided TEEs are a very promising area for security research. Many security-critical features are actually implemented by OEMs and not by chip manufacturers. We discovered a set of vulnerabilities in trusted Xiaomi applications responsible for managing device security and mobile payments.

Through our tests, we observed two ways to attack the Tencent soter platform built into Xiaomi smartphones and used by millions of users in China for mobile payments.

An unprivileged Android application could exploit the CVE-2020-14125 vulnerability to execute code in the `soter` trusted app and forge payment packets. This vulnerability was patched by Xiaomi in June 2022.

In addition, we showed how the downgrade vulnerability in Xiaomi’s TEE can be chained with the arbitrary read vulnerability in the old version of the `soter` app to steal the soter private keys. The presented read vulnerability has been patched by Xiaomi. The downgrade issue, which has been confirmed by Xiaomi to belong to a third-party vendor, is being fixed.