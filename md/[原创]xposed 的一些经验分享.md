> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-263239.htm)

*   **加壳的应用上如何用 xposed**
    --------------------
    
    如下代码是某个通过棒棒加固的 APP，首先获取到加固替换掉的 Application 然后再获取到 ClassLoader, 其实跟 Xposed 的 loadPackageParam.classLoader 一个原理，只是加固应用多了一步。
    

```
final Class clazz = XposedHelpers.findClassIfExists("com.secneo.apkwrapper.AW",loadPackageParam.classLoader);
if(clazz!=null) {
    XposedHelpers.findAndHookMethod(clazz, "onCreate", new XC_MethodHook() {
        public void afterHookedMethod(XC_MethodHook.MethodHookParam param) throws Throwable {
 
                sApplication = (Application)XposedHelpers.getStaticObjectField(clazz,"a");
                sClassLoader = syApplication.getClassLoader();
                HookTools.hookFunc(sClassLoader);
        }
    });
}

```

*    函数中传的接口类（interface）
    --------------------
    

       这个很多网站上有案例，通过 java.lang.reflect.Proxy 能解决。很多人到了 hook 异步请求或者 rxjava 就头疼，因为很多参数或者返回值都是类似 A<T> 或者 interface，配合下面实例可以轻松解决。

```
public static Object Proxy(ClassLoader classLoader) {
    return Proxy.newProxyInstance(classLoader, new Class[]{XposedHelpers.findClass("bolts.Continuation", classLoader)}, new InvocationHandler() {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (method.getName().equals("then")) {
                Object task = args[0];
                Gson gson = new Gson();
                JSONObject object = JSON.parseObject(gson.toJson(XposedHelpers.callMethod(task,"getResult")));
                //获取值
                return null;
            }
            return null;
        }
    });
}
 
//调用方法
XposedHelpers.callMethod(respon,"continueWith",Proxy(classLoader));

```

*   **其他问题**
    --------
    
*   如果需要用 APP 内的 Context 的话直接搜索 public static Context 关键词能搜索到很多函数随便调用。
    
*   需要某个实例化类先找找 APP 内的 instance 函数或者获取实例的 static 函数，如果没有的话用 xposed 去 instance
    
*   callMetchod 传 enum 类型的参数
    

```
//
XposedHelpers.callMethod(classObject,"method", Enum.valueOf(enumClass, "One"))

```

*   处理异步请求结果时配合 Map 和 sleep  
    

```
//保存异步结果的全局map
public static Map keymap = new ConcurrentHashMap();
 
//异步框架的参数实现类代理
public static Object Proxy(ClassLoader classLoader,String key) {
    return Proxy.newProxyInstance(classLoader, new Class[]{XposedHelpers.findClass("bolts.Continuation", classLoader)}, new InvocationHandler() {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            //获取到值后放到map里
            keymap.put(key,object);
            return null;
        }
    });
}
 
/**
 * 省略一段代码
 */
//response等于bolts框架的Task.onSuccessTask或者Task.call的返回。
String key = String.valueOf(System.currentTimeMillis());
XposedHelpers.callMethod(response,"continueWith",Proxy(classLoader,key));
Thread.sleep(1000);
if(keymap.containsKey(key)){
    return JSON.parseObject(keymap.remove(key));
} 
```

上述的一些问题可能不常见或者网上资料少分散的，xposed 开发时遇到的问题其实很多，有些跟国产 ROM 相关，有些跟安卓版本相关，这些只能看命了。

[[公告]5 月 14 日腾讯安全零信任发展趋势论坛重磅开幕！邀您一起从 “零” 开始，共建信任！！](https://zta.insecworld.com/?utm_campaign=MJTG&utm_source=KX&utm_medium=WZLJ)

最后于 2020-11-5 10:51 被 AyonA333 编辑 ，原因：