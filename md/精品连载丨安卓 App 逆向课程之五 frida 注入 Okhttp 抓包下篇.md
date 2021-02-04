> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/SBEKXSO6LrFYsO5pOtfxJA)

#### **本篇内容是「肉丝姐教你安卓逆向之 frida 注入 Okhttp 抓包系列的第三篇，建议配合前两篇一起阅读，效果更佳。**

*   #### [**精品连载丨安卓 App 逆向课程之三 frida 注入 Okhttp 抓包上篇**](http://mp.weixin.qq.com/s?__biz=MzIzNzA4NDk3Nw==&mid=2457739888&idx=1&sn=96ca660c676543f235f9b4c8c088300e&chksm=ff448a2ec8330338ccacf61e13c74cf5fd6d69559935073d134bba57d58172abc789110302d7&scene=21#wechat_redirect)
    
*    **[精品连载丨安卓 App 逆向课程之四 frida 注入 Okhttp 抓包中篇](http://mp.weixin.qq.com/s?__biz=MzIzNzA4NDk3Nw==&mid=2457739933&idx=2&sn=4e7d23739b7aa9169879e42bd64890d9&chksm=ff448ac3c83303d5caf409ddda6cd0fc533dd628eab546cdbb52f8bf4b7250740501bb4d01ab&scene=21#wechat_redirect)**
    

  

“

阅读本文大概需要 8 分钟。

”

#### 2.3 Yang Okhttp 拦截器思路讲解

接下来我们分析 Yang 大佬的 Frida 实现 okhttp3.Interceptor[1]。

代码完整如下，建议使用该份代码测试：

```
function hook_okhttp3() {
    // 1. frida Hook java层的代码必须包裹在Java.perform中，Java.perform会将Hook Java相关API准备就绪。
    Java.perform(function () {
        // 2. 准备相应类库，用于后续调用，前两个库是Android自带类库，后三个是使用Okhttp网络库的情况下才有的类
        var ByteString = Java.use("com.android.okhttp.okio.ByteString");
        var Buffer = Java.use("com.android.okhttp.okio.Buffer");
        var Interceptor = Java.use("okhttp3.Interceptor");
        var ArrayList = Java.use("java.util.ArrayList");
        var OkHttpClient = Java.use("okhttp3.OkHttpClient");
        //  注册一个Java类
        var MyInterceptor = Java.registerClass({
            name: "okhttp3.MyInterceptor",
            implements: [Interceptor],
            methods: {
                intercept: function (chain) {
                    var request = chain.request();
                    try {
                        console.log("MyInterceptor.intercept onEnter:", request, "\nrequest headers:\n", request.headers());
                        var requestBody = request.body();
                        var contentLength = requestBody ? requestBody.contentLength() : 0;
                        if (contentLength > 0) {
                            var BufferObj = Buffer.$new();
                            requestBody.writeTo(BufferObj);
                            try {
                                console.log("\nrequest body String:\n", BufferObj.readString(), "\n");
                            } catch (error) {
                                try {
                                    console.log("\nrequest body ByteString:\n", ByteString.of(BufferObj.readByteArray()).hex(), "\n");
                                } catch (error) {
                                    console.log("error 1:", error);
                                }
                            }
                        }
                    } catch (error) {
                        console.log("error 2:", error);
                    }
                    var response = chain.proceed(request);
                    try {
                        console.log("MyInterceptor.intercept onLeave:", response, "\nresponse headers:\n", response.headers());
                        var responseBody = response.body();
                        var contentLength = responseBody ? responseBody.contentLength() : 0;
                        if (contentLength > 0) {
                            console.log("\nresponsecontentLength:", contentLength, "responseBody:", responseBody, "\n");
                            var ContentType = response.headers().get("Content-Type");
                            console.log("ContentType:", ContentType);
                            if (ContentType.indexOf("video") == -1) {
                                if (ContentType.indexOf("application") == 0) {
                                    var source = responseBody.source();
                                    if (ContentType.indexOf("application/zip") != 0) {
                                        try {
                                            console.log("\nresponse.body StringClass\n", source.readUtf8(), "\n");
                                        } catch (error) {
                                            try {
                                                console.log("\nresponse.body ByteString\n", source.readByteString().hex(), "\n");
                                            } catch (error) {
                                                console.log("error 4:", error);
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } catch (error) {
                        console.log("error 3:", error);
                    }
                    return response;
                }
            }
        });
        OkHttpClient.$init.overload('okhttp3.OkHttpClient$Builder').implementation = function (Builder) {
            console.log("OkHttpClient.$init:", this, Java.cast(Builder.interceptors(), ArrayList));
            this.$init(Builder);
        };
        var MyInterceptorObj = MyInterceptor.$new();
        var Builder = Java.use("okhttp3.OkHttpClient$Builder");
        console.log(Builder);
        Builder.build.implementation = function () {
            this.interceptors().clear();
            this.interceptors().add(MyInterceptorObj);
            var result = this.build();
            return result;
        };
        Builder.addInterceptor.implementation = function (interceptor) {
            this.interceptors().clear();
            this.interceptors().add(MyInterceptorObj);
            return this;
        };
        console.log("hook_okhttp3...");
    });
}
hook_okhttp3();

```

#### 2.3.1 使用效果

接下来的演示效果均由 Pixel 展示

Yang 大佬使用在 Okhttp 中添加用户自定义拦截器的方式达到抓包效果，先前我们说过，App 不可能发送一次请求就创建一个 client 客户端，往往是全局一个 client，我们先修改 DEMO 代码，使之更接近真实 App。

```
package com.r0ysue.learnokhttp;
import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import java.io.IOException;
import okhttp3.Call;
import okhttp3.Callback;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
public class MainActivity extends AppCompatActivity {
    private static String TAG = "learnokhttp";
    public static final String requestUrl = "http://www.kuaidi100.com/query?type=yuantong&postid=11111111111";
    // 全局只使用这一个拦截器
    public static final OkHttpClient client = new OkHttpClient.Builder()
            .addNetworkInterceptor(new LoggingInterceptor())
            .build();
    Request request = new Request.Builder()
            .url(requestUrl)
            .build();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 定位发送请求按钮
        Button btn = findViewById(R.id.mybtn);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 发起异步请求
                client.newCall(request).enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        call.cancel();
                    }
                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        //打印输出
                        Log.d(TAG,  response.body().string());
                    }
                                                }
                );
            }
        });
    }
}

```

Frida 有 spawn 和 attach 两种启动方式，接下来使用 Spawn 模式和 Attach 模式分别测试，attach 模式下，Frida 会附加到当前的目标进程中，即需要 App 处于启动状态，这也意味着只能从当前时机往后 Hook，而 spawn 模式下，Frida 会自行启动并注入进目标 App，Hook 的时机非常早，好处在于不会错过 App 中相对较早 (比如 App 启动时产生的参数), 缺点是假如想要 Hook 的时机点偏后，则会带来大量干扰信息，严重甚至会导致 server 崩溃。

之前我们提过，App 全局只有一个 client，因此它在 App 启动的较早时机被创建，如果采用 attach 模式 Hook OkhttpClient，大概率会一无所获。六月天想看樱花——你来晚了。

因此只能用 Spawn 模式启动，对应 frida 命令即必须使用 - f 参数：

```
frida -U -f com.r0ysue.learnokhttp -l C:\Users\Lenovo\Desktop\抓包\teach\yang1.js --no-pause

```

```
C:\Users\Lenovo>frida -U -f com.r0ysue.learnokhttp -l C:\Users\Lenovo\Desktop\抓包\teach\yang1.js --no-pause
     ____
/ _  |   Frida12.8.14- A world-classdynamic instrumentation toolkit
| (_| |
> _  |   Commands:
/_/ |_|       help      -> Displays the help system
. . . .       object?   -> Display information about 'object'
. . . .       exit/quit -> Exit
. . . .
. . . .   More info at https://www.frida.re/docs/home/
Spawned`com.r0ysue.learnokhttp`. Resuming main thread!
TypeError: cannot read property'apply' of undefined
    at [anon] (../../../frida-gum/bindings/gumjs/duktape.c:56618)
    at frida/runtime/core.js:55
[Pixel::com.r0ysue.learnokhttp]-> <class: okhttp3.OkHttpClient>
<class: okhttp3.OkHttpClient$Builder>
hook_okhttp3...
OkHttpClient.$init: okhttp3.OkHttpClient@ba4f0a3[okhttp3.MyInterceptor@79e4fa0]
MyInterceptor.intercept onEnter: Request{method=GET, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111, tags={}}
request headers:
MyInterceptor.intercept onLeave: Response{protocol=http/1.1, code=200, message=OK, url=http://www.kuaidi100.com/query?type=yuantong&postid=11111111111}
response headers:
Server: nginx
Date: Mon, 25May202015:59:33 GMT
Content-Type: text/html;charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
P3P: CP="IDC DSP COR ADM DEVi TAIi PSA PSD IVAi IVDi CONi HIS OUR IND CNT"
Cache-Control: no-cache
vary: accept-encoding

```

打印出了 Request 的信息以及 Response 的部分内容，但缺少响应体，整体似乎有不少可以补充的地方，具体看一下代码。

#### 2.3.2 代码分析

代码虽然才几十行，但对于新手来说，可能会略显复杂，我们解构一下，从无到有重新实现一下：

STEP 1

```
function hook_okhttp3() {
    // frida Hook java层的代码必须包裹在Java.perform中，Java.perform会将Hook Java相关API准备就绪。
    Java.perform(function () {
        console.log("hook_okhttp3...");
    });
}
hook_okhttp3();

```

Java.perform（fn）主要用于当前线程附加到 Java VM 并且调用 fn 方法。frida Hook java 层的代码必须包裹在 Java.perform 中，Java.perform 会将 Hook Java 相关 API 准备就绪。

STEP 2 实现自定义 interceptor

```
// 获取interceptor类
var Interceptor = Java.use("okhttp3.Interceptor");
//  注册一个Java类
var MyInterceptor = Java.registerClass({
    name: "okhttp3.MyInterceptor",
    implements: [Interceptor],
    methods: {
        intercept: function (chain) {
            var request = chain.request();
            try {
                console.log("MyInterceptor.intercept onEnter:", request, "\nrequest headers:\n", request.headers());
                var requestBody = request.body();
                var contentLength = requestBody ? requestBody.contentLength() : 0;
                if (contentLength > 0) {
                    var BufferObj = Buffer.$new();
                    requestBody.writeTo(BufferObj);
                    try {
                        console.log("\nrequest body String:\n", BufferObj.readString(), "\n");
                    } catch (error) {
                        try {
                            console.log("\nrequest body ByteString:\n", ByteString.of(BufferObj.readByteArray()).hex(), "\n");
                        } catch (error) {
                            console.log("error 1:", error);
                        }
                    }
                }
            } catch (error) {
                console.log("error 2:", error);
            }
            var response = chain.proceed(request);
            try {
                console.log("MyInterceptor.intercept onLeave:", response, "\nresponse headers:\n", response.headers());
                var responseBody = response.body();
                var contentLength = responseBody ? responseBody.contentLength() : 0;
                if (contentLength > 0) {
                    console.log("\nresponsecontentLength:", contentLength, "responseBody:", responseBody, "\n");
                    var ContentType = response.headers().get("Content-Type");
                    console.log("ContentType:", ContentType);
                    if (ContentType.indexOf("video") == -1) {
                        if (ContentType.indexOf("application") == 0) {
                            var source = responseBody.source();
                            if (ContentType.indexOf("application/zip") != 0) {
                                try {
                                    console.log("\nresponse.body StringClass\n", source.readUtf8(), "\n");
                                } catch (error) {
                                    try {
                                        console.log("\nresponse.body ByteString\n", source.readByteString().hex(), "\n");
                                    } catch (error) {
                                        console.log("error 4:", error);
                                    }
                                }
                            }
                        }
                    }
                }
            } catch (error) {
                console.log("error 3:", error);
            }
            return response;
        }
    }
});

```

这部分代码量比较多，但实际上根本不用慌，我们拆解一下。首先，它的意图是在 App 中注册一个 Java 类，在第二节我们演示过自定义拦截器，换而言之，STEP2 相当于在我们正向开发中，新建了一个类，实现了 interceptor 接口，是一个正儿八经的用户自定义拦截器。

看一下 API Java.registerClass：创建一个新的 Java 类并返回一个包装器，规范如下：

name：指定类名称的字符串。

superClass：（可选）父类。要从 java.lang.Object 继承的省略。

implements：（可选）由此类实现的接口数组。

fields：（可选）对象，指定要公开的每个字段的名称和类型。

methods：（可选）对象，指定要实现的方法。

在此处：

```
name: "okhttp3.MyInterceptor",  //全类名：okhttp3.MyInterceptor，类名：MyInterceptor
implements: [Interceptor], // 实现Interceptor接口，即为一个拦截器
   methods: {
       // 该类中只有一个方法，即实现了Interceptor的interceptor方法
        intercept: function (chain) {
            // 具体逻辑
        }
    }

```

换而言之，上述 Frida 中的操作，与如下 JAVA 类等价：

```
package okhttp3;
import java.io.IOException;
public class MyInterceptor implements Interceptor{
    @Override
    public Response intercept(Chain chain) throws IOException {
        // 具体逻辑
        return null;
    }
}

```

接下来看 interceptor 中的具体实现，看一下其对 Request 的处理：

```
var request = chain.request();
try {
    console.log("MyInterceptor.intercept onEnter:", request, "\nrequest headers:\n", request.headers());
    var requestBody = request.body();
    var contentLength = requestBody ? requestBody.contentLength() : 0;
    if (contentLength > 0) {
        var BufferObj = Buffer.$new();
        requestBody.writeTo(BufferObj);
        try {
            console.log("\nrequest body String:\n", BufferObj.readString(), "\n");
        } catch (error) {
            try {
                console.log("\nrequest body ByteString:\n", ByteString.of(BufferObj.readByteArray()).hex(), "\n");
            } catch (error) {
                console.log("error 1:", error);
            }
        }
    }
} catch (error) {
    console.log("error 2:", error);
}

```

我将其翻译成 java，可以一一对照：MyInterceptor.java

```
package com.r0ysue.learnokhttp;
import android.util.Log;
import java.io.IOException;
import java.nio.charset.Charset;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.ResponseBody;
import okio.Buffer;
import okio.ByteString;
public class MyInterceptor implements Interceptor {
    private static String TAG = "learnokhttp";
    private final Charset UTF8 = Charset.forName("UTF-8");
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        try{
            // 打印GET/POST，URL，HEADERS
            Log.i(TAG, "MyInterceptor.intercept onEnter:"+request+"\nrequest headers:\n"+request.headers());
            // 如果请求方式为POST，下面的逻辑负责打印RequestBody
            RequestBody requestBody = request.body();
            if(requestBody != null){
                long contentLength = requestBody.contentLength();
                if(contentLength != 0){
                    Buffer BufferObj = new Buffer();
                    requestBody.writeTo(BufferObj);
                    try {
                        Log.i(TAG, "\nrequest body String:\n"+BufferObj.readString(UTF8)+"\n");
                    }catch (Exception e){
                        try {
                            Log.i(TAG, "\nrequest body ByteString:\n"+ByteString.of(BufferObj.readByteArray()).hex()+"\n");
                        }catch (Exception e1){
                            Log.i(TAG, "error 1:");
                        }
                    }
                }
            }
        }catch (Exception e2){
            Log.i(TAG, "error 2:");
        }
        Response response = chain.proceed(request);
        return response;
    }
}

```

将此拦截器放入 client，测试效果：MainActivity.java 中：

```
publicstaticfinalOkHttpClient client = newOkHttpClient.Builder()
.addNetworkInterceptor(newMyInterceptor())
.build();

```

运行结果正常：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7EHh7z3fibt0h7d5J9EMPWY8boksOGezt7LKSBYtdvvNKJG5AGK0CsLNCXUc07ZbmVUrxxCEpqMUGQ/640?wx_fmt=png)

