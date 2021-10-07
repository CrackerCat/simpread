> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269667.htm)

> [原创] 安卓 GDB 硬件断点使用说明，结合 krhook 的进行了功能演示

[](#安卓gdb硬件断点使用说明，结合krhook的进行了功能演示)安卓 GDB 硬件断点使用说明，结合 krhook 的进行了功能演示
=====================================================================

前言
--

前段时间做了内核层堆栈回溯的功能，打算进一步参考 [rwProcMem33](https://github.com/abcz316/rwProcMem33) 做硬件断点的功能, 设置任意断点然后堆栈回溯。  
在这之前了解了下 GDB 的相关功能，发现也有硬件断点的功能，但是会被 Ptrace 住。但是不管怎样先学学原理吧。因此这是一篇过渡文章

 

关于 krhook 相关功能说明可以参考:

 

[https://github.com/yhnu/op7t/tree/dev/kr_offline](https://github.com/yhnu/op7t/tree/dev/kr_offline)

准备工作
----

1.  Linux 内核要在 2.6.37 以后的版本才支持对 ARM 添加硬件断点
    
    我们使用的内核版本是 4.14.117, 因此完全支持的
    
2.  GDB 要在 7.3 以后的版本才支持对 ARM 添加硬件断点 (我们使用最新的 ndk-23)
    
    因为工作环境使用比较老的 NDK 版本, 因此会出现下面的错误
    
    ![](https://bbs.pediy.com/upload/attach/202110/933020_YSZZQFZ2BAMNPP2.jpg)
    
3.  内核的编译选项需要开启硬件断点支持
    
    硬件断点的功能主要由 kernel 的 ptrace 模块针对 arm 硬件中断进行了封装处理, 但是需要开启对应宏编译, 如果想自己实现硬件断点也可以参考对应的实现
    
    ```
    # https://github.com/yhnu/op7t/blob/dev/blu7t/op7-r70/arch/arm64/kernel/ptrace.c
    CONFIG_COMPAT=y
    CONFIG_HAVE_HW_BREAKPOINT=y
    
    ```
    

小试牛刀
----

1.  使用 ndk 编译下面的示例代码
    
    ```
    // hello.cpp
    #include #include char gBuf[1024] = "1024";
    int gInt = 0;
    int main(int argc, char const *argv[])
    {
        printf("gBuf=%p %p %d\n", gBuf, &gInt, getpid());
        while(1) {
            gInt++;
            // printf("gBuf=%p %d\n", gBuf, gInt);
            sleep(5);
        }
        return 0;
    } 
    ```
    
2.  将 64 位 gdbserver 和 hello push to /data/local/tmp
    
    ```
    adb push android-arm64/gdbserver/gdbserver /data/local/tmp
    adb push hello /data/local/tmp
    
    ```
    
3.  android 启动对应 gdbserver
    
    ```
    OnePlus7T:/data/local/tmp # ./gdbserver :1234 hello
    warning: Found custom handler for signal 39 (Unknown signal 39) preinstalled.
    Some signal dispositions inherited from the environment (SIG_DFL/SIG_IGN)
    won't be propagated to spawned programs.
    Process /data/local/tmp/hello created; pid = 8022
    Listening on port 1234
    
    ```
    
4.  pc 连接安卓 gdbserver
    
    ```
    adb forward tcp:1234 tcp:1234
    #gdb: aliased to /opt/android-ndk-r21e//prebuilt/darwin-x86_64/bin/gdb
    ➜  op7t git:(dev) ✗ gdb
    GNU gdb (GDB) 8.3
    Copyright (C) 2019 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.
    Type "show copying" and "show warranty" for details.
    This GDB was configured as "x86_64-apple-darwin14.5.0".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    .
    Find the GDB manual and other documentation resources online at:
        .
     
    For help, type "help".
    Type "apropos word" to search for commands related to "word".
    (gdb) target remote :1234
    Remote debugging using :1234
    Reading /data/local/tmp/hello from remote target...
    warning: File transfers from remote targets can be slow. Use "set sysroot" to access files locally instead.
    Reading /data/local/tmp/hello from remote target...
    Reading symbols from target:/data/local/tmp/hello...
    Reading /system/bin/linker64 from remote target...
    Reading /system/bin/linker64 from remote target...
    Reading symbols from target:/system/bin/linker64...
    (No debugging symbols found in target:/system/bin/linker64)
    0x0000007fbf634a50 in __dl__start () from target:/system/bin/linker64
    (gdb) b *main #set breakpoint
    Breakpoint 1 at 0x5555555754: file F:/F2021-07/ak47hook/hello/hello.c, line 7.
    (gdb) c       #run
    Continuing.
    Reading /system/lib64/libandroid.so from remote target...
    Reading /apex/com.android.runtime/lib64/bionic/libm.so from remote target...
    Reading /apex/com.android.runtime/lib64/bionic/libdl.so from remote target...
    Reading /apex/com.android.runtime/lib64/bionic/libc.so from remote target...
     
    #trigger the breakpoint
     
    Breakpoint 1, main (argc=0, argv=0x0) at F:/F2021-07/ak47hook/hello/hello.c:7
    7    F:/F2021-07/ak47hook/hello/hello.c: No such file or directory.
     
    (gdb) hbreak *sleep #trigger the hard breakpoint
    Hardware assisted breakpoint 3 at 0x7fbc9ea430
    (gdb) c
    Continuing.
     
    Breakpoint 3, 0x0000007fbc9ea430 in sleep () from target:/apex/com.android.runtime/lib64/bionic/libc.so
    (gdb) 
    ```
    

使用 GDB 分析下 krhook 的相关 demo
--------------------------

1.  通过上面的示例我们已经验证的工具的正确性，下面我们来结合 krhook 的 UserSpace StackWalk 功能达到回溯对战的目的
    
    ```
    OnePlus7T:/data/local/tmp # ps -ef|grep krhook
    u0_a226       6239   757 0 09:05:48 ?     00:27:42 com.DefaultCompany.krhook_unity3d
    root          8117  6707 0 17:18:29 pts/1 00:00:00 grep krhook
    OnePlus7T:/data/local/tmp # ./gdbserver :1234 --attach 6239 #the app status will be trace stopped
    warning: Found custom handler for signal 39 (Unknown signal 39) preinstalled.
    Some signal dispositions inherited from the environment (SIG_DFL/SIG_IGN)
    won't be propagated to spawned programs.
    Attached; pid = 6239
    Listening on port 1234
    
    ```
    
2.  使用 krhook 了解到 PC 调用链为 stack:0x7277206388|0x718592ac18|0x718594c568。。。
    
    ```
    4,1005409478,695976246251,-;[20211006_17:24:08.100513]@4 [e]openat [/storage/emulated/0/Android/data/com.DefaultCompany.krhook_unity3d/files/a.txt] current->pid:[8235] ppid:[8278] uid:[10226] tgid:[8206] stack:0x7277206388|0x718592ac18|0x718594c568|0x718639f054|0x7186392930|0x7186390a6c|0x718638c93c|0x718638c8b0|0x71863bf918|0x7185eb0fd8|0x7185ebd290|0x7185ec061c|0x7185d0e8b0|0x7185d0e980|0x718580e750|0x7185cf90b8|0x718600ed38|0x71861e63fc|0x7185811ab0|0x7185d08388|0x7185d0683c|0x7185d060cc|0x718577084c|0x7185cf77e4|0x718578c230|0x71859c0eac|0x7185923f04|0x718b43b1d0|0x71f4176ac6|0xea33e188d80cff79|0xffffffffffffffff|
    
    ```
    
3.  开始下硬件断点
    
    ```
    ➜  op7t git:(dev) ✗ gdb
    GNU gdb (GDB) 8.3
    Copyright (C) 2019 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.
    Type "show copying" and "show warranty" for details.
    This GDB was configured as "x86_64-apple-darwin14.5.0".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    .
    Find the GDB manual and other documentation resources online at:
        .
     
    For help, type "help".
    Type "apropos word" to search for commands related to "word".
    (gdb)
    (gdb)
    (gdb) target remote :1234
    Remote debugging using :1234
    (gdb) b *0x718594c564
    Breakpoint 1 at 0x718594c564
    (gdb) c
    Continuing.
    [Switching to Thread 8206.8235]
     
    Thread 13 "UnityMain" hit Breakpoint 1, 0x000000718594c564 in il2cpp::icalls::mscorlib::System::IO::MonoIO::Open40(char16_t*, int, int, int, int, int*) ()
    from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    (gdb) b *0x718594c564 #普通断点
    Breakpoint 1 at 0x718594c564
     
    (gdb) hbreak *0x718592ac14 #硬件短点
    Hardware assisted breakpoint 2 at 0x718592ac14
    (gdb) c
    Continuing.
     
    Thread 13 "UnityMain" hit Breakpoint 2, 0x000000718592ac14 in il2cpp::os::File::Open(std::__ndk1::basic_string, std::__ndk1::allocator > const&, int, int, int, int, int*) ()
    from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    (gdb) bt #对应堆栈
    (gdb) bt
    #0  0x000000718592ac14 in il2cpp::os::File::Open(std::__ndk1::basic_string, std::__ndk1::allocator > const&, int, int, int, int, int*) ()
    from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #1  0x000000718594c568 in il2cpp::icalls::mscorlib::System::IO::MonoIO::Open40(char16_t*, int, int, int, int, int*) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #2  0x000000718639f054 in MonoIO_Open_m194115823A6163255C8845AB97ADF010DAD88E22 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #3  0x0000007186392930 in MonoIO_Open_m75D574F44B3C1E6FA4E245D48D5AC73F70BE16B7 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #4  0x0000007186390a6c in FileStream__ctor_mBC5F76C88DBC8C81D1F83407197D75F36E1ADBD7 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #5  0x000000718638c93c in FileStream__ctor_mB254658F1E758D76B41C942CB91BDF38FD544C83 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #6  0x000000718638c8b0 in File_Open_mDA5EB4A312EAEBF8543B13C572271FB5F673A501 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #7  0x00000071863bf918 in FileOpenTest_FileOpen_m76B8151D8C479F745ECF6F56D62D556DB2435397 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #8  0x0000007185eb0fd8 in UnityAction_Invoke_mC9FF5AA1F82FDE635B3B6644CE71C94C31C3E71A () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #9  0x0000007185ebd290 in InvokableCall_Invoke_m0B9E7F14A2C67AB51F01745BD2C6C423114C9394 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #10 0x0000007185ec061c in UnityEvent_Invoke_mB2FA1C76256FE34D5E7F84ABE528AC61CE8A0325 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #11 0x0000007185d0e8b0 in Button_Press_m33BA6E9820146E8EED7AB489A8846D879B76CF41 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #12 0x0000007185d0e980 in Button_OnPointerClick_m4C4EDB8613C2C5B391EFD3A29C58B0AA00DD9B91 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #13 0x000000718580e750 in InterfaceActionInvoker1::Invoke(unsigned int, Il2CppClass*, Il2CppObject*, PointerEventData_tC18994283B7753E430E316A62D9E45BA6D644C63*)
        () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #14 0x0000007185cf90b8 in ExecuteEvents_Execute_m24768528CCF25F4ADB0E66538ABF950C8EE2E9B0 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #15 0x000000718600ed38 in EventFunction_1_Invoke_mB923A0E7E49A56D420C97EB6D98A660EAF8A348D_gshared () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #16 0x00000071861e63fc in ExecuteEvents_Execute_TisRuntimeObject_m69C612263456A3111F97114B38B8A0E2E16E4347_gshared () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #17 0x0000007185811ab0 in ExecuteEvents_Execute_TisIPointerClickHandler_t337D40B4F0C87DA190B55BF225ADB716F4ADCA13_mB8A59713F468FB6A061C8A5DF7FF205EE1C9A855(GameObject_tBD1244AD56B4E59AAD76E5E7C9282EC5CE434F0F*, BaseEventData_t46C9D2AE3183A742EDE89944AF64A23DBF1B80A5*, EventFunction_1_t7BFB6A90DB6AE5607866DE2A89133CA327285B1E*, MethodInfo const*) ()
    from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #18 0x0000007185d08388 in StandaloneInputModule_ProcessTouchPress_m46FBF040EAB0A0F8D832FEB600EF0B9C48E13F61 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #19 0x0000007185d0683c in StandaloneInputModule_ProcessTouchEvents_m74C783AF0B4D517978ECCE3E8A1081F49D174F69 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #20 0x0000007185d060cc in StandaloneInputModule_Process_mF637455BCED017FB359E090B58F15C490EFD2B54 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #21 0x000000718577084c in VirtActionInvoker0::Invoke(unsigned int, Il2CppObject*) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #22 0x0000007185cf77e4 in EventSystem_Update_m12CAEF521A10D406D1A6EA01E00DD851683C7208 () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #23 0x000000718578c230 in RuntimeInvoker_TrueVoid_t22962CB4C05B1D89B55A6E1139F0E87A90987017(void (*)(), MethodInfo const*, void*, void**) ()
    from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #24 0x00000071859c0eac in il2cpp::vm::Runtime::Invoke(MethodInfo const*, void*, void**, Il2CppException**) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #25 0x0000007185923f04 in il2cpp_runtime_invoke () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libil2cpp.so
    #26 0x000000718b43b1d0 in scripting_method_invoke(ScriptingMethodPtr, ScriptingObjectPtr, ScriptingArguments&, ScriptingExceptionPtr*, bool) ()
    from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #27 0x000000718b44b6dc in ScriptingInvocation::Invoke(ScriptingExceptionPtr*, bool) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #28 0x000000718b455c54 in MonoBehaviour::CallUpdateMethod(int) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #29 0x000000718b052eb4 in void BaseBehaviourManager::CommonUpdate() () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #30 0x000000718b052dd4 in BehaviourManager::Update() () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #31 0x000000718b1f9224 in InitPlayerLoopCallbacks()::UpdateScriptRunBehaviourUpdateRegistrator::Forward() () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #32 0x000000718b1ef754 in ExecutePlayerLoop(NativePlayerLoopSystem*) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #33 0x000000718b1ef7ac in ExecutePlayerLoop(NativePlayerLoopSystem*) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #34 0x000000718b1efa10 in PlayerLoop() () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #35 0x000000718b4d1c58 in UnityPlayerLoop() () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #36 0x000000718b4fe390 in nativeRender(_JNIEnv*, _jobject*) () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/lib/arm64/libunity.so
    #37 0x00000071ea78b630 in art_jni_trampoline () from target:/data/app/com.DefaultCompany.krhook_unity3d-PL6MaRBI5vxhNhVP3URXtA==/oat/arm64/base.odex
    #38 0x000000009b94c5ec in com.unity3d.player.UnityPlayer.access$300 ()
    #39 0x000000009b94db00 in com.unity3d.player.UnityPlayer$e$1.handleMessage ()
    #40 0x000000009b94e2d0 in android.os.Handler.dispatchMessage ()
    #41 0x00000071f598c338 in art_quick_invoke_stub () from target:/apex/com.android.runtime/lib64/libart.so
    #42 0x00000071f599aff0 in art::ArtMethod::Invoke(art::Thread*, unsigned int*, unsigned int, art::JValue*, char const*) () from target:/apex/com.android.runtime/lib64/libart.so
    #43 0x00000071f5b3894c in art::interpreter::ArtInterpreterToCompiledCodeBridge(art::Thread*, art::ArtMethod*, art::ShadowFrame*, unsigned short, art::JValue*) () from target:/apex/com.android.runtime/lib64/libart.so
    #44 0x00000071f5b33bac in bool art::interpreter::DoCall(art::ArtMethod*, art::Thread*, art::ShadowFrame&, art::Instruction const*, unsigned short, art::JValue*) ()
    from target:/apex/com.android.runtime/lib64/libart.so
    #45 0x00000071f5df6138 in MterpInvokeVirtual () from target:/apex/com.android.runtime/lib64/libart.so
    #46 0x00000071f5986818 in mterp_op_invoke_virtual () from target:/apex/com.android.runtime/lib64/libart.so
    #47 0x00000071f5df8ea8 in MterpInvokeStatic () from target:/apex/com.android.runtime/lib64/libart.so
    #48 0x00000071f5986998 in mterp_op_invoke_static () from target:/apex/com.android.runtime/lib64/libart.so
    #49 0x00000071f5b09c60 in art::interpreter::Execute(art::Thread*, art::CodeItemDataAccessor const&, art::ShadowFrame&, art::JValue, bool, bool) [clone .llvm.12938883504528282530] ()
    from target:/apex/com.android.runtime/lib64/libart.so
    #50 0x00000071f5de76a0 in artQuickToInterpreterBridge () from target:/apex/com.android.runtime/lib64/libart.so
    #51 0x00000071f599546c in art_quick_to_interpreter_bridge () from target:/apex/com.android.runtime/lib64/libart.so
    #52 0x00000071f598c338 in art_quick_invoke_stub () from target:/apex/com.android.runtime/lib64/libart.so
    #53 0x00000071f599aff0 in art::ArtMethod::Invoke(art::Thread*, unsigned int*, unsigned int, art::JValue*, char const*) () from target:/apex/com.android.runtime/lib64/libart.so
    #54 0x00000071f5d06040 in art::(anonymous namespace)::InvokeWithArgArray(art::ScopedObjectAccessAlreadyRunnable const&, art::ArtMethod*, art::(anonymous namespace)::ArgArray*, art::JValue*, char const*) ()
    from target:/apex/com.android.runtime/lib64/libart.so 
    ```
    
    ![](https://bbs.pediy.com/upload/attach/202110/933020_S4D37D3KN5Q7W2B.jpg)
    

硬件断点内核工作原理
----------

通过阅读下面的几篇文章并结合源码我们就能够理解背后的工作原理了，然后再去看 rwProcMem33 的代码也就不会感觉特别困难了

 

![](https://bbs.pediy.com/upload/attach/202110/933020_CCDZ8XWHSNG988K.jpg)

 

https://www.kernel.org/doc/ols/2009/ols2009-pages-149-158.pdf

 

https://lwn.net/Articles/317153/

 

https://lwn.net/Articles/353050/

总结
--

通过上面的简单使用，我们已经简单了解如何通过 gdb 进行硬件断点的使用。 后续会从内核层讲讲 ptrace 的相关代码，逆向不易，互勉。

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#源码分析](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm)