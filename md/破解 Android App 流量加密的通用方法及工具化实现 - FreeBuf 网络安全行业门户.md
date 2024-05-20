> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.freebuf.com](https://www.freebuf.com/articles/mobile/400766.html)

> 本篇文章主要讨论破解 APP 流量加密的通用方法，从被动触发、主动构造、透明加密代理服务器。

1 概述
----

本篇文章主要讨论破解 APP 流量加密的通用方法，从被动触发、主动构造、透明加密代理服务器 三种角度实现工具化，以及使用 Cookiecutter 模版减少重复性工作。

Repo 地址：[https://github.com/PadishahIII/cookiecutter-frida](https://github.com/PadishahIII/cookiecutter-frida "https://github.com/PadishahIII/cookiecutter-frida")，**在使用此 repo 前，请先阅读本文**。

### 1.1 应用场景

在漏洞挖掘或渗透测试过程中，经常遇到 APP 流量加密的情况，以银行类 APP 为例，所有 Http 请求及响应都会被加密，加密算法会生成随机的对称密钥，用非对称算法加密该随机密钥，并对加密内容计算 MAC，防止篡改。我们需要分析 APP 的加密流程，让测试人员能够直接操作请求和响应的明文。根据工具化的程度不同，可以将需求分成以下几类：

1.  绕过流量加密和完整性校验，拦截和修改请求和响应的明文
    
2.  主动构造加密请求包，并解密得到响应明文
    
3.  实现透明加密网关，该网关自动加密请求、解密响应，完全无视 APP 的流量加密
    

本文会依次阐述如何从这三个角度实现工具化，以及如何快速定位加解密逻辑，并给出一个 Cookiecutter 模版，所有工具化的代码都可复用，测试人员只需编写 hook 加解密函数的 Frida 脚本。

### 1.2 说明

逆向方面，本文只关注如何快速破解流量加密，对于环境检测绕过、反代理等方面不做详细讨论，但会给出通过绕过方法。下面的内容默认你已经将 APP 脱壳、反编译出源码（尤其是发送网络请求的逻辑），并且可以正常用 frida 注入 js.

2 环境检测绕过
--------

这里不做详细讨论，我把一些常用的环境检测方法写到了一个脚本里，该脚本包含在 cookiecutter 模版里面，主要功能有：root 检测绕过、frida 检测绕过、SSL Unpinning、动态库加载 hook，还有一些工具函数，可以从[此处](https://github.com/PadishahIII/cookiecutter-frida/blob/master/%7B%7B%20cookiecutter.directory_name%20%7D%7D/hook_script/fridaBypass.js "此处")获取。

下面讲一种我最近遇到的反代理。在分析某银行 APP 时，我发现它会在本地 8899 端口开启一个 HTTP 代理服务器，然后把所有 http 流量代理到 8899 端口，这个代理服务器会使用国密 SSL 与服务端通信：

```
ProxyUtil.startProxy(num.intValue());//8899
GMSSLConfig.setSSLProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("127.0.0.1", num.intValue())));


```

除此之外，这个代理还会修改请求包，比如修改 Content-Type 请求头。对于这种情况，如果服务端兼容标准 SSL 与国密 SSL，那只需要把`java.net.Proxy`类 hook 掉，把代理的 IP 和 Port 改成 Burp 就可以抓包：

```
function start() {
    Java.perform(function () {
        let proxyClz = Java.use("java.net.Proxy")
        let addressClz = Java.use("java.net.InetSocketAddress")
        proxyClz.address.implementation = function () {
            var res = this.address()
            let myproxy = addressClz.$new("192.168.43.246", 8080) // change here

            console.log(`Proxy: ${res} -> ${myproxy}`)
            return myproxy
        }
    })
}
start()


```

如果服务端只支持国密 SSL，Burp 还要设置一层上游国密代理（可以用 Yakit），但真实情况没有这么简单，服务端可能存在证书绑定，或者手机端的这个代理服务器会对报文做修改，导致 Burp 转发出去的包是不合法的，这种情况就要分析这个代理服务器的动态库。一个更简单的方法，就是把 Burp 的流量转发回手机端的代理 8899 端口上，由于这个代理监听在 127.0.0.1 地址，局域网没法直接访问，需要用 SuperProxy 在手机端开启 HTTPS 代理，并设置上游地址为本地的 8899，这样就可以把 Burp 抓到的流量成功发到服务端，代理链为: App→Burp→SuperProxy→8899 端口→服务端。

3 分析加解密逻辑
---------

本节以某银行 APP 为例，定位并分析其加密逻辑，我们需要定位到加解密和构造请求的函数。

### 3.1 快速定位加解密逻辑

**3.1.1 通过 OkHttpClient 获取调用栈**

Frida-trace 跟踪`OkHttpClient`:

```
frida-trace -F -H 127.0.0.1:3333 -j "*OkHttpClient\!newCall/i"


```

修改 frida-trace 生成的 js 脚本，打印调用栈：

```
onEnter(log, args, state) {
    log(`OkHttpClient.newCall(${args.map(JSON.stringify).join(', ')})`);
    var Exception = Java.use('java.lang.Exception');
    var Log = Java.use('android.util.Log');
    var stackinfo = Log.getStackTraceString(Exception.$new());
    console.log(stackinfo)
  }


```

![](https://image.3001.net/images/20240513/1715573195_664191cbf4140ad785c0c.png!small)

由此可定位到`APPRestClient`类的`post`方法：

![](https://image.3001.net/images/20240513/1715573209_664191d93a553277d8434.png!small)

继续分析`APPRestClient`类，发现它还有一个`get`方法，不过跟踪发现这个方法并没有调用。后面我们就用`post`方法构造请求。

**3.1.2 通过函数名定位**

加解密函数一般会包含 encrypt 或者 decrypt 关键字，用 frida-trace 过滤函数名：

```
frida-trace -F -H 127.0.0.1:3333 -j "*\!*encrypt*/i"


```

定位到`V2DataProcessor.encrypt`，可以看到加密前的明文

![](https://image.3001.net/images/20240513/1715573241_664191f9babd17e05cacc.png!small)

反编译查看`V2DataProcessor`代码，定位到加解密逻辑：

![](https://image.3001.net/images/20240513/1715573248_66419200bf68b9a1995e8.png!small)

继续跟踪这个类：

```
frida-trace -F -H 127.0.0.1:3333 -j "*DataProcessor*\!*/i"


```

![](https://image.3001.net/images/20240513/1715573257_66419209002bfdcdf1730.png!small)

由此可以确定`V2DataProcessor`类的`encrypt`和`decrypt`函数就是用来加密请求、解密响应的。

继续跟`CryptoUtil`类的`encryptDataWithSMByToken`方法：

![](https://image.3001.net/images/20240513/1715573265_664192114d84fc74bb082.png!small)

跟进`reqEncode`，发现是 native 方法：

![](https://image.3001.net/images/20240513/1715573272_664192184194736614a09.png!small)

我们不用知道 APP 具体是如何加密的，只要定位到加解密函数就够了。

**3.1.3 通过日志类定位**

App 可能会将发送请求、参数加解密的信息打印到日志里，我们需要先找到日志类，随便找一个 Activity 类，很容易就能找到打印日志的代码，上面我们已经在`APPRestClient`类中看到了，日志类为：

```
com.yitong.mobile.component.logging.Logs


```

跟踪日志类所有方法：

```
frida-trace -F -H 127.0.0.1:3333 -j "com.yitong.mobile.component.logging.Logs\!*/i"


```

可以看到请求和响应的处理过程：

![](https://image.3001.net/images/20240513/1715573283_6641922354bf51a690ecc.png!small)

经过上面的分析，我们已经定位到发送请求的方法为`APPRestClient.post`，加解密由`V2DataProcessor`实现。下面我们将利用这几个 hook 点来绕过流量加密。

4 FridaRPC 转发 Burp 实现拦截和修改明文
----------------------------

本节介绍如何通过 frida-rpc 与 Burp 联动实现拦截和修改请求响应的明文，参考自[这个 repo](https://github.com/noobpk/frida-intercept-encrypted-api "这个repo"). 架构图如下：

![](https://image.3001.net/images/20240513/1715573295_6641922f2eeb88a6d2496.png!small)

简单来讲，就是用 frida-rpc 在加密前把请求明文发送到 python 脚本，然后转发到 Burp，去访问一个 echoServer，这个 echoServer 将 http 请求原封不动作为响应返回，Burp 可以拦截或修改请求，最终传递回 frida 脚本里，后续加密的是修改之后的请求。**注意这里的注入点要在计算签名之前**。

首先，hook`APPRestClient`的`post`方法，

```
let APPRestClient = Java.use("com.yitong.mobile.network.http.APPRestClient");
APPRestClient["post"].overload('java.lang.String', 'java.lang.String', 'java.lang.String', 'com.yitong.http.ResponseHandlerInterface', 'java.lang.String').implementation = function (url, body, str3, responseHandlerInterface, str4) {
    console.log(`\nrequest: str=${url}, str2=${body}`);
    var data = sendToBurpReq(body, url)
    console.log("body>>>>>>>")
    console.log(data)
    // body = strClz.$new(data.toString())
    // console.log(`body: ${body}`)
    this["post"](url, body, str3, responseHandlerInterface, str4);
};


```

其中`sendToBurpReq`函数通过 frida-rpc 将请求明文发送到 python 脚本，并阻塞地等待应答，实现如下：

```
function sendToBurpReq(res, operation_type = "") {
    var data = res.toString()
    send({ from: "/http", payload: data, api_path: "request", method: operation_type });
    var op = recv("input", function (value) {
        data = value.payload
    });
    op.wait();
    return data
}


```

在`burpTracer.py`中接收 rpc 数据：

```
def frida_process_message(message, data):
        if message["type"] == "input":
            handled = True
        elif message["type"] == "send":
            body = message["payload"]
            req = requests.request(
                "FRIDA",
                "http://%s:%d/%s" % (BURP_HOST, BURP_PORT, API_PATH),
                headers={
                    "content-type": "application/json",
                },
                data=body["payload"].encode("utf-8"),
            )
            script.post(
                {"type": "input", "payload": req.text}
            )  # 把修改后的数据传输回给js

script.on("message", frida_process_message)


```

然后启动 echoServer，监听在 27080 端口：

```
ECHO_PORT = 27080

class RequestHandler(BaseHTTPRequestHandler):

    def do_FRIDA(self):
        request_path = self.path

        request_headers = self.headers
        content_length = request_headers.get('content-length')
        # length = int(content_length[0]) if content_length else 0
        length = int(content_length) if content_length else 0

        self.send_response(200)
        self.end_headers()

        self.wfile.write(self.rfile.read(length))

def main():
    try:
        logger.info('[*] Listening on 127.0.0.1:%d' % ECHO_PORT)
        server = HTTPServer(('', ECHO_PORT), RequestHandler)
        server.serve_forever()

    except KeyboardInterrupt:
        logger.info("Stop echoServer!!")

if __name__ == "__main__":
    logger.info('[*] Starting echoServer on port %d' % ECHO_PORT)
    main()


```

Burp 设置 26080 端口的监听，转发至 27080 端口：

![](https://image.3001.net/images/20240513/1715573317_66419245f04dd76a27041.png!small)

代理链为：frida 脚本→`burpTracer.py`→burp(26080)→echoServer(27080)

在此基础上，可以添加一个 mitmproxy 代理，用来自动化修改请求内容，监听在 27081 端口上，上游代理为 27080 端口，修改 burp 的转发端口为 27081

```
mitmproxy -s mitm.py --listen-host 0.0.0.0 -p 27081 --mode upstream:http://127.0.0.1:27080 -k


```

![](https://image.3001.net/images/20240513/1715573326_6641924edd736c8c9d8c8.png!small)

代理链变为：frida 脚本→`burpTracer.py`→burp(26080)→mitmproxy(27081)→echoServer(27080)

启动流程：

1.  开启 echoServer,`python echoServer.py`
    
2.  开启 mitmproxy，`mitmproxy -s mitm.py --listen-host 0.0.0.0 -p 27081 --mode upstream:http://127.0.0.1:27080 -k`
    
3.  Burp 设置 26080 端口监听，转发至 27081
    
4.  启动`burpTracer.py`，attach 到进程上，`python burpTracer.py -s hook.js -n <AppName>`
    
5.  通过 Burp 拦截和修改请求响应
    

至此，一个基本的修改明文的功能就实现了，上面只演示了修改请求，想要修改响应只需加一个 hook 点，这里我选择在解密函数上做 hook:

```
let V2DataProcessor = Java.use("com.yitong.mobile.network.process.V2DataProcessor")
V2DataProcessor.decrypt.implementation = function (s) {
    console.log(`\nresponse:`)
    var dec = this.decrypt(s)
    console.log(dec)
    try {
        var res = sendToBurpRes(dec)
        console.log(">>>>>>")
        console.log(res.toString())
        dec = strClz.$new(res.toString())
    } catch (e) {
        console.error(e)
    }
    return dec
}


```

Burp 抓到的包是这样的：

![](https://image.3001.net/images/20240513/1715573344_66419260e357a7201f037.png!small)

这不是一次真实的 http 请求，因为 python 脚本收到的只有你 hook 出来的请求体，所以它需要把数据包装成一个 http 请求，注意到上图中的 http 方法是`FRIDA`，这个请求会发送到 echoServer，它只是把请求体作为响应体返回，所以上图中的响应和请求体相同，如果你在 burp 或 mitmproxy 中修改了请求，那响应将会是修改之后的结果。

这种方法可以快速创造出渗透条件，你不用把 APP 的加密流程完全破解，只需定位到加解密的关键点，但缺点也很明显，不能主动构造请求，你想测试一个接口，只能一次次在手机上点击相应的按钮，来触发 hook 点。

5 主动构造请求包
---------

假如我们想 Fuzz 接口参数或爆破一个接口，上面的方法就做不到了，我们需要知道加密包是如何构造的。利用 frida-rpc 的强大功能，我们可以主动调用 APP 的加密函数。在上面的章节中，我们定位到`CryptoUtil`类的`encryptDataWithSMByToken`方法，并通过 hook 确定这个函数就是用来加密请求体的：

![](https://image.3001.net/images/20240513/1715573355_6641926b733b22b0e55e4.png!small)

![](https://image.3001.net/images/20240513/1715573389_6641928dec2a647151e11.png!small)

Hook 脚本如下：

```
CryptoUtil["encryptDataWithSMByToken"].implementation = function (application, str, str2, str3) {
    console.log(`CryptoUtil.encryptDataWithSMByToken is called: application=${application}, str=${str}, str2=${str2}, str3=${str3}`);
    let result = this["encryptDataWithSMByToken"](application, str, str2, str3);
    console.log(`CryptoUtil.encryptDataWithSMByToken result=${result}`);
    return result;
};


```

![](https://image.3001.net/images/20240513/1715573400_6641929865b5d6c7ed4fa.png!small)

![](https://image.3001.net/images/20240513/1715573405_6641929db4caf6f346b55.png!small)

`str2`和`str3`是复用的，第一个参数`application`可以通过调用`YTBaseApplication.getInstance()`获取，`str`参数是请求体明文，如果我们想主动调用`encryptDataWithSMByToken`，就需要知道当前的`str2`和`str3`，方法很简单，先触发一次 hook 点，把这两个参数存储下来，后面主动调用时直接传入就可以了：

```
var key = null
var uuid = null
function hook() {
    Java.perform(function () {
        let CryptoUtil = Java.use("com.yitong.mbank.util.security.CryptoUtil");
        CryptoUtil["encryptDataWithSMByToken"].implementation = function (application, str, str2, str3) {
            // console.log(`CryptoUtil.encryptDataWithSMByToken is called: application=${application}, str=${str}, str2=${str2}, str3=${str3}`);
            let result = this["encryptDataWithSMByToken"](application, str, str2, str3);
            key = str2
            uuid = str3
            // console.log(`CryptoUtil.encryptDataWithSMByToken result=${result}`);
            return result;
        };
    })
}


```

主动调用：

```
function encrypt(s) {
    var data = "NONE"
    Java.performNow(function () {
        let CryptoUtil = Java.use("com.yitong.mbank.util.security.CryptoUtil");
        let YTBaseApplication = Java.use("com.yitong.mobile.framework.app.application.YTBaseApplication");
        data = CryptoUtil.encryptDataWithSMByToken(YTBaseApplication.getInstance(), s, key, uuid)
    })
    return data
}


```

![](https://image.3001.net/images/20240513/1715573417_664192a90b59a3a32c6cd.png!small)

将`encrypt`函数导出到 frida-rpc，就可以在 python 脚本里调用了

```
rpc.exports.encrypt = encrypt


```

Python 脚本中的调用方法：

```
device = frida.get_remote_device()
device = frida.get_device_manager().add_remote_device("192.168.43.230:3333")
process = device.attach(process_name)

with open(script_name, encoding="utf-8", errors="ignore") as f:
    script = process.create_script(f.read())
script.on("message", frida_process_message)
script.load()
logger.info("Load Script")

logger.info(script.list_exports_sync())

encrypt = script.exports_sync.encrypt
encrypt("sss")


```

这样就可以在 python 脚本里构造请求包、批量发送请求了。很多时候，加密请求并不是一个`encrypt`函数就能构造出来的，可能还有其他请求参数需要添加，请求头也需要自行构造。另一种方法，我们可以直接调用 APP 发送请求的函数，由 APP 构造请求，我们就不用去分析请求包里每个参数的含义了，例如，APP 使用下面这个函数发送请求：

```
public List doPost(String s, Object object0, String s1, Class class0) {


```

hook 这个函数，打印参数和返回值，发现传入的请求体和返回的响应结果都是明文

![](https://image.3001.net/images/20240513/1715573426_664192b2e980d0eaa8572.png!small)

所以用这个函数就可以非常方便的构造请求并获取响应，这里不放代码了。

总结一下，我们有两种主动构造加密请求包的方法：

一种是调用加密函数，在 python 脚本中构造并发送请求，获取到加密请求体后，需要自行构造请求头和其他请求参数，得到响应后需要自己解密（调用解密函数也是一样的方法），如果请求没那么复杂，这种方法就够用了；

另一种是在 APP 侧构造并发送请求，这种方法的优点是不用考虑其他请求参数，只用把请求明文传入，由 APP 发送请求，运气好的话，找到一个像上面的`doPost`之类的函数，甚至响应都不用自己解密。

6 透明加密网关
--------

在上一节中，我们用 frida-rpc 调用加密函数的方式实现了主动构造请求包，并且能够并发发送请求，把这个思路再延伸一下，既然流量加密的问题解决了，我们可以搭一个服务器，它会加密明文请求、解密响应，将加密请求发送给指定的 URL。下面用 flask 实现，将 URL 放在请求头`Operation-Type`中，请求体为明文，这里把加解密过程放在`rpc_request`中，可以根据具体情况自行实现，它的返回值为响应明文。

```
app = Flask(__name__)
rpc_request = script.exports_sync.request

@app.route("/request", methods=["FRIDA"])
def send_request():
    if request.method == "FRIDA":
        data = request.get_data()
        path = request.headers.get("Operation-Type")
        print(f"{path}: {data}")
        res = rpc_request(path, data.decode("utf8")) # call rpc method
        res = str(json.loads(res))
        print(res)
        response = flask.Response(res, 200, content_type="application/json")
        response.data = res
        return response
    else:
        flask.abort(405)
app.run(port=8989))


```

向 8989 端口发送请求：

![](https://image.3001.net/images/20240513/1715573436_664192bc858fbb0c3243d.png!small)

有了透明加密网关，你可以完全无视 APP 的流量加密，你可以 Fuzz、爆破接口，可以用渗透中用到的任何工具。

到这里，工具化的三个角度已经讲完了，总结一下，

<table><thead><tr><th>需求</th><th>方法</th><th>特点</th></tr></thead><tbody><tr><td>绕过流量加密和完整性校验，拦截和修改请求和响应的明文</td><td>Hook 加解密函数，用 FridaRPC 转发 Burp</td><td>被动触发，不能构造请求</td></tr><tr><td>主动构造加密请求包，并解密得到响应明文</td><td>FridaRPC 主动调用加解密函数或请求函数</td><td>主动调用</td></tr><tr><td>实现透明加密网关，该代理自动加密请求、解密响应，完全无视 APP 的流量加密</td><td>搭建一个 Http 服务器，自动加密请求、解密响应</td><td>主动调用</td></tr></tbody></table>

那么在实际场景中如何运用呢？其实第三个方法 - 透明加密网关，是第二个方法的延伸版本，在实际中，可以用第一个方法抓包，然后搭一个透明加密网关用来兼容渗透工具。把破解 APP 流量加密到工具化的整个过程总结一下：

1.  定位到关键加解密函数以及发送请求的函数
    
2.  编写 hook 脚本，FridaRPC 联动 Burp，拦截和修改请求响应
    
3.  编写后端接口，实现透明加密
    

最终的效果：

用 Burp 查看明文请求和响应，以及原始请求包

![](https://image.3001.net/images/20240513/1715573449_664192c92cce20f91eb8f.png!small)

![](https://image.3001.net/images/20240513/1715573453_664192cde20b2b0e54f0c.png!small)

![](https://image.3001.net/images/20240513/1715573459_664192d3955d5bf2a6623.png!small)

能够爆破接口：

![](https://image.3001.net/images/20240513/1715573465_664192d98937a3943aec1.png!small)

7 使用 Cookiecutter 模版
--------------------

我把工具化相关的部分放在了一个 cookiecutter 模板里，你只需编写 hook 代码，再修改一下后端接口调用 rpc 函数的逻辑，就能实现前面描述的所有功能。

1.  首先需要安装 [cookiecutter](https://cookiecutter.readthedocs.io/en/1.7.3/README.html)
    

```
pip insatll --user cookiecutter


```

1.  拉取仓库，初始化项目：
    

```
> cookiecutter https://github.com/PadishahIII/cookiecutter-frida.git
[1/9] directory_name (sample_project): test
[2/9] package_name (com.certain.package): 
[3/9] app_name (AppName): 
[4/9] local_ip (192.168.43.246): 
[5/9] mitm_http_port (8082): 
[6/9] mitm_frida_port (27081): 
[7/9] frida_ip (127.0.0.1): 
[8/9] frida_port (3333): 
[9/9] rpc_server_port (8989):


```

这些参数的含义如下：

*   `directory_name`: 新建目录的名称
    
*   `package_name`（必需）: 目标 APP 的包名
    
*   `app_name`（必需）：目标 APP 的应用名
    
*   `local_ip`（必需）：电脑的局域网地址，需要和测试机在同一局域网
    
*   `mitm_http_port`：代理 http 流量的 mitmproxy 监听端口
    
*   `mitm_frida_port`：代理 frida 流量的 mitmproxy 监听端口
    
*   `frida_ip`：测试机 IP
    
*   `frida_port`：frida 监听端口
    
*   `rpc_server_port`：透明加密网关的监听端口
    

除了必需的三个参数，其他保持默认就可以。

1.  编写 hook 脚本及 RPC 函数
    

目录结构如下：

```
{{ cookiecutter.directory_name }}
├── Readme.md
├── burpTracer.py # 将hook到的数据转发Burp
├── echoServer.py # 镜像服务器
├── echoServer2.py # 辅助工具，另一种镜像服务器，打印收到的所有http请求
├── hook_script # 该目录存放所有frida脚本，及frida命令
│   ├── attach.sh # frida attach到目标App
│   ├── encrypt_rpc.js # 实现加解密函数，提供RPC接口
│   ├── fridaBypass.js # 绕过环境检测
│   ├── hook.js # hook加解密函数，发送数据到burpTracer.py
│   ├── proxy.js # 设置全局代理
│   ├── self_spit.js # 加密算法自吐
│   └── spawn.sh # frida spawn目标App
├── mitmproxy_script # 该目录存放所有mitmproxy脚本
│   ├── mitm.py # 过滤frida流量
│   └── mitm_http.py # 过滤http流量
├── rpc_server.py # 透明加密网关
├── scripts # 一些常用的命令
└── utils 
    ├── aes.py
    ├── log.py
    └── mysm4.py


```

需要修改的文件如下：

<table><thead><tr><th>文件名</th><th>功能</th><th><br></th></tr></thead><tbody><tr><td>hook.js</td><td>hook 加解密函数，把明文发送到 burpTracer.py</td><td><br></td></tr><tr><td>encrypt_rpc.js</td><td>主动调用加解密函数，提供 RPC 接口，用于透明加密网关</td><td><br></td></tr><tr><td>rpc_server.py</td><td>透明加密网关，根据实际情况调用 encrypt_rpc.js 中的接口</td><td><br></td></tr></tbody></table>

1.  使用方法
    

以默认配置为例。

启动 echoServer:

```
python echoServer.py


```

启动 mitmproxy：

```
mitmproxy -s mitmproxy_script/mitm.py --listen-host 0.0.0.0 -p 27081 --mode upstream:http://127.0.0.1:27080 -k


```

```
mitmproxy -s mitmproxy_script/mitm_http.py --listen-host 0.0.0.0 -p 8082 --mode upstream:http://127.0.0.1:8081 -k


```

Burp 设置 26080 端口监听，转发至 27081：

![](https://image.3001.net/images/20240513/1715573482_664192eaa947d1a6cb394.png!small)

手机端启动 APP，或者`hook_script/spawn.sh`启动；

启动 burpTracer, 并注入 js：

```
python burpTracer.py -s hook_script/hook.js -r 127.0.0.1:3333 -n AppName


```

启动透明加密网关：

```
python rpc_server.py hook_script/encrypt_rpc.js


```

或者一句话启动 burpTracer + 透明加密网关：

```
python burpTracer.py -s hook_script/hook.js -r 127.0.0.1:3333 -n AppName --rpc hook_script/encrypt_rpc.js -a hook_script/proxy.js


```

然后就可以愉快的渗透了！

8 总结
----

本文介绍了如何破解 App 流量加密以及工具化方法，首先介绍快速定位 App 加解密逻辑的方法，然后从被动触发、主动构造、透明加密代理服务器三个角度实现工具化，实现成一个 Cookiecutter 模版，大大减少了工具化成本。

感谢阅读，希望本文对你有所帮助！如有问题，请联系作者: [350717997@qq.com](mailto:350717997@qq.com).