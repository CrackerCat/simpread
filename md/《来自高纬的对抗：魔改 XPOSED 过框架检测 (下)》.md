> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/YAMCrQSi0LFJGNIwB9qHDA)

  

以上文章由作者【r0ysue】的连载有赏投稿，共有五篇，每周更新一篇；也欢迎广大朋友继续投稿，详情可点击 [OSRC 重金征集文稿！！！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247484531&idx=1&sn=6925d63e60984c8dccd4b0162dd32970&chksm=fa7b053fcd0c8c2938d1c5e0493a20ec55c2090ae43419c7aef933bcc077692b1997f4710afa&scene=21#wechat_redirect)了解~~  

温馨提示：建议投稿的朋友尽量用 markdown 格式，特别是包含大量代码的文章

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulK0U3qoFcpBd09CAAA7ibmoLqxDw2gHYXficMIDYlMHiclOO2ONdiawdf9g/640?wx_fmt=png)

本篇中我们继续前文的话题，对`Xposed`源码进行修改，过`XposedChecker`框架检测，并基于修改后的`API`进行插件项目二次开发。

本文所涉及的环境、实验数据结果及代码资料等都在作者的`Github`：https://github.com/r0ysue/AndroidSecurityStudy 上，欢迎大家取用和`star。`

### 魔改官方原版过检测

#### 修改`XposedInstaller`源码

对于`XposedInstaller`来说，要改的只有两个地方，一个是整体的包名，还有一个就是`.prop`配置文件。

先来改下整体的包名，首先将目录折叠给取消掉，否则无法重构包名。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulBNYaHnIS9GrZgQbfGWauHpE1lu7fEoHTVcttgdo64IfGyFvdOVv74w/640?wx_fmt=jpeg)

然后我们在包名路径中，将`xposed`改成`xppsed`，这样可以保证包名长度是一样，同时`xposed`特征消失不见，这就是总体的一个改名的策略。选择`Refactor`→`Rename`。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulLWQhOkIAC8UF52cPq4KPRWzedNzykD90hNuGvicAIdQRXkbjhibgRblw/640?wx_fmt=jpeg)

改前是`xposed`，改后是`xppsed`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulVSUvk3mYFRobNHm3JIwD8Y25GceAOO2NEgTz2GTZhz2vIDSPaZlibzQ/640?wx_fmt=png)

点击`Refactor`，会找到当前项目下所有包名中包含`xposed`的地方，点击`Do Refactor`才会真正的修改，进行重构，把所有地方的包名改成`xppsed`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulX0BIFqOuLBzLibbgt3EKZFhBJhNyjiak9ekbTpYnFTu4djoZ6icxSXA6g/640?wx_fmt=png)

可以随便打开某个代码文件看下，包名那里已经成为了

`de.robv.android.xppsed.installer.util`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIuloOj6iadl1iaaXAWWoPHP7gZQTqibTISzD8wiaZpn5lT8CRXtmKfgianOapg/640?wx_fmt=png)

接下来就是在整个项目的根文件夹下，进行整体的包名替换，因为还有很多编译配置、或者路径配置等等，需要进行包名的更换。

在`app`文件夹右击，选择`Replace in Path`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulpialniboofu9EwGFM28HjtwvBKps7KRcOk9pCVefEKribYNFiad9KBaLJg/640?wx_fmt=png)

把所有的`de.robv.android.xposed.installer`都改成`de.robv.android.xppsed.installer`。可以先搜一下，看看匹配的地方多不多。

其实搜出来匹配的地方只有在`5`个文件中的合计`7`处地方，并不多，直接`Replace All`替换掉即可。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulM94TFE0n7K14hYbZZMeWrSXNk3hcLMHtouRXvpawsw1LPCdy6iaX4fg/640?wx_fmt=png)

到这里第一处就改好了。第二处也非常简单，就是把如下图处的`xposed.prop`改成`xppsed.prop`即可。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIul8UiavqxKJ4f7S4RGsrKaZ5rb5kWypUC7ZYbwjzNAiaOBtXwPjKiaMar4Q/640?wx_fmt=png)

接下来就是编译了。编译时先`Build`→`Clean`一下，然后再`Build`→`Make Project`，这样就直接编译通过了。可以连接到手机，刷到手机上去，`App`会被装在手机上，但是无法自动启动，得手动点开。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIullkrhicrzoFwhUib38D0HrzyOhPib8S6w8sicnxvuPfAtib6deXU4bJl2Vrg/640?wx_fmt=png)

可以看到是没有问题的。

#### 修改`XposedBridge`源码及生成新`API`

对于`XposedBridge`修改的地方也是两个，一个是包名，一个是生成出来的`XposedBridge.jar`文件。

首先是改包名，方法与上文一模一样，也是首先将`xposed`进行重构，改成`xppsed`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulukiaVKMZSvQT3TwLmCcw9XHLfdD6Ma22icMVLerkOtJD8VOLv6qQ0yibw/640?wx_fmt=png)

改完之后随便打开几个文件看看，头顶的包名应该已经变成`xppsed`了。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulvSiad0L7SwXicLEZY8ovrRXgtI3kaEIXsybascmCMJmWdC3Mb0DysCzQ/640?wx_fmt=png)

然后也是一样的在项目根目录下，执行`Replace in Path`，将所有的`de.robv.android.xposed.installer`都改成`de.robv.android.xppsed.installer`。当然也可以先搜一下，其实匹配的也不是很多，大多数都在一个`txt`文件里。

最后就是编译，记得先`Make Clean`一下，然后编译，将编译出来的文件复制一份，命名为`XppsedBridge.jar`即可。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulkuJs5POAR26ZfHanPB5U2pibj3BZOuiaoyjnwAYbicRvuHcXFiapwVQV9A/640?wx_fmt=png)

编译`API`还是跟之前一样的流程，在操作上无需更改，会编译出来新的`api.jar`，后面会用来替换原来的`api.jar`。

有时候会怀疑，编译出来的文件是新生成的么？会不会是老文件？其实只要看下文件的生成 (或修改) 时间就可以，如果是当下的时间，那肯定是新生成的。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulmOotpVR25YaMry59HodgH3x8wnrDNY2dXtlFkAjOIwcmhQPJ8VFP3w/640?wx_fmt=png)

#### 修改`Xposed`框架源码去特征

在前文中，对于`Xposed`框架的编译是最轻松的，而且其实`Xposed`框架的代码文件并不是很多，但是改的地方倒还是不少。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulKM2ibwhXavtlMmibzG02ia6uTQUdx8VCW6qKXoiaKhLMRxciarIDrmhNiaTA/640?wx_fmt=png)

首先改下`libxposed_common.h`中的源码：

改之前

```
#define CLASS_XPOSED_BRIDGE "de/robv/android/xposed/XposedBridge"
#define CLASS_ZYGOTE_SERVICE "de/robv/android/xposed/services/ZygoteService"
#define CLASS_FILE_RESULT "de/robv/android/xposed/services/FileResult"

```

改之后

```
#define CLASS_XPOSED_BRIDGE "de/robv/android/xppsed/XposedBridge"
#define CLASS_ZYGOTE_SERVICE "de/robv/android/xppsed/services/ZygoteService"
#define CLASS_FILE_RESULT "de/robv/android/xppsed/services/FileResult"

```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulk2oXzGLsibq4vTXF6g7QYLL3ibbWrHE8sZkB4uW25TCsRmK8Viag3lcZQ/640?wx_fmt=png)

然后修改`Xposed.h`文件：

改之前

```
#define XPOSED_PROP_FILE "/system/xposed.prop"
#define XPOSED_LIB_ART XPOSED_LIB_DIR "libxposed_art.so"
#define XPOSED_JAR "/system/framework/XposedBridge.jar
#define XPOSED_CLASS_DOTS_ZYGOTE "de.robv.android.xposed.XposedBridge"
#define XPOSED_CLASS_DOTS_TOOLS "de.robv.android.xposed.XposedBridge$ToolEntryPoint"

```

改之后

