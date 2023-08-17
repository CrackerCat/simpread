> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278445.htm)

> [原创] 初探安卓隐藏 Bootloader 锁

[原创] 初探安卓隐藏 Bootloader 锁

44 分钟前 44

### [原创] 初探安卓隐藏 Bootloader 锁

 [![](http://passport.kanxue.com/upload/avatar/784/844784.png?1655549110)](user-home-844784.htm) [cslime](user-home-844784.htm) ![](https://bbs.kanxue.com/view/img/rank/8.png) 2  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 44 分钟前  44

初探安卓隐藏 Bootloader 锁
===================

做完硬件断点记录器后就开始研究如何隐藏 Android 的 Bootloader 锁了, 当然最近没啥兴趣继续研究了, 把目前为止的思路发出来

首先想到的一个方向是要先 spoof 安卓 Setting 里的 oemlock status

收集所需信息
------

在 aoxpxref 网站上搜索

`bootloader unlocked`

看到一个可疑的文件

`/frameworks/base/core/java/android/service/oemlock/OemLockManager.java`

xref 找到

`/frameworks/base/services/core/java/com/android/server/oemlock/OemLockService.java`

看到函数

```
private static final String FLASH_LOCK_PROP = "ro.boot.flash.locked";
private static final String FLASH_LOCK_UNLOCKED = "0"
 
public boolean isDeviceOemUnlocked() {
    enforceOemUnlockReadPermission();
    String locked = SystemProperties.get(FLASH_LOCK_PROP);
    switch (locked) {
        case FLASH_LOCK_UNLOCKED:
            return true;
        default:
            return false;
    }
}

```

可以看到 Setting 检测的是系统属性 `ro.boot.flash.locked` 来判断 Bootloader 是否解锁

AOSP 搜索`ro.boot.flash.locked`, 发现`/system/core/init/init.cpp` 有初始化该属性值的代码

```
static void export_oem_lock_status() {
    if (!android::base::GetBoolProperty("ro.oem_unlock_supported", false)) {
        return;
    }
    SetProperty(
            "ro.boot.flash.locked",
            android::base::GetProperty("ro.boot.verifiedbootstate", "") == "orange" ? "0" : "1");
}

```

`ro.boot.flash.locked` 的值是受到 `ro.boot.verifiedbootstate` 的值影响

google 搜了一下后面的这个属性, 发现这个属性可能的值有 red,orange,yellow,green

看到一个隐藏 bl 锁的 github 项目

[https://github.com/topjohnwu/Magisk/blob/3c04dab47274e9bbbfb3ddd1fcf71c929c8ed0c0/native/jni/magiskhide/hide_policy.cpp#L12](https://github.com/topjohnwu/Magisk/blob/3c04dab47274e9bbbfb3ddd1fcf71c929c8ed0c0/native/jni/magiskhide/hide_policy.cpp#L12)

看到替换表

```
static const char *prop_key[] =
        { "ro.boot.vbmeta.device_state", "ro.boot.verifiedbootstate", "ro.boot.flash.locked",
          "ro.boot.veritymode", "ro.boot.warranty_bit", "ro.warranty_bit",
          "ro.debuggable", "ro.secure", "ro.build.type", "ro.build.tags",
          "ro.vendor.boot.warranty_bit", "ro.vendor.warranty_bit",
          "vendor.boot.vbmeta.device_state", nullptr };
 
static const char *prop_val[] =
        { "locked", "green", "1",
          "enforcing", "0", "0",
          "0", "1", "user", "release-keys",
          "0", "0",
          "locked", nullptr };

```

可以确定 green 是 bl 未解锁, 还有一个 `ro.boot.vbmeta.device_state` 要改成 locked 状态, 实机看这个属性解锁 bl 状态下是 unlocked, 除了前三个之外其他属性解 bl 之后没变

还有 googleplay service 的验证

[https://developer.android.com/training/safetynet/attestation](https://developer.android.com/training/safetynet/attestation)

[https://developer.android.com/google/play/integrity/overview](https://developer.android.com/google/play/integrity/overview)

对应的解决方案是

[https://github.com/kdrag0n/safetynet-fix](https://github.com/kdrag0n/safetynet-fix)

这个问题不大, 国内又不用 googleplay service

总结需要修改的信息
---------

按照上述总结一下, 隐藏 bl 锁需要修改以下属性

<table><thead><tr><th>Property</th><th>Value</th></tr></thead><tbody><tr><td>ro.boot.flash.locked</td><td>1</td></tr><tr><td>ro.boot.verifiedbootstate</td><td>green</td></tr><tr><td>ro.boot.vbmeta.device_state</td><td>locked</td></tr></tbody></table>

寻找修改 Property 的方法
-----------------

Java 层最终会调用`native_get`方法获取 Property 的值

顺着`native_get` 找到函数`__system_property_find` 这个是用户层标准的获取 property 值的函数, 来自 libc.so, 往下翻就是 android property 系统的实现了, 可以看这篇文章

[https://bbs.kanxue.com/thread-274100.htm](https://bbs.kanxue.com/thread-274100.htm)

进用户层初始化 property service 的地方

`/system/core/init/property_service.cpp`

找到函数 `PropertyInit` 从名字猜测这是 init 进程初始化系统基本 property 值的地方,`ProcessBootconfig`和`ProcessKernelCmdline`最后是通过读取`/proc/bootconfig`和`/proc/cmdline`调用`_system_property_add`来设置`ro.boot.verifiedbootstate`和`ro.boot.vbmeta.device_state`的值

cat 一下这两个设备文件, 确实有相关信息

```
adb shell su -c cat /proc/bootconfig
androidboot.hardware = "qcom"
androidboot.memcg = "1"
androidboot.usbcontroller = "a600000.dwc3"
androidboot.bootdevice = "1d84000.ufshc"
androidboot.boot_devices = "soc/1d84000.ufshc"
androidboot.prjname = "22803"
androidboot.startupmode = "pwrkey"
androidboot.mode = "normal"
androidboot.product.hardware.sku = "0"
androidboot.hw_region_id = "1"
androidboot.serialno = "xxxxxxx"
androidboot.baseband = "msm"
androidboot.dtbo_idx = "12"
androidboot.dtb_idx = "1"
androidboot.force_normal_boot = "0"
androidboot.fstab_suffix = "default"
androidboot.verifiedbootstate = "orange"
androidboot.keymaster = "1"
androidboot.vbmeta.device = "PARTUUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
androidboot.vbmeta.avb_version = "1.0"
androidboot.vbmeta.device_state = "unlocked"
androidboot.vbmeta.hash_alg = "sha256"
androidboot.vbmeta.size = "23232"
androidboot.vbmeta.digest = "xxxxx"
androidboot.vbmeta.invalidate_on_error = "yes"
androidboot.veritymode = "enforcing"
androidboot.slot_suffix = "_a"

```

我想到了三个思路

1.  在内核层修改 / proc/bootconfig 和 / proc/cmdline 的内容
2.  对_system_property_add 下硬件断点, 回调里修改 value 的参数
3.  修改 / dev/**properties**/u:object_r:bootloader_prop:s0 里的信息

第一个思路
-----

kernel 版本 5.10.168 路径`/common/fs/proc/bootconfig.c` 中的`proc_boot_config_init` 函数初始化了`/proc/bootconfig`的输出, 顺着全局变量 `xbc_data` 找到系统获取 bootconfig 的地方`/common/init/main.c -> setup_bootconfig` , 可以看到 bootconfig 是从 initrd 里拿的

![](https://bbs.kanxue.com/upload/attach/202308/844784_ACGYEBG974DBN95.png)

将返回的 data copy 进全局变量, 然后由驱动打印出来看到数据

```
www1@www1s-Mac-mini ~ % adb shell su -c dmesg | grep "\[+"
[    0.278576] [+] androidboot.hardware=qcom\x0aandroidboot.memcg=1\x0aandroidboot.usbcontroller=a600000.dwc3\x0a\x0aandroidboot.bootdevice=1d84000.ufshc\x0aandroidboot.boot_devices=soc/1d84000.ufshc\x0aandroidboot.prjname=22803\x0aandroidboot.startupmode=hard_reset\x0aandroidboot.mode=normal\x0aandroidboot.product.hardware.sku=0\x0aandroidboot.hw_region_id=1\x0aandroidboot.serialno=xxxxxxxx\x0aandroidboot.baseband=msm\x0aandroidboot.dtbo_idx=12\x0aandroidboot.dtb_idx=1\x0aandroidboot.force_normal_boot=0\x0aandroidboot.fstab_suffix=default\x0a\x0aandroidboot.verifiedbootstate=orange\x0aandroidboot.keymaster=1\x0aandroidboot.vbmeta.device=PARTUUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx\x0aandroidboot.vbmeta.avb_version=1.0\x0aandroidboot.vbmeta.device_state=unlocked\x0aandroidboot.vbmeta.hash_alg=sha256\x0aandroidboot.vbmeta.size=23232\x0aandroidboot.vbmeta.digest=xxxxxxxxxxxxx\x0aandroidboot.vbmeta.invalidate_on_error=yes\x0aandroidboot.veritymode=enforcing\x0aandroidboot.slot_suffix=_a\x0a\x0d

```

尝试文本替换

```
void regen_bootcfg(char* buffer)
 {
   char * pos;
   pos = strstr(buffer, "verifiedbootstate=orange");
   if(pos)
   {
         memcpy(pos,"verifiedbootstate=green\n",24);
   }
   pos = strstr(buffer, "vbmeta.device_state=unlocked");
   if(pos)
   {
     memcpy(pos,"vbmeta.device_state=locked\n\n",28);
   }
     pos = strstr(buffer, "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx");
     if(pos)
     {
         memcpy(pos,"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx1",36);
     }
 }

```

实验发现替换 uuid 和 vbmeta.device_state 系统正常启动, 一旦修改 verifiedbootstate 系统就无法启动了, 第一个思路就结束在这里了

第二个思路
-----

先说结果: **成功了!!!!!!**

安卓的 linker 使用的是 mmap 映射的 elf 文件, hook do_mmap 的返回处, 判断 pgoff 和 filepath 就能拦截到 init 进程加载 libc.so, 在这个时机里对_system_property_add 函数下 execute hook,

```
void rwhandler_hidebl_mmap_callback(unsigned long addr,struct file *file)
{
  char fullpath[1024];
    char *path = d_path(&file->f_path, fullpath, 1024);
    if(
        path &&
        !strcmp(path,"/system/lib64/bootstrap/libc.so") &&
        current->pid == 1)
    {
        rwhandler_start_debug();
        rwhandler_set_hbp(1,addr+0x90430,4,HW_BREAKPOINT_X);
        printk("task = %s file = %s addr = %llX\r\n",current->comm,path,addr);
    }
}

```

hook 回调里对参数进行简单处理

```
static const char *hide_props[]={"ro.boot.verifiedbootstate","ro.boot.vbmeta.device_state",NULL};
static char *hide_value[]={"green","locked",NULL};
 
static bool hidebl_hbpcallback(struct perf_event *bp,
          struct perf_sample_data *data,
          struct pt_regs *regs)
{
    if(current->pid == 1)
  {
    char key[100];
    unsigned long keyaddr = regs->regs[0];
    unsigned long varaddr = regs->regs[2];
    memset(key, 0, 100);
    printk("[+] init hit pc = %llX x0 = %llX x1 = %llX\r\n",
      regs->pc,regs->regs[0],regs->regs[1]);
     
    keyaddr &= 0xFFFFFFFFFFFFFF;
    varaddr &= 0xFFFFFFFFFFFFFF;
    if(rwhandler_read_memory(
      1,
      (unsigned long)keyaddr,
      key, 99)>0)
    {
      char value[100];
      memset(value, 0, 100);
      printk("[+] value = %llX\r\n",*(uint64_t*)key);
      printk("[+] init key = %s\r\n",key);
       
      if(rwhandler_read_memory(
        1,
        (unsigned long)varaddr,
        value, 99)>0)
      {
                int idx = 0;
           printk("[+] init value = %s\r\n",value);
                while(hide_props[idx])
                {
                    if(!strcmp(key,hide_props[idx]))
                    {
                         
                        if(rwhandler_do_ptrace_write(1, varaddr, hide_value[idx], strlen(hide_value[idx])+1))
                        {
                            regs->regs[3]=strlen(hide_value[idx]);
                        }
                        break;
                    }
                    idx++;
                }
      }
    }
    return true;
  }
    return false;
};

```

修改成功

![](https://bbs.kanxue.com/upload/attach/202308/844784_BKVA9CRF3N9FJNX.png)

成功 Spoof Setting

![](https://bbs.kanxue.com/upload/attach/202308/844784_7EZE3XXJWHBBQMU.png)

结尾
--

第三个方法暂时没试过, 感兴趣的可以试试看

有空就把第二个思路用 kprobe 来实现 hook, 包装成驱动连带硬件断点记录器分别开源到 github 上

刚入坑安卓, 如有不对的地方或者有其他方法检测 bl 锁请大佬们在评论区指出!!!

  

[议题征集启动！看雪 · 第七届安全开发者峰会 10 月 23 日上海](https://bbs.kanxue.com/thread-276136.htm)

最后于 40 分钟前 被 cslime 编辑 ，原因： 修复图片 [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm)