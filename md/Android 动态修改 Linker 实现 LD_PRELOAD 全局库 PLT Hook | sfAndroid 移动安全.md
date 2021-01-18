> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [sanfengandroid.github.io](https://sanfengandroid.github.io/2021/01/10/Android-%E5%8A%A8%E6%80%81%E4%BF%AE%E6%94%B9Linker%E5%AE%9E%E7%8E%B0LD-PRELOAD%E5%85%A8%E5%B1%80%E5%BA%93PLT-Hook/#more)

[](#前言 "前言")前言
--------------

我们知道 linux 系统中存在 `LD_PRELOAD` 环境变量更改库的链接顺序，影响库的导入函数重定位，而 Android 使用 linux 是内核也包含 `LD_PRELOAD`环境变量，具体使用路径在 [linker_main.cpp](https://cs.android.com/android/platform/superproject/+/master:bionic/linker/linker_main.cpp;l=352) 中 (本文分析源码如未特别提及则都基于最新主分支)。在执行`linker`初始化时访问了该环境变量，后续优先加载将其作为全局库从而影响之后该进程加载的所有库的符号链接，因此后续我们将围绕该环境变量展开

[](#环境变量的初始化、获取、修改 "环境变量的初始化、获取、修改")环境变量的初始化、获取、修改
--------------------------------------------------

环境变量的获取：获取环境变量使用 libc 中的`getenv`方法，而其中使用的全局变量声明为`extern "C" char** environ;`，该导出符号在`libc.so`库中，因此我们也可以通过`dlsym(libchandler, "environ")`来遍历当前进程的所有环境变量。

环境变量的修改：调用 libc 中的`putenv`，如果不存在则添加，存在则会替换为新值

环境变量的初始化：在`linker_main.cpp`文件中的 [linker_main](https://cs.android.com/android/platform/superproject/+/master:bionic/linker/linker_main.cpp;l=318) 方法中有调用`__libc_init_AT_SECURE(args.envp)`，该函数具体代码如下：

```
void __libc_init_AT_SECURE(char** env) {
  
  errno = 0;
  unsigned long is_AT_SECURE = getauxval(AT_SECURE);
  if (errno != 0) __early_abort(__LINE__);

  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  if ((getpid() != 1) || is_AT_SECURE) {
    __nullify_closed_stdio();
  }

  if (is_AT_SECURE) {
    __sanitize_environment_variables(env);
  }

  
  environ = __libc_shared_globals()->init_environ = env;

  __initialize_personality();
}
```

该函数初始化了`environ`全局变量，但是该初始化很明显是在`linker`可执行文件内初始化的跟我们上面所说的在`libc.so`中不同，且这个时候还根本没有装载`libc.so`这个库，这就要看`__libc_shared_globals()`是如何返回的，查找申明结果查找到两个，一个是在 [libc_init_dynamic.cpp](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/bionic/libc_init_dynamic.cpp;l=159;drc=master;bpv=1;bpt=1)

```
extern "C" libc_shared_globals* __loader_shared_globals();
__LIBC_HIDDEN__ libc_shared_globals* __libc_shared_globals() {
  return __loader_shared_globals();
}
```

一个是在 [libc_init_static.cpp](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/bionic/libc_init_static.cpp;drc=master;bpv=1;bpt=1;l=329)

```
__LIBC_HIDDEN__ libc_shared_globals* __libc_shared_globals() {
  static libc_shared_globals globals;
  return &globals;
}
```

可以看出第一个是调用一个外部函数，而第二个是直接返回静态变量，这是因为`libc`编译成静态库`libc.a`使用`libc_init_static`，编译成动态库`libc.so`使用`libc_init_dynamic`，因此`libc.so`中并不存在`__libc_shared_globals`方法的实现，使用 IDA 分析`libc.so`也可发现`__libc_shared_globals`是导入函数。那么谁实现了该函数呢，正是 `linker` 实现了该函数并导出该符号，使用 IDA 分析 `linker` 可发现该导出函数。因为 `linker` 模块本身要使用一些`libc`函数，而它本身又作为动态链接器无法依赖其它库，因此自身静态链接了一份`libc`，所以上面使用`getenv`实际上是访问了`linker`中局部静态变量`globals`数据

环境变量的值设置：上述初始化环境变量使用`__libc_init_AT_SECURE(args.envp)`，`args.envp`正是该环境变量，函数调用向上回溯`__linker_init_post_relocation()`，继续向上回溯[__linker_init()](https://cs.android.com/android/platform/superproject/+/master:bionic/linker/linker_main.cpp;l=661;bpv=0;bpt=1)，该函数是 `linker` 的入口函数，其有关环境变量设置代码如下：

```
extern "C" ElfW(Addr) __linker_init(void* raw_args) {
  
  
  KernelArgumentBlock args(raw_args);
  bionic_tcb temp_tcb __attribute__((uninitialized));
  linker_memclr(&temp_tcb, sizeof(temp_tcb));
  __libc_init_main_thread_early(args, &temp_tcb);

  ...
}
```

[KernelArgumentBlock](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/private/KernelArgumentBlock.h;drc=master;bpv=0;bpt=1;l=30) 源码

```
#pragma once

#include <elf.h>
#include <link.h>
#include <stdint.h>
#include <sys/auxv.h>

#include "platform/bionic/macros.h"





class KernelArgumentBlock {
 public:
  explicit KernelArgumentBlock(void* raw_args) {
    uintptr_t* args = reinterpret_cast<uintptr_t*>(raw_args);
    argc = static_cast<int>(*args);
    argv = reinterpret_cast<char**>(args + 1);
    
    envp = argv + argc + 1;

    
    
    char** p = envp;
    while (*p != nullptr) {
      ++p;
    }
    ++p; 

    auxv = reinterpret_cast<ElfW(auxv_t)*>(p);
  }

  
  
  unsigned long getauxval(unsigned long type) {
    for (ElfW(auxv_t)* v = auxv; v->a_type != AT_NULL; ++v) {
      if (v->a_type == type) {
        return v->a_un.a_val;
      }
    }
    return 0;
  }

  int argc;
  char** argv;
  char** envp;
  ElfW(auxv_t)* auxv;

 private:
  BIONIC_DISALLOW_COPY_AND_ASSIGN(KernelArgumentBlock);
};
```

查看源码可知原来环境变量紧跟着可执行程序参数`argv`后面，而可执行文件都是通过`exec`簇函数调用，最终都进行系统调用 `int execve(const char* __file, char* const* __argv, char* const* __envp);` 该环境变量默认是继承的父进程

[](#如何设置LD-PRELOAD环境变量 "如何设置LD_PRELOAD环境变量")如何设置 LD_PRELOAD 环境变量
----------------------------------------------------------------

上面分析通过 `putenv` 方法确实可以添加`LD_PRELOAD`环境变量，但是在本进程中执行时机太晚，因为 `linker` 在启动的时候才获取`LD_PRELOAD`环境变量，后面就算是设置成功但是也不会触发了，那如何才能生效呢，有以下几种方法

1.  在该进程的父进程设置该环境变量，然后再启动该进程，根据继承关系得到该环境变量，由于 Android 特性所有应用进程都是`zygote`的子进程，似乎设置`zygote`进程环境变量比较完美，但实际情况并非如此，见下面分析
2.  使用`exec`簇函数手动设置程序启动的环境变量，Android Java 层`Runtime.exec`设置环境变量就是使用该方法，只是是`fork`子进程后在子进程调用`exec`簇函数
3.  使用 [Wrap shell script](https://developer.android.google.cn/ndk/guides/wrap-script?hl=en) 以全新进程运行程序，参考官网解释

接下来分析每种方式，我们的目的是在 Hook 每个 App 中的函数。

1.  方式 1 针对非 Apk 执行（如直接执行可执行文件) 等可行，而这相当于在 shell 中设置环境变量或使用`Runtime.exec`执行。而正常 Apk 执行是要经过`zygote`进程`fork`，看似在`zygote`进程设置然后所有 APP 进程都继承，虽然如此但执行时机已经太晚，因为 APP 进程从`zygote`进程`fork`下来其`linker`已经初始化执行过了，且`zygote`连 Java 环境都已经准备好还预加载了一些代码和资源，因此后续不会在获取`LD_PRELOAD`环境变量，有关源码如下
    
    ```
    private Runnable handleChildProc(ZygoteArguments parsedArgs,
                FileDescriptor pipeFd, boolean isZygote) {
            
    
    
    
    
    
            closeSocket();
    
            Zygote.setAppProcessName(parsedArgs, TAG);
    
            
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            if (parsedArgs.mInvokeWith != null) {
                
                WrapperInit.execApplication(parsedArgs.mInvokeWith,
                        parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                        VMRuntime.getCurrentInstructionSet(),
                        pipeFd, parsedArgs.mRemainingArgs);
    
                
                throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
            } else {
                if (!isZygote) {
                    
                    return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                            parsedArgs.mDisabledCompatChanges,
                            parsedArgs.mRemainingArgs, null );
                } else {
                    return ZygoteInit.childZygoteInit(parsedArgs.mTargetSdkVersion,
                            parsedArgs.mRemainingArgs, null );
                }
            }
        }
    
    Runnable processOneCommand(ZygoteServer zygoteServer) {
            String[] args;
    
            try {
                args = Zygote.readArgumentList(mSocketReader);
            } catch (IOException ex) {
                throw new IllegalStateException("IOException on command socket", ex);
            }
    
            ...略
    
            pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
                    parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
                    parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
                    parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mIsTopApp,
                    parsedArgs.mPkgDataInfoList, parsedArgs.mWhitelistedDataInfoList,
                    parsedArgs.mBindMountAppDataDirs, parsedArgs.mBindMountAppStorageDirs);
    
            try {
                if (pid == 0) {
                    
                    zygoteServer.setForkChild();
    
                    zygoteServer.closeServerSocket();
                    IoUtils.closeQuietly(serverPipeFd);
                    serverPipeFd = null;
    
                    return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote);
                } else {
                    
                    
                    IoUtils.closeQuietly(childPipeFd);
                    childPipeFd = null;
                    handleParentProc(pid, serverPipeFd);
                    return null;
                }
            } finally {
                IoUtils.closeQuietly(childPipeFd);
                IoUtils.closeQuietly(serverPipeFd);
            }
        }
    ```
    
    当然还可在`init`进程设置环境变量，`zygote`从`init`进程`fork`下来则继承该环境变量
2.  使用`Runtime.exec`执行，该情况的弊端在于本进程无法生效，后续子进程生效，而正常情况 APP 内执行`Runtime.exec`也只是启动`shell`执行一些短暂脚本，因此 APK 进程并不会被 Hook。使用`exec`簇函数在当前进程执行，那完全是替换当前进程，该方式与方法 3 是相同道理，只是方法 3 执行时机是在`zygote`进程`fork`后立即执行，而本方式是在`App`执行自身代码时才能执行，相当于二次重启 APP，但在实际 APK 执行却无法成功
3.  使用`Wrap shell script`执行，通过文档其最基本脚本如下
    
    ```
    #!/system/bin/sh
    exec "$@"
    ```
    
    通过执行`shell`脚本内部也是调用`exec`簇函数，官方文档[打包 wrap.sh](https://developer.android.google.cn/ndk/guides/wrap-script#packaging_wrapsh) 也要求 APK 必须是可调试的`android:debuggable="true"`，还要设置`android:extractNativeLibs="true"`以及将`wrap.sh`打包到原生库中，这些参数作为启动参数传递给`zygote`，在`Zygote.java`中有如下代码
    
    ```
    static void applyInvokeWithSystemProperty(ZygoteArguments args) {
        if (args.mInvokeWith == null) {
            args.mInvokeWith = getWrapProperty(args.mNiceName);
        }
    }
    public static String getWrapProperty(String appName) {
        if (appName == null || appName.isEmpty()) {
            return null;
        }
    
        String propertyValue = SystemProperties.get("wrap." + appName);
        if (propertyValue != null && !propertyValue.isEmpty()) {
            return propertyValue;
        }
        return null;
    }
    ```
    
    也就是可以设置全局属性`wrap.apppackage`来指定`Wrap shell script`脚本位置方便我们测试。接着上面`handleChildProc`函数分析，当`parsedArgs.mInvokeWith != null`执行 `WrapperInit.execApplication`源码如下
    
    ```
    public static void execApplication(String invokeWith, String niceName,
            int targetSdkVersion, String instructionSet, FileDescriptor pipeFd,
            String[] args) {
        StringBuilder command = new StringBuilder(invokeWith);
    
        final String appProcess;
        if (VMRuntime.is64BitInstructionSet(instructionSet)) {
            appProcess = "/system/bin/app_process64";
        } else {
            appProcess = "/system/bin/app_process32";
        }
        command.append(' ');
        command.append(appProcess);
    
        
        
        
        
        
        
        command.append(" -Xcompiler-option --generate-mini-debug-info");
    
        command.append(" /system/bin --application");
        if (niceName != null) {
            command.append(" '--nice-);
        }
        command.append(" com.android.internal.os.WrapperInit ");
        command.append(pipeFd != null ? pipeFd.getInt$() : 0);
        command.append(' ');
        command.append(targetSdkVersion);
        Zygote.appendQuotedShellArgs(command, args);
        preserveCapabilities();
        Zygote.execShell(command.toString());
    }
    ```
    
    也就是将我们的`Wrap shell script`当作 shell 执行参数，其展开命令如下
    
    ```
    sh -c mywrap.sh /system/bin/app_process64 -Xcompiler-option --generate-mini-debug-info /system/bin --application --nice-name=App包名 com.android.internal.os.WrapperInit pipeFd 手机SDK版本 android.app.ActivityThread seq=数字
    ```
    
    其中 `android.app.ActivityThread seq=数字` 是启动 Apk 传递过来的参数`args`，回过头来方式 2 使用`exec`构造该执行参数即可。这种方式从`app_process`启动 Apk 与启动 Java 程序类似，只是相应参数，uid，Java 环境不同，Apk 包含 Android 上下文而 Java 程序则不包含。使用`app_process`执行必然又会需要动态链接，因此重新运行`linker`使我们设置的环境变量生效。因此我们可以修改`Wrap shell script`脚本如下
    
    ```
    #!/system/bin/sh
    export LD_PRELOAD=/data/local/tmp/ld-test.so
    exec "$@"
    ```
    
    设置全局属性`setprop wrap.app.package path/to/mywrap.sh`, 或者直接设置属性`setprop wrap.app.package LD_PRELOAD=/data/local/tmp/ld-test.so`根据 shell 命令展开都能生效，**注意预加载 so 库的位数要与 APP 位数匹配，脚本和 so 的执行权限和 selinux 属性**

三种方式都有其优劣势，方式 1 如果在`init`进程设置环境变量，那么`zygote`就会包含，进而传染给所有 APP，但是同样所有 APP 都被 Hook 无法指定 APP 生效，且修改难度大。方式 2 使用`exec`子进程生效而本进程由于执行时机较晚不会成功。方式 3 则有更多的权限限制及打包限制。

[](#测试-LD-PRELOAD "测试 LD_PRELOAD")测试 LD_PRELOAD
-----------------------------------------------

方式 1 `init`进程注入其结果显而易见且操作困难（解锁`boot.img`，修改`init`/`启动脚本.rc`等总之要在很早的时机拿到执行权限，可以通过`Magisk`实现，这里不做讲解，当然也可以重新编译源码模拟测试）因此不做测试。

方式 2 `Runtime.exec`子进程生效不做测试

方式 3 设置`Wrap shell script`测试如下，为了方便直接设置全局属性`wrap.app.package=mywrap.sh`预加载库`ld-test.so`源码如下

```
#include <dlfcn.h>
#include <android/log.h>
#include <cstring>
#include "hook_module.h"

#define C_API extern "C"
#define PUBLIC_METHOD __attribute__ ((visibility ("default")))

#define LOGD(format, ...) __android_log_print(ANDROID_LOG_DEBUG, "PreloadTest", format, ##__VA_ARGS__);

C_API PUBLIC_METHOD int access(const char *pathname, int mode) {
    static int (*orig_access)(const char *pathname, int mode) = nullptr;
    if (orig_access == nullptr) {
        void *libc = dlopen("libc.so", 0);
        if (libc == nullptr) {
            return 0;
        }
        orig_access = reinterpret_cast<int (*)(const char *, int)>(dlsym(libc, "access"));
        if (orig_access == nullptr) {
            return 0;
        }
    }
    
    
    if (strcmp("/dev/socket/logdw", pathname) == 0){
        return orig_access(pathname, mode);
    }
    LOGD("Hook模块监听access函数,路径: %s, mode: %d", pathname, mode);
    if (strcmp("/data/local/tmp/ld-test.txt", pathname) == 0){
        
        return -1;
    }
    return orig_access(pathname, mode);
}

static __attribute__((constructor)) void Init() {
    LOGD("LD_PRELOAD模块被加载");
}
```

Java 相关源码如下

```
package org.sfandroid.nativehook.test;

import android.os.Bundle;
import android.util.Log;
import android.view.View;

import androidx.appcompat.app.AppCompatActivity;

import org.sfandroid.nativehook.test.databinding.ActivityMainBinding;

import java.io.File;

public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private ActivityMainBinding binding;

    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
        findViewById(R.id.btn_test).setOnClickListener(this);
        findViewById(R.id.btn_test1).setOnClickListener(this);
        
    }

    public native String stringFromJNI();
    public native void execShell();
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.btn_test:

                Log.d("PreloadTest", "文件存在: " + new File("/data/local/tmp/ld-test.txt").exists());
                break;
            case R.id.btn_test1:
                try {
                    execShell();
                } catch (Throwable e) {
                    Log.e("PreloadTest", "执行错误", e);
                }
                break;
        }
    }
}
```

使用 Android11 官方 x86_64 模拟器，测试 64 位，将预加载库库放至`/data/local/tmp`目录下，并创建`data/local/tmp/ld-test.txt`测试文件，关闭`selinux`方便后续测试，命令如下

```
adb root
adb push path/to/ld-test.so /data/local/tmp
adb shell touch /data/local/tmp/ld-test.txt
adb shell chmod 777 /data/local/tmp/libld-test.so
adb shell chmod 666 /data/local/tmp/ld-test.txt
adb shell setenforce 0
```

当我们未设置`wrap.org.sfandroid.nativehook.test`属性时执行测试结果如下

```
7667-7667/? D/PreloadTest: 文件存在: true
```

上面我们创建了测试文件因此它是真实存在的而当我们执行`adb shell setprop wrap.org.sfandroid.nativehook.test "LD_PRELOAD=/data/local/tmp/libld-test.so"`后重新运行 APP 测试结果如下

```
7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /dev/urandom, mode: 4
7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /dev/__properties__/property_info, mode: 4
7806-7806/? D/PreloadTest: LD_PRELOAD模块被加载
7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /dev/boringssl/selftest/1cca4bc9f53317a49929af0b9283b0f20d38a542739d7db04ff0aa7ca9b9dcd1, mode: 0
7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /dev/boringssl/selftest/1cca4bc9f53317a49929af0b9283b0f20d38a542739d7db04ff0aa7ca9b9dcd1, mode: 0
7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /vendor/overlay, mode: 0
7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /vendor/overlay/config/config.xml, mode: 0

...省略部分

7806-7806/? D/PreloadTest: Hook模块监听access函数,路径: /data/local/tmp/ld-test.txt, mode: 0
7806-7806/? D/PreloadTest: 文件存在: false
```

可以看到`libld-test.so`确实被首先加载，并影响了 Java 层调用`File.exist`（native 层调用 access 函数）结果，从而达到屏蔽的效果。

我们尝试了`wrap.sh`脚本方式需要设置环境变量或指定打包那么再测试一下直接构造`exec`执行参数试试，跟踪`Zygote.exeShell`最终调用`execv`函数，而它`return execve(name, argv, environ);`，因此只需要将`LD_PRELOAD`添加到当前进程环境变量然后调用`execv`函数即可，有关 native 代码如下

```
extern "C"
JNIEXPORT void JNICALL
Java_org_sfandroid_nativehook_test_MainActivity_execShell(JNIEnv *env, jobject thiz) {
    char *ld_preload = getenv("LD_PRELOAD");
    if (ld_preload != nullptr) {
        
        LOGD("已经存在LD_PRELOAD变量: %s", ld_preload);
        return;
    }
    putenv((char *) "LD_PRELOAD=/data/local/tmp/libld-test.so");
    
    const char *args[] = {"/system/bin/sh", "-c",
                          "/system/bin/app_process64 -Xcompiler-option --generate-mini-debug-info /system/bin --application --nice-,
                          nullptr};
    execv(args[0], (char *const *) args);
}
```

这就相当于手动执行`wrap.sh`脚本，只是时间在 App 应用执行时，原理虽然通过但是执行却得到以下结果

```
596-789/? W/InputDispatcher: channel '7c0090b org.sfandroid.nativehook.test/org.sfandroid.nativehook.test.MainActivity (server)' ~ Consumer closed input channel or an error occurred.  events=0x9
596-789/? E/InputDispatcher: channel '7c0090b org.sfandroid.nativehook.test/org.sfandroid.nativehook.test.MainActivity (server)' ~ Channel is unrecoverably broken and will be disposed!
596-2056/? I/system_server: oneway function results will be dropped but finished with status OK and parcel size 4
0-0/? I/binder: undelivered TRANSACTION_COMPLETE
0-0/? I/binder: undelivered transaction 858386, process died.
9687-9687/? D/PreloadTest: Hook模块监听access函数,路径: /dev/urandom, mode: 4
9687-9687/? D/PreloadTest: Hook模块监听access函数,路径: /dev/__properties__/property_info, mode: 4
9687-9687/? D/PreloadTest: LD_PRELOAD模块被加载
9687-9687/? D/PreloadTest: Hook模块监听access函数,路径: /system/bin/app_process64, mode: 1
596-2930/? I/WindowManager: WIN DEATH: Window{7c0090b u0 org.sfandroid.nativehook.test/org.sfandroid.nativehook.test.MainActivity}
596-2930/? W/InputDispatcher: Attempted to unregister already unregistered input channel '7c0090b org.sfandroid.nativehook.test/org.sfandroid.nativehook.test.MainActivity (server)'
596-2921/? I/ActivityManager: Process org.sfandroid.nativehook.test (pid 9687) has died: fg  TOP 
9687-9687/? I/sh: type=1400 audit(0.0:180): avc: denied { execute } for path="/data/local/tmp/libld-test.so" dev="dm-5" ino=65540 scontext=u:r:untrusted_app:s0:c153,c256,c512,c768 tcontext=u:object_r:shell_data_file:s0 tclass=file permissive=1 app=org.sfandroid.nativehook.test
300-300/? I/Zygote: Process 9687 exited due to signal 9 (Killed)
596-2056/? I/system_server: oneway function results will be dropped but finished with status OK and parcel size 4
0-0/? I/init: Untracked pid 9723 received signal 9
596-2921/? W/ActivityTaskManager: Force removing ActivityRecord{c639d07 u0 org.sfandroid.nativehook.test/.MainActivity t34}: app died, no saved state
```

`ActivityManager: Process org.sfandroid.nativehook.test (pid 9687) has died: fg TOP` 最终进程会死亡被`kill`掉，当然你有可能看不到`PreloadTest: LD_PRELOAD模块被加载`该日志，因为进程过早就被杀死掉了，多次尝试可能会查看到（我也试了很多次才出现），这证明`exec`执行确实我们的环境变量已生效，只是因为其它原因被杀死而已。这与`wrap.sh`方式相同只是执行时机一个在`zygote fork`后就执行，一个在 APP 启动后才执行，而之所以被杀也不难猜测，因为 APP 环境全部都准备好的`system_server`里已经包含该 APP 的进程信息，而我们执行`execv`又把这些数据给清除掉了，这理所当然会被`system_server`认为 APP 可能已经死亡了因此被杀死。而在`zygote fork`后执行该 APP 环境根本还未执行，`system_server`没有它的记录，因此可以成功。具体原因可以分析源码，这里就不分析了，因为这离我们主题更远，且我们不走这条路径。

综上分析可实现的路径要么在父进程设置子进程生效要么就设置`Wrap shell script`，而我们针对 APP 指定进程生效最简单方法是设置`wrap.app.package`全局属性，但同时也需要 root 权限，注入`init`进程太麻烦也无法指定 APP 生效。即使`LD_PRELOAD`生效我们也无法在 Hook 模块内拦截`dlopen,dlsym`等函数，因为我们 Hook 模块内部要使用它获取原函数地址，就算自己实现`dlopen,dlsym`函数但是在 Android7.0 以上的命名空间限制`caller`的地址都变成我们的 Hook 模块内的地址，那么跨命名空间调用就会存在问题。

经过以上分析引入了我们本文章的主题 **动态修改 Linker 实现 LD_PRELOAD 效果** 该方式避免了上面的缺点，优点如下

1.  APP 进程内生效，可以指定进程
2.  可以重打包到 APP 内部无需权限，也可进程注入
3.  可以 HOOK 指定模块（如`libandroid-runtime.so`等）
4.  可以 Hook `dlopen,dlsym`等符号，不受命名空间限制
5.  进程`fork`子进程默认继承

[](#LD-PRELOAD原理分析 "LD_PRELOAD原理分析")`LD_PRELOAD`原理分析
----------------------------------------------------

上面已经分析过在`linker_main`函数中获取该环境变量，紧接着往下分析

```
static ElfW(Addr) linker_main(KernelArgumentBlock& args, const char* exe_to_load) {
  ...省略
  
  if (!getauxval(AT_SECURE)) {
    ldpath_env = getenv("LD_LIBRARY_PATH");
    if (ldpath_env != nullptr) {
      INFO("[ LD_LIBRARY_PATH set to \"%s\" ]", ldpath_env);
    }
    
    ldpreload_env = getenv("LD_PRELOAD");
    if (ldpreload_env != nullptr) {
      INFO("[ LD_PRELOAD set to \"%s\" ]", ldpreload_env);
    }
  }

  ...省略

  
  parse_LD_LIBRARY_PATH(ldpath_env);
  
  parse_LD_PRELOAD(ldpreload_env);

  std::vector<android_namespace_t*> namespaces = init_default_namespaces(exe_info.path.c_str());

  if (!si->prelink_image()) __linker_cannot_link(g_argv[0]);

  
  si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL);
  
  for (auto linked_ns : namespaces) {
    if (linked_ns != &g_default_namespace) {
      linked_ns->add_soinfo(somain);
      somain->add_secondary_namespace(linked_ns);
    }
  }

  linker_setup_exe_static_tls(g_argv[0]);

  
  std::vector<const char*> needed_library_name_list;
  size_t ld_preloads_count = 0;

  
  for (const auto& ld_preload_name : g_ld_preload_names) {
    needed_library_name_list.push_back(ld_preload_name.c_str());
    ++ld_preloads_count;
  }
  
  
  for_each_dt_needed(si, [&](const char* name) {
    needed_library_name_list.push_back(name);
  });

  const char** needed_library_names = &needed_library_name_list[0];
  size_t needed_libraries_count = needed_library_name_list.size();
  
  
  if (needed_libraries_count > 0 &&
      !find_libraries(&g_default_namespace,
                      si,
                      needed_library_names,
                      needed_libraries_count,
                      nullptr,
                      &g_ld_preloads,
                      ld_preloads_count,
                      RTLD_GLOBAL,
                      nullptr,
                      true ,
                      &namespaces)) {
    __linker_cannot_link(g_argv[0]);
  } 
  ...省略
}
```

上面分析`LD_PRELOAD`库最先添加到加载队列，然后调用`find_libraries`执行加载，此时`needed_library_names`最前面的元素就是预加载库，继续查看源码

```
bool find_libraries(android_namespace_t* ns,
                    soinfo* start_with,
                    const char* const library_names[],
                    size_t library_names_count,
                    soinfo* soinfos[],
                    std::vector<soinfo*>* ld_preloads,
                    size_t ld_preloads_count,
                    int rtld_flags,
                    const android_dlextinfo* extinfo,
                    bool add_as_children,
                    std::vector<android_namespace_t*>* namespaces) {
  
  std::unordered_map<const soinfo*, ElfReader> readers_map;
  LoadTaskList load_tasks;

  for (size_t i = 0; i < library_names_count; ++i) {
    const char* name = library_names[i];
    load_tasks.push_back(LoadTask::create(name, start_with, ns, &readers_map));
  }

  
  
  
  
  
  

  if (soinfos == nullptr) {
    size_t soinfos_size = sizeof(soinfo*)*library_names_count;
    soinfos = reinterpret_cast<soinfo**>(alloca(soinfos_size));
    memset(soinfos, 0, soinfos_size);
  }

  
  size_t soinfos_count = 0;

  auto scope_guard = android::base::make_scope_guard([&]() {
    for (LoadTask* t : load_tasks) {
      LoadTask::deleter(t);
    }
  });

  ZipArchiveCache zip_archive_cache;

  
  
  for (size_t i = 0; i<load_tasks.size(); ++i) {
    LoadTask* task = load_tasks[i];
    soinfo* needed_by = task->get_needed_by();

    bool is_dt_needed = needed_by != nullptr && (needed_by != start_with || add_as_children);
    task->set_extinfo(is_dt_needed ? nullptr : extinfo);
    task->set_dt_needed(is_dt_needed);

    LD_LOG(kLogDlopen, "find_libraries(ns=%s): task=%s, is_dt_needed=%d", ns->get_name(),
           task->get_name(), is_dt_needed);

    
    
    
    
    if (!find_library_internal(const_cast<android_namespace_t*>(task->get_start_from()),
                               task,
                               &zip_archive_cache,
                               &load_tasks,
                               rtld_flags)) {
      return false;
    }

    soinfo* si = task->get_soinfo();

    if (is_dt_needed) {
      needed_by->add_child(si);
    }

    
    
    
    if (ld_preloads != nullptr && soinfos_count < ld_preloads_count) {
      ld_preloads->push_back(si);
    }

    if (soinfos_count < library_names_count) {
      soinfos[soinfos_count++] = si;
    }
  }
  

  
  LoadTaskList load_list;
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    auto pred = [&](const LoadTask* t) {
      return t->get_soinfo() == si;
    };

    
    if (!si->is_linked() &&
        std::find_if(load_list.begin(), load_list.end(), pred) == load_list.end() ) {
      load_list.push_back(task);
    }
  }
  ...省略

  
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    
    if (!si->is_linked() && !si->prelink_image()) {
      return false;
    }
    register_soinfo_tls(si);
  }

  
  

  
  
  
  if (ld_preloads != nullptr) {
    for (auto&& si : *ld_preloads) {
      si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL);
    }
  }

  
  
  soinfo_list_t new_global_group_members;
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    if (!si->is_linked() && (si->get_dt_flags_1() & DF_1_GLOBAL) != 0) {
      
      new_global_group_members.push_back(si);
    }
  }

  
  if (namespaces != nullptr) {
    for (auto linked_ns : *namespaces) {
        
      for (auto si : new_global_group_members) {
        if (si->get_primary_namespace() != linked_ns) {
          linked_ns->add_soinfo(si);
          si->add_secondary_namespace(linked_ns);
        }
      }
    }
  }

  
  
  
  
  
  std::vector<soinfo*> local_group_roots;
  if (start_with != nullptr && add_as_children) {
    local_group_roots.push_back(start_with);
  } else {
    CHECK(soinfos_count == 1);
    local_group_roots.push_back(soinfos[0]);
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    soinfo* needed_by = task->get_needed_by();
    bool is_dt_needed = needed_by != nullptr && (needed_by != start_with || add_as_children);
    android_namespace_t* needed_by_ns =
        is_dt_needed ? needed_by->get_primary_namespace() : ns;

    if (!si->is_linked() && si->get_primary_namespace() != needed_by_ns) {
      auto it = std::find(local_group_roots.begin(), local_group_roots.end(), si);
      LD_LOG(kLogDlopen,
             "Crossing namespace boundary (si=%s@%p, si_ns=%s@%p, needed_by=%s@%p, ns=%s@%p, needed_by_ns=%s@%p) adding to local_group_roots: %s",
             si->get_realpath(),
             si,
             si->get_primary_namespace()->get_name(),
             si->get_primary_namespace(),
             needed_by == nullptr ? "(nullptr)" : needed_by->get_realpath(),
             needed_by,
             ns->get_name(),
             ns,
             needed_by_ns->get_name(),
             needed_by_ns,
             it == local_group_roots.end() ? "yes" : "no");

      if (it == local_group_roots.end()) {
        local_group_roots.push_back(si);
      }
    }
  }

  
  for (auto root : local_group_roots) {
    soinfo_list_t local_group;
    android_namespace_t* local_group_ns = root->get_primary_namespace();

    walk_dependencies_tree(root,
      [&] (soinfo* si) {
        if (local_group_ns->is_accessible(si)) {
          local_group.push_back(si);
          return kWalkContinue;
        } else {
          return kWalkSkip;
        }
      });

    
    
    soinfo_list_t global_group = local_group_ns->get_global_group();
    SymbolLookupList lookup_list(global_group, local_group);
    soinfo* local_group_root = local_group.front();

    bool linked = local_group.visit([&](soinfo* si) {
      
      
      
      if (!si->is_linked() && si->get_primary_namespace() == local_group_ns) {
        const android_dlextinfo* link_extinfo = nullptr;
        if (si == soinfos[0] || reserved_address_recursive) {
          
          
          link_extinfo = extinfo;
        }
        if (__libc_shared_globals()->load_hook) {
          __libc_shared_globals()->load_hook(si->load_bias, si->phdr, si->phnum);
        }
        lookup_list.set_dt_symbolic_lib(si->has_DT_SYMBOLIC ? si : nullptr);
        
        if (!si->link_image(lookup_list, local_group_root, link_extinfo, &relro_fd_offset) ||
            !get_cfi_shadow()->AfterLoad(si, solist_get_head())) {
          return false;
        }
      }

      return true;
    });

    if (!linked) {
      return false;
    }
  }
  ...省略


  return true;
}
```

经上面代码注释了解到预加载库作为首先加载的库，且库调用`si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL);`设置了`DF_1_GLOBAL`标志，后续再将全局库添加到它所有的链接命名空间内

```
if (namespaces != nullptr) {
    for (auto linked_ns : *namespaces) {
        
      for (auto si : new_global_group_members) {
        if (si->get_primary_namespace() != linked_ns) {
          linked_ns->add_soinfo(si);
          si->add_secondary_namespace(linked_ns);
        }
      }
    }
  }

  
  soinfo_list_t android_namespace_t::get_global_group() {
  soinfo_list_t global_group;
  soinfo_list().for_each([&](soinfo* si) {
    if ((si->get_dt_flags_1() & DF_1_GLOBAL) != 0) {
      global_group.push_back(si);
    }
  });

  return global_group;
}
```

我们后面添加全局库就按照这个步骤来，后续查找符号时全局库集合调用`local_group_ns->get_global_group();`，因为我们本身加载的全局库跟待加载的库都在一个命名空间所以这里包含了我们的全局库，接下来开始链接库`si->link_image(lookup_list, local_group_root, link_extinfo, &relro_fd_offset) ||!get_cfi_shadow()->AfterLoad(si, solist_get_head())` 其中`lookup_list`包含了全局组和本地组，其中`link_image`中调用了`relocate(lookup_list)`来重定向库的导入符号

```
bool soinfo::link_image(const SymbolLookupList& lookup_list, soinfo* local_group_root,
                        const android_dlextinfo* extinfo, size_t* relro_fd_offset) {
  if (is_image_linked()) {
    
    return true;
  }

  if (g_is_ldd && !is_main_executable()) {
    async_safe_format_fd(STDOUT_FILENO, "\t%s => %s (%p)\n", get_soname(),
                         get_realpath(), reinterpret_cast<void*>(base));
  }

  local_group_root_ = local_group_root;
  if (local_group_root_ == nullptr) {
    local_group_root_ = this;
  }

  if ((flags_ & FLAG_LINKER) == 0 && local_group_root_ == this) {
    target_sdk_version_ = get_application_target_sdk_version();
  }

  ...省略

  if (!relocate(lookup_list)) {
    return false;
  }
  ...省略
}
```

符号重定向源码分析

```
bool soinfo::relocate(const SymbolLookupList& lookup_list) {

  VersionTracker version_tracker;

  if (!version_tracker.init(this)) {
    return false;
  }

  Relocator relocator(version_tracker, lookup_list);
  relocator.si = this;
  relocator.si_strtab = strtab_;
  relocator.si_strtab_size = has_min_version(1) ? strtab_size_ : SIZE_MAX;
  relocator.si_symtab = symtab_;
  relocator.tlsdesc_args = &tlsdesc_args_;
  relocator.tls_tp_base = __libc_shared_globals()->static_tls_layout.offset_thread_pointer();

  if (android_relocs_ != nullptr) {
    
    if (android_relocs_size_ > 3 &&
        android_relocs_[0] == 'A' &&
        android_relocs_[1] == 'P' &&
        android_relocs_[2] == 'S' &&
        android_relocs_[3] == '2') {
      DEBUG("[ android relocating %s ]", get_realpath());

      const uint8_t* packed_relocs = android_relocs_ + 4;
      const size_t packed_relocs_size = android_relocs_size_ - 4;

      if (!packed_relocate<RelocMode::Typical>(relocator, sleb128_decoder(packed_relocs, packed_relocs_size))) {
        return false;
      }
    } else {
      DL_ERR("bad android relocation header.");
      return false;
    }
  }

  if (relr_ != nullptr) {
    DEBUG("[ relocating %s relr ]", get_realpath());
    if (!relocate_relr()) {
      return false;
    }
  }

#if defined(USE_RELA)
  if (rela_ != nullptr) {
    DEBUG("[ relocating %s rela ]", get_realpath());

    if (!plain_relocate<RelocMode::Typical>(relocator, rela_, rela_count_)) {
      return false;
    }
  }
  if (plt_rela_ != nullptr) {
    DEBUG("[ relocating %s plt rela ]", get_realpath());
    if (!plain_relocate<RelocMode::JumpTable>(relocator, plt_rela_, plt_rela_count_)) {
      return false;
    }
  }
#else

  if (rel_ != nullptr) {
    DEBUG("[ relocating %s rel ]", get_realpath());
    if (!plain_relocate<RelocMode::Typical>(relocator, rel_, rel_count_)) {
      return false;
    }
  }
  if (plt_rel_ != nullptr) {
    DEBUG("[ relocating %s plt rel ]", get_realpath());
    if (!plain_relocate<RelocMode::JumpTable>(relocator, plt_rel_, plt_rel_count_)) {
      return false;
    }
  }
#endif

  
  
#if defined(__aarch64__)
  
  for (const std::pair<TlsDescriptor*, size_t>& pair : relocator.deferred_tlsdesc_relocs) {
    TlsDescriptor* desc = pair.first;
    desc->func = tlsdesc_resolver_dynamic;
    desc->arg = reinterpret_cast<size_t>(&tlsdesc_args_[pair.second]);
  }
#endif

  return true;
}
```

最终调用 `plain_relocate()`

```
template <RelocMode OptMode, typename ...Args>
static bool plain_relocate(Relocator& relocator, Args ...args) {
  return needs_slow_relocate_loop(relocator) ?
      plain_relocate_impl<RelocMode::General>(relocator, args...) :
      plain_relocate_impl<OptMode>(relocator, args...);
}

template <RelocMode Mode>
__attribute__((noinline))
static bool plain_relocate_impl(Relocator& relocator, rel_t* rels, size_t rel_count) {
  for (size_t i = 0; i < rel_count; ++i) {
    if (!process_relocation<Mode>(relocator, rels[i])) {
      return false;
    }
  }
  return true;
}

template <RelocMode Mode>
__attribute__((always_inline))
static inline bool process_relocation(Relocator& relocator, const rel_t& reloc) {
  return Mode == RelocMode::General ?
      process_relocation_general(relocator, reloc) :
      process_relocation_impl<Mode>(relocator, reloc);
}

__attribute__((noinline))
static bool process_relocation_general(Relocator& relocator, const rel_t& reloc) {
  return process_relocation_impl<RelocMode::General>(relocator, reloc);
}
```

最终实现符号重定向是在`process_relocation_impl`方法

```
template <RelocMode Mode>
__attribute__((always_inline))
static bool process_relocation_impl(Relocator& relocator, const rel_t& reloc) {
  constexpr bool IsGeneral = Mode == RelocMode::General;

  void* const rel_target = reinterpret_cast<void*>(reloc.r_offset + relocator.si->load_bias);
  const uint32_t r_type = ELFW(R_TYPE)(reloc.r_info);
  const uint32_t r_sym = ELFW(R_SYM)(reloc.r_info);

  soinfo* found_in = nullptr;
  const ElfW(Sym)* sym = nullptr;
  const char* sym_name = nullptr;
  ElfW(Addr) sym_addr = 0;

  if (r_sym != 0) {
    
    sym_name = relocator.get_string(relocator.si_symtab[r_sym].st_name);
  }

 ...省略

#if defined(USE_RELA)
  auto get_addend_rel   = [&]() -> ElfW(Addr) { return reloc.r_addend; };
  auto get_addend_norel = [&]() -> ElfW(Addr) { return reloc.r_addend; };
#else
  auto get_addend_rel   = [&]() -> ElfW(Addr) { return *static_cast<ElfW(Addr)*>(rel_target); };
  auto get_addend_norel = [&]() -> ElfW(Addr) { return 0; };
#endif
    
  if (IsGeneral && is_tls_reloc(r_type)) {
    ... 省略
  } else {
    if (r_sym == 0) {
      
    } else {
      
      if (!lookup_symbol<IsGeneral>(relocator, r_sym, sym_name, &found_in, &sym)) return false;
      if (sym != nullptr) {
        const bool should_protect_segments = handle_text_relocs &&
                                             found_in == relocator.si &&
                                             ELF_ST_TYPE(sym->st_info) == STT_GNU_IFUNC;
        if (should_protect_segments && !protect_segments()) return false;
        
        sym_addr = found_in->resolve_symbol_address(sym);
        if (should_protect_segments && !unprotect_segments()) return false;
      } else if constexpr (IsGeneral) {
        
        
        switch (r_type) {
#if defined(__x86_64__)
          case R_X86_64_PC32:
            sym_addr = reinterpret_cast<ElfW(Addr)>(rel_target);
            break;
#elif defined(__i386__)
          case R_386_PC32:
            sym_addr = reinterpret_cast<ElfW(Addr)>(rel_target);
            break;
#endif
        }
      }
    }
  }

  
  if constexpr (IsGeneral || Mode == RelocMode::JumpTable) {
    if (r_type == R_GENERIC_JUMP_SLOT) {
      count_relocation_if<IsGeneral>(kRelocAbsolute);
      const ElfW(Addr) result = sym_addr + get_addend_norel();
      trace_reloc("RELO JMP_SLOT %16p <- %16p %s",
                  rel_target, reinterpret_cast<void*>(result), sym_name);
        
      *static_cast<ElfW(Addr)*>(rel_target) = result;
      return true;
    }
  }
  ...省略其它类型符号地址计算及赋值
}
```

其中最关键的符号查找调用`lookup_symbol`

```
template <bool DoLogging>
__attribute__((always_inline))

static inline bool lookup_symbol(Relocator& relocator, uint32_t r_sym, const char* sym_name,
                                 soinfo** found_in, const ElfW(Sym)** sym) {
  if (r_sym == relocator.cache_sym_val) {
    
    *found_in = relocator.cache_si;
    *sym = relocator.cache_sym;
    count_relocation_if<DoLogging>(kRelocSymbolCached);
  } else {
    
    const version_info* vi = nullptr;
    if (!relocator.si->lookup_version_info(relocator.version_tracker, r_sym, sym_name, &vi)) {
      return false;
    }

    soinfo* local_found_in = nullptr;
    
    const ElfW(Sym)* local_sym = soinfo_do_lookup(sym_name, vi, &local_found_in, relocator.lookup_list);

    relocator.cache_sym_val = r_sym;
    relocator.cache_si = local_found_in;
    relocator.cache_sym = local_sym;
    *found_in = local_found_in;
    *sym = local_sym;
  }

  if (*sym == nullptr) {
    if (ELF_ST_BIND(relocator.si_symtab[r_sym].st_info) != STB_WEAK) {
      DL_ERR("cannot locate symbol \"%s\" referenced by \"%s\"...", sym_name, relocator.si->get_realpath());
      return false;
    }
  }

  count_relocation_if<DoLogging>(kRelocSymbol);
  return true;
}

const ElfW(Sym)* soinfo_do_lookup(const char* name, const version_info* vi,
                                  soinfo** si_found_in, const SymbolLookupList& lookup_list) {
  return lookup_list.needs_slow_path() ?
      soinfo_do_lookup_impl<true>(name, vi, si_found_in, lookup_list) :
      soinfo_do_lookup_impl<false>(name, vi, si_found_in, lookup_list);
}

template <bool IsGeneral>
__attribute__((noinline)) static const ElfW(Sym)*

soinfo_do_lookup_impl(const char* name, const version_info* vi,
                      soinfo** si_found_in, const SymbolLookupList& lookup_list) {
  const auto [ hash, name_len ] = calculate_gnu_hash(name);
  constexpr uint32_t kBloomMaskBits = sizeof(ElfW(Addr)) * 8;
  SymbolName elf_symbol_name(name);
  
  const SymbolLookupLib* end = lookup_list.end();
  const SymbolLookupLib* it = lookup_list.begin();

  while (true) {
    const SymbolLookupLib* lib;
    uint32_t sym_idx;

    
    
    
    while (true) {
      if (it == end) return nullptr;
      lib = it++;

      if (IsGeneral && lib->needs_sysv_lookup()) {
        
        if (const ElfW(Sym)* sym = lib->si_->find_symbol_by_name(elf_symbol_name, vi)) {
          *si_found_in = lib->si_;
          return sym;
        }
        continue;
      }

      if (IsGeneral) {
        TRACE_TYPE(LOOKUP, "SEARCH %s in %s@%p (gnu)",
                   name, lib->si_->get_realpath(), reinterpret_cast<void*>(lib->si_->base));
      }
     
      const uint32_t word_num = (hash / kBloomMaskBits) & lib->gnu_maskwords_;
      const ElfW(Addr) bloom_word = lib->gnu_bloom_filter_[word_num];
      const uint32_t h1 = hash % kBloomMaskBits;
      const uint32_t h2 = (hash >> lib->gnu_shift2_) % kBloomMaskBits;

      if ((1 & (bloom_word >> h1) & (bloom_word >> h2)) == 1) {
        sym_idx = lib->gnu_bucket_[hash % lib->gnu_nbucket_];
        if (sym_idx != 0) {
          break;
        }
      }

      if (IsGeneral) {
        TRACE_TYPE(LOOKUP, "NOT FOUND %s in %s@%p",
                   name, lib->si_->get_realpath(), reinterpret_cast<void*>(lib->si_->base));
      }
    }

    
    ElfW(Versym) verneed = kVersymNotNeeded;
    bool calculated_verneed = false;

    uint32_t chain_value = 0;
    const ElfW(Sym)* sym = nullptr;
    
    do {
      sym = lib->symtab_ + sym_idx;
      chain_value = lib->gnu_chain_[sym_idx];
      if ((chain_value >> 1) == (hash >> 1)) {
        if (vi != nullptr && !calculated_verneed) {
          calculated_verneed = true;
          verneed = find_verdef_version_index(lib->si_, vi);
        }
        
        if (check_symbol_version(lib->versym_, sym_idx, verneed) &&
            static_cast<size_t>(sym->st_name) + name_len + 1 <= lib->strtab_size_ &&
            memcmp(lib->strtab_ + sym->st_name, name, name_len + 1) == 0 &&
            is_symbol_global_and_defined(lib->si_, sym)) {
          *si_found_in = lib->si_;
          if (IsGeneral) {
            TRACE_TYPE(LOOKUP, "FOUND %s in %s (%p) %zd",
                       name, lib->si_->get_realpath(), reinterpret_cast<void*>(sym->st_value),
                       static_cast<size_t>(sym->st_size));
          }
          return sym;
        }
      }
      ++sym_idx;
    } while ((chain_value & 1) == 0);

    if (IsGeneral) {
      TRACE_TYPE(LOOKUP, "NOT FOUND %s in %s@%p",
                 name, lib->si_->get_realpath(), reinterpret_cast<void*>(lib->si_->base));
    }
  }
}
```

可以发现最终实现符号查找在`soinfo_do_lookup_impl`查找规则是先查找全局库后查找依赖库，有 hash 表则查询 hash 表没用则直接查询符号名称，hash 表也分为`elf hash`（低版本）与`gnu hash`（高版本）

因此我们已经知道了添加全局库的步骤如下

1.  将该全局库`soinfo`添加`DF_1_GLOBAL`标志`si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL)`
2.  添加该`soinfo`到所有链接命名空间`linked_ns->add_soinfo(si);si->add_secondary_namespace(linked_ns);`

分析源码也可知只有在`linker`初始化执行时才会有全局库的加载，后续使用`dlopen`并未解析，那么后续创建的命名空间也会包含全局库吗

```
android_namespace_t* create_namespace(const void* caller_addr,
                                      const char* name,
                                      const char* ld_library_path,
                                      const char* default_library_path,
                                      uint64_t type,
                                      const char* permitted_when_isolated_path,
                                      android_namespace_t* parent_namespace) {
  
  if (parent_namespace == nullptr) {
    
    soinfo* caller_soinfo = find_containing_library(caller_addr);

    parent_namespace = caller_soinfo != nullptr ?
                       caller_soinfo->get_primary_namespace() :
                       g_anonymous_namespace;
  }

  ProtectedDataGuard guard;
  std::vector<std::string> ld_library_paths;
  std::vector<std::string> default_library_paths;
  std::vector<std::string> permitted_paths;

  parse_path(ld_library_path, ":", &ld_library_paths);
  parse_path(default_library_path, ":", &default_library_paths);
  parse_path(permitted_when_isolated_path, ":", &permitted_paths);

  android_namespace_t* ns = new (g_namespace_allocator.alloc()) android_namespace_t();
  ns->set_name(name);
  ns->set_isolated((type & ANDROID_NAMESPACE_TYPE_ISOLATED) != 0);
  ns->set_greylist_enabled((type & ANDROID_NAMESPACE_TYPE_GREYLIST_ENABLED) != 0);
  ns->set_also_used_as_anonymous((type & ANDROID_NAMESPACE_TYPE_ALSO_USED_AS_ANONYMOUS) != 0);

  
  if ((type & ANDROID_NAMESPACE_TYPE_SHARED) != 0) {
    
    
    std::copy(parent_namespace->get_ld_library_paths().begin(),
              parent_namespace->get_ld_library_paths().end(),
              back_inserter(ld_library_paths));

    std::copy(parent_namespace->get_default_library_paths().begin(),
              parent_namespace->get_default_library_paths().end(),
              back_inserter(default_library_paths));

    std::copy(parent_namespace->get_permitted_paths().begin(),
              parent_namespace->get_permitted_paths().end(),
              back_inserter(permitted_paths));

    
    
    add_soinfos_to_namespace(parent_namespace->soinfo_list(), ns);
    
    
    for (auto& link : parent_namespace->linked_namespaces()) {
      ns->add_linked_namespace(link.linked_namespace(), link.shared_lib_sonames(),
                               link.allow_all_shared_libs());
    }
  } else {
    
    
    add_soinfos_to_namespace(parent_namespace->get_shared_group(), ns);
  }

  ns->set_ld_library_paths(std::move(ld_library_paths));
  ns->set_default_library_paths(std::move(default_library_paths));
  ns->set_permitted_paths(std::move(permitted_paths));

  if (ns->is_also_used_as_anonymous() && !set_anonymous_namespace(ns)) {
    DL_ERR("failed to set namespace: [name=\"%s\", ld_library_path=\"%s\", default_library_paths=\"%s\""
           " permitted_paths=\"%s\"] as the anonymous namespace",
           ns->get_name(),
           android::base::Join(ns->get_ld_library_paths(), ':').c_str(),
           android::base::Join(ns->get_default_library_paths(), ':').c_str(),
           android::base::Join(ns->get_permitted_paths(), ':').c_str());
    return nullptr;
  }

  return ns;
}

soinfo_list_t android_namespace_t::get_shared_group() {
  if (this == &g_default_namespace) {
    return get_global_group();
  }

  soinfo_list_t shared_group;
  soinfo_list().for_each([&](soinfo* si) {
    if ((si->get_rtld_flags() & RTLD_GLOBAL) != 0) {
      shared_group.push_back(si);
    }
  });

  return shared_group;
}
```

这里 JAVA 使用命名空间一般有两个，都是跟`ClassLoader`绑定在一起，包含系统`ClassLoader`和应用`ClassLoader`，关于命名空间介绍可以查看我的另一篇文章 [Android7.0 以上命名空间详解 (dlopen 限制)](https://www.52pojie.cn/thread-948942-1-1.html)，`get_shared_group`查找了命名空间内`soinfo`的`RTLD_GLOBAL`库查找，所以只要我们将全局库添加至所有命名空间并设置好标志后续创建的库都会默认包含我们的库。

分析到这里我们的目的已经很明确了：1. 设置全局标志。2. 将库添加到所有命名空间。当然 Android7.0 以下没有命名空间则直接省略第二步。

[](#代码实现添加全局库-参考fake-linker源码 "代码实现添加全局库  参考fake-linker源码")代码实现添加全局库 参考 [fake-linker](https://github.com/sanfengAndroid/fake-linker) 源码
---------------------------------------------------------------------------------------------------------------------------------------

1.  获取所有命名空间
    
    在`linker_main.cpp`中有`static soinfo* solist;`全局静态变量，通过它可以获取到所有已加载的`soinfo`进而得到所有命名空间，执行时机越早则找到的命名空间越全，可能存在部分 Android 使用的命名空间无法找到，只需要在 Hook 模块特殊处理`android_dlopen_ext`确保添加全局库即可
    
2.  设置全局库
    
    参考项目 [fake-linker](https://github.com/sanfengAndroid/fake-linker) 源码
    
3.  项目其它的一些处理
    
    1.  每个 Android 版本`soinfo`几乎都有变动，且不同版本使用 STL 版本可能还不一样，因此针对每个 API 等级一个库，目前支持 Android 5.0~Android 11
    2.  源码涉及到一些`linker`内部符号查找，通过解析节区符号表来实现，通常来说`linker`都没有去除内部符号表，但是其它模块几乎都是去除掉的，因此对于某些特别手机`linker`去除掉内部符号表则无法使用
    3.  由于执行时机的问题某些模块（”libjavacore.so”, “libnativehelper.so”, “libnativeloader.so”, “libart.so”, “libopenjdk.so”，我们在 Application 执行其系统环境已经准备好了）已经链接过了，这时需要调用提供的接口重新链接符号
    4.  模块内部处理了`dlopen,dlsym`等函数，Hook 模块可以拦截这些函数，然后调用提供的接口就可以恢复环境
    5.  内部提供手动重链接和系统重链接，手动重链接自己实现解析符号因此可以设置过滤项

[](#总结 "总结")总结
--------------

通过设置`LD_PRELOAD`环境变量一是执行时机问题，二是权限问题，三是某些符号无法拦截等缺陷，而我们手动更改设置全局库则不受这些影响，甚至可以控制指定模块，指定符号的 Hook，只是内部解析符号和适配 Android 版本稍微麻烦一点。

上面代码分析可能有些乱，但是你只需要记住 Android7.0 以下全局库只需要设置`soinfo`的全局标志即可，Android7.0 以上命名空间限制则还要要将全局库`soinfo`添加到所有命名空间即可，后续创建的命名空间必然继承全局库，然后因为执行时机问题一些库已经加载并链接了，因此我们要重新链接它的导入符号。