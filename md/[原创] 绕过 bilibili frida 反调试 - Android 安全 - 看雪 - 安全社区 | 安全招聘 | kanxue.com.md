> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277034.htm)

> [原创] 绕过 bilibili frida 反调试

[原创] 绕过 bilibili frida 反调试

2023-4-29 15:51 19517

[举报](javascript:void(0);)

### [原创] 绕过 bilibili frida 反调试

 [![](http://passport.kanxue.com/upload/avatar/475/929475.png?1655038993)](user-home-929475.htm) [马到成功 *](user-home-929475.htm) ![](https://bbs.kanxue.com/view/img/rank/3.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 2023-4-29 15:51  19517

原创文章，首发至博客 http://t.csdn.cn/Pd9aL

 

文章仅供思路参考，请勿用作非法攻击

### 环境

*   bilibili 7.26.1
*   arm
*   frida 15.2.2(去除特征版本)
*   pixel 6 android 12

### 正文

使用 frida 以 spawn 模式启动应用，frida 进程直接被杀掉了

 

![](https://bbs.kanxue.com/upload/attach/202304/929475_PFPSN9V6JPNZPU9.jpg)

 

我需要知道是那个 so 在检测 frida，可以 hook dlopen 看一下 so 的加载流程

```
function hook_dlopen() {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    console.log("load " + path);
                }
            }
        }
    );
}

```

![](https://bbs.kanxue.com/upload/attach/202304/929475_FNMGQ6BYPN7SXSY.jpg)

 

由 so 的加载流程可知，当 libmsaoaidsec.so 被加载之后，frida 进程就被杀掉了，因此监测点在 libmsaoaidsec.so 中。

 

如果有了解过 so 的加载流程，那么就会知道 linker 会先对 so 进行加载与链接，然后调用 so 的. init_proc 函数，接着调用. init_array 中的函数，最后才是 JNI_OnLoad 函数，所以我需要先确定检测点大概在哪个函数中。

 

使用 frida hook JNI_OnLoad 函数，如果调用了该函数就输出一行日志，如果没有日志输出，那么就说明检测点在. init_xxx 函数中，注入的时机可以选择 dlopen 加载 libmsaoaidsec.so 完成之后。

```
function hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_JNI_OnLoad()
                }
            }
        }
    );
}
 
function hook_JNI_OnLoad(){
    let module = Process.findModuleByName("libmsaoaidsec.so")
    Interceptor.attach(module.base.add(0xC6DC + 1), {
        onEnter(args){
            console.log("call JNI_OnLoad")
        }
    })
}
 
setImmediate(hook_dlopen, "libmsaoaidsec.so")

```

![](https://bbs.kanxue.com/upload/attach/202304/929475_3MNWXSB5QNYXFYA.jpg)

 

并没有输出日志，那么说明检测的位置在 JNI_OnLoad 函数之前，所以我需要 hook .init_xxx 的函数，但这里有一个问题，dlopen 函数调用完成之后. init_xxx 函数已经执行完成了，这个时候不容易使用 frida 进行 hook

 

![](https://bbs.kanxue.com/upload/attach/202304/929475_2XY6FG3JTBDJM42.jpg)

 

这个问题其实很麻烦的，因为你想要 hook linker 的 call_function 并不容易，这里面涉及到 linker 的自举，我想到了一个取巧的办法，请看接下来的操作。

 

首先在. init_proc 函数中找一个调用了外部函数的位置，时机越早越好

 

![](https://bbs.kanxue.com/upload/attach/202304/929475_H3M92FC9ETTQETZ.jpg)

 

我选择了_system_property_get 函数，接下来使用 frida hook dlopen 函数，当加载 libmsaoaidsec.so 时，在 onEnter 回调方法中 hook _system_property_get 函数，以 "ro.build.version.sdk" 字符串作为过滤器。

 

如果_system_property_get 函数被调用了，那么这个时候也就是. init_proc 函数刚刚调用的时候，在这个时机点可以注入我想要的代码，具体实现如下：

```
function hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        locate_init()
                    }
                }
            }
        }
    );
}
 
function locate_init() {
    let secmodule = null
    Interceptor.attach(Module.findExportByName(null, "__system_property_get"),
        {
            // _system_property_get("ro.build.version.sdk", v1);
            onEnter: function (args) {
                secmodule = Process.findModuleByName("libmsaoaidsec.so")
                var name = args[0];
                if (name !== undefined && name != null) {
                    name = ptr(name).readCString();
                    if (name.indexOf("ro.build.version.sdk") >= 0) {
                        // 这是.init_proc刚开始执行的地方，是一个比较早的时机点
                        // do something
                    }
                }
            }
        }
    );
}
 
setImmediate(hook_dlopen, "libmsaoaidsec.so")

```

在获取了一个非常早的注入时机之后，就可以定位具体的 frida 检测点了。网上对 frida 的检测通常会使用 openat、open、strstr、pthread_create、snprintf、sprintf、readlinkat 等一系列函数，从这里下手是一个不错的选择。

 

我对 pthread_create 函数进行 hook，打印一下新线程要执行的函数地址

```
function hook_pthread_create(){
    console.log("libmsaoaidsec.so --- " + Process.findModuleByName("libmsaoaidsec.so").base)
    Interceptor.attach(Module.findExportByName("libc.so", "pthread_create"),{
        onEnter(args){
            let func_addr = args[2]
            console.log("The thread function address is " + func_addr)
        }
    })
}

```

![](https://bbs.kanxue.com/upload/attach/202304/929475_M2527H4BAQPFJTY.jpg)

 

这里面有两个线程是 libmsaoaidsec.so 创建的，对应的函数偏移分别是 0x11129 和 0x10975

 

这两个函数都检测了 frida，想要了解具体检测方法的可以自己看看，这里不再深入。

 

绕过的方法很简单，直接 nop 掉 pthread_create 或者替换检测函数的代码逻辑都可以，我是直接把 pthread_create 函数 nop 掉了，下面是完整代码。

```
function hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        locate_init()
                    }
                }
            }
        }
    );
}
 
function locate_init() {
    let secmodule = null
    Interceptor.attach(Module.findExportByName(null, "__system_property_get"),
        {
            // _system_property_get("ro.build.version.sdk", v1);
            onEnter: function (args) {
                secmodule = Process.findModuleByName("libmsaoaidsec.so")
                var name = args[0];
                if (name !== undefined && name != null) {
                    name = ptr(name).readCString();
                    if (name.indexOf("ro.build.version.sdk") >= 0) {
                        // 这是.init_proc刚开始执行的地方，是一个比较早的时机点
                        // do something
                        // hook_pthread_create()
                        bypass()
                    }
                }
            }
        }
    );
}
 
function hook_pthread_create() {
    console.log("libmsaoaidsec.so --- " + Process.findModuleByName("libmsaoaidsec.so").base)
    Interceptor.attach(Module.findExportByName("libc.so", "pthread_create"), {
        onEnter(args) {
            let func_addr = args[2]
            console.log("The thread function address is " + func_addr)
        }
    })
}
 
function nop(addr) {
    Memory.patchCode(ptr(addr), 4, code => {
        const cw = new ThumbWriter(code, { pc: ptr(addr) });
        cw.putNop();
        cw.putNop();
        cw.flush();
    });
}
 
function bypass(){
    let module = Process.findModuleByName("libmsaoaidsec.so")
    nop(module.base.add(0x10AE4))
    nop(module.base.add(0x113F8))
}
 
setImmediate(hook_dlopen, "libmsaoaidsec.so")

```

![](https://bbs.kanxue.com/upload/attach/202304/929475_3QFVB25SFRG9P4V.jpg)

 

至此，frida 检测也就成功绕过了。

  

[看雪 ·2023 KCTF 年度赛即将来袭！ [防守方] 规则发布，征题截止 08 月 25 日](https://bbs.kanxue.com/thread-276642.htm)

[#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#HOOK 注入](forum-161-1-125.htm)