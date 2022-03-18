> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271921.htm)

> [原创] 基于 ptrace 的 Android 系统调用跟踪 & hook 工具

前言
==

大概一年前写的吧，arm64 版本已开源：https://github.com/onesss19/Syscall_intercept_arm64（求 star

 

hook svc 的方案已经有好几种，前几天有个大佬开源的 Frida-Seccomp，罗哥开源的 krhook，内存搜索 + inlinehook，还有一些大佬没开源的核武器

 

这个工具是基于 ptrace 实现的，开发涉及到的关键 API 都是直接参考官方文档 https://man7.org/linux/man-pages/man2/ptrace.2.html

使用
==

```
void show_helper(){
    printf(
            "\nSyscall_intercept -z -n -p \n"
            "options:\n"
            "\t-z : pid of zygote\n"
            "\t-t : application name\n"
            "\t-p : pid of application\n"
    );
} 
```

支持 spawn 模式和 attach 模式

spawn 模式
--------

```
Syscall_intercept -z zygote_pid -n package_name

```

运行上述指令后手动打开目标 app

attach 模式
---------

```
Syscall_intercept -p target_pid

```

打开目标 app 后运行上述指令

原理
==

主要就是依赖于 ptrace 的这个参数：

```
// the tracee to be stopped at the next entry to or exit from a system call
ptrace(PTRACE_SYSCALL, wait_pid, 0, 0);

```

spawn 模式的原理是 ptrace 到 zygote 进程，然后跟踪 zygote 进程的 fork 系统调用，如果 fork 出来的新进程是指定包名的 app，那么 detach 掉 zygote 进程，进而跟踪目标 app 进程的系统调用

 

attach 模式的原理是直接 ptrace 目标 app 进程的所有线程

功能
==

大体功能和 strace 类似，实现原理也是一样的，主要是多了 hook 的能力

 

起初是想在 strace 的基础上改，源码框架没看太懂，转而自己写了个小玩具（逃

 

以拦截 openat 系统调用为例，运行结果：

 

![](https://bbs.pediy.com/upload/attach/202203/882371_73YTTDCPDXKK6AJ.png)

 

对应源码：

```
void openat_item(pid_t pid,user_pt_regs regs){
    char        filename[256];
    char        path[256];
    uint32_t    filenamelength=0;
 
    get_addr_path(pid,regs.ARM_x1,path);
    if(strstr(path,"/data/app")!=0 || strstr(path,"[anon:libc_malloc]")!=0){
        getdata(pid,regs.ARM_x1,filename,256);
        if(strcmp(filename,"/dev/ashmem")!=0){
            print_register_enter(regs,pid,(char*)"__NR_openat",regs.ARM_x8);
            printf("filename: %s\n",filename);
            printf("path: %s\n",path);
            if(strcmp(filename,"/proc/sys/kernel/random/boot_id")==0){
                char tmp[256]="/data/local/tmp/boot_id";
                filenamelength=strlen(tmp)+1;
                putdata(pid,regs.ARM_x1,tmp,filenamelength);
                getdata(pid,regs.ARM_x1,filename,256);
                printf("changed filename: %s\n",filename);
            }
        }
    }
}

```

编译
==

```
clang++ -target aarch64-linux-android21 Syscall_intercept_arm64.cpp Syscall_item_enter_arm64.cpp -o Syscall_intercept_arm64 -static-libstdc++

```

用 ndk 里面自带的 clang++ 编译即可

TODO
====

只是一个能跑的玩具，主要是把思路抛出来，后续可以适配更多的系统调用，可以添加栈回溯等等功能~

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

[#工具脚本](forum-161-1-128.htm) [#系统相关](forum-161-1-126.htm)