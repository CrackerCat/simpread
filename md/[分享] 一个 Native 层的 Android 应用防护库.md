> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271302.htm)

> [分享] 一个 Native 层的 Android 应用防护库

前几个月自己的应用被破解了，作为安全小白在看雪努力学习了一段时间，搞了一个 Native 层的 Android 应用防护库，不知道有没有实用价值，大佬轻拍。

 

[https://github.com/uestccokey/EZProtect](https://github.com/uestccokey/EZProtect)

 

具有一些比较孱弱的检测功能

 

**1. 检测是否被动态调试**  
有经验的破解者通常会通过 IDA 等工具来动态调试 So，以过掉 So 库中的检测功能，因此首先需要对动态调试进行拦截，通常采用  
1. 通过 ptrace 自添加防止被调试器挂载来反调试  
2. 通过系统提供的 Debug.isDebuggerConnected() 方法来进行判断

 

**2. 检测 APK 的签名**  
非常基础的检测手段，因为容易被一键破解，因此需要配合其他手段防御

 

**3. 检测 APK 的包名**  
判断 opendir("/data/data/CORRECT_PACKAGE_NAME") 是否等于 nullptr，如果没有指定私有目录的访问权限，说明不是正确的包名

 

**4. 检测是否被 Xposed 等框架注入**  
检测 "/proc/self/maps" 目录下的文件，如果出现包含 xposed、substrate、frida 等名称的文件，说明有框架正在注入

 

**5. 检测 PMS 是否被 Hook**  
通过检测 PMS 是否继承 Proxy 类可以知道是否已被 Hook

 

**6. 检测 Application 的 className 属性**  
大部分工具都会通过修改入口 Application 类，并在修改后的 Application 类中添加各种破解代码来实现去签名校验，它们常常费了很大的劲比如 Hook 了 PMS，同时为了防止你读取原包做文件完整性校验可能还进行了 IO 重定向，但偏偏忽视了对 Application 类名的隐藏，经测试该检测可以防御大部分工具的一键破解

 

**7. 检测 Application 的 allowBackup 属性**  
正常情况下应该为 false，如果检测到为 true，说明应用被 Hook 了，破解者正在尝试导出应用私有信息

 

**8. 检测应用版本号**  
破解者为了防止应用更新导致破解失效，通常会修改此 versionCode，因此需要进行检测

 

**9. 检测 Native 与 Java 层获取到的 APK 的文件路径或者大小**  
因为 pm 命令获取到的 APK 文件路径通常不容易被修改，这里与通过 JavaApi 获取到的 APK 文件路径做比较，如果文件路径或者文件大小不一致时，大概率是应用被 IO 重定向了，或者使用了 VirtualXposed 等分身软件

 

**10.Apk 和 So 文件完整性校验**  
可以获取到当前 APK 文件和此加密 So 包的文件信息用于服务器校验

 

目标是能防住纯新手小白，哈哈

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 10 小时前 被 uestccokey 编辑 ，原因：

[#混淆加固](forum-161-1-121.htm)