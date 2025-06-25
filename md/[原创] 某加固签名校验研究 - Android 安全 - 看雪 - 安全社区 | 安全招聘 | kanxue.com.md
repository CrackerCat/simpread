> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287338.htm)

> [原创] 某加固签名校验研究

闲言碎语
====

> 之前我们讨论了去除签名校验的方法，无法去除最新的某 60 加固，始终是我心中的大山。我心有不甘，想看看这加固的签名校验的强度，去体会移动安全开发工程师在代码里书写的智慧与浪漫。于是就有了这篇文章。

基础知识的文章请看之前的文章 --→[https://bbs.kanxue.com/thread-285647.htm](https://bbs.kanxue.com/thread-285647.htm)

0x1 分析前的准备
==========

我认为良好的环境是分析提高效率的关键

在频繁的使用 ida 对 app 进行分析调试，经常会打开，可以编写一份脚本快速打开 ida 的 server

```
#!/bin/bash
# 设置端口转发
if ! adb forward tcp:4567 tcp:4567; then
    echo "[错误] 端口转发失败"
    exit 2
fi
 
echo "IDA服务已在端口4567启动"
# 启动IDA服务
if ! adb shell "su -c 'cd /data/local/tmp && ./ida9.1 -p4567 &'"; then
    echo "[错误] 无法启动IDA服务"
    exit 1
fi
 
# 等待服务初始化
echo "ok..."
```

这里面的 ida9.1 是你的 ida server。还有 frida 快速启动脚本

```
#!/bin/bash
adb shell "su -c 'cd /data/local/tmp && ./hluda-server-16.2.1-android-arm64'"
```

将 “hluda-server-16.2.1-android-arm64” 修改为 你自己的的 server 就好了

这里我分析的环境是 piexl 5,Android 12 的版本。

样本是 6 月最新的加固。

准备好你的 ida ， frida 基本环境，与我一起分析吧！

0x2 so 脱壳
=========

![](https://bbs.kanxue.com/upload/attach/202506/957038_QU8VXZPU29BS5XG.webp)

现在最新的加固样本的 so 文件是加密状态，我们要先进行初步的 dump 获取基本的符号信息，现在的符号信息空空如也。

这里使用 frida dump 再修复

```
function dump_so(so_name) {
    var libso = Process.getModuleByName(so_name);
    console.log("[name]:", libso.name);
    console.log("[base]:", libso.base);
    console.log("[size]:", ptr(libso.size));
    console.log("[path]:", libso.path);
    var file_path = "/data/data/"+get_self_process_name()+"/" + libso.name + "_" + libso.base + "_" + ptr(libso.size) + ".so";
    var file_handle = new File(file_path, "wb");
    if (file_handle && file_handle != null) {
        Memory.protect(ptr(libso.base), libso.size, 'rwx');
        var libso_buffer = ptr(libso.base).readByteArray(libso.size);
        file_handle.write(libso_buffer);
        file_handle.flush();
        file_handle.close();
        console.log("[dump]:", file_path);
    }
}
  
function get_self_process_name() {
    var openPtr = Module.getExportByName('libc.so', 'open');
    var open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
 
    var readPtr = Module.getExportByName("libc.so", "read");
    var read = new NativeFunction(readPtr, "int", ["int", "pointer", "int"]);
 
    var closePtr = Module.getExportByName('libc.so', 'close');
    var close = new NativeFunction(closePtr, 'int', ['int']);
 
    var path = Memory.allocUtf8String("/proc/self/cmdline");
    var fd = open(path, 0);
    if (fd != -1) {
        var buffer = Memory.alloc(0x1000);
 
        var result = read(fd, buffer, 0x1000);
        close(fd);
        result = ptr(buffer).readCString();
        return result;
    }
 
    return "-1";
}
 
 
 
//8.0以下所有的so加载都通过dlopen
function hook_dlopen() {
    var dlopen = Module.findExportByName(null, "dlopen");
    Interceptor.attach(dlopen, {
        onEnter: function (args) {
            this.call_hook = false;
            var so_name = ptr(args[0]).readCString();
            if (so_name.indexOf("libshell-super.2019.so") >= 0) {
                console.log("dlopen:", ptr(args[0]).readCString());
                this.call_hook = true;//dlopen函数找到了
            }
 
        }, onLeave: function (retval) {
            if (this.call_hook) {//dlopen函数找到了就hook so
                inline_hook();
            }
        }
    });
    // 高版本Android系统使用android_dlopen_ext
    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            this.call_hook = false;
            var so_name = ptr(args[0]).readCString();
            if (so_name.indexOf("libshell-super.2019.so") >= 0) {
                console.log("android_dlopen_ext:", ptr(args[0]).readCString());
                this.call_hook = true;
            }
 
        }, onLeave: function (retval) {
            if (this.call_hook) {
                inline_hook();
            }
        }
    });
}
function main (){
    Java.perform(
        function(){
        var tart_Name = "libjiagu_64.so"
        dump_so(tart_Name)
        }
    );
}
setImmediate(main)
```

再使用 SoFixer64 进行修复

[https://github.com/F8LEFT/SoFixer/releases](elink@dbcK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6r3z5p5I4q4c8W2c8Q4x3V1k6e0L8@1k6A6P5r3g2J5i4K6u0r3M7X3g2D9k6h3q4K6k6i4x3`.)

这样就具备了基本的 so 信息符号了

![](https://bbs.kanxue.com/upload/attach/202506/957038_EDHGDUMY643S8UE.webp)

在最新的加固样本里面 外壳的 so 似乎都用的 c++ 代码编译了，对抗了之前所有的文章的地方

按照之前的文章和方法几乎 无法 将 内部 so 进行 dump 。

这里我再想 实现 linker 的步骤里面是无法脱离 dlopen 这个函数的，我尝试搜索相关的引用，发现并不是 linker 的预连接逻辑

而这个方法里面调用了线程相关的函数

pthread_rwlock_rdlock

那么我就要大胆的猜测，它是不是通过线程来进行通信，来实现 linker 的！

我引用信号相关的函数 kill

发现了之前 预连接的方法里面了？ 是不是很巧妙？（其实是千疮百孔的调试分析得到的！）

![](https://bbs.kanxue.com/upload/attach/202506/957038_VJY4JWSYHM52J92.webp)

那就使用我之前文章里面的脱壳脚本，进行脱壳

```
function my_hook_dlopen(soName = '') {
 
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
                    hook();
                }
            }
        }
    );
}
function addr_in_so(addr){
    var process_Obj_Module_Arr = Process.enumerateModules();
    for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
        if(addr>process_Obj_Module_Arr[i].base && addr
```

然后 按照我之前的文章内容 修复就可以了

[https://bbs.kanxue.com/thread-285539.htm](https://bbs.kanxue.com/thread-285539.htm)

0x4 遇到的拦路虎
==========

### SVC 指令的自实现

加固样本 里面普遍使用 自己实现的 svc 指令来进行 系统函数的调用。

比如： openat 调用

它的实现方式很有趣: 先 mmap 一块可运行的内存，在把加密的 svc 代码解密放进去，再调用函数

这样做的好处是 直接断绝了——通过内存扫描的形式，对 svc 指令的 hook!

### 签名校验

在 java 层

1.  对包路径 进行两次不同的方法进行获取
    
2.  调用了常规的签名信息，进行 hashcode
    

java 层的强度 似乎不太高，我建议构造 binder 通讯 ams

native 的强度还是可以的！

1.  svc openat 打开包
    
2.  readklinkat 读取包路径
    
3.  fstat 读取包的权限
    

都是以 svc 的方式实现的

### 反调试

这里反调试是我最大的困惑，也是最精彩的地方。

它是怎样实现反调试的呢？

1.  经典 status 文件里面的 trace pid 的值 和 state 进程状态
    
2.  在 status 里面 还检测了 NoNewPrivs ， 这个值很有意思！ 你可以百度或者 ai 问一下！
    
3.  通过 fork 子进程 来进行 测试 ptrace ，wait 来校验 进程信号是否被 调试
    
4.  fork 的子进程与父进程通过管道 通讯 ，子进程调用 excel 函数 cat /proc/self/status , 观察 进程名 是否正确？
    
5.  这里还对 /proc 的文件进行 mmap 查看它的 error 值 ，如果我们替换了 就会成功 ！ error 的值为 0
    

0x5 过关斩将
========

### 如何实现 java 的 hook

我使用的 lspant 的 hook 来对 java 进行 hook ，hook 点在我上一篇文章有提到。

### 如何实现 svc 的 hook

这里我有两种方案 一种是

```
基于 bpf + sigation 的方式
```

优点： hook 比较简单就能实现， 也是大多数 多开 app 使用方案

缺点： 稳定性差，遇到基于信号的检测就完了！还有就是对于低版本的内核 有很多功能实现不了！

```
基于bpf + ptrace
```

优点： ptrace hook 的稳定性 对比上一个好，而且有很好的项目可以学习 proot , 有兴趣可以看我之前的 proot 分析

缺点： 必须处理反调试检测

我的选择是 ptrace 的方案，这里有很多资料可以学习。

这里的方案 来自 proot 和 abyss

[https://github.com/proot-me/proot](elink@e6bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6H3M7X3!0G2N6q4)9J5k6r3#2W2i4K6u0r3M7s2u0G2L8%4b7`.)

[https://github.com/iofomo/abyss](elink@2faK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6A6L8$3k6G2L8h3!0Q4x3V1k6S2j5Y4W2K6M7H3`.`.)

### java 层签名对抗

包路径 这里的替换，这里可以参考插件化开发技术！

替换 loadapk 里面 application 的 sourice 的值就可以了

### native 层签名对抗

使用 io 重定向 修改读取的 apk 为原来没有签名打包的 apk.

readlink ，fstat 也是修改替换返回值. 达到欺骗的效果。

### 反调试对抗

也是对 status stat 这几个目录进行 io 重定向 主要就是修改 一些文件的值

```
void generate_fake_status(char* status_path, const char* output_path,Tracee *tra) {
    // 新增：检查并删除目标文件
    if (access(output_path, F_OK) == 0) {
        if (remove(output_path) != 0) {  // 删除失败处理
            LOGE("generate_fake_status Failed to delete existing file");
            return;
        }
    }
 
    FILE* src = fopen(status_path, "r");
    FILE* dst = fopen(output_path, "w");
    if (!src || !dst) {
        LOGE("generate_fake_status Failed to open file");
        if (src) fclose(src);
        if (dst) fclose(dst);
        return;
    }
 
    char process_name[256];
    memset(process_name, 0, sizeof(process_name));
    get_pid_proceess_Name(tra->pid, process_name);
 
    if(strlen(process_name)>0){
        LOGE("generate_fake_status process_name: %s", process_name);
    }
 
 
    char line[256];
    while (fgets(line, sizeof(line), src)) {
        /* 关键字段修改 */
        // 1. 清除调试痕迹
        if (strstr(line, "TracerPid:") != NULL) {
            snprintf(line, sizeof(line), "TracerPid:\t0\n");
        }
        if (strstr(line, "Name:") != NULL) {
            if(strlen(process_name)>0){
                LOGE("修改 process_name: %s", process_name);
                snprintf(line, sizeof(line), "Name:\t%s\n",process_name);
            }
        }
 
        //PPid
        if (strstr(line, "PPid:") != NULL) {
            snprintf(line, sizeof(line), "PPid:\t%d\n",PPID);
        }
            // 2. 修正线程状态
         if (strstr(line, "State:") != NULL) {
            snprintf(line, sizeof(line), "State:\tS (sleeping)\n"); // 比R更常见
        }
            // 3. 修复信号掩码
         if (strstr(line, "SigIgn:") != NULL) {
            snprintf(line, sizeof(line), "SigIgn:\t0000000000000001\n");
        }
         if (strstr(line, "SigCgt:") != NULL) {
//            snprintf(line, sizeof(line), "SigCgt:\t0000000000a00000\n"); // 移除非正常信号
        }
            // 4. 隐藏seccomp保护
         if (strstr(line, "Seccomp:") != NULL) {
//            snprintf(line, sizeof(line), "Seccomp:\t0\n");
        }
            // 5. 修正上下文切换次数
         if (strstr(line, "nonvoluntary_ctxt_switches:") != NULL) {
            snprintf(line, sizeof(line), "nonvoluntary_ctxt_switches:\t12\n"); // 典型低值
        }
            // 6. 移除Speculation漏洞标记
         if (strstr(line, "Speculation_Store_Bypass:") != NULL) {
            snprintf(line, sizeof(line), "Speculation_Store_Bypass:\tthread vulnerable\n");
        }
         if (strstr(line, "NoNewPrivs") != NULL) {
 
//             if(strlen(process_name)>0  && strcmp(process_name, "cat") ==0){
//                 LOGE("修改 process_name: %s", process_name);
//                 snprintf(line, sizeof(line), "NoNewPrivs:\t1\n");
//             }else{
                 snprintf(line, sizeof(line), "NoNewPrivs:\t0\n");
//             }
 
        }
        fputs(line, dst);
    }
 
    fclose(src);
    fclose(dst);
}
```

然后将 ptrace 的值 返回为 0 ，

就基本完成了 对抗！ 呼出一口气。

最后，我要感谢

空壳的作者 [Tiony.bo](https://bbs.kanxue.com/user-home-909890.htm) 也是我的前辈，感谢你对我插件化开发的指点，以及 abyss 的开源！

感谢 [珍惜 Any](elink@71cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6H3j5i4y4K6M7r3!0J5N6q4)9J5k6h3E0S2L8Y4S2#2k6g2)9J5k6h3y4G2L8g2)9J5c8Y4g2K6k6i4u0Q4x3X3c8U0k6h3&6@1k6i4u0Q4x3X3b7^5x3e0V1&6x3K6c8Q4x3X3g2Z5N6r3@1`.)，[王麻子本人](elink@e2cK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6H3j5i4y4K6M7r3!0J5N6q4)9J5k6h3E0S2L8Y4S2#2k6g2)9J5k6h3y4G2L8g2)9J5c8Y4g2K6k6i4u0Q4x3X3c8U0k6h3&6@1k6i4u0Q4x3X3b7&6x3U0R3H3y4K6W2Q4x3X3g2Z5N6r3@1`.) 等等 大佬关于 ptrace 的文章 让我有所启蒙！

参考的链接：

[https://bbs.kanxue.com/thread-285339.htm](https://bbs.kanxue.com/thread-285339.htm)

[https://bbs.kanxue.com/thread-273160.htm](https://bbs.kanxue.com/thread-273160.htm)

[https://bbs.kanxue.com/thread-275511.htm](https://bbs.kanxue.com/thread-275511.htm)

[https://github.com/iofomo/abyss](elink@513K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6A6L8$3k6G2L8h3!0Q4x3V1k6S2j5Y4W2K6M7H3`.`.)

[https://github.com/proot-me/proot](elink@cc9K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6H3M7X3!0G2N6q4)9J5k6r3#2W2i4K6u0r3M7s2u0G2L8%4b7`.)

[[培训] 科锐逆向工程师培训第 53 期 2025 年 7 月 8 日开班！](https://bbs.kanxue.com/thread-51839.htm)

最后于 20 小时前 被逆天而行编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#脱壳反混淆](forum-161-1-122.htm) [#HOOK 注入](forum-161-1-125.htm)

上传的附件：

*   [360 加固. apk](javascript:void(0);) （8.05MB，42 次下载）
*   [360 去签名. apk](javascript:void(0);) （8.15MB，51 次下载）