> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284166.htm)

> [原创]x 加密流程分析

声明
==

本文仅限于技术讨论，不得用于非法途径，后果自负。

入口
==

如果说 apk 的在启动的时候最早的运行的时候可能很多人会下意思的认为是 application 中 name 这个类中的 attachBaseContext，然而这个包很特别，如果你使用 frida hook 这个 name（s.h.e.l.l.S） 你会发现根本 hook 不到，但是你 hookdlopen 你会发现 libexec 确实加载了，所以一定有比这更早的时候，在 jadx 中搜索加载 libexec 的代码，打印堆栈 最后定位到了 application 中的 android:appComponentFactory 这个类中的 instantiateApplication，观察逻辑，如果这个 s.h.e.l.l.S 被加载，那么他的名称会被替换。  
![](https://bbs.kanxue.com/upload/attach/202410/856431_T3PBSY6VGGQHCJK.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_P8RPBKHTHV9PN4S.webp)  
继续观察剩下代码发现 classLoader 也被替换，那么 loader.al 就是重中之重  
![](https://bbs.kanxue.com/upload/attach/202410/856431_N6XQPSZ8PFG7WEN.webp)  
查看 a1 发现是 so 的函数  
![](https://bbs.kanxue.com/upload/attach/202410/856431_V5CZ565Q4AWYDX5.webp)  
可以看到 libexec 已经被加载了  
![](https://bbs.kanxue.com/upload/attach/202410/856431_REN7QRSSA66CHQJ.webp)

initproc
========

使用 ida 打开 apk 中的 libexec.so 发现被加密了，那么动态 dump，定位 initproc，简单看了下 大致是一个压缩壳，那么在 initarraydump 下来，sofix 修复  
![](https://bbs.kanxue.com/upload/attach/202410/856431_WJG7TSYCSEFM2YB.webp)

initarray
=========

查看 initarray 不得不说，函数是真的多 看了前半部分全部是解密字符串 那么定位到 initarray 最后一项, 重新 dump  
![](https://bbs.kanxue.com/upload/attach/202410/856431_XG4WANE9M4UCBNR.webp)  
定位后半部分

大致行为
----

获取了手机基本信息和 libc 中的 pthread_create ，还有就是反调试检查, 最重要的行为是填充了一个全局结构体，我叫做 jni_data(懒得改，实际上应该叫 IJM_Global)，这个结构体贯穿所有行为逻辑，考验结构体逆向的功底  
![](https://bbs.kanxue.com/upload/attach/202410/856431_TF533GE965TPRBD.webp)

jni_onload
==========

jni_onload 只做了 2 件事情

1. 获取 apk 全面信息
--------------

大致上获取了 apk 的路径 如 base.apk 的路径 files 路径等，其他根据版本获取了手机的能获取到的信息，所有需要的东西  
![](https://bbs.kanxue.com/upload/attach/202410/856431_T6WKE58MAS6V9TN.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_GYV7HTD7EP8TR86.webp)

### jni_fast_call

jni_data 引用了一组函数指针结构体的指针，用于快速调用 jni 函数  
![](https://bbs.kanxue.com/upload/attach/202410/856431_QD73JFKYRE876X7.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_9KFUQ6KFK4CXE8K.webp)

2.RegisterNatives
-----------------

![](https://bbs.kanxue.com/upload/attach/202410/856431_5FTX56CNFEGYM87.webp)

```
/* TID 28896 */
[+] JNIEnv->RegisterNatives
|- JNIEnv*          : 0x7ce6beb6c0
|- jclass           : 0x101
|- JNINativeMethod* : 0x7fff774e30
|: 0x7bf43d5fe4 - l(Landroid/app/Application;Ljava/lang/String;)Z
|: 0x7bf43d7d54 - r(Landroid/app/Application;Ljava/lang/String;)Z
|: 0x7bf43d840c - ra(Landroid/app/Application;Ljava/lang/String;)Z
|: 0x7bf43d85bc - b2b([BI)[B
|: 0x7bf43d85c4 - m(Ljava/lang/String;I)V
|: 0x7bf43d85c8 - sa(Ljava/lang/String;Ljava/lang/String;)V
|: 0x7bf43d864c - al(Ljava/lang/ClassLoader;Landroid/content/pm/ApplicationInfo;Ljava/lang/String;Ljava/lang/String;)Ljava/lang/ClassLoader;
|- jint             : 7
|= jint             : 0
0x7bf4383000
```

a1
==

继续向下看 观察 a1 这就应该就是加载 dex 的函数了 返回 classload。  
![](https://bbs.kanxue.com/upload/attach/202410/856431_PVZN34H75457R6J.webp)

下面根据行为大致列出流程
------------

![](https://bbs.kanxue.com/upload/attach/202410/856431_C6MMJJWZUJ9NJMB.webp)  
检查 initarray 设下的反调试标志  
![](https://bbs.kanxue.com/upload/attach/202410/856431_S5H94DKDJU8JZ86.webp)  
兼容大客户 不过这个不是，有兴趣的去学习学习  
![](https://bbs.kanxue.com/upload/attach/202410/856431_SHS8FRF9V6JYNAT.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_F57PU56R3GW25TM.webp)  
值得注意的是自实现了一组加载解密函数  
![](https://bbs.kanxue.com/upload/attach/202410/856431_Q482867HEBMT3QX.webp)  
初始化 ijm 的功能强度配置 通过文件来设置  
![](https://bbs.kanxue.com/upload/attach/202410/856431_2NSSTS4RZ2HCSKK.webp)  
重定向输出流 ，也就是干掉 log 不过缺陷是 ijm 内部加载文件 log 会在我的 frida 打印搞不懂  
![](https://bbs.kanxue.com/upload/attach/202410/856431_VE7JS9YZ3C8X9HY.webp)  
计算出包名的 md5 这个 md5 会参与后面的文件解密  
![](https://bbs.kanxue.com/upload/attach/202410/856431_FXD9GBEGSQFVN9K.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_XCN89EQGDBTJQKK.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_8H3MZK2Z8XTAKYV.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_PQ8UTG42PSDZT8Z.webp)  
打开文件解密出包名校验  
![](https://bbs.kanxue.com/upload/attach/202410/856431_RK4Y7AJK9DYN2VW.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_P8SANVKNV4BKVHU.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_788DZYQC7S3QEAK.webp)  
创建 2 个线程 一个签名校验，一个校验资源 androidmanifest  
![](https://bbs.kanxue.com/upload/attach/202410/856431_GBAW4XZF3BFRT2Y.webp)  
jni 检查 isdebug  
![](https://bbs.kanxue.com/upload/attach/202410/856431_S5GQPU98RBPESVT.webp)  
检查嫌疑机型  
![](https://bbs.kanxue.com/upload/attach/202410/856431_5GVU685WBWN9Q3D.webp)  
检查虚拟机 su root  
![](https://bbs.kanxue.com/upload/attach/202410/856431_9JSTUQ4GHP4H44Q.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_JNUJNKVZVGQKK9C.webp)  
检查了看雪上已知的脱壳系统 fart youpk 还有几个 super 功能没开 应该是得加钱

```
__int64 antiFartYoupkAndAntiFrida()
{
  Jni_Data **v0; // x21
  jclass v1; // x19
  unsigned int v2; // w19
  __int64 v4; // x19
  Jni_Data *v5; // x8
  __int64 v6; // x19
  void *v7; // x0
  Jni_Data *v8; // x8
  __int64 v9; // [xsp+8h] [xbp-B8h] BYREF
  _BYTE v10[8]; // [xsp+10h] [xbp-B0h] BYREF
  char v11[128]; // [xsp+18h] [xbp-A8h] BYREF
  __int64 v12; // [xsp+98h] [xbp-28h]
 
  v12 = *(_QWORD *)(_ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2)) + 40);
  if ( (unsigned int)faccessat("/data/dexname") != -1 )
  {
    return 1;
  }
  v0 = ppjni_data;
  v1 = (*(*ppjni_data)->Env)->FindClass((*ppjni_data)->Env, "cn/youlor/Unpacker");
  (*v0)->jni_func_D5D58->checkException((*v0)->Env);
  if ( v1 )
  {
    return 1;
  }
  if ( (*v0)->androidVersion < 24
    || (v2 = 1,
        !((__int64 (__fastcall *)(const char *, const char *, __int64))(*v0)->hookfunc_D59F8->dlsym)(
           "libart.so",
           "_ZN3art8Unpacker12dumpAllDexesEv",
           1LL))
    && (v2 = 1,
        !((__int64 (__fastcall *)(const char *, const char *, __int64))(*v0)->hookfunc_D59F8->dlsym)(
           "libart.so",
           "_ZN3art4Aupk13aupkArtMethodE",
           1LL)) )
  {
    if ( (unsigned int)faccessat("/data/local/tmp/unpacker.config") != -1 )
    {
      return 1;
    }
    if ( (unsigned int)faccessat("/data/local/tmp/aupk.config") != -1 )
    {
      return 1;
    }
    if ( (unsigned int)faccessat("/data/fart") != -1 )
    {
      return 1;
    }
    v4 = (__int64)(*(*v0)->Env)->FindClass((*v0)->Env, "cn/mik/Fartext");
    (*v0)->jni_func_D5D58->checkException((*v0)->Env);
    if ( v4 )
    {
      return 1;
    }
    if ( (unsigned int)faccessat("/data/system/mik.conf") != -1 )
    {
      return 1;
    }
    v5 = *v0;
    v9 = 0LL;
    v6 = (__int64)(*v5->Env)->NewStringUTF(v5->Env, "mikrom");
    (*v0)->jni_func_D5D58->CallStaticObjectMethodV(
      (*v0)->Env,
      &v9,
      "android/os/ServiceManager",
      aLjava,
      "getService",
      v6);
    (*(*v0)->Env)->DeleteLocalRef((*v0)->Env, (jobject)v6);
    if ( v9 || !strstr(aData, "re.frida.server") )
    {
      return 1;
    }
    if ( (*v0)->androidVersion < 24 )
    {
      v7 = dlopen("libart.so", 0);
      if ( v7 && dlsym(v7, "myfartInvoke") )
      {
        return 1;
      }
    }
    else if ( ((__int64 (__fastcall *)(const char *, const char *, _QWORD))(*v0)->hookfunc_D59F8->dlsym)(
                "libart.so",
                "myfartInvoke",
                0LL) )
    {
      return 1;
    }
    v8 = *v0;
    if ( !(*v0)->isPakeName )
    {
LABEL_25:
      (*(void (__fastcall **)(_BYTE *, _QWORD, _DWORD *(*)(), _QWORD))v8->pthread_create)(v10, 0LL, sub_43D80, 0LL);
      return 0;
    }
    memset(v11, 0, sizeof(v11));
    if ( (int)_system_property_get_1((__int64)"ro.dalvik.vm.native.bridge", (__int64)v11) < 1
      || !strcmp_1(v11, "libriruloader.so") && (unsigned int)faccessat("/system/lib/libriruloader.so") == -1 )
    {
      (*(void (__fastcall **)(_BYTE *, _QWORD, void (__fastcall __noreturn *)(__int64), _QWORD))(*v0)->pthread_create)(
        v10,
        0LL,
        antiFrida,
        0LL);
      v8 = *v0;
      goto LABEL_25;
    }
    return 1;
  }
  return v2;
}
```

![](https://bbs.kanxue.com/upload/attach/202410/856431_8FB56A584AGE395.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_MJ38MFCHJ5GMDNU.webp)  
这里自实现了一组 dlsym hook 了抽取壳需要的 api，如果实在找不到关键 api 甚至会对特定机型进行硬编码  
![](https://bbs.kanxue.com/upload/attach/202410/856431_DG8TYJS468N8E3P.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_EZU8YYBP62KAFPP.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_KF77HPK8D2NTDK9.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_PJX68QPR6953UFB.webp)  
这里这里 dex 被解密（需要用到之前的 md5）并加载进 classload 但是这个是个抽取壳  
![](https://bbs.kanxue.com/upload/attach/202410/856431_X23T4WM3RTDJUMU.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_ESG9XG7AWDVXW44.webp)  
![](https://bbs.kanxue.com/upload/attach/202410/856431_4K2ZG8UESNG8ZT7.webp)  
大致行为就这样

对抗
==

1. 壳并没有使用控制流混淆，但是大量使用了不透明谓词去干扰 还有虚假控制流，这个需要耐心对汇编有一定功底要求  
2.ida 静态干扰，ida 函数识别是错的需要你自己去重新定义 同样需要 nop 汇编  
3. 我不知道这个有没有对抗 但是 sofix 修复的部分重定位表 是有问题的 会导致 ida 解析函数调用的时候忽略参数  
4. 如果你使用 frida 打印堆栈 或者 DebugSymbol.fromAddress libc 的锁会崩溃

回填
==

这个需要你耐心读一遍 android 源码 弄明白 loadmethod 这个函数的参数的结构 ，和弄明白 new_loadmethod 的行为 dump 数据修复

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm)