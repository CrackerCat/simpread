> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1891483-1-1.html)

> [md]# 1. 前言业务中时常会遇到需要部分 APP 有严格风控如（地域 IP 限制、IP 访问频次限制等）。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)xinjun_ying _ 本帖最后由 xinjun_ying 于 2024-2-20 17:37 编辑_  

1. 前言
=====

业务中时常会遇到需要部分 APP 有严格风控如（地域 IP 限制、IP 访问频次限制等）。本次利用 redsocks + iptables 来实现安卓全局代 {过}{滤} 理切换。

2.redsocks 介绍
=============

redsocks 可将任何 TCP 连接重定向到 Socks4、Socks5 或 HTTPS (HTTP/CONNECT) 代 {过}{滤} 理服务器。

Socks5/HTTPS 连接支持登录 / 密码身份验证。Socks4 仅支持用户名，密码被忽略。对于 HTTPS，目前仅支持 Basic 和 Digest 方案。

3.iptables 介绍
=============

iptables 本质上是定义 linux 防火墙规则的工具，定义的规则，可以让在内核空间当中的 netfilter 来读取，并且实现让防火墙工作。

3.1 五处控制规则
----------

1.PREROUTING (路由前)  
2.INPUT (数据包流入口)  
3.FORWARD (转发管卡)  
4.OUTPUT(数据包出口)  
5.POSTROUTING（路由后）

3.2 数据流走向
---------

![](https://attach.52pojie.cn/forum/202402/20/172334bscw656egrwse686.png)

**2.png** _(208.65 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY3NjMwOHw4MGViN2EwZXwxNzA4NDgwOTY2fDIxMzQzMXwxODkxNDgz&nothumb=yes)

2024-2-20 17:23 上传

4. 环境准备
=======

```
# 测试使用版本 redsocks 0.4、iptables v1.4.20
# https://github.com/darkk/redsocks 直接拉取代码进行编译
apt-get install libevent-dev
cd redsocks
make

```

5. 设置代 {过}{滤} 理
===============

```
# 启动代{过}{滤}理并配置项，<IP> 代{过}{滤}理IP <PORT>端口
# proxy.sh 一共7个入参，详情查看脚本
/proxy.sh start http <IP> <PORT> false "" ""

# 对代{过}{滤}理ip不再代{过}{滤}理操作，<IP> 代{过}{滤}理IP
/iptables -t nat -A OUTPUT -p tcp -d <IP> -j RETURN

# 添加群控软件不走全局代{过}{滤}理规则，<UID>应用程序，如果你想让安卓上某个进程的网络都不走带可以可以这样设置
/iptables -t nat -m owner --uid-owner <UID> -A OUTPUT -p tcp  -j RETURN

# 对应域名不走代{过}{滤}理<xxx1> 不需要走代{过}{滤}理的域名
/iptables -t nat -A OUTPUT -p tcp -d <xxx1> -j RETURN
/iptables -t nat -A OUTPUT -p tcp -d <xxx2> -j RETURN

# 添加http代{过}{滤}理规则，可以按照自己的业务场景进行端口代{过}{滤}理设置
/iptables -t nat -A OUTPUT -p tcp --dport 80 -j REDIRECT --to 8123
/iptables -t nat -A OUTPUT -p tcp --dport 443 -j REDIRECT --to 8124
/iptables -t nat -A OUTPUT -p tcp --dport 5228 -j REDIRECT --to 8124

# 添加socks5代{过}{滤}理规则
/iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to 8123

```

6. 关闭代 {过}{滤} 理
===============

```
# 清除所有代{过}{滤}理配置
/iptables -t nat -F OUTPUT
# 清除配置并干掉进程，具体查看proxy.sh 脚本
/proxy.sh stop

```

7.proxy.sh 代 {过}{滤} 理脚本
=======================

```
#!/system/bin/sh

DIR=/data/user/0/com.netease.rcs/files
type=$2
host=$3
port=$4
auth=$5
user=$6
pass=$7

PATH=$DIR:$PATH

case $1 in
 start)

echo "
base {
 log_debug = off;
 log_info = off;
 log = stderr;
 daemon = on;
 redirector = iptables;
}
" >$DIR/redsocks.conf
proxy_port=8123

 case $type in
  http)
  proxy_port=8124
 case $auth in
  true)
  echo "
redsocks {
 local_ip = 127.0.0.1;
 local_port = 8123;
 ip = $host;
 port = $port;
 type = http-relay;
 login = \"$user\";
 password = \"$pass\";
}
redsocks {
 local_ip = 0.0.0.0;
 local_port = 8124;
 ip = $host;
 port = $port;
 type = http-connect;
 login = \"$user\";
 password = \"$pass\";
}
" >>$DIR/redsocks.conf
   ;;
   false)
   echo "
redsocks {
 local_ip = 127.0.0.1;
 local_port = 8123;
 ip = $host;
 port = $port;
 type = http-relay;
}
redsocks {
 local_ip = 0.0.0.0;
 local_port = 8124;
 ip = $host;
 port = $port;
 type = http-connect;
}
 " >>$DIR/redsocks.conf
   ;;
 esac
   ;;
  socks5)
   case $auth in
  true)
    echo "
redsocks {
 local_ip = 0.0.0.0;
 local_port = 8123;
 ip = $host;
 port = $port;
 type = socks5;
 login = \"$user\";
 password = \"$pass\";
 }
 " >>$DIR/redsocks.conf
   ;;
 false)
  echo "
redsocks {
 local_ip = 0.0.0.0;
 local_port = 8123;
 ip = $host;
 port = $port;
 type = socks5;
 }
 " >>$DIR/redsocks.conf
   ;;
 esac
 ;;
   socks4)
   case $auth in
  true)
    echo "
redsocks {
 local_ip = 0.0.0.0;
 local_port = 8123;
 ip = $host;
 port = $port;
 type = socks4;
 login = \"$user\";
 password = \"$pass\";
 }
 " >>$DIR/redsocks.conf
   ;;
 false)
  echo "
redsocks {
 local_ip = 0.0.0.0;
 local_port = 8123;
 ip = $host;
 port = $port;
 type = socks4;
 }
 " >>$DIR/redsocks.conf
   ;;
 esac
 ;;
 esac

 $DIR/redsocks -p $DIR/redsocks.pid -c $DIR/redsocks.conf

 ;;
stop)

  $DIR/busybox killall -9 redsocks
  $DIR/busybox killall -9 cntlm
  $DIR/busybox killall -9 stunnel
  $DIR/busybox killall -9 tproxy

  kill -9 `cat $DIR/redsocks.pid`

  rm $DIR/redsocks.pid

  rm $DIR/redsocks.conf
esac


```

8. 附件
=====

 ![](https://static.52pojie.cn/static/image/filetype/zip.gif) [data.zip](forum.php?mod=attachment&aid=MjY3NjMwOXw0NjRjYjFkNHwxNzA4NDgwOTY2fDIxMzQzMXwxODkxNDgz) _(168.55 KB, 下载次数: 27)_

2024-2-20 17:27 上传 点击文件名下载附件 下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/data/avatar/000/93/53/52_avatar_middle.jpg)xixicoco 这个可以破解外皮恩检测 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xyq3q 太厉害了，感谢分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) dujiu3611 ![](https://static.52pojie.cn/static/image/smiley/default/17.gif)  感谢分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) LCGPeace ![](https://static.52pojie.cn/static/image/smiley/default/48.gif)2 反反复复凤飞飞 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fuchensheng 感谢分享![](https://static.52pojie.cn/static/image/smiley/default/31.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) cruise666 感谢分享，学洗一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) ppplp 感谢分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) myconan 安卓没 root 应该用不了吧？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)janken 感谢分享！！