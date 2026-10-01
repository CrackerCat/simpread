> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293118.htm#msg_header_h3_4)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

本文记录了基础的`Frida`和`Root`的环境检测方案和绕过方案，本人学习时间实在不长，若文章有纰漏，望各位高手们找出后及时提出，我会及时编辑，学习，修改，谢谢各位

在多数教程中，都指导把`Frida-server`推送到`/data/local/tmp`中，并使用`27042`或者`27043`端口的`TCP`连接，应用层采取类`D-BUS`协议，传输`RPC`调用，再与`frida-server`建立通信，最终由`frida-server`完成对目标进程的注入与脚本执行。`Spawn` 模式下，`Frida` 通常先请求系统创建目标进程并使其保持暂停状态，然后完成附加和脚本加载，最后恢复进程执行，因此，它通常早于多数 `Application.onCreate()` 及业务检测，但无法覆盖 `Zygote` 阶段、链接器初始化以及部分极早期 `native` 初始化

我们从启动点的源代码开始分析

```
#https://github.com/frida/frida-tools/blob/main/frida_tools/application.py L652

elif target_type == "file":
    argv = target_value
    if not self._quiet:
     self._update_status(f"Spawning `{' '.join(argv)}`...")

 aux_kwargs = {}
    if self._aux is not None:
     aux_kwargs = dict([parse_typed_option(o) for o in self._aux])

    self._spawned_pid = self._device.spawn(argv, stdio=self._stdio, **aux_kwargs)
    self._spawned_argv = argv
    attach_target = self._spawned_pid
else:
 attach_target = target_value
    if not isinstance(attach_target, numbers.Number):
     attach_target = self._device.get_process(attach_target).pid
    if not self._quiet:
     self._update_status("Attaching...")
spawning = False
self._attach(attach_target)
```

分析进程注入的源代码可以得知，不论是`spawn`还是`attach`都最终调用了`attach`，不过`spawn`是先挂起进程，再使用`attach`附加

他们最终都是通过`perform_attach_to()`函数实现`frida-agent`注入

```
protected override async Future<IOStream> perform_attach_to (uint pid, HashTable<string, Variant> options,
                                                             Cancellable? cancellable, out Object? transport) throws Error, IOError {
    uint id;
    string entrypoint = "frida_agent_main";#固定入口点
    string parameters = make_agent_parameters (pid, "", options);#收集参数
    AgentFeatures features = CONTROL_CHANNEL;#建立对话通道
    var linjector = (Linjector) injector;#注入器
    // frida-agent注入
#if HAVE_EMBEDDED_ASSETS 从内存资源注入
    id = yield linjector.inject_library_resource (pid, agent, entrypoint, parameters, features, cancellable);
#else 从磁盘上的 frida-agent.so 路径注入
    id = yield linjector.inject_library_file_with_template (pid, PathTemplate (Config.FRIDA_AGENT_PATH), entrypoint,
                                                            parameters, features, cancellable);
#endif
    injectee_by_pid[pid] = id;#返回id

    var stream_request = new Promise<IOStream> ();#开一条双向通信流
    IOStream stream = yield linjector.request_control_channel (id, cancellable);
    stream_request.resolve (stream);

    transport = null;

    return stream_request.future;
}
```

下一步我们看看`Linjector`的具体实现

```
# src\linux\linjector.vala L84
public async uint inject_library_resource (uint pid, AgentDescriptor agent, string entrypoint, string data,
                                           AgentFeatures features, Cancellable? cancellable) throws Error, IOError {
 # 系统支持memfd（匿名内存文件）
    if (MemoryFileDescriptor.is_supported ()) {
        // 优先走内存文件描述符方式。memfd 可以创建一个匿名内存文件，不需要把 .so 真正落地到普通磁盘路径。
        unowned string arch_name = arch_name_from_pid (pid);
        // 根据架构构建agent名称并从内嵌资源中查找
        string name = agent.name_template.expand (arch_name);
        AgentResource? resource = agent.resources.first_match (r => r.name == name);
        if (resource == null) {
            throw new Error.NOT_SUPPORTED ("Unable to handle %s-bit processes due to build configuration",
                                           arch_name);
        }
        // 把内嵌 agent 资源转换成 memfd，然后交给 inject_library_fd() 注入目标进程。
        return yield inject_library_fd (pid, resource.get_memfd (), entrypoint, data, features, cancellable);
    }
 # 系统不支持memfd
    ensure_tempdir_prepared ();//准备临时目录
    // 与HAVE_EMBEDDED_ASSETS为false的情况相同
    // 通过 agent.get_path_template() 把内嵌 agent 写成临时 .so 文件，再按文件路径方式注入。
    return yield inject_library_file_with_template (pid, agent.get_path_template (), entrypoint, data, features,
                                                    cancellable);
}
# src\linux\linjector.vala L48
public async uint inject_library_file_with_template (uint pid, PathTemplate path_template, string entrypoint, string data,
                                                     AgentFeatures features, Cancellable? cancellable) throws Error, IOError {
    string path = path_template.expand (arch_name_from_pid (pid));
    // 打开agent文件
    int fd = Posix.open (path, Posix.O_RDONLY);
    if (fd == -1)
        throw new Error.INVALID_ARGUMENT ("Unable to open library: %s", strerror (errno));
    var library_so = new UnixInputStream (fd, true);
 # 最终调用inject_library_fd()
    return yield inject_library_fd (pid, library_so, entrypoint, data, features, cancellable);
}
```

根据是否支持匿名内存的方式来确认`agent`是否要落地到磁盘上，他们最终都都调用了`inject_library_fd`

```
public async uint inject_library_fd (uint pid, UnixInputStream library_so, string entrypoint, string data,
                                     AgentFeatures features, Cancellable? cancellable) throws Error, IOError {
    uint id = next_injectee_id++;
    yield helper.inject_library (pid, library_so, entrypoint, data, features, id, cancellable);
    #又调用了 helper.inject_library 继续跟踪查询
    pid_by_id[id] = pid;

    return id;
}
```

```
public async void inject_library (uint pid, UnixInputStream library_so, string entrypoint, string data,
                                  AgentFeatures features, uint id, Cancellable? cancellable) throws Error, IOError {
    var spec = new InjectSpec (library_so, entrypoint, data, features, id);
    var task = new InjectTask (this, spec);
    RemoteAgent agent = yield perform (task, pid, cancellable);#perform()函数最终调用的是InjectTask对象的run()方法
    take_agent (agent);
}
```

```
public async RemoteAgent run (uint pid, Cancellable? cancellable) throws Error, IOError {
    PausedSyscallSession? pss = backend.paused_syscalls[pid];
    if (pss != null)
        yield pss.interrupt (cancellable);
    # 重点
    var session = yield InjectSession.open (pid, cancellable);
    RemoteAgent agent = yield session.inject (spec, cancellable);
    if (session.was_group_stopped)
        backend.suspended_by_inject[pid] = session;
    else
        session.close ();
    return agent;
}
```

`InjectSession.open()`会通过 `ptrace` `attach` 到目标进程并保存寄存器状态。然后 `session.inject()` 才是真正做注入工作的

```
InjectSession.open(pid)
 -> SeizeSession.init_async()
  -> ptrace(SEIZE/ATTACH)
        -> 中断/等待目标线程停止
        -> get_regs(&saved_regs)
```

```
public async RemoteAgent inject (InjectSpec spec, Cancellable? cancellable) throws Error, IOError {
    string fallback_address = make_fallback_address ();
    #计算要写入目标进程的内存布局
    LoaderLayout loader_layout = compute_loader_layout (spec, fallback_address);
    BootstrapResult bootstrap_result = yield bootstrap (loader_layout.size, cancellable);
    uint64 loader_base = (uintptr) bootstrap_result.context.allocation_base;
    #远程 mmap 一块可写、可执行的内存，得到 loader_base
    try {
        #取出 Frida 预编译的 loader 机器码，对应的是 src/linux/helpers/loader.c 编译出来的小型加载器
        unowned uint8[] loader_code = Frida.Data.HelperBackend.get_loader_bin_blob ().data;
        #把 loader 写进目标进程内存
        write_memory (loader_base, loader_code);
        maybe_fixup_helper_code (loader_base, loader_code);// 做架构相关修正，例如重定位
        #写入 loader 的运行参数
        var loader_ctx = HelperLoaderContext ();
        loader_ctx.ctrlfds = bootstrap_result.context.ctrlfds;
        loader_ctx.agent_entrypoint = (string *) (loader_base + loader_layout.agent_entrypoint_offset);
        loader_ctx.agent_data = (string *) (loader_base + loader_layout.agent_data_offset);
        loader_ctx.fallback_address = (string *) (loader_base + loader_layout.fallback_address_offset);
        loader_ctx.libc = (HelperLibcApi *) (loader_base + loader_layout.libc_api_offset);
        #相关值写入
        write_memory (loader_base + loader_layout.ctx_offset, (uint8[]) &loader_ctx);
        write_memory (loader_base + loader_layout.libc_api_offset, (uint8[]) &bootstrap_result.libc);
        write_memory_string (loader_base + loader_layout.agent_entrypoint_offset, spec.entrypoint);
        write_memory_string (loader_base + loader_layout.agent_data_offset, spec.data);
        write_memory_string (loader_base + loader_layout.fallback_address_offset, fallback_address);
        #远程执行loader
        return yield launch_loader (FROM_SCRATCH, spec, bootstrap_result, null, fallback_address, loader_layout,
                                    cancellable);
    }
    //...
}
```

```
private async RemoteAgent launch_loader (LoaderLaunch launch, InjectSpec spec, BootstrapResult bres, UnixConnection? agent_ctrl, string fallback_address, LoaderLayout loader_layout, Cancellable? cancellable) throws Error, IOError {
    # 与 helper建立通信连接
    Future<RemoteAgent> future_agent =
        establish_connection (launch, spec, bres, agent_ctrl, fallback_address, cancellable);
    # 构造 loader 调用环境
    uint64 loader_base = (uintptr) bres.context.allocation_base;# loader入口地址
    GPRegs regs = saved_regs;
    regs.stack_pointer = bres.allocated_stack.stack_root#指向新栈
    var call_builder = new RemoteCallBuilder (loader_base, regs);#从loader_base开始执行
    call_builder.add_argument (loader_base + loader_layout.ctx_offset);# 添加参数，类型为HelperLoaderContext*
    RemoteCall loader_call = call_builder.build (this);
    RemoteCallResult loader_result = yield loader_call.execute (cancellable);
    // ...

    var establish_cancellable = new Cancellable ();
    var main_context = MainContext.get_thread_default ();

    var timeout_source = new TimeoutSource.seconds (5);
    timeout_source.set_callback (() => {
        establish_cancellable.cancel ();
        return Source.REMOVE;
    });
    timeout_source.attach (main_context);

    var cancel_source = new CancellableSource (cancellable);
    cancel_source.set_callback (() => {
        establish_cancellable.cancel ();
        return Source.REMOVE;
    });
    cancel_source.attach (main_context);

    RemoteAgent agent = null;
    try {
        agent = yield future_agent.wait_async (establish_cancellable);
    }
    // ...

    agent.ack ();//发送 ACK，允许 loader 进入 agent main

    return agent;
}
```

用之前 `ptrace` 停住时的上下文，把 `PC/IP` 指到 `loader_base`（`loader` 入口），把第一个参数设成 `HelperLoaderContext*`，栈换成刚分配的

之后的代码逻辑就是在`src\linux\helpers\loader.c`中，入口是`frida_load()`，里面通过线程启动`frida_main()`方法，在后者中，通过`dlopen()` 加载 `agent`，通过 `dlsym()` 找到 `frida_agent_main`，最后调用 `agent` 入口函数

大概看完上述注入原理，我们可以发现`frida`的几个特征还是比较明显的，我们以下细细说明

#### 检测 frida 特征文件

上述提到`frida-server` 通常被推送到 `/data/local/tmp`

`java`层有：

```
public boolean checkFridaFiles() {
    String[] paths = {
        "/data/local/tmp/frida-server",
        "/data/local/tmp/re.frida.server",
        "/data/local/tmp/frida-gadget.so"
    };

    // 方式一：直接检查已知路径
    for (String path : paths) {
        if (new File(path).exists()) {
            return true;
        }
    }

    // 方式二：遍历 /data/local/tmp，匹配关键字
    File tmpDir = new File("/data/local/tmp");
    File[] files = tmpDir.listFiles();
    if (files != null) {
        for (File f : files) {
            String name = f.getName().toLowerCase();
            if (name.contains("frida") || name.contains("gadget")
                || name.contains("re.frida")) {
                return true;
            }
        }
    }

    return false;
}
```

`Native`层有：

```
#include <dirent.h>
#include <string.h>

int check_frida_files() {
    const char *dirs[] = {
        "/data/local/tmp",
        "/data/local",
        NULL
    };

    for (int i = 0; dirs[i] != NULL; i++) {
        DIR *d = opendir(dirs[i]);
        if (d == NULL) continue;

        struct dirent *entry;
        while ((entry = readdir(d)) != NULL) {
            char *name = entry->d_name;
            if (strstr(name, "frida") || strstr(name, "gadget")
                || strstr(name, "re.frida")) {
                closedir(d);
                return 1;
            }
        }
        closedir(d);
    }
    return 0;
}
```

##### 绕过方式

对`frida-server`进行改名  
放到其他目录，比如 `/data/local/tmp/.hidden/`，不过这只能骗路径写死的弱检测  
用 `ZygiskFrida`，不依赖 `frida-server`

#### 端口检测以及协议认证

`frida-server` 默认监听 27042（和 27043）`App` 直接尝试连接 127.0.0.1:27042，能连上就说明有 `Frida`嫌疑

`java`层实现：

```
public boolean checkFridaPort() {
    int[] ports = {27042, 27043};
    for (int port : ports) {
        try {
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("127.0.0.1", port), 200);
            socket.close();
            return true; // 连上了
        } catch (Exception e) {
            // 连不上，继续试下一个
        }
    }
    return false;
}
```

`Native`层实现：

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

struct sockaddr_in {
    sa_family_t    sin_family;  // 地址族，IPv4 填 AF_INET
    in_port_t      sin_port;    // 端口，必须用网络字节序
    struct in_addr sin_addr;    // IP 地址
    char           sin_zero[8]; // 填充
};

struct timeval {
    time_t      tv_sec;   // 秒
    suseconds_t tv_usec;  // 微秒
};

int check_frida_port() {
    int ports[] = {27042, 27043};

    for (int i = 0; i < 2; i++) {
        int fd = socket(AF_INET, SOCK_STREAM, 0);// 建 TCP socket
        if (fd < 0) continue;

        struct sockaddr_in addr;// 目标地址结构体
        memset(&addr, 0, sizeof(addr));
        addr.sin_family = AF_INET;
        addr.sin_port = htons(ports[i]);
        addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        // 设置超时
        struct timeval tv;
        tv.tv_sec = 0;
        tv.tv_usec = 200000; // 200ms
        setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
        setsockopt(fd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv));

        int ret = connect(fd, (struct sockaddr *)&addr, sizeof(addr));
        close(fd);

        if (ret == 0) {
            return 1; // 连上了
        }
    }
    return 0;
}
```

当然更严谨的要发送`D-BUS`认证消息，`frida-server` 使用 `D-Bus` 协议进行通信，因此会对 `D-Bus` 的 `AUTH` 握手请求做出响应，更彻底的检测会遍历 0~65535 所有端口，对每个开放端口都发 `D-Bus AUTH` 探测

下面是用于说明思路的简化伪代码，不代表完整的协议实现：

```
Socket socket = new Socket("127.0.0.1", port);
InputStream is = socket.getInputStream();
OutputStream os = socket.getOutputStream();

// 发送 D-Bus 握手空字节
os.write(new byte[]{0});

// 读取响应
byte[] buffer = new byte[512];
int len = is.read(buffer);
String response = new String(buffer, 0, len);

// 检测 REJECT 关键字
if (response.contains("REJECT")) {
    // 发现 Frida！
}
```

##### 绕过方式

```
./frida-server -l 0.0.0.0:1337   # 改端口
frida -H 127.0.0.1:1337 -f com.target.app
或者用 frida-gadget、ZygiskFrida，完全不监听端口
```

或者更绝一点，直接给比较点`Hook`了，这是因为我们知道认证消息可以包括`REJECT`和`OK`

```
const strcmpAddr = Module.findExportByName("libc.so", "strcmp");
if (strcmpAddr !== null) {
    Interceptor.attach(strcmpAddr, {
        onEnter: function (args) {
            // 在 onEnter 中保存参数，供 onLeave 使用
            this.s1 = args[0].readCString();
            this.s2 = args[1].readCString();
        },
        onLeave: function (retval) {
            // 只检查特定条件：任意一个参数包含 "REJECT"
            const s1 = this.s1 || "";
            const s2 = this.s2 || "";

            if (s1.indexOf("REJECT") >= 0 || s2.indexOf("REJECT") >= 0) {
                // 满足条件时才改写返回值
                console.log(`[strcmp] forcing equal for: ${s1} vs ${s2}`);
                retval.replace(0); // 0 表示相等
            }
            // 不满足条件时，这里什么都不做，retval 保持原样
        }
    });
}
```

#### 内存扫描

`Frida` 注入后，其 `frida-agent` 和运行时引擎会在目标进程的内存中留下痕迹

上文也提到了关于系统是否支持`memfd`时的`Frida`注入的差别，`Frida`常制造匿名`rwx`权限段来放置`frida-agent.so`，目的是逃过磁盘上上的文件扫描

`Android`系统强制执行`W^X`，但`Frida`在安装`inline` `hook`时，需要修改目标函数指令并写入跳转代码（`trampoline`），这要求`Frida`将相关内存页权限修改为`rwx`，说明我们可以检查内存权限段，当然`rwx`段并不一定就是，许多合法库也有的

```
#include <stdio.h>
#include <string.h>

int detect_frida_in_maps() {
    FILE *fp = fopen("/proc/self/maps", "r");
    if (!fp) return 0;

    char line[512];
    while (fgets(line, sizeof(line), fp)) {
        // 检测1：查找 rwx 权限段
        if (strstr(line, "rwx")) {
            fclose(fp);
            return 1; // 发现可疑RWX内存
        }
        // 检测2：查找 memfd 特征（Frida 17+）
        if (strstr(line, "/memfd:")) {
            fclose(fp);
            return 1; // 发现memfd匿名文件
        }
        // 检测3：查找通用frida特征字符串
        if (strstr(line, "frida") || strstr(line, "gum-js") || strstr(line, "gadget")) {
            fclose(fp);
            return 1; // 发现Frida特征字符串
        }
    }
    fclose(fp);
    return 0; // 未检测到
}
```

在实际的`App`加固方案中，检测代码会更为复杂，还有命中后自毁`abort()`等手段

##### 绕过方式

可以`Hook` `fopen`、`open`、`read`等文件操作函数，在检测代码读取`maps`时，动态修改返回的内容，擦除`Frida`的特征

```
// Frida脚本示例：Hook fopen，过滤maps文件中的frida特征
var fopen = Module.findExportByName(null, "fopen");
if (fopen) {
    Interceptor.attach(fopen, {
        onEnter: function(args) {
            this.path = args[0].readCString();
        },
        onLeave: function(retval) {
            if (this.path && this.path.indexOf("maps") !== -1 && retval != 0) {
                // 对返回的FILE*进行后续处理比较复杂，通常需要Hook read/fgets
                console.log("[*] Maps file opened, will filter content on read.");
            }
        }
    });
}

// 更通用的做法：Hook read，过滤缓冲区内容
var read = Module.findExportByName(null, "read");
if (read) {
    Interceptor.attach(read, {
        onEnter: function(args) {
            this.fd = args[0].toInt32();
            this.buf = args[1];
            this.count = args[2].toInt32();
        },
        onLeave: function(retval) {
            if (retval.toInt32() > 0) {
                var content = this.buf.readUtf8String(retval.toInt32());
                if (content && (content.indexOf("frida") !== -1 || content.indexOf("memfd:") !== -1)) {
                    // 将敏感内容替换为空或无害字符串
                    var filtered = content.replace(/.*(frida|memfd:).*\n/g, "");
                    this.buf.writeUtf8String(filtered);
                    retval.replace(filtered.length);
                }
            }
        }
    });
}
```

当然，最简单的方法就是直接上魔改的得了，社区一堆

#### ptrace 检测

`Linux/Android` 内核规定：​同一个进程同时只能被一个调试器（`Tracer`）附加

`Frida` 在注入阶段通过 `ptrace` 附加并暂停目标线程，读写其寄存器和内存，让目标线程临时执行 `loader`，在其创建新线程后立即返回，`Frida` 恢复寄存器并 `detach`

`App` 在启动的极早期（通常在 `.init_array`​ 或 `JNI_OnLoad`​ 中），主动调用 `ptrace(PTRACE_TRACEME, 0, 0, 0)`，如果 `Frida` 已经附加，`App` 再调用相关 `ptrace` 操作可能失败，由此检测出问题

再有一种变体就是开`Watcher`进程先 `ptrace` 附加 `Worker`，占住 `ptrace` 字段

##### 绕过方式

`Hook` `ptrace`，不再调用原始实现，直接返回伪造结果

```
function bypassPtrace() {
    var ptracePtr = Module.findExportByName("libc.so", "ptrace");

    if (ptracePtr) {
        // 使用 Interceptor.replace 完全替换掉 ptrace 的实现
        Interceptor.replace(ptracePtr, new NativeCallback(function(request, pid, addr, data) {
            console.log("[+] ptrace called - bypassing...");
            console.log("    request: " + request);

            // PTRACE_TRACEME 的常量值通常是 0
            if (request == 0) {
                console.log("    [+] Blocked PTRACE_TRACEME!");
                // 直接返回 0，欺骗 App 说“你已经成功附加自己了”
                return 0;
            }

            // 如果有其他 ptrace 需求，可以在这里放行，或者全部屏蔽
            return 0;

        }, 'long', ['int', 'int', 'pointer', 'pointer']));

        console.log("[+] ptrace bypass applied!");
    } else {
        console.log("[-] ptrace not found");
    }
}

setImmediate(function() {
    bypassPtrace();
});
```

#### 检查 / proc/pid/fd

`Frida`通过 `memfd` 加载 `frida-agent.so` 时，会在目标进程里留下一个 `fd`，指向 `/memfd:frida-agent-64.so`

`Native`实现如下：

```
#include <dirent.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int check_frida_fd() {
    DIR *d = opendir("/proc/self/fd");
    if (!d) return 0;

    struct dirent *entry;
    while ((entry = readdir(d)) != NULL) {
        if (entry->d_name[0] == '.') continue;

        char path[256];
        snprintf(path, sizeof(path), "/proc/self/fd/%s", entry->d_name);

        char target[256] = {0};
        ssize_t len = readlink(path, target, sizeof(target) - 1);
        if (len <= 0) continue;

        if (strstr(target, "frida") || strstr(target, "memfd:")) {
            closedir(d);
            return 1;
        }
    }
    closedir(d);
    return 0;
}
```

##### 绕过方式

魔改 `Frida` 改 `memfd` 名称，或 `Hook` `readlink` 返回假路径

#### 检查 / proc/pid/tasks

`Frida` 运行时依赖 `GLib`，会创建名称固定的辅助线程，可以通过遍历 `/proc/self/task/` 目录，读取各线程的`comm`文件，匹配特征线程名

伪代码如下：

```
File taskDir = new File("/proc/self/task");
for (File file : taskDir.listFiles()) {
    // 读取 comm 文件获取线程名
    String threadName = readFile(file.getAbsolutePath() + "/comm").trim();
    if (threadName.equals("gmain") || threadName.equals("gum-js-loop")) {
        // 发现异常线程
        killApp();
    }
}
```

##### 绕过方式

`Hook` 文件打开与读取：核心思路是拦截对 `/proc`​ 目录下 `comm`​ 或 `status`​ 文件的读取操作。当发现 `App` 试图读取特定线程的信息时，检查读取到的内容，如果是 `gmain`​ 等敏感名称，将其替换为合法的系统线程名（如 `epoll`​、`config_store` 等）

```
function bypassThreadCheck() {
    var openat = Module.findExportByName("libc.so", "openat");
    var read = Module.findExportByName("libc.so", "read");

    // 保存打开的文件描述符与路径的映射
    var fdMap = new Map();

    if (openat) {
        Interceptor.attach(openat, {
            onEnter: function(args) {
                this.path = Memory.readUtf8String(args[1]);
                this.fd_ptr = null;
            },
            onLeave: function(retval) {
                // 如果打开的是 task 目录下的文件，记录 FD
                if (retval.toInt32() > 0 && this.path && this.path.indexOf("/task/") !== -1) {
                    fdMap.set(retval.toInt32(), this.path);
                }
            }
        });
    }

    if (read) {
        Interceptor.attach(read, {
            onEnter: function(args) {
                this.fd = args[0].toInt32();
                this.buf = args[1];
                this.count = args[2];
            },
            onLeave: function(retval) {
                if (fdMap.has(this.fd) && retval.toInt32() > 0) {
                    var content = Memory.readUtf8String(this.buf, retval.toInt32());

                    // 检查是否存在敏感线程名
                    if (content.indexOf("gmain") !== -1 ||
                        content.indexOf("gdbus") !== -1 ||
                        content.indexOf("gum-js-loop") !== -1) {

                        // console.log("[*] Hiding thread name: " + content.trim());

                        // 替换为普通线程名，注意长度不要超过原长度太多
                        var fakeName = "jit_thread";
                        Memory.writeUtf8String(this.buf, fakeName);
                        retval.replace(fakeName.length);
                    }
                }
            }
        });
    }
}

setImmediate(bypassThreadCheck);
```

或者干脆使用去特征的魔改

对于 `fd`​ (文件描述符)、`status`​ (线程状态)、`maps`​ (内存映射) 这三类基于 `/proc`​ 文件系统的检测，上述 `IO` 重定向 方案是通解，只要将敏感文件指向无害文件（如 `/dev/null` 或提前备份的正常文件），即可实现完美绕过

#### 为什么基础对抗会失效？

```
Interceptor.attach(Module.findExportByName("libc.so", "open"), ...);
Interceptor.attach(Module.findExportByName("libc.so", "read"), ...);
Interceptor.attach(Module.findExportByName("libc.so", "strcmp"), ...);
Interceptor.attach(Module.findExportByName("libc.so", "connect"), ...);
```

以上我们大多通过`hook libc`导出函数来绕过检查，那么假如不走`libc`，直接 `svc`进内核，自己实现 字符串比较、读文件，不调你 `Hook` 的符号，那我们就难以实现反调试

#### 处理方式

##### 直接使用 syscall 替代 libc 函数

```
#include <sys/syscall.h>  //SYS_openat SYS_read SYS_close都是syscall.h中的常量
                          //代表系统调用的编号。在Linux系统中，每个系统调用都有一个唯一的编号
#include <unistd.h>
#include <fcntl.h>

int my_open(const char *pathname, int flags) {
    return syscall(SYS_openat, AT_FDCWD, pathname, flags, 0);
}

ssize_t my_read(int fd, void *buf, size_t count) {
    return syscall(SYS_read, fd, buf, count);
}

int my_close(int fd) {
    return syscall(SYS_close, fd);
}

// 使用示例
int main() {
    int fd = my_open("/path/to/file", O_RDONLY);  //fd代表文件标识符，代表打开的是哪个文件
    if (fd != -1) {
        char buffer[100];
        ssize_t bytes_read = my_read(fd, buffer, sizeof(buffer));
        my_close(fd);
    }
    return 0;
}
```

##### 动态生成 syscall

```
#include <sys/mman.h>
#include <string.h>

typedef long (*syscall_fn)(long, ...);

syscall_fn generate_write_syscall() {
    // 分配可执行内存
    void* mem = mmap(NULL, 4096, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    // x86-64 架构的 write syscall 机器码
    unsigned char code[] = {
        0x48, 0xc7, 0xc0, 0x01, 0x00, 0x00, 0x00,  // mov rax, 1 (write syscall number)
        0x0f, 0x05,                                // syscall
        0xc3                                       // ret
    };
    // 复制代码到可执行内存
    memcpy(mem, code, sizeof(code));

    return (syscall_fn)mem;
}
// 使用示例
int main() {
    syscall_fn my_write = generate_write_syscall();
    const char *msg = "Hello, World!\n";
    my_write(1, msg, strlen(msg));
    return 0;
}
```

`AArch64` 下

```
#include <sys/mman.h>
#include <string.h>
#include <unistd.h>

typedef long (*syscall_fn)(long, ...);

syscall_fn generate_write_syscall() {
    void* mem = mmap(NULL, 4096, PROT_READ | PROT_WRITE | PROT_EXEC,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

    unsigned char code[] = {
        0x08, 0x08, 0x80, 0xd2,  // mov x8, #64      (write)
        0x01, 0x00, 0x00, 0xd4,  // svc #0
        0xc0, 0x03, 0x5f, 0xd6   // ret
    };

    memcpy(mem, code, sizeof(code));
    return (syscall_fn)mem;
}
```

以下给出实际例子：

```
#include <jni.h>
#include <string.h>
#include <fcntl.h>
#include <sys/syscall.h>
#include <unistd.h>

JNIEXPORT jboolean JNICALL
Java_com_example_SecurityCheck_detectFrida(JNIEnv *env, jobject thiz) {
    char line[256];
    int fd = syscall(SYS_openat, AT_FDCWD, "/proc/self/maps", O_RDONLY, 0);
    if (fd != -1) {
        ssize_t bytes_read;
        while ((bytes_read = syscall(SYS_read, fd, line, sizeof(line) - 1)) > 0) {
            line[bytes_read] = '\0';
            if (strstr(line, "frida") || strstr(line, "gum-js-loop")) {
                syscall(SYS_close, fd);
                return JNI_TRUE;
            }
        }
        syscall(SYS_close, fd);
    }
    return JNI_FALSE;
}
```

##### 绕过方式

我们发现，自实现系统函数一个重要的前提就是它们都有标准的系统调用号，标准的机器码，直接搜索对应函数的特征码，定位到之后再使用`Interceptor`进行`Hook`，但是需要处理架构差异

```
function hookSysOpen() {
    let SYS_OPENAT;
    let SVC_INSTRUCTION_HEX;
    const arch = Process.arch;

    if (arch === "arm64") {
        SYS_OPENAT = 56;  // ARM64架构下openat系统调用的编号
        SVC_INSTRUCTION_HEX = "01 00 00 D4";  // ARM64架构下svc指令的十六进制表示
    } else if (arch === "arm") {
        SYS_OPENAT = 5;  // ARM架构下open系统调用的编号
        SVC_INSTRUCTION_HEX = "00 00 00 EF";  // ARM架构下svc指令的十六进制表示
    } else {
        console.log("不支持的架构: " + arch);
        return;
    }

    console.log("当前架构: " + arch);
    console.log("开始搜索SYS_OPENAT系统调用...");

    //系统调用指令（如svc）通常位于可执行代码段中（r-x）
    Process.enumerateRanges('r-x').forEach(function(range) {
        if (range.file && range.file.path && range.file.path.endsWith(".so")) {
            console.log("搜索模块: " + range.file.path);

            Memory.scan(range.base, range.size, SVC_INSTRUCTION_HEX, {
                onMatch: function(address) {
                    let sysCallNumber;
                    if (arch === "arm64") {
                        // 在ARM64中，系统调用号在svc指令之前的指令中
                        sysCallNumber = address.sub(4).readU32() & 0xFFFF;
                    } else if (arch === "arm") {
                        // 在ARM中，系统调用号通常在r7寄存器中，这里我们只能近似处理
                        sysCallNumber = address.sub(4).readU16() & 0xFF;
                    }

                    if (sysCallNumber === SYS_OPENAT) {
                        console.log("找到SYS_OPENAT调用，地址: " + address);

                        Interceptor.attach(address, {
                            onEnter: function(args) {
                                let fileName;
                                if (arch === "arm64") {
                                    fileName = args[1].readUtf8String();
                                } else if (arch === "arm") {
                                    fileName = args[0].readUtf8String();
                                }
                                console.log("SYS_OPENAT被调用，文件名: " + fileName);
                            },
                            onLeave: function(retval) {
                                console.log("SYS_OPENAT返回值: " + retval);
                            }
                        });
                    }
                },
                onComplete: function() {
                    console.log("搜索完成");
                }
            });
        }
    });
}

hookSysOpen();
```

##### 检测 Inline Hook

`Inline Hook` 是 `Frida Interceptor` 模块的核心机制。原理是直接修改内存中目标函数的前几条指令（`Prologue`），替换为一条跳转指令（`Trampoline`/ 跳板），把执行流引导到 `Frida` 的处理函数，还有软件断点，调试器插入断点会写入 `0xCC (INT 3)` 指令，同样会改变 `CRC`

```
; Hook 前（原始指令）
sub sp, sp, #0x30          ; ff c3 04 d1
stp x29, x30, [sp, #0x20]  ; fd 7b 0e a9
str x28, [sp, #0x10]       ; fc 7b 00 f9
str x24, [sp, #0x20]       ; f8 5f 10 a9

; Hook 后（Frida 特征）
ldr x16, #8                ; 50 00 00 58
br  x16                    ; 00 02 1f d6
.long 0x747d3e6e           ; 跳转地址低位
.long 0x00000000           ; 跳转地址高位
```

一般的检测是 `CRC/MD5` 比对：计算内存中函数前 `N` 字节的哈希，和磁盘 `.so` 里原始机器码比对，不一致说明被 `Hook`

###### 绕过方式

方法一：`Hook` `memcmp`

```
var memcmpAddr = Module.findExportByName("libc.so", "memcmp");
Interceptor.attach(memcmpAddr, {
    onEnter: function (args) {
        this.s1 = args[0];
        this.s2 = args[1];
        this.n = args[2].toInt32();
    },
    onLeave: function (retval) {
        if (this.n === 16) {  // 常见比对长度
            retval.replace(0);  // 强行相等
        }
    }
});
```

但是`Hook strcmp`带来的开销太大，我们常`Hook`哈希函数作为替代

`Root`的权限判断并没有想象中仅仅修改一个`uid`这么简单，权限检查是多重配合的，我目前已知是：`UID / GID` `capabilities` `SELinux` `seccomp` `挂载命名空间`

#### CRED 结构体与能力集 (DAC)

每个进程都有一个`task_struct`，其内部有一个`cred`结构体记载了权限信息

```
struct task_struct {
    pid_t pid;                  // 进程 ID
    pid_t tgid;                 // 线程组 ID
    struct task_struct *parent; // 父进程
    struct cred *cred;          // 权限信息
    struct ptrace_relation *ptrace; // ptrace 关系
    struct mm_struct *mm;       // 内存描述符
    ...
};
```

```
struct cred {
    atomic_t usage;                  // 引用计数
    kuid_t uid;                      // 真实用户 ID
    kgid_t gid;                      // 真实组 ID
    kuid_t suid;                     // 保存的用户 ID
    kgid_t sgid;                     // 保存的组 ID
    kuid_t euid;                     // 有效用户 ID
    kgid_t egid;                     // 有效组 ID
    kuid_t fsuid;                    // 文件系统用户 ID
    kgid_t fsgid;                    // 文件系统组 ID
    unsigned securebits;
    kernel_cap_t cap_inheritable;
    kernel_cap_t cap_permitted;
    kernel_cap_t cap_effective;      // 有效能力集
    kernel_cap_t cap_bset;           // 能力边界集
    kernel_cap_t cap_ambient;
    // ... 其他字段如 user, user_ns, group_info, security 等
} __randomize_layout;
```

每个 `App` 安装时，`PackageManager` 分配一个独立 `UID`，当访问文件的时候，第一层是检查`DAC`(`Discretionary Access Control`)，根据 `UID/GID` 和文件权限位（`rwx`）决定能不能访问，文件属主可以自己 `chmod` 改权限

举例：

一个 `App`，`UID` 是 `10123`，去读另一个 `App` 的文件

```
文件：/data/data/com.other.app/files/secret
属主：UID 10124
属组：GID 10124
权限：-rw-------  (600)

1. 当前进程 fsuid = 10123
   文件属主 uid = 10124

2. 当前进程 fsgid = 10123
   文件属组 gid = 10124

3. other 是 ---
   没有任何权限

4. 拒绝，返回 EACCES
```

如果是`Root`权限，会直接无视这一步的检查，这是因为`DAC`检查了一个`CAP_DAC_OVERRIDE`，有这个的默认放行，而`Root`的能力集`cap_effective` 里有这一项

```
// fs/namei.c, generic_permission()
int generic_permission(...)
{
    int ret;

    // 第一步：DAC 检查
    ret = acl_permission_check(idmap, inode, mask);
    if (ret != -EACCES)
        return ret;   // DAC 通过，直接返回

    // 第二步：DAC 拒绝了，才查 CAP_DAC_OVERRIDE
    if (capable(CAP_DAC_OVERRIDE))
        return 0;

    return -EACCES;
}
```

#### Security-Enhanced Linux(MAC)

`SELinux` 是强制访问控制 (`MAC，Mandatory Access Control`)

过了`DAC`权限检查后，并不能直接地获得所有文件的读写权限，否则太易受攻击，而`Security-Enhanced Linux`采取的是最小权限原则

`SELinux`的核心是它的安全上下文，格式为：`user:role:type:level`

他是从域和策略的角度来决定的是否进程能访问某个文件

举例如下：

```
进程：init
  uid = 0
  euid = 0
  cap_effective = 全开（CAP_DAC_OVERRIDE、CAP_SYS_ADMIN 等都有）
  domain = kernel

目标：/data/data/com.example.app/files/secret
  i_uid = 10089
  i_mode = 600
  type = app_data_file

  subject domain = kernel
  object type    = app_data_file

  但是策略表不存在allow kernel app_data_file:file read就被拒绝了
```

#### seccomp(secure computing mode)

`seccomp (Secure Computing Mode)` 是 `Linux` 内核的一种安全机制，用于限制进程可以调用的系统调用集，本质上是`linux`系统调用的防火墙，可查看文档 [http://man7.org/linux/man-pages/man3/seccomp_rule_add.3.html](http://man7.org/linux/man-pages/man3/seccomp_rule_add.3.html)

`seccomp` 的核心工作原理是基于 `cBPF`（`classic` `Berkeley Packet Filter`），一种内核中的 "虚拟机" 机制，来定义和执行系统调用的过滤规则

`BPF`就理解成内核里的`if-else`规则表就好

假设内核要决定：某个系统调用 (`syscall`) 不让执行，没有 `BPF` 的话，内核只能写死

可以把 `Seccomp` 想象成一个海关检查站：

应用程序 (`Application`): 发起一个请求

`Seccomp` 过滤器 (`The Checkpoint`): 拦截这个请求，并检查手中的 “规则手册”（`BPF Profiles`）

规则：允许删除文件吗？`No`

内核 (`Kernel`): 根据 `Seccomp` 的指令，直接拒绝请求，并可能惩罚该进程（终止运行）

结果: 恶意操作未遂，系统安全得到保障

#### 挂载命名空间

前四层管的是 “能不能做这个操作”：

`UID/GID`：你的身份是什么

`capabilities`：你有没有这个特权

`SELinux`：你的域允许不允许

`seccomp`：这个 `syscall` 能不能发出去

挂载命名空间管的是你这个进程能看到什么

#### Su

老版本 `Android` 的 `/system` 分区还没有挂载 `nosuid`，那么这就给了普通 `App`执行`/system/bin/su`的机会，大概原理是`su`文件含有`setuid`位，以 “文件属主” 的身份运行，而`su`的身份显然是`Root`，现在我们就有了一个`Root`进程，`su`已经以较高权限跑起来了，但是它自己还要完成策略和切换，修改 `cred uid=0, capabilities...`等等参数，然后通常再 `exec` 成一个 `shell`  
，但是问题在于，`Andriod`根本不想给你提权的机会，它的`/system/bin/`下根本不会给你这些`su`文件，这就带来了检测方案

##### 检测

我们直接对着扫目录就行

`Java`层：

```
public static boolean isDeviceRooted() {
        String[] locations = {"/system/bin/", "/system/xbin/", "/sbin/", "/system/sd/xbin/",
                "/system/bin/failsafe/", "/data/local/xbin/", "/data/local/bin/", "/data/local/",
                "/system/sbin/", "/usr/bin/", "/vendor/bin/"};
        for (String location : locations) {
            if (new File(location + "su").exists()) {
                return true;
            }
        }
        return false;
    }
```

`Native`层：

```
#include <stdio.h>
#include <unistd.h>
#include <stdbool.h>

bool is_device_rooted(void) {
    const char *locations[] = {
        "/system/bin/",
        "/system/xbin/",
        "/sbin/",
        "/system/sd/xbin/",
        "/system/bin/failsafe/",
        "/data/local/xbin/",
        "/data/local/bin/",
        "/data/local/",
        "/system/sbin/",
        "/usr/bin/",
        "/vendor/bin/",
        NULL
    };

    char path[256];
    for (int i = 0; locations[i] != NULL; i++) {
        snprintf(path, sizeof(path), "%ssu", locations[i]);
        // F_OK：只判断文件是否存在
        if (access(path, F_OK) == 0) {
            return true;
        }
    }
    return false;
}
```

但是很显然地，这个直接`mount namespace`就解决了，所以我们可以尝试执行提权命令，通过回返判断

```
public static boolean checkSuCommand() {
    Process process = null;
    try {
        process = Runtime.getRuntime().exec(new String[]{"su", "-c", "id"});
        BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()));
        String line = reader.readLine();
        int code = process.waitFor();

        if (line != null && line.contains("uid=0")) {
            return true; // 真执行了
        }
        // 有的环境输出在 stderr，也可以再读 getErrorStream()
    } catch (Exception e) {
        // 执行失败：没 su、被拒绝、被隐藏等
        return false;
    } finally {
        if (process != null) process.destroy();
    }
    return false;
}
```

`Native`层：

```
#include <stdio.h>
#include <string.h>
#include <stdbool.h>

bool check_su_command(void) {
    FILE *fp = popen("su -c id", "r");
    if (!fp) {
        return false; // 启动失败
    }

    char buf[256];
    bool rooted = false;

    while (fgets(buf, sizeof(buf), fp) != NULL) {
        if (strstr(buf, "uid=0") != NULL) {
            rooted = true;
            break;
        }
    }

    pclose(fp);
    return rooted;
}
```

尽管如此，但仍可能被策略拒绝、隐藏、`Hook` 执行结果

#### Magisk

[https://bbs.kanxue.com/thread-292966.htm](https://bbs.kanxue.com/thread-292966.htm) 内有一张不错的流程图

![](https://bbs.kanxue.com/upload/attach/202610/1073886_SQQR5WUDNH42RTA.webp)

##### Magisk 的启动劫持

###### Magiskinit

了解过 `Android` 启动过程的应该知道，`Bootloader` 负责加载并启动内核，内核随后启动 `PID` 1 的 `init`，再由 `init` 按启动配置拉起其他系统服务和进程（可以看这位师傅的 [https://bbs.kanxue.com/thread-285949.htm](https://bbs.kanxue.com/thread-285949.htm)），`Magisk` 正是通过修改 `boot` 镜像，在早期启动阶段插入自己的逻辑（核心是 `magiskinit`），从更靠前的位置完成 `Root` 环境的部署

下面我们对着源代码分析一下关键节点：

```
#https://github.com/topjohnwu/Magisk/blob/master/scripts/boot_patch.sh
./magiskboot unpack "$BOOTIMAGE"

unset RAMDISK #清空 RAMDISK 变量，防止之前残留的值干扰
for path in ramdisk.cpio vendor_ramdisk/init_boot.cpio vendor_ramdisk/ramdisk.cpio; do
  if [ -e $path ]; then
    RAMDISK=$path
    break
  fi
done
```

用 `magiskboot` 工具解包 `boot` 镜像，并找出真正的 `ramdisk`

```
./magiskboot cpio $RAMDISK \
"add 0750 init magiskinit" \ #替换 init
"mkdir 0750 overlay.d" \
"mkdir 0750 overlay.d/sbin" \
"add 0644 overlay.d/sbin/magisk.xz magisk.xz" \ #塞 magisk.xz，这是 Magisk 的核心二进制，它在启动早期被解压，提供 su、magiskd 等能力
"add 0644 overlay.d/sbin/stub.xz stub.xz" \
"add 0644 overlay.d/sbin/init-ld.xz init-ld.xz" \
"patch" \
"$SKIP_BACKUP backup ramdisk.cpio.orig" \
"mkdir 000 .backup" \
"add 000 .backup/.magisk config" \
|| abort "! Unable to patch ramdisk"
```

把 `ramdisk` 里的 `init` 换成 `magiskinit`，备份原始 `ramdisk`，记录配置，重新打包

那么在`magiskinit`里到底做了什么呢，我们追踪到 [https://github.com/topjohnwu/Magisk/blob/master/native/src/init/init.rs](https://github.com/topjohnwu/Magisk/blob/master/native/src/init/init.rs)

```
#[unsafe(no_mangle)]
pub unsafe extern "C" fn main(
    argc: i32,
    argv: *mut *mut c_char,
    _envp: *const *const c_char,
) -> i32 {
    unsafe {
        umask(0);
        // basename("/init") -> "init"
        // basename("/system/bin/magisk") -> "magisk"
        let name = basename(*argv);
        // 如果当前程序名是 "magisk"，就走 Magisk CLI/applet 的代理入口
        // 这里不是我们现在分析的 Android init 启动链，所以先跳过
        if CStr::from_ptr(name) == c"magisk" {
            return magisk_proxy_main(argc, argv);
        }
        // init PID 1
        if getpid() == 1 {
            MagiskInit::new(argv).start().log_ok();
        }

    }
}
```

接下来看`start`

```
// 因为启动早期和启动后期的环境不一样，Android 的 init，在启动过程中会执行自己两次
// 第一阶段：
//   - 根目录是 ramdisk
//   - /system 还没挂载
//   - 只能做最基础的事

// SwitchRoot 之后：
//   - 根目录切换到 system 分区
//   - 环境完全不同了
//   - 需要重新初始化

// 所以 init 干脆把自己重新执行一遍，用新的环境重新开始
// 而第二次运行的时候，argv[1] 是 "selinux_setup"
let argv1 = unsafe { *self.argv.offset(1) };

if !argv1.is_null()
    && unsafe { CStr::from_ptr(argv1) == c"selinux_setup" }
{
    // 如果是 selinux_setup：
    // 说明现在已经进入 Android init 的 Second Stage
    self.second_stage();

} else if unsafe {
    CStr::from_ptr(self.config.boot_mode.as_ptr())
} == c"charger" {
    // 如果 boot_mode == "charger"，
    // 说明设备当前是在充电模式启动
    // 这种情况下 Magisk 不做正常的 root/init 接管
    self.recovery_or_charger();

} else if self.config.skip_initramfs {
// 识别旧式 System-as-Root（Legacy SAR）设备
    self.legacy_system_as_root();
} else if self.config.force_normal_boot {
    // 强制按照正常 Android boot 流程处理
    // 这里进入的就是 Magisk 对 First Stage Init 进行处理的地方
    self.first_stage();

} else if cstr!("/sbin/recovery").exists()
    || cstr!("/system/bin/recovery").exists()
{
    // 如果文件系统中发现 recovery，
    // 说明当前很可能处于 recovery 环境
    // 所以恢复原始 init
    self.recovery_or_charger();

} else if self.check_two_stage() {
    // 前面的特殊情况都排除
    self.first_stage();

} else {
    // 如果不是 Two-Stage Init，
    // 那就是传统的 rootfs 启动路径
    //
    // 走 Magisk 针对传统 rootfs 的处理逻辑
    self.rootfs();
}

// 无论前面走了哪条分支，
// Magisk 自己需要做的初始化工作完成以后，
// 最终都会来到这里
//
// 这因为 magiskinit 并不代替 Android init
// 它只是先占住 /init 入口，在最早期启动阶段完成自己的工作，
// 然后继续执行真正的 Android init
self.exec_init();
```

分析`first_stage`:

```
fn first_stage(&self) {
    info!("First Stage Init");
    self.prepare_data();
    // 判断 /sdcard 是否存在
    // /sdcard 是 Android 的存储挂载点，SwitchRoot 之后系统会创建它
    //   /sdcard 不存在  ->  SwitchRoot 还没做，可以用 SwitchRoot 方式劫持
    //   /sdcard 存在    ->  SwitchRoot 已经做了，SwitchRoot 方式没用了
    if !cstr!("/sdcard").exists() && !cstr!("/first_stage_ramdisk/sdcard").exists() {

        // 分支一：SwitchRoot
        // 布置陷阱，让原始 init 在 SwitchRoot 时自己把 magiskinit 搬到 /system/bin/init
        // 具体做三件事：
        //   1. 创建符号链接 /storage/self/primary -> /system/system/bin/init
        //   2. 把 /init（magiskinit）改名为 /sdcard
        //   3. 绑定挂载 /sdcard，让它成为挂载点
        // 效果：SwitchRoot 会递归移动 / 下所有挂载点到 /system
        //       /sdcard 被移动到 /system/sdcard，而 /system/sdcard 是符号链接
        //       最终指向 /system/bin/init，init 把 magiskinit 搬过去了
        self.hijack_init_with_switch_root();

        // 恢复 /init 为原始 init
        self.restore_ramdisk_init();
    } else {
        // 分支二：hexpatch 方式（/sdcard 已经存在，SwitchRoot 时机已过）
        // 先恢复 /init 为原始 init
        self.restore_ramdisk_init();
        // 直接改 /init 二进制里的字符串
        // 把 /init 里的 "/system/bin/init" 字符串替换成 "/data/magiskinit"
        // 效果：第二阶段 init 尝试 execve("/system/bin/init") 时，
        //       实际执行的是 /data/magiskinit（第一步 prepare_data() 拷过去的那份 magiskinit）
        // Fallback to hexpatch if /sdcard exists
        hexpatch_init_for_second_stage(true);
    }
}
```

其实就是

```
【你（magiskinit）在 first_stage() 里做的事】
1. 备份自己到 /data/magiskinit
2. 把自己改名为 /sdcard
3. 建符号链接，让 /sdcard 最终指向 /system/bin/init
4. 把 /sdcard 变成挂载点
5. 把 /init 恢复成真正的 init
6. execve("/init") 退场

【真正的 init 启动后做的事】
1. 它执行 SwitchRoot
2. 它把 /sdcard（里面是你）搬到 /system/sdcard
3. 但符号链接让它实际上搬到了 /system/bin/init
4. 它执行 /system/bin/init，实际执行的是你（magiskinit）
5. 你第二次被启动，进入 second_stage()，做 SELinux 注入
```

接下来分析`second_stage`：

```
fn second_stage(&mut self) {
    info!("Second Stage Init");
    // 清理第一阶段留下的挂载
    // 第一阶段通过 SwitchRoot 陷阱或 hexpatch 把自己的副本挂到了 /init 或 /system/bin/init 上
    cstr!("/init").unmount().ok();
    cstr!("/system/bin/init").unmount().ok(); // just in case
    cstr!("/data/init").remove().ok();

    // 后续 init 会打印 dmesg 日志，会显示进程名
    // 如果显示 "magiskinit"，一眼就被看出被劫持了
    unsafe {
        // Make sure init dmesg logs won't get messed up
        *self.argv = raw_cstr!("/system/bin/init") as *mut _;
    }

    // 根据根文件系统类型，选择 SELinux 策略注入方式
    if is_rootfs() {
        // 还在 rootfs 上：根目录可写，走 patch_rw_root()
        // 删掉 /init，创建符号链接指向 /system/bin/init
        // 为什么需要：有些设备 SwitchRoot 后根目录还是 rootfs
        // 需要手动把 /init 指到第二阶段 init 的位置
        let init_path = cstr!("/init");
        init_path.remove().ok();
        init_path
            .create_symlink_to(cstr!("/system/bin/init"))
            .log_ok();

        // 注入 SELinux 策略 rootfs 可写
        self.patch_rw_root();
    } else {
        // 不在 rootfs 上：根目录只读，走 patch_ro_root()
        // 这是标准 2SI 设备的情况，/ 已经是 system 分区
        self.patch_ro_root();
    }
    //   patch_rw_root()：
    //     根目录可写，可以直接在根目录下创建/修改文件
    //     但实际上用了 overlay 机制（把 $INTERNALDIR/rootdir 绑定挂载到 /）
    //   patch_ro_root()：
    //     根目录只读，不能直接创建/修改文件
    //     必须用 overlay 机制（绑定挂载）来“覆盖”根目录下的文件
}
```

两个`patch_root`，由于篇幅过长，简单总结

```
1. 恢复 /sbin 结构（Magisk 之前占用了它）
2. 处理 overlay.d 里的自定义 rc 脚本
3. 打补丁 init.rc：删 vaultkeeper、改 flash_recovery、注入 magiskd 服务
4. 处理 init.zygote*.rc：注入 zygote restart 钩子
5. 解压 magisk.xz / stub.xz / init-ld.xz
6. 注入 SELinux 策略（handle_sepolicy）
7. 挂载 overlay 到根目录
```

这一阶段主要是处理第一阶段的痕迹，再者就是修改`selinux`策略，原始 `SELinux` 策略里没有关于`magisk`的域，所以 `Magisk` 必须修改 `SELinux` 策略，修改策略原理如下

```
// 核心手法：用 FIFO 阻塞 init 的控制流
//   init 启动时要加载 SELinux 策略，会读写 selinuxfs 的几个节点
//   Magisk 把这些节点用 FIFO 替换掉
//   init 一读写就阻塞，Magisk 把策略改了，再放行

use crate::consts::{PREINITMIRR, SELINUXMOCK};
use crate::ffi::{MagiskInit, preload_ack, preload_lib, preload_policy, split_plat_cil};
use base::const_format::concatcp;
use base::nix::fcntl::OFlag;
use base::{
    BytesExt, LibcReturn, LoggedResult, MappedFile, ResultExt, Utf8CStr, cstr, debug, error, info,
    libc, raw_cstr,
};
use magiskpolicy::ffi::SePolicy;
use std::io::{Read, Write};
use std::ptr;
use std::thread::sleep;
use std::time::Duration;

// 伪节点路径（Magisk 自己创建的，用来替换真节点）
// SELINUXMOCK = $MAGISKTMP/.magisk/selinux
const MOCK_VERSION: &Utf8CStr = cstr!(concatcp!(SELINUXMOCK, "/version"));
const MOCK_LOAD: &Utf8CStr = cstr!(concatcp!(SELINUXMOCK, "/load"));
const MOCK_ENFORCE: &Utf8CStr = cstr!(concatcp!(SELINUXMOCK, "/enforce"));
const MOCK_REQPROT: &Utf8CStr = cstr!(concatcp!(SELINUXMOCK, "/checkreqprot"));

// 真正的 selinuxfs 节点
const SELINUX_MNT: &str = "/sys/fs/selinux";
const SELINUX_ENFORCE: &Utf8CStr = cstr!(concatcp!(SELINUX_MNT, "/enforce"));
const SELINUX_LOAD: &Utf8CStr = cstr!(concatcp!(SELINUX_MNT, "/load"));
const SELINUX_REQPROT: &Utf8CStr = cstr!(concatcp!(SELINUX_MNT, "/checkreqprot"));

// 三种注入策略，按 Android 版本选
enum SePatchStrategy {
    // Android 10+，2SI 设备
    // 第二阶段 init 是动态可执行文件，可以 LD_PRELOAD
    // 直接替换 security_load_policy 函数，最干净
    LdPreload,

    // Android 8.0+，Treble 设备
    // selinuxfs 在 init 里挂载，失败会忽略
    // Magisk 可以自己挂 selinuxfs，劫持里面的节点
    SelinuxFs,

    // Android 6.0 - 7.1
    // selinuxfs 挂载失败是致命的，要等 init 自己挂好
    // 用 FIFO 阻塞 init 的控制流，趁机劫持节点
    Legacy,
}

// 用 FIFO 劫持节点
// 创建命名管道，绑定挂载覆盖目标节点
// init 读写时会阻塞，直到 Magisk 处理
fn mock_fifo(target: &Utf8CStr, mock: &Utf8CStr) -> LoggedResult<()> {
    debug!("Hijack [{}]", target);
    mock.mkfifo(0o666)?;                     // 创建 FIFO
    mock.bind_mount_to(target, false).log()  // 绑定挂载到目标节点
}

// 用普通文件劫持节点
// 不阻塞，直接返回
fn mock_file(target: &Utf8CStr, mock: &Utf8CStr) -> LoggedResult<()> {
    debug!("Hijack [{}]", target);
    drop(mock.create(OFlag::O_RDONLY, 0o666)?);  // 创建空文件
    mock.bind_mount_to(target, false).log()       // 绑定挂载
}

impl MagiskInit {
    // 对外入口
    pub(crate) fn handle_sepolicy(&mut self) {
        self.handle_sepolicy_impl().ok();
    }

    // 清理劫持，加载修补后的策略
    // SelinuxFs 和 Legacy 策略用
    fn cleanup_and_load(&self, rules: &str) {
        // 卸掉所有劫持，恢复真实节点
        cstr!("/init").unmount().ok();
        SELINUX_LOAD.unmount().log_ok();
        SELINUX_ENFORCE.unmount().ok();
        SELINUX_REQPROT.unmount().ok();

        // 读原始策略，注入 Magisk 规则和用户规则，写回内核
        let mut sepol = SePolicy::from_file(MOCK_LOAD);
        sepol.magisk_rules();          // 注入 Magisk 规则（建域、开权限）
        sepol.load_rules(rules);       // 加载用户自定义规则
        sepol.to_file(SELINUX_LOAD);   // 写 /sys/fs/selinux/load，加载到内核

        // 手动设 /init 的 SELinux 上下文
        // restorecon 在某些设备上不工作
        cstr!("/init")
            .follow_link()
            .set_secontext(cstr!("u:object_r:init_exec:s0"))
            .ok();

        // 恢复 overlay 文件的上下文
        self.restore_overlay_contexts();
    }

    fn handle_sepolicy_impl(&mut self) -> LoggedResult<()> {
        // 创建临时目录 $MAGISKTMP/.magisk/selinux
        cstr!(SELINUXMOCK).mkdir(0o711)?;

        // 读用户自定义规则（如果有）
        let mut rules = String::new();
        let mut policy_ver = cstr!("/selinux_version");
        let rule_file = cstr!(concatcp!("/data/", PREINITMIRR, "/sepolicy.rule"));
        if rule_file.exists() {
            debug!("Loading custom sepolicy patch: [{}]", rule_file);
            rule_file
                .open(OFlag::O_RDONLY)?
                .read_to_string(&mut rules)?;
        }
        // 判断设备类型，选策略
        let strat: SePatchStrategy;

        if cstr!("/system/bin/init").exists() {
            // /system/bin/init 存在 -> 2SI 设备
            strat = SePatchStrategy::LdPreload;
        } else {
            // 打开 /init 二进制，看它包含哪些字符串来判断
            let init = MappedFile::open(cstr!("/init"))?;
            if init.contains(split_plat_cil().as_str().as_bytes()) {
                // 有 split policy -> Android 8.0+ Treble
                strat = SePatchStrategy::SelinuxFs;
            } else if init.contains(policy_ver.as_bytes()) {
                // 有 /selinux_version -> 旧式，劫持 /selinux_version
                strat = SePatchStrategy::Legacy;
            } else if init.contains(cstr!("/sepolicy_version").as_bytes()) {
                // 有三星定制的 /sepolicy_version
                policy_ver = cstr!("/sepolicy_version");
                strat = SePatchStrategy::Legacy;
            } else {
                error!("Unknown sepolicy setup, abort...");
                return Ok(());
            }
        }

        match strat {
            SePatchStrategy::LdPreload => {
                info!("SePatchStrategy: LD_PRELOAD");

                // 把 init-ld.so 拷到 preload_lib() 位置
                // init-ld 实现了 security_load_policy 的替换版
                cstr!("init-ld").copy_to(preload_lib())?;

                // 设 LD_PRELOAD，init 启动时会加载 init-ld
                unsafe {
                    libc::setenv(raw_cstr!("LD_PRELOAD"), preload_lib().as_ptr(), 1);
                }

                // 创建 ack FIFO，用于和 init-ld 通信
                preload_ack().mkfifo(0o666)?;
            }
            SePatchStrategy::SelinuxFs => {
                info!("SePatchStrategy: SELINUXFS");

                // selinuxfs 没挂的话自己挂
                if !SELINUX_ENFORCE.exists() {
                    // 重挂 /proc，加 hidepid=2,gid=3009
                    cstr!("/proc").remount_with_data(cstr!("hidepid=2,gid=3009"))?;

                    // 从 mount_list 移除 /proc /sys，因为已经挂过了，退出时不要卸
                    self.mount_list.retain(|s| s != "/proc" && s != "/sys");

                    // 挂 selinuxfs
                    unsafe {
                        libc::mount(
                            raw_cstr!("selinuxfs"),
                            raw_cstr!(SELINUX_MNT),
                            raw_cstr!("selinuxfs"),
                            0,
                            ptr::null(),
                        )
                        .check_err()?;
                    }
                }

                // 劫持 /load（普通文件，不阻塞）
                mock_file(SELINUX_LOAD, MOCK_LOAD)?;
                // 劫持 /enforce（FIFO，阻塞 init）
                mock_fifo(SELINUX_ENFORCE, MOCK_ENFORCE)?;
            }
            SePatchStrategy::Legacy => {
                info!("SePatchStrategy: LEGACY");

                // /selinux_version 不存在就创建一个
                if !policy_ver.exists() {
                    drop(policy_ver.create(OFlag::O_RDONLY, 0o666)?);
                }

                // 用 FIFO 劫持 /selinux_version
                // init 调 selinux_android_load_policy() -> set_policy_index() -> open(policy_ver)
                // 打开时会阻塞，趁机接管
                mock_fifo(policy_ver, MOCK_VERSION)?;
            }
        }

        // fork 子进程
        // 父进程返回继续，子进程负责处理 init 的阻塞操作
        let pid = unsafe { libc::fork() };
        if pid != 0 {
            return Ok(());  // 父进程返回
        }

        // 以下是子进程

        let wait = Duration::from_millis(100);

        if matches!(strat, SePatchStrategy::Legacy) {
            // 忙等 selinuxfs 挂载完成
            while !SELINUX_ENFORCE.exists() {
                sleep(wait);
            }

            // Android 6.0 上 init 直接调 security_setenforce()，
            // 而且用 O_RDWR 打开 enforce 节点，FIFO 不会阻塞。
            // 变通：不劫持 enforce，改用 checkreqprot 阻塞。
            // Android 7.0-7.1 没这个问题，但为了简单，都用同样策略。

            mock_file(SELINUX_LOAD, MOCK_LOAD)?;
            mock_fifo(SELINUX_REQPROT, MOCK_REQPROT)?;

            // 这一步解除 init 在 set_policy_index() 的阻塞
            drop(MOCK_VERSION.open(OFlag::O_WRONLY)?);

            policy_ver.unmount()?;
        }
        match strat {
            SePatchStrategy::LdPreload => {
                // 打开 ack FIFO，会阻塞到 init-ld 写完策略
                let mut ack_fd = preload_ack().open(OFlag::O_WRONLY)?;

                let mut sepol = SePolicy::from_file(preload_policy());

                // 留一份原始策略副本
                preload_policy().copy_to(MOCK_LOAD)?;

                // 加载前删掉临时文件
                preload_policy().remove()?;
                preload_ack().remove()?;

                sepol.magisk_rules();
                sepol.load_rules(&rules);
                sepol.to_file(SELINUX_LOAD);

                self.restore_overlay_contexts();

                // 写 ack，解除 init-ld 的阻塞
                ack_fd.write_all("0".as_bytes())?;
            }
            SePatchStrategy::SelinuxFs => {
                // 打开 MOCK_ENFORCE，会阻塞到 init 调 security_getenforce()
                let mut mock_enforce = MOCK_ENFORCE.open(OFlag::O_WRONLY)?;

                self.cleanup_and_load(&rules);

                // 从真节点读，转发到 mock
                let mut data = vec![];
                SELINUX_ENFORCE
                    .open(OFlag::O_RDONLY)?
                    .read_to_end(&mut data)?;
                mock_enforce.write_all(&data)?;
            }
            SePatchStrategy::Legacy => {
                let mut sz = 0_usize;
                // 忙等 sepolicy 完全写完（看文件大小稳定了）
                loop {
                    let attr = MOCK_LOAD.get_attr()?;
                    if sz != 0 && sz == attr.st.st_size as usize {
                        break;
                    }
                    sz = attr.st.st_size as usize;
                    sleep(wait);
                }

                self.cleanup_and_load(&rules);

                // init 被 checkreqprot 阻塞，先写真节点，再打开 mock 解除阻塞
                SELINUX_REQPROT
                    .open(OFlag::O_WRONLY)?
                    .write_all("0".as_bytes())?;
                let mut v = vec![];
                MOCK_REQPROT.open(OFlag::O_RDONLY)?.read_to_end(&mut v)?;
            }
        }

        // init 解除阻塞，继续 restorecon + re-exec
        // 子进程退出
        std::process::exit(0);
    }
}
```

`Magisk` 用 `FIFO` 把 `init` 卡在 " 加载 `SELinux` 策略 " 这一步，趁机把策略换掉，再放 `init` 走，`init` 以为自己加载了原始策略，实际上加载的是 `Magisk` 改过的版本

接下来我们本来应该追`exec_init`，位于 [https://github.com/topjohnwu/Magisk/blob/master/native/src/init/mount.rs](https://github.com/topjohnwu/Magisk/blob/master/native/src/init/mount.rs)

但是该文件内部其他段落也比较重要，所以我一并分析了

```
pub(crate) fn switch_root(path: &Utf8CStr) {//把 / 从 ramdisk 换成 system 分区
    || -> LoggedResult<()> {
        debug!("Switch root to {}", path);
        let mut mounts = BTreeSet::new();
        let rootfs = Directory::open(cstr!("/"))?;
        for info in parse_mount_info("self") {//把旧根下的挂载点搬过去
            if info.target == "/" || info.target.as_str() == path.as_str() {
                continue;
            }
            if let Some(last_mount) = mounts
                .range::<String, _>((Unbounded, Excluded(&info.target)))
                .last()
                && info.target.starts_with(&format!("{}/", *last_mount))
            {
                continue;
            }

            let mut target = info.target.clone();
            let target = Utf8CStr::from_string(&mut target);
            let new_path = cstr::buf::default()
                .join_path(path)
                .join_path(info.target.trim_start_matches('/'));
            new_path.mkdirs(0o755).ok();
            target.move_mount_to(&new_path)?;
            mounts.insert(info.target);
        }
        chdir(path)?;//进 /system
        path.move_mount_to(cstr!("/"))?;// /system -> /
        chroot(cstr!("."))?;// 把 cwd 设为根

        debug!("Cleaning rootfs");
        rootfs.remove_all()?;// 删掉旧 ramdisk
        Ok(())
    }()
    .ok();
}
```

```
impl MagiskInit {

    pub(crate) fn prepare_data(&self) {

        // 创建 /data 目录
        debug!("Setup data tmp");
        cstr!("/data").mkdir(0o755).log_ok();
        // 把 tmpfs 挂载到 /data
        nix::mount::mount(
            Some(cstr!("magisk")),       // filesystem/source 名称
            cstr!("/data"),              // 挂载目标
            Some(cstr!("tmpfs")),        // filesystem 类型
            MsFlags::empty(),
            Some(cstr!("mode=755")),
        )
        .check_os_err("mount", Some("/data"), Some("tmpfs"))
        .log_ok();

        // 保存当前正在运行的 magiskinit

        cstr!("/init").copy_to(cstr!("/data/magiskinit")).ok();

        // 保存 Magisk 的 backup 数据
        cstr!("/.backup").copy_to(cstr!("/data/.backup")).ok();

        // 保存 overlay.d
        //
        // overlay.d 里面是 Magisk 在启动过程中需要处理的
        // overlay / early-init 相关内容
        cstr!("/overlay.d").copy_to(cstr!("/data/overlay.d")).ok();
    }
}
```

启动早期，真正的 `/data` 分区还没挂载，但 `Magisk` 需要在 `/data` 下放东西，所以在 `/data` 上挂一个 `tmpfs`（内存文件系统），临时充当 `/data`

```
pub(crate) fn exec_init(&mut self) {
    //卸载自己挂的东西，mount_list 里记的是 magiskinit 自己挂的临时挂载点（/proc、/sys 等）.rev() 反向遍历，后挂的先卸
    for path in self.mount_list.iter_mut().rev() {
        let path = Utf8CStr::from_string(path);
        if path.unmount().log().is_ok() {
            debug!("Unmount [{}]", path);
        }
    }

    unsafe {

        libc::execve(
            raw_cstr!("/init"),

            // 把原来的 argv 传给新的 init
            self.argv.cast(),

            // 把当前环境变量传给真正 init
            environ.cast(),
        )
        .check_err()
        .log_ok();
    }
    // execve() 失败 直接退出
    std::process::exit(1);
}
```

`magiskinit` 干完所有活后，执行真正的 `init`，把自己替换掉，回到了`init`

`magiskinit` 在启动时，通过 `patch_rc_scripts()` 和 `inject_magisk_rc()` 函数，将一段启动 `magiskd` 的配置写入系统的 `init.rc` 文件，大致如下

```
on post-fs-data
    exec u:r:magisk:s0 0 0 -- /data/adb/magisk/magisk --post-fs-data
```

此时，`magisk --post-fs-data` 作为一个客户端被启动。它的首要任务是尝试连接 `magiskd` 的 `Unix Domain Socket`由于 `magiskd` 尚未运行，连接会失败。

随后，客户端代码会调用 `connect_daemon()` 函数，根据源码逻辑，该函数会检测到 `Socket` 不存在，然后 `fork()` 一个子进程。这个子进程会调用 `daemon_entry()`，从而正式启动 `magiskd` 守护进程

###### magiskd

`Magisk`维护的守护进程

[https://github.com/topjohnwu/Magisk/tree/master/native/src/core](https://github.com/topjohnwu/Magisk/tree/master/native/src/core)

```
pub fn daemon_entry() {
    // 设置进程名
    // 把进程名改成 "magiskd"，方便在 ps 里识别
    set_nice_name(cstr!("magiskd"));
    android_logging();

    // 屏蔽信号（SIGKILL 除外）
    // 防止守护进程被意外终止，保证持久运行
    // 为什么：magiskd 是常驻进程，不能被随便杀掉
    SigSet::all().thread_set_mask().log_ok();

    // 重定向标准 I/O
    // stdout/stderr -> /dev/null（避免日志输出阻塞）
    // stdin -> /dev/zero（避免读取阻塞）
    if let Ok(null) = cstr!("/dev/null").open(OFlag::O_WRONLY).log() {
        dup2_stdout(null.as_fd()).log_ok();
        dup2_stderr(null.as_fd()).log_ok();
    }
    if let Ok(zero) = cstr!("/dev/zero").open(OFlag::O_RDONLY).log() {
        dup2_stdin(zero).log_ok();
    }

    // 创建独立会话
    // setsid() 使进程成为新会话的领导者，脱离控制终端
    // 这样即使终端关闭，magiskd 也不会收到 SIGHUP 被杀
    setsid().log_ok();

    // 设置 SELinux 上下文
    // 把自身上下文设为 "u:r:magisk:s0"
    // 这是 Magisk 的自定义域，拥有较高权限
    // 能执行各种系统级操作（这是 selinux.rs 注入策略时开的权限）
    if let Ok(mut current) = cstr!("/proc/self/attr/current").open(OFlag::O_WRONLY | OFlag::O_CLOEXEC) {
        let con = cstr!(MAGISK_PROC_CON);
        current.write_all(con.as_bytes_with_nul()).log_ok();
    }

    // 第六步：启动日志守护线程
    start_log_daemon();
    magisk_logging();
    info!("Magisk {MAGISK_FULL_VER} daemon started");

    // 读取配置与检测环境
    let is_emulator = ...;   // 检测是否为模拟器（通过 /dev/socket 等特征）
    let magisk_tmp = get_magisk_tmp();
    // 读取 MAIN_CONFIG 文件，判断是否处于 recovery 模式
    // 读取 SDK 版本（从 /system/build.prop 或 getprop）

    // 逃离 Cgroup 限制
    // Android 的 cgroup 会限制进程资源（CPU、内存等）
    // magiskd 把自己 PID 写入不同 cgroup 路径，避免被限制
    let pid = getpid().as_raw();
    switch_cgroup("/acct", pid);        // 账户相关 cgroup
    switch_cgroup("/dev/cg2_bpf", pid); // BPF cgroup
    switch_cgroup("/sys/fs/cgroup", pid); // 主 cgroup

    // 清理 pre-init 阶段的挂载
    // magiskinit 在 pre-init 阶段创建了一些挂载点（记录在 ROOTMNT 文件中）
    // 这里逐一卸载，避免污染系统视图
    // 同时删除 ROOTOVL 目录（overlay 的临时目录）
        // 第十步：初始化 MagiskD 单例
    // 把收集到的所有状态打包
    let daemon = MagiskD { sdk_int, is_emulator, is_recovery, exe_attr, ..Default::default() };
    MAGISKD.set(daemon).ok();

    // 创建 IPC Socket
    // 在 $MAGISK_TMP/.magisk/device/socket 创建 Unix domain socket
    // 设置权限 0666，打上 Magisk 的 SELinux 标签，供客户端连接
    let sock_path = ...;
    let Ok(sock) = UnixListener::bind(&sock_path).log() else { exit(1); };
    sock_path.follow_link().chmod(0o666).log_ok();
    sock_path.set_secontext(cstr!(MAGISK_FILE_CON)).log_ok();

    // 进入主循环
    // 无限循环接受客户端连接
    // 每个连接交给 handle_requests() 处理
    let daemon = MagiskD::get();
    for client in sock.incoming() {
        if let Ok(client) = client.log() {
            daemon.handle_requests(client);
        } else {
            exit(1);
        }
    }
}
// 1. 设置进程名 "magiskd"
// 2. 屏蔽所有信号（防止被杀）
// 3. 重定向 stdio 到 /dev/null、/dev/zero
// 4. setsid 创建独立会话
// 5. 设置 SELinux 上下文 u:r:magisk:s0
// 6. 启动日志线程
// 7. 检测环境（模拟器、SDK 版本、recovery）
// 8. 逃离 cgroup 限制
// 9. 清理 magiskinit 留下的挂载
// 10. 初始化 MagiskD 单例
// 11. 创建 IPC socket
// 12. 进入主循环，等客户端连接
```

随后调用`handle_requests处理权限校验`：

```
// daemon_entry() 进入主循环后，每来一个客户端连接，就调用这个函数
// 它负责验证客户端身份、检查权限，并把请求分发给对应的处理函数
// 采取 UID + SELinux 上下文 + 请求码权限矩阵
impl MagiskD {
    fn handle_requests(&'static self, mut client: UnixStream) {
        // 第一步：获取客户端凭证
        // SO_PEERCRED 是内核提供的机制，能拿到客户端的 UID 和 PID
        // 这些信息由内核填充，客户端无法伪造
        let Ok(cred) = client.peer_cred() else { return; };

        // 第二步：获取客户端 SELinux 上下文
        // SO_PEERSEC 能拿到客户端的 SELinux 上下文
        // 为什么需要这个：光看 UID 不够，同样是 uid=0 可能是不同身份的进程
        //   uid=0 + u:r:zygote:s0   -> Zygote 进程
        //   uid=0 + u:r:magisk:s0   -> magiskd 自己
        //   uid=0 + u:r:shell:s0    -> ADB shell
        // 只有加上 SELinux 上下文才能区分这些身份
        let mut context = ...;
        unsafe { libc::getsockopt(...) };

        // 第三步：身份识别
        let is_root = cred.uid == 0;          // 是否为 root 用户
        let is_shell = cred.uid == 2000;      // 是否为 ADB shell
        let is_zygote = &context == "u:r:zygote:s0"; // 是否来自 Zygote

        // 第四步：基础客户端校验
        // 如果客户端既不是 root，也不是 Zygote，也不是 Magisk 自己的客户端
        // 直接拒绝访问
        // 换句话说：普通 App 连不上 magiskd，除非它被 Magisk 认证过
        // is_client() 检查这个 PID 是否属于 Magisk Manager App
        if !is_root && !is_zygote && !self.is_client(cred.pid.unwrap_or(-1)) {
            client.write_pod(&RespondCode::ACCESS_DENIED.repr).log_ok();
            return;
        }

        // 第五步：读取请求码
        // 客户端连上后第一件事就是发一个请求码（RequestCode）
        // 告诉 magiskd 它想干什么（比如 POST_FS_DATA、SUPERUSER、ZYGISK 等）
        let mut code = -1;
        client.read_pod(&mut code).ok();

        // 第六步：权限检查（请求码权限矩阵）
        // 不是所有请求码都能被所有身份调用
        // 每个请求码有对应的最低权限要求
        match code {
            // 这些请求要求必须是 root
            RequestCode::POST_FS_DATA | RequestCode::LATE_START | RequestCode::BOOT_COMPLETE |
            RequestCode::ZYGOTE_RESTART | RequestCode::SQLITE_CMD | RequestCode::DENYLIST |
            RequestCode::STOP_DAEMON if !is_root => {
                client.write_pod(&RespondCode::ROOT_REQUIRED.repr).log_ok();
                return;
            }
            // 移除模块请求，允许 root 和 ADB shell
            RequestCode::REMOVE_MODULES if !is_root && !is_shell => {
                client.write_pod(&RespondCode::ACCESS_DENIED.repr).log_ok();
                return;
            }
            // Zygisk 相关请求，只允许来自 Zygote 上下文的客户端
            RequestCode::ZYGISK if !is_zygote => {
                client.write_pod(&RespondCode::ACCESS_DENIED.repr).log_ok();
                return;
            }
            _ => {}
        }

        // 第七步：发送 OK 响应
        if client.write_pod(&RespondCode::OK.repr).is_err() { return; }

        // 第八步：根据请求类型分发
        // 同步请求：在监听线程中直接处理，要求快速返回
        if code.repr < RequestCode::_SYNC_BARRIER_.repr {
            self.handle_request_sync(client, code);
        }
        // 异步请求：提交到线程池处理，避免阻塞主循环
        else if code.repr < RequestCode::_STAGE_BARRIER_.repr {
            ThreadPool::exec_task(move || {
                self.handle_request_async(client, code, cred);
            });
        }
        // 启动阶段请求：也提交到线程池，用于处理开机启动的不同阶段
        else {
            ThreadPool::exec_task(move || {
                self.boot_stage_handler(client, code);
            });
        }
    }
}
```

接着是`su_daemon_handler`

```
//   App 执行 su -> su 连接 magiskd -> 发 SuRequest
//   -> su_daemon_handler() 收到请求
//   -> 读请求、查策略、做决定
//   -> 允许则 fork root 进程执行命令
impl MagiskD {
    pub fn su_daemon_handler(&self, mut client: UnixStream, cred: UCred) {
        // 打日志：记录谁在请求 su
        debug!("su: request from uid=[{}], pid=[{}], client=[{}]",
               cred.uid, cred.pid.unwrap_or(-1), client.as_raw_fd());
        // 第一步：读请求
        // SuRequest 是客户端（su 命令）发来的结构体
        // 里面写着：要执行什么命令、切什么 uid、什么 SELinux 上下文
        let mut req = match client.read_decodable::<SuRequest>().log() {
            Ok(req) => req,
            Err(_) => {
                // 客户端在发请求过程中断了，直接拒绝
                warn!("su: remote process probably died, abort");
                client.write_pod(&SuPolicy::Deny.repr).ok();
                return;
            }
        };

        // 第二步：查策略
        // get_su_info 用请求者的 uid 去查这个 App 的策略
        let info = self.get_su_info(cred.uid as i32);

        {

            let mut access = info.access.lock();
            // 第三步：如果需要，弹框让用户决定
            // SuAppContext 负责和 Magisk Manager App 通信
            // 如果策略是 query，Manager 会弹授权框
            // 如果是 allow 或 deny，这里什么都不做
            let mut app = SuAppContext {
                cred,                            // 客户端凭证
                request: &req,                   // 请求内容
                info: &info,                     // 策略信息
                settings: &mut access.settings,  // 可变设置（用户决定后更新）
                sdk_int: self.sdk_int(),
            };

            // connect_app() 内部：
            //   1. 创建 FIFO 管道
            //   2. 通知 Magisk Manager：有新 su 请求
            //   3. Manager 弹框：允许 xxx 获取 root
            //   4. 用户点允许/拒绝
            //   5. 结果写回 FIFO
            //   6. magiskd 读到结果，更新 settings
            app.connect_app();
            access.refresh();
        }

        // 第四步：根据策略执行
        // 到这里策略已经确定（用户决定 或 缓存命中）
        //
        // 策略 = Allow：
        //   -> fork 子进程
        //   -> 子进程 setuid(0) 切成 root
        //   -> 设置 SELinux 上下文
        //   -> 执行 App 请求的命令
        //   -> 结果通过 socket 返回给 App
        // 策略 = Deny：
        //   -> 返回 SuPolicy::Deny
        //   -> App 拿不到 root
        //App 本身不会变成 root
        // App 进程始终是 uid=10123
        // su 是独立进程，App 通过管道指挥
    }
}
```

##### Magisk 的隐藏机制

###### Magic Mount

`Magisk`采取了`Magic Mount`配合`DenyList`来做到隐藏机制，是 `Magisk` 实现 `systemless` 修改的核心机制，所谓 “`systemless`”，指的是不修改真实的系统分区，而是在当前进程的挂载命名空间里，用 `bind mount` 把模块文件 “覆盖” 到系统分区的对应路径上，对上层应用来说，`/system/bin/xxx` 看起来就是系统原本的文件，实际可能来自模块目录

举例如下：

```
原版：/system/build.prop         ->  系统原版内容
Magisk 的模块：bind mount 盖上去
  /system/build.prop            ->  实际指向模块文件

原版：/system/bin/su            ->  不存在
Magisk：把 su 挂载上去
  /system/bin/su                ->  存在（Magisk 提供的）

DenyList 对这两种情况的操作：
摘掉 bind mount
  -> 露出原始 /system/build.prop
  -> App 看到原版（干净）
摘掉挂载
  -> su 的挂载点没了
  -> /system/bin/su 不存在
  -> App 看不到 su（干净）
```

###### DenyList

工作原理：利用挂载命名空间（`Mount Namespace`）实现 “视图隔离”

`Linux`系统允许每个进程拥有独立的文件系统视图，这就是挂载命名空间，`DenyList`正是利用了这一机制

隔离视图：当被列入`DenyList`的应用启动时，`Magisk`会为它创建一个全新的、干净的挂载命名空间。在这个 “房间” 里，`Magisk`的`su`二进制、模块目录等所有挂载痕迹都被移除了

效果：对于这个应用而言，它看到的系统是 “原厂、未修改” 的。当它执行检查`/system/bin/su、/data/adb/modules`等操作时，得到的会是 “文件不存在” 的结果

###### Zygisk

`Zygisk`会伪装成系统原生的 “`Native Bridge`” 库，在系统启动时被`Zygote`进程自动加载

加载后，`Zygisk`会在`Zygote`进程中注册一个 “`fork`前” 的回调，此时`Zygisk`会检查目标`App`是否在`DenyList`中，或者是否有`Zygisk`模块需要注入

由于注入在应用代码执行之前就已完成，应用没有机会在启动早期检测到这些模块的存在，而且这个时候进程还没触发`UID drop`

```
Zygote（root）
  │
  ├─ fork 出子进程
  │    │
  │    ├─ Zygisk preAppSpecialize hook     <- 权限窗口
  │    │    ├─ 检查 DenyList
  │    │    ├─ 卸载 Magisk 挂载点（需要 root）
  │    │    └─ 注入模块代码
  │    │
  │    ├─ 权限降级（UID drop）
  │    │    └─ 子进程变成 App uid（10123）
  │    │
  │    └─ App 代码开始执行
  │         └─ App 看到的是"干净"的文件系统
  │
  └─ Zygote 继续等待下一个请求
```

##### 对 Magisk 的检测

简单了解了`magisk`的注入原理后，应继续了解怎么进行检测

###### 一、/proc/self/maps 检测 Zygisk 注入

`Zygisk` 会向目标进程注入一个动态库，该库被映射到进程的地址空间，并在内存中留下持久记录，`Shamiko` 无法将其移除

`Shamiko` 的功能在于清理文件系统挂载，使得 `/data/adb/modules` 等目录在目标进程的挂载命名空间中不可见

然而，`Zygisk` 的注入发生在文件系统隐藏之前，其动态库已经通过 `mmap` 映射到进程内存中，因此文件系统层面的隐藏无法消除内存中的映射痕迹

读 `/proc/self/maps`，一行一行看

如果看到

```
7f0000000000-7f0000100000 r-xp 00000000 fd:01 12345 /data/adb/modules/xxx/zygisk.so
```

这就是 `Zygisk` 注入痕迹

可以查询这些关键词

```
zygisk
/data/adb/modules/
libriru
libzygisk
```

搜到就是有 `Zygisk`，但关键词只能抓没改名的

于是我们可以不匹配关键词，改白名单校验，只看可执行段，路径不在白名单里的就可疑

```
合法：/system/*  /apex/*  /data/app/*  /data/dalvik-cache/*  /dev/ashmem/*  以及匿名段
可疑：其他任何带路径的可执行映射
```

```
int check_exec_mapping(void) {
    FILE *fp = fopen("/proc/self/maps", "r");
    if (!fp) return 0;

    char line[512];
    while (fgets(line, sizeof(line), fp)) {
        if (!strstr(line, "r-xp") && !strstr(line, "rwxp"))
            continue;

        char *path = strchr(line, '/');
        if (!path) continue;

        if (!is_legal_path(path)) {   // 白名单判断
            fclose(fp);
            return 1;
        }
    }
    fclose(fp);
    return 0;
}
```

还可以用 `dl_iterate_phdr()` 枚举所有加载的 `ELF`，回调里检查 `dlpi_name` 和 `base/size`

这是因为`maps` 是 `/proc` 文本接口，`dl_iterate_phdr` 走的是进程内的 `link_map` 链表，想过滤 `maps`，只要 `hook read、fopen` 这些函数，把包含 `zygisk` 的行删掉，要绕这个需要`hook`函数，跳过 `Zygisk` 节点

只要 `Zygisk` 在跑，`maps` 里就有记录，`Zygisk` 模块必须被 `dlopen` 进来才能加载，所以它一定在链接器视野里，`Shamiko` 清理的是文件系统挂载视图，不清理内存映射

###### 二、挂载信息结构化分析与设备号一致性检查

`Magisk` 把 `/data` 的文件挂到 `/system` 上，路径说是 `/system`，但设备号暴露了它其实来自 `/data`

比如模块里有个 `su`，实际在 `/data/adb/modules/xxx/system/bin/su`

`Magisk` 把它 `bind mount` 到 `/system/bin/su`

`bind mount` 之后，`/proc/self/mountinfo` 里多一行，描述 `/system/bin/su` 这个挂载点

这一行的设备号，是 `data` 分区的，不是 `system` 分区的

正常情况，`/system/bin/su` 来自 `system` 分区，设备号跟 `/system` 一样，但现在`/system/bin/su` 被 `bind mount` 后，设备号变成 `data` 分区的

两种查法

**查法一，直接看设备号**

```
1. 读 /proc/self/mountinfo
2. 拿到 /data 分区的设备号
3. 遍历所有挂载点
4. 如果挂载点在 /system、/vendor 下
   但设备号等于 /data 的设备号
   → Magic Mount
```

**查法二，设备号一致性**

```
1. 从 mountinfo 拿 /system/bin/su 的设备号，比如 0:48
2. 调 stat("/system/bin/su")，拿 st_dev，比如 0:32
3. 两个不一样 → 挂载源被换过
```

###### 三、SELinux Oracle

`Magisk` 往 `SELinux` 策略里塞了 `magisk` 域的规则，普通 `App` 查不了，但 `app_zygote` 能查

上述我们看到`Magisk` 加载时会往策略里注入规则，比如 `untrusted_app -> magisk` 的访问权限

普通 `App` 在 `untrusted_app` 域，的确不能直接查策略，但 `app_zygote` 域可以调 `selinux_check_access()`，查规则存不存在

前提是设备启用 `App Zygote`，厂商没收紧 `app_zygote` 域的策略

`AndroidManifest` 里配

```
<application android:zygotePreload>
    <service android:
             android:isolatedProcess="true"
             android:useAppZygote="true" />
</application>
```

这样会触发应用私有 `Zygote`，`preload` 阶段以 `app_zygote` 域执行

然后查这些规则

```
(app_zygote    -> magisk, ...)      是否存在 allow
(untrusted_app -> magisk, ...)      是否存在 allow
(magisk        -> system_file, ...) 是否存在越权 allow
```

```
allow   → 该规则存在，策略被注入，判定成立
deny    → 该规则不存在，可能没注入，也可能探针选错了组合
失败    → 域权限不足，这个域当不了 oracle
```

#### KernelSU

`Magisk` 是用户空间的伪装，它在文件系统里把 `su` 等文件藏起来，但进程内存里仍会留下痕迹，复杂的检测手段依然能发现

`KernelSU`是内核空间的隔离它的核心逻辑全部在内核层实现，对于未授权的 `App`，`KernelSU` 根本不会把挂载点应用到它的命名空间，这意味着，`App` 看到的就是一个完全干净、未经修改的原生系统

哎简单来说，`Magisk`，是刷 `boot` 镜像，劫持 `init`，在启动早期插入 `magiskinit`，开 `magiskd` 进程，维护 `su`，而`KernelSU`刷内核 (内置模式) 或加载内核模块(`LKM`)，`hook` 系统调用，拦截 `su` 请求，开 `ksud` 进程，维护 `su` 和模块

##### 提权原理

###### init

[https://github.com/tiann/KernelSU/blob/main/kernel/core/init.c](https://github.com/tiann/KernelSU/blob/main/kernel/core/init.c)

```
// L109-L113加载模式判断
// 内置模式随内核启动，SELinux 尚未 enforce，KernelSU 无需手动注入策略，系统加载策略时会自然带上
// LKM 模式在系统启动后动态加载，SELinux 已 enforce，必须手动调用 apply_kernelsu_rules() 注入规则
#ifdef MODULE
    // current->pid != 1 说明不是内核启动时加载的
    // 而是开机后 insmod 动态加载的 LKM 模式
    // 此时 SELinux 已经 enforce，后面必须手动注入策略
    ksu_late_loaded = (current->pid != 1);
#else
    // 编译进内核，跟着内核一起启动
    // 此时 SELinux 还没 enforce，后面不需要手动注入
    ksu_late_loaded = false;
#endif

// 准备一个全局的高权限 cred
// 后面授权 root 时直接替换进程 cred 指针即可
ksu_cred = prepare_creds();
if (!ksu_cred) {
    pr_err("prepare cred failed!\n");
    return -ENOSYS;
}

ksu_init_symbol_resolver();  // 解析 selinux_state、selinux_policy 等未导出符号的地址
ksu_syscall_hook_init();     // 准备改 syscall table，记录每个被 hook 的条目以便恢复
ksu_feature_init();          // 初始化功能子系统，管理运行时功能开关，比如 SELinux 隐藏
ksu_sulog_init();            // 初始化 su 日志，记录谁在什么时候请求了 root
ksu_adb_root_init();         // 初始化 adb root，让 adb shell 可以通过 su 拿到 root
ksu_lsm_hook_init();         // 初始化 LSM hook，挂到内核 LSM 框架上拦截权限检查
ksu_selinux_hide_init();     // 初始化 SELinux 隐藏，对 app UID 的查询结果做净化
ksu_supercalls_init();       // 初始化 supercall 通信接口，用户态和内核态的主要通道
ksu_app_profile_init();      // 初始化 App Profile 权限模型，管理每个 App 的 su 授权

// LKM 模式路径
if (ksu_late_loaded) {
    pr_info("late load mode, skipping kprobe hooks\n");

    // 手动往内核 SELinux 策略里注入 KernelSU 的规则
    // 因为 late load 时 SELinux 已经 enforce
    // 不注入的话 ksud 和 su 都跑不起来
    // 注入的规则里有 untrusted_app -> ksu 的 binder 权限
    // 这就是 SELinux Oracle 攻击查的那条
    apply_kernelsu_rules();

    cache_sid();                 // 缓存 SELinux SID
    setup_ksu_cred();            // 设置 ksu_cred

    // 给当前进程提权，因为 late load 是 ksud 触发的
    // ksud 需要 root 才能继续干活
    escape_to_root_for_init();

    ksu_allowlist_init();
    ksu_load_allow_list();       // 从 /data/adb/ksu/.allowlist 加载白名单
    ksu_syscall_hook_manager_init();
    ksu_throne_tracker_init();   // 扫描 /data/app 找 Manager APK
    ksu_observer_init();
    ksu_file_wrapper_init();

    ksu_boot_completed = true;
    track_throne(false);

    // 如果是 permissive 就强制 enforce
    // 避免留下 permissive 的痕迹被检测
    if (!getenforce()) {
        pr_info("Permissive SELinux, enforcing\n");
        setenforce(true);
    }
} else {
    // 内置模式，启动早期加载，SELinux 还没 enforce
    // 不需要注入策略，后面系统加载策略时会自然带上
    ksu_syscall_hook_manager_init();
    ksu_allowlist_init();
    ksu_throne_tracker_init();
    ksu_ksud_init();             // 注册 kprobe 拦截 execve，ksud启动
    ksu_file_wrapper_init();
}

#ifdef MODULE
#ifndef CONFIG_KSU_DEBUG
    // 删掉模块在 /sys/module/ 下的 kobject
    // 正常加载的模块在 /sys/module/kernelsu/ 下会有一堆文件
    // 删掉之后这些就看不到了，是 KernelSU 隐藏自己的手段之一
    kobject_del(&THIS_MODULE->mkobj.kobj);
#endif
#endif

// 卸载
void __exit kernelsu_exit(void)
{
    ksu_syscall_hook_manager_exit();
    ksu_supercalls_exit();
    if (!ksu_late_loaded)
        ksu_ksud_exit();

    synchronize_rcu();

    ksu_observer_exit();
    ksu_throne_tracker_exit();
    ksu_allowlist_exit();
    ksu_selinux_hide_exit();
    ksu_lsm_hook_exit();
    ksu_adb_root_exit();
    ksu_sulog_exit();
    ksu_feature_exit();

    put_cred(ksu_cred);
}
```

`kernel/ksu.c` 是 `KernelSU` 内核模块的入口，负责整个模块的初始化和卸载

那么用户空间执行 `su` 后，`KernelSU` 是在哪里接管这次请求的？又是怎么处理的？

追到`kernel/sucompat.c`

```
// sucompat.c
// 让白名单进程执行 /system/bin/su 时，实际执行的是 ksud，并拿到 root

#define SU_PATH "/system/bin/su"
#define SH_PATH "/system/bin/sh"

bool ksu_su_compat_enabled __read_mostly = true;
// 全局开关，可以通过 feature handler 运行时开闭

static bool is_ksud_exists()
{
    struct path path;
    if (kern_path(KSUD_PATH, 0, &path) < 0) {
        return false;
    }
    path_put(&path);
    return true;
}
// 检查 ksud 文件

long ksu_handle_faccessat_sucompat(int orig_nr, struct pt_regs *regs)
{
    if (!ksu_is_allow_uid_for_current(current_uid().val)) {
        goto do_orig_facessat;
        // 不在白名单，直接走原始 faccessat
    }

    filename_user = (const char __user **)&PT_REGS_PARM2(regs);

    char path[sizeof(su_path) + 1];
    memset(path, 0, sizeof(path));
    strncpy_from_user_nofault(path, *filename_user, sizeof(path));
    // 从用户态把文件路径拷进来

    if (unlikely(!memcmp(path, su_path, sizeof(su_path)))) {
        // 路径匹配 /system/bin/su
        old_cred = override_creds(ksu_cred);
        if (is_ksud_exists()) {
            orig_filename = *filename_user;
            *filename_user = ksud_user_path();
            // 把文件名换成 ksud 路径
            ret = ksu_syscall_table[orig_nr](regs);
            // 用新路径重新执行 faccessat
            revert_creds(old_cred);
            *filename_user = orig_filename;
            return ret;
        } else {
            revert_creds(old_cred);
        }
    }

do_orig_facessat:
    return ksu_syscall_table[orig_nr](regs);
}
// App 执行 su 前会先查 /system/bin/su 在不在
// 这里把查询路径改成 ksud，让 App 以为 su 存在

long ksu_handle_stat_sucompat(int orig_nr, struct pt_regs *regs)
{
    // 同 faccessat，只是换成 stat
    // App 用它确认 su 的权限和属性
}

static long ksu_handle_execve_sucompat_common(const char __user **filename_user,
                                              const char __user *const __user *argv_user, unsigned long envp,
                                              bool execveat, int orig_nr, struct pt_regs *regs)
{
    if (execveat && ((int)PT_REGS_SYSCALL_PARM1(regs) != AT_FDCWD || (int)PT_REGS_PARM5(regs) != 0))
        goto do_orig_execve;
        // execveat 只处理 AT_FDCWD 且 flags 为 0 的普通调用

    if (!ksu_is_allow_uid_for_current(current_uid().val))
        goto do_orig_execve;
        // 不在白名单，走原始 execve

    addr = untagged_addr((unsigned long)*filename_user);
    fn = (const char __user *)addr;
    memset(path, 0, sizeof(path));

    ret = strncpy_from_user(path, fn, sizeof(path));
    if (ret < 0) {
        goto do_orig_execve;
    }

    if (likely(memcmp(path, su_path, sizeof(su_path))))
        goto do_orig_execve;
        // 路径不是 /system/bin/su，走原始 execve

    pr_info("sys_execve su found\n");

    tmp_fd = get_unused_fd_flags(O_CLOEXEC);
    // 分配一个 fd，用来打开 ksud

    old_cred = override_creds(ksu_cred);
    ksud_file = filp_open(KSUD_PATH, O_PATH, 0);
    // 用高权限 cred 打开 ksud
    revert_creds(old_cred);

    if (IS_ERR(ksud_file)) {
        put_unused_fd(tmp_fd);
        goto do_orig_execve;
    }

    fd_install(tmp_fd, ksud_file);
    // 把打开的 ksud 装到 fd 上

    pending_sucompat = ksu_sulog_capture_sucompat(*filename_user, argv_user, GFP_KERNEL);

    // 保存原始寄存器参数
    orig_regs[0] = PT_REGS_SYSCALL_PARM1(regs);
    orig_regs[1] = regs->__PT_PARM2_REG;
    orig_regs[2] = regs->__PT_PARM3_REG;
    orig_regs[3] = regs->__PT_SYSCALL_PARM4_REG;
    orig_regs[4] = regs->__PT_PARM5_REG;

    // 把 execve 改造成 execveat
    // execve(file, argv, envp)
    // execveat(fd, file, argv, envp, flags)
    regs->__PT_PARM5_REG = AT_EMPTY_PATH;
    regs->__PT_SYSCALL_PARM4_REG = envp;
    regs->__PT_PARM3_REG = (unsigned long)argv_user;
    regs->__PT_PARM2_REG = empty_user_path();
    PT_REGS_SYSCALL_PARM1(regs) = tmp_fd;
    // fd 是刚打开的 ksud，路径传空，flags 用 AT_EMPTY_PATH
    // 这样 execveat 会执行 fd 指向的 ksud

    ret = escape_with_root_profile();
    // 根据 App Profile 提权，设置 UID、GID、capabilities、SELinux 域

    ksu_sulog_emit_pending(pending_sucompat, ret, GFP_KERNEL);

    ret = ksu_syscall_table[__NR_execveat](regs);
    // 执行改造后的 execveat，实际执行 ksud

    if (ret < 0) {
        ksu_close_fd(tmp_fd);
        // 失败，还原寄存器
        PT_REGS_SYSCALL_PARM1(regs) = orig_regs[0];
        regs->__PT_PARM2_REG = orig_regs[1];
        regs->__PT_PARM3_REG = orig_regs[2];
        regs->__PT_SYSCALL_PARM4_REG = orig_regs[3];
        regs->__PT_PARM5_REG = orig_regs[4];
    } else {
        su_fd = ksu_install_su_fd();
        // 成功后安装 su session fd，后续 App 通过它和 ksud 通信
    }
    return ret;

do_orig_execve:
    return ksu_syscall_table[orig_nr](regs);
}

long ksu_handle_execve_sucompat(const char __user **filename_user, int orig_nr, struct pt_regs *regs)
{
    return ksu_handle_execve_sucompat_common(filename_user, (const char __user *const __user *)PT_REGS_PARM2(regs),
                                             PT_REGS_PARM3(regs), false, orig_nr, regs);
}
// execve 入口，argv 在 PARM2，envp 在 PARM3

long ksu_handle_execveat_sucompat(const char __user **filename_user, int orig_nr, struct pt_regs *regs)
{
    return ksu_handle_execve_sucompat_common(filename_user, (const char __user *const __user *)PT_REGS_PARM3(regs),
                                             PT_REGS_SYSCALL_PARM4(regs), true, orig_nr, regs);
}
// execveat 入口，argv 在 PARM3，envp 在 PARM4
// 比 execve 多一个 PARM1 是 fd，PARM5 是 flags
```

拦截 `faccessat、stat、execve、execveat` 四个系统调用，把路径从 `su` 换成 `ksud`，对于 `execve` 还额外改寄存器把调用改造成 `execveat`，用 `fd` 方式执行 `ksud`，最后调 `escape_with_root_profile()` 提权

但是改`cred`过了`dac`， `mac`也要过才行，不然这个`root`和没用没啥区别，这时我们看到`rules.c`，正是他给 `KernelSU` 在 `SELinux` 策略里开了合法的 `permissive` 域

```
// kernel/selinux/rules.c
// 往内核 SELinux 策略里注入 KernelSU 的规则
void apply_kernelsu_rules()
{
    struct selinux_policy *pol, *old_pol;
    struct policydb *db;

    mutex_lock(&selinux_state.policy_mutex);
    // 加锁，防止多个进程同时改策略

    old_pol = rcu_dereference_protected(selinux_state.policy, lockdep_is_held(&selinux_state.policy_mutex));
    backup_sepolicy = ksu_dup_sepolicy(old_pol);
    // 先备份一份当前策略，用于后续恢复

    pol = ksu_dup_sepolicy(old_pol);
    // 再复制一份出来，在副本上改

    if (IS_ERR(pol)) {
        goto out_unlock;
    }

    db = &pol->policydb;

    ksu_type(db, KERNEL_SU_DOMAIN, "domain");
    // 建一个 su 域，KERNEL_SU_DOMAIN

    ksu_permissive(db, KERNEL_SU_DOMAIN);
    // 让 su 域变成 permissive，不受 SELinux 约束

    ksu_typeattribute(db, KERNEL_SU_DOMAIN, "mlstrustedsubject");
    ksu_typeattribute(db, KERNEL_SU_DOMAIN, "netdomain");
    ksu_typeattribute(db, KERNEL_SU_DOMAIN, "bluetoothdomain");
    // 给 su 域加上属性组，方便后面批量授权

    ksu_type(db, KERNEL_SU_FILE, "file_type");
    ksu_typeattribute(db, KERNEL_SU_FILE, "mlstrustedobject");
    ksu_allow(db, "domain", KERNEL_SU_FILE, ALL, ALL);
    // 建一个文件类型 KERNEL_SU_FILE，任何域都能访问

    ksu_allow(db, KERNEL_SU_DOMAIN, ALL, ALL, ALL);
    // su 域可以操作一切

    if (db->policyvers >= POLICYDB_VERSION_XPERMS_IOCTL) {
        ksu_allowxperm(db, KERNEL_SU_DOMAIN, ALL, "blk_file", ALL);
        ksu_allowxperm(db, KERNEL_SU_DOMAIN, ALL, "fifo_file", ALL);
        ksu_allowxperm(db, KERNEL_SU_DOMAIN, ALL, "chr_file", ALL);
        ksu_allowxperm(db, KERNEL_SU_DOMAIN, ALL, "file", ALL);
    }
    // 允许 ioctl 操作

    ksu_allow(db, "init", KERNEL_SU_DOMAIN, ALL, ALL);
    // init 可以调 su 域，因为 ksud 是 init 启动的

    ksu_allow(db, "servicemanager", KERNEL_SU_DOMAIN, "dir", "search");
    ksu_allow(db, "servicemanager", KERNEL_SU_DOMAIN, "dir", "read");
    ksu_allow(db, "servicemanager", KERNEL_SU_DOMAIN, "file", "open");
    ksu_allow(db, "servicemanager", KERNEL_SU_DOMAIN, "file", "read");
    ksu_allow(db, "servicemanager", KERNEL_SU_DOMAIN, "process", "getattr");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "process", "sigchld");
    // 从 Magisk 抄的规则，让 servicemanager 能访问 su

    ksu_allow(db, "logd", KERNEL_SU_DOMAIN, "dir", "search");
    ksu_allow(db, "logd", KERNEL_SU_DOMAIN, "file", "read");
    ksu_allow(db, "logd", KERNEL_SU_DOMAIN, "file", "open");
    ksu_allow(db, "logd", KERNEL_SU_DOMAIN, "file", "getattr");
    // 允许 su 域写日志到 logd

    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "fd", "use");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "fifo_file", "write");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "fifo_file", "read");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "fifo_file", "open");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "fifo_file", "getattr");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "unix_stream_socket", "read");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "unix_stream_socket", "write");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "unix_stream_socket", "connectto");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "unix_stream_socket", "getopt");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "unix_stream_socket", "getattr");
    // 允许任何域通过 fd、fifo、unix socket 和 su 域通信

    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "memfd_file", "execute");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "memfd_file", "getattr");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "memfd_file", "map");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "memfd_file", "read");
    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "memfd_file", "write");
    // 允许使用 su 域创建的 memfd

    ksu_allow(db, "hwservicemanager", KERNEL_SU_DOMAIN, "dir", "search");
    ksu_allow(db, "hwservicemanager", KERNEL_SU_DOMAIN, "file", "read");
    ksu_allow(db, "hwservicemanager", KERNEL_SU_DOMAIN, "file", "open");
    ksu_allow(db, "hwservicemanager", KERNEL_SU_DOMAIN, "process", "getattr");
    // bootctl 相关

    ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "binder", ALL);
    // 任何域都能向 su 域发 binder 调用
    // 这就是 SELinux Oracle 攻击查的那条

    ksu_allow(db, "system_server", KERNEL_SU_DOMAIN, "process", "getpgid");
    ksu_allow(db, "system_server", KERNEL_SU_DOMAIN, "process", "sigkill");
    // 允许 system_server 杀掉 su 进程

    rcu_assign_pointer(selinux_state.policy, pol);
    // 原子替换策略

    synchronize_rcu();
    // 等所有 in-flight RCU reader 退出

    ksu_destroy_sepolicy(old_pol);
    // 销毁旧策略

    reset_avc_cache();
    // 清缓存

out_unlock:
    mutex_unlock(&selinux_state.policy_mutex);
}
```

###### Ksud

不同于`Magisk`的通过修改文件`init.rc`来启动守护进程，`ksu`采取的是`hook read、fstat、input_event` ，`init` 调 `read` 时如果是 `init.rc` 就替换 `file_operations`，之后 `init` 读 `init.rc` 读到 `EOF` 时追加 `KERNEL_SU_RC`，`fstat` 钩子负责把 `st_size` 改大让 `init` 能读到追加内容，`execve` 钩子则在 `init` 第二阶段注入 `SELinux` 规则、在 `zygote` 启动时加载白名单，下面细看

```
// runtime/ksud_integration.c

void __init ksu_ksud_init()
{
    int ret;

    ksu_syscall_table_hook(__NR_read, ksu_sys_read, &orig_sys_read);
    ksu_syscall_table_hook(__NR_fstat, ksu_sys_fstat, &orig_sys_fstat);
    // 拦 init 的 read 和 fstat，之后 init 读 init.rc 会先过这里

    ret = register_kprobe(&input_event_kp);
    // 拦 input_event，用于音量键安全模式检测

    INIT_WORK(&stop_input_hook_work, do_stop_input_hook);
}

static long ksu_sys_read(const struct pt_regs *regs)
{
    unsigned int fd = PT_REGS_SYSCALL_PARM1(regs);
    char __user **buf_ptr = (char __user **)&PT_REGS_PARM2(regs);
    size_t *count_ptr = (size_t *)&PT_REGS_PARM3(regs);

    ksu_handle_sys_read(fd, buf_ptr, count_ptr);
    // 检查这个 fd 是不是 init.rc
    return orig_sys_read(regs);
}

static void ksu_handle_sys_read(unsigned int fd, char __user **buf_ptr, size_t *count_ptr)
{
    struct file *file = fget(fd);
    if (!file) {
        return;
    }
    ksu_install_rc_hook(file);
    fput(file);
}

static void ksu_install_rc_hook(struct file *file)
{
    if (!is_init_rc(file)) {
        return;
    }

    static bool rc_hooked = false;
    if (rc_hooked) {
        return;
    }
    rc_hooked = true;
    stop_init_rc_hook();
    // 已经找到目标文件，卸掉 read/fstat 的 syscall hook

    load_module_rc_once();
    // 加载模块自定义 rc

    memcpy(&fops_proxy, file->f_op, sizeof(struct file_operations));
    // 复制一份 file_operations，不能直接改原结构体，它在只读内存

    orig_read = file->f_op->read;
    if (orig_read) {
        fops_proxy.read = read_proxy;
    }
    orig_read_iter = file->f_op->read_iter;
    if (orig_read_iter) {
        fops_proxy.read_iter = read_iter_proxy;
    }

    file->f_op = &fops_proxy;
    // 替换指针，之后 init 对该文件的所有 read 都走 read_proxy
}

static bool is_init_rc(struct file *fp)
{
    if (strcmp(current->comm, "init")) {
        return false;

    }
    if (!d_is_reg(fp->f_path.dentry)) {
        return false;
    }
    const char *short_name = fp->f_path.dentry->d_name.name;
    if (strcmp(short_name, "init.rc")) {
        return false;
    }
    char path[256];
    char *dpath = d_path(&fp->f_path, path, sizeof(path));
    if (strcmp(dpath, "/system/etc/init/hw/init.rc")) {
        return false;
    }
    return true;
}

static ssize_t read_proxy(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    ssize_t ret = 0;
    size_t append_count;

    if (ksu_rc_pos && ksu_rc_pos < ksu_rc_len)
        goto append_ksu_rc;
    if (ksu_rc_pos >= ksu_rc_len && module_rc_pos < module_rc_len)
        goto append_module_rc;

    ret = orig_read(file, buf, count, pos);
    if (ret != 0) {
        return ret;
    }
    // init 读到 EOF 了

append_ksu_rc:
    if (ksu_rc_pos < ksu_rc_len) {
        append_count = ksu_rc_len - ksu_rc_pos;
        if (append_count > count - ret)
            append_count = count - ret;
        copy_to_user(buf + ret, KERNEL_SU_RC + ksu_rc_pos, append_count);
        // 把 KERNEL_SU_RC 追加到读缓冲区
        ksu_rc_pos += append_count;
        ret += append_count;
    }

append_module_rc:
    if (module_rc_pos < module_rc_len && (size_t)ret < count) {
        // 追加模块自定义 rc
    }

    return ret;
}
// init 读文件是循环读，返回 0 表示 EOF
// 利用这个时机追加内容，init 以为这些命令原本就在 init.rc 里

static long ksu_sys_fstat(const struct pt_regs *regs)
{
    unsigned int fd = PT_REGS_SYSCALL_PARM1(regs);
    void __user *statbuf = (void __user *)PT_REGS_PARM2(regs);
    bool is_rc = false;
    long ret;

    struct file *file = fget(fd);
    if (file) {
        if (is_init_rc(file)) {
            is_rc = true;
            load_module_rc_once();
        }
        fput(file);
    }

    ret = orig_sys_fstat(regs);

    if (is_rc) {
        void __user *st_size_ptr = statbuf + offsetof(struct stat, st_size);
        long size, new_size;
        size_t extra = ksu_rc_len + module_rc_len;
        if (!copy_from_user_nofault(&size, st_size_ptr, sizeof(long))) {
            new_size = size + extra;
            copy_to_user_nofault(st_size_ptr, &new_size, sizeof(long));
            // 把 st_size 改大
        }
    }
    return ret;
}
// init 读文件前会先 fstat 拿大小
// 不改 st_size 的话 init 按原始大小读，追加的内容读不到

void ksu_handle_execveat_ksud(const char *path, struct user_arg_ptr *argv)
{
    static const char app_process[] = "/system/bin/app_process";
    static bool first_zygote = true;
    static const char system_bin_init[] = "/system/bin/init";
    static bool init_second_stage_executed = false;

    if (unlikely(!memcmp(path, system_bin_init, sizeof(system_bin_init) - 1) && argv)) {
        char buf[16];
        if (!init_second_stage_executed && check_argv(*argv, 1, "second_stage", buf, sizeof(buf))) {
            // init 进入第二阶段，SELinux 策略已加载、即将 enforce
            ksu_selinux_hide_handle_second_stage();
            apply_kernelsu_rules();
            cache_sid();
            setup_ksu_cred();
            init_second_stage_executed = true;
        }
    }

    if (unlikely(first_zygote && !memcmp(path, app_process, sizeof(app_process) - 1) && argv)) {
        char buf[16];
        if (check_argv(*argv, 1, "-Xzygote", buf, sizeof(buf))) {
            // zygote 启动
            on_post_fs_data();
            first_zygote = false;
            ksu_stop_ksud_execve_hook();
            // 后续不再需要 execve hook
        }
    }
}
```

##### 检测与对抗

###### 时间侧信道

`KernelSU` 在 `sucompat.c` 里对 `faccessat`、`stat`、`execve` 挂了 `kprobe` 钩子，把 `su` 重定向到 `ksud`，这些钩子在内核里引入额外执行路径，让被 `hook` 的系统调用耗时明显增加

正常情况下 `faccessat` 稳定快于 `fchownat`，大概 50 到 100 纳秒，但 `faccessat` 被 `kprobe` 拦了之后，路径变成 `syscall → kernel → hook 层 → 返回`，耗时涨到 200 到 500 纳秒，直接反超 `fchownat`

检测的核心思路是，`faccessat` 在 `KSU` 的 `hook` 列表里，`fchownat` 不在，两个都是轻量系统调用，正常耗时接近，一个被拖慢了，两个的耗时对比就会翻转

具体是绑定到大核稳定测量，收集 `__NR_faccessat` 和 `__NR_fchownat` 各几百次，用 `NEON` 并行比较两个耗时数组，统计 `faccessat > fchownat` 的次数

```
// 简化示意，实际用 NEON 并行比较
int faccessat_times[N], fchownat_times[N];
// ... 各采集 N 次 ...

int anomaly = 0;
for (int i = 0; i < N; i++) {
    if (faccessat_times[i] > fchownat_times[i]) {
        anomaly++;
    }
}

if (anomaly > THRESHOLD) {
    // 存在 hook
}
```

阈值不能定太低，正常设备上受调度、温度、中断影响，`faccessat` 偶尔也会慢于 `fchownat`，采样次数要够多，取统计比例而不是单次比较

选 `faccessat` 而不是别的系统调用，是因为它在 `KSU` 的 `hook` 列表里，而且简单、快速、失败无副作用，适合做基线对照，`fchownat` 同样轻量但不在 `hook` 列表里，做对照项

`KernelSU` 官方在 `v2.1.2` 里换了新的 `sucompat` 实现，通过 `LSM` `hook` 和调用黑名单大幅缓解了侧信道，但检测和对抗还在继续

我这部分没太明白，各位可以参考我找的一些大佬们的文章

```
看雪分析文章   https://bbs.kanxue.com/thread-288928-1.htm
官方 Issue     https://github.com/tiann/KernelSU/issues/2782
修复 Commit    814d65cc28 "kernel: Fix side channel attack"
v2.1.2 说明    https://github.com/tiann/KernelSU/releases/tag/v2.1.2
```

###### SELinux Oracle

几乎同上，都是 `app_zygote` 查 `live sepolicy`

`KernelSU` 加载时调 `apply_kernelsu_rules()` 往内核 `SELinux` 策略里注入规则

```
ksu_allow(db, "domain", KERNEL_SU_DOMAIN, "binder", ALL);
```

这条规则允许任何域（包括 `untrusted_app`）向 `KernelSU` 的 `su` 域发 `binder` 调用

普通 `App` 在 `untrusted_app` 域，查不了全局 `SELinux` 策略，但 `App Zygote` 进程是例外，它要切 `SELinux` 上下文才能创建隔离服务，所以系统给了它调 `selinux_check_access` 的权限

检测方在 `AndroidManifest.xml` 里配

```
<application android:zygotePreload>
    <service android:
             android:isolatedProcess="true"
             android:useAppZygote="true" />
</application>
```

`useAppZygote` 加 `isolatedProcess` 会触发应用私有 `Zygote`，`zygotePreloadName` 指向的类在 `preload` 阶段以 `app_zygote` 域执行

然后在代码里查规则

```
// app_zygote 域下执行
boolean hasKsuRule = SELinux.checkSELinuxAccess(
    "u:r:untrusted_app:s0",   // 源
    "u:r:ksu:s0",             // 目标
    "binder",                  // 类别
    "call"                     // 权限
);
if (hasKsuRule) {
    // 存在 KernelSU
}
```

可参考

```
官方 Issue   https://github.com/tiann/KernelSU/issues/3458
DirtySepolicy      LSPosed 团队 PoC，通过 app_zygote 查询 live sepolicy
selinux_hide 源码  kernel/feature/selinux_hide.c
```

本人才疏学浅，水平实在有限，各位高手们找出纰漏后及时联系我，我会及时编辑，学习，修改

望共勉

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)