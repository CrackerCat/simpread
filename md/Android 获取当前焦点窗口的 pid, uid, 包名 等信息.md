> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [gist.github.com](https://gist.github.com/5ec1cff/9292d51d7fe68991e28e8e0a6534116a)

> Android 获取当前焦点窗口的 pid, uid, 包名 等信息. GitHub Gist: instantly share code, notes, and snippets.

使用 frida 的时候经常需要通过 pid 附加到进程，因为 frida 自带的 `-n` 参数实在太不好用了——本来 Android 的「进程名」概念是很清楚的，在 Manifest 就能看到，要是不清楚也可以通过 packageName 或者 cmdline 来替代，但 frida 却非要让「应用的名字」作为注入应用主进程的唯一标识，什么包名、命令行，统统不认，遇到中文应用名就很麻烦；此外，这个操作似乎还要注入系统服务，从而导致 stop server 后发生各种诡异崩溃，因此我不太喜欢用这个参数。然而每次都要打开 shell ，输入 `ps -ef | grep` 找应用的 pid 很是麻烦，并且对于多进程的应用并非总是能「猜对」它的进程名的。于是写了这个脚本，至少可以确定当前看到的这个窗口属于哪个进程（没焦点的就不好说了）。

脚本运行在 Android 上的 shell (建议 `/system/bin/sh`)，在主机端只要简单用 adb shell 之类的封装一下用起来就很方便了。

脚本主要利用 `dumpsys window` 的 mFocusedWindow 取得当前焦点窗口的 hash ，并查找 hash 对应窗口的 Session 信息获取 uid 和 pid 。

仅在 Android 11 上通过测试，adb 权限即可。

写的过程中各种 google 「shell 字符串处理」，绞尽脑汁，最后写成了这么一个只用了一次 dumpsys ，然而带来了大量的 grep 调用的脚本。因为用到了 shell 数组，导致不支持个别 shell （如 magisk busybox ash）。grep 对正则表达式的兼容在不同系统的不同版本似乎也不尽相同。另外 dumpsys 在不同 Android 版本的输出也不一致。总之，这个脚本的兼容性奇差无比。

本来想通过解析 dumpsys 的 protobuf 输出来获取这些信息的，然而 protobuf 并不包含 Session 的相关信息，不过倒是有 ActivityRecord 的 pid ，但是对非 activity 似乎就没太大用处了（不过一般的应用场景也就是 activity 了，或许可以考虑写一个处理 protobuf 的？）。

最后附上研究过程中学到的 protobuf 的使用方法：

```
从 Android 源码下载对应 protobuf 文件：
platform/frameworks/proto_logging/+/refs/heads/master:protos: .proto
platform/frameworks/base/+/refs/heads/master/core/proto:protos: .proto

编译成某种语言（如 python ，对后面的步骤不是必须的）：
find protos -name "*.proto" |xargs -I {} protoc --proto_path=protos --python_out=py "{}"

dump window proto:

dumpsys window --proto > s.buf

decode:

cat s.buf | protoc --decode_raw

cat s.buf|protoc --decode=com.android.server.wm.WindowManagerServiceDumpProto --proto_path=protos protos/frameworks/base/core/proto/android/server/windowmanagerservice.proto > dump.txt


```