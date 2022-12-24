> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [robot9.me](https://robot9.me/android-tls-hook/)

> Robot 9 - 无悬的个人博客

Hook 常用在调试或监控场景，比如 hook malloc/free 可以用来排查内存泄漏，hook OpenGL api 可以用来调试渲染效果等 (如 RenderDoc)，Android 常用的 native hook 主要有 PLT hook 和 inline hook 两类，PLT hook 通过修改动态库的 plt/got 表来替换链接的函数地址，inline hook 则是修改内存中的方法指令，实现相对复杂，这里介绍一种针对 OpenGL ES 函数的 hook 方法：TLS hook，实现简单并且稳定性 & 兼容性高。

TLS hook 基于 Android 系统 OpenGL ES 驱动加载逻辑来实现，我们先基于源码梳理下这部分的流程：

#### 线程 TLS

Android 在线程 TLS 中存储了一些基础数据，其结构如下：[bionic_tls.h](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/private/bionic_tls.h "bionic_tls.h")

```
struct bionic_tcb {
  void* raw_slots_storage[BIONIC_TLS_SLOTS];
};

```

可以看看该结构体在 arm64 平台上的写入：[__set_tls.c](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/arch-arm64/bionic/__set_tls.c "__set_tls.c")

```
__LIBC_HIDDEN__ void __set_tls(void* tls) {
  asm("msr tpidr_el0, %0" : : "r" (tls));
}

```

读取：[tls.h](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/platform/bionic/tls.h "tls.h")

```
# define __get_tls() ({ void** __val; __asm__("mrs %0, tpidr_el0" : "=r"(__val)); __val; })

```

可以看到操作的是 `tpidr_el0` 寄存器，这个寄存器是用来给操作系统存储线程信息的，详见文档 [TPIDR-EL0–EL0-Read-Write-Software-Thread-ID-Register](https://robot9.me/android-tls-hook/(https://developer.arm.com/documentation/ddi0595/2021-06/AArch64-Registers/TPIDR-EL0--EL0-Read-Write-Software-Thread-ID-Register))。tls 指针数组的每一项具体定义具体可以查看 [tls_defines.h](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/platform/bionic/tls_defines.h)，其中对我们比较关键的是：

```
#define TLS_SLOT_OPENGL_API       4

```

接下来我们来看 EGL 如何和这个 tls slot 交互

#### EGL 初始化

我们知道 OpenGL ES 的实现 (驱动) 是不同厂商提供的，因此 EGL 需要从厂商提供的驱动库 (libGLESv1_CM.so、libGLESv2.so) 中加载 OpenGL ES api 的函数地址。首先是一些结构体定义：[hooks.h](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/hooks.h "hooks.h")

```
struct gl_hooks_t {
    struct gl_t {
        #include "entries.in"
    } gl;
    struct gl_ext_t {
        __eglMustCastToProperFunctionPointerType extensions[MAX_NUMBER_OF_GL_EXTENSIONS];
    } ext;
};

// We have a dedicated TLS slot in bionic
inline gl_hooks_t const * volatile * get_tls_hooks() {
    volatile void *tls_base = __get_tls();
    gl_hooks_t const * volatile * tls_hooks =
            reinterpret_cast<gl_hooks_t const * volatile *>(tls_base);
    return tls_hooks;
}

inline EGLAPI gl_hooks_t const* getGlThreadSpecific() {
    gl_hooks_t const * volatile * tls_hooks = get_tls_hooks();
    gl_hooks_t const* hooks = tls_hooks[TLS_SLOT_OPENGL_API];
    return hooks;
}


```

这里定义了 getGlThreadSpecific 函数，返回 gl_hooks_t 结构体，而 `#include "entries.in"` 其实就是定义了一系列函数指针，即 struct gl_t 就是所有 OpenGL ES api 函数指针列表，tls_hooks 地址就是前面提到的特殊寄存器中的值，再通过 TLS_SLOT_OPENGL_API 索引，转化为 gl_hooks_t 结构体指针，如下图：

![](https://robot9.me/wp-content/uploads/2022/06/Pasted-image-20220621165329.png)

我们知道 OpenGL ES api 调用是线程关联的，在当前线程没有绑定 Context(MakeCurrent) 情况下调 gl api 会报错，来看一下实现：[egl.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/egl.cpp "egl.cpp")

```
static pthread_once_t once_control = PTHREAD_ONCE_INIT;
static int sEarlyInitState = pthread_once(&once_control, &early_egl_init);

void setGLHooksThreadSpecific(gl_hooks_t const* value) {
    setGlThreadSpecific(value);
}

static int gl_no_context() {
    if (egl_tls_t::logNoContextCall()) {
        const char* const error = "call to OpenGL ES API with "
                                  "no current context (logged once per thread)";
        if (LOG_NDEBUG) {
            ALOGE(error);
        } else {
            LOG_ALWAYS_FATAL(error);
        }
        if (base::GetBoolProperty("debug.egl.callstack", false)) {
            CallStack::log(LOG_TAG);
        }
    }
    return 0;
}

static void early_egl_init(void) {
    int numHooks = sizeof(gHooksNoContext) / sizeof(EGLFuncPointer);
    EGLFuncPointer* iter = reinterpret_cast<EGLFuncPointer*>(&gHooksNoContext);
    for (int hook = 0; hook < numHooks; ++hook) {
        *(iter++) = reinterpret_cast<EGLFuncPointer>(gl_no_context);
    }

    setGLHooksThreadSpecific(&gHooksNoContext);
}

```

即通过 pthread_once 执行一次 early_egl_init，将 gl_no_context 函数指针设置到 gHooksNoContext 实例中的所有函数指针，然后将 gHooksNoContext 设置到 tls，本质上就是用 gl_no_context 来填充 TLS_SLOT_OPENGL_API 指向的函数指针列表，所以这个时候调任何 OpenGL ES api，都会跑到 gl_no_context 函数。

接下来看下 EGL 如何加载厂商驱动中的函数指针，关键函数为 `egl_init_drivers` (可以从[这里](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/eglApi.cpp)看到调用 `eglGetDisplay` 时会先调用 `egl_init_drivers`)

```
egl_connection_t gEGLImpl;
gl_hooks_t gHooks[2];

static EGLBoolean egl_init_drivers_locked() {
    if (sEarlyInitState) {
        // initialized by static ctor. should be set here.
        return EGL_FALSE;
    }

    // get our driver loader
    Loader& loader(Loader::getInstance());

    // dynamically load our EGL implementation
    egl_connection_t* cnx = &gEGLImpl;
    cnx->hooks[egl_connection_t::GLESv1_INDEX] = &gHooks[egl_connection_t::GLESv1_INDEX];
    cnx->hooks[egl_connection_t::GLESv2_INDEX] = &gHooks[egl_connection_t::GLESv2_INDEX];
    cnx->dso = loader.open(cnx);

    // Check to see if any layers are enabled and route functions through them
    if (cnx->dso) {
        // Layers can be enabled long after the drivers have been loaded.
        // They will only be initialized once.
        LayerLoader& layer_loader(LayerLoader::getInstance());
        layer_loader.InitLayers(cnx);
    }

    return cnx->dso ? EGL_TRUE : EGL_FALSE;
}

```

具体的驱动加载逻辑在 loader.open 函数中 ([详见源码](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/Loader.cpp))，加载后的指针存放在 egl_connection_t 结构体中 (gEGLImpl 实例)，这个结构体的定义：[egldefs.h](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/egldefs.h "egldefs.h")

```
struct egl_connection_t {
    enum { GLESv1_INDEX = 0, GLESv2_INDEX = 1 };
    ...
    gl_hooks_t* hooks[2];
    ...
};

```

其中 hooks 指针数组指向 gHooks 全局对象，其中存储的就是从驱动中加载的不同版本 OpenGL ES 函数指针。

到这里还没完，我们需要在要用的时候把 TLS_SLOT_OPENGL_API slot 填充为从驱动加载的函数指针，这部分逻辑实现在 EGL 的 MakeCurrent 中：[egl_platform_entries.cpp](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/egl_platform_entries.cpp "egl_platform_entries.cpp")

```
EGLBoolean eglMakeCurrentImpl(EGLDisplay dpy, EGLSurface draw, EGLSurface read, EGLContext ctx) {
    ...
    EGLBoolean result = dp->makeCurrent(c, cur_c, draw, read, ctx, impl_draw, impl_read, impl_ctx);
    if (result == EGL_TRUE) {
        if (c) {
            setGLHooksThreadSpecific(c->cnx->hooks[c->version]);
            egl_tls_t::setContext(ctx);
            _c.acquire();
            _r.acquire();
            _d.acquire();
        } else {
            setGLHooksThreadSpecific(&gHooksNoContext);
            egl_tls_t::setContext(EGL_NO_CONTEXT);
        }
    } else {
        // this will ALOGE the error
        egl_connection_t* const cnx = &gEGLImpl;
        result = setError(cnx->egl.eglGetError(), (EGLBoolean)EGL_FALSE);
    }
    return result;
}

```

EGL 内部对 EGLContext 的存储也是基于 TLS，详见 [egl_tls_t](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/EGL/egl_tls.h) 定义， 从上面 `eglMakeCurrentImpl` 函数实现可以看出，如果是 MakeCurrent 的是 EGL_NO_CONTEXT，则将 TLS_SLOT_OPENGL_API slot 填充为 gl_no_context，否则填充为 `egl_connection_t` 中的 `hooks[version]`(驱动加载得到的函数指针)。同时，MakeCurrent 还会将 ctx 设置到 TLS `egl_tls_t::setContext(ctx)`

#### TLSHook 实现

在理解 EGL 的加载流程后，要实现 Hook 就比较简单了，我们只需要在合适的时候把 TLS_SLOT_OPENGL_API 指向的函数指针 (gHooks 全局对象) 换掉即可，主要步骤如下：

##### 1、获取 tls

这里我们直接使用 Android [源码](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/platform/bionic/tls.h) 以兼容不同平台

```
#if defined(__aarch64__)  
# define __get_tls() ({ void** __val; __asm__("mrs %0, tpidr_el0" : "=r"(__val)); __val; })  
#elif defined(__arm__)  
# define __get_tls() ({ void** __val; __asm__("mrc p15, 0, %0, c13, c0, 3" : "=r"(__val)); __val; })  
#elif defined(__i386__)  
# define __get_tls() ({ void** __val; __asm__("movl %%gs:0, %0" : "=r"(__val)); __val; })  
#elif defined(__x86_64__)  
# define __get_tls() ({ void** __val; __asm__("mov %%fs:0, %0" : "=r"(__val)); __val; })  
#else  
#error unsupported architecture  
#endif

void *volatile *get_tls_hooks() {  
  volatile void *tls_base = __get_tls();  
  void *volatile *tls_hooks = reinterpret_cast<void *volatile *>(tls_base);  
  return tls_hooks;  
}  

void *getGlThreadSpecific(int idx) {  
  void *volatile *tls_hooks = get_tls_hooks();  
  void *hooks = tls_hooks[idx];  
  return hooks;  
}

```

##### 2、确保已加载驱动

根据前面的分析，如果线程当前没有设置 EGLContext(MakeCurrent)，tls 指向的是 gHooksNoContext，因此需要手动创建一个并 MakeCurrent：

```
bool needCreateEGL = (eglGetCurrentContext() == EGL_NO_CONTEXT);  
if (needCreateEGL) {   
  if (egl_core.create(1, 1)) {  
    egl_core.makeCurrent();    
  }
}

```

##### 3、替换函数

如前面的分析，gl_hooks_t 中的指针列表是按 entries.in 中的定义顺序排列，因此每个 OpenGL ES api 都有一个数组 index（也即相对起始地址的偏移），根据这个 index，找到对应位置替换函数指针即可：

```
size_t *tlsPtr = static_cast<size_t *>(getGlThreadSpecific(TLS_SLOT_OPENGL_API));
size_t *slot = tlsPtr + index;  

*old_func = reinterpret_cast<size_t *>(*slot);  // 保存原指针 
*slot = reinterpret_cast<size_t>(new_func);  // 替换为新指针

```

##### 4、兼容

整个 Hook 看起来比较简单，但存在一定的兼容性问题，即 OpenGL ES api 的 index 并不是固定的，可以查看 entries.in 文件的[提交记录](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/entries.in;bpv=1)，这里有两种解决方法：分版本兼容或手动计算 index。

分版本兼容：我们根据提交记录，保存现有不同版本的 entries.in，然后根据系统版本加载：

```
    int idx = 0;  

#define GL_ENTRY(_r, _api, ...) hookMap[#_api] = idx++;  
    if (api_level >= 28) {  
#include "entry/entries.28.in"  
    } else if (api_level >= 24) {  
#include "entry/entries.24.in"  
    } else if (api_level >= 21) {  
#include "entry/entries.21.in"  
    } else if (api_level >= 18) {  
#include "entry/entries.18.in"  
    } else if (api_level >= 16) {  
#include "entry/entries.16.in"  
    }

```

这样实现简单，除非厂商私自改动这个 entries.in，否则都是兼容的，实测云平台的 80 + 款设备都没有兼容问题。

另一种方法是手动计算 index，即我们先把地址填充成特定的 Hook 函数，然后尝试去调一下要 Hook 的 api，如果 Hook 成功，即找到了该 api 的 index，否则继续测试下一个地址，可以看下微信开源的 Matrix 中的[实现](https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-opengl-leak/src/main/cpp/com_tencent_matrix_openglleak_detector_FuncSeeker.cpp)：

```
extern "C"
JNIEXPORT jint JNICALL
Java_com_tencent_matrix_openglleak_detector_FuncSeeker_getGlTexImage2DIndex(JNIEnv *env,
                                                                            jclass clazz) {
    gl_hooks_t *hooks = get_gl_hooks();
    if (NULL == hooks) {
        return -1;
    }

    for (i_glTexImage2D = 0; i_glTexImage2D < 1000; i_glTexImage2D++) {
        if (has_hook_glTexImage2D) {
            i_glTexImage2D = i_glTexImage2D - 1;

            void **method = (void **) (&hooks->gl.foo1 + i_glTexImage2D);
            *method = (void *) _system_glTexImage2D;
            break;
        }

        if (_system_glTexImage2D != NULL) {
            void **method = (void **) (&hooks->gl.foo1 + (i_glTexImage2D - 1));
            *method = (void *) _system_glTexImage2D;
        }

        void **replaceMethod = (void **) (&hooks->gl.foo1 + i_glTexImage2D);
        _system_glTexImage2D = (System_GlTexImage2D) *replaceMethod;

        *replaceMethod = (void *) _my_glTexImage2D;

        glTexImage2D(0, 0, 0, 0, 0, 0, 0, 0, NULL);
    }

    // release
    _system_glTexImage2D = NULL;
    has_hook_glTexImage2D = false;
    int result = i_glTexImage2D;
    i_glTexImage2D = 0;

    return result;
}

```

这种方法兼容性更好，但需要提前对 api 的 index 进行查找，实现相对复杂一点。

#### 示例 demo

首先定义 Hook 函数，这里我们 Hook `glClearColor`，不管什么参数都改为蓝色 (0.f, 0.f, 1.f, 1.f)

```
// origin function  
PFNGLCLEARCOLORPROC cb_glClearColor = nullptr;  

// new function  
void hook_glClearColor(GLfloat red, GLfloat green, GLfloat blue, GLfloat alpha) {  
  LOGD("hook call glClear: (%f, %f, %f, %f)", red, green, blue, alpha);  
  if (cb_glClearColor) {  
    cb_glClearColor(0.f, 0.f, 1.f, 1.f);  
  }  
}

```

然后开始 Hook

```
TLSHook::tls_hook_init();
TLSHook::tls_hook_func("glClearColor", 
                       (void *) hook_glClearColor,
                       (void **) &cb_glClearColor);

```

再创建个简单的 GLSurfaceView，指定 clear color 为红色

```
surfaceView.setRenderer(new GLSurfaceView.Renderer() {  
    @Override  
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {  
    }  

    @Override  
    public void onSurfaceChanged(GL10 gl, int width, int height) {  
    }  

    @Override  
    public void onDrawFrame(GL10 gl) {  
        GLES20.glClearColor(1.f, 0.f, 0.f, 1.f);  
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);  
    }  
});

```

最终跑起来可以看到 GLSurfaceView 显示蓝色，Hook 成功。

#### Github

完整代码详见：[https://github.com/keith2018/TLSHook](https://github.com/keith2018/TLSHook)