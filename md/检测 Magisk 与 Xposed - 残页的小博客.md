> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2021/05/01/anti-magisk-xposed/)

不久前，开发者 Rikka & vvb2060 上架了一款环境检测应用 [Momo](https://www.coolapk.com/apk/io.github.vvb2060.mahoshojo)，把大家一直以来信任的各种反检测手段击得粉碎。下面我会通过部分已公开的源码，分析这个可能是史上最强的环境检测应用。

[](#检测-Magisk "检测 Magisk")检测 Magisk
-----------------------------------

这一部分只分析有趣的东西，关于 MagiskDetector 的其他一些具体实现细节请直接查看 [https://github.com/vvb2060/MagiskDetector/blob/master/README_ZH.md](https://github.com/vvb2060/MagiskDetector/blob/master/README_ZH.md)

### [](#反Magisk-Hide "反Magisk Hide")反 Magisk Hide

首先分析 Magisk Hide 的原理：

```
static void new_zygote(int pid) {
    struct stat st;
    if (read_ns(pid, &st))
        return;

    auto it = zygote_map.find(pid);
    if (it != zygote_map.end()) {
        
        it->second = st;
        return;
    }

    LOGD("proc_monitor: ptrace zygote PID=[%d]\n", pid);
    zygote_map[pid] = st;

    xptrace(PTRACE_ATTACH, pid);

    waitpid(pid, nullptr, __WALL | __WNOTHREAD);
    xptrace(PTRACE_SETOPTIONS, pid, nullptr,
            PTRACE_O_TRACEFORK | PTRACE_O_TRACEVFORK | PTRACE_O_TRACEEXIT);
    xptrace(PTRACE_CONT, pid);
}

void proc_monitor() {
    

    
    check_zygote();

    for (int status;;) {
        const int pid = waitpid(-1, &status, __WALL | __WNOTHREAD);
        if (pid < 0) {
            
        }

        if (!WIFSTOPPED(status) )
            DETACH_AND_CONT;

        int event = WEVENT(status);
        int signal = WSTOPSIG(status);

        if (signal == SIGTRAP && event) {
            unsigned long msg;
            xptrace(PTRACE_GETEVENTMSG, pid, nullptr, &msg);
            if (zygote_map.count(pid)) {
                
                switch (event) {
                    case PTRACE_EVENT_FORK:
                    case PTRACE_EVENT_VFORK:
                        PTRACE_LOG("zygote forked: [%lu]\n", msg);
                        attaches[msg] = true;
                        break;
                    
                }
            } else {
                switch (event) {
                    case PTRACE_EVENT_CLONE:
                        PTRACE_LOG("create new threads: [%lu]\n", msg);
                        if (attaches[pid] && check_pid(pid)) 
                            continue;
                        break;
                    
                }
            }
            xptrace(PTRACE_CONT, pid);
        } else if (signal == SIGSTOP) {
            if (!attaches[pid]) {
                
                attaches[pid] = is_process(pid);
            }
            if (attaches[pid]) {
                
                PTRACE_LOG("SIGSTOP from child\n");
                xptrace(PTRACE_SETOPTIONS, pid, nullptr,
                        PTRACE_O_TRACECLONE | PTRACE_O_TRACEEXEC | PTRACE_O_TRACEEXIT);
                xptrace(PTRACE_CONT, pid);
            } 
        } 
    }
}
```

可以看见，magisk hide 通过 ptrace 机制跟踪所有 zygote，通过`cat /proc/<pid>/status`看见 TracerPid 也可以证实我们的发现。子进程的第一个线程创建时，对其实际进行 hide。

```
static bool check_pid(int pid) {
    char path[128];
    char cmdline[1024];
    struct stat st;

    sprintf(path, "/proc/%d/cmdline", pid);
    if (auto f = open_file(path, "re")) {
        fgets(cmdline, sizeof(cmdline), f.get());
    } else {
        
        detach_pid(pid);
        return true;
    }
    
    if (cmdline == "zygote"sv || cmdline == "zygote32"sv || cmdline == "zygote64"sv ||
        cmdline == "usap32"sv || cmdline == "usap64"sv)
        return false;

    
    if (!is_hide_target(uid, cmdline))
        goto not_target;

    
    read_ns(pid, &st);
    for (auto &zit : zygote_map) {
        if (zit.second.st_ino == st.st_ino &&
            zit.second.st_dev == st.st_dev) {
            
            LOGW("proc_monitor: skip [%s] PID=[%d] UID=[%d]\n", cmdline, pid, uid);
            goto not_target;
        }
    }

    
    
    LOGI("proc_monitor: [%s] PID=[%d] UID=[%d]\n", cmdline, pid, uid);
    detach_pid(pid, SIGSTOP);
    hide_daemon(pid);
    return true;

not_target:
    PTRACE_LOG("[%s] is not our target\n", cmdline);
    detach_pid(pid);
    return true;
}
```

hide_daemon 里会 fork 一个新的进程，setns 到目标进程的命名空间，然后卸载所有被 magisk 修改过的东西。注意，里面有个 if 判断，如果命名空间未分离，进行 unmount 会影响到 zygote 从而影响到而后启动的所有进程，那么直接跳过。这就是 Magisk Hide 的第一个问题。

在 MagiskDetector 的[实现细节介绍](https://github.com/vvb2060/MagiskDetector/blob/master/README_ZH.md#%E5%8F%8Dmagisk-hide)里说明了有两种情况符合：

> 一个是应用 appops 的读取存储空间 op 为忽略，一个是该进程为隔离进程。

此处的隔离进程指的就是配置了 `android:isolatedProcess="true"`的 service。而且，Android 10 上还有一种有（私）趣（货）的东西，叫做 App Zygote，这玩意几乎找不到说明，唯一的文档就是 [ZygotePreload](https://developer.android.google.cn/reference/android/app/ZygotePreload)，感觉更像谷歌给 Chrome 开的后门。咳咳，偏题了，这玩意运行在一个单独的进程，也不会分离命名空间。

目前已知解决此问题的方案有两种，第一种就是 [Magisk Lite](https://github.com/vvb2060/Magisk/tree/lite)，直接对 zygote 卸载而非应用，但这种方式会破坏很多现有模块；另一种就是利用进程注入，强行分离命名空间，典型的解决方案是 [Riru-Unshare](https://github.com/vvb2060/riru-unshare)。

好的，这个问题说完了，下一个~~

在上面的判断代码里，读取进程名部分，是通过读取`/proc/<pid>/cmdline`进行判断的；而实际上，这个文件内容的长度是有限制的！这表示，当配置的进程名过长时，Magisk 读取到的进程名会不匹配，从而跳过这个进程！这也就是 [Issue #3997](https://github.com/topjohnwu/Magisk/issues/3997) 的原理。Magisk 对此做了临时修复：如果前缀匹配就直接认为是目标进程进行 hide。

完了吗？没有。下一个问题在把进程添加到数据库的时候：

```
static int add_list(const char *pkg, const char *proc) {
    if (proc[0] == '\0')
        proc = pkg;

    if (!validate(pkg) || !validate(proc))
        return HIDE_INVALID_PKG;
    
    
}

static bool validate(const char *s) {
    if (strcmp(s, ISOLATED_MAGIC) == 0)
        return true;
    bool dot = false;
    for (char c; (c = *s); ++s) {
        if ((c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z') ||
            (c >= '0' && c <= '9') || c == '_' || c == ':') {
            continue;
        }
        if (c == '.') {
            dot = true;
            continue;
        }
        return false;
    }
    return dot;
}
```

这里会对包名和进程名进行检查，如果含有非法字符或者没有点，那么认为是无效进程。Android 对包名有严格规定，通过`android:process`配置的进程名也有规定，似乎无法作妖？然而问题确实发生了：[Issue #4176](https://github.com/topjohnwu/Magisk/issues/4176)。

经过检查，该应用程序使用了隔离进程来检查 Magisk，但不同的是，其服务类名含有非法字符（Java 并没有限制类名），且 Android 10+，系统会给隔离进程的名字追加类名（[https://t.me/vvb2060Channel/441），导致检查不通过。解决方法也很简单，修改一下这个 validate 就好。](https://t.me/vvb2060Channel/441%EF%BC%89%EF%BC%8C%E5%AF%BC%E8%87%B4%E6%A3%80%E6%9F%A5%E4%B8%8D%E9%80%9A%E8%BF%87%E3%80%82%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95%E4%B9%9F%E5%BE%88%E7%AE%80%E5%8D%95%EF%BC%8C%E4%BF%AE%E6%94%B9%E4%B8%80%E4%B8%8B%E8%BF%99%E4%B8%AAvalidate%E5%B0%B1%E5%A5%BD%E3%80%82)

### [](#检测init-rc的修改： "检测init.rc的修改：")检测 init.rc 的修改：

> 随机只有在无法遍历的情况下才有效。如果可以遍历，使用统计方法即可准确找出每次都不一样的东西。

这句话看着有点迷惑，看看 Magisk 源码就知道了。[https://github.com/topjohnwu/Magisk/blob/master/native/jni/init/rootdir.cpp#L45：](https://github.com/topjohnwu/Magisk/blob/master/native/jni/init/rootdir.cpp#L45%EF%BC%9A)

```
char pfd_svc[16], ls_svc[16], bc_svc[16];
gen_rand_str(pfd_svc, sizeof(pfd_svc));
gen_rand_str(ls_svc, sizeof(ls_svc));
gen_rand_str(bc_svc, sizeof(bc_svc));
LOGD("Inject magisk services: [%s] [%s] [%s]\n", pfd_svc, ls_svc, bc_svc);
fprintf(rc, MAGISK_RC, tmp_dir, pfd_svc, ls_svc, bc_svc);
```

Magisk 在启动时会往 init.rc 中注入三个自己的服务，用来接收 post-fs-data 等事件；这三个服务的名称是做了随机化处理，而 init 实际上会往系统属性里添加像`init.svc.<service name>`这样子的属性，值是 running 或者 stopped，以告诉其他进程该服务的状态。MagiskDetector 就利用了这个机制，遍历系统属性记录所有服务名，然后在用户重启之后就能知道是否有服务的名称发生了变化。

### [](#检测SELinux规则 "检测SELinux规则")检测 SELinux 规则

[https://github.com/topjohnwu/Magisk/blob/master/native/jni/magiskpolicy/rules.cpp](https://github.com/topjohnwu/Magisk/blob/master/native/jni/magiskpolicy/rules.cpp)

```
const char *clients[] { "init", "shell", "appdomain", "zygote" };
for (auto type : clients) {
    if (!exists(type))
        continue;
    allow(type, SEPOL_PROC_DOMAIN, "unix_stream_socket", "connectto");
    allow(type, SEPOL_PROC_DOMAIN, "unix_stream_socket", "getopt");

    
    const char *pts[] { "devpts", "untrusted_app_devpts" };
    for (auto pts_type : pts) {
        allow(type, pts_type, "chr_file", "ioctl");
        if (db->policyvers >= POLICYDB_VERSION_XPERMS_IOCTL)
            allowxperm(type, pts_type, "chr_file", "0x5400-0x54FF");
    }
}
```

由于 Magisk 允许了一些 ioctl，所以会被检测到。解决方法是，更新到 Android 8+ & Magisk 21+，Magisk 会自动使用新的规则。

同时，不仅仅是 Magisk 自身的锅，错误使用 SELinux 也可能会造成 Magisk 被轻易检测到。举例：

```
type(SEPOL_PROC_DOMAIN, "domain");
type(SEPOL_FILE_TYPE, "file_type");
```

加了两个 magisk 自己的 domain，看起来没问题，但是，如果用户把 selinux 设置为宽容模式（permissive），那么 app 可以进行`selinux_check_access()`（java 层对应的接口为`SELinux.checkSELinuxAccess()`），如果获得允许，那么代表这个 domain 存在 => 安装了 Magisk。

不只是宽容模式，如果添加了`allow appdomain xxx relabelfrom`之类的规则，又没有`deny appdomain magisk_file relabelto`，则 app 可能把某个文件的 context 给 chcon 成`magisk_file`，然后通过尝试操作这个文件判断有没有被拒绝就可以测试出系统中有没有这个 domain。

**SELinux 是 Android 安全机制中的重要组成部分，强烈反对将其设置为宽容模式或忽略 neverallow 随意添加规则。**

### [](#题外话：检测-magiskd "题外话：检测 magiskd")题外话：检测 magiskd

虽然 MagiskDetector 没有使用这个方法，但觉得有点意思，可以拿出来讲一讲。  
Android 7 之前，`/proc`没有限制，任何人都能遍历获得进程列表；在 7 的时候，加了`hidepid=2`，但并不是所有厂商都跟上了；对于这些设备，扫一下看看有没有个叫`magiskd`的进程就能确定有没有 magisk。

[](#Xposed "Xposed")Xposed
--------------------------

### [](#检测-Xposed "检测 Xposed")检测 Xposed

原版 Xposed 框架将自己的类加入到 bootclasspath 中，这导致任何人都能轻易找到。之后，大家都选择把 classloader 隔离开来，让检测没有那么容易；但是，只要它存在于内存中，那就可以被找到。XposedDetector 的原理很简单，通过 art 的一个内部接口（VisitRoots），找到堆里的所有 ClassLoader，然后一个一个尝试。目前 lsp、edxp、梦境等都选择只在目标应用加载，以阻止误伤。把这个函数 hook 了当然可以，但我们并不想玩这种猫鼠游戏，只能保证非目标应用的环境不被修改。

### [](#反-Xposed-Hook "反 Xposed Hook")反 Xposed Hook

XposedDetector 的做法是，通过上面的方法，可以找到当前进程里的所有类，依据此法找到 XposedBridge 把 `disableHooks` 和 `sHookedMethodCallbacks` 改掉就可以。

实际上，还有很多其他方法：  
除了原版 xposed 和 0.5 之前的 edxposed，其他框架基本都直接忽略隔离进程，可以把重要的东西放在隔离进程。

通过 Xposed hook 一个方法，最终都会走到这个方法：

```
public static XC_MethodHook.Unhook hookMethod(Member hookMethod, XC_MethodHook callback) {
    
    else if (hookMethod.getDeclaringClass().isInterface()) {
		throw new IllegalArgumentException("Cannot hook interfaces: " + hookMethod.toString());
	}
}
```

这段检查在 Android 7 之后是有问题的，因为 Android 7 支持了一个 Java 8 特性，叫做 interface default method。interface 不再只能 “说空话”，而也能有自己的方法体，而实现类只要不重写，该方法的 declaring class 就是 interface，Xposed 进行 hook 时就会抛出异常。

Xposed 和各路实现基本原理都是对`entry_point_from_quick_compiled_code_`这个成员动手脚，可以直接修改这个成员也可以 inline hook；而在 art 中有一个 “万能入口”：解释执行入口，通过设置方法入口为解释执行可以使得 Xposed hook 失效，但对 Frida 这种修改了解释器的无效。

  

> 博客内容遵循 署名 - 非商业性使用 - 相同方式共享 4.0 国际 (CC BY-NC-SA 4.0) 协议
> 
> 本文永久链接是：[https://blog.canyie.top/2021/05/01/anti-magisk-xposed/](https://blog.canyie.top/2021/05/01/anti-magisk-xposed/)