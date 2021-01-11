> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265160.htm)

Frida 是非常灵活的 Hook 框架，支持多平台。  
在这里就不过多介绍了，详细可以参看官网：https://frida.re/  
使用 Frida 也挺长时间了，结合平时实战的经验，系统的梳理了一下开发环境、调试环境、整合的常用脚本和新特性分享给大家，在这里抛个砖。

TypeScript 开发环境
===============

一开始使用 frida 时，直接写个 js 脚本然后注入就完事了，优点是快捷方便，缺点也很明显就是 hook 代码写多了都挤在一个 js 里冗长，事后阅读起来比较费事，也不利于模块化和代码复用。  
TypeScript + npm 的方式搭建 frida 开发环境，使得在用 IDE 编写 frida 时可以有代码补全提示同时 TypeScript 又能使开发更容易达到模块化，代码复用更方便。

优点
--

1.  JavaScript 的一个超集，扩展了 JavaScript 的语法。
2.  加强代码可读性。明确参数类型，代码语义更清晰易懂。
3.  更友好、更精准的代码补全提示。
4.  更贴近面向对象编程的编写习惯，利于模块化和复用。
5.  以前用 js 写的脚本也可以被 ts 直接引用，不会浪费。

环境搭建
----

Frida 使用 TypeScript， 环境的搭建很简单。

1.  github 直接下载老奥的示例仓库: [https://github.com/oleavr/frida-agent-example](https://github.com/oleavr/frida-agent-example)
2.  按说明构建环境
    
    ```
    $ git clone git://github.com/oleavr/frida-agent-example.git
    $ cd frida-agent-example/
    $ npm install
    
    ```
    
    实际上就是利用 `frida-compile` 编译脚本 `frida-compile agent/index.ts -o _agent.js -c`。

用 pycharm 打开构建好环境的工程，编写 frida ts 脚本已经可以看到代码补全提示了。

 

![](https://bbs.pediy.com/upload/attach/202101/357319_N2PZRZAB2FBZ4Q2.png)

1.  启用实时编译

```
$ npm run watch
# 或者
$ frida-compile agent/index.ts -o _agent.js -w

```

使用
--

将上面生成的目标文件 `_agent.js` 注入到目标进程即可。

```
$ frida -U -f com.example.android --no-pause -l _agent.js

```

JS 单步调试
=======

能愉快的单步调试 frida 的 js 脚本，可以方便不少。

 

首先运行 frida 脚本

```
frida -l  --debug --runtime=v8 
```

或者

```
session = dev.attach(app.pid)
script = session.create_script(jscode, runtime="v8")
session.enable_debugger()

```

启动后会回显 Inspector 正在监听 9229 默认端口

```
Chrome Inspector server listening on port 9229

```

chrome
------

打开 chrome://inspect, 点击 `Open dedicated DevTools for Node`

 

![](https://bbs.pediy.com/upload/attach/202101/357319_VKHM4SGJ33S3PSF.png)

 

此时 debug 已经连接，切换至 `Sources`，按 `Command + P` 加载要调试的脚本，即可下断调试了。

 

![](https://bbs.pediy.com/upload/attach/202101/357319_Y95T7MA2673CFE5.png)

pycharm
-------

1.  首先安装 Node.js 插件，重启
2.  添加调试器 `Attaching to Node.js/Chrome`，端口默认即可。`Attach to` 应选择 `Node.js < 8 started with --debug`， 下面的自动重连选项可选可不选。
    
    ![](https://bbs.pediy.com/upload/attach/202101/357319_QCJ38RVTYS8YXXY.png)
    
3.  触发断点需要在 debug 窗口切换到 script 选项卡，右键要调试的脚本，选择 Open Actual Source，在新打开的 Actual Source 窗口设置好断点后，需要再取消 / 启用一次所有断点作为激活，发现断点上打上对勾才真正可用了。
    
    ![](https://bbs.pediy.com/upload/attach/202101/357319_DZRDXKCCTGAAJEU.png)
    
4.  接下来就可以愉快的调试了
    
    ![](https://bbs.pediy.com/upload/attach/202101/357319_5393W6WQJQ9QVP5.png)
    

优缺点
---

1.  用 Chrome 调试支持的更为顺滑，调试脚本自动重加载，断点也能正确响应。
2.  用 PyCharm 调试断点有时需要手动激活有点麻烦，纯粹是个人偏爱 PyCharm 的 Debug 窗口和快捷键。
3.  PyCharm 使用 ts 环境，调试时可以直接在 ts 文件上下断，也不需要手动激活断点，比较顺畅。

FridaContainer 脚本集分享
====================

FridaContainer 整合了网上流行的和平时常用的 frida 脚本，为逆向工作提效之用。  
该仓库已经申请了开源，地址：[https://github.com/deathmemory/FridaContainer](https://github.com/deathmemory/FridaContainer)

反反调试 anti-anti-debug
--------------------

整理了一些普通反调试的绕过或定位的方法。因为很多反调试的手段是通过读取各个状态文件，查找特征字符串来判断是否被调试，而读取文件内容又通常都用到了 `fgets` 函数，如此我们就直接 hook 此函数加入过滤规则就能过滤掉许多反调试检测。

 

例如：  
`/proc/<pid>/status` 检测 `TracerPid: 0` 、 `State: S (sleeping)` 、 `SigBlk: 0000000000001204` ，  
`/proc/<pid>/stat` 检测 `t (tracing stop)`  
`/proc/<pid>/wchan` 检测 `SyS_epoll_wait` 等

 

Hook fgets 后，触发时首先获取一下 LR 寄存器的值，保存一下现场，之后返回信息时可以把 LR 带出来，方便定位。然后调用原函数，对赋值的 buffer 进行过滤就可以了。

```
static anti_fgets() {
  const tag = 'anti_fgets';
  const fgetsPtr = Module.findExportByName(null, 'fgets');
  DMLog.i(Anti.tag, 'fgets addr: ' + fgetsPtr);
  if (null == fgetsPtr) {
    return;
  }
  var fgets = new NativeFunction(fgetsPtr, 'pointer', ['pointer', 'int', 'pointer']);
  Interceptor.replace(fgetsPtr, new NativeCallback(function (buffer, size, fp) {
    if (null == this) {
      return 0;
    }
    var logTag = null;
    // 进入时先记录现场
    const lr = FCCommon.getLR(this.context);
    // 读取原 buffer
    var retval = fgets(buffer, size, fp);
    var bufstr = (buffer as NativePointer).readCString();
 
    if (null != bufstr) {
      if (bufstr.indexOf("TracerPid:") > -1) {
        buffer.writeUtf8String("TracerPid:\t0");
        logTag = 'TracerPid';
      }
      //State:    S (sleeping)
      else if (bufstr.indexOf("State:\tt (tracing stop)") > -1) {
        buffer.writeUtf8String("State:\tS (sleeping)");
        logTag = 'State';
      }
      // ptrace_stop
      else if (bufstr.indexOf("ptrace_stop") > -1) {
        buffer.writeUtf8String("sys_epoll_wait");
        logTag = 'ptrace_stop';
      }
      else if (bufstr.indexOf(") t") > -1) {
        buffer.writeUtf8String(bufstr.replace(") t", ") S"));
        logTag = 'stat_t';
      }
 
      // SigBlk
      else if (bufstr.indexOf('SigBlk:') > -1) {
        buffer.writeUtf8String('SigBlk:\t0000000000001204');
        logTag = 'SigBlk';
      }
      if (logTag) {
        DMLog.i(tag + " " + logTag, bufstr + " -> " + buffer.readCString() + ' lr: ' + lr
                + "(" + FCCommon.getModuleByAddr(lr) + ")");
      }
    }
    return retval;
  }, 'pointer', ['pointer', 'int', 'pointer']));
}

```

**FridaContainer 调用：**

```
FCAnd.anti.anti_debug();

```

**小结：**

 

当上面的方法无法绕过反调试时，可以再 Hook 一些常用的退出函数来定位反调点，比如 hook `exit`, `kill` ，再总结出一些其他过反调的方法，思路类似。

anti-ssl-pinning
----------------

我们也在 FridaContainer 里面集成了 Frida 版的 JustTrustMe 来过 SSL Pinning 检测。

 

此部分代码主要借鉴了：https://codeshare.frida.re/@akabe1/frida-multiple-unpinning/

 

支持 20 种类库的 SSL 验证绕过：

*   TrustManager (Android < 7)
*   TrustManagerImpl (Android> 7)
*   OkHTTPv3 (quadruple bypass)
*   Trustkit (triple bypass)
*   Appcelerator Titanium
*   OpenSSLSocketImpl Conscrypt
*   OpenSSLEngineSocketImpl Conscrypt
*   OpenSSLSocketImpl Apache Harmony
*   PhoneGap sslCertificateChecker
*   IBM MobileFirst pinTrustedCertificatePublicKey (double bypass)
*   IBM WorkLight (ancestor of MobileFirst) HostNameVerifierWithCertificatePinning (quadruple bypass)
*   Conscrypt CertPinManager
*   CWAC-Netsecurity (unofficial back-port pinner for Android<4.2) CertPinManager
*   Worklight Androidgap WLCertificatePinningPlugin
*   Netty FingerprintTrustManagerFactory
*   Squareup CertificatePinner [OkHTTP<v3] (double bypass)
*   Squareup OkHostnameVerifier [OkHTTP v3] (double bypass)
*   Android WebViewClient (double bypass)
*   Apache Cordova WebViewClient
*   Boye AbstractVerifier

**FridaContainer 调用：**

```
FCAnd.anti.anti_ssl_unpinning();

```

**小结：**

 

主要的出发点就是想摆脱 Xposed 框架，调试时减少被检测的风险，要绕过检测只绕 Frida 一个就好了。

dump dex
--------

这里的 dump dex 是 dump 二代壳，通过 hook art so 的 `art::DexFile::OpenCommon` (`_ZN3art7DexFile10OpenCommonEPKhjRKNSt3__112basic_stringIcNS3_11char_traitsIcEENS3_9allocatorIcEEEEjPKNS_10OatDexFileEbbPS9_PNS0_12VerifyResultE`) dump 动态加载的 dex 文件。

 

通过 frida 动态加载自定义的 dex，在 dex 中实现类的枚举和主动加载，来尝试加载所有的类，使其触发 dex 动态加载，再通过之前的 hook ，将加载的 dex dump 下来。

 

**FridaContainer 调用：**

```
FCAnd.dump_dex_common();

```

也有通过搜索内存的方式将 dex dump 出来 [FRIDA-DEXDump](https://github.com/hluwa/FRIDA-DEXDump)

multi-dex
---------

有时候遇到动态加载的 dex 又不是用的 application 的 loader ，`Java.use` 可能会提示找不到类，遇到这种情况可以用修改 java loader 的方式来应对。  
比如遇到利用 InMemoryDexClassLoader 来加载内存中的 dex 时，可以 Hook 它，当其触发时先走原流程，让其动态加载 dex ，然后利用此时的 loader object 修改 `Java.classFactory.loader`，再使用 `Java.use` 来获取类就可以了，使用后记得恢复现场，否则会崩溃。

 

**动态加载实例：**

 

![](https://bbs.pediy.com/upload/attach/202101/357319_EBAB3A4EXY7XAUT.png)

 

**完整代码：**

```
function anti_InMemoryDexClassLoader(callbackfunc) {
    //  dalvik.system.InMemoryDexClassLoader
    const InMemoryDexClassLoader = Java.use('dalvik.system.InMemoryDexClassLoader');
    InMemoryDexClassLoader.$init.overload('java.nio.ByteBuffer', 'java.lang.ClassLoader')
        .implementation = function (buff, loader) {
        this.$init(buff, loader);
        var oldcl = Java.classFactory.loader;
        Java.classFactory.loader = this;
        callbackfunc();
        Java.classFactory.loader = oldcl; // 恢复现场
    }
}

```

**使用：**

```
FCAnd.anti.anti_InMemoryDexClassLoader(function(){
    const cls = Java.use('find/same/multi/dex/class');
    ...
});

```

frida 的 java 反射调用
-----------------

有时 dex 使用了一些非常规的动态加载方式，找 loader 定位起来也比较麻烦。假设我们只想调用某 jni 函数，看输入输出值的话，还可以用 frida java 反射的方式调用。

 

**整体调用流程：**

```
// 获取 so 基址
var base = Module.findBaseAddress('libxxxx.so');
// 根据偏移获取 jni 函数地址
var jnifunc_ptr = libsgmainso.add(0xE729);
// 声明 jni 函数
var jnifunc = new NativeFunction(jnifunc_ptr, 'pointer', ['pointer', 'pointer', 'int', 'pointer']);
// ********* 拼装 obj *********
// JNIEnv
var env = Java.vm.getEnv();
// 调用
var retval = jnifunc(env.handle,  ptr(0),  10401,  obj);

```

**拼装 Obj：**

```
// staticVaMethod 实现 Integer.valueOf(7)
const Integer_jcls = env.findClass('java/lang/Integer');
const Integer_valueOf = env.getStaticMethodId(Integer_jcls, 'valueOf', '(I)Ljava/lang/Integer;');
const invorkeStaticOjbectMethod = env.staticVaMethod('pointer', ['int']);
var pIn2 = invorkeStaticOjbectMethod(env.handle, Integer_jcls, Integer_valueOf, 7);
// 利用 constructor | vaMethod 组装 HashMap obj
// new HashMap().put('INPUT', 'xxxxxxxxxx')
const HashMap_jcls = env.findClass('java/util/HashMap');
const invokeHashmap_constructor = env.constructor([]);
const HashMap_init = env.getMethodId(HashMap_jcls, '', '()V');
var HashMap_obj = invokeHashmap_constructor(env.handle, HashMap_jcls, HashMap_init);
const HashMap_put = env.getMethodId(HashMap_jcls, 'put', '(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;');
const invokeOjbectMethod = env.vaMethod('pointer', ['pointer', 'pointer']);
invokeOjbectMethod(env.handle, HashMap_obj, HashMap_put, env.newStringUtf('INPUT'), env.newStringUtf('xxxxxxxxxx')); 
```

上面都是在反射调用的角度简单介绍了 `staticVaMethod`, `vaMethod`, `constructor` 的使用方法。

trace java methods
------------------

用 Frida 也可以实现 java 层的 trace 。  
因为基于 Frida 框架，如果直接 trace 所有的类效率太慢，也容易崩溃。所以这里是以白名单的方式实现的。核心方法就是枚举所有类，按过滤名单，匹配需要 trace 的类，Hook 目标类的所有方法（可指定），在方法被调用时，将其入参和返回值记录下来。

 

**核心逻辑：**  
遍历所有类，Hook 白名单中的类及方法。

 

**调用方式：**

```
/**
 * java 方法追踪
 * @param clazzes 要追踪类数组 ['M:Base64', 'E:java.lang.String'] ，类前面的 M 代表 match 模糊匹配，E 代表 equal 精确匹配
 * @param whitelist 指定某类方法 Hook 细则，可按白名单或黑名单过滤方法。
 *                  { '类名': {white: true, methods: ['toString', 'getBytes']} }
 * @stackFilter 按匹配字串打印堆栈。如果要匹配 bytes 数组需要十进制无空格字串，例如："104,113,-105"
 */
FCAnd.traceArtMethods(
    ['M:MainActivity', 'E:java.lang.String'],
    {'java.lang.String': {white: true, methods:['substring', 'getChars']}},
    "match_str_show_stacks"
);

```

*   支持精确 / 模糊匹配类名
*   支持某类按白名单方式 trace 方法
*   支持匹配到指定值时收集栈信息

**具体实现：**

```
Java.enumerateLoadedClassesSync().forEach((curClsName, index, array) => {
    dest_cls.forEach((destCls) => {
        // 按规则匹配是否需要 trace
        if (match(destCls, curClsName)) {
            // trace 核心方法
            traceArtMethodsCore(curClsName);
            return false; // end forEach
        }
    });
});
// Hook 核心逻辑
function traceArtMethodsCore(clsname: string) {
    let cls = Java.use(clsname);
    // 枚举方法
    let methods = cls.class.getDeclaredMethods();
    methods.forEach(function (method: any) {
        ...
        // 枚举重载
        let methodOverloads = cls[methodName].overloads;
        methodOverloads.forEach(function (overload: any) {
            ...
            // Hook
            overload.implementation = function () {
                // ... send entry msg
                // 利用 js 参数特性 arguments ，调用原函数以适配所有 Hook 方法的传参
                const retval = this[methodName].apply(this, arguments);
                // ... send exit msg
                return retval;
            }
        }
    }
}

```

通过 `python/android/traceLogCleaner.py` 脚本收集 trace 日志，将回传的日志`按线程`、`格式化`输出日志，并且对字节数组，尝试进行 string 和 hex 转换以方便搜索。

 

**格式化 trace 效果：**

 

![](https://bbs.pediy.com/upload/attach/202101/357319_B24D8Y8QFMSBCPU.png)

 

**单条日志：**

 

![](https://bbs.pediy.com/upload/attach/202101/357319_37VK2DJXBW9G3UA.png)

 

**小结：**

 

优点：用 Frida 做 java 方法级的 trace ，优点就是方便、灵活、轻量级。  
缺点：限于 Frida 框架，该方式效率较低，trace 方法过多容易崩溃，所以无法做全量 trace。  
推荐用于轻量级的 Java 方法 trace 可以有效的定位核心算法。

jni hook & trace
----------------

利用 Frida 的 `Java.vm.getEnv()` 获取 `JNIEnv` 指针，再根据 jni 结构体函数偏移，可以获取到各个 jni 函数地址，之后就可以根据需要进行 Hook 了。  
例如

*   可以将其封装成更便捷的获取各 jni 函数地址的功能，方便 Hook

**核心逻辑：**

```
// 列出 JNI 函数数组
const jni_struct_array = [
    "reserved0",
    "reserved1",
    "reserved2",
    "reserved3",
    "GetVersion",
    "DefineClass",
    "FindClass",
    "FromReflectedMethod",
    ...
];
// 获取 JNIEnv 地址
var env = Java.vm.getEnv();
var env_ptr = env.handle.readPointer();
// 根据函数名计算索引偏移
var offset = jni_struct_array.indexOf(func_name) * Process.pointerSize;
// 读取函数地址
jnienv_addr.add(offset).readPointer();
// Hook
Interceptor.attach(addr, callbacksOrProbe);

```

**应用：**

```
FCAnd.jni.hookJNI('NewStringUTF', {
    onEnter: function (args) {
      ...
    }
});

```

*   可以 Hook `RegistNatives` 来获取动态注册的 jni 函数地址

```
export function hook_registNatives() {
    const tag = 'fridaRegstNtv';
    Jni.hookJNI("RegisterNatives", {
        onEnter: function (args) {
            var env = Java.vm.getEnv();
            var p_size = Process.pointerSize;
            var methods = args[2];
            var methodcount = args[3].toInt32();
            // 获取类名
            var name = env.getClassName(args[1]);
            DMLog.i(tag, "==== class: " + name + " ====");
            DMLog.i(tag, "==== methods: " + methods + " nMethods: " + methodcount + " ====");
            /** 根据函数结构原型遍历动态注册信息
             typedef struct {
                const char* name;
                const char* signature;
                void* fnPtr;
             } JNINativeMethod;
             jint RegisterNatives(JNIEnv* env, jclass clazz, const JNINativeMethod* methods, jint nMethods)
             */
            for (var i = 0; i < methodcount; i++) {
                var idx = i * p_size * 3;
                var fnPtr = methods.add(idx + p_size * 2).readPointer();
                const module = Process.getModuleByAddress(fnPtr);
                if (module) {
                    const modulename = module.name;
                    const modulebase = module.base;
                    var logstr = "name: " + methods.add(idx).readPointer().readCString()
                        + ", signature: " + methods.add(idx + p_size).readPointer().readCString()
                        + ", fnPtr: " + fnPtr
                        + ", modulename: " + modulename + " -> base: " + modulebase;
                    if (null != modulebase) {
                        logstr += ", offset: " + fnPtr.sub(modulebase);
                    }
                    DMLog.i(tag, logstr);
                }
                else {
                    DMLog.e(tag, 'module is null');
                }
            }
        }
    });
}

```

返回效果

```
==== class: com.xxxx.class.name ====
==== methods: 0xcd52d428 nMethods: 41 ====
[INFO][fridaRegstNtv]: name: initialize, signature: ()V, fnPtr: 0xcd50b6bd, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x66bd
[INFO][fridaRegstNtv]: name: onExit, signature: ()V, fnPtr: 0xcd50b6c7, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x66c7
[INFO][fridaRegstNtv]: name: getMMKVWithID, signature: (Ljava/lang/String;ILjava/lang/String;)J, fnPtr: 0xcd50b6d1, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x66d1                  
[INFO][fridaRegstNtv]: name: encodeBool, signature: (JLjava/lang/String;Z)Z, fnPtr: 0xcd50b76d, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x676d
[INFO][fridaRegstNtv]: name: decodeBool, signature: (JLjava/lang/String;Z)Z, fnPtr: 0xcd50b7bf, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x67bf
[INFO][fridaRegstNtv]: name: encodeInt, signature: (JLjava/lang/String;I)Z, fnPtr: 0xcd50b80f, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x680f
[INFO][fridaRegstNtv]: name: decodeInt, signature: (JLjava/lang/String;I)I, fnPtr: 0xcd50b85b, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x685b
[INFO][fridaRegstNtv]: name: encodeLong, signature: (JLjava/lang/String;J)Z, fnPtr: 0xcd50b8a5, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x68a5
[INFO][fridaRegstNtv]: name: decodeLong, signature: (JLjava/lang/String;J)J, fnPtr: 0xcd50b8f7, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x68f7
[INFO][fridaRegstNtv]: name: encodeFloat, signature: (JLjava/lang/String;F)Z, fnPtr: 0xcd50b953, modulename: libxxxx.so -> base: 0xcd505000, offset: 0x6953
......

```

**FridaContainer 调用：**

```
FCAnd.jni.hook_registNatives();

```

此功能因功能比较固定、使用频率高，也单独拆出了一个仓库：https://github.com/deathmemory/fridaRegstNtv

*   另外 jni 的部分，还可以做成 jni trace

这里参考了 [jnitrace](https://github.com/chame1eon/jnitrace) 做了简化和嵌入版。

 

**具体实现：**

 

因为有了上面的基础，使 Jni 的 trace 变得简洁了

```
export function traceAllJNISimply() {
    // 遍历 Hook Jni 函数
    jni_struct_array.forEach(function (func_name, idx) {
        if (!func_name.includes("reserved")) {
            Jni.hookJNI(func_name, {
                onEnter(args) {
                    // 触发时将信息保存到对象中
                    let md = new MethodData(this.context, func_name, JNI_ENV_METHODS[idx], args);
                    this.md = md;
                },
                onLeave(retval) {
                    // 退出时将返回值追加到对象中
                    this.md.setRetval(retval);
                    // 发送日志
                    send(JSON.stringify({tid: this.threadId, status: "jnitrace", data: this.md}));
                }
            });
        }
    })
}

```

因为性能问题，在信息收集时去除了 `jnitrace` 中的 `Thread.backtrace` 和 `DebugSymbol.fromAddress` 可能会造成线程阻塞的因素，对 trace 效果没有太多影响。

 

**格式化日志输出效果：**

 

![](https://bbs.pediy.com/upload/attach/202101/357319_ZT3683VRW2CSV7J.png)

 

**FridaContainer 调用：**

```
FCAnd.jni.traceAllJNISimply();

```

参考：[jnitrace](https://github.com/chame1eon/jnitrace)

 

**小结：**  
jni 的 trace 做了速度上的优化，去除了一些线程阻塞的因素，使得在 jni 全量 trace 下也可以很好的运行。结合上面的 Java trace ，寻常难度的算法定位效率可以有一个很好的提高。

Stalker 的应用
===========

Stalker 部分的内容，bmax 大佬已经发了一篇贴子，大家可以跳转看一下。  
地址：[https://bbs.pediy.com/thread-264680.htm](https://bbs.pediy.com/thread-264680.htm)

Frida 的检测方式
===========

1.  文件名 `frida-agent**`
2.  默认端口 `27042`
3.  特征字符 `frida:rpc`、`LIBFRIDA`
4.  端口应答特征

```
if (connect(sock , (struct sockaddr*)&sa , sizeof sa) != -1) {
  memset(res, 0 , 7);
 
  send(sock, "\x00", 1, NULL);
  send(sock, "AUTH\r\n", 6, NULL);
 
  usleep(100); // Give it some time to answer
 
  if ((ret = recv(sock, res, 6, MSG_DONTWAIT)) != -1) {
    if (strcmp(res, "REJECT") == 0) {
      __android_log_print(ANDROID_LOG_VERBOSE, APPNAME,  "FRIDA DETECTED [1] - frida server running on port %d!", i);
    }
  }
}

```

**参考：**  
[AntiFrida](https://github.com/b-mueller/frida-detection-demo/blob/master/AntiFrida/app/src/main/cpp/native-lib.cpp)

总结
==

以上是我们对使用 Frida 经验的一些总结。在逆向工程里，个人认为有两个方面对逆向的提效是帮助非常大的，一个是 trace 一个是算法识别。而 Frida 在有了 Stalker 的加持下，这两个方面都可以实现。在轻量级和便捷性上，首选推荐。

 

FridaContainer：[https://github.com/deathmemory/FridaContainer](https://github.com/deathmemory/FridaContainer)  
fridaRegstNtv：[https://github.com/deathmemory/fridaRegstNtv](https://github.com/deathmemory/fridaRegstNtv)  
以上两个仓库还请大佬们多多拍砖，提建议。  
抛砖引玉，希望能和业界大佬多多交流。感谢。

[看雪社区年底排行榜，查查你的排名？](https://www.kanxue.com/rank.htm)