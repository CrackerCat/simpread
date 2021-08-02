> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/M11b5zmjuc7llpdniwFgMA)

![](https://mmbiz.qpic.cn/mmbiz_jpg/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgS6nsjT7R0xK7Ribrx6HCtZn4ILRx62XzTxAbpmRkaW8W2ZaQMTNQ5HHQ/640?wx_fmt=jpeg)

声明：本文内容仅供学习交流，严禁用于商业用途，否则由此产生的一切后果均与作者无关，请于 24 小时内删除。

**1，前言**

    本文不是一个细节贴，因为，加密文件解混淆后逻辑就很清晰了，就是一步步走，然后补环境，低级工作。

    所以本文就 3 个任务：

    1.1，分享一个基于 babel 的 nodejs 解混淆脚本，经多天测试，可以适应多个版本的混淆文件。

    1.2，分享补出来的环境，需要注意的是不同的 js 文件似乎检测的环境有些区别，多个 js 文件加起来的环境才是一个持续可用的结果？所以，我的环境并不完全，成功了一天后就失效了，你还是需要跟逻辑继续补的。

    1.3，宏观层面进行逻辑分析，细节老哥们自己慢慢看。

**2，脚本分享**

    有几步是 ob 混淆的常规解法，还有几步是针对 boss 直聘的混淆单独写的，我之前写过具体的说明，这里就不详细说了。

```
参考链接：
https://bbs.nightteam.cn/thread-417.htm
https://bbs.nightteam.cn/thread-423.htm

```

**3，补环境**

    3.1，补环境方法介绍：

        3.1.1，初步提取环境：使用  Proxy 和 hook 写一个脚本，能初步提取出一些环境，提取出多少和写的脚本有关。

      3.1.2，debug 完善环境：hook 住要逆向的 js 文件中使用的生成随机数的方法（Date.gettime 和 Math.random 之类的几个方法，debug 过程中遇到了再 hook 即可，不用最开始全部 hook 全。），然后一步步 debug，对比自己代码和浏览器中代码 运行时的变量，找到同一个变量不同值的原因。  

        例如下图：

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgS2zkrlhbJfcQoH9DRWmwl15WGuMcibFEKicEllZN3XBHYVKpjoBiaiaASmQ/640?wx_fmt=png)

        3.1.3，小提示：寄希望于一键脱所有环境的脚本，有可能会浪费自己更多的时间，所以脚本慢慢完善即可，该 debug 的时候不要怂，哈哈。

    3.2，根据一个 js 文件补出的环境，不完善，需要多个 js 文件补出环境后汇总一下。

```
Function.prototype.toString = function () {
    console.log("Function.prototype.toString", this.name)
    return "";
}
let location = {
    "hostname": "www.zhipin.com",
    "href": "https://www.zhipin.com/web/common/security-check.html?seed=fngF9gcPr%2Buu99%2F8Jl37qOXVRgblPTzo6TZhNsG5MUQ%3D&
}
let document = {
    "location": location ,
    "cookie": "",
    "createElement": function (a, b, c, d, e, f, g) {
        if (a == 'canvas') {
            return {
                "getContext": function (a, b, c, d, e, f, g) {
                    if (a == "2d") {
                        return {
                            "fillRect": function (a, b, c, d, e, f, g) {
                            }, "fillText": function (a, b, c, d, e, f, g) {
                            }
                        }
                    }
                },
                "toDataURL": function () {
                    return "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"
                }
            }
        } else {
            debugger;
        }
    },
    "getElementById": function (a, b, c, d, e, f, g) {
    }
}
let navigator = {
    "cookieEnabled": true,
    "language": "zh-CN",
    "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.212 Safari/537.36",
    "webdriver": false,
    "appVersion": "5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.212 Safari/537.36"
}
let localStorageObj = {}
let localStorage = {
    "setItem": function (a, b, c, d, e, f, g) {
        localStorageObj[a] = b;
    },
    "getItem": function (a, b, c, d, e, f, g) {
        return localStorageObj[a];
    },
}
let sessionStorage = {}
let window = {
    "closed": false,
    "sessionStorage": sessionStorage,
    "localStorage": localStorage,
    "setInterval": function (a, b, c, d, e, f, g) {
        debugger;
    },
    "document": document,
    "navigator": navigator,
    "length": 0,
    "decodeURI": decodeURI,
    "history": {
        "length": "2",
        "scrollRestoration": "auto",
        "state": null
    },
    "location": location,
    "eval": function (a, b, c, d, e, f, g) {
        console.log("eval", a)
        return eval(a);
    },
    "outerHeight": 28,
    "innerHeight": 0,
    "outerWidth": 160,
    "innerWidth": 0,
    "Math": Math,
    "Date": Date,
    "OfflineAudioContext": function (a, b, c, d, e, f, g) {
        return {
            "createOscillator": function (a, b, c, d, e, f, g) {
                return {
                    "frequency": {
                        "setValueAtTime": function (a, b, c, d, e, f, g) {
                        }
                    }
                };
            },
            "createDynamicsCompressor": function (a, b, c, d, e, f, g) {
            }
        }
    },
    "Function": Function,
    "top": {
        "location": location
    },
    "atob": function (a, b, c, d, e, f, g) {
        return Buffer.from(a, 'base64').toString("binary");
    },
    "toString": function () {
        return "[object Window]";
    }
}
window.window = window;
Object.keys(window).forEach(property => {
    try {
        if (typeof global[property] === 'undefined') {
            global[property] = window[property];
        }
    } catch (e) {
        // console.log(e);
    }
});
global = undefined;
process.argv = undefined;
/**
* 这里放入原js代码 或者 解混淆之后的代码
*/
function get_zp_token_(seed, ts) {
    var _zp_token_ = new ABC().z(seed, ts)
    return _zp_token_;
}
// let _zp_token_ = get_zp_token_("b7lQU7A+0iRGlzbRhS/hqxPs0GaqGXRz46PNaKfAlM4=", 1621756701529);
// console.log(_zp_token_);

```

