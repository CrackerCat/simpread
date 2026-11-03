> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293407.htm#msg_header_h2_11)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 目标：在 ** 办（梆梆企业版加固）正常联网的前提下，用 Burp 看到明文 HTTP 报文。
> 
> 真机环境：小米 9 / MIUI（Android 11）/ 自研内核 root（ShadowCore）  
> 电脑环境：Windows + Burp Suite Pro 2022.2.3  
> 最后验证：2026-11-03 10:50，App 首页 / 服务 / 我的全部正常，Burp 持续有流量

**系统代理关掉（检测天然过）→ iptables 把 App 的 443 在内核层改道到本地 8080 → adb reverse 送到电脑 Burp。**

两条都要命：

开系统代理后，启动即弹原生 Dialog：

```
安全警告
检测到系统代理开启，出于安全考虑请关闭代理后重新启动应用
[退出]                      4秒后自动关闭
```

当前 Activity 直接暴露检测模块：

```
com.hanweb.android.product.utils.proxy.ProxyAlertActivity
```

检测实现（hook 日志实锤）—— 逐个查 `System.getProperty`：

```
http.proxyHost    http.proxyPort
https.proxyHost   https.proxyPort
socksProxyHost    socksProxyPort
```

就算绕过检测，Burp 也一条流量都没有。`ss` 实锤 App 是直连：

```
jszwfw.activity  192.168.137.60:46580 -> 124.236.19.250:443
jszwfw.activity  192.168.137.60:42808 -> 124.236.19.250:443
jszwfw.activity  192.168.137.60:42822 -> 124.236.19.250:443
jszwfw.activity  192.168.137.60:46580 -> 123.183.232.33:443
jszwfw.activity  192.168.137.60:49918 ->  49.77.204.17:443
```

它走自己的 native 网络栈。所以**必须在更底层（iptables）动手**。

> 为什么之前 v0.5/v0.6 能抓到 URL？那是 WebView/Weex 加载静态资源  
> （index.js / 图片），走的是 Chromium 自己的代理逻辑，会读系统代理。  
> 真正的业务 API 一直是直连。

<table><thead><tr><th>项</th><th>要求</th><th>检查命令</th></tr></thead><tbody><tr><td>手机 root</td><td>可执行 <code>/data/local/tmp/.cch sh -c '...'</code></td><td><code>adb shell "/data/local/tmp/.cch sh -c 'id'"</code></td></tr><tr><td>adb</td><td>能连上设备</td><td><code>adb devices</code></td></tr><tr><td>Burp</td><td>监听 <code>127.0.0.1:8080</code></td><td><code>Get-NetTCPConnection -LocalPort 8080</code></td></tr><tr><td>openssl / curl</td><td>装 Burp CA 用</td><td><code>openssl version</code></td></tr><tr><td>python3</td><td>跑脚本</td><td><code>python --version</code></td></tr></tbody></table>

App 包名与 uid：

```
# 包名
com.hanweb.android.jszwfw.activity
# uid（实测 10264，换机/重装可能变，脚本会自动查）
adb shell dumpsys package com.hanweb.android.jszwfw.activity | grep userId=
```

Android 7+ 用户证书不再被 App 信任，必须装进 **system** 证书目录。  
做法是 bind mount（不 remount、不碰 dm-verity）。

```
curl -s -o certs/burp_ca.der http://127.0.0.1:8080/cert
```

```
cd certs
openssl x509 -inform DER -in burp_ca.der -out burp_ca.pem
H=$(openssl x509 -in burp_ca.pem -subject_hash_old -noout | head -1)
cp burp_ca.pem "$H.0"
echo "证书名: $H.0"
```

实测 Burp 2022.2.3 的 hash 是 `9a5ba575`（不同 Burp 版本 / CA 会变）。

```
adb push certs/9a5ba575.0 /data/local/tmp/ztca.0
adb push certs/cert_setup.sh /data/local/tmp/cert_setup.sh
adb shell "/data/local/tmp/.cch sh -c 'mkdir -p /data/adb; \
  cp /data/local/tmp/cert_setup.sh /data/adb/cert_setup.sh; \
  chmod 755 /data/adb/cert_setup.sh'"
adb shell "/data/local/tmp/.cch sh -c 'sh /data/adb/cert_setup.sh'"
```

`cert_setup.sh` 做的事：把原系统 139 张证书拷进 `/data/adb/cacerts`，  
再放入我们的 1 张 → `mount -o bind /data/adb/cacerts /system/etc/security/cacerts`。  
**必须拷全系统证书**，否则 App 里所有正常 HTTPS 全炸。

```
adb shell "/data/local/tmp/.cch sh -c 'ls /system/etc/security/cacerts | wc -l'"
# 期望 140（139 原厂 + 1 我们的）

adb shell "/data/local/tmp/.cch sh -c 'grep cacerts /proc/mounts'"
# 期望能看到 bind mount 行

# 指纹必须与本地 CA 一致
adb shell "/data/local/tmp/.cch sh -c 'cat /system/etc/security/cacerts/9a5ba575.0'" > /tmp/dev.pem
openssl x509 -in /tmp/dev.pem -noout -fingerprint -sha256
openssl x509 -in certs/burp_ca.pem -noout -fingerprint -sha256
```

```
python burp_proxy.py        # 下载 CA + 装证书 + reverse + 设代理（系统代理模式）
python burp_proxy.py --off  # 清理
python burp_proxy.py --check
```

> 注意：`burp_proxy.py` 走的是**系统代理**路线，只适合验证链路。  
> 真正抓苏服办要用第 4 节的透明代理。

```
adb reverse --remove-all
adb reverse tcp:8080 tcp:8080
adb reverse --list        # 应显示 "UsbFfs tcp:8080 tcp:8080"
```

为什么用 reverse 而不是同 Wi-Fi：

*   不需要管理员权限加防火墙入站规则
*   不受 Wi-Fi 隔离 / 网段影响
*   手机侧固定 `127.0.0.1:8080`，换网络不用改

验证手机侧确实有监听：

```
adb shell "/data/local/tmp/.cch sh -c 'ss -tlnp | grep 8080'"
# 期望：LISTEN ... *:8080 ... users:(("adbd",pid=...,fd=...))
```

```
adb shell "/data/local/tmp/.cch sh -c 'echo 1 > /proc/sys/net/ipv4/conf/all/route_localnet'"
adb shell "/data/local/tmp/.cch sh -c 'echo 1 > /proc/sys/net/ipv4/conf/lo/route_localnet'"
adb shell "/data/local/tmp/.cch sh -c 'echo 1 > /proc/sys/net/ipv4/conf/wlan0/route_localnet'"
```

**默认是 0。不开的话 REDIRECT 到 127.0.0.1 的包会被内核静默丢弃** ——  
规则看着加了、计数也在涨，但连接永远建立不起来。

```
UID=10264   # 用 dumpsys package 查到的真实 uid

adb shell "/data/local/tmp/.cch sh -c \
  'iptables -t nat -I OUTPUT 1 -p tcp -m owner --uid-owner $UID \
   --dport 443 -j REDIRECT --to-ports 8080'"

adb shell "/data/local/tmp/.cch sh -c \
  'iptables -t nat -I OUTPUT 1 -p tcp -m owner --uid-owner $UID \
   --dport 80 -j REDIRECT --to-ports 8080'"
```

用 `-I OUTPUT 1` 插到最前面，确保优先于系统的 `wmsctrl_nat_OUTPUT` 链。

```
python transparent.py on      # 查 uid + 开 route_localnet + 加规则
python transparent.py status  # 看规则/代理/route_localnet/监听
python transparent.py off     # 清规则
```

