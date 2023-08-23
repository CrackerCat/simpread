> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278533.htm)

> [原创] Frida-Core 源代码速通笔记

[原创] Frida-Core 源代码速通笔记

12 小时前 175

### [原创] Frida-Core 源代码速通笔记

 [![](http://passport.kanxue.com/upload/avatar/548/924548.png?1681269251)](user-home-924548.htm) [Tokameine](user-home-924548.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png) 2  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 12 小时前  175

前言
==

书接上文，在理解了 frida 是如何对代码进行 hook 的以后，接下来笔者打算研究一下 Frida 是如何与用户进行交互实现动态的 hook 。因此还是按照前文的逻辑，我们从 frida-core 开始。

> 前篇：《Frida-gum 源代码速通笔记》[https://bbs.kanxue.com/thread-278423.htm](https://bbs.kanxue.com/thread-278423.htm)

本文内容目录
======

本文主要涉及的是 Frida-core 模块的实现代码分析，大致内容包括：

*   Frida-Core
    *   Frida-Server
        *   进程注入
        *   frida-server 工作流程
        *   frida-agent
        *   frida-helper
    *   Frida-Gadget
    *   launchd
    *   总结 / 以及一个奇怪的问题

Frida-Core
==========

进程注入
----

首先要解决的第一个问题是，我们尽管知道 Frida-gum 能够对程序进行动态插桩，并也在上一节中介绍过它的工作原理，但是也正如我们所知，无论在 Android 还是 iOS 或者其他平台，进程和进程直接相互是透明的，就算对 Frida 来说目标进程并非透明的，但是对目标进程来说，至少 Frida 也该是透明的。但是现实情况显然是，二者往往需要相互之间识别并交互。

那么解决方案似乎也呼之欲出了，既然两个进程不能相互识别，那么让它们成为一个进程不就好了？

由于笔者本文主要是面向 iOS 的，因此下文中笔者会选择 darwin 平台的代码进行分析。对于 Android 平台，frida 的实现是有所不同的，不过大体的逻辑还是有些相似的，仅供参考。

本部分的代码主要定义在 `inject/src/darwin/darwin-host-session.vala` 中，可以注意到，它是由 vala 编写的语言，这对我来说有些陌生。不过好在它的语法结构和 C 很像，并且编译 vala 的过程其实就是把它先转为 C 代码再使用 C 的编译器完成的，因此大体上还是能够从语意上理解逻辑。

> 糟糕的是，我的 VSCode 不再支持代码跟踪了，即便装了插件也还是如此，要命。

```
//inject/src/darwin/darwin-host-session.vala
 
        private async uint inject_agent (uint pid, string agent_parameters, Cancellable? cancellable) throws Error, IOError {
            uint id;
 
            unowned string entrypoint = "frida_agent_main";
#if HAVE_EMBEDDED_ASSETS
            id = yield fruitjector.inject_library_resource (pid, agent, entrypoint, agent_parameters, cancellable);
#else
            string agent_path = Config.FRIDA_AGENT_PATH;
#if IOS || TVOS
            unowned string? cryptex_path = Environment.get_variable ("CRYPTEX_MOUNT_PATH");
            if (cryptex_path != null)
                agent_path = cryptex_path + agent_path;
#endif
            id = yield fruitjector.inject_library_file (pid, agent_path, entrypoint, agent_parameters, cancellable);
#endif
 
            return id;
        }

```

跟入 `inject_library_file` ：

```
//inject/src/darwin/fruitjector.vala
        public async uint inject_library_file (uint pid, string path, string entrypoint, string data,
                Cancellable? cancellable) throws Error, IOError {
            var id = yield helper.inject_library_file (pid, path, entrypoint, data, cancellable);
            pid_by_id[id] = pid;
            return id;
        }

```

因此再次跟入 `inject_library_file` ：

```
//inject/src/darwin/frida-helper-service.vala
        public async uint inject_library_file (uint pid, string path, string entrypoint, string data, Cancellable? cancellable)
                throws Error, IOError {
            return yield backend.inject_library_file (pid, path, entrypoint, data, cancellable);
        }

```

好吧，我们再次跟入：

```
//inject/src/darwin/frida-helper-backend.vala
        public async uint inject_library_file (uint pid, string path, string entrypoint, string data, Cancellable? cancellable)
                throws Error, IOError {
            return yield _inject (pid, path, null, entrypoint, data, cancellable);
        }

```

继续跟入 `_inject` ：

```
//inject/src/darwin/frida-helper-backend.vala
        private async uint _inject (uint pid, string path_or_name, MappedLibraryBlob? blob, string entrypoint, string data,
                Cancellable? cancellable) throws Error, IOError {
            yield prepare_target (pid, cancellable);
 
            var task = task_for_pid (pid);
            try {
                return _inject_into_task (pid, task, path_or_name, blob, entrypoint, data);
            } finally {
                deallocate_port (task);
            }
        }

```

可以看见此处将会使用 `_inject_into_task` 实现进程注入的方式，这里跟入 `_frida_darwin_helper_backend_inject_into_task` ，由于函数体比较大，这里省略部分代码：

```
guint
_frida_darwin_helper_backend_inject_into_task (FridaDarwinHelperBackend * self, guint pid, guint task, const gchar * path_or_name, FridaMappedLibraryBlob * blob,
    const gchar * entrypoint, const gchar * data, GError ** error)
{
//此处省略
//1. 初始化实例
  self_task = mach_task_self ();
 
  instance = frida_inject_instance_new (self, self->next_id++, pid);
  mach_port_mod_refs (self_task, task, MACH_PORT_RIGHT_SEND, 1);
  instance->task = task;
 
  resolver = gum_darwin_module_resolver_new (task, &io_error);
//此处省略
//2. 在进程的内存空间中开辟内存
  kr = mach_vm_allocate (task, &payload_address, instance->payload_size, VM_FLAGS_ANYWHERE);
  CHECK_MACH_RESULT (kr, ==, KERN_SUCCESS, "mach_vm_allocate(payload)");
  instance->payload_address = payload_address;
 
  kr = mach_vm_allocate (self_task, &agent_context_address, layout.data_size, VM_FLAGS_ANYWHERE);
  g_assert (kr == KERN_SUCCESS);
  instance->agent_context = (FridaAgentContext *) agent_context_address;
  instance->agent_context_size = layout.data_size;
 
  data_address = payload_address + layout.data_offset;
  kr = mach_vm_remap (task, &data_address, layout.data_size, 0, VM_FLAGS_OVERWRITE, self_task, agent_context_address,
      FALSE, &cur_protection, &max_protection, VM_INHERIT_SHARE);
//此处省略
//3. 修改内存段权限
  kr = mach_vm_protect (task, payload_address + layout.stack_guard_offset, layout.stack_guard_size, FALSE, VM_PROT_NONE);
  CHECK_MACH_RESULT (kr, ==, KERN_SUCCESS, "mach_vm_protect");
//4. 初始化实例
  if (!frida_agent_context_init (&agent_ctx, &details, &layout, payload_address, instance->payload_size, resolver, mapper, error))
    goto failure;
//5. 创建代码
  frida_agent_context_emit_mach_stub_code (&agent_ctx, mach_stub_code, resolver, mapper);
 
  frida_agent_context_emit_pthread_stub_code (&agent_ctx, pthread_stub_code, resolver, mapper);

```

正如注释中所述，其实就是通过 iOS 平台本身提供的 api 往进程的内存空间中开辟内存段。

关键是接下来的部分。如果该平台允许存在 rwx 段那么将执行如下代码：

```
  if (gum_query_is_rwx_supported () || !gum_code_segment_is_supported ())
  {
//1. 向进程内存空间中写入 mach_stub_code 的代码
    kr = mach_vm_write (task, payload_address + layout.mach_code_offset,
        (vm_offset_t) mach_stub_code, sizeof (mach_stub_code));
    CHECK_MACH_RESULT (kr, ==, KERN_SUCCESS, "mach_vm_write(mach_stub_code)");
//2. 向进程内存空间中写入 pthread_stub_code 的代码
    kr = mach_vm_write (task, payload_address + layout.pthread_code_offset,
        (vm_offset_t) pthread_stub_code, sizeof (pthread_stub_code));
    CHECK_MACH_RESULT (kr, ==, KERN_SUCCESS, "mach_vm_write(pthread_stub_code)");
//3. 将权限改为 rx
    kr = mach_vm_protect (task, payload_address + layout.code_offset, page_size, FALSE, VM_PROT_READ | VM_PROT_EXECUTE);
    CHECK_MACH_RESULT (kr, ==, KERN_SUCCESS, "mach_vm_protect");
  }

```

> 注意，对于已越狱的 iOS 设备，这么做是可行的；但是对于非越狱设备，系统并不允许由用户分配带有执行权限的内存段，同理也不允许修改段段权限为可执行。

而对于不支持上述条件的情况，包括设备未越狱，那么则使用如下分支：

```
  else
  {
    GumCodeSegment * segment;
    guint8 * scratch_page;
    mach_vm_address_t code_address;
//1.创建一个新的内存段，此时它尚且没有执行权限
    segment = gum_code_segment_new (page_size, NULL);
//2. 将代码拷贝到该内存段
    scratch_page = gum_code_segment_get_address (segment);
    memcpy (scratch_page + layout.mach_code_offset, mach_stub_code, sizeof (mach_stub_code));
    memcpy (scratch_page + layout.pthread_code_offset, pthread_stub_code, sizeof (pthread_stub_code));
//3. 为代码构造具有可执行权限的内存段
    gum_code_segment_realize (segment);
    gum_code_segment_map (segment, 0, page_size, scratch_page);
//4。 将其映射到 code_address 去
    code_address = payload_address + layout.code_offset;
    kr = mach_vm_remap (task, &code_address, page_size, 0, VM_FLAGS_OVERWRITE, self_task, (mach_vm_address_t) scratch_page,
        FALSE, &cur_protection, &max_protection, VM_INHERIT_COPY);
 
    gum_code_segment_free (segment);
 
    CHECK_MACH_RESULT (kr, ==, KERN_SUCCESS, "mach_vm_remap(code)");
  }

```

> 笔者大致查了一下，在 iOS 中，程序的代码段都是需要经过签名验证的，因此对于未经过签名的代码段是会因此而产生异常的，所以在 iOS 上不能使用 smc 也是这个原因，因为代码要么不可变要么不可执行。但反过来，它们允许用户删除对代码段的执行权限，这是合法的。

这里我们跟入 `gum_code_segment_realize` 看看是如何得到可执行内存段的：

```
static gboolean
gum_code_segment_try_realize (GumCodeSegment * self)
{
  gchar * dylib_path;
  GumCodeLayout layout;
  guint8 * dylib_header;
  gsize dylib_header_size;
  guint8 * code_signature;
  gint res;
  fsignatures_t sigs;
//1. 创建临时文件 frida-XXXXXX.dylib
  self->fd = gum_file_open_tmp ("frida-XXXXXX.dylib", &dylib_path);
  if (self->fd == -1)
    return FALSE;
 
  gum_code_segment_compute_layout (self, &layout);
//2. 构造 mach 文件头
  dylib_header = g_malloc0 (layout.header_file_size);
  gum_put_mach_headers (dylib_path, &layout, dylib_header, &dylib_header_size);
//3. 构造 code signature
  code_signature = g_malloc0 (layout.code_signature_file_size);
  gum_put_code_signature (dylib_header, self->data, &layout, code_signature);
//4. 写入文件
  gum_file_write_all (self->fd, GUM_OFFSET_NONE, dylib_header,
      dylib_header_size);
  gum_file_write_all (self->fd, layout.text_file_offset, self->data,
      layout.text_size);
  gum_file_write_all (self->fd, layout.code_signature_file_offset,
      code_signature, layout.code_signature_file_size);
 
  sigs.fs_file_start = 0;
  sigs.fs_blob_start = GSIZE_TO_POINTER (layout.code_signature_file_offset);
  sigs.fs_blob_size = layout.code_signature_file_size;
//3. 添加签名
  res = fcntl (self->fd, F_ADDFILESIGS, &sigs);
 
  unlink (dylib_path);
 
  g_free (code_signature);
  g_free (dylib_header);
  g_free (dylib_path);
 
  return res == 0;
}

```

以上操作完成了构造一个 dylib 的行为，接下来程序将会调用 `gum_code_segment_try_map` 将该动态库映射到内存中：

```
static gboolean
gum_code_segment_try_map (GumCodeSegment * self,
                          gsize source_offset,
                          gsize source_size,
                          gpointer target_address)
{
  gpointer result;
 
  result = mmap (target_address, source_size, PROT_READ | PROT_EXEC,
      MAP_PRIVATE | MAP_FIXED, self->fd,
      gum_query_page_size () + source_offset);
 
  return result != MAP_FAILED;
}

```

注意到此处使用 `mmap` 去映射文件到 `target_address` 并给出了可读可执行的权限标志。接下来这段内存就已经被注入到进程中去了。

其实这个地方用到的就是一个小 trick。正如前文所说，iOS 不允许让一个 rw 页面变为 rx，也不允许让 rx 页面变为 rwx ，但是加载可执行文件的行为是不被禁止的，因为那属于正常的诉求，因此这里用需要注入的代码去构建可执行文件，最后再将其映射到内存中去，这样就没有修改任何页面了权限了。

> 其实实际原因是，对于正常组织的代码段，有一个 max_protection 去限制能够允许 mprotect 设定权限的范围，默认情况下会被限制为 rx，也就是说只允许对这段内存给出 rx 中的范围。但是通过 mmap 创建的内存段的 max_protection 是允许给出 rwx 权限的。所以实际是只要最后是用 mmap 去构建注入内存，总会有办法解决的。

> 您也可以参考本文：[https://www.codercto.com/a/63507.html](https://www.codercto.com/a/63507.html)

最后调用 `mach_vm_remap` 把这段内存重新映射回去即可。

frida-server
------------

在介绍了大致的 frida 进程注入的原理以后，接下来我们正式开始跟一下 frida 具体是如何开始工作的。

在启动 frida 后，程序将从 `run_application` 开始向下调用 `application.run` ，然后再往下调用 `start.begin`，此时，它将通过 `service.start` 去启动一个 `ControlService` ：

```
        public async void start (Cancellable? cancellable = null) throws Error, IOError {
            if (state != STOPPED)
                throw new Error.INVALID_OPERATION ("Invalid operation");
            state = STARTING;
 
            try {
//1. WebService 被启动
                yield service.start (cancellable);
 
                if (options.enable_preload) {
//2. 创建 BaseDBusHostSession
                    var base_host_session = host_session as BaseDBusHostSession;
                    if (base_host_session != null)
                        base_host_session.preload.begin (io_cancellable);
                }
 
                state = STARTED;
            } finally {
                if (state != STARTED)
                    state = STOPPED;
            }
        }

```

接下来我们跟入 `WebService` ：

```
        public async void start (Cancellable? cancellable) throws Error, IOError {
            frida_context = MainContext.ref_thread_default ();
            dbus_context = yield get_dbus_context ();
 
            cancellable.set_error_if_cancelled ();
 
            var start_request = new Promise ();
//1. handle_start_request 开始调度
            schedule_on_dbus_thread (() => {
                handle_start_request.begin (start_request, cancellable);
                return false;
            });
 
            _listen_address = yield start_request.future.wait_async (cancellable);
        } 
```

而 `handle_start_request` 会向下调用 `do_start` 完成具体的工作，包括监听地址，设置处理函数等：

```
        private async SocketAddress do_start (Cancellable? cancellable) throws Error, IOError {
            server = (Soup.Server) Object.new (typeof (Soup.Server),
                "tls-certificate", endpoint_params.certificate);
//1. 设置 websocket_handler
            server.add_websocket_handler ("/ws", endpoint_params.origin, null, on_websocket_opened);
//......此处省略
                SocketAddress? effective_address = null;
                InetSocketAddress? inet_address = address as InetSocketAddress;
                if (inet_address != null) {
                    uint16 start_port = inet_address.get_port ();
                    uint16 candidate_port = start_port;
                    do {
                        try {
//2. 监听地址
                            server.listen (inet_address, listen_options);

```

跟入 `on_websocket_opened` ：

```
private void on_websocket_opened (Soup.Server server, Soup.ServerMessage msg, string path,
        Soup.WebsocketConnection connection) {
    var peer = new WebConnection (connection);
 
    IOStream soup_stream = connection.get_io_stream ();
 
    SocketConnection socket_stream;
    soup_stream.get ("base-iostream", out socket_stream);
 
    SocketAddress remote_address;
    try {
        remote_address = socket_stream.get_remote_address ();
    } catch (GLib.Error e) {
        assert_not_reached ();
    }
 
    schedule_on_frida_thread (() => {
        incoming (peer, remote_address);
        return false;
    });
}

```

此处发送了 `incoming` 信号，而对应的处理被在构造函数中可见：

```
        construct {
            host_session.spawn_added.connect (notify_spawn_added);
            host_session.child_added.connect (notify_child_added);
            host_session.child_removed.connect (notify_child_removed);
            host_session.process_crashed.connect (notify_process_crashed);
            host_session.output.connect (notify_output);
            host_session.agent_session_detached.connect (on_agent_session_detached);
            host_session.uninjected.connect (notify_uninjected);
 
            service = new WebService (endpoint_params, CONTROL);
//1. 此处注册了 on_server_connection
            service.incoming.connect (on_server_connection);
 
            broker_service.incoming.connect (on_broker_service_connection);
        }

```

从 `on_server_connection` 跟入 `handle_server_connection` ：

```
        private async void handle_server_connection (IOStream raw_connection) throws GLib.Error {
//1. 创建 DBusConnection
            var connection = yield new DBusConnection (raw_connection, null, DELAY_MESSAGE_PROCESSING, null, io_cancellable);
            connection.on_closed.connect (on_connection_closed);
 
            Peer peer;
            AuthenticationService? auth_service = endpoint_params.auth_service;
            if (auth_service != null)
                peer = new AuthenticationChannel (this, connection, auth_service);
            else
//2. 创建 controlchannel
                peer = setup_control_channel (connection);
            peers[connection] = peer;
 
            connection.start_message_processing ();
        }

```

对于不需要认证的情况，将会调用 `setup_control_channel` 完成初始化，而该函数将会返回一个 `ControlChannel` 对象，其构造函数如下：

```
construct {
    try {
        HostSession session = this;
        registrations.add (connection.register_object (ObjectPath.HOST_SESSION, session));
 
        AuthenticationService null_auth = new NullAuthenticationService ();
        registrations.add (connection.register_object (Frida.ObjectPath.AUTHENTICATION_SERVICE, null_auth));
 
        TransportBroker broker = this;
        registrations.add (connection.register_object (Frida.ObjectPath.TRANSPORT_BROKER, broker));
    } catch (IOError e) {
        assert_not_reached ();
    }
}

```

该对象在构造时将会把 `HostSession` 、`AuthenticationService` 和 `TransportBroker` 都注册到 Dbus 对象中，这会使得远程的电脑端能够直接调用这些类中的方法从而实现通信。

而主要的负责通信的部分都由 `ControlChannel` 中的函数负责实现，常见的几个函数实现如下：

```
public async uint spawn (string program, HostSpawnOptions options, Cancellable? cancellable) throws GLib.Error {
    return yield parent.host_session.spawn (program, options, cancellable);
}
 
public async void resume (uint pid, Cancellable? cancellable) throws GLib.Error {
    yield parent.resume (pid, this);
}
 
public async AgentSessionId attach (uint pid, HashTable options,
        Cancellable? cancellable) throws GLib.Error {
    return yield parent.attach (pid, options, this, cancellable);
} 
```

可以看到，当我们使用 `spawn` 或者 `attach` 去附加或启动某个进程时，最终还是调用了 `host_session` 中的对应函数。

> 这里的 parent 指的是 `ControlService` 对象

而 `host_session` 其实是一个 `BaseDBusHostSession` 对象，该对象是平台相关的，不同平台又不同的实现方法，以 drawin 为例，我们跟一下 `spawn`：

```
        public override async uint spawn (string program, HostSpawnOptions options, Cancellable? cancellable)
                throws Error, IOError {
#if IOS || TVOS
            if (!program.has_prefix ("/"))
                return yield fruit_controller.spawn (program, options, cancellable);
#endif
 
            return yield helper.spawn (program, options, cancellable);
        }

```

可以看出，实际上它只是一个封装，会向下调用 frida-helper 中的 spawn 去实现。对于 darwin 平台，helper 是一个 `DarwinHelperBackend` ，顺带一提，`host_session` 其实是 `DarwinHostSession` 。

我们跟入实际的函数：

```
        public async uint spawn (string path, HostSpawnOptions options, Cancellable? cancellable) throws Error, IOError {
            if (!FileUtils.test (path, EXISTS))
                throw new Error.EXECUTABLE_NOT_FOUND ("Unable to find executable at '%s'", path);
 
            StdioPipes? pipes;
//1. 启动进程
            var child_pid = _spawn (path, options, out pipes);
 
            ChildWatch.add ((Pid) child_pid, on_child_dead);
 
            if (pipes != null) {
                stdin_streams[child_pid] = new UnixOutputStream (pipes.input, false);
                process_next_output_from.begin (new UnixInputStream (pipes.output, false), child_pid, 1, pipes);
                process_next_output_from.begin (new UnixInputStream (pipes.error, false), child_pid, 2, pipes);
            }
 
            return child_pid;
        }

```

到此我们就完成启动了，对于 spawn 模式启动的进程，将在启动后挂起等待附加，接下来我们跟一下 `attach` 。该函数也是平台相关的，最终的附加部分会由 `perform_attach_to` 实现：

```
protected override async Future perform_attach_to (uint pid, HashTable options,
        Cancellable? cancellable, out Object? transport) throws Error, IOError {
    transport = null;
 
    string remote_address;
    var stream_future = yield helper.open_pipe_stream (pid, cancellable, out remote_address);
 
    var id = yield inject_agent (pid, make_agent_parameters (pid, remote_address, options), cancellable);
    injectee_by_pid[pid] = id;
 
    return stream_future;
} 
```

此处可以看出，frida 调用 `inject_agent` 将 frida-agant 注入到进程中去。

frida-agant
-----------

frida-agant 在大多数时候充当了 app 内部的服务端。在用户向应用传递新脚本的时候，通过 RPC 服务与 frida-agant 通信，由它来负责注入 hook 。

在 `inject_agent` 中注入 agant 之后，`entrypoint` 为 `frida_agent_main` ，该函数也由 vala 转换而来，其实现定义在 `lib/agant/agant.vala` 中：

```
namespace Frida.Agent {
    public void main (string agent_parameters, ref Frida.UnloadPolicy unload_policy, void * injector_state) {
        if (Runner.shared_instance == null)
            Runner.create_and_run (agent_parameters, ref unload_policy, injector_state);
        else
            Runner.resume_after_transition (ref unload_policy, injector_state);
    }

```

从 `create_and_run` 一路往下跟：

```
private void run (owned FileDescriptorTablePadder padder) throws Error {
    main_context.push_thread_default ();
 
    start.begin ((owned) padder);
 
    main_loop.run ();
 
    main_context.pop_thread_default ();
 
    if (start_error != null)
        throw start_error;
}

```

此处开始之后会先调用 `start.run` 完成各项初始化任务，然后在 `main_loop.run()` 中进入循环，直到进程退出。

而此处 `start.begin()` 实际上执行的是如下函数：

```
        private async void start (owned FileDescriptorTablePadder padder) {
            string[] tokens = agent_parameters.split ("|");
            unowned string transport_uri = tokens[0];
            bool enable_exceptor = true;
#if DARWIN
            enable_exceptor = !Gum.Darwin.query_hardened ();
#endif
//此处省略
 
            {
                var interceptor = Gum.Interceptor.obtain ();
                interceptor.begin_transaction ();
//此处省略
 
            try {
                yield setup_connection_with_transport_uri (transport_uri);
            } catch (Error e) {
                start_error = e;
                main_loop.quit ();
                return;
            }
            Gum.ScriptBackend.get_scheduler ().push_job_on_js_thread (Priority.DEFAULT, () => {
                schedule_idle (start.callback);
            });
            yield;
 
            padder = null;
        }

```

可以看到，这里分别初始化了 `ScriptBackend` 和 `Interceptor` 。并连接到启动时指定的 `transport_uri` 建立通信隧道。

frida-helper
------------

其实这部分已经在前面介绍过了。frida-helper 的作用其实就是用于实现包括通信、注入进程、启动进程等各项功能等模块。这里就不再赘述了。

frida-gadget
============

源代码来自于 `lib/gadget/gadget.vala`，这部分也是笔者一直比较关心的部分，因为它允许我们在非越狱环境下使用 frida。

直接跟进主要函数：

```
    public void load (Gum.MemoryRange? mapped_range, string? config_data, int * result) {
        if (loaded)
            return;
        loaded = true;
 
        Environment.init ();
 
        Gee.Promise? request = null;
        if (result != null)
            request = new Gee.Promise ();
 
        location = detect_location (mapped_range);
//1. 解析或加载配置文件
        try {
            config = (config_data != null)
                ? parse_config (config_data)
                : load_config (location);
        } catch (Error e) {
            log_warning (e.message);
            return;
        }
 
        Gum.Process.set_code_signing_policy (config.code_signing);
 
        Gum.Cloak.add_range (location.range);
 
        interceptor = Gum.Interceptor.obtain ();
        interceptor.begin_transaction ();
        exceptor = Gum.Exceptor.obtain ();
//2. 设定 frida 的启动方式
        try {
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
                throw new Error.NOT_SUPPORTED ("Invalid interaction specified");
            }
        } catch (Error e) {
            resume ();
 
            if (request != null) {
                request.set_exception (e);
            } else {
                log_warning ("Failed to start: " + e.message);
            }
        }
//3. 启动 interceptor
        interceptor.end_transaction ();
 
        if (controller == null)
            return;
 
        wait_for_resume_needed = true;
//4. 确定是否需要直接恢复进程
        var listen_interaction = config.interaction as ListenInteraction;
        if (listen_interaction != null && listen_interaction.on_load == ListenInteraction.LoadBehavior.RESUME) {
            wait_for_resume_needed = false;
        }
 
        if (!wait_for_resume_needed)
            resume ();
//5. 完成初始化并加载脚本进入 main_loop
        if (wait_for_resume_needed && Environment.can_block_at_load_time ()) {
            var scheduler = Gum.ScriptBackend.get_scheduler ();
 
            scheduler.disable_background_thread ();
 
            wait_for_resume_context = scheduler.get_js_context ();
 
            var ignore_scope = new ThreadIgnoreScope (APPLICATION_THREAD);
 
            start (request);
 
            var loop = new MainLoop (wait_for_resume_context, true);
            wait_for_resume_loop = loop;
 
            wait_for_resume_context.push_thread_default ();
            loop.run ();
            wait_for_resume_context.pop_thread_default ();
 
            scheduler.enable_background_thread ();
 
            ignore_scope = null;
        } else {
            start (request);
        }
 
        if (result != null) {
            try {
                *result = request.future.wait ();
            } catch (Gee.FutureError e) {
                *result = -1;
            }
        }
    } 
```

首先 Frida-gadget 在被加载时会自动搜索配置文件，如果找到了则根据配置文件处理。

随后完成一系列工作以后，在进入主循环以前会调用 `start` 加载脚本：

```
private void start (Gee.Promise? request) {
    var source = new IdleSource ();
    source.set_callback (() => {
        perform_start.begin (request);
        return false;
    });
    source.attach (Environment.get_worker_context ());
} 
```

在 `perform_start` 中会调用 `controller.start ()` ，此时调用的函数将会根据先前用户配置文件中选择的类型完成。

比方说常用的 Listen 类型就会调用 `ControlServer` 下的 `on_start` ：

```
        protected override async void on_start () throws Error, IOError {
            var interaction = (ListenInteraction) config.interaction;
 
            string? token = interaction.token;
            auth_service = (token != null) ? new StaticAuthenticationService (token) : null;
 
            File? asset_root = null;
            string? asset_root_path = interaction.asset_root;
            if (asset_root_path != null)
                asset_root = File.new_for_path (location.resolve_asset_path (asset_root_path));
 
            var endpoint_params = new EndpointParameters (interaction.address, interaction.port,
                parse_certificate (interaction.certificate, location), interaction.origin, auth_service, asset_root);
// 1. 启动一个 WebService 与用户进行交互
            service = new WebService (endpoint_params, CONTROL, interaction.on_port_conflict);
            service.incoming.connect (on_incoming_connection);
            yield service.start (io_cancellable);
        }

```

然后该服务就会监听特定地址了，如果用户传递了文件或代码等，则会与 frida-agant 通过 IPC 服务通信，由对方去负责具体的 hook 行为。

又比如指定一个脚本令其自动运行：

```
public async void start () throws Error, IOError {
    save_terminal_config ();
 
    yield load ();
 
    if (enable_development && script_path != null) {
        try {
            script_monitor = File.new_for_path (script_path).monitor_file (FileMonitorFlags.NONE);
            script_monitor.changed.connect (on_script_file_changed);
        } catch (GLib.Error e) {
            printerr (e.message + "\n");
        }
    }
}

```

此处会调用 `load` 完成加载和启动的操作，我们跟入：

```
        private async void load () throws Error, IOError {
            load_in_progress = true;
 
            try {
                string source;
 
                var options = new ScriptOptions ();
//1. 读取脚本内容
                if (script_path != null) {
                    try {
                        FileUtils.get_contents (script_path, out source);
                    } catch (FileError e) {
                        throw new Error.INVALID_ARGUMENT ("%s", e.message);
                    }
//2. 读取脚本路径
                    options.name = Path.get_basename (script_path).split (".", 2)[0];
                } else {
                    source = script_source;
 
                    options.name = "frida";
                }
 
                options.runtime = script_runtime;
//3. 创建脚本
                var s = yield session.create_script (source, options, io_cancellable);
 
                if (script != null) {
                    yield script.unload (io_cancellable);
                    script = null;
                }
                script = s;
//4. 加载脚本到进程
                script.message.connect (on_message);
                yield script.load (io_cancellable);
//5. 启动目标进程
                yield call_init ();
 
                terminal_mode = yield query_terminal_mode ();
                apply_terminal_mode (terminal_mode);
 
                if (eternalize)
                    yield script.eternalize (io_cancellable);
            } finally {
                load_in_progress = false;
            }
        }

```

其中的 `call_init` 负责通过 rpc 服务去调用 `init` ：

```
private async void call_init () {
    var stage = new Json.Node.alloc ().init_string ("early");
 
    try {
        yield rpc_client.call ("init", new Json.Node[] { stage, parameters }, io_cancellable);
    } catch (GLib.Error e) {
    }
}

```

在完成以上操作后，frida 会进入 `main_loop`，而被启动的应用会等待被 resume。

然后是 `perform_start` 的后半：

```
    private async void perform_start (Gee.Promise? request) {
        worker_ignore_scope = new ThreadIgnoreScope (FRIDA_THREAD);
 
        try {
            yield controller.start ();
 
            var server = controller as ControlServer;
            if (server != null) {
//1. 如果 controller 是一个服务端的话，那么就需要监听网络地址来和用户进行交互
//比如常用的 Listen 模式就需要监听特定地址端口
                var listen_address = server.listen_address;
                var inet_address = listen_address as InetSocketAddress;
                if (inet_address != null) {
                    uint16 listen_port = inet_address.get_port ();
                    Environment.set_thread_name ("frida-gadget-tcp-%u".printf (listen_port));
                    if (request != null) {
                        request.set_value (listen_port);
                    } else {
                        log_info ("Listening on %s TCP port %u".printf (
                            inet_address.get_address ().to_string (),
                            listen_port));
                    }
                } else {
#if !WINDOWS
//2. 对于不是 windows 的系统，这里使用的是 frida-gadget-unix，这里监听 unix socket
//主要是负责 IPC 服务用的
                    var unix_address = (UnixSocketAddress) listen_address;
                    Environment.set_thread_name ("frida-gadget-unix");
                    if (request != null) {
                        request.set_value (0);
                    } else {
                        log_info ("Listening on UNIX socket at “%s”".printf (unix_address.get_path ()));
                    }
#else
                    assert_not_reached ();
#endif
                }
            } else {
                if (request != null)
                    request.set_value (0);
            }
        } catch (GLib.Error e) {
            resume ();
 
            if (request != null) {
                request.set_exception (e);
            } else {
                log_warning ("Failed to start: " + e.message);
            }
        }
    } 
```

由于该函数是以回调函数的形式被注册的，因此附加的进程在每次触发请求的时候都会重新调用该函数处理。

然后我们再看看使用这种方式的情况下是如何附加进程和启动进程的。

在前文中曾说过，通过 IPC 的方式，主机端能够直接调用类中的方法，其中一个比较关键的类是 `ControlChannel` ，它负责了几个关键行为的设定。

```
public async uint spawn (string program, HostSpawnOptions options, Cancellable? cancellable) throws Error, IOError {
    if (program != this_app.identifier)
        throw new Error.NOT_SUPPORTED ("Unable to spawn other apps when embedded");
 
    resume_on_attach = false;
 
    return this_process.pid;
}

```

对于 `spawn` 方式的启动，由于所有模块都被打包在同一个进程中，因此当前进程就是将要附加的进程，因此 `spawn` 可以直接返回当前进程的 pid。

attach 倒是没太大变化：

```
public async AgentSessionId attach (uint pid, HashTable options,
        Cancellable? cancellable) throws Error, IOError {
    validate_pid (pid);
 
    if (resume_on_attach)
        Frida.Gadget.resume ();
 
    return yield parent.attach (options, this, cancellable);
} 
```

它仍然调用 `parent.attach` 去附加。

总结一下就是：

*   frida-gadget 被注入以后会在加载该库的时候调用 `load` 方法
*   该方法根据用户提供的配置文件选择接下来的行为
*   对于 Listen 则是监听地址端口并触发回调完成交互
*   如果用户在配置中指定来 resume，那么此前会先调用 `resume` 恢复进程
*   对于 Script 则是读取给定路径下的脚本解析并加载
*   如果需要监听文件变化时候动态修改 hook ，那么还需要额外操作

launchd
=======

上文大致介绍完了 frida 的大体逻辑，但是还有一个细节上的小问题没有解决。具体来说就是，“frida 到底是怎么通过 spawn 启动的进程？”

实际上，frida 除了对进程本身进行注入以外，还会对 launchd 进行注入：

```
Interceptor.attach(Module.getExportByName('/usr/lib/system/libsystem_kernel.dylib', '__posix_spawn'), {
  onEnter(args) {
    const env = parseStringv(args[4]);
    const prewarm = isPrewarmLaunch(env);
 
    if (prewarm && !gating)
      return;
 
    const path = args[1].readUtf8String();
 
    let rawIdentifier;
    if (path === '/usr/libexec/xpcproxy') {
      rawIdentifier = args[3].add(pointerSize).readPointer().readUtf8String();
    } else {
      rawIdentifier = tryParseXpcServiceName(env);
      if (rawIdentifier === null)
        return;
    }
 
    let identifier, event;
    if (rawIdentifier.startsWith('UIKitApplication:')) {
      identifier = rawIdentifier.substring(17, rawIdentifier.indexOf('['));
      if (!prewarm && upcoming.has(identifier))
        event = 'launch:app';
      else if (gating)
        event = 'spawn';
      else
        return;
    } else if (gating || (reportCrashes && crashServices.has(rawIdentifier))) {
      identifier = rawIdentifier;
      event = 'spawn';
    } else {
      return;
    }
 
    const attrs = args[2].add(pointerSize).readPointer();
 
    let flags = attrs.readU16();
    flags |= POSIX_SPAWN_START_SUSPENDED;
    attrs.writeU16(flags);
 
    this.event = event;
    this.path = path;
    this.identifier = identifier;
    this.pidPtr = args[0];
  },
  onLeave(retval) {
    const { event } = this;
    if (event === undefined)
      return;
 
    const { path, identifier, pidPtr, threadId } = this;
 
    if (event === 'launch:app')
      upcoming.delete(identifier);
 
    if (retval.toInt32() < 0)
      return;
 
    const pid = pidPtr.readU32();
 
    suspendedPids.add(pid);
 
    if (pidsToIgnore !== null)
      pidsToIgnore.add(pid);
 
    if (substrateInvocations.has(threadId)) {
      substratePidsPending.set(pid, notifyFridaBackend);
    } else {
      notifyFridaBackend();
    }
 
    function notifyFridaBackend() {
      send([event, path, identifier, pid]);
    }
  }
});

```

这个地方直接把 `__posix_spawn` 给 hook 掉了，并且加上了 `POSIX_SPAWN_START_SUSPENDED` 的 flag，该标记能够让进程在启动后被挂起。

总结
==

各个组件的功能如下：

*   frida-server / 一个服务端。负责在设备上与本机通讯
*   frida-agant / 被注入到进程中去的动态库，通常由 frida-server 释放注入，负责编译脚本注入进程
*   frida-helper / 负责具体的进程注入、启动进程等功能
*   frida-gadgat / frida-server+frida-agant+frida-helper ，将三者的功能全都集成在一个动态库中，由用户手动注入到应用中

这里引出一个小问题，对于被注入 Frida-gadget 的 app 来说，如果我不使用 frida 去启动它，而是通过点击图标的方式原生启动应用，那么应用还能正常启动吗？

如果仅凭上文的分析，主机端通过 IPC 通信去调用设备上对应的函数从而启动了应用，但是原生启动是不通过 IPC 的，这种情况下，frida-gadget 要如何工作呢？它还会正常去启动应用吗？

问了一些师傅，他们表示 Android 平台下，即便注入的 frida-gadget 也是可以正常点击打开的，但是笔者在 iOS16 上测试发现这将导致闪退，但是诡异的是，我能够用 `frida -U -f bundleid` 正常打开应用。  
而在 iOS14 上，笔者发现应用将会停在启动页面无法继续执行，并且 frida 也没办法附加，以及 `frida -U -f bundleid` 也无法正常启动了，唯独 Xcode 启动时，一切正常，这十分的诡异。

以上问题目前笔者还不清楚原因，欢迎师傅们讨论。

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

[#调试逆向](forum-4-1-1.htm) [#其他内容](forum-4-1-10.htm)