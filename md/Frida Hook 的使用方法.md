> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247483740&idx=1&sn=25499c1320858095cb425aabe8acbd31&chksm=cebb4412f9cccd047f89b179437291473e646e07f01df2d2bec44fdb76441cdac04ab48f9181&scene=21#wechat_redirect)

**目录**

一、 Frida 支持的功能及作用 1

二、 安装和搭建 Python 环境 1

三、 安装和搭建 Frida 环境 2

   1. 安装 pip2

   2. 安装 frida2

   3. 验证 frida 是否安装成功 3

   4. 卸载 frida3

   5. 获取 frida 安装版本对应的 frida-server 版本 3

   6. 验证 frida 是否能正常使用 3

   7. Frida JavaScript API 的官网 4

   8. Ubuntu 环境修改系统默认 python 版本为 3.8.2（并更新）4

四、 Frida 的使用实例 4

   1. Hook Android Java 层 4

   2. Hook Android Native 层 5

   3. Hook Unity3d 获取解密后的 dll6

   4. Hook Cocos 获取解密后的 lua 源码 6

五、 Frida 通过 js 脚本实现 Hook 的常用命令 6

一、**Frida 支持的功能及作用**

 支持 Android/IOS 等多平台 Hook

二、**安装和搭建 Python 环境**  

1. 安装 Python 版本（比如: Python 2.7.5），安装最新版本的 Python 容易导致安装 Frida 时报错

2. 把 Python 的默认安装路径配置到系统环境变量（默认安装路径配置好后可以直接使用 pip 命令）

C:\python27-x64

C:\python27-x64\Scripts

3. 验证 Python 是否安装成功（**管理员权限执行如下命令**）

python --version

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2gGLIjwWAFxMG3RxeIBtnee2xGpYDTW3gCqI7bOic1YicmyQmeYot4ciacS7sBzL5OjlG7HmSqDvF6Q/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQpUOo4H9y6ukwEOiaPm8tfYnFckWldiamlsE6icFRUjxBENAa11KLrkia2Q/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQp0prhERKymBovR1BrxibOUaAl2oD7xIrzCK6MEzu5a5X2mnmBoeVkpg/640?wx_fmt=png)

三、**安装和搭建 Frida 环境**

1. 安装 pip

1) 最新 pip 的下载地址

https://pypi.python.org/pypi/pip

2) 解压安装

python setup.py install

2. 安装 frida

1) 最新 frida 源码或 frida-server 的下载地址

https://github.com/frida/frida/releases

2) 通过 pip 安装 frida（Windows 环境的首选安装方法，默认安装最新版本的 frida）

pip install numpy matplotlib

pip install frida

pip install --user frida

pip install frida-tools

pip install frida-tools --user

npm config set unsafe-perm true

npm install frida-compile -g

npm install frida-compile

3) 编译 frida 源码安装（适合于 Ubuntu 环境的安装，Windows 环境可能会导致编译出错）

make

make install

4) 安装 frida 过程 pip 会提示 PermissionError 的解决方法

pip install --user frida

3. 验证 frida 是否安装成功

frida --version

4. 卸载 frida

pip uninstall frida

5. 获取 frida 安装版本对应的 frida-server 版本

1) frida 和 frida-server 的版本号必须保持一致

2) 使用 root 过的 Android 手机或模拟器

3) frida-server 与 Android 手机或模拟器的架构必须保持一致（arm 32 或 arm 64 或 x86）

4) 把 frida-server 拷贝到 root 过的 Android 设备中

adb push frida-server /data/local/tmp/

5) 修改 frida-server 的权限

chmod 777 frida-server

6. 验证 frida 是否能正常使用

1) 手机端（执行 frida-server）

./frida-server

2) PC 端（转发 Android tcp 端口到本地）

adb forward tcp:27042 tcp:27042

adb forward tcp:27043 tcp:27043

3) PC 端（测试 frida 环境，如果出现 Android 设备的进程列表说明 frida 环境搭建成功）

frida-ps -R

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQuzLgHr0PlcRibfPahzbdbYv4ZfRuPEIQYV8Zd3o4YrdSPoibB60HmC8g/640?wx_fmt=png)

7. Frida JavaScript API 的官网

https://www.frida.re/docs/javascript-api/

8. Ubuntu 环境修改系统默认 python 版本为 3.8.2（并更新）

sudo apt-get update

sudo apt-get upgrade

sudo apt-get install python-pip

sudo apt-get install python3-pip

cd /usr/bin

rm python

ln -s python3.8.2 python

python pip install --upgrade pip

四、**Frida 的使用实例**

1. Hook Android Java 层

1) python3 Test.py 命令时注意查看 python 版本

2) 枚举 *** 游戏进程加载的所有模块以及模块中的导出函数

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQVwLibjJNxqDXrUjRSXfRNg6hEJN7juGGf9lFvU2mibPDjTutENr1eKiaw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQBKPzpMgt22pGfc52lq0wTJHKVSaricyDwibHo7ez6VYKibibWgqtWMyh3w/640?wx_fmt=png)

2. Hook Android Native 层

1) 获取跟踪调试 *** 游戏相关 cocos lua 源码

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQJ1XmpQlnrmNsIJ1C0ccG3oWufb2PkSPia1Ysu1BffGpg3qKesicicVgTg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF3Ct5ysxtxhafgee7eN9lIQYsiarXKu1icZOjUlCoQ2jooxJJJugM5jkC0Xy0GGq7J0QbWR1MPteaZA/640?wx_fmt=png)

2) 获取跟踪调试 *** 游戏相关 unity3d mono 引擎 dll 源码

import frida

import sys

rdev = frida.get_remote_device()

session = rdev.attach("cn.***.lof")

scr = """

Interceptor.attach(Module.findExportByName("libmono.so" , "mono_image_open_from_data_with_name"), {

onEnter: function(args) {

send("mono_image_open_from_data_with_name("+Memory.readCString(args[5])+","+Memory.readCString(args[0])+")");

},

onLeave:function(retval){

}

});

"""

script = session.create_script(scr)

def on_message(message ,data):

print(message)

script.on("message" , on_message)

script.load()

sys.stdin.read()

3. Hook Unity3d 获取解密后的 dll

4. Hook Cocos 获取解密后的 lua 源码

五、Frida 通过 js 脚本实现 Hook 的常用命令

frida -U -f 包名 --no-pause -l raptor_frida_android_trace_fixed.js -o 1.log

其中 "-f" 参数表示需要重启并且 attach 上 app

frida -U 包名 --no-pause -l raptor_frida_android_trace_fixed.js -o 2.log

只想 attach 到正在运行的应用程序的某一个进程可以用 "-p" 参数

先启动 app，然后执行命令，再按手机返回键，最后再点击 app 才可以 hook 成功

其中命令中加 "-l" 参数指定 js hook 代码，load 到目标进程

如果忘了使用 "-l" 参数，可以在交互窗口中用 "%load" 命令来指定需要加载的 js 代码

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2gGLIjwWAFxMG3RxeIBtnefEam4SbvXfgkicBw8Q53ouzgEa7DUfnMTO60TtIcDjaktfbQLjicHIug/640?wx_fmt=png)