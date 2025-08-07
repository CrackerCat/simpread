> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2050639-1-1.html)

> [md]# Overt 安全检测工具深度解析 ## 前言在移动互联网时代，设备安全检测是风控体系中的重要一环。掌握前沿风控技术是每个安全人员的重要的必修课程之一。Overt 项 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Aa865186652

Overt 安全检测工具深度解析
----------------

### 前言

在移动互联网时代，设备安全检测是风控体系中的重要一环。掌握前沿风控技术是每个安全人员的重要的必修课程之一。Overt 项目致力于收集并集成当前公开的检测技术于一身，构建了一个全面、高效、安全的 Android 设备安全检测体系。

本文将从技术原理、架构设计、模块实现等多个维度，深入解析 Overt 项目的核心技术和实现细节，为安全从业者提供一份全面的技术参考。通过详细分析 JVM 获取、ClassLoader 遍历、Linker 解析、SSL 证书验证、TEE 检测等关键技术难点，帮助读者理解当前的 Android 设备安全检测技术，所以赶快来领取顶级攻防游戏的入场券吧。

### 技术架构概览

#### 整体架构设计

Overt 采用分层架构设计，确保核心检测逻辑的安全性和性能：

```
┌─────────────────────────────────────┐
│            Java层 (UI层)             │
│  ├── MainActivity                   │
│  ├── InfoCardContainer              │
│  ├── InfoCard                       │
│  └── MainApplication                │
├─────────────────────────────────────┤
│            JNI桥接层                 │
│  ├── native-lib.cpp                 │
│  └── JNI接口函数                      │
├─────────────────────────────────────┤
│          Native C++层               │
│  ├── 设备信息管理中心                  │
│  │  └── zDevice                     │
│  ├── 检测模块 (_info结尾)             │
│  │  ├── tee_info                    │
│  │  ├── root_file_info              │
│  │  ├── package_info                │
│  │  ├── system_prop_info            │
│  │  ├── system_setting_info         │
│  │  ├── mounts_info                 │
│  │  ├── maps_info                   │
│  │  ├── linker_info                 │
│  │  ├── class_loader_info           │
│  │  ├── ssl_info                    │
│  │  ├── time_info                   │
│  │  ├── task_info                   │
│  │  └── port_info                   │
│  ├── 工具类 (z开头)                  │
│  │  ├── zJavaVm (JVM管理)           │
│  │  ├── zLinker (动态链接器)         │
│  │  ├── zClassLoader (类加载器)      │
│  │  ├── zHttps (HTTPS客户端)         │
│  │  ├── zTee (TEE检测)              │
│  │  ├── zFile (文件操作)             │
│  │  ├── zElf (ELF解析)              │
│  │  ├── zJson (JSON处理)            │
│  │  ├── zUtil (通用工具)            │
│  │  ├── zCrc32 (CRC校验)            │
│  │  └── zLog (日志记录)              │
│  └── 自定义API层                    │
│     ├── config.h (宏控制)           │
│     ├── nonstd_libc (自定义库函数)   │
│     ├── string (自定义字符串)        │
│     ├── vector (自定义向量)          │
│     └── map (自定义映射)             │
└─────────────────────────────────────┘

```

#### 技术特点

*   **模块化设计**: 每个检测功能独立成模块，便于维护和扩展
*   **安全性**: 核心逻辑在 Native 层，更难被反编译和篡改

### 核心技术难点深度解析

#### 1. Root 检测技术实现

##### 技术背景

Root 检测是 Android 设备安全检测的核心环节，通过检测系统中是否存在 Root 权限获取工具来判断设备是否已被 Root。传统的 Root 检测往往容易被绕过，因此需要采用更底层、更隐蔽的检测方式。

##### 核心实现原理

**1.1 自定义 Syscall 实现**

```
// syscall.h - 自定义系统调用封装
#define __asm_syscall(...) \
    __asm__ __volatile__("svc #0" \
        : "=r"(x0) : __VA_ARGS__ : "memory", "cc")

// ARM64系统调用内联函数
__attribute__((always_inline))
static inline long __syscall4(long n, long a, long b, long c, long d) {
    register long x8 __asm__("x8") = n;
    register long x0 __asm__("x0") = a;
    register long x1 __asm__("x1") = b;
    register long x2 __asm__("x2") = c;
    register long x3 __asm__("x3") = d;
    __asm_syscall("r"(x8), "0"(x0), "r"(x1), "r"(x2), "r"(x3));
    return x0;
}

// 系统调用号定义 (Linux ARM64)
#define SYS_openat      56
#define SYS_stat        4
#define SYS_access      21
#define SYS_readlink    89
#define SYS_newfstatat  79

```

**1.2 自定义文件操作实现**

```
// nonstd_libc.cpp - 自定义文件操作函数
int nonstd_open(const char *pathname, int flags, ...) {
    LOGE("nonstd_open called: path, pathname ? pathname : "NULL", flags);

    if (!pathname) {
        LOGV("open: NULL pathname");
        return -1;
    }

    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list args;
        va_start(args, flags);
        mode = va_arg(args, mode_t);
        va_end(args);
    }

    // 使用 openat 系统调用，AT_FDCWD 表示相对于当前工作目录
    int fd = (int)__syscall4(SYS_openat, AT_FDCWD, (long)pathname, flags, mode);
    LOGV("open: returned fd=%d", fd);
    return fd;
}

int nonstd_access(const char* __path, int __mode) {
    LOGV("nonstd_access called: path='%s', mode=0x%x", __path ? __path : "NULL", __mode);

    if (!__path) {
        LOGV("access: NULL path");
        errno = EFAULT;  // 错误地址
        return -1;
    }

    long result = __syscall2(SYS_access, (uintptr_t)__path, (long)__mode);

    if (result < 0) {
        errno = -result;
        LOGV("access: failed, errno=%d", errno);
        return -1;
    }

    LOGV("access: success");
    return 0;
}

```

**1.3 Root 文件检测实现**

```
// root_file_info.cpp - Root文件检测
map<string, map<string, string>> get_root_file_info() {
    LOGD("get_root_file_info called");
    map<string, map<string, string>> info;

    // 常见的Root工具文件路径
    const char* paths[] = {
        "/sbin/su",                    // SuperSU
        "/system/bin/su",              // 系统su
        "/system/xbin/su",             // 系统xbin su
        "/data/local/xbin/su",         // 用户空间su
        "/data/local/bin/su",          // 用户空间bin su
        "/system/sd/xbin/su",          // SD卡su
        "/system/bin/failsafe/su",     // 安全模式su
        "/data/local/su",              // 数据分区su
        "/system/xbin/mu",             // Magisk su
        "/system_ext/bin/su",          // 系统扩展su
        "/apex/com.android.runtime/bin/suu", // APEX su
    };

    for (const char* path : paths) {
        LOGD("Checking path: %s", path);
        zFile file(path);
        if(file.exists()){
            LOGI("Black file exists: %s", path);
            info[path]["risk"] = "error";
            info[path]["explain"] = "black file but exist";
        }
    }

    return info;
}

```

**技术优势：**

*   **自定义 Syscall**：绕过系统调用 Hook，直接使用汇编指令

**技术难点解析：**

*   **系统调用 Hook 检测**：需要理解 ARM64 系统调用机制

#### 2. JVM 和 Context 在 init 期间的获取机制

##### 技术挑战

在 Android Native 库的`__attribute__((constructor))`初始化阶段，Java 层尚未完全启动，此时获取 JVM 实例和 Context 对象面临巨大挑战。

##### 核心实现原理

**1.2 JVM 获取机制**

```
zJavaVm::zJavaVm() {
    LOGD("Constructor called");

    zElf libart = zLinker::getInstance()->find_lib("libart.so");

    auto *JNI_GetCreatedJavaVMs  = (jint (*)(JavaVM **, jsize, jsize *))libart.find_symbol("JNI_GetCreatedJavaVMs");
    LOGI("JNI_GetCreatedJavaVMs: %p", JNI_GetCreatedJavaVMs);
    if (JNI_GetCreatedJavaVMs == nullptr) {
        LOGE("GetCreatedJavaVMs not found");
        return;
    }

    JavaVM* vms[1];
    jsize num_vms = 0;
    if (JNI_GetCreatedJavaVMs(vms, 1, &num_vms) != JNI_OK || num_vms == 0) {
        LOGE("GetCreatedJavaVMs failed");
        return;
    }

    jvm = vms[0];
    LOGI("JVM initialized successfully");
}

```

**关键技术点：**

*   通过`zLinker`动态解析`libart.so`获取`JNI_GetCreatedJavaVMs`函数指针
*   绕过 JNI_OnLoad 限制，直接获取已创建的 JVM 实例
*   实现跨线程 JNI 环境管理

**1.3 Context 创建机制**

```
jobject createNewContext(JNIEnv* env) {
    LOGD("createNewContext called");
    if (env == nullptr) {
        LOGD("createNewContext: env is null");
        return nullptr;
    }

    // 1. 获取 ActivityThread 实例
    jclass clsActivityThread = env->FindClass("android/app/ActivityThread");
    if (clsActivityThread == nullptr) {
        LOGE("Failed to find ActivityThread class");
        return nullptr;
    }

    jmethodID m_currentAT = env->GetStaticMethodID(clsActivityThread, "currentActivityThread", "()Landroid/app/ActivityThread;");
    if (m_currentAT == nullptr) {
        LOGE("Failed to find currentActivityThread method");
        return nullptr;
    }

    jobject at = env->CallStaticObjectMethod(clsActivityThread, m_currentAT);
    if (at == nullptr) {
        LOGE("Failed to get current ActivityThread");
        return nullptr;
    }

    // 2. 获取 mBoundApplication 字段
    jfieldID fid_mBoundApp = env->GetFieldID(clsActivityThread, "mBoundApplication", "Landroid/app/ActivityThread$AppBindData;");
    if (fid_mBoundApp == nullptr) {
        LOGE("Failed to find mBoundApplication field");
        env->DeleteLocalRef(at);
        return nullptr;
    }

    jobject mBoundApp = env->GetObjectField(at, fid_mBoundApp);
    if (mBoundApp == nullptr) {
        LOGE("Failed to get mBoundApplication");
        env->DeleteLocalRef(at);
        return nullptr;
    }

    // 3. 获取 LoadedApk 信息
    jclass clsAppBindData = env->FindClass("android/app/ActivityThread$AppBindData");
    if (clsAppBindData == nullptr) {
        LOGE("Failed to find AppBindData class");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        return nullptr;
    }

    jfieldID fid_info = env->GetFieldID(clsAppBindData, "info", "Landroid/app/LoadedApk;");
    if (fid_info == nullptr) {
        LOGE("Failed to find info field");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        return nullptr;
    }

    jobject loadedApk = env->GetObjectField(mBoundApp, fid_info);
    if (loadedApk == nullptr) {
        LOGE("Failed to get LoadedApk");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        return nullptr;
    }

    // 4. 创建 Application 实例
    jclass clsLoadedApk = env->FindClass("android/app/LoadedApk");
    if (clsLoadedApk == nullptr) {
        LOGE("Failed to find LoadedApk class");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        env->DeleteLocalRef(loadedApk);
        return nullptr;
    }

    jmethodID m_makeApp = env->GetMethodID(clsLoadedApk, "makeApplication", "(ZLandroid/app/Instrumentation;)Landroid/app/Application;");
    if (m_makeApp == nullptr) {
        LOGE("Failed to find makeApplication method");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        env->DeleteLocalRef(loadedApk);
        return nullptr;
    }

    jobject app = env->CallObjectMethod(loadedApk, m_makeApp, JNI_FALSE, nullptr);
    if (app == nullptr) {
        LOGE("Failed to create Application instance");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        env->DeleteLocalRef(loadedApk);
        return nullptr;
    }

    // 5. 获取 Application 的 base context
    jclass clsApp = env->GetObjectClass(app);
    if (clsApp == nullptr) {
        LOGE("Failed to get Application class");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        env->DeleteLocalRef(loadedApk);
        env->DeleteLocalRef(app);
        return nullptr;
    }

    jmethodID m_getBaseContext = env->GetMethodID(clsApp, "getBaseContext", "()Landroid/content/Context;");
    if (m_getBaseContext == nullptr) {
        LOGE("Failed to find getBaseContext method");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        env->DeleteLocalRef(loadedApk);
        env->DeleteLocalRef(app);
        return nullptr;
    }

    jobject context = env->CallObjectMethod(app, m_getBaseContext);
    if (context == nullptr) {
        LOGE("Failed to get base context");
        env->DeleteLocalRef(at);
        env->DeleteLocalRef(mBoundApp);
        env->DeleteLocalRef(loadedApk);
        env->DeleteLocalRef(app);
        return nullptr;
    }

    return context;
}

```

**技术难点解析：**

*   **反射机制**: 通过反射访问 Android 内部类，绕过 API 限制
*   **生命周期管理**: 在 Application 未完全初始化时创建 Context
*   **内存管理**: 正确处理 JNI 局部引用，避免内存泄漏

#### 3. ClassLoader 遍历技术实现

##### 技术背景

Android 的 ClassLoader 体系复杂，包括 BootClassLoader、PathClassLoader、DexClassLoader 等，需要深度遍历所有 ClassLoader 来检测异常类加载。

##### 实现流程

**3.1 标准 API 遍历方式**

```
vector<string> getClassNameList(JNIEnv *env, jobject classloader) {
    LOGD("getClassNameList called");
    vector<string> classNameList = {};

    // 获取 classloader 的类
    jclass classloaderClass = env->GetObjectClass(classloader);

    // 获取 pathList 字段
    jfieldID pathListFieldID = env->GetFieldID(classloaderClass, "pathList","Ldalvik/system/DexPathList;");
    if (pathListFieldID == nullptr) {
        LOGE("Failed to find field 'pathList' in classloader");
        return classNameList;
    }

    jobject pathList = env->GetObjectField(classloader, pathListFieldID);
    if (pathList == nullptr) {
        LOGE("pathList is null");
        return classNameList;
    }

    // 获取 dexElements 字段
    jclass dexPathListClass = env->GetObjectClass(pathList);
    jfieldID dexElementsFieldID = env->GetFieldID(dexPathListClass, "dexElements",
                                                  "[Ldalvik/system/DexPathList$Element;");
    if (dexElementsFieldID == nullptr) {
        LOGE("Failed to find field 'dexElements' in DexPathList");
        return classNameList;
    }

    jobjectArray dexElements = (jobjectArray) env->GetObjectField(pathList, dexElementsFieldID);
    if (dexElements == nullptr) {
        LOGE("dexElements is null");
        return classNameList;
    }

    // 遍历 dexElements 数组
    jint dexElementsLength = env->GetArrayLength(dexElements);
    for (jint i = 0; i < dexElementsLength; i++) {
        jobject dexElement = env->GetObjectArrayElement(dexElements, i);

        // 获取 dexFile 字段
        jclass dexElementClass = env->GetObjectClass(dexElement);
        jfieldID dexFileFieldID = env->GetFieldID(dexElementClass, "dexFile",
                                                  "Ldalvik/system/DexFile;");
        if (dexFileFieldID == nullptr) {
            LOGE("Failed to find field 'dexFile' in DexPathList$Element");
            continue;
        }

        jobject dexFile = env->GetObjectField(dexElement, dexFileFieldID);
        if (dexFile == nullptr) {
            LOGE("dexFile is null");
            continue;
        }

        // 获取 DexFile 的类名列表
        jclass dexFileClass = env->GetObjectClass(dexFile);
        jmethodID entriesMethodID = env->GetMethodID(dexFileClass, "entries", "()Ljava/util/Enumeration;");
        if (entriesMethodID == nullptr) {
            LOGE("Failed to find method 'entries' in DexFile");
            continue;
        }

        jobject entries = env->CallObjectMethod(dexFile, entriesMethodID);
        if (entries == nullptr) {
            LOGE("entries is null");
            continue;
        }

        // 遍历类名
        jclass enumerationClass = env->FindClass("java/util/Enumeration");
        jmethodID hasMoreElementsMethodID = env->GetMethodID(enumerationClass, "hasMoreElements", "()Z");
        jmethodID nextElementMethodID = env->GetMethodID(enumerationClass, "nextElement","()Ljava/lang/Object;");
        while (env->CallBooleanMethod(entries, hasMoreElementsMethodID)) {
            jstring className = (jstring) env->CallObjectMethod(entries, nextElementMethodID);
            const char *classNameStr = env->GetStringUTFChars(className, nullptr);
            classNameList.push_back(classNameStr);
            env->ReleaseStringUTFChars(className, classNameStr);
            env->DeleteLocalRef(className);
        }

        // 清理局部引用
        env->DeleteLocalRef(entries);
        env->DeleteLocalRef(dexFile);
        env->DeleteLocalRef(dexElement);
        env->DeleteLocalRef(dexElementClass);
        env->DeleteLocalRef(dexFileClass);
        env->DeleteLocalRef(enumerationClass);
    }

    // 清理局部引用
    env->DeleteLocalRef(dexElements);
    env->DeleteLocalRef(pathList);
    env->DeleteLocalRef(classloaderClass);

    return classNameList;
}

```

**3.2 ART 内部 API 遍历方式**

```
class ClassLoaderVisitor : public art::SingleRootVisitor {
public:
    vector<string> classLoaderStringList;
    vector<string> classNameList;

    ClassLoaderVisitor(JNIEnv *env, jclass classLoader) : env_(env), classLoader_(classLoader) {
        classLoaderStringList = vector<string>();
        classNameList = vector<string>();
    }

    void VisitRoot(art::mirror::Object *root, const art::RootInfo &info ATTRIBUTE_UNUSED) final {
        jobject object = newLocalRef(env_, (jobject) root);
        if (object != nullptr) {
            if (env_->IsInstanceOf(object, classLoader_)) {
                string classLoaderName = getClassName((JNIEnv *) env_, object);
                LOGE("ClassLoaderVisitor classLoaderName %s", classLoaderName.c_str());
                string classLoaderString = get_class_loader_string(env_, object);
                LOGE("ClassLoaderVisitor %s", classLoaderString.c_str());
                vector<string> classLoaderClassNameList = getClassNameList((JNIEnv *) env_, object);

                classLoaderStringList.push_back(classLoaderString);
                classNameList.insert(classNameList.end(), classLoaderClassNameList.begin(), classLoaderClassNameList.end());

            }else{
                deleteLocalRef(env_, object);
            }
        }
    }

private:
    JNIEnv *env_;
    jclass classLoader_;
};

void zClassLoader::checkGlobalRef(JNIEnv *env, jclass clazz) {
    LOGD("checkGlobalRef called");
    auto VisitRoots = (void (*)(void *, void *)) zLinker::getInstance()->find_lib("libart.so").find_symbol("_ZN3art9JavaVMExt10VisitRootsEPNS_11RootVisitorE");

    if (VisitRoots == nullptr) {
        LOGE("Failed to find method 'VisitRoots' in JavaVMExt");
        return;
    }
    JavaVM *jvm;
    env->GetJavaVM(&jvm);
    ClassLoaderVisitor visitor(env, clazz);
    VisitRoots(jvm, &visitor);

    classLoaderStringList.insert(classLoaderStringList.end(), visitor.classLoaderStringList.begin(), visitor.classLoaderStringList.end());
    classNameList.insert(classNameList.end(), visitor.classNameList.begin(), visitor.classNameList.end());
}

```

**技术难点解析：**

*   **ART 内部 API**: 直接调用 ART 运行时内部 API，绕过 Java 层限制
*   **内存布局**: 理解 ART 对象内存布局，正确解析类信息

#### 4. Linker 遍历技术实现

##### 技术背景

Android 的动态链接器 (linker) 管理所有加载的共享库，通过遍历 linker 内部数据结构可以检测异常库加载和 Hook 行为。

##### 实现流程

**4.1 Linker 内部结构解析**

```
zLinker::zLinker() {
    LOGD("Constructor called");
    this->elf_file_ptr = parse_elf_file("/system/bin/linker64");
    LOGI("linker64 elf_file_ptr %p", this->elf_file_ptr);

    parse_elf_head();
    parse_program_header_table();
    parse_section_table();

    this->elf_mem_ptr = get_maps_base("linker64");
    LOGI("linker64 elf_mem_ptr %p", this->elf_mem_ptr);

    soinfo*(*solist_get_head)() = (soinfo*(*)())(this->find_symbol("__dl__Z15solist_get_headv"));

    LOGI("linker64 solist_get_head %p", solist_get_head);

    soinfo_head = solist_get_head();
    LOGI("soinfo_head %p", soinfo_head);

    soinfo_get_realpath =  (char*(*)(void*))(this->find_symbol("__dl__ZNK6soinfo12get_realpathEv"));
    LOGI("soinfo_get_realpath %p", soinfo_get_realpath);
}

```

**4.2 库遍历机制**

```
vector<string> zLinker::get_libpath_list(){
    LOGD("get_libpath_list called");
    vector<string> libpath_list = vector<string>();
    soinfo* soinfo = soinfo_head;
    while(soinfo->next != nullptr){
        char* real_path = soinfo_get_realpath(soinfo);
        libpath_list.push_back(real_path);
        soinfo = soinfo->next;
    }
    LOGI("get_libpath_list: found %zu libraries", libpath_list.size());
    return libpath_list;
}

```

**4.3 库完整性校验**

```
bool zLinker::check_lib_crc(const char* so_name){
    LOGD("check_lib_crc called with so_name: %s", so_name);
    LOGI("check_lib_hash so_name: %s", so_name);
    zElf elf_lib_file = zLinker::getInstance()->find_lib(so_name);
    uint64_t elf_lib_file_crc = elf_lib_file.get_elf_header_crc() + elf_lib_file.get_program_header_crc() + elf_lib_file.get_text_segment_crc();

    zElf elf_lib_mem = zElf((void*)zLinker::get_maps_base(so_name));
    uint64_t elf_lib_mem_crc = elf_lib_mem.get_elf_header_crc() + elf_lib_mem.get_program_header_crc() + elf_lib_mem.get_text_segment_crc();

    LOGI("check_lib_hash elf_lib_file: %p crc: %lu", elf_lib_file.elf_file_ptr, elf_lib_file_crc);
    LOGI("check_lib_hash elf_lib_mem: %p crc: %lu", elf_lib_mem.elf_mem_ptr, elf_lib_mem_crc);

    bool crc_mismatch = elf_lib_file_crc != elf_lib_mem_crc;
    if (crc_mismatch) {
        LOGW("check_lib_crc: CRC mismatch detected for %s", so_name);
    } else {
        LOGI("check_lib_crc: CRC match for %s", so_name);
    }
    return crc_mismatch;
}

```

**技术难点解析：**

*   **内存布局**: 理解 linker 内部数据结构布局
*   **符号解析**: 动态解析 linker 内部符号
*   **完整性校验**: 通过 CRC 检测内存篡改

#### 5. SSL 证书检测技术实现

##### 技术背景

SSL 证书检测是验证网络通信安全性的重要手段，本项目通过 mbedtls 库实现完整的 TLS 握手和证书验证流程。

##### 实现流程

**5.1 自定义 HTTPS 客户端**

```
HttpsResponse zHttps::performRequest(const HttpsRequest& request) {
    HttpsResponse response;

    // 使用请求的超时时间，如果没有设置则使用默认超时
    int timeout_seconds = request.timeout_seconds > 0 ? request.timeout_seconds : default_timeout_seconds;

    // 为本次请求创建独立的计时器和资源
    RequestTimer timer(timeout_seconds);
    RequestResources resources;
    LOGI("Starting HTTPS request with timeout: %d seconds", timer.timeout_seconds);

    // 安全检查：确保只使用HTTPS协议，但允许任意端口
    if (request.url.substr(0, 8) != "https://") {
        response.error_message = "Only HTTPS URLs are supported";
        LOGE("Security Error: Only HTTPS protocol is allowed");
        return response;
    }

    // 初始化mbedtls（如果需要）
    if (!initialized) {
        if (!initialize()) {
            response.error_message = "Failed to initialize mbedtls";
            LOGE("Failed to initialize mbedtls");
            return response;
        }
    }

    // 确保每次请求都使用干净的状态
    // 这可以防止前一次请求的失败状态影响当前请求
    LOGI("Starting new HTTPS request to %s", request.host.c_str());

    // 重置全局SSL状态，确保每次请求都是独立的
    if (initialized) {
        // 重新初始化全局资源以确保干净状态
        cleanup();
        if (!initialize()) {
            response.error_message = "Failed to reinitialize mbedtls";
            LOGE("Failed to reinitialize mbedtls");
            return response;
        }
    }

    // 记录本地设置的证书固定信息
    auto it = pinned_certificates.find(request.host);
    if (it != pinned_certificates.end()) {
        response.pinned_certificate = it->second;
    }

    char error_buf[0x1000];
    int ret;

    // 使用自定义socket连接
    LOGI("Connecting to %s:%d with custom socket...", request.host.c_str(), request.port);

    // 检查超时
    if (timer.isTimeout()) {
        response.error_message = "Connection timeout before establishing connection";
        LOGE("Connection timeout before establishing connection");
        return response;
    }

    // 记录连接开始时间
    time_t connect_start = time(nullptr);
    LOGI("Connection attempt started at: %ld", connect_start);

    // 使用自定义socket连接，带超时控制
    resources.sockfd = connectWithTimeout(request.host, request.port, timeout_seconds);

    // 记录连接结束时间
    time_t connect_end = time(nullptr);
    int connect_duration = connect_end - connect_start;
    LOGI("Connection attempt completed in %d seconds", connect_duration);

    if (resources.sockfd < 0) {
        response.error_message = "Connection failed with custom socket";
        LOGE("Connection failed after %d seconds with custom socket", connect_duration);
        return response;
    }

    // 将socket文件描述符设置到mbedtls网络上下文
    resources.server_fd.fd = resources.sockfd;

    // 标记连接建立完成
    timer.markConnection();

    // ... TLS握手和HTTP请求处理代码 ...

    return response;
}

```

**5.2 证书指纹验证**

```
bool zHttps::verifyCertificatePinning(const mbedtls_x509_crt* cert, const string& hostname) {
    LOGD("verifyCertificatePinning called for hostname: %s", hostname.c_str());
    auto it = pinned_certificates.find(hostname);
    if (it == pinned_certificates.end()) {
        LOGI("No pinned certificate for hostname: %s, skipping pinning check.", hostname.c_str());
        return true; // 没有固定证书时跳过验证
    }

    const CertificateInfo& pinned = it->second;
    bool serial_match = (pinned.serial_number == extractCertificateInfo(cert).serial_number);
    bool fingerprint_match = (pinned.fingerprint_sha256 == extractCertificateInfo(cert).fingerprint_sha256);

    // 只有当期望的subject不为空时才验证subject
    bool subject_match = true;
    if (!pinned.subject.empty()) {
        subject_match = (pinned.subject == extractCertificateInfo(cert).subject);
    }

    if (!serial_match || !fingerprint_match || !subject_match) {
        LOGW("Certificate pinning verification failed for %s", hostname.c_str());
        LOGD("serial_match %d fingerprint_match %d subject_match %d ", serial_match, fingerprint_match, subject_match);

        // 输出详细信息用于调试
        if (!serial_match) {
            LOGD("Expected serial: %s, Got: %s", 
                 pinned.serial_number.c_str(), 
                 extractCertificateInfo(cert).serial_number.c_str());
        }
        if (!fingerprint_match) {
            LOGD("Expected fingerprint: %s, Got: %s", 
                 pinned.fingerprint_sha256.c_str(), 
                 extractCertificateInfo(cert).fingerprint_sha256.c_str());
        }
        if (!subject_match && !pinned.subject.empty()) {
            LOGD("Expected subject: %s, Got: %s", 
                 pinned.subject.c_str(), 
                 extractCertificateInfo(cert).subject.c_str());
        }

        return false;
    }
    LOGI("Certificate pinning verification passed for %s", hostname.c_str());
    return true;
}

```

**5.3 mbedtls 静态库编译**

```
wget https://dl.google.com/android/repository/android-ndk-r25c-linux.zip
unzip android-ndk-r25c-linux.zip

export ANDROID_NDK=/home/lxz/android-ndk-r25c
export TOOLCHAIN=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64
export TARGET=aarch64-linux-android
export API=21

export CC=$TOOLCHAIN/bin/$TARGET$API-clang
export AR=$TOOLCHAIN/bin/llvm-ar
export CFLAGS="--target=$TARGET$API"

git clone https://github.com/Mbed-TLS/mbedtls.git
cd mbedtls
git checkout mbedtls-2.28 # 或者选择较稳定版本

git submodule update --init --recursive

```

```
#!/bin/bash

set -e

# 配置路径
NDK_ROOT="/home/lxz/android-ndk-r25c"
API=21
TARGET=aarch64-linux-android
TOOLCHAIN=$NDK_ROOT/toolchains/llvm/prebuilt/linux-x86_64
CLANG=$TOOLCHAIN/bin/${TARGET}${API}-clang
SYSROOT=$TOOLCHAIN/sysroot

SRC_DIR=$(pwd)
OUT_DIR="$SRC_DIR/build-android-arm64"
LIB_DIR="$OUT_DIR/lib"
OBJ_DIR="$OUT_DIR/obj"

echo "使用编译器: $CLANG"

CFLAGS="--target=$TARGET --sysroot=$SYSROOT -I$SRC_DIR/include -DMBEDTLS_CONFIG_FILE=\"mbedtls/config.h\" -Os -fPIC"

mkdir -p "$LIB_DIR" "$OBJ_DIR/mbedtls" "$OBJ_DIR/mbedx509" "$OBJ_DIR/mbedcrypto"

# 编译 libmbedtls.a 相关源文件
MBEDTLS_SRCS=$(cd "$SRC_DIR/library" && ls ssl_*.c net_sockets.c timing.c version.c)

echo "编译 libmbedtls.a"
for file in $MBEDTLS_SRCS; do
    echo "  -> $file"
    $CLANG $CFLAGS -c "$SRC_DIR/library/$file" -o "$OBJ_DIR/mbedtls/${file%.c}.o"
done
$TOOLCHAIN/bin/llvm-ar rcs "$LIB_DIR/libmbedtls.a" "$OBJ_DIR/mbedtls"/*.o

# 编译 libmbedx509.a 相关源文件
MBEDX509_SRCS=$(cd "$SRC_DIR/library" && ls x509*.c)

echo "编译 libmbedx509.a"
for file in $MBEDX509_SRCS; do
    echo "  -> $file"
    $CLANG $CFLAGS -c "$SRC_DIR/library/$file" -o "$OBJ_DIR/mbedx509/${file%.c}.o"
done
$TOOLCHAIN/bin/llvm-ar rcs "$LIB_DIR/libmbedx509.a" "$OBJ_DIR/mbedx509"/*.o

# 编译 libmbedcrypto.a 相关源文件（排除 ssl/x509 相关）
MBEDCRYPTO_SRCS=$(cd "$SRC_DIR/library" && ls *.c | grep -v -E 'ssl_|x509|psa_|net_sockets|timing|version')

echo "编译 libmbedcrypto.a"
for file in $MBEDCRYPTO_SRCS; do
    echo "  -> $file"
    $CLANG $CFLAGS -c "$SRC_DIR/library/$file" -o "$OBJ_DIR/mbedcrypto/${file%.c}.o"
done
$TOOLCHAIN/bin/llvm-ar rcs "$LIB_DIR/libmbedcrypto.a" "$OBJ_DIR/mbedcrypto"/*.o

echo "编译完成，输出库位于: $LIB_DIR"
ls -lh "$LIB_DIR"

```

**技术难点解析：**

*   **TLS 协议**: 完整实现 TLS 握手协议
*   **证书解析**: 使用 mbedTLS 解析 X.509 证书

#### 6. TEE 检测模块

##### 技术背景

TEE（Trusted Execution Environment）是 Android 设备的重要安全特性，它提供了一个与主操作系统隔离的安全执行环境。TEE 检测能够从硬件层面验证设备的安全状态，通过解析 Android KeyStore Attestation 证书来获取设备的可信启动状态、安全级别等关键信息。

##### 核心原理

TEE 检测基于 Android KeyStore 的 Attestation 功能，通过生成的密钥对，获取包含设备安全状态的证书链，然后使用纯 C++ 实现解析证书中的 TEE 扩展信息，避免依赖 Java 三方库，提高安全性和性能。

##### 实现流程

**5.1 获取 TEE 证书（JNI 桥接）**

```
// 通过JNI获取Android KeyStore Attestation证书
vector<uint8_t> get_attestation_cert_from_java(JNIEnv* env, jobject context) {
    LOGD("get_attestation_cert_from_java called");
    vector<uint8_t> result;
    LOGI("Start get_attestation_cert_from_java");
    if (!env || !context) {
        LOGE("env or context is null");
        return result;
    }

    // 1. KeyStore.getInstance("AndroidKeyStore")
    jclass clsKeyStore = env->FindClass("java/security/KeyStore");
    LOGD("FindClass KeyStore: %p", clsKeyStore);
    jmethodID midGetInstance = env->GetStaticMethodID(clsKeyStore, "getInstance", "(Ljava/lang/String;)Ljava/security/KeyStore;");
    LOGD("GetMethodID getInstance: %p", midGetInstance);
    jstring jAndroidKeyStore = env->NewStringUTF("AndroidKeyStore");
    jobject keyStore = env->CallStaticObjectMethod(clsKeyStore, midGetInstance, jAndroidKeyStore);
    LOGD("keyStore: %p", keyStore);

    // 2. keyStore.load(null)
    jmethodID midLoad = env->GetMethodID(clsKeyStore, "load", "(Ljava/security/KeyStore$LoadStoreParameter;)V");
    LOGD("GetMethodID load: %p", midLoad);
    env->CallVoidMethod(keyStore, midLoad, (jobject)NULL);
    LOGD("keyStore.load(null) called");

    // 3. KeyGenParameterSpec.Builder(alias, purpose)
    jclass clsKeyGenBuilder = env->FindClass("android/security/keystore/KeyGenParameterSpec$Builder");
    LOGD("FindClass KeyGenParameterSpec$Builder: %p", clsKeyGenBuilder);
    jstring jAlias = env->NewStringUTF("tee_check_key");
    jclass clsKeyProperties = env->FindClass("android/security/keystore/KeyProperties");
    LOGD("FindClass KeyProperties: %p", clsKeyProperties);
    jfieldID fidPurposeSign = env->GetStaticFieldID(clsKeyProperties, "PURPOSE_SIGN", "I");
    jfieldID fidPurposeVerify = env->GetStaticFieldID(clsKeyProperties, "PURPOSE_VERIFY", "I");
    jint purpose = env->GetStaticIntField(clsKeyProperties, fidPurposeSign) | env->GetStaticIntField(clsKeyProperties, fidPurposeVerify);
    LOGD("purpose: %d", purpose);
    jmethodID midBuilderCtor = env->GetMethodID(clsKeyGenBuilder, "<init>", "(Ljava/lang/String;I)V");
    jobject builder = env->NewObject(clsKeyGenBuilder, midBuilderCtor, jAlias, purpose);
    LOGD("builder: %p", builder);

    // 4. setAlgorithmParameterSpec(new ECGenParameterSpec("secp256r1"))
    jclass clsECGenParamSpec = env->FindClass("java/security/spec/ECGenParameterSpec");
    LOGD("FindClass ECGenParamSpec: %p", clsECGenParamSpec);
    jmethodID midECGenCtor = env->GetMethodID(clsECGenParamSpec, "<init>", "(Ljava/lang/String;)V");
    jstring jCurve = env->NewStringUTF("secp256r1");
    jobject ecSpec = env->NewObject(clsECGenParamSpec, midECGenCtor, jCurve);
    LOGD("ecSpec: %p", ecSpec);
    jmethodID midSetAlgParam = env->GetMethodID(clsKeyGenBuilder, "setAlgorithmParameterSpec", "(Ljava/security/spec/AlgorithmParameterSpec;)Landroid/security/keystore/KeyGenParameterSpec$Builder;");
    builder = env->CallObjectMethod(builder, midSetAlgParam, ecSpec);
    LOGD("builder after setAlgorithmParameterSpec: %p", builder);

    // 5. setDigests(KeyProperties.DIGEST_SHA256)
    jfieldID fidDigestSHA256 = env->GetStaticFieldID(clsKeyProperties, "DIGEST_SHA256", "Ljava/lang/String;");
    jstring jDigestSHA256 = (jstring)env->GetStaticObjectField(clsKeyProperties, fidDigestSHA256);
    jobjectArray digestArray = env->NewObjectArray(1, env->FindClass("java/lang/String"), nullptr);
    env->SetObjectArrayElement(digestArray, 0, jDigestSHA256);
    jmethodID midSetDigests = env->GetMethodID(clsKeyGenBuilder, "setDigests", "([Ljava/lang/String;)Landroid/security/keystore/KeyGenParameterSpec$Builder;");
    builder = env->CallObjectMethod(builder, midSetDigests, digestArray);
    LOGD("builder after setDigests: %p", builder);

    // 6. setAttestationChallenge
    const char* challengeStr = "tee_check";
    jbyteArray challenge = env->NewByteArray(strlen(challengeStr));
    env->SetByteArrayRegion(challenge, 0, strlen(challengeStr), (const jbyte*)challengeStr);
    jmethodID midSetChallenge = env->GetMethodID(clsKeyGenBuilder, "setAttestationChallenge", "([B)Landroid/security/keystore/KeyGenParameterSpec$Builder;");
    builder = env->CallObjectMethod(builder, midSetChallenge, challenge);
    LOGD("builder after setAttestationChallenge: %p", builder);

    // 继续生成密钥对并获取证书...
    return result;
}

```

