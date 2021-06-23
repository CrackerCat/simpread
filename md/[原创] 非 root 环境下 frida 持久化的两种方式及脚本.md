> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268175.htm)

> [原创] 非 root 环境下 frida 持久化的两种方式及脚本

非 root 环境下 frida 持久化的两种方式及脚本
============================

frida 是一个非常好用的 hook 框架，但使用中有两个问题，一是非 root 手机使用挺麻烦的，二是 frida 相较于其他 HOOK 框架没那么持久。网上的持久化大多基于 xposed、刷 ROM 或者是 virtualapp，前面两个是比较重量级，不够轻便。虚拟化技术本身就自带风险，很容易被检测到。

在 Android 端，网上教程里大部分都是使用 frida server 来进行 hook，其实还有一种使用方法为 frida gadget，此方法需要将 frida-gadget.so 注入到 apk 中，纯手动的话过于麻烦，所以这里实现两个脚本，分别用修改 smali、修改 so 来注入目标。

**我使用的 frida-gadget 版本为 14.2.18。有其他版本的需求，需要替换 tools 下的 so 文件**

方法一 调试 apk 中含有 so
-----------------

此方法相对简单。原理来自于古早的静态注入方式：[Android 平台感染 ELF 文件实现模块注入](https://gslab.qq.com/portal.php?mod=view&aid=163)

而这种注入方式有工具可以快速实现：[How to use frida on a non-rooted device](https://lief.quarkslab.com//doc/latest/tutorials/09_frida_lief.html)

该方法优点在于可以让 gadget 是第一个启动的，缺点是没有 so 的 apk 不能用。

### 1. 效果

首先运行注入脚本，获得注入且重签名后的 apk。直接安装。

![](https://bbs.pediy.com/upload/attach/202106/833877_CJSKT5J5VVHZVBE.jpg)

将 frida_script.js push 到 / data/local/tmp。frida_script.js 为你的 hook 代码：

```
Java.perform(function () {
  var Log = Java.use("android.util.Log");
  Log.e("frida-OOOK", "Have fun!");
});//android 不要使用console.log

```

打开 app 即可看到效果，app 每次启动都会成功的打印 LOG。：

![](https://bbs.pediy.com/upload/attach/202106/833877_6YXBZW6TQQTGE7C.jpg)

不想使用持久化（本地 js 脚本），也可以通过电脑连接：

![](https://bbs.pediy.com/upload/attach/202106/833877_9372N368STPP4KK.jpg)  

不使用持久化，就不必添加 config 文件，所以脚本执行不需要执行 - persistence，执行下面的就可以：

```
python LIEFInjectFrida.py apkfile  outdir  libnative-lib.so  -apksign

```

### 2. 代码

工具详细代码：[https://github.com/nszdhd1/UtilScript/blob/main/LIEFInjectFrida.py](https://github.com/nszdhd1/UtilScript/blob/main/LIEFInjectFrida.py)

运行脚本记得安装 lief（pip install lief）

其实关键代码就几行：

```
    for soname in injectsolist: #遍历apk中指定SO有哪几种架构，并添加gadget.so为依赖库。
        if soname.find("x86") != -1:
            continue
        so = lief.parse(os.getcwd()+"\\"+soname)
        so.add_library("libfrida-gadget.so")
        so.write(soname+"gadget.so")

```

方法二  apk 中没有 so
---------------

在实际情况下，并不是所有的 apk 都有 so。没有 so，方法一便没有用武之地了。

此方法呢，是通过修改 smali，调用 System.loadLibrary 来加载 so。该原理更简单，但是有一个弊端就是时机不够靠前，没有办法 hook Activity 启动之前的代码。

手动修改太麻烦，还是写一个脚本自动化注入。

此方法优点是原理简单，缺点是脚本实现麻烦，容易写 bug

### 1. 效果

首先运行注入脚本，获得注入且重签名后的 apk。直接安装。

![](https://bbs.pediy.com/upload/attach/202106/833877_D2H5UCNVTSFNMQV.jpg)  
  

![](https://bbs.pediy.com/upload/attach/202106/833877_6NTJK4CGD4ZRRNS.jpg)

frida_script.js 代码同上，同样也可以使用电脑连接：

![](https://bbs.pediy.com/upload/attach/202106/833877_3BKCEWKWHR3TQZG.jpg)

### 2. 代码

工具详细代码：[https://github.com/nszdhd1/UtilScript/blob/main/SmaliInjectFrida.py](https://github.com/nszdhd1/UtilScript/blob/main/SmaliInjectFrida.py)

关键代码：

```
    def get_launchable_activity_aapt(self): #通过aapt找到apk的启动activity
        aapt_path = os.path.join(self.toolPath, 'aapt.exe')
        cmd = '%s dump badging "%s" ' % (aapt_path, self.apkpath)
        p = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=True)
        out,err = p.communicate()
        cmd_output = out.decode('utf-8').split('\r')
        for line in cmd_output:
            #正则，pattern.search正常，pattern.match就会有问题=-=懒得解决了
            pattern = re.compile("launchable-activity: name='(\S+)'")
            match = pattern.search(line)
            if match:
                # print match.group()[27:-1]
                return match.group()[27:-1]
           
    def injectso(self):
        target_activity = self.get_launchable_activity_aapt()
        for dex in self.dexList:
            print(dex)
            if self.dexDecompile(dex):
                smali_path = os.path.join(self.decompileDir,target_activity.replace('.','\\'))+".smali"
                print(smali_path)
                with open(smali_path, 'r') as fp:
                    lines = fp.readlines()
                    has_clinit = False
                    start = 0
                    for i in range(len(lines)):  
                        #start是获取smali中，可以添加代码的位置
                        if lines[i].find(".source") != -1:
                            start = i
                        #找到初始化代码
                        if lines[i].find(".method static constructor <clinit>()V") != -1:
                            if lines[i + 3].find(".line") != -1:
                                code_line = lines[i + 3][-3:]
                                lines.insert(i + 3, "%s%s\r" % (lines[i + 3][0:-3], str(int(code_line) - 2)))
                                print("%s%s" % (lines[i + 3][0:-3], str(int(code_line) - 2)))
                                #添加相关代码
                                lines.insert(i + 4, "const-string v0, \"frida-gadget\"\r")
                                lines.insert(i + 5,
                                             "invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V\r")
                                has_clinit = True
                                break
                    #如果碰上本身没有clinit函数的apk，就需要自己添加
                    if not has_clinit:
                        lines.insert(start + 1, ".method static constructor <clinit>()V\r")
                        lines.insert(start + 2, ".registers 1\r")
                        lines.insert(start + 3, ".line 10\r")
                        lines.insert(start + 4, "const-string v0, \"frida-gadget\"\r")
                        lines.insert(start + 5,
                                     "invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V\r")
                        lines.insert(start + 6, "return-void\r")
                        lines.insert(start + 7, ".end method\r")

                    with open(smali_path, "w") as fp:
                        fp.writelines(lines)
                self.dexCompile(dex)

```

Frida 持久化检测特征
-------------

我因为方便，将 frida js 放在了 / data/local/tmp 下，如果直接放在 app 的沙箱下，这就是一个稳定的 hook 框架了。

既然做了持久化，就要从防御者角度看看哪些方面可以检测到应用被注入了。

首先，当然是内存中会有 frida-gadget.so。但这个 so 可以被重命名（我可以命名为常见的模块，比如 libBugly.so），所以检测 / proc/pid/maps 下是否有 frida-gadget 并不准确。因为 frida 有一个 config 文件，是持久化必须存在的。所以检测 libs 下是否有 lib*.so 和 lib*.config.so 是一种较为可行的方法。但是，如果你不使用持久化，或者去 github 上找到 frida 的源码修改 gaget.vala（ps. 这一点是合理的猜想，还未验证过），就可以让防御者检测不到。

```
#gaget.vala 代码片段
if ANDROID
  if (!FileUtils.test (config_path, FileTest.EXISTS)) {
   var ext_index = config_path.last_index_of_char ('.');
   if (ext_index != -1) {
    config_path = config_path[0:ext_index] + ".config.so";#修改这里，就可以检测不到。需要保持后缀不变（例如改成symbols.so）
   } else {
    config_path = config_path + ".config.so";
   }
  }
#endif

```

除去端口检测这种几乎没什么用的，还有一种比较可行的是内存扫描，扫描内存中是否有 LIBFRIDA_GADGET 关键词，具体实现网上有教程我就不介绍了。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 21 小时前 被八重嘤编辑 ，原因：