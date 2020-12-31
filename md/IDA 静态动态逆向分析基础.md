> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247483812&idx=1&sn=9300f66fa60946fb45858b9b523fe82f&chksm=cebb44eaf9cccdfc74395a06bd9d48d4869e67d61fad78bd2fb833e7e83bb5e213681f12b814&scene=21#wechat_redirect)

1.JDB 和 IDA 调试步骤 2

  (1).JDB 调试步骤 2

  (2).IDA 调试步骤 2

  (3). 定位函数 2

  (4). 开始调试 2

2. 动态调试 Android so 库函数的方法 2

3.strace 查看系统调用 3

4.IDA 静态动态分析的快捷键 3

5.gdb 调试 4

  (1). 准备调试 4

  (2). 修改权限 5

  (3). 端口转发 5

  (4).ps 命令查询 PID5

  (5).gdbserver attach 调试进程 5

  (6). 运行 gdb.exe5

6. 常见工具 5

  (1). 命令工具 5

7.OAT/DEX/ELF 的文件格式

8. 在线加解密 5

  (1). 二维码解码器 5

  (2). 在线 JSON 校验格式化工具 5

  (3). 在线解密 6

**1.JDB 和 IDA 调试步骤**

**(1).JDB 调试步骤**

①adb shell am start -D -n {pkgname}/{Activity}

②ida android_server 放到手机，以 root 身份运行，PC 端 idapro 远程连接、 attach、下断点

③adb forward tcp:8700 jdwp:{pid}

④jdb -connect “com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700”

**(2).IDA 调试步骤**

①adb shell am start -D -n {pkgname}/{Activity}

②adb forward tcp:23946 tcp:23946

③ida android_server 放到手机，以 root 身份运行，PC 端 idapro 远程连接、 attach、下断点

④打开 DDMS, 查看端口

⑤jdb -connect “com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700”

**(3). 定位函数**

①使用快捷键 Ctrl+S 打开 segment 窗口，选择 so 文件，这里可以使用 Ctrl+F 进行搜索；同时这里需要记下 so 文件的起始地址 (A8924000) 用另一个 IDA 打开 so 文件，找到对应函数的偏移位置，在上面的图可以看到偏移为(00000E9C)

绝对地址 = 基址 + 偏移地址 = A8924000+00000E9C=A8924E9C

②按下快捷键 G，输入 A8924E9C 即可跳转到正确的函数，然后使用 F2 或者点击前面的小圆点下一个断点  

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDTQrVm4YnramOiajqJ9jZnRKhlwm59DqeRH1yae6kQFibN6O679wVMz7g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDSEwNxtWgjHrSOLnmxWtJibiaa75HiaJVSiaD9Zgl35nDlDw4qWuwwcOdxw/640?wx_fmt=png)  

**(4). 开始调试**

①按 F9 或点击绿色三角形按钮运行程序，触发断点，接着按 F7/F8 进行调试

2. **动态调试 Android so 库函数的方法**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDZysD58uuq409rSUdVruuTF3CjhHaKtibznl2ics7gquPzLaH2RibZewtw/640?wx_fmt=png)

./android_server p2345

p 端口号

adb shell ps | grep 包名

netstat -ano | findstr "8900"

tasklist | findstr "1220"

可以通过命令强制杀死 pid 为 2816 的进程

taskkill /f /pid 1220

adb forward tcp:23946 tcp:23946

adb forward tcp:8700 jdwp:25631

jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700

**3.strace 查看系统调用**

strace -f -p PID -o file.txt

**4.IDA 静态动态分析的快捷键**

(1).F2 下断点

(2).F7，f8 单步步入

(3).N 重名

(4).G 跳到地址和函数名

(5).U 取消把函数汇编变成机器码

(6).C 就是把机器码变成汇编

(7).F5---> 刷新

(8).P 分析函数，把机器码那些东西翻译成函

(9).Ctrl+S 看见系统所有的模块

(10).Ctrl+F 搜索

(11). 单步调试注意右上角，寄存器变蓝色表示被改了

(12).otions->number of opcode bytes 可以查看机器码，填入 4 一行看 4 个机器码

(13). 在 hex view-1 按 F2 可以修改机器码，再次按 F2 确定修改

(14).Alt+G 看是 THUMB 还是 ARM 指令（寄存器 ARM 与 THUMB 转换）

(15). 在函数名上按 X 可以看见上层调用

(16). 在 F5 伪 C/C++ 代码的情况下，注释是 /，汇编情况下注释是;

(17).F4 移动到光标处

(18). 在寄存器窗口按 E 可以修改寄存器的值

(19). 在内存窗口 F2 可以修改内存的值

(20). 搜索特征字符串, 具体操作为

①快捷键 Ctrl+S，打开搜索类型选择对话框 --> 双击 Strings，跳到字符串段 --> 菜单项 “Search-->Text”

②快捷键 Alt+T，打开文本搜索对话框，在 String 文本框中输入要搜索的字符串点击 OK 即可

(21).Tab 查看函数

(22).Ctrl+1，Spance

(23).Alt+Q

(24).Alt+S

