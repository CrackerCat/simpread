> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484008&idx=1&sn=7a3a1655da795d625e382b21fee580f5&chksm=cebb4726f9ccce30dbafaeb210d0da75eb182665c4fc9531d932c1e889eec38dabafe27182f4&scene=21#wechat_redirect)

一、获取 Android 源码网络配置可以访问 google（方法一）

二、获取 Android 源码网络配置可以访问 google（方法二）

三、Ubuntu18.04 下更改 apt 源为阿里云源

四、代理导致网络无法访问时，证书问题的解决方法

五、Ubuntu 环境普通用户自动化下载 LineageOS 支持机型的各版本 Android 系统源码的操作方法

六、MoKee（魔趣）或 LineageOS

七、MoKee 魔趣

八、常用的 Android 系统源码编译基础

九、端口占用问题的解决方法

十、Ubuntu 16.04 上增加 Swap 分区 

十一、磁盘空间不足可以采用如下方法扩展磁盘空间

十二、无法访问 google 的情况下魔趣 Mokee 镜像添加编译系统的设备选型号（lunch）

十三、无法访问 google 的情况下 Lineage OS 镜像添加编译系统的设备选型号（lunch）

**一、获取 Android 源码网络配置****可以访问 google****（方法一）**   

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYvs12D1BAgoC2NyOF4J1zp1hgfBZia8IcouGjpauLx8AXiaggIZAh9oicA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYxb09Fibx4FHoupqA6Al8R0Wp7MR52rNYIGD3bSY7GK5gOQKkWzs4ybg/640?wx_fmt=png)

**设置 http、https 以及 ftp 代理****（****其中 IP 为主机 IP 地址****）**

export http_proxy=socks5://172.20.10.2:1080

export https_proxy=socks5://172.20.10.2:1080

export FTP_PROXY=socks5://172.20.10.2:1080

export ALL_PROXY=socks5://172.20.10.2:1080

**设置 http 代理****（****其中 IP 为主机 IP 地址****）**

git config --global http.proxy http://172.20.10.2:1080

git config --global https.proxy https://172.20.10.2:1080

git config --global user.email "****@email.address"

git config --global user.name "****"

**取消 http 代理**

git config --system (或 --global 或 --local) --unset http.proxy

git config --system (或 --global 或 --local) --unset https.proxy

**二、获取 Android 源码网络配置****可以访问 google****（****方法二****）** 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUY1PTek718zibWVh9Dgq5IYibmjxLubXmvIV0UAXNHGnWkeFzV3vtVhy5A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUY01XSqHFVCp9Uiavl5p8mhmOjvjRyiaceSziceIq8e5lJEM1ibQVRKf1ojQ/640?wx_fmt=png)

**三、Ubuntu18.04 下更改 apt 源为阿里云源**

sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

sudo apt-get install vim

**将原有的内容注释掉，添加以下内容**

sudo vim /etc/apt/sources.list

**或**

sudo gedit /etc/apt/sources.list

deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

**更新软件列表**

sudo apt-get update

**更新软件包**

sudo apt-get upgrade

**四、代理导致网络无法访问时，****证书问题的解决方法**

**取消代理**

git config --global --unset http.proxy

git config --global --unset https.proxy

**删除本地配置文件**

apt-get purge openssl

rm -rf /etc/ssl

**重新安装**

sudo apt-get install libssl-dev openssl

apt-get install ca-certificates

**五、Ubuntu 环境普通用户自动化下载 LineageOS** **支持机型的各****版本 Android** **系统****源码****的****操作方法**

**(1). 获取 root 权限**

sudo su

**按 https://wiki.lineageos.org/devices / 的操作步骤获取 Android 系统源码（各机型获取源码的方法类似），如图所示:**

https://wiki.lineageos.org/devices/hammerhead/build（其他机型类似）

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYVp2kUpgEwqtoR27WhEicUFYzsfsxOxdBn3QbFEFLHLckGNhGicDMSu0A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYFzKS8gGoa9nBOv1UNCwgU7Wpbk1ibekga2JOGWI0b2bRNkY9iaRRvvKQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYodFcKA79HSiafavS6SI06UnV4Y3xibJh7lQsrR3PBmdS8z7UREp9AfTg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYpksMehaib5jRzjxPxkaicMvrlsY9ic3TLVGcZ1uYy3kXzHvfqo3HIpgYA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYuB66UfUAvZMx8LmhSEYhRPRYMqpqdkMo5HkBSFiazcKUpDqayNZJ8Uw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYib3ibudtME9gW4iak6IUj3ibsgqwaDdeNPvFqAXYe9HoicJb6fROEsVyJWg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYcBNuBes3DfGCzxsC3YRcjZhRC0bBAgwia8GVzibeKvhX41aPIHEdERibw/640?wx_fmt=png)

**六、MoKee（魔趣）或 LineageOS**

**(1). 通过 git 下载各机型支持的 device、kernel、vendor 系统源码的方法（各机型类似），如下所示:**

https://github.com/LineageOS/android_device_xiaomi_cancro

https://github.com/LineageOS/android_kernel_xiaomi_cancro

https://github.com/MoKee/android_vendor_xiaomi_cancro

https://github.com/MoKee/android_device_xiaomi_cancro

https://github.com/MoKee/android_kernel_xiaomi_cancro

https://github.com/LineageOS/android_device_lge_hammerhead

https://github.com/LineageOS/android_kernel_lge_hammerhead

https://github.com/MoKee/android_vendor_lge_hammerhead

https://github.com/MoKee/android_kernel_lge_hammerhead

https://github.com/LineageOS/android_device_huawei_angler

https://github.com/MoKee/android_vendor_huawei_angler

https://github.com/LineageOS/android_kernel_huawei_angler

**(2). 配置 Android 系统源码编译环境（普通用户）**

vim ~/.bashrc

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYfIbicicmNzAhIp2IibwpIqlC6eaErfAwXDxc6cfo6mtoRvMbko6faRoaA/640?wx_fmt=png)

source ~/.bashrc

./prebuilts/sdk/tools/jack-admin kill-server

./prebuilts/sdk/tools/jack-admin start-server

**(3). 配置 Android 系统源码编译环境（root 用户）**

vim ~/.bashrc

export USE_CCACHE=1

export CCACHE_EXEC=/usr/bin/ccache

ccache -M 50G

export CCACHE_COMPRESS=1

export ANDROID_JACK_VM_ARGS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4096"

export JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4096m"

export LC_ALL=C

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYRwHP7tVXOnbEZkVhuPdr2Jx0EgF0xtTEEyUO3Lasl259U7Miaibbhlug/640?wx_fmt=png)

source ~/.bashrc

./prebuilts/sdk/tools/jack-admin kill-server

./prebuilts/sdk/tools/jack-admin start-server

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYicrtO4OKHCMGaaEVFkGCL0ziarafnX5THglMQQPDMrMzk6XLnawiaWILQ/640?wx_fmt=png)

**七、MoKee** **魔趣**

**安装 Repo**

**在当前用户的根目录创建 bin 文件夹，并添加到系统环境变量中**

mkdir ~/bin

PATH=~/bin:$PATH

**下载 Repo 并给予执行权限**

curl https://download.mokeedev.com/git-repo-downloads/repo > ~/bin/repo

**curl https://raw.githubusercontent.com/MoKee/git-repo/stable/repo > ~/bin/repo**

chmod a+x ~/bin/repo

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYLZMRMFtiatMnQiaGkWgHWTM389U7dg9tc8VOdtUJZ54IWwXSVt3nIa5w/640?wx_fmt=png)

**8.1 的魔趣源码，分支修改成 mko-mr1**

repo init -u https://github.com/MoKee/android -b mko-mr1

repo sync

repo sync -f -j8

repo sync -c -f -j8 --force-sync --no-clone-bundle

**编译 Android8.1 的命令**

make bacon

make bacon -j4

make bacon -j8

make bacon -j16

**9****.****0** **的魔趣源码，分支修改成** **mkp**

git config --global http.proxy http://192.168.101.102:1080

git config --global https.proxy https://192.168.101.102:1080

git config --global user.email "gouyp@email.address"

git config --global user.name "gouyp"

repo init -u https://github.com/MoKee/android -b mkp

repo init -u https://github.com/MoKee/android -b mkp-dev

**repo init -u https://github.com/MoKee/android.git -b mkp**

repo sync

**9****.****0** **的魔趣源码****的编译**

**. build/envsetup.sh && lunch (enter device number) && mka bacon**

**repo init -u https://github.com/MoKee/android -b mkp --depth 1**

repo sync

. build/envsetup.sh

lunch

**编译选项:28（****OnePlus 6** **机型）**

mka bacon

**mka bacon** **-j8**

**Android 系统一键刷机的方法**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYgr57fCz5nZZbPdeniaC4Ff3Uxz8eE7B6H6O8JnKu6CYnsQER0t3rTkg/640?wx_fmt=png)

fastboot --disable-verity --disable-verification flash vbmeta stock_vbmeta.img  

fastboot flash vbmeta vbmeta.img

fastboot flash boot boot.img

fastboot flash system system.img

fastboot flash userdata userdata.img

fastboot -w reboot

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUY6Ba4P9Qc0cDKqgWP1IELia7Qr3eWsRJgCmBMVlucPPIgGbKGtsCoSLg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYG8Y0hhVcV5eZjcCfK9qjibBzjmicp4eC8GVQtR41icgQZS01rJcpnW8Nw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYiargVDYbzE3AkxhW7UAXRjmXvxZNvVXu7PNP3IswcOnQukmGm45sp0g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYBoic1bInic5oicf4OpZEsHh3EibIhTadOHbMxfku8WgKX1ibjw1QHkBkgkg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYfRatKzB9icYUeExibRnqPcGr7tnflJBMD4p1QEIxaBh2ia7wKLZ4ts9uQ/640?wx_fmt=png)

**八、常用的 Android 系统源码编译基础**

**删除所有配置所编译输出的结果文件**

make clobber