```
#define XPOSED_PROP_FILE "/system/xppsed.prop"
#define XPOSED_LIB_ART XPOSED_LIB_DIR "libxppsed_art.so"
#define XPOSED_JAR "/system/framework/XppsedBridge.jar
#define XPOSED_CLASS_DOTS_ZYGOTE "de.robv.android.xppsed.XposedBridge"
#define XPOSED_CLASS_DOTS_TOOLS "de.robv.android.xppsed.XposedBridge$ToolEntryPoint"

```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulDXDVowDCdsQeh1EdmMibp0t5ecHdBX3fOwjvIJjdrsKibibhApl5wicUKw/640?wx_fmt=png)

然后在`xposed_service.cpp`之后，只有一处修改：

改之前

```
IMPLEMENT_META_INTERFACE(XposedService, "de.robv.android.xposed.IXposedService");

```

改之后

```
IMPLEMENT_META_INTERFACE(XposedService, "de.robv.android.xppsed.IXposedService");

```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulEU1BPWqHibJbVkYEdA0y8KyvPHUhRfHqSoTibTiajMBvyva2opGBCpMUA/640?wx_fmt=png)

接下来是`xposed_shared.h`。

改之前

```
#define XPOSED_DIR "/data/user_de/0/de.robv.android.xposed.installer/"
#define XPOSED_DIR "/data/data/de.robv.android.xposed.installer/"

```

改之后

```
#define XPOSED_DIR "/data/user_de/0/de.robv.android.xppsed.installer/"
#define XPOSED_DIR "/data/data/de.robv.android.xppsed.installer/"

```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIuleIrIOsgGfeXbib6bRWzTqzEicWjL4sohzeygKADtL51une57HU3nGibicA/640?wx_fmt=png)

接下来是`ART.mk`。

改之前

```
libxposed_art.cpp
LOCAL_MODULE := libxposed_art

```

改之后

```
libxppsed_art.cpp
LOCAL_MODULE := libxppsed_art

```

这个截图有些截不全了，分散得比较开，总之就这两个地方，没有别处。

最后将文件夹下的`libxposed_art.cpp`文件，重命名为`libxppsed_art.cpp`。

到这里`Xposed`框架源码修改完成。

#### 修改`XposedTools`编译工具源码

修改`XposedTools`主要的原则是，只要编译过程不报错就行，因为`XposedTools`中的代码并不会进入最终的编译包，它只是负责准备环境和将`Xposed`各个模块进行编译而已。

前面我们改生成文件的地方主要有三个，`xppsed.prop`、`XppsedBridge.jar`和`libxppsed_art`。

将`build.pl`和`zipstatic/_all/META-INF/com/google/android/flash-script.sh`这两个文件中的上述字符串，改成修改后的字符串即可，可以充分发挥查找替换的功能。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulbJKxxrX0IcD6lV3hHtibYiaicxPnGX0S9tKPiaN84QXt4Fv3YeaicOBloag/640?wx_fmt=png)

记得不要有遗漏，可以在修改完之后，到根目录下运行下述`grep`命令试试看，找不到相应的字符串即为全部替换完成。

```
# grep -ril "xposed.prop" *

```

在最终编译之前，记得把编译出来的`XppsedBridge.jar`放到`$AOSP/out/java/`目录中去噢，替换旧的`XposedBridge.jar`（删掉）。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulsuqP3MLMHE6yU3RJHghF74G7qJlzibuibwFjkEv5icSiaMibUgosAHLTCOw/640?wx_fmt=png)

然后运行编译的命令，开始编译即可。

```
# ./build.pl -t arm64:25

```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulFRJpe9IhyX4IwWiceQicBqWvXzwzTpia5XGvLQEOE3hvichQCA7cfibAldQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulic6N15HgNgnQ4GOXVhticN5ic82RbOWpuqXkCpBYWMrLdYg607gUkW4ZQ/640?wx_fmt=png)

编译的过程是非常快的，毕竟已经编译过一次了，  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulHRG8HNWOgkxshxcHK54N3znUNVVpcOD2nyATxJDkJnK2G0NBedERcQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulyWL2KTLE2lYrCpiaiay9DNkfUicpKLAYjwQlziaUzdicfuGd4qfp8wVwb4g/640?wx_fmt=png)

按照本篇的流程来，应该会一步成功，如果有报错，那就看下报错的地方在哪里，根据提示把漏看的、漏改的地方改掉即可。

### 魔改后的`API`插件开发

#### 刷机过`Xposed Checker`框架检

按照前文将新的`zip`刷入到手机中，可以发现已经成功了，主界面显示绿色，系统日志也显示已经按照将`XppsedBridge.jar`加载到`CLASSPATH`中了，`Xposed`框架基本正常运行。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulJPUXtMPdGgvdMqsC69fib21sY1PBLpx4MQ6ib6Moxcw22qpj8b0Y9cCA/640?wx_fmt=png)

然后可以上`Xposed Checker`框架检测来看下，可以发现`Xposed`框架检测的部分全部通过了，毕竟`Xposed`的特征已经变成`Xppsed`了。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulQVXFq4RXfl7icXoRicqz6D6v2HrtJIMzOvROwejcYscictQ155H9ick91w/640?wx_fmt=png)

最后的`root`检测其实只要给`su`文件换个名字就可以过，当然这是题外话了。

#### 市面上现有模块不再生效

如果给现在的手机装上`Gravity`这款`Xposed`模块的话，会出现 “系统框架未响应 GravityBox，正在退出” 的提示，模块无法正常工作，设置界面都进不去。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulSFvlia0phicMAgLt6rugAIpP28DeyBZylOR471FCRDyqjmkOT8v8xJNA/640?wx_fmt=png)

原因是在于`GravityBox`会在后台寻找`de.robv.android.xposed.IXposedHookZygoteInit`这个类，而这个类很明显包名被改掉了，报`ClassNotFoundError`。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIul2iaTQvhibTYAmI7obSjDm286WzDeObsV0Jcn1JN4tR5VGqAPxpj07sTw/640?wx_fmt=png)

所以说如果想要继续使用魔改后的`Xposed`框架，得自己基于编译出来的源码`API`，重新开发。

#### 基于新`API`的插件开发

前文中介绍了使用编译出来的`api.jar`进行开发的方法，这里主要也就是将新编译出来的`api.jar`替换原来的`api.jar`，然后会发现马上`xposed`包名相关的代码就变红了。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulnhWMXx2KfjIeTGEYdPxn7jQ8FeWcxl5peOGlEbYzVibsXHFROVom5ew/640?wx_fmt=png)

只要在`import`的地方，将包名处的`xposed`改成`xppsed`即可。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulkZaKX9Y25eAlwZcRDpestt2Ej0dR44GZtuSoBohmIMk5k7cQcPa1gA/640?wx_fmt=png)

接下来就是正常的编译和安装，重启之后模块即可生效，可以看到时钟后面加了`ID`，颜色也变了，同时也是`bypass`所有框架检测项目的。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulrwCEICFxMF1ynNjatf7tSWia96oPViaa30icKOsAUuGItvIicW1FlAw8NQ/640?wx_fmt=png)

基于新编译的`api.jar`二次开发成功。

#### 魔改后的项目源码和编译结果

最后稍微总结下，魔改`XPOSED`后的源码，修改后的工程源码见附件文件夹，合计五个工程，`android_art`没有改源码，用原来的就可以。

编译成果是三个`XposedApp2.zip`、`xposed-v89-sdk25-arm64.zip`、`XposedInstaller_3.1.5-debug.apk.zip`，也在附件文件夹中，不想编译源码的话可以直接使用。

整个项目的开发流程如下：

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulaQeRSs6sA8UnXpnPZ1jicq75TsZgViaVbJ93iaG7lsgk7ullSX66ONlnw/640?wx_fmt=png)

全部弄好有`134G`，最高标准压缩后也有`94G`，我也打包上传到我的百度云盘上了，大家可以下载来玩。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8LVp3gpj9g26BfianiayiaVIulj3bCP8icuydMyq1YibqqgY9T1b46QBQdd8vfTr1A35DJHL6mrjgxF0gw/640?wx_fmt=png)

### 小总结

最后来总结一下，在前文中介绍了官方原版`XPOSED`的编译流程，刷入手机、安装插件、开发插件都能正常使用；然后在其基础之上，本篇中进行源码的一些魔改，修改之后再编译刷入手机，最后讲根据新源码编译出来的`API`进行自己的插件开发流程，实现了从源码的**高维**来`bypass`**低维**框架检测的目的。

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)