> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278145.htm)

> [原创] frida 常用检测点及其原理 -- 一把梭方案

[原创] frida 常用检测点及其原理 -- 一把梭方案

14 小时前 549

### [原创] frida 常用检测点及其原理 -- 一把梭方案

 [![](http://passport.kanxue.com/upload/avatar/141/976141.png?1681117609)](user-home-976141.htm) [Dem0li](user-home-976141.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star_0.gif) 14 小时前  549

frida 常见反调试
===========

> 查看哪个 so 在检测 frida

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

*   so 的加载流程，那么就会知道 linker 会先对 so 进行加载与链接，然后调用 so 的. init_proc 函数，接着调用. init_array 中的函数，最后才是 JNI_OnLoad 函数，所以我需要先确定检测点大概在哪个函数中。

> 管他检测啥、直接一把梭

```
function replace_str() {
    var pt_strstr = Module.findExportByName("libc.so", 'strstr');
    var pt_strcmp = Module.findExportByName("libc.so", 'strcmp');
 
    Interceptor.attach(pt_strstr, {
        onEnter: function (args) {
            var str1 = args[0].readCString();
            var str2 = args[1].readCString();
            if (
                str2.indexOf("REJECT") !== -1 ||
                str2.indexOf("tmp") !== -1 ||
                str2.indexOf("frida") !== -1 ||
                str2.indexOf("gum-js-loop") !== -1 ||
                str2.indexOf("gmain") !== -1 ||
                str2.indexOf("linjector") !== -1
            ) {
                console.log("strstr-->", str1, str2);
                this.hook = true;
            }
        }, onLeave: function (retval) {
            if (this.hook) {
                retval.replace(0);
            }
        }
    });
 
    Interceptor.attach(pt_strcmp, {
        onEnter: function (args) {
            var str1 = args[0].readCString();
            var str2 = args[1].readCString();
            if (
                str2.indexOf("REJECT") !== -1 ||
                str2.indexOf("tmp") !== -1 ||
                str2.indexOf("frida") !== -1 ||
                str2.indexOf("gum-js-loop") !== -1 ||
                str2.indexOf("gmain") !== -1 ||
                str2.indexOf("linjector") !== -1
            ) {
                //console.log("strcmp-->", str1, str2);
                this.hook = true;
            }
        }, onLeave: function (retval) {
            if (this.hook) {
                retval.replace(0);
            }
        }
    })
 
}
 
replace_str();

```

*   测试 xhs 、bilibili 都可以过

1、检测文件名（改名）、端口名 27042（改端口）、双进程保护（spawn 启动）
------------------------------------------

2、检测 D-Bus
----------

*   `检测的原因是因为安卓系统中，D-Bus通信并不常见，而且不同于在传统Linux系统中广泛使用，安卓系统使用Binder机制来实现进程间通信（IPC），而不是使用D-Bus`
*   D-Bus 是一种进程间通信 (IPC) 和远程过程调用 (RPC) 机制, 最初是为 Linux 开发的, 目的是用一个统一的协议替代现有的和竞争的 IPC 解决方案。

```
遍历连接手机所有端口发送D-bus消息，如果返回"REJECT"这个特征则认为存在frida-server。
 
内存中存在frida rpc字符串，认为有frida-server
/*
 * Mini-portscan to detect frida-server on any local port.
 */
for(i = 0 ; i <= 65535 ; i++) {
    sock = socket(AF_INET , SOCK_STREAM , 0);
    sa.sin_port = htons(i);
    if (connect(sock , (struct sockaddr*)&sa , sizeof sa) != -1) {
        __android_log_print(ANDROID_LOG_VERBOSE, APPNAME,  "FRIDA DETECTION [1]: Open Port: %d", i);
        memset(res, 0 , 7);
        // send a D-Bus AUTH message. Expected answer is “REJECT"
        send(sock, "\x00", 1, NULL);
        send(sock, "AUTH\r\n", 6, NULL);
        usleep(100);
        if (ret = recv(sock, res, 6, MSG_DONTWAIT) != -1) {
            if (strcmp(res, "REJECT") == 0) {
               /* Frida server detected. Do something… */
            }
        }
    }
    close(sock);
}

```

*   检测 D-Bus 可以通过 hook 系统库函数，比如 strstr、strcmp 等等
    
    ```
    function replace_str() {
        var pt_strstr = Module.findExportByName("libc.so", 'strstr');
        var pt_strcmp = Module.findExportByName("libc.so", 'strcmp');
     
        Interceptor.attach(pt_strstr, {
            onEnter: function (args) {
                var str1 = args[0].readCString();
                var str2 = args[1].readCString();
                if (str2.indexOf("REJECT") !== -1) {
                    //console.log("strcmp-->", str1, str2);
                    this.hook = true;
                }
            }, onLeave: function (retval) {
                if (this.hook) {
                    retval.replace(0);
                }
            }
        });
     
        Interceptor.attach(pt_strcmp, {
            onEnter: function (args) {
                var str1 = args[0].readCString();
                var str2 = args[1].readCString();
                if (str2.indexOf("REJECT") !== -1) {
                    //console.log("strcmp-->", str1, str2);
                    this.hook = true;
                }
            }, onLeave: function (retval) {
                if (this.hook) {
                    retval.replace(0);
                }
            }
        })
     
    }
     
    replace_str();
    
    ```
    

3、检测 / proc/pid/maps 依赖文件
-------------------------

*   `maps文件中存储的是APP运行时加载的依赖`
    
*   当启动 frida 后，在 maps 文件中就会存在 `frida-agent-64.so`、`frida-agent-32.so` 文件。
    
    ```
    char line[512];
    FILE* fp;
    fp = fopen("/proc/self/maps", "r");
    if (fp) {
        while (fgets(line, 512, fp)) {
            if (strstr(line, "frida")) {
                /* Evil library is loaded. Do something… */
            }
        }
        fclose(fp);
        } else {
           /* Error opening /proc/self/maps. If this happens, something is off. */
        }
    }
    
    ```
    
*   绕过
    
    ```
    function replace_str_maps() {
        var pt_strstr = Module.findExportByName("libc.so", 'strstr');
        var pt_strcmp = Module.findExportByName("libc.so", 'strcmp');
     
        Interceptor.attach(pt_strstr, {
            onEnter: function (args) {
                var str1 = args[0].readCString();
                var str2 = args[1].readCString();
                if (str2.indexOf("REJECT") !== -1  || str2.indexOf("frida") !== -1) {
                    this.hook = true;
                }
            }, onLeave: function (retval) {
                if (this.hook) {
                    retval.replace(0);
                }
            }
        });
     
        Interceptor.attach(pt_strcmp, {
            onEnter: function (args) {
                var str1 = args[0].readCString();
                var str2 = args[1].readCString();
                if (str2.indexOf("REJECT") !== -1  || str2.indexOf("frida") !== -1) {
                    this.hook = true;
                }
            }, onLeave: function (retval) {
                if (this.hook) {
                    retval.replace(0);
                }
            }
        })
     
    }
     
    replace_str();
    
    ```
    

4、检测 / proc/pid/tast 下线程、fd 目录下打开文件
-----------------------------------

*   在 `/proc/pid/task` 目录下，可以通过查看不同的线程子目录，来获取进程中每个线程的运行时信息。这些信息包括线程的状态、线程的寄存器内容、线程占用的 CPU 时间、线程的堆栈信息等。通过这些信息，可以实时观察和监控进程中每个线程的运行状态，帮助进行调试、性能优化和问题排查等工作。
    
*   `/proc/pid/fd` 目录的作用在于提供了一种方便的方式来查看进程的文件描述符信息，这对于调试和监控进程非常有用。通过查看文件描述符信息，可以了解进程打开了哪些文件、网络连接等，帮助开发者和系统管理员进行问题排查和分析工作。
    
*   打开 frida 调试后这个 task 目录下会多出几个线程，检测点在查看这些多出来的线程是否和 frida 调试相关。
    
*   在某些 app 中就会去读取 `/proc/stask/线程ID/status` 文件，如果是运行 frida 产生的，则进行反调试。例如：`gmain/gdbus/gum-js-loop/pool-frida`等
    
    1.  gmain：Frida 使用 Glib 库，其中的主事件循环被称为 GMainLoop。在 Frida 中，gmain 表示 GMainLoop 的线程。
    2.  gdbus：GDBus 是 Glib 提供的一个用于 D-Bus 通信的库。在 Frida 中，gdbus 表示 GDBus 相关的线程。
    3.  gum-js-loop：Gum 是 Frida 的运行时引擎，用于执行注入的 JavaScript 代码。gum-js-loop 表示 Gum 引擎执行 JavaScript 代码的线程。
    4.  pool-frida：Frida 中的某些功能可能会使用线程池来处理任务，pool-frida 表示 Frida 中的线程池。
    5.  linjector 是一种用于 Android 设备的开源工具，它允许用户在运行时向 Android 应用程序注入动态链接库（DLL）文件。通过注入 DLL 文件，用户可以修改应用程序的行为、调试应用程序、监视函数调用等，这在逆向工程、安全研究和动态分析中是非常有用的。
*   绕过
    
    ```
    function replace_str() {
        var pt_strstr = Module.findExportByName("libc.so", 'strstr');
        var pt_strcmp = Module.findExportByName("libc.so", 'strcmp');
     
        Interceptor.attach(pt_strstr, {
            onEnter: function (args) {
                var str1 = args[0].readCString();
     
                var str2 = args[1].readCString();
                if (str2.indexOf("tmp") !== -1 ||
                    str2.indexOf("frida") !== -1 ||
                    str2.indexOf("gum-js-loop") !== -1 ||
                    str2.indexOf("gmain") !== -1 ||
                    str2.indexOf("gdbus") !== -1 ||
                    str2.indexOf("pool-frida") !== -1||
                    str2.indexOf("linjector") !== -1) {
                    //console.log("strcmp-->", str1, str2);
                    this.hook = true;
                }
            }, onLeave: function (retval) {
                if (this.hook) {
                    retval.replace(0);
                }
            }
        });
     
        Interceptor.attach(pt_strcmp, {
            onEnter: function (args) {
                var str1 = args[0].readCString();
                var str2 = args[1].readCString();
                if (str2.indexOf("tmp") !== -1 ||
                    str2.indexOf("frida") !== -1 ||
                    str2.indexOf("gum-js-loop") !== -1 ||
                    str2.indexOf("gmain") !== -1 ||
                    str2.indexOf("gdbus") !== -1 ||
                    str2.indexOf("pool-frida") !== -1||
                    str2.indexOf("linjector") !== -1) {
                    //console.log("strcmp-->", str1, str2);
                    this.hook = true;
                }
            }, onLeave: function (retval) {
                if (this.hook) {
                    retval.replace(0);
                }
            }
        })
     
    }
     
    replace_str();
    
    ```
    

5、****frida-server 启动****
-------------------------

*   当 frida-server 启动时，在 `/data/local/tmp/`目录下会生成一个文件夹并且会生成一下带有 frida 的特征。
    *   重新编译去特征
    *   hook

6、直接调用 openat 的 syscall 的检测在 text 节表中搜索 frida-gadget.so / frida-agent.so 字符串，避免了 hook libc 来 anti-anti 的方法
----------------------------------------------------------------------------------------------------------

*   注入方式改为
    *   1、frida-inject
    *   2、ptrace 注入 配置文件的形式注入
    *   3、编译 rom 注入

6、从 inlinehook 角度检测 frida
-------------------------

*   [https://bbs.kanxue.com/thread-269862.htm](https://bbs.kanxue.com/thread-269862.htm) 未学习

[https://zhuanlan.zhihu.com/p/557713016](https://zhuanlan.zhihu.com/p/557713016)

7、hook_open 函数可以查看是哪里检测了 so 文件
------------------------------

```
function hook_open(){
    var pth = Module.findExportByName(null,"open");
    Interceptor.attach(ptr(pth),{
        onEnter:function(args){
            this.filename = args[0];
            console.log("",this.filename.readCString())
            if (this.filename.readCString().indexOf(".so") != -1){
                args[0] = ptr(0)
            }
        },onLeave:function(retval){
            return retval;
        }
    })
}
setImmediate(hook_open)

```

替换一个正常的

  

[议题征集启动！看雪 · 第七届安全开发者峰会 10 月 23 日上海](https://bbs.kanxue.com/thread-276136.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm)