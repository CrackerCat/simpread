> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247483908&idx=2&sn=2f43cbb7f845a21ad5d67024d00da84c&chksm=cebb474af9ccce5c1f0a25624bdb3952c1a52a67b74706e8f9d2f749d431cef19df51348b4ca&scene=21#wechat_redirect)

一、radare2 的安装更新 2  

二、Adhrit 安全分析 App（命令） 3

    (1)、搭建 Adhrit 的安全测试环境 3

    (2)、Adhrit 的使用方法（提取 App 的信息） 3

    (3)、Adhrit 安全分析 App（1.apk） 7

    (4)、Adhrit 安全分析 App（2.apk） 9

三、Adhrit 安全分析 App（Web） 16

一、radare2 的安装更新

    git clone https://github.com/radareorg/radare2.git

    cd radare2

    r2pm init

    r2pm update

    sys/install.sh

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4vIjVNPnEdgA3o2tZDyiaicjJmGE1IrEeRiaLlfabwm0zYP7Lj1omW0y2w/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4r3KDN4JBFNKuoeNjwPlofnShIQIoUznMICx08wleianKteetvt5YuGA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI42OJ3rLzunsqic4dhmbIPK1Z2ZWibmtFPWKd4h5qc9oPT9TnSKfN4bwAw/640?wx_fmt=png)

二、Adhrit 安全分析 App（命令）  

    (1)、搭建 Adhrit 的安全测试环境

            git clone https://github.com/abhi-r3v0/Adhrit.git

            cd Adhrit

            python3 installer.py

    (2)、Adhrit 的使用方法（提取 App 的信息）

            python3 adhrit.py -h

            python3 adhrit.py -a ****-1.0.4.apk

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4AlYafJrLJmx7JPVnD4iazxPibdRNaYICj7ic22S2ZuUDyibzaf5ibuRlHZg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4GsdacZWcgtdKCIrLuAhmlBTmaTU0zTdtjDQuicQlYBEx4Odc91pW0Ow/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4yUSIk9UHju4N11krjWia82ElFMlDicA8q9Op0poX3Immt8KbODUkR6nQ/640?wx_fmt=png)

    (3)、Adhrit 安全分析 App（1.apk）  

            python3 adhrit.py -pen com.***.rp.apk

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI496Q1JPK8AUheT1en7vvXcl9baRBnicBptwH7uPsMEQmGcBJJ0H7ib1mg/640?wx_fmt=png)

    (4)、Adhrit 安全分析 App（2.apk）  

            python3 adhrit.py -a com.*****.cn.apk

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4Am6RVwicNFkaMkibehey7K8URExeVia7sibHBiay2ORD8ZgBndHwicc0TVLQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4qnLjqxUI7Dyr3Lb6iaZvjfDWKeibs0ic8zo2m1XaUdUv9356YCfwVJUhw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4MecIhic1xU46k79bWI8xljOzNXF6y8sQp6yd3IJQSRPL33K5bQ30u8g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4ibgnc7hcwRbZtnAaVZlkwcga5QiaG34rF5tzzc3mr4nJyLKhE1OqHcuQ/640?wx_fmt=png)

            python3 adhrit.py -pen com.****.cn.apk  

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4B46az5dcQEZvoHf09BVNYfj4aGQfibOiaFl4WxS5PPwrC1MYUPXGktjQ/640?wx_fmt=png)

            python3 adhrit.py -a com.***.kig.apk  

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4yy3FicjPSP7Mib3EFRNicAhO2d3xQicR5qnYzB8HZHGLnFhJtQM5QjVLwQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4VSzgSKb4j843PUwq0lFjLaaCpNvb8lAXQcde1wALAHiaGjTAwEibCZPQ/640?wx_fmt=png)

三、Adhrit 安全分析 App（Web）  

    sudo apt install ng-common

    sudo npm install -g angular-cli

    python3 run.py

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4tsCJudczc7oOibubaicDY4IzIP8uh8hynglfZU7KLv4Awpubxde5Pqwg/640?wx_fmt=png)

    curl -X POST -F file=@app.apk http://localhost:5000/scan

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4nbDOibh2KMaaQkGel7icD8bLicTMFFuk0avmT4Faa6XUibQD52AJKAdgeA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4qSvMHBJb9PtxpR4iclnSSbbPDzgDZmibkicVbtqyAeiabSd05GIGH0OZkQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4zLVVNqddRRUJoicLRGPBra7k1J3m4gKeo1hlgj6SQZEjA9iawUR7eHJA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI42B8fWbXe0E7IWGmW2qwFIibTU3UpD1v11KkRQ3WTaiakNiccc3CXw4iavw/640?wx_fmt=png)

http://127.0.0.1:5000/  

http://localhost:5000/

http://localhost:4200/

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF1icjb0VdOWkVnAoWWcIHzI4JLyCsk93XVUsDrAf2E7YYb7cGFXymeGOJQYuUBUAW68VbhHPNkgBXw/640?wx_fmt=jpeg)

    您好，该公众号专注于打造服务于大众的哆啦社群 / 社区生活圈，陆续也会分享 Java、Android、C#、Qt、Web、前端 Nodejs/Python/JavaScript/lua/go、嵌入式、C/C++、Android 系统定制、一键新机、安全开发等软件开发相关的技术文章，也会分享二进制安全、区块链安全、IOT 安全、Web 安全、移动安全 Android/IOS / 小程序 / 游戏（逆向、加固、脱壳、混淆、Hook、反调试、反作弊、安全防护、安全风控、渗透、漏洞、病毒分析等安全研究相关的技术文章），以及广告变现、网站搭建、服务器环境搭建和配置、服务器安全配置方法及维护、开发类、安全类学习研究资料共享，提供给对软件开发 / 安全研究有兴趣的初学者 / 爱好者学习研究。

搭建的网站地址:

https://duolashq.com/

如果您需要远程支撑、商务合作可以联系作者，希望能与大家一起学习交流，感谢您的关注！