**4，宏观逻辑分析**

    4.1，目标网址：

```
aHR0cHM6Ly93d3cuemhpcGluLmNvbS9jMTAxMDEwMTAwLXAxNDAxMDQv

```

    4.2，加密定位

        4.2.1，由于直接访问目标网址时未携带需要的 cookie(__zp_stoken__)，所以返回 302 页面，response 的 headers 中有 name 为 location 的 header 了要跳转的 url。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgSO7HeSy6MYg1JZ4icHf3t2Ft9Hy82icCGWngJZvZRMDmXbWFLLU24ovJQ/640?wx_fmt=png)  

        4.2.2，location 的 url 中包含了生成 cookie 的逻辑，生成 cookie 的入口处 定位方式就是 hook 一下 cookie 的 set 过程，这个我就不细说了，比较基础，不太明白的话可以群里问。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgSp6FdGw3d3bL29W7D4Eg4LejePnic2iaUnJIsiattFQMlIYj9So1LBRhow/640?wx_fmt=png)

        4.2.3，/web/common/security-js/****.js 就是那个每天都会变的 js，每天的名称不一样，里面的逻辑也是不一样的，里面包含了所需 cookie 的生成逻辑。

    4.3，复杂处分析

        整个 js 有点复杂的就一个地方：

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgSK0Q52GpRic7Nic6rDGXSVuzupNr6XDCTRibcFBOjeu6qIO82WpeyiapD4A/640?wx_fmt=png)

        4.3.1，数字 2 处初始化后的部分变量的值代表了一个错误的环境参数。

        4.3.2，搜索 Function 或者 eval 可以看到类似图中方框的内容，每一个这种判断都是在修改数字 2 处中错误的环境参数。

        4.3.3，最后将各个值汇总成一个数组，生成一个指纹。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgSx2C39HnfvwtD256dCj5GKK9XgdkEM1HJTPWjicJ4AaKWekxdqzRlf1A/640?wx_fmt=png)

    这个问题，只要你 hook 住 Math.random（参照 3.1.2），就很清楚了。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgSpsaIcLG1ib2ZoicoKoE3Dlx06MibOWXPFOyEVr9lwVbd6T8nwkIwT79gQ/640?wx_fmt=png)

    这些不一样的都是错误的，最后看看这个列表里哪一个没有被修改再回找就行了，不用看具体的那一大坨逻辑。

    否则，你会看到

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMW2G8FqSiciaNjMYH8bebzBgSuBX5ib1XfgfglAVF0SA7RPFjomdFk0VX8o3LhGiaJrqEH8aicIPMRe3GA/640?wx_fmt=png)

    这样的初始化。

**5，结束**

    反混淆文件地址及本文相关 js 文件：  

```
https://github.com/chencchen/webcrawler/tree/master/6,某直聘

```

    大概就这样吧，有啥不太理解的可以群里讨论。

求个点赞！再加个再看就更好了。

哈哈。拜拜~  

![](https://mmbiz.qpic.cn/mmbiz_jpg/huXBGBmwjMWq6U8cJFU8fZvCKoG7h4xNuJqlp0ia8hUK1RkriaQ8OE8jncEe7yhenEhyHV3nvZLRUk2sOgCfFIjQ/640?wx_fmt=jpeg)