Response 的逻辑也类似，在此不做额外讲解。

因此我们可以理解成，STEP1+STEP2 后，好似在原 App 中添加了一个自定义拦截器链，那剩下的工作应该就是将我们的自定义拦截器添加到拦截器链里，在开发中，我们只用如下一行代码，但逆向中似乎不是这么容易。

```
.addNetworkInterceptor(newMyInterceptor())

```

STEP 3 添加拦截器 这部分代码可能存在一些问题，我问 yang 神，他说也忘记当时为啥这么写了：

```
OkHttpClient.$init.overload('okhttp3.OkHttpClient$Builder').implementation = function (Builder) {
            console.log("OkHttpClient.$init:", this, Java.cast(Builder.interceptors(), ArrayList));
            this.$init(Builder);
        };
var MyInterceptorObj = MyInterceptor.$new();
var Builder = Java.use("okhttp3.OkHttpClient$Builder");
console.log(Builder);
Builder.build.implementation = function () {
    this.interceptors().clear();
    this.interceptors().add(MyInterceptorObj);
    var result = this.build();
    return result;
};
Builder.addInterceptor.implementation = function (interceptor) {
    this.interceptors().clear();
    this.interceptors().add(MyInterceptorObj);
    return this;
};

```

简而言之，一共选择了三个 Hook 点，我们在正向开发中标出它们的位置，需要注意，Builder 类是 Okhttpclient 中的内部类，Java 编译器会将内部类编译成外部类名 $ 内部类名格式，因此不论 Frida 还是 Xposed 中，如果我们想对内部类进行操作，都应该使用 $ 连接符。

Hook 点 1——Okhttpclient 的有参构造函数，参数为 Builder

```
OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    this.protocols = builder.protocols;
}

```

先前我们讲过三种 client 创建的方式，每一种都必经过此构造函数，因此可以避免遗漏，yang 大佬在此选择了简单打印对象。

Hook 点 2 和 3：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7EHh7z3fibt0h7d5J9EMPWY872dibFSuhzx9Gd9ib7S9tgLF0g7sVHyXuMvIYjicGiam8xHrbqibibtqopBw/640?wx_fmt=png)

如果采用默认方式创建 Okhttpclient，这两个 Hook 点就会失效，且在大佬的 hook 代码逻辑中，会将原拦截器数组清空，这可能会造成 App 本身拦截器失效或者无法访问网络，我们不妨做一些修改。

首先选择 Hook 点，我们使用 Hook 点 2，开发中很少会使用默认方式创建 client。

将源代码中 STEP 3 做删减，如下：

```
var MyInterceptorObj = MyInterceptor.$new();
var Builder = Java.use("okhttp3.OkHttpClient$Builder");
console.log(Builder);
Builder.build.implementation = function () {
    this.interceptors().add(MyInterceptorObj);
    return this.build();
};

```

测试后打印内容与原先不变，只做到这里多少有些不够味儿，下面开始整活儿。

#### 2.4 天外飞仙拦截器

可以从 2.3 看出，通过 Hook 新增拦截器来实现打印内容是有效果的，但脚本远远称不上完善，多少有点鸡肋，除此之外，Java 层面的拦截器逻辑在 Frida 中编写多少有些不自在，有隔靴搔痒之感。

Frida 提供了如下 API 用于将 DEX 加载进内存，从而使用 DEX 中的方法和类，因为 DEX 是外来之物，因此称为天外飞仙。（需要注意的是，无法加载 JAR 包）：

```
Java.openClassFile(dexPath).load();

```

2.3 中依照 yang 的 Hook 脚本，编写了对应的 MyInterceptor.java 类 (有所阉割，只实现了 request 部分的逻辑处理)

#### 2.4.1 加载自定义 DEX

接下来我们取出 App 中的 DEX，如果有多 DEX，则用 JADX 查看想要使用的类在哪一个 DEX 中，最后 push 到手机，最后调用。

检查 MyInterceptor.java 是否编写正确，编译：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7EHh7z3fibt0h7d5J9EMPWY8L4WpzxMrBfiblhstYVsKAicoHpCN1yhxDfgjibUeNFfC5ia9f8iaibFoGsnQ/640?wx_fmt=png)在 DEMO APK 项目目录中找到如下位置  

```
C:\xxx\xxx\learnokhttp\app\build\outputs\apk\debug

```

(1). 解压 app-debug.apk 取出 classes1.dex 文件 (其中有目标类)

(2).push 到 / data/local/tmp 下

```
C:\Users\Lenovo>adb push C:\xxx\learnokhttp\app\build\outputs\apk\debug\classes.dex /data/local/tmp
C:\Users\Lenovo\Desktop\teach\AndroidProj\learnokhttp\app\...ile pushed, 0 skipped. 15.0 MB/s (2485932 bytes in0.158s)

```

(3). 修改 Frida Hook 代码：

```
function hook_okhttp3() {
    Java.perform(function () {
        Java.openClassFile("/data/local/tmp/classes.dex").load();
        var MyInterceptor = Java.use("com.r0ysue.learnokhttp.MyInterceptor");
        var MyInterceptorObj = MyInterceptor.$new();
        var Builder = Java.use("okhttp3.OkHttpClient$Builder");
        console.log(Builder);
        Builder.build.implementation = function () {
            this.interceptors().add(MyInterceptorObj);
            return this.build();
        };
        console.log("hook_okhttp3...");
    });
}
hook_okhttp3();

```

可以发现整体代码量大减，这是因为原先创建拦截器类的逻辑被写在了 dex 中。(4).Frida Hook，在此之前将 DEMO 中 OKhttpclient 的添加拦截器代码注销，以防干扰，查看 Android Studio 日志，拦截器是否生效。

