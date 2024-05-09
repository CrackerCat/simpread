> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281132.htm)

> [原创]APP 加固系统分析

APP 加固系统分析
==========

本分析是四年前的，时过境迁，应该没啥价值了。只当纪念那个曾经的疯魔时光！

    APP 加固系统也算是一个完整的 APP 加固保护系统，基本功能应该都具备，基本上也是以对 dex 的文件和代码保护为立足点，这个也是 Android APP 加固保护的基本思路。下面从三个方面来讲解他的保护系统架构和功能。

本次使用 * 翻译 APP 作为分析的样本，应该是专业版加固系统。

    本分析还是基本上以 AndroidNativeEmU 模拟器分析日志为主。不过为了保证函数可以正常运行，部分数据来自真实环境，并还原到原偏移地址中，保证模拟器正常执行。本次分析还首次在模拟器中实现 art 模式下的 Dex 加载过程，不用到真实环境中去 dump dex 了。

第一、      对自身模块的保护：

    APP 加固系统都自身模块的保护并没有像其他加固保护系统那么强大和复杂，就是使用了 code 段加密，然后中 so 的 init_array 中进行解密，然后抹除 so 头，防止内存 dump 的方式。到 JNI_OnLoad 的时候代码段已经解密完成，这个时候 dump 下代码，然后把原始的头还原回去基本上就能用来分析了，如果要用模拟器调试，还得 patch 掉解密和抹除 so 头的代码。

![](https://bbs.kanxue.com/upload/attach/202403/981_7CBZWV2FTZYAJMJ.webp)

下面来看具体的代码实现。

Calling Init for: samples/appjiagu/libappprotect-up.so

Calling Init function: cbff8d85// 这个就是 init_array 中的第一个函数，解密函数。

把 so 头 0x1000 字节清零：

Executing syscall mprotect(cbfab000, 00001000, 00000003) at 0xcbffd165

call mprotect:0xcbfab000  ç 基地址，即 so 头地址

======================= Registers =======================

  R0=0xcbfab034  R1=0x0  R2=0x1  R3=0x1

R4=0x7ffce8  R5=0x7ffce8  R6=0x7ffe90  R7=0x7ffff8  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0x7ffce0  SP=0x7ffce0

LR=0xcbffd165  PC=0xcbffd60c

======================= Disassembly =====================

0xcbffd60c:    0170   strb   r1, [r0]   ç 开始把 so 头清零

0xcbffd60e:    6e48   ldr    r0, [pc, #0x1b8]

0xcbffd610:    7f49   ldr    r1, [pc, #0x1fc]

0xcbffd612:    7944   add    r1, pc

0xcbffd614:    4018   adds   r0, r0, r1

解密代码段中加密的代码：

Executing syscall mprotect(cbfae6f1, 00042C22, 00000007) at 0xcbffab7f

![](https://bbs.kanxue.com/upload/attach/202403/981_ZZ5W2PBU963W73V.webp)

解密代码段中加密的代码：

Executing syscall mprotect(cbfae6f1, 00042C22, 00000007) at 0xcbffab7f

![](https://bbs.kanxue.com/upload/attach/202403/981_XKUQ5WXAWRP8M4N.webp)

到这里：

![](https://bbs.kanxue.com/upload/attach/202403/981_3W9866H3569TAVU.webp)

都是被加密的代码。

======================= Registers =======================

  R0=0x47  R1=0xcbfae6f1  R2=0x7ffe60  R3=0x7ffe58

R4=0x7ffcf8  R5=0x7ffdd8  R6=0x7ffe90  R7=0x7ffff8  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xffff1c30  SP=0x7ffce0

LR=0xcbffe2f3  PC=0xcbffca94

======================= Disassembly =====================

0xcbffca94:    0870   strb   r0, [r1] < --R1=0xcbfae6f1

0xcbffca96:    1878   ldrb   r0, [r3]

0xcbffca98:    1168   ldr    r1, [r2]

0xcbffca9a:    0870   strb   r0, [r1]

0xcbffca9c:    1020   movs   r0, #0x10

这个函数执行完就可以 dump so，然后修复代码了：

JNI_OnLoad：代码已经解密出来了：

0xcbfae9c4:    f0b5   push   {r4, r5, r6, r7, lr}

0xcbfae9c6:    03af   add    r7, sp, #0xc

0xcbfae9c8:    89b0   sub    sp, #0x24

0xcbfae9ca:    6e46   mov    r6, sp

0xcbfae9cc:    3162   str    r1, [r6, #0x20]

>dump 0xcbfab000

CBFAB000: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

CBFAB010: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

CBFAB020: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

CBFAB030: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

可以看到 so 头被清除了，

![](https://bbs.kanxue.com/upload/attach/202403/981_26HVNPAXKR9JNBA.webp)

修复 so 方法：

先 dump 数据，然后用 010Editor 把原始的头贴回去。

![](https://bbs.kanxue.com/upload/attach/202403/981_769Y9WA8TK5M3BB.webp)

由于代码已经被解出来，所以后面不能再被解密了，patch 代码，第一个就是把清除 so 头的代码除掉。然后把解密代码的写入操作代码也 nop 掉。这样就保证了可以二次加载这个 so 进行运行分析。

0xcbffd60c:    0170   strb   r1, [r0]   ç 开始把 so 头清零   nop 掉代码

0xcbffca94:    0870   strb   r0, [r1]    < --R1=0xcbfae6f1   nop 掉

0xcbffca96:    1878   ldrb   r0, [r3]

0xcbffca98:    1168   ldr    r1, [r2]

0xcbffca9a:    0870   strb   r0, [r1]     nop 掉代码

0xcbffca9c:    1020   movs   r0, #0x10

从分析来看，APP 加固对自身模块的保护力度不是很强，修复起来也比较容易。

JNI_OnLoad 函数主要就是把 libc 中的这些函数填到函数列表中，然后通过列表的索引进行调用。这样防止分析代码：

Called dlopen(libc.so)

Loading module 'vfs/system/lib/libc.so'.

call malllos size:0x9 at 0xcbfc15e7

malloc addr:0x2022000

Called dlsym(0xcbbdf000, _exit) at 0xcbfb1669

symbol:_exit addr->: 0xcbc2780c

call malllos size:0x9 at 0xcbfc166f

malloc addr:0x2023000

Called dlsym(0xcbbdf000, exit) at 0xcbfb167f

导入的函数有：

#libc_fun_name = ["_exit","exit","pthread_create","pthread_join","memcpy","malloc","calloc","memset","fopen","fclose","fgets","strtoul","strtoull","strstr","ptrace","mprotect","strlen","sscanf","free","strdup","strcmp","strcasecmp","utime","mkdir","open","close","unlink","stat64","time","snprintf","strchr","strncmp","pthread_detach","pthread_self","opendir","readdir","closedir","mmap","munmap","lseek","fstat","read","select","bsd_signal","fork","prctl","setrlimit","getppid","getpid","waitpid","kill","flock","write","execve","execv","execl","sysconf","__system_property_get","ftruncate","gettid","pread64","pwrite64","pread","pwrite","","statvfs"]

后面就是注册加固系统 com.app.protect.A 主功能函数：

JNIEnv->FindClass(com/app/protect/A) was called

JNIEnv->RegisterNatives(1, 0x007fff58, 4) was called

Register native ('n001', '(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;IZ)V',function->'0xcbfaf829') failed on class com_app_protect_A.

Register native ('n002', '(Landroid/content/Context;)V',function->'0xcbfaf999') failed on class com_app_protect_A.

Register native ('n003', '()[Ljava/lang/String;',function->'0xcbfaf9f9') failed on class com_app_protect_A.

Register native ('n004', '()V',function->'0xcbfafad5') failed on class com_app_protect_A.

这个版本的加固中，注册了四个函数。主要的功能在'n001'这个函数中。包括 dex 解压，解密，加载，附加数据的处理。Dex_VMP 数据初始化。

另外 Native so 中大量使用字符串加密，防止关键字符串暴露，也是 APP 加固的技术特点：

![](https://bbs.kanxue.com/upload/attach/202403/981_RGF5SP2VSZENZTA.webp)

第一、     

第二、      对 dex 文件的保护：

Android APP 加固系统保护的重点就是 dex，所以对 dex 文件的保护是比较重要的。APP 加固系统这块也做的比较好，甚至全程没有 dex 明文文件落地，都是从 apk 包中在运行时解压解密加载。具体我们来看看流程：

APP 加固的 JNI_OnLoad 中注册了四个 Native 函数，其中 n001 函数就是处理 dex 解压，解密，加载和 vmp 数据的。

    protected void attachBaseContext(Context arg8) {

        StubApplication.mContext = arg8;

        StubApplication.mBootStrapApplication = this;

        AppInfo.APKPATH = arg8.getApplicationInfo().sourceDir;

        AppInfo.DATAPATH = StubApplication.getDataFolder(arg8.getApplicationInfo().dataDir);

        if(!Debug.isDebuggerConnected()) {

            StubApplication.loadLibrary();

            A.n001(AppInfo.PKGNAME, AppInfo.APPNAME, AppInfo.APKPATH, AppInfo.DATAPATH, Build.VERSION.SDK_INT, AppInfo.REPORT_CRASH);

        }

        if(AppInfo.APPNAME != null && AppInfo.APPNAME.length() > 0) {

            StubApplication.mRealApplication = MonkeyPatcher.createRealApplication(AppInfo.APPNAME);

        }

        super.attachBaseContext(arg8);

        if(StubApplication.mRealApplication != null) {

            MonkeyPatcher.attachBaseContext(arg8, StubApplication.mRealApplication);

        }

}

Java 层的入口，可以看出来，首先对调试状态进行了检测，如果是调试状态就不会加载 so 模块，也不会执行 dex 的解密函数 A.n001. 防止 APP 被调试。如果没有被调试则加载完 libappprotect.so 后进入 dex 的加载函数，即 n001 函数。从上面我们知道 JNI_OnLoad 中注册了这个函数，函数地址是：function->'0xcbfaf829'，我们就看看这个 Native 函数的具体功能：

首先获取 APP 的包信息，从包信息中获取 signatures，从 apk 包的 META-INF 目录读取签名文件 TRANSLAT.RSA，然后从 0x3A 开始读两个字节的签名数据长度，根据长度读取后面的签名数据。比如这个 APP 的签名长度是 0x235

读取数据如下：

00000040: 30 82 02 31 30 82 01 9A  A0 03 02 01 02 02 04 4E  0..10..........N

00000050: 25 42 6B 30 0D 06 09 2A  86 48 86 F7 0D 01 01 05  %Bk0...*.H......

00000060: 05 00 30 5D 31 0B 30 09  06 03 55 04 06 13 02 43  ..0]1.0...U....C

00000070: 4E 31 10 30 0E 06 03 55  04 08 13 07 62 65 69 6A  N1.0...U....beij

然后对这个签名文件取 hash：

.text:CBFB71CE 28 4D                       LDR     R5, =(_GLOBAL_OFFSET_TABLE_ - 0xCBFB71D4)

.text:CBFB71D0 7D 44                       ADD     R5, PC          ; _GLOBAL_OFFSET_TABLE_

.text:CBFB71D2 44 19                       ADDS    R4, R0, R5      ; byte_CC00F664

.text:CBFB71D4 20 46                       MOV     R0, R4

.text:CBFB71D6 39 46                       MOV     R1, R7

.text:CBFB71D8 1B F0 E8 FA                 BL      Fun_md5         ; 获取 apk 数字签名，然后 MD5 这个签名，给后面的 dex decode 做 key

2020-11-11 17:39:40,906   DEBUG            androidemu.native.hooks | call memcpy (len:0x10)

2020-11-11 17:39:40,906   DEBUG            androidemu.native.hooks | memcpy_data:0xcc00f664-->0x20cdb11

2020-11-11 17:39:40,907    INFO            androidemu.native.hooks | addr -->0xcbfb6f9b

2020-11-11 17:39:40,907   DEBUG            androidemu.native.hooks |

CC00F664: 05 86 74 2E 88 A2 E6 A1  9E 99 65 98 EC 33 6B 61  ..t.......e..3ka   <== md5

这个 hash 值用于后面 dex 前 0x1000 字节的解密 key。这样设计的目的是在用防止 APP 被二次打包，打包后的签名已经改变，改变后的这个签名 hash 值也会改变，然后就不能再解密出正确的 dex 文件。对 dex 的保护起到了一定的作用，如果在不能完整恢复原 dex 代码的情况下，加固保护系统就能起作用。

后面就是从 apk 包中解压 assets 目录下所有的 jar 文件。这个 jar 文件就是 dex 的密文，也就是前 0x1000 个字节没有被解密的 dex 文件。

2020-11-16 14:23:04,042    INFO            androidemu.native.hooks | string-->'assets/appprotect1.jar'

2020-11-16 14:23:04,045   DEBUG            androidemu.native.hooks | call memcmp;data1->0x21435d9,data2->0x7ff8d8,len->24

2020-11-16 14:23:04,045   DEBUG            androidemu.native.hooks | mem1_data：

2020-11-16 14:23:04,045   DEBUG            androidemu.native.hooks |

021435D9: 61 73 73 65 74 73 2F 62  61 69 64 75 70 72 6F 74  assets/appprot

021435E9: 65 63 74 31 2E 6A 61 72                           ect1.jar

2020-11-19 11:23:30,452   DEBUG           androidemu.native.memory | call malllos size:0x7e62dc at 0xcbfd291b

2020-11-19 11:23:30,455    INFO           androidemu.native.memory | malloc addr:0x21db000  ß 申请到的空间，后面存放解压数据。size:0x7e62dc 是 appprotect1.jar 的大小。

读取这个数据后使用 libart 模块中的 zlib 函数进行解压：

if (!j_inflateInit2_(&v19, 0xFFFFFFF1, "1.2.3", 0x38) )

  {

    if (j_inflate(&v19, 4) == 1 )

    {

      v10 = v23;

      j_inflateEnd(&v19);

      if (v10 == v9)

      {

LABEL_13:

        v6 = 1;

        if (*(_DWORD *)&v16 >= 0x8001u )

          sub_CBFB11A0(v8, 0);

        goto LABEL_16;

      }

    }

    else

    {

      j_inflateEnd(&v19);

    }

  }

![](https://bbs.kanxue.com/upload/attach/202403/981_UBZVP88MF7J33NH.webp)

这个时候的 dex 结构如上图所示，下面开始解这些部分。首先用签名的 hash 生成的 key 解密前 0x1000 字节：

![](https://bbs.kanxue.com/upload/attach/202403/981_39RG8MU7UM5DQYY.webp)

这个解压出来的数据前 0x1000 个字节是密文的，后面需要用 APP 签名的 hash 值来解密：

解密算法：

======================= Registers =======================

  R0=0x21db000  R1=0x7ff758  R2=0x20cdc3c  R3=0xcbff04f9

R4=0x21db000  R5=0x7ff7b0  R6=0x1000  R7=0xcbff04f9  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00eef8  SP=0x7ff720

LR=0xcbff0e11  PC=0xcbff0ca8

======================= Disassembly =====================

0xcbff0ca8: b847   blx    r7    <= 解密函数

0xcbff0caa: 3b46   mov    r3, r7

0xcbff0cac: 2868   ldr    r0, [r5]

0xcbff0cae: 029a   ldr    r2, [sp, #8]

0xcbff0cb0: 1168   ldr    r1, [r2]

>dump 0x20cdc3c     // APP 签名 hash 生成的解密 key

020CDC3C: 4D 23 BC 03 9D 27 9A 95  B2 85 D7 ED 76 C3 3D BA  M#...'......v.=.

020CDC4C: 10 D7 1A A0 C0 A0 FE FA  1D E9 14 58 9D C1 CD AE  ...........X....

020CDC5C: 52 4B 9B 2D D0 77 E4 5A  DD 49 EA A2 80 28 D9 F6  RK.-.w.Z.I...(..

020CDC6C: 7C 0A 2B 97 82 3C 7F 77  0D 3E 0E F8 5D 61 33 54  |.+..<.w.>..]a3T

>dump 0x21db000  // dex 密文数据

021DB000: E3 7A E6 20 86 8C 4B 89  F8 74 A9 AF 65 5B 06 78  .z. ..K..t..e[.x

021DB010: 69 DD 9E C2 C2 68 85 1E  A5 A7 35 E0 88 96 E4 4F  i....h....5....O

021DB020: 38 16 B1 B7 84 AB 0A E8  7F E1 5D 3C 74 A6 24 FF  8.........]<t.$.

021DB030: 82 CB 1D 61 60 AF 48 8C  9F 50 11 BF C9 72 98 2E  ...a`.H..P...r..

到这里全部解出来了：

======================= Registers =======================

  R0=0x76e69e89  R1=0x195a3f  R2=0x7ff758  R3=0xcbff04f9

R4=0x21dc000  R5=0x7ff7b0  R6=0x0  R7=0xcbff04f9  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00eef8  SP=0x7ff720

LR=0xcbff0cab  PC=0xcbff0ce6

======================= Disassembly =====================

0xcbff0ce6: 0b98   ldr    r0, [sp, #0x2c]

0xcbff0ce8: 0a99   ldr    r1, [sp, #0x28]

![](https://bbs.kanxue.com/upload/attach/202403/981_2CS4DMYPTAMEZYH.webp)

这个时候的 dex 有部分代码被移走了，需要重新解密填充回来：

![](https://bbs.kanxue.com/upload/attach/202403/981_ASW8DXCSYEAP8AY.webp)

下面的函数就是解密填充:

======================= Registers =======================

  R0=0x21db000  R1=0x7e62dc  R2=0x0  R3=0xcbff04f9

R4=0x0  R5=0x21db000  R6=0x7ff804  R7=0xcbc3064d  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00eef8  SP=0x7ff7d8

LR=0xcbff0cab  PC=0xcbfd2954

======================= Disassembly =====================

0xcbfd2954: e8f79efc bl     #0xcbfbb294  // 解密并填充回被移动走的代码的函数。

2020-11-19 15:40:58,411   DEBUG            androidemu.native.hooks | call memcpy (len:0x739dc)

2020-11-19 15:40:58,411   DEBUG            androidemu.native.hooks | memcpy_data:0x2944514-->0x28d06ac

2020-11-19 15:40:58,412    INFO            androidemu.native.hooks | addr -->0xcbfc1003

2020-11-19 15:40:58,412   DEBUG            androidemu.native.hooks |

密文数据：

02944514: 33 A9 E7 6F C3 9B 4C DD  F2 82 6E 56 D5 0D 40 91  3..o..L...nV..@.

02944524: C3 FA A6 2C 82 F1 5A 6A  57 24 C8 F3 68 92 4C A0  ...,..ZjW$..h.L.

02944534: 93 0B C4 13 AF DC 0A A0  B0 FE 26 36 89 D2 1E F1  ..........&6....

02944544: C2 D9 97 1D 30 43 95 04  2B 5B B7 8F 0D 56 9A 75  ....0C..+[...V.u

02944554: 7B 63 AC 1B F4 D8 1C 8E  A1 D1 78 1B 8B D3 23 CC  {c........x...#.

02944564: AA 69 35 BF 11 61 AA 4F  72 01 ED D5 0E 3B E5 09  .i5..a.Or....;..

这个数据的解密也是用到 APP 签名的 hash 生成的 key 来解密。所以如果签名改变 dex 就会解密失败。

![](https://bbs.kanxue.com/upload/attach/202403/981_WTR5J7HUW4Z79MM.webp)

                                【附加数据解密并填充被移走的 dex 数据】

到这里 dex 的解压和解密完成，可以 dump 下来了。

下面是 dex 加载过程：

用 InMemoryDexClassLoader 类调用 NewObjectV 加载 dex，InMemoryDexClassLoader 内部会使用 mmap 分配内存存放 dex，分配一个新的 DexPathList$Element 数组，将原来系统的类加载器和刚才的 InMemoryDexClassLoader 中的 classLoader.pathList.dexElements 合并成一个数组，然后替换原来系统中的类加载器的 dexElements。

参考资料：https://blog.csdn.net/xingzhong128/article/details/80470796

JNIEnv->NewByteArray(8282844) was called

JNIEnv->SetByteArrayRegion was called

JNIEnv->CallStaticObjectMethodV(java/nio/ByteBuffer, wrap <([B)Ljava/nio/ByteBuffer;>, 0x7ff984) was called

NIEnv->NewObjectV(dalvik/system/InMemoryDexClassLoader, <init>, 0x7ff984) was called

JNIEnv->FindClass(dalvik/system/BaseDexClassLoader) was called

JNIEnv->ExceptionCheck() was called

JNIEnv->FindClass(dalvik/system/DexPathList) was called

JNIEnv->GetFieldId(29 [dalvik/system/BaseDexClassLoader], pathList, Ldalvik/system/DexPathList;) was called

JNIEnv->GetFieldId(30 [dalvik/system/DexPathList], dexElements, [Ldalvik/system/DexPathList$Element;) was called

JNIEnv->FindClass(dalvik/system/DexPathList$Element) was called

JNIEnv->GetObjectField(dalvik/system/BaseDexClassLoader, pathList <Ldalvik/system/DexPathList;>) was called

JNIEnv->GetObjectField(dalvik/system/DexPathList, dexElements <[Ldalvik/system/DexPathList$Element;>) was called

JNIEnv->GetObjectField(dalvik/system/BaseDexClassLoader, pathList <Ldalvik/system/DexPathList;>) was called

JNIEnv->GetObjectField(dalvik/system/DexPathList, dexElements <[Ldalvik/system/DexPathList$Element;>) was called

JNIEnv->GetArrayLength(33) was called

JNIEnv->GetArrayLength(35) was called

JNIEnv->newobjectarray(2, 31) was called

JNIEnv->GetObjectArrayElement(33, 0) was called

JNIEnv->SetObjectArrayElement(0) was called

JNIEnv->GetObjectArrayElement(35, 0) was called

JNIEnv->SetObjectArrayElement(1) was called

这样刚才解压解密还原的 dex 被加载进内存，而整个过程中没有明文文件落地，避免被 copy 出来。

.text:CBFB56A6 7C 69                       LDR     R4, [R7,#0x74+var_60]  dex 个数计数器

.text:CBFB56A8 1E F0 DC FB                 BL      sub_CBFD3E64   // 获取总个数

.text:CBFB56AC 00 F0 DA F8                 BL      sub_CBFB5864

.text:CBFB56B0 84 42                       CMP     R4, R0   // 循环解压解密加载所有的 dex

.text:CBFB56B2 44 DA                       BGE     loc_CBFB573E

.text:CBFB56B4 FF E7                       B       loc_CBFB56B6

加载完成后注册 dex vmp 函数：

JNIEnv->FindClass(com/app/protect/A) was called

JNIEnv->RegisterNatives(57, 0x007ff984, 10) was called

Register native ('V', '(ILjava/lang/Object;[Ljava/lang/Object;)V',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('Z', '(ILjava/lang/Object;[Ljava/lang/Object;)Z',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('B', '(ILjava/lang/Object;[Ljava/lang/Object;)B',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('C', '(ILjava/lang/Object;[Ljava/lang/Object;)C',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('S', '(ILjava/lang/Object;[Ljava/lang/Object;)S',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('I', '(ILjava/lang/Object;[Ljava/lang/Object;)I',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('J', '(ILjava/lang/Object;[Ljava/lang/Object;)J',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('F', '(ILjava/lang/Object;[Ljava/lang/Object;)F',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('D', '(ILjava/lang/Object;[Ljava/lang/Object;)D',function->'0xcbfd9ed9') failed on class com_app_protect_A.

Register native ('L', '(ILjava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;',function->'0xcbfd9ed9') failed on class com_app_protect_A.

这十个 vmp 函数根据类型不同而分别调用。

初始化附加段的 vmp 数据：

======================= Registers =======================

  R0=0x2100000  R1=0x29be544  R2=0x2c80  R3=0x2100028

R4=0x7ff928  R5=0x7ff940  R6=0x7ff998  R7=0x7ffa08  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0x0  SP=0x7ff920

LR=0xcbfdac4d  PC=0xcbfe62a0

======================= Disassembly =====================

0xcbfe62a0: f0b5   push   {r4, r5, r6, r7, lr}                    // 初始化 vmp 函数

0xcbfe62a2: 03af   add    r7, sp, #0xc

0xcbfe62a4: 93b0   sub    sp, #0x4c

0xcbfe62a6: 6e46   mov    r6, sp

0xcbfe62a8: 7262   str    r2, [r6, #0x24]

>dump 0x29be544 0x2c80  //vmp 数据偏移 和 长度

029BE544: 42 44 30 35 32 37 26 00  00 00 01 00 00 00 26 00  BD0527&.......&.

029BE554: 00 00 00 00 00 00 00 00  00 00 80 00 00 00 2C 00  ..............,.

029BE564: 00 00 80 2B 00 00 02 00  00 00 4C 00 00 00 00 0A  ...+......L.....

![](https://bbs.kanxue.com/upload/attach/202403/981_836Z3VZYNTB2AHP.webp)

.text:CBFE7338             ; ---------------------------------------------------------------------------

.text:CBFE7338

.text:CBFE7338             loc_CBFE7338                            ; CODE XREF: sub_CBFE62A0+108C↑j

.text:CBFE7338                                                     ; sub_CBFE62A0+1092↑j ...

.text:CBFE7338 01 20                       MOVS    R0, #1

.text:CBFE733A 30 64                       STR     R0, [R6,#0x40] 

.text:CBFE733C 24 21                       MOVS    R1, #0x24 ; '$'

.text:CBFE733E 11 F0 1F F9                  BL      j_calloc            // 申请 128 个长度为 0x24 的空间创建索引

解析这段数据，创建索引

![](https://bbs.kanxue.com/upload/attach/202403/981_QCE8CW76PT5EFQX.webp)

>dump 0x2104000

02104000: 00 00 00 0A 26 00 00 00  04 00 00 00 01 00 01 00  ....&...........

02104010: 02 00 02 00 00 00 07 00  86 E5 9B 02 00 00 00 00  ................

02104020: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

02104030: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ...............

一直到：

>dump 0x219b000

0219B000: 7F 00 00 0A 58 00 00 00  01 00 00 00 01 00 02 00  ....X...........

0219B010: 02 00 03 00 00 00 20 00  80 10 9C 02 00 00 00 00  ...... .........

0219B020: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

0219B030: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................

一共 00-7F ：128 个索引表。那就是应该有 128 个函数被 vmp 保护了。

CODE:006CCB0C # Source file: SourceFile

CODE:006CCB0C public void com.app.apptranslate.reading.dailyreading.fragment.PunchReadingScoreappFragment.onCreate(

CODE:006CCB0C       android.os.Bundle p0)

CODE:006CCB0C this = v4

CODE:006CCB0C p0 = v5

CODE:006CCB0C                 .line 88

CODE:006CCB0C                 const                           v1, 0xA00007F

CODE:006CCB12                 .line 89

CODE:006CCB12                 const                           v3, 1

CODE:006CCB18                 new-array                       v0, v3, <t: Object[]>

CODE:006CCB1C                 const                           v3, 0

CODE:006CCB22                 check-cast                      p0, <t: Object>

CODE:006CCB26                 aput-object                     p0, v0, v3

CODE:006CCB2A                 invoke-static                   {v1, this, v0}, <void A.V(int, ref, ref) imp. @ _def_A_V@VILL>

CODE:006CCB30

CODE:006CCB30 locret:

CODE:006CCB30                 return-void

CODE:006CCB30 Method End

CODE:006CCB30 # ---------------------------------------------------------------------------

还真是有这个，厉害！

.text:CBFDACA4 B1 6C                       LDR     R1, [R6,#0x48]

.text:CBFDACA6 13 F0 CD F8                 BL      sub_CBFEDE44    ; 初始化 vmp JAVA 类型

.text:CBFDACAA 02 B0                       ADD     SP, SP, #8

.text:CBFDACAC 01 21                       MOVS    R1, #1

JNIEnv->FindClass([F) was called

JNIEnv->FindClass([D) was called

JNIEnv->FindClass(java/lang/Boolean) was called

JNIEnv->FindClass(java/lang/Byte) was called

JNIEnv->FindClass(java/lang/Character) was called

JNIEnv->FindClass(java/lang/Short) was called

JNIEnv->FindClass(java/lang/Integer) was called

JNIEnv->FindClass(java/lang/Long) was called

JNIEnv->FindClass(java/lang/Float) was called

JNIEnv->FindClass(java/lang/Double) was called

JNIEnv->FindClass(java/lang/Object) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Boolean'>, <init>, (Z)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Byte'>, <init>, (B)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Character'>, <init>, (C)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Short'>, <init>, (S)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Integer'>, <init>, (I)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Long'>, <init>, (J)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Float'>, <init>, (F)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Double'>, <init>, (D)V) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Boolean'>, booleanValue, ()Z) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Byte'>, byteValue, ()B) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Character'>, charValue, ()C) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Short'>, shortValue, ()S) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Integer'>, intValue, ()I) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Long'>, longValue, ()J) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Float'>, floatValue, ()F) was called

JNIEnv->GetMethodId(<class '__main__.java_lang_Double'>, doubleValue, ()D) was called

第三、      对 dex 代码的保护 (VMP)：

APP 加固的 vmp 保护没有像其他加固那样在原代码处加密代码，运行时解密。而是把原始代码移动到附加数据中，使用时根据 vmp 的代理函数解析参数，查询附加数据块，然后运行 vmp 代码的方式。具体看下面的代码分析：

com.app.apptranslate.activity. MainActivity.onCreate 函数是个被 vmp 保护的 dex 函数。

    @Override  // com.app.apptranslate.common.base.BaseObserveActivity

    protected void onCreate(Bundle arg5) {

        A.V(0xA000011, this, new Object[]{((Object)arg5)});

    }

这个函数的参数为 0xA000011，也就是 0x11 号：

======================= Registers =======================

  R0=0xe11f2a00  R1=0x11  R2=0x7ffc70  R3=0xe11f2f00

R4=0xfffffff8  R5=0xcc00ee9c  R6=0x7ffc08  R7=0x7ffc60  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0x7ffff8  SP=0x7ffbc0

LR=0xcbfda033  PC=0xcbfde9b2

======================= Disassembly =====================

0xcbfde9b2:    0bf0adfe bl     #0xcbfea710

R0 是附加数据解析出来的 vmp 部分，R1 就是参数号。下面的函数就是通过参数查询资源。

======================= Registers =======================

  R0=0xe3a26f60  R1=0x44  R2=0x7ffc70  R3=0xe11f2f00

R4=0xfffffff8  R5=0xcc00ee9c  R6=0x7ffc08  R7=0x7ffc60  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0x7ffff8  SP=0x7ffbc0

LR=0xcbfde9b7  PC=0xcbfde9b6

======================= Disassembly =====================

R0 vmp 数据资源地址：

>dump 0xe3a26f60

E3A26F60: 11 00 00 0A 42 01 00 00  04 00 00 00 01 00 05 00  ....B...........

E3A26F70: 02 00 06 00 00 00 95 00  D7 8B 71 D1 00 00 00 00  ..........q.....

E3A26F80: 00 00 00 00 00 00 00 00  12 00 00 0A 38 00 00 00  ............8...

E3A26F90: 04 00 00 00 01 00 01 00  02 00 02 00 00 00 10 00  ................

也就是 n001 函数中 vmp 初始化时候解析的那个 128 个函数。参数数值对应初始化的索引值。

也就是理论上他支持 128 种函数 vmp。

通过这个索引找到 classes.dex 中的附加数据偏移地址：

.text:CBFE0328 31 88                       LDRH    R1, [R6]

======================= Registers =======================

  R0=0xcbfe51a2  R1=0x2056  R2=0xff  R3=0xe11f2f00

R4=0x2056  R5=0xe11f2f00  R6=0xd1718bd7  R7=0x0  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00efc0  SP=0x7ffa98

LR=0xcbfe0315  PC=0xcbfe0348

======================= Disassembly =====================

>dump 0xd1718bd7   // 这个就是 vmp 的 code

D1718BD7: 56 20 54 87 54 00 6F 05  FF 40 8B 00 1B 1F 47 10  V T.T.o..@....G.

D1718BE7: 49 DB 00 00 6F 01 F8 A3  54 20 53 DB 10 00 77 00  I...o...T S...w.

D1718BF7: 60 DB 00 00 2E 01 54 30  50 DB 10 02 54 10 5D DB  `.....T0P...T.].

D1718C07: 00 00 42 00 77 20 FB 0F  05 00 77 00 60 DB 00 00  ..B.w ....w.`...

根据 dex 函数格式，前 0x10 是函数头：

>dump 0xd1718bc7

D1718BC7: 04 00 00 00 01 00 05 00  02 00 06 00 00 00 95 00  ................

D1718BD7: 56 20 54 87 54 00 6F 05  FF 40 8B 00 1B 1F 47 10  V T.T.o..@....G.

D1718BE7: 49 DB 00 00 6F 01 F8 A3  54 20 53 DB 10 00 77 00  I...o...T S...w.

D1718BF7: 60 DB 00 00 2E 01 54 30  50 DB 10 02 54 10 5D DB  `.....T0P...T.].

再回头看看索引：

>dump 0xe3a26f60

E3A26F60: 11 00 00 0A 42 01 00 00  04 00 00 00 01 00 05 00  ....B...........

E3A26F70: 02 00 06 00 00 00 95 00  D7 8B 71 D1 00 00 00 00  ..........q.....

E3A26F80: 00 00 00 00 00 00 00 00  12 00 00 0A 38 00 00 00  ............8...

E3A26F90: 04 00 00 00 01 00 01 00  02 00 02 00 00 00 10 00  ................

说明前 8 个字节就是索引的表，后面跟着 0x10 是函数头，然后就是函数的 code 偏移地址。

索引表 8 个字节，其中前 4 个字节是索引号，后面四个字节是本数据长度。每段数据以 \ 0000 结束。

按照这个就可以解析出附加数据中的 vmp 代码了。

下面执行这个 code：

.text:CBFE0328 31 88                       LDRH    R1, [R6]

.text:CBFE032A FF 22                       MOVS    R2, #0xFF

.text:CBFE032C 16 92                       STR     R2, [SP,#0x10C+var_B4]

.text:CBFE032E 08 46                       MOV     R0, R1

.text:CBFE0330 10 40                       ANDS    R0, R2

======================= Registers =======================

  R0=0x56  R1=0x2056  R2=0xff  R3=0xe11f2f00

R4=0xe11f3304  R5=0xe11f2f00  R6=0xd1718ed3  R7=0x7ffb68  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00efc0  SP=0x7ffa98

LR=0xcbfe0315  PC=0xcbfe0332

======================= Disassembly =====================

0xcbfe0332:    8000   lsls   r0, r0, #2

0xcbfe0334:    1818   adds   r0, r3, r0

0xcbfe0336:    0430   adds   r0, #4

0xcbfe0338:    0024   movs   r4, #0

0xcbfe033a:    1494   str    r4, [sp, #0x50]

>dump 0xd1718ed3

D1718ED3: 56 20 54 87 21 00 54 10  B2 80 01 00 42 02 B7 02  V T.!.T.....B...

D1718EE3: 0B 00 54 10 B2 80 01 00  42 02 EC 00 00 04 54 30  ..T.....B.....T0

D1718EF3: 71 13 02 00 86 02 39 00  0C 7F 54 20 C4 80 21 00  q.....9...T ..!.

D1718F03: 77 10 CF CF 01 00 42 02  78 12 6F 4C 77 00 58 E3  w.....B.x.oLw.X.

解释：

从函数 code 中取两个字节，然后 and 0xFF 就是取一个字节 opcode，这个是 app 加固的 vmp 的 opcode。也就是获取 vmp 的 opcode。继续：

.text:CBFE0332 80 00                       LSLS    R0, R0, #2

.text:CBFE0334 18 18                       ADDS    R0, R3, R0

.text:CBFE0336 04 30                       ADDS    R0, #4

.text:CBFE0338 00 24                       MOVS    R4, #0

.text:CBFE033A 14 94                       STR     R4, [SP,#0x10C+var_BC]

.text:CBFE033C 00 68                       LDR     R0, [R0]

======================= Registers =======================

  R0=0xe11f305c  R1=0x2056  R2=0xff  R3=0xe11f2f00

R4=0xe11f3304  R5=0xe11f2f00  R6=0xd1718ed3  R7=0x7ffb68  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00efc0  SP=0x7ffa98

LR=0xcbfe0315  PC=0xcbfe0338

======================= Disassembly =====================

0xcbfe0338:    0024   movs   r4, #0

0xcbfe033a:    1494   str    r4, [sp, #0x50]

0xcbfe033c:    0068   ldr    r0, [r0]

0xcbfe033e:    0d94   str    r4, [sp, #0x34]

0xcbfe0340:    0b94   str    r4, [sp, #0x2c]

>dump 0xe11f2f00  // 跳转表

E11F2F00: 00 2A 1F E1 5C 3C FE CB  9E 46 FE CB 16 05 FE CB  .*..\<...F......

E11F2F10: 94 2B FE CB 52 4E FE CB  18 06 FE CB FA 36 FE CB  .+..RN.......6..

E11F2F20: D0 52 FE CB AC 44 FE CB  AC 04 FE CB BC 20 FE CB  .R...D....... ..

E11F2F30: 20 24 FE CB F8 2D FE CB  FA 03 FE CB 00 22 FE CB   $...-......."..

.text:CBFE9CE2             loc_CBFE9CE2                            ; CODE XREF: sub_CBFE96A8+62E↑j

.text:CBFE9CE2                                                     ; sub_CBFE96A8+634↑j ...

.text:CBFE9CE2 28 68                       LDR     R0, [R5]

.text:CBFE9CE4 19 68                       LDR     R1, [R3]

.text:CBFE9CE6 80 00                       LSLS    R0, R0, #2

.text:CBFE9CE8 09 18                       ADDS    R1, R1, R0

.text:CBFE9CEA 09 68                       LDR     R1, [R1]

.text:CBFE9CEC 32 69                       LDR     R2, [R6,#0x10]

.text:CBFE9CEE 10 18                       ADDS    R0, R2, R0

.text:CBFE9CF0 00 6B                       LDR     R0, [R0,#0x30]

.text:CBFE9CF2 22 68                       LDR     R2, [R4]

.text:CBFE9CF4 80 00                       LSLS    R0, R0, #2

.text:CBFE9CF6 10 18                       ADDS    R0, R2, R0

.text:CBFE9CF8 01 60                       STR     R1, [R0]

.text:CBFE9CFA 6B 48                       LDR     R0, =(dword_CC01355C - 0xCC00EE9C)

.text:CBFE9CFC 7A 49                       LDR     R1, =(_GLOBAL_OFFSET_TABLE_ - 0xCBFE9D02)

.text:CBFE9CFE 79 44                       ADD     R1, PC          ; _GLOBAL_OFFSET_TABLE_

.text:CBFE9D00 40 18                       ADDS    R0, R0, R1      ; dword_CC01355C

.text:CBFE9D02 02 68                       LDR     R2, [R0]

.text:CBFE9D04 69 48                       LDR     R0, =(dword_CC013574 - 0xCC00EE9C)

.text:CBFE9D06 40 18                       ADDS    R0, R0, R1      ; dword_CC013574

.text:CBFE9D08 00 68                       LDR     R0, [R0]

.text:CBFE9D0A 51 1E                       SUBS    R1, R2, #1

.text:CBFE9D0C 51 43                       MULS    R1, R2

.text:CBFE9D0E 01 22                       MOVS    R2, #1

.text:CBFE9D10 11 42                       TST     R1, R2

.text:CBFE9D12 04 D0                       BEQ     loc_CBFE9D1E

.text:CBFE9D14 FF E7                       B       loc_CBFE9D16

解释：

通过 vmp 的 opcode 来查询执行代码, 首先 opcode 左移两位，就是 x4 然后加上执行函数表的基地址

总结下：就是 vmp 利用 vmp 的 opcode x 4 + 4 然后加上跳转表基地址，得到执行这个 opcode 的代码入口。完成模拟真实 opcode 的过程。也就是这个跳转表其实就是真实 opcode 的映射表。重新来看看这个映射表的生成过程：

模拟器记录如下：

2020-11-24 11:16:53,878  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9a6a, data value = 0x0

2020-11-24 11:16:53,885  Memory WRITE at 0xe11f3060, data size = 4, pc: cbfe9cf8, data value = 0xcbfe034a

2020-11-24 11:16:53,888  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x1

2020-11-24 11:16:53,895  Memory WRITE at 0xe11f2fa0, data size = 4, pc: cbfe9cf8, data value = 0xcbfe0 其他

2020-11-24 11:16:53,898  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x2

2020-11-24 11:16:53,904  Memory WRITE at 0xe11f3074, data size = 4, pc: cbfe9cf8, data value = 0xcbfe0396

2020-11-24 11:16:53,907  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x3

2020-11-24 11:16:53,913  Memory WRITE at 0xe11f32c4, data size = 4, pc: cbfe9cf8, data value = 0xcbfe03ca

2020-11-24 11:16:53,916  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x4

2020-11-24 11:16:53,922  Memory WRITE at 0xe11f2f38, data size = 4, pc: cbfe9cf8, data value = 0xcbfe03fa

2020-11-24 11:16:53,925  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x5

2020-11-24 11:16:53,932  Memory WRITE at 0xe11f2f88, data size = 4, pc: cbfe9cf8, data value = 0xcbfe0438

2020-11-24 11:16:53,935  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x6

2020-11-24 11:16:53,941  Memory WRITE at 0xe11f32ac, data size = 4, pc: cbfe9cf8, data value = 0xcbfe0474

2020-11-24 11:16:53,944  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x7

2020-11-24 11:16:53,951  Memory WRITE at 0xe11f2f28, data size = 4, pc: cbfe9cf8, data value = 0xcbfe04ac

2020-11-24 11:16:53,954  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x8

2020-11-24 11:16:53,960  Memory WRITE at 0xe11f3190, data size = 4, pc: cbfe9cf8, data value = 0xcbfe04e2

……………

2020-11-24 11:16:56,224  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0xfb

2020-11-24 11:16:56,230  Memory WRITE at 0xe11f3034, data size = 4, pc: cbfe9cf8, data value = 0xcbfe4e52

2020-11-24 11:16:56,233  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0xfc

2020-11-24 11:16:56,239  Memory WRITE at 0xe11f313c, data size = 4, pc: cbfe9cf8, data value = 0xcbfe4e52

2020-11-24 11:16:56,242  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0xfd

2020-11-24 11:16:56,249  Memory WRITE at 0xe11f3264, data size = 4, pc: cbfe9cf8, data value = 0xcbfe4e52

2020-11-24 11:16:56,251  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0xfe

2020-11-24 11:16:56,258  Memory WRITE at 0xe11f3280, data size = 4, pc: cbfe9cf8, data value = 0xcbfe4e52

2020-11-24 11:16:56,261  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0xff

2020-11-24 11:16:56,267  Memory WRITE at 0xe11f2f98, data size = 4, pc: cbfe9cf8, data value = 0xcbfe4e52

上面的代码组装了这个表，而这个表就是原始 opcode 跟执行代码的映射表。通过这种方式完成了 vmp 的 opcode 到真实 opcode 的映射。

比如，0xA000011 参数时：

.text:CBFE0328 31 88                       LDRH    R1, [R6]   获取 opcode

.text:CBFE032A FF 22                       MOVS    R2, #0xFF

.text:CBFE032C 16 92                       STR     R2, [SP,#0x10C+var_B4]

.text:CBFE032E 08 46                       MOV     R0, R1

.text:CBFE0330 10 40                       ANDS    R0, R2

.text:CBFE0332 80 00                       LSLS    R0, R0, #2

.text:CBFE0334 18 18                       ADDS    R0, R3, R0   定位表偏移

.text:CBFE0336 04 30                       ADDS    R0, #4

.text:CBFE0338 00 24                       MOVS    R4, #0

.text:CBFE033A 14 94                       STR     R4, [SP,#0x10C+var_BC]

.text:CBFE033C 00 68                       LDR     R0, [R0]       获取执行代码地址

.text:CBFE033E 0D 94                       STR     R4, [SP,#0x10C+var_D8]

.text:CBFE0340 0B 94                       STR     R4, [SP,#0x10C+var_E0]

.text:CBFE0342 06 94                       STR     R4, [SP,#0x10C+var_F4]

.text:CBFE0344 27 46                       MOV     R7, R4

.text:CBFE0346 0C 46                       MOV     R4, R1

.text:CBFE0348 87 46                       MOV     PC, R0  跳转到执行代码地址

======================= Registers =======================

  R0=0xcbfe51a2  R1=0x2056  R2=0x54  R3=0xe11f2f00

R4=0x2056  R5=0xe11f2f00  R6=0xd1718bd7  R7=0x0  R8=0x0

R9=0x0  R10=0x0  R11=0x0  R12=0xcc00efc0  SP=0x7ffa98

LR=0xcbfe0315  PC=0xcbfe51a6

======================= Disassembly =====================

0xcbfe51a6:    0f20   movs   r0, #0xf

0xcbfe51a8:    1040   ands   r0, r2

0xcbfe51aa:    002f   cmp    r7, #0

0xcbfe51ac:    00d0   beq    #0xcbfe51b0

0xcbfe51ae:    1046   mov    r0, r2

>dump 0xd1718bd7   //code

D1718BD7: 56 20 54 87 54 00 6F 05  FF 40 8B 00 1B 1F 47 10  V T.T.o..@....G.

D1718BE7: 49 DB 00 00 6F 01 F8 A3  54 20 53 DB 10 00 77 00  I...o...T S...w.

D1718BF7: 60 DB 00 00 2E 01 54 30  50 DB 10 02 54 10 5D DB  `.....T0P...T.].

D1718C07: 00 00 42 00 77 20 FB 0F  05 00 77 00 60 DB 00 00  ..B.w ....w.`...

R1 为 vmp 的 opcode，为 0x56，查询到的执行代码地址为：R0=0xcbfe51a2 ，我们从映射表中查询下这个执行代码映射的是真实 opcode 值为多少：

2020-11-24 11:16:54,914  Memory WRITE at 0x7ffa40, data size = 4, pc: cbfe9e1c, data value = 0x6f

2020-11-24 11:16:54,921  Memory WRITE at 0xe11f305c, data size = 4, pc: cbfe9cf8, data value = 0xcbfe51a2

从这个记录就可以看到映射的真实 opcode 为 0x6f，查询下这个 opcode 命令是啥：

![](https://bbs.kanxue.com/upload/attach/202403/981_TQ7GP9ZM2J3CANP.webp)

这是个 invoke-super，也就是调用父类的 onCreate，执行下看看结果：

JNIEnv->FindClass

(com/app/apptranslate/common/base/BaseObserveActivity) was called

JNIEnv->GetMethodId

(<class '__main__.com_app_apptranslate_common_base_BaseObserveActivity'>, onCreate, (Landroid/os/Bundle;)V) was called

确实是对类 com_app_apptranslate_common_base_BaseObserveActivity 的 onCreate 函数的调用。

![](https://bbs.kanxue.com/upload/attach/202403/981_QT9U3P5P5KCDH56.webp)

                    【映射表函数代码】

从这段代码也可以看出来，APP 加固 vmp 也是只替换了一个字节的 opcode，其他操作数都没有改变，并且 vmp 的 opcode 和原始的 opcode 存在映射表关系。当然这个映射关系是通过 opcode 的执行代码地址联系起来的。

![](https://bbs.kanxue.com/upload/attach/202403/981_QHSYCYWA4BX9WZK.webp)

具体代码这样的：

    @Override  // com.app.rp.lib.base.BaseFragmentActivity

    protected void onCreate(Bundle arg5) {

        A.V(0xA00000F, this, new Object[]{((Object)arg5)});

}

第四、      VMP_dex 代码还原：

通过上面的分析可以知道，APP 加固的 dex 代码 vmp 保护中原始的代码数据被转移到附加数据中，并建立了 vmp_fun_ID 与函数体的索引。原始的 dex 代码位置给加固保护系统的 vmp 代理函数替代，并传递 vmp_fun_ID 到 Native 层处理。Vmp 的 Native 处理函数根据传递的 vmp_fun_ID 到索引表中查询函数数据地址。并按照函数头参数初始化函数。然后根据索引表中 code 的地址开始执行 vmp 代码。而 vmp 代码只是替换了一个字节的 opcode 指令，其他操作数都保持没有改变。而 vmp 的处理函数初始化时就进行了原始 opcode 与 vmp 执行流程代码的映射表，而这个映射表可以跟 vmp_fun_ID 进行关联。而 vmp_fun_ID 又跟 vmp 的代码进行了关联。所以 vmp 的 opcode 其实就是跟原始的 opcode 进行了关联。根据这个我们就可以还原出原始的 dex 代码。还需要把 dex 函数的执行偏移地址修正到被 vmp 转移的还原出来的 dex 代码地址。也就是需要解决两个问题：

1、   还原被替换的 dex 代码中的 opcode

还原 vmp 的 dex 代码的 opcode，需要把原始的 opcode 与 vmp 的 opcode 一一对应起来。而程序是通过 vmp 的 opcode 数值进行计算，查询执行代码地址，执行代码流程的。而这个执行代码地址又在原始 opcode 表中进行了映射。所以可以根据这两个表进行逆查询，找到 vmp 的 opcode 与原始 opcode 的对应关系。我们到内存中获取这两个表就可以了。

2、   修正 dex 方法索引中 offset 到还原的代码地址

这个 vmp 流程里面并没有 dex 方法索引表与 vmp 代码的直接关联，所以需要解决 dex 方法索引中被 vmp 保护的函数，以及这些函数与调用 Native 入口函数的对应关系。也就是需要解决 vmp_fun_ID 与 dex 方法索引中的方法对应关系，这个在看雪论坛的帖子中是运行时记录 dex_method_id 与 vmp_fun_ID 关系，然后手动建立联系表的方式。这样如果运行时没有被执行的就不能被记录下来。也比较复杂。而我决定使用 IDA 的分析功能建立这个对应表来：

首先我们知道 vmp 保护的都是 onCreate 这个函数，所以我们在 IDA 中过滤下这个函数，效果如下：

![](https://bbs.kanxue.com/upload/attach/202403/981_GF55RZ5K7W2EHDW.webp)

然后找到长度为 26 的所有这类函数：

![](https://bbs.kanxue.com/upload/attach/202403/981_ZNRC6BER54RH9VJ.webp)

这样我们就可以把 128 个 vmp 的 onCreate 函数找出来。

然后再根据这个找到方法索引中这些函数的索引地址和对应的 vmp_fun_ID，双击上面的这些函数：

![](https://bbs.kanxue.com/upload/attach/202403/981_GSNPJ7URTZ5K6CQ.webp)

我们记录下这些数据，并形成查询表：

![](https://bbs.kanxue.com/upload/attach/202403/981_7K69H3JD7MSNZEC.webp)

这样我们就得到了所有的 vmp 的 vmp_fun_ID 和方法索引表中的方法索引地址，以及加固调用入口函数地址。

其实我们只需要 vmp_fun_ID 和方法索引表方法地址，加固调用入口在我们修正方法索引表的方法代码 offse 时就没有作用了。

先需要修改 dex 文件：

1、   修改 dex struct header_item dex_header 头里面的 uint data_size 把附加数据部分加进去。

2、   Dex 的 struct map_list_type dex_map_list 中增加一个节，把 vmp 的代码段加进去。

![](https://bbs.kanxue.com/upload/attach/202403/981_H8HBTV2JYY374AF.webp)1、  

3、   修改 map_list 项数量：

![](https://bbs.kanxue.com/upload/attach/202403/981_PYEWVZV45U6Z5FP.webp)

1、  

4、   APP 加固自己解析了 Method 的头，需要重建 Method 头。

![](https://bbs.kanxue.com/upload/attach/202403/981_459HAEWE6NGM88K.webp)

好了，有了这些数据和表我们就可以写代码来还原 dex 的 vmp 了。

Python

Python 代码如下：

```
import codecs
import leb128
  
def Get_uleb128(vmp_addr,mode):
    if mode == 1:
        offset = leb128.u.encode(vmp_addr)
    if mode == 2:
        offset = leb128.u.decode(vmp_addr)
    return offset
  
def ReadFile(file, address, len,flag=0):
    add = int(address)
    file.seek(add)
    if flag == 0:
        data = int.from_bytes(file.read(len), byteorder="little", signed=False)
    else:
        data =file.read(len)
    return data
  
def main():
    """
    opcode长度表，按照这个长度表获取每条指令的长度，才能查询到下条指令
    """
    opcode_Length = [2, 2, 4, 4, 4, 4, 4, 2, 4, 4, 2, 2, 2, 2, 2, 2, 2, 2, 2, 4, 6, 4, 4, 6, 10, 4, 4, 4, 4, 4, 2, 4, 4,
                     2, 4, 4, 6, 6, 6, 2, 2, 4, 4, 6, 6, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
                     4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
                     4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 6, 6, 6, 6, 6, 4, 6, 6, 6, 6, 6, 4, 4, 2, 4, 2, 4, 2, 2, 2, 2, 2,
                     2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
                     4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
                     2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
                     4, 4, 4, 4, 4, 4, 4, 6, 4, 6, 4, 4, 4, 4, 4, 4, 4, 6, 6, 6, 6, 4, 4, 4, 4]
    """
      利用IDA建立的vmp_fun_ID与方法索引表的方法索引地址关联表。利用这个表修正方法索引中的offset地址，这个offset地址是uleb128格式的，需要转换下。
    """
    vmp_dex_fun_idtomethod_id = [0xA000001, 0x6F8559, 0xA000003, 0x6F9EF5, 0xA000005, 0x6FA139, 0xA000006, 0x6FAA81,
                             0xA000002, 0x6F9E75, 0xA000000, 0x6F84D8, 0xA000008, 0x6FB05F, 0xA000007, 0x6FAD9B,
                             0xA000004, 0x6F9FDD, 0xA000009, 0x72B90C, 0xA00000A, 0x72BEFC, 0xA00000B, 0x72CC31,
                             0xA00000C, 0x72CC75, 0xA00000D, 0x72CF40, 0xA000010, 0x72D0E3, 0xA000011, 0x72D3C9,
                             0xA000013, 0x72D592, 0xA000014, 0x72D6E1, 0xA000016, 0x72D78F, 0xA000017, 0x72D806,
                             0xA000018, 0x72D977, 0xA000019, 0x72DA58, 0xA00001A, 0x72DAD2, 0xA00001B, 0x72DBAE,
                             0xA00001C, 0x72DD1E, 0xA00001D, 0x72DE8F, 0xA00001E, 0x72DF14, 0xA00001F, 0x72DFC2,
                             0xA000020, 0x72E0E5, 0xA000021, 0x72E29A, 0xA000022, 0x72E334, 0xA000023, 0x73178C,
                             0xA00000F, 0x72D06C, 0xA000012, 0x72D466, 0xA000015, 0x72D739, 0xA000025, 0x738125,
                             0xA000026, 0x73E615, 0xA000027, 0x744157, 0xA000028, 0x744267, 0xA000029, 0x7443EB,
                             0xA00002A, 0x7445DE, 0xA00002B, 0x74482A, 0xA00002C, 0x7449F1, 0xA00002D, 0x74600C,
                             0xA00002E, 0x747E40, 0xA00002F, 0x74E453, 0xA000030, 0x74E4C5, 0xA000031, 0x74E541,
                             0xA000032, 0x74E727, 0xA000033, 0x74E7BB, 0xA000034, 0x74E843, 0xA000035, 0x74E90B,
                             0xA000036, 0x74E971, 0xA000037, 0x74EA36, 0xA000038, 0x74EAD0, 0xA000039, 0x74EC03,
                             0xA00003A, 0x74ED09, 0xA00003B, 0x74EDAD, 0xA00003C, 0x74EE7A, 0xA00003D, 0x74EEE9,
                             0xA00003E, 0x74EF78, 0xA00003F, 0x74F045, 0xA000040, 0x7500A6, 0xA000041, 0x7505F4,
                             0xA000042, 0x7507A4, 0xA000043, 0x7508C5, 0xA000044, 0x750937, 0xA000045, 0x750981,
                             0xA000046, 0x750A09, 0xA000047, 0x750A99, 0xA000048, 0x750B6C, 0xA000049, 0x750D70,
                             0xA00004A, 0x750E71, 0xA00004B, 0x750F22, 0xA00004D, 0x7512BE, 0xA00004E, 0x751306,
                             0xA00004F, 0x7513F8, 0xA000050, 0x751436, 0xA000051, 0x7514AE, 0xA000052, 0x7517AB,
                             0xA000053, 0x7518B5, 0xA000054, 0x751961, 0xA000055, 0x751993, 0xA000056, 0x751A4D,
                             0xA000057, 0x751AA1, 0xA000058, 0x751B27, 0xA000059, 0x751BF1, 0xA00005A, 0x751DC3,
                             0xA00005B, 0x751E84, 0xA00004C, 0x7510D3, 0xA00005C, 0x7520E2, 0xA00005D, 0x752534,
                             0xA00005E, 0x7526E1, 0xA00005F, 0x75292C, 0xA000060, 0x752AB6, 0xA000061, 0x753A08,
                             0xA000062, 0x75694D, 0xA000063, 0x757072, 0xA000064, 0x75B38B, 0xA000065, 0x75B49E,
                             0xA000066, 0x75B54D, 0xA000067, 0x75B5E0, 0xA000068, 0x75B799, 0xA000069, 0x75C4CC,
                             0xA00006A, 0x75C5CA, 0xA00006B, 0x75C6FF, 0xA00006C, 0x75CAAE, 0xA00006D, 0x75CB1C,
                             0xA00006E, 0x75CD6F, 0xA00006F, 0x75CEB2, 0xA000070, 0x75CF32, 0xA000071, 0x75CFE3,
                             0xA000072, 0x75D02D, 0xA000073, 0x75D04F, 0xA000074, 0x75D0ED, 0xA000075, 0x75D4F8,
                             0xA000076, 0x75E83D, 0xA000077, 0x75E8B4, 0xA000078, 0x762043, 0xA000079, 0x762284,
                             0xA00007A, 0x762434, 0xA00007B, 0x763175, 0xA00007C, 0x763233, 0xA00007D, 0x76334C,
                             0xA00007E, 0x763696, 0xA00007F, 0x765011, 0xA000024, 0x731856, 0xA00000E, 0x72CFA9]
    vmp_dex2_fun_idtomethod_id = [0xa010000, 0x752ecf, 0xa010001, 0x757ba7, 0xa010002, 0x757cc4, 0xa010003, 0x757e5f,
                             0xa010004, 0x757f52, 0xa010005, 0x757fb2, 0xa010006, 0x758003, 0xa010007, 0x758056,
                             0xa010008, 0x7580a7, 0xa010009, 0x758104, 0xa01000a, 0x75815f, 0xa01000b, 0x7581ac,
                             0xa01000c, 0x75822c, 0xa01000d, 0x7588a8, 0xa01000e, 0x75ab80, 0xa01000f, 0x75b2a0,
                             0xa010010, 0x75d394, 0xa010011, 0x75d57f, 0xa010012, 0x75f249, 0xa010013, 0x75f8a1,
                             0xa010014, 0x76479b, 0xa010015, 0x764d50, 0xa010016, 0x764d7a, 0xa010017, 0x76e7b4,
                             0xa010018, 0x76f81c, 0xa010019, 0x76ff10, 0xa01001a, 0x771356, 0xa01001b, 0x77f579,
                             0xa01001c, 0x7857ca, 0xa01001d, 0x785886, 0xa01001e, 0x7863be, 0xa01001f, 0x786646,
                             0xa010020, 0x78a519, 0xa010021, 0x78f2d7, 0xa010022, 0x790310, 0xa010023, 0x7908c5,
                             0xa010024, 0x790bb0, 0xa010025, 0x790d67, 0xa010026, 0x790d91, 0xa010027, 0x791787,
                             0xa010028, 0x791a94, 0xa010029, 0x791de6, 0xa01002a, 0x791f58, 0xa01002b, 0x79200e,
                             0xa01002c, 0x793a62, 0xa01002d, 0x793c5d, 0xa01002e, 0x793d61, 0xa01002f, 0x793e04,
                             0xa010030, 0x793e84, 0xa010031, 0x795417, 0xa010032, 0x79558c, 0xa010033, 0x795721,
                             0xa010034, 0x7958a7, 0xa010035, 0x796616, 0xa010036, 0x7966ea, 0xa010037, 0x796f0e,
                             0xa010038, 0x79728d, 0xa010039, 0x7975e0, 0xa01003a, 0x797620, 0xa01003b, 0x797a50,
                             0xa01003c, 0x7988fa, 0xa01003d, 0x799ac0, 0xa01003e, 0x79b78c, 0xa01003f, 0x79c924,
                             0xa010040, 0x79cbb3, 0xa010041, 0x79da68, 0xa010042, 0x79deeb, 0xa010043, 0x79ea97,
                             0xa010044, 0x79eea4, 0xa010045, 0x7a2cdf, 0xa010046, 0x7a2e2b, 0xa010047, 0x7a2eef,
                             0xa010048, 0x7a2f6f, 0xa010049, 0x7a306b, 0xa01004a, 0x7a3772, 0xa01004b, 0x7a3a3d,
                             0xa01004c, 0x7a3aaf, 0xa01004d, 0x7a3b93, 0xa01004e, 0x7a3bfb, 0xa01004f, 0x7a3cbc,
                             0xa010050, 0x7a3d20, 0xa010051, 0x7a3df5, 0xa010052, 0x7a3e9d, 0xa010053, 0x7a5018,
                             0xa010054, 0x7a51d6, 0xa010055, 0x7a52c8, 0xa010056, 0x7a548e, 0xa010057, 0x7a5bd3,
                             0xa010058, 0x7a5d74, 0xa010059, 0x7a5f0c, 0xa01005a, 0x7a605c, 0xa01005b, 0x7a60fa,
                             0xa01005c, 0x7a627f, 0xa01005d, 0x7a636f, 0xa01005e, 0x7a6953, 0xa01005f, 0x7a6a8d,
                             0xa010060, 0x7a6bc0, 0xa010061, 0x7a6c6f, 0xa010062, 0x7a6ccb, 0xa010063, 0x7a6ee6,
                             0xa010064, 0x7a7086, 0xa010065, 0x7a72ea, 0xa010066, 0x7a74fa, 0xa010067, 0x7a76fc,
                             0xa010068, 0x7a784c, 0xa010069, 0x7a7a60, 0xa01006a, 0x7a7baa, 0xa01006b, 0x7a7c20,
                             0xa01006c, 0x7a7c94, 0xa01006d, 0x7a7db5, 0xa01006e, 0x7a7df9, 0xa01006f, 0x7a7e43,
                             0xa010070, 0x7a7f1c, 0xa010071, 0x7a887c, 0xa010072, 0x7aa2b1, 0xa010073, 0x7aa401,
                             0xa010074, 0x7aa7e8, 0xa010075, 0x7aa961, 0xa010076, 0x7aaa3c, 0xa010077, 0x7aade0,
                             0xa010078, 0x7aae1a, 0xa010079, 0x7ab0ab, 0xa01007a, 0x7b8bc2, 0xa01007b, 0x7b8c75,
                             0xa01007c, 0x7bb79e]
    vmp_fun_idtomethod_id = [0xA020000,0x567960,0xA020001,0x56BB4C,0xA020002,0x56C039,0xA020003,0x57FC9F,0xA020004,0x57FD18,0xA020005,0x581D21]
    """
    由于APP加固把vmp的dex code作为附加数据段附加值dex文件的尾部，不能被正常加载，且vmp的附加数据中dex code header被修改，code部分也没有对齐，需要恢复时我们需要把code恢复到code段，并且
    每段code要对齐，然后需要把MAP段重新放到增加了code的code段后面，就涉及到MAP段在header中的offset修正，以及MAP段中MAP段中的定义。由于增加了code段数据，所以header中uint data_size 需要把
    新增加的code长度加上,header中的文件大小也需要修正，最好包括check值。
    Name    Start      End R          W   X      D          L   Align  Base   Type   Class  AD     ds
    HEADER   00000000 00000070 ?      ?      ?      .      L      byte   0002   public DATA   32     0002
    STR_IDS  00000070 00034670 ?      ?      ?      .      L      byte   0003   public DATA   32     0003
    TYPES    00034670 0003D030 ?      ?      ?      .      L      byte   0004   public DATA   32     0004
    PROTO    0003D030 0005E780 ?      ?      ?      .      L      byte   0005   public DATA   32     0005
    FIELDS   0005E780 000DE760 ?      ?      ?      .      L      byte   0006   public DATA   32     0006
    METHODS  000DE760 00151000 ?      ?      ?      .      L      byte   0007   public DATA   32     0007
    CLASS_DEF 00151000 001842A0 ?      ?      ?      .      L      byte   0008   public DATA   32     0008
    STRINGS  001842A0 002EAE18 ?      ?      ?      .      L      byte   000A   public DATA   32     000A
    CODE    002EAE18 00769088 ?      ?      ?      .      L      byte   0001   public CODE   32     0001
    MAP     00769088 00769164 ?      ?      ?      .      L      byte   0009   public DATA   32     0009
     
    """
    vmp_dex_filename = r"AndroidNativeEm/samples/appjiagu/data/classes_vmp.dex"
    vmp_dex1_filename = r"AndroidNativeEm/samples/appjiagu/data/classes1_vmp.dex"
    vmp_dex1_filename = r"AndroidNativeEm/samples/appjiagu/data/classes2_vmp.dex"
    vm_data = codecs.open(vmp_dex1_filename, "rb")
    vmp_data_buff = vm_data.read()
    """
    这段代码是vmp的opcode与原始opcode的对应关系映射表的构造算法，我们利用这段代码直接构造出一个vmp opcode与原始opcode对应的映射表
    .text:CBFE9CE2             ; ---------------------------------------------------------------------------
    .text:CBFE9CE2
    .text:CBFE9CE2             loc_CBFE9CE2                            ; CODE XREF: sub_CBFE96A8+62E↑j
    .text:CBFE9CE2                                                     ; sub_CBFE96A8+634↑j ...
    .text:CBFE9CE2 28 68                       LDR     R0, [R5]        ; 原始opcode
    .text:CBFE9CE4 19 68                       LDR     R1, [R3]        ; vmp 执行代码表基地址
    .text:CBFE9CE6 80 00                       LSLS    R0, R0, #2
    .text:CBFE9CE8 09 18                       ADDS    R1, R1, R0
    .text:CBFE9CEA 09 68                       LDR     R1, [R1]        ; 获取vmp执行代码地址值
    .text:CBFE9CEC 32 69                       LDR     R2, [R6,#0x10]  ; 方法表基地址
    .text:CBFE9CEE 10 18                       ADDS    R0, R2, R0
    .text:CBFE9CF0 00 6B                       LDR     R0, [R0,#0x30]  ; method_ID
    .text:CBFE9CF2 22 68                       LDR     R2, [R4]        ; vmp opcode映射表基地址
    .text:CBFE9CF4 80 00                       LSLS    R0, R0, #2
    .text:CBFE9CF6 10 18                       ADDS    R0, R2, R0
    .text:CBFE9CF8 01 60                       STR     R1, [R0]        ; 写映射表
     
    修改如下：
    .text:CBFE9CEA 29 68                       LDR     R1, [R5]  ;获取原始opcode 写入到表中，后面查询到的就是原始opcode了
    获得的数据如下：
    Offset      0  1  2  3  4  5  6  7   8  9  A  B  C  D  E  F
  
    00000000   AA 00 00 00 C9 00 00 00  09 00 00 00 6C 00 00 00   ª   É       l   
    00000010   E5 00 00 00 11 00 00 00  9C 00 00 00 77 00 00 00   å       œ   w   
    00000020   C3 00 00 00 07 00 00 00  56 00 00 00 5D 00 00 00   Ã       V   ]   
    00000030   81 00 00 00 04 00 00 00  59 00 00 00 B0 00 00 00           Y   °   
    00000040   3D 00 00 00 DF 00 00 00  60 00 00 00 2C 00 00 00   =   ß   `   ,   
    00000050   67 00 00 00 E0 00 00 00  44 00 00 00 19 00 00 00   g   à   D       
    00000060   E7 00 00 00 A2 00 00 00  B7 00 00 00 52 00 00 00   ç   ¢   ·   R   
    00000070   12 00 00 00 2B 00 00 00  7D 00 00 00 3A 00 00 00       +   }   :   
    00000080   E9 00 00 00 05 00 00 00  F5 00 00 00 10 00 00 00   é       õ       
    00000090   7C 00 00 00 FF 00 00 00  42 00 00 00 01 00 00 00   |   ÿ   B       
    000000A0   16 00 00 00 46 00 00 00  B8 00 00 00 1D 00 00 00       F   ¸       
    000000B0   AB 00 00 00 8F 00 00 00  0B 00 00 00 32 00 00 00   «           2   
    000000C0   D2 00 00 00 8E 00 00 00  A4 00 00 00 53 00 00 00   Ò   Ž   ¤   S   
    000000D0   87 00 00 00 4F 00 00 00  25 00 00 00 DB 00 00 00   ‡   O   %   Û   
    000000E0   9E 00 00 00 EA 00 00 00  F1 00 00 00 57 00 00 00   ž   ê   ñ   W   
    000000F0   90 00 00 00 E8 00 00 00  D3 00 00 00 6D 00 00 00       è   Ó   m   
    00000100   68 00 00 00 99 00 00 00  0C 00 00 00 C5 00 00 00   h   ™       Å   
    00000110   79 00 00 00 CF 00 00 00  6A 00 00 00 70 00 00 00   y   Ï   j   p   
    00000120   C7 00 00 00 B1 00 00 00  D8 00 00 00 BD 00 00 00   Ç   ±   Ø   ½   
    00000130   FB 00 00 00 A5 00 00 00  9D 00 00 00 41 00 00 00   û   ¥       A   
    00000140   B4 00 00 00 AE 00 00 00  3C 00 00 00 82 00 00 00   ´   ®   <   ‚   
    00000150   6E 00 00 00 A3 00 00 00  6F 00 00 00 00 00 00 00   n   £   o       
    00000160   E6 00 00 00 47 00 00 00  E4 00 00 00 BE 00 00 00   æ   G   ä   ¾   
    00000170   02 00 00 00 E1 00 00 00  9B 00 00 00 5F 00 00 00       á   ›   _   
    00000180   B9 00 00 00 0D 00 00 00  17 00 00 00 48 00 00 00   ¹           H   
    00000190   D5 00 00 00 9F 00 00 00  0A 00 00 00 AC 00 00 00   Õ   Ÿ       ¬   
    000001A0   76 00 00 00 1E 00 00 00  72 00 00 00 4A 00 00 00   v       r   J   
    000001B0   C1 00 00 00 F2 00 00 00  CB 00 00 00 1A 00 00 00   Á   ò   Ë       
    000001C0   23 00 00 00 7A 00 00 00  DE 00 00 00 F0 00 00 00   #   z   Þ   ð   
    000001D0   85 00 00 00 89 00 00 00  28 00 00 00 71 00 00 00   …   ‰   (   q   
    000001E0   5B 00 00 00 97 00 00 00  26 00 00 00 B3 00 00 00   [   —   &   ³   
    000001F0   30 00 00 00 92 00 00 00  4D 00 00 00 C4 00 00 00   0   ’   M   Ä   
    00000200   F9 00 00 00 65 00 00 00  98 00 00 00 73 00 00 00   ù   e   ˜   s   
    00000210   DC 00 00 00 5A 00 00 00  14 00 00 00 7B 00 00 00   Ü   Z       {   
    00000220   40 00 00 00 3E 00 00 00  C6 00 00 00 22 00 00 00   @   >   Æ   "   
    00000230   29 00 00 00 BC 00 00 00  FC 00 00 00 49 00 00 00   )   ¼   ü   I   
    00000240   F8 00 00 00 83 00 00 00  EC 00 00 00 21 00 00 00   ø   ƒ   ì   !   
    00000250   FA 00 00 00 B2 00 00 00  BB 00 00 00 36 00 00 00   ú   ²   »   6   
    00000260   94 00 00 00 1F 00 00 00  7F 00 00 00 A8 00 00 00   ”           ¨   
    00000270   50 00 00 00 91 00 00 00  5C 00 00 00 3B 00 00 00   P   ‘   \   ;   
    00000280   DA 00 00 00 C8 00 00 00  18 00 00 00 08 00 00 00   Ú   È           
    00000290   AF 00 00 00 2F 00 00 00  1C 00 00 00 55 00 00 00   ¯   /       U   
    000002A0   62 00 00 00 35 00 00 00  63 00 00 00 33 00 00 00   b   5   c   3   
    000002B0   E3 00 00 00 64 00 00 00  EE 00 00 00 CE 00 00 00   ã   d   î   Î   
    000002C0   0E 00 00 00 2D 00 00 00  8A 00 00 00 88 00 00 00       -   Š   ˆ   
    000002D0   A7 00 00 00 95 00 00 00  B5 00 00 00 38 00 00 00   §   •   µ   8   
    000002E0   4C 00 00 00 69 00 00 00  5E 00 00 00 54 00 00 00   L   i   ^   T   
    000002F0   34 00 00 00 6B 00 00 00  43 00 00 00 75 00 00 00   4   k   C   u   
    00000300   B6 00 00 00 2A 00 00 00  F6 00 00 00 93 00 00 00   ¶   *   ö   “   
    00000310   66 00 00 00 3F 00 00 00  8C 00 00 00 A0 00 00 00   f   ?   Œ       
    00000320   D6 00 00 00 D9 00 00 00  96 00 00 00 D0 00 00 00   Ö   Ù   –   Ð   
    00000330   20 00 00 00 ED 00 00 00  EB 00 00 00 CA 00 00 00       í   ë   Ê   
    00000340   A1 00 00 00 BA 00 00 00  61 00 00 00 45 00 00 00   ¡   º   a   E   
    00000350   39 00 00 00 78 00 00 00  74 00 00 00 37 00 00 00   9   x   t   7   
    00000其他   FD 00 00 00 F7 00 00 00  CD 00 00 00 F3 00 00 00   ý   ÷   Í   ó   
    00000370   24 00 00 00 0F 00 00 00  4E 00 00 00 FE 00 00 00   $       N   þ   
    00000380   AD 00 00 00 CC 00 00 00  C2 00 00 00 1B 00 00 00   ­   Ì   Â       
    00000390   E2 00 00 00 F4 00 00 00  84 00 00 00 80 00 00 00   â   ô   „   €   
    000003A0   27 00 00 00 58 00 00 00  06 00 00 00 EF 00 00 00   '   X       ï   
    000003B0   13 00 00 00 7E 00 00 00  2E 00 00 00 15 00 00 00       ~   .       
    000003C0   03 00 00 00 8B 00 00 00  A6 00 00 00 8D 00 00 00       ‹   ¦       
    000003D0   9A 00 00 00 D4 00 00 00  D7 00 00 00 4B 00 00 00   š   Ô   ×   K   
    000003E0   A9 00 00 00 BF 00 00 00  D1 00 00 00 51 00 00 00   ©   ¿   Ñ   Q   
    000003F0   31 00 00 00 C0 00 00 00  86 00 00 00 DD 00 00 00   1   À   †   Ý   
    """
    # opcode 映射表
    opcode_v_opcode_table = codecs.open(r"AndroidNativeEm/samples/appjiagu/data/vmp_opcode_table", "rb")
    # 把MAP段前的数据都读进来
    # uint map_off   8138768 34h    4h     Fg: Bg: File offset of map list map address
    data_ptr = ReadFile(vm_data,0x34,4)
    map_list_size = ReadFile(vm_data,data_ptr,4)
    code_ptr = data_ptr
    magic_begin = data_ptr + (map_list_size * 0xC) +4
    # dex_data = list(vm_data.read(data_ptr))
    dex_data = list(ReadFile(vm_data,0,data_ptr,1))
    # vmp 数据开始地址
    # magic = int.from_bytes((vmp_data_buff[magic_begin:(magic_begin+4)]), byteorder="little", signed=False)
    magic_data =bytearray.fromhex("42443035323726")
    magic = vmp_data_buff[magic_begin:(magic_begin+7)]
    while magic != magic_data:
        magic_begin += 1
        magic = vmp_data_buff[magic_begin:(magic_begin+7)]
    begin_addr = magic_begin + 0x2C
    # 由于附加数据里面的vmp没有对齐，所以长度不正确，所以需要查询这个索引，才能确保正确的vmp dex数据
    # vmp_methon_id = 0xA000000
    vmp_fun_id = ReadFile(vm_data, begin_addr, 4)
    vmp_len = ReadFile(vm_data,(magic_begin + 0x1A),2)
    for i in range(vmp_len):
        # 先读取vmp dex code 长度
        code_len = ReadFile(vm_data, (begin_addr + 0x14), 2)
        if code_len > 0x13:
            # 如果code 长度大于 vmp代理入口函数长度就在code段后面增加这个Method code,恢复代码后需要修正Method 索引中的offset
            for j in range(2):
                # 先读取vmp dex Method 头的前2个字节
                data_head = ReadFile(vm_data,(begin_addr + 6 + j),1)
                dex_data.append(data_head)
            # 由于APP加固把Method 的header 进行了重新拼装，所以需要重新修正
            for j in range(6):
                # 先读取vmp dex Method 头的前6个字节
                data_head = ReadFile(vm_data,(begin_addr + 0xE + j),1)
                dex_data.append(data_head)
            index_fun = vmp_fun_idtomethod_id.index(vmp_fun_id) + 1
            method_fun = vmp_fun_idtomethod_id[index_fun]
            print(hex(method_fun))
            vmp_fun_ep = ReadFile(vm_data, method_fun, 4, 1)
            # 修正Method fun offset
            new_offset = Get_uleb128(code_ptr,1)
            for j in range(4):
                dex_data[method_fun + j] = new_offset[j]
            # 获取vmp 代理入口函数地址
            ep_addr = Get_uleb128(vmp_fun_ep, 2)
            # 获取debug info address 写入到新的code中
            for j in range(4):
                debug_info = ReadFile(vm_data, (ep_addr + 8 + j), 1)
                dex_data.append(debug_info)
            # 把vmp Method header 中code length写入新code中
            for j in range(2):
                len_dat = ReadFile(vm_data, (begin_addr + 0x14 + j), 1)
                dex_data.append(len_dat)
            # 四个字节对齐
            for j in range(2):
                dex_data.append(0)
            vmp_code_addr = begin_addr + 0x16  # code 地址
            # Method header 中的code length 是按双字节计算的，所以需要乘以2
            code_len = code_len * 2
            code_ptr += code_len +0x10
            while code_len > 0:
                vmp_opcode = ReadFile(vm_data, vmp_code_addr, 1)
                index_opcode = vmp_opcode * 4
                opcode = ReadFile(opcode_v_opcode_table, index_opcode, 1)
                # 写入新的code中
                dex_data.append(opcode)
                # 获取这个opcode 的指令长度，然后减去opcode一个字节
                opcode_len = opcode_Length[opcode] - 1
                code_length = opcode_Length[opcode]
                # 把code 指令写到新的code中
                for k in range(opcode_len):
                    data_code = ReadFile(vm_data, (vmp_code_addr + 1 + k), 1)
                    dex_data.append(data_code)
                vmp_code_addr += code_length
                code_len -= code_length
                print("vmp_opcode:", hex(vmp_opcode))
                print("opcode:", hex(opcode))
            # 写两个字节0结束符，防止小于vmp 代理入口函数时没有结束标记问题
            for k in range(2):
                dex_data.append(0)
            code_ptr += 2
            begin_addr = vmp_code_addr + code_length
        else:
            # 如果小于等于vmp代理入口长度，就写回到这个代理入口地址，且不需要修正Method 索引的offset
            index_fun = vmp_fun_idtomethod_id.index(vmp_fun_id) + 1
            method_fun = vmp_fun_idtomethod_id[index_fun]
            vmp_fun_ep =ReadFile(vm_data, method_fun, 4,1)
            # 获取vmp 代理入口函数地址
            ep_addr = Get_uleb128(vmp_fun_ep, 2)
            for j in range(2):
                # 先读取vmp dex Method 头的前2个字节
                data_head = ReadFile(vm_data,(begin_addr + 6 + j),1)
                dex_data[ep_addr+j] = data_head
            # 由于APP加固把Method 的header 进行了重新拼装，所以需要重新修正
            for j in range(6):
                # 先读取vmp dex Method 头的前6个字节
                data_head = ReadFile(vm_data, (begin_addr + 0xE + j), 1)
                dex_data[ep_addr + j + 2] = data_head
            # 保存原debug info
            # 写入code 长度
            for j in range(2):
                data_head = ReadFile(vm_data, (begin_addr + 0x14 + j), 1)
                dex_data[ep_addr + 0x12 +j] = data_head
            code_addr = begin_addr + 0x16  # code 地址
            code_old_addr = ep_addr + 0x10
            # Method header 中的code length 是按双字节计算的，所以需要乘以2
            code_len = code_len * 2
            while code_len > 0:
                vmp_opcode = ReadFile(vm_data, code_addr, 1)
                index_opcode = vmp_opcode * 4
                opcode = ReadFile(opcode_v_opcode_table, index_opcode, 1)
                dex_data[code_old_addr] = opcode
                # 获取这个opcode 的指令长度，然后减去opcode一个字节
                opcode_len = opcode_Length[opcode] - 1
                code_length = opcode_Length[opcode]
                # 把code 指令写到原始的地方
                for k in range(opcode_len):
                    data_code = ReadFile(vm_data, (code_addr + 1 + k), 1)
                    dex_data[code_old_addr+ 1 + k] = data_code
                code_old_addr = code_old_addr + code_length
                code_addr += code_length
                code_len -= code_length
                print("vmp_opcode:", hex(vmp_opcode))
                print("opcode:", hex(opcode))
            # 写两个字节0结束符，防止小于vmp 代理入口函数时没有结束标记问题
            for k in range(2):
                dex_data[code_old_addr] = 0
            begin_addr = code_addr + code_length
        vmp_fun_id_n = vmp_fun_id + 1
        vmp_fun_id = ReadFile(vm_data, begin_addr, 4)
        vmp_fun_end = vmp_fun_id_n % 0x100
        if vmp_fun_end > (vmp_len-1):
            break
        while vmp_fun_id != vmp_fun_id_n:
            begin_addr += 1
            vmp_fun_id = ReadFile(vm_data, begin_addr, 4)
  
    # data = bytearray(dex_data)
    map_data = list(ReadFile(vm_data,data_ptr,0xd8,1))
    for i in range(len(map_data)):
        dex_data.append(map_data[i])
    # dex_data.append(map_data)
    # 修正MAP段属性
    # 修正header中MAP段offset
    new_code_len = code_ptr - data_ptr
    data_size = ReadFile(vm_data,0x68,4) + new_code_len
    data_size_new = data_size.to_bytes(4, byteorder="little", signed=False)
    for i in range(4):
        dex_data[0x68 + i] = data_size_new[i]
    map_offset = code_ptr.to_bytes(4, byteorder="little", signed=False)
    for i in range(4):
        dex_data.append(map_offset[i])
        dex_data[0x34 + i] = map_offset[i]
    # 修正header 中文件大小
    file_size = len(dex_data)
    file_size_new = file_size.to_bytes(4,byteorder="little", signed=False)
    for i in range(4):
        dex_data[0x20 + i] = file_size_new[i]
    data = bytearray(dex_data)
    bak_filename = r"AndroidNativeEm/samples/appjiagu/data/classes2_vmp_new.dex"
    f = open(bak_filename,"wb")
    if f:
        f.write(data)
    f.close()
if __name__ == '__main__':
    main()

```

[[培训]《安卓高级研修班 (网课)》月薪三万计划，掌握调试、分析还原 ollvm、vmp 的方法，定制 art 虚拟机自动化脱壳的方法](https://www.kanxue.com/book-section_list-84.htm)

最后于 2024-4-30 18:02 被 kanxue 编辑 ，原因：

[#混淆加固](forum-161-1-121.htm)