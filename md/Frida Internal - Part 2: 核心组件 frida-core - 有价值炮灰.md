> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [evilpan.com](https://evilpan.com/2022/04/09/frida-core/)

> 前文已经介绍了 frida 中的核心组件 frida-gum 以及对应的 js 接口 gum-js，但仅有这些基础功能并不能让 frida 成为如此受欢迎的 Instrumentation (hook) 框架。

前文已经介绍了 frida 中的核心组件 `frida-gum` 以及对应的 js 接口 `gum-js`，但仅有这些基础功能并不能让 frida 成为如此受欢迎的 Instrumentation (hook) 框架。为了实现一个完善框架或者说工具，需要实现许多系统层的功能。比如进程注入、进程间通信、会话管理、脚本生命周期管理等功能，屏蔽部分底层的实现细节并给最终用户提供开箱即用的操作接口。而这一切的实现都在 `frida-core` 之中，正如名字所言，这其中包含了 frida 相关的大部分关键模块和组件，比如 frida-server、frida-gadget、frida-agent、frida-helper、frida-inject 以及之间的互相通信底座。本文主要节选其中关键的部分进行分析和介绍。

传送门:

*   [Frida Internal - Part 1: 架构、Gum 与 V8](https://evilpan.com/2022/04/05/frida-internal/)
*   [Frida Internal - Part 2: frida-core (本文)](https://evilpan.com/2022/04/09/frida-core/)
*   [Frida Internal - Part 3: frida-java-bridge 与 ART hook](https://evilpan.com/2022/04/17/frida-java/)

如果曾经浏览过 frida-core 的代码，会发现其大部分源文件都是 _.vala_ 后缀的。这其实是 GNOME 中使用的一个高级语言，和传统高级语言不同的是 vala 代码会被编译器先编译成 C 代码，然后再编译成二进制文件，因此也可以认为 vala 语言是 C 的一个语法糖拓展。

[Vala](https://wiki.gnome.org/Projects/Vala/) 使用 glib 的 `GObject` 类型系统来构造类和接口以实现面向对象，其语法有点类似于 `C#`，支持许多现代语言的高级特性，包括但不限于接口、属性、内存管理、异常、lambda、信号等等。Vala 既可以通过 API 文件访问已有的 C 库文件，也可以从 C 中很容易调用 Vala 的方法。

一个简单的 Vala 代码示例如下:

```
// test.vala
int main (string[] args) {
    stdout.printf ("nargs=%d\n", args.length);
    foreach (string arg in args) {
        stdout.printf ("%s\n", arg);
    }
    return 0;
}
```

编译运行也非常简单:

```
$ valac test.vala
$ ./test hello world
nargs=3
./test
hello
world

$ rabin2 -l test
[Linked libraries]
libglib-2.0.so.0
libc.so.6

2 libraries
```

Vala 程序也可以直接编译为 C 代码，使用 `valac -C test.vala` 即可生成对应的 `test.c` 文件。 更多的示例程序可以查看 [Vala 的官方文档](https://wiki.gnome.org/Projects/Vala/Documentation#Sample_Code)。

在上文中我们介绍了 frida-gum 和 gum-js，这二者都有对应的 release devkit 可以下载，frida-core 也一样。我们在 frida-core-devkit 中可以获取到编译好的静态库、头文件以及简单的示例程序，下面就以接口为着手点进行分析。

以 Android 平台为例，最常用的方式是先将 `frida-server` 推送到设备端启动，然后在本地使用 `frida-tools` 去加载 JS 脚本并执行 hook 操作。`frida-tools` 是基于 Python 的 binding 编写的，本质上还是调用了 `frida-core`，连接设备并加载脚本的过程如下所示:

```
// 获取设备句柄
FridaManager *manager = frida_device_manager_new ();
FridaDeviceList *devices = frida_device_manager_enumerate_devices_sync (manager, NULL, &error);
FridaDevice * local_device = frida_device_list_get (devices, i);

// 连接设备上的指定进程
FridaSession *session = frida_device_attach_sync (local_device, target_pid, FRIDA_REALM_NATIVE, NULL, &error);

// 创建脚本
options = frida_script_options_new ();
frida_script_options_set_name (options, "example");
frida_script_options_set_runtime (options, FRIDA_SCRIPT_RUNTIME_QJS);
script = frida_session_create_script_sync (session,
         "console.log('hello');",
         options, NULL, &error);

// 加载脚本
g_signal_connect (script, "message", G_CALLBACK (on_message), NULL);
frida_script_load_sync (script, NULL, &error);
```

虽然这里用的是 C 的接口，但实际上代码是在 `vala` 中以类方法的方式定义的，以 `frida_device_attach_sync` 这个方法为例，其定义在 `src/frida.vala` 中:

```
namespace {
    // ...
    public class Device : Object {
        // ...
        public Session attach_sync (uint pid, SessionOptions? options = null,
                Cancellable? cancellable = null) throws Error, IOError {
            var task = create<AttachTask> ();
            task.pid = pid;
            task.options = options;
            return task.execute (cancellable);
        }
    }
}
```

因此 vala 编译后的 C 代码符号也是可以比较容易对应到实际代码的，这对于开发和调试有很大的帮助。

虽然连接、加载脚本的逻辑是在 PC 端编写和执行的，但实际操作还是在设备端，即 `frida-server` 所运行的系统中。

该进程的主函数定义在 `server/server.vala` 中:

```
namespace Frida.Server {
private static int main (string[] args) {
    Environment.init ();
    // ...
    return run_application (endpoint_params, options, on_ready);
}
private static int run_application (EndpointParameters endpoint_params, ControlServiceOptions options, ReadyHandler on_ready) {
    // ...
    TemporaryDirectory.always_use ((directory != null) ? directory : DEFAULT_DIRECTORY);

    application = new Application (new ControlService (endpoint_params, options));
    // ...
    return application.run ();
}
// ...
}
```

`Application` 实际上是一层简单的封装，其 `run` 方法会间接调用到 `servcie.start`，即构造函数中传入的 `ControlService`。该类定义在 _src/control-service.vala_ 中，包含 3 个主要的属性:

*   HostSession host_session
*   EndpointParameters endpoint_params
*   ControlServiceOptions options

其中最重要的是 `HostSession`，该属性是一个接口类型，基于面向对象的多态方式实现了不同平台下的实现，如下所示:

```
public ControlService (EndpointParameters endpoint_params, ControlServiceOptions? options = null) {
    HostSession host_session;
#if WINDOWS
    var tempdir = new TemporaryDirectory ();
    host_session = new WindowsHostSession (new WindowsHelperProcess (tempdir), tempdir);
#endif
#if DARWIN
    host_session = new DarwinHostSession (new DarwinHelperBackend (), new TemporaryDirectory (),
        opts.report_crashes);
#endif
#if LINUX
    var tempdir = new TemporaryDirectory ();
    host_session = new LinuxHostSession (new LinuxHelperProcess (tempdir), tempdir, opts.report_crashes);
#endif
#if FREEBSD
    host_session = new FreebsdHostSession ();
#endif
#if QNX
    host_session = new QnxHostSession ();
#endif
```

`HostSession` 的接口定义在 _lib/base/session.vala_ 中，接口如下所示:

```
public interface HostSession : Object {

public abstract async void ping (uint interval_seconds, Cancellable? cancellable) throws GLib.Error;

public abstract async HashTable<string, Variant> query_system_parameters (Cancellable? cancellable) throws GLib.Error;
public abstract async HostApplicationInfo get_frontmost_application (HashTable<string, Variant> options,
    Cancellable? cancellable) throws GLib.Error;
public abstract async HostApplicationInfo[] enumerate_applications (HashTable<string, Variant> options,
    Cancellable? cancellable) throws GLib.Error;
public abstract async HostProcessInfo[] enumerate_processes (HashTable<string, Variant> options,
    Cancellable? cancellable) throws GLib.Error;

public abstract async void enable_spawn_gating (Cancellable? cancellable) throws GLib.Error;
public abstract async void disable_spawn_gating (Cancellable? cancellable) throws GLib.Error;
public abstract async HostSpawnInfo[] enumerate_pending_spawn (Cancellable? cancellable) throws GLib.Error;
public abstract async HostChildInfo[] enumerate_pending_children (Cancellable? cancellable) throws GLib.Error;
public abstract async uint spawn (string program, HostSpawnOptions options, Cancellable? cancellable) throws GLib.Error;
public abstract async void input (uint pid, uint8[] data, Cancellable? cancellable) throws GLib.Error;
public abstract async void resume (uint pid, Cancellable? cancellable) throws GLib.Error;
public abstract async void kill (uint pid, Cancellable? cancellable) throws GLib.Error;
public abstract async AgentSessionId attach (uint pid, HashTable<string, Variant> options,
    Cancellable? cancellable) throws GLib.Error;
public abstract async void reattach (AgentSessionId id, Cancellable? cancellable) throws GLib.Error;
public abstract async InjectorPayloadId inject_library_file (uint pid, string path, string entrypoint, string data,
    Cancellable? cancellable) throws GLib.Error;
public abstract async InjectorPayloadId inject_library_blob (uint pid, uint8[] blob, string entrypoint, string data,
    Cancellable? cancellable) throws GLib.Error;

public signal void spawn_added (HostSpawnInfo info);
public signal void spawn_removed (HostSpawnInfo info);
public signal void child_added (HostChildInfo info);
public signal void child_removed (HostChildInfo info);
public signal void process_crashed (CrashInfo crash);
public signal void output (uint pid, int fd, uint8[] data);
public signal void agent_session_detached (AgentSessionId id, SessionDetachReason reason, CrashInfo crash);
public signal void uninjected (InjectorPayloadId id);
}
```

其中包括了应用枚举、进程查找、进程注入、进程启动以及各类信号回调的接口原型，基于这些接口实现对目标进程的劫持和动态修改。由于我们重点是分析 Android 系统上的实现，因此后文主要分析 `LinuxHostSession`，对于其他平台的分析也是类似的。

`LinuxHostSession` 的代码在 _src/linux/linux-host-session.vala_ 中，其中定义了几个重要的属性，分别是:

*   injector: 类型为 Linjector
*   agent: 类型为 AgentDescriptor
*   system_server_agent: 类型为 SystemServerAgent
*   robo_launcher: 类型为 RoboLauncher

它们都在构造函数中进行初始化，分别实现了进程注入、server 与注入代码之间的进程间通信以及远程调用行为，下文会分别进行简要介绍。

从接口名称来看，进程注入的实现大概率是 `inject_library_file`，表示注入一个动态库到目标进程中。所注入的动态库称为 agent。

回到注入的实现上，frida-server 是怎么实现进程注入的呢？ 根据已有经验，由于我们以 root 权限启动，在 Linux 中一般有多种实现进程注入的方式，比如 ptrace、`/proc/pid/mem`，DLL/SO 劫持、LD_PERLOAD 等，因此实际实现还要回到代码。

Android 中实现进程注入的方法如下:

```
protected override async Future<IOStream> perform_attach_to (uint pid, HashTable<string, Variant> options,
        Cancellable? cancellable, out Object? transport) throws Error, IOError {
    PipeTransport.set_temp_directory (tempdir.path);

    var t = new PipeTransport ();

    var stream_request = Pipe.open (t.local_address, cancellable);

    uint id;
    string entrypoint = "frida_agent_main";
    string agent_parameters = make_agent_parameters (t.remote_address, options);
    var linjector = injector as Linjector;
#if HAVE_EMBEDDED_ASSETS
    id = yield linjector.inject_library_resource (pid, agent, entrypoint, agent_parameters, cancellable);
#else
    id = yield linjector.inject_library_file (pid, Config.FRIDA_AGENT_PATH, entrypoint, agent_parameters, cancellable);
#endif
    injectee_by_pid[pid] = id;

    transport = t;

    return stream_request;
}
```

确实是使用了 `inject_library_xxx` 的方式，`HAVE_EMBEDDED_ASSETS` 为真时使用的是从自身代码中释放出来的 agent 库，位置在:

```
agent = new AgentDescriptor (PathTemplate ("frida-agent-<arch>.so"),
    new Bytes.static (blob32.data),
    new Bytes.static (blob64.data),
    new AgentResource[] {
        new AgentResource ("frida-agent-arm.so", new Bytes.static (emulated_arm.data), tempdir),
        new AgentResource ("frida-agent-arm64.so", new Bytes.static (emulated_arm64.data), tempdir),
    },
    AgentMode.INSTANCED,
    tempdir);
```

这也是 Android 平台中 frida-server 的默认实现。

`Linjector` 定义在 _src/linux/linjector.vala_ ，即 Linux 平台上通用的进程注入实现，调用链路为:

*   inject_library_resource
*   inject_library_file_with_template
*   helper.inject_library_file
*   helper.`_do_inject` (_src/linux/frida-helper-backend.vala_)

`_do_inject` 不使用 vala 而是直接使用 C 实现，代码在 _src/linux/frida-helper-backend-glue.c_ :

```
void
_frida_linux_helper_backend_do_inject (FridaLinuxHelperBackend * self, guint pid, const gchar * path, const gchar * entrypoint, const gchar * data, const gchar * temp_path, guint id, GError ** error)
{
  params.open_impl = frida_resolve_libc_function (pid, "open");
  params.close_impl = frida_resolve_libc_function (pid, "close");
  params.write_impl = frida_resolve_libc_function (pid, "write");
  params.syscall_impl = frida_resolve_libc_function (pid, "syscall");

#if defined (HAVE_GLIBC)
// ...
#elif defined (HAVE_UCLIBC)
// ...
#elif defined (HAVE_ANDROID)
  params.dlopen_impl = frida_resolve_android_dlopen (pid);
  params.dlopen_flags = RTLD_LAZY;
  params.dlclose_impl = frida_resolve_linker_address (pid, dlclose);
  params.dlsym_impl = frida_resolve_linker_address (pid, dlsym);
#endif

  instance = frida_inject_instance_new (self, id, pid, temp_path);
  frida_inject_instance_attach (instance, &saved_regs, error);
  // ...
  frida_inject_instance_start_remote_thread (instance, &exited, error);
  frida_inject_instance_detach (instance, &saved_regs, NULL);
}
```

`frida_inject_instance_attach` 中定义了真正的 attach 实现:

```
static gboolean
frida_inject_instance_attach (FridaInjectInstance * self, FridaRegs * saved_regs, GError ** error)
{
  const pid_t pid = self->pid;

  if (can_seize)
    ret = ptrace (PTRACE_SEIZE, pid, NULL, PTRACE_O_TRACEEXEC);
  else
    ret = ptrace (PTRACE_ATTACH, pid, NULL, NULL);
  // ...
}
```

因此我们可以得出结论，**frida 的进程注入是通过 ptrace 实现的**，注入之后会在远端进程分配一段内存将 agent 拷贝过去并在目标进程中执行代码，**执行完成后会 detach 目标进程**。

这也是为什么在 frida 先连接上目标进程后还可以用 gdb 等调试器连接，而先 gdb 连接进程后 frida 就无法再次连上的原因。

`frida-agent` 注入到目标进程并启动后会启动一个新进程与 host 进行通信，从而 host 可以给目标进行发送命令，比如执行代码，激活 / 关闭 hook，同时也能接收到目标进程的执行返回以及异步事件信息等。

在上节中调用 `inject_library` 指定了注入动态库后执行的的函数符号为 `frida_agent_main`，该函数也是由 vala 生成而来，源文件定义在 _lib/agent/agent.vala_ 中:

```
namespace Frida.Agent {
    public void main (string agent_parameters, ref Frida.UnloadPolicy unload_policy, void * injector_state) {
        if (Runner.shared_instance == null)
            Runner.create_and_run (agent_parameters, ref unload_policy, injector_state);
        else
            Runner.resume_after_transition (ref unload_policy, injector_state);
    }
    // ...
```

关键的调用路径如下:

*   Runner.create_and_run
*   new Runner (…)
*   shared_instance.run
*   Runner.run
*   Runner.start

其中 `Runner.run` 函数在调用完 `start` 后会进入 `main_loop` 循环，直至进程退出或者收到 server 的解除命令。

`Runner.start` 的作用是准备 Interceptor 以及 GumJS 的 ScriptBackend，并连接到启动时指定的 `transport_uri` 建立通信隧道。

另外一个我觉得比较有意思的是 frida-server 中除了注入到目标进程的 agent，还有一个 agent，即 `system_server_agent`。

用过 `frida-tools` 的朋友应该都知道其中有个 `frida-ps` 命令，可以查看设备中的应用名称、包名甚至是 Icon，也可以查看当前运行在窗口顶部的应用。之前一直以为是通过 `pm list packages` 和 `dumpsys` 等命令实现的，看过代码之后才发现原来 frida-server 还对 `system_server` 进程进行了注入，并且所使用的 agent 就是 `system_server_agent`。

如果不了解 `system_server`，可以先回顾一下笔者之前的文章 “[Android 用户态启动流程分析](https://evilpan.com/2020/11/08/android-init/) "，其中提到在 Android 系统启动时，zygote 启动的第一个进程就是 `system_server`，这也是系统中第一个启动的 Java 进程，其中包含了 `ActivityManagerService`、 `PackageManagerService` 等系统服务。frida-server 对其进行注入，一方面是为了获取系统中的应用信息，另一方面也是为了可以实现 `spawn` 方式启动应用的功能。

以获取当前窗口中展示在最上层的应用功能为例，接口为 `get_frontmost_application`，最终的实现在 `SystemServerAgent`:

```
public async HostApplicationInfo get_frontmost_application (FrontmostQueryOptions options,
        Cancellable? cancellable) throws Error, IOError {
    var scope = options.scope;
    var scope_node = new Json.Node.alloc ().init_string (scope.to_nick ());

    Json.Node result = yield call ("getFrontmostApplication", new Json.Node[] { scope_node }, cancellable);
    // ...
}
```

这实际上是调用了 _src/linux/agent/system-server.js_ 中导出的方法:

```
rpc.exports = {
  getFrontmostApplication(scope) {
    return performOnJavaVM(() => {
      const pkgName = getFrontmostPackageName();
      if (pkgName === null)
        return null;

      const appInfo = packageManager.getApplicationInfo(pkgName, 0);

      const appLabel = loadAppLabel.call(appInfo, packageManager).toString();
      const pid = computeAppPids(getAppProcesses()).get(pkgName) ?? 0;
      const parameters = (scope !== 'minimal') ? fetchAppParameters(pkgName, appInfo, scope) : null;

      return [pkgName, appLabel, pid, parameters];
    });
  }
  // ...
}
```

这部分 JS 代码也是内置到 frida-server 中的，基于 gum-js 实现了对 `system_server` 的 Java 代码调用和劫持功能。

除了 `frida-server`，另外一个比较常用的模块就是 [frida-gadget](https://frida.re/docs/gadget/) 了。攻击者可以通过重打包修改动态库的依赖或者修改 smali 代码去实现向三方应用注入 gadget，当然也可以通过任意其他加载动态库的方式。

gadget 本身是一个动态库，在加载到目标进程中后会马上触发 ctor 执行指定代码，默认情况下是挂起当前进程并监听在 27042 端口等待 Host 的连接并恢复运行。其文件路径为 _lib/gadget/gadget.vala_ ，启动入口为 `Frida.Gadget.load`:

```
public void load (Gum.MemoryRange? mapped_range, string? config_data, int * result) {
    location = detect_location (mapped_range);
    config = (config_data != null)
                ? parse_config (config_data)
                : load_config (location);

    Gum.Cloak.add_range (location.range);

    interceptor = Gum.Interceptor.obtain ();
    interceptor.begin_transaction ();
    exceptor = Gum.Exceptor.obtain ();

    // try-catch
    var interaction = config.interaction;
    if (interaction is ScriptInteraction) {
        controller = new ScriptRunner (config, location);
    } else if (interaction is ScriptDirectoryInteraction) {
        controller = new ScriptDirectoryRunner (config, location);
    } else if (interaction is ListenInteraction) {
        controller = new ControlServer (config, location);
    } else if (interaction is ConnectInteraction) {
        controller = new ClusterClient (config, location);
    } else {
        // throw "Invalid interaction specified"
    }
    interceptor.end_transaction ();

    if (!wait_for_resume_needed)
        resume ();
    // ..
    start (request);
}
```

正如文档中所说，Gadget 启动时会去指定路径搜索配置文件，并根据其设置进入不同的运行模式，默认的配置文件如下所示:

```
{
  "interaction": {
    "type": "listen",
    "address": "127.0.0.1",
    "port": 27042,
    "on_port_conflict": "fail",
    "on_load": "wait"
  }
}
```

即使用 `listen` 模式，监听在 27042 端口并等待连接。除了 listen 以外，还支持以下几种模式:

*   **connect**: Gadget 启动后主动连接到指定地址;
*   **script**: 启动后直接加载指定的 JavaScript 文件；
*   **script-directory**: 启动后加载指定目录下的所有 JavaScript 文件；

值得一提的是，`script-directory` 模式可以用来实现系统级别的插件功能，比如在指定目录中编写一个 `twitter.js`，用于实现针对 Twitter 的 Hook 功能，比如自定义过滤、自动点赞等等；另外写一个 `twitter.config` 来过滤应用:

```
{
  "filter": {
    "executables": ["Twitter"],
    "bundles": ["com.twitter.twitter-mac"],
    "objc_classes": ["Twitter"]
  }
}
```

这样就只有满足指定条件的进程才会加载 `twitter.js` 脚本，从而实现特定应用的持久化 Hook 功能。熟悉 Xposed 或者 Tweak 开发的应该都知道这意味着什么。

在 frida-core 中有许多需要进程间通信的行为，比如 frida-server 需要与注入到目标进程中的 agent 进行通信，通知目标进程开启或者关闭 Interceptor；agent 同样也需要与 host 进行通信，在 gum-js 中将 `console.log` 或者 `send` 的消息发给 host，或者接收一些异步的应用退出和异常事件等。而其中这些进程间的交互都是通过 [D-Bus](https://en.wikipedia.org/wiki/D-Bus) 去实现的。

`D-Bus` 是一种基于消息的进程间通信机制，全称为 Desktop Bus，最初从 FreeDesktop 中的模块独立出来。其主要作用是提供在 Linux 操作系统桌面环境中的组件通信，比如 GNOME 或 KDE。D-Bus 使用 C 语言开发，提供了 GLib、Qt、Python 等编程接口，在 frida-core 中主要使用其 Vala 接口进行集成。

下面从实际代码看一个 D-Bus Vala 接口实现的 C/S 通信示例。 首先是服务端，定义一个 `DemoServer`， 内部定义了四个方法:

```
[DBus (name = "org.example.Demo")]
public class DemoServer : Object {

    private int counter;

    public int ping (string msg) {
        stdout.printf ("%s\n", msg);
        return counter++;
    }

    public int ping_with_signal (string msg) {
        stdout.printf ("%s\n", msg);
        pong(counter, msg);
        return counter++;
    }

    /* Including any parameter of type GLib.BusName won't be added to the
       interface and will return the dbus sender name (who is calling the method) */
    public int ping_with_sender (string msg, GLib.BusName sender) {
        stdout.printf ("%s, from: %s\n", msg, sender);
        return counter++;
    }

    public void ping_error () throws Error {
        throw new DemoError.SOME_ERROR ("There was an error!");
    }

    public signal void pong (int count, string msg);
}

[DBus (name = "org.example.DemoError")]
public errordomain DemoError
{
    SOME_ERROR
}

void on_bus_aquired (DBusConnection conn) {
    try {
        conn.register_object ("/org/example/demo", new DemoServer ());
    } catch (IOError e) {
        stderr.printf ("Could not register service\n");
    }
}

void main () {
    Bus.own_name (BusType.SESSION, "org.example.Demo", BusNameOwnerFlags.NONE,
                  on_bus_aquired,
                  () => {},
                  () => stderr.printf ("Could not aquire name\n"));

    new MainLoop ().run ();
}
```

其中使用 `Bus.own_name` 去连接总线 `org.example.Demo`，连接成功后注册了名为 `/org/example/demo` 的服务。客户端可以通过连接同样的总线和服务去获取到对应的远程对象接口，从而在本地实现远程调用。客户端示例如下:

```
[DBus (name = "org.example.Demo")]
interface Demo : Object {
    public abstract int ping (string msg) throws IOError;
    public abstract int ping_with_sender (string msg) throws IOError;
    public abstract int ping_with_signal (string msg) throws IOError;
    public signal void pong (int count, string msg);
}

void main () {
    /* Needed only if your client is listening to signals; you can omit it otherwise */
    var loop = new MainLoop();

    /* Important: keep demo variable out of try/catch scope not lose signals! */
    Demo demo = null;

    try {
        demo = Bus.get_proxy_sync (BusType.SESSION, "org.example.Demo",
                                                    "/org/example/demo");

        /* Connecting to signal pong! */
        demo.pong.connect((c, m) => {
            stdout.printf ("Got pong %d for msg '%s'\n", c, m);
            loop.quit ();
        });

        int reply = demo.ping ("Hello from Vala");
        stdout.printf ("%d\n", reply);

        reply = demo.ping_with_sender ("Hello from Vala with sender");
        stdout.printf ("%d\n", reply);

        reply = demo.ping_with_signal ("Hello from Vala with signal");
        stdout.printf ("%d\n", reply);

    } catch (IOError e) {
        stderr.printf ("%s\n", e.message);
    }
    loop.run();
}
```

客户端中需要定义与远程对象一致的 `interface`，并通过 `get_proxy` 请求获得远程对象的本地代理，从而实现透明的远程调用。注意的是编译需要添加对应的库:

```
$ valac --pkg gio-2.0 demo-server.vala
$ valac --pkg gio-2.0 demo-client.vala
```

从效果上看有点类似于 Android 的 binder 通信机制。但是 D-Bus 是一种更加上层的封装，在不同操作系统上可以使用不同的底层实现。在 Linxu 操作系统中通常是基于 UNIX socket 实现的多进程通信，当然也可以修改配置去通过 TCP socket 去实现。

frida 执行进程注入时同时也实现了 IPC 相关的初始化，在 _src/host-session-service.vala_ 中可以看到关键代码如下:

```
private async AgentEntry establish (uint pid, HashTable<string, Variant> options,
        Cancellable? cancellable) throws Error, IOError {
    // ...
    DBusConnection connection;
    var stream_request = yield perform_attach_to (pid, options, io_cancellable, out transport);
    IOStream stream = yield stream_request.wait_async (io_cancellable);

    connection = yield new DBusConnection (
        stream,
        ServerGuid.HOST_SESSION_SERVICE,
        AUTHENTICATION_SERVER | AUTHENTICATION_ALLOW_ANONYMOUS | DELAY_MESSAGE_PROCESSING,
        null, io_cancellable);
    controller_registration_id = connection.register_object (
        ObjectPath.AGENT_CONTROLLER,
        (AgentController) this);
    connection.start_message_processing ();
    provider = yield connection.get_proxy (null, ObjectPath.AGENT_SESSION_PROVIDER,
        DO_NOT_LOAD_PROPERTIES, io_cancellable);
}
```

这里使用了 `DBusConnection` 来连接总线，效果和 `own_name` 是类似的。其中总线的名称为 `ServerGuid.HOST_SESSION_SERVICE`，所注册的 RPC 服务名称为 `ObjectPath.AGENT_CONTROLLER`。另外这里还通过总线去获取了一个远程对象 `ObjectPath.AGENT_SESSION_PROVIDER`，用以实现透明的远程调用。这些服务的名称都定义在 _lib/base/session.vala_ 文件中:

```
namespace Frida {
    // ...
    namespace ServerGuid {
        public const string HOST_SESSION_SERVICE = "6769746875622e636f6d2f6672696461";
    }

    namespace ObjectPath {
        public const string HOST_SESSION = "/re/frida/HostSession";
        public const string AGENT_SESSION_PROVIDER = "/re/frida/AgentSessionProvider";
        public const string AGENT_SESSION = "/re/frida/AgentSession";
        public const string AGENT_CONTROLLER = "/re/frida/AgentController";
        public const string AGENT_MESSAGE_SINK = "/re/frida/AgentMessageSink";
        public const string CHILD_SESSION = "/re/frida/ChildSession";
        public const string TRANSPORT_BROKER = "/re/frida/TransportBroker";
        public const string PORTAL_SESSION = "/re/frida/PortalSession";
        public const string BUS_SESSION = "/re/frida/BusSession";
        public const string AUTHENTICATION_SERVICE = "/re/frida/AuthenticationService";
    }
}
```

通过搜索 `[DBus (name = "xxx")]` 可以看到 frida-core 中定义的所有 IPC 服务，感兴趣的可以有针对性地去进行深入分析。

拓展阅读:

*   [https://dbus.freedesktop.org/doc/dbus-tutorial.html](https://dbus.freedesktop.org/doc/dbus-tutorial.html)
*   [https://wiki.gnome.org/Projects/Vala/DBusServerSample](https://wiki.gnome.org/Projects/Vala/DBusServerSample)
*   [https://www.reddit.com/r/commandline/comments/13o581/dbus_vs_unix_sockets/](https://www.reddit.com/r/commandline/comments/13o581/dbus_vs_unix_sockets/)

在 [Android/Linux Root 的那些事儿](https://evilpan.com/2020/12/06/android-rooting/#selinux) 中曾经介绍过 Android 系统的安全访问控制体系，不是简单的 `uid=0` 就可以为所欲为的。frida 这种又注入进程又各种进程间通信的，显然违反了 SELinux 的默认规则，那么它是如何实现的呢？其实说起来也很简单，就是对当前系统的 SELinux 规则进行了 patch，本节就来分析下其具体是如何做的。

patch 的代码主要集中在 _lib/selinux/patch.c_ 文件中，代码就不贴了，主要流程可以简单描述如下:

1.  从 _/sys/fs/selinux/policy_ 文件中加载当前系统的 SELinux policy；
2.  在当前 sepolicy 中增加新的 SELinux 文件类型 `frida_file`；
3.  依次插入新的 SELinux 规则 (见后文)；
4.  通过写入 _/sys/fs/selinux/load_ 文件保存修改后的 sepolicy；
5.  (可选) 如果上述保存操作失败，就先临时将 SELinux 设置为 permissive 然后再保存一次，保存成功后重新恢复为 enforcing；

第 3 步中插入的 SELinux 规则如下:

```
typedef struct _FridaSELinuxRule FridaSELinuxRule;
struct _FridaSELinuxRule
{
  const gchar * sources[4];
  const gchar * target;
  const gchar * klass;
  const gchar * permissions[16];
};
static const FridaSELinuxRule frida_selinux_rules[] =
{
  { { "domain", NULL }, "domain", "process", { "execmem", NULL } },
  { { "domain", NULL }, "frida_file", "dir", { "search", NULL } },
  { { "domain", NULL }, "frida_file", "fifo_file", { "open", "write", NULL } },
  { { "domain", NULL }, "frida_file", "file", { "open", "read", "getattr", "execute", "?map", NULL } },
  { { "domain", NULL }, "frida_file", "sock_file", { "write", NULL } },
  { { "domain", NULL }, "shell_data_file", "dir", { "search", NULL } },
  { { "domain", NULL }, "zygote_exec", "file", { "execute", NULL } },
  { { "domain", NULL }, "$self", "process", { "sigchld", NULL } },
  { { "domain", NULL }, "$self", "fd", { "use", NULL } },
  { { "domain", NULL }, "$self", "unix_stream_socket", { "connectto", "read", "write", "getattr", "getopt", NULL } },
  { { "domain", NULL }, "$self", "tcp_socket", { "read", "write", "getattr", "getopt", NULL } },
  { { "zygote", NULL }, "zygote", "capability", { "sys_ptrace", NULL } },
  { { "?app_zygote", NULL }, "zygote_exec", "file", { "read", NULL } },
};
```

其中 `$self` 表示当前进程的 context，即 getcon 获取的值。在完成对 sepolicy 的动态修改后，frida 就可以正常的在 enforcing 的环境下运行了。

`frida-core` 在 `gum-js` 的基础上实现了操作系统层面的主要功能，比如进程注入、进程间通信、系统信息采集、系统权限管理等功能。正是因为这些核心组件的加入，使得 frida 成为了一个上手简单却又功能强大的动态分析框架。另外由于这些核心组件大部分使用 Vala 语言编写，在保障内存安全性的同时也提供了丰富的跨平台能力，支持 Android、iOS、Linux、MacOS、Windows 这些主流操作系统的不同架构。虽然本文仅针对 Android/Linux 操作系统的主要组件进行了介绍，但相信从中也能看出 frida-core 项目本身的代码风格架构设计，对于其他组件或者其他操作系统上的实现也是大同小异的，后续遇到具体问题的时候再深入分析对应组件即可。

> **版权声明**: 自由转载 - 非商用 - 非衍生 - 保持署名 ([CC 4.0 BY-SA](https://creativecommons.org/licenses/by-nc/4.0/))  
> **原文地址**: [https://evilpan.com/2022/04/09/frida-core/](https://evilpan.com/2022/04/09/frida-core/)  
> **微信订阅**: **『[有价值炮灰](http://t.evilpan.com/qrcode.jpg)』**  
> – _TO BE CONTINUED_.