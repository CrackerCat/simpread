> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282605.htm)

> [原创]Android 系统启动源码分析

```
源码版本
android-7.0.0_r1
 
链接：https://pan.baidu.com/s/1yla9fqd4EbxSBSemrYVjsA?pwd=ukvt
提取码：ukvt
--来自百度网盘超级会员V3的分享
 
测试机
pixel3 android10

```

读前一段话
=====

​ 这篇系统启动流程分析借鉴了不少网络上的资源，如下

```
https://blog.csdn.net/qq_43369592/article/details/123113889
https://juejin.cn/post/7232947178690412602#heading-22
https://blog.csdn.net/fdsafwagdagadg6576/article/details/116333916
https://www.cnblogs.com/wanghao-boke/p/18099022
https://juejin.cn/post/6844903506915098637#heading-8
《Android进阶解密》 刘望舒
《深入理解Android Java虚拟机ART》邓凡平

```

​ 写这篇文章的目的是在阅读上面师傅的产出并且对照手上的 Android7.0 版本的时候，有很多代码做了修改。

​ 同时我也是读源码的小白，对于启动流程本来只知道先 init 再 zygote 再 systemserver 再 launcher，但是很多时候都不知道哪里的代码进行的过程的推进，也算是对自己知识的精细化。但本篇并没有到特别的深入的层次，例如：**AMS 启动，SystemServiceManager 启动，Launcher 获取应用信息的机制**，这些没有写，仅仅做了介绍。如果看的师傅感兴趣，我也贴了我觉得写的好的资源，大家可以看看。

​ 可能会有一些造轮子的嫌疑，但毕竟是自己一步步分析的成果，希望能通过这篇自己有些成长。同时也希望和我一样的源码阅读初学者有些助力。

​ 这是在看雪的第一篇博客，很多地方写的有点啰嗦，可能还有不对的地方，希望师傅们不吝赐教。

```
提醒：
建议阅读本篇文章的时候先读每一个部分的总结（如4，5，6的总结，有总结的一般是我觉得比较复杂的），有源码阅读经验的师傅应该晓得有些时候我们跟源码的时候可能会忘了为什么跟到这里，所以可以先看总结，以了解写某块分析的目的。
 
第四部分init当中4.2.1和4.2.2可能会有些啰嗦和抽象，可以自己看看源码，会有一些收获。

```

先来一个总览

