> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247483908&idx=1&sn=af38a850da7881ee0fef9b3f85ccd10d&chksm=cebb474af9ccce5c2042b986867d81c04130bb8f660253e387b898bb3cfd749000f5a1bb58a3&scene=21#wechat_redirect)

一、Android 的反混淆工具 2

二、一键反编译 apk/aar/dex/jar3

  (1)、Windows 环境配置使用 TTDeDroid3

  (2)、Ubuntu 环境配置使用 TTDeDroid4

三、Apk 脱壳后合并多个 dex 并使用 jadx/JEB/JEB3 分析合并后的 Apk4

四、常用的 Android 逆向分析工具

**一、Android 的反混淆工具**

git clone --recursive https://github.com/CalebFenton/simplify.git

git submodule update --init --recursive

./gradlew fatjar

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHuyo6QMaCUSnFgVn3Spg3iblyib3WGW4uPwu69kwDaicj8a7ueMPqlU7gg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHMbvTneYv5aJlY04KAlyTRTcc6pmxSwiaIuW2ADmmqX3N3oicT4p6NeBg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHOnqSrMDvuoqO9ykPvtia5u4q3C19QlIM8q2RYXvMz9WWkicjhliaNjaMQ/640?wx_fmt=png)

gradlew.bat fatjar

java -jar simplify/build/libs/simplify-1.2.1.jar -h

java -jar simplify/build/libs/simplify.jar -it 'org/cf/obfuscated' -et 'MainActivity' simplify/obfuscated-app.apk

java -jar simplify/build/libs/simplify-1.2.1.jar -it 'org/cf' G:\workspace\simplify\simplify\src\test\resources\obfuscated-example -o g:\example.dex

**二、一键反编译 apk/aar/dex/jar**

git clone https://github.com/tp7309/TTDeDroid.git

**(1)、Windows 环境配置使用 TTDeDroid**

把 H:\SecurityAnalysis\TTDeDroid\bin 添加到环境变量，如图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQKM21cK2FEdPt69sPzkAAvqOx3MeXlKB3uLp7rqAQm3vVYUQEDRiaQMw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQbuqGdcE2icBhFTuAn4ibg96G7Ca8a5qNlvW4FzpvgQxdFjHKROy8YsQQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQia9B8IemjiaDEKogafmrxAgrW50Z95OqhicxNoUB3ERdiatEIb2kd5lckA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHicuKqy6rs4qUqytTnTBicgbrOwPfwyFic6JC0TFmOSsPJ3YvPAqNv1rXA/640?wx_fmt=png)

**(2)、Ubuntu 环境配置使用 TTDeDroid**

把 export PATH=$PATH:/home/gyp/gyp/SecurityAnalysis/TTDeDroid/bin 添加到~/.bashrc 中，如图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHh5dvXRlI5NAucECvM0JQxZbvgqjscTfnEgt6w9v2Yt5NKAStEfdPmA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHHVoLgUJrjoLRYTBNgGOKGjuFkAruILTVyl4WSscpJvHNxXlIaKspoA/640?wx_fmt=png)

vim ~/.bashrc

source ~/.bashrc

chmod a+x /home/gyp/gyp/SecurityAnalysis/TTDeDroid/bin/showjar

showjar test.apk

**三、Apk 脱壳后合并多个 dex 并使用 jadx/JEB/JEB3 分析合并后的 Apk**

https://github.com/Simp1er/AndroidSec.git

python3 dex2apk.py -a ****.apk -i dex_unpack/ -o ****-output.apk

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBH0y6uG5chxN9dLWGmMk7lria1Wcy1qibRTKWYvldgRTZsdAl9nsjvtib9w/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHJ4HIhnSwzKTNg4zsbUNabJ96XCibicICghxIsITQ9q3QIReFC29hJWXg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHw9hYicmAbofCK3uhuJ6xKEERMXbEqUOqDLGMFhDtaFicbicUfBGQ8JGYQ/640?wx_fmt=png)

**四、常用的 Android 逆向分析工具**

 apktool

       https://bitbucket.org/iBotPeaches/apktool/downloads/

       XJad

       https://www.lanzous.com/i2vvdvi

       Smali2JavaUI

       https://forum.xda-developers.com/showthread.php?t=2430413

       jadx

       https://github.com/skylot/jadx/releases

       JEB

       https://www.pnfsoftware.com/

      JEB3

      https://www.pnfsoftware.com/blog/jeb3-alpha-is-available/

      https://www.pnfsoftware.com/blog/category/jeb3/

      GDA

      http://www.gda.wiki:9090/

      IDA

      https://www.hex-rays.com/products/ida/news/

      https://www.hex-rays.com/products/ida/

      https://ida2020.org/

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHPptEcK1rrx8vaF9NZKHwG9ia2YurRYj5Kx1brd28IR37zxhnEwQr7Pw/640?wx_fmt=jpeg)

**欢迎各位关注公众号**