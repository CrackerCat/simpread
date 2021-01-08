> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1346211-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/000/59/08/90_avatar_middle.jpg)蟑螂二号

一、安卓内核日志系统介绍
------------

Android 系统中日志信息分为内核空间日志信息和用户空间日志信息，查看用户空间的 log 直接用 logcat 就可以了。比如如下命令:

```
adb logcat 

```

或者使用如下命令:

```
adb shell logcat

```

如果需要查看内核控件日志信息，有以下几种方式查看。

1.  读取 / proc/kmsg, 命令如下
    
    ```
    adb shell cat /proc/kmsg
    
    ```
    
    读取 / proc/kmsg 属于消费型读取，读取之后再次读取不会显示已经读取过的日志信息。
    
2.  读取 / dev/kmsg, 命令如下
    
    ```
    adb shell cat /dev/kmsg
    
    ```
    
    读取 / dev/kmsg 会显示缓存区里面的所有日志信息。新写入的日志信息会不断累加到日志缓冲器中。
    
3.  使用 dmesg 命令读取
    
    ```
    adb shell dmesg
    
    ```
    
    dmesg 命令读取一次只显示一部分日志, 非阻塞执行, 从他的源码中分析他是读取 / dev / 调用 klogctl 方式读取。demsg 源码实现方式如下。
    

实现代码路径位于源码目录:

> external/toybox/toys/lsb/dmesg.c

```
void dmesg_main(void)
{
   //...省略
  if (!(toys.optflags&FLAG_S)) {
        //...省略
    fd = open("/dev/kmsg", O_RDONLY|(O_NONBLOCK*!(toys.optflags&FLAG_w)));
    if (fd == -1) goto klogctl_mode;
    lseek(fd, 0, SEEK_DATA);
    for (;;) {
      if (-1==(len = read(fd, msg, sizeof(msg))) && errno==EPIPE) continue;
      // read() from kmsg always fails on a pre-3.5 kernel.
      if (len==-1 && errno==EINVAL) goto klogctl_mode;
      if (len<1) break;
      msg[len] = 0;
      format_message(msg, 1);
    }
    close(fd);
  } else {
    char *data, *to, *from, *end;
    int size;

klogctl_mode:
    // Figure out how much data we need, and fetch it.
    if (!(size = TT.size)) size = xklogctl(10, 0, 0);
    data = from = xmalloc(size+1);
    data[size = xklogctl(3+(toys.optflags&FLAG_c), data, size)] = 0;

    // Send each line to format_message.
    to = data + size;
    while (from < to) {
      if (!(end = memchr(from, '\n', to-from))) break;
      *end = 0;
      format_message(from, 0);
      from = end + 1;
    }

    if (CFG_TOYBOX_FREE) free(data);
  }
no_output:
  // Set the log level?
  if (toys.optflags & FLAG_n) xklogctl(8, 0, TT.level);
  // Clear the buffer?
  if (toys.optflags & (FLAG_C|FLAG_c)) xklogctl(5, 0, 0);
}

```

二、logcat 输出内核日志方案设计
-------------------

在本文中将采用 init.rc 中注册后台服务 kernellogd。kernellogd 不断读取 / proc/kmsg 信息，然后写到用户层的日志系统的方式达到目的。  
流程图如下:  
![](https://imgkr2.cn-bj.ufileos.com/6cc46dcf-80d4-4c2f-aced-f8cd5287a0b5.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=I2rbSmyiSmy%252Byl%252BGePsBUhaPnSE%253D&Expires=1610110289)

三、开发 kernellogd 程序
------------------

kernellogd 有两种方式集成到系统，一种是直接将源码放到系统某一个文件夹，创建 Android.mk 文件，加入到系统编译链中; 另种方式使用 android studio 工具开发，使用 cmake 或者 ndk 将源码编译成二进制可执行文件。目前我采用了使用 android studio 开发的方式，编译为 arm64 平台的可执行程序 kernellogd。android studio 开发流程此处不讨论，直接贴几句核心代码如下:

```
  //内核日志tag标识，可以用这个标识进行日志过滤
  #define LOG_TAG  "kERNEL_LOG"
  #define LOGkD(...)  __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
    FILE* fp=fopen("/proc/kmsg","r");
    if(fp==NULL)
    {
        LOGkE("fopen /proc/kmsg error");
        return -1;
    }
    while(true)
    {
        char buf[512]={0};
        char* ptrstr=fgets(buf,510,fp);
        if(ptrstr==NULL)
        {
            sleep(1);
        }else{
            LOGkD("%s",buf);
        }
    }
    fclose(fp);

```

四、配置 kernellogd 为系统服务
---------------------

具体步骤如下:

1.  **内置 kernellogd 可执行二进制文件到系统**
    
    如何内置可以参考之前的文章:
    
    [玩转 Android10 源码开发定制 (九) 内置 frida-gadget so 文件和 frida-server 可执行文件到系统](https://mp.weixin.qq.com/s/3RXB5PtHvos77oFVZHanfA)
    
2.  **配置 kernellogd 到 init.rc**
    
    安卓 10 系统源码中，init.rc 文件路径为:  
    >system/core/rootdir/init.rc
    
    在 init.rc 文件末尾添加 kernellogd 后台服务的配置，内容如下:
    
    ```
    # ///ADD START
    # ///ADD END
    service kernellogd  /system/bin/kernellogd
      class main
      user root
      seclabel u:r:init:s0
    
    ```
    
3.  修改 init.te 文件, 配置 init 进程可以执行 kernellogd 命令的 seandroid 权限
    
    涉及修改的 init.te 文件路径分别如下:  
    >system/sepolicy/public/init.te
    
    > system/sepolicy/prebuilts/api/29.0/public/init.te
    
    以上两个文件内容是一样的，需要保证同时被修改。修改如下:
    
    ```
    #...省略很多
    # 将下面的neverallow屏蔽掉
    # ///ADD START
    #neverallow init { file_type fs_type }:file execute_no_trans;
    # ///ADD END
    #...省略了很多
    # ///ADD START
    # ///ADD END
    allow init  system_file:file {execute_no_trans};
    
    ```
    
    **allow init system_file:file {execute_no_trans}** 语句含义: 允许所有属于 init 域的进程对所有属于 system_file 类型的文件具有执行权限。
    

五、测试验证
------

以上配置完成之后编译刷机。  
在终端执行 adb logcat 之后可以看到内核日志和用户空间日志都输出来了。测试数据样本如下:

```
01-07 21:26:41.617  1393  2142 I ActivityManager: Killing 5097:com.android.keychain/1000 (adj 999): empty for 6733s
01-07 21:26:41.633   912   912 I Zygote  : Process 5097 exited due to signal 9 (Killed)
01-07 21:26:41.662   462   462 I hwservicemanager: getTransport: Cannot find entry android.hardware.graphics.mapper@3.0::IMapper/default in either framework or device manifest.
01-07 21:27:43.847   997  7946 E ResolverController: No valid NAT64 prefix (100, <unspecified>/0)
01-07 21:27:48.017   894   985 I ThermalEngine: Monitor : quiet_therm = 25, msm_therm = 25, ufs_therm = 25, battery_therm = 231,current_now = 209000
01-07 21:28:00.031  1932  1932 D KeyguardClockSwitch: Updating clock: 9:28
01-07 21:28:00.792   891   891 D kERNEL_LOG: <12>[ 6859.400113] healthd: battery l=100 v=4313 t=23.1 h=2 st=5 c=208 fc=2966000 chg=u
01-07 21:28:06.799   891   891 D kERNEL_LOG: <12>[ 6865.406981] healthd: battery l=100 v=4313 t=23.2 h=2 st=5 c=202 fc=2966000 chg=u
01-07 21:28:08.525   891   891 D kERNEL_LOG: <4>[ 6867.133081] helloworld: module license 'unspecified' taints kernel.
01-07 21:28:08.525   891   891 D kERNEL_LOG: <4>[ 6867.133092] Disabling lock debugging due to kernel taint
01-07 21:28:08.526   891   891 D kERNEL_LOG: <1>[ 6867.133659] Hello World!

```

玩转 Android10 源码更多文章:

[玩转 Android10 源码开发定制 (一) 源码下载编译](https://mp.weixin.qq.com/s/mpiimuKmm_QsDMo27Yu6CQ)

[玩转 Android10 源码开发定制 (二) 刷机操作](https://mp.weixin.qq.com/s/_qSR87FOGsz1IiaLZnDqDg)

[玩转 Android10 源码开发定制 (二) 刷机操作之 fastboot 刷机演示](https://mp.weixin.qq.com/s/lecEZZpGvi98d8qAfxyp0w)

[玩转 Android10 源码开发定制 (二) 刷机操作之 Recovery 刷机演示](https://mp.weixin.qq.com/s/q2Q8QAG6efElI6W9yryHyQ)

[玩转 Android10 源码开发定制 (三) 源码中编译手机刷机包](https://mp.weixin.qq.com/s/3HDvF4VhgqE0ba2sJKbIQw)

[玩转 Android10 源码开发定制 (四) 源码开发环境搭建](https://mp.weixin.qq.com/s/9r3rm-s1ZaWPZ3K5Kq11cg)

[玩转 Android10 源码开发定制 (五) 源码编译开发中常用命令](https://mp.weixin.qq.com/s/SQgJ_HWdpxg7f3-DZz-FzA)

[玩转 Android10 源码开发定制 (六) 修改内核源码绕过反调试检测](https://mp.weixin.qq.com/s/jnU4HTVIjn6-DL1O0w_5gQ)

[玩转 Android10 源码开发定制 (七) 修改 ptrace 绕过反调试](https://mp.weixin.qq.com/s/IFrq5L2DuM8ewKnDTolxjQ)

[玩转 Android10 源码开发定制 (八) 内置 Apk 到系统](https://mp.weixin.qq.com/s/sdhBMx4yzVIx5RtDXPPfXw)

[玩转 Android10 源码开发定制 (九) 内置 frida-gadget so 文件和 frida-server 可执行文件到系统](https://mp.weixin.qq.com/s/3RXB5PtHvos77oFVZHanfA)

[玩转 Android10 源码开发定制 (十) 增加获取当前运行最顶层的 Activity 命令](https://mp.weixin.qq.com/s/LSQi1Q7_CvWZrzxDdEBESA)

[玩转 Android10 源码开发定制 (11) 内核篇之安卓内核模块开发编译](https://mp.weixin.qq.com/s/RXSa7hpG6DNiyHueBz68JA)

![](https://avatar.52pojie.cn/data/avatar/001/55/90/19_avatar_middle.jpg)I_amagoodboy 谢谢大佬分享 ![](https://avatar.52pojie.cn/data/avatar/001/32/78/56_avatar_middle.jpg) zkt190 谢谢分享 坐等大佬分析安卓 11 ![](https://avatar.52pojie.cn/data/avatar/001/30/37/75_avatar_middle.jpg)chanmo1314 大佬牛逼 ![](https://avatar.52pojie.cn/data/avatar/001/24/44/20_avatar_middle.jpg) feng645806 不错！！![](https://avatar.52pojie.cn/data/avatar/001/30/96/13_avatar_middle.jpg)寒冰流火 谢谢楼主发此帖  对 Android 的日志梳理得很清楚  相当于提供了一条大路