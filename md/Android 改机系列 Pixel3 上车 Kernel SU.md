> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1766826-1-1.html)

> [md]## 背景让 Pixel3 AOSP Android10 4.9 内核用上 `Kernel SU` 喜欢看视频的戳这里：[Android 改机系列 Pixel3 上车 Kernel SU_哔哩哔哩_bili......

![](https://avatar.52pojie.cn/data/avatar/000/73/84/40_avatar_middle.jpg)debug_cat _ 本帖最后由 debug_cat 于 2023-3-30 18:40 编辑_  

背景
--

![](https://attach.52pojie.cn/forum/202303/29/212743pluceqob8xzlzu88.jpg)

**kernel 封面. jpg** _(60 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzA5Nnw2ZDEzOGIyY3wxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:27 上传

让 Pixel3  AOSP Android10  4.9 内核用上`Kernel SU`

喜欢看视频的戳这里：  
[Android 改机系列 Pixel3 上车 Kernel SU_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ss4y1D7HC/?vd_source=53d2a1b45490a5af700cafc66ccffb90)

环境：

Ubuntu 18.04 vm

aosp10r2

内核下载和编译查看之前的文章

文章 [AOSP Android 10 内核编译刷入 Pixel3](http://www.debuglive.cn/article/1074036099963158528)

视频 [Android 改机系列 AOSP 10 Pixel3 内核编译_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1oT411X7gc/?vd_source=76d437fab6961fcbf9da5564bd8f8776)

移植参考官方，和 github 项目

[Commits · OnlyTomInSecond/android_kernel_xiaomi_sdm845 (github.com)](https://github.com/OnlyTomInSecond/android_kernel_xiaomi_sdm845/commits/lineageos-20-no-kprobe)

这个项目是 LineageOS/android_kernel_xiaomi_sdm845

### 编译的前提

已经有完整的 AOSP 编译环境，且成功编译并刷机的。

下载内核代码，我选择的是 qpr1 的

```
android-msm-crosshatch-4.9-android10-qpr1 --depth=1

```

具体怎么下载内核代码，看上面的文章，这里不再细说。

下载好的目录结构是这样的。  
![](https://attach.52pojie.cn/forum/202303/29/212805dh2ii0lli7ll20ej.jpg)

**QQ 截图 20230328211929.jpg** _(273.88 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzA5N3xjMDA0NTBkNnwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

要成功编译一次下载好的内核，在移植 kernel su。

到这里假设你已经成功编译，且刷入手机正常使用。

### 开始移植 kernel su

kernel su 的官方网站：[Android 上的内核级的 root 方案 | KernelSU](https://kernelsu.org/zh_CN/)

pixel3 内核支持最高版本是 4.9，kernel su 目前不支持这个版本，需要我们手动移植。

> KernelSU 可以被集成到非 GKI 内核中，现在它最低支持到内核 4.14 版本；理论上也可以支持更低的版本。

首先，把 KernelSU 添加到你的内核源码树，在内核的根目录执行以下命令：

```
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -

```

根据文档的描述，在`内核的根目录执行以下命令`那么哪里是内核的根目录呢？

![](https://attach.52pojie.cn/forum/202303/29/212809kr363r3f66crlmcl.jpg)

**QQ 截图 20230328212715.jpg** _(151.28 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzA5OHwwZWQwMjA4OHwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

进入 msm-google 之后（全球通上网）执行

```
aosp@ubuntu:~/aosp/kernel/private/msm-google$ curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -
++ pwd
+ GKI_ROOT=/home/aosp/aosp/kernel/private/msm-google
+ echo '[+] GKI_ROOT: /home/aosp/aosp/kernel/private/msm-google'
[+] GKI_ROOT: /home/aosp/aosp/kernel/private/msm-google
+ test -d /home/aosp/aosp/kernel/private/msm-google/common/drivers
+ test -d /home/aosp/aosp/kernel/private/msm-google/drivers
+ DRIVER_DIR=/home/aosp/aosp/kernel/private/msm-google/drivers
+ test -d /home/aosp/aosp/kernel/private/msm-google/KernelSU
+ git clone https://github.com/tiann/KernelSU
Cloning into 'KernelSU'...
remote: Enumerating objects: 11090, done.
remote: Counting objects: 100% (940/940), done.
remote: Compressing objects: 100% (373/373), done.
remote: Total 11090 (delta 565), reused 877 (delta 514), pack-reused 10150
Receiving objects: 100% (11090/11090), 8.80 MiB | 2.34 MiB/s, done.
Resolving deltas: 100% (5740/5740), done.
+ cd /home/aosp/aosp/kernel/private/msm-google/KernelSU
+ git stash
No local changes to save
+ git pull
Already up to date.
+ cd /home/aosp/aosp/kernel/private/msm-google
+ echo '[+] GKI_ROOT: /home/aosp/aosp/kernel/private/msm-google'
[+] GKI_ROOT: /home/aosp/aosp/kernel/private/msm-google
+ echo '[+] Copy kernel su driver to /home/aosp/aosp/kernel/private/msm-google/drivers'
[+] Copy kernel su driver to /home/aosp/aosp/kernel/private/msm-google/drivers
+ test -e /home/aosp/aosp/kernel/private/msm-google/drivers/kernelsu
+ ln -sf /home/aosp/aosp/kernel/private/msm-google/KernelSU/kernel /home/aosp/aosp/kernel/private/msm-google/drivers/kernelsu
+ echo '[+] Add kernel su driver to Makefile'
[+] Add kernel su driver to Makefile
+ DRIVER_MAKEFILE=/home/aosp/aosp/kernel/private/msm-google/drivers/Makefile
+ grep -q kernelsu /home/aosp/aosp/kernel/private/msm-google/drivers/Makefile
+ printf '\nobj-y += kernelsu/\n'
+ echo '[+] Done.'
[+] Done.
aosp@ubuntu:~/aosp/kernel/private/msm-google$ 

```

这样就下载好代码。

* * *

#### 手动修改内核源码

全部修改的位置

```
kernel\private\msm-google\arch\arm64\configs\b1c1_defconfig
kernel\private\msm-google\fs\exec.c
kernel\private\msm-google\fs\read_write.c
kernel\private\msm-google\fs\open.c
kernel\private\msm-google\drivers\input\input.c

```

打开`kprobe`

pixel3 的内核配置文件在

\192.168.216.133\aosp\aosp\kernel\private\msm-google\arch\arm64\configs\\192.168.216.133\aosp\aosp\kernel\private\msm-google\arch\arm64\configs\b1c1_defconfig

这个路径

![](https://attach.52pojie.cn/forum/202303/29/212812i4p8awro668o99wz.jpg)

**QQ 截图 20230328213543.jpg** _(79.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzA5OXw3NGJlYTkwMnwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

用文本编辑器打开 b1c1_defconfig

增加以下配置，如果配置已经存在，不用重复添加

```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y

```

\192.168.216.133\aosp\aosp\kernel\private\msm-google\fs\exec.c

1673 行

```
//addcode start
extern int ksu_handle_execveat(int *fd, struct filename **filename_ptr, void *argv,
            void *envp, int *flags);
//addcode end
/*
 * sys_execve() executes a new program.
 */
static int do_execveat_common(int fd, struct filename *filename,
                  struct user_arg_ptr argv,
                  struct user_arg_ptr envp,
                  int flags)
{
    char *pathbuf = NULL;
    struct linux_binprm *bprm;
    struct file *file;
    struct files_struct *displaced;
    int retval;
    //addcode
    ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags);
    //addcode end
    if (IS_ERR(filename))
        return PTR_ERR(filename);

```

\192.168.216.133\aosp\aosp\kernel\private\msm-google\fs\read_write.c

```
//addcode start
extern int ksu_handle_vfs_read(struct file **file_ptr, char __user **buf_ptr,
            size_t *count_ptr, loff_t **pos);
//addcode end
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    ssize_t ret;
    //addcode start
    ksu_handle_vfs_read(&file, &buf, &count, &pos);
    //addcode end
    if (!(file->f_mode & FMODE_READ))
        return -EBADF;

```

\192.168.216.133\aosp\aosp\kernel\private\msm-google\fs\open.c

357 行

```
//addcode start
extern int ksu_handle_faccessat(int *dfd, const char __user **filename_user, int *mode,
             int *flags);
//addcode end
/*
 * access() needs to use the real uid/gid, not the effective uid/gid.
 * We do this by temporarily clearing all FS-related capabilities and
 * switching the fsuid/fsgid around to the real ones.
 */
SYSCALL_DEFINE3(faccessat, int, dfd, const char __user *, filename, int, mode)
{
    const struct cred *old_cred;
    struct cred *override_cred;
    struct path path;
    struct inode *inode;
    struct vfsmount *mnt;
    int res;
    unsigned int lookup_flags = LOOKUP_FOLLOW;
    //addcode
    ksu_handle_faccessat(&dfd, &filename, &mode, NULL);
    //addcode end
    if (mode & ~S_IRWXO)    /* where's F_OK, X_OK, W_OK, R_OK? */
        return -EINVAL;

```

\192.168.216.133\aosp\aosp\kernel\private\msm-google\drivers\input\input.c

368 行

```
//addcode
extern int ksu_handle_input_handle_event(unsigned int *type, unsigned int *code, int *value);
//addcode end

static void input_handle_event(struct input_dev *dev,
                   unsigned int type, unsigned int code, int value)
{
    int disposition = input_get_disposition(dev, type, code, &value);
    //addcode
    ksu_handle_input_handle_event(&type, &code, &value);
    //addcode end
    if (disposition != INPUT_IGNORE_EVENT && type != EV_SYN)
        add_input_randomness(type, code, value);

```

开始编译，载入编译源码环境

```
# aosp源码目录下
source build/envsetup.sh
lunch 3
# 切换到内核目录
./build/build.sh

```

利用 aosp 环境重新生成 boot，使用自编译的内核

```
# aosp源码目录下
export TARGET_PREBUILT_KERNEL=/home/aosp/aosp/kernel/out/android-msm-pixel-4.9/dist/Image.lz4
make bootimage

```

编译成功后，把产物中的 ko 文件推送到手机中，再刷入新的 boot。

```
adb root 
## 关闭验证
adb disable-verity 
## 重启手机
adb reboot 
## 重启机器成功之后：
adb root 
adb remount -R 

```

![](https://attach.52pojie.cn/forum/202303/29/212820dxdd212daxxdu2df.png)

**Snipaste_2023-03-29_20-26-36.png** _(454.41 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzEwMHwzOWI1OGNhMXwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

在内核目录下执行

![](https://attach.52pojie.cn/forum/202303/29/212828jwwmdszh5msss8w4.png)

**Snipaste_2023-03-29_20-27-08.png** _(727.43 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzEwMXw5N2M5ODIzMHwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

```
adb push out/android-msm-pixel-4.9/dist/*.ko /vendor/lib/modules

```

重启手机到 bootloader

```
adb reboot bootloader

```

进入 aosp 的产品目录下

我的位置是 aosp/out/target/product/blueline

![](https://attach.52pojie.cn/forum/202303/29/212837sz1drcff1pdu5fff.png)

**Snipaste_2023-03-29_20-27-23.png** _(605.88 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzEwMnxmM2VhYTk5ZnwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

```
## 烧入新内核
fastboot flash boot boot.img

```

查看内核：

```
λ adb shell
dipper:/ # cat /proc/version
Linux version 4.9.185_KernelSU-g726f44b-dirty (build-user@build-host) (Android (5484270 based on r353983c) clang version 9.0.3 (https://android.googlesource.com/toolchain/clang 745b335211bb9eadfa6aa6301f84715cee4b37c5) (https://android.googlesource.com/toolchain/llvm 60cf23e54e46c807513f7a36d0a7b777920b5881) (based on LLVM 9.0.3svn)) #1 SMP PREEMPT Tue Mar 28 14:27:58 UTC 2023
dipper:/ #

```

新编译的内核信息, 在内核 out/android-msm-pixel-4.9/dist 下执行

```
grep -a 'Linux version' Image.lz4

```

验证是否可以用，安装管理器。

![](https://attach.52pojie.cn/forum/202303/29/212843w5wsss7pfvvqv95l.png)

**2023-03-29_20_36_51.png** _(101.51 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzEwM3w2MjAzNTIxMHwxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

![](https://attach.52pojie.cn/forum/202303/29/212847yb41oeekgwsnasgq.png)

**2023-03-29_20_37_08.png** _(151.42 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYwMzEwNHxmNjJmOTc4N3wxNjgwMjExOTMzfDB8MTc2NjgyNg%3D%3D&nothumb=yes)

2023-3-29 21:28 上传

最后
--

再次感谢 [Commits · OnlyTomInSecond/android_kernel_xiaomi_sdm845 (github.com)](https://github.com/OnlyTomInSecond/android_kernel_xiaomi_sdm845/commits/lineageos-20-no-kprobe)

希望大家编译永不报错，写代码冇 bug。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)sylzh 不 root 的机型能改么？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)panda56166 谢谢大神分享![](https://static.52pojie.cn/static/image/smiley/default/17.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) larrylee2005 哇，膜拜大神![](https://static.52pojie.cn/static/image/smiley/default/42.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 5ayuan 哇，膜拜大神 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) weiyanli 学习中，膜拜大神 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) SuperMingZhi 真是好东西啊 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) wslqarenas0718 模块可以正常运行吗 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zjh889 好东西，向楼主学习！