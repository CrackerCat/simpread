> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1405917-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/000/50/78/87_avatar_middle.jpg)ogli324### 前提条件

```
一台root手机
frida环境一套
还要会搜索（回复比较慢）

```

### 开启抓包

```
1. 手机中执行 tcpdump
tcpdump -i any -s 0 -w /sdcard/Download/capture.pcap

2. 手机没有tcpdump的
下载地址https://www.androidtcpdump.com/android-tcpdump/downloads
adb push tcpdump /data/local/tmp/ (如果遇到权限不够的，先push到sdcard/ 再移动过去)

2.1在手机中执行给权限
chmod 777 tcpdump

2.2继续执行1
./tcpdump -i any -s 0 -w /sdcard/Download/capture.pcap


```

### hook app 拿到 sslkey

frida -U -f package -l ./sslkeyfilelog.js --no-pause

```
// frida 命令选项 更多关于frida信息 可以查看frida官方信息 https://frida.re/docs/home/
C:\Users\User>frida -h
Usage: frida [options] target

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -D ID, --device=ID    connect to device with the given ID
  -U, --usb             connect to USB device
  -R, --remote          connect to remote frida-server
  -H HOST, --host=HOST  connect to remote frida-server on HOST
  -f FILE, --file=FILE  spawn FILE
  -F, --attach-frontmost
                        attach to frontmost application
  -n NAME, --attach-name=NAME
                        attach to NAME
  -p PID, --attach-pid=PID
                        attach to PID
  --stdio=inherit|pipe  stdio behavior when spawning (defaults to “inherit”)
  --runtime=duk|v8      script runtime to use (defaults to “duk”)
  --debug               enable the Node.js compatible script debugger
  -l SCRIPT, --load=SCRIPT
                        load SCRIPT
  -P PARAMETERS_JSON, --parameters=PARAMETERS_JSON
                        Parameters as JSON, same as Gadget
  -C CMODULE, --cmodule=CMODULE
                        load CMODULE
  -c CODESHARE_URI, --codeshare=CODESHARE_URI
                        load CODESHARE_URI
  -e CODE, --eval=CODE  evaluate CODE
  -q                    quiet mode (no prompt) and quit after -l and -e
  --no-pause            automatically start main thread after startup
  -o LOGFILE, --output=LOGFILE
                        output to log file
  --exit-on-error       exit with code 1 after encountering any exception in
                        the SCRIPT

```

```
function startTLSKeyLogger(SSL_CTX_new, SSL_CTX_set_keylog_callback) {
    console.log("start----")
    function keyLogger(ssl, line) {
        console.log(new NativePointer(line).readCString());
    }
    const keyLogCallback = new NativeCallback(keyLogger, 'void', ['pointer', 'pointer']);

    Interceptor.attach(SSL_CTX_new, {
        onLeave: function(retval) {
            const ssl = new NativePointer(retval);
            const SSL_CTX_set_keylog_callbackFn = new NativeFunction(SSL_CTX_set_keylog_callback, 'void', ['pointer', 'pointer']);
            SSL_CTX_set_keylog_callbackFn(ssl, keyLogCallback);
        }
    });
}
startTLSKeyLogger(
    Module.findExportByName('libssl.so', 'SSL_CTX_new'),
    Module.findExportByName('libssl.so', 'SSL_CTX_set_keylog_callback')
)
// https://codeshare.frida.re/@k0nserv/tls-keylogger/

```

### 将抓包文件拿到 pc

将 / sdcard/Download/capture.pcap pull 到 pc

### 保存 frida 输出的打印信息到 sslkey.txt（还没做到一键傻瓜化）

```
// 格式是这样紫的， 不要参杂别的哟~
CLIENT_RANDOM 557e6dc49faec93dddd41d8c55d3a0084c44031f14d66f68e3b7fb53d3f9586d 886de4677511305bfeaee5ffb072652cbfba626af1465d09dc1f29103fd947c997f6f28962189ee809944887413d8a20
CLIENT_RANDOM e66fb5d6735f0b803426fa88c3692e8b9a1f4dca37956187b22de11f1797e875 65a07797c144ecc86026a44bbc85b5c57873218ce5684dc22d4d4ee9b754eb1961a0789e2086601f5b0441c35d76c448
CLIENT_RANDOM e1c1dcaaf73a8857ee60f5b38979084c3e95fdebd9791bbab985a8f954132426 41dcf3d5e41cb469494bf5014a1ecca9f40124f5728895265fadd38f8dc9d5ac15c5fa6588c1ea68f38476297fe76183
CLIENT_RANDOM 66c4f37afb2152e3837c8a7c48ce51e8307e6739e1fe3efc542887bbcae4f02a bbafe4881084570af01bed59f95bfcf7bc49d2e55acbc7fe33c1e06f8ff0bc2e747c2c428e7cd13f1c77c2141085f951
CLIENT_RANDOM 8d0d92154ee030486a2b13f9441f85ef33c5e06732fbb06a1ac81fe34b6f2ce3 8270b34eee784e7f7de45f39af36f26e6abf99bb52fa8350945e3ebf79dc1c53a0693c24b0780ce3a54d39fd4b5b5149
CLIENT_RANDOM b5d58899346db525f14312cfb52c1247ed7adb710ae43428bd331ce27d77dbc1 9effd5b469ef6fdf7a056ea50fc3ff0fdf9fa40ae709805bea8678ddce404f211ed534623876a5c616f3e7bc43121f48
CLIENT_RANDOM af1b3f9ba0b4c27756c93595eb54cac6f0d8c6e9e4f0fcb1a36c45f0cd12060d 696a6fff39bf6c9863901a2145703de948c37e1abf6b4c03628118bee11c292239304ee020c71ff31a293fc6b9439364
CLIENT_RANDOM e2a3d8e6b638976aa27c8cf031be5e6b03cf7ffa573be101816d5103025d404b 2b006379423d7252c864a129b6c5a693b75d477dc5d3f894af5f02db755c4f6dd54470b659882871c62ce002792e211a
CLIENT_RANDOM 1c8cfe911e2111d80dc81c275c791c04467e8d7bca16963acec6a20051429981 bf08334d973d44d80c8f4542c2356a5fd9e0d390afde0374179cc81dd82aaa15aae52604988e9c9616ad0795c79c81ed

```

### 打开. pcap 文件 进入 Wireshark

```
// Wireshark快速过滤
(http.request or tls.handshake.type eq 1) and !(ssdp)
// 快速参考
// https://www.jianshu.com/p/5525594600db

```

### 设置 Wireshark tls

[b] 编辑 --> 首选项 --> Protocols--> TLS  
![](https://attach.52pojie.cn/forum/202103/30/164930izabb5y5ej339yu1.png)

 ![](https://attach.52pojie.cn/forum/202103/30/165000mroorroponiorjjr.png)   
 ![](https://attach.52pojie.cn/forum/202103/30/165014l8595449scr90yg0.png) 

选择到 tls 后将之前的 sslkey.txt 导入下。 再使用下前面的过滤命令， 正常情况下， 现在就能正常看到了。  
![](https://attach.52pojie.cn/forum/202103/30/165025nsjzh49ujohu0csh.png)

#### Wireshark 番外篇

[b] 编辑 --> 首选项 --> 外观 --> 布局  
设置完这个， 就也和我一样能看到教学图了， 一下子友好很多。  
![](https://attach.52pojie.cn/forum/202103/30/165037f3h2witeestwh3hh.png)

 ![](https://attach.52pojie.cn/forum/202103/30/165046jozqt31077otdcc6.png) 

### pc 浏览器 sslkey 番外篇

```
// 一样要把key导入哟， 不然Wireshark怎么会知道呢？
chrome.exe --ssl-version-max=tls1.3  --ssl-key-log-file="./ssl_keys.txt"

```

### 致敬大佬

```
@菱志漪 其实要感谢菱志漪大佬，没有他遇到的问题，我就不会深入去看研究这个问题，还要感谢@肉师傅， 没有陈老师通俗易懂的讲解， 我压根找不到北。 【手动撒花~~~】

```

### 以下参考内容

```
Wireshark Tutorial: Display Filter Expressions
https://unit42.paloaltonetworks.com/using-wireshark-display-filter-expressions/

HTTPS Traffic Without the Key Log File
https://unit42.paloaltonetworks.com/wireshark-tutorial-decrypting-https-traffic/

The First Few Milliseconds of an HTTPS Connection
http://www.moserware.com/2009/06/first-few-milliseconds-of-https.html

Sniffing HTTPS Traffic in Chromium with Wireshark
https://adw0rd.com/2020/12/2/chromium-https-ssl-tls-sniffing-with-wireshark/en/

``` ![](https://avatar.52pojie.cn/data/avatar/000/50/78/87_avatar_middle.jpg)ogli324

> [xianyucoder 发表于 2021-3-30 19:40](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=37761837&ptid=1405917)  
> 给 南山大佬 打 call

![](https://static.52pojie.cn/static/image/smiley/laohu/laohu34.gif)大佬好～![](https://avatar.52pojie.cn/data/avatar/000/50/78/87_avatar_middle.jpg)ogli324

> [wykdz 发表于 2021-3-31 11:26](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=37773643&ptid=1405917)  
> frida -f package -l ./sslkeyfilelog.js --no-pause  
> 这个 package 应该替换成什么？  我的手机里有 com.y ...

仔细排查后发现文档写的有点小问题，  
frida 有两种选项， 1. -U -->usb 2.-H -->host ip:port  
所以正确的是 frida -U -f package -l ./sslkeyfilelog.js --no-pause  package-->app 包名。![](https://avatar.52pojie.cn/data/avatar/000/97/11/42_avatar_middle.jpg)泳诗 很好的抓包思路，谢谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/63/25/55_avatar_middle.jpg) cob 感谢大佬的讲解 ![](https://avatar.52pojie.cn/data/avatar/001/52/24/68_avatar_middle.jpg) alibunny 感谢大佬的讲解 ![](https://avatar.52pojie.cn/data/avatar/001/66/77/67_avatar_middle.jpg) LearningPawn5 第一次知道 Wireshark 还可以改布局的![](https://static.52pojie.cn/static/image/smiley/default/2.gif)学习了![](https://avatar.52pojie.cn/data/avatar/001/48/22/60_avatar_middle.jpg)武汉小 Liz 这个操作高端了, 学习了 ![](https://avatar.52pojie.cn/data/avatar/001/65/76/35_avatar_middle.jpg) yeah1 小白，我看不懂 ![](https://avatar.52pojie.cn/data/avatar/001/67/30/67_avatar_middle.jpg) jiklop 很好的抓包思路，谢谢分享![](https://avatar.52pojie.cn/data/avatar/000/78/42/02_avatar_middle.jpg)13169456869 感谢分享！！！![](https://avatar.52pojie.cn/data/avatar/000/72/81/72_avatar_middle.jpg)VNRKDOEA 直接舔爆