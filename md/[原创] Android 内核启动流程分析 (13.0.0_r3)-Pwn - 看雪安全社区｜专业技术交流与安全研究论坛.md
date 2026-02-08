> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289978.htm)

> [原创] Android 内核启动流程分析 (13.0.0_r3)

[原创] Android 内核启动流程分析 (13.0.0_r3)

发表于: 17 小时前 382

[举报](javascript:void(0);)

### [原创] Android 内核启动流程分析 (13.0.0_r3)

 [![](http://passport.kanxue.com/upload/avatar/584/994584.png?1762403320)](user-home-994584.htm) [Elenia](user-home-994584.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png) 7  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 17 小时前  382

代码版本
----

> 本次分析代码都是基于 android-13.0.0_r3 版本进行分析
> 
> 主要是分析内核的启动过程，大概将各个板块在启动过程中什么时候启动，对应的函数调用链条，方便后续进行详细分析的时候有个大概的框架概念。详细的各个板块的拆解分析，留给后面的专门的解析文章。

参考文章
----

*   SELinux 详细解释: https://www.lategege.com/?p=1524#google_vignette
*   DAC,MAC 解释: https://blog.csdn.net/oDwyane03/article/details/145346354
*   zygote 与 SystemServer 各自职责: https://blog.csdn.net/zhanghui962623727/article/details/144009274

函数调用栈
-----

```
Kernel启动
  └─> /init (PID 1)
      └─> main() [system/core/init/main.cpp]
          └─> FirstStageMain() [system/core/init/first_stage_init.cpp]
              ├─> 挂载基础文件系统 (tmpfs, proc, sysfs, devpts, selinuxfs)
              ├─> 创建设备节点 (/dev/kmsg, /dev/null等)
              ├─> LoadKernelModules() - 加载内核模块
              ├─> DoFirstStageMount() - 挂载分区
              └─> execv("/system/bin/init", "selinux_setup")
                  └─> SetupSelinux() [system/core/init/selinux.cpp]
                      ├─> 加载SELinux策略
                      └─> execv("/system/bin/init", "second_stage")
                          └─> SecondStageMain() [system/core/init/init.cpp]
                              ├─> PropertyInit() - 初始化属性服务
                              ├─> MountExtraFilesystems() - 挂载额外文件系统
                              ├─> SelabelInitialize() - 初始化SELinux标签
                              ├─> LoadBootScripts() - 加载init.rc脚本
                              │   └─> ParseConfig("/system/etc/init/hw/init.rc")
                              │       └─> 解析service、on、import等section
                              ├─> QueueEventTrigger("early-init")
                              ├─> QueueEventTrigger("init")
                              ├─> QueueEventTrigger("late-init")
                              └─> 进入主循环 (epoll.Wait())
                                  └─> 根据init.rc启动服务
                                      └─> 启动Zygote进程
                                          └─> ZygoteInit.main() [frameworks/base/core/java/com/android/internal/os/ZygoteInit.java]
                                              ├─> RuntimeInit.preForkInit()
                                              ├─> preload() - 预加载类和资源
                                              │   ├─> preloadClasses() - 预加载类
                                              │   ├─> preloadResources() - 预加载资源
                                              │   └─> preloadSharedLibraries() - 预加载共享库
                                              ├─> gcAndFinalize() - GC清理
                                              ├─> forkSystemServer() - Fork SystemServer
                                              │   └─> Zygote.forkSystemServer()
                                              │       └─> handleSystemServerProcess()
                                              │           └─> ZygoteInit.zygoteInit()
                                              │               └─> RuntimeInit.applicationInit()
                                              │                   └─> findStaticMain("com.android.server.SystemServer")
                                              │                       └─> SystemServer.main()
                                              │                           └─> SystemServer.run()
                                              │                               ├─> createSystemContext() - 创建系统上下文
                                              │                               ├─> startBootstrapServices() - 启动引导服务
                                              │                               │   ├─> ActivityManagerService
                                              │                               │   ├─> PowerManagerService
                                              │                               │   ├─> DisplayManagerService
                                              │                               │   └─> PackageManagerService
                                              │                               ├─> startCoreServices() - 启动核心服务
                                              │                               │   ├─> BatteryService
                                              │                               │   ├─> UsageStatsService
                                              │                               │   └─> WebViewUpdateService
                                              │                               ├─> startOtherServices() - 启动其他服务
                                              │                               │   ├─> WindowManagerService
                                              │                               │   ├─> InputManagerService
                                              │                               │   ├─> NotificationManagerService
                                              │                               │   └─> 等等...
                                              │                               └─> startApexServices() - 启动Apex服务
                                              └─> zygoteServer.runSelectLoop() - 等待应用启动请求


```

1.  Init 进程入口：system/core/init/main.cpp
2.  第一阶段 Init：system/core/init/first_stage_init.cpp
3.  SELinux 设置：system/core/init/selinux.cpp
4.  第二阶段 Init：system/core/init/init.cpp
5.  Zygote 启动：frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
6.  RuntimeInit：frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
7.  SystemServer：frameworks/base/services/java/com/android/server/SystemServer.java

流程图
---

```
┌─────────────────────────────────────────────────────────────┐
│                    Kernel 启动                                │
│              (PID 1: /init)                                  │
└────────────────────┬──────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │   main() [main.cpp:53]      │
        │   - 设置优先级 -20          │
        │   - 路由到不同阶段           │
        └────────────┬────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ FirstStageMain()             │
        │ [first_stage_init.cpp:333]   │
        └────────────┬────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │挂载文件│  │创建设备│  │加载内核│
    │系统    │  │节点    │  │模块    │
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ execv("selinux_setup")      │
        └────────────┬────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ SetupSelinux()              │
        │ [selinux.cpp:701]           │
        └────────────┬────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │挂载缺失│  │读取策略│  │加载策略│
    │分区    │  │文件    │  │到内核  │
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ execv("second_stage")       │
        └────────────┬────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ SecondStageMain()           │
        │ [init.cpp:922]              │
        └────────────┬────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │初始化属│  │加载init│  │创建epoll│
    │性服务  │  │.rc脚本 │  │事件循环 │
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ 触发启动事件                │
        │ - early-init                │
        │ - init                      │
        │ - late-init                 │
        └────────────┬────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ 启动Zygote服务              │
        │ (通过init.rc)               │
        └────────────┬────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ ZygoteInit.main()           │
        │ [ZygoteInit.java:796]       │
        └────────────┬────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │预加载类│  │预加载资│  │预加载共│
    │和资源  │  │源      │  │享库    │
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │ forkSystemServer()          │
        └────────────┬────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
    ┌──────────┐          ┌──────────┐
    │父进程    │          │子进程    │
    │(Zygote)  │          │(System   │
    │          │          │ Server)   │
    └──────────┘          └────┬─────┘
                                 │
                                 ▼
                    ┌────────────────────────────┐
                    │ SystemServer.main()         │
                    │ [SystemServer.java:682]     │
                    └────────────┬───────────────┘
                                 │
                                 ▼
                    ┌────────────────────────────┐
                    │ SystemServer.run()          │
                    │ [SystemServer.java:790]     │
                    └────────────┬───────────────┘
                                 │
                     ┌───────────┼───────────┐
                     │           │           │
                     ▼           ▼           ▼
                ┌────────┐  ┌────────┐  ┌────────┐
                │启动引导│  │启动核心│  │启动其他│
                │服务    │  │服务    │  │服务    │
                └────────┘  └────────┘  └────────┘
                     │           │           │
                     └───────────┼───────────┘
                                 │
                                 ▼
                    ┌────────────────────────────┐
                    │ Looper.loop()              │
                    │ 进入主循环                  │
                    └────────────────────────────┘

```

Kernel(入口)
----------

```
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#elif __has_feature(hwaddress_sanitizer)
    __hwasan_set_error_report_callback(AsanReportCallback);
#endif
    // Boost prio which will be restored later
    setpriority(PRIO_PROCESS, 0, -20);
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
 
    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
 
            return SubcontextMain(argc, argv, &function_map);
        }
 
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }
 
        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }
 
    return FirstStageMain(argc, argv);
}

```

FirstStageMain(init 第一阶段)
-------------------------

1.  挂载基础文件系统（tmpfs、proc、sysfs、devpts、selinuxfs）
2.  创建设备节点（/dev/kmsg、/dev/null、/dev/random 等）
3.  加载内核模块
4.  挂载必要分区（system、vendor 等）
5.  执行 execv("/system/bin/init", "selinux_setup") 进入 SELinux 设置阶段

```
int FirstStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
 
    boot_clock::time_point start_time = boot_clock::now();
 
    std::vector> errors;
#define CHECKCALL(x) \
    if ((x) != 0) errors.emplace_back(#x " failed", errno);
 
    // Clear the umask.
    umask(0);
 
    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
    CHECKCALL(mkdir("/dev/pts", 0755));
    CHECKCALL(mkdir("/dev/socket", 0755));
    CHECKCALL(mkdir("/dev/dm-user", 0755));
    CHECKCALL(mount("devpts", "/dev/pts", "devpts", 0, NULL));
#define MAKE_STR(x) __STRING(x)
    CHECKCALL(mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
#undef MAKE_STR
    // Don't expose the raw commandline to unprivileged processes.
    CHECKCALL(chmod("/proc/cmdline", 0440));
    std::string cmdline;
    android::base::ReadFileToString("/proc/cmdline", &cmdline);
    // Don't expose the raw bootconfig to unprivileged processes.
    chmod("/proc/bootconfig", 0440);
    std::string bootconfig;
    android::base::ReadFileToString("/proc/bootconfig", &bootconfig);
    gid_t groups[] = {AID_READPROC};
    CHECKCALL(setgroups(arraysize(groups), groups));
    CHECKCALL(mount("sysfs", "/sys", "sysfs", 0, NULL));
    CHECKCALL(mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL));
 
    CHECKCALL(mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11)));
 
    if constexpr (WORLD_WRITABLE_KMSG) {
        CHECKCALL(mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11)));
    }
 
    CHECKCALL(mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8)));
    CHECKCALL(mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9)));
 
    // This is needed for log wrapper, which gets called before ueventd runs.
    CHECKCALL(mknod("/dev/ptmx", S_IFCHR | 0666, makedev(5, 2)));
    CHECKCALL(mknod("/dev/null", S_IFCHR | 0666, makedev(1, 3)));
 
    // These below mounts are done in first stage init so that first stage mount can mount
    // subdirectories of /mnt/{vendor,product}/.  Other mounts, not required by first stage mount,
    // should be done in rc files.
    // Mount staging areas for devices managed by vold
    // See storage config details at http://source.android.com/devices/storage/
    CHECKCALL(mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=1000"));
    // /mnt/vendor is used to mount vendor-specific partitions that can not be
    // part of the vendor partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/vendor", 0755));
    // /mnt/product is used to mount product-specific partitions that can not be
    // part of the product partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/product", 0755));
 
    // /debug_ramdisk is used to preserve additional files from the debug ramdisk
    CHECKCALL(mount("tmpfs", "/debug_ramdisk", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));
 
    // /second_stage_resources is used to preserve files from first to second
    // stage init
    CHECKCALL(mount("tmpfs", kSecondStageRes, "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));
 
    if (IsMicrodroid() && android::virtualization::IsOpenDiceChangesFlagEnabled()) {
        CHECKCALL(mount("tmpfs", "/microdroid_resources", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                        "mode=0750,uid=0,gid=0"));
    }
#undef CHECKCALL
 
    SetStdioToDevNull(argv);
    // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
    // talk to the outside world...
    InitKernelLogging(argv);
 
    if (!errors.empty()) {
        for (const auto& [error_string, error_errno] : errors) {
            LOG(ERROR) << error_string << " " << strerror(error_errno);
        }
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }
 
    LOG(INFO) << "init first stage started!";
 
    auto old_root_dir = std::unique_ptr{opendir("/"), closedir};
    if (!old_root_dir) {
        PLOG(ERROR) << "Could not opendir(\"/\"), not freeing ramdisk";
    }
 
    struct stat old_root_info {};
    if (stat("/", &old_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }
 
    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline, bootconfig) : 0;
    auto want_parallel =
            bootconfig.find("androidboot.load_modules_parallel = \"true\"") != std::string::npos;
 
    boot_clock::time_point module_start_time = boot_clock::now();
    int module_count = 0;
    BootMode boot_mode = GetBootMode(cmdline, bootconfig);
    if (!LoadKernelModules(boot_mode, want_console,
                           want_parallel, module_count)) {
        if (want_console != FirstStageConsoleParam::DISABLED) {
            LOG(ERROR) << "Failed to load kernel modules, starting console";
        } else {
            LOG(FATAL) << "Failed to load kernel modules";
        }
    }
    if (module_count > 0) {
        auto module_elapse_time = std::chrono::duration_cast(
                boot_clock::now() - module_start_time);
        setenv(kEnvInitModuleDurationMs, std::to_string(module_elapse_time.count()).c_str(), 1);
        LOG(INFO) << "Loaded " << module_count << " kernel modules took "
                  << module_elapse_time.count() << " ms";
    }
 
    MaybeResumeFromHibernation(bootconfig);
 
    std::unique_ptr fsm;
 
    bool created_devices = false;
    if (want_console == FirstStageConsoleParam::CONSOLE_ON_FAILURE) {
        if (!IsRecoveryMode()) {
            fsm = CreateFirstStageMount(cmdline);
            if (fsm) {
                created_devices = fsm->DoCreateDevices();
                if (!created_devices) {
                    LOG(ERROR) << "Failed to create device nodes early";
                }
            }
        }
        StartConsole(cmdline);
    }
 
    if (access(kBootImageRamdiskProp, F_OK) == 0) {
        std::string dest = GetRamdiskPropForSecondStage();
        std::string dir = android::base::Dirname(dest);
        std::error_code ec;
        if (!fs::create_directories(dir, ec) && !!ec) {
            LOG(FATAL) << "Can't mkdir " << dir << ": " << ec.message();
        }
        if (!fs::copy_file(kBootImageRamdiskProp, dest, ec)) {
            LOG(FATAL) << "Can't copy " << kBootImageRamdiskProp << " to " << dest << ": "
                       << ec.message();
        }
        LOG(INFO) << "Copied ramdisk prop to " << dest;
    }
 
    // If "/force_debuggable" is present, the second-stage init will use a userdebug
    // sepolicy and load adb_debug.prop to allow adb root, if the device is unlocked.
    if (access("/force_debuggable", F_OK) == 0) {
        constexpr const char adb_debug_prop_src[] = "/adb_debug.prop";
        constexpr const char userdebug_plat_sepolicy_cil_src[] = "/userdebug_plat_sepolicy.cil";
        std::error_code ec;  // to invoke the overloaded copy_file() that won't throw.
        if (access(adb_debug_prop_src, F_OK) == 0 &&
            !fs::copy_file(adb_debug_prop_src, kDebugRamdiskProp, ec)) {
            LOG(WARNING) << "Can't copy " << adb_debug_prop_src << " to " << kDebugRamdiskProp
                         << ": " << ec.message();
        }
        if (access(userdebug_plat_sepolicy_cil_src, F_OK) == 0 &&
            !fs::copy_file(userdebug_plat_sepolicy_cil_src, kDebugRamdiskSEPolicy, ec)) {
            LOG(WARNING) << "Can't copy " << userdebug_plat_sepolicy_cil_src << " to "
                         << kDebugRamdiskSEPolicy << ": " << ec.message();
        }
        // setenv for second-stage init to read above kDebugRamdisk* files.
        setenv("INIT_FORCE_DEBUGGABLE", "true", 1);
    }
 
    if (ForceNormalBoot(cmdline, bootconfig)) {
        mkdir("/first_stage_ramdisk", 0755);
        PrepareSwitchRoot();
        // SwitchRoot() must be called with a mount point as the target, so we bind mount the
        // target directory to itself here.
        if (mount("/first_stage_ramdisk", "/first_stage_ramdisk", nullptr, MS_BIND, nullptr) != 0) {
            PLOG(FATAL) << "Could not bind mount /first_stage_ramdisk to itself";
        }
        SwitchRoot("/first_stage_ramdisk");
    }
 
    if (IsRecoveryMode()) {
        LOG(INFO) << "First stage mount skipped (recovery mode)";
    } else {
        if (!fsm) {
            fsm = CreateFirstStageMount(cmdline);
        }
        if (!fsm) {
            LOG(FATAL) << "FirstStageMount not available";
        }
 
        if (!created_devices && !fsm->DoCreateDevices()) {
            LOG(FATAL) << "Failed to create devices required for first stage mount";
        }
 
        if (!fsm->DoFirstStageMount()) {
            LOG(FATAL) << "Failed to mount required partitions early ...";
        }
    }
 
    struct stat new_root_info {};
    if (stat("/", &new_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }
 
    if (old_root_dir && old_root_info.st_dev != new_root_info.st_dev) {
        FreeRamdisk(old_root_dir.get(), old_root_info.st_dev);
    }
 
    SetInitAvbVersionInRecovery();
 
    setenv(kEnvFirstStageStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(),
           1);
 
    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    execv(path, const_cast(args));
 
    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";
 
    return 1;
} 
```

SELinuxSetup
------------

### 调用链

```
SetupSelinux() [selinux.cpp:701]
  │
  ├─> SetStdioToDevNull()
  ├─> InitKernelLogging()
  ├─> SelinuxSetupKernelLogging() [selinux.cpp:527]
  │   └─> selinux_set_callback(SELINUX_CB_LOG, SelinuxKlogCallback)
  │
  └─> IsMicrodroid() 判断设备类型
      │
      ├─> [Microdroid 路径]
      │   └─> LoadSelinuxPolicyMicrodroid() [selinux.cpp:643]
      │       ├─> open("/system/etc/selinux/microdroid_precompiled_sepolicy", O_RDONLY)
      │       ├─> ReadFdToString() - 读取策略文件内容
      │       └─> LoadSelinuxPolicy() [selinux.cpp:632]
      │           ├─> set_selinuxmnt("/sys/fs/selinux")
      │           └─> security_load_policy(policy.data(), policy.size())
      │
      └─> [Android 设备路径]
          └─> LoadSelinuxPolicyAndroid() [selinux.cpp:676]
              │
              ├─> MountMissingSystemPartitions() [selinux.cpp:565]
              │   ├─> ReadDefaultFstab()
              │   ├─> ReadFstabFromFile("/proc/mounts")
              │   └─> fs_mgr_do_mount_one() - 挂载system_ext/product
              │
              ├─> ReadPolicy() [selinux.cpp:409]
              │   ├─> IsSplitPolicyDevice() [selinux.cpp:196]
              │   │   └─> access("/system/etc/selinux/plat_sepolicy.cil", R_OK)
              │   │
              │   ├─> OpenSplitPolicy() [selinux.cpp:224] (Split Policy)
              │   │   ├─> GetUserdebugPlatformPolicyFile() [selinux.cpp:200]
              │   │   ├─> FindPrecompiledSplitPolicy() [selinux.cpp:128]
              │   │   │   ├─> 检查 /odm/etc/selinux/precompiled_sepolicy
              │   │   │   ├─> 或 /vendor/etc/selinux/precompiled_sepolicy
              │   │   │   └─> 验证SHA256哈希匹配
              │   │   │       ├─> ReadFirstLine() 读取哈希文件
              │   │   │       └─> 比较 plat/system_ext/product 哈希
              │   │   │
              │   │   └─> 如果预编译策略不匹配，编译策略
              │   │       ├─> GetVendorMappingVersion() [selinux.cpp:182]
              │   │       ├─> 构建编译参数列表
              │   │       │   ├─> plat_sepolicy.cil
              │   │       │   ├─> vendor_sepolicy.cil
              │   │       │   ├─> system_ext_sepolicy.cil (可选)
              │   │       │   ├─> product_sepolicy.cil (可选)
              │   │       │   ├─> odm_sepolicy.cil (可选)
              │   │       │   └─> mapping文件
              │   │       └─> ForkExecveAndWaitForCompletion("/system/bin/secilc", ...)
              │   │
              │   └─> OpenMonolithicPolicy() [selinux.cpp:396] (Monolithic Policy)
              │       └─> open("/sepolicy", O_RDONLY)
              │
              ├─> SnapuserdSelinuxHelper::CreateIfNeeded()
              │   └─> snapuserd_helper->StartTransition() - 停止snapuserd避免audit
              │
              ├─> LoadSelinuxPolicy() [selinux.cpp:632]
              │   ├─> set_selinuxmnt("/sys/fs/selinux")
              │   └─> security_load_policy(policy.data(), policy.size())
              │
              └─> snapuserd_helper->FinishTransition() - 重新启动snapuserd
  │
  ├─> SelinuxSetEnforcement() [selinux.cpp:423]
  │   ├─> security_getenforce() - 获取内核enforcing状态
  │   ├─> IsEnforcing() [selinux.cpp:109]
  │   │   └─> StatusFromProperty() [selinux.cpp:98]
  │   │       └─> 检查 androidboot.selinux=permissive
  │   └─> security_setenforce() - 设置enforcing模式
  │
  ├─> [Microdroid 特殊处理]
  │   └─> IsMicrodroid() && IsOpenDiceChangesFlagEnabled()
  │       └─> selinux_android_restorecon("/microdroid_resources", RECURSE)
  │
  ├─> selinux_android_restorecon("/system/bin/init", 0)
  │   └─> 恢复init文件的SELinux上下文
  │
  └─> execv("/system/bin/init", "second_stage")
      └─> 进入第二阶段init


```

### 流程图

```
┌─────────────────────────────────────────────────────────┐
│          SetupSelinux() 入口                           │
│  [system/core/init/selinux.cpp:701]                    │
└────────────────┬──────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ 初始化日志系统  │
        │ SelinuxSetupKernelLogging()
        └────────┬───────┘
                 │
                 ▼
        ┌─────────────────────────────┐
        │  判断设备类型                │
        │  IsMicrodroid()?            │
        └──────┬──────────────┬───────┘
               │              │
        ┌──────▼──────┐  ┌───▼──────────────────┐
        │ Microdroid  │  │  Android设备         │
        │ 加载策略     │  │  LoadSelinuxPolicyAndroid()
        └─────────────┘  └──────┬───────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
        ┌──────────────────────┐  ┌──────────────────────┐
        │ 挂载缺失分区          │  │ 读取策略文件          │
        │ MountMissingSystem   │  │ ReadPolicy()         │
        │ Partitions()         │  └──────┬───────────────┘
        └──────────────────────┘         │
                                          │
                    ┌─────────────────────┴─────────────────────┐
                    │                                           │
                    ▼                                           ▼
        ┌──────────────────────┐              ┌──────────────────────┐
        │ Split Policy设备?    │              │ Monolithic Policy    │
        │ IsSplitPolicyDevice()│              │ OpenMonolithicPolicy()│
        └──────┬───────────────┘              │ 打开 /sepolicy        │
               │                               └──────────────────────┘
               ▼
        ┌──────────────────────┐
        │ OpenSplitPolicy()    │
        └──────┬───────────────┘
               │
               ├──────────────────────────────┐
               │                              │
               ▼                              ▼
        ┌──────────────────┐        ┌──────────────────────┐
        │ 查找预编译策略    │        │ 编译策略              │
        │ FindPrecompiled  │        │ 调用secic编译CIL文件  │
        │ SplitPolicy()    │        │ ForkExecveAndWaitFor │
        │                  │        │ Completion(secilc)   │
        │ 验证SHA256哈希    │        └──────────────────────┘
        └──────────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ Snapuserd过渡处理    │
        │ StartTransition()    │
        │ (停止snapuserd)      │
        └──────┬───────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ 加载策略到内核        │
        │ LoadSelinuxPolicy()  │
        │ security_load_policy()│
        └──────┬───────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ 完成Snapuserd过渡    │
        │ FinishTransition()   │
        │ (重启snapuserd)      │
        └──────┬───────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ 设置Enforcing模式    │
        │ SelinuxSetEnforcement()│
        │ security_setenforce() │
        └──────┬───────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ 恢复init文件上下文    │
        │ restorecon("/system/  │
        │ bin/init")           │
        └──────┬───────────────┘
               │
               ▼
        ┌──────────────────────┐
        │ execv到second_stage  │
        │ execv("/system/bin/  │
        │ init", "second_stage")│
        └──────────────────────┘

```

### SetupSelinux (入口)

*   加载 SELinux 策略
*   设置 SELinux 上下文
*   执行 execv("/system/bin/init", "second_stage") 进入第二阶段

```
int SetupSelinux(char** argv) {
    // 重定向标准输入输出，避免干扰启动过程
    SetStdioToDevNull(argv);
    // 初始化内核日志，确保SELinux日志能正确输出
    InitKernelLogging(argv);
 
    // 如果启用了panic时重启到bootloader，安装信号处理器
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
 
    // 记录SELinux设置开始时间，用于性能分析
    boot_clock::time_point start_time = boot_clock::now();
 
    // 设置SELinux日志回调，将SELinux日志输出到内核日志
    SelinuxSetupKernelLogging();
 
    // 根据设备类型选择不同的策略加载方式
    // Microdroid是Android虚拟化容器，使用简化的策略加载
    if (IsMicrodroid()) {
        LoadSelinuxPolicyMicrodroid();
    } else {
        LoadSelinuxPolicyAndroid();
    }
 
    // 设置SELinux enforcing模式
    // enforcing: 拒绝未授权的访问
    // permissive: 允许访问但记录日志(用于调试)
    SelinuxSetEnforcement();
 
    // Microdroid特殊处理：恢复microdroid_resources的上下文
    if (IsMicrodroid() && android::virtualization::IsOpenDiceChangesFlagEnabled()) {
        const int flags = SELINUX_ANDROID_RESTORECON_RECURSE;
        if (selinux_android_restorecon("/microdroid_resources", flags) == -1) {
            PLOG(FATAL) << "restorecon of /microdroid_resources failed";
        }
    }
 
    // 恢复init文件的SELinux上下文
    // 这是关键步骤：init进程需要从kernel domain转换到init domain
    // ext4等支持xattr的文件系统可能不需要，但ramdisk需要
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }
 
    // 记录SELinux设置完成时间，供后续性能分析使用
    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);
 
    // 执行第二阶段init，此时SELinux已完全启用
    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast(args));
 
    // execv()只在出错时返回
    PLOG(FATAL) << "execv(\"" << path << "\") failed";
    return 1;
} 
```

### SELinux 策略加载

#### 函数调用链

```
└─> IsMicrodroid() 判断设备类型
      │
      ├─> [Microdroid 路径]
      │   └─> LoadSelinuxPolicyMicrodroid() [selinux.cpp:643]
      │       ├─> open("/system/etc/selinux/microdroid_precompiled_sepolicy", O_RDONLY)
      │       ├─> ReadFdToString() - 读取策略文件内容
      │       └─> LoadSelinuxPolicy() [selinux.cpp:632]
      │           ├─> set_selinuxmnt("/sys/fs/selinux")
      │           └─> security_load_policy(policy.data(), policy.size())
      │
      └─> [Android 设备路径]
          └─> LoadSelinuxPolicyAndroid() [selinux.cpp:676]
              │
              ├─> MountMissingSystemPartitions() [selinux.cpp:565]
              │   ├─> ReadDefaultFstab()
              │   ├─> ReadFstabFromFile("/proc/mounts")
              │   └─> fs_mgr_do_mount_one() - 挂载system_ext/product
              │
              ├─> ReadPolicy() [selinux.cpp:409]
              │   ├─> IsSplitPolicyDevice() [selinux.cpp:196]
              │   │   └─> access("/system/etc/selinux/plat_sepolicy.cil", R_OK)
              │   │
              │   ├─> OpenSplitPolicy() [selinux.cpp:224] (Split Policy)
              │   │   ├─> GetUserdebugPlatformPolicyFile() [selinux.cpp:200]
              │   │   ├─> FindPrecompiledSplitPolicy() [selinux.cpp:128]
              │   │   │   ├─> 检查 /odm/etc/selinux/precompiled_sepolicy
              │   │   │   ├─> 或 /vendor/etc/selinux/precompiled_sepolicy
              │   │   │   └─> 验证SHA256哈希匹配
              │   │   │       ├─> ReadFirstLine() 读取哈希文件
              │   │   │       └─> 比较 plat/system_ext/product 哈希
              │   │   │
              │   │   └─> 如果预编译策略不匹配，编译策略
              │   │       ├─> GetVendorMappingVersion() [selinux.cpp:182]
              │   │       ├─> 构建编译参数列表
              │   │       │   ├─> plat_sepolicy.cil
              │   │       │   ├─> vendor_sepolicy.cil
              │   │       │   ├─> system_ext_sepolicy.cil (可选)
              │   │       │   ├─> product_sepolicy.cil (可选)
              │   │       │   ├─> odm_sepolicy.cil (可选)
              │   │       │   └─> mapping文件
              │   │       └─> ForkExecveAndWaitForCompletion("/system/bin/secilc", ...)
              │   │
              │   └─> OpenMonolithicPolicy() [selinux.cpp:396] (Monolithic Policy)
              │       └─> open("/sepolicy", O_RDONLY)
              │
              ├─> SnapuserdSelinuxHelper::CreateIfNeeded()
              │   └─> snapuserd_helper->StartTransition() - 停止snapuserd避免audit
              │
              ├─> LoadSelinuxPolicy() [selinux.cpp:632]
              │   ├─> set_selinuxmnt("/sys/fs/selinux")
              │   └─> security_load_policy(policy.data(), policy.size())
              │
              └─> snapuserd_helper->FinishTransition() - 重新启动snapuserd


```

#### LoadSelinuxPolicyMicrodroid（Microdroid 策略加载）

*   Microdroid 是 Android 虚拟化容器，使用简化策略
*   直接加载预编译策略文件 /system/etc/selinux/microdroid_precompiled_sepolicy
*   无需编译，流程简单

```
// Encapsulates steps to load SELinux policy in Microdroid.
// So far the process is very straightforward - just load the precompiled policy from /system.
void LoadSelinuxPolicyMicrodroid() {
    constexpr const char kMicrodroidPrecompiledSepolicy[] =
            "/system/etc/selinux/microdroid_precompiled_sepolicy";
 
    LOG(INFO) << "Opening SELinux policy from " << kMicrodroidPrecompiledSepolicy;
    unique_fd policy_fd(open(kMicrodroidPrecompiledSepolicy, O_RDONLY | O_CLOEXEC | O_NOFOLLOW));
    if (policy_fd < 0) {
        PLOG(FATAL) << "Failed to open " << kMicrodroidPrecompiledSepolicy;
    }
 
    std::string policy;
    if (!android::base::ReadFdToString(policy_fd, &policy)) {
        PLOG(FATAL) << "Failed to read policy file: " << kMicrodroidPrecompiledSepolicy;
    }
 
    LoadSelinuxPolicy(policy);
}

```

#### LoadSeLinuxPolicyAndroid(Android 策略加载)

*   先挂载缺失的分区（system_ext、product）
*   读取策略（Split Policy 或 Monolithic Policy）
*   处理 snapuserd 过渡（避免审计日志）
*   加载策略到内核
*   完成 snapuserd 过渡

```
// The SELinux setup process is carefully orchestrated around snapuserd. Policy
// must be loaded off dynamic partitions, and during an OTA, those partitions
// cannot be read without snapuserd. But, with kernel-privileged snapuserd
// running, loading the policy will immediately trigger audits.
//
// We use a five-step process to address this:
//  (1) Read the policy into a string, with snapuserd running.
//  (2) Rewrite the snapshot device-mapper tables, to generate new dm-user
//      devices and to flush I/O.
//  (3) Kill snapuserd, which no longer has any dm-user devices to attach to.
//  (4) Load the sepolicy and issue critical restorecons in /dev, carefully
//      avoiding anything that would read from /system.
//  (5) Re-launch snapuserd and attach it to the dm-user devices from step (2).
//
// After this sequence, it is safe to enable enforcing mode and continue booting.
void LoadSelinuxPolicyAndroid() {
    MountMissingSystemPartitions();
 
    LOG(INFO) << "Opening SELinux policy";
 
    // Read the policy before potentially killing snapuserd.
    std::string policy;
    ReadPolicy(&policy);
 
    auto snapuserd_helper = SnapuserdSelinuxHelper::CreateIfNeeded();
    if (snapuserd_helper) {
        // Kill the old snapused to avoid audit messages. After this we cannot read from /system
        // (or other dynamic partitions) until we call FinishTransition().
        snapuserd_helper->StartTransition();
    }
 
    LoadSelinuxPolicy(policy);
 
    if (snapuserd_helper) {
        // Before enforcing, finish the pending snapuserd transition.
        snapuserd_helper->FinishTransition();
        snapuserd_helper = nullptr;
    }
}

```

#### Split Policy 处理

```
bool OpenSplitPolicy(PolicyFile* policy_file) {
    // IMPLEMENTATION NOTE: Split policy consists of three or more CIL files:
    // * platform -- policy needed due to logic contained in the system image,
    // * vendor -- policy needed due to logic contained in the vendor image,
    // * mapping -- mapping policy which helps preserve forward-compatibility of non-platform policy
    //   with newer versions of platform policy.
    // * (optional) policy needed due to logic on product, system_ext, or odm images.
    // secilc is invoked to compile the above three policy files into a single monolithic policy
    // file. This file is then loaded into the kernel.
 
    const auto userdebug_plat_sepolicy = GetUserdebugPlatformPolicyFile();
    const bool use_userdebug_policy = userdebug_plat_sepolicy.has_value();
    if (use_userdebug_policy) {
        LOG(INFO) << "Using userdebug system sepolicy " << *userdebug_plat_sepolicy;
    }
 
    // Load precompiled policy from vendor image, if a matching policy is found there. The policy
    // must match the platform policy on the system image.
    // use_userdebug_policy requires compiling sepolicy with userdebug_plat_sepolicy.cil.
    // Thus it cannot use the precompiled policy from vendor image.
    if (!use_userdebug_policy) {
        if (auto res = FindPrecompiledSplitPolicy(); res.ok()) {
            unique_fd fd(open(res->c_str(), O_RDONLY | O_CLOEXEC | O_BINARY));
            if (fd != -1) {
                policy_file->fd = std::move(fd);
                policy_file->path = std::move(*res);
                return true;
            }
        } else {
            LOG(INFO) << res.error();
        }
    }
    // No suitable precompiled policy could be loaded
 
    LOG(INFO) << "Compiling SELinux policy";
 
    // We store the output of the compilation on /dev because this is the most convenient tmpfs
    // storage mount available this early in the boot sequence.
    char compiled_sepolicy[] = "/dev/sepolicy.XXXXXX";
    unique_fd compiled_sepolicy_fd(mkostemp(compiled_sepolicy, O_CLOEXEC));
    if (compiled_sepolicy_fd < 0) {
        PLOG(ERROR) << "Failed to create temporary file " << compiled_sepolicy;
        return false;
    }
 
    // Determine which mapping file to include
    std::string vend_plat_vers;
    if (!GetVendorMappingVersion(&vend_plat_vers)) {
        return false;
    }
    std::string plat_mapping_file("/system/etc/selinux/mapping/" + vend_plat_vers + ".cil");
 
    std::string plat_compat_cil_file("/system/etc/selinux/mapping/" + vend_plat_vers +
                                     ".compat.cil");
    if (access(plat_compat_cil_file.c_str(), F_OK) == -1) {
        plat_compat_cil_file.clear();
    }
 
    std::string system_ext_policy_cil_file("/system_ext/etc/selinux/system_ext_sepolicy.cil");
    if (access(system_ext_policy_cil_file.c_str(), F_OK) == -1) {
        system_ext_policy_cil_file.clear();
    }
 
    std::string system_ext_mapping_file("/system_ext/etc/selinux/mapping/" + vend_plat_vers +
                                        ".cil");
    if (access(system_ext_mapping_file.c_str(), F_OK) == -1) {
        system_ext_mapping_file.clear();
    }
 
    std::string system_ext_compat_cil_file("/system_ext/etc/selinux/mapping/" + vend_plat_vers +
                                           ".compat.cil");
    if (access(system_ext_compat_cil_file.c_str(), F_OK) == -1) {
        system_ext_compat_cil_file.clear();
    }
 
    std::string product_policy_cil_file("/product/etc/selinux/product_sepolicy.cil");
    if (access(product_policy_cil_file.c_str(), F_OK) == -1) {
        product_policy_cil_file.clear();
    }
 
    std::string product_mapping_file("/product/etc/selinux/mapping/" + vend_plat_vers + ".cil");
    if (access(product_mapping_file.c_str(), F_OK) == -1) {
        product_mapping_file.clear();
    }
 
    std::string vendor_policy_cil_file("/vendor/etc/selinux/vendor_sepolicy.cil");
    if (access(vendor_policy_cil_file.c_str(), F_OK) == -1) {
        LOG(ERROR) << "Missing " << vendor_policy_cil_file;
        return false;
    }
 
    std::string plat_pub_versioned_cil_file("/vendor/etc/selinux/plat_pub_versioned.cil");
    if (access(plat_pub_versioned_cil_file.c_str(), F_OK) == -1) {
        LOG(ERROR) << "Missing " << plat_pub_versioned_cil_file;
        return false;
    }
 
    // odm_sepolicy.cil is default but optional.
    std::string odm_policy_cil_file("/odm/etc/selinux/odm_sepolicy.cil");
    if (access(odm_policy_cil_file.c_str(), F_OK) == -1) {
        odm_policy_cil_file.clear();
    }
    const std::string version_as_string = std::to_string(SEPOLICY_VERSION);
 
    std::vector genfs_cil_files;
 
    int vendor_genfs_version = get_genfs_labels_version();
    std::string genfs_cil_file =
            std::format("/system/etc/selinux/plat_sepolicy_genfs_{}.cil", vendor_genfs_version);
    if (access(genfs_cil_file.c_str(), F_OK) != 0) {
        LOG(INFO) << "Missing " << genfs_cil_file << "; skipping";
        genfs_cil_file.clear();
    } else {
        LOG(INFO) << "Using " << genfs_cil_file << " for genfs labels";
    }
 
    // clang-format off
    std::vector compile_args {
        "/system/bin/secilc",
        use_userdebug_policy ? *userdebug_plat_sepolicy : plat_policy_cil_file,
        "-m", "-M", "true", "-G", "-N",
        "-c", version_as_string.c_str(),
        plat_mapping_file.c_str(),
        "-o", compiled_sepolicy,
        // We don't care about file_contexts output by the compiler
        "-f", "/sys/fs/selinux/null",  // /dev/null is not yet available
    };
    // clang-format on
 
    if (!plat_compat_cil_file.empty()) {
        compile_args.push_back(plat_compat_cil_file.c_str());
    }
    if (!system_ext_policy_cil_file.empty()) {
        compile_args.push_back(system_ext_policy_cil_file.c_str());
    }
    if (!system_ext_mapping_file.empty()) {
        compile_args.push_back(system_ext_mapping_file.c_str());
    }
    if (!system_ext_compat_cil_file.empty()) {
        compile_args.push_back(system_ext_compat_cil_file.c_str());
    }
    if (!product_policy_cil_file.empty()) {
        compile_args.push_back(product_policy_cil_file.c_str());
    }
    if (!product_mapping_file.empty()) {
        compile_args.push_back(product_mapping_file.c_str());
    }
    if (!plat_pub_versioned_cil_file.empty()) {
        compile_args.push_back(plat_pub_versioned_cil_file.c_str());
    }
    if (!vendor_policy_cil_file.empty()) {
        compile_args.push_back(vendor_policy_cil_file.c_str());
    }
    if (!odm_policy_cil_file.empty()) {
        compile_args.push_back(odm_policy_cil_file.c_str());
    }
    if (!genfs_cil_file.empty()) {
        compile_args.push_back(genfs_cil_file.c_str());
    }
    compile_args.push_back(nullptr);
 
    if (!ForkExecveAndWaitForCompletion(compile_args[0], (char**)compile_args.data())) {
        unlink(compiled_sepolicy);
        return false;
    }
    unlink(compiled_sepolicy);
 
    policy_file->fd = std::move(compiled_sepolicy_fd);
    policy_file->path = compiled_sepolicy;
    return true;
} 
```

#### **ReadPolicy(策略文件读取)**

```
void ReadPolicy(std::string* policy) {
    PolicyFile policy_file;
 
    bool ok = IsSplitPolicyDevice() ? OpenSplitPolicy(&policy_file)
                                    : OpenMonolithicPolicy(&policy_file);
    if (!ok) {
        LOG(FATAL) << "Unable to open SELinux policy";
    }
 
    if (!android::base::ReadFdToString(policy_file.fd, policy)) {
        PLOG(FATAL) << "Failed to read policy file: " << policy_file.path;
    }
}

```

### Init 和 Zygote 的 DAC 权限 (Linux 权限控制)

> DAC 基于 UID/GID 的传统 Linux 权限控制，通过文件所有者和权限位控制访问。
> 
> 比如我们最重要的进程的权限设置

Init 进程中的 DAC 权限设置 - system/core/init/service_utils.cpp:235

```
if (attr.gid) {
    if (setgid(attr.gid) != 0) {
        return ErrnoError() << "setgid failed";
    }
}
if (setgroups(attr.supp_gids.size(), const_cast(&attr.supp_gids[0])) != 0) {
    return ErrnoError() << "setgroups failed";
}
if (attr.uid()) {
    if (setuid(attr.uid()) != 0) {
        return ErrnoError() << "setuid failed";
    }
} 
```

Zygote 中的 DAC 权限设置 - frameworks/base/core/jni/com_android_internal_os_Zygote.cpp:1891

```
if (setresgid(gid, gid, gid) == -1) {
    fail_fn(CREATE_ERROR("setresgid(%d) failed: %s", gid, strerror(errno)));
}
 
// Must be called when the new process still has CAP_SYS_ADMIN, in this case,
// before changing uid from 0, which clears capabilities.  The other
// alternative is to call prctl(PR_SET_NO_NEW_PRIVS, 1) afterward, but that
// breaks SELinux domain transition (see b/71859146).  As the result,
// privileged syscalls used below still need to be accessible in app process.
SetUpSeccompFilter(uid, is_child_zygote);
 
// Must be called before losing the permission to set scheduler policy.
SetSchedulerPolicy(fail_fn, is_top_app);
 
if (setresuid(uid, uid, uid) == -1) {
    fail_fn(CREATE_ERROR("setresuid(%d) failed: %s", uid, strerror(errno)));
}

```

#### 调用链

```
Service::Start() [service.cpp:586]
  └─> Service::Fork()
      └─> Service::SetProcessAttributesAndCaps()
          └─> SetProcessAttributes() [service_utils.cpp:235]
              ├─> setgid() - 设置组ID
              ├─> setgroups() - 设置补充组
              └─> setuid() - 设置用户ID (失去root权限)
 
Zygote.forkAndSpecialize()
  └─> SpecializeCommon() [Zygote.cpp:1891]
      ├─> SetGids() - setgroups()
      ├─> setresgid() - 设置GID
      ├─> SetUpSeccompFilter() - 必须在setuid前
      └─> setresuid() - 设置UID (失去root权限)

```

### Init 和 zygote 的 MAC 权限 (SELinux 权限控制)

Zygote 中的 MAC 权限设置 - frameworks/base/core/jni/com_android_internal_os_Zygote.cpp:2133

```
const char* se_info_ptr = se_info.has_value() ? se_info.value().c_str() : nullptr;
 
if (selinux_android_setcontext(uid, is_system_server, se_info_ptr, nice_name_ptr) == -1) {
    fail_fn(CREATE_ERROR("selinux_android_setcontext(%d, %d, \"%s\", \"%s\") failed", uid,
                         is_system_server, se_info_ptr, nice_name_ptr));
}

```

Init 中 MAC 权限设置

```
if (!seclabel_.empty()) {
    if (setexeccon(seclabel_.c_str()) < 0) {
        PLOG(FATAL) << "cannot setexeccon('" << seclabel_ << "') for " << name_;
    }
}

```

SecondStageMain(init 第二阶段)
--------------------------

### 调用链

```
SecondStageMain()
├─> InstallRebootSignalHandlers() - 安装重启信号处理器
├─> SetStdioToDevNull() - 重定向标准输入输出
├─> InitKernelLogging() - 初始化内核日志
├─> SelinuxSetupKernelLogging() - 设置SELinux日志
├─> setenv("PATH", _PATH_DEFPATH, 1) - 设置PATH
│
├─> 设置信号处理
│   ├─> sigaction(SIGPIPE, ...) - 忽略SIGPIPE
│   └─> WriteFile("/proc/1/oom_score_adj", ...) - 设置OOM调整值
│
├─> PropertyInit() - 初始化属性服务
│   ├─> property_set_fd = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK)
│   ├─> property_set("ro.property_service.version", "2")
│   └─> StartPropertyServiceThread() - 启动属性服务线程
│
├─> MountExtraFilesystems() - 挂载额外文件系统
│   ├─> mount("tmpfs", "/apex", "tmpfs", ...)
│   ├─> mount("tmpfs", "/bootstrap-apex", "tmpfs", ...) (如果需要)
│   └─> mount("tmpfs", "/linkerconfig", "tmpfs", ...)
│
├─> SelabelInitialize() - 初始化SELinux标签
├─> SelinuxRestoreContext() - 恢复SELinux上下文
│
├─> Epoll epoll - 创建epoll对象
│   ├─> epoll.Open() - 打开epoll
│   └─> epoll.SetFirstCallback(ReapAnyOutstandingChildren) - 设置回调
│
├─> InstallSignalFdHandler() - 安装信号文件描述符处理器
│   ├─> RegisterSignalFd(epoll, SIGCHLD, Service::GetSigchldFd())
│   └─> CreateAndRegisterSignalFd(epoll, SIGTERM) (如果不可重启)
│
├─> InstallInitNotifier() - 安装init通知器
│   └─> eventfd(0, EFD_CLOEXEC) - 创建eventfd
│
├─> StartPropertyService() - 启动属性服务
│
├─> RecordStageBoottimes() - 记录启动时间
│
├─> fs_mgr_vendor_overlay_mount_all() - 挂载vendor overlay
├─> export_oem_lock_status() - 导出OEM锁定状态
├─> MountHandler mount_handler(&epoll) - 创建挂载处理器
├─> SetUsbController() - 设置USB控制器
├─> SetKernelVersion() - 设置内核版本属性
│
├─> SetupMountNamespaces() - 设置挂载命名空间
│
├─> InitializeSubcontext() - 初始化子上下文
│
├─> LoadBootScripts() - 加载启动脚本
│   ├─> CreateParser() - 创建解析器
│   │   ├─> AddSectionParser("service", ServiceParser)
│   │   ├─> AddSectionParser("on", ActionParser)
│   │   └─> AddSectionParser("import", ImportParser)
│   │
│   └─> ParseConfig() - 解析配置文件
│       ├─> ParseConfig("/system/etc/init/hw/init.rc")
│       ├─> ParseConfig("/system/etc/init")
│       ├─> ParseConfig("/system_ext/etc/init")
│       ├─> ParseConfig("/vendor/etc/init")
│       ├─> ParseConfig("/odm/etc/init")
│       └─> ParseConfig("/product/etc/init")
│
├─> QueueBuiltinAction() - 队列内置动作
│   ├─> SetupCgroupsAction - 设置cgroups
│   ├─> SetKptrRestrictAction - 设置kptr_restrict
│   ├─> TestPerfEventSelinuxAction - 测试perf事件SELinux
│   ├─> ConnectEarlyStageSnapuserdAction - 连接早期snapuserd
│   └─> wait_for_coldboot_done_action - 等待coldboot完成
│
├─> QueueEventTrigger() - 队列事件触发器
│   ├─> "early-init"
│   ├─> "init"
│   └─> "late-init" (或 "charger" 如果是充电模式)
│
├─> QueueBuiltinAction(queue_property_triggers_action) - 队列属性触发器
│
└─> 主循环 (while(true))
    ├─> shutdown_state.CheckShutdown() - 检查关机命令
    ├─> am.ExecuteOneCommand() - 执行一个命令
    ├─> HandleProcessActions() - 处理进程动作
    │   ├─> 检查服务超时
    │   └─> 重启需要重启的服务
    ├─> epoll.Wait() - 等待事件
    ├─> HandleControlMessages() - 处理控制消息
    └─> SetUsbController() - 更新USB控制器


```

### SecondStageMain(入口)

```
初始化属性服务（PropertyInit）
挂载额外文件系统（/apex、/linkerconfig 等）
初始化 SELinux 上下文
创建 epoll 事件循环
加载 init.rc 脚本（LoadBootScripts）
触发启动事件（early-init、init、late-init）
进入主循环，处理服务启动和事件
int SecondStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
 
    // No threads should be spin up until signalfd
    // is registered. If the threads are indeed required,
    // each of these threads _should_ make sure SIGCHLD signal
    // is blocked. See b/223076262
    boot_clock::time_point start_time = boot_clock::now();
 
    trigger_shutdown = [](const std::string& command) { shutdown_state.TriggerShutdown(command); };
 
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";
 
    SelinuxSetupKernelLogging();
 
    // Update $PATH in the case the second stage init is newer than first stage init, where it is
    // first set.
    if (setenv("PATH", _PATH_DEFPATH, 1) != 0) {
        PLOG(FATAL) << "Could not set $PATH to '" << _PATH_DEFPATH << "' in second stage";
    }
 
    // Init should not crash because of a dependence on any other process, therefore we ignore
    // SIGPIPE and handle EPIPE at the call site directly.  Note that setting a signal to SIG_IGN
    // is inherited across exec, but custom signal handlers are not.  Since we do not want to
    // ignore SIGPIPE for child processes, we set a no-op function for the signal handler instead.
    {
        struct sigaction action = {.sa_flags = SA_RESTART};
        action.sa_handler = [](int) {};
        sigaction(SIGPIPE, &action, nullptr);
    }
 
    // Set init and its forked children's oom_adj.
    if (auto result =
                WriteFile("/proc/1/oom_score_adj", StringPrintf("%d", DEFAULT_OOM_SCORE_ADJUST));
        !result.ok()) {
        LOG(ERROR) << "Unable to write " << DEFAULT_OOM_SCORE_ADJUST
                   << " to /proc/1/oom_score_adj: " << result.error();
    }
 
    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
 
    // See if need to load debug props to allow adb root, when the device is unlocked.
    const char* force_debuggable_env = getenv("INIT_FORCE_DEBUGGABLE");
    bool load_debug_prop = false;
    if (force_debuggable_env && AvbHandle::IsDeviceUnlocked()) {
        load_debug_prop = "true"s == force_debuggable_env;
    }
    unsetenv("INIT_FORCE_DEBUGGABLE");
 
    // Umount the debug ramdisk so property service doesn't read .prop files from there, when it
    // is not meant to.
    if (!load_debug_prop) {
        UmountDebugRamdisk();
    }
 
    PropertyInit();
 
    // Umount second stage resources after property service has read the .prop files.
    UmountSecondStageRes();
 
    // Umount the debug ramdisk after property service has read the .prop files when it means to.
    if (load_debug_prop) {
        UmountDebugRamdisk();
    }
 
    // Mount extra filesystems required during second stage init
    MountExtraFilesystems();
 
    // Now set up SELinux for second stage.
    SelabelInitialize();
    SelinuxRestoreContext();
 
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }
 
    // We always reap children before responding to the other pending functions. This is to
    // prevent a race where other daemons see that a service has exited and ask init to
    // start it again via ctl.start before init has reaped it.
    epoll.SetFirstCallback(ReapAnyOutstandingChildren);
 
    InstallSignalFdHandler(&epoll);
    InstallInitNotifier(&epoll);
    StartPropertyService(&property_fd);
 
    // Make the time that init stages started available for bootstat to log.
    RecordStageBoottimes(start_time);
 
    // Set libavb version for Framework-only OTA match in Treble build.
    if (const char* avb_version = getenv("INIT_AVB_VERSION"); avb_version != nullptr) {
        SetProperty("ro.boot.avb_version", avb_version);
    }
    unsetenv("INIT_AVB_VERSION");
 
    fs_mgr_vendor_overlay_mount_all();
    export_oem_lock_status();
    MountHandler mount_handler(&epoll);
    SetUsbController();
    SetKernelVersion();
 
    const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
    Action::set_function_map(&function_map);
 
    if (!SetupMountNamespaces()) {
        PLOG(FATAL) << "SetupMountNamespaces failed";
    }
 
    InitializeSubcontext();
 
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();
 
    LoadBootScripts(am, sm);
 
    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();
 
    // Make the GSI status available before scripts start running.
    auto is_running = android::gsi::IsGsiRunning() ? "1" : "0";
    SetProperty(gsi::kGsiBootedProp, is_running);
    auto is_installed = android::gsi::IsGsiInstalled() ? "1" : "0";
    SetProperty(gsi::kGsiInstalledProp, is_installed);
    if (android::gsi::IsGsiRunning()) {
        std::string dsu_slot;
        if (android::gsi::GetActiveDsu(&dsu_slot)) {
            SetProperty(gsi::kDsuSlotProp, dsu_slot);
        }
    }
 
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    am.QueueBuiltinAction(TestPerfEventSelinuxAction, "TestPerfEventSelinux");
    am.QueueEventTrigger("early-init");
    am.QueueBuiltinAction(ConnectEarlyStageSnapuserdAction, "ConnectEarlyStageSnapuserd");
 
    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
    Keychords keychords;
    am.QueueBuiltinAction(
            [&epoll, &keychords](const BuiltinArguments& args) -> Result {
                for (const auto& svc : ServiceList::GetInstance()) {
                    keychords.Register(svc->keycodes());
                }
                keychords.Start(&epoll, HandleKeychord);
                return {};
            },
            "KeychordInit");
 
    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");
 
    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }
 
    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
 
    // Restore prio before main loop
    setpriority(PRIO_PROCESS, 0, 0);
    while (true) {
        // By default, sleep until something happens. Do not convert far_future into
        // std::chrono::milliseconds because that would trigger an overflow. The unit of boot_clock
        // is 1ns.
        const boot_clock::time_point far_future = boot_clock::time_point::max();
        boot_clock::time_point next_action_time = far_future;
 
        auto shutdown_command = shutdown_state.CheckShutdown();
        if (shutdown_command) {
            LOG(INFO) << "Got shutdown_command '" << *shutdown_command
                      << "' Calling HandlePowerctlMessage()";
            HandlePowerctlMessage(*shutdown_command);
        }
 
        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) {
                next_action_time = boot_clock::now();
            }
        }
        // Since the above code examined pending actions, no new actions must be
        // queued by the code between this line and the Epoll::Wait() call below
        // without calling WakeMainInitThread().
        if (!IsShuttingDown()) {
            auto next_process_action_time = HandleProcessActions();
 
            // If there's a process that needs restarting, wake up in time for that.
            if (next_process_action_time) {
                next_action_time = std::min(next_action_time, *next_process_action_time);
            }
        }
 
        std::optional epoll_timeout;
        if (next_action_time != far_future) {
            epoll_timeout = std::chrono::ceil(
                    std::max(next_action_time - boot_clock::now(), 0ns));
        }
        auto epoll_result = epoll.Wait(epoll_timeout);
        if (!epoll_result.ok()) {
            LOG(ERROR) << epoll_result.error();
        }
        if (!IsShuttingDown()) {
            HandleControlMessages();
            SetUsbController();
        }
    }
 
    return 0;
} 
```

Zygote
------

> zygote 进程是 Android 系统中的一个关键组件，主要负责应用程序的启动和管理。
> 
> 在启动时，zygote 会预加载一些常用的类和资源，以提高后续应用程序的启动速度。这种预加载机制可以减少内存使用和启动时间。
> 
> zygote 使用 fork 机制来创建新的应用程序进程。当应用程序需要启动时，zygote 会复制自身的内存空间，从而快速生成新的进程。这种方式比传统的进程创建方式更高效。

### 调用链

```
ZygoteInit.main()
├─> ZygoteHooks.startZygoteNoThreadCreation() - 禁止线程创建
├─> Os.setpgid(0, 0) - 设置进程组ID
│
├─> RuntimeInit.preForkInit() - Fork前初始化
│   ├─> RuntimeInit.enableDdms() - 启用DDMS
│   └─> MimeMap.setDefaultSupplier(DefaultMimeMapFactory::create) - 设置MIME映射
│
├─> 解析命令行参数
│   ├─> start-system-server - 是否启动SystemServer
│   ├─> --enable-lazy-preload - 是否启用延迟预加载
│   ├─> --abi-list= - ABI列表
│   └─> --socket-name= - Socket名称
│
├─> preload() - 预加载 (如果未启用lazy preload)
│   ├─> beginPreload() - 开始预加载
│   │   └─> ZygoteHooks.onBeginPreload()
│   │
│   ├─> preloadClasses() - 预加载类
│   │   ├─> 读取 /system/etc/preloaded-classes
│   │   ├─> Os.setregid(ROOT_GID, UNPRIVILEGED_GID) - 降权
│   │   ├─> 循环加载每个类
│   │   │   └─> Class.forName(line, true, null)
│   │   ├─> runtime.preloadDexCaches() - 预填充Dex缓存
│   │   └─> Os.setreuid(ROOT_UID, ROOT_UID) - 恢复root权限
│   │
│   ├─> cacheNonBootClasspathClassLoaders() - 缓存非启动类路径类加载器
│   │   ├─> 创建SharedLibraryInfo列表
│   │   └─> ApplicationLoaders.getDefault().createAndCacheNonBootclasspathSystemClassLoaders()
│   │
│   ├─> Resources.preloadResources() - 预加载资源
│   │
│   ├─> nativePreloadAppProcessHALs() - 预加载HAL
│   │
│   ├─> maybePreloadGraphicsDriver() - 预加载图形驱动
│   │   └─> nativePreloadGraphicsDriver()
│   │
│   ├─> preloadSharedLibraries() - 预加载共享库
│   │   ├─> System.loadLibrary("android")
│   │   ├─> System.loadLibrary("jnigraphics")
│   │   └─> System.loadLibrary("compiler_rt") (如果启用)
│   │
│   ├─> preloadTextResources() - 预加载文本资源
│   │   ├─> Hyphenator.init()
│   │   └─> TextView.preloadFontCache()
│   │
│   ├─> HttpEngine.preload() (如果启用)
│   │
│   ├─> WebViewFactory.prepareWebViewInZygote()
│   │
│   ├─> endPreload() - 结束预加载
│   │   └─> ZygoteHooks.onEndPreload()
│   │
│   └─> warmUpJcaProviders() - 预热JCA提供者
│       ├─> AndroidKeyStoreProvider.install()
│       └─> Security.getProviders()[].warmUpServiceProvision()
│
├─> gcAndFinalize() - GC清理
│   └─> ZygoteHooks.gcAndFinalize()
│
├─> Zygote.initNativeState(isPrimaryZygote) - 初始化native状态
│
├─> ZygoteHooks.stopZygoteNoThreadCreation() - 允许创建线程
│
├─> new ZygoteServer(isPrimaryZygote) - 创建Zygote服务器
│   ├─> Zygote.getUsapPoolEventFD()
│   ├─> Zygote.createManagedSocketFromInitSocket(PRIMARY_SOCKET_NAME)
│   └─> Zygote.createManagedSocketFromInitSocket(USAP_POOL_PRIMARY_SOCKET_NAME)
│
├─> forkSystemServer() - Fork SystemServer (如果startSystemServer=true)
│   ├─> 计算capabilities
│   ├─> 构建SystemServer启动参数
│   ├─> ZygoteArguments.getInstance() - 解析参数
│   ├─> Zygote.applyDebuggerSystemProperty()
│   ├─> Zygote.applyInvokeWithSystemProperty()
│   ├─> 设置内存标签级别 (MTE)
│   ├─> Zygote.forkSystemServer() - Fork SystemServer
│   │   └─> nativeForkSystemServer() - JNI调用fork
│   │       └─> ForkCommon() - 通用fork逻辑
│   │           ├─> 设置seccomp过滤器
│   │           ├─> 设置SELinux上下文
│   │           ├─> 设置UID/GID
│   │           └─> 返回pid (0=子进程, >0=父进程)
│   │
│   └─> handleSystemServerProcess() - 处理SystemServer进程 (子进程)
│       ├─> Os.umask(S_IRWXG | S_IRWXO)
│       ├─> Process.setArgV0("system_server")
│       ├─> prepareSystemServerProfile() (如果启用profiling)
│       ├─> getOrCreateSystemServerClassLoader()
│       └─> ZygoteInit.zygoteInit()
│           ├─> RuntimeInit.redirectLogStreams()
│           ├─> RuntimeInit.commonInit()
│           ├─> ZygoteInit.nativeZygoteInit()
│           │   └─> JNI调用初始化Binder线程池
│           └─> RuntimeInit.applicationInit()
│               └─> findStaticMain("com.android.server.SystemServer")
│                   └─> 返回MethodAndArgsCaller
│                       └─> SystemServer.main()
│
└─> zygoteServer.runSelectLoop(abiList) - 进入select循环等待应用启动请求
    ├─> acceptCommandPeer() - 接受连接
    ├─> processCommand() - 处理命令
    │   └─> Zygote.forkAndSpecialize() - Fork应用进程
    └─> 循环处理


```

### 流程图

```
┌─────────────────────────────────────────────────────────┐
│          ZygoteInit.main() 入口                          │
│  [frameworks/base/core/java/com/android/internal/os/    │
│   ZygoteInit.java:796]                                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ 禁止线程创建    │
        │ startZygoteNo  │
        │ ThreadCreation │
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────┐
        │ 设置进程组     │
        │ setpgid(0,0)   │
        └────────┬───────┘
                 │
                 ▼
        ┌─────────────────────────────┐
        │ RuntimeInit.preForkInit()   │
        │ - 启用DDMS                  │
        │ - 设置MimeMap               │
        └────────┬────────────────────┘
                 │
                 ▼
        ┌─────────────────────────────┐
        │ 解析命令行参数               │
        │ - start-system-server        │
        │ - --abi-list                 │
        │ - --socket-name              │
        └────────┬────────────────────┘
                 │
                 ▼
        ┌─────────────────────────────┐
        │ preload() 预加载            │
        │ (如果未启用lazy preload)     │
        └────────┬────────────────────┘
                 │
                 ├──────────────────────────────────────────┐
                 │                                          │
                 ▼                                          ▼
        ┌──────────────────┐              ┌──────────────────────────┐
        │ beginPreload()   │              │ preloadClasses()         │
        │ ZygoteHooks.on  │              │ - 读取preloaded-classes  │
        │ BeginPreload()  │              │ - 降权到unprivileged     │
        └──────────────────┘              │ - Class.forName()循环    │
                 │                         │ - preloadDexCaches()     │
                 │                         │ - 恢复root权限           │
                 │                         └────────┬─────────────────┘
                 │                                    │
                 ▼                                    ▼
        ┌──────────────────┐              ┌──────────────────────────┐
        │ preloadResources │              │ cacheNonBootClasspath    │
        │ Resources.preload│              │ ClassLoaders()           │
        │ Resources()      │              └────────┬─────────────────┘
        └──────────────────┘                        │
                 │                                    │
                 ▼                                    ▼
        ┌──────────────────┐              ┌──────────────────────────┐
        │ preloadShared    │              │ preloadTextResources()    │
        │ Libraries()      │              │ - Hyphenator.init()      │
        │ - libandroid     │              │ - TextView.preloadFont  │
        │ - libjnigraphics │              └────────┬─────────────────┘
        └──────────────────┘                        │
                 │                                    │
                 ▼                                    ▼
        ┌──────────────────┐              ┌──────────────────────────┐
        │ maybePreload     │              │ warmUpJcaProviders()     │
        │ GraphicsDriver() │              │ - AndroidKeyStoreProvider│
        └──────────────────┘              └────────┬─────────────────┘
                 │                                    │
                 └────────────────┬───────────────────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ endPreload()   │
                         │ ZygoteHooks.on │
                         │ EndPreload()   │
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ gcAndFinalize()│
                         │ 执行GC清理     │
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ initNativeState│
                         │ 初始化native   │
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ stopZygoteNo   │
                         │ ThreadCreation │
                         │ 允许创建线程   │
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ new ZygoteServer│
                         │ 创建socket     │
                         └────────┬───────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
        ┌──────────────────────┐    ┌──────────────────────┐
        │ forkSystemServer()   │    │ runSelectLoop()      │
        │ (如果startSystemServer)│    │ 等待应用启动请求     │
        └────────┬─────────────┘    └──────────────────────┘
                 │
                 ├─────────────────────────────────────┐
                 │                                     │
                 ▼                                     ▼
        ┌──────────────────┐              ┌──────────────────────┐
        │ 父进程 (Zygote)  │              │ 子进程 (SystemServer)│
        │ 返回null         │              │ pid == 0             │
        │ 继续运行         │              └────────┬─────────────┘
        └──────────────────┘                       │
                                                    ▼
                                        ┌──────────────────────┐
                                        │ handleSystemServer   │
                                        │ Process()            │
                                        │ - 设置umask          │
                                        │ - 创建ClassLoader    │
                                        └────────┬─────────────┘
                                                 │
                                                 ▼
                                        ┌──────────────────────┐
                                        │ ZygoteInit.zygoteInit│
                                        │ - commonInit()       │
                                        │ - nativeZygoteInit() │
                                        └────────┬─────────────┘
                                                 │
                                                 ▼
                                        ┌──────────────────────┐
                                        │ RuntimeInit.         │
                                        │ applicationInit()    │
                                        │ - findStaticMain()   │
                                        └────────┬─────────────┘
                                                 │
                                                 ▼
                                        ┌──────────────────────┐
                                        │ SystemServer.main()  │
                                        │ 启动SystemServer     │
                                        └──────────────────────┘

```

### Zygote.main

1.  预加载类和资源（preload）
2.  创建 ZygoteServer，监听 socket
3.  Fork SystemServer（如果参数包含 start-system-server）
4.  进入 select 循环 (runSelectLoop())，等待应用启动请求

```
@UnsupportedAppUsage
public static void main(String[] argv) {
    ZygoteServer zygoteServer = null;
    // Mark zygote start. This ensures that thread creation will throw
    // an error.
    ZygoteHooks.startZygoteNoThreadCreation();
    // Zygote goes into its own process group.
    try {
        Os.setpgid(0, 0);
    } catch (ErrnoException ex) {
        throw new RuntimeException("Failed to setpgid(0,0)", ex);
    }
    Runnable caller;
    try {
        // Store now for StatsLogging later.
        final long startTime = SystemClock.elapsedRealtime();
        final boolean isRuntimeRestarted = "1".equals(
                SystemProperties.get("sys.boot_completed"));
        String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
        TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                Trace.TRACE_TAG_DALVIK);
        bootTimingsTraceLog.traceBegin("ZygoteInit");
        
        // Fork前初始化
        RuntimeInit.preForkInit();
        // 解析命令行参数
        boolean startSystemServer = false;
        String zygoteSocketName = "zygote";
        String abiList = null;
        boolean enableLazyPreload = false;
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if ("--enable-lazy-preload".equals(argv[i])) {
                enableLazyPreload = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }
        final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
        
        if (abiList == null) {
            throw new RuntimeException("No ABI list supplied.");
        }
        // 预加载（如果未启用lazy preload）
        if (!enableLazyPreload) {
            bootTimingsTraceLog.traceBegin("ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());
            preload(bootTimingsTraceLog);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());
            bootTimingsTraceLog.traceEnd(); // ZygotePreload
        }
        // GC清理
        bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
        gcAndFinalize();
        bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC
        bootTimingsTraceLog.traceEnd(); // ZygoteInit
        // 初始化native状态
        Zygote.initNativeState(isPrimaryZygote);
        // 允许创建线程
        ZygoteHooks.stopZygoteNoThreadCreation();
        // 创建Zygote服务器
        zygoteServer = new ZygoteServer(isPrimaryZygote);
        // Fork SystemServer
        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }
        Log.i(TAG, "Accepting command socket connections");
        // 进入select循环，等待应用启动请求
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with fatal exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }
    // 子进程执行命令
    if (caller != null) {
        caller.run();
    }
}


```

### preLoad

> 预加载过程

```
    static void preload(TimingsTraceLog bootTimingsTraceLog) {
        Log.d(TAG, "begin preload");
        bootTimingsTraceLog.traceBegin("BeginPreload");
        beginPreload();
        bootTimingsTraceLog.traceEnd(); // BeginPreload
        bootTimingsTraceLog.traceBegin("PreloadClasses");
        preloadClasses();
        bootTimingsTraceLog.traceEnd(); // PreloadClasses
        bootTimingsTraceLog.traceBegin("CacheNonBootClasspathClassLoaders");
        cacheNonBootClasspathClassLoaders();
        bootTimingsTraceLog.traceEnd(); // CacheNonBootClasspathClassLoaders
        bootTimingsTraceLog.traceBegin("PreloadResources");
        Resources.preloadResources();
        bootTimingsTraceLog.traceEnd(); // PreloadResources
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadAppProcessHALs");
        nativePreloadAppProcessHALs();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PreloadGraphicsDriver");
        maybePreloadGraphicsDriver();
        Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
        preloadSharedLibraries();
        preloadTextResources();

        // TODO: remove the try/catch and the flag read as soon as the flag is ramped and 25Q2
        // starts building from source.
        if (preloadHttpengineInZygote()) {
            try {
                HttpEngine.preload();
            } catch (NoSuchMethodError e){
                // The flag protecting this API is not an exported
                // flag because ZygoteInit happens before the
                // system service has initialized the flag which means
                // that we can't query the real value of the flag
                // from the tethering module. In order to avoid crashing
                // in the case where we have (new zygote, old tethering).
                // we catch the NoSuchMethodError and just log.
                Log.d(TAG, "HttpEngine.preload() threw " + e);
            }
        }
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        endPreload();
        warmUpJcaProviders();
        Log.d(TAG, "end preload");

        sPreloadComplete = true;
    }


```

*   beginPreload()
    *   ZygoteHooks.onBeginPreload()：标记预加载开始
*   preloadClasses()
    *   读取 /system/etc/preloaded-classes
    *   临时降权到 UNPRIVILEGED_UID/GID（9999）
    *   循环加载每个类：Class.forName(line, true, null)
    *   预填充 Dex 缓存：runtime.preloadDexCaches()
    *   恢复 root 权限
*   cacheNonBootClasspathClassLoaders()
    *   缓存非启动类路径的 ClassLoader（如 HIDL、WindowManager Extensions）
*   PreloadResources()
    *   Resources.preloadResources()：预加载系统资源
*   PreloadAppProcessHALs()
    *   nativePreloadAppProcessHALs()：预加载 HAL
*   PreloadGraphicsDriver()
    *   maybePreloadGraphicsDriver()：预加载图形驱动（OpenGL/Vulkan）
*   preloadSharedLibraries()
    *   System.loadLibrary("android")
    *   System.loadLibrary("jnigraphics")
    *   System.loadLibrary("compiler_rt")（如果启用）
*   preloadTextResources()
    *   Hyphenator.init()：连字符处理
    *   TextView.preloadFontCache()：字体缓存
*   HttpEngine.preload()（可选）
    *   预加载 HTTP 引擎
*   WebViewFactory.prepareWebViewInZygote()
    *   WebView 初始化
*   endPreload()
    *   ZygoteHooks.onEndPreload()：标记预加载结束
*   warmUpJcaProviders()
    *   AndroidKeyStoreProvider.install()：安装密钥库提供者
    *   预热 JCA 提供者

### ZygoteServer

> 运行在 Zygote 进程中监听 socket，fork 应用进程, 管理 USAP 池等

```
ZygoteServer(boolean isPrimaryZygote) {
    mUsapPoolEventFD = Zygote.getUsapPoolEventFD();

    if (isPrimaryZygote) {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME);
        mUsapPoolSocket =
                Zygote.createManagedSocketFromInitSocket(
                        Zygote.USAP_POOL_PRIMARY_SOCKET_NAME);
    } else {
        mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.SECONDARY_SOCKET_NAME);
        mUsapPoolSocket =
                Zygote.createManagedSocketFromInitSocket(
                        Zygote.USAP_POOL_SECONDARY_SOCKET_NAME);
    }

    mUsapPoolSupported = true;
    fetchUsapPoolPolicyProps();
}


```

*   创建并管理 Zygote socket（zygote 或 zygote_secondary）
*   创建并管理 USAP 池 socket（usap_pool_primary 或 usap_pool_secondary）
*   获取 USAP 池事件文件描述符

### SystemServer

> zygote 进程在启动时会创建 SystemServer 进程，SystemServer 进程则依赖于 zygote 提供的功能来管理和启动其他服务。
> 
> SystemServer 进程负责启动和管理各种系统服务，如 ActivityManager、WindowManager、PackageManager、PowerManager 等。这些服务是 Android 系统正常运行所必需的。

#### 调用链

```
SystemServer.run()
├─> InitBeforeStartServices - 启动服务前初始化
│   ├─> SystemProperties.set(SYSPROP_START_COUNT, ...) - 设置启动计数
│   ├─> SystemTimeZone.initializeTimeZoneSettingsIfRequired() - 初始化时区
│   ├─> 处理locale设置
│   ├─> Binder.setWarnOnBlocking(true) - 设置Binder警告
│   ├─> PackageItemInfo.forceSafeLabels() - 强制安全标签
│   ├─> SQLiteGlobal.sDefaultSyncMode = SYNC_MODE_FULL - SQLite同步模式
│   ├─> VMRuntime.getRuntime().clearGrowthLimit() - 清除内存增长限制
│   ├─> Build.ensureFingerprintProperty() - 确保指纹属性
│   ├─> Environment.setUserRequired(true) - 设置用户必需
│   ├─> BaseBundle.setShouldDefuse(true) - 设置Bundle解包
│   ├─> Parcel.setStackTraceParceling(true) - 设置Parcel堆栈跟踪
│   ├─> BinderInternal.disableBackgroundScheduling(true) - 禁用后台调度
│   ├─> BinderInternal.setMaxThreads(sMaxBinderThreads) - 设置Binder线程数
│   ├─> Looper.prepareMainLooper() - 准备主Looper
│   ├─> System.loadLibrary("android_servers") - 加载native库
│   ├─> initZygoteChildHeapProfiling() - 初始化堆分析
│   ├─> spawnFdLeakCheckThread() - 生成FD泄漏检查线程 (DEBUG)
│   ├─> performPendingShutdown() - 执行待处理的关机
│   ├─> createSystemContext() - 创建系统上下文
│   │   └─> ActivityThread.systemMain()
│   │       └─> ActivityThread.getSystemContext()
│   ├─> ActivityThread.initializeMainlineModules() - 初始化主线模块
│   ├─> mSystemServiceManager = new SystemServiceManager() - 创建系统服务管理器
│   └─> SystemServerInitThreadPool.start() - 启动初始化线程池
│
├─> StartServices - 启动服务
│   ├─> startBootstrapServices() - 启动引导服务
│   │   ├─> Installer installer = startInstaller() - 启动安装器
│   │   ├─> ActivityManagerService.Lifecycle ams = startBootstrapServices()
│   │   │   └─> mActivityManagerService = new ActivityManagerService()
│   │   ├─> startPowerManager() - 启动电源管理
│   │   ├─> startDisplayManager() - 启动显示管理
│   │   ├─> startPackageManagerService() - 启动包管理服务
│   │   │   └─> PackageManagerService.main()
│   │   ├─> startUserManager() - 启动用户管理
│   │   ├─> AttributeCache.init() - 初始化属性缓存
│   │   ├─> SystemConfig.getInstance() - 获取系统配置
│   │   ├─> startOverlayManager() - 启动Overlay管理
│   │   ├─> startSensorService() - 启动传感器服务
│   │   └─> 等等...
│   │
│   ├─> startCoreServices() - 启动核心服务
│   │   ├─> startBatteryService() - 启动电池服务
│   │   ├─> startUsageStatsService() - 启动使用统计服务
│   │   ├─> startWebViewUpdateService() - 启动WebView更新服务
│   │   └─> 等等...
│   │
│   ├─> startOtherServices() - 启动其他服务
│   │   ├─> startWindowManagerService() - 启动窗口管理服务
│   │   ├─> startInputManagerService() - 启动输入管理服务
│   │   ├─> startNotificationManagerService() - 启动通知管理服务
│   │   ├─> startLocationManagerService() - 启动位置管理服务
│   │   ├─> startCountryDetectorService() - 启动国家检测服务
│   │   ├─> startNetworkTimeUpdateService() - 启动网络时间更新服务
│   │   ├─> startConnectivityService() - 启动连接服务
│   │   ├─> startNetworkManagementService() - 启动网络管理服务
│   │   ├─> startNetworkStatsService() - 启动网络统计服务
│   │   ├─> startNetworkPolicyService() - 启动网络策略服务
│   │   ├─> startWifiService() - 启动WiFi服务
│   │   ├─> startBluetoothService() - 启动蓝牙服务
│   │   ├─> startAudioService() - 启动音频服务
│   │   ├─> startCameraService() - 启动相机服务
│   │   ├─> startMediaRouterService() - 启动媒体路由服务
│   │   ├─> startTelephonyRegistry() - 启动电话注册表
│   │   ├─> startTelecomService() - 启动电信服务
│   │   ├─> startWallpaperManagerService() - 启动壁纸管理服务
│   │   ├─> startSystemUI() - 启动系统UI
│   │   └─> 等等...
│   │
│   └─> startApexServices() - 启动Apex服务
│
└─> Looper.loop() - 进入主循环


```

### Select 循环

#### runSelectLoop(入口)

```
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
        ArrayList<ZygoteConnection> peers = new ArrayList<>();

        socketFDs.add(mZygoteSocket.getFileDescriptor());
        peers.add(null);

        mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

        while (true) {
            fetchUsapPoolPolicyPropsWithMinInterval();
            mUsapPoolRefillAction = UsapPoolRefillAction.NONE;

            int[] usapPipeFDs = null;
            StructPollfd[] pollFDs;

            // Allocate enough space for the poll structs, taking into account
            // the state of the USAP pool for this Zygote (could be a
            // regular Zygote, a WebView Zygote, or an AppZygote).
            if (mUsapPoolEnabled) {
                usapPipeFDs = Zygote.getUsapPipeFDs();
                pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
            } else {
                pollFDs = new StructPollfd[socketFDs.size()];
            }

            /*
             * For reasons of correctness the USAP pool pipe and event FDs
             * must be processed before the session and server sockets.  This
             * is to ensure that the USAP pool accounting information is
             * accurate when handling other requests like API deny list
             * exemptions.
             */

            int pollIndex = 0;
            for (FileDescriptor socketFD : socketFDs) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = socketFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;
            }

            final int usapPoolEventFDIndex = pollIndex;

            if (mUsapPoolEnabled) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = mUsapPoolEventFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;

                // The usapPipeFDs array will always be filled in if the USAP Pool is enabled.
                assert usapPipeFDs != null;
                for (int usapPipeFD : usapPipeFDs) {
                    FileDescriptor managedFd = new FileDescriptor();
                    managedFd.setInt$(usapPipeFD);

                    pollFDs[pollIndex] = new StructPollfd();
                    pollFDs[pollIndex].fd = managedFd;
                    pollFDs[pollIndex].events = (short) POLLIN;
                    ++pollIndex;
                }
            }

            int pollTimeoutMs;

            if (mUsapPoolRefillTriggerTimestamp == INVALID_TIMESTAMP) {
                pollTimeoutMs = -1;
            } else {
                long elapsedTimeMs = System.currentTimeMillis() - mUsapPoolRefillTriggerTimestamp;

                if (elapsedTimeMs >= mUsapPoolRefillDelayMs) {
                    // The refill delay has elapsed during the period between poll invocations.
                    // We will now check for any currently ready file descriptors before refilling
                    // the USAP pool.
                    pollTimeoutMs = 0;
                    mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                    mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

                } else if (elapsedTimeMs <= 0) {
                    // This can occur if the clock used by currentTimeMillis is reset, which is
                    // possible because it is not guaranteed to be monotonic.  Because we can't tell
                    // how far back the clock was set the best way to recover is to simply re-start
                    // the respawn delay countdown.
                    pollTimeoutMs = mUsapPoolRefillDelayMs;

                } else {
                    pollTimeoutMs = (int) (mUsapPoolRefillDelayMs - elapsedTimeMs);
                }
            }

            int pollReturnValue;
            try {
                pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }

            if (pollReturnValue == 0) {
                // The poll returned zero results either when the timeout value has been exceeded
                // or when a non-blocking poll is issued and no FDs are ready.  In either case it
                // is time to refill the pool.  This will result in a duplicate assignment when
                // the non-blocking poll returns zero results, but it avoids an additional
                // conditional in the else branch.
                mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

            } else {
                boolean usapPoolFDRead = false;

                while (--pollIndex >= 0) {
                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                        continue;
                    }

                    if (pollIndex == 0) {
                        // Zygote server socket
                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                        peers.add(newPeer);
                        socketFDs.add(newPeer.getFileDescriptor());
                    } else if (pollIndex < usapPoolEventFDIndex) {
                        // Session socket accepted from the Zygote server socket

                        try {
                            ZygoteConnection connection = peers.get(pollIndex);
                            boolean multipleForksOK = !isUsapPoolEnabled()
                                    && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
                            final Runnable command =
                                    connection.processCommand(this, multipleForksOK);

                            // TODO (chriswailes): Is this extra check necessary?
                            if (mIsForkChild) {
                                // We're in the child. We should always have a command to run at
                                // this stage if processCommand hasn't called "exec".
                                if (command == null) {
                                    throw new IllegalStateException("command == null");
                                }

                                return command;
                            } else {
                                // We're in the server - we should never have any commands to run.
                                if (command != null) {
                                    throw new IllegalStateException("command != null");
                                }

                                // We don't know whether the remote side of the socket was closed or
                                // not until we attempt to read from it from processCommand. This
                                // shows up as a regular POLLIN event in our regular processing
                                // loop.
                                if (connection.isClosedByPeer()) {
                                    connection.closeSocket();
                                    peers.remove(pollIndex);
                                    socketFDs.remove(pollIndex);
                                }
                            }
                        } catch (Exception e) {
                            if (!mIsForkChild) {
                                // We're in the server so any exception here is one that has taken
                                // place pre-fork while processing commands or reading / writing
                                // from the control socket. Make a loud noise about any such
                                // exceptions so that we know exactly what failed and why.

                                Slog.e(TAG, "Exception executing zygote command: ", e);

                                // Make sure the socket is closed so that the other end knows
                                // immediately that something has gone wrong and doesn't time out
                                // waiting for a response.
                                ZygoteConnection conn = peers.remove(pollIndex);
                                conn.closeSocket();

                                socketFDs.remove(pollIndex);
                            } else {
                                // We're in the child so any exception caught here has happened post
                                // fork and before we execute ActivityThread.main (or any other
                                // main() method). Log the details of the exception and bring down
                                // the process.
                                Log.e(TAG, "Caught post-fork exception in child process.", e);
                                throw e;
                            }
                        } finally {
                            // Reset the child flag, in the event that the child process is a child-
                            // zygote. The flag will not be consulted this loop pass after the
                            // Runnable is returned.
                            mIsForkChild = false;
                        }

                    } else {
                        // Either the USAP pool event FD or a USAP reporting pipe.

                        // If this is the event FD the payload will be the number of USAPs removed.
                        // If this is a reporting pipe FD the payload will be the PID of the USAP
                        // that was just specialized.  The `continue` statements below ensure that
                        // the messagePayload will always be valid if we complete the try block
                        // without an exception.
                        long messagePayload;

                        try {
                            byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
                            int readBytes =
                                    Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);

                            if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                                DataInputStream inputStream =
                                        new DataInputStream(new ByteArrayInputStream(buffer));

                                messagePayload = inputStream.readLong();
                            } else {
                                Log.e(TAG, "Incomplete read from USAP management FD of size "
                                        + readBytes);
                                continue;
                            }
                        } catch (Exception ex) {
                            if (pollIndex == usapPoolEventFDIndex) {
                                Log.e(TAG, "Failed to read from USAP pool event FD: "
                                        + ex.getMessage());
                            } else {
                                Log.e(TAG, "Failed to read from USAP reporting pipe: "
                                        + ex.getMessage());
                            }

                            continue;
                        }

                        if (pollIndex > usapPoolEventFDIndex) {
                            Zygote.removeUsapTableEntry((int) messagePayload);
                        }

                        usapPoolFDRead = true;
                    }
                }

                if (usapPoolFDRead) {
                    int usapPoolCount = Zygote.getUsapPoolCount();

                    if (usapPoolCount < mUsapPoolSizeMin) {
                        // Immediate refill
                        mUsapPoolRefillAction = UsapPoolRefillAction.IMMEDIATE;
                    } else if (mUsapPoolSizeMax - usapPoolCount >= mUsapPoolRefillThreshold) {
                        // Delayed refill
                        mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                    }
                }
            }

            if (mUsapPoolRefillAction != UsapPoolRefillAction.NONE) {
                int[] sessionSocketRawFDs =
                        socketFDs.subList(1, socketFDs.size())
                                .stream()
                                .mapToInt(FileDescriptor::getInt$)
                                .toArray();

                final boolean isPriorityRefill =
                        mUsapPoolRefillAction == UsapPoolRefillAction.IMMEDIATE;

                final Runnable command =
                        fillUsapPool(sessionSocketRawFDs, isPriorityRefill);

                if (command != null) {
                    return command;
                } else if (isPriorityRefill) {
                    // Schedule a delayed refill to finish refilling the pool.
                    mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                }
            }
        }
    }


```

1.  初始化
    1.  创建 socketFDs 和 peers 列表
    2.  添加主 socket 到列表
2.  循环处理
    1.  获取 USAP 池策略属性
    2.  构建 poll 结构数组（包含 socket、USAP event FD、USAP pipe FDs）
    3.  调用 Os.poll() 等待事件
3.  处理事件
    1.  pollIndex == 0：接受新连接（acceptCommandPeer()）
    2.  pollIndex <usapPoolEventFDIndex：处理会话 socket（processCommand()）
    3.  pollIndex >= usapPoolEventFDIndex：处理 USAP 池事件
4.  Fork 应用进程
    1.  processCommand() 解析命令并调用 forkAndSpecialize()
    2.  子进程返回 Runnable，父进程继续循环
5.  USAP 池管理
    1.  监听 USAP 事件和报告管道
    2.  根据池大小决定是否补充

  

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础知识](forum-171-1-180.htm) [#内核](forum-171-1-185.htm)