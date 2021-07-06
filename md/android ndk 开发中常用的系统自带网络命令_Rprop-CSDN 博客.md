> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/rrrfff/article/details/7364592)

android 本身内置了一些网络命令，使用这些命令程序尤其基于 ndk 开发时会获得很多便利，并在某种程度上可以绕开上层的限制、获得更多详细的信息和更好的灵活性等。

1. 操作路由表

> 获取：ip route 或者 busybox route
> 
> 新增：ip route add 10.0.0.172/8 dev wlan0
> 
> 删除：ip route del 10.0.0.172
> 
> 删除一条默认路由：ip route del default
> 
> 添加默认路由：route add default gw 10.0.0.172  

2. 测试网络

> ping 10.0.0.172
> 
> 测试 DNS：busybox nslookup blog.csdn.net
> 
> 测试 DNS：dsdnsutil -s 114.114.114.114 -q blog.csdn.net  
> 
> 测试路由路径：busybox traceroute 10.0.0.172

3. 查看接口

> netcfg
> 
> netcfg rmnet0  
> 
> busybox ifconfig (和 route 类似，android 自带的 ifconfig 默认不会打印出接口信息)

4. 操作接口

> netcfg wlan0 up/down
> 
> netcfg wlan0 dhcp up  
> 
> ifconfig rmnet0 10.0.0.172 up  
> 
> ifconfig rmnet0 10.0.0.172 netmask 255.0.0.0 up

5. 系统属性

> 打印全部属性：getprop
> 
> 设置 DNS：setprop net.dns1 128.224.160.11   // 此外 net.dns2 类似，不过在我的机子上好像无效，最终采用的 dns 是 dhcp.wlan0.dns1 指定的值  

6. 无线 wifi

