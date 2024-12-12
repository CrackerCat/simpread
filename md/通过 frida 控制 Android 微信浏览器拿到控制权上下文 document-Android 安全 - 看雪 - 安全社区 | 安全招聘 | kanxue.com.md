> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284761.htm)

> 通过 frida 控制 Android 微信浏览器拿到控制权上下文 document

通过 frida 控制 Android 微信浏览器拿到控制权上下文 document

发表于: 9 小时前 618

### 通过 frida 控制 Android 微信浏览器拿到控制权上下文 document

 [![](http://passport.kanxue.com/upload/avatar/524/885524.png?1587310702)](user-home-885524.htm) [kzzll](user-home-885524.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 9 小时前  618

> 段落引用 代码前几个月的了，具体的逆向过程就不讲解了，直接上代码，代码已经注释了具体

**主要 hook 点：**  
com.tencent.xweb.WebView.loadUrl(String url)  
通过 this 拿到当前类实例，最后可以检测浏览器是否加载完毕，执行 js 代码等操作

```
/*
    hook微信浏览器，拿到webview
    使用frida反射java调用http请求
    最后通过执行js代码
*/
 
    function log(text) {
        console.log(text)
        //Java.send({"act":"Frida_Log","msg": text});
    }
    var analyToken = "4aab795025bbeee52bca2a8ee4123cb5"
    var mWebView = null;
    var analyGoodsInfo = {}
    var historyUrl;
 
   
 
    function httpGet(url){
        var URL = Java.use("java.net.URL");
        var HttpURLConnection = Java.use("java.net.HttpURLConnection");
        var BufferedReader = Java.use("java.io.BufferedReader");
        var InputStreamReader = Java.use("java.io.InputStreamReader");
 
        var urlObj = URL.$new(url);
        var connection = urlObj.openConnection();
        var inputStream = connection.getInputStream();
        var inputStreamReader = InputStreamReader.$new(inputStream);
        var bufferedReader = BufferedReader.$new(inputStreamReader);
 
        var response = "";
        var line;
        while ((line = bufferedReader.readLine()) !== null) {
        response += line;
        }   
        try{
            bufferedReader.close();
            inputStreamReader.close();
            inputStream.close();
            connection.disconnect();
        }catch(e){
            console.log(e)
        }
         
        return response
    }
 
    function httpPost(url,data){
        var URL = Java.use("java.net.URL");
        var HttpURLConnection = Java.use("java.net.HttpURLConnection");
        var OutputStreamWriter = Java.use("java.io.OutputStreamWriter");
 
        var urlObj = URL.$new(url);
        var connection = urlObj.openConnection();
        connection.setDoOutput(true);
        connection.setRequestMethod("POST");
        connection.setRequestProperty("Content-Type", "application/json");
        connection.setRequestProperty("Accept", "application/json");
        var wr = OutputStreamWriter.$new(connection.getOutputStream());
        wr.write(data);
        wr.flush();
 
        var BufferedReader = Java.use("java.io.BufferedReader");
        var InputStreamReader = Java.use("java.io.InputStreamReader");
        var inputStream = connection.getInputStream();
        var inputStreamReader = InputStreamReader.$new(inputStream);
        var bufferedReader = BufferedReader.$new(inputStreamReader);
 
        var response = "";
        var line;
        while ((line = bufferedReader.readLine()) !== null) {
        response += line;
        }
        console.log(response);
        
        try{
            bufferedReader.close();
            inputStreamReader.close();
            inputStream.close();
            connection.disconnect();
        }catch(e){
            console.log(e)
        }
         
        return response
    }
 
    function toast(message) {
        try{
            console.log(message)
        }catch(e){
             
        }
        Java.perform(function() {
            var Looper = Java.use("android.os.Looper");
            // 检查当前线程是否已有Looper，如果没有，则调用prepare()
            if (Looper.myLooper() === null) {
                Looper.prepare();
            }
     
            var Toast = Java.use("android.widget.Toast");
            var context = Java.use("android.app.ActivityThread").currentApplication().getApplicationContext();
            var javaString = Java.use("java.lang.String").$new(message);
            Toast.makeText(context, javaString, Toast.LENGTH_SHORT.value).show();
     
            // 启动消息队列处理，仅在调用了prepare()后才调用loop()
            if (Looper.myLooper() === Looper.getMainLooper()) {
                Looper.loop();
            }
        });
    }
     
     
    function analys(){
        try{
            var sj = setInterval(function(){
                var analyGoodsId = analyGetGoods()
                if(analyGoodsId){
                    var rawGoodsId = historyUrl.split("goods_id=")[1].split("&")[0]
                    mWebView.loadUrl(historyUrl.replace(rawGoodsId,analyGoodsId))
                    clearInterval(sj)
                }else{
                    toast("暂无可解析商品，稍后重新获取")
                }
            },2000);
        }catch(e){
            toast("跳转商品出错，浏览器可能关闭")
            clearInterval(sj)
        }
        
    }
 
    function analyGetGoods(){
        var data = httpGet("http://localhost:8080/get_goods")
        try{
            var retJson = JSON.parse(data)
            if(retJson['code'] == 0){
                analyGoodsInfo = retJson;
                return retJson['goods_id'];
            }
        }catch(e){
            console.log("获取失败",e)
        }
        return null
    }
 
    Java.perform(function () {
        toast("开始Hook微信浏览器")
        let WebView = Java.use("com.tencent.xweb.WebView");
        WebView["loadUrl"].overload('j  ').implementation = function (str) {
            var myWebView = this;
            myWebView.loadUrl(str);
            mWebView = Java.retain(myWebView);
 
            //toast("等待页面加载完毕...")
 
            var sj = setInterval(function(){
                try{
                    var title = mWebView.getTitle();
                    var progress = mWebView.getProgress()
                    var url = mWebView.getUrl();
                    if(title == null && url == null){
                        //浏览器可以已经关闭，停止循环
                        mWebView = null;
                        clearInterval(sj);
                    }
                     
                     //等待页面加载完毕
                    if(progress == 100 && historyUrl != url){
                        console.log("加载完成",url)
                        historyUrl = url;
                        var goods_id = undefined;
                        //从url获取goods_id参数
                        try{
                            goods_id = url.split("goods_id=")[1].split("&")[0]
                        }catch(e){
                        }
                         
                        if(analyGoodsInfo && goods_id){
                            if(goods_id == analyGoodsInfo['goods_id']){
                                toast("读取商品数据并上报:"+goods_id)
                                
                                var script_str =   "(function() {" +
                                " window.analyGoodsInfo = '"+JSON.stringify(analyGoodsInfo)+"';"+
                                "   var script = document.createElement('script');"+
                                "   script.type = 'text/javascript';" +
                                "   script.src = 'http://localhost:8000/wechat_post.js?t='+ new Date().getTime();"
                                "   document.body.appendChild(script);" +
                                "})()"
 
                     
                                 
                                mWebView.evaluateJavascript(script_str, null);
 
                                //模拟上报完成，5秒后加载新商品
                                setTimeout(function(){
                                    analys()
                                },6000)
                            }else{
                                toast("拉取商品并跳转中...")
                                analys()
                            }
                            clearInterval(sj);
                        }else{
                            toast("当前非多多页面")
                            
                        }
                        //
                         
                         
                    }
 
                }catch(e){
                    console.log(e)
                }
            },200)
        };
    });
 
 
    setInterval(function() {
        //log("Frida Process is "+Process.id)
    }, 3 * 1000);
```

  

[[注意] 传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm)