> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247483908&idx=4&sn=141f287e66d79e0f947e0941d9668cfa&chksm=cebb474af9ccce5cb971d4083fd69372821c048c7e632146fb30f8f30930e52e183041f6c517&scene=21#wechat_redirect)

**1.AppMon 工作原理**

 AppMon 使用了多平台动态框架环境 Frida，Frida 是一款基于 Python + JavasSript 的 hook 框架，适应 android\ios\linux\win\osx 等平台的脚本交互环境。AppMon 还包括了一系列 app 事件监控和行为修改脚本，并能通过 web 接口显示和操作。

**2. 获取 appmon 源码**  

git clone https://github.com/dpnishant/appmon.git

**3.Windows 环境安装 Python2.7 的最新版本（如: Python 2.7.18），并配置 Python 环境** 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHPjLkwcVB2qACL60owBD2HCTp7NrRToeZAhh3R4tJVt8Gv3A252XrPQ/640?wx_fmt=png)

**4.Windows 环境管理员权限安装 frida**

pip install numpy matplotlib

pip install frida

pip install frida-tools 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHE852JsZeDsianWZzK3EuZZCGWTZr2Zd3nnrD3HZTv2rjkwtOPR5XBMA/640?wx_fmt=png)

**5. 使用 pip 安装 appmon 的依赖库**

pip install -r requirements.txt 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHB0yAtDaxLlI3W9buCU29fR3aQj8V3SeY6K3tafu1VAt1OGb5hEzhvg/640?wx_fmt=png)

**6. 下载 frida 安装版本匹配的 frida-server 版本，并启动 frida-server**

https://github.com/frida/frida/releases 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHaSWBaRnDkPHZ32Q4NFV4gCflhojudE1Yr1g0UQI5FCh3ktL3jLicSeQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHOTJxwZkonepx9PFfxRnCEe5WBialOick6qNDj7NMr8WcbKT6wqzYmhAQ/640?wx_fmt=png)

**7. 执行 python appmon.py -p android -s scripts/Android -a com.xm.cn 命令（其中 com.xm.cn 为 apk 的包名）** 

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHszMIGKJ50U6N8Ric2ibmrzXm6OON3Tv66gzgY4j4E9SGzDfp4iaRre1Ww/640?wx_fmt=png)

    通过 web 界面日志记录，可以看到具体的 HTTP 网络连接活动

**8. 浏览器访问 http://127.0.0.1:5000 / 链接，选择要分析的 Apk 包名，点击 “Next” 按钮**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHaGAlPp9BBgRqP71ZG4L87eubxza1cIITmiaFLfO32FXApx1u1y9DcEg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHlkmrD8n5nr3dxM91JvK9P8kRics41jT6YAoaFIntCice1QewDs2wFhkg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHcTiaZcd5kUGMZ23ofteTYHNE3BbzFwTceAI0Pr3340mIyXQKCPfkGpQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF30Of76bFpefVDL7bBrcEBHPptEcK1rrx8vaF9NZKHwG9ia2YurRYj5Kx1brd28IR37zxhnEwQr7Pw/640?wx_fmt=jpeg)

    您好，该公众号专注于打造服务于大众的哆啦社群 / 社区生活圈，陆续也会分享 Java、Android、C#、Qt、Web、前端 Nodejs/Python/JavaScript/lua/go、嵌入式、C/C++、Android 系统定制、一键新机、安全开发等软件开发相关的技术文章，也会分享二进制安全、区块链安全、IOT 安全、Web 安全、移动安全 Android/IOS / 小程序 / 游戏（逆向、加固、脱壳、混淆、Hook、反调试、反作弊、安全防护、安全风控、渗透、漏洞、病毒分析等安全研究相关的技术文章），以及广告变现、网站搭建、服务器环境搭建和配置、服务器安全配置方法及维护、开发类、安全类学习研究资料共享，提供给对软件开发 / 安全研究有兴趣的初学者 / 爱好者学习研究。

搭建的网站地址:

https://duolashq.com/

如果您需要远程支撑、商务合作可以联系作者，希望能与大家一起学习交流，感谢您的关注！