Frida 端:

```
 C:\Users\Lenovo>frida -U -f com.r0ysue.learnokhttp -l C:\Users\Lenovo\Desktop\抓包\teach\yang2.js  --no-pause
     ____
/ _  |   Frida12.8.14- A world-classdynamic instrumentation toolkit
| (_| |
> _  |   Commands:
/_/ |_|       help      -> Displays the help system
. . . .       object?   -> Display information about 'object'
. . . .       exit/quit -> Exit
. . . .
. . . .   More info at https://www.frida.re/docs/home/
Spawned`com.r0ysue.learnokhttp`. Resuming main thread!
[Pixel::com.r0ysue.learnokhttp]->
[Pixel::com.r0ysue.learnokhttp]-> <okhttp3.OkHttpClient$Builder>
hook_okhttp3...

```

Android Studio 日志查看：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7EHh7z3fibt0h7d5J9EMPWY8nicZ9Dd9qTqwrsu7y5icmwaqRiclybjWU5vLRq2TRtNAbsgdVZ53lGpow/640?wx_fmt=png)

红色框中即为我们拦截器输出的内容，可以发现，headers 为空，这是因为我们拦截器添加的位置，我们在 Frida 代码中为应用添加的是 Application Interceptor，这个拦截器在 BridgeInterceptor 等拦截器前，因此如果是 BridgeInterceptor 中添加的 headers 字段等，无法通过 Application Interceptor 打印出来，修改 Frida 代码，改成 Network Application。

修改代码，将 interceptor 修改成如下：

```
Builder.build.implementation = function() {
// 原先添加到interceptors(即Application Interceptor)
// 修改为添加至networkInterceptors
this.networkInterceptors().add(MyInterceptorObj);
returnthis.build();
};

```

重新测试，结果符合预期：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7EHh7z3fibt0h7d5J9EMPWY82QiblV0wCV72IlmbBictQkn2YD0hdqO6SMK2BWoicDvbBRDrUsX4cS5kw/640?wx_fmt=png)

#### 2.4.2 加载 Okhttp logging-interceptor

Okhttp 官方也提供了一款简单易用的日志打印拦截器——okhttp3:logging-interceptor

对其稍作修改，完整 Java 代码如下

