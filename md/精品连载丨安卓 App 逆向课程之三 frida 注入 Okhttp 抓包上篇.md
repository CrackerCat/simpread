> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/F_UGRoAsfDW4SAa7cXMKrg)

“

阅读本文大概需要 3 分钟。

”

### 前言

抓包常常是 Android 协议分析的第一步，抓不到包困扰着众多爬虫工程师，因此很有必要抽丝剥茧，了解和学习 Android 的网络通信相关知识，并且打算写一些爬虫 er 学习安卓网络库的系列文章。

这几篇文章的主体思路的通过`Frida`来`Hook`网络框架`Okhttp`注入拦截器的方式抓包打印网络传输数据，相较于`Charles`，`Httpcanary`等抓包工具需设置复杂的环境，`Hook`网络框架进行抓包则直接输出安卓`app`网络层传输的内容，比较方便。

当然，同时也意味着此篇也是稍微高阶一些，算是想到哪儿写到哪儿吧，先写些难的，告诉大家结果，再写简单的内容，教大家如何使用`Frida`等等，帮助大家入门。

本篇文章过程会穿插介绍`Okhttp3`、`Frida`、`Xposed`、`Objection`等工具以及`Android`混淆等内容，本项目的工程代码已经放在我的星球中，我的星球非常便宜（只要 50 元），并且特地为崔大大生成了半价券，现在只要一杯咖啡就成，欢迎大家加入向我提问、共同交流：

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0bQOztjV4deVmxPicFXSLNvwMrGJHfBuVIyapeUNLuOND79L0r9hZQGw/640?wx_fmt=png)

（进入星球内搜索 “肉丝姐教你”）

本系列文章的目录最终会放在我的`Github`，欢迎大家点赞，蟹蟹大家：https://github.com/r0ysue/AndroidSecurityStudy

话不多说，我们进入正题。

### 1 Android 网络通信框架介绍

我们在这里只讨论原生 Android App 的网络通信框架。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0tfOP9ehhxRYicVbubKKuGzlsvR80aG8nR1hjicFzVp5UWKV72FaGp3Mw/640?wx_fmt=png)

以 Python 类比，Python3 有 urllib 和 urllib3 这样的原生网络请求库，也有 requests 库这样封装良好的第三方库，更加方便和优雅。Android 网络通信领域也一样，我们按照远近亲疏，也罗列一下 Android 中常用的网络通信框架。

#### 1.1 网络通信库概览

#### 1.1.1 原生网络通信库

•**HttpURLConnection**•**HttpClient**

这部分不用说太多，陈芝麻烂谷子的事儿了。从 Android 5（2014 年）开始，Android 官方不再推荐使用 HttpClient, Android 6.0 的 SDK 中去掉了 HttpCient，Android 9 后，Android 更是彻底取消了对 Apache HTTPClient 的支持, Android 开发官方推荐用 HttpUrlConnection。

在 Python 中 urllib2 已经可以很好的完成网络通信的相关工作，但耐不住 requests 更为优雅和简介。Android 世界也一样，一般实际开发并不会用 HttpURLConnection 和 HttpClient，而是使用经过时间和大量开发者验证的、封装良好的第三方网络请求框架，因为网络操作涉及异步、多线程以及效率的问题，自己搞吃力不讨好，因此我们直接看第三方网络请求框架吧。

#### 1.1.2 Okhttp3

OkHttp 是大名鼎鼎的 Square 公司的开源网络请求框架，Okhttp 有 2、3、4 这几个大版本，目前主流使用 Okhttp3，因此我们讨论 Okhttp3。Okhttp 本想做面向整个 Java 世界的网络框架，但从 OKhttp3 开始，似乎开始专注于 Android 领域，较新的版本都是用 Kotlin 编写和构建。Okhttp3 相比 HttpUrlConnection，更加优雅和高效，大部分其他 Android App 的网络框架，都是基于 Okhttp3 的再封装。因此 Okhttp3 是本篇文章的重点和轴心。

注：Okhttp 目前分为 Okhttp3 和 Okhttp4 两个大版本，目前主流的版本是 3，3 和 4 的 API 有不少变动，我们这里只讨论主流的 Okhttp3。除此之外，将 HttpUrlConnection 和 Okhttp3 类比，只是因为它们都 “比原生库优秀和更广泛使用”，这可以帮助理解，但两者是有区别的，requests 是基于 urllib3 的封装，但 Okhttp3 并非基于 HttpUrlConnection 或 HttpClient 的封装或补充，它在底层实现上完全自成一派，事实上，三个网络框架是平级关系，甚至构成竞争。从 Android 4.4 开始，HttpURLConnection 的底层实现已被 OkHttp 替代，由此可见 OkHttp3 是时下当之无愧最热门的 HTTP 框架。

#### 1.1.3 Retrofit2

Retrofit2 同样出自 Square 公司，Retrofit2 是对 Okhttp 的封装。

#### 1.1.4 Android-Async-Http

Android-Async-Http 是基于 HttpClient 封装的异步网络请求处理库，现在已经不怎么用了。一是因为 HttpClient 被 Android 弃用，二是因为框架作者已停止维护，这个库知道即可。

#### 1.1.5 Volley

Volley 在 2013 年的 Google I/O 大会上被推出，这是一款异步网络请求框架和图片加载框架。它特别适合数据量小，通信频繁的网络操作。它基于 HttpUrlConnection，目前也有一定的使用量。后续也会有关于这个框架的分析和实例讲解，这篇中不会做相应介绍。

综上所述，Okhttp3 是今天的重点。

#### 1.2 Okhttp3 DEMO App

使用 Okhttp3 简单写一个 DEMO APP，使用 Android Studio 创建应用。

#### 1.2.1 编写 DEMO App

STEP1 设置简单的点击按钮，点击一次反应一次。

activity_main.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center|center_horizontal|center_vertical"
tools:context=".MainActivity">
    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center|center_horizontal|center_vertical"
        android:id="@+id/mybtn"
        android:text="发送请求"
        android:textSize="45sp">
    </Button>
</LinearLayout>

```

MainActivity.java

```
package com.r0ysue.learnokhttp;
import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
public class MainActivity extends AppCompatActivity {
    private static String TAG = "learnokhttp";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 定位发送请求按钮
        Button btn = findViewById(R.id.mybtn);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.e(TAG, "点击");
            }
        });
    }
}

```

运行后尝试点击，运行正常。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB04B4dbBX2TiaGANZQlacIZkibUa6eDib8jIshqyApthy4PjovtqvnW1eQg/640?wx_fmt=png)

STEP2 配置 Okhttp 所需环境

在 app 级的 gradle 中增加对 okhttp3 的引用，修改后点击右上角 Sync Now 进行同步。

```
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.1.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test.ext:junit:1.1.1'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'
    // 增加对Okhttp3的依赖
    implementation("com.squareup.okhttp3:okhttp:3.12.0")
}

```

在 manifests 注册表中申请网络请求权限

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.learnmyokhttp">
    <!-- 申请网络请求权限 -->
    <uses-permission android: />
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:>
            <intent-filter>
                <action android: />
                <category android: />
            </intent-filter>
        </activity>
    </application>
</manifest>

```

STEP3 这才开始真正的工作，原本的逻辑是每次点击按钮时打印一条日志，修改成每次使用 Okhttp3 发出请求，访问百度首页。

新建类，example 类中包含了发起请求所需的最简代码

```
package com.r0ysue.learnokhttp;
import android.os.Build;
import android.util.Log;
import androidx.annotation.RequiresApi;
import java.io.IOException;
import okhttp3.Call;
import okhttp3.Callback;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
public class example {
    // TAG即为日志打印时的标签
    private static String TAG = "learnokhttp";
    // 新建一个Okhttp客户端
    OkHttpClient client = new OkHttpClient();
    void run(String url) throws IOException {
        // 构造request
        Request request = new Request.Builder()
                .url(url)
                .build();
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
}

```

同时修改 MainActivity.java 如下

```
package com.r0ysue.learnokhttp;
import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import java.io.IOException;
public class MainActivity extends AppCompatActivity {
    private static String TAG = "learnokhttp";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 定位发送请求按钮
        Button btn = findViewById(R.id.mybtn);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 访问百度首页
                String requestUrl = "https://www.baidu.com/";
                example myexample = new example();
                try {
                    myexample.run(requestUrl);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}

```

运行后点击 “发送请求” 按钮，日志中显示 Response 正常。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0icwiaUXd86cVDmAotcFQF74EzibnZBXoQwLNtiaStSElvZzDcggQtHZlDQ/640?wx_fmt=png)

在真实场景中，我们的抓包返回结果往往是 JSON 数据，因此替换访问 URL 为 "http://www.kuaidi100.com/query?type=yuantong&postid=11111111111", 每次返回随机的物流信息（查询结果可能为空）。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0sqbRfaoGXIzVXcqZTicDLkgYeZygljNRHPqyEt8qkoWZ7ficCEQqnIag/640?wx_fmt=png)

一个 DEMO App 完成了，同时我们看一下 Fiddler 抓包得到的请求和相应，从抓包结果可以看出，Okhttp 为我们默认配置了 Http 协议版本、部分 Headers 信息，这些内容也可以自定义添加。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0wK2OhaXxnicFrpTJGv0P8ZpUFXEaHvBlypOgmC6GOaP7gtzojW1PLUw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7FjVVl6jbtk0MT5vrc2GZB0NGCDX5RMKC3WJVt1j6gKcSYoIR1VjfHvu9DmsxIW3icwiaZmicJdZgLeA/640?wx_fmt=png)

#### 1.3 DEMO 流程分析

基于 DEMO，在这部分介绍一些 Okhttp3 的知识点。

#### 1.3.1 OkhttpClient 对象

在 example 类中，首先创建了一个 OkHttpClient 对象

```
OkHttpClient client = new OkHttpClient();

```

OkhttpClient 主要用来配置 Okhttp 框架的各种设置。你可能会怀疑 emmm，我们似乎并没有做什么设置，一个参数都没写, 其实在构造函数中默认诸多配置，比如超时等待时间，是否设置代理，SSL 验证，协议版本等等，我们也可以自定义配置如下，在此处先不详细展开。

```
OkHttpClient mHttpClient = new OkHttpClient.Builder()
        .readTimeout(5, TimeUnit.SECONDS)//设置读超时
        .writeTimeout(5,TimeUnit.SECONDS)////设置写超时
        .connectTimeout(15,TimeUnit.SECONDS)//设置连接超时
        .retryOnConnectionFailure(true)//是否自动重连
        .build();

```

#### 1.3.2 Request 对象

```
// 构造request
Request request = new Request.Builder()
        .url(url)
        .build();

```

接下来把目光转到我们自定义的 run 方法内，如果说 OkhttpClient 是在搭建生产车间，那 Request 即意味着每一个产品线，Request 使用建造者模式构建，可以在此环节添加 headers 参数等。

```
// 构造request
Request request = new Request.Builder()
    .url(url)
    .header(key, value)
    .header(key, value)
    ...
    .build();

```

#### 1.3.3 发起异步请求

```
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

```

在将 Request 对象封装成 Call 对象后，每次 enqueue 都会产生一次真实的网络请求。（网络请求可分为同步和异步方式，Android 中主要使用异步方式，因此我们这里直接不讲同步请求，除此之外，GET 和 POST 是两种常用的请求，这里先演示 GET 方式）。

本文先到这里，下一节我们真正开始抓包实操。

怎么样？如果大家对安卓逆向感兴趣，想学到更多的知识，或者想与肉丝姐进一步交流的话，欢迎加入肉丝姐的星球来学习。

这里我跟肉丝姐还申请到了专属的半价（原价 50 元）优惠，一杯咖啡的钱大家就能学到更多关于安卓逆向的知识，感兴趣的朋友来扫码加入吧。

![](https://mmbiz.qpic.cn/mmbiz_png/yjdDibu0qm7GDpTl7VFsAxaElZNpINjS8tjibkzq3tgqlWQ8sV9xQrhmHHgwiaogeukJRfbEokmia38Ayg5enYY9Ng/640?wx_fmt=png)