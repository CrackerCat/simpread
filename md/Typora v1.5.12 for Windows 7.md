> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2042969-1-1.html)

> 注：理论上适合所有版本！一、按照系统架构下载并安装：https://download.typora.io/windows/typora-setup-x64-1.5.12.exehttps://downl......

![](https://avatar.52pojie.cn/data/avatar/001/18/15/49_avatar_middle.jpg)chishingchan 注：理论上适合所有版本！  
一、按照系统架构下载并安装：  
[https://download.typora.io/windows/typora-setup-x64-1.5.12.exe](https://download.typora.io/windows/typora-setup-x64-1.5.12.exe)  
[https://download.typora.io/windows/typora-setup-ia32-1.5.12.exe](https://download.typora.io/windows/typora-setup-ia32-1.5.12.exe)  
二、开始[破解](https://www.52pojie.cn)步骤：  
解包工具：WinAsar.exe  
下载地址：[https://github.com/aardio/WinAsar](https://github.com/aardio/WinAsar)  
解包文件：C:\Program Files\Typora\resources\node_modules.asar  
解包路径：C:\Program Files\Typora\resources\node_modules.asar.unpack  
操作步骤：  
        1、运行 WinAsar.exe  
        2、点击 解包 标签  
        3、点击 打开（C:\Program Files\Typora\resources\node_modules.asar）  
        4、点击 提取（C:\Program Files\Typora\resources\node_modules.asar.unpack）  
编辑工具：EditPlus v4.3  
下载地址：[https://www.editplus.com/latest4.html](https://www.editplus.com/latest4.html)  
编辑文件：C:\Program Files\Typora\resources\node_modules.asar.unpack\raven\index.js  
添加内容以下：  
        行号        内容  
        12        require('./hook')  
        13          
添加文件：C:\Program Files\Typora\resources\node_modules.asar.unpack\raven\hook.js  
文件内容如下：---- 与 ==== 之间的内容  
----------------------------------------------------------------  
// 本文件是一个小例子，说明如何使用注入能力钩住一个函数并改变其行为，  
// 在本例中，我们钩住了负责解密许可证的函数，并改变了行为，返回有效的许可证。  
// 仅用于学习目的，不要使用它来破解软件  
// Adding hook  
const crypto = require("crypto");  
const pubdec = crypto["publicDecrypt"];  
delete crypto["publicDecrypt"];  
let fingerprint, email, uuid, license, computerInfo = "";  
let License = ""  
crypto.publicDecrypt = function (key, buffer) {  
    log("PubDec Key:" + key);  
    log("buf:" + buffer.toString('base64'));  
    if (buffer.slice(0, 26).compare(Buffer.from("CRACKED_BY_DIAMOND_HUNTERS")) == 0) {  
        License = buffer.toString('base64');  
        let ret = buffer.toString().replace("CRACKED_BY_DIAMOND_HUNTERS", "");  
        log("backdoor data,return :" + ret);  
        return Buffer.from(ret);  
    }  
    return pubdec(key, buffer);  
};  
const fetch = require("electron-fetch")  
fetch_bak = fetch['default'];  
delete fetch['default'];  
fetch.default = async function fetch(url, options) {  
    log('[fetch]fetch' + url);  
    log('[fetch]Arg' + JSON.stringify(options));  
    data = await fetch_bak(url, options);  
    if (url.indexOf('api/client/activate') != -1) {  
        params = JSON.parse(options.body);  
        fingerprint = params.f, email = params.email, uuid = params.u, license = params.license, computerInfo = params.l  
        log('[activate]Fingerprint' + fingerprint);  
        log('[activate]Email' + email);  
        log('[activate]UUID' + uuid);  
        log('[activate]License' + license);  
        log('[activate]ComputerInfo' + computerInfo);  
        log('[fetch]RetCode' + data.status);  
        ret = await data.buffer();  
        log('[fetch]Ret' + ret.toString());  
        ret = Buffer.from('{"code":0,"retry":true,"msg":"' + Buffer.from("CRACKED_BY_DIAMOND_HUNTERS" + JSON.stringify(  
            {  
                "fingerprint": fingerprint,  
                "email": email,  
                "license": license,  
                "type": ""  
            })).toString('base64') + '"}');  
        log("replace ret:" + ret.toString());  
        data.text = () => {  
            return new Promise((resolve, reject) => {  
                resolve(ret.toString());  
            });  
        };  
        data.json = () => {  
            return new Promise((resolve, reject) => {  
                resolve(JSON.parse(ret.toString()));  
            });  
        };  
    }  
    if (url.indexOf('api/client/renew') != -1) {  
        ret = await data.buffer();  
        log('[fetch]Ret' + ret.toString());  
        ret = Buffer.from('{"success":true,"code":0,"retry":true,"msg":"' + License + '"}');  
        log("replace ret:" + ret.toString());  
        data.text = () => {  
            return new Promise((resolve, reject) => {  
                resolve(ret.toString());  
            });  
        };  
        data.json = () => {  
            return new Promise((resolve, reject) => {  
                resolve(JSON.parse(ret.toString()));  
            });  
        };  
    }  
    return new Promise((resolve, reject) => {  
        resolve(data);  
    });  
}  
http = require("http")  
function log(str) {  
    http.get('http://127.0.0.1:3000/log?str=' + str, res => {  
    }).on('error', err => {  
        console.log('Error:', err.message);  
    });  
}  
log = console.log;  
log('Hook Init')  
var Module = require('module');  
var originalRequire = Module.prototype.require;  
Module.prototype.require = function () {  
    log('Require' + arguments[0])  
    if (arguments[0] == 'crypto') {  
        log('Hooking crypto');  
        return crypto;  
    }  
    if (arguments[0] == 'electron-fetch') {  
        log('Hooking electron-fetch');  
        return fetch;  
    }  
    return originalRequire.apply(this, arguments);  
};  
console.log = log  
let val[IDA](https://www.52pojie.cn/thread-1874203-1-1.html)tor = {  
    set: function (target, key, value) {  
        if (key === 'log') {  
            log('console.log override blocked');  
            rerturn;  
        }  
        target[key] = value;  
    }  
}  
let proxy = new Proxy(console, validator);  
console = proxy  
module.exports = fetch  
// hook finished  
================================================================  
打包工具：WinAsar.exe  
打包路径：C:\Program Files\Typora\resources\node_modules.asar.unpack  
打包文件：C:\Program Files\Typora\resources\node_modules.asar.unpack.asar  
操作步骤：  
        1、运行 WinAsar.exe  
        2、点击 打包 标签  
        3、点击 添加 按钮，选择文件夹：（C:\Program Files\Typora\resources\node_modules.asar.unpack）  
        4、点击 打包 按钮，生成成品文件（C:\Program Files\Typora\resources\node_modules.asar.unpack.asar）  
        5、删除 C:\Program Files\Typora\resources\node_modules.asar  
        6、将 node_modules.asar.unpack.asar 重命名为 node_modules.asar  
![](https://attach.52pojie.cn/forum/202507/02/113729sgttdp9cpzhyypy9.png)

**1.png** _(50.81 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4Njc0MHxmOWZhNDM4YnwxNzUxNDM4OTU0fDIxMzQzMXwyMDQyOTY5&nothumb=yes)

2025-7-2 11:37 上传

  
![](https://attach.52pojie.cn/forum/202507/02/113731y5m977tevtx2t72v.png)

**2.png** _(56.83 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4Njc0MXxjZTJkM2QzMHwxNzUxNDM4OTU0fDIxMzQzMXwyMDQyOTY5&nothumb=yes)

2025-7-2 11:37 上传

  
![](https://attach.52pojie.cn/forum/202507/02/113733ggzpvi0s0v7rzu8z.png)

**3.png** _(49.82 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4Njc0MnwxN2Y2ZTliM3wxNzUxNDM4OTU0fDIxMzQzMXwyMDQyOTY5&nothumb=yes)

2025-7-2 11:37 上传![](https://avatar.52pojie.cn/data/avatar/001/18/15/49_avatar_middle.jpg)chishingchan 此帖子大部分内容摘自网络，只是总结的尽量明白而已。![](https://avatar.52pojie.cn/data/avatar/000/23/12/19_avatar_middle.jpg)孤狼微博 谢谢分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Duke0910 谢谢分享，准备试着弄下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) aide99 支持！谢谢分享![](https://avatar.52pojie.cn/images/noavatar_middle.gif)楚吾爱 谢谢 分享 自己试着弄一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif)yemind 这和系统自带的记事本有啥区别 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Echo001

> [yemind 发表于 2025-7-2 12:11](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=53350969&ptid=2042969)  
> 这和系统自带的记事本有啥区别

格式好一点 ![](https://avatar.52pojie.cn/data/avatar/000/49/30/61_avatar_middle.jpg) DrCatcher 清晰明了，谢谢分享！![](https://avatar.52pojie.cn/data/avatar/001/43/39/30_avatar_middle.jpg)宜城小站 感谢楼主分享专业级的文本编辑器![](https://static.52pojie.cn/static/image/smiley/default/42.gif)