> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280769.htm)

> boss app sig 及 sp 参数, 魔改 base64(上)

boss app sig 及 sp 参数, 魔改 base64(上)

1 天前 325

### boss app sig 及 sp 参数, 魔改 base64(上)

 [![](http://passport.kanxue.com/upload/avatar/110/983110.png?1702097191)](user-home-983110.htm) [杨如画](user-home-983110.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 1 天前  325

前言
==

大家好呀, 欢迎来到我的博客. 2023 年 12 月 4 日, boss web 上线了最新的 zp_token, 环境检测点又增加了, 与此同时 app 端的关键加密 so 从 32 位换成了 64 位, 两者 ida 反编译 so 的时候都有反调试, 无法直接 f5, 需要手动调整让 ida 重新识别. google 了一下几乎找不到任何有关 boss app 的文章, 所以这篇文章讲解 app 端的加密. 篇幅较长, 坐稳发车啦!

本章所有样本均上传 123 云盘, 需要复刻的自行下载.

[https://www.123pan.com/s/4O7Zjv-UYFBd.html](https://www.123pan.com/s/4O7Zjv-UYFBd.html)

设备 pixel 4xl android10

版本: 11.240

下载地址: aHR0cHM6Ly93d3cud2FuZG91amlhLmNvbS9hcHBzLzYyMDIyMjIvaGlzdG9yeV92MTEyNDAxMA==

工具: charles(抓包) socksdroid(流量转发) jadx(反编译 dex) ida(反编译 so) frida(注入) frida-trace(还原算法)

声明
==

本文章中所有内容仅供学习交流使用，不用于其他任何目的，不提供完整代码，抓包内容、敏感网址、数据接口等均已做脱敏处理，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关！  
本文章未经许可禁止转载，禁止任何修改后二次传播，擅自使用本文讲解的技术而导致的任何意外，作者均不负责，若有侵权，请联系作者立即删除！

抓包
==

![](https://bbs.kanxue.com/upload/attach/202403/983110_77NXBSB5FHQZSUN.png)

反复抓包后确定了是这个包, 只不过响应是加密的, params 里有 sp 和 sig 参数, sp 参数是一个长串, sig 由 V3.0 拼接一个 32 位字符串, 猜测是 md5

sig 分析
======

可以尝试搜索字符串 sig 或者 hook hashmap 等等方法定位, 我这里 hook hashmap

```
Java.perform(function (){
    var hashMap = Java.use("java.util.HashMap");
    hashMap.put.implementation = function (a, b) {
        console.log("hashMap.put: ", a, b);
    return this.put(a, b);
}
})

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_KXDX7H5Q5UAUAPY.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_Q7QNEUVZWVBYME6.png)

可以看到 sig 参数和 sp 参数都 hook 到了, 接下来调整下 hook 代码, 把堆栈输出下

![](https://bbs.kanxue.com/upload/attach/202403/983110_RXEX93C4VTD4943.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_UF4QW67GKKURD2H.png)

搜索 net.bosszhipin.base.m 这个类可以看到 sp 和 sig, 八成就是这里了, 这里先 hook sig

![](https://bbs.kanxue.com/upload/attach/202403/983110_T8MZ2V82CCG2QKQ.png)

从 h 方法点进去看看

![](https://bbs.kanxue.com/upload/attach/202403/983110_ZY3UVCBHUHCN2TR.png)

右键复制 frida 片段

![](https://bbs.kanxue.com/upload/attach/202403/983110_TJRFJWQ2Y8EUKKS.png)

抓个包后确认就是这里了

接着从 signature 方法点进去

![](https://bbs.kanxue.com/upload/attach/202403/983110_9GV58KKXFSCTAYT.png)

接下来方便还原算法我们需要固定好入参, 写 java 层的主动调用, 然后配合着 ida 静态分析来还原算法

```
function call(){
    Java.perform(function (){
let YZWG = Java.use("com.twl.signer.YZWG");
var str = '/api/batch/batchRunV2batch_method_feed=%5B%22method%3DzpCommon.adActivity.getV2%26dataType%3D0%26expectId%3D802924422%26dataSwitch%3D1%22%2C+%22method%3Dzpgeek.app.f1.newgeek.jobcard%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%22%2C+%22method%3Dzpgeek.app.geek.trait.tip%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%22%2C+%22method%3Dzpgeek.cvapp.applystatus.change.tip%22%2C+%22method%3Dzpinterview.geek.interview.f1.complainTip%22%2C+%22method%3Dzpgeek.cvapp.geek.remind.warnexp%26entrance%3D1%26itemType%3D1%22%2C+%22method%3Dzpgeek.app.f1.banner.query%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%26filterParams%3D%257B%2522cityCode%2522%253A%2522101010100%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26gpsCityCode%3D0%26jobType%3D0%26mixExpectType%3D0%26sortType%3D1%22%2C+%22method%3Dzpinterview.geek.interview.f1%22%2C+%22method%3Dzpgeek.app.f1.recommend.filter%26commute%3D%26distance%3D0%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectPosition%3D%26filterFlag%3D0%26filterParams%3D%257B%2522cityCode%2522%253A%2522101010100%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26filterValue%3D%26jobType%3D0%26mixExpectType%3D0%26partTimeDirection%3D%26positionCode%3D%26sortType%3D1%22%2C+%22method%3Dzpgeek.app.bluecollar.topic.banner%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%22%2C+%22method%3Dzpgeek.cvapp.geek.homeexpectaddress.query%26cityCode%3D101010100%22%2C+%22method%3Dzpgeek.app.f1.interview.recjob.tip%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%22%2C+%22method%3Dzpgeek.app.geek.recommend.joblist%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26sortType%3D1%26expectPosition%3D100514%26pageSize%3D15%26expectId%3D802924422%26page%3D1%26filterParams%3D%257B%2522cityCode%2522%253A%2522101010100%2522%252C%2522switchCity%2522%253A%25220%2522%257D%22%2C+%22method%3Dzpgeek.app.studyabroad.article.headlines%22%2C+%22method%3Dzpgeek.cvapp.geek.resume.queryquality%22%5D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221703159294618%22%2C%22resume_time%22%3A%221703159294618%22%2C%22channel%22%3A%2227%22%2C%22model%22%3A%22google%7C%7CPixel+4+XL%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%227fe541f3-a666-4845-9186-cff9d5429f77%22%2C%22oaid%22%3A%22NA%22%2C%22did%22%3A%22DUzpQpzBYtoakGWwhYSfr2VDxKhBVPnGWdbfRFV6cFFwekJZdG9ha0dXd2hZU2ZyMlZEeEtoQlZQbkdXZGJmc2h1%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A0%7D&curidentity=0&req_time=1703162554818&uniqid=7fe541f3-a666-4845-9186-cff9d5429f77&v=11.200'
       var str2 = null
var res = YZWG["signature"](str,str2)
        console.log(res)
    })
}

```

可以发现 return 的值是来着 nativeSignature 方法, 看名字就知道他应该是一个 native 方法

![](https://bbs.kanxue.com/upload/attach/202403/983110_BZMZM2XAH4NH63S.png)

点进去后发现确实是 native 方法, 并且上面还有许多 native 方法, 包含 sp 的加密方法 nativeEncodeRequest(sp 的寻找方式后续就不介绍了, 和 sig 差不多), 而且从字面上看, 里面有解密数据的方法, 正好对应响应数据的解密

![](https://bbs.kanxue.com/upload/attach/202403/983110_ESUQAFMPMDERJTY.png)

往上找可以发现加载自 yzwg 这个 so 文件

![](https://bbs.kanxue.com/upload/attach/202403/983110_KF84MMN8XYBM5SE.png)

解包后可以看到只有 64 的 so 11.230 版本以前都是 32 的 so, 找到 libyzwg.so 并拖到 ida64 里反编译

![](https://bbs.kanxue.com/upload/attach/202403/983110_UYM6FCYN26NRXUY.png)

在导出表里搜索 jni 发现是动态注册, 这里可以直接上脚本找出这个 so 注册的函数

```
// 获取 RegisterNatives 函数的内存地址，并赋值给addrRegisterNatives。
var addrRegisterNatives = null;
// 列举 libart.so 中的所有导出函数（成员列表）
var symbols = Module.enumerateSymbolsSync("libart.so");
for (var i = 0; i < symbols.length; i++) {
    var symbol = symbols[i];
    // console.log(symbol.name)
    //_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi
    if (symbol.name.indexOf("art") >= 0 &&
        symbol.name.indexOf("JNI") >= 0 &&
        symbol.name.indexOf("RegisterNatives") >= 0 &&
        symbol.name.indexOf("CheckJNI") < 0) {
 
        addrRegisterNatives = symbol.address;
        console.log("RegisterNatives is at ", symbol.address, symbol.name);
        break
    }
}
if (addrRegisterNatives) {
    // RegisterNatives(env, 类型, Java和C的对应关系,个数)
    Interceptor.attach(addrRegisterNatives, {
        onEnter: function (args) {
            var env = args[0];        // jni对象
            var java_class = args[1]; // 类
            var class_name = Java.vm.tryGetEnv().getClassName(java_class);
            var taget_class = "com.twl.signer.YZWG";   //111 某个类中动态注册的so
            if (class_name === taget_class) {
                //只找我们自己想要类中的动态注册关系
                console.log("\n[RegisterNatives] method_count:", args[3]);
                var methods_ptr = ptr(args[2]);
                var method_count = parseInt(args[3]);
                for (var i = 0; i < method_count; i++) {
                    // Java中函数名字的
                    var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));
                    // 参数和返回值类型
                    var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));
                    // C中的函数内存地址
                    var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize * 2));
                    var name = Memory.readCString(name_ptr);
                    var sig = Memory.readCString(sig_ptr);
                    var find_module = Process.findModuleByAddress(fnPtr_ptr);
                    // 地址、偏移量、基地址
                    var offset = ptr(fnPtr_ptr).sub(find_module.base);
                    console.log("name:", name, "sig:", sig,'module_name:',find_module.name ,"offset:", offset);
 
                }
 
            }
        }
    });
}
命令 frida -U -f com.hpbr.bosszhipin -l 文件名.js

```

结果 nativeSignature 也就是 sig 的加密, 偏移 0x21864 nativeEncodeRequest 也就是 sp 的加密, 偏移 0x209a4, 后续 ida 中按 g 就可以跳到制定函数处![](https://bbs.kanxue.com/upload/attach/202403/983110_2VJ23ABKHKG5N2Y.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_AQGP3WUHMDCYF47.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_DAJEZPZXV6MXD5N.png)

可以看到 text 段是金色的, 也就是说 ida 错误的把原本是代码的地方识别成了数据, 这个时候需要手动帮 ida 一把, 让他重新把数据识别成代码

![](https://bbs.kanxue.com/upload/attach/202403/983110_5UTFW2GVZC3DV5Z.png)

选中金色段按 c(code) 转为代码

![](https://bbs.kanxue.com/upload/attach/202403/983110_UY8GMG7ZM6JDVAP.png)

转化后出现红色段就按 p(create function)

![](https://bbs.kanxue.com/upload/attach/202403/983110_M94N6VQF7V6M7FR.png)

重复这个过程直到把关键函数转为代码后就可以 f5 了

![](https://bbs.kanxue.com/upload/attach/202403/983110_C5HJXE6MEEJA8WW.png)

跳到 0x21864 位置转为伪 c 代码

![](https://bbs.kanxue.com/upload/attach/202403/983110_4AUQY4DZJHPN766.png)

点进去 300 多行代码, 不太好分析, 可以借助 frida trace 来跟踪 native 函数执行的时候调用了哪些函数

使用方法 [https://github.com/Pr0214/trace_natives](https://github.com/Pr0214/trace_natives)

![](https://bbs.kanxue.com/upload/attach/202403/983110_TYUFMHTBSPMZPDS.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_V6DCMDVVVASZJPK.png)

然后主动调用上面的 java 方法

![](https://bbs.kanxue.com/upload/attach/202403/983110_588A48R24JVV6KT.png)

把执行的函数复制到 notepad 里分析一下, 调用了挺多函数, 这里就没什么特殊的技巧了, 只能凭借着经验猜测哪个是关键函数

在 hook 1c714 函数的时候我发现了结果, hook 代码

```
var soAddr = Module.findBaseAddress("libyzwg.so");
var funcAddr = soAddr.add(0x1c714)  //32位+1
 
Interceptor.attach(funcAddr,{
            onEnter: function(args){
                console.log('onEnter arg[0]: ',hexdump(args[0],{length:args[1].toInt32()}))
                console.log('onEnter arg[1]: ',args[1])
 
                this.arg0 = args[0]
            },
            onLeave: function(retval){
                // console.log('onLeave arg[0]: ',hexdump(this.arg0.readPointer()))
                console.log('onLeave result: ',hexdump(retval))
 
            }
        });

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_96U5B486MWXZD7M.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_Y4A4ZYJNG7VZ4U8.png)

arg0 是 java 传进来的明文, onleave 的时候结果出来了, 并且明文传进去的时候还加了一个 salt

复制到 CyberChef 里加密一下

![](https://bbs.kanxue.com/upload/attach/202403/983110_JREEKGFP9MEU552.png)

是标准的 md5, 笔者在分析这个 sig 的时候当时并没有直接尝试加密, 而是 hook 了他下面的函数 2a5b8

![](https://bbs.kanxue.com/upload/attach/202403/983110_QW8VCVTKUUU473P.png)

hook 2a5b8 代码

```
// 2A5B8
var soAddr = Module.findBaseAddress("libyzwg.so");
var funcAddr = soAddr.add(0x2A5B8)  //32位+1
 
var num = 0
Interceptor.attach(funcAddr,{
            onEnter: function(args){
                num+=1
                console.log(`onEnter arg[0]  ${num}次:  ${args[2]} `,hexdump(args[0]))
                console.log('onEnter arg[1]: ',hexdump(args[1],{length:512}))
                // console.log('onEnter arg[2]: ',args[2])
 
                this.arg0 = args[0]
            },
            onLeave: function(retval){
                console.log('onLeave arg[0]: ',hexdump(this.arg0))
 
            }
        });

```

可以看到在执行第 8 次 2a5b8 函数后结果也是出现了

![](https://bbs.kanxue.com/upload/attach/202403/983110_CRMSHWAKNDG5792.png)

再来看第一次调用, 可以看到 arg0 像是 md5 的 4 个初始化魔数, 只不过内存中是小端字节续, 由于 md5 的分组处理长度是 512bit, 所以需要多次压入数据, 正好对应调用多次 2a5b8 函数, 这个函数类似 c md5 中的 updata 和 final 过程

![](https://bbs.kanxue.com/upload/attach/202403/983110_W9RBF3UUSYZFQ8F.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_M3K89Q4Z3K6P3VN.png)

这里可以修改 c++ 中 md5 的最后填充数据和附加消息长度来验证是否是标准 md5

![](https://bbs.kanxue.com/upload/attach/202403/983110_3DDGYQ5FHS777YH.png)

可以发现是标准的 md5, 到此 sig 参数就分析完毕了. 我为什么要提这个 2a5b8 函数的执行过程, 有人会说, 我直接把明文拼接 salt 后 md5 发现是结果了不就行了吗, 是的, 但是如果有一天你把明文和 salt 拼接后 md5 发现不是你要的结果你该如何处理? 你不懂算法细节如何在不准确的伪 c 代码中分析还原算法? 并且还有可能遇到魔改算法你又该如何处理?

sp 分析
=====

md5 算法是 hash 算法, 不可逆, 作用是用来验签的, 防止数据包被篡改, 那就肯定有一个传递加密前明文的参数, 从上面的抓包来看只有可能是 sp 参数, 这就说的通了, 明文加密成 sp, 并且和明文的 MD5 一起传给后台, 后台接受数据包后解密 sp 得到明文, 并再次加密明文和传来的 sig 对比以防止数据包被篡改

分析 sig 的时候已经提了 sp 的分析, 和 sig 差不多, 同理可以主动调用

```
function call(){
    Java.perform(function (){
        let a = Java.use("com.twl.signer.a");
        var str = 'batch_method_feed=%5B%22method%3DzpCommon.adActivity.getV2%26dataType%3D0%26expectId%3D802924422%26dataSwitch%3D1%22%2C+%22method%3Dzpgeek.app.f1.newgeek.jobcard%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%22%2C+%22method%3Dzpgeek.app.geek.trait.tip%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%22%2C+%22method%3Dzpgeek.cvapp.applystatus.change.tip%22%2C+%22method%3Dzpinterview.geek.interview.f1.complainTip%22%2C+%22method%3Dzpgeek.cvapp.geek.remind.warnexp%26entrance%3D1%26itemType%3D1%22%2C+%22method%3Dzpgeek.app.f1.banner.query%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%26filterParams%3D%257B%2522cityCode%2522%253A%2522101010100%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26gpsCityCode%3D0%26jobType%3D0%26mixExpectType%3D0%26sortType%3D1%22%2C+%22method%3Dzpinterview.geek.interview.f1%22%2C+%22method%3Dzpgeek.app.f1.recommend.filter%26commute%3D%26distance%3D0%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectPosition%3D%26filterFlag%3D0%26filterParams%3D%257B%2522cityCode%2522%253A%2522101010100%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26filterValue%3D%26jobType%3D0%26mixExpectType%3D0%26partTimeDirection%3D%26positionCode%3D%26sortType%3D1%22%2C+%22method%3Dzpgeek.app.bluecollar.topic.banner%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%22%2C+%22method%3Dzpgeek.cvapp.geek.homeexpectaddress.query%26cityCode%3D101010100%22%2C+%22method%3Dzpgeek.app.f1.interview.recjob.tip%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26expectId%3D802924422%22%2C+%22method%3Dzpgeek.app.geek.recommend.joblist%26encryptExpectId%3Def7d7c83e4017a4233R40t-5FFBS%26sortType%3D1%26expectPosition%3D100514%26pageSize%3D15%26expectId%3D802924422%26page%3D1%26filterParams%3D%257B%2522cityCode%2522%253A%2522101010100%2522%252C%2522switchCity%2522%253A%25220%2522%257D%22%2C+%22method%3Dzpgeek.app.studyabroad.article.headlines%22%2C+%22method%3Dzpgeek.cvapp.geek.resume.queryquality%22%5D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221703222473770%22%2C%22resume_time%22%3A%221703222473770%22%2C%22channel%22%3A%2228%22%2C%22model%22%3A%22google%7C%7CPixel+4+XL%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%22b99e5c38-858b-4097-8b4d-084a6e75ec62%22%2C%22oaid%22%3A%22NA%22%2C%22did%22%3A%22DUzpQpzBYtoakGWwhYSfr2VDxKhBVPnGWdbfRFV6cFFwekJZdG9ha0dXd2hZU2ZyMlZEeEtoQlZQbkdXZGJmc2h1%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A1%7D&curidentity=0&req_time=1703222656138&uniqid=b99e5c38-858b-4097-8b4d-084a6e75ec62&v=11.240'
        var str2 = null
        var res = a["d"](str, str2)
        console.log(res)
    })
}

```

同 sig 的 frida-trace 方法一样, trace 后有几千行, 这时需要分析哪些函数是关键函数并 hook

![](https://bbs.kanxue.com/upload/attach/202403/983110_TXWBX32SZ3UB7C7.png)

最终的 sp 结果是魔改的 base64, 码表从 A-Za-z0-9+/= 替换成了 A-Za-z0-9-_~ , 这里埋个坑, 尚不清楚传进去的明文和密文有什么联系, 先写到这里, 后续再看看

总结
==

1 出于安全考虑, 本章未提供完整流程, 调试环节省略较多, 只提供大致思路, 具体细节要你自己还原, 相信你也能调试出来.

2 本人写作水平有限, 如有讲解不到位或者讲解错误的地方, 还请各位大佬在评论区多多指教, 共同进步, 也可加本人微信 lyaoyao__i(两个_)

最后
==

微信公众号: 爬虫爬呀爬

![](https://bbs.kanxue.com/upload/attach/202403/983110_X9Y74KHDR5NM3DH.png)

知识星球

![](https://bbs.kanxue.com/upload/attach/202403/983110_2YXUYZVNM3KJ646.png)

如果你觉得这篇文章对你有帮助, 不妨请作者喝一杯咖啡吧!

![](https://bbs.kanxue.com/upload/attach/202403/983110_CHHR3YFQ5UUNW9V.png)

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#逆向分析](forum-161-1-118.htm)