![](https://bbs.kanxue.com/upload/attach/202407/993634_DHT6JBXDECNKGNU.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/993634_A75HTRRZVBWAY8W.jpg)

1. 启动电源
=======

按下电源之后，引导芯片代码从预定义的地方开始执行（硬件写死的一个位置）开始执行，加载 Bootloader 到 RAM，开始执行。

2.Bootloader 程序
===============

在 Android 操作系统开始前的一个程序，主要作用把系统 OS 给拉起

```
这一块经常root的朋友应该接触的比较多，我对这个研究比较浅，但是冲浪的时候看到了一个比较有趣比较猛的Bootloader破解的方式
https://www.4hou.com/posts/5VBq

```

3.idle 进程（内核空间）
===============

之前看权威指南的时候，没有写到这里的这个内核空间的一个进程，但实际上这才是第一个进程，这个进程是在内核空间做初始化的，和启动 init 进程的。

![](https://bbs.kanxue.com/upload/attach/202407/993634_VMUQ6SP4ECTSWNN.jpg)

可以参考一下这个帖子

[https://blog.csdn.net/marshal_zsx/article/details/80225854](https://blog.csdn.net/marshal_zsx/article/details/80225854)

4.init 进程
=========

由上 init 进程是由内核空间启动的，里面有这样一段代码

```
if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init") ||
        !try_to_run_init_process("/bin/init") ||
        !try_to_run_init_process("/bin/sh"))
        return 0;

```

需要关心的就是这里的 init，文件在 system/bin/init

根据 mk 文件我们可以看到它是如何编译出来的

D:\android-7.0.0_r1\system\core\init\Android.mk

```
LOCAL_SRC_FILES:= \
    bootchart.cpp \
    builtins.cpp \
    devices.cpp \
    init.cpp \
    keychords.cpp \
    property_service.cpp \
    signal_handler.cpp \
    ueventd.cpp \
    ueventd_parser.cpp \
    watchdogd.cpp \

```

可以看到这里的这个 init.cpp

4.1 init.cpp
------------

main 主要的逻辑不是很长

```
int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
 
    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
 
    // Clear the umask.
    umask(0);
 
    add_environment("PATH", _PATH_DEFPATH);
 
    bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);
 
    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    // 这里是挂载上文件系统
    if (is_first_stage) {
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }
 
    // We must have some place other than / to create the device nodes for
    // kmsg and null, otherwise we won't be able to remount / read-only
    // later on. Now that tmpfs is mounted on /dev, we can actually talk
    // to the outside world.
    open_devnull_stdio();
    klog_init();
    klog_set_level(KLOG_NOTICE_LEVEL);
 
    NOTICE("init %s started!\n", is_first_stage ? "first stage" : "second stage");
 
    if (!is_first_stage) {
        // Indicate that booting is in progress to background fw loaders, etc.
        close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
        // 初始化属性
        property_init();
 
        // If arguments are passed both on the command line and in DT,
        // properties set in DT always have priority over the command-line ones.
        process_kernel_dt();
        process_kernel_cmdline();
 
        // Propagate the kernel variables to internal variables
        // used by init as well as the current required properties.
        export_kernel_boot_props();
    }
     
    // Set up SELinux, including loading the SELinux policy if we're in the kernel domain.
    selinux_initialize(is_first_stage);
 
    // If we're in the kernel domain, re-exec init to transition to the init domain now
    // that the SELinux policy has been loaded.
    if (is_first_stage) {
        if (restorecon("/init") == -1) {
            ERROR("restorecon failed: %s\n", strerror(errno));
            security_failure();
        }
        char* path = argv[0];
        char* args[] = { path, const_cast("--second-stage"), nullptr };
        if (execv(path, args) == -1) {
            ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
            security_failure();
        }
    }
 
    // These directories were necessarily created before initial policy load
    // and therefore need their security context restored to the proper value.
    // This must happen before /dev is populated by ueventd.
    NOTICE("Running restorecon...\n");
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon("/property_contexts");
    restorecon_recursive("/sys");
 
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        ERROR("epoll_create1 failed: %s\n", strerror(errno));
        exit(1);
    }
 
    signal_handler_init();
 
    property_load_boot_defaults();
    export_oem_lock_status();
    start_property_service();
 
    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);
    //在这里建立一个parser对象 开始解析init.rc
    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique());
    parser.AddSectionParser("on", std::make_unique());
    parser.AddSectionParser("import", std::make_unique());
    parser.ParseConfig("/init.rc");
 
    ActionManager& am = ActionManager::GetInstance();
 
    am.QueueEventTrigger("early-init");
 
    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");
 
    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");
 
    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
 
    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = property_get("ro.bootmode");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }
 
    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
 
    while (true) {
        if (!waiting_for_exec) {
            am.ExecuteOneCommand();
            restart_processes();
        }
 
        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }
 
        if (am.HasMoreCommands()) {
            timeout = 0;
        }
 
        bootchart_sample(&timeout);
 
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
 
    return 0;
} 
```

其实 Android7.0 我看的这个版本 init.cpp 的 main 还是比较简单的，到后面 Android10 代码结构变了一些，将 first stage，second stage，解析的代码全都封装进别的函数了，对分析者来说可能更好懂一些吧。

做一下总结，init 这个进程做了什么在 init.cpp 可以看个大概

1.  挂载文件 or 创建文件
2.  初始化操作（例如 umask(0)，进行权限设置等）
3.  setlinux linux 相关安全策略
4.  解析 init.rc

4.2 init.rc
-----------

这一块稍微有点搞，按照正常流程走的话，里面就到了要开始准备进入 zygote 相关的逻辑了。

因为 Android 稍高一些的逻辑是如下这样，可以找到 start zygote 字样

![](https://bbs.kanxue.com/upload/attach/202407/993634_HXTRGGBPZFJTSEF.jpg)

但对于 Android7.0 版本

![](https://bbs.kanxue.com/upload/attach/202407/993634_WUUU44M8BNYMCDG.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/993634_85J5M5UAPBRHWDK.jpg)

所以暂时刚开始来到这里的时候其实还是去研究了一下的，先说一下，是这里 class_start main 进行启动的

![](https://bbs.kanxue.com/upload/attach/202407/993634_2VHPGPE2HZP8RQU.jpg)

要知道他怎么启动的，需要掌握一些 rc 的 AIL 文件和 Parser 如何解析 init.rc 相关知识。

### 4.2.1 AIL 语言

Android Init Language 安卓初始化语言，这一块本人了解不深，研究启动流程也不需要太深理解，所以大概看一下就选（我写的有点啰嗦）

主要有五类语句，Actions、Commands、Services、Options、Imports

Actions、Services、Import 可以确定一个 Section，如下是一个 section

```
on boot
    ifup lo
    hostname localhost
    domainname localdomain

```

#### 4.2.1.1 Action

格式

后面会包含一个 trigger 触发器 表明何是执行这个 Action

```
on [&& ]*
   
   
   
   ... 
```

#### 4.2.1.2 Commands

简单来讲就是要执行的命令

<table><thead><tr><th>命令</th><th>解释</th></tr></thead><tbody><tr><td><code>bootchart_init</code></td><td>如果配置了 bootcharing, 则启动. 包含在默认的 init.rc 中</td></tr><tr><td><code>chmod</code></td><td>更改文件权限</td></tr><tr><td><code>chown &lt;owner&gt; &lt;group&gt; &lt;path&gt;</code></td><td>更改文件的所有者和组</td></tr><tr><td><code>calss_start &lt;serviceclass&gt;</code></td><td>启动指定类别服务下的所有未启动的服务</td></tr><tr><td><code>class_stop &lt;serviceclass&gt;</code></td><td>停止指定类别服务类下的所有已运行的服务</td></tr><tr><td><code>class_reset &lt;serviceclass&gt;</code></td><td>停止指定类别的所有服务 (服务还在运行), 但不会禁用这些服务. 后面可以通过 class_start 重启这些服务</td></tr><tr><td><code>copy &lt;src&gt; &lt;dst&gt;</code></td><td>复制文件, 对二进制 / 大文件非常有用</td></tr><tr><td><code>domainname &lt;name&gt;</code></td><td>设置域名称</td></tr><tr><td><code>enable &lt;servicename&gt;</code></td><td>启用已经禁用的服务</td></tr><tr><td><code>exec [ &lt;seclabel&gt; [ &lt;user&gt; [ &lt;group&gt; ]* ]]</code> <code>--&lt;command&gt; [ &lt;argument&gt; ]*</code></td><td>fork 一个进程执行指定命令, 如果有参数, 则带参数执行</td></tr><tr><td><code>export &lt;name&gt;</code></td><td>在全局环境中, 将<code>&lt;name&gt;</code>变量的值设置为<code>&lt;value&gt;</code>, 即以键值对的方式设置全局环境变量. 这些变量对之后的任何进程都有效</td></tr><tr><td><code>hostname</code></td><td>设置主机名</td></tr><tr><td><code>ifup &lt;interface&gt;</code></td><td>启动某个网络接口</td></tr><tr><td><code>insmod [-f] &lt;path&gt; [&lt;options&gt;]</code></td><td>加载指定路径下的驱动模块。-f 强制加载，即不管当前模块是否和 linux kernel 匹配</td></tr><tr><td><code>load_all_props</code></td><td>从 / system，/vendor 加载属性。默认包含在 init.rc</td></tr><tr><td><code>load_persist_props</code></td><td>当 / data 被加密时，加载固定属性</td></tr><tr><td><code>loglevel &lt;level&gt;</code></td><td>设置 kernel 日志等级</td></tr><tr><td><code>mkdir &lt;path&gt; [mode] [owner] [group]</code></td><td>在制定路径下创建目录</td></tr><tr><td><code>mount_all &lt;fstab&gt; [ &lt;path&gt; ]*</code></td><td>在给定的 fs_mgr-format 上调用 fs_mgr_mount 和引入 rc 文件</td></tr><tr><td><code>mount &lt;type&gt; &lt;device&gt; &lt;dir&gt;[ &lt;flag&gt; ]* [&lt;options&gt;]</code></td><td>挂载指定设备到指定目录下.</td></tr><tr><td><code>powerct</code></td><td>用来应对 sys.powerctl 中系统属性的变化, 用于系统重启</td></tr><tr><td><code>restart &lt;service&gt;</code></td><td>重启制定服务，但不会禁用该服务</td></tr><tr><td><code>restorecon &lt;path&gt; [ &lt;path&gt; ]*</code></td><td>恢复指定文件到 file_contexts 配置中指定的安全上线文环境</td></tr><tr><td><code>restorecon_recursive &lt;path&gt; [ &lt;path&gt; ]*</code></td><td>以递归的方式恢复指定目录到 file_contexts 配置中指定的安全上下文中</td></tr><tr><td><code>rm &lt;path&gt;</code></td><td>删除指定路径下的文件</td></tr><tr><td><code>rmdir &lt;path&gt;</code></td><td>删除制定路径下的目录</td></tr><tr><td><code>setprop &lt;name&gt; &lt;value&gt;</code></td><td>将系统属性<code>&lt;name&gt;</code>的值设置为<code>&lt;value&gt;</code>, 即以键值对的方式设置系统属性</td></tr><tr><td><code>setrlimit &lt;resource&gt; &lt;cur&gt; &lt;max&gt;</code></td><td>设置资源限制</td></tr><tr><td><code>start &lt;service&gt;</code></td><td>启动服务 (如果该服务还未启动)</td></tr><tr><td><code>stop &lt;service&gt;</code></td><td>关闭服务 (如果该服务还未停止)</td></tr><tr><td><code>swapon_all &lt;fstab&gt;</code></td><td></td></tr><tr><td><code>symlink &lt;target&gt; &lt;path&gt;</code></td><td>创建一个指向<code>&lt;path&gt;</code>的符合链接<code>&lt;target&gt;</code></td></tr><tr><td><code>sysclktz &lt;mins_west_of_gmt&gt;</code></td><td>设置系统时钟的基准, 比如 0 代表 GMT, 即以格林尼治时间为准</td></tr><tr><td><code>trigger &lt;event&gt;</code></td><td>触发一个事件, 将该 action 排在某个 action 之后 (用于 Action 排队)</td></tr><tr><td><code>verity_load_state</code></td><td></td></tr><tr><td><code>verity_update_state &lt;mount_point&gt;</code></td><td></td></tr><tr><td><code>wait &lt;path&gt; [ &lt;timeout&gt; ]</code></td><td>等待一个文件是否存在, 存在时立刻返回或者超时后返回. 默认超时事件是 5s</td></tr><tr><td><code>write &lt;path&gt; &lt;content&gt;</code></td><td>写内容到指定文件中</td></tr></tbody></table>

复制自 [https://blog.csdn.net/w2064004678/article/details/105510821](https://blog.csdn.net/w2064004678/article/details/105510821)

#### 4.2.1.3 Services

表明一些在初始化时就启动或者退出时需要重启的程序

格式

```
service [ ]*
          ... 
```

一个例子

```
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0

```

#### 4.2.1.4 Options

是用来修饰服务的，告诉服务要怎么进行运行

<table><thead><tr><th>选项</th><th>解释</th></tr></thead><tbody><tr><td><code>console</code></td><td>服务需要一个控制台.</td></tr><tr><td><code>critical</code></td><td>表示这是一个关键设备服务. 如果 4 分钟内此服务退出 4 次以上, 那么这个设备将重启进入 recovery 模式</td></tr><tr><td><code>disabled</code></td><td>服务不会自动启动, 必须通过服务名显式启动</td></tr><tr><td><code>setenv &lt;name&gt; &lt;value&gt;</code></td><td>在进程启动过程中, 将环境变量<code>&lt;name&gt;</code>的值设置为<code>&lt;value&gt;</code>, 即以键值对的方式设置环境变量</td></tr><tr><td><code>socket &lt;name&gt; &lt;type&gt; &lt;perm&gt; [ &lt;user&gt; [ &lt;group&gt; [seclabel]]]</code></td><td>创建一个 unix 域下的 socket, 其被命名<code>/dev/socket/&lt;name&gt;</code>. 并将其文件描述符 fd 返回给服务进程. 其中, type 必须为 dgram,stream 或者 seqpacke,user 和 group 默认是 0.seclabel 是该 socket 的 SELLinux 的安全上下文环境, 默认是当前 service 的上下文环境, 通过 seclabel 指定.</td></tr><tr><td><code>user &lt;username&gt;</code></td><td>在执行此服务之前切换用户名, 当前默认的是 root. 自 Android M 开始, 即使它要求 linux capabilities, 也应该使用该选项. 很明显, 为了获得该功能, 进程需要以 root 用户运行</td></tr><tr><td><code>group &lt;groupname&gt;</code></td><td>在执行此服务之前切换组名, 除了第一个必须的组名外, 附加的组名用于设置进程的补充组 (借助 setgroup() 函数), 当前默认的是 root</td></tr><tr><td><code>seclabel &lt;seclabel&gt;</code></td><td>在执行该服务之前修改其安全上下文, 默认是 init 程序的上下文</td></tr><tr><td><code>oneshot</code></td><td>当服务退出时, 不重启该服务</td></tr><tr><td><code>class &lt;name&gt;</code></td><td>为当前 service 设定一个类别. 相同类别的服务将会同时启动或者停止, 默认类名是 default.</td></tr><tr><td><code>onrestart</code></td><td>当服务重启时执行该命令</td></tr><tr><td><code>priority &lt;priority&gt;</code></td><td>设置服务进程的优先级. 优先级取值范围为 - 20~19, 默认是 0. 可以通过 setpriority() 设置</td></tr></tbody></table>

#### 4.2.1.5 Imports

引入配置文件

```
import 
```

例子

```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc

```

AIL 的介绍就到这了，**如果想要详细了解请阅读 system/core/init 下的 readme.txt 文件**。

### 4.2.2 init_parser.cpp

目光先转到 init.cpp 里面，下面这段代码对 init.rc 进行解析

```
Parser& parser = Parser::GetInstance();
parser.AddSectionParser("service",std::make_unique());
parser.AddSectionParser("on", std::make_unique());
parser.AddSectionParser("import", std::make_unique());
parser.ParseConfig("/init.rc"); 
```

ok，这个 parser 是怎么写的呢，逻辑就在 init_parser.cpp 里面

一点点来，既然上面先调用了 parser.AddSectionParser，那我们就看看这个函数

可以看到第二个参数 parser 被保存在以第一个参数 name

为标号的 section_parsers_里面，而 section_parsers__是一个 map 集合，也就是这个函数的作用是**将 parser 和对应的 section 进行绑定**

```
void Parser::AddSectionParser(const std::string& name,
                              std::unique_ptr parser) {
    section_parsers_[name] = std::move(parser);
} 
```

其实以上操作很大意义上是在做解释器初始化的工作。

接下来看看解析的这一行 parser.ParseConfig("/init.rc");，其中 parser.ParseConfig

可以看到这个函数作用就是根据传进来的参数看是文件路径还是目录路径，目录路径就调用 ParseConfigDir 进行下一步的递归，找到文件路径然后再调用 ParseConfigFile

```
bool Parser::ParseConfig(const std::string& path) {
    if (is_dir(path.c_str())) {
        return ParseConfigDir(path);
    }
    return ParseConfigFile(path);
}
 
 
bool Parser::ParseConfigDir(const std::string& path) {
    INFO("Parsing directory %s...\n", path.c_str());
    std::unique_ptr config_dir(opendir(path.c_str()), closedir);
    if (!config_dir) {
        ERROR("Could not import directory '%s'\n", path.c_str());
        return false;
    }
    dirent* current_file;
    while ((current_file = readdir(config_dir.get()))) {
        std::string current_path =
            android::base::StringPrintf("%s/%s", path.c_str(), current_file->d_name);
        // Ignore directories and only process regular files.
        if (current_file->d_type == DT_REG) {
            if (!ParseConfigFile(current_path)) {
                ERROR("could not import file '%s'\n", current_path.c_str());
            }
        }
    }
    return true;
} 
```

那么重点来到 ParseConfigFile

可以看到下面的作用就是

1.  从 rc 文件中读取内容保存在 data 中
2.  调用 ParseData 进行解析

```
bool Parser::ParseConfigFile(const std::string& path) {
    INFO("Parsing file %s...\n", path.c_str());
    Timer t;
    std::string data;
    if (!read_file(path.c_str(), &data)) {
        return false;
    }
 
    data.push_back('\n'); // TODO: fix parse_config.
    ParseData(path, data);
    for (const auto& sp : section_parsers_) {
        sp.second->EndFile(path);
    }
 
    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();
 
    NOTICE("(Parsing %s took %.2fs.)\n", path.c_str(), t.duration());
    return true;
}

```

进一步来到 ParseData

代码稍长，把分析写在代码里面了

```
void Parser::ParseData(const std::string& filename, const std::string& data) {
    //TODO: Use a parser with const input and remove this copy
    //将 rc 中的内容保存在 vector 中便于逐个字符进行解析
    std::vector data_copy(data.begin(), data.end());
    data_copy.push_back('\0');
 
    parse_state state;
    state.filename = filename.c_str();
    state.line = 0;
    state.ptr = &data_copy[0];
    state.nexttoken = 0;
 
    SectionParser* section_parser = nullptr;
    //存放每一行的内容
    std::vector args;
 
    for (;;) {
        switch (next_token(&state)) {
        case T_EOF:
            if (section_parser) {
                section_parser->EndSection();
            }
            return;
        case T_NEWLINE:
            state.line++;
            //某行为空 则不进行解析
            if (args.empty()) {
                break;
            }
            // 通过判断args[0]的内容是不是services on import来判断是否是section的起始位置
            if (section_parsers_.count(args[0])) {
                if (section_parser) {
                    //结束解析
                    section_parser->EndSection();
                }
                //取出对应的解释器
                section_parser = section_parsers_[args[0]].get();
                std::string ret_err;
                //进行解析了
                if (!section_parser->ParseSection(args, &ret_err)) {
                    parse_error(&state, "%s\n", ret_err.c_str());
                    section_parser = nullptr;
                }
            } else if (section_parser) {
                //不是section起始位置的话，这行就属于是section子块，进行line解析
                std::string ret_err;
                if (!section_parser->ParseLineSection(args, state.filename,
                                                      state.line, &ret_err)) {
                    parse_error(&state, "%s\n", ret_err.c_str());
                }
            }
            args.clear();
            break;
        case T_TEXT:
            args.emplace_back(state.text);
            break;
        }
    }
} 
```

简单来说上述函数就是通过判断，调用

1.  ParseSection
2.  ParseLineSection
3.  EndSection

这根据解释器的不同，逻辑也有些不同，如果你是 Action，则在 ActionParser ，如果 service 则在 ServiceParser

#### 4.2.2.1 ActionParser

action.cpp

ParseSection 是创建 Action 对象（我理解下来就是对 action 的操作） 为对象添加一个触发器 并且将 action_移动到目前的 Action 对象里面去

```
bool ActionParser::ParseSection(const std::vector& args,
                                std::string* err) {
    std::vector triggers(args.begin() + 1, args.end());
    if (triggers.size() < 1) {
        *err = "actions must have a trigger";
        return false;
    }
 
    auto action = std::make_unique(false);
    if (!action->InitTriggers(triggers, err)) {
        return false;
    }
 
    action_ = std::move(action);
    return true;
} 
```

ParseLineSection 这个就是每一行（我理解下来就是对 command 的操作）

```
bool ActionParser::ParseLineSection(const std::vector& args,
                                    const std::string& filename, int line,
                                    std::string* err) const {
    return action_ ? action_->AddCommand(args, filename, line, err) : false;
}
调用下面这个AddCommand
bool Action::AddCommand(const std::vector& args,
                        const std::string& filename, int line, std::string* err) {
    if (!function_map_) {
        *err = "no function map available";
        return false;
    }
 
    if (args.empty()) {
        *err = "command needed, but not provided";
        return false;
    }
//这里提及了一个function_map_ 很重要
    auto function = function_map_->FindFunction(args[0], args.size() - 1, err);
    if (!function) {
        return false;
    }
 
    AddCommand(function, args, filename, line);
    return true;
}
在调用下面这个AddCommand
void Action::AddCommand(BuiltinFunction f,
                        const std::vector& args,
                        const std::string& filename, int line) {
    commands_.emplace_back(f, args, filename, line);
} 
```

**根据上面的这个 function_map_ 查找设置对应的 command 处理函数**

例如启动 zygote 的 class start，就是在这里被设置好 碰见 class start 的时候调用哪个函数

那具体调用哪个函数呢，继续往下看 function_map_在那被复值

```
static const KeywordMap* function_map_;
 
static void set_function_map(const KeywordMap* function_map) {
    function_map_ = function_map;
} 
```

需要再次回到 init.cpp 找到下面这行

```
const BuiltinFunctionMap function_map;
Action::set_function_map(&function_map);

```

c 下面就要找到 BuiltinFunctionMap 实在 builtins.cpp 里面的

我们找到下面这个 map

```
BuiltinFunctionMap::Map& BuiltinFunctionMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits::max();
    static const Map builtin_functions = {
        {"bootchart_init",          {0,     0,    do_bootchart_init}},
        {"chmod",                   {2,     2,    do_chmod}},
        {"chown",                   {2,     3,    do_chown}},
        {"class_reset",             {1,     1,    do_class_reset}},
        {"class_start",             {1,     1,    do_class_start}},
        {"class_stop",              {1,     1,    do_class_stop}},
        {"copy",                    {2,     2,    do_copy}},
        {"domainname",              {1,     1,    do_domainname}},
        {"enable",                  {1,     1,    do_enable}},
        {"exec",                    {1,     kMax, do_exec}},
        {"export",                  {2,     2,    do_export}},
        {"hostname",                {1,     1,    do_hostname}},
        {"ifup",                    {1,     1,    do_ifup}},
        {"init_user0",              {0,     0,    do_init_user0}},
        {"insmod",                  {1,     kMax, do_insmod}},
        {"installkey",              {1,     1,    do_installkey}},
        {"load_persist_props",      {0,     0,    do_load_persist_props}},
        {"load_system_props",       {0,     0,    do_load_system_props}},
        {"loglevel",                {1,     1,    do_loglevel}},
        {"mkdir",                   {1,     4,    do_mkdir}},
        {"mount_all",               {1,     kMax, do_mount_all}},
        {"mount",                   {3,     kMax, do_mount}},
        {"powerctl",                {1,     1,    do_powerctl}},
        {"restart",                 {1,     1,    do_restart}},
        {"restorecon",              {1,     kMax, do_restorecon}},
        {"restorecon_recursive",    {1,     kMax, do_restorecon_recursive}},
        {"rm",                      {1,     1,    do_rm}},
        {"rmdir",                   {1,     1,    do_rmdir}},
        {"setprop",                 {2,     2,    do_setprop}},
        {"setrlimit",               {3,     3,    do_setrlimit}},
        {"start",                   {1,     1,    do_start}},
        {"stop",                    {1,     1,    do_stop}},
        {"swapon_all",              {1,     1,    do_swapon_all}},
        {"symlink",                 {2,     2,    do_symlink}},
        {"sysclktz",                {1,     1,    do_sysclktz}},
        {"trigger",                 {1,     1,    do_trigger}},
        {"verity_load_state",       {0,     0,    do_verity_load_state}},
        {"verity_update_state",     {0,     0,    do_verity_update_state}},
        {"wait",                    {1,     2,    do_wait}},
        {"write",                   {2,     2,    do_write}},
    };
    return builtin_functions;
} 
```

那 class_start 对应的就是 do_class_start，在 builtins 里面就写好了

![](https://bbs.kanxue.com/upload/attach/202407/993634_5G8RDWH3XUTFRN3.jpg)

具体的逻辑我们下面再去看，先结束掉 ActionParser 的 ParseLineSection

再是 EndSection

```
void ActionParser::EndSection() {
    if (action_ && action_->NumCommands() > 0) {
        ActionManager::GetInstance().AddAction(std::move(action_));
    }
}
 
void ActionManager::AddAction(std::unique_ptr action) {
    ...
 
    if (old_action_it != actions_.end()) {
        (*old_action_it)->CombineAction(*action);
    } else {
      //将解析之后的 action 对象增加到 actions_ 链表中，用于遍历执行。
        actions_.emplace_back(std::move(action));
    }
}
 
//ActionManager 在 action.h 中的定义
class ActionManager {
public:
    static ActionManager& GetInstance();
 
    void AddAction(std::unique_ptr action);
    void QueueEventTrigger(const std::string& trigger);
    void QueuePropertyTrigger(const std::string& name, const std::string& value);
    void QueueAllPropertyTriggers();
    void QueueBuiltinAction(BuiltinFunction func, const std::string& name);
    void ExecuteOneCommand();
    bool HasMoreCommands() const;
    void DumpState() const;
 
private:
    ActionManager();
 
    ActionManager(ActionManager const&) = delete;
    void operator=(ActionManager const&) = delete;
 
    std::vector> actions_; //actions_ 的定义
    std::queue> trigger_queue_;
    std::queue current_executing_actions_;
    std::size_t current_command_;
}; 
```

来总结一下解析 Action 的过程

1.  创建一个 Action 对象
2.  把 Action 对象添加 Trigger 以及对应的 command
3.  添加 command 时 为 command 设置好对应的处理函数逻辑
4.  把 Action 对象添加到 ActionManager vector 类型的 actions_ 链表当中去

#### 4.2.2.2 ServiceParser

起始大体逻辑和 Action 是很像的

```
bool ServiceParser::ParseSection(const std::vector& args,
                                 std::string* err) {
    ...
  //获取服务名
    const std::string& name = args[1];
    ...
  //保存服务名外的参数（如执行路径等）
    std::vector str_args(args.begin() + 2, args.end());
  //将 service_ 指针指向当前 Service 对象
    service_ = std::make_unique(name, "default", str_args);
    return true;
} 
```

```
bool ServiceParser::ParseLineSection(const std::vector& args,
                                     const std::string& filename, int line,
                                     std::string* err) const {
  //为 Service 中的每一个 Option 指定处理函数
    return service_ ? service_->HandleLine(args, err) : false;
}
 
bool Service::HandleLine(const std::vector& args, std::string* err) {
    ...
 
    static const OptionHandlerMap handler_map;
  //寻找对应 option 的处理函数
    auto handler = handler_map.FindFunction(args[0], args.size() - 1, err);
 
    ...
 
    return (this->*handler)(args, err);
} 
```

```
void ServiceParser::EndSection() {
    if (service_) {
        ServiceManager::GetInstance().AddService(std::move(service_));
    }
}
 
void ServiceManager::AddService(std::unique_ptr service) {
    Service* old_service = FindServiceByName(service->name());
    if (old_service) {
        ERROR("ignored duplicate definition of service '%s'",
              service->name().c_str());
        return;
    }
  //将解析之后 service 对象增加到 services_ 链表中
    services_.emplace_back(std::move(service));
}
 
//ServiceManager 在 service.h 中的定义
class ServiceManager {
public:
    static ServiceManager& GetInstance();
 
    void AddService(std::unique_ptr service);
    Service* MakeExecOneshotService(const std::vector& args);
    Service* FindServiceByName(const std::string& name) const;
    Service* FindServiceByPid(pid_t pid) const;
    Service* FindServiceByKeychord(int keychord_id) const;
    void ForEachService(std::function callback) const;
    void ForEachServiceInClass(const std::string& classname,
                               void (*func)(Service* svc)) const;
    void ForEachServiceWithFlags(unsigned matchflags,
                             void (*func)(Service* svc)) const;
    void ReapAnyOutstandingChildren();
    void RemoveService(const Service& svc);
    void DumpState() const;
 
private:
    ServiceManager();
    bool ReapOneProcess();
    static int exec_count_; // Every service needs a unique name.
    std::vector> services_; //services_ 的定义
}; 
```

与 Action 类似

1.  创建 Service 对象 解析 Service 对象并把对应参数添加进去
2.  为每个 Option 对象设置处理函数

#### 4.2.2.3 command 如何被执行

现在有了 AIL 的知识，和解释器如何解释每一条的 command，现在设备就可以正常读取，并且管理这个 init.rc 文件当中的内容了（感觉还是在做初始化，还没到运行）

我们的目的是要找到如何启动 zygote，那如何执行 command 也很重要（比如 class_start main 如何锁定到 zygote）

回到 init.cpp

```
while (true) {
        if (!waiting_for_exec) {
            am.ExecuteOneCommand();
            restart_processes();
        }
 
        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }
 
        if (am.HasMoreCommands()) {
            timeout = 0;
        }
 
        bootchart_sample(&timeout);
 
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

```

可以看到 action 的执行，是通过 ExecuteOneCommand 的调用的，往上找，很容易找到是 ActionManager 的 ExecuteOneCommand，进去看

```
void ActionManager::ExecuteOneCommand() {
    while (current_executing_actions_.empty() && !trigger_queue_.empty()) {
        //遍历 actions_
        for (const auto& action : actions_) {
            if (trigger_queue_.front()->CheckTriggers(*action)) {
                //将 action 加入到 current_executing_actions_ 中
                current_executing_actions_.emplace(action.get());
            }
        }
        trigger_queue_.pop();
    }
 
    ...
 
    //每次只执行一个 action，下次 init 进程 while 循环时，跳过上面的 while 循环，接着执行
    auto action = current_executing_actions_.front();
 
    if (current_command_ == 0) {
        std::string trigger_name = action->BuildTriggersString();
        INFO("processing action (%s)\n", trigger_name.c_str());
    }
 
    //执行 action 的 command
    action->ExecuteOneCommand(current_command_);
 
    ++current_command_;
    ...
}
 
void Action::ExecuteOneCommand(std::size_t command) const {
  //执行 action 对象中保存的 command
    ExecuteCommand(commands_[command]);
}
 
void Action::ExecuteCommand(const Command& command) const {
    Timer t;
  //调用 command 对应的处理函数
    int result = command.InvokeFunc();
 
    ...
}

```

小小总结一下

1.  init 进入无限循环
2.  按照 ActionManager 的 actions_保存的顺序对每个 Action 进行处理

#### 4.2.2.4 zygote 如何启动

好，现在基本上了解了 AIL，如何解析启动 zygote 所在的 init.rc，init.rc 中的 command 如何运行

通识的东西基本上大差不差，接下来要去看看如何启动 zygote（感觉前面有点啰嗦）

还记得之前我说很重要的这个函数吧

```
static int do_class_start(const std::vector& args) {
    ServiceManager::GetInstance().
        ForEachServiceInClass(args[1], [] (Service* s) { s->StartIfNotDisabled(); });
    return 0;
} 
```

里面调用 service 的 StartIfNotDisabled

```
bool Service::StartIfNotDisabled() {
    if (!(flags_ & SVC_DISABLED)) {
        return Start();
    } else {
        flags_ |= SVC_DISABLED_START;
    }
    return true;
}
 
bool Service::Start() {
    ....
   //创建子进程
    pid_t pid = fork();
    if (pid == 0) {
        ...
       //执行对应 service 对应的执行文件，args_[0].c_str() 就是执行路径
        if (execve(args_[0].c_str(), (char**) &strs[0], (char**) ENV) < 0) {
            ERROR("cannot execve('%s'): %s\n", args_[0].c_str(), strerror(errno));
        }
 
        _exit(127);
    }
 
    ....
    return true;
}

```

可以看到又调用了 Service 的 Start 函数

代码里的是 class-start main

我们看到，里面 class 的标识就是 main

![](https://bbs.kanxue.com/upload/attach/202407/993634_JTM94QBGGF3GKPP.jpg)

好了，做一下总结

**init.cpp 做了一些 command 的初始化，并且在解释器里面把 init.rc 读入让设备看懂 init 的初始化配置，然后再在 init.cpp 里面无限循环中调用 init.rc 当中的 command，其中就包括了 zygote 的启动**

4.3 总结
------

init 主要工作

1.  创建一些文件夹 挂载文件 挂载设备
2.  初始化和启动属性服务
3.  通过解析 init.rc 和 其他对应 rc 文件，启动对应的系统级进程。其中包括后面要讲的 zygote

接下来逻辑进入到 zygote

5 zygote
========

到我手上这个 pixel3 的 android10 版本是有俩 zygote 的 分别是两个不同版本

![](https://bbs.kanxue.com/upload/attach/202407/993634_AJYPFC3F5VDE4S5.jpg)

进入到 zygote，其实是先进入 native 层的 zygote

5.1 native 层的 zygote
--------------------

首先根据设备的信息启动不同类型的 zygote（位数），我以 64 32 为例

可以看到二进制文件是 app_process64 和 app_process32，-Xzygote /system/bin --zygote --start-system-server --socket-name=zygote 都是他的参数

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks /sys/fs/cgroup/stune/foreground/tasks
 
service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
    class main
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks

```

上面这个 app_process64 和 app_process32 都被编译完了，我们找到他的源代码看逻辑

代码比较长，我们主要关注三部分

1.  创建 app 运行时对象
2.  关键代码 1
3.  关键代码 2
4.  关键代码 2 的 runtime.start

```
int main(int argc, char* const argv[])
{
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        // Older kernels don't understand PR_SET_NO_NEW_PRIVS and return
        // EINVAL. Don't die on such kernels.
        if (errno != EINVAL) {
            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
            return 12;
        }
    }
创建app运行时对象
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;
 
    // Everything up to '--' or first non '-' arg goes to the vm.
    //
    // The first argument after the VM args is the "parent dir", which
    // is currently unused.
    //
    // After the parent dir, we expect one or more the following internal
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.
    //
    // For non zygote starts, these arguments will be followed by
    // the main class name. All remaining arguments are passed to
    // the main method of this class.
    //
    // For zygote starts, all remaining arguments are passed to the zygote.
    // main function.
    //
    // Note that we must copy argument string values since we will rewrite the
    // entire argument block when we apply the nice name to argv0.
 
    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }
 
    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;
关键代码1
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
 
    Vector args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();
 
        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }
 
        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
 
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);
 
        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
 
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }
关键代码2
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
} 
```

### 5.1.1 关键代码 1

可以看到里面如 --start-system-server 是我们的参数 - Xzygote /system/bin --zygote --start-system-server 当中的一部分

他会进行标志位的设置

```
关键代码1
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

```

### 5.1.2 关键代码 2

上面根据我们的参数知道以 zygote 参数启动这个 app_process，里面 zygote 被设置为 true，那么就会进入下面这个判断中，特别别是 runtime.start("com.android.internal.os.ZygoteInit", args, zygote);

```
if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }

```

### 5.1.3 关键代码 2 的 runtime.start

这玩意就很长了 我先贴出来

```
void AndroidRuntime::start(const char* className, const Vector& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());
 
    static const String8 startSystemServer("start-system-server");
 
    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }
 
    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }
 
    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);
 
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
 
    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
 
    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
 
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
 
    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
 
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
 
#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);
 
    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
} 
```

里面这两处对理解 Android 系统还是有很大帮助的，，startVM 更是理解虚拟机的入口，标记一下，startreg 是用来注册 JNI 的

```
startVm(&mJavaVM, &env, zygote)
 
startReg(env)

```

然后就到这里 作用是 **找到 ZygoteInit 的 main 函数，然后通过 JNI 调用（写法很熟悉了），进入到 zygote 的 Java 层**

```
jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
"([Ljava/lang/String;)V”);
env->CallStaticVoidMethod(startClass, startMeth, strArray);

```

5.2 java 层的 zygote
------------------

如上 ZygoteInit.java.main 方法

先上代码

```
public static void main(String argv[]) {
        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();
 
        try {
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygoteInit");
            RuntimeInit.enableDdms();
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();
 
            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }
 
            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
 
            registerZygoteSocket(socketName);
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            preload();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
 
            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();
 
            // Do an initial gc to clean up after startup
            Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PostZygoteInitGC");
            gcAndFinalize();
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
 
            Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
 
            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);
 
            // Zygote process unmounts root storage spaces.
            Zygote.nativeUnmountStorageOnInit();
 
            ZygoteHooks.stopZygoteNoThreadCreation();
 
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }
 
            Log.i(TAG, "Accepting command socket connections");
            runSelectLoop(abiList);
 
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }

```

里面几个重要函数

```
registerZygoteSocket(socketName);
注册zygote用的socket用来和AMS进行通信，用来创建新的应用进程
 
preload();
预加载的资源、类、虚拟机实例等
 
startSystemServer(abiList, socketName);
启动SystemServer进程，就有之前学过的PMS这些
 
runSelectLoop(abiList);
循环等待并处理AMS发送来的创建新应用进程请求。如果收到创建应用程序的请求，则调用ZygoteConnection的runOnce函数来创建一个新的应用程序进程

```

上个时序图

![](https://bbs.kanxue.com/upload/attach/202407/993634_PTW4BNTHSXM2P3C.jpg)

5.3 总结
------

解析 rc 的参数

1.  在 native 层调用运行时的 start 方法，里面又创建虚拟机和注册 JNI 函数
    
2.  通过 JNI 方式调用 ZygoteInit 的 main 函数，进入到 java 层
    
3.  registerZygoteSocket 注册 zygote 用的 socket 用来和 AMS 进行通信，用来创建新的应用进程
    
4.  preload 预加载的资源、类、虚拟机实例等
    
5.  startSystemServer 启动 SystemServer 进程，就有之前学过的 PMS 这些
    
6.  runSelectLoop 循环等待并处理 AMS 发送来的创建新应用进程请求。
    
7.  到这里 zygote 就差不多干完初始化的活了，开始等待
    

6 Systemserver
==============

刚刚学到 Zygote 启动了 SyetemServer

我们先上一个时序图

![](https://bbs.kanxue.com/upload/attach/202407/993634_2BURHFQGQVGGCR8.jpg)

这一块我经理就稍微少一点，也是偷的别的师傅的分析，有个概念先

ZygoteInit.startSystemServer()

> fork 子进程 system_**server**，进入 system_**server** 进程。

ZygoteInit.handleSystemServerProcess()

> 设置当前进程名为 “system_**server**”，创建 PathClassLoader 类加载器。

RuntimeInit.zygoteInit()

> 重定向 log 输出, 通用的初始化 (设置默认异常捕捉方法, 时区等), 初始化 Zygote -> nativeZygoteInit()

app_main::onZygoteInit()

> proc->startThreadPool(); 启动 Binder 线程池，这样就可以与其他进程进行通信。

ZygoteInit.main()

> 开启 DDMS 功能, preload() 加载资源, 预加载 OpenGL, 调用 SystemServer.main() 方法

SystemServer.main()

> 先初始化 SystemServer 对象，再调用对象的 run() 方法。

到下面这里，就比较熟悉了

```
SystemServer.run() 
          createSystemContext
          startBootstrapServices();
          startCoreServices();
          startOtherServices();
          Looper.loop();

```

6.1 系统服务
--------

**startBootstrapServices：引导服务**

<table><thead><tr><th>服务</th><th>作用</th></tr></thead><tbody><tr><td>Installer</td><td>系统安装 apk 时的一个服务类，启动完成 Installer 服务之后才能启动其他的系统服务</td></tr><tr><td>ActivityManagerService</td><td>负责<strong>四</strong>大组件的启动、切换、调度。</td></tr><tr><td>PowerManagerService</td><td>计算系统中和 Power 相关的计算，然后决策系统应该如何反应</td></tr><tr><td>LightsService</td><td>管理和显示背光 LED</td></tr><tr><td>DisplayManagerService</td><td>用来管理所有显示设备</td></tr><tr><td>UserManagerService</td><td>多用户模式管理</td></tr><tr><td>SensorService</td><td>为系统提供各种感应器服务</td></tr><tr><td>PackageManagerService</td><td>用来对 apk 进行安装、解析、删除、卸载等等操作</td></tr></tbody></table>

**startCoreServices：核心服务**

<table><thead><tr><th>服务</th><th>作用</th></tr></thead><tbody><tr><td>BatteryService</td><td>管理电池相关的服务</td></tr><tr><td>UsageStatsService</td><td>收集用户使用每一个 APP 的频率、使用时常</td></tr><tr><td>WebViewUpdateService</td><td>WebView 更新服务</td></tr></tbody></table>

**startOtherServices：其他服务 (60 多种)**

<table><thead><tr><th>服务</th><th>作用</th></tr></thead><tbody><tr><td>CameraService</td><td>摄像头相关服务</td></tr><tr><td>AlarmManagerService</td><td>全局定时器管理服务</td></tr><tr><td>InputManagerService</td><td>管理输入事件</td></tr><tr><td>WindowManagerService</td><td>窗口管理服务</td></tr><tr><td>VrManagerService</td><td>VR 模式管理服务</td></tr><tr><td>BluetoothService</td><td>蓝牙管理服务</td></tr><tr><td>NotificationManagerService</td><td>通知管理服务</td></tr><tr><td>DeviceStorageMonitorService</td><td>存储相关管理服务</td></tr><tr><td>LocationManagerService</td><td>定位管理服务</td></tr><tr><td>AudioService</td><td>音频相关管理服务</td></tr><tr><td>…</td><td></td></tr></tbody></table>

都是一些 Android 必要的服务

6.2 总结
------

1.  启动 Binder 线程池，这样就可以与其他进程进行通信。
2.  创建 SystemServiceManager，其用于对系统服务进程创建、启动和生命周期管理。
3.  启动各种系统服务（引导服务、核心服务、其他服务）。

7 Launcher
==========

可能写的有点突然，这个 Launcher 其实是由 SystemServer 启动的 AMS 启动的。这个 Launcher 的作用就是来显示 已经安装的应用程序，**Lanucher 在启动过程中会请求 PackageManagerService 返回系统中已经安装的应用程序信息**，并将这些信息封装成一个快捷图标列表显示在系统屏幕上。

代码逻辑弯弯绕绕，非常长，这里仅作系统启动学习，就不写那么多了，详情可以去看这个师傅的博客

[https://blog.csdn.net/fdsafwagdagadg6576/article/details/116333916](https://blog.csdn.net/fdsafwagdagadg6576/article/details/116333916)

这边只做总结

7.1 Launcher 启动过程
-----------------

1.  SystemServer 启动 PMS，PMS 启动后会先安装好系统应用程序，在此前成功启动的 AMS 将 Launcher 给启动，在 AMS 的 systemReady 方法出启动。是在 startOtherServices 被调用的。
2.  systemReady 方法中最终是通过 Intent 进行启动的，其中 action 为：Intent.ACTION_MAIN、Category 为 Intent.CATEGORY_HOME。启动方法与普通的 activity 方式类似。

7.2 Launcher 应用图表显示过程
---------------------

1.  首先加载所有应用信息是在内部创建了一个 HandlerThread，然后通过 handler 循环调用来加载所以应用的。然后把 allApps 信息传递给一个 AllAppsRecyclerView 控件的 adapter 来进行展示的。
2.  Launcher 是用工作区的形式来显示系统安装的应用程序的快捷图标的，每一个工作区都是用来描述一个抽象桌面的，它由 n 个屏幕组成，每个屏幕又分为 n 个单元格，每个单元格用来显示一个应用程序的快捷图标。

8 总结
====

好了，这篇到这里就结束了，有不对的地方请指出，我会在看到的第一时间改正。希望对于阅读的小伙伴们有帮助。

后面应该还会有一些源码阅读的部分写出来，一般是涉及到跟安全强相关的部分。写完这些我认为关键的源码阅读分析之后，会着手多分析分析如 Magisk 原理，Xposed 相关的一些内容，并尽量从底层进行一些分析。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#基础理论](forum-161-1-117.htm)