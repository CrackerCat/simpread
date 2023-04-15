> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-274396.htm#msg_header_h1_2)

> [原创]Android - 系统级源码调试

[原创]Android - 系统级源码调试

2022-9-14 10:48 5485

[举报](javascript:void(0);)

### [原创]Android - 系统级源码调试

 [![](http://passport.kanxue.com/upload/avatar/140/853140.png?1666448184)](user-home-853140.htm) [iyue_t](user-home-853140.htm) ![](https://bbs.kanxue.com/view/img/rank/6.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 2022-9-14 10:48  5485

Android - 系统源码调试
================

*   理论上 只要有编译好的 idegen 三个操作系统都能跑

目录

*   Android - 系统源码调试
*   Android Java 层源码调试
*            [编译 idegen](#编译idegen)
*            [编辑导入配置](#编辑导入配置)
*            导入 Android Studio
*            排除 tests 目录 右键
*            配置 Android 源码项目
*                    点击 File -> Project Structure–>SDKs 配置项目的 JDK、SDK。
*                    根据源码版本选择对应 API 级别 这里使用的 Android10 对应 29
*                    点击 Edit API SDK
*                    Modules Structure
*                    配置 Run/Debug Configurations
*            开始调试 - 使用教程
*                    [常用命令](#常用命令)
*                    [案例一](#案例一)
*            第三方 app 触发 系统 java 层 调试成功
*   Android Native 层源码调试 - 需以源码编译的模拟器启动
*            前置配置 source build/envsetup.sh lunch 22
*            配置 VScode 运行和调试
*                    获取 vscodelunch.json 配置
*            [启动 native 调试](#启动native调试)
*            最后 打开 AS open Android ipr 项目
*            vscode 附加调试失败
*                    [手动运行 gdbserver](#手动运行gdbserver)
*   [异常问题处理](#异常问题处理)

Android Java 层源码调试
==================

编译 idegen
---------

*   成功会在源码根目录生成`android.iml` 和 `android.ipr`两个文件

```
# 在源码根目录执行
source build/envsetup.sh
lunch 22
mmm development/tools/idegen
# or make idegen
sudo development/tools/idegen/idegen.sh
sudo chmod 777 android.iml
sudo chmod 777 android.ipr

```

编辑导入配置
------

`sudo deepin-editor android.iml`

*   搜索 excludeFolder，在下面加入这些配置
*   过滤不需要的源码模块

导入 Android Studio
-----------------

*   通过 AS 的`Open an existing Android Studio project`选项选择源码根目录的`android.ipr`就可以导入源码

![](https://bbs.kanxue.com/upload/attach/202209/853140_XBH6AFATJKH39ZX.png)

排除 tests 目录 右键
--------------

*   mark Directory as Excluded

配置 Android 源码项目
---------------

### 点击 File -> Project Structure–>SDKs 配置项目的 JDK、SDK。

![](https://bbs.kanxue.com/upload/attach/202209/853140_D9542MK6PH52BRN.png)

### 根据源码版本选择对应 API 级别 这里使用的 Android10 对应 29

<table><thead><tr><th>代号</th><th>版本</th><th>API 级别 / NDK 版本</th></tr></thead><tbody><tr><td>Android13</td><td>13</td><td>API 级别 33</td></tr><tr><td>Android12L</td><td>12</td><td>API 级别 32</td></tr><tr><td>Android12</td><td>12</td><td>API 级别 31</td></tr><tr><td>Android11</td><td>11</td><td>API 级别 30</td></tr><tr><td>Android10</td><td>10</td><td>API 级别 29</td></tr><tr><td>Pie</td><td>9</td><td>API 级别 28</td></tr><tr><td>Oreo</td><td>8.1.0</td><td>API 级别 27</td></tr><tr><td>Oreo</td><td>8.0.0</td><td>API 级别 26</td></tr><tr><td>Nougat</td><td>7.1</td><td>API 级别 25</td></tr><tr><td>Nougat</td><td>7.0</td><td>API 级别 24</td></tr><tr><td>Marshmallow</td><td>6.0</td><td>API 级别 23</td></tr></tbody></table>

### 点击 Edit API SDK

![](https://bbs.kanxue.com/upload/attach/202209/853140_HKR93YAGD3QFPRS.png)

*   不用管下面的配置选项

![](https://bbs.kanxue.com/upload/attach/202209/853140_BYKU4PPP7CC88NY.png)

### Modules Structure

```
/android/android/android-10.0.0_r2/frameworks/base/core/res/AndroidManifest.xml
/android/android/android-10.0.0_r2/frameworks/base/core/res/res
/android/android/android-10.0.0_r2/frameworks/base/core/res/assets

```

![](https://bbs.kanxue.com/upload/attach/202209/853140_FH25ZW7CKANVRAG.png)

### 配置 Run/Debug Configurations

![](https://bbs.kanxue.com/upload/attach/202209/853140_CV6UXGDMMBC22FS.png)

开始调试 - 使用教程
-----------

在源码的根目录创建 start_emulator.sh 脚本，为了方便的启动模拟器，输入以下内容 后执行

```
#!/bin/bash
source build/envsetup.sh
lunch 6
emulator
# sudo chmod 777 ./start_emulator.sh
# ./start_emulator.sh

```

### 常用命令

```
# 第三方app需先以调试模式启动app 点击运行
adb shell am set-debug-app -w com.example.dexlassloaders
# 等待附加调试 会自动继续运行 直到触发断点
 
# 系统进程可直接进行附加调试

```

### 案例一

在系统源码找到 ActivityStarter 这个类，在 startActivityMayWait 这个方法打断点

 

![](https://bbs.kanxue.com/upload/attach/202209/853140_N7ZWS5TVBRREZAA.png)

 

点击菜单的 Run–>Attach Debugger to Android Process，勾选 Show all processer，选择 system_process 随便启动 app 触发断点

 

![](https://bbs.kanxue.com/upload/attach/202209/853140_96XWY7ZDY4DAJJ4.png)

第三方 app 触发 系统 java 层 调试成功
-------------------------

![](https://bbs.kanxue.com/upload/attach/202209/853140_GPCUMQ3Z5Q4RXUR.png)

Android Native 层源码调试 - 需以源码编译的模拟器启动
===================================

前置配置 source build/envsetup.sh lunch 22
--------------------------------------

```
# 进入源码目录
cd /android/android/android-10.0.0_r2
# 先初始化环境 主要为lunch 目标
source build/envsetup.sh
lunch 22
# 进入gdbclient.py 脚本目录
cd development/scripts
# 调试模式启动 第三方app 此命令需手动点击
adb shell am set-debug-app -w com.example.dexlassloaders
# 以调试模式启动 无需手动点击
adb shell am start -D -n  com.example.dexlassloaders/.MainActivity
# 查看进程pid
adb shell "ps -ef | grep com.example.dexlassloaders"
# u0_a103       6018  1631 0 18:24:11 ?     00:00:00 com.example.dexlassloaders
# root          6046  1677 0 18:25:54 ?     00:00:00 sh -c ps -ef | grep com.example.dexlassloaders
# root          6049  6046 0 18:25:54 ?     00:00:00 grep com.example.dexlassloaders
# 执行此命令等待 输出 vscode launch.json配置 报错 请检查pid
gdbclient.py -p 6018 --setup-forwarding vscode
# 没调试完不要 按enter
# 接着使用as 附加调试 或者
adb forward tcp:12345 jdwp:6018  # (Where XXX is the PID of the debugged process.)
jdb -attach localhost:12345

```

配置 VScode 运行和调试
---------------

###  获取 vscodelunch.json 配置

*   注意先选择 C/C++ 源码 下好断点 此时按 F5 触发
*   `gdbclient.py -p 6018 --setup-forwarding vscode`输出下面内容 把其中 {} 复制到 `VScode launch.json`

```
{
    "configurations": [
         // {} 复制到这里
{
    "miDebuggerPath": "/android/android/android-10.0.0_r2/prebuilts/gdb/linux-x86/bin/gdb",
    "program": "/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/bin/app_process64",
    "setupCommands": [
        {
            "text": "-enable-pretty-printing",
            "description": "Enable pretty-printing for gdb",
            "ignoreFailures": true
        },
        {
            "text": "-environment-directory /android/android/android-10.0.0_r2",
            "description": "gdb command: dir",
            "ignoreFailures": false
        },
        {
            "text": "-gdb-set solib-search-path /android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/lib64/:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/lib64/hw:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/lib64/ssl/engines:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/lib64/drm:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/lib64/egl:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/system/lib64/soundfx:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/vendor/lib64/:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/vendor/lib64/hw:/android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols/vendor/lib64/egl",
            "description": "gdb command: set solib-search-path",
            "ignoreFailures": false
        },
        {
            "text": "-gdb-set solib-absolute-prefix /android/android/android-10.0.0_r2/out/target/product/generic_x86_64/symbols",
            "description": "gdb command: set solib-absolute-prefix",
            "ignoreFailures": false
        },
        {
            "text": "-interpreter-exec console \"source /android/android/android-10.0.0_r2/development/scripts/gdb/dalvik.gdb\"",
            "description": "gdb command: source art commands",
            "ignoreFailures": false
        }
    ],
    "name": "(gdbclient.py) Attach app_process64 (port: 5039)",
    "miDebuggerServerAddress": "localhost:5039",
    "request": "launch",
    "type": "cppdbg",
    "cwd": "/android/android/android-10.0.0_r2",
    "MIMode": "gdb"
}
    ]
}

```

启动 native 调试
------------

*   输出符号加载为连接调试 gdb 成功

![](https://bbs.kanxue.com/upload/attach/202209/853140_KR39B84TMQVB38T.png)

最后 打开 AS open Android ipr 项目
----------------------------

*   按照 java 层 调试方法附加调试 触发 第三方 app 执行
*   到断点处自然断下来

![](https://bbs.kanxue.com/upload/attach/202209/853140_P4NQK6GKT4Z9C75.png)

 

![](https://bbs.kanxue.com/upload/attach/202209/853140_7UESR9UUTWB68EV.png)

vscode 附加调试失败
-------------

`gdbclient.py -p 6018 --setup-forwarding vscode` 执行之后 vscode 附加 发现链接失败时

### 手动运行 gdbserver

```
# 调试模式启动 第三方app 此命令需手动点击
adb shell am set-debug-app -w com.example.dexlassloaders
# 以调试模式启动 无需手动点击
adb shell am start -D -n  com.example.dexlassloaders/.MainActivity
# 查看进程pid
adb shell "ps -ef | grep com.example.dexlassloaders"
# u0_a103       6018  1631 0 18:24:11 ?     00:00:00 com.example.dexlassloaders
# root          6046  1677 0 18:25:54 ?     00:00:00 sh -c ps -ef | grep com.example.dexlassloaders
# root          6049  6046 0 18:25:54 ?     00:00:00 grep com.example.dexlassloaders
# 1. 进入手机 shell
adb shell
# 2. 切换root模式 普通手机为su
su
# 3. 手动执行gdbserver
gdbserver64 :1234 --attach 6018
# 出现下面的表示 附加调试成功
# Remote debugging from host 127.0.0.1
# 4. 重新启动一个终端
# 4.1 进行端口映射
adb forward tcp:5039 tcp:1234
# 4.2  按照 获取vscodelunch.json 配置 这个做 配置vscode 检查5039端口
# 5. 启动vscode附加调试 - 先下断点
vscode 按 F5 启动调试 查看调试控制台，应该开始Loaded symbols 了
# 6. 使用as 附加调试或者 执行下面的jdb 开始调试
adb forward tcp:12345 jdwp:6018
jdb -attach localhost:12345

```

异常问题处理
======

*   跳转到 .class  
    在 Edit API SDK 的时候 选择 Sourcepath 删除原本的 ipr 还是什么忘记了 全删 添加 源码目录的  
    • frameworks  
    • libcore  
    • 需要其他再添加
*   lunch 真机的时候 gdbclient.py 没有输出 必须模拟器才有 可修改模拟的输出 适配真机 vscode lunch 自行尝试 。

  

[4h 入门 PHP 代码审计之反序列化](https://www.kanxue.com/book-section_list-94.htm)

最后于 2022-11-21 22:38 被 iyue_t 编辑 ，原因： 板块不对