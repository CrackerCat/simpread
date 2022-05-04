> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272604.htm)

> [原创] 向 Typora 学习 electron 安全攻防

[](#【原创】向typora学习electron安全攻防)【原创】向 Typora 学习 electron 安全攻防
===========================================================

![](https://bbs.pediy.com/upload/attach/202205/860779_4UMX87EAUA79V9M.png)

 

@钞能力大叔

 

目标应用: `aHR0cHM6Ly90eXBvcmEuaW8v`

 

越来越多的应用开始使用 `electron` 来构建跨平台桌面应用。从实现方式上来说，其本质还是基于 chrome 内核的 html、js、css 构成的应用，基于浏览器，代码必定会暴露在用户侧，任何加密手段只是增加破解门槛跟时间成本而已。

认识 electron 项目文件目录特征
--------------------

![](https://bbs.pediy.com/upload/attach/202205/860779_MD7FZHAZDF6WTR2.png)

 

![](https://bbs.pediy.com/upload/attach/202205/860779_TSXAK9WUDF6X7SA.png)

 

electron 打包的项目，最常见的就是 `asar` 格式的私有编码文件，里面包含文件名、大小、内容偏移量等数据，按文件头部的 json 内容 解析即可提取出所有文件。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_SGDD8Y9TK38G36B.png)

electron asar 解包
----------------

目前来说，官方的版本并没有提供保护源码的方法。在 github 开源的找到个大神提供的解决方案（https://github.com/toyobayashi/electron-asar-encrypt-demo） ，该方案可以把启动文件编译为 node 二进制文件，作为启动入口，来保护薄弱的 js 代码。在项目启动时，将加密后的代码进行解密，交回 electron 流程进行执行，从而避免上述步骤直接解包拿到源代码的可能。

 

经过分析对比，typora 用的恰好是这个 demo 提供的思路。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_QYWFNJPSN5PMEBB.png)

 

找到上述步骤解压成功的 `package.json` 文件，`main`属性就是 electron 项目启动的主入口。 把 `main.node` 拖到 ida 中, 分析执行流程。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_DG33Y9DAHP4K5VG.png)

 

结合 `electron-asar-encrypt-demo` 跟 IDA 伪代码， 可以定位到两个关键函数 `napi_create_string_utf8` 跟 `napi_call_function`

万能 js 提取方案
----------

已知经过编译后的 node 文件, 在执行前，都会调用 `napi_create_string_utf8` 创建 js 字符串，所以在调用该函数的时候，基于 electron 没有自带加密的特性，想要运行 js 流程，必定会传递明文，所以在加载流程上截断，就可以导出解密后的数据。

 

直接上 frida 大法

```
let napi_create_string_utf8 = Module.getExportByName(null, 'napi_create_string_utf8');
var index = 0;
if (napi_create_string_utf8) {
    console.log('绑定成功');
    Interceptor.attach(napi_create_string_utf8, {
        onEnter: function (args) {
            console.log('napi_create_string_utf8', '调用', args[0], args[1].readCString().substring(0, 100), args[2], args[3]);
 
            if (args[2].toInt32() > 100) { // 过滤出大文件
                index += 1;
                var f = new File('export_' + String(index) + '.js', 'wb');
                f.write(args[1].readByteArray(args[2].toInt32()));
                f.flush();
                f.close();
 
            }
        }
    });
} else {
    console.log('绑定失败');
}

```

![](https://bbs.pediy.com/upload/attach/202205/860779_6YWSGGPJBMEN3R5.png)

 

解包项目中的所有 `asar`文件，就能拿到所有的源代码，并且把解压的文件夹，改成对应的 `asar` 文件名即可正常运行项目，无需重打包即可调试。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_8AZQEBKVXWW8PE7.png)

无限试用思路
------

删除注册表键，让程序认为是第一次安装

 

![](https://bbs.pediy.com/upload/attach/202205/860779_6W2JYYUFVX9TJWJ.png)

 

![](https://bbs.pediy.com/upload/attach/202205/860779_SJS288W2V82GRCE.png)

 

不想进入注册表这个繁琐的话，可以保存下面的代码, 把名字改成 `xxx.reg`, 双击运行即可无限畅玩。

```
Windows Registry Editor Version 5.00
 
[HKEY_CURRENT_USER\Software\Typora]
"IDate"="1/24/9999"
"SLicense"=""

```

去除授权功能实现单机版本思路
--------------

将所有的网络验证给他删掉 / 注释掉, 也可以直接把 “是否授权” 的变量，改成 true 即可。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_VUXH58SUP4REP7Q.png)

 

![](https://bbs.pediy.com/upload/attach/202205/860779_TC4USCC3RVFAAMH.png)

注册机思路
-----

由于该示例使用了 rsa 解密，所以要基于官方版本编写注册机就不太行了。一定要出注册机的话，需要先替换截图部分的 rsa 公钥即可。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_D3WHZ3ERPQGZS56.png)

 

这个地方使用了 node 自带的公钥解密，一些语言好像不能直接生成（可能我的打开方式不对），我就直接给出 node 版本的创建实例。

```
const crypto = require('crypto');
const fs = require('fs');
const path = require('path');
 
const keyPair = crypto.generateKeyPairSync('rsa', {
   modulusLength: 2048,
   publicKeyEncoding: {
      type: 'spki',
      format: 'pem'
   },
   privateKeyEncoding: {
      type: 'pkcs8',
      format: 'pem',
      cipher: 'aes-256-cbc',
      passphrase: ''
   }
});
 
fs.writeFileSync("public_key.pem", keyPair.publicKey);
fs.writeFileSync("private_key.pem", keyPair.privateKey);

```

生成后的公钥, 覆盖即可，然后编写自己的注册机。

自建授权网关实现注册机思路
-------------

自建网关的话，可以把应用自带的域名，替换成自己的，然后按接口需要的返回值，给他返回响应的数据格式即可。 这个实例仅需要修改两处即可。

 

![](https://bbs.pediy.com/upload/attach/202205/860779_SNVE8QC83KE9W67.png)

 

![](https://bbs.pediy.com/upload/attach/202205/860779_A6BDF6CC98PCPHU.png)

 

接口就不提供了，放张效果图

 

![](https://bbs.pediy.com/upload/attach/202205/860779_5D6ZBCQBXVHHA46.png)

重打包发版思路
-------

把解压的文件夹打包回 `asar`格式的文件即可， 这个网上一大把资料。

对于 electron 加密方式的思考
-------------------

相对于原生开发来说，js 安全做客户端毕竟薄弱，UI 交互是没问题的，关键代码可以放到 dll、so 层去做，但是也没办法避免从 js 层面去剥离 dll 层函数的调用。所以目前来说并没有很好的解决方案，本文只是起到抛砖引玉的作用。期待 electron 有更好、更安全的解决方案。

[【看雪培训】目录重大更新！《安卓高级研修班》2022 年春季班开始招生！](https://bbs.pediy.com/thread-271992.htm)

最后于 17 小时前 被钞能力大叔编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#软件保护](forum-4-1-3.htm) [#问题讨论](forum-4-1-197.htm)