(25).G - 转到一个 Jump Address

(26).Create function--->P

(27).Data--->D

(28).X---> 追踪某函数

(29).C---> 转成代码 --->N - 重命名函数，多看汇编，少看伪代码

(30).H--->16 进制转换

(31).A---> 生成一个字符串

(32).U---> 转成原数据

(33).*---> 重新定义数组长度

**5.gdb 调试**

**(1). 准备调试**

android-ndk-r10e\prebuilt\android-arm\gdbserver

android-ndk-r10e\prebuilt\android-arm64\gdbserver

手机或模拟机已经 root

adb remount

adb push gdbserver /system/bin

android-ndk-r10e\toolchains\arm-linux-androideabi-4.9\prebuilt\windows-x86_64\bin\arm-linux-androideabi-gdb.exe

android-ndk-r10e\toolchains\aarch64-linux-android-4.9\prebuilt\windows-x86_64\bin\aarch64-linux-android-gdb.exe

arm-linux-androideabi-gdb.exe 修改为 gdb.exe

**(2). 修改权限**

adb shell chmod 777 /system/bin/gdbserver

**(3). 端口转发**

adb forward tcp:23946 tcp:23946

**(4).ps 命令查询 PID**

adb shell ps | grep 包名

**(5).gdbserver** **attach** **调试进程**

adb shell gdbserver :23946 –attach [PID]

**(6). 运行 gdb.exe**

显示汇编代码

set disassemble-next on

打开单步调试

set step-mode on

在 gdb 界面输入如下命令，连接上 gdbserver

target remote 127.0.0.1:23946

**6. 常见工具**

 **(1). 命令工具**

tmux: 可以关闭窗口将程序放在后台运行

jnettop: 监测网络流量, 得到通讯 IP、端口、URL、速率信息

netstat -tunlp：端口对应进程号、监听、收发包端口

htop: top 的增强版, 当前系统负载、前台活跃进程、线程和占用

apt install tmux jnettop htop

ps -e |grep -i termux

入门 Android 逆向

https://mp.weixin.qq.com/s/K1tvKOzDRwiO8uDON4u_MA

**7.**OAT**/DEX/ELF 的文件格式**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDxgZOTax5Csa4Xsib4VfTI47axHlJC3Plu35XNfhAWkvgV7cw0QvokOg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDrOy7td2EYrS2XXogwPO6YP1kZKSMJdZ15Qerkbh4gic2G5D4DmRzoRw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDicXmPc4UciaFmvRbHqDkiblib9WNmXiamCAsD0CpicLHNyUVb9Tq7m8rXeFw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDnur2AAjBrIrsHP5ticdbqdogslcMfRofYkdKws9hqWXoSeiavz65ra5Q/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDNGGrkm0J3vribpB5vicM6c9uoIcSHTicSLst41f1OFuIvp9ia3GLshB32Q/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWD6FddrGnvb3f6vMNbvdAOKbe3yibMRia6g3wzTOaXCdXdkUyjvHYpdLNg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDK1OZPD9V217RQ34t90ibOwRzaZPsgKDllicE42xvR3q0PDtMhMSBRYDQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDGzwSoVJMsgibsGzzCQEFyOsF6J7APnIoQKb0foSH5FicOj4Txud437fQ/640?wx_fmt=png)

**8. 在线加解密**

(1). **二维码解码器**

  https://cli.im/deqr/

(2). **在线 JSON 校验格式化工具**   

  http://www.bejson.com/

  http://www.jsons.cn/

(3). **在线解密**

  https://www.ssleye.com/

  http://www.ttmd5.com/rang.php

  https://cmd5.com/

  https://jwt.io/

  https://the-x.cn/cryptography/Aes.aspx

  http://tool.chacuo.net/cryptdes

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDq4WjibDxsVHCfZtqjQZoZhXNytAB7MUk5VPzfeJ5vicWkHMVLs54jPGg/640?wx_fmt=jpeg)

您好，该公众号专注于打造服务于大众的哆啦社群 / 社区生活圈，陆续也会分享 Java、Android、C#、Qt、Web、前端 Nodejs/Python/JavaScript/lua/go、嵌入式、C/C++、Android 系统定制、一键新机、安全开发等软件开发相关的技术文章，也会分享二进制安全、区块链安全、IOT 安全、Web 安全、移动安全 Android/IOS / 小程序 / 游戏（逆向、加固、脱壳、混淆、Hook、反调试、反作弊、安全防护、安全风控、渗透、漏洞、病毒分析等安全研究相关的技术文章），以及广告变现、网站搭建、服务器环境搭建和配置、服务器安全配置方法及维护、开发类、安全类学习研究资料共享，提供给对软件开发 / 安全研究有兴趣的初学者 / 爱好者学习研究。

搭建的网站地址:

https://duolashq.com/

如果您需要远程支撑、商务合作可以联系作者，希望能与大家一起学习交流，感谢您的关注！

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF12lmYCSXjLr54fxmLVepWDp9N1f2wSuleE2O0ymfaldkBKb1iaJR7kibPUkMmn94YnrIKeA13mkg7A/640?wx_fmt=png)