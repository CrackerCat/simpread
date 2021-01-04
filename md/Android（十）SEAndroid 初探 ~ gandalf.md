> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.gandalf.site](http://www.gandalf.site/2019/02/androidseandroid.html)

SEAndroid（Security-Enhanced Android），是将原本运用在 Linux 操作系统上的 MAC 强制存取控管套件 SELinux，移植到 Android 平台上。可以用来强化 Android 操作系统对 App 的存取控管，建立类似沙箱的执行隔离效果，来确保每一个 App 之间的独立运作，也因此可以阻止恶意 App 对系统或其它应用程序的攻击。

SEAndroid 在架构和机制上与 SELinux 完全一样，中心理念为：即使恶意应用获取了 root 权限，依然可以阻止应用的越权读取行为。考虑到移动设备的特点，所以移植到 SEAndroid 的只是 SELinux 的一个子集。

[![](https://user-images.githubusercontent.com/11291711/51018489-bf56a780-15b2-11e9-8965-7ce499aeea5d.png)](https://user-images.githubusercontent.com/11291711/51018489-bf56a780-15b2-11e9-8965-7ce499aeea5d.png)

SEAndroid 工作流程如下，程序访问资源时，会先向 SEAndroid/SELinux 发送请求，SEAndroid 根据策略数据库进行策略分析，比对安全上下文，控制应用程序的资源存取。  
[![](https://user-images.githubusercontent.com/11291711/51018490-bfef3e00-15b2-11e9-8c57-33669cf55c91.png)](https://user-images.githubusercontent.com/11291711/51018490-bfef3e00-15b2-11e9-8c57-33669cf55c91.png)

SEAndroid 是一种基于安全策略的 MAC 安全机制。这种安全策略又是建立在对象的安全上下文的基础上的。这里所说的对象分为两种类型，一种称主体（Subject），一种称为客体（Object）。主体通常就是指进程，而客观就是指进程所要访问的资源，例如文件、系统属性等。

SEAndroid 的安全检查覆盖了所有重要的方面包括了域转换、类型转换、进程相关操作、内核相关操作、文件目录相关操作、文件系统相关操作、对设备相关操作、对 app 相关操作、对网络相关操作、对 IPC 相关操作。

在用户空间中，SEAndroid 包含有三个主要的模块，分别是安全上下文（Security Context）、安全策略（SEAndroid Policy）和安全服务（Security Server）。

0x21 SEAndroid 工作模式
-------------------

目前 SELinux 支持三种模式，分别如下：

*   enforcing：强制模式，代表 SELinux 运行中，且已经正确的开始限制 domain/type 了；
    
*   permissive：宽容模式：代表 SELinux 运行中，不过仅会有警告信息并不会实际限制 domain/type 的存取，这种模式可以运来作为 SELinux 的 debug 之用；
    
*   disabled：关闭，SELinux 并没有实际运行。
    

显示 SEAndroid 工作状态可以使用以下指令，输出结果为`Enforcing`，说明使用强制模式：

```
$ getenforce
Enforcing


```

设置 SEAndroid 工作模式可以使用以下指令，enforcing 为 1，permissive 为 0，但需要相应权限才能执行：

```
$ setenforce --help
usage: setenforce [enforcing|permissive|1|0]
Sets whether SELinux is enforcing (1) or permissive (0).
$ setenforce 1
setenforce: Could not set enforcing status to '1': Permission denied
$ setenforce 0
setenforce: Could not set enforcing status to '0': Permission denied


```

0x22 安全上下文（Security Context）
----------------------------

SEAndroid 的安全上下文与 SELinux 基本一致，实际上就是一个附加在对象上的标签（Tag）。这个标签实际上就是一个字符串，它由四部分内容组成，分别是 SELinux 用户、SELinux 角色、类型、安全级别，每一个部分都通过一个冒号来分隔，格式为`user:role:type:sensitivity`。

通过`ls -Z`指令可以查看文件的安全上下文信息：

```
1|sailfish:/ $ ls -Z

u:object_r:cgroup:s0           acct         u:object_r:tmpfs:s0            mnt

u:object_r:rootfs:s0           bin          u:object_r:vendor_file:s0      odm

u:object_r:rootfs:s0           bugreports   u:object_r:oemfs:s0            oem

u:object_r:rootfs:s0           charger      u:object_r:proc:s0             proc

u:object_r:configfs:s0         config       u:object_r:system_file:s0      product

u:object_r:rootfs:s0           d            u:object_r:rootfs:s0           res

u:object_r:system_data_file:s0 data         u:object_r:rootfs:s0           sbin

u:object_r:rootfs:s0           default.prop u:object_r:rootfs:s0           sdcard

u:object_r:device:s0           dev          u:object_r:storage_file:s0     storage

u:object_r:rootfs:s0           dsp          u:object_r:sysfs:s0            sys

u:object_r:rootfs:s0           etc          u:object_r:system_file:s0      system

u:object_r:firmware_file:s0    firmware     u:object_r:vendor_file:s0      vendor

u:object_r:rootfs:s0           lost+found


```

通过`ps -Z`指令可以查看进程的安全上下文信息：

```
sailfish:/ $ ps -Z

LABEL                          USER           PID  PPID     VSZ    RSS WCHAN            ADDR S NAME

u:r:shell:s0                   shell         8987  8920    9256   1988 SyS_rt_si+ 7c60833e58 S sh

u:r:shell:s0                   shell         9666  8987   11724   2200 0          7192d327b8 R ps


```

以`u:object_r:system_data_file:s0`为例，这些标记代表含义如下：

*   user：安全上下文的第一列为 SELinux 用户，在 SEAndroid 中的 user 只有一个就是 u。
    
*   role：第二列表示 SELinux 角色，在 SEAndroid 中的 role 有两个，分别为 r 和 object_r。
    
*   type：第三列为 type 类型，SEAndroid 中共定义了 139 种不同的 type。
    
*   security level：第四列为安全级别，由敏感性（Sensitivity）和类别（Category）两部分内容组成的，格式为`sensitivity[:category_set]`，`category_set`可选。
    

安全上下文中最重要的部分就是第三列的 type，type 是整个 SEAndroid 中最重要的一个参量，所有的 policy 都围绕这一参量展开，所以为系统中每个文件标记上合适的 type 就显得极为重要。而 SELinux 用户、SELinux 角色和安全级别都几乎可以忽略不计的。

在 SEAndroid 中，我们通常将用来标注文件的安全上下文中的类型称为 file_type，而用来标注进程的安全上下文的类型称为 domain，并且每一个用来描述文件安全上下文的类型都将 file_type 设置为其属性，每一个用来进程安全上下文的类型都将 domain 设置为其属性。

0x23 SEAndroid 配置文件
-------------------

四种类型的对象的安全上下文，分别是 App 进程、App 数据文件、系统文件和系统属性。这四种类型对象的安全上下文通过四个文件来描述：mac_permissions.xml、seapp_contexts、file_contexts 和 property_contexts，这几个文件通常在`/sys/fs/selinux`（早期版本）或`/etc/selinux/`目录下：

```
$ ls -ahlZ

ls: ./plat_hwservice_contexts: Permission denied

ls: ./plat_mac_permissions.xml: Permission denied

total 536K

drwxr-xr-x  3 root root u:object_r:system_file:s0            4.0K 2009-01-01 16:00 .

drwxr-xr-x 17 root root u:object_r:system_file:s0            4.0K 2009-01-01 16:00 ..

drwxr-xr-x  2 root root u:object_r:system_file:s0            4.0K 2009-01-01 16:00 mapping

-rw-r--r--  1 root root u:object_r:sepolicy_file:s0            65 2009-01-01 16:00 plat_and_mapping_sepolicy.cil.sha256

-rw-r--r--  1 root root u:object_r:file_contexts_file:s0      23K 2009-01-01 16:00 plat_file_contexts

-rw-r--r--  1 root root u:object_r:property_contexts_file:s0 6.5K 2009-01-01 16:00 plat_property_contexts

-rw-r--r--  1 root root u:object_r:seapp_contexts_file:s0    1.2K 2009-01-01 16:00 plat_seapp_contexts

-rw-r--r--  1 root root u:object_r:sepolicy_file:s0          0.9M 2009-01-01 16:00 plat_sepolicy.cil

-rw-r--r--  1 root root u:object_r:service_contexts_file:s0   14K 2009-01-01 16:00 plat_service_contexts


```

AOSP 提供的所有 Android 策略文件都在源码路径 external/sepolicy 目录下面，在编译完成之后一共会生成如下个 module:

0x31 sepolicy
-------------

sepolicy 其主要用于配置进程的安全上下文用来控制进程访问内核资源 (文件，端口等等) 策略和设置虚拟文件系统的安全上下文，系统启动之后，会由 init 进程在 `/sys/fs /selinux`中安装一个 selinux 虚拟文件系统，接着再加载 SEAndroid 安全策略到内核空间的 selinux lsm 模块中去。

sepolicy 文件其实就是 SEAndroid 的安全策略配置文件，里面有所有进程的权限配置，进程只能进行它的权限规定内的操作。这个文件 root 权限也删不掉，把这个文件的内容 dump 出来后会发现里面有好多条规则，看两条例子：

```
allow untrusted_app system_app_data_file : file { read }     
allow zygote sdcard_type : file { read write creat rename }


```

它的具体格式为：`allow Domain Type : Class { Permission }` (Domain 是指进程的 type)  
通过这个格式解读上面的两条规则就是：

1.  允许 untrusted_app 类型的进程对 system_app_data_file 类型的文件进行 read。
2.  允许 zygote 类型的进程对 sdcard_type 的 file 进行 read write creat rename。

所以 MAC 控制方式是这个样子的：  
当一个进程去操作一个文件的时候，系统会去检测这个进程和文件的上下文，看看这个进程的所属的 type 有没有对这个的文件的 type 操作的权限。比如：zygote 如果要去读 sdcard 上的一个文件，这个文件的 type 为 sdcard，zygote 的 type（Domain）为 zygote， 系统去检测看看发现这条规则 `allow zygote sdcard_type : file { read write creat rename }`。那么这个操作就会被允许执行。zygote 要是想删除一个 sdcard 上的文件，系统发现对应的规则里没有 delete，那么就会被 deny。

0x32 file_contexts
------------------

file_contexts 模块用于设置打包在 ROM 里面的文件的安全上下文。其是由`external/sepolicy/file_contexts`文件编译而成。

例如在`build systemimage`时会将这个 file_contexts 文件路径传递给命令 make_ext4fs 时，就会根据它设置的规则给打包在 system.img 里面的文件关联安全上下文。这样就获得了一个关联有安全上下文的 system.img 镜像文件了。

通过 fastboot 命令将 system.img 刷入 system 分区 mount 到 / system 目录之后，因为设置了相应的安全上下文，这样就能控制进程访问 system 目录下相关文件.

0x33 seapp_contexts 和 mac_permissions.xml
-----------------------------------------

`seapp_contexts`是负责设置 APP 数据文件的安全上下文，`mac_permissions.xml`是负责设置 APP 进程的安全上下文.  
路径: `external/sepolicy/mac_permissons.xml`  
文件`mac_permissions.xml`给不同签名的 App 分配不同的 seinfo 字符串，例如，在 AOSP 源码环境下编译并且使用平台签名的 App 获得的 seinfo 为 “platform”，使用第三方签名安装的 App 获得的 seinfo 签名为”default”。这个 seinfo 描述的是其实并不是安全上下文中的 Type，它是用来在另外一个文件 seapp_contexts 中查找对应的 type 的。  
路径：`external/sepolicy/seapp_context`

```
sisSystemServer=true domain=system_server
user=system domain=system_app type=system_app_data_file
user=bluetooth domain=bluetooth type=bluetooth_data_file
user=nfc domain=nfc type=nfc_data_file
user=radio domain=radio type=radio_data_file
user=shared_relro domain=shared_relro
user=shell domain=shell type=shell_data_file
user=_isolated domain=isolated_app
user=_app seinfo=platform domain=platform_app type=app_data_file
user=_app domain=untrusted_app type=app_data_file


```

例如，于使用平台签名的 App 来说，它的 seinfo 为`platform`。用户空间的 SecurityServer(比如 installd) 在为它查找 对应的 Type 时，使用的 user 输入为`_app`。这样在`seapp_contexts`文件中，与它匹配的一行即为：`user=_app seinfo=platform domain=platform_app type=app_data_file` 这样就可以知道，使用平台签名的 App 所运行在的进程 domain 为`platform_app`，并且它的数据文件的`file_type`为`app_data_file`。

0x34 property_contexts
----------------------

在 Android 系统中，有一种特殊的资源——属性，App 通过读写它们能够获得相应的信息，以及控制系统的行为，因此，SEAndroid 也需要对它们进行保护。这意味着 Android 系统的属性也需要关联有安全上下文。  
路径：external/sepolicy/property_contexts

```
##########################
# property service keys
#
#
net.rmnet               u:object_r:net_radio_prop:s0
net.gprs                u:object_r:net_radio_prop:s0
net.ppp                 u:object_r:net_radio_prop:s0
...


```

属性的安全上下文与文件的安全上下文是类似的，将属性`net.rmnet`设置为`u:object_r:net_radio_prop:s0`， 意味着只有有权限访问 type 为 net_radio_prop 的资源的进程才可以访问这个属性。

0x35 service_contexts
---------------------

在 AndroidL 系统中，还将服务 (属于进程或线程) 也作为一种资源来定义，给其定义对应的安全上下文，用来保护服务不被恶意进程访问.  
路径：`external/sepolicy/service_contexts`

```
...
window                                    u:object_r:system_server_service:s0
*                                         u:object_r:default_android_service:s0


```

服务的安全上下文同样与文件的安全上下文是类似的，将 window 服务设置为 u:object_r:system_server_service:s0，意味着只有有权限访问 type 为 system_server_service 的资源的进程才可以访问这个服务。

0x41 绕过 SEAndroid 几个方法
----------------------

由于 SELinux 引入 android 不久，还有很多不完善的地方。在 DEFCON 21 上，来自 viaForensics 的 Pau Oliva 就演示了几个方法来[绕过 SEAndroid](https://viaforensics.com/mobile-security/implementing-seandroid-defcon-21-presentation.html)：

1.  用恢复模式（recovery）刷回 permissive 模式的镜像
2.  Su 超级用户没有设置 SELinux 模式权限，但是 system user 系统用户可以。
3.  Android 通过 / system/app/SEAndroidManager.apk 来设置 SELinux 模式，所以只要在 recovery 模式下将其删除就可以绕过
4.  在 Android 启动时直接操作内核内存，通过将内核里的 unix_ioctl 符号改写成 reset_security_ops 重置 LSM（Linux Security Modules）

0x42 Fastboot 中将 SELinux 更改为 Permissive 模式
------------------------------------------

```
fastboot oem selinux permissive
...
OKAY [  0.045s]
finished. total time: 0.047s
....
OnePlus3:/ $ getenforce
Permissive
OnePlus3:/ $


```

​