> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mabin004.github.io](https://mabin004.github.io/hint/)

> Docs ARM 调用约定 http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf 在线汇编和反汇编（基于......

Tools
=====

[](#Docs "Docs")Docs
====================

*   ARM 调用约定  
    [http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf)
    
*   在线汇编和反汇编（基于 keystone 和 capstone 的 web 版）  
    [http://shell-storm.org/online/Online-Assembler-and-Disassembler/](http://shell-storm.org/online/Online-Assembler-and-Disassembler/)
    
*   arm、x86 系统调用表  
    [https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#cross_arch-numbers](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#cross_arch-numbers)
    
*   TEE 安全研究 [https://github.com/enovella/TEE-reversing](https://github.com/enovella/TEE-reversing)
    
*   IDA 操作知识记录 [https://juejin.im/entry/5a37674d6fb9a04514642358](https://juejin.im/entry/5a37674d6fb9a04514642358)
    
*   mitre 维护的 wiki，很多攻击技术的英文表达  
    [https://attack.mitre.org/wiki/Main_Page](https://attack.mitre.org/wiki/Main_Page)
    
*   各平台下的 shellcode 收集 [http://shell-storm.org/shellcode/](http://shell-storm.org/shellcode/)
    
*   iOS APP 安全开发手册，含 URL 处理、数据存储、目录结构等  
    [https://github.com/felixgr/secure-ios-app-dev](https://github.com/felixgr/secure-ios-app-dev)
    
*   利用电话的 UI 漏洞，使受害者以为挂掉了电话，实际只是变了界面，通话仍在后台进行，达到监听的目的（iOS 上的漏洞，Android 或微信语音是否可行？强制挂掉？多个通话切换？）  
    [https://www.martinvigo.com/diy-spy-program-abusing-apple-call-relay-protocol/](https://www.martinvigo.com/diy-spy-program-abusing-apple-call-relay-protocol/)
    
*   Weex、ReactNative 等开发的跨平台应用有什么漏洞？
    
*   iOS 逆向资料汇总 [https://everettjf.github.io/2018/01/15/ios-app-reverse-engineering-stuff/](https://everettjf.github.io/2018/01/15/ios-app-reverse-engineering-stuff/)
    
*   操作系统或第三方 app 的自动填充密码功能 [http://www.s3.eurecom.fr/projects/modern-android-phishing/](http://www.s3.eurecom.fr/projects/modern-android-phishing/)
    

[](#iOS "iOS")iOS
-----------------

*   iOSRE 相关文档，包括 macos、iOS 等 [https://github.com/writeups/iOS](https://github.com/writeups/iOS)
*   iOS 固件 ROM 下载 [https://ipsw.me/#!/download](https://ipsw.me/#!/download)
*   了解和分析 iOS Crash Report [https://juejin.im/post/5c5edb37e51d457f926d2290](https://juejin.im/post/5c5edb37e51d457f926d2290)
*   iOS APP 测试环境搭建  
    [https://spaceraccoon.dev/from-checkra1n-to-frida-ios-app-pentesting-quickstart-on-ios-13](https://spaceraccoon.dev/from-checkra1n-to-frida-ios-app-pentesting-quickstart-on-ios-13)
*   （简单的静态 + 动态分析）Hunting Credentials and Secrets in iOS Apps [https://spaceraccoon.dev/low-hanging-apples-hunting-credentials-and-secrets-in-ios-apps](https://spaceraccoon.dev/low-hanging-apples-hunting-credentials-and-secrets-in-ios-apps)
*   官方安全指导 apple-platform-security-guide  
    [https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf](https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf)

[](#IoT "IoT")IoT
-----------------

*   [https://paper.seebug.org/649/](https://paper.seebug.org/649/) [https://paper.seebug.org/649/](https://paper.seebug.org/649/)

[](#渗透相关 "渗透相关")渗透相关
--------------------

*   端口与口令安全 [http://www.freebuf.com/articles/web/165340.html](http://www.freebuf.com/articles/web/165340.html)
    
*   安全运维那些洞 [https://mp.weixin.qq.com/s/5TfAF5-HR8iDA_qSIJkQ0Q](https://mp.weixin.qq.com/s/5TfAF5-HR8iDA_qSIJkQ0Q)
    

[](#Tools "Tools")Tools
=======================

*   GitHub 安全搜集
    
    *   sec-tool-list [https://github.com/alphaSeclab/sec-tool-list](https://github.com/alphaSeclab/sec-tool-list)
    *   awesome-reverse-engineering [https://github.com/alphaSeclab/awesome-reverse-engineering](https://github.com/alphaSeclab/awesome-reverse-engineering)
    *   awesome-cyber-security [https://github.com/alphaSeclab/awesome-cyber-security](https://github.com/alphaSeclab/awesome-cyber-security)
*   spy-debugger 微信调试，各种 WebView 样式调试、手机浏览器的页面真机调试。便捷的远程调试手机页面、抓包工具，支持：HTTP/HTTPS，无需 USB 连接设备  
    [https://github.com/wuchangming/spy-debugger](https://github.com/wuchangming/spy-debugger)
    
*   Rundll32.exe 打开 url
    

```
rundll32 url.dll, OpenURL file://c:\windows\system32\calc.exe

rundll32 url.dll, OpenURLA file://c:\windows\system32\calc.exe

rundll32 url.dll, FileProtocolHandler calc.exe
```

*   binary diff 工具  
    DarunGrim  
    BinDiff  
    Diaphora  
    radiff2  
    Ghidra
    
*   webshell 查杀
    
    *   windows D 盾 [http://www.d99net.net/](http://www.d99net.net/)
    *   linux rkhunter [http://rkhunter.sourceforge.net/](http://rkhunter.sourceforge.net/)
*   elasticsearch 搭建社工库 搜云数据、暗网数据、
    
*   代码美化 carbon 一个在线将代码生成高逼格的图片工具 [https://carbon.now.sh/](https://carbon.now.sh/)
    
*   ssl 流量抓取
    
    *   (hook libssl)[https://github.com/5alt/ssl_logger/](https://github.com/5alt/ssl_logger/)
    *   [https://github.com/WooyunDota/DroidSSLUnpinning](https://github.com/WooyunDota/DroidSSLUnpinning)
    *   [https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/](https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/)

*   lief 解析 elf、pe、MathO 等可执行文件的 python 库，可修改到处函数等，跨平台 （[https://lief.quarkslab.com/、https://github.com/lief-project/LIEF）](https://lief.quarkslab.com/、https://github.com/lief-project/LIEF）)
    
*   [http://lmgtfy.com/](http://lmgtfy.com/) 记录搜索一个关键字的过程，适用于那些懒得搜索就提问的人，打脸专用
    
*   DNS Rebinding 工具 [https://lock.cmpxchg8b.com/rebinder.html](https://lock.cmpxchg8b.com/rebinder.html)
    
*   打造智能家居 [https://github.com/home-assistant/home-assistant](https://github.com/home-assistant/home-assistant)
    
*   Hashcat 各种 hash 破解工具 [http://www.freebuf.com/sectool/164507.html](http://www.freebuf.com/sectool/164507.html)
    
*   Android uiautomator 的 python wrapper(自动化测试或 Fuzz)  
    [https://github.com/xiaocong/uiautomator](https://github.com/xiaocong/uiautomator)
    
*   Unicode 同形字，对抗和谐必备 [http://www.unicode.org/Public/security/latest/confusablesSummary.txt](http://www.unicode.org/Public/security/latest/confusablesSummary.txt)
*   营业执照生成 [http://zz.iis1.cn/](http://zz.iis1.cn/)
    
*   巧笔输入法 [https://hanzi.unihan.com.cn/Qpen](https://hanzi.unihan.com.cn/Qpen) 当你不知道怎么把一个汉字输入到电脑里时，可以手写识别
    
*   安卓设备的取证、数据读取解析工具 [https://github.com/den4uk/andriller](https://github.com/den4uk/andriller)
    
*   查看 apk、dex、elf 使用的加固、加壳、混淆工具 [https://github.com/rednaga/APKiD](https://github.com/rednaga/APKiD)
    
*   专利标准查询
    
    *   国家标准查询 [http://openstd.samr.gov.cn/](http://openstd.samr.gov.cn/)
    *   专利搜索 [http://pss-system.cnipa.gov.cn/](http://pss-system.cnipa.gov.cn/)
    *   美国标准 [http://everyspec.com/](http://everyspec.com/)
*   人脸识别相关论文搜集 [https://github.com/ChanChiChoi/awesome-Face_Recognition#face-anti-spoofing](https://github.com/ChanChiChoi/awesome-Face_Recognition#face-anti-spoofing)
    
*   jeb 脚本自动生成 frida、xposed hook 代码  
    [https://github.com/LeadroyaL/JebScript](https://github.com/LeadroyaL/JebScript)
    
*   AI 处理图片、视频工具
    
    *   补帧 DAIN (Depth-Aware Video Frame Interpolation)[https://github.com/baowenbo/DAIN](https://github.com/baowenbo/DAIN)
    *   增大分辨率 ESRGAN [https://github.com/xinntao/ESRGAN](https://github.com/xinntao/ESRGAN)
    *   老照片上色 DeOldify [https://github.com/jantic/DeOldify](https://github.com/jantic/DeOldify)
    *   换脸 DeepFake [https://www.deepfaker.xyz/](https://www.deepfaker.xyz/)
*   常用正则
    
    *   提取 url  
        curl [https://www.baidu.com](https://www.baidu.com) | grep -Eo “(http|https)://[a-zA-Z0-9./?=_-]*”
*   不同手机的工程模式
    
    *   华为
        *   *#*#2846579#*#* 含打开 APP log
    *   oppo
        *   *#*#4636#*#*
    *   xiaomi
        *   *#*#6484#*#*
    *   vivo
        *   *#558#
*   各类子域名搜集工具  
    [https://www.freebuf.com/articles/web/246555.html](https://www.freebuf.com/articles/web/246555.html)