```
adb shell "/data/local/tmp/.cch sh -c 'settings put global http_proxy :0'"
adb shell "/data/local/tmp/.cch sh -c 'settings delete global http_proxy'"
adb shell settings get global http_proxy      # 期望 null
```

系统代理关着 → `ProxyAlertActivity` 的检测天然通过，**不需要任何 hook**。

**Proxy → Options → Proxy Listeners → 选中 `127.0.0.1:8080` → Edit  
→ Request handling 标签 → 勾选 "Support invisible proxying (enable only if needed)" → OK**

不开会怎样：Burp 把 TLS ClientHello 当成 HTTP 请求解析，一直等完整  
请求头 → 连接挂死 → 表现为 App 卡在闪屏、8080 上一堆 ESTABLISHED 不释放。

验证 Burp 是否已开（不需要手机）：

```
python - << 'EOF'
import socket, ssl
s = socket.create_connection(('127.0.0.1', 8080), timeout=6)
ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
try:
    ss = ctx.wrap_socket(s, server_hostname='example.com')
    print("OK 隐形代理已开, 证书长度", len(ss.getpeercert(binary_form=True)))
except Exception as e:
    print("未开:", type(e).__name__, e)
EOF
```

返回证书长度 → 已开；超时 → 没开。

```
adb shell am force-stop com.hanweb.android.jszwfw.activity
sleep 2
adb shell monkey -p com.hanweb.android.jszwfw.activity \
  -c android.intent.category.LAUNCHER 1
sleep 10
```

```
adb shell "/data/local/tmp/.cch sh -c 'iptables -t nat -L OUTPUT -v -n | grep REDIRECT'"
```

期望（pkts 会持续增长）：

```
pkts bytes target     prot opt in  out  source     destination
    5   300 REDIRECT   tcp  --  *   *    0.0.0.0/0  0.0.0.0/0  owner UID match 10264 tcp dpt:80  redir ports 8080
   55  3300 REDIRECT   tcp  --  *   *    0.0.0.0/0  0.0.0.0/0  owner UID match 10264 tcp dpt:443 redir ports 8080
```

```
Get-NetTCPConnection -LocalPort 8080 |
  Where-Object State -ne Listen |
  Group-Object State | Select-Object Name, Count
```

期望有 ESTABLISHED + TIME_WAIT。

*   **App 正常加载** → 链路全通（TLS 握手成功、证书被信任、双向通畅）
*   **App 卡闪屏** → Burp 没勾 invisible proxying，或证书没生效
*   **App 报网络错误** → SSL Pinning 挡了（见第 8 节）

直接操作 App（点登录、进服务、搜索），Burp 的 HTTP history 就会出现  
`ydzt.jszwfw.gov.cn` 的明文请求。

```
python transparent.py off
adb reverse --remove-all

# 保险起见再确认一次
adb shell "/data/local/tmp/.cch sh -c 'iptables -t nat -S OUTPUT'"
# 期望只剩系统的 wmsctrl_nat_OUTPUT
```

证书可以留着（bind mount 重启后就没了，不影响日常）。

<table><thead><tr><th>现象</th><th>原因</th><th>解法</th></tr></thead><tbody><tr><td>规则计数不动</td><td>uid 不对 / 规则位置靠后</td><td>重新查 uid；确认 <code>-I OUTPUT 1</code></td></tr><tr><td>计数在涨但 Burp 无连接</td><td>adb reverse 掉了</td><td><code>adb reverse --list</code>；手机侧 <code>ss -tlnp | grep 8080</code></td></tr><tr><td>连接建立但立刻断（CLOSE-WAIT）</td><td>route_localnet=0</td><td>按 4.2 开启</td></tr><tr><td>App 卡闪屏</td><td>Burp 没勾 invisible proxying</td><td>按第 5 节勾上</td></tr><tr><td>App 一开就弹 "安全警告"</td><td>系统代理没关干净</td><td><code>settings delete global http_proxy</code> 后确认 <code>null</code></td></tr><tr><td>App 报网络错误 / 证书错误</td><td>SSL Pinning</td><td>见下方</td></tr><tr><td>部分请求在 Burp 里显示握手失败</td><td>Pinning 做在 native 层</td><td>见下方</td></tr><tr><td>手机完全没网</td><td>iptables 规则没清</td><td><code>python transparent.py off</code></td></tr></tbody></table>

1.  先确认是不是 Java 层 pinning：Burp 里看该请求是 "握手失败" 还是 "完全没出现"
2.  Java 层：hook `X509TrustManager.checkServerTrusted` 直接返回
3.  native 层（梆梆企业版常见）：需要 native hook（ARM64 trampoline），  
    目前 zthook 还没做到这一层
4.  兜底：进程内 dump（下一节）

如果 pinning 实在绕不过，可以完全放弃 TLS 层，直接在 App 进程里  
拿加解密后的明文：

*   hook `java.net.HttpURLConnection.getInputStream()` / `getOutputStream()`
*   hook App 自己的网络封装类
*   或者利用已抓到的密钥（第 10 节）直接解应用层密文

优点：确定性最高，pinning 完全不影响。  
缺点：看不到原始 HTTP 报文格式（Header、状态码、Cookie 顺序）。

即使 TLS 通了，App 的部分业务数据还有**第二层 AES 加密**。  
下面这些是 v0.5/v0.6 通过 hook `SecretKeySpec` / `Cipher` 拿到的：

<table><thead><tr><th>密钥</th><th>值</th><th>算法</th><th>用途</th></tr></thead><tbody><tr><td>KEY-1</td><td><code>8765432112345678</code></td><td>AES/CBC/PKCS7Padding</td><td>埋点数据加解密</td></tr><tr><td>IV</td><td><code>0a010b05040f070917030106080c0d5b</code></td><td>—</td><td>固定硬编码</td></tr><tr><td>KEY-2</td><td><code>abcdfe0987612345</code></td><td>AES/ECB</td><td>Java 序列化配置</td></tr></tbody></table>

解密示例（Python）：

```
from Crypto.Cipher import AES
key = b'8765432112345678'
iv  = bytes.fromhex('0a010b05040f070917030106080c0d5b')
pt  = AES.new(key, AES.MODE_CBC, iv).decrypt(cipher_bytes)
```

实测明文（埋点上报）：

```
[{"page_name":"com.hanweb.android.jszwfw.activity.com.hanweb.android.product.appproject.main.activity.MainActivity",
  "duration":93385,"page_start":1793669055167,"type":0}]
```

KEY-2 解出来是 `java.util.HashMap` 的序列化字节（`sr java.util...loadFactor...threshold`）。

<table><thead><tr><th>文件</th><th>作用</th></tr></thead><tbody><tr><td><code>transparent.py</code></td><td>透明重定向开关（推荐日常用）</td></tr><tr><td><code>burp_proxy.py</code></td><td>CA 下载 + 装证书 + reverse + 系统代理（链路验证用）</td></tr><tr><td><code>certs/cert_setup.sh</code></td><td>设备端证书安装（bind mount，幂等）</td></tr><tr><td><code>certs/install_cert.sh</code></td><td>电脑端一键装证书</td></tr><tr><td><code>src/zthook7.c</code></td><td>代理检测 hook（系统代理模式下骗过 ProxyAlertActivity）</td></tr><tr><td><code>src/zthook5.c</code> / <code>zthook6.c</code></td><td>网络侦察（URL / Cipher / 密钥 / IV / 算法）</td></tr><tr><td><code>04_文档/PROGRESS.md</code></td><td>完整进度与踩坑记录（坑 14-17）</td></tr></tbody></table>

[[招生] 科锐逆向工程师培训 (2026 年 11 月 13 日实地，远程教学同时开班, 第 57 期)](https://bbs.kanxue.com/thread-51839.htm)