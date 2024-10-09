> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283709.htm)

> [原创] 记录一次 **App H5 逆向

1. 调试前第一部曲 使用 frida 打开 webview 调试
---------------------------------

*   d * 版本 5.31.1

根据历来活动界面经验，各大 app 使用 webview 进行活动的展开

使用 frida hookd*app 开启 webview 调试

```
console.log("脚本加载成功");
function main(){
    Java.perform(function(){
        var WebView = Java.use('android.webkit.WebView');
        WebView.$init.overload('android.content.Context').implementation = function(a){
            var result = this.$init(a);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
        WebView.$init.overload('android.content.Context', 'android.util.AttributeSet').implementation = function(a,b){
            var result = this.$init(a,b);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
        WebView.$init.overload('android.content.Context', 'android.util.AttributeSet', 'int').implementation = function(a,b,c){
            var result = this.$init(a,b,c);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
        WebView.$init.overload('android.content.Context', 'android.util.AttributeSet', 'int', 'int').implementation = function(a,b,c,d){
            var result = this.$init(a,b,c,d);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
        WebView.$init.overload('android.content.Context', 'android.util.AttributeSet', 'int', 'boolean').implementation = function(a,b,c,d){
            var result = this.$init(a,b,c,d);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
        WebView.$init.overload('android.content.Context', 'android.util.AttributeSet', 'int', 'java.util.Map', 'boolean').implementation = function(a,b,c,d,e){
            var result = this.$init(a,b,c,d,e);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
        WebView.$init.overload('android.content.Context', 'android.util.AttributeSet', 'int', 'int', 'java.util.Map', 'boolean').implementation = function(a,b,c,d,e,f){
            var result = this.$init(a,b,c,d,e,f);
            this.setWebContentsDebuggingEnabled(true);
            return result;
        }
    });
}
setImmediate(main);
```

*   浏览器输入 chrome://inspect/#devices 发现已经有目标调试网页出现

![](https://bbs.kanxue.com/upload/attach/202409/967562_PRFFUWNGGCX5S4Y.png)

![](https://bbs.kanxue.com/upload/attach/202409/967562_E54RBAYX7DJEWWU.png)

> [!NOTE]
> 
> 在逆向之前要确定好百分百加密的接口
> 
> 首先我们要考虑接口的可触发次数，以及触发是否有其他杂乱接口干扰
> 
> 以喂鱼接口为例子，我们只能触发有限次数
> 
> 但是以任务菜单为例子，我们能触发多次，并且没有其他杂包

2. 调试前第二部曲，过掉 ob 内自带无限 debugger
-------------------------------

![](https://bbs.kanxue.com/upload/attach/202409/967562_7H494RHVMWJK4JS.png)

点击后触发了无限 debugger 逻辑，考虑过掉该逻辑

1.  可以考虑做抓包工具做远程替换，将'debu', 'gger'改为'',''即可
    
    `['constructor'](_0x2f614e['uzRAM']('debu', 'gger'))['call']('action');`
    
2.  这里我们选择手动过掉
    

![](https://bbs.kanxue.com/upload/attach/202409/967562_Y56AJ3XUYUP3ZD6.png)

> [!TIP]
> 
> OB 混淆总是给人一种此地无银三百两的感觉，往下翻文件轻松的发现了加密逻辑位置，基本到这里就定位结束了。每次看到 OB 一些混淆，就知道接下来的任务简单了，攻破这个混淆就可以了

3. 开始确定调试目标
-----------

目标分析： data 内加密字段

首先猜测是否为 RSA、AES、DES

![](https://bbs.kanxue.com/upload/attach/202409/967562_UR67D54GMD43VQX.png)

*   发送内容

```
data: vyOviR13Aj2v2YfHwBzt9AZALCv7Tb7GGgbFy3bwIVlp/p1z8bInxbuAhqQlw3tuopM5OBzVnD0oyAX6CttfpA​59E9F16BA7DA911473B34942199E63906BB47BF669E930A449417783275F7A24F49E5456B23BE2C78286D67D85DB552D
```

*   返回内容

```
0 DBBB668BAE191E448D975EC8321C631029ACCCEFF60F02B75B45731272A242E1500984E8A3794AF305613918247A9DCBAA62BC3A91B343D953B4E0574724B90F139F78CF3EBA716FA3B408FA0F25770285461FBA3481769948BD67F36A8584E9657B6111143A4F38AC89AB6975E61776964986E9A2D72F2365B9D6F0CF1A1BBC368CC632290B352DA22E8D412F11692F3FB683BBC03C650910E3373E74FA1C92D5881C594E9F32E259742E4F55AC4B06CE0A33054A9EEFC2AA6781F0CB0B7549007CC71053E3C042CEA2BE34ABDB228D3817067CEC998328EAA5E05C043D193D80A04F1CEC7E22BD396A4.........省略n多
```

*   由于发送以及返回都为加密内容，猜测为对称加密

4. 确定加密位置
---------

*   观察启动器调用堆栈，发现调用堆栈极其复杂

![](https://bbs.kanxue.com/upload/attach/202409/967562_BQRDHUATNGWZXNU.png)

*   所以我们开始使用**一种技巧** 观察文件名来确定业务逻辑主要的位置，

![](https://bbs.kanxue.com/upload/attach/202409/967562_8PY3DUX9UPU5KUG.png)

首先观测第一个 js 文件，我们可以将其标记为大环境框架 js，内部有多种反射调用，疑似为异步调用框架。

![](https://bbs.kanxue.com/upload/attach/202409/967562_A3ZDCZ6HUCMGP7T.png)

由于这一层调用再 fetch 上面，所以有很大疑点，我们打断点进行观察

![](https://bbs.kanxue.com/upload/attach/202409/967562_WVEEWKME2QTTN8C.png)

![](https://bbs.kanxue.com/upload/attach/202409/967562_JYX6PDCCNDNJ9EU.png)

在上面发现是 content-type 等头部设置，我们在这里打上断点，观察这里的数据是否进行了加密。

> [!TIP]
> 
> 我认为 JS 逆向大部分跟参的过程都是在逐渐缩短自己的范围，如果我能确定这里已经生成了加密参数，那么在我接下来的逆向中，将会减少很多的工作量

![](https://bbs.kanxue.com/upload/attach/202409/967562_KB7S6TG7XN6KH4R.png)

由于 ob 混淆内部分逻辑在调试时会崩溃，所以采取去混淆行为。

> [!tip]
> 
> 新发现，在浏览器使用本地替换也可在远程调试 webview 的 js 中生效

![](https://bbs.kanxue.com/upload/attach/202409/967562_JR9U3MG4GEW9QVW.png)

确定接口名称和目标接口名称一致

在这时发现 data 并没有被加密，所以打在下一行，并且观察是否有网络请求发出

![](https://bbs.kanxue.com/upload/attach/202409/967562_KH2S5WB3ZPRNH83.png)

发现加密消失了（蜜罐），我们再次进行尝试，并观察密文生成位置

```
"https://app.dewu.com/hacking-fish/v1/task/list?data=i4aGW0KBO1Rn6MBDfu%2FthRGTH4WL26tsRG5epeKND4xFVZdsJvwMJt9xJbAMY4hi09HOCqxTRQCRrEWqcsKt1w%E2%80%8BC74F514DAA73D51BBB723CC4B8FDEA29168327DD99B4BA5C1A0CA918F74375F359D3E405D4BBAFEE29E9CB872CA6CA14"
```

发现在 const v = yield c(g, p) 这里的时候 我们的加密逻辑已经完成了，需要往上继续跟踪

![](https://bbs.kanxue.com/upload/attach/202409/967562_BGD4T8QHY62HDUP.png)

我们跨过工具代码部分逻辑，继续分析。

![](https://bbs.kanxue.com/upload/attach/202409/967562_HZD4JSW7ETUVA5P.png)

发现这里已经生成了

![](https://bbs.kanxue.com/upload/attach/202409/967562_UYEMEBSDJ8RXZPR.png)

由于堆栈跟踪断层，我们发现在请求函数的上层作用域有疑似参数处理过程 打好断点重新触发

![](https://bbs.kanxue.com/upload/attach/202409/967562_5TU59XBUBPEXPGV.png)

发现了生成前的 param 对象

![](https://bbs.kanxue.com/upload/attach/202409/967562_DQXTXZRM8SHUHEA.png)

观察本作用域触发后的上一层，发现正在疑似拼接函数，跳出函数进行单步跟踪

![](https://bbs.kanxue.com/upload/attach/202409/967562_P5TRDRJATTYYFGX.png)

### 定位方式 1 硬跟异步，善用工具框架公用逻辑部分

时刻留意 data 动向进行继续单步跟踪（对付异步的比较不错的方法）

![](https://bbs.kanxue.com/upload/attach/202409/967562_4Z3JTRCJRAP4NFH.png)

![](https://bbs.kanxue.com/upload/attach/202409/967562_34CQDBZPXMF34WW.png)

切记一定要用这个跟，其他的不要用

![](https://bbs.kanxue.com/upload/attach/202409/967562_QAE24JFJUCR7KTP.png)

在 yield 这里狂按 f9

![](https://bbs.kanxue.com/upload/attach/202409/967562_23URXZW673CD395.png)

```
return new (n || (n = Promise))((function(o, i) {
              function a(t) {
                  try {
                      f(r.next(t))
                  } catch (t) {
                      i(t)
                  }
              }
              function c(t) {
                  try {
                      f(r.throw(t))
                  } catch (t) {
                      i(t)
                  }
              }
```

在这里发现异步实现框架，进行参数过滤

![](https://bbs.kanxue.com/upload/attach/202409/967562_ZFW59XY2MJXPM6G.png)

当 t 为关键参数时候，需要使用上面的 f9 跟进关键函数

![](https://bbs.kanxue.com/upload/attach/202409/967562_KF2TDVPCTN3NESW.png)

在这里就可以使用条件断点，来过滤想要跟入的异步函数

![](https://bbs.kanxue.com/upload/attach/202409/967562_AQ6MGW7M5ZUUVWG.png)

我们发现，我们已经成功跟入了一个新的业务逻辑 js（从名字观看）

![](https://bbs.kanxue.com/upload/attach/202409/967562_68QAB3W82EBUHQQ.png)

从谷歌自带的提示中发现，貌似在循环执行五个函数

貌似是在对请求的数据进行处理，**这也是我们需要的**

![](https://bbs.kanxue.com/upload/attach/202409/967562_A349Y8YTWGMB6CM.png)

（这里没法看到函数列表 一个一个去打断点，只能回到

![](https://bbs.kanxue.com/upload/attach/202409/967562_KF2TDVPCTN3NESW.png)

继续单步跟入查看，查看位置都是一样的

![](https://bbs.kanxue.com/upload/attach/202409/967562_58F68DUZVMVFSUP.png)

直到跟入核心函数，对数据处理的部分

![](https://bbs.kanxue.com/upload/attach/202409/967562_PVBPEGATKN2RTGY.png)

S.Fun110 就是我们的目标函数

到此刻我们定位成功

### 定位方式 2 巧用代理，定位加密参数生成位置（推荐）

![](https://bbs.kanxue.com/upload/attach/202409/967562_45PKS4W6N6SEHSQ.png)

首先我们先找到一个时机点，这个时机点应该是加密参数生成前，在请求基本生成完成后

请求基本生成完成后是指什么时刻？给出大家一个例子

![](https://bbs.kanxue.com/upload/attach/202409/967562_J3G9G48QP4M7745.png)

在这里创建了基本请求后，然后有未加密的数据。

当然在要代理的地方我们不用过于担心设置的是否正确，我们可以正确的拿到承接关系

比如我们代理

```
{
                            path: this.listPath,
                            method: "get",
                            params: t
                        }
```

这个对象，但是对象赋值给了多个对象，我们对赋值的对象均进行跟踪

对每个值的以及每个值的拷贝进行跟踪

```
new Proxy(obj,
        {set: function(obj, prop, value)
            {
                console.log("set--->",obj, prop, value);
                debugger
                return Reflect.set(...arguments);},
            get: function(obj, prop)
            {
                console.log("get--->",obj, prop);
                debugger
                return Reflect.get(...arguments);
            }
        }
    )
```

所以在此处我们设置

![](https://bbs.kanxue.com/upload/attach/202409/967562_F8SPNVQ7HV6E7VY.png)

来观察 e.data 属性的获取情况，重点关注 param 的情况

![](https://bbs.kanxue.com/upload/attach/202409/967562_26TFEUARJPXCP2P.png)

![](https://bbs.kanxue.com/upload/attach/202409/967562_ANYU33D7Q8DE4WT.png)

这里出现了我们说的拷贝情况，把 o.data 一起代理上

![](https://bbs.kanxue.com/upload/attach/202409/967562_DM2WXVZGQ95EKR2.png)

观察获取到的属性，有 param 我们就观察上层堆栈

![](https://bbs.kanxue.com/upload/attach/202409/967562_S5ETXWBPUEBGAUE.png)

![](https://bbs.kanxue.com/upload/attach/202409/967562_2M6BG3W25TYSGTF.png)

在这里发现 v.data 也需要设置

![](https://bbs.kanxue.com/upload/attach/202409/967562_57SDYMQ4XYVZ8ZM.png)

发现在获取 param 时候也成功定位到了加密函数

两种定位方式都需要一定调试技巧，在跟异步的时候都可以对流程的理清有一定帮助

5. 进行算法还原
---------

```
'value': function(_0x29b3a1, _0x105de1, _0x46790f, _0x5db7e4) {
                 var _0x495033 = {
                     'jnDMy': function(_0x399cc7, _0x21f72e) {
                         return _0x399cc7 - _0x21f72e;
                     },
                     'kREPA': function(_0x27b72a, _0x456f3a) {
                         return _0x27b72a >> _0x456f3a;
                     },
                     'jbiNJ': function(_0x1694e5, _0x37a70c) {
                         return _0x1694e5 + _0x37a70c;
                     },
                     'pgDxk': function(_0x1f7057, _0x573714) {
                         return _0x35594c['aMZtC'](_0x1f7057, _0x573714);
                     },
                     "\u0077\u0056\u0065\u004b\u004e": function(_0x2beba4) {
                         return _0x35594c['Hwkfd'](_0x2beba4);
                     },
                     'FtodS': function(_0x18aecc, _0x493efd) {
                         return _0x18aecc - _0x493efd;
                     },
                     'dNODL': function(_0x1aabb7, _0xce4941) {
                         return _0x1aabb7 < _0xce4941;
                     },
                     'eovOE': function(_0x20e70a, _0x57ec84) {
                         return _0x35594c['XTSbV'](_0x20e70a, _0x57ec84);
                     },
                     'JGYXo': function(_0x4a600c, _0x471cfd) {
                         return _0x35594c['HwQGE'](_0x4a600c, _0x471cfd);
                     },
                     'DKWdp': function(_0x34df49, _0x1efc82) {
                         return _0x35594c['cqNxF'](_0x34df49, _0x1efc82);
                     },
                     'uLcaf': function(_0x896ecd, _0x3cdc64) {
                         return _0x35594c['Haher'](_0x896ecd, _0x3cdc64);
                     },
                     'fFTKY': function(_0x37b5b4, _0x373c04) {
                         return _0x37b5b4 >= _0x373c04;
                     },
                     'NAEpD': function(_0x68aa83, _0x460689) {
                         return _0x35594c['aLSFO'](_0x68aa83, _0x460689);
                     },
                     'NCrAQ': function(_0x1fe228, _0x57c621) {
                         return _0x1fe228 % _0x57c621;
                     },
                     'Rbmel': function(_0x34d831, _0x4e6a74) {
                         return _0x34d831 / _0x4e6a74;
                     },
                     'siLbZ': function(_0x104496, _0x5bdbaf) {
                         return _0x104496 & _0x5bdbaf;
                     },
                     'Uzghn': function(_0x4454e1, _0x2f9444) {
                         return _0x4454e1 - _0x2f9444;
                     },
                     'iImdx': function(_0x11bc2f, _0x4039cb) {
                         return _0x11bc2f + _0x4039cb;
                     },
                     'bboWb': function(_0x2a5bac, _0x982394) {
                         return _0x2a5bac != _0x982394;
                     },
                     'Cmwqf': function(_0x557ece, _0x3778c7) {
                         return _0x557ece - _0x3778c7;
                     },
                     'RWuoe': function(_0x3d5eaa, _0x5dad97) {
                         return _0x3d5eaa * _0x5dad97;
                     },
                     'YqrBL': function(_0x2a7355) {
                         return _0x2a7355();
                     },
                     'IDltp': function(_0x21a13a) {
                         return _0x21a13a();
                     },
                     'fklWl': function(_0x166bac, _0x98d225) {
                         return _0x166bac(_0x98d225);
                     },
                     'ZhTHm': function(_0x24cc52) {
                         return _0x24cc52();
                     },
                     'tbWeE': function(_0x5a1e1d, _0x52bb9e) {
                         return _0x5a1e1d >= _0x52bb9e;
                     },
                     'nUvFD': function(_0x4ee77b, _0x436c41) {
                         return _0x4ee77b == _0x436c41;
                     },
                     'COxtG': function(_0x1433c5, _0x3b08e8) {
                         return _0x1433c5 == _0x3b08e8;
                     },
                     'maQEK': function(_0x2600f8, _0x5a5b55) {
                         return _0x2600f8 > _0x5a5b55;
                     },
                     'Pteet': function(_0xf9798b, _0x132e77) {
                         return _0xf9798b >> _0x132e77;
                     },
                     'mxZIe': function(_0xf2e1e4, _0xd1632a) {
                         return _0xf2e1e4 >= _0xd1632a;
                     },
                     'bcKuF': function(_0x870c5e, _0x1a7bf3) {
                         return _0x870c5e < _0x1a7bf3;
                     },
                     "\u0059\u0050\u0064\u0073\u006c": function(_0x1dc8d0, _0x271b0b) {
                         return _0x35594c['gUBLr'](_0x1dc8d0, _0x271b0b);
                     },
                     'uniSO': function(_0x280a53, _0x2843ba) {
                         return _0x280a53 & _0x2843ba;
                     },
                     'hhxpV': function(_0x43fda3, _0x3ef904) {
                         return _0x43fda3 < _0x3ef904;
                     },
                     'nUORC': function(_0x49652d, _0x303afc) {
                         return _0x49652d < _0x303afc;
                     },
                     'blZcG': function(_0x34a5e5, _0x47dfc3) {
                         return _0x34a5e5 + _0x47dfc3;
                     },
                     'nDhum': function(_0x20cee7, _0x3c40c1) {
                         return _0x20cee7(_0x3c40c1);
                     },
                     'PRHhW': function(_0x4ca3a6, _0x2fd0b0) {
                         return _0x4ca3a6 >> _0x2fd0b0;
                     },
                     'YMZWA': function(_0x1b56d9, _0x4e7143) {
                         return _0x1b56d9 | _0x4e7143;
                     },
                     'xmGMu': function(_0x3668cf, _0x24817e) {
                         return _0x3668cf > _0x24817e;
                     },
                     'wkVpc': function(_0x46a0e7, _0x41ee8a) {
                         return _0x35594c['ohatu'](_0x46a0e7, _0x41ee8a);
                     },
                     'IUSwf': 'unsupported\x20PKCS#8\x20public\x20key\x20hex',
                     'Glggr': function(_0x2d00ef, _0x4f6fe4) {
                         return _0x2d00ef !== _0x4f6fe4;
                     }
                 };
                 var _0x4879b7, _0x4a8f01, _0x1a0eab, _0x5da258, _0x2bd525, _0xe412e2, _0x345581, _0x4dfef5, _0x483389, _0x3bf831, _0x1c4a22, _0x4595dc, _0x3291c5, _0x10fab7 = _0x42a8e8(0x30, 0x10), _0x3b1c65 = '', _0x4a9b51 = '0', _0x11dd4c = _0x29b3a1;
                 if ("teg".split("").reverse().join("") === (_0x105de1 || 'post')['toLocaleLowerCase']())
                     try {
                         var _0x1e5a73 = []
                           , _0x3e45e7 = JSON['parse'](_0x29b3a1);
                         Object['keys'](_0x3e45e7)['map'](function(_0x9b096) {
                             return _0x1e5a73['push'](''['concat'](_0x9b096, '=')['concat'](encodeURIComponent(_0x3e45e7[_0x9b096]))),
                             _0x9b096;
                         }),
                         _0x11dd4c = _0x1e5a73['join']('&');
                     } catch (_0x4997f8) {
                         console['log'](_0x35594c['MSvQu']);
                     }
                 if (!_0x5db7e4 && '0' === String(_0x257145))
                     return {
                         'a': _0x10fab7,
                         'b': _0x42a8e8(0x30, 0x10),
                         'c': ''['concat'](_0x4a9b51, ',')['concat'](_0x128360),
                         'd': _0x29b3a1,
                         'e': _0x42a8e8(0x30, 0x10)
                     };
                 try {
                     var _0x173191, _0x59ca79 = function(_0x1cabba, _0x15b456, _0x4acfca) {
                         _0x337f99['dnHfa'](null, _0x1cabba) && (_0x337f99['FOejE'](_0x337f99['upliq'], typeof _0x1cabba) ? this['fromNumber'](_0x1cabba, _0x15b456, _0x4acfca) : null == _0x15b456 && _0x337f99['TVpeu']('string', typeof _0x1cabba) ? this['fromString'](_0x1cabba, 0x100) : this['fromString'](_0x1cabba, _0x15b456));
                     }, _0x400e8f = function() {
                         return new _0x59ca79(null);
                     }, _0x316e10 = function(_0x2569d6) {
                         var _0x528ed4, _0x5217de, _0x1de34f, _0x20f23b = '', _0x191816 = 0x0;
                         for (_0x528ed4 = 0x0; _0x528ed4 < _0x2569d6['length'] && _0x2569d6['charAt'](_0x528ed4) != _0xe72856; ++_0x528ed4)
                             _0x337f99['IOuwo'](_0x1de34f = _0x16d26a['indexOf'](_0x2569d6['charAt'](_0x528ed4)), 0x0) || (0x0 == _0x191816 ? (_0x20f23b += _0x4d7989(_0x1de34f >> 0x2),
                             _0x5217de = _0x337f99['SjWnk'](0x3, _0x1de34f),
                             _0x191816 = 0x1) : 0x1 == _0x191816 ? (_0x20f23b += _0x4d7989(_0x337f99['iVTra'](_0x5217de, 0x2) | _0x1de34f >> 0x4),
                             _0x5217de = _0x337f99['SjWnk'](0xf, _0x1de34f),
                             _0x191816 = 0x2) : 0x2 == _0x191816 ? (_0x20f23b += _0x337f99['qEDXs'](_0x4d7989, _0x5217de),
                             _0x20f23b += _0x4d7989(_0x1de34f >> 0x2),
                             _0x5217de = _0x337f99['OJQhi'](0x3, _0x1de34f),
                             _0x191816 = 0x3) : (_0x20f23b += _0x4d7989(_0x5217de << 0x2 | _0x1de34f >> 0x4),
                             _0x20f23b += _0x4d7989(_0x337f99['KuSZA'](0xf, _0x1de34f)),
                             _0x191816 = 0x0));
                         return 0x1 == _0x191816 && (_0x20f23b += _0x4d7989(_0x337f99['iVTra'](_0x5217de, 0x2))),
                         _0x20f23b;
                     }, _0x4d7989 = function(_0x579c8f) {
                         return _0x5ca101['charAt'](_0x579c8f);
                     }, _0x10aeb0 = function(_0x428ff9, _0x276953) {
                         var _0x1d865e = _0x12330c[_0x428ff9['charCodeAt'](_0x276953)];
                         return null == _0x1d865e ? -0x1 : _0x1d865e;
                     }, _0x271122 = function(_0x244e5b) {
                         var _0x1792c6, _0x250c55 = 0x1;
                         return _0x337f99['dnHfa'](0x0, _0x1792c6 = _0x337f99['PlLEe'](_0x244e5b, 0x10)) && (_0x244e5b = _0x1792c6,
                         _0x250c55 += 0x10),
                         0x0 != (_0x1792c6 = _0x244e5b >> 0x8) && (_0x244e5b = _0x1792c6,
                         _0x250c55 += 0x8),
                         _0x337f99['PnfLG'](0x0, _0x1792c6 = _0x244e5b >> 0x4) && (_0x244e5b = _0x1792c6,
                         _0x250c55 += 0x4),
                         0x0 != (_0x1792c6 = _0x244e5b >> 0x2) && (_0x244e5b = _0x1792c6,
                         _0x250c55 += 0x2),
                         0x0 != (_0x1792c6 = _0x244e5b >> 0x1) && (_0x244e5b = _0x1792c6,
                         _0x250c55 += 0x1),
                         _0x250c55;
                     }, _0x1de423 = function(_0x470164) {
                         this['m'] = _0x470164;
                     }, _0x5033d5 = function(_0x1c3314) {
                         this['m'] = _0x1c3314,
                         this['mp'] = _0x1c3314['invDigit'](),
                         this['mpl'] = 0x7fff & this['mp'],
                         this['mph'] = this['mp'] >> 0xf,
                         this['um'] = _0x495033['jnDMy'](0x1 << _0x1c3314['DB'] - 0xf, 0x1),
                         this['mt2'] = 0x2 * _0x1c3314['t'];
                     }, _0xa66064 = function() {
                         this['i'] = 0x0,
                         this['j'] = 0x0,
                         this['S'] = new Array();
                     }, _0x4f62c5 = function() {
                         !function(_0x39abac) {
                             _0x171384[_0x5c9259++] ^= 0xff & _0x39abac,
                             _0x171384[_0x5c9259++] ^= _0x39abac >> 0x8 & 0xff,
                             _0x171384[_0x5c9259++] ^= _0x39abac >> 0x10 & 0xff,
                             _0x171384[_0x5c9259++] ^= _0x495033['kREPA'](_0x39abac, 0x18) & 0xff,
                             _0x5c9259 >= _0x446f14 && (_0x5c9259 -= _0x446f14);
                         }(new Date()['getTime']());
                     }, _0x27cf6d = function() {
                         if (null == _0x42b025) {
                             for (_0x4f62c5(),
                             (_0x42b025 = new _0xa66064())['init'](_0x171384),
                             _0x5c9259 = 0x0; _0x5c9259 < _0x171384['length']; ++_0x5c9259)
                                 _0x171384[_0x5c9259] = 0x0;
                             _0x5c9259 = 0x0;
                         }
                         return _0x42b025['next']();
                     }, _0x13d83e = function() {}, _0x47f03c = function() {
                         this['n'] = null,
                         this['e'] = 0x0,
                         this['d'] = null,
                         this['p'] = null,
                         this['q'] = null,
                         this['dmp1'] = null,
                         this['dmq1'] = null,
                         this['coeff'] = null;
                     };
                     'Microsoft\x20Internet\x20Explorer' == navigator['appName'] ? (_0x59ca79['prototype']['am'] = function(_0x997169, _0x13e72e, _0x30fb98, _0x4d34cb, _0x51786c, _0x452448) {
                         for (var _0x1345c7 = 0x7fff & _0x13e72e, _0x4c73ab = _0x13e72e >> 0xf; --_0x452448 >= 0x0; ) {
                             var _0x547e23 = 0x7fff & this[_0x997169]
                               , _0x190547 = _0x337f99['CkSfX'](this[_0x997169++], 0xf)
                               , _0x3aefc8 = _0x4c73ab * _0x547e23 + _0x190547 * _0x1345c7;
                             _0x51786c = _0x337f99['CwSBC'](_0x337f99['PlLEe'](_0x547e23 = _0x337f99['cccOp'](_0x1345c7 * _0x547e23 + ((0x7fff & _0x3aefc8) << 0xf), _0x30fb98[_0x4d34cb]) + (0x3fffffff & _0x51786c), 0x1e) + _0x337f99['PlLEe'](_0x3aefc8, 0xf), _0x337f99['UBVQR'](_0x4c73ab, _0x190547)) + (_0x51786c >>> 0x1e),
                             _0x30fb98[_0x4d34cb++] = _0x337f99['OJQhi'](0x3fffffff, _0x547e23);
                         }
                         return _0x51786c;
                     }
                     ,
                     _0x173191 = 0x1e) : _0x35594c['VTQiX']("epacsteN".split("").reverse().join(""), navigator['appName']) ? (_0x59ca79['prototype']['am'] = function(_0x53ad28, _0x5b1d73, _0xe60088, _0x574d0d, _0x40bb42, _0xb27fd2) {
                         for (; --_0xb27fd2 >= 0x0; ) {
                             var _0x54c12e = _0x495033['jbiNJ'](_0x5b1d73 * this[_0x53ad28++], _0xe60088[_0x574d0d]) + _0x40bb42;
                             _0x40bb42 = Math['floor'](_0x54c12e / 0x4000000),
                             _0xe60088[_0x574d0d++] = _0x495033['pgDxk'](0x3ffffff, _0x54c12e);
                         }
                         return _0x40bb42;
                     }
                     ,
                     _0x173191 = 0x1a) : (_0x59ca79['prototype']['am'] = function(_0xa394d3, _0x1ac0ad, _0x5a4adf, _0x2727f7, _0x25ef30, _0xb77c06) {
                         for (var _0x3f16d3 = 0x3fff & _0x1ac0ad, _0x4b2a4f = _0x1ac0ad >> 0xe; --_0xb77c06 >= 0x0; ) {
                             var _0x2096ad = _0x337f99['YirMV'](0x3fff, this[_0xa394d3])
                               , _0x383759 = this[_0xa394d3++] >> 0xe
                               , _0x1d1f92 = _0x337f99['CwSBC'](_0x4b2a4f * _0x2096ad, _0x383759 * _0x3f16d3);
                             _0x25ef30 = ((_0x2096ad = _0x3f16d3 * _0x2096ad + (_0x337f99['XFjfj'](0x3fff, _0x1d1f92) << 0xe) + _0x5a4adf[_0x2727f7] + _0x25ef30) >> 0x1c) + (_0x1d1f92 >> 0xe) + _0x4b2a4f * _0x383759,
                             _0x5a4adf[_0x2727f7++] = 0xfffffff & _0x2096ad;
                         }
                         return _0x25ef30;
                     }
                     ,
                     _0x173191 = 0x1c);
                     var _0x16d26a = "/+9876543210zyxwvutsrqponmlkjihgfedcbaZYXWVUTSRQPONMLKJIHGFEDCBA".split("").reverse().join("")
                       , _0xe72856 = '=';
                     _0x59ca79['prototype']['DB'] = _0x173191,
                     _0x59ca79['prototype']['DM'] = _0x35594c['Haher'](0x1, _0x173191) - 0x1,
                     _0x59ca79['prototype']['DV'] = 0x1 << _0x173191,
                     _0x59ca79['prototype']['FV'] = Math['pow'](0x2, 0x34),
                     _0x59ca79['prototype']['F1'] = 0x34 - _0x173191,
                     _0x59ca79['prototype']['F2'] = _0x35594c['qQlTX'](0x2, _0x173191) - 0x34;
                     var _0x278663, _0x31bca6, _0x5ca101 = '0123456789abcdefghijklmnopqrstuvwxyz', _0x12330c = new Array();
                     for (_0x278663 = '0'['charCodeAt'](0x0),
                     _0x31bca6 = 0x0; _0x31bca6 <= 0x9; ++_0x31bca6)
                         _0x12330c[_0x278663++] = _0x31bca6;
                     for (_0x278663 = 'a'['charCodeAt'](0x0),
                     _0x31bca6 = 0xa; _0x35594c['gUBLr'](_0x31bca6, 0x24); ++_0x31bca6)
                         _0x12330c[_0x278663++] = _0x31bca6;
                     for (_0x278663 = 'A'['charCodeAt'](0x0),
                     _0x31bca6 = 0xa; _0x31bca6 < 0x24; ++_0x31bca6)
                         _0x12330c[_0x278663++] = _0x31bca6;
                     _0x1de423['prototype']['convert'] = function(_0x1d9733) {
                         return _0x1d9733['s'] < 0x0 || _0x1d9733['compareTo'](this['m']) >= 0x0 ? _0x1d9733['mod'](this['m']) : _0x1d9733;
                     }
                     ,
                     _0x1de423['prototype']['revert'] = function(_0x1a46be) {
                         return _0x1a46be;
                     }
                     ,
                     _0x1de423['prototype']['reduce'] = function(_0x143f01) {
                         _0x143f01['divRemTo'](this['m'], null, _0x143f01);
                     }
                     ,
                     _0x1de423['prototype']['mulTo'] = function(_0x12a399, _0x5b95c6, _0x12a816) {
                         _0x12a399['multiplyTo'](_0x5b95c6, _0x12a816),
                         this['reduce'](_0x12a816);
                     }
                     ,
                     _0x1de423['prototype']['sqrTo'] = function(_0x35a5b5, _0x3a2efe) {
                         _0x35a5b5['squareTo'](_0x3a2efe),
                         this['reduce'](_0x3a2efe);
                     }
                     ,
                     _0x5033d5['prototype']['convert'] = function(_0x36c1f1) {
                         var _0x4ef797 = _0x400e8f();
                         return _0x36c1f1['abs']()['dlShiftTo'](this['m']['t'], _0x4ef797),
                         _0x4ef797['divRemTo'](this['m'], null, _0x4ef797),
                         _0x36c1f1['s'] < 0x0 && _0x4ef797['compareTo'](_0x59ca79['ZERO']) > 0x0 && this['m']['subTo'](_0x4ef797, _0x4ef797),
                         _0x4ef797;
                     }
                     ,
                     _0x5033d5['prototype']['revert'] = function(_0x10be0c) {
                         var _0x32d8cc = _0x495033['wVeKN'](_0x400e8f);
                         return _0x10be0c['copyTo'](_0x32d8cc),
                         this['reduce'](_0x32d8cc),
                         _0x32d8cc;
                     }
                     ,
                     _0x5033d5['prototype']['reduce'] = function(_0x5e09a8) {
                         for (; _0x5e09a8['t'] <= this['mt2']; )
                             _0x5e09a8[_0x5e09a8['t']++] = 0x0;
                         for (var _0x8bb497 = 0x0; _0x8bb497 < this['m']['t']; ++_0x8bb497) {
                             var _0x3934d6 = 0x7fff & _0x5e09a8[_0x8bb497]
                               , _0x245588 = _0x337f99['VUyyp'](_0x3934d6 * this['mpl'], (_0x337f99['UBVQR'](_0x3934d6, this['mph']) + (_0x5e09a8[_0x8bb497] >> 0xf) * this['mpl'] & this['um']) << 0xf) & _0x5e09a8['DM'];
                             for (_0x5e09a8[_0x3934d6 = _0x337f99['NScfM'](_0x8bb497, this['m']['t'])] += this['m']['am'](0x0, _0x245588, _0x5e09a8, _0x8bb497, 0x0, this['m']['t']); _0x337f99['OKtob'](_0x5e09a8[_0x3934d6], _0x5e09a8['DV']); )
                                 _0x5e09a8[_0x3934d6] -= _0x5e09a8['DV'],
                                 _0x5e09a8[++_0x3934d6]++;
                         }
                         _0x5e09a8['clamp'](),
                         _0x5e09a8['drShiftTo'](this['m']['t'], _0x5e09a8),
                         _0x337f99['pAsGx'](_0x5e09a8['compareTo'](this['m']), 0x0) && _0x5e09a8['subTo'](this['m'], _0x5e09a8);
                     }
                     ,
                     _0x5033d5['prototype']['mulTo'] = function(_0x455b2f, _0x54e3f6, _0x366c2a) {
                         _0x455b2f['multiplyTo'](_0x54e3f6, _0x366c2a),
                         this['reduce'](_0x366c2a);
                     }
                     ,
                     _0x5033d5['prototype']['sqrTo'] = function(_0x2f5649, _0x4ca1d4) {
                         _0x2f5649['squareTo'](_0x4ca1d4),
                         this['reduce'](_0x4ca1d4);
                     }
                     ,
                     _0x59ca79['prototype']['copyTo'] = function(_0x246d98) {
                         for (var _0x670209 = _0x495033['FtodS'](this['t'], 0x1); _0x670209 >= 0x0; --_0x670209)
                             _0x246d98[_0x670209] = this[_0x670209];
                         _0x246d98['t'] = this['t'],
                         _0x246d98['s'] = this['s'];
                     }
                     ,
                     _0x59ca79['prototype']['fromInt'] = function(_0x592524) {
                         this['t'] = 0x1,
                         this['s'] = _0x495033['dNODL'](_0x592524, 0x0) ? -0x1 : 0x0,
                         _0x495033['eovOE'](_0x592524, 0x0) ? this[0x0] = _0x592524 : _0x592524 < -0x1 ? this[0x0] = _0x495033['jbiNJ'](_0x592524, this['DV']) : this['t'] = 0x0;
                     }
                     ,
                     _0x59ca79['prototype']['fromString'] = function(_0x4ec61a, _0x54fd36) {
                         var _0x4e4aa4;
                         if (_0x495033['JGYXo'](0x10, _0x54fd36))
                             _0x4e4aa4 = 0x4;
                         else if (_0x495033['JGYXo'](0x8, _0x54fd36))
                             _0x4e4aa4 = 0x3;
                         else if (0x100 == _0x54fd36)
                             _0x4e4aa4 = 0x8;
                         else if (0x2 == _0x54fd36)
                             _0x4e4aa4 = 0x1;
                         else if (0x20 == _0x54fd36)
                             _0x4e4aa4 = 0x5;
                         else {
                             if (0x4 != _0x54fd36)
                                 return void this['fromRadix'](_0x4ec61a, _0x54fd36);
                             _0x4e4aa4 = 0x2;
                         }
                         this['t'] = 0x0,
                         this['s'] = 0x0;
                         for (var _0x3feedf = _0x4ec61a['length'], _0x26405f = !0x1, _0x60ba00 = 0x0; --_0x3feedf >= 0x0; ) {
                             var _0x256069 = 0x8 == _0x4e4aa4 ? 0xff & _0x4ec61a[_0x3feedf] : _0x10aeb0(_0x4ec61a, _0x3feedf);
                             _0x256069 < 0x0 ? _0x495033['DKWdp']('-', _0x4ec61a['charAt'](_0x3feedf)) && (_0x26405f = !0x0) : (_0x26405f = !0x1,
                             0x0 == _0x60ba00 ? this[this['t']++] = _0x256069 : _0x495033['jbiNJ'](_0x60ba00, _0x4e4aa4) > this['DB'] ? (this[this['t'] - 0x1] |= (_0x256069 & (0x1 << this['DB'] - _0x60ba00) - 0x1) << _0x60ba00,
                             this[this['t']++] = _0x256069 >> this['DB'] - _0x60ba00) : this[this['t'] - 0x1] |= _0x495033['uLcaf'](_0x256069, _0x60ba00),
                             _0x495033['fFTKY'](_0x60ba00 += _0x4e4aa4, this['DB']) && (_0x60ba00 -= this['DB']));
                         }
                         0x8 == _0x4e4aa4 && 0x0 != _0x495033['pgDxk'](0x80, _0x4ec61a[0x0]) && (this['s'] = -0x1,
                         _0x60ba00 > 0x0 && (this[this['t'] - 0x1] |= _0x495033['uLcaf']((0x1 << _0x495033['jnDMy'](this['DB'], _0x60ba00)) - 0x1, _0x60ba00))),
                         this['clamp'](),
                         _0x26405f && _0x59ca79['ZERO']['subTo'](this, this);
                     }
                     ,
                     _0x59ca79['prototype']['clamp'] = function() {
                         for (var _0x524ab4 = this['s'] & this['DM']; this['t'] > 0x0 && this[_0x337f99['DYHQZ'](this['t'], 0x1)] == _0x524ab4; )
                             --this['t'];
                     }
                     ,
                     _0x59ca79['prototype']['dlShiftTo'] = function(_0x127b5a, _0x5a3f5d) {
                         var _0x3ae35a;
                         for (_0x3ae35a = _0x495033['jnDMy'](this['t'], 0x1); _0x3ae35a >= 0x0; --_0x3ae35a)
                             _0x5a3f5d[_0x3ae35a + _0x127b5a] = this[_0x3ae35a];
                         for (_0x3ae35a = _0x495033['FtodS'](_0x127b5a, 0x1); _0x3ae35a >= 0x0; --_0x3ae35a)
                             _0x5a3f5d[_0x3ae35a] = 0x0;
                         _0x5a3f5d['t'] = _0x495033['NAEpD'](this['t'], _0x127b5a),
                         _0x5a3f5d['s'] = this['s'];
                     }
                     ,
                     _0x59ca79['prototype']['drShiftTo'] = function(_0x491999, _0x11a0b1) {
                         for (var _0x34467b = _0x491999; _0x34467b < this['t']; ++_0x34467b)
                             _0x11a0b1[_0x34467b - _0x491999] = this[_0x34467b];
                         _0x11a0b1['t'] = Math['max'](_0x495033['FtodS'](this['t'], _0x491999), 0x0),
                         _0x11a0b1['s'] = this['s'];
                     }
                     ,
                     _0x59ca79['prototype']['lShiftTo'] = function(_0x4a74ad, _0x396269) {
                         var _0x5b8409, _0x3b0ced = _0x495033["\u004e\u0043\u0072\u0041\u0051"](_0x4a74ad, this['DB']), _0x10ba0f = this['DB'] - _0x3b0ced, _0x41ca43 = _0x495033['jnDMy'](0x1 << _0x10ba0f, 0x1), _0x3ce424 = Math['floor'](_0x495033['Rbmel'](_0x4a74ad, this['DB'])), _0x3d2ed7 = _0x495033['siLbZ'](this['s'] << _0x3b0ced, this['DM']);
                         for (_0x5b8409 = this['t'] - 0x1; _0x5b8409 >= 0x0; --_0x5b8409)
                             _0x396269[_0x5b8409 + _0x3ce424 + 0x1] = this[_0x5b8409] >> _0x10ba0f | _0x3d2ed7,
                             _0x3d2ed7 = (this[_0x5b8409] & _0x41ca43) << _0x3b0ced;
                         for (_0x5b8409 = _0x495033['Uzghn'](_0x3ce424, 0x1); _0x5b8409 >= 0x0; --_0x5b8409)
                             _0x396269[_0x5b8409] = 0x0;
                         _0x396269[_0x3ce424] = _0x3d2ed7,
                         _0x396269['t'] = _0x495033['NAEpD'](this['t'], _0x3ce424) + 0x1,
                         _0x396269['s'] = this['s'],
                         _0x396269['clamp']();
                     }
                     ,
                     _0x59ca79['prototype']['rShiftTo'] = function(_0x196e06, _0x618c6c) {
                         _0x618c6c['s'] = this['s'];
                         var _0x2625e7 = Math['floor'](_0x196e06 / this['DB']);
                         if (_0x2625e7 >= this['t'])
                             _0x618c6c['t'] = 0x0;
                         else {
                             var _0x18d979 = _0x337f99['deNGZ'](_0x196e06, this['DB'])
                               , _0x195461 = _0x337f99['DYHQZ'](this['DB'], _0x18d979)
                               , _0x4ef2d5 = _0x337f99['DYHQZ'](0x1 << _0x18d979, 0x1);
                             _0x618c6c[0x0] = this[_0x2625e7] >> _0x18d979;
                             for (var _0x17d5a2 = _0x2625e7 + 0x1; _0x17d5a2 < this['t']; ++_0x17d5a2)
                                 _0x618c6c[_0x337f99['DYHQZ'](_0x17d5a2, _0x2625e7) - 0x1] |= (this[_0x17d5a2] & _0x4ef2d5) << _0x195461,
                                 _0x618c6c[_0x17d5a2 - _0x2625e7] = _0x337f99['KbFOT'](this[_0x17d5a2], _0x18d979);
                             _0x18d979 > 0x0 && (_0x618c6c[_0x337f99['DYHQZ'](_0x337f99['DYHQZ'](this['t'], _0x2625e7), 0x1)] |= _0x337f99['FTCXD'](this['s'] & _0x4ef2d5, _0x195461)),
                             _0x618c6c['t'] = this['t'] - _0x2625e7,
                             _0x618c6c['clamp']();
                         }
                     }
                     ,
                     _0x59ca79['prototype']['subTo'] = function(_0x4dbc12, _0x20a2d1) {
                         for (var _0x189c46 = 0x0, _0x30233b = 0x0, _0x4252e4 = Math['min'](_0x4dbc12['t'], this['t']); _0x337f99['kiEVa'](_0x189c46, _0x4252e4); )
                             _0x30233b += _0x337f99['DYHQZ'](this[_0x189c46], _0x4dbc12[_0x189c46]),
                             _0x20a2d1[_0x189c46++] = _0x337f99['Vdwnb'](_0x30233b, this['DM']),
                             _0x30233b >>= this['DB'];
                         if (_0x4dbc12['t'] < this['t']) {
                             for (_0x30233b -= _0x4dbc12['s']; _0x189c46 < this['t']; )
                                 _0x30233b += this[_0x189c46],
                                 _0x20a2d1[_0x189c46++] = _0x30233b & this['DM'],
                                 _0x30233b >>= this['DB'];
                             _0x30233b += this['s'];
                         } else {
                             for (_0x30233b += this['s']; _0x189c46 < _0x4dbc12['t']; )
                                 _0x30233b -= _0x4dbc12[_0x189c46],
                                 _0x20a2d1[_0x189c46++] = _0x337f99['peDge'](_0x30233b, this['DM']),
                                 _0x30233b >>= this['DB'];
                             _0x30233b -= _0x4dbc12['s'];
                         }
                         _0x20a2d1['s'] = _0x30233b < 0x0 ? -0x1 : 0x0,
                         _0x30233b < -0x1 ? _0x20a2d1[_0x189c46++] = this['DV'] + _0x30233b : _0x337f99['RShhk'](_0x30233b, 0x0) && (_0x20a2d1[_0x189c46++] = _0x30233b),
                         _0x20a2d1['t'] = _0x189c46,
                         _0x20a2d1['clamp']();
                     }
```

映入眼帘的是一个简单的 ob 混淆，简单的 ob 混淆和看源码一样

> [!tip]
> 
> 这里有一个小技巧，打 return 语句的断点，看从哪里返回，再次出发，反向跟踪

![](https://bbs.kanxue.com/upload/attach/202409/967562_KQS82DG8QXSX2W6.png)

将这个函数里的 return 语句都打上断点

建议拖到可以折叠的 ide 里面，搞清函数作用域再下手

![](https://bbs.kanxue.com/upload/attach/202409/967562_FMMTGZ2CTCSDRRJ.png)

![](https://bbs.kanxue.com/upload/attach/202409/967562_M8TW8SDBU34DUJP.png)

发现在尾部的对象比较可疑，并且发现了一定的浏览器检测函数

![](https://bbs.kanxue.com/upload/attach/202409/967562_ATQHM3HGTSUBSA7.png)

和我们猜想的一样，数据最终在此处生成，我们首先先跟踪_0x10fab7 的生成方式

![](https://bbs.kanxue.com/upload/attach/202409/967562_VDEYGZWQJM2ZK55.png)

用搜索功能发现这个是随机生成的哈希值

遗憾的发现 abcde 中 abce 都是随机值

分析一下 data 数据的来源

![](https://bbs.kanxue.com/upload/attach/202409/967562_3JN9ZT4XDUDPN94.png)

发现数据来源一个 aes

```
'data': ''['concat'](_0x4879b7, '​')['concat'](''['concat'](_0x4a8f01)['concat'](_0x3bf831)['concat'](_0x1c4a22)['concat'](_0x2bd525)['concat'](_0xe412e2)['concat'](_0x345581)['concat'](_0x4dfef5)['concat'](_0x483389)['concat'](_0x1a0eab)['concat'](_0x5da258)['concat'](_0x3291c5)['concat'](_0x4595dc)['length'] > 0x5 ? ''['concat'](_0x4a8f01)['concat'](_0x3bf831)['concat'](_0x1c4a22)['concat'](_0x2bd525)['concat'](_0xe412e2)['concat'](_0x345581)['concat'](_0x4dfef5)['concat'](_0x5b3ed3)['concat'](_0x1a0eab)['concat'](_0x5da258)['concat'](_0x3291c5)['concat'](_0x4595dc) : '')['concat'](_0x3d369)
```

在这里发现一顿 concat ，只有_0x4879b7 _0x3d369 有值

在上面发现 aes 加密逻辑

![](https://bbs.kanxue.com/upload/attach/202409/967562_MRMU4BFTFKTPER8.png)

key 和 iv 都很明显

9AB3F04305F7C09AA29B48B729461DD2F5437976EDF4806D

key:

`_0x24bbfa['enc']['Utf8']['parse'](_0x10fab7['substr'](0xa, 0x10))`

'F7C09AA29B48B729'

iv:

`_0x24bbfa['enc']['Utf8']['parse'](_0x10fab7['substr'](0x14, 0x10)`

'48B729461DD2F543'

加密结果（为第二个参数的值）

'7369676e3d6665323662656663343934343464333632633866313734363336333062646261'

```
var _0x3d369 = _0x3b1c65 ? _0x3b1c65['ciphertext'] ? _0x3b1c65['ciphertext']['toString']()['toUpperCase']() : _0x3b1c65 : '', _0x43a85d = _0x10fab7['split']("")
, _0xc7af72 = Number(_0x35594c['qNJuC'](new Date()['valueOf'](), 0x3e8)['toFixed'](0x0))['toString'](0x10)['toUpperCase']()['split']('');
```

接下来发现对密文进行处理

`var _0x3d369 = _0x3b1c65 ? _0x3b1c65['ciphertext'] ? _0x3b1c65['ciphertext']['toString']()['toUpperCase']() : _0x3b1c65 : ''`

碰巧拿到了上面拼接的第二个参数

接下来继续研究第一参数的生成

![](https://bbs.kanxue.com/upload/attach/202409/967562_MNYYCCT5G37VQP8.png)

发现其参数生成和日期有关系

![](https://bbs.kanxue.com/upload/attach/202409/967562_6VFFX7PTC7D59TK.png)

这个是在生成时间戳

```
var _0x16af4, _0x2551e2, _0x30a9c6 = _0x109447['getKey'](_0x35594c['atlzK'])['encrypt'](_0x43a85d['join']('')), _0x365b44 = "".split("").reverse().join("");
```

在这里发现了对 aes 的密钥的进一步加密 从 getkey 可以大致推断出他是 rsa 加密

'MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBANMGZPlLobHYWoZyMvHD0a6emIjEmtf5Z6Q++VIBRulxsUfYvcczjB0fMVvAnd1douKmOX4G690q9NZ6Q7z/TV8CAwEAAQ=='从返回值中拿到密钥

![](https://bbs.kanxue.com/upload/attach/202409/967562_FRSHAC64UNZPH9E.png)

看到了 npq 以及 e 即可确定这是一个标准的 rsa 加密，我们接下来只需要得到加密前的明文即可完成分析

![](https://bbs.kanxue.com/upload/attach/202409/967562_34GASRPPK26PG6R.png)

在 encrypt 后打断点观察明文

![](https://bbs.kanxue.com/upload/attach/202409/967562_D76U5KC5CWGV7Y6.png)

发现明文就是 92B3F04305F7C09AA29B48B729461DD2F5437976662B4ED4

49714c457c58d6734ad87bda0ec38b1bad41d89ceecea1eb4d9cdcfaccf654ec48e0ec2c0cee3722a48aa1b56b8a14a5fbb003e80d5b920e33dad3072408af3d

加密的结果长度和第一个差不多，我推测就是一个编码的转换问题

"SXFMRXxY1nNK2HvaDsOLG61B2JzuzqHrTZzc+sz2VOxI4OwsDO43IqSKobVrihSl+7AD6A1bkg4z2tMHJAivPQ"

![](https://bbs.kanxue.com/upload/attach/202409/967562_FSMU37BJRYNJVKT.png)

发现我们推断的结果正确，到此加密分析全部结束

总结
--

![](https://bbs.kanxue.com/upload/attach/202409/967562_K5RFS7HVPCGVB2Z.png)

在分析过程中，发现大量明文对浏览器环境的监测，防止大家放到 node 里面一把梭，总的来说是做算法更容易一些。

> [!tip]
> 
> 对 d * 的防护建议：
> 
> 对关键字符串进行加密，此文件中多次出现 encrypt 关键字，并且 AES 加密也没有进行隐藏，很容易搜索定位到。
> 
> 建议取消无限 debugger 的防护，这样给人一种此地无银三百两的感觉，直接定位到了关键加密的函数

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 2024-10-1 21:39 被 kanxue 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)