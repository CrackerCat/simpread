> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/UAPwYIMRFyDyPkvMfjNXCg)

**一、seandroid 简介**  

       SEAndroid 是 Google 在 Android4.4 上正式推出的一套以 SELinux 为核心的系统安全机制。在 Android 源码中，系统默认的 seandroid 配置存放如下路径:

```
/home/qiang/lineageOs/system/sepolicy

```

  目录中存放了比如 adbd、system_server、系统 app、第三方 app 等 te 配置文件。

    由于 Android 系统引进了 seandroid 策略。强化了 app 对资源的访问限制。安全性大大提高。比如举一个获取 wifi mac 的例子为例说明:

     在 android app 中很多 app 通过读取 /sys/class/net/wlan0/address 来获取手机的 wifi mac 地址。通过 adb 命令查看该文件的权限如下:

```
C:\Users\Qiang>adb shell ls -la  /sys/class/net/wlan0/address
-r--r--r-- 1 root root 4096 2021-01-12 14:57 /sys/class/net/wlan0/address

```

     以上说明手机的 app 都可以读取访问该文件。但是在 Android10 中，系统配置了普通 App 不能读取 /sys/class/net/wlan0/address 的 seandroid 策略权限。导致 Android 10 中读取失败，提示权限拒绝。由于 seandroid 强化了系统安全性，要想一个 App 访问访问系统的某一个目录或者路径，需要专门去配置 te 文件策略。对于不熟悉 seandroid 配置的开发者配置起来有点难度。那有没有办法不配置 seandroid 策略文件，设置文件或者目录可读权限就能访问的方法。答案就是全局关闭 **seandroid**。

**二、安卓中关闭 seandroid 的方式讨论**

  **  1. 使用 setenforce 命令临时关闭**  

       命令如下

```
adb shell setenforce 0

```

    setenforce 命令只能暂时关闭 seandroid，如果手机重启了会被恢复为正常状态。  

   setenforce 在安卓源码中的路径如下:

```
external/toybox/toys/android/setenforce.c

```

setenforce 实现代码如下：

```
#define FOR_setenforce
#include "toys.h"
void setenforce_main(void)
{
  char *new = *toys.optargs;
  int state, ret;
  if (!is_selinux_enabled()) error_exit("SELinux is disabled");
  else if (!strcmp(new, "1") || !strcasecmp(new, "enforcing")) state = 1;
  else if (!strcmp(new, "0") || !strcasecmp(new, "permissive")) state = 0;
  else error_exit("Invalid state: %s", new);
  ret = security_setenforce(state);
  if (ret == -1) perror_msg("Couldn't set enforcing status to '%s'", new);
}

```

  从以上代码可知，setenforce 最终调用的是函数 security_setenforce 完成 selinux 的控制。

  **2. 在 kernel 关闭 selinux**

    在内核中配置 SECURITY_SELINUX 设置为 false，重新编译 kernel 刷机。可以永久关闭 seandroid。

    以下是测试的内核编译中. config 文件中关闭 selinux 之后的配置信息:  

```
CONFIG_SECURITY_SELINUX=n

```

**3. 在 init 进程启动的时候关闭 selinux** 

 安卓系统启动过程中，init 进程会进行 selinux 的初始化。通过读取 / proc/cmdline 文件，判断 androidboot.selinux 的值是否需要开启 selinux。因此，我们可以 init 进程初始化 selinux 的时候强制执行关闭操作。

    以下将讨论第三种方案来实现全局关闭 selinux。

 **三、init 进程中全局关闭 selinux**

   ** 1.init 进程中 selinux 的初始化流程分析**

       init 进程中 selinux 初始化相关的文件路径如下:

```
system/core/init/selinux.cpp
system/core/init/main.cpp

```

     大概的初始化流程如下:

      **a.** main.cpp 中的 main 函数调用 selinux.cpp 中的 SetupSelinux:

```
int main(int argc, char** argv) {
        ...省略
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }
        ...省略
}

```

    **b.** selinux.cpp 中 SetupSelinux 函数实现如下:  

```
int SetupSelinux(char** argv) {
    ...省略
    SelinuxInitialize();
    ...省略
    return 1;
}

```

  **c.** SetupSelinux 调用了 SelinuxInitialize 方法。SelinuxInitialize 方法代码如下:

```
//SelinuxInitialize 中可以看到调用了IsEnforcing方法判断
void SelinuxInitialize() {
    ...省略
    bool kernel_enforcing = (security_getenforce() == 1);
    //判断是否强制模式
    bool is_enforcing = IsEnforcing();
    if (kernel_enforcing != is_enforcing) {
        //调用security_setenforce函数，和setenforce原理一样
        if (security_setenforce(is_enforcing)) {
            PLOG(FATAL) << "security_setenforce(%s) failed" << (is_enforcing ? "true" : "false");
        }
    }
    ...省略
}

```

    **d.**IsEnforcing 方法实现如下:  

```
//判断是否需要强制模式
bool IsEnforcing() {
    if (ALLOW_PERMISSIVE_SELINUX) {
        return StatusFromCmdline() == SELINUX_ENFORCING;
    }
    return true;
}

```

    从 IsEnforcing 中可以知道，如果一直返回 false，那么将会关闭 selinux。  

**2. 全局强制关闭 selinux 修改**  

   从以上 init 进程初始化 selinux 的流程可以提供两种修改方案来全局关闭。

*     第一种修改 IsEnforcing 函数永远返回 false。 修改如下:
    

```
  bool IsEnforcing() {
     ///ADD START
    if(1>0)
    {
       //一直返回false
       return false;
    }
    ///ADD END
    if (ALLOW_PERMISSIVE_SELINUX) {
        return StatusFromCmdline() == SELINUX_ENFORCING;
    }
    return true;
}

```

*     第二种修改 SelinuxInitialize 方法，在函数中主动调用 security_setenforce（false）。修改之后如下:
    

```
void SelinuxInitialize() {
    Timer t;
    LOG(INFO) << "Loading SELinux policy";
    if (!LoadPolicy()) {
        LOG(FATAL) << "Unable to load SELinux policy";
    }
    bool kernel_enforcing = (security_getenforce() == 1);
    bool is_enforcing = IsEnforcing();
    if (kernel_enforcing != is_enforcing) {
        if (security_setenforce(is_enforcing)) {
            PLOG(FATAL) << "security_setenforce(%s) failed" << (is_enforcing ? "true" : "false");
        }
    }
    //直接调用security_setenforce方法来关闭
    ///ADD START
    security_setenforce(false);
    ///ADD END
    if (auto result = WriteFile("/sys/fs/selinux/checkreqprot", "0"); !result) {
        LOG(FATAL) << "Unable to write to /sys/fs/selinux/checkreqprot: " << result.error();
    }
    // init's first stage can't set properties, so pass the time to the second stage.
    setenv("INIT_SELINUX_TOOK", std::to_string(t.duration().count()).c_str(), 1);
}

```

修改之后编译源码刷机，开机之后生效。  

玩转 Android10 源码开发定制更多文章:

[玩转 Android10 源码开发定制 (一) 源码下载编译](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483664&idx=1&sn=d526ceae91aafc3176a7d88da2cfc6a4&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (二) 刷机操作](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483690&idx=1&sn=f15656343046f6bc8af2304982288a23&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (二) 刷机操作之 fastboot 刷机演示](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483690&idx=2&sn=bd298d08da8978ba7d5bf3329b059cbe&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (二) 刷机操作之 Recovery 刷机演示](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483690&idx=3&sn=547e5269ede412973a03ba3090848bf9&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (三) 源码中编译手机刷机包](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483703&idx=1&sn=a5518fc9083b2ec92c995b79d691fa6a&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (四) 源码开发环境搭建](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483826&idx=1&sn=c3398f87832db6550fc8c1123cefff84&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (五) 源码编译开发中常用命令](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483833&idx=1&sn=b0d82bafa3c27b2825b4f8176dc94917&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (六) 修改内核源码绕过反调试检测](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483842&idx=1&sn=4e54d0ed08bf653fef3faa5830a615b1&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (七) 修改 ptrace 绕过反调试](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483847&idx=1&sn=786c050dbf588658423e6c026aed44dc&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (八) 内置 Apk 到系统](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483855&idx=1&sn=06138a9db04ba2a761f8dd9495ecd56a&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (九) 内置 frida-gadget so 文件和 frida-server 可执行文件到系统](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483871&idx=1&sn=ef4971b75f64b56891d372524b02f36d&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (十) 增加获取当前运行最顶层的 Activity 命令](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483918&idx=1&sn=8894d70d8b62b4ab2f7d9f8424e1e642&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (11) 内核篇之安卓内核模块开发编译](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483936&idx=1&sn=89cdee39c0bda4cca7ea4d289a03c62f&scene=21#wechat_redirect)

[玩转 Android10 源码开发定制 (12) 内核篇之 logcat 输出内核日志](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483951&idx=1&sn=ff938c7e5714e94473c440a60dd51329&chksm=ce07536af970da7c8a77e9bd220b3897faf920663f20db5822635f9c219b9d8e22508b55de10&scene=21#wechat_redirect)  

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)