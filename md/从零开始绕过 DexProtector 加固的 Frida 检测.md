> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2074484-1-1.html)

> [md]# 从零开始绕过 DexProtector 加固的 Frida 检测 ## 一个可复盘、可扩展、可工程化的对抗实录（本文由 id：小佳、fyrlove、roysue 共同完成）为保证结论稳 .........

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)fyr666 _ 本帖最后由 fyr666 于 2025-11-21 22:54 编辑_  

从零开始绕过 DexProtector 加固的 Frida 检测
--------------------------------

### 一个可复盘、可扩展、可工程化的对抗实录

（本文由 id：小佳、fyrlove、roysue 共同完成）

为保证结论稳健且可迁移，文中所有实验均在以下环境中完成（LineageOS 21 / Nexus 5X，Magisk 29.0.0，LSPosed，Zygisk Frida Gadget 开源模块（[https://github.com/sucsand/sucsand](https://github.com/sucsand/sucsand)） ，Frida Server 16.5.2）

合规：本文仅用于安全研究与对抗评估，旨在帮助甲方团队识别自身加固薄弱点、完善自检与回归策略；不针对具体业务落地攻击，不提供可直接用于对第三方应用的利用脚本。

你在这篇文章里会学到这些硬技能（都能复现）：

1. 入口卡位：不死盯 System.loadLibrary，而是改从 __loader_android_dlopen_ext 抓 “真实装载面”，提早拿到证据和时序。

2. 匿名段定位与转储：用 JNI_OnLoad → 函数指针 → 匿名可执行段 这条线，结合 /proc/<pid>/maps 找段，Frida 直接 dump，只修 text 段也能在 IDA 里反汇到可用程度。

3. 最小化修复与类型库引入：在 IDA 里手动补区段、引 android_arm64 / gnulnx_arm64 类型库，把 JNIEnv / 动态注册链条（RegisterNatives/FindClass/...）梳顺。

4. 校验链路拆解：识别 xxHash / SHA256 / HMAC 的落点（含内联 SHA 指令），用 “等式化替换 + 调用点定位（靠 LR 定位调用者）” 做最小侵入的绕过。

5. 二分法定位：从 “可卸载点” 开始逐段排除，把 “必崩区间” 缩到少量函数，再精确打补丁。

6. 入口完整性绕过：遇到对 “当前段基址” 的校验，复制一份干净 text，参数基址替换为干净副本过检。

7. 线程面处理：顺着 /proc/self/maps 的反向引用追到 pthread_create，定位监控线程入口。

工程化习惯：每一步都留 “能回头验证” 的观测点，避免“一刀切”，降低误伤和回归压力。

分析过程
----

### 01. 样本获取与安装

安装相同的版本，确保 app 可安装、可启动，并且能够完整复线整个过程。

#### 1.1 样本获取

样本：Hyatt6.8.0.apkm  
下载地址：下载地址：[https://www.apkmirror.com/apk/hyatt-corporation/world-of-hyatt/world-of-hyatt-6-8-0-release/world-of-hyatt-6-8-0-android-apk-download/#google_vignette](https://www.apkmirror.com/apk/hyatt-corporation/world-of-hyatt/world-of-hyatt-6-8-0-release/world-of-hyatt-6-8-0-android-apk-download/#google_vignette)

#### 1.2 样本安装

安装方法：通过 APKMirror Installer / MT 管理器均可。

### 02. 设备与运行环境

该基线用于复现与对比。不同 SoC / API level / ART 实现可能造成 “加载顺序、符号可见性、maps 标记” 差异。

#### 2.1 设备环境

设备：Nexus 5X（LineageOS 21）。

Root / 框架：Magisk 29.0.0、LSPosed、Zygisk Frida Gadget 模块（[https://github.com/sucsand/sucsand](https://github.com/sucsand/sucsand)）

Frida：frida-server 16.5.2。

PC 环境: 肉丝 (r0ysue) 大佬提供的 r0env kali 虚拟机（安装了逆向需要的工具和环境配置，能省掉很多安装和环境的问题）。

#### 2.2 运行结果

不运行 frida-server 时可进入主页但提示要升级；一旦 attach/spawn，进程迅速崩溃。

APP 点开也不会崩溃，正常进入到主页，只是强制提示必须升级到最新版才可用。说明壳可能没有 root 检测，或者没有检测到 Magisk、LSPosed。  
![](https://attach.52pojie.cn/forum/202511/20/220545mq0837baz065qaak.png)

**无 frida-server - 正常启动. png** _(87.62 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0MHw1Y2E5NjNlYXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传

frida 不管是 attach 模式还是 spawn 模式，均会迅速的进程崩溃退出。  
![](https://attach.52pojie.cn/forum/202511/20/220540a3okq0jss3sqz0zf.png)

**使用了小工具 - 未过掉 frida 检测. png** _(90.86 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTgzOHw4YTA1OThjM3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传

### 03.Java 层入口识别（Application & System.loadLibrary）

使用 jadx 对 apk 进行分析。  
通过 AndroidManifest 与 Application 类，定位主壳入口与 native 装载点，判断逻辑是否下沉到 so。

#### 3.1 AndroidManifest.xml 文件简介

Manifest 是干嘛的？

1. 给系统登记应用身份证与四大组件（Activity/Service/BroadcastReceiver/ContentProvider）。

2. 声明权限、最低 / 目标系统版本、硬件能力等。

3. 配置应用级开关（如是否可调试、是否允许备份、网络安全配置等）。

其中还包含了应用的包名。这只是简述了一个大概，想了解完整的朋友请自行搜索。

#### 3.2 样本 apk 的 AndroidManifest.xml 分析

六十几兆的 apkm 文件后缀名改成 zip 后解压，里面有个一百多兆的 base.apk，其实大部分内容都在这个 base.apk 里。jadx-gui 打开瞅瞅。  
1. 搜`application`, 找到 application 节点，这个节点中的 name 就是主壳入口点，这里是`ProtectedTopHyattApplication`类。

```
<application
        android:theme="@style/AppThemeV4.HorizontalAnimation"
        android:label="@string/app_name_value"
        android:icon="@mipmap/ic_launcher"
        android:/>

```

2. 查找启动 Activity，安卓中，启动入口的 Activity 会有一个 intent-filter 标签

```
<intent-filter>
      <action android:/>
      <category android:/>
</intent-filter>

```

搜索一下可以发现，入口 Activity 是 SplashActivity:

```
<activity android:theme="@style/SplashTheme" android:label="@string/app_name_value" android: >
            <intent-filter>
                <action android:/>
                <category android:/>
            </intent-filter>
</activity>

```

3. 查看 ProtectedTopHyattApplication 类，可以看到有一些加载`so`库的操作，一般都清楚，主要逻辑一般肯定在`so`库里。

```
protected void attachBaseContext(Context context) {
        super.attachBaseContext(context);
        try {
            s.a((Context) this);
            System.loadLibrary("dpboot");
            wxyrq();
        } catch (Throwable th) {
            b.fooldg(this, th);
        }
    }
public static void a(Context context, String str) {
            System.loadLibrary(c);
            boolean z = !r.a();
            b = z;
            if (z) {
                a = new File(context.getFilesDir().getAbsolutePath());
                r.a(context);
                r.a(context.getFilesDir().getAbsolutePath());
            }
        }

```

还有很多`native`函数更加验证了其逻辑在`java`层几乎没有。

```
public static native InputStream BC(Object obj, String str);
public static native boolean JaucCymn(String str, int i, List list);
private static native byte[] iIbBs();
public static native String s(String str);
private static native void ttghdCr(Object obj);
private static native void wxyrq();

```

基本可以确定，Java 层只做了引导，核心检测链条在 native 中。

### 04. 壳的 native 层入口点

那就想先 hook 这个`System.loadLibrary`函数，但很明显一 hook 就会崩。  
那就是要找更早的时机，hook 这个`System.loadLibrary`最底层的函数，那最底层的函数是哪个呢？  
是 linker 里的`__loader_android_dlopen_ext`函数。（参考文章：[SystemLoadLibrary :: 郑欢的学习总结](https://huanle19891345.github.io/en/android/art/jni/systemloadlibrary/)  
）  
这个函数是全局的，直接取符号就可以。（以下都是 frida16.5.2，切记先别上 frida17 噢）

#### 4.1 启动 frida-server，并进行端口转发

启动 frida-server 并指定 14725，不使用默认端口 (27042)，很多检测会检测这个默认端口，所以一开始就处理一下。

```
adb shell                                                                                                                            
bullhead:/ $ su
bullhead:/ # cd /data/local/tmp                                  
bullhead:/data/local/tmp # ./frida-server  -l 0.0.0.0:14725 

```

进行端口转发：

```
adb forward tcp:14725 tcp:14725

```

#### 4.2 hook dlopen，定位 so 加载时机

执行脚本：frida -H 127.0.0.1:14725 -f com.Hyatt.hyt -l hook_dexprotect.js。  
根据结果分析：hook 日志显示先后加载：libalice.so、libdpboot.so、libdexprotector.so；都输出了 "结束"，随后崩溃。  
崩溃时机在 “加载完成之后”（非 .init_array 内）。  
所以入口库可以基本确定为 libdexprotector.so；  
检测不在 .init_array，后续应聚焦 JNI_OnLoad 或动态注册链。

```
function hook_dlopen() {
    var android_dlopen_ext = Module.findExportByName(null, "__loader_android_dlopen_ext")
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            var pathptr = args[0];
            console.log("path is => ", pathptr.readCString())
        },
        onLeave: function () {
            console.log("结束")
        }
    })
}

hook_dlopen()

```

运行如下：

```
> frida -H 127.0.0.1:14725 -f com.Hyatt.hyt -l hook_dexprotect.js                       
     ____
    / _  |   Frida 16.5.2 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to 127.0.0.1:14725 (id=socket@127.0.0.1:14725)
Spawned `com.Hyatt.hyt`. Resuming main thread!                          
[Remote::com.Hyatt.hyt ]-> path is =>  libframework-connectivity-tiramisu-jni.so
结束
path is =>  /system/framework/oat/arm64/org.apache.http.legacy.odex
结束
path is =>  /data/app/~~IvxV7LJ9cvtqbZUadvpHsQ==/com.Hyatt.hyt-q8YukchM306gv-ilc-P6Ig==/oat/arm64/base.odex
结束
path is =>  /data/app/~~IvxV7LJ9cvtqbZUadvpHsQ==/com.Hyatt.hyt-q8YukchM306gv-ilc-P6Ig==/split_config.arm64_v8a.apk!/lib/arm64-v8a/libalice.so
结束
path is =>  /data/app/~~IvxV7LJ9cvtqbZUadvpHsQ==/com.Hyatt.hyt-q8YukchM306gv-ilc-P6Ig==/split_config.arm64_v8a.apk!/lib/arm64-v8a/libdpboot.so
结束
path is =>  /data/app/~~IvxV7LJ9cvtqbZUadvpHsQ==/com.Hyatt.hyt-q8YukchM306gv-ilc-P6Ig==/split_config.arm64_v8a.apk!/lib/arm64-v8a/libdexprotector.so
结束
Process crashed: java.lang.RuntimeException: DP: 786 01120321050d0580a011078020058060078080010580e003078020058080040780200580a0100780400580c0030780200580400115032105090580a0560780200580a06e078020058020078020058080a8020780400580a0010116032105050580e0320780600580200780200580c029                                                                                   

***
FATAL EXCEPTION: main
Process: com.Hyatt.hyt, PID: 8020
java.lang.RuntimeException: Unable to create application com.Hyatt.hyt.ProtectedTopHyattApplication: com.Hyatt.hyt.MessageGuardException_RFA6IDc4NiAwMTEyMDMyMTA1MGQwNTgwYTAxMTA3ODAyMDA1ODA2MDA3ODA4MDAxMDU4MGUwMDMwNzgwMjAwNTgwODAwNDA3ODAyMDA1ODBhMDEwMDc4MDQwMDU4MGMwMDMwNzgwMjAwNTgwNDAwMTE1MDMyMTA1MDkwNTgwYTA1NjA3ODAyMDA1ODBhMDZlMDc4MDIwMDU4MDIwMDc4MDIwMDU4MDgwYTgwMjA3ODA0MDA1ODBhMDAxMDExNjAzMjEwNTA1MDU4MGUwMzIwNzgwNjAwNTgwMjAwNzgwMjAwNTgwYzAyOSBbMjAyNTA0MjEtMjAyNTA1MjIxOTAzIGI3OmI3IDM0IGdvb2dsZS9idWxsaGVhZC9idWxsaGVhZDo4LjEuMC9PUE0zLjE3MTAxOS4wMTQvNDUwMzk5ODp1c2VyL3JlbGVhc2Uta2V5cyBibG9ja2VkXSAwMTk5ZjdjOS1hYjI5LTQ0ZGUtYmMzYi04OWI5ODk5N2RiNTg: DP: 786 01120321050d0580a011078020058060078080010580e003078020058080040780200580a0100780400580c0030780200580400115032105090580a0560780200580a06e078020058020078020058080a8020780400580a0010116032105050580e0320780600580200780200580c029
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:7403)
        at android.app.ActivityThread.-$$Nest$mhandleBindApplication(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2379)
        at android.os.Handler.dispatchMessage(Handler.java:107)
        at android.os.Looper.loopOnce(Looper.java:232)
        at android.os.Looper.loop(Looper.java:317)
        at android.app.ActivityThread.main(ActivityThread.java:8592)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:580)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:878)
Caused by: com.Hyatt.hyt.MessageGuardException_RFA6IDc4NiAwMTEyMDMyMTA1MGQwNTgwYTAxMTA3ODAyMDA1ODA2MDA3ODA4MDAxMDU4MGUwMDMwNzgwMjAwNTgwODAwNDA3ODAyMDA1ODBhMDEwMDc4MDQwMDU4MGMwMDMwNzgwMjAwNTgwNDAwMTE1MDMyMTA1MDkwNTgwYTA1NjA3ODAyMDA1ODBhMDZlMDc4MDIwMDU4MDIwMDc4MDIwMDU4MDgwYTgwMjA3ODA0MDA1ODBhMDAxMDExNjAzMjEwNTA1MDU4MGUwMzIwNzgwNjAwNTgwMjAwNzgwMjAwNTgwYzAyOSBbMjAyNTA0MjEtMjAyNTA1MjIxOTAzIGI3OmI3IDM0IGdvb2dsZS9idWxsaGVhZC9idWxsaGVhZDo4LjEuMC9PUE0zLjE3MTAxOS4wMTQvNDUwMzk5ODp1c2VyL3JlbGVhc2Uta2V5cyBibG9ja2VkXSAwMTk5ZjdjOS1hYjI5LTQ0ZGUtYmMzYi04OWI5ODk5N2RiNTg: DP: 786 01120321050d0580a011078020058060078080010580e003078020058080040780200580a0100780400580c0030780200580400115032105090580a0560780200580a06e078020058020078020058080a8020780400580a0010116032105050580e0320780600580200780200580c029
        at com.Hyatt.hyt.ProtectedTopHyattApplication$b.qC(Unknown Source:9)
        at com.Hyatt.hyt.ProtectedTopHyattApplication$b.xDzqsetu(Unknown Source:0)
        at com.Hyatt.hyt.ProtectedTopHyattApplication$b.EHo(Unknown Source:6)
        at com.Hyatt.hyt.ProtectedTopHyattApplication$b.fooldg(Unknown Source:1)
        at com.Hyatt.hyt.ProtectedTopHyattApplication.onCreate(Unknown Source:49)
        at android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1386)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:7398)
        ... 9 more
Caused by: java.lang.RuntimeException: DP: 786 01120321050d0580a011078020058060078080010580e003078020058080040780200580a0100780400580c0030780200580400115032105090580a0560780200580a06e078020058020078020058080a8020780400580a0010116032105050580e0320780600580200780200580c029
        at com.Hyatt.hyt.ProtectedTopHyattApplication.ttghdCr(Native Method)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.Hyatt.hyt.ProtectedTopHyattApplication$k.zeHo(Unknown Source:13)
        at com.Hyatt.hyt.ProtectedTopHyattApplication$k.wolzmlnlx(Unknown Source:472)
        at com.Hyatt.hyt.ProtectedTopHyattApplication.ttghdCr(Native Method)
        at com.Hyatt.hyt.ProtectedTopHyattApplication.onCreate(Unknown Source:44)
        ... 11 more
***
[Remote::com.Hyatt.hyt ]->

Thank you for using Frida!

```

#### 4.3 hook JNI_OnLoad 函数

通过再去 hook 它的 JNI_OnLoad 函数。  
发现它的 JNI_OnLoad 函数也很顺利执行了，但是还是死掉了。  
我们在 onLeave 这里加上休眠，它是不会死的，，它可能是在 JNI_OnLoad 注册了 JNI 函数，然后外部调用的

```
 var libdexprotector = Process.findModuleByName("libdexprotector.so")
            Interceptor.attach(libdexprotector.findExportByName("JNI_OnLoad"), {
                onEnter: function (args) {                    
                     console.log("JNI_OnLoad onEnter")
                },onLeave:function(ret){
                    console.log("JNI_OnLoad  结束")

                    // Thread.sleep(60)
                }
            })

```

结果：  
![](https://attach.52pojie.cn/forum/202511/20/222101ossaoxgcxoxvs9go.png)

**img_24.png** _(94.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg3MHw2MTg1YTA5ZnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:21 上传

### 05. 定位检测函数位置，追到 MMAP 匿名内存段里

检测函数应该不在 init_array 里面，那还有可能在 JNI_OnLoad 里面，或者其他 native 函数里面，刚刚壳的 Java 层看到了很多 native 函数，可能是 Java 层调过来检测的也未必。

#### 5.1 使用 IDA 分析 libdexprotector.so

先把`libdexprotector.so`拖到 IDA 里分析看下，有日志可以，这个 so 在 split_config.arm64_v8a.apk 里面，前面安装包里解压出来有，再给它解压就得到`libdexprotector.so`。

看下`JNI_OnLoad`函数，发现啥也没干，常见的动态注册也没有，也不是没有，它是把 vm 传给`off_C838`这个指针函数了，让它去做事情。

```
jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
  int v3; // w0

  if ( dword_C830 )
    return -dword_C830;
  v3 = off_C838(vm, 0LL);
  off_C838 = 0LL;
  if ( v3 )
    return -v3;
  else
    return 65540;
}

```

frida 读下这个 off_C838 指针函数的值：

```
var loader_android_dlopen_ext = Module.findExportByName(null,"__loader_android_dlopen_ext")
console.log("dlopen address is => ", loader_android_dlopen_ext)
Interceptor.attach(loader_android_dlopen_ext, {
    onEnter: function (args) {
        var pathptr = args[0]
        console.log("path is => ", pathptr.readCString())
        if (pathptr.readCString().indexOf("libdexprotector.so") >= 0) {
            this.match = true
        }
    }, onLeave: function (ret) {    
        console.log("结束")    
        if (this.match) {
            var libdexprotector = Process.findModuleByName("libdexprotector.so")
            Interceptor.attach(libdexprotector.findExportByName("JNI_OnLoad"), {
                onEnter: function (args) {                    
                    console.log("off_C838 is => ", libdexprotector.base.add(0xC838).readPointer())
                },onLeave:function(ret){}
            })
        }        
    }
})

```

跑一下结果是：

```
off_C838 is =>  0x7e25b9a984

```

再看它属于哪个 so：

```
console.log("off_C838 module is => ", Process.findModuleByAddress(libdexprotector.base.add(0xC838).readPointer()))

```

跑出来结果是 null，地址不属于任何已知 module：

```
off_C838 module is =>  null

```

那么再看这个地址属于哪段内存（findRangeByAddress 显示该地址落在的匿名可执行段），为了防止进程崩溃看不到内存，加上线程休眠 60 秒：

```
console.log("off_C838 mem is => ",JSON.stringify(Process.findRangeByAddress(libdexprotector.base.add(0xC838).readPointer())))
Thread.sleep(60)

```

跑出来内存段是：

```
off_C838 mem is =>  {"base":"0x7e25afc000","size":507904,"protection":"r-x"}

```

输出 cat maps:

```
console.log(`cat /proc/${Process.id}/maps | grep ${Process.findRangeByAddress(libdexprotector.base.add(0xc838).readPointer()).base}`)

```

结果是：

```
cat /proc/8467/maps | grep 0x7e24570000

```

把上面的结果拿去手机 APP 进程的 maps 里看下（注意去掉`0x7e24570000`这里的 0x）：

```
bullhead:/ # cat /proc/8467/maps | grep 7e24570000                                                             
7e24570000-7e245ec000 r-xp 00000000 00:00 0                              [anon:15f1e]

```

GPT 解释说，[anon:15f1e] 表示这是一段可执行的可读、私有匿名内存映射，内核为其分配的内部标识符（伪 inode）是 0x15f1e。

执行的函数位于这段匿名内存里，那得把它 dump 下来分析看看。

#### 5.2 IDA 分析结论

分析到这里，可以知道，DexProtector 将关键逻辑放入运行时生成的匿名映射段；分析需转向内存态 dump 与单段反汇。

### 06. 匿名可执行段定位与 /proc/<pid>/maps 交叉验证

通过 Frida 拿到 off_C838 所在内存区 base/size/prot，再交叉 cat /proc/<pid>/maps 验证段属性与范围。  
Frida 转储 dump 匿名内存：  
使用 frida 直接输出一下进程号 pid：

```
console.log("off_C838 pid is => ",Process.id)

```

这样就直接有了内存段的起始地址，长度，进程号，再使用线程休眠，把 APP 卡主使其不闪退：

```
off_C838 mem is =>  {"base":"0x7e25afc000","size":507904,"protection":"r-x"}
off_C838 pid is =>  8601

```

如果手速较慢，可以把休眠时间设置为更长。配合以下 dump 脚本，把匿名内存脱下来：

```
import frida
js_script = """
function dump_anon() {
    console.log("开始dump")
    const base = ptr(0x7e25afc000);
    const module_size = 507904;
    Memory.protect(base, module_size, 'rwx');
    const soMemory = Memory.readByteArray(base, module_size);
    send({name: "libanon.so", base: base, size: module_size}, soMemory);
}
dump_anon()
"""
def on_message(message, data):
    if message['type'] == 'send':
        payload = message['payload']
        so_name = payload['name']
        base_address = payload['base']
        size = payload['size']
        print(f"Dumping {so_name} (Base: {base_address}, Size: {size})")
        # 保存dump的.so文件
        with open(so_name, "wb") as f:
            f.write(data)
        print(f"{so_name} dumped successfully!")
    else:
        print(f"Error: {message}")
def main():
    # 附加到目标进程
    device = frida.get_usb_device()
    session = device.attach(8601)
    # 加载Frida脚本
    script = session.create_script(js_script)
    # 设置消息处理函数
    script.on("message", on_message)
    # 加载并执行脚本
    script.load()
if __name__ == "__main__":
    main()

```

执行结果如下：

```
>dumpso.py
开始dump
Dumping libanon.so (Base: 0x7e25afc000, Size: 507904)
libanon.so dumped successfully!

```

脱下来的 libanon.so 位于当前执行脚本的目录下。

### 07.IDA 手修匿名内存 SO

拖到 010 editor 里去可以看到前面一大片都是 0，也就是没有 ELF 的文件头，果然是匿名内存段的风格。大概率也 sofix 没法修。  
拖到 IDA 里，IDA 也无法判断这是什么汇编格式，手动选择一个处理器类型：ARM Little-endian ，点击确定。  
![](https://attach.52pojie.cn/forum/202511/20/220558efed99a91of2ed9m.png)

**img_3.png** _(167.6 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0NnxhMDNkMDBhYXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传

一路点击确定，按照 ARM 默认设置来分析这段匿名内存 so。此时 IDA 左边已经有了一堆`sub_`符号，说明 IDA 是可以正常分析里面的函数的。  
说明哪怕只有一个 text 段，IDA 也是可以正常反汇编的，哪怕没有导入导出表，没有文件头没有符号表，没有其他区段，也没有关系。  
接下来开始做基础修复，视图→打开子视图→类型库，导入基础库，右键，加载类型库，导入`android_arm64`库，和`gnulnx_arm64`，前者是安卓的，后者是 C++ 的。导进去才能识别`JNI`的东西，才能有 JNIEnv 和 jclass 这些，后续要改参数类型识别 JNI 里面的函数如 RegisterNatives、FindClass 那些。  
![](https://attach.52pojie.cn/forum/202511/20/220600dnynhpuiw72uhpu0.png)

**img_4.png** _(84.13 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0N3xhNWVlMzM1ZHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

### 08.IDA 手修匿名内存 SO(2), 匿名指针函数代码追踪

以 ARM64 Little-endian 打开；导入 android_arm64 / gnulnx_arm64 类型库以恢复 JNI/GLIBC 符号语义；

手工增加 rodata 等伪区段解决 “无效内存访问”。

#### 8.1 寻找真实执行的函数

到这里还没找到前面实际执行的`off_C838`函数在哪里，可以把函数指针地址减去基地址即可得到真实的偏移。

```
console.log("off_C838 real offset => ",libdexprotector.base.add(0xC838).readPointer().sub(Process.findRangeByAddress(libdexprotector.base.add(0xC838).readPointer()).base))

```

执行一下，也就是 sub_4e984。

```
off_C838 real offset =>  0x4e984

```

#### 8.2 无效内存访问修复

在 IDA 里按 g 跳到 0x4e984 的地址，就是 sub_4e984 的函数头，按 F5 汇编即可开始分析。  
首先有些报红字的，无效内存访问报错：  
![](https://attach.52pojie.cn/forum/202511/20/220602vxof8wozhz3m01r7.png)

**img_5.png** _(251.85 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0OHxhOTQ1ZjdmOHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

```
MEMORY[0x8A800] = v1;
MEMORY[0x8A808] = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v7 + 168LL))(v7, v6);

```

其实就是 dump 下来的 so 里没有数据段的原因，视图→打开子视图→区段，右键→添加区段，区段名称可以取 rodata，开始地址可以填 0x8A800，结束地址 0x8B800，给大一些。后面再遇到红字的，可以继续扩大一些，覆盖到红字指向地址的范围即可。  
再回到 sub_4e984 函数再按一下 F5，红字就消失了，变成了：  
![](https://attach.52pojie.cn/forum/202511/20/220604ayz5755a5d9v0iab.png)

**img_6.png** _(130.62 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0OXw1MTRhODViOHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

```
unk_8A800 = v1;
unk_8A808 = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v7 + 168LL))(v7, v6);

```

点在`unk_8A800`上按 x，就可以追踪交叉引用了，可以看到有四处引用到这个地址:  
![](https://attach.52pojie.cn/forum/202511/20/220607xck6qmm5ex1h6fvd.png)

**img_7.png** _(90.29 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1MHw3ZTU0Y2FjN3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

前面 JNIOnLoad 里面就一个 vm 参数传到 off_C838 了，那这里`sub_4e984`参数类型就是`JavaVM*`，按 y 修改。修改成功后，一些指针也会从这样：

```
   if ( (*(unsigned int (__fastcall **)(__int64, __int64 *, __int64))(*(_QWORD *)a1 + 48LL))(a1, &v7, 65540) )

```

变成这样：

```
if ( (*a1)->GetEnv(a1, (void **)&v8, 65540LL) )

```

完整的结果：  
![](https://attach.52pojie.cn/forum/202511/20/220609lyf2ic32t3cyfcij.png)

**img_8.png** _(234.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1MXw3NzcwY2ExMXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

识别出来了 env 里的指针，这也是前面导入库发挥了作用。  
接下来即可进行基本的逐行手撕了，以及不停地做一些尝试绕过检测的 hook 了。在逆向中也是这样不断地做尝试的。  
接下来理论上要进行全量分析了，逆向到最后，归根结底都是体力活罢了。

### 09.JNIEnv 恢复与动态注册链梳理

逐行手撕定位敏感函数

#### 9.1 JNIEnv 恢复

继续分析`sub_4e984`，a1 获取 env 传给 v7 了：

```
(*a1)->GetEnv(a1, (void **)&v7, 65540) 

```

那 v7 的参数类型就是 JNIEnv*，在定义那里`__int64 v8;`按 y 修改一下参数类型。原本的几个不明意义的指针：

```
if ( v3 )
  {
    (*(void (__fastcall **)(__int64))(*(_QWORD *)v7 + 136LL))(v7);
    return (unsigned int)(v3 + 1000);
  }
  else
  {
    v4 = sub_4EBE0(v7);
    if ( v4 )
    {
      v5 = v4;
      if ( !(*(unsigned __int8 (__fastcall **)(__int64))(*(_QWORD *)v7 + 1824LL))(v7) )
        sub_4EFC4(v7, v5);
    }
    if ( (*(unsigned __int8 (__fastcall **)(__int64))(*(_QWORD *)v7 + 1824LL))(v7) )
    {
      v6 = (*(__int64 (__fastcall **)(__int64))(*(_QWORD *)v7 + 120LL))(v7);
      (*(void (__fastcall **)(__int64))(*(_QWORD *)v7 + 136LL))(v7);
      unk_8A808 = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v7 + 168LL))(v7, v6);
    }
    return 0;
  }

```

修改成功后，又变得有意义了：

```
  if ( v3 )
  {
    (*v7)->ExceptionClear(v7);
    return (unsigned int)(v3 + 1000);
  }
  else
  {
    v4 = sub_4EBE0(v7);
    if ( v4 )
    {
      v5 = v4;
      if ( !(*v7)->ExceptionCheck(v7) )
        sub_4EFC4(v7, v5);
    }
    if ( (*v7)->ExceptionCheck(v7) )
    {
      v6 = (__int64)(*v7)->ExceptionOccurred(v7);
      (*v7)->ExceptionClear(v7);
      unk_8A808 = (*v7)->NewGlobalRef(v7, (jobject)v6);
    }
    return 0;
  }

```

#### 9.2 动态注册链路梳理，函数分析，发现重点可疑目标

从上往下看，接下来它把 v8 这个`jnienv`依次传给了`sub_4EAA0`、`sub_4EBE0`、`sub_4EFC4`这几个函数，这些函数都要跟进去逐行查看，每一行到底做了什么。  
先看第一个`sub_4EAA0`，唯一的参数 a1 修改类型为`JNIEnv*`，下面立刻动态注册的 API 出来了。

```
jint (*RegisterNatives)(JNIEnv *, jclass, const JNINativeMethod *, jint); 

```

动态注册的函数列表保存在 v6 参数中，

```
    v6[0] = v9;
    v6[1] = sub_40814(&unk_681F);
    v6[2] = sub_4F0F4;
    v6[3] = v8;
    v6[4] = sub_40814(&unk_346B);
    v6[5] = sub_4F254;
    v6[6] = v7;
    v4 = sub_40814(&unk_5523);
    RegisterNatives = (*a1)->RegisterNatives;
    v6[7] = v4;
    v6[8] = MEMORY[0x82988];
    if ( RegisterNatives(a1, (jclass)v3, (const JNINativeMethod *)v6, 3) )

```

其中`sub_4F0F4`开辟了一段字节数组，像是初始化函数，先不看。  
`sub_4F254`特别长，业务逻辑十分丰富，还夹杂着很多 Java 类方法的使用，大概五六百行，一眼看不出逻辑，需要细细拆分研究。这里是接下来分析的重点。

### 10. 关键函数 sub_4F254 的行为探测（返回值与入栈点）

直接 hook 这个 sub_4F254，观察一下结果，并首次尝试修改结果过检测：

```
Interceptor.attach(libanon.base.add(0x4F254),{
     onEnter: function(){
        console.log("onEnter  0x4F254 ")
     },
     onLeave: function(retval){
        console.log("onLeave  0x4F254  ",retval.toInt32())   
     }
})

```

结果是：

```
onEnter  0x4F254 
onEnter  0x4F254 
onLeave  0x4F254   0
onLeave  0x4F254   1

```

尝试修改返回值为 0，测试是否能过掉检测：

```
Interceptor.attach(libanon.base.add(0x4F254),{
     onEnter: function(){
        console.log("onEnter  0x4F254 ")
     },
     onLeave: function(retval){
        retval.replace(0)
        console.log("onLeave  0x4F254  ",retval.toInt32())   
     }
})

```

返回值修改为 0，成功。但是 app 依然崩溃了，果然，一切不会这么简单。继续分析中间的逻辑，检测大概率就是在中间的逻辑完成。

```
onEnter  0x4F254 
onEnter  0x4F254 
onLeave  0x4F254   0
onLeave  0x4F254   0

```

必须深入其内部校验路径（而非仅 “短路返回”）——尤其是完整性 / 哈希相关分支。

### 11. 浮点运算特征联合 GPT 定位 CRC 哈希算法

进这个`sub_4F254`函数，从上往下看，第一个可疑的地方：

```
if ( v34 == sub_161E8(unk_82948, unk_82990 - unk_82948, &v75) )
    sub_14D24(byte_872AC, 64, v32, 32, v33);

```

点进`sub_161E8`函数，可以看到大量浮点数寄存器，`vaddq_s64`、`veorq_s8`、`vorrq_s8`，一般这种不是哈希就是加解密，再结合函数开头就有一些初始常量，大概率是哈希：

```
  v12 = v10 ^ 0x7465646279746573LL;
  v13 = v10 ^ 0x646F72616E646F6DLL;
  v14 = a2 << 56;
  v15 = *a3 ^ 0x6C7967656E657261LL;
  v16 = *a3 ^ 0x736F6D6570736575LL;

```

其实猜算法这件事情，最拿手的应该是 GPT，直接整个 F5 全部复制黏贴过去问 GPT，GPT 告诉我这很像是 SipHash 哈希算法的实现，具体是 SipHash-2-4 变种（2 轮压缩，4 轮最终化），真假先不做评价，先 hook 看下。

```
var libanon = Process.findRangeByAddress(libdexprotector.base.add(0xC838).readPointer()).base
Interceptor.attach(libanon.base.add(0x161E8),{
    onEnter:function(args){
        console.log("sub_161E8 enter")
    },onLeave:function(ret){
        console.log("sub_161E8 leave",ret)
    }
})

```

单纯 hook 结果没什么特别：

```
sub_161E8 enter
sub_161E8 leave 0xff4414fe9bf00ecd
sub_161E8 enter
sub_161E8 leave 0xff4414fe9bf00ecd

```

但从结果来看这里其实是做 crc 哈希校验的地方。sub_161E8 属于完整性哈希链路；但仅满足判等未必足以阻断后续自校验 / 熔断。

### 12.HMAC-SHA256 路径确认（内联 SHA 指令识别）

全局 hex 搜挂哈希魔术定位 CRC 校验

#### 12.1 分析定位 HMAC-SHA256

问了下 GPT，`0xff4414fe9bf00ecd`这个值看起来像是一个 64 位的哈希值（16 个十六进制字符 * 4 位 / 字符 = 64 位）。根据其长度和格式，它最有可能来自以下几种哈希算法：

*   xxHash: 一个极快的非加密哈希算法，64 位版本会生成一个 16 字符的十六进制值。这是一个非常强有力的候选。
*   MurmurHash3: 另一个经典的非加密哈希函数，其 128 位版本更常见，但它也有 64 位的变体。
*   CityHash, FarmHash: 由 Google 发布的哈希函数系列，用于类似的目的，能产生 64 位哈希。
*   SipHash: 虽然也是 64 位输出，但它更注重防止哈希洪水攻击，常用在编程语言的字典实现中（如 Python、Ruby）。  
    当然如果是结果导向的话，逆完之后发现是`xxHash`算法。
*   继续回到`sub_4F254`里面，如果返回值与 v34 相等，则会进入`sub_14D24`，那再进入`sub_14D24`看下做了什么。
    
    ```
    v34 = unk_8A810;
    if ( v34 == sub_161E8(unk_82948, unk_82990 - unk_82948, &v75) )
    sub_14D24(byte_872AC, 64, v32, 32, v33);
    
    ```
    
    在`sub_14D24`里面可以看到，`sub_2FA44`做了一些初始化，`sub_304A8`、`sub_30B14`里面则演都不演了，直接内联汇编了 SHA256 的算法，也可能是编译优化的产物，为了加快运行速度。  
    ![](https://attach.52pojie.cn/forum/202511/20/220611iw2kkqdw6mkq1kuw.png)
    
    **img_9.png** _(276.95 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=MjgxNTg1MnwyMGUzMmUxOXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)
    
    2025-11-20 22:06 上传
    
    两次`SHA256`的计算，那合理猜测是做的`HMAC`算法，合理猜测`sub_14D24`就是`HMAC`的入口。
    

#### 12.2 通过`010editor`，快速查找使用了 sha256 的函数

打开 IDA 的选项→常规，操作码字节数改成 8，点确定，随便找个`SHA256`里面已经优化好的汇编，看下操作码，比如这一行：

```
ROM:0000000000030708 83 28 28 5E   SHA256SU0       V3.4S, V4.4S

```

就是`8328285E`，把`linanon.so`文件拖到`010 editor`里面去，全局搜这个二进制，可以得到如图 12 处结果。  
复制行号，到 IDA 里去看了下，前六处属于刚刚的`sub_304A8`函数，后六处属于`sub_30B14`函数，也就是一共就俩函数进行 SHA256 校验。

### 13. 等式化替换策略：让 CRC/Hash“必相等”

基于 lr 获取调用点（0x4f5a4 / 0x4f6cc / 0x5c494），对 sub_161E8 的返回值在不同入栈点替换为预期内存值（0x8A810/0x8AB20）以通过等式校验。

#### 13.1 `HMAC-SHA256`入口函数分析

再回到`sub_14D24`可以发现，先调用了`sub_304A8`函数之后，又立即调用了`sub_30B14`，也就是`sub_14D24`应该是`HMAC-SHA256`的入口。  
那同样 hook 看下有没有经过`sub_14D24`函数。

```
Interceptor.attach(libanon.base.add(0x14D24),{
    onEnter:function(args){
        console.log("sub_14D24 enter",this.context.lr.sub(libanon.base))
    },onLeave:function(ret){
        console.log("sub_14D24 leave",ret)
    }
})

```

输出是：

```
sub_14D24 enter 0x4ee38
sub_14D24 leave 0x7ff213aa70

```

而此处调用的汇编地址是`0x4F5C8`，也就是`sub_4F254+354`处，很明显不是`0x4ee38`，也就是此处没有进入执行`sub_14D24`的逻辑。  
![](https://attach.52pojie.cn/forum/202511/20/220613h3dgtft3dr6v9d1g.png)

**img_10.png** _(116.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1M3wwZTAyMTU2MXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

#### 13.2 HOOK 绕过 CRC 校验使其相等

那如何让它进入呢？只要`v34`与`sub_161E8`运行的结果相等即可进入。这其实就是一段 CRC 的内存校验，可以写个脚本来使其相等。  
首先看下有哪些地方对`sub_161E8`进行了校验，前面的代码加上一句`lr`返回值地址的输出：

```
console.log("sub_161E8 enter",this.context.lr.sub(libanon.base))

```

跑一下：

```
sub_161E8 enter 0x4f5a4
sub_161E8 leave 0x7799165d5282bf95
sub_161E8 enter 0x4f6cc
sub_161E8 leave 0x7799165d5282bf95

```

两处进行了校验，`0x4f5a4`和`0x4f6cc`处，那就把这两处的返回值都看一下，要等于哪处内存的值，才能进入相等后的逻辑：  
通过 x 查找引用，发现还有一个地方也使用 sub_161E8 做了比较`0x5C494`

```
var hash_crc = {
        "0x4f5a4" : 0x8A810,
        "0x4f6cc" : 0x8A810,
        "0x5c494" : 0x8AB20
}

```

最终完整的相等逻辑代码是：

```
Interceptor.attach(libanon.add(0x161E8),{
    onEnter:function(args){
        console.log("sub_161E8 enter",this.context.lr.sub(libanon))
        this.lr = this.context.lr.sub(libanon)
    },onLeave:function(ret){
        if(hash_crc[this.lr.toString()]){
            ret.replace(libanon.add(hash_crc[this.lr.toString()]).readU64())
            console.log("sub_161E8 leave",ret)
        }                            
    }
})

```

为何`lr`寄存器要使用`onEnter`时机的而不能是`onLeave`时机的？因为 frida 在 hook 替换的时候已经把 lr 修改的面目全非了，这涉及到 native hook 的调用顺序和核心原理，可以问问 GPT，这里不再赘述。所以得保留原来的 lr 才是正确的。  
跑一下：

```
sub_14D24 enter 0x4ee38
sub_14D24 leave 0x7ff213aa70
sub_161E8 enter 0x4f5a4
sub_161E8 leave 0x40a05af38cb96159
sub_14D24 enter 0x4f5c8
sub_14D24 leave 0x7ff2139460
sub_161E8 enter 0x4f6cc
sub_161E8 leave 0x40a05af38cb96159

```

很明显替换成功了，`sub_161E8`返回值由`0x7799165d5282bf95`替换成了`0x40a05af38cb96159`，且进入了`0x4f5c8`处的`sub_14D24`函数计算逻辑。  
只是很不幸，还是没能绕过，进程还是崩溃了。胜败乃兵家常事，英雄请重新来过。

SO 里的每一行汇编都要扒光，要让它没有秘密。全扒光就拥有了维多利亚的秘密。

该策略仅能跨过 “第一道门”，后续仍有追加校验 / 副通道检测（如 maps 轮询 / 线程监控）。

### 14. 字符串解密链与 /proc/self/maps 检测点确认

分析发现字符串处理，关键信息

#### 14.1 发现字符串检测

sub_50130-> 修改 a1 类型为 JNIEnv 之后，发现调用了字符串的方法：  
![](https://attach.52pojie.cn/forum/202511/20/220618rw9o9z9amavckews.png)

**img_12.png** _(309.79 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1NXw4YzA1YjdiOXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

v4 是一个类，传入了 v19，那么猜测 sub_55650 可能是一个字符串解密的函数，下面的 sub_40814 似乎也是一个字符串揭秘函数，  
那么接下来直接 hook 它们

```
sub_55650(&unk_87419, v19, 257);
  v4 = (*a1)->FindClass(a1, v19);
  if ( (*a1)->ExceptionCheck(a1) )
    return 46;
  v5 = (void *)sub_3CF88();
  sub_55650(&unk_87218, v18, 65);
  GetStaticMethodID = (*a1)->GetStaticMethodID;
  v7 = sub_40814(dword_346B, &v16, 22);
  v8 = GetStaticMethodID(a1, v5, v18, v7);
  if ( (*a1)->ExceptionCheck(a1) )
  {
    return 46;
  }
  else
  {
    v11 = (*a1)->ToReflectedMethod(a1, v5, v8, 1);
    sub_55650(&unk_872F5, v17, 65);
    v12 = (*a1)->GetStaticMethodID;
    v13 = sub_40814(dword_3EC3, &v15, 24);
    v14 = v12(a1, v4, v17, v13);
    v9 = 46;
    if ( !(*a1)->ExceptionCheck(a1) )
    {
      (*a1)->CallStaticVoidMethod(a1, v4, v14, v11, a2);
      if ( (*a1)->ExceptionCheck(a1) )
        return 46;
      else
        return 0;
    }
  }

```

```
Interceptor.attach(libanon.base.add(0x55650),{
    onEnter: function(){
        this.lr = this.context.lr.sub(libanon.base)
    },
    onLeave: function(retval){
        console.log("[0x55650]",retval.readCString()," lr => ",this.lr);
    }
})
//Interceptor.attach(libanon.base.add(0x40814),{ //0x40814这个函数内部做了inlinehook，直接这样hook，就崩溃了
//那么这个我们可以偏移4条指令进行hook 
Interceptor.attach(libanon.base.add(0x40814+ 4*4),{ 
    onEnter: function(){
        if(libanon.base){
            this.lr = this.context.lr.sub(libanon.base)
        }  
    },
    onLeave: function(retval){
        if(this.lr){
            console.log("[0x40814]",retval.readCString()," lr => ",this.lr);
        }else{
            console.log("[0x40814]",retval.readCString());
        }

    }
})

```

结果是：  
![](https://attach.52pojie.cn/forum/202511/20/220620sefj2p8ej6gsfc11.png)

**img_13.png** _(172.43 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1NnwwYzExMzA3Y3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

#### 14.2 通过输出的 libart.so 的地址，找到调用函数，并修改返回值

分析这个结果，发现检测了分段 maps, 这里通过 libart.so 的 0x61a78 去找到函数进行 hook，检测一般都是通过这里进行。  
在 ida 中 g 这个 0x61a78，找到调用的函数为 sub_61974，hook 这个看一下结果：

```
Interceptor.attach(libanon.base.add(0x61974),{ 
        onEnter: function(){
            this.lr = this.context.lr.sub(libanon.base)
        },
        onLeave: function(retval){      
            console.log("[0x61974]",retval.toInt32()," lr => ",this.lr);
        }
    })

```

运行结果，发现返回值是 786。在 ida 中分析 sub_61974，返回值 786 是出现了异常，正常返回应该是 v1=0，那么我们这里 hook 替换一下返回值：

```
0x61974 =>  786  lr =>  0x4f630

```

```
retval.replace(0);

```

这里修改返回之后，运行依然崩溃了，但是日志输出要比之前多一些了。  
说明 maps 检测非唯一触发，且与其他面耦合。

### 15. 二分排除法缩窗（定位 “必崩区间”）

注意，这里开始，我们切换 Zygisk Frida Gadget，使用方法在这里（[https://github.com/sucsand/sucsand](https://github.com/sucsand/sucsand)） ，绕过一部分基于 ptrace 的检测，并开始使用二分排除法，来定位出问题的地方。  
将 “必崩窗口” 缩至少量函数，有利于后续精确打补丁。

#### 15.1 使用 Zygisk Frida Gadget 模块

注意: 从 frida-server 换到 Zygisk Frida Gadget 模块后，重启一下手机。使用 Zygisk Frida Gadget 的时候，要先正常启动过一次 app。  
在 sucsand 中，勾选酒店 app，并设置 200 延迟。  
![](https://attach.52pojie.cn/forum/202511/20/220753ljn9k9g2i9iq6kti.png)

**img_22.png** _(160.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2OXwwNTFlMzRlMHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:07 上传

配置完成后，在桌面的酒店应用图标长按，然后点强行停止，然后再重新打开 app，此时 app 启动会被阻塞：  
![](https://attach.52pojie.cn/forum/202511/20/220542ht3nwrsf3wo2b3wg.png)

**使用 zygisk-gadget.png** _(75.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTgzOXxiYWY5ZmMyZHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传

现在开始使用 frida -H 192.168.0.101:9999 -F -l dexprotect2.js 来执行脚本。9999 是模块内置的端口号。  
![](https://attach.52pojie.cn/forum/202511/20/220638jcgf0fr2t98g2fb6.png)

**img_23.png** _(185.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2MnxiYjdiYzJiYnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

#### 15.2 继续分析检测点

继续往下分析 sub_4F254，现在都是体力活儿，只能挨着往下分析，看着确实无聊。

现在来到了 sub_27398, 这里看着像是在做检测：

```
      v41 = sub_27398(v39, &v70, 32, &v69, 8, &v68, 4);
      if ( (unsigned __int8)v68 != unk_872EC
        || BYTE1(v68) != unk_872ED
        || BYTE2(v68) != unk_872EE
        || HIBYTE(v68) != unk_872EF )
      {
        sub_62F48(11);
        v41 = sub_62F74(&v68, 4);
      }
      v24 = v74;
      if ( !v74 )

```

```
Interceptor.attach(libanon.base.add(0x27398),{     //fail
    onEnter: function(){
        this.lr = this.context.lr.sub(libanon.base)
        console.log("onEnter  0x27398 ",this.lr)
        Interceptor.detachAll()
    },
    onLeave: function(retval){
        console.log("[0x27398] ",retval.toInt32(),"lr",this.lr)
    }
})

```

我们在这里执行全部卸载，发现还是崩溃了，说明在这里之前就已经对整个内存进行检测了，那么我们要找一个没有检测的点，来缩小排查范围。  
找这个点的原则是，这个函数只调用了一次，并且卸载后能保证 app 顺利运行。  
4F254 ，可以  
593FC ，可以  
27398 ，不行  
那么就可以确定， 问题出在 593FC~27398 之间。那么我们又从这个中间的函数开始进行排查，一步一步缩小位置：  
3EA0C ，可以  
15128 ，不行  
那么现在进一步缩小到了 3EA0C～15128 之间，分析的范围大大减小了：  
![](https://attach.52pojie.cn/forum/202511/20/220622vr5uiewjheniwdib.png)

**img_14.png** _(71.27 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1N3w5OGI3YjQ3YnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

进一步分析，sub_14F7C，没有看出什么问题，暂时排除。

目前这些函数都没有分析出什么有效的信息。

### 16. 入口校验绕过（复制 text → 替换参数基址）

在没有什么有效信息的情况下，当一次侥幸哥，进行暴力拆解

#### 16.1 发现对入口函数进行了检测

上面分析出来的 sha256 的 sub_304A8，来 hook 一下：

```
Interceptor.attach(libanon.base.add(0x304A8), {
    onEnter: function (args) {
        console.log("onEnter  0x304A8 ",args[0],args[1],args[2],"base ",rangeDetails.base)
    }
})

```

结果是：  
![](https://attach.52pojie.cn/forum/202511/20/220624z99czt11ftst9tn4.png)

**img_15.png** _(68.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1OHxhYTFmOTljYnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

通过分析结果，发现 304A8，对入口的函数也做了检测

#### 16.2 处理对入口函数检测

针对 sub_304A8 的入口校验：

先把匿名段 text 拷贝一份 origin；

若发现其校验针对当前段基址，则把参数中的基址替换为 origin（干净副本）以规避校验。

```
    var origin
    var size=0x7a2a0

    origin = Memory.alloc(size)
    origin.writeByteArray(libanon.base.readByteArray(size))

    Interceptor.attach(libanon.base.add(0x304A8), {
        onEnter: function (args) {
            if (args[1].toString() === libanon.base.toString()) {
                var rangeDetails = Process.findRangeByAddress(args[0]);
                console.log("onEnter  0x304A8 ",args[0],args[1],args[2],"base ",rangeDetails.base)
                args[1] = origin
                console.log("[0x304A8] 替换成功")
            }
        }
    })

```

此时已经过掉了检测，并进入了首页。但是输出看到，不停在刷，maps 检测明显开了线程。  
![](https://attach.52pojie.cn/forum/202511/20/220626trmvr7ztmntlmbn2.png)

**img_16.png** _(150.96 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1OXw4ZjNkMzg5M3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

### 17. 线程面收敛：定位 pthread_create 入口并 “即刻返回”

虽然已经完成了过掉 frida 检测，为了完美一点，我们再来找到线程并处理掉它。

#### 17.1 定位`pthread_create`

根据 "/proc/self/maps" 调用的反向引用，追到 pthread_create

我们使用`[0x40814] /proc/self/maps  lr =>  0x3b66c`的 0x3b66c，在 ids 中按 x，一步一步向上查找。最终发现了 sub_7A230，这个看起来很像是 pthread_create 函数：  
![](https://attach.52pojie.cn/forum/202511/20/220628y4jrji76ivflvfe1.png)

**img_17.png** _(66.46 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2MHxlNjI2MDdiZnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

pthread_create 函数原型：

```
#include <pthread.h>

int pthread_create(
    pthread_t *thread,                  // [out] 新线程ID
    const pthread_attr_t *attr,         // [in]  线程属性(可为NULL)
    void *(*start_routine)(void *),     // [in]  线程入口函数
    void *arg                           // [in]  传给入口函数的参数
);

```

#### 17.2 查找所有使用`pthread_create`的地方，并处理掉

找到被创建的线程入口地址，在运行态用 Arm64Writer 写入 RET，实现就地空返回。

在 ida 中，我们通过 x 键，查找 sub_7A230 的引用，然后去把线程入口函数给处理掉。

![](https://attach.52pojie.cn/forum/202511/20/220630w220sahlmliiyinn.png)

**img_18.png** _(198.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2MXw4ZjNkNDRhMHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传

```
function retFunc(parg2) {
    // 修改内存保护，使其可写
    Memory.protect(parg2, 4, 'rwx');
    // 使用 Arm64Writer 写入 'ret' 指令
    var writer = new Arm64Writer(parg2);
    writer.putRet();
    writer.flush();
    writer.dispose();
    console.log("ret " + parg2 + " success");
}
retFunc(libanon.base.add(0x5A708))
retFunc(libanon.base.add(0x5BE28))
retFunc(libanon.base.add(0x5D1A4))
retFunc(libanon.base.add(0x5D710))

```

最终结果，没有那么多线程检测一直刷了，看着比较舒服。  
![](https://attach.52pojie.cn/forum/202511/20/220749z326v2n2o99n7q88.png)

**img_20.png** _(151.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2N3xkODkxZTkwM3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:07 上传

![](https://attach.52pojie.cn/forum/202511/20/220747ecttt8ot98hac9bd.png)

**img_19.png** _(103.19 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2Nnw4ODNiMDZiMXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:07 上传

结论回顾（方法 > 结论）
-------------

入口选择决定成败：相对直接 Hook System.loadLibrary，从 __loader_android_dlopen_ext 切入能更早获得 “真实装载面” 的证据；

匿名映射段是关键战场：JNI_OnLoad → 函数指针 → 匿名段 这一跳，要求在内存态完成 dump 与 “仅 text 段” 的最小可用反汇编；

校验链路是主线：识别 xxHash/SHA256/HMAC 的组合与落点，用等式化替换与调用点定位做最小侵入的试探；

系统性缩小问题空间：从可卸载点开始做二分排除，定位到刷 maps 的线程入口，补丁而非大面积禁用，以减小误伤面与回归压力。

### 限制与风险

结论对 ROM / 内核 / ART 版本敏感：不同 SoC 与 API Level 下，maps 命名、权限组合、linker 细节均可能影响可见性与时序。

某些对抗属于 “只对当下样本有效”：比如栈上 / 堆上指针偏移、匿名段大小、HMAC 初始材料位置等，需在发布后持续校验与回归。

工具链差异：Zygisk Frida Gadget 与纯 frida-server 的可见性与时序差异

[img_21.png](forum.php?mod=attachment&aid=MjgxNTg2OHw3ZmUyNGFiZHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(233.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2OHw3ZmUyNGFiZHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:07 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220751d2y20qon2lnooohv.png) [img_11.png](forum.php?mod=attachment&aid=MjgxNTg1NHwyOWMwMmFkNnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(186.36 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg1NHwyOWMwMmFkNnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220616hevci398wz9wglyi.png) [libart 的 maps 检测. png](forum.php?mod=attachment&aid=MjgxNTg2NXwwYTEyODYzYXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(114.61 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2NXwwYTEyODYzYXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220644bpgwp0k94p4mpquc.png) [img.png](forum.php?mod=attachment&aid=MjgxNTg2NHxiMGI5NTBlZHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(114.61 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2NHxiMGI5NTBlZHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220642l1aqepccs0cgdhpw.png) [img_24.png](forum.php?mod=attachment&aid=MjgxNTg2M3wwNTcxNWYzNnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(94.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg2M3wwNTcxNWYzNnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:06 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220640ab2s58t86sws702s.png) [过掉的 frida 命令. png](forum.php?mod=attachment&aid=MjgxNTgzN3wxNjlmYjBlYnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(396.75 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTgzN3wxNjlmYjBlYnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220537rfz2ey5rz9z2efz2.png) [小工具配置. png](forum.php?mod=attachment&aid=MjgxNTg0MXwwNzgzZTcxY3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(156.44 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0MXwwNzgzZTcxY3wxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220547yi3vy3k77kwywy5r.png) [img_2.png](forum.php?mod=attachment&aid=MjgxNTg0NXwwNDMyZjdmNXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(193.9 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0NXwwNDMyZjdmNXwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220556ohb9pb9uwmsi1h9c.png) [img_1.png](forum.php?mod=attachment&aid=MjgxNTg0NHw2YjY5YTAyZnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(128.86 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0NHw2YjY5YTAyZnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220553vl4kzb1j1lj1e1d6.png) [frida 过掉的结果. png](forum.php?mod=attachment&aid=MjgxNTg0MnxiYmNiZTk5NHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(148.07 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0MnxiYmNiZTk5NHwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220549lpwro6w68jjriqyo.png) [frida 检测过掉之后进入首页. png](forum.php?mod=attachment&aid=MjgxNTg0M3xiNTU3YTc5ZnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes) _(88.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxNTg0M3xiNTU3YTc5ZnwxNzYzOTU1MDgwfDIxMzQzMXwyMDc0NDg0&nothumb=yes)

2025-11-20 22:05 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202511/20/220551h5gc21gcjdj22cd3.png) ![](https://avatar.52pojie.cn/data/avatar/001/10/94/58_avatar_middle.jpg)正己 表哥，看看把 md 的部分转化一下格式，现在阅读体验不是最佳，论坛的编辑器可以转换 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 实战派狂喜！这篇文章最打动我的是不回避问题从 hook System.loadLibrary 崩，到改 hook __loader_android_dlopen_ext，再到发现检测逻辑藏在匿名段，每一次碰壁后的转向都特别真实，完全还原了实际逆向中的试错过程。  
印象最深的是哈希校验绕过的思路：没有硬刚算法实现，而是通过 LR 寄存器定位调用点，用内存中预存的预期值做等式替换，这种 “借力打力” 的技巧太巧妙了！还有二分法缩窗定位必崩区间，从 593FC~27398 到 3EA0C～15128，把复杂问题拆解成可落地的小目标，这种排查思路不仅适用于逆向，在日常调试中也超实用。  
另外，文中对环境基线的强调（LineageOS 21/Nexus 5X/Magisk 29.0.0）特别良心，很多新手做逆向时忽略环境一致性导致复现失败，这点真的要重点学习。最后想问下大佬们，后续有没有计划补充 “不同 API Level 下 maps 标记差异” 的适配技巧？比如在 Android 14 上匿名段的命名规则是否有变化，hook 时机是否需要调整？期待更多延伸分享！  
评论 3（简洁致敬 + 价值肯定）![](https://avatar.52pojie.cn/images/noavatar_middle.gif)zglhappy 常规思路一般是直接过 pthread_create，有时候暴力出奇迹啊 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zepp7289 这帖子起码写一天吧![](https://static.52pojie.cn/static/image/smiley/laohu/laohu15.gif)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)唐小样儿 最近工作有用到 frida，前来学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) cyberjoo

> [正己 发表于 2025-11-21 10:04](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=54310004&ptid=2074484)  
> 表哥，看看把 md 的部分转化一下格式，现在阅读体验不是最佳，论坛的编辑器可以转换

奇怪，我是用论坛的 MD，把我文章复制进去的。格式怎么带出来。我研究一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fyr666

> [唐小样儿 发表于 2025-11-21 10:23](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=54310294&ptid=2074484)  
> 这帖子起码写一天吧

绝对不止一天，自己复现过程就好几天 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fyr666 太硬核了 ![](https://avatar.52pojie.cn/data/avatar/001/95/57/91_avatar_middle.jpg) mscsky 感谢分享，很有收获