**5.2 C++ ASN.1 解析器实现**

```
// TEE attestation extension OID
#define TEE_ATTESTATION_OID "1.3.6.1.4.1.11129.2.1.17"

// ASN.1 object structure
typedef struct {
    int tag;
    int tag_class;
    int is_constructed;
    int length;
    const unsigned char *data;
    int data_len;
} asn1_object_t;

// Forward declarations
static int parse_asn1_tag(const unsigned char *data, int len, int *offset, asn1_object_t *obj);
static int parse_asn1_length(const unsigned char *data, int len, int *offset);
static int parse_asn1_tag_number(const unsigned char *data, int len, int *offset, int tag);
static int parse_x509_certificate(const unsigned char *data, int len, tee_info_t *tee_info);
static int parse_attestation_record(const unsigned char *data, int len, tee_info_t *tee_info);
static int find_root_of_trust_in_authorization_list(const unsigned char *data, int len,
                                                    root_of_trust_t *root_of_trust);
static int parse_root_of_trust_sequence(const unsigned char *data, int len, root_of_trust_t *root_of_trust);

// 解析ASN.1标签
static int parse_asn1_tag(const unsigned char *data, int len, int *offset, asn1_object_t *obj) {
    if (*offset >= len) return -1;

    unsigned char tag_byte = data[*offset];
    obj->tag_class = tag_byte & 0xC0;
    obj->is_constructed = (tag_byte & 0x20) ? 1 : 0;
    obj->tag = tag_byte & 0x1F;

    (*offset)++;

    // 处理长标签
    if (obj->tag == 0x1F) {
        obj->tag = parse_asn1_tag_number(data, len, offset, 0x1F);
    }

    return 0;
}

// 解析ASN.1长度
static int parse_asn1_length(const unsigned char *data, int len, int *offset) {
    if (*offset >= len) return -1;

    unsigned char length_byte = data[*offset];
    (*offset)++;

    if (length_byte & 0x80) {
        // 长长度格式
        int num_bytes = length_byte & 0x7F;
        int length = 0;
        for (int i = 0; i < num_bytes; i++) {
            if (*offset >= len) return -1;
            length = (length << 8) | data[*offset];
            (*offset)++;
        }
        return length;
    } else {
        // 短长度格式
        return length_byte;
    }
}

```

**5.3 TEE 扩展解析**

```
// 解析X.509证书中的TEE扩展
static int parse_x509_certificate(const unsigned char *data, int len, tee_info_t *tee_info) {
    LOGD("parse_x509_certificate: called with len=%d", len);

    int offset = 0;
    asn1_object_t obj;

    // 解析证书序列
    if (parse_asn1_tag(data, len, &offset, &obj) != 0) {
        LOGE("parse_x509_certificate: failed to parse certificate sequence tag");
        return -1;
    }
    if (obj.tag != ASN1_SEQUENCE || !obj.is_constructed) {
        LOGE("parse_x509_certificate: expected constructed sequence");
        return -1;
    }

    int cert_len = parse_asn1_length(data, len, &offset);
    int cert_end = offset + cert_len;
    LOGD("parse_x509_certificate: certificate length=%d", cert_len);

    // 解析TBSCertificate
    if (parse_asn1_tag(data, len, &offset, &obj) != 0) {
        LOGE("parse_x509_certificate: failed to parse TBSCertificate tag");
        return -1;
    }
    if (obj.tag != ASN1_SEQUENCE || !obj.is_constructed) {
        LOGE("parse_x509_certificate: expected constructed TBSCertificate sequence");
        return -1;
    }

    int tbs_len = parse_asn1_length(data, len, &offset);
    int tbs_end = offset + tbs_len;
    LOGD("parse_x509_certificate: TBSCertificate length=%d", tbs_len);

    // 跳过版本、序列号、签名算法、颁发者、有效期、主题、主题公钥信息
    // 直接查找扩展部分
    while (offset < tbs_end) {
        if (parse_asn1_tag(data, len, &offset, &obj) != 0) break;

        if (obj.tag == 3 && obj.is_constructed) { // 扩展
            int ext_len = parse_asn1_length(data, len, &offset);
            int ext_end = offset + ext_len;
            LOGD("parse_x509_certificate: found extensions, length=%d", ext_len);

            // 解析扩展序列
            while (offset < ext_end) {
                if (parse_asn1_tag(data, len, &offset, &obj) != 0) break;
                if (obj.tag != ASN1_SEQUENCE || !obj.is_constructed) break;

                int seq_len = parse_asn1_length(data, len, &offset);
                int seq_end = offset + seq_len;

                // 解析扩展OID
                if (parse_asn1_tag(data, len, &offset, &obj) != 0) break;
                if (obj.tag == ASN1_OBJECT_IDENTIFIER) {
                    int oid_len = parse_asn1_length(data, len, &offset);
                    char oid_str[256];
                    parse_oid_to_string(data + offset, oid_len, oid_str, sizeof(oid_str));

                    LOGD("parse_x509_certificate: found extension OID: %s", oid_str);

                    if (strcmp(oid_str, TEE_ATTESTATION_OID) == 0) {
                        LOGI("parse_x509_certificate: found TEE attestation extension");
                        // 找到TEE扩展，解析其值
                        offset += oid_len;
                        if (parse_asn1_tag(data, len, &offset, &obj) != 0) break;
                        if (obj.tag == ASN1_OCTET_STRING_TAG) {
                            int ext_value_len = parse_asn1_length(data, len, &offset);
                            LOGD("parse_x509_certificate: TEE extension value length=%d", ext_value_len);
                            return parse_attestation_record(data + offset, ext_value_len, tee_info);
                        }
                    } else {
                        offset += oid_len;
                    }
                }
                offset = seq_end;
            }
        } else {
            int skip_len = parse_asn1_length(data, len, &offset);
            offset += skip_len;
        }
    }

    LOGW("parse_x509_certificate: TEE extension not found");
    return -1; // 未找到TEE扩展
}

```

**5.4 RootOfTrust 信息解析**

```
// Keymaster标签定义
#define KM_TAG_ROOT_OF_TRUST 0x000002C0
#define VERIFIED_BOOT_STATE_VERIFIED 0
#define VERIFIED_BOOT_STATE_SELF_SIGNED 1
#define VERIFIED_BOOT_STATE_UNVERIFIED 2
#define VERIFIED_BOOT_STATE_FAILED 3

// 解析Attestation记录
static int parse_attestation_record(const unsigned char *data, int len, tee_info_t *tee_info) {
    int offset = 0;

    // 解析授权列表
    while (offset < len) {
        uint32_t tag = parse_keymaster_tag(data + offset, len - offset, &offset);
        if (tag == KM_TAG_ROOT_OF_TRUST) {
            return parse_root_of_trust_sequence(data + offset, len - offset, &tee_info->root_of_trust);
        }
        // 跳过其他标签
        int tag_len = parse_keymaster_value_length(data + offset, len - offset, &offset);
        offset += tag_len;
    }

    return -1;
}

// 解析RootOfTrust序列
static int parse_root_of_trust_sequence(const unsigned char *data, int len, root_of_trust_t *root_of_trust) {
    int offset = 0;
    asn1_object_t obj;

    // 解析序列
    if (parse_asn1_tag(data, len, &offset, &obj) != 0) return -1;
    if (obj.tag != ASN1_SEQUENCE || !obj.is_constructed) return -1;

    int seq_len = parse_asn1_length(data, len, &offset);
    int seq_end = offset + seq_len;

    // 解析VerifiedBootState
    if (parse_asn1_tag(data, len, &offset, &obj) != 0) return -1;
    if (obj.tag == ASN1_ENUMERATED) {
        int enum_len = parse_asn1_length(data, len, &offset);
        root_of_trust->verified_boot_state = parse_asn1_integer(data + offset, enum_len);
        offset += enum_len;
    }

    // 解析VerifiedBootKey
    if (parse_asn1_tag(data, len, &offset, &obj) != 0) return -1;
    if (obj.tag == ASN1_OCTET_STRING_TAG) {
        int key_len = parse_asn1_length(data, len, &offset);
        memcpy(root_of_trust->verified_boot_key, data + offset, 
               key_len > 32 ? 32 : key_len);
        offset += key_len;
    }

    // 解析DeviceLocked状态
    if (parse_asn1_tag(data, len, &offset, &obj) != 0) return -1;
    if (obj.tag == ASN1_BOOLEAN) {
        int bool_len = parse_asn1_length(data, len, &offset);
        root_of_trust->device_locked = (data[offset] != 0);
        offset += bool_len;
    }

    return 0;
}

```

**5.5 主解析函数**

```
int parse_tee_certificate(const unsigned char *cert_data, int cert_len, tee_info_t *tee_info) {
    LOGD("parse_tee_certificate: parsing certificate of length %d", cert_len);

    // 初始化TEE信息结构
    memset(tee_info, 0, sizeof(tee_info_t));

    // 解析X.509证书
    int ret = parse_x509_certificate(cert_data, cert_len, tee_info);
    if (ret != 0) {
        LOGE("Failed to parse X.509 certificate");
        return -1;
    }

    // 验证解析结果
    LOGI("TEE Certificate parsed successfully:");
    LOGI("  Verified Boot State: %d", tee_info->root_of_trust.verified_boot_state);
    LOGI("  Device Locked: %s", tee_info->root_of_trust.device_locked ? "true" : "false");

    // 计算VerifiedBootKey的SHA256指纹
    char key_fingerprint[65];
    bytes_to_hex(tee_info->root_of_trust.verified_boot_key, 32, key_fingerprint);
    LOGI("  Verified Boot Key: %s", key_fingerprint);

    return 0;
}

```

**技术优势：**

*   **纯 C++ 实现**：避免依赖 Java 三方库，减少攻击面
*   **性能优化**：直接解析，无 JNI 开销

**技术难点解析：**

*   **ASN.1 复杂性**：需要实现 ASN.1 DER 编码解析
    
*   **证书结构**：理解 X.509 证书和 TEE 扩展的复杂结构
    

#### 7. 标准 API 与自定义 API 的无缝切换机制

##### 设计原理

为了实现更高的防护能力，Overt 自实现了非标准的 string、vector、map 等基础 api，并通过`config.h`中的宏替换机制实现了标准 API 与非标 API 之间的无缝切换，这是一种编译时切换策略。

##### 实现流程

**7.1 宏控制机制**

```
// config.h 中的宏控制机制
// 通过这个宏来控制标准 API 和 非标准 API 的使用
// 定义 USE_NONSTD_API 时使用 nonstd 命名空间同时开启 nonstd_ 系列的宏替换
// 未定义时使用 std 命名空间

#define USE_NONSTD_API

#ifdef USE_NONSTD_API
    #include "nonstd_libc.h"
    #include "string.h"
    #include "vector.h"
    #include "map.h"

    using nonstd::string;
    using nonstd::vector;
    using nonstd::map;

#else
    // 当使用 std 命名空间时，包含标准库头文件
    #include <string>
    #include <vector>
    #include <map>

    // 使用 std 命名空间（但避免与系统函数冲突）
    using std::string;
    using std::vector;
    using std::map;

#endif

```

**7.2 宏映射机制**

```
// nonstd_libc.h 中的宏映射
extern "C" {
// ==================== 字符串函数 ====================
int nonstd_strcmp(const char *str1, const char *str2);
size_t nonstd_strlen(const char *str);
char *nonstd_strcpy(char *dest, const char *src);
char *nonstd_strcat(char *dest, const char *src);
int nonstd_strncmp(const char *str1, const char *str2, size_t n);
char *nonstd_strrchr(const char *str, int character);
char *nonstd_strncpy(char *dst, const char *src, size_t n);
size_t nonstd_strlcpy(char *dst, const char *src, size_t siz);
char *nonstd_strstr(const char *s, const char *find);
char *nonstd_strchr(const char *p, int ch);

// ==================== 内存函数 ====================
void *nonstd_malloc(size_t size);
void nonstd_free(void *ptr);
void *nonstd_calloc(size_t nmemb, size_t size);
void *nonstd_realloc(void *ptr, size_t size);
void *nonstd_memset(void *dst, int val, size_t count);
void *nonstd_memcpy(void *dst, const void *src, size_t len);
int nonstd_memcmp(const void *s1, const void *s2, size_t n);

// ==================== 文件操作函数 ====================
int nonstd_open(const char *pathname, int flags, ...);
int nonstd_close(int fd);
ssize_t nonstd_read(int fd, void *buf, size_t count);
ssize_t nonstd_write(int fd, const void *buf, size_t count);
int nonstd_fstat(int __fd, struct stat *__buf);
off_t nonstd_lseek(int __fd, off_t __offset, int __whence);
ssize_t nonstd_readlinkat(int __dir_fd, const char *__path, char *__buf, size_t __buf_size);
int nonstd_access(const char *pathname, int mode);
int nonstd_stat(const char *pathname, struct stat *buf);

// ==================== 网络函数 ====================
int nonstd_socket(int domain, int type, int protocol);
int nonstd_connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int nonstd_bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int nonstd_listen(int sockfd, int backlog);
int nonstd_accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// ==================== 时间函数 ====================
time_t nonstd_time(time_t *tloc);
int nonstd_gettimeofday(struct timeval *tv, struct timezone *tz);

// ==================== 进程函数 ====================
pid_t nonstd_getpid(void);
pid_t nonstd_getppid(void);

// ==================== 信号函数 ====================
int nonstd_kill(pid_t pid, int sig);

// ==================== 其他常用函数 ====================
int nonstd_atoi(const char *nptr);
long nonstd_atol(const char *nptr);

// ==================== 扩展系统函数 ====================
int nonstd_nanosleep(const struct timespec *__request, struct timespec *__remainder);
int nonstd_mprotect(void *__addr, size_t __size, int __prot);
int nonstd_inotify_init1(int flags);
int nonstd_inotify_add_watch(int __fd, const char *__path, uint32_t __mask);
int nonstd_inotify_rm_watch(int __fd, uint32_t __watch_descriptor);
int nonstd_tgkill(int __tgid, int __tid, int __signal);
void nonstd_exit(int __status);

ssize_t nonstd_readlink(const char *pathname, char *buf, size_t bufsiz);
struct tm *nonstd_localtime(const time_t *timep);
int nonstd_stat(const char *__path, struct stat *__buf);
int nonstd_access(const char *__path, int __mode);
}

#ifdef USE_NONSTD_API
// 使用非标准库实现
#define nonstd_strcmp strcmp
#define nonstd_strlen strlen
#define nonstd_strcpy strcpy
#define nonstd_strcat strcat
#define nonstd_strncmp strncmp
#define nonstd_strrchr strrchr
#define nonstd_strncpy strncpy
#define nonstd_strlcpy strlcpy
#define nonstd_strstr strstr
#define nonstd_strchr strchr
#define nonstd_malloc malloc
#define nonstd_free free
#define nonstd_calloc calloc
#define nonstd_realloc realloc
#define nonstd_memset memset
#define nonstd_memcpy memcpy
#define nonstd_memcmp memcmp
#define nonstd_open open
#define nonstd_close close
#define nonstd_read read
#define nonstd_write write
#define nonstd_fstat fstat
#define nonstd_lseek lseek
#define nonstd_readlinkat readlinkat
#define nonstd_access access
#define nonstd_stat stat
#define nonstd_socket socket
#define nonstd_connect connect
#define nonstd_bind bind
#define nonstd_listen listen
#define nonstd_accept accept
#define nonstd_time time
#define nonstd_gettimeofday gettimeofday
#define nonstd_getpid getpid
#define nonstd_getppid getppid
#define nonstd_kill kill
#define nonstd_atoi atoi
#define nonstd_atol atol
#define nonstd_nanosleep nanosleep
#define nonstd_mprotect mprotect
#define nonstd_inotify_init1 inotify_init1
#define nonstd_inotify_add_watch inotify_add_watch
#define nonstd_inotify_rm_watch inotify_rm_watch
#define nonstd_tgkill tgkill
#define nonstd_exit exit
#define nonstd_readlink readlink
#define nonstd_localtime localtime
#endif

```

**7.3 自定义 API 实现特点**

```
// 自定义strcmp实现示例
int nonstd_strcmp(const char *str1, const char *str2) {
    LOGV("nonstd_strcmp called: str1='%s', str2='%s'", str1 ? str1 : "NULL", str2 ? str2 : "NULL");

    // 处理NULL指针
    if (!str1 && !str2) {
        LOGV("Both strings are NULL, returning 0");
        return 0;
    }
    if (!str1) {
        LOGV("str1 is NULL, returning -1");
        return -1;
    }
    if (!str2) {
        LOGV("str2 is NULL, returning 1");
        return 1;
    }

    // 手动实现strcmp逻辑
    int i = 0;
    while (str1[i] != '\0' && str2[i] != '\0') {
        if (str1[i] != str2[i]) {
            int result = (unsigned char)str1[i] - (unsigned char)str2[i];
            LOGV("Strings differ at position %d: '%c' vs '%c', returning %d", i, str1[i], str2[i], result);
            return result;
        }
        i++;
    }

    // 检查字符串长度
    if (str1[i] == '\0' && str2[i] == '\0') {
        LOGV("Strings are identical, returning 0");
        return 0;
    } else if (str1[i] == '\0') {
        LOGV("str1 is shorter, returning -1");
        return -1;
    } else {
        LOGV("str2 is shorter, returning 1");
        return 1;
    }
}

// 自定义open实现示例
int nonstd_open(const char *pathname, int flags, ...) {
    LOGE("nonstd_open called: path, pathname ? pathname : "NULL", flags);

    if (!pathname) {
        LOGV("open: NULL pathname");
        return -1;
    }

    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list args;
        va_start(args, flags);
        mode = va_arg(args, mode_t);
        va_end(args);
    }

    // 使用 openat 系统调用，AT_FDCWD 表示相对于当前工作目录
    int fd = (int)__syscall4(SYS_openat, AT_FDCWD, (long)pathname, flags, mode);
    LOGV("open: returned fd=%d", fd);
    return fd;
}

// 自定义access实现示例
int nonstd_access(const char* __path, int __mode) {
    LOGV("nonstd_access called: path='%s', mode=0x%x", __path ? __path : "NULL", __mode);

    if (!__path) {
        LOGV("access: NULL path");
        errno = EFAULT;  // 错误地址
        return -1;
    }

    long result = __syscall2(SYS_access, (uintptr_t)__path, (long)__mode);

    if (result < 0) {
        errno = -result;
        LOGV("access: failed, errno=%d", errno);
        return -1;
    }

    LOGV("access: success");
    return 0;
}

```

**技术优势：**

*   **编译时确定**：无运行时开销，性能最优
*   **代码结构清晰**：通过宏定义实现，易于维护和理解
*   **灵活切换**：只需修改`USE_NONSTD_API`宏即可切换实现策略
*   **安全检查**：自定义实现可以添加额外的安全检查和日志记录

**技术难点解析：**

*   **宏定义管理**：需要确保所有 API 都有对应的宏定义
*   **函数签名一致性**：自定义实现必须与标准库函数签名完全一致
*   **性能平衡**：在添加安全检查的同时保持性能

### 结语

其实是否选择开源还是纠结了挺长时间的，因为最开始是打算开源的，但写着写着就突然理解了为什么例如 NativeTest、hunter、momo 等检测项目为什么选择闭源，确实倾注了很多心血。每一个技术细节的打磨，每一次检测逻辑的优化，从单一功能到完整的检测体系，每一个功能都来之不易。希望本项目能为其它同学提供有价值的技术参考，无论选择开源还是闭源，最重要的是保持对技术的热爱和对安全的执着，这才是行业发展的真正动力。

### 项目地址

[https://github.com/lxz-jiandan/Overt](https://github.com/lxz-jiandan/Overt)

### 参考资料

[[原创] 自动化采集 Android 系统级设备指纹对抗 & 如何四两拨千斤?](https://bbs.kanxue.com/thread-281889.htm)

[[原创]Android 风控详细解读以及对照工具](https://bbs.kanxue.com/thread-286120.htm)

[珍惜 hunter]()

[https://github.com/Mbed-TLS/mbedtls](https://github.com/Mbed-TLS/mbedtls)

[https://github.com/vvb2060/KeyAttestation](https://github.com/vvb2060/KeyAttestation)

[https://github.com/vvb2060/XposedDetector](https://github.com/vvb2060/XposedDetector)

[https://github.com/byxiaorun/Ruru](https://github.com/byxiaorun/Ruru)

[https://github.com/ac3ss0r/obfusheader.h](https://github.com/ac3ss0r/obfusheader.h)

![](https://avatar.52pojie.cn/data/avatar/001/26/93/39_avatar_middle.jpg)快乐的小跳蛙 太强了，有时间仔细学习下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) djgsdj 支持学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) uyj666 支持学习一下![](https://avatar.52pojie.cn/images/noavatar_middle.gif)b686jT5D6mWe 太强了楼主，打卡学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Arne23 感谢分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zhangsf123 写的真不错。加油。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)dlovec 学习更安全的知识![](https://static.52pojie.cn/static/image/smiley/default/40.gif)![](https://static.52pojie.cn/static/image/smiley/default/40.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xin1you1di1 太厉害了，学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) huoxin123 太强了，向大佬学习