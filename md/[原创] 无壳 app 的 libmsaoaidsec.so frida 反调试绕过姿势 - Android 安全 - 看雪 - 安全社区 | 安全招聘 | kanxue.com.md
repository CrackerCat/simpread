> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285811.htm)

> [原创] 无壳 app 的 libmsaoaidsec.so frida 反调试绕过姿势

前言
==

当前许多应用通过集成 `libmsaoaidsec.so` 实现针对 Frida 的反调试机制，其核心逻辑是在应用启动过程中，当加载 `libmsaoaidsec.so` 动态库时，会通过 `pthread_create()` 创建独立线程，并在该线程内执行反调试函数，主动扫描 Frida 进程、端口、内存特征等痕迹。若检测到 Frida 存在，则触发进程终止或崩溃。  
针对这种机制的绕过思路可以从破坏反调试线程的加载或执行入手，文章会对不同的 app 进行测试，对于使用 libmsaoaidsec.so 来进行反调试的 app，思路是一样的。

定位
==

通过 hook dlopen() 函数根据最后加载的 so 文件来确定程序是在加载哪个 so 之后退出的

```
function hook_dlopen(){
    //Android8.0之后加载so通过android_dlopen_ext函数
    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    console.log("addr_android_dlopen_ext",android_dlopen_ext);
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            var pathptr = args[0];
            if(pathptr!=null && pathptr != undefined){
                var path = ptr(pathptr).readCString();
                console.log("android_dlopen_ext:",path);
            }
        },
        onLeave:function(retvel){
            console.log("leave!");
        }
    })
}
hook_dlopen()
```

如下，可以看到加载完 libmsaoaidsec.so 之后程序就退出了。  
![](https://bbs.kanxue.com/upload/attach/202503/993971_ANPP2BM5TBFQUKY.webp)  
接下来我们找一个时机，就是在加载 libmsaoaidsec.so 的时候打印它里面创建的线程

```
function hook_dlopen(){
    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    console.log("addr_android_dlopen_ext",android_dlopen_ext);
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            var pathptr = args[0];
            if(pathptr!=null && pathptr != undefined){
                var path = ptr(pathptr).readCString();
                if(path.indexOf("libmsaoaidsec.so")!=-1){
                    console.log("android_dlopen_ext:",path);
                    hook_pthread()
                }
            }
        },
        onLeave:function(retvel){
             
        }
    })
}
 
function hook_pthread() {
    var pth_create = Module.findExportByName("libc.so", "pthread_create");
    console.log("[pth_create]", pth_create);
    Interceptor.attach(pth_create, {
        onEnter: function (args) {
            var module = Process.findModuleByAddress(args[2]);
            if (module != null) {
                console.log("address", module.name, args[2].sub(module.base));
            }
        },
        onLeave: function (retval) {}
    });
}
 
function main(){
    hook_dlopen()
}
 
main()
```

如下看打印结果，有兴趣的小伙伴可以利用 IDA 反编译 libmsaoaidsec.so 跳转到这些地址去看这些函数都干了什么事情  
![](https://bbs.kanxue.com/upload/attach/202503/993971_VXW379PYFSY7MNX.webp)

反调试绕过
=====

现在已经知道了是哪些函数来检测的 frida，然后一个很重要的事情就是找到一个时机去把这几个函数都替换掉或者 nop 掉。  
如果去 IDA 里分析的话，跳转到这几个函数的地址，通过交叉引用查看他们的上一级函数，会发现最终他们都会聚集到`init_proc`函数里，这是一个初始化函数，在 so 加载时会自动被调用，而且是 so 加载过程中最先执行的函数，那么就要找一个比`init_proc`函数执行更早的一个时机来把检测 frida 的函数替换掉或者 nop 掉。  
在 so 文件的链接过程中 linker 里的`call_constructors()`函数会触发构造函数，对初始化函数进行注册，然后执行初始化函数，注册的初始化函数会放在`.init_array`函数列表里，可以在调用`call_constructors()`函数的时候动态替换 so 文件里检测 frida 的函数的地址，使它指向自定义的空函数。

姿势 1
----

替换函数

```
function hook_dlopen(){
    //Android8.0之后加载so通过android_dlopen_ext函数
    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    console.log("addr_android_dlopen_ext",android_dlopen_ext);
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            var pathptr = args[0];
            if(pathptr!=null && pathptr != undefined){
                var path = ptr(pathptr).readCString();
                if(path.indexOf("libmsaoaidsec.so")!=-1){
                    console.log("android_dlopen_ext:",path);
                    hook_call_constructors()
                }
            }
        },
        onLeave:function(retvel){
            //console.log("leave!");
        }
    })
}
 
function hook_call_constructors() {
    let linker = null;
    if (Process.pointerSize === 4) {
        linker = Process.findModuleByName("linker");
    } else {
        linker = Process.findModuleByName("linker64");
    }
    let call_constructors_addr, get_soname
    let symbols = linker.enumerateSymbols();
    for (let index = 0; index < symbols.length; index++) {
        let symbol = symbols[index];
        if (symbol.name === "__dl__ZN6soinfo17call_constructorsEv") {
            call_constructors_addr = symbol.address;
        } else if (symbol.name === "__dl__ZNK6soinfo10get_sonameEv") {
            get_soname = new NativeFunction(symbol.address, "pointer", ["pointer"]);
        }
    }
    console.log(call_constructors_addr)
    var listener = Interceptor.attach(call_constructors_addr, {
        onEnter: function (args) {
            console.log("hooked call_constructors")
            var module = Process.findModuleByName("libmsaoaidsec.so")
            if (module != null) {
                Interceptor.replace(module.base.add(0x1c544), new NativeCallback(function () {
                    console.log("0x1c544:替换成功")
                }, "void", []))
                Interceptor.replace(module.base.add(0x1b8d4), new NativeCallback(function () {
                    console.log("0x1b8d4:替换成功")
                }, "void", []))
                Interceptor.replace(module.base.add(0x26e5c), new NativeCallback(function () {
                    console.log("0x26e5c:替换成功")
                }, "void", []))
                listener.detach()
            }
             
        },
    })
}
function main(){
    hook_dlopen()
}
 
main()
```

如下成功绕过了  
![](https://bbs.kanxue.com/upload/attach/202503/993971_H89DCEVXF45N5N5.webp)

姿势 2
----

nop 函数

```
function nop_addr(addr) {
    Memory.protect(addr, 4 , 'rwx');
    var w = new Arm64Writer(addr);
    w.putRet();
    w.flush();
    w.dispose();
}
 
function hook_dlopen(){
    //Android8.0之后加载so通过android_dlopen_ext函数
    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    console.log("addr_android_dlopen_ext",android_dlopen_ext);
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            var pathptr = args[0];
            if(pathptr!=null && pathptr != undefined){
                var path = ptr(pathptr).readCString();
                if(path.indexOf("libmsaoaidsec.so")!=-1){
                    console.log("android_dlopen_ext:",path);
                    hook_call_constructors()
                }
            }
        },
        onLeave:function(retvel){
            //console.log("leave!");
        }
    })
}
 
function hook_call_constructors() {
    let linker = null;
    if (Process.pointerSize === 4) {
        linker = Process.findModuleByName("linker");
    } else {
        linker = Process.findModuleByName("linker64");
    }
    let call_constructors_addr, get_soname
    let symbols = linker.enumerateSymbols();
    for (let index = 0; index < symbols.length; index++) {
        let symbol = symbols[index];
        if (symbol.name === "__dl__ZN6soinfo17call_constructorsEv") {
            call_constructors_addr = symbol.address;
        } else if (symbol.name === "__dl__ZNK6soinfo10get_sonameEv") {
            get_soname = new NativeFunction(symbol.address, "pointer", ["pointer"]);
        }
    }
    console.log(call_constructors_addr)
    var listener = Interceptor.attach(call_constructors_addr, {
        onEnter: function (args) {
            console.log("hooked call_constructors")
            var module = Process.findModuleByName("libmsaoaidsec.so")
            if (module != null) {
                nop_addr(module.base.add(0x1c544))
                console.log("0x1c544:替换成功")
                nop_addr(module.base.add(0x1b8d4))
                console.log("0x1b8d4:替换成功")
                nop_addr(module.base.add(0x26e5c))
                console.log("0x26e5c:替换成功")
                listener.detach()
            }
             
        },
    })
}
function main(){
    hook_dlopen()
}
 
main()
```

如下也可以绕过  
![](https://bbs.kanxue.com/upload/attach/202503/993971_GTREY75XA8EUJN9.webp)

参考文章：  
https://mp.weixin.qq.com/s/mBJzRqP-0bGsiTqp6aJSAg  
https://mp.weixin.qq.com/s/RyJiHrSO4CU9QLJitC7Tqg  
https://bbs.kanxue.com/thread-284816.htm

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm)