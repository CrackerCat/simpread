> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293142.htm#msg_header_h3_25)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

新手机到了必须狠狠的折腾（也没有多狠），之前用脱壳网站的时候对某 bang 壳的 app 进行脱壳会直接任务失败，那要是每次都去分析 so 中 dex 的解密逻辑那不麻烦死了，自己弄一个脱壳系统是比较一劳永逸的方案，可以在手机上先把检测项给过掉或者隐藏掉就可以美美脱壳了，社区内的前辈开源了许多的优秀项目作为参考（FART、youpk、FARText、r0dump 等等）

在壳保护对抗的过程中，往往伴随着各种各样的检测，譬如：frida 检测、root 检测、xp 检测、调试与 adb 检测、模拟器检测、完整性与多开检测、签名与 ROM 校验等等。比较痛苦的就是这些检测项被壳检测到了而触发闪退、上报等操作，影响分析。想要对抗一般需要对壳进行比较深入的挖掘与分析，或者有一定的经验可以快速定位 bypass。

那么依托于 syscall-trace 与隐藏 frida 特征能否减轻这部分的对抗，如前面的脱壳系统一样做到一定程度上的一劳永逸。

当然是可以的。通过 syscall-trace 可以快速知道这个 app 的某个 so 做了哪些疑点比较大的操作，比如说开辟了一块比较大的匿名内存，比如说将权限修改成 RWX，比如说 openat 了某个路径，这些信息都可以帮助我们在分析过程中以极低成本且能快速获取一部分信息用于加速分析进度。

隐藏 frida 特征主要是为了缩小壳保护的检测项范围，哪怕被检测到了也可以知道自己大概率是什么地方被检测到了，可以根据自己暴露的特征进行反向定位检测函数，从而达到过检测的目的。

本文是比较详细的入门级实操教程，详细记录了我为了实现上面一劳永逸工具组的落地流程，以及避坑指南。附件提供成品。

![](https://bbs.kanxue.com/upload/attach/202610/951483_SA5HASFZGZGEUN5.png)

先说明 解锁 BL 会清空设备数据 我这新手机就无所谓了.

能不能解锁就要先进入 Fastboot 模式，也就是这个页面。

![](https://bbs.kanxue.com/upload/attach/202610/951483_UQXYPEUZVF5YZ5A.png)

可以通过 adb 命令进入

```
adb reboot bootloader
```

然后去看，这里显示 unlockable 说明 OEM 解锁是允许的

![](https://bbs.kanxue.com/upload/attach/202610/951483_X8998NJNJ5QMGJ4.png)

首先打开开发者模式，关于本机里面狂点 build 号后即可进入开发者模式

![](https://bbs.kanxue.com/upload/attach/202610/951483_KECN4GRNWQ6CPGC.png)

然后找到开发者模式

![](https://bbs.kanxue.com/upload/attach/202610/951483_NF3UXRNEH7H2583.png)

打开 OEM

![](https://bbs.kanxue.com/upload/attach/202610/951483_DGBFH4C3BGSGWJ4.png)

下载一下[](https://bbs.kanxue.com/elink@1eeK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6V1k6i4k6W2L8r3!0H3k6i4u0Q4x3X3g2S2L8X3c8J5L8$3W2V1i4K6u0W2j5$3!0E0i4K6u0r3N6r3!0G2L8s2y4Q4x3V1k6J5k6h3I4W2j5i4y4W2M7#2)9J5c8Y4m8D9j5i4c8X3L8%4u0E0i4K6u0V1N6r3!0G2L8s2y4Q4x3@1k6Z5L8q4)9K6c8s2A6Z5i4K6u0V1j5$3^5%60.) platform-tools，里面有 adb 和 fastboot 两个工具

下载好后解压到文件夹

![](https://bbs.kanxue.com/upload/attach/202610/951483_8Q8764M5JEHEJ2M.png)

进入 fastboot

```
adb reboot bootloader
```

确认 fastboot 能认到设备：

我这里用的数据线无法找到设备，更换了买 iqoo 手机送的数据线就可以了，很多线连 adb 都无法识别到设备一般都是充电线，必须要去网上买数据线（明确标注 "支持数据传输 / Data Sync" 的线）

```
fastboot devices
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_7QMF3SYNKCGRXCA.png)

执行解锁

```
fastboot flashing unlock
```

执行完这个命令并非就是解锁好了，只是可以选择解锁了

![](https://bbs.kanxue.com/upload/attach/202610/951483_NGHMPJWD6XVSNNH.png)

还需要手动去选择，这里通过音量按键上下切换到 **Unlock the bootloader**

然后按下电源键确认

![](https://bbs.kanxue.com/upload/attach/202610/951483_ZA526H52RWQXPTG.png)

这样就是解锁好了

![](https://bbs.kanxue.com/upload/attach/202610/951483_9SNCXCC63P5CZ6F.png)

在安装路径创建这么两个文件夹 VMS 和 VMS\Ubuntu-LineageOS23

![](https://bbs.kanxue.com/upload/attach/202610/951483_9W4GHC273MQ4H8Z.png)

[](https://bbs.kanxue.com/elink@7c1K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0K9i4u0J5L8%4u0K6i4K6u0W2N6i4y4@1j5#2)9J5k6h3g2V1N6g2)9J5k6h3y4F1i4K6u0r3N6h3u0#2L8Y4c8#2i4K6u0V1M7X3g2D9k6h3q4K6k6i4y4Q4x3V1j5J5x3W2)9J5k6e0l9@1i4K6u0r3N6h3u0#2L8Y4c8#2i4K6u0V1x3U0u0Q4x3X3f1H3y4q4)9J5k6e0g2Q4x3X3c8V1k6i4y4C8N6r3!0H3i4K6u0V1j5h3#2V1y4U0c8Q4x3X3g2A6M7$3)9%60.)下载地址

[下载地址](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware Workstation Pro&freeDownloads=true)

![](https://bbs.kanxue.com/upload/attach/202610/951483_67PVZ42RJERHY2S.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_TBRSJESJ99FAC8U.png)

硬件兼容性: Workstation 17.x

安装来源：稍后安装操作系统

![](https://bbs.kanxue.com/upload/attach/202610/951483_VC5E8TJPXB6VKMD.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_ERHUS82QYQCTRQY.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_HRSZB46T9Y933PG.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_4WUMUG9PHMT32K2.png)

网络类型：使用网络地址转换

I/0 控制器类型：LSI Logic （L） 就是默认的那个

![](https://bbs.kanxue.com/upload/attach/202610/951483_SE83H28GDJKRYW9.png)

选择磁盘：创建新虚拟磁盘

![](https://bbs.kanxue.com/upload/attach/202610/951483_JMM573MSN9S2JYF.png)

这里点击自定义硬件

![](https://bbs.kanxue.com/upload/attach/202610/951483_K46PY5UG87AVZ7W.png)

勾选

![](https://bbs.kanxue.com/upload/attach/202610/951483_NJ8JYYPZTRBYN8T.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_5GD5X8ZV6WV54JA.png)

切换到 usb3.1

![](https://bbs.kanxue.com/upload/attach/202610/951483_NGEVUYVUJFM2GGS.png)

然后点击关闭，再点击完成

![](https://bbs.kanxue.com/upload/attach/202610/951483_XSPACH6CVXPGBYF.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_22C9AA5D5RUPYWJ.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_G3H994MVQXE8ZMG.png)

```
您的姓名:        aosp
您的计算机名:    aosp-vm
选择用户名:      aosp
选择一个密码:    1234
确认密码:        1234
```

我这边拉取 + 编译总共花了一天半左右，配置如下可以作为参考

![](https://bbs.kanxue.com/upload/attach/202610/951483_68MDVG89K9FZVBE.png)

#### 一、安装 VMware Tools 并更新系统

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y open-vm-tools open-vm-tools-desktop
sudo reboot
```

重启后分辨率自适应和剪贴板共享就正常了。

#### 二、安装编译依赖

```
sudo dpkg --add-architecture i386
sudo apt update

sudo apt install -y bc bison build-essential ccache curl erofs-utils flex \
  g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick \
  protobuf-compiler python3-protobuf lib32readline-dev lib32z1-dev libdw-dev \
  libelf-dev libgnutls28-dev lz4 libsdl1.2-dev libssl-dev libxml2 \
  libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc xxd \
  zip zlib1g-dev python-is-python3 libncurses5 libncurses5:i386 lib32z1 \
  android-sdk-platform-tools-common
```

其中 ​`libncurses5`​ 是必须的, AOSP 里部分预编译 host 工具仍然链接 ncurses5, 缺了会报 ​`libncurses.so.5: cannot open shared object file`​。​`android-sdk-platform-tools-common` 提供 adb 的 udev 规则, 否则非 root 下连手机会显示 no permissions。

#### 三、安装 platform-tools(adb / fastboot)

```
cd ~
wget https://dl.google.com/android/repository/platform-tools-latest-linux.zip
unzip platform-tools-latest-linux.zip -d ~
```

配置 PATH

```
cat >> ~/.profile << 'EOF'

# add Android SDK platform tools to path
if [ -d "$HOME/platform-tools" ] ; then
    PATH="$HOME/platform-tools:$PATH"
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi
EOF

source ~/.profile
```

验证

```
adb version
fastboot --version
```

#### 四、创建目录并安装 repo

```
mkdir -p ~/bin ~/android/lineage

echo 'export PATH=$PATH:~/bin' >> ~/.bashrc
source ~/.bashrc
repo version

curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
repo version
```

#### 五、配置 Git

```
git config --global user.name "beiniao"
git config --global user.email "3211138814@qq.com"
git config --global core.fsyncObjectFiles true
git config --global trailer.changeid.key "Change-Id"
git lfs install
```

`core.fsyncObjectFiles true` 会强制 git 写对象时落盘, 牺牲一点速度换取异常断电 / 写入中断时的数据安全, 对虚拟机环境很有必要。

`git lfs install`​ 在非 git 目录下会附带一句 rev-parse 的报错, 只要最后输出 ​`Git LFS initialized.` 就是成功的, 可以忽略。

![](https://bbs.kanxue.com/upload/attach/202610/951483_H5FDKRMKX8ADTJT.png)

#### 六、扩大 Swap 到 32GB

```
sudo swapoff -a
sudo rm -f /swapfile
sudo fallocate -l 32G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
free -h
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_K3MVVF9HDYHW4Q6.png)

#### 七、开启 ccache 加速

```
cat >> ~/.bashrc << 'EOF'
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache
EOF
source ~/.bashrc

ccache -M 50G
ccache -o compression=true
```

#### 八、同步前环境自检

```
df -h ~/android              # 可用空间至少 400G
stat -f -c %T ~/android      # 必须输出 ext2/ext3(即 ext4)
free -h                      # 确认 swap 已生效
git --version                # 确认 2.34+
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_8KBPJTU6J9JEXJ2.png)

#### 九、初始化并同步源码

bluejay(Pixel 6a) 官方支持 lineage-23.2(基于 Android 16):

```
cd ~/android/lineage
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs --no-clone-bundle
repo sync -c --no-tags 2>&1 | tee /tmp/sync.log
```

#### 十、验证同步完整性

三条命令, **全部无输出才算成功**:

```
grep -iE 'fatal|error|failed' /tmp/sync.log
repo status
repo forall -j4 -c 'git fsck --no-progress --connectivity-only 2>&1 | grep -E "^(error|fatal|missing)" >/dev/null && echo "BAD: $REPO_PATH"' 2>/dev/null
```

第一条查同步日志有无失败, 第二条查工作区文件有无缺失或改动, 第三条查每个仓库的 git 对象库是否完整。git 是内容寻址的, fsck 通过就不存在 "内容和远端不一致" 的可能, 只存在 "下漏了" 和 "下坏了", 这三条正好覆盖。

#### 十一、准备设备代码

```
cd ~/android/lineage
source build/envsetup.sh
breakfast bluejay
```

#### 十二、提取厂商 blob

blobs 是闭源驱动和固件, 不在开源仓库里, 必须自己提取。有两种方式。

##### 方式一: 从 OTA 包提取 (推荐)

官方 nightly 是 user 版本,​`adb root` 被禁用, 所以大多数情况下只能走这条路。

下载对应设备的官方包:

```
mkdir -p ~/下载 && cd ~/下载
wget -c "https://mirrorbits.lineageos.org/full/bluejay/20260912/lineage-23.2-20260912-nightly-bluejay-signed.zip"
```

具体版本号去 [](https://bbs.kanxue.com/elink@9d6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6V1L8%4N6F1L8r3!0S2k6q4)9J5k6h3I4A6L8X3g2S2k6$3g2G2M7#2)9J5k6h3!0J5k6#2)9J5c8X3c8W2N6X3W2U0k6i4y4Q4x3V1k6T1L8s2g2W2K9X3q4&6i4K6u0r3j5Y4g2A6L8r3c8K6)https://download.lineageos.org/devices/bluejay/builds 查最新的。

![](https://bbs.kanxue.com/upload/attach/202610/951483_WA7WFSMWZ4TUS8P.png)

提取:

```
cd ~/android/lineage/device/google/bluejay
./extract-files.py ~/下载/lineage-23.2-20260912-nightly-bluejay-signed.zip
```

**方式二: 从已刷机的设备提取**

仅当手机上跑的是对应版本的 LineageOS 且已获取 root 时可用:

```
cd ~/android/lineage/device/google/bluejay
./extract-files.py
```

无论哪种方式, 驱动都会被提取到 ​`~/android/lineage/vendor/google`。完成后确认一下:

```
ls ~/android/lineage/vendor/google/bluejay/
```

#### 十三、内核

Tensor 芯片的 Pixel（6/6a/7/8/9 系列）内核用 Kleaf/Bazel 单独构建。device/google/bluejay-kernels/6.1/ 在 git 里只有几个模块列表文件，.gitignore 排除了所有编译产物，所以编译前必须往这个目录放入内核产物，否则会报：

```
FAILED: ninja: 'device/google/bluejay-kernels/6.1/dtbo.img', needed by 'out/target/product/bluejay/dtbo.img', missing and no known rule to make it
```

LineageOS 已经把内核编译自动化了。​`device/google/bluejay/device-bluejay.mk`​ 里定义了 ​`TARGET_KERNEL_PLATFORM_SOURCE`​，执行 ​`breakfast`​ 时会自动调用 ​`build_kernel`，依次完成下面几件事：

1.  把内核源码 repo sync 到 ​`~/android/lineage/out-kernel/google/gs-6.1`
2.  执行 ​`./build_bluejay.sh`
3.  把 ​`out/bluejay/dist/*`​ 拷到 ​`device/google/bluejay-kernels/6.1/`

所以不需要自己去单独同步、编译内核。下面两种方式选一种就行。

##### 方式一：自动编译（需要网络和磁盘）

直接执行第十一步的 ​`breakfast bluejay`​。内核 manifest 里的仓库来自 ​`android.googlesource.com`，如果配置了镜像，注意下面几点：

```
# 镜像改写（TUNA）
git config --global url.https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/.insteadof https://android.googlesource.com/

# 国内镜像必须直连，不能走代理，否则会报 gnutls_handshake() failed
echo 'export no_proxy="localhost,127.0.0.1,::1,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,.tsinghua.edu.cn,.ustc.edu.cn"' >> ~/.bashrc
echo 'export NO_PROXY="$no_proxy"' >> ~/.bashrc
source ~/.bashrc

# 测试连通
git ls-remote https://android.googlesource.com/kernel/build | head -3
```

#### 同步失败可以进内核目录单独重试。TUNA 有并发限制，​`-j` 不要开太大：

```
cd ~/android/lineage/out-kernel/google/gs-6.1
until repo sync -c --no-tags -j2; do echo "重试..."; sleep 30; done
```

看到 ​`Kernel build output copied to device/google/bluejay-kernels/6.1/` 就说明成功了。

##### 方式二：从官方 OTA 包提取预编译内核（推荐，省磁盘和流量）

第十二步已经下载过 OTA 包，里面有 LineageOS 官方编好的内核，版本和提取的 blobs 完全一致。

**1. 拆出分区镜像**

```
mkdir -p ~/kernel-extract && cd ~/kernel-extract
unzip -o ~/下载/lineage-23.2-20260912-nightly-bluejay-signed.zip payload.bin

# 版本号去 https://github.com/ssut/payload-dumper-go/releases 查最新
wget https://github.com/ssut/payload-dumper-go/releases/download/2.1.0/payload-dumper-go_2.1.0_linux_amd64.tar.gz
tar xf payload-dumper-go_2.1.0_linux_amd64.tar.gz
./payload-dumper-go -p boot,dtbo,vendor_boot,vendor_dlkm -o img payload.bin
```

**2. 解出 dtb 和内核模块**

```
# vendor_boot：dtb 和 ramdisk 里的模块
python3 ~/android/lineage/system/tools/mkbootimg/unpack_bootimg.py --boot_img img/vendor_boot.img --out vb
mkdir -p ramdisk && cd ramdisk
for f in ../vb/vendor_ramdisk*; do lz4 -dc "$f" | cpio -idm 2>/dev/null; done
cd ..

# vendor_dlkm：ext4 格式（file 命令显示 ext2 + extents，其实就是 ext4）
mkdir -p dlkm
debugfs -R "rdump /lib $(pwd)/dlkm" img/vendor_dlkm.img

# 模块数量检查（实测 204 / 318）
find ramdisk -name '*.ko' | wc -l
find dlkm -name '*.ko' | wc -l
```

**3. 放进内核目录并校验**

```
K=~/android/lineage/device/google/bluejay-kernels/6.1
cp img/boot.img img/dtbo.img $K/
cp vb/dtb $K/gs101.dtb
find ramdisk dlkm -name '*.ko' -exec cp -n {} $K/ \;

# 对照模块列表检查，没有输出就是齐的
cd $K
for m in $(cat modules.load vendor_dlkm.modules.load); do
  [ -f "$(basename $m)" ] || echo "缺: $m"
done
```

`BoardConfig-common.mk`​ 会从这个目录读取 ​`boot.img`​、​`dtbo.img`​、​`*.dtb`​ 和 ​`*.ko`，这几类文件都有了就可以编译。

**4. 关闭 lunch 时的自动内核编译**

```
cd ~/android/lineage
sed -i 's/^TARGET_KERNEL_PLATFORM_SOURCE/# TARGET_KERNEL_PLATFORM_SOURCE/' device/google/bluejay/device-bluejay.mk
grep KERNEL_PLATFORM device/google/bluejay/device-bluejay.mk   # 行首应为 #
rm -rf out-kernel ~/kernel-extract
```

#### 十四、开始编译

```
cd ~/android/lineage
source build/envsetup.sh
brunch bluejay 2>&1
```

`brunch` 会自动选择目标并执行全量编译。

查看产物:

```
cd ~/android/lineage/out/target/product/bluejay/
ls -lh *.zip *.img
```

#### 需要先配置 udev 规则

Google 设备的 USB vendor ID 是 18d1

创建 udev 规则并授权：

```
sudo tee /etc/udev/rules.d/51-android.rules > /dev/null << 'EOF'
# Google devices (adb + fastboot)
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", MODE="0666", GROUP="plugdev"
EOF

sudo chmod a+r /etc/udev/rules.d/51-android.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

然后确认在 plugdev 组里

```
groups              # 看看有没有 plugdev
sudo usermod -aG plugdev $USER
```

最后重启 adb 服务（手机确认一下）

```
adb kill-server
adb start-server
adb devices
adb reboot bootloader
```

#### 开刷

```
cd ~/android/lineage/out/target/product/bluejay/
```

先刷 boot 和 dtbo，然后重启回 bootloader

```
fastboot flash boot boot.img
fastboot flash dtbo dtbo.img
fastboot reboot bootloader
```

刷 vendor_boot（这就是 recovery）

```
fastboot flash vendor_boot vendor_boot.img
```

进 recovery

```
fastboot reboot recovery
```

recovery 长这样

![](https://bbs.kanxue.com/upload/attach/202610/951483_9VMG7THCZ5P3Y22.png)

第一步，先格式化数据

![](https://bbs.kanxue.com/upload/attach/202610/951483_79KYQMTWMKMSD9T.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_ZEV5JS6AEWX89PK.png)

确认格式化

![](https://bbs.kanxue.com/upload/attach/202610/951483_FXMHTN84QC69Y66.png)

第二步：在 recovery 菜单里选 Apply update → Apply from ADB

到这一步要先确定 usb 是否能够正常连接到虚拟机，如果不能有两个办法，第一个就是点击下图页面的坐上角返回，然后重新进入。

第二个就是重启虚拟机 or 重启电脑

![](https://bbs.kanxue.com/upload/attach/202610/951483_3T9Q5KAMVPN7ZA5.png)

确认设备链接

```
adb devices
```

刷入

```
cd ~/android/lineage/out/target/product/bluejay
adb -d sideload lineage-23.2-20260925-UNOFFICIAL-bluejay.zip
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_6ECDN3C2MPC7KC5.png)

选 No → 回主菜单 → Reboot system now 先开机再说

![](https://bbs.kanxue.com/upload/attach/202610/951483_DWPPKWZEX5VH3SS.png)

首先需要确定设备的内核版本

APatch 官方只支持 ARM64、Android 内核版本 3.18 - 6.12

```
adb shell
uname -r
zcat /proc/config.gz | grep -w CONFIG_KALLSYMS
```

这个是我的

![](https://bbs.kanxue.com/upload/attach/202610/951483_T548RVN9JP5GEJJ.png)

理想输出是 ​`CONFIG_KALLSYMS=y`

如果这条命令报错说没有 ​`/proc/config.gz`​，说明内核没开 ​`CONFIG_IKCONFIG_PROC`

装 APatch Manager

直接从官方的[](https://bbs.kanxue.com/elink@b4aK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6T1L8h3q4^5x3e0t1I4i4K6u0r3b7g2m8S2N6r3y4Z5i4K6u0r3M7X3g2D9k6h3q4K6k6i4x3%60.) github 上去下载就行了，下载好后安装到手机上。

准备 boot.img，这里直接用之前编译出来的 boot.img 就行，把它传到手机上

```
adb push ~/android/lineage/out/target/product/bluejay/boot.img /sdcard/Download/
```

在 APatch app 里 patch

打开 APatch，然后点击右上角的 patch 图标，选择 / sdcard/Download/boot.img

设置一个 SuperKey

123456abc

patch 完会生成一个新 img

把 patch 好的 img 从手机拉回电脑

```
adb pull /sdcard/Download/apatch.img ~/
adb reboot bootloader
fastboot flash boot ~/apatch.img
fastboot reboot
```

一定要记得点一下这个安卓系统补丁！！！

![](https://bbs.kanxue.com/upload/attach/202610/951483_XYCGXMJU2WBJQ6D.png)

六条，缺一不可

##### 设备已 root，且能拿到 uid=0

```
id
```

输出里有 ​`uid=0(root)`

##### 内核开了 BPF 核心支持

```
zcat /proc/config.gz | grep -wE "CONFIG_BPF|CONFIG_BPF_SYSCALL|CONFIG_BPF_JIT|CONFIG_BPF_EVENTS|CONFIG_PERF_EVENTS"
```

这五项**全部等于** ​**​`y`​**​。**不满足**：需要重编内核开启对应选项。

CONFIG_BPF=y  
CONFIG_BPF_SYSCALL=y  
CONFIG_BPF_JIT=y  
CONFIG_BPF_EVENTS=y  
CONFIG_PERF_EVENTS=y

##### BTF 存在

```
zcat /proc/config.gz | grep -w CONFIG_DEBUG_INFO_BTF
ls -l /sys/kernel/btf/vmlinux
```

`CONFIG_DEBUG_INFO_BTF=y`​ **并且** ​`/sys/kernel/btf/vmlinux`​ 文件真实存在（通常几 MB）。**两个都要满足**——配置开了但文件不在也不行。**不满足**：如果只能重编内核，务必加上 ​`CONFIG_DEBUG_INFO_BTF=y`

##### raw_syscalls 采集点存在

```
ls /sys/kernel/tracing/events/raw_syscalls/ 2>/dev/null || ls /sys/kernel/debug/tracing/events/raw_syscalls/
```

输出里能看到 ​`sys_enter`​ 和 ​`sys_exit`​。**不满足**：正常内核都有。若没有，检查 tracefs 是否挂载。

##### 能读用户态内存的 helper 存在

```
cat /proc/kallsyms | grep -w bpf_probe_read_user
```

能 grep 到 ​`bpf_probe_read_user`​ 这个符号（地址显示为全 0 是权限问题，不影响判断，**有符号名即可**）

**不满足**：内核太旧，需要 backport 或换设备。

##### SELinux 可放开（开发阶段）

```
getenforce
setenforce 0
getenforce
```

第一次可能是 ​`Enforcing`​，执行 ​`setenforce 0`​ 后第三次能变成 ​`Permissive`

**不满足**：如果 ​`setenforce 0` 无效（有些设备锁死），就得处理 SELinux policy 或换到合适的域运行，会麻烦不少。

1.  装工具链

```
sudo apt update
sudo apt install -y clang-15 llvm-15 pkg-config build-essential git \
    gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu

sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-15 100
sudo update-alternatives --install /usr/bin/llvm-strip llvm-strip /usr/bin/llvm-strip-15 100
clang --version | head -1
```

2.  装 arm64 交叉库

```
sudo dpkg --add-architecture arm64
sudo sed -i 's|deb http|deb [arch=amd64] http|g' /etc/apt/sources.list

echo "deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports jammy main universe
deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports jammy-updates main universe" | sudo tee /etc/apt/sources.list.d/arm64-ports.list

sudo apt update
sudo apt install -y libelf-dev:arm64 zlib1g-dev:arm64
```

3.  拉脚手架并编 bpftool

```
cd ~
git clone --recurse-submodules https://github.com/libbpf/libbpf-bootstrap
cd ~/libbpf-bootstrap/examples/c
make minimal
```

编完记下 bpftool 路径（后面 Makefile 要用）：

```
find ~/libbpf-bootstrap -name "bpftool" -type f 2>/dev/null
```

/home/aosp/libbpf-bootstrap/examples/c/.output/bpftool/bootstrap/bpftool

![](https://bbs.kanxue.com/upload/attach/202610/951483_4XFK2922FWBXXK7.png)

4.  交叉编译 arm64 版 libbpf.a

```
mkdir -p ~/syscall-tracer/libbpf-arm64
cd ~/libbpf-bootstrap/libbpf/src
CC=aarch64-linux-gnu-gcc BUILD_STATIC_ONLY=y \
  OBJDIR=/tmp/libbpf-arm64-obj \
  DESTDIR=~/syscall-tracer/libbpf-arm64 \
  make install

find ~/syscall-tracer/libbpf-arm64 -name "libbpf.a"
```

记下 ​`libbpf.a`​ 的实际路径（可能在 ​`lib64/`​ 也可能在 ​`lib/`），Makefile 里要填对。

/home/aosp/syscall-tracer/libbpf-arm64/usr/lib64/libbpf.a

![](https://bbs.kanxue.com/upload/attach/202610/951483_DVK6PQQ3Z3GG7YF.png)

5.  建项目目录

```
mkdir -p ~/syscall-tracer
cd ~/syscall-tracer
```

从手机拉 BTF 并生成 vmlinux.h（用第 3 步 find 出来的 bpftool 路径）：

```
adb shell "cat /sys/kernel/btf/vmlinux > /data/local/tmp/vmlinux.btf && chmod 644 /data/local/tmp/vmlinux.btf"
adb pull /data/local/tmp/vmlinux.btf ~/vmlinux.btf.phone
ls -l ~/vmlinux.btf.phone
```

`ls -l` 看到几 MB 的大小就成了。

然后生成 vmlinux.h:

```
/home/aosp/libbpf-bootstrap/examples/c/.output/bpftool/bootstrap/bpftool \
  btf dump file ~/vmlinux.btf.phone format c > ~/syscall-tracer/vmlinux.h

wc -l ~/syscall-tracer/vmlinux.h
grep -c "struct trace_event_raw_sys_enter" ~/syscall-tracer/vmlinux.h
```

行数几万行、​`grep` 结果是 1, 就说明 vmlinux.h 正常。

![](https://bbs.kanxue.com/upload/attach/202610/951483_RD783PFERH9G5DZ.png)

6. 安装 vscode

一行搞定

```
sudo snap install code --classic
```

打开项目

```
cd ~/syscall-tracer
code .
```

7. 新建文件

然后新建以下文件

**文件一:** ​**​`tracer.bpf.c`​**(跑在内核里的 BPF 程序, 就抓 openat)

```
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} rb SEC(".maps");

struct event {
    int pid;
    char comm[16];
};

SEC("tp/raw_syscalls/sys_enter")
int handle_sys_enter(struct trace_event_raw_sys_enter *ctx)
{
    if (ctx->id != 56)   // 56 = openat (arm64)
        return 0;

    struct event *e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e)
        return 0;

    e->pid = bpf_get_current_pid_tgid() >> 32;
    bpf_get_current_comm(&e->comm, sizeof(e->comm));
    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

**文件二:** ​**​`tracer.c`​**(跑在手机上的加载器, 把抓到的打印出来)

```
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include "tracer.skel.h"

struct event {
    int pid;
    char comm[16];
};

static volatile int exiting = 0;
static void on_sig(int s) { exiting = 1; }

static int on_event(void *ctx, void *data, size_t sz)
{
    struct event *e = data;
    printf("openat  pid=%-6d  comm=%s\n", e->pid, e->comm);
    return 0;
}

int main(void)
{
    struct tracer_bpf *skel;
    struct ring_buffer *rb;

    signal(SIGINT, on_sig);

    skel = tracer_bpf__open_and_load();
    if (!skel) { printf("load 失败\n"); return 1; }

    if (tracer_bpf__attach(skel)) { printf("attach 失败\n"); return 1; }

    rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), on_event, NULL, NULL);

    printf("开始抓 openat... Ctrl+C 退出\n");
    while (!exiting)
        ring_buffer__poll(rb, 100);

    ring_buffer__free(rb);
    tracer_bpf__destroy(skel);
    return 0;
}
```

**文件三:** ​**​`Makefile`​**(注意: Makefile 里缩进必须是 Tab, 不能是空格, VSCode 里粘完检查一下)

```
CLANG    := <你装的 clang 版本，比如 clang-15>
BPFTOOL  := <第3步 find 出来的 bpftool 完整路径>
CROSS_CC := aarch64-linux-gnu-gcc
LIBBPF   := $(HOME)/syscall-tracer/libbpf-arm64/usr

all: tracer

tracer.bpf.o: tracer.bpf.c vmlinux.h
	$(CLANG) -g -O2 -target bpf -D__TARGET_ARCH_arm64 -I$(LIBBPF)/include -c tracer.bpf.c -o tracer.bpf.o

tracer.skel.h: tracer.bpf.o
	$(BPFTOOL) gen skeleton tracer.bpf.o > tracer.skel.h

tracer: tracer.c tracer.skel.h
	$(CROSS_CC) -g -O2 -I. -I$(LIBBPF)/include tracer.c $(LIBBPF)/<lib64 或 lib>/libbpf.a -lelf -lz -static -o tracer

clean:
	rm -f tracer tracer.bpf.o tracer.skel.h
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_9UUFBJM55T3E2QX.png)

8. 开始编译

完事之后就直接开始编译

编译之前还需要补一下这两个东西

```
sudo apt install -y gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
sudo apt install -y gcc-multilib
```

现在在 VSCode 里打开终端, 然后:

```
cd ~/syscall-tracer
find libbpf-arm64 -name libbpf.a
```

先看这条输出的路径末尾是 ​`lib64`​ 还是 ​`lib`​。确认后 (如果是 ​`lib`​ 就去 Makefile 把 ​`lib64` 改掉), 直接编:

![](https://bbs.kanxue.com/upload/attach/202610/951483_P4JE9J54VYKG8QC.png)

是 ​`lib64`​, 那 Makefile 最后那行的 ​`lib64` 不用改, 保持原样。

```
make
```

编译完了，没啥毛病

![](https://bbs.kanxue.com/upload/attach/202610/951483_KKKBKJB4VZ6TGYG.png)

9. 刷入测试

```
adb push tracer /data/local/tmp/
adb shell chmod 755 /data/local/tmp/tracer
```

运行

```
adb shell /data/local/tmp/tracer
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_B9YE5E7BCF8A9ZH.png)

前面只是快速的搭建好了开发平台，并且成功的编译刷入做了测试，但是对其的了解还是知之甚少，下面就对 ebpf 来做扫盲和刚刚的程序解析学习。

#### 内核态和用户态

计算机运行的程序分为两个世界。

用户态：开发者平时写的所有程序，比如说微信、QQ、浏览器，包括前面写的 tracer.c 都跑在这里。用户态权限受限，不能直接碰硬件，不能直接读取别的进程的内存，要想做这些事情就必须要依赖内核。

内核态：操作系统内核本身运行的地方。管理内存、进程、硬件、文件系统，权限最高。

这两个世界是隔离的，中间的桥梁就是系统调用（system call，简称 syscall）。当程序要打开一个文件，他自己没有权限直接操作磁盘，于是他会发起一个叫 openat 的系统调用，把请求交给内核，内核在内核态完成时机操作，再将结果返回。

#### ebpf 是什么

ebpf 是一项能够在内核里运行沙箱程序、而无需修改内核源码或加载内核模块的技术。写一小段代码，它可以被安全的加载进正在运行的内核并挂到某个事件上，事件发生时这段代码就执行。

往内核里塞代码本身是一件风险较高的事情，一旦有 bug（死循环、越界访问内存）整个系统会崩。ebpf 靠验证器（verifier）保证安全。

官方文档明确说明，程序在加载进内核前，验证器会对他做安全检查，以此来确保他会正常结束（不会死循环）、没有越界内存访问等等，任何一项不满足就拒绝加载。这就是为什么 BPF 程序有诸多限制（不能用任意循环、栈空间很小、不能调用任意内核函数），这些限制是为了让验证器能够证明程序安全。通过验证后，程序还会被 JIT 编译成机器码以提升运行速度。

#### 三个核心构件

一个 ebpf 项目由两个程序加一种共享存储组成。

内核程序（tracer.bpf.c）编译成 BPF 字节码后加载进内核，挂在钩子上运行，负责在事件现场采集信息。

用户态程序（tracer.c）普通程序，负责把内核态程序加载进内核、挂载到钩子上、并循环将采集到的数据取回处理。

map：官方定义中，map 是 “存在于内核中的数据结构，ebpf 程序和用户态程序都能访问，因此是两者之间的通信层”。

前面用到的是其中一种类型叫做 ring buffer（环形缓冲区，BPF_MAP_TYPE_RINGBUF），专门用于内核态单项、按顺序的把一条条消息发给用户态。

#### 辅助函数是什么

官方定义：BPF 程序不能调用任意内核函数，否则会破坏内核稳定、也会让 BPF 程序绑死在特定内核版本上。作为替代，内核开放了一批由内核定义、可从 ebpf 程序调用的函数，这就是辅助函数，他们构成一套知名且稳定的 API。在内核态代码里能做的所有事，基本都通过调用这些 bpf_开头的辅助函数完成。

#### 逐行拆解内核态程序 tracer.bpf.c

```
#include "vmlinux.h"
```

从手机 BTF 导出的头文件，包含这台手机内核的所有数据结构定义。BPF 代码要读取内核数据结构就必须知道他们的布局，而不同内核版本布局可能不同，所以针对目标手机生成专属的 vmlinux.h

```
#include <bpf/bpf_helpers.h>
```

提供辅助函数的声明，以及 SEC 等宏的定义。

```
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

BPF 程序必须声明许可证，因为许多辅助函数只允许 GPL 兼容的程序调用，不声明会导致加载失败。SEC("license") 里的 SEC 是宏，作用是把这个变量放进编译产物中名为 license 的节区。一个编译好的文件内部分成若干块，每块叫一个 section；ebpf 用 section 的名字来标识每块内容的用途，加载器和内核靠 section 名来识别这是许可证、这是 map、这是要挂载的程序，所以后面出现的每个 SEC（...）都是在给对应的东西贴用途标签

```
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} rb SEC(".maps");
```

定义那个 ring buffer，变量名 rb.SEC(".maps") 标明它是一个 map。type 指定类型为 ring buffer。对 ring buffer 而言，max_entries 表示缓冲区的字节大小，这里是 256×1025= 256KB。内核为此缓冲区分配 256KB，采集的数据先堆在这里等用户态来取。官方要求 ring buffer 的大小必须是 4KB 的整数倍且为 2 的幂，256KB 满足这个条件。

```
struct event {
    int pid;
    char comm[16];
};
```

自定义的一条记录格式，pid 和进程名 comm。

极其重要：用户态 tracer.c 里也定义了字段完全相同的 struct event。因为内核态往缓冲区写入的是这块结构体的原始字节，用户态读出来后按自己的结构定义区解读，变量只要有任何差异（字段顺序、类型、大小）解读就会出错。所以两端必须严格一致

```
SEC("tp/raw_syscalls/sys_enter")
int handle_sys_enter(struct trace_event_raw_sys_enter *ctx)
```

tp 是 tracepoint 的缩写。tracepoint 是内核开发者预先埋在内核代码里的固定钩子，分布在内核各关键位置。raw_syscalls/sys_enter 这个特定 tracepoint 会在任何系统调用即将开始时触发。所以这行的意思是把 handle_sys_enter 挂到这个 tracepoint 上，挂上去后，系统里任何进程发起任何系统调用，内核都会先执行一次这个函数。参数 ctx 由内核在触发时传入，携带这次系统调用的信息，其类型 struct trace_event_raw_sys_enter 定义在 vmlinux.h 内。

```
if (ctx->id != 56)   // 56 = openat (arm64)
        return 0;
```

sys_enter 会捕获所有系统调用，数量极大，所以过滤。ctx->id 是这次系统调用的编号，arm64 上 openat 编号是 56，

有一个特别的点 就是这里的系统调用编号是跟架构相关联的，arm64 上 openat 是 56，x86_64 上则是另外一个号。

```
struct event *e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e)
        return 0;
```

bpf_ringbuf_reserve 是辅助函数，在 ring buffer 里预留一块空间。参数依次是：往那个 ring buffer 预留 (&rb)、预留多大 (sizeof(*e)，刚好一个 event)、标志位（填 0）。返回指向预留空间的指针 e。若缓冲区已满（用户态取得太慢导致堆积）则预留失败返回 NULL。这里就必须要对 e 做检查，这是验证其强制要求的。

ring buffer 的使用套路固定为三步: reserve(预留)→ 填数据 → submit(提交)。

```
e->pid = bpf_get_current_pid_tgid() >> 32;
```

bpf_get_current_pid_tgid() 返回一个 64 位的整数，官方定义是 tgid << 32 | pid ，也就是高 32 位是 TGID，低 32 位是 PID。关键在于内核里 PID 和 TGID 这两个词的含义跟日常习惯不同，内核里 TGID（线程组 ID）才是平时说的进程 ID，内核里的 PID 其实指的是单个线程的 ID。所以这行>> 32 取出的是高 32 位，即 TGID，也就是日常意义上的进程号。变量名叫 pid 只是沿用习惯叫法，他存的实际是 TGID（进程号）。如果以后想要区分具体哪个是线程，就取低 32 位（内核意义上的 PID）

```
bpf_get_current_comm(&e->comm, sizeof(e->comm));
```

辅助函数，把当前进程的名字拷贝到指定位置。参数为目标地址（&e->comm）和最多拷贝的字节数（sizeof（e->comm），即 16）

```
bpf_ringbuf_submit(e, 0);
    return 0;
}
```

bpf_ringbuf_submit 提交这块已填好的数据，提交后这条记录正式进入 ring buffer，用户态即可读到。第二个参数是标志位，填 0。最后 return 0 表示函数正常结束。

内核这边做的事情一个总结就是：每次系统发生调用 》判断是不是 openat 》 是则在缓冲区预留空间 》 填入进程号和进程名 》 提交

#### 逐行讲用户态程序 tracer.c

```
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <bpf/libbpf.h>
```

前面三个是标准 c 库头文件（打印、信号处理等）。libbpf.h 是 libbpf 的头文件，libbpf 是官方提供的、专门操作 ebpf 的 c 库，把加载程序、管理 map、读 ring buffer 这些底层操作都封装好了。

```
#include "tracer.skel.h"
```

骨架文件，不是手写的，是编译时由 bpftool 从编译好的内核态目标文件 tracer.bpf.o 自动生成。它把加载这个特定 BPF 程序、找到其中的 map、挂载它等操作包装成以 tracer_bpf__开头的函数，省去手写底层加载代码。

```
struct event {
    int pid;
    char comm[16];
};
```

跟内核态那个完全一致，用于正确解读从缓冲区读回的字节。

```
static volatile int exiting = 0;
static void on_sig(int s) { exiting = 1; }
```

用于 Ctrl+C 的退出。exiting 是标志变量。初始 0。on_sig 是信号处理函数，按 ctrl+c 会产生中断信号，系统调用它把 exiting 设为 1，主循环看到后就退出。volatile 告诉编译器这个变量可能被随时改变，不要做优化缓存。

```
static int on_event(void *ctx, void *data, size_t sz)
{
    struct event *e = data;
    printf("openat  pid=%-6d  comm=%s\n", e->pid, e->comm);
    return 0;
}
```

回调函数，处理数据的地方。所谓回调，是把这个函数交给 libbpf，之后每当 ring buffer 里有一条新数据，libbpf 就自动调用一次它。参数 data 指向那条数据（内核态提交的字节），struct event *e = data 把它按 struct event 解读，于是能取出 e->pid、e->comm。%-6d 表示打印整数左对齐占 6 格。以后过滤、统计、写文件的等逻辑都加载这个函数里。

```
int main(void)
{
    struct tracer_bpf *skel;
    struct ring_buffer *rb;

    signal(SIGINT, on_sig);
```

main 是程序入口。skel 是骨架对象指针，rb 是 ring buffer 读取器指针。signal（SIGINT，on_sig）注册：收到 SIGINT 信号（ctrl+c 产生）时调用 on_sig。

```
skel = tracer_bpf__open_and_load();
    if (!skel) { printf("load 失败\n"); return 1; }
```

骨架函数，一步完成打开 BPF 对象并把内核态程序加载进内核。就在加载这一步，前面讲的验证器会检查程序安全性，不通过则加载失败。

```
rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), on_event, NULL, NULL);
```

创建 ring buffer 读取端。skel->maps.rb 是骨架关联好的、内核里那个名叫 rb 的 map;bpf_map__fd（...）取得他的文件描述符；on_event 是回调函数，告诉 libbpf 每读到一条数据就调用它，后面两个是 NULL 是可选参数了，这里用不着。

```
printf("开始抓 openat... Ctrl+C 退出\n");
    while (!exiting)
        ring_buffer__poll(rb, 100);
```

打印提示后进入主循环。while（！exiting）表示只要 exiting 为 0 就一直循环。ring_buffer__poll(rb，100) 去缓冲区查看有无新数据，参数 100 是超时毫秒数，最多等 100 毫秒，期间有数据就处理（自动触发 on_event），没数据就返回再进入下一轮。用 "轮询 + 超时" 是为了让循环能定期检查 exiting，从而在 ctrl+c 时能够及时退出。

```
ring_buffer__free(rb);
    tracer_bpf__destroy(skel);
    return 0;
}
```

退出循环后清理：释放 ring buffer 读取器、销毁骨架对象（会自动把 BPF 程序从内核卸载）。return 0 表示正常结束了。

#### 整条链路

编译阶段 clang 把 tracer.bpf.c 编译成 BPF 字节码 tracer.bpf.o，bpftool 从它生成骨架 tracer.skel.h 交叉编译器把 include 了骨架的 tracer.c 编译成手机 arm64 可执行文件 tracer。

运行阶段：在手机运行 tracer》骨架函数把内核态程序加载进内核（验证器检查通过）》挂载到 sys_enter reacepoint 》进入循环读 ring buffer。此后每发生一次系统调用，内核就执行一次 handle_sys_enter，它过滤处 openat, 把进程号和进程名写进 ring buffer；用户态读到后触发 on_event 打印。ctrl+c 后清理资源退出，内核程序被卸载。

#### 引言

目前论坛当中比较安利的就是这两篇帖子，讲的很好，推荐看看。

[《](https://bbs.kanxue.com/thread-271043-1.htm)​**[Linux 内核监控在 Android 攻防中的应用](https://bbs.kanxue.com/thread-271043-1.htm) **​[》](https://bbs.kanxue.com/thread-271043-1.htm)

[《](https://bbs.kanxue.com/thread-274546.htm)​** [定制 bcc/ebpf 在 android 平台上实现基于 dwarf 的用户态栈回溯](https://bbs.kanxue.com/thread-274546.htm) **​[》](https://bbs.kanxue.com/thread-274546.htm)

下面只是根据程序的设计思路来进行部分讲解，提供编译好的成品。

先明确一下目标：对某个指定的 app 将他执行的系统调用全部记录下来，并且能够知道每次调用是来自哪个 so。

面对反调试的项目有两个比较难的点，首先就是 app 启动后检测到环境异常或者处于被调试状态会很快的就闪退，常规的做法就是进程活着的时候去读取他的 / proc/pid/maps 来做地址归属，但是这个进程根本不会活到去读。

另外一个就是进程启动早期身份不明确。app 进程是 zygote fork 出来的。fork 出来的那一刻他的 uid 还是 0，要过一会才会被设置成目标 uid。这中间有一段盲窗，光看 uid 是分辨不出来是不是目标 app 的。

#### 难点处理（自己维护一份影子地址空间 --- 影子表）

既然不能指望进程活着的时候去读取他的 maps，那就自己积累一份这个进程的内存里哪段地址属于哪个 so 的表，而且这个表不依赖进程还活着。

表数据来源有三个：

一个是 zygote。程序启动的时候读取一次 zygote 的 maps 存下来，只留有文件名的段（匿名段不在种子里）。因为所有 app 都是 fork zygote 来的，继承了同一批地址布局，所以这就是每个新生 app 的出生时内存布局，进程一秒没活也能用。

二是增量更新。进程运行中会调 mmap/munmap/mprotect 来增删改内存映射，这些调用的参数本身就说明了哪段地址发生了什么变化，抓到这些 syscall 就能更加精确更新影子表，其中 mmap 只有 MAP_FIXED 能在入口建段，非 FIXED 的段靠 kretprobe 取返回值才建得起来。

第三就是刷新，如果说进程还活着，那就读取一次真的 maps，然后做整表覆盖，让影子表更加准确可靠。

这三个来源整合之后，哪怕进程死了，这份表依旧是他最后一刻的内存布局。归属照样成立；但 exec 过的新进程（崩溃转储进程）不适用，它的地址空间和 zygote 无关，只能在它活着时读到 maps

#### 整体数据流

内核侧，目标进程调了一个 openat，kprobe 触发，BPF 抓取寄存器、一小段栈、路径参数，判断这个进程是不是目标，如果是的话就把抓取到的信息写入 ringbuf。

用户态侧，一个读取线程从 ringbuf 把记录读出来，根据进程身份是否确认，决定直接入队处理还是先扣进盲窗暂存，盲窗记录被扣住直到该进程露出目标 uid；退出标记或超时才按出身放行或丢弃——所以端到端滞后几秒是正常的，要看的是排队滞后。入队后由一组 worker 线程去查影子表定位地址属于哪个 so，必要的时候展栈还原调用链，解码 syscall 参数 （把数字翻译成可读的标志位，把 fd 按事件自己的时间戳回查 fd 时间线翻译成路径，不能用现在的表：读表的 worker 滞后于消费线程，用 "现在" 会把早期 mmap 命名成后来才打开的文件），最后格式化成一行输出。

#### 两侧为什么是两个文件

其实我做了好几个版本，尝试过交叉编译成一个文件，但是。。。失败了，后边想着不去依赖 BPF 去做过滤，但是但是但是，整个系统哪怕你不去触碰手机他一秒钟都有大十几万的 syscall 数据量过于庞大，导致缓冲区根本接不下，丢包特别严重（通过一系列优化收紧入口其实后面也能用，就是很勉强，丢包概率有百分之四十，很多时候都得重启好几次才能抓到），后面受不了了，重新整一个，也就是现在这个（挺好用的其实）。

内核 BPF 代码能够做的事情很有限，最主要是验证器很严格、并且不能用复杂数据结构，也不能去做 DWARF 展栈，所以说这里内核侧只需要去抓信息，所有重活全部交给用户态去做。

内核侧要抓的东西就是寄存器 + 一段原始栈，为什么不直接去抓调用链？因为展栈要去读取 ELF 文件，要做比较复杂的计算。所以说内核只需要负责把展栈的原料准备好（寄存器 pc/sp、栈内容）交给用户态慢慢去计算。

#### ringbuf 上的记录

混着三种尺寸、四种语义的记录：退出标记走的是大记录（sizeof(stracer_ev)），只是不填栈和路径。

完整事件（stracer_ev，大）用于需要现场快照的普通 syscall，带寄存器、栈、路径。紧凑记录（stracer_x，小）用于 openat 的返回，只需要带回一个 fd 号。mmap 返回记录（stracer_mmapret）用于带回 mmap 实际映射到的地址。

后两种为什么要小？因为 openat 调用极其频繁，如果每条都用大记录，环会被压垮。它们怎么区分？三种记录的前面一段字节布局故意做成一样，里面有个 sysnr 字段，用户态先读这个字段，再决定按哪种格式解析剩下的部分。

#### 影子表结构

影子表是按照地址区间存的一串段，每段记录了起止地址、这段属于哪个文件（匿名段就没有文件名）、以及权限（可读可写可执行）。查一个地址属于哪个 so，就是拿这个地址去这串区间里找，落在哪一段就属于哪一段，因为区间之间互不重叠、按段首排好了序，查起来很快。增量更新就是拿 mmap/munmap/mprotect 的参数去改这串区间，mmap 是在某段地址上建一段新的，munmap 是把某段删掉，mprotect 是把某段的权限改掉。这三种操作都可能和已有的段部分重叠，所以更新时要把整段按新区间的边界切开，重叠的部分替换掉，两侧没碰到的部分保留下来。刷新则是直接拿读到的真 maps 整表重建。

#### 归属于展栈

要想要知道调用来自哪个 so，分为两步。先要把寄存器里的当前地址拿去查影子表，落在哪个段就归属到哪个 so，这一步进程死了也成立，因为查的是影子表不是活进程。接着就是展栈，也就是还原出完整的调用链，光知道当前这一层还不够，往往需要回推几层才能知道到底是哪个 so 发起的。展栈有三级回退，一级比一级糙。第一级是 DWARF 展栈，靠 so 文件里的调试信息精确回推，最准。如果说拿不到就退到第二级帧指针链，靠寄存器里约定好的帧指针一层层往回跳，再不行就第三级栈回扫，直接扫那段栈，把落在可执行段里的地址当作候选的返回地址，这一级是猜的、可能误报，所以默认只当参考、不拿他作判定。

#### 使用文档

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">选项</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">含义</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>采什么</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--sc a,b,c</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">只采点名的，只认真名、没有分组。没点名的完全不采；不给 = 全部（这台机器实际 29 个，​<code>fstat</code> 挂不上）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>谁发起的</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-s &lt;so&gt;</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">子串匹配发起方。默认只显示命中行；第一次解析到该 so 时打一行 ​<code>[基址]</code>​；<strong>会主动读</strong> ​ <strong>​<code>/proc</code>​</strong>​ <strong>去学这块 so</strong>（两种证据绕开 2s TTL：查不到的地址、表里从没见过该 so；100ms→1s 自动退避，12 次预算）；<strong>命中可能是 "当场补判" 出来的</strong>（见下）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--all</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">给了 ​<code>-s</code> 也显示未命中行（未命中会打归属失败的原因）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--only</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">显式要求只显示命中（已是默认，必须配 ​<code>-s</code>，否则报错退出）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--heuristic-match</code>​ / ​<code>--fp-match</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">让栈内回扫候选 / ​<code>[fp]</code>​ 帧参与命中判定（默认不算：会误报）。回扫命中打 ​<code>[命中(回扫) x]</code>​；候选列表只在 ​<code>--frames</code> 下打印</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>哪些进程</strong>（不显示的仍采集、仍计数）</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--children</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">显示目标子孙进程的行（含崩溃转储进程）。默认不显示。<strong>跑不漏对表必须加它</strong>，否则裁判会误报 "少采"（少的是打印，不是采集）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--foreign</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">显示同 uid 但没看见其出生的进程。默认不显示</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>详略（一条轴）</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">（默认）</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">只打 syscall 行：归属退化成 ​<code>at=0x38ee4 caller=0x2edf0</code>​（纯模块内偏移，无 so 名、无 ​<code>[命中]</code>​、<strong>无补判标记</strong>）；启动行 /​<code>进程退出</code>​/ 每进程清单 / 收尾统计全不打。唯一例外：真丢事件时一行 ​<code>[W]</code></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-v</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">加回启动行 + ​<code>attach 完成</code>​ + ​<code>进程退出</code>​ + 每进程清单 + 收尾五块统计 + ​<code>[补判:实时判定早于影子表]</code>​ 标记与 ​<code>[补判] …重判 N 条翻盘 K 条</code></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-vv</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">再加每 2 秒 ​<code>[STAT]</code> 内核计数与队列状态</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-q</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">只挡<strong>事件行</strong>；​<code>[W]/[E]</code>​ 与 ​<code>-v</code>​ 的统计仍走屏幕（它们不落 ​<code>-o</code>）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>生命周期</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--launch</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">先 force-stop 再冷启动。唯一有副作用的开关，所以不默认。启动动作跑在独立线程，不挡收环</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--attach</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">只挂已经在跑的进程，不杀（默认）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--no-until-exit</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标全退出后继续跑（默认退出即收摊）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-t SEC</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">到时自动停止。<strong>默认不限时</strong>（跑到 Ctrl-C；安全，照常 detach 并恢复 ​<code>kptr_restrict</code>）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>展栈与读内存</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--frames</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">打印完整调用栈（并显示 ​<code>at=</code>​ 与 ​<code>cand=</code>）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--no-unwind</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不展栈，只做 pc/lr 归属（吞吐最高）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--unwind-children</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">连子孙进程也认真展栈（默认不：不含目标代码，逐条 DWARF 会拖慢消费端）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--no-strings</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不读字符串 / fd 路径（少访问 ​<code>/proc/&lt;pid&gt;/mem</code>）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--depth N</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">每事件拷的用户栈字节 0..1024，按 512 取整；队列堆积时自动降到 0，只丢展栈不丢事件</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--full-path</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">mmap 的后备文件也打完整路径（​<code>openat</code> 参数与 fd 名从不缩略）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>输出与对象</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-o FILE</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">另写文件，只写事件行、统计不进文件。路径要绝对且在 ​<code>/data/local/tmp/</code>​ 下——这是 ​<code>adb shell</code>​ 的 cwd 是 ​<code>/</code> 造成的 OS 限制，工具本身不校验</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>-w N</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">展栈线程数（默认按 CPU 核数）</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>--obj PATH</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">BPF 对象路径，默认 ​<code>/data/local/tmp/stracer.bpf</code></td></tr></tbody></table>

支持的 30 个 syscall

`openat close read write pread64 pwrite64 faccessat readlinkat newfstatat fstat lseek getdents64 statx mmap munmap mprotect madvise mremap clone execve prctl futex ioctl fcntl socket connect sendto recvfrom getrandom rt_sigprocmask`

#### 效果展示

这里拿两个企业加固做展示

一个就是我上篇帖的某 bang 企业版的 so 内存解密

![](https://bbs.kanxue.com/upload/attach/202610/951483_FEWMUNYEXDGRZWA.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_X23J3ABRD2C6NPD.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_G8MJ2D7QU2M929R.png)

在一个是某数字加固企业版的解密 ELF 头

![](https://bbs.kanxue.com/upload/attach/202610/951483_Q66HTT39GYHMTNP.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_UDJY8KUW97AGZT5.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_6M2YA9W4UFQG98F.png)

效果还是蛮好的，可以快速知道当前 app 的某个 so 用了哪些系统调用

之前我是在 aosp12 上去对 [Fartext](https://bbs.kanxue.com/thread-268760.htm) 进行移植（这次移植挺轻松的 变化没很大），这次打算在 LineageOS23 上做移植，

本来打算自己啃的（啃过 坑真多），但是有大佬已经跑通了，那就照抄就行。下面是大佬 r0dump 的原文及 github

原文：[[原创] 使用 Kimi K3 进行脱壳工具迁移开发：R0DUMP —— 将 FART 迁移到 Android 16](https://bbs.kanxue.com/thread-292107.htm)

github：[](https://bbs.kanxue.com/elink@8b3K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6@1K9i4N6W2x3q4)9J5c8Y4t1H3k6s2g2E0M7l9%60.%60.)https://github.com/tiwe0/r0dump

下载 Android Studio for Platform

```
cd ~/下载
wget https://dl.google.com/android/asfp/asfp-Panda%202-2025.3.2.6-linux.deb
```

下载好了就解压一下

```
sudo dpkg -i ~/下载/"asfp-Panda 2-2025.3.2.6-linux.deb"
```

启动 ASfP

```
/opt/android-studio-for-platform/bin/studio.sh
```

next

![](https://bbs.kanxue.com/upload/attach/202610/951483_XJPG3E5HXDCM7EN.png)

等他慢慢下

![](https://bbs.kanxue.com/upload/attach/202610/951483_K7HNZ98JEHXTZQP.png)

创建项目之前需要先知道 lunch target 是什么 去跑一下编译，等他跑完

```
cd ~/android/lineage
source build/envsetup.sh
breakfast bluejay
```

编译完成后打印这两个

```
echo "$TARGET_PRODUCT-$TARGET_BUILD_VARIANT"
```

将打印出来的结果拼接一下得到

lineage_bluejay-userdebug

然后再拼接上 BUILD_ID 得到

lineage_bluejay-bp4a-userdebug

这个就是 Lunch target 后面用的着

![](https://bbs.kanxue.com/upload/attach/202610/951483_ECTBDK4PASX6T96.png)

设置一下路径

![](https://bbs.kanxue.com/upload/attach/202610/951483_G4Y8A6B76K6ZSZD.png)

修改一下 lunch target（图片中是错的，改成前面得到的那个即可）lineage_bluejay-bp4a-userdebug，然后直接下一步

![](https://bbs.kanxue.com/upload/attach/202610/951483_2DGHX7X8X38KWPQ.png)

不需要填写直接完成

![](https://bbs.kanxue.com/upload/attach/202610/951483_YG44P5D4X7NGCVF.png)

然后需要配置一下网络

![](https://bbs.kanxue.com/upload/attach/202610/951483_YZQ37FK6AVKMT8A.png)

配置完毕就可以 Close Project 了

找一下文件路径

```
find ~/AsfpProjects -name ".asfp-project" 2>/dev/null
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_US36ANJRYBTGJ5M.png)

用记事本打开他编辑一下

```
gedit /home/aosp/AsfpProjects/lineage/.asfp-project
```

这边你们应该是跟我是一样的 都是空的

![](https://bbs.kanxue.com/upload/attach/202610/951483_JV2VDD9WVQY4CUV.png)

需要配置两个东西 一个是 include 包含哪些目录，然后 env 就是传给构建的环境变量

这里这个变量很重要，我一直导入不进去的原因就是没有添加这个变量，这里简单提一嘴这玩意，SOONG_ALLOW_MISSING_DEPENDENCIES 是 AOSP 官方 art 构建脚本 buildboot-build.sh 来在依赖 / 源码不全的树上让构建通过，这里之所以加这个的主要原因是因为 lineage23.2 的上游回归的问题（可能，因为也有别人也遇到了这个情况）

```
repo: /home/aosp/android/lineage
lunch: lineage_bluejay-trunk_staging-userdebug
directories:
  include:
    - art
    - frameworks
    - libcore
    - system
    - bionic
  exclude: []
modules:
  include: []
  exclude: []
test_sources: []
other_languages:
  - cpp
build_config:
  flags: []
  env:
    SOONG_ALLOW_MISSING_DEPENDENCIES: "true"
```

这边是根据[](https://bbs.kanxue.com/elink@c78K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6@1K9i4N6W2x3q4)9J5c8Y4t1H3k6s2g2E0M7q4)9J5c8Y4c8J5k6h3g2Q4x3V1k6E0j5i4y4@1k6i4u0Q4x3V1k6H3j5i4c8U0K9r3g2K6i4K6u0r3N6U0t1H3x3U0k6Q4x3X3b7H3z5g2)9J5k6o6p5J5i4K6u0V1k6Y4g2K6K9h3!0F1) https://github.com/tiwe0/r0dump/tree/master/patches/v2026-09-12-fusion 的 patch 内容

搭配着 [](https://bbs.kanxue.com/elink@e2eK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0M7#2)9J5k6h3q4F1k6s2u0G2K9h3c8Q4x3X3g2U0L8$3#2Q4x3V1j5%60.)https://cs.android.com/ 源码来交叉着学习分析的。

fusion 的整个 patch 内容是比较大的，这边以脱壳为主线，走一遍整体运行的流程，对照源码进行学习。

#### zygote fork 加载配置项

源码位置：frameworks/base/core/java/android/app/ActivityThread.java

这里的这个时机是 zygote fork 出 app 进程后，进程接收绑定数据的处理函数。

`createAppContext`​创建了这个进程的第一个与 app 包关联的 context。r0dump 在创建后插入了两个函数，正好处于​`createAppContext`​创建后​`makeApplicationInner`前。在这个时机包信息可用、context 也可用，并且 app 的代码一行都还没有开始执行。

```
final IActivityManager mgr = ActivityManager.getService();
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        final r0dump_FusionConfig r0dumpConfig =
                r0dump_FusionConfig.r0dump_configureForApp(
                        appContext, data.appInfo.packageName, data.processName);
        r0dump_ClassLoaderWalk.r0dump_prepare(r0dumpConfig, data.info, data.appInfo);
        mConfigurationController.updateLocaleListFromAppContext(appContext);
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_3NP9HPEVCN4GXMR.png)

先来看第一个函数 r0dump_FusionConfig.r0dump_configureForApp

源码位置：frameworks/base/core/java/android/app/r0dump_FusionConfig.java

这里这个函数就是相当于是一个总开关，去把配置读取出来然后判断当前进程是否匹配，如果没问题就把配置交给 native 层去做初始化，如果不是就返回 null

![](https://bbs.kanxue.com/upload/attach/202610/951483_VJ8R5VMZGWYEJJB.png)

再来看第二个函数 r0dump_ClassLoaderWalk.r0dump_prepare

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

第一个参数是上一个函数 r0dump_configureForApp 返回的脱壳配置项，第二个是 App 运行时加载对象，里面持有并且能够创建这个 app 的 classloader，第三个参数是 app 的静态信息内容 里边有包名、数据目录、SDK 版本、flags 等等。

然后​`r0dump_REQUEST.compareAndSet`这里是只有第一次调用会存成功，重复调用会被忽略掉，以此来确保一个进程上只存储一份。

这里就只做了存储别的啥也没做，那在哪里触发呢？

![](https://bbs.kanxue.com/upload/attach/202610/951483_5ZY8SZU2DZ6U9MG.png)

#### 触发时机

同样在 frameworks/base/core/java/android/app/ActivityThread.java 里面有两个触发点

第一个时机是在 Application 的 onCreate 跑完之后

第二个时机在第一个 Activity 的 onCreate 跑完之后

这两个时机调用的是同一个函数 r0dump_schedule，只是入参的 flags 不同

![](https://bbs.kanxue.com/upload/attach/202610/951483_4W9PF92BZZDW43E.png)

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

这里把前面存的那个 request 取出来，然后判空，然后再去判断一下配置里的 flags 跟入参是否对的上。

如果说没问题 那就把当前的 context ClassLoader 给记录下来，然后调用​`r0dump_schedule`并且传入两个参数

![](https://bbs.kanxue.com/upload/attach/202610/951483_XPTUKFQ2XHEPB8U.png)

源码位置：frameworks/base/core/java/android/app/r0dump_FileWorker.java

r0dump_schedule 这里的 compareAndSet(false, true) 确保这个线程全程只跑一次，哪怕两个触发时机都调用了 r0dump_schedule 那也只有第一个能够真正创建线程。

这里先做延时等待 r0dump_waitAndServiceFlush，等待过程顺便刷盘；延时到点后执行 work.r0dump_run()（也就是遍历）；遍历完在 finally 里先 r0dump_CompleteWalk 通知 native 遍历结束并强制刷盘，然后进入 r0dump_serviceFlushForever，每 250 毫秒刷一次盘，一直到进程死亡。

![](https://bbs.kanxue.com/upload/attach/202610/951483_ZDUZ9N7Q5ESFWCX.png)

#### 边等边刷盘

源码位置：frameworks/base/core/java/android/app/r0dump_FileWorker.java

`r0dump_waitAndServiceFlush`​这边默认是延迟 5000 毫秒，但是不是直接延迟五秒，是切成最长 500 毫秒的小片来延迟，每过 500 毫秒就调用​`` `r0dump_Runtime.r0dump_ServiceFlush()``，把这段时间可能已经获取到的东西刷到磁盘，刷到没东西可刷为止，再延迟 500 毫秒。

![](https://bbs.kanxue.com/upload/attach/202610/951483_UYERCGWHTPYPZED.png)

源码位置：art/runtime/native/dalvik_system_r0dump_Runtime.cc

`r0dump_ServiceFlush()`​这边调用​`r0dump_ServiceCaptureFlush`函数，继续跟进看

![](https://bbs.kanxue.com/upload/attach/202610/951483_MPFNX5PZ3DJ8Z8X.png)

源码位置：art/runtime/r0dump/r0dump_capture.cc

r0dump_ServiceCaptureFlush 这里取拿当前进程的配置以及暂存区。然后把这两个传入到​`r0dump_ServiceFlush`函数

![](https://bbs.kanxue.com/upload/attach/202610/951483_KYBRWDVN2GWU6NY.png)

源码位置：art/runtime/r0dump/r0dump_file.cc

`r0dump_ServiceFlush`这里先把全局刷盘锁拿到，保证同一时刻全进程只有一个刷盘动作。拿到后再调用 r0dump_ServiceFlushLocked 进行落盘。

![](https://bbs.kanxue.com/upload/attach/202610/951483_8UK4W3UYU4XREG3.png)

#### Main 函数

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

这边就是主要的脱壳逻辑了，总共四个步骤

解析 Manifest、收集类加载器、遍历类和方法、主动调用

![](https://bbs.kanxue.com/upload/attach/202610/951483_87362JZ8AGSKJKY.png)

#### Manifest 解析

源码位置：frameworks/base/core/java/android/app/r0dump_ManifestHelper.java

通过解析 AndroidManifest 可以得到里边声明的 Application、Activity、Service、Receiver、Provider、instrumentation 这些都是 App 自己的类，可以从他们的包名推导出这个 App 的命名空间

![](https://bbs.kanxue.com/upload/attach/202610/951483_C8YV6DDTEGYBQF4.png)

#### 收集类加载器

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

收集 ClassLoader 的三条来源，一个是 App 自己的​`loadedApk.getClassLoader()`​。另一个是前面记录的那个 context ClassLoader。在一个就是​`r0dump_GetRegisteredClassLoaders`从 ART 里面取拿所有注册过的类加载器。把收集到的每个 ClassLoader 调用 getParent() 去拿他的父 ClassLoader 直到走到顶。用 IdentityHashMap 对每收集到的类加载器进行去重。

![](https://bbs.kanxue.com/upload/attach/202610/951483_K92K4XE8VBU5NU3.png)

![](https://bbs.kanxue.com/upload/attach/202610/951483_C6HWCWZC95G55VM.png)

#### 遍历类和方法

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

`r0dump_State`这里按加载器去重, 重复返回 false, 新类 classCount++ 返回 true

![](https://bbs.kanxue.com/upload/attach/202610/951483_ZAV73YJGNQTMJV4.png)

拿到了去重后的 ClassLoader 后依次从四个来源拿类名去进行遍历，先是 cookie、然后是 DexCache、接着是已加载类表、最后是 Manifest

![](https://bbs.kanxue.com/upload/attach/202610/951483_ZEM3MG6W2J6J2A3.png)

#### 捕获（指定类方法，不执行方法体）

![](https://bbs.kanxue.com/upload/attach/202610/951483_XCAFW7CK3C3JH33.png)

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

在类加载器里面找目标类，然后​`r0dump_findForceExecutable`​按照方法名 + 描述符找到唯一匹配。找到了就去执行​`r0dump_Runtime.r0dump_ForceInvoke(executable, null, config.forceClass);`

![](https://bbs.kanxue.com/upload/attach/202610/951483_Z7R5B2FNM7R2B5W.png)

源码位置：art/runtime/native/dalvik_system_r0dump_Runtime.cc

这一段代码挺多的 不全贴了

说一下流程，依旧是检查配置清单项，筛选一下 ArmMethod，筛掉这几个

```
(method == nullptr || method->IsRuntimeMethod() || method->IsAbstract() ||method->IsNative() || !method->HasCodeItem())
```

然后再去判断 java 层传进来的的 className 必须要和方法真正的声明类保持一致

![](https://bbs.kanxue.com/upload/attach/202610/951483_UMK6DWP9J4HWB4Z.png)

强制类初始化

![](https://bbs.kanxue.com/upload/attach/202610/951483_XUKY4D66N6WKD5N.png)

给​`ArtMethod::Invoke`传入 nullptr

![](https://bbs.kanxue.com/upload/attach/202610/951483_P7D5DFSP8JYVQT7.png)

源码位置：art/runtime/art_method.cc

`ArtMethod::Invoke`​的 self == nullptr 的时候就调用​`r0dump_TryHandleForceInvokeNoExecute`然后直接 return 不去执行方法体

![](https://bbs.kanxue.com/upload/attach/202610/951483_DAHE9JVNF2FVGGY.png)

源码位置：art/runtime/r0dump/r0dump_force_invoke.cc

r0dump_TryHandleForceInvokeNoExecute 这里先核对 method 能不能跟前面记录的那个对的上。如果对上了就确认类已经初始化、Codeitem 也在，就复制、标记以及来源，然后存入暂存区，再将 captured 置 true。返回 true 后 invoke 直接就 return 了。捕获到这里闭环。

![](https://bbs.kanxue.com/upload/attach/202610/951483_6EAP2W3X6DZV766.png)

那如果说这里返回 false 呢，那就回到 Main 函数（r0dump_run），继续走 r0dump_forceManifestNamespaces 调用。

![](https://bbs.kanxue.com/upload/attach/202610/951483_XCAFW7CK3C3JH33.png)

#### 捕获（全量类方法，不执行方法体）

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

r0dump_forceManifestNamespaces 这里把需要对哪些类的哪些方法做捕获先确定。合并出命名空间前缀范围，然后按照先 manifest 组件类、再逐个 classloader 的 cookie 类表和已加载类表的顺序，把符合前缀、没处理过的类名清单交给​`r0dump_forceNames`

![](https://bbs.kanxue.com/upload/attach/202610/951483_TWR77R728VSUWE7.png)

源码位置：frameworks/base/core/java/android/app/r0dump_ClassLoaderWalk.java

这里类名依旧进行筛选过滤，然后 Class.forName 加载（不初始化）然后进入​`r0dump_forceExecutables`

![](https://bbs.kanxue.com/upload/attach/202610/951483_4CC2BQER589CMKB.png)

到这里就跟前面是一样的了也是用同一个​`r0dump_ForceInvoke`去发起捕获。只不过在这之前需要先把这个类的构造函数和方法按照名字 + 描述符排序，还要跳过抽象和 native 方法。

![](https://bbs.kanxue.com/upload/attach/202610/951483_SK2FY28H5FZUV5E.png)

到这里 r0dump_run 的四个步骤就全部走完了。接着就是收尾、刷盘、落盘........

对后面步骤感兴趣的朋友可以去拉一下作者的源码去接着看[](https://bbs.kanxue.com/elink@987K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6@1K9i4N6W2x3q4)9J5c8Y4t1H3k6s2g2E0M7q4)9J5c8Y4c8J5k6h3g2Q4x3V1k6E0j5i4y4@1k6i4u0Q4x3V1k6H3j5i4c8U0K9r3g2K6i4K6u0r3N6U0t1H3x3U0k6Q4x3X3b7H3z5g2)9J5k6o6p5J5i4K6u0V1k6Y4g2K6K9h3!0F1) https://github.com/tiwe0/r0dump/tree/master/patches/v2026-09-12-fusion。

从[](https://bbs.kanxue.com/elink@29dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6@1K9i4N6W2x3q4)9J5c8Y4t1H3k6s2g2E0M7q4)9J5c8Y4c8J5k6h3g2Q4x3V1k6E0j5i4y4@1k6i4u0Q4x3V1k6H3j5i4c8U0K9r3g2K6) github 上下载下来，然后直接打 patch，我这边测试了是可以直接打的没有冲突。

```
P=/home/aosp/MYcodes/r0dump/r0dump-master/patches/v2026-09-12-fusion

# 1) art —— 基线 r4
cd /home/aosp/android/lineage/art
git am -3 $P/art/*.patch

# 2) frameworks/base —— 同为 lineage-23.2
cd /home/aosp/android/lineage/frameworks/base
git am -3 $P/frameworks_base/*.patch

# 3) system/sepolicy
cd /home/aosp/android/lineage/system/sepolicy
git am -3 $P/system_sepolicy/*.patch

# 4) 新建 Manager 仓库（32 个 patch 从 root commit 顺序 am）
mkdir -p /home/aosp/android/lineage/packages/apps/R0dumpManager
cd /home/aosp/android/lineage/packages/apps/R0dumpManager
git init && git am $P/packages_apps_R0dumpManager/*.patch

# 5) vendor/lineage 品牌名
cd /home/aosp/android/lineage/vendor/lineage
git apply $P/vendor_lineage/0001-brand-r0dump-fusion-16-display-version.diff
```

但是编译的时候报错，找不到导入包文件，日志报 1 个，让 ai 找出四个。

![](https://bbs.kanxue.com/upload/attach/202610/951483_PDEEBD8DR3TQ346.png)

这边直接让黑奴 AI 补一下这四个文件然后继续编译，后面还是大量报错让 AI 修了老半天。下面是修复过程及原因。

fusion 的 patch 不包含 libcore 整组改动，framework 里 import dalvik.system.r0dump_Runtime 找不到符号，就新建文件补齐代码（patch 0001~0004）。补完还是找不到符号，因为新文件要登记进编译源，在 libcore/non_openjdk_java_files.bp 的 non_openjdk_javadoc_libart_files 里加一行。

然后还是不能编译，metalava 把这个类当新增公开 API，checkapi 报错，在类上加 @hide。

结果 @hide 又把它挡在 framework 编译用的 module-lib API 面之外，还是找不到符号，再加 @SystemApi(MODULE_LIBRARIES)（保留 @hide），并跑 m art.module.public.api.stubs.source.module_lib-update-current-api 把它写进 libcore/api/module-lib-current.txt。到这一步 framework 编译过了，全量 m 也过了。

但刷机后 Manager 能选 app 却抓不到东西，日志 Output directory preparation was rejected。查下来不是 SELinux，是 native helper（r0dump_file_helper）和 MCP bridge（r0dump_mcp_bridge）根本没编进系统——它们是独立 cc_binary，不在默认 PRODUCT_PACKAGES。

单独 m r0dump_file_helper 和 m r0dump_mcp_bridge 能编，但 mka bacon 打包报 artifact path 错误（新版构建系统不让设备 mk 直接往 system/ 装文件）。解决：新建 vendor/lineage/config/r0dump.mk，把 R0dumpManager、r0dump_file_helper、r0dump_mcp_bridge、privapp 权限加进 PRODUCT_PACKAGES，并加 PRODUCT_ARTIFACT_PATH_REQUIREMENT_ALLOWED_LIST 白名单，再在 lineage_bluejay.mk 里 include 它。重新 mka bacon 打包成功，刷入后 helper 正常起来，能抓到日志了。

有一说一，这要是换没有 AI 之前，光安卓 16 编译这一块就得好好研究研究，现在直接让 ai 查资料还是很舒服的。

下面是步骤（让你的 ai 去修一下就可以了，或者后面我改进好了发布到 github 上直接打我的 patch 也行）

```
# 1. 打 4 个 patch（在 libcore 仓库根目录）
cd /home/aosp/android/lineage/libcore
git apply /home/aosp/MYcodes/r0dump/r0dump-libcore-fusion/0001-r0dump-fusion-add-libcore-java-bridge.patch
git apply /home/aosp/MYcodes/r0dump/r0dump-libcore-fusion/0002-r0dump-register-runtime-in-core-libart-sources.patch
git apply /home/aosp/MYcodes/r0dump/r0dump-libcore-fusion/0003-r0dump-hide-runtime-from-public-api.patch
git apply /home/aosp/MYcodes/r0dump/r0dump-libcore-fusion/0004-r0dump-mark-runtime-as-module-lib-systemapi.patch

# 2. 进源码树根目录，配环境
cd /home/aosp/android/lineage
source build/envsetup.sh
lunch lineage_bluejay-bp4a-userdebug

# 3. 重新生成 module-lib API 文本
#    把 r0dump_Runtime 写进 libcore/api/module-lib-current.txt，
#    否则 framework 编译期还是找不到符号）
m art.module.public.api.stubs.source.module_lib-update-current-api -j$(nproc)

# 4. 全量编译（system/vendor/product 等镜像）
m -j$(nproc)

# 5. Manager 应用（不在默认 PRODUCT_PACKAGES，要单独编）
m R0dumpManager -j$(nproc)

# 6. 打 OTA 更新包
cd /home/aosp/android/lineage
source build/envsetup.sh
lunch lineage_bluejay-bp4a-userdebug
mka bacon
# 产出：out/target/product/bluejay/lineage-23.2-<日期>-UNOFFICIAL-bluejay.zip

# 查看一下是哪个
ls -lht out/target/product/bluejay/*.zip | head

# 设备在正常系统里，连 adb
cd ~/android/lineage/out/target/product/bluejay

# 进 recovery（跳过格式化那步）
adb reboot recovery

# recovery 菜单选 Apply update → Apply from ADB（不要碰 Factory reset）

# 电脑侧刷入
adb devices
adb -d sideload lineage-23.2-20260925-UNOFFICIAL-bluejay.zip

# 刷完回主菜单 → Reboot system now
```

整体使用起来还是比较丝滑的。其实这里已经搭好架子了，后面可以在现有的基础上对脱壳系统和管理器 app 在做一些调整和优化让使用体验再上一个台阶，这算是后话了，后边要是我改好了应该会放在 github。

![](https://bbs.kanxue.com/upload/attach/202610/951483_H357BSR3WDHEZNR.png)

#### 工具链安装

```
cd ~/.local/toolchains
curl -fsSL -C - -o arm-toolchain.tar.xz https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu/12.2.rel1/binrel/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-elf.tar.xz
tar -Jxf arm-toolchain.tar.xz && rm -f arm-toolchain.tar.xz
```

#### 持久化环境变量

```
printf '%s\n' 'export KP_TOOLCHAIN=$HOME/.local/toolchains/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-elf' 'export PATH=$KP_TOOLCHAIN/bin:$PATH' 'export TARGET_COMPILE=aarch64-none-elf-' >> ~/.bashrc
source ~/.bashrc
aarch64-none-elf-gcc --version
```

![](https://bbs.kanxue.com/upload/attach/202610/951483_KQPGFZRXVGNF493.png)

#### 编译模块

这里要先去下载一下 KernelPatch

```
https://github.com/bmax121/kernelpatch
```

```
cd ~/MYcodes/KPM_module/KernelPatch/kpms/demo-hello
make clean
make
file hello.kpm
readelf -S hello.kpm | grep '\.kpm\.'
```

期望：ELF 64-bit LSB relocatable, ARM aarch64，且能看到 .kpm.info .kpm.init .kpm.exit .kpm.ctl0 .kpm.ctl1 五个段。

![](https://bbs.kanxue.com/upload/attach/202610/951483_EJN86YKAU5JH3P2.png)

#### 推到手机

```
export PATH=$PATH:~/platform-tools
adb push hello.kpm /data/local/tmp/
adb shell 'chmod 644 /data/local/tmp/hello.kpm'
adb shell 'ls -l /data/local/tmp/hello.kpm'
```

#### 加载并验证

```
adb shell '/system/bin/truncate apatch设置的密码 module load /data/local/tmp/hello.kpm t1'
```

成功会回显路径 /data/local/tmp/hello.kpm；失败会打印 supercmd error code: N，同时 adb shell dmesg | tail -40 里有 logke 的具体原因（未解析符号 / 重定位类型不支持）。

![](https://bbs.kanxue.com/upload/attach/202610/951483_WXZWTTY4SGC8URF.png)

```
adb shell '/system/bin/truncate apatch设置的密码 module num'
adb shell '/system/bin/truncate apatch设置的密码 module list'
adb shell '/system/bin/truncate apatch设置的密码 module info kpm-hello-demo'
adb shell 'logcat -b kernel -d | grep -E "kpm hello|kernelpatch version"'
```

判据：module num 变 1，且 adb shell 'logcat -b kernel -d | grep -E"kpm hello|kernelpatch version"' 打出 kpm hello init, event: load-file, args: t1 和 kernelpatch version: d03。

![](https://bbs.kanxue.com/upload/attach/202610/951483_R36C2Y2PKPS47WQ.png)

#### 控制与卸载

```
adb shell '/system/bin/truncate apatch设置的密码 module ctl0 kpm-hello-demo ping'
adb shell '/system/bin/truncate apatch设置的密码 module unload kpm-hello-demo'
adb shell 'dmesg | grep "kpm hello exit"'
adb shell '/system/bin/truncate apatch设置的密码 module num'
```

ctl0 期望回显 echo: ping（这是内核 compat_copy_to_user 写回用户态的）。卸载后 num 回 0。

![](https://bbs.kanxue.com/upload/attach/202610/951483_VE9SZJYDM39M3DV.png)

我当前的安卓源码是无法以 spawn 的方式去启动 app 的，尝试了更换最新的 fridaserver 还是一样的报错

![](https://bbs.kanxue.com/upload/attach/202610/951483_RRVJ7RUAARPCHNW.png)

让 ai 去彻查了一下原因。结果是因为 frida 在注入 zygote 前，要先在目标进程内存里面找到 android.os.Process.setArgV0 的方法入口指针存放位置。他扫描哪些内存段，由这个判断决定（frida-core/src/linux/linux-host-session.vala 的 is_boot_heap）

```
if (!m.readable || !m.writable || m.executable || m.shared) return false;
return "boot.art" in m.path || "boot-framework.art" in m.path
    || "dalvik-LinearAlloc" in m.path;
```

而当前我手机上的安卓 16 的 ART 把启动镜像的方法段单独放进了一个名为 / memfd:/boot-image-methods.art 的可写私有映射，同时 boot.art、boot-framework.art 在映射表里是只读。结果就是：满足可写的文件名不在白名单里，符合白名单的映射又是只读，两条都不成立，扫描集合里面没用那块内存，指针对不上，报 unable to locate ... setArgV0() slot。

那么只需要让白名单有 boot-image-methods 就行了，修复后重编就可以正常 spawn 了。

要隐藏 frida 首先需要知道 frida 都有什么特征，下面特征来自 frida 17.15.0。最终我会提供完整的 patch 和 KPM 模块。

首先特征分为五个阶段，启动 fs 后但没注入任何进程、注入过程中、会话保持期间、hook 动作、会话结束后

特征点其实还是蛮多的（有不少检测方案是某些企业加固也在使用的）

这一阶段的特征与服务是否被使用无关，只要进程存在就成立。

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">特征</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">实测取值</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">观测方式</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">能否消除</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">控制端口</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>127.0.0.1:27042</code> 处于 LISTEN，使用默认参数启动即存在，通过 USB 连接时同样存在</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">对该端口执行 bind 观察是否返回 EADDRINUSE，或直接发起 connect</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可由内核侧隐藏监听状态；改端口会导致主机侧无法通过 USB 连接</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">辅助监听</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">broker 端口绑定在 ​<code>*</code>​ 上，实测为 ​<code>34539</code>​、​<code>33193</code> 等随机值，会话结束后不释放</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">同上；该端口对外部网络也可达</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可，改为绑定 ​<code>127.0.0.1</code> 后该特征消失，已验证</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">抽象命名空间 socket</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">名称形如 ​<code>@/frida-zymbiote-&lt;32位十六进制&gt;</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">读取 ​<code>/proc/net/unix</code>，按前缀匹配</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可，但服务端查找串与仓库中预编译的注入体需同步等长修改</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">HTTP 行为</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">对 ​<code>GET /</code>​ 返回 ​<code>404 Not Found</code>​；对 WebSocket 升级请求返回 ​<code>400</code>​；对 ​<code>AUTH</code> 请求不返回任何数据</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">本机连接后发送对应报文</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">部分可，取决于是否愿意移除该层功能</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">服务进程线程名</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">含 ​<code>frida-android-helper</code> 等</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">枚举 ​<code>/proc/&lt;pid&gt;/task/*/comm</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">临时文件与本地 socket</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>/data/local/tmp</code>​ 下的 helper dex 文件，以及形如 ​<code>/frida-helper-&lt;id&gt;</code> 的本地 socket</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目录枚举、读取 ​<code>/proc/net/unix</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">服务进程自身内存中的明文</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">仍含 500 余处 frida 相关字符串，来源多为编译期源码路径</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标进程无法读取，仅文件取证或以 root 读取该进程内存时可见</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可通过构建参数消除源码路径部分</td></tr></tbody></table>

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">特征</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">实测取值</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">观测方式</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">能否消除</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">被跟踪状态</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">注入期间 ​<code>TracerPid</code> 短暂非零，注入完成后归零，稳态下读不到跟踪关系</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">轮询 ​<code>/proc/self/status</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不需要；也可由内核侧覆写</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">进程方法入口</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">以 spawn 方式注入时，启动镜像中 ​<code>setArgV0</code> 对应的方法入口指针在恢复执行前指向外部地址，恢复后被还原；宿主 zygote 侧的修改是持续的</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">进程扫描自身内存中该启动镜像段，检查该指针是否仍指向原来的原生库</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可。以附加方式注入时不触发</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">注入体位置</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">注入代码被写入 ​<code>libstagefright</code>​、​<code>libmedia</code> 等库已映射的可执行页</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">将这些文件映射的内存内容与磁盘文件比对</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可</td></tr></tbody></table>

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">特征</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">实测取值</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">观测方式</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">能否消除</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">代理库映射名</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">形如 ​<code>/memfd:frida-agent-64.so (deleted)</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">读取 ​<code>/proc/self/maps</code> 的路径字段</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">代理库映射条数</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">固定 9 条</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">统计条目</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">与合法映射的命名冲突</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">若改用与运行库相同的名字，会出现同名称但 inode 不同的条目（实测合法侧 inode 3460，注入侧 1455）</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">按名称分组后检查 inode 是否唯一</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可，除非内核同时伪造 inode</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">线程名</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">4 条与事件循环、RPC 调度相关的线程，其中池线程名由进程名派生</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">读取 ​<code>/proc/self/task/*/comm</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">部分可。个别名称位于依赖的静态库内，不在 frida 源码范围内</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">线程总数</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">21 增至 25</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">统计 ​<code>task/</code> 目录</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">文件描述符数量</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">116 增至 123，新增项包含 socket、eventfd、eventpoll 以及一个 userfaultfd</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">统计 ​<code>/proc/self/fd</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可。其中 userfaultfd 在普通应用中极少出现</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">匿名可读写可执行映射</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">3 条增至 7 条</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">统计 ​<code>rwx</code> 匿名映射数量；Android 正常进程此类映射本就少见</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">进程出向连接</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标进程自身与控制端口之间存在 ESTABLISHED 连接，该通道走 TCP 回环而非本地套接字</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">读取 ​<code>/proc/self/net/tcp</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">需内核侧处理</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">内存内容中的标识字符串</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">未修改版本在自身映射内可检出 frida、gum 等标识；完成改名与注入体改写后，对 2019 个可读范围、共 676MB 全量扫描结果为 0。另有 10 处 frida 子串命中，均落在系统词表中的英文单词内</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">遍历自身可读映射做子串扫描</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">可，已验证</td></tr></tbody></table>

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">特征</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">实测取值</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">观测方式</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">能否消除</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标函数入口</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>fopen</code> 起始 8 字节被替换为跳转指令，随后 8 字节为跳板地址。写入不是同步完成的，实测有秒级延迟；会话结束恢复原值</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">将关键函数起始字节与磁盘上对应库文件比对</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可。改变插桩方式可以回避改指令，但不能回避 "改指令" 这一行为本身</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">跳板与返回地址</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">跳板位于 frida 自身的匿名段</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">回溯时校验返回地址是否落在合法文件映射内</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">过程链接表</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">直接改写指令的手法不修改表项，表改写手法不修改指令</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">两类检查互为补充</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">取决于所选手法</td></tr></tbody></table>

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">特征</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">实测取值</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">观测方式</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">能否消除</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">匿名可执行映射数量</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">未回落，基线 3 条，会话中 7 条，卸载后停在 6 条</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">任一时点统计一次即可</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不可</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">连接记录</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">控制端口的连接转入 TIME_WAIT，服务停止后记录仍在</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">读取 ​<code>/proc/net/tcp</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">需内核侧处理</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">代理映射、线程、文件描述符</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">全部回收，无残留</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">—</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">—</td></tr></tbody></table>

上述特征分成两类。第一类是名称和内容，包括映射名、符号名、socket 名、线程名、内存中的标识字符串，这类可以完全消除，本机已验证在目标进程内扫描结果为 0。第二类是数量与关系，包括映射条数、线程数、文件描述符数、可执行匿名段数、端口占用状态、连接记录、被改写的指令字节和方法入口指针，这类无法通过修改 frida 消除，只能由内核侧覆写观测结果，或改变注入方式规避其中个别项。

实际上并不需要死磕 frida，后面有空可以研究一下无痕 hook，就不需要花时间绞尽脑汁藏拙了，但是现在来都来了，不管了

魔改去除 frida 特征后在配合上 KPM 的特征隐藏 可以过以下点

<table><thead><tr><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">特征</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">原版 frida 的样子</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">谁抹的</th><th data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">现在</th></tr></thead><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">maps 里 agent 映射行</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>/memfd:frida-agent-64.so (deleted)</code> ×9</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">源码改名成 ​<code>svc-profile</code>​ + <strong>KPM 整行抹</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>9 → 0</strong></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">maps 里匿名可执行段</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">3 条 ​<code>rwxp … 00:00 0</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>KPM 整行抹</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>3 → 0</strong></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">内存明文标识</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>frida</code>​/​<code>gum</code>​/​<code>GumJS</code>​/ 版本串 /​<code>frida_agent_main</code> 遍布</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>源码 patch</strong>（改名 + 等长 blob 重写）</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">全量扫 676MB <strong>0 命中</strong></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">agent 导出符号</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>frida_agent_main</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">源码 patch 改名</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">搜不到</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>/proc/net/tcp</code> 端口行</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>0100007F:69A2</code> LISTEN + EST + TIME_WAIT</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>KPM 删行</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标读不到</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>/proc/net/tcp</code>​ broker 绑 ​<code>*</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">随机口绑在 ​<code>0.0.0.0</code>（外部可达）</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>源码改回环</strong>（127.0.0.1）</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">不再是外部可达特征</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>/proc/net/unix</code> 抽象套接字</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>@/frida-zymbiote-&lt;hex&gt;</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">源码改名 ​<code>@/glsyn-syncsock</code>​ + <strong>KPM 删行</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>5 → 0</strong></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">线程名</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>gum-js-loop</code>​/​<code>pool-frida</code>​/​<code>linjector</code>​/​<code>gmain</code>​/​<code>gdbus</code></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">源码改 ​<code>profile-loop</code>​/​<code>pool-stats</code>​ + <strong>KPM 把 comm 抹成空格</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标自读 task/*/comm → 空</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>/data/local/tmp</code> server 文件</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><code>frida-server</code>​、​<code>frida-helper*.dex</code> 可按名 stat/open</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">源码改文件名 ​<code>frn15f</code>​/​<code>fs64</code>​ + <strong>KPM stat/access→ENOENT</strong></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">目标 stat → <strong>不存在</strong></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">端口可用性探测</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">connect 通 / bind EADDRINUSE</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><strong>KPM</strong>：connect→ECONNREFUSED、bind→谎报成功</td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left="">三角度都显示 "端口空闲"</td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""></td></tr></tbody></table>

边界的话就是完整性校验和 inlinehook 特征这两个没招，属于是 hook 框架的死穴。

最终产出的效果我还是比较满意的，确确实实是能在分析过程中起到一定的帮助，在构建和开发的过程中也收获了许多知识点，需要注意的是，一定要打好虚拟机快照，避免主机异常关机导致的数据丢失（避雷某耳机品牌）。

最后，国庆快乐！！！

参考文章

[《](https://bbs.kanxue.com/thread-271043-1.htm)​**[Linux 内核监控在 Android 攻防中的应用](https://bbs.kanxue.com/thread-271043-1.htm) **​[》](https://bbs.kanxue.com/thread-271043-1.htm)

[《](https://bbs.kanxue.com/thread-274546.htm)​** [定制 bcc/ebpf 在 android 平台上实现基于 dwarf 的用户态栈回溯](https://bbs.kanxue.com/thread-274546.htm) **​[》](https://bbs.kanxue.com/thread-274546.htm)

**[使用 Kimi K3 进行脱壳工具迁移开发：R0DUMP —— 将 FART 迁移到 Android 16](https://bbs.kanxue.com/thread-292107.htm)**

**[](https://bbs.kanxue.com/elink@75dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2*7M7$3E0C8K9#2)9J5k6h3y4F1i4K6u0r3M7r3!0K6N6s2y4Q4x3V1j5K6x3e0b7H3y4W2)9J5c8R3%60.%60.)Apatch 内核模块搭建到隐藏 Frida 注入**

‍

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)

最后于 2 小时前 被北袅编辑 ，原因：