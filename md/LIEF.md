> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [lief.quarkslab.com](https://lief.quarkslab.com/blog/2021-03-10-profiling-cpp-code-with-frida/)

Frida is a well-known reverse engineering framework that enables (along with other functionalities) to hook functions on closed-source binaries. While hooking is generally used to get dynamic information about functions for which we don’t have the source code, this blog post introduces another use case to profile C/C++ code.

### Code Profiling

LIEF starts to be quite mature but there are still some concerns regarding:

1.  The speed (especially when rebuilding large ELF binaries)
2.  The memory consumption
3.  Compilation time

These limitations are “quite” acceptable on modern computers but when we target embedded systems like iPhone or Android devices, it starts to reach the limits. Since (spoiler) I started to implement a parser for the Dyld shared cache and for parsing in-memory Mach-O files, I faced some of these issues.

To address these problems, we must identify where are the bottleneck and ideally, without modifying too much the source code. To profile memory consumption, `valgrind --tool=massif` does the job pretty well out of the box: we don’t need to pass extra compilation flags nor modifying the source code. Regarding the code execution, we can profile it with:

1.  Valgrind (or QBDI?)
2.  Inserting log functions in the source code
3.  Using compiler instrumentation: `-finstrument-functions`

In the context of profiling LIEF, I’m mostly interested in profiling the code at the functions level: “How long does a function take to be executed?”

Valgrind provides CPU cycles that are somehow correlated to the execution time but it requires an extra processing step to identify the function’s overhead. Moreover, since Valgrind instruments the code, it can take time to profile a large codebase.

On the other hand, inserting log messages in the code is the easiest way to get the execution time of functions. I was not completely convinced with this solution since it adds log messages that are not always needed.

Finally, Clang and GCC enable to instrument the source code through the `-finstrument-functions` compilation flag. This flag basically inserts the `__cyg_profile_func_enter` and `__cyg_profile_func_exit` functions at the beginning and at the end of the original functions.

Frida works on compiled code and provides a mechanism (hook) to insert a callback before a given function and after the execution of the function. It is very similar to the `-finstrument-functions`, except that it is done post-compilation.

To setup a hook, we only have to provide a pointer to the function that aims at being hooked. In the context of profiling execution time, the callback at the beginning of the function can initialize a `std::chrono` object and the callback at the end of the function can print the time spent since the initialization of the `std::chrono`.

Let’s take a simple example to explain what Frida does. If we have the following function:

```
1void heavy_function() {
2  for (size_t i = 0; i < 1000000; ++i) {
3    // Code that takes time ...
4  }
5}


```

Frida enables (from a logical point of view) to have:

```
1void heavy_function() {
2  frida_on_enter();
3
4  for (size_t i = 0; i < 1000000; ++i) {
5    // Code that takes time ...
6  }
7
8  frida_on_leave();
9}


```

… without tweaking the compilation flags :)

Frida Bootstrap
---------------

Most of the documentation and the blog posts that we can find on the internet about Frida are based on the JavaScript API but Frida also provides in the first place the frida-gum SDK [1](#fn:1) that exposes a C API over the hook engine. This SDK comes with the `frida-gum-example.c` file that shows how to setup the hook engine.

Regarding the API of our profiler, we would like to have :

```
1#include <LIEF/ELF.hpp>
2
3// Functions to profile
4profile(&LIEF::ELF::Parser::parse_symbol_version);
5profile(&LIEF::ELF::Parser::parse_segments<LIEF::ELF::ELF64>);
6
7LIEF::ELF::Parser::parse("./sample.bin");


```

And an output like:

```
1$ ./run
2LIEF::ELF::Parser::parse_symbol_version() took 39ms
3LIEF::ELF::Parser::parse_segments() took 109ms


```

I won’t go through all the details of the implementation of the profiler since the source code is on [Github](https://github.com/lief-project/frida-profiler) but the next section covers some tricky parts.

Firstly, and as mentioned previous section, Frida takes a **void* pointer** on the function to hook. Therefore, we have to cast `&LIEF::ELF::Parser::parse_symbol_version` into a `void*`. One might want to do `reinterpret_cast<void*>()` on the function pointer but it does not work. The trick here is to use a union to get the `void*`:

```
1template<typename Func>
2inline void* cast_func(Func f) {
3  union {
4    Func func;
5    void* p;
6  };
7  func = f;
8  return p;
9}


```

Secondly, the example `frida-gum-example.c` uses an enum to identify the function being hooked:

```
 1typedef enum _ExampleHookId ExampleHookId;
 2enum _ExampleHookId
 3{
 4  EXAMPLE_HOOK_OPEN,
 5  EXAMPLE_HOOK_CLOSE
 6};
 7...
 8gum_interceptor_attach (interceptor,
 9    GSIZE_TO_POINTER (gum_module_find_export_by_name (NULL, "open")),
10    listener,
11    GSIZE_TO_POINTER (EXAMPLE_HOOK_OPEN));
12


```

In our case, we don’t know beforehand which functions will be hooked or profiled by the user. Consequently, instead of using an enum we use the function’s absolute address and we register its name in a map:

```
 1template<typename Func>
 2void profile_func(Func func, std::string name) {
 3  void* addr = cast_func(func);
 4  funcs[reinterpret_cast<uintptr_t>(addr)] = std::move(name);
 5  gum_interceptor_begin_transaction(ctx_->interceptor);
 6  gum_interceptor_attach(ctx_->interceptor,
 7      /* Target */ reinterpret_cast<gpointer>(addr),
 8      /* Param  */ reinterpret_cast<GumInvocationListener*>(ctx_),
 9      /* id     */ reinterpret_cast<gpointer>(addr));
10  gum_interceptor_end_transaction(ctx_->interceptor);
11}


```

Last but not least, we might want to profile private or protected functions. To enable the access to the Profiler to protected/private members we can _friend_ an opaque Profile structure:

```
1struct Profiler;
2
3namespace LIEF {
4class LIEF_API Parser : public LIEF::Parser {
5  public:
6  friend struct ::Profiler;
7  ...
8};
9}


```

Conclusion
----------

Through this blog post, we have shown that Frida also has some applications in the field of software engineering not only for reverse-engineering :)

This approach can be quite convenient to isolate the profiling process from the compilation process. It also enables to quickly switch from a given SDK version to another as long as the profiled functions still exist.

```
1$ clang++ [-other-flags] LIEF-0.9.0/lib/libLIEF.a profile.cpp
2$ clang++ [-other-flags] LIEF-0.12.0/lib/libLIEF.a profile.cpp


```

The source code used in this blog post is available on Github: [lief-project/frida-profiler](https://github.com/lief-project/frida-profiler)