**添加系统 API 或者修改 @hide 的 API 后，需要执行如下命令，然后再 make**

make update-api

**修改公共 API 后，需要执行如下命令，然后再 make**

make update-api

**分模块编译**

编译 boot.img

make bootimage

**编译 userdata.img**

make userdataimage

**编译 system.img**

make systemimage

**重新打包 system.img**

make snod

**将 App 预装到系统中（需要预装的 App 是以源码形式提供，则需要先编译）**

**在源码根目录执行以下命令**

source build/envsetup.sh

mmm packages/apps/TestApp

**编译完成后，会在 out/target/product/xxx/system/app / 路径下生成对应的 apk 文件 (xxx 为设备代号)，如果已经有 apk 文件则直接放在该路径下；如果是系统应用，则应放在 out/target/product/xxx/system/priv-app / 路径下，接下来需要重新打包成镜像文件，回到源码根目录，执行以下命令重新打包 system.img**

make snod

**九、端口占用问题的解决方法**

netstat -tln | grep jack - 端口号，只查看端口 jack - 端口的使用情况

lsof -i:jack - 端口号 查看端口属于哪个程序，端口被哪个进程占用

kill -9 jack - 端口号进程 pid

sudo vim ~/.jack-settings

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYib97rHxZTcfX0d3EwZsQ7wkbvwgD9yjrciab01FhaJQwRX8pjR6wEhUA/640?wx_fmt=png)

sudo vim ~/.jack-server/config.propertie 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYosBPlvMU5mUAlTsPB4DUVicKqIhxInibYiafe8bJSLoPM2KzZn6Yiae4Lg/640?wx_fmt=png) 

sudo chmod 600 ~/.jack-settings

sudo chmod 600 ~/.jack-server/config.properties

sudo vim ~/.bashrc

sudo source ~/.bashrc 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYibubBNmm3tLmQDoB8NmibiaPz3pSB6ay6cbblic8d6P3SxyibsOiba73A0Fg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYdrbibrVW03Je2xXxr2NqKZiccHPt7DEvQA2u3TxEdXPn12brQ3ScrPJw/640?wx_fmt=png)

export USE_CCACHE=1

export CCACHE_EXEC=/usr/bin/ccache

ccache -M 50G

export CCACHE_COMPRESS=1

export ANDROID_JACK_VM_ARGS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4096"

export JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4096m"

export LC_ALL=C

./out/host/linux-x86/bin/jack-admin stop-server

./out/host/linux-x86/bin/jack-admin start-server

./prebuilts/sdk/tools/jack-admin kill-server

./prebuilts/sdk/tools/jack-admin start-server

**十、Ubuntu 16.04 上增加 Swap 分区** 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUY2ibFG2bXBGu9lMibnL8JAdDYJLlOFEsSib4L4x2aiazyqyR9PObDsYRqzA/640?wx_fmt=png)

**查看磁盘空间大小**

df -h

**查看磁盘中的文件大小**

du -sh

**查看内存使用情况**

free -m

gnome-system-monitor

**十一、磁盘空间不足可以采用如下方法扩展磁盘空间**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYKIZNCyRx9aEtm8CIAibOC852PGicLy1FqdXrAJLdQvfSq6Biab1HfQuYw/640?wx_fmt=png)

**安装 gparted 分区管理软件**

sudo apt-get install gparted

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYcgiaNOEcaakrXmwTfbexOHCBW18ptJePoG2mWCcGLWibWY6PRVsiaoibyA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUY0GZDBXP5Cs5ED4A7AGWuia2IfSmhsmE21Lwpkyic1nhkicC7Fl0hSXRSg/640?wx_fmt=png)

https://blog.csdn.net/u011345885/article/details/73060897

**十二、无法访问 google 的****情况****下****魔趣 Mokee 镜像添加****编译系统的****设备选型****号（lunch）** 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYozsRzfbo1Eb84pFDZEXSNEWYrAicVsWjciaBy5ncXFALHhUic8AulyVuQ/640?wx_fmt=png)

**十三、无法访问 google 的****情况****下 L****ineage** **OS** **镜像添加****编译系统的****设备选型****号（lunch）**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYicPhso3kDE5TFXzyD4Jic1tBLWaTO4iahRSnnWxmFBaNDHdx2y2vsNgGA/640?wx_fmt=png)

vendor 和 device 相应子目录下的 vendorsetup.sh 文件的实现，它们主要就是添加相应的设备型号及其编译类型支持到 Lunch 菜单中去

把 device/huawei/angler/vendorsetup.sh 中的 add_lunch_combo aosp_angler-userdebug 添加到 vendor/cm/vendorsetup.sh 的最后，编译源码时就可以输入 lunch 命令获取到设备选型号

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYqahlxmUIZYn2I1NL3L0fakGfbvCH4bxT44S7NQEPu3haYX6Y6fxouA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF2SrZIhqP3dVe7L1QrReiaUYw6dRs6uqNAkPAlngmUo50T7ic4EiaXYMWugoR6g5tmYxvzpXhWYoUgkw/640?wx_fmt=jpeg)