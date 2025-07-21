> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287579.htm)

> [原创] spawn 模式注入 so 的实现

注入篇二 spawn 模式的实现
================

前言：在[上一篇文章](elink@838K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6&6N6i4g2C8K9g2)9J5k6h3y4G2L8$3I4Q4x3V1j5J5x3o6t1#2i4K6u0r3x3o6N6Q4x3V1j5I4x3W2)9J5c8V1W2F1K9X3g2U0N6q4y4G2i4K6u0r3)中大概解释了 attach 模式注入 so 的实现思路，但是实战中 spawn 模式注入也是很常用的

0x01 什么是 spawn 注入模式
-------------------

我的理解是 在目标进程**启动时**对其进行注入。这可以帮助我们绕过部分反调试，同时可以监听一些初始化函数

0x02 大致的实现思路
------------

spawn 翻译过来是`产卵`的意思，刚接触 frida 的时候我不理解 spawn 这个单词和注入这个操作之间的联系，后来才知道这个操作是需要注入 zygote 的，而 zygote 翻译过来刚好是`受精卵`的意思，现在感觉还挺有意思的，spawn 这个名字确实很恰当

我们想要在进程启动时就注入，那么时机很重要，这个时机当然越早越好... 吗？我们知道 app 进程都是由 zygote 进程 fork 而来的，fork 之后才会有 app 进程，这里差不多就是 app 进程刚出生的时候了，我们在这时，把自己的 so 注入就可以实现上述的效果了

0x03 开始实现
---------

由于我们要 hook zygote 进程里的一些函数，所以首先要把 hook 模块注入进 zygote 进程，关于如何通过 ptrace 注入目标进程，[上一篇文章](elink@a22K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6&6N6i4g2C8K9g2)9J5k6h3y4G2L8$3I4Q4x3V1j5J5x3o6t1#2i4K6u0r3x3o6N6Q4x3V1j5I4x3W2)9J5c8V1W2F1K9X3g2U0N6q4y4G2i4K6u0r3)已经讲的比较清晰了，这里就不过多赘述了，这里先来讲讲 hook 模块应该实现哪些功能吧。这里的 hook 框架依旧是使用 dobby

**首先要明确 hook 点**

上面说了，我们是 hook fork 函数，所以目标 so 是`/system/lib64/libc.so`，查找函数地址可以先 dlopen libc.so，然后再 dlsym fork 函数，这里 dlopen 是为了拿到对应 so 的 handle，正常进程里 libc.so 99% 是已经被加载过的，你再打开一次也无所谓，因为 linker 会直接返回 solist 里的 handle 给你，不会浪费很多时间。当然也可以使用 elf_utils 解析磁盘上的 libc.so 拿到偏移，然后查找基地址两个相加得到真实的地址

**其次编写 hook 函数**

```
static std::string getCurrentProcessName() {
    char process_name[64] = {0};
    int fd = open("/proc/self/cmdline", O_RDONLY);
    if (fd != -1) {
        ssize_t len = read(fd, process_name, sizeof(process_name) - 1);
        if (len > 0) {
            process_name[len] = '\0';
        }
        close(fd);
    }
    return std::string(process_name);
}
 
static pid_t hooked_fork() {
    if (in_hook_fork) {
        return orig_fork();
    }
 
    LOGD("enter fork func!");
 
    pid_t result = orig_fork();
 
    if (result == 0) {
        LOGD("enter child process");
        std::string current_process = getCurrentProcessName();
        if (!current_process.empty()) {
            LOGD("Child process started: %s", current_process.c_str());
            //todo 然后判断进程名是否为目标进程，后续再dlopen自己的so
        }
    }
 
    return result;
}

```

我刚开始是这么写的，我觉得没啥毛病，但是一运行就有问题了，各位可以先思考一下为什么这样不行哈哈哈哈

运行时输出了如下日志：

```
07-09 10:52:08.046 19197 19197 I [Zygote_Hook]: enter child process
07-09 10:52:08.047 19197 19197 I [Zygote_Hook]: Child process started: zygote64

```

子进程的进程名依然是`zygote64`，为什么会这样呢？？？其实是因为获取进程名的时机太早了，这时子进程还没初始化完全，所以获取到的名字还是父进程的

为了解决这个问题，我们就得先研究一下 android 中 app 进程的进程名是如何被设置的，查阅了一番资料后得知`android.os.Process.setArgv0`函数负责设置进程名，它是一个 native 函数，长这样

```
void android_os_Process_setArgV0(JNIEnv* env, jobject clazz, jstring name)
{
    if (name == NULL) {
        jniThrowNullPointerException(env, NULL);
        return;
    }
 
    const jchar* str = env->GetStringCritical(name, 0);
    String8 name8;
    if (str) {
        name8 = String8(reinterpret_cast(str),
                        env->GetStringLength(name));
        env->ReleaseStringCritical(name, str);
    }
 
    if (!name8.empty()) {
        AndroidRuntime::getRuntime()->setArgv0(name8.c_str(), true /* setProcName */);
    }
} 
```

它的第三个参数就是我们需要的进程名，所以我们可以在子进程中 hook 这个函数，当函数执行时取出参数，与我们的目标进程名作比较，如果一致，则加载我们自己的 so。加载 so 使用 dlopen 就行了，但是 dlopen 只能从那几个路径加载，虽然本文不涉及注入痕迹的隐藏，但简单的能直接避免还是就避免一下吧。如果从 app 私有目录加载，那么 app 自己是有权限检查该路径下的文件的，不是很安全。如果从 / system/lib 下加载，那每次注入不同的 so 都需要重启手机重新挂载文件，这太麻烦了。所以还是从 / data/local/tmp 下加载吧，当然这个路径也不安全，但相对好一点。但如果选择这个路径，就会有新的问题。普通 app 进程是没办法访问这个目录的，所以 dlopen 肯定会失败。解决方案也比较简单，把 selinux 设置为宽容模式即可。那么新的问题又来了，hook 代码是运行在 app 进程的，app 进程没有权限设置 selinux，这里只有 Injector 进程有这个权限，所以需要设计跨进程通信模块，用于在 dlopen 前后通知 Injector 模块修改 selinux 模式

**设计通信模块**

为了方便，本来想直接用信号的，因为这个通信场景本身也不复杂，但是后面写完才发现权限不够，于是又重写，这里直接用 UDS。为了后续开发方便，就把模块单拎出来写了，另外两个模块共享通信模块

```
#include "socket_comm.h"
#include #include #include #include #include #include #include #define LOG_TAG "[Socket_Comm]"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
 
namespace comm {
 
    SocketServer::SocketServer() : server_fd_(-1), running_(false), background_thread_(nullptr) {
    }
 
    SocketServer::~SocketServer() {
        Stop();
    }
 
    bool SocketServer::Start() {
        if (server_fd_ != -1) {
            LOGE("Server already started");
            return false;
        }
 
        unlink(COMM_SOCKET_PATH);
 
        // 创建Unix Domain Socket
        server_fd_ = socket(AF_UNIX, SOCK_STREAM, 0);
        if (server_fd_ < 0) {
            LOGE("Failed to create socket: %s", strerror(errno));
            return false;
        }
 
        struct sockaddr_un server_addr;
        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sun_family = AF_UNIX;
        strncpy(server_addr.sun_path, COMM_SOCKET_PATH, sizeof(server_addr.sun_path) - 1);
 
        if (bind(server_fd_, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
            LOGE("Failed to bind socket: %s", strerror(errno));
            close(server_fd_);
            server_fd_ = -1;
            return false;
        }
 
        // 设置socket文件权限，允许所有用户访问
        chmod(COMM_SOCKET_PATH, 0666);
 
        if (listen(server_fd_, 1) < 0) {
            LOGE("Failed to listen on socket: %s", strerror(errno));
            close(server_fd_);
            server_fd_ = -1;
            unlink(COMM_SOCKET_PATH);
            return false;
        }
 
        running_ = true;
        LOGI("Unix Domain Socket server started at %s", COMM_SOCKET_PATH);
        return true;
    }
 
    void SocketServer::Stop() {
        running_ = false;
 
        if (server_fd_ != -1) {
            shutdown(server_fd_, SHUT_RDWR);
            close(server_fd_);
            server_fd_ = -1;
        }
 
        unlink(COMM_SOCKET_PATH);
 
        if (background_thread_) {
            if (background_thread_->joinable()) {
                background_thread_->join();
            }
            delete background_thread_;
            background_thread_ = nullptr;
        }
 
        LOGI("Socket server stopped");
    }
 
    std::string SocketServer::WaitForMessage(int timeout_ms) {
        if (server_fd_ == -1) {
            LOGE("Server not started");
            return "";
        }
 
        fd_set readfds;
        FD_ZERO(&readfds);
        FD_SET(server_fd_, &readfds);
 
        struct timeval* tv_ptr = nullptr;
        struct timeval tv;
        if (timeout_ms >= 0) {
            tv.tv_sec = timeout_ms / 1000;
            tv.tv_usec = (timeout_ms % 1000) * 1000;
            tv_ptr = &tv;
        }
 
        int ret = select(server_fd_ + 1, &readfds, nullptr, nullptr, tv_ptr);
        if (ret <= 0) {
            if (ret < 0 && errno != EINTR) {
                LOGE("select error: %s", strerror(errno));
            }
            return "";
        }
 
        struct sockaddr_un client_addr;
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(server_fd_, (struct sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) {
            if (errno != EINTR && errno != EAGAIN) {
                LOGE("Failed to accept connection: %s", strerror(errno));
            }
            return "";
        }
 
        char buffer[256] = {0};
        ssize_t bytes_read = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
        close(client_fd);
 
        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            LOGI("Received message: %s", buffer);
            return std::string(buffer);
        }
 
        return "";
    }
 
    void SocketServer::RunInBackground(std::function callback) {
        if (background_thread_) {
            LOGE("Background thread already running");
            return;
        }
 
        background_thread_ = new std::thread(&SocketServer::AcceptLoop, this, callback);
    }
 
    void SocketServer::AcceptLoop(std::function callback) {
        LOGI("Background accept loop started");
 
        while (running_) {
            std::string msg = WaitForMessage(1000);
            if (!msg.empty() && callback) {
                callback(msg);
            }
        }
 
        LOGI("Background accept loop ended");
    }
 
    bool SocketClient::SendMessage(const std::string& message) {
        int client_fd = socket(AF_UNIX, SOCK_STREAM, 0);
        if (client_fd < 0) {
            LOGE("Failed to create client socket: %s", strerror(errno));
            return false;
        }
 
        struct sockaddr_un server_addr;
        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sun_family = AF_UNIX;
        strncpy(server_addr.sun_path, COMM_SOCKET_PATH, sizeof(server_addr.sun_path) - 1);
 
        if (connect(client_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
            LOGE("Failed to connect to server: %s (path: %s)", strerror(errno), COMM_SOCKET_PATH);
            close(client_fd);
            return false;
        }
 
        ssize_t bytes_sent = send(client_fd, message.c_str(), message.length(), 0);
        close(client_fd);
 
        if (bytes_sent < 0) {
            LOGE("Failed to send message: %s", strerror(errno));
            return false;
        }
 
        LOGI("Message sent: %s", message.c_str());
        return true;
    }
 
} 
```

这样就解决权限的问题了:) 是不是就可以加载 so 了呢，直接 dlopen(SO_PATH, FLAG)，那么`SO_PATH`从哪来？当然完善上面的跨进程通信模块，也能把 path 传过来，但是有点麻烦了，这里我的解决方案是在 hook 模块里导出一个函数用于被 Injector 模块远程调用，参数里传递一些必要的信息，比如`SO_PATH`

```
// 全局变量存储目标信息
static char g_target_process_name[256] = {0};
static char g_target_library_path[512] = {0};
 
extern "C" void setTargetInfo(char* target_process_name, char* target_library_path) {
    strcpy(g_target_process_name, target_process_name);
    strcpy(g_target_library_path, target_library_path);
    LOGI("Target process name: %s", g_target_process_name);
    LOGI("Target library path: %s", g_target_library_path);
}

```

**总的代码就是这样子的**

```
#include #include #include #include #include #include "utils.h"
#include "../ipc/socket_comm.h"
 
#define LIBC "/system/lib64/libc.so"
#define LIBANDROID_RUNTIME "/system/lib64/libandroid_runtime.so"
 
// 全局变量存储目标信息
static char g_target_process_name[256] = {0};
static char g_target_library_path[512] = {0};
 
typedef pid_t (*Fork_t)();
static Fork_t orig_fork = nullptr;
void* fork_addr = nullptr;
 
typedef void (*SetArgV0_t)(JNIEnv*, jobject, jstring);
static SetArgV0_t orig_setArgV0 = nullptr;
void* setArgV0_addr = nullptr;
 
static volatile int* inject_flag = nullptr;
static pid_t g_zygote_pid = 0;
 
static bool g_injection_done = false;
 
void unhook() {
    if(fork_addr)
        DobbyDestroy(fork_addr);
     
    if(setArgV0_addr)
        DobbyDestroy(setArgV0_addr);
 
    LOGI("All hooks removed");
}
 
extern "C" void cleanup_hooks() {
    unhook();
 
    if (inject_flag && inject_flag != MAP_FAILED) {
        LOGI("Cleaning up shared memory");
        munmap((void*)inject_flag, sizeof(int));
        inject_flag = nullptr;
    }
 
    memset(g_target_process_name, 0, sizeof(g_target_process_name));
    memset(g_target_library_path, 0, sizeof(g_target_library_path));
    g_injection_done = false;
 
    LOGI("cleanup_hooks completed");
}
 
static void hook_setArgV0(JNIEnv* env, jobject clazz, jstring name) {
    std::string process_name;
    if (name && env) {
        const char* name_utf = env->GetStringUTFChars(name, nullptr);
        if (name_utf) {
            process_name = name_utf;
            env->ReleaseStringUTFChars(name, name_utf);
            LOGD("Process name: %s", process_name.c_str());
        }
    }
 
    if (!process_name.empty() && process_name == g_target_process_name && !g_injection_done) {
        // 标记已完成，避免重复注入
        g_injection_done = true;
 
        // 在子进程中 unhook
        if(setArgV0_addr) {
            DobbyDestroy(setArgV0_addr);
            setArgV0_addr = nullptr;
        }
 
        LOGI("Target process detected: %s, injecting library: %s",
             process_name.c_str(), g_target_library_path);
 
        void* handle = dlopen(g_target_library_path, RTLD_NOW | RTLD_NODELETE | RTLD_GLOBAL);
 
        bool inject_success = (handle != nullptr);
        if (inject_success) {
            LOGI("Successfully loaded library: %s", g_target_library_path);
            comm::SocketClient::NotifySuccess();
        } else {
            LOGE("Failed to load library: %s, error: %s",
                 g_target_library_path, dlerror());
            comm::SocketClient::NotifyFailed();
        }
 
        if (inject_flag && inject_flag != MAP_FAILED) {
            *inject_flag = 1;
            LOGI("Set inject_flag to 1 in shared memory");
        }
    }
 
    if (orig_setArgV0) {
        orig_setArgV0(env, clazz, name);
    }
}
 
static void setup_setArgV0_hook() {
    void* handle = dlopen(LIBANDROID_RUNTIME, RTLD_NOW);
    if (!handle) {
        LOGE("Failed to open libandroid_runtime.so: %s", dlerror());
        return;
    }
 
    const char* symbol = "_Z27android_os_Process_setArgV0P7_JNIEnvP8_jobjectP8_jstring";
    setArgV0_addr = dlsym(handle, symbol);
    if (!setArgV0_addr) {
        LOGE("Failed to find %s: %s", symbol, dlerror());
        return;
    }
 
    LOGI("Found %s at %p", symbol, setArgV0_addr);
 
    orig_setArgV0 = (SetArgV0_t)InlineHook(setArgV0_addr, (void*)hook_setArgV0);
    if (orig_setArgV0) {
        LOGI("Successfully hooked %s", symbol);
    } else {
        LOGE("Failed to hook %s", symbol);
    }
}
 
static pid_t hook_fork() {
    LOGD("enter fork func!");
 
    pid_t result = orig_fork();
 
    if (result == 0) {
        LOGD("enter child process");
        setup_setArgV0_hook();
    } else if (result > 0) {
        if (getpid() == g_zygote_pid && inject_flag && inject_flag != MAP_FAILED && *inject_flag == 1) {
            LOGI("Injection completed in child process, unhooking fork in parent");
            if (fork_addr) {
                DobbyDestroy(fork_addr);
                fork_addr = nullptr;
            }
 
            munmap((void*)inject_flag, sizeof(int));
            inject_flag = nullptr;
        }
    }
 
    return result;
}
 
void setup_fork_hook() {
    void* handle = dlopen(LIBC, RTLD_NOW);
    if (!handle) {
        LOGE("Failed to open libc.so");
        return;
    }
 
    const char* fork_symbol = "fork";
    fork_addr = dlsym(handle, fork_symbol);
    if (!fork_addr) {
        LOGE("Failed to find fork");
        return;
    }
 
    LOGI("Found fork at %p", fork_addr);
 
    orig_fork = (Fork_t)InlineHook(fork_addr, (void*)hook_fork);
    if (orig_fork) {
        LOGI("Successfully hooked fork");
    } else {
        LOGE("Failed to hook fork");
    }
}
 
extern "C" void setTargetInfo(char* target_process_name, char* target_library_path) {
    strcpy(g_target_process_name, target_process_name);
    strcpy(g_target_library_path, target_library_path);
    LOGI("Target process name: %s", g_target_process_name);
    LOGI("Target library path: %s", g_target_library_path);
 
    g_injection_done = false;
 
    if (inject_flag && inject_flag != MAP_FAILED) {
        *inject_flag = 0;
        LOGI("Reset inject_flag to 0");
    }
 
    if (fork_addr == nullptr && orig_fork == nullptr) {
        LOGI("Re-hooking fork for new injection");
        setup_fork_hook();
    }
}
 
__attribute__((constructor))
void initialize_hook() {
    LOGI("Zygote hook module loaded");
 
    g_zygote_pid = getpid();
 
    inject_flag = (volatile int*)mmap(NULL, sizeof(int),
                                      PROT_READ | PROT_WRITE,
                                      MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (inject_flag != MAP_FAILED) {
        *inject_flag = 0;
        LOGI("Created shared memory for inject_flag");
    } else {
        LOGE("Failed to create shared memory");
        inject_flag = nullptr;
    }
 
    setup_fork_hook();
}
 
__attribute__((destructor))
void cleanup_hook() {
    LOGD("Cleaning up hooks in destructor");
    cleanup_hooks();
} 
```

自己写时记得及时 unhook，不然有的软件会进不去

0x04 小结
-------

在学校时，用完 frida 之后去食堂吃饭，付钱的软件老是打不开，当时不明白为什么我明明调试的是别的 app，这个 app 凭什么打不开，其实是用完之后 zygote 进程里的一些函数没有被 unhook，app 启动会 fork zygote 进程，所以就被检测到了:(

再来聊聊检测和过检测吧，就说我写的这个工具吧，如果不开源，可能就是 maps 里有可疑路径的 so，soinfo list 里也会有，`/proc/self/fd/` 中的文件描述符，还有就是通信时的痕迹，别的地方感觉也没啥了 (注入留下的痕迹，hook 的不归我管)。前面两个，目前用户层有的解决方案就是：

*   遍历 `/proc/<pid>/maps`，找到目标库的所有内存段
*   对每个段，远程分配匿名内存、远程 memcpy 覆盖内容、调用 `mremap` 将匿名内存映射到原来的地址
*   最后还原原有的内存保护权限

但这样还是解决不了问题的，因为正常 app 基本没用可执行的匿名内存段，但是一些加固之后的似乎有，我之前看 36O 是有的

solist 里的痕迹就是通过找到 [solist_remove_soinfo](elink@6ecK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0M7#2)9J5k6h3q4F1k6s2u0G2K9h3c8Q4x3X3g2U0L8$3#2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6K6N6i4m8W2M7Y4m8J5L8$3A6W2j5%4c8Q4x3V1k6E0j5h3W2F1i4K6u0r3i4K6u0n7i4K6u0r3L8h3q4A6L8W2)9K6b7h3u0A6L8$3&6A6j5#2)9J5c8X3I4A6L8X3E0W2M7W2)9J5c8X3I4A6L8X3E0W2M7W2)9#2k6X3#2S2K9h3&6Q4x3X3g2U0M7s2m8Q4x3@1u0D9i4K6y4p5z5e0W2Q4x3@1k6I4i4K6y4p5M7$3!0D9K9i4y4@1i4K6g2X3M7X3g2E0L8%4k6W2i4K6g2X3M7$3!0A6L8X3k6G2) 函数的地址，然后调用它，移除目标 so 的 soinfo，当然这里还涉及获取目标 so 的 soinfo，遍历 solist 等操作，不赘述了

使用[自定义 linker](elink@1a8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6e0L8%4W2n7k6h3q4F1e0h3W2D9K9%4S2Q4x3V1k6K6L8@1I4G2j5h3c8W2M7R3`.`.) 的方案可以在 so 的路径上多一些选择，而且不会在 solist 留下痕迹，但也并非一点痕迹没有的:( 所以还是配合[内核模块](elink@a97K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6e0L8%4W2n7k6h3q4F1e0h3W2D9K9%4S2Q4x3V1k6X3k6h3q4@1N6i4u0W2)一起用吧

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 2025-7-11 19:02 被 Yangser 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码框架](forum-161-1-127.htm)