> 打印无线拓展：iwconfig  
> 
> 使用 netcfg 打印接口, 在 wifi 没有打开的情况下是没有 wlan0 接口的, 因为 wifi 内核模块并未加载,
> 
> 加载 wifi 驱动模块：insmod /system/lib/modules/wlan.ko  (这时候 wlan0 就出来了，补充：后续可以参考这篇文章 [Android 下同时使用 WIFI 与 3G 网络](http://blog.csdn.net/roger__wong/article/details/8603275)。)
> 
> 卸载 wifi 驱动模块: rmmod /system/lib/modules/wlan.ko  (注意手动加载后没有卸载的话用户界面是开不了 wifi 的)
> 
> 扫描 wifi 热点：iwlist wlan0 scanning  

7.ndc 命令 (需要 ROOT 权限)

<table width="924" height="1534" cellspacing="0" cellpadding="0"><colgroup><col width="83"><col width="303"></colgroup><tbody><tr><td rowspan="6" width="83">interface</td><td width="303">list</td></tr><tr><td width="303">readrxcounter| readtxcounter</td></tr><tr><td width="303">getthrottle &lt;iface&gt; &lt;”rx|tx”&gt;</td></tr><tr><td width="303">setthrottle &lt;iface&gt; &lt;rx_kbps|tx_kbps&gt;</td></tr><tr><td width="303">driver &lt;iface&gt; &lt;cmd&gt; &lt;args&gt;</td></tr><tr><td width="303">route &lt;add|remove&gt; &lt;iface&gt; &lt;”default|secondary”&gt; &lt;dst&gt; &lt;prefix&gt; &lt;gateway&gt;</td></tr><tr><td width="83">list_ttys</td><td width="303">&nbsp;</td></tr><tr><td rowspan="2" width="83">ipfwd</td><td width="303">status</td></tr><tr><td width="303">enable|disable</td></tr><tr><td rowspan="7" width="83">tether</td><td width="303">status</td></tr><tr><td width="303">start-reverse|stop-reverse</td></tr><tr><td width="303">stop&lt;</td></tr><tr><td width="303">start&lt;addr_1 addr_2 addr_3 addr_4 [addr_2n]&gt;</td></tr><tr><td width="303">interface &lt;add | remove | list&gt;</td></tr><tr><td width="303">dnslist</td></tr><tr><td width="303">dnsset &lt;addr_1&gt; &lt; addr_2&gt;</td></tr><tr><td width="83">nat</td><td width="303">&lt;enable | disable&gt; &lt;iface&gt; &lt;extface&gt; &lt;addrcnt&gt; &lt;nated-ipaddr/prelength&gt;</td></tr><tr><td rowspan="2" width="83">pppd</td><td width="303">attach &lt;tty&gt; &lt;addr_local&gt; &lt;add_remote&gt; &lt;dns_1&gt; &lt;dns_2&gt;</td></tr><tr><td width="303">detach &lt;tty&gt;</td></tr><tr><td rowspan="5" width="83">softap</td><td width="303">startap | stopap</td></tr><tr><td width="303">fwreload &lt;iface&gt; &lt;AP|P2P&gt;</td></tr><tr><td width="303">clients</td></tr><tr><td width="303">status</td></tr><tr><td width="303">set &lt;iface&gt; &lt;SSID&gt; &lt;wpa-psk|wpa2-psk|open&gt; [&lt;key&gt;&lt;channel&gt; &lt;preamble&gt; &lt;max SCB&gt;]</td></tr><tr><td rowspan="4" width="83">resolver</td><td width="303">setdefaultif &lt;iface&gt;</td></tr><tr><td width="303">setifdns &lt;iface&gt; &lt;dns_1&gt; &lt;dns_2&gt;</td></tr><tr><td width="303">flushdefaultif &nbsp; 刷新 DNS 缓存</td></tr><tr><td width="303">flushif &lt;iface&gt;</td></tr><tr><td rowspan="17" width="83">bandwith</td><td width="303">enable | disable</td></tr><tr><td width="303">removequota | rq</td></tr><tr><td width="303">getquota | gq</td></tr><tr><td width="303">getiquota | giq &lt;iface&gt;</td></tr><tr><td width="303">setquota | sq &lt;bytes&gt; &lt;iface&gt;</td></tr><tr><td width="303">removequota | rqs &lt;iface&gt;</td></tr><tr><td width="303">removeiiquota | riq &lt;iface&gt;</td></tr><tr><td width="303">setiquota | sq &lt;interface&gt; &lt;bytes&gt;</td></tr><tr><td width="303">addnaughtyapps | ana &lt;appUid&gt;</td></tr><tr><td width="303">removenaughtyapps | rna &lt;appUid&gt;</td></tr><tr><td width="303">setgolbalalert | sga &lt;bytes&gt;</td></tr><tr><td width="303">debugsettetherglobalalert | dstga &lt;iface0&gt; &lt;iface1&gt;</td></tr><tr><td width="303">setsharedalert | ssa&lt;bytes&gt;</td></tr><tr><td width="303">removesharedalert | rsa</td></tr><tr><td width="303">setinterfacealert | sia &lt;iface&gt; &lt;bytes&gt;</td></tr><tr><td width="303">removeinterfacealert | ria &lt;iface&gt;</td></tr><tr><td width="303">gettetherstats | gts &lt;iface0&gt; &lt;iface1&gt;</td></tr><tr><td rowspan="2" width="83">idletimer</td><td width="303">enable|disable</td></tr><tr><td width="303">add|remove &lt;iface&gt; &lt;timeout&gt; &lt;classLabel&gt;</td></tr><tr><td rowspan="5" width="83">firewall</td><td width="303">enable|disable|is_enabled</td></tr><tr><td width="303">set_interface_rule &lt;rmnet0&gt; &lt;allow | deny&gt;</td></tr><tr><td width="303">set_egress_source_rule &lt;ip_addr&gt; &lt;allow | deny&gt;</td></tr><tr><td width="303">set_egress_dest_rule &lt;ip_addr&gt; &lt;port&gt; &lt;allow | deny&gt;</td></tr><tr><td width="303">set_uid_rule &lt;uid&gt; &lt;allow|deny&gt;</td></tr><tr><td width="83">clatd</td><td width="303">stop | status| start &lt;iface&gt;<p></p><p></p></td></tr></tbody></table>