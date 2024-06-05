> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282034.htm)

> [原创] 小米路由器固件仿真模拟方案

前言
==

书接[上回](https://bbs.kanxue.com/thread-281901.htm)，由于我们学校实验室里正好用的是小米`AX9000`这款路由器，因此当时我就选了这款设备挖，也就没有对固件仿真了。既有女朋友又努力上进的`ZIKH26`师傅与我这个摆烂人不同，在复现我这个洞的时候，想要研究一下小米路由器的仿真模拟。正好最近我也没啥事，于是也就一起看了看。

恰巧前几天也有师傅私信我关于小米路由器该如何仿真的问题，并且网上目前似乎还没有师傅分析过小米固件的仿真，因此我就写了此文放了出来（所以，我说`FirmAE`是对某些固件特制的学术玩具会不会被打，逃）。

![](https://bbs.kanxue.com/upload/attach/202406/949925_SEK4HNC2U6H5XEK.png)

除了一些特殊的设备，其实对于固件的仿真模拟，总结成一句话就是：**IoT 设备的仿真过程就是不断地改报错，只要与外设硬件无关，都是小问题**。也就是说，主要是要有耐心。

这里依然以小米路由器`AX9000`为例，小米其他型号的路由器也都可以用如下的方案仿真。在本文的最后，会用我们仿真模拟的路由器环境验证`CVE-2023-26315`这个漏洞。

环境配置
====

小米路由器`AX9000`相对其他一些型号来说，仿真稍微复杂一些，因为小米`AX9000`的固件是`AArch64el`架构的，而网上似乎还没有公开的`AArch64`的内核与文件系统。因此，我们需要自己装一个`AArch64`的虚拟机，并从中`extract`提取出内核与磁盘镜像。可参考[这篇文章](https://www.diozero.com/boards/qemuaarch64_bullseye.html)中的步骤完成。

我本人比较懒，装`AArch64`虚拟机很耗时间和内存，所以这部分是`ZIKH26`师傅提取的内核与磁盘镜像，在此表示崇高的敬意。

关于网络环境配置的问题，这里就不再多说了。只简单提一句，`qemu-system-aarch64`启动后，可以用`ip addr`看到`enp0s1`网卡是`DOWN`的状态，且没有分配`IP`地址。

![](https://bbs.kanxue.com/upload/attach/202406/949925_2Q4GZQS26GZG7K9.png)

因此，我们需要手动分配一个`IP`地址（与网桥`br0`处于同一网段即可，`qemu`的`enp0s1`接口与宿主机的`eth0`接口通过虚拟网桥`br0`转发数据），然后再将`enp0s1`网卡给`UP`启用即可，命令如下：

```
ip add add 192.168.192.132/24 dev enp0s1
ip link set enp0s1 up

```

完成后，`enp0s1`网卡的状态应该正常了，再检测下`qemu`虚拟机与宿主机能否互相正常通信：

![](https://bbs.kanxue.com/upload/attach/202406/949925_SHVA3H5JSUDVN2D.png)

下面，就正式开始仿真模拟了。

起手式
===

将固件解压后，首先将文件系统用`scp`传入`qemu`虚拟机，然后就是最经典的三行起手式：

```
mount --bind /proc proc
mount --bind /dev dev
chroot . /bin/sh

```

好吧，这一步我竟然还截了一张图：

![](https://bbs.kanxue.com/upload/attach/202406/949925_QQD3BPF7SPBXWUF.png)

根据`openwrt`的内核初始化流程，按理说应该先启动`/etc/preinit`，其中会执行`/sbin/init`进行初始化，但是在这套固件仿真的时候，这样会导致`qemu`重启，所以我们首先先执行`/sbin/init`中最重要的`/sbin/procd &`，启动进程管理器即可。

![](https://bbs.kanxue.com/upload/attach/202406/949925_8YYMPG4EUUY26AH.png)

启动 httpd 服务
===========

下面就该启动`httpd`服务了，简单检索一下，发现有`uhttpd`，`mihttpd`，`sysapihttpd`。进一步查看一下配置文件（如`/etc/sysapihttpd/sysapihttpd.conf`），发现`sysapihttpd`其实就是`nginx`，监听了`80`端口，有了`nginx`自然就不需要再启动`uhttpd`了。而`mihttpd`中监听了`8198`端口，定义了一些文件上传下载的`API`，可以暂时先不启。

![](https://bbs.kanxue.com/upload/attach/202406/949925_4VN3629DGSQJVSK.png)

所以，只需要启动`sysapihttpd`，执行`/etc/init.d/sysapihttpd start`即可：

![](https://bbs.kanxue.com/upload/attach/202406/949925_CP7QZ5F6FFQJYZW.png)

首先，报错缺失`/var/lock/procd_sysapihttpd.lock`这个文件，这个创建一下对应的目录和文件就行了。接着，会报错`Failed to connect to ubus`，很显然这里是用到了`ubus`总线通信，我们需要启动`/sbin/ubusd &`。但是，接下来又继续报错`usock: No such file or directory`，但是并没有给出缺失哪个文件，因此我们需要将`ubusd`拖进`IDA`定位一下报错点。

很容易定位到`sub_20B0`函数，我们执行的是`ubusd`，而不是它的软链接`tbusd`，因此`v8`会是路径`/var/run/ubus.sock`，接着其作为参数传入`usock`函数中，当`usock`函数的返回值错误时，就会走到`perror("usock")`报错。

![](https://bbs.kanxue.com/upload/attach/202406/949925_2YPD4JARDZDYHM4.png)

显然我们缺少`/var/run/ubus.sock`这个文件，创建后`ubusd`即可正常启动，接着`sysapihttpd`服务也启动成功（这里缺失`/dev/nvram`芯片可以先不用管，因为后续没有用到相关操作，暂时不用`hook`）：

![](https://bbs.kanxue.com/upload/attach/202406/949925_KXVJ4KYGR8G8REB.png)

到这里，检查一下进程里相关程序都已经挂起，且`netstat`查看`web`端口也正常对外开放：

![](https://bbs.kanxue.com/upload/attach/202406/949925_HXN6CRPKUMT9T8M.png)

![](https://bbs.kanxue.com/upload/attach/202406/949925_76HQ3KR4VG66C4D.png)

崩溃 & 排查
=======

虽然这一切看上去没有任何问题，`nmap`也能扫到端口，但是我们访问`IP`就会报错，显示服务器没有传回任何数据。

![](https://bbs.kanxue.com/upload/attach/202406/949925_QHYY3BYKJBRG6BH.png)

此外，查看进程，可以发现`sysapihttpd`的主进程进程号没变，但是子进程的进程号却已经改变，这就说明当我们访问`IP`的时候，`server`崩溃了，所以子进程`crash`了，主进程作为守护进程又开了个新的子进程。

![](https://bbs.kanxue.com/upload/attach/202406/949925_CXQZ43EMZEZ873W.png)

因此，我们可以用`strace`跟踪一下子进程的进程号，判断出现了什么问题导致`sysapihttpd`会`crash`的。这里直接在外面的`qemu`文件系统中用`apt`或者`dpkg`安装一下`strace_arm64`的`deb`包，然后再`ssh`开个窗口跟踪进程即可。先`attach`上去卡住，然后访问`IP`后，`strace`跟踪的结果如下：

![](https://bbs.kanxue.com/upload/attach/202406/949925_T2CU454MCDVDNBV.png)

可以看到最后是发生了段错误导致了`crash`的。我们需要解决的是上面的三个报错，前两个错误都是由于缺少`/etc/TZ`文件所导致的，这是一个和时区有关的配置文件，`echo "WAUST-8WAUDT" > /etc/TZ`创建一下即可。

![](https://bbs.kanxue.com/upload/attach/202406/949925_BNW96VJC949GYY9.png)

对于后面`getsockopt`的报错，我们可以将`/usr/sbin/sysapihttpd`拖进`IDA`，定位一下`getsockopt`参数一致（主要关注第三个参数是`0x50`）的位置，可以找到`sub_1EDAC`和`sub_27570`两个函数。至于是哪个函数中`getsockopt`调用的位置才是关键，结合`strace`上下文，根据上面调用了`accept4`系统调用，可以确定`sub_27570`函数中`getsockopt`的位置才是调用点。

![](https://bbs.kanxue.com/upload/attach/202406/949925_5BA6PCMYWACH9TH.png)

根据上面的代码，结合`strace`后面输出的`log`报错信息，可以得知这里是由于`getsockopt`发生了错误，从而重定向到了`0.0.0.1:65535`，导致了后续的崩溃。因此，最简单粗暴的办法就是绕过这个`if`分支，不执行其中的内容，即将`CBZ`给`patch`成相反的`CBNZ`即可：

![](https://bbs.kanxue.com/upload/attach/202406/949925_X5874JFN67DMFSA.png)

将`patch`后的`sysapihttpd`程序按照如下的步骤更新至固件的文件系统中，并重启`httpd`服务：

![](https://bbs.kanxue.com/upload/attach/202406/949925_RZ5Z2QS35W6HHJX.png)

然后，访问`IP`并重定向至`/init.html`，可以看到路由器初始化配置页面终于是成功出现了：

![](https://bbs.kanxue.com/upload/attach/202406/949925_GB442257FRQGS48.png)

跳过初始化配置
=======

然而，由于我们只启动了部分服务，以及有部分配置与硬件相关，所以毫无疑问按照路由器初始化配置页面中的流程进行配置的过程中一定会出现各种问题。当然，我们仿真模拟的环境并不需要完整的配置，只是为了验证或调试漏洞而已，所以我们也没必要折腾这部分，直接跳过就行了。

我们通过`grep`在固件文件系统中查找是在哪里重定向至`/init.html`的，很容易定位到`/usr/lib/lua/luci/view/web/sysauth.htm`文件：

![](https://bbs.kanxue.com/upload/attach/202406/949925_BVX4372XCNUTM47.png)

其中，相关代码如下：

![](https://bbs.kanxue.com/upload/attach/202406/949925_C7VFDJ6BPQMATWF.png)

这里最简单粗暴的办法当然就是将这几行都注释掉，这样虽然的确可以跳过初始化配置，但是在之后漏洞验证的时候会出点小问题，留到后文再说。

我们这里还是更优雅地想办法去满足这个`if`判断的条件，使之不重定向至`/init.html`。根据这里调用的函数，很容易定位到如下代码的调用关系（这里的源码来自于[小米路由器 4 Pro 稳定版](https://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/r1350/miwifi_r1350_firmware_7449d_1.0.31.bin)，这个版本的`Lua`源码没有编译，小米各个型号路由器的`Lua`部分变化不大，源码当然比反编译出来的伪代码好看很多）：

![](https://bbs.kanxue.com/upload/attach/202406/949925_R5NS4JS7ZDUMZG6.png)

很容易弄清这里的逻辑，只需要按照如下命令设置`uci`配置项，标记为已初始化即可：

```
uci set xiaoqiang.common.INITTED=1
uci commit

```

![](https://bbs.kanxue.com/upload/attach/202406/949925_FPKHMCANATDVQAT.png)

`uci set`设置后，再次访问`IP`，即可绕过初始化配置，跳转至登录页面：

![](https://bbs.kanxue.com/upload/attach/202406/949925_4CVGR7G9A8YQCWV.png)

设置登录密码
======

由于我们跳过了初始化配置阶段，并没有设置路由器后台的登录密码，且我们需要验证的`CVE-2023-26315`是一个授权认证后的漏洞，因此我们需要设置登录密码以登录进后台拿到`token`的值（当然，上文说过有些型号的设备固件中`Lua`是没有编译的源码，因此也可以拿这部分源码把相关`API`改成未授权的，然后用固件里自带的`/usr/bin/luac`编译一下替换即可，但这样确实太暴力了）。

上篇文章提过，身份校验的过程在`/usr/lib/lua/luci/dispatcher.lua`的`jsonauth`函数中，其中调用了`checkUser`函数根据从`POST`报文中获取的`username`，`password`和`nonce`（现时）字段进行身份验证。

![](https://bbs.kanxue.com/upload/attach/202406/949925_VE6DZFJPWRT2RKN.png)

在`/usr/lib/lua/xiaoqiang/util/XQSecureUtil.lua`的`checkUser`函数中，首先获取了系统`uci`配置项中存储的密码，这里的`XQPreference.get`函数在本文的上一节中已经给出，可分析出此处的配置项为`account.common.(用户名)`。接着，需要`POST`报文中传入的现时字段`nonce`与系统中`uci`存储的`password`的值拼接后进行`sha1`哈希的结果等于`POST`报文中传入的密码字段。

![](https://bbs.kanxue.com/upload/attach/202406/949925_MJ9SNDX78XJ766U.png)

因此，接下来，我们需要确定`POST`报文中传入的密码和用户名字段是什么。很显然，`POST`请求报文中的密码字段不可能是明文的形式，不然随便拦截一下就寄了。故而，一定会有相关的`JavaScript`代码对用户提交的密码进行加密（哈希）后再进行传输。所以，可以直接在浏览器登录页面中，查看一下相关的`web`代码：

![](https://bbs.kanxue.com/upload/attach/202406/949925_J2BKT36T3H6ZMZC.png)

![](https://bbs.kanxue.com/upload/attach/202406/949925_N48CNF5KJ3GS3VY.png)

可以看出，报文中的用户名字段固定就是`admin`，而密码字段是通过`oldPwd()`函数加密后的结果。这里的`oldPwd()`函数将用户提交的密码明文与一个固定的`key`值（`a2ffa5c9be07488bbb04a3a47d3c5f6a`）拼接后，进行`sha1`哈希，再将结果继续与现时`nonce`拼接后，再`sha1`哈希一次，作为`POST`请求报文中的密码字段。

结合上述分析，我们需要将`account.common.admin`这个`uci`配置项设置为`sha1(登录密码+key)`，比如说登录密码设置为`winmt`，那么这个值就是`sha1(winmta2ffa5c9be07488bbb04a3a47d3c5f6a)=b264db0fca361ef8eca919fa28e70d7a57d4c2db`。

可通过如下命令`set`设置`uci`的配置项：

```
uci set account.common.admin=b264db0fca361ef8eca919fa28e70d7a57d4c2db
uci commit

```

![](https://bbs.kanxue.com/upload/attach/202406/949925_FV5U86R3EXNT947.png)

设置好`account.common.admin`这个`uci`配置项后，用设定的密码`winmt`即可成功登录入路由器后台：

![](https://bbs.kanxue.com/upload/attach/202406/949925_DYN2AD5E2VJ4ADK.png)

此时，`token`值为`789c292990be8ca7d36947145aaea700`。

一个小插曲
=====

在 “跳过初始化配置” 一节中，提到过：如果将`sysauth.htm`中重定向的`if`分支都注释掉的话，在后面进行漏洞验证的时候会出现一些问题。具体来说，是当访问`/api/xqdatacenter/request`这个`API`的时候，会出现`503`报错。

下面来分析一下这个问题的成因。

在`/usr/lib/lua/luci/controller/api/xqdatacenter.lua`中，对于`/api/xqdatacenter/request`这个`entry{}`没有设置第五个参数与权限有关的`flag`位：

```
entry({"api", "xqdatacenter", "request"}, call("tunnelRequest"), _(""), 301)

```

具体在`/usr/lib/lua/luci/dispatcher.lua`文件中，可以看到`entry{}`的相关定义：

```
--- Create a new dispatching node and define common parameters.
-- @param   path    Virtual path
-- @param   target  Target function to call when dispatched.
-- @param   title   Destination node title
-- @param   order   Destination node order value (optional)
-- @param   flag    For extension (optional)
-- @return          Dispatching tree node
function entry(path, target, title, order, flag)
    local c = node(unpack(path))
 
    c.target = target
    c.title  = title
    c.order  = order
    c.flag   = flag
    c.module = getfenv(2)._NAME
 
    return c
end

```

同样是在`/usr/lib/lua/luci/dispatcher.lua`文件中，其内的`dispatch`函数中对`API`作了最顶层的权限校验（`sysauth_authenticator`内的函数等各种模式是在其中被调用做进一步验证的）。在`dispatch`函数中，有一段如下代码：

```
if not _noinitAccessAllowed(track.flag) then
    luci.http.status(403, "Forbidden")
    return
end

```

其中，`_noinitAccessAllowed`函数的代码如下：

```
function _noinitAccessAllowed(flag)
    local xqsys = require("xiaoqiang.util.XQSysUtil")
    if xqsys.getInitInfo() then
        return true
    else
        if flag == nil then
            return false
        end
        if bit.band(flag, 0x08) == 0x08 then
            return true
        else
            return false
        end
    end
end

```

综上，也就是说只有`getInitInfo()`函数的返回值为`true`或`flag`位为`0x08`（或`& 0x08 == 0x08`），才能绕过这个`_noinitAccessAllowed`的校验，不然就会报`503`错误。

这里的`getInitInfo()`函数我们已经不陌生了，在 “跳过初始化配置” 一节中出现过，也就是`xiaoqiang.common.INITTED`这个`uci`配置项。这样也就解释了为什么在处理`sysauth.htm`中重定向的`if`分支时，还是最好设置一下相关`uci`配置项。当然，直接找一份`xqdatacenter.lua`的源码，在对应`entry{}`中加上`flag`位`0x08`，再用固件自带的`luac`编译后替换一下也是可以的。

验证漏洞
====

接下来，我们完成本次仿真模拟的最终目的：验证`CVE-2023-26315`这个漏洞。根据我[上篇文章](https://bbs.kanxue.com/thread-281901.htm)中的分析，想要完成此漏洞的验证，还需要启动`datacenter`以及`plugincenter`，命令如下：

```
/usr/sbin/datacenter &
/usr/sbin/plugincenter &

```

![](https://bbs.kanxue.com/upload/attach/202406/949925_EM4T74MSVXYQ697.png)

启动后，可能会有一些报错，但并不影响，`netstat`查看`9090`和`9091`端口正常即可：

![](https://bbs.kanxue.com/upload/attach/202406/949925_V7UT69Z4D29UWBG.png)

最后，跑一遍`exp`即可成功拿到`shell`，漏洞验证完成：

![](https://bbs.kanxue.com/upload/attach/202406/949925_AD2NABS9VAB4EQB.png)

完结撒花，**Are You OK**？

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工 作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 19 分钟前 被 winmt 编辑 ，原因： 修改格式

[#家用设备](forum-128-1-173.htm) [#固件分析](forum-128-1-170.htm) [#技术分享](forum-128-1-168.htm) [#安全研究](forum-128-1-167.htm)