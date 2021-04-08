> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266784.htm)

序
-

去年下半年由于业务的需求，希望能够将 frida 的 js 脚本持久化到手机，只要 APP 启动就可以自行运行指定的脚本；一来希望可以脱离 pc，二来可以为 xposed 增加没有的 native hook 能力，三就是部署以后方便通过命令行来更新脚本；

 

我想到了两个方案

*   方案一. 将 frida-gumjs 编译进一个 riru 插件，通过面具刷入手机，这样每个进程都具备加载 frida js 的能力；
*   方案二. 使用 xposed，在 app 启动时将 frida-gumjs 载入 app 进程内；

好消息是两个方案目前都已经实现。方案一由于是去年完成的，使用的 riru23 版本的模板，现在的 riru 版本号已经 25 了，不知道是否能够兼容；再有就是开发比较早，几位同学觉得上手不容易，所以早早结束了开发，虽然功能完整但未达预期。因此今天先介绍方案二并开源。

 

我们大的产品叫做 Xcube，而这个模块就叫做 xcubebase；

原理实现
----

### 编译 frida-gumjs 为 so

frida-gumjs 是 frida 武器库的 js 封装；官方提供了 libfrida-gumjs.a 的下载，可惜的是文档资料极少，代码看到吐血也只是摸清了一些关键函数的使用；官方提供了这个库的头文件，我们平时 hook 用的 js 脚本都可以用这个库来运行；  
frida-gumjs.cpp 中的代码如下：

```
gumjsHook(const char *scriptpath) {
    ...
    ...
    script = gum_script_backend_create_sync(backend, "example", js, cancellable, &error);
    g_assert (error == NULL);
    gum_script_set_message_handler(script, on_message, NULL, NULL);
    gum_script_load_sync(script, cancellable);
    //执行脚本
    context = g_main_context_get_thread_default();
    while (g_main_context_pending(context))
        g_main_context_iteration(context, FALSE);
...
}
gum_script_backend_create_sync中的js参数就是我们的hook脚本

```

再添加一个 jni 入口函数

```
extern "C"
JNIEXPORT void JNICALL Java_org_xtgo_xcube_base_XcubeBase_gumjsHook(
        JNIEnv *env, jclass clazz, jstring script) {
    const char *scriptpath = env->GetStringUTFChars(script, 0);
    gumjsHook(scriptpath);
    env->ReleaseStringUTFChars(script, scriptpath);
}

```

将以上 libfrida-gumjs.a 和相关代码使用 ndk 编译为我们 app 可以加载的 so

```
# 编译gumjs
add_library(gumjs STATIC IMPORTED)
set_target_properties(gumjs
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libfrida-gumjs.a)
 
 
# 添加到apk
add_library(xcubebase SHARED frida-gumjs.cpp NativeEntry.cpp)
target_link_libraries(xcubebase gumjs ${log-lib})

```

这样就得到了 libxcubebase.so

### 使用 xposed 让 app 加载 libxcubebase.so

为了方便使用，我把 libxcubebase.so 集成到了 xposed 的插件中，插件上有初始化功能，可以一键将 libxcubebase.so、配置文件 xcube.yaml 导入到 / data/local/tmp。  
实现比较麻烦基本就是先从 assets 拷贝到 / data/app/packagename/cache 目录下，再复制到 / data/local/tmp/。复制到 / data/local/tmp 这一步需要 su 权限，所以如果有提示 su 权限需要允许；  
下面的代码是 app 加载 so、执行 js 的核心代码

```
private static String configPath = "/data/local/tmp/xcube/xcube.yaml";
public static boolean hooked = false;
 
@Override
public void handleLoadPackage(XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable {
    String remoteName = Utils.getRemoteName();
    XcubeConfig config = new XcubeConfig(configPath);
    if (!config.active || !config.contains(loadPackageParam.packageName)) {
        return;
    }
    String script = config.getScriptPath(loadPackageParam.packageName);
    Log.e(TAG, "current app packageName : " + loadPackageParam.packageName + ":" + remoteName);
 
    XposedHelpers.findAndHookMethod(Application.class, "attach", Context.class, new XC_MethodHook() {
        @Override
        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
            Log.d(TAG, "attach beforeHookedMethod script:" + script);
            if (hooked || !remoteName.isEmpty()) {
                //只hook主进程一次
                return;
            }
            try {
                Context base = (Context) param.args[0];
                File toPath = base.getDir("libs", Context.MODE_PRIVATE);
                String libpath = "/data/local/tmp/xcube/";
                String ABI = android.os.Process.is64Bit() ? "arm64-v8a" : "armeabi-v7a";
                //System.loadlibrary使用的classloader是当前classloader，而param.args[0]是目标应用的classloader，二者不同
                Log.d(TAG, "attach beforeHookedMethod toPath:" + toPath);
                LoadLibraryUtil.loadSoFile(this.getClass().getClassLoader(), libpath + ABI, toPath);
                System.loadLibrary("xcubebase");
                Log.d(TAG, "script path :" + script);
                gumjsHook(script);
                hooked = true;
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
 
        }
 
 
    });
    XposedHelpers.findAndHookMethod(Activity.class, "onCreate", Bundle.class, new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) throws Throwable {
            ((Activity) param.thisObject).getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
        }
    });
}

```

### 配置

```
cmi:/data/local/tmp/xcube # ls -l
total 10
drwxrwxrwx 2 root root 3488 2021-03-30 18:28 arm64-v8a
drwxrwxrwx 2 root root 3488 2021-03-30 18:28 armeabi-v7a
-rwxrwxrwx 1 root root  348 2021-03-30 18:28 xcube.yaml
cmi:/data/local/tmp/xcube # ls armeabi-v7a/ -l
total 51232
-rwxrwxrwx 1 root root   144692 2021-03-30 18:28 libhellofrida.so
-rwxrwxrwx 1 root root 52161248 2021-03-30 18:28 libxcubebase.so
-rwxrwxrwx 1 root root    95564 2021-03-30 18:28 shellcmd
cmi:/data/local/tmp/xcube # cat xcube.yaml
# 是否开启xposed加载frida js hook 引擎功能
active: true
 
# 要hook的app包名和要使用的js脚本
packageConfigList:
 com.tencent.mtt: /data/local/tmp/mtt.js
 org.xtgo.xcube.forxcubetest: /data/local/tmp/myscript.js
 com.tencent.zgqyz: /data/local/tmp/myscript.js
 org.xtgo.xcube.verification: /data/local/tmp/myscript.js

```

xcube.yaml 是配置文件，规定了我们要 hook 的 app 及其要运行的 js 脚本，需要自行修改

### 使用

1.  安装 Xcubebase.apk，启动该程序，点击初始化按钮，需要赋予 su 权限
2.  在 xposed 中启用该插件，重启应用
3.  修改 xcube.yaml，指定你想 hook 的 apk 和脚本
4.  启动目标 app 即可在 logcat 中看到 js 脚本的输出内容

### feature

1.  frida hook java 时稳定性不好，尤其是锁屏再回来。这个问题是 frida 本身 java hook 方案的问题，暂时无解。因此推荐 javahook 用 xposed ，nativehook 用 frida 脚本
2.  目前这个框架中，frida 脚本目前还不能动态更新，只能重启 app 脚本才能生

源码今天会放出来。稍等一个午休  
代码出来了，工程里 libfrida-gumjs.a 太大传不上去，我给压缩了，自己解压一下  
https://github.com/svengong/xcubebase.git

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)

最后于 6 天前 被 svengong 编辑 ，原因： 修改标题