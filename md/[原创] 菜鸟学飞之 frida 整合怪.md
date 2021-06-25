> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268219.htm)

> [原创] 菜鸟学飞之 frida 整合怪

学习历程
----

 

入行两年了，从刚开始的丝毫不懂。到现在的蹒跚学步，可谓是一步一个脚印走过来的。当然现在依然还是菜鸟一只。但是比起两年前茫然不知学习方向来说，现在有了明确前进的目标，只要不停下脚本，终有一天能飞上枝头成为合格的老鸟。下面整理了一些我的学习方式。希望能够帮到和曾经的我一样茫然的萌新。

### [](#1、阅读书籍)1、阅读书籍

最核心的基础大多是在书中有详细的介绍。看书一般不是只读一遍。我个人的方式是先大致的过一遍，每页都粗略的翻一遍，过完一遍就能大概知道哪些内容以自己目前的知识量，很难看明白，哪些是自己熟悉但是却理解不深的。然后第二遍才开始细翻，暂时跳过很难读懂的，先做个标记，以后知识面广了。再回过头来看。简单推荐几本《android 软件安全权威指南》、《android 应用安全防护和逆向分析》、《深入理解 android java 虚拟机 ART》、《深入理解计算机系统》

### [](#2、论坛)2、论坛

很多大佬都会发些实战的帖子。还有一些工具的使用。还有各种经验之谈。看书相当于是闭门造车，单靠自己的摸索的路是非常艰难的。站在巨人的肩膀上才能前进的更快。我最早先啥也不懂。就是看着大佬们的帖子渐渐入门。然后慢慢的才能看的懂书籍。

### [](#3、培训)3、培训

我经历过很多次培训，2011 年培训前我本职是修电脑的。在北京培训了 asp.net 然后入行做网站。2016 年我在广州培训 c++ 然后转行做游戏。2018 年自学后转行做安全，然后发现基础特烂，有些问题稍微变一点就很难自己解决。2020 年又报了看雪的 2w 和 3w 班。关于培训很多人有各种说法，只能说是智者见智，仁者见仁。不能以一概全。至少我觉得物有所值，如果你缺乏自制力，或者自学很久也不见成效。可以找合适的线上课程试试。

### [](#4、开源项目)4、开源项目

github 说它是公认最大的学习网站，应该不过分把？很多时候，伟大的前人，早就踩过了数不清的弯路，然后他们为我们铺平了道路。感谢开源精神，让知识遍地开花结果。当使用一些优秀的工具时，我们可以阅读源码，看看是如何实现的，熟悉代码后，完全根据自己的使用场景来修改 bug，或者做一些优化。如果对底层非常熟悉，甚至可以看穿作者的核心思路，知道核心实现的原理。

### [](#5、个人博客)5、个人博客

学习这么久，最吃亏的就是好多看过的东西，久久不用，基本都忘干净了。但是如果在当时，有详细的记录整个思路的话，翻一翻还是可以捡的回来。个人博客的存在我觉得其实很像是一个线上的笔记簿。可以随时在任意地方翻看。而且现在 markdown 的风格也很漂亮，我是用 hexo 搭建的个人博客，空间是 github 的免费空间。写笔记的时候记住一个要点，这个记录的第一目标客户是未来的自己。为了确保自己肯定看的懂，要尽量的详细。

### [](#6、练手和实战)6、练手和实战

学习了新的知识点后，一般会自己做个正向的 apk 练手。或者是拿别人的 crashMe 之类的来练手。当我们前置知识准备的差不多了，就可以拿一些自己常用的软件来进行练手。比如说某色流 app。或者是某小说软件去广告。不论最后能否成功达到目的，主要是在实战逆向中的一些见识，总结碰到的问题，最后如何解决的。如果解决不了，那又是因为什么因素。结果不重要，重要的是过程中的收获。

fridaUiTools 整合怪
----------------

> 整合工具开发，我觉得是一种比较高效的学习方式。将别人优秀的项目魔改，并且进行一定的拼合，整理成一个成套的工具。在整理的过程中，必然要熟悉对方的代码，并且对部分代码进行调整。在这个过程中，就能快速汲取到他人的经验。这种行为。就是所谓的整合怪 / 缝合怪了。

 

fridaUiTools 主要是把一些常用的 frida 的 hook 脚本简单统一输出方式后，整合进来。并且将自己觉得常用的功能做成界面调用的。并且在附加进程成功时获取一些信息默认的直接展示。后续会根据自己实战的经验。不断完善这个工具。

 

一直想做一个 frida 的脚本整理工具，有很多化腐朽为神奇的脚本由于常年不使用，自己都忘记了，觉得应该有一个工具把这些东西统一起来调用，因为常年使用 win 系统。导致我倾向于界面化的工具。我个人感觉。界面化的至少不用再记命令。操作也方便。然后我直接参考 ZenTracer，在他的原理上，重新对整体的流程以及界面和功能做的更加完善一些。

 

我对这个工具整体功能划分为三块。

### [](#1、js脚本的hook和管理（对批量多个脚本同时hook，可以自定义脚本进行管理，可以保存加载。必须是在附加前进行操作）)1、js 脚本的 hook 和管理（对批量多个脚本同时 hook，可以自定义脚本进行管理，可以保存加载。必须是在附加前进行操作）

> *   整合 r0capture
> *   整合 jnitrace
> *   整合 ZenTracer
> *   java 层的加解密相关自吐
> *   ssl 证书导出
> *   ssl pining（整合 DroidSSLUnpinning）
> *   模糊匹配函数进行批量 hook（整合 ZenTracer）
> *   模糊匹配 so 函数批量 hook（参数统一方式打印。所以输出只能做参考）
> *   native 的 sub 函数批量 hook（参数统一方式打印。所以输出只能做参考）
> *   stalker 的 trace
> *   脱壳相关（整合 frida_dump、FRIDA-DEXDump、fart）
> *   自定义脚本添加 （todo 待开发）
> *   patch 汇编代码 （todo 待开发）

### [](#2、常用功能的调用（常见的内存漫游操作进行功能化，以后再根据实战需求增加新功能。必须是在附加后进行操作）)2、常用功能的调用（常见的内存漫游操作进行功能化，以后再根据实战需求增加新功能。必须是在附加后进行操作）

> *   fart 主动调用
> *   DUMPDex 主动调用
> *   dump 打印指定地址
> *   dump 指定模块
> *   wallBreak 整合

### [](#3、初始化信息)3、初始化信息

> *   附加进程成功后，将一些常用的信息在界面展示，目前只处理了 module 列表和 class 列表。以后再根据需求增加新的信息

### [](#github：)github：

