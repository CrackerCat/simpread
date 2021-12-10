> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [evilpan.com](https://evilpan.com/2020/11/08/android-init/)

> 从硬件上电启动到终端软件可用是一个漫长而复杂的过程，我们在开心享受着丰富的应用程序时候，可能并没想到这其中所包含的工程师心血。

从硬件上电启动到终端软件可用是一个漫长而复杂的过程，我们在开心享受着丰富的应用程序时候，可能并没想到这其中所包含的工程师心血。

之前在 [从 STM32L4 看 ARM 裸板的启动过程](https://evilpan.com/2020/04/11/stm32-bare-metal/) 一文中介绍了对于 ARM 芯片而言，如何从硬件中断到初始化的软件执行。对于真实设备而言，还包含更多的板载外设，通常需要一个独立的 bootloader 去对其进行初始化，比如 U-Boot、aboot。随后加载并进入 Linux 内核，这一部分不是本文的重点，现在只需要知道，内核初始化完毕后所执行的第一个进程是 init，本文就以 Android O (8.1.0_r81) 为例，从 init 开始梳理其启动流程。

init 是用户态的第一个进程，由 Linux 内核启动，进程号为`1`。其代码在 [system/core/init/init.cpp](http://androidxref.com/8.1.0_r33/xref/system/core/init/init.cpp)，这里也不做源码复读机了，而是根据自己的阅读理解介绍其工作过程，以及背后一些可能会被忽略的点。

按照执行流程，init 实际上被执行了 4 次，分别是:

```
int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);

    if (is_first_stage) {
      	// ...
		setenv("INIT_SECOND_STAGE", "true", 1);
        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args);
    }
  	// second stage
  	// ...
}


```

在 Linux 2.6 之后，[udev](https://en.wikipedia.org/wiki/Udev) 取代了 DevFS，即 `userspace /dev`。其作用主要是管理 /dev 目录下的设备结点，以及当硬件设备插入或者拔出系统时候负责处理用户空间对应的事件，比如创建对应的结点、设置文件权限和 SELinux 标签等。在 Android 中对应的用户空间程序就是 ueventd。

从全局看，当 ueventd 启动的时候，会为所有当前注册的设备重新生成 uevent，主要是通过遍历 `/sys` 目录中的 `uevent` 文件并向其中写入 `add`，从而导致内核生成并重新发送 uevent 事件信息给所有注册的设备。为什么要这么做呢？因为当这些设备注册的时候 ueventd 还没运行，所以没有办法接收到完整的设备注册信息，也就没办法正确地对其进行处理，所以要重新激活一遍，这个过程就叫做`冷启动`(cold boot)。init 进程的运行需要等待冷启动完成。

watchdogd 是独立的看门狗进程，负责定期喂狗。核心代码:

```
int watchdogd_main() {
  int fd = open("/dev/watchdog", O_RDWR|O_CLOEXEC);
  ioctl(fd, WDIOC_SETTIMEOUT, &timeout);
  while (true) {
    write(fd, "", 1);
    sleep(interval);
  }
}


```

看门狗是一个独立的硬件，在嵌入式设备中通常用来检测设备存活状态，在一段时间超时后，看门狗硬件会认为系统已经跑飞，从而重启设备。

首先进行最基本的文件系统初始化:

```
mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
mkdir("/dev/pts", 0755);
mkdir("/dev/socket", 0755);
mount("devpts", "/dev/pts", "devpts", 0, NULL);
#define MAKE_STR(x) __STRING(x)
mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
// Don't expose the raw commandline to unprivileged processes.
chmod("/proc/cmdline", 0440);
gid_t groups[] = { AID_READPROC };
setgroups(arraysize(groups), groups);
mount("sysfs", "/sys", "sysfs", 0, NULL);
mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));
mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));


```

然后，first stage 才正式开始。

第一步，挂载文件系统。需要挂载内容主要从 `device_tree_fstab_` 而来，即 Linux 设备树中指定的 fstab，第一阶段中对于每个 fstab 中的记录 (fstab_rec):

*   不允许 verifyatboot，即要求 `fs_mgr_is_verifyatboot(fstab_rec) == 0`；
*   检查是否启用 dm-verity，`fs_mgr_is_verified(fstab_rec)`；
*   获取 dm-verity 需要的 metadata: `fstab_rec->verity_loc`；
*   如果使用了 A/B 更新，则修改对应的分区名称；

挂载过程主要是通过 fs_mgr 的接口:

1.  fs_mgr_setup_verity(fstab_rec, false);
2.  fs_mgr_do_mount_one(fstab_rec);

第二步，AVB(Android Verified Boot) 校验，根据内核命令行 `/proc/cmdline` 中的 **androidboot.vbmeta.{hash_alg, size, digest}** 参数来确定校验的算法。AVB 校验的结果有几种:

1.  metadata 处理出错，比如 I/O 错误、校验和或者大小不匹配；
2.  **kAvbHandleHashtreeDisabled**，AVB 为关闭状态，用于支持 `adb disable-verity`；
3.  **kAvbHandleVerificationDisabled**，跳过校验，用于支持 `avbctl disable-verification`；
4.  **kAvbHandleVerificationError**，AVB 校验错误，如果设备不是解锁状态会直接报错；
5.  **kAvbHandleSuccess**，AVB 校验成功。

第三步，初始化 SELinux。注意此时我们还是在 kernel domain 中，加载完 sepolicy 之后，需要重新设置 `/init` 文件的 SELinux 属性 (`selinux_android_restorecon("/init", 0)`)，并以新的 init domain 重新执行 init 程序，进入第二阶段。

第二阶段的 init 进程，就是我们在 Android 用户态中见到的真正程序。这一阶段主要包括对设备属性 (property) 相关操作以及 initrc 的处理，执行对应的开机启动进程；其他还有比如 SELinux 的二次初始化，以及对子进程退出信号和 property_change 等事件的 epoll 循环监听和处理等。

首先说说 property，这是 Android 中一个简单的 key/value 内存数据库，可通过 getprop/setprop 来获取和设置。在第二阶段初期时 init 进程就通过 `property_init` 初始化一段共享内存用于保存这些 property 数据。随后，**依次**从下面的参数或者文件中读取并设置对应的属性:

1.  内核设备树中，获取 compatible 和 name 属性，保存到 **ro.boot.copmatible/name** 中；
2.  内核命令行中，获取 **androidboot.** 开头的值，保存到 **ro.boot.** 属性中；
3.  设置默认的 ro 属性，包括 serialno、mode、baseband、bootloader、hardware 和 revision；
4.  设置 ro.boottime.init、ro.boottime.init.selinux、ro.boot.avb_version 等无关紧要的属性；
5.  加载默认属性，顺序依次是:

```
void property_load_boot_defaults() {
    if (!load_properties_from_file("/system/etc/prop.default", NULL)) {
        // Try recovery path
        if (!load_properties_from_file("/prop.default", NULL)) {
            // Try legacy path
            load_properties_from_file("/default.prop", NULL);
        }
    }
    load_properties_from_file("/odm/default.prop", NULL);
    load_properties_from_file("/vendor/default.prop", NULL);

    update_sys_usb_config();
}


```

6.  启动属性服务 (property_service) 和 action manager，然后设置对应的默认 action，其中属性相关的有 **load_system_props** 和 **load_persist_props**。前者的顺序如下:

```
void load_system_props() {
    load_properties_from_file("/system/build.prop", NULL);
    load_properties_from_file("/odm/build.prop", NULL);
    load_properties_from_file("/vendor/build.prop", NULL);
    load_properties_from_file("/factory/factory.prop", "ro.*");
    load_recovery_id_prop();
}


```

load_persist_props 则主要是从 `/data/property/*` 目录中加载属性。

除了属性服务，init 中另外一个重要的功能就是对 initrc 的处理，毕竟作为用户态的第一个进程，其肩负了启动其他进程和服务的使命。在 Linux 中使用 `init.rc` 文件来描述各个启动项的启动属性和顺序，关于该文件格式的详细介绍可以参考 [initrc/README.md](https://android.googlesource.com/platform/system/core/+/master/init/README.md)。这里还是用代码来描述初始化的过程比较直接:

```
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }


```

在 `/init.rc` 文件中，通常使用 `import` 去包含了多个其他的 rc 文件，比如:

```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc


```

其中末尾的 zygote 就是下一节要介绍的受精卵进程。

另外值得一提的是，在 init 初始化属性内存映射之后，就重新设置了 SELinux 权限，因为很多文件是由 init 进程创建的，需要恢复其本身的 SELinux context:

```
static void selinux_restore_context() {
    LOG(INFO) << "Running restorecon...";
    selinux_android_restorecon("/dev", 0);
    selinux_android_restorecon("/dev/kmsg", 0);
    selinux_android_restorecon("/dev/socket", 0);
    selinux_android_restorecon("/dev/random", 0);
    selinux_android_restorecon("/dev/urandom", 0);
    selinux_android_restorecon("/dev/__properties__", 0);

    selinux_android_restorecon("/plat_file_contexts", 0);
    selinux_android_restorecon("/nonplat_file_contexts", 0);
    selinux_android_restorecon("/plat_property_contexts", 0);
    selinux_android_restorecon("/nonplat_property_contexts", 0);
    selinux_android_restorecon("/plat_seapp_contexts", 0);
    selinux_android_restorecon("/nonplat_seapp_contexts", 0);
    selinux_android_restorecon("/plat_service_contexts", 0);
    selinux_android_restorecon("/nonplat_service_contexts", 0);
    selinux_android_restorecon("/plat_hwservice_contexts", 0);
    selinux_android_restorecon("/nonplat_hwservice_contexts", 0);
    selinux_android_restorecon("/sepolicy", 0);
    selinux_android_restorecon("/vndservice_contexts", 0);

    selinux_android_restorecon("/dev/block", SELINUX_ANDROID_RESTORECON_RECURSE);
    selinux_android_restorecon("/dev/device-mapper", 0);

    selinux_android_restorecon("/sbin/mke2fs_static", 0);
    selinux_android_restorecon("/sbin/e2fsdroid_static", 0);
}


```

恢复之后，根据对应的 sepolicy，即便是 init 进程本身，也不再对这些文件有完全的访问权限。

init 进程本身是不退出的，在初始化完成后，会进入循环侦听子进程的退出事件，并根据需要，重新拉起这些进程。此外也在 property_service 中侦听了属性的变化，并对每个属性的变化执行 rc 文件中对应的操作。通过阅读代码发现一个有趣的地方是当属性为 sys.powerctl 的时候，在 init 的主循环执行对应 powerctl 的操作，所以我们可以用下面的命令去关机:

```
setprop sys.powerctl shutdown


```

~又学到了一个没用的技巧。~

前面说了 init 进程会依次执行 rc 文件，来启动对应的进程，其中 zogote 进程就在 `/init.zygote64_32.rc` 中:

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks

service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    priority -20
    user root
    group root readproc
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks


```

**zygote** 是所有 Android 进程的父进程，因为 Java 虚拟机就是在该进程中初始化的。其可执行文件名称为 `app_process`，入口代码在 [frameworks/base/cmds/app_process/app_main.cpp](http://androidxref.com/8.1.0_r33/xref/frameworks/base/cmds/app_process/app_main.cpp) 。zygote 的代码分为两大部分，一部分在 native 层，另一部分则使用 Java 来编写。

native 层的入口就是上述 app_main 的入口，根据命令行参数执行不同的操作，在末尾进入了 `AppRuntime` 的 main 函数:

```
// ...
AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
// ...
if (zygote) {
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
} else if (className) {
    runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
} else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
}


```

AppRuntime 继承于 `AndroidRuntime`，后者的定义在 `frameworks/base/core/jni/AndroidRuntime.cpp`。其 start 函数的作用有两个:

1.  启动 Java 虚拟机；
2.  调用第一个参数对应 Java 类的 main 函数；

核心代码如下:

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
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
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    jclass startClass = env->FindClass(slashClassName);
		jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
}


```

中间省略了一些错误处理代码，更加突出主要逻辑。其中:

*   startVm 负责创建 Java 虚拟机，可根据参数和属性值调整虚拟机的特性；
*   startReg 负责动态绑定一系列 JNI native 函数 (使用`JNIEnv->RegisterNatives` 来注册)；
*   最后调用对应 Java 类的主函数 `static void main(String[] args)` 。

根据代码逻辑，调入到 Java 类中只有不再返回，并且该线程称为 zygote 的主线程。否则该函数返回后 VM 也随之销毁。

进入 Java 层之后，执行的是 `com.android.internal.os.ZygoteInit` 类的 main 函数。代码在 `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`，Java 代码写起来虽然繁琐，但是不得不说阅读起来还是很直观的。还是按照执行顺序，省略一些细枝末节，zygote 初始化可以归纳如下:

1.  启动 DDMS，即 **Dalvik Debug Monitor Server**，在新版本中被 [**Android Profiler**](https://developer.android.com/studio/profile/android-profiler) 代替；
2.  预加载一些常用类和资源。由于 app 进程由 zygote fork 而来，因此子进程也继承了预加载的资源，从而加速应用的创建和初始化过程，子进程使用 `copy-on-write` 方式复用资源；
3.  unmount root 存储空间，设置 seccomp policy；
4.  启动 system_server，这是咱们下一节的主角；
5.  zygoteServer.runSelectLoop: 监听创建的 socket 并循环等待连接，接收到子进程启动命令后则运行`ZygoteConnection.processOneCommand` 来 fork 并启动子进程。

启动 app 进程的代码如下:

```
Runnable processOneCommand(ZygoteServer zygoteServer) {
    // ...
    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
            parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
            parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.instructionSet,
            parsedArgs.appDataDir);

    try {
        if (pid == 0) {
            // in child
            zygoteServer.setForkChild();

            zygoteServer.closeServerSocket();
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;

            return handleChildProc(parsedArgs, descriptors, childPipeFd);
        } else {
            // In the parent. A pid < 0 indicates a failure and will be handled in
            // handleParentProc.
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            handleParentProc(pid, descriptors, serverPipeFd);
            return null;
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}


```

当然在 processOneCommand 中还做了很多权限的校验，比如 UID 是否合法，以及根据 ro.debuggable 和启动参数决定是否开启 JDWP 调试。Zygote 进程创建 app 进程之后一步步进入到最终 Android 应用的 `Activity .onCreate` 函数中，最终执行应用程序的用户代码。

在 app 进程启动前我们可以做很多有趣的事情，比如大家熟知的 Xposed 框架，原理上就是修改了 zygote 的代码，在应用启动前判断目标的包名是否需要 hook，如果匹配则 hook 对应的函数 (比如将函数 flag 修改为 native，增加注入回调处理等)。

在上节中，我们说到 zygote 启动了 system_server，这也是一个 Java 进程，为什么需要单独启动呢，有什么特殊的地方？要知道在 Android 中一个进程可以对应多个服务 (service)，而各类操作通常都是客户端通过 Binder (AIDL) 与对应的服务端进行通信并执行，比如`SurfaceFlinger`、`LocationService`、`TelephoneService`等。其中，system_server 这一个进程中，就包含了 **100+** 系统服务，可谓是支撑了 Android 系统的半壁江山。

在 **zygote.rc** 中可以发现，只有 **app_process64** 进程指定了 `--start-system-server`，所以 system_server 只执行一个架构即可。

```
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
    long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_IPC_LOCK,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_PTRACE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG,
            OsConstants.CAP_WAKE_ALARM
            );
    /* Containers run without this capability, so avoid setting it in that case */
    if (!SystemProperties.getBoolean(PROPERTY_RUNNING_IN_CONTAINER, false)) {
        capabilities |= posixCapabilitiesAsBits(OsConstants.CAP_BLOCK_SUSPEND);
    }
    /* Hardcoded command line to start the system server */
    String args[] = {
        "--setuid=1000",
        "--setgid=1000",
        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
        "--capabilities=" + capabilities + "," + capabilities,
        "--nice-,
        "--runtime-args",
        "com.android.server.SystemServer",
    };
    ZygoteConnection.Arguments parsedArgs = null;

    int pid;

    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}


```

可以看到，system_server 的 uid/gid 都是 1000 (system)，并且有许多高权限的 capabilities。虽然看起来是一个 native 程序，但实际上是一个 Java 程序，fork 之后执行的 Java 类是 `com.android.server.SystemServer`，代码在 `frameworks/base/services/java/com/android/server/SystemServer.java`。

在代码中已经比较详细注释了 system_server 启动的过程，由于该进程是在 Android 开启期间持续运行的，因此需要配置好合适的虚拟机参数，之后主要的工作就是按顺序启动一些列的服务。

Native services 使用 `System.loadLibrary("android_servers")` 来进行 JNI 初始化。在当前版本中包括:

*   SensorService
*   HidlServices

后者启动了 HIDL server，负责启动一些需要在 system_server 进程中注册的服务，具体可以参考上篇文章: [Android HAL 与 HIDL 开发笔记](https://evilpan.com/2020/11/01/hidl-basics/)。

这类服务是启动系统所必须的核心服务，并且他们有许多共同的复杂依赖，这些服务包括:

<table><thead><tr><th>类名</th><th>备注</th></tr></thead><tbody><tr><td>com.android.server.pm.Installer</td><td>installd</td></tr><tr><td>com.android.server.os.DeviceIdentifiersPolicyService</td><td></td></tr><tr><td>com.android.server.am.ActivityManagerService.Lifecycle</td><td></td></tr><tr><td>com.android.server.power.PowerManagerService</td><td></td></tr><tr><td>RecoverySystemService</td><td></td></tr><tr><td>com.android.server.lights.LightsService</td><td></td></tr><tr><td>com.android.server.display.DisplayManagerService</td><td></td></tr><tr><td>com.android.server.pm.PackageManagerService</td><td>PMS</td></tr><tr><td>com.android.server.pm.UserManagerService.LifeCycle</td><td></td></tr><tr><td>com.android.server.om.OverlayManagerService</td><td></td></tr><tr><td>SensorService</td><td>native</td></tr></tbody></table>

同样是核心服务，但是没有在 Bootstrap 阶段启动，包括:

<table><thead><tr><th>类名</th><th>备注</th></tr></thead><tbody><tr><td>DropBoxManagerService</td><td></td></tr><tr><td>BatteryService</td><td></td></tr><tr><td>com.android.server.usage.UsageStatsService</td><td></td></tr><tr><td>com.android.server.webkit.WebViewUpdateService</td><td></td></tr></tbody></table>

这里则用来启动剩下的系统服务全家桶，列表如下:

<table><thead><tr><th>类名</th><th>备注</th></tr></thead><tbody><tr><td>com.android.server.security.KeyAttestationApplicationIdProviderService</td><td>sec_key_att_app_id_provider</td></tr><tr><td>com.android.server.security.KeyChainSystemService</td><td></td></tr><tr><td>com.android.server.os.SchedulingPolicyService</td><td>scheduling_policy</td></tr><tr><td>com.android.server.telecom.TelecomLoaderService</td><td></td></tr><tr><td>TelephonyRegistry</td><td>telephony.registry</td></tr><tr><td>com.android.server.accounts.AccountManagerService$Lifecycle</td><td></td></tr><tr><td>com.android.server.content.ContentService$Lifecycle</td><td></td></tr><tr><td>VibratorService</td><td>vibrator</td></tr><tr><td>ConsumerIrService</td><td></td></tr><tr><td>AlarmManagerService</td><td></td></tr><tr><td>Watchdog</td><td></td></tr><tr><td>com.android.server.input.InputManagerService</td><td></td></tr><tr><td>com.android.server.wm.WindowManagerService</td><td>WMS</td></tr><tr><td>com.android.server.vr.VrManagerService</td><td></td></tr><tr><td>BluetoothService</td><td></td></tr><tr><td>com.android.server.connectivity.IpConnectivityMetrics</td><td></td></tr><tr><td>PinnerService</td><td></td></tr><tr><td>InputMethodManagerService.Lifecycle</td><td></td></tr><tr><td>com.android.server.accessibility.AccessibilityManagerService</td><td></td></tr><tr><td>com.android.server.StorageManagerService$Lifecycle</td><td></td></tr><tr><td>com.android.server.usage.StorageStatsService$Lifecycle</td><td></td></tr><tr><td>UiModeManagerService</td><td></td></tr><tr><td>com.android.server.locksettings.LockSettingsService$Lifecycle</td><td></td></tr><tr><td>PersistentDataBlockService</td><td></td></tr><tr><td>com.android.server.oemlock.OemLockService</td><td></td></tr><tr><td>DeviceIdleController</td><td></td></tr><tr><td>com.android.server.statusbar.StatusBarManagerService</td><td></td></tr><tr><td>com.android.server.clipboard.ClipboardService</td><td></td></tr><tr><td>NetworkManagementService</td><td></td></tr><tr><td>TextServicesManagerService.Lifecycle</td><td></td></tr><tr><td>NetworkScoreService</td><td></td></tr><tr><td>com.android.server.net.NetworkStatsService</td><td></td></tr><tr><td>com.android.server.net.NetworkPolicyManagerService</td><td></td></tr><tr><td>com.android.server.wifi.WifiService</td><td></td></tr><tr><td>com.android.server.wifi.scanner.WifiScanningService</td><td></td></tr><tr><td>com.android.server.wifi.RttService</td><td></td></tr><tr><td>com.android.server.wifi.aware.WifiAwareService</td><td></td></tr><tr><td>com.android.server.wifi.p2p.WifiP2pService</td><td></td></tr><tr><td>com.android.server.lowpan.LowpanService</td><td></td></tr><tr><td>com.android.server.ethernet.EthernetService</td><td></td></tr><tr><td>ConnectivityService</td><td></td></tr><tr><td>NsdService</td><td></td></tr><tr><td>UpdateLockService</td><td></td></tr><tr><td>…</td><td></td></tr></tbody></table>

本来想对着代码列举完所有 service，发现实在太多，有需要的可以对照对应的 AOSP 代码使用下述方式搜索:

```
grep traceBeginAndSlog SystemServer.java | awk -F"\"" '{print $2}' | sort | uniq


```

上面类名中包含 `Lifecycle` 的部分，其实是一个内部静态类，继承自 SystemService，并且包含本身服务的一个实例。比如`ActivityManagerService.Lifecycle` :

```
    public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        @Override
        public void onCleanupUser(int userId) {
            mService.mBatteryStatsService.onCleanupUser(userId);
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }


```

所有服务启动完成后，核心系统进入就绪状态，就可以等待用户 app 的运行了。在服务准备完毕后，system_server 会调用对应服务的 `systemReady` 回调，并且在其中调用合适的 **SystemServiceManager.startBootPhase(SystemService.PHASE_XXX)** 来触发不同阶段应该执行的操作。

后面的流程就是 ActivityManagerService 在 systemReady 启动所有依赖的服务，并且调用它们的 systemReady 和 systemRunning 方法，随后调用 startHomeActivityLocked 启动桌面 Activity，即 Launcher app，展现我们的第一个图形界面窗口，让用户可以通过点击 app 图标的方式去打开不同的应用。

值得一提的是，启动 Launcher 和启动许多系统应用一样，都是通过 Android 的消息机制去启动的，这里是 `Intent.CATEGORY_HOME`。也就是说用户 app 可以在 manifest 中声明响应这个 Intent，从而构造自己的 Launcher 程序。

每次对 Android 系统进行深入了解，都总能发现从前忽略的地方，这一方面说明 Android Framework 本身功能复杂，另一方面也说明自己不够细致。当然，有新发现总是快乐的，尤其是这种快乐还能带来意想不到的作用时。比如，通过修改 zygote，我们可以实现一套自己的 hook 框架，避免常规 app 的安全检测；又比如，通过修改 Service，可以实现系统整体的监控，例如 MIUI 12 中**隐私照明弹**功能对 `AppOpsService` 的魔改和拓展。当然，这都是题外话了，以后有机会的话可以详细分享一番。

*   [Android 操作系统架构](http://gityuan.com/android/)
*   [Android HAL 与 HIDL 开发笔记](https://evilpan.com/2020/11/01/hidl-basics/)
*   [Android 进程间通信与逆向分析](https://evilpan.com/2020/07/11/android-ipc-tips/)
*   [从 STM32L4 看 ARM 裸板的启动过程](https://evilpan.com/2020/04/11/stm32-bare-metal/)