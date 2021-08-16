> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jmpews.github.io](https://jmpews.github.io/2017/08/09/darwin/%E5%8F%8D%E6%B3%A8%E5%85%A5%E5%8F%8A%E7%BB%95%E8%BF%87/)

> 任何带特征的检测都是不安全的 & 隐而不发 之前说了下关于反调试的相关, 这次说下反注入相关. 前言要实现 hook 的首要前提就是能够注入. 但是注入的手段比较多. 1. DYLD_INSE......

```
// frida-core/src/darwin/frida-helper-backend-glue.m

void

_frida_darwin_helper_backend_prepare_spawn_instance_for_injection (FridaDarwinHelperBackend * self, void * opaque_instance, guint task, GError ** error)

{

  FridaSpawnInstance * instance = opaque_instance;

  FridaHelperContext * ctx = self->context;

  const gchar * failed_operation;

  kern_return_t ret;

  mach_port_t self_task, child_thread;

  guint page_size;

  thread_act_array_t threads;

  guint thread_index;

  mach_msg_type_number_t thread_count = 0;

  GumDarwinUnifiedThreadState state;

  mach_msg_type_number_t state_count = GUM_DARWIN_THREAD_STATE_COUNT;

  thread_state_flavor_t state_flavor = GUM_DARWIN_THREAD_STATE_FLAVOR;

  GumAddress dyld_start, dyld_granularity, dyld_chunk, dyld_header;

  GumAddress probe_address, dlerror_clear_address;

  GumDarwinModule * dyld;

  FridaExceptionPortSet * previous_ports;

  dispatch_source_t source;

  /*

   * We POSIX_SPAWN_START_SUSPENDED which means that the kernel will create

   * the task and its main thread, with the main thread's instruction pointer

   * pointed at __dyld_start. At this point neither dyld nor libc have been

   * initialized, so we won't be able to inject frida-agent at this point.

   *

   * So here's what we'll do before we consider spawn() done:

   * - Get hold of the main thread to read its instruction pointer, which will

   *   tell us where dyld is in memory.

   * - Walk backwards to find dyld's Mach-O header.

   * - Walk its symbols and find a function that's called at a point where the

   *   process is sufficiently initialized to load frida-agent, but early enough

   *   so that app's initializer still didn't run. In this case we choose

   *   dyld::initializeMainExecutable(). At the beginning of this function dyld is

   *   initialized but libSystem is still missing.

   * - Set a hardware breakpoint at the beginning of this function.

   * - Swap out the thread's exception ports with our own.

   * - Resume the task.

   * - Wait until we get a message on our exception port, meaning our breakpoint

   *   was hit.

   * - Hijack thread's instruction pointer to call dlopen("/usr/lib/libSystem.B.dylib")

   *   and then return back to the beginning of initializeMainExecutable() and restore

   *   previous thread state.

   * - Swap back the thread's orginal exception ports.

   * - Clear the hardware breakpoint by restoring the thread's debug registers.

   *

   * It's actually more complex than that, because:

   * - This doesn't work on newer versions of dyld because to call dlopen() it's

   *   necessary to registerThreadHelpers() first, which is normally done by libSystem

   *   itself during its initialization.

   * - To overcome this catch-22 we alloc a fake LibSystemHelpers object and register

   *   it (also by hijacking thread's instruction pointer as described above).

   * - On older dyld versions, registering helpers before loading libSystem led to

   *   crashes, so we detect this condition and unset the helpers before calling dlopen(),

   *   by writing a NULL directly into the global dyld::gLibSystemHelpers because in

   *   some dyld versions calling registerThreadHelpers(NULL) causes a NULL dereference.

   * - At the end of dlopen(), we set the global "libSystemInitialized" flag present in

   *   the global dyld::qProcessInfo structure, because on newer dyld versions that doesn't

   *   happen automatically due to the presence of our fake helpers.

   * - One of the functions provided by the helper should return a buffer for the errors,

   *   but since our fake helpers object implements its functions only using a return,

   *   it will not return any buffer. To avoid this to happen, we set a breakpoint also

   *   on dyld:dlerrorClear function and inject an immediate return,

   *   effectively disabling the function.

   * - At the end of dlopen() we finally deallocate our fake helpers (because now they've

   *   been replaced by real libSystem ones) and the string we used as a parameter for dlopen.

   *

   * Then later when resume() is called:

   * - Send a response to the message we got on our exception port, so the

   *   kernel considers it handled and resumes the main thread for us.

   */

```