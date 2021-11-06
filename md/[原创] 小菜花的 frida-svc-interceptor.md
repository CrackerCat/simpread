> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270144.htm)

> [原创] 小菜花的 frida-svc-interceptor

一、目标
----

现在很多 App 不讲武德了，为了防止 openat 、read、kill 等等底层函数被 hook

1，干脆就直接通过 syscall 的方式来做

2，或者就直接通过内联汇编 svc 的方式去做

最近在研究 svc bypass，想用一个通用点的方案来实现它，继而继续进行 io 重定向。。。，嗯，io 重定向的好处就不说了。

听说的方案有

1，ptrace 直接搞（ps：还在看资料）

2，找到 svc 指令地址，继续去 inlinehook 它

我们今天采用第二种方法，用 frida 来实现  

*   frida momory search svc
    
*   parse insn and count call_number  
    
*   frida native hook svc
    

二、步骤
----

1，参考了葫芦娃 dumpDex 扫描内存的思路

2，frida 是可以直接 hook svc 的，也不用地址减 0x4

3，上 frida 脚本

```
let target_code_hex;
let call_number_openat;
let call_number_faccessat;
let arch = Process.arch;
if ("arm" === arch){
    target_code_hex = "00 00 00 EF";
    call_number_openat = 322;
    call_number_faccessat = 334;
}else if("arm64" === arch){
    target_code_hex = "01 00 00 D4";
    call_number_openat = 56;
    call_number_faccessat = 48;
}else {
    console.log("arch not support!")
}
 
if (arch){
    console.log("\nthe_arch = " + arch);
    // 直接Process.enumerateModules()，可能会因为某些地址不可读造成非法访问
    Process.enumerateRanges('r--').forEach(function (range) {
        if(!range.file || !range.file.path){
            return;
        }
        let path = range.file.path;
        if ((!path.startsWith("/data/app/")) || (!path.endsWith(".so"))){
            return;
        }
        let baseAddress = Module.getBaseAddress(path);
        console.log("\npath = " + path + " , baseAddress = " + baseAddress + " , rangeAddress = " + range.base + " , size = " + range.size);
 
        Memory.scan(range.base, range.size, target_code_hex, {
            onMatch: function (match){
                let code_address = match;
                let code_address_str = code_address.toString();
                if (code_address_str.endsWith("0") || code_address_str.endsWith("4") || code_address_str.endsWith("8") || code_address_str.endsWith("c")){
                    console.log("--------------------------");
                    let call_number = 0;
                    if ("arm" === arch){
                        // call_number = (code_address.sub(0x4).readS16() - 28672);  // 0x7000
                        call_number = (code_address.sub(0x4).readS32()) & 0xFFF;
                    }else if("arm64" === arch){
                        call_number = (code_address.sub(0x4).readS32() >> 5) & 0xFFFF;
                    }else {
                        console.log("the arch get call_number not support!")
                    }
                    console.log("find svc : address = " + code_address + " , call_number = " + call_number + " , offset = " + code_address.sub(baseAddress));
 
                    // hook svc __NR_openat
                    if (call_number_openat === call_number){
                        let target_hook_addr = code_address;
                        let target_hook_addr_offset = target_hook_addr.sub(baseAddress);
                        console.log("find svc openat , start inlinehook by frida!")
                        Interceptor.attach(target_hook_addr, {
                            onEnter: function (args){
                                console.log("\nonEnter_" + target_hook_addr_offset + " , __NR_openat , args[1] = " + args[1].readCString());
                                this.new_addr = Memory.allocUtf8String("/proc/self/status11");
                                args[1] = this.new_addr;
                                console.log("onEnter_" + target_hook_addr_offset + " , __NR_openat , args[1] = " + args[1].readCString());
                            }, onLeave: function (retval){
                                console.log("onLeave_" + target_hook_addr_offset + " , __NR_openat , retval = " + retval)
                            }
                        });
 
                    }
                    // hook svc __NR_faccessat
                    if (call_number_faccessat === call_number){
                        let target_hook_addr = code_address;
                        let target_hook_addr_offset = target_hook_addr.sub(baseAddress);
                        console.log("find svc faccessat , start inlinehook by frida!")
                        Interceptor.attach(target_hook_addr, {
                            onEnter: function (args){
                                console.log("\nonEnter_" + target_hook_addr_offset + " , __NR_faccessat , args[1] = " + args[1].readCString());
                                // this.new_addr = Memory.allocUtf8String("/proc/self/status11");
                                // args[1] = this.new_addr;
                                console.log("onEnter_" + target_hook_addr_offset + " , __NR_faccessat , args[1] = " + args[1].readCString());
                            }, onLeave: function (retval){
                                console.log("onLeave_" + target_hook_addr_offset + " , __NR_faccessat , retval = " + retval)
                            }
                        });
 
                    }
                }
            }, onComplete: function () {}
        });
 
    });
}

```

三、总结
----

0， 这里样本是 mov x8(r7) 调用号 和 svc 0 紧挨着的情况，也可以持续向上访问地址查找目标寄存器指令适配的更好一点  

1， 这里只对目标 app 的 so 进行了内存扫描，同理也可以对系统库进行内存扫描  

2，用了同步的 Memory.scan 扫描方式，为了区分到底是哪个目标 so 文件

3，这里只是 frida attach 脚本，同理可以改成 frida spawn 方式，选个好的注入时机就好了

4，这是只搞了两个系统调用号，同理可以多搞一点

5，落地成 Dobby 实现，如果直接 DobbyHook svc 地址的话，进入 new_func 一切正常，进 org_func 的时候就开始闪退了，那是因为 x8（r7）寄存器的值变了，导致系统调用号没了，进而闪退。此时可以 svc address - 0x4 去 hook 也是可以的，当然前提是 mov 和 svc 指令紧挨着

6，感觉稳定的生产环境之一应该是用 riru 模块来搞  
7，当然此方案有一定局限性，比如 svc 在扫描的内存之外，inlinehook 特征等等

8,  git 地址：[https://github.com/huaerxiela/frida-script](https://github.com/huaerxiela/frida-script)

9，很简单的思路，好像也是 va 核心 io 的思路。。。

四、参考文献
------

[https://bbs.pediy.com/thread-269895.htm](https://bbs.pediy.com/thread-269895.htm) 批量检测 android app 的 so 中是否有 svc 调用 

反调试及绕过 [https://jmpews.github.io/2017/08/09/darwin/%E5%8F%8D%E8%B0%83%E8%AF%95%E5%8F%8A%E7%BB%95%E8%BF%87/](https://jmpews.github.io/2017/08/09/darwin/%E5%8F%8D%E8%B0%83%E8%AF%95%E5%8F%8A%E7%BB%95%E8%BF%87/)

bypass svc 0 PTRACE_TRACEME [https://bbs.pediy.com/thread-212404.htm](https://bbs.pediy.com/thread-212404.htm)

[http://91fans.com.cn/post/fridasyscallinterceptor/](http://91fans.com.cn/post/fridasyscallinterceptor/)【奋飞】Frida-syscall-interceptor

[https://bbs.pediy.com/thread-268283.htm](https://bbs.pediy.com/thread-268283.htm) [原创]iOS 反反调试助手

[https://bbs.pediy.com/thread-260731.htm](https://bbs.pediy.com/thread-260731.htm) [原创] 使用 ptrace 过 ptrace 反调试

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 4 小时前 被 huaerxiela 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm)