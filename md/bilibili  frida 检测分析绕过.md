> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2012106-1-1.html)

> [md] 本文是对于了 ** 版本 7.26.1，以及版本 7.76.0** 的 frida 检测进行的分析研究以及绕过# bilibili 7.26.1bilibili 的旧版本 frida 检测可以看到按照 Spawn 的......

![](https://avatar.52pojie.cn/data/avatar/001/97/85/65_avatar_middle.jpg)chenchenchen777 _ 本帖最后由 chenchenchen777 于 2025-3-6 16:15 编辑_  

本文是对于了**版本 7.26.1，以及版本 7.76.0** 的 frida 检测进行的分析研究以及绕过

bilibili 7.26.1
---------------

bilibili 的旧版本 frida 检测  
![](https://attach.52pojie.cn/forum/202503/06/160055j7duyud5oryf0ftz.png)

**image-20250304173059677.png** _(323.94 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM4NHxiMDhkY2ZkZXwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:00 上传

可以看到按照 Spawn 的方式启动的时候，直接 frida 就被检测了。我们按照 hook dlopen 去查看可能出现的对应 frida 检测的 so 文件。

```
function hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
        onEnter: function(args) {
            var pathptr = args[0];
            if (pathptr) {
                var path = ptr(pathptr).readCString();
                console.log("Loading: " + path);
                if (path.indexOf(soName) >= 0) {
                    console.log("Already loading: " + soName);
                    // hook_system_property_get();
                }
            }
        }
    });
}
setImmediate(hook_dlopen, "libmsaoaidsec.so");
```

![](https://attach.52pojie.cn/forum/202503/06/160058h664ghhdj1yj2h5h.png)

**image-20250304173504546.png** _(439.55 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM4NXw1M2I5OWE0ZHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:00 上传

可以看到的是 libmsaoaidsec.so 文件，在 dlopen 了之后，frida 就被检测到了，所以大概率的可能是在 libmsaoaidsec.so 进行的 frida 检测。在这里之前，我们需要知道是在哪进行 HOOK 是最为有用的

这里我们可以通过在 dlopen 结束之后，去 HOOK JNI_Onload 函数，去判断检测函数在 JNI_Onload 之前还是之后，我们通过 IDA 可以去查看 JNI_Onload 的地址。这里是在 JNI_Onload 之前出现的 frida 检测。

![](https://attach.52pojie.cn/forum/202503/06/160100sm8n3lzlgx37ql7z.png)

**image-20250304174423941.png** _(109.82 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM4NnwzYzgxZDA4ZnwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

```
function hook_JNI_OnLoad(){
    let module = Process.findModuleByName("libmsaoaidsec.so")
    Interceptor.attach(module.base.add(0xC6DC + 1), {
        onEnter(args){
            console.log("JNI_OnLoad")
        }
    })
}
```

我们通过 HOOK 进程创建，来看看对于 frida 检测的线程是在哪里启动的。在复现过程中，原作者使用了 hook _system_property_get 函数，这里是 init_proc 函数较早的位置，这里涉及到了安卓源码中 dlopen 和. init_xx 函数的执行流程比较，在我的 so 文件执行流程的过程中有细节分析。

![](https://attach.52pojie.cn/forum/202503/06/160104fjp3rua1ga1liruw.png)

**image-20250304175152829.png** _(118.6 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM4OHwyOWQ5NjEyNHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

![](https://attach.52pojie.cn/forum/202503/06/160102fo6o2taa5amlmyjj.png)

**image-20250304175126362.png** _(19.96 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM4N3w2MzQyOWU0NnwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

```
function hook_system_property_get() {
    var system_property_get_addr = Module.findExportByName(null, "__system_property_get");
    if (!system_property_get_addr) {
        console.log("__system_property_get not found");
        return;
    }

    Interceptor.attach(system_property_get_addr, {
        onEnter: function(args) {
            var nameptr = args[0];
            if (nameptr) {
                var name = ptr(nameptr).readCString();
                if (name.indexOf("ro.build.version.sdk") >= 0) {
                    console.log("Found ro.build.version.sdk, need to patch");
                    // hook_pthread_create();
                    // bypass()
                    //这里可以开始进行HOOK
                }
            }
        }
    });
}
```

由于我们知道，frida 检测的是在 JNI_Onload 函数之前，那么我们就要在 init_proc 的越早的地方可以进行 HOOK，我们 HOOK 的地方就是线程创建的位置 pthread_create。

```
function hook_pthread_create() {
    var pthread_create = Module.findExportByName("libc.so", "pthread_create");
    var libmsaoaidsec = Process.findModuleByName("libmsaoaidsec.so");

    if (!libmsaoaidsec) {
        console.log("libmsaoaidsec.so not found");
        return;
    }

    console.log("libmsaoaidsec.so base: " + libmsaoaidsec.base);

    if (!pthread_create) {
        console.log("pthread_create not found");
        return;
    }

    Interceptor.attach(pthread_create, {
        onEnter: function(args) {
            var thread_ptr = args[2];
            if (thread_ptr.compare(libmsaoaidsec.base) < 0 || thread_ptr.compare(libmsaoaidsec.base.add(libmsaoaidsec.size)) >= 0) {
                console.log("pthread_create other thread: " + thread_ptr);
            } else {
                console.log("pthread_create libmsaoaidsec.so thread: " + thread_ptr + " offset: " + thread_ptr.sub(libmsaoaidsec.base));
            }
        },
        onLeave: function(retval) {}
    });
}
```

在这里我们通过去 HOOK dlopen 的位置，通过 dlopen 的位置去 HOOK **system_property_get** 当参数是 ro.build.version.sdk 然后去 hook pthread_create

dlopen ———>system_property_get————>pthread_create  
![](https://attach.52pojie.cn/forum/202503/06/160122e1f1o86je37fjjmp.png)

**image-20250305101304937.png** _(1005.51 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM4OXw4OTkwZjkzZnwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

这里去比较了对应所以由 pthread_create 创建的线程，当对应的线程的地址在 libmsaoaidsec.so 的地址区域内的时候，打印对应的地址以及偏移。可以看到这里有两个线程出现了

我们可以去通过 IDA 看看  
![](https://attach.52pojie.cn/forum/202503/06/160124n2i72vr2u22wriml.png)

**image-20250305101938410.png** _(91.55 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5MHxhMDZmZGNkNnwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

![](https://attach.52pojie.cn/forum/202503/06/160126yfob7fnz5w50bi5a.png)

**image-20250305102030763.png** _(25.16 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5MXw3ZDkwNGM3NXwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

  
![](https://attach.52pojie.cn/forum/202503/06/160129myiyfi88e8ztowyr.png)

**image-20250305102554308.png** _(41.49 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5MnxjZTViNWY5ZXwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

这些位置的像出现的 strcmp        openat        strstr....... 很多的 frida 的检测，我们交叉引用一下看看 pthread_create，然后直接实现 NOP 就可以了  
![](https://attach.52pojie.cn/forum/202503/06/160131aqfspsq8ifs8dfqq.png)

**image-20250305102842641.png** _(103.62 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5M3w5ZTcwMTEyOHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

![](https://attach.52pojie.cn/forum/202503/06/160133knt8bvjj4gjzhb8n.png)

**image-20250305103007170.png** _(119.26 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5NHw2MDdjZWE0OHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

#### 总体代码

```
function hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
        onEnter: function(args) {
            var pathptr = args[0];
            if (pathptr) {
                var path = ptr(pathptr).readCString();
                console.log("Loading: " + path);
                if (path.indexOf(soName) >= 0) {
                    console.log("Already loading: " + soName);
                    hook_system_property_get();
                }
            }
        }
    });
}

function hook_system_property_get() {
    var system_property_get_addr = Module.findExportByName(null, "__system_property_get");
    if (!system_property_get_addr) {
        console.log("__system_property_get not found");
        return;
    }

    Interceptor.attach(system_property_get_addr, {
        onEnter: function(args) {
            var nameptr = args[0];
            if (nameptr) {
                var name = ptr(nameptr).readCString();
                if (name.indexOf("ro.build.version.sdk") >= 0) {
                    console.log("Found ro.build.version.sdk, need to patch");
                    hook_pthread_create();
                    // bypass()
                }
            }
        }
    });
}

function hook_pthread_create() {
    var pthread_create = Module.findExportByName("libc.so", "pthread_create");
    var libmsaoaidsec = Process.findModuleByName("libmsaoaidsec.so");

    if (!libmsaoaidsec) {
        console.log("libmsaoaidsec.so not found");
        return;
    }

    console.log("libmsaoaidsec.so base: " + libmsaoaidsec.base);

    if (!pthread_create) {
        console.log("pthread_create not found");
        return;
    }

    Interceptor.attach(pthread_create, {
        onEnter: function(args) {
            var thread_ptr = args[2];
            if (thread_ptr.compare(libmsaoaidsec.base) < 0 || thread_ptr.compare(libmsaoaidsec.base.add(libmsaoaidsec.size)) >= 0) {
                console.log("pthread_create other thread: " + thread_ptr);
            } else {
                console.log("pthread_create libmsaoaidsec.so thread: " + thread_ptr + " offset: " + thread_ptr.sub(libmsaoaidsec.base));
            }
        },
        onLeave: function(retval) {}
    });
}
function nop_code(addr)
{
    Memory.patchCode(ptr(addr),4,code => {
        const cw =new ThumbWriter(code,{pc:ptr(addr)});
        cw.putNop();
        cw.putNop();
        cw.flush();
    })
}

function bypass()
{
    let module = Process.findModuleByName("libmsaoaidsec.so")
    nop_code(module.base.add(0x010AE4))
    nop_code(module.base.add(0x113F8))
}
setImmediate(hook_dlopen, "libmsaoaidsec.so");
```

![](https://attach.52pojie.cn/forum/202503/06/160137no7xoouxo9qnieui.png)

**image-20250305103610008.png** _(2.07 MB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5NXxhNzdkYWM2ZnwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

这里就绕过 frida 了，或者通过直接在 IDA 中去 patch 掉上面两个位置的 pthread_create，然后把补丁之后的 so 再放到 APK 中也可以。

参考文章：[[原创] 绕过 bilibili frida 反调试 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-277034.htm)

bilibili7.76.0
--------------

**7.26.1:**  
![](https://attach.52pojie.cn/forum/202503/06/160146gri2uq7hjiajhu8a.png)

**image-20250305141556864.png** _(45.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5OHwwNzliN2Q4N3wxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

在高一点的版本上面，pthread_create 函数是被隐藏了的  
![](https://attach.52pojie.cn/forum/202503/06/160148eegns0s4ugggkshs.png)

**image-20250305141808889.png** _(81.61 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTM5OXw5NWJlMWRjZHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

但是其实我们通过之前的方法也能看到对应的 pthead_create 创建线程的位置

```
function hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
        onEnter: function(args) {
            var pathptr = args[0];
            if (pathptr) {
                var path = ptr(pathptr).readCString();
                console.log("Loading: " + path);
                if (path.indexOf(soName) >= 0) {
                    console.log("Already loading: " + soName);
                    hook_system_property_get();
                }
            }
        }
    });
}

function hook_system_property_get() {
    var system_property_get_addr = Module.findExportByName(null, "__system_property_get");
    if (!system_property_get_addr) {
        console.log("__system_property_get not found");
        return;
    }

    Interceptor.attach(system_property_get_addr, {
        onEnter: function(args) {
            var nameptr = args[0];
            if (nameptr) {
                var name = ptr(nameptr).readCString();
                if (name.indexOf("ro.build.version.sdk") >= 0) {
                    console.log("Found ro.build.version.sdk, need to patch");
                    hook_pthread_create();
                    // bypass()
                }
            }
        }
    });
}

function hook_pthread_create() {
    var pthread_create = Module.findExportByName("libc.so", "pthread_create");
    var libmsaoaidsec = Process.findModuleByName("libmsaoaidsec.so");

    if (!libmsaoaidsec) {
        console.log("libmsaoaidsec.so not found");
        return;
    }

    console.log("libmsaoaidsec.so base: " + libmsaoaidsec.base);

    if (!pthread_create) {
        console.log("pthread_create not found");
        return;
    }

    Interceptor.attach(pthread_create, {
        onEnter: function(args) {
            var thread_ptr = args[2];
            if (thread_ptr.compare(libmsaoaidsec.base) < 0 || thread_ptr.compare(libmsaoaidsec.base.add(libmsaoaidsec.size)) >= 0) {
                console.log("pthread_create other thread: " + thread_ptr);
            } else {
                console.log("pthread_create libmsaoaidsec.so thread: " + thread_ptr + " offset: " + thread_ptr.sub(libmsaoaidsec.base));
            }
        },
        onLeave: function(retval) {}
    });
}
setImmediate(hook_dlopen, "libmsaoaidsec.so");
```

![](https://attach.52pojie.cn/forum/202503/06/160151onzf2uduugqoftgt.png)

**image-20250305144616783.png** _(860.96 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwMHw3ODRkOGNlZXwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

这里可以看到在 libmsaoaidsec.so 中创建了三个新的线程，这里多半也就是进行 frida 检测的位置的了

```
pthread_create libmsaoaidsec.so thread: 0x76bcce0544 offset: 0x1c544
pthread_create libmsaoaidsec.so thread: 0x76bccdf8d4 offset: 0x1b8d4
pthread_create libmsaoaidsec.so thread: 0x76bcceae5c offset: 0x26e5c
```

同时这里我们进行 HOOK 的检测 frida 线程在哪的时候，也是在 system_property_get 的函数的位置进行 HOOK 的，但是实际上这里的位置也已经被混淆了，但是这个函数没有被混淆，可以直接在导入表里面找到的，那么我们就去交叉引用看看在哪引用的  
![](https://attach.52pojie.cn/forum/202503/06/160153r99q9mxl66p92qde.png)

**image-20250305144950838.png** _(44.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwMXw0OGMxMTQ0N3wxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

![](https://attach.52pojie.cn/forum/202503/06/160155cmd6g0lz3llitggu.png)

**image-20250305145028014.png** _(72.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwMnwyNzc2YmE0NHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

  
其实是能够发现还是在 init_proc 的 sub_123F0 函数的位置的  
![](https://attach.52pojie.cn/forum/202503/06/160157o66dv4w4zkzjem62.png)

**image-20250305145110772.png** _(38.4 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwM3xhNzU1MjNlMXwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

但是这里去判断参数的为 "ro.build.version.sdk" 已经被混淆了，但是我们既然能通过这个代码找到对应的线程，说明实际上还是去执行了 **_system_property_get（"ro.build.version.sdk"）**这个函数的，而且既然代码可以直接检测得到 frida 检测的线程的地址，那么说明检测点还是再_system_property_get（"ro.build.version.sdk"）之前的。

我们同样按照交叉引用看看 pthread_create 被混淆成了什么  
![](https://attach.52pojie.cn/forum/202503/06/160159heywmaey8gks8cf7.png)

**image-20250305150332542.png** _(17.96 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwNHw2NjY1MGEzZnwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:01 上传

![](https://attach.52pojie.cn/forum/202503/06/160201me7hs7z2vrsg400g.png)

**image-20250305150358745.png** _(14.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwNXw5MWRhOGEwY3wxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:02 上传

  
![](https://attach.52pojie.cn/forum/202503/06/160203b5qugd0s6qdkjz5q.png)

**image-20250305150437030.png** _(19.08 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwNnxhOTlkNzk4ZHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:02 上传

  
其实通过参数的形式，我们就可以判断大概率就是被混淆了的 pthead_create 函数

### 正面对抗

按照之前旧版本的 frida 检测，我们的反检测是通过 NOP 掉对应的 frida 检测线程，在 7.26.1 中 frida 检测是有两个线程的，而在 7.76.0 中，这里的 frida 检测有了三个线程，那么我们也可以对于这里的三个线程进行 NOP

这里按照最原始的方法，不去 NOP 掉 **pthead_create 函数**，而是去 patch 掉对应的 frida 函数。以及其中的 NOP 的实际，我选择的位置是在判断到进入 libmsaoaidsec.so 的时候，并且开始进行 frida 检测线程创建的 pthead_create 函数时期  

```
function hook_pthread_create() {
    var pthread_create = Module.findExportByName("libc.so", "pthread_create");
    var libmsaoaidsec = Process.findModuleByName("libmsaoaidsec.so");

    if (!libmsaoaidsec) {
        console.log("libmsaoaidsec.so not found");
        return;
    }

    console.log("libmsaoaidsec.so base: " + libmsaoaidsec.base);

    if (!pthread_create) {
        console.log("pthread_create not found");
        return;
    }

    Interceptor.attach(pthread_create, {
        onEnter: function(args) {
            var thread_ptr = args[2];
            if (thread_ptr.compare(libmsaoaidsec.base) < 0 || thread_ptr.compare(libmsaoaidsec.base.add(libmsaoaidsec.size)) >= 0) {
                console.log("pthread_create other thread: " + thread_ptr);
            } else {
                console.log("pthread_create libmsaoaidsec.so thread: " + thread_ptr + " offset: " + thread_ptr.sub(libmsaoaidsec.base));
                Interceptor.replace(libmsaoaidsec.base.add(0x1c544),new NativeCallback(function(){
                    console.log("Interceptor.replace: 0x1c544")
                },"void",[]))
                Interceptor.replace(libmsaoaidsec.base.add(0x1b8d4),new NativeCallback(function(){
                    console.log("Interceptor.replace: 0x1c544")
                },"void",[]))
                Interceptor.replace(libmsaoaidsec.base.add(0x26e5c),new NativeCallback(function(){
                    console.log("Interceptor.replace: 0x1c544")
                },"void",[]))
            }
        },
        onLeave: function(retval) {}
    });
}
```

![](https://attach.52pojie.cn/forum/202503/06/160207nl7l5tqlqtss7hq6.png)

**image-20250306152208226.png** _(1.89 MB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwN3xiOGMwNTIzMHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:02 上传

可以看到是直接绕过了 frida 检测的位置的。并且我写的一个 frida 打印函数也是成功执行了

### 取巧绕过

[绕过最新版 bilibili app 反 frida 机制 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-281584.htm)

在这一篇文章中，作者并没有去实现正面对抗，而且取巧绕过了，通过的方式就是在 HOOK dlsym 函数，在进入 libmsaoaidsec.so 之后去 HOOK dlsym 判断调用 pthead_create 函数的次数，在前两次进行调用 pthead_create 函数时，去调用 fake_pthead_create 函数，从而实现 frida 线程不启动，达成绕过。

以下是取巧绕过的代码 (复制于上面的网址)：

```
function create_fake_pthread_create() {
    const fake_pthread_create = Memory.alloc(4096)
    Memory.protect(fake_pthread_create, 4096, "rwx")
    Memory.patchCode(fake_pthread_create, 4096, code => {
        const cw = new Arm64Writer(code, { pc: ptr(fake_pthread_create) })
        cw.putRet()
    })
    return fake_pthread_create
}

function hook_dlsym() {
    var count = 0
    console.log("=== HOOKING dlsym ===")
    var interceptor = Interceptor.attach(Module.findExportByName(null, "dlsym"),
        {
            onEnter: function (args) {
                const name = ptr(args[1]).readCString()
                console.log("[dlsym]", name)
                if (name == "pthread_create") {
                    count++
                }
            },
            onLeave: function(retval) {
                if (count == 1) {
                    retval.replace(fake_pthread_create)
                }
                else if (count == 2) {
                    retval.replace(fake_pthread_create)
                    // 完成2次替换, 停止hook dlsym
                    interceptor.detach()
                }
            }
        }
    )
    return Interceptor
}

function hook_dlopen() {
    var interceptor = Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    console.log("[LOAD]", path)
                    if (path.indexOf("libmsaoaidsec.so") > -1) {
                        hook_dlsym()
                    }
                }
            }
        }
    )
    return interceptor
}

// 创建虚假pthread_create
var fake_pthread_create = create_fake_pthread_create()
var dlopen_interceptor = hook_dlopen()
```

![](https://attach.52pojie.cn/forum/202503/06/160211hzbaaxibbbgwi8nb.png)

**image-20250306153129200.png** _(2.25 MB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc1OTQwOHwzYjY2N2VmOHwxNzQxMjUwNDk1fDIxMzQzMXwyMDEyMTA2&nothumb=yes)

2025-3-6 16:02 上传

同样也可以实现绕过，不过，我们其实能知道，最开始的时候，其实是发现在 libmsaoaidsec.so 一共是开启了三个线程的，其实我觉得这里可以把 count 设置到三

```
onLeave: function(retval) {
                if (count == 1) {
                    retval.replace(fake_pthread_create)
                }
                else if (count == 2) {
                    retval.replace(fake_pthread_create)
                }
                else if (count == 3) {
                    retval.replace(fake_pthread_create)
                    // 完成3次替换, 停止hook dlsym
                    interceptor.detach()
                }
            }
```

这样我尝试过，也能实现绕过的。