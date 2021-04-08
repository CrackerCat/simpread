> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266787.htm)

上一篇介绍了基于 xposed 的 frida 持久化方案，这次带来基于 magisk 和 riru 的持久化方案。方案原理其实简单的很，开发过程却是非常坑，比如把手机刷成砖，比如每次出错都得重启手机加载插件。还好这些坑已经克服，现在只要编写靠谱的 js 脚本，自己过掉 app 的各种检测了！

原理
==

1.  riru 插件  
    我们知道 riru 给出了在 app 进程 fork 出来的时候的几个回调函数
    
    ```
    forkAndSpecializePre
    forkAndSpecializePost
    specializeAppProcessPre
    specializeAppProcessPost
    forkSystemServerPre
    forkSystemServerPost
    
    ```
    
    这里选择了在如下方法处调用 frida-gumjs 引擎库执行 js 脚本
    
    ```
    static void forkAndSpecializePost(JNIEnv *env, jclass clazz, jint res) {
     if (res == 0) {
         // in app process
         enable_hack = rirutest(env, *_appDataDir);
         if (enable_hack) {
             gumjsHook();
         }
     
     } else {
         // in zygote process, res is child pid
         // don't print log here, see https://github.com/RikkaApps/Riru/blob/77adfd6a4a6a81bfd20569c910bc4854f2f84f5e/riru-core/jni/main/jni_native_method.cpp#L55-L66
     }
    }
    
    ```
    
2.  frida-gumjs.a 是了 frida 封装的 hook 库和 quickjs 引擎，上一篇里也提到过，官方有下载，我们这里不细说。
    
3.  配置  
    同样使用配置文件来管理哪些 package 需要 hook，配置文件固定放在
    
    ```
    const char *filepath = "/data/local/tmp/myscript.js";
    
    ```
    
    要 hook 的包名每个一行
    
    ```
    org.xtgo.xcube.base2
    org.xtgo.xcube.base
    
    ```
    
    咱这个插件使用的 frida 脚本路径也是写死的
    
    ```
    const char *confpath = "/data/local/tmp/pkg.conf";
    
    ```
    

使用
==

1. 安装 magisk 和 magisk manager，安装 riru-core 核心组件  
2.frida 官网下载 frida-gumjs.a 放到 / module/src/main/cpp / 对应的 ABI 下  
3. 执行 build，out 目录会生成 zip 插件包  
4.magisk manager 安装插件，重启手机  
5./data/local/tmp/pkg.conf 中添加要 hook 的 packagename  
6.frida 脚本 push 到 / data/local/tmp/myscript.js  
7.logcat 中可以看到 frida 脚本中输出的日志

源码
==

https://github.com/svengong/xcubebase_riru.git

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)