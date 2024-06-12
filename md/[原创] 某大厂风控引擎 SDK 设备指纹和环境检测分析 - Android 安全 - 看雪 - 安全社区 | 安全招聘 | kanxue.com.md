> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280869.htm)

> [原创] 某大厂风控引擎 SDK 设备指纹和环境检测分析

前言：
---

###### 闲来无事，花了亿点时间将某大厂的风控引擎 sdk 看了一遍，主要分析对象是 sdk 里面的 libNetHTProtect.so。

通过初步分析，请求网址 / v4/c 是初始化的步骤，请求网址 / v4/a/up 就是关键的设备参数的上传。

![](https://bbs.kanxue.com/upload/attach/202403/978841_6NHBNR4KB7N4GT2.webp)

每次登录的操作都会发送这个请求，看来这个包应该就是数据上传的点了，不过很可惜，数据加密了，先到 so 里去定位吧。

不过当我进到 so 里的时候，发现 so 中的字符串都被加密了，运行时才会解密并且是在栈空间里面储存。  

不过好在算法很简单，就是异或或者其他的运算，写个脚本批量解就好了，这里大约有 5000 多行：

![](https://bbs.kanxue.com/upload/attach/202403/978841_WVPVKVNTXSUBBTN.webp)

即使如此，还是有部分算法没解出来，不过不要灰心，由于解密之后会放到一个栈空间中，之后这块内存将被回收，hook 下 free 函数，也能打印出不少东西：

![](https://bbs.kanxue.com/upload/attach/202403/978841_VVWZT5QQ797HECM.webp)

这些数据就成为线索了，解密出一部分数据之后，找到上面的 / v4/a/up 的链接，很快就能分析出对应的请求函数了。

这里它是反射 java 层的函数去请求的，在请求之前就将数据加密了。

![](https://bbs.kanxue.com/upload/attach/202403/978841_8QBWP9KD8BBT7VD.webp)

请求的数据在收集完成之后，使用 protobuf 序列化方便传输，然后就被送进加密函数里加密了，数据在请求前放到 sub_2CD2A8 中去加密，加密算法简单看了下是 AES，并且 key 是随机生成的

![](https://bbs.kanxue.com/upload/attach/202403/978841_78KVNXU8HVRFV6N.webp)

这里就不逆加密算法了，直接上 frida，把原始 protobuf 数据拿到手。由于网络请求基本上都会走这里，所以在 hook 这个函数的同时，hook 他的上一跳函数，把请求的 URL 也打印出来，代码很简单：

```
  function hook_2CD2A8() {
    var so_name = "libNetHTProtect.so"; // hook的so名称
    var so_base = Module.findBaseAddress(so_name);
    var offset__exit = 0x2CD2A8;
    var true_add = so_base.add(offset__exit);
    var flag_ptr_exit = new NativePointer(true_add);
    Interceptor.attach(flag_ptr_exit, {
      onEnter: function (args) {
        var lenx = Memory.readInt(ptr(args[1]).add(0x8));
        console.log("请求数据："+ hexdump(Memory.readPointer(ptr(args[1]).add(0x10)), { length: lenx }));
      },
      onLeave: function (retval) {
      },
    });
  }
  function hook_2A846C() {
    var so_name = "libNetHTProtect.so"; // hook的so名称
    var so_base = Module.findBaseAddress(so_name);
    var offset__exit = 0x2A846C;
    var true_add = so_base.add(offset__exit);
    var flag_ptr_exit = new NativePointer(true_add);
    Interceptor.attach(flag_ptr_exit, {
      onEnter: function (args) {
        console.log("访问URL："+ Memory.readCString(args[0]) + Memory.readCString(args[1])  );
      },
      onLeave: function (retval) {
      },
    });
  }

```

这样就能把请求的 protobuf 数据拿到了。

![](https://bbs.kanxue.com/upload/attach/202403/978841_QGWRP6BE24E96JR.webp)

然后再用 protoc 反序列化一下，差不多收集上传了有 350 多条数据。

详细的分析过程有点复杂，但可以大致将设备指纹和环境检测的项说明一下，看看大厂是怎么做的，大厂做的也是基本部分，大部分检测原理还是相似的。

由于这里差不多收集了 300 多项，这里把部分重点的收集项简单说一下.

设备指纹：
-----

##### SD 卡 / storage/emulated/0 下可用空间大小和总空间大小

基本思路是调用 stat 函数，去获取 / storage/emulated/0 目录下的块大小和总数据块数量，还有空闲块的数量。

代码逻辑很明显，通过 syscall 去调用 statfs 函数，然后获取 stat 结构体中的数据，最后算一下：

![](https://bbs.kanxue.com/upload/attach/202403/978841_T7HE7SR79BDJW59.webp)

![](https://bbs.kanxue.com/upload/attach/202403/978841_WNUAM8MGB5FJYWC.webp)

##### 获取 DRM ID

这个是很多大厂很核心的设备指纹，包括某美，某盾的 sdk 都在收集。  

具体是通过 dlsym 去导入链接上 libmediandk.so，然后调用 getPropertyByteArray 函数去获取：

![](https://bbs.kanxue.com/upload/attach/202403/978841_TAE8RE9GX6SQEB2.webp)

##### system prop

这个也不说了，也是必收集的字段，通过 system_property_get 函数去获取，包括以下字段，其中既有与设备指纹相关的，比如说 buildid 和 fingerprint 等等，也有可以用于风控的（sys.usb.state，ro.setupwizard.mode 这些）

先看下通用的：

```
ro.build.version.release
ro.product.model
ro.product.brand
ro.boot.bootloader
ro.build.version.securitypatch
ro.build.version.incremental
gsm.version.baseband
gsm.version.ril-impl
ro.build.fingerprint
ro.build.description
ro.build.product
ro.boot.vbmeta.digest
ro.hardware
ro.product.name
ro.product.board
ro.recovery_id
ro.expect.recovery_id
ro.board.platform
ro.product.manufacturer
ro.product.device
sys.usb.state
ro.setupwizard.mode
ro.build.id
ro.build.tags
ro.build.type
ro.debuggable

```

然后还有一些不常见的，可能是其他厂商的设备中才会有的字段：

```
persist.sys.meid
vendor.serialno
sys.serialno
persist.sys.wififactorymac
ro.boot.deviceid
ro.rpmb.board
ro.vold.serialno
persist.oppo.wlan.macaddress
persist.sys.oppo.serialno
ril.serialnumber
ro.boot.ap_serial
ro.boot.uniqueno
persist.sys.oppo.opmuuid
persist.sys.oppo.nlp.id
persist.sys.oplus.nlp.id
persist.sys.dcs.hash
ro.ril.oem.sno
ro.ril.oem.psno
persist.vendor.sys.fp.uid
ro.ril.miui.imei0
ro.ril.miui.imei1
ro.ril.oem.imei
ro.ril.oem.meid
persist.radio.imei
persist.radio.imei1
persist.radio.imei2
persist.sys.lite.uid
persist.radio.serialno
vendor.boot.serialno
persist.sys.oneplus.serialno
ro.meizu.hardware.imei1
ro.meizu.hardware.imei2
ro.meizu.hardware.meid
ro.meizu.hardware.psn
ro.meizu.hardware.sn
persist.radio.factory_phone_sn
persist.radio.factory_sn
ro.meizu.serialno
ro.boot.psn
ro.boot.meid
ro.boot.imei1
ro.boot.imei2
ro.wifimac
ro.wifimac_2
ro.vendor.deviceid
ro.isn
ro.vendor.isn
persist.radio.device.imei
persist.radio.device.imei2
persist.radio.device.meid
persist.radio.device.meid2
persist.asus.serialno
sys.wifimac
sys.bt.address
persist.btpw.bredr
persist.radio.imei
persist.radio.imei2
persist.radio.meid
persist.radio.meid2
ro.boot.fpd.uid
ro.vendor.boot.serialno
ro.boot.wifimacaddr
persist.sys.wifi.mac
persist.sys.wifi_mac
sys.prop.writeimei
ril.gm.imei
ril.cdma.meid
ro.boot.em.did
ro.qchip.serialno
ro.ril.oem.btmac
ro.ril.oem.ifimac

```

还有一部分用于设备识别和风控的，我会在后面放出来。  

##### 设备标识

这个就很多了，比如说 IMEI、androidID 都可以作为设备标识，这里还是把它收集的设备标识列出来。

大概就是这些：  

```
uuid
ad_aaid
ReaperAssignedDeviceId
IMEI
mdm_uuid
ps_imei
op_security_uuid
ai_stored_imei
device_serial

```

##### 通过 stat 获取文件的最近访问时间、最近修改时间、最近改变时间，Innode 编号  

这个也有很多大厂用了，主要针对以下一些文件和文件目录，读取文件结构体，然后上传修改时间等信息。  

```
/sdcard/Android/data/.nomedia
/sdcard/Android/data/com.google.android.gms
/sdcard/
/storage/emulated/0

```

##### 硬件相关

这个很常见，一般主要有：  
cpu 核心数以及最大频率

CPU 型号  

内存（RAM）可用空间和总空间

cpu 支持的系统架构

屏幕亮度、屏幕尺寸和屏幕超时时间

传感器生产厂家和类型（重要）

显示设备厂商名称（GL_VENDOR）和渲染器名称（GL_RENDERER）（重要）

内部存储（EMMC 或 UFS 闪存）的序列号：/sys/block/mmcblk0/device/serial  （核心）

显示设备序列号：/sys/devices/soc0/serial_number  (核心)

内部存储 SD 卡的 CID：/sys/block/mmcblk0/device/cid（核心）

电池相关，例如电压、电池容量、电池温度、电池健康百分比、充电状态等

input 设备相关，读取 /proc/bus/input/devices，获取注册的 input 设备信息，比如 Name 和 Sysfs。

##### 网络相关

比较常见的就是局域网接口 ip、mac、子网掩码、dns 服务器列表了，这个就不说了，基本上就是 SVC 获取网卡的信息。

##### 文件哈希

由于部分文件从系统安装后基本上不会有变化，因此很多大厂用了某些路径下的文件，通过一定的策略，比如说 stat 选择谋个目录下修改时间最早的（一般是 1970 年几月几日），然后将这个文件算个 hash，或者直接调用 getdents64 函数，去获取某一目录下的文件和文件夹，按照修改时间从小到大排列起来，然后将排列起来的文件名算个 hash，亦或者获取文件名 + 文件 inode 号（st_ino）+ 文件结构体长度（d_reclen），然后计算哈希。

总之策略有很多种，效果如何需要运营验证。

一般来说有几个重要的文件目录，大厂的处理方法具有相似性。  

1、/data/system

2、/vendor/firmware

3、/system/bin

4、/vendor/lib

5、/system/framework

6、/system/fonts

##### boot id

cat 命令读取 /proc/sys/kernel/random/boot_id，这个也是大厂必用的一个关键的设备指纹字段。

##### apk 路径

读取设备上的应用安装列表以及 apk 路径，然后选择部分包名的路径作为设备指纹。

比如微信的随机路径 ，作为核心的唯一设备指纹，只要你不卸载重装微信，这个随机的路径就不会有变化

例如：  

![](https://bbs.kanxue.com/upload/attach/202403/978841_CTYW9SNWT4DHERM.webp)

##### 输入法列表

有些 SDK 会收集设备上安装的输入法列表，一般来说国内的 HMOV 手机预装的是定制搜狗输入法，谷歌 pixel 是谷歌原生输入法。

举个栗子，这是 SDK 上传的输入法信息：  

![](https://bbs.kanxue.com/upload/attach/202403/978841_2GRC72VSVRZ5W8C.webp)

获取输入法列表的方法：

```
ArrayList arrayList = new ArrayList();
InputMethodManager inputMethodManager = (InputMethodManager)context.getSystemService("input_method");            
List list = inputMethodManager.getInputMethodList();
Iterator iterator = list.iterator();
while(true) {
    if(!iterator.hasNext()) {
        return arrayList;
    }
Object object = iterator.next();
arrayList.add(((InputMethodInfo)object).toString());
}

```

环境检测：
-----

### 一、HOOK 环境

在 libNetHTProtect.so 中主要检测了 crc 校验，还有 frida 和 Xposed 的一些特征：

##### 1、检测 frida 的特征：

进程 / proc/self/fd / 目录，寻找是否存在关键字 linjector

/data/local/tmp / 下是否存在 re.frida.server

##### 2、检测 Xposed 特征：

方法 1：检测下列文件或文件路径是否存在：  
[](http://libzposed_art.so/)

```
/sbin/.magisk/modules/riru_lsposed
/data/adb/lspd
/sbin/.magisk/modules/zygisk_lsposed
/sbin/.magisk/modules/riru_edxposed
/data/misc/riru/modules/edxp
/data/adb/riru/modules/edxp.prop
/sbin/.magisk/modules/taichi
/data/misc/taichi
/sbin/.magisk/modules/dreamland
/data/misc/riru/modules/dreamland
/data/adb/riru/modules/dreamland
/system/bin/app_process.orig
/system/xposed.prop
/system/framework/XposedBridge.jar
/system/lib/
libxposed_art.so
/system/lib/
libxposed_art.so
.no_orig
/system/lib64/
libxposed_art.so
/system/lib64/
libxposed_art.so
.no_orig
/system/bin/app_process_zposed
/system/framework/ZposedBridge.jar
/system/lib/
libzposed_art.so

```

方法 2：Hook loadClass 加载类，检查程序加载的 Class 中是否包含名为 de.robv.android.xposed.XposedBridge 的类名路径。

方法 3: 遍历 / data/data / 目录，寻找有无以下包名：

```
de.robv.android.xposed.installer #Xposed框架
org.meowcat.edxposed.manager #EdXposed框架
com.tsng.hidemyapplist #隐藏应用列表
com.tsng.hidemyroot #隐藏Root
org.lsposed.manager #LSPosed框架
me.weishu.exp #VirtualXposed 太极
top.canyie.dreamland.manager #VirtualXposed 夢境
io.va.exposed #VirtualXposed 夢境
io.va.exposed64 #VirtualXposed 夢境
io.virtualapp #VirtualApp
io.virtualapp.sandvxposed64 #VirtualApp

```

##### 3、libc 的 CRC 校验  

拿到 proc/self/maps 中 libc.so 的基地址，并计算 ELF 文件中. text+.rodata+.eh_frame+.eh_frame_hdr 段的 CRC 校验值。

### **二、机型和系统判定**

１、检测是否是 lineageOS 系统，这个 sdk 主要的检测方法是：

```
1、查找是否存在特征文件：
/system/framework/org.lineageos.platform.jar
/system/framework/org.lineageos.hardware.jar
2、检测systemprop中是否存在：ro.lineage.build.version

```

2、检测是否是 OpenHarmony 系统，检测方法是：

```
检测systemprop中是否存在：ro.build.ohos.devicetype

```

3、检测品牌机系统和设备能否对应，检测方法是：  

```
遍历/system/framework/下所有以.jar结尾的文件名称，与机型字符串进行比较（strcasestr比较，不区分大小写）

```

例如在 vivo 手机内，/system/framework / 下存在以下文件，文件名中包含 vivo：

![](https://bbs.kanxue.com/upload/attach/202403/978841_VAYEZVFNKZX8YXW.webp)

### 三、Bootloader 解锁状态

检测 bootloader 是否已解锁。

方法 1：检测 props，主要检测项：

```
1、检测ro.boot.verifiedbootstate的值是否是orange
2、检测ro.secureboot.lockstate的值是否是unlocked
3、检测vendor.boot.vbmeta.device_state的值是否是unlocked
4、检测vendor.boot.verifiedbootstate的值是否是orange
5、检测ro.boot.vbmeta.device_state的值是否是unlocked
6、检测ro.boot.flash.locked的值是否是unlocked

```

方法 2：获取 sys.oem_unlock_allowed 的值

### 四、SIM 卡

这个可以检测 SIM 卡是否存在，获取 SIM 卡的网络类型，SIM 卡的 ICCID 等。  

1. 检测 SIM 卡是否存在, 这个一般是读取 props, 获取 gsm.sim.state 这个的属性

2. 获取 SIM 卡的网络运营商, 通过系统服务 TELEPHONY_SERVICE, 然后调用 getSubscriberId 获取 IMSIString, 然后判断前 5 位:

中国地区的运营商以及对应的号段见下表:

```
中国移动 ：46000，46002，46004，46007，46008
中国联通 ：46001，46006，46009，46010
中国电信 ：46003，46005，46011，46012
中国广电 ：46015
港澳地区：
中国移动（香港） ：45412，45413，45430
中国电信（香港） ：45407
中国电信（澳门） ：45502，45507

```

### 五、截图工具和模拟点击工具

1. 检测运行的服务中是否存在 minicap、minitouch、stfservice、minitouchagent、stfagent、testminicap 关键字。

2. 检查 / data/local/tmp / 有无以下文件或文件目录：

```
minicap.so
minicap
minitouch
mini
mini/minicap
oat/arm64/scrcpy-server.odex
vysor.pwd
mobile_info.properties
tc/mobileagent
tc/input3.sh
tc/mainputjar7
com.cyjh.mobileanjian.id
com.cyjh.mobileanjianen.id
juejinAzykb/
minicap.so
juejinAzykb/TouchService.jar
mqc-scrcpy.jar
uiautomator-stub.jar
cloudtestig/cloudscreen
cloudtesting/touchserver
txysvr.apk
yijianwanservice.apk
screen-shread10x64.so
screen-shread5x32.so
maxpresent.jar
libtxysvr.so

```

3. 检查应用安装列表

检查有没有安装下列模拟点击的应用：  

```
com.cygery.repetitouch.pro
com.cyjh.mobileanjian
com.touchsprite.android
com.cjzs123.zhushou
com.touchspriteent.android
com.zidongdianji
org.autojs.autojspro
org.autojs.autojs
com.zdanjian.zdanjian
com.zdnewproject
com.ifengwoo.zyjdkj
com.angel.nrzs
com.cyjh.mobileanjian.vip
com.shumai.shudaxia
fun.tooling.clicker.cn
com.dianjiqi
com.miaodong.autoactionssss
com.mxz.wxautojiafujinderen
com.touchelf.app
com.stardust.scriptdroid
com.adinall.autoclick
com.i_cool.auto_clicker
com.kongshan.aidianji
com.xptech.catclicker
com.tingniu.autoclick
com.yicu.yichujifa
com.smallyin.autoclick
com.ksxkq.autoclick
com.x2.clicker
com.scott.autoclickhelper
com.auyou.auyouwzs

```

 检查有没有安装下列截图应用：  

```
com.github.uiautomator
com.github.uiautomator2
com.sigma_rt.totalcontrol
com.genymobile.scrcpy

```

### 六、模拟器

扫描常见的模拟器特征，主要检测的方法是：  

1、是否存在以下文件目录：

```
/system/bin/ldinit
/system/bin/ldmountsf
/system/lib/libldutils.so
/system/bin/microvirt-prop
/system/lib/libdroid4x.so
/system/bin/windroyed
/system/lib/libnemuVMprop.so
/system/bin/microvirtd
/system/bin/nox-prop
/system/lib/libnoxspeedup.so
/data/property/persist.nox.simulator_version
/data/misc/profiles/ref/com.bignox.google.installer
/data/misc/profiles/ref/com.bignox.app.store.hd
/system/bin/ttVM-prop
/system/bin/droid4x-prop
/system/bin/duosconfig
/system/etc/xxzs_prop.sh
/system/etc/mumu-configs/device-prop-configs/mumu.config
/boot/bstsetup.env
/boot/bstmods
/system/xbin/bstk
/data/bluestacks.prop
/data/data/com.anrovmconfig
/data/data/com.bluestacks.appmart
/data/data/com.bluestacks.home
/data/data/com.microvirt.market
/dev/nemuguest
/data/data/com.microvirt.toolst
/data/data/com.mumu.launcher
/data/data/com.mumu.store
/data/data/com.netease.mumu.cloner
/system/bin/bstshutdown
/sys/module/bstinput
/sys/class/misc/bstXqpb
/system/phoenixos
/xbin/phoenix_compat
/init.dundi.rc
/system/etc/init.dundi.sh
/data/data/com.ddmnq.dundidevhelper
/init.andy.cloud.rc
/system/bin/xiaopiVM-prop
/system/bin/XCPlayer-prop
/system/lib/liblybox_prop.so
/system/bin/tencent_virtual_input
/vendor/bin/init.tencent.sh
/data/youwave_id
/dev/vboxguest
/dev/vboxuser
/sys/bus/pci/drivers/vboxguest
/sys/class/bdi/vboxsf-c
/sys/class/misc/vboxguest
/sys/class/misc/vboxuser
/sys/devices/virtual/bdi/vboxsf-c
/sys/devices/virtual/misc/vboxguest
/sys/devices/virtual/misc/vboxuser
/sys/module/vboxguest
/sys/module/vboxsf
/sys/module/vboxvideo
/system/bin/androVM-vbox-sf
/system/bin/androVM_setprop
/system/bin/get_androVM_host
/system/bin/mount.vboxsf
/system/etc/init.androVM.sh
/system/etc/init.buildroid.sh
/system/lib/vboxguest.ko
/system/lib/vboxsf.ko
/system/lib/vboxvideo.ko
/system/xbin/mount.vboxsf
/dev/goldfish_pipe
/sys/devices/virtual/misc/goldfish_pipe
/sys/module/goldfish_audio
/sys/module/goldfish_battery
/sys/module/kvm_intel/
/sys/module/kvm_amd/
/init.android_x86_64.rc
/init.android_x86.rc
/init.androidVM_x86.rc
/init.intel.rc
/init.vbox2345_x86.rc

```

2、system.prop 下是否存在以下字段特征：

```
init.svc.microvirtd
bst.version
ro.phoenix.version.code
ro.phoenix.version.codename
init.svc.droid4x
microvirt.memu_version
microvirt.imsi
microvirt.simserial
ro.px.version.build
ro.phoenix.os.branch
init.svc.su_kpbs_daemon
init.svc.noxd
init.svc.ttVM_x86-setup
init.svc.xxkmsg
ro.bild.remixos.version
microvirt.mut
init.svc.ldinit
sys.tencent.os_version
sys.tencent.android_id
ro.genymotion.version
init.svc.pkVM_x86-setup
ro.andy.version
ro.build.version.release
ro.product.model
ro.product.brand
ro.boot.bootloader
ro.build.version.securitypatch
ro.build.version.incremental
gsm.version.baseband
gsm.version.ril-impl
ro.build.fingerprint
ro.build.description
ro.build.product
ro.boot.vbmeta.digest
ro.hardware
ro.product.name
ro.product.board
ro.recovery_id
ro.expect.recovery_id
ro.board.platform
ro.product.manufacturer
ro.product.device
sys.usb.state
ro.setupwizard.mode
ro.build.id
ro.build.tags
ro.build.type
ro.debuggable

```

3. 检查挂载点, 是否存在以下目录:  

```
/mnt/shared/Sharefolder
/tiantian.conf
/data/share1
/hardware_device.conf
/mnt/shared/products
/mumu_hardware.conf
/Andy.conf
/mnt/windows/BstSharedFolder
/bst.conf
/mnt/shared/Applications
/ld.conf

```

检查挂载点, 检查 /proc/mounts 中是否存在 vboxsf 字段.

4. 检查是否安装了虚拟机软件:

```
1.VMOS虚拟机
方法1：运行服务查找：tomq_MANAGER_SERVICE
方法2：ROOT_DIR查找VMOS_SYS_NUM
方法3：/proc/self/root/data/data/查找包名：“com.vmos.app”、“com.vmos.pro”、“com.vmos.ggp”
方法4：props查找：vmprop.androidid、vmprop.dev_ashmem、vmprop.ip、ro.vmos.simplest.rom
2.X8沙箱
方法1:props查找：ro.x8.version、ro.x8.uuid
方法2:查找文件目录：/x8/config/root.pkg.blacklist、/x8/config/full_vm是否存在
3.51虚拟机
/proc/self/root/data/data/查找包名：“com.f1player”、“com.f1player.play”
4.虚拟精灵和虚拟大师
/proc/self/root/data/data/查找包名：com.pspace.vandroid、com.yiqiang.xmaster

```

### 七、改机软件

1. 检测是否安装了改机软件:  

```
检查包名是否存在:
com.yztc.studio.plugin
com.soft.apk008v
com.uwish.app
zpp.wjy.xxsq
com.bigsing.changer
zap.fh.wipe
com.sollyu.xposed.hook.model
com.soft.apk008Tool
com.doubee.ig
com.variable.apkhook
com.addeasy.fastest
com.xenice.mask
com.shyl.artifact
com.android1500.androidfaker

```

2. 检查 / dev/wgzs 目录下的内容是否存在，若存在则获取值：

```
wg.cust.config.phone.id
wg.cust.s_android_id
wg.cust.sys.prop.filter
wg.cust.config.phone.imeimackey
wg.cust.config.phone.imei
wg.cust.config.pkg.name
wg.cust.destUids

```

### 八、Root 和 Root 工具

1. 包名检测, 遍历 / data/data / 目录，寻找有无以下包名：

```
com.topjohnwu.magisk
eu.chainfire.supersu
com.noshufou.android.su
com.noshufou.android.su.elite
com.koushikdutta.superuser
com.thirdparty.superuser
com.yellowes.su
com.fox2code.mmm
io.github.vvb2060.magisk
com.kingroot.kinguser
com.kingo.root
com.smedialink.oneclickroot
com.zhiqupk.root.global
com.alephzain.framaroot
io.github.huskydg.magisk
me.weishu.kernelsu

```

2. 检查 su 文件是否存在

```
/su/bin/su
/sbin/su
/data/local/xbin/su
/data/local/bin/su
/data/local/su
/system/xbin/su
/system/bin/su
/system/sd/xbin/su
/system/bin/failsafe/su
/system/bin/.ext/.su
/system/etc/.installed_su_daemon
/system/etc/.has_su_daemon
/system/xbin/sugote
/system/xbin/sugote-mksh
/system/xbin/supolicy
/system/etc/init.d/99SuperSUDaemon
/system/.supersu
/product/bin/su
/apex/com.android.runtime/bin/su
/apex/com.android.art/bin/su
/system_ext/bin/su
/system/xbin/bstk/su
/system/app/SuperUser/SuperUser.apk
/system/app/Superuser.apk
/system/xbin/mu_bak
/odm/bin/su
/vendor/bin/su
/vendor/xbin/su
/system/bin/.ext/su
/system/usr/we-need-root/su
/cache/su
/data/su
/dev/su
/system/bin/cph_su
/dev/com.koushikdutta.superuser.daemon
/system/xbin/daemonsu
/sbin/.mianju
/sbin/nvsu
/system/bin/.hid/su
/system/addon.d/99-magisk.sh

```

3. 检查 magisk 相关文件是否存在  

```
/cache/.disable_magisk
/dev/magisk/img
/sbin/.magisk
/cache/magisk.log
/data/adb/magisk
/system/etc/init/magisk
/system/etc/init/magisk.rc
/data/magisk.apk

```

4. 检测 Android PATH 环境变量，检测 path 路径下是否存在 “/su”

5. 检测 /proc/self/maps 中的内容

```
1、检测 /proc/self/maps 是否存在名为“/memfd:/jit-cache”的段（加载zygisk模块时（也就是liblspd.so)的时候会讲其名称设置为jit-cache，这样的话so的内存段在maps中就是/memfd:/jit-cache）
2、通过检测map表是否存在匿名的并且具有可执行属性的内存判断是否存在lsposed
3、检测栈空间[stack]的权限是否为“rw-p”

```

6. 检测 ro.build.tags 的值，读取 /system/build.prop 并检测 ro.build.tags 的值是否为 “test-keys”  

7. 检测 seLinux 安全上下文，cat /proc/%d/attr/prev 检测 app 进程的 selinux 安全上下文是否为 “u:r:zygote:s0”

8. 检查 / data/local/tmp / 有无以下文件

```
shizuku
shizuku_starter

```

### 九、ebpf 检测

```
1、系统调用 prctl(PR_GET_SECCOMP)用来获取seccomp的状态，返回值为0时代表进程没有被施加seccomp，否则代表进程被施加seccomp。
2、系统调用 prctl(PR_GET_NO_NEW_PRIVS)获取线程的no_new_prives属性，值为0表示常规，值为1代表在特权中操作。

```

其他的都是比较常规的环境检测, 比如 VPN 和代理，USB 调试和开发者状态，是否开启了无障碍服务，以及调试 TracerPid 等，太多了就不一一列举了。

除了上面比较普通的环境检测之外，该大厂还引入了 AI 检测模拟点击，接口函数的原型是这样的:

![](https://bbs.kanxue.com/upload/attach/202403/978841_WSW6FXYS682KGFB.webp)

看了下代码, 我觉得原理就是 adb shell -> getevent, 当在手机屏幕点击某个指定的 X,Y 坐标位置后，在命令行窗口可见监听到很多 event，类似以下内容

![](https://bbs.kanxue.com/upload/attach/202403/978841_XXDHFKM6TD6W74P.webp)

然后它将 event 的 [type] [code] [value] 全部上传了，去做后端的 AI 检测。

一种可能的判断策略猜测：

作弊和工作站的批量模拟点击，其获取的 X,Y 坐标位置等数据每次都相同，而真人使用设备的滑动点击操作，一定是随机的屏幕轨迹，不可能每次都是同样的 X，Y 坐标值。

这个 SDK 的 AI 模拟点击检测，可能就是将某一场景的触发屏幕操作的数据，上传到后台，后台通过上述策略完成判断，这样就可以检测工作室的批量模拟点击了。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)