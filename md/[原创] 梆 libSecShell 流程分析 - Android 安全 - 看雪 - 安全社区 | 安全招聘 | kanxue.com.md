> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283583.htm)

> [原创] 梆 libSecShell 流程分析

声明
==

本文仅限于技术讨论，不得用于非法途径，后果自负。

init_proc
=========

![](https://bbs.kanxue.com/upload/attach/202409/856431_M28DCTB2V3EEFAX.webp)  
在这个 so 中 init_proc 负责在内存中解密 so 中的 JNI_OnLoad，并且填充导入表，值得注意的是，如果你 hook 了 initProc 那么会导致 JNI_OnLoad 解密失败，这部分原因笔者不在细究，有兴趣的自行研究。

dump so
=======

由于初始 so 关键信息被抹除，我们需要在 init_proc 执行完之后 dump so 用作静态分析，这里采用 hook libc 的__dl__ZL10call_arrayIPFviPPcS1_EEvPKcPT_jbS5_（每台手机有可能不一样），在调用 initarray 的第一个函数时 dump。

修复 so
=====

这里使用的是 git 上 [https://github.com/Chenyangming9/SoFixer](https://github.com/Chenyangming9/SoFixer) 修复后发现导入表被 initProc 填充为绝对地址，需要还原符号。  
发现 ida 自动生成的 extern 表 这里直接拿到 ida 自动生成的表地址，通过 frida 的符号解析出 so 中的导入表中的每一个表指向的函数名称，进行比对然后将 dump 下来的 so 中的导入表填充 ida 的 extern 表 dump 脚本  
![](https://bbs.kanxue.com/upload/attach/202409/856431_9TU44CVN35QNMNP.webp)

```
dump() {
        if (this.libso == null) {
            return -1;
        }
 
        var file_path = this.path + "/" + this.soName;
        logd("dump so:" + this.soName + " to " + file_path);
        var file_handle = new File(file_path, "wb+");
        if (file_handle && file_handle != null) {
            Memory.protect(ptr(this.libso.base.toString()), this.libso.size, 'rwx');
            logd("libso_buffer:" + ptr(this.libso.base.toString()) + " " + this.libso.size);
            var libso_buffer = ptr(this.libso.base.toString()).readByteArray(this.libso.size);
        
            this.patchGot(libso_buffer!)
            var pGot = new BigInt64Array(libso_buffer!, 0x1352B8, 424)
             
            //创建extern 表
            var table = [{ key: 0x155000, value:"sleep"}, { key: 0x155008, value:"popen"}, { key: 0x155010, value:"mprotect"}, { key: 0x155018, value:"sigemptyset"}, { key: 0x155020, value:"lseek64"}, { key: 0x155028, value:"deflateEnd"}, { key: 0x155030, value:"pipe"}, { key: 0x155038, value:"atoi"}, { key: 0x155040, value:"pthread_create"}, { key: 0x155048, value:"wait"}, { key: 0x155050, value:"realloc"}, { key: 0x155058, value:"open"}, { key: 0x155060, value:"pthread_key_create"}, { key: 0x155068, value:"inflate"}, { key: 0x155070, value:"pthread_once"}, { key: 0x155078, value:"__cxa_finalize"}, { key: 0x155080, value:"ftell"}, { key: 0x155088, value:"ptrace"}, { key: 0x155090, value:"siglongjmp"}, { key: 0x155098, value:"mkdir"}, { key: 0x1550A0, value:"setpgid"}, { key: 0x1550A8, value:"calloc"}, { key: 0x1550B0, value:"fread"}, { key: 0x1550B8, value:"syslog"}, { key: 0x1550C0, value:"stpcpy"}, { key: 0x1550C8, value:"inflateInit2_"}, { key: 0x1550D0, value:"AAsset_getBuffer"}, { key: 0x1550D8, value:"strncmp"}, { key: 0x1550E0, value:"read"}, { key: 0x1550E8, value:"fstat"}, { key: 0x1550F0, value:"inotify_rm_watch"}, { key: 0x1550F8, value:"strncasecmp"}, { key: 0x155100, value:"AAsset_close"}, { key: 0x155108, value:"pthread_mutex_init"}, { key: 0x155110, value:"signal"}, { key: 0x155118, value:"abort"}, { key: 0x155120, value:"closedir"}, { key: 0x155128, value:"strerror"}, { key: 0x155130, value:"lstat"}, { key: 0x155138, value:"lstat64"}, { key: 0x155140, value:"_exit"}, { key: 0x155148, value:"__errno"}, { key: 0x155150, value:"srand"}, { key: 0x155158, value:"snprintf"}, { key: 0x155160, value:"getpid"}, { key: 0x155168, value:"dl_iterate_phdr"}, { key: 0x155170, value:"strcat"}, { key: 0x155178, value:"sscanf"}, { key: 0x155180, value:"android_set_abort_message"}, { key: 0x155188, value:"deflate"}, { key: 0x155190, value:"islower"}, { key: 0x155198, value:"isupper"}, { key: 0x1551A0, value:"write"}, { key: 0x1551A8, value:"toupper"}, { key: 0x1551B0, value:"getenv"}, { key: 0x1551B8, value:"strcasecmp"}, { key: 0x1551C0, value:"strrchr"}, { key: 0x1551C8, value:"access"}, { key: 0x1551D0, value:"time"}, { key: 0x1551D8, value:"rand"}, { key: 0x1551E0, value:"__sF"}, { key: 0x1551E8, value:"memcmp"}, { key: 0x1551F0, value:"fclose"}, { key: 0x1551F8, value:"lseek"}, { key: 0x155200, value:"fputs"}, { key: 0x155208, value:"rewind"}, { key: 0x155210, value:"fputc"}, { key: 0x155218, value:"__stack_chk_fail"}, { key: 0x155220, value:"fgets"}, { key: 0x155228, value:"select"}, { key: 0x155230, value:"fork"}, { key: 0x155238, value:"gettimeofday"}, { key: 0x155240, value:"dlclose"}, { key: 0x155248, value:"pthread_cond_wait"}, { key: 0x155250, value:"strftime"}, { key: 0x155258, value:"memchr"}, { key: 0x155260, value:"prctl"}, { key: 0x155268, value:"ioctl"}, { key: 0x155270, value:"strcasestr"}, { key: 0x155278, value:"pthread_setspecific"}, { key: 0x155280, value:"strncpy"}, { key: 0x155288, value:"opendir"}, { key: 0x155290, value:"dlsym"}, { key: 0x155298, value:"atol"}, { key: 0x1552A0, value:"openlog"}, { key: 0x1552A8, value:"__stack_chk_guard"}, { key: 0x1552B0, value:"environ"}, { key: 0x1552B8, value:"__android_log_print"}, { key: 0x1552C0, value:"inotify_init"}, { key: 0x1552C8, value:"unlink"}, { key: 0x1552D0, value:"inflateEnd"}, { key: 0x1552D8, value:"setenv"}, { key: 0x1552E0, value:"sysconf"}, { key: 0x1552E8, value:"strchr"}, { key: 0x1552F0, value:"tolower"}, { key: 0x1552F8, value:"fseek"}, { key: 0x155300, value:"strcmp"}, { key: 0x155308, value:"flock"}, { key: 0x155310, value:"fgetc"}, { key: 0x155318, value:"sprintf"}, { key: 0x155320, value:"strncat"}, { key: 0x155328, value:"sigaction"}, { key: 0x155330, value:"pthread_mutex_lock"}, { key: 0x155338, value:"mmap"}, { key: 0x155340, value:"setjmp"}, { key: 0x155348, value:"closelog"}, { key: 0x155350, value:"pthread_getspecific"}, { key: 0x155358, value:"AAssetManager_open"}, { key: 0x155360, value:"memmove"}, { key: 0x155368, value:"ferror"}, { key: 0x155370, value:"isxdigit"}, { key: 0x155378, value:"inotify_add_watch"}, { key: 0x155380, value:"AAsset_getLength"}, { key: 0x155388, value:"readlink"}, { key: 0x155390, value:"strstr"}, { key: 0x155398, value:"getpagesize"}, { key: 0x1553A0, value:"strdup"}, { key: 0x1553A8, value:"strtok"}, { key: 0x1553B0, value:"usleep"}, { key: 0x1553B8, value:"kill"}, { key: 0x1553C0, value:"readdir"}, { key: 0x1553C8, value:"fdopen"}, { key: 0x1553D0, value:"strlen"}, { key: 0x1553D8, value:"crc32"}, { key: 0x1553E0, value:"exit"}, { key: 0x1553E8, value:"close"}, { key: 0x1553F0, value:"vasprintf"}, { key: 0x1553F8, value:"remove"}, { key: 0x155400, value:"dlopen"}, { key: 0x155408, value:"stat"}, { key: 0x155410, value:"localtime"}, { key: 0x155418, value:"rename"}, { key: 0x155420, value:"munmap"}, { key: 0x155428, value:"get_crc_table"}, { key: 0x155430, value:"fprintf"}, { key: 0x155438, value:"malloc"}, { key: 0x155440, value:"memcpy"}, { key: 0x155448, value:"waitpid"}, { key: 0x155450, value:"deflateInit2_"}, { key: 0x155458, value:"connect"}, { key: 0x155460, value:"memset"}, { key: 0x155468, value:"fopen"}, { key: 0x155470, value:"AAssetManager_fromJava"}, { key: 0x155478, value:"socket"}, { key: 0x155480, value:"pthread_cond_broadcast"}, { key: 0x155488, value:"sigsetjmp"}, { key: 0x155490, value:"pclose"}, { key: 0x155498, value:"strtol"}, { key: 0x1554A0, value:"pthread_kill"}, { key: 0x1554A8, value:"free"}, { key: 0x1554B0, value:"fscanf"}, { key: 0x1554B8, value:"strcpy"}, { key: 0x1554C0, value:"__system_property_get"}, { key: 0x1554C8, value:"pwrite"}, { key: 0x1554D0, value:"pthread_exit"}, { key: 0x1554D8, value:"symlink"}, { key: 0x1554E0, value:"vfprintf"}, { key: 0x1554E8, value:"pthread_mutex_unlock"}, { key: 0x1554F0, value:"clock_gettime"}, { key: 0x1554F8, value:"__cxa_atexit"}, { key: 0x155500, value:"isspace"}]
            var base =this.libso.base
             
            for (var i = 0; i < pGot.length; i++) {
                var addr = pGot[i]
                var funcName = DebugSymbol.fromAddress(ptr(addr.toString())).toString().split("!")[1]
                logd("pgot1:" + i + " " + ptr(addr.toString()) + " " + funcName)
                table.forEach(function (item: any) {
                    var name = item["value"].toString()
                    if (funcName?.indexOf(name) !=-1 && funcName?.length > 0) {
                        pGot[i] = BigInt(ptr(item["key"]).toString())
                        logd("replace pgot:" + i + funcName + "  " + ptr(0x1352B8).add(i*8).add(base).readPointer()+" "+ ptr(pGot[i-1].toString()) +" "+ name + " to " + ptr(item["key"]))
                        return
                    }
                })
                if(ptr(pGot[i].toString()) > base){
                    pGot[i] = BigInt(ptr(pGot[i].toString()).sub(base).toString())
                }
 
            }
            logd(pGot.toString())
            logd("dump so:" + this.soName + " to " + file_path);
            file_handle.write(libso_buffer!);
            file_handle.flush();
            file_handle.close();
            log("[dump]:"+ file_path);
        }
    }
```

字符串解密
=====

程序使用的大部分字符串都被加密，加密方法为栈赋值密文，解密函数解密密文  
![](https://bbs.kanxue.com/upload/attach/202409/856431_2ZG2W2NTHXMF2D3.webp)

解密函数
----

```
__int64 __fastcall sub_181D4(__int64 result, int a2, char a3)
{
  int v3; // w5
  char v4; // w7
 
  v3 = 0;
  v4 = *(result + 1) ^ a3;
  while ( a2 > v3 )
  {
    *(result + v3) = v4 ^ *(result + v3 + 2);
    ++v3;
  }
  *(result + v3) = 0;
  return result;
}
```

间接跳转
====

为了干扰 ida 的引用分析，使用间接跳转  
![](https://bbs.kanxue.com/upload/tmp/856431_84JD7E7C5P8D7EK.webp)  
![](https://bbs.kanxue.com/upload/tmp/856431_TZUFR24PABUHEER.webp)

init_array
==========

以下将按程序顺序讲述程序执行功能

getAndroidVersion
-----------------

通过_system_property_get("ro.build.version.sdk"),_system_property_get("ro.build.version.release") 来获取 androdi 版本

getDalvikAndART
---------------

_system_property_get("ro.yunos.version"),_system_property_get("ro.yunos.version.release") 获取 java 虚拟机的类型

getLibcApi
----------

使用 dlopen 打开 libc dlsym 获取下列几个函数

```
mprotect
mmap
munmap
fopen
fclose
fgets
fwrite
fread
sprintf
pthread_create
```

其中 fopen，fclose，fgets，fwrite，fread，sprintf，pthread_create 被定义为了全局结构体，其余为全局函数指针因此，ida 定义结构结构体

```
00000000 struct libcPFuncAry // sizeof=0x38
00000000 {                                       // XREF: .data:g_func_fopen_ptr/r
00000000     FILE *(__fastcall *pFopen)(const char *filename, const char *modes);
00000000                                         // XREF: sub_158AC+30/r
00000000                                         // sub_15AE0+14/r ...
00000008     int (__fastcall *pFclose)(FILE *stream); // XREF: sub_E1F78/o
00000008                                         // sub_E1F78+4/r
00000010     char *(__fastcall *pFgets)(char *s, int n, FILE *stream);
00000010                                         // XREF: inotify_init_function+28/o
00000010                                         // inotify_init_function+AC/r
00000018     unsigned __int64 (__fastcall *pFwrite)(__int64 a1, unsigned __int64 a2, unsigned __int64 a3, __int64 a4);
00000018                                         // XREF: sub_1989C/o
00000018                                         // sub_1989C+C/r ...
00000020     unsigned __int64 (__fastcall *pFread)(char *a1, unsigned __int64 a2, unsigned __int64 a3, __int64 a4);
00000020                                         // XREF: .text&ARM.extab:0000000000037FE8/o
00000020                                         // .text&ARM.extab:0000000000037FF4/r
00000028     __int64 (*pSprintf)(__int64 a1, unsigned __int8 *a2, ...);
00000028                                         // XREF: struc_func_ptr_ctor+18/o
00000028                                         // struc_func_ptr_ctor+1C/r
00000030     __int64 (__fastcall *pPthread_create)(_QWORD *a1, __int64 a2, __int64 a3, __int64 a4);
00000030                                         // XREF: operator new(ulong)+44/o
00000030                                         // operator new(ulong)+4C/r ...
00000038 };
```

值得疑惑的是 这个函数还顺手通过 _system_property_get("ro.board.platform") 获取主板名称，对 "rk3399" 进行比对，结果存放全局变量  
![](https://bbs.kanxue.com/upload/tmp/856431_VNV3U7K5CQZB4AC.webp)  
![](https://bbs.kanxue.com/upload/tmp/856431_3X6CW6C3E9WBWGF.webp)

findDex2Oat
-----------

fopen("/proc/18607/cmdline","r")，比较 / system/bin/dex2oat, 如果找到返回，  
![](https://bbs.kanxue.com/upload/tmp/856431_M9XZYXC28FVQ48J.webp)  
##getLD_OPT_PACKAGENAME  
通过 getEnv 获取这两个字段 没有看懂在干嘛，不过条件没有满足，程序返回了  
![](https://bbs.kanxue.com/upload/tmp/856431_QHFXTX7SXC4TCX5.webp)

JNI_OnLoad
==========

这里 ida 没有识别为函数，我们需要手动添加地址, 从开头向下翻翻到

```
BL              __stack_chk_fail
```

就是结束地址了  
ida_funcs.add_func（startEa.endEa）

switch 识别
---------

由于大量使用了控制流混淆导致 ida 识别 switch 出现问题这里我们手动告诉 ida  
![](https://bbs.kanxue.com/upload/tmp/856431_UKTYA44U7S4ZRXX.webp)  
Edit =>Other=>Specify switch idiom  
![](https://bbs.kanxue.com/upload/tmp/856431_M42Q3PNKSSDRXXJ.webp)  
手动修复下

checkcpuabi
-----------

获取 cpuabi 到全局变量

get SoName
----------

获取 so 名称到全局变量

decodeStr
---------

解密 com_SecShell_SecShell_H

getLibcApi
----------

重新获取 Api

getPKGNAME
----------

获取 com_SecShell_SecShell_H 中的 PKGNAME 字段

解密数据
----

接下来一顿骚操作，我晕了，解密数据，略过  
![](https://bbs.kanxue.com/upload/tmp/856431_XSXSJMVQUZJ3QDR.webp)

反射调用
----

```
FindClass android/app/ActivityThread
GetStaticMethodID currentActivityThread
CallStaticObjectMethod currentActivityThread
GetMethodID getSystemContext
CallObjectMethod getSystemContext
FindClass  android/app/ContextImpl
GetMethodID getPackageManager
CallObjectMethod getPackageManager
GetMethodID getPackageInfo
NewStringUTF 包名
CallObjectMethod getPackageInfo
GetFieldID applicationInfo
GetObjectField  applicationInfo:Landroid/content/pm/ApplicationInfo
>GetObjectClass applicationInfo:Landroid/content/pm/ApplicationInfo
GetFieldID sourceDir
GetObjectField sourceDir
GetStringUTFChars sourceDir
GetFieldID dataDir
GetObjectField dataDir
GetStringUTFChars dataDir
GetFieldID nativeLibraryDir
GetObjectField nativeLibraryDir
GetStringUTFChars nativeLibraryDir
```

CheckPakeName
-------------

校验包名  
![](https://bbs.kanxue.com/upload/tmp/856431_MNGHFMY773ZE3F6.webp)

getMPAAS
--------

![](https://bbs.kanxue.com/upload/tmp/856431_7H55JX4K8KVXUA4.webp)

GetLibcArt
----------

打开 / proc/self/maps 获取 lib_libart 地址

解密字符串
-----

![](https://bbs.kanxue.com/upload/tmp/856431_HQ7KCX25GYEKZ6S.webp)  
![](https://bbs.kanxue.com/upload/tmp/856431_6KW2BZURDFFGGF6.webp)

RegisterNatives
---------------

注册 jni 函数  
![](https://bbs.kanxue.com/upload/tmp/856431_NHQSVKRS4YMFE6Z.webp)

loadDex
-------

拼接字符串  
![](https://bbs.kanxue.com/upload/tmp/856431_D8YSUGRGGSX2EFR.webp)

### hooklibcNewFunc

防 dexDump  
[https://bbs.kanxue.com/thread-223320.htm](https://bbs.kanxue.com/thread-223320.htm)  
![](https://bbs.kanxue.com/upload/tmp/856431_C5XA265S42FT3RB.webp)  
![](https://bbs.kanxue.com/upload/tmp/856431_MA3G3YPGYUQD4MB.webp)

### nopDex2Oat

![](https://bbs.kanxue.com/upload/tmp/856431_SKW2N2EMXCH6UWW.webp)

### loadDex

解密字符串  
![](https://bbs.kanxue.com/upload/tmp/856431_RA9GBM2VFH3K662.webp)  
调 javaapi  
![](https://bbs.kanxue.com/upload/tmp/856431_5P4B9N22CRDTJU4.webp)  
![](https://bbs.kanxue.com/upload/tmp/856431_A5WF6TMHQ8F53S2.webp)

### d.e

解密字符串

```
dalvik/system/DexPathList
akeInMemoryDexElements
([Ljava/nio/ByteBuffer;Ljava/util/List;)[Ldalvik/system/DexPathList$Element;
java/nio/ByteBuffer
wra
([B)Ljava/nio/ByteBuffer;
java/util/ArrayList
size
```

加载 dex 这里可以 dump 了  
![](https://bbs.kanxue.com/upload/tmp/856431_UPD4B7BCRUHWMY8.webp)

antiDebug
---------

### checkFridaTaskGdbus

打开 / proc/self/task /proc/self/task/6268/status 查找 frida 特征

### createAntiThread

![](https://bbs.kanxue.com/upload/tmp/856431_B9225AAE4G3QUHC.webp)

#### antiThread

![](https://bbs.kanxue.com/upload/tmp/856431_YZK6C9FAH9C2EJT.webp)  
#### 透明加密  
[https://bbs.kanxue.com/thread-226358.htm](https://bbs.kanxue.com/thread-226358.htm)  
![](https://bbs.kanxue.com/upload/tmp/856431_DJDP5YWGM3VRURN.webp)

CheckPakename
-------------

打开 / proc/self/cmdline 比较包名

结语
--

至此，加固分析结束，由于 trace 只会走流程其他分析流程可能走不到，但是控制流混淆太恶心了，告一段落了

[[课程]Android-CTF 解题方法汇总！](https://www.kanxue.com/book-section_list-181.htm)

最后于 16 小时前 被 method 编辑 ，原因： 修复图片

[#脱壳反混淆](forum-161-1-122.htm)