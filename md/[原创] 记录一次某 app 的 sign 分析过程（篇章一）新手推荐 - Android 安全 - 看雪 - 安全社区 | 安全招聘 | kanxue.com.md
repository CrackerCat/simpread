> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282678.htm)

> [原创] 记录一次某 app 的 sign 分析过程（篇章一）新手推荐

> [!tip]
> 
> 前言：某东 sign 有三种算法，这次讲解的是最简单的一个分支。
> 
> 某东算法已经被开源烂了，但是给新入门的朋友进行学习再合适不过了
> 
> 打通任意一个分支都可以拿到正确的，可以请求的 sign

目标样本版本 12.2.2（最新版地址可能会发生偏移以及变化）

jadx 打开样本 开始定位 sign 的生成逻辑

采用搜索![](https://bbs.kanxue.com/upload/attach/202407/967562_RSP3TDGV5QBQHA7.jpg)

x-api-eid-token 的方法来定位

![](https://bbs.kanxue.com/upload/attach/202407/967562_3APU426BRFMVFK7.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/967562_KUJG3MZJ9MKMTCG.jpg)

发现在此处进行引用，从函数功能可以分析出，这是在组包，并返回组装完的字符串

按经验来说，组装成字符串之后就要加密了，我们按 x 查找函数上一层引用

![](https://bbs.kanxue.com/upload/attach/202407/967562_AS8V6JH89D83W3T.jpg)

发现上一层函数也在组包，返回的也是 str 类型

![](https://bbs.kanxue.com/upload/attach/202407/967562_F996387PHC88GPA.jpg)

继续网上寻找，发现同上

![](https://bbs.kanxue.com/upload/attach/202407/967562_KC98XZRDJ7VAU3E.jpg)

继续向上寻找：

![](https://bbs.kanxue.com/upload/attach/202407/967562_A8SYJMRTAWD3VWK.jpg)

最终找到加密位置

![](https://bbs.kanxue.com/upload/attach/202407/967562_SGBDYM2WWVXVCX5.jpg)

```
try {
        String signature = JDHttpTookit.getEngine().getSignatureHandlerImpl().signature(JDHttpTookit.getEngine().getApplicationContext(), queryParameter, str, deviceUUID, property, versionName);
        if (OKLog.D) {
            OKLog.d("Signature", "native signature sucess " + signature);
        }
        if (TextUtils.isEmpty(signature) || (urlParams = getUrlParams(signature)) == null || urlParams.isEmpty()) {
            return;
        }
        for (String str8 : urlParams.keySet()) {
            builder.addQueryParameter(str8, urlParams.get(str8));
        }
    } catch (Exception unused) {
    }

```

发现加密的是一个接口

![](https://bbs.kanxue.com/upload/attach/202407/967562_K32AY7R6QTDEDWF.jpg)

我们要找实现这个类型的方法

implements ISignatureHandler

搜索不到

所以我们直接搜索 ISignatureHandler

发现在这个类里有一定线索

![](https://bbs.kanxue.com/upload/attach/202407/967562_RG7P7F5E7MX66GG.jpg)

在初始化时传入了

![](https://bbs.kanxue.com/upload/attach/202407/967562_CX7CGS593ZH44ES.jpg)

查找调用此函数的位置的方法

![](https://bbs.kanxue.com/upload/attach/202407/967562_TV5AWBENYEFR77F.jpg)

在 initapp 中传入了，我们往上跟入

![](https://bbs.kanxue.com/upload/attach/202407/967562_E423MV9YABBD9J9.jpg)

进入方法查看

![](https://bbs.kanxue.com/upload/attach/202407/967562_KPEYFHDCMQSDRAV.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/967562_253W9R7UZRCQP93.jpg)

原来是使用 new 创建的 所以下次搜索可以使用

```
new ISignatureHandler

```

来搜索定位

![](https://bbs.kanxue.com/upload/attach/202407/967562_MDH5D74SKPF49FC.jpg)

定位到关键函数，接下来进行 hook 并主动调用：

```
function call(){
    Java.perform(function () {
        let BitmapkitUtils = Java.use("com.jingdong.common.utils.BitmapkitUtils");
 
        let context = Java.use("android.app.ActivityThread").currentApplication().getApplicationContext();
        let str = "wareBusiness";
        let str2 ='{"abTest800":true,"acceptPrivacy":true,"avoidLive":false,"bbtf":"","brand":"Redmi","businessType":"","bybt":"","cityCode":72,"cityId":0,"cpsNoTuan":null,"darkModelEnum":3,"districtId":0,"euaf":false,"eventId":"MyHistory_Product","fromType":0,"isDesCbc":true,"isFromOpenApp":true,"latitude":"0.0","lego":true,"longitude":"0.0","model":"Redmi Note 11T Pro","ocrFlag":false,"oneboxChannel":false,"oneboxKeyword":"","oneboxSource":"","openSimilarFlag":"","overseas":0,"pdVersion":"1","personas":null,"pluginVersion":101050,"plusClickCount":0,"plusLandedFatigue":0,"popBusinessType":"","poplayer":false,"productJdv":"-1|kong|t_1000210271_502774|zssc|d36d13b9-61c4-4fdf-b7f2-11dbc28d14dd-p_1999-pr_100746-at_502774-tg_ext_0-00-0-tgx-5050508-3935-20231110|1699610674","provinceId":"0","prstate":"0","refreshMe":null,"searchWareflag":"","selfDelivery":"0","skuId":"48905840961","source_type":"wojing_history","source_value":"","townId":0,"uAddrId":"0","utmMedium":null,"wareInnerSource":"extra.inner.source.init","yrqNew":"1"}'
        let str3 = "789e43b8e08521ee";
        let str4 = "android";
        let str5 = "12.2.2";
 
        let result = BitmapkitUtils.getSignFromJni(context, str, str2, str3, str4, str5);
        console.log("BitmapkitUtils.getSignFromJni result = " + result);
    });
}

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_JV54A46D6D6Q2VM.jpg)

发现疑似是 hash 函数

hook dlsym 函数，得知函数加载的 so

libjdbitmapkit.so

![](https://bbs.kanxue.com/upload/attach/202407/967562_WW37MBTUZZTP44F.jpg)

定位到要分析的函数，由于位数疑似 md5，所以使用龙哥的 findhash 插件，进行寻找

![](https://bbs.kanxue.com/upload/attach/202407/967562_6W8DRZQF5Y8GRZ7.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/967562_S6FJDRGWKCZV95R.jpg)

熟悉的朋友应该认出了，这个就是 md5 运算部分，我们寻找上层引用

![](https://bbs.kanxue.com/upload/attach/202407/967562_4DYE33XGK8CBR9N.jpg)

再次寻找上层引用

![](https://bbs.kanxue.com/upload/attach/202407/967562_T9XGN52T9RTMA9H.jpg)

发现调用位置就在我们目标分析的函数里 (如果没跟到的小伙伴可以打印堆栈)

![](https://bbs.kanxue.com/upload/attach/202407/967562_Y9SVB5M8X77NVPW.jpg)

我们进行 hook 来分析入参

```
(function () {
 
    // @ts-ignore
    function print_arg(addr) {
        try {
            var module = Process.findRangeByAddress(addr);
            if (module != null) return "\n"+hexdump(addr) + "\n";
            return ptr(addr) + "\n";
        } catch (e) {
            return addr + "\n";
        }
    }
 
    // @ts-ignore
    function hook_native_addr(funcPtr, paramsNum) {
        var module = Process.findModuleByAddress(funcPtr);
        try {
            Interceptor.attach(funcPtr, {
                onEnter: function (args) {
                    this.logs = "";
                    this.params = [];
                    // @ts-ignore
                    this.logs=this.logs.concat("So: " + module.name + "  Method: sub_25E0 offset: " + ptr(funcPtr).sub(module.base) + "\n");
                    for (let i = 0; i < paramsNum; i++) {
                        this.params.push(args[i]);
                        this.logs=this.logs.concat("this.args" + i + " onEnter: " + print_arg(args[i]));
                    }
                }, onLeave: function (retval) {
                    for (let i = 0; i < paramsNum; i++) {
                        this.logs=this.logs.concat("this.args" + i + " onLeave: " + print_arg(this.params[i]));
                    }
                    this.logs=this.logs.concat("retval onLeave: " + print_arg(retval) + "\n");
                    console.log(this.logs);
                }
            });
        } catch (e) {
            console.log(e);
        }
    }
    // @ts-ignore
    hook_native_addr(Module.findBaseAddress("libjdbitmapkit.so").add(0x25e0), 0x3);
})();

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_RUYCT2DGKUB69QJ.jpg)

发现入参是一段 base64，解密后是乱码

在 md5 前，明文进行了额外处理：

![](https://bbs.kanxue.com/upload/attach/202407/967562_G3S865MX5JNKHVX.jpg)

v39 是我们要分析的密文，

继续往上跟踪 v39 生成的位置

![](https://bbs.kanxue.com/upload/attach/202407/967562_HJPP7UHTYXACHBD.jpg)

经过 hook 可知

密文由 sub_18c9c 计算而来

![](https://bbs.kanxue.com/upload/attach/202407/967562_CVVAV7SS6VYGP9Q.jpg)

其中两个参数是 rand 随机生成的

在这里决定了 sign 由哪个函数进行加密

![](https://bbs.kanxue.com/upload/attach/202407/967562_38YV392VJEE75BC.jpg)

今天我们分析 case2 里面的加密函数，也是最简单的，后面我会分析剩下两个算法的计算方法

![](https://bbs.kanxue.com/upload/attach/202407/967562_GDD6JJCP3FX3RY7.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/967562_PVUK8ASNMG5WWDQ.jpg)

算法肉眼可见的可以复现，所以我们进行 hook 入参和出参进行分析

```
(function () {
 
    // @ts-ignore
    function print_arg(addr) {
        try {
            var module = Process.findRangeByAddress(addr);
            if (module != null) return "\n"+hexdump(addr) + "\n";
            return ptr(addr) + "\n";
        } catch (e) {
            return addr + "\n";
        }
    }
 
    // @ts-ignore
    function hook_native_addr(funcPtr, paramsNum) {
        var module = Process.findModuleByAddress(funcPtr);
        try {
            Interceptor.attach(funcPtr, {
                onEnter: function (args) {
                    this.logs = "";
                    this.params = [];
                    // @ts-ignore
                    this.logs=this.logs.concat("So: " + module.name + "  Method: sub_6858 offset: " + ptr(funcPtr).sub(module.base) + "\n");
                    for (let i = 0; i < paramsNum; i++) {
                        this.params.push(args[i]);
                        if (i==0){
                            // this.logs=this.logs.concat("this.args" + i + " onEnter: " +args[i].readCString());
                        }
                        else if(i==3){
                            this.logs=this.logs.concat("this.args" + i + " onEnter: " +hexdump(args[3]));
                        }
                        else {
                            this.logs=this.logs.concat("this.args" + i + " onEnter: " + print_arg(args[i]));
 
                        }
                    }
                }, onLeave: function (retval) {
                    for (let i = 0; i < paramsNum; i++) {
                        this.logs=this.logs.concat("this.args" + i + " onLeave: " + print_arg(this.params[i]));
                    }
                    this.logs=this.logs.concat("retval onLeave: " + print_arg(retval) + "\n");
                    console.log(this.logs);
                }
            });
        } catch (e) {
            console.log(e);
        }
    }
    // @ts-ignore
    hook_native_addr(Module.findBaseAddress("libjdbitmapkit.so").add(0x1882C), 0x4);
})();

```

![](https://bbs.kanxue.com/upload/attach/202407/967562_KEY34CDXWYG8TBD.jpg)

不是所有的 call 都能触发这个分支，得多 call 几次

![](https://bbs.kanxue.com/upload/attach/202407/967562_GQEEDJ5DKZ67W3J.jpg)

我们可以看到明文数据了

```
(function () {
 
    // @ts-ignore
    function print_arg(addr) {
        try {
            var module = Process.findRangeByAddress(addr);
            if (module != null) return "\n"+hexdump(addr) + "\n";
            return ptr(addr) + "\n";
        } catch (e) {
            return addr + "\n";
        }
    }
 
    // @ts-ignore
    function hook_native_addr(funcPtr, paramsNum) {
        var module = Process.findModuleByAddress(funcPtr);
        try {
            Interceptor.attach(funcPtr, {
                onEnter: function (args) {
                    this.logs = "";
                    this.params = [];
                    // @ts-ignore
                    this.logs=this.logs.concat("So: " + module.name + "  Method: sub_6858 offset: " + ptr(funcPtr).sub(module.base) + "\n");
                    for (let i = 0; i < paramsNum; i++) {
                        this.params.push(args[i]);
                        if (i==1){
                            this.logs=this.logs.concat("this.args" + i + " onEnter: " +args[i].readCString());
                        }
                        else if(i==3){
                            this.logs=this.logs.concat("this.args" + i + " onEnter: " +hexdump(args[3]));
                        }
                        else {
                            this.logs=this.logs.concat("this.args" + i + " onEnter: " + print_arg(args[i]));
 
                        }
                    }
                }, onLeave: function (retval) {
                    for (let i = 0; i < paramsNum; i++) {
                        this.logs=this.logs.concat("this.args" + i + " onLeave: " + print_arg(this.params[i]));
                    }
                    this.logs=this.logs.concat("retval onLeave: " + print_arg(retval) + "\n");
                    console.log(this.logs);
                }
            });
        } catch (e) {
            console.log(e);
        }
    }
    // @ts-ignore
    hook_native_addr(Module.findBaseAddress("libjdbitmapkit.so").add(0x1882C), 0x4);
})();

```

修改脚本 拿到 arg1 的 key![](https://bbs.kanxue.com/upload/attach/202407/967562_V8XVVBF7YE7ARE8.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/967562_985QTZPMMSQYPT6.jpg)

![](https://bbs.kanxue.com/upload/attach/202407/967562_5WJ3673HVJKU5Z5.jpg)

afcb2afb1f349bed06555aef7fd47cecbec115c6083eafa51f608df10bdd350bab2a21cb1ffb6cb7df2b9dea43cf07ccaaf12ffd1b378cf126678edb57840d1ebbcb2184c52ca2ee265595e144cf35c4aff728cb08ef5ee11f658f9a088435f66bf03ef931275eb9df4382dc7bd335f66bf031cb0c2991f2284586e873840dcc6b82eefa1c2da0a1f7134ba430c5401ea2d12bfc08ed76b6ef1d4bdb47de5013adb0e688f7ed9ff728bfb4cc43cb41cc63fc31c437ef5eeb1e63b0c57dce7c368efc31c5c5c56f93df55b6eb42d4400dbddf20baddf358a122668ede309c790b95c12184c520a2e42b6596dc309c3517a2de15cb1f2aaef814419be772df7a1e.... 省略

使用 c 语言进行复现，发现结果一致，接下来把结果 from hex 再 base64

把 base64 进行 md5 加密 即可得到京东的 sign

```
void TenSeattosEncrypt(char* input, int input_len)
 
{
 
    int v4, v5;
 
    char v6;
 
    const char* TenSeattos_key = "80306f4370b39fd5630ad0529f77adb6";
 
    unsigned char table[0x10] = { 0x37, 0x92, 0x44, 0x68, 0xA5, 0x3D, 0xCC, 0x7F, 0xBB, 0xF, 0xD9, 0x88, 0xEE, 0x9A, 0xE9, 0x5A };
 
    for (int i = 0; i != input_len; ++i) {
 
        v4 = i & 7;
 
        v5 = table[i & 0xF];
 
        v6 = (v5 + (*(unsigned char*)(input + i) ^ *(unsigned char*)(TenSeattos_key + v4) ^ table[i & 0xF])) ^ table[i & 0xF];
 
        *(unsigned char*)(input + i) = v6;
 
        *(unsigned char*)(input + i) = *(unsigned char*)(TenSeattos_key + v4) ^ v6;
 
    }
 
}

```

至此，我们已经完成了最容易的一个分支的京东 sign 的计算

下一篇文章将分析另外的两个加密过程，涉及到 unidbg/unicorn 的使用，记录算法还原的过程

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#逆向分析](forum-161-1-118.htm)