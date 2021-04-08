> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-266785.htm)

app 的启动在很早期的阶段会经由 zygote 进程 fork 出子进程, 该过程在 android 源码中搜索`nativeForkAndSpecialize`可以找到.

 

通过 lief 之类的工具感染 android 系统库令 zygote 加载我们的 so, 并在 so 中用`pthread_atfork`捕获 fork 行为, 之后可以进行 app 包名的判断, 然后干点啥 (比如加载 frida-gadget).

 

在这个过程中有小踩了几个坑, 分别是:

 

较高版本的 android 的链接器命名空间机制导致不能将 so 随便放 (比如`/data/local/tmp/`), 就算指定绝对路进行加载, 在开启了 selinux 的情况下仍然会报错找不到库从而加载失败. 解决方式可以看看 https://github.com/hack0z/byOpen 或者关闭 selinux, 或者老老实实把库放在`/system/lib(64)/`目录下面.

 

在 fork 之后不能马上进行用户或 app 的判断 (此时还是 root 用户) 和线程的创建(加载 frida-gadget 会创建线程), 否则会发生 selinux_context 的切换失败导致 app 无法启动. 解决方式是通过 setitimer 和 signal 延迟一段时间等它切换完了再执行. 在自机上测的是使用 ITIMER_PROF 的话等待 60 毫秒就可以了, 可以再长些因为之后还要等它将 cmdline 切换至 app 的进程名.

 

frida-gadget 配置使用 v8 环境时在调用 dlopen 进行初始化时需要一个足够大的栈空间, 否则 v8 会初始化失败, 错误原因是栈空间不足, 可以通过手动指定一个足够大的栈空间解决.

 

源码见 https://github.com/tacesrever/easy-frida/tree/master/gadget

 

完整食用方式

1.  编辑源码中`gadget-loader.h`的 LIB_FRIDA(frida 库名, 可以修改, 此后以 frida-gadget.so 为例) 和 FRIDA_SCRIPT_DIR(脚本在手机上的位置), 注意如果将 FRIDA_SCRIPT_DIR 放在 sdcard 目录下需要为 app 分配存储卡使用权限.
2.  编译 `gadget-loader.cpp` 可以使用自己常用的 IDE 新建一个 android C++ 库项目导入`gadget-loader.h`, `gadget-loader.cpp` 和 `nlohmann/json.hpp`
    
3.  在`/proc/zygote's pid/maps`中找一个看着顺眼的库, 以 libart.so 为例
    
4.  使用`adb pull`将该库的 32 位与 64 位版本都 pull 下来, 先做个备份
    
5.  使用 lief 分别为它们添加一个依赖库, 库名称可以随便起, 看自己喜好
    
    ```
    import lief
    bin = lief.parse("libart.so") 
    bin.add_library("gadget-loader.so") # 你起的库名称 
    bin.write("libart.so") # 之前要做好备份
    
    ```
    
6.  将编译完后的`gadget-loader`库名改为你起的名字, 和注入后的 libart.so, 下载下来改名后的 frida-gadget.so 一起用 adb push 放进 / system/lib(64)/ 下面, 注意 32 位和 64 位的分别放一起, 别放错了
    
7.  参考 [gadget 文档](https://frida.re/docs/gadget/#scriptdirectory), 在 / system/lib(64)/ 下分别新建与 frida-gadget.so 同名的扩展名为. config 的文件, 内容为
    
    ```
    {
      "interaction": {
        "type": "script-directory",
        "path": "和FRIDA_SCRIPT_DIR相同"
      }
    }
    
    ```
    
8.  运行`adb shell pkill -f zygote`, 等待手机从黑屏重新恢复, 这一步不会断开 adb 连接, 如果万一这一步手机黑屏醒不过来就要用到备份的 libart.so 恢复, 错误原因可以使用 logcat 自查.
    
9.  等手机重新亮起来后运行`adb shell ps -ef | grep zygote`得到 zygote 的 pid, 再运行`adb shell cat /proc/zygote的pid/maps | grep gadget-loader.so(你起的库名)` 看到有输出就成功了.
    
10.  之后的使用方法仍然参考 [gadget 文档](https://frida.re/docs/gadget/#scriptdirectory), 在手机上脚本目录下新建一个 name.config 文件, 内容为
    
    ```
    {"filter": {"executables": ["目标app包名", "目标app其它运行时进程名"]}}
    
    ```
    
    之后将 frida 脚本放在脚本目录 / name.js 就可. name 自己取, 建议为目标 app 包名, 便于管理.

[[公告] 春风十里不如你，看雪团队诚邀你的加入！](https://mp.weixin.qq.com/s/bJEtd2Fu_MwEjUdkT4H5bQ)

最后于 6 天前 被 tacesrever 编辑 ，原因： 添加最后一个步骤