```
package com.r0ysue.learnokhttp;
/*
 * Copyright (C) 2015 Square, Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
import android.util.Log;
import java.io.EOFException;
import java.io.IOException;
import java.nio.charset.Charset;
import java.util.concurrent.TimeUnit;
import okhttp3.Connection;
import okhttp3.Headers;
import okhttp3.Interceptor;
import okhttp3.MediaType;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.ResponseBody;
import okhttp3.internal.http.HttpHeaders;
import okio.Buffer;
import okio.BufferedSource;
import okio.GzipSource;
public final class okhttp3Logging implements Interceptor {
    private static final String TAG = "okhttpGET";
    private static final Charset UTF8 = Charset.forName("UTF-8");
    @Override public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        RequestBody requestBody = request.body();
        boolean hasRequestBody = requestBody != null;
        Connection connection = chain.connection();
        String requestStartMessage = "--> "
                + request.method()
                + ' ' + request.url();
        Log.e(TAG, requestStartMessage);
        if (hasRequestBody) {
            // Request body headers are only present when installed as a network interceptor. Force
            // them to be included (when available) so there values are known.
            if (requestBody.contentType() != null) {
                Log.e(TAG, "Content-Type: " + requestBody.contentType());
            }
            if (requestBody.contentLength() != -1) {
                Log.e(TAG, "Content-Length: " + requestBody.contentLength());
            }
        }
        Headers headers = request.headers();
        for (int i = 0, count = headers.size(); i < count; i++) {
            String name = headers.name(i);
            // Skip headers from the request body as they are explicitly logged above.
            if (!"Content-Type".equalsIgnoreCase(name) && !"Content-Length".equalsIgnoreCase(name)) {
                Log.e(TAG, name + ": " + headers.value(i));
            }
        }
        if (!hasRequestBody) {
            Log.e(TAG, "--> END " + request.method());
        } else if (bodyHasUnknownEncoding(request.headers())) {
            Log.e(TAG, "--> END " + request.method() + " (encoded body omitted)");
        } else {
            Buffer buffer = new Buffer();
            requestBody.writeTo(buffer);
            Charset charset = UTF8;
            MediaType contentType = requestBody.contentType();
            if (contentType != null) {
                charset = contentType.charset(UTF8);
            }
            Log.e(TAG, "");
            if (isPlaintext(buffer)) {
                Log.e(TAG, buffer.readString(charset));
                Log.e(TAG, "--> END " + request.method()
                        + " (" + requestBody.contentLength() + "-byte body)");
            } else {
                Log.e(TAG, "--> END " + request.method() + " (binary "
                        + requestBody.contentLength() + "-byte body omitted)");
            }
        }
        long startNs = System.nanoTime();
        Response response;
        try {
            response = chain.proceed(request);
        } catch (Exception e) {
            Log.e(TAG, "<-- HTTP FAILED: " + e);
            throw e;
        }
        long tookMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startNs);
        ResponseBody responseBody = response.body();
        long contentLength = responseBody.contentLength();
        String bodySize = contentLength != -1 ? contentLength + "-byte" : "unknown-length";
        Log.e(TAG, "<-- "
                + response.code()
                + (response.message().isEmpty() ? "" : ' ' + response.message())
                + ' ' + response.request().url()
                + " (" + tookMs + "ms" + (", " + bodySize + " body:" + "") + ')');
        Headers myheaders = response.headers();
        for (int i = 0, count = myheaders.size(); i < count; i++) {
            Log.e(TAG, myheaders.name(i) + ": " + myheaders.value(i));
        }
        if (!HttpHeaders.hasBody(response)) {
            Log.e(TAG, "<-- END HTTP");
        } else if (bodyHasUnknownEncoding(response.headers())) {
            Log.e(TAG, "<-- END HTTP (encoded body omitted)");
        } else {
            BufferedSource source = responseBody.source();
            source.request(Long.MAX_VALUE); // Buffer the entire body.
            Buffer buffer = source.buffer();
            Long gzippedLength = null;
            if ("gzip".equalsIgnoreCase(myheaders.get("Content-Encoding"))) {
                gzippedLength = buffer.size();
                GzipSource gzippedResponseBody = null;
                try {
                    gzippedResponseBody = new GzipSource(buffer.clone());
                    buffer = new Buffer();
                    buffer.writeAll(gzippedResponseBody);
                } finally {
                    if (gzippedResponseBody != null) {
                        gzippedResponseBody.close();
                    }
                }
            }
            Charset charset = UTF8;
            MediaType contentType = responseBody.contentType();
            if (contentType != null) {
                charset = contentType.charset(UTF8);
            }
            if (!isPlaintext(buffer)) {
                Log.e(TAG, "");
                Log.e(TAG, "<-- END HTTP (binary " + buffer.size() + "-byte body omitted)");
                return response;
            }
            if (contentLength != 0) {
                Log.e(TAG, "");
                Log.e(TAG, buffer.clone().readString(charset));
            }
            if (gzippedLength != null) {
                Log.e(TAG, "<-- END HTTP (" + buffer.size() + "-byte, "
                        + gzippedLength + "-gzipped-byte body)");
            } else {
                Log.e(TAG, "<-- END HTTP (" + buffer.size() + "-byte body)");
            }
        }
        return response;
    }
    /**
     * Returns true if the body in question probably contains human readable text. Uses a small sample
     * of code points to detect unicode control characters commonly used in binary file signatures.
     */
    static boolean isPlaintext(Buffer buffer) {
        try {
            Buffer prefix = new Buffer();
            long byteCount = buffer.size() < 64 ? buffer.size() : 64;
            buffer.copyTo(prefix, 0, byteCount);
            for (int i = 0; i < 16; i++) {
                if (prefix.exhausted()) {
                    break;
                }
                int codePoint = prefix.readUtf8CodePoint();
                if (Character.isISOControl(codePoint) && !Character.isWhitespace(codePoint)) {
                    return false;
                }
            }
            return true;
        } catch (EOFException e) {
            return false; // Truncated UTF-8 sequence.
        }
    }
    private boolean bodyHasUnknownEncoding(Headers myheaders) {
        String contentEncoding = myheaders.get("Content-Encoding");
        return contentEncoding != null
                && !contentEncoding.equalsIgnoreCase("identity")
                && !contentEncoding.equalsIgnoreCase("gzip");
    }
}

```

同 2.4.1 操作，编译——取出`dex`改名为`okhttp3logging.dex`，`push`到`/data/locol/tmp`目录下，`frida`代码修改如下：

```
function hook_okhttp3() {
    // 1. frida Hook java层的代码必须包裹在Java.perform中，Java.perform会将Hook Java相关API准备就绪。
    Java.perform(function () {
        Java.openClassFile("/data/local/tmp/okhttplogging.dex").load();
        // 只修改了这一句，换句话说，只是使用不同的拦截器对象。
        var MyInterceptor = Java.use("com.r0ysue.learnokhttp.okhttp3Logging");
        var MyInterceptorObj = MyInterceptor.$new();
        var Builder = Java.use("okhttp3.OkHttpClient$Builder");
        console.log(Builder);
        Builder.build.implementation = function () {
            this.networkInterceptors().add(MyInterceptorObj);
            return this.build();
        };
        console.log("hook_okhttp3...");
    });
}
hook_okhttp3();

```

打印结果十分好，几乎和抓包能得到的信息一样多。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7EHh7z3fibt0h7d5J9EMPWY8ibR6ywK3WnhHVITicVra1dRly8icjdVPnmbBmsQQmS9mKDIEMGAaQgxrQ/640?wx_fmt=png)

小总结：

在本篇文章中，我们学习了安卓中应用最为基本的网络库`Okhttp`，并通过小`Demo`学习其基本开发方法，进一步探索定位拦截位置，最后通过`Frida`构造一个拦截器并挂载，打印出通过`Okttp`传输的所有内容。

下一篇会关注这几个要点：

1. 当前的`okhttp3logging`够好了吗？有没有办法让信息更清晰？或者功能更强大。2. 一定要用 Spawn 模式启动吗？Attach 方式常常更方便，是否能在 Attach 模式下也添加拦截器。3. 混淆怎么办？混淆是否会对 Hook 产生影响？面对一般混淆是否有办法自识别？4.App 加固是否会对 Hook 产生影响？

敬请期待。

### References

`[1]` Frida 实现 okhttp3.Interceptor: _https://bbs.pediy.com/thread-252129.htm_

怎么样？如果大家对安卓逆向感兴趣，想学到更多的知识，或者想与肉丝姐进一步交流的话，欢迎加入肉丝姐的星球来学习。

这里我跟肉丝姐还申请到了专属的半价（原价 50 元）优惠，一杯咖啡的钱大家就能学到更多关于安卓逆向的知识，感兴趣的朋友来扫码加入吧。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7G3vcLPYLZnBlHySsX5WyXkqpNQYPicAbXVfy2lNKict2cmqqWF8SwoibTiacd9eQiczvuHLEPIwI7gSiaA/640?wx_fmt=png)