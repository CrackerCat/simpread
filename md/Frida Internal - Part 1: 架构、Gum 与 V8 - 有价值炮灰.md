> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [evilpan.com](https://evilpan.com/2022/04/05/frida-internal/)

> frida 是一个非常优秀的开源项目，因为项目活跃，代码整洁，接口清晰，加上用灵活的脚本语言 (JS) 来实现指令级代码追踪的能力，为广大的安全研究人员所喜爱。

frida 是一个非常优秀的开源项目，因为项目活跃，代码整洁，接口清晰，加上用灵活的脚本语言 (JS) 来实现指令级代码追踪的能力，为广大的安全研究人员所喜爱。虽然使用人群广泛，但对其内部实现的介绍却相对较少，因此笔者就越俎代庖，替作者写写 frida 内部实现介绍，同时也作为自己的阅读理解记录。

系列文章传送门:

*   [Frida Internal - Part 1: 架构、Gum 与 V8 (本文)](https://evilpan.com/2022/04/05/frida-internal/)
*   [Frida Internal - Part 2: frida-core](https://evilpan.com/2022/04/09/frida-core/)
*   [Frida Internal - Part 3: frida-java-bridge 与 ART hook](https://evilpan.com/2022/04/17/frida-java/)

在 [frida](https://github.com/frida/frida) 的主仓库中，我们一般是直接在其 release 页面下载 CI 的成品使用，其中可以看到有许多可供下载的组件，比如:

*   frida-server
*   frida-gadget
*   frida-inject
*   frida-core/gum/gumjs-devkit
*   frida-clr
*   frida-portal
*   frida-tools
*   …

按照封装层级来划分可以分为 4 级，分别是:

1.  CPU 指令集级别的 inline-hook 框架: frida-gum；
2.  使用 JavaScript 引擎对 gum 进行封装实现脚本拓展的能力: gum-js；
3.  运行时进程注入、脚本加载、RPC 通信管理等功能: frida-core；
4.  针对特殊运行环境的 js 模块及其接口，如 frida-java-bridge、frida-objc-bridge 等；

还有一些其他辅助组件如 clr、portal 和针对不同语言的 binding 等功能相对花哨，就不额外进行讨论了。

由于笔者的关注点主要在系统层的实现上，因此对于 frida-gum 和 gum-js 的实现细节不会太过深入，就让二老委屈点挤在本文中一并进行介绍了。后续针对 frida-core 和 frida-java-bridge 都会有单独的文章进行分析。

说起 [frida-gum](https://github.com/frida/frida-gum) 大家都知道它提供了 inline-hook 的核心实现，但实际上它还包含了许多其他的模块，比如用于代码跟踪 Stalker、用于内存访问监控的 MemoryAccessMonitor，以及符号查找、栈回溯实现、内存扫描、动态代码生成和重定位等功能，可以说是上层丰富功能的关键基础设施。

[Interceptor](https://github.com/frida/frida-gum/blob/main/gum/guminterceptor.h) 是 inline-hook 的封装，从接口上看，Interceptor 的使用方法大致如下:

```
GumInterceptor * interceptor;
GumInvocationListener * listener;
gum_init ();
interceptor = gum_interceptor_obtain ();
listener = g_object_new (EXAMPLE_TYPE_LISTENER, NULL);

// 开始 hook `open` 函数
gum_interceptor_begin_transaction (interceptor);
gum_interceptor_attach_listener (interceptor,
      GSIZE_TO_POINTER (gum_module_find_export_by_name (NULL, "open")),
      listener,
      GSIZE_TO_POINTER (EXAMPLE_HOOK_OPEN));
gum_interceptor_end_transaction (interceptor);

// 测试 hook 效果
close (open ("/etc/hosts", O_RDONLY));

// 结束 hook
gum_interceptor_detach_listener (interceptor, listener);
g_object_unref (listener);
g_object_unref (interceptor);
```

其中 `listener` 是一个 `GumInvocationListener *` 类型的接口，用户可以自行实现 `on_enter` 和 `on_leave` 来控制注入的逻辑:

```
typedef void (* GumInvocationCallback) (GumInvocationContext * context, gpointer user_data);

struct _GumInvocationListenerInterface {
  GTypeInterface parent;

  void (* on_enter) (GumInvocationListener * self,
      GumInvocationContext * context);
  void (* on_leave) (GumInvocationListener * self,
      GumInvocationContext * context);
};
```

一般直接从 release 中下载 `frida-gum-devkit` 去静态编译使用，内部包含了一个静态库、头文件和示例程序，并且头文件有丰富的注释说明:

```
$ tar -xvf frida-gum-devkit-15.1.17-android-arm.tar.xz
x frida-gum.h
x libfrida-gum.a
x frida-gum-example.c
```

frdia-gum 使用 C 语言来编写，但主要依赖于 [glib](https://wiki.gnome.org/Projects/GLib) 去实现面向对象编程，这是由 GNOME 开发并从 GTK 中分离出来的一个基础库。使用纯 C 编程的话这是个非常好用的工具库，其提供了许多有用的数据类型、宏定义、类型转换、字符串 / 文件处理以及线程的抽象支持。在 Linux 内核中也能看到很多 glib 封装设计的思想在，因此若是有 C 开发需求比如嵌入式场景，也可以考虑使用 glib 去进行辅助。

Inline hook 的原理这里就不展开了，但是值得一提的是在构造目标函数的跳板函数时需要根据用户指定的地址去动态生成跳板代码，因此使用了 GumWriter 来实现；同时因为要备份原函数还需要使用 GumRelocator 去动态修复 (重定位) 地址相关的代码。frida-gum 实现了 5 种指令集的代码生成和重定向功能(基于 capstone)，分别是 arm、arm64、x86、x64 和 mips。以 arm64 为例，其实现在 _gum/backend-arm64/guminterceptor-arm64.c_ 的 `_gum_interceptor_backend_create_trampoline` 中。

拓展阅读:

*   [frida-example](https://gist.github.com/oleavr/3edc47c9f69eb048de9d70ed45998f9c)
*   [Profiling C++ code with Frida](https://lief-project.github.io/blog/2021-03-10-profiling-cpp-code-with-frida/)
*   [FRIDA-GUM 源码解读](http://jmpews.github.io/2017/06/27/pwn/frida-gum%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/)

[Stalker](https://frida.re/docs/stalker) 又称为尾行痴汉 (?)，可以实现指定线程中所有函数、所有基本块、甚至所有指令的跟踪。

一般而言调试器实现函数或者指令跟踪是通过断点，但是断点有几个问题。一是断点容易被反调试检测到，软件断点自不必说，会在原指令中插入断点指令，如果函数本身有完整性校验的话会检测出异常，而硬件断点本身也很容易被检测或者破坏掉；断点的另一个问题是性能，代码触发断点后会先中断到内核态，然后再返回到用户态 (调试器) 执行跟踪回调，处理完后再返回内核态，然后再回到用户态继续执行，这来来回回的黄花菜都凉了。

因此，Stalker 不是使用断点，而是基于动态修改代码的方式实现。其基本思想很简单，在线程即将执行下一条指令前，先将目标指令拷贝一份到新建的内存中，然后在新的内存中对代码进行插桩，如下图所示:

[![](https://t00img.yangkeduo.com/chat/images/2024-07-10/db7722d58b0b149e3cf0b53b0e4431b0.png)](https://t00img.yangkeduo.com/chat/images/2024-07-10/db7722d58b0b149e3cf0b53b0e4431b0.png "img")Ole André: Anatomy of a code tracer

这其中使用到了代码**动态重编译**的方法，好处是原本的代码没有被修改，因此即便代码有完整性校验也不影响，另外由于执行过程都在用户态，省去了多次中断内核切换，性能损耗也达到了可以接受的水平。由于代码的位置发生了改变，如前文 Interceptor 一样，同样要对代码进行重定位的修复。

Stalker 中可以以任意单位基本进行跟踪，越细粒度的跟踪性能损耗越高。比如在流量分析时可以只针对 SSL_read、SSL_write 进行跟踪获取流量明文；又比如在 Fuzzing 中经常以基本块为单位进行覆盖率反馈，此时可以以块为单位进行跟踪，只对 blx、br、jmp、call 等跳转指令进行插桩；或者针对某些自定义的加密算法，可以以指令级别进行跟踪，通过动态运行的路径记录来辅助逆向分析还原加密流程，等等等等。

此外 Stalker 中还针对动态编译进行了大量的性能优化，进一步减少运行时的额外开销。由于动态重编译与系统架构关系较大，代码中需要对当前平台的指令集进行准确的归类和处理，因此当前 Stalker 只支持常用的 ARM64、X86 和 IA32 架构，而且对于动态自修改的代码支持也不完善，但即便如此也足以满足大部分日常的需求了。

拓展阅读:

*   [Anatomy of a code tracer](https://medium.com/@oleavr/anatomy-of-a-code-tracer-b081aadb0df8)
*   [frida docs - stalker](https://frida.re/docs/stalker)

frida-gum 中另外一个很有意思的模块是 `MemoryAccessMonitor`，可以实现对指定内存区间的访问监控，在目标内存区间发生读写行为时可以触发用户指定的回调函数。

通过阅读源码发现这个功能的实现方法非常简洁，本质上是将目标内存页设置为不可读写，这样在发生读写行为时会触发事先注册好的中断处理函数，其中会调用到用户使用 `gum_memory_access_monitor_new` 注册的回调方法中。

```
gboolean
gum_memory_access_monitor_enable (GumMemoryAccessMonitor * self,
                                  GError ** error)
{
  if (self->enabled)
    return TRUE;
  // ...
  self->exceptor = gum_exceptor_obtain ();
  gum_exceptor_add (self->exceptor, gum_memory_access_monitor_on_exception,
      self);
  // ...
}
```

事实上目前市面上也有一些加壳的应用使用这种方法来进行加固，在 `art::DexFile` 的某些地址区间中加上内存监控的功能，一旦发现读取行为就崩溃退出以实现代码保护的目的。至于效果嘛，就见仁见智了。……

作为一个 inline-hook 框架，自然还需要有一定的内省能力，比如搜索当前虚拟内存中已加载的动态库信息，在动态库中查找符号地址，在内存中搜索数据 / 代码等功能，这些 frida-gum 中都有实现，并且支持 Linux、Darwin、FreeBSD、QNX、Windows 等操作系统环境。在不同平台中往往有不同的实现方法，比如搜索符号在 Android 中就是通过 linker 的一些内部函数去实现模块查找并通过解析 ELF 的方式去定位符号。对其中哪些平台的实现感兴趣的，去查看对应 backend 的代码即可。

`frida-gum` 虽然功能强大，但由于使用了 C 语言的接口，调用起来各种宏和 typedef 颇为不便，因此就有了以此为基础的上层封装 (binding)。目前仓库中只有两种语言的封装，分别是 C++(gumpp) 和 JavaScript(gumjs)，后者也是 frida-core 和大多数人日常使用的接口底座。

大多数人一开始接触 frida 应该也和笔者一样很奇怪为什么 frida 使用 JavaScript 作为编写 hook 的语言，为什么还特地集成了一个 JS 脚本引擎，具体是如何实现的，……

其实这个问题可以简化成: JS 代码如何与 native 代码进行交互？我们以 v8 为例，先看如何从 native 代码中去解析 JS 脚本。一般来说，v8 解析脚本的流程可以概况如下:

```
int main(int argc, char* argv[]) {
  // 1. 初始化 V8.
  v8::V8::InitializeICUDefaultLocation(argv[0]);
  v8::V8::InitializeExternalStartupData(argv[0]);
  std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();
  
  // 2. 新建 Isolate 并将其设为当前默认.
  v8::Isolate::CreateParams create_params;
  create_params.array_buffer_allocator = v8::ArrayBuffer::Allocator::NewDefaultAllocator();
  v8::Isolate* isolate = v8::Isolate::New(create_params);

  {
    v8::Isolate::Scope isolate_scope(isolate);
    v8::HandleScope handle_scope(isolate);
    // 3. 创建上下文
    v8::Local<v8::Context> context = v8::Context::New(isolate);
    // 4. 进入 JS 脚本编译和执行的上下文
    v8::Context::Scope context_scope(context);
    // 5. 创建脚本
    v8::Local<v8::String> source =
        v8::String::NewFromUtf8(isolate, "'Hello' + ', World!'",
                                v8::NewStringType::kNormal)
            .ToLocalChecked();
    // 6. 编译脚本
    v8::Local<v8::Script> script =
        v8::Script::Compile(context, source).ToLocalChecked();
    // 7. 执行脚本并获取结果
    v8::Local<v8::Value> result = script->Run(context).ToLocalChecked();
    // 8. 将结果转换为字符串并打印输出
    v8::String::Utf8Value utf8(isolate, result);
    printf("%s\n", *utf8);
  }

  // n. 关闭 V8，释放相关资源
  isolate->Dispose();
  v8::V8::Dispose();
  v8::V8::ShutdownPlatform();
  delete create_params.array_buffer_allocator;
```

如果我们不仅仅是从 JS 中获取执行结果，而是需要向 JS 动态传递参数呢？比如在 frida 中 `Interceptor.attach` 的参数之一实际上就是目标函数 (指令) 的 native 地址值，我们需要在 JS 中将这个值进行处理并传递到 `frida-gum` 的 `gum_interceptor_attach_listener` 函数中。

这种情况下需要先在 JS 脚本中定义一个函数，姑且称之为 `Attach`，前面的步骤都一样，先创建脚本并编译执行，执行之后可以从当前的 Isolate 中获取到目标函数对象，进而转换为可以调用的 Function 类型从而进行调用，如下所示:

```
// 1-7 步类似，执行脚本
// 定义函数名称
Local<String> attach_name =
      String::NewFromUtf8Literal(GetIsolate(), "Attach");
// 判断对象是否存在，以及类型是否是函数
Local<Value> attach_val;
if (!context->Global()->Get(context, attach_name).ToLocal(&attach_val) || !attach_val->IsFunction()) {
    return false;
}
// 如果是，则转换为函数类型
Local<Function> attach_func = attach_val.As<Function>();

// 将调用参数封装为 JS 对象
Local<Object> obj =
      templ->NewInstance(GetIsolate()->GetCurrentContext()).ToLocalChecked();
obj->SetInternalField(0, 0xdeadbeef);
// 使用自定义的参数调用该 JS 函数，并获取返回结果
TryCatch try_catch(GetIsolate());
const int argc = 1;
Local<Value> argv[argc] = {obj};
Local<Value> result;
attach_func->Call(context, context->Global(), argc, argv).ToLocal(&result);
```

有了这个理论基础，后面的实现就是工程化编码的问题了。早期 gum-js 默认使用 [Duktape](https://duktape.org/) 作为脚本引擎进行集成，后来也增加了对 [QuickJS](https://bellard.org/quickjs/) 和 [V8](https://v8.dev/docs/) 的支持，实际上 frida 对于不同的脚本引擎也做了一层封装，可以对不同引擎的接口实现透明的切换。

> gum-js 同样提供了类似的 devkit，里面包含对应的接口介绍和示例程序，可自行下载食用。

软件有 Bug 其实是在所难免的事情，尤其是对于 frdia 这种偏底层而又复杂的项目而言，因此在出现未知错误时需要能够通过调试器或者日志去进行定位。另外从学习的角度来说，也希望可以通过实时调试去加深对于 frida 中一些操作比如 trampoline/shellcode 生成的理解。

首先是调试。在 frida 主仓库的 `config.mk` 中去掉 `--strip` 以保留符号，或者加上 make 参数 `FRIDA_ASAN=true` 加上 ASAN 信息辅助定位内存问题。在 hook 代码中可以加入以下代码来等待调试器的挂载:

```
while (!gum_process_is_debugger_attached ())
{
  g_printerr ("Waiting for debugger in PID %u...\n", getpid ());
  g_usleep (G_USEC_PER_SEC);
}
```

或者直接在 JS 代码中实现:

```
while (!Process.isDebuggerAttached()) {
  console.log('Waiting for debugger in PID:', Process.id);
  Thread.sleep(1);
}
```

除了使用调试器进行调试，也可以使用 print 大法在代码的关键位置插入打印日志来帮助理解代码的执行流程，常用的打印函数是:

```
// 输出到系统日志中
g_info ("Test %d\n", __line__);

// 输出到 stderr 
g_printerr ("Test %d\n", __line__);
```

作者还专门写过一个 gist 来介绍[使用日志功能来辅助分析的 Tricks](https://gist.github.com/oleavr/00d71868d88d597ee322a5392db17af6)，主要是通过 patch 实现向 `/data/local/tmp` 中写入指定的日志文件。

本文介绍了 frida 的主要工程结构以及 frida-gum 和 gumjs 这两大基础设施的基本原理。当然文章所提及的也只是冰山一角，感兴趣的人自然会去阅读对应模块的具体实现。希望这个系列能起到个抛砖引玉的作用，让后来者不至于搜索 frida internal 或者 frida 源码解析时再经历一遍鄙人的痛苦 :)

*   [frida.re/Hacking](https://frida.re/docs/hacking/)
*   [Ole André Vadla Ravnås - Frida: The engineering behind the reverse-engineering (Video)](https://www.youtube.com/watch?v=uc1mbN9EJKQ&ab_channel=OSDCNordic)

> **版权声明**: 自由转载 - 非商用 - 非衍生 - 保持署名 ([CC 4.0 BY-SA](https://creativecommons.org/licenses/by-nc/4.0/))  
> **原文地址**: [https://evilpan.com/2022/04/05/frida-internal/](https://evilpan.com/2022/04/05/frida-internal/)  
> **微信订阅**: **『[有价值炮灰](http://t.evilpan.com/qrcode.jpg)』**  
> – _TO BE CONTINUED_.