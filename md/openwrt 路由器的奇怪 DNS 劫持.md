> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [green-m.me](https://green-m.me//2022/11/01/openwrt-dns-hijack/)

> pentest; 渗透测试; Green-m;Metasploit;Security

### 0x00 前言

* * *

最近宽带升级了，从 300M 电信小水管直接升级到了 1000M 联通，这个升级还是挺大的，我的老网络架构直接承担不起这个大带宽了，所以决定对网络进行改造。

以前的网络是使用 x86 架构的 openwrt 软路由作为旁路由，开启 openclash 之类的东西方便浏览资料，主路由使用好几年前买的 K2P 拨号上网，并承担全屋的 WI-FI 功能。这个架构在 300M 的时候，K2P 完全能跑满速度，升级后就最多只能跑到 600M，CPU 飙升到 100%，估计瓶颈应该就在 CPU 了。

正好联通师傅上门的时候给了我一个运营商定制的 WI-FI 6 路由器，这个路由器凭我的直觉是不能够作为主路由使用的（估计也不支持旁路由设置），现在只能通过 openwrt 拨号作为主路由。

### 0x01 现象

* * *

之前我的局域网中是有一个 NAS 搭建的 AdGuardHome 作为 DNS 服务器的，改造后发现所有的 DNS 请求都不走这个 AdGuardHome 了，不管是在 openwrt 中主动广播 DNS 地址，还是在 Client 里手动设置 DNS 服务器，DNS 流量都不会走到 AdGuardHome。

进一步排查发现，甚至使用 dig 命令指定任意不存在的 DNS 服务器，也有结果返回。 这个现象完全和 GFW 的 DNS 劫持立即返回一模一样。

如下：

```
dig baidu.com @22.22.33.33
<O> DiG 9.10.3-P4-Debian <O› baidu.com @22.22.33.33
global options: +cmd
Got answer:
->>-HEADER<<- opcode: QUERY, status: NOERROR, id: 46357
flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
OPT PSEUDOSECTION:
EDNS: version: 0, flags:; MBZ: 0001 udp: 4096
QUESTION SECTION:
;baidu.com.
IN
A
ANSWER SECTION:
baidu.com.
baidu.com.
H H
IN
IN
D D
110.242.68.66
39.156.66.10
Query time: 1 msec
SERVER: 22.22.33.33#53(22.22.33.33)
WHEN: Tue Nov 01 10:35:44 CST 2022
MSG SIZE
rovd: 70


```

### 0x02 排查

* * *

经过各种搜索和排查，最后在 iptables 中发现了端倪。

分别有两段涉及到 DNS 查询的 53 端口：

```
root@OpenWrt:~# iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 482 packets, 47519 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            8.8.4.4              /* OpenClash Google DNS Hijack */ tcp dpt:53 redir ports 7892
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            8.8.8.8              /* OpenClash Google DNS Hijack */ tcp dpt:53 redir ports 7892
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53 /* OpenClash DNS Hijack */ redir ports 53
   29  1989 REDIRECT   udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53 /* OpenClash DNS Hijack */ redir ports 53


root@OpenWrt:~# iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination
   89  7154 prerouting_rule  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* !fw3: Custom prerouting rule chain */
   87  6850 zone_lan_prerouting  all  --  br-lan *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    2   304 zone_wan_prerouting  all  --  pppoe-wan *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_wan_prerouting  all  --  eth1   *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_VPN_prerouting  all  --  ipsec0 *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_vpn_prerouting  all  --  tun0   *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
    0     0 zone_docker_prerouting  all  --  docker0 *       0.0.0.0/0            0.0.0.0/0            /* !fw3 */
   86  6790 REDIRECT   udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53 /* Rule For Control */ redir ports 53
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53 /* Rule For Control */ redir ports 53



```

这里有两段 PREROUTING 规则都劫持了 DNS 流量。

### 0x03 openclash 的 DNS 劫持

* * *

第一个是 openclash 的设置，如果勾选了`本地 DNS 劫持`的选项，就会加上这一条 iptables 的规则，取消勾选此选项即可。 但可能会影响 DNS 转发的功能，所以我就写了一个脚本手动修改 Dnsmasq 的转发。

```
#!/bin/sh /etc/rc.common
# file: /etc/init.d/dnswatcher

START=10
STOP=15

watchdir=/var/etc/
LOGFILE=/tmp/openclash_mylogger.log


start() {
    inotifywait -q -m "$watchdir"  -e delete,create |
        while read -r path action file; do
            #echo "The file '$file' appeared in directory '$path' via '$action'" >> $LOGFILE

            if  printf "%s" "$file" |grep -q "openclash.include" ; then
                sleep 5

                enable=$(uci get openclash.config.enable)

                # if enable ,modify dns
                if [ "$enable" -eq 1 ]; then
                    #echo "enabled"
                    echo "$(date)-----openclash dns not set right, changing it!" >> $LOGFILE
                    uci del dhcp.@dnsmasq[-1].server >/dev/null 2>&1
                    uci add_list dhcp.@dnsmasq[0].server="127.0.0.1#7874"
                    uci delete dhcp.@dnsmasq[0].resolvfile
                    uci set dhcp.@dnsmasq[0].noresolv=1
                    uci set dhcp.@dnsmasq[0].cachesize=0
                    uci commit dhcp
                    /etc/init.d/dnsmasq restart

                # else revert dns setting
                else
                    #echo "disabled"
                    echo "$(date)-----Reverting dns!" >> $LOGFILE
                    uci del dhcp.@dnsmasq[-1].server >/dev/null 2>&1
                    uci add_list dhcp.@dnsmasq[0].server="223.5.5.5#53"
                    uci set dhcp.@dnsmasq[0].resolvfile=/tmp/resolv.conf.auto >/dev/null 2>&1
                    uci set dhcp.@dnsmasq[0].noresolv=0
                    uci set dhcp.@dnsmasq[0].cachesize=0
                    uci commit dhcp
                    /etc/init.d/dnsmasq restart
                fi
            fi
    done &
    return 0
}

stop() {
     ps -ef | grep inotifywait | grep -v grep | awk {'print $1'} | xargs kill -9
     return 0
}


```

大概功能就是当开启 openclash 的时候，自动将 dnsmasq 的上游设置为 127.0.0.1:7874，如果关闭 openclash 的话，则恢复到正常的运营商 DNS 设置。

### 0x04 mia 的 DNS 劫持

* * *

这个 mia 的规则在 `/etc/init.d/mia`，对应的部分代码为：

```
start(){
  stop
        enable=$(uci get mia.@basic[0].enable)
        [ $enable -eq 0 ] && exit 0
  iptables -t filter -N MIA
  iptables -I INPUT -p udp --dport 53 -m comment --comment "Rule For Control" -j MIA
  iptables -I INPUT -p tcp --dport 53 -m comment --comment "Rule For Control" -j MIA
  iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53 -m comment --comment "Rule For Control"
  iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53 -m comment --comment "Rule For Control"
  strict=$(uci get mia.@basic[0].strict)
  [ $strict -eq 1 ] && iptables -t filter -I FORWARD -m comment --comment "Rule For Control" -j MIA
  add_rule
}


```

搜索后发现这个功能应该是家长控制的代码（真坑啊），直接去掉该功能即可。 如果还需要限制某些客户端上网的话，可以直接通过防火墙设置，如：

```
iptables -A FORWARD -i eth1 -o wan -m mac --mac-source 04:18:d6:52:1d:71 -m comment --comment "Reject A client" -j Reject


```

也可以直接在 web 界面进行防火墙设置，这里不再赘述。

### 0xff 结语

* * *

openwrt 中各种组件结合起来确实有各种坑，该问题困扰了我两三天，各种搜索无解，最后还是凭直觉在 iptables 中排查到了。要想高度自定义还是要做好踩坑的准备。

遗留问题： WI-FI6 使用 macbook 能跑满千兆，但 iphone 却跑不满，估计和运营商的路由器有关系。