> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jhftss.github.io](https://jhftss.github.io/Install-decrypted-IPA-on-M1-Mac/)

> As mentioned in my last blog, I can launch any decrypted iOS App on M1 Mac with free developer certif......

As mentioned in my [last blog](https://jhftss.github.io/Debug-any-iOS-Apps-on-M1-Mac/), I can launch any decrypted iOS App on M1 Mac with **free developer certificate**. Here, I will try to install the decrypted IPA dircectly without re-signing.

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/Screenshot2021_02_22_161745.jpg)

`/System/Library/CoreServices/Applications/iOS App Installer.app` is the default open method for `IPA` file. It is responsible for the decompression, invoking the installation and displaying the installing progress dialog.

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/image-20210222151143906.png)

Directly launch the app, it will exit quickly.

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/image-20210222151405717.png)

The function above will be called when application did finish launching, it will check if there is any active installation pending, and exit the program if none.

In order to debug the program with `IDA Pro`, I need to know how the target ipa file path is passed to the program. Because `IDA Pro` is hard to attach to the program **immediately when launching**.

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/image-20210222152044818.png)

The delegate function will be called automatically if we open an `IPA` file. But how to pass the `IPA` file path `URL` when I launched the program directly from `IDA Pro` ?

Luckily, I found there is another way to open the `IPA` :

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/image-20210222152958065.png)

The function has no caller. I just call it by an `IDC` script function `dbg_appcall`, and then an open file dialog poped up.

Through debugging, I found the real installation work is performed by `appinstalld` process.

Then I found a hint from the console log:

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/Screenshot2021_02_20_125118.jpg)

Set a breakpoint at the function `+[MICodeSigningVerifier _validateSignatureAndCopyInfoForURL:withOptions:error:]`, I got the call stack :

![](https://jhftss.github.io/res/2021-2-22-Install-decrypted-IPA-on-M1-Mac/Screenshot2021_02_20_143015.jpg)

From the call stack, I noted the return type of function `-[MIInstallableBundle performVerificationWithError:]` is `bool`.

I just path it to always return `true`. And it works !

If `amfid` process is patched with the method from my [last blog](https://jhftss.github.io/Debug-any-iOS-Apps-on-M1-Mac/), and the `IPA` file is re-signed with **free developer certificate** before installing, the iOS Application could be launched as usual after the installation. Don’t need the help of the command line program I wrote before. And don’t need to patch the `UIKitSystem` process, all is normal now.