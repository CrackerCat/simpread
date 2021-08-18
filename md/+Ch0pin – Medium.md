> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [valsamaras.medium.com](https://valsamaras.medium.com/)

> Read writing from +Ch0pin on Medium. Security Researcher, former Camel Rider, developer of https://gi......

Maybe the rest of the audience need some context regarding the paragraph title, but I am pretty sure that most of the malware analysts know exactly what I am talking about. So…yes, **speed** is a critical factor when it comes to malware analysis especially when you have pile up a big number of apps in your queue and you have to identify potential malware activity for each one of them.

This need was my motivation when I started [MEDUSA](https://github.com/Ch0pin/medusa) which was created taking under consideration the day-to-day tasks of a malware analyst. In this write-up, grabbing the chance that was _generated_ from [this post](https://twitter.com/sh1shk0va/status/1336482195230380032), I am going to show some basic workflows of this tool and how to use it in order to uncover a malicious behaviour.

[Tatyana](https://twitter.com/sh1shk0va) was probably right since I wasn’t able to find the app on Google Play (not much) after her post. So, I downloaded the app from [here](https://apktada.com/download-apk/ocr.numscan.text.scanner) in order to do the analysis.

Installation and a first “reconnaissance” is as simple as:

./**apkutils.py** **ocr.numscan.text.scanner_17_apktada.com.apk**

Thanks to [APKEnum](https://github.com/shivsahni/APKEnum), MEDUSA will parse the manifest file, search for URLs, IPs, S3 buckets, Activities, Services and other “Goodies” in order to present them to the analyst:

![](https://miro.medium.com/max/875/1*JSzJ8dPRyz-64oxfqA8YLQ.png)

I am staring the main medusa module alongside with the apkutils in order to load my modules:

![](https://miro.medium.com/max/875/1*-Wx5cm3IhrGA3htA6hSPKA.png)

When it comes to “Joker” apps the following four modules are my favourite:

0) modules/http_comnunications/multiple_unpinner.med **(SSL pinning)**  
1) modules/clickers/click_toll_fraud.med **(**[**Toll Frauds**](https://developers.google.com/android/play-protect/phacategories)**)**  
2) modules/file_system/input_ouput.med (**File Monitoring)**  
3) modules/helpers/unlinker.med **(Prevent delete)**

Use the “**use module/name**” medusa command to import the modules and set the first run:

![](https://miro.medium.com/max/875/1*vmV3BMH5c23G6nddug00WQ.png)

![](https://miro.medium.com/max/875/1*vGU8vHoNEbhE6EpmLzz4tg.png)

There is some interesting stuff here … , e.g. calls to the **Telephony Manager** (network operator), some **motion events** out of the blue, but nothing solid so far. The jar is a legit google ads SDK, the motion events are not dispatched… Can’t come to a verdict for trojan with that…

Back to the apkutils to get a shell and open the apk using jadx:

apkutils>shell → $ jadx-gui ocr.numscan.text.scanner_17_apktada.com.apk

It can’t be a trojan without a **DexClassLoader,** so lets search for its usage:

![](https://miro.medium.com/max/875/1*98yGlKpXHwi1zR44qExSjw.png)

**ocr.c.a** stands out of everything, so lets see whats going on there**:**

![](https://miro.medium.com/max/875/1*NF2q8yKsRLxyhPUDYS4n5w.png)

The flow seems pretty straightforward:

> Check the **files** directory for a file named **.num** and if it doesn’t exist then create it. Open the **numtextscan.png** (?) from the assets, xor each byte with the value 136 and write it to the .num. Finally use the class loader to DCL the file.

What else we got here ?

> The flow is triggered inside the **ocr.c.a.a** function which takes as a parameter a Context and an instance of ‘b’ object.

It is obvious that I can extract the file from the assets, xor it and save it as dex in order to open it and see what it does. But, there is even a faster way…e.g. just trigger the **ocr.c.a.a** function.

Medusa has a special module, called scratchpad and can be used for “app specific hooks”. Lets import a **ScheduleOnMainThread** Frida call and do some necessary modifications:

![](https://miro.medium.com/max/875/1*2TqSB3cU1dX_cEI_l51_6A.png)

**medusa>**pad

![](https://miro.medium.com/max/875/1*_BWoWvBBY0O3SchWHYj1OA.png)

TL;DR I am creating an instance of the **ocr.c.a** and **ocr.b.a** class in order to trigger the **ocr.c.a.a(Context context, ocr.b.a b)** function in order to “drop” the dex file.

Lets add the scratchpad to our modules and run the app:

![](https://miro.medium.com/max/875/1*-nu9UxxkHaBui6WtKd0BpA.png)

![](https://miro.medium.com/max/875/1*PHZUp9XVoEmjbyFSAzpKSA.png)

As expected, the app drops the file to the files folder:

![](https://miro.medium.com/max/875/1*vEpC0wCOqnDiog3Ol4lpSQ.png)

Back to the apkutils:

**apkutils>** **adb → adb**: **pull** file → **adb: exit → apkutils>shell**

$ jadx-gui .num

![](https://miro.medium.com/max/875/1*3y4AyXkiPfiwHF4D3TyZTw.png)

I am sure this is not a png ! You know what… lets leave minute five’ for another post but … don’t forget to save the session:

**medusa>export ocr.txt**

… and medusa.py -r ocr.txt to start the same session :)