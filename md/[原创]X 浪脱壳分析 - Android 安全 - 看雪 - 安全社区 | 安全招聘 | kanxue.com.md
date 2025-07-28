> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287788.htm)

> [原创]X 浪脱壳分析

**版本：**  
x_14.10.0_vcode_7021_wm_3333_1001_so_32_64_x_6165_225001

**运行环境：**  
ANDROID 14  
Magisk 面具  
magisk-frida

**使用工具：**  
抓包工具 Charles  
JAVA 逆向工具 JADX  
SO 逆向工具 IDA  
动态分析工具 FRIDA

**ANDROID 14 环境配置：**  
Magisk 的环境配置 https://github.com/topjohnwu/Magisk  
Magisk 的 Frida 环境 https://github.com/ViRb3/magisk-frida  
ANDROID 14 的证书配置 https://github.com/wjf0214/Magisk-MoveCACerts

**CA 证书的重要变化**  
原来是 /system/etc/security/cacerts 目录  
新增加 /apex/com.android.conscrypt/cacerts 目录

**IDA 动态调试的重要变化**  
现象为 DDMS 界面白屏，无待调试的进程。Android 官方的更新文档解释为：

> 运行时  
> JDWP 线程创建  
> Android 14 添加了 persist.debug.dalvik.vm.jdwp.enabled 系统属性，用于控制是否在 userdebug build 中创建 Java 调试网络协议 (JDWP) 线程。如需了解详情，请参阅 JDWP 选项。

> JDWP 选项  
> persist.debug.dalvik.vm.jdwp.enabled 系统属性用于控制是否在 userdebug build 中创建 Java 调试网络协议 (JDWP) 线程。默认情况下，系统不会设置此属性，且系统只会为可调试应用创建 JDWP 线程。如需同时为可调试和不可调试应用启用 JDWP 线程，请将 persist.debug.dalvik.vm.jdwp.enabled 设为 1。必须重新启动设备，才能使属性更改生效。  
> 如需在 userdebug build 中调试某个不可调试应用，请运行以下命令来启用 JDWP：  
> adb shell setprop persist.debug.dalvik.vm.jdwp.enabled 1  
> adb reboot  
> 注意：将 persist.debug.dalvik.vm.jdwp.enabled 属性设为 1，这不会使应用变得可调试，而只会允许将调试程序附加到进程。  
> 对于搭载 Android 13 或更低版本的设备，运行时会在 userdebug build 中为可调试和不可调试应用创建 JDWP 线程。这意味着，您可以在 userdebug build 中附加调试程序或分析应用性能。

**Magisk 命令为**

```
su
 
magisk resetprop ro.debuggable 1
magisk resetprop ro.force.debuggable 1
 
magisk resetprop ro.build.type eng
magisk resetprop ro.build.type userdebug
magisk resetprop persist.debug.dalvik.vm.jdwp.enabled 1
 
stop;start

```

抓包确定需要的分析的关键点在 com.sina.weibo.security 下的 SLib 类八个 native 函数

```
private native boolean nativeCheck();
 
private native String nativeGetCheckErr();
 
private native int nativeGetCheckErrType();
 
private native long nativeGetPkgTimeCost();
 
private native long nativeGetV1TimeCost();
 
private native long nativeGetV2TimeCost();
 
private native void nativeInit(String str);
 
private native String nativeNewCalculateS(Context context, String str, int i);

```

同时 com.sina.weibo.security 下的 WBSecurity 类的四个 native 函数

```
public native String getSignature(String str);
 
public native String getTimeCost();
 
public native String getUnPackResult();
 
public native boolean isUnpackSuccess();

```

他对应的 SO 文件为 libslib.so，正常流程为加载 libslib.so 模型，运行 nativeInit 初始化环境参数，运行 nativeNewCalculateS 来计算 s 参数的值；  
查看 IDA 的导出表 发现没有静态注册的 JNI 函数  
![](https://bbs.kanxue.com/upload/tmp/920134_DDWAHVP4KDXD2P3.webp)

直接上 Android 的 unidbg 跑了一下（https://github.com/zhkl0228/unidbg），直接提示

```
INFO [com.github.unidbg.linux.AndroidElfLoader] (AndroidElfLoader:481) - libsoshell.so load dependency libziparchive-private.so failed

```

查看了一下 libziparchive-private.so 这个文件，他是一个解压文件，把 libziparchive-private.so 文件放入 libslib.so 的同目录再次运行  
![](https://bbs.kanxue.com/upload/tmp/920134_BR7X2PVCPF64U5D.webp)

这时的运行结果没有任何提示，发现原来的 so 内有 JNI_OnLoad 函数和. init_array 的段，.init_array 段的函数 在加载 so 时会自动执行，JNI_OnLoad 需要手动调用 mDalvikModule.callJNI_OnLoad(emulator); 来执行；  
![](https://bbs.kanxue.com/upload/tmp/920134_TR7AWG2SARMJ8R7.webp)

这时的运行结果为 RegisterNative 动态注册了 WBSecurity 下的三个 native 函数，但是 SLib 的函数并没有注册；调用 isUnpackSuccess() 和 getUnPackResult() 两个结果，提示原 so 并没有解压成功。  
![](https://bbs.kanxue.com/upload/tmp/920134_FAQ34C4E5UH5V9E.webp)

这时就需要分析 libslib.so 这个文件，他的共享名称竟然是 libsoshell.so，他的依赖库包含 libziparchive-private.so，说明这是一个壳文件，在内部运行中调用 zip 来对原文件进行解压，生成的文件才是 libslib.so 文件。  
![](https://bbs.kanxue.com/upload/tmp/920134_SF8RDYN9W7DWPHT.webp)

其在 init_array 内执行了解压的操作，sub_EEE0 为关键的解压解压函数；  
![](https://bbs.kanxue.com/upload/tmp/920134_8P8X5Q5TEVSJBR9.webp)

这时的一个简单的方法是对这个 ZIP 解压的函数进行 HOOK，就可以得到真正的 libslib.so 文件。  
![](https://bbs.kanxue.com/upload/tmp/920134_2C358Q9SY4VKVY2.webp)

使用 Frida 的 trace 功能来对 init_array 下的所有函数进行 trace，发现其根本没有调用解压函数。

再比较真机上 Frida 的 trace 结果和 unidbg 的 trace 结果，发现其调用的流程发生了变化，有可能是根据环境来判断的执行流程，在 diff 两个 trace 结果时发现了许多的跳转错乱问题，这个壳是经过 Obfuscator-LLVM clang version 4.0.1 (based on Obfuscator-LLVM 4.0.1) 混淆的，流程已经发生了许多变化。

HOOK 了 libc 的 open 函数，发现其在 sub_EEE0 中打开了 / proc/self/maps 文件和 / lib/arm64/libslib.so 文件。其打开 maps 文件有可能是进行反调试，来查看进程运行的实时信息，也是为了得到 libslib.so 在 Android 系统中的绝对路径。

HOOK libc 的 pread 函数，来查看 so 读取了壳文件的哪些内容，首先使用 open 来打开了 / lib/arm64/libslib.so 这个动态库，返回的 fd 是 0x84，然后调用 pread 来读取了文件的部分内容

```
一：读取了ELF的头文件
pread arg 0: 0x84
pread arg 1: 0x7febe06928
pread arg 2: 0x40
pread arg 3: 0x0
 
二：读取了0x6c008处的0x20个字节信息
pread arg 0: 0x84
pread arg 1: 0xb400007563533b98
pread arg 2: 0x20
pread arg 3: 0x6c008
 
三：再次读取了0x6c008出的0x20个字节的信息
pread arg 0: 0x84
pread arg 1: 0xb40000773365efd0
pread arg 2: 0x20
pread arg 3: 0x6c008
 
四：读取了0x0x6c028处的0x140个字节信息
pread arg 0: 0x84
pread arg 1: 0xb40000773365eff0
pread arg 2: 0x140
pread arg 3: 0x6c028
  
五：读取了0x6c168处的0x40e28个字节的信息
pread arg 0: 0x84
pread arg 1: 0xb40000773365f1d0
pread arg 2: 0x40e28
pread arg 3: 0x6c168

```

![](https://bbs.kanxue.com/upload/tmp/920134_CFKEPC9JSXUN2JJ.webp)

其中目的地址并不是连续的，而从 0xb40000773365efd0 至 0xb40000773365f1d0 的地址是连续的，拷贝了 0x6c008 处的 0x20 个字节，0x6c028 处的 0x140 个字节，0x6c168 处的 0x40e28 个字节。使用 Python 根据以上的信息拼接成一个文件来分析一下，生成的新的 so 文件，还需要再次解压。

在进行 trace 的时候发现 Frida 的默认 Stalker.follow 并没有跟踪 call 和 arm 的 bl，需要在设置 events 的 call 为 true，但是在我的运行环境，只要开启了这个选项，Frida 运行就崩溃。

```
events: {
    call: true, // CALL instructions: yes please
    ret: false, // RET instructions
    exec: false, // all instructions: not recommended as it's
    block: false, // block executed: coarse execution trace
    compile: false // block compiled: useful for coverage
}

```

这段需要进行后续的详细分析，分析其解压的逻辑，其中其密文是确定的。

![](https://bbs.kanxue.com/upload/tmp/920134_BVQ36DY7BB8XUJK.webp)

其执行流程为

```
一：打开/lib/arm64/libslib.so 文件；
open arg: /data/app/~~M0_AMbekAj3bHBcAv32Zmg==/com.sina.weibo
1yMlgldzELG6p0wvPZePqw==/lib/arm64/libslib.so
open retval: 0x87
 
二：依次读取/lib/arm64/libslib.so 各段的地址内容；读取密文至0xb400007733604fd0 的开始处。
pread arg 0: 0x87
pread arg 1: 0xb400007733604fd0
pread arg 2: 0x20
pread arg 3: 0x6c008
pread arg 0: 0x87
pread arg 1: 0xb400007733604ff0
pread arg 2: 0x140
pread arg 3: 0x6c028
pread arg 0: 0x87
pread arg 1: 0xb4000077336051d0
pread arg 2: 0x40e28
pread arg 3: 0x6c168
  
三：按照内部的解压函数，来解压0xb400007733604fd0开始处的密文；
  
  
四：将解压后的内容拷贝至0x7733370000处；
memcpy arg 0: 0x7733370000
memcpy arg 1: 0xb400007733604fd0
memcpy arg 2: 0x3f5b8
  
memcpy arg 0: 0x77333bf000
memcpy arg 1: 0xb400007733643fd0
memcpy arg 2: 0x1ed4
  
五：设置0x7733370000处的内容执行权限为执行权限；
  
mprotect arg 0: 0x7733370000
mprotect arg 1: 0x3f5b8
  
mprotect arg 0: 0x77333bf000
mprotect arg 1: 0x1ed4

```

详细信息：

```
open onEnter: 0x7745350f00
open arg: /data/app/~~M0_AMbekAj3bHBcAv32Zmg==/com.sina.weibo
1yMlgldzELG6p0wvPZePqw==/lib/arm64/libslib.so
open retval: 0x87
open onLeave: 0x7745350f00
pread onEnter: 0x77453aba80
pread arg 0: 0x87
pread arg 1: 0xb400007733604fd0
pread arg 2: 0x20
pread arg 3: 0x6c008
pread onLeave: 0x77453aba80
pread onEnter: 0x77453aba80
pread arg 0: 0x87
pread arg 1: 0xb400007733604ff0
pread arg 2: 0x140
pread arg 3: 0x6c028
pread onLeave: 0x77453aba80
pread onEnter: 0x77453aba80
pread arg 0: 0x87
pread arg 1: 0xb4000077336051d0
pread arg 2: 0x40e28
pread arg 3: 0x6c168
pread onLeave: 0x77453aba80
memcpy onEnter: 0x774533c480
memcpy arg 0: 0x7733370000
memcpy arg 1: 0xb400007733604fd0
memcpy arg 2: 0x3f5b8
memcpy onLeave: 0x774533c480
mprotect onEnter: 0x77453abc40
mprotect arg 0: 0x7733370000
mprotect arg 1: 0x3f5b8
mprotect onLeave: 0x77453abc40
memcpy onEnter: 0x774533c480
memcpy arg 0: 0x77333bf000
memcpy arg 1: 0xb400007733643fd0
memcpy arg 2: 0x1ed4
memcpy onLeave: 0x774533c480
mprotect onEnter: 0x77453abc40
mprotect arg 0: 0x77333bf000
mprotect arg 1: 0x1ed4
mprotect onLeave: 0x77453abc40

```

**其他的解决方案:**

HOOK 真机上的 RegisterNatives 函数（https://github.com/lasting-yang/frida_hook_libart），来打印一下调试的结果。

发现其动态注册了 SLib 的 nativeInit 函数和 nativeNewCalculateS 函数，并且给出了其调用动态注册函数的地址和 nativeInit 函数和 nativeNewCalculateS 函数的地址。

![](https://bbs.kanxue.com/upload/tmp/920134_WG5VM7ETCZSSZG9.webp)

现在只需要根据 nativeInit 的地址来找到在 maps 中对应的 so 文件的起始地址和结束地址，使用 Frida 的 dump 功能将内存 dump 下来就是脱壳后的 so 文件。

**ELF 头文件修复**

内存 DUMP 的 so 文件的 ELF 的头是损坏的，与原来的 libslib 的壳头文件相比，前 0x200 处的头不一致，从 0x200 开始一致。

脱壳后我 ELF 头如下：  
![](https://bbs.kanxue.com/upload/tmp/920134_65AQ96NV6E4RCDM.webp)

原始的 libslib 的头文件如下：  
![](https://bbs.kanxue.com/upload/tmp/920134_TYFR9Y2VMECSE2H.webp)

将 libslib 的原始的前 0x200 个头拷贝至脱壳文件，然后修改内存 dump 文件的 elf_header 各项的值为其真实的值；

将 Section header table 的所有项的值都置为 0；主要是修复 Program header table 各项的值；  
![](https://bbs.kanxue.com/upload/tmp/920134_HF2YHZG9MD6VSMG.webp)

主要是修复 program_header_table 的各项的值：  
![](https://bbs.kanxue.com/upload/tmp/920134_ASKFSD9MHZJSTH5.webp)

其中 (R_X) Loadable Segment 和 (RW_) Loadable Segment , 可以根据内存的 maps 信息，来查看各个段的执行权限和读写权限；为方便我将所有的地址都设置为 RWX 权限；

(RW_) Dynamic Segment 就是动态表的地址，他与 got 表相关，需要在找到其动态表的开始地址；可以根据原 libslib 的特征值，来找到动态段的开始地址；

![](https://bbs.kanxue.com/upload/tmp/920134_V9KP84CE3G4BAY7.webp)

其他的各项值都设置为了 0：

```
(R__) Note
(R__) Note
(R__) GCC .eh_frame_hdr Segment
(RW_) GNU Stack (executability)
(R__) GNU Read-only After Relocation

```

修复后的信息如下:  
![](https://bbs.kanxue.com/upload/tmp/920134_U8GDRYF4YEKKP5Z.webp)

![](https://bbs.kanxue.com/upload/tmp/920134_74ZT4Z262A8NVUT.webp)

IDA 打开修复后的文件，发现其动态库的名称已经有 libsoshell.so 变为了 libslib.so, 动态库的依赖也发生了变化，不再依赖 libziparchive-private.so。  
![](https://bbs.kanxue.com/upload/tmp/920134_3ZE7FA8UBA4UPPX.webp)

GOT 表的修复也正常，错误的 GOT 表__imp_内容显示的就是 0x 开始的内存地址。  
![](https://bbs.kanxue.com/upload/tmp/920134_U7EB6ETHNDGAG2U.webp)

代码执行处的内容显示为：  
![](https://bbs.kanxue.com/upload/tmp/920134_A9RGXKN4JR3BHCA.webp)

字符串处也出现了 nativeInit 和 nativeNewCalculateS 函数名称，和 PackageManager 、PackageInfo 和 Signature 的 Android 的签名校验的类。  
![](https://bbs.kanxue.com/upload/tmp/920134_58DZ8WBVNFPDXGS.webp)

把修复后的脱壳文件放到 unidbg-android 项目中，调用 JNI_OnLoad 加载函数 mDalvikModule.callJNI_OnLoad(emulator) 出现动态成功的信息。  
![](https://bbs.kanxue.com/upload/tmp/920134_J528A57GNFXV3UA.webp)

现在只需要按照要求调用 nativeInit(Ljava/lang/String;)V 来初始化环境，和调用  
nativeNewCalculateS(Landroid/content/Context;Ljava/lang/String;I)Ljava/lang/String; 来计算 S 值即可。

调用 nativeInit 的代码如下：

```
public void nativeInit(){
 
    String M = "10EA095010";
    String N = "10EA095010";
    String O = "10EA095060";
 
    String nativeInit = "nativeInit(Ljava/lang/String;)V";
    SLib.callStaticJniMethod(emulator, nativeInit, M);
 
}

```

提示调用成功图如下：  
![](https://bbs.kanxue.com/upload/tmp/920134_7FP9UN44RSN96YJ.webp)

调用 nativeNewCalculateS 的函数代码如下：

```
public void nativeNewCalculateS(){
 
    Object custom = null;
 
    DvmObject context =
    vm.resolveClass("android/content/Context").newObject(custom);
    System.out.println(context);
 
    String str = "2015876998797";
 
    int mode = 3;
 
    String nativeNewCalculateS =
    "nativeNewCalculateS(Landroid/content/Context;Ljava/lang/String;I)Ljava/
     lang/String;";
 
    Object obj = SLib.callStaticJniMethodObject(emulator,
    nativeNewCalculateS, context, str, mode);
 
    System.out.println(obj);
}

```

运行报错，提示信息如下：  
![](https://bbs.kanxue.com/upload/tmp/920134_UZ7ZXFPNTZGP9FH.webp)

日志信息如下

```
FindClass(android/content/ContextWrapper) was called from
RX@0x120046a0[libslib.so]0x46a0
JNIEnv->GetMethodID(android/content/ContextWrapper.getPackageManager()La
 ndroid/content/pm/PackageManager;) => 0x53f2c391 was called from
RX@0x12004440[libslib.so]0x4440
JNIEnv->FindClass(android/content/pm/PackageManager) was called from
RX@0x120047f8[libslib.so]0x47f8
[17:05:57 267]  WARN [com.github.unidbg.linux.ARM64SyscallHandler]
(ARM64SyscallHandler:410) - handleInterrupt intno=2, NR=-130432,
svcNumber=0x11f, PC=unidbg@0xfffe0284,
LR=RX@0x12004af8[libslib.so]0x4af8, syscall=null
com.github.unidbg.arm.backend.BackendException:
dvmObject=android.content.Context@50675690, dvmClass=class
android/content/Context, jmethodID=unidbg@0x53f2c391
 at
com.github.unidbg.linux.android.dvm.DalvikVM64$32.handle(DalvikVM64.java
 :556)
 at
com.github.unidbg.linux.ARM64SyscallHandler.hook(ARM64SyscallHandler.jav
 a:119)
 at
com.github.unidbg.arm.backend.Unicorn2Backend$11.hook(Unicorn2Backend.ja
 va:352)
 at
com.github.unidbg.arm.backend.unicorn.Unicorn$NewHook.onInterrupt(Unicor
 n.java:109)
 at com.github.unidbg.arm.backend.unicorn.Unicorn.emu_start(Native
Method)
 at
com.github.unidbg.arm.backend.unicorn.Unicorn.emu_start(Unicorn.java:312
 )
 at
com.github.unidbg.arm.backend.Unicorn2Backend.emu_start(Unicorn2Backend.
 java:389)
 at
com.github.unidbg.AbstractEmulator.emulate(AbstractEmulator.java:378)
 at com.github.unidbg.thread.Function64.run(Function64.java:39)
 at com.github.unidbg.thread.MainTask.dispatch(MainTask.java:19)
 at
com.github.unidbg.thread.UniThreadDispatcher.run(UniThreadDispatcher.jav
 a:165)
 at
com.github.unidbg.thread.UniThreadDispatcher.runMainForResult(UniThreadD
 ispatcher.java:97)
 at
com.github.unidbg.AbstractEmulator.runMainForResult(AbstractEmulator.jav
 a:341)
 at
com.github.unidbg.arm.AbstractARM64Emulator.eFunc(AbstractARM64Emulator.
 java:262)
 at com.github.unidbg.Module.emulateFunction(Module.java:163)
 at
com.github.unidbg.linux.android.dvm.DvmObject.callJniMethod(DvmObject.ja
 va:135)
 at
com.github.unidbg.linux.android.dvm.DvmClass.callStaticJniMethodObject(D
 vmClass.java:316)
at com.sina.weibo.security.SLib.nativeNewCalculateS(SLib.java:70)
at com.sina.weibo.security.SLib.main(SLib.java:91)
[17:05:57 269]  WARN [com.github.unidbg.AbstractEmulator]
(AbstractEmulator:417) - emulate RX@0x1200241c[libslib.so]0x241c
exception sp=unidbg@0xe4fff1d0,
msg=dvmObject=android.content.Context@50675690, dvmClass=class
android/content/Context, jmethodID=unidbg@0x53f2c391, offset=12ms @
Runnable|Function64 address=0x1200241c, arguments=[unidbg@0xfffe1640,
1627241951, 1348949648, 834133664, 3]
Null

```

根据其提示的信息 android/content/pm/PackageManager 可以断定其主要功能为获取 APK 的签名信息，来进行环境判断，是进行反调试的功能。其 android/content/Context 的获取就是我们传进去的第一个参数。

定位其出错的信息 DalvikVM64.java:556 处的代码如下：  
![](https://bbs.kanxue.com/upload/tmp/920134_D9BVAHC2EJ29FAY.webp)

其报错的主要原因是 dvmClass 根据 jmethodID 来找 DvmMethod 发生错误，错误的原因有可能是我们传入参数 android/content/Context 时发生错误。

定位 Android 源码，我们发现 Context 类和 getPackageManager 的方法都是 abstract 抽象的，Java 的抽象类不能被实例化，他的实现方法必须由子类来继承和实现。

![](https://bbs.kanxue.com/upload/tmp/920134_QVGC3YQQ6DN8GHD.webp)

继续分析错误提示，jmethodID=unidbg@0x53f2c391 这个报错，而这个 jmethodID 的值是从 JNIEnv->GetMethodID(android/content/ContextWrapper.getPackageManager()Landroid/content/pm/PackageManager;) => 0x53f2c391 处获取的，而参数传入的是他的父类 android/content/Context，应该修改为他自己的类名 android/content/ContextWrapper。

修改后这段代码调试通过，不再提示错误。

新的错误提示信息如下：  
![](https://bbs.kanxue.com/upload/tmp/920134_MZYEJ9JESS2JPJ3.webp)

日志信息如下

```
JNIEnv->GetStringUtfChars("2015876998797") was called from
RX@0x1203b48c[libslib.so]0x3b48c
JNIEnv->ReleaseStringUTFChars("2015876998797") was called from
RX@0x1203b390[libslib.so]0x3b390
[18:23:33 969]  WARN [com.github.unidbg.linux.ARM64SyscallHandler]
(ARM64SyscallHandler:410) - handleInterrupt intno=2, NR=-128048,
svcNumber=0x1b4, PC=unidbg@0xfffe0bd4,
LR=RX@0x1203b3d4[libslib.so]0x3b3d4, syscall=null
java.lang.NullPointerException
 at java.util.Objects.requireNonNull(Objects.java:203)
 at
com.github.unidbg.linux.android.dvm.DalvikVM64$181.handle(DalvikVM64.jav
 a:2962)
 at
com.github.unidbg.linux.ARM64SyscallHandler.hook(ARM64SyscallHandler.jav
 a:119)
 at
com.github.unidbg.arm.backend.Unicorn2Backend$11.hook(Unicorn2Backend.ja
 va:352)
 at
com.github.unidbg.arm.backend.unicorn.Unicorn$NewHook.onInterrupt(Unicor
 n.java:109)
 at com.github.unidbg.arm.backend.unicorn.Unicorn.emu_start(Native
Method)
 at
com.github.unidbg.arm.backend.unicorn.Unicorn.emu_start(Unicorn.java:312
 )
 at
com.github.unidbg.arm.backend.Unicorn2Backend.emu_start(Unicorn2Backend.
 java:389)
 at
com.github.unidbg.AbstractEmulator.emulate(AbstractEmulator.java:378)
 at com.github.unidbg.thread.Function64.run(Function64.java:39)
 at com.github.unidbg.thread.MainTask.dispatch(MainTask.java:19)
 at
com.github.unidbg.thread.UniThreadDispatcher.run(UniThreadDispatcher.jav
 a:165)
 at
com.github.unidbg.thread.UniThreadDispatcher.runMainForResult(UniThreadD
 ispatcher.java:97)
 at
com.github.unidbg.AbstractEmulator.runMainForResult(AbstractEmulator.jav
 a:341)
 at
com.github.unidbg.arm.AbstractARM64Emulator.eFunc(AbstractARM64Emulator.
 java:262)
 at com.github.unidbg.Module.emulateFunction(Module.java:163)
 at
com.github.unidbg.linux.android.dvm.DvmObject.callJniMethod(DvmObject.ja
 va:135)
 at
com.github.unidbg.linux.android.dvm.DvmClass.callStaticJniMethodObject(D
 vmClass.java:316)
 at com.sina.weibo.security.SLib.nativeNewCalculateS(SLib.java:70)
 at com.sina.weibo.security.SLib.main(SLib.java:91)

```

提示的信息为 java.lang.NullPointerException，可能的原因是 C 环境的指针传入 Java 环境内错误。

启用调试模式，在 03B3D0 处下断点。

```
public void setDebug(){
  
    Module module = mDalvikModule.getModule();
    Debugger debugger = emulator.attach(DebuggerType.CONSOLE);
    debugger.addBreakPoint(module.base+0x03B3D0);
  
}

```

根据其汇编代码，可以确定其三个参数为 X0 X1 X8 , 根据错误的提示信息，可以确定其是 JNI 调用的 Java 代码，X0 和 X1 是参数，X8 是地址。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_RSYV6QMQMY474R5.webp)

![](https://bbs.kanxue.com/upload/attach/202507/920134_QGKHTEG9NK72WRW.webp)

```
x0=0xfffe1640(-125376) x1=0x3930015a x8=0xfffe0bd0

```

s 进入发现其调用的是 svc 软中断，来调用 Java 代码。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_JM337PA3U3TV8AB.webp)

```
LDR             X8, [X8,#0x520] 中的 0x520 就是SVC 的偏移值。

```

在 log4j.properties 文件开启 log4j.logger.com.github.unidbg.linux.android.dvm=DEBUG 的调试模式。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_DQDWFA6R67SN2UC.webp)

日志信息如下

```
svcNumber=0x1b4, PC=unidbg@0xfffe0bd4,
LR=RX@0x1203b3d4[libslib.so]0x3b3d4, syscall=null
java.lang.NullPointerException
 at java.util.Objects.requireNonNull(Objects.java:203)
 at
com.github.unidbg.linux.android.dvm.DalvikVM64$181.handle(DalvikVM64.jav
 a:2962)
 at
com.github.unidbg.linux.ARM64SyscallHandler.hook(ARM64SyscallHandler.jav
 a:119)
 at
com.github.unidbg.arm.backend.Unicorn2Backend$11.hook(Unicorn2Backend.ja
 va:352)
 at
com.github.unidbg.arm.backend.unicorn.Unicorn$NewHook.onInterrupt(Unicor
 n.java:109)
 at com.github.unidbg.arm.backend.unicorn.Unicorn.emu_start(Native
Method)
 at
com.github.unidbg.arm.backend.unicorn.Unicorn.emu_start(Unicorn.java:312
 )
 at
com.github.unidbg.arm.backend.Unicorn2Backend.emu_start(Unicorn2Backend.
 java:389)
 at
com.github.unidbg.AbstractEmulator.emulate(AbstractEmulator.java:378)
 at com.github.unidbg.thread.Function64.run(Function64.java:39)
 at com.github.unidbg.thread.MainTask.dispatch(MainTask.java:19)
 at
com.github.unidbg.thread.UniThreadDispatcher.run(UniThreadDispatcher.jav
 a:165)
 at
com.github.unidbg.thread.UniThreadDispatcher.runMainForResult(UniThreadD
 ispatcher.java:97)
 at
com.github.unidbg.AbstractEmulator.runMainForResult(AbstractEmulator.jav
 a:341)
 at
com.github.unidbg.arm.AbstractARM64Emulator.eFunc(AbstractARM64Emulator.
 java:262)
 at com.github.unidbg.Module.emulateFunction(Module.java:163)
at
com.github.unidbg.linux.android.dvm.DvmObject.callJniMethod(DvmObject.ja
 va:135)
at
com.github.unidbg.linux.android.dvm.DvmClass.callStaticJniMethodObject(D
 vmClass.java:316)
at com.sina.weibo.security.SLib.nativeNewCalculateS(SLib.java:80)
at com.sina.weibo.security.SLib.main(SLib.java:102)

```

提示的错误信息是 string 类型为 NULL，所以提示 NULL 指针错误。

提示错误的代码为：  
![](https://bbs.kanxue.com/upload/attach/202507/920134_6D46YX65RMWQH27.webp)

基本功能是_GetStringLength 返回字符串的长度，其对应的 offset 正是前述的 0x520。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_WEX9UABKMX4A8S5.webp)

正常的调用_GetStringLength 方法的参数：

```
x0=0xfffe1640(-125376) x1=0x3930015a x8=0xfffe0bd0

```

报错时调用_GetStringLength 方法的参数：

```
x0=0xfffe1640(-125376) x1=0x31aa x8=0xfffe0bd0

```

可以确定为 x1=0x31aa 为 NULL, 查看函数的调用栈，发现了 0x00F160 处的代码如下：  
![](https://bbs.kanxue.com/upload/attach/202507/920134_ESHFRDDG78W63GY.webp)

第一次调用 sub_3B4E4 是正常的，第二次调用 sub_3B4E4 就出现了 NULL 指针的错误，而 qword_50F58 的值正是 0x31AA。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_HE83RX65MDBFGVP.webp)

因为这个 SO 是从内存 dump 的，0x50F58 处的值为运行时的信息，查看 0x50F58 处的引用，发现有 STR 用于写值。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_2Z8XHSPM55SR7UF.webp)

在 0x00E9A8 处是写值到 0x050F58，在 0x00E9A8 处下断点，发现根本没有执行，这个有可能是因为 0x050F58 处已经有值，所以就不再写值。

在 0x50F58 处下内存访问断点

```
public void setMemoryHook(){
    Module module = mDalvikModule.getModule();
  
    emulator.getBackend().hook_add_new(new ReadHook() {
        private UnHook unHook;
  
        @Override
        public void hook(Backend backend, long address, int size, Object user) {
            String hexAddress = Long.toHexString(address - module.base);
            System.err.println("ReadHook Address: 0x" + hexAddress);
            System.err.println("user: " + user);
            AndroidEmulator muser = (AndroidEmulator)user;
            System.err.println("MemoryHook PC: " + muser.getContext().getPCPointer());
        }
  
        @Override
        public void onAttach(UnHook unHook) {
            this.unHook = unHook;
        }
  
        @Override
        public void detach() {
            this.unHook.unhook();
        }
    }, module.base + 0x50F58, module.base + 0x50F58 + 8L, emulator);
  
}

```

提示信息如下：  
![](https://bbs.kanxue.com/upload/attach/202507/920134_8VG8VAMKA3V3S3B.webp)

第一个值 0xeb88 为判断是否为 0 的代码，第二个值 0xf17c 就是调用其报错的代码。  
![](https://bbs.kanxue.com/upload/attach/202507/920134_542VGHCKF6PHTRV.webp)

提示 e_shnum is SHN_UNDEF(0) 错误，修改 ELF 的头的项值：e_shnum_NUMBER_OF_SECTION_HEADER_ENTRIES 为 1 。

![](https://bbs.kanxue.com/upload/attach/202507/920134_FSAMJ9BBQUR8WTH.webp)

动态修改 0x50F58 处的值为 0；

```
public void fix_0x50F58(){
    Module module = mDalvikModule.getModule();
    Pointer pointer_base = UnidbgPointer.pointer(emulator, module.base);
    pointer_base.setLong(0x50F58, 0x0);
}

```

修改后运行正常取得 S 的值。

![](https://bbs.kanxue.com/upload/attach/202507/920134_6DTU4D8HTHK5K9V.webp)

完整的运行信息如下：

```
[20:46:10 870] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:10 871] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x1fd6c1a, global=true
[20:46:10 871] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:10 871] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff9f024221, global=true
[20:46:10 871] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikModule]
(DalvikModule:31) - Call [libslib.so]JNI_OnLoad: 0x1200274c
[20:46:10 872] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$232:3878) - AttachCurrentThread vm=unidbg@0xfffe0080,
env=null, args=null
JNIEnv->FindClass(com/sina/weibo/security/SLib) was called from
RWX@0x120026d4[libslib.so]0x26d4
[20:46:10 873] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff9f024221, global=true
[20:46:10 873] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$3:87) - FindClass env=unidbg@0xfffe1640,
className=com/sina/weibo/security/SLib, hash=0x9f024221
[20:46:10 873] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3366) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib,
methods=RWX@0x12050008[libslib.so]0x50008, nMethods=8
JNIEnv->RegisterNatives(com/sina/weibo/security/SLib,
RWX@0x12050008[libslib.so]0x50008, 8) was called from
RWX@0x1200259c[libslib.so]0x259c
[20:46:10 874] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeInit,
signature=(Ljava/lang/String;)V,
fnPtr=RWX@0x120023b8[libslib.so]0x23b8
RegisterNative(com/sina/weibo/security/SLib,
nativeInit(Ljava/lang/String;)V,
RWX@0x120023b8[libslib.so]0x23b8)
[20:46:10 874] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeNewCalculateS,
signature=(Landroid/content/Context;Ljava/lang/String;I)Ljava/lan
 g/String;, fnPtr=RWX@0x1200241c[libslib.so]0x241c
RegisterNative(com/sina/weibo/security/SLib,
nativeNewCalculateS(Landroid/content/Context;Ljava/lang/String;I)
 Ljava/lang/String;, RWX@0x1200241c[libslib.so]0x241c)
[20:46:10 874] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeGetV1TimeCost,
signature=()J, fnPtr=RWX@0x12002080[libslib.so]0x2080
RegisterNative(com/sina/weibo/security/SLib,
nativeGetV1TimeCost()J, RWX@0x12002080[libslib.so]0x2080)
[20:46:10 874] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeGetV2TimeCost,
signature=()J, fnPtr=RWX@0x12002150[libslib.so]0x2150
RegisterNative(com/sina/weibo/security/SLib,
nativeGetV2TimeCost()J, RWX@0x12002150[libslib.so]0x2150)
[20:46:10 874] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeCheck, signature=()Z,
fnPtr=RWX@0x120022e4[libslib.so]0x22e4
RegisterNative(com/sina/weibo/security/SLib, nativeCheck()Z,
RWX@0x120022e4[libslib.so]0x22e4)
[20:46:10 874] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeGetCheckErr,
signature=()Ljava/lang/String;,
fnPtr=RWX@0x12001e08[libslib.so]0x1e08
RegisterNative(com/sina/weibo/security/SLib,
nativeGetCheckErr()Ljava/lang/String;,
RWX@0x12001e08[libslib.so]0x1e08)
[20:46:10 876] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeGetCheckErrType,
signature=()I, fnPtr=RWX@0x12001fb8[libslib.so]0x1fb8
RegisterNative(com/sina/weibo/security/SLib,
nativeGetCheckErrType()I, RWX@0x12001fb8[libslib.so]0x1fb8)
[20:46:10 876] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$212:3379) - RegisterNatives dvmClass=class
com/sina/weibo/security/SLib, name=nativeGetPkgTimeCost,
signature=()J, fnPtr=RWX@0x12002218[libslib.so]0x2218
RegisterNative(com/sina/weibo/security/SLib,
nativeGetPkgTimeCost()J, RWX@0x12002218[libslib.so]0x2218)
[20:46:10 876] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x9f024221
[20:46:10 877] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikModule]
(DalvikModule:36) - Call [libslib.so]JNI_OnLoad finished:
version=0x10006, offset=6ms
Find native function Java_com_sina_weibo_security_SLib_nativeInit
=> RWX@0x120023b8[libslib.so]0x23b8
[20:46:10 877] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff9f024221, global=false
[20:46:10 878] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:10 878] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff83d61724, global=true
[20:46:10 878] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x17ed40e0, global=false
[20:46:10 878] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$18:268) - NewGlobalRef object=unidbg@0x17ed40e0,
dvmObject="10EA095010"
JNIEnv->NewGlobalRef("10EA095010") was called from
RWX@0x120023f0[libslib.so]0x23f0
[20:46:10 879] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x17ed40e0, global=true
[20:46:10 894] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:10 894] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffffd2318c19, global=true
android.content.ContextWrapper@50675690
Find native function
Java_com_sina_weibo_security_SLib_nativeNewCalculateS =>
RWX@0x1200241c[libslib.so]0x241c
[20:46:10 895] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff9f024221, global=false
[20:46:10 895] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x50675690, global=false
[20:46:10 895] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff83d61724, global=true
[20:46:10 895] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x31b7dea0, global=false
JNIEnv->FindClass(android/content/ContextWrapper) was called from
RWX@0x120046a0[libslib.so]0x46a0
[20:46:10 896] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffffd2318c19, global=true
[20:46:10 896] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$3:87) - FindClass env=unidbg@0xfffe1640,
className=android/content/ContextWrapper, hash=0xd2318c19
[20:46:10 897] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$30:502) - GetMethodID class=unidbg@0xd2318c19,
methodName=getPackageManager,
args=()Landroid/content/pm/PackageManager;,
LR=RWX@0x12004440[libslib.so]0x4440
[20:46:10 897] DEBUG
[com.github.unidbg.linux.android.dvm.DvmClass] (DvmClass:133) -
getMethodID
signature=android/content/ContextWrapper->getPackageManager()Land
 roid/content/pm/PackageManager;, hash=0x53f2c391
JNIEnv->GetMethodID(android/content/ContextWrapper.getPackageMana
 ger()Landroid/content/pm/PackageManager;) => 0x53f2c391 was
called from RWX@0x12004440[libslib.so]0x4440
JNIEnv->FindClass(android/content/pm/PackageManager) was called
from RWX@0x120047f8[libslib.so]0x47f8
[20:46:10 898] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:10 898] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffffdd884762, global=true
[20:46:10 898] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$3:87) - FindClass env=unidbg@0xfffe1640,
className=android/content/pm/PackageManager, hash=0xdd884762
[20:46:10 898] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$32:550) - CallObjectMethodV object=unidbg@0x50675690,
jmethodID=unidbg@0x53f2c391, va_list=unidbg@0xe4fff290,
lr=RWX@0x12004af8[libslib.so]0x4af8
[20:46:10 898] DEBUG
[com.github.unidbg.linux.android.dvm.VaList64] (VaList64:131) -
VaList64 base_p=0xe4fff300, base_integer=0xe4fff290,
base_float=0xe4fff260, mask_integer=0xffffffd8,
mask_float=0xffffff80,
args=()Landroid/content/pm/PackageManager;, shorty=[]
[20:46:10 898] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffffdd884762, global=true
JNIEnv->CallObjectMethodV(android.content.ContextWrapper@50675690
 , getPackageManager() =>
android.content.pm.PackageManager@47d384ee) was called from
RWX@0x12004af8[libslib.so]0x4af8
[20:46:10 899] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x47d384ee, global=false
JNIEnv->FindClass(android/content/pm/PackageInfo) was called from
RWX@0x120049ec[libslib.so]0x49ec
[20:46:10 899] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:10 899] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x30f944f7, global=true
[20:46:10 899] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$3:87) - FindClass env=unidbg@0xfffe1640,
className=android/content/pm/PackageInfo, hash=0x30f944f7
[20:46:10 899] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$30:502) - GetMethodID class=unidbg@0xdd884762,
methodName=getPackageInfo,
args=(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;,
LR=RWX@0x12004700[libslib.so]0x4700
[20:46:10 899] DEBUG
[com.github.unidbg.linux.android.dvm.DvmClass] (DvmClass:133) -
getMethodID
signature=android/content/pm/PackageManager->getPackageInfo(Ljava
 /lang/String;I)Landroid/content/pm/PackageInfo;, hash=0x3bca8377
JNIEnv->GetMethodID(android/content/pm/PackageManager.getPackageI
 nfo(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;) =>
0x3bca8377 was called from RWX@0x12004700[libslib.so]0x4700
[20:46:10 900] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$30:502) - GetMethodID class=unidbg@0xd2318c19,
methodName=getPackageName, args=()Ljava/lang/String;,
LR=RWX@0x12004964[libslib.so]0x4964
[20:46:10 900] DEBUG
[com.github.unidbg.linux.android.dvm.DvmClass] (DvmClass:133) -
getMethodID
signature=android/content/ContextWrapper->getPackageName()Ljava/l
 ang/String;, hash=0xffffffff8bcc2d71
JNIEnv->GetMethodID(android/content/ContextWrapper.getPackageName
 ()Ljava/lang/String;) => 0x8bcc2d71 was called from
RWX@0x12004964[libslib.so]0x4964
[20:46:10 900] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$32:550) - CallObjectMethodV object=unidbg@0x50675690,
jmethodID=unidbg@0xffffffff8bcc2d71, va_list=unidbg@0xe4fff290,
lr=RWX@0x12004af8[libslib.so]0x4af8
[20:46:10 900] DEBUG
[com.github.unidbg.linux.android.dvm.VaList64] (VaList64:131) -
VaList64 base_p=0xe4fff300, base_integer=0xe4fff290,
base_float=0xe4fff260, mask_integer=0xffffffd8,
mask_float=0xffffff80, args=()Ljava/lang/String;, shorty=[]
[20:46:11 012] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff83d61724, global=true
JNIEnv->CallObjectMethodV(android.content.ContextWrapper@50675690
 , getPackageName() => "com.sina.weibo") was called from
RWX@0x12004af8[libslib.so]0x4af8
[20:46:11 013] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x506e6d5e, global=false
[20:46:11 013] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$32:550) - CallObjectMethodV object=unidbg@0x47d384ee,
jmethodID=unidbg@0x3bca8377, va_list=unidbg@0xe4fff290,
lr=RWX@0x12004af8[libslib.so]0x4af8
[20:46:11 013] DEBUG
[com.github.unidbg.linux.android.dvm.VaList64] (VaList64:131) -
VaList64 base_p=0xe4fff300, base_integer=0xe4fff290,
base_float=0xe4fff260, mask_integer=0xffffffe8,
mask_float=0xffffff80,
args=(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;,
shorty=[Ljava/lang/String;, I]
[20:46:11 013] DEBUG
[com.github.unidbg.linux.android.dvm.AbstractJni]
(AbstractJni:318) - callObjectMethodV getPackageInfo
packageName=com.sina.weibo, flags=0x40
[20:46:11 013] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x30f944f7, global=true
JNIEnv->CallObjectMethodV(android.content.pm.PackageManager@47d38
 4ee, getPackageInfo("com.sina.weibo", 0x40) =>
android.content.pm.PackageInfo@96532d6) was called from
RWX@0x12004af8[libslib.so]0x4af8
[20:46:11 014] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x96532d6, global=false
[20:46:11 014] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$12:203) - ExceptionOccurred: 0x0
[20:46:11 014] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$91:1406) - GetFieldID class=unidbg@0x30f944f7,
fieldName=signatures, args=[Landroid/content/pm/Signature;
[20:46:11 014] DEBUG
[com.github.unidbg.linux.android.dvm.DvmClass] (DvmClass:167) -
getFieldID
signature=android/content/pm/PackageInfo->signatures:[Landroid/co
 ntent/pm/Signature;, hash=0x25f17218
JNIEnv->GetFieldID(android/content/pm/PackageInfo.signatures
[Landroid/content/pm/Signature;) => 0x25f17218 was called from
RWX@0x12004794[libslib.so]0x4794
[20:46:11 015] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$92:1428) - GetObjectField object=unidbg@0x96532d6,
jfieldID=unidbg@0x25f17218
[20:46:11 055] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:11 055] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffffdb4a449b, global=true
JNIEnv->GetObjectField(android.content.pm.PackageInfo@96532d6,
signatures [Landroid/content/pm/Signature; =>
[android.content.pm.Signature@3551a94]) was called from
RWX@0x12004618[libslib.so]0x4618
[20:46:11 055] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x531be3c5, global=false
JNIEnv->FindClass(android/content/pm/Signature) was called from
RWX@0x1200473c[libslib.so]0x473c
[20:46:11 055] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffffdb4a449b, global=true
[20:46:11 055] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$3:87) - FindClass env=unidbg@0xfffe1640,
className=android/content/pm/Signature, hash=0xdb4a449b
[20:46:11 056] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$30:502) - GetMethodID class=unidbg@0xdb4a449b,
methodName=toByteArray, args=()[B,
LR=RWX@0x120045ac[libslib.so]0x45ac
[20:46:11 056] DEBUG
[com.github.unidbg.linux.android.dvm.DvmClass] (DvmClass:133) -
getMethodID
signature=android/content/pm/Signature->toByteArray()[B,
hash=0x6a3e2031
JNIEnv->GetMethodID(android/content/pm/Signature.toByteArray()[B)
=> 0x6a3e2031 was called from RWX@0x120045ac[libslib.so]0x45ac
[20:46:11 056] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$163:2693) - GetObjectArrayElement
array=[android.content.pm.Signature@3551a94], index=0
JNIEnv->GetObjectArrayElement([android.content.pm.Signature@3551a
 94], 0) => android.content.pm.Signature@3551a94 was called from
RWX@0x12004844[libslib.so]0x4844
[20:46:11 057] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x3551a94, global=false
[20:46:11 057] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$32:550) - CallObjectMethodV object=unidbg@0x3551a94,
jmethodID=unidbg@0x6a3e2031, va_list=unidbg@0xe4fff290,
lr=RWX@0x12004af8[libslib.so]0x4af8
[20:46:11 057] DEBUG
[com.github.unidbg.linux.android.dvm.VaList64] (VaList64:131) -
VaList64 base_p=0xe4fff300, base_integer=0xe4fff290,
base_float=0xe4fff260, mask_integer=0xffffffd8,
mask_float=0xffffff80, args=()[B, shorty=[]
[20:46:11 057] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x76f84423, global=true
[20:46:11 057] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xb66, global=true
JNIEnv->CallObjectMethodV(android.content.pm.Signature@3551a94,
toByteArray() => [B@52af6cff) was called from
RWX@0x12004af8[libslib.so]0x4af8
[20:46:11 057] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x735b478, global=false
[20:46:11 057] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x3551a94
[20:46:11 057] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$174:2826) - GetByteArrayElements
arrayPointer=unidbg@0x735b478, isCopy=null
JNIEnv->GetByteArrayElements(false) => [B@52af6cff was called
from RWX@0x12003f74[libslib.so]0x3f74
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0xd2318c19
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0xdd884762
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x47d384ee
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x30f944f7
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x96532d6
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x531be3c5
[20:46:11 064] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0xdb4a449b
[20:46:11 065] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$189:3075) - ReleaseByteArrayElements
arrayPointer=unidbg@0x735b478,
pointer=RW@0x12081000[libc++.so]0x1000, mode=0
[20:46:11 065] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x735b478
[20:46:11 065] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x506e6d5e
[20:46:11 072] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$184:3023) - NewStringUTF bytes=RW@0x124d30c0,
string=5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo
JNIEnv->NewStringUTF("5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo") was
called from RWX@0x1200e97c[libslib.so]0xe97c
[20:46:11 072] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff83d61724, global=true
[20:46:11 072] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x2096442d, global=false
[20:46:11 072] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$18:268) - NewGlobalRef object=unidbg@0x2096442d,
dvmObject="5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo"
JNIEnv->NewGlobalRef("5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo") was
called from RWX@0x1200e9a4[libslib.so]0xe9a4
[20:46:11 072] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x2096442d, global=true
[20:46:11 072] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$20:309) - DeleteLocalRef object=unidbg@0x2096442d
[20:46:11 074] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$181:2960) - GetStringLength string="2015876998797",
lr=RWX@0x1203b3d4[libslib.so]0x3b3d4
JNIEnv->GetStringUtfChars("2015876998797") was called from
RWX@0x1203b48c[libslib.so]0x3b48c
[20:46:11 074] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$159:2616) - GetStringUTFChars string="2015876998797",
isCopy=null, value=2015876998797,
lr=RWX@0x1203b48c[libslib.so]0x3b48c
JNIEnv->ReleaseStringUTFChars("2015876998797") was called from
RWX@0x1203b390[libslib.so]0x3b390
[20:46:11 075] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$160:2636) - ReleaseStringUTFChars
string="2015876998797", pointer=RW@0x12081000[libc++.so]0x1000,
lr=RWX@0x1203b390[libslib.so]0x3b390
[20:46:11 075] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$225:3563) - ExceptionCheck throwable=null
[20:46:11 075] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$181:2960) - GetStringLength
string="5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo",
lr=RWX@0x1203b3d4[libslib.so]0x3b3d4
JNIEnv->GetStringUtfChars("5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo") was
called from RWX@0x1203b48c[libslib.so]0x3b48c
[20:46:11 075] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$159:2616) - GetStringUTFChars
string="5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo", isCopy=null,
value=5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo,
lr=RWX@0x1203b48c[libslib.so]0x3b48c
JNIEnv->ReleaseStringUTFChars("5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo")
was called from RWX@0x1203b390[libslib.so]0x3b390
[20:46:11 076] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$160:2636) - ReleaseStringUTFChars
string="5l0WXnhiY4pJ794KIJ7Rw5F45VXg9sjo",
pointer=RW@0x12081000[libc++.so]0x1000,
lr=RWX@0x1203b390[libslib.so]0x3b390
[20:46:11 076] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$225:3563) - ExceptionCheck throwable=null
[20:46:11 076] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$181:2960) - GetStringLength string="10EA095010",
lr=RWX@0x1203b3d4[libslib.so]0x3b3d4
JNIEnv->GetStringUtfChars("10EA095010") was called from
RWX@0x1203b48c[libslib.so]0x3b48c
ReadHook Address: 0x50f58
user:
com.github.unidbg.linux.android.AndroidARM64Emulator@2c9f9fb0
MemoryHook PC: RWX@0x1200eb88[libslib.so]0xeb88
ReadHook Address: 0x50f58
user:
com.github.unidbg.linux.android.AndroidARM64Emulator@2c9f9fb0
MemoryHook PC: RWX@0x1200f17c[libslib.so]0xf17c
[20:46:11 076] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$159:2616) - GetStringUTFChars string="10EA095010",
isCopy=null, value=10EA095010,
lr=RWX@0x1203b48c[libslib.so]0x3b48c
JNIEnv->ReleaseStringUTFChars("10EA095010") was called from
RWX@0x1203b390[libslib.so]0x3b390
[20:46:11 077] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$160:2636) - ReleaseStringUTFChars
string="10EA095010", pointer=RW@0x12081000[libc++.so]0x1000,
lr=RWX@0x1203b390[libslib.so]0x3b390
[20:46:11 077] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$225:3563) - ExceptionCheck throwable=null
[20:46:11 088] DEBUG
[com.github.unidbg.linux.android.dvm.DalvikVM64]
(DalvikVM64$184:3023) - NewStringUTF bytes=unidbg@0xe4fff631,
string=a17486b9
JNIEnv->NewStringUTF("a17486b9") was called from
RWX@0x1200ec2c[libslib.so]0xec2c
[20:46:11 088] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0xffffffff83d61724, global=true
[20:46:11 088] DEBUG [com.github.unidbg.linux.android.dvm.BaseVM]
(BaseVM:146) - addObject hash=0x9f70c54, global=false
"a17486b9"

```

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#协议分析](forum-161-1-120.htm) [#脱壳反混淆](forum-161-1-122.htm)