> [fridaUiTools](https://github.com/dqzg12300/fridaUiTools)

 

_**重点并不是开发整合工具的流程，而是学习并利用别人的项目的过程，所以下面主要是分析我整合用到的项目。**_

ZenTracer
---------

### [](#github：)github：

> **[ZenTracer](https://github.com/hluwa/ZenTracer)**

### [](#功能：)功能：

> 界面化的批量 hook 多个类，可以通过拉黑过滤掉一些调用率特别高的类

### [](#实现原理：)实现原理：

> 在 js 脚本中使用占位符 {MATCHREGEX} 和{BLACKREGEX}在后续替换，来传递参数需要批量 trace 的类名以及黑名单。遍历所有类匹配出符合要求的类名。并且不在黑名单中。则进行批量 hook。批量 hook 时会将函数所有重载都 hook 上。最后是三种输出方式，正常的 log 输出、函数进入时的参数输出、函数结束时的返回值输出。

### [](#核心代码（简略）：)核心代码（简略）：

```
//批量hook
function traceClass(clsname) {
    try {
        var target = Java.use(clsname);
          //获取本类所有函数（注意getMethods这个是获取本类和父类中的函数。）
        var methods = target.class.getDeclaredMethods();
        methods.forEach(function (method) {
            var methodName = method.getName();
              //获取所有重载
            var overloads = target[methodName].overloads;
            overloads.forEach(function (overload) {
                  //参数的类型
                var proto = "(";
                overload.argumentTypes.forEach(function (type) {
                    proto += type.className + ", ";
                });
                if (proto.length > 1) {
                    proto = proto.substr(0, proto.length - 2);
                }
                proto += ")";
                log("hooking: " + clsname + "." + methodName + proto);
                  //hook 函数
                overload.implementation = function () {
                    var args = [];
                    var tid = getTid();
                    var tName = getTName();
                    for (var j = 0; j < arguments.length; j++) {
                        args[j] = arguments[j] + ""
                    }
                      //函数进入时的参数啥的在里面通过send传给py
                    enter(tid, tName, clsname, methodName + proto, args);
                    var retval = this[methodName].apply(this, arguments);
                      //函数结束时的返回值在里面通过send传给py
                    exit(tid, "" + retval);
                    return retval;
                }
            });
        });
    } catch (e) {
        log("'" + clsname + "' hook fail: " + e)
    }
}
//匹配符合要求的类。并且不在黑名单中的
if (Java.available) {
    Java.perform(function () {
        log('ZenTracer Start...');
          //在js被读取时，会替换这里的数据
        var matchRegEx = {MATCHREGEX};
        var blackRegEx = {BLACKREGEX};
        Java.enumerateLoadedClasses({
            onMatch: function (aClass) {
                for (var index in matchRegEx) {
                    // console.log(matchRegEx[index]);
                    if (match(matchRegEx[index], aClass)) {
                        var is_black = false;
                        for (var i in blackRegEx) {
                            if (match(blackRegEx[i], aClass)) {
                                is_black = true;
                                log(aClass + "' black by '" + blackRegEx[i] + "'");
                                break;
                            }
                        }
                        if (is_black) {
                            break;
                        }
                        log(aClass + "' match by '" + matchRegEx[index] + "'");
                        traceClass(aClass);
                    }
                }
 
            },
            onComplete: function () {
                log("Complete.");
            }
        });
    });
}

```

### [](#改造并整合：)改造并整合：

> 优化界面显示，优化类名的输入环节。每次附加进程后。都将所有类名都保存下来。这里就可以选择之前缓存下来的所有类数据。然后根据输入智能过滤。可以快捷方便的找到自己想要 hook 的类。快捷添加操作。将经常要 hook 的类放在里面。就可以迅速的 hook 了。

### [](#相关贴图：)相关贴图：

![](https://bbs.pediy.com/upload/attach/202106/659397_455YX7TP9Q7A6MP.png)

r0capture
---------

### [](#github：)github：

> [r0capture](https://github.com/r0ysue/r0capture)

### [](#功能：)功能：

> 安卓应用抓包的通杀脚本。并且可以过证书校验解绑定 ssl pining。可以导出客户端 ssl 证书。可以导出 pcap 文件

### [](#实现原理：)实现原理：

> 通过分析 http https tcp udp ssl 的系统框架或者第三方框架的调用流程。找到底层调用的地方进行 hook。就可以在一定程度上通杀了。关于调用流程的分析的详细过程可以看看我之前整理的另一篇文章 [android 抓包学习的整理和归纳](https://bbs.pediy.com/thread-267940.htm)。
> 
> 证书的导出是选择一个证书会调用的时机，函数 getPrivateKey 和函数 getCertificateChain。hook 后。取出私钥和证书内容。重新设置密码导出新的证书。
> 
> 定位 sslpinning 证书绑定的位置，是通过 hook 类型 File 的构造函数，并且打印出调用堆栈。然后在里面匹配证书绑定函数是否在调用链。这是一种技巧。在其他场合同样可以使用类似的技巧来找到关键代码的位置。
> 
> 可以将抓包结果保存为 pcap 数据。便捷于一些擅长使用网卡抓包工具的人导入分析（例如 wireshark）。这个关键是需要熟悉网络数据包的组成结构，然后按照格式写入文件。可以参考他的这个顺便学习一下数据包的组成。

### [](#核心代码（简略，网络抓包相关的就不贴了，太多了）：)核心代码（简略，网络抓包相关的就不贴了，太多了）：

```
//导出证书到指定路径,并使用新密码
function storeP12(pri, p7, p12Path, p12Password) {
      var X509Certificate = Java.use("java.security.cert.X509Certificate")
      var p7X509 = Java.cast(p7, X509Certificate);
      var chain = Java.array("java.security.cert.X509Certificate", [p7X509])
      var ks = Java.use("java.security.KeyStore").getInstance("PKCS12", "BC");
      ks.load(null, null);
      ks.setKeyEntry("client", pri, Java.use('java.lang.String').$new(p12Password).toCharArray(), chain);
      try {
        var out = Java.use("java.io.FileOutputStream").$new(p12Path);
        ks.store(out, Java.use('java.lang.String').$new(p12Password).toCharArray())
      } catch (exp) {
        console.log(exp)
      }
    }
    //在服务器校验客户端的情形下，帮助dump客户端证书，并保存为p12的格式，证书密码为r0ysue
    Java.use("java.security.KeyStore$PrivateKeyEntry").getPrivateKey.implementation = function () {
      var result = this.getPrivateKey()
      var packageName = Java.use("android.app.ActivityThread").currentApplication().getApplicationContext().getPackageName();
      storeP12(this.getPrivateKey(), this.getCertificate(), '/sdcard/Download/' + packageName + uuid(10, 16) + '.p12', 'r0ysue');
      var message = {};
      message["function"] = "dumpClinetCertificate=>" + '/sdcard/Download/' + packageName + uuid(10, 16) + '.p12' + '   pwd: r0ysue';
      message["stack"] = Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new());
      var data = Memory.alloc(1);
      send(message, Memory.readByteArray(data, 1))
      return result;
    }
 
//SSLpinning helper 帮助定位证书绑定的关键代码
    Java.use("java.io.File").$init.overload('java.io.File', 'java.lang.String').implementation = function (file, cert) {
      var result = this.$init(file, cert)
      //打印堆栈
      var stack = Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new());
      //匹配证书绑定的函数是否在调用链中
      if (file.getPath().indexOf("cacert") >= 0 && stack.indexOf("X509TrustManagerExtensions.checkServerTrusted") >= 0) {
        var message = {};
        message["function"] = "SSLpinning position locator => " + file.getPath() + " " + cert;
        message["stack"] = stack;
        var data = Memory.alloc(1);
        send(message, Memory.readByteArray(data, 1))
      }
      return result;
    }

```

下面是保存到 pcap 的代码

```
def log_pcap(pcap_file, ssl_session_id, function, src_addr, src_port,
             dst_addr, dst_port, data):
    """Writes the captured data to a pcap file.
    Args:
      pcap_file: The opened pcap file.
      ssl_session_id: The SSL session ID for the communication.
      function: The function that was intercepted ("SSL_read" or "SSL_write").
      src_addr: The source address of the logged packet.
      src_port: The source port of the logged packet.
      dst_addr: The destination address of the logged packet.
      dst_port: The destination port of the logged packet.
      data: The decrypted packet data.
    """
    t = time.time()
    if ssl_session_id not in ssl_sessions:
        ssl_sessions[ssl_session_id] = (random.randint(0, 0xFFFFFFFF),
                                        random.randint(0, 0xFFFFFFFF))
    client_sent, server_sent = ssl_sessions[ssl_session_id]
    if function == "SSL_read":
        seq, ack = (server_sent, client_sent)
    else:
        seq, ack = (client_sent, server_sent)
    for writes in (
            # PCAP record (packet) header
            ("=I", int(t)),  # Timestamp seconds
            ("=I", int((t * 1000000) % 1000000)),  # Timestamp microseconds
            ("=I", 40 + len(data)),  # Number of octets saved
            ("=i", 40 + len(data)),  # Actual length of packet
            # IPv4 header
            (">B", 0x45),  # Version and Header Length
            (">B", 0),  # Type of Service
            (">H", 40 + len(data)),  # Total Length
            (">H", 0),  # Identification
            (">H", 0x4000),  # Flags and Fragment Offset
            (">B", 0xFF),  # Time to Live
            (">B", 6),  # Protocol
            (">H", 0),  # Header Checksum
            (">I", src_addr),  # Source Address
            (">I", dst_addr),  # Destination Address
            # TCP header
            (">H", src_port),  # Source Port
            (">H", dst_port),  # Destination Port
            (">I", seq),  # Sequence Number
            (">I", ack),  # Acknowledgment Number
            (">H", 0x5018),  # Header Length and Flags
            (">H", 0xFFFF),  # Window Size
            (">H", 0),  # Checksum
            (">H", 0)):  # Urgent Pointer
        pcap_file.write(struct.pack(writes[0], writes[1]))
    pcap_file.write(data)
    if function == "SSL_read":
        server_sent += len(data)
    else:
        client_sent += len(data)
    ssl_sessions[ssl_session_id] = (client_sent, server_sent)

```

### [](#改造并整合：)改造并整合：

> 我去掉了 pcap 的保存，直接调用脚本。把输出方式统一起来（去掉所有 js 的 console.log 打印。统一格式 send 到 py 进行输出）。其他功能都保持原有的。

jnitrace/JNI-Frida-Hook
-----------------------

### [](#github：)github：

> [jnitrace](https://github.com/chame1eon/jnitrace)/[JNI-Frida-Hook](https://github.com/Areizen/JNI-Frida-Hook)

### [](#功能：)功能：

> 对所有 jni 的函数进行 hook。比如 vmp 中大量使用到了 jni 的函数来模拟 java 的实现。对所有的 jni 进行 hook 就可以获得一些线索。
> 
> 这里我列了两个项目，是因为这两个我都分析了一下。jnitrace 的使用和输出都非常的方便，可以打印 jni 函数的结构，以及所有参数和返回值，并且代码结构优美，全部用 ts 实现的，可以说是非常完美的 hook 脚本开发模式，虽然很香，但是想要整合进来并不容易。我需要的是逻辑清晰易读的 js 文件来方便的嵌入，并且可以简单的修改。太过复杂庞大的 js 不利于我整合进来，所以最终选择了简单的 JNI-Frida-Hook，这个项目只是简单的 hook 了 jni 函数，打印了一下函数名，并没有详细的参数和返回值。我们可以后续再进行优化

### [](#实现原理：)实现原理：

> hook 的 js 开发比较麻烦的问题是多文件的调用会很难处理。所以 jnitrace 使用了 ts 写脚本再生成 js 来解决。
> 
> 而 JNI-Frida-Hook 直接使用的`require("./utils/jni_struct.js")`。然后再通过`frida-compile agent.js -o _agent.js`来将多个文件合并。
> 
> 这里我就只讲 JNI-Frida-Hook 的实现了。首先需要设置 hook 的目标模块 library_name 以及要监控的目标函数 function_name。
> 
> 这里他对 android_dlopen_ext 进行 hook。判断目标模块加载完成了，再进行目标函数的 hook。如果不这样做，在 spwan 的附加的时候，就会找不到模块，因为模块还未加载。
> 
> 遍历所有 export 符号，如果有找到设定的目标函数，就进行 hook 所有 jni 函数，并且在函数结束时，关掉所有 jni 函数的 hook。
> 
> 所有 jni 函数的 hook 实现就是准备所有 jni 函数的名称列表，然后遍历所有，然后 hook 的时候将 jnienv 的指针传进来，再根据 jni 函数名和 jnienv 的指针进行偏移，找到对应的函数地址。直接 hook 即可。最后 FindClass 可能比较特殊，就单独 hook 了。
> 
> 这个项目的关键就是计算偏移，这里只要熟悉类对象结构的存储，再看看 jnienv 这个类的结构，就看的很明白了，贴一篇我以前写的笔记博文：[类对象的内存布局](http://missking.cc/2020/08/12/ClassStructure/)

### [](#核心代码（简略）：)核心代码（简略）：

```
//在要hook的模块加载完后，才调用hook代码
Interceptor.attach(Module.findExportByName(null, 'android_dlopen_ext'),{
    onEnter: function(args){
        // first arg is the path to the library loaded
        var library_path = Memory.readCString(args[0])
                //判断当前加载的模块是否是目标模块
        if( library_path.includes(library_name)){
            console.log("[...] Loading library : " + library_path)
            library_loaded = 1
        }
    },
    onLeave: function(args){
 
        // if it's the library we want to hook, hooking it
        if(library_loaded ==  1){
            console.log("[+] Loaded")
              //hook目标函数
            hook_jni(library_name, function_name)
            library_loaded = 0
        }
    }
})
 
 
/*
Calculate the given funcName address from the JNIEnv pointer  //计算出jni函数的地址
*/
function getJNIFunctionAdress(jnienv_addr,func_name){
      //最关键的起始就是这里，根据jnienv的地址和函数名,计算出偏移，其实就是拿函数的当前索引。这个了解类对象的结构就很清楚了。
    var offset = jni_struct_array.indexOf(func_name) * Process.pointerSize
 
    // console.log("offset : 0x" + offset.toString(16))
 
    return Memory.readPointer(jnienv_addr.add(offset))
}
 
// Hook all function to have an overview of the function called     //hook全部jni函数
function hook_all(jnienv_addr){
    jni_struct_array.forEach(function(func_name){
        // Calculating the address of the function
        if(!func_name.includes("reserved"))
       {
            var func_addr = getJNIFunctionAdress(jnienv_addr,func_name)
            Interceptor.attach(func_addr,{
                onEnter: function(args){
                    console.log("[+] Entered : " + func_name)
                }
            })
        }
    })
}

```

### [](#改造并整合：)改造并整合：

> 他只针对了 spawn 的附加情况进行 hook。我调整了下，判断是哪种附加，再进行不同方式的调用。最后统一下输出的方式。

DroidSSLUnpinning
-----------------

### [](#github：)github：

> [DroidSSLUnpinning](https://github.com/WooyunDota/DroidSSLUnpinning)

### [](#功能：)功能：

> 主要是处理防抓包的双向验证的，客户端验证服务端的证书。这个项目可以解掉证书绑定，让中间人抓包正常运行。效果和 JustTrustMe 差不多。他厉害的地方在于支持各种库的解绑定。市面上大多数的绑定方式他都有处理到。下面列一下大佬的支持的库
> 
> ```
> 1.SSLcontext
> 2.okhttp
> 3.webview
> 4.XUtils
> 5.httpclientandroidlib
> 6.JSSE
> 7.network\_security\_config (android 7.0+)
> 8.Apache Http client (support partly)
> 9.OpenSSLSocketImpl
> 10.TrustKit
> 11.Cronet
> 
> ```

### [](#实现原理：)实现原理：

> 其实实现不难。但是关键是你要熟悉各种库的正向解绑定，知道是哪个函数来绑定的，然后将绑定函数给替换掉，直接改成空函数。所以像他这种支持这么多的，就比较厉害了。

### [](#核心代码（简略）：)核心代码（简略）：

```
    //代码太长。这里只简单放两种解绑定的例子
    //WebView的解绑定
    var WebViewClient = Java.use("android.webkit.WebViewClient");
    WebViewClient.onReceivedSslError.implementation = function(webView, sslErrorHandler, sslError) {
        quiet_send("WebViewClient onReceivedSslError invoke");
        //执行proceed方法
        sslErrorHandler.proceed();
        return;
    };
    WebViewClient.onReceivedError.overload('android.webkit.WebView', 'int', 'java.lang.String', 'java.lang.String').implementation = function(a, b, c, d) {
        quiet_send("WebViewClient onReceivedError invoked");
        return;
    };
    WebViewClient.onReceivedError.overload('android.webkit.WebView', 'android.webkit.WebResourceRequest', 'android.webkit.WebResourceError').implementation = function() {
        quiet_send("WebViewClient onReceivedError invoked");
        return;
    };
//okhttp的解绑定
var OkHttpClient = Java.use("com.squareup.okhttp.OkHttpClient");
OkHttpClient.setCertificatePinner.implementation = function(certificatePinner) {
  // do nothing
  quiet_send("OkHttpClient.setCertificatePinner Called!");
  return this;
};
 
// Invalidate the certificate pinnet checks (if "setCertificatePinner" was called before the previous invalidation)
var CertificatePinner = Java.use("com.squareup.okhttp.CertificatePinner");
CertificatePinner.check.overload('java.lang.String', '[Ljava.security.cert.Certificate;').implementation = function(p0, p1) {
  // do nothing
  quiet_send("okhttp Called! [Certificate]");
  return;
};
CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function(p0, p1) {
  // do nothing
  quiet_send("okhttp Called! [List]");
  return;
};

```

stalker
-------

### [](#github：)github：

> [sktrace](https://github.com/bmax121/sktrace)

### [](#功能：)功能：

> 这个项目主要是用 stalker 来实现 trace 汇编代码，打印每一句汇编指令执行后寄存器的变化。一般用于辅助分析算法还原。但是由于 frida 的 stalker 本身对 arm32 的支持不太好。所以这个项目目前还不支持 arm32。目前还未支持 spawn 附加。对于 c 的打印方式还未完善，没有打印出寄存器的具体数值

### [](#实现原理：)实现原理：

> 首先 CModule 声明了一段 c 的代码。然后 transform 设置使用 c 的函数。我想，他可能是为了方便打印数据。工作流程比较简单，就是设置了目标模块，设置了符号或地址（一般是函数开始的地址，会一直执行到这个函数完，所以不用设置终止位置），他设置了两种打印方式 stalkerTraceRangeC 和 stalkerTraceRange。用 c 的打印方式结果展示的比较好。但是缺少寄存器数值变化。另一种则是直接发送到 py。让 py 部分来处理结果。但是我看他 py 部分也是没有解析输出。自己动手改良了一下。

### [](#核心代码（简略）：)核心代码（简略）：

```
function traceAddr(addr) {
    let moduleMap = new ModuleMap();   
    let targetModule = moduleMap.find(addr);
    console.log(JSON.stringify(targetModule))
    let exports = targetModule.enumerateExports();
    let symbols = targetModule.enumerateSymbols();
      //先是hook要trace的位置
    Interceptor.attach(addr, {
        onEnter: function(args) {
            this.tid = Process.getCurrentThreadId()
              //这个trace方式是c打印的，下面那个是发送详细数据给py打印的。
            //stalkerTraceRangeC(this.tid, targetModule.base, targetModule.size)
            stalkerTraceRange(this.tid, targetModule.base, targetModule.size)
        },
        onLeave: function(ret) {
            Stalker.unfollow(this.tid);
            Stalker.garbageCollect()
            send({
                type: "fin",
                tid: this.tid
            })
        }
    })
}
//我准备使用发送到py详细数据的打印方式，不是很喜欢混c的语言来处理。感觉会容易出错
function stalkerTraceRange(tid, base, size) {
    Stalker.follow(tid, {
        transform: (iterator) => {
            const instruction = iterator.next();
            const startAddress = instruction.address;
            const isModuleCode = startAddress.compare(base) >= 0 &&
                startAddress.compare(base.add(size)) < 0;
            // const isModuleCode = true;
              //transform是每个block触发。这里每个block触发的时候遍历出所有指令。
            do {
                iterator.keep();
                if (isModuleCode) {
                      //这里可以看到数据如果是inst就是一个指令，我们就需要解析打印
                      //输出样本如下
                      //'payload': {'type': 'inst', 'tid': 19019, 'block': '0x74fd8d4ff4', 'val': '{"address":"0x74fd8d4ffc","next":"0x4","size":4,"mnemonic":"add","opStr":"sp, sp, #0x70","operands":[{"type":"reg","value":"sp"},{"type":"reg","value":"sp"},{"type":"imm","value":"112"}],"regsRead":[],"regsWritten":[],"groups":[]}'}}
                      //py解析打印格式"add sp, sp, #0x70  //sp=112"        这里的处理应该还要更复杂。暂时先简单处理
 
                    send({
                        type: 'inst',
                        tid: tid,
                        block: startAddress,
                        val: JSON.stringify(instruction)
                    })
                         //这里是打印所有寄存器
                      //输出样本如下
                      //{'type': 'ctx', 'tid': 19019, 'val': '{"pc":"0x74fd8d4fe8","sp":"0x7fc28609d0","x0":"0x0","x1":"0x7fc2860908","x2":"0x0","x3":"0x756aec1349","x4":"0x7fc28608f0","x5":"0x14059dbe","x6":"0x7266206f6c6c6548","x7":"0x2b2b43206d6f7266","x8":"0x0","x9":"0x65af2e18847fd289","x10":"0x1","x11":"0x7fc2860a20","x12":"0xe","x13":"0x7fc2860a20","x14":"0xffffff0000000000","x15":"0x756aeed1b5","x16":"0x74fd8fadc8","x17":"0x74fd8d50d8","x18":"0x75f0bda000","x19":"0x75f02f9c00","x20":"0x756af59490","x21":"0x75f02f9c00","x22":"0x7fc2860c90","x23":"0x74ffcee337","x24":"0x4","x25":"0x75f04b4020","x26":"0x75f02f9cb0","x27":"0x1","x28":"0x756b3f2000","fp":"0x7fc2860a30","lr":"0x74fd8d4fdc"}'}}
                      //这里是寄存器变化时调用
                    iterator.putCallout((context) => {
                            send({
                                type: 'ctx',
                                tid: tid,
                                val: JSON.stringify(context)
                            })
                    })
                }
            } while (iterator.next() !== null);
        }
    })
}

```

### [](#改造并整合：)改造并整合：

> 优化 py 结果打印，增加 spawn 支持。

frida_hook_libart
-----------------

### [](#github：)github：

> [frida_hook_libart](https://github.com/lasting-yang/frida_hook_libart)

### [](#功能：)功能：

> （ps：yang 大神的三件套。向大佬学习。给大佬递茶。）
> 
> hook_RegisterNatives.js，hook 打印动态注册的函数，分析 so 时静态注册的函数我们一般直接搜索 Java 开头的符号名就基本都是了，但是动态注册的我们静态分析没法找到对应的 native 函数。
> 
> hook_artmethod.js，java 的函数打印最终都是调用的 ArtMethod 的 Invoke。对这里进行 hook。就可以打印所有 java 函数的调用了。
> 
> hook_art.js，hook art 中的 jni 函数并且有打印参数和返回值，在 aosp10 上面测试了一下。一个都没 hook 成功。发现是判断_ZN3art3JNIILb0 的问题。用 ida 打开 libart.so。然后搜索一个里面想要 hook 的函数 GetStringUTFChars。找到他的符号名是`_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc`。所以修改下过滤的判断。改成`_ZN3art3JNI`。然后正常输出结果。不过这个打印数据相当之多。另外这个也将上面的 hook_RegisterNative.js 的部分给包含了。
> 
> 另外在测试的时候发现。hook_artmethod.js 和 hook_art.js 里面用到的 class StdString 好像在 frida12 的版本会出错。升到 frida14 就正常了。

### [](#实现原理：)实现原理：

> hook_RegisterNatives.js 遍历 libart.so 的所有符号，找到 RegisterNative 函数的地址。然后 hook 打印
> 
> hook_artmethod.js 遍历 libart.so 的所有符号，找到_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc 符号的地址，也就是 ArtMethod 的 Invoke，然后 hook 了打印堆栈和函数名
> 
> hook_art.js 遍历 libart.so 的所有符号，找到一些常用的 jni 函数，取出函数地址。然后 hook 函数，用对应的方式打印

### [](#核心代码（简略）：)核心代码（简略）：

```
//hook_RegisterNative.js的部分，就是这段。找RegisterNative函数的地址。
var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrRegisterNatives = null;
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
 
        //_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi
        if (symbol.name.indexOf("art") >= 0 &&
                symbol.name.indexOf("JNI") >= 0 &&
                symbol.name.indexOf("RegisterNatives") >= 0 &&
                symbol.name.indexOf("CheckJNI") < 0) {
            addrRegisterNatives = symbol.address;
            log("RegisterNatives is at "+symbol.address+" "+symbol.name);
        }
    }
 
//hook_artmethod.js的部分
//这里是遍历所有符号，匹配出ArtMethod的Invoke
var module_libart = Process.findModuleByName("libart.so");
    var symbols = module_libart.enumerateSymbols();
    var ArtMethod_Invoke = null;
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        var address = symbol.address;
        var name = symbol.name;
        var indexArtMethod = name.indexOf("ArtMethod");
        var indexInvoke = name.indexOf("Invoke");
        var indexThread = name.indexOf("Thread");
        if (indexArtMethod >= 0
            && indexInvoke >= 0
            && indexThread >= 0
            && indexArtMethod < indexInvoke
            && indexInvoke < indexThread) {
              //将后面的hook代码去掉。可以看到这里最终匹配到的结果是_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc
              //转换下格式之后的结果是art::ArtMethod::Invoke(art::Thread*, unsigned int*, unsigned int, art::JValue*, char const*)
            console.log(name);
            ArtMethod_Invoke = address;
        }
    }
        //如果上面匹配到了Invoke函数后。就hook打印。
    if (ArtMethod_Invoke) {
        Interceptor.attach(ArtMethod_Invoke, {
            onEnter: function (args) {
                var method_name = prettyMethod(args[0], 0);
                if (!(method_name.indexOf("java.") == 0 || method_name.indexOf("android.") == 0)) {
                    console.log("ArtMethod Invoke:" + method_name + '  called from:\n' +
                        Thread.backtrace(this.context, Backtracer.ACCURATE)
                            .map(DebugSymbol.fromAddress).join('\n') + '\n');
                }
            }
        });
    }
//这里也是个重点。打印当前函数名的方式。
function prettyMethod(method_id, withSignature) {
    const result = new StdString();
    Java.api['art::ArtMethod::PrettyMethod'](result, method_id, withSignature ? 1 : 0);
    return result.disposeToString();
}
 
//hook_art.js的重点部分
//这个就不放了。一整块有点大。简单说下，就是遍历libart.so所有符号列表找到一些常用的jni函数地址。然后打印输出

```

### [](#改造并整合：)改造并整合：

> 没啥好改的。调整下日志打印方式。直接淦就完了。
> 
> （ps：另外说一下。虽然这个 hook_art.js 也是对 jni 的 hook。但是和我之前封装的 Jni-Frida-Hook 是有一定区别的。这个是直接 hook 系统底层的。所有触发都会调用。而那个是指定某个函数触发时，hook 所有 jni 函数。然后函数结束后，清掉所有 hook。用哪种就看自己的需求拉。）

frida_dump
----------

### [](#github：)github：

> [frida_dump](https://github.com/lasting-yang/frida_dump)

### [](#功能：)功能：

> dump_module.js 从内存中 dump so 模块保存到文件（以前叫 dump_so.js）。有时候用 unicorn 模拟执行片段指令的时候，直接使用 apk 中的 so 是不行的。因为缺少上下文数据，如果有对外部数据的使用，就无法正常执行。但是从内存中 dump 出来的 so 是在执行过程中的，所以自带了上下文数据。
> 
> dump_dex.js 就是脱壳，在 DefineClass 函数调用的时机进行 dump dex 保存到文件
> 
> dump_dex_class.js 和上面的差不多，多了个步骤 load_all_class，这个函数主要是用来遍历所有 classloader。加载所有类的。

### [](#实现原理：)实现原理：

> 前面两个比较简单，就只说 dump_dex_class.js 了。
> 
> 首先是找到 DefineClass 的函数地址。然后从参数中取出 dexFile。根据 dexFile 的结构，偏移指针的距离得到 begin 的值和大小。那么就可以把这个 dexfile 保存出来了。可以先把所有 classloader 里面的所有 dexfile 的 class 全部都加载了一遍，最后在保存的。下面是遍历 loadClass 的流程
> 
> 遍历所有 classloader。然后转换成 BaseDexClassLoader，获取到 DexPathList，获取到 dexElements，再遍历所有的 dexfile，最后通过 _entries_ 枚举所有类名，最后 loader.loadClass 来加载这个类。可能是有的壳 loadClass 之后才会生成解密的 dex。所以就先全部 loadClass 一遍，然后再进行 dump

### [](#核心代码：)核心代码：

> （不贴了，文章太长了。感兴趣的大家自己翻翻看吧）

### [](#改造并整合：)改造并整合：

> 测试了下 libart.so 不需要 spawn 判断。都可以获取到。所以去掉 spawn 判断部分。然后在 dex 保存的时候，碰到了权限问题。不知道是不是和安卓 10 有关。总之修改成 py 来负责创建目录，赋值权限。增加功能从手机直接把脱壳好的文件下载到项目内。dump_dex_class 的 dump_dex 抽成功能单独调用

### FRIDA-DEXDump

### [](#github：)github：

> [FRIDA-DEXDump](https://github.com/hluwa/FRIDA-DEXDump)

### [](#功能：)功能：

> 也是脱壳，不过和上面的方式不一样，是在内存中检索 dex 的特征的，再 dump 出来进行脱壳。同时支持 objection

### [](#实现原理：)实现原理：

> 首先是设置了三个 rpc 功能。然后 py 根据这三个功能去根据 dex 特征检索内存，找到后验证数据，然后读取这段数据出来，再保存到文件。
> 
> 1、scandex：枚举内存中所有只读的数据，检索只读数据中所有出现`64 65 78 0a 30 ?? ?? 00`数据的段。这个是 dex 的二进制数据的头部特征。如下图。版本有的是 035，有的是 037. 所以他把版本部分没有匹配。匹配到结果后，就开始验证这段数据是不是一个正常的 dex。验证通过后就返回这段数据的地址和大小。
> 
> ![](https://bbs.pediy.com/upload/attach/202106/659397_YF3BYDSY5RHPTKT.png)
> 
> 2、memorydump：用来读取指定地址，指定大小的数据，并返回。代码很简单。
> 
> 3、switchmode：设置是否深度搜索。这里是有一个比较模糊的特征`70 00 00 00`来进行搜索。上面的图也有一个正常 dex 中的该特征。

### [](#核心代码：)核心代码：

> （不贴了，文章太长了。感兴趣的大家自己翻翻看吧）

### [](#改造并整合：)改造并整合：

> 统一下日志输出

fart
----

### [](#github：)github：

> [FART](https://github.com/hanbinglengyue/FART)

### [](#功能：)功能：

> 也是脱壳用的。基于主动调用的脱壳，可以过掉大多数函数抽取壳。这是 fart 的 frida 版本。免去了编译 rom。

### [](#实现原理：)实现原理：

> fart 的原理比较长，详细的可以直接看作者的详细文章
> 
> 1、[FART：ART 环境下基于主动调用的自动化脱壳方案](https://bbs.pediy.com/thread-252630.htm)
> 
> 2、[FART 正餐前甜点：ART 下几个通用简单高效的 dump 内存中 dex 方法](https://bbs.pediy.com/thread-254028.htm)
> 
> 3、[拨云见日：安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点](https://bbs.pediy.com/thread-254555.htm)
> 
> 另外有我以前自己整理的一片文章
> 
> [fart 的理解和分析过程](https://bbs.pediy.com/thread-263401.htm)

### [](#改造并整理：)改造并整理：

> 将 fart 主动调用和 dumpclass 主动调用设置到 rpc 中。在功能里面来调用触发。增加将 libart.so 快捷 push 到手机并 chmod 权限的功能，另外测试发现 LoadMethod 地址的获取处，在安卓 10 无法获取到函数地址。修改成支持安卓 10 的。代码如下
> 
> ```
> var versionData="ClassDataItemIterator";
> if(Java.androidVersion=="10"){
>   versionData="ClassAccessor";
> }
> var symbols = Module.enumerateSymbolsSync("libart.so");
> for (var i = 0; i < symbols.length; i++) {
>   var symbol = symbols[i];
>  //_ZN3art11ClassLinker10LoadMethodERKNS_7DexFileERKNS_21ClassDataItemIteratorENS_6HandleINS_6mirror5ClassEEEPNS_9ArtMethodE
>   if (symbol.name.indexOf("ClassLinker") >= 0
>       && symbol.name.indexOf("LoadMethod") >= 0
>       && symbol.name.indexOf("DexFile") >= 0
>       && symbol.name.indexOf(versionData) >= 0
>       && symbol.name.indexOf("ArtMethod") >= 0) {
>     addrLoadMethod = symbol.address;
>     break;
>   }
> }
> 
> ```

Wallbreaker
-----------

### [](#github：)github：

> [Wallbreaker](https://github.com/hluwa/Wallbreaker)

### [](#功能：)功能：

> 主要是内存漫游，搜索内存中的 java 类和对象，并且可以打印类的结构体，以及对象的数据。

### [](#实现原理：)实现原理：

> 首先找到几个关键的文件如下。
> 
> `Wallbreaker/__init__.py`功能调用的入口，四个功能 classsearch、classdump、objectsearch、objectdump
> 
> `Wallbreaker/wallbreaker/agent/command/__init__.py`功能实现的关键代码，这里使用 rpc 调用 js 的函数来获取相关数据出来加工处理。
> 
> `Wallbreaker/agent/_agent.js`核心的 js。这里提供了一系列查询内存的 rpc 接口。searchHandles、getRealClassNameByHandle、getObjectFieldValue、instanceOf、mapDump、collectionDump。
> 
> 这个项目主要是使用 rpc 交互达到 py 调用 js 访问 frida 函数并封装各种便利功能。从这个项目延伸的话，我们用这个模式可以打造各种强大的 frida 交互工具。

### [](#改造并整理：)改造并整理：

> 结果输出方式调整。rpc.exports 修改初始化方式，以免覆盖到其他 js 的 rpc 函数。将需要使用的所有脚本添加后，最后默认追加这个脚本。由于我默认做了类列表和过滤功能，所以 classsearch 可能有点鸡肋。

### [](#相关贴图：)相关贴图：

![](https://bbs.pediy.com/upload/attach/202106/659397_QE3PS4WBJGJDPQN.png)

 

![](https://bbs.pediy.com/upload/attach/202106/659397_M9GJC48GUW28275.png)

 

**最后整理完成，目前功能还不是很多，而且 bug 估计还挺多的。希望大佬们多多指点，有什么比较好的想法也可以给点建议。**

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 3 小时前 被 misskings 编辑